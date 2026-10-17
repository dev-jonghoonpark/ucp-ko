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

# 주문 기능 { #order-capability }

* **기능 이름:** `dev.ucp.shopping.order`

## 개요 { #overview }

주문(Order)은 체크아웃 제출이 성공한 결과로 확정된 거래를 나타낸다. 무엇을 구매했는지,
어떻게 배송될지, 주문 이후 어떤 일이 있었는지에 대한 완전한 기록을 제공한다.

### 핵심 개념 { #key-concepts }

주문은 세 가지 주요 구성 요소로 이루어진다.

**라인 아이템** — 체크아웃에서 구매한 것:

* 현재 수량 집계(전체, 이행 완료)를 포함한다
* 주문 후 변경될 수 있다(예: 주문 수정, 교환). 수정이나 변경 여부와 관계없이 주문에
  존재했던 모든 라인 아이템을 반드시 포함해야 한다(**MUST**)

**풀필먼트** — 상품이 전달되는 방식:

* **예상(Expectations)** — 상품이 언제/어떻게 도착할지에 대해 구매자에게 보여주는 *약속*
* **이벤트(Events)**(추가 전용 로그) — 실제로 일어난 일(예: 👕 발송됨)

**조정(Adjustments)** — 풀필먼트와 무관한 주문 후 이벤트:

* 주로 금전 이동(환불, 반품, 크레딧, 분쟁, 취소)
* 주문 후 어떤 변경이든 될 수 있다
* 풀필먼트 이전, 도중, 이후 언제든 일어날 수 있다
* 비즈니스는 기존 항목을 변경하지 않고 새 항목을 추가해야 한다(**SHOULD**).
  추가 전용 원장 방식을 권장한다. 조정 이력을 관리하지 않는 비즈니스는 기존 항목을
  제자리에서 갱신할 수 있다(**MAY**)
  (예: 하나의 `return` 조정이 `pending`에서 `completed`로 전환될 수 있다)

## 데이터 모델 { #data-model }

### 라인 아이템 { #line-items }

라인 아이템은 체크아웃에서 구매한 것과 그 현재 상태를 반영한다.

