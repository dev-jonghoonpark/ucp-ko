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

# 체크아웃 기능 - REST 바인딩 { #checkout-capability-rest-binding }

이 문서는 [체크아웃 기능(Checkout Capability)](index.md)의 REST 바인딩을 정의합니다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필로 REST 전송 방식 지원 여부를
알립니다.

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
          "endpoint": "https://business.example.com/ucp/v1"
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
      ]
    },
    "payment_handlers": {}
  }
}
```

### 기본 URL { #base-url }

모든 UCP REST 엔드포인트는 비즈니스의 기본 URL에 대한 상대 경로입니다. 기본 URL은
`/.well-known/ucp`의 UCP 프로필에서 찾습니다. 체크아웃 기능의 엔드포인트는 비즈니스
프로필의 `rest.endpoint` 필드에 정의되어 있습니다.

### 콘텐츠 유형 { #content-types }

* **요청**: `application/json`
* **응답**: `application/json`

모든 요청 및 응답 본문은 반드시
[RFC 8259](https://tools.ietf.org/html/rfc8259){ target="_blank" }에 정의된 유효한
JSON이어야 합니다(**MUST**).

### 전송 보안 { #transport-security }

모든 REST 엔드포인트는 반드시 TLS 1.3 이상의 HTTPS로 제공해야 합니다(**MUST**).

## 작업 { #operations }

| 작업                                          | 메서드 | 엔드포인트                           | 설명                |
| :------------------------------------------------- | :----- | :--------------------------------- | :------------------------- |
| [체크아웃 생성](index.md#create-checkout)        | `POST` | `/checkout-sessions`               | 체크아웃 세션을 생성합니다. |
| [체크아웃 조회](index.md#get-checkout)              | `GET`  | `/checkout-sessions/{id}`          | 체크아웃 세션을 조회합니다.    |
| [체크아웃 갱신](index.md#update-checkout)        | `PUT`  | `/checkout-sessions/{id}`          | 체크아웃 세션을 갱신합니다. |
| [체크아웃 완료](index.md#complete-checkout)    | `POST` | `/checkout-sessions/{id}/complete` | 주문을 확정합니다.           |
| [체크아웃 취소](index.md#cancel-checkout)        | `POST` | `/checkout-sessions/{id}/cancel`   | 체크아웃 세션을 취소합니다. |

## 예시 { #examples }

### 체크아웃 생성 { #create-checkout }

=== "요청"

    <!-- ucp:example schema=shopping/checkout op=create direction=request -->
    ```json
    POST /checkout-sessions HTTP/1.1
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json

    {
      "line_items": [
        {
          "item": {
            "id": "item_123"
          },
          "quantity": 2
        }
      ]
    }
    ```

=== "응답"

    <!-- ucp:example schema=shopping/checkout op=read -->
    ```json
    HTTP/1.1 201 Created
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.shopping.checkout": [
            {"version": "{{ ucp_version }}"}
          ]
        },
        "payment_handlers": {
          "com.shopify.shop_pay": [
            {
              "id": "shop_pay_1234",
              "version": "{{ ucp_version }}",
              "available_instruments": [
                {"type": "shop_pay"}
              ],
              "config": {
                "merchant_id": "shop_merchant_123"
              }
            }
          ]
        }
      },
      "id": "chk_1234567890",
      "status": "incomplete",
      "messages": [
        {
          "type": "error",
          "code": "missing",
          "path": "$.buyer.email",
          "content": "Buyer email is required",
          "severity": "recoverable"
        }
      ],
      "currency": "USD",
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
      "totals": [
        {
          "type": "subtotal",
          "amount": 5000
        },
        {
          "type": "tax",
          "amount": 400
        },
        {
          "type": "total",
          "amount": 5400
        }
      ],
      "links": [
        {
          "type": "terms_of_service",
          "url": "https://business.example.com/terms"
        }
      ],
      "payment": {
        "instruments": [
          {
            "id": "instr_shop_pay_1",
            "handler_id": "shop_pay_1234",
            "type": "shop_pay",
            "selected": true,
            "display": {
              "email": "buyer@example.com"
            }
          }
        ]
      }
    }
    ```

=== "에러 응답"

    모든 상품이 품절인 경우입니다. 체크아웃 리소스는 생성되지 않습니다.

    <!-- ucp:example schema=common/types/error_response op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": { "version": "2026-01-11", "status": "error" },
      "messages": [
        {
          "type": "error",
          "code": "out_of_stock",
          "content": "All requested items are currently out of stock",
          "severity": "unrecoverable"
        }
      ],
      "continue_url": "https://merchant.com/"
    }
    ```

