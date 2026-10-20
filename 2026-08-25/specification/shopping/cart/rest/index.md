# 장바구니 기능 - REST 바인딩

이 문서는 [장바구니 기능](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/cart/index.md)의 REST 바인딩을 명세합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필에서 REST 전송 방식 지원 여부를 알립니다.

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
      "dev.ucp.shopping.checkout": [
        {
          "version": "2026-08-25",
          "spec": "https://ucp.dev/2026-08-25/specification/shopping/checkout",
          "schema": "https://ucp.dev/2026-08-25/schemas/shopping/checkout.json"
        }
      ],
      "dev.ucp.shopping.cart": [
        {
          "version": "2026-08-25",
          "spec": "https://ucp.dev/2026-08-25/specification/shopping/cart",
          "schema": "https://ucp.dev/2026-08-25/schemas/shopping/cart.json"
        }
      ]
    },
    "payment_handlers": {}
  }
}
```

### 기본 URL

모든 UCP REST 엔드포인트는 비즈니스의 기본 URL을 기준으로 합니다. 기본 URL은 `/.well-known/ucp`의 UCP 프로필에서 찾을 수 있습니다. 장바구니 기능의 엔드포인트는 비즈니스 프로필의 `rest.endpoint` 필드에 정의합니다.

### 콘텐츠 타입

- **요청**: `application/json`
- **응답**: `application/json`

모든 요청 및 응답 본문은 반드시 [RFC 8259](https://tools.ietf.org/html/rfc8259)에 명시된 유효한 JSON이어야 합니다(**MUST**).

### 전송 보안

모든 REST 엔드포인트는 반드시 TLS 1.3 이상의 HTTPS로 제공해야 합니다(**MUST**).

## 작업

| 작업                          | 메서드 | 엔드포인트           | 설명                        |
| ----------------------------- | ------ | -------------------- | --------------------------- |
| [장바구니 생성](#create-cart) | `POST` | `/carts`             | 장바구니 세션을 생성합니다. |
| [장바구니 조회](#get-cart)    | `GET`  | `/carts/{id}`        | 장바구니 세션을 조회합니다. |
| [장바구니 수정](#update-cart) | `PUT`  | `/carts/{id}`        | 장바구니 세션을 수정합니다. |
| [장바구니 취소](#cancel-cart) | `POST` | `/carts/{id}/cancel` | 장바구니 세션을 취소합니다. |

### 장바구니 생성

#### 입력 스키마

| 이름        | 타입                                                                                   | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | -------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| line_items  | Array\[[Line Item](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#line-item)\] | **Required** | Cart line items. Same structure as checkout. Full replacement on update.                                                                                                                                                                                                                                                                                                                                |
| context     | [Context](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#context)              | Optional     | Buyer signals for localization (country, region, postal_code). Merchant uses for pricing, availability, currency. Falls back to geo-IP if omitted.                                                                                                                                                                                                                                                      |
| signals     | [Signals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#signals)              | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace. |
| attribution | [Attribution](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#attribution)      | Optional     | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                        |
| buyer       | [Buyer](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#buyer)                  | Optional     | Optional buyer information for personalized estimates.                                                                                                                                                                                                                                                                                                                                                  |

#### 출력 스키마

| 이름         | 타입                                                                                            | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ----------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | any                                                                                             | **Required** | UCP metadata for cart responses. No payment handlers needed pre-checkout.                                                                                                                                                                                                                                                                                                                               |
| id           | string                                                                                          | **Required** | Unique cart identifier.                                                                                                                                                                                                                                                                                                                                                                                 |
| line_items   | Array\[[Line Item Response](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#line-item)\] | **Required** | Cart line items. Same structure as checkout. Full replacement on update.                                                                                                                                                                                                                                                                                                                                |
| context      | [Context](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#context)                       | Optional     | Buyer signals for localization (country, region, postal_code). Merchant uses for pricing, availability, currency. Falls back to geo-IP if omitted.                                                                                                                                                                                                                                                      |
| signals      | [Signals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#signals)                       | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace. |
| attribution  | [Attribution](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#attribution)               | Optional     | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                        |
| buyer        | [Buyer](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#buyer)                           | Optional     | Optional buyer information for personalized estimates.                                                                                                                                                                                                                                                                                                                                                  |
| currency     | string                                                                                          | **Required** | ISO 4217 currency code. Determined by merchant based on context or geo-IP.                                                                                                                                                                                                                                                                                                                              |
| totals       | [Totals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#totals)                         | **Required** | Estimated cost breakdown. May be partial if shipping/tax not yet calculable.                                                                                                                                                                                                                                                                                                                            |
| actions      | [Actions](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#actions)                       | Optional     | Outstanding extension-defined Actions for this cart.                                                                                                                                                                                                                                                                                                                                                    |
| messages     | Array\[[Message](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message)\]              | Optional     | Validation messages, warnings, or informational notices.                                                                                                                                                                                                                                                                                                                                                |
| links        | Array\[[Link](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#link)\]                    | Optional     | Optional merchant links (policies, FAQs).                                                                                                                                                                                                                                                                                                                                                               |
| policies     | Array\[[Policy](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#policy)\]                | Optional     | Policies (e.g., return/refund terms) that apply to the items in this cart. `applies_to` targets are relative to the response root; when absent or empty, refer to the URLs in `links[]`.                                                                                                                                                                                                                |
| continue_url | string                                                                                          | Optional     | URL for cart handoff and session recovery. Enables sharing and human-in-the-loop flows.                                                                                                                                                                                                                                                                                                                 |
| expires_at   | string                                                                                          | Optional     | Cart expiry timestamp (RFC 3339). Optional.                                                                                                                                                                                                                                                                                                                                                             |

#### 예시

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

```json
HTTP/1.1 201 Created
Content-Type: application/json

