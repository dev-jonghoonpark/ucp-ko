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

# 결제 핸들러 명세 작성 가이드 { #payment-handler-specification-guide }

## 소개 { #introduction }

이 가이드는 UCP 결제 핸들러를 명세할 때 쓰는 표준 구조와 용어를 정의합니다. 구현자가
일관되고 완전하며 명확한 명세를 받아 볼 수 있도록, 모든 결제 핸들러 명세는 이 구조를 따라야
합니다(**SHOULD**).

### 목적 { #purpose }

결제 핸들러는 플랫폼, 비즈니스, 결제 제공자 사이의 "N 대 N" 상호 운용성을 가능하게 합니다.
잘 작성된 핸들러 명세는 각 참여자에 대해 다음 질문에 답해야 합니다.

- **누가 참여하는가?** 어떤 참여자가 관여하며, 각자의 역할은 무엇인가?
- **사전 요건은 무엇인가?** 어떤 온보딩이나 설정이 필요한가?
- **어떻게 구성되는가?** 어떤 구성을 공개하거나 사용하는가?
- **어떻게 실행되는가?** 결제 수단을 획득하거나 처리하기 위해 어떤 프로토콜을 따르는가?

이 가이드는 모든 핸들러 명세가 이 질문들에 체계적으로 답할 수 있도록 프레임워크를
제공합니다.

### 적용 범위 { #scope }

이 가이드의 적용 대상은 다음과 같습니다.

- **핸들러**(예: `com.google.pay`, `dev.shopify.shop_pay`) — 특정 결제 방법의
  구현

---

## 핵심 개념 { #core-concepts }

모든 결제 핸들러 명세는 아래 핵심 요소를 반드시 정의해야 합니다(**MUST**).

**프로토콜 시그니처에 관한 참고:**: 이 섹션의 함수 시그니처(예:
`PROCESSING(...)`)는 실제 함수 호출이 아니라 **논리적 데이터 흐름**을 나타냅니다. 명세
작성자는 이 논리적 흐름을 자신의 구현에서 쓰는 실제 전송 프로토콜에 대응시켜야 합니다.

```text
+------------------------------------------------------------------------------+
|                        Payment Handler Framework                             |
+------------------------------------------------------------------------------+
|                                                                              |
|   +--------------+                                                           |
|   | PARTICIPANTS |  Who participates in this handler?                        |
|   +------+-------+                                                           |
|          |                                                                   |
|          v                                                                   |
|   +--------------+                                                           |
|   |PREREQUISITES |  How does each participant obtain identity & configs?     |
|   +------+-------+                                                           |
|          |                                                                   |
|          +--------------------+----------------------+                       |
|          v                    v                      v                       |
|   +--------------+    +--------------+      +--------------+                 |
|   |   HANDLER    |    |  INSTRUMENT  |      |  PROCESSING  |                 |
|   | DECLARATION  |    |  ACQUISITION |      |              |                 |
|   +--------------+    +--------------+      +--------------+                 |
|   Business advertises  platform acquires     Participant                     |
|   handler config       checkout instrument   processes instrument            |
|                                                                              |
+------------------------------------------------------------------------------+
```

### 참여자 { #participants }

**정의:** 결제 핸들러의 수명 주기에 참여하는 개별 행위자. 모든 핸들러에는 최소 두
참여자(비즈니스와 플랫폼)가 있으며, 특정 역할을 맡는 참여자를 추가로 정의할 수
있습니다(**MAY**).

**용어에 관한 참고:**: 이 가이드에서는 해당 참여자를 "**비즈니스(Business)**"라고 부르지만,
기술적인 스키마 필드에는 업계 표준 명칭인 **`merchant_*`**(예: `merchant_id`,
`merchant_name`)가 그대로 남아 있을 수 있습니다. 명세에는 이런 필드 대응 관계를 반드시
명시적으로 문서화해야 합니다(**MUST**).

**표준 참여자:**

| 참여자  | 역할                                                               |
| :----------- | :----------------------------------------------------------------- |
| **비즈니스** | 핸들러 구성 공개, 결제 수단 처리    |
| **플랫폼** | 핸들러 탐색, 결제 수단 획득, 체크아웃 제출 |

**확장 참여자**(핸들러별 참여자 예시):

| 참여자   | 역할 예시                                                           |
| :------------ | :--------------------------------------------------------------------- |
| **토크나이저(Tokenizer)** | 원본 자격 증명 저장, 토큰 자격 증명 발급                    |
| **PSP**       | 체크아웃 결제 수단으로 비즈니스를 대신해 결제 처리 |

### 사전 요건 { #prerequisites }

**정의:** 참여자가 핸들러 흐름에 참여하기 전에 완료해야 하는 온보딩, 설정 또는 구성.

**시그니처:**

```text
PREREQUISITES(participant, onboarding_input) → prerequisites_output
```

| 필드                  | 설명                                                |
| :--------------------- | :--------------------------------------------------------- |
| `participant`          | 온보딩 대상 참여자(비즈니스, 플랫폼 등) |
| `onboarding_input`     | 참여자가 설정 과정에서 제공하는 정보                 |
| `prerequisites_output` | 받은 신원 정보와 추가 구성     |

**사전 요건 출력:**

`prerequisites_output`에는 참여자가 온보딩에서 받는 내용이 담깁니다. 최소한
**신원**(identity)은 포함됩니다([결제 신원](site:schemas/common/types/payment_identity.json) 참고).
핸들러별 추가 구성, 자격 증명, 설정도 포함할 수 있습니다(**MAY**).

