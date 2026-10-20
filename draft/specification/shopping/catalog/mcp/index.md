# 카탈로그 - MCP 바인딩

이 문서는 [카탈로그 기능](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/index.md)의 Model Context Protocol(MCP) 바인딩을 설명합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필로 MCP 전송 방식 지원 여부를 알립니다.

```json
{
  "ucp": {
    "version": "draft",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/overview",
          "transport": "mcp",
          "schema": "https://ucp.dev/draft/services/shopping/mcp.openrpc.json",
          "endpoint": "https://business.example.com/ucp/mcp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.catalog.search": [{
        "version": "draft",
        "spec": "https://ucp.dev/draft/specification/shopping/catalog/search",
        "schema": "https://ucp.dev/draft/schemas/shopping/catalog_search.json"
      }],
      "dev.ucp.shopping.catalog.lookup": [{
        "version": "draft",
        "spec": "https://ucp.dev/draft/specification/shopping/catalog/lookup",
        "schema": "https://ucp.dev/draft/schemas/shopping/catalog_lookup.json"
      }]
    },
    "payment_handlers": {}
  }
}
```

### 요청 메타데이터

MCP 클라이언트는 모든 요청에 프로토콜 메타데이터를 담은 `meta` 객체를 반드시 포함해야 합니다(**MUST**).

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_catalog",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/shopping-agent.json"
        }
      },
      "catalog": {
        "query": "blue running shoes",
        "context": {
          "address_country": "US",
          "intent": "looking for comfortable everyday shoes"
        }
      }
    }
  }
}
```

버전 호환성 확인과 기능 협상에 쓰이므로 모든 요청에 `meta["ucp-agent"]` 필드가 \*\*필수(required)\*\*입니다.

## 도구

| 도구             | 기능                                                                                                                         | 설명                                                       |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `search_catalog` | [검색](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/search/index.md)                 | 상품을 검색합니다.                                         |
| `lookup_catalog` | [조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/index.md)                 | 식별자로 하나 이상의 상품 또는 변형(variant)을 조회합니다. |
| `get_product`    | [조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/#get-product-get_product) | 식별자로 상품의 전체 상세 정보를 가져옵니다.               |

### `search_catalog`

[카탈로그 검색](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/search/index.md) 기능에 대응합니다.

#### 검색 요청

| 이름        | 타입                                                                                  | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | ------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| query       | string                                                                                | Optional  | Free-text search query.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)                       | Optional  | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                       | Optional  | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)               | Optional  | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| filters     | [Search Filters](/ucp-ko/draft/draft/specification/reference/#search-filters)         | Optional  | Filter criteria to narrow search results. All specified filters combine with AND logic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| pagination  | [Pagination Request](/ucp-ko/draft/draft/specification/reference/#pagination-request) | Optional  | Pagination parameters for requests.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

### 검색 응답

| 이름       | 타입                                                                                                    | 필수 여부    | 설명                                                                                                                                                                                                   |
| ---------- | ------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ucp        | [UCP Catalog Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-catalog-schema) | **Required** | UCP metadata for catalog responses.                                                                                                                                                                    |
| products   | Array\[[Product](/ucp-ko/draft/draft/specification/reference/#product)\]                                | **Required** | Products matching the search criteria.                                                                                                                                                                 |
| pagination | [Pagination Response](/ucp-ko/draft/draft/specification/reference/#pagination-response)                 | Optional     | Pagination information in responses.                                                                                                                                                                   |
| actions    | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | Optional     | Outstanding extension-defined Actions for this catalog search response.                                                                                                                                |
| messages   | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | Optional     | Errors, warnings, or informational messages about the search results.                                                                                                                                  |
| policies   | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | Optional     | Policies (e.g., return/refund terms) that apply to the products in these search results. `applies_to` targets are relative to the response root; when absent or empty, refer to the URLs in `links[]`. |

#### 검색 예시

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_catalog",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/shopping-agent.json"
        }
      },
      "catalog": {
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
          "dev.ucp.shopping.catalog.search": [
            {"version": "draft"}
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
  }
}
```

### `lookup_catalog`

[카탈로그 조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/index.md) 기능에 대응합니다. 지원되는 식별자, 해석 동작, 클라이언트 대응 관계 요구 사항은 해당 기능 문서를 참고하세요.

`catalog.ids` 매개변수는 식별자 배열과 선택적 컨텍스트를 받습니다.

