# 주문 기능

- **기능 이름:** `dev.ucp.shopping.order`

## 개요

주문(Order)은 체크아웃 제출이 성공해 확정된 거래를 나타냅니다. 무엇을 구매했는지, 어떻게 배송될지, 주문 이후 어떤 일이 있었는지를 빠짐없이 기록합니다.

### 핵심 개념

주문은 세 가지 주요 구성 요소로 이루어집니다.

**라인 아이템** — 체크아웃에서 구매한 것:

- 현재 수량 집계(전체, 이행 완료)가 들어 있습니다
- 주문 후 바뀔 수 있습니다(예: 주문 수정, 교환). 수정이나 변경 여부와 관계없이 주문에 있었던 모든 라인 아이템을 반드시 포함해야 합니다(**MUST**)

**풀필먼트** — 상품이 전달되는 방식:

- **예상(Expectations)** — 상품이 언제/어떻게 도착할지 구매자에게 보여 주는 *약속*
- **이벤트(Events)**(추가 전용 로그) — 실제로 일어난 일(예: 👕 발송됨)

**조정(Adjustments)** — 풀필먼트와 무관한 주문 후 이벤트:

- 주로 금전 이동(환불, 반품, 크레딧, 분쟁, 취소)
- 주문 후의 어떤 변경이든 조정이 될 수 있습니다
- 풀필먼트 이전, 도중, 이후 언제든 일어날 수 있습니다
- 비즈니스는 기존 항목을 바꾸지 말고 새 항목을 추가해야 합니다(**SHOULD**). 추가 전용 원장 방식을 권장합니다. 조정 이력을 관리하지 않는 비즈니스는 기존 항목을 그 자리에서 갱신할 수 있습니다(**MAY**) (예: `return` 조정 하나가 `pending`에서 `completed`로 바뀔 수 있습니다)

## 데이터 모델

### 라인 아이템

라인 아이템은 체크아웃에서 구매한 항목과 그 현재 상태를 반영합니다.

