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

# 장바구니 기능 - REST 바인딩 { #cart-capability-rest-binding }

이 문서는 [장바구니 기능](index.md)의 REST 바인딩을 명세한다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필을 통해 REST 전송 방식 지원 여부를
공지한다.

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

### 기본 URL { #base-url }

모든 UCP REST 엔드포인트는 비즈니스의 기본 URL을 기준으로 하며, 기본 URL은
`/.well-known/ucp`의 UCP 프로필을 통해 디스커버리된다. 장바구니 기능의 엔드포인트는
비즈니스 프로필의 `rest.endpoint` 필드에 정의된다.

### 콘텐츠 타입 { #content-types }

* **요청**: `application/json`
* **응답**: `application/json`

모든 요청 및 응답 본문은 반드시
[RFC 8259](https://tools.ietf.org/html/rfc8259){ target="_blank" }에 명시된 유효한 JSON이어야 한다(**MUST**).

### 전송 보안 { #transport-security }

모든 REST 엔드포인트는 반드시 최소 TLS 1.3 버전의 HTTPS로 제공되어야 한다(**MUST**).

## 작업 { #operations }

| 작업 | 메서드 | 엔드포인트 | 설명 |
| :---- | :---- | :---- | :---- |
| [장바구니 생성](#create-cart) | `POST` | `/carts` | 장바구니 세션을 생성한다. |
| [장바구니 조회](#get-cart) | `GET` | `/carts/{id}` | 장바구니 세션을 조회한다. |
| [장바구니 수정](#update-cart) | `PUT` | `/carts/{id}` | 장바구니 세션을 수정한다. |
| [장바구니 취소](#cancel-cart) | `POST` | `/carts/{id}/cancel` | 장바구니 세션을 취소한다. |

### 장바구니 생성 { #create-cart }

#### 입력 스키마 { #input-schema }

{{ schema_fields('cart_create_req', 'shopping/cart') }}

#### 출력 스키마 { #output-schema }

{{ schema_fields('cart_resp', 'shopping/cart') }}

#### 예시 { #example }

=== "요청"

    <!-- ucp:example schema=shopping/cart op=create direction=request -->
    ```json
    POST /carts HTTP/1.1
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
      ],
      "context": {
        "address_country": "US",
        "address_region": "CA",
        "postal_code": "94105"
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=shopping/cart op=read -->
    ```json
    HTTP/1.1 201 Created
    Content-Type: application/json

    {
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
          "amount": 5000,
          "display_text": "Estimated total (taxes calculated at checkout)"
        }
      ],
      "continue_url": "https://business.example.com/checkout?cart=cart_abc123",
      "expires_at": "2026-01-16T12:00:00Z"
    }
    ```

=== "에러 응답"

    모든 상품이 품절된 경우 — 장바구니 리소스가 생성되지 않는다:

    <!-- ucp:example schema=common/types/error_response op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
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
    }
    ```

### 장바구니 조회 { #get-cart }

#### 입력 스키마 { #input-schema_1 }

* `id` (String, 필수): 장바구니 세션 ID(경로 매개변수).

#### 출력 스키마 { #output-schema_1 }

{{ schema_fields('cart_resp', 'shopping/cart') }}

#### 예시 { #example_1 }

=== "요청"

    ```http
    GET /carts/{id} HTTP/1.1
    UCP-Agent: profile="https://platform.example/profile"
    ```

=== "응답"

    <!-- ucp:example schema=shopping/cart op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
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
    }
    ```

=== "찾을 수 없음"

    <!-- ucp:example schema=common/types/error_response op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
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
    }
    ```

### 장바구니 수정 { #update-cart }

#### 입력 스키마 { #input-schema_2 }

* `id` (String, 필수): 장바구니 세션 ID(경로 매개변수).

{{ schema_fields('cart_update_req', 'shopping/cart') }}

#### 출력 스키마 { #output-schema_2 }

{{ schema_fields('cart_resp', 'shopping/cart') }}

#### 예시 { #example_2 }

=== "요청"

    <!-- ucp:example schema=shopping/cart op=update direction=request -->
    ```json
    PUT /carts/{id} HTTP/1.1
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json

    {
      "line_items": [
        {
          "item": {
            "id": "item_123"
          },
          "id": "li_1",
          "quantity": 3
        },
        {
          "item": {
            "id": "item_456"
          },
          "id": "li_2",
          "quantity": 1
        }
      ],
      "context": {
        "address_country": "US",
        "address_region": "CA",
        "postal_code": "94105"
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=shopping/cart op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
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
    }
    ```

### 장바구니 취소 { #cancel-cart }

#### 입력 스키마 { #input-schema_3 }

* `id` (String, 필수): 장바구니 세션 ID(경로 매개변수).

#### 출력 스키마 { #output-schema_3 }

{{ schema_fields('cart_resp', 'shopping/cart') }}

#### 예시 { #example_3 }

=== "요청"

    <!-- ucp:example schema=shopping/cart op=cancel direction=request -->
    ```json
    POST /carts/{id}/cancel HTTP/1.1
    UCP-Agent: profile="https://platform.example/profile"
    Content-Type: application/json

    {}
    ```

=== "응답"

    <!-- ucp:example schema=shopping/cart op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
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
    }
    ```

## HTTP 헤더 { #http-headers }

다음 헤더는 HTTP 바인딩용으로 정의되며, 별도로 명시하지 않는 한 모든 작업에
적용된다.

{{ header_fields('create_cart', 'shopping/rest.openapi.json') }}

### 개별 헤더 요구 사항 { #specific-header-requirements }

{{ header_requirements('ucp_agent', 'idempotency_key') }}

## 프로토콜 동작 방식 { #protocol-mechanics }

### 상태 코드 { #status-codes }

| 상태 코드 | 설명 |
| :--- | :--- |
| `200 OK` | 요청이 성공했다. |
| `201 Created` | 장바구니가 성공적으로 생성되었다. |
| `400 Bad Request` | 요청이 유효하지 않거나 처리할 수 없다. |
| `401 Unauthorized` | 인증이 필요하지만 실패했거나 제공되지 않았다. |
| `403 Forbidden` | 요청은 인증되었지만 사용자에게 필요한 권한이 없다. |
| `409 Conflict` | 충돌로 인해 요청을 완료할 수 없다(예: 멱등성 키 재사용). |
| `422 Unprocessable Entity` | 프로필 내용의 형식이 잘못되었다(디스커버리 실패). |
| `424 Failed Dependency` | 프로필 URL은 유효하지만 가져오기에 실패했다(디스커버리 실패). |
| `429 Too Many Requests` | 요청 한도를 초과했다. |
| `500 Internal Server Error` | 서버에서 예기치 않은 상황이 발생했다. |
| `503 Service Unavailable` | 일시적으로 사용할 수 없다. |

### 에러 응답 { #error-responses }

전체 에러 코드 레지스트리와 전송 방식 바인딩 예시는
[핵심 명세](../../overview/index.md#error-handling)를 참고한다.

* **프로토콜 에러**: 적절한 HTTP 상태 코드(401, 403, 409, 429,
    503)와 함께 `code`와 `content`를 담은 JSON 본문을 반환한다.
* **비즈니스 결과**: UCP 엔벨로프와 `messages` 배열을 담아 HTTP 200을 반환한다.

#### 비즈니스 결과 { #business-outcomes }

비즈니스 결과(찾을 수 없음 및 유효성 검사 에러 포함)는 HTTP 200과 함께
`messages`를 담은 UCP 엔벨로프로 반환된다:

<!-- ucp:example schema=common/types/error_response op=read -->
```json
{
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
}
```

## 보안 고려 사항 { #security-considerations }

### 인증 { #authentication }

인증은 선택 사항이며 비즈니스 요구 사항에 따라 달라진다. 인증이 필요한 경우
REST 전송 방식은 다음을 사용할 수 있다(**MAY**):

1. **Open API**: 공개 작업에는 인증이 필요하지 않다.
2. **API 키**: `X-API-Key` 헤더를 통해 전달한다.
3. **OAuth 2.0**: `Authorization: Bearer {token}` 헤더를 통해 전달한다. 에이전트 인증
   접근에서는 플랫폼을 식별하고, 사용자 인증 접근에서는 플랫폼과 사용자를 모두
   식별한다([신원 연결](../../common/identity-linking/index.md) 참고).
4. **Mutual TLS**: 높은 보안이 요구되는 환경에 사용한다.

비즈니스는 일부 작업에는 인증을 요구하고 나머지 작업은 공개로 둘 수 있다(**MAY**)
(예: 인증 없는 공개 장바구니).
