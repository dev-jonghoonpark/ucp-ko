# 할인 확장

## 개요

할인 확장을 사용하면 비즈니스가 장바구니와 체크아웃 세션에서 할인 코드를 지원한다고 알릴 수 있습니다. 이 확장은 플랫폼과 비즈니스가 할인 코드를 주고받는 방식도 명세합니다.

**주요 특징:**

- 하나 이상의 할인 코드 제출
- 사람이 읽을 수 있는 제목과 금액이 포함된 적용 할인 수신
- 거부된 코드는 상세 에러 코드와 함께 `messages[]`로 전달
- 코드 기반 할인과 함께 자동 할인도 노출

**의존성:**

- 장바구니 기능 또는 체크아웃 기능

## 디스커버리

비즈니스는 프로필에서 할인 지원 여부를 알립니다. 이 기능은 장바구니, 체크아웃 또는 둘 다를 확장할 수 있습니다.

```json
{
  "ucp": {
    "version": "2026-04-08",
    "capabilities": {
      "dev.ucp.shopping.discount": [
        {
          "version": "2026-04-08",
          "extends": ["dev.ucp.shopping.cart", "dev.ucp.shopping.checkout"],
          "spec": "https://ucp.dev/2026-04-08/specification/discount",
          "schema": "https://ucp.dev/2026-04-08/schemas/shopping/discount.json"
        }
      ]
    }
  }
}
```

비즈니스는 장바구니만, 체크아웃만, 또는 둘 다에 할인 지원을 알릴 수 있습니다(MAY). 플랫폼은 할인 코드를 제출하기 전에 어떤 리소스가 확장되었는지 확인해야 합니다(SHOULD).

## 스키마

이 기능이 활성화되면 장바구니나 체크아웃(또는 둘 다)에 `discounts` 객체가 추가됩니다.

### Discounts 객체

할인 코드 입력과 적용된 할인 출력.

| 이름    | 타입          | 필수   | 설명                                                                                                          |
| ------- | ------------- | ------ | ------------------------------------------------------------------------------------------------------------- |
| codes   | Array[string] | 아니요 | 적용할 할인 코드. 대소문자를 구분하지 않습니다. 이전에 제출한 코드를 대체합니다. 지우려면 빈 배열을 보내세요. |
| applied | Array[object] | 아니요 | 성공적으로 적용된 할인(코드 기반 및 자동).                                                                    |

### 적용된 할인

성공적으로 적용된 할인.

| 이름        | 타입          | 필수   | 설명                                                                                                                                                      |
| ----------- | ------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| code        | string        | 아니요 | 할인 코드. 자동 할인에서는 생략됩니다.                                                                                                                    |
| title       | string        | **예** | 사람이 읽을 수 있는 할인 이름(예: 'Summer Sale 20% Off').                                                                                                 |
| amount      | integer       | **예** | ISO 4217 최소 단위로 표시한 총 할인 금액.                                                                                                                 |
| automatic   | boolean       | 아니요 | 판매자 규칙에 따라 자동으로 적용되면(코드 불필요) true.                                                                                                   |
| method      | string        | 아니요 | 배분 방식. 'each' = 항목마다 독립적으로 적용. 'across' = 금액에 비례해 나눔. **허용 값:** `each`, `across`                                                |
| priority    | integer       | 아니요 | 할인 계산 시 적용 순서. 숫자가 낮을수록 먼저 적용됩니다(1 = 첫 번째).                                                                                     |
| provisional | boolean       | 아니요 | 이 할인에 추가 검증이 필요하면 true.                                                                                                                      |
| eligibility | string        | 아니요 | 비즈니스가 이 할인에 대해 인정한 적격성 주장. context.eligibility의 값 중 하나에 해당합니다. 코드 기반 할인과 적격성과 무관한 자동 할인에서는 생략됩니다. |
| allocations | Array[object] | 아니요 | 이 할인이 배분된 위치 내역. 배분 금액의 합은 총액과 같습니다.                                                                                             |

### 배분

할인 금액이 특정 대상에 어떻게 배분되었는지 보여 주는 내역.

| 이름   | 타입    | 필수   | 설명                                                                       |
| ------ | ------- | ------ | -------------------------------------------------------------------------- |
| path   | string  | **예** | 할당 대상을 가리키는 JSONPath(예: '$.line_items[0]', '$.totals.shipping'). |
| amount | integer | **예** | 이 대상에 배분된 금액(ISO 4217 최소 통화 단위).                            |

