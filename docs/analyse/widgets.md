# 위젯 시스템

## 위젯 계약

[../../internal/glance/widget.go](../../internal/glance/widget.go)의 `widget` 인터페이스는 모든 위젯에 다음 동작을 요구한다.

| 메서드 | 역할 |
| --- | --- |
| `initialize()` | 설정 기본값·검증, 정적 HTML 준비 |
| `requiresUpdate(now)` | 캐시 상태에 따라 갱신 필요 여부 판단 |
| `update(ctx)` | 데이터 취득과 화면 모델 갱신 |
| `Render()` | `template.HTML` 반환 |
| `setProviders()` | 내장 asset URL resolver 주입 |
| `GetType()`, `GetID()`, `setID()` | 형식과 런타임 식별자 관리 |
| `handleRequest()` | 개별 위젯 HTTP 요청 계약. 현재 공통 구현은 501이며 라우트도 비활성 상태와 같음 |
| `setHideHeader()` | group 등 컨테이너가 하위 헤더 표시를 제어 |

YAML의 위젯 배열은 custom `UnmarshalYAML`을 사용한다. 각 항목에서 먼저 `type`만 읽고 factory switch로 구체 구조체를 만든 뒤 같은 YAML node를 다시 그 구조체에 decode한다. 알 수 없거나 빈 type은 YAML line 번호를 포함한 오류가 된다.

## 공통 상태

`widgetBase`가 보유하는 상태는 다음과 같다.

- 설정값: `type`, `title`, `title-url`, `hide-header`, `css-class`, `cache`
- 런타임값: atomic counter에서 부여한 ID, provider, 콘텐츠 존재 여부, WIP 표시
- 오류값: 전체 실패 `Error`, 부분 실패 `Notice`
- 캐시값: cache type, 기본/사용자 duration, 다음 갱신 시각, 연속 재시도 횟수
- 렌더링값: 재사용하는 `bytes.Buffer`

템플릿 렌더링이 실패하면 buffer를 비우고 Error가 포함된 상태로 같은 템플릿을 한 번 더 실행한다. 두 번째도 실패하면 빈 결과를 반환한다.

## 캐시와 오류 모델

캐시는 세 유형이다.

| 유형 | 동작 |
| --- | --- |
| Infinite | `initialize`에서 완성되는 정적 위젯. 갱신하지 않음 |
| Duration | 성공 시 `현재 시각 + duration`까지 재사용 |
| On the hour | 성공 시 다음 정각까지 재사용 |

Duration 위젯은 공통 `cache` YAML 속성이 있으면 기본 duration 대신 사용한다. duration parser는 정수와 `s/m/h/d` 단위만 지원한다.

갱신 실패 시 1, 4, 9, 16, 25분 간격의 제곱 backoff를 계산하되 정상 캐시 만료보다 늦어지면 정상 만료 시각을 택한다. `errPartialContent`이면 성공한 일부 결과를 저장하고 Notice를 표시한다. 그 외 오류는 새 모델을 적용하지 않고 Error를 표시한다. 성공 시 오류와 notice를 지우고 정상 캐시 시각을 예약한다.

모든 캐시는 메모리 전용이다. 설정 재로드나 프로세스 재시작 시 사라진다.

## 병렬 수집

- 페이지는 갱신이 필요한 최상위 위젯마다 goroutine을 만든다.
- `group`과 `split-column`은 갱신이 필요한 자식마다 다시 goroutine을 만든다.
- generic worker pool의 기본 worker 수는 10이다.
- Hacker News 및 YouTube feed 수집은 30 workers, ChangeDetection은 15, Monitor는 20을 사용한다.
- Repository는 상세·PR·issue·commit 요청을 최대 네 goroutine으로 실행한다.
- Custom API의 primary와 subrequest는 동시에 실행하며 하나가 실패하면 공유 context를 cancel한다.
- `server-stats`는 원격 서버별 goroutine을 사용한다.

worker pool 결과는 입력 index에 맞춰 배열에 저장하므로 실행 순서와 관계없이 입력 순서를 보존할 수 있다. job context는 현재 항상 `context.Background()`이며 context 주입 메서드는 주석 처리되어 있다.

## HTTP 공통 계층

[../../internal/glance/widget-utils.go](../../internal/glance/widget-utils.go)에 기본 HTTP client와 decoder가 있다.