결제 핸들러 명세에서 `prerequisites_output`의 정식 스키마를 정의할 **필요는 없습니다**. 대신
명세에 다음 내용을 명확히 문서화해야 합니다(**SHOULD**).

- 어떤 신원이 할당되는지(그리고 `PaymentIdentity`에 어떻게 대응하는지)
- 어떤 추가 구성이 제공되는지
- 사전 요건 출력이 핸들러 선언, 결제 수단 획득, 처리 단계에서 어떻게 쓰이는지

**참고:**

- 사전 요건은 보통 대역 외(out-of-band)에서 처리합니다(포털, 계약, API 호출).
- 참여자마다 독립적인 사전 요건이 있을 수 있습니다(**MAY**).
- 사전 요건에서 받은 신원은 보통 핸들러의 `config` 객체 안에 들어갑니다(예:
  `merchant_id` 또는 이와 비슷한 핸들러별 필드).
- 원본 자격 증명을 받는 참여자(예: 비즈니스, PSP)는 보통 온보딩 중에 보안 확인 절차를 마쳐, 자격 증명 처리와 규정 준수에 대한 책임을 수락해야 합니다.

### 핸들러 선언 { #handler-declaration }

**정의:** 비즈니스가 이 핸들러를 지원한다는 사실을 나타내며, 플랫폼이 핸들러를 호출할 수 있도록
공개하는 구성.

**시그니처:**

```text
HANDLER_DECLARATION(prerequisites_output) → handler_declaration
```

| 필드                  | 설명                                                    |
| :--------------------- | :------------------------------------------------------------- |
| `prerequisites_output` | 비즈니스 사전 요건에서 얻은 신원과 구성     |
| `handler_declaration`  | `ucp.payment_handlers`에 공개되는 핸들러 객체        |

**출력 구조:**

핸들러 선언은 [`PaymentHandler`](site:schemas/payment_handler.json) 스키마를 따릅니다.
명세는 사용 가능한 구성 스키마와 결제 수단 스키마를 정의하고, 비즈니스의 사전 요건 출력과
원하는 구성을 바탕으로 각각을 어떻게 구성하는지 정의해야 합니다(**SHOULD**).

<!-- ucp:example schema=profile def=business_schema target=$.ucp.payment_handlers -->
```json
{
  "com.example.handler": [
    {
      "id": "processor_tokenizer_1234",
      "version": "{{ ucp_version }}",
      "spec": "https://example.com/ucp/handler",
      "schema": "https://example.com/ucp/handler/schema.json",
      "available_instruments": [ ... ],
      "config": {
        // Handler-specific configuration (see Config Shapes)
      }
    }
  ]
}
```

**`available_instruments`** 필드는 선택 사항입니다. 이 필드가 없으면 핸들러는 결제 수단 유형이나 제약
조건을 전혀 제한하지 않으며, 핸들러 스키마가 정의하는 모든 결제 수단 유형을
지원합니다. 이 필드가 있으면 공개하는 유형을 좁히거나 추가 제약 조건을 적용합니다(예: 카드 브랜드를
`visa`와 `mastercard`로 제한). 비즈니스 프로필과 권위 있는 응답에서 배열 순서는 선호하는
결제 수단 표시 순서이며, 앞에 있을수록 우선합니다.

