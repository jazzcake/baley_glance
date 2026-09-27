# 로컬 운영 대시보드 구현 로드맵

## 문서 목적

이 문서는 [requirements-analysis.md](requirements-analysis.md)의 분석을 실제 작업 순서로 바꾼다. 목표는 요구사항 전체를 미리 일반화하는 것이 아니라, 읽기 전용 로컬 운영 대시보드를 작은 수직 단위로 완성하는 것이다.

로드맵은 다음 개발 원칙을 전제로 한다.

- 검증은 단위 테스트, HTTP handler 테스트, `go test ./...`, 짧은 수동 smoke test까지만 수행한다.
- 브라우저 호환성 행렬, 부하 시험, 침투 시험, 장기 soak test, 완전한 장애 복구 체계는 초기 범위에서 제외한다.
- 각 변경은 먼저 작은 실패 테스트를 만들고, 그 테스트를 통과시키는 최소 코드만 작성한다.
- 현재 필요하지 않은 추상화와 범용 framework는 만들지 않는다.
- 문제가 실제로 발견되거나 새 요구가 생기면 그때 범위를 확장한다.
- 원본 데이터, 사용자 설정, 기존 source 또는 운영 상태가 삭제·덮어쓰기·비가역 변환될 가능성이 있으면 실행 전에 영향과 복구 가능성을 알리고 동의를 받는다.

## 로드맵의 기본 판단

초기 구현 대상은 **최상위 read-only `custom-api` widget**으로 제한한다. 모든 built-in widget과 `group`·`split-column` 내부 child까지 범용 partial refresh를 지원하려 하면 frontend component 생명주기와 상태 보존 문제가 선행되어 범위가 급격히 커진다.

첫 결과물은 다음 흐름이 실제로 동작하는 상태다.

```text
Overview custom-api 카드
  ├─ 지정된 주기로 자기 HTML만 갱신
  ├─ server cache가 남아 있으면 upstream을 다시 호출하지 않음
  ├─ 카드 클릭으로 Glance detail page 이동
  └─ detail page에서 외부 운영 UI로 이동
```

이 흐름이 안정된 뒤 실제 운영 source를 하나씩 연결한다. Attention과 Activity 같은 통합 화면은 개별 source의 상태 카드가 먼저 동작한 뒤 만든다.

## 초기 범위 결정

분석 문서의 미결정 사항 가운데 첫 구현을 위해 필요한 것은 다음과 같이 고정한다.

| 항목 | 초기 결정 |
| --- | --- |
| `refresh` 기본값 | 미지정 시 비활성화하고 기존 동작 유지 |
| 허용 대상 | 최상위 `custom-api` widget만 |
| 시간 범위 | 최소 2초, 최대 1시간 |
| cache 관계 | refresh는 view 재요청일 뿐 cache를 무시하지 않음 |
| 요청 중첩 | 이전 요청이 끝난 뒤 다음 timer 예약 |
| background tab | 초기에는 별도 최적화 없이 계속 갱신 |
| widget 식별 | 기존 runtime numeric ID를 document 수명 handle로 사용 |
| stale ID | 404 응답 후 해당 widget polling 중지 및 page reload 안내 |
| fragment 범위 | 공통 `.widget` root 전체 HTML |
| frontend 재초기화 | 상호작용 없는 custom-api fragment만 지원하므로 수행하지 않음 |
| 인증 | 기존 page content와 같은 인증 검사 재사용 |
| 카드 이동 | 내부 relative URL, 같은 tab 이동을 우선 지원 |
| interactive child | link, button, input 등의 클릭은 카드 이동에서 제외 |
| detail navigation | 초기에는 기존 global navigation 노출을 허용 |
| 오류 표현 | 기존 widget Error/Notice와 last-good 표현 유지 |

이 결정은 영구 API 약속이 아니라 초기 제품 범위다. 실제 사용에서 built-in widget refresh, hidden detail page, background polling 절감 등이 필요해질 때 별도 작업으로 확장한다.

## 테스트 우선 작업 규칙

