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

# 위치 조회 기능 { #location-lookup-capability }

* **기능(Capability) 이름:** `dev.ucp.common.location.lookup`

식별자를 실제 물리적 위치(Location)로 해석한다.
여러 위치의 전체 상세 정보를 일괄 조회하거나, 단일 위치를 조회(위치 전용 상세
페이지에 유용)할 수 있다. 선택적으로 검색(Search)과 동일한 명시적 공간 관계와
필터로 결과를 좁힐 수 있다([결과 좁히기](#refinement) 참고).

## 작업 { #operation }

| 작업              | 설명                                          |
| :--------------------- | :--------------------------------------------------- |
| **위치 조회** | 식별자로 하나 또는 여러 위치를 조회한다. |

## 지원하는 식별자 { #supported-identifiers }

`ids` 파라미터는 식별자 배열을 받는다. 비즈니스는 위치의 안정적인 정식(canonical)
식별자인 `Location.id`로 조회하는 기능을 반드시 지원해야 하며(**MUST**), 보조
식별자나 별칭 식별자를 추가로 지원할 수 있다(**MAY**).

비즈니스는 요청 안의 중복 식별자를 반드시 제거해야 한다(**MUST**). 하나의
식별자가 여러 위치와 일치하면 비즈니스는 일치하는 위치들을 반환하며, 결과 집합의
크기를 제한할 수 있다(**MAY**). 여러 식별자가 같은 위치로 해석되면 비즈니스는 그
위치를 반드시 한 번만 반환해야 한다(**MUST**).

### 플랫폼 측 대응 관계 파악 { #platform-correlation }

응답은 순서를 보장하지 않는다. 각 위치에는 요청한 식별자 중 어떤 것이 그 위치로
해석되었는지 알려 주는 `inputs` 배열이 들어 있다. 모든 항목에는 조회 요청에서 그대로
복사한 필수 `id`가 있다.

요청의 여러 식별자가 같은 위치로 해석될 수 있다. 이 경우 해당 위치의 `inputs`
배열에는 해석된 식별자마다 항목이 하나씩 들어간다. 비즈니스는 `inputs` 항목이 없는
위치를 조회 응답에 포함해서는 안 된다(**MUST NOT**).

### 배치 크기 { #batch-size }

비즈니스는 요청당 최소 10개의 식별자를 받아야 하며(**SHOULD**), 최대 배치 크기를
강제할 수 있다(**MAY**). 최대값은 중복 식별자를 제거한 뒤에 적용된다.

요청이 최대값을 넘으면, 비즈니스는 요청 순서대로 서로 다른 식별자를 최대값만큼
앞에서부터 처리하고 그 앞부분에 대해 성공 응답을 반드시 반환해야 한다(**MUST**).
이때 `code: "batch_limit_applied"`인 정보성 메시지를 반드시 포함해야 하며
(**MUST**), 메시지 내용에는 적용된 최대값과 처리되지 않은 식별자 수를 반드시
밝혀야 한다(**MUST**). 비즈니스는 생략된 뒷부분을 평가하거나 그 식별자들을 해석되지
않은 것으로 보고해서는 안 된다(**MUST NOT**). 플랫폼은 생략된 뒷부분을 해석되지
않은 것으로 해석해서는 안 되며(**MUST NOT**), 이후 요청에서 다시 제출할 수
있다(**MAY**).

### 결과 좁히기 { #refinement }

비즈니스는 먼저 식별자를 해석하고 반복된 값을 제거한다. 그다음 선택 사항인 최상위
`distance`, `serves` 관계와 `filters` 조건으로 해석된 집합을 좁힌다. 제공된 모든
조건은 AND로 결합된다. 즉, 해석된 위치는 제공된 모든 관계와 필터를 만족할 때만
반환된다. 관계와 조건은 검색과 같은 스키마와 의미를 사용한다.
[공간 관계](search.md#spatial-relations),
[검색 필터](search.md#search-filters),
[상품 재고 필터](search.md#item-availability-filter)를 참고한다.

예를 들어 플랫폼이 `["loc_downtown", "loc_uptown"]`을 영업시간 필터
`{"open_at": "2026-05-18T17:00:00Z"}`와 함께 요청한다면 다음과 같다.

1. 비즈니스는 먼저 두 식별자를 각각의 위치로 해석한다.
2. 비즈니스는 해석된 각 위치에 대해 제공된 시각을 평가한다.
3. 그 시각에 `loc_uptown`이 영업하지 않는다면, 비즈니스는 이를 제외하고
    `loc_downtown`만 반환한다.

해석되지 않은 식별자에는 대응하는 반환 위치나 `inputs` 항목이 없다. 식별자가
해석된 위치가 제공된 조건 중 하나라도 만족하지 못하면(`filters.items` 식별자를
알 수 없거나, 해당 위치에서 제공할 수 없거나, 평가할 수 없는 경우 포함) 그 위치와
대응하는 `inputs` 항목은 생략된다. 이 경우에도 요청은 성공한다. 비즈니스는
해석되지 않은 위치 식별자에 대해 정보성 `not_found` 메시지를 첨부할 수
있지만(**MAY**), 결과 좁히기에서 제외된 경우에는 식별자별 설명이 보장되지 않는다.

### 요청 { #request }

{{ extension_schema_fields('location_lookup.json#/$defs/lookup_request', 'common/location') }}

### 응답 { #response }

{{ extension_schema_fields('location_lookup.json#/$defs/lookup_response', 'common/location') }}

## 예시 {: #examples }

다음 요청과 응답은 전송 방식에 중립적인 UCP 페이로드다.

### Downtown Store 영업 일정 { #downtown-store-schedule }

=== "요청"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=request -->
    ```json
    {
      "ids": ["loc_downtown", "downtown-store"]
    }
    ```

=== "응답"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=response -->
    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.common.location.lookup": [
            {"version": "{{ ucp_version }}"}
          ]
        }
      },
      "locations": [
        {
          "id": "loc_downtown",
          "inputs": [
            {"id": "loc_downtown"},
            {"id": "downtown-store"}
          ],
          "name": "Downtown Store",
          "address": {
            "street_address": "100 Broadway",
            "address_locality": "New York",
            "address_region": "NY",
            "address_country": "US",
            "postal_code": "10005"
          },
          "geo": {
            "latitude": 40.707,
            "longitude": -74.011
          },
          "amenities": {
            "dev.ucp.amenity.shopping.curbside_pickup": {
              "description": "Curbside pickup"
            },
            "dev.ucp.amenity.shopping.in_store_pickup": {
              "description": "In-store pickup"
            },
            "dev.ucp.amenity.parking": {
              "description": "On-site parking"
            }
          },
          "timezone": "America/New_York",
          "hours": [
            {"day": "monday", "opens": "09:00", "closes": "21:00"},
            {"day": "tuesday", "opens": "09:00", "closes": "12:00"},
            {"day": "tuesday", "opens": "13:00", "closes": "21:00"},
            {"day": "wednesday", "opens": "09:00", "closes": "21:00"},
            {"day": "thursday", "opens": "09:00", "closes": "21:00"},
            {"day": "friday", "opens": "09:00", "closes": "22:00"},
            {"day": "saturday", "opens": "10:00", "closes": "20:00"}
          ],
          "exception_hours": [
            {
              "title": "Thanksgiving",
              "valid_from": "2026-11-26",
              "valid_through": "2026-11-26"
            }
          ]
        }
      ]
    }
    ```

응답은 정식 식별자와 비즈니스가 지원하는 별칭을 하나의 위치에 대응시킨다.
화요일의 `hours` 항목 두 개는 분할 영업(중간 휴식)을 나타낸다. 일요일에는 `hours`
항목이 없으므로 그날 시작하는 정규 영업 구간이 없다는 뜻이다. `exception_hours`
항목은 `opens`와 `closes`를 생략했으므로 종일 휴업을 의미한다. 일정 표현 방식과
평가 규칙은 [영업시간](index.md#operating-hours)을 참고한다.

### 상품 재고로 결과 좁히기 { #item-availability-refinement }

=== "요청"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=request -->
    ```json
    {
      "ids": ["loc_downtown", "loc_uptown"],
      "filters": {
        "items": ["item_id_phone_15_pro"]
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=response -->
    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.common.location.lookup": [
            {"version": "{{ ucp_version }}"}
          ]
        }
      },
      "locations": [
        {
          "id": "loc_downtown",
          "inputs": [
            {"id": "loc_downtown"}
          ],
          "name": "Downtown Store"
        }
      ]
    }
    ```

두 식별자 모두 해석되지만, 비즈니스가 현재 `loc_uptown`에서 해당 상품을 제공할 수
없으므로 그 위치와 `inputs` 항목은 생략된다. 요청은 여전히 성공하며, 이처럼 결과
좁히기에서 일반적으로 제외되는 경우에는 메시지가 필요하지 않다
([상품 재고 필터](search.md#item-availability-filter) 참고).

### 부분 성공 { #partial-success }

=== "요청"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=request -->
    ```json
    {
      "ids": ["loc_downtown", "loc_invalid_id"]
    }
    ```

=== "응답"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=response -->
    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.common.location.lookup": [
            {"version": "{{ ucp_version }}"}
          ]
        }
      },
      "locations": [
        {
          "id": "loc_downtown",
          "inputs": [
            {"id": "loc_downtown"}
          ],
          "name": "Downtown Store"
        }
      ],
      "messages": [
        {
          "type": "info",
          "code": "not_found",
          "content": "Unable to find the location associated with loc_invalid_id."
        }
      ]
    }
    ```

