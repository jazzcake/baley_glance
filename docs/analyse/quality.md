# 테스트와 구현 품질 현황

## 분석 시 검증 결과

| 검사 | 결과 |
| --- | --- |
| `go build ./...` | 성공 |
| `go test ./...` | 성공 |
| `go test -race ./...` | 성공 |
| `go vet ./...` | 성공 |
| `go run . --config docs/glance.yml config:validate` | 성공 |
| 샘플 서버 `/` | HTTP 200, 7,939 bytes |
| 샘플 서버 `/api/healthz` | HTTP 200 |
| 샘플 서버 `/manifest.json` | HTTP 200, `application/json` |
| `gofmt -l .` | 2개 파일 보고 |

실행 검증은 샘플 설정의 page content API를 호출하지 않았다. 해당 호출은 외부 RSS, Twitch, YouTube, Reddit, Yahoo, GitHub API를 실제로 조회하기 때문이다. 브라우저 기반 시각 회귀 검증도 이 분석 범위에서는 수행하지 않았다.

## 테스트 구성

테스트 함수는 세 개다.

| 파일·테스트 | 검증 범위 |
| --- | --- |
| `auth_test.go: TestAuthTokenGenerationAndVerification` | secret 생성, token 발급·검증·재발급 시점·만료·각 byte 변조 탐지 |
| `address_test.go: TestAddressOfRequest` | proxied 여부, 단일/복수 X-Forwarded-For, spoofed leftmost, 공백과 빈 값 |
| `widget-shared_test.go: TestCalculateEngagementTimeDepreciation` | 게시물 engagement 시간 감쇠, 상한, 단조 감소 |

`internal/glance` statement coverage는 4.2%다. 주로 token crypto, client 주소, embed hash와 forum engagement만 실행된다. 다음 핵심 영역은 coverage 보고서에서 0%였다.

- CLI 파싱과 main dispatch
- 설정 변수·include·watcher·검증
- application 생성과 HTTP handler
- 인증 HTTP handler와 cookie
- theme 처리
- 모든 개별 widget initialize/update/render
- worker pool과 대부분 공통 helper

## CI 상태

GitHub Actions에는 tag 기반 release workflow만 있다. 일반 push나 pull request에서 build, test, race, vet, gofmt를 실행하는 workflow는 현재 없다. Release workflow 안에도 명시적인 test step은 없고 GoReleaser build가 수행된다.

## 포맷 상태

`gofmt -l .`은 다음 두 파일을 보고한다.

- `internal/glance/address_test.go`: 파일 끝 newline 누락
- `internal/glance/widget-search.go`: map literal의 `kagi`, `startpage` 값 정렬 불일치

`git diff --check`는 통과하므로 trailing whitespace 오류는 없다.

## 실제 재현된 결함

### `mountpoint:info` 명령 도달 불가

[../../internal/glance/cli.go](../../internal/glance/cli.go)에서 `len(args) == 2` 조건이 연속 두 번 사용된다. 첫 번째 분기는 `password:hash`가 아니면 즉시 unknown command를 반환하므로 두 번째 `mountpoint:info` 분기에 도달할 수 없다.

분석 시 다음 실행으로 재현했다.

```text
$ go run . mountpoint:info /
unknown command: mountpoint:info /
exit status 1
```

`cliMountpointInfo` 구현 자체는 존재하지만 parser가 해당 intent를 반환하지 않는다.

## 현재 구현상 명시된 미완성 경로

### Widget request API

`/api/widgets/{widget}/{path...}`와 widget interface의 `handleRequest`가 있지만 application handler는 즉시 501을 반환한다. ID parse, map lookup, 위젯 위임 코드는 주석 처리되어 있다. 주석은 페이지 전체가 아닌 위젯별 잠금으로 갱신 모델을 바꿔야 한다고 설명한다.

### Server Stats

`server-stats`는 initialize 시 `WIP = true`를 설정한다. update 코드에도 feedback에 따라 대부분 바뀔 수 있다는 주석이 있다.

### 이전 호환 코드

- Docker config 위치 안내 서버는 v0.10 제거 예정으로 표시되어 있다.
- Markets의 `stocks` 설정 호환도 v0.10 제거 예정이다.

## 코드에 기록된 주요 TODO

