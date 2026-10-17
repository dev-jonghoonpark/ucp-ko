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

# 예약 기능 - MCP 바인딩 { #booking-capability-mcp-binding }

이 문서는 [예약 기능(Booking Capability)](index.md)의 Model Context Protocol(MCP)
바인딩을 명세한다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`의 UCP 프로필을 통해 MCP 전송 방식 지원 여부를
알린다.

<!-- ucp:example schema=profile def=business_schema op=read direction=response -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "services": {
      "dev.ucp.lodging": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview",
          "transport": "mcp",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/lodging/mcp.openrpc.json",
          "endpoint": "https://business.example.com/ucp/mcp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.lodging.booking": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/lodging/booking",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/lodging/booking.json"
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
          "config": {...}
        }
      ]
    }
  }
}
```

### 요청 메타데이터 { #request-metadata }

MCP 클라이언트는 모든 요청에 프로토콜 메타데이터를 담은 `meta` 객체를 반드시 포함해야
한다(**MUST**).

<!-- ucp:example schema=lodging/booking op=create direction=request extract=$.params.arguments.booking -->
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "create_booking_session",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/lodging-agent.json"
        },
        "idempotency-key": "550e8400-e29b-41d4-a716-446655440000"
      },
      "booking": {
        "property": {...},
        "stays": [...]
      }
    }
  }
}
```

[기능 협상](../../overview/index.md#negotiation-protocol)을 위해 모든 요청에
`meta["ucp-agent"]` 필드가 **필수**다. `complete_booking_session`과
`cancel_booking_session` 작업은 안전한 재시도를 위해 `meta["idempotency-key"]`도
필요하다. 플랫폼은 추가 메타데이터 필드를 포함할 수 있다(**MAY**).

## 도구 { #tools }

UCP 기능은 MCP 도구와 1:1로 대응한다.

### 식별자 패턴 { #identifier-pattern }

MCP 도구는 리소스 식별과 페이로드 데이터를 분리한다.

* **요청:** 기존 예약 세션에 대한 작업(`get`, `update`, `complete`, `cancel`)에서는
    최상위 `id` 파라미터가 대상 리소스를 식별한다. 요청 페이로드의 `booking` 객체에는
    `id` 필드가 있어서는 안 된다(**MUST NOT**).
* **응답:** 모든 응답은 전체 리소스 상태의 일부로 `booking.id`를 반드시 포함해야 한다(**MUST**).
* **생성:** `create_booking_session` 작업은 요청에 `id`가 필요 없으며, 응답에 새로 할당된 `booking.id`가 포함된다.

| 도구 | 작업 | 설명 |
| :------------------------- | :------------------------------------------------------------------ | :------------------------- |
| `create_booking_session` | [예약 세션 생성](index.md#create-booking-session) | 예약 세션을 생성한다. |
| `get_booking_session` | [예약 세션 조회](index.md#get-booking-session) | 예약 세션을 조회한다. |
| `update_booking_session` | [예약 세션 수정](index.md#update-booking-session) | 예약 세션을 수정한다. |
| `complete_booking_session` | [예약 세션 완료](index.md#complete-booking-session) | 예약을 완료한다. |
| `cancel_booking_session` | [예약 세션 취소](index.md#cancel-booking-session) | 예약 세션을 취소한다. |

### `create_booking_session` { #create_booking_session }

[예약 세션 생성](index.md#create-booking-session) 작업에 대응한다.

#### 입력 스키마 { #input-schema }

{{ schema_fields('booking_create_req', 'lodging/booking/mcp') }}

#### 출력 스키마 { #output-schema }

{{ schema_fields('booking_resp', 'lodging/booking/mcp') }}

#### 예시 { #example }

=== "요청"

    <!-- ucp:example schema=lodging/booking op=create direction=request extract=$.params.arguments.booking -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "method": "tools/call",
      "params": {
        "name": "create_booking_session",
        "arguments": {
          "meta": {
            "ucp-agent": {
              "profile": "https://platform.example/profiles/v2026-07/lodging-agent.json"
            }
          },
          "booking": {
            "property": {
              "id": "hotel_123"
            },
            "stays": [
              {
                "id": "stay_luxury_queen__rp_avg_base_rate",
                "accommodation_type": {
                  "id": "rt_luxury_queen"
                },
                "rate_plan": {
                  "id": "rp_avg_base_rate"
                },
                "occupancy": {
                  "adults": 2,
                  "total": 2
                },
                "stay_dates": {
                  "start_date": "2026-07-15",
                  "end_date": "2026-07-21"
                }
              }
            ]
          }
        }
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=lodging/booking op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.lodging.booking": [
                {"version": "{{ ucp_version }}"}
              ]
            },
            "payment_handlers": {
              "com.example.vendor.delegate_payment": [
                {"id": "handler_1", "version": "{{ ucp_version }}", "available_instruments": [{"type": "card"}], "config": {}}
              ]
            }
          },
          "id": "booking_123",
          "status": "incomplete",
          "property": {
            "id": "hotel_123",
            "name": "Beautiful Scenery Hotel",
            "address": {
              "street_address": "123 Scene St",
              "address_locality": "Phoenix",
              "address_region": "AZ",
              "address_country": "US",
              "postal_code": "85004"
            }
          },
          "stays": [
            {
              "id": "stay_luxury_queen__rp_avg_base_rate",
              "accommodation_type": {
                "id": "rt_luxury_queen",
                "title": "Luxury Queen Room with Two Queen Beds",
                "capacity": {
                  "adults": 2,
                  "children": 2,
                  "child_age_ranges": [
                    {
                      "ages": {
                        "min": 0,
                        "max": 5
                      },
                      "limit": 1
                    },
                    {
                      "ages": {
                        "min": 6,
                        "max": 17
                      },
                      "limit": 1
                    }
                  ],
                  "total": 4
                }
              },
              "rate_plan": {
                "id": "rp_avg_base_rate",
                "title": "Best Available Rate"
              },
              "occupancy": {
                "adults": 2,
                "total": 2
              },
              "stay_dates": {
                "start_date": "2026-07-15",
                "end_date": "2026-07-21"
              },
              "totals": [
                {
                  "type": "subtotal",
                  "amount": 330000
                },
                {
                  "type": "tax",
                  "amount": 33000
                },
                {
                  "type": "total",
                  "amount": 363000
                }
              ]
            }
          ],
          "currency": "USD",
          "totals": [
            {
              "type": "subtotal",
              "amount": 330000
            },
            {
              "type": "tax",
              "amount": 33000
            },
            {
              "type": "fee",
              "display_text": "Booking fee",
              "amount": 1000
            },
            {
              "type": "total",
              "amount": 364000
            }
          ],
          "links": [
            {
              "type": "privacy_policy",
              "url": "https://business.example.com/privacy"
            },
            {
              "type": "terms_of_service",
              "url": "https://business.example.com/terms"
            },
            {
              "type": "cancellation_policy",
              "url": "https://business.example.com/cancellation"
            }
          ],
          "continue_url": "https://business.example.com/booking-sessions/booking_123",
          "expires_at": "2026-06-01T18:30:00Z"
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

    선택한 객실을 더 이상 이용할 수 없어 예약 세션 리소스가 생성되지 않는다:

    <!-- ucp:example schema=common/types/error_response op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "status": "error"
          },
          "messages": [
            {
              "type": "error",
              "code": "inventory_exhausted",
              "content": "Selected room is no longer available.",
              "severity": "unrecoverable"
            }
          ],
          "continue_url": "https://business.com/"
        },
        "content": [
          {"type": "text", "text": "..."}
        ]
      }
    }
    ```