각 작업 항목은 다음 순서로 진행한다.

1. 바꾸려는 동작 한 가지를 보여 주는 실패 테스트를 추가한다.
2. 테스트를 통과시키는 최소 구현을 작성한다.
3. 해당 package 테스트를 실행한다.
4. milestone 끝에서 `go test ./...`를 한 번 실행한다.
5. HTTP나 browser 동작이 포함된 milestone만 짧게 수동 확인한다.
6. 구현 중 드러난 실제 제약만 문서에 반영한다.

동시성 경계를 수정하는 milestone에서는 해당 package의 `go test -race`를 한 번 수행한다. 이는 별도의 무거운 검증 단계가 아니라 공유 widget state를 바꾸는 코드에 필요한 최소 확인으로 취급한다.

테스트 수를 늘리는 것이 목적은 아니다. 각 테스트는 다음 중 하나만 증명해야 한다.

- 새 설정이 올바르게 해석된다.
- 새 endpoint가 약속한 status와 fragment를 반환한다.
- cache와 refresh의 호출 횟수 관계가 유지된다.
- click 또는 polling의 핵심 browser 동작이 유지된다.
- 기존 동작이 새 옵션 미지정 시 바뀌지 않는다.

## Milestone 0 — 기준선 고정

### 목표

변경 전에 현재 동작을 짧은 characterization test로 고정한다.

### 작업

- 기존 widget ID, cache 만료, Error/Notice 렌더링 중 새 기능이 의존하는 동작만 테스트한다.
- page content 인증 검사와 custom-api 기본 렌더링 경로를 재현 가능한 fixture로 만든다.
- 테스트에서 사용할 fake widget 또는 local `httptest.Server` helper를 최소 범위로 둔다.
- 현재 문서의 기준선 명령 `go test ./...`를 실행한다.

### 완료 조건

- 새 기능 구현 없이 기준선 테스트가 통과한다.
- 이후 endpoint 테스트가 실제 외부 API나 인터넷에 의존하지 않는다.

### 제외

- 전체 기존 코드의 테스트 보강
- coverage 목표 설정
- end-to-end test framework 도입

## Milestone 1 — `refresh` 설정 계약

### 목표

YAML에서 widget별 view refresh interval을 선언하되 옵션을 쓰지 않은 기존 설정은 그대로 동작하게 한다.

### 먼저 작성할 테스트

- `refresh` 미지정 시 polling metadata가 생성되지 않는다.
- `refresh: 2s`가 정상 해석된다.
- 2초 미만 또는 1시간 초과 값은 config 오류가 된다.
- 지원하지 않는 widget에 `refresh`를 지정하면 명확한 config 오류가 된다.
- `cache`와 `refresh`는 서로 다른 값으로 유지된다.

### 최소 구현

- `widgetBase` 또는 custom-api 설정에 optional refresh duration을 추가한다.
- 공통 widget root에 runtime ID와 refresh interval을 data attribute로 출력한다.
- 기존 `cache` 계산과 update 주기는 수정하지 않는다.
- 사용자 설정 reference에 새 필드를 짧게 기록한다.

### 완료 조건

- 기존 YAML은 출력과 동작이 동일하다.
- 지원 범위 밖 설정은 server 시작 시 바로 설명 가능한 오류를 낸다.

## Milestone 2 — 위젯 단위 동시성 경계

### 목표

page render와 partial render가 같은 widget state와 template buffer를 동시에 변경하지 않게 한다.

### 먼저 작성할 테스트

- 동시에 들어온 동일 widget update 두 개가 upstream fetch를 중복 수행하지 않는다.
- cache가 유효하면 반복 render가 fetch 횟수를 늘리지 않는다.
- page render와 widget render를 동시에 실행해도 결과가 깨지지 않는다.

### 최소 구현

- widget update와 render를 함께 감싸는 per-widget lock을 도입한다.
- 초기 page content도 같은 lock 경계를 사용한다.
- page 전체 lock은 application/page map 보호에 필요한 범위로만 남긴다.
- context propagation이나 전면적인 state immutable화는 이 단계에서 하지 않는다.

