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

# 체크아웃 기능 - MCP 바인딩 { #checkout-capability-mcp-binding }

이 문서는 [체크아웃 기능(Capability)](index.md)의 Model Context Protocol(MCP)
바인딩을 정의합니다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필로 MCP 전송 방식을 지원하는지
알립니다.

<!-- ucp:example schema=profile def=business_schema op=read direction=response -->
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
      "dev.ucp.shopping.fulfillment": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/fulfillment",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
          "extends": "dev.ucp.shopping.checkout"
        }
      ]
    },
    "payment_handlers": {
      "com.example.vendor.delegate_payment": [
        {
          "id": "handler_1",
          "version": "{{ ucp_version }}",
          "spec": "https://example.vendor.com/specs/delegate-payment",
          "schema": "https://example.vendor.com/schemas/delegate-payment-config.json",
          "available_instruments": [
            {"type": "card", "constraints": {"properties": {"brand": {"enum": ["visa", "mastercard"]}}}}
          ],
          "config": {}
        }
      ]
    }
  }
}
```

### 요청 메타데이터 { #request-metadata }

MCP 클라이언트는 모든 요청에 프로토콜 메타데이터를 담은 `meta` 객체를 반드시
포함해야 합니다(**MUST**).

<!-- ucp:example schema=shopping/checkout op=create direction=request extract=$.params.arguments.checkout -->
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "create_checkout",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/shopping-agent.json"
        },
        "idempotency-key": "550e8400-e29b-41d4-a716-446655440000"
      },
      "checkout": { "line_items": [ ... ] }
    }
  }
}
```

