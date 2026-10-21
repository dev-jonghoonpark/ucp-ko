# 로열티 확장

## 개요

로열티 확장은 충실도 높은 로열티 경험을 지원하도록 설계되었습니다. 기존 로열티 회원은 에이전트 기반 카탈로그·장바구니·체크아웃 경험에서도 자신의 혜택을 매끄럽게 이용할 수 있습니다. 구매자는 구매를 확정하기 전에 자신의 등급, 받을 수 있는 리워드, 즉시 적용되는 혜택을 확인할 수 있습니다. 덕분에 프로그램 회원이 당연히 기대하는 수준을 충족하고 체크아웃 퍼널의 마찰을 없앨 수 있습니다.

구체적으로는 알려진 회원이 자신의 혜택을 인식하는 것과 관련된 다음 핵심 사용 사례를 다룹니다.

- 가격에 영향을 주는 혜택: 회원 전용 할인과 무료 배송 혜택을 실시간으로 적용하고, 혜택의 출처를 명확히 표시합니다.
- 가격 외 혜택: 적립된 리워드나 향후 구매에 쓸 수 있는 리워드를 투명하게 보여 줍니다.
- 등급 인식: 프로그램 안에서 구매자의 로열티 등급을 확인하고 표시합니다.

## 핵심 개념

로열티는 네 가지 주요 구성 요소로 이루어집니다.

**멤버십**: 사용자가 가입할 수 있는 별개의 가입 경로 또는 프로그램 범주

- 같은 브랜드가 제공하더라도 독립된 프로그램(예: "Rewards Club"과 "제휴 신용카드")은 각각 독립적으로 검증할 수 있는 별도 멤버십으로 모델링합니다. 구조상으로는 로열티 확장 맵 안에서 역도메인 이름으로 네임스페이스를 구분한, 서로 형제 관계인 별도의 최상위 키로 표현합니다.

**등급(Tier)**: 멤버십 안에서 활동이나 지출이 쌓일수록 더 큰 가치를 열어 주는 특정 달성 순위 또는 등급 마일스톤

- 회원의 활성 등급은 보통 멤버십마다 하나입니다. 다만 등급 차원이 병렬로 있는 프로그램(예: "Gold"와 "Lifetime Platinum"을 동시에 보유)에서는 여러 등급이 동시에 활성화될 수 있습니다.

**혜택(Benefit)**: 고객의 현재 등급이나 멤버십 상태에 따라 부여되는 지속적인 특전과 권한

- 지연형 혜택(예: "회원은 전용 고객 서비스를 이용할 수 있음")과 즉시 가치형 혜택(예: "회원은 5% 할인을 받음")을 모두 포함합니다.

**리워드(Reward)**: 현재 거래에서 적립될 수 있는 정량화 가능한 로열티 가치. 참고: 사용 가능한 잔액과 충전식 가치(stored value)는 이 로열티 확장이 아니라 협상된 결제 수단이나 향후 도입될 사용(redemption) 기능에서 모델링합니다.

- 하나의 멤버십에서 여러 종류의 적립·수집형 리워드를 제공할 수 있으며, 리워드마다 고유한 사용 및 교환 규칙이 있습니다.

```json
{
  "loyalty": {
    "com.example.loyalty": {
      "id": "membership_1",
      "name": "Example Rewards",
      "tiers": [
        {
          "id": "gold",
          "name": "Gold",
          "benefits": [
            { "id": "BEN_001", "description": "Early access to sales" }
          ]
        }
      ],
      "rewards": [
        {
          "currency": { "name": "LoyaltyStars", "code": "LST" }
        }
      ],
      "provisional": false
    }
  }
}
```

## 디스커버리

비즈니스는 표준 공지 메커니즘에 따라 비즈니스 프로필에서 로열티 지원을 알릴 수 있습니다. 현재 로열티 확장으로 확장할 수 있는 기능은 카탈로그 검색, 카탈로그 조회, 장바구니, 체크아웃입니다. 비즈니스는 이 중 원하는 기능에만 로열티 지원을 선언할 수 있습니다(MAY). 플랫폼은 어떤 리소스가 확장되었는지 확인해야 합니다(SHOULD).