#### 조회 요청

Request body for catalog lookup.

| 이름        | 타입                                                                          | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | ----------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ids         | Array[string]                                                                 | **Required** | Identifiers to lookup. Implementations MUST support product ID and variant ID; MAY support secondary identifiers (SKU, handle, etc.).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| filters     | [Search Filters](/ucp-ko/draft/draft/specification/reference/#search-filters) | Optional     | Filter criteria to narrow returned products and variants. All specified filters combine with AND logic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)               | Optional     | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)               | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)       | Optional     | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |

### 조회 응답

| 이름     | 타입                                                                                                    | 필수 여부    | 설명                                                                                                                                                                                            |
| -------- | ------------------------------------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp      | [UCP Catalog Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-catalog-schema) | **Required** | UCP metadata for catalog responses.                                                                                                                                                             |
| products | Array\[[Product](/ucp-ko/draft/draft/specification/reference/#product)\]                                | **Required** | Products matching the requested identifiers. May contain fewer items if some identifiers not found, or more if identifiers match multiple products.                                             |
| actions  | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | Optional     | Outstanding extension-defined Actions for this catalog lookup response.                                                                                                                         |
| messages | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | Optional     | Errors, warnings, or informational messages about the requested items.                                                                                                                          |
| policies | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | Optional     | Policies (e.g., return/refund terms) that apply to the products in this response. `applies_to` targets are relative to the response root; when absent or empty, refer to the URLs in `links[]`. |

#### 조회 예시

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "lookup_catalog",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/shopping-agent.json"
        }
      },
      "catalog": {
        "ids": ["prod_abc123", "var_xyz789"],
        "context": {
          "address_country": "US"
        }
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
          "dev.ucp.shopping.catalog.lookup": [
            {"version": "draft"}
          ]
        }
      },
      "products": [
        {
          "id": "prod_abc123",
          "title": "Blue Runner Pro",
          "description": {
            "plain": "Lightweight running shoes with responsive cushioning."
          },
          "price_range": {
            "min": { "amount": 12000, "currency": "USD" },
            "max": { "amount": 12000, "currency": "USD" }
          },
          "variants": [
            {
              "id": "prod_abc123_size10",
              "sku": "BRP-BLU-10",
              "title": "Size 10",
              "description": { "plain": "Size 10 variant" },
              "price": { "amount": 12000, "currency": "USD" },
              "availability": { "available": true },
              "inputs": [
                { "id": "prod_abc123", "match": "featured" }
              ],
              "tags": ["running", "road", "neutral"],
              "seller": {
                "name": "Example Store",
                "links": [
                  {
                    "type": "refund_policy",
                    "url": "https://business.example.com/policies/refunds"
                  }
                ]
              }
            }
          ],
          "metadata": {
            "collection": "Winter 2026",
            "technology": {
              "midsole": "React foam",
              "outsole": "Continental rubber"
            }
          }
        },
        {
          "id": "prod_def456",
          "title": "Trail Master X",
          "description": {
            "plain": "Rugged trail running shoes with aggressive tread."
          },
          "price_range": {
            "min": { "amount": 15000, "currency": "USD" },
            "max": { "amount": 15000, "currency": "USD" }
          },
          "variants": [
            {
              "id": "var_xyz789",
              "sku": "TMX-GRN-11",
              "title": "Size 11 - Green",
              "description": { "plain": "Size 11 Green variant" },
              "price": { "amount": 15000, "currency": "USD" },
              "availability": { "available": true },
              "inputs": [
                { "id": "var_xyz789", "match": "exact" }
              ],
              "tags": ["trail", "waterproof"],
              "seller": {
                "name": "Example Store"
              }
            }
          ]
        }
      ]
    }
  }
}
```

#### 부분 성공

일부 식별자를 찾지 못하면 응답에는 찾은 상품만 담깁니다. 어떤 식별자를 찾지 못했는지 알려 주는 정보성 메시지를 응답에 포함할 수 있습니다(MAY).

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "draft",
        "capabilities": {
          "dev.ucp.shopping.catalog.lookup": [
            {"version": "draft"}
          ]
        }
      },
      "products": [
        {
          "id": "prod_abc123",
          "title": "Blue Runner Pro",
          "description": {
            "plain": "A comfortable everyday running shoe."
          },
          "price_range": {
            "min": { "amount": 12000, "currency": "USD" },
            "max": { "amount": 12000, "currency": "USD" }
          },
          "variants": [ ... ]
        }
      ],
      "messages": [
        {
          "type": "info",
          "code": "not_found",
          "content": "prod_notfound1"
        },
        {
          "type": "info",
          "code": "not_found",
          "content": "prod_notfound2"
        }
      ]
    }
  }
}
```