각 항목은 두 축으로 범위를 좁힙니다. 둘 다
[제약 표현식(Constraint Expression)](../overview/index.md#constraint-expression)이므로
평가기 하나로 둘 다 처리할 수 있으며, 차이는 설명하는 객체뿐입니다.

```text
available_instruments[]
├── type                       selects an instrument schema
├── constraints                describes that schema's `constraint_target`
└── ucp.request_constraints    describes the request that submits it, via `path`
```

`constraints`에는 `path`를 지정할 수 없습니다. 대상 객체는 선택하는 것이 아니라 고정되어
있기 때문입니다. 항목의 `type`에 해당하는 결제 수단 스키마가 이 객체를 `$defs/constraint_target`으로
선언하며, `properties`의 키는 그 객체의 멤버를 가리킵니다. 대상에는 비즈니스가 받는 값이
아니라 도출하는 값이 담깁니다. 카드라면 계좌 번호에서 읽어 내고 토큰화 후에도 유지되는
`brand`가 그런 값입니다. 그래서 이런 값은 요청 데이터에 대한 요구 사항으로 표현할 수 없습니다.

`ucp.request_constraints`는 실제로 전송되는 *모든 것*을 다룹니다. 어떤 결제 수단 필드가
필수인지, 어떤 자격 증명 유형을 받는지, 자격 증명 유형마다 어떤 필드를 담아야 하는지를
기술합니다. 모델링된 데이터에는 핸들러별 불리언 대신 필드 요구 사항을 사용합니다.

권위 있는 응답에서 사용 가능한 결제 수단에 `ucp.request_constraints`가 포함되어 있으면,
사용 가능한 결제 수단의 응답 정규화 경로(Normalized Path)로는 제출된 결제 수단을 식별할 수
없으므로 비즈니스는 반드시 명시적인 `path`를 포함해야 합니다(**MUST**). 이 경로는 해당
항목을 담은 핸들러의 `id`를 제출된 `handler_id`에, 사용 가능한 결제 수단의 `type`을
제출된 `type`에 매칭하며, 다음 요청에 일치하는 결제 수단이 있을 때 적용됩니다. 쿼리에
더 강한 연관이 필요하면 결제 핸들러 명세와 결제 수단 명세에서 정의합니다.
[요청 제약 조건](../overview/index.md#request-constraints)을 참고하세요.

경로 하나가 제출된 결제 수단을 선택하고, 제약 표현식 하나가 그 결제 수단을 설명합니다. 해당
유형의 모든 결제 수단에 공통인 요구 사항은 형제 멤버로 두고, 자격 증명 계열마다 다른 요구
사항은 중첩된 `credential` 객체의 `anyOf` 분기로 둡니다. 자격 증명 계열을 별도 스키마로
나누므로 분기마다 안정적인 판별자가 생깁니다.

<!-- ucp:example schema=payment_handler def=business_schema -->
```json
{
  "id": "processor_tokenizer_1234",
  "version": "{{ ucp_version }}",
  "available_instruments": [
    {
      "type": "card",
      "constraints": { "properties": { "brand": { "enum": ["visa", "mastercard"] } } },
      "ucp": {
        "request_constraints": {
          "path": "$['payment']['instruments'][?@['handler_id'] == 'processor_tokenizer_1234' && @['type'] == 'card']",
          "required": ["billing_address", "credential"],
          "properties": {
            "billing_address": { "required": ["postal_code", "address_country"] },
            "credential": {
              "anyOf": [
                { "properties": { "type": { "const": "pan" } }, "required": ["cvc"] },
                { "properties": { "type": { "const": "network_token" } } }
              ]
            }
          }
        }
      }
    }
  ]
}
```

형제 멤버는 이 핸들러의 모든 카드 결제 수단에 적용됩니다. 분기는 `cvc`를 담은 PAN 자격
증명이나 네트워크 토큰 중 하나를 받습니다. 네트워크 토큰은 `network_token_credential.json`에서
이미 `cryptogram`을 담도록 요구합니다. 그 밖의 유형의 자격 증명은 어느 분기도 충족하지
못합니다. 선언된 제약 조건은 미리 제시하는 최소 요건이며, 동적 요구 사항에는 여전히 복구
가능한 오류와 [`message_error.path`](site:schemas/common/types/message_error.json)를
사용합니다.

---

#### 핸들러 선언 변형 { #handler-declaration-variants }

`PaymentHandler` 스키마는 맥락에 따라 세 가지 변형을 정의합니다. 기술적으로 필수인 것은
`id`와 `version`뿐이지만, 변형마다 목적이 다르고 보통 담기는 구성도
다릅니다.

| 변형 | 맥락 | 목적 |
| :------ | :------ | :------ |
| **business_schema** | 비즈니스 디스커버리(`/.well-known/ucp`) | 비즈니스 신원과, 비즈니스가 이 핸들러를 어떻게 구성했는지 선언합니다. 판매자별 설정을 담습니다. |
| **platform_schema** | 플랫폼 프로필(공개된 URI) | 플랫폼 신원과, 플랫폼이 이 핸들러를 어떻게 지원하는지 선언합니다. 구현자를 위한 `spec`, `schema` URL을 포함합니다. |
| **response_schema** | 체크아웃/주문 API 응답 | 확정된 맥락을 담은 **런타임 구성**: 판매자 신원, 이 체크아웃에 확정된 `available_instruments`, 토큰화 명세, 거래 처리에 필요한 기타 상태. 플랫폼은 반드시 이 구성을 권위 있는 정보로 취급해야 합니다(**MUST**). |

**비즈니스 스키마 예시**(비즈니스가 핸들러 구성을 선언):

<!-- ucp:example schema=payment_handler def=business_schema -->
```json
{
  "id": "processor_tokenizer_1234",
  "version": "{{ ucp_version }}",
  "spec": "https://example.com/ucp/handler",
  "schema": "https://example.com/ucp/handler/schema.json",
  "available_instruments": [
    {
      "type": "card",
      "constraints": {
        "properties": { "brand": { "enum": ["visa", "mastercard"] } }
      }
    }
  ],
  "config": {
    "environment": "production",
    "business_id": "business_xyz_789"
  }
}
```

**플랫폼 스키마 예시**(플랫폼이 핸들러 지원을 선언):

<!-- ucp:example schema=payment_handler def=platform_schema -->
```json
{
  "id": "platform_tokenizer_2345", // note: ids are for disambiguation, they may differ between business and platform
  "version": "{{ ucp_version }}",
  "spec": "https://example.com/ucp/handler",
  "schema": "https://example.com/ucp/handler/schema.json",
  "available_instruments": [
    {
      "type": "card",
      "constraints": {
        "properties": { "brand": { "enum": ["visa", "mastercard", "amex", "discover"] } }
      }
    }
  ],
  "config": {
    "environment": "production",
    "platform_id": "platform_abc_123"
  }
}
```

**응답 스키마 예시**(체크아웃의 런타임 맥락):

<!-- ucp:example schema=payment_handler def=response_schema -->
```json
{
  "id": "processor_tokenizer_1234",
  "version": "{{ ucp_version }}",
  "available_instruments": [
    {
      "type": "card",
      "constraints": {
        "properties": { "brand": { "enum": ["visa", "mastercard"] } }
      }
    }
  ],
  "config": {
    "api_version": 2,
    "environment": "production",
    "business_id": "business_xyz_789"
  }
}
```

#### `available_instruments` 확정 { #resolving-available_instruments }

플랫폼과 비즈니스는 각자의 프로필에서 따로 `available_instruments`를 공개합니다.
이를 `response_schema`로 반환할 권위 있는 값으로 확정하는 일은 비즈니스가 맡습니다.

**확정 흐름:**

1. **플랫폼이 지원 범위를 선언합니다** — 플랫폼 프로필의 각 핸들러 선언에
   `available_instruments`가 들어 있습니다. 비즈니스는 이 값으로 플랫폼이 무엇을 처리할 수
   있는지 알 수 있습니다(예: `["visa", "mastercard", "amex", "discover"]`만
   지원).

2. **비즈니스가 확정합니다** — 요청을 받으면 비즈니스는 다음 항목의 교집합을 구해 해당
   체크아웃에 확정된 `available_instruments`를 계산합니다.
   - 플랫폼이 공개한 `available_instruments`(플랫폼의 지원 범위)
   - 자신의 `business_schema` 선언(판매자가 실제로 받도록 설정된 결제 수단)
   - 장바구니/체크아웃 맥락(예: 특정 상품 유형은 사용할 수 있는 결제 방법을 제한할 수 있음)

3. **응답이 우선합니다** — `response_schema`의 `available_instruments`는 이 체크아웃에
   대해 비즈니스가 확정한 선택입니다. 플랫폼은 반드시 이 값을 권위 있는
   정보로 취급해야 하며(**MUST**), 이와 모순되는 결제 수단 유형을 사용하거나 제약
   조건을 적용하려 해서는 안 됩니다(**MUST NOT**).

**예시:**

| 출처 | `available_instruments` |
| :----- | :---------------------- |
| 플랫폼 프로필 | `[{type: "card", constraints: {properties: {brand: {enum: ["visa", "mastercard", "amex", "discover"]}}}}]` |
| 비즈니스 프로필 | `[{type: "card", constraints: {properties: {brand: {enum: ["visa", "mastercard", "amex"]}}}}]` |
| **응답(확정)** | `[{type: "card", constraints: {properties: {brand: {enum: ["visa", "mastercard", "amex"]}}}}]` |

이 예시에서는 비즈니스의 PSP에 Discover 지원이 설정되어 있지 않으므로, 플랫폼이
Discover를 지원하더라도 응답에서 빠집니다.

---

#### 스키마 정의 { #defining-the-schema }

`schema` 필드는 핸들러별 형태를 정의하는 JSON 스키마를 가리킵니다. 작성자는 보통 형태마다
별도 파일을 만들어 참조합니다.

- **구성(Config)** — 플랫폼/비즈니스 선언과 런타임 응답을 위한 구성
- **결제 수단(Instrument)** — 플랫폼에 반환되는 결제 수단 구조
- **자격 증명(Credential)** — 결제 수단 안의 자격 증명 구조

**핸들러 스키마 예시:**

<!-- ucp:example skip reason="handler schema definition" -->
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/ucp/handlers/tokenizer/schema.json",
  "title": "Tokenizer Handler Schema",
  "description": "Schema for the com.example.tokenizer payment handler.",
  "name": "com.example.tokenizer",
  "version": "{{ ucp_version }}",

  "$defs": {
    "tokenizer_token": { "$ref": "types/tokenizer_token.json" },
    "tokenizer_alt_token": { "$ref": "types/tokenizer_alt_token.json" },

    "tokenizer_instrument": { "$ref": "types/tokenizer_instrument.json" },
    "tokenizer_alt_instrument": { "$ref": "types/tokenizer_alt_instrument.json" },

    "com.example.tokenizer": {
      "payment_instrument": {
        "title": "Tokenizer Payment Instrument",
        "description": "Any instrument type supported by this handler.",
        "oneOf": [
          { "$ref": "#/$defs/tokenizer_instrument" },
          { "$ref": "#/$defs/tokenizer_alt_instrument" }
        ]
      },
      "platform_schema": {
        "title": "Tokenizer (Platform)",
        "description": "Platform-level handler configuration for discovery.",
        "allOf": [
          { "$ref": "https://ucp.dev/{{ ucp_version }}/schemas/payment_handler.json#/$defs/platform_schema" },
          {
            "properties": {
              "config": {
                "$ref": "types/platform_config.json",
                "description": "Platform configuration for this handler."
              }
            }
          }
        ]
      },
      "business_schema": {
        "title": "Tokenizer (Business)",
        "description": "Business-level handler configuration for discovery.",
        "allOf": [
          { "$ref": "https://ucp.dev/{{ ucp_version }}/schemas/payment_handler.json#/$defs/business_schema" },
          {
            "properties": {
              "config": {
                "$ref": "types/business_config.json",
                "description": "Business configuration for this handler."
              }
            }
          }
        ]
      },
      "response_schema": {
        "title": "Tokenizer (Response)",
        "description": "Runtime handler configuration in checkout responses.",
        "allOf": [
          { "$ref": "https://ucp.dev/{{ ucp_version }}/schemas/payment_handler.json#/$defs/response_schema" },
          {
            "properties": {
              "config": {
                "$ref": "types/response_config.json",
                "description": "Runtime configuration for this handler."
              }
            }
          }
        ]
      }
    }
  }
}
```

---

#### 구성 형태 { #config-shapes }

변형마다 맥락에 맞춘 고유한 구성 스키마가 있습니다.

| 변형 | 구성 파일 | 목적 |
| :------ | :---------- | :------ |
| **business_schema** | `types/business_config.json` | 비즈니스 신원과 판매자별 설정 |
| **platform_schema** | `types/platform_config.json` | 플랫폼 신원과 플랫폼 수준 설정 |
| **response_schema** | `types/response_config.json` | 전체 런타임 상태: 신원, 토큰화 명세 |

**`types/business_config.json` 예시:**

<!-- ucp:example skip reason="handler schema definition" -->
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/ucp/handlers/tokenizer/types/business_config.json",
  "title": "Tokenizer Business Config",
  "type": "object",
  "properties": {
    "environment": {
      "type": "string",
      "enum": ["sandbox", "production"],
      "default": "production"
    },
    "business_id": {
      "type": "string",
      "description": "Business identifier for this handler."
    }
  }
}
```

**`types/platform_config.json` 예시:**

<!-- ucp:example skip reason="handler schema definition" -->
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/ucp/handlers/tokenizer/types/platform_config.json",
  "title": "Tokenizer Platform Config",
  "type": "object",
  "properties": {
    "environment": {
      "type": "string",
      "enum": ["sandbox", "production"],
      "default": "production"
    },
    "platform_id": {
      "type": "string",
      "description": "Platform identifier for this handler."
    }
  }
}
```

**`types/response_config.json` 예시:**

<!-- ucp:example skip reason="handler schema definition" -->
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/ucp/handlers/tokenizer/types/response_config.json",
  "title": "Tokenizer Response Config",
  "type": "object",
  "properties": {
    "api_version": { "type": "integer" },
    "environment": {
      "type": "string",
      "enum": ["sandbox", "production"]
    },
    "business_id": {
      "type": "string",
      "description": "Business identifier for this handler."
    },
    "tokenization_specification": {
      "type": "object",
      "description": "Handler-specific tokenization settings.",
      "properties": {
        "type": { "type": "string" },
        "parameters": { "type": "object" }
      }
    }
  }
}
```

