# 카탈로그 - REST 바인딩

이 문서는 [카탈로그 기능](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/index.md)의 HTTP/REST 바인딩을 정의합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`의 UCP 프로필에서 REST 전송 방식 지원 여부를 선언합니다.

```json
{
  "ucp": {
    "version": "2026-04-08",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "2026-04-08",
          "spec": "https://ucp.dev/2026-04-08/specification/overview",
          "transport": "rest",
          "schema": "https://ucp.dev/2026-04-08/services/shopping/rest.openapi.json",
          "endpoint": "https://business.example.com/ucp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.catalog.search": [{
        "version": "2026-04-08",
        "spec": "https://ucp.dev/2026-04-08/specification/catalog/search",
        "schema": "https://ucp.dev/2026-04-08/schemas/shopping/catalog_search.json"
      }],
      "dev.ucp.shopping.catalog.lookup": [{
        "version": "2026-04-08",
        "spec": "https://ucp.dev/2026-04-08/specification/catalog/lookup",
        "schema": "https://ucp.dev/2026-04-08/schemas/shopping/catalog_lookup.json"
      }]
    }
  }
}
```

## 엔드포인트

| 엔드포인트         | 메서드 | 기능                                                                                                                          | 설명                                         |
| ------------------ | ------ | ----------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| `/catalog/search`  | POST   | [검색](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/search/index.md)                 | 상품을 검색합니다.                           |
| `/catalog/lookup`  | POST   | [조회](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/lookup/index.md)                 | ID로 하나 이상의 상품을 조회합니다.          |
| `/catalog/product` | POST   | [조회](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/lookup/#get-product-get_product) | 식별자로 상품의 전체 상세 정보를 가져옵니다. |

### `POST /catalog/search`

[카탈로그 검색](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/search/index.md) 기능에 대응합니다.

**Inputs**

| 이름        | 타입                                                                                            | 필수 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | ----------------------------------------------------------------------------------------------- | ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| query       | string                                                                                          | No   | Free-text search query.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| context     | [Context](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#context)                       | No   | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals     | [Signals](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#signals)                       | No   | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| attribution | [Attribution](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#attribution)               | No   | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| filters     | [Search Filters](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#search-filters)         | No   | Filter criteria to narrow search results. All specified filters combine with AND logic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| pagination  | [Pagination Request](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#pagination-request) | No   | Cursor-based pagination for list operations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |

**Output**

| 이름       | 타입                                                                                              | 필수    | 설명                                                                  |
| ---------- | ------------------------------------------------------------------------------------------------- | ------- | --------------------------------------------------------------------- |
| ucp        | any                                                                                               | **Yes** | UCP metadata for catalog responses.                                   |
| products   | Array\[[Product](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#product)\]                | **Yes** | Products matching the search criteria.                                |
| pagination | [Pagination Response](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#pagination-response) | No      | Cursor-based pagination for list operations.                          |
| messages   | Array\[[Message](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#message)\]                | No      | Errors, warnings, or informational messages about the search results. |

#### 예시

```json
{
  "query": "blue running shoes",
  "context": {
    "address_country": "US",
    "address_region": "CA",
    "intent": "looking for comfortable everyday shoes"
  },
  "filters": {
    "categories": ["Footwear"],
    "price": {
      "max": 15000
    }
  },
  "pagination": {
    "limit": 20
  }
}
```

```json
{
  "ucp": {
    "version": "2026-04-08",
    "capabilities": {
      "dev.ucp.shopping.catalog.search": [
        {"version": "2026-04-08"}
      ]
    }
  },
  "products": [
    {
      "id": "prod_abc123",
      "handle": "blue-runner-pro",
      "title": "Blue Runner Pro",
      "description": {
        "plain": "Lightweight running shoes with responsive cushioning."
      },
      "url": "https://business.example.com/products/blue-runner-pro",
      "categories": [
        { "value": "187", "taxonomy": "google_product_category" },
        { "value": "aa-8-1", "taxonomy": "shopify" },
        { "value": "Footwear > Running", "taxonomy": "merchant" }
      ],
      "price_range": {
        "min": { "amount": 12000, "currency": "USD" },
        "max": { "amount": 12000, "currency": "USD" }
      },
      "media": [
        {
          "type": "image",
          "url": "https://cdn.example.com/products/blue-runner-pro.jpg",
          "alt_text": "Blue Runner Pro running shoes"
        }
      ],
      "options": [
        {
          "name": "Size",
          "values": [
            {"label": "8"},
            {"label": "9"},
            {"label": "10"},
            {"label": "11"},
            {"label": "12"}
          ]
        }
      ],
      "variants": [
        {
          "id": "prod_abc123_size10",
          "sku": "BRP-BLU-10",
          "title": "Size 10",
          "description": { "plain": "Size 10 variant" },
          "price": { "amount": 12000, "currency": "USD" },
          "availability": { "available": true },
          "options": [
            { "name": "Size", "label": "10" }
          ],
          "tags": ["running", "road", "neutral"],
          "seller": {
            "name": "Example Store",
            "links": [
              {
                "type": "refund_policy",
                "url": "https://business.example.com/refunds"
              }
            ]
          }
        }
      ],
      "rating": {
        "value": 4.5,
        "scale_max": 5,
        "count": 128
      },
      "metadata": {
        "collection": "Winter 2026",
        "technology": {
          "midsole": "React foam",
          "outsole": "Continental rubber"
        }
      }
    }
  ],
  "pagination": {
    "cursor": "eyJwYWdlIjoxfQ==",
    "has_next_page": true,
    "total_count": 47
  }
}
```

### `POST /catalog/lookup`

[카탈로그 조회](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/lookup/index.md) 기능에 대응합니다. 지원하는 식별자, 해석 동작, 클라이언트 대응 요건은 기능 문서를 참고하세요.

요청 본문에는 식별자 배열과, 배치 안의 모든 조회에 적용되는 선택적 컨텍스트가 담깁니다.

**Inputs**

| 이름        | 타입                                                                                    | 필수    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | --------------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ids         | Array[string]                                                                           | **Yes** | Identifiers to lookup. Implementations MUST support product ID and variant ID; MAY support secondary identifiers (SKU, handle, etc.).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| filters     | [Search Filters](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#search-filters) | No      | Filter criteria to narrow returned products and variants. All specified filters combine with AND logic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| context     | [Context](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#context)               | No      | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals     | [Signals](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#signals)               | No      | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| attribution | [Attribution](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#attribution)       | No      | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |

**Output**

| 이름     | 타입                                                                               | 필수    | 설명                                                                                                                                                |
| -------- | ---------------------------------------------------------------------------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp      | any                                                                                | **Yes** | UCP metadata for catalog responses.                                                                                                                 |
| products | Array[any]                                                                         | **Yes** | Products matching the requested identifiers. May contain fewer items if some identifiers not found, or more if identifiers match multiple products. |
| messages | Array\[[Message](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#message)\] | No      | Errors, warnings, or informational messages about the requested items.                                                                              |

#### 예시: 컨텍스트를 포함한 일괄 조회

```json
POST /catalog/lookup HTTP/1.1
Host: business.example.com
Content-Type: application/json

