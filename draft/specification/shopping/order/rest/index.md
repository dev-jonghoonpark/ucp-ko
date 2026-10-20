# 주문 기능 - REST 바인딩

이 문서는 [주문 기능(Order Capability)](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/order/index.md)의 REST 바인딩을 정의합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`의 UCP 프로필에서 REST 전송 방식 지원 여부를 알립니다.

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
          "endpoint": "https://business.example.com/ucp/v1"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.order": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/shopping/order",
          "schema": "https://ucp.dev/draft/schemas/shopping/order.json"
        }
      ]
    },
    "payment_handlers": {}
  }
}
```

### 기본 URL

모든 UCP REST 엔드포인트는 비즈니스의 기본 URL을 기준으로 하며, 기본 URL은 `/.well-known/ucp`의 UCP 프로필에서 찾을 수 있습니다. 주문 기능의 엔드포인트는 비즈니스 프로필의 `rest.endpoint` 필드에 정의합니다.

### 콘텐츠 타입

- **응답**: `application/json`

모든 응답 본문은 반드시 [RFC 8259](https://tools.ietf.org/html/rfc8259)에 명시된 유효한 JSON이어야 합니다(**MUST**).

### 전송 보안

모든 REST 엔드포인트는 반드시 TLS 1.3 이상의 HTTPS로 제공해야 합니다(**MUST**).

## 작업

| 작업                    | 메서드 | 엔드포인트     | 설명                           |
| ----------------------- | ------ | -------------- | ------------------------------ |
| [주문 조회](#get-order) | `GET`  | `/orders/{id}` | 주문의 현재 상태를 조회합니다. |

주문 이벤트 웹훅(비즈니스 -> 플랫폼 푸시)은 [주문 기능 개요](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/order/#order-event-webhook)를 참고하세요.

### 주문 조회

주문의 현재 상태 스냅숏을 반환합니다.

#### 입력 스키마

- `id` (String, 필수): 주문 ID(경로 파라미터).

#### 출력 스키마

| 이름          | 타입                                                                                                | 필수 여부    | 설명                                                                                                                                                               |
| ------------- | --------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ucp           | [UCP Order Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-order-schema) | **Required** | UCP metadata for order responses. No payment handlers needed post-purchase.                                                                                        |
| id            | string                                                                                              | **Required** | Unique order identifier.                                                                                                                                           |
| label         | string                                                                                              | Optional     | Human-readable label for identifying the order. MUST only be provided by the business.                                                                             |
| checkout_id   | string                                                                                              | **Required** | Associated checkout ID for reconciliation.                                                                                                                         |
| permalink_url | string                                                                                              | **Required** | Permalink to access the order on merchant site.                                                                                                                    |
| line_items    | Array\[[Order Line Item](/ucp-ko/draft/draft/specification/reference/#order-line-item)\]            | **Required** | Line items representing what was purchased — can change post-order via edits or exchanges.                                                                         |
| fulfillment   | object                                                                                              | **Required** | Fulfillment data: buyer expectations and what actually happened.                                                                                                   |
| adjustments   | Array\[[Adjustment](/ucp-ko/draft/draft/specification/reference/#adjustment)\]                      | Optional     | Post-order events (refunds, returns, credits, disputes, cancellations, etc.) that exist independently of fulfillment.                                              |
| currency      | string                                                                                              | **Required** | ISO 4217 currency code. MUST match the currency from the originating checkout session.                                                                             |
| totals        | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                       | **Required** | Different totals for the order.                                                                                                                                    |
| policies      | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                              | Optional     | Snapshot of the policies that applied to the items at checkout, captured on the order as a durable record. `applies_to` targets are relative to the response root. |
| messages      | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                            | Optional     | Business outcome messages (errors, warnings, informational). Present when the business needs to communicate status or issues to the platform.                      |
| attribution   | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                             | Optional     | Snapshot of the attribution associated with the originating checkout. Read-only on the order.                                                                      |

#### 예시

```http
GET /orders/order_abc123 HTTP/1.1
UCP-Agent: profile="https://platform.example/.well-known/ucp"
Accept: application/json
Signature-Input: sig1=("@method" "@authority" "@path" "ucp-agent");created=1706800000;keyid="platform-2026"
Signature: sig1=:6G4i8TS6oUkGrx8KnCFUpsSPwd74...:
```

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "ucp": {
    "version": "draft",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "draft"}]
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

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "ucp": {
    "version": "draft",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "draft"}]
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

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "ucp": {
    "version": "draft",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "draft"}]
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

## HTTP 헤더

**Request Headers**

| 헤더              | 필수    | 설명                                                                                                                                                                                                                                                          |
| ----------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Authorization`   | No      | Should contain oauth token representing the following 2 schemes: 1. Platform self authenticating (client_credentials). 2. Platform authenticating on behalf of end user (authorization_code).                                                                 |
| `X-API-Key`       | No      | Authenticates the platform with a reusable api key allocated to the platform by the business.                                                                                                                                                                 |
| `Signature`       | No      | RFC 9421 HTTP Message Signature. Required when using HTTP Message Signatures for authentication. Format: `sig1=:<base64-signature>:`.                                                                                                                         |
| `Signature-Input` | No      | RFC 9421 Signature-Input header. Required when using HTTP Message Signatures for authentication. Format: `sig1=("@method" "@path" ...);created=<timestamp>;keyid="<key-id>"`.                                                                                 |
| `Request-Id`      | **Yes** | For tracing the requests across network layers and components.                                                                                                                                                                                                |
| `User-Agent`      | No      | Identifies the user agent string making the call.                                                                                                                                                                                                             |
| `UCP-Agent`       | **Yes** | Identifies the UCP agent making the call. All requests MUST include the UCP-Agent header containing the signer's profile URI using RFC 8941 Dictionary syntax. The URL MUST point to /.well-known/ucp. Format: profile="https://example.com/.well-known/ucp". |
| `Content-Type`    | No      | Representation Metadata. Tells the receiver what the data in the message body actually is.                                                                                                                                                                    |
| `Accept`          | No      | Content Negotiation. The client tells the server what data formats it is capable of understanding.                                                                                                                                                            |
| `Accept-Language` | No      | Localization. Tells the receiver the user's preferred natural languages, often with "weights" or priorities.                                                                                                                                                  |
| `Accept-Encoding` | No      | Compression. The client tells the server which content-codings it supports, usually for compression.                                                                                                                                                          |

**Response Headers**

| 헤더              | 필수 | 설명                                                                                                                  |
| ----------------- | ---- | --------------------------------------------------------------------------------------------------------------------- |
| `Signature`       | No   | RFC 9421 HTTP Message Signature for response. Contains the signature value in the format `sig1=:<base64-signature>:`. |
| `Signature-Input` | No   | RFC 9421 Signature-Input header for response. Describes signed components, timestamp, and key ID.                     |
| `Content-Digest`  | No   | JCS-canonicalized body digest per RFC 8785. Format: `sha-256=:<base64-digest>:`.                                      |

### 헤더별 요구 사항

- **UCP-Agent**: All requests **MUST** include the `UCP-Agent` header containing the platform profile URI using Dictionary Structured Field syntax ([RFC 8941](https://datatracker.ietf.org/doc/html/rfc8941)). Format: `profile="https://platform.example/profile"`.

## 메시지 서명

요청과 응답 서명은 RFC 9421 HTTP Message Signatures 기반의 [메시지 서명](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/signatures/index.md) 명세를 따릅니다. 전체 알고리즘은 [REST 요청 서명](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/signatures/#rest-request-signing)과 [REST 요청 검증](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/signatures/#rest-request-verification)을 참고하세요.

## 적합성

REST 바인딩을 구현하는 플랫폼은 다음을 지켜야 합니다.

- 모든 요청에 프로필 URL이 담긴 `UCP-Agent` 헤더를 반드시 포함해야 합니다(**MUST**).
- 주문 데이터에 접근하기 전에 응답의 `messages` 배열을 반드시 확인해야 합니다(**MUST**).
- 권위 있는 주문 경험은 `permalink_url`로 비즈니스에 맡겨야 합니다(**SHOULD**). 주문 상세 정보와 구매 후 작업의 기준 출처는 비즈니스 사이트입니다.

REST 바인딩을 구현하는 비즈니스는 다음을 지켜야 합니다.

- 모든 엔드포인트를 반드시 TLS 1.3 이상의 HTTPS로 제공해야 합니다(**MUST**).
- [메시지 서명](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/signatures/index.md) 명세에 따라 응답에 서명해야 합니다(**SHOULD**).

모든 전송 방식에 공통으로 적용되는 기능 수준의 요구 사항은 [주문 기능 - 가이드라인](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/order/#operations-guidelines)을 참고하세요.
