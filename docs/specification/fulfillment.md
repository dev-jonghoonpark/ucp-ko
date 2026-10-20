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

# 풀필먼트 확장 { #fulfillment-extension }

## 개요 { #overview }

풀필먼트 확장을 사용하면 비즈니스가 배송, 픽업 같은 실물 상품 풀필먼트를 지원한다고
알릴 수 있습니다.

이 확장은 체크아웃에 `fulfillment` 필드를 추가하며, 이 필드에는 다음이 담깁니다.

* `methods[]` — 장바구니 상품에 적용되는 풀필먼트 방법(배송, 픽업 등)
    * `line_item_ids` — 이 방법으로 풀필먼트할 상품
    * `destinations[]` — 풀필먼트할 장소(주소, 매장 위치)
    * `groups[]` — 비즈니스가 생성한 패키지. 패키지마다 선택할 수 있는 `options[]`가 있음
* `available_methods[]` — 상품별 재고 가용성(선택 사항)

**멘탈 모델:**

* `methods[0]` 배송
    * `line_item_ids` 👕👖
    * `selected_destination_id` = `destinations[0].id` 🔘✅ 123 Fake St
    * `groups[0]` 📦👕👖
        * `selected_option_id` = `options[0].id` 🔘✅ 일반 배송 $5
        * `options[1]` 🔘 특급 배송 $10
* `methods[1]` 매장 픽업
    * `line_item_ids` 👞
    * `selected_destination_id` = `destinations[0].id` 🔘✅ Uptown Store
    * `groups[0]` 📦👞
        * `selected_option_id` = `options[0].id` 🔘✅ 매장 내 픽업
        * `options[1]` 🔘 커브사이드 픽업

## 스키마 { #schema }

풀필먼트는 물리적 배송이 필요한 상품에만 적용됩니다. 풀필먼트가 필요 없는 상품(예:
디지털 상품)은 방법에 할당하지 않아도 됩니다.

### 속성 { #properties }

{{ extension_fields('fulfillment', 'fulfillment') }}

### 엔티티 { #entities }

#### 풀필먼트 { #fulfillment }

{{ schema_fields('types/fulfillment_resp', 'fulfillment') }}

#### 풀필먼트 방법 { #fulfillment-method }

{{ schema_fields('types/fulfillment_method_resp', 'fulfillment') }}

#### 풀필먼트 목적지 { #fulfillment-destination }

{{ schema_fields('types/fulfillment_destination_resp', 'fulfillment') }}

#### 배송 목적지 { #shipping-destination }

{{ schema_fields('types/shipping_destination_resp', 'fulfillment') }}

#### 매장 위치 { #retail-location }

{{ schema_fields('types/retail_location_resp', 'fulfillment') }}

#### 풀필먼트 그룹 { #fulfillment-group }

{{ schema_fields('types/fulfillment_group_resp', 'fulfillment') }}

#### 풀필먼트 옵션 { #fulfillment-option }

{{ schema_fields('types/fulfillment_option_resp', 'fulfillment') }}

#### 풀필먼트 가능 방법 { #fulfillment-available-method }

{{ schema_fields('types/fulfillment_available_method_resp', 'fulfillment') }}

#### 합계 { #total }

{{ schema_fields('types/total_resp', 'fulfillment') }}

#### 우편 주소 { #postal-address }

{{ schema_fields('postal_address', 'fulfillment') }}

### 예시 { #example }

