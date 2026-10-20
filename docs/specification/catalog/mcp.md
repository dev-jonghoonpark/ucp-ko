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

# 카탈로그 - MCP 바인딩 { #catalog-mcp-binding }

이 문서는 [카탈로그 기능](index.md)의 Model Context Protocol(MCP) 바인딩을 설명합니다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필로 MCP 전송 방식 지원 여부를
알립니다.

```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview",
          "transport": "mcp",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/mcp.openrpc.json",
          "endpoint": "https://business.example.com/ucp/mcp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.catalog.search": [{
        "version": "{{ ucp_version }}",
        "spec": "https://ucp.dev/{{ ucp_version }}/specification/catalog/search",
        "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/catalog_search.json"
      }],
      "dev.ucp.shopping.catalog.lookup": [{
        "version": "{{ ucp_version }}",
        "spec": "https://ucp.dev/{{ ucp_version }}/specification/catalog/lookup",
        "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/catalog_lookup.json"
      }]
    }
  }
}
```

### 요청 메타데이터 { #request-metadata }

MCP 클라이언트는 모든 요청에 프로토콜 메타데이터를 담은 `meta` 객체를 반드시 포함해야
합니다(**MUST**).

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

버전 호환성 확인과 기능 협상에 쓰이므로 모든 요청에 `meta["ucp-agent"]` 필드가
**필수(required)**입니다.

## 도구 { #tools }

