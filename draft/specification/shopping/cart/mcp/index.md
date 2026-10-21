# 장바구니 기능 - MCP 바인딩

이 문서는 [장바구니 기능(Cart Capability)](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/index.md)의 Model Context Protocol(MCP) 바인딩을 정의합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`의 UCP 프로필에서 MCP 전송 방식 지원 여부를 알립니다.

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
      "dev.ucp.shopping.checkout": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/shopping/checkout",
          "schema": "https://ucp.dev/draft/schemas/shopping/checkout.json"
        }
      ],
      "dev.ucp.shopping.cart": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/shopping/cart",
          "schema": "https://ucp.dev/draft/schemas/shopping/cart.json"
        }
      ]
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
    "name": "create_cart",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/shopping-agent.json"
        }
      },
      "cart": { "line_items": [ ... ] }
    }
  }
}
```

[기능 협상](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#negotiation-protocol)을 위해 모든 요청에 `meta["ucp-agent"]` 필드가 **필수**입니다. 플랫폼은 추가 메타데이터 필드를 포함할 수 있습니다(**MAY**).

## 도구

UCP 기능은 MCP 도구와 1:1로 대응합니다.

### 식별자 패턴

MCP 도구는 리소스 식별과 페이로드 데이터를 분리합니다.

- **요청:** 기존 장바구니에 대한 작업(`get`, `update`, `cancel`)에서는 최상위 `id` 파라미터로 대상 리소스를 식별합니다. 요청 페이로드의 `cart` 객체에는 `id` 필드가 있어서는 안 됩니다(**MUST NOT**).
- **응답:** 모든 응답은 전체 리소스 상태의 일부로 `cart.id`를 포함합니다.
- **생성:** `create_cart` 작업은 요청에 `id`가 필요 없으며, 응답에 새로 할당된 `cart.id`가 포함됩니다.

| 도구          | 작업                                                                                                            | 설명                        |
| ------------- | --------------------------------------------------------------------------------------------------------------- | --------------------------- |
| `create_cart` | [장바구니 생성](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/#create-cart) | 장바구니 세션을 생성합니다. |
| `get_cart`    | [장바구니 조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/#get-cart)    | 장바구니 세션을 조회합니다. |
| `update_cart` | [장바구니 수정](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/#update-cart) | 장바구니 세션을 수정합니다. |
| `cancel_cart` | [장바구니 취소](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/#cancel-cart) | 장바구니 세션을 취소합니다. |

### `create_cart`

[장바구니 생성](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/#create-cart) 작업에 대응합니다.

#### 입력 스키마

| 이름        | 타입                                                                         | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | ---------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| line_items  | Array\[[Line Item](/ucp-ko/draft/draft/specification/reference/#line-item)\] | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)              | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)              | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)      | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer       | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                  | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |

#### 출력 스키마

| 이름         | 타입                                                                                              | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | [UCP Cart Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-cart-schema) | **필수**  | 장바구니 응답용 UCP 메타데이터. 체크아웃 전에는 결제 핸들러가 필요 없습니다.                                                                                                                                                                                                                                                                            |
| id           | string                                                                                            | **필수**  | 고유 장바구니 식별자.                                                                                                                                                                                                                                                                                                                                   |
| line_items   | Array\[[Line Item Response](/ucp-ko/draft/draft/specification/reference/#line-item)\]             | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                                   | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                                   | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution  | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                           | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer        | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                                       | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |
| currency     | string                                                                                            | **필수**  | ISO 4217 통화 코드. 판매자가 컨텍스트나 geo-IP를 바탕으로 결정합니다.                                                                                                                                                                                                                                                                                   |
| totals       | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                     | **필수**  | 예상 비용 내역. 배송비·세금을 아직 계산할 수 없으면 일부만 포함될 수 있습니다.                                                                                                                                                                                                                                                                          |
| actions      | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                   | 선택      | 이 장바구니에 대해 확장이 정의한 미해결 Action.                                                                                                                                                                                                                                                                                                         |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                          | 선택      | 검증 메시지, 경고 또는 안내성 알림.                                                                                                                                                                                                                                                                                                                     |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]                                | 선택      | 판매자 링크(선택 사항, 정책, FAQ).                                                                                                                                                                                                                                                                                                                      |
| policies     | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                            | 선택      | 이 장바구니의 항목에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다.                                                                                                                                                                                                 |
| continue_url | string                                                                                            | 선택      | 장바구니 핸드오프와 세션 복구용 URL. 공유와 사람이 개입하는(human-in-the-loop) 흐름을 지원합니다.                                                                                                                                                                                                                                                       |
| expires_at   | string                                                                                            | 선택      | 장바구니 만료 타임스탬프(RFC 3339). 선택 사항.                                                                                                                                                                                                                                                                                                          |

#### 예시

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "create_cart",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/shopping-agent.json"
        }
      },
      "cart": {
        "line_items": [
          {
            "item": {
              "id": "item_123"
            },
            "quantity": 2
          }
        ],
        "context": {
          "address_country": "US",
          "address_region": "CA",
          "postal_code": "94105"
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
          "dev.ucp.shopping.checkout": [{"version": "draft"}],
          "dev.ucp.shopping.cart": [{"version": "draft"}]
        }
      },
      "id": "cart_abc123",
      "line_items": [
        {
          "id": "li_1",
          "item": {
            "id": "item_123",
            "title": "Red T-Shirt",
            "price": 2500
          },
          "quantity": 2,
          "totals": [
            {"type": "subtotal", "amount": 5000},
            {"type": "total", "amount": 5000}
          ]
        }
      ],
      "currency": "USD",
      "totals": [
        {
          "type": "subtotal",
          "amount": 5000
        },
        {
          "type": "total",
          "amount": 5000
        }
      ],
      "continue_url": "https://business.example.com/checkout?cart=cart_abc123",
      "expires_at": "2026-01-16T12:00:00Z"
    },
    "content": [
      {
        "type": "text",
        "text": "{\"ucp\":{…},…}"
      }
    ]
  }
}
```

모든 상품이 품절이라 장바구니 리소스가 생성되지 않습니다.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": { "version": "draft", "status": "error" },
      "messages": [
        {
          "type": "error",
          "code": "out_of_stock",
          "content": "All requested items are currently out of stock",
          "severity": "unrecoverable"
        }
      ],
      "continue_url": "https://merchant.com/"
    },
    "content": [
      {"type": "text", "text": "{\"ucp\":{…},…}"}
    ]
  }
}
```

### `get_cart`

[장바구니 조회](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/#get-cart) 작업에 대응합니다.

#### 입력 스키마

- `id` (String, 필수): 장바구니 세션의 ID.

#### 출력 스키마

| 이름         | 타입                                                                                              | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | [UCP Cart Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-cart-schema) | **필수**  | 장바구니 응답용 UCP 메타데이터. 체크아웃 전에는 결제 핸들러가 필요 없습니다.                                                                                                                                                                                                                                                                            |
| id           | string                                                                                            | **필수**  | 고유 장바구니 식별자.                                                                                                                                                                                                                                                                                                                                   |
| line_items   | Array\[[Line Item Response](/ucp-ko/draft/draft/specification/reference/#line-item)\]             | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                                   | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                                   | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution  | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                           | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer        | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                                       | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |
| currency     | string                                                                                            | **필수**  | ISO 4217 통화 코드. 판매자가 컨텍스트나 geo-IP를 바탕으로 결정합니다.                                                                                                                                                                                                                                                                                   |
| totals       | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                     | **필수**  | 예상 비용 내역. 배송비·세금을 아직 계산할 수 없으면 일부만 포함될 수 있습니다.                                                                                                                                                                                                                                                                          |
| actions      | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                   | 선택      | 이 장바구니에 대해 확장이 정의한 미해결 Action.                                                                                                                                                                                                                                                                                                         |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                          | 선택      | 검증 메시지, 경고 또는 안내성 알림.                                                                                                                                                                                                                                                                                                                     |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]                                | 선택      | 판매자 링크(선택 사항, 정책, FAQ).                                                                                                                                                                                                                                                                                                                      |
| policies     | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                            | 선택      | 이 장바구니의 항목에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다.                                                                                                                                                                                                 |
| continue_url | string                                                                                            | 선택      | 장바구니 핸드오프와 세션 복구용 URL. 공유와 사람이 개입하는(human-in-the-loop) 흐름을 지원합니다.                                                                                                                                                                                                                                                       |
| expires_at   | string                                                                                            | 선택      | 장바구니 만료 타임스탬프(RFC 3339). 선택 사항.                                                                                                                                                                                                                                                                                                          |

#### 예시

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_cart",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/shopping-agent.json"
        }
      },
      "id": "cart_abc123"
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
          "dev.ucp.shopping.checkout": [{"version": "draft"}],
          "dev.ucp.shopping.cart": [{"version": "draft"}]
        }
      },
      "id": "cart_abc123",
      "line_items": [
        {
          "id": "li_1",
          "item": {
            "id": "item_123",
            "title": "Red T-Shirt",
            "price": 2500
          },
          "quantity": 2,
          "totals": [
            {"type": "subtotal", "amount": 5000},
            {"type": "total", "amount": 5000}
          ]
        }
      ],
      "currency": "USD",
      "totals": [
        {
          "type": "subtotal",
          "amount": 5000
        },
        {
          "type": "total",
          "amount": 5000
        }
      ],
      "continue_url": "https://business.example.com/checkout?cart=cart_abc123",
      "expires_at": "2026-01-16T12:00:00Z"
    },
    "content": [
      {
        "type": "text",
        "text": "{\"ucp\":{…},…}"
      }
    ]
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
        "status": "error",
        "capabilities": {
          "dev.ucp.shopping.cart": [{"version": "draft"}]
        }
      },
      "messages": [
        {
          "type": "error",
          "code": "not_found",
          "content": "Cart not found or has expired",
          "severity": "unrecoverable"
        }
      ],
      "continue_url": "https://merchant.com/"
    },
    "content": [
      {
        "type": "text",
        "text": "{\"ucp\":{…},…}"
      }
    ]
  }
}
```

### `update_cart`

[장바구니 수정](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/#update-cart) 작업에 대응합니다.

#### 입력 스키마

- `id` (String, 필수): 수정할 장바구니 세션의 ID.

| 이름        | 타입                                                                         | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | ---------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| line_items  | Array\[[Line Item](/ucp-ko/draft/draft/specification/reference/#line-item)\] | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)              | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)              | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)      | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer       | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                  | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |

#### 출력 스키마

| 이름         | 타입                                                                                              | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | [UCP Cart Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-cart-schema) | **필수**  | 장바구니 응답용 UCP 메타데이터. 체크아웃 전에는 결제 핸들러가 필요 없습니다.                                                                                                                                                                                                                                                                            |
| id           | string                                                                                            | **필수**  | 고유 장바구니 식별자.                                                                                                                                                                                                                                                                                                                                   |
| line_items   | Array\[[Line Item Response](/ucp-ko/draft/draft/specification/reference/#line-item)\]             | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                                   | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                                   | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution  | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                           | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer        | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                                       | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |
| currency     | string                                                                                            | **필수**  | ISO 4217 통화 코드. 판매자가 컨텍스트나 geo-IP를 바탕으로 결정합니다.                                                                                                                                                                                                                                                                                   |
| totals       | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                     | **필수**  | 예상 비용 내역. 배송비·세금을 아직 계산할 수 없으면 일부만 포함될 수 있습니다.                                                                                                                                                                                                                                                                          |
| actions      | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                   | 선택      | 이 장바구니에 대해 확장이 정의한 미해결 Action.                                                                                                                                                                                                                                                                                                         |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                          | 선택      | 검증 메시지, 경고 또는 안내성 알림.                                                                                                                                                                                                                                                                                                                     |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]                                | 선택      | 판매자 링크(선택 사항, 정책, FAQ).                                                                                                                                                                                                                                                                                                                      |
| policies     | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                            | 선택      | 이 장바구니의 항목에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다.                                                                                                                                                                                                 |
| continue_url | string                                                                                            | 선택      | 장바구니 핸드오프와 세션 복구용 URL. 공유와 사람이 개입하는(human-in-the-loop) 흐름을 지원합니다.                                                                                                                                                                                                                                                       |
| expires_at   | string                                                                                            | 선택      | 장바구니 만료 타임스탬프(RFC 3339). 선택 사항.                                                                                                                                                                                                                                                                                                          |

#### 예시

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "update_cart",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/shopping-agent.json"
        }
      },
      "id": "cart_abc123",
      "cart": {
        "line_items": [
          {
            "item": {
              "id": "item_123"
            },
            "quantity": 3
          },
          {
            "item": {
              "id": "item_456"
            },
            "quantity": 1
          }
        ],
        "context": {
          "address_country": "US",
          "address_region": "CA",
          "postal_code": "94105"
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
          "dev.ucp.shopping.checkout": [{"version": "draft"}],
          "dev.ucp.shopping.cart": [{"version": "draft"}]
        }
      },
      "id": "cart_abc123",
      "line_items": [
        {
          "id": "li_1",
          "item": {
            "id": "item_123",
            "title": "Red T-Shirt",
            "price": 2500
          },
          "quantity": 3,
          "totals": [
            {"type": "subtotal", "amount": 7500},
            {"type": "total", "amount": 7500}
          ]
        },
        {
          "id": "li_2",
          "item": {
            "id": "item_456",
            "title": "Blue Jeans",
            "price": 7500
          },
          "quantity": 1,
          "totals": [
            {"type": "subtotal", "amount": 7500},
            {"type": "total", "amount": 7500}
          ]
        }
      ],
      "currency": "USD",
      "totals": [
        {
          "type": "subtotal",
          "amount": 15000
        },
        {
          "type": "total",
          "amount": 15000
        }
      ],
      "continue_url": "https://business.example.com/checkout?cart=cart_abc123",
      "expires_at": "2026-01-16T12:00:00Z"
    },
    "content": [
      {
        "type": "text",
        "text": "{\"ucp\":{…},…}"
      }
    ]
  }
}
```