```json
{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": {
      "dev.ucp.shopping.catalog.search": [
        {
          "version": "2026-08-25",
          "spec": "https://ucp.dev/2026-08-25/specification/shopping/catalog/search",
          "schema": "https://ucp.dev/2026-08-25/schemas/shopping/catalog_search.json"
        }
      ],
      "dev.ucp.shopping.catalog.lookup": [
        {
          "version": "2026-08-25",
          "spec": "https://ucp.dev/2026-08-25/specification/shopping/catalog/lookup",
          "schema": "https://ucp.dev/2026-08-25/schemas/shopping/catalog_lookup.json"
        }
      ],
      "dev.ucp.shopping.cart": [
        {
          "version": "2026-08-25",
          "spec": "https://ucp.dev/2026-08-25/specification/shopping/cart",
          "schema": "https://ucp.dev/2026-08-25/schemas/shopping/cart.json"
        }
      ],
      "dev.ucp.shopping.checkout": [
        {
          "version": "2026-08-25",
          "spec": "https://ucp.dev/2026-08-25/specification/shopping/checkout",
          "schema": "https://ucp.dev/2026-08-25/schemas/shopping/checkout.json"
        }
      ],
      "dev.ucp.common.loyalty": [
        {
          "version": "2026-08-25",
          "extends": [
            "dev.ucp.shopping.catalog.search",
            "dev.ucp.shopping.catalog.lookup",
            "dev.ucp.shopping.cart",
            "dev.ucp.shopping.checkout"
          ],
          "spec": "https://ucp.dev/2026-08-25/specification/common/extensions/loyalty",
          "schema": "https://ucp.dev/2026-08-25/schemas/common/loyalty.json"
        }
      ]
    }
  }
}
```

**의존성:**

- 카탈로그 검색 기능
- 카탈로그 조회 기능
- 장바구니 기능
- 체크아웃 기능

## 스키마

### 엔터티

#### 로열티

키는 구매자/플랫폼이 주장하는 자격 조건 클레임을, 값은 관련 멤버십 정보를 나타내는 키-값 맵. 여러 확장이 공유 네임스페이스에 값을 넣을 때 출처를 보장하고 충돌을 막기 위해 모든 로열티 키는 반드시 역 도메인 명명을 사용해야 합니다(MUST).

#### 로열티 멤버십

상위 맵 키가 나타내는 자격 조건 클레임에 대해 비즈니스가 수락한 로열티 멤버십. 독립적으로 가입할 수 있는 프로그램은 반드시 로열티 맵 아래에 별도의 형제 항목으로 모델링하고, 역 도메인 명명(예: 'com.example.rewards'와 'com.example.rewards.card')으로 구분해야 합니다(MUST).

| 이름        | 타입          | 필수 여부 | 설명                                                                                                                                                                                                                                       |
| ----------- | ------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| id          | string        | **필수**  | 고유 로열티 멤버십 식별자.                                                                                                                                                                                                                 |
| name        | string        | **필수**  | 로열티 멤버십/프로그램의 비즈니스별 이름.                                                                                                                                                                                                  |
| display_id  | string        | 선택      | 사용자가 알아볼 수 있도록 마스킹되거나 일부만 표시한 멤버십 ID(예: '\*\*\*\*5678'). 멤버십이 검증되지 않았다면 설정해서는 안 됩니다(MUST NOT).                                                                                             |
| tiers       | Array[object] | 선택      | 이 멤버십의 활성 또는 표시해도 안전한 등급 정보. 대부분의 프로그램은 단일 등급(항목 하나)이며, 병렬 등급 차원(예: 현재 등급과 평생 등급)이 있는 프로그램은 활성 등급마다 항목을 하나씩 채웁니다. 등급 정보가 확정되지 않았으면 생략합니다. |
| rewards     | Array[object] | 선택      | 이 멤버십과 연결된 리워드 유형과 적립 예측입니다. 각 객체는 리워드 유형 하나를 담습니다.                                                                                                                                                   |
| provisional | boolean       | **필수**  | 이 멤버십에 추가 검증이 필요하면 true.                                                                                                                                                                                                     |