### `get_product`

[카탈로그 조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/#get-product-get_product) 기능에 대응합니다. 대화형 옵션 선택을 지원하는 전체 상품 상세 정보를 담아 `product` 객체 하나를 반환합니다.

#### 상품 조회 요청

Request body for single-product retrieval. Supports interactive variant narrowing via selected and preferences.

| 이름        | 타입                                                                                     | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | ---------------------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id          | string                                                                                   | **Required** | Product or variant identifier. Implementations MUST support product ID and variant ID.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| selected    | Array\[[Selected Option](/ucp-ko/draft/draft/specification/reference/#selected-option)\] | Optional     | Partial or full option selections for interactive variant narrowing. When provided, response option values include availability signals (available, exists) relative to these selections.                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| preferences | Array[string]                                                                            | Optional     | Option names in relaxation priority order. When no exact variant matches all selections, the server drops options from the end of this list first. E.g., ['Color', 'Size'] keeps Color and relaxes Size.                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| filters     | [Search Filters](/ucp-ko/draft/draft/specification/reference/#search-filters)            | Optional     | Filter criteria to narrow returned variants. All specified filters combine with AND logic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)                          | Optional     | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                          | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                  | Optional     | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |

#### 상품 조회 응답

| 이름     | 타입                                                                                                    | 필수 여부    | 설명                                                                                                                                                                           |
| -------- | ------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ucp      | [UCP Catalog Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-catalog-schema) | **Required** | UCP metadata for catalog responses.                                                                                                                                            |
| product  | object                                                                                                  | **Required** | The requested product with full detail. Singular — this is a single-resource operation.                                                                                        |
| actions  | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | Optional     | Outstanding extension-defined Actions for this product response.                                                                                                               |
| messages | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | Optional     | Warnings or informational messages about the product (e.g., price recently changed, limited availability).                                                                     |
| policies | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | Optional     | Policies (e.g., return/refund terms) that apply to this product. `applies_to` targets are relative to the response root; when absent or empty, refer to the URLs in `links[]`. |

#### 상품 조회 예시

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "get_product",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/shopping-agent.json"
        }
      },
      "catalog": {
        "id": "prod_abc123",
        "selected": [
          { "name": "Color", "label": "Blue" }
        ],
        "preferences": ["Color", "Size"],
        "context": {
          "address_country": "US"
        }
      }
    }
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "draft",
        "capabilities": {
          "dev.ucp.shopping.catalog.lookup": [
            {"version": "draft"}
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
            "alt_text": "Runner Pro in Blue",
            "width": 1600,
            "height": 1600,
            "sources": [
              {
                "url": "https://cdn.example.com/products/runner-pro-blue.avif",
                "mime_type": "image/avif",
                "format": "avif",
                "width": 1600,
                "height": 1600
              },
              {
                "url": "https://cdn.example.com/products/runner-pro-blue.jpg",
                "mime_type": "image/jpeg",
                "format": "jpg",
                "width": 1600,
                "height": 1600
              }
            ]
          },
          {
            "type": "video",
            "url": "https://cdn.example.com/products/runner-pro-demo-720p.mp4",
            "name": "Runner Pro flex demo",
            "alt_text": "Runner Pro shoe flexing to show sole cushioning",
            "duration": 24,
            "preview": {
              "url": "https://cdn.example.com/products/runner-pro-demo-poster.jpg",
              "width": 1280,
              "height": 720
            },
            "sources": [
              {
                "url": "https://cdn.example.com/products/runner-pro-demo-720p.mp4",
                "mime_type": "video/mp4",
                "format": "mp4",
                "width": 1280,
                "height": 720,
                "filesize": 4200000
              },
              {
                "url": "https://cdn.example.com/products/runner-pro-demo-1080p.mp4",
                "mime_type": "video/mp4",
                "format": "mp4",
                "width": 1920,
                "height": 1080,
                "filesize": 8480000
              },
              {
                "url": "https://cdn.example.com/products/runner-pro-demo.m3u8",
                "mime_type": "application/vnd.apple.mpegurl",
                "format": "m3u8"
              }
            ]
          },
          {
            "type": "model_3d",
            "url": "https://cdn.example.com/products/runner-pro.glb",
            "alt_text": "Runner Pro 3D model",
            "preview": {
              "url": "https://cdn.example.com/products/runner-pro-3d-thumb.jpg"
            },
            "sources": [
              {
                "url": "https://cdn.example.com/products/runner-pro.glb",
                "mime_type": "model/gltf-binary",
                "format": "glb",
                "filesize": 4200000
              },
              {
                "url": "https://cdn.example.com/products/runner-pro.usdz",
                "mime_type": "model/vnd.usdz+zip",
                "format": "usdz",
                "filesize": 5100000
              }
            ]
          },
          {
            "type": "external_video",
            "url": "https://videos.example.com/embed/123",
            "name": "Runner Pro trail review",
            "alt_text": "Reviewer running a trail in the Runner Pro",
            "duration": 212,
            "preview": {
              "url": "https://cdn.example.com/products/runner-pro-review-poster.jpg",
              "width": 1280,
              "height": 720
            }
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
  }
}
```

#### 상품을 찾을 수 없음

식별자가 어떤 상품으로도 해석되지 않으면, 서버는 `ucp.status: "error"`와 설명 메시지를 담은 성공 JSON-RPC 결과를 반환합니다. 전송 오류가 아니라 애플리케이션 수준의 결과입니다.

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "draft",
        "status": "error",
        "capabilities": {
          "dev.ucp.shopping.catalog.lookup": [
            {"version": "draft"}
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
  }
}
```