---

#### 결제 수단 형태 { #instrument-shapes }

**기본 결제 수단 스키마:**

| 스키마                                                                                     | 설명                                                      |
| :----------------------------------------------------------------------------------------- | :--------------------------------------------------------------- |
| [`payment_instrument.json`](site:schemas/common/types/payment_instrument.json)             | 기본: id, handler_id, type, billing_address, credential, display |
| [`card_payment_instrument.json`](site:schemas/common/types/card_payment_instrument.json)   | 기본 스키마에 표시 정보 추가: brand, last_digits, expiry, 카드 이미지  |

UCP는 `card` 같은 범용 결제 수단용 기본 스키마를 제공합니다. 명세 작성자는 기본 결제
수단을 확장해 핸들러별 표시 데이터를 추가하거나 자격 증명 참조를 원하는 대로 바꿀 수
있습니다(**MAY**). 핸들러는 결제 흐름별로 여러 결제 수단 유형을 정의할 수
있습니다(**MAY**).

**제약 대상(Constraint Target):**

결제 수단 스키마는 가용성 제약 조건이 가리킬 수 있는 대상을 `$defs/constraint_target`에
선언합니다. 이 대상은 멤버와 그 타입으로 이루어진 일반 객체이며, 페이로드에 실려 전송되지
않습니다. 핸들러의 `payment_instrument` 집합은 각 `type`을 그 대상을 소유한 스키마에
연결합니다.
[`card_payment_instrument.json`](site:schemas/common/types/card_payment_instrument.json)은
`brand`를 문자열로 선언하므로
`{ "properties": { "brand": { "enum": ["visa", "mastercard"] } } }`는 선언된 멤버를
가리키며 그 값을 문자열 값으로 고정합니다. 문법은 기본 결제 핸들러가 검증하므로 형식이
잘못된 `constraints`는 어디서나 실패합니다. 의미는 대상이 정하므로, 선언되지 않은 멤버를
가리키거나 잘못된 타입으로 고정하면 기본 스키마가 `type`에 따라 분기하지 않아도 이를
보고할 수 있습니다. 결제 수단을 확장하는 핸들러는 해당 결제 수단의 대상을 다시 기술하지 말고
확장해야 합니다(**SHOULD**).