* 상품 세부 정보(제품, 가격, 주문 수량)
* 수량 집계와 풀필먼트 상태 — `original`, `total`, `fulfilled`는 공통
  [수량과 단위](../../overview/index.md#quantities-and-units) 규약에 따라 상품에서 상속한
  `quantity_unit`의 정수 단계 수다. `quantity_unit`이 없으면 수량은 `scale` 0에서
  상품 개수(`each`)를 센다

수량에 대한 모든 주문 수명 주기 연산은 상품에서 상속한 판매 기준 단계 수를 대상으로
한다. 상품의 다른 특성은 이 연산에 관여하지 않는다. 비즈니스가 기록하는 수량(풀필먼트
이벤트, 조정, 수정된 합계)은 `scale`의 제약만 받는다. 선언된 주문
[`increment`](../../overview/index.md#ordering-increment)는 장바구니와 체크아웃에서
플랫폼 요청을 제약할 뿐, 비즈니스가 기록하는 값은 제한하지 않는다(0.25 lb 주문 증분이
있어도 실제로 피킹한 무게 1.90 lb를 기록할 수 있다).

### 풀필먼트 { #fulfillment }

풀필먼트는 상품이 구매자에게 어떻게 전달되는지를 추적한다.

#### 예상 { #expectations }

**예상(Expectations)**은 구매자에게 보여주는 상품 묶음이다(예: "패키지 📦"). 예상은
다음을 나타낸다.

* 어떤 상품이 함께 묶이는지
* 어디로 가는지(`destination`)
* 어떻게 배송되는지(`method_type`)
* 언제 도착하는지(`description`, `fulfillable_on`)

예상은 주문 후에 분할, 병합, 조정될 수 있다. 예를 들면:

* 모든 것을 배송일별로 묶는다: "무엇이 언제 오는가"
* 유연성을 위해 넓은 날짜 범위를 가진 예상 하나를 사용한다
* 목표는 최상의 구매자 경험을 위해 **구매자의 기대치를 설정하는 것**이다

#### 풀필먼트 이벤트 { #fulfillment-events }

**풀필먼트 이벤트**는 물리적 발송을 추적하는 추가 전용 로그다.

* ID와 수량으로 라인 아이템을 참조한다
* 배송 추적 정보를 포함한다
* 유형은 개방형 문자열 필드다. 비즈니스는 의미에 맞는 어떤 값이든 사용할 수 있다
  (일반적인 예: `processing`, `shipped`, `in_transit`, `delivered`,
  `failed_attempt`, `canceled`, `undeliverable`, `returned_to_sender`)

### 어트리뷰션 { #attribution }

비즈니스는 원래 체크아웃의 `attribution` 스냅샷을 주문에 노출할 수 있다(MAY). 주문에서는
읽기 전용이며, 에이전트는 `order.attribution`을 쓰지 않는다. 기반 규약은
[어트리뷰션](../../overview/index.md#attribution)을 참고한다.

### 정책 { #policies }

비즈니스는 체크아웃 시점에 적용된 정책의 스냅샷을 주문에 노출할 수 있다(MAY). 기반 규약은
[정책](../../overview/index.md#policies)을 참고한다.

### 조정 { #adjustments }

**조정(Adjustments)**은 풀필먼트와 독립적으로 존재하는 주문 후 이벤트다.

* 유형은 개방형 문자열 필드다. 비즈니스는 의미에 맞는 어떤 값이든 사용할 수 있다
  (주로 `refund`, `return`, `credit`, `price_adjustment`, `dispute`,
  `cancellation` 같은 금전 이동)
* 주문 후 어떤 변경이든 될 수 있다
* 선택적으로 라인 아이템에 연결된다(배송비 환불 같은 경우에는 주문 수준)
* 수량은 부호 있는 단계 수이고 금액도 부호가 있다. 감소(반품, 환불)는 음수, 증가(교환)는
  양수다
* 관련이 있으면 합계 내역을 포함한다
* 풀필먼트 상태와 관계없이 언제든 일어날 수 있다

## 스키마 { #schema }

### 주문 { #order }

{{ schema_fields('order', 'shopping/order') }}

### 주문 라인 아이템 { #order-line-item }

라인 아이템은 체크아웃에서 구매한 것과 그 현재 상태를 반영한다.

{{ schema_fields('order_line_item', 'shopping/order') }}

**수량 구조:**

<!-- ucp:example schema=shopping/types/order_line_item target=$.quantity -->
```json
{
  "original": 3,   // Quantity from the original checkout
  "total": 3,      // Current total (may differ after edits/exchanges)
  "fulfilled": 2   // What has been fulfilled
}
```

상품이 계량 단위로 판매되는 경우 이 값들은 단계 수다. `item.quantity_unit`이
`{ "unit": "LBR", "scale": 2, "display_text": "lb" }`이면 `fulfilled: 50`은
0.50 lb가 이행되었다는 뜻이다.

**상태 도출:**

`total`과 `fulfilled`는 같은 상속 `item.quantity_unit`의 단계 수이며, 도출은 이 값을
대상으로 한다.

```text
if (total == 0) → "removed"
else if (fulfilled == total) → "fulfilled"
else if (fulfilled > 0) → "partial"
else → "processing"
```

### 예상 { #expectation }

예상은 상품이 언제/어떻게 전달될지를 나타내는, 구매자에게 보여주는 묶음이다. 구매자에
대한 현재의 약속을 나타내며 주문 후에 분할, 병합, 조정될 수 있다.

{{ schema_fields('expectation', 'shopping/order') }}

### 풀필먼트 이벤트 { #fulfillment-event }

이벤트는 실제 발송을 추적하는 추가 전용 기록이다. `type` 필드는 개방형 문자열이며,
비즈니스는 자신의 풀필먼트 프로세스에 맞는 어떤 값이든 사용할 수 있다.

{{ schema_fields('fulfillment_event', 'shopping/order') }}

예: `processing`, `shipped`, `in_transit`, `delivered`, `failed_attempt`,
`canceled`, `undeliverable`, `returned_to_sender` 등.

### 조정 { #adjustment }

조정은 풀필먼트와 독립적으로 존재하는 다형적 이벤트다. `type` 필드는 개방형 문자열이며,
비즈니스는 자신에게 맞는 어떤 값이든 사용할 수 있다.

{{ schema_fields('adjustment', 'shopping/order') }}

예: `refund`, `return`, `credit`, `price_adjustment`, `dispute`,
`cancellation` 등.

## 예시 { #example }

<!-- ucp:example schema=shopping/order op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "{{ ucp_version }}"}]
    }
  },
  "id": "order_abc123",
  "checkout_id": "checkout_xyz789",
  "permalink_url": "https://business.example.com/orders/abc123",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_shoes",
      "item": { "id": "prod_shoes", "title": "Running Shoes", "price": 3000 },
      "quantity": { "original": 3, "total": 3, "fulfilled": 3 },
      "totals": [
        {"type": "subtotal", "amount": 9000},
        {"type": "total", "amount": 9000}
      ],
      "status": "fulfilled"
    },
    {
      "id": "li_shirts",
      "item": { "id": "prod_shirts", "title": "Cotton T-Shirt", "price": 2000 },
      "quantity": { "original": 2, "total": 2, "fulfilled": 0 },
      "totals": [
        {"type": "subtotal", "amount": 4000},
        {"type": "total", "amount": 4000}
      ],
      "status": "processing"
    }
  ],
  "fulfillment": {
    "expectations": [
      {
        "id": "exp_1",
        "line_items": [{ "id": "li_shoes", "quantity": 3 }],
        "method_type": "shipping",
        "destination": {
          "street_address": "123 Main St",
          "address_locality": "Austin",
          "address_region": "TX",
          "address_country": "US",
          "postal_code": "78701"
        },
        "description": "Arrives in 2-3 business days",
        "fulfillable_on": "now"
      },
      {
        "id": "exp_2",
        "line_items": [{ "id": "li_shirts", "quantity": 2 }],
        "method_type": "shipping",
        "destination": {
          "street_address": "123 Main St",
          "address_locality": "Austin",
          "address_region": "TX",
          "address_country": "US",
          "postal_code": "78701"
        },
        "description": "Backordered - ships Jan 15, arrives in 7-10 days",
        "fulfillable_on": "2025-01-15T00:00:00Z"
      }
    ],
    "events": [
      {
        "id": "evt_1",
        "occurred_at": "2025-01-08T10:30:00Z",
        "type": "delivered",
        "line_items": [{ "id": "li_shoes", "quantity": 3 }],
        "tracking_number": "123456789",
        "tracking_url": "https://fedex.com/track/123456789",
        "description": "Delivered to front door"
      }
    ]
  },
  "adjustments": [
    {
      "id": "adj_1",
      "type": "refund",
      "occurred_at": "2025-01-10T14:30:00Z",
      "status": "completed",
      "line_items": [{ "id": "li_shoes", "quantity": -1 }],
      "totals": [
        { "type": "total", "amount": -3000 }
      ],
      "description": "Defective item"
    }
  ],
  "totals": [
    { "type": "subtotal", "amount": 13000 },
    { "type": "fulfillment", "amount": 1200 },
    { "type": "tax", "amount": 1142 },
    { "type": "total", "amount": 15342 }
  ]
}
```

## 예시: 계량 단위 판매 상품 { #example-goods-sold-by-measure }

파운드 단위로 판매되는 바나나 주문이다(`item.quantity_unit`
`{ "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }`).
수량은 단계 수다. 구매자는 1.50 lb를 주문했고(`total: 150`), 그중 0.50 lb가
발송되었으므로(`fulfilled: 50`) 라인은 `partial`이다. 이후 0.25 lb 반품은 `-25`
단계의 조정으로 기록된다. 금액은 $0.79/lb로 책정되고 한 번만 반올림된다. 라인 합계는
`79 × 1.50 = 118.5`를 반올림한 `119`이고, 반품 크레딧은 `79 × 0.25 = 19.75`를
반올림한 `20`이다.

<!-- ucp:example schema=shopping/order op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": { "dev.ucp.shopping.order": [{"version": "{{ ucp_version }}"}] }
  },
  "id": "order_bananas_1",
  "checkout_id": "chk_bananas_1",
  "permalink_url": "https://business.example.com/orders/bananas1",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_bananas",
      "item": {
        "id": "var_bananas",
        "title": "Bananas",
        "price": 79,
        "quantity_unit": { "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }
      },
      "quantity": { "original": 150, "total": 150, "fulfilled": 50 },
      "totals": [
        { "type": "subtotal", "amount": 119 },
        { "type": "total", "amount": 119 }
      ],
      "status": "partial"
    }
  ],
  "fulfillment": {
    "events": [
      {
        "id": "evt_1",
        "occurred_at": "2026-01-08T10:30:00Z",
        "type": "shipped",
        "line_items": [{ "id": "li_bananas", "quantity": 50 }],
        "tracking_number": "123456789",
        "tracking_url": "https://carrier.example/track/123456789",
        "description": "0.50 lb shipped"
      }
    ]
  },
  "adjustments": [
    {
      "id": "adj_1",
      "type": "return",
      "occurred_at": "2026-01-10T14:30:00Z",
      "status": "completed",
      "line_items": [{ "id": "li_bananas", "quantity": -25 }],
      "totals": [{ "type": "total", "amount": -20 }],
      "description": "Returned 0.25 lb"
    }
  ],
  "totals": [
    { "type": "subtotal", "amount": 119 },
    { "type": "total", "amount": 119 }
  ]
}
```

## 예시: 개수로 판매하고 계량으로 정산하는 경우 { #example-count-sold-measure-priced-settlement }

어떤 상품은 개수로 판매되지만 가격은 계량으로 책정된다. 예를 들어 사과를 $2.00/lb에
`each` 단위로 판매하고, 사과 한 개당 명목 무게를 둔다. 여기에는 두 가지 기준이 함께
작동하며 둘 다 전달된다. 판매 기준은 `quantity`를 결정하고(사과 개수, 수명 주기 연산은
그대로), 가격 기준인 라인에 반영된
[`unit_price`](../checkout/index.md#quantity-and-sale-basis)는 요금 계산의 근거가 되는
단가와 명목 계량을 담는다. 가격을 실제 계량으로 정산할 때 조정은 반드시 정산된 `measure`를
담아야 한다(**MUST**). 그 단위 식별은 반드시 가격 기준과 일치해야 하며(**MUST**)(변환
없음), 순수한 가격 정산은 개수가 바뀌지 않도록 `quantity: 0`을 사용한다. (정산된
`measure`는 비즈니스가 기록한 사실이다. 상품 식별을 정의하는 구매자 설정 계량은 별도로
협상되는 확장이다.)

구매자가 사과 3개를 개당 $0.80에 주문한다(명목 0.40 lb × $2.00/lb, 라인 합계 `240`).
피커가 잰 사과 세 개의 무게는 명목 1.20 lb 대비 1.14 lb이며, 정산은 주문만으로 완전히
검증할 수 있다. 명목 단계 `3 × 40 = 120`, 정산 `114`, 차이 `6` 단계
`× 200 × 10^-2 = 12`이므로 `-12` 조정이 정수 연산으로 맞아떨어진다.

<!-- ucp:example schema=shopping/order op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": { "dev.ucp.shopping.order": [{"version": "{{ ucp_version }}"}] }
  },
  "id": "order_apples_1",
  "checkout_id": "chk_apples_1",
  "permalink_url": "https://business.example.com/orders/apples1",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_apples",
      "item": {
        "id": "var_apple",
        "title": "Honeycrisp Apple",
        "price": 80,
        "unit_price": {
          "amount": 200,
          "currency": "USD",
          "measure": { "value": 40, "scale": 2, "unit": "LBR", "display_text": "lb" },
          "reference": { "value": 1, "unit": "LBR", "display_text": "lb" }
        }
      },
      "quantity": { "original": 3, "total": 3, "fulfilled": 3 },
      "totals": [
        { "type": "subtotal", "amount": 240 },
        { "type": "total", "amount": 240 }
      ],
      "status": "fulfilled"
    }
  ],
  "fulfillment": {
    "events": [
      {
        "id": "evt_1",
        "occurred_at": "2026-01-14T09:15:00Z",
        "type": "shipped",
        "line_items": [{ "id": "li_apples", "quantity": 3 }],
        "description": "Picked 3 apples, 1.14 lb total"
      }
    ]
  },
  "adjustments": [
    {
      "id": "adj_1",
      "type": "price_adjustment",
      "occurred_at": "2026-01-14T09:15:00Z",
      "status": "completed",
      "line_items": [
        {
          "id": "li_apples",
          "quantity": 0,
          "measure": { "value": 114, "scale": 2, "unit": "LBR", "display_text": "lb" }
        }
      ],
      "totals": [{ "type": "total", "amount": -12 }],
      "description": "Settled to actual picked weight of 1.14 lb"
    }
  ],
  "totals": [
    { "type": "subtotal", "amount": 240 },
    { "type": "total", "amount": 240 }
  ]
}
```

