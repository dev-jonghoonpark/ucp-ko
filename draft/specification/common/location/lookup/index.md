# 위치 조회 기능

- **기능(Capability) 이름:** `dev.ucp.common.location.lookup`

식별자를 실제 물리적 위치(Location)로 해석합니다. 여러 위치의 전체 상세 정보를 한 번에 조회할 수도 있고, 위치 하나만 조회할 수도 있습니다(위치 전용 상세 페이지에 유용합니다). 필요하면 검색(Search)과 같은 명시적 공간 관계와 필터로 결과를 좁힐 수도 있습니다([결과 좁히기](#refinement) 참고).

## 작업

| 작업          | 설명                                          |
| ------------- | --------------------------------------------- |
| **위치 조회** | 식별자로 위치를 하나 또는 여러 개 조회합니다. |

## 지원하는 식별자

`ids` 파라미터에는 식별자 배열을 넣습니다. 비즈니스는 위치의 안정적인 정식(canonical) 식별자인 `Location.id`로 조회하는 기능을 반드시 지원해야 하며(**MUST**), 보조 식별자나 별칭 식별자를 추가로 지원할 수 있습니다(**MAY**).

비즈니스는 요청에 들어 있는 중복 식별자를 반드시 제거해야 합니다(**MUST**). 식별자 하나가 여러 위치와 일치하면 비즈니스는 일치하는 위치를 모두 반환하되, 결과 집합의 크기를 제한할 수 있습니다(**MAY**). 여러 식별자가 같은 위치로 해석되면 비즈니스는 그 위치를 반드시 한 번만 반환해야 합니다(**MUST**).

### 플랫폼 측 대응 관계 파악

응답의 순서는 보장되지 않습니다. 대신 각 위치에는 요청한 식별자 중 어떤 것이 그 위치로 해석되었는지 알려 주는 `inputs` 배열이 들어 있습니다. 모든 항목에는 조회 요청에서 그대로 복사한 필수 `id`가 있습니다.

요청의 여러 식별자가 같은 위치로 해석될 수도 있습니다. 이때 해당 위치의 `inputs` 배열에는 해석된 식별자마다 항목이 하나씩 들어갑니다. 비즈니스는 `inputs` 항목이 없는 위치를 조회 응답에 포함해서는 안 됩니다(**MUST NOT**).

### 배치 크기

비즈니스는 요청당 식별자를 10개 이상 받을 수 있어야 하며(**SHOULD**), 최대 배치 크기를 강제할 수 있습니다(**MAY**). 최대값은 중복 식별자를 제거한 뒤에 적용합니다.

요청이 최대값을 넘으면 비즈니스는 요청 순서대로 앞에서부터 서로 다른 식별자를 최대값만큼 처리하고, 그 앞부분에 대한 성공 응답을 반드시 반환해야 합니다(**MUST**). 이때 `code: "batch_limit_applied"`인 정보성 메시지를 반드시 포함해야 하며 (**MUST**), 메시지 내용에 적용된 최대값과 처리하지 않은 식별자 수를 반드시 밝혀야 합니다(**MUST**). 비즈니스가 생략된 뒷부분을 평가하거나 그 식별자를 해석되지 않은 것으로 보고해서는 안 됩니다(**MUST NOT**). 플랫폼은 생략된 뒷부분을 해석되지 않은 것으로 받아들여서는 안 되며(**MUST NOT**), 이후 요청에서 다시 제출할 수 있습니다(**MAY**).

### 결과 좁히기

비즈니스는 먼저 식별자를 해석하고 반복된 값을 제거합니다. 그다음 선택 사항인 최상위 `distance`, `serves` 관계와 `filters` 조건으로 해석된 집합을 좁힙니다. 주어진 조건은 모두 AND로 결합됩니다. 즉, 해석된 위치는 주어진 관계와 필터를 모두 만족할 때만 반환됩니다. 관계와 조건의 스키마와 의미는 검색과 같습니다. [공간 관계](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/search/#spatial-relations), [검색 필터](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/search/#search-filters), [상품 재고 필터](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/search/#item-availability-filter)를 참고하세요.

예를 들어 플랫폼이 `["loc_downtown", "loc_uptown"]`을 영업시간 필터 `{"open_at": "2026-05-18T17:00:00Z"}`와 함께 요청하면 다음과 같이 처리됩니다.

1. 비즈니스는 먼저 두 식별자를 각각의 위치로 해석합니다.
1. 비즈니스는 해석된 위치마다 주어진 시각을 평가합니다.
1. 그 시각에 `loc_uptown`이 영업하지 않으면 비즈니스는 이를 제외하고 `loc_downtown`만 반환합니다.

해석되지 않은 식별자에는 대응하는 반환 위치나 `inputs` 항목이 없습니다. 식별자가 해석된 위치가 주어진 조건 중 하나라도 만족하지 못하면(`filters.items` 식별자를 알 수 없거나, 해당 위치에서 제공할 수 없거나, 평가할 수 없는 경우 포함) 그 위치와 대응하는 `inputs` 항목은 생략됩니다. 이 경우에도 요청은 성공합니다. 비즈니스는 해석되지 않은 위치 식별자에 정보성 `not_found` 메시지를 첨부할 수 있지만(**MAY**), 결과 좁히기에서 제외된 경우에는 식별자별 설명이 보장되지 않습니다.

### 요청

Request body for batch location lookup. The Business resolves and deduplicates `ids` before applying `distance`, `serves`, and every supplied `filters` predicate; all structured predicates combine with AND.

| 이름     | 타입                                                                                | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| -------- | ----------------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ids      | Array[string]                                                                       | **Required** | Identifiers of the Locations to look up. The Business MUST support canonical `Location.id` values and MAY support secondary or alias identifiers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| distance | [Location Distance](/ucp-ko/draft/draft/specification/reference/#location-distance) | Optional     | Optional explicit-center radius predicate applied after ID resolution. It combines with `serves` and every supplied `filters` predicate using AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| serves   | [Location Serves](/ucp-ko/draft/draft/specification/reference/#location-serves)     | Optional     | Optional authoritative service-target predicate applied after ID resolution. It combines with `distance` and every supplied `filters` predicate using AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| filters  | [Location Filter](/ucp-ko/draft/draft/specification/reference/#location-filter)     | Optional     | Filter criteria to narrow Location Search and Lookup results. All supplied filters combine with AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| context  | [Context](/ucp-ko/draft/draft/specification/reference/#context)                     | Optional     | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals  | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                     | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |

### 응답

| 이름      | 타입                                                                                                      | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                           |
| --------- | --------------------------------------------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp       | [UCP Location Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-location-schema) | **Required** | UCP metadata for location responses.                                                                                                                                                                                                                                                                                                                           |
| locations | Array\[[Location](/ucp-ko/draft/draft/specification/reference/#location)\]                                | **Required** | Locations matching the requested identifiers and refinements. May contain fewer Locations if some identifiers do not resolve or their resolved Locations are filtered out, or more if one identifier resolves to multiple Locations. When multiple identifiers resolve to the same Location, one returned Location carries all corresponding `inputs` entries. |
| messages  | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                  | Optional     | Errors, warnings, or informational messages about the requested Locations, including `batch_limit_applied` when the Business processes only its configured maximum number of identifiers.                                                                                                                                                                      |

## 예시

다음 요청과 응답은 전송 방식에 중립적인 UCP 페이로드입니다.

### Downtown Store 영업 일정

```json
{
  "ids": ["loc_downtown", "downtown-store"]
}
```

```json
{
  "ucp": {
    "version": "draft",
    "capabilities": {
      "dev.ucp.common.location.lookup": [
        {"version": "draft"}
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

응답에서는 정식 식별자와 비즈니스가 지원하는 별칭이 하나의 위치에 대응됩니다. 화요일의 `hours` 항목 두 개는 분할 영업(중간 휴식)을 나타냅니다. 일요일에는 `hours` 항목이 없으므로 그날 시작하는 정규 영업 구간이 없다는 뜻입니다. `exception_hours` 항목은 `opens`와 `closes`를 생략했으므로 종일 휴업을 뜻합니다. 일정 표현 방식과 평가 규칙은 [영업시간](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/#operating-hours)을 참고하세요.

### 상품 재고로 결과 좁히기

```json
{
  "ids": ["loc_downtown", "loc_uptown"],
  "filters": {
    "items": ["item_id_phone_15_pro"]
  }
}
```

```json
{
  "ucp": {
    "version": "draft",
    "capabilities": {
      "dev.ucp.common.location.lookup": [
        {"version": "draft"}
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

두 식별자 모두 해석되지만, 비즈니스가 현재 `loc_uptown`에서 해당 상품을 제공할 수 없으므로 그 위치와 `inputs` 항목은 생략됩니다. 요청은 그래도 성공하며, 이처럼 결과 좁히기에서 흔히 제외되는 경우에는 메시지가 필요하지 않습니다 ([상품 재고 필터](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/search/#item-availability-filter) 참고).

### 부분 성공

```json
{
  "ids": ["loc_downtown", "loc_invalid_id"]
}
```

```json
{
  "ucp": {
    "version": "draft",
    "capabilities": {
      "dev.ucp.common.location.lookup": [
        {"version": "draft"}
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

요청은 성공하고 해석된 위치를 반환합니다. 비즈니스는 정보성 `not_found` 메시지로 해석되지 않은 식별자를 알려 줄 수 있습니다.

### 배치 한도 적용

다음 예시의 비즈니스는 조회 요청 하나당 서로 다른 식별자를 최대 두 개까지 받습니다.

```json
{
  "ids": ["loc_downtown", "loc_uptown", "loc_airport"]
}
```

```json
{
  "ucp": {
    "version": "draft",
    "capabilities": {
      "dev.ucp.common.location.lookup": [
        {"version": "draft"}
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

비즈니스는 요청 순서대로 `loc_downtown`과 `loc_uptown`을 평가합니다. `loc_airport`는 평가되지 않았을 뿐 해석되지 않은 것이 아닙니다. 플랫폼은 이후 요청에서 이 식별자를 제출할 수 있습니다.

## 전송 바인딩

- [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/rest/#post-locationslookup): `POST /locations/lookup`
- [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/mcp/#lookup_locations): `lookup_locations` 도구