### 완료 조건

- 관련 단위 테스트와 `go test -race`가 통과한다.
- cache hit와 miss의 기존 의미가 유지된다.

## Milestone 3 — Partial widget endpoint

### 목표

현재 501인 `/api/widgets/{widget}/{path...}` 경로가 대상 widget의 최신 HTML fragment를 반환하게 한다.

### 먼저 작성할 테스트

- 존재하는 refresh 대상 ID는 HTTP 200과 하나의 `.widget` root를 반환한다.
- 존재하지 않거나 설정 reload 뒤 사라진 ID는 404를 반환한다.
- refresh 비대상 widget ID는 404 또는 명시된 비지원 status를 반환한다.
- 인증 설정이 있으면 인증 없는 요청은 기존 content API와 같은 방식으로 거부된다.
- upstream 갱신 실패는 가능한 경우 HTTP 200 widget error frame으로 표현된다.

### 최소 구현

- 주석 처리된 ID parse와 lookup 흐름을 활성화한다.
- lookup map에는 초기 범위인 최상위 refresh-enabled custom-api widget만 등록한다.
- endpoint에서 Milestone 2의 update/render 경계를 호출한다.
- GET만 허용한다.
- 기존 인증 helper를 재사용하고 새로운 인증 체계는 만들지 않는다.

### 완료 조건

- endpoint를 반복 호출했을 때 `refresh < cache`여도 upstream 호출 수는 cache 정책을 따른다.
- 한 widget의 실패가 page 전체 endpoint를 실패시키지 않는다.

## Milestone 4 — 브라우저 polling

### 목표

브라우저가 refresh metadata가 있는 widget만 독립적으로 교체한다.

### 먼저 작성할 테스트

JavaScript에 이미 사용 중인 test runner가 없다면 새 framework를 도입하지 않는다. 가능한 작은 순수 함수 테스트 또는 Go가 렌더링한 markup 검증을 우선하고, timer 동작은 수동 smoke test로 확인한다.

- endpoint URL이 widget runtime ID로 만들어진다.
- 요청 완료 전 같은 widget 요청을 다시 시작하지 않는다.
- 200 응답은 해당 root만 교체한다.
- 404는 polling을 멈추고 reload 안내 상태를 표시한다.
- network error 뒤에는 같은 interval로 다음 시도를 예약한다.

### 최소 구현

- page 초기화 시 `[data-widget-refresh]` root를 찾아 각각 polling을 시작한다.
- `setInterval` 대신 fetch 완료 후 `setTimeout`을 예약한다.
- 응답의 첫 root가 `.widget`인지 확인한 뒤 현재 root를 교체한다.
- 교체된 root에서 새 metadata를 읽어 다음 polling을 계속한다.
- 범용 component reinitialization과 DOM 상태 복원은 구현하지 않는다.

### 수동 smoke test

- 서로 다른 2개 interval의 카드가 독립적으로 갱신된다.
- 페이지 전체가 깜박이거나 scroll 위치가 초기화되지 않는다.
- cache interval 안에서는 fake upstream 호출 횟수가 증가하지 않는다.
- server config reload 후 오래된 카드가 무한 오류 loop에 빠지지 않는다.

### 완료 조건

- read-only custom-api 카드의 텍스트 값이 page reload 없이 변한다.
- refresh 미지정 widget은 추가 network 요청을 만들지 않는다.

## Milestone 5 — 카드 내비게이션

### 목표

Overview 카드에서 내부 Glance detail page로 이동할 수 있게 한다.

### 먼저 작성할 테스트

- `href`가 있는 widget root에 navigation metadata가 렌더링된다.
- `href`가 없으면 기존 markup과 동작을 유지한다.
- 내부 link나 button을 누르면 card navigation이 실행되지 않는다.
- keyboard Enter로 같은 내부 경로에 이동할 수 있다.
- `base-url` 아래에서도 relative detail URL이 올바르게 해석된다.

