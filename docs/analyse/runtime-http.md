# 런타임과 HTTP 처리

## CLI 명령 모델

`glance.Main()`은 [../../internal/glance/cli.go](../../internal/glance/cli.go)의 파싱 결과에 따라 다음 동작을 선택한다.

| 명령 | 구현 동작 |
| --- | --- |
| 인자 없음 | 설정을 읽고 HTTP 서버 실행 |
| `--version`, `-v`, `version` | 빌드 버전 출력 |
| `config:validate` | include와 변수 치환을 포함해 설정 파싱·위젯 초기화까지 검증 |
| `config:print` | include를 펼친 YAML 출력. 환경변수 치환 전 단계의 내용이다. |
| `diagnose` | DNS 및 외부 API 연결 진단을 병렬 실행 |
| `sensors:print` | gopsutil 온도 센서 목록 출력 |
| `secret:make` | 인증용 64바이트 난수를 Base64로 출력 |
| `password:hash <pwd>` | bcrypt 기본 cost로 비밀번호 해시 생성 |
| `mountpoint:info <path>` | 의도상 filesystem 정보를 출력하지만 현재 파서 분기 결함으로 선택되지 않음 |

기본 설정 경로는 실행 위치의 `glance.yml`이고 `--config`로 변경한다. 이 저장소 루트에는 `glance.yml`이 없으므로 포함된 샘플로 실행할 때는 `--config docs/glance.yml`이 필요하다.

## 서버 시작 생명주기

1. 메인 YAML과 include 파일을 읽는다.
2. `fsnotify` watcher를 구성한다.
3. watcher 설정에 성공하면 초기 내용도 `onChange` 콜백으로 전달한다.
4. 콜백이 설정을 파싱하고 `application`을 생성한다.
5. 새 `http.Server`를 goroutine에서 시작한다.
6. 설정 변경 시 새 application을 먼저 생성하고, 기존 서버를 `Close()`한 후 새 서버를 시작한다.

초기 설정이 잘못되면 exit channel을 닫고 프로세스가 오류 코드로 끝난다. 실행 중 변경된 설정이 잘못되면 오류만 로그에 남기고 기존 서버는 계속 실행된다. watcher 자체를 만들지 못한 경우에는 자동 재로드 없이 한 번만 파싱해 서버를 동기적으로 실행한다.

재로드는 전체 application 교체다. 따라서 위젯 캐시, 인증 실패 기록, 페이지 mutex와 렌더링된 manifest가 모두 새로 만들어진다. `widgetIDCounter`만 패키지 전역 atomic 값이므로 재로드 후에도 증가를 계속한다.

## HTTP 라우트

라우트는 Go 1.22 이후 `ServeMux` 메서드·경로 패턴을 사용한다.

| 메서드와 경로 | 인증 | 응답·역할 |
| --- | --- | --- |
| `GET /` | 설정 시 필요 | 첫 번째 페이지 프레임 |
| `GET /{page}` | 설정 시 필요 | slug에 해당하는 페이지 프레임 |
| `GET /api/pages/{page}/content/` | 설정 시 필요 | 위젯 갱신 후 HTML 조각 |
| `POST /api/set-theme/{key}` | 불필요 | 테마 쿠키 설정과 CSS 반환 |
| `/api/widgets/{widget}/{path...}` | 불필요 | 현재 항상 501 Not Implemented |
| `GET /api/healthz` | 불필요 | 본문 없이 200 OK |
| `GET /login` | 불필요 | 로그인 페이지. 이미 인증되면 `/`로 리다이렉트 |
| `GET /logout` | 불필요 | 세션 쿠키 만료 후 로그인으로 리다이렉트 |
| `POST /api/authenticate` | 불필요 | 자격 증명 검증과 세션 쿠키 발급 |
| `GET /static/{hash}/{path...}` | 불필요 | 내장 정적 파일 |
| `GET /static/{hash}/css/bundle.css` | 불필요 | 런타임에 결합한 CSS |
| `GET /manifest.json` | 불필요 | 설정 기반 PWA manifest |
| `/assets/{path...}` | 불필요 | `assets-path`가 있을 때 사용자 파일 |