#### 멤버십 등급

회원이 활동이나 지출을 통해 단계를 올라갈수록 더 큰 혜택을 여는 특정 달성 등급 또는 상태 단계.

| 이름     | 타입          | 필수 여부 | 설명                                           |
| -------- | ------------- | --------- | ---------------------------------------------- |
| id       | string        | **필수**  | 멤버십 등급의 고유 식별자.                     |
| name     | string        | **필수**  | 사람이 읽을 수 있는 등급 이름(예: 'Platinum'). |
| benefits | Array[object] | 선택      | 이 등급에 연결된 혜택 목록.                    |

#### 멤버십 등급 혜택

멤버십 등급에 연관된 혜택.

| 이름        | 타입   | 필수 여부 | 설명                                                                                  |
| ----------- | ------ | --------- | ------------------------------------------------------------------------------------- |
| id          | string | **필수**  | 등급 혜택의 고유 식별자.                                                              |
| description | string | **필수**  | 이 혜택을 바로 표시할 수 있는, 사람이 읽을 수 있는 설명(예: 'Early access to sales'). |

#### 멤버십 리워드

수량화할 수 있는 리워드 유형과 현재 거래에 대한 선택적 적립 예측.

| 이름             | 타입   | 필수 여부 | 설명                                                    |
| ---------------- | ------ | --------- | ------------------------------------------------------- |
| currency         | object | **필수**  | 고객이 다양한 상거래 활동으로 적립할 수 있는 가치 단위. |
| earning_forecast | object | 선택      | 현재 거래로 적립될 리워드 미리보기.                     |

#### 리워드 수량

연결된 리워드 통화의 최소 단위로 나타낸 음이 아닌 정수 금액. 최소 단위와 기본 단위의 비율은 연결된 리워드 통화의 `decimal_places`가 정의하며, 생략하면 기본값은 0입니다.

#### 리워드 통화

로열티 리워드의 통화.

| 이름           | 타입    | 필수 여부 | 설명                                                             |
| -------------- | ------- | --------- | ---------------------------------------------------------------- |
| name           | string  | **필수**  | 사람이 읽을 수 있는 통화 이름(예: 'LoyaltyStars').               |
| code           | string  | **필수**  | 비즈니스별 통화 표기(예: 'LST').                                 |
| decimal_places | integer | 선택      | 소수점 오른쪽 자릿수. 리워드의 모든 금액 관련 필드에 적용됩니다. |

#### 적립 예상

현재 거래로 적립될 리워드 미리보기.

| 이름      | 타입          | 필수 여부 | 설명                              |
| --------- | ------------- | --------- | --------------------------------- |
| amount    | integer       | **필수**  | 거래가 완료되면 적립될 총 리워드. |
| breakdown | Array[object] | 선택      | 총액을 구성하는 적립 내역 목록.   |

#### 적립 내역

리워드 적립 내역 규칙

| 이름        | 타입    | 필수 여부 | 설명                                                                                                                                     |
| ----------- | ------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| id          | string  | **필수**  | 고유 리워드 내역 규칙 식별자.                                                                                                            |
| amount      | integer | **필수**  | 이 규칙으로 적립된 리워드.                                                                                                               |
| description | string  | **필수**  | 특정 리워드에 대해 바로 표시할 수 있는, 사람이 읽을 수 있는 근거(예: 신발 2배 적립).                                                     |
| benefit_id  | string  | 선택      | 이 리워드 규칙을 만든 membership_tier_benefit의 `id`(선택 사항). 같은 상위 로열티 멤버십 안의 `membership_tier_benefit.id`로 해석됩니다. |

## 로열티 동작

