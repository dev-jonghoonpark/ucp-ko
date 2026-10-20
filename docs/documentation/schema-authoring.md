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

# 스키마 작성 가이드 { #schema-authoring-guide }

이 가이드에서는 UCP JSON 스키마 작성 규칙(메타데이터 필드, 레지스트리 패턴, 스키마 변형,
버전 관리)을 설명합니다.

## 스키마 메타데이터 필드 { #schema-metadata-fields }

UCP 스키마는 표준 JSON Schema 필드와 UCP 전용 메타데이터를 함께 사용합니다.

| 필드 | 표준 | 용도 | 필수 대상 |
| ----- | -------- | ------- | ------------ |
| `$schema` | JSON Schema | JSON Schema 초안 버전 선언(`draft/2020-12` 사용을 권장(**SHOULD**)) | 모든 스키마 |
| `$id` | JSON Schema | `$ref` 해석을 위한 스키마의 정식 URI | 모든 스키마 |
| `title` | JSON Schema | 사람이 읽을 수 있는 표시 이름 | 모든 스키마 |
| `description` | JSON Schema | 스키마의 목적과 사용법 | 모든 스키마 |
| `name` | UCP | 역방향 도메인 식별자(레지스트리 키로도 사용) | 기능, 서비스, 핸들러 |
| `version` | UCP | 엔티티 버전(`YYYY-MM-DD` 형식) | 기능, 서비스, 결제 핸들러 |
| `id` | UCP | 여러 구성을 구분하기 위한 인스턴스 식별자 | 결제 핸들러 전용 |

### 왜 자기 서술적이어야 하는가? { #why-self-describing }

기능 스키마는 **자기 서술적이어야 합니다**. 플랫폼이 스키마를 가져왔을 때 다른 문서를
교차 참조하지 않고도 그 스키마가 어떤 기능의 어떤 버전인지 정확히 알 수 있어야 합니다.
이유는 다음과 같습니다.

1. **독립적인 버전 관리**: 기능은 각자 독립적으로 버전을 매길 수 있습니다. 따라서 스키마는
   버전을 명시적으로 선언해야 하며, 버전을 URL에서 추론할 수는 없습니다.

2. **검증**: 검증기는 기능 선언의 `schema` URL이 가리키는 스키마에 들어 있는
   `name`/`version`이 선언과 일치하는지 교차 확인할 수 있습니다. 불일치는 빌드 시점에 잡히는
   작성 오류입니다.

3. **개발자 경험**: 연동 개발자는 스키마 파일을 읽을 때 `$id` URL을 거꾸로 해석하지
   않아도 그 파일이 어떤 기능을 정의하는지 바로 알 수 있습니다.

4. **간결한 네임스페이스**: `name` 필드는 전체 `$id` URL보다 간결하고 의미가 분명한 표준
   역방향 도메인 식별자(예: `dev.ucp.shopping.checkout`)를 제공합니다.

### 왜 `$id`와 `name`이 모두 필요한가? { #why-both-id-and-name }

| 필드 | 역할 | 형식 |
| ------ | ------------------------------------------------------- | ---------------------- |
| `$id`  | `$ref` 해석과 도구 지원을 위한 JSON Schema 기본 요소 | URI(명세상 필수) |
| `name` | 레지스트리 키이자 안정적인 식별자 | 역방향 도메인 |

`$id`는 JSON Schema 명세에 따라 유효한 URI여야 합니다. `name`은 **레지스트리에서 쓰는
키**(`capabilities`, `services`, `payment_handlers`)이자 기능 협상에 쓰는 와이어
프로토콜 식별자입니다. 스키마 호스팅과 분리되어 있으므로 인프라가 바뀌면 `schema` URL을
바꿀 수 있습니다.

역방향 도메인 형식은 **네임스페이스 거버넌스**를 제공합니다. 도메인 소유자가 자신의
네임스페이스(`dev.ucp.*`, `com.shopify.*`)를 관리하므로 UCP 엔티티와 벤더 엔티티가
충돌하지 않습니다. 이 안정적인 식별 계층 덕분에 신뢰 및 해석 메커니즘은 독립적으로 발전할
수 있습니다. 향후 버전에서는 기능 협상을 깨뜨리지 않고도 검증 가능한 자격 증명, 콘텐츠 주소
기반 스키마, 또는 다른 검증 방식을 도입할 수 있습니다.