{
  "ids": ["prod_abc123", "prod_def456"],
  "context": {
    "address_country": "US",
    "language": "es"
  }
}
```

```json
{
  "ucp": {
    "version": "2026-04-08",
    "capabilities": {
      "dev.ucp.shopping.catalog.lookup": [
        {"version": "2026-04-08"}
      ]
    }
  },
  "products": [
    {
      "id": "prod_abc123",
      "title": "Blue Runner Pro",
      "description": {
        "plain": "Zapatillas ligeras con amortiguación reactiva."
      },
      "price_range": {
        "min": { "amount": 12000, "currency": "USD" },
        "max": { "amount": 12000, "currency": "USD" }
      },
      "variants": [
        {
          "id": "prod_abc123_size10",
          "sku": "BRP-BLU-10",
          "title": "Talla 10",
          "description": { "plain": "Variante talla 10" },
          "price": { "amount": 12000, "currency": "USD" },
          "availability": { "available": true },
          "inputs": [
            { "id": "prod_abc123", "match": "featured" }
          ]
        }
      ]
    },
    {
      "id": "prod_def456",
      "title": "Trail Blazer X",
      "description": {
        "plain": "Zapatillas de trail con tracción superior."
      },
      "price_range": {
        "min": { "amount": 15000, "currency": "USD" },
        "max": { "amount": 15000, "currency": "USD" }
      },
      "variants": [
        {
          "id": "prod_def456_size10",
          "sku": "TBX-GRN-10",
          "title": "Talla 10",
          "price": { "amount": 15000, "currency": "USD" },
          "availability": { "available": true },
          "inputs": [
            { "id": "prod_def456", "match": "featured" }
          ]
        }
      ]
    }
  ]
}
```

#### 예시: 부분 성공 (일부 식별자를 찾을 수 없음)

배치의 식별자 중 일부를 찾을 수 없으면, 응답의 `products` 배열에는 찾은 상품만 들어 있습니다. 응답에 찾지 못한 식별자를 알리는 정보성 메시지를 포함할 수 있습니다(MAY).

```json
{
  "ids": ["prod_abc123", "prod_invalid", "prod_def456"]
}
```

```json
{
  "ucp": {
    "version": "2026-04-08",
    "capabilities": {
      "dev.ucp.shopping.catalog.lookup": [
        {"version": "2026-04-08"}
      ]
    }
  },
  "products": [
    {
      "id": "prod_abc123",
      "title": "Blue Runner Pro",
      "price_range": {
        "min": { "amount": 12000, "currency": "USD" },
        "max": { "amount": 12000, "currency": "USD" }
      }
    },
    {
      "id": "prod_def456",
      "title": "Trail Blazer X",
      "price_range": {
        "min": { "amount": 15000, "currency": "USD" },
        "max": { "amount": 15000, "currency": "USD" }
      }
    }
  ],
  "messages": [
    {
      "type": "info",
      "code": "not_found",
      "content": "prod_invalid"
    }
  ]
}
```

### `POST /catalog/product`

[카탈로그 조회](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/lookup/#get-product-get_product) 기능에 대응합니다. 상품 상세 페이지 전체를 렌더링할 수 있도록 (배열이 아닌) 단일 `product` 객체를 반환합니다.

**Inputs**

| 이름        | 타입                                                                                               | 필수    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | -------------------------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id          | string                                                                                             | **Yes** | Product or variant identifier. Implementations MUST support product ID and variant ID.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| selected    | Array\[[Selected Option](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#selected-option)\] | No      | Partial or full option selections for interactive variant narrowing. When provided, response option values include availability signals (available, exists) relative to these selections.                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| preferences | Array[string]                                                                                      | No      | Option names in relaxation priority order. When no exact variant matches all selections, the server drops options from the end of this list first. E.g., ['Color', 'Size'] keeps Color and relaxes Size.                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| filters     | [Search Filters](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#search-filters)            | No      | Filter criteria to narrow returned variants. All specified filters combine with AND logic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| context     | [Context](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#context)                          | No      | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals     | [Signals](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#signals)                          | No      | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| attribution | [Attribution](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#attribution)                  | No      | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |

**Output**

This object MUST be one of the following types: [Catalog Lookup Get Product Response](/ucp-ko/2026-04-08/2026-04-08/specification/catalog/rest/#catalog-lookup-get-product-response), [Error Response](/ucp-ko/2026-04-08/2026-04-08/specification/catalog/rest/#error-response).

#### 예시: 옵션 선택 포함

사용자가 Color=Blue를 선택한 상황입니다. 응답의 옵션 값에는 재고 상태 신호가 포함되며, 선택과 일치하는 변형이 반환됩니다.

```json
POST /catalog/product HTTP/1.1
Host: business.example.com
Content-Type: application/json

