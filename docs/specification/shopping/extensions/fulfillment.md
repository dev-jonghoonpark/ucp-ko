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

풀필먼트 확장을 사용하면 비즈니스가 실물 상품 풀필먼트(배송, 픽업 등) 지원을 알릴 수
있다.

이 확장은 체크아웃 및/또는 카탈로그에 `fulfillment` 필드를 추가한다.

* **체크아웃**(`dev.ucp.shopping.checkout`) — 선택과 비용: 어떤 상품을 어디로, 어떤
    방법으로, 얼마의 가격과 예상 도착 시간(ETA)으로 보낼지.
* **카탈로그**(`dev.ucp.shopping.catalog.search` 및
    `dev.ucp.shopping.catalog.lookup`) — 디스커버리: 제공된 구매자 컨텍스트를 바탕으로
    각 변형(variant)이 자신에게 가능한 풀필먼트 옵션을 알린다.
    [카탈로그 디스커버리](#catalog-discovery)를 참고한다.

체크아웃에서 `fulfillment` 필드는 다음을 담는다.

* `methods[]` — 장바구니 상품에 적용되는 풀필먼트 방법(배송, 픽업 등)
    * `line_item_ids` — 이 방법으로 풀필먼트할 상품
    * `destinations[]` — 풀필먼트할 장소(주소, 매장 위치)
    * `groups[]` — 비즈니스가 생성한 패키지. 각 패키지에는 선택 가능한 `options[]`가 있다
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

## 위치 컨텍스트 { #location-context }

기본 [컨텍스트](../catalog/index.md#context)는 선택 사항인 `location`을 정의한다.
이는 비즈니스 네임스페이스 안에서 안정적이고 불투명한
[위치(Location)](../../glossary.md#commerce) 식별자다. 이 필드는 카탈로그 요청과
장바구니 및 체크아웃 생성·수정 요청에 나타난다([체크아웃 컨텍스트](../checkout/index.md#context)
참고). 풀필먼트는 이 필드를 추가하거나 재정의하지 않는다. 풀필먼트 확장이 활성화되면
이 필드의 효과를 정의할 뿐이다.

풀필먼트가 카탈로그나 체크아웃을 확장하는 경우, 해당 기능에 대한 요청의
`context.location`은 잠정적인 비즈니스 위치를 지정한다. 비즈니스는 이 위치를
기준으로 풀필먼트 가용성을 평가하고 초기 풀필먼트 선택지를 생성한다. 이 필드가
풀필먼트 목적지를 선택하지는 않는다.

다음 필드들은 한 값의 정밀도 수준이 아니라 각기 다른 역할을 맡는다.

| 필드 | 역할 |
| --- | --- |
| `context.location` | 풀필먼트를 평가할 기준이 되는 잠정적 비즈니스 위치. |
| 카탈로그 `filters.fulfills_to` | 상품을 풀필먼트할 명시적 목적지이자 결과에 대한 필터. |
| 카탈로그 `methods[].location` | 변형의 장소 기반 방법(예: `pickup`)에 대해 확정된 비즈니스 위치. |
| 체크아웃 `methods[].selected_destination_id` | 해당 방법의 `destinations[]` 중에서 선택된 목적지. |

우선순위는 각 필드가 관장하는 범위 안에서만 적용된다.

* 카탈로그 `filters.fulfills_to`가 있으면, 비즈니스는 풀필먼트 목적지와 방법의
    `availability`를 `context`가 아니라 이 필드로부터 반드시 확정해야 한다(**MUST**).
    다른 `context` 필드는 영향을 받지 않는다.
* 플랫폼이 체크아웃 방법에 `selected_destination_id`를 설정하면, 비즈니스는 그 방법의
    풀필먼트 범위에 대해 `context.location`이 아니라 참조된 목적지를 반드시 사용해야
    한다(**MUST**). 다른 방법은 영향을 받지 않는다.

**위치 이어 가기.** 같은 식별자가 이 필드들 중 여러 곳에 나타나더라도 각 역할이
합쳐지지는 않는다. 카탈로그 응답은 `loc_123`에서의 장소 기반 방법을 보고할 수 있다.
기본 장바구니 컨텍스트에 이미 `location`이 있으므로, `loc_123`은 장바구니 요청의
`context.location`으로 이어질 수 있다.
[장바구니-체크아웃 전환](../cart/index.md#cart-to-checkout-conversion)은 장바구니의
`context`로 체크아웃을 초기화하므로, 비즈니스는 `loc_123`을 사용해 초기 체크아웃
풀필먼트 목적지를 생성할 수 있다(**MAY**). 플랫폼이 `selected_destination_id`를
설정하면 그 명시적 선택이 우선한다.

## 스키마 { #schema }

풀필먼트는 물리적 배송이 필요한 상품에만 적용된다. 풀필먼트가 필요 없는 상품(예: 디지털
상품)은 방법에 할당할 필요가 없다.

### 속성 { #properties }

{{ extension_fields('fulfillment', 'shopping/extensions/fulfillment') }}

### 엔티티 { #entities }

#### 풀필먼트 { #fulfillment }

{{ schema_fields('types/fulfillment_resp', 'shopping/extensions/fulfillment') }}

#### 풀필먼트 방법 { #fulfillment-method }

{{ schema_fields('types/fulfillment_method_resp', 'shopping/extensions/fulfillment') }}

#### 풀필먼트 목적지 { #fulfillment-destination }

{{ schema_fields('types/fulfillment_destination_resp', 'shopping/extensions/fulfillment') }}

#### 배송 목적지 { #shipping-destination }

{{ schema_fields('types/shipping_destination_resp', 'shopping/extensions/fulfillment') }}

#### 비즈니스 위치 목적지 { #business-location-destination }

{{ schema_fields('types/location_destination_resp', 'shopping/extensions/fulfillment') }}

#### 위치 요약 { #location-summary }

{{ schema_fields('location_summary', 'shopping/extensions/fulfillment') }}

#### 풀필먼트 그룹 { #fulfillment-group }

{{ schema_fields('types/fulfillment_group_resp', 'shopping/extensions/fulfillment') }}

#### 풀필먼트 옵션 { #fulfillment-option }

{{ schema_fields('types/fulfillment_option_resp', 'shopping/extensions/fulfillment') }}

#### 풀필먼트 가능 방법 { #fulfillment-available-method }

{{ schema_fields('types/fulfillment_available_method_resp', 'shopping/extensions/fulfillment') }}

#### 합계 { #total }

{{ schema_fields('types/total_resp', 'shopping/extensions/fulfillment') }}

#### 우편 주소 { #postal-address }

{{ schema_fields('postal_address', 'shopping/extensions/fulfillment') }}

### 예시 { #example }

<!-- ucp:example schema=shopping/checkout op=read -->
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

## 목적지 { #destinations }

풀필먼트 방법의 `type`은 상품이 *어떻게* 풀필먼트되는지(예: 배송 또는 픽업)를
나타낸다. 목적지의 `type`은 풀필먼트가 *어디서* 이루어지는지(예: 배송 주소 또는
비즈니스 위치)를 나타낸다. 두 구분자는 방향에 따라 역할이 다르다. 비즈니스 응답에서는
모든 목적지가 필수 `type`을 가지므로 응답이 스스로를 설명한다. 플랫폼 요청에서는
방법의 계약이 `destinations[]`를 누가 작성하는지 결정하며, `type`이 생략된 경우의
기본 목적지 형태를 정의할 수 있다(MAY).

기본적으로 플랫폼은 방법의 `destinations[]`를 작성하지 않는다. 방법의 `type`을 키로
하는 계약은 플랫폼이 작성할 수 있는 요청 형태를 선택적으로 허용할 수 있다(**MAY**).

| 방법 `type` | 요청 `destinations[]` |
| --- | --- |
| `shipping` | 플랫폼이 작성 가능. 플랫폼이 구매자의 배송 주소 정보를 작성한다. |
| `pickup` | 플랫폼이 작성 불가. 비즈니스가 응답에서 위치를 나열하고, 플랫폼은 비즈니스 범위의 ID로 위치를 선택한다([선택과 위치 식별](#selection-and-location-identity) 참고). |
| 그 밖의 모든 방법 `type` | 플랫폼이 작성 불가(기본값). 비즈니스가 응답에서 목적지를 나열하고, 플랫폼은 `selected_destination_id`로 하나를 선택한다. |

목적지 `type`은 **응답에서는 필수, 요청에서는 선택 사항**이다. 응답에서는 모든
목적지가 `type`을 가지므로, 목적지는 어디에 나타나든 스스로를 설명한다. 요청에서는
방법별 계약이 플랫폼 작성 가능 형태를 허용하지 않는 한 `destinations[]`를 생략한다.

* 잘 알려진 `shipping` 방법에서는 모든 목적지가 배송 목적지다. `type`을 생략한
    목적지는 기본값 `shipping_address`가 된다.
* 잘 알려진 `pickup` 방법에서는 목적지가 응답 전용이다. 플랫폼은
    `selected_destination_id`로 위치를 선택한다
    ([선택과 위치 식별](#selection-and-location-identity) 참고).
* 그 밖의 방법 유형에는 기본값이 적용된다. `destinations[]`는 응답 전용이고, 응답의
    각 목적지는 필수인 `type`과 `id`로 스스로를 설명하며, 플랫폼은
    `selected_destination_id`로 하나를 선택한다. 해당 방법을 정의하는 계약(이 명세의
    향후 개정판이나 협상된 확장)은 응답 목적지 형태를 좁히거나 플랫폼 작성 가능 요청
    형태를 허용할 수 있다(**MAY**).

`destinations[]`를 포함하는 요청은 방법의 `type`도 반드시 포함해야 한다(MUST). 잘
알려진 값은 다음과 같다.

| 값 | 의미 |
| --- | --- |
| `shipping_address` | 평탄한 우편 주소 필드를 가진 배송 목적지. |
| `business_location` | 비즈니스 범위의 `id`로 식별되는 비즈니스 위치 목적지. |

추가 값은 협상된 확장이 정의한다. 그런 값에 고유한 목적지 필드는 협상된 확장의
스키마로 검증한다.

### 배송 목적지 { #shipping-destination_1 }

배송 목적지는 주소 자체를 담거나, 저장된 주소를 참조하는 `id`를 담을 수 있다.
비즈니스는 그 `id`를 자체 기록에서 확인하거나, 디지털 지갑이나 신원 제공자 같은 신뢰할
수 있는 자격 증명 제공자를 통해 확인할 수 있다.

#### 플랫폼 요청 { #platform-request }

<!-- ucp:example schema=shopping/types/shipping_destination op=update direction=request -->
```json
{
  "street_address": "123 Main St",
  "address_locality": "Springfield",
  "address_region": "IL",
  "postal_code": "62701",
  "address_country": "US"
}
```

#### 비즈니스 응답 { #business-response }

<!-- ucp:example schema=shopping/types/shipping_destination op=read direction=response -->
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

### 비즈니스 위치 목적지 { #business-location-destination_1 }

비즈니스 위치 목적지는 응답에만 나타난다. `pickup` 방법의 `destinations[]`는 요청
필드가 아니다. 플랫폼은 `destinations[]`에 비즈니스 위치 목적지를 작성해서는 안
되며(**MUST NOT**), 안정적이고 불투명한 비즈니스 범위의 ID를
`selected_destination_id`로 제출해 위치를 선택한다. 비즈니스는
`type: "business_location"`, `id`, 구매자에게 보이는 `name`을 반드시 반환해야
하며(**MUST**), `address`에 우편 주소를 반환할 수 있다(**MAY**).

#### 플랫폼 요청 { #platform-request_1 }

<!-- ucp:example schema=shopping/types/fulfillment_method op=update direction=request -->
```json
{
  "type": "pickup",
  "line_item_ids": ["shirt", "pants"],
  "selected_destination_id": "loc_downtown"
}
```

#### 비즈니스 응답 { #business-response_1 }

<!-- ucp:example schema=shopping/types/location_destination op=read direction=response -->
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

### 선택과 위치 식별 { #selection-and-location-identity }

`selected_destination_id`는 방법 안에서 선택된 목적지를 `id`로 식별하며, 비즈니스
위치를 선택하는 유일한 경로다. 비즈니스가 해당 방법에 대해 인식하는 안정적인 비즈니스
범위 ID라면 무엇이든 받으며, 비즈니스가 그 방법의 `destinations[]`에 (아직) 나열하지
않은 ID도 포함된다.

선택은 출처와 무관하다. null이 아닌 `selected_destination_id`에는 플랫폼이 카탈로그
응답, 위치 검색이나 조회, 이전 체크아웃, 또는 비즈니스가 해당 방법에 대해 인식하는 그
밖의 출처에서 얻은 ID를 담을 수 있다(**MAY**). ID를 어디서 얻었는지는 아래 계약에
영향을 주지 않는다.

비즈니스가 null이 아닌 `selected_destination_id`를 수락하면, 그 응답은 다음을
따른다.

* 방법에 같은 `selected_destination_id` 값을 반드시 담아야 한다(**MUST**).
* 같은 방법의 `destinations[]`에, `id`가 그 값과 같고
    [목적지](#destinations)에 명시된 대로 유형이 지정된 목적지를 정확히 하나 반드시
    포함해야 한다(**MUST**). 그래야 ID의 출처와 상관없이 응답이 스스로를 설명한다.

비즈니스는 선택된 목적지의 현재 가용성과 조건을 반드시 다시 검증해야 한다(**MUST**).
ID를 인식했다고 해서 재고가 확보되거나 자격이 보장되는 것은 아니다. 비즈니스는 다른
목적지로 몰래 대체해서는 안 된다(**MUST NOT**). 다른 `selected_destination_id`를
반환해서도 안 되고(**MUST NOT**), 제출된 ID를 유지하면서 다른 위치를 설명하는
목적지를 반환해서도 안 된다(**MUST NOT**). 제출된 선택을 이행할 수 없는 비즈니스는
선택을 대체하지 않고 거부한다.

[체크아웃 수정](../checkout/index.md#update-checkout)에서 제출된
`selected_destination_id`를 비즈니스가 수락할 수 없는 경우(해당 방법에 대해 인식되지
않는 ID이거나 재검증에 실패한 경우), 비즈니스는 거부된 수정 요청에 대한 일반 동작을
따른다. 즉, 현재 체크아웃을 반드시 변경하지 않은 채로 두어야 하며(**MUST**),
`severity: "recoverable"`이고 `path`가 시도된 방법의 `selected_destination_id`를
가리키는(예: `$.fulfillment.methods[0].selected_destination_id`) 에러 메시지와 함께
그 체크아웃을 반드시 반환해야 한다(**MUST**).
[에러 처리](../checkout/index.md#error-handling)와
[`path` 필드](../checkout/index.md#the-path-field)를 참고한다.

## 렌더링 { #rendering }

풀필먼트 옵션은 **방법에 구애받지 않는 렌더링(method-agnostic rendering)**을 염두에
두고 설계되었다. 플랫폼은 특정 방법 유형(배송, 픽업 등)을 이해하지 못해도 옵션을
의미 있게 보여 줄 수 있다. 비즈니스가 미리 계산된, 사람이 읽을 수 있는 필드를 제공하고
플랫폼은 이를 그대로 렌더링한다.

### 사람이 읽을 수 있는 필드 { #human-readable-fields }

| 위치              | 필드         | 필수 | 목적                                                 |
| --------------------- | ------------- | -------- | ------------------------------------------------------- |
| `groups[].options[]`  | `title`       | 예      | 형제 옵션과 구분되는 기본 레이블          |
| `groups[].options[]`  | `description` | 아니요       | 제목을 보충하는 맥락 정보                     |
| `groups[].options[]`  | `totals`      | 예      | 비용 내역: `total` 객체의 배열             |
| `available_methods[]` | `description` | 아니요       | 대체 가용성에 대한 독립적인 설명      |

### 비즈니스의 책임 { #business-responsibilities }

**`options[].title`의 경우:**

* 형제 옵션과 반드시 구분되어야 한다(**MUST**)
* 방법과 속도를 포함해야 한다(**SHOULD**)(예: "Express Shipping", "Curbside Pickup")
* `description`이 없을 때도 구매자가 결정을 내리기에 반드시 충분해야 한다(**MUST**)

**`options[].description`의 경우:**

* `title`이나 `total`을 반복해서는 안 된다(**MUST NOT**). 보충 맥락만 제공한다
* 배송 시점, 운송사, 그 밖에 결정에 관련된 세부 정보를 포함해야 한다(**SHOULD**)
* 완결된 구문이어야 한다(**SHOULD**)(예: "Arrives Dec 12-15 via FedEx")
* 제목만으로 의미가 분명하면 생략할 수 있다(**MAY**)

**`available_methods[].description`의 경우:**

* 무엇을, 언제, 어디서인지 설명하는 독립적인 문장이어야 한다(**MUST**)
* 플랫폼 대화에 그대로 사용할 수 있어야 한다(**SHOULD**)(예: "Pants available
    for pickup at Downtown Store today at 2pm")

**정렬의 경우:**

* 비즈니스는 `options[]`를 의미 있는 순서(예: 저렴한 순, 빠른 순)로 반환해야
    한다(**SHOULD**)
* 플랫폼은 그 순서를 유지해야 하지만(**SHOULD**), 순서를 바꿀 수도 있다(**MAY**)
    (예: 알려진 구매자 선호도나 화면별 순위에 맞추기 위해). 다만 방법/옵션 그룹 구조는
    반드시 유지해야 한다(**MUST**)

### 플랫폼의 책임 { #platform-responsibilities }

플랫폼은 풀필먼트를 범용적이고 렌더링 가능한 구조로 다뤄야 한다(**SHOULD**).

* 각 옵션을 `title`, `description`, `total`을 사용해 카드로 렌더링한다
* 반환된 모든 방법을 보여 준다. 방법 선택은 구매자가 결정할 일이다
* 방법과 옵션 구조를 유지한다. 병합하거나 중복을 제거하지 않는다. 정렬은 플랫폼이
    결정한다
* `available_methods[].description`을 사용해 구매자에게 대안을 보여 준다

플랫폼은 인식하는 방법 유형에 대해 향상된 UX(픽업용 매장 선택기, 배송용 운송사 로고)를
제공할 수 있지만(**MAY**), 이는 선택 사항이다. 기본 계약은 **`title` + `description`
+ `total`만으로 어떤 옵션이든 렌더링할 수 있다**는 것이다.

구매자가 플랫폼이 완전히 처리할 수 없는 옵션을 선택하면, 플랫폼은 `continue_url`을
사용해 비즈니스의 체크아웃으로 넘겨야 한다(**SHOULD**).

## 가능한 방법 { #available-methods }

가능한 방법(available methods)은 특정 방법으로 상품을 풀필먼트할 수 있는지, 그리고
언제 가능한지를 나타낸다. 사용 사례는 다음과 같다.

* **대체 방법**: "이 바지는 Downtown Store에서 픽업도 가능합니다"
* **나중에 풀필먼트**: 예약 주문, 먼 창고에서 출고되는 상품, 매장에 재고가 들어오면
  픽업

<!-- ucp:example schema=shopping/checkout op=read -->
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

`description` 필드를 통해 플랫폼은 구매자에게 대안을 보여 줄 수 있다.

> 🤖 셔츠와 바지는 $5에 배송되며 5~8일 후 도착합니다. 또는 바지는 4시간 후
> Downtown Store에서 픽업할 수 있습니다.

구매자가 픽업을 선택했지만 플랫폼이 분할 풀필먼트를 지원하지 않으면, 플랫폼은
`continue_url`을 사용해 비즈니스의 체크아웃으로 넘겨야 한다(**SHOULD**).

## 카탈로그 디스커버리 { #catalog-discovery }

풀필먼트 확장이 카탈로그 기능을 확장하면, 카탈로그 응답의 각 변형은 그 변형에 가능한
풀필먼트 방법과 가용성을 나열하는 `fulfillment` 객체를 가진다. 덕분에 카탈로그를
둘러보는 구매자는 상품을 어떻게 풀필먼트할 수 있는지 알 수 있다.

### 방법 { #methods }

`fulfillment.methods[]`는 변형에 가능한 방법을 나열한다. 각 방법은 다음을 가진다.

* `type` — 풀필먼트 방법(예: `shipping`, `pickup`).
    [방법 유형](#method-types)을 참고한다.
* `description` — 이 방법으로 변형이 어떻게 풀필먼트되는지에 대한, 구매자를 위한
    짧은 요약(예: "Ships in 2–4 business days"). 바로 렌더링할 수 있다.
    [렌더링](#rendering)을 참고한다.
* `availability` — 지정되었거나 추론된 위치에서 이 방법으로 변형을 이용할 수 있는지
    여부.
* `location` — `pickup` 같은 장소 기반 방법의 경우, 그 방법에 대해 확정된
    [위치](../../glossary.md#commerce)이자 해당 장소에 대한 비즈니스의 안정적인
    식별자. `location`에서의 픽업을 알리는 비즈니스는 체크아웃에서 해당 방법의
    `selected_destination_id`로 같은 ID를 반드시 수락해야 한다(**MUST**). 비즈니스는
    현재 가용성과 조건을 다시 검증한다. 디스커버리는 재고를 확보하지도, 자격을
    보장하지도 않는다. 선택 계약은 출처와 무관하다.
    [선택과 위치 식별](#selection-and-location-identity)을 참고한다.
* `options` — 이 방법 안의 구체적인 풀필먼트 선택지(예: Standard, Express).
    [옵션](#options)을 참고한다. 선택 사항이다.

카탈로그는 방법마다 단일 위치, 즉 `fulfills_to`로 지정되었거나 `context`에서 추론된
위치에 대한 가용성만 보고한다. 다른 위치를 찾고 비교하는 일은 별도로 처리한다.

변형 수준의 `availability`는 변형을 *어떤* 방법으로든 얻을 수 있는지를 나타낸다.
방법 자체의 `availability`가 그 방법에 대해서는 권위를 가진다. 방법에 `availability`가
명시되어 있으면, 소비자는 그 방법에 대해 반드시 그 값을 사용해야 하며(MUST), 변형
수준의 값으로부터 방법별 가용성을 추론해서는 안 된다(MUST NOT).

### 옵션 { #options }

방법은 `options[]`를 가질 수 있으며(MAY), 이는 풀필먼트 옵션의 전체 목록이 아니라
대표적인 일부다. 목적지나 전체 장바구니가 없는 상태에서 비즈니스는 구매자에게 의미 있는
경계 옵션(예: 가장 저렴한 옵션, 가장 빠른 옵션)을 미리 보여 줘야 한다(**SHOULD**).
더 구체적인 옵션은 라인 아이템과 목적지가 정해진 뒤 체크아웃에서 협상한다.

각 옵션은 `id`와 `title`(형제 옵션과 구분되는 짧은 레이블)을 가지며, 맥락을 위한
렌더링 가능한 `description`을 선택적으로 가진다. 이것은 공통 기반이다. 체크아웃에서는
같은 옵션에 비용과 시점 정보(`totals`, 운송사, 풀필먼트 시간)가 결합된다. 옵션은 열려
있으므로 비즈니스가 추가 필드를 덧붙일 수 있다(MAY). 방법이 옵션을 하나도 갖지 않고
`type`, `description`, `availability`만 보여 줄 수도 있다(MAY). 옵션은 그룹 계층 없이
방법 바로 아래에 중첩된다(체크아웃의 `methods[].groups[].options[]`와 다르다).

디스커버리에서 얻은 옵션 `id`를 통해 구매자의 선택을 이어 갈 수 있다. 비즈니스는
체크아웃에서 같은 ID를 `methods[].groups[].selected_option_id`로 수락해야
한다(**SHOULD**). 이 ID는 최선의 노력(best-effort) 수준의 핸들일 뿐 일치를 보장하지
않는다. 단일 상품에 대해 발견된 옵션은 체크아웃에서 달라질 수 있는데, 체크아웃에서는
다른 상품, 수량, 합배송이 옵션을 바꾸기 때문이다.

### 형태 { #shapes }

#### 카탈로그 풀필먼트 { #catalog-fulfillment }

{{ extension_schema_fields('fulfillment.json#/$defs/catalog_fulfillment', 'shopping/extensions/fulfillment') }}

#### 카탈로그 풀필먼트 방법 { #catalog-fulfillment-method }

{{ extension_schema_fields('fulfillment.json#/$defs/catalog_fulfillment_method', 'shopping/extensions/fulfillment') }}

#### 풀필먼트 옵션 기반 { #fulfillment-option-base }

{{ schema_fields('types/fulfillment_option_base', 'shopping/extensions/fulfillment') }}

#### 가용성 { #availability }

{{ schema_fields('types/availability', 'shopping/extensions/fulfillment') }}

#### 풀필먼트 목적지 필터 { #fulfillment-destination-filter }

{{ schema_fields('types/fulfillment_destination_filter', 'shopping/extensions/fulfillment') }}

### 위치와 방법: `context`와 `filters` { #location-and-method-context-and-filters }

* **`context`**는 비즈니스가 `availability`를 보고할 때 사용하는 구속력 없는 힌트를
    담는다. *구매자*가 있는 곳을 나타내는 대략적인 지역 필드와, 잠정적 비즈니스 위치를
    나타내는 `location`이다([위치 컨텍스트](#location-context) 참고). 시장 범위가
    지정된 카탈로그에서는 비즈니스가 이 힌트로 결과를 좁힐 수 있다(**MAY**). 그렇지
    않으면 힌트는 결과를 제거하지 않고 주석만 단다.
* **`filters.fulfills_to`**는 상품이 *풀필먼트되는* 곳이다. 단일 목적지
    ([풀필먼트 목적지 필터](#fulfillment-destination-filter))로, 값(대략적인 지역)이나
    참조(`location` id, 즉 매장, 픽업 지점, 저장된 주소)로 지정한다. 플랫폼은 둘 중
    하나만 제공해야 하며(**SHOULD**), 둘 다 있으면 비즈니스는 더 구체적인 쪽(보통
    `location`)을 사용해야 한다(**SHOULD**). 이 필터는 결과를 그곳으로 풀필먼트할 수
    있는 것으로 제한하고 방법의 `availability`의 기준이 되며, `context`와 다를 수
    있다(예: 선물).
* **`filters.methods`**는 결과를 특정 방법 유형(예: `["pickup"]`)으로 제한한다.

`context`는 힌트만 줄 뿐이고, `fulfills_to`가 목적지를 지정한다. 우선순위 규칙은
[위치 컨텍스트](#location-context)를 참고한다.

### 예시 { #example_1 }

한 변형이 두 가지 풀필먼트 방법을 제공한다. 구매자의 배송지로 배송하는 방법과, 지정된
매장에서 오늘 픽업하는 방법이다. 각 방법은 자체 가용성을 가지며, `pickup`은 확정된
위치를 id로 참조한다.

<!-- ucp:example schema=shopping/fulfillment def=fulfillment_search_response op=read -->
```json
{
  "ucp": { "version": "{{ ucp_version }}" },
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

각 방법은 변형을 풀필먼트할 수 있는 하나의 방식이며 자체 `availability`를 가진다. 각
방법의 `description`은 바로 렌더링할 수 있으므로, 플랫폼은 `type`을 인식하지 못해도
이를 보여 줄 수 있다([렌더링](#rendering) 참고). 배송 방법의 `description`은 배송
기간을 미리 보여 주고, `options[]`가 이를 세분화한다(Standard, Express). 픽업에는
옵션이 없다. `options`는 선택 사항이다.

## 구성 { #configuration }

비즈니스와 플랫폼은 각자의 프로필에 풀필먼트 제약을 선언한다. 비즈니스는 플랫폼 프로필을
가져와 그에 맞게 응답을 조정한다.

`extends` 배열은 이 확장이 풀필먼트를 추가하는 기능을 나열한다. 체크아웃은 권위 있는
거래 화면이고, 카탈로그는 디스커버리용이다. 비즈니스는 카탈로그에서 풀필먼트를
노출하려면 `extends`에 카탈로그 기능을 나열하고, 체크아웃으로만 범위를 한정하려면 이를
생략한다.

### 플랫폼 프로필 { #platform-profile }

플랫폼은 `platform_schema`를 사용해 자신의 렌더링 능력을 선언한다.

{{ schema_fields('types/platform_fulfillment_config', 'shopping/extensions/fulfillment') }}

config를 생략하거나 `supports_multi_group: false`로 설정한 플랫폼은 단일 그룹 응답을
받는다. 응답 형태는 항상 `methods[].groups[]`이며, 차이는 각 방법 안에서
`groups.length`가 1을 넘을 수 있는지 여부다.

기본 선언(방법당 단일 그룹, 체크아웃과 카탈로그 디스커버리 모두에서 풀필먼트 노출):

<!-- ucp:example schema=profile def=platform_schema target=$.ucp.capabilities -->
```json
{
  "dev.ucp.shopping.fulfillment": [
    {
      "version": "{{ ucp_version }}",
      "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/fulfillment",
      "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
      "extends": [
        "dev.ucp.shopping.checkout",
        "dev.ucp.shopping.catalog.search",
        "dev.ucp.shopping.catalog.lookup"
      ]
    }
  ]
}
```

카탈로그 디스커버리를 노출하지 않는 당사자는 `extends`를
`"dev.ucp.shopping.checkout"`(문자열 형태)이나 단일 요소 배열로 좁힐 수 있다(MAY).

옵트인 선언(비즈니스가 방법당 여러 그룹을 반환할 수 있음(MAY)):

<!-- ucp:example schema=profile def=platform_schema target=$.ucp.capabilities -->
```json
{
  "dev.ucp.shopping.fulfillment": [
    {
      "version": "{{ ucp_version }}",
      "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/fulfillment",
      "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
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

### 비즈니스 프로필 { #business-profile }

비즈니스는 `business_config`를 사용해 지원하는 풀필먼트 구성을 선언한다.

{{ schema_fields('types/business_fulfillment_config', 'shopping/extensions/fulfillment') }}

<!-- ucp:example schema=profile def=business_schema target=$.ucp.capabilities -->
```json
{
  "dev.ucp.shopping.fulfillment": [
    {
      "version": "{{ ucp_version }}",
      "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/fulfillment",
      "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
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

이 예시는 배송을 여러 주소로 보낼 수 있고, 장바구니에서 배송과 픽업을 섞을 수 있다는
뜻이다.

### 비즈니스 응답 동작 { #business-response-behavior }

**`supports_multi_group: false`(기본값)인 경우:**

* 비즈니스는 모든 상품을 **방법당 단일 그룹**으로 반드시 통합해야 한다(**MUST**)
* 응답은 여전히 배열 구조 `methods[].groups[]`를 사용하며 `groups.length === 1`이다
* 장바구니 상품에 필요하다면 비즈니스는 여전히 여러 방법(예: 배송 + 픽업)을 반환할 수
    있다(**MAY**)

**`supports_multi_group: true`인 경우:**

* 비즈니스는 재고, 포장, 창고 로직에 따라 방법당 여러 그룹을 반환할 수 있다(**MAY**)
* 그룹 선택 UI(예: 패키지별 배송 속도 선택)를 렌더링하는 책임은 플랫폼에 있다

### 방법 유형 { #method-types }

`fulfillment_method.type`(체크아웃)과 `catalog_fulfillment_method.type`(카탈로그)은
하나의 열린 문자열 어휘를 공유한다. 표시 방식은 방법에 구애받지 않는다. 플랫폼은
`type`과 관계없이 `description`과 `availability`를 렌더링해 모든 방법을 보여 줘야
하며(**SHOULD**)([렌더링](#rendering) 참고), `type`을 인식하지 못한다는 이유만으로
방법을 생략하지 않아야 한다(**SHOULD NOT**). `type`을 인식하면 유형별 UX를 선택적으로
제공할 수 있을 뿐이다.

방법은 `type`과 풀필먼트 범위(무엇을 어디로 풀필먼트하는지)로 식별된다. 비즈니스는
같은 범위 안의 변형(예: Standard vs Express)을 `options`로 모델링해야 하며(**SHOULD**),
옵션 수준의 세부 사항만 다른 여러 방법을 내보내지 않아야 한다(**SHOULD NOT**). 같은
`type`의 방법이라도 범위가 다르면 유효하다. 예를 들어 체크아웃에는 서로 다른 목적지로
가는 두 `shipping` 방법이 있을 수 있다. 카탈로그에서는 한 방법이 하나의 확정된 위치에
있는 단일 변형을 다루므로, `type`당 최대 하나의 방법으로 줄어든다.

**잘 알려진 값:**

| 값 | 의미 |
| --- | --- |
| `shipping` | 운송사가 구매자의 주소로 배송한다. |
| `pickup` | 구매자가 지정된 위치에서 픽업한다. |
| `curbside` | 구매자가 차량에서 내리지 않고 위치에서 픽업한다(드라이브업). |

**방법 유형 추가.** `type`은 열린 문자열이므로, 비즈니스는 소비자 측 변경 없이 언제든
새 값을 도입할 수 있다(MAY). 비즈니스가 그 값을 알리고(`filters.methods`로 필터링도
하고), 소비자는 다른 방법과 똑같이 이를 보여 준다.

**예시 — `home_installation` 추가.** 스키마 변경이나 등록이 필요 없다. 카탈로그와
체크아웃에서 이 값을 `type`으로 바로 내보내고,
`filters.methods: ["home_installation"]`로 필터링한다. 체크아웃 협상을 위해서는 비즈니스 프로필 `config`에 그 동작을 선언한다.
예를 들어 `method_combinations`에 `["shipping", "home_installation"]`을 포함하면
장바구니에서 배송 상품과 설치 상품을 섞을 수 있다([비즈니스 프로필](#business-profile)
참고). 카탈로그 변형의 방법에서는 다음과 같다.

<!-- ucp:example schema=shopping/fulfillment def=catalog_fulfillment_method op=read -->
```json
{
  "type": "home_installation",
  "description": { "plain": "Delivered and installed in your home" },
  "availability": {
    "available": true
  }
}
```

## 예시 { #examples }

### 기본 { #basic }

**구성:** 필요 없음(기본 동작)

<!-- ucp:example schema=shopping/checkout op=read -->
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

### 그룹 분할 { #split-groups }

**구성:** 플랫폼 프로필에 `config.supports_multi_group: true`가 필요하다

비즈니스는 상품을 여러 패키지로 나누고, 구매자는 패키지별로 배송 요금을 선택한다.

<!-- ucp:example schema=shopping/checkout op=read -->
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

### 목적지 분할 { #split-destinations }

**구성:** 비즈니스 프로필의 `config.multi_destination`에 `shipping`이 나열되어 있다

셔츠는 엄마(미국)에게, 바지는 할머니(홍콩)에게 배송된다. 같은 유형의 방법 두 개가 각자
자신의 목적지를 가진다.

<!-- ucp:example schema=shopping/checkout op=read -->
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