### 최소 구현

- widget 공통 설정에 `href`를 추가한다.
- root에 URL, focus, 접근성 metadata를 추가한다.
- document-level delegated handler 하나로 click과 keyboard 이동을 처리한다.
- 초기에는 내부 relative URL과 같은 tab만 지원한다.
- 외부 URL은 기존 `title-url` 또는 template 내부 link를 사용한다.

### 완료 조건

- 카드의 빈 영역, 제목, 수치 영역을 눌러 detail page로 이동한다.
- 카드 안의 기존 interactive element는 원래 동작을 유지한다.

## Milestone 6 — 첫 번째 실제 수직 slice

### 목표

하나의 실제 read API로 Overview → detail → 외부 운영 UI 흐름을 완성해 구조 변경의 실용성을 확인한다.

첫 대상은 응답 형식과 인증이 이미 준비되어 있고 데이터 손실 가능성이 없는 source 중 가장 단순한 것을 선택한다. 일반적으로 HTTP health 또는 machine summary가 적합하다.

### 먼저 작성할 테스트

- 고정 fixture JSON이 healthy, warning, unavailable 상태로 렌더링된다.
- source timestamp와 Glance view refresh 시각을 혼동하지 않는다.
- malformed/unavailable 응답에서 카드 오류가 표시된다.

### 작업

- Overview custom-api 카드 한 개를 만든다.
- 대응하는 Glance detail page 한 개를 만든다.
- 외부 UI가 있으면 명시적 link를 둔다.
- 실제 응답의 단위, timestamp, threshold, cache, refresh를 source별 문서에 기록한다.

### 데이터 보호 gate

다음 중 하나라도 필요하면 작업을 멈추고 사용자에게 영향과 대안을 설명한 뒤 동의를 받는다.

- 기존 dashboard 설정의 덮어쓰기 또는 광범위한 재구성
- 운영 source의 schema·보존 데이터·credential 변경
- DB migration, queue purge, cache clear, volume 교체
- 기존 endpoint나 config key의 삭제·rename
- 실제 운영 데이터에 쓰기 요청을 보내는 검증

단순 read request, 새 파일 추가, 독립된 새 config 추가처럼 기존 데이터에 영향을 주지 않는 작업은 이 gate 대상이 아니다.

### 완료 조건

- 사용자가 실제 값의 변화와 stale/error 상태를 Overview에서 확인할 수 있다.
- 클릭으로 detail과 외부 UI까지 이동할 수 있다.
- Glance가 사용하는 credential은 read-only다.

## Milestone 7 — 도메인 카드 확장

### 목표

검증된 수직 slice 패턴을 실제 source에 반복 적용한다. 한 번에 한 source만 연결하고 다음 우선순위를 기본값으로 사용한다.

1. Machine Health
2. Data Pipeline
3. Public Web
4. Database Health
5. Backup
6. Paid API Usage/Cost

우선순위는 실제 endpoint 준비도에 따라 바꿀 수 있다. 각 source마다 다음 작은 cycle을 반복한다.

```text
fixture 작성
→ template rendering test
→ Overview 카드
→ detail page
→ 실제 read endpoint 연결
→ 짧은 smoke test
→ 다음 source
```

### source 착수 조건

- endpoint와 read credential이 있다.
- 대표 응답 예시가 있다.
- timestamp와 단위가 식별된다.
- warning 기준을 source 또는 dashboard template 중 어디서 판단할지 정해져 있다.
- detail URL과 외부 운영 UI URL이 알려져 있다.

조건이 빠졌다면 adapter나 template를 추측해서 만들지 않고 해당 source만 보류한다. 다른 준비된 source 작업은 계속할 수 있다.

### 완료 조건

- 각 카드가 정상·주의·오류 또는 stale 상태를 구분한다.
- Overview에는 핵심 count, worst value, age만 표시한다.
- 복잡한 history와 control은 detail 또는 외부 UI에 남는다.

## Milestone 8 — Attention과 Activity 조립

### 목표

