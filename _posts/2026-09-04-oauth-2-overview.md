---
layout: single
title: "OAuth 2.0 핵심 이해: 권한 위임의 원리와 사용 예제"
categories:
  - "컴퓨터 기술"
permalink: /topics/computer-technology/oauth-2-overview/
tags:
  - "OAuth 2.0"
  - "인증"
  - "인가"
  - "웹 보안"
  - "API"
toc: true
author_profile: true
sidebar:
  nav: "docs"
---

사용자가 일정 관리 앱에 자신의 캘린더 일부를 읽을 권한만 주고 싶다고 가정해 보겠습니다. 가장 단순하지만 위험한 방법은 캘린더 서비스의 아이디와 비밀번호를 일정 앱에 직접 전달하는 것입니다. 이렇게 하면 일정 앱이 필요 이상의 권한을 갖게 되고, 접근을 취소하려면 비밀번호까지 바꿔야 하며, 앱이 침해됐을 때 사용자 계정 전체가 위험해집니다.

**OAuth 2.0**은 비밀번호를 제3자 애플리케이션에 주지 않고도 특정 자원에 대한 제한된 접근 권한을 위임하기 위한 authorization framework입니다. 사용자는 자신이 신뢰하는 권한 부여 서버에서 동의하고, 애플리케이션은 그 결과로 발급받은 access token을 사용해 API에 접근합니다.

이 글은 OAuth 2.0이 필요한 배경, 구성 요소, token과 scope, 현재 권장되는 Authorization Code + PKCE 흐름과 간단한 HTTP 예제를 정리합니다.