### 왜 `version`에 날짜를 쓰는가? { #why-version-uses-dates }

`version` 필드에 날짜 기반 버전(`YYYY-MM-DD`)을 쓰면 다음이 가능합니다.

- **기능 협상**: 플랫폼이 지원하는 특정 버전을 요청할 수 있습니다
- **호환성을 깨는 변경 관리**: 새 버전에는 새 날짜가 붙고, 이전 버전은 계속 유효하며
  해석할 수 있습니다
- **독립적인 라이프사이클**: 확장은 자체 일정에 따라 릴리스할 수 있습니다

## 스키마 분류 { #schema-categories }

UCP 스키마는 프로토콜에서 맡는 역할에 따라 여섯 가지로 나뉩니다.

### 기능 스키마 { #capability-schemas }

`ucp.capabilities{}` 레지스트리에 나타나는, 협상 대상 기능을 정의합니다.

- **최상위 필드**: `$schema`, `$id`, `title`, `description`, `name`, `version`
- **변형**: `platform_schema`, `business_schema`, `response_schema`

예: `checkout.json`, `fulfillment.json`, `discount.json`, `order.json`

### 서비스 스키마 { #service-schemas }

`ucp.services{}` 레지스트리에 나타나는 전송 바인딩을 정의합니다. 전송 방식(REST, MCP,
A2A, Embedded)마다 별도의 항목을 둡니다.

- **최상위 필드**: `$schema`, `$id`, `title`, `description`, `name`, `version`
- **변형**: `platform_schema`, `business_schema`
- **전송 방식 요구 사항**(공통 기반에 추가되는 사항):
    - 플랫폼 프로필(`platform_schema`): REST/MCP/Embedded에는 `schema`(OpenAPI/OpenRPC URL)가 필요합니다. A2A에는 추가 요구 사항이 없습니다.
    - 비즈니스 프로필(`business_schema`): REST/MCP/A2A에는 `endpoint`(A2A의 경우 Agent Card URL)가 필요합니다. Embedded에는 추가 요구 사항이 없습니다.

### 결제 핸들러 스키마 { #payment-handler-schemas }

`ucp.payment_handlers{}` 레지스트리의 결제 핸들러 구성을 정의합니다.

- **최상위 필드**: `$schema`, `$id`, `title`, `description`, `name`, `version`, `available_instruments`
- **변형**: `platform_schema`, `business_schema`, `response_schema`
- **인스턴스 `id`**: 같은 핸들러의 여러 구성을 구분하는 데 필요합니다
- **`available_instruments`**: 선택 사항입니다. 지원하는 결제 수단 유형의 배열이며, 유형별 제약(예: 신용카드의 브랜드)을 함께 둘 수 있습니다. 생략하면 핸들러는 아무 제한도 두지 않으며, 핸들러 스키마가 정의한 모든 결제 수단 유형을 지원합니다.

예: `com.google.pay`, `dev.shopify.shop_pay`, `dev.ucp.processor_tokenizer`

핸들러 구조, config/instrument/credential 스키마, 전체 명세 템플릿에 대한 자세한 안내는
**→ [결제 핸들러 가이드](../specification/payment-handler-guide.md)를 참고하세요**.

### 구성 요소 스키마 { #component-schemas }

기능 안에 포함되지만 독립적으로 협상되지 않는 데이터 구조입니다. 레지스트리에 나타나지
**않습니다**.

- **최상위 필드**: `$schema`, `$id`, `title`, `description`
- **생략**: `name`, `version`(독립적으로 버전을 관리하지 않음)

예:

- `schemas/shopping/payment.json` — 결제 구성(체크아웃의 일부)

### 타입 스키마 { #type-schemas }

다른 스키마가 참조하는 재사용 가능한 정의입니다. 레지스트리에 나타나지 **않습니다**.

