# 소스 코드 지도

이 문서는 현재 파일을 책임 단위로 빠르게 찾기 위한 지도다. 세부 제어 흐름은 각 주제 문서를 함께 참조한다.

## 루트와 빌드 파일

| 파일 | 책임 |
| --- | --- |
| [../../main.go](../../main.go) | `glance.Main()` 호출과 process exit code 전달 |
| [../../go.mod](../../go.mod) | module path, Go version, 직접·간접 의존성 |
| [../../Dockerfile](../../Dockerfile) | 소스 build와 Alpine runtime image |
| [../../Dockerfile.goreleaser](../../Dockerfile.goreleaser) | GoReleaser가 만든 binary를 image에 넣는 runtime Dockerfile |
| [../../.goreleaser.yaml](../../.goreleaser.yaml) | cross-platform binary, archive, Docker manifest 정의 |
| [../../.github/workflows/release.yaml](../../.github/workflows/release.yaml) | `v*` tag 기반 release job |
| [../../.dockerignore](../../.dockerignore) | Docker build context 제외 규칙 |
| [../../.gitignore](../../.gitignore) | Git 제외 규칙 |

## 애플리케이션 코어

| 파일 | 책임과 주요 타입·함수 |
| --- | --- |
| [../../internal/glance/main.go](../../internal/glance/main.go) | CLI intent dispatch, `serveApp`, config reload, 이전 Docker config 안내 server |
| [../../internal/glance/cli.go](../../internal/glance/cli.go) | flag/command parser, sensor와 mountpoint CLI 출력 |
| [../../internal/glance/glance.go](../../internal/glance/glance.go) | `application`, page 초기화, widget 갱신, handler, route와 `http.Server` 구성 |
| [../../internal/glance/config.go](../../internal/glance/config.go) | config/page model, 변수·include, watcher, 논리 검증, ordered YAML map |
| [../../internal/glance/config-fields.go](../../internal/glance/config-fields.go) | HSL, duration, icon, proxy, query parameter custom YAML field |
| [../../internal/glance/auth.go](../../internal/glance/auth.go) | token crypto, login rate limit, authorization, cookie, login/logout handler |
| [../../internal/glance/theme.go](../../internal/glance/theme.go) | theme model, CSS/preview compile, picker POST handler |
| [../../internal/glance/embed.go](../../internal/glance/embed.go) | static/template embed.FS, static hash, CSS import flattening과 minification |
| [../../internal/glance/templates.go](../../internal/glance/templates.go) | 공통 template FuncMap, parse helper, number/time HTML helper |
| [../../internal/glance/utils.go](../../internal/glance/utils.go) | URL·문자열·시간·숫자·색상 helper, cached file server, template 실행 helper |
| [../../internal/glance/diagnose.go](../../internal/glance/diagnose.go) | DNS·외부 API 병렬 진단 명령 |
| [../../internal/glance/singleflight.go](../../internal/glance/singleflight.go) | generic 단일 동시 호출 병합 |

## 위젯 기반 코드

| 파일 | 책임 |
| --- | --- |
| [../../internal/glance/widget.go](../../internal/glance/widget.go) | widget factory/interface/base, cache와 오류·재시도 모델 |
| [../../internal/glance/widget-container.go](../../internal/glance/widget-container.go) | 컨테이너 자식의 초기화·병렬 갱신·provider 전달 |
| [../../internal/glance/widget-utils.go](../../internal/glance/widget-utils.go) | HTTP client와 JSON/XML decoder, cached entry, generic worker pool |
| [../../internal/glance/widget-shared.go](../../internal/glance/widget-shared.go) | Twitch endpoint/client ID, forum post model과 engagement 계산 |

## 개별 위젯 파일