```json
{
  "fulfillment": {
    "methods": [
      {
        "id": "method_1",
        "type": "shipping",
        "line_item_ids": ["shirt", "pants"],
        "selected_destination_id": "dest_1",
        "destinations": [
          {
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
                "description": "Arrives Dec 12-15 via USPS",
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
                "description": "Arrives Dec 10-11 via FedEx",
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

## 렌더링 { #rendering }

풀필먼트 옵션은 **방법에 구애받지 않는 렌더링(method-agnostic rendering)**을 염두에
두고 설계되었습니다. 플랫폼은 특정 방법 유형(배송, 픽업 등)을 이해하지 못해도 옵션을
의미 있게 보여 줄 수 있습니다. 비즈니스가 미리 계산해 둔, 사람이 읽을 수 있는 필드를
제공하면 플랫폼은 이를 그대로 렌더링합니다.

### 사람이 읽을 수 있는 필드 { #human-readable-fields }

| 위치              | 필드         | 필수 | 목적                                                 |
| --------------------- | ------------- | -------- | ------------------------------------------------------- |
| `groups[].options[]`  | `title`       | 예      | 형제 옵션과 구분되는 기본 레이블          |
| `groups[].options[]`  | `description` | 아니요       | 제목을 보충하는 맥락 정보                     |
| `groups[].options[]`  | `total`       | 예      | 최소 단위 가격(아직 확정되지 않았으면 null일 수 있음) |
| `available_methods[]` | `description` | 아니요       | 대체 가용성에 대한 독립적인 설명      |

### 비즈니스의 책임 { #business-responsibilities }

**`options[].title`의 경우:**

* 형제 옵션과 반드시 구분되어야 합니다(**MUST**).
* 방법과 속도를 포함해야 합니다(**SHOULD**)(예: "Express Shipping", "Curbside Pickup").
* `description`이 없어도 구매자가 결정을 내리기에 반드시 충분해야 합니다(**MUST**).

**`options[].description`의 경우:**

* `title`이나 `total`을 반복해서는 안 됩니다(**MUST NOT**). 보충 맥락만 제공합니다.
* 배송 시점, 운송사, 그 밖에 결정에 필요한 세부 정보를 포함해야 합니다(**SHOULD**).
* 완결된 구문이어야 합니다(**SHOULD**)(예: "Arrives Dec 12-15 via FedEx").
* 제목만으로 의미가 분명하면 생략할 수 있습니다(**MAY**).

**`available_methods[].description`의 경우:**

* 무엇을, 언제, 어디서 이용할 수 있는지 설명하는 독립적인 문장이어야 합니다(**MUST**).
* 플랫폼 대화에 그대로 쓸 수 있어야 합니다(**SHOULD**)(예: "Pants available
    for pickup at Downtown Store today at 2pm").

**정렬의 경우:**

* 비즈니스는 `options[]`를 의미 있는 순서(예: 저렴한 순, 빠른 순)로 반환해야
    합니다(**SHOULD**).
* 플랫폼은 제공된 순서대로 옵션을 렌더링해야 합니다(**SHOULD**).

### 플랫폼의 책임 { #platform-responsibilities }

플랫폼은 풀필먼트를 범용적이고 렌더링 가능한 구조로 다뤄야 합니다(**SHOULD**).

* 각 옵션을 `title`, `description`, `total`로 카드 형태로 렌더링합니다.
* 비즈니스가 제공한 순서대로 옵션을 보여 줍니다.
* 반환된 방법을 모두 보여 줍니다. 어떤 방법을 고를지는 구매자가 결정합니다.
* `available_methods[].description`으로 구매자에게 대안을 보여 줍니다.

플랫폼은 인식하는 방법 유형이라면 향상된 UX(픽업용 매장 선택기, 배송용 운송사 로고)를
제공할 수 있지만(**MAY**), 어디까지나 선택 사항입니다. 기본 계약은 **`title` + `description`
+ `total`만으로 어떤 옵션이든 렌더링할 수 있어야 한다**는 것입니다.

플랫폼이 완전히 처리할 수 없는 옵션을 구매자가 선택하면, 플랫폼은 `continue_url`로
비즈니스의 체크아웃에 넘겨야 합니다(**SHOULD**).

## 가능한 방법 { #available-methods }

가능한 방법(available methods)은 상품을 특정 방법으로 풀필먼트할 수 있는지, 그리고
언제 가능한지를 나타냅니다. 다음과 같은 경우에 씁니다.

* **대체 방법**: "이 바지는 Downtown Store에서 픽업도 가능합니다"
* **나중에 풀필먼트**: 예약 주문, 먼 창고에서 출고되는 상품, 매장에 재고가 들어오면 픽업

```json
{
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

> 🤖 셔츠와 바지는 $5에 배송되며 5~8일 후 도착합니다. 또는 바지는 4시간 후
> Downtown Store에서 픽업할 수 있습니다.

구매자가 픽업을 선택했는데 플랫폼이 분할 풀필먼트를 지원하지 않으면, 플랫폼은
`continue_url`로 비즈니스의 체크아웃에 넘겨야 합니다(**SHOULD**).

## 구성 { #configuration }

비즈니스와 플랫폼은 각자의 프로필에 풀필먼트 제약을 선언합니다. 비즈니스는 플랫폼 프로필을
가져와 그에 맞게 응답을 조정합니다.

### 플랫폼 프로필 { #platform-profile }

플랫폼은 `platform_schema`로 자신의 렌더링 능력을 선언합니다.

{{ schema_fields('types/platform_fulfillment_config', 'fulfillment') }}

config를 생략하거나 `supports_multi_group: false`로 설정한 플랫폼은 단일 그룹 응답을
받습니다. 응답 형태는 항상 `methods[].groups[]`이며, 차이는 방법마다 `groups.length`가
1을 넘을 수 있느냐뿐입니다.

```json
// Default: single group per method
{ "dev.ucp.shopping.fulfillment": [{"version": "{{ ucp_version }}"}] }

// Opt-in: business MAY return multiple groups per method
{ "dev.ucp.shopping.fulfillment": [{"version": "{{ ucp_version }}", "config": { "supports_multi_group": true }}] }
```

### 비즈니스 프로필 { #business-profile }

비즈니스는 `merchant_config`로 지원하는 풀필먼트 구성을 선언합니다.

{{ schema_fields('types/merchant_fulfillment_config', 'fulfillment') }}

```json
{
  "capabilities": {
    "dev.ucp.shopping.fulfillment": [
      {
        "version": "{{ ucp_version }}",
        "config": {
          "allows_multi_destination": {
            "shipping": true
          },
          "allows_method_combinations": [["shipping", "pickup"]]
        }
      }
    ]
  }
}
```

이 예시는 배송지를 여러 주소로 지정할 수 있고, 장바구니에서 배송과 픽업을 섞을 수
있다는 뜻입니다.

### 비즈니스 응답 동작 { #business-response-behavior }

**`supports_multi_group: false`(기본값)인 경우:**

* 비즈니스는 모든 상품을 **방법당 단일 그룹**으로 반드시 통합해야 합니다(**MUST**).
* 응답은 여전히 배열 구조인 `methods[].groups[]`를 사용하며 `groups.length === 1`입니다.
* 장바구니 상품에 필요하다면 비즈니스는 여전히 여러 방법(예: 배송 + 픽업)을 반환할 수
    있습니다(**MAY**).

**`supports_multi_group: true`인 경우:**

* 비즈니스는 재고, 포장, 창고 로직에 따라 방법당 여러 그룹을 반환할 수 있습니다(**MAY**).
* 그룹 선택 UI(예: 패키지별 배송 속도 선택)는 플랫폼이 렌더링합니다.

### 새 방법 추가 { #adding-new-methods }

새로운 방법 유형(예: `local_delivery`)으로 풀필먼트를 확장하는 확장은 반드시 다음을
충족하는 확장 스키마를 추가해야 합니다(**MUST**).

1. `fulfillment_method`의 `type` enum에 해당 방법을 추가합니다.
2. 대응하는 비즈니스 구성 옵션을 추가합니다.
    * `allows_multi_destination.local_delivery: boolean`
    * `allows_method_combinations` 항목 enum(`"local_delivery"` 포함)

참고: 플랫폼의 `supports_multi_group`은 방법에 구애받지 않는 단일 boolean이므로 확장이
필요 없습니다.

## 예시 { #examples }

### 기본 { #basic }

**구성:** 필요 없음(기본 동작)

```json
{
  "fulfillment": {
    "methods": [
      {
        "id": "method_1",
        "type": "shipping",
        "line_item_ids": ["shirt", "pants"],
        "selected_destination_id": "dest_1",
        "destinations": [
          {
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
                "description": "Arrives Dec 12-15 via USPS",
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
                "description": "Arrives Dec 10-11 via FedEx",
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

### 그룹 분할 { #split-groups }

**구성:** 플랫폼 프로필에 `config.supports_multi_group: true` 필요

비즈니스는 상품을 여러 패키지로 나누고, 구매자는 패키지마다 배송 요금을 선택합니다.

```json
{
  "fulfillment": {
    "methods": [
      {
        "id": "method_1",
        "type": "shipping",
        "line_item_ids": ["shirt", "pants"],
        "selected_destination_id": "dest_1",
        "destinations": [
          {
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

### 목적지 분할 { #split-destinations }

**구성:** 비즈니스 프로필에
`config.allows_multi_destination.shipping: true` 필요

셔츠는 엄마(미국)에게, 바지는 할머니(홍콩)에게 보냅니다. 같은 유형의 방법 두 개에
각자 목적지가 있습니다.

```json
{
  "fulfillment": {
    "methods": [
      {
        "id": "method_1",
        "type": "shipping",
        "line_item_ids": ["shirt"],
        "selected_destination_id": "dest_mom",
        "destinations": [
          {
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