개수 수명 주기는 전혀 움직이지 않고(사과 개수 기준으로 `fulfilled == total`이 유지된다)
금액만 실제 계량에 맞춰 조정되며, 크레딧을 검증하는 데 필요한 모든 숫자가 주문 자체에
들어 있다.

## 예시: 실중량(catch-weight) 조정 { #example-catch-weight-reconciliation }

무게를 다는 상품은 피킹한 무게가 주문한 무게와 다른 경우가 흔하다. 이 차이는 수치 오류가
아니라 상거래상의 사실이다. 비즈니스는 실제 피킹 결과를 기록하고, 금액과 수량을 함께
움직이는 [조정](#adjustments)으로 차이를 맞춘다. 허용 오차 비교는 개입하지 않으며, 상태
도출은 처음부터 끝까지 정확한 단계 수를 대상으로 한다.

구매자가 같은 바나나를 $0.79/lb에 2.00 lb 주문한다. 판매 단위는 1/4파운드 증분이다
(`quantity` `200`, 라인 합계 `79 × 2.00 = 158`). 피커가 1.90 lb를 담는다. 증분에 맞지
않는 값이지만 있는 그대로 기록한다.
[`increment`](../../overview/index.md#ordering-increment)는 플랫폼의 주문을 제약할 뿐,
비즈니스의 기록을 제약하지 않기 때문이다. 풀필먼트 이벤트는 실제 `190` 단계를 기록하고,
`-10` 단계의 `price_adjustment`가 가격 차이(`79 × 0.10 = 7.9`를 한 번 반올림한 `8`)와
함께 `total`을 실제 피킹 값에 맞춘다. 그러면 `fulfilled == total`이 정확히
성립하고(`190 == 190`) 라인은 `fulfilled`로 도출된다.

<!-- ucp:example schema=shopping/order op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": { "dev.ucp.shopping.order": [{"version": "{{ ucp_version }}"}] }
  },
  "id": "order_bananas_2",
  "checkout_id": "chk_bananas_2",
  "permalink_url": "https://business.example.com/orders/bananas2",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_bananas",
      "item": {
        "id": "var_bananas",
        "title": "Bananas",
        "price": 79,
        "quantity_unit": { "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }
      },
      "quantity": { "original": 200, "total": 190, "fulfilled": 190 },
      "totals": [
        { "type": "subtotal", "amount": 158 },
        { "type": "total", "amount": 158 }
      ],
      "status": "fulfilled"
    }
  ],
  "fulfillment": {
    "events": [
      {
        "id": "evt_1",
        "occurred_at": "2026-01-12T09:15:00Z",
        "type": "shipped",
        "line_items": [{ "id": "li_bananas", "quantity": 190 }],
        "description": "Picked 1.90 lb"
      }
    ]
  },
  "adjustments": [
    {
      "id": "adj_1",
      "type": "price_adjustment",
      "occurred_at": "2026-01-12T09:15:00Z",
      "status": "completed",
      "line_items": [{ "id": "li_bananas", "quantity": -10 }],
      "totals": [{ "type": "total", "amount": -8 }],
      "description": "Adjusted to actual picked weight of 1.90 lb"
    }
  ],
  "totals": [
    { "type": "subtotal", "amount": 158 },
    { "type": "total", "amount": 158 }
  ]
}
```

## 스코프 { #scopes }

주문 기능은 사용자 인증 접근을 위해 다음의 잘 알려진 스코프를 정의한다.

| 스코프 | 설명 |
| :--- | :--- |
| `dev.ucp.shopping.order:read` | 사용자 주문에 대한 읽기 접근 — 인증된 사용자가 소유한 리소스에 대한 주문 조회(Get Order). |
| `dev.ucp.shopping.order:manage` | 사용자 주문에 대한 구매 후 작업 — 취소, 반품, 기타 수정. |

스코프 선언, 도출, 그리고 사용자 정의 스코프로 이 집합을 확장하는 규칙은
[신원 연결 — 스코프](../../common/identity-linking/index.md#scopes)에 정의되어 있다.

## 작업 { #operations }

주문 엔티티는 **현재 상태 스냅샷**이다. 즉 조회 또는 전달 시점의 주문에 대한 권한 있는
최신 상태다. 비즈니스는 반드시 모든 응답에서 전체 주문 엔티티를 반환해야 한다(**MUST**).
동기식 조회(이 섹션)와 비동기식 이벤트 전달([이벤트](#events) 참고)에 같은 스키마를 사용한다.

`permalink_url`은 타임라인, 구매 후 작업, 반품 등 전체 주문 경험에 대한 권한 있는
참조다. API는 대화형 및 운영 사용 사례를 위해 현재 상태에 대한 프로그래밍 방식의 접근을
제공한다.

| 작업                                    | 메서드 | 엔드포인트     | 설명                                    |
| :-------------------------------------- | :----- | :------------- | :-------------------------------------- |
| [주문 조회(Get Order)](#get-order)      | `GET`  | `/orders/{id}` | 플랫폼이 현재 주문 상태를 가져온다.     |

전송 방식별 세부 사항은 [REST 바인딩](rest.md)과
[MCP 바인딩](mcp.md)을 참고한다.

### 주문 조회 { #get-order }

주문의 현재 상태 스냅샷을 반환한다.

#### 인가 { #authorization }

비즈니스는 응답을 반환하기 전에, 지원되는 UCP 메커니즘(API 키, OAuth 2.0, 상호 TLS,
HTTP Message Signatures 중 하나)으로 주문 데이터 요청을 반드시 인증해야 한다(**MUST**)
([신원 확인 및 인증](../checkout/rest.md#authentication) 참고). 인증 방식에 따라 호출자가
접근할 수 있는 주문이 결정된다.

| 인증 | 접근 가능한 주문 |
| :------------- | :----------------------- |
| 플랫폼 자격 증명 | 해당 플랫폼에서 생성된 주문 |
| 구매자 인가 | 부여된 OAuth 스코프 범위 내에서 구매자가 소유한 주문 |

**플랫폼 자격 증명**(API 키, 서명, OAuth 클라이언트 자격 증명) - 비즈니스는 플랫폼이
생성한 주문에 대한 접근을 허용할 수 있다(**MAY**). 플랫폼은 체크아웃 흐름에서 구매자 및
결제 정보를 제공했고, 주문 확인을 지켜보았으며, 이미 맥락을 알고 있는 주문의 최신 상태를
조회하는 것이다.

**구매자 인가** - 플랫폼은 필요한 스코프로 [신원 연결(Identity Linking)](../../common/identity-linking/index.md)
또는 유사한 메커니즘을 통해 구매자 인가를 얻는다. 이렇게 하면 어느 플랫폼에서 생성되었는지와
관계없이 구매자의 주문에 접근할 수 있다.

비즈니스는 인가와는 별개로, 추가 접근 정책(예: 신뢰할 수 있는 파트너 계약)을 정의하고,
데이터 가용성 제약(예: 보존 기간, 규제에 따른 삭제)을 적용하고, 컨텍스트, 비즈니스 정책,
기타 요구 사항에 따라 응답에서 선택 필드를 생략하거나 가릴 수 있다(**MAY**).

#### 에러 응답 { #error-responses }

비즈니스가 주문을 반환할 수 없으면, 결과를 설명하는 `messages` 배열을 포함한 오류를
응답으로 반환한다.

**주문을 찾을 수 없음:**

<!-- ucp:example schema=common/types/error_response op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "{{ ucp_version }}"}]
    }
  },
  "messages": [
    {
      "type": "error",
      "code": "not_found",
      "severity": "unrecoverable",
      "content": "Order not found."
    }
  ]
}
```

