# 카탈로그 - REST 바인딩

이 문서는 [카탈로그 기능](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/index.md)의 HTTP/REST 바인딩을 정의합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`의 UCP 프로필에서 REST 전송 방식 지원 여부를 선언합니다.

```json
{
  "ucp": {
    "version": "draft",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/overview",
          "transport": "rest",
          "schema": "https://ucp.dev/draft/services/shopping/rest.openapi.json",
          "endpoint": "https://business.example.com/ucp"
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

## 엔드포인트

| 엔드포인트         | 메서드 | 기능                                                                                                                         | 설명                                         |
| ------------------ | ------ | ---------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| `/catalog/search`  | POST   | [검색](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/search/index.md)                 | 상품을 검색합니다.                           |
| `/catalog/lookup`  | POST   | [조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/index.md)                 | ID로 하나 이상의 상품을 조회합니다.          |
| `/catalog/product` | POST   | [조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/#get-product-get_product) | 식별자로 상품의 전체 상세 정보를 가져옵니다. |

### `POST /catalog/search`

[카탈로그 검색](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/search/index.md) 기능에 대응합니다.

**Inputs**

| 이름        | 타입                                                                                  | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ----------- | ------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| query       | string                                                                                | 선택      | 자유 텍스트 검색어.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)                       | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                       | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)               | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| filters     | [Search Filters](/ucp-ko/draft/draft/specification/reference/#search-filters)         | 선택      | 검색 결과를 좁히는 필터 조건. 지정한 모든 필터는 AND 논리로 결합됩니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| pagination  | [Pagination Request](/ucp-ko/draft/draft/specification/reference/#pagination-request) | 선택      | 목록 작업을 위한 커서 기반 페이지네이션.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

**Output**

| 이름       | 타입                                                                                                    | 필수 여부 | 설명                                                                                                                                                     |
| ---------- | ------------------------------------------------------------------------------------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp        | [UCP Catalog Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-catalog-schema) | **필수**  | 카탈로그 응답용 UCP 메타데이터.                                                                                                                          |
| products   | Array\[[Product](/ucp-ko/draft/draft/specification/reference/#product)\]                                | **필수**  | 검색 조건과 일치하는 상품.                                                                                                                               |
| pagination | [Pagination Response](/ucp-ko/draft/draft/specification/reference/#pagination-response)                 | 선택      | 목록 작업을 위한 커서 기반 페이지네이션.                                                                                                                 |
| actions    | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | 선택      | 이 카탈로그 검색 응답에 대해 확장이 정의한 미해결 Action.                                                                                                |
| messages   | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | 선택      | 검색 결과에 관한 오류, 경고, 정보 메시지.                                                                                                                |
| policies   | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | 선택      | 이 검색 결과의 상품에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다. |

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
```

### `POST /catalog/lookup`

[카탈로그 조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/index.md) 기능에 대응합니다. 지원하는 식별자, 해석 동작, 클라이언트 대응 요건은 기능 문서를 참고하세요.

요청 본문에는 식별자 배열과, 배치 안의 모든 조회에 적용되는 선택적 컨텍스트가 담깁니다.

**Inputs**

| 이름        | 타입                                                                          | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ----------- | ----------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ids         | Array[string]                                                                 | **필수**  | 조회할 식별자. 구현은 반드시 상품 ID와 변형 ID를 지원해야 하며(MUST), 보조 식별자(SKU, handle 등)를 지원할 수 있습니다(MAY).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| filters     | [Search Filters](/ucp-ko/draft/draft/specification/reference/#search-filters) | 선택      | 반환되는 상품과 변형을 좁히는 필터 조건. 지정한 모든 필터는 AND 논리로 결합됩니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)               | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)               | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)       | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                                                                                                                                                                                                                                                                                     |

**Output**

| 이름     | 타입                                                                                                    | 필수 여부 | 설명                                                                                                                                                |
| -------- | ------------------------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp      | [UCP Catalog Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-catalog-schema) | **필수**  | 카탈로그 응답용 UCP 메타데이터.                                                                                                                     |
| products | Array\[[Product](/ucp-ko/draft/draft/specification/reference/#product)\]                                | **필수**  | 요청한 식별자와 일치하는 상품입니다. 일부 식별자를 찾지 못하면 항목이 더 적을 수 있고, 식별자가 여러 상품과 일치하면 더 많을 수 있습니다.           |
| actions  | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | 선택      | 이 카탈로그 조회 응답에 대해 확장이 정의한 미해결 Action.                                                                                           |
| messages | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | 선택      | 요청된 항목에 관한 오류, 경고, 정보 메시지.                                                                                                         |
| policies | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | 선택      | 이 응답의 상품에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다. |

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

### `POST /catalog/product`

[카탈로그 조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/#get-product-get_product) 기능에 대응합니다. 상품 상세 페이지 전체를 렌더링할 수 있도록 (배열이 아닌) 단일 `product` 객체를 반환합니다.

**Inputs**

| 이름        | 타입                                                                                     | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ----------- | ---------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id          | string                                                                                   | **필수**  | 상품 또는 변형 식별자입니다. 구현체는 반드시 상품 ID와 변형 ID를 모두 지원해야 합니다(MUST).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| selected    | Array\[[Selected Option](/ucp-ko/draft/draft/specification/reference/#selected-option)\] | 선택      | 대화형으로 변형을 좁혀 가기 위한 부분 또는 전체 옵션 선택. 제공되면 응답의 옵션 값에 이 선택을 기준으로 한 가용성 신호(available, exists)가 포함됩니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| preferences | Array[string]                                                                            | 선택      | 완화 우선순위 순서로 나열한 옵션 이름. 모든 선택과 정확히 일치하는 변형이 없으면 서버는 이 목록의 끝에 있는 옵션부터 제외합니다. 예를 들어 ['Color', 'Size']는 Color를 유지하고 Size를 완화합니다.                                                                                                                                                                                                                                                                                                                                                                                                                    |
| filters     | [Search Filters](/ucp-ko/draft/draft/specification/reference/#search-filters)            | 선택      | 반환되는 변형을 좁히는 필터 조건. 지정한 모든 필터는 AND 논리로 결합됩니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)                          | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                          | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                  | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                                                                                                                                                                                                                                                                                     |

**Output**

This object MUST be one of the following types: [Catalog Lookup Get Product Response](/ucp-ko/draft/draft/specification/shopping/catalog/rest/#catalog-lookup-get-product-response), [Error Response](/ucp-ko/draft/draft/specification/reference/#error-response).

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
```

Green은 품절입니다(`available: false, exists: true`). Blue에는 사이즈 11이 존재하지 않습니다(`exists: false`). 반환된 변형은 Color=Blue 선택과 일치합니다.

#### 상품을 찾을 수 없음

식별자가 상품으로 해석되지 않으면, 서버는 `ucp.status: "error"`와 설명 메시지를 담아 HTTP 200을 반환합니다. 이는 전송 에러가 아니라 애플리케이션 처리 결과입니다. 핸들러가 실행되었고, 그 결과를 UCP 봉투(envelope)로 보고한 것입니다.

```json
{
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
```

(배치 요청에 부분 결과를 반환하는) `/catalog/lookup`과 달리 `/catalog/product`는 단일 리소스 작업입니다. 상품이 없으면 `unrecoverable` 심각도의 애플리케이션 에러이며, 에이전트는 같은 식별자로 재시도하지 않는 것이 좋습니다.

## HTTP 헤더

다음 헤더는 HTTP 바인딩에서 정의하며, 별도 언급이 없으면 모든 작업에 적용됩니다.

**Request Headers**

| 헤더              | 필수   | 설명                                                                                                                                                                                                                                                               |
| ----------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Authorization`   | 아니요 | 다음 두 방식을 나타내는 OAuth 토큰을 담아야 합니다. 1. 플랫폼 자체 인증(client_credentials). 2. 최종 사용자를 대신한 플랫폼 인증(authorization_code).                                                                                                              |
| `X-API-Key`       | 아니요 | 비즈니스가 플랫폼에 할당한 재사용 가능한 API 키로 플랫폼을 인증합니다.                                                                                                                                                                                             |
| `Signature`       | 아니요 | RFC 9421 HTTP Message Signature입니다. 인증에 HTTP Message Signatures를 사용할 때 필수입니다. 형식: `sig1=:<base64-signature>:`.                                                                                                                                   |
| `Signature-Input` | 아니요 | RFC 9421 Signature-Input 헤더입니다. 인증에 HTTP Message Signatures를 사용할 때 필수입니다. 형식: `sig1=("@method" "@path" ...);created=<timestamp>;keyid="<key-id>"`.                                                                                             |
| `Content-Digest`  | 아니요 | RFC 9530 본문 다이제스트. 본문이 있는 요청/응답에는 필수입니다. 형식: `sha-256=:<base64-digest>:`.                                                                                                                                                                 |
| `Request-Id`      | **예** | 네트워크 계층과 구성 요소 전반에서 요청을 추적하기 위한 값.                                                                                                                                                                                                        |
| `User-Agent`      | 아니요 | 호출하는 사용자 에이전트 문자열을 식별합니다.                                                                                                                                                                                                                      |
| `UCP-Agent`       | **예** | 호출하는 UCP 에이전트를 식별합니다. 모든 요청은 반드시 RFC 8941 Dictionary 구문으로 서명자의 프로필 URI를 담은 UCP-Agent 헤더를 포함해야 합니다(MUST). URL은 반드시 /.well-known/ucp를 가리켜야 합니다(MUST). 형식: profile="https://example.com/.well-known/ucp". |
| `Content-Type`    | 아니요 | 표현 메타데이터(Representation Metadata)입니다. 메시지 본문의 데이터가 실제로 무엇인지 수신자에게 알려 줍니다.                                                                                                                                                     |
| `Accept`          | 아니요 | 콘텐츠 협상. 클라이언트가 이해할 수 있는 데이터 형식을 서버에 알립니다.                                                                                                                                                                                            |
| `Accept-Language` | 아니요 | 현지화. 사용자가 선호하는 자연어를 수신자에게 알리며, 흔히 "가중치"나 우선순위가 함께 붙습니다.                                                                                                                                                                    |
| `Accept-Encoding` | 아니요 | 압축. 클라이언트가 지원하는 콘텐츠 인코딩(주로 압축)을 서버에 알립니다.                                                                                                                                                                                            |

**Response Headers**

| 헤더              | 필수   | 설명                                                                                                        |
| ----------------- | ------ | ----------------------------------------------------------------------------------------------------------- |
| `Signature`       | 아니요 | 응답에 대한 RFC 9421 HTTP Message Signature입니다. 서명 값을 `sig1=:<base64-signature>:` 형식으로 담습니다. |
| `Signature-Input` | 아니요 | 응답에 대한 RFC 9421 Signature-Input 헤더입니다. 서명된 구성 요소, 타임스탬프, 키 ID를 기술합니다.          |
| `Content-Digest`  | 아니요 | RFC 8785에 따라 JCS로 정규화한 본문 다이제스트. 형식: `sha-256=:<base64-digest>:`.                          |

### 헤더별 요건

- **UCP-Agent**: All requests **MUST** include the `UCP-Agent` header containing the platform profile URI using Dictionary Structured Field syntax ([RFC 8941](https://datatracker.ietf.org/doc/html/rfc8941)). Format: `profile="https://platform.example/profile"`.

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

애플리케이션 수준의 처리 결과는 모두 UCP 봉투와 선택적 `messages` 배열을 담아 HTTP 200으로 반환합니다. 메시지 의미와 일반적인 시나리오는 [카탈로그 개요](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/#messages-and-error-handling)를 참고하세요.

#### 예시: 모든 상품을 찾을 수 없음

요청한 식별자를 하나도 찾지 못하면 `products` 배열은 비어 있습니다. 응답에 찾지 못한 식별자를 알리는 정보성 메시지를 포함할 수 있습니다(MAY).

```json
{
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

카탈로그 응답용 UCP 메타데이터.

| 이름             | 타입   | 필수 여부 | 설명                                                                                                                    |
| ---------------- | ------ | --------- | ----------------------------------------------------------------------------------------------------------------------- |
| version          | string | **필수**  | YYYY-MM-DD 형식의 버전 식별자.                                                                                          |
| map_order        | object | 선택      | 루트 `ucp` 엔벌로프 안에 있는 형제 레지스트리 필드(`services`, `capabilities`, `payment_handlers`)의 선호 키 순회 순서. |
| status           | string | 선택      | UCP 작업의 애플리케이션 수준 상태. **허용 값:** `success`, `error`                                                      |
| services         | object | 선택      | 역도메인 이름을 키로 하는 서비스 레지스트리.                                                                            |
| capabilities     | object | 선택      | 역도메인 이름을 키로 하는 기능 레지스트리.                                                                              |
| payment_handlers | object | 선택      | 역 도메인 이름을 키로 하는 결제 핸들러 레지스트리.                                                                      |

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

### Get Product 응답

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

적합한 REST 전송 방식 구현은 반드시 다음을 충족해야 합니다(**MUST**).

1. 비즈니스의 UCP 프로필에 선언된 각 카탈로그 기능에 대해, 해당 기능의 요건([검색](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/search/index.md), [조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/index.md))에 따라 엔드포인트를 구현합니다. 각 기능은 독립적으로 채택할 수 있습니다. 조회 기능을 선언했다면 `/catalog/lookup`과 `/catalog/product`를 반드시 모두 제공해야 합니다(MUST).
1. 유효한 `Price` 객체(금액 + 통화)가 포함된 상품을 반환합니다.
1. 공통 페이지네이션 계약에 따라 검색에서 커서 기반 페이지네이션을 지원합니다 ([페이지네이션](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/search/#pagination) 참고).
1. 조회 요청에는 HTTP 200을 반환합니다. 알 수 없는 식별자가 있으면 반환되는 상품 수가 줄어듭니다(정보성 `not_found` 메시지를 포함할 수 있습니다(MAY)).
1. 배치 크기 제한을 넘는 요청에는 `request_too_large` 에러와 함께 HTTP 400을 반환합니다.