**`types/tokenizer_instrument.json` 예시**:

<!-- ucp:example skip reason="handler schema definition" -->
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/ucp/handlers/tokenizer/types/tokenizer_instrument.json",
  "title": "Tokenizer Card Instrument",
  "description": "Card-based payment instrument for com.example.tokenizer.",

  "$defs": {
    "constraint_target": {
      "title": "Tokenizer Card Constraint Target",
      "description": "Extends the card target with tokenizer-specific members.",
      "allOf": [
        { "$ref": "https://ucp.dev/{{ ucp_version }}/schemas/common/types/card_payment_instrument.json#/$defs/constraint_target" },
        {
          "type": "object",
          "properties": {
            "tokenization_type": {
              "type": "string",
              "description": "How the credential was tokenized (e.g., 'network_token', 'merchant_token')."
            }
          }
        }
      ]
    }
  },

  "allOf": [
    { "$ref": "https://ucp.dev/{{ ucp_version }}/schemas/common/types/card_payment_instrument.json" }
  ],
  "type": "object",
  "required": ["type"],
  "properties": {
    "type": { "const": "tokenizer_card" },
    "credential": {
      "oneOf": [
        { "$ref": "tokenizer_token.json" },
        { "$ref": "tokenizer_alt_token.json" }
      ]
    },
    "special_tokenizer_context": {
      "type": "object",
      "description": "Handler-specific context for tokenizer instruments."
    }
  }
}
```

**`types/tokenizer_alt_instrument.json` 예시:**

<!-- ucp:example skip reason="handler schema definition" -->
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/ucp/handlers/tokenizer/types/tokenizer_alt_instrument.json",
  "title": "Tokenizer Alt Instrument",
  "description": "Alternative payment instrument for com.example.tokenizer.",
  "allOf": [
    { "$ref": "https://ucp.dev/{{ ucp_version }}/schemas/common/types/payment_instrument.json" }
  ],
  "type": "object",
  "required": ["type"],
  "properties": {
    "type": { "const": "tokenizer_alt" },
    "credential": {
      "oneOf": [
        { "$ref": "tokenizer_token.json" },
        { "$ref": "tokenizer_alt_token.json" }
      ]
    },
    "special_tokenizer_context": {
      "type": "object",
      "description": "Handler-specific context for tokenizer instruments."
    }
  }
}
```

