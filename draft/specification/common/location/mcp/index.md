# Location - MCP 바인딩

이 문서는 [Location 기능(Capability)](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/index.md)의 Model Context Protocol(MCP) 바인딩을 설명합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`의 UCP 프로필에서 Common 서비스와 Location 기능의 MCP 전송 방식 지원 여부를 알립니다.

```json
{
  "ucp": {
    "version": "draft",
    "services": {
      "dev.ucp.common": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/overview",
          "transport": "mcp",
          "schema": "https://ucp.dev/draft/services/common/mcp.openrpc.json",
          "endpoint": "https://business.example.com/ucp/mcp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.common.location.search": [{
        "version": "draft",
        "spec": "https://ucp.dev/draft/specification/common/location/search",
        "schema": "https://ucp.dev/draft/schemas/common/location_search.json"
      }],
      "dev.ucp.common.location.lookup": [{
        "version": "draft",
        "spec": "https://ucp.dev/draft/specification/common/location/lookup",
        "schema": "https://ucp.dev/draft/schemas/common/location_lookup.json"
      }]
    },
    "payment_handlers": {}
  }
}
```

### 요청 메타데이터

MCP를 사용하는 플랫폼은 모든 요청에 반드시 `meta["ucp-agent"].profile`이 포함된 `meta` 객체를 넣어야 합니다(**MUST**). 이 필드는 버전 호환성 확인과 기능 협상에 쓰이는 플랫폼의 UCP 프로필을 식별합니다. 프로토콜 메타데이터는 `location`의 도메인 요청과 분리해 `meta`에 둡니다.

## 도구

| 도구               | 기능                                                                                                          | 설명                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `search_locations` | [Search](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/search/index.md) | 텍스트, 명시적 공간 관계, 필터로 Location을 검색합니다.                                          |
| `lookup_locations` | [Lookup](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/lookup/index.md) | 식별자로 Location을 일괄 조회하며, 선택적으로 명시적 공간 관계와 필터로 범위를 좁힐 수 있습니다. |

### `search_locations`

[Location Search](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/search/index.md) 기능에 대응합니다. [전송 방식에 중립적인 Search 전체 예시](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/search/#examples)를 참고하세요.

#### 요청 인자

Request body for location search. The `distance` and `serves` relations and every supplied `filters` predicate combine with AND; `query` does not relax them.

| 이름       | 타입                                                                                  | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ---------- | ------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| query      | string                                                                                | Optional  | Free-text search query for natural language location search (e.g., 'restaurants near me that deliver', 'hotels with pool').                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| context    | [Context](/ucp-ko/draft/draft/specification/reference/#context)                       | Optional  | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals    | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                       | Optional  | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| distance   | [Location Distance](/ucp-ko/draft/draft/specification/reference/#location-distance)   | Optional  | Optional explicit-center radius predicate. When present, it combines with `serves` and every supplied `filters` predicate using AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| serves     | [Location Serves](/ucp-ko/draft/draft/specification/reference/#location-serves)       | Optional  | Optional authoritative service-target predicate. When present, it combines with `distance` and every supplied `filters` predicate using AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| filters    | [Location Filter](/ucp-ko/draft/draft/specification/reference/#location-filter)       | Optional  | Filter criteria to narrow Location Search and Lookup results. All supplied filters combine with AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| pagination | [Pagination Request](/ucp-ko/draft/draft/specification/reference/#pagination-request) | Optional  | Pagination parameters for requests.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

#### 응답 스키마

| 이름       | 타입                                                                                                      | 필수 여부    | 설명                                                                  |
| ---------- | --------------------------------------------------------------------------------------------------------- | ------------ | --------------------------------------------------------------------- |
| ucp        | [UCP Location Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-location-schema) | **Required** | UCP metadata for location responses.                                  |
| locations  | Array\[[Location](/ucp-ko/draft/draft/specification/reference/#location)\]                                | **Required** | Locations matching the search criteria.                               |
| pagination | [Pagination Response](/ucp-ko/draft/draft/specification/reference/#pagination-response)                   | Optional     | Pagination information in responses.                                  |
| messages   | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                  | Optional     | Errors, warnings, or informational messages about the search results. |

#### 바인딩 엔벌로프 예시

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_locations",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/agent.json"
        }
      },
      "location": {
        "query": "grocery store"
      }
    }
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "draft",
        "capabilities": {
          "dev.ucp.common.location.search": [
            {"version": "draft"}
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
  }
}
```

