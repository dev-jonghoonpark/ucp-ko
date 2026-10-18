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

# 예약 기능 - REST 바인딩 { #booking-capability-rest-binding }

이 문서는 [예약 기능(Booking Capability)](index.md)의
REST 바인딩을 정의합니다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필에서 REST 전송 방식 지원 여부를
알립니다.

<!-- ucp:example schema=profile def=business_schema -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "services": {
      "dev.ucp.lodging": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview",
          "transport": "rest",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/lodging/rest.openapi.json",
          "endpoint": "https://business.example.com/ucp/v1"
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

### 기본 URL { #base-url }

모든 UCP REST 엔드포인트는 비즈니스의 기본 URL을 기준으로 한 상대 경로이며, 기본 URL은
`/.well-known/ucp`의 UCP 프로필에서 확인합니다. 예약 기능의 엔드포인트는 비즈니스
프로필의 `rest.endpoint` 필드에 정의됩니다.

### 콘텐츠 타입 { #content-types }

* **요청**: `application/json`
* **응답**: `application/json`

모든 요청 및 응답 본문은 반드시
[RFC 8259](https://tools.ietf.org/html/rfc8259){ target="_blank" }에 정의된 유효한 JSON이어야 합니다(**MUST**).

### 전송 보안 { #transport-security }

모든 REST 엔드포인트는 반드시 최소 TLS 1.3 이상의 HTTPS로 제공해야
합니다(**MUST**).

## 오퍼레이션 { #operations }

| 오퍼레이션                                                     | 메서드 | 엔드포인트                         | 설명                       |
| :------------------------------------------------------------- | :----- | :--------------------------------- | :------------------------- |
| [예약 세션 생성](index.md#create-booking-session)              | `POST` | `/booking-sessions`                | 예약 세션을 생성합니다.    |
| [예약 세션 조회](index.md#get-booking-session)                 | `GET`  | `/booking-sessions/{id}`           | 예약 세션을 조회합니다.    |
| [예약 세션 수정](index.md#update-booking-session)              | `PUT`  | `/booking-sessions/{id}`           | 예약 세션을 수정합니다.    |
| [예약 세션 완료](index.md#complete-booking-session)            | `POST` | `/booking-sessions/{id}/complete`  | 예약을 완료합니다.         |
| [예약 세션 취소](index.md#cancel-booking-session)              | `POST` | `/booking-sessions/{id}/cancel`    | 예약 세션을 취소합니다.    |

## 예시 { #examples }

### 예약 세션 생성 { #create-booking-session }

#### 단일 숙박 예약 { #single-stay-booking }

=== "요청"

    <!-- ucp:example schema=lodging/booking op=create direction=request -->
    ```json
    POST /booking-sessions HTTP/1.1
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json
    ...other required headers...

    {
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
    ```

=== "응답"

    <!-- ucp:example schema=lodging/booking op=read -->
    ```json
    HTTP/1.1 201 Created
    Content-Type: application/json

    {
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
    }
    ```

=== "에러 응답"

    선택한 객실을 더 이상 이용할 수 없어 예약 세션 리소스가 생성되지 않은 경우:

    <!-- ucp:example schema=common/types/error_response op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
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
    }
    ```

#### 다중 숙박 예약 { #multi-stay-booking }

=== "요청"

    <!-- ucp:example schema=lodging/booking op=create direction=request -->
    ```json
    POST /booking-sessions HTTP/1.1
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json
    ...other required headers...

    {
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
        },
        {
          "id": "stay_standard_king__rp_avg_base_rate",
          "accommodation_type": {
            "id": "rt_standard_king"
          },
          "rate_plan": {
            "id": "rp_avg_base_rate"
          },
          "occupancy": {
            "adults": 1,
            "total": 1
          },
          "stay_dates": {
            "start_date": "2026-07-15",
            "end_date": "2026-07-21"
          }
        }
      ]
    }
    ```

=== "응답"

    <!-- ucp:example schema=lodging/booking op=read -->
    ```json
    HTTP/1.1 201 Created
    Content-Type: application/json

    {
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
      "id": "booking_124",
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
        },
        {
          "id": "stay_standard_king__rp_avg_base_rate",
          "accommodation_type": {
            "id": "rt_standard_king",
            "title": "Standard King Room",
            "capacity": {
              "adults": 2,
              "total": 2
            }
          },
          "rate_plan": {
            "id": "rp_avg_base_rate",
            "title": "Best Available Rate"
          },
          "occupancy": {
            "adults": 1,
            "total": 1
          },
          "stay_dates": {
            "start_date": "2026-07-15",
            "end_date": "2026-07-21"
          },
          "totals": [
            {
              "type": "subtotal",
              "amount": 270000
            },
            {
              "type": "tax",
              "amount": 27000
            },
            {
              "type": "total",
              "amount": 297000
            }
          ]
        }
      ],
      "currency": "USD",
      "totals": [
        {
          "type": "subtotal",
          "amount": 600000
        },
        {
          "type": "tax",
          "amount": 60000
        },
        {
          "type": "fee",
          "display_text": "Booking fee",
          "amount": 1000
        },
        {
          "type": "total",
          "amount": 661000
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
      "continue_url": "https://business.example.com/booking-sessions/booking_124",
      "expires_at": "2026-06-01T18:30:00Z"
    }
    ```

### 예약 세션 수정 { #update-booking-session }

이 오퍼레이션은 전체 교체 방식입니다. 플랫폼은 반드시 쓰기 전용 필드의 데이터 변경까지
포함해 예약 세션 리소스 전체를 보내야 하며(**MUST**), 전달된 리소스가 기존 예약 세션
상태를 대체합니다. 예약 세션이 `complete_in_progress` 상태인 동안 플랫폼은 새 수정
오퍼레이션을 시작해서는 안 됩니다(**MUST NOT**). 중복 요청에는 여전히
[재전송 방지](../../signatures.md#replay-protection) 규칙이 적용됩니다. 그 상태에서
새 수정 요청을 받으면 비즈니스는 반드시 예약 세션을 변경하지 않고, 복구 가능한 에러
메시지와 함께 현재 예약 세션을 반환해야 합니다(**MUST**).

`guests`, `booker`, `travel_purpose`, `stays[].guest_assignments`의 모든 필드는 선택
사항이므로, 플랫폼은 여러 번 호출하면서 예약 세션을 점진적으로 구성할 수 있습니다.
`complete_in_progress`가 아닐 때 각 수정은 예약 세션 전체를 교체하므로, 플랫폼은 반드시
유지하려는 기존 설정 필드를 모두 포함해야 합니다(**MUST**).

비즈니스가 `guests`, `booker`, `guest_assignments` 또는 주소(예: `billing_address`)에
특정 필드가 반드시 있어야 한다는 자체 로직이 있다면, 이 지점에서 `messages`로
그 요구 사항을 알리면 됩니다.

=== "요청"

    <!-- ucp:example schema=lodging/booking op=update direction=request -->
    ```json
    PUT /booking-sessions/{id} HTTP/1.1
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json
    ...other required headers...

    {
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
      // New data introduced in the update call.
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
      "booker": {
        "first_name": "John",
        "last_name": "Doe",
        "email": "john.doe@example.com",
        "phone_number": "+14155559876",
        "birthdate": "1980-08-26",
        "address": {
          "street_address": "1600 Amphitheatre Pkwy",
          "address_locality": "Mountain View",
          "address_region": "CA",
          "address_country": "US",
          "postal_code": "94043"
        }
      },
      "travel_purpose": "leisure"
    }
    ```

=== "응답"

    <!-- ucp:example schema=lodging/booking op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
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
      "booker": {
        "first_name": "John",
        "last_name": "Doe",
        "email": "john.doe@example.com",
        "phone_number": "+14155559876",
        "birthdate": "1980-08-26",
        "address": {
          "street_address": "1600 Amphitheatre Pkwy",
          "address_locality": "Mountain View",
          "address_region": "CA",
          "address_country": "US",
          "postal_code": "94043"
        }
      },
      "travel_purpose": "leisure",
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
    }
    ```

### 예약 세션 완료 { #complete-booking-session }

=== "요청"

    <!-- ucp:example schema=lodging/booking op=complete direction=request -->
    ```json
    POST /booking-sessions/{id}/complete
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json
    ...other required headers...

    {
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
    ```

=== "응답"

    <!-- ucp:example schema=lodging/booking op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
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
      "booker": {
        "first_name": "John",
        "last_name": "Doe",
        "email": "john.doe@example.com",
        "phone_number": "+14155559876",
        "birthdate": "1980-08-26",
        "address": {
          "street_address": "1600 Amphitheatre Pkwy",
          "address_locality": "Mountain View",
          "address_region": "CA",
          "address_country": "US",
          "postal_code": "94043"
        }
      },
      "travel_purpose": "leisure",
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
      // Booking confirmation that MUST be set in the response.
      "confirmation": {
        "id": "confirmation_123",
        "label": "CON123AZ",
        "pincode": "1234"
      },
      "payment": {...},
      "signals": {...}
    }
    ```

### 예약 세션 조회 { #get-booking-session }

=== "요청"

    <!-- ucp:example schema=lodging/booking op=read direction=request -->
    ```json
    GET /booking-sessions/{id}
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json
    ...other required headers...

    {}
    ```

=== "응답"

    <!-- ucp:example schema=lodging/booking op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
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
      "booker": {
        "first_name": "John",
        "last_name": "Doe",
        "email": "john.doe@example.com",
        "phone_number": "+14155559876",
        "birthdate": "1980-08-26",
        "address": {
          "street_address": "1600 Amphitheatre Pkwy",
          "address_locality": "Mountain View",
          "address_region": "CA",
          "address_country": "US",
          "postal_code": "94043"
        }
      },
      "travel_purpose": "leisure",
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
    }
    ```

### 예약 세션 취소 { #cancel-booking-session }

=== "요청"

    <!-- ucp:example schema=lodging/booking op=cancel direction=request -->
    ```json
    POST /booking-sessions/{id}/cancel
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json
    ...other required headers...

    {}
    ```

=== "응답"

    <!-- ucp:example schema=lodging/booking op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
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
      // Status is updated upon a successful cancellation.
      "status": "canceled",
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
      "booker": {
        "first_name": "John",
        "last_name": "Doe",
        "email": "john.doe@example.com",
        "phone_number": "+14155559876",
        "birthdate": "1980-08-26",
        "address": {
          "street_address": "1600 Amphitheatre Pkwy",
          "address_locality": "Mountain View",
          "address_region": "CA",
          "address_country": "US",
          "postal_code": "94043"
        }
      },
      "travel_purpose": "leisure",
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
      "continue_url": "https://business.example.com/booking-sessions/booking_123"
    }
    ```

## HTTP 헤더 { #http-headers }

다음 헤더는 HTTP 바인딩용으로 정의되며, 별도 언급이 없으면 모든 오퍼레이션에
적용됩니다.

{{ header_fields('create_booking_session', 'lodging/rest.openapi.json') }}

### 헤더별 요구 사항 { #specific-header-requirements }

* **UCP-Agent**: 모든 요청은 반드시 Dictionary Structured Field 문법
    ([RFC 8941](https://datatracker.ietf.org/doc/html/rfc8941){target="_blank"})으로 플랫폼
    프로필 URI를 담은 `UCP-Agent` 헤더를 포함해야 합니다(**MUST**).
    형식: `profile="https://platform.example/profile"`.
* **Idempotency-Key**: 상태를 변경하는 오퍼레이션은 멱등성을 지원해야
    합니다(**SHOULD**). 이 헤더가 제공되면 서버는 반드시 다음을 지켜야 합니다(**MUST**).
    1. 키를 오퍼레이션 결과와 함께 최소 24시간 동안 저장합니다.
    2. 요청 본문이 원래 요청과 일치하는 중복 키에는 캐시된 결과를 반환합니다.
    3. 본문이 다른데 키가 재사용되면 `409 Conflict`를 반환합니다.
    페이로드 일치 규약 전체는 [메시지 서명 — 멱등성 키 요구 사항](../../signatures.md#replay-protection)을
    참고하세요.

## 프로토콜 동작 방식 { #protocol-mechanics }

### 상태 코드 { #status-codes }

UCP는 표준 HTTP 상태 코드로 API 요청의 성공 또는 실패를
나타냅니다.

| 상태 코드                   | 설명                                                                               |
| :-------------------------- | :--------------------------------------------------------------------------------- |
| `200 OK`                    | 요청이 성공했습니다.                                                               |
| `201 Created`               | 리소스가 생성되었습니다.                                                           |
| `400 Bad Request`           | 요청이 유효하지 않거나 처리할 수 없습니다.                                         |
| `401 Unauthorized`          | 인증이 필요한데 실패했거나 인증 정보가 제공되지 않았습니다.                        |
| `403 Forbidden`             | 요청은 인증되었지만 사용자에게 필요한 권한이 없습니다.                             |
| `409 Conflict`              | 충돌 때문에 요청을 완료할 수 없습니다(예: 멱등성 키 재사용).                       |
| `422 Unprocessable Entity`  | 프로필 내용의 형식이 잘못되었습니다(디스커버리 실패).                              |
| `424 Failed Dependency`     | 프로필 URL은 유효하지만 가져오지 못했습니다(디스커버리 실패).                      |
| `429 Too Many Requests`     | 요청 빈도 제한을 초과했습니다.                                                     |
| `503 Service Unavailable`   | 일시적으로 사용할 수 없습니다.                                                     |
| `500 Internal Server Error` | 서버에서 예기치 않은 상황이 발생했습니다.                                          |

### 에러 응답 { #error-responses }

전체 에러 코드 레지스트리와 전송 바인딩 예시는 [핵심 명세](../../overview/index.md#error-handling)를
참고하세요.

* **프로토콜 에러**: `code`와 `content`를 담은 JSON 본문과 함께 적절한 HTTP 상태 코드(401,
    403, 409, 429, 503)를 반환합니다.
* **비즈니스 결과**: UCP 엔벨로프와 `messages` 배열을 담아 HTTP 200을 반환합니다.

#### 비즈니스 결과 { #business-outcomes }

비즈니스 결과(유효하지 않은 투숙객 정보 같은 에러 포함)는 `messages`를 담은 UCP
엔벨로프와 함께 HTTP 200으로 반환됩니다.

<!-- ucp:example schema=lodging/booking op=read -->
```json
{
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
}
```

`create_booking_session`에서 예약 세션을 생성할 수 없는 경우, 비즈니스는 반드시
HTTP 200과 `messages`를 담은 UCP 엔벨로프를 반환해야 합니다(**MUST**).

<!-- ucp:example schema=common/types/error_response op=read -->
```json
{
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
}
```

## 메시지 서명 { #message-signing }

플랫폼은 인증 메커니즘(API 키, OAuth, mTLS, HTTP Message Signatures) 중에서 선택할 수
있습니다(**MAY**). HTTP Message Signatures를 사용하는 경우 예약 오퍼레이션은
[메시지 서명](../../signatures.md) 명세를 따릅니다.

### 요청 서명 { #request-signing }

HTTP Message Signatures를 사용하는 경우, 요청은 반드시 RFC 9421에 따라 유효한
`Signature-Input`, `Signature` 헤더(본문이 있으면 `Content-Digest`도)를 포함해야
합니다(**MUST**).

| 헤더                     | 필수     | 설명                                     |
| :----------------------- | :------- | :--------------------------------------- |
| `Signature-Input`        | 예       | 서명된 구성 요소 목록                    |
| `Signature`              | 예       | 서명 값                                  |
| `Content-Digest`         | 조건부*  | 요청 본문의 SHA-256 해시                 |
| `UCP-Agent`              | 예       | 서명자 신원(프로필 URL)                  |
| `Idempotency-Key`        | 예       | 재전송 방지를 위한 고유 키               |

\* 본문이 있는 요청(POST, PUT)에서 필수

**서명된 요청 예시:**

```http
POST /booking-sessions HTTP/1.1
Host: business.example.com
Content-Type: application/json
UCP-Agent: profile="https://platform.example/.well-known/ucp"
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Digest: sha-256=:X48E9qOokqqrvdts8nOJRJN3OWDUoyWxBf7kbu9DBPE=:
Signature-Input: sig1=("@method" "@authority" "@path" "idempotency-key" "content-digest" "content-type");keyid="platform-2025"
Signature: sig1=:MEUCIQDTxNq8h7LGHpvVZQp1iHkFp9+3N8Mxk2zH1wK4YuVN8w...:

{"payment":{...}, ...}
```

전체 서명 알고리즘은 [메시지 서명 - REST 요청 서명](../../signatures.md#rest-request-signing)을
참고하세요.

### 응답 서명 { #response-signing }

다음 응답에는 서명을 권장합니다(**RECOMMENDED**).

* `complete_booking_session` 응답(예약 확정)

다음 응답의 서명은 선택 사항입니다(**OPTIONAL**).

* `create_booking_session`, `get_booking_session`, `update_booking_session`, `cancel_booking_session`

**서명된 응답 예시:**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Digest: sha-256=:Y5fK8nLmPqRsT3vWxYzAbCdEfGhIjKlMnO...:
Signature-Input: sig1=("@status" "content-digest" "content-type");keyid="business-2026"
Signature: sig1=:MFQCIH7kL9nM2oP5qR8sT1uV4wX6yZaB3cD...:

{"id":"booking_123","status":"completed","confirmation":{...}}
```

전체 서명 알고리즘은 [메시지 서명 - REST 응답 서명](../../signatures.md#rest-response-signing)을
참고하세요.

## 보안 고려 사항 { #security-considerations }

### 인증 { #authentication }

인증은 선택 사항이며 비즈니스 요구 사항에 따라 달라집니다. 인증이 필요한 경우 REST 전송
방식은 다음을 사용할 수 있습니다(**MAY**).

1. **Open API**: 공개 오퍼레이션에는 인증이 필요 없습니다.
2. **API 키**: `X-API-Key` 헤더를 사용합니다.
3. **OAuth 2.0**: `Authorization: Bearer {token}` 헤더를 사용합니다. 에이전트 인증 접근에서는
   플랫폼을, 사용자 인증 접근에서는 플랫폼과 사용자를 함께 식별합니다
   ([신원 연결(Identity Linking)](../../common/identity-linking/index.md) 참고).
4. **상호 TLS(Mutual TLS)**: 높은 보안이 필요한 환경에 사용합니다.
5. **HTTP Message Signatures**: [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421)을 따릅니다
    (위의 [메시지 서명](#message-signing) 참고).

비즈니스는 일부 오퍼레이션에만 인증을 요구하고 나머지는 공개해 둘 수 있습니다(**MAY**)
(예: 인증 없는 공개 예약).
