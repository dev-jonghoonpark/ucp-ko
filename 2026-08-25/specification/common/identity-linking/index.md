# 신원 연결 기능

- **기능(Capability) 이름:** `dev.ucp.common.identity_linking`
- **스키마:** `https://ucp.dev/2026-08-25/schemas/common/identity_linking.json`

## 개요

신원 연결(Identity Linking) 기능을 사용하면 **플랫폼**이 **비즈니스**(신뢰 당사자, relying party) 사이트에서 **사용자**를 대신해 작업할 수 있는 인가를 받을 수 있습니다.

이 연결은 사용자 인증 기반 커머스 경험의 토대입니다. 로열티 혜택, 개인화된 제안, 저장된 주소, 위시리스트, 주문 내역에 접근하려면 신원 연결이 필요합니다. 신원 연결이 없어도 기능은 공개 또는 에이전트 인증 수준에서 그대로 동작합니다. 신원 연결은 경험을 한 단계 끌어올릴 뿐, 접근을 막는 관문이 아닙니다.

**이 명세는 인가에 [OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)을 사용합니다.** 비즈니스 도메인을 상대로 하는 직접 OAuth 2.0([디스커버리](#discovery) 이용)은 언제나 사용할 수 있습니다. 비즈니스가 `config.providers`에 신뢰하는 외부 신원 제공자(IdP)를 선언했다면, 플랫폼은 직접 OAuth 대신 [가속 IdP 흐름](#accelerated-idp-flow)으로 제공자의 신원을 체이닝할 수 있습니다(**MAY**). 적합한 상위(upstream) 토큰을 이미 보유하고 있다면 브라우저 기반 흐름을 건너뛸 수 있습니다.

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

### 비즈니스가 채우는 응답 값

사용자 인증된 요청이라면 비즈니스는 플랫폼이 보낸 값을 되돌려 주는 데 그치지 않고, 자신이 소유한 응답 필드를 그 사용자에 대해 저장해 둔 상태로 채울 수 있습니다(**MAY**).

요청 어노테이션은 플랫폼이 어떤 필드를 보낼 수 있는지만 정할 뿐, 그 값의 의미상 소유권을 정하지는 않습니다. `ucp_request: "omit"`으로 어노테이션된 필드는 요청에서 빠지며, 응답에 허용되고 값이 있다면 비즈니스가 채웁니다(예: `totals`, `messages`, `order`). 해당 작업에서 `optional`로 어노테이션되었거나 어노테이션이 없는 필드는 플랫폼이 요청에 넣어 보낼 수도 있습니다. 응답 스키마와 필드의 의미 계약이 비즈니스 소유의 사용자 상태를 허용한다면, 비즈니스는 응답의 그 필드를 저장된 상태로 채울 수 있습니다(**MAY**)(예: `buyer`, `payment.instruments[]`).

신원 연결은 이런 값을 채우는 데 필요한 인증된 사용자 컨텍스트를 제공합니다. 비즈니스는 해당 작업을 제한하는 스코프를 따르는 범위에서 사용자의 로열티 멤버십을 `loyalty`에([로열티](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/extensions/loyalty/index.md) 참고), 저장된 결제 수단을 `payment.instruments[]`에([체크아웃](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/checkout/index.md) 참고), 구매자 프로필 데이터를 `buyer`에 담아 반환할 수 있습니다(**MAY**).

비즈니스는 요청이 사용자 인증되고 해당 작업에 인가된 경우가 아니라면 저장된 사용자별 상태를 반환해서는 안 됩니다(**MUST NOT**). 인증된 사용자에게 속하고 현재 거래에 적절한 값만 반환해야 하며(**MUST**), 그 거래에 필요한 데이터로 범위를 제한해야 합니다(**SHOULD**). 플랫폼은 반환된 비즈니스 소유 식별자를 불투명하고 비즈니스 범위에 한정된 값으로 취급해야 하며(**MUST**), 표시용 값에서 원시 자격 증명을 추론해서는 안 됩니다(**MUST NOT**).

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

- `config.providers`가 있으면 플랫폼은 [가속 IdP 흐름](#accelerated-idp-flow)으로 나열된 제공자의 신원을 체이닝할 수 있습니다(**MAY**). 나열된 제공자 중 지원되거나 적합한 것이 없으면 플랫폼은 반드시 [디스커버리](#discovery)를 이용한 비즈니스 도메인 직접 OAuth로 대체해야 합니다(**MUST**)([신원 제공자](#identity-providers) 참고).

- 비즈니스와 신원 체이닝을 시작하기 전에 플랫폼은 사용 가능한 신원 제공자 중 하나를 사용자가 선택할 수 있게 하고, 어느 제공자의 신원이 비즈니스와 공유되는지 알려야 합니다(**SHOULD**).

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
- 신뢰하는 외부 신원 제공자를 `config.providers`에 선언할 수 있습니다(**MAY**) ([신원 제공자](#identity-providers) 참고). 비즈니스는 명시적으로 신뢰하는 제공자만 나열해야 하며(**MUST**), 자신의 인가 서버를 나열해서는 안 됩니다(**MUST NOT**).
- 비즈니스가 `config.providers`에 `type: oauth2`인 외부 신원 제공자를 나열한다면, 해당 IdP의 JWT 인가 그랜트를 받아들일 수 있도록 토큰 엔드포인트에서 JWT bearer 어서션 그랜트 타입 ([RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523))을 반드시 지원해야 하며(**MUST**), RFC 8414 메타데이터의 `grant_types_supported`에 `urn:ietf:params:oauth:grant-type:jwt-bearer`를 반드시 포함해야 합니다(**MUST**).
- 사용자에게 아직 계정이 없으면 계정 생성 흐름을 제공하거나, 온보딩 흐름을 가리키는 `continue_url`을 `identity_required` 오류 응답에 담아 반환해야 합니다(**SHOULD**)([오류 처리](#error-handling) 참고).
- [스코프](#scopes) 섹션에 정의된 표준 UCP 스코프를 반드시 지원해야 합니다(**MUST**).
- `/.well-known/oauth-protected-resource`에 보호 리소스 메타데이터 ([RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728))를 게시하고, `WWW-Authenticate` 챌린지의 `resource_metadata` 파라미터로 이 문서를 가리켜야 합니다(**SHOULD**). 이렇게 하면 플랫폼은 도메인 관례에 기대지 않고도 리소스를 보호하는 인가 서버를 찾을 수 있고, 앞으로 나올 위임 도메인 관례에도 대비할 수 있습니다. 인가 서버가 비즈니스 도메인에 있지 않다면 비즈니스는 이 메타데이터를 반드시 게시해야 합니다(**MUST**).
- 플랫폼에 철회와 계정 상태 변경을 알리기 위해 [OpenID RISC Profile 1.0](https://openid.net/specs/openid-risc-1_0-final.html)을 지원해야 합니다(**SHOULD**).

## 디스커버리

UCP 디스커버리는 3단계 파이프라인입니다.

**1단계 — 인가 서버(AS) 발급자 확인.** 플랫폼은 [RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728)에 따라 비즈니스의 보호 리소스 메타데이터를 가져오고, `authorization_servers`에서 선택한 항목을 AS 발급자로 사용합니다. AS 발급자는 비즈니스 도메인과 다른 오리진에 호스팅될 수 있습니다(**MAY**). 비즈니스가 보호 리소스 메타데이터를 게시하지 않으면 AS 발급자는 기본적으로 비즈니스 도메인이 됩니다(단일 호스트 배포).

**2단계 — AS 메타데이터 가져오기.** 플랫폼은 1단계에서 얻은 발급자를 사용해 엄격한 2단계 계층 구조로 인가 서버 메타데이터를 확인합니다. well-known URL은 [RFC 8414 §3.1](https://datatracker.ietf.org/doc/html/rfc8414#section-3.1)에 따라 구성합니다(well-known 세그먼트를 뒤에 덧붙이지 않고 호스트와 발급자 경로 사이에 끼워 넣습니다).

1. **RFC 8414(1순위):** `https://{host}/.well-known/oauth-authorization-server{path}`를 가져옵니다.

   - `2xx` 응답: 이 메타데이터를 사용합니다. 디스커버리 완료.
   - `404 Not Found`: 2단계로 진행합니다.
   - 그 밖의 비-2xx 응답, 네트워크 오류, 타임아웃: 반드시 중단해야 합니다(**MUST**). 2단계로 진행해서는 안 됩니다(**MUST NOT**).

1. **OIDC 디스커버리(대체):** `{issuer}/.well-known/openid-configuration`을 가져옵니다.

   - `2xx` 응답: 이 메타데이터를 사용합니다. 디스커버리 완료.
   - 비-2xx 응답, 네트워크 오류, 타임아웃: 반드시 중단해야 합니다(**MUST**).

플랫폼은 1단계에서 `404` 이외의 오류가 발생했을 때 아무 표시 없이 다음 단계로 넘어가서는 안 됩니다(**MUST NOT**).

**3단계 — 발급자 검증.** 발견한 메타데이터의 `issuer` 값은 1단계에서 선택한 AS 발급자와 바이트 단위로 반드시 일치해야 합니다(**MUST**) ([RFC 8414 §3.3](https://datatracker.ietf.org/doc/html/rfc8414#section-3.3)에 따름). 플랫폼은 비교 전에 정규화(예: 끝의 슬래시 제거)를 해서는 안 됩니다(**MUST NOT**).

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

## 신원 제공자

`config.providers` 맵에는 비즈니스가 신뢰하는 외부 신원 제공자를 선언합니다. 비즈니스는 [가속 IdP 흐름](#accelerated-idp-flow)에서 이 제공자들이 JWT bearer 어서션으로 체이닝한 신원을 받아들입니다. 각 키는 IdP 네임스페이스를 식별하며 메커니즘 항목의 배열에 대응합니다. IdP는 키 하나 아래에 토큰 획득 메커니즘을 여러 개 제공할 수 있습니다(**MAY**). 이 맵은 비즈니스 도메인을 상대로 언제나 사용할 수 있는 직접 OAuth 경로([디스커버리](#discovery) 참고)에 덧붙는 메타데이터이며, 체이닝 경로에서는 닫힌 허용 목록 역할을 합니다. 즉, 비즈니스는 `iss`가 나열된 `oauth2` 메커니즘 항목과 일치하지 않는 JWT 인가 그랜트를 반드시 거부해야 합니다(**MUST**)([비즈니스 토큰 발급](#business-token-issuance) 참고).

- **없거나 비어 있는 경우:** 플랫폼은 [디스커버리](#discovery)를 거쳐 비즈니스 도메인을 상대로 직접 OAuth를 수행합니다.
- **있는 경우:** 플랫폼은 자신이 지원하는 `type`의 메커니즘 항목을 선택해 [가속 IdP 흐름](#accelerated-idp-flow)으로 신원을 체이닝할 수 있습니다(**MAY**). 보통은 유효한 상위 토큰을 이미 보유한 IdP에 속한 항목을 고릅니다. 나열된 메커니즘 중 지원되거나 적합한 것이 없으면 플랫폼은 반드시 비즈니스 도메인 직접 OAuth로 대체해야 합니다(**MUST**).
- **자기 자신 나열 금지.** 비즈니스는 자신의 인가 서버를 `config.providers`에 나열해서는 안 됩니다(**MUST NOT**). 자기 자신에게 체이닝하는 것은 의미가 없고(같은 서버가 어서션을 발급하고 검증하게 됩니다), 직접 OAuth는 이미 [디스커버리](#discovery)로 사용할 수 있습니다. 플랫폼은 `auth_url`이 비즈니스 자신의 발급자 URI와 일치하는 `oauth2` 메커니즘 항목을 반드시 무시해야 합니다(**MUST**).

### 제공자 구성

`config.providers`의 각 키는 IdP 네임스페이스를 나타내는 역방향 도메인 식별자이며, 값은 메커니즘 항목의 배열입니다.

제공자 키는 스키마가 있는 엔티티가 아니라 역방향 도메인 **식별자**입니다. `schema` URL을 선언하지 않으므로 [권한 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#authority-binding)(엔티티의 `schema` URL을 네임스페이스 권한 주체에 결합하는 규칙)은 여기에 적용되지 않습니다. 제공자의 신뢰 앵커는 `auth_url`이며, 아래 디스커버리 규칙이 적용됩니다. `auth_url`을 제공자의 네임스페이스 권한 주체에 결합하는 방안은 향후 보안 강화책으로 별도로 검토하고 있습니다.

각 항목은 `type`으로 구분됩니다.

| 필드              | 타입             | 필수                | 설명                                                                                                                                                           |
| ----------------- | ---------------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type`            | string           | 예                  | 제공자 메커니즘 구분자입니다. 이 버전에 정의된 타입은 `oauth2`뿐이며, 향후 버전에서 호환성을 깨지 않는 확장으로 타입을 추가로 정의할 수 있습니다(**MAY**).     |
| `auth_url`        | string (URI)     | `oauth2`이면 예     | 인가 서버 메타데이터 디스커버리의 기준 URL입니다.                                                                                                              |
| `required_claims` | array of strings | `oauth2`이면 아니요 | 비즈니스가 JWT 인가 그랜트에 요구하는 OIDC Core §5.1 클레임 이름(예: `email`)입니다. 선택적인 사전 필터링 힌트입니다. [제공자 선택](#provider-selection) 참고. |

`type` 값은 닫힌 열거형이 아니라 열린 문자열입니다. 플랫폼은 지원하지 않는 `type`의 제공자 항목을 비즈니스 구성 오류로 거부하지 말고, 걸러낸 것으로 취급해야 합니다(**MUST**)([제공자 선택](#provider-selection) 참고).

`oauth2` 제공자라면 플랫폼은 `auth_url`을 발급자로 보고 [디스커버리](#discovery) 2단계와 동일한 2단계 메타데이터 계층 구조(§3.1 경로 삽입을 적용한 RFC 8414 1순위, 404일 때만 OIDC 대체)를 사용해 `auth_url`에서 인가 서버 메타데이터를 반드시 찾아야 합니다(**MUST**). 보호 리소스 단계는 적용되지 않습니다. `auth_url`이 이미 IdP 발급자이기 때문입니다. 비즈니스는 [비즈니스 토큰 발급](#business-token-issuance)에 따라 JWT 그랜트의 `iss`를 이 값과 대조해 검증합니다.

### 제공자 선택

플랫폼은 비즈니스의 `config.providers` 맵에 있는 `(provider key, mechanism entry)` 쌍을 순회하며 자신이 지원하는 항목을 선택합니다. 보통은 유효한 상위 토큰을 이미 보유한 IdP에 속한 항목을 골라 [가속 IdP 흐름](#accelerated-idp-flow)을 사용합니다. 같은 제공자 키 아래의 메커니즘 항목은 서로 대안 관계이므로 플랫폼은 그중 어느 것과 일치해도 됩니다.

메커니즘이 토큰·스코프 모델에 참여하는지는 그 메커니즘의 `type`이 결정합니다. `oauth2` 타입은 토큰 교환으로 신원을 체이닝하며, 비즈니스가 발급하는 토큰의 스코프에 기여합니다. 다른 타입은 스코프에 전혀 기여하지 않을 수도 있습니다(**MAY**). `providers`에 있다고 해서 스코프나 토큰 발급 모델에 참여한다는 뜻은 아닙니다. 선택 여부는 해당 타입이 토큰을 발급하는지와 무관하게, 플랫폼이 메커니즘의 `type`을 *지원하는지*, 그리고 적합한 상위 신원을 보유하고 있는지(또는 얻을 수 있는지)에 달려 있습니다.

메커니즘 항목이 `required_claims`를 선언했는데 플랫폼의 상위 신원에 나열된 클레임이 하나라도 없다면(예: 상위 토큰에 `email`이 없음) 선택 단계에서 그 항목을 걸러내야 합니다(**SHOULD**). 그러면 토큰 엔드포인트에서 거부될 체이닝을 헛되이 시도하지 않아도 됩니다. 그렇더라도 사후 강제는 여전히 필요합니다 ([토큰 엔드포인트의 체이닝 오류](#chaining-errors-at-the-token-endpoint) 참고). 모든 비즈니스가 `required_claims`를 선언하지는 않으며, 이 힌트는 클레임의 존재만 나타내기 때문입니다. 값에 대한 제약(예: `email_verified=true` 요구)은 사후에 강제됩니다.

### 프로필 예시

체이닝용으로 외부 IdP를 신뢰하면서 직접 OAuth 흐름(디스커버리로 언제나 사용 가능)도 받아들이는 비즈니스의 예입니다.

```json
{
  "ucp": {
    "version": "2026-08-25",
    "services": {},
    "capabilities": {
      "dev.ucp.common.identity_linking": [{
        "version": "2026-08-25",
        "spec": "https://ucp.dev/2026-08-25/specification/common/identity-linking/",
        "schema": "https://ucp.dev/2026-08-25/schemas/common/identity_linking.json",
        "config": {
          "providers": {
            "app.example.login": [
              {
                "type": "oauth2",
                "auth_url": "https://accounts.example-login.app/",
                "required_claims": ["email"]
              }
            ]
          },
          "scopes": {
            "dev.ucp.shopping.order:read":   {},
            "dev.ucp.shopping.order:manage": {}
          }
        }
      }]
    },
    "payment_handlers": {}
  }
}
```

플랫폼이 `app.example.login`의 유효한 토큰을 이미 보유하고 있다면 가속 IdP 흐름을 사용할 수 있습니다. 그렇지 않으면 [디스커버리](#discovery)를 거쳐 비즈니스 도메인을 상대로 표준 [계정 연결 흐름](#account-linking-flow)을 수행합니다.

## 가속 IdP 흐름

플랫폼이 신뢰할 수 있는 IdP와 사용자 신원을 연결한 뒤에는 브라우저 리디렉션 없이 그 신원을 새 비즈니스로 체이닝할 수 있습니다. 플랫폼은 IdP에서 **JWT 인가 그랜트**를 얻어 비즈니스의 토큰 엔드포인트에 제시합니다. 비즈니스는 그랜트를 검증하고 자신의 권한으로 자체 액세스 토큰을 발급합니다.

이 흐름은 [draft-ietf-oauth-identity-chaining](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-identity-chaining-08)의 신원 및 인가 체이닝 패턴을 프로파일링한 것입니다. UCP는 JWT 인가 그랜트에 기반 RFC보다 더 엄격한 조건을 둡니다. `aud`는 반드시 단일 값 URI여야 하고 고유한 `jti`가 함께 있어야 합니다(**MUST**) ([JWT 인가 그랜트](#jwt-authorization-grant) 참고).

### 흐름

1. 플랫폼은 비즈니스의 신원 연결 기능에서 `config.providers`를 발견하고, 유효한 토큰을 이미 보유한 제공자를 선택합니다.
1. 플랫폼은 IdP의 토큰 엔드포인트에서 토큰 교환 ([RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693))으로 IdP에 JWT 인가 그랜트를 요청합니다.
   - `grant_type`: `urn:ietf:params:oauth:grant-type:token-exchange`
   - `subject_token`: 플랫폼이 이미 보유한 IdP 액세스 토큰
   - `subject_token_type`: `urn:ietf:params:oauth:token-type:access_token`
   - `resource` 및/또는 `audience`: 비즈니스 인가 서버의 발급자 URI입니다. 플랫폼은 둘 중 하나 이상을 반드시 포함해야 합니다(**MUST**). [RFC 8693 §2.1](https://datatracker.ietf.org/doc/html/rfc8693#section-2.1)은 두 파라미터 모두에 URI 값을 허용하며, 어느 쪽을 받아들이는지는 IdP 구현마다 다릅니다. IdP는 이 값을 결과 그랜트의 `aud` 클레임에 대응시킵니다. 둘 다 보낸다면 반드시 같은 값을 담아야 합니다(**MUST**).
   - `requested_token_type`: `urn:ietf:params:oauth:token-type:jwt`
1. IdP는 subject 토큰을 검증하고, 플랫폼에 대상 비즈니스용 그랜트를 요청할 권한이 있는지 확인한 뒤, `issued_token_type`을 `urn:ietf:params:oauth:token-type:jwt`로 설정한 단기 JWT 인가 그랜트를 반환합니다.
1. 플랫폼은 JWT bearer 어서션 그랜트 ([RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523))로 비즈니스의 토큰 엔드포인트에 그랜트를 제시합니다.
   - `grant_type`: `urn:ietf:params:oauth:grant-type:jwt-bearer`
   - `assertion`: JWT 인가 그랜트
   - `scope`: 도출한 스코프 집합([스코프 도출](#scope-derivation) 참고)
1. 비즈니스는 그랜트를 검증하고 사용자 신원을 확인한 뒤, 자신의 권한으로 액세스 토큰을 발급합니다.
1. 플랫폼은 이후 요청에서 비즈니스가 발급한 토큰을 `Authorization: Bearer <access_token>`으로 사용합니다.

플랫폼은 IdP 원본 토큰을 비즈니스에 직접 제시해서는 안 됩니다(**MUST NOT**). 신원 체이닝을 거치면 각 비즈니스가 올바른 대상(audience) 바인딩과 스코프 정책을 적용해 자신의 권한으로 토큰을 발급하게 됩니다.

### JWT 인가 그랜트

JWT 인가 그랜트는 IdP가 발급하는 서명된 JWT ([RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519))로, 특정 비즈니스에서 사용할 사용자 신원을 단언합니다. 액세스 토큰이 **아닙니다**. 플랫폼이 액세스 토큰을 받으려고 비즈니스의 인가 서버에 제시하는 단기 자격 증명입니다.

그랜트는 반드시 [RFC 7523 §3](https://datatracker.ietf.org/doc/html/rfc7523#section-3)을 준수해야 하며(**MUST**), UCP 고유의 제약이 두 가지 추가됩니다.

- `aud`는 배열이 아니라 반드시 단일 값(비즈니스의 AS 발급자 URI)이어야 합니다(**MUST**). 체이닝은 그랜트 하나당 비즈니스 하나를 대상으로 합니다.
- 비즈니스가 1회용 재사용 방지를 강제할 수 있도록 `jti`가 반드시 있어야 합니다(**MUST**)(RFC 7523은 MAY로 규정)([보안 고려 사항](#security-considerations) 참고).

`exp`는 `iat` 이후 60초를 넘지 않아야 합니다(**SHOULD**). IdP는 인가 컨텍스트(동의 기록, 사용자 속성 등)를 전달하려고 추가 클레임을 포함할 수 있습니다(**MAY**).

### 비즈니스 토큰 발급

토큰 엔드포인트에서 JWT 인가 그랜트를 받으면, 비즈니스의 인가 서버는 [RFC 7523 §3](https://datatracker.ietf.org/doc/html/rfc7523#section-3)에 따라 어서션을 반드시 검증해야 하며(**MUST**), 다음 UCP 고유 요구 사항을 함께 적용합니다.

- `iss`는 `config.providers`에 나열된 `oauth2` 메커니즘 항목의 `auth_url`과 반드시 일치해야 합니다(**MUST**).
- `aud`는 비즈니스 자신의 AS 발급자 URI와 반드시 정확히 일치해야 합니다(**MUST**).
- JWT 서명은 IdP의 `jwks_uri`로 반드시 검증해야 합니다(**MUST**). JWKS를 가져올 수 없으면 비즈니스는 반드시 실패 시 차단(fail closed)해야 합니다(**MUST**).

어서션이 검증되면 비즈니스는 `sub`로 사용자를 확인하고(자동 프로비저닝 허용), 액세스 토큰을 발급합니다. 이 토큰의 범위는 요청된 스코프 중 그랜트의 클레임(`acr`, `auth_time`, `amr` 등)이 스코프별 정책을 충족하는 부분 집합으로 한정됩니다. 충족되는 요청 스코프가 하나도 없으면 비즈니스는 [RFC 6749 §5.2](https://datatracker.ietf.org/doc/html/rfc6749#section-5.2)에 따라 반드시 `invalid_scope`를 반환해야 합니다(**MUST**). 플랫폼은 IdP에 단계 상향(step-up) 그랜트를 요청하거나 직접 OAuth를 수행해 복구합니다. 사용자 상호작용(약관 동의, 온보딩)이 필요하면 비즈니스는 그랜트를 반드시 `invalid_grant`로 거부해야 합니다(**MUST**) ([토큰 엔드포인트의 체이닝 오류](#chaining-errors-at-the-token-endpoint) 참고). 플랫폼은 대화형 제공자를 상대로 [계정 연결 흐름](#account-linking-flow)을 수행해 복구합니다.

**사용자 확인을 위한 클레임.** 비즈니스에는 `sub` 외에도 추가 클레임이 필요한 경우가 많습니다. 사용 가능한 연락처 식별자로 새 계정을 프로비저닝하거나, 기존 계정과 일치할 가능성이 있을 때 UX 힌트를 보여 주려면 이런 클레임이 있어야 합니다. IdP별 안정적인 신원 키는 `(iss, sub)`입니다. `oauth2` 제공자라면 IdP는 가능하고 사용자가 동의했다면 관련 [OIDC Core §5.1 표준 클레임](https://openid.net/specs/openid-connect-core-1_0-31.html#StandardClaims), 특히 `email`과 `email_verified`를 JWT 인가 그랜트에 포함해야 합니다(**SHOULD**). 비즈니스는 이메일이나 다른 클레임을 대조해 IdP 간 계정을 자동으로 연결하지 않아야 하며(**SHOULD NOT**), 신원을 병합하기 전에 사용자가 직접 개입하는 연결(사용자가 기존 계정에 인증하는 방식)을 요구해야 합니다(**SHOULD**). [보안 고려 사항](#security-considerations)을 참고하세요.

### 토큰 엔드포인트의 체이닝 오류

검증 실패에는 표준 OAuth 2.0 토큰 엔드포인트 오류 형식 ([RFC 6749 §5.2](https://datatracker.ietf.org/doc/html/rfc6749#section-5.2))을 사용합니다. JWT 그랜트 고유의 실패(서명, `iss`, `aud`, `exp`, `jti` 재사용, 인식되지 않는 제공자, 사용자 상호작용 필요)는 [RFC 7523 §3.1](https://datatracker.ietf.org/doc/html/rfc7523#section-3.1)에 따라 `invalid_grant`에 대응합니다. 비즈니스는 진단을 돕거나 온보딩 문서를 안내하려고 `error_description`과 `error_uri`를 포함할 수 있습니다(**MAY**). 플랫폼은 `error_description`을 기계가 읽을 수 있는 값으로 취급해서는 안 됩니다(**MUST NOT**).

**누락되었거나 불충분한 클레임.** JWT 인가 그랜트에 비즈니스가 요구하는 클레임(예: 계정 확인용 `email`)이 없으면 비즈니스는 반드시 `invalid_grant`로 거부해야 합니다(**MUST**). 비즈니스는 사람이 진단할 수 있도록 누락된 클레임을 명시한 `error_description`(예: `"missing required claim: email"`)을 포함할 수 있습니다(**MAY**). 플랫폼은 자동 복구용으로 이 값을 파싱해서는 안 되며(**MUST NOT**), 비즈니스가 사용자에게 누락된 정보를 요청할 수 있는 직접 OAuth로 대체해야 합니다(**SHOULD**). 비즈니스는 플랫폼이 사전 필터링할 수 있도록 `required_claims` ([제공자 구성](#provider-configuration) 참고)로 표준 클레임 요구 사항을 알려야 하며(**SHOULD**), OIDC Core §5.1을 넘어서는 요구 사항은 개발자용 자료에 문서화해야 합니다(**SHOULD**).

### 토큰 수명 주기

JWT bearer 어서션 그랜트는 장기 세션을 만들지 않습니다. 비즈니스는 응답으로 리프레시 토큰을 발급하지 않는 것이 좋습니다(**SHOULD NOT**). 리프레시 토큰은 IdP 세션보다 오래 살아남아, 사용자가 IdP 관계를 철회한 뒤에도 접근을 계속 허용하게 되기 때문입니다 ([draft-ietf-oauth-identity-chaining §5.4](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-identity-chaining-08#section-5.4)). 비즈니스가 발급한 토큰이 만료되면 플랫폼은 IdP에서 새 JWT 그랜트를 받아 다시 제시합니다.

철회는 체인을 따라 전파되지 않습니다. 연결을 해제할 때 플랫폼은 IdP와 비즈니스 *양쪽* 계층 모두에서 철회 엔드포인트 ([RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009))를 호출해야 합니다(**SHOULD**).

## IdP 요구 사항

이 섹션의 요구 사항은 `type: oauth2`인 신원 제공자에 적용됩니다. 다른 제공자 타입은 각자 디스커버리 및 증명 제시 요구 사항을 정의합니다([향후 확장성](#future-extensibility) 참고).

`config.providers`에 나열된 `type: oauth2` 신원 제공자는 [RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414) 또는 OpenID Connect Discovery 방식으로 인가 서버 메타데이터를 반드시 게시해야 합니다(**MUST**). 메타데이터에는 반드시 다음이 포함되어야 합니다(**MUST**).

- `revocation_endpoint` — [토큰 수명 주기](#token-lifecycle) 섹션에 따른 토큰 철회 지원용
- `jwks_uri` — IdP가 발급한 JWT 인가 그랜트의 서명을 비즈니스가 검증하는 데 사용
- `grant_types_supported`의 `urn:ietf:params:oauth:grant-type:token-exchange` — 가속 IdP 흐름 사용을 위해 필요

JWT 인가 그랜트를 위한 토큰 교환 요청을 처리할 때 IdP는 반드시 다음을 수행해야 합니다(**MUST**).

- 플랫폼을 인증하고, 플랫폼이 subject 토큰을 제시할 권한이 있는지 확인합니다 ([RFC 8693 §2.1](https://datatracker.ietf.org/doc/html/rfc8693#section-2.1)).
- 대상 비즈니스(`resource` 및/또는 `audience`로 식별하며, 둘 다 보낸다면 반드시 같은 값을 담아야 합니다(**MUST**) — [흐름](#flow) 참고)가 알려진 신뢰 당사자이고 사용자가 그 비즈니스와의 신원 공유를 인가했는지 확인합니다. IdP는 사용자가 인가하지 않은 비즈니스용 그랜트를 발급해서는 안 됩니다(**MUST NOT**).
- [JWT 인가 그랜트](#jwt-authorization-grant) 요구 사항을 준수하는 JWT 인가 그랜트를 발급합니다.
- `issued_token_type`을 `urn:ietf:params:oauth:token-type:jwt`로 반환합니다.

사용자가 공유에 동의했다면 IdP는 비즈니스의 계정 프로비저닝과 UX 힌트를 지원할 수 있도록 JWT 인가 그랜트에 OIDC Core §5.1 표준 클레임을 채워야 합니다(**SHOULD**). 해당된다면 최소한 `email`과 `email_verified`는 포함합니다. 표준 클레임은 참고용이며, IdP별 안정적 식별자는 여전히 `(iss, sub)`입니다.

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

선언된 스코프는 신원 경로와 관계없이 반드시 똑같이 적용되어야 합니다(**MUST**). 플랫폼이 직접 OAuth로 사용자 신원을 얻었든 [가속 IdP 흐름](#accelerated-idp-flow)으로 얻었든 같은 `config.scopes` 맵이 스코프 가용성을 결정하고, 어떤 어서션이 스코프를 충족하는지는 스코프별 정책이 결정합니다. 예를 들어 `min_acr`을 적용하는 비즈니스는 경로와 관계없이 반드시 같은 기준을 적용해야 합니다(**MUST**).

#### `description`

사람이 읽을 수 있는 스코프 설명이며 선택 사항입니다. 플랫폼은 이 설명으로 사용자에게 컨텍스트(요구 사항과 가치)를 보여 주고 설명할 수 있습니다.

```json
{
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
- **JWT 그랜트 수명.** JWT 인가 그랜트는 반드시 단기여야 합니다(**MUST**). `exp` 클레임은 `iat` 이후 60초를 넘지 않아야 합니다(**SHOULD**). 수명이 짧으면 그랜트 탈취와 재사용이 가능한 시간 창이 줄어듭니다.
- **JWT 그랜트 1회용.** 비즈니스는 그랜트의 유효 기간 동안 `jti` 클레임을 추적해 JWT 인가 그랜트를 반드시 1회용으로 강제해야 합니다(**MUST**). 짧은 exp는 재사용 창을 좁힐 뿐이고, 그 창을 완전히 닫을 수 있는 것은 jti 추적뿐입니다.
- **그랜트 중계.** 비즈니스는 플랫폼에서 받은 JWT 인가 그랜트를 저장하거나 전달해서는 안 됩니다(**MUST NOT**). 그랜트는 단일 대상(`aud`)과 1회 사용으로 범위가 한정된 bearer 자격 증명입니다.
- **IdP 간 계정 연결.** 페더레이션은 신뢰 경계를 무너뜨립니다. IdP가 단언한 클레임(이메일, 전화번호, 이름)을 근거로 나열된 제공자 사이에서 계정을 자동으로 연결하는 비즈니스는, 각 제공자의 검증 절차가 허술하면 그 허점이 그대로 계정 탈취 위험이 됩니다. 사용자가 통제하지 않는 이메일에 `email_verified=true`를 발급하는 제공자는 그 이메일을 공유하는 비즈니스의 모든 계정을 탈취할 수 있습니다. IdP별 안정적 식별자는 `(iss, sub)`이며 다른 클레임은 참고용입니다. 비즈니스는 IdP 간 계정을 병합하기 전에 사용자가 직접 개입하는 연결을 요구해야 합니다(**SHOULD**). 즉, 사용자가 현재 세션 인증이나 그에 상응하는 방법으로 기존 계정을 통제하고 있음을 증명하게 해야 합니다.

## 향후 확장성

스키마는 OAuth가 아닌 제공자 메커니즘을 호환성을 깨지 않는 확장으로 받아들일 수 있게 설계되었습니다. `provider.type` 구분자는 필수인 열린 문자열입니다. 이 버전에 정의된 타입은 `oauth2`뿐이며, 지갑 증명(wallet attestation), 검증 가능한 자격 증명, 기타 신원 증명 프로토콜 같은 향후 타입을 위한 여지를 남겨 둡니다. 향후 버전에서 추가 `type` 값과 그에 대응하는 디스커버리 및 증명 제시 방식을 정의할 수 있습니다.

**플랫폼을 위한 상위 호환 규칙:** `config`에 이 명세 버전에 정의되지 않은 필드가 있으면 플랫폼은 해당 필드를 반드시 무시해야 합니다(**MUST**). 플랫폼은 지원하지 않는 `type`의 제공자 항목을 걸러낸 것으로 반드시 취급해야 하며(**MUST**), 남은 항목에 [신원 제공자](#identity-providers)의 규칙을 적용합니다.

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
  "grant_types_supported": [
    "authorization_code",
    "refresh_token",
    "urn:ietf:params:oauth:grant-type:jwt-bearer"
  ],
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

참고: `authorization_response_iss_parameter_supported: true`는 [RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207) 지원을 알립니다. `code_challenge_methods_supported: ["S256"]`은 PKCE 지원을 나타냅니다. UCP를 준수하는 메타데이터에는 둘 다 반드시 있어야 합니다(**MUST**). `urn:ietf:params:oauth:grant-type:jwt-bearer` 그랜트 타입은 비즈니스가 [가속 IdP 흐름](#accelerated-idp-flow)에서 신뢰하는 IdP의 JWT 인가 그랜트를 받아들인다는 뜻입니다.

`token_endpoint_auth_methods_supported`에는 비즈니스가 받아들이는 모든 방식이 나열됩니다. 이 예시는 기밀 클라이언트용 비대칭 방식(`private_key_jwt`, `tls_client_auth`), 레거시 호환용 `client_secret_basic`, 공개 클라이언트 ([RFC 8252](https://datatracker.ietf.org/doc/html/rfc8252)에 따른 네이티브, 데스크톱, 온디바이스 에이전트)용 `none`을 선언합니다. `none`을 선언한다면 `S256` 방식의 PKCE가 필수입니다. 공개 클라이언트를 지원하지 않는 비즈니스는 `none`을 생략할 수 있습니다(**MAY**).

### 비즈니스 프로필(`/.well-known/ucp`)

`config.scopes`의 구성은 비즈니스의 정책을 반영합니다.

#### B2C 소매업체

공개 카탈로그와 게스트 체크아웃(스코프 불필요)을 허용하고, 사용자에 귀속된 주문 작업은 제한하는 경우입니다.

```json
{
  "ucp": {
    "version": "2026-08-25",
    "services": {},
    "capabilities": {
      "dev.ucp.common.identity_linking": [{
        "version": "2026-08-25",
        "spec": "https://ucp.dev/2026-08-25/specification/common/identity-linking/",
        "schema": "https://ucp.dev/2026-08-25/schemas/common/identity_linking.json",
        "config": {
          "scopes": {
            "dev.ucp.shopping.order:read":    {},
            "dev.ucp.shopping.order:manage":  {}
          }
        }
      }]
    },
    "payment_handlers": {}
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
    "version": "2026-08-25",
    "services": {},
    "capabilities": {
      "dev.ucp.common.identity_linking": [{
        "version": "2026-08-25",
        "spec": "https://ucp.dev/2026-08-25/specification/common/identity-linking/",
        "schema": "https://ucp.dev/2026-08-25/schemas/common/identity_linking.json",
        "config": {
          "scopes": {
            "dev.ucp.shopping.checkout:manage":  {},
            "dev.ucp.shopping.order:read":       {},
            "dev.ucp.shopping.order:manage":     {}
          }
        }
      }]
    },
    "payment_handlers": {}
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