### `lookup_locations`

[Location Lookup](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/lookup/index.md) 기능에 대응합니다. [전송 방식에 중립적인 Lookup 전체 예시](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/lookup/#examples)를 참고하세요.

#### 요청 인자

Request body for batch location lookup. The Business resolves and deduplicates `ids` before applying `distance`, `serves`, and every supplied `filters` predicate; all structured predicates combine with AND.

| 이름     | 타입                                                                                | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| -------- | ----------------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ids      | Array[string]                                                                       | **Required** | Identifiers of the Locations to look up. The Business MUST support canonical `Location.id` values and MAY support secondary or alias identifiers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| distance | [Location Distance](/ucp-ko/draft/draft/specification/reference/#location-distance) | Optional     | Optional explicit-center radius predicate applied after ID resolution. It combines with `serves` and every supplied `filters` predicate using AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| serves   | [Location Serves](/ucp-ko/draft/draft/specification/reference/#location-serves)     | Optional     | Optional authoritative service-target predicate applied after ID resolution. It combines with `distance` and every supplied `filters` predicate using AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| filters  | [Location Filter](/ucp-ko/draft/draft/specification/reference/#location-filter)     | Optional     | Filter criteria to narrow Location Search and Lookup results. All supplied filters combine with AND.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| context  | [Context](/ucp-ko/draft/draft/specification/reference/#context)                     | Optional     | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals  | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                     | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |

#### 응답 스키마

| 이름      | 타입                                                                                                      | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                           |
| --------- | --------------------------------------------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp       | [UCP Location Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-location-schema) | **Required** | UCP metadata for location responses.                                                                                                                                                                                                                                                                                                                           |
| locations | Array\[[Location](/ucp-ko/draft/draft/specification/reference/#location)\]                                | **Required** | Locations matching the requested identifiers and refinements. May contain fewer Locations if some identifiers do not resolve or their resolved Locations are filtered out, or more if one identifier resolves to multiple Locations. When multiple identifiers resolve to the same Location, one returned Location carries all corresponding `inputs` entries. |
| messages  | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                  | Optional     | Errors, warnings, or informational messages about the requested Locations, including `batch_limit_applied` when the Business processes only its configured maximum number of identifiers.                                                                                                                                                                      |

#### 바인딩 엔벌로프 예시

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "lookup_locations",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/agent.json"
        }
      },
      "location": {
        "ids": ["loc_downtown"]
      }
    }
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "structuredContent": {
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
  }
}
```

## 오류 처리

UCP는 전송 수준 오류와 비즈니스 결과를 구분하는 2계층 오류 모델을 사용합니다.

### 전송 오류

요청 처리를 막는 전송 수준 실패(인증, 요청 한도 제한, 잘못된 파라미터)는 JSON-RPC `error`로 반환합니다. 자세한 내용은 [핵심 명세](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#error-codes)를 참고하세요.

### 비즈니스 결과

모든 애플리케이션 수준 결과는 UCP 엔벌로프와 선택적 `messages` 배열을 담은 성공 JSON-RPC result로 반환합니다. 메시지의 의미는 [Location 개요](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/#messages-and-error-handling)를 참고하세요.

## 엔티티

### Amenity Type

| 이름                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | 타입 | 필수 여부 | 설명 |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- | --------- | ---- |
| Reverse-domain identifier used for collision-safe namespacing of capabilities, services, handlers, eligibility claims, and extension-contributed keys. Must contain at least two dot-separated segments (e.g., 'dev.ucp.shopping.checkout', 'com.example.loyalty_gold'). Segments after the first are domain- or identifier-derived: they may contain interior hyphens, may start with a digit, and may contain underscores (e.g., 'com.example-shop.checkout', 'com.2example.cart', 'dev.ucp.common.identity_linking'), but must not start or end with a hyphen. The first segment (the reversed top-level domain) is letters and digits, and may contain interior hyphens to support internationalized (punycode) top-level domains such as 'xn--p1ai'. |      |           |      |

**Pattern:** `^[a-z](?:[a-z0-9-]*[a-z0-9])?(?:\.[a-z0-9](?:[a-z0-9_-]*[a-z0-9_])?)+$`

### Amenity

Buyer-facing presentation metadata for one amenity identifier. The containing map key, not this metadata, defines amenity identity and filter matching.

| 이름        | 타입   | 필수 여부    | 설명                                                                                                                                                                                                                                                                           |
| ----------- | ------ | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| description | string | **Required** | Short, plain-text, buyer-facing label or phrase for the amenity, suitable for direct use in a compact list (e.g., 'Curbside pickup'). The Business SHOULD localize it for the request when possible. This content does not participate in amenity identity or filter matching. |

### Lookup Location

Location with required correlation metadata for lookup responses.

| 이름            | 타입                                                                                   | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                           |
| --------------- | -------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| id              | string                                                                                 | **Required** | Stable, opaque, Business-scoped Location identifier.                                                                                                                                                                                                                                                                                                                           |
| name            | string                                                                                 | **Required** | Buyer-facing, Business-owned display name.                                                                                                                                                                                                                                                                                                                                     |
| address         | [Postal Address](/ucp-ko/draft/draft/specification/reference/#postal-address)          | Optional     | Physical address of the location.                                                                                                                                                                                                                                                                                                                                              |
| geo             | [Geo](/ucp-ko/draft/draft/specification/reference/#geo)                                | Optional     | Geographic coordinates for the location.                                                                                                                                                                                                                                                                                                                                       |
| amenities       | object                                                                                 | Optional     | Static features, services, or capabilities of the Location, keyed by reverse-domain amenity identifier. Each value provides a buyer-facing description; the key alone defines amenity identity and filter matching.                                                                                                                                                            |
| hours           | Array\[[Daily Hour](/ucp-ko/draft/draft/specification/reference/#daily-hour)\]         | Optional     | Regular weekly operating hours whose day and time values use this Location's canonical local civil-time frame. Multiple entries for the same day support split shifts. An omitted day has no regular interval beginning that day; an interval beginning on the preceding day can carry into it. Omission of the entire `hours` property means the regular schedule is unknown. |
| exception_hours | Array\[[Exception Hour](/ucp-ko/draft/draft/specification/reference/#exception-hour)\] | Optional     | Date-specific operating-hour exceptions, including full closures, whose date and time values use this Location's canonical local civil-time frame.                                                                                                                                                                                                                             |
| timezone        | string                                                                                 | Optional     | The Business-owned IANA Time Zone Database identifier (e.g., 'America/New_York') defining this Location's canonical local civil-time frame for all returned schedule day, time, and date fields. The Business does not vary this canonical framing by the requesting Platform's or Buyer's timezone. Required when hours or exception_hours is present.                        |
| inputs          | Array[object]                                                                          | **Required** | Which request identifiers resolved to this Location. Each entry preserves one identifier exactly as supplied in the request.                                                                                                                                                                                                                                                   |

### Location

| 이름            | 타입                                                                                   | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                           |
| --------------- | -------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| id              | string                                                                                 | **Required** | Stable, opaque, Business-scoped Location identifier.                                                                                                                                                                                                                                                                                                                           |
| name            | string                                                                                 | **Required** | Buyer-facing, Business-owned display name.                                                                                                                                                                                                                                                                                                                                     |
| address         | [Postal Address](/ucp-ko/draft/draft/specification/reference/#postal-address)          | Optional     | Physical address of the location.                                                                                                                                                                                                                                                                                                                                              |
| geo             | [Geo](/ucp-ko/draft/draft/specification/reference/#geo)                                | Optional     | Geographic coordinates for the location.                                                                                                                                                                                                                                                                                                                                       |
| amenities       | object                                                                                 | Optional     | Static features, services, or capabilities of the Location, keyed by reverse-domain amenity identifier. Each value provides a buyer-facing description; the key alone defines amenity identity and filter matching.                                                                                                                                                            |
| hours           | Array\[[Daily Hour](/ucp-ko/draft/draft/specification/reference/#daily-hour)\]         | Optional     | Regular weekly operating hours whose day and time values use this Location's canonical local civil-time frame. Multiple entries for the same day support split shifts. An omitted day has no regular interval beginning that day; an interval beginning on the preceding day can carry into it. Omission of the entire `hours` property means the regular schedule is unknown. |
| exception_hours | Array\[[Exception Hour](/ucp-ko/draft/draft/specification/reference/#exception-hour)\] | Optional     | Date-specific operating-hour exceptions, including full closures, whose date and time values use this Location's canonical local civil-time frame.                                                                                                                                                                                                                             |
| timezone        | string                                                                                 | Optional     | The Business-owned IANA Time Zone Database identifier (e.g., 'America/New_York') defining this Location's canonical local civil-time frame for all returned schedule day, time, and date fields. The Business does not vary this canonical framing by the requesting Platform's or Buyer's timezone. Required when hours or exception_hours is present.                        |

### Location Filter

| 이름      | 타입                                                                               | 필수 여부 | 설명                                                                                                                                                                                                   |
| --------- | ---------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| hours     | object                                                                             | Optional  | Filter by operating hours, evaluated at the one supplied instant.                                                                                                                                      |
| amenities | Array\[[Amenity Type](/ucp-ko/draft/draft/specification/reference/#amenity-type)\] | Optional  | Filter by amenity identifier. A Location matches only when its `amenities` map contains every supplied identifier as an exact key; descriptions and namespace prefixes do not participate in matching. |
| items     | Array[string]                                                                      | Optional  | Current item-availability filter. A candidate Location matches only when the Business can currently provide every referenced item at that Location; all references combine with AND.                   |

### Location Distance

| 이름   | 타입                                                    | 필수 여부    | 설명                                                                                                                                                                                    |
| ------ | ------------------------------------------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| center | [Geo](/ucp-ko/draft/draft/specification/reference/#geo) | **Required** | Explicit center of the radius. The Platform MUST supply it; the Business MUST NOT derive it from context, signals, an IP address, or `serves`.                                          |
| max    | number                                                  | **Required** | Inclusive maximum distance in RFC 7035 distance unit (meters). A Business unable to honor the supplied value MUST reject the request rather than clamp it or substitute another radius. |

### Location Serves

| 이름    | 타입                                                              | 필수 여부 | 설명                                      |
| ------- | ----------------------------------------------------------------- | --------- | ----------------------------------------- |
| point   | [Geo](/ucp-ko/draft/draft/specification/reference/#geo)           | Optional  | WGS 84 coordinates of the service target. |
| address | [Locality](/ucp-ko/draft/draft/specification/reference/#locality) | Optional  | Coarse locality of the service target.    |

### Error Response

| 이름         | 타입                                                                     | 필수 여부    | 설명                                                              |
| ------------ | ------------------------------------------------------------------------ | ------------ | ----------------------------------------------------------------- |
| ucp          | UCP Error                                                                | **Required** | UCP protocol metadata. Status MUST be 'error' for error response. |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\] | **Required** | Array of messages describing why the operation failed.            |
| continue_url | string                                                                   | Optional     | URL for buyer handoff or session recovery.                        |

## 적합성

적합한 MCP 전송 방식 구현은 반드시 다음을 지켜야 합니다(**MUST**).

1. JSON-RPC 2.0 프로토콜을 올바르게 구현합니다.
1. 비즈니스의 UCP 프로필에 알린 각 Location 기능의 도구를 해당 기능의 요구 사항([Search](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/search/index.md), [Lookup](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/lookup/index.md))에 따라 구현합니다. 각 기능은 독립적으로 채택할 수 있습니다(**MAY**).
1. `distance`, `serves` 관계와 `filters.items` 재고 조건은 플랫폼이 명시적으로 제공한 피연산자로만 평가합니다. `context`, `signals`, IP 주소에서 피연산자를 유도해서는 안 되며, 입력이 없을 때 서비스 가능 여부나 상품 재고 여부를 암묵적으로 확인해서도 안 됩니다.
1. `distance`, `serves`, 그리고 제공된 모든 `filters` 조건을 논리곱(AND)으로 적용합니다.
1. 공통 페이지네이션 규약에 따라 Search에서 커서 기반 페이지네이션을 지원합니다([페이지네이션](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/location/search/#pagination) 참고).
1. Lookup 요청에는 성공 JSON-RPC result를 반환합니다. 알 수 없는 식별자가 있으면 반환되는 Location이 줄거나 없을 뿐입니다(`messages` 배열에 정보성 `not_found` 메시지를 포함할 수 있습니다(**MAY**)).
1. Lookup 요청이 비즈니스의 배치 최대치를 초과하면 성공 JSON-RPC result를 반환하고, 요청 순서대로 고유 식별자를 최대치만큼 앞에서부터 처리하며, 정보성 `batch_limit_applied` 메시지를 포함합니다.
1. 도구 입력을 UCP 스키마로 검증합니다.