- 기본 timeout: 5초
- 기본 transport: 환경변수 proxy 사용, host당 idle connection 최대 10
- insecure client: 인증서 검증을 끄고 같은 5초 timeout 사용
- JSON/XML decoder: 전체 body를 메모리로 읽고 HTTP 200만 성공으로 인정
- 오류 응답: status, URL, 최대 256자의 body를 오류 문자열에 포함
- 브라우저 User-Agent가 필요한 서비스는 Firefox 형태의 값을 생성하며 낮은 확률로 버전 숫자를 바꾼다.

일부 위젯은 이 공통 계층을 사용하지 않는다. Extension은 timeout이 없는 `http.DefaultClient`를 사용하고, Docker는 source별 client와 5초 request context를 만든다. Reddit은 HTTP/2와 uTLS를 사용하는 별도 client 경로가 있다.

## 위젯 카탈로그

factory가 인식하는 canonical type은 28개다. `stocks`는 `markets`의 legacy alias다.

### 정적·브라우저 위젯

| type | 서버 캐시 | 구현과 상태 원천 |
| --- | --- | --- |
| `bookmarks` | Infinite | YAML 그룹·링크를 초기화 시 HTML로 렌더링. group 값과 link override를 합성 |
| `calendar` | Infinite | 빈 calendar shell을 렌더링하고 브라우저 `calendar.js`가 현재 날짜와 월 전환 UI 생성 |
| `clock` | Infinite | timezone을 서버에서 검증하고 브라우저가 현재 시간을 계속 갱신 |
| `html` | Infinite | YAML `source`를 신뢰된 `template.HTML`로 그대로 반환 |
| `iframe` | Infinite | source URL과 높이를 반영한 iframe. 높이는 최소 50, 정확히 50이면 300으로 교정 |
| `search` | Infinite | DuckDuckGo/Google/Bing/Perplexity/Kagi/Startpage 또는 사용자 URL, bang을 브라우저에서 처리 |
| `to-do` | Infinite | 서버는 shell만 제공. 항목은 `todo-{id}` key로 브라우저 localStorage에 저장 |
| `calendar-legacy` | 정각 | 서버가 현재 주를 중심으로 21일을 계산하는 이전 calendar 구현 |

### 외부 콘텐츠 위젯

| type | 기본 캐시 | 데이터 원천과 주요 동작 |
| --- | --- | --- |
| `weather` | 다음 정각 | Open-Meteo geocoding과 forecast. 장소 결과를 위젯에 보존하고 metric/imperial 변환 |
| `markets` | 1시간 | Yahoo Finance chart API. 21개 종가로 sparkline 생성, change 정렬 지원 |
| `rss` | 2시간 | 임의 RSS/Atom feed. `gofeed`, ETag/Last-Modified, feed별 마지막 성공 결과 사용. 네 가지 layout |
| `videos` | 1시간 | YouTube channel/playlist Atom feed. channel별 `sync.Map` 캐시와 실패 시 이전 목록 사용 |
| `reddit` | 30분 | 공개 JSON 또는 OAuth app API. proxy, 검색·정렬·thumbnail·card layout, 별도 uTLS/HTTP2 경로 |
| `hacker-news` | 30분 | Firebase API에서 story ID 후 item을 병렬 조회. 상위 40개를 취득한 뒤 limit 적용 |
| `lobsters` | 1시간 | 기본/사용자 Lobsters 호환 JSON feed. instance, tag, hot/new 경로 조합 |
| `twitch-channels` | 10분 | Twitch persisted GraphQL query. 채널별 요청 후 viewers/live 정렬 |
| `twitch-top-games` | 10분 | Twitch directory persisted GraphQL query. exclude와 limit 적용 |
| `releases` | 2시간 | GitHub, GitLab, Codeberg, Docker Hub 최신 release/tag를 병렬 수집하고 시간순 정렬 |
| `repository` | 1시간 | GitHub repository 상세, open PR, issue, 최근 commit을 병렬 수집 |
| `change-detection` | 1시간 | ChangeDetection.io API. watch ID 목록 또는 지정 IDs를 조회하고 최근 변경순 정렬 |

