# 인증과 보안 경계

이 문서는 현재 구현의 보안 관련 동작과 신뢰 경계를 설명한다. 취약점 판정이나 운영 정책 제안이 아니라 코드가 실제로 수행하는 검증과 예외를 기록한다.

## 인증 활성화

`auth.users`가 한 명 이상이면 application이 인증 필요 상태가 된다. 사용자가 없으면 login·logout·authenticate 라우트 자체를 등록하지 않고 페이지와 콘텐츠 API를 공개한다.

각 사용자는 다음 둘 중 하나를 가진다.

- `password`: application 생성 시 bcrypt 기본 cost로 hash하고 구조체의 평문 필드를 빈 문자열로 바꾼다.
- `password-hash`: 문자열을 byte slice로 옮기고 원래 문자열 필드를 비운다.

username은 최소 3자, 평문 password는 최소 6자다. 로그인 요청에서는 username 최대 50자, password 최대 100자로 제한한다.

## Secret과 세션 토큰

`secret-key`는 Base64 decode 결과가 정확히 64바이트여야 한다. `secret:make` 명령이 이 형식의 값을 만든다. 내부적으로 앞 32바이트와 뒤 32바이트를 분리해 사용한다.

```text
64-byte secret
├─ first 32 bytes: token HMAC-SHA256 key
└─ last 32 bytes: username HMAC-SHA256 key
```

세션 토큰의 binary payload는 다음 구조다.

```text
username HMAC (32 bytes)
expiration Unix timestamp, uint32 little endian (4 bytes)
payload HMAC-SHA256 signature (32 bytes)
```

총 68바이트를 표준 Base64로 encode한다. username 자체는 토큰에 들어가지 않고 secret으로 HMAC한 32바이트 값만 들어간다. application은 시작할 때 모든 username hash를 map에 만들어 검증된 hash를 실제 username에 역매핑한다.

- 토큰 유효기간: 14일
- 남은 유효기간이 7일 미만이면 요청 처리 중 새 토큰 발급
- 서명 비교: `hmac.Equal`
- 서버 측 세션 저장소나 revoke 목록: 없음
- secret 또는 사용자 구성이 바뀌어 application이 재로드되면 기존 토큰은 새 구성에 따라 검증됨

## 세션 쿠키

| 속성 | 값 |
| --- | --- |
| 이름 | `session_token` |
| HttpOnly | true |
| SameSite | Lax |
| Path | `base-url + "/"` |
| Expires | 발급 시점 + 14일 |
| Secure | `X-Forwarded-Proto`가 대소문자 무시 `https`일 때만 true |

애플리케이션 자체에 TLS listener가 없으므로 일반적으로 reverse proxy가 HTTPS 종료와 `X-Forwarded-Proto` 전달을 담당한다.

## 로그인 처리

`POST /api/authenticate`의 순서는 다음과 같다.

1. `Content-Type`이 정확히 `application/json`인지 검사
2. 요청 주소별 실패 기록을 조회하고 rate limit 검사
3. body를 최대 512 KiB로 제한해 읽기
4. JSON decode와 길이 검증
5. username map 조회
6. bcrypt 비교
7. 세션 토큰과 cookie 발급
8. 해당 주소의 실패 기록 삭제

실패 응답에는 500–999ms의 무작위 지연이 들어간다. 사용자 없음과 password 불일치를 모두 401로 응답한다. 실패 로그에는 시도한 username과 계산된 client 주소가 기록된다.

## Rate limit과 proxy 주소

- 기간: 5분
- 최대 시도 기록: 5회
- 이후 응답: 429와 초 단위 `Retry-After`
- 저장: application 내부 map, mutex 보호
- 정리: 새 로그인 요청이 들어왔을 때 5분보다 오래된 항목 삭제

`server.proxied`가 false이면 `RemoteAddr`에서 마지막 colon 앞을 주소로 사용한다. true이면 `X-Forwarded-For`의 가장 오른쪽 값을 사용한다. 이는 신뢰한 reverse proxy가 마지막에 추가한 주소를 택하고 client가 만든 왼쪽 값을 피하려는 구현이다. proxy가 헤더를 적절히 덮어쓰거나 추가한다는 운영 전제가 있다.