### `get_booking_session` { #get_booking_session }

[예약 세션 조회](index.md#get-booking-session) 작업에 대응한다.

#### 입력 스키마 { #input-schema_1 }

* `id` (String): **필수**. 조회할 예약 세션의 ID.

#### 출력 스키마 { #output-schema_1 }

{{ schema_fields('booking_resp', 'lodging/booking/mcp') }}

### `update_booking_session` { #update_booking_session }

[예약 세션 수정](index.md#update-booking-session) 작업에 대응한다.

#### 입력 스키마 { #input-schema_2 }

* `id` (String): **필수**. 수정할 예약 세션의 ID.

{{ schema_fields('booking_update_req', 'lodging/booking/mcp') }}

#### 출력 스키마 { #output-schema_2 }

{{ schema_fields('booking_resp', 'lodging/booking/mcp') }}

#### 예시 { #example_1 }

=== "요청"

    <!-- ucp:example schema=lodging/booking op=update direction=request extract=$.params.arguments.booking -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 3,
      "method": "tools/call",
      "params": {
        "name": "update_booking_session",
        "arguments": {
          "meta": {
            "ucp-agent": {
              "profile": "https://platform.example/profiles/v2026-07/lodging-agent.json"
            }
          },
          "id": "booking_123",
          "booking": {
            "property": {
              "id": "hotel_123"
            },
            "stays": [
              {
                "id": "stay_luxury_queen__rp_avg_base_rate",
                "accommodation_type": {
                  "id": "rt_luxury_queen"
                },
                "rate_plan": {
                  "id": "rp_avg_base_rate"
                },
                "occupancy": {
                  "adults": 2,
                  "total": 2
                },
                "stay_dates": {
                  "start_date": "2026-07-15",
                  "end_date": "2026-07-21"
                },
                "guest_assignments": [
                  {
                    "guest_id": "gst_01",
                    "role": "primary"
                  },
                  {
                    "guest_id": "gst_02",
                    "role": "accompanying"
                  }
                ]
              }
            ],
            "guests": [
              {
                "id": "gst_01",
                "first_name": "Jane",
                "last_name": "Doe",
                "email": "jane.doe@example.com",
                "phone_number": "+14155551234"
              },
              {
                "id": "gst_02",
                "first_name": "Mary",
                "last_name": "Doe"
              }
            ]
          }
        }
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=lodging/booking op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 3,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.lodging.booking": [
                {"version": "{{ ucp_version }}"}
              ]
            },
            "payment_handlers": {
              "com.example.vendor.delegate_payment": [
                {"id": "handler_1", "version": "{{ ucp_version }}", "available_instruments": [{"type": "card"}], "config": {}}
              ]
            }
          },
          "id": "booking_123",
          "status": "ready_for_complete",
          "property": {
            "id": "hotel_123",
            "name": "Beautiful Scenery Hotel",
            "address": {
              "street_address": "123 Scene St",
              "address_locality": "Phoenix",
              "address_region": "AZ",
              "address_country": "US",
              "postal_code": "85004"
            }
          },
          "stays": [
            {
              "id": "stay_luxury_queen__rp_avg_base_rate",
              "accommodation_type": {
                "id": "rt_luxury_queen",
                "title": "Luxury Queen Room with Two Queen Beds",
                "capacity": {
                  "adults": 2,
                  "children": 2,
                  "child_age_ranges": [
                    {
                      "ages": {
                        "min": 0,
                        "max": 5
                      },
                      "limit": 1
                    },
                    {
                      "ages": {
                        "min": 6,
                        "max": 17
                      },
                      "limit": 1
                    }
                  ],
                  "total": 4
                }
              },
              "rate_plan": {
                "id": "rp_avg_base_rate",
                "title": "Best Available Rate"
              },
              "occupancy": {
                "adults": 2,
                "total": 2
              },
              "stay_dates": {
                "start_date": "2026-07-15",
                "end_date": "2026-07-21"
              },
              "guest_assignments": [
                {
                  "guest_id": "gst_01",
                  "role": "primary"
                },
                {
                  "guest_id": "gst_02",
                  "role": "accompanying"
                }
              ],
              "totals": [
                {
                  "type": "subtotal",
                  "amount": 330000
                },
                {
                  "type": "tax",
                  "amount": 33000
                },
                {
                  "type": "total",
                  "amount": 363000
                }
              ]
            }
          ],
          "guests": [
            {
              "id": "gst_01",
              "first_name": "Jane",
              "last_name": "Doe",
              "email": "jane.doe@example.com",
              "phone_number": "+14155551234"
            },
            {
              "id": "gst_02",
              "first_name": "Mary",
              "last_name": "Doe"
            }
          ],
          "currency": "USD",
          "totals": [
            {
              "type": "subtotal",
              "amount": 330000
            },
            {
              "type": "tax",
              "amount": 33000
            },
            {
              "type": "fee",
              "display_text": "Booking fee",
              "amount": 1000
            },
            {
              "type": "total",
              "amount": 364000
            }
          ],
          "links": [
            {
              "type": "privacy_policy",
              "url": "https://business.example.com/privacy"
            },
            {
              "type": "terms_of_service",
              "url": "https://business.example.com/terms"
            },
            {
              "type": "cancellation_policy",
              "url": "https://business.example.com/cancellation"
            }
          ],
          "continue_url": "https://business.example.com/booking-sessions/booking_123",
          "expires_at": "2026-06-01T18:30:00Z"
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