[기능 협상](../../overview/index.md#negotiation-protocol)을 위해 모든 요청에
`meta["ucp-agent"]` 필드가 **필수**입니다. `complete_checkout`과 `cancel_checkout`
작업에는 안전한 재시도를 위해 `meta["idempotency-key"]`도 있어야 합니다. 플랫폼은
메타데이터 필드를 추가로 포함할 수 있습니다(**MAY**).

## 도구 { #tools }

UCP 기능은 MCP 도구(Tool)와 1:1로 대응합니다.

### 식별자 패턴 { #identifier-pattern }

MCP 도구는 리소스 식별 정보와 페이로드 데이터를 분리합니다.

* **요청:** 기존 체크아웃에 대한 작업(`get`, `update`, `complete`, `cancel`)에서는
    최상위 `id` 파라미터로 대상 리소스를 식별합니다. 요청 페이로드의 `checkout`
    객체에 `id` 필드를 넣어서는 안 됩니다(**MUST NOT**).
* **응답:** 모든 응답은 전체 리소스 상태의 일부로 `checkout.id`를 포함합니다.
* **생성:** `create_checkout` 작업은 요청에 `id`가 필요 없으며, 응답에 새로 할당된 `checkout.id`가 들어 있습니다.

| 도구                | 작업                                          | 설명                |
| :------------------ | :------------------------------------------------- | :------------------------- |
| `create_checkout`   | [체크아웃 생성](index.md#create-checkout)        | 체크아웃 세션 생성 |
| `get_checkout`      | [체크아웃 조회](index.md#get-checkout)              | 체크아웃 세션 조회    |
| `update_checkout`   | [체크아웃 업데이트](index.md#update-checkout)        | 체크아웃 세션 업데이트 |
| `complete_checkout` | [체크아웃 완료](index.md#complete-checkout)    | 주문 확정           |
| `cancel_checkout`   | [체크아웃 취소](index.md#cancel-checkout)        | 체크아웃 세션 취소 |

### `create_checkout` { #create_checkout }

[체크아웃 생성](index.md#create-checkout) 작업에 대응합니다.

#### 입력 스키마 { #input-schema }

* `checkout` ([Checkout](index.md#create-checkout)): **필수**. 초기 체크아웃
    세션 데이터와 선택적 확장을 담습니다.
    * 확장(선택 사항):
        * `dev.ucp.shopping.buyer_consent`: [구매자 동의](../extensions/buyer-consent.md)
        * `dev.ucp.shopping.fulfillment`: [풀필먼트](../extensions/fulfillment.md)
        * `dev.ucp.shopping.discount`: [할인](../extensions/discount.md)
        * `dev.ucp.common.payment.ap2_mandate`: [AP2 위임](../../payment/extensions/ap2-mandates.md)

#### 출력 스키마 { #output-schema }

* [Checkout](index.md#create-checkout) 객체.

#### 예시 { #example }

=== "요청"

    <!-- ucp:example schema=shopping/checkout op=create direction=request extract=$.params.arguments.checkout -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "method": "tools/call",
      "params": {
        "name": "create_checkout",
        "arguments": {
          "meta": {
            "ucp-agent": {
              "profile": "https://platform.example/profiles/v2026-01/shopping-agent.json"
            }
          },
          "checkout": {
            "buyer": {
              "email": "jane.doe@example.com",
              "first_name": "Jane",
              "last_name": "Doe"
            },
            "line_items": [
              {
                "id": "li_1",
                "item": {
                  "id": "item_123"
                },
                "quantity": 1
              }
            ],
            "currency": "USD",
            "fulfillment": {
              "methods": [
                {
                  "type": "shipping",
                  "destinations": [
                    {
                      "street_address": "123 Main St",
                      "address_locality": "Springfield",
                      "address_region": "IL",
                      "postal_code": "62701",
                      "address_country": "US"
                    }
                  ]
                }
              ]
            }
          }
        }
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=shopping/checkout op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.shopping.checkout": [
                {"version": "{{ ucp_version }}"}
              ],
              "dev.ucp.shopping.fulfillment": [
                {"version": "{{ ucp_version }}"}
              ]
            },
            "payment_handlers": {
              "com.example.vendor.delegate_payment": [
                {"id": "handler_1", "version": "{{ ucp_version }}", "available_instruments": [{"type": "card"}], "config": {}}
              ]
            }
          },
          "id": "checkout_abc123",
          "status": "incomplete",
          "buyer": {
            "email": "jane.doe@example.com",
            "first_name": "Jane",
            "last_name": "Doe"
          },
          "line_items": [
            {
              "id": "li_1",
              "item": {
                "id": "item_123",
                "title": "Blue Jeans",
                "price": 5000
              },
              "quantity": 1,
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
              "type": "fulfillment",
              "display_text": "Shipping",
              "amount": 500
            },
            {
              "type": "total",
              "amount": 5500
            }
          ],
          "fulfillment": {
            "methods": [
              {
                "id": "shipping_1",
                "type": "shipping",
                "line_item_ids": ["li_1"],
                "selected_destination_id": "dest_home",
                "destinations": [
                  {
                    "type": "shipping_address",
                    "id": "dest_home",
                    "street_address": "123 Main St",
                    "address_locality": "Springfield",
                    "address_region": "IL",
                    "postal_code": "62701",
                    "address_country": "US"
                  }
                ],
                "groups": [
                  {
                    "id": "package_1",
                    "line_item_ids": ["li_1"],
                    "selected_option_id": "standard",
                    "options": [
                      {
                        "id": "standard",
                        "title": "Standard Shipping",
                        "description": { "plain": "Arrives in 5-7 business days" },
                        "totals": [
                          {
                            "type": "total",
                            "amount": 500
                          }
                        ]
                      },
                      {
                        "id": "express",
                        "title": "Express Shipping",
                        "description": { "plain": "Arrives in 2-3 business days" },
                        "totals": [
                          {
                            "type": "total",
                            "amount": 1000
                          }
                        ]
                      }
                    ]
                  }
                ]
              }
            ]
          },
          "links": [
            {
              "type": "privacy_policy",
              "url": "https://business.example.com/privacy"
            },
            {
              "type": "terms_of_service",
              "url": "https://business.example.com/terms"
            }
          ],
          "continue_url": "https://business.example.com/checkout-sessions/checkout_abc123",
          "expires_at": "2026-01-11T18:30:00Z"
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

    모든 상품의 재고가 없어 체크아웃 리소스가 생성되지 않는 경우입니다.

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

### `get_checkout` { #get_checkout }

[체크아웃 조회](index.md#get-checkout) 작업에 대응합니다.

#### 입력 스키마 { #input-schema_1 }

* `id` (String): **필수**. 체크아웃 세션의 ID.

#### 출력 스키마 { #output-schema_1 }

* [Checkout](index.md#get-checkout) 객체.

### `update_checkout` { #update_checkout }

[체크아웃 업데이트](index.md#update-checkout) 작업에 대응합니다.

체크아웃이 `complete_in_progress` 상태인 동안 플랫폼은 새 `update_checkout` 작업을
시작해서는 안 됩니다(**MUST NOT**). 중복 요청에는 이때도
[재전송 방지](../../signatures.md#replay-protection) 규칙이 적용됩니다. 이 상태에서
새 `update_checkout` 요청을 받으면 비즈니스는 체크아웃을 변경하지 않고, 복구 가능한
오류 메시지와 함께 현재 체크아웃을 반드시 반환해야 합니다(**MUST**).

#### 입력 스키마 { #input-schema_2 }

* `id` (String): **필수**. 업데이트할 체크아웃 세션의 ID.
* `checkout` ([Checkout](index.md#update-checkout)): **필수**.
    업데이트된 체크아웃 세션 데이터를 담습니다.
    * 확장(선택 사항):
        * `dev.ucp.shopping.buyer_consent`: [구매자 동의](../extensions/buyer-consent.md)
        * `dev.ucp.shopping.fulfillment`: [풀필먼트](../extensions/fulfillment.md)
        * `dev.ucp.shopping.discount`: [할인](../extensions/discount.md)
        * `dev.ucp.common.payment.ap2_mandate`: [AP2 위임](../../payment/extensions/ap2-mandates.md)

#### 출력 스키마 { #output-schema_2 }

* [Checkout](index.md#update-checkout) 객체.

#### 예시 { #example_1 }

=== "요청"

    <!-- ucp:example schema=shopping/checkout op=update direction=request extract=$.params.arguments.checkout -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 2,
      "method": "tools/call",
      "params": {
        "name": "update_checkout",
        "arguments": {
          "meta": {
            "ucp-agent": {
              "profile": "https://platform.example/profiles/v2026-01/shopping-agent.json"
            }
          },
          "id": "checkout_abc123",
          "checkout": {
            "buyer": {
              "email": "jane.doe@example.com",
              "first_name": "Jane",
              "last_name": "Doe"
            },
            "line_items": [
              {
                "item": {
                  "id": "item_123"
                },
                "id": "li_1",
                "quantity": 1
              }
            ],
            "currency": "USD",
            "fulfillment": {
              "methods": [
                {
                  "id": "shipping_1",
                  "line_item_ids": ["li_1"],
                  "groups": [
                    {
                      "id": "package_1",
                      "selected_option_id": "express"
                    }
                  ]
                }
              ]
            }
          }
        }
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=shopping/checkout op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 2,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.shopping.checkout": [
                {"version": "{{ ucp_version }}"}
              ],
              "dev.ucp.shopping.fulfillment": [
                {"version": "{{ ucp_version }}"}
              ]
            },
            "payment_handlers": {
              "com.example.vendor.delegate_payment": [
                {"id": "handler_1", "version": "{{ ucp_version }}", "available_instruments": [{"type": "card"}], "config": {}}
              ]
            }
          },
          "id": "checkout_abc123",
          "status": "incomplete",
          "buyer": {
            "email": "jane.doe@example.com",
            "first_name": "Jane",
            "last_name": "Doe"
          },
          "line_items": [
            {
              "id": "li_1",
              "item": {
                "id": "item_123",
                "title": "Blue Jeans",
                "price": 5000
              },
              "quantity": 1,
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
              "type": "fulfillment",
              "display_text": "Shipping",
              "amount": 1000
            },
            {
              "type": "total",
              "amount": 6000
            }
          ],
          "fulfillment": {
            "methods": [
              {
                "id": "shipping_1",
                "type": "shipping",
                "line_item_ids": ["li_1"],
                "selected_destination_id": "dest_home",
                "destinations": [
                  {
                    "type": "shipping_address",
                    "id": "dest_home",
                    "street_address": "123 Main St",
                    "address_locality": "Springfield",
                    "address_region": "IL",
                    "postal_code": "62701",
                    "address_country": "US"
                  }
                ],
                "groups": [
                  {
                    "id": "package_1",
                    "line_item_ids": ["li_1"],
                    "selected_option_id": "express",
                    "options": [
                      {
                        "id": "standard",
                        "title": "Standard Shipping",
                        "description": { "plain": "Arrives in 5-7 business days" },
                        "totals": [
                          {
                            "type": "total",
                            "amount": 500
                          }
                        ]
                      },
                      {
                        "id": "express",
                        "title": "Express Shipping",
                        "description": { "plain": "Arrives in 2-3 business days" },
                        "totals": [
                          {
                            "type": "total",
                            "amount": 1000
                          }
                        ]
                      }
                    ]
                  }
                ]
              }
            ]
          },
          "links": [
            {
              "type": "privacy_policy",
              "url": "https://business.example.com/privacy"
            },
            {
              "type": "terms_of_service",
              "url": "https://business.example.com/terms"
            }
          ],
          "continue_url": "https://business.example.com/checkout-sessions/checkout_abc123",
          "expires_at": "2026-01-11T18:30:00Z"
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

### `complete_checkout` { #complete_checkout }

[체크아웃 완료](index.md#complete-checkout) 작업에 대응합니다.

#### 입력 스키마 { #input-schema_3 }

* `meta` (Object): **필수**. 다음을 담은 요청 메타데이터:
    * `ucp-agent` (Object): **필수**. 플랫폼 에이전트 식별 정보.
    * `idempotency-key` (String, UUID): **필수**. 안전한 재시도를 위한 고유 키.
* `id` (String): **필수**. 체크아웃 세션의 ID.
* `checkout` ([Checkout](index.md#complete-checkout)): **필수**.
    거래 실행에 필요한 결제 자격 증명과 기타 최종 확정 데이터를 담습니다.
    * 확장(선택 사항):
        * `dev.ucp.shopping.buyer_consent`: [구매자 동의](../extensions/buyer-consent.md)
        * `dev.ucp.common.payment.ap2_mandate`: [AP2 위임](../../payment/extensions/ap2-mandates.md)

#### 출력 스키마 { #output-schema_3 }

* [Checkout](index.md#complete-checkout) 객체. 상태와 `order`의 의미는 핵심
   명세의 [체크아웃 완료](index.md#complete-checkout)를 참고하세요.

### `cancel_checkout` { #cancel_checkout }

[체크아웃 취소](index.md#cancel-checkout) 작업에 대응합니다.

#### 입력 스키마 { #input-schema_4 }

* `meta` (Object): **필수**. 다음을 담은 요청 메타데이터:
    * `ucp-agent` (Object): **필수**. 플랫폼 에이전트 식별 정보.
    * `idempotency-key` (String, UUID): **필수**. 안전한 재시도를 위한 고유 키.
* `id` (String): **필수**. 체크아웃 세션의 ID.

#### 출력 스키마 { #output-schema_4 }

* `status: canceled`인 [Checkout](index.md#cancel-checkout) 객체.

## 오류 처리 { #error-handling }

UCP는 프로토콜 오류와 비즈니스 결과를 구분합니다. 전체 오류 코드 레지스트리와 전송
바인딩 예시는 [핵심 명세](../../overview/index.md#error-handling)를 참고하세요.

* **프로토콜 오류**: 요청 처리를 가로막는 전송 계층 수준의 실패(인증, 속도 제한,
    서비스 불가)입니다. 코드 `-32000`(디스커버리 오류는 `-32001`)과 함께 JSON-RPC
    `error`로 반환됩니다.
* **비즈니스 결과**: 요청이 정상적으로 처리된 뒤의 애플리케이션 수준 결과입니다.
    UCP 엔벨로프와 `messages`를 담은 JSON-RPC `result`로 반환됩니다.

### 비즈니스 결과 { #business-outcomes }

비즈니스 결과(구매할 수 없는 상품 같은 오류 포함)는 JSON-RPC `result`로
반환되며, 그 `structuredContent`에 UCP 엔벨로프와 `messages`가 담깁니다.

<!-- ucp:example schema=shopping/checkout op=read direction=response extract=$.result.structuredContent -->
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "{{ ucp_version }}",
        "payment_handlers": {},
        "capabilities": {
          "dev.ucp.shopping.checkout": [{"version": "{{ ucp_version }}"}]
        }
      },
      "id": "checkout_abc123",
      "status": "incomplete",
      "line_items": [
        {
          "id": "li_1",
           "item": {
              "id": "item_123",
              "title": "Blue Jeans",
              "price": 5000
            },
          "quantity": 12,
          "totals": [...]
        }
      ],
      "totals": [...],
      "currency": "USD",
      "links": [],
      "messages": [
        {
          "type": "warning",
          "code": "quantity_adjusted",
          "content": "Quantity adjusted, requested 100 units but only 12 available",
          "path": "$.line_items[0].quantity"
        }
      ],
      "continue_url": "https://merchant.com/checkout/checkout_abc123"
    },
    "content": [
      {"type": "text", "text": "{\"ucp\":{…},…}"}
    ]
  }
}
```

`create_checkout`에서 모든 상품을 구매할 수 없어 체크아웃을 생성하지 못하는
경우에도 결과는 JSON-RPC `result`로 반환되며, 그 `structuredContent`에 UCP
엔벨로프와 `messages`가 담깁니다.

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
          "code": "item_unavailable",
          "content": "Items are not available for purchase in your region",
          "severity": "unrecoverable",
          "path": "$.line_items"
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

## 메시지 서명 { #message-signing }

MCP 전송 방식을 사용할 때 플랫폼은 에이전트를 인증해야 합니다(**SHOULD**). HTTP
메시지 서명을 사용한다면 모든 체크아웃 작업은
[메시지 서명](../../signatures.md) 명세를 따릅니다.

### 요청 서명 { #request-signing }

UCP의 MCP 전송 방식은 **스트리밍 가능한 HTTP**(streamable HTTP)를 사용하므로
REST와 같은 RFC 9421 서명 메커니즘을 쓸 수 있습니다. 서명은 HTTP 계층에 적용됩니다.

| 헤더                   | 필수 | 설명                              |
| :----------------------- | :------- | :--------------------------------------- |
| `Signature-Input`        | 예      | 서명된 구성 요소 기술              |
| `Signature`              | 예      | 서명 값             |
| `Content-Digest`         | 예      | 요청 본문의 SHA-256 해시             |
| `UCP-Agent`              | 예      | 서명자 신원(프로필 URL)            |
| `Idempotency-Key`        | 조건부*   | 재전송 방지를 위한 고유 키         |

\* `complete_checkout`과 `cancel_checkout`에서는 필수

**서명된 요청 예시:**

```http
POST /mcp HTTP/1.1
Host: business.example.com
Content-Type: application/json
UCP-Agent: profile="https://platform.example/.well-known/ucp"
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Digest: sha-256=:RK/0qy18MlBSVnWgjwz6lZEWjP/lF5HF9bvEF8FabDg=:
Signature-Input: sig1=("@method" "@authority" "@path" "content-digest" "content-type" "ucp-agent" "idempotency-key");keyid="platform-2026"
Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:

{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"complete_checkout","arguments":{"id":"checkout_abc123","checkout":{"payment":{...}}}}}
```

`Content-Digest`가 JSON-RPC 본문을 서명에 결합합니다. JSON 정규화(canonicalization)는
필요하지 않습니다.

자세한 내용은 [메시지 서명 - MCP 전송 방식](../../signatures.md#mcp-transport)을
참고하세요.

### 응답 서명 { #response-signing }

다음 응답에는 서명을 권장합니다(**RECOMMENDED**).

* `complete_checkout` 응답(주문 확정)

다음 응답의 서명은 선택 사항입니다(**OPTIONAL**).

* `create_checkout`, `get_checkout`, `update_checkout`, `cancel_checkout`

**서명된 응답 예시:**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Digest: sha-256=:Y5fK8nLmPqRsT3vWxYzAbCdEfGhIjKlMnO...:
Signature-Input: sig1=("@status" "content-digest" "content-type");keyid="merchant-2026"
Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:

{"jsonrpc":"2.0","id":1,"result":{"content":[{"type":"text","text":"..."}],"structuredContent":{"id":"checkout_abc123","status":"completed"}}}
```

서명 알고리즘은 [메시지 서명 - REST 응답 서명](../../signatures.md#rest-response-signing)을
참고하세요(HTTP 기반 MCP에도 똑같이 적용됩니다).

## 적합성 { #conformance }

적합한 MCP 전송 방식 구현은 반드시 다음을 충족해야 합니다(**MUST**).

1. JSON-RPC 2.0 프로토콜을 올바르게 구현합니다.
2. 이 명세에 정의된 핵심 체크아웃 도구를 모두 제공합니다.
3. [핵심 명세](../../overview/index.md#error-handling)에 따라 오류를 반환합니다.
4. 비즈니스 결과를 UCP 엔벨로프와 `messages` 배열을 담은 JSON-RPC `result`로
    반환합니다.
5. 도구 입력을 UCP 스키마로 검증합니다.
6. 스트리밍을 지원하는 HTTP 전송 방식을 제공합니다.

적합한 구현은 다음을 충족해야 합니다(**SHOULD**).

1. 지원되는 메커니즘(API 키, OAuth, mTLS, 또는
    [메시지 서명](../../signatures.md)에 따른 HTTP 메시지 서명) 중 하나로 에이전트를 인증합니다.
2. 들어오는 요청을 처리하기 전에 인증을 검증합니다.

## 구현 { #implementation }

UCP 작업은 [OpenRPC](https://open-rpc.org/)(JSON-RPC 스키마 형식)로 정의되어
있습니다. [MCP 명세](https://modelcontextprotocol.io/)에 따르면 모든 도구 호출은
`tools/call` 메서드를 사용하고, 작업 이름과 인자를 `params`로 감싸야 합니다.
구현자는 반드시 다음 변환을 적용해야 합니다(**MUST**).

| OpenRPC  | MCP                |
|:---------|:-------------------|
| `method` | `params.name`      |
| `params` | `params.arguments` |

**파라미터 규칙:**

* `meta`: 요청 메타데이터를 담습니다.
* `id`: 대상 리소스를 식별합니다(경로 파라미터에 해당).
* `checkout`: 도메인 페이로드를 담습니다(본문에 해당).

**예시:** OpenRPC에 다음과 같이 정의된 `complete_checkout` 작업이 있다고 해 봅시다.

<!-- ucp:example schema=shopping/checkout op=complete direction=request extract=$.params.checkout -->
```json
{
  "method": "complete_checkout",
  "params": {
    "meta": {
      "ucp-agent": { "profile": "https://..." },
      "idempotency-key": "550e8400-e29b-41d4-a716-446655440000"
    },
    "id": "checkout_abc123",
    "checkout": { "payment": {...} }
  }
}
```

구현자는 이 작업을 반드시 MCP `tools/call` 엔드포인트로 노출해야 합니다(**MUST**).

<!-- ucp:example schema=shopping/checkout op=complete direction=request extract=$.params.arguments.checkout -->
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "complete_checkout",
    "arguments": {
      "meta": {
        "ucp-agent": { "profile": "https://..." },
        "idempotency-key": "550e8400-e29b-41d4-a716-446655440000"
      },
      "id": "checkout_abc123",
      "checkout": { "payment": {...} }
    }
  }
}
```
