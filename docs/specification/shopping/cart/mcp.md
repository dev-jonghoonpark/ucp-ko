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

# 장바구니 기능 - MCP 바인딩 { #cart-capability-mcp-binding }

이 문서는 [장바구니 기능(Cart Capability)](index.md)의 Model Context Protocol(MCP)
바인딩을 명세한다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`의 UCP 프로필을 통해 MCP 전송 방식 지원 여부를
알린다.

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
          "transport": "mcp",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/mcp.openrpc.json",
          "endpoint": "https://business.example.com/ucp/mcp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.checkout": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/checkout",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/checkout.json"
        }
      ],
      "dev.ucp.shopping.cart": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/cart",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/cart.json"
        }
      ]
    },
    "payment_handlers": {}
  }
}
```

### 요청 메타데이터 { #request-metadata }

MCP 클라이언트는 모든 요청에 프로토콜 메타데이터를 담은 `meta` 객체를 반드시 포함해야
한다(**MUST**).

<!-- ucp:example schema=shopping/cart op=create direction=request extract=$.params.arguments.cart -->
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

[기능 협상](../../overview/index.md#negotiation-protocol)을 위해 모든 요청에
`meta["ucp-agent"]` 필드가 **필수**다. 플랫폼은 추가 메타데이터 필드를 포함할 수
있다(**MAY**).

## 도구 { #tools }

UCP 기능은 MCP 도구와 1:1로 대응한다.

### 식별자 패턴 { #identifier-pattern }

MCP 도구는 리소스 식별과 페이로드 데이터를 분리한다.

* **요청:** 기존 장바구니에 대한 작업(`get`, `update`, `cancel`)에서는
    최상위 `id` 파라미터가 대상 리소스를 식별한다. 요청 페이로드의 `cart`
    객체에는 `id` 필드가 있어서는 안 된다(**MUST NOT**).
* **응답:** 모든 응답은 전체 리소스 상태의 일부로 `cart.id`를 포함한다.
* **생성:** `create_cart` 작업은 요청에 `id`가 필요 없으며, 응답에 새로 할당된
    `cart.id`가 포함된다.

| 도구 | 작업 | 설명 |
| :---- | :---- | :---- |
| `create_cart` | [장바구니 생성](index.md#create-cart) | 장바구니 세션을 생성한다. |
| `get_cart` | [장바구니 조회](index.md#get-cart) | 장바구니 세션을 조회한다. |
| `update_cart` | [장바구니 수정](index.md#update-cart) | 장바구니 세션을 수정한다. |
| `cancel_cart` | [장바구니 취소](index.md#cancel-cart) | 장바구니 세션을 취소한다. |

### `create_cart` { #create_cart }

[장바구니 생성](index.md#create-cart) 작업에 대응한다.

#### 입력 스키마 { #input-schema }

{{ schema_fields('cart_create_req', 'shopping/cart') }}

#### 출력 스키마 { #output-schema }

{{ schema_fields('cart_resp', 'shopping/cart') }}

#### 예시 { #example }

=== "요청"

    <!-- ucp:example schema=shopping/cart op=create direction=request extract=$.params.arguments.cart -->
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

=== "응답"

    <!-- ucp:example schema=shopping/cart op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.shopping.checkout": [{"version": "{{ ucp_version }}"}],
              "dev.ucp.shopping.cart": [{"version": "{{ ucp_version }}"}]
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

=== "에러 응답"

    모든 상품이 품절이라 장바구니 리소스가 생성되지 않는다.

    <!-- ucp:example schema=common/types/error_response op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
          "ucp": { "version": "{{ ucp_version }}", "status": "error" },
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

### `get_cart` { #get_cart }

[장바구니 조회](index.md#get-cart) 작업에 대응한다.

#### 입력 스키마 { #input-schema_1 }

* `id` (String, 필수): 장바구니 세션의 ID.

#### 출력 스키마 { #output-schema_1 }

{{ schema_fields('cart_resp', 'shopping/cart') }}

#### 예시 { #example_1 }

=== "요청"

    <!-- ucp:example schema=transports/mcp_tool_call def=request direction=request -->
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

=== "응답"

    <!-- ucp:example schema=shopping/cart op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.shopping.checkout": [{"version": "{{ ucp_version }}"}],
              "dev.ucp.shopping.cart": [{"version": "{{ ucp_version }}"}]
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

=== "찾을 수 없음"

    <!-- ucp:example schema=common/types/error_response op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "status": "error",
            "capabilities": {
              "dev.ucp.shopping.cart": [{"version": "{{ ucp_version }}"}]
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

### `update_cart` { #update_cart }

[장바구니 수정](index.md#update-cart) 작업에 대응한다.

#### 입력 스키마 { #input-schema_2 }

* `id` (String, 필수): 수정할 장바구니 세션의 ID.

{{ schema_fields('cart_update_req', 'shopping/cart') }}

#### 출력 스키마 { #output-schema_2 }

{{ schema_fields('cart_resp', 'shopping/cart') }}

#### 예시 { #example_2 }

=== "요청"

    <!-- ucp:example schema=transports/mcp_tool_call def=request direction=request -->
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

=== "응답"

    <!-- ucp:example schema=shopping/cart op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 2,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.shopping.checkout": [{"version": "{{ ucp_version }}"}],
              "dev.ucp.shopping.cart": [{"version": "{{ ucp_version }}"}]
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

### `cancel_cart` { #cancel_cart }

[장바구니 취소](index.md#cancel-cart) 작업에 대응한다.

#### 입력 스키마 { #input-schema_3 }

* `id` (String, 필수): 장바구니 세션의 ID.

#### 출력 스키마 { #output-schema_3 }

{{ schema_fields('cart_resp', 'shopping/cart') }}

#### 예시 { #example_3 }

=== "요청"

    <!-- ucp:example schema=transports/mcp_tool_call def=request direction=request -->
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

=== "응답"

    <!-- ucp:example schema=shopping/cart op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.shopping.checkout": [{"version": "{{ ucp_version }}"}],
              "dev.ucp.shopping.cart": [{"version": "{{ ucp_version }}"}]
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

## 오류 처리 { #error-handling }

UCP는 프로토콜 오류와 비즈니스 결과를 구분한다. 전체 오류 코드 레지스트리와 전송
바인딩 예시는 [핵심 명세](../../overview/index.md#error-handling)를 참고한다.

* **프로토콜 오류**: 요청 처리를 막는 전송 수준 실패(인증, 요청 한도 제한, 서비스
    불가). 코드 `-32000`(디스커버리 오류는 `-32001`)과 함께 JSON-RPC `error`로
    반환한다.
* **비즈니스 결과**: 요청 처리가 성공한 뒤의 애플리케이션 수준 결과. UCP 엔벌로프와
    `messages`를 담은 JSON-RPC `result`로 반환한다.

### 비즈니스 결과 { #business-outcomes }

비즈니스 결과(찾을 수 없음, 유효성 검사 오류 포함)는 UCP 엔벌로프와 `messages`를
담은 `structuredContent`가 있는 JSON-RPC `result`로 반환한다.

<!-- ucp:example schema=common/types/error_response op=read direction=response extract=$.result.structuredContent -->
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "{{ ucp_version }}",
        "status": "error",
        "capabilities": {
          "dev.ucp.shopping.cart": [{"version": "{{ ucp_version }}"}]
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

## 준수 { #conformance }

준수하는 MCP 전송 방식 구현은 반드시 다음을 지켜야 한다(**MUST**).

1. JSON-RPC 2.0 프로토콜을 올바르게 구현한다.
2. 이 명세에 정의된 모든 핵심 장바구니 도구를 제공한다.
3. [핵심 명세](../../overview/index.md#error-handling)에 따라 오류를 반환한다.
4. 비즈니스 결과를 UCP 엔벌로프와 `messages` 배열을 담은 JSON-RPC `result`로
    반환한다.
5. UCP 스키마에 따라 도구 입력을 검증한다.
6. 스트리밍을 지원하는 HTTP 전송 방식을 지원한다.