---

#### 자격 증명 형태 { #credential-shapes }

**기본 자격 증명 스키마:**

| 스키마                                                                                         | 설명                                 |
| :--------------------------------------------------------------------------------------------- | :------------------------------------------ |
| [`payment_credential.json`](site:schemas/common/types/payment_credential.json)                 | 기본: 유형 판별자만 포함               |
| [`token_credential.json`](site:schemas/common/types/token_credential.json)                     | 토큰: 유형 + 토큰 문자열                  |
| [`pan_credential.json`](site:schemas/common/types/pan_credential.json)                         | 원본 FPAN, `cvc`로 검증. 원천(source) 전용. |
| [`network_token_credential.json`](site:schemas/common/types/network_token_credential.json)     | 네트워크 토큰, `cryptogram`으로 검증   |

UCP는 범용 결제 자격 증명용 기본 스키마를 제공합니다. 작성자는 이 스키마를 확장해
핸들러별 자격 증명 맥락을 포함할 수 있습니다(**MAY**). 핸들러는 결제 수단 흐름별로
여러 자격 증명 유형을 정의할 수 있습니다(**MAY**).

명세는 핸들러가 받는 자격 증명 유형을 반드시 정의해야 합니다(**MUST**).

**중요:** 토큰 자격 증명을 사용한다면 플랫폼이 자격 증명을 언제 갱신해야 하는지 알 수
있도록 스키마에 만료 필드(`expiry`, `ttl` 또는 비슷한 필드)를 반드시 포함해야
합니다(**MUST**).

**`types/tokenizer_token.json` 예시**(만료되는 토큰):

<!-- ucp:example skip reason="handler schema definition" -->
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/ucp/handlers/tokenizer/types/tokenizer_token.json",
  "title": "Tokenizer Card Token",
  "description": "Card token credential for com.example.tokenizer.",
  "allOf": [
    { "$ref": "https://ucp.dev/{{ ucp_version }}/schemas/common/types/token_credential.json" }
  ],
  "type": "object",
  "required": ["type", "token", "expiry"],
  "properties": {
    "type": {
      "const": "tokenizer_card_token",
      "description": "Credential type discriminator."
    },
    "expiry": {
      "type": "string",
      "format": "date-time",
      "description": "Token expiration. Platforms must refresh before this time."
    }
  }
}
```

**`types/tokenizer_alt_token.json` 예시**(대체 토큰):

<!-- ucp:example skip reason="handler schema definition" -->
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/ucp/handlers/tokenizer/types/tokenizer_alt_token.json",
  "title": "Tokenizer Alt Token",
  "description": "Alt token credential for com.example.tokenizer, adding routing hints",
  "allOf": [
    { "$ref": "https://ucp.dev/{{ ucp_version }}/schemas/common/types/token_credential.json" }
  ],
  "type": "object",
  "required": ["type", "token", "expiry"],
  "properties": {
    "type": {
      "const": "tokenizer_alt_token",
      "description": "Credential type discriminator."
    },
    "expiry": {
      "type": "string",
      "format": "date-time",
      "description": "Token expiration. Platforms must refresh before this time."
    },
    "routing_hint": {
      "type": "string",
      "description": "Optional routing number hint."
    }
  }
}
```

### 결제 수단 획득 { #instrument-acquisition }

**정의:** 비즈니스의 체크아웃에 제출할 결제 수단을 얻기 위해 플랫폼이 따르는
프로토콜.

**시그니처:**

```text
INSTRUMENT_ACQUISITION(
  platform_prerequisites_output,
  handler_declaration,
  binding,
  buyer_input
) → checkout_instrument
```

| 필드                           | 설명                                                                    |
| :------------------------------ | :----------------------------------------------------------------------------- |
| `platform_prerequisites_output` | 사전 요건이 필요했던 경우 플랫폼의 사전 요건 출력(구성)       |
| `handler_declaration.config`    | 비즈니스가 제공한 핸들러별 구성                               |
| `binding`                       | **(2.6 참고)** 자격 증명을 특정 기능 리소스에 결속하는 맥락 |
| `buyer_input`                   | 구매자의 결제 선택 또는 자격 증명                                       |
| `checkout_instrument`           | 체크아웃에 제출할 결제 수단                                   |

결제 핸들러 명세에서 결제 수단 획득의 정식 절차를 정의할 필요는 없습니다. 대신 명세에
다음 내용을 명확히 문서화해야 합니다(**SHOULD**).

- 핸들러의 `config`를 적용해 유효한 `checkout_instrument`를 만드는 방법
- 사용 가능한 `config`와 `binding`을 바탕으로, 보안상 매우 중요한 특정 기능 리소스에
  자격 증명을 효과적으로 바인딩하는 방법

### 결제 액션 { #payment-actions }