- **최상위 필드**: `$schema`, `$id`, `title`, `description`
- **생략**: `name`, `version`

예: `types/buyer.json`, `types/line_item.json`, `types/postal_address.json`

### 메타 스키마 { #meta-schemas }

엔티티 페이로드가 아니라 프로토콜 구조를 정의합니다.

- **최상위 필드**: `$schema`, `$id`, `title`, `description`
- **생략**: `name`, `version`

예: `ucp.json`(엔티티 기반), `capability.json`, `service.json`, `payment_handler.json`

## 레지스트리 패턴 { #the-registry-pattern }

UCP는 기능, 서비스, 핸들러를 **레지스트리**로 구성합니다. 레지스트리는 `name` 필드가 있는
객체의 배열이 아니라 `name`을 키로 하는 객체입니다.

```json
{
  "capabilities": {
    "dev.ucp.shopping.checkout": [{"version": "{{ ucp_version }}"}],
    "dev.ucp.shopping.fulfillment": [{"version": "{{ ucp_version }}"}]
  },
  "services": {
    "dev.ucp.shopping": [
      {"version": "{{ ucp_version }}", "transport": "rest"},
      {"version": "{{ ucp_version }}", "transport": "mcp"}
    ]
  },
  "payment_handlers": {
    "com.google.pay": [{"id": "gpay_1234", "version": "{{ ucp_version }}", "available_instruments": [{"type": "google_pay_card"}]}]
  }
}
```

### 레지스트리 컨텍스트 { #registry-contexts }

같은 레지스트리 구조가 세 가지 컨텍스트에 나타나며, 컨텍스트마다 필드 요구 사항이 다릅니다.

| 컨텍스트 | 위치 | 필수 필드 |
| ------- | -------- | --------------- |
| 플랫폼 프로필 | 플랫폼이 알린 URI | `version`, `spec`, `schema` |
| 비즈니스 프로필 | `/.well-known/ucp` | `version`; `config` 추가 가능 |
| API 응답 | 체크아웃/주문 페이로드 | `version`(핸들러는 `id` 추가) |

## 엔티티 패턴 { #the-entity-pattern }

모든 기능, 서비스, 핸들러는 공통 `entity` 기반 스키마를 확장합니다.

| 필드 | 타입 | 설명 |
| ----- | ---- | ----------- |
| `version` | string | 엔티티 버전(`YYYY-MM-DD`) — 항상 필수 |
| `spec` | URI | 사람이 읽을 수 있는 명세 |
| `schema` | URI | JSON Schema URL |
| `id` | string | 인스턴스 식별자(핸들러 전용) |
| `config` | object | 엔티티별 구성 |

### 스키마 변형 { #schema-variants }

각 엔티티 유형은 컨텍스트에 따라 **세 가지 변형**을 정의합니다.

**`platform_schema`** — 디스커버리를 위한 전체 선언

```json
{
  "dev.ucp.shopping.fulfillment": [{
    "version": "{{ ucp_version }}",
    "spec": "https://ucp.dev/{{ ucp_version }}/specification/fulfillment",
    "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
    "config": {
      "supports_multi_group": true
    }
  }]
}
```

**`business_schema`** — 비즈니스별 재정의

```json
{
  "dev.ucp.shopping.fulfillment": [{
    "version": "{{ ucp_version }}",
    "config": {
      "allows_multi_destination": {"shipping": true}
    }
  }]
}
```

**`response_schema`** — API 응답의 최소 참조

```json
{
  "ucp": {
    "capabilities": {
      "dev.ucp.shopping.fulfillment": [{"version": "{{ ucp_version }}"}]
    }
  }
}
```

세 가지 모두 스키마의 `$defs`에 정의합니다.

```json
"$defs": {
  "platform_schema": {
    "allOf": [{"$ref": "../capability.json#/$defs/platform_schema"}]
  },
  "business_schema": {
    "allOf": [{"$ref": "../capability.json#/$defs/business_schema"}]
  },
  "response_schema": {
    "allOf": [{"$ref": "../capability.json#/$defs/response_schema"}]
  }
}
```

## 문자열 어휘와 열거형 { #string-vocabularies-vs-enums }