## 배분 세부 사항

`applied` 배열은 할인이 어떻게 계산되고 배분되었는지 설명합니다. `applied[].amount`는 적용된 할인의 크기이고(항상 양수), 이에 대응하는 `totals[]` 항목의 금액은 영수증에 반영되는 부호 있는 값입니다(할인은 음수).

### 배분 방식

`method` 필드는 할인을 계산한 방식을 나타냅니다.

| 방식     | 의미                          | 예시                                           |
| -------- | ----------------------------- | ---------------------------------------------- |
| `each`   | 대상 상품마다 독립적으로 적용 | "각 상품 10% 할인" → 10% × 상품 가격           |
| `across` | 금액에 비례하여 분할          | "주문 $10 할인" → $60 상품에 $6, $40 상품에 $4 |

### 중복 적용 순서

여러 할인이 적용될 때는 `priority`가 계산 순서를 나타냅니다. 숫자가 작을수록 먼저 적용됩니다.

```text
Cart: $100
Discount A (priority: 1): 20% off → $100 × 0.8 = $80
Discount B (priority: 2): $10 off → $80 - $10 = $70
```

퍼센트 할인은 언제 적용하느냐에 따라 누적 결과가 달라지므로 순서가 중요합니다.

### Allocations 배열

`allocations` 배열은 JSONPath로 대상을 지정해 할인 금액이 각각 어디에 반영되었는지 세분화해 보여 줍니다.

| 경로 패턴           | 대상                |
| ------------------- | ------------------- |
| `$.line_items[0]`   | 첫 번째 라인 아이템 |
| `$.line_items[1]`   | 두 번째 라인 아이템 |
| `$.totals.shipping` | 배송비              |

덕분에 플랫폼은 여러 할인이 중복 적용되더라도 각 할인이 라인 아이템마다 얼마씩 기여했는지 정확히 설명할 수 있습니다.

**불변 조건:** `allocations[].amount`의 합은 `applied_discount.amount`와 같습니다.

## 작업

할인 코드는 표준 장바구니 또는 체크아웃 생성/수정 작업으로 제출합니다. 두 리소스 모두 같은 의미 체계를 따릅니다.

**요청 동작:**

- **교체 의미 체계**: `discounts.codes`를 제출하면 이전에 제출한 코드를 모두 대체합니다.
- **코드 초기화**: 빈 배열 `"codes": []`를 보내면 모든 할인 코드가 제거됩니다.
- **대소문자 무시**: 비즈니스는 대소문자를 구분하지 않고 코드를 매칭합니다.

**응답 동작:**

- `discounts.applied`에는 활성화된 모든 할인(코드 기반 + 자동)이 포함됩니다.
- 거부된 코드는 `messages[]`로 전달됩니다(아래 참고).
- 할인 금액은 `totals[]`와 `line_items[].totals[]`에 반영됩니다.

**장바구니에서 체크아웃으로의 연속성:** 장바구니 기능의 `cart_id` 필드로 장바구니가 체크아웃으로 전환되면, 비즈니스는 반드시 장바구니에 적용된 할인 코드를 그대로 이어 가야 합니다(MUST). 체크아웃 시점에 더 이상 유효하지 않은 코드(예: 만료, 자격 미달)는 표준 거부 코드를 사용해 `messages[]`로 전달해야 합니다(SHOULD).

## 거부된 코드

제출된 할인 코드를 적용할 수 없으면 비즈니스는 `messages[]` 배열로 그 사실을 알립니다.

```json
{
  "messages": [
    {
      "type": "warning",
      "code": "discount_code_expired",
      "path": "$.discounts.codes[0]",
      "content": "Code 'SUMMER20' expired on December 1st"
    }
  ]
}
```

> **구현 지침:** 주문 합계나 합계에 대한 사용자의 기대에 영향을 주는 작업에는 `type: "warning"`을 사용해, 플랫폼이 조용히 넘기지 않고 사용자에게 보여 주도록 해야 합니다(**SHOULD**). 거부된 할인이 대표적인 예입니다. 사용자는 할인을 기대했지만 실제로는 받지 못하므로 그 사실을 알려야 합니다.

**거부된 할인의 에러 코드:**

