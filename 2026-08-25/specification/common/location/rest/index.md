# 위치(Location) - REST 바인딩

이 문서는 [위치 기능(Location Capability)](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/location/index.md)의 HTTP/REST 바인딩을 정의합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필에서 Common 서비스와 위치 기능의 REST 전송 방식 지원 여부를 알립니다.

```json
{
  "ucp": {
    "version": "2026-08-25",
    "services": {
      "dev.ucp.common": [
        {
          "version": "2026-08-25",
          "spec": "https://ucp.dev/2026-08-25/specification/overview",
          "transport": "rest",
          "schema": "https://ucp.dev/2026-08-25/services/common/rest.openapi.json",
          "endpoint": "https://business.example.com/ucp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.common.location.search": [{
        "version": "2026-08-25",
        "spec": "https://ucp.dev/2026-08-25/specification/common/location/search",
        "schema": "https://ucp.dev/2026-08-25/schemas/common/location_search.json"
      }],
      "dev.ucp.common.location.lookup": [{
        "version": "2026-08-25",
        "spec": "https://ucp.dev/2026-08-25/specification/common/location/lookup",
        "schema": "https://ucp.dev/2026-08-25/schemas/common/location_lookup.json"
      }]
    },
    "payment_handlers": {}
  }
}
```

## 엔드포인트

| 엔드포인트          | 메서드 | 기능                                                                                                                  | 설명                                                                             |
| ------------------- | ------ | --------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `/locations/search` | POST   | [검색](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/location/search/index.md) | 물리적 위치를 검색합니다.                                                        |
| `/locations/lookup` | POST   | [조회](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/location/lookup/index.md) | 식별자로 위치를 조회합니다. 명시적인 공간 관계와 필터로 결과를 좁힐 수 있습니다. |

### `POST /locations/search`

