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

이 가이드는 UCP JSON 스키마 작성 규칙(메타데이터 필드, 레지스트리 패턴, 스키마 변형,
버전 관리)을 설명한다.

## 스키마 메타데이터 필드 { #schema-metadata-fields }

UCP 스키마는 표준 JSON Schema 필드와 UCP 전용 메타데이터를 함께 사용한다.

| 필드 | 표준 | 용도 | 필수 대상 |
| ----- | -------- | ------- | ------------ |
| `$schema` | JSON Schema | JSON Schema 초안 버전을 선언한다(`draft/2020-12` 사용을 권장(**SHOULD**)) | 모든 스키마 |
| `$id` | JSON Schema | `$ref` 해석을 위한 스키마의 정식 URI | 모든 스키마 |
| `title` | JSON Schema | 사람이 읽을 수 있는 표시 이름 | 모든 스키마 |
| `description` | JSON Schema | 스키마의 목적과 사용법 | 모든 스키마 |
| `name` | UCP | 역방향 도메인 식별자. 레지스트리 키로도 쓰인다 | 기능, 확장, 결제 핸들러 |
| `version` | UCP | 엔티티 버전(`YYYY-MM-DD` 형식) | 기능, 확장, 결제 핸들러 |
| `id` | UCP | 여러 구성을 구분하기 위한 인스턴스 식별자 | 결제 핸들러 전용 |

이 표는 게시된 스키마 문서에 포함되는 최상위 메타데이터를 나열한다. 프로필 레지스트리
선언은 아래 섹션에서 다룬다.

### 왜 자기 서술적이어야 하는가? { #why-self-describing }

기능 스키마와 결제 핸들러 스키마는 **자기 서술적이어야 한다**. 플랫폼이 스키마를
가져왔을 때 다른 문서를 교차 참조하지 않고도 그 스키마가 어떤 엔티티의 어떤 버전을
나타내는지 정확히 알 수 있어야 한다. 그 이유는 다음과 같다.

1. **명시적인 버전 식별**: 기능, 확장, 결제 핸들러는 버전을 명시적으로 선언한다. 버전은
   URL에서 추론할 수 없다. UCP가 작성한 `dev.ucp.*` 기능과 확장은 각 UCP 릴리스 `D`에서
   버전 `D`를 선언한다. 결제 핸들러는 작성자가 정한 일정에 따라 게시된다.

2. **검증**: 검증기는 기능 선언의 `schema` URL이 가리키는 스키마에 포함된
   `name`/`version`이 선언과 일치하는지 교차 확인할 수 있다. 불일치는 빌드 시점에 잡히는
   작성 오류다.

3. **개발자 경험**: 연동 개발자는 스키마 파일을 읽을 때 `$id` URL을 역으로 해석하지
   않아도 그 파일이 어떤 엔티티를 정의하는지 바로 알 수 있다.

4. **간결한 네임스페이스**: `name` 필드는 전체 `$id` URL보다 간결하고 의미가 분명한 표준
   역방향 도메인 식별자(예: `dev.ucp.shopping.checkout`)를 제공한다. 서비스 레지스트리
   키도 같은 안정적인 식별자 역할을 한다.

### 왜 `$id`와 `name`이 모두 필요한가? { #why-both-id-and-name }

| 필드 | 역할 | 형식 |
| ------ | ------------------------------------------------------- | ---------------------- |
| `$id`  | `$ref` 해석과 도구 지원을 위한 JSON Schema 기본 요소 | URI(명세상 필수) |
| `name` | 레지스트리 키이자 안정적인 식별자 | 역방향 도메인 |

`$id`는 JSON Schema 명세에 따라 유효한 URI여야 한다. `name`은 **레지스트리에서 쓰이는
키**(`capabilities`, `services`, `payment_handlers`)이자 기능 협상에서 쓰이는 와이어
프로토콜 식별자다. 스키마 호스팅과 분리되어 있으므로 인프라가 바뀌어도 `schema` URL을
변경할 수 있다.

역방향 도메인 형식은 **네임스페이스 거버넌스**를 제공한다. 도메인 소유자가 자신의
네임스페이스(`dev.ucp.*`, `com.shopify.*`)를 관리하므로 UCP 엔티티와 벤더 엔티티가
충돌하지 않는다. 이 안정적인 식별 계층 덕분에 신뢰 및 해석 메커니즘은 독립적으로 발전할
수 있다. 향후 버전에서는 기능 협상을 깨뜨리지 않고도 검증 가능한 자격 증명, 콘텐츠 주소
기반 스키마, 또는 다른 검증 방식을 도입할 수 있다.

### 왜 `version`에 날짜를 쓰는가? { #why-version-uses-dates }

`version` 필드는 날짜 기반 버전(`YYYY-MM-DD`)을 사용하여 다음을 가능하게 한다.

- **기능 협상**: 플랫폼이 지원하는 특정 버전을 요청한다
- **인증된 릴리스 식별**: 각 날짜는 엔티티의 완전한, 게시된 스키마 폐포(closure)를
  식별한다
- **작성자별 독립적인 라이프사이클**: 서드파티 확장과 결제 핸들러는 작성자가 정한 일정에
  따라 릴리스할 수 있다

## 스키마 분류 { #schema-categories }

UCP 스키마는 프로토콜에서의 역할에 따라 여섯 가지로 분류된다.

### 기능 스키마 { #capability-schemas }

`ucp.capabilities{}` 레지스트리에 나타나는, 협상 대상 기능을 정의한다.

- **최상위 필드**: `$schema`, `$id`, `title`, `description`, `name`, `version`
- **변형**: `platform_schema`, `business_schema`, `response_schema`

예: `checkout.json`, `fulfillment.json`, `discount.json`, `order.json`

### 서비스 스키마 { #service-schemas }

`ucp.services{}` 레지스트리에 나타나는 전송 바인딩을 정의한다. 각 전송 방식(REST, MCP,
A2A, Embedded)은 별도의 항목이다.