닫힌 `enum` 배열보다는 잘 알려진 값을 문서화한 **열린 문자열 어휘**를 권합니다. 열거형은
한번 들어가면 되돌릴 수 없는 문입니다. 새 값을 추가하면 엄격한 검증기에서는 호환성을 깨는 변경이 되고,
값을 제거하면 기존 생산자가 깨집니다.

```json
// PREFER: open vocabulary — extensible without schema changes
"type": {
  "type": "string",
  "description": "Media type. Well-known values: `image`, `video`, `model_3d`."
}

// AVOID: closed enum — adding `audio` requires a schema version bump
"type": {
  "type": "string",
  "enum": ["image", "video", "model_3d"]
}
```

**`enum`은 닫혀 있다고 확신할 수 있는 집합에만 사용합니다.** 즉, 새 값이 곧 프로토콜의 근본적인
변경을 뜻하는 경우입니다(예: `checkout.status: open | completed | expired`). 새로운 사용
사례가 생기면서 집합이 커질 수 있다면, 잘 알려진 값을 `description`에 문서화한 열린
문자열을 사용합니다.

## 버전 관리 전략 { #versioning-strategy }

### UCP 기능(`dev.ucp.*`) { #ucp-capabilities-devucp }

UCP가 작성한 기능은 기본적으로 프로토콜 릴리스와 함께 버전이 매겨집니다. 필요하면 개별
기능이 독립적으로 버전을 매길 수도 있습니다(**may**).

### 벤더 기능(`com.{vendor}.*`)

`dev.ucp.*` 밖의 기능은 완전히 독립적으로 버전을 매깁니다.

```json
{
  "name": "com.shopify.loyalty",
  "version": "2025-09-01",
  "spec": "https://shopify.dev/ucp/loyalty",
  "schema": "https://shopify.dev/ucp/schemas/loyalty.json"
}
```

벤더 스키마도 똑같이 자기 서술적이어야 한다는 요구 사항을 따릅니다.

## 확장성과 전방 호환성 { #extensibility-and-forward-compatibility }

스키마를 설계할 때는 이전 클라이언트가 새 페이로드를 어떻게 검증할지 고려해야 합니다.
Protobuf 같은 직렬화 형식에서는 새 필드나 enum 값을 추가해도 대체로 안전한 전방 호환
변경입니다.

