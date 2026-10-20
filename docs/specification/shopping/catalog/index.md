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

# 카탈로그 기능 { #catalog-capability }

## 개요 { #overview }

카탈로그 기능(Capability)을 사용하면 플랫폼이 비즈니스의 상품 카탈로그를 검색하고
탐색할 수 있습니다. 체크아웃 전에 상품을 찾는 단계로, 다음과 같은 사용 사례를
지원합니다.

* 자유 텍스트 상품 검색
* 카테고리 및 필터 기반 탐색
* 식별자로 상품/변형 일괄 조회
* 변형 간 가격 비교

## 기능 { #capabilities }

| 기능 | 설명 |
| :--- | :--- |
| [`dev.ucp.shopping.catalog.search`](search.md) | 쿼리 텍스트와 필터로 상품을 검색합니다. |
| [`dev.ucp.shopping.catalog.lookup`](lookup.md) | 식별자로 상품 또는 변형을 조회합니다. |

## 핵심 개념 { #key-concepts }

* **상품(Product)**: 제목, 설명, 미디어와 하나 이상의 변형으로 구성된 카탈로그
  항목입니다.
* **변형(Variant)**: 특정 옵션 선택(예: "Blue / Large"), 가격, 재고 상태가 정해진
  구매 가능한 항목입니다.
* **가격(Price)**: 가격 값에는 금액(통화의 최소 단위)과 통화 코드가 모두 들어 있어
  다중 통화 카탈로그를 지원합니다.
