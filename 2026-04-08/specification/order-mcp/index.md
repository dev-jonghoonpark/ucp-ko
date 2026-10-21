# 주문 기능 - MCP 바인딩

이 문서는 [주문 기능(Order Capability)](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/order/index.md)의 Model Context Protocol(MCP) 바인딩을 정의합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필로 MCP 전송 방식 지원 여부를 알립니다.

```json
{
  "ucp": {
    "version": "2026-04-08",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "2026-04-08",
          "spec": "https://ucp.dev/2026-04-08/specification/overview",
          "transport": "mcp",
          "schema": "https://ucp.dev/2026-04-08/services/shopping/mcp.openrpc.json",
          "endpoint": "https://business.example.com/ucp/mcp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.order": [
        {
          "version": "2026-04-08",
          "spec": "https://ucp.dev/2026-04-08/specification/order",
          "schema": "https://ucp.dev/2026-04-08/schemas/shopping/order.json"
        }
      ]
    }
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
    "name": "get_order",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/.well-known/ucp"
        }
      },
      "id": "order_abc123"
    }
  }
}
```

[기능 협상](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/#negotiation-protocol)에 쓰이므로 모든 요청에 `meta["ucp-agent"]` 필드가 **필수**입니다. 플랫폼은 추가 메타데이터 필드를 포함할 수 있습니다(**MAY**).

## 도구

UCP 기능은 MCP 도구(Tool)와 1:1로 대응합니다.

| 도구        | 작업                                                                                                        | 설명                           |
| ----------- | ----------------------------------------------------------------------------------------------------------- | ------------------------------ |
| `get_order` | [주문 조회](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/order/#get-order) | 주문의 현재 상태를 조회합니다. |

### `get_order`

[주문 조회](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/order/#get-order) 작업에 대응합니다. 주문의 현재 상태 스냅숏을 반환합니다.

#### 입력 스키마

- `meta` (Object, 필수): `ucp-agent.profile`을 포함한 요청 메타데이터.
- `id` (String, 필수): 주문 ID.

#### 출력 스키마

| 이름          | 타입                                                                                               | 필수   | 설명                                                                                                 |
| ------------- | -------------------------------------------------------------------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------- |
| ucp           | any                                                                                                | **예** | 주문 응답용 UCP 메타데이터. 구매 후에는 결제 핸들러가 필요 없습니다.                                 |
| id            | string                                                                                             | **예** | 고유 주문 식별자.                                                                                    |
| label         | string                                                                                             | 아니요 | 주문을 식별하기 위한 사람이 읽을 수 있는 라벨. 반드시 비즈니스만 제공해야 합니다(MUST).              |
| checkout_id   | string                                                                                             | **예** | 정산 대사를 위한 연관 체크아웃 ID.                                                                   |
| permalink_url | string                                                                                             | **예** | 판매자 사이트에서 주문에 접근하는 퍼머링크.                                                          |
| line_items    | Array\[[Order Line Item](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#order-line-item)\] | **예** | 구매한 항목을 나타내는 라인 아이템. 주문 후 수정이나 교환으로 바뀔 수 있습니다.                      |
| fulfillment   | object                                                                                             | **예** | 풀필먼트 데이터: 구매자 예상 정보와 실제로 일어난 일.                                                |
| adjustments   | Array\[[Adjustment](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#adjustment)\]           | 아니요 | 풀필먼트와 독립적으로 존재하는 주문 후 이벤트(환불, 반품, 크레딧, 분쟁, 취소 등).                    |
| currency      | string                                                                                             | **예** | ISO 4217 통화 코드. 반드시 원래 체크아웃 세션의 통화와 일치해야 합니다(MUST).                        |
| totals        | [Totals](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#totals)                            | **예** | 주문의 각종 합계.                                                                                    |
| messages      | Array\[[Message](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#message)\]                 | 아니요 | 비즈니스 결과 메시지(오류, 경고, 정보). 비즈니스가 플랫폼에 상태나 문제를 전달해야 할 때 포함됩니다. |
| attribution   | [Attribution](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#attribution)                  | 아니요 | 원래 체크아웃과 연결된 어트리뷰션의 스냅샷입니다. 주문에서는 읽기 전용입니다.                        |

#### 예시

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_order",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/.well-known/ucp"
        }
      },
      "id": "order_abc123"
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
        "version": "2026-04-08",
        "capabilities": {
          "dev.ucp.shopping.order": [{"version": "2026-04-08"}]
        }
      },
      "id": "order_abc123",
      "checkout_id": "checkout_xyz789",
      "permalink_url": "https://business.example.com/orders/abc123",
      "currency": "USD",
      "line_items": [
        {
          "id": "li_shoes",
          "item": { "id": "prod_shoes", "title": "Running Shoes", "price": 3000 },
          "quantity": { "total": 1, "fulfilled": 1 },
          "totals": [
            {"type": "subtotal", "amount": 3000},
            {"type": "total", "amount": 3000}
          ],
          "status": "fulfilled"
        }
      ],
      "fulfillment": {
        "expectations": [
          {
            "id": "exp_1",
            "line_items": [{ "id": "li_shoes", "quantity": 1 }],
            "method_type": "shipping",
            "destination": {
              "street_address": "123 Main St",
              "address_locality": "Austin",
              "address_region": "TX",
              "address_country": "US",
              "postal_code": "78701"
            },
            "description": "Delivered"
          }
        ],
        "events": [
          {
            "id": "evt_1",
            "occurred_at": "2026-01-08T10:30:00Z",
            "type": "delivered",
            "line_items": [{ "id": "li_shoes", "quantity": 1 }],
            "tracking_number": "1Z999AA10123456784",
            "tracking_url": "https://ups.com/track/1Z999AA10123456784",
            "description": "Delivered to front door"
          }
        ]
      },
      "adjustments": [],
      "totals": [
        { "type": "subtotal", "amount": 3000 },
        { "type": "fulfillment", "amount": 800 },
        { "type": "tax", "amount": 304 },
        { "type": "total", "amount": 4104 }
      ]
    },
    "content": [
      {
        "type": "text",
        "text": "{\"ucp\":{...},\"id\":\"order_abc123\",...}"
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
        "version": "2026-04-08",
        "status": "error",
        "capabilities": {
          "dev.ucp.shopping.order": [{"version": "2026-04-08"}]
        }
      },
      "messages": [
        {
          "type": "error",
          "code": "not_found",
          "severity": "unrecoverable",
          "content": "Order not found."
        }
      ]
    },
    "content": [
      {
        "type": "text",
        "text": "Order not found."
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
        "version": "2026-04-08",
        "status": "error",
        "capabilities": {
          "dev.ucp.shopping.order": [{"version": "2026-04-08"}]
        }
      },
      "messages": [
        {
          "type": "error",
          "code": "unauthorized",
          "severity": "unrecoverable",
          "content": "Not authorized to access this order."
        }
      ]
    },
    "content": [
      {
        "type": "text",
        "text": "Not authorized to access this order."
      }
    ]
  }
}
```

## 오류 처리

비즈니스가 주문을 반환할 수 없으면 응답의 `messages` 배열에 그 결과가 설명되어 있습니다. 플랫폼은 주문 필드에 접근하기 전에 반드시 `messages`를 확인해야 합니다(**MUST**).

## 준수 요건

MCP 바인딩을 구현하는 플랫폼의 요건은 다음과 같습니다.

- 모든 요청에 `meta.ucp-agent.profile`을 반드시 포함해야 합니다(**MUST**).
- 주문 데이터에 접근하기 전에 응답의 `messages` 배열을 반드시 확인해야 합니다(**MUST**).
- 공식 주문 경험은 `permalink_url`로 비즈니스에 넘겨야 합니다(**SHOULD**). 주문 세부 정보와 구매 후 작업의 기준(source of truth)은 비즈니스 사이트입니다.

MCP 바인딩을 구현하는 비즈니스의 요건은 다음과 같습니다.

- [OpenRPC 스키마](https://ucp.dev/services/shopping/mcp.openrpc.json)에 따라 `get_order` 도구를 반드시 구현해야 합니다(**MUST**).

모든 전송 방식에 공통으로 적용되는 기능 수준 요구 사항은 [주문 기능 - 가이드라인](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/order/#operations-guidelines)을 참고하세요.
