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

# 주문 기능 - REST 바인딩 { #order-capability-rest-binding }

이 문서는 [주문 기능(Order Capability)](index.md)의 REST 바인딩을 정의한다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필을 통해 REST 전송 방식 지원 여부를
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
          "transport": "rest",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/rest.openapi.json",
          "endpoint": "https://business.example.com/ucp/v1"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.order": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/order",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/order.json"
        }
      ]
    },
    "payment_handlers": {}
  }
}
```

### 기본 URL { #base-url }

모든 UCP REST 엔드포인트는 비즈니스의 기본 URL을 기준으로 하며, 기본 URL은
`/.well-known/ucp`의 UCP 프로필을 통해 디스커버리된다. 주문 기능의 엔드포인트는
비즈니스 프로필의 `rest.endpoint` 필드에 정의된다.

### 콘텐츠 타입 { #content-types }

* **응답**: `application/json`

모든 응답 본문은 반드시
[RFC 8259](https://tools.ietf.org/html/rfc8259){ target="_blank" }에 명시된 유효한 JSON이어야 한다(**MUST**).

### 전송 보안 { #transport-security }

모든 REST 엔드포인트는 반드시 최소 TLS 1.3 버전의 HTTPS로 제공되어야 한다(**MUST**).

## 작업 { #operations }

| 작업 | 메서드 | 엔드포인트 | 설명 |
| :---- | :---- | :---- | :---- |
| [주문 조회](#get-order) | `GET` | `/orders/{id}` | 주문의 현재 상태를 조회한다. |

주문 이벤트 웹훅(비즈니스 -> 플랫폼 푸시)은
[주문 기능 개요](index.md#order-event-webhook)를 참고한다.

### 주문 조회 { #get-order }

주문의 현재 상태 스냅숏을 반환한다.

#### 입력 스키마 { #input-schema }

* `id` (String, 필수): 주문 ID(경로 파라미터).

#### 출력 스키마 { #output-schema }

{{ schema_fields('order', 'shopping/order') }}

#### 예시 { #example }

=== "요청"

    ```http
    GET /orders/order_abc123 HTTP/1.1
    UCP-Agent: profile="https://platform.example/.well-known/ucp"
    Accept: application/json
    Signature-Input: sig1=("@method" "@authority" "@path" "ucp-agent");created=1706800000;keyid="platform-2026"
    Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:
    ```

=== "응답"

    <!-- ucp:example schema=shopping/order op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.shopping.order": [{"version": "{{ ucp_version }}"}]
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
          "dev.ucp.shopping.order": [{"version": "{{ ucp_version }}"}]
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
    }
    ```

=== "권한 없음"

    <!-- ucp:example schema=common/types/error_response op=read -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "status": "error",
        "capabilities": {
          "dev.ucp.shopping.order": [{"version": "{{ ucp_version }}"}]
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
    }
    ```

## HTTP 헤더 { #http-headers }

{{ header_fields('get_order', 'shopping/rest.openapi.json') }}

### 헤더별 요구 사항 { #specific-header-requirements }

{{ header_requirements('ucp_agent') }}

## 메시지 서명 { #message-signing }

요청 및 응답 서명은 RFC 9421 HTTP Message Signatures를 사용하는
[메시지 서명](../../signatures.md) 명세를 따른다. 전체 알고리즘은
[REST 요청 서명](../../signatures.md#rest-request-signing)과
[REST 요청 검증](../../signatures.md#rest-request-verification)을
참고한다.

## 적합성 { #conformance }

REST 바인딩을 구현하는 플랫폼은 다음을 지킨다.

* 모든 요청에 프로필 URL이 담긴 `UCP-Agent` 헤더를 반드시 포함해야 한다(**MUST**)
* 주문 데이터에 접근하기 전에 응답의 `messages` 배열을 반드시 확인해야 한다(**MUST**)
* 권위 있는 주문 경험을 위해 `permalink_url`을 통해 비즈니스에 위임해야
  한다(**SHOULD**). 주문 상세 정보와 구매 후 작업의 기준이 되는 출처는 비즈니스
  사이트다

REST 바인딩을 구현하는 비즈니스는 다음을 지킨다.

* 모든 엔드포인트를 반드시 TLS 1.3 이상의 HTTPS로 제공해야 한다(**MUST**)
* [메시지 서명](../../signatures.md) 명세에 따라 응답에
  서명해야 한다(**SHOULD**)

모든 전송 방식에 공통으로 적용되는 기능 수준의 요구 사항은
[주문 기능 - 가이드라인](index.md#operations-guidelines)을 참고한다.
