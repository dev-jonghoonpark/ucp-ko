# 신원 연결 기능

- **기능(Capability) 이름:** `dev.ucp.common.identity_linking`
- **스키마:** `https://ucp.dev/schemas/common/identity_linking.json`

## 개요

신원 연결(Identity Linking) 기능을 사용하면 **플랫폼**이 **비즈니스**(신뢰 당사자, relying party) 사이트에서 **사용자**를 대신해 작업할 수 있는 인가를 받을 수 있습니다.

이 연결은 사용자 인증 기반 커머스 경험의 토대입니다. 로열티 혜택, 개인화된 제안, 저장된 주소, 위시리스트, 주문 내역에 접근하려면 신원 연결이 필요합니다. 신원 연결이 없어도 기능은 공개 또는 에이전트 인증 수준에서 그대로 동작합니다. 신원 연결은 경험을 한 단계 끌어올릴 뿐, 접근을 막는 관문이 아닙니다.

**이 명세는 인가에 [OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)을 v1 인증 메커니즘으로 사용합니다.** 스키마는 점진적으로 확장할 수 있게 설계되었습니다. 향후 버전에서는 위임 신원 제공자(IdP) 지원과 OAuth가 아닌 인증 메커니즘을 모두 하나의 `config.providers` 확장 지점으로 추가할 예정이며, 이 버전과의 호환성을 깨지 않습니다([향후 확장성](#future-extensibility) 참고).

### 참여자

| UCP 역할     | 신원 역할                     | 설명                                                                                                      |
| ------------ | ----------------------------- | --------------------------------------------------------------------------------------------------------- |
| **플랫폼**   | 사용자 에이전트(User Agent)   | 신원 연결을 시작하고, 사용자를 대신해 비즈니스에 사용자 신원 토큰을 제시하는 신뢰할 수 있는 중개자입니다. |
| **비즈니스** | 인가 서버 / 신뢰 당사자       | 자체 OAuth 2.0 인가 서버를 운영합니다. 사용자를 인증하고 UCP 기능 범위로 한정된 액세스 토큰을 발급합니다. |
| **사용자**   | 리소스 소유자(Resource Owner) | 신원이 연결되는 당사자입니다. OAuth 인가 흐름 중에 플랫폼에 명시적으로 동의합니다.                        |

### 접근 수준

기능은 세 가지 접근 수준에서 동작합니다.

| 수준              | 인증                                            | 예시                                       |
| ----------------- | ----------------------------------------------- | ------------------------------------------ |
| **공개**          | 없음                                            | 공개 카탈로그 탐색                         |
| **에이전트 인증** | 플랫폼 자격 증명(`client_id` / `client_secret`) | 게스트 체크아웃, 장바구니 생성             |
| **사용자 인증**   | 플랫폼 자격 증명 + 사용자 신원 토큰             | 저장된 주소, 전체 주문 내역, 개인화된 가격 |

신원 연결은 에이전트 인증 접근과 사용자 인증 접근 사이를 이어 줍니다. 플랫폼은 아래에 설명하는 OAuth 흐름을 완료해 사용자 신원 토큰을 얻고, 이후 요청에서 이 토큰을 제시합니다.

**신원 연결과 기능 협상은 서로 독립된 계층입니다.** 기능이 알려지고 협상되는지는 그 기능 자체가 프로필에 있는지에 달려 있으며, 신원 연결이 없다고 해서 제외되지는 않습니다. 신원 연결이 있으면, 협상된 기능 *안에서* 사용자 인증이 필요한 작업을 제한하는 **스코프**를 선언합니다([스코프](#scopes) 참고). 프로필에 `dev.ucp.shopping.order`를 나열한 판매자라면 어느 경우든 협상된 교집합에 이 기능이 포함됩니다. 그 프로필에 신원 연결도 *함께* 나열하고 `config.scopes`에 `dev.ucp.shopping.order:read`를 두었다면, 해당 스코프가 적용되는 작업에는 사용자 신원 토큰이 필요합니다.

### UCP와 OAuth

UCP는 커머스 의미 체계(어떤 스코프가 무엇을 뜻하고, 어떤 스코프가 어떤 작업을 제한하는지)를 정의합니다. OAuth([RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414))는 신원 메커니즘(엔드포인트, 흐름, 허용되는 스코프 어휘)을 정의합니다. 런타임 메시지는 요청별 안내를 전달합니다.

- **UCP `config.scopes`**: **강제 관문**을 선언합니다. 즉, 적용 대상 작업에 사용자 인증을 *요구하는* 스코프입니다.
- **OAuth `scopes_supported`**(RFC 8414): **허용 스코프 어휘**를 선언합니다. 즉, 요청되면 인가 서버가 받아들일 모든 스코프입니다.
- 그 차집합(`scopes_supported` ∖ `config.scopes`)이 **선택 계층**입니다. 판매자가 받아들이지만 강제하지는 않는 스코프로, 인증을 요구하지 않으면서 인증하면 열리는 기능을 알릴 때 씁니다.
- **UCP `messages[]`**: **런타임 컨텍스트 힌트**를 전달합니다. `identity_optional` 같은 요청별 알림([선택적 인증](#optional-authentication) 참고)으로, 현재 컨텍스트에서 인증하면 추가 가치를 얻을 수 있음을 알립니다.

## 일반 지침

### 플랫폼

- 비즈니스의 `token_endpoint_auth_methods_supported` 메타데이터 ([RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414))에 명시된 방식으로 토큰 엔드포인트 요청을 반드시 인증해야 합니다(**MUST**).

  - **기밀 클라이언트**(자격 증명을 보호할 수 있는 서버 측 플랫폼)는 비대칭 방식인 `private_key_jwt` ([RFC 7523 §2.2](https://datatracker.ietf.org/doc/html/rfc7523#section-2.2)) 또는 `tls_client_auth` ([RFC 8705](https://datatracker.ietf.org/doc/html/rfc8705))를 우선 사용해야 하며(**SHOULD**), 비즈니스가 지원하는 경우 `client_secret_basic` ([RFC 6749 §2.3.1](https://datatracker.ietf.org/doc/html/rfc6749#section-2.3.1), [RFC 7617](https://datatracker.ietf.org/doc/html/rfc7617))을 사용할 수 있습니다(**MAY**).
  - **공개 클라이언트**([RFC 8252 §8.5](https://datatracker.ietf.org/doc/html/rfc8252#section-8.5)에 따른 네이티브, 데스크톱, 브라우저 확장 프로그램, 온디바이스 에이전트 런타임)는 반드시 `none`을 사용하고, 인가 코드의 소유 증명(proof-of-possession)으로 `S256` 방식의 PKCE ([RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636))에 의존해야 합니다(**MUST**). 공개 클라이언트는 `client_secret`을 내장해서는 안 됩니다(**MUST NOT**).

  플랫폼은 비즈니스가 제공하는 방식 중 플랫폼의 배포 모델과 호환되는 가장 강력한 방식을 반드시 선택해야 합니다(**MUST**).

- 사용자 신원 토큰을 Bearer 방식으로 HTTP `Authorization` 헤더에 반드시 포함해야 합니다(**MUST**): `Authorization: Bearer <access_token>` ([RFC 6750 §2.1](https://datatracker.ietf.org/doc/html/rfc6750#section-2.1)).

- 사용자 인증 작업의 `401`, `403` 응답에 담긴 `WWW-Authenticate: Bearer` 챌린지를 [RFC 6750 §3](https://datatracker.ietf.org/doc/html/rfc6750#section-3)에 따라 반드시 처리해야 합니다(**MUST**). 플랫폼은 이후 인가 요청을 구성할 수 있도록 `scope` 파라미터(있는 경우)를 반드시 추출해야 하며(**MUST**), 리소스를 보호하는 인가 서버를 찾으려면 `resource_metadata` 포인터([RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728))가 있으면 이를 따라가야 합니다(**SHOULD**).

- 계정 연결 메커니즘으로 OAuth 2.0 인가 코드(Authorization Code) 흐름 ([RFC 6749 §4.1](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1))을 반드시 구현해야 합니다(**MUST**).

- 모든 인가 코드 교환에서 `code_challenge_method=S256`과 함께 PKCE ([RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636))를 반드시 사용해야 합니다(**MUST**).

- 믹스업 공격(Mix-Up Attack)을 막기 위해 인가 응답의 `iss` 파라미터 ([RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207))를 반드시 검증해야 합니다(**MUST**). 플랫폼은 `iss` 값이 인가 서버의 발급자(issuer) URI([RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414) 메타데이터에 선언된 값)와 일치하는지 반드시 확인해야 합니다(**MUST**). 값이 일치하지 않으면 플랫폼은 반드시 중단하고 인가 응답을 폐기해야 합니다(**MUST**).

- CSRF를 막기 위해 인가 요청에 고유하고 추측할 수 없는 `state` 파라미터를 포함해야 합니다(**SHOULD**) ([RFC 6749 §10.12](https://datatracker.ietf.org/doc/html/rfc6749#section-10.12)).

- 철회 및 보안 이벤트:

  - 사용자가 플랫폼 측에서 연결 해제를 시작하면 비즈니스의 토큰 철회 엔드포인트 ([RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009))를 반드시 호출해야 합니다(**MUST**).
  - 비즈니스가 시작하는 비동기 계정 업데이트와 교차 계정 보호 이벤트를 처리하기 위해 [OpenID RISC Profile 1.0](https://openid.net/specs/openid-risc-1_0-final.html)을 지원해야 합니다(**SHOULD**).

### 비즈니스

- OAuth 2.0 ([RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749))을 반드시 구현해야 합니다(**MUST**).
- `/.well-known/oauth-authorization-server`에 [RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414)에 따라 인가 서버 메타데이터를 반드시 게시해야 합니다(**MUST**).
- 플랫폼이 인가 흐름을 시작하기 전에 스코프 불일치를 감지할 수 있도록 [RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414) 메타데이터에 `scopes_supported`를 반드시 채워야 합니다(**MUST**).
- 인가 응답에 `iss` 파라미터 ([RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207))를 반드시 반환해야 합니다(**MUST**).
- 토큰 엔드포인트에서 모든 인가 코드 교환에 PKCE ([RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)) 검증을 반드시 강제해야 합니다(**MUST**). 유효한 `code_verifier`가 없는 요청은 반드시 거부해야 합니다(**MUST**).
- 오픈 리디렉션과 토큰 탈취를 막기 위해 인가 요청에서 `redirect_uri` 파라미터를 정확한 문자열 일치로 반드시 검사해야 합니다(**MUST**). 토큰 요청의 `redirect_uri`는 인가 요청의 것과 반드시 동일해야 합니다(**MUST**). **예외 — 루프백 리디렉션:** `127.0.0.1` 또는 `[::1]`을 대상으로 하는 리디렉션 URI라면, 런타임에 OS에서 임시 포트를 받는 네이티브·데스크톱 클라이언트를 수용할 수 있도록 비즈니스는 포트 부분을 반드시 무시하고 스킴, 호스트, 경로만 비교해야 합니다(**MUST**) ([RFC 8252 §7.3](https://datatracker.ietf.org/doc/html/rfc8252#section-7.3)).
- 지원하는 클라이언트 인증 방식을 `token_endpoint_auth_methods_supported` ([RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414))에 반드시 선언하고, 토큰 엔드포인트에서 선언한 방식 중 하나를 강제해야 합니다(**MUST**). 비즈니스는 비대칭 기밀 클라이언트 방식(`private_key_jwt` 또는 `tls_client_auth`)을 하나 이상 지원해야 하며(**SHOULD**), [RFC 8252](https://datatracker.ietf.org/doc/html/rfc8252)에 따라 공개 클라이언트용 `none`을 지원할 수 있습니다(**MAY**). `none`을 선언한다면 비즈니스는 반드시 `S256` 방식의 PKCE를 요구해야 하며 (**MUST**), 유효한 `code_verifier`가 없는 인가 코드 사용 요청은 반드시 거부해야 합니다(**MUST**). 협상된 인증 방식을 통과하지 못한 요청은 반드시 `invalid_client`로 거부해야 하고(**MUST**), PKCE를 통과하지 못한 요청은 반드시 `invalid_grant`로 거부해야 합니다(**MUST**).
- 모든 사용자 인증 요청에서 사용자 신원 토큰을 반드시 검증해야 합니다(**MUST**). `iss`, `aud`(비즈니스의 리소스 서버 식별자), `exp`, 스코프, 그리고 `client_id` / `azp`(또는 그에 상응하는 값)를 확인해, 인증된 플랫폼 클라이언트에게 발급된 토큰인지 확인합니다 ([RFC 9068 §4](https://datatracker.ietf.org/doc/html/rfc9068#section-4)).
- 사용자 인증 작업의 `401 Unauthorized`(`identity_required`) 및 `403 Forbidden`(`insufficient_scope`) 응답에는 [RFC 6750 §3](https://datatracker.ietf.org/doc/html/rfc6750#section-3)에 따른 `WWW-Authenticate: Bearer` 챌린지를 반드시 담아 보내야 합니다(**MUST**). 전체 규범 요구 사항은 [오류 처리](#error-handling)를 참고하세요.
- 토큰 철회 ([RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009))를 반드시 구현해야 합니다(**MUST**). `refresh_token`을 철회하면 그 토큰으로 발급된 모든 `access_token`도 반드시 즉시 무효화해야 합니다(**MUST**).
- 토큰 엔드포인트에서 쓰는 것과 같은 클라이언트 자격 증명으로 인증하는 철회 요청을 반드시 지원해야 합니다(**MUST**).
- 사용자에게 아직 계정이 없으면 계정 생성 흐름을 제공하거나, 온보딩 흐름을 가리키는 `continue_url`을 `identity_required` 오류 응답에 담아 반환해야 합니다(**SHOULD**)([오류 처리](#error-handling) 참고).
- [스코프](#scopes) 섹션에 정의된 표준 UCP 스코프를 반드시 지원해야 합니다(**MUST**).
- `/.well-known/oauth-protected-resource`에 보호 리소스 메타데이터 ([RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728))를 게시하고, `WWW-Authenticate` 챌린지의 `resource_metadata` 파라미터로 이 문서를 가리켜야 합니다(**SHOULD**). 이렇게 하면 플랫폼은 도메인 관례에 기대지 않고도 리소스를 보호하는 인가 서버를 찾을 수 있고, 인가 서버가 비즈니스 도메인에 있지 않을 수도 있는 향후 위임 IdP 시나리오에도 배포를 대비할 수 있습니다.
- 플랫폼에 철회와 계정 상태 변경을 알리기 위해 [OpenID RISC Profile 1.0](https://openid.net/specs/openid-risc-1_0-final.html)을 지원해야 합니다(**SHOULD**).

## 디스커버리

플랫폼은 다음과 같은 **엄격한 2단계 계층 구조**로 비즈니스의 인가 서버 메타데이터를 확인합니다. 디스커버리 기준으로 쓰는 발급자(issuer) URI는 비즈니스가 UCP 프로필에 선언한 도메인입니다.

1. **RFC 8414(1순위):** `{business-domain}/.well-known/oauth-authorization-server`를 가져옵니다.

   - `2xx` 응답: 이 메타데이터를 사용합니다. 디스커버리 완료.
   - `404 Not Found`: 2단계로 진행합니다.
   - 그 밖의 비-2xx 응답, 네트워크 오류, 타임아웃: 반드시 중단해야 합니다(**MUST**). 2단계로 진행해서는 안 됩니다(**MUST NOT**).

1. **OIDC 디스커버리(대체):** `{business-domain}/.well-known/openid-configuration`을 가져옵니다.

   - `2xx` 응답: 이 메타데이터를 사용합니다. 디스커버리 완료.
   - 비-2xx 응답, 네트워크 오류, 타임아웃: 신원 연결 과정을 반드시 중단해야 합니다(**MUST**).

플랫폼은 1단계에서 `404` 이외의 오류가 발생했을 때 아무 표시 없이 다음 단계로 넘어가서는 안 됩니다(**MUST NOT**). 이렇게 하면 서버가 잘못 구성되었거나 일시적으로 사용할 수 없을 때 일부만 처리되거나 정의되지 않은 동작이 일어나는 것을 막을 수 있습니다.

발견한 메타데이터의 `issuer` 값은 디스커버리 기준 URI와 정확히 일치해야 합니다(**MUST**) ([RFC 8414 §3.3](https://datatracker.ietf.org/doc/html/rfc8414#section-3.3)에 따름). 플랫폼은 비교 전에 정규화(예: 끝의 슬래시 제거)를 해서는 안 됩니다(**MUST NOT**). 값은 바이트 단위로 일치해야 합니다.

## 계정 연결 흐름

신원 연결은 PKCE를 적용한 OAuth 2.0 인가 코드 흐름을 사용합니다.

```text
Platform                              Business AS
   |                                       |
   |-- (1) Discover metadata via RFC 8414 -->|
   |<-- authorization_endpoint, token_endpoint, scopes_supported --|
   |                                       |
   |-- (2) Authorization Request --------->|
   |       response_type=code              |
   |       client_id, redirect_uri         |
   |       scope=<derived scope set>       |
   |       code_challenge (S256)           |
   |       state                           |
   |                                       |
   |       [user authenticates and         |
   |        grants consent at business]    |
   |                                       |
   |<-- (3) Authorization Response --------|
   |       code, state, iss                |
   |                                       |
   |  Validate: state matches, iss matches |
   |  discovered issuer URI                |
   |                                       |
   |-- (4) Token Request ----------------->|
   |       grant_type=authorization_code   |
   |       code, redirect_uri              |
   |       code_verifier                   |
   |       client auth (per advertised     |
   |       token_endpoint_auth_method)     |
   |                                       |
   |<-- (5) Token Response ----------------|
   |       access_token, refresh_token     |
   |       token_type=Bearer, scope        |
```

**2단계 — 스코프 집합:** 플랫폼은 비즈니스의 `config.scopes` 맵에서 인가 스코프 집합을 도출합니다([스코프 도출](#scope-derivation) 참고). 플랫폼은 도출한 스코프 집합만 반드시 요청해야 하며(**MUST**), 그 상위 집합을 요청해서는 안 됩니다.

**3단계 — 검증:** 플랫폼은 `state` 파라미터가 2단계에서 보낸 값과 일치하는지, 그리고 `iss` 파라미터가 발견한 메타데이터의 인가 서버 `issuer` URI와 일치하는지 반드시 확인해야 합니다(**MUST**). 어느 하나라도 실패하면 플랫폼은 인가 응답을 반드시 폐기해야 합니다(**MUST**).

**4단계 — PKCE:** `code_verifier`는 2단계에서 보낸 `code_challenge`와 반드시 대응해야 합니다(**MUST**). 비즈니스는 `code_verifier`가 없거나 저장된 `code_challenge`로 검증되지 않는 토큰 요청을 반드시 거부해야 합니다(**MUST**).

## 스코프

스코프는 비즈니스가 플랫폼에 부여하는 사용자 인증 권한을 정의합니다. 비즈니스는 제공하는 스코프를 `dev.ucp.common.identity_linking` 항목의 `config.scopes`에 선언합니다. 각 키는 실제 전송되는 형태 그대로의 OAuth 스코프 문자열 (`{capability}:{scope}`, 예: `dev.ucp.shopping.order:read`)이고, 각 값은 스코프별 정책 객체입니다.

`config.scopes`에 스코프를 나열하면 해당 작업에 사용자 신원 토큰이 필요하다고 선언하는 것입니다. 나열된 어떤 스코프로도 제한되지 *않는* 작업은 비즈니스가 허용하는 접근 수준(공개, 에이전트 인증 등)에서 동작합니다. 스코프가 없는 작업의 접근 정책은 비즈니스가 정의하며, UCP는 기본값을 규정하지 않습니다.

### 스코프 토큰 형식

스코프 토큰은 `{capability-name}:{scope-name}` 규칙을 따릅니다.

- `dev.ucp.shopping.order:read`
- `dev.ucp.shopping.order:manage`
- `dev.ucp.shopping.checkout:manage`

기능 이름은 UCP의 역방향 DNS 명명법을 따릅니다. 스코프 이름은 부여되는 **권한**을 나타내며, 보통 각 기능의 명세에서 정의하는 리소스에 대한 작업 그룹(`read`, `manage`, `write`)이나 진입점 작업(`create`)입니다.

스코프 이름은 반드시 `^[a-z][a-z0-9_]*$` 패턴과 일치해야 합니다(**MUST**). 서드파티 기능도 자체 역방향 DNS 이름으로 같은 규칙을 따릅니다: `com.example.loyalty:points`.

각 기능의 명세는 플랫폼이 기대하는 표준 집합인 **잘 알려진(well-known)** 스코프를 정의합니다. 비즈니스는 같은 규칙에 따라 **커스텀** 스코프를 추가로 선언해 더 세밀한 단위로 작업을 제한할 수 있습니다(**MAY**)(예: 잘 알려진 `dev.ucp.shopping.checkout:manage`와 별개로 `complete`를 제한하거나, 고액 구매에 더 높은 수준의 인증을 요구). 플랫폼은 `config.scopes`에 나열된 스코프라면 잘 알려진 것이든 커스텀이든 모두, 해당 작업을 사용자 인증이 있어야만 쓸 수 있게 제한하는 것으로 반드시 취급해야 합니다(**MUST**).

### 스코프별 정책과 메타데이터

각 스코프의 값은 스코프별 정책과 메타데이터를 담는 열린 객체입니다. 빈 `{}`는 "사용자 인증 필요, 그 외 조건 없음"을 뜻합니다. 인증 제약(`min_acr`, `max_token_age`, `require_mfa`), 선언적 메타데이터(부여 시 생성되는 `claims`), 그 밖의 스코프별 구성 등을 필드로 넣을 수 있습니다. 플랫폼은 인식하지 못하는 필드를 반드시 무시해야 합니다(**MUST**).

#### `description`

사람이 읽을 수 있는 스코프 설명이며 선택 사항입니다. 플랫폼은 이 설명으로 사용자에게 컨텍스트(요구 사항과 가치)를 보여 주고 설명할 수 있습니다.

```json
"dev.ucp.shopping.order:manage": {
  "description": {
    "plain": "Manage your orders: cancel, return, or modify post-purchase.",
    "markdown": "**Manage your orders**: cancel, return, or modify post-purchase."
  }
}
```

비즈니스는 선언하는 각 스코프에 설명을 제공해야 합니다(**SHOULD**). 플랫폼은 자체 동의 UX에 참고하도록 이 텍스트를 표시할 수 있습니다(**MAY**).

### 스코프 도출

플랫폼은 계정 연결 흐름을 시작하기 전에 비즈니스의 `config.scopes`에서 인가 스코프 집합을 도출합니다.

1. 비즈니스의 `dev.ucp.common.identity_linking` 기능 항목에서 `config.scopes`를 읽습니다.
1. 기능 접두사가 협상된 기능 집합에 포함된 스코프만 남깁니다. 플랫폼이 지원하지 않는 기능의 스코프는 무시합니다.
1. 남은 집합에서 플랫폼이 사용하려는 스코프를 선택합니다(호출하려는 작업이 기준이며, 작업과 스코프의 대응 관계는 각 기능의 명세를 참고하세요).
1. 인가 요청을 구성할 때 선택한 각 스코프에 스코프별 정책을 적용합니다.

플랫폼이 호출하려는 작업에 필요한 스코프가 없으면 플랫폼은 신원 연결 흐름을 건너뛸 수 있습니다. 그러면 작업은 공개 또는 에이전트 인증 접근으로 동작합니다. 다만 세션 고유의 개인화(저장된 주소, 회원 가격, 주문 내역 조회 등)를 활용하려면 연결하는 편이 여전히 나을 수 있습니다. 판매자는 어떤 스코프가 부여되었는지와 관계없이 인증된 사용자 컨텍스트에서 이런 정보를 확인합니다.

### 동의 화면 표시

동의 화면은 비즈니스의 인가 서버가 렌더링합니다. 이 명세는 스코프 설명 문자열을 정의하지 않습니다. 지원하는 스코프의 동의 문구를 사람이 읽을 수 있게 보여 주는 일은 인가 서버가 맡습니다. 동의 화면은 개별 작업을 나열하기보다 관련 스코프를 이해하기 쉽게 묶어야 합니다(**SHOULD**). 예를 들어 "grant `dev.ucp.shopping.order:read`" 대신 "[플랫폼]이 주문 내역을 볼 수 있도록 허용"처럼 표시합니다.

## 오류 처리

### `identity_required`

`config.scopes`에 나열된 스코프로 제한되는 작업에 요청이 왔는데 사용자 신원 토큰이 없거나, 만료되었거나, 유효하지 않거나, 검증할 수 없다면 비즈니스는 반드시 다음을 반환해야 합니다(**MUST**).

- HTTP `401 Unauthorized`
- [RFC 6750 §3](https://datatracker.ietf.org/doc/html/rfc6750#section-3)에 따른 `WWW-Authenticate: Bearer` 챌린지 헤더
- `code: "identity_required"`인 메시지를 담은 UCP 오류 응답 본문

`WWW-Authenticate` 헤더에는 [RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414) 메타데이터에 선언된 비즈니스의 발급자 URI로 설정한 `realm` 파라미터가 반드시 포함되어야 합니다(**MUST**). 요청에 토큰이 있었지만 유효하지 않거나, 만료되었거나, 검증할 수 없는 경우에는 헤더에 `error="invalid_token"`도 반드시 포함해야 하며 (**MUST**), RFC 6750 §3에 따라 `error_description`을 포함할 수 있습니다(**MAY**). 토큰이 제시되지 않은 경우에는 `error` 파라미터를 생략해야 합니다(**SHOULD**) (RFC 6750 §3.1).

헤더에는 보호 리소스 메타데이터 문서(`/.well-known/oauth-protected-resource`)를 가리키는 `resource_metadata` 파라미터 ([RFC 9728 §5.1](https://datatracker.ietf.org/doc/html/rfc9728#section-5.1))를 포함해야 합니다(**SHOULD**).

사용자가 다시 인증하기 전에 호스팅된 단계를 완료해야 하는 **OAuth가 아닌 온보딩 흐름**(예: 계정 생성, 약관 동의)이라면 비즈니스는 응답 본문에 `continue_url`을 포함할 수 있습니다(**MAY**). 미리 만들어 둔 OAuth 인가 요청을 `continue_url`로 전달해서는 안 됩니다(**MUST NOT**). 플랫폼은 `WWW-Authenticate` 챌린지와 발견한 메타데이터를 바탕으로, 자신이 소유한 PKCE 값, `state`, `redirect_uri`를 포함해 인가 요청을 직접 구성합니다.

**토큰이 제시되지 않은 경우**(제한된 작업에 대한 첫 요청 — RFC 6750 §3.1에 따라 `error` 생략):

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="https://merchant.example.com",
                  resource_metadata="https://merchant.example.com/.well-known/oauth-protected-resource"
Content-Type: application/json

{
  "messages": [
    {
      "type": "error",
      "code": "identity_required",
      "content": "User identity is required to access order history.",
      "severity": "requires_buyer_review"
    }
  ]
}
```

**토큰은 있지만 유효하지 않거나 만료된 경우**(RFC 6750 §3에 따라 `error="invalid_token"` 포함):

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="https://merchant.example.com",
                  error="invalid_token",
                  error_description="The access token expired",
                  resource_metadata="https://merchant.example.com/.well-known/oauth-protected-resource"
Content-Type: application/json

{
  "messages": [
    {
      "type": "error",
      "code": "identity_required",
      "content": "User identity is required to access order history.",
      "severity": "requires_buyer_review"
    }
  ]
}
```

### `insufficient_scope`

요청에 유효한 사용자 신원 토큰이 있지만, 토큰에 작업이 요구하는 스코프가 없거나 `min_acr`, `max_token_age` 같은 스코프 수준 정책을 충족하지 못하면 비즈니스는 반드시 다음을 반환해야 합니다(**MUST**).

- HTTP `403 Forbidden`
- RFC 6750 §3에 따른 `WWW-Authenticate: Bearer` 챌린지 헤더
- `code: "insufficient_scope"`인 메시지를 담은 UCP 오류 응답 본문

`WWW-Authenticate` 헤더에는 반드시 다음이 포함되어야 합니다(**MUST**).

- `realm="<business issuer URI>"`
- `error="insufficient_scope"`
- `scope="<space-separated full required scope set for the operation>"`

또한 보호 리소스 메타데이터 문서(RFC 9728)를 가리키는 `resource_metadata` 파라미터를 포함해야 합니다(**SHOULD**).

`scope` 파라미터에는 누락된 스코프만이 아니라 작업에 필요한 **전체** 스코프 집합을 반드시 나열해야 합니다(**MUST**). 플랫폼은 이 전체 집합을 현재 토큰에 이미 부여된 스코프와 비교하고, 점진적 인가(incremental authorization)로 아직 없는 스코프만 요청합니다. 그러면 사용자가 이미 승인한 스코프에 다시 동의를 구하는 일이 없습니다.

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer realm="https://merchant.example.com",
                  error="insufficient_scope",
                  scope="dev.ucp.shopping.order:read dev.ucp.shopping.order:manage",
                  resource_metadata="https://merchant.example.com/.well-known/oauth-protected-resource"
Content-Type: application/json

{
  "messages": [
    {
      "type": "error",
      "code": "insufficient_scope",
      "content": "This operation requires scopes: dev.ucp.shopping.order:read, dev.ucp.shopping.order:manage",
      "severity": "requires_buyer_review"
    }
  ]
}
```

> **참고:** `identity_required`와 `insufficient_scope`는 의도적으로 구분됩니다. 플랫폼은 `insufficient_scope` 응답을 받았을 때 새 계정 연결 흐름을 처음부터 다시 시작하는 식으로 재시도해서는 안 됩니다(**MUST NOT**). 대신 이전에 부여된 스코프를 유지할 수 있도록 점진적 인가로 누락된 스코프만 반드시 요청해야 합니다(**MUST**).

## 선택적 인증

작업이 인증 없이 성공했더라도 현재 컨텍스트에서 인증할 수 있고 인증하면 얻는 것이 있다고 **비즈니스**가 **플랫폼**에 알리는 메커니즘입니다. 플랫폼은 이 신호를 사용자에게 보여 줄 수 있습니다.

### `identity_optional`

현재 컨텍스트에서 인증할 수 있고, 인증하면 의미 있는 추가 기능이 열린다면 비즈니스는 성공 응답에 이 정보(info) 수준 코드를 포함해야 합니다(**SHOULD**). `content` 필드는 비즈니스가 플랫폼에 전하는 가치 안내 문구를 담습니다(예: "Sign in for member pricing and personalized results.").

```json
{
  ...,
  "messages": [
    {
      "type": "info",
      "code": "identity_optional",
      "content": "Sign in for member pricing and personalized results."
    }
  ]
}
```

## 보안 고려 사항

- **PKCE.** 모든 인가 코드 흐름에서 PKCE(`S256`)는 필수입니다(REQUIRED). 평문 PKCE(`plain`)를 사용해서는 안 됩니다(**MUST NOT**). 비즈니스는 유효한 `code_verifier`가 없는 인가 코드 교환을 반드시 거부해야 합니다(**MUST**).
- **클라이언트 인증.** 비즈니스는 `token_endpoint_auth_methods_supported` ([RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414))로 클라이언트 인증 방식을 협상합니다. 기밀 클라이언트는 공유 비밀 유출 위험을 없앨 수 있도록 `client_secret_basic`보다 비대칭 방식(`private_key_jwt`, `tls_client_auth`)을 우선 사용해야 합니다(**SHOULD**). 공개 클라이언트 ([RFC 8252 §8.5](https://datatracker.ietf.org/doc/html/rfc8252#section-8.5)에 따른 네이티브, 데스크톱, 온디바이스 에이전트)는 `client_secret`을 기밀로 유지할 수 없으므로 반드시 `none`을 사용해야 합니다(**MUST**). 이런 클라이언트에서는 `S256` 방식의 PKCE가 인가 그랜트를 인증하는 소유 증명이 됩니다. 비즈니스는 네이티브 또는 에이전트 플랫폼을 지원할 때 `client_secret_basic`을 유일한 방식으로 요구해서는 안 됩니다(**MUST NOT**).
- **믹스업 공격 방지.** 플랫폼은 인가 응답의 `iss` 파라미터 ([RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207))를 반드시 검증해야 합니다(**MUST**). 비즈니스는 모든 인가 응답에 `iss`를 반드시 반환해야 합니다(**MUST**). `iss` 검증이 없으면 인가 서버 하나를 장악한 공격자가 피해자의 인가 코드를 다른 서버로 빼돌릴 수 있습니다.
- **인증 챌린지.** 비즈니스는 사용자 인증 작업의 `401`, `403` 응답에 [RFC 6750 §3](https://datatracker.ietf.org/doc/html/rfc6750#section-3)에 따른 `WWW-Authenticate: Bearer` 챌린지를 반드시 담아 보내야 합니다(**MUST**). 플랫폼은 인가 흐름을 결정할 때 구조화된 `scope`와 `error` 파라미터를 반드시 처리해야 합니다(**MUST**). `error_description`은 사람이 읽는 힌트일 뿐이므로 제어 흐름 결정에 사용해서는 안 됩니다(**MUST NOT**). 플랫폼이 챌린지를 올바른 인가 서버와 연관시킬 수 있도록 `realm` 파라미터는 비즈니스의 발급자 URI와 반드시 일치해야 합니다(**MUST**).
- **`redirect_uri` 정확성.** 비즈니스는 `redirect_uri`를 정확한 문자열 일치로 반드시 검사해야 합니다(**MUST**). 부분 일치나 접두사 일치로 구현하면 오픈 리디렉션과 토큰 탈취 취약점이 생기기 쉽습니다.
- **`issuer` 정확성.** RFC 8414 메타데이터의 `issuer` 값과 인가 응답의 `iss` 파라미터는 바이트 단위로 반드시 동일해야 합니다(**MUST**). 플랫폼은 비교 전에 정규화해서는 안 됩니다(**MUST NOT**). 정규화(예: 끝의 슬래시 제거)는 `iss` 검증을 우회하게 만드는 것으로 알려진 원인입니다.
- **전송 보안.** 플랫폼과 비즈니스 사이의 모든 통신은 TLS 1.2 이상의 HTTPS를 반드시 사용해야 합니다(**MUST**) ([RFC 6749 §1.6](https://datatracker.ietf.org/doc/html/rfc6749#section-1.6)).
- **`scopes_supported`.** 비즈니스는 RFC 8414 메타데이터에 `scopes_supported`를 반드시 채워야 합니다(**MUST**). 플랫폼은 동의 화면에 가서야 스코프 불일치를 발견하는 대신 일찍 실패할 수 있도록, 인가 흐름을 시작하기 전에 도출한 스코프 집합이 `scopes_supported`의 부분 집합인지 확인해야 합니다(**SHOULD**).
- **토큰 철회.** 사용자가 계정 연결을 해제하면 플랫폼은 비즈니스의 철회 엔드포인트(RFC 7009)에서 사용자 신원 토큰을 반드시 철회해야 합니다(**MUST**). 비즈니스는 이후 철회된 토큰을 제시하는 요청을 반드시 거부해야 합니다(**MUST**).

## 향후 확장성

이 명세는 v1의 범위를 의도적으로 비즈니스가 호스팅하는 OAuth 2.0으로 한정합니다. 스키마와 프로토콜은 향후 버전에서 추가 인증 패턴을 호환성을 깨지 않는 확장으로 받아들일 수 있게 설계되었습니다.

### 위임 신원 제공자와 메커니즘 확장성(`config.providers`)

향후 버전에서는 비즈니스가 역도메인 식별자를 키로 하는 `config.providers` 맵에 신뢰하는 신원 제공자를 선언할 수 있게 됩니다. 각 항목에는 기본값이 `oauth2`인 `type` 구분자가 있어, 위임 OAuth IdP와 향후 지갑 증명(wallet attestation)이나 검증 가능한 자격 증명 같은 OAuth가 아닌 메커니즘을 모두 이 하나의 확장 지점으로 다룰 수 있습니다.

위임 OAuth IdP(예: `com.google`, `com.shopify`)의 경우, 신뢰하는 제공자와 이미 OAuth 세션을 맺은 플랫폼은 새 브라우저 기반 OAuth 흐름을 시작하는 대신 비즈니스의 토큰 엔드포인트에 JWT 기반 인가 그랜트를 제시할 수 있습니다. 판매자가 N곳이라고 해서 동의 화면을 N번 거치지 않아도 되므로, 여러 판매자를 상대하는 에이전트 커머스에 유용합니다.

OAuth가 아닌 메커니즘의 경우, `type` 값이 `oauth2`가 아닌 항목으로 지갑 증명 같은 방식을 쓸 수 있습니다. 플랫폼은 비즈니스가 선호하는 순서대로 항목을 살펴 `type`을 지원하는 첫 항목을 선택합니다. TLS 암호 스위트 협상과 비슷한 방식입니다.

`config.providers`가 있으면 플랫폼은 해당 버전에 정의된 제공자 선택 및 신원 체이닝 흐름을 사용합니다. (v1처럼) `config.providers`가 없으면 플랫폼은 반드시 RFC 8414 디스커버리로 비즈니스 도메인에 대한 직접 OAuth 2.0을 사용해야 합니다(**MUST**).

**플랫폼을 위한 상위 호환 규칙:** `config`에 이 명세 버전에 정의되지 않은 필드(`providers` 또는 그 밖의 향후 필드)가 있으면 플랫폼은 해당 필드를 반드시 무시하고(**MUST**), 여기 정의된 대로 비즈니스 도메인에서 RFC 8414 디스커버리를 거쳐 OAuth 2.0을 사용해야 합니다. 이렇게 하면 명세가 발전하더라도 v1 플랫폼 구현이 계속 유효합니다.

## 예시

### 인가 서버 메타데이터

[RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414)에 따라 `/.well-known/oauth-authorization-server`에 호스팅되는 메타데이터 예시입니다.

```json
{
  "issuer": "https://merchant.example.com",
  "authorization_endpoint": "https://merchant.example.com/oauth2/authorize",
  "token_endpoint": "https://merchant.example.com/oauth2/token",
  "revocation_endpoint": "https://merchant.example.com/oauth2/revoke",
  "jwks_uri": "https://merchant.example.com/oauth2/jwks",
  "scopes_supported": [
    "dev.ucp.shopping.order:read",
    "dev.ucp.shopping.order:manage"
  ],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256"],
  "token_endpoint_auth_methods_supported": [
    "private_key_jwt",
    "tls_client_auth",
    "client_secret_basic",
    "none"
  ],
  "authorization_response_iss_parameter_supported": true,
  "service_documentation": "https://merchant.example.com/docs/oauth2"
}
```

참고: `authorization_response_iss_parameter_supported: true`는 [RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207) 지원을 알립니다. `code_challenge_methods_supported: ["S256"]`은 PKCE 지원을 나타냅니다. UCP를 준수하는 메타데이터에는 둘 다 반드시 있어야 합니다(**MUST**).

`token_endpoint_auth_methods_supported`에는 비즈니스가 받아들이는 모든 방식이 나열됩니다. 이 예시는 기밀 클라이언트용 비대칭 방식(`private_key_jwt`, `tls_client_auth`), 레거시 호환용 `client_secret_basic`, 공개 클라이언트 ([RFC 8252](https://datatracker.ietf.org/doc/html/rfc8252)에 따른 네이티브, 데스크톱, 온디바이스 에이전트)용 `none`을 선언합니다. `none`을 선언한다면 `S256` 방식의 PKCE가 필수입니다. 공개 클라이언트를 지원하지 않는 비즈니스는 `none`을 생략할 수 있습니다(**MAY**).

### 비즈니스 프로필(`/.well-known/ucp`)

`config.scopes`의 구성은 비즈니스의 정책을 반영합니다.

#### B2C 소매업체

공개 카탈로그와 게스트 체크아웃(스코프 불필요)을 허용하고, 사용자에 귀속된 주문 작업은 제한하는 경우입니다.

```json
{
  "ucp": {
    "capabilities": {
      "dev.ucp.common.identity_linking": [{
        "version": "Working Draft",
        "spec": "https://ucp.dev/specification/identity-linking",
        "schema": "https://ucp.dev/schemas/common/identity_linking.json",
        "config": {
          "scopes": {
            "dev.ucp.shopping.order:read":    {},
            "dev.ucp.shopping.order:manage":  {}
          }
        }
      }]
    }
  }
}
```

**이 구성의 해석:**

- `dev.ucp.shopping.order:read`와 `:manage` — 나열됨 → 얻으려면 사용자 인증 필요.
- 카탈로그, 체크아웃, 장바구니 등 나머지 전부 — 나열되지 않음 → 사용자 인증 스코프 불필요. 공개/에이전트 인증 접근. 그래도 개인화, 저장된 주소와 자격 증명, 로열티 가격 같은 추가 기능을 쓰려고 사용자가 연결을 요청하거나 에이전트가 연결을 제안할 수 있습니다.

#### B2B 도매업체

게스트 체크아웃이 없어 모든 거래에 인증된 사용자가 필요한 경우입니다.

```json
{
  "ucp": {
    "capabilities": {
      "dev.ucp.common.identity_linking": [{
        "version": "Working Draft",
        "spec": "https://ucp.dev/specification/identity-linking",
        "schema": "https://ucp.dev/schemas/common/identity_linking.json",
        "config": {
          "scopes": {
            "dev.ucp.shopping.checkout:manage":  {},
            "dev.ucp.shopping.order:read":       {},
            "dev.ucp.shopping.order:manage":     {}
          }
        }
      }]
    }
  }
}
```

**차이점:** 이번에는 `dev.ucp.shopping.checkout:manage`가 나열되어 있습니다. B2C 예시는 체크아웃을 제한하지 않으므로 누구나 게스트 세션을 시작할 수 있습니다. 이 판매자는 생성, 업데이트, 완료, 취소 등 모든 체크아웃 작업에 사용자 인증을 요구합니다.

사용자에게 B2B 자격이 있는지, 어떤 가격을 보는지, 어떤 결제 조건이 적용되는지는 추가 스코프가 아니라 판매자가 런타임에 확인하는 사용자 속성입니다.

### 엔드투엔드 예시

**구성:** 플랫폼(AI 쇼핑 에이전트) + 비즈니스(위 예시의 B2C 소매업체).

**협상된 기능:** `dev.ucp.shopping.checkout`, `dev.ucp.shopping.order`, `dev.ucp.common.identity_linking`.

**1단계 — 스코프 도출.** 플랫폼이 비즈니스의 `config.scopes`를 읽습니다.

- `dev.ucp.shopping.order:read`(주문 내역 조회)
- `dev.ucp.shopping.order:manage`(취소/반품)

사용자는 에이전트가 자신을 대신해 주문 내역을 조회하고 주문을 취소할 수 있기를 원하므로, 플랫폼은 두 스코프를 모두 요청합니다.

도출된 스코프 집합: `dev.ucp.shopping.order:read dev.ucp.shopping.order:manage`

**2단계 — 디스커버리.** 플랫폼이 `https://merchant.example.com/.well-known/oauth-authorization-server`를 가져와 `2xx`를 받고, `authorization_endpoint`와 `token_endpoint`를 추출합니다. 두 스코프가 모두 `scopes_supported`에 있는지 확인합니다.

**3단계 — 인가 요청.** 플랫폼이 PKCE 쌍(`code_verifier`, `code_challenge`)을 생성하고 사용자를 다음 주소로 보냅니다.

```text
GET https://merchant.example.com/oauth2/authorize
  ?response_type=code
  &client_id=platform-client-id
  &redirect_uri=https://agent.example.com/callback
  &scope=dev.ucp.shopping.order:read dev.ucp.shopping.order:manage
  &code_challenge=<S256-hash>
  &code_challenge_method=S256
  &state=<random>
```

실제 요청에서는 `redirect_uri`의 예약 문자와 스코프 토큰의 `:`를 퍼센트 인코딩해야 합니다. 여기서는 가독성을 위해 디코딩된 형태로 표시했습니다.

**4단계 — 인가 응답.** 사용자가 인증하고 동의합니다. 비즈니스가 다음 주소로 리디렉션합니다.

```text
https://agent.example.com/callback
  ?code=<auth-code>
  &state=<random>
  &iss=https://merchant.example.com
```

플랫폼은 `state`가 일치하는지, `iss`가 발견한 `issuer`와 같은지 검증합니다.

**5단계 — 토큰 교환.** 플랫폼이 토큰 엔드포인트를 호출합니다.

```http
POST https://merchant.example.com/oauth2/token
Authorization: Basic <base64(client_id:client_secret)>
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=<auth-code>
&redirect_uri=https://agent.example.com/callback
&code_verifier=<verifier>
```

비즈니스는 `code_verifier`를 저장된 `code_challenge`와 대조해 검증하고 다음을 반환합니다.

```json
{
  "access_token": "<token>",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "<refresh>",
  "scope": "dev.ucp.shopping.order:read dev.ucp.shopping.order:manage"
}
```

이제 플랫폼은 이후 사용자 인증 기능 엔드포인트로 보내는 요청에 `Authorization: Bearer <token>`을 포함합니다.