비즈니스가 결제 수단을 처리하는 동안 결제 핸들러가 플랫폼에 추가 작업을 요구할 수 있습니다.
비즈니스는 그 작업을 체크아웃의 `actions` 맵으로 반환합니다.

핸들러 작성자가 고를 수 있는 방법은 두 가지입니다.

- **표준 결제 인증 확장 사용.** 기기 데이터 수집이나 3DS 챌린지가 필요한 핸들러는
  [`dev.ucp.common.payment.authentication`](extensions/authentication.md)을 사용합니다.
  이 확장은 `dev.ucp.common.payment.device_data_collection`과
  `dev.ucp.common.payment.three_ds_challenge` 액션 유형을 정의합니다. 핸들러 명세에는
  핸들러가 발생시킬 수 있는 유형과, 허용된 출처(origin) 같은 제공자별 신뢰 또는 폴백 요구
  사항을 명시합니다.
- **핸들러별 액션 정의.** 표준 유형이 맞지 않으면 핸들러 작성자는 액션 유형 키와
  `config` 형태를 선언하는 체크아웃 확장을 게시합니다. 해당 액션이 발생하기 전에 비즈니스와
  플랫폼 모두 그 확장을 지원한다고 선언해야 합니다.

핸들러별 액션마다 언제 발생하는지, 그 유형이 통제하는 정확한 효과, 플랫폼이 구성을 처리하는
방법, 비즈니스가 완료를 확인하는 방법, 같은 발생 건을 안전하게 재시도할 수 있는 경우가 있는지,
그리고 신뢰·실패·중단·폴백 동작을 문서화합니다.