- 상품 세부 정보(제품, 가격, 주문 수량)
- 수량 집계와 풀필먼트 상태 — `original`, `total`, `fulfilled`는 공통 [수량과 단위](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#quantities-and-units) 규약에 따라 상품에서 상속한 `quantity_unit`의 정수 단계 수입니다. `quantity_unit`이 없으면 수량은 `scale` 0 기준의 상품 개수(`each`)입니다

수량을 다루는 모든 주문 수명 주기 연산은 상품에서 상속한 판매 기준 단계 수를 대상으로 합니다. 상품의 다른 특성은 이 연산에 관여하지 않습니다. 비즈니스가 기록하는 수량(풀필먼트 이벤트, 조정, 수정된 합계)은 `scale`의 제약만 받습니다. 선언된 주문 [`increment`](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#ordering-increment)는 장바구니와 체크아웃에서 플랫폼 요청을 제약할 뿐, 비즈니스가 기록하는 값은 제한하지 않습니다(주문 증분이 0.25 lb여도 실제로 피킹한 무게 1.90 lb를 기록할 수 있습니다).

### 풀필먼트

풀필먼트는 상품이 구매자에게 어떻게 전달되는지를 추적합니다.

#### 예상

\*\*예상(Expectations)\*\*은 구매자에게 보여 주는 상품 묶음입니다(예: "패키지 📦"). 예상에는 다음 내용이 담깁니다.

- 어떤 상품이 함께 묶이는지
- 어디로 가는지(`destination`)
- 어떻게 배송되는지(`method_type`)
- 언제 도착하는지(`description`, `fulfillable_on`)

예상은 주문 후에 분할, 병합, 조정될 수 있습니다. 예를 들면 다음과 같습니다.

- 모든 상품을 배송일별로 묶습니다: "무엇이 언제 오는가"
- 유연하게 대응하려고 날짜 범위가 넓은 예상 하나를 사용합니다
- 목표는 최상의 구매자 경험을 위해 **구매자의 기대치를 정해 주는 것**입니다

#### 풀필먼트 이벤트

**풀필먼트 이벤트**는 물리적 발송을 추적하는 추가 전용 로그입니다.

- ID와 수량으로 라인 아이템을 참조합니다
- 배송 추적 정보가 들어 있습니다
- 유형은 개방형 문자열 필드입니다. 비즈니스는 의미에 맞는 값이면 무엇이든 쓸 수 있습니다 (일반적인 예: `processing`, `shipped`, `in_transit`, `delivered`, `failed_attempt`, `canceled`, `undeliverable`, `returned_to_sender`)

### 어트리뷰션

비즈니스는 원래 체크아웃의 `attribution` 스냅샷을 주문에 노출할 수 있습니다(MAY). 주문에서는 읽기 전용이며, 에이전트는 `order.attribution`에 값을 쓰지 않습니다. 기반 규약은 [어트리뷰션](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#attribution)을 참고하세요.

### 정책

비즈니스는 체크아웃 시점에 적용된 정책의 스냅샷을 주문에 노출할 수 있습니다(MAY). 기반 규약은 [정책](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#policies)을 참고하세요.

### 조정

\*\*조정(Adjustments)\*\*은 풀필먼트와 별개로 존재하는 주문 후 이벤트입니다.

- 유형은 개방형 문자열 필드입니다. 비즈니스는 의미에 맞는 값이면 무엇이든 쓸 수 있습니다 (주로 `refund`, `return`, `credit`, `price_adjustment`, `dispute`, `cancellation` 같은 금전 이동)
- 주문 후의 어떤 변경이든 조정이 될 수 있습니다
- 선택적으로 라인 아이템에 연결됩니다(배송비 환불 같은 경우에는 주문 수준)
- 수량은 부호 있는 단계 수이고 금액에도 부호가 있습니다. 감소(반품, 환불)는 음수, 증가(교환)는 양수입니다
- 관련이 있으면 합계 내역을 포함합니다
- 풀필먼트 상태와 관계없이 언제든 일어날 수 있습니다

## 스키마

### 주문

| 이름          | 타입                                                                                               | 필수 여부    | 설명                                                                                                                                                               |
| ------------- | -------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ucp           | any                                                                                                | **Required** | UCP metadata for order responses. No payment handlers needed post-purchase.                                                                                        |
| id            | string                                                                                             | **Required** | Unique order identifier.                                                                                                                                           |
| label         | string                                                                                             | Optional     | Human-readable label for identifying the order. MUST only be provided by the business.                                                                             |
| checkout_id   | string                                                                                             | **Required** | Associated checkout ID for reconciliation.                                                                                                                         |
| permalink_url | string                                                                                             | **Required** | Permalink to access the order on merchant site.                                                                                                                    |
| line_items    | Array\[[Order Line Item](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#order-line-item)\] | **Required** | Line items representing what was purchased — can change post-order via edits or exchanges.                                                                         |
| fulfillment   | object                                                                                             | **Required** | Fulfillment data: buyer expectations and what actually happened.                                                                                                   |
| adjustments   | Array\[[Adjustment](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#adjustment)\]           | Optional     | Post-order events (refunds, returns, credits, disputes, cancellations, etc.) that exist independently of fulfillment.                                              |
| currency      | string                                                                                             | **Required** | ISO 4217 currency code. MUST match the currency from the originating checkout session.                                                                             |
| totals        | [Totals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#totals)                            | **Required** | Different totals for the order.                                                                                                                                    |
| policies      | Array\[[Policy](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#policy)\]                   | Optional     | Snapshot of the policies that applied to the items at checkout, captured on the order as a durable record. `applies_to` targets are relative to the response root. |
| messages      | Array\[[Message](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message)\]                 | Optional     | Business outcome messages (errors, warnings, informational). Present when the business needs to communicate status or issues to the platform.                      |
| attribution   | [Attribution](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#attribution)                  | Optional     | Snapshot of the attribution associated with the originating checkout. Read-only on the order.                                                                      |

### 주문 라인 아이템

라인 아이템은 체크아웃에서 구매한 항목과 그 현재 상태를 반영합니다.

| 이름      | 타입                                                                                    | 필수 여부    | 설명                                                                                                                                                                                                                                                                            |
| --------- | --------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id        | string                                                                                  | **Required** | Line item identifier.                                                                                                                                                                                                                                                           |
| item      | [Item](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#item)                     | **Required** | Purchased item data, including identity, price, and sale basis.                                                                                                                                                                                                                 |
| quantity  | object                                                                                  | **Required** | Tracks the line item's original, current active, and fulfilled quantities. All three values use the same inherited `item.quantity_unit`. When `item.quantity_unit` is absent on an authoritative order response, each step is one whole item (`each`) under the shared default. |
| totals    | Array\[[Total Response](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#total)\] | **Required** | Line item totals breakdown.                                                                                                                                                                                                                                                     |
| status    | string                                                                                  | **Required** | Derived status: removed if quantity.total == 0, fulfilled if quantity.total > 0 and quantity.fulfilled == quantity.total, partial if quantity.total > 0 and quantity.fulfilled > 0, otherwise processing. **Enum:** `processing`, `partial`, `fulfilled`, `removed`             |
| parent_id | string                                                                                  | Optional     | Parent line item identifier for any nested structures.                                                                                                                                                                                                                          |

**수량 구조:**

```json
{
  "original": 3,   // Quantity from the original checkout
  "total": 3,      // Current total (may differ after edits/exchanges)
  "fulfilled": 2   // What has been fulfilled
}
```

상품이 계량 단위로 판매되면 이 값들은 단계 수입니다. `item.quantity_unit`이 `{ "unit": "LBR", "scale": 2, "display_text": "lb" }`이면 `fulfilled: 50`은 0.50 lb가 이행되었다는 뜻입니다.

**상태 도출:**

`total`과 `fulfilled`는 상속받은 같은 `item.quantity_unit`의 단계 수이며, 상태는 이 값으로 도출합니다.

```text
if (total == 0) → "removed"
else if (fulfilled == total) → "fulfilled"
else if (fulfilled > 0) → "partial"
else → "processing"
```

### 예상

예상은 상품이 언제/어떻게 전달될지를 구매자에게 보여 주는 묶음입니다. 구매자에게 하는 현재의 약속을 나타내며, 주문 후에 분할, 병합, 조정될 수 있습니다.

| 이름           | 타입                                                                                    | 필수 여부    | 설명                                                                                                        |
| -------------- | --------------------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------- |
| id             | string                                                                                  | **Required** | Expectation identifier.                                                                                     |
| line_items     | Array[object]                                                                           | **Required** | Which line items and quantities are in this expectation.                                                    |
| method_type    | string                                                                                  | **Required** | Delivery method type. Well-known values: `shipping`, `pickup`, `digital`; additional values MAY be used.    |
| destination    | [Postal Address](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#postal-address) | **Required** | Delivery destination address.                                                                               |
| description    | string                                                                                  | Optional     | Human-readable delivery description (e.g., 'Arrives in 5-8 business days').                                 |
| fulfillable_on | string                                                                                  | Optional     | When this expectation can be fulfilled: 'now' or ISO 8601 timestamp for future date (backorder, pre-order). |

### 풀필먼트 이벤트

이벤트는 실제 발송을 추적하는 추가 전용 기록입니다. `type` 필드는 개방형 문자열이며, 비즈니스는 자신의 풀필먼트 프로세스에 맞는 값이면 무엇이든 쓸 수 있습니다.

| 이름            | 타입          | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                    |
| --------------- | ------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id              | string        | **Required** | Fulfillment event identifier.                                                                                                                                                                                                                                                                                                           |
| occurred_at     | string        | **Required** | RFC 3339 timestamp when this fulfillment event occurred.                                                                                                                                                                                                                                                                                |
| type            | string        | **Required** | Fulfillment event type. Common values include: processing (preparing to ship), shipped (handed to carrier), in_transit (in delivery network), delivered (received by buyer), failed_attempt (delivery attempt failed), canceled (fulfillment canceled), undeliverable (cannot be delivered), returned_to_sender (returned to merchant). |
| line_items      | Array[object] | **Required** | Which line items and quantities are fulfilled in this event.                                                                                                                                                                                                                                                                            |
| tracking_number | string        | Optional     | Carrier tracking number (required if type != processing).                                                                                                                                                                                                                                                                               |
| tracking_url    | string        | Optional     | URL to track this shipment (required if type != processing).                                                                                                                                                                                                                                                                            |
| carrier         | string        | Optional     | Carrier name (e.g., 'FedEx', 'USPS').                                                                                                                                                                                                                                                                                                   |
| description     | string        | Optional     | Human-readable description of the shipment status or delivery information (e.g., 'Delivered to front door', 'Out for delivery').                                                                                                                                                                                                        |

예: `processing`, `shipped`, `in_transit`, `delivered`, `failed_attempt`, `canceled`, `undeliverable`, `returned_to_sender` 등.

### 조정

조정은 풀필먼트와 별개로 존재하는 다형적 이벤트입니다. `type` 필드는 개방형 문자열이며, 비즈니스는 자신에게 맞는 값이면 무엇이든 쓸 수 있습니다.

| 이름        | 타입                                                                                    | 필수 여부    | 설명                                                                                                                                                                                            |
| ----------- | --------------------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id          | string                                                                                  | **Required** | Adjustment event identifier.                                                                                                                                                                    |
| type        | string                                                                                  | **Required** | Type of adjustment (open string). Typically money-related like: refund, return, credit, price_adjustment, dispute, cancellation. Can be any value that makes sense for the merchant's business. |
| occurred_at | string                                                                                  | **Required** | RFC 3339 timestamp when this adjustment occurred.                                                                                                                                               |
| status      | string                                                                                  | **Required** | Adjustment status. **Enum:** `pending`, `completed`, `failed`                                                                                                                                   |
| line_items  | Array[object]                                                                           | Optional     | Which line items and quantities are affected (optional).                                                                                                                                        |
| totals      | Array\[[Total Response](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#total)\] | Optional     | Adjustment totals breakdown. Signed values - negative for money returned to buyer (refunds, credits), positive for additional charges (exchanges).                                              |
| description | string                                                                                  | Optional     | Human-readable reason or description (e.g., 'Defective item', 'Customer requested').                                                                                                            |

예: `refund`, `return`, `credit`, `price_adjustment`, `dispute`, `cancellation` 등.

## 예시

```json
{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "2026-08-25"}]
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

## 예시: 계량 단위 판매 상품

파운드 단위로 판매되는 바나나 주문입니다(`item.quantity_unit` `{ "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }`). 수량은 단계 수입니다. 구매자는 1.50 lb를 주문했고(`total: 150`), 그중 0.50 lb가 발송되었으므로(`fulfilled: 50`) 라인은 `partial`입니다. 이후 0.25 lb 반품은 `-25` 단계의 조정으로 기록됩니다. 금액은 $0.79/lb로 책정하고 반올림은 한 번만 합니다. 라인 합계는 `79 × 1.50 = 118.5`를 반올림한 `119`이고, 반품 크레딧은 `79 × 0.25 = 19.75`를 반올림한 `20`입니다.

```json
{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": { "dev.ucp.shopping.order": [{"version": "2026-08-25"}] }
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

## 예시: 개수로 판매하고 계량으로 정산하는 경우

어떤 상품은 개수로 판매하지만 가격은 계량으로 책정합니다. 예를 들어 사과를 $2.00/lb에 `each` 단위로 판매하고, 사과 한 개마다 명목 무게를 둡니다. 이때는 두 가지 기준이 함께 작동하며 둘 다 전달됩니다. 판매 기준은 `quantity`를 정하고(사과 개수, 수명 주기 연산은 그대로), 가격 기준인 라인의 [`unit_price`](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/checkout/#quantity-and-sale-basis)에는 요금 계산의 근거가 되는 단가와 명목 계량이 담깁니다. 가격을 실제 계량으로 정산할 때 조정에는 반드시 정산된 `measure`가 있어야 합니다(**MUST**). 그 단위 식별은 반드시 가격 기준과 같아야 하며(**MUST**)(변환 없음), 가격만 정산할 때는 개수가 바뀌지 않도록 `quantity: 0`을 사용합니다. (정산된 `measure`는 비즈니스가 기록한 사실입니다. 상품 식별을 정의하는 구매자 설정 계량은 별도로 협상하는 확장입니다.)

구매자가 사과 3개를 개당 $0.80에 주문합니다(명목 0.40 lb × $2.00/lb, 라인 합계 `240`). 피커가 잰 사과 세 개의 무게는 명목 1.20 lb보다 적은 1.14 lb이며, 정산 내용은 주문만 보고도 완전히 검증할 수 있습니다. 명목 단계 `3 × 40 = 120`, 정산 `114`, 차이 `6` 단계 `× 200 × 10^-2 = 12`이므로 `-12` 조정이 정수 연산으로 정확히 맞아떨어집니다.

```json
{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": { "dev.ucp.shopping.order": [{"version": "2026-08-25"}] }
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

개수 수명 주기는 전혀 바뀌지 않고(사과 개수 기준으로 `fulfilled == total`이 유지됩니다) 금액만 실제 계량에 맞춰 조정되며, 크레딧을 검증하는 데 필요한 숫자는 모두 주문 안에 들어 있습니다.

## 예시: 실중량(catch-weight) 조정

무게를 다는 상품은 피킹한 무게가 주문한 무게와 다른 경우가 흔합니다. 이 차이는 수치 오류가 아니라 상거래상의 사실입니다. 비즈니스는 실제 피킹 결과를 기록하고, 금액과 수량을 함께 바꾸는 [조정](#adjustments)으로 차이를 맞춥니다. 허용 오차 비교는 끼어들지 않으며, 상태 도출은 처음부터 끝까지 정확한 단계 수로 합니다.

구매자가 같은 바나나를 $0.79/lb에 2.00 lb 주문합니다. 판매 단위는 1/4파운드 증분입니다 (`quantity` `200`, 라인 합계 `79 × 2.00 = 158`). 피커가 1.90 lb를 담습니다. 증분에 맞지 않는 값이지만 있는 그대로 기록합니다. [`increment`](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#ordering-increment)는 플랫폼의 주문을 제약할 뿐, 비즈니스의 기록은 제약하지 않기 때문입니다. 풀필먼트 이벤트는 실제 값인 `190` 단계를 기록하고, `-10` 단계의 `price_adjustment`가 가격 차이(`79 × 0.10 = 7.9`를 한 번 반올림한 `8`)와 함께 `total`을 실제 피킹 값에 맞춥니다. 그러면 `fulfilled == total`이 정확히 성립하고(`190 == 190`) 라인 상태는 `fulfilled`로 도출됩니다.

```json
{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": { "dev.ucp.shopping.order": [{"version": "2026-08-25"}] }
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

## 스코프

주문 기능은 사용자 인증 접근용으로 다음과 같은 잘 알려진 스코프를 정의합니다.

| 스코프                          | 설명                                                                                      |
| ------------------------------- | ----------------------------------------------------------------------------------------- |
| `dev.ucp.shopping.order:read`   | 사용자 주문에 대한 읽기 접근 — 인증된 사용자가 소유한 리소스에 대한 주문 조회(Get Order). |
| `dev.ucp.shopping.order:manage` | 사용자 주문에 대한 구매 후 작업 — 취소, 반품, 기타 수정.                                  |

스코프 선언과 도출, 사용자 정의 스코프로 이 집합을 확장하는 규칙은 [신원 연결 — 스코프](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/identity-linking/#scopes)에 정의되어 있습니다.

## 작업

주문 엔티티는 **현재 상태 스냅샷**입니다. 즉 조회하거나 전달한 시점의 주문을 나타내는 권한 있는 최신 상태입니다. 비즈니스는 반드시 모든 응답에서 전체 주문 엔티티를 반환해야 합니다(**MUST**). 동기식 조회(이 섹션)와 비동기식 이벤트 전달([이벤트](#events) 참고)에는 같은 스키마를 사용합니다.

`permalink_url`은 타임라인, 구매 후 작업, 반품 등 전체 주문 경험을 가리키는 권한 있는 참조입니다. API는 대화형 사용 사례와 운영 사용 사례를 위해 현재 상태에 프로그래밍 방식으로 접근하는 수단을 제공합니다.

| 작업                               | 메서드 | 엔드포인트     | 설명                                  |
| ---------------------------------- | ------ | -------------- | ------------------------------------- |
| [주문 조회(Get Order)](#get-order) | `GET`  | `/orders/{id}` | 플랫폼이 현재 주문 상태를 가져옵니다. |

전송 방식별 세부 사항은 [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/order/rest/index.md)과 [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/order/mcp/index.md)을 참고하세요.

### 주문 조회

주문의 현재 상태 스냅샷을 반환합니다.

#### 인가

비즈니스는 응답을 반환하기 전에 지원되는 UCP 메커니즘(API 키, OAuth 2.0, 상호 TLS, HTTP Message Signatures 중 하나)으로 주문 데이터 요청을 반드시 인증해야 합니다(**MUST**) ([신원 확인 및 인증](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/checkout/rest/#authentication) 참고). 호출자가 접근할 수 있는 주문은 인증 방식에 따라 달라집니다.

| 인증             | 접근 가능한 주문                                     |
| ---------------- | ---------------------------------------------------- |
| 플랫폼 자격 증명 | 해당 플랫폼에서 생성된 주문                          |
| 구매자 인가      | 부여된 OAuth 스코프 범위 내에서 구매자가 소유한 주문 |

**플랫폼 자격 증명**(API 키, 서명, OAuth 클라이언트 자격 증명) - 비즈니스는 플랫폼이 생성한 주문에 대한 접근을 허용할 수 있습니다(**MAY**). 플랫폼은 체크아웃 흐름에서 구매자 정보와 결제 정보를 제공했고 주문 확인도 지켜보았으므로, 이미 맥락을 알고 있는 주문의 최신 상태를 조회하는 셈입니다.

**구매자 인가** - 플랫폼은 필요한 스코프로 [신원 연결(Identity Linking)](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/identity-linking/index.md) 또는 그와 비슷한 메커니즘으로 구매자 인가를 받습니다. 이렇게 하면 어느 플랫폼에서 생성되었는지와 관계없이 구매자의 주문에 접근할 수 있습니다.

비즈니스는 인가와 별개로 추가 접근 정책(예: 신뢰할 수 있는 파트너 계약)을 정의하고, 데이터 가용성 제약(예: 보존 기간, 규제에 따른 삭제)을 적용하고, 컨텍스트, 비즈니스 정책, 기타 요구 사항에 따라 응답에서 선택 필드를 생략하거나 가릴 수 있습니다(**MAY**).

#### 에러 응답

비즈니스가 주문을 반환할 수 없으면, 결과를 설명하는 `messages` 배열이 담긴 오류를 응답으로 반환합니다.

**주문을 찾을 수 없음:**

```json
{
  "ucp": {
    "version": "2026-08-25",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "2026-08-25"}]
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

```json
{
  "ucp": {
    "version": "2026-08-25",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "2026-08-25"}]
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

### 지침

**플랫폼:**

- 모든 요청에 프로필 URL을 담은 `UCP-Agent` 헤더를 반드시 포함해야 합니다(**MUST**)
- 주문 업데이트는 주로 웹훅([이벤트](#events) 참고)으로 받고, 주문 조회는 대사(reconciliation)나 필요할 때의 조회에만 사용해야 합니다(**SHOULD**)
- 주문 데이터는 일시적인 데이터로 취급하고, 진행 중인 커머스 흐름에 더 이상 필요하지 않으면 폐기해야 합니다(**SHOULD**)

**비즈니스:**

- 응답을 반환하기 전에 주문 데이터 요청을 반드시 인증해야 합니다(**MUST**) ([인가](#authorization) 참고)

## 이벤트

비즈니스는 주문 수명 주기 업데이트를 웹훅으로 플랫폼에 푸시합니다. 페이로드는 [작업](#operations)에서 설명한 것과 같은 **현재 상태 스냅샷**, 즉 전체 주문 엔티티입니다.

| 이벤트                                   | 메서드 | 엔드포인트          | 설명                                                  |
| ---------------------------------------- | ------ | ------------------- | ----------------------------------------------------- |
| [주문 이벤트 웹훅](#order-event-webhook) | `POST` | 플랫폼이 제공한 URL | 비즈니스가 주문 수명 주기 이벤트를 플랫폼에 보냅니다. |

### 주문 이벤트 웹훅

비즈니스는 파트너 온보딩 때 플랫폼이 제공한 웹훅 URL로 주문 이벤트를 POST합니다. URL 형식은 플랫폼마다 다릅니다.

헤더는 \*\*[Standard Webhooks](https://www.standardwebhooks.com/)\*\*를 따르되, 요청 서명만은 [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421)을 따릅니다. 자세한 내용은 [메시지 서명](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/index.md)을 참고하세요.

**필수 헤더:**

| 헤더                | 설명                         |
| ------------------- | ---------------------------- |
| `Webhook-Timestamp` | 이벤트 발생 타임스탬프(unix) |
| `Webhook-Id`        | 고유한 이벤트 식별자         |

**Inputs**

| 이름          | 타입                                                                                               | 필수 여부    | 설명                                                                                                                                          |
| ------------- | -------------------------------------------------------------------------------------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp           | any                                                                                                | **Required** | UCP metadata for order responses. No payment handlers needed post-purchase.                                                                   |
| id            | string                                                                                             | **Required** | Unique order identifier.                                                                                                                      |
| label         | string                                                                                             | Optional     | Human-readable label for identifying the order. MUST only be provided by the business.                                                        |
| checkout_id   | string                                                                                             | **Required** | Associated checkout ID for reconciliation.                                                                                                    |
| permalink_url | string                                                                                             | **Required** | Permalink to access the order on merchant site.                                                                                               |
| line_items    | Array\[[Order Line Item](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#order-line-item)\] | **Required** | Line items representing what was purchased — can change post-order via edits or exchanges.                                                    |
| fulfillment   | object                                                                                             | **Required** | Fulfillment data: buyer expectations and what actually happened.                                                                              |
| adjustments   | Array\[[Adjustment](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#adjustment)\]           | Optional     | Post-order events (refunds, returns, credits, disputes, cancellations, etc.) that exist independently of fulfillment.                         |
| totals        | [Totals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#totals)                            | **Required** | Different totals for the order.                                                                                                               |
| messages      | Array\[[Message](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message)\]                 | Optional     | Business outcome messages (errors, warnings, informational). Present when the business needs to communicate status or issues to the platform. |

**Output**

| 이름 | 타입                                                                   | 필수 여부    | 설명 |
| ---- | ---------------------------------------------------------------------- | ------------ | ---- |
| ucp  | [Ucp](/ucp-ko/2026-08-25/2026-08-25/specification/shopping/order/#ucp) | **Required** |      |

### 웹훅 URL 설정

플랫폼은 기능 협상 과정에서 주문 기능의 `config` 필드에 웹훅 URL을 넣어 둡니다. 비즈니스는 플랫폼 프로필에서 이 URL을 찾아 주문 수명 주기 이벤트를 보낼 때 사용합니다.

Platform's order capability configuration.

| 이름        | 타입   | 필수 여부    | 설명                                                        |
| ----------- | ------ | ------------ | ----------------------------------------------------------- |
| webhook_url | string | **Required** | URL where merchant sends order lifecycle events (webhooks). |

**예시:**

```json
{
  "dev.ucp.shopping.order": [
    {
      "version": "2026-08-25",
      "spec": "https://ucp.dev/2026-08-25/specification/shopping/order",
      "schema": "https://ucp.dev/2026-08-25/schemas/shopping/order.json",
      "config": {
        "webhook_url": "https://platform.example.com/webhooks/ucp/orders"
      }
    }
  ]
}
```

### 웹훅 서명 검증

진본성과 무결성을 보장하려면 웹훅 페이로드는 반드시 비즈니스가 서명하고 플랫폼이 검증해야 합니다(**MUST**). 서명은 REST 바인딩(RFC 9421)을 사용하는 [메시지 서명](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/index.md) 명세를 따릅니다.

**필수 헤더:**

| 헤더              | 설명                                     |
| ----------------- | ---------------------------------------- |
| `UCP-Agent`       | 비즈니스 프로필 URL(RFC 8941 Dictionary) |
| `Signature-Input` | 서명된 구성 요소 기술                    |
| `Signature`       | 서명 값                                  |
| `Content-Digest`  | 본문 다이제스트(RFC 9530)                |

**웹훅 요청 예시:**

```http
POST /webhooks/ucp/orders HTTP/1.1
Host: platform.example.com
Content-Type: application/json
UCP-Agent: profile="https://merchant.example/.well-known/ucp"
Content-Digest: sha-256=:X48E9q...:
Signature-Input: sig1=("@method" "@authority" "@path" "content-digest" "content-type");keyid="merchant-2026"
Signature: sig1=:MEUCIQDTxNq8h7LGHpvVZQp1iHkFp9+3N8Mxk2zH1wK4YuVN8w...:

{"id":"order_abc123","event_id":"evt_123","created_time":"2026-01-15T12:00:00Z",...}
```

#### 서명(비즈니스)

1. 원시 요청 본문의 SHA-256 다이제스트를 계산하고 `Content-Digest` 헤더를 설정합니다
1. [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421)에 따라 서명 기반을 만듭니다
1. 비즈니스 UCP 프로필의 `keys`에 있는 키로 서명합니다
1. `Signature-Input`과 `Signature` 헤더를 설정합니다

전체 알고리즘은 [메시지 서명 - REST 요청 서명](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/#rest-request-signing)을 참고하세요.

#### 검증(플랫폼)

**인증**(서명 검증):

1. `Signature-Input`을 파싱해 `keyid`와 서명된 구성 요소를 추출합니다
1. `/.well-known/ucp`에서 비즈니스의 UCP 프로필을 가져옵니다(적절히 캐시)
1. `keys`에서 `kid`가 일치하는 키를 찾습니다
1. `Content-Digest`가 원시 본문의 SHA-256과 일치하는지 확인합니다
1. 서명 기반을 재구성하고 서명을 검증합니다

전체 알고리즘은 [메시지 서명 - REST 요청 검증](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/#rest-request-verification)을 참고하세요.

**인가**(주문 소유권):

서명을 검증한 뒤 플랫폼은 반드시 서명자에게 참조된 주문의 이벤트를 보낼 권한이 있는지 확인해야 합니다(**MUST**).

1. 웹훅 페이로드에서 주문 ID를 추출합니다
1. 해당 주문이 이 비즈니스에서 생성되었는지 확인합니다(프로필 URL 일치)
1. 서명자의 프로필이 주문의 비즈니스와 일치하지 않는 웹훅은 거부합니다

이렇게 하면 악의적인 비즈니스가 유효한 서명을 갖고 있더라도 다른 비즈니스의 주문에 대한 가짜 이벤트를 보낼 수 없습니다.

#### 키 순환

무중단 키 순환 절차는 [메시지 서명 - 키 순환](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/#key-rotation)을 참고하세요.

### 지침

**플랫폼:**

- 웹훅을 받았음을 알리기 위해 반드시 2xx HTTP 상태 코드로 빠르게 응답해야 하며(**MUST**), 이벤트는 응답한 뒤 비동기로 처리합니다

**비즈니스:**

- 서명자를 식별할 수 있도록 프로필 URL을 담은 `UCP-Agent` 헤더를 반드시 포함해야 합니다(**MUST**)
- 모든 웹훅 페이로드를 [메시지 서명](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/index.md) 명세에 따라 RFC 9421 헤더 (`Signature`, `Signature-Input`, `Content-Digest`)로 반드시 서명해야 합니다(**MUST**)
- 모든 필드를 채운 주문 엔티티와 함께 "주문 생성" 이벤트를 반드시 보내야 합니다(**MUST**)
- 업데이트할 때는 반드시 전체 주문 엔티티를 보내야 합니다(증분 변경분이 아님)(**MUST**)
- 실패한 웹훅 전달은 반드시 재시도해야 합니다(**MUST**)

## 엔티티

### 상품

| 이름          | 타입                                                                                  | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------- | ------------------------------------------------------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id            | string                                                                                | **Required** | The product identifier, often the SKU, required to resolve the product details associated with this line item. Should be recognized by both the Platform, and the Business.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| title         | string                                                                                | **Required** | Product title.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| price         | [Amount](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#amount)               | **Required** | Unit price in ISO 4217 minor units. Price is the amount per one whole `quantity_unit.unit` (for example, per lb or per hour); when `quantity_unit` is absent, it is per `each`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| quantity_unit | [Quantity Unit](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#quantity-unit) | Optional     | Sale basis this item's `quantity` is denominated in. On an authoritative Business response, absence encodes the default `each` machine identity (`C62`, 0); the Business MUST include this descriptor for every non-`each` response. On Platform requests, omission makes no assertion: the Business interprets `quantity` using the item's authoritative sale basis. If the Platform includes this descriptor, it asserts the unit-descriptor machine identity. The Business MUST compare that machine identity (`unit`, effective `scale`), ignore `display_text` and `increment`, and resolve a mismatch by conversion surfaced as a visible line revision with a warning, or by rejection with a recoverable business outcome; silent reinterpretation is forbidden. An explicit `C62` descriptor at effective scale 0 matches an authoritative basis represented by an absent descriptor. |
| unit_price    | [Unit Price](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#unit-price)       | Optional     | Pricing basis for this item. On an authoritative Business response, the Business MUST include `unit_price` on every line whose pricing basis differs from its sale basis (for example, priced per pound but sold per `each`); presence on a line marks the rate as transactional rather than display-only. When the pricing basis is the sale basis, `item.price` fully denominates the charge and this field MAY be omitted.                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| image_url     | string                                                                                | Optional     | Product image URI.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |

### 우편 주소

| 이름             | 타입   | 필수 여부 | 설명                                                                                                                                                                                                                                      |
| ---------------- | ------ | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| extended_address | string | Optional  | An address extension such as an apartment number, C/O or alternative name.                                                                                                                                                                |
| street_address   | string | Optional  | The street address.                                                                                                                                                                                                                       |
| address_locality | string | Optional  | The locality in which the street address is, and which is in the region. For example, Mountain View.                                                                                                                                      |
| address_region   | string | Optional  | The region in which the locality is, and which is in the country. Required for applicable countries (i.e. state in US, province in CA). For example, California or another appropriate first-level Administrative division.               |
| address_country  | string | Optional  | The country. Recommended to be in 2-letter ISO 3166-1 alpha-2 format, for example "US". For backward compatibility, a 3-letter ISO 3166-1 alpha-3 country code such as "SGP" or a full country name such as "Singapore" can also be used. |
| postal_code      | string | Optional  | The postal code. For example, 94043.                                                                                                                                                                                                      |
| first_name       | string | Optional  | Optional. First name of the contact associated with the address.                                                                                                                                                                          |
| last_name        | string | Optional  | Optional. Last name of the contact associated with the address.                                                                                                                                                                           |
| phone_number     | string | Optional  | Optional. Phone number of the contact associated with the address.                                                                                                                                                                        |

### 응답

Capability reference in responses. Only name/version required to confirm active capabilities.

| 이름    | 타입                       | 필수 여부    | 설명                                                                                                                            |
| ------- | -------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| version | string                     | **Required** | Entity version in YYYY-MM-DD format.                                                                                            |
| spec    | string                     | Optional     | URL to human-readable specification document.                                                                                   |
| schema  | string                     | Optional     | URL to JSON Schema defining this entity's structure and payloads.                                                               |
| id      | string                     | Optional     | Unique identifier for this entity instance. Used to disambiguate when multiple instances exist.                                 |
| config  | object                     | Optional     | Entity-specific configuration. Structure defined by each entity's schema.                                                       |
| extends | OneOf\[`string`, `array`\] | Optional     | Parent capability(s) this extends. Present for extensions, absent for root capabilities. Use array for multi-parent extensions. |

### 합계

| 이름         | 타입                                                                                  | 필수 여부    | 설명                                                                                                                                                                                                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                                | **Required** | Cost category. Well-known values: subtotal, items_discount, discount, fulfillment, tax, fee, total. Businesses MAY use additional values.                                                                                                                                                   |
| display_text | string                                                                                | Optional     | Text to display against the amount. Should reflect appropriate method (e.g., 'Shipping', 'Delivery').                                                                                                                                                                                       |
| amount       | [Signed Amount](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#signed-amount) | **Required** | Monetary amount in the currency's minor unit as defined by ISO 4217. Refer to the currency's exponent to determine minor-to-major ratio (e.g., 2 for USD, 0 for JPY, 3 for KWD). May be negative — the sign is intrinsic to the value (e.g., discounts are negative, charges are positive). |

### UCP 응답 주문 스키마

UCP metadata for order responses. No payment handlers needed post-purchase.

| 이름             | 타입   | 필수 여부    | 설명                                                                                                                                           |
| ---------------- | ------ | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| version          | string | **Required** | Version identifier in YYYY-MM-DD format.                                                                                                       |
| map_order        | object | Optional     | Preferred key-traversal order for sibling registry fields inside the root `ucp` envelope (`services`, `capabilities`, and `payment_handlers`). |
| status           | string | Optional     | Application-level status of the UCP operation. **Enum:** `success`, `error`                                                                    |
| services         | object | Optional     | Service registry keyed by reverse-domain name.                                                                                                 |
| capabilities     | object | Optional     | Capability registry keyed by reverse-domain name.                                                                                              |
| payment_handlers | object | Optional     | Payment handler registry keyed by reverse-domain name.                                                                                         |
| capabilities     | any    | Optional     |                                                                                                                                                |