### 체크아웃 갱신 { #update-checkout }

체크아웃 갱신은 전체 교체 작업입니다. 플랫폼은 쓰기 전용 필드의 데이터 갱신까지 포함해
체크아웃 리소스 전체를 반드시 보내야 하며(**MUST**), 보낸 리소스가 기존 체크아웃 세션
상태를 대체합니다. 체크아웃이 `complete_in_progress` 상태인 동안 플랫폼은 새 체크아웃 갱신
작업을 시작해서는 안 됩니다(**MUST NOT**). 중복 요청에는 여전히
[재전송 방지](../../signatures.md#replay-protection)가 적용됩니다. 이 상태에서 새 체크아웃
갱신 요청을 받으면 비즈니스는 반드시 체크아웃을 변경하지 않고, 복구 가능한 오류
메시지와 함께 현재 체크아웃을 반환해야 합니다(**MUST**).

#### 구매자 정보 갱신 { #update-buyer-info }

`buyer`의 모든 필드는 선택 사항이므로, 플랫폼은 여러 번 호출하면서 체크아웃 상태를
점진적으로 채워 나갈 수 있습니다. `complete_in_progress` 상태가 아닐 때 체크아웃 갱신은 매번
체크아웃 세션 전체를 대체하므로, 플랫폼은 유지하려는 기존 필드를 반드시 모두
포함해야 합니다(**MUST**).

=== "요청"

    <!-- ucp:example schema=shopping/checkout op=update direction=request -->
    ```json
    PUT /checkout-sessions/{id} HTTP/1.1
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json

    {
      "buyer": {
        "email": "jane@example.com",
        "first_name": "Jane",
        "last_name": "Doe"
      },
      "line_items": [
        {
          "item": {
            "id": "item_123"
          },
          "id": "li_1",
          "quantity": 2
        }
      ]
    }
    ```

=== "응답"

    <!-- ucp:example schema=shopping/checkout op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.shopping.checkout": [
            {"version": "{{ ucp_version }}"}
          ]
        },
        "payment_handlers": {
          "com.shopify.shop_pay": [
            {
              "id": "shop_pay_1234",
              "version": "{{ ucp_version }}",
              "available_instruments": [
                {"type": "shop_pay"}
              ],
              "config": {
                "merchant_id": "shop_merchant_123"
              }
            }
          ]
        }
      },
      "id": "chk_1234567890",
      "status": "incomplete",
      "messages": [
        {
          "type": "error",
          "code": "missing",
          "path": "$.fulfillment.methods[0].selected_destination_id",
          "content": "Fulfillment address is required",
          "severity": "recoverable"
        }
      ],
      "currency": "USD",
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
      "buyer": {
        "email": "jane@example.com",
        "first_name": "Jane",
        "last_name": "Doe"
      },
      "totals": [
        {
          "type": "subtotal",
          "amount": 5000
        },
        {
          "type": "tax",
          "amount": 400
        },
        {
          "type": "total",
          "amount": 5400
        }
      ],
      "links": [
        {
          "type": "terms_of_service",
          "url": "https://business.example.com/terms"
        }
      ],
      "payment": {
        "instruments": [
          {
            "id": "instr_shop_pay_1",
            "handler_id": "shop_pay_1234",
            "type": "shop_pay",
            "selected": true,
            "display": {
              "email": "buyer@example.com"
            }
          }
        ]
      }
    }
    ```

#### 풀필먼트 갱신 { #update-fulfillment }

풀필먼트는 체크아웃 기능의 확장입니다. 플랫폼은 원하는 풀필먼트 방식 유형, 배송지 주소,
비즈니스 매장 위치 선택 같은 구매자 입력을 제출하고, 비즈니스는 그에 따라 정해진 풀필먼트
상태를 반환합니다.

=== "요청"

    <!-- ucp:example schema=shopping/checkout op=update direction=request -->
    ```json
    PUT /checkout-sessions/{id} HTTP/1.1
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json

    {
      "buyer": {
        "email": "jane@example.com",
        "first_name": "Jane",
        "last_name": "Doe"
      },
      "line_items": [
        {
          "item": {
            "id": "item_123"
          },
          "id": "li_1",
          "quantity": 2
        }
      ],
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
    ```

=== "응답"

    <!-- ucp:example schema=shopping/checkout op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.shopping.checkout": [
            {"version": "{{ ucp_version }}"}
          ]
        },
        "payment_handlers": {
          "com.google.pay": [
            {
              "id": "gpay_1234",
              "version": "{{ ucp_version }}",
              "config": {
                "allowed_payment_methods": [
                  {
                    "type": "CARD",
                    "parameters": {
                      "allowed_card_networks": ["VISA", "MASTERCARD", "AMEX"]
                    }
                  }
                ]
              }
            }
          ]
        }
      },
      "id": "chk_1234567890",
      "status": "incomplete",
      "messages": [
        {
          "type": "error",
          "code": "missing",
          "path": "$.selected_fulfillment_option",
          "content": "Please select a fulfillment option",
          "severity": "recoverable"
        }
      ],
      "currency": "USD",
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
      "buyer": {
        "email": "jane@example.com",
        "first_name": "Jane",
        "last_name": "Doe"
      },
      "totals": [
        {
          "type": "subtotal",
          "amount": 5000
        },
        {
          "type": "tax",
          "amount": 400
        },
        {
          "type": "total",
          "amount": 5400
        }
      ],
      "links": [
        {
          "type": "terms_of_service",
          "url": "https://merchant.com/terms"
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
                    "description": "Arrives in 5-7 business days",
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
                    "description": "Arrives in 2-3 business days",
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
      "payment": {
        "instruments": [
          {
            "id": "pi_gpay_5678",
            "handler_id": "gpay_1234",
            "type": "card",
            "selected": true,
            "display": {
              "brand": "mastercard",
              "last_digits": "5678",
              "rich_text_description": "Google Pay •••• 5678"
            }
          }
        ]
      }
    }
    ```

#### 풀필먼트 선택 갱신 { #update-fulfillment-selection }

최초 `fulfillment` 데이터를 보낸 뒤 선택을 바꿀 때 쓰는 후속 호출입니다.

=== "요청"

    <!-- ucp:example schema=shopping/checkout op=update direction=request -->
    ```json
    PUT /checkout-sessions/{id} HTTP/1.1
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json

    {
      "buyer": {
        "email": "jane@example.com",
        "first_name": "Jane",
        "last_name": "Doe"
      },
      "line_items": [
        {
          "item": {
            "id": "item_123"
          },
          "id": "li_1",
          "quantity": 2
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
                "selected_option_id": "express"
              }
            ]
          }
        ]
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=shopping/checkout op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.shopping.checkout": [
            {"version": "{{ ucp_version }}"}
          ]
        },
        "payment_handlers": {
          "com.shopify.shop_pay": [
            {
              "id": "shop_pay_1234",
              "version": "{{ ucp_version }}",
              "available_instruments": [
                {"type": "shop_pay"}
              ],
              "config": {
                "merchant_id": "shop_merchant_123"
              }
            }
          ]
        }
      },
      "id": "chk_1234567890",
      "status": "ready_for_complete",
      "currency": "USD",
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
      "buyer": {
        "email": "jane@example.com",
        "first_name": "Jane",
        "last_name": "Doe"
      },
      "totals": [
        {
          "type": "subtotal",
          "amount": 5000
        },
        {
          "type": "tax",
          "amount": 400
        },
        {
          "type": "total",
          "amount": 5400
        }
      ],
      "links": [
        {
          "type": "terms_of_service",
          "url": "https://merchant.com/terms"
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
                    "description": "Arrives in 5-7 business days",
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
                    "description": "Arrives in 2-3 business days",
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
      "payment": {
        "instruments": [
          {
            "id": "instr_shop_pay_1",
            "handler_id": "shop_pay_1234",
            "type": "shop_pay",
            "selected": true,
            "display": {
              "email": "buyer@example.com"
            }
          }
        ]
      }
    }
    ```

### 체크아웃 완료 { #complete-checkout }

비즈니스에 `buyer`와 주소(예: `fulfillment_address`, `billing_address`) 필드가 반드시 있어야
한다는 로직이 있다면, 이 단계에서 `messages`로 그 요구 사항을 알리는 것이
적절합니다.

응답은 체크아웃 객체이며, 아래 예시는 동기식 `completed` 사례입니다. 상태와 `order`의
의미는 핵심 명세의 [체크아웃 완료](index.md#complete-checkout)를 참고하세요.

=== "요청"

    <!-- ucp:example schema=shopping/checkout op=complete direction=request -->
    ```json
    POST /checkout-sessions/{id}/complete
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json

    {
      "payment": {
        "instruments": [
          {
            "id": "pi_gpay_5678",
            "handler_id": "gpay_1234",
            "type": "card",
            "selected": true,
            "display": {
              "brand": "mastercard",
              "last_digits": "5678",
              "card_art": "https://cart-art-1.html",
              "description": "Google Pay •••• 5678"
            },
            "billing_address": {
              "street_address": "123 Main St",
              "address_locality": "Anytown",
              "address_region": "CA",
              "address_country": "US",
              "postal_code": "12345"
            },
            "credential": {
              "type": "PAYMENT_GATEWAY",
              "token": "examplePaymentMethodToken"
            }
          }
        ]
      },
      "signals": {
        "dev.ucp.buyer_ip": "203.0.113.42",
        "dev.ucp.user_agent": "Mozilla/5.0 ..."
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=shopping/checkout op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.shopping.checkout": [
            {"version": "{{ ucp_version }}"}
          ]
        },
        "payment_handlers": {
          "com.google.pay": [
            {
              "id": "gpay_1234",
              "version": "{{ ucp_version }}",
              "config": {
                "allowed_payment_methods": [
                  {
                    "type": "CARD",
                    "parameters": {
                      "allowed_card_networks": ["VISA", "MASTERCARD", "AMEX"]
                    }
                  }
                ]
              }
            }
          ]
        }
      },
      "id": "chk_123456789",
      "status": "completed",
      "currency": "USD",
      "order": {
        "id": "ord_99887766",
        "permalink_url": "https://merchant.com/orders/ord_99887766"
      },
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
      "buyer": {
        "email": "jane@example.com",
        "first_name": "Jane",
        "last_name": "Doe"
      },
      "totals": [
        {
          "type": "subtotal",
          "amount": 5000
        },
        {
          "type": "tax",
          "amount": 400
        },
        {
          "type": "total",
          "amount": 5400
        }
      ],
      "links": [
        {
          "type": "terms_of_service",
          "url": "https://merchant.com/terms"
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
                    "description": "Arrives in 5-7 business days",
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
                    "description": "Arrives in 2-3 business days",
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
      "payment": {
        "instruments": [
          {
            "id": "pi_gpay_5678",
            "handler_id": "gpay_1234",
            "type": "card",
            "selected": true,
            "display": {
              "brand": "mastercard",
              "last_digits": "5678",
              "rich_text_description": "Google Pay •••• 5678"
            }
          }
        ]
      }
    }
    ```

### 체크아웃 조회 { #get-checkout }

=== "요청"

    <!-- ucp:example schema=shopping/checkout op=read direction=request -->
    ```json
    GET /checkout-sessions/{id}
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json

    {}
    ```

=== "응답"

    <!-- ucp:example schema=shopping/checkout op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.shopping.checkout": [
            {"version": "{{ ucp_version }}"}
          ]
        },
        "payment_handlers": {
          "com.shopify.shop_pay": [
            {
              "id": "shop_pay_1234",
              "version": "{{ ucp_version }}",
              "available_instruments": [
                {"type": "shop_pay"}
              ],
              "config": {
                "merchant_id": "shop_merchant_123"
              }
            }
          ]
        }
      },
      "id": "chk_123456789",
      "status": "completed",
      "currency": "USD",
      "order": {
        "id": "ord_99887766",
        "permalink_url": "https://merchant.com/orders/ord_99887766"
      },
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
      "buyer": {
        "email": "jane@example.com",
        "first_name": "Jane",
        "last_name": "Doe"
      },
      "totals": [
        {
          "type": "subtotal",
          "amount": 5000
        },
        {
          "type": "tax",
          "amount": 400
        },
        {
          "type": "total",
          "amount": 5400
        }
      ],
      "links": [
        {
          "type": "terms_of_service",
          "url": "https://merchant.com/terms"
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
                    "description": "Arrives in 5-7 business days",
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
                    "description": "Arrives in 2-3 business days",
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
      "payment": {
        "instruments": [
          {
            "id": "instr_shop_pay_1",
            "handler_id": "shop_pay_1234",
            "type": "shop_pay",
            "selected": true,
            "display": {
              "email": "buyer@example.com"
            }
          }
        ]
      }
    }
    ```

### 체크아웃 취소 { #cancel-checkout }

=== "요청"

    <!-- ucp:example schema=shopping/checkout op=cancel direction=request -->
    ```json
    POST /checkout-sessions/{id}/cancel
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json

    {}
    ```

=== "응답"

    <!-- ucp:example schema=shopping/checkout op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.shopping.checkout": [
            {"version": "{{ ucp_version }}"}
          ]
        },
        "payment_handlers": {
          "com.google.pay": [
            {
              "id": "gpay_1234",
              "version": "{{ ucp_version }}",
              "config": {
                "allowed_payment_methods": [
                  {
                    "type": "CARD",
                    "parameters": {
                      "allowed_card_networks": ["VISA", "MASTERCARD", "AMEX"]
                    }
                  }
                ]
              }
            }
          ]
        }
      },
      "id": "chk_123456789",
      "status": "canceled", // Status is updated to canceled.
      "currency": "USD",
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
      "buyer": {
        "email": "jane@example.com",
        "first_name": "Jane",
        "last_name": "Doe"
      },
      "totals": [
        {
          "type": "subtotal",
          "amount": 5000
        },
        {
          "type": "tax",
          "amount": 400
        },
        {
          "type": "total",
          "amount": 5400
        }
      ],
      "links": [
        {
          "type": "terms_of_service",
          "url": "https://merchant.com/terms"
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
                    "description": "Arrives in 5-7 business days",
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
                    "description": "Arrives in 2-3 business days",
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
      "payment": {
        "instruments": [
          {
            "id": "pi_gpay_5678",
            "handler_id": "gpay_1234",
            "type": "card",
            "selected": true,
            "display": {
              "brand": "mastercard",
              "last_digits": "5678",
              "rich_text_description": "Google Pay •••• 5678"
            }
          }
        ]
      }
    }
    ```

## HTTP 헤더 { #http-headers }

다음 헤더는 HTTP 바인딩에 정의되어 있으며, 별도로 명시하지 않는 한 모든 작업에 적용됩니다.

{{ header_fields('create_checkout', 'shopping/rest.openapi.json') }}

### 헤더별 요구 사항 { #specific-header-requirements }

* **UCP-Agent**: 모든 요청은 반드시 Dictionary Structured Field 문법
    ([RFC 8941](https://datatracker.ietf.org/doc/html/rfc8941){target="_blank"})으로 플랫폼
    프로필 URI를 담은 `UCP-Agent` 헤더를 포함해야 합니다(**MUST**).
    형식: `profile="https://platform.example/profile"`.
* **Idempotency-Key**: 상태를 변경하는 작업은 멱등성을 지원해야
    합니다(**SHOULD**). 이 헤더가 제공되면 서버는 반드시 다음을 지켜야 합니다(**MUST**).
    1. 키를 작업 결과와 함께 최소 24시간 동안 저장합니다.
    2. 요청 본문이 원래 요청과 일치하는 중복 키에는 캐시된 결과를 반환합니다.
    3. 본문이 다른데 키가 재사용되면 `409 Conflict`를 반환합니다.
    페이로드 일치 규약 전체는 [메시지 서명 — 멱등성 키 요구 사항](../../signatures.md#replay-protection)을
    참고하세요.

## 프로토콜 동작 방식 { #protocol-mechanics }

### 상태 코드 { #status-codes }

UCP는 API 요청의 성공이나 실패를 표준 HTTP 상태 코드로 나타냅니다.

| 상태 코드                 | 설명                                                                        |
| :-------------------------- | :--------------------------------------------------------------------------------- |
| `200 OK`                    | 요청이 성공했습니다.                                                        |
| `201 Created`               | 리소스가 생성되었습니다.                                             |
| `400 Bad Request`           | 요청이 잘못되었거나 처리할 수 없습니다.                                       |
| `401 Unauthorized`          | 인증이 필요한데 실패했거나 인증 정보가 없습니다.                |
| `403 Forbidden`             | 요청은 인증되었지만 사용자에게 필요한 권한이 없습니다. |
| `409 Conflict`              | 충돌(예: 멱등성 키 재사용) 때문에 요청을 완료할 수 없습니다. |
| `422 Unprocessable Entity`  | 프로필 내용의 형식이 잘못되었습니다(디스커버리 실패).                              |
| `424 Failed Dependency`     | 프로필 URL은 유효하지만 가져오지 못했습니다(디스커버리 실패).                     |
| `429 Too Many Requests`     | 요청 한도를 초과했습니다.                                                               |
| `503 Service Unavailable`   | 일시적으로 사용할 수 없습니다.                                                          |
| `500 Internal Server Error` | 서버에서 예상치 못한 상황이 발생했습니다.                             |

### 에러 응답 { #error-responses }

전체 오류 코드 레지스트리와 전송 바인딩 예시는
[핵심 명세](../../overview/index.md#error-handling)를 참고하세요.

* **프로토콜 오류**: 적절한 HTTP 상태 코드(401, 403, 409, 429, 503)와 함께 `code`와
    `content`를 담은 JSON 본문을 반환합니다.
* **비즈니스 결과**: HTTP 200과 함께 UCP 엔벨로프와 `messages` 배열을 반환합니다.

#### 비즈니스 결과 { #business-outcomes }

비즈니스 결과(구매할 수 없는 상품 같은 오류 포함)는 HTTP 200과 `messages`를 담은 UCP
엔벨로프로 반환합니다.

<!-- ucp:example schema=shopping/checkout op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "status": "success",
    "payment_handlers": { ... },
    "capabilities": {
      "dev.ucp.shopping.checkout": [{"version": "{{ ucp_version }}"}]
    }
  },
  "id": "checkout_abc123",
  "status": "incomplete",
  "currency": "USD",
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
  "links": [...],
  "messages": [
    {
      "type": "warning",
      "code": "quantity_adjusted",
      "content": "Quantity adjusted, requested 100 units but only 12 available",
      "path": "$.line_items[0].quantity"
    }
  ],
  "continue_url": "https://merchant.com/checkout/checkout_abc123"
}
```

`create_checkout`에서 모든 상품을 구매할 수 없어 체크아웃을 생성하지 못하면, HTTP 200과
`messages`를 담은 UCP 엔벨로프를 반환합니다.

<!-- ucp:example schema=common/types/error_response op=read -->
```json
{
  "ucp": { "version": "2026-01-11", "status": "error" },
  "messages": [
    {
      "type": "error",
      "code": "item_unavailable",
      "content": "All items are not available for purchase",
      "severity": "unrecoverable"
    }
  ],
  "continue_url": "https://merchant.com/"
}
```

## 메시지 서명 { #message-signing }

플랫폼은 인증 메커니즘(API 키, OAuth, mTLS, HTTP 메시지 서명) 중 하나를 고를 수
있습니다(**MAY**). HTTP 메시지 서명을 사용하면 체크아웃 작업은
[메시지 서명](../../signatures.md) 명세를 따릅니다.

### 요청 서명 { #request-signing }

HTTP 메시지 서명을 사용하면 요청에 RFC 9421에 맞는 유효한 `Signature-Input`과
`Signature` 헤더(본문이 있으면 `Content-Digest`도)를 반드시 포함해야 합니다(**MUST**).

| 헤더                   | 필수 | 설명                              |
| :----------------------- | :------- | :--------------------------------------- |
| `Signature-Input`        | 예      | 서명된 구성 요소 설명              |
| `Signature`              | 예      | 서명 값             |
| `Content-Digest`         | 조건부*   | 요청 본문의 SHA-256 해시             |

\* 본문이 있는 요청(POST, PUT)에서 필수

**서명된 요청 예시:**

```http
POST /checkout-sessions HTTP/1.1
Host: merchant.example.com
Content-Type: application/json
UCP-Agent: profile="https://platform.example/.well-known/ucp"
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Digest: sha-256=:X48E9q...:
Signature-Input: sig1=("@method" "@authority" "@path" "idempotency-key" "content-digest" "content-type");keyid="platform-2025"
Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:

{"line_items":[{"item":{"id":"item_123"},"quantity":2}]}
```

전체 서명 알고리즘은 [메시지 서명 - REST 요청 서명](../../signatures.md#rest-request-signing)을
참고하세요.

### 응답 서명 { #response-signing }

다음 응답에는 응답 서명을 권장합니다(**RECOMMENDED**).

* `complete_checkout` 응답(주문 확인)

다음 응답에서는 응답 서명이 선택 사항입니다(**OPTIONAL**).

* `create_checkout`, `get_checkout`, `update_checkout`, `cancel_checkout`

**서명된 응답 예시:**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Digest: sha-256=:Y5fK8nLmPqRsT3vWxYzAbCdEfGhIjKlMnO...:
Signature-Input: sig1=("@status" "content-digest" "content-type");keyid="merchant-2026"
Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:

{"id":"chk_123","status":"completed","order":{"id":"ord_456"}}
```

전체 서명 알고리즘은 [메시지 서명 - REST 응답 서명](../../signatures.md#rest-response-signing)을
참고하세요.

## 보안 고려 사항 { #security-considerations }

### 인증 { #authentication }

인증은 선택 사항이며 비즈니스 요구 사항에 따라 달라집니다. 인증이 필요하면 REST 전송
방식에서 다음 방법을 사용할 수 있습니다(**MAY**).

1. **공개 API**: 공개 작업에는 인증이 필요 없습니다.
2. **API 키**: `X-API-Key` 헤더를 사용합니다.
3. **OAuth 2.0**: `Authorization: Bearer {token}` 헤더를 사용합니다. 에이전트 인증 접근에서는
   플랫폼을, 사용자 인증 접근에서는 플랫폼과 사용자를 모두 식별합니다
   ([신원 연결](../../common/identity-linking/index.md) 참고).
4. **상호 TLS(Mutual TLS)**: 높은 보안이 필요한 환경에 사용합니다.
5. **HTTP 메시지 서명**: [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421)을 따릅니다
    (위의 [메시지 서명](#message-signing) 참고).

비즈니스는 일부 작업에만 인증을 요구하고 나머지는 공개로 둘 수 있습니다(**MAY**)(예: 인증 없는
공개 체크아웃).
