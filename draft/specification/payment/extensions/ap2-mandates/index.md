# AP2 위임(Mandate) 확장

## 개요

AP2 위임 확장을 사용하면 **검증 가능한 디지털 자격 증명**(Verifiable Digital Credentials)으로 사용자의 의도와 승인을 안전하게 주고받을 수 있습니다. 이 확장은 표준 쇼핑 서비스의 체크아웃 기능을 확장해 \*\*[AP2 프로토콜](https://ap2-protocol.org/)\*\*을 지원합니다.

이 기능이 협상되어 활성화되면 표준 체크아웃 세션이 암호학적으로 묶인 합의로 바뀝니다.

- **비즈니스**는 체크아웃 응답에 암호 서명을 반드시 포함해(**MUST**) 조건(가격, 라인 아이템)이 진본임을 증명해야 합니다.
- **플랫폼**은 `complete` 작업에서 암호 서명된 증명(위임)을 반드시 제공해(**MUST**), 사용자가 특정 체크아웃 상태와 자금 이체를 명시적으로 승인했음을 증명해야 합니다.

**보안 바인딩:** 이 확장이 기능 교집합에서 협상되면 세션은 **보안 잠금(Security Locked)** 상태가 됩니다. 이후에는 어느 쪽도 표준(보호되지 않은) 체크아웃 흐름으로 되돌아갈 수 없습니다.

### 설계

AP2 전용 필드는 요청과 응답 모두에서 `ap2` 객체 아래에 중첩됩니다. 이렇게 설계하면 다음과 같은 이점이 있습니다.

- **스키마 모듈성** — 기본 체크아웃 스키마는 깔끔하게 유지되고, AP2는 자신의 모든 데이터를 담은 필드 하나만 추가합니다.
- **일관된 정규화** — 비즈니스의 서명을 계산할 때 `ap2`를 제외한다는 규칙 하나만 있으면 됩니다. 앞으로 추가될 AP2 필드도 자동으로 처리됩니다.
- **확장 공존** — 여러 보안 확장이 네임스페이스 충돌 없이 함께 쓰일 수 있습니다.
- **기능 신호** — `ap2` 객체가 있으면 AP2가 활성화되었음을 분명히 알 수 있습니다.

## 디스커버리와 협상

이 확장은 표준 UCP 협상 프로토콜을 따릅니다. 비즈니스와 플랫폼 양쪽의 **기능 교집합**(Capability Intersection)에 들어 있을 때만 활성화됩니다.

### 비즈니스 프로필 선언

비즈니스는 `/.well-known/ucp`의 `capabilities` 목록에 `dev.ucp.common.payment.ap2_mandate`를 추가해 지원 여부를 선언합니다.

**비즈니스 프로필 예시:**

```json
{
  "ucp": {
    "version": "draft",
    "services": {},
    "capabilities": {
      "dev.ucp.shopping.checkout": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/shopping/checkout",
          "schema": "https://ucp.dev/draft/schemas/shopping/checkout.json"
        }
      ],
      "dev.ucp.common.payment.ap2_mandate": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/payment/extensions/ap2-mandates",
          "schema": "https://ucp.dev/draft/schemas/common/payment_ap2_mandate.json",
          "extends": "dev.ucp.shopping.checkout",
          "config": {
            "vp_formats_supported": {
              "dc+sd-jwt": { }
            }
          }
        }
      ]
    },
    "payment_handlers": {}
  }
}
```

### 플랫폼 프로필 선언

플랫폼은 자신의 프로필에 지원 여부를 선언합니다. 신뢰할 수 있는 플랫폼 제공자 모델로 운영하는 플랫폼은 프로필 최상위 `keys` 배열에 키를 적어도 하나 반드시 제공해야 합니다(**MUST**).

### 활성화와 세션 잠금

1. 플랫폼이 자신의 프로필 URI를 알립니다(전송 방식별 메커니즘).
1. 비즈니스가 프로필을 가져와 교집합을 계산합니다.
1. 교집합에 `dev.ucp.common.payment.ap2_mandate`가 있으면 다음이 적용됩니다.
   - 비즈니스는 모든 체크아웃 응답에 `ap2.merchant_authorization`을 반드시 포함해야 합니다(**MUST**).
   - 비즈니스는 `ap2.checkout_mandate`가 없는 `complete_checkout` 요청을 수락해서는 안 됩니다(**MUST NOT**).
   - 플랫폼은 사용자에게 체크아웃을 보여 주기 전에 비즈니스의 서명을 반드시 검증해야 합니다(**MUST**).

### 서명 키 요구 사항

이 확장을 사용하려면 비즈니스가 위임 서명을 검증할 때 쓸 공개 서명 키가 반드시 있어야 합니다(**MUST**).

- **플랫폼 제공자 흐름:** 플랫폼 프로필의 `keys`로 제공되는 키
- **사용자 자격 증명 흐름:** 디지털 결제 자격 증명에 바인딩된 키

공개 키를 확인할 수 없거나 서명이 유효하지 않으면 비즈니스는 반드시 에러를 반환해야 합니다(**MUST**).

## 암호학적 요구 사항

이 확장은 [메시지 서명](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/signatures/index.md) 명세에 정의된 암호 기본 요소를 사용합니다.

- **알고리즘:** AP2의 Checkout JWT 서명 규칙을 따릅니다. AP2 v0.2는 ECDSA (`ES256`/`ES384`/`ES512`)를 요구합니다. 아래 참고 사항을 확인하세요.
- **정규화:** JCS ([RFC 8785](https://datatracker.ietf.org/doc/html/rfc8785))
- **키 형식:** JWK ([RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517))
- **키 디스커버리:** `/.well-known/ucp`의 `keys[]` ([키 디스커버리](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#key-discovery) 참고)

키 형식과 교체(rotation)는 [메시지 서명](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/signatures/index.md)을 참고하세요.

> **참고(알고리즘 요구 사항).** AP2는 `hash(checkout_jwt)`로 결제 위임(Payment Mandate)을 체크아웃에 묶습니다. 근본적으로 필요한 보안 속성은 서명된 바이트를 세션마다 예측할 수 없어야 한다는 것이며, UCP에서는 세션마다 고유한 체크아웃 `id`가 구조적으로 이를 보장합니다. 그런데 AP2 v0.2는 이를 요구하는 방식이 문서 안에서 일관되지 않습니다. `specification.md`는 알고리즘 계열 규칙(비결정적 알고리즘만 허용, 예: ECDSA)을 명시하는 반면, Security & Privacy 고려 사항에는 페이로드 엔트로피만 충분하면 어떤 알고리즘이든 괜찮다는 엔트로피 규칙이 나와 있습니다. [AP2 #268](https://github.com/google-agentic-commerce/AP2/issues/268)에서 엔트로피 방식으로 정리하는 논의가 진행 중입니다. 공식 규칙은 AP2를 따릅니다. 엔트로피 해석을 따르면 UCP Checkout JWT는 어떤 알고리즘(Ed25519 포함)으로도 서명할 수 있으므로, 키 하나로 AP2 위임 서명과 Web Bot Auth를 모두 처리할 수 있습니다.

### 비즈니스 승인

비즈니스는 **JWS Detached Content** 형식 ([RFC 7515 Appendix F](https://datatracker.ietf.org/doc/html/rfc7515#appendix-F))으로 체크아웃 응답 본문의 `ap2.merchant_authorization` 아래에 자신의 서명을 반드시 포함해야 합니다(**MUST**).

**서명이 포함된 체크아웃 응답:**

```json
{
  "ucp": { ... },
  "id": "chk_abc123",
  "status": "ready_for_complete",
  "currency": "USD",
  "line_items": [ ... ],
  "totals": [ ... ],
  "links": [ ... ],
  "ap2": {
    "merchant_authorization": "eyJhbGciOiJFUzI1NiIsImtpZCI6Im1lcmNoYW50XzIwMjUifQ..signature"
  }
}
```

`merchant_authorization` 값은 `<header>..<signature>` 형식으로 페이로드를 분리한 JWS입니다. 이중 점(`..`)은 페이로드를 따로(체크아웃 본문 자체로) 전송한다는 뜻입니다.

**JWS 헤더 클레임:**

| 클레임 | 타입   | 필수 | 설명                                      |
| ------ | ------ | ---- | ----------------------------------------- |
| `alg`  | string | 예   | AP2가 허용하는 서명 알고리즘(예: `ES256`) |
| `kid`  | string | 예   | 비즈니스의 `keys`를 참조하는 키 ID        |

**서명 계산:**

서명은 반드시 JWS 헤더와 체크아웃 페이로드를 모두 포함해야 합니다(**MUST**). 그래야 공격자가 서명을 무효화하지 않은 채 `alg` 클레임을 바꾸는 알고리즘 치환 공격을 막을 수 있습니다.

```text
sign_checkout(checkout, private_key, kid, alg="ES256"):
    // Extract payload (checkout minus ap2)
    payload = checkout without "ap2" field

    // Canonicalize using JCS (RFC 8785)
    canonical_bytes = jcs_canonicalize(payload)

    // Create protected header
    header = {"alg": alg, "kid": kid}
    encoded_header = base64url_encode(json_encode(header))

    // Sign header + payload per JWS
    signing_input = encoded_header + "." + base64url_encode(canonical_bytes)
    signature = sign(signing_input, private_key, alg)

    // Return detached JWS (header..signature, no payload)
    checkout.ap2.merchant_authorization = encoded_header + ".." + base64url_encode(signature)
    return checkout
```

### 위임 구조

위임은 키 바인딩(`+kb`)을 적용한 **SD-JWT** 자격 증명입니다. 플랫폼은 서로 다른 두 가지 위임 산출물을 반드시 생성해야 합니다(**MUST**).

| 위임                 | UCP 내 위치                               | 목적                                          |
| -------------------- | ----------------------------------------- | --------------------------------------------- |
| **checkout_mandate** | `ap2.checkout_mandate`                    | 체크아웃 조건에 묶인 증명으로 비즈니스를 보호 |
| **payment_mandate**  | `payment.instruments[*].credential.token` | 결제 승인에 묶인 증명으로 자금을 보호         |

체크아웃 위임에는 `ap2.merchant_authorization` 필드를 포함한 전체 체크아웃 응답이 반드시 담겨야 합니다(**MUST**). 이렇게 하면 플랫폼의 서명이 비즈니스의 서명을 감싸는 중첩된 암호학적 바인딩이 만들어집니다.

**명세 경계:** 이 확장은 UCP 요청과 응답에서 위임이 *어디에* 들어가는지를 정의합니다. 위임 자격 증명의 구조(클레임, 선택적 공개, 키 바인딩)는 [AP2 프로토콜 명세](https://ap2-protocol.org/specification)에서 정의합니다.

### 정규화

모든 JSON 페이로드는 [RFC 8785](https://datatracker.ietf.org/doc/html/rfc8785)에 따라 **JSON Canonicalization Scheme**(JCS)으로 반드시 정규화해야 합니다(**MUST**).

**위임에 JCS를 쓰는 이유:** UCP 요청 서명은 정규화 없이 `Content-Digest`(원시 바이트)를 사용합니다. 요청은 같은 HTTP 연결에서 곧바로 서명하고 검증하기 때문입니다. 위임은 사정이 다릅니다.

- **지속성** — 위임은 사용자 동의의 증거로 저장되며, 며칠 또는 몇 달 뒤에 다시 꺼내 검증할 수 있습니다.
- **시스템 간 전송** — 위임은 JSON을 다시 직렬화할 수 있는 여러 시스템(플랫폼 → 비즈니스 → PSP → 카드 네트워크)을 거칩니다.
- **재현성** — 직렬화 방식이 달라도 어느 당사자든 논리적 JSON 내용만으로 서명된 바이트를 정확히 재구성할 수 있어야 합니다.

JCS를 쓰면 의미가 같은 JSON은 바이트 단위로도 같은 출력이 나오므로, 구현체와 시점이 달라도 서명을 재현할 수 있습니다.

검증과 위임 생성은 스키마가 아는 필드만 추린 결과가 아니라 전체 체크아웃 JSON을 대상으로 합니다. `merchant_authorization`이 다루는 멤버를 하나라도 빼면 JCS 페이로드가 바뀌어 서명이 무효가 됩니다. 이 범위에는 체크아웃에 있는 `ucp` 프로토콜 네임스페이스 멤버도 들어가며, 아래에 명시한 대로 `ap2` 필드만 제외됩니다.

**AP2 전용 규칙:** 비즈니스의 `merchant_authorization` 서명을 계산할 때는 `ap2` 필드를 통째로 제외합니다. 이렇게 하면 앞으로 추가될 AP2 필드도 자동으로 처리됩니다.

## 위임 흐름

`dev.ucp.common.payment.ap2_mandate` 기능이 협상되면 세션은 다음 흐름으로 고정됩니다. 암호학적 무결성을 보장하려면 양쪽 모두 이 단계를 반드시 따라야 하며(**MUST**), 이 단계를 우회하거나 위임 없이 완료 요청을 제출하려고 하면 반드시 세션 실패로 이어져야 합니다(**MUST**).

### 1단계: 체크아웃 생성과 서명

플랫폼이 세션을 시작합니다. 비즈니스는 응답 본문에 `ap2.merchant_authorization`이 담긴 `Checkout` 객체를 반환합니다.

AP2 위임(Mandate) 지원이 추가된 체크아웃.

| 이름         | 타입                                                                                                      | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ------------ | --------------------------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | [UCP Checkout Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-checkout-schema) | **필수**  | 체크아웃 응답용 UCP 메타데이터.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| id           | string                                                                                                    | **필수**  | 체크아웃 세션의 고유 식별자.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| line_items   | Array\[[Line Item](/ucp-ko/draft/draft/specification/reference/#line-item)\]                              | **필수**  | 체크아웃 중인 라인 아이템 목록.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| buyer        | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                                               | 선택      | 구매자 정보.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                                           | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                                           | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |
| attribution  | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                                   | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| status       | string                                                                                                    | **필수**  | 현재 단계와 필요한 처리를 나타내는 체크아웃 상태. 상태 전이에 대한 자세한 내용은 체크아웃 상태 수명 주기 문서를 참고하세요. **허용 값:** `incomplete`, `requires_escalation`, `ready_for_complete`, `complete_in_progress`, `completed`, `canceled`                                                                                                                                                                                                                                                                                                                                                                   |
| currency     | string                                                                                                    | **필수**  | 판매자가 판단한 시장을 반영하는 ISO 4217 통화 코드. 주소, 컨텍스트, geo IP로 결정합니다. 구매자는 신호를 제공하고 통화는 판매자가 결정합니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| totals       | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                             | **필수**  | 장바구니의 각종 합계.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| actions      | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                           | 선택      | 이 체크아웃에 대해 확장이 정의한 미해결 Action.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                  | 선택      | 체크아웃 세션 상태에 대한 오류·정보 메시지 목록.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]                                        | **필수**  | 플랫폼이 표시할 링크(개인정보처리방침, 이용약관). 법적 준수를 위해 필수입니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| policies     | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                    | 선택      | 이 체크아웃의 항목에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| expires_at   | string                                                                                                    | 선택      | RFC 3339 만료 타임스탬프입니다. 보내지 않으면 기본 TTL은 생성 시점부터 6시간입니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| continue_url | string                                                                                                    | 선택      | 체크아웃 핸드오프와 세션 복구용 URL. status가 requires_escalation이면 반드시 제공해야 합니다(MUST). 형식과 가용성 요구 사항은 명세를 참고하세요.                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| payment      | [Payment](/ucp-ko/draft/draft/specification/reference/#payment)                                           | 선택      | 핸들러를 포함한 결제 구성.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| order        | [Order Confirmation](/ucp-ko/draft/draft/specification/reference/#order-confirmation)                     | 선택      | 이 체크아웃 세션으로 생성된 주문의 세부 정보.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| ap2          | object                                                                                                    | 선택      |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |

**응답 예시:**

```json
{
  "ucp": { ... },
  "id": "chk_abc123",
  "status": "ready_for_complete",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_1",
      "item": {"id": "item_123", "title": "Widget", "price": 2500},
      "quantity": 2,
      "totals": [
        {"type": "subtotal", "amount": 5000},
        {"type": "total", "amount": 5000}
      ]
    }
  ],
  "totals": [
    {"type": "subtotal", "amount": 5000},
    {"type": "tax", "amount": 400},
    {"type": "total", "amount": 5400}
  ],
  "links": [ ... ],
  "ap2": {
    "merchant_authorization": "eyJhbGciOiJFUzI1NiIsImtpZCI6Im1lcmNoYW50XzIwMjUifQ..signature"
  }
}
```

플랫폼은 반드시 서명을 검증해야 합니다(**MUST**).

```text
verify_merchant_authorization(checkout, merchant_profile):
    // Parse detached JWS (header..signature)
    jws = checkout.ap2.merchant_authorization
    [encoded_header, empty, encoded_signature] = jws.split(".")

    // Decode and validate header
    header = json_decode(base64url_decode(encoded_header))
    assert header.alg in ap2_accepted_algorithms  // ES256/ES384/ES512 per AP2 v0.2

    // Reconstruct signed payload (checkout minus ap2)
    payload = checkout without "ap2" field
    canonical_bytes = jcs_canonicalize(payload)

    // Reconstruct signing input (header + payload)
    signing_input = encoded_header + "." + base64url_encode(canonical_bytes)

    // Get business's public key and verify
    public_key = get_key_by_kid(merchant_profile.keys, header.kid)
    return verify(encoded_signature, signing_input, public_key, header.alg)
```

### 2단계: 사용자 동의와 위임 생성

사용자가 구매를 확인하면, 플랫폼은 암호학적으로 검증할 수 있는 위임이 생성되도록 반드시 지원해야 합니다(**MUST**).

#### 옵션 1: 신뢰할 수 있는 플랫폼 제공자

신뢰할 수 있는 플랫폼 제공자가 사용자를 대신해 위임 자격 증명을 생성합니다. 플랫폼 제공자는 신뢰할 수 있고 결정론적인 채널로 사용자의 명시적 동의를 받지 않고는 위임이 생성되지 않도록 반드시 보장해야 합니다(**MUST**).

사용자가 동의하면 플랫폼은 서버 측 키로 위임에 서명합니다. 비즈니스는 플랫폼의 서명을 사용자 동의의 표시로 신뢰합니다.

#### 옵션 2: 디지털 결제 자격 증명

이 모델에서 사용자는 비즈니스가 신뢰하는 출처에서 발급받은 VDC(예: 은행이나 네트워크가 발급한 디지털 결제 자격 증명)를 보유하고 있습니다.

플랫폼은 OpenID4VP 같은 프로토콜로 제시(presentation)를 요청합니다. 사용자의 지갑(또는 그에 준하는 수단)이 요청을 처리하고, 결제 자격 증명에 연결된 개인 키로 위임에 서명합니다.

비즈니스는 자격 증명 발급자(은행)를 신뢰하고, 사용자의 키 바인딩(+kb) 서명을 검증합니다.

### 3단계: 제출(`complete_checkout`)

위임이 생성되면 플랫폼은 이를 완료 요청에 담아 제출합니다.

체크아웃 위임(Mandate)을 포함한 AP2 확장 데이터.

| 이름             | 타입   | 필수 여부 | 설명                                                  |
| ---------------- | ------ | --------- | ----------------------------------------------------- |
| checkout_mandate | string | 선택      | 사용자가 이 체크아웃을 승인했음을 증명하는 SD-JWT+kb. |

```json
{
  "payment": {
    "instruments": [
      {
        "id": "instr_1",
        "handler_id": "gpay_1234",
        "type": "card",
        "selected": true,
        "display": {
          "description": "Visa •••• 1234"
        },
        "billing_address": {
          "street_address": "123 Main St",
          "address_locality": "Anytown",
          "address_region": "CA",
          "address_country": "US",
          "postal_code": "12345"
        },
        "credential": {
          "type": "PAYMENT_GATEWAY",
          "token": "examplePaymentMethodToken"
        }
      }
    ]
  },
  "ap2": {
    "checkout_mandate": "eyJhbGciOiJFUzI1NiIsInR5cCI6InZjK3NkLWp3dCJ9.payload.signature" // The User-Signed SD-JWT+kb / platform provider signed SD-JWT / delegated SD-JWT-KB
  }
}
```

- `ap2.checkout_mandate`: 전체 체크아웃(`ap2.merchant_authorization` 포함)을 담은 SD-JWT+kb 체크아웃 위임
- `payment.instruments[*].credential.token`: 결제 위임(복합 토큰)

## 검증과 처리

### 비즈니스 검증

`complete` 요청을 받으면 비즈니스는 반드시 다음을 수행해야 합니다(**MUST**).

1. **협상 강제:** AP2가 협상되었는데 `ap2.checkout_mandate`가 없으면 `mandate_required` 에러 코드로 요청을 거부합니다.

**위임 검증(AP2 명세에 따름):**

1. **위임 검증:** [AP2 프로토콜 명세](https://ap2-protocol.org/specification)에 따라 SD-JWT를 디코딩하고 서명, 키 바인딩, 만료를 검증합니다.
1. **포함된 체크아웃 추출:** 검증된 위임 클레임에서 체크아웃 객체를 꺼냅니다.

**UCP 검증:**

1. **비즈니스 승인 검증:** 포함된 체크아웃의 `ap2.merchant_authorization`이 비즈니스 자신이 만든 유효한 서명인지 확인합니다.

   ```text
   jws = embedded_checkout.ap2.merchant_authorization
   [encoded_header, _, encoded_signature] = jws.split(".")
   header = json_decode(base64url_decode(encoded_header))

   payload = embedded_checkout without "ap2" field
   signing_input = encoded_header + "." + base64url_encode(jcs_canonicalize(payload))

   my_key = get_key_by_kid(my_keys, header.kid)
   verify(encoded_signature, signing_input, my_key, header.alg)
   ```

1. **조건 일치 검증:** 포함된 체크아웃의 조건이 현재 세션 상태(id, totals, line items)와 일치하는지 확인합니다.

### PSP 검증

비즈니스는 `token`(복합 객체)을 자신의 결제 핸들러 / PSP에 전달합니다. PSP는 [AP2 프로토콜 명세](https://ap2-protocol.org/specification)에 따라 `payment_mandate`를 검증합니다. 여기에는 서명 검증, 만료 확인, 체크아웃과의 연관성 확인이 포함됩니다.

## 스키마

### 비즈니스 승인

체크아웃 응답 본문(ap2 필드 제외)에 대한 JWS Detached Content 서명(RFC 7515 Appendix F). 형식: `<base64url-header>..<base64url-signature>`. 헤더에는 반드시 'alg'(ES256/ES384/ES512)와 'kid' 클레임이 있어야 합니다(MUST). 서명은 헤더와 JCS로 정규화한 체크아웃 페이로드를 모두 대상으로 합니다.

**패턴:** `^[A-Za-z0-9_-]+\.\.[A-Za-z0-9_-]+$`

### AP2 체크아웃 응답

체크아웃 응답에 포함되는 `ap2` 객체입니다.

판매자 인가를 포함한 AP2 확장 데이터.

| 이름                   | 타입   | 필수 여부 | 설명                                             |
| ---------------------- | ------ | --------- | ------------------------------------------------ |
| merchant_authorization | string | 선택      | 체크아웃 조건이 진본임을 증명하는 판매자의 서명. |

### 체크아웃 위임

`ap2.checkout_mandate`에 담기는 SD-JWT+kb 자격 증명입니다. 사용자가 체크아웃을 승인했음을 증명합니다. `ap2.merchant_authorization`을 포함한 체크아웃 전체가 들어 있습니다.

**패턴:** `^[A-Za-z0-9_-]+\.[A-Za-z0-9_-]*\.[A-Za-z0-9_-]+(~[A-Za-z0-9_-]+)*$`

### AP2 완료 요청

COMPLETE 체크아웃 요청에 포함되는 `ap2` 객체입니다.

체크아웃 위임(Mandate)을 포함한 AP2 확장 데이터.

| 이름             | 타입   | 필수 여부 | 설명                                                  |
| ---------------- | ------ | --------- | ----------------------------------------------------- |
| checkout_mandate | string | 선택      | 사용자가 이 체크아웃을 승인했음을 증명하는 SD-JWT+kb. |

### 에러 코드

AP2 위임(Mandate) 검증에 특화된 오류 코드.

**허용 값:** `mandate_required`, `agent_missing_key`, `mandate_invalid_signature`, `mandate_expired`, `mandate_scope_mismatch`, `merchant_authorization_invalid`, `merchant_authorization_missing`

| 에러 코드                        | 설명                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| `mandate_required`               | AP2가 협상되었지만 요청에 `ap2.checkout_mandate`가 없습니다. |
| `agent_missing_key`              | 플랫폼 프로필에 유효한 `keys` 항목이 없습니다.               |
| `mandate_invalid_signature`      | 위임 서명을 검증할 수 없습니다.                              |
| `mandate_expired`                | 위임의 `exp` 타임스탬프가 지났습니다.                        |
| `mandate_scope_mismatch`         | 위임이 다른 체크아웃에 묶여 있습니다.                        |
| `merchant_authorization_invalid` | 비즈니스 승인 서명을 검증할 수 없었습니다.                   |
| `merchant_authorization_missing` | 체크아웃 응답에 `ap2.merchant_authorization`이 없습니다.     |