이 글은 **2026년 9월 기준** [OAuth 2.0 RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html), [PKCE RFC 7636](https://www.rfc-editor.org/rfc/rfc7636.html)과 최신 보안 권고인 [RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html)을 기준으로 작성했습니다. OAuth 2.1은 현재도 [IETF Internet-Draft](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/) 단계이므로 확정된 RFC처럼 표현해서는 안 됩니다.
{: .notice--warning}

## OAuth 2.0이 해결하는 문제

OAuth 이전의 비밀번호 공유 방식에는 다음 문제가 있었습니다.

- 제3자 앱이 사용자의 전체 계정 권한을 갖습니다.
- 앱이 비밀번호를 안전하게 보관해야 합니다.
- 앱 하나의 접근만 개별적으로 취소하기 어렵습니다.
- 앱이 침해되면 사용자의 다른 데이터까지 노출될 수 있습니다.
- 읽기만 허용하거나 짧은 시간만 허용하기 어렵습니다.

OAuth는 사용자 비밀번호 대신 목적, 범위와 유효 기간이 제한된 token을 사용합니다.

~~~text
비밀번호 공유
사용자 ── 비밀번호 ── 제3자 앱 ── 서비스 전체 접근

OAuth 권한 위임
사용자 ── 동의 ── 권한 부여 서버
                    │
                    └─ 제한된 access token ── 제3자 앱 ── 허용된 API
~~~

## OAuth는 인증이 아니라 인가를 위한 규격이다

- **인증(Authentication)**: 사용자가 누구인지 확인합니다.
- **인가(Authorization)**: 확인된 주체가 어떤 자원에 무엇을 할 수 있는지 결정합니다.

OAuth 2.0은 원래 **인가와 권한 위임**을 위한 규격입니다. 사용자가 권한 부여 서버에 로그인하는 단계가 흐름에 포함되기 때문에 OAuth 자체를 로그인 규격으로 오해하기 쉽습니다.

“외부 계정으로 로그인” 기능이 필요하면 OAuth 2.0 위에 인증 계층을 추가한 [OpenID Connect](https://openid.net/specs/openid-connect-core-1_0.html)를 사용합니다. OpenID Connect의 ID token은 사용자 인증 정보를 전달하고, OAuth access token은 resource server의 API를 호출하는 데 사용합니다.

access token을 해석해 로그인 사용자를 임의로 판단하거나, ID token을 API access token 대신 보내서는 안 됩니다.
{: .notice--info}

## 네 가지 역할

OAuth 2.0은 다음 네 역할을 정의합니다.

| 역할 | 의미 | 캘린더 예시 |
|---|---|---|
| Resource Owner | 보호된 자원에 대한 권한을 부여할 수 있는 주체 | 캘린더 사용자 |
| Client | 사용자 대신 자원에 접근하려는 애플리케이션 | 일정 관리 앱 |
| Authorization Server | 사용자의 동의를 받고 token을 발급하는 서버 | 계정·권한 서버 |
| Resource Server | access token을 검사하고 보호된 API를 제공하는 서버 | 캘린더 API |

Authorization Server와 Resource Server는 같은 서비스에 있을 수도 있고 별도 시스템일 수도 있습니다. OAuth에서 `client`는 브라우저 자체가 아니라 권한을 요청하는 애플리케이션을 뜻합니다.

## 알아야 할 핵심 용어

### Client ID와 Client Secret

`client_id`는 Authorization Server에 등록된 애플리케이션의 공개 식별자입니다. 브라우저 URL에 노출돼도 되는 값이며 비밀번호가 아닙니다.

`client_secret`은 안전한 서버 환경에서만 보관할 수 있는 client 인증 정보입니다. 브라우저 JavaScript, 모바일 앱과 배포된 데스크톱 앱은 사용자가 코드를 확인하거나 추출할 수 있으므로 secret을 안전하게 숨길 수 있는 confidential client로 볼 수 없습니다.

### Redirect URI

사용자의 동의가 끝난 뒤 Authorization Server가 브라우저를 돌려보낼 client의 주소입니다. 공격자의 주소로 authorization code가 유출되지 않도록 사전에 등록하고 요청값과 정확히 비교해야 합니다.

~~~text
https://client.example.com/oauth/callback
~~~

### Scope

client가 요청하는 권한의 범위입니다.

~~~text
calendar.read
calendar.write
profile.read
~~~

필요한 최소 scope만 요청하고, Resource Server는 API마다 token에 허용된 scope를 검사해야 합니다. scope 문자열의 정확한 의미는 Authorization Server와 API가 정의합니다.

### Authorization Code

사용자 동의 후 redirect URI로 전달되는 짧고 일회용인 중간 자격 증명입니다. client는 이 code를 token endpoint에서 access token으로 교환합니다. code 자체로 API를 호출하지 않습니다.

### Access Token

보호된 API에 접근할 때 사용하는 자격 증명입니다. scope와 유효 기간이 제한되며 불투명한 임의 문자열일 수도 있고 서명된 구조를 가진 token일 수도 있습니다. client는 token이 반드시 JWT라고 가정하면 안 됩니다.

Bearer token은 소유한 사람이 사용할 수 있으므로 전송과 저장 중에 노출되지 않도록 보호해야 합니다. [RFC 6750](https://www.rfc-editor.org/rfc/rfc6750.html)은 HTTP `Authorization` header 사용 방식을 정의합니다.

~~~http
Authorization: Bearer ACCESS_TOKEN
~~~

### Refresh Token

만료된 access token을 새로 발급받을 때 사용할 수 있는 장기 자격 증명입니다. 모든 흐름이나 client에 항상 발급되는 것은 아닙니다. access token보다 오래 살아남을 수 있으므로 더 엄격하게 보관하고 rotation과 재사용 탐지를 적용해야 합니다.

### State와 PKCE

`state`는 authorization 요청과 callback이 같은 사용자 흐름에서 시작됐는지 확인하는 일회용 값으로 CSRF 방어에 사용합니다.

**PKCE(Proof Key for Code Exchange)**는 client가 무작위 `code_verifier`를 만들고 그 hash인 `code_challenge`를 authorization 요청에 보냅니다. 나중에 token을 교환할 때 원본 verifier를 제출하게 해 탈취된 authorization code만으로 token을 발급받기 어렵게 합니다.

현재 보안 권고는 public client에 PKCE를 요구하고 confidential client에도 PKCE 사용을 권장합니다.

## Authorization Code + PKCE 흐름

사용자가 참여하는 웹, 모바일과 데스크톱 애플리케이션에서 중심이 되는 흐름입니다.

~~~text
사용자        Client        Authorization Server       Resource Server
  │              │                    │                        │
  │ 기능 선택    │                    │                        │
  │─────────────>│                    │                        │
  │              │ code_verifier 생성│                        │
  │<──── authorization endpoint로 redirect ────────────────  │
  │──────── 로그인·동의 ─────────────>│                        │
  │<──── code + state로 callback ─────│                        │
  │              │                    │                        │
  │              │ code + verifier ──>│                        │
  │              │<──── access token ─│                        │
  │              │                                             │
  │              │──── Bearer access token으로 API 호출 ─────>│
  │              │<──────────── 보호된 자원 응답 ─────────────│
~~~

### 1. PKCE 값 만들기

다음 Python 코드는 예제를 위한 `code_verifier`, `code_challenge`와 `state`를 생성합니다.

~~~python
import base64
import hashlib
import secrets


def base64url_without_padding(value: bytes) -> str:
    return base64.urlsafe_b64encode(value).rstrip(b"=").decode("ascii")


code_verifier = secrets.token_urlsafe(64)
code_challenge = base64url_without_padding(
    hashlib.sha256(code_verifier.encode("ascii")).digest()
)
state = secrets.token_urlsafe(32)
~~~

`code_verifier`와 `state`는 사용자의 authorization transaction에 안전하게 연결해 저장합니다. 로그, 분석 URL이나 외부 페이지에 노출하지 않습니다.

실제 서비스에서는 OAuth client library를 사용하고 직접 보안 프로토콜을 구현하지 않는 편이 안전합니다. 이 코드는 각 값의 관계를 설명하기 위한 예제입니다.
{: .notice--warning}

### 2. Authorization endpoint로 이동하기

client는 사용자의 브라우저를 다음과 같은 URL로 보냅니다.

~~~text
https://auth.example.com/authorize
  ?response_type=code
  &client_id=calendar-client
  &redirect_uri=https%3A%2F%2Fclient.example.com%2Foauth%2Fcallback
  &scope=calendar.read
  &state=RANDOM_STATE
  &code_challenge=BASE64URL_SHA256_CHALLENGE
  &code_challenge_method=S256
~~~

사용자는 Authorization Server에서 인증하고 client 이름, 요청 scope와 접근 대상을 확인한 뒤 동의하거나 거절합니다. client가 사용자의 비밀번호를 받는 것이 아닙니다.

### 3. Callback 검증하기

동의하면 Authorization Server가 등록된 redirect URI로 authorization code와 state를 전달합니다.

~~~text
https://client.example.com/oauth/callback
  ?code=ONE_TIME_AUTHORIZATION_CODE
  &state=RANDOM_STATE
~~~

client는 callback의 `state`가 처음 저장한 값과 정확히 일치하는지 먼저 확인합니다. 누락되거나 다르면 흐름을 중단합니다. authorization code는 짧은 시간 안에 한 번만 사용합니다.

### 4. Code를 token으로 교환하기

client backend는 token endpoint에 authorization code와 원래의 `code_verifier`를 전송합니다.

~~~bash
curl -X POST "https://auth.example.com/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=authorization_code" \
  --data-urlencode "client_id=calendar-client" \
  --data-urlencode "code=ONE_TIME_AUTHORIZATION_CODE" \
  --data-urlencode "redirect_uri=https://client.example.com/oauth/callback" \
  --data-urlencode "code_verifier=ORIGINAL_CODE_VERIFIER"
~~~

예시 응답은 다음과 같습니다.

~~~json
{
  "access_token": "ACCESS_TOKEN",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "calendar.read",
  "refresh_token": "REFRESH_TOKEN_IF_ISSUED"
}
~~~

confidential client는 token endpoint에서 자신을 인증해야 할 수 있습니다. 정확한 인증 방식은 Authorization Server metadata와 client 등록 정보를 따릅니다.

### 5. Access token으로 API 호출하기

access token은 query string보다 HTTP Authorization header에 넣습니다.

~~~bash
curl "https://api.example.com/v1/calendars/me/events" \
  -H "Authorization: Bearer ACCESS_TOKEN"
~~~

Resource Server는 token의 유효성, 발급자, 대상 API, 만료와 필요한 scope를 검사한 뒤 요청을 처리합니다. JWT를 사용한다면 서명만 확인하고 끝내지 말고 `iss`, `aud`, `exp` 등 API가 요구하는 claim도 검증해야 합니다.

## 서버 간 통신: Client Credentials

사용자 대신이 아니라 서비스가 **자기 권한으로** 다른 API를 호출할 때 Client Credentials grant를 사용할 수 있습니다. 예를 들어 사내 batch server가 정해진 scope로 통계 API를 호출하는 경우입니다.

~~~text
Batch Client ── client 인증 + scope ──> Authorization Server
Batch Client <────── access token ───── Authorization Server
Batch Client ───── access token ──────> Resource Server
~~~

간단한 token 요청 형태는 다음과 같습니다.

~~~bash
curl -X POST "https://auth.example.com/token" \
  -u "CLIENT_ID:CLIENT_SECRET" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "scope=reports.read"
~~~

이 흐름에는 사용자의 로그인과 동의 화면이 없습니다. 발급된 token은 특정 사용자가 아니라 client service의 권한을 나타냅니다. 브라우저나 모바일 앱처럼 secret을 안전하게 보관할 수 없는 public client에는 적합하지 않습니다.

실제 고보안 환경에서는 공유 secret 대신 `private_key_jwt`나 mutual TLS 같은 비대칭 client 인증도 고려할 수 있습니다.

## 어떤 흐름을 선택해야 하는가

| 상황 | 주로 검토할 흐름 |
|---|---|
| 사용자가 참여하는 웹 애플리케이션 | Authorization Code + PKCE |
| SPA | Authorization Code + PKCE, 필요하면 BFF 구조 |
| 모바일·데스크톱 앱 | Authorization Code + PKCE와 시스템 브라우저 |
| 입력이 어려운 TV·CLI 장치 | Device Authorization Grant |
| 서버 간 서비스 통신 | Client Credentials |

새 구현에서는 implicit grant와 Resource Owner Password Credentials grant를 피합니다. 최신 [RFC 9700 보안 권고](https://www.rfc-editor.org/rfc/rfc9700.html)는 access token이 authorization response에 직접 노출되는 implicit 방식 대신 code flow를 사용하도록 하고, 사용자의 비밀번호를 client가 직접 다루는 방식도 사용하지 않도록 안내합니다.

OAuth grant를 직접 조합하기보다 사용하는 플랫폼이 권장하는 검증된 library와 흐름을 선택해야 합니다.

## Access token과 ID token을 구분하기

| 항목 | Access token | ID token |
|---|---|---|
| 규격 | OAuth 2.0 | OpenID Connect |
| 목적 | Resource Server API 접근 | Client에 사용자 인증 결과 전달 |
| 주요 수신자 | API | 로그인 client |
| 사용 위치 | `Authorization: Bearer` | client session 생성·사용자 확인 |

ID token의 `aud`는 일반적으로 client를 가리키므로 API가 access token 대신 받아서는 안 됩니다. 반대로 access token의 payload에서 이메일을 꺼내 로그인 결과로 사용하는 것도 올바른 OpenID Connect 처리 방식이 아닙니다.

## Token이 반드시 JWT인 것은 아니다

OAuth 2.0은 access token의 내부 형식을 하나로 고정하지 않습니다.

- **Opaque token**: client가 해석할 수 없는 임의 문자열이며 API가 introspection이나 서버 저장소로 확인할 수 있습니다.
- **JWT access token**: 필요한 claim과 서명을 담아 API가 로컬에서 검증할 수 있습니다.

JWT는 암호화된 문자열이 아닐 수 있습니다. payload를 누구나 읽을 수 있는 signed JWT가 일반적이므로 비밀번호나 불필요한 개인정보를 넣지 않습니다. token 종류와 검증 규칙은 Authorization Server가 제공하는 metadata와 API 계약을 따릅니다.

## 보안에서 반드시 확인할 것

### Redirect URI를 정확히 비교한다

부분 일치나 wildcard가 잘못 적용되면 authorization code가 공격자 주소로 전달될 수 있습니다. 등록값과 요청값을 exact string matching하고 자유로운 redirect URL을 받는 open redirector를 만들지 않습니다.

### PKCE는 S256을 사용한다

각 authorization 요청마다 충분히 무작위인 verifier를 새로 만들고 `S256` challenge를 사용합니다. 고정된 verifier를 재사용하지 않습니다.

### State를 일회용으로 검증한다

state를 예측하기 어려운 값으로 만들고 사용자의 브라우저 session과 연결합니다. callback에서 비교한 뒤 즉시 폐기합니다. OpenID Connect에서는 ID token의 `nonce`도 검증합니다.

### Token을 URL에 넣지 않는다

URL query는 browser history, proxy, analytics와 referrer log에 남을 수 있습니다. access token은 Authorization header에 넣고 서버 로그와 오류 메시지에서 제거합니다.

### Token을 안전하게 저장한다

브라우저에서 접근 가능한 저장소는 XSS에 노출될 수 있습니다. 애플리케이션 구조에 따라 backend session 또는 BFF를 검토하고, refresh token은 암호화와 접근 통제, rotation을 적용합니다.

### Scope와 audience를 최소화한다

모든 API에 사용할 수 있는 장기 token 하나보다 짧은 수명, 좁은 scope와 특정 resource server를 대상으로 한 token이 안전합니다. Resource Server가 실제 endpoint에 필요한 권한을 다시 확인해야 합니다.

### TLS와 검증된 library를 사용한다

authorization endpoint, token endpoint와 API 통신을 HTTPS로 보호합니다. 직접 서명 검증, redirect 처리와 token 갱신 로직을 만들기보다 유지보수되는 OAuth·OpenID Connect library를 사용합니다.

## 자주 하는 오해

### OAuth로 로그인했으니 사용자 인증도 끝났다

OAuth access token을 받았다는 사실만으로 client가 사용자의 신원을 안전하게 확인한 것은 아닙니다. 로그인에는 OpenID Connect 흐름과 ID token 검증을 사용합니다.

### Client ID도 숨겨야 한다

client ID는 공개 식별자입니다. 보호해야 하는 값은 client secret, access token, refresh token과 PKCE verifier입니다.

### JWT면 서버 저장소나 폐기가 필요 없다

JWT도 key rotation, 만료, audience와 issuer 검증이 필요합니다. 즉시 철회가 필요한 정책이라면 짧은 수명, deny list, introspection 또는 별도 session 전략을 설계해야 합니다.

### Refresh token은 만료되지 않는 access token이다

Refresh token은 새 access token을 요청하기 위한 별도 자격 증명입니다. 일반 API 호출에 사용하지 않으며 탈취와 재사용을 탐지하도록 rotation을 적용할 수 있습니다.

### Scope만 있으면 모든 권한 검사가 끝난다

Scope는 넓은 권한 범위일 뿐입니다. API는 사용자가 해당 문서나 계정에 실제로 접근할 수 있는지도 객체 단위로 검사해야 합니다.

## 구현 전 체크리스트

- OAuth 2.0과 OpenID Connect 중 필요한 목적을 구분했는가
- Authorization Code + PKCE를 사용하는가
- Redirect URI를 정확히 등록하고 비교하는가
- State와 필요한 경우 nonce를 일회용으로 검증하는가
- 최소 scope와 올바른 audience를 요청하는가
- Token을 URL과 로그에 노출하지 않는가
- Access token과 refresh token을 안전하게 저장하는가
- Resource Server가 만료, 발급자, 대상과 scope를 검증하는가
- 사용 중인 library의 보안 업데이트를 적용하는가
- Implicit와 password grant 같은 구형 흐름을 새로 도입하지 않는가

## 정리

OAuth 2.0의 핵심은 사용자 비밀번호를 공유하지 않고 제한된 API 접근 권한을 위임하는 것입니다. Resource Owner, Client, Authorization Server와 Resource Server가 역할을 나누고, client는 사용자의 동의를 authorization code와 access token으로 바꿔 보호된 자원에 접근합니다.

현재 사용자 참여형 애플리케이션의 기본 선택은 Authorization Code + PKCE입니다. 서버 간 통신에는 Client Credentials를 사용할 수 있습니다. 로그인 기능이 필요하면 OAuth만 임의로 해석하지 말고 OpenID Connect를 사용합니다.

예제 요청 자체는 단순하지만 실제 보안은 redirect URI, PKCE, state, token 저장, scope, audience와 Resource Server의 검증이 함께 맞아야 완성됩니다. 운영 서비스에서는 직접 구현보다 검증된 library와 Authorization Server의 공식 가이드를 따르는 것이 안전합니다.

## 참고 자료

- IETF, [RFC 6749: The OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749.html)
- IETF, [RFC 6750: Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)
- IETF, [RFC 7636: Proof Key for Code Exchange](https://www.rfc-editor.org/rfc/rfc7636.html)
- IETF, [RFC 9700: Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)
- IETF, [RFC 8628: OAuth 2.0 Device Authorization Grant](https://www.rfc-editor.org/rfc/rfc8628.html)
- OpenID Foundation, [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- IETF OAuth Working Group, [OAuth 2.1 Internet-Draft](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/)