| 파일 | type과 책임 |
| --- | --- |
| `widget-bookmarks.go` | `bookmarks`: 그룹·링크 상속과 정적 렌더링 |
| `widget-calendar.go` | `calendar`: 브라우저 calendar용 shell과 요일 검증 |
| `widget-old-calendar.go` | `calendar-legacy`: 서버 측 21일 calendar 계산 |
| `widget-clock.go` | `clock`: timezone 검증과 정적 clock markup |
| `widget-weather.go` | `weather`: Open-Meteo 장소·예보 조회와 날씨 코드 변환 |
| `widget-search.go` | `search`: engine URL, bang 검증과 검색 UI |
| `widget-todo.go` | `to-do`: browser todo component용 shell |
| `widget-html.go` | `html`: 설정의 raw HTML 반환 |
| `widget-iframe.go` | `iframe`: source와 height 검증·렌더링 |
| `widget-rss.go` | `rss`: feed 병렬 취득, ETag cache, item 정규화와 네 layout |
| `widget-videos.go` | `videos`: YouTube feed 병렬 취득, channel cache와 세 layout |
| `widget-reddit.go` | `reddit`: 공개/OAuth API, proxy·uTLS, post·thumbnail 변환 |
| `widget-hacker-news.go` | `hacker-news`: story ID와 item 병렬 수집 |
| `widget-lobsters.go` | `lobsters`: Lobsters 호환 JSON feed 수집 |
| `widget-twitch-channels.go` | `twitch-channels`: persisted GraphQL channel metadata |
| `widget-twitch-top-games.go` | `twitch-top-games`: persisted GraphQL directory 조회 |
| `widget-markets.go` | `markets`/`stocks`: Yahoo Finance와 sparkline model |
| `widget-releases.go` | `releases`: GitHub/GitLab/Codeberg/Docker Hub release 통합 |
| `widget-repository.go` | `repository`: GitHub 상세·PR·issue·commit 집계 |
| `widget-changedetection.go` | `change-detection`: watch 목록·상세와 diff URL |
| `widget-monitor.go` | `monitor`: HTTP status와 latency 병렬 검사 |
| `widget-docker-containers.go` | `docker-containers`: Docker API, label override, parent/child 구성 |
| `widget-dns-stats.go` | `dns-stats`: 네 DNS 제품의 API adapter와 공통 view model |
| `widget-server-stats.go` | `server-stats`: local sysinfo 또는 remote agent 응답 통합 |
| `widget-custom-api.go` | `custom-api`: request model, GJSON wrapper, template 함수와 렌더링 |
| `widget-extension.go` | `extension`: Widget header protocol과 HTML trust option |
| `widget-group.go` | `group`: tab container와 중첩 제약 |
| `widget-split-column.go` | `split-column`: 내부 column container |

표의 개별 위젯 파일은 모두 [../../internal/glance](../../internal/glance) 디렉터리에 있다.

## 시스템 정보

| 파일 | 책임 |
| --- | --- |
| [../../pkg/sysinfo/sysinfo.go](../../pkg/sysinfo/sysinfo.go) | host/boot, CPU load·온도, memory/swap, disk mountpoint 수집과 JSON model |

`pkg/sysinfo`는 `internal/glance`와 분리된 유일한 application package다. 원격 agent가 사용할 수 있도록 JSON/YAML model 일부가 export되어 있다.

## 테스트 파일

| 파일 | 범위 |
| --- | --- |
| [../../internal/glance/auth_test.go](../../internal/glance/auth_test.go) | session token 생성·검증·변조·만료 |
| [../../internal/glance/address_test.go](../../internal/glance/address_test.go) | client address와 proxy header |
| [../../internal/glance/widget-shared_test.go](../../internal/glance/widget-shared_test.go) | forum engagement 감쇠 |

## 템플릿

### 문서와 page

| 파일 | 책임 |
| --- | --- |
| `document.html` | HTML document skeleton, meta, CSS, PWA link, `pageData` |
| `page.html` | navigation, mobile controls, content shell, loading UI |
| `page-content.html` | head widget과 columns 반복 렌더링 |
| `login.html` | login form |
| `footer.html` | 기본·사용자 footer |
| `manifest.json` | branding 기반 web manifest |
| `v0.7-update-notice-page.html` | 이전 Docker config 위치 안내 |

### 공통·테마

| 파일 | 책임 |
| --- | --- |
| `widget-base.html` | frame/header/error/notice/WIP 공통 구조 |
| `forum-posts.html` | Hacker News·Lobsters·기본 Reddit 공통 목록 |
| `video-card-contents.html` | video card 공유 조각 |
| `theme-style.gotmpl` | theme properties를 CSS 변수로 변환 |
| `theme-preset-preview.html` | picker preview markup |

그 밖의 template은 대부분 같은 이름의 widget 파일이 선택한다. RSS, Videos, Reddit, Monitor는 style에 따라 여러 template 중 하나를 사용한다.

## JavaScript와 CSS

JavaScript 파일별 책임은 [frontend.md](frontend.md#javascript-모듈)에 정리되어 있다. CSS는 `main.css`가 entrypoint이며 `site`, `widgets`, `popover`, `utils`, `mobile`을 가져온다. `widgets.css`는 개별 `widget-*.css`를 가져온다.

정적 파일과 template은 [../../internal/glance/embed.go](../../internal/glance/embed.go)의 두 `//go:embed` 선언에 의해 실행 파일에 포함된다. 소스 tree의 template/CSS/JS를 수정한 결과는 Go binary를 다시 build하거나 `go run`할 때 반영된다. 사용자 `assets-path`만 embed 바깥의 runtime filesystem을 읽는다.

## 사용자 문서

| 파일 | 범위 |
| --- | --- |
| [../configuration.md](../configuration.md) | 전체 YAML schema와 widget 사용법 |
| [../custom-api.md](../custom-api.md) | Custom API template 예제와 함수 |
| [../extensions.md](../extensions.md) | Extension response header protocol |
| [../themes.md](../themes.md) | theme preset |
| [../preconfigured-pages.md](../preconfigured-pages.md) | page 구성 예시 |
| [../v0.7.0-upgrade.md](../v0.7.0-upgrade.md) | Docker config 위치 migration |