로열티 확장은 역도메인 식별자를 키로 하는 키-값 맵입니다. 비즈니스 프로필의 서비스, 기능, 결제 핸들러와 같은 규칙을 따릅니다. 키는 비즈니스가 인식하는 로열티 멤버십에 대한 자격 주장(eligibility claim)을 나타냅니다. 값에는 그 주장에 대응하는 멤버십 정보가 담기며, 필수 필드인 `provisional`로 검증 상태를 표시합니다.

독립적으로 가입할 수 있는 프로그램은 반드시 로열티 맵 아래에서 역도메인 이름으로 구분되는 별도의 형제 항목으로 모델링해야 합니다(MUST).

플랫폼은 로열티 확장을 활성화하고 로열티 혜택을 주장하기 위해 요청의 `context.eligibility`에 구매자의 로열티 멤버십 주장을 담아 보낼 수 있습니다(MAY). 또는 구매자가 인증되어 있고 비즈니스가 인증된 신원으로 로열티 멤버십을 판단할 수 있다면, 비즈니스는 명시적인 자격 주장 없이도 로열티 확장을 채울 수 있습니다(MAY). 이때 맵 키는 반드시 비즈니스가 주장 값으로 받아들이는 역도메인 식별자와 같아야 합니다(MUST). 이는 비즈니스가 신원 연결(Identity Linking)로 확인한 사용자 상태를 응답에 반환하는 일반 패턴의 한 사례입니다. [비즈니스가 채우는 응답 값](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/identity-linking/#business-populated-response-values)을 참고하세요.

- 비즈니스가 멤버십 주장을 검증했거나 인증된 신원으로 멤버십을 판단했다면 반드시 `provisional: false`를 반환해야 합니다(MUST). **프로그램에 등급 구조가 있다면** 구매자가 보유한 활성 등급을 반드시 `tiers` 배열에 채워야 합니다(MUST). 등급 개념이 없는 프로그램(정률 캐시백, 단일 상태 멤버십)에서는 `tiers`를 생략할 수 있습니다(MAY). 또한 구매자의 마스킹된 고유 식별자를 `display_id`에 설정해야 합니다(SHOULD).
- 요청의 멤버십 주장을 인식하고 수락했지만 아직 검증하지 않았다면, 비즈니스는 반드시 `provisional: true`를 반환해야 합니다(MUST). 세션 중 수락된 상태로 표시해도 안전한 등급 정보는 반환할 수 있지만(MAY), 멤버십이 검증되기 전까지 `display_id`를 반환해서는 안 됩니다(MUST NOT).
- 요청의 멤버십 주장을 수락했지만 검증할 수 없다면, 비즈니스는 반드시 `type: "error"`와 `code: "eligibility_invalid"`가 담긴 복구 가능한 `message`로 실패를 알려야 합니다(MUST). 그러면 플랫폼은 멤버십 주장을 빼고 로열티 혜택 없이 체크아웃을 진행할 수 있습니다(MAY).

체크아웃을 완료할 때, 수락되었지만 검증되지 않은 로열티 주장은 모두 반드시 체크아웃 기능에 정의된 [완료 시 자격 검증](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/checkout/#eligibility-verification-at-completion) 계약에 따라 해소되어야 합니다(MUST).

### 금전적 로열티 혜택

가격에 영향을 주는 금전적 로열티 혜택(예: 회원 가격·배송)에는 "$500 이상 구매 시에만 $10 할인"처럼 멤버십 외의 조건이 붙을 수 있습니다. 비즈니스는 혜택을 적용하기 전에 반드시 그 조건을 평가해야 합니다(MUST).

혜택을 적용하면 비즈니스는 반드시 기본 기능의 가격 필드로 가격 영향을 드러내야 합니다(MUST). 카탈로그 응답에서는 `price` / `list_price`와 `price_range` / `list_price_range`를 사용합니다. 장바구니와 체크아웃 응답에서는 `totals` 또는 `line_items[].totals`에 `type: "items_discount"`(회원 가격) / `type: "discount"`(회원 배송)를 사용하고, 가능하면 `display_text`로 로열티 출처를 표시합니다.

장바구니와 체크아웃 응답에서 할인 확장이 활성화되어 있다면, 비즈니스는 출처를 구조적으로 표시할 수 있도록 `discounts.applied[]`도 채워야 합니다(SHOULD). 이때 `eligibility`는 해당 할인에 필요한 주장을 가리킵니다. 수락된 로열티 주장 중 하나 이상이 아직 검증되지 않은 경우처럼 할인에 여전히 검증이 필요하다면, 해당 적용 할인은 반드시 `provisional: true`로 설정해야 합니다(MUST).

혜택을 적용하지 않는다면 비즈니스는 `type: "warning"` 메시지로 구매자에게 알리고, 해당 금전적 로열티 혜택이 적용되지 않는 이유를 설명해야 합니다(SHOULD). 적용되지 않는 혜택을 할인 확장에 넣어서는 안 됩니다(MUST NOT). 대신 로열티 확장의 `benefits`에 넣고, 추가 맥락을 주도록 경고 메시지의 `path`가 관련 멤버십을 가리키게 할 수 있습니다(MAY).

로열티 멤버십 주장을 수락했다면, 비즈니스는 `type: "info"`로 적용된 금전적 로열티 혜택의 효과를 설명할 수 있습니다(MAY).

**로열티 혜택 메시지 코드:**

| 타입      | 코드                            | 시점                                                |
| --------- | ------------------------------- | --------------------------------------------------- |
| `info`    | `membership_benefit_eligible`   | 특정 혜택을 이 주문에 적용할 수 있다고 확인된 경우  |
| `warning` | `membership_benefit_ineligible` | 혜택은 있지만 이 주문이 조건을 충족하지 못하는 경우 |

아래 예시는 로열티와 할인 확장 필드만 보이도록 축약했습니다. 실제 장바구니·체크아웃 응답에는 `ucp`, `id`, `currency`, `totals` 같은 기본 필수 필드도 포함됩니다.

[완료 시 자격 검증](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/checkout/#eligibility-verification-at-completion)의 매장 로열티 카드 예시를 이어 가 보겠습니다. 이 카드는 무조건 적용되는 상품 할인 하나와, 현재 체크아웃 장바구니가 충족하지 못하는 조건부 할인 하나를 제공한다고 가정합니다. 플랫폼은 첫 번째 잠정 할인을 "구매 시 검증됨" 같은 고지와 함께 보여 주고, 두 번째 할인이 적용되지 않는다는 경고 메시지도 함께 표시할 수 있습니다.

```json
{
  "context": {
    "eligibility": ["com.example.loyalty.store_card"]
  },
  "line_items": [
    {
      "item": {
        "id": "prod_1"
      },
      "quantity": 1
    }
  ]
}
```

```json
{
  "discounts": {
    "applied": [
      {
        "title": "Loyalty benefit 1",
        "amount": 10,
        "provisional": true,
        "eligibility": "com.example.loyalty.store_card",
        "allocations": [
          {"path": "$.line_items[0]", "amount": 10}
        ]
      }
    ]
  },
  "loyalty": {
    "com.example.loyalty.store_card": {
      "id": "membership_1",
      "name": "My Loyalty Program",
      "tiers": [
        {
          "id": "cardholder",
          "name": "Store Cardholder",
          "benefits": [
            { "id": "BEN_001", "description": "Get $10 off with $500+ purchase." },
            { "id": "BEN_002", "description": "Early access to sales" }
          ]
        }
      ],
      "provisional": true
    }
  },
  "messages": [
    {
      "type": "warning",
      "code": "membership_benefit_ineligible",
      "path": "$.loyalty['com.example.loyalty.store_card']",
      "content": "Cart size is smaller than required to receive the $10 discount."
    }
  ]
}
```

구매자는 장바구니를 업데이트하지 않고 바로 체크아웃으로 진행할 수 있습니다. 비즈니스가 주장을 검증하면 무조건 적용되는 회원 가격 할인은 더 이상 잠정 상태가 아니며 `display_id`도 함께 반환됩니다.

```json
{
  "context": {
    "eligibility": ["com.example.loyalty.store_card"]
  },
  "line_items": [
    {
      "item": {
        "id": "prod_1"
      },
      "quantity": 1
    }
  ]
}
```

```json
{
  "discounts": {
    "applied": [
      {
        "title": "Loyalty benefit 1",
        "amount": 10,
        "provisional": false,
        "eligibility": "com.example.loyalty.store_card",
        "allocations": [
          {"path": "$.line_items[0]", "amount": 10}
        ]
      }
    ]
  },
  "loyalty": {
    "com.example.loyalty.store_card": {
      "id": "membership_1",
      "display_id": "****5678",
      "name": "My Loyalty Program",
      "tiers": [
        {
          "id": "tier_1",
          "name": "Store Cardholder",
          "benefits": [
            { "id": "BEN_001", "description": "Get $10 off with $500+ purchase." },
            { "id": "BEN_002", "description": "Early access to sales" }
          ]
        }
      ],
      "provisional": false
    }
  }
}
```

주장을 검증할 수 없다면, 비즈니스는 반드시 `messages[]`에 `code: "eligibility_invalid"`가 담긴 복구 가능한 에러를 반환해야 합니다(MUST).

```json
{
  "context": {
    "eligibility": ["com.example.loyalty.store_card"]
  },
  "line_items": [
    {
      "item": {
        "id": "prod_1"
      },
      "quantity": 1
    }
  ]
}
```

```json
{
  "messages": [
    {
      "type": "error",
      "severity": "recoverable",
      "code": "eligibility_invalid",
      "content": "Buyer is not a store card holder."
    }
  ]
}
```

## 사용 사례와 예시

로열티 확장으로 카탈로그, 장바구니, 체크아웃 기능을 확장하면 구매자의 회원 전용 특전을 빠짐없이 보여 줄 수 있고, 플랫폼은 이 추가 정보를 렌더링해 거래를 도울 수 있습니다.

### 가격에 영향을 주는 혜택

로열티 확장은 플랫폼이 회원 할인을 설명하는 데 필요한 구매자 상태 정보를 제공할 수 있습니다. 가격에 영향을 주는 로열티 혜택은 기본 기능의 가격 필드에 반영됩니다. 할인 확장도 활성화되어 있다면, 플랫폼은 `discounts.applied[].title`로 각 할인을 설명하고 `discounts.applied[].eligibility`를 `loyalty` 항목과 연결해, 수락된 멤버십 주장 중 어느 것이 금전적 혜택을 만들었는지 보여 줄 수 있습니다. 아래 예시에서 구매자는 Retail Club 멤버십과 Retail Card를 보유하고 있어 각각 15% 보너스 할인과 무료 배송 혜택을 받습니다. 플랫폼은 이를 예컨대 "Retail Club Gold Member 및 Retail Card 혜택 적용됨"처럼 렌더링할 수 있습니다.

```json
{
  "context": {
    "eligibility": [
      "com.example.retail_club",
      "com.example.retail_card"
    ]
  },
  "line_items": [
    {
      "item": {
        "id": "prod_1"
      },
      "quantity": 1
    }
  ]
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
        "price": 1000
      },
      "quantity": 1,
      "totals": [
        {"type": "subtotal", "amount": 1000},
        {"type": "items_discount", "display_text": "Club member benefit", "amount": -150},
        {"type": "total", "amount": 850}
      ]
    }
  ],
  "discounts": {
    "applied": [
      {
        "title": "Club Members get 15% Bonus",
        "amount": 150,
        "method": "each",
        "provisional": false,
        "eligibility": "com.example.retail_club",
        "allocations": [
          {"path": "$.line_items[0]", "amount": 150}
        ]
      },
      {
        "title": "Free shipping for Retail Card holder",
        "amount": 199,
        "automatic": true,
        "provisional": false,
        "eligibility": "com.example.retail_card"
      }
    ]
  },
  "loyalty": {
    "com.example.retail_club": {
      "id": "membership_1",
      "display_id": "****5678",
      "name": "Retail Club",
      "provisional": false,
      "tiers": [
        {
          "id": "gold",
          "name": "Gold Member",
          "benefits": [
            { "id": "BEN_001", "description": "Early access to sales" }
          ]
        }
      ]
    },
    "com.example.retail_card": {
      "id": "membership_2",
      "display_id": "****1234",
      "name": "Retail Card",
      "provisional": false,
      "tiers": [
        {
          "id": "cardholder",
          "name": "Retail Card",
          "benefits": [
            { "id": "BEN_002", "description": "Exclusive customer support" }
          ]
        }
      ]
    }
  },
  "totals": [
    {"type": "subtotal", "display_text": "Subtotal", "amount": 1000},
    {"type": "items_discount", "display_text": "Club member benefit", "amount": -150},
    {"type": "discount", "display_text": "Free shipping for Retail Card holder", "amount": -199},
    {"type": "fulfillment", "display_text": "Shipping", "amount": 0},
    {"type": "total", "display_text": "Estimated Total", "amount": 651}
  ]
}
```

### 리워드 적립 예상

회원 가격·배송 같은 즉시 가치형 혜택 말고도, 지연 가치형인 수집형 리워드 혜택은 로열티 생태계의 또 다른 핵심 요소입니다. 구매자가 결정하기 전에 리워드 적립 예상치를 보여 주면 가격 할인을 보완할 수 있고, 에이전트가 가격 관련 이의를 다루는 데에도 어느 정도 도움이 됩니다. 리워드 적립이 가격 할인에 더해지는 추가 가치가 되기 때문입니다. 이 예시에서 비즈니스는 적립 예상치를 내역과 함께 제공하고, `benefit_id`로 해당 규칙을 만든 `membership_tier_benefit`을 연결합니다. 덕분에 플랫폼은 구매자가 왜 리워드를 적립하는지, 적립량이 어떻게 계산되는지를 투명하게 설명할 수 있습니다.

```json
{
  "context": {
    "eligibility": ["com.example.retail_club"]
  },
  "line_items": [
    {
      "item": {
        "id": "prod_1"
      },
      "quantity": 1
    }
  ]
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
        "price": 1000
      },
      "quantity": 1,
      "totals": [
        {"type": "subtotal", "amount": 1000},
        {"type": "total", "amount": 1000}
      ]
    }
  ],
  "loyalty": {
    "com.example.retail_club": {
      "id": "membership_1",
      "display_id": "****5678",
      "name": "Retail Club",
      "provisional": false,
      "tiers": [
        {
          "id": "gold",
          "name": "Gold",
          "benefits": [
            { "id": "BEN_001", "description": "1 point per $1 on everything" },
            { "id": "BEN_002", "description": "2 extra point/dollar on footwear" }
          ]
        }
      ],
      "rewards": [
        {
          "currency": {
            "name": "LoyaltyStars",
            "code": "LST"
          },
          "earning_forecast": {
            "amount": 30,
            "breakdown": [
              {
                "id": "RULE_1",
                "description": "1 point/dollar on everything",
                "amount": 10,
                "benefit_id": "BEN_001"
              },
              {
                "id": "RULE_2",
                "description": "2 extra point/dollar on footwear",
                "amount": 20,
                "benefit_id": "BEN_002"
              }
            ]
          }
        }
      ]
    }
  },
  "totals": [
    {"type": "subtotal", "display_text": "Subtotal", "amount": 1000},
    {"type": "total", "display_text": "Estimated Total", "amount": 1000}
  ]
}
```

## 구현 지침

- 로열티 확장 응답은 반드시 데이터를 최소화해야 하며(MUST), 변하지 않는 원본 회원 식별자를 노출해서는 안 됩니다(MUST NOT). 노출하면 플랫폼이 개별 구매자를 고유하게 식별할 수 있게 되기 때문입니다.
- 로열티 확장 응답은 반드시 멤버십이 검증·인증된 뒤에만 `display_id`를 포함해야 합니다(MUST).
- 로열티 확장 응답은 반드시 요청의 모든 `context.eligibility` 값을 증명이 아닌 구매자의 주장으로 취급해야 합니다(MUST).
