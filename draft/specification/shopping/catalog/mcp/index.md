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

| 이름        | 타입                                                                                  | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ----------- | ------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| query       | string                                                                                | 선택      | 자유 텍스트 검색어.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)                       | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                       | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)               | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| filters     | [Search Filters](/ucp-ko/draft/draft/specification/reference/#search-filters)         | 선택      | 검색 결과를 좁히는 필터 조건. 지정한 모든 필터는 AND 논리로 결합됩니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| pagination  | [Pagination Request](/ucp-ko/draft/draft/specification/reference/#pagination-request) | 선택      | 요청의 페이지네이션 매개변수.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |

### 검색 응답

| 이름       | 타입                                                                                                    | 필수 여부 | 설명                                                                                                                                                     |
| ---------- | ------------------------------------------------------------------------------------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp        | [UCP Catalog Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-catalog-schema) | **필수**  | 카탈로그 응답용 UCP 메타데이터.                                                                                                                          |
| products   | Array\[[Product](/ucp-ko/draft/draft/specification/reference/#product)\]                                | **필수**  | 검색 조건과 일치하는 상품.                                                                                                                               |
| pagination | [Pagination Response](/ucp-ko/draft/draft/specification/reference/#pagination-response)                 | 선택      | 응답의 페이지네이션 정보.                                                                                                                                |
| actions    | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | 선택      | 이 카탈로그 검색 응답에 대해 확장이 정의한 미해결 Action.                                                                                                |
| messages   | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | 선택      | 검색 결과에 관한 오류, 경고, 정보 메시지.                                                                                                                |
| policies   | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | 선택      | 이 검색 결과의 상품에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다. |

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

카탈로그 조회 요청 본문.

| 이름        | 타입                                                                          | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ----------- | ----------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ids         | Array[string]                                                                 | **필수**  | 조회할 식별자. 구현은 반드시 상품 ID와 변형 ID를 지원해야 하며(MUST), 보조 식별자(SKU, handle 등)를 지원할 수 있습니다(MAY).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| filters     | [Search Filters](/ucp-ko/draft/draft/specification/reference/#search-filters) | 선택      | 반환되는 상품과 변형을 좁히는 필터 조건. 지정한 모든 필터는 AND 논리로 결합됩니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)               | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)               | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)       | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                                                                                                                                                                                                                                                                                     |

### 조회 응답

| 이름     | 타입                                                                                                    | 필수 여부 | 설명                                                                                                                                                |
| -------- | ------------------------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp      | [UCP Catalog Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-catalog-schema) | **필수**  | 카탈로그 응답용 UCP 메타데이터.                                                                                                                     |
| products | Array\[[Product](/ucp-ko/draft/draft/specification/reference/#product)\]                                | **필수**  | 요청한 식별자와 일치하는 상품입니다. 일부 식별자를 찾지 못하면 항목이 더 적을 수 있고, 식별자가 여러 상품과 일치하면 더 많을 수 있습니다.           |
| actions  | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | 선택      | 이 카탈로그 조회 응답에 대해 확장이 정의한 미해결 Action.                                                                                           |
| messages | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | 선택      | 요청된 항목에 관한 오류, 경고, 정보 메시지.                                                                                                         |
| policies | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | 선택      | 이 응답의 상품에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다. |

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

단일 상품 조회 요청 본문입니다. selected와 preferences로 대화형 변형 좁히기를 지원합니다.

| 이름        | 타입                                                                                     | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ----------- | ---------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id          | string                                                                                   | **필수**  | 상품 또는 변형 식별자입니다. 구현체는 반드시 상품 ID와 변형 ID를 모두 지원해야 합니다(MUST).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| selected    | Array\[[Selected Option](/ucp-ko/draft/draft/specification/reference/#selected-option)\] | 선택      | 대화형으로 변형을 좁혀 가기 위한 부분 또는 전체 옵션 선택. 제공되면 응답의 옵션 값에 이 선택을 기준으로 한 가용성 신호(available, exists)가 포함됩니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| preferences | Array[string]                                                                            | 선택      | 완화 우선순위 순서로 나열한 옵션 이름. 모든 선택과 정확히 일치하는 변형이 없으면 서버는 이 목록의 끝에 있는 옵션부터 제외합니다. 예를 들어 ['Color', 'Size']는 Color를 유지하고 Size를 완화합니다.                                                                                                                                                                                                                                                                                                                                                                                                                    |
| filters     | [Search Filters](/ucp-ko/draft/draft/specification/reference/#search-filters)            | 선택      | 반환되는 변형을 좁히는 필터 조건. 지정한 모든 필터는 AND 논리로 결합됩니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)                          | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                          | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                  | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                                                                                                                                                                                                                                                                                     |

#### 상품 조회 응답

| 이름     | 타입                                                                                                    | 필수 여부 | 설명                                                                                                                                         |
| -------- | ------------------------------------------------------------------------------------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp      | [UCP Catalog Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-catalog-schema) | **필수**  | 카탈로그 응답용 UCP 메타데이터.                                                                                                              |
| product  | object                                                                                                  | **필수**  | 요청한 상품의 전체 세부 정보. 단수형이며, 단일 리소스 작업입니다.                                                                            |
| actions  | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | 선택      | 이 상품 응답에 대해 확장이 정의한 미해결 Action.                                                                                             |
| messages | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | 선택      | 상품에 대한 경고 또는 안내 메시지(예: 최근 가격 변경, 재고 부족).                                                                            |
| policies | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | 선택      | 이 상품에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다. |

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

get_product 응답의 상품으로, 유효 선택 항목과 옵션 값의 재고 신호가 추가됩니다.

| 이름             | 타입                                                                                     | 필수 여부 | 설명                                                                                                                                                    |
| ---------------- | ---------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id               | string                                                                                   | **필수**  | 이 상품을 고유하게 식별하는 전역 ID(GID).                                                                                                               |
| handle           | string                                                                                   | 선택      | SEO 친화적인 URL을 위한 URL 안전 슬러그(예: 'blue-runner-pro'). 안정적인 API 참조에는 id를 사용하세요.                                                  |
| title            | string                                                                                   | **필수**  | 상품 제목.                                                                                                                                              |
| description      | [Description](/ucp-ko/draft/draft/specification/reference/#description)                  | **필수**  | 하나 이상의 형식으로 작성된 상품 설명.                                                                                                                  |
| url              | string                                                                                   | 선택      | 표준 상품 페이지 URL.                                                                                                                                   |
| categories       | Array\[[Category](/ucp-ko/draft/draft/specification/reference/#category)\]               | 선택      | 선택적 분류 체계 식별자가 포함된 상품 카테고리.                                                                                                         |
| price_range      | [Price Range](/ucp-ko/draft/draft/specification/reference/#price-range)                  | **필수**  | 모든 변형에 걸친 가격 범위.                                                                                                                             |
| list_price_range | [Price Range](/ucp-ko/draft/draft/specification/reference/#price-range)                  | 선택      | 할인 전 정가 범위(취소선 표시용).                                                                                                                       |
| media            | Array\[[Media](/ucp-ko/draft/draft/specification/reference/#media)\]                     | 선택      | 상품 미디어(이미지, 동영상, 3D 모델). 첫 번째 항목은 목록에 표시되는 대표 미디어입니다.                                                                 |
| options          | Array[object]                                                                            | 선택      | 현재 유효한 선택을 기준으로 한 재고 신호가 포함된 상품 옵션. 상품 옵션(Size, Color 등).                                                                 |
| variants         | Array\[[Variant](/ucp-ko/draft/draft/specification/reference/#variant)\]                 | **필수**  | 이 상품의 구매 가능한 변형입니다. 첫 번째 항목은 목록에 표시되는 대표 변형입니다.                                                                       |
| rating           | [Rating](/ucp-ko/draft/draft/specification/reference/#rating)                            | 선택      | 상품 종합 평점.                                                                                                                                         |
| tags             | Array[string]                                                                            | 선택      | 분류와 검색에 쓰는 상품 태그.                                                                                                                           |
| metadata         | object                                                                                   | 선택      | 표준 상품 모델을 확장하는 비즈니스 정의 사용자 데이터.                                                                                                  |
| selected         | Array\[[Selected Option](/ucp-ko/draft/draft/specification/reference/#selected-option)\] | 선택      | 대표 변형과 가용성 신호의 기준이 되는 실제 옵션 선택. 상품에 구성 가능한 옵션이 있으면 필수이며, 옵션 축이 없는 상품에서는 비우거나 생략할 수 있습니다. |

### 상품 조회 응답

| 이름     | 타입                                                                                                    | 필수 여부 | 설명                                                                                                                                         |
| -------- | ------------------------------------------------------------------------------------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp      | [UCP Catalog Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-catalog-schema) | **필수**  | 카탈로그 응답용 UCP 메타데이터.                                                                                                              |
| product  | object                                                                                                  | **필수**  | 요청한 상품의 전체 세부 정보. 단수형이며, 단일 리소스 작업입니다.                                                                            |
| actions  | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | 선택      | 이 상품 응답에 대해 확장이 정의한 미해결 Action.                                                                                             |
| messages | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | 선택      | 상품에 대한 경고 또는 안내 메시지(예: 최근 가격 변경, 재고 부족).                                                                            |
| policies | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | 선택      | 이 상품에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다. |

### 에러 응답

| 이름         | 타입                                                                     | 필수 여부 | 설명                                                                               |
| ------------ | ------------------------------------------------------------------------ | --------- | ---------------------------------------------------------------------------------- |
| ucp          | UCP Error                                                                | **필수**  | UCP 프로토콜 메타데이터. 오류 응답에서는 status가 반드시 'error'여야 합니다(MUST). |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\] | **필수**  | 작업이 실패한 이유를 설명하는 메시지 배열.                                         |
| continue_url | string                                                                   | 선택      | 구매자 핸드오프 또는 세션 복구용 URL.                                              |

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
