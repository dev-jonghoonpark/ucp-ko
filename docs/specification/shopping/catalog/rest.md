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

# 카탈로그 - REST 바인딩 { #catalog-rest-binding }

이 문서는 [카탈로그 기능](index.md)의 HTTP/REST 바인딩을 명세한다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`의 UCP 프로필로 REST 전송 방식 지원 여부를 공지한다.

<!-- ucp:example schema=profile def=business_schema -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview",
          "transport": "rest",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/rest.openapi.json",
          "endpoint": "https://business.example.com/ucp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.catalog.search": [{
        "version": "{{ ucp_version }}",
        "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/catalog/search",
        "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/catalog_search.json"
      }],
      "dev.ucp.shopping.catalog.lookup": [{
        "version": "{{ ucp_version }}",
        "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/catalog/lookup",
        "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/catalog_lookup.json"
      }]
    },
    "payment_handlers": {}
  }
}
```

## 엔드포인트 { #endpoints }

| 엔드포인트 | 메서드 | 기능 | 설명 |
| :--- | :--- | :--- | :--- |
| `/catalog/search` | POST | [검색](search.md) | 상품을 검색한다. |
| `/catalog/lookup` | POST | [조회](lookup.md) | ID로 하나 이상의 상품을 조회한다. |
| `/catalog/product` | POST | [조회](lookup.md#get-product-get_product) | 식별자로 상품 상세 정보 전체를 가져온다. |

### `POST /catalog/search` { #post-catalogsearch }

[카탈로그 검색](search.md) 기능에 대응한다.

{{ method_fields('search_catalog', 'shopping/rest.openapi.json', 'shopping/catalog/rest') }}

#### 예시 { #example }

=== "요청"

    <!-- ucp:example schema=shopping/catalog_search op=search direction=request -->
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

=== "응답"

    <!-- ucp:example schema=shopping/catalog_search op=search -->
    ```json
    {
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
    ```

### `POST /catalog/lookup` { #post-cataloglookup }

[카탈로그 조회](lookup.md) 기능에 대응한다. 지원하는 식별자, 해석 동작, 클라이언트 대응
요건은 기능 문서를 참고한다.

요청 본문에는 식별자 배열과, 배치 내 모든 조회에 적용되는 선택적 컨텍스트가 담긴다.

{{ method_fields('lookup_catalog', 'shopping/rest.openapi.json', 'shopping/catalog/rest') }}

#### 예시: 컨텍스트를 포함한 일괄 조회 { #example-batch-lookup-with-context }

=== "요청"

    <!-- ucp:example schema=shopping/catalog_lookup op=lookup direction=request -->
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

=== "응답"

    <!-- ucp:example schema=shopping/catalog_lookup op=lookup -->
    ```json
    {
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
              "description": { "plain": "Variante talla 10" },
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

#### 예시: 부분 성공 (일부 식별자를 찾을 수 없음) { #example-partial-success-some-identifiers-not-found }

배치의 일부 식별자를 찾을 수 없으면, 응답의 `products` 배열에는 찾은 상품만 포함된다.
응답에는 찾지 못한 식별자를 알리는 정보성 메시지를 포함할 수 있다(MAY).

=== "요청"

    <!-- ucp:example schema=shopping/catalog_lookup op=lookup direction=request -->
    ```json
    {
      "ids": ["prod_abc123", "prod_invalid", "prod_def456"]
    }
    ```

=== "응답"

    <!-- ucp:example schema=shopping/catalog_lookup op=lookup -->
    ```json
    {
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
          "description": { "plain": "Lightweight running shoes." },
          "variants": [ ... ],
          "price_range": {
            "min": { "amount": 12000, "currency": "USD" },
            "max": { "amount": 12000, "currency": "USD" }
          }
        },
        {
          "id": "prod_def456",
          "title": "Trail Blazer X",
          "description": { "plain": "Trail shoes with superior traction." },
          "variants": [ ... ],
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

### `POST /catalog/product` { #post-catalogproduct }

[카탈로그 조회](lookup.md#get-product-get_product) 기능에 대응한다. 상품 상세 페이지
전체를 렌더링할 수 있도록 (배열이 아닌) 단일 `product` 객체를 반환한다.

{{ method_fields('get_product', 'shopping/rest.openapi.json', 'shopping/catalog/rest') }}

#### 예시: 옵션 선택 포함 { #example-with-option-selection }

사용자가 Color=Blue를 선택했다. 응답은 옵션 값에 재고 상태 신호를 포함하며, 선택과
일치하는 변형을 반환한다.

=== "요청"

    <!-- ucp:example schema=shopping/catalog_lookup op=get_product direction=request -->
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

=== "응답"

    <!-- ucp:example schema=shopping/catalog_lookup op=get_product -->
    ```json
    {
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
    ```

Green은 품절이다(`available: false, exists: true`). Blue에는 사이즈 11이 존재하지
않는다(`exists: false`). 반환된 변형은 Color=Blue 선택과 일치한다.

#### 상품을 찾을 수 없음 { #product-not-found }

식별자가 상품으로 해석되지 않으면, 서버는 `ucp.status: "error"`와 설명 메시지를 담아
HTTP 200을 반환한다. 이는 전송 에러가 아니라 애플리케이션 처리 결과다. 핸들러가 실행되어
UCP 봉투(envelope)로 결과를 보고한 것이다.

<!-- ucp:example schema=common/types/error_response op=read -->
```json
{
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
```

(배치 요청에 부분 결과를 반환하는) `/catalog/lookup`과 달리 `/catalog/product`는 단일
리소스 작업이다. 상품이 없으면 `unrecoverable` 심각도의 애플리케이션 에러이며,
에이전트는 같은 식별자로 재시도하지 않아야 한다.

## HTTP 헤더 { #http-headers }

다음 헤더는 HTTP 바인딩에 정의되며, 별도 언급이 없으면 모든 작업에 적용된다.

{{ header_fields('search_catalog', 'shopping/rest.openapi.json') }}

### 헤더별 요건 { #specific-header-requirements }

{{ header_requirements('ucp_agent') }}

## 에러 처리 { #error-handling }

UCP는 전송 에러와 비즈니스 처리 결과를 분리하는 2계층 에러 모델을 사용한다.

### 전송 에러 { #transport-errors }

요청 처리를 가로막는 프로토콜 수준 문제에는 HTTP 상태 코드를 사용한다.

| 상태 | 의미 |
| :--- | :--- |
| 400 | Bad Request - 잘못된 형식의 JSON 또는 필수 파라미터 누락 |
| 401 | Unauthorized - 인증 정보가 없거나 유효하지 않음 |
| 429 | Too Many Requests - 요청 속도 제한 |
| 500 | Internal Server Error |

### 비즈니스 처리 결과 { #business-outcomes }

모든 애플리케이션 수준 처리 결과는 UCP 봉투와 선택적 `messages` 배열을 담아 HTTP 200을
반환한다. 메시지 의미와 일반적인 시나리오는
[카탈로그 개요](index.md#messages-and-error-handling)를 참고한다.

#### 예시: 모든 상품을 찾을 수 없음 { #example-all-products-not-found }

요청한 식별자의 조회가 모두 실패하면 `products` 배열은 비어 있다. 응답에는 찾지 못한
식별자를 알리는 정보성 메시지를 포함할 수 있다(MAY).

<!-- ucp:example schema=shopping/catalog_lookup op=lookup -->
```json
{
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

비즈니스 처리 결과는 표준 HTTP 200 상태를 사용하고 응답 본문에 메시지를 담는다.

## 엔터티 { #entities }

### UCP 응답 카탈로그 {: #ucp-response-catalog-schema }

{{ extension_schema_fields('ucp.json#/$defs/response_catalog_schema', 'shopping/catalog/rest') }}

### 상세 상품 {: #detail-product }

{{ extension_schema_fields('catalog_lookup.json#/$defs/detail_product', 'shopping/catalog/rest') }}

### Get Product 응답 {: #catalog-lookup-get-product-response }

{{ extension_schema_fields('catalog_lookup.json#/$defs/get_product_response', 'shopping/catalog/rest') }}

### 에러 응답 {: #error-response }

{{ schema_fields('types/error_response', 'shopping/catalog/rest') }}

## 적합성 { #conformance }

적합한 REST 전송 방식 구현은 반드시 다음을 충족해야 한다(**MUST**).

1. 비즈니스의 UCP 프로필에 공지된 각 카탈로그 기능에 대해, 해당 기능의 요건([검색](search.md), [조회](lookup.md))에 따라 엔드포인트를 구현한다. 각 기능은 독립적으로 채택할 수 있다. 조회 기능을 공지했다면 `/catalog/lookup`과 `/catalog/product`를 반드시 모두 제공해야 한다(MUST).
2. 유효한 `Price` 객체(금액 + 통화)를 가진 상품을 반환한다.
3. 공통 페이지네이션 계약에 따라 검색에서 커서 기반 페이지네이션을 지원한다
    ([페이지네이션](search.md#pagination) 참고).
4. 조회 요청에는 HTTP 200을 반환한다. 알 수 없는 식별자가 있으면 반환되는 상품 수가 줄어든다(정보성 `not_found` 메시지를 포함할 수 있다(MAY)).
5. 배치 크기 제한을 초과하는 요청에는 `request_too_large` 에러와 함께 HTTP 400을 반환한다.
