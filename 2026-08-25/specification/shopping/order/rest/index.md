# 주문 기능 - REST 바인딩

이 문서는 [주문 기능(Order Capability)](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/order/index.md)의 REST 바인딩을 정의합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`의 UCP 프로필에서 REST 전송 방식 지원 여부를 알립니다.

```json
{
  "ucp": {
    "version": "2026-08-25",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "2026-08-25",
          "spec": "https://ucp.dev/2026-08-25/specification/overview",
          "transport": "rest",
          "schema": "https://ucp.dev/2026-08-25/services/shopping/rest.openapi.json",
          "endpoint": "https://business.example.com/ucp/v1"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.order": [
        {
          "version": "2026-08-25",
          "spec": "https://ucp.dev/2026-08-25/specification/shopping/order",
          "schema": "https://ucp.dev/2026-08-25/schemas/shopping/order.json"
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

주문 이벤트 웹훅(비즈니스 -> 플랫폼 푸시)은 [주문 기능 개요](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/order/#order-event-webhook)를 참고하세요.

### 주문 조회

주문의 현재 상태 스냅숏을 반환합니다.

#### 입력 스키마

- `id` (String, 필수): 주문 ID(경로 파라미터).

#### 출력 스키마

| 이름          | 타입                                                                                               | 필수 여부 | 설명                                                                                                                               |
| ------------- | -------------------------------------------------------------------------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| ucp           | any                                                                                                | **필수**  | 주문 응답용 UCP 메타데이터. 구매 후에는 결제 핸들러가 필요 없습니다.                                                               |
| id            | string                                                                                             | **필수**  | 고유 주문 식별자.                                                                                                                  |
| label         | string                                                                                             | 선택      | 주문을 식별하기 위한 사람이 읽을 수 있는 라벨. 반드시 비즈니스만 제공해야 합니다(MUST).                                            |
| checkout_id   | string                                                                                             | **필수**  | 정산 대사를 위한 연관 체크아웃 ID.                                                                                                 |
| permalink_url | string                                                                                             | **필수**  | 판매자 사이트에서 주문에 접근하는 퍼머링크.                                                                                        |
| line_items    | Array\[[Order Line Item](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#order-line-item)\] | **필수**  | 구매한 항목을 나타내는 라인 아이템. 주문 후 수정이나 교환으로 바뀔 수 있습니다.                                                    |
| fulfillment   | object                                                                                             | **필수**  | 풀필먼트 데이터: 구매자 예상 정보와 실제로 일어난 일.                                                                              |
| adjustments   | Array\[[Adjustment](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#adjustment)\]           | 선택      | 풀필먼트와 독립적으로 존재하는 주문 후 이벤트(환불, 반품, 크레딧, 분쟁, 취소 등).                                                  |
| currency      | string                                                                                             | **필수**  | ISO 4217 통화 코드. 반드시 원래 체크아웃 세션의 통화와 일치해야 합니다(MUST).                                                      |
| totals        | [Totals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#totals)                            | **필수**  | 주문의 각종 합계.                                                                                                                  |
| policies      | Array\[[Policy](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#policy)\]                   | 선택      | 체크아웃 시 항목에 적용된 정책의 스냅샷으로, 지속적인 기록으로 주문에 저장됩니다. `applies_to` 대상은 응답 루트를 기준으로 합니다. |
| messages      | Array\[[Message](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message)\]                 | 선택      | 비즈니스 결과 메시지(오류, 경고, 정보). 비즈니스가 플랫폼에 상태나 문제를 전달해야 할 때 포함됩니다.                               |
| attribution   | [Attribution](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#attribution)                  | 선택      | 원래 체크아웃과 연결된 어트리뷰션의 스냅샷입니다. 주문에서는 읽기 전용입니다.                                                      |

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
    "version": "2026-08-25",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "2026-08-25"}]
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
    "version": "2026-08-25",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "2026-08-25"}]
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
    "version": "2026-08-25",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "2026-08-25"}]
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

| 헤더              | 필수   | 설명                                                                                                                                                                                                                                                               |
| ----------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Authorization`   | 아니요 | 다음 두 방식을 나타내는 OAuth 토큰을 담아야 합니다. 1. 플랫폼 자체 인증(client_credentials). 2. 최종 사용자를 대신한 플랫폼 인증(authorization_code).                                                                                                              |
| `X-API-Key`       | 아니요 | 비즈니스가 플랫폼에 할당한 재사용 가능한 API 키로 플랫폼을 인증합니다.                                                                                                                                                                                             |
| `Signature`       | 아니요 | RFC 9421 HTTP Message Signature입니다. 인증에 HTTP Message Signatures를 사용할 때 필수입니다. 형식: `sig1=:<base64-signature>:`.                                                                                                                                   |
| `Signature-Input` | 아니요 | RFC 9421 Signature-Input 헤더입니다. 인증에 HTTP Message Signatures를 사용할 때 필수입니다. 형식: `sig1=("@method" "@path" ...);created=<timestamp>;keyid="<key-id>"`.                                                                                             |
| `Request-Id`      | **예** | 네트워크 계층과 구성 요소 전반에서 요청을 추적하기 위한 값.                                                                                                                                                                                                        |
| `User-Agent`      | 아니요 | 호출하는 사용자 에이전트 문자열을 식별합니다.                                                                                                                                                                                                                      |
| `UCP-Agent`       | **예** | 호출하는 UCP 에이전트를 식별합니다. 모든 요청은 반드시 RFC 8941 Dictionary 구문으로 서명자의 프로필 URI를 담은 UCP-Agent 헤더를 포함해야 합니다(MUST). URL은 반드시 /.well-known/ucp를 가리켜야 합니다(MUST). 형식: profile="https://example.com/.well-known/ucp". |
| `Content-Type`    | 아니요 | 표현 메타데이터(Representation Metadata)입니다. 메시지 본문의 데이터가 실제로 무엇인지 수신자에게 알려 줍니다.                                                                                                                                                     |
| `Accept`          | 아니요 | 콘텐츠 협상. 클라이언트가 이해할 수 있는 데이터 형식을 서버에 알립니다.                                                                                                                                                                                            |
| `Accept-Language` | 아니요 | 현지화. 사용자가 선호하는 자연어를 수신자에게 알리며, 흔히 "가중치"나 우선순위가 함께 붙습니다.                                                                                                                                                                    |
| `Accept-Encoding` | 아니요 | 압축. 클라이언트가 지원하는 콘텐츠 인코딩(주로 압축)을 서버에 알립니다.                                                                                                                                                                                            |

**Response Headers**

| 헤더              | 필수   | 설명                                                                                                        |
| ----------------- | ------ | ----------------------------------------------------------------------------------------------------------- |
| `Signature`       | 아니요 | 응답에 대한 RFC 9421 HTTP Message Signature입니다. 서명 값을 `sig1=:<base64-signature>:` 형식으로 담습니다. |
| `Signature-Input` | 아니요 | 응답에 대한 RFC 9421 Signature-Input 헤더입니다. 서명된 구성 요소, 타임스탬프, 키 ID를 기술합니다.          |
| `Content-Digest`  | 아니요 | RFC 8785에 따라 JCS로 정규화한 본문 다이제스트. 형식: `sha-256=:<base64-digest>:`.                          |

### 헤더별 요구 사항

**UCP-Agent** (모든 요청에 필수):

[RFC 8941 Dictionary](https://www.rfc-editor.org/rfc/rfc8941#name-dictionaries) 문법으로 플랫폼을 식별합니다.

```http
UCP-Agent: profile="https://platform.example/.well-known/ucp"
```

## 메시지 서명

요청과 응답 서명은 RFC 9421 HTTP Message Signatures 기반의 [메시지 서명](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/index.md) 명세를 따릅니다. 전체 알고리즘은 [REST 요청 서명](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/#rest-request-signing)과 [REST 요청 검증](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/#rest-request-verification)을 참고하세요.

## 적합성

REST 바인딩을 구현하는 플랫폼은 다음을 지켜야 합니다.

- 모든 요청에 프로필 URL이 담긴 `UCP-Agent` 헤더를 반드시 포함해야 합니다(**MUST**).
- 주문 데이터에 접근하기 전에 응답의 `messages` 배열을 반드시 확인해야 합니다(**MUST**).
- 권위 있는 주문 경험은 `permalink_url`로 비즈니스에 맡겨야 합니다(**SHOULD**). 주문 상세 정보와 구매 후 작업의 기준 출처는 비즈니스 사이트입니다.

REST 바인딩을 구현하는 비즈니스는 다음을 지켜야 합니다.

- 모든 엔드포인트를 반드시 TLS 1.3 이상의 HTTPS로 제공해야 합니다(**MUST**).
- [메시지 서명](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/index.md) 명세에 따라 응답에 서명해야 합니다(**SHOULD**).

모든 전송 방식에 공통으로 적용되는 기능 수준의 요구 사항은 [주문 기능 - 가이드라인](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/order/#operations-guidelines)을 참고하세요.