RSS와 Videos는 위젯 수준 캐시 외에 feed 단위 내부 캐시도 가진다. 여러 원천 중 일부가 실패해도 이전 feed 결과 또는 성공한 원천을 조합해 부분 콘텐츠를 만들 수 있다.

### 인프라·상태 위젯

| type | 기본 캐시 | 데이터 원천과 주요 동작 |
| --- | --- | --- |
| `monitor` | 5분 | URL별 GET, 기본 timeout 3초, Basic Auth·TLS 검증 해제·대체 성공 status 지원. 20 workers |
| `docker-containers` | 1분 | 기본 `/var/run/docker.sock`; Unix socket 또는 tcp/http/https Docker API. 5초 context |
| `dns-stats` | 10분 | AdGuard Home, Pi-hole v5, Pi-hole v6, Technitium. 통계·시계열·차단 상위 도메인 |
| `server-stats` | 15초 | 기본 local gopsutil 수집 또는 원격 `/api/sysinfo/all`. 소스에서 WIP flag 설정 |

Docker widget은 label과 YAML override를 결합해 이름·icon·URL·category·parent·숨김 여부를 결정한다. parent/child 관계를 구성한 뒤 상태 icon 우선순위와 이름으로 정렬한다.

Server Stats의 로컬 수집은 hostname/platform/boot time, CPU load·온도, memory/swap, mountpoint를 읽는다. hostname/platform/boot time은 프로세스 전체에서 무기한 캐시된다. remote server는 기본 3초 timeout과 선택적 bearer token을 사용한다.

### 사용자 정의·컨테이너 위젯

| type | 기본 캐시 | 구현 동작 |
| --- | --- | --- |
| `custom-api` | 1시간 | 임의 HTTP method/body/header/query/Basic Auth 요청, GJSON 조회와 Go HTML template 렌더링 |
| `extension` | 30분 | 임의 URL의 응답 body와 `Widget-*` 헤더를 widget 모델로 변환 |
| `group` | 자식에 위임 | 여러 위젯을 tab group으로 렌더링. 자식 헤더를 숨기며 group/split-column 중첩 금지 |
| `split-column` | 자식에 위임 | 자식 위젯을 동적 내부 열로 렌더링. 최소 max-columns는 2 |

Custom API는 primary request가 없어도 빈 응답 모델로 동작할 수 있다. subrequest가 있으면 모두 병렬 실행한다. 응답은 기본적으로 JSON 유효성을 검사하고, 2xx인데 JSON이 아니면 `invalid response JSON`, 비-2xx이면 status 오류를 반환한다. template 함수에는 GJSON 접근, 산술, 시간, 정렬, regexp, 문자열, 동적 추가 요청이 포함된다.

Extension은 응답의 `Widget-Title`, `Widget-Title-URL`, `Widget-Content-Type`, `Widget-Content-Frameless` 헤더를 해석한다. `allow-potentially-dangerous-html`이 false이면 HTML도 `<pre>` 안에 escape해 출력하고, true일 때만 응답 body를 raw HTML로 취급한다.

## 공통 템플릿 상태 표현

[../../internal/glance/templates/widget-base.html](../../internal/glance/templates/widget-base.html)이 widget frame을 정의한다.

- `Error`가 있고 이전 콘텐츠가 없으면 오류 메시지가 본문을 대체한다.
- 이전 콘텐츠가 있으면 기존 결과를 유지하면서 오류 상태를 표시할 수 있다.
- `Notice`는 부분 콘텐츠 경고에 사용된다.
- `WIP`은 제목 영역에 실험 상태 표시를 추가한다.
- `HideHeader`, `CSSClass`, title URL은 모든 위젯에서 공통 적용된다.

## 런타임 ID와 위젯 API

ID는 YAML decode 시 패키지 전역 `atomic.Uint64`에서 부여된다. application은 최상위 head/column 위젯을 `widgetByID` map에 등록한다. 컨테이너 내부 자식은 이 map에 별도로 등록하지 않는다.

`/api/widgets/{widget}/{path...}` 라우트와 `handleRequest` 인터페이스는 존재하지만 application handler는 현재 즉시 501을 반환하며 ID 파싱·조회·위임 코드는 주석 처리되어 있다. 따라서 현재 사용자 동작에서 widget ID는 HTML data attribute와 내부 식별 용도에 한정된다.

