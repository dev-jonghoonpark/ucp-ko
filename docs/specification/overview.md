<!--
   Copyright 2026 UCP Authors

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->

# Universal Commerce Protocol(UCP) 공식 명세 { #universal-commerce-protocol-ucp-official-specification }

## 전반 지침 { #overarching-guidelines }

이 문서에 나오는 **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, **OPTIONAL** 키워드는
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.html){ target="_blank" } 및
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174.html){ target="_blank" }에
정의된 의미로 해석합니다.

스키마 참고 사항:

- 날짜 형식: 별도 명시가 없으면 항상
    [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339.html){ target="_blank" }
    형식
- 금액 형식: 최소 단위(센트)

## 디스커버리, 거버넌스, 협상 { #discovery-governance-and-negotiation }

UCP는 프로토콜 버전 호환성과 기능 협상을 분리합니다.
`/.well-known/ucp`에 있는 비즈니스 프로필은 그 프로필이 선언한 프로토콜 버전의
기능을 기술합니다. 이전 프로토콜 버전을 지원하는 비즈니스는 버전별 프로필을
게시하고, 프로토콜 버전을 프로필 URI에 매핑하는 `supported_versions` 필드로 이를
알려야 합니다(**SHOULD**). 이렇게 하면 플랫폼이 특정 프로토콜 버전에서 쓸 수 있는
기능을 정확히 찾을 수 있습니다. 이전 버전을 언제 지원 중단하거나
`supported_versions`에서 제거할지를 포함한 버전 수명 주기는 비즈니스가 정할
정책이며, 프로토콜은 지원 중단(deprecation) 일정을 규정하지 않습니다.
기능 협상은 서버가 선택하는 구조를 따릅니다. 즉 비즈니스(서버)가 양측이 선언한
기능의 교집합에서 활성 기능을 결정합니다. 비즈니스 프로필과 플랫폼 프로필은
양측 모두 캐시할 수 있으므로, 플랫폼과 비즈니스 사이의 일반적인 요청/응답 흐름
안에서 기능을 효율적으로 협상할 수 있습니다.

### 네임스페이스 거버넌스 { #namespace-governance }

UCP는 역도메인(reverse-domain) 명명 방식으로 거버넌스 권한을 기능 식별자에 직접
담습니다. 그래서 중앙 레지스트리가 필요 없습니다.

#### 명명 규칙 { #naming-convention }

모든 기능 이름과 서비스 이름은 반드시 다음 형식을 따라야 합니다(**MUST**).

```text
{reverse-domain}.{service}.{capability}
```

**구성 요소:**

- `{reverse-domain}` - 도메인 소유권에서 도출되는 권한 주체 식별자
- `{service}` - 서비스/버티컬 분류(예: `shopping`, `common`)
- `{capability}` - 구체적인 기능 이름

**예시:**

| 이름                                | 권한 주체   | 서비스   | 기능             |
| ----------------------------------- | ----------- | -------- | ---------------- |
| `dev.ucp.shopping.checkout`         | ucp.dev     | shopping | checkout         |
| `dev.ucp.shopping.fulfillment`      | ucp.dev     | shopping | fulfillment      |
| `dev.ucp.common.identity_linking`   | ucp.dev     | common   | identity_linking |
| `com.example.payments.installments` | example.com | payments | installments     |

#### 명세 URL 바인딩 { #spec-url-binding }

모든 기능에서 `spec` 필드와 `schema` 필드는 필수입니다(**REQUIRED**). 이 URL의
오리진은 반드시 네임스페이스 권한 주체와 일치해야 합니다(**MUST**).

| 네임스페이스    | 필수 오리진               |
| --------------- | ------------------------- |
| `dev.ucp.*`     | `https://ucp.dev/...`     |
| `com.example.*` | `https://example.com/...` |

플랫폼은 반드시 이 바인딩을 검증해야 하며(**MUST**), 명세 오리진이 네임스페이스
권한 주체와 일치하지 않는 기능은 거부해야 합니다(**SHOULD**).

#### 거버넌스 모델 { #governance-model }

| 네임스페이스 패턴 | 권한 주체    | 거버넌스            |
| ----------------- | ------------ | ------------------- |
| `dev.ucp.*`       | ucp.dev      | UCP 운영 기구       |
| `com.{vendor}.*`  | {vendor}.com | 벤더 조직           |
| `org.{org}.*`     | {org}.org    | 조직                |

`dev.ucp.*` 네임스페이스는 UCP 운영 기구가 승인한 기능용으로 예약되어 있습니다.
벤더는 커스텀 기능에 반드시 자체 역도메인 네임스페이스를 사용해야 합니다(**MUST**).

### 서비스 { #services }

**서비스**는 하나의 버티컬(shopping, common 등)에 대한 API 표면을 정의합니다.
서비스에는 표준 형식으로 정의한 오퍼레이션, 이벤트, 전송 바인딩이 들어 있습니다.

- **REST**: OpenAPI 3.x(JSON 형식)
- **MCP**: OpenRPC(JSON 형식)
- **A2A**: Agent Card 명세
- **EP(embedded)**: OpenRPC(JSON 형식)

#### 서비스 정의 { #service-definition }

{{ extension_schema_fields('service.json#/$defs/platform_schema', 'overview') }}