- **레지스트리 식별자**: 레지스트리 키로 쓰이는 역방향 도메인 서비스 이름
- **항목 필드**: `version`, `spec`, `schema`, `config`, 그리고 전송 방식별 필드
- **버전 관리**: 모든 서비스 항목은 명시적인 `version`을 선언하며, 릴리스 `D`에서는 그
  값이 `D`다. 각 항목은 서비스와 하나의 전송 바인딩을 짝지으므로 서비스 `version`이
  항목마다 반복되며, 전송 바인딩에는 별도의 버전이 없다. 참조되는 OpenAPI/OpenRPC
  아티팩트는 자체 `info.version`을 갖지만, 이는 협상 대상 버전이 아니라 아티팩트
  메타데이터다.
  [구성 요소 버전 관리와 릴리스 스냅샷](../specification/overview/index.md#component-versioning-and-release-snapshots)을 참고한다
- **변형**: `platform_schema`, `business_schema`
- **전송 방식 요구 사항**(공통 기반에 추가되는 사항):
    - 플랫폼 프로필(`platform_schema`): REST/MCP/Embedded는 `schema`(OpenAPI/OpenRPC URL)가 필요하다. A2A는 추가 요구 사항이 없다.
    - 비즈니스 프로필(`business_schema`): REST/MCP/A2A는 `endpoint`(A2A의 경우 Agent Card URL)가 필요하다. Embedded는 추가 요구 사항이 없다.

### 결제 핸들러 스키마 { #payment-handler-schemas }

`ucp.payment_handlers{}` 레지스트리의 결제 핸들러 구성을 정의한다.

- **최상위 필드**: `$schema`, `$id`, `title`, `description`, `name`, `version`, `available_instruments`
- **변형**: `platform_schema`, `business_schema`, `response_schema`
- **인스턴스 `id`**: 같은 핸들러의 여러 구성을 구분하기 위해 필요하다
- **`available_instruments`**: 선택 사항. 지원하는 결제 수단 유형의 배열이며, 각 항목은 유형별 멤버(예: 신용카드의 `brand`)에 대한 선택적 제약 표현식(Constraint Expression)을 가질 수 있다. 생략하면 핸들러는 아무 제한도 두지 않으며, 핸들러 스키마가 정의한 모든 결제 수단 유형을 지원한다.

예: `com.google.pay`, `dev.shopify.shop_pay`, `dev.ucp.processor_tokenizer`

핸들러 구조, config/instrument/credential 스키마, 전체 명세 템플릿에 대한 자세한 안내는
**→ [결제 핸들러 가이드](../specification/payment/guide.md)를 참고한다**.

### 구성 요소 스키마 { #component-schemas }

기능 안에 포함되지만 독립적으로 협상되지 않는 데이터 구조다. 레지스트리에 나타나지
**않는다**.

- **최상위 필드**: `$schema`, `$id`, `title`, `description`
- **생략**: `name`, `version`(독립적으로 버전이 관리되지 않음)

예:

- `schemas/common/types/payment.json` — 결제 구성(소매 쇼핑의 체크아웃 같은 하위 퍼널 기능의 일부)

### 타입 스키마 { #type-schemas }

다른 스키마가 참조하는 재사용 가능한 정의다. 레지스트리에 나타나지 **않는다**.

- **최상위 필드**: `$schema`, `$id`, `title`, `description`
- **생략**: `name`, `version`
- **표현식 문법**: 독립적으로 배치할 수 있는 UCP 필드가 아니라 표현식 구문을 정의하는
  타입 스키마는 방향(요청/응답)과 무관하다. 그 문법 속성에는 `ucp_request` 주석이 없으며,
  문법을 노출하는 각 UCP 속성이 자신의 적용 범위를 선언한다. 예:
  `common/types/constraint_expression.json`.

예: `types/buyer.json`, `types/line_item.json`, `types/postal_address.json`

### 메타 스키마 { #meta-schemas }

엔티티 페이로드가 아니라 프로토콜 구조를 정의한다.

- **최상위 필드**: `$schema`, `$id`, `title`, `description`
- **생략**: `name`, `version`

예: `ucp.json`(엔티티 기반), `capability.json`, `service.json`, `payment_handler.json`

## 레지스트리 패턴 { #the-registry-pattern }

UCP는 기능, 서비스, 핸들러를 **레지스트리**로 구성한다. 레지스트리는 `name` 필드를 가진
객체의 배열이 아니라 `name`을 키로 하는 객체다.

<!-- ucp:example skip reason="schema authoring example" -->
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

같은 레지스트리 구조가 필드 요구 사항이 서로 다른 세 가지 컨텍스트에 나타난다.

| 레지스트리 | 플랫폼 프로필 | 비즈니스 프로필 | API 응답 |
| -------- | ---------------- | ---------------- | ------------- |
| 서비스 | `version`, `transport`, `spec`; REST·MCP·Embedded는 `schema` | `version`, `transport`; REST·MCP·A2A는 `endpoint` | `version`, `transport`; 해당하는 경우 전송 방식별 `config` |
| 기능/확장 | `version`, `spec`, `schema` | `version`, `schema`; `config` 추가 가능 | `version` |
| 결제 핸들러 | `id`, `version`, `spec`, `schema` | `id`, `version`; `config` 추가 가능 | `id`, `version` |

## 예약된 `ucp` 멤버 { #the-reserved-ucp-member }

멤버 이름 `ucp`는 모든 구조화된 UCP 객체 범위(멤버가 스키마에 정의된 필드인 객체)에서
프로토콜 네임스페이스로 예약되어 있다. 최상위 엔벌로프는 그 루트 형태다. 딕셔너리
컨테이너는 키가 필드가 아니라 데이터이므로 제외된다. 규범적 규칙은
[`ucp` 프로토콜 네임스페이스](../specification/overview/index.md#the-ucp-protocol-namespace)를
참고한다. 스키마 작성자에게 이는 다음을 의미한다.

- **`ucp`라는 이름의 구조화된 도메인 필드를 절대 만들지 않는다.** 스키마 작성자는
  구조화된 객체에 `ucp`라는 이름의 도메인 필드를 선언해서는 안 된다(**MUST NOT**). 그
  위치에서 이 이름은 프로토콜 네임스페이스를 뜻한다.
- **열린 중첩 객체에 암묵적(ambient) `ucp`를 선언하지 않는다.** 루트 `ucp` 엔벌로프는
  암묵적 배치가 아니며, UCP 루트 스키마가 이미 필수 프로토콜 메타데이터를 선언하고 있다.
  루트 아래에서 이 멤버는 중앙에서 정의되는 UCP 문서 문법이므로, 스키마 작성자는 열린
  중첩 객체에 이를 선언해서는 안 된다(**MUST NOT**). 비즈니스나 플랫폼은 도메인 스키마에
  명시되어 있지 않아도 적격한 열린 구조화 범위에 이를 포함할 수 있다(**MAY**). 열린 UCP
  원본 도메인 스키마에 대한 일반적인 인스턴스 검증은 이를 무시되는 알 수 없는 객체로
  취급한다. 그 내용을 검증하는 것은 준수 검증 도구의 몫이다.
- **딕셔너리 키는 데이터로 남는다.** 딕셔너리 컨테이너는 프로토콜 네임스페이스를 담을 수
  없다. 스키마 작성자는 `ucp`라는 이름의 딕셔너리 키를 그 네임스페이스로 모델링해서는 안
  된다(**MUST NOT**). 그 키는 딕셔너리의 키·값 스키마가 규정하는 일반 데이터다.
  딕셔너리 값으로 쓰이는 구조화된 객체는 여전히 적격한 범위이며 이 규칙을 따른다. 예를
  들어 `attribution`은 문자열 값의 딕셔너리이므로, 그 안의 `ucp` 키는 일반적인 어트리뷰션
  데이터다.
- **어휘는 UCP 코어에서만 늘어난다.** 새 프로토콜 멤버는 도메인이나 확장 스키마가 아니라
  `ucp.json#/$defs/members`에만 추가된다. 또한 무시해도 안전한 멤버만 허용된다. 즉, 그
  멤버를 처리하지 않는 비즈니스나 플랫폼은 그 멤버의 이점만 잃을 뿐 정확성을 잃지 않는다.
  이 컨테이너가 열려 있는 이유는 향후 UCP 버전과의 전방 호환성 때문이지, 확장 지점으로
  쓰기 위해서가 아니다.
- **레지스트리 맵은 결코 `ucp`를 담을 수 없다.** 레지스트리 맵은 딕셔너리이므로 프로토콜
  네임스페이스를 담을 수 없다. 게다가 역방향 도메인 `propertyNames` 제약이 리터럴 `ucp`
  키를 거부한다. 그래서 `map_order` 같은 레지스트리 맵의 구조 메타데이터는 맵 내부가
  아니라 상위 구조화 범위의 프로토콜 네임스페이스에 위치한다.
- **닫힌 중첩 구조화 객체는 선택적 `ucp`를 명시적으로 선언해야 한다.** 중첩된 구조화
  객체를 닫는다고 해서 프로토콜 문법에서 면제되지는 않는다. `additionalProperties: false`로
  그런 객체를 정의하는 스키마 작성자는 `ucp.json#/$defs/members`를 참조하는 선택적 `ucp`
  속성을 반드시 선언해야 한다(**MUST**). 이 예외는 중첩된 구조화 객체에만 적용되며,
  딕셔너리를 닫는다고 해서 그 안에 암묵적 네임스페이스가 생기지는 않는다.
- **멤버 적용 범위를 선언한다.** 스키마 작성자는 `ucp.json#/$defs/members`에 등록된 모든
  속성에 `ucp_request`(상황에 맞게 `omit`, `optional`, `required`) 주석을 반드시 달아야
  한다(**MUST**). 같은 멤버가 `ucp.json`의 다른 곳에서 노출될 때마다 이 주석을 반드시
  반복해야 한다(**MUST**). 명시적으로 선언된 암묵적 `ucp` 속성은 그것을 담은 스키마의 적용
  범위를 따르며, 요청에서 자동으로 생략되지 않는다.

## 엔티티 패턴 { #the-entity-pattern }

모든 기능, 서비스, 핸들러는 공통 `entity` 기반 스키마를 확장한다.

| 필드 | 타입 | 설명 |
| ----- | ---- | ----------- |
| `version` | string | 엔티티 버전(`YYYY-MM-DD`) — 항상 필수 |
| `spec` | URI | 사람이 읽을 수 있는 명세 |
| `schema` | URI | JSON Schema URL |
| `id` | string | 인스턴스 식별자(핸들러 전용) |
| `config` | object | 엔티티별 구성 |

### 스키마 변형 { #schema-variants }

각 엔티티 유형은 컨텍스트에 따라 **세 가지 변형**을 정의한다.

**`platform_schema`** — 디스커버리를 위한 전체 선언

<!-- ucp:example skip reason="schema authoring example" -->
```json
{
  "dev.ucp.shopping.fulfillment": [{
    "version": "{{ ucp_version }}",
    "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/fulfillment",
    "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
    "config": {
      "supports_multi_group": true
    }
  }]
}
```

**`business_schema`** — 비즈니스별 재정의

<!-- ucp:example skip reason="schema authoring example" -->
```json
{
  "dev.ucp.shopping.fulfillment": [{
    "version": "{{ ucp_version }}",
    "config": {
      "multi_destination": [{"method": "shipping"}]
    }
  }]
}
```

**`response_schema`** — API 응답의 최소 참조

<!-- ucp:example skip reason="schema authoring example" -->
```json
{
  "ucp": {
    "capabilities": {
      "dev.ucp.shopping.fulfillment": [{"version": "{{ ucp_version }}"}]
    }
  }
}
```

세 가지 모두 스키마의 `$defs`에 정의한다.

<!-- ucp:example skip reason="schema authoring example" -->
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

닫힌 `enum` 배열보다 잘 알려진 값을 문서화한 **열린 문자열 어휘**를 선호한다. 열거형은
되돌릴 수 없는 문이다. 새 값을 추가하면 엄격한 검증기에서는 호환성을 깨는 변경이 되고,
값을 제거하면 기존 생산자가 깨진다.

<!-- ucp:example skip reason="schema authoring example" -->
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

**`enum`은 닫혀 있음이 확실한 집합에만 사용한다.** 즉, 새 값이 프로토콜의 근본적인
변경을 의미하는 경우다(예: `checkout.status: open | completed | expired`). 새로운 사용
사례가 생기면서 집합이 커질 수 있다면, 잘 알려진 값을 `description`에 문서화한 열린
문자열을 사용한다.

### 값별 필드 의미 { #per-value-field-semantics }

열린 `type` 어휘는 검증 결과는 그대로 두면서 기존 필드의 *의미*를 바꿀 수 있다. 위의
미디어 `type`이 바로 그런 예다. `url`은 `image`에서는 표시할 이미지, `video`에서는 재생
가능한 파일이나 스트리밍 매니페스트, `external_video`에서는 임베드할 서드파티 플레이어다.
어느 경우든 같은 필수 URL이다. 기반에는 하나의 스키마와 하나의 필드 집합을 유지하고,
값별 의미는 형제 스키마 파일의 설명 전용 재정의로 표현한다. 기반 스키마는 `type`에 대한
`allOf` `if`/`then` 분기로 그 형제 파일을 선택한다.

<!-- ucp:example skip reason="schema authoring example" -->
```json
// media.json — every field is defined here; a branch selects a sibling by `type`
"allOf": [{
  "if": { "properties": { "type": { "const": "video" } }, "required": ["type"] },
  "then": { "$ref": "media_video.json" }
}]

// media_video.json — description-only overrides: no types, no constraints
"properties": {
  "url": { "description": "A playable rendition of the video: a progressive file or an adaptive-streaming manifest." },
  "sources": { "description": "Alternate renditions (MP4/WebM) and adaptive manifests (HLS/DASH)." }
}
```

분기에는 제약이 없다. 모든 필드는 기반에 있으며, 시간 기반 미디어의 `duration`처럼
일부 값에만 적용되는 필드도 마찬가지다. 그런 필드는 설명에 적용 범위를 밝힌다. JSON
Schema 코드 생성기는 `if`/`then`을 반영하지 않으므로, 분기 안에서만 정의된 필드는 생성된
타입에서 빠지는 반면 기반에 있는 필드는 모든 값에 존재한다. `schema_fields` 매크로는 각
형제 파일을 값별 표로 렌더링하며, 타입과 필수 여부는 그 파일을 참조하는 기반에서 채운다.
그래서 재정의 행은 `url | string | Required | <per-value description>`처럼 표시된다.

어떤 값에 실제 제약(그 값만 요구하는 필드, 그 값만 허용하는 형식)이 필요해지면, 그
제약을 둘 곳은 이미 분기로 마련되어 있다.

## 버전 관리 전략 { #versioning-strategy }

### UCP 서비스(`dev.ucp.*`) { #ucp-services-devucp }

각 UCP 릴리스 `D`에서 UCP가 정의한 모든 서비스 항목은 버전 `D`를 선언한다. 평탄화된
레지스트리가 서비스 버전과 전송 바인딩을 어떻게 표현하는지는
[서비스 스키마](#service-schemas)를 참고한다.

### UCP 기능(`dev.ucp.*`) { #ucp-capabilities-devucp }

각 UCP 릴리스 `D`에서 UCP가 정의한 모든 기능과 확장은 스키마가 직접 바뀌지 않았더라도
버전 `D`를 선언한다. 버전 `D`를 선언한다고 해서 협상이 대체되지는 않는다. 기능과 확장은
여전히 정확한 버전의 교집합으로 선택된다.

이전에 지원되던 릴리스의 프로필을 포함한 프로필 선택 방식은
[프로토콜 버전](../specification/overview/index.md#protocol-version)에 정의되어 있다.

### 서드파티 확장과 결제 핸들러 { #third-party-extensions-and-payment-handlers }

서드파티 확장과 결제 핸들러는 작성자가 정한 일정에 따라 버전을 게시한다. 그 버전은
선택된 `ucp.version`과 독립적이며 `D`로 제한되지 않는다. UCP의 `payment_handler.json`은
공통 선언 형태를 정의할 뿐, 결제 핸들러 구현이나 릴리스 주기를 정의하지 않는다.

서드파티 확장은 독립적으로 버전을 관리한다.

<!-- ucp:example skip reason="schema authoring example" -->
```json
{
  "name": "com.shopify.loyalty",
  "version": "2025-09-01",
  "spec": "https://shopify.dev/ucp/loyalty",
  "schema": "https://shopify.dev/ucp/schemas/loyalty.json"
}
```

확장의 버전 요구 사항은 확장 스키마 문서에 둔다. `requires.protocol`은 선택된
`ucp.version`을, `requires.capabilities`는 선택된 상위 기능의 버전을 제약한다.

## 확장성과 전방 호환성 { #extensibility-and-forward-compatibility }

스키마를 설계할 때는 이전 클라이언트가 새로운 페이로드를 어떻게 검증할지 고려해야 한다.
Protobuf 같은 직렬화 형식에서는 새 필드나 enum 값을 추가하는 것이 대체로 안전한 전방 호환
변경이다.

그러나 최신 코드 생성기(예: [Quicktype](https://quicktype.io/))는 JSON Schema를 엄격하게
타입이 지정된 클래스(예: Go 구조체나 Java Enum)로 변환하므로, 프로토콜이 발전하면서 특정
스키마 제약이 이전 클라이언트에서 역직렬화 오류를 일으킨다. 이런 변경을 피하면 프로토콜
버전을 올려야 하는 일을 최소화할 수 있다.

두 번째 실패 유형은 합성 자체에서 비롯된다. 확장은 `allOf`로 기반을 확장하는데, 이 방식은
제약을 *추가*할 수만 있다. 두 enum(또는 두 `oneOf`)의 `allOf`는 멤버의 *교집합*을 만들 뿐
새 값이나 분기를 합집합으로 더하지 않는다. 따라서 닫힌 `enum`, `oneOf`,
`additionalProperties: false`는 확장이 넓힐 수 없는, 되돌릴 수 없는 문이다.

### 열린 열거형 { #open-enumerations }

필드의 값 목록이 앞으로 늘어날 수 있다면(예: `"refunded"` 상태나 새 결제 방식 추가)
**`enum`을 사용하지 않는다**.

대신 표준 `string`을 정의하고, 알 수 없는 값을 무시해야 한다는 요구 사항을 `description`에
문서화하며, `examples`로 현재 예상되는 값을 코드 생성기에 전달한다. 복잡한 "열린 집합"
검증 패턴(예: `anyOf`와 `const`의 조합)은 클라이언트 측 코드 생성기를 자주 혼란스럽게 하고
스키마를 읽기 어렵게 만들므로 피한다.

<!-- ucp:example skip reason="schema authoring example" -->
```json
"cancellation_reason": {
  "type": "string",
  "description": "Reason for order cancellation. Clients MUST tolerate and ignore unknown values.",
  "examples": ["customer_requested", "inventory_shortage", "fraud_suspected"]
}
```

### 닫힌 열거형 { #closed-enumerations }

엄격한 `enum`이나 `const`는 영구적으로 고정된 도메인이거나 알 수 없는 값을 본질적으로
지원할 수 없는 경우에만 사용한다. 새 값을 추가하면 연동 개발자가 반드시 코드를 수정해야
하는 경우(예: 프로토콜 버전, 엄격한 타입 판별자, 요일)를 위해 남겨 둔다.

<!-- ucp:example skip reason="schema authoring example" -->
```json
"status": {
  "type": "string",
  "enum": ["open", "completed", "expired"],
  "description": "Lifecycle state. This domain is strictly bounded; unknown states represent a breakdown in the state machine and MUST be rejected."
}
```

### 변형(`oneOf`) { #variants-oneof }

`oneOf`는 합 타입(*여러 형태 중 하나*인 필드)을 모델링하지만 **닫혀 있으므로** `enum`과
같은 되돌릴 수 없는 문 문제가 있다. 확장은 분기를 추가할 수 없고(두 `oneOf`의 `allOf`는
분기의 교집합을 만들 뿐 합집합을 만들지 않는다), 열린 객체(`additionalProperties: true`)에서는
두 분기와 일치하는 페이로드가 `oneOf` 검증에 실패한다.

늘어날 수 있는 변형 집합은 다음 두 방식 중 하나로 열린 형태로 모델링한다.

- **열린 판별자** — 활성 형태를 가리키는 열린 `type`/`kind` 문자열. 열린 열거형과
  똑같이, 읽는 쪽이 알 수 없는 값을 관대하게 처리한다.
- 우선순위가 문서화된 단일 **열린 객체** — 활성 형태가 태그가 아니라 어떤 필드가 있는지로
  드러나는 경우, 둘 이상이 나타나면 어느 쪽이 우선하는지를 명시된 규칙으로 정한다.

`oneOf`는 영구적으로 고정된 변형 집합에만 사용한다. 닫힌 `enum`과 같은 기준이다.

<!-- ucp:example skip reason="schema authoring example" -->
```json
// PREFER: an open `kind` discriminator — a new variant is a new value; readers tolerate unknowns
"target": {
  "type": "object",
  "required": ["kind"],
  "properties": {
    "kind": {
      "type": "string",
      "description": "Delivery target. Well-known: `email`, `sms`. Unknown values MUST be tolerated.",
      "examples": ["email", "sms"]
    }
  }
}

// AVOID: oneOf — adding a `push` variant means adding a branch, which an extension cannot do
"target": {
  "oneOf": [
    { "required": ["email_address"], "properties": { "email_address": { "type": "string" } } },
    { "required": ["phone_number"],  "properties": { "phone_number":  { "type": "string" } } }
  ]
}
```

### 열린 객체(`additionalProperties`) { #open-objects-additionalproperties }

객체를 미리 닫아 두면 앞으로 스키마에 호환성을 깨지 않는 추가를 할 수 없게 된다. 분산
프로토콜에서는 원래라면 하위 호환되는 필드 추가(예: 주문에 "gift_message" 필드 추가)도
닫힌 스키마로 검증하는 클라이언트에게는 호환성을 깨는 변경이 된다.

JSON Schema는 기본적으로 열려 있어 알 수 없는 속성을 무시한다. 작성자는 드문 경우를 제외하고
이 키워드를 생략해야 한다. 드문 경우란 다형성 판별자(엄격함이 oneOf 검증의 모호성을
막는 경우)나 보안이 중요한 페이로드(알 수 없는 필드가 변조를 의미할 수 있는 경우)다.
`ucp` 프로토콜 네임스페이스 자체도 의도적으로 열려 있다(관대한 읽는 쪽은 인식하지 못하는
멤버를 무시한다). 그 안에서 오타를 막는 것은 작성 시점의 관심사이지 와이어 검증의 관심사가
아니다.

**안티패턴(버전 되돌림 없이는 새 필드를 추가할 수 없게 만든다):**

<!-- ucp:example skip reason="schema authoring example" -->
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

UCP 스키마는 기본적으로 객체 필드에 `minProperties`나 `maxProperties`를 설정하지 않는다.

- **`maxProperties`** — 한도는 구현자에게 맡긴다. 특정 한도를 정하려면 판단이 필요하고
  그 판단은 결국 예외에 부딪히므로 프로토콜은 상한을 정의하지 않는다. 구현자는 자체 제약을
  두고, 디버깅과 올바른 동작을 돕도록 명확한 오류 피드백을 제공할 것을 권한다.
- **`minProperties`** — 빈 객체(`{}`)는 형식이 올바르며 해가 없다. 구현자는 이를 받아들여
  아무 동작도 하지 않는 것(no-op)으로 처리해야 한다.

## `request_constraints` 프로토콜 멤버 { #the-request_constraints-protocol-member }

규범적 처리, 범위, 라이프사이클, 잘못된 멤버에 대한 동작은
[요청 제약](../specification/overview/index.md#request-constraints)에 정의되어 있다. 이
섹션은 스키마 작성과 관련된 경계만 다룬다.

### 로컬 구조 { #local-structure }

`request_constraints`는 `ucp.json#/$defs/members`에 `ucp_request: "omit"`인 응답 전용
프로토콜 멤버로 중앙 등록되어 있다. 기능, 확장, 핸들러, 공통 타입 스키마는 이를 다시
선언하지 않는다. 준수하는 UCP 인식 리졸버는 등록된 멤버를 적격한 응답 범위에 구체화하므로,
일반적인 해석된 스키마 검증에서 공통 스키마가 적용된다.

`schemas/common/types/constraint_expression.json`은 닫힌 제약 표현식(Constraint Expression)
문법을 정의한다. `schemas/common/types/request_constraints.json`은 그 문법을 다음 UCP
요청의 데이터에 결합하며, 등록된 멤버가 참조하는 스키마다.

| 위치 | 허용되는 멤버 | 형태 |
| :-- | :-- | :-- |
| 객체 제약 | `required`, `properties`, `anyOf` | `required`는 고유한 필드 이름을 담는다. `properties`는 필드 이름을 객체 제약 또는 값 제약에 매핑한다. `anyOf`는 비어 있지 않은 객체 제약으로 이루어진 비어 있지 않은 배열이다. 빈 객체는 `anyOf` 분기로 쓰일 때를 제외하면 no-op이다. |
| 값 제약 | `enum`, `const` | `enum`은 비어 있지 않고 값이 고유하다. 최소 하나의 멤버가 있어야 하며, 둘 다 있으면 둘 다 적용된다. |

나열된 멤버가 문법을 정의한다. `request_constraints` 값은 최상위에 `path`를 추가로 담을
수 있다.

문법 변경은 `constraint_expression.json`에서 이루어지며, 루트 객체 제약에 대한 변경은
`request_constraints.json`에도 반영해야 한다.

### 주석과 적용 범위 { #annotations-and-applicability }

`constraint_expression.json`에는 `ucp_request` 주석이 없다. 그 속성은 독립적으로 배치할 수
있는 UCP 필드가 아니라 제약 표현식 구문이므로, 문법 자체에는 방향이 없다.

적용 범위는 그것을 담는 UCP 속성에 속한다. 제약 표현식을 노출하는 각 UCP 속성은 자신에게
필요한 방향별 적용 범위 주석을 가지며, 그 주석만으로 문법이 언제 도달 가능한지가 결정된다.
`request_constraints`의 경우 담는 속성은 `ucp.json#/$defs/members`에 등록된 멤버다. 이
멤버는 `ucp_request: "omit"`으로 유지되며, 응답 전용이라는 가시성이 선언되는 유일한 곳이다.
문법이 방향과 무관하므로 같은 파일을 요청에 적용되는 속성과 응답 전용 속성 모두에서 변경
없이 참조할 수 있다.

### 제약 표현식 문법 { #constraint-expression-grammar }

바깥쪽 `path` 멤버를 제외하면, 내보내는 값과 그 안에 중첩된 제약 객체는 구조화된 UCP 도메인
객체가 아니라 JSON Schema Draft 2020-12 제약 표현식 구문이다. 그 안에 암묵적 `ucp`를
추가하거나 구체화하지 않는다. `properties` 맵의 키는 선택된 요청 객체의 필드를 가리키는
데이터이며, 암묵적 멤버가 아니다.

이를 담는 `ucp` 객체는 여전히 [예약된 `ucp` 멤버](#the-reserved-ucp-member)를 따른다. 닫힌
구조화 객체는 `request_constraints`를 직접 선언하는 것이 아니라
`ucp.json#/$defs/members`에 대한 선택적 참조를 통해 그 컨테이너를 허용한다.

## 확장이 선언하는 Action 유형 { #extension-declared-action-types }

[Actions](../specification/overview/index.md#actions)에 정의된 대로, 모든 Action 유형은
확장이 선언하며 해당 확장이 협상되었을 때만 사용할 수 있다. 지원을 알리기 전에 비즈니스와
플랫폼 모두 확장의 Action 규약 전체를 검토해야 한다. 협상은 그 규약의 의미와 지원에 대한
런타임 이전의 합의일 뿐, 앞으로 올 모든 `config` 값이나 위임 대상을 미리 승인하는 것이
아니다. 각 구체적인 인스턴스는 여전히 합성된 스키마, Action 유형 규약의 런타임 규칙, 플랫폼
정책을 준수해야 한다.

Action 유형에 해당하는 경우, 확장 작성자는 다음을 정의해야 한다.

- Action 유형 키, 상위 기능, 발생 조건
- Action 유형이 통제하는 정확한 효과
- 구체적인 `config` 스키마와 플랫폼의 처리 방식. 실행 가능하거나 콘텐츠를 로드하게 하는
  필드는 명시적으로 식별한다
- 관련 스킴, 출처(origin), 위임 대상, 신뢰 앵커
- 표시, 격리, 권한 통제
- 비즈니스가 Action 완료를 어떻게 관찰하고 이후의 상위 응답에 반영하는지
- 상위 응답이 지연될 수 있다면 타임아웃과 제한된 백오프 규칙, 그리고 플랫폼이 같은
  Action을 안전하게 다시 처리할 수 있는지. 가능하다면 안전한 재시도 조건 전부
- 처리가 실패·거절·중단되었을 때, Action이 지원되지 않거나 만료되었을 때, 또는
  비즈니스가 상위 응답을 갱신하지 않았을 때의 대체 동작
- 작업이 해결·대체·교체되는 Action별 조건, 그리고 필요한 경우 같은 유형 내 또는 유형 간
  순서 의미

확장 작성자는 구체적인 처리 모델에 관련된 통제만 정의해야 한다. 예를 들어 로드할 URL이
없는 Action에는 URL 스킴이나 출처 규칙이 필요 없다. 확장 작성자는 Action별 데이터를
`config` 아래에 두고, 기존의 `allOf` 확장 합성을 사용해 각 Action 키와 `config` 형태를 상위
기능 스키마에 추가해야 한다. 독립적인 Actions 기능이나 레지스트리를 도입해서는 안 된다.
실행기, 콜백/결과 모델, 상태 기계, 대체 동작 enum, 재시도 필드, 폴링 프로토콜 같은 범용
장치로 공통 Actions 규약을 확장해서도 안 된다. 구체적인 확장은 그 Action 유형에 정말
필요할 때 자체적인 콜백, 결과, 상태, 대체 동작, 재시도, 타임아웃, 백오프 의미를 정의할 수
있다.

## 전체 예시: 기능 스키마 { #complete-example-capability-schema }

기능 스키마는 페이로드 구조와 선언 변형을 모두 정의한다.

<!-- ucp:example skip reason="schema authoring example" -->
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
    "totals": {"$ref": "../common/types/totals.json"},
    "links": {"type": "array", "items": {"$ref": "../common/types/link.json"}}
  }
}
```

핵심 사항:

- **최상위 `name`과 `version`**이 스키마를 자기 서술적으로 만든다
- **`$defs` 변형**으로 서로 다른 컨텍스트에서 검증할 수 있다
- **페이로드 속성**이 실제 체크아웃 응답 구조를 정의한다

## JSON 예시 문서화 { #documenting-json-examples }

UCP 명세 문서는 기계적으로 검증된다. 모든 ` ```json ` 블록은 명세가 정의한 스키마로
검사되거나, 검증 범위 밖이라고 명시적으로 표시된다. 스키마와 문서가 어긋나면 독자를
조용히 오도하는 대신 CI가 실패한다.

이를 위해 UCP 예시는 **전용 JSON 기능 집합**을 사용한다. 엄격한 JSON에 작고 고정된 작성
편의 기능을 더한 것이다. 검증기는 스키마로 검증하기 전에 이 편의 기능을 정규 JSON으로
변환한다. 작성자는 확장된 JSON을 쓰지만, 와이어 형식은 여전히 엄격한 JSON이다.

### 주석 규약 { #the-annotation-contract }

명세의 모든 ` ```json ` 블록 앞에는 반드시 주석이 있어야 한다(**MUST**). 주석이 없는
블록은 CI에서 실패한다.

<!-- ucp:example skip reason="example validation guide" -->
```json
<!-- ucp:example schema=shopping/checkout op=read -->
{ ... }
```

#### 주석 문법 { #annotation-grammar }

```text
<!-- ucp:example schema=PATH [op=OP] [direction=DIR] [extract=JSONPATH] [target=JSONPATH] [def=NAME] -->
<!-- ucp:example skip reason="..." -->
```

| 속성 | 필수 | 기본값 | 용도 |
| ------------- | ----------------- | ---------- | -------------------------------------------------------------------------- |
| `schema`      | 예(skip이 아닌 경우) | — | 검증에 사용할 스키마. 예: `shopping/checkout` |
| `op`          | 아니요 | `read` | 작업: `create`, `read`, `update`, `complete`, `cancel` 등 |
| `direction`   | 아니요 | `response` | `request` 또는 `response` |
| `extract`     | 아니요 | `$` | 표시된 블록 내부의 JSONPath. 선택된 하위 트리가 예시가 된다 |
| `target`      | 아니요 | `$` | 스키마/스캐폴드 내부의 JSONPath. 예시가 하위 트리를 대체한다 |
| `def`         | 아니요 | — | 스키마에서 `$defs/<name>`을 꺼내 그것으로 검증한다 |
| `skip reason` | 예(skip인 경우) | — | 이 블록을 검증할 수 없는 이유를 설명하는 자유 형식 문장 |

#### 배치 규칙 { #placement-rules }

- 주석은 반드시 ` ```json ` 펜스 앞의 독립된 줄에 있어야 한다(**MUST**). 사이에 빈 줄이
  있어도 된다.
- 블록당 주석은 하나다. **여러 주석을 겹쳐 쓰면 거부된다.**
<!-- cspell:ignore shema -->
- **알 수 없는 속성 이름은 거부된다.** `shema=` 같은 오타는 속성을 조용히 무시하는 대신
  CI를 실패시킨다.

### 작성 편의 기능 { #authoring-conveniences }

검증기는 엄격한 JSON 외에 다음 기능을 허용한다. 명확성에 도움이 될 때 사용하고, 그 외에는
엄격한 JSON을 기본으로 한다.

#### 줄 주석 { #line-comments }

`//`부터 줄 끝까지. 검증 전에 제거된다.

<!-- ucp:example skip reason="example validation guide" -->
```json
{
  "currency": "USD",       // ISO 4217
  "amount": 5000           // minor units (cents)
}
```

블록 주석(`/* */`)은 지원하지 **않는다**. 여러 줄 메모가 필요하면 `//` 줄을 여러 개
사용한다.

**제한 사항:** `//` 제거기는 줄 단위로 문자열 경계를 추적하므로 근사적이다. 이스케이프된
백슬래시 뒤에 `//`가 오는 문자열 리터럴이 포함된 예시는 잘못 파싱된다. 현재 문서 중 이에
해당하는 예시는 없다. 이 문제를 만나면 예시 구조를 바꾼다.

#### 템플릿 변수 { #template-variable }

정확히 하나의 변수만 치환된다. `{{ ucp_version }}`은 유효한 날짜 값이 된다. 다른 템플릿
변수는 인식되지 않으며, 그 밖의 `{{ name }}`은 그대로 JSON 파싱 단계까지 남아 실패한다.

#### HTTP 엔벌로프 { #http-envelope }

비어 있지 않은 첫 줄이 HTTP 요청 줄(`GET|POST|PUT|PATCH|DELETE`)이나 상태 줄(`HTTP/`)과
일치하면, 검증기는 첫 빈 줄 다음의 JSON 본문을 추출한다. 그 사이의 헤더는 무시된다.

<!-- ucp:example skip reason="example validation guide" -->
```json
POST /checkout-sessions HTTP/1.1
Host: api.example.com
Content-Type: application/json

{ "line_items": [ ... ] }
```

다른 HTTP 메서드(`OPTIONS`, `HEAD`, `CONNECT`, `TRACE`)는 엔벌로프로 인식되지 않는다.
JSON으로 파싱되어 실패한다.

#### 엔벌로프에서 추출하기 { #extracting-from-envelopes }

표시된 JSON 블록이 전송 객체나 래퍼 객체이고 검증할 UCP 페이로드가 그 안에 중첩되어 있을
때 `extract=`를 사용한다. `extract=`는 표시된 예시에서 값을 읽고, `target=`은 추출한 값을
검증 스캐폴드에 써 넣는다.

```text
<!-- ucp:example schema=shopping/checkout op=create direction=request extract=$.params.arguments.checkout -->
```

```text
<!-- ucp:example schema=shopping/checkout extract=$.result.structuredContent.totals target=$.totals -->
```

첫 번째 예시는 중첩된 체크아웃 요청을 검증한다. 두 번째 예시는 표시된 엔벌로프에서
`totals` 조각을 추출해 체크아웃 스캐폴드의 `$.totals`에 삽입한 다음, 병합된 체크아웃을
검증한다.

#### 생략 표식 { #elision-markers }

검증기는 **"이 필수 필드는 존재하지만 그 값은 단정하지 않는다"**는 의미의 형태를 이해한다.
커버리지 검사는 여전히 해당 필드가 언급되었는지 확인한다. 생략된 하위 트리에서 발생하는
스키마 검증 오류는 억제된다.

| 형태 | 의미 |
| -------------------- | ------------------------------------------------------ |
| `"..."`              | 필드 값이 생략됨 |
| `[ ... ]`            | 비어 있지 않은 배열. 내용 생략 |
| `{ ... }`            | 비어 있지 않은 객체. 내용 생략 |
| `[ "..." ]`          | `[ ... ]`와 동일 |
| `{ "...": "..." }`   | `{ ... }`와 동일 |

<!-- ucp:example skip reason="example validation guide" -->
```json
{
  "ucp": { ... },
  "id": "chk_abc",
  "currency": "USD",
  "line_items": [ ... ],
  "totals": [ ... ]
}
```

따옴표 없는 형태인 `[ ... ]`와 `{ ... }`가 컨테이너 내용을 생략하는 정규 방식이다. 이
형태는 *멤버가 존재하지만 표시되지 않는 비어 있지 않은 컨테이너*라는 올바른 의미를
전달한다. 문자열 센티널 형태(`["..."]`, `{"...": "..."}`)도 파서 편의를 위해 허용되지만,
문자 그대로 보면 *문자열 하나를 담은 배열*이나 *키 하나를 가진 객체*를 나타내므로 미묘하게
틀린 의미가 된다. 새 예시에서는 따옴표 없는 형태를 사용한다.

**제한 사항:**

- 따옴표 없는 `...`는 배열이나 객체의 **유일한 내용**일 때만 인식된다. `[a, ..., b]`처럼
  중간에 들어간 형태는 지원하지 않는다.
- 일부만 생략하려면(일부 항목은 표시하고 일부는 생략) 생략할 위치에 문자열 형태 `"..."`를
  사용한다: `[1, "...", 3]`.
- 세 글자 리터럴 문자열 `"..."`은 생략 센티널로 예약되어 있으므로 예시에서 실제 데이터로
  쓸 수 없다. 정말 필요하다면 유니코드 이스케이프(`"\u002e\u002e\u002e"`)를 사용한다.

### 지원하지 않는 것 { #what-is-not-supported }

- `}`나 `]` 앞의 **후행 쉼표**. 엄격한 JSON만 허용한다. 와이어 형식이 엄격하므로 명세도
  그에 맞게 정직하게 유지한다.
- **블록 주석** `/* */`.
- **JSON5 기능**: 작은따옴표 문자열, 따옴표 없는 키, 16진수 리터럴, `NaN` / `Infinity`,
  여러 줄 문자열.
- `{{ ucp_version }}` 외의 **여러 템플릿 변수**.
- **중간에 들어간 따옴표 없는 생략 부호** `[a, ..., b]`.

### skip 사유 { #skip-reasons }

블록을 검증할 수 없을 때는 정확한 사유와 함께 `skip`을 사용한다. skip 사유는 CI에서
grep으로 찾을 수 있으며, 아직 검증되지 않은 부분을 추적하는 데 쓰인다.

확립된 분류는 다음과 같다. 필요하면 늘려도 되지만 구체적으로 작성한다.

- `"JSON-RPC transport binding"` — JSON-RPC 엔벌로프로 감싼 경우
- `"embedded protocol binding"` — Embedded Protocol 전송 래퍼
- `"A2A transport binding"` — A2A 전송 래퍼
- `"profile document, no wrapper schema"` — 감싸는 엔티티가 없는 최상위 `ucp`
  블록
- `"schema authoring example"` — UCP 페이로드가 아닌 JSON Schema 조각
- `"handler config example"` / `"handler schema definition"` — 결제 핸들러
  내부 구조
- `"capability declaration fragment"` — 기능 레지스트리 일부
- `"OAuth metadata, not UCP payload"` — 서드파티 프로토콜 페이로드
- `"cryptographic material, not UCP payload"` — 키, 서명
- `"<feature> fragment"` — 중첩된 필드 하나만 보여 주는 불완전한 객체

`"conceptual example"` 같은 모호한 사유는 피한다. 이 분류 체계를 바탕으로 다음에 무엇을
검증할지 우선순위를 정한다.

### 일반적인 패턴 { #common-patterns }

**전체 요청 또는 응답.** 기본 경우다. 예시는 해당 작업의 완전한 페이로드다.

```text
<!-- ucp:example schema=shopping/cart op=create direction=request -->
```

**주변 컨텍스트가 있는 하위 트리.** 예시가 하나의 필드에 집중할 때 `target=`을
사용한다. 예시는 유효함이 확인된 스캐폴드의 해당 대상 경로에 끼워 넣어지고, 나머지는
스캐폴드의 기본값을 사용한다.

```text
<!-- ucp:example schema=shopping/checkout target=$.totals -->
```

**중첩된 페이로드가 있는 표시된 엔벌로프.** 코드 블록은 엔벌로프를 보여 주지만 검증할
UCP 페이로드는 하위 트리일 때 `extract=`를 사용한다.

```text
<!-- ucp:example schema=shopping/checkout op=create direction=request extract=$.params.arguments.checkout -->
```

**`$defs`가 있는 스키마.** 일부 스키마는 `$defs` 아래에 여러 메시지 형태를 담는다.
기능의 요청과 응답이 서로 다른 객체라면(예: 카탈로그에서 검색 요청은 질의이고 검색 응답은
상품 목록이다) 작업과 방향만 지정하면 된다. 검증기가 `$defs/{op}_{direction}`을 자동으로
선택한다.

```text
<!-- ucp:example schema=shopping/catalog_search op=search direction=response extract=$.result.structuredContent -->
```

작업+방향으로 표현되지 않는 형태(전송 방식의 `error_response`, 프로필의
`business_schema`, 이름 있는 하위 타입)는 `def=`로 명시적으로 선택한다.

```text
<!-- ucp:example schema=transports/jsonrpc def=error_response op=read -->
```

**빈 본문.** `{}` 페이로드(예: cancel, GET)는 해당 op + direction에 대해 당연히 통과한다.
특별한 구문이 필요 없다.

**확장 필드.** 기능 스키마는 열려 있으므로, 확장만 정의하는 필드(예: 체크아웃의
`fulfillment`나 `discounts`)는 기능 스키마로 검증할 때 확인되지 않은 채 통과한다. 예시에
그런 필드가 있으면 검증기는 그 확장이 기능을 합성한 스키마(예:
`shopping/fulfillment.json`의 `$defs/dev.ucp.shopping.checkout`)로도 검증한다. 별도 주석은
필요 없다. 기능에 대한 주석을 그대로 두면 확장 필드도 함께 검사된다. `payment`나 `buyer`처럼
기능 자체가 선언하는 필드는 기능 스키마로 검증되며 이 동작을 일으키지 않는다.

### 검증기 연결 정보는 드러내지 않는다 { #keep-validator-wiring-invisible }

검증 규약은 저장소 인프라(주석, 스캐폴드, 스키마 파일 경로)다. 렌더링된 명세의 독자는
프로토콜 설명과 JSON 예시만 보며, 연결 정보는 보지 않는다.

이것이 가능한 이유는 다음과 같다.

- 주석은 렌더링되지 않는 HTML 주석(`<!-- ucp:example ... -->`)에 들어 있다.
- 스캐폴드는 `scripts/scaffolds/` 아래에 있다.
- 검증기 스키마는 `source/schemas/` 아래(엔벌로프 스키마는
  `source/schemas/transports/`)에 있다.

JSON 예시를 추가할 때 검증기가 올바른 스키마를 가리키게 하는 것은 **본문 작업이 아니라
주석 작업이다.** 주석이 이미 스키마를 지정하고 검증기가 이미 그 범위를 강제한다. *"이
바인딩은 `transports/X.json`이 스키마로 정의하며, A는 검증하지만 B는 검증하지 않는다"* 같은
문장은 주석 내용을 중복하고 검증기 내부 사항을 독자용 페이지에 흘린다.

바인딩의 범위에 미리 짚어 둘 만한 실제 혼동이 있다면(예: *"UCP의 A2A 바인딩은 A2A
프로토콜을 재정의하지 않는다"*), 스키마 커버리지 메모가 아니라 **프로토콜 관점**에서
설명한다. 프로토콜 측면의 우려는 실재하지만, 파일 경로는 그와 무관하다.

### 작성자가 하지 않는 것 { #what-authors-dont-do }

- **버그를 숨기는 skip 사유를 지어내지 않는다.** 예시가 틀려서 검증에 실패한다면 예시를
  고친다.
- **주석(코드 주석)에 검증 지시문을 넣지 않는다.** 코드 주석은 사람 독자를 위한 문서이며
  검증기가 해석하지 않는다.
- **지원하지 않는 구문을 사용하지 않는다.** 위의 "지원하지 않는 것" 목록은 빠짐없는
  목록이다. 무언가를 추가하려면 파서를 억지로 늘리는 것이 아니라 규약과 검증기를 함께
  갱신해야 한다.
- **` ```json ` 블록을 중첩하지 않으며**, 마크다운 파서가 놓칠 수 있는 들여쓰기 컨텍스트에
  주석을 두지 않는다.

### 로컬에서 검증기 실행하기 { #running-the-validator-locally }

검증기는 표준 라이브러리만 사용하는 Python이며, 스키마 해석과 페이로드 검증은
[`ucp-schema`](https://github.com/universal-commerce-protocol/ucp-schema) 바이너리를
호출해 수행한다. 최초 설정은 다음과 같다.

```bash
uv sync                                   # Python deps
cargo install ucp-schema                  # validator backend
uv tool install pre-commit                # if not already installed
pre-commit install --hook-type pre-commit --hook-type pre-push
```

`--hook-type pre-push` 플래그가 중요하다. pre-commit은 기본적으로 `pre-commit` 단계
훅만 설치하지만, 이 저장소는 안전망으로 `pre-push` 훅도 사용한다. 두 플래그를 모두 전달해야
전체 검사가 적용된다.

수동 실행:

```bash
python3 scripts/validate_examples.py --schema-base source/schemas/
python3 scripts/validate_examples.py --schema-base source/schemas/ --file docs/specification/shopping/checkout/rest.md docs/specification/shopping/cart/index.md
python3 scripts/validate_examples.py --schema-base source/schemas/ --audit
```

`--audit` 모드는 블록을 검증하지 않고 나열만 한다. skip 개수를 세거나 주석이 없는 블록을
찾을 때 유용하다. `--file`은 점진적 검증을 위해 하나 이상의 경로를 받는다.

#### 자동으로 실행되는 것 { #what-runs-automatically }

위에서 말한 "스키마와 문서가 어긋나면 CI가 실패한다"는 원칙은 세 곳에서 강제된다.

| 위치 | 범위 | 시점 |
| --- | --- | --- |
| `pre-commit` 단계 훅 | 변경된 `docs/*.md` 파일만 | 모든 `git commit`(설치한 경우) |
| `pre-commit` 단계 훅 | 전체 문서 | `source/schemas/`나 검증기 자체를 건드리는 모든 `git commit` |
| `pre-push` 단계 훅 | pre-commit과 동일 | 모든 `git push` — `--no-verify` 우회를 잡아낸다 |
| CI(`.github/workflows/docs.yml`) | 전체 문서 | 모든 PR — 필수 최종 방어선 |

pre-commit 훅은 선택 사항이며(위의 설치 명령이 필요하다) CI는 무조건 실행된다. 로컬 훅을
건너뛰어도 문제가 생기지는 않는다. 주석이 없는 블록이나 검증 실패가 있는 PR은 CI에서
실패하기 때문이다. 다만 로컬 훅을 쓰면 GitHub Actions 실행을 기다리는 것보다 더 일찍
피드백을 받을 수 있다.

#### 전체 문서 검사가 실행되는 시점(과 그 이유) { #when-the-full-corpus-check-fires-and-why }

pre-commit/pre-push에서 "변경된 파일만"과 "전체 문서"를 나눈 것은 의도된 것이다.

- **문서 편집**(`docs/*.md`)은 변경된 파일만 검증한다. 편집 중인 파일의 직접적인
  오류(주석이 없는 블록, 잘못된 스키마 이름, 깨진 예시 페이로드)를 빠르게 잡아낸다.
- **스키마나 검증기 코드 편집**은 전체 문서 검사를 실행한다.
  `source/schemas/shopping/cart.json`(또는 `validate_examples.py` 자체)을 한 번만
  바꿔도 관련 없어 보이는 여러 문서의 예시가 무효화될 수 있다. 전체 검사는 그런 파일 간
  회귀를 CI에 도달하기 전에 로컬에서 잡을 수 있는 유일한 방법이다.