**인가되지 않음:**

<!-- ucp:example schema=common/types/error_response op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "{{ ucp_version }}"}]
    }
  },
  "messages": [
    {
      "type": "error",
      "code": "unauthorized",
      "severity": "unrecoverable",
      "content": "Not authorized to access this order."
    }
  ]
}
```

### 지침 {: #operations-guidelines }

**플랫폼:**

* 모든 요청에 프로필 URL을 담은 `UCP-Agent` 헤더를 반드시 포함해야 한다(**MUST**)
* 주된 주문 업데이트 채널로 웹훅([이벤트](#events) 참고)에 의존하고, 주문 조회는 대사(reconciliation)나
  필요 시 조회에 사용해야 한다(**SHOULD**)
* 주문 데이터를 일시적인 것으로 취급하고, 진행 중인 커머스 흐름에 더 이상 필요하지 않으면
  폐기해야 한다(**SHOULD**)

**비즈니스:**

* 응답을 반환하기 전에 주문 데이터 요청을 반드시 인증해야 한다(**MUST**)
  ([인가](#authorization) 참고)

## 이벤트 { #events }

비즈니스는 웹훅을 통해 주문 수명 주기 업데이트를 플랫폼에 푸시한다. 페이로드는
[작업](#operations)에서 설명한 것과 같은 **현재 상태 스냅샷**, 즉 전체 주문 엔티티다.

| 이벤트                                      | 메서드 | 엔드포인트            | 설명                                                   |
| :------------------------------------------ | :----- | :-------------------- | :----------------------------------------------------- |
| [주문 이벤트 웹훅](#order-event-webhook)    | `POST` | 플랫폼이 제공한 URL   | 비즈니스가 주문 수명 주기 이벤트를 플랫폼에 보낸다.    |

### 주문 이벤트 웹훅 { #order-event-webhook }

비즈니스는 파트너 온보딩 시 플랫폼이 제공한 웹훅 URL로 주문 이벤트를 POST한다. URL
형식은 플랫폼마다 다르다.

헤더는 **[Standard Webhooks](https://www.standardwebhooks.com/){ target="_blank" }**를
따르되, 요청 서명만은 [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421)을 따른다.
자세한 내용은 [메시지 서명](../../signatures.md)을 참고한다.

**필수 헤더:**

| 헤더                 | 설명                                        |
| :------------------- | :------------------------------------------ |
| `Webhook-Timestamp`  | 이벤트 발생 타임스탬프(unix)                |
| `Webhook-Id`         | 이벤트 식별자. 재시도 간에 변하지 않는다    |

{{ method_fields('order_event_webhook', 'shopping/rest.openapi.json', 'shopping/order') }}

### 웹훅 URL 설정 { #webhook-url-configuration }

플랫폼은 기능 협상 중에 주문 기능의 `config` 필드에 웹훅 URL을 제공한다. 비즈니스는
플랫폼 프로필에서 이 URL을 찾아 주문 수명 주기 이벤트를 보내는 데 사용한다.

{{ extension_schema_fields('order.json#/$defs/platform_schema', 'shopping/order') }}

**예시:**

<!-- ucp:example schema=profile def=platform_schema target=$.ucp.capabilities -->
```json
{
  "dev.ucp.shopping.order": [
    {
      "version": "{{ ucp_version }}",
      "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/order",
      "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/order.json",
      "config": {
        "webhook_url": "https://platform.example.com/webhooks/ucp/orders"
      }
    }
  ]
}
```

### 웹훅 서명 검증 { #webhook-signature-verification }

진본성과 무결성을 보장하기 위해 웹훅 페이로드는 반드시 비즈니스가 서명하고 플랫폼이
검증해야 한다(**MUST**). 서명은 REST 바인딩(RFC 9421)을 사용하는
[메시지 서명](../../signatures.md) 명세를 따른다.

**필수 헤더:**

| 헤더             | 설명                                       |
| :--------------- | :----------------------------------------- |
| `UCP-Agent`      | 비즈니스 프로필 URL(RFC 8941 Dictionary)   |
| `Signature-Input`| 서명된 구성 요소를 기술한다                |
| `Signature`      | 서명 값을 담는다                           |
| `Content-Digest` | 본문 다이제스트(RFC 9530)                  |

`Webhook-Id`와 `Webhook-Timestamp`는 이 전달의 필수 헤더이므로,
[REST 요청 서명](../../signatures.md#rest-request-signing)의 대상 구성 요소 규칙에 따라
서명이 이 두 헤더를 포함해야 하며,
[REST 요청 검증](../../signatures.md#rest-request-verification)에 따라 플랫폼은 서명에
둘 중 하나라도 빠진 전달을 거부해야 한다. 웹훅에는 `Idempotency-Key`가 없으므로 이 두
헤더가 이벤트별 재전송(replay)을 판별하는 유일한 신호다.

주문 웹훅은 변경분이 아니라 주문의 현재 상태 전체를 담는다. 따라서 `Webhook-Timestamp`
순서대로 전달을 적용하거나 `GET /orders/{id}`로 대사하는 플랫폼은 전달별 상태를 따로
유지하지 않고도 현재 상태로 수렴한다. 플랫폼은 `Webhook-Id`로 중복을 제거해야
한다(**SHOULD**). 이렇게 하면 재시도를 빠르게 걸러낼 수 있고, 해상도가 1초인
`Webhook-Timestamp`가 같은 이벤트들도 구분할 수 있다.

비즈니스는 반드시 원래의 `Webhook-Id`와 `Webhook-Timestamp`로 재시도를 보내야
한다(**MUST**). 재시도는 같은 이벤트를 다시 전달하는 것이며, 위의 중복 제거와 순서
지정은 두 값이 변하지 않는다는 데 의존한다.

**웹훅 요청 예시:**

```http
POST /webhooks/ucp/orders HTTP/1.1
Host: platform.example.com
Content-Type: application/json
UCP-Agent: profile="https://merchant.example/.well-known/ucp"
Webhook-Timestamp: 1737000000
Webhook-Id: 018f8c2a-7b3e-7c1d-9a2b-4e5f6a7b8c9d
Content-Digest: sha-256=:X48E9q...:
Signature-Input: sig1=("@method" "@authority" "@path" "ucp-agent" "content-digest" "content-type" "webhook-id" "webhook-timestamp");keyid="merchant-2026"
Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:

