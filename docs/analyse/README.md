# Glance 구현 분석

이 디렉터리는 현재 저장소에 체크아웃된 Glance 구현을 정적 분석하고, 실제 빌드·테스트·기동 결과와 대조해 기록한 문서다. 신규 기능 설계나 향후 개발 방향은 범위에 포함하지 않는다.

## 분석 기준

- 저장소: `https://github.com/jazzcake/baley_glance.git`
- 브랜치와 커밋: `main`, `372466c6d75318670dc66e4e452179350fc50c97`
- 분석일: 2026-09-16
- Go 모듈: `github.com/glanceapp/glance`
- Go 도구 체인: `go 1.27.1`
- 작업 트리: 분석 시작 시 변경 없음
- 로컬 태그: 없음. 따라서 소스 자체의 릴리스 버전은 특정할 수 없으며, 일반 빌드의 `buildVersion`은 `dev`다.

## 문서 구성

| 문서 | 내용 |
| --- | --- |
| [architecture.md](architecture.md) | 시스템 경계, 구성 요소, 메모리 상태, 동시성, 코드 지도 |
| [runtime-http.md](runtime-http.md) | CLI, 시작·재로드 생명주기, HTTP 라우트, 페이지 렌더링, 정적 자산 |
| [configuration.md](configuration.md) | YAML 처리 순서, include·변수, 검증 규칙, 테마·브랜딩 |
| [widgets.md](widgets.md) | 위젯 계약, 캐시·오류 모델, 병렬 처리, 전체 위젯 카탈로그 |
| [frontend.md](frontend.md) | 템플릿, 브라우저 부트스트랩, JS 모듈, CSS 번들, 클라이언트 상태 |
| [security.md](security.md) | 인증 토큰, 쿠키, rate limit, 신뢰 경계, 네트워크 보안 속성 |
| [operations.md](operations.md) | 빌드·배포, 의존성, 상태 저장, 로그·진단·헬스체크, 성능 특성 |
| [quality.md](quality.md) | 테스트·커버리지·CI 현황, 재현된 결함, 구현상 제약과 TODO |
| [source-map.md](source-map.md) | Go·template·JavaScript·CSS·빌드 파일별 책임 지도 |
| [requirements-analysis.md](requirements-analysis.md) | 로컬 운영 대시보드 요구안의 정규화, 현 구현 적합성, 간극과 결정 항목 |
| [implementation-roadmap.md](implementation-roadmap.md) | 최소 검증·테스트 우선 원칙에 따른 단계별 구현 순서와 완료 기준 |

기존 사용자용 설정 레퍼런스는 [../configuration.md](../configuration.md), 확장 프로토콜은 [../extensions.md](../extensions.md), Custom API 템플릿 레퍼런스는 [../custom-api.md](../custom-api.md)에 있다. 이 문서 세트는 그 사용법을 반복하지 않고 내부 구현을 설명한다.

## 요약

Glance는 YAML로 페이지와 위젯을 선언하는 단일 프로세스 대시보드다. Go 표준 HTTP 서버가 HTML 템플릿을 렌더링하고, 위젯 데이터는 요청 시 외부 서비스에서 수집되어 프로세스 메모리에 캐시된다. 프론트엔드는 빌드 도구 없는 ES 모듈과 CSS로 구성되며 모든 자산은 `embed.FS`를 통해 바이너리에 포함된다.

초기 페이지 응답은 내비게이션과 빈 콘텐츠 컨테이너를 제공한다. 브라우저가 별도의 페이지 콘텐츠 API를 호출하면 서버가 해당 페이지의 만료된 위젯을 병렬 갱신하고 완성된 HTML 조각을 반환한다. 서버 측 주기 작업, 데이터베이스, 메시지 큐, 별도 프론트엔드 빌드 단계는 없다.

## 검증한 명령

아래 항목은 분석 당시 실제로 실행했다.

```text
go build ./...                                      성공
go test ./...                                       성공
go test -race ./...                                 성공
go vet ./...                                        성공
go run . --config docs/glance.yml config:validate   성공
```

샘플 설정으로 서버를 기동해 `/`, `/api/healthz`, `/manifest.json`이 모두 HTTP 200을 반환하는 것도 확인했다. 테스트 커버리지는 `internal/glance` 기준 statement 4.2%였다. 자세한 내용은 [quality.md](quality.md)에 기록한다.
