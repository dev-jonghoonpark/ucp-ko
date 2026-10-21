# 장바구니 기능 - REST 바인딩

이 문서는 [장바구니 기능](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/index.md)의 REST 바인딩을 명세합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필에서 REST 전송 방식 지원 여부를 알립니다.

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
      "dev.ucp.shopping.checkout": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/shopping/checkout",
          "schema": "https://ucp.dev/draft/schemas/shopping/checkout.json"
        }
      ],
      "dev.ucp.shopping.cart": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/shopping/cart",
          "schema": "https://ucp.dev/draft/schemas/shopping/cart.json"
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

| 이름        | 타입                                                                         | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | ---------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| line_items  | Array\[[Line Item](/ucp-ko/draft/draft/specification/reference/#line-item)\] | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)              | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)              | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)      | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer       | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                  | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |

#### 출력 스키마

| 이름         | 타입                                                                                              | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | [UCP Cart Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-cart-schema) | **필수**  | 장바구니 응답용 UCP 메타데이터. 체크아웃 전에는 결제 핸들러가 필요 없습니다.                                                                                                                                                                                                                                                                            |
| id           | string                                                                                            | **필수**  | 고유 장바구니 식별자.                                                                                                                                                                                                                                                                                                                                   |
| line_items   | Array\[[Line Item Response](/ucp-ko/draft/draft/specification/reference/#line-item)\]             | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                                   | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                                   | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution  | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                           | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer        | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                                       | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |
| currency     | string                                                                                            | **필수**  | ISO 4217 통화 코드. 판매자가 컨텍스트나 geo-IP를 바탕으로 결정합니다.                                                                                                                                                                                                                                                                                   |
| totals       | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                     | **필수**  | 예상 비용 내역. 배송비·세금을 아직 계산할 수 없으면 일부만 포함될 수 있습니다.                                                                                                                                                                                                                                                                          |
| actions      | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                   | 선택      | 이 장바구니에 대해 확장이 정의한 미해결 Action.                                                                                                                                                                                                                                                                                                         |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                          | 선택      | 검증 메시지, 경고 또는 안내성 알림.                                                                                                                                                                                                                                                                                                                     |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]                                | 선택      | 판매자 링크(선택 사항, 정책, FAQ).                                                                                                                                                                                                                                                                                                                      |
| policies     | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                            | 선택      | 이 장바구니의 항목에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다.                                                                                                                                                                                                 |
| continue_url | string                                                                                            | 선택      | 장바구니 핸드오프와 세션 복구용 URL. 공유와 사람이 개입하는(human-in-the-loop) 흐름을 지원합니다.                                                                                                                                                                                                                                                       |
| expires_at   | string                                                                                            | 선택      | 장바구니 만료 타임스탬프(RFC 3339). 선택 사항.                                                                                                                                                                                                                                                                                                          |

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
    "version": "draft",
    "capabilities": {
      "dev.ucp.shopping.checkout": [{"version": "draft"}],
      "dev.ucp.shopping.cart": [{"version": "draft"}]
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
  "ucp": { "version": "draft", "status": "error" },
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

| 이름         | 타입                                                                                              | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | [UCP Cart Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-cart-schema) | **필수**  | 장바구니 응답용 UCP 메타데이터. 체크아웃 전에는 결제 핸들러가 필요 없습니다.                                                                                                                                                                                                                                                                            |
| id           | string                                                                                            | **필수**  | 고유 장바구니 식별자.                                                                                                                                                                                                                                                                                                                                   |
| line_items   | Array\[[Line Item Response](/ucp-ko/draft/draft/specification/reference/#line-item)\]             | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                                   | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                                   | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution  | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                           | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer        | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                                       | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |
| currency     | string                                                                                            | **필수**  | ISO 4217 통화 코드. 판매자가 컨텍스트나 geo-IP를 바탕으로 결정합니다.                                                                                                                                                                                                                                                                                   |
| totals       | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                     | **필수**  | 예상 비용 내역. 배송비·세금을 아직 계산할 수 없으면 일부만 포함될 수 있습니다.                                                                                                                                                                                                                                                                          |
| actions      | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                   | 선택      | 이 장바구니에 대해 확장이 정의한 미해결 Action.                                                                                                                                                                                                                                                                                                         |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                          | 선택      | 검증 메시지, 경고 또는 안내성 알림.                                                                                                                                                                                                                                                                                                                     |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]                                | 선택      | 판매자 링크(선택 사항, 정책, FAQ).                                                                                                                                                                                                                                                                                                                      |
| policies     | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                            | 선택      | 이 장바구니의 항목에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다.                                                                                                                                                                                                 |
| continue_url | string                                                                                            | 선택      | 장바구니 핸드오프와 세션 복구용 URL. 공유와 사람이 개입하는(human-in-the-loop) 흐름을 지원합니다.                                                                                                                                                                                                                                                       |
| expires_at   | string                                                                                            | 선택      | 장바구니 만료 타임스탬프(RFC 3339). 선택 사항.                                                                                                                                                                                                                                                                                                          |

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
    "version": "draft",
    "capabilities": {
      "dev.ucp.shopping.checkout": [{"version": "draft"}],
      "dev.ucp.shopping.cart": [{"version": "draft"}]
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
    "version": "draft",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.cart": [{"version": "draft"}]
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

| 이름        | 타입                                                                         | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | ---------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| line_items  | Array\[[Line Item](/ucp-ko/draft/draft/specification/reference/#line-item)\] | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context     | [Context](/ucp-ko/draft/draft/specification/reference/#context)              | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals     | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)              | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)      | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer       | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                  | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |

#### 출력 스키마

| 이름         | 타입                                                                                              | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | [UCP Cart Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-cart-schema) | **필수**  | 장바구니 응답용 UCP 메타데이터. 체크아웃 전에는 결제 핸들러가 필요 없습니다.                                                                                                                                                                                                                                                                            |
| id           | string                                                                                            | **필수**  | 고유 장바구니 식별자.                                                                                                                                                                                                                                                                                                                                   |
| line_items   | Array\[[Line Item Response](/ucp-ko/draft/draft/specification/reference/#line-item)\]             | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                                   | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                                   | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution  | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                           | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer        | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                                       | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |
| currency     | string                                                                                            | **필수**  | ISO 4217 통화 코드. 판매자가 컨텍스트나 geo-IP를 바탕으로 결정합니다.                                                                                                                                                                                                                                                                                   |
| totals       | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                     | **필수**  | 예상 비용 내역. 배송비·세금을 아직 계산할 수 없으면 일부만 포함될 수 있습니다.                                                                                                                                                                                                                                                                          |
| actions      | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                   | 선택      | 이 장바구니에 대해 확장이 정의한 미해결 Action.                                                                                                                                                                                                                                                                                                         |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                          | 선택      | 검증 메시지, 경고 또는 안내성 알림.                                                                                                                                                                                                                                                                                                                     |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]                                | 선택      | 판매자 링크(선택 사항, 정책, FAQ).                                                                                                                                                                                                                                                                                                                      |
| policies     | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                            | 선택      | 이 장바구니의 항목에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다.                                                                                                                                                                                                 |
| continue_url | string                                                                                            | 선택      | 장바구니 핸드오프와 세션 복구용 URL. 공유와 사람이 개입하는(human-in-the-loop) 흐름을 지원합니다.                                                                                                                                                                                                                                                       |
| expires_at   | string                                                                                            | 선택      | 장바구니 만료 타임스탬프(RFC 3339). 선택 사항.                                                                                                                                                                                                                                                                                                          |

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
    "version": "draft",
    "capabilities": {
      "dev.ucp.shopping.checkout": [{"version": "draft"}],
      "dev.ucp.shopping.cart": [{"version": "draft"}]
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

| 이름         | 타입                                                                                              | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | [UCP Cart Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-cart-schema) | **필수**  | 장바구니 응답용 UCP 메타데이터. 체크아웃 전에는 결제 핸들러가 필요 없습니다.                                                                                                                                                                                                                                                                            |
| id           | string                                                                                            | **필수**  | 고유 장바구니 식별자.                                                                                                                                                                                                                                                                                                                                   |
| line_items   | Array\[[Line Item Response](/ucp-ko/draft/draft/specification/reference/#line-item)\]             | **필수**  | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                                   | 선택      | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                                   | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution  | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                           | 선택      | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer        | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                                       | 선택      | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |
| currency     | string                                                                                            | **필수**  | ISO 4217 통화 코드. 판매자가 컨텍스트나 geo-IP를 바탕으로 결정합니다.                                                                                                                                                                                                                                                                                   |
| totals       | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                     | **필수**  | 예상 비용 내역. 배송비·세금을 아직 계산할 수 없으면 일부만 포함될 수 있습니다.                                                                                                                                                                                                                                                                          |
| actions      | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                   | 선택      | 이 장바구니에 대해 확장이 정의한 미해결 Action.                                                                                                                                                                                                                                                                                                         |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                          | 선택      | 검증 메시지, 경고 또는 안내성 알림.                                                                                                                                                                                                                                                                                                                     |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]                                | 선택      | 판매자 링크(선택 사항, 정책, FAQ).                                                                                                                                                                                                                                                                                                                      |
| policies     | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                            | 선택      | 이 장바구니의 항목에 적용되는 정책(예: 반품/환불 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다.                                                                                                                                                                                                 |
| continue_url | string                                                                                            | 선택      | 장바구니 핸드오프와 세션 복구용 URL. 공유와 사람이 개입하는(human-in-the-loop) 흐름을 지원합니다.                                                                                                                                                                                                                                                       |
| expires_at   | string                                                                                            | 선택      | 장바구니 만료 타임스탬프(RFC 3339). 선택 사항.                                                                                                                                                                                                                                                                                                          |

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
    "version": "draft",
    "capabilities": {
      "dev.ucp.shopping.checkout": [{"version": "draft"}],
      "dev.ucp.shopping.cart": [{"version": "draft"}]
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

| 헤더              | 필수   | 설명                                                                                                                                                                                                                                                               |
| ----------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Authorization`   | 아니요 | 다음 두 방식을 나타내는 OAuth 토큰을 담아야 합니다. 1. 플랫폼 자체 인증(client_credentials). 2. 최종 사용자를 대신한 플랫폼 인증(authorization_code).                                                                                                              |
| `X-API-Key`       | 아니요 | 비즈니스가 플랫폼에 할당한 재사용 가능한 API 키로 플랫폼을 인증합니다.                                                                                                                                                                                             |
| `Signature`       | 아니요 | RFC 9421 HTTP Message Signature입니다. 인증에 HTTP Message Signatures를 사용할 때 필수입니다. 형식: `sig1=:<base64-signature>:`.                                                                                                                                   |
| `Signature-Input` | 아니요 | RFC 9421 Signature-Input 헤더입니다. 인증에 HTTP Message Signatures를 사용할 때 필수입니다. 형식: `sig1=("@method" "@path" ...);created=<timestamp>;keyid="<key-id>"`.                                                                                             |
| `Content-Digest`  | 아니요 | RFC 9530 본문 다이제스트. 본문이 있는 요청/응답에는 필수입니다. 형식: `sha-256=:<base64-digest>:`.                                                                                                                                                                 |
| `Idempotency-Key` | **예** | 재시도 중에 같은 작업이 중복 실행되지 않도록 보장합니다.                                                                                                                                                                                                           |
| `Request-Id`      | **예** | 네트워크 계층과 구성 요소 전반에서 요청을 추적하기 위한 값.                                                                                                                                                                                                        |
| `User-Agent`      | 아니요 | 호출하는 사용자 에이전트 문자열을 식별합니다.                                                                                                                                                                                                                      |
| `UCP-Agent`       | **예** | 호출하는 UCP 에이전트를 식별합니다. 모든 요청은 반드시 RFC 8941 Dictionary 구문으로 서명자의 프로필 URI를 담은 UCP-Agent 헤더를 포함해야 합니다(MUST). URL은 반드시 /.well-known/ucp를 가리켜야 합니다(MUST). 형식: profile="https://example.com/.well-known/ucp". |
| `Content-Type`    | 아니요 | 표현 메타데이터(Representation Metadata)입니다. 메시지 본문의 데이터가 실제로 무엇인지 수신자에게 알려 줍니다.                                                                                                                                                     |
| `Accept`          | 아니요 | 콘텐츠 협상. 클라이언트가 이해할 수 있는 데이터 형식을 서버에 알립니다.                                                                                                                                                                                            |
| `Accept-Language` | 아니요 | 현지화. 사용자가 선호하는 자연어를 수신자에게 알리며, 흔히 "가중치"나 우선순위가 함께 붙습니다.                                                                                                                                                                    |
| `Accept-Encoding` | 아니요 | 압축. 클라이언트가 지원하는 콘텐츠 인코딩(주로 압축)을 서버에 알립니다.                                                                                                                                                                                            |

### 개별 헤더 요구 사항

- **UCP-Agent**: All requests **MUST** include the `UCP-Agent` header containing the platform profile URI using Dictionary Structured Field syntax ([RFC 8941](https://datatracker.ietf.org/doc/html/rfc8941)). Format: `profile="https://platform.example/profile"`.
- **Idempotency-Key**: Operations that modify state **SHOULD** support idempotency. When provided, the server **MUST**:
  1. Store the key with the operation result for at least 24 hours.
  1. Return the cached result for duplicate keys whose request body matches the original.
  1. Return `409 Conflict` if the key is reused with a mismatched body. See [Message Signatures — Idempotency Key Requirements](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/signatures/#replay-protection) for the full payload-matching contract.

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

전체 에러 코드 레지스트리와 전송 방식 바인딩 예시는 [핵심 명세](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#error-handling)를 참고하세요.

- **프로토콜 에러**: 적절한 HTTP 상태 코드(401, 403, 409, 429, 503)와 함께 `code`와 `content`를 담은 JSON 본문을 반환합니다.
- **비즈니스 결과**: UCP 엔벨로프와 `messages` 배열을 담아 HTTP 200을 반환합니다.

#### 비즈니스 결과

비즈니스 결과(찾을 수 없음, 유효성 검사 에러 포함)는 HTTP 200과 함께 `messages`를 담은 UCP 엔벨로프로 반환합니다.

```json
{
  "ucp": {
    "version": "draft",
    "status": "error",
    "capabilities": {
      "dev.ucp.shopping.cart": [{"version": "draft"}]
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
1. **OAuth 2.0**: `Authorization: Bearer {token}` 헤더로 전달합니다. 에이전트 인증 접근에서는 플랫폼을 식별하고, 사용자 인증 접근에서는 플랫폼과 사용자를 모두 식별합니다([신원 연결](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/identity-linking/index.md) 참고).
1. **Mutual TLS**: 높은 보안이 필요한 환경에서 사용합니다.

비즈니스는 일부 작업에는 인증을 요구하고 나머지 작업은 공개로 둘 수 있습니다(**MAY**) (예: 인증 없는 공개 장바구니).