전송 방식 정의는 반드시 얇아야 합니다(**MUST**). 즉 메서드 이름을 선언하고 기본
스키마를 참조하기만 합니다. 자세한 내용은 [요구 사항](#requirements)을 참고하세요.

#### 엔드포인트 해석 { #endpoint-resolution }

`endpoint` 필드는 API 호출의 기본 URL입니다. 이 엔드포인트 뒤에 OpenAPI 경로를
붙여 전체 URL을 만듭니다.

**예시:**

```json
{
  "version": "{{ ucp_version }}",
  "transport": "rest",
  "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/rest.openapi.json",
  "endpoint": "https://business.example.com/api/v2"
}
```

OpenAPI 경로가 `/checkout-sessions`라면 해석된 URL은 다음과 같습니다.

```text
POST https://business.example.com/api/v2/checkout-sessions
```

**규칙:**

- `endpoint`는 반드시 스킴(https)을 포함한 유효한 URL이어야 합니다(**MUST**).
- `endpoint` 끝에는 슬래시를 두지 않는 것이 좋습니다(**SHOULD NOT**).
- OpenAPI 경로는 상대 경로이며 endpoint 바로 뒤에 붙습니다.
- JSON-RPC 호출용 MCP 엔드포인트에도 같은 해석 방식을 적용합니다.
- A2A 전송 방식의 `endpoint`는 에이전트의 Agent Card URL을 가리킵니다.

### 기능 { #capabilities }

**기능**(capability)은 서비스 안의 기능 단위 하나입니다. 기능은 무엇을
지원하는지, 문서와 스키마를 어디서 찾을 수 있는지를 선언합니다.

#### 기능 정의 { #capability-definition }

{{ extension_schema_fields('capability.json#/$defs/platform_schema', 'capability-schema') }}

#### 확장 { #extensions }

**확장**(extension)은 다른 기능을 보강하는 선택적 모듈입니다. 확장은 `extends`
필드로 자신의 부모(들)를 선언합니다.

```json
{
  "dev.ucp.shopping.fulfillment": [
    {
      "version": "{{ ucp_version }}",
      "spec": "https://ucp.dev/{{ ucp_version }}/specification/fulfillment",
      "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
      "extends": "dev.ucp.shopping.checkout"
    }
  ]
}
```

##### 다중 부모 확장 { #multi-parent-extensions }

확장은 배열로 여러 부모 기능을 확장할 수 있습니다(**MAY**).

```json
{
  "dev.ucp.shopping.discount": [
    {
      "version": "{{ ucp_version }}",
      "spec": "https://ucp.dev/{{ ucp_version }}/specification/discount",
      "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/discount.json",
      "extends": ["dev.ucp.shopping.checkout", "dev.ucp.shopping.cart"]
    }
  ]
}
```

확장이 여러 부모를 선언하는 경우:

- 확장은 자신이 확장하는 기능마다 다른 필드를 정의할 수 있습니다(**MAY**)
    (예: 체크아웃에는 `loyalty_earned`, 장바구니에는 `loyalty_preview`).
- 협상 규칙은 [교집합 알고리즘](#intersection-algorithm)을 참고하세요.

확장의 종류는 다음과 같습니다.

- **공식**: `dev.ucp.shopping.fulfillment`는 `dev.ucp.shopping.checkout`을 확장합니다.
- **벤더**: `com.example.installments`는 `dev.ucp.shopping.checkout`을 확장합니다.

### 스키마 조합 { #schema-composition }

확장은 새 필드를 추가하고 공유 구조를 수정할 수 있습니다(예: 할인은 `totals`를
수정하고, 풀필먼트는 `totals.type`에 fulfillment를 추가합니다).

#### 요구 사항 { #requirements }

- 전송 방식 정의(OpenAPI/OpenRPC)는 반드시 기본 스키마만 참조해야
    합니다(**MUST**). 필드를 나열하거나 페이로드 형태를 인라인으로 정의해서는
    안 됩니다(**MUST NOT**).
- 확장은 반드시 자기 기술적이어야 합니다(**MUST**). 각 확장 스키마는 자신이
    도입하는 타입과, `allOf` 조합으로 기본 타입을 어떻게 수정하는지를 반드시
    선언해야 합니다(**MUST**).
- 플랫폼은 기본 스키마와 활성 확장 스키마를 가져와 조합해서, 반드시
    클라이언트 측에서 스키마를 해석해야 합니다(**MUST**).

#### 확장 스키마 패턴 { #extension-schema-pattern }

확장 스키마는 `allOf`로 조합 타입을 정의합니다. 스키마를 결정적으로 해석할 수
있도록 `$defs` 키에는 반드시 부모 기능의 전체 이름(역도메인 형식)을 써야
합니다(**MUST**).

```json
{
  "$defs": {
    "discounts_object": { ... },
    "dev.ucp.shopping.checkout": {
      "title": "Checkout with Discount",
      "allOf": [
        {"$ref": "checkout.json"},
        {
          "type": "object",
          "properties": {
            "discounts": {
              "$ref": "#/$defs/discounts_object"
            }
          }
        }
      ]
    }
  }
}
```

**요구 사항:**

- 확장 스키마에는 반드시 `extends`에 선언한 부모마다 `$defs` 항목이
    있어야 합니다(**MUST**).
- `$defs` 키는 반드시 부모 기능의 전체 이름과 정확히 일치해야 합니다(**MUST**).

이 규칙으로 다음이 보장됩니다.

- **자기 문서화**: 스키마가 어떤 부모를 확장하는지 정확히 선언합니다.
- **결정적 해석**: `extends` 값이 `$defs` 키에 바로 대응합니다.
- **검증 가능성**: 빌드 시점 검사로 각 `extends` 항목에 대응하는 `$defs` 키가
    있는지 확인할 수 있습니다.

##### 버전 요구 사항 { #version-requirements }

확장 스키마는 올바르게 동작하는 데 필요한 프로토콜 버전과 기능 버전을 나타내도록
(`name`, `title`, `description`과 함께) `requires` 객체를 선언해야
합니다(**SHOULD**).

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://acme.com/ucp/schemas/loyalty.json",
  "name": "com.acme.shopping.loyalty",
  "title": "Acme Loyalty Points",
  "requires": {
    "protocol": { "min": "2026-01-23" },
    "capabilities": {
      "dev.ucp.shopping.checkout": { "min": "2026-06-01" }
    }
  },
  "$defs": {
    "dev.ucp.shopping.checkout": { ... }
  }
}
```

버전 요구 사항은 프로필 게시자가 아니라 스키마 작성자가 선언합니다. 프로필
게시자는 호환되는 버전을 골라 자신의 프로필에 공개합니다.

각 제약은 필수인 `min`(포함)과 선택 사항인 `max`(포함) 버전으로 이루어진
객체입니다. `max`가 없으면 상한도 없습니다.

```json
"requires": {
  "protocol": { "min": "2026-01-23", "max": "2026-09-01" },
  "capabilities": {
    "dev.ucp.shopping.checkout": { "min": "2026-06-01" }
  }
}
```

`requires.capabilities`의 키는 반드시 확장의 `$defs` 키의 부분집합이어야
합니다(**MUST**). `requires`가 있으면 플랫폼과 비즈니스는 스키마를 해석하면서
협상된 프로토콜 버전과 기능 버전이 선언된 제약을 충족하는지 반드시 검증해야
합니다(**MUST**). 호환되지 않는 확장은 활성 기능 집합에서 빠집니다
([해석 흐름](#resolution-flow) 참고). `requires`가 없으면 해당 확장은 프로필이
선언한 버전과 호환된다고 봅니다.

#### 스키마 해석 규칙 { #schema-resolution-convention }

페이로드를 검증하려면 구현체는 다음과 같이 확장 스키마를 해석합니다.

1. 오퍼레이션에서 루트 기능을 정합니다(예: 체크아웃 오퍼레이션은
    `dev.ucp.shopping.checkout`을 사용합니다).
2. 각 활성 확장마다 해당 확장의 `$defs[{root_capability}]`를 해석해 적용합니다.

**예시:** 체크아웃 응답에 할인 확장이 포함되는 경우.

- 루트 기능: `dev.ucp.shopping.checkout`
- 확장 스키마: `discount.json`
- 해석 대상: `discount.json#/$defs/dev.ucp.shopping.checkout`

#### 해석 흐름 { #resolution-flow }

플랫폼은 반드시 다음 순서대로 스키마를 해석해야 합니다(**MUST**).

1. **디스커버리**: `/.well-known/ucp`에서 비즈니스 프로필을 가져옵니다.
2. **협상**: 기능 교집합을 계산합니다
    ([교집합 알고리즘](#intersection-algorithm) 참고).
3. **스키마 가져오기**: 기본 스키마와 모든 활성 확장 스키마를 가져옵니다.
4. **버전 호환성**: 가져온 확장 스키마에 `requires`가 있으면, 협상된
    프로토콜 버전과 기능 버전이 선언된 제약을 충족하는지 검증합니다. 호환되지
    않는 확장을 빼고, 고아가 된 확장을 다시 정리합니다
    ([교집합 알고리즘](#intersection-algorithm)의 3-4단계).
5. **조합**: 활성 확장을 기준으로 `allOf` 체인으로 스키마를 병합합니다.
6. **검증**: 조합된 스키마로 요청과 응답을 검증합니다.

### 프로필 구조 { #profile-structure }

#### 비즈니스 프로필 { #business-profile }

비즈니스는 `/.well-known/ucp`에 프로필을 게시합니다. 예시는 다음과 같습니다.

```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview",
          "transport": "rest",
          "endpoint": "https://business.example.com/ucp/v1",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/rest.openapi.json"
        },
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview",
          "transport": "mcp",
          "endpoint": "https://business.example.com/ucp/mcp",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/mcp.openrpc.json"
        },
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview",
          "transport": "a2a",
          "endpoint": "https://business.example.com/.well-known/agent-card.json"
        },
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview",
          "transport": "embedded",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/embedded.openrpc.json"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.checkout": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/checkout",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/checkout.json"
        }
      ],
      "dev.ucp.shopping.fulfillment": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/fulfillment",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
          "extends": "dev.ucp.shopping.checkout"
        }
      ],
      "dev.ucp.shopping.discount": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/discount",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/discount.json",
          "extends": "dev.ucp.shopping.checkout"
        }
      ],
      "dev.ucp.common.identity_linking": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/identity-linking",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/common/identity_linking.json",
          "config": {
            "scopes": {
              "dev.ucp.shopping.order:read":   {},
              "dev.ucp.shopping.order:manage": {}
            }
          }
        }
      ]
    },
    "payment_handlers": {
      "com.example.processor_tokenizer": [
        {
          "id": "processor_tokenizer",
          "version": "{{ ucp_version }}",
          "spec": "https://example.com/specs/payments/processor_tokenizer",
          "schema": "https://example.com/specs/payments/merchant_tokenizer.json",
          "available_instruments": [
            {
              "type": "card",
              "constraints": {
                "brands": ["visa", "mastercard", "amex"]
              }
            }
          ],
          "config": {
            "type": "CARD",
            "tokenization_specification": {
              "type": "PUSH",
              "parameters": {
                "token_retrieval_url": "https://api.psp.example.com/v1/tokens"
              }
            }
          }
        }
      ]
    }
  },
  "signing_keys": [
    {
      "kid": "business_2025",
      "kty": "EC",
      "crv": "P-256",
      "x": "WbbXwVYGdJoP4Xm3qCkGvBRcRvKtEfXDbWvPzpPS8LA",
      "y": "sP4jHHxYqC89HBo8TjrtVOAGHfJDflYxw7MFMxuFMPY",
      "use": "sig",
      "alg": "ES256"
    }
  ]
}
```

`ucp` 객체에는 프로토콜 메타데이터(버전, 서비스, 기능, 결제 핸들러)가 담깁니다.
`signing_keys` 배열에는 비즈니스가 보내는 웹훅과 그 밖의 인증된 메시지의 서명을
검증하는 데 쓰는 공개 키(JWK 형식)가 담깁니다. 키 조회와 해석은
[키 디스커버리](#key-discovery)를, 서명 방식은 [메시지 서명](signatures.md)을
참고하세요.

이전 프로토콜 버전을 지원하는 비즈니스는 각 이전 버전을 버전별 프로필 URI에
매핑하는 `supported_versions` 객체를 포함해야 합니다(**SHOULD**). 자세한 내용은
[프로토콜 버전](#protocol-version)을 참고하세요.

#### 플랫폼 프로필 { #platform-profile }

플랫폼 프로필도 비슷하며, 암호학적 검증이 필요한 기능을 위한 서명 키를
포함합니다. 기능에는 기능별 설정(예: 콜백 URL, 기능 플래그)을 담는 `config` 객체를
넣을 수 있습니다(**MAY**). 예시는 다음과 같습니다.

```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview",
          "transport": "rest",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/rest.openapi.json"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.checkout": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/checkout",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/checkout.json"
        }
      ],
      "dev.ucp.shopping.fulfillment": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/fulfillment",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
          "extends": "dev.ucp.shopping.checkout"
        }
      ],
      "dev.ucp.shopping.order": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/order",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/order.json",
          "config": {
            "webhook_url": "https://platform.example.com/webhooks/ucp/orders"
          }
        }
      ],
      "dev.ucp.common.identity_linking": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/identity-linking",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/common/identity_linking.json"
        }
      ]
    },
    "payment_handlers": {
      "com.google.pay": [
        {
          "id": "gpay_1234",
          "version": "2024-12-03",
          "spec": "https://developers.google.com/merchant/ucp/guides/gpay-payment-handler",
          "schema": "https://pay.google.com/gp/p/ucp/2026-01-11/schemas/gpay_config.json"
        }
      ],
      "dev.shopify.shop_pay": [
        {
          "id": "shop_pay_1234",
          "version": "{{ ucp_version }}",
          "spec": "https://shopify.dev/ucp/shop-pay-handler",
          "schema": "https://shopify.dev/ucp/schemas/shop-pay-config.json",
          "available_instruments": [
            {"type": "shop_pay"}
          ]
        }
      ],
      "dev.ucp.processor_tokenizer": [
        {
          "id": "processor_tokenizer",
          "version": "{{ ucp_version }}",
          "spec": "https://example.com/specs/payments/processor_tokenizer-payment",
          "schema": "https://example.com/schemas/payments/delegate-payment.json",
          "available_instruments": [
            {"type": "card", "constraints": {"brands": ["visa", "mastercard"]}}
          ]
        }
      ]
    }
  },
  "signing_keys": [
    {
      "kid": "platform_2025",
      "kty": "EC",
      "crv": "P-256",
      "x": "MKBCTNIcKUSDii11ySs3526iDZ8AiTo7Tu6KPAqv7D4",
      "y": "4Etl6SRW2YiLUrN5vfvVHuhp7x8PxltmWWlbbM4IFyM",
      "use": "sig",
      "alg": "ES256"
    }
  ]
}
```

### 요청 시 플랫폼 프로필 전달 { #platform-advertisement-on-request }

플랫폼은 기능 협상이 가능하도록 모든 요청에 반드시 자신의 프로필 URI를 담아야
합니다(**MUST**).

**HTTP 전송 방식:** 플랫폼은 UCP-Agent 헤더에 반드시 Dictionary Structured Field
문법([RFC 8941](https://datatracker.ietf.org/doc/html/rfc8941){ target="_blank" })을
사용해야 합니다(**MUST**).

```text
POST /checkout HTTP/1.1
UCP-Agent: profile="https://agent.example/profiles/shopping-agent.json"
Content-Type: application/json

{"line_items": [...]}
```

**MCP 전송 방식:** 플랫폼은 반드시 요청 메타데이터를 담은 `meta` 객체를 포함해야
합니다(**MUST**).

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "create_checkout",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://agent.example/profiles/shopping-agent.json"
        }
      },
      "checkout": {
        "line_items": [...]
      }
    }
  },
  "id": 1
}
```

### 협상 프로토콜 { #negotiation-protocol }

#### 플랫폼 요구 사항 { #platform-requirements }

1. **프로필 전달**: 플랫폼은 전송 방식에 맞는 메커니즘으로 모든 요청에
    반드시 자신의 프로필 URI를 포함해야 합니다(**MUST**).
2. **디스커버리**: 플랫폼은 요청을 시작하기 전에 `/.well-known/ucp`에서 비즈니스
    프로필을 가져올 수 있습니다(**MAY**). 가져왔다면 HTTP cache-control
    지시어에 따라 프로필을 캐시해야 합니다(**SHOULD**).
3. **네임스페이스 검증**: 플랫폼은 기능의 `spec` URI 오리진이 네임스페이스
    권한 주체와 일치하는지 반드시 검증해야 합니다(**MUST**).
4. **스키마 해석**: 플랫폼은 요청을 보내기 전에 협상된 기능의 스키마를 반드시
    가져와 조합해야 합니다(**MUST**).

#### 비즈니스 요구 사항 { #business-requirements }

1. **프로필 해석**: 플랫폼 프로필 URI가 담긴 요청을 받으면, 비즈니스는 이미
    캐시해 두지 않았다면 반드시 플랫폼 프로필을 가져와 검증해야 합니다(**MUST**).
2. **기능 교집합**: 비즈니스는 반드시 플랫폼 기능과 비즈니스 기능의 교집합을
    계산해야 합니다(**MUST**).
3. **확장 검증**: 교집합에 부모 기능이 없는 확장은 반드시 제외해야 합니다(**MUST**).
4. **응답 요구 사항**: 비즈니스는 모든 응답에 다음을 담은 `ucp` 필드를 반드시
    포함해야 합니다(**MUST**).
    - `version`: 요청 처리에 사용한 UCP 버전
    - `capabilities`: 이 응답에서 활성화된 기능의 배열

#### 교집합 알고리즘 { #intersection-algorithm }

기능 교집합 알고리즘은 세션에서 어떤 기능을 활성화할지 정합니다.

1. **교집합 계산**: 각 비즈니스 기능마다 `name`이 같은 플랫폼 기능이
    있으면 결과에 넣습니다.

2. **버전 선택**: 교집합의 각 기능마다 비즈니스 배열과 플랫폼 배열
    **양쪽에** 있는 버전 문자열 집합을 계산합니다. 집합이 비어 있지 않으면
    **가장 높은** 버전(가장 최근 날짜)을 고릅니다. 집합이 비어 있으면(공통 버전
    없음) 해당 기능을 교집합에서 **제외합니다**.

3. **고아 확장 정리**: `extends`가 설정되어 있지만 부모 기능 중 **어느 것도**
    교집합에 없는 기능을 제거합니다.
    - 단일 부모 확장(`extends: "string"`): 부모가 반드시 있어야 합니다.
    - 다중 부모 확장(`extends: ["a", "b"]`): 부모가 하나 이상 있어야
        합니다.

4. **정리 반복**: 더는 제거되는 기능이 없을 때까지 3단계를 반복합니다(전이적인
    확장 체인을 처리하기 위해서입니다).

결과는 양측이 서로 호환되는 버전으로 지원하고 확장 의존성까지 충족된 기능
집합입니다.

#### 오류 처리 { #error-handling }

UCP 협상이 실패하는 경우는 두 가지입니다.

1. **디스커버리 실패**: 비즈니스가 플랫폼 프로필을 가져오거나 파싱할 수 없습니다.

2. **협상 실패**: 제공된 프로필은 유효하지만 기능 교집합이 비어 있거나 버전이
   호환되지 않습니다.

디스커버리 실패는 전송 계층 오류입니다. 필요한 입력을 가져오지 못했거나 입력
형식이 잘못된 경우입니다. 협상 실패는 비즈니스 결과입니다. 핸들러가 제공된
입력으로 실행된 뒤 그 결과를 UCP 응답으로 보고한 것입니다.

- **디스커버리 또는 버전 실패** → 선택적 `continue_url`을 담은 전송 계층 오류
- **기능 협상 실패** → 선택적 `continue_url`을 담은 UCP 응답

##### 오류 코드 { #error-codes }

**협상 오류:**

| 코드                        | 설명                                                 | REST | MCP    |
| --------------------------- | ---------------------------------------------------- | ---- | ------ |
| `invalid_profile_url`       | 프로필 URL 형식이 잘못되었거나, 없거나, 해석할 수 없음 | 400  | -32001 |
| `profile_unreachable`       | URL은 해석했지만 가져오기 실패(타임아웃, 2xx 외 응답) | 424  | -32001 |
| `profile_malformed`         | 가져온 내용이 유효한 JSON이 아니거나 스키마를 위반함 | 422  | -32001 |
| `version_unsupported`       | 플랫폼의 프로토콜 버전을 지원하지 않음               | 422  | -32001 |
| `capabilities_incompatible` | 교집합에 호환되는 기능이 없음                        | 200  | result |

**서명 오류:**

| 코드                   | 설명                                                   | REST | MCP    |
| ---------------------- | ------------------------------------------------------ | ---- | ------ |
| `signature_missing`    | 필요한 서명 헤더/필드가 없음                           | 401  | -32000 |
| `signature_invalid`    | 서명 검증 실패                                         | 401  | -32000 |
| `key_not_found`        | 서명자의 `signing_keys`에서 키 ID를 찾을 수 없음       | 401  | -32000 |
| `digest_mismatch`      | 본문 다이제스트가 `Content-Digest` 헤더와 일치하지 않음 | 400  | -32600 |
| `algorithm_unsupported`| 지원하지 않는 서명 알고리즘                            | 400  | -32600 |

서명 검증은 [메시지 서명](signatures.md)에서 자세히 설명합니다.

**프로토콜 오류:**

| HTTP | 설명                                            | MCP        |
| ---- | ----------------------------------------------- | ---------- |
| 401  | 인증이 필요하거나 자격 증명이 유효하지 않음     | -32000     |
| 403  | 인증되었지만 권한이 부족함                      | -32000     |
| 409  | 멱등성 키를 다른 페이로드로 재사용함            | -32000     |
| 429  | 요청이 너무 많음                                | -32000     |
| 500  | 예기치 않은 서버 오류                           | -32603     |
| 503  | 서버가 일시적으로 요청을 처리할 수 없음         | -32000     |

HTTP 위의 MCP에서는 HTTP 상태 코드가 1차 신호이고, JSON-RPC `error.code`는 2차
신호입니다. 두 전송 방식 모두 429와 503 응답에 `Retry-After` 헤더(REST) 또는
`error.data.retry_after`(MCP)를 포함해야 합니다(**SHOULD**).

임베디드 프로토콜은 호스트와 임베디드 컨텍스트 간 피어 투 피어 통신에 같은
JSON-RPC 오류 코드를 씁니다. 서버에 특화된 상황(요청 속도 제한, 멱등성)은
임베디드 전송 방식에 적용되지 않습니다. 전체 오류 처리 명세는
[임베디드 프로토콜 — 응답 처리](embedded-protocol.md#response-handling)를
참고하세요.

##### `continue_url` 필드 { #the-continue_url-field }

UCP 협상이 실패하면 `continue_url`이 대체 웹 경험을 제공합니다. 비즈니스는 맥락상
가장 관련성 높은 URL을 제공해야 합니다(**SHOULD**).

- 체크아웃 오퍼레이션: 장바구니 또는 체크아웃 페이지로 연결
- 카탈로그 오퍼레이션: 상품 또는 검색 결과로 연결
- 대체 수단: 스토어프런트 홈페이지로 연결

이렇게 하면 우아한 성능 저하(graceful degradation)가 가능합니다. 에이전트는
구매자를 표준 웹 인터페이스로 보내 작업을 마무리하게 할 수 있습니다.

##### 전송 바인딩 { #transport-bindings }

=== "REST"

    **디스커버리 실패(424):**

    ```http
    HTTP/1.1 424 Failed Dependency
    Content-Type: application/json

    {
      "code": "profile_unreachable",
      "content": "Unable to fetch agent profile: connection timeout",
      "continue_url": "https://merchant.com/cart"
    }
    ```

    **지원하지 않는 버전(422):**

    ```http
    HTTP/1.1 422 Unprocessable Content
    Content-Type: application/json

    {
      "code": "version_unsupported",
      "content": "Protocol version 2026-01-12 is not supported. This business supports versions 2026-01-11 and 2026-01-23.",
      "continue_url": "https://merchant.com/cart"
    }
    ```

    **호환되지 않는 기능(200):**

    ```http
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "status": "error",
        "capabilities": {}
      },
      "messages": [
        {
          "type": "error",
          "code": "version_unsupported",
          "content": "UCP version 2024-01-01 is not supported",
          "severity": "unrecoverable"
        }
      ],
      "continue_url": "https://merchant.com"
    }
    ```

    **프로토콜 오류 — 요청 속도 제한(429):**

    ```http
    HTTP/1.1 429 Too Many Requests
    Retry-After: 60
    ```

    **프로토콜 오류 — 인증 실패(401):**

    ```http
    HTTP/1.1 401 Unauthorized
    WWW-Authenticate: Bearer realm="ucp"
    ```

    프로토콜 오류는 표준 HTTP 상태 코드와 헤더를 사용합니다. 응답 본문은 선택
    사항입니다.

=== "MCP"

    **디스커버리 실패(JSON-RPC error):**

    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "error": {
        "code": -32001,
        "message": "UCP discovery failed",
        "data": {
          "code": "profile_unreachable",
          "content": "Unable to fetch agent profile: connection timeout",
          "continue_url": "https://merchant.com/cart"
        }
      }
    }
    ```

    **지원하지 않는 버전(JSON-RPC error):**

    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "error": {
        "code": -32001,
        "message": "Protocol version not supported",
        "data": {
          "code": "version_unsupported",
          "content": "Protocol version 2026-01-12 is not supported. This business supports versions 2026-01-11 and 2026-01-23.",
          "continue_url": "https://merchant.com/cart"
        }
      }
    }
    ```

    **호환되지 않는 기능(JSON-RPC result):**

    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "status": "error"
          },
          "messages": [
            {
              "type": "error",
              "code": "version_unsupported",
              "content": "UCP version 2024-01-01 is not supported",
              "severity": "unrecoverable"
            }
          ],
          "continue_url": "https://merchant.com"
        },
        "content": [
          {"type": "text", "text": "{\"ucp\":{...},\"messages\":[...],\"continue_url\":\"...\"}"}
        ]
      }
    }
    ```

    **프로토콜 오류 — 요청 속도 제한(JSON-RPC error):**

    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "error": {
        "code": -32000,
        "message": "Rate limit exceeded",
        "data": {
          "retry_after": 60
        }
      }
    }
    ```

    **프로토콜 오류 — 인증 실패(JSON-RPC error):**

    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "error": {
        "code": -32000,
        "message": "Unauthorized"
      }
    }
    ```

    Streamable HTTP 전송 방식을 사용할 때 서버는 JSON-RPC 오류와 함께 그에
    대응하는 HTTP 상태 코드(예: 요청 속도 제한이면 `429`)를 반드시 반환해야
    합니다(**MUST**). 오류 유형을 나타내는 1차 신호는 HTTP 상태 코드입니다.

