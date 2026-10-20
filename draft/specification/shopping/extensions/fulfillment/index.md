# 풀필먼트 확장

## 개요

풀필먼트 확장을 사용하면 비즈니스가 배송, 픽업 같은 실물 상품 풀필먼트를 지원한다고 알릴 수 있습니다.

이 확장은 체크아웃 및/또는 카탈로그에 `fulfillment` 필드를 추가합니다.

- **체크아웃**(`dev.ucp.shopping.checkout`) — 선택과 비용: 어떤 상품을 어디로, 어떤 방법으로, 얼마의 가격과 예상 도착 시간(ETA)으로 보낼지
- **카탈로그**(`dev.ucp.shopping.catalog.search` 및 `dev.ucp.shopping.catalog.lookup`) — 디스커버리: 제공된 구매자 컨텍스트를 바탕으로 각 변형(variant)에서 이용할 수 있는 풀필먼트 옵션을 알립니다. [카탈로그 디스커버리](#catalog-discovery)를 참고하세요.

체크아웃의 `fulfillment` 필드에는 다음이 담깁니다.

- `methods[]` — 장바구니 상품에 적용되는 풀필먼트 방법(배송, 픽업 등)
  - `line_item_ids` — 이 방법으로 풀필먼트할 상품
  - `destinations[]` — 풀필먼트할 장소(주소, 매장 위치)
  - `groups[]` — 비즈니스가 생성한 패키지. 패키지마다 선택할 수 있는 `options[]`가 있음
- `available_methods[]` — 상품별 재고 가용성(선택 사항)

**멘탈 모델:**

- `methods[0]` 배송
  - `line_item_ids` 👕👖
  - `selected_destination_id` = `destinations[0].id` 🔘✅ 123 Fake St
  - `groups[0]` 📦👕👖
    - `selected_option_id` = `options[0].id` 🔘✅ 일반 배송 $5
    - `options[1]` 🔘 특급 배송 $10
- `methods[1]` 매장 픽업
  - `line_item_ids` 👞
  - `selected_destination_id` = `destinations[0].id` 🔘✅ Uptown Store
  - `groups[0]` 📦👞
    - `selected_option_id` = `options[0].id` 🔘✅ 매장 내 픽업

## 위치 컨텍스트

기본 [컨텍스트](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/#context)에는 선택 사항인 `location`이 정의되어 있습니다. 비즈니스 네임스페이스 안에서 안정적이고 불투명한 [위치(Location)](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/glossary/#commerce) 식별자입니다. 이 필드는 카탈로그 요청과 장바구니 및 체크아웃 생성·수정 요청에 나타납니다([체크아웃 컨텍스트](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#context) 참고). 풀필먼트 확장은 이 필드를 추가하거나 재정의하지 않고, 확장이 활성화되었을 때 이 필드가 어떤 효과를 내는지만 정의합니다.

풀필먼트가 카탈로그나 체크아웃을 확장할 때, 해당 기능에 보내는 요청의 `context.location`은 잠정적인 비즈니스 위치를 가리킵니다. 비즈니스는 이 위치를 기준으로 풀필먼트 가용성을 평가하고 초기 풀필먼트 선택지를 만듭니다. 이 필드로 풀필먼트 목적지가 선택되지는 않습니다.

다음 필드는 한 값의 정밀도 차이가 아니라 각기 다른 역할을 맡습니다.

| 필드                                         | 역할                                                            |
| -------------------------------------------- | --------------------------------------------------------------- |
| `context.location`                           | 풀필먼트를 평가하는 기준이 되는 잠정적 비즈니스 위치            |
| 카탈로그 `filters.fulfills_to`               | 상품을 풀필먼트할 명시적 목적지이자 결과 필터                   |
| 카탈로그 `methods[].location`                | 변형의 장소 기반 방법(예: `pickup`)에 대해 확정된 비즈니스 위치 |
| 체크아웃 `methods[].selected_destination_id` | 해당 방법의 `destinations[]` 중에서 선택된 목적지               |

우선순위는 각 필드가 관장하는 범위 안에서만 적용됩니다.

- 카탈로그 `filters.fulfills_to`가 있으면 비즈니스는 풀필먼트 목적지와 방법의 `availability`를 `context`가 아니라 이 필드를 기준으로 반드시 확정해야 합니다(**MUST**). 다른 `context` 필드는 영향을 받지 않습니다.
- 플랫폼이 체크아웃 방법에 `selected_destination_id`를 설정하면, 비즈니스는 그 방법의 풀필먼트 범위에서 `context.location` 대신 참조된 목적지를 반드시 사용해야 합니다(**MUST**). 다른 방법은 영향을 받지 않습니다.

**위치 이어 가기.** 같은 식별자가 이 필드 중 여러 곳에 나타나더라도 역할이 서로 합쳐지지는 않습니다. 예를 들어 카탈로그 응답이 `loc_123`에서 이용할 수 있는 장소 기반 방법을 보고했다고 합시다. 기본 장바구니 컨텍스트에 이미 `location`이 있으므로 `loc_123`을 장바구니 요청의 `context.location`으로 이어 쓸 수 있습니다. [장바구니-체크아웃 전환](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/#cart-to-checkout-conversion)에서는 장바구니의 `context`로 체크아웃을 초기화하므로, 비즈니스는 `loc_123`으로 초기 체크아웃 풀필먼트 목적지를 만들 수 있습니다(**MAY**). 플랫폼이 `selected_destination_id`를 설정하면 그 명시적 선택이 우선합니다.

## 스키마

풀필먼트는 물리적 배송이 필요한 상품에만 적용됩니다. 풀필먼트가 필요 없는 상품(예: 디지털 상품)은 방법에 할당하지 않아도 됩니다.

### 속성

| 이름        | 타입                                                                                          | 필수 여부                     | 설명                 |
| ----------- | --------------------------------------------------------------------------------------------- | ----------------------------- | -------------------- |
| fulfillment | [Fulfillment](/ucp-ko/draft/draft/specification/shopping/extensions/fulfillment/#fulfillment) | Optional; omitted on complete | Fulfillment details. |

### 엔티티

#### 풀필먼트

| 이름              | 타입                                                                                                               | 필수 여부 | 설명                                |
| ----------------- | ------------------------------------------------------------------------------------------------------------------ | --------- | ----------------------------------- |
| methods           | Array\[[Fulfillment Method](/ucp-ko/draft/draft/specification/reference/#fulfillment-method)\]                     | Optional  | Fulfillment methods for cart items. |
| available_methods | Array\[[Fulfillment Available Method](/ucp-ko/draft/draft/specification/reference/#fulfillment-available-method)\] | Optional  | Inventory availability hints.       |

#### 풀필먼트 방법

| 이름                    | 타입                                                                                                     | 필수 여부    | 설명                                                                                                                                                                         |
| ----------------------- | -------------------------------------------------------------------------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id                      | string                                                                                                   | **Required** | Unique fulfillment method identifier.                                                                                                                                        |
| type                    | string                                                                                                   | **Required** | Fulfillment method type. Well-known values: `shipping`, `pickup`. Businesses MAY use additional values.                                                                      |
| line_item_ids           | Array[string]                                                                                            | **Required** | Line item IDs fulfilled via this method.                                                                                                                                     |
| destinations            | Array\[[Fulfillment Destination](/ucp-ko/draft/draft/specification/reference/#fulfillment-destination)\] | Optional     | Available destinations for this method. In Business responses, each destination carries a `type` and `id`.                                                                   |
| selected_destination_id | ['string', 'null']                                                                                       | Optional     | ID of the selected destination. Accepts any stable, Business-scoped ID the Business recognizes for this method, including Location IDs not yet enumerated in `destinations`. |
| groups                  | Array\[[Fulfillment Group](/ucp-ko/draft/draft/specification/reference/#fulfillment-group)\]             | Optional     | Fulfillment groups for selecting options. Agent sets selected_option_id on groups to choose shipping method.                                                                 |

#### 풀필먼트 목적지

| 이름 | 타입   | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                              |
| ---- | ------ | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type | string | **Required** | Destination contract discriminator. Required in Business responses and optional in Platform requests. Well-known values: `shipping_address`, `business_location`. The enclosing method contract defines request defaults and which fields the Platform may write; negotiated extensions define additional values. |
| id   | string | **Required** | Fulfillment destination identifier.                                                                                                                                                                                                                                                                               |

#### 배송 목적지

| 이름             | 타입   | 필수 여부    | 설명                                                                                                                                                                                                                                      |
| ---------------- | ------ | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| extended_address | string | Optional     | An address extension such as an apartment number, C/O or alternative name.                                                                                                                                                                |
| street_address   | string | Optional     | The street address.                                                                                                                                                                                                                       |
| address_locality | string | Optional     | The locality in which the street address is, and which is in the region. For example, Mountain View.                                                                                                                                      |
| address_region   | string | Optional     | The region in which the locality is, and which is in the country. Required for applicable countries (i.e. state in US, province in CA). For example, California or another appropriate first-level Administrative division.               |
| address_country  | string | Optional     | The country. Recommended to be in 2-letter ISO 3166-1 alpha-2 format, for example "US". For backward compatibility, a 3-letter ISO 3166-1 alpha-3 country code such as "SGP" or a full country name such as "Singapore" can also be used. |
| postal_code      | string | Optional     | The postal code. For example, 94043.                                                                                                                                                                                                      |
| first_name       | string | Optional     | Optional. First name of the contact associated with the address.                                                                                                                                                                          |
| last_name        | string | Optional     | Optional. Last name of the contact associated with the address.                                                                                                                                                                           |
| phone_number     | string | Optional     | Optional. Phone number of the contact associated with the address.                                                                                                                                                                        |
| id               | string | **Required** | ID specific to this shipping destination.                                                                                                                                                                                                 |
| type             | string | **Required** | **Constant = shipping_address**. Destination type discriminator.                                                                                                                                                                          |

#### 비즈니스 위치 목적지

| 이름    | 타입                                                                          | 필수 여부    | 설명                                                                             |
| ------- | ----------------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------- |
| id      | string                                                                        | **Required** | Stable, opaque, Business-scoped Location identifier.                             |
| name    | string                                                                        | **Required** | Buyer-facing, Business-owned display name.                                       |
| address | [Postal Address](/ucp-ko/draft/draft/specification/reference/#postal-address) | Optional     | Physical address of the location.                                                |
| type    | string                                                                        | **Required** | **Constant = business_location**. Destination type discriminator. Response-only. |

#### 위치 요약

| 이름    | 타입                                                                          | 필수 여부    | 설명                                                 |
| ------- | ----------------------------------------------------------------------------- | ------------ | ---------------------------------------------------- |
| id      | string                                                                        | **Required** | Stable, opaque, Business-scoped Location identifier. |
| name    | string                                                                        | **Required** | Buyer-facing, Business-owned display name.           |
| address | [Postal Address](/ucp-ko/draft/draft/specification/reference/#postal-address) | Optional     | Physical address of the location.                    |

#### 풀필먼트 그룹

| 이름               | 타입                                                                                           | 필수 여부    | 설명                                                                   |
| ------------------ | ---------------------------------------------------------------------------------------------- | ------------ | ---------------------------------------------------------------------- |
| id                 | string                                                                                         | **Required** | Group identifier for referencing merchant-generated groups in updates. |
| line_item_ids      | Array[string]                                                                                  | **Required** | Line item IDs included in this group/package.                          |
| options            | Array\[[Fulfillment Option](/ucp-ko/draft/draft/specification/reference/#fulfillment-option)\] | Optional     | Available fulfillment options for this group.                          |
| selected_option_id | ['string', 'null']                                                                             | Optional     | ID of the selected fulfillment option for this group.                  |

#### 풀필먼트 옵션

| 이름                      | 타입                                                                          | 필수 여부    | 설명                                                                                                                                                    |
| ------------------------- | ----------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id                        | string                                                                        | **Required** | Unique identifier for this fulfillment option.                                                                                                          |
| title                     | string                                                                        | **Required** | Short label that distinguishes this option from its siblings (e.g. 'Standard', 'Express Shipping', 'Curbside Pickup').                                  |
| description               | [Description](/ucp-ko/draft/draft/specification/reference/#description)       | Optional     | Supplementary context for the title (e.g. 'Arrives in 4 business days', 'Arrives Dec 12-15 via FedEx'). Directly renderable; MUST NOT repeat the title. |
| carrier                   | string                                                                        | Optional     | Carrier name (for shipping).                                                                                                                            |
| earliest_fulfillment_time | string                                                                        | Optional     | Earliest fulfillment date.                                                                                                                              |
| latest_fulfillment_time   | string                                                                        | Optional     | Latest fulfillment date.                                                                                                                                |
| totals                    | Array\[[Total Response](/ucp-ko/draft/draft/specification/reference/#total)\] | **Required** | Fulfillment option totals breakdown.                                                                                                                    |

#### 풀필먼트 가능 방법

| 이름           | 타입               | 필수 여부    | 설명                                                                                                                                 |
| -------------- | ------------------ | ------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| type           | string             | **Required** | Fulfillment method type this availability applies to. Well-known values: `shipping`, `pickup`; businesses MAY use additional values. |
| line_item_ids  | Array[string]      | **Required** | Line items available for this fulfillment method.                                                                                    |
| fulfillable_on | ['string', 'null'] | Optional     | 'now' for immediate availability, or ISO 8601 date for future (preorders, transfers).                                                |
| description    | string             | Optional     | Human-readable availability info (e.g., 'Available for pickup at Downtown Store today').                                             |

#### 합계

| 이름         | 타입                                                                        | 필수 여부    | 설명                                                                                                                                                                                                                                                                                        |
| ------------ | --------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                      | **Required** | Cost category. Well-known values: subtotal, items_discount, discount, fulfillment, tax, fee, total. Businesses MAY use additional values.                                                                                                                                                   |
| display_text | string                                                                      | Optional     | Text to display against the amount. Should reflect appropriate method (e.g., 'Shipping', 'Delivery').                                                                                                                                                                                       |
| amount       | [Signed Amount](/ucp-ko/draft/draft/specification/reference/#signed-amount) | **Required** | Monetary amount in the currency's minor unit as defined by ISO 4217. Refer to the currency's exponent to determine minor-to-major ratio (e.g., 2 for USD, 0 for JPY, 3 for KWD). May be negative — the sign is intrinsic to the value (e.g., discounts are negative, charges are positive). |

#### 우편 주소

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

### 예시

```json
{
  "ucp": { ... },
  "id": "...",
  "status": "...",
  "currency": "...",
  "line_items": [ ... ],
  "totals": [ ... ],
  "links": [ ... ],
  "fulfillment": {
    "methods": [
      {
        "id": "method_1",
        "type": "shipping",
        "line_item_ids": ["shirt", "pants"],
        "selected_destination_id": "dest_1",
        "destinations": [
          {
            "type": "shipping_address",
            "id": "dest_1",
            "street_address": "123 Main St",
            "address_locality": "Springfield",
            "address_region": "IL",
            "postal_code": "62701",
            "address_country": "US"
          }
        ],
        "groups": [
          {
            "id": "package_1",
            "line_item_ids": ["shirt", "pants"],
            "selected_option_id": "standard",
            "options": [
              {
                "id": "standard",
                "title": "Standard Shipping",
                "description": { "plain": "Arrives Dec 12-15 via USPS" },
                "totals": [
                  {
                    "type": "total",
                    "amount": 500
                  }
                ]
              },
              {
                "id": "express",
                "title": "Express Shipping",
                "description": { "plain": "Arrives Dec 10-11 via FedEx" },
                "totals": [
                  {
                    "type": "total",
                    "amount": 1000
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

## 목적지

풀필먼트 방법의 `type`은 상품을 *어떻게* 풀필먼트하는지(예: 배송 또는 픽업)를 나타내고, 목적지의 `type`은 풀필먼트가 *어디서* 이루어지는지(예: 배송 주소 또는 비즈니스 위치)를 나타냅니다. 두 구분자는 방향에 따라 역할이 다릅니다. 비즈니스 응답에서는 모든 목적지에 필수 `type`이 있으므로 응답만 보고도 내용을 알 수 있습니다. 플랫폼 요청에서는 방법의 계약이 `destinations[]`를 누가 작성할지 정하며, `type`을 생략했을 때 쓸 기본 목적지 형태를 정의할 수 있습니다(MAY).

기본적으로 플랫폼은 방법의 `destinations[]`를 작성하지 않습니다. 다만 방법의 `type`별 계약에서 플랫폼이 작성할 수 있는 요청 형태를 선택적으로 허용할 수 있습니다(**MAY**).

| 방법 `type`              | 요청 `destinations[]`                                                                                                                                                       |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `shipping`               | 플랫폼이 작성할 수 있습니다. 플랫폼이 구매자의 배송 주소 정보를 작성합니다.                                                                                                 |
| `pickup`                 | 플랫폼이 작성할 수 없습니다. 비즈니스가 응답에 위치를 나열하고, 플랫폼은 비즈니스 범위의 ID로 위치를 선택합니다([선택과 위치 식별](#selection-and-location-identity) 참고). |
| 그 밖의 모든 방법 `type` | 플랫폼이 작성할 수 없습니다(기본값). 비즈니스가 응답에 목적지를 나열하고, 플랫폼은 `selected_destination_id`로 하나를 선택합니다.                                           |

목적지 `type`은 **응답에서는 필수, 요청에서는 선택 사항**입니다. 응답에서는 모든 목적지에 `type`이 있으므로, 목적지가 어디에 나타나든 그 자체로 의미가 분명합니다. 요청에서는 방법별 계약이 플랫폼 작성 형태를 허용하지 않는 한 `destinations[]`를 생략합니다.

- 잘 알려진 `shipping` 방법에서는 모든 목적지가 배송 목적지입니다. `type`을 생략한 목적지는 기본값인 `shipping_address`로 간주합니다.
- 잘 알려진 `pickup` 방법에서는 목적지가 응답 전용입니다. 플랫폼은 `selected_destination_id`로 위치를 선택합니다 ([선택과 위치 식별](#selection-and-location-identity) 참고).
- 그 밖의 방법 유형에는 기본값이 적용됩니다. `destinations[]`는 응답 전용이고, 응답의 각 목적지는 필수인 `type`과 `id`로 스스로를 설명하며, 플랫폼은 `selected_destination_id`로 하나를 선택합니다. 해당 방법을 정의하는 계약(이 명세의 향후 개정판이나 협상된 확장)에서 응답 목적지 형태를 좁히거나 플랫폼이 작성할 수 있는 요청 형태를 허용할 수 있습니다(**MAY**).

`destinations[]`가 들어 있는 요청에는 방법의 `type`도 반드시 있어야 합니다(MUST). 잘 알려진 값은 다음과 같습니다.

| 값                  | 의미                                                 |
| ------------------- | ---------------------------------------------------- |
| `shipping_address`  | 평탄한 우편 주소 필드로 이루어진 배송 목적지         |
| `business_location` | 비즈니스 범위의 `id`로 식별하는 비즈니스 위치 목적지 |

그 밖의 값은 협상된 확장에서 정의합니다. 그런 값에만 있는 목적지 필드는 해당 협상된 확장의 스키마로 검증합니다.

### 배송 목적지

배송 목적지에는 주소 자체를 담을 수도 있고, 저장된 주소를 참조하는 `id`를 담을 수도 있습니다. 비즈니스는 그 `id`를 자체 기록에서 확인하거나, 디지털 지갑이나 신원 제공자 같은 신뢰할 수 있는 자격 증명 제공자를 통해 확인할 수 있습니다.

#### 플랫폼 요청

```json
{
  "street_address": "123 Main St",
  "address_locality": "Springfield",
  "address_region": "IL",
  "postal_code": "62701",
  "address_country": "US"
}
```

#### 비즈니스 응답

```json
{
  "type": "shipping_address",
  "id": "dest_1",
  "street_address": "123 Main St",
  "address_locality": "Springfield",
  "address_region": "IL",
  "postal_code": "62701",
  "address_country": "US"
}
```

### 비즈니스 위치 목적지

비즈니스 위치 목적지는 응답에만 나타납니다. `pickup` 방법의 `destinations[]`는 요청 필드가 아닙니다. 플랫폼은 `destinations[]`에 비즈니스 위치 목적지를 작성해서는 안 되며(**MUST NOT**), 안정적이고 불투명한 비즈니스 범위 ID를 `selected_destination_id`로 제출해 위치를 선택합니다. 비즈니스는 `type: "business_location"`, `id`, 구매자에게 보이는 `name`을 반드시 반환해야 하며(**MUST**), `address`에 우편 주소를 반환할 수 있습니다(**MAY**).

#### 플랫폼 요청

```json
{
  "type": "pickup",
  "line_item_ids": ["shirt", "pants"],
  "selected_destination_id": "loc_downtown"
}
```

#### 비즈니스 응답

```json
{
  "type": "business_location",
  "id": "loc_downtown",
  "name": "Downtown Store",
  "address": {
    "street_address": "123 Main St",
    "address_locality": "Springfield",
    "address_region": "IL",
    "postal_code": "62701",
    "address_country": "US"
  }
}
```

### 선택과 위치 식별

`selected_destination_id`는 방법 안에서 선택된 목적지를 `id`로 식별하며, 비즈니스 위치를 선택하는 유일한 수단입니다. 비즈니스가 해당 방법에서 인식하는 안정적인 비즈니스 범위 ID라면 무엇이든 받습니다. 비즈니스가 그 방법의 `destinations[]`에 (아직) 나열하지 않은 ID도 마찬가지입니다.

선택은 ID의 출처를 따지지 않습니다. null이 아닌 `selected_destination_id`에는 플랫폼이 카탈로그 응답, 위치 검색이나 조회, 이전 체크아웃, 또는 비즈니스가 해당 방법에서 인식하는 그 밖의 출처에서 얻은 ID를 담을 수 있습니다(**MAY**). ID를 어디서 얻었든 아래 계약은 똑같이 적용됩니다.

비즈니스가 null이 아닌 `selected_destination_id`를 수락하면, 그 응답은 다음 조건을 지켜야 합니다.

- 방법에 같은 `selected_destination_id` 값을 반드시 담아야 합니다(**MUST**).
- 같은 방법의 `destinations[]`에 `id`가 그 값과 같고 [목적지](#destinations)에 명시된 대로 유형이 지정된 목적지를 정확히 하나 반드시 포함해야 합니다(**MUST**). 그래야 ID의 출처와 상관없이 응답만 보고도 내용을 알 수 있습니다.

비즈니스는 선택된 목적지의 현재 가용성과 조건을 반드시 다시 검증해야 합니다(**MUST**). ID를 인식했다고 해서 재고가 확보되거나 자격이 보장되지는 않습니다. 비즈니스는 다른 목적지로 몰래 대체해서는 안 됩니다(**MUST NOT**). 다른 `selected_destination_id`를 반환해서도 안 되고(**MUST NOT**), 제출된 ID는 그대로 둔 채 다른 위치를 설명하는 목적지를 반환해서도 안 됩니다(**MUST NOT**). 제출된 선택을 이행할 수 없다면 비즈니스는 선택을 대체하지 말고 거부합니다.

[체크아웃 수정](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#update-checkout)에서 제출된 `selected_destination_id`를 비즈니스가 수락할 수 없을 때(해당 방법에서 인식하지 못하는 ID이거나 재검증에 실패한 경우)는 수정 요청을 거부할 때의 일반 동작을 따릅니다. 즉, 현재 체크아웃을 반드시 변경하지 않은 채로 두어야 하며(**MUST**), `severity: "recoverable"`이고 `path`가 시도된 방법의 `selected_destination_id`를 가리키는(예: `$.fulfillment.methods[0].selected_destination_id`) 에러 메시지와 함께 그 체크아웃을 반드시 반환해야 합니다(**MUST**). [에러 처리](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#error-handling)와 [`path` 필드](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#the-path-field)를 참고하세요.

## 렌더링

풀필먼트 옵션은 \*\*방법에 구애받지 않는 렌더링(method-agnostic rendering)\*\*을 염두에 두고 설계되었습니다. 플랫폼은 특정 방법 유형(배송, 픽업 등)을 이해하지 못해도 옵션을 의미 있게 보여 줄 수 있습니다. 비즈니스가 미리 계산해 둔, 사람이 읽을 수 있는 필드를 제공하면 플랫폼은 이를 그대로 렌더링합니다.

### 사람이 읽을 수 있는 필드

| 위치                  | 필드          | 필수   | 목적                             |
| --------------------- | ------------- | ------ | -------------------------------- |
| `groups[].options[]`  | `title`       | 예     | 형제 옵션과 구분되는 기본 레이블 |
| `groups[].options[]`  | `description` | 아니요 | 제목을 보충하는 맥락 정보        |
| `groups[].options[]`  | `totals`      | 예     | 비용 내역: `total` 객체의 배열   |
| `available_methods[]` | `description` | 아니요 | 대체 가용성에 대한 독립적인 설명 |

### 비즈니스의 책임

**`options[].title`의 경우:**

- 형제 옵션과 반드시 구분되어야 합니다(**MUST**).
- 방법과 속도를 포함해야 합니다(**SHOULD**)(예: "Express Shipping", "Curbside Pickup").
- `description`이 없어도 구매자가 결정을 내리기에 반드시 충분해야 합니다(**MUST**).

**`options[].description`의 경우:**

- `title`이나 `total`을 반복해서는 안 됩니다(**MUST NOT**). 보충 맥락만 제공합니다.
- 배송 시점, 운송사, 그 밖에 결정에 필요한 세부 정보를 포함해야 합니다(**SHOULD**).
- 완결된 구문이어야 합니다(**SHOULD**)(예: "Arrives Dec 12-15 via FedEx").
- 제목만으로 의미가 분명하면 생략할 수 있습니다(**MAY**).

**`available_methods[].description`의 경우:**

- 무엇을, 언제, 어디서 이용할 수 있는지 설명하는 독립적인 문장이어야 합니다(**MUST**).
- 플랫폼 대화에 그대로 쓸 수 있어야 합니다(**SHOULD**)(예: "Pants available for pickup at Downtown Store today at 2pm").

**정렬의 경우:**

- 비즈니스는 `options[]`를 의미 있는 순서(예: 저렴한 순, 빠른 순)로 반환해야 합니다(**SHOULD**).
- 플랫폼은 그 순서를 유지해야 하지만(**SHOULD**), 알려진 구매자 선호도나 화면별 순위에 맞추려고 순서를 바꿀 수도 있습니다(**MAY**). 다만 방법/옵션 그룹 구조는 반드시 유지해야 합니다(**MUST**).

### 플랫폼의 책임

플랫폼은 풀필먼트를 범용적이고 렌더링 가능한 구조로 다뤄야 합니다(**SHOULD**).

- 각 옵션을 `title`, `description`, `total`로 카드 형태로 렌더링합니다.
- 반환된 방법을 모두 보여 줍니다. 어떤 방법을 고를지는 구매자가 결정합니다.
- 방법과 옵션 구조를 그대로 유지합니다. 병합하거나 중복을 제거하지 않습니다. 정렬은 플랫폼이 정합니다.
- `available_methods[].description`으로 구매자에게 대안을 보여 줍니다.

플랫폼은 인식하는 방법 유형이라면 향상된 UX(픽업용 매장 선택기, 배송용 운송사 로고)를 제공할 수 있지만(**MAY**), 어디까지나 선택 사항입니다. 기본 계약은 \*\*`title` + `description`

- `total`만으로 어떤 옵션이든 렌더링할 수 있어야 한다\*\*는 것입니다.

플랫폼이 완전히 처리할 수 없는 옵션을 구매자가 선택하면, 플랫폼은 `continue_url`로 비즈니스의 체크아웃에 넘겨야 합니다(**SHOULD**).

## 가능한 방법

가능한 방법(available methods)은 상품을 특정 방법으로 풀필먼트할 수 있는지, 그리고 언제 가능한지를 나타냅니다. 다음과 같은 경우에 씁니다.

- **대체 방법**: "이 바지는 Downtown Store에서 픽업도 가능합니다"
- **나중에 풀필먼트**: 예약 주문, 먼 창고에서 출고되는 상품, 매장에 재고가 들어오면 픽업

```json
{
  "ucp": { ... },
  "id": "...",
  "status": "...",
  "currency": "...",
  "line_items": [ ... ],
  "totals": [ ... ],
  "links": [ ... ],
  "fulfillment": {
    "methods": [
      {
        "id": "shipping",
        "type": "shipping",
        "line_item_ids": ["shirt", "pants"]
      },
      {
        "id": "pickup",
        "type": "pickup",
        "line_item_ids": []
      }
    ],
    "available_methods": [
      {
        "type": "shipping",
        "line_item_ids": ["shirt", "pants"],
        "fulfillable_on": "now"
      },
      {
        "type": "pickup",
        "line_item_ids": ["pants"],
        "fulfillable_on": "2026-12-01T10:00:00Z",
        "description": "Available for pickup at Downtown Store today at 2pm"
      }
    ]
  }
}
```

플랫폼은 `description` 필드로 구매자에게 대안을 보여 줄 수 있습니다.

> 🤖 셔츠와 바지는 $5에 배송되며 5~8일 후 도착합니다. 또는 바지는 4시간 후 Downtown Store에서 픽업할 수 있습니다.

구매자가 픽업을 선택했는데 플랫폼이 분할 풀필먼트를 지원하지 않으면, 플랫폼은 `continue_url`로 비즈니스의 체크아웃에 넘겨야 합니다(**SHOULD**).

## 카탈로그 디스커버리

풀필먼트 확장이 카탈로그 기능을 확장하면, 카탈로그 응답의 각 변형에 `fulfillment` 객체가 붙어 그 변형에서 이용할 수 있는 풀필먼트 방법과 가용성을 나열합니다. 덕분에 카탈로그를 둘러보는 구매자가 상품을 어떻게 받을 수 있는지 알 수 있습니다.

### 방법

`fulfillment.methods[]`는 변형에서 이용할 수 있는 방법을 나열합니다. 각 방법에는 다음 항목이 있습니다.

- `type` — 풀필먼트 방법(예: `shipping`, `pickup`). [방법 유형](#method-types) 참고
- `description` — 이 방법으로 변형을 어떻게 풀필먼트하는지 구매자에게 알려 주는 짧은 요약(예: "Ships in 2–4 business days"). 바로 렌더링할 수 있습니다. [렌더링](#rendering) 참고
- `availability` — 지정되었거나 추론된 위치에서 이 방법으로 변형을 이용할 수 있는지 여부
- `location` — `pickup` 같은 장소 기반 방법에서 그 방법에 대해 확정된 [위치](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/glossary/#commerce)이자 해당 장소를 가리키는 비즈니스의 안정적인 식별자. `location`에서 픽업할 수 있다고 알린 비즈니스는 체크아웃에서 해당 방법의 `selected_destination_id`로 같은 ID를 반드시 수락해야 합니다(**MUST**). 비즈니스는 현재 가용성과 조건을 다시 검증합니다. 디스커버리 단계에서 재고가 확보되거나 자격이 보장되지는 않으며, 선택 계약은 ID의 출처와 무관합니다. [선택과 위치 식별](#selection-and-location-identity) 참고
- `options` — 이 방법 안의 구체적인 풀필먼트 선택지(예: Standard, Express). [옵션](#options) 참고. 선택 사항

카탈로그는 방법마다 위치 하나, 즉 `fulfills_to`로 지정되었거나 `context`에서 추론된 위치에 대한 가용성만 보고합니다. 다른 위치를 찾아 비교하는 일은 별도로 처리합니다.

변형 수준의 `availability`는 변형을 *어떤* 방법으로든 받을 수 있는지를 나타냅니다. 특정 방법에 대해서는 그 방법의 `availability`가 권위를 가집니다. 방법에 `availability`가 명시되어 있으면 소비자는 그 방법에 대해 반드시 그 값을 사용해야 하며(MUST), 변형 수준의 값으로 방법별 가용성을 추론해서는 안 됩니다(MUST NOT).

### 옵션

방법에는 `options[]`가 있을 수 있습니다(MAY). 이는 풀필먼트 옵션의 전체 목록이 아니라 대표적인 일부입니다. 목적지나 전체 장바구니를 모르는 상태라면 비즈니스는 구매자에게 의미 있는 양 끝의 옵션(예: 가장 저렴한 옵션, 가장 빠른 옵션)을 미리 보여 줘야 합니다(**SHOULD**). 더 구체적인 옵션은 라인 아이템과 목적지가 정해진 뒤 체크아웃에서 협상합니다.

각 옵션에는 `id`와 `title`(형제 옵션과 구분되는 짧은 레이블)이 있고, 맥락을 설명하는 렌더링 가능한 `description`이 선택적으로 붙습니다. 이것이 공통 기반입니다. 체크아웃에서는 같은 옵션에 비용과 시점 정보(`totals`, 운송사, 풀필먼트 시간)가 더해집니다. 옵션은 열린 구조이므로 비즈니스가 필드를 추가로 덧붙일 수 있습니다(MAY). 옵션 없이 `type`, `description`, `availability`만 보여 주는 방법도 있을 수 있습니다(MAY). 옵션은 그룹 계층 없이 방법 바로 아래에 중첩됩니다(체크아웃의 `methods[].groups[].options[]`와 다릅니다).

디스커버리에서 얻은 옵션 `id`로 구매자의 선택을 체크아웃까지 이어 갈 수 있습니다. 비즈니스는 체크아웃에서 같은 ID를 `methods[].groups[].selected_option_id`로 수락해야 합니다(**SHOULD**). 이 ID는 최선의 노력(best-effort) 수준의 핸들일 뿐, 같은 옵션이 나온다고 보장하지 않습니다. 체크아웃에서는 다른 상품, 수량, 합배송에 따라 옵션이 바뀌므로, 단일 상품 기준으로 찾은 옵션이 체크아웃에서는 달라질 수 있습니다.

### 형태

#### 카탈로그 풀필먼트

How a catalog variant can be fulfilled. Mirrors checkout `fulfillment`.

| 이름    | 타입          | 필수 여부 | 설명                                  |
| ------- | ------------- | --------- | ------------------------------------- |
| methods | Array[object] | Optional  | Fulfillment methods for this variant. |

#### 카탈로그 풀필먼트 방법

A fulfillment method on a catalog variant: how the variant can be fulfilled, and its availability.

| 이름         | 타입                                                                                                     | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                       |
| ------------ | -------------------------------------------------------------------------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                                                   | **Required** | Fulfillment method type. Well-known values: `shipping`, `pickup`. Businesses MAY use additional values.                                                                                                                                                                                                    |
| description  | [Description](/ucp-ko/draft/draft/specification/reference/#description)                                  | Optional     | Short buyer-facing summary (e.g. 'Ships in 2–4 business days').                                                                                                                                                                                                                                            |
| availability | [Availability](/ucp-ko/draft/draft/specification/reference/#availability)                                | Optional     | Availability of this variant via this method at the specified or inferred location.                                                                                                                                                                                                                        |
| location     | string                                                                                                   | Optional     | Stable, opaque identifier for the Business Location resolved for this place-based fulfillment method. The Business recognizes the same ID when submitted as `selected_destination_id` for that method; recognition does not reserve inventory or guarantee eligibility, and current terms are revalidated. |
| options      | Array\[[Fulfillment Option Base](/ucp-ko/draft/draft/specification/reference/#fulfillment-option-base)\] | Optional     | Representative fulfillment options for this method (e.g. Standard, Express). Without a destination or full cart, a Business SHOULD preview meaningful boundary options (e.g. cheapest, fastest); more specific options are negotiated in Checkout once line items and destination are known.               |

#### 풀필먼트 옵션 기반

| 이름        | 타입                                                                    | 필수 여부    | 설명                                                                                                                                                    |
| ----------- | ----------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id          | string                                                                  | **Required** | Unique identifier for this fulfillment option.                                                                                                          |
| title       | string                                                                  | **Required** | Short label that distinguishes this option from its siblings (e.g. 'Standard', 'Express Shipping', 'Curbside Pickup').                                  |
| description | [Description](/ucp-ko/draft/draft/specification/reference/#description) | Optional     | Supplementary context for the title (e.g. 'Arrives in 4 business days', 'Arrives Dec 12-15 via FedEx'). Directly renderable; MUST NOT repeat the title. |

#### 가용성

| 이름      | 타입    | 필수 여부 | 설명                                                                                                                                |
| --------- | ------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| available | boolean | Optional  | Whether this can be obtained. See status for fulfillment details.                                                                   |
| status    | string  | Optional  | Qualifies available with fulfillment state. Well-known values: `in_stock`, `backorder`, `preorder`, `out_of_stock`, `discontinued`. |

#### 풀필먼트 목적지 필터

| 이름            | 타입   | 필수 여부 | 설명                                                                                                                           |
| --------------- | ------ | --------- | ------------------------------------------------------------------------------------------------------------------------------ |
| address_country | string | Optional  | The country, as a 2-letter ISO 3166-1 alpha-2 code (e.g. "US"). A 3-letter alpha-3 code or full country name MAY also be used. |
| address_region  | string | Optional  | The first-level administrative region within the country (e.g. a state or province such as California).                        |
| postal_code     | string | Optional  | The postal code (e.g. "94043").                                                                                                |
| location        | string | Optional  | A reference to the destination (e.g. store, pickup location, saved address).                                                   |

### 위치와 방법: `context`와 `filters`

- \*\*`context`\*\*에는 비즈니스가 `availability`를 보고할 때 참고하는 구속력 없는 힌트가 담깁니다. *구매자*가 있는 곳을 나타내는 대략적인 지역 필드와, 잠정적 비즈니스 위치를 나타내는 `location`입니다([위치 컨텍스트](#location-context) 참고). 시장 범위가 정해진 카탈로그에서는 비즈니스가 이 힌트로 결과를 좁힐 수 있습니다(**MAY**). 그 밖의 경우에는 힌트가 결과를 제거하지 않고 주석만 덧붙입니다.
- \*\*`filters.fulfills_to`\*\*는 상품을 *풀필먼트받을* 곳입니다. 단일 목적지 ([풀필먼트 목적지 필터](#fulfillment-destination-filter))이며, 값(대략적인 지역)이나 참조(`location` id, 즉 매장, 픽업 지점, 저장된 주소)로 지정합니다. 플랫폼은 둘 중 하나만 제공해야 하며(**SHOULD**), 둘 다 있으면 비즈니스는 더 구체적인 쪽(보통 `location`)을 사용해야 합니다(**SHOULD**). 이 필터는 결과를 그곳으로 풀필먼트할 수 있는 상품으로 제한하고 방법별 `availability`의 기준이 되며, `context`와 다를 수 있습니다(예: 선물).
- \*\*`filters.methods`\*\*는 결과를 특정 방법 유형(예: `["pickup"]`)으로 제한합니다.

`context`는 힌트일 뿐이고, 목적지를 지정하는 것은 `fulfills_to`입니다. 우선순위 규칙은 [위치 컨텍스트](#location-context)를 참고하세요.

### 예시

한 변형이 두 가지 풀필먼트 방법을 제공하는 예입니다. 하나는 구매자의 배송지로 배송하는 방법이고, 다른 하나는 지정된 매장에서 오늘 픽업하는 방법입니다. 방법마다 가용성이 따로 있으며, `pickup`은 확정된 위치를 id로 참조합니다.

```json
{
  "ucp": { "version": "draft" },
  "products": [
    {
      "id": "prod_kettle",
      "title": "Electric Kettle",
      "description": { "plain": "1.7L electric kettle." },
      "price_range": {
        "min": { "amount": 4999, "currency": "USD" },
        "max": { "amount": 4999, "currency": "USD" }
      },
      "variants": [
        {
          "id": "var_ss",
          "title": "Stainless Steel",
          "description": { "plain": "Stainless steel finish." },
          "price": { "amount": 4999, "currency": "USD" },
          "availability": { "available": true, "status": "in_stock" },
          "fulfillment": {
            "methods": [
              {
                "type": "shipping",
                "description": { "plain": "Ships to your address in 1–4 business days" },
                "availability": { "available": true, "status": "in_stock" },
                "options": [
                  {
                    "id": "std",
                    "title": "Standard",
                    "description": { "plain": "Arrives in 4 business days" }
                  },
                  {
                    "id": "exp",
                    "title": "Express",
                    "description": { "plain": "Next business day" }
                  }
                ]
              },
              {
                "type": "pickup",
                "description": { "plain": "Pickup today at Downtown Store" },
                "location": "loc_downtown",
                "availability": { "available": true, "status": "in_stock" }
              }
            ]
          }
        }
      ]
    }
  ]
}
```

각 방법은 변형을 풀필먼트하는 한 가지 방식이며, 저마다 `availability`가 있습니다. 각 방법의 `description`은 바로 렌더링할 수 있으므로, 플랫폼은 `type`을 인식하지 못해도 방법을 보여 줄 수 있습니다([렌더링](#rendering) 참고). 배송 방법의 `description`은 배송 기간을 미리 알려 주고, `options[]`가 이를 Standard, Express로 세분화합니다. 픽업에는 옵션이 없습니다. `options`는 선택 사항입니다.

## 구성

비즈니스와 플랫폼은 각자의 프로필에 풀필먼트 제약을 선언합니다. 비즈니스는 플랫폼 프로필을 가져와 그에 맞게 응답을 조정합니다.

`extends` 배열에는 이 확장이 풀필먼트를 추가할 기능을 나열합니다. 체크아웃은 권위 있는 거래 화면이고, 카탈로그는 디스커버리용입니다. 카탈로그에서도 풀필먼트를 노출하려는 비즈니스는 `extends`에 카탈로그 기능을 나열하고, 체크아웃으로만 한정하려면 카탈로그 기능을 빼면 됩니다.

### 플랫폼 프로필

플랫폼은 `platform_schema`로 자신의 렌더링 능력을 선언합니다.

| 이름                 | 타입    | 필수 여부 | 설명                                |
| -------------------- | ------- | --------- | ----------------------------------- |
| supports_multi_group | boolean | Optional  | Enables multiple groups per method. |

config를 생략하거나 `supports_multi_group: false`로 설정한 플랫폼은 단일 그룹 응답을 받습니다. 응답 형태는 항상 `methods[].groups[]`이며, 차이는 방법마다 `groups.length`가 1을 넘을 수 있느냐뿐입니다.

기본 선언(방법당 단일 그룹, 체크아웃과 카탈로그 디스커버리 모두에서 풀필먼트 노출):

```json
{
  "dev.ucp.shopping.fulfillment": [
    {
      "version": "draft",
      "spec": "https://ucp.dev/draft/specification/shopping/extensions/fulfillment",
      "schema": "https://ucp.dev/draft/schemas/shopping/fulfillment.json",
      "extends": [
        "dev.ucp.shopping.checkout",
        "dev.ucp.shopping.catalog.search",
        "dev.ucp.shopping.catalog.lookup"
      ]
    }
  ]
}
```

카탈로그 디스커버리를 노출하지 않는 당사자는 `extends`를 `"dev.ucp.shopping.checkout"`(문자열 형태)이나 요소가 하나인 배열로 좁힐 수 있습니다(MAY).

옵트인 선언(비즈니스가 방법당 여러 그룹을 반환할 수 있음(MAY)):

```json
{
  "dev.ucp.shopping.fulfillment": [
    {
      "version": "draft",
      "spec": "https://ucp.dev/draft/specification/shopping/extensions/fulfillment",
      "schema": "https://ucp.dev/draft/schemas/shopping/fulfillment.json",
      "extends": [
        "dev.ucp.shopping.checkout",
        "dev.ucp.shopping.catalog.search",
        "dev.ucp.shopping.catalog.lookup"
      ],
      "config": { "supports_multi_group": true }
    }
  ]
}
```

### 비즈니스 프로필

비즈니스는 `business_config`로 지원하는 풀필먼트 구성을 선언합니다.

| 이름                | 타입          | 필수 여부 | 설명                                                                                                                                                                                                        |
| ------------------- | ------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| multi_destination   | Array[object] | Optional  | Method types that permit multiple destinations within one cart (e.g. split shipping across addresses). Listing a method permits it; an omitted method does not. Open — businesses MAY list any method type. |
| method_combinations | Array[array]  | Optional  | Method-type combinations the business permits within one cart. Each inner array is a permitted set of method `type` values (e.g. shipping + pickup).                                                        |

```json
{
  "dev.ucp.shopping.fulfillment": [
    {
      "version": "draft",
      "spec": "https://ucp.dev/draft/specification/shopping/extensions/fulfillment",
      "schema": "https://ucp.dev/draft/schemas/shopping/fulfillment.json",
      "extends": [
        "dev.ucp.shopping.checkout",
        "dev.ucp.shopping.catalog.search",
        "dev.ucp.shopping.catalog.lookup"
      ],
      "config": {
        "multi_destination": [
          { "method": "shipping" }
        ],
        "method_combinations": [["shipping", "pickup"]]
      }
    }
  ]
}
```

이 예시는 배송지를 여러 주소로 지정할 수 있고, 장바구니에서 배송과 픽업을 섞을 수 있다는 뜻입니다.

### 비즈니스 응답 동작

**`supports_multi_group: false`(기본값)인 경우:**

- 비즈니스는 모든 상품을 **방법당 단일 그룹**으로 반드시 통합해야 합니다(**MUST**).
- 응답은 여전히 배열 구조인 `methods[].groups[]`를 사용하며 `groups.length === 1`입니다.
- 장바구니 상품에 필요하다면 비즈니스는 여전히 여러 방법(예: 배송 + 픽업)을 반환할 수 있습니다(**MAY**).

**`supports_multi_group: true`인 경우:**

- 비즈니스는 재고, 포장, 창고 로직에 따라 방법당 여러 그룹을 반환할 수 있습니다(**MAY**).
- 그룹 선택 UI(예: 패키지별 배송 속도 선택)는 플랫폼이 렌더링합니다.

### 방법 유형

`fulfillment_method.type`(체크아웃)과 `catalog_fulfillment_method.type`(카탈로그)은 열린 문자열 어휘 하나를 공유합니다. 표시 방식은 방법에 구애받지 않습니다. 플랫폼은 `type`과 관계없이 `description`과 `availability`를 렌더링해 모든 방법을 보여 줘야 하며(**SHOULD**)([렌더링](#rendering) 참고), `type`을 인식하지 못한다는 이유만으로 방법을 빼지 않는 것이 좋습니다(**SHOULD NOT**). `type`을 인식하면 유형별 UX를 추가로 제공할 수 있을 뿐입니다.

방법은 `type`과 풀필먼트 범위(무엇을 어디로 풀필먼트하는지)로 식별합니다. 비즈니스는 같은 범위 안의 변형(예: Standard vs Express)을 `options`로 모델링해야 하며(**SHOULD**), 옵션 수준의 세부 사항만 다른 방법을 여러 개 내보내지 않는 것이 좋습니다(**SHOULD NOT**). `type`이 같더라도 범위가 다르면 별개의 방법으로 둘 수 있습니다. 예를 들어 체크아웃에 서로 다른 목적지로 가는 `shipping` 방법이 두 개 있을 수 있습니다. 카탈로그에서는 한 방법이 확정된 위치 하나에 있는 단일 변형을 다루므로, `type`당 방법이 최대 하나가 됩니다.

**잘 알려진 값:**

| 값         | 의미                                                         |
| ---------- | ------------------------------------------------------------ |
| `shipping` | 운송사가 구매자의 주소로 배송합니다.                         |
| `pickup`   | 구매자가 지정된 위치에서 픽업합니다.                         |
| `curbside` | 구매자가 차에서 내리지 않고 위치에서 픽업합니다(드라이브업). |

**방법 유형 추가.** `type`은 열린 문자열이므로 비즈니스는 소비자 쪽을 바꾸지 않고도 언제든 새 값을 도입할 수 있습니다(MAY). 비즈니스가 그 값을 알리고 `filters.methods`로 필터링도 지원하면, 소비자는 다른 방법과 똑같이 보여 주면 됩니다.

**예시 — `home_installation` 추가.** 스키마 변경이나 등록이 필요 없습니다. 카탈로그와 체크아웃에서 이 값을 `type`으로 바로 내보내고, `filters.methods: ["home_installation"]`로 필터링합니다. 체크아웃 협상에 쓰려면 비즈니스 프로필 `config`에 그 동작을 선언합니다. 예를 들어 `method_combinations`에 `["shipping", "home_installation"]`을 넣으면 장바구니에서 배송 상품과 설치 상품을 섞을 수 있습니다([비즈니스 프로필](#business-profile) 참고). 카탈로그 변형의 방법에서는 다음과 같이 표현합니다.

```json
{
  "type": "home_installation",
  "description": { "plain": "Delivered and installed in your home" },
  "availability": {
    "available": true
  }
}
```

## 예시

### 기본

**구성:** 필요 없음(기본 동작)

```json
{
  "ucp": { ... },
  "id": "...",
  "status": "...",
  "currency": "...",
  "line_items": [ ... ],
  "totals": [ ... ],
  "links": [ ... ],
  "fulfillment": {
    "methods": [
      {
        "id": "method_1",
        "type": "shipping",
        "line_item_ids": ["shirt", "pants"],
        "selected_destination_id": "dest_1",
        "destinations": [
          {
            "type": "shipping_address",
            "id": "dest_1",
            "street_address": "123 Main St",
            "address_locality": "Springfield",
            "address_region": "IL",
            "postal_code": "62701",
            "address_country": "US"
          }
        ],
        "groups": [
          {
            "id": "package_1",
            "line_item_ids": ["shirt", "pants"],
            "selected_option_id": "standard",
            "options": [
              {
                "id": "standard",
                "title": "Standard Shipping",
                "description": { "plain": "Arrives Dec 12-15 via USPS" },
                "totals": [
                  {
                    "type": "total",
                    "amount": 500
                  }
                ]
              },
              {
                "id": "express",
                "title": "Express Shipping",
                "description": { "plain": "Arrives Dec 10-11 via FedEx" },
                "totals": [
                  {
                    "type": "total",
                    "amount": 1000
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

### 그룹 분할

**구성:** 플랫폼 프로필에 `config.supports_multi_group: true` 필요

비즈니스는 상품을 여러 패키지로 나누고, 구매자는 패키지마다 배송 요금을 선택합니다.

```json
{
  "ucp": { ... },
  "id": "...",
  "status": "...",
  "currency": "...",
  "line_items": [ ... ],
  "totals": [ ... ],
  "links": [ ... ],
  "fulfillment": {
    "methods": [
      {
        "id": "method_1",
        "type": "shipping",
        "line_item_ids": ["shirt", "pants"],
        "selected_destination_id": "dest_1",
        "destinations": [
          {
            "type": "shipping_address",
            "id": "dest_1",
            "street_address": "123 Main St",
            "address_locality": "Springfield",
            "address_region": "IL",
            "postal_code": "62701",
            "address_country": "US"
          }
        ],
        "groups": [
          {
            "id": "package_1",
            "line_item_ids": ["shirt"],
            "selected_option_id": "standard",
            "options": [
              {
                "id": "standard",
                "title": "Standard",
                "totals": [ {"type": "total", "amount": 500} ]
              },
              {
                "id": "express",
                "title": "Express",
                "totals": [ {"type": "total", "amount": 1000} ]
              }
            ]
          },
          {
            "id": "package_2",
            "line_item_ids": ["pants"],
            "selected_option_id": "express",
            "options": [
              {
                "id": "standard",
                "title": "Standard",
                "totals": [ {"type": "total", "amount": 500} ]
              },
              {
                "id": "express",
                "title": "Express",
                "totals": [ {"type": "total", "amount": 1000} ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

### 목적지 분할

**구성:** 비즈니스 프로필의 `config.multi_destination`에 `shipping` 나열

셔츠는 엄마(미국)에게, 바지는 할머니(홍콩)에게 보냅니다. 같은 유형의 방법 두 개에 각자 목적지가 있습니다.

```json
{
  "ucp": { ... },
  "id": "...",
  "status": "...",
  "currency": "...",
  "line_items": [ ... ],
  "totals": [ ... ],
  "links": [ ... ],
  "fulfillment": {
    "methods": [
      {
        "id": "method_1",
        "type": "shipping",
        "line_item_ids": ["shirt"],
        "selected_destination_id": "dest_mom",
        "destinations": [
          {
            "type": "shipping_address",
            "id": "dest_mom",
            "street_address": "123 Mom St",
            "address_locality": "Springfield",
            "address_region": "IL",
            "postal_code": "62701",
            "address_country": "US"
          }
        ],
        "groups": [
          {
            "id": "package_1",
            "line_item_ids": ["shirt"],
            "selected_option_id": "standard",
            "options": [
              {
                "id": "standard",
                "title": "Standard",
                "totals": [
                  {
                    "type": "total",
                    "amount": 500
                  }
                ]
              },
              {
                "id": "express",
                "title": "Express",
                "totals": [
                  {
                    "type": "total",
                    "amount": 1000
                  }
                ]
              }
            ]
          }
        ]
      },
      {
        "id": "method_2",
        "type": "shipping",
        "line_item_ids": ["pants"],
        "selected_destination_id": "dest_grandma",
        "destinations": [
          {
            "type": "shipping_address",
            "id": "dest_grandma",
            "street_address": "88 Queensway",
            "address_locality": "Hong Kong",
            "address_country": "HK"
          }
        ],
        "groups": [
          {
            "id": "package_2",
            "line_item_ids": ["pants"],
            "selected_option_id": "standard",
            "options": [
              {
                "id": "standard",
                "title": "Standard",
                "totals": [
                  {
                    "type": "total",
                    "amount": 500
                  }
                ]
              },
              {
                "id": "express",
                "title": "Express",
                "totals": [
                  {
                    "type": "total",
                    "amount": 1000
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```