개별 source에서 검증된 상태와 event를 Overview 상단의 Attention 및 Activity로 모은다.

### 초기 구현 선택

별도 영구 event store나 범용 metric schema는 만들지 않는다. 우선 각 source가 이미 제공하는 read endpoint를 custom-api subrequest로 읽고 template에서 다음 최소 envelope로 정규화한다.

```text
severity, source, summary, occurred_at, detail_url
```

source 수가 늘어 template가 유지하기 어려워질 때만 작은 aggregate adapter를 별도 작업으로 도입한다.

### 먼저 작성할 테스트

- fixture 여러 개가 severity 순으로 합쳐진다.
- source unavailable 자체가 Attention 항목으로 표시된다.
- event가 최신순으로 표시되고 최대 개수를 넘지 않는다.
- 항목 클릭이 올바른 detail page로 이동한다.

### 완료 조건

- Overview 첫 화면에서 중요한 이상을 source별 카드보다 먼저 볼 수 있다.
- Activity가 상태 저장소 역할을 하지 않고 source가 제공한 최근 event만 표시한다.

## Milestone 9 — 사용 후 보완 backlog

다음 항목은 지금 구현하지 않는다. 실제 불편이나 문제가 확인되면 각각 독립된 issue로 평가한다.

- built-in widget과 nested widget partial refresh
- fragment 교체 후 calendar, carousel, todo 등의 lifecycle 재초기화
- detail page의 global navigation 숨김 속성
- hidden tab이나 inactive group tab에서 polling 일시 정지
- exponential backoff, jitter, request cancellation 전파
- 안정적인 YAML widget ID
- 설정 reload 후 자동 page reload
- last-good snapshot의 영속 저장
- Attention aggregate service와 Activity event store
- browser별 자동화 회귀 테스트
- load, soak, penetration, failover test
- Tailscale identity 연동 또는 별도 권한 체계
- control action을 Glance 안에 추가하는 기능

이 backlog는 구현 약속이 아니다. 사용 중 얻은 증거가 있을 때만 우선순위를 부여한다.

## 예상 작업 단위

일정 대신 독립적으로 완료 가능한 작업 단위로 관리한다. 외부 source 준비 상태가 달라 달력 기준 추정은 정확하지 않기 때문이다.

| 작업 단위 | 주요 산출물 | 의존성 |
| --- | --- | --- |
| A | 기준선 fixture와 characterization tests | 없음 |
| B | `refresh` config와 markup | A |
| C | per-widget lock | A |
| D | partial endpoint | B, C |
| E | browser polling | D |
| F | card `href` | A |
| G | 첫 실제 수직 slice | E, F, 첫 read API |
| H1–H6 | 도메인별 카드와 detail page | G, 각 read API |
| I | Attention과 Activity | 필요한 H 작업 |

`B`와 `C`, `F`는 서로 독립적으로 진행할 수 있지만 한 작업에서 변경 범위를 작게 유지하기 위해 별도 commit/issue로 나누는 편이 좋다.

## 전체 완료 기준

초기 로드맵은 다음 상태에서 완료로 본다.

- Overview가 Attention과 핵심 운영 카드로 구성된다.
- refresh-enabled custom-api widget이 각자 독립적으로 갱신된다.
- backend cache와 browser refresh 주기가 분리되어 동작한다.
- 각 주요 카드는 Glance detail page 또는 외부 운영 UI로 이동한다.
- Overview와 detail page는 read-only다.
- 준비된 운영 source는 정상·주의·오류·stale을 구분한다.
- 소스 코드 단위 테스트, `go test ./...`, 핵심 browser smoke test가 통과한다.
- 보류된 source와 의도적으로 제외한 복잡한 검증 항목이 문서에 남아 있다.

전체 완료는 모든 잠재적 장애와 확장 요구를 선제적으로 해결했다는 뜻이 아니다. 현재 사용 범위에서 필요한 흐름이 단순하게 동작하고, 이후 문제를 별도 작업으로 다룰 수 있는 상태를 뜻한다.