### `complete_booking_session` { #complete_booking_session }

[예약 세션 완료](index.md#complete-booking-session) 작업에 대응한다.

#### 입력 스키마 { #input-schema_3 }

* `id` (String): **필수**. 예약 세션의 ID.

{{ schema_fields('booking_complete_req', 'lodging/booking/mcp') }}

#### 출력 스키마 { #output-schema_3 }

{{ schema_fields('booking_resp', 'lodging/booking/mcp') }}

**참고:** 완료에 성공하면 응답에 `confirmation` 객체가 반드시 포함되어야 한다(**MUST**).

#### 예시 { #example_2 }

=== "요청"

    <!-- ucp:example schema=lodging/booking op=complete direction=request extract=$.params.arguments.booking -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 2,
      "method": "tools/call",
      "params": {
        "name": "complete_booking_session",
        "arguments": {
          "meta": {
            "ucp-agent": {
              "profile": "https://platform.example/profiles/v2026-07/lodging-agent.json"
            },
            "idempotency-key": "550e8400-e29b-41d4-a716-446655440000"
          },
          "id": "booking_123",
          "booking": {
            "payment": {
              "instruments": [
                {
                  "id": "pi_handler_1",
                  "handler_id": "handler_1",
                  "type": "card",
                  "selected": true,
                  "display": {
                    "brand": "mastercard",
                    "last_digits": "5678",
                    "card_art": "https://cart-art-1.html",
                    "description": "Vendor Pay •••• 5678"
                  },
                  "billing_address": {
                    "street_address": "123 Main St",
                    "address_locality": "Mountain View",
                    "address_region": "CA",
                    "address_country": "US",
                    "postal_code": "94040"
                  },
                  "credential": {
                    "type": "PAYMENT_GATEWAY",
                    "token": "examplePaymentMethodToken"
                  }
                }
              ]
            },
            "signals": {
              "dev.ucp.user_agent": "Mozilla/5.0 ..."
            }
          }
        }
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=lodging/booking op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 2,
      "result": {
        "structuredContent": {
         "ucp": {
            "version": "{{ ucp_version }}",
            "capabilities": {
              "dev.ucp.lodging.booking": [
                {"version": "{{ ucp_version }}"}
              ]
            },
            "payment_handlers": {
              "com.example.vendor.delegate_payment": [
                {"id": "handler_1", "version": "{{ ucp_version }}", "available_instruments": [{"type": "card"}], "config": {}}
              ]
            }
          },
          "id": "booking_123",
          "status": "completed",
          "property": {
            "id": "hotel_123",
            "name": "Beautiful Scenery Hotel",
            "address": {
              "street_address": "123 Scene St",
              "address_locality": "Phoenix",
              "address_region": "AZ",
              "address_country": "US",
              "postal_code": "85004"
            }
          },
          "stays": [
            {
              "id": "stay_luxury_queen__rp_avg_base_rate",
              "accommodation_type": {
                "id": "rt_luxury_queen",
                "title": "Luxury Queen Room with Two Queen Beds",
                "capacity": {
                  "adults": 2,
                  "children": 2,
                  "child_age_ranges": [
                    {
                      "ages": {
                        "min": 0,
                        "max": 5
                      },
                      "limit": 1
                    },
                    {
                      "ages": {
                        "min": 6,
                        "max": 17
                      },
                      "limit": 1
                    }
                  ],
                  "total": 4
                }
              },
              "rate_plan": {
                "id": "rp_avg_base_rate",
                "title": "Best Available Rate"
              },
              "occupancy": {
                "adults": 2,
                "total": 2
              },
              "stay_dates": {
                "start_date": "2026-07-15",
                "end_date": "2026-07-21"
              },
              "guest_assignments": [
                {
                  "guest_id": "gst_01",
                  "role": "primary"
                },
                {
                  "guest_id": "gst_02",
                  "role": "accompanying"
                }
              ],
              "totals": [
                {
                  "type": "subtotal",
                  "amount": 330000
                },
                {
                  "type": "tax",
                  "amount": 33000
                },
                {
                  "type": "total",
                  "amount": 363000
                }
              ]
            }
          ],
          "guests": [
            {
              "id": "gst_01",
              "first_name": "Jane",
              "last_name": "Doe",
              "email": "jane.doe@example.com",
              "phone_number": "+14155551234"
            },
            {
              "id": "gst_02",
              "first_name": "Mary",
              "last_name": "Doe"
            }
          ],
          "currency": "USD",
          "totals": [
            {
              "type": "subtotal",
              "amount": 330000
            },
            {
              "type": "tax",
              "amount": 33000
            },
            {
              "type": "fee",
              "display_text": "Booking fee",
              "amount": 1000
            },
            {
              "type": "total",
              "amount": 364000
            }
          ],
          "links": [
            {
              "type": "privacy_policy",
              "url": "https://business.example.com/privacy"
            },
            {
              "type": "terms_of_service",
              "url": "https://business.example.com/terms"
            },
            {
              "type": "cancellation_policy",
              "url": "https://business.example.com/cancellation"
            }
          ],
          "confirmation": {
            "id": "confirmation_123",
            "label": "CON123AZ",
            "pincode": "1234"
          },
          "payment": {...},
          "signals": {...}
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

### `cancel_booking_session` { #cancel_booking_session }

[예약 세션 취소](index.md#cancel-booking-session) 작업에 대응한다.

#### 입력 스키마 { #input-schema_4 }

* `id` (String): **필수**. 예약 세션의 ID.

#### 출력 스키마 { #output-schema_4 }

{{ schema_fields('booking_resp', 'lodging/booking/mcp') }}

**참고:** 취소에 성공하면 응답에 `"status": "canceled"`가 반드시 포함되어야 한다(**MUST**).

## 오류 처리 { #error-handling }

UCP는 프로토콜 오류와 비즈니스 결과를 구분한다. 전체 오류 코드 레지스트리와 전송 바인딩
예시는 [핵심 명세](../../overview/index.md#error-handling)를 참고한다.

* **프로토콜 오류**: 요청 처리를 막는 전송 수준 실패(인증, 요청 한도 제한, 서비스
    불가). 코드 `-32000`(디스커버리 오류는 `-32001`)과 함께 JSON-RPC `error`로
    반환한다.
* **비즈니스 결과**: 요청 처리가 성공한 뒤의 애플리케이션 수준 결과. UCP 엔벌로프와
    `messages`를 담은 JSON-RPC `result`로 반환한다.

### 비즈니스 결과 { #business-outcomes }

비즈니스 결과(이용할 수 없는 상품 같은 오류 포함)는 UCP 엔벌로프와 `messages`를 담은
`structuredContent`가 있는 JSON-RPC `result`로 반환한다.

<!-- ucp:example schema=lodging/booking op=read direction=response extract=$.result.structuredContent -->
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.lodging.booking": [
            {"version": "{{ ucp_version }}"}
          ]
        },
        "payment_handlers": {
          "com.example.vendor.delegate_payment": [
            {"id": "handler_1", "version": "{{ ucp_version }}", "available_instruments": [{"type": "card"}], "config": {}}
          ]
        }
      },
      "id": "booking_123",
      "status": "incomplete",
      "property": {
        "id": "hotel_123",
        "name": "Beautiful Scenery Hotel"
      },
      "stays": [
        {
          "id": "stay_luxury_queen__rp_avg_base_rate",
          "accommodation_type": {
            "id": "rt_luxury_queen",
            "title": "Luxury Queen Room with Two Queen Beds",
            "capacity": {
              "adults": 2,
              "children": 2,
              "child_age_ranges": [
                {
                  "ages": {
                    "min": 0,
                    "max": 5
                  },
                  "limit": 1
                },
                {
                  "ages": {
                    "min": 6,
                    "max": 17
                  },
                  "limit": 1
                }
              ],
              "total": 4
            }
          },
          "rate_plan": {
            "id": "rp_avg_base_rate",
            "title": "Best Available Rate"
          },
          "occupancy": {
            "adults": 6,
            "total": 6
          },
          "stay_dates": {
            "start_date": "2026-07-15",
            "end_date": "2026-07-21"
          },
          "totals": [
            {
              "type": "subtotal",
              "amount": 350000
            },
            {
              "type": "tax",
              "amount": 42000
            },
            {
              "type": "total",
              "amount": 392000
            }
          ]
        }
      ],
      "currency": "USD",
      "totals": [
        {
          "type": "subtotal",
          "amount": 350000
        },
        {
          "type": "tax",
          "amount": 42000
        },
        {
          "type": "fee",
          "display_text": "Resort fee",
          "amount": 32500
        },
        {
          "type": "total",
          "amount": 424500
        }
      ],
      "links": [],
      "continue_url": "https://business.example.com/booking-sessions/booking_123",
      "expires_at": "2026-06-01T18:30:00Z",
      "messages": [
        {
          "type": "error",
          "code": "occupancy_exceeded_capacity",
          "content": "Number of additional guests requested surpassed room capacity.",
          "path": "$.stays[0]",
          "severity": "recoverable"
        }
      ]
    },
    "content": [
      {"type": "text", "text": "{\"ucp\":{…},…}"}
    ]
  }
}
```

`create_booking_session`에서 예약 세션을 생성할 수 없을 때는 UCP 엔벌로프와
`messages`를 담은 `structuredContent`가 있는 JSON-RPC `result`를 반환한다.

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
          "code": "inventory_exhausted",
          "content": "Selected room is no longer available.",
          "severity": "unrecoverable"
        }
      ],
      "continue_url": "https://business.com/"
    },
    "content": [
      {"type": "text", "text": "{\"ucp\":{…},…}"}
    ]
  }
}
```