## 인증 적용 범위

인증 검사는 page document와 page content API에 적용된다. 정적 자산, manifest, health check, theme 변경은 공개다. widget API도 인증 검사 없이 등록되지만 현재 항상 501을 반환한다.

- 미인증 page document 요청: 303으로 `base-url/login` 이동
- 미인증 content API 요청: 401과 `{"error": "Unauthorized"}` body
- logout: GET 요청으로 cookie를 과거 시각에 만료시킨 뒤 login으로 이동

별도 CSRF token은 없다. 세션 cookie의 SameSite=Lax가 기본 cross-site cookie 전달을 제한한다. theme cookie 변경은 인증 없이 호출 가능하지만 응답을 받는 client 자신의 theme cookie만 바꾼다.

## 설정과 secret 취급

민감 값은 YAML 평문 외에도 다음 경로로 불러올 수 있다.

- 환경변수 `${NAME}`
- Docker secret `${secret:name}`
- 환경변수가 지시하는 절대 파일 `${readFileFromEnv:NAME}`

치환된 값은 config와 위젯 구조체의 문자열로 메모리에 남는다. 로그는 일반 외부 요청 오류에 URL과 일부 응답 body를 포함할 수 있지만 request header나 설정 token을 의도적으로 출력하지 않는다. URL query에 secret을 넣으면 오류 URL에 나타날 수 있다.

## HTML과 브라우저 신뢰 경계

일반 Go template 값은 context에 맞게 escape된다. 다음 경로는 의도적으로 이 보호를 우회할 수 있다.

- `document.head`
- `branding.custom-footer`
- `html` 위젯의 `source`
- 공통 template function `safeHTML`, `safeURL`, `safeCSS`
- Extension의 `allow-potentially-dangerous-html: true`
- Custom API template이 `safeHTML`을 사용하는 경우

이 값들은 config 작성자를 신뢰하는 설계다. 특히 Extension과 Custom API에서 외부 응답을 raw HTML로 승격하면 외부 서비스도 같은 신뢰 수준을 갖게 된다.

페이지 콘텐츠는 fetch 결과를 브라우저의 `innerHTML`에 넣는다. 정상 콘텐츠는 서버 template에서 생성되지만 HTTP status를 확인하지 않으므로 proxy나 서버가 반환한 오류 body도 동일한 삽입 경로를 사용한다.

## 서버 측 외부 요청 경계

다음 위젯은 설정된 임의 URL 또는 socket으로 서버 측 요청을 보낼 수 있다.

- Custom API와 동적 `getResponse`
- Extension
- Monitor
- RSS
- Lobsters custom/instance URL
- ChangeDetection instance
- DNS Stats instance
- Docker socket 또는 tcp/http/https source
- remote Server Stats

이는 config 작성자가 서버가 접근 가능한 내부 네트워크와 Unix socket에 접근할 수 있음을 의미한다. HTTP redirect는 Go client 기본 정책을 따른다. 응답 body 크기를 제한하지 않고 `io.ReadAll`하는 경로가 다수 존재한다.

## TLS 검증 예외

`allow-insecure`가 있는 Custom API, Monitor, DNS Stats 및 proxy 설정은 `tls.Config.InsecureSkipVerify`를 활성화할 수 있다. 기본값은 검증 활성화다. Docker의 HTTPS source는 별도 allow-insecure 설정 없이 Go 기본 TLS 검증을 사용한다.

Extension은 `http.DefaultClient`를 사용해 기본 TLS 검증을 수행하지만 client timeout을 설정하지 않는다. iframe과 외부 icon은 브라우저가 직접 요청하므로 브라우저 보안 정책과 대상 서버의 header가 적용된다.

## HTTP 보안 header

서버 구현은 CSP, HSTS, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, frame-ancestors/X-Frame-Options를 별도로 설정하지 않는다. 일부 응답은 Content-Type을 명시하고 페이지 응답은 `net/http`의 content sniffing에 맡긴다.