* **판매 기준(Sale basis)**: 수량을 표시하는 방식입니다. 개수 단위(`each`, 기본값)이거나
  무게, 길이, 면적, 부피, 시간 같은 측정 단위입니다.
  [수량과 단위](../../overview/index.md#quantities-and-units)를 참고하세요.

### 체크아웃과의 관계 { #relationship-to-checkout }

카탈로그 작업은 체크아웃의 `line_items[].item.id`에 바로 쓸 수 있는 상품 ID와 변형
ID를 반환합니다. 카탈로그 조회로 얻은 변형 ID는 체크아웃이 기대하는 아이템 ID와
일치해야 합니다.

카탈로그 응답(가격, 재고 상태 등)은 해당 요청 시점의 비즈니스 조건을 반영하지만
거래상의 확약은 아닙니다. 권위는 체크아웃에 있습니다. 응답은 세션마다 다를 수 있으므로,
재검증 없이 다른 세션에서 재사용하지 않는 것이 좋습니다(**SHOULD NOT**).

## 판매 기준과 수량 단위 { #sale-basis-and-quantity-units }

`variants[].quantity_unit`은 거래 전에 변형의 판매 기준을 알려 줍니다. 이 서술자는 공통
[수량과 단위](../../overview/index.md#quantities-and-units) 계약을 따릅니다. 서술자가
없으면 기본값인 `each` 기준이라는 뜻이며, `each`가 아닌 기준은 비즈니스가 서술자를
포함해 알립니다.

플랫폼은 거래 전에 카탈로그에서 판매 기준을 파악합니다. `unit`과 `scale`은 수량을
표시하는 방식을 정의하고, 선택 사항인
[`increment`](../../overview/index.md#ordering-increment)는 비즈니스가 판매하는 주문
단위를 알려 줍니다. 플랫폼은 이를 바탕으로 수량 조절 UI를 만들고 제출 전에 입력을
검증할 수 있습니다. 바나나를 파운드 단위로, 1/4파운드 배수로 판매하는 비즈니스라면
`{ "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }`를 알립니다.

판매 기준이 물리적 측정에만 국한되지는 않습니다. 분 단위 주차, 시간 단위 인건비 같은
계량형 상품도 같은 계약에 따라 같은 서술자(`MIN`, `HUR`)를 사용합니다.

`variants[].id`는 구매 가능한 변형을 식별하고, `quantity_unit`은 그 변형을 주문할 때
쓰는 단위와 세분성을 정의합니다. `quantity_unit`이
`{ "unit": "LBR", "scale": 2, "display_text": "lb" }`인 바나나는 1/100파운드 단위로
판매되므로, `quantity`가 `150`이면 1.50 lb입니다.

### 단가와의 구분 { #distinction-from-unit-price }

`quantity_unit`과 변형의 [`unit_price`](#variant)는 서로 다른 질문에 답하며, 각각
독립적으로 설정합니다.

* `quantity_unit`은 **판매 기준**입니다. `quantity`를 세는 단위이자 `price`를 매기는
  단위입니다.
* `unit_price`는 **표시용 비교 지표**입니다. 매대에서처럼 비교할 수 있도록 파생한
  "표준 측정량당 가격"(예: 100 mL당)입니다. `measure`는 변형의 포장 또는 내용량이고,
  `reference`는 비교 기준이 되는 분모입니다.

`quantity_unit`, `unit_price.measure`, `unit_price.reference`의 단위 필드는 공통
[수량과 단위](../../overview/index.md#quantities-and-units) 계약을 따릅니다. 두 단가
필드는 공통 측정 타입을 사용하므로 정수 `value` 필드는 단계 수입니다. 비즈니스는 변형에
`quantity_unit`이나 `unit_price` 중 하나만 제공하거나, 둘 다 제공하거나, 둘 다 제공하지
않을 수 있습니다(**MAY**).

표시용 비교 지표가 의미를 잃지 않도록, 비즈니스는 반드시
`unit_price.measure.value`와 `unit_price.reference.value` 모두에 양의 정수를 사용해야
합니다(**MUST**). 비즈니스는 반드시 `unit_price.currency`를 `price.currency`와 같게
설정해야 합니다(**MUST**). `unit_price` 안에서 비즈니스는 반드시 `measure.unit`과
`reference.unit`에 같은 값을 사용해야 합니다(**MUST**). 두 측정값의 `scale` 값은 서로
달라도 됩니다(**MAY**). 비즈니스는 단가를 계산하면서 단위 간 변환이나 통화 변환을 해서는
안 됩니다(**MUST NOT**). 각 측정값은 공통 단위 기준으로 정수 `value × 10^-scale`를
나타냅니다. 비즈니스는 반드시 `price.amount`와 이 스케일 적용 값으로 비교 지표를
계산해야 합니다(**MUST**).

```text
(price.amount / (measure.value × 10^-measure.scale)) × (reference.value × 10^-reference.scale)
```

비즈니스는 반드시 가격 규칙에 따라 결과를 통화의 최소 단위로 한 번 반올림해
`unit_price.amount`로 반환해야 합니다(**MUST**). 반환된 `unit_price.amount`가 권위를
가집니다. 플랫폼은 이 값을 다시 계산하거나 자체 결과로 대체해서는 안 됩니다(**MUST NOT**).

동일 단위·동일 통화 규칙은 의미상의 불변 조건입니다. JSON Schema는 해당 필드를 각각
독립적으로 검증할 뿐, 두 동일성 중 어느 것도 강제하지 않습니다.

예를 들어 `each`로 판매하는 50 m 케이블 스풀은 `quantity_unit`을 생략하고 미터당
`unit_price`를 둘 수 있습니다. 이때 `measure`는
`{ "value": 5000, "unit": "MTR", "scale": 2, "display_text": "m" }`, `reference`는
`{ "value": 1, "unit": "MTR", "display_text": "m" }`가 될 수 있습니다. 두 측정값 모두 `MTR`을
사용하며, 단위 간 변환 없이 각각 50 m와 1 m를 나타냅니다.

거래 라인에서는 필드가 있느냐 없느냐로 역할이 드러납니다. 가격 기준이 판매 기준과 다른
라인에는 비즈니스가 반드시 `unit_price`를 빠짐없이 담아 보내야 합니다(**MUST**)([체크아웃 — 가격
기준](../checkout/index.md#quantity-and-sale-basis) 참고). 이 스풀의 미터당 가격처럼
카탈로그에만 있는 `unit_price`는 표시용 비교 지표이고, 라인 수준의 `unit_price`에는
청구 금액을 계산하는 기준 요율이 담깁니다.

### 예시: 무게 단위로 판매하는 상품 { #example-a-good-sold-by-weight }

파운드 단위로 판매하는 바나나의 `get_product` 응답입니다. 변형은 `quantity_unit`으로
`{ "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }`를, `price`로
`79`를 알립니다. 즉 1파운드당 $0.79이며 0.25 lb 배수로 판매합니다.

<!-- ucp:example schema=shopping/catalog_lookup op=get_product -->
```json
{
  "ucp": { "version": "{{ ucp_version }}" },
  "product": {
    "id": "prod_bananas",
    "title": "Bananas",
    "description": { "plain": "Fresh bananas sold by the pound." },
    "price_range": {
      "min": { "amount": 79, "currency": "USD" },
      "max": { "amount": 79, "currency": "USD" }
    },
    "variants": [
      {
        "id": "var_bananas",
        "title": "Bananas",
        "description": { "plain": "Fresh bananas sold by the pound." },
        "price": { "amount": 79, "currency": "USD" },
        "quantity_unit": { "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 },
        "availability": { "available": true }
      }
    ]
  }
}
```

## 공통 엔터티 { #shared-entities }

### 컨텍스트 { #context }

카탈로그 작업에 쓰는 위치 및 시장 컨텍스트입니다. 모든 필드는 관련성과 현지화를 위한
선택적 힌트입니다. 플랫폼은 요청 헤더로 컨텍스트를 지리적으로 감지할 수 있습니다(MAY).

컨텍스트 신호는 잠정적인 값일 뿐 권위 있는 데이터가 아닙니다. 비즈니스는 검증된
입력(예: 배송 주소)이 없을 때 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은
신호(인증된 계정, 위험 탐지)나 규제상 제약(수출 통제)과 맞지 않으면 무시하거나 순위를
낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래
데이터로 이루어져야 합니다(MUST).

비즈니스는 컨텍스트 신호를 바탕으로 통화를 포함한 시장 배정을 결정합니다. 가격 필터 값은
`context.currency` 기준이며, 표시 통화가 다르면 비즈니스는 적용 전에 변환해야
합니다(SHOULD)([가격 필터](search.md#price-filter) 참고). 응답 가격에는 결정 결과를
확인할 수 있도록 통화 코드가 명시됩니다.

`context.eligibility` 주장이 있고 비즈니스가 이를 수락했다면, 취소선 표시를 위해
`price` / `list_price`를 직접 조정할 수 있으며(**MAY**), `code: "eligibility_benefit"`이
담긴 `messages`로 조정 내용이 어떤 주장에 따른 것인지 밝힐 수 있습니다(**MAY**).

{{ schema_fields('types/context', 'shopping/catalog') }}

### 신호 { #signals }

인가와 악용 방지에 도움이 되도록 플랫폼이 제공하는 환경 데이터입니다. 신호 값은 구매자가
주장한 내용이어서는 안 됩니다(MUST NOT). 자세한 내용과 개인정보 보호 요건은
[신호](../../overview/index.md#signals)를 참고하세요.

{{ schema_fields('types/signals', 'shopping/catalog') }}

### 어트리뷰션 { #attribution }

플랫폼이 제공하는 유입 및 전환 이벤트 컨텍스트로, 플랫폼이 전달하는 캠페인 ID, 클릭
식별자, 소스/매체 표식 등이 해당합니다. 자세한 내용과 동의 요건은
[어트리뷰션](../../overview/index.md#attribution)을 참고하세요.

{{ schema_fields('types/attribution', 'shopping/catalog') }}

### 상품 { #product }

판매 가능한 항목을 나타내는 카탈로그 항목으로, 구매 가능한 변형이 하나 이상 있습니다.

`media`와 `variants`는 순서가 있는 배열입니다. 비즈니스는 가장 관련성 높은 변형과
이미지를 맨 앞에 반환해야 합니다(SHOULD). 조회에서는 기본값을, 검색에서는 쿼리와
컨텍스트에 가장 잘 맞는 항목을 앞에 둡니다. 플랫폼은 첫 번째 요소를 대표 항목으로
취급해야 합니다(SHOULD).

{{ schema_fields('types/product', 'shopping/catalog') }}

### 변형 { #variant }

특정 옵션 선택, 가격, 재고 상태가 정해진 구매 가능한 항목입니다.

조회 응답의 각 변형에는 대응 관계를 파악할 수 있도록 `inputs` 배열이 있습니다. 어떤
요청 식별자가 이 변형으로 해석되었는지, 일치 방식이 `exact`였는지 `featured`(서버가
선택)였는지가 담깁니다. 자세한 내용은 [클라이언트 대응](lookup.md#client-correlation)을
참고하세요.

`media`는 순서가 있는 배열입니다. 비즈니스는 변형의 대표 이미지를 첫 번째 요소로
반환해야 합니다(SHOULD). 플랫폼은 첫 번째 요소를 대표 항목으로 취급해야 합니다(SHOULD).

{{ schema_fields('types/variant', 'shopping/catalog') }}

### 가격 { #price }

{{ schema_fields('types/price', 'shopping/catalog') }}

### 가격 범위 { #price-range }

{{ schema_fields('types/price_range', 'shopping/catalog') }}

### 미디어 { #media }

{{ schema_fields('types/media', 'shopping/catalog') }}

### 상품 옵션 { #product-option }

{{ schema_fields('types/product_option', 'shopping/catalog') }}

### 옵션 값 { #option-value }

{{ schema_fields('types/option_value', 'shopping/catalog') }}

### 선택된 옵션 { #selected-option }

{{ schema_fields('types/selected_option', 'shopping/catalog') }}

### 평점 { #rating }

{{ schema_fields('types/rating', 'shopping/catalog') }}

### 정책 { #policy }

카탈로그 응답의 상품에 적용되는 정책(반품·환불 조건, 보증 등)입니다. `applies_to`의
JSONPath 대상은 응답 루트를 기준으로 합니다. 검색과 일괄 조회에서는 `$.products[N]`,
get_product에서는 `$.product`입니다. 전체 모델은 [정책](../../overview/index.md#policies)을 참고하세요.

{{ schema_fields('types/policy', 'shopping/catalog') }}

## 액션 { #actions }

카탈로그 검색, 일괄 조회, 성공한 Get Product 응답에는 확장이 정의한 액션(Action)이
포함될 수 있습니다. 카탈로그에서 액션은 해당 타입이 정의하는 효과를 막고 있는 미처리
작업으로, 비즈니스가 어떤 상품을 반환하는지나 플랫폼이 상품을 어떻게 다루는지에 영향을
줄 수 있습니다.

검색, 일괄 조회, 성공한 Get Product는 서로 독립된 카탈로그 작업이며, 각 응답이 상위
리소스의 수명을 공유하지 않습니다. 비즈니스는 서로 다른 응답에서 같은 액션 `id`를 사용할
수 있으며(**MAY**), 값이 같다고 해서 같은 작업을 가리키지는 않습니다. 구체적인 액션 타입
계약에서 자신의 인스턴스에 대해 더 강한 대응 관계를 정의할 수는 있습니다(**MAY**).

검색과 일괄 조회에서 비즈니스는 액션 타입 계약과 자체 정책에 따라, 원래라면 관련
있었을 상품을 하나도 반환하지 않을지, 일부만 반환할지, 모두 반환할지 결정합니다.
메시지에서 액션을 가리켜 응답을 설명할 수 있습니다. 성공한 Get Product에는 여전히
`product`가 포함되며, 기존 에러 응답은 바뀌지 않습니다.

액션을 처리한 뒤 플랫폼은 카탈로그 작업을 새로 수행하며, 그 뒤의 비즈니스 응답이
권위를 가집니다. 카탈로그는 액션의 수명 주기, 폴링, 재개 동작을 정의하지 않습니다.
구체적인 액션 타입 계약에서 자신의 인스턴스 처리에 관한 그런 동작을 정의할 수
있습니다(**MAY**). 공통 형태와 규칙은 [개요 — 액션](../../overview/index.md#actions)에
정의되어 있습니다.

예를 들어 다음 검색 응답은 상품을 반환하지 않고, 연령 확인이 결과에 영향을 줄 수
있다고 설명합니다.

<!-- ucp:example schema=shopping/catalog_search op=search -->
```json
{
  "ucp": {...},
  "products": [],
  "actions": {
    "com.example.identity.age_verification": [
      {
        "id": "age-check-1"
      }
    ]
  },
  "messages": [
    {
      "type": "info",
      "code": "age_verification_required",
      "content": "Complete age verification to see age-restricted products matching your search.",
      "path": "$.actions['com.example.identity.age_verification'][0]"
    }
  ]
}
```

상품을 하나도 반환하지 않는 것은 비즈니스가 고를 수 있는 방법 중 하나일 뿐이며, 일부
또는 전체 결과를 반환해도 명세에 부합합니다. 액션 타입과 메시지 코드는 예시이며,
카탈로그는 둘 다 정의하지 않습니다.

## 메시지와 에러 처리 { #messages-and-error-handling }

모든 카탈로그 응답에는 선택 사항인 `messages` 배열이 있어, 비즈니스가 에러, 경고, 정보성
알림의 맥락을 전달할 수 있습니다.

### 메시지 타입 { #message-types }

메시지는 비즈니스의 처리 결과를 전달하고 맥락을 제공합니다.

| 타입 | 사용 시점 | 예시 코드 |
| :--- | :--- | :--- |
| `error` | 비즈니스 수준 에러 | `NOT_FOUND`, `OUT_OF_STOCK`, `REGION_RESTRICTED` |
| `warning` | 구매에 영향을 주는 중요한 조건 | `DELAYED_FULFILLMENT`, `FINAL_SALE` |
| `info` | 문제없이 덧붙이는 추가 맥락 | `PROMOTIONAL_PRICING`, `LIMITED_AVAILABILITY` |

`presentation: "disclosure"`가 있는 경고에는 플랫폼이 숨기거나 닫아서는 안 되는 고지(예:
알레르기 유발 성분 표시, 안전 경고)가 담깁니다. 전체 렌더링 계약은
[경고 표시](../checkout/index.md#warning-presentation)를 참고하세요.

**참고**: 대부분의 카탈로그 에러는 `severity: "recoverable"`을 사용하며, 에이전트가 이를
프로그램으로 처리합니다(재시도, 사용자에게 알림, 대안 제시). `get_product`는 식별자를
해석할 수 없으면 `severity: "unrecoverable"`을 반환하며, 이때 에이전트는 같은 `id`로
재시도해서는 안 됩니다(MUST NOT)([REST](rest.md#product-not-found)와
[MCP](mcp.md#product-not-found) 예시 참고).

#### 메시지 (에러) { #message-error }

{{ schema_fields('types/message_error', 'shopping/catalog') }}

#### 메시지 (경고) { #message-warning }

{{ schema_fields('types/message_warning', 'shopping/catalog') }}

#### 메시지 (정보) { #message-info }

{{ schema_fields('types/message_info', 'shopping/catalog') }}

### 일반적인 시나리오 { #common-scenarios }

#### 빈 검색 결과 { #empty-search }

검색 결과가 없으면 메시지 없이 빈 배열을 반환합니다.

<!-- ucp:example schema=shopping/catalog_search op=search -->
```json
{
  "ucp": {...},
  "products": []
}
```

이는 에러가 아닙니다. 쿼리는 유효했지만 결과가 없었을 뿐입니다.

#### 이월 주문 경고 { #backorder-warning }

상품은 구매할 수 있지만 풀필먼트가 지연되는 경우, 경고 메시지와 함께 상품을 반환합니다.
특정 변형을 가리키려면 `path` 필드를 사용합니다.

<!-- ucp:example schema=shopping/catalog_search op=search -->
```json
{
  "ucp": {...},
  "products": [
    {
      "id": "prod_xyz789",
      "title": "Professional Chef Knife Set",
      "description": { "plain": "Complete professional knife collection." },
      "price_range": {
        "min": { "amount": 29900, "currency": "USD" },
        "max": { "amount": 29900, "currency": "USD" }
      },
      "variants": [
        {
          "id": "var_abc",
          "title": "12-piece Set",
          "description": { "plain": "Complete professional knife collection." },
          "price": { "amount": 29900, "currency": "USD" },
          "availability": { "available": true }
        }
      ]
    }
  ],
  "messages": [
    {
      "type": "warning",
      "code": "delayed_fulfillment",
      "path": "$.products[0].variants[0]",
      "content": "12-piece set on backorder, ships in 2-3 weeks"
    }
  ]
}
```

에이전트는 해당 옵션을 보여 주면서 사용자에게 지연 사실을 알릴 수 있습니다. `path` 필드는
RFC 9535 JSONPath로 특정 구성 요소를 가리킵니다.

#### 식별자를 찾을 수 없음 { #identifiers-not-found }

요청한 식별자가 존재하지 않아도, 찾은 상품(있다면)과 함께 성공으로 반환합니다. 응답에
찾지 못한 식별자를 알리는 정보성 메시지를 포함할 수 있습니다(MAY).

<!-- ucp:example schema=shopping/catalog_lookup op=lookup -->
```json
{
  "ucp": {...},
  "products": [],
  "messages": [
    {
      "type": "info",
      "code": "not_found",
      "content": "prod_invalid"
    }
  ]
}
```

에이전트는 각 변형의 `inputs` 배열로 결과의 대응 관계를 파악합니다.
[클라이언트 대응](lookup.md#client-correlation)을 참고하세요.

#### 상품 고지 { #product-disclosure }

상품에 고지(예: 알레르기 유발 성분 안내, 안전 경고)가 필요하면
`presentation: "disclosure"`가 있는 경고로 반환합니다. `path` 필드는 응답에서 관련 구성
요소를 가리키며, 상품을 가리키면 고지가 그 상품의 모든 변형에 적용됩니다.

<!-- ucp:example schema=shopping/catalog_search op=search -->
```json
{
  "ucp": {...},
  "products": [
    {
      "id": "prod_nut_butter",
      "title": "Artisan Nut Butter Collection",
      "description": { "plain": "Assorted artisan nut butters." },
      "price_range": {
        "min": { "amount": 1299, "currency": "USD" },
        "max": { "amount": 1499, "currency": "USD" }
      },
      "variants": [
        {
          "id": "var_almond",
          "title": "Almond Butter",
          "description": { "plain": "Smooth almond butter." },
          "price": { "amount": 1299, "currency": "USD" },
          "availability": { "available": true }
        },
        {
          "id": "var_cashew",
          "title": "Cashew Butter",
          "description": { "plain": "Creamy cashew butter." },
          "price": { "amount": 1499, "currency": "USD" },
          "availability": { "available": true }
        }
      ]
    }
  ],
  "messages": [
    {
      "type": "warning",
      "code": "allergens",
      "path": "$.products[0]",
      "content": "**Contains: tree nuts.** Produced in a facility that also processes peanuts, milk, and soy.",
      "content_type": "markdown",
      "presentation": "disclosure",
      "image_url": "https://merchant.com/allergen-tree-nuts.svg",
      "url": "https://merchant.com/allergen-info"
    }
  ]
}
```

전체 렌더링 계약은 [경고 표시](../checkout/index.md#warning-presentation)를 참고하세요.

## 스코프 { #scopes }

카탈로그 검색과 카탈로그 조회 기능은 사용자 인증 접근을 위해 다음과 같은 잘 알려진
스코프를 정의합니다.

| 스코프 | 설명 |
| :--- | :--- |
| `dev.ucp.shopping.catalog.search:read` | 인증된 사용자를 대신해 검색합니다. 개인화된 결과, 회원 가격, 접근이 제한된 재고 등이 해당합니다. |
| `dev.ucp.shopping.catalog.lookup:read` | 인증된 사용자를 대신해 조회합니다. 특정 상품의 개인화된 가격이나 재고 상태 등이 해당합니다. |

스코프 선언과 도출, 사용자 정의 스코프로 이 집합을 확장하는 규칙은
[신원 연결 — 스코프](../../common/identity-linking/index.md#scopes)에 정의되어 있습니다.

## 전송 바인딩 { #transport-bindings }

위 기능은 다음 전송 프로토콜에 바인딩됩니다.

* [REST 바인딩](rest.md): RESTful API 매핑
* [MCP 바인딩](mcp.md): JSON-RPC 기반 Model Context Protocol 매핑