#### 응답 내 기능 선언 { #capability-declaration-in-responses }

응답의 `capabilities` 레지스트리는 활성화된 기능을 나타냅니다.

```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": {
      "dev.ucp.shopping.checkout": [
        {"version": "{{ ucp_version }}"}
      ],
      "dev.ucp.shopping.fulfillment": [
        {"version": "{{ ucp_version }}"}
      ]
    },
    "payment_handlers": {
      "com.example.processor_tokenizer": [
        {"id": "processor_tokenizer", "version": "{{ ucp_version }}", "available_instruments": [{"type": "card"}]}
      ]
    }
  },
  "id": "checkout_123",
  "line_items": [...]
  ... other fields
}
```

#### 응답 기능 선택 { #response-capability-selection }

비즈니스는 `ucp.capabilities`에 다음 조건을 모두 충족하는 기능만 반드시 넣어야
합니다(**MUST**).

1. 이 세션에서 협상된 교집합에 속함, 그리고(AND)
2. 이 응답의 오퍼레이션 유형과 관련 있음

**루트 기능의 관련성:**

루트 기능은 오퍼레이션 유형과 일치할 때 관련이 있습니다.

- `create_checkout` / `update_checkout` / `complete_checkout` →
    `dev.ucp.shopping.checkout`
- `create_cart` / `update_cart` → `dev.ucp.shopping.cart`
- 주문 웹훅 → `dev.ucp.shopping.order`

