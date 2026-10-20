# 카탈로그 검색 기능(Capability)

- **기능 이름:** `dev.ucp.shopping.catalog.search`

비즈니스의 상품 카탈로그에서 상품을 검색합니다. 자유 텍스트 쿼리, 카테고리·가격 필터링, 페이지네이션을 지원합니다.

## 작업

| 작업              | 설명                                    |
| ----------------- | --------------------------------------- |
| **카탈로그 검색** | 주어진 입력과 필터로 상품을 검색합니다. |

### 요청

| 이름        | 타입                                                                                  | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | ------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| query       | string                                                                                | Optional  | Free-text search query.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)                       | Optional  | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                       | Optional  | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)               | Optional  | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| filters     | [Search Filters](/ucp-ko/draft/draft/specification/reference/#search-filters)         | Optional  | Filter criteria to narrow search results. All specified filters combine with AND logic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| pagination  | [Pagination Request](/ucp-ko/draft/draft/specification/reference/#pagination-request) | Optional  | Pagination parameters for requests.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

### 응답

| 이름       | 타입                                                                                                    | 필수 여부    | 설명                                                                                                                                                                                                   |
| ---------- | ------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ucp        | [UCP Catalog Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-catalog-schema) | **Required** | UCP metadata for catalog responses.                                                                                                                                                                    |
| products   | Array\[[Product](/ucp-ko/draft/draft/specification/reference/#product)\]                                | **Required** | Products matching the search criteria.                                                                                                                                                                 |
| pagination | [Pagination Response](/ucp-ko/draft/draft/specification/reference/#pagination-response)                 | Optional     | Pagination information in responses.                                                                                                                                                                   |
| actions    | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | Optional     | Outstanding extension-defined Actions for this catalog search response.                                                                                                                                |
| messages   | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | Optional     | Errors, warnings, or informational messages about the search results.                                                                                                                                  |
| policies   | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | Optional     | Policies (e.g., return/refund terms) that apply to the products in these search results. `applies_to` targets are relative to the response root; when absent or empty, refer to the URLs in `links[]`. |

## 검색 입력

유효한 검색 요청에는 `query` 문자열, 하나 이상의 `filters`, 확장이 정의한 입력 중 적어도 하나가 반드시 들어 있어야 합니다(MUST). `query`가 없는 요청은 둘러보기(browse) 작업으로 간주하며, 이때 비즈니스는 텍스트 관련도 순위를 매기지 않고 주어진 필터에 맞는 상품을 반환합니다. 확장은 시각적 유사도나 상품 참조 같은 입력을 추가로 정의할 수 있습니다(MAY).

구현체는 들어오는 요청에 인식할 수 있는 입력이 적어도 하나 있는지 반드시 검증해야 하며(MUST), 비어 있거나 유효하지 않은 요청은 적절한 에러와 함께 거부해야 합니다(SHOULD). 어떤 입력이 있어야 하고 내용이 어때야 하는지는 구현체가 직접 정해서 적용합니다. 예를 들어 `query`를 필수로 요구하거나, 빈 `query` 문자열을 거부하거나, 카테고리 둘러보기용으로 필터만 있는 요청을 허용할 수 있습니다.

## 검색 필터

검색 결과를 좁히는 필터 조건입니다. 표준 필터는 아래에 정의되어 있으며, 판매자는 `additionalProperties`로 사용자 정의 필터를 추가로 지원할 수 있습니다(MAY).

| 이름       | 타입                                                                      | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ---------- | ------------------------------------------------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| categories | Array[string]                                                             | Optional  | Filter by product categories (OR logic — matches products in any listed categories). Values match against the value field in product category entries. Valid values can be discovered from the categories field in search results, merchant documentation, or standard taxonomies that businesses may align with.                                                                                                                                                |
| price      | [Price Filter](/ucp-ko/draft/draft/specification/reference/#price-filter) | Optional  | Price range filter denominated in context.currency. When context.currency matches the presentment currency, businesses apply the filter directly. When it differs, businesses SHOULD convert filter values to the presentment currency before applying; if conversion is not supported, businesses MAY ignore the filter and SHOULD indicate this via a message. When context.currency is absent, filter denomination is ambiguous and businesses MAY ignore it. |

### 가격 필터

| 이름 | 타입                                                          | 필수 여부 | 설명                                   |
| ---- | ------------------------------------------------------------- | --------- | -------------------------------------- |
| min  | [Amount](/ucp-ko/draft/draft/specification/reference/#amount) | Optional  | Minimum price in ISO 4217 minor units. |
| max  | [Amount](/ucp-ko/draft/draft/specification/reference/#amount) | Optional  | Maximum price in ISO 4217 minor units. |

## 페이지네이션

목록 작업에서 쓰는 커서 기반 페이지네이션입니다. 커서는 불투명(opaque) 문자열입니다. 비즈니스는 커서를 상태 없는 키셋(keyset) 토큰으로 인코딩할 수 있습니다(**MAY**).

### 페이지 크기

`limit` 파라미터는 원하는 페이지 크기를 요청할 뿐, 결과 개수를 보장하지 않습니다. `limit`가 없으면 비즈니스는 반드시 기본 페이지 크기를 적용해야 합니다(**MUST**). 기본값은 10을 권장하지만(**RECOMMENDED**), 비즈니스가 다른 값을 정할 수도 있습니다(**MAY**).

비즈니스는 요청된 페이지 크기나 기본 페이지 크기보다 적은 결과를 반환할 수 있으며(**MAY**), 최대 페이지 크기를 적용하는 경우도 여기에 해당합니다. 플랫폼은 응답에 담긴 결과 개수가 두 값 중 어느 하나와 같다고 가정해서는 안 됩니다(**MUST NOT**).

### 페이지네이션 요청

Pagination parameters for requests.

| 이름   | 타입    | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                             |
| ------ | ------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| cursor | string  | Optional  | Opaque cursor from previous response.                                                                                                                                                                                                                                                                                                                                                            |
| limit  | integer | Optional  | Requested page size, not a guaranteed result count. When omitted, the Business MUST apply a default page size. A default of 10 is RECOMMENDED, but the Business MAY choose another value. The Business MAY return fewer results than the requested or default page size, including when enforcing its maximum page size. A Platform MUST NOT assume that the response count equals either value. |

### 페이지네이션 응답

Pagination information in responses.

| 이름          | 타입    | 필수 여부    | 설명                                                                                  |
| ------------- | ------- | ------------ | ------------------------------------------------------------------------------------- |
| cursor        | string  | Optional     | Cursor to fetch the next page of results. MUST be present when has_next_page is true. |
| has_next_page | boolean | **Required** | Whether more results are available.                                                   |
| total_count   | integer | Optional     | Total number of matching items, if available.                                         |

## 액션

검색 응답은 응답 전용 `actions` 맵을 사용합니다. 필수 항목인 `products` 배열은 비어 있을 수 있습니다. 관련 상품을 하나도 넣지 않을지, 일부만 넣을지, 전부 넣을지는 액션 유형 계약에 따라 비즈니스가 결정합니다. 상위 계약과 예시는 [카탈로그 — 액션](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/#actions)을, 공통 규칙은 [개요 — 액션](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#actions)을 참고하세요.

## 전송 방식 바인딩

- [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/rest/#post-catalogsearch): `POST /catalog/search`
- [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/mcp/#search_catalog): `search_catalog` 도구