| 도구 | 기능 | 설명 |
| :--- | :--- | :--- |
| `search_catalog` | [검색](search.md) | 상품을 검색합니다. |
| `lookup_catalog` | [조회](lookup.md) | 식별자로 하나 이상의 상품 또는 변형(variant)을 조회합니다. |
| `get_product` | [조회](lookup.md#get-product-get_product) | 식별자로 상품의 전체 상세 정보를 가져옵니다. |

### `search_catalog` { #search_catalog }

[카탈로그 검색](search.md) 기능에 대응합니다.

#### 검색 요청 { #search-request }

{{ extension_schema_fields(
  'catalog_search.json#/$defs/search_request', 'catalog/mcp'
) }}

### 검색 응답 { #search-response }

{{ extension_schema_fields(
  'catalog_search.json#/$defs/search_response', 'catalog/mcp'
) }}

#### 검색 예시 { #search-example }

=== "요청"

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

=== "응답"

    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.shopping.catalog.search": [
                {"version": "{{ ucp_version }}"}
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

### `lookup_catalog` { #lookup_catalog }

[카탈로그 조회](lookup.md) 기능에 대응합니다. 지원되는 식별자, 해석 동작, 클라이언트
대응 관계 요구 사항은 해당 기능 문서를 참고하세요.

`catalog.ids` 매개변수는 식별자 배열과 선택적 컨텍스트를 받습니다.

#### 조회 요청 { #lookup-request }

{{ extension_schema_fields(
  'catalog_lookup.json#/$defs/lookup_request', 'catalog/mcp'
) }}

### 조회 응답 { #lookup-response }

{{ extension_schema_fields(
  'catalog_lookup.json#/$defs/lookup_response', 'catalog/mcp'
) }}

#### 조회 예시 { #lookup-example }

=== "요청"

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

=== "응답"

    ```json
    {
      "jsonrpc": "2.0",
      "id": 2,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.shopping.catalog.lookup": [
                {"version": "{{ ucp_version }}"}
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

#### 부분 성공 { #partial-success }

일부 식별자를 찾지 못하면 응답에는 찾은 상품만 담깁니다. 어떤 식별자를 찾지 못했는지
알려 주는 정보성 메시지를 응답에 포함할 수 있습니다(MAY).

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.shopping.catalog.lookup": [
            {"version": "{{ ucp_version }}"}
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
          },
          "variants": []
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

### `get_product` { #get_product }

[카탈로그 조회](lookup.md#get-product-get_product) 기능에 대응합니다. 대화형 옵션 선택을
지원하는 전체 상품 상세 정보를 담아 `product` 객체 하나를 반환합니다.

#### 상품 조회 요청 { #get-product-request }

{{ extension_schema_fields(
  'catalog_lookup.json#/$defs/get_product_request', 'catalog/mcp'
) }}

#### 상품 조회 응답 { #get-product-response }

{{ extension_schema_fields(
  'catalog_lookup.json#/$defs/get_product_response', 'catalog/mcp'
) }}

#### 상품 조회 예시 { #get-product-example }

=== "요청"

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

=== "응답"

    ```json
    {
      "jsonrpc": "2.0",
      "id": 3,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.shopping.catalog.lookup": [
                {"version": "{{ ucp_version }}"}
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
      }
    }
    ```

#### 상품을 찾을 수 없음 { #product-not-found }

식별자가 어떤 상품으로도 해석되지 않으면, 서버는 `ucp.status: "error"`와 설명 메시지를
담은 성공 JSON-RPC 결과를 반환합니다. 전송 오류가 아니라 애플리케이션 수준의 결과입니다.

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "{{ ucp_version }}",
        "status": "error",
        "capabilities": {
          "dev.ucp.shopping.catalog.lookup": [
            {"version": "{{ ucp_version }}"}
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

## 오류 처리 { #error-handling }

UCP는 전송 오류와 비즈니스 결과를 나누는 2계층 오류 모델을 사용합니다.

### 전송 오류 { #transport-errors }

요청 처리를 가로막는 전송 수준의 실패(인증, 속도 제한, 서비스 불가)는 JSON-RPC `error`로
반환됩니다. 전체 오류 코드 레지스트리와 JSON-RPC 오류 코드 매핑은
[핵심 명세](../overview.md#error-codes)를 참고하세요.

### 비즈니스 결과 { #business-outcomes }

모든 애플리케이션 수준의 결과는 UCP 엔벌로프와 선택적 `messages` 배열을 담은 성공
JSON-RPC 결과로 반환됩니다. 메시지 의미와 일반적인 시나리오는
[카탈로그 개요](index.md#messages-and-error-handling)를 참고하세요.

#### 예시: 모든 상품을 찾을 수 없음 { #example-all-products-not-found }

요청한 식별자를 하나도 해석하지 못하면 응답에는 빈 `products` 배열이 담깁니다. 어떤
식별자를 찾지 못했는지 알려 주는 정보성 메시지를 응답에 포함할 수 있습니다(MAY).

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.shopping.catalog.lookup": [
            {"version": "{{ ucp_version }}"}
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

비즈니스 결과는 JSON-RPC `result` 필드를 사용하고, 메시지는 응답 페이로드에 담습니다.
성공과 실패가 섞인 결과를 처리하는 방법은 [부분 성공](#partial-success) 섹션을 참고하세요.

## 엔티티 { #entities }

### 상세 상품 {: #detail-product }

{{ extension_schema_fields('catalog_lookup.json#/$defs/detail_product', 'catalog/mcp') }}

### 상품 조회 응답 {: #catalog-lookup-get-product-response }

{{ extension_schema_fields('catalog_lookup.json#/$defs/get_product_response', 'catalog/mcp') }}

### 에러 응답 {: #error-response }

{{ schema_fields('types/error_response', 'catalog/mcp') }}

## 적합성 { #conformance }

적합한 MCP 전송 방식 구현은 반드시 다음 요건을 충족해야 합니다(**MUST**).

1. JSON-RPC 2.0 프로토콜을 올바르게 구현합니다.
2. 비즈니스의 UCP 프로필에 선언된 카탈로그 기능마다 해당 기능의 요구 사항([검색](search.md), [조회](lookup.md))에 따라 도구를 구현합니다. 각 기능은 따로따로 채택할 수 있습니다. 조회 기능을 선언했다면 `lookup_catalog`와 `get_product` 도구를 반드시 모두 제공해야 합니다(MUST).
3. 전송 문제에는 JSON-RPC 오류를, 비즈니스 결과에는 `messages` 배열을 사용합니다.
4. 조회 요청에는 성공 결과를 반환합니다. 알 수 없는 식별자가 있으면 그만큼 반환되는 상품 수가 줄어듭니다(정보성 `not_found` 메시지를 포함할 수 있습니다(MAY)).
5. 도구 입력을 UCP 스키마로 검증합니다.
6. 유효한 `Price` 객체(금액 + 통화)가 있는 상품을 반환합니다.
7. 기본 limit 10으로 커서 기반 페이지네이션을 지원합니다.
8. 배치 크기 제한을 넘는 요청에는 `-32602`(Invalid params)를 반환합니다.