## 메시지 서명 { #message-signing }

플랫폼은 MCP 전송 방식을 사용할 때 에이전트를 인증해야 한다(**SHOULD**). HTTP 메시지
서명을 사용하는 경우 모든 예약 작업은 [메시지 서명](../../signatures.md) 명세를 따른다.

### 요청 서명 { #request-signing }

UCP의 MCP 전송 방식은 **스트리밍 가능한 HTTP**를 사용하므로 REST와 같은 RFC 9421 서명
메커니즘을 쓸 수 있다. 서명은 HTTP 계층에서 적용된다.

| 헤더 | 필수 | 설명 |
| :----------------------- | :------- | :--------------------------------------- |
| `Signature-Input`        | 예 | 서명된 구성 요소를 설명한다 |
| `Signature`              | 예 | 서명 값을 담는다 |
| `Content-Digest`         | 예 | 요청 본문의 SHA-256 해시 |
| `UCP-Agent`              | 예 | 서명자 신원(프로필 URL) |
| `Idempotency-Key`        | 조건부* | 재전송 방지를 위한 고유 키 |

\* `complete_booking_session`과 `cancel_booking_session`에서 필수

**서명된 요청 예시:**

```http
POST /mcp HTTP/1.1
Host: business.example.com
Content-Type: application/json
UCP-Agent: profile="https://platform.example/.well-known/ucp"
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Digest: sha-256=:RK/0qy18MlBSVnWgjwz6lZEWjP/lF5HF9bvEF8FabDg=:
Signature-Input: sig1=("@method" "@authority" "@path" "content-digest" "content-type" "ucp-agent" "idempotency-key");keyid="platform-2026"
Signature: sig1=:MEUCIQDXyK9N3p5Rt...:

{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"complete_booking_session","arguments":{"id":"booking_123","booking":{"payment":{...}, ...}}}}
```