**확장의 관련성:**

확장은 `extends` 값 중 **하나라도** 관련 있는 루트 기능과 일치하면 관련이 있습니다.

**선택 예시:**

| 응답 유형     | 포함                            | 포함하지 않음                |
| ------------- | ------------------------------- | ---------------------------- |
| 체크아웃      | checkout, discount, fulfillment | cart, order                  |
| 장바구니      | cart, discount                  | checkout, fulfillment, order |
| 주문          | order                           | checkout, cart, discount     |

## 신원 확인과 인증 { #identity-authentication }

UCP 프로필은 두 가지 용도로 쓰입니다. 협상을 위해 당사자의 **기능**을
선언하고([프로필 구조](#profile-structure) 참고), 신원 검증을 위한 **서명 키**를
게시합니다. 그래서 문서 하나로 기능 협상과 암호학적 인증을 모두 할 수 있습니다.

비즈니스는 디스커버리 진입점으로 `/.well-known/ucp`에 프로필을 게시합니다.
플랫폼은 이를 가져와 프로토콜 지원 여부를 확인하고, 엔드포인트를 찾고, 기능을
협상합니다. 플랫폼은 요청마다 `UCP-Agent` 헤더로 자신의 프로필 URL을 알리며,
비즈니스는 이 URL로 기능을 협상하고 신원을 검증합니다. 이 설계 덕분에 **허가 없는
온보딩**(permissionless onboarding)이 가능합니다. 디스커버리 가능한 프로필이 있는
플랫폼이라면 사전 등록 없이 어떤 비즈니스와도 상호작용할 수 있습니다.

### 인증 메커니즘 { #authentication-mechanisms }

비즈니스는 사칭을 막고 메시지 무결성을 보장하기 위해 플랫폼을 인증해야
합니다(**SHOULD**). UCP는 여러 인증 메커니즘과 호환됩니다.

- **API 키** — 대역 외(out-of-band)로 교환한 사전 공유 비밀
- **OAuth 2.0** — 클라이언트 자격 증명 또는 그 밖의 OAuth 흐름
- **mTLS** — 클라이언트 인증서를 사용하는 상호 TLS
- **HTTP Message Signatures** — [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421)에
  따른 암호학적 서명(전체 명세는 [메시지 서명](signatures.md) 참고)

HTTP Message Signatures를 쓰면 허가 없는 온보딩이 가능합니다. 비즈니스는 공유
비밀을 협상하지 않고도 플랫폼이 공개한 공개 키로 플랫폼을 검증할 수 있습니다.
나머지 메커니즘은 사전에 자격 증명을 교환해야 하므로 이미 관계가 맺어져 있어야
합니다.

비즈니스에서 플랫폼으로 보내는 웹훅에는 반드시 서명해야 합니다(**MUST**).
[메시지 서명 — 서명 적용 대상](signatures.md#when-signatures-apply)을 참고하세요.

#### 신원 바인딩 { #identity-binding }

인증 메커니즘이 무엇이든 검증자는 인증된 신원이 `UCP-Agent` 헤더와 일치하는지
반드시 확인해야 합니다(**MUST**).

- **HTTP Message Signatures** — 서명자의 프로필(`UCP-Agent`에서 얻음)은 서명
    검증으로 확인되므로 추가 검사가 필요 없습니다.
- **API 키 / OAuth / mTLS** — 검증자는 인증된 주체가 `UCP-Agent`로 식별되는
    프로필을 대신해 행동할 권한이 있는지 반드시 확인해야 합니다(**MUST**). 인증된
    신원과 주장된 프로필이 충돌하는 요청은 거부합니다.

### 키 디스커버리 { #key-discovery }

양측 모두 UCP 프로필의 `signing_keys` 배열에 공개 키를 게시합니다. 플랫폼은
`/.well-known/ucp`에서 비즈니스 프로필을 가져오고, 비즈니스는 `UCP-Agent` 헤더로
플랫폼 프로필을 가져옵니다. 기능을 제공하는 바로 그 프로필이 검증 키도
제공합니다. 이것이
[RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) HTTP Message Signatures를 위한
UCP의 키 해석 메커니즘입니다.

**키 조회:**

1. 서명자의 프로필 URL을 얻습니다.
2. 프로필을 가져옵니다(또는 캐시에서 제공합니다).
3. `Signature-Input`에서 `keyid`를 추출해 `signing_keys[]`의 `kid`와
   대조합니다.
4. 해당 공개 키로 서명을 검증합니다.

키 형식(JWK), 지원 알고리즘, 키 교체 절차, 서명/검증 방식 전체는
[메시지 서명](signatures.md)에서 다룹니다.

### 프로필 요구 사항 { #profile-requirements }

#### 호스팅 { #hosting }

두 프로필 모두 안정적으로 호스팅해야 합니다. 프로필 엔드포인트가 불안정하거나
잘못 구성되면 상대방이 요청을 처리하지 못할 수 있습니다.

1. 프로필은 반드시 HTTPS로 제공해야 합니다(**MUST**).
2. 프로필 엔드포인트는 리디렉션(3xx)을 사용해서는 안 됩니다(**MUST NOT**).
3. 프로필 응답에는 반드시 `public`과 60초 이상의 `max-age`를 담은
   `Cache-Control` 헤더를 포함해야 합니다(**MUST**). 프로필을 `private`,
   `no-store`, `no-cache` 지시어와 함께 제공해서는 안 됩니다(**MUST NOT**).

프로필은 당사자의 안정적인 신원과 기능을 나타냅니다. 프로필 URL은 요청이 바뀌어도
일관되게 유지되어야 하며, 트랜잭션별 또는 세션별 설정을 담아서는 안 됩니다. 위
캐싱 정책은 최소 TTL이 있는 공유 캐시 지원을 요구해서 이를 강제합니다.

#### 가져오기 { #fetching }

비즈니스는 기능 협상과 신원 검증을 위해 플랫폼 프로필을 가져옵니다. UCP는 허가
없는 온보딩을 가능하게 하는 모범 사례를 정의하지만, 접근 정책에 대한 통제권은
전적으로 비즈니스에 있습니다. 비즈니스는 확립된 신뢰, 관찰된 행동, 운영 요구
사항에 따라 추가 규칙을 강제할 수 있습니다(**MAY**).

비즈니스는 사전 승인된 플랫폼 레지스트리를 관리해야 합니다(**SHOULD**). 사전
승인된 플랫폼이란 프로필이 검증되었고 대역 외 메커니즘(API 키, OAuth 자격 증명,
mTLS 인증서, 사전 심사)으로 신뢰가 확립된 플랫폼을 말합니다. 알려진 플랫폼은
캐시된 신원과 기능으로 효율적으로 처리할 수 있으며, 디스커버리 예산 제약을 받지
않습니다.

플랫폼을 *인식하지 못하면* 동적 프로필 디스커버리가 시작됩니다. 비즈니스는
디스커버리 사용량을 고정해 두어, 접근을 요청하는 플랫폼이 아무리 많아도 인식하지
못한 플랫폼을 해석하는 데 드는 자원이 일정하게 유지되도록 해야
합니다(**SHOULD**). 전략은 다음과 같습니다.

- **고정 크기 프로필 캐시**(예: LRU) — 마주치는 고유 프로필 URL 수와
  관계없이 메모리 사용량에 상한을 둡니다.
- 디스커버리 가져오기에 대한 **전역 속도 제한** — 오리진별 상태를 추적하지 않고도
  외부로 나가는 네트워크 사용량에 상한을 둡니다.
- **반복 실패 시 백오프** — 계속 쓸 수 없거나 악의적인 프로필 엔드포인트로의
  재시도를 줄입니다.
- **비동기 디스커버리** — `503` 상태 코드와 `Retry-After` 헤더로 응답해 프로필
  해석을 미루고, 백그라운드에서 프로필을 해석합니다. 플랫폼이 재시도할 때는 검증된
  프로필이 캐시되어 있으므로 기능 협상이 동기적으로 진행됩니다.

프로필을 가져올 때는 다음이 적용됩니다.

1. 구현체는 HTTPS로 제공되지 않는 프로필 URL을 반드시 거부해야 합니다(**MUST**).
2. 구현체는 프로필을 가져올 때 리디렉션(3xx)을 따라가서는 안 됩니다(**MUST NOT**).
3. 구현체는 프로필을 가져올 때 연결 타임아웃과 응답 타임아웃을 강제해야
   합니다(**SHOULD**).
4. 구현체는 오리진의 `Cache-Control` 헤더와 관계없이 최소 60초의 TTL 하한을 두고
   프로필을 캐시해야 합니다(**SHOULD**).
5. 구현체는 stale-while-revalidate 방식으로 프로필을 비동기 갱신할 수
   있습니다(**MAY**).
6. 알 수 없는 `kid` 때문에 서명 검증에 실패하면 구현체는 캐시된 프로필을 강제로
   갱신할 수 있습니다(**MAY**). 단, 오리진별로 TTL 하한 기간마다 두 번 이상
   그렇게 해서는 안 됩니다(**MUST NOT**).

프로필을 가져올 수 없거나(타임아웃, DNS 실패, 5xx) 검증에 실패하면(유효하지 않은
스키마, 서명 키, 서명 불일치), 비즈니스는 적절한 오류와 상태 코드로 요청을 반드시
거부해야 합니다(**MUST**)([오류 처리](#error-handling) 참고).

## 결제 아키텍처 { #payment-architecture }

UCP는 결제에 분리형 아키텍처를 채택해 **플랫폼**, **비즈니스**, **결제 자격 증명
제공자** 사이의 "N 대 N" 복잡성 문제를 해결합니다. 이 설계는 **결제 수단**(무엇을
받는가)과 **결제 핸들러**(결제 수단을 어떻게 처리하는지에 대한 명세)를 분리해
보안과 확장성을 보장합니다.

### 보안 및 신뢰 모델 { #security-and-trust-model }

결제 아키텍처는 "설계에 의한 신뢰(Trust-by-Design)" 철학을 바탕으로 합니다.
비즈니스와 결제 자격 증명 제공자는 신뢰할 수 있는 법적 관계를 맺고 있지만,
중개자인 플랫폼(클라이언트)은 원본 금융 자격 증명을 다루지 않는 것이
좋다(**SHOULD NOT**)고 가정합니다.

#### 신뢰 삼각형 { #the-trust-triangle }

1. **비즈니스 ↔ 결제 자격 증명 제공자:** 이미 맺어진 법적·기술적 관계입니다. 비즈니스는 결제 자격 증명 제공자와의 계약과 API 키를 보유합니다.
2. **플랫폼 ↔ 결제 자격 증명 제공자:** 플랫폼은 데이터를 토큰화하려고 결제 자격 증명 제공자의 인터페이스(예: iframe 또는 API)와 상호작용하지만, 자금의 "소유자"는 아닙니다.
3. **플랫폼 ↔ 비즈니스:** 플랫폼은 비즈니스가 주문을 확정할 수 있도록 결과(토큰 또는 위임)를 전달합니다.

#### 자율 커머스를 위한 강화된 보안 { #enhanced-security-for-autonomous-commerce }

사용자 승인을 암호학적으로 증명해야 하는 시나리오(예: 자율 AI 에이전트)를 위해
UCP는 **AP2 위임 확장**(`dev.ucp.shopping.ap2_mandate`)을 지원합니다. 이 선택적
확장은 검증 가능한 디지털 자격 증명으로 부인할 수 없는 승인을 제공합니다.

이 확장을 언제, 어떻게 쓰는지는
[트랜잭션 무결성](#transaction-integrity-and-non-repudiation)과
[AP2 위임 확장](ap2-mandates.md)을 참고하세요.

#### 자격 증명 흐름과 PCI 범위 { #credential-flow-pci-scope }

규정 준수(PCI-DSS) 부담을 줄이기 위해 다음을 따릅니다.

1. **단방향 흐름:** 자격 증명은 **플랫폼 → 비즈니스** 방향으로만 흐릅니다. 비즈니스는 응답에 자격 증명을 되돌려 보내서는 안 됩니다(**MUST NOT**).
2. **불투명한 자격 증명:** 플랫폼은 원본 PAN이 아니라 토큰(네트워크 토큰 등), 암호화된 페이로드, 위임을 다룹니다.
3. **핸들러 ID 라우팅:** 페이로드의 `handler_id` 덕분에 비즈니스는 복호화/청구에 어떤 결제 자격 증명 제공자 키를 써야 하는지 정확히 알 수 있고, 키 혼동 공격도 막을 수 있습니다.

### 역할과 책임: 누가 무엇을 구현하는가? { #roles-responsibilities-who-implements-what }

흔히 헷갈리는 부분이 역할 분담입니다. UCP 결제 모델은 책임을 다음과 같이 나눕니다.

| 역할                            | 책임                       | 행동                                                                                                                                                                                                                                                                |
| :------------------------------ | :------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **결제 자격 증명 제공자**       | **명세 정의**              | **핸들러 정의**를 만듭니다. 카드를 어떻게 토큰화하고 어떤 설정 입력이 필요한지 규정하는 "청사진"(JSON 스키마)을 게시합니다.<br>*예: "'com.psp-x.tokenization' 핸들러의 스키마는 이것입니다."*                                                                                |
| **비즈니스**                    | **핸들러 설정**            | 사용할 핸들러를 고르고, UCP 체크아웃 응답에 자신만의 **설정**(공개 키, 판매자 ID)을 제공합니다. *예: "이 Publishable Key와 함께 'com.psp-x.tokenization'을 사용해 Visa를 받습니다."*                                                                                     |
| **플랫폼**                      | **프로토콜 실행**          | 비즈니스의 설정을 읽고, 결제 자격 증명 제공자의 명세가 정의한 로직을 실행해 토큰을 얻습니다. *예: "이 비즈니스는 결제 자격 증명 제공자를 사용하는군요. 비즈니스의 키로 제공자의 SDK를 호출해 토큰을 받겠습니다."*                                                         |

### 체크아웃 수명 주기에서의 결제 { #payment-in-the-checkout-lifecycle }

결제가 필요할 때 결제 과정은 UCP 안에서 표준 3단계 수명 주기를 따릅니다. 바로
**협상**, **획득**, **완료**입니다.

![상위 수준 결제 흐름 시퀀스 다이어그램](site:specification/images/ucp-payment-flow.png)

1. **협상(비즈니스 → 플랫폼):** 비즈니스는 UCP 프로필에 사용 가능한 결제 핸들러를 공개합니다. 이렇게 플랫폼에 *어떻게* 결제할지 알려 줍니다(예: "이 공개 키로 이 결제 자격 증명 제공자 엔드포인트를 사용하세요").
2. **획득(플랫폼 ↔ 결제 자격 증명 제공자):** 플랫폼이 핸들러의 로직을 실행합니다. 이 과정은 클라이언트 측 또는 에이전트 측에서 결제 자격 증명 제공자와 직접 이루어집니다(예: 자격 증명을 네트워크 토큰으로 교환). 비즈니스가 관여하지 않으므로 원본 데이터가 비즈니스의 프런트엔드 API에 닿지 않습니다.
3. **완료(플랫폼 → 비즈니스):** 플랫폼은 불투명한 자격 증명(토큰)을 비즈니스에 제출합니다. 비즈니스는 결제 자격 증명 제공자와의 백엔드 연동으로 이 자격 증명을 사용해 대금을 받습니다.

### 결제 핸들러 { #payment-handlers }

결제 핸들러는 결제 수단을 어떻게 처리하는지 정의하는 (엔티티가 아닌)
**명세**입니다. 결제 핸들러는 세 참여자를 하나로 묶는 계약입니다.

**중요한 구분:**

- **결제 자격 증명 제공자** = 참여자(Google Pay, Shop Pay 같은 엔티티)
- **결제 핸들러** = 제공자가 작성하는 명세(예: `com.google.pay`, `dev.shopify.shop_pay`)

결제 핸들러를 쓰면 네트워크 토큰을 포함해 다양한 결제 수단과 토큰 유형을
지원할 수 있습니다. 결제 핸들러는 보통 결제 자격 증명 제공자나 UCP 운영 기구가
작성하는 표준화된 정의입니다.

**동적 필터링:** 비즈니스는 장바구니 맥락에 따라 `handlers` 목록을 반드시
필터링해야 합니다(**MUST**)(예: 구독 상품에는 "선구매 후결제(Buy Now Pay Later)"를
빼거나, 배송 주소에 따라 지역별 결제 방식을 거릅니다).

**사용 가능한 결제 수단 해석:** 각 활성 핸들러 안에서 플랫폼과 비즈니스는 각자
`available_instruments`, 즉 자신이 지원하는 결제 수단 유형과 제약의 집합을
따로 공개합니다. 이를 체크아웃 응답에서 권위 있는 값으로 해석할 책임은
비즈니스에 있습니다. 플랫폼의 선언(프로필에서 가져온 것)은 플랫폼이 무엇을 처리할 수
있는지 알려 줍니다. 비즈니스는 이를 자신의 `business_schema` 선언 및 장바구니 맥락과
교차시킨 뒤 해석된 결과를 반환합니다. 플랫폼은 응답의 `available_instruments`를 해당
체크아웃에 대한 권위 있는 값으로 반드시 취급해야 합니다(**MUST**). 전체 해석
의미론은 [결제 핸들러 가이드](payment-handler-guide.md#resolving-available_instruments)를
참고하세요.

### 구현 시나리오 { #implementation-scenarios }

다음 시나리오는 구체적인 데이터 예시로 다양한 결제 핸들러와 결제 수단을 어떻게
협상하고 실행하는지 보여 줍니다.

#### 시나리오 A: 디지털 지갑 { #scenario-a-digital-wallet }

이 시나리오에서 플랫폼은 결제 자격 증명 제공자(예: `com.google.pay`,
`dev.shopify.shop_pay`)를 식별하고 그 API로 암호화된 결제 토큰을 얻습니다.

##### 1. 비즈니스 측 공개(체크아웃 생성 응답) { #1-business-advertisement-response-from-create-checkout }

```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "payment_handlers": {
      "com.google.pay": [
        {
          "id": "8c9202bd-63cc-4241-8d24-d57ce69ea31c",
          "version": "{{ ucp_version }}",
          "config": {
            "api_version": 2,
            "api_version_minor": 0,
            "environment": "TEST",
            "merchant_info": {
              "merchant_name": "Example Merchant",
              "merchant_id": "01234567890123456789",
              "merchant_origin": "checkout.merchant.com"
            },
            "allowed_payment_methods": [
              {
                "type": "CARD",
                "parameters": {
                  "allowed_auth_methods": ["PAN_ONLY"],
                  "allowed_card_networks": ["VISA", "MASTERCARD"]
                },
                "tokenization_specification": {
                  "type": "PAYMENT_GATEWAY",
                  "parameters": {
                    "gateway": "example",
                    "gatewayMerchantId": "exampleGatewayMerchantId"
                  }
                }
              }
            ]
          }
        }
      ],
      "dev.shopify.shop_pay": [
        {
          "id": "shop_pay_1234",
          "version": "{{ ucp_version }}",
          "available_instruments": [
            {"type": "shop_pay"}
          ],
          "config": {
            "shop_id": "shopify-559128571",
            "environment": "production"
          }
        }
      ]
    }
  }
}
```

##### 2. 토큰 실행(플랫폼 측) { #2-token-execution-platform-side }

플랫폼은 `com.google.pay` 또는 `dev.shopify.shop_pay`를 인식합니다. 플랫폼이 `config`를
해당 핸들러 API에 전달하면 핸들러는 암호화된 토큰 데이터를 반환합니다.

##### 3. 체크아웃 완료(비즈니스로 보내는 요청) { #3-complete-checkout-request-to-business }

플랫폼은 결제 핸들러 응답을 결제 수단으로 감쌉니다.

```json
POST /checkout-sessions/{id}/complete

{
  "payment": {
    "instruments": [
      {
        "id": "pm_1234567890abc",
        "handler_id": "8c9202bd-63cc-4241-8d24-d57ce69ea31c",
        "type": "card",
        "selected": true,
        "display": {
          "brand": "visa",
          "last_digits": "4242"
        },
        "billing_address": {
          "street_address": "123 Main Street",
          "extended_address": "Suite 400",
          "address_locality": "Charleston",
          "address_region": "SC",
          "postal_code": "29401",
          "address_country": "US",
          "first_name": "Jane",
          "last_name": "Smith"
        },
        "credential": {
          "type": "PAYMENT_GATEWAY",
          "token": "{\"signature\":\"...\",\"protocolVersion\":\"ECv2\"...}"
        }
      }
    ]
  },
  "signals": {
    "dev.ucp.buyer_ip": "203.0.113.42",
    "dev.ucp.user_agent": "Mozilla/5.0 ..."
  }
}
```

#### 시나리오 B: 챌린지가 있는 직접 토큰화(SCA) { #scenario-b-direct-tokenization-with-challenge-sca }

이 시나리오에서 플랫폼은 범용 토크나이저로 세션 토큰이나 네트워크 토큰을
요청합니다. 은행이 강력한 고객 인증(SCA/3DS)을 요구하므로, 비즈니스는 완료 처리를
멈추고 챌린지를 요청해야 합니다.

##### 1. 비즈니스 측 공개 { #1-business-advertisement }

```json
{
  "ucp": {
    "payment_handlers": {
      "com.example.tokenizer": [
        {
          "id": "merchant_tokenizer",
          "version": "{{ ucp_version }}",
          "spec": "https://example.com/specs/tokenizer",
          "schema": "https://example.com/schemas/tokenizer.json",
          "available_instruments": [
            {
              "type": "card",
              "constraints": {
                "brands": ["visa", "mastercard"]
              }
            }
          ],
          "config": {
            "token_url": "https://api.psp.com/tokens",
            "public_key": "pk_123"
          }
        }
      ]
    }
  }
}
```

##### 2. 토큰 실행(플랫폼 측) { #2-token-execution-platform-side_1 }

플랫폼은 `https://api.psp.com/tokens`를 호출하고(이 엔드포인트의 주체는 플랫폼과
사전에 법적 구속력이 있는 관계를 맺고 있어야 합니다(**SHOULD**)) `tok_visa_123`을
받습니다(이 값은 보관된 카드나 네트워크 토큰을 나타낼 수 있습니다).

##### 3. 체크아웃 완료(비즈니스로 보내는 요청) { #3-complete-checkout-request-to-business_1 }

```json
POST /checkout-sessions/{id}/complete

{
  "payment": {
    "instruments": [
      {
        "handler_id": "merchant_tokenizer",
        // ... more instrument required field
        "credential": { "token": "tok_visa_123" }
      }
    ]
  },
  "signals": {
    "dev.ucp.buyer_ip": "203.0.113.42",
    "dev.ucp.user_agent": "Mozilla/5.0 ..."
  }
}
```

##### 4. 챌린지 필요(비즈니스의 응답) { #4-challenge-required-response-from-business }

비즈니스가 청구를 시도하지만, PSP가 3DS를 요구하는 "소프트 거절(Soft Decline)"을
반환합니다.

```json
HTTP/1.1 200 OK
{
  "status": "requires_escalation",
  "messages": [{
    "type": "error",
    "code": "requires_3ds",
    "content": "bank requires verification.",
    "severity": "requires_buyer_input"
  }],
  "continue_url": "https://psp.com/challenge/123"
}
```

*이제 플랫폼은 사용자가 은행 확인을 마칠 수 있도록 반드시 WebView/창에서
`continue_url`을 열어야 하며(**MUST**), 그 뒤 완료 요청을 다시 시도해야 합니다.*

#### 시나리오 C: 자율 에이전트(AP2) { #scenario-c-autonomous-agent-ap2 }

이 시나리오는 **에이전트에게 권장하는 흐름**을 보여 줍니다. 에이전트는 세션 토큰
대신 암호학적 위임을 생성합니다.

##### 1. 비즈니스 측 공개 { #1-business-advertisement_1 }

```json
{
  "ucp": {
    "payment_handlers": {
      "dev.ucp.ap2_mandate_compatible_handlers": [
        {
          "id": "ap2_234352",
          "version": "{{ ucp_version }}",
          "spec": "https://example.com/specs/ap2-handler",
          "schema": "https://example.com/schemas/ap2-handler.json",
          "available_instruments": [
            {"type": "ap2_mandate"}
          ]
        }
      ]
    }
  }
}
```

##### 2. 에이전트 실행 { #2-agent-execution }

에이전트는 에이전트가 아닌 화면(non-agentic surface)에서 사용자의 개인 키로
객체에 암호학적으로 서명합니다.

##### 3. 체크아웃 완료 { #3-complete-checkout }

```json
POST /checkout-sessions/{id}/complete

{
  "payment": {
    "instruments": [
      {
        "handler_id": "ap2_234352",
        // other required instruments fields
        "credential": {
          "type": "card",
          "token": "eyJhbGciOiJ...", // Token would contain payment_mandate, the signed proof of funds auth
        }
      }
    ]
  },
  "signals": {
    "dev.ucp.buyer_ip": "203.0.113.42",
    "com.example.risk_score": 0.95
  },
  "ap2": {
    "checkout_mandate": "eyJhbGciOiJ...", // Signed proof of checkout terms
  }
}
```

*이로써 비즈니스는 사용자가 바로 이 트랜잭션을 승인했다는 부인할 수 없는 증거를
얻고, 안심하고 자율 처리를 할 수 있습니다.*

### PCI-DSS 범위 관리 { #pci-dss-scope-management }

#### 플랫폼 범위 { #platform-scope }

대부분의 플랫폼 구현은 다음 방법으로 **PCI-DSS 범위를 피할 수 있습니다**.

- 불투명한 자격 증명(암호화된 데이터, 토큰 참조 등)을 제공하는 핸들러를 사용합니다.
- 원본 결제 데이터(카드 번호, CVV 등)에 절대 접근하거나 저장하지 않습니다.
- 자격 증명을 직접 사용할 수 없는 상태로 전달만 합니다.
- 원본 자격 증명이 플랫폼을 거치지 않는 PSP 토큰화 결제 핸들러를 사용합니다.

#### 비즈니스 범위 { #business-scope }

비즈니스는 다음 방법으로 PCI 범위를 최소화할 수 있습니다.

- 결제 자격 증명 제공자가 호스팅하는 토큰화를 사용합니다(제공자가 자격 증명을
    저장하고, 비즈니스는 토큰 참조를 받습니다).
- 암호화된 자격 증명을 제공하는 지갑 제공자(Google Pay, Shop Pay)를 사용합니다.
- 원본 자격 증명을 절대 로그에 남기지 않습니다.
- 자격 증명 처리를 PCI 인증을 받은 결제 자격 증명 제공자에게 위임합니다.

#### 결제 자격 증명 제공자 범위 { #payment-credential-provider-scope }

결제 자격 증명 제공자(PSP, 지갑)는 보통 PCI-DSS Level 1 인증을 받았으며 다음을
처리합니다.

- 원본 자격 증명 수집
- 자격 증명 보호(토큰화, 암호화, 안전한 저장)
- 자격 증명 검증과 처리
- PCI 규정을 준수하는 인프라

### 보안 모범 사례 { #security-best-practices }

**비즈니스:**

1. 처리 전에 handler_id를 검증합니다(핸들러가 공개한 집합에 속하는지 확인).
2. TEST 환경과 PRODUCTION 환경에 서로 다른 PSP 자격 증명을 사용합니다.
3. 결제 처리에 멱등성을 구현합니다(이중 청구 방지).
4. 결제 이벤트는 기록하되 자격 증명은 기록하지 않습니다.
5. 적절한 자격 증명 타임아웃을 설정합니다.
6. 암호학적 증명이 필요한 자율 커머스 시나리오라면
    `dev.ucp.shopping.ap2_mandate` 확장 지원을 고려합니다
    ([AP2 위임 확장](ap2-mandates.md) 참고).

**플랫폼:**

1. 체크아웃 API 호출에는 항상 HTTPS를 사용합니다.
2. 프로토콜을 실행하기 전에 핸들러 설정을 검증합니다.
3. 자격 증명 획득에 타임아웃 처리를 구현합니다.
4. 제출 후에는 메모리에서 자격 증명을 지웁니다.
5. 자격 증명 만료를 우아하게 처리합니다(필요하면 다시 획득).
6. 자율 에이전트라면 승인을 암호학적으로 증명하기 위해
    `dev.ucp.shopping.ap2_mandate` 확장 사용을 고려합니다
    ([AP2 위임 확장](ap2-mandates.md) 참고).

**결제 자격 증명 제공자:**

1. 특정 비즈니스용으로 자격 증명을 보호합니다(암호화, 토큰화 또는 핸들러별
    방식).
2. 자격 증명 획득에 요청 속도 제한을 구현합니다.
3. 자격 증명을 제공하기 전에 플랫폼의 권한을 검증합니다.
4. 합리적인 자격 증명 만료 시간을 설정합니다(예: 토큰은 15분, 시간 제한이 있는
    암호화 페이로드).
5. 플랫폼이 자격 증명을 직접 사용할 수 없도록 보장합니다(의도된 비즈니스만 사용
    가능).

### 사기 방지 연동 { #fraud-prevention-integration }

UCP는 [시그널](#signals)과 결제 아키텍처로 사기 방지를 지원합니다.

- 플랫폼은 카탈로그, 장바구니, 체크아웃 요청에 트랜잭션 환경
    [시그널](#signals)(IP, 사용자 에이전트)을 제공합니다.
- 비즈니스는 핸들러 설정에서 추가 필드를 요구할 수 있습니다(예: 3DS 요구 사항).
- 결제 자격 증명 제공자는 자격 증명 획득 과정에서 위험도를 평가할 수 있습니다.
- 비즈니스는 고위험 트랜잭션을 거부하고 시그널 피드백으로 추가 검증을 요청할
    수 있습니다.

### 결제 아키텍처 확장 { #payment-architecture-extensions }

앞서 설명한 핵심 결제 아키텍처는 특수한 사용 사례에 맞게 확장할 수 있습니다.

- **AP2 위임 확장**(`dev.ucp.shopping.ap2_mandate`): 부인할 수 없는 증거가
    필요한 자율 커머스 시나리오를 위해 사용자 승인의 암호학적 증명을 더합니다.
    [AP2 위임 확장](ap2-mandates.md)을 참고하세요.

- **커스텀 핸들러 유형**: 결제 자격 증명 제공자는 새로운 결제 수단을 지원하려고
    커스텀 핸들러를 정의할 수 있습니다. 자세한 내용은
    [결제 핸들러 가이드](payment-handler-guide.md)를 참고하세요.

이 확장 모델 덕분에 핵심 아키텍처는 단순하게 유지하면서도, 필요할 때 고급 보안과
규정 준수 요구 사항을 지원할 수 있습니다.

## 전송 계층 { #transport-layer }

UCP는 여러 전송 프로토콜을 지원합니다. 플랫폼과 비즈니스는 사실상 프로필의
`services`로 전송 방식을 협상합니다.

### REST 전송 방식(코어) { #rest-transport-core }

UCP는 RESTful 패턴을 따르는 **HTTP/1.1**(또는 그 이상)을 지원합니다.

- **Content-Type:** 요청과 응답은 반드시 `application/json`을 사용해야
    합니다(**MUST**).
- **메서드:** 구현체는 반드시 표준 HTTP 동사를 사용해야 합니다(**MUST**)(예: 생성에는
    `POST`, 조회에는 `GET`).
- **상태 코드:** 구현체는 반드시 표준 HTTP 상태 코드를 사용해야 합니다(**MUST**)
    (예: 200, 201, 400, 401, 500).

### Model Context Protocol(MCP) { #model-context-protocol-mcp }

UCP는 **[MCP 프로토콜](https://modelcontextprotocol.io/specification/)**(JSON-RPC
기반)을 지원합니다.

#### 요청 형식 { #request-format }

MCP 요청은 `tools/call` 메서드를 쓰며, `params.name`에 오퍼레이션 이름을,
`params.arguments`에 UCP 페이로드를 담습니다.

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "create_checkout",
    "arguments": {
      "meta": {"ucp-agent": {"profile": "https://..."}},
      "checkout": {"line_items": [...]}
    }
  },
  "id": 1
}
```

#### 응답 형식 { #response-format }

MCP 도구 응답은 하위 호환성을 위해 이중 출력 패턴을 사용합니다. UCP MCP 서버는:

- 반드시 `structuredContent`에 UCP 응답 페이로드를 반환해야 합니다(**MUST**).
- 도구 정의에 해당 기능의 UCP JSON Schema를 참조하는 `outputSchema`를
    선언해야 합니다(**SHOULD**).
- `structuredContent`를 지원하지 않는 클라이언트와의 하위 호환성을 위해
    `content[]`에 직렬화된 JSON도 반환해야 합니다(**SHOULD**).

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": {"version": "{{ ucp_version }}", "capabilities": {...}},
      "id": "checkout_abc123",
      "status": "incomplete",
      ...
    },
    "content": [
      {"type": "text", "text": "{\"ucp\":{...},\"id\":\"checkout_abc123\",...}"}
    ]
  }
}
```

### Agent-to-Agent Protocol(A2A) { #agent-to-agent-protocol-a2a }

비즈니스는 UCP를 A2A 확장(A2A Extension)으로 지원하는 A2A 에이전트를 노출할 수
있으며(**MAY**), 이렇게 하면 구조화된 UCP 데이터 타입으로 플랫폼과 연동할 수 있습니다.

### 임베디드 프로토콜(EP) { #embedded-protocol-ep }

비즈니스는 적격 호스트에 인터페이스를 임베드할 수 있습니다(**MAY**). 호스트는
사용자가 인터페이스와 상호작용할 때 이벤트를 받고, 핵심 사용자 행동을 위임받습니다.

시작은 비즈니스가 반환하는 `continue_url`로 이루어집니다.

## 표준 기능 { #standard-capabilities }

UCP는 다음 표준 기능 집합을 정의합니다.

| 기능 이름            | ID(URI)                                                               | 설명                                                                                                         |
| :------------------- | :-------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------- |
| **Cart**.            | [schemas/shopping/cart.json](site:schemas/shopping/cart.json)         | 구매 의도가 확정되기 전에 장바구니를 구성할 수 있게 합니다.                                                  |
| **Checkout**         | [schemas/shopping/checkout.json](site:schemas/shopping/checkout.json) | 장바구니 관리와 세금 계산을 포함해 체크아웃 세션의 생성과 관리를 돕습니다.                                   |
| **Identity Linking** | -                                                                     | 플랫폼이 OAuth 2.0으로 권한을 얻어 사용자를 대신해 행동할 수 있게 합니다.                                    |
| **Order**            | [schemas/shopping/order.json](site:schemas/shopping/order.json)       | 비즈니스가 주문 수명 주기(배송, 배달, 반품)의 비동기 업데이트를 푸시할 수 있게 합니다.                       |

### 정의와 확장 { #definition-extensions }

각 기능의 엔드포인트, 스키마, 유효한 확장의 상세 정의는 각 명세 파일에 있습니다.
확장은 보통 버전이 매겨지며 부모 기능과 함께 정의됩니다.

## 보안 { #security }

### 전송 보안 { #transport-security }

모든 UCP 통신은 반드시 **HTTPS**로 이루어져야 합니다(**MUST**).

### 데이터 프라이버시 { #data-privacy }

민감한 데이터(결제 자격 증명이나 개인 식별 정보(PII) 등)는 반드시 PCI-DSS와 GDPR
지침에 따라 다루어야 합니다(**MUST**). UCP는 비즈니스와 플랫폼의 책임 부담을 줄이도록
토큰화된 결제 데이터 사용을 권장합니다.

### 시그널 { #signals }

비즈니스는 승인, 요청 속도 제한, 남용 방지를 위해 환경 데이터가 필요합니다. 시그널 값은
구매자가 스스로 주장한 내용이어서는 안 됩니다(**MUST NOT**). 플랫폼은 직접 관찰한
결과(예: 연결 IP, 사용자 에이전트)를 바탕으로, 또는 독립적으로 검증 가능한 서드파티
증명을 중계하는 방식으로 시그널을 제공합니다. 서드파티 증명의 예로는 외부 검증자가
암호학적으로 서명한 결과가 있으며, 비즈니스는 이를 제공자가 게시한 키 세트로 검증할
수 있습니다.

모든 시그널 키는 출처를 보장하고, 여러 확장이 공유 네임스페이스에 값을 넣을 때 충돌을
막도록 반드시 역도메인 명명 방식을 사용해야 합니다(**MUST**). 잘 알려진 시그널은 `dev.ucp`
네임스페이스를 쓰고(예: `dev.ucp.buyer_ip`), 확장 시그널은 자체 네임스페이스를
씁니다(예: `com.example.device_id`).

```json
{
  "signals": {
    "dev.ucp.buyer_ip": "203.0.113.42",
    "dev.ucp.user_agent": "Mozilla/5.0 ...",
    "com.example.attestation": {
      "provider_jwks": "https://example.com/.well-known/jwks.json",
      "kid": "example-key-2026-01",
      "payload": { "id": "att-7c3e9f", "pass": true, "...": "..." },
      "sig": "base64url..."
    }
  }
}
```

시그널 필드에는 개인 식별 정보(PII)가 담길 수 있습니다. 플랫폼은 현재 트랜잭션과 관련된
시그널만 포함해야 합니다(**SHOULD**). 비즈니스는 트랜잭션의 운영상 필요(예: 주문 확정,
사기 검토)를 넘어 시그널 데이터를 보관하지 않는 것이 좋습니다(**SHOULD NOT**).

비즈니스는 코드가 `signal`인 메시지로 추가 데이터를 요청할 수 있습니다(**MAY**).
`path` 필드는 요청하는 시그널을 식별하고, 메시지 `type`이 강제 수준을 정합니다.
`error`는 시그널이 제공될 때까지 상태 진행을 막고, `info`는 권고일 뿐 진행을 막지
않습니다.

```json
{
  "messages": [
    {
      "type": "error",
      "code": "signal",
      "path": "$.signals['dev.ucp.buyer_ip']",
      "content": "Buyer IP is required to proceed."
    },
    {
      "type": "info",
      "code": "signal",
      "path": "$.signals['dev.ucp.user_agent']",
      "content": "Providing user agent may improve checkout outcomes."
    }
  ]
}
```

### 어트리뷰션 { #attribution }

플랫폼은 유료 광고, 자연 추천, 인플루언서 링크, AI 에이전트 등 다양한 채널로
사용자를 비즈니스에 유입시킵니다. 브라우저 기반 흐름에서는 유입 맥락(캠페인, 클릭
식별자, 소스/매체 표시)이 URL 쿼리 파라미터로 전달됩니다. `attribution` 필드를 쓰면
플랫폼이 같은 파라미터를 비즈니스에 전달할 수 있습니다.

UCP는 어트리뷰션 모델, 기간, 할당 로직을 규정하지 **않습니다**(**NOT**). 플랫폼은 기존
관례(GA4 캠페인 파라미터, `gclid` / `fbclid` / `ttclid` 같은 클릭 식별자 등)를 사용하고,
비즈니스는 자체 분석 요구에 따라 이를 받아 처리합니다.

```json
{
  "attribution": {
    "campaign_id": "18234567890",
    "campaign_source": "google",
    "campaign_medium": "cpc",
    "campaign_name": "spring_2026",
    "gclid": "EAIaIQobChMI..."
  }
}
```

어트리뷰션은 정보 제공용이며, 플랫폼이 선택적으로 제공합니다.
비즈니스는 지원 여부를 협상하거나 공개하지 않습니다. 이 필드가 있는지 없는지가 응답이나
협상에 영향을 주어서는 안 됩니다(MUST NOT).

이 데이터에는 관련 데이터 보호법상 개인 데이터로 취급되는 가명 식별자(클릭 ID, 세션
키)가 담길 수 있습니다. 플랫폼과 비즈니스는 각자의 관할권에서 규정 준수 책임을
집니다. 플랫폼은 무엇을 내보내고 고지할지 정하고, 비즈니스는 자체 데이터 처리, 보관,
동의 정책을 적용합니다. `buyer_consent` 확장은 구매자가 동의 상태를 전달할 수 있는
구조화된 채널을 제공합니다.

어트리뷰션은 장바구니, 체크아웃, 카탈로그 요청에서는 플랫폼이 제공하는 어트리뷰션
맥락으로 나타나고, 주문에서는 비즈니스가 원래 체크아웃의 어트리뷰션을 스냅숏으로 내보낸
형태로 나타납니다.

### 트랜잭션 무결성과 부인 방지 { #transaction-integrity-and-non-repudiation }

승인을 암호학적으로 증명해야 하는 시나리오(예: 자율 에이전트, 고액 트랜잭션)를 위해
UCP는 **AP2 위임 확장**(`dev.ucp.shopping.ap2_mandate`)을 지원합니다. 이 선택적
확장이 협상되면:

- 비즈니스는 체크아웃 조건에 대한 암호학적 서명을 제공합니다.
- 플랫폼은 사용자 승인을 증명하는 암호학적 위임을 제공합니다.

이 메커니즘은 트랜잭션 세부 사항과 참여자 동의에 대해 강력한 종단 간 암호학적 보증을
제공하므로 변조와 분쟁 위험을 크게 줄여 줍니다.

전체 명세, 구현 가이드, 예시는 [AP2 위임 확장](ap2-mandates.md)을
참고하세요.

## 버전 관리 { #versioning }

### 버전 형식 { #version-format }

UCP는 `YYYY-MM-DD` 형식의 날짜 기반 버전 관리를 사용합니다. 이 방식은 시간 순서가
명확하고 버전 비교가 모호하지 않습니다.

### 버전 디스커버리와 협상 { #version-discovery-and-negotiation }

UCP는 강력한 하위 호환성을 중시합니다. 어떤 버전을 구현한 비즈니스는 그 버전 또는
그보다 오래된 버전을 쓰는 플랫폼의 요청을 처리해야 합니다(**SHOULD**).

비즈니스와 플랫폼은 모두 프로필에 하나의 버전을 선언합니다.

#### 예시 { #example }

=== "비즈니스 프로필"

    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "services": { ... },
        "capabilities": { ... },
        "payment_handlers": { ... }
      }
    }
    ```

=== "플랫폼 프로필"

    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "services": { ... },
        "capabilities": { ... },
        "payment_handlers": { ... }
      }
    }
    ```

### 버전 협상 { #version-negotiation }

![상위 수준 해석 흐름 시퀀스 다이어그램](site:specification/images/ucp-discovery-negotiation.png)

버전 호환성은 **프로토콜 버전**과 **기능 버전**의 두 수준에서 작동합니다.
프로토콜 버전(`ucp.version`)은 디스커버리, 협상 흐름, 전송 바인딩, 서명 요구
사항 같은 핵심 프로토콜 메커니즘을 관장합니다. 기능 버전은
[독립적인 컴포넌트 버전 관리](#independent-component-versioning)에 정의된 대로
각 기능의 의미를 독립적으로 관장합니다.

#### 프로토콜 버전 { #protocol-version }

`version` 필드는 비즈니스의 현재 프로토콜 버전을 선언합니다. `/.well-known/ucp`의
프로필은 그 버전에서 쓸 수 있는 기능, 서비스, 결제 핸들러를 기술합니다.

이전 프로토콜 버전을 지원하는 비즈니스는 각 이전 버전을 프로필 URI에 매핑하는
`supported_versions` 객체를 선언해야 합니다(**SHOULD**). 각 URI는 해당 버전의
완전하고 자기 완결적인 프로필을 가리키며, 그 프로필은 자체 기능, 서비스, 결제
핸들러, 서명 키를 포함합니다. `supported_versions`를 생략하면 `version`만
지원합니다.

```json
{
  "ucp": {
    "version": "2026-01-23",
    "supported_versions": {
      "2026-01-11": "https://business.example.com/.well-known/ucp/2026-01-11"
    }
  }
}
```

##### 초기 서비스 및 기능 디스커버리 { #initial-service-and-capability-discovery }

플랫폼은 다음 흐름으로 비즈니스의 기능을 디스커버리합니다.

1. 플랫폼이 `/.well-known/ucp`를 가져옵니다. 이것이 현재 버전의 프로필입니다.
2. 플랫폼의 프로토콜 버전이 `version`과 같으면 이 프로필을 그대로 씁니다.
    기능 협상으로 넘어갑니다.
3. 플랫폼의 프로토콜 버전이 `supported_versions`의 키라면 매핑된 URI에서
    프로필을 가져옵니다. 이 프로필은 해당 프로토콜 버전에서 쓸 수 있는 기능을
    기술합니다. 기능 협상으로 넘어갑니다.
4. 그 밖의 경우에는 비즈니스가 플랫폼의 프로토콜 버전을 지원하지 않는 것입니다.
    플랫폼은 호환되지 않는 버전으로 요청을 보내지 않는 것이 좋으며(**SHOULD NOT**),
    비즈니스는 반드시 `version_unsupported` 오류로 응답해야 합니다(**MUST**).

버전별 프로필은 말단(leaf) 문서입니다. 정확히 하나의 프로토콜 버전만 기술하며,
`supported_versions` 필드를 포함해서는 안 됩니다(**MUST NOT**).

##### 요청 시점 검증 { #request-time-validation }

비즈니스는 모든 요청에서 플랫폼의 프로토콜 버전을 반드시 검증해야 합니다(**MUST**).

1. 플랫폼은 요청에서 참조하는 프로필의 `version` 필드로 자신이 쓰는 프로토콜
    버전을 선언합니다.
2. 비즈니스는 다음과 같이 검증합니다.
    - 플랫폼의 `version`이 비즈니스의 `version`과 같거나 `supported_versions`의
        키라면, 요청은 비즈니스 프로필의 해당 버전으로 기능 협상을 진행할 수
        있습니다(**MAY**).
    - 그렇지 않으면 비즈니스는 반드시 `version_unsupported` 오류를 반환해야
        합니다(**MUST**).
3. 요청된 오퍼레이션에 필요한 기능에 대해 기능 협상 결과 양측이 지원하는 버전이 없으면,
    비즈니스는 반드시 `capabilities_incompatible` 오류를 반환해야 합니다(**MUST**)
    ([오류 처리](#error-handling) 참고).
4. 비즈니스는 모든 응답에 협상된 프로토콜 버전을 반드시 포함해야 합니다(**MUST**).

버전 확인이 담긴 응답:

```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": { ... },
    "payment_handlers": { ... }
  },
  "id": "checkout_123",
  "status": "incomplete"
  ...other checkout fields
}
```

지원하지 않는 버전 오류 — 리소스가 생성되지 않습니다.

```json
{
  "ucp": { "version": "2026-01-11", "status": "error" },
  "messages": [{
    "type": "error",
    "code": "version_unsupported",
    "content": "Version 2026-01-12 is not supported. This business implements version 2026-01-11.",
    "severity": "unrecoverable"
  }],
  "continue_url": "https://merchant.com/"
}
```

##### 사전 릴리스 버전 { #pre-release-versions }

프로토콜 버전은 반드시 `YYYY-MM-DD` 형식의 날짜가 붙은 릴리스여야 합니다(**MUST**).
비즈니스는 프로필의 `version` 필드나 `supported_versions`에 날짜가 아닌 버전
문자열(예: `"draft"`)을 공개해서는 안 됩니다(**MUST NOT**). 사전 릴리스 구현은 안정적이지
않으므로 공개 디스커버리로 노출해서는 안 됩니다(MUST NOT). 그렇게 하면 생태계 전반이
정의되지 않은 동작과 예고 없는 비호환 변경에 노출됩니다.

플랫폼과 비즈니스는 공개 디스커버리 밖에서 사전 릴리스 구현을 조율할 수
있습니다(**MAY**). 이런 사용에는 안정성이나 호환성 보장이 전혀 없습니다. 기반 동작은 언제든
예고 없이 바뀔 수 있습니다.

#### 기능 버전 { #capability-versions }

기능 버전은 프로토콜 버전과 별개로 협상합니다. 프로필의 각 기능은 배열입니다.
같은 기능에 `version`이 서로 다른 항목을 여러 개 두면 그 기능의 여러 버전을
지원한다는 뜻입니다. 기능 교집합 알고리즘은 양측이 모두 지원하는 기능 버전만
고려합니다.

비즈니스는 응답에 협상된 프로토콜 버전과 호환되는 기능만 반드시 포함해야
합니다(**MUST**). 더 새로운 프로토콜 버전에서 도입된 기능에 의존하는 기능은 이전 프로토콜
버전으로 처리할 때 포함해서는 안 됩니다(**MUST NOT**).

### 하위 호환성 { #backwards-compatibility }

#### 하위 호환 변경 { #backwards-compatible-changes }

다음 변경은 새 버전 없이 도입할 수 있습니다(**MAY**).

- 응답에 필수가 아닌 새 필드 추가
- 요청에 필수가 아닌 새 파라미터 추가
- 전송 방식에 새 엔드포인트, 메서드, 오퍼레이션 추가
- 기존 오류 구조를 사용하는 새 오류 코드 추가
- enum에 새 값 추가(전체 목록이라고 명시적으로 문서화된 경우 제외)
- 응답의 필드 순서 변경
- 불투명 문자열(ID, 토큰)의 길이나 형식 변경

#### 호환성을 깨는 변경 { #breaking-changes }

다음 변경은 새 버전 없이 도입해서는 안 됩니다(**MUST NOT**).

- 기존 필드 제거 또는 이름 변경
- 필드 타입이나 의미 변경
- 필수가 아닌 필드를 필수로 변경
- 오퍼레이션, 메서드, 엔드포인트 제거
- 인증 또는 권한 부여 요구 사항 변경
- 기존 프로토콜 흐름이나 상태 머신 수정
- 기존 오류 코드의 의미 변경

### 독립적인 컴포넌트 버전 관리 { #independent-component-versioning }

- UCP 프로토콜은 기능과 별개로 버전이 매겨집니다.
- 각 기능은 다른 기능과 별개로 버전이 매겨집니다.
- 기능은 반드시 프로토콜과 같은 하위 호환성 규칙을 따라야 합니다(**MUST**).
- 비즈니스는 위에서 설명한 것과 같은 로직으로 기능 버전 호환성을 반드시
    검증해야 합니다(**MUST**).
- 전송 방식은 자체 버전 처리 메커니즘을 정의할 수 있습니다(**MAY**).

#### UCP 기능(`dev.ucp.*`) { #ucp-capabilities-devucp }

UCP가 작성한 기능은 기본적으로 프로토콜 릴리스와 함께 버전이 매겨집니다. 프로토콜
릴리스 주기와 별개로 호환성을 깨는 변경이 필요하면 개별 기능은 독립적으로 버전을
매길 수 있습니다(**MAY**).

#### 벤더 기능(`com.{vendor}.*`) { #vendor-capabilities-comvendor }

`dev.ucp.*` 네임스페이스 밖의 기능은 완전히 독립적으로 버전이 매겨집니다.
릴리스 일정과 버전 관리 전략은 벤더가 직접 정합니다.

## 용어집 { #glossary }

UCP 명세 전반에서 쓰는 약어와 용어의 정의는 [용어집](glossary.md)을 참고하세요.