{"id":"order_abc123",...}
```

#### 서명(비즈니스) { #signing-business }

1. 원시 요청 본문의 SHA-256 다이제스트를 계산하고 `Content-Digest` 헤더를 설정한다
2. [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421)에 따라 서명 기반을 만든다.
   이때 [REST 요청 서명](../../signatures.md#rest-request-signing)이 요구하는
   모든 구성 요소를 포함한다
3. 비즈니스 UCP 프로필의 `keys`에 있는 키로 서명한다
4. `Signature-Input`과 `Signature` 헤더를 설정한다

전체 알고리즘은 [메시지 서명 - REST 요청 서명](../../signatures.md#rest-request-signing)을
참고한다.

#### 검증(플랫폼) { #verification-platform }

**인증**(서명 검증):

1. `Signature-Input`을 파싱해 `keyid`와 서명된 구성 요소를 추출한다
2. `/.well-known/ucp`에서 비즈니스의 UCP 프로필을 가져온다(적절히 캐시)
3. `keys`에서 `kid`가 일치하는 키를 찾는다
4. `Content-Digest`가 원시 본문의 SHA-256과 일치하는지 확인한다
5. 서명 기반을 재구성하고 서명을 검증한다
6. [REST 요청 검증](../../signatures.md#rest-request-verification)이
   요구하는 모든 구성 요소가 서명된 구성 요소에 포함되어 있는지 확인하고,
   그렇지 않으면 전달을 거부한다

전체 알고리즘은 [메시지 서명 - REST 요청 검증](../../signatures.md#rest-request-verification)을
참고한다.

**인가**(주문 소유권):

서명을 검증한 후, 플랫폼은 반드시 서명자가 참조된 주문에 대한 이벤트를 보낼 권한이
있는지 확인해야 한다(**MUST**).

1. 웹훅 페이로드에서 주문 ID를 추출한다
2. 해당 주문이 이 비즈니스에서 생성되었는지 확인한다(프로필 URL 일치)
3. 서명자의 프로필이 주문의 비즈니스와 일치하지 않는 웹훅은 거부한다

이렇게 하면 유효한 서명을 가진 악의적인 비즈니스라도 다른 비즈니스의 주문에 대한 가짜
이벤트를 보낼 수 없다.

#### 키 순환 { #key-rotation }

무중단 키 순환 절차는 [메시지 서명 - 키 순환](../../signatures.md#key-rotation)을 참고한다.

### 지침 {: #events-guidelines }

**플랫폼:**

* 웹훅 수신을 확인하기 위해 2xx HTTP 상태 코드로 반드시 빠르게 응답해야 하며(**MUST**),
  이벤트는 응답 후 비동기로 처리한다

**비즈니스:**

* 서명자 식별을 위해 프로필 URL을 담은 `UCP-Agent` 헤더를 반드시 포함해야 한다(**MUST**)
* 모든 웹훅 페이로드를 [메시지 서명](../../signatures.md) 명세에 따라 RFC 9421 헤더
  (`Signature`, `Signature-Input`, `Content-Digest`)로 반드시 서명해야 한다(**MUST**)
* 모든 필드가 채워진 주문 엔티티와 함께 "주문 생성" 이벤트를 반드시 보내야 한다(**MUST**)
* 업데이트 시 반드시 전체 주문 엔티티를 보내야 한다(증분 변경분이 아님)(**MUST**)
* 실패한 웹훅 전달은 반드시 재시도해야 한다(**MUST**)
* 재시도는 반드시 원래의 `Webhook-Id`와 `Webhook-Timestamp`로 보내야 한다(**MUST**).
  재시도는 같은 이벤트를 다시 전달하는 것이며, 플랫폼은 `Webhook-Id`로 중복을 제거하고
  `Webhook-Timestamp`로 전달 순서를 정한다

## 엔티티 { #entities }

### 상품 { #item }

{{ schema_fields('types/item_resp', 'shopping/order') }}

### 우편 주소 { #postal-address }

{{ schema_fields('postal_address', 'shopping/order') }}

### 응답 { #response }

{{ extension_schema_fields('capability.json#/$defs/response_schema', 'shopping/order') }}

### 합계 { #total }

{{ schema_fields('types/total_resp', 'shopping/order') }}

### UCP 응답 주문 스키마 <span id="ucp"></span> {: #ucp-response-order-schema }

{{ extension_schema_fields('ucp.json#/$defs/response_order_schema', 'shopping/order') }}