공통 엔벨로프는 [개요 — 액션](../overview/index.md#actions)을, 확장 스키마 패턴은
[스키마 작성 — 확장 선언 액션 유형](../../documentation/schema-authoring.md#extension-declared-action-types)을
참고하세요.

### 처리 { #processing }

**정의:** 참여자(보통 비즈니스 또는 PSP)가 받은 결제 수단을 처리해 거래를 완료하기 위해
수행하는 단계.

**시그니처:**

```text
PROCESSING(
  identity,
  checkout_instrument,
  binding,
  transaction_context
) → processing_result
```

| 필드                 | 설명                                    |
| :-------------------- | :--------------------------------------------- |
| `identity`            | 처리를 수행하는 참여자의 `PaymentIdentity` |
| `checkout_instrument` | 플랫폼에서 받은 결제 수단      |
| `binding`             | 검증을 위한 바인딩 맥락           |
| `transaction_context` | 체크아웃 합계, 라인 아이템 등              |
| `processing_result`   | 결제 세부 정보를 포함한 성공/실패 결과           |

#### 오류 처리 { #error-handling }

명세는 일반적인 실패(예: '거절(Declined)', '잔액 부족(Insufficient Funds)', '네트워크
오류(Network Error)')를 표준 UCP 오류 정의에 대응시키는 매핑을 반드시 정의해야
합니다(**MUST**). 그래야 플랫폼이 기반 결제 처리사와 관계없이 구매자에게 현지화되고
일관된 오류 메시지를 보여 줄 수 있습니다.

### 주요 정의 { #key-definitions }

| 용어        | 정의                                                                                                                                                                                                                                                                             |
| :---------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **바인딩(Binding)** | `type`과 `id`로 식별되는 특정 기능 리소스에 결제 수단을 논리적으로 연관시키는 것. 한 리소스용 자격 증명을 가로채 다른 리소스에 쓰는 재전송(replay) 공격을 막습니다. 토큰을 발급받는 대상 참여자와는 다른 개념입니다. |

---

## 명세 템플릿 { #specification-template }

핸들러 명세는 표준 템플릿 구조를 사용해야 합니다(**SHOULD**). **[REQUIRED]** 표시가 있는
섹션은 반드시 있어야 하며(**MUST**), **[CONDITIONAL]** 표시가 있는 섹션은 해당하는 경우에만
필요합니다.

**→ [결제 핸들러 템플릿](template.md)**

## 명세 작성자를 위한 준수 체크리스트 { #conformance-checklist-for-spec-authors }

결제 핸들러 명세를 게시하기 전에 다음 항목을 확인하세요.

### 구조 { #structure }

- [ ] 표준 템플릿 구조를 사용합니다
- [ ] 모든 [REQUIRED] 섹션이 있습니다
- [ ] 해당하는 경우 [CONDITIONAL] 섹션이 있습니다

### 참여자 { #participants_1 }

- [ ] 모든 참여자가 나열되어 있습니다
- [ ] 각 참여자의 역할이 명확히 설명되어 있습니다
- [ ] 해당하는 경우 "Business"와 "Merchant" 용어에 관한 참고가 추가되어 있습니다

### 사전 요건 { #prerequisites_1 }

- [ ] 사전 요건이 필요한 참여자마다 사전 요건 절차가 문서화되어 있습니다
- [ ] 온보딩 입력이 명시되어 있습니다
- [ ] 사전 요건 출력(신원 + 추가 구성)이 설명되어 있습니다
- [ ] 신원이 `PaymentIdentity` 구조(`access_token`)에 대응됩니다

### 핸들러 선언 { #handler-declaration_1 }

- [ ] 신원 스키마(기본 또는 확장)가 문서화되어 있습니다
- [ ] 구성 스키마가 (해당하는 경우) 문서화되어 있고 환경(environment)을 포함합니다
- [ ] 결제 수단 스키마(기본 또는 확장)가 문서화되어 있습니다

### 결제 수단 획득 { #instrument-acquisition_1 }

- [ ] 프로토콜 단계가 빠짐없이 명확하게 나열되어 있습니다
- [ ] 논리적 흐름이 실제 프로토콜에 대응되어 있습니다
- [ ] API 호출이나 SDK 사용법을 예시와 함께 제시합니다
- [ ] 바인딩 요구 사항이 명시되어 있습니다
- [ ] 체크아웃 결제 수단의 생성 방식과 형태가 잘 정의되어 있습니다

### 결제 인증 액션(해당하는 경우) { #payment-authentication-actions-if-applicable }

- [ ] 모든 액션 유형을 협상된 확장에서 선언합니다
- [ ] 통제되는 효과, 발생 조건, 순서가 문서화되어 있습니다
- [ ] 결제 수단과 핸들러의 연관 관계가 모호하지 않습니다
- [ ] 핸들러가 운영하는 액션 화면의 허용된 출처(origin)가 문서화되어 있습니다
- [ ] 미지원, 중단, 만료, 실패 시의 폴백이 문서화되어 있습니다
- [ ] 안전한 재시도 조건이 정의되어 있거나, 같은 발생 건의 재시도를 금지합니다
- [ ] 서버 측 결과 확인과 대기 중인 시도의 정리 방법이 정의되어 있습니다

### 처리 { #processing_1 }

- [ ] 처리 단계가 빠짐없이 명확하게 나열되어 있습니다
- [ ] 검증 요구 사항이 명시되어 있습니다
- [ ] 오류 처리와 매핑을 다룹니다

### 보안 { #security }

- [ ] 보안 요구 사항이 나열되어 있습니다
- [ ] 바인딩 검증이 필수로 지정되어 있습니다
- [ ] 자격 증명 처리 지침을 제공합니다
- [ ] (해당하는 경우) 토큰 만료가 정의되어 있습니다

### 일반 { #general }

- [ ] 핸들러 이름이 역 DNS(reverse-DNS) 규칙을 따릅니다
- [ ] 버전이 YYYY-MM-DD 형식을 따릅니다
- [ ] 모든 스키마 URL이 네임스페이스 권한과 일치합니다
- [ ] 참고 자료 섹션에 모든 스키마가 포함되어 있습니다

---

## 모범 사례 { #best-practices }

품질이 높고 유지 관리하기 쉬운 핸들러 명세를 만들려면 다음 지침을 따르세요.

### 스키마 설계 { #schema-design }

| 사례                         | 설명                                                                             |
| :------------------------------- | :-------------------------------------------------------------------------------------- |
| **새로 만들지 말고 확장하기**       | `allOf`로 기본 스키마를 조합합니다. `brand`, `last_digits` 등을 재정의하지 않습니다.        |
| **판별자에는 const 사용** | 자격 증명 유형을 모호함 없이 식별할 수 있도록 `credential.type`을 `const`로 정의합니다.       |
| **일찍 검증하기**               | 구현자가 검증할 수 있도록 명세를 확정하기 전에 안정적인 URL에 스키마를 게시합니다. |
| **만료 포함하기**               | 토큰 자격 증명을 설계할 때는 항상 `expiry` 또는 `ttl`을 포함합니다.                     |

### 문서화 { #documentation }

| 사례                  | 설명                                                            |
| :------------------------ | :--------------------------------------------------------------------- |
| **말로만 설명하지 말고 보여 주기** | 모든 스키마와 프로토콜 단계마다 완전한 JSON 예시를 포함합니다.     |
| **오류 사례 문서화** | 어떤 오류가 발생할 수 있고 참여자가 어떻게 처리해야 하는지 명시합니다. |
| **독립적인 버전 관리** | 핸들러 버전은 UCP 코어 버전과 별개로 발전합니다.        |

### 보안 { #security_1 }

| 사례                         | 설명                                                                    |
| :------------------------------- | :----------------------------------------------------------------------------- |
| **바인딩 필수화**              | 항상 `binding`으로 자격 증명을 특정 기능 리소스에 결속합니다.        |
| **자격 증명 노출 최소화** | 원본 자격 증명(PAN 등)이 되도록 적은 시스템만 거치도록 흐름을 설계합니다. |
| **토큰 수명 명시**      | 토큰이 일회용인지, 시간 제한이 있는지, 세션 범위인지 문서화합니다.       |

### 유지 관리성 { #maintainability }

| 사례                        | 설명                                                                                            |
| :------------------------------ | :----------------------------------------------------------------------------------------------------- |
| **안정적인 URL에 스키마 호스팅** | 스키마 URL은 바뀌지 않아야 합니다. 필요하면 버전이 포함된 경로를 사용합니다.                                          |
| **우아하게 실패하기**             | 일반적인 실패 시나리오마다 명확한 오류 응답을 정의합니다.                                             |
| **예시 참고하기**            | 일반적인 흐름은 기존 핸들러 명세와 [토큰화 가이드](tokenization.md)를 참고하세요.       |

---

## 함께 보기 { #see-also }

- **[토큰화 가이드](tokenization.md)** — 토큰화 결제 핸들러 구축
  가이드
- **[Google Pay 핸들러](https://developers.google.com/merchant/ucp/guides/google-pay-payment-handler){ target="_blank" }**
  — Google Pay 통합용 핸들러
- **[Shop Pay 핸들러](https://shopify.dev/docs/agents/checkout/shop-pay-handler){ target="_blank" }**
  — Shop Pay 통합용 핸들러