[위치 검색](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/location/search/index.md) 기능에 대응합니다. 전체 예시는 [전송 방식과 무관한 검색 예시](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/location/search/#examples)를 참고하세요.

**Inputs**

| 이름       | 타입                                                                                            | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ---------- | ----------------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| query      | string                                                                                          | Optional  | Free-text search query for natural language location search (e.g., 'restaurants near me that deliver', 'hotels with pool').                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| context    | [Context](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#context)                       | Optional  | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals    | [Signals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#signals)                       | Optional  | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| distance   | [Location Distance](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#location-distance)   | Optional  | Optional explicit-center radius predicate. When present, it combines with `serves` and every supplied `filters` predicate using AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| serves     | [Location Serves](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#location-serves)       | Optional  | Optional authoritative service-target predicate. When present, it combines with `distance` and every supplied `filters` predicate using AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| filters    | [Location Filter](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#location-filter)       | Optional  | Filter criteria to narrow Location Search and Lookup results. All supplied filters combine with AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| pagination | [Pagination Request](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#pagination-request) | Optional  | Cursor-based pagination for list operations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |

**Output**

| 이름       | 타입                                                                                              | 필수 여부    | 설명                                                                  |
| ---------- | ------------------------------------------------------------------------------------------------- | ------------ | --------------------------------------------------------------------- |
| ucp        | any                                                                                               | **Required** | UCP metadata for location responses.                                  |
| locations  | Array\[[Location](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#location)\]              | **Required** | Locations matching the search criteria.                               |
| pagination | [Pagination Response](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#pagination-response) | Optional     | Cursor-based pagination for list operations.                          |
| messages   | Array\[[Message](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message)\]                | Optional     | Errors, warnings, or informational messages about the search results. |

#### 바인딩 엔벨로프 예시

```json
POST /locations/search HTTP/1.1
Host: business.example.com
Content-Type: application/json
Request-Id: 8ef9b0c2-78d1-4e4b-91c2-3e2ef0d3ab9f
UCP-Agent: profile="https://platform.example/profiles/v2026-01/agent.json"

{
  "query": "grocery store"
}
```

```json
HTTP/1.1 200 OK
Content-Type: application/json
Content-Digest: sha-256=:yG9a8bC7...:

{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": {
      "dev.ucp.common.location.search": [
        {"version": "2026-08-25"}
      ]
    }
  },
  "locations": [
    {
      "id": "loc_valley_grocers",
      "name": "Valley Grocers"
    }
  ]
}
```

### `POST /locations/lookup`

[위치 조회](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/location/lookup/index.md) 기능에 대응합니다. 전체 예시는 [전송 방식과 무관한 조회 예시](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/location/lookup/#examples)를 참고하세요.

**Inputs**

| 이름     | 타입                                                                                          | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| -------- | --------------------------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ids      | Array[string]                                                                                 | **Required** | Identifiers of the Locations to look up. The Business MUST support canonical `Location.id` values and MAY support secondary or alias identifiers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| distance | [Location Distance](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#location-distance) | Optional     | Optional explicit-center radius predicate applied after ID resolution. It combines with `serves` and every supplied `filters` predicate using AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| serves   | [Location Serves](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#location-serves)     | Optional     | Optional authoritative service-target predicate applied after ID resolution. It combines with `distance` and every supplied `filters` predicate using AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| filters  | [Location Filter](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#location-filter)     | Optional     | Filter criteria to narrow Location Search and Lookup results. All supplied filters combine with AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| context  | [Context](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#context)                     | Optional     | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals  | [Signals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#signals)                     | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |

**Output**

| 이름      | 타입                                                                                                           | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                           |
| --------- | -------------------------------------------------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp       | any                                                                                                            | **Required** | UCP metadata for location responses.                                                                                                                                                                                                                                                                                                                           |
| locations | Array\[ [Lookup Location](/ucp-ko/2026-08-25/2026-08-25/specification/common/location/rest/#lookup-location)\] | **Required** | Locations matching the requested identifiers and refinements. May contain fewer Locations if some identifiers do not resolve or their resolved Locations are filtered out, or more if one identifier resolves to multiple Locations. When multiple identifiers resolve to the same Location, one returned Location carries all corresponding `inputs` entries. |
| messages  | Array\[[Message](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message)\]                             | Optional     | Errors, warnings, or informational messages about the requested Locations, including `batch_limit_applied` when the Business processes only its configured maximum number of identifiers.                                                                                                                                                                      |

#### 바인딩 엔벨로프 예시

```json
POST /locations/lookup HTTP/1.1
Host: business.example.com
Content-Type: application/json
Request-Id: 2c9b0c2a-18d1-4e4b-91c2-3e2ef0d3ab9f
UCP-Agent: profile="https://platform.example/profiles/v2026-01/agent.json"

{
  "ids": ["loc_downtown"]
}
```

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": {
      "dev.ucp.common.location.lookup": [
        {"version": "2026-08-25"}
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

## 에러 처리

UCP는 전송 계층 에러와 비즈니스 결과를 분리하는 2계층 에러 모델을 사용합니다.

### 전송 에러

요청 처리를 가로막는 프로토콜 수준의 문제에는 HTTP 상태 코드를 사용합니다.

| 상태 | 의미                                                     |
| ---- | -------------------------------------------------------- |
| 400  | Bad Request - 잘못된 형식의 JSON 또는 필수 파라미터 누락 |
| 401  | Unauthorized - 인증 정보 누락 또는 유효하지 않은 인증    |
| 429  | Too Many Requests - 요청 빈도 제한 초과                  |
| 500  | Internal Server Error                                    |

### 비즈니스 결과

모든 애플리케이션 수준 결과는 UCP 엔벨로프와 선택적 `messages` 배열을 담아 HTTP 200으로 반환합니다. 메시지의 의미는 [위치 개요](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/location/#messages-and-error-handling)를 참고하세요.

## 엔티티

### UCP 응답 위치(엔벨로프)

UCP metadata for location responses.

| 이름             | 타입   | 필수 여부    | 설명                                                                                                                                           |
| ---------------- | ------ | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| version          | string | **Required** | Version identifier in YYYY-MM-DD format.                                                                                                       |
| map_order        | object | Optional     | Preferred key-traversal order for sibling registry fields inside the root `ucp` envelope (`services`, `capabilities`, and `payment_handlers`). |
| status           | string | Optional     | Application-level status of the UCP operation. **Enum:** `success`, `error`                                                                    |
| services         | object | Optional     | Service registry keyed by reverse-domain name.                                                                                                 |
| capabilities     | object | Optional     | Capability registry keyed by reverse-domain name.                                                                                              |
| payment_handlers | object | Optional     | Payment handler registry keyed by reverse-domain name.                                                                                         |
| capabilities     | any    | Optional     |                                                                                                                                                |

### 편의 시설 유형

| 이름                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | 타입 | 필수 여부 | 설명 |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- | --------- | ---- |
| Reverse-domain identifier used for collision-safe namespacing of capabilities, services, handlers, eligibility claims, and extension-contributed keys. Must contain at least two dot-separated segments (e.g., 'dev.ucp.shopping.checkout', 'com.example.loyalty_gold'). Segments after the first are domain- or identifier-derived: they may contain interior hyphens, may start with a digit, and may contain underscores (e.g., 'com.example-shop.checkout', 'com.2example.cart', 'dev.ucp.common.identity_linking'), but must not start or end with a hyphen. The first segment (the reversed top-level domain) is letters and digits, and may contain interior hyphens to support internationalized (punycode) top-level domains such as 'xn--p1ai'. |      |           |      |

**Pattern:** `^[a-z](?:[a-z0-9-]*[a-z0-9])?(?:\.[a-z0-9](?:[a-z0-9_-]*[a-z0-9_])?)+$`

### 편의 시설

Buyer-facing presentation metadata for one amenity identifier. The containing map key, not this metadata, defines amenity identity and filter matching.

| 이름        | 타입   | 필수 여부    | 설명                                                                                                                                                                                                                                                                           |
| ----------- | ------ | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| description | string | **Required** | Short, plain-text, buyer-facing label or phrase for the amenity, suitable for direct use in a compact list (e.g., 'Curbside pickup'). The Business SHOULD localize it for the request when possible. This content does not participate in amenity identity or filter matching. |

### 위치

| 이름            | 타입                                                                                             | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                           |
| --------------- | ------------------------------------------------------------------------------------------------ | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| id              | string                                                                                           | **Required** | Stable, opaque, Business-scoped Location identifier.                                                                                                                                                                                                                                                                                                                           |
| name            | string                                                                                           | **Required** | Buyer-facing, Business-owned display name.                                                                                                                                                                                                                                                                                                                                     |
| address         | [Postal Address](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#postal-address)          | Optional     | Physical address of the location.                                                                                                                                                                                                                                                                                                                                              |
| geo             | [Geo](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#geo)                                | Optional     | Geographic coordinates for the location.                                                                                                                                                                                                                                                                                                                                       |
| amenities       | object                                                                                           | Optional     | Static features, services, or capabilities of the Location, keyed by reverse-domain amenity identifier. Each value provides a buyer-facing description; the key alone defines amenity identity and filter matching.                                                                                                                                                            |
| hours           | Array\[[Daily Hour](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#daily-hour)\]         | Optional     | Regular weekly operating hours whose day and time values use this Location's canonical local civil-time frame. Multiple entries for the same day support split shifts. An omitted day has no regular interval beginning that day; an interval beginning on the preceding day can carry into it. Omission of the entire `hours` property means the regular schedule is unknown. |
| exception_hours | Array\[[Exception Hour](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#exception-hour)\] | Optional     | Date-specific operating-hour exceptions, including full closures, whose date and time values use this Location's canonical local civil-time frame.                                                                                                                                                                                                                             |
| timezone        | string                                                                                           | Optional     | The Business-owned IANA Time Zone Database identifier (e.g., 'America/New_York') defining this Location's canonical local civil-time frame for all returned schedule day, time, and date fields. The Business does not vary this canonical framing by the requesting Platform's or Buyer's timezone. Required when hours or exception_hours is present.                        |

### 조회 위치

Location with required correlation metadata for lookup responses.

| 이름            | 타입          | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                           |
| --------------- | ------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| id              | string        | **Required** | Stable, opaque, Business-scoped Location identifier.                                                                                                                                                                                                                                                                                                                           |
| name            | string        | **Required** | Buyer-facing, Business-owned display name.                                                                                                                                                                                                                                                                                                                                     |
| address         | object        | Optional     | Physical address of the location.                                                                                                                                                                                                                                                                                                                                              |
| geo             | object        | Optional     | Geographic coordinates for the location.                                                                                                                                                                                                                                                                                                                                       |
| amenities       | object        | Optional     | Static features, services, or capabilities of the Location, keyed by reverse-domain amenity identifier. Each value provides a buyer-facing description; the key alone defines amenity identity and filter matching.                                                                                                                                                            |
| hours           | Array[object] | Optional     | Regular weekly operating hours whose day and time values use this Location's canonical local civil-time frame. Multiple entries for the same day support split shifts. An omitted day has no regular interval beginning that day; an interval beginning on the preceding day can carry into it. Omission of the entire `hours` property means the regular schedule is unknown. |
| exception_hours | Array[object] | Optional     | Date-specific operating-hour exceptions, including full closures, whose date and time values use this Location's canonical local civil-time frame.                                                                                                                                                                                                                             |
| timezone        | string        | Optional     | The Business-owned IANA Time Zone Database identifier (e.g., 'America/New_York') defining this Location's canonical local civil-time frame for all returned schedule day, time, and date fields. The Business does not vary this canonical framing by the requesting Platform's or Buyer's timezone. Required when hours or exception_hours is present.                        |
| inputs          | Array[object] | **Required** | Which request identifiers resolved to this Location. Each entry preserves one identifier exactly as supplied in the request.                                                                                                                                                                                                                                                   |

### 위치 필터

| 이름      | 타입                                                                                         | 필수 여부 | 설명                                                                                                                                                                                                   |
| --------- | -------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| hours     | object                                                                                       | Optional  | Filter by operating hours, evaluated at the one supplied instant.                                                                                                                                      |
| amenities | Array\[[Amenity Type](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#amenity-type)\] | Optional  | Filter by amenity identifier. A Location matches only when its `amenities` map contains every supplied identifier as an exact key; descriptions and namespace prefixes do not participate in matching. |
| items     | Array[string]                                                                                | Optional  | Current item-availability filter. A candidate Location matches only when the Business can currently provide every referenced item at that Location; all references combine with AND.                   |

### 위치 거리

| 이름   | 타입                                                              | 필수 여부    | 설명                                                                                                                                                                                    |
| ------ | ----------------------------------------------------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| center | [Geo](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#geo) | **Required** | Explicit center of the radius. The Platform MUST supply it; the Business MUST NOT derive it from context, signals, an IP address, or `serves`.                                          |
| max    | number                                                            | **Required** | Inclusive maximum distance in RFC 7035 distance unit (meters). A Business unable to honor the supplied value MUST reject the request rather than clamp it or substitute another radius. |

### 위치 서비스 지역

| 이름    | 타입                                                              | 필수 여부 | 설명                                      |
| ------- | ----------------------------------------------------------------- | --------- | ----------------------------------------- |
| point   | [Geo](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#geo) | Optional  | WGS 84 coordinates of the service target. |
| address | any                                                               | Optional  | Coarse locality of the service target.    |

### 에러 응답

| 이름         | 타입                                                                               | 필수 여부    | 설명                                                              |
| ------------ | ---------------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------- |
| ucp          | any                                                                                | **Required** | UCP protocol metadata. Status MUST be 'error' for error response. |
| messages     | Array\[[Message](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message)\] | **Required** | Array of messages describing why the operation failed.            |
| continue_url | string                                                                             | Optional     | URL for buyer handoff or session recovery.                        |

## 적합성

적합한 REST 전송 방식 구현은 반드시 다음을 지켜야 합니다(**MUST**).

1. 비즈니스의 UCP 프로필에 선언된 각 위치 기능의 엔드포인트를 해당 기능의 요구 사항([검색](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/location/search/index.md), [조회](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/location/lookup/index.md))에 따라 구현합니다. 각 기능은 독립적으로 도입할 수 있습니다(**MAY**).
1. `distance`·`serves` 관계와 `filters.items` 재고 여부 조건은 플랫폼이 명시적으로 제공한 피연산자로만 평가합니다. `context`, `signals`, IP 주소에서 피연산자를 유추하지 않으며, 입력이 없을 때 서비스 가능 여부나 상품 재고 여부를 암묵적으로 확인하지 않습니다.
1. `distance`, `serves`, 그리고 제공된 모든 `filters` 조건을 논리곱(AND)으로 적용합니다.
1. 공통 페이지네이션 규약에 따라 검색에서 커서 기반 페이지네이션을 지원합니다 ([페이지네이션](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/location/search/#pagination) 참고).
1. 조회 요청에는 HTTP 200을 반환합니다. 알 수 없는 식별자가 있으면 반환되는 위치가 줄어들거나 없을 수 있습니다(정보성 `not_found` 메시지를 포함할 수 있습니다(**MAY**)).
1. 조회 요청이 비즈니스의 일괄 처리 최대치를 넘으면 HTTP 200을 반환하고, 요청 순서상 앞에서부터 최대치만큼의 고유 식별자를 처리하며, 정보성 `batch_limit_applied` 메시지를 포함합니다.