{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": {
      "dev.ucp.shopping.checkout": [{"version": "2026-08-25"}],
      "dev.ucp.shopping.cart": [{"version": "2026-08-25"}]
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

모든 상품이 품절된 경우에는 장바구니 리소스가 생성되지 않습니다.

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "ucp": { "version": "2026-08-25", "status": "error" },
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

### 장바구니 조회

#### 입력 스키마

- `id` (String, 필수): 장바구니 세션 ID(경로 매개변수).

#### 출력 스키마

| 이름         | 타입                                                                                            | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ----------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | any                                                                                             | **Required** | UCP metadata for cart responses. No payment handlers needed pre-checkout.                                                                                                                                                                                                                                                                                                                               |
| id           | string                                                                                          | **Required** | Unique cart identifier.                                                                                                                                                                                                                                                                                                                                                                                 |
| line_items   | Array\[[Line Item Response](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#line-item)\] | **Required** | Cart line items. Same structure as checkout. Full replacement on update.                                                                                                                                                                                                                                                                                                                                |
| context      | [Context](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#context)                       | Optional     | Buyer signals for localization (country, region, postal_code). Merchant uses for pricing, availability, currency. Falls back to geo-IP if omitted.                                                                                                                                                                                                                                                      |
| signals      | [Signals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#signals)                       | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace. |
| attribution  | [Attribution](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#attribution)               | Optional     | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                        |
| buyer        | [Buyer](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#buyer)                           | Optional     | Optional buyer information for personalized estimates.                                                                                                                                                                                                                                                                                                                                                  |
| currency     | string                                                                                          | **Required** | ISO 4217 currency code. Determined by merchant based on context or geo-IP.                                                                                                                                                                                                                                                                                                                              |
| totals       | [Totals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#totals)                         | **Required** | Estimated cost breakdown. May be partial if shipping/tax not yet calculable.                                                                                                                                                                                                                                                                                                                            |
| actions      | [Actions](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#actions)                       | Optional     | Outstanding extension-defined Actions for this cart.                                                                                                                                                                                                                                                                                                                                                    |
| messages     | Array\[[Message](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message)\]              | Optional     | Validation messages, warnings, or informational notices.                                                                                                                                                                                                                                                                                                                                                |
| links        | Array\[[Link](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#link)\]                    | Optional     | Optional merchant links (policies, FAQs).                                                                                                                                                                                                                                                                                                                                                               |
| policies     | Array\[[Policy](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#policy)\]                | Optional     | Policies (e.g., return/refund terms) that apply to the items in this cart. `applies_to` targets are relative to the response root; when absent or empty, refer to the URLs in `links[]`.                                                                                                                                                                                                                |
| continue_url | string                                                                                          | Optional     | URL for cart handoff and session recovery. Enables sharing and human-in-the-loop flows.                                                                                                                                                                                                                                                                                                                 |
| expires_at   | string                                                                                          | Optional     | Cart expiry timestamp (RFC 3339). Optional.                                                                                                                                                                                                                                                                                                                                                             |

#### 예시

```http
GET /carts/{id} HTTP/1.1
UCP-Agent: profile="https://platform.example/profile"
```

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": {
      "dev.ucp.shopping.checkout": [{"version": "2026-08-25"}],
      "dev.ucp.shopping.cart": [{"version": "2026-08-25"}]
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

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "ucp": {
    "version": "2026-08-25",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.cart": [{"version": "2026-08-25"}]
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

### 장바구니 수정

#### 입력 스키마

- `id` (String, 필수): 장바구니 세션 ID(경로 매개변수).

| 이름        | 타입                                                                                   | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | -------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| line_items  | Array\[[Line Item](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#line-item)\] | **Required** | Cart line items. Same structure as checkout. Full replacement on update.                                                                                                                                                                                                                                                                                                                                |
| context     | [Context](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#context)              | Optional     | Buyer signals for localization (country, region, postal_code). Merchant uses for pricing, availability, currency. Falls back to geo-IP if omitted.                                                                                                                                                                                                                                                      |
| signals     | [Signals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#signals)              | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace. |
| attribution | [Attribution](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#attribution)      | Optional     | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                        |
| buyer       | [Buyer](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#buyer)                  | Optional     | Optional buyer information for personalized estimates.                                                                                                                                                                                                                                                                                                                                                  |

#### 출력 스키마

| 이름         | 타입                                                                                            | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ----------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | any                                                                                             | **Required** | UCP metadata for cart responses. No payment handlers needed pre-checkout.                                                                                                                                                                                                                                                                                                                               |
| id           | string                                                                                          | **Required** | Unique cart identifier.                                                                                                                                                                                                                                                                                                                                                                                 |
| line_items   | Array\[[Line Item Response](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#line-item)\] | **Required** | Cart line items. Same structure as checkout. Full replacement on update.                                                                                                                                                                                                                                                                                                                                |
| context      | [Context](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#context)                       | Optional     | Buyer signals for localization (country, region, postal_code). Merchant uses for pricing, availability, currency. Falls back to geo-IP if omitted.                                                                                                                                                                                                                                                      |
| signals      | [Signals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#signals)                       | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace. |
| attribution  | [Attribution](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#attribution)               | Optional     | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                        |
| buyer        | [Buyer](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#buyer)                           | Optional     | Optional buyer information for personalized estimates.                                                                                                                                                                                                                                                                                                                                                  |
| currency     | string                                                                                          | **Required** | ISO 4217 currency code. Determined by merchant based on context or geo-IP.                                                                                                                                                                                                                                                                                                                              |
| totals       | [Totals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#totals)                         | **Required** | Estimated cost breakdown. May be partial if shipping/tax not yet calculable.                                                                                                                                                                                                                                                                                                                            |
| actions      | [Actions](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#actions)                       | Optional     | Outstanding extension-defined Actions for this cart.                                                                                                                                                                                                                                                                                                                                                    |
| messages     | Array\[[Message](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message)\]              | Optional     | Validation messages, warnings, or informational notices.                                                                                                                                                                                                                                                                                                                                                |
| links        | Array\[[Link](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#link)\]                    | Optional     | Optional merchant links (policies, FAQs).                                                                                                                                                                                                                                                                                                                                                               |
| policies     | Array\[[Policy](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#policy)\]                | Optional     | Policies (e.g., return/refund terms) that apply to the items in this cart. `applies_to` targets are relative to the response root; when absent or empty, refer to the URLs in `links[]`.                                                                                                                                                                                                                |
| continue_url | string                                                                                          | Optional     | URL for cart handoff and session recovery. Enables sharing and human-in-the-loop flows.                                                                                                                                                                                                                                                                                                                 |
| expires_at   | string                                                                                          | Optional     | Cart expiry timestamp (RFC 3339). Optional.                                                                                                                                                                                                                                                                                                                                                             |

#### 예시

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

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": {
      "dev.ucp.shopping.checkout": [{"version": "2026-08-25"}],
      "dev.ucp.shopping.cart": [{"version": "2026-08-25"}]
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

### 장바구니 취소

#### 입력 스키마

- `id` (String, 필수): 장바구니 세션 ID(경로 매개변수).

#### 출력 스키마

| 이름         | 타입                                                                                            | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ----------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | any                                                                                             | **Required** | UCP metadata for cart responses. No payment handlers needed pre-checkout.                                                                                                                                                                                                                                                                                                                               |
| id           | string                                                                                          | **Required** | Unique cart identifier.                                                                                                                                                                                                                                                                                                                                                                                 |
| line_items   | Array\[[Line Item Response](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#line-item)\] | **Required** | Cart line items. Same structure as checkout. Full replacement on update.                                                                                                                                                                                                                                                                                                                                |
| context      | [Context](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#context)                       | Optional     | Buyer signals for localization (country, region, postal_code). Merchant uses for pricing, availability, currency. Falls back to geo-IP if omitted.                                                                                                                                                                                                                                                      |
| signals      | [Signals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#signals)                       | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace. |
| attribution  | [Attribution](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#attribution)               | Optional     | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                        |
| buyer        | [Buyer](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#buyer)                           | Optional     | Optional buyer information for personalized estimates.                                                                                                                                                                                                                                                                                                                                                  |
| currency     | string                                                                                          | **Required** | ISO 4217 currency code. Determined by merchant based on context or geo-IP.                                                                                                                                                                                                                                                                                                                              |
| totals       | [Totals](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#totals)                         | **Required** | Estimated cost breakdown. May be partial if shipping/tax not yet calculable.                                                                                                                                                                                                                                                                                                                            |
| actions      | [Actions](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#actions)                       | Optional     | Outstanding extension-defined Actions for this cart.                                                                                                                                                                                                                                                                                                                                                    |
| messages     | Array\[[Message](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message)\]              | Optional     | Validation messages, warnings, or informational notices.                                                                                                                                                                                                                                                                                                                                                |
| links        | Array\[[Link](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#link)\]                    | Optional     | Optional merchant links (policies, FAQs).                                                                                                                                                                                                                                                                                                                                                               |
| policies     | Array\[[Policy](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#policy)\]                | Optional     | Policies (e.g., return/refund terms) that apply to the items in this cart. `applies_to` targets are relative to the response root; when absent or empty, refer to the URLs in `links[]`.                                                                                                                                                                                                                |
| continue_url | string                                                                                          | Optional     | URL for cart handoff and session recovery. Enables sharing and human-in-the-loop flows.                                                                                                                                                                                                                                                                                                                 |
| expires_at   | string                                                                                          | Optional     | Cart expiry timestamp (RFC 3339). Optional.                                                                                                                                                                                                                                                                                                                                                             |

#### 예시

```json
POST /carts/{id}/cancel HTTP/1.1
UCP-Agent: profile="https://platform.example/profile"
Content-Type: application/json

{}
```

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "ucp": {
    "version": "2026-08-25",
    "capabilities": {
      "dev.ucp.shopping.checkout": [{"version": "2026-08-25"}],
      "dev.ucp.shopping.cart": [{"version": "2026-08-25"}]
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

## HTTP 헤더

다음 헤더는 HTTP 바인딩용으로 정의하며, 별도로 명시하지 않는 한 모든 작업에 적용됩니다.

**Request Headers**

| 헤더              | 필수    | 설명                                                                                                                                                                                                                                                          |
| ----------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Authorization`   | No      | Should contain oauth token representing the following 2 schemes: 1. Platform self authenticating (client_credentials). 2. Platform authenticating on behalf of end user (authorization_code).                                                                 |
| `X-API-Key`       | No      | Authenticates the platform with a reusable api key allocated to the platform by the business.                                                                                                                                                                 |
| `Signature`       | No      | RFC 9421 HTTP Message Signature. Required when using HTTP Message Signatures for authentication. Format: `sig1=:<base64-signature>:`.                                                                                                                         |
| `Signature-Input` | No      | RFC 9421 Signature-Input header. Required when using HTTP Message Signatures for authentication. Format: `sig1=("@method" "@path" ...);created=<timestamp>;keyid="<key-id>"`.                                                                                 |
| `Content-Digest`  | No      | Body digest per RFC 9530. Required for requests/responses with a body. Format: `sha-256=:<base64-digest>:`.                                                                                                                                                   |
| `Idempotency-Key` | **Yes** | Ensures duplicate operations don't happen during retries.                                                                                                                                                                                                     |
| `Request-Id`      | **Yes** | For tracing the requests across network layers and components.                                                                                                                                                                                                |
| `User-Agent`      | No      | Identifies the user agent string making the call.                                                                                                                                                                                                             |
| `UCP-Agent`       | **Yes** | Identifies the UCP agent making the call. All requests MUST include the UCP-Agent header containing the signer's profile URI using RFC 8941 Dictionary syntax. The URL MUST point to /.well-known/ucp. Format: profile="https://example.com/.well-known/ucp". |
| `Content-Type`    | No      | Representation Metadata. Tells the receiver what the data in the message body actually is.                                                                                                                                                                    |
| `Accept`          | No      | Content Negotiation. The client tells the server what data formats it is capable of understanding.                                                                                                                                                            |
| `Accept-Language` | No      | Localization. Tells the receiver the user's preferred natural languages, often with "weights" or priorities.                                                                                                                                                  |
| `Accept-Encoding` | No      | Compression. The client tells the server which content-codings it supports, usually for compression.                                                                                                                                                          |

### 개별 헤더 요구 사항

- **UCP-Agent**: 모든 요청은 반드시 Dictionary Structured Field 문법 ([RFC 8941](https://datatracker.ietf.org/doc/html/rfc8941))으로 플랫폼 프로필 URI를 담은 `UCP-Agent` 헤더를 포함해야 합니다(**MUST**). 형식: `profile="https://platform.example/profile"`.
- **Idempotency-Key**: 상태를 변경하는 오퍼레이션은 멱등성을 지원해야 합니다(**SHOULD**). 이 헤더가 제공되면 서버는 반드시 다음을 지켜야 합니다(**MUST**).
  1. 키를 오퍼레이션 결과와 함께 최소 24시간 동안 저장합니다.
  1. 요청 본문이 원래 요청과 일치하는 중복 키에는 캐시된 결과를 반환합니다.
  1. 본문이 다른데 키가 재사용되면 `409 Conflict`를 반환합니다. 페이로드 일치 규약 전체는 [메시지 서명 — 멱등성 키 요구 사항](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/#replay-protection)을 참고하세요.

## 프로토콜 동작 방식

### 상태 코드

| 상태 코드                   | 설명                                                          |
| --------------------------- | ------------------------------------------------------------- |
| `200 OK`                    | 요청이 성공했습니다.                                          |
| `201 Created`               | 장바구니가 생성되었습니다.                                    |
| `400 Bad Request`           | 요청이 유효하지 않거나 처리할 수 없습니다.                    |
| `401 Unauthorized`          | 인증이 필요한데 실패했거나 인증 정보가 없습니다.              |
| `403 Forbidden`             | 요청은 인증되었지만 사용자에게 필요한 권한이 없습니다.        |
| `409 Conflict`              | 충돌 때문에 요청을 완료할 수 없습니다(예: 멱등성 키 재사용).  |
| `422 Unprocessable Entity`  | 프로필 내용의 형식이 잘못되었습니다(디스커버리 실패).         |
| `424 Failed Dependency`     | 프로필 URL은 유효하지만 가져오지 못했습니다(디스커버리 실패). |
| `429 Too Many Requests`     | 요청 한도를 초과했습니다.                                     |
| `500 Internal Server Error` | 서버에서 예기치 않은 상황이 발생했습니다.                     |
| `503 Service Unavailable`   | 일시적으로 사용할 수 없습니다.                                |

### 에러 응답

전체 에러 코드 레지스트리와 전송 방식 바인딩 예시는 [핵심 명세](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#error-handling)를 참고하세요.

- **프로토콜 에러**: 적절한 HTTP 상태 코드(401, 403, 409, 429, 503)와 함께 `code`와 `content`를 담은 JSON 본문을 반환합니다.
- **비즈니스 결과**: UCP 엔벨로프와 `messages` 배열을 담아 HTTP 200을 반환합니다.

#### 비즈니스 결과

비즈니스 결과(찾을 수 없음, 유효성 검사 에러 포함)는 HTTP 200과 함께 `messages`를 담은 UCP 엔벨로프로 반환합니다.

```json
{
  "ucp": {
    "version": "2026-08-25",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.cart": [{"version": "2026-08-25"}]
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

## 보안 고려 사항

### 인증

인증은 선택 사항이며 비즈니스 요구 사항에 따라 달라집니다. 인증이 필요하면 REST 전송 방식에서 다음 방법을 사용할 수 있습니다(**MAY**).

1. **Open API**: 공개 작업에는 인증이 필요하지 않습니다.
1. **API 키**: `X-API-Key` 헤더로 전달합니다.
1. **OAuth 2.0**: `Authorization: Bearer {token}` 헤더로 전달합니다. 에이전트 인증 접근에서는 플랫폼을 식별하고, 사용자 인증 접근에서는 플랫폼과 사용자를 모두 식별합니다([신원 연결](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/identity-linking/index.md) 참고).
1. **Mutual TLS**: 높은 보안이 필요한 환경에서 사용합니다.

비즈니스는 일부 작업에는 인증을 요구하고 나머지 작업은 공개로 둘 수 있습니다(**MAY**) (예: 인증 없는 공개 장바구니).