### `cancel_cart`

[장바구니 취소](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/#cancel-cart) 작업에 대응합니다.

#### 입력 스키마

- `id` (String, 필수): 장바구니 세션의 ID.

#### 출력 스키마

| 이름         | 타입                                                                                              | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | [UCP Cart Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-cart-schema) | **필수**  | 장바구니 응답용 UCP 메타데이터. 체크아웃 전에는 결제 핸들러가 필요 없습니다.                                                                                                                                                                                                                                                                            |
| id           | string                                                                                            | **필수**  | 고유 장바구니 식별자.                                                                                                                                                                                                                                                                                                                                   |
| line_items   | Array\[[Line Item Response](/ucp-ko/draft/draft/specification/reference/#line-item)\]             | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                                   | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                                   | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution  | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                           | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer        | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                                       | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |
| currency     | string                                                                                            | **필수**  | ISO 4217 통화 코드. 판매자가 컨텍스트나 geo-IP를 바탕으로 결정합니다.                                                                                                                                                                                                                                                                                   |
| totals       | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                     | **필수**  | 예상 비용 내역. 배송비·세금을 아직 계산할 수 없으면 일부만 포함될 수 있습니다.                                                                                                                                                                                                                                                                          |
| actions      | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                   | 선택      | 이 장바구니에 대해 확장이 정의한 미해결 Action.                                                                                                                                                                                                                                                                                                         |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                          | 선택      | 검증 메시지, 경고 또는 안내성 알림.                                                                                                                                                                                                                                                                                                                     |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]                                | 선택      | 판매자 링크(선택 사항, 정책, FAQ).                                                                                                                                                                                                                                                                                                                      |
| policies     | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                            | 선택      | 이 장바구니의 항목에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다.                                                                                                                                                                                                 |
| continue_url | string                                                                                            | 선택      | 장바구니 핸드오프와 세션 복구용 URL. 공유와 사람이 개입하는(human-in-the-loop) 흐름을 지원합니다.                                                                                                                                                                                                                                                       |
| expires_at   | string                                                                                            | 선택      | 장바구니 만료 타임스탬프(RFC 3339). 선택 사항.                                                                                                                                                                                                                                                                                                          |

#### 예시

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "cancel_cart",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/shopping-agent.json"
        },
        "idempotency-key": "660e8400-e29b-41d4-a716-446655440001"
      },
      "id": "cart_abc123"
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
          "dev.ucp.shopping.checkout": [{"version": "draft"}],
          "dev.ucp.shopping.cart": [{"version": "draft"}]
        }
      },
      "id": "cart_abc123",
      "line_items": [
        {
          "id": "li_1",
          "item": {
            "id": "item_123",
            "title": "Red T-Shirt",
            "price": 2500
          },
          "quantity": 2,
          "totals": [
            {"type": "subtotal", "amount": 5000},
            {"type": "total", "amount": 5000}
          ]
        }
      ],
      "currency": "USD",
      "totals": [
        {
          "type": "subtotal",
          "amount": 5000
        },
        {
          "type": "total",
          "amount": 5000
        }
      ],
      "continue_url": "https://business.example.com/checkout?cart=cart_abc123"
    },
    "content": [
      {
        "type": "text",
        "text": "{\"ucp\":{…},…}"
      }
    ]
  }
}
```

## 오류 처리

UCP는 프로토콜 오류와 비즈니스 결과를 구분합니다. 전체 오류 코드 레지스트리와 전송 바인딩 예시는 [핵심 명세](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#error-handling)를 참고하세요.

- **프로토콜 오류**: 요청 처리를 막는 전송 수준 실패(인증, 요청 한도 제한, 서비스 불가). 코드 `-32000`(디스커버리 오류는 `-32001`)과 함께 JSON-RPC `error`로 반환합니다.
- **비즈니스 결과**: 요청 처리가 성공한 뒤의 애플리케이션 수준 결과. UCP 엔벌로프와 `messages`를 담은 JSON-RPC `result`로 반환합니다.

### 비즈니스 결과

비즈니스 결과(찾을 수 없음, 유효성 검사 오류 포함)는 UCP 엔벌로프와 `messages`를 담은 `structuredContent`가 있는 JSON-RPC `result`로 반환합니다.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "draft",
        "status": "error",
        "capabilities": {
          "dev.ucp.shopping.cart": [{"version": "draft"}]
        }
      },
      "messages": [
        {
          "type": "error",
          "code": "not_found",
          "content": "Cart not found or has expired",
          "severity": "unrecoverable"
        }
      ],
      "continue_url": "https://merchant.com/"
    },
    "content": [
      {"type": "text", "text": "{\"ucp\":{…},…}"}
    ]
  }
}
```

## 준수

준수하는 MCP 전송 방식 구현은 반드시 다음을 지켜야 합니다(**MUST**).

1. JSON-RPC 2.0 프로토콜을 올바르게 구현합니다.
1. 이 명세에 정의된 모든 핵심 장바구니 도구를 제공합니다.
1. [핵심 명세](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#error-handling)에 따라 오류를 반환합니다.
1. 비즈니스 결과를 UCP 엔벌로프와 `messages` 배열을 담은 JSON-RPC `result`로 반환합니다.
1. UCP 스키마에 따라 도구 입력을 검증합니다.
1. 스트리밍을 지원하는 HTTP 전송 방식을 지원합니다.