`Content-Digest`가 JSON-RPC 본문을 서명에 결합한다. JSON 정규화는 필요하지 않다.

자세한 내용은 [메시지 서명 - MCP 전송 방식](../../signatures.md#mcp-transport)을
참고한다.

### 응답 서명 { #response-signing }

다음에 대해서는 응답 서명을 권장한다(**RECOMMENDED**).

* `complete_booking_session` 응답(예약 확정)

다음에 대해서는 응답 서명이 선택 사항이다(**OPTIONAL**).

* `create_booking_session`, `get_booking_session`, `update_booking_session`, `cancel_booking_session`

**서명된 응답 예시:**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Digest: sha-256=:Y5fK8nLmPqRsT3vWxYzAbCdEfGhIjKlMnO...:
Signature-Input: sig1=("@status" "content-digest" "content-type");keyid="business-2026"
Signature: sig1=:MFQCIH7kL9nM2oP5qR8sT1uV4wX6yZaB3cD...:

{"jsonrpc":"2.0","id":1,"result":{"content":[{"type":"text","text":"..."}],"structuredContent":{"id":"booking_123","status":"completed", ...}}}
```

서명 알고리즘은 [메시지 서명 - REST 응답 서명](../../signatures.md#rest-response-signing)을
참고한다(HTTP 위의 MCP에서도 동일하다).

## 준수 { #conformance }

준수하는 MCP 전송 방식 구현은 반드시 다음을 지켜야 한다(**MUST**).

1. JSON-RPC 2.0 프로토콜을 올바르게 구현한다.
2. 이 명세에 정의된 모든 핵심 예약 도구를 제공한다.
3. [핵심 명세](../../overview/index.md#error-handling)에 따라 오류를 반환한다.
4. 비즈니스 결과를 UCP 엔벌로프와 `messages` 배열을 담은 JSON-RPC `result`로
    반환한다.
5. UCP 스키마에 따라 도구 입력을 검증한다.
6. 스트리밍을 지원하는 HTTP 전송 방식을 지원한다.

준수하는 구현은 다음을 지켜야 한다(**SHOULD**).

1. 지원되는 메커니즘(API 키, OAuth, mTLS, 또는 [메시지 서명](../../signatures.md)에 따른
    HTTP 메시지 서명) 중 하나로 에이전트를 인증한다.
2. 들어오는 요청을 처리하기 전에 인증을 검증한다.

## 구현 { #implementation }

UCP 작업은 [OpenRPC](https://open-rpc.org/)(JSON-RPC 스키마 형식)로 정의된다.
[MCP 명세](https://modelcontextprotocol.io/)는 모든 도구 호출에서 작업 이름과 인자를
`params`로 감싼 `tools/call` 메서드를 사용하도록 요구한다. 구현자는 반드시 다음 변환을
적용해야 한다(**MUST**).

| OpenRPC  | MCP                |
|:---------|:-------------------|
| `method` | `params.name`      |
| `params` | `params.arguments` |

**파라미터 규칙:**

* `meta`는 요청 메타데이터를 담는다
* `id`는 대상 리소스를 식별한다(경로 파라미터에 해당)
* `booking`은 도메인 페이로드를 담는다(본문에 해당)