{
  "id": "prod_abc123",
  "selected": [
    { "name": "Color", "label": "Blue" }
  ],
  "preferences": ["Color", "Size"],
  "context": {
    "address_country": "US"
  }
}
```

```json
{
  "ucp": {
    "version": "2026-04-08",
    "capabilities": {
      "dev.ucp.shopping.catalog.lookup": [
        {"version": "2026-04-08"}
      ]
    }
  },
  "product": {
    "id": "prod_abc123",
    "handle": "runner-pro",
    "title": "Runner Pro",
    "description": {
      "plain": "Lightweight running shoes with responsive cushioning."
    },
    "url": "https://business.example.com/products/runner-pro",
    "price_range": {
      "min": { "amount": 12000, "currency": "USD" },
      "max": { "amount": 15000, "currency": "USD" }
    },
    "media": [
      {
        "type": "image",
        "url": "https://cdn.example.com/products/runner-pro-blue.jpg",
        "alt_text": "Runner Pro in Blue"
      }
    ],
    "options": [
      {
        "name": "Color",
        "values": [
          {"label": "Blue", "available": true, "exists": true},
          {"label": "Red", "available": true, "exists": true},
          {"label": "Green", "available": false, "exists": true}
        ]
      },
      {
        "name": "Size",
        "values": [
          {"label": "8", "available": true, "exists": true},
          {"label": "9", "available": true, "exists": true},
          {"label": "10", "available": true, "exists": true},
          {"label": "11", "available": false, "exists": false},
          {"label": "12", "available": true, "exists": true}
        ]
      }
    ],
    "selected": [
      { "name": "Color", "label": "Blue" }
    ],
    "variants": [
      {
        "id": "prod_abc123_blu_10",
        "sku": "BRP-BLU-10",
        "title": "Blue, Size 10",
        "description": { "plain": "Blue, Size 10" },
        "price": { "amount": 12000, "currency": "USD" },
        "availability": { "available": true },
        "options": [
          { "name": "Color", "label": "Blue" },
          { "name": "Size", "label": "10" }
        ]
      },
      {
        "id": "prod_abc123_blu_12",
        "sku": "BRP-BLU-12",
        "title": "Blue, Size 12",
        "description": { "plain": "Blue, Size 12" },
        "price": { "amount": 15000, "currency": "USD" },
        "availability": { "available": true },
        "options": [
          { "name": "Color", "label": "Blue" },
          { "name": "Size", "label": "12" }
        ]
      }
    ],
    "rating": {
      "value": 4.5,
      "scale_max": 5,
      "count": 128
    }
  }
}
```

Green은 품절입니다(`available: false, exists: true`). Blue에는 사이즈 11이 존재하지 않습니다(`exists: false`). 반환된 변형은 Color=Blue 선택과 일치합니다.

#### 상품을 찾을 수 없음

식별자가 상품으로 해석되지 않으면, 서버는 `ucp.status: "error"`와 설명 메시지를 담아 HTTP 200을 반환합니다. 이는 전송 에러가 아니라 애플리케이션 처리 결과입니다. 핸들러가 실행되었고, 그 결과를 UCP 봉투(envelope)로 보고한 것입니다.

```json
{
  "ucp": {
    "version": "2026-04-08",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.catalog.lookup": [
        {"version": "2026-04-08"}
      ]
    }
  },
  "messages": [
    {
      "type": "error",
      "code": "not_found",
      "content": "Product not found: prod_invalid",
      "severity": "unrecoverable"
    }
  ]
}
```

(배치 요청에 부분 결과를 반환하는) `/catalog/lookup`과 달리 `/catalog/product`는 단일 리소스 작업입니다. 상품이 없으면 `unrecoverable` 심각도의 애플리케이션 에러이며, 에이전트는 같은 식별자로 재시도하지 않는 것이 좋습니다.

## 에러 처리

UCP는 전송 에러와 비즈니스 처리 결과를 분리하는 2계층 에러 모델을 사용합니다.

### 전송 에러

요청 처리를 가로막는 프로토콜 수준 문제에는 HTTP 상태 코드를 사용합니다.

| 상태 | 의미                                                     |
| ---- | -------------------------------------------------------- |
| 400  | Bad Request - 잘못된 형식의 JSON 또는 필수 파라미터 누락 |
| 401  | Unauthorized - 인증 정보가 없거나 유효하지 않음          |
| 429  | Too Many Requests - 요청 속도 제한                       |
| 500  | Internal Server Error                                    |

### 비즈니스 처리 결과

애플리케이션 수준의 처리 결과는 모두 UCP 봉투와 선택적 `messages` 배열을 담아 HTTP 200으로 반환합니다. 메시지 의미와 일반적인 시나리오는 [카탈로그 개요](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/#messages-and-error-handling)를 참고하세요.

#### 예시: 모든 상품을 찾을 수 없음

요청한 식별자를 하나도 찾지 못하면 `products` 배열은 비어 있습니다. 응답에 찾지 못한 식별자를 알리는 정보성 메시지를 포함할 수 있습니다(MAY).

```json
{
  "ucp": {
    "version": "2026-04-08",
    "capabilities": {
      "dev.ucp.shopping.catalog.lookup": [
        {"version": "2026-04-08"}
      ]
    }
  },
  "products": [],
  "messages": [
    {
      "type": "info",
      "code": "not_found",
      "content": "prod_invalid1"
    },
    {
      "type": "info",
      "code": "not_found",
      "content": "prod_invalid2"
    }
  ]
}
```

비즈니스 처리 결과에는 표준 HTTP 200 상태를 사용하고, 메시지는 응답 본문에 담습니다.

## 엔터티

### UCP 응답 카탈로그

UCP metadata for catalog responses.

| 이름             | 타입   | 필수    | 설명                                                                        |
| ---------------- | ------ | ------- | --------------------------------------------------------------------------- |
| version          | string | **Yes** | UCP version in YYYY-MM-DD format.                                           |
| status           | string | No      | Application-level status of the UCP operation. **Enum:** `success`, `error` |
| services         | object | No      | Service registry keyed by reverse-domain name.                              |
| capabilities     | object | No      | Capability registry keyed by reverse-domain name.                           |
| payment_handlers | object | No      | Payment handler registry keyed by reverse-domain name.                      |
| capabilities     | any    | No      |                                                                             |

### 상세 상품

A product in a get_product response, extended with effective selections and availability signals on option values.

| 이름     | 타입          | 필수 | 설명                                                                                                                                                                                                 |
| -------- | ------------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| selected | Array[object] | No   | Effective option selections that anchor the featured variant and availability signals. Required when the product has configurable options; may be empty or omitted for products with no option axes. |
| options  | Array[object] | No   | Product options with availability signals relative to the effective selections.                                                                                                                      |

### Get Product 응답

| 이름     | 타입          | 필수    | 설명                                                                                                       |
| -------- | ------------- | ------- | ---------------------------------------------------------------------------------------------------------- |
| ucp      | any           | **Yes** | UCP metadata for catalog responses.                                                                        |
| product  | object        | **Yes** | The requested product with full detail. Singular — this is a single-resource operation.                    |
| messages | Array[object] | No      | Warnings or informational messages about the product (e.g., price recently changed, limited availability). |

### 에러 응답

| 이름         | 타입                                                                               | 필수    | 설명                                                              |
| ------------ | ---------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------------- |
| ucp          | any                                                                                | **Yes** | UCP protocol metadata. Status MUST be 'error' for error response. |
| messages     | Array\[[Message](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#message)\] | **Yes** | Array of messages describing why the operation failed.            |
| continue_url | string                                                                             | No      | URL for buyer handoff or session recovery.                        |

## 적합성

적합한 REST 전송 방식 구현은 반드시 다음을 충족해야 합니다(**MUST**).

1. 비즈니스의 UCP 프로필에 선언된 각 카탈로그 기능에 대해, 해당 기능의 요건([검색](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/search/index.md), [조회](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/lookup/index.md))에 따라 엔드포인트를 구현합니다. 각 기능은 독립적으로 채택할 수 있습니다. 조회 기능을 선언했다면 `/catalog/lookup`과 `/catalog/product`를 반드시 모두 제공해야 합니다(MUST).
1. 유효한 `Price` 객체(금액 + 통화)가 포함된 상품을 반환합니다.
1. 기본 limit 10으로 커서 기반 페이지네이션을 지원합니다.
1. 조회 요청에는 HTTP 200을 반환합니다. 알 수 없는 식별자가 있으면 반환되는 상품 수가 줄어듭니다(정보성 `not_found` 메시지를 포함할 수 있습니다(MAY)).
1. 배치 크기 제한을 넘는 요청에는 `request_too_large` 에러와 함께 HTTP 400을 반환합니다.