그러나 최신 코드 생성기(예: [Quicktype](https://quicktype.io/))는 JSON Schema를 엄격하게
타입이 지정된 클래스(예: Go 구조체나 Java Enum)로 변환하므로, 프로토콜이 발전하면서 특정
스키마 제약이 이전 클라이언트에서 역직렬화 오류를 일으킵니다. 이런 변경을 피하면 프로토콜
버전을 올려야 하는 일을 최소화할 수 있습니다.

### 열린 열거형 { #open-enumerations }

필드의 값 목록이 앞으로 늘어날 수 있다면(예: `"refunded"` 상태나 새 결제 방식 추가)
**`enum`을 사용하지 않습니다**.

대신 표준 `string`을 정의하고, 알 수 없는 값을 무시해야 한다는 요구 사항을 `description`에
문서화하며, `examples`로 현재 예상되는 값을 코드 생성기에 알려 줍니다. 복잡한 "열린 집합"
검증 패턴(예: `anyOf`와 `const`의 조합)은 클라이언트 측 코드 생성기를 자주 혼란스럽게 하고
스키마를 읽기 어렵게 만들므로 피합니다.

```json
"cancellation_reason": {
  "type": "string",
  "description": "Reason for order cancellation. Clients MUST tolerate and ignore unknown values.",
  "examples": ["customer_requested", "inventory_shortage", "fraud_suspected"]
}
```

### 닫힌 열거형 { #closed-enumerations }

엄격한 `enum`이나 `const`는 도메인이 영구적으로 고정되어 있거나 알 수 없는 값을 본질적으로
지원할 수 없는 경우에만 사용합니다. 새 값을 추가하면 연동 개발자가 반드시 코드를 수정해야
하는 경우(예: 프로토콜 버전, 엄격한 타입 판별자, 요일)를 위해 남겨 둡니다.

```json
"status": {
  "type": "string",
  "enum": ["open", "completed", "expired"],
  "description": "Lifecycle state. This domain is strictly bounded; unknown states represent a breakdown in the state machine and MUST be rejected."
}
```

### 열린 객체(`additionalProperties`) { #open-objects-additionalproperties }

객체를 미리 닫아 두면 앞으로 스키마에 호환성을 깨지 않는 추가를 할 수 없게 됩니다. 분산
프로토콜에서는 원래 하위 호환인 필드 추가(예: 주문에 "gift_message" 필드 추가)도
닫힌 스키마로 검증하는 클라이언트에게는 호환성을 깨는 변경이 됩니다.

JSON Schema는 기본적으로 열려 있어 알 수 없는 속성을 무시합니다. 작성자는 드문 경우를 빼고는
이 키워드를 생략해야 합니다. 드문 경우란 다형성 판별자(엄격함이 oneOf 검증의 모호성을
막는 경우), 보안이 중요한 페이로드(알 수 없는 필드가 변조를 뜻할 수 있는 경우), 프로토콜
엔벌로프(`ucp` 블록 같은 핵심 메타데이터의 오타를 잡는 데 엄격함이 유용한 경우)입니다.

**안티패턴(버전을 되돌리지 않고는 새 필드를 추가할 수 없게 만듭니다):**

```json
"totals": {
  "type": "object",
  "properties": {
    "subtotal": {"type": "integer"}
  },
  "additionalProperties": false
}
```

### 속성 개수 제약(`minProperties` / `maxProperties`) { #property-count-constraints-minproperties-maxproperties }

UCP 스키마는 기본적으로 객체 필드에 `minProperties`나 `maxProperties`를 설정하지 않습니다.

- **`maxProperties`** — 한도는 구현자에게 맡깁니다. 특정 한도를 정하려면 판단이 필요하고
  그 판단은 결국 예외에 부딪히므로, 프로토콜은 상한을 정의하지 않습니다. 구현자는 자체 제약을
  두되, 디버깅과 올바른 동작을 돕도록 명확한 오류 피드백을 제공하기를 권합니다.
- **`minProperties`** — 빈 객체(`{}`)는 형식이 올바르며 해가 없습니다. 구현자는 이를 받아들여
  아무 동작도 하지 않는 것(no-op)으로 처리해야 합니다.

## 전체 예시: 기능 스키마 { #complete-example-capability-schema }

기능 스키마는 페이로드 구조와 선언 변형을 모두 정의합니다.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/checkout.json",
  "name": "dev.ucp.shopping.checkout",
  "version": "{{ ucp_version }}",
  "title": "Checkout",
  "description": "Base checkout schema. Extensions compose via allOf.",

  "$defs": {
    "platform_schema": {
      "allOf": [{"$ref": "../capability.json#/$defs/platform_schema"}]
    },
    "business_schema": {
      "allOf": [{"$ref": "../capability.json#/$defs/business_schema"}]
    },
    "response_schema": {
      "allOf": [{"$ref": "../capability.json#/$defs/response_schema"}]
    }
  },

  "type": "object",
  "required": ["ucp", "id", "line_items", "status", "currency", "totals", "links"],
  "properties": {
    "ucp": {"$ref": "../ucp.json#/$defs/response_checkout_schema"},
    "id": {"type": "string", "description": "Checkout identifier"},
    "line_items": {"type": "array", "items": {"$ref": "types/line_item.json"}},
    "status": {"type": "string", "enum": ["open", "completed", "expired"]},
    "currency": {"type": "string", "pattern": "^[A-Z]{3}$"},
    "totals": {"$ref": "types/totals.json"},
    "links": {"$ref": "types/links.json"}
  }
}
```

핵심 사항:

- **최상위 `name`과 `version`** 덕분에 스키마가 자기 서술적입니다
- **`$defs` 변형**으로 서로 다른 컨텍스트에서 검증할 수 있습니다
- **페이로드 속성**이 실제 체크아웃 응답 구조를 정의합니다