요청은 해석된 위치들과 함께 성공한다. 비즈니스는 정보성 `not_found` 메시지로
해석되지 않은 식별자를 알려 줄 수 있다.

### 배치 한도 적용 { #batch-limit-applied }

다음 예시의 비즈니스는 조회 요청 하나당 서로 다른 식별자를 최대 두 개까지 받는다.

=== "요청"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=request -->
    ```json
    {
      "ids": ["loc_downtown", "loc_uptown", "loc_airport"]
    }
    ```

=== "응답"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=response -->
    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.common.location.lookup": [
            {"version": "{{ ucp_version }}"}
          ]
        }
      },
      "locations": [
        {
          "id": "loc_downtown",
          "inputs": [{"id": "loc_downtown"}],
          "name": "Downtown Store"
        },
        {
          "id": "loc_uptown",
          "inputs": [{"id": "loc_uptown"}],
          "name": "Uptown Store"
        }
      ],
      "messages": [
        {
          "type": "info",
          "code": "batch_limit_applied",
          "content": "Processed the first 2 of 3 distinct identifiers; 1 identifier was not processed."
        }
      ]
    }
    ```

비즈니스는 요청 순서대로 `loc_downtown`과 `loc_uptown`을 평가한다.
`loc_airport`는 평가되지 않았을 뿐 해석되지 않은 것이 아니며, 플랫폼은 이후 요청에서
이를 제출할 수 있다.

## 전송 바인딩 { #transport-bindings }

* [REST 바인딩](rest.md#post-locationslookup): `POST /locations/lookup`
* [MCP 바인딩](mcp.md#lookup_locations): `lookup_locations` 도구