## 오류 처리

UCP는 전송 오류와 비즈니스 결과를 나누는 2계층 오류 모델을 사용합니다.

### 전송 오류

요청 처리를 가로막는 전송 수준의 실패(인증, 속도 제한, 서비스 불가)는 JSON-RPC `error`로 반환됩니다. 전체 오류 코드 레지스트리와 JSON-RPC 오류 코드 매핑은 [핵심 명세](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#error-codes)를 참고하세요.

### 비즈니스 결과

모든 애플리케이션 수준의 결과는 UCP 엔벌로프와 선택적 `messages` 배열을 담은 성공 JSON-RPC 결과로 반환됩니다. 메시지 의미와 일반적인 시나리오는 [카탈로그 개요](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/#messages-and-error-handling)를 참고하세요.

#### 예시: 모든 상품을 찾을 수 없음

요청한 식별자를 하나도 해석하지 못하면 응답에는 빈 `products` 배열이 담깁니다. 어떤 식별자를 찾지 못했는지 알려 주는 정보성 메시지를 응답에 포함할 수 있습니다(MAY).

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "draft",
        "capabilities": {
          "dev.ucp.shopping.catalog.lookup": [
            {"version": "draft"}
          ]
        }
      },
      "products": [],
      "messages": [
        {
          "type": "info",
          "code": "not_found",
          "content": "prod_invalid"
        }
      ]
    }
  }
}
```

비즈니스 결과는 JSON-RPC `result` 필드를 사용하고, 메시지는 응답 페이로드에 담습니다. 성공과 실패가 섞인 결과를 처리하는 방법은 [부분 성공](#partial-success) 섹션을 참고하세요.

## 엔티티

### 상세 상품

A product in a get_product response, extended with effective selections and availability signals on option values.

| 이름             | 타입                                                                                     | 필수 여부    | 설명                                                                                                                                                                                                 |
| ---------------- | ---------------------------------------------------------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id               | string                                                                                   | **Required** | Global ID (GID) uniquely identifying this product.                                                                                                                                                   |
| handle           | string                                                                                   | Optional     | URL-safe slug for SEO-friendly URLs (e.g., 'blue-runner-pro'). Use id for stable API references.                                                                                                     |
| title            | string                                                                                   | **Required** | Product title.                                                                                                                                                                                       |
| description      | [Description](/ucp-ko/draft/draft/specification/reference/#description)                  | **Required** | Product description in one or more formats.                                                                                                                                                          |
| url              | string                                                                                   | Optional     | Canonical product page URL.                                                                                                                                                                          |
| categories       | Array\[[Category](/ucp-ko/draft/draft/specification/reference/#category)\]               | Optional     | Product categories with optional taxonomy identifiers.                                                                                                                                               |
| price_range      | [Price Range](/ucp-ko/draft/draft/specification/reference/#price-range)                  | **Required** | Price range across all variants.                                                                                                                                                                     |
| list_price_range | [Price Range](/ucp-ko/draft/draft/specification/reference/#price-range)                  | Optional     | List price range before discounts (for strikethrough display).                                                                                                                                       |
| media            | Array\[[Media](/ucp-ko/draft/draft/specification/reference/#media)\]                     | Optional     | Product media (images, videos, 3D models). First item is the featured media for listings.                                                                                                            |
| options          | Array[object]                                                                            | Optional     | Product options with availability signals relative to the effective selections. Product options (Size, Color, etc.).                                                                                 |
| variants         | Array\[[Variant](/ucp-ko/draft/draft/specification/reference/#variant)\]                 | **Required** | Purchasable variants of this product. First item is the featured variant for listings.                                                                                                               |
| rating           | [Rating](/ucp-ko/draft/draft/specification/reference/#rating)                            | Optional     | Aggregate product rating.                                                                                                                                                                            |
| tags             | Array[string]                                                                            | Optional     | Product tags for categorization and search.                                                                                                                                                          |
| metadata         | object                                                                                   | Optional     | Business-defined custom data extending the standard product model.                                                                                                                                   |
| selected         | Array\[[Selected Option](/ucp-ko/draft/draft/specification/reference/#selected-option)\] | Optional     | Effective option selections that anchor the featured variant and availability signals. Required when the product has configurable options; may be empty or omitted for products with no option axes. |

### 상품 조회 응답

| 이름     | 타입                                                                                                    | 필수 여부    | 설명                                                                                                                                                                           |
| -------- | ------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ucp      | [UCP Catalog Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-catalog-schema) | **Required** | UCP metadata for catalog responses.                                                                                                                                            |
| product  | object                                                                                                  | **Required** | The requested product with full detail. Singular — this is a single-resource operation.                                                                                        |
| actions  | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | Optional     | Outstanding extension-defined Actions for this product response.                                                                                                               |
| messages | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | Optional     | Warnings or informational messages about the product (e.g., price recently changed, limited availability).                                                                     |
| policies | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | Optional     | Policies (e.g., return/refund terms) that apply to this product. `applies_to` targets are relative to the response root; when absent or empty, refer to the URLs in `links[]`. |

### 에러 응답

| 이름         | 타입                                                                     | 필수 여부    | 설명                                                              |
| ------------ | ------------------------------------------------------------------------ | ------------ | ----------------------------------------------------------------- |
| ucp          | UCP Error                                                                | **Required** | UCP protocol metadata. Status MUST be 'error' for error response. |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\] | **Required** | Array of messages describing why the operation failed.            |
| continue_url | string                                                                   | Optional     | URL for buyer handoff or session recovery.                        |

## 적합성

적합한 MCP 전송 방식 구현은 반드시 다음 요건을 충족해야 합니다(**MUST**).

1. JSON-RPC 2.0 프로토콜을 올바르게 구현합니다.
1. 비즈니스의 UCP 프로필에 선언된 카탈로그 기능마다 해당 기능의 요구 사항([검색](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/search/index.md), [조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/index.md))에 따라 도구를 구현합니다. 각 기능은 따로따로 채택할 수 있습니다. 조회 기능을 선언했다면 `lookup_catalog`와 `get_product` 도구를 반드시 모두 제공해야 합니다(MUST).
1. 전송 문제에는 JSON-RPC 오류를, 비즈니스 결과에는 `messages` 배열을 사용합니다.
1. 조회 요청에는 성공 결과를 반환합니다. 알 수 없는 식별자가 있으면 그만큼 반환되는 상품 수가 줄어듭니다(정보성 `not_found` 메시지를 포함할 수 있습니다(MAY)).
1. 도구 입력을 UCP 스키마로 검증합니다.
1. 유효한 `Price` 객체(금액 + 통화)가 있는 상품을 반환합니다.
1. 공통 페이지네이션 규약에 따라 검색에 커서 기반 페이지네이션을 지원합니다([페이지네이션](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/search/#pagination) 참고).
1. 배치 크기 제한을 넘는 요청에는 `-32602`(Invalid params)를 반환합니다.