| 코드                                   | 설명                                 |
| -------------------------------------- | ------------------------------------ |
| `discount_code_expired`                | 코드가 만료됨                        |
| `discount_code_invalid`                | 코드를 찾을 수 없거나 형식이 잘못됨  |
| `discount_code_already_applied`        | 코드가 이미 적용됨                   |
| `discount_code_combination_disallowed` | 다른 활성 할인과 함께 사용할 수 없음 |
| `discount_code_user_not_logged_in`     | 인증된 사용자만 사용할 수 있는 코드  |
| `discount_code_user_ineligible`        | 사용자가 자격 요건을 충족하지 않음   |

## 자동 할인

비즈니스는 장바구니 내용, 고객 세그먼트, 프로모션 규칙에 따라 할인을 자동으로 적용할 수 있습니다. 자동 할인의 특징은 다음과 같습니다.

- `discounts.applied`에 `automatic: true`로 나타나며 `code` 필드가 없습니다.
- 플랫폼이 아무 조치를 하지 않아도 적용됩니다.
- 플랫폼이 제거할 수 없습니다.
- 투명성을 위해 노출됩니다(플랫폼은 할인이 적용된 이유를 사용자에게 설명할 수 있습니다).

## 자격 주장

자격 주장(eligibility claim)은 로열티 멤버십, 결제 수단 혜택처럼 구매자가 자신이 받을 수 있다고 내세우는 혜택입니다([컨텍스트](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#context) 참고). 할인 확장이 활성화된 상태에서 자격 주장을 수락하기로 한 비즈니스는 반드시 그로 인한 가격 변동을 `applied` 배열에 잠정 할인으로 노출해야 합니다(**MUST**). 플랫폼은 반드시 잠정 할인을 구매자에게 표시해야 합니다(**MUST**).

### 할인 동작

플랫폼은 장바구니 또는 체크아웃 요청의 `context.eligibility`로 구매자의 주장을 보냅니다([컨텍스트](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#context) 참고). 비즈니스가 주장을 인식했고 그 주장이 가격에 영향을 준다면, 비즈니스는 반드시 `discounts.applied` 배열에 해당 잠정 할인을 노출해야 합니다(**MUST**). 그러면 플랫폼은 구매자에게 표시할 구조화된 귀속 정보를 얻게 됩니다.

자격 주장으로 적용된 할인은 다음 필드를 사용합니다.

| 필드          | 값                         | 목적              |
| ------------- | -------------------------- | ----------------- |
| `automatic`   | `true`                     | 코드가 필요 없음  |
| `provisional` | `true`                     | 완료 시 검증 필요 |
| `eligibility` | `"com.example.store_card"` | 수락된 주장       |
| `code`        | *(생략)*                   | 코드 기반이 아님  |

다른 할인과 중복 적용할 때는 표준 `priority`, `method`, `allocations` 필드를 그대로 씁니다.

### 체크아웃 시 검증

수락되었지만 아직 검증되지 않은 주장에서 나온 할인에는 `provisional: true`가 붙습니다. 잠정 할인은 세션 중에 주장이 검증되거나 철회되거나 대체될 때까지 유지됩니다. 체크아웃 완료 시점에는 남아 있는 잠정 주장을 반드시 모두 해소해야 합니다(**MUST**) ([완료 시 자격 검증](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#eligibility-verification-at-completion) 참고).

### 예시: 귀속 정보가 포함된 잠정 할인

[완료 시 자격 검증](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#eligibility-verification-at-completion)의 스토어 카드 예시를 바탕으로, 할인 확장은 구조화된 귀속 정보를 제공합니다. 플랫폼이 스토어 카드 혜택을 주장하면 비즈니스는 중복 적용 및 배분 세부 정보를 모두 담아 잠정 할인을 노출합니다.

```json
{
  "context": {
    "eligibility": ["com.example.store_card"]
  },
  "line_items": [
    {
      "item": {
        "id": "prod_shirt"
      },
      "quantity": 2
    }
  ]
}
```

```json
{
  "discounts": {
    "applied": [
      {
        "title": "Store Card 5% Off",
        "amount": 250,
        "automatic": true,
        "provisional": true,
        "eligibility": "com.example.store_card",
        "priority": 1,
        "method": "each",
        "allocations": [
          {"path": "$.line_items[0]", "amount": 250}
        ]
      }
    ]
  },
  "totals": [
    {"type": "subtotal", "display_text": "Subtotal", "amount": 5000},
    {"type": "items_discount", "display_text": "Discounts", "amount": -250},
    {"type": "total", "display_text": "Total", "amount": 4750}
  ]
}
```

이제 플랫폼은 귀속 정보, 금액, 배분을 믿고 "Store Card 5% Off: -$2.50 *(구매 시 검증)*"처럼 렌더링할 수 있습니다.

## 라인 아이템과 합계에 미치는 영향

적용된 할인은 서로 다른 두 가지 합계 유형으로 핵심 장바구니 또는 체크아웃 필드에 반영됩니다.

| 합계 유형        | 사용 시점                                      |
| ---------------- | ---------------------------------------------- |
| `items_discount` | 라인 아이템에 배분된 할인(`$.line_items[*]`)   |
| `discount`       | 주문 수준 할인(배송비, 수수료, 주문 정액 할인) |

**유형 결정:** 할인에 라인 아이템을 가리키는 `allocations`가 있으면 `items_discount`에 반영됩니다. 배분이 없거나 배송비/수수료에 배분된 할인은 `discount`에 반영됩니다.

| 할인 유형        | 반영 위치                                  |
| ---------------- | ------------------------------------------ |
| 라인 아이템 할인 | `line_items[].totals[type=items_discount]` |
| 주문 수준 할인   | `totals[type=discount]`                    |

**불변 조건:** `totals[type=items_discount].amount`는 `sum(line_items[].totals[type=items_discount].amount)`와 같습니다.

`discounts.applied` 배열은 **무엇이** 적용되었는지 보여 줍니다. `totals[]`와 `line_items[].totals[]`는 **어디에** **얼마나** 적용되었는지 보여 줍니다.

**금액 규칙:** `discounts.applied`의 할인 금액은 양의 정수(할인 값)입니다. `totals[]`의 할인 항목은 음수(영수증에 반영되는 값)이며, 부호는 스키마가 강제합니다.

## 예시

### 할인 코드가 적용된 장바구니

장바구니를 살펴보는 중에 할인 코드를 적용한 경우입니다. 장바구니 응답에 예상 할인 금액이 포함되므로 구매자는 체크아웃으로 넘어가기 전에 절약 금액을 확인할 수 있습니다.

```json
{
  "line_items": [
    {
      "item": {
        "id": "prod_1",
      },
      "quantity": 2
    }
  ],
  "discounts": {
    "codes": ["SUMMER20"]
  }
}
```

```json
{
  "id": "cart_abc123",
  "line_items": [
    {
      "id": "li_1",
      "item": {
        "id": "prod_1",
        "title": "T-Shirt",
        "price": 2000
      },
      "quantity": 2,
      "totals": [
        {"type": "subtotal", "amount": 4000},
        {"type": "items_discount", "amount": -800},
        {"type": "total", "amount": 3200}
      ]
    }
  ],
  "discounts": {
    "codes": ["SUMMER20"],
    "applied": [
      {
        "code": "SUMMER20",
        "title": "Summer Sale 20% Off",
        "amount": 800,
        "method": "each",
        "allocations": [
          {"path": "$.line_items[0]", "amount": 800}
        ]
      }
    ]
  },
  "currency": "USD",
  "totals": [
    {"type": "subtotal", "display_text": "Subtotal", "amount": 4000},
    {"type": "items_discount", "display_text": "Item Discounts", "amount": -800},
    {"type": "total", "display_text": "Estimated Total", "amount": 3200}
  ]
}
```

### 주문 수준 할인

주문 합계에 정액 할인을 적용한 경우입니다. 배분은 없습니다. 할인은 주문 전체에 적용되며 합계에서 `type: "discount"`를 사용합니다.

```json
{
  "discounts": {
    "codes": ["SAVE10"]
  }
}
```

```json
{
  "discounts": {
    "codes": ["SAVE10"],
    "applied": [
      {
        "code": "SAVE10",
        "title": "$10 Off Your Order",
        "amount": 1000
      }
    ]
  },
  "totals": [
    {"type": "subtotal", "display_text": "Subtotal", "amount": 5000},
    {"type": "discount", "display_text": "Order Discount", "amount": -1000},
    {"type": "total", "display_text": "Total", "amount": 4000}
  ]
}
```

### 혼합 할인(상품 + 주문 수준)

이 예시는 두 가지 할인 유형을 모두 보여 줍니다. 라인 아이템에 배분된 상품별 할인(20% 할인)과 주문 수준의 자동 배송비 할인입니다.

```json
{
  "discounts": {
    "codes": ["SUMMER20"]
  }
}
```

```json
{
  "line_items": [
    {
      "id": "li_1",
      "item": {
        "id": "prod_1",
        "title": "T-Shirt",
        "price": 2000
      },
      "quantity": 2,
      "totals": [
        {"type": "subtotal", "amount": 4000},
        {"type": "items_discount", "amount": -800},
        {"type": "total", "amount": 3200}
      ]
    }
  ],
  "discounts": {
    "codes": ["SUMMER20"],
    "applied": [
      {
        "code": "SUMMER20",
        "title": "Summer Sale 20% Off",
        "amount": 800,
        "allocations": [
          {"path": "$.line_items[0]", "amount": 800}
        ]
      },
      {
        "title": "Free shipping on orders over $30",
        "amount": 599,
        "automatic": true
      }
    ]
  },
  "totals": [
    {"type": "subtotal", "display_text": "Subtotal", "amount": 4000},
    {"type": "items_discount", "display_text": "Item Discounts", "amount": -800},
    {"type": "discount", "display_text": "Order Discounts", "amount": -599},
    {"type": "fulfillment", "display_text": "Shipping", "amount": 0},
    {"type": "total", "display_text": "Total", "amount": 2601}
  ]
}
```

### 거부된 할인 코드

할인 코드를 적용할 수 없으면 거부 사실이 `messages[]` 배열로 전달됩니다. 해당 코드는 `discounts.codes`에는 그대로 되돌아오지만 `discounts.applied`에는 나타나지 않습니다.

```json
{
  "discounts": {
    "codes": ["SAVE10", "EXPIRED50"]
  }
}
```

```json
{
  "discounts": {
    "codes": ["SAVE10", "EXPIRED50"],
    "applied": [
      {
        "code": "SAVE10",
        "title": "$10 Off Your Order",
        "amount": 1000
      }
    ]
  },
  "totals": [
    {"type": "subtotal", "display_text": "Subtotal", "amount": 5000},
    {"type": "discount", "display_text": "Order Discount", "amount": -1000},
    {"type": "total", "display_text": "Total", "amount": 4000}
  ],
  "messages": [
    {
      "type": "warning",
      "code": "discount_code_expired",
      "path": "$.discounts.codes[1]",
      "content": "Code 'EXPIRED50' expired on December 1st"
    }
  ]
}
```

### 배분이 포함된 중복 할인

여러 할인이 적용된 경우의 전체 배분 내역입니다.

```json
{
  "line_items": [
    {
      "id": "li_1",
      "item": {
        "id": "prod_1",
        "title": "T-Shirt",
        "price": 6000
      },
      "quantity": 1,
      "totals": [
        {"type": "subtotal", "amount": 6000},
        {"type": "items_discount", "amount": -1500},
        {"type": "total", "amount": 4500}
      ]
    },
    {
      "id": "li_2",
      "item": {
        "id": "prod_2",
        "title": "Socks",
        "price": 4000
      },
      "quantity": 1,
      "totals": [
        {"type": "subtotal", "amount": 4000},
        {"type": "items_discount", "amount": -1000},
        {"type": "total", "amount": 3000}
      ]
    }
  ],
  "discounts": {
    "codes": ["SUMMER20", "LOYALTY5"],
    "applied": [
      {
        "code": "SUMMER20",
        "title": "Summer Sale 20% Off",
        "amount": 2000,
        "method": "each",
        "priority": 1,
        "allocations": [
          {"path": "$.line_items[0]", "amount": 1200},
          {"path": "$.line_items[1]", "amount": 800}
        ]
      },
      {
        "code": "LOYALTY5",
        "title": "$5 Loyalty Reward",
        "amount": 500,
        "method": "across",
        "priority": 2,
        "allocations": [
          {"path": "$.line_items[0]", "amount": 300},
          {"path": "$.line_items[1]", "amount": 200}
        ]
      }
    ]
  },
  "totals": [
    {"type": "subtotal", "display_text": "Subtotal", "amount": 10000},
    {"type": "items_discount", "display_text": "Item Discounts", "amount": -2500},
    {"type": "total", "display_text": "Total", "amount": 7500}
  ]
}
```

이 데이터가 있으면 에이전트는 다음과 같이 설명할 수 있습니다.

> "티셔츠($60)는 20% 여름 세일로 $12, 로열티 리워드로 $3(비례 분할)가 할인되었습니다. 이 상품의 총 할인 금액은 $15입니다.