| 영역 | 현재 주석의 내용 |
| --- | --- |
| 서버 재로드 | callback과 동시 작업 때문에 이해하기 어려우며 단일 goroutine/channel 구조를 언급 |
| 설정 watcher | 두 위치가 `flaky`로 표시됨 |
| 설정 검증 | 논리 검증과 application 생성 검증이 나뉨 |
| 설정 변수 | YAML 주석 안의 변수 패턴도 매치함 |
| 404 | 전용 not-found 페이지 없음 |
| widget API | 위젯별 잠금 모델 필요 |
| widget 오류 | 최초와 재렌더 모두 실패할 때 generic 오류 template 없음 |
| retry 정책 | partial failure 종류에 따른 조기 갱신 판단이 단순함 |
| Repository | 일부 오류를 덮어쓰는 구간 표시 |
| Twitch Channels | 채널별 요청 대신 batch 가능하다고 표시 |
| widget utils | JSON/XML decoder 중복 |
| Calendar JS | spillover 현재 날짜와 다른 달 auto-advance 동작 |
| page fetch | non-200, timeout, retry 미처리 |
| dynamic import | calendar prefetch 없이 waterfall 발생 |
| CSS | 일부 dynamic column과 weather layout 구간 refactor 표시 |

## 정적 분석으로 확인한 동작상 제약

다음은 추측이 아니라 현재 제어 흐름에서 직접 확인되는 특성이다.

- 첫 페이지도 JavaScript가 콘텐츠 API를 성공적으로 호출해야 위젯이 표시된다.
- 한 페이지의 가장 느린 갱신 위젯이 전체 content API 응답 시간을 결정한다.
- 같은 페이지 콘텐츠 요청은 page mutex 때문에 순차 처리된다.
- Extension은 timeout 없는 `http.DefaultClient`를 사용한다.
- frontend content fetch는 status code, timeout, retry와 catch UI를 처리하지 않는다.
- duplicate page slug를 검증하지 않아 뒤 page가 routing map을 덮어쓴다.
- YAML unknown field를 거부하지 않는다.
- 설정 재로드는 기존 server에 graceful `Shutdown`이 아니라 `Close`를 호출한다.
- health check는 외부 의존성이나 widget 상태를 보지 않는다.
- 개별 외부 응답 body의 크기 제한이 대부분 없다.
- 인증 실패 rate limit과 모든 widget cache는 process/application 메모리 전용이다.
- root `glance.yml`이 없어 이 checkout에서 옵션 없는 `go run .`은 설정 파일 읽기 오류가 된다.

## 패키지 구조의 응집도

도메인 코드 대부분이 `internal/glance` 단일 package에 있다. 장점으로는 unexported type과 helper를 위젯 간 쉽게 공유하고 한 binary로 단순하게 build할 수 있다. 현재 코드 형태에서 나타나는 결합은 다음과 같다.

- package 초기화 시 모든 주요 template과 CSS bundle을 parse한다. 오류는 panic으로 build artifact 실행 직후 드러난다.
- 개별 widget이 HTTP request, API response type, domain 변환, template 선택을 한 파일에서 함께 담당한다.
- 전역 HTTP client, feed parser, static hash, template, widget ID counter를 package 전체에서 공유한다.
- `application.Config`와 page/widget 구조가 template에 직접 노출된다.
- 별도 service/repository 계층 없이 handler가 page와 widget 메서드를 직접 호출한다.

## 문서와 소스의 차이

[../../README.md](../../README.md)의 source build 요구사항은 Go `>= 1.23`이라고 적혀 있지만 [../../go.mod](../../go.mod)과 [../../Dockerfile](../../Dockerfile)은 정확히 `1.27.1`을 사용한다. 현대 Go의 toolchain 자동 취득이 활성화된 환경에서는 1.27.1을 내려받아 build할 수 있지만, 문서의 최소 버전 표현과 module directive는 서로 다른 정보를 준다.

README의 개발 실행 예시는 `go run .`이지만 이 repository root에는 기본 `glance.yml`이 없고 샘플은 `docs/glance.yml`에 있다.

## 분석 재현 정보

분석에는 다음 읽기·검증 수단을 사용했다.

- `rg`로 type, function, route, TODO, URL, template 연결 조사
- `go list -m all`로 resolved dependency 확인
- `go test -coverprofile`과 `go tool cover -func`
- build, test, race, vet, gofmt
- 샘플 config validate
- 샘플 서버의 root, health, manifest smoke request
- Git branch, remote, commit graph, 작업 트리 확인

분석 문서 작성 외에 애플리케이션 source와 기존 문서는 수정하지 않았다.

