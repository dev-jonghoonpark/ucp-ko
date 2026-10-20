# 메시지 서명

이 명세는 진위성과 무결성을 보장할 수 있도록 UCP 메시지에 암호학적으로 서명하는 방법을 정의합니다.

## 개요

이 명세는 [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) HTTP Message Signatures로 UCP 메시지에 서명하고 이를 검증하는 방법을 정의합니다. UCP의 신원 모델, 지원하는 인증 메커니즘, 키 디스커버리 프로토콜은 [신원과 인증](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#identity-authentication)을 참고하세요.

HTTP Message Signatures는 다음 공격을 막아 줍니다.

- **사칭** — 공격자가 정당한 참여자인 척하며 메시지를 보내는 행위
- **변조** — 전송 중 메시지 내용을 수정하는 행위
- **재전송 공격** — 탈취한 메시지를 다른 엔드포인트나 다른 시점에 다시 보내는 행위
- **메서드/엔드포인트 혼동** — 서명된 페이로드를 다른 HTTP 메서드나 다른 경로로 재전송하는 행위

### 아키텍처

UCP는 모든 HTTP 기반 전송 방식에서 HTTP Message Signatures([RFC 9421](https://www.rfc-editor.org/rfc/rfc9421))를 사용합니다.

```text
+-----------------------------------------------------------------+
|                     SHARED FOUNDATION                           |
+-----------------------------------------------------------------+
|  Signature Format: RFC 9421 (HTTP Message Signatures)           |
|  Body Digest: RFC 9530 (Content-Digest, raw bytes)              |
|  Algorithms: must verify ES256 (baseline); other algorithms     |
|              optional — open vocabulary, counterparty-driven    |
|              (see Signature Algorithms)                         |
|  Key Format: JWK (RFC 7517 + RFC 8037 for Ed25519)              |
|  Key Discovery: keys[] (RFC 7517 JWK Set) in /.well-known/ucp   |
|  Replay Protection: idempotency-key (business layer)            |
+-----------------------------------------------------------------+
                              |
                              v
+-----------------------------------------------------------------+
|                     HTTP TRANSPORTS                             |
+-----------------------------------------------------------------+
|  REST API: Standard HTTP requests                               |
|  MCP: Streamable HTTP transport (JSON-RPC over HTTP)            |
+-----------------------------------------------------------------+
|  Headers:                                                       |
|    Signature-Input    (describes signed components)             |
|    Signature          (contains signature value)                |
|    Content-Digest     (body hash, raw bytes)                    |
+-----------------------------------------------------------------+
```

**참고:** UCP는 MCP 전송 방식으로 SSE 기반 전송 대신 streamable HTTP를 지정합니다. 덕분에 같은 RFC 9421 서명 메커니즘을 모든 UCP 전송 방식에 일관되게 적용할 수 있습니다.

## 공통 기반

다음 암호 기본 요소는 모든 UCP HTTP 전송 방식에서 공통으로 사용합니다.

### 서명 알고리즘

UCP는 ECDSA(NIST P 곡선 기반)와 EdDSA(Ed25519) 두 가지 알고리즘 계열을 인정합니다. ECDSA P-256은 보편적인 기본선이고, EdDSA는 Web Bot Auth(WBA)와 상호 운용할 수 있게 해 주는 추가 선택지입니다.

| 계열          | JWK `kty` / `crv` | JWA `alg` | 해시    |
| ------------- | ----------------- | --------- | ------- |
| ECDSA P-256   | `EC` / `P-256`    | `ES256`   | SHA-256 |
| ECDSA P-384   | `EC` / `P-384`    | `ES384`   | SHA-384 |
| EdDSA Ed25519 | `OKP` / `Ed25519` | `EdDSA`   | (내장)  |

**구현 요건:**

- 모든 구현은 반드시 `ES256`(ECDSA P-256) 서명 검증을 지원해야 합니다(**MUST**). 이것이 UCP의 보편적 기본선입니다.
- `ES384`(ECDSA P-384) 지원은 선택 사항입니다(**OPTIONAL**).
- `EdDSA`(Ed25519) 지원은 선택 사항입니다(**OPTIONAL**). WBA를 인식하는 검증자는 자신의 서명자들이 실제로 쓰는 알고리즘을 지원해야 합니다(**SHOULD**)(현재 WBA 서명자 사이에서는 Ed25519가 가장 흔합니다). UCP는 보편적인 `ES256` 기본선 외에는 알고리즘 요건을 두지 않습니다.
- `kty`/`crv`/`alg` 어휘는 **개방형**입니다. 지원하지 않는 타입, 곡선, 알고리즘의 키를 만나더라도 검증자는 공개된 키 집합을 거부해서는 안 됩니다(**MUST NOT**). 지원하지 않는 키는 그 검증자가 쓸 수 없는 키로 남을 뿐입니다. 그런 키를 참조하는 서명은 `algorithm_unsupported`로 실패합니다(다중 서명 요청에서는 [신원 확인 알고리즘](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#identity-resolution-algorithm)에 따라 건너뜁니다).

**사용 지침:**

- **알고리즘은 상대방에 맞춰 고릅니다.** 서명자는 같은 서명이 충족해야 하는 모든 검증자, 즉 수신 측 UCP 검증자와 해당 서명을 적용하기로 한 WBA 검증자나 AP2 계층이 모두 수락하는 알고리즘을 사용해야 합니다(**SHOULD**). 알고리즘 하나로 모두를 충족할 수 있으면 키도 하나면 됩니다. 어떤 단일 알고리즘으로도 충족할 수 없을 때만 알고리즘이 서로 다른 키를 여러 개 공개합니다(서명마다 `kid`로 선택).
- 상대방에게 특별한 제약이 없으면 \*\*기본값은 `ES256`\*\*입니다. UCP의 보편적 기본선이면서 Web Bot Auth에서도 유효한 알고리즘입니다. (WBA의 알고리즘 규칙과 현재 배포 현황은 [WBA 상호 운용](#wba-interop)에서 다룹니다.)
- **AP2 위임(Mandate) 서명은 AP2 자체의 알고리즘 규칙을 따릅니다.** [AP2 위임](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/payment/extensions/ap2-mandates/index.md)을 참고하세요.
- 알고리즘은 JWK의 `kty`/`crv` 필드로 도출하며, `alg`는 `Signature-Input` 파라미터에 포함하지 **않습니다(NOT)**.

**서명 키의 개수.** 당사자가 공개할 키의 개수는 UCP 규칙이 아니라 알고리즘 호환성에 따라 정해집니다. 서명 대상이 모두 수락하는 단일 알고리즘이 있다면 키 하나로 충분합니다. 대상들의 알고리즘 제약이 서로 양립할 수 없을 때만 (`kid`로 선택하는) 별도 키가 필요합니다. 예를 들어 Ed25519만 수락하는 WBA 검증자와 Ed25519를 배제하는 AP2 위임 알고리즘 요건이 함께 있는 경우입니다([AP2 위임](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/payment/extensions/ap2-mandates/index.md) 참고). 알고리즘 하나로 모든 대상을 충족하면 키 하나로 모두 처리합니다. 키 두 개를 쓰는 예시는 [비즈니스 프로필](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#business-profile)을 참고하세요.

전송 시 서명 인코딩에 관한 자세한 내용은 [REST 요청 서명 — 서명 인코딩](#rest-request-signing)을 참고하세요.

### 키 형식 (JWK)

공개 키는 반드시 [RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517)에 정의된 **JSON Web Key(JWK)** 형식으로 표현해야 합니다(**MUST**). UCP는 잘 알려진 JWK 형태 두 가지를 정의합니다. ECDSA 키용 **EC**(RFC 7518 §6.2)와 EdDSA 키용 **OKP**([RFC 8037](https://datatracker.ietf.org/doc/html/rfc8037))입니다. JWK 어휘는 개방형입니다([서명 알고리즘](#signature-algorithms) 참고). 프로필에 다른 타입의 키를 공개할 수 있고(MAY), 검증자는 사용할 수 없는 키를 건너뜁니다.

**EC 키 구조 (ECDSA P-256, P-384):**

| 필드  | 타입   | 필수   | 설명                            |
| ----- | ------ | ------ | ------------------------------- |
| `kid` | string | 예     | 키 ID(서명에서 참조)            |
| `kty` | string | 예     | 키 타입(`EC`)                   |
| `crv` | string | 예     | 곡선 이름(`P-256` 또는 `P-384`) |
| `x`   | string | 예     | X 좌표(base64url 인코딩)        |
| `y`   | string | 예     | Y 좌표(base64url 인코딩)        |
| `use` | string | 아니요 | 키 용도(서명용은 `sig`)         |
| `alg` | string | 아니요 | 알고리즘(`ES256`, `ES384`)      |

**OKP 키 구조 (EdDSA Ed25519):**

| 필드  | 타입   | 필수   | 설명                                            |
| ----- | ------ | ------ | ----------------------------------------------- |
| `kid` | string | 예     | 키 ID(서명에서 참조)                            |
| `kty` | string | 예     | 키 타입(`OKP`)                                  |
| `crv` | string | 예     | 곡선 이름(`Ed25519`)                            |
| `x`   | string | 예     | 공개 키 값(RFC 8037 §2에 따른 base64url 인코딩) |
| `use` | string | 아니요 | 키 용도(서명용은 `sig`)                         |
| `alg` | string | 아니요 | 알고리즘(`EdDSA`)                               |

**EC 예시 (ES256):**

```json
{
  "kid": "key-2024-01-15",
  "kty": "EC",
  "crv": "P-256",
  "x": "WKn-ZIGevcwGIyyrzFoZNBdaq9_TsqzGl96oc0CWuis",
  "y": "y77t-RvAHRKTsSGdIYUfweuOvwrvDD-Q3Hv5J0fSKbE",
  "use": "sig",
  "alg": "ES256"
}
```

**OKP 예시 (Ed25519 / EdDSA):**

```json
{
  "kid": "poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U",
  "kty": "OKP",
  "crv": "Ed25519",
  "x": "JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs",
  "use": "sig",
  "alg": "EdDSA"
}
```

**이중 대상(dual-audience)** 서명(`tag="web-bot-auth"`가 있는 서명)에 쓰는 키는, 그 키를 나열하는 모든 배열에서 반드시 해당 키의 JWK SHA-256 Thumbprint([RFC 7638](https://www.rfc-editor.org/rfc/rfc7638))를 `kid`로 공개해야 합니다(**MUST**). WBA 형태 서명의 `keyid`가 바로 그 thumbprint이고, UCP 검증자는 `keyid`를 공개된 `kid`와 대조해 키를 찾습니다. 둘 다 thumbprint로 설정하면 `UCP-Agent` 조회와 `Signature-Agent` 조회가 같은 키를 찾게 됩니다. 그 밖의 키에서 `kid`는 불투명 식별자(RFC 7517)이며, 안정적인 값이라면 무엇이든 쓸 수 있습니다(MAY).

### 키 디스커버리

공개 키는 서명자의 UCP 프로필에 공개됩니다. 공개 계약은 [프로필 구조](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#profile-structure)를, 검증자의 조회 규칙(확인 메커니즘별로 어떤 키 목록을 읽는지)은 [키 디스커버리](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#key-discovery)를 참고하세요.

### 키 교체

서비스 중단 없이 키를 교체하는 절차는 다음과 같습니다.

1. **새 키 추가** — 프로필의 `keys[]`에 기존 키와 함께 새 키를 공개합니다
1. **서명 시작** — 새 키로 서명을 시작합니다
1. **유예 기간** — 기존 키로 만든 서명도 계속 수락합니다(최소 7일)
1. **기존 키 제거** — `keys[]`에서 기존 키를 제거합니다. `keys[]`에 남아 있는 키는 계속 검증에 사용됩니다.

**권장 사항:**

- 운영자는 90일마다 키를 교체해야 합니다(SHOULD).
- 프로필은 전환 기간 동안 여러 활성 키를 지원해야 합니다(SHOULD).

**키 유출 대응:**

1. 유출된 키를 즉시 `keys[]`에서 제거합니다. 배열에서 사라지기 전까지는 계속 검증에 사용됩니다
1. 다른 `kid`로 새 키를 추가합니다
1. 유출된 키로 만든 서명은 모두 거부합니다

### WBA 상호 운용

UCP 통합자는 자신의 기본 서명을 Web Bot Auth 호환 형태로 만들 수 있습니다(MAY). 키 하나, 서명 작업 한 번, 전송되는 서명 하나로 이루어진 단일 **이중 대상** 서명을 UCP 검증자(`UCP-Agent`로 확인)와 WBA 검증자(`Signature-Agent`로 확인)가 모두 수락하게 하는 방식입니다. WBA 준수 검증자와 상호 운용하려는 통합자에게 권장하며(RECOMMENDED), 아래 요건은 `tag="web-bot-auth"`가 있는 모든 서명에 적용됩니다. WBA 상호 운용은 요청에만 해당합니다. 응답은 (`@status`를 포함하는) 표준 UCP 서명으로 서명하며 `tag="web-bot-auth"`를 담지 않습니다.

**전송 형태.** 이중 대상 서명은 일반 UCP 서명에 몇 가지를 더한 형태입니다. `UCP-Agent`와 **함께** `Signature-Agent` 헤더를 담고(추가 방식 — `Signature-Agent`가 `UCP-Agent`를 대체하지 않습니다), `signature-agent` 컴포넌트에 서명하며, 서명 키의 RFC 7638 thumbprint를 `keyid`로 쓰고, `created`, `expires`, `tag="web-bot-auth"`를 추가합니다.

```json
POST /checkout-sessions HTTP/1.1
Host: merchant.example.com
Content-Type: application/json
UCP-Agent: profile="https://platform.example/.well-known/ucp"
Signature-Agent: sig1="https://platform.example/.well-known/ucp";type=jwks_uri
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Digest: sha-256=:X48E9q...:
Signature-Input: sig1=("@method" "@authority" "@path" "signature-agent";key="sig1" "ucp-agent" "idempotency-key" "content-digest" "content-type");keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U";created=1738617600;expires=1738621200;tag="web-bot-auth"
Signature: sig1=:base64_ed25519_signature_value:

{
  "line_items": [
    {
      "item": {"id": "item_123"},
      "quantity": 2
    }
  ]
}
```

전송되는 서명은 하나, 대상은 둘입니다. UCP 형태 검증자는 `UCP-Agent`로 확인하고, 자신이 기대하는 컴포넌트(`ucp-agent`, `idempotency-key`)를 찾은 뒤 나머지는 무시합니다. WBA 형태 검증자는 `Signature-Agent`로 확인하고 자신의 컴포넌트(`@authority`, `signature-agent`, `tag`, `created`/`expires`)를 찾습니다. 두 검증자 모두 같은 키로 같은 바이트를 검증합니다. 이 예시에서는 두 헤더가 같은 `/.well-known/ucp` URL을 가리키지만(`type=jwks_uri`이므로 WBA 검증자는 프로필의 `keys[]`를 JWK Set으로 읽습니다 — [배포 패턴](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#deployment-patterns-for-wba-interop) 참고), `Signature-Agent`가 다른 곳을 가리켜도 됩니다(MAY).

세 개의 `sig1` 레이블, 즉 `Signature-Agent` 딕셔너리 멤버 키, 서명된 `signature-agent` 컴포넌트의 `;key="sig1"` 파라미터, `Signature-Input` 서명 레이블은 아래 옵트인 목록 3번 항목에 따라 서로 묶여 있습니다.

옵트인하려면 서명자는 기본 UCP 서명을 다음과 같이 바꿉니다. **MUST** 표시 항목은 [draft-meunier-webbotauth-httpsig-protocol-00](https://datatracker.ietf.org/doc/draft-meunier-webbotauth-httpsig-protocol/00/) §4.2가 요구하는 사항입니다. 자세한 내용은 해당 초안을 참고하세요.

1. **WBA 검증자가 수락하는 알고리즘을 사용합니다.** WBA는 RFC 9421 HTTP Message Signatures Algorithm 레지스트리의 모든 알고리즘(예: `ed25519`, `ecdsa-p256-sha256` = `ES256`, `ecdsa-p384-sha384`, `rsa-pss-sha512`)을 허용합니다. 대상 검증자가 수락하는지 확인하세요. (비규범: 현재 가장 널리 배포된 알고리즘은 Ed25519입니다.)
1. `UCP-Agent`와 함께 **반드시 `Signature-Agent` 헤더를 보내야 합니다(MUST)**. 이 헤더는 RFC 8941 Dictionary Structured Field로, 멤버의 sf-string 값은 HTTPS URL이고 `type` 파라미터로 디스커버리 메커니즘을 고릅니다. 멤버 키는 `Signature-Input` 서명 레이블과 일치합니다. `jwks_uri`/`cimd`/`directory` 변형과, 각 변형에서 UCP 프로필을 재사용하는 방법은 [배포 패턴](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#deployment-patterns-for-wba-interop)을 참고하세요. (`data:` URI 인라인 형태는 범위 밖입니다.)
1. `Signature-Agent` 딕셔너리 멤버 키(`Signature-Input` 서명 레이블과 같음)와 일치하는 **`;key="<label>"`로 반드시 `signature-agent` 컴포넌트에 서명해야 합니다(MUST)**. WBA §4.2.1을 따릅니다.
1. [RFC 7638](https://www.rfc-editor.org/rfc/rfc7638)에 따라 **반드시 `keyid`를 서명 키의 JWK SHA-256 Thumbprint로 설정해야 하며(MUST)**, `UCP-Agent` 조회와 `Signature-Agent` 조회가 같은 키를 찾도록 그 키를 같은 thumbprint 값의 `kid`로 공개합니다 ([키 형식](#key-format-jwk) 참고). Ed25519(OKP) 키라면 thumbprint 멤버는 [RFC 8037](https://www.rfc-editor.org/rfc/rfc8037) §2에 따라 `crv`, `kty`, `x`이며, 부록 A.3에 계산 예시가 있습니다.
1. **반드시 `created`와 `expires` 파라미터를 포함해야 합니다(MUST).** `expires` 간격은 최대 24시간이어야 합니다(SHOULD).
1. 재전송을 막으려면 **`nonce`를 포함해야 합니다(SHOULD)**. `nonce`는 base64url로 인코딩한 무작위 값(권장(RECOMMENDED) 64바이트)이며, `created`/`expires` 기간 안에서 고유해야 합니다(WBA §4.2.3). UCP의 `Idempotency-Key`는 비즈니스 계층에서 페이로드 중복을 걸러 내는 장치일 뿐 전송에 결합된 nonce가 아니므로 그 역할을 대신하지 못합니다. 검증하는 오리진은 `nonce`를 요구할 수 있으며(**MAY**), nonce가 없거나 재사용된 서명에는 다시 챌린지(HTTP 429)할 수 있습니다(WBA §4.3–4.4).
1. **반드시 `tag="web-bot-auth"`를 포함해야 합니다(MUST).** WBA 검증자는 이 태그를 보고 서명을 고릅니다.

**컴포넌트 요건은 그대로 유지되고 강제됩니다.** WBA 형태 서명에도 반드시 기본 UCP 서명과 같은 컴포넌트([서명 컴포넌트](#rest-request-signing) 표의 필수 집합)가 포함되어야 합니다(**MUST**). WBA는 [draft-meunier-webbotauth-httpsig-protocol-00](https://datatracker.ietf.org/doc/draft-meunier-webbotauth-httpsig-protocol/00/) §4.2.4에 따라 이를 "추가 컴포넌트"로 수락합니다. 검증자는 [신원 확인 알고리즘](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#identity-resolution-algorithm)에 따라 `tag`와 상관없이 이 요건을 강제하므로, Web Bot Auth에 옵트인해도 UCP가 인증하는 범위는 넓어지지 않습니다.

**상호 운용은 단방향입니다.** UCP 서명자는 Web Bot Auth 검증자의 요건을 충족합니다. WBA 검증자는 UCP의 더 풍부한 포함 컴포넌트 집합을 허용된 "추가 컴포넌트"로 수락하기 때문입니다(protocol-00 §4.2.4). 반대 방향은 성립하지 않습니다. (`@authority`만 포함하는) 최소한의 WBA 서명은 UCP의 포함 범위 검사를 통과하지 못해 거부됩니다. UCP의 목표는 WBA 검증자*가* 검증할 수 있게 하는 것이지, 임의의 WBA 서명자를 수락하는 것이 아닙니다. UCP 검증자가 수락하는 범위는 WBA 검증자가 수락하는 범위의 진부분집합입니다.

UCP 검증자는 같은 서명에서 새로운 요소 세 가지를 보게 됩니다.

- `tag` 파라미터는 RFC 9421 §2.3의 서명 파라미터입니다. UCP 전용 검증자는 이 파라미터를 알지 못하며, RFC 9421의 관대한 파라미터 처리 규칙에 따라 무시합니다.
- `signature-agent` 컴포넌트는 포함된 HTTP 필드로서 평소처럼 처리됩니다. `;key="<label>"` 파라미터는 RFC 9421 §2.1.2에 따라 Dictionary 멤버를 선택합니다. WBA 형태 서명을 검증하려면 §2.1.2를 지원해야 합니다. UCP 기본 서명은 Dictionary 멤버 컴포넌트 선택을 쓰지 않으므로, UCP 기본 서명만 염두에 두고 만든 검증자라면 이 지원을 추가해야 할 수도 있습니다.
- WBA 형태 서명에서는 `created`와 `expires`가 필수입니다(5번 항목, WBA §4.2). 이를 강제할지는 **애플리케이션이 정합니다**(RFC 9421 §3.2.1). RFC 9421을 준수한다고 자동으로 강제되는 것도 아니고, UCP가 별도로 요구하지도 않습니다. UCP 자체의 재전송 방지는 비즈니스 계층의 `Idempotency-Key`가 맡습니다. 신선도를 강제하는 WBA 인식 검증자는 유효 기간을 벗어난 서명을 거부합니다.

**신원 확인.** WBA에 옵트인해도 기본 UCP 검증은 바뀌지 않습니다. [신원 확인 알고리즘](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#identity-resolution-algorithm)을 참고하세요.

**태그.** UCP는 자체 `tag`(RFC 9421 §2.3)를 정의하지 않습니다. UCP 검증자는 `UCP-Agent` 헤더, 서명된 컴포넌트 집합, URL 라우팅으로 자신의 서명을 식별합니다.

**다중 서명.** UCP 요청에는 [RFC 9421 §4.3](https://www.rfc-editor.org/rfc/rfc9421#section-4.3) 레이블 메커니즘으로 여러 서명을 담을 수 있습니다(**MAY**). 이 패턴은 실제로 분리가 필요할 때(대상마다 다른 키, 여러 당사자의 공동 서명, 대상별 컴포넌트 집합이 서로 충돌하는 경우)만 사용합니다. 키 하나로 UCP와 WBA를 함께 지원하려면 위에서 설명한 단일 서명 형태를 쓰는 편이 낫습니다.

## REST 바인딩

HTTP REST 전송 방식에서 UCP는 [RFC 9421 (HTTP Message Signatures)](https://www.rfc-editor.org/rfc/rfc9421)를 사용합니다.

### 헤더

| 헤더              | 방향      | 필수        | 설명                                       |
| ----------------- | --------- | ----------- | ------------------------------------------ |
| `Signature-Input` | 요청/응답 | 예          | 서명된 컴포넌트 기술                       |
| `Signature`       | 요청/응답 | 예          | 서명 값                                    |
| `Content-Digest`  | 요청/응답 | 조건부 `*`  | 요청/응답 본문의 SHA-256 해시              |
| `Signature-Agent` | 요청      | 조건부 `**` | WBA 키 출처([WBA 상호 운용](#wba-interop)) |

- `*` 요청/응답에 본문이 있으면 필수
- `**` Web Bot Auth 호환 서명 형태에 옵트인하면 필수. 기본 UCP 서명에는 없음(검증자는 `UCP-Agent`에서 도출한 신원을 사용)

`Content-Digest`는 [RFC 9530](https://www.rfc-editor.org/rfc/rfc9530)을 따르며 원본 본문 바이트를 해시합니다. 덕분에 JSON 정규화 없이도 메시지 본문을 서명에 결합할 수 있습니다. 구현은 반드시 `sha-256`을 사용해야 합니다(**MUST**). 정규화가 필요한 영속적 아티팩트는 [AP2 위임 - 정규화](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/payment/extensions/ap2-mandates/#canonicalization)를 참고하세요.

**중개자 경고:** 프록시, API 게이트웨이 같은 중개자는 JSON 본문을 다시 직렬화해서는 안 됩니다(**MUST NOT**). 다시 직렬화하면 서명이 무효가 됩니다. `Content-Digest`는 원본 바이트로 계산하므로 조금이라도 수정하면 검증이 깨집니다.

### REST 요청 서명

**서명 컴포넌트:**

| 컴포넌트          | 필수          | 설명                                                       |
| ----------------- | ------------- | ---------------------------------------------------------- |
| `@method`         | 예            | HTTP 메서드(GET, POST 등)                                  |
| `@authority`      | 예            | 대상 호스트(다른 호스트로의 중계 방지)                     |
| `@path`           | 예            | 요청 경로                                                  |
| `@query`          | 조건부 `*`    | 쿼리 문자열(있는 경우)                                     |
| `ucp-agent`       | 조건부 `**`   | 프로필 URL(신원 결합)                                      |
| `signature-agent` | 조건부 `***`  | WBA 키 출처([WBA 상호 운용](#wba-interop)에 옵트인한 경우) |
| `idempotency-key` | 조건부 `****` | 멱등성 헤더(상태 변경 요청)                                |
| `content-digest`  | 조건부 `†`    | 본문 다이제스트(본문이 있는 경우)                          |
| `content-type`    | 조건부 `†`    | Content-Type(본문이 있는 경우)                             |

- `*` 요청에 쿼리 파라미터가 있으면 필수
- `**` `UCP-Agent` 헤더가 있으면 필수
- `***` `Signature-Agent` 헤더가 있으면(즉 WBA 형태 서명이면) 필수
- `****` POST, PUT, DELETE, PATCH에서 필수
- `†` 요청에 본문이 있으면 필수

**서명 생성:**

```text
sign_rest_request(method, path, query, body_bytes, idempotency_key, private_key, kid):
    // 1. Compute body digest (if body present)
    if body_bytes:
        digest = sha256(body_bytes)  // Hash raw bytes, no canonicalization
        digest_header = "sha-256=:" + base64(digest) + ":"

    // 2. Build component list
    components = ["@method", "@authority", "@path"]
    if query: components.append("@query")
    if ucp_agent: components.append("ucp-agent")
    if idempotency_key: components.append("idempotency-key")
    if body: components.extend(["content-digest", "content-type"])

    // 3. Build signature base (RFC 9421)
    signature_base = build_signature_base(
        components=components,
        method=method,
        path=path,
        query=query,
        headers={
            "idempotency-key": idempotency_key,
            "content-digest": digest_header,
            "content-type": "application/json"
        },
        keyid=kid
    )

    // 4. Sign
    signature = sign(signature_base, private_key)  // ecdsa for EC, eddsa for OKP

    // 5. Return headers
    return {
        "Idempotency-Key": idempotency_key,
        "Content-Digest": digest_header,
        "Signature-Input": format_signature_input(components, kid),
        "Signature": "sig1=:" + base64(signature) + ":"
    }
```

**서명 인코딩:**

- **ECDSA** 서명은 반드시 ASN.1/DER이 **아니라** RFC 9421 §3.3.1에 따른 고정 길이 원시 `r||s` 인코딩을 사용해야 합니다(**MUST**). 서명 값은 고정 길이 부호 없는 빅엔디언 정수 `r`과 `s`를 이어 붙인 값으로, P-256은 64바이트(32 + 32), P-384는 96바이트(48 + 48)입니다. 많은 암호 라이브러리(OpenSSL, Java, .NET)가 기본으로 DER 인코딩을 사용하므로 명시적으로 변환해야 합니다.
- **EdDSA (Ed25519)** 서명은 반드시 RFC 8032 §5.1.6에 정의된 인코딩, 즉 인코딩된 `R` 점과 정수 `S`를 이어 붙인 64바이트를 사용해야 합니다(**MUST**). Ed25519 서명 라이브러리의 표준 출력이 이 형태이므로 DER 변환이 필요 없습니다.

**전체 요청 예시:**

```json
POST /checkout-sessions HTTP/1.1
Host: merchant.example.com
Content-Type: application/json
UCP-Agent: profile="https://platform.example/.well-known/ucp"
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Digest: sha-256=:X48E9q...:
Signature-Input: sig1=("@method" "@authority" "@path" "ucp-agent" "idempotency-key" "content-digest" "content-type");keyid="platform-2026"
Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:

{
  "line_items": [
    {
      "item": {"id": "item_123"},
      "quantity": 2
    }
  ]
}
```

**GET 요청 예시 (본문 없음, 멱등성 키 없음):**

```http
GET /checkout-sessions/chk_123 HTTP/1.1
Host: merchant.example.com
Signature-Input: sig1=("@method" "@authority" "@path");keyid="platform-2026"
Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:
```

### REST 응답 서명

응답 서명에는 `@method` 대신 `@status`를 사용합니다.

**서명 컴포넌트:**

| 컴포넌트         | 필수       | 설명                              |
| ---------------- | ---------- | --------------------------------- |
| `@status`        | 예         | HTTP 상태 코드(200, 201 등)       |
| `content-digest` | 조건부 `*` | 본문 다이제스트(본문이 있는 경우) |
| `content-type`   | 조건부 `*` | Content-Type(본문이 있는 경우)    |

- `*` 응답에 본문이 있으면 필수

**전체 응답 예시:**

아래 응답 본문은 이해를 돕기 위해 축약했으며, 서명에 쓰이는 핵심 필드만 보여 줍니다. 실제 체크아웃 응답에는 추가 필수 필드(`ucp`, `currency`, `line_items`, `totals`, `links`)도 포함됩니다. 전체 형태는 [체크아웃 생성 응답](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/checkout/rest/#create-checkout)을 참고하세요.

```http
HTTP/1.1 201 Created
Content-Type: application/json
Content-Digest: sha-256=:Y5fK8nLmPqRsT3vWxYzAbCdEfGhIjKlMnO...:
Signature-Input: sig1=("@status" "content-digest" "content-type");created=1738617601;keyid="merchant-2026"
Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:

{
  "id": "chk_123",
  "status": "ready_for_complete",
  "...": "abbreviated; see linked spec for full shape"
}
```

**응답 서명 생성:**

응답 서명은 `@method` 대신 `@status`를 쓴다는 점만 빼면 요청 서명과 같습니다.

```text
sign_rest_response(status, body_bytes, private_key, kid):
    // 1. Compute body digest (if body present)
    if body_bytes:
        digest = sha256(body_bytes)  // Hash raw bytes, no canonicalization
        digest_header = "sha-256=:" + base64(digest) + ":"

    // 2. Build signature base (RFC 9421)
    signature_base = build_signature_base(
        components=["@status", "content-digest", "content-type"],
        status=status,
        headers={"content-digest": digest_header, "content-type": "application/json"},
        created=current_timestamp(),
        keyid=kid
    )

    // 3. Sign
    signature = sign(signature_base, private_key)  // ecdsa for EC, eddsa for OKP

    // 4. Return headers
    return {
        "Content-Digest": digest_header,
        "Signature-Input": 'sig1=("@status" "content-digest" "content-type");created=...;keyid="..."',
        "Signature": "sig1=:" + base64(signature) + ":"
    }
```

### REST 요청 검증

**서명자의 키 확인:**

키 확인 규칙(서명의 `tag`가 아니라 검증자의 능력과 실제로 있는 헤더로 결정)은 [신원 확인 알고리즘](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#identity-resolution-algorithm)을 참고하세요. 이 절에서는 헤더 파싱만 다룹니다. 기본 UCP 방식에서는 `UCP-Agent`를, WBA 형태 방식에서는 `Signature-Agent`를 파싱합니다.

**`UCP-Agent` 파싱 규칙** (기본 UCP 방식):

1. RFC 8941 Dictionary로 파싱합니다
1. `profile` 키를 추출합니다(필수(REQUIRED))
1. 값은 반드시 HTTPS URL을 담은 따옴표 문자열이어야 합니다(MUST)
1. 비즈니스 프로필이라면 URL은 반드시 `/.well-known/ucp`를 가리켜야 합니다(MUST). 플랫폼 프로필 URL에는 경로 제약이 없습니다
1. HTTPS가 아닌 URL은 거부합니다

**`Signature-Agent` 파싱 규칙** (WBA 형태 방식):

1. RFC 8941 Dictionary로 파싱합니다.
1. 반드시 검증 중인 서명 레이블과 키가 같은 딕셔너리 멤버를 찾아야 합니다(**MUST**) (`Signature-Input: sig1=...`이면 멤버 `sig1`을 찾습니다). 일치하는 멤버가 없으면 이 서명의 검증은 반드시 실패해야 합니다(**MUST**).
1. 멤버의 값은 반드시 HTTPS URL을 담은 sf-string이어야 합니다(**MUST**). 확인 방식은 `type` 파라미터로 고르며, `jwks_uri`(JWK Set URL, 예: UCP 프로필), `cimd`(Client ID Metadata Document), `directory`(잘 알려진 디렉터리를 호스팅하는 오리진) 중 하나입니다. [배포 패턴](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#deployment-patterns-for-wba-interop)을 참고하세요. `data:` URI 인라인 형태는 UCP-WBA 상호 운용의 범위 밖입니다.
1. URL이 HTTPS가 아니면 이 서명의 검증은 반드시 실패해야 합니다(**MUST**).

**예시 (기본 UCP):**

```text
// Header
UCP-Agent: profile="https://platform.example/.well-known/ucp"

// Parsed
profile_url = "https://platform.example/.well-known/ucp"
```

**예시 (Signature-Agent, WBA 형태):**

```text
// Headers
Signature-Agent: sig1="https://platform.example/.well-known/ucp";type=jwks_uri
Signature-Input: sig1=("@method" "@authority" ...);...

// Parsed (member key matches sig1)
type = jwks_uri
jwks_uri = "https://platform.example/.well-known/ucp"
```

**적용 범위:**

- **플랫폼 → 비즈니스 요청:** `UCP-Agent` 헤더의 프로필 URL
- **비즈니스 → 플랫폼 웹훅:** `UCP-Agent` 헤더의 프로필 URL

아래 두 루틴은 모두 **단일 후보** 서명을 검증합니다. `skip_signature(reason)`은 해당 후보로는 메시지를 인증할 수 없다는 뜻입니다. 다중 서명을 처리할 때 ([RFC 9421 §4.3](https://www.rfc-editor.org/rfc/rfc9421#section-4.3), [신원 확인 알고리즘](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#identity-resolution-algorithm) 참고) 검증자는 다음 후보를 시도하며, **모든** 후보를 건너뛴 경우에만 메시지를 거부합니다. `success()`는 메시지를 인증합니다.

```text
verify_rest_request(request):
    // 1. Parse Signature-Input
    sig_input = parse_signature_input(request.headers["Signature-Input"])
    keyid = sig_input.keyid
    components = sig_input.components

    // 2. Resolve signer's public key (capability-based; see
    // overview/index.md#identity-resolution-algorithm).
    key_set = resolve_signer_key_set(request.headers)
    // sig_capable skips keys not usable for verification: use:"enc", or
    // key_ops present without "verify" (RFC 7517 §4.2, §4.3)
    public_key = find_key_by_kid(sig_capable(key_set), keyid)
    if not public_key:
        return skip_signature("key_not_found")

    // pre-2a. WBA-shape signatures bind key identity to key bytes:
    // keyid MUST equal the matched JWK's RFC 7638 thumbprint
    // (see IRA step 4 / WBA architecture draft §4.2).
    if sig_input.tag == "web-bot-auth":
        if keyid != rfc7638_thumbprint(public_key):
            return skip_signature("signature_invalid")
    // 2a. Skip keys whose algorithm this verifier does not support.
    // The kty/crv/alg vocabularies are open (see Signature Algorithms);
    // an unsupported key never invalidates the whole key set.
    if not algorithm_supported(public_key):
        return skip_signature("algorithm_unsupported")

    // 2b. Enforce covered-component requirements (all regimes/transports).
    // Bind the target, the body when present, and every integrity-relevant
    // header the request carries. A signature covering only WBA's minimum
    // (@authority, signature-agent) does not authenticate a request whose
    // body/method/path is unbound.
    // (Which requests must CARRY Idempotency-Key is a binding rule,
    // separate from this coverage check.)
    required = ["@method", "@authority", "@path"]
    if request.query:                        required += ["@query"]
    if request.has_body:                     required += ["content-digest", "content-type"]
    if "Idempotency-Key" in request.headers: required += ["idempotency-key"]
    if "UCP-Agent" in request.headers:       required += ["ucp-agent"]
    if "Signature-Agent" in request.headers: required += ["signature-agent"]
    for component in required:
        if component not in components:
            // coverage failure, a target/body/header the signature does not
            // cover is treated as unsigned (see IRA step 5)
            return skip_signature("coverage_insufficient")

    // 3. Verify body digest (if body present)
    if "content-digest" in components:
        expected = "sha-256=:" + base64(sha256(request.body_bytes)) + ":"
        if request.headers["Content-Digest"] != expected:
            return skip_signature("digest_mismatch")

    // 4. Reconstruct signature base
    signature_base = build_signature_base(
        components, request.method, request.path, request.query,
        request.headers, keyid
    )

    // 5. Verify signature
    signature = parse_signature(request.headers["Signature"])
    if not verify(signature_base, signature, public_key):
        return skip_signature("signature_invalid")

    return success()

    // Note: Replay protection handled by the signed idempotency-key header
```

### REST 응답 검증

응답 검증은 `@method` 대신 `@status`를 쓴다는 점만 빼면 요청 검증과 같습니다.

```text
verify_rest_response(response, signer_profile_url):
    // 1. Parse Signature-Input
    sig_input = parse_signature_input(response.headers["Signature-Input"])
    keyid = sig_input.keyid
    components = sig_input.components

    // 2. Resolve signer's public key from the signer's profile.
    profile = fetch_profile(signer_profile_url)
    // signature-capable keys only (see request path; RFC 7517 §4.2, §4.3)
    public_key = find_key_by_kid(sig_capable(profile.keys), keyid)
    if not public_key:
        return skip_signature("key_not_found")

    // 2a. Skip keys whose algorithm this verifier does not support.
    // The kty/crv/alg vocabularies are open (see Signature Algorithms);
    // an unsupported key never invalidates the whole key set.
    if not algorithm_supported(public_key):
        return skip_signature("algorithm_unsupported")

    // 2b. Enforce covered-component requirements for responses (all regimes).
    // No method/idempotency to bind, but the body still MUST be covered.
    required = ["@status"]
    if response.has_body: required += ["content-digest", "content-type"]
    for component in required:
        if component not in components:
            return skip_signature("signature_invalid")

    // 3. Verify body digest (if body present)
    if "content-digest" in components:
        expected = "sha-256=:" + base64(sha256(response.body_bytes)) + ":"
        if response.headers["Content-Digest"] != expected:
            return skip_signature("digest_mismatch")

    // 4. Reconstruct signature base
    signature_base = build_signature_base(
        components, response.status,
        response.headers, keyid
    )

    // 5. Verify signature
    signature = parse_signature(response.headers["Signature"])
    if not verify(signature_base, signature, public_key):
        return skip_signature("signature_invalid")

    return success()
```

### 재전송 방지

UCP는 재전송 방지를 서명 계층이 아니라 **비즈니스 계층**에서 멱등성 키로 처리합니다. 관심사는 다음과 같이 나뉩니다.

| 계층       | 책임                       |
| ---------- | -------------------------- |
| **서명**   | 인증(누가), 무결성(무엇을) |
| **멱등성** | 안전한 재시도, 재전송 방지 |

**동작 방식:**

1. 상태를 변경하는 작업은 요청에 `idempotency-key`를 포함합니다
1. 멱등성 키는 서명된 페이로드의 일부입니다
1. 공격자는 서명을 무효화하지 않고서는 키를 수정할 수 없습니다
1. 중복 요청에는 캐시된 응답을 반환합니다(새로운 부수 효과 없음)

**멱등성 키 위치:**

`Idempotency-Key` 헤더는 서명 컴포넌트에 포함됩니다.

```http
POST /checkout-sessions HTTP/1.1
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Signature-Input: sig1=("@method" "@authority" "@path" "idempotency-key" ...);keyid="platform-2026"
Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:
```

**멱등성 키 요건:**

| 요건                          | 값                                                           |
| ----------------------------- | ------------------------------------------------------------ |
| **엔트로피**                  | 최소 128비트(예: UUID v4, 22자 이상의 영숫자)                |
| **고유성**                    | 클라이언트별, 작업 유형별                                    |
| **서버 저장 기간**            | 최소 24시간, 권장 48시간                                     |
| **중복 시 (페이로드 일치)**   | 캐시된 응답 반환(다시 실행하지 않음)                         |
| **중복 시 (페이로드 불일치)** | `409 Conflict`(REST) / `-32000`(MCP)으로 거부(실행하지 않음) |
| **저장소 장애 시**            | 실패 시 차단(fail closed)(503으로 요청 거부)                 |

**페이로드 대조:** 비즈니스는 반드시 원본 본문 바이트의 SHA-256 해시, 즉 RFC 9530이 `Content-Digest`로 규정하는 것과 같은 다이제스트를 비교해, 중복 키 요청의 페이로드가 원래 요청의 페이로드와 일치하는지 판별해야 합니다(**MUST**). 서명을 사용하면 이 값이 `Content-Digest` 헤더로 제공되며, 앞의 중개자 경고 덕분에 종단 간 바이트 동일성이 보장됩니다. 비즈니스는 이 값을 멱등성 키와 함께 저장합니다. 서명되지 않은 요청이라면 비즈니스가 받은 본문 바이트로 같은 다이제스트를 계산합니다. 따라서 플랫폼은 요청 페이로드를 수정할 때마다 반드시 새 멱등성 키를 생성해야 합니다(**MUST**). 결제 수단을 바꾼 재시도, 배송 주소 변경, 라인 아이템 교체 등 요청 본문이 조금이라도 바뀌면 모두 해당합니다.

**참고:** **기본 UCP** 서명에서 RFC 9421 `created` 파라미터는 선택 사항이며(**OPTIONAL**), 재전송 방지는 서명 타임스탬프가 아니라 비즈니스 계층의 멱등성 키로 처리합니다. **WBA 형태** 서명은 WBA 검증자를 위해 `created`/`expires`를 추가로 담으며 `nonce`도 담아야 합니다(SHOULD). 그 신선도 기간을 강제할지는 애플리케이션이 정합니다(RFC 9421 §3.2.1). [WBA 상호 운용](#wba-interop)을 참고하세요. 오래된 서명을 무효화하려면 키를 교체합니다(프로필에 공개된 키 집합에서 유출된 키를 제거).

### 서명 적용 대상

**요청:** HTTP Message Signatures를 사용한다면 플랫폼은 모든 요청에 서명해야 합니다(**SHOULD**). 다른 인증 메커니즘(API 키, OAuth, mTLS)을 대신 사용할 수도 있습니다.

**웹훅:** 웹훅 알림에는 반드시 서명해야 합니다(**MUST**). 그러지 않으면 수신자가 서버에서 시작된 푸시 메시지의 진위를 검증할 수 없습니다.

**그 밖의 응답:** 다음 응답에는 서명을 권장합니다(**RECOMMENDED**).

- 결제 승인 응답
- 체크아웃 완료 응답

다음 응답에는 서명이 선택 사항입니다(**OPTIONAL**).

- 장바구니 작업(저가치, 동기식)
- 카탈로그 쿼리(읽기 전용)
- 에러 응답(4xx, 5xx)

## MCP 전송 방식

UCP는 MCP 전송 방식으로 SSE 기반 전송 대신 **streamable HTTP**를 지정합니다. MCP 요청은 JSON-RPC 본문을 담은 표준 HTTP 요청이므로 같은 RFC 9421 서명 메커니즘이 적용됩니다.

- `Content-Digest` 헤더가 JSON-RPC 메시지 본문을 대상으로 합니다
- `Signature-Input`과 `Signature` 헤더로 인증합니다
- `UCP-Agent` 헤더는 REST와 똑같이 동작합니다. `Idempotency-Key`는 있으면 서명되지만, MCP 요청은 모두 POST이므로 필요 여부는 HTTP 메서드가 아니라 JSON-RPC 작업(상태 변경 여부)에 따라 정해집니다

**서명이 포함된 MCP 요청 예시:**

```http
POST /mcp HTTP/1.1
Host: business.example.com
Content-Type: application/json
UCP-Agent: profile="https://platform.example/.well-known/ucp"
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Digest: sha-256=:RK/0qy18MlBSVnWgjwz6lZEWjP/lF5HF9bvEF8FabDg=:
Signature-Input: sig1=("@method" "@authority" "@path" "content-digest" "content-type" "ucp-agent" "idempotency-key");keyid="platform-2026"
Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:

{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"complete_checkout","arguments":{"id":"chk_123","checkout":{...}}}}
```

JSON-RPC 메시지가 곧 HTTP 본문입니다. `Content-Digest`가 이 본문을 서명에 결합하므로 JSON 정규화는 필요 없습니다.

## 에러 처리

서명 검증 에러에는 표준 UCP 에러 코드를 사용합니다. 전체 에러 코드 레지스트리와 전송 바인딩은 명세 개요의 [에러 처리](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#error-handling)를 참고하세요.

**서명 관련 에러:**

| 코드                    | HTTP | 설명                                                    |
| ----------------------- | ---- | ------------------------------------------------------- |
| `signature_missing`     | 401  | 필수 서명 헤더/필드가 없음                              |
| `signature_invalid`     | 401  | 서명 검증 실패                                          |
| `key_not_found`         | 401  | 서명자가 공개한 키 집합에서 키 ID를 찾을 수 없음        |
| `digest_mismatch`       | 400  | 본문 다이제스트가 `Content-Digest` 헤더와 일치하지 않음 |
| `algorithm_unsupported` | 400  | 지원하지 않는 서명 알고리즘                             |

**프로필 관련 에러** (기능 협상에도 사용):

| 코드                  | HTTP | 설명                                                  |
| --------------------- | ---- | ----------------------------------------------------- |
| `invalid_profile_url` | 400  | 프로필 URL의 형식이 잘못되었거나 스킴이 유효하지 않음 |
| `profile_unreachable` | 424  | 서명자의 프로필을 가져올 수 없음                      |
| `profile_not_trusted` | 403  | 프로필 URL이 사전 승인된 플랫폼 레지스트리에 없음     |

**참고:** 재전송 방지는 서명 계층이 아니라 비즈니스 계층의 멱등성 키로 처리합니다. 중복 요청에는 서명 에러가 아니라 캐시된 응답이 반환됩니다.

### REST 에러 응답

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "code": "signature_invalid",
  "content": "Request signature verification failed for key kid=platform-2026"
}
```

### MCP 에러 응답

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "error": {
    "code": -32000,
    "message": "Signature verification failed",
    "data": {
      "code": "signature_invalid",
      "content": "Signature verification failed for key kid=platform-2026"
    }
  }
}
```

## 참고 자료

- [RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517) — JSON Web Key (JWK)
- [RFC 7518](https://datatracker.ietf.org/doc/html/rfc7518) — JSON Web Algorithms (JWA), §6.2 (EC 공개 키)
- [RFC 8032](https://datatracker.ietf.org/doc/html/rfc8032) — Edwards-Curve Digital Signature Algorithm (EdDSA)
- [RFC 8037](https://datatracker.ietf.org/doc/html/rfc8037) — CFRG Elliptic Curve Diffie-Hellman (ECDH) and Signatures in JOSE
- [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) — HTTP Message Signatures
- [RFC 9530](https://www.rfc-editor.org/rfc/rfc9530) — Digest Fields (Content-Digest)