`base-url`은 생성되는 URL과 쿠키 path에 접두사로 사용된다. mux 라우트 자체를 해당 접두사 아래에 등록하지는 않으므로 reverse proxy가 외부 접두사를 제거해서 전달하는 구성을 전제로 한다.

## 페이지 식별

- 첫 번째 설정 페이지는 빈 slug `""`에도 등록되어 `/`가 해당 페이지를 표시한다.
- 명시적 slug가 없으면 페이지 제목을 소문자화하고 영숫자가 아닌 구간을 `-`로 바꿔 생성한다.
- `login`, `logout`은 예약 slug다.
- `slugToPage`는 map이며 중복 slug 검증은 없다. 동일 slug가 반복되면 뒤 페이지가 해당 map 항목을 덮어쓴다.
- 내비게이션은 map이 아니라 설정의 `Pages` 순서를 그대로 사용한다.

## 두 단계 페이지 렌더링

첫 응답의 [../../internal/glance/templates/page.html](../../internal/glance/templates/page.html)은 내비게이션, 헤더, footer, 로딩 표시와 빈 `#page-content`를 만든다. 위젯 HTML은 포함하지 않는다.

브라우저의 `page.js`가 페이지 콘텐츠 API를 호출하면 서버는 다음 순서로 처리한다.

1. slug를 조회하고 인증을 확인한다.
2. 페이지 mutex를 잠근다.
3. 모든 head widget과 column widget에 대해 `requiresUpdate`를 검사한다.
4. 필요한 위젯을 각각 goroutine에서 갱신하고 `WaitGroup`으로 기다린다.
5. [../../internal/glance/templates/page-content.html](../../internal/glance/templates/page-content.html)을 실행한다.
6. HTML을 반환하고 mutex를 해제한다.

초기 HTML에 실제 콘텐츠가 없기 때문에 정상적인 화면 완성에는 JavaScript가 필요하다. 콘텐츠 API가 느리면 로딩 상태가 그대로 유지된다.

## HTTP 서버 설정

`http.Server`에는 다음 값이 설정된다.

- 주소: `server.host:server.port`
- `ReadHeaderTimeout`: 5초
- `ReadTimeout`: 10초
- `WriteTimeout`: 설정하지 않음
- `IdleTimeout`: 설정하지 않음
- TLS: 애플리케이션 자체에서는 제공하지 않음

설정 재로드 시 `Shutdown`이 아니라 `Close`를 사용하므로 열려 있는 연결을 즉시 닫는다. OS signal을 직접 처리하는 graceful shutdown 로직은 없다.

## 정적 자산과 캐시

- `static` 디렉터리 전체 내용으로 MD5를 계산하고 앞 10자리 hex를 URL에 넣는다. 이 해시는 보안 목적이 아니라 cache busting용이다.
- 내장 정적 파일과 CSS 번들은 `public, max-age=86400`을 사용한다.
- 사용자 `/assets` 파일은 2시간 캐시한다.
- manifest는 URL query에 application 생성 시각을 사용하지만 응답 자체에는 정적 자산과 동일한 24시간 cache header가 붙는다.
- CSS 번들은 `css/main.css`의 `@import`를 최대 깊이 20까지 펼치고 단일 행 주석, 행 앞 공백, 개행을 제거한다.

## 이전 Docker 설정 감지

Docker 내부에서 지정된 config 경로가 없고 작업 디렉터리에 예전 위치의 `glance.yml`이 있으면 일반 애플리케이션 대신 8080 포트에 503 안내 페이지를 제공한다. 이는 v0.7 설정 경로 이전을 위한 호환 코드이며 소스 주석상 v0.10에서 제거 예정인 동작이다.

