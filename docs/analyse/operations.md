# 빌드, 배포와 운영 특성

## 도구 체인과 산출물

[../../go.mod](../../go.mod)은 모듈을 `github.com/glanceapp/glance`, Go 버전을 `1.27.1`로 선언한다. 루트 package 하나를 build하면 템플릿과 정적 자산이 포함된 단일 실행 파일이 생성된다.

```text
go build .
```

`buildVersion` 기본값은 `dev`다. GoReleaser는 link flag `-X github.com/glanceapp/glance/internal/glance.buildVersion={{ .Tag }}`로 release tag를 주입한다.

내장 자산과 순수 Go 의존성 덕분에 release build는 `CGO_ENABLED=0`을 사용한다. 별도 frontend build, code generation, migration, asset download 단계가 없다.

## 직접 의존성

| 모듈 | 구현 역할 |
| --- | --- |
| `fsnotify` | main/include config 파일 감시 |
| `gofeed` | RSS/Atom 파싱 |
| `utls` | Reddit 호출의 브라우저형 TLS client hello |
| `gopsutil/v4` | CPU, memory, disk, sensor, host 정보 |
| `gjson` | Custom API JSON path와 typed 접근 |
| `x/crypto` | bcrypt password hash |
| `x/net` | Reddit HTTP/2 transport 등 네트워크 기능 |
| `x/text` | 영문 숫자 formatting |
| `yaml.v3` | 설정 decode와 custom field unmarshalling |

나머지 HTTP, template, embed, crypto/HMAC, concurrency 기능은 Go 표준 라이브러리를 사용한다.

## Docker 이미지

[../../Dockerfile](../../Dockerfile)은 두 단계 image다.

1. `golang:1.27.1-alpine3.24.1`에서 `CGO_ENABLED=0 go build .`
2. `alpine:3.24.1`에 `/app/glance`만 복사

container entrypoint는 다음과 같다.

```text
/app/glance --config /app/config/glance.yml
```

8080/tcp를 expose하며 config나 assets volume은 Dockerfile에 선언하지 않는다. `USER`도 선언하지 않아 기본 root 사용자로 실행된다. `.dockerignore`가 build context 제외 항목을 제어한다.

## Release 자동화

[../../.github/workflows/release.yaml](../../.github/workflows/release.yaml)은 `v*` tag push에서만 실행된다.

- checkout full history
- Docker Hub login
- `go.mod` 기준 Go setup
- Docker buildx setup
- GoReleaser 실행

[../../.goreleaser.yaml](../../.goreleaser.yaml)은 다음 대상을 정의한다.

- OS: Linux, OpenBSD, FreeBSD, Windows, macOS
- architecture: amd64, arm64, armv7, 386 일부 조합
- archive: 일반 tar 계열, Windows zip
- Docker: amd64, arm64, armv7 개별 image 후 multi-architecture manifest
- image name: `glanceapp/glance:{tag}`와 조건부 `latest`
- checksum: 비활성화

현재 repository에는 release 외에 test/lint 전용 workflow가 없다.

## 설정과 filesystem

프로세스가 읽는 외부 filesystem 상태는 다음과 같다.

- main YAML 및 include 파일
- `/run/secrets/*` 또는 환경변수로 지정한 secret 파일
- 선택적 `server.assets-path`
- 선택적 Docker Unix socket
- server-stats가 조회하는 local mountpoint와 system interface

애플리케이션 데이터 파일을 쓰는 로직은 없다. 모든 서버 캐시는 메모리에 있고 To-do만 브라우저 localStorage에 기록된다.

## 네트워크와 reverse proxy

- 기본 bind: 빈 host의 port 8080, 즉 플랫폼상 모든 interface
- application TLS: 없음
- 외부 prefix: `base-url`로 생성 URL에 반영
- proxy client IP: `proxied: true`에서 오른쪽 `X-Forwarded-For`
- secure cookie: `X-Forwarded-Proto: https`에 의존
- outbound proxy: 공통 HTTP transport는 `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`를 따름

`base-url` 경로를 application mux가 직접 strip하지 않으므로 reverse proxy가 외부 prefix를 제거한 내부 경로로 전달해야 한다.

## Health와 진단

`GET /api/healthz`는 application route가 동작하면 항상 200을 반환한다. config 유효성이나 외부 서비스, widget cache, filesystem, auth 상태를 추가로 확인하지 않으므로 liveness 성격이다.

`diagnose` CLI는 다음 검사를 goroutine으로 동시에 수행한다.

- Cloudflare·Google DNS over HTTPS
- 일반 DNS로 GitHub, Reddit, Twitch 해석
- YouTube feed
- Twitch GraphQL
- GitHub API
- Open-Meteo
- Reddit API
- Yahoo Finance
- Hacker News Firebase
- Docker Hub

각 검사는 context 15초를 만들지만 실제 client의 기본 5초 timeout이 더 짧게 적용된다. 결과에는 성공 여부, 일부 body 정보, 소요 ms가 Markdown code block 형태로 출력된다.

## 로그와 관찰 가능성

두 로깅 API를 혼용한다.

- `log.Printf`: 서버 시작·재로드·watcher·인증 실패 등 application 수준 이벤트
- `slog.Error/Warn`: widget 외부 요청·파싱·시스템 정보 수집 오류

별도 logger 설정이 없어 표준 출력/표준 오류의 기본 text 형식을 사용한다. request access log, structured request ID, metric endpoint, tracing, cache hit 지표는 없다. Widget 오류는 로그 외에도 widget frame의 Error/Notice로 사용자에게 표시된다.

## 성능과 자원 특성

- 정상 캐시 hit에서는 page content가 메모리 모델을 template으로 렌더링하는 작업만 수행한다.
- cache miss에서는 같은 페이지의 만료 위젯을 모두 병렬 실행하지만 응답은 가장 늦은 위젯까지 기다린다.
- 같은 페이지의 content 요청은 page mutex로 직렬화된다. 다른 페이지는 별도 mutex이므로 병렬이다.
- 일부 위젯의 내부 worker 수가 높아 한 페이지 첫 요청에서 다수의 outbound connection이 동시에 만들어질 수 있다.
- 기본 외부 client timeout은 5초지만 Extension은 timeout이 없다.
- server에는 WriteTimeout이 없어 느리거나 끝나지 않는 widget 갱신에 대한 응답 쓰기 상한이 없다.
- JSON/XML과 다수 외부 body는 크기 제한 없이 메모리로 전부 읽는다.
- CSS bundle과 템플릿은 package 초기화 시 준비되고 이후 재사용된다.

## 플랫폼 특성

gopsutil 기반 Server Stats는 운영체제별 차이가 있다.

- Windows CPU load는 core count로 나누지 않는 별도 계산을 사용한다.
- Windows와 BSD 계열에서는 CPU 온도 수집을 비활성화한다.
- Linux의 Intel/AMD/Raspberry Pi에 대해 알려진 sensor key를 자동 추론한다.
- mountpoint 목록과 filesystem 정보는 OS가 반환하는 값에 따른다.

GoReleaser 대상에는 Linux, BSD, Windows, macOS가 포함되지만 Docker image는 Linux amd64/arm64/armv7만 생성한다.

## 라이선스

[../../LICENSE](../../LICENSE)는 GNU Affero General Public License v3 전문이다. 이 repository의 source와 network server 사용 형태는 해당 라이선스 범위 안에 있다.

