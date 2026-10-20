# 장바구니 기능

- **기능 이름:** `dev.ucp.shopping.cart`

## 개요

장바구니(Cart) 기능을 사용하면 체크아웃처럼 복잡한 절차 없이 장바구니를 구성할 수 있습니다. [체크아웃](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/checkout/index.md)이 결제 핸들러, 상태 수명 주기, 주문 확정을 관리한다면, 장바구니는 구매 의사가 확정되기 전에 상품을 모아 두는 가벼운 CRUD 인터페이스를 제공합니다.

**장바구니와 체크아웃 중 무엇을 쓸까:**

- **장바구니**: 사용자가 상품을 둘러보고, 비교하고, 나중을 위해 저장하는 단계입니다. 결제 설정이 필요 없습니다. 플랫폼/에이전트가 상품을 자유롭게 추가, 삭제, 수정할 수 있습니다.
- **체크아웃**: 사용자가 구매 의사를 밝힌 단계입니다. 결제 핸들러가 설정되고 상태 수명 주기가 시작되며, 세션이 완료를 향해 진행됩니다.

일반적인 흐름: `cart session` → `checkout session` → `order`

장바구니가 지원하는 기능:

- **점진적 구성**: 여러 세션에 걸쳐 상품 추가/삭제
- **현지화된 예상 금액**: 전체 체크아웃 부담 없이 컨텍스트를 반영한 가격 책정
- **공유**: `continue_url`로 장바구니 공유와 복구 가능

## 장바구니와 체크아웃 비교

| 항목          | 장바구니                    | 체크아웃                              |
| ------------- | --------------------------- | ------------------------------------- |
| **목적**      | 구매 전 탐색                | 구매 확정                             |
| **결제**      | 없음                        | 필수(핸들러, 결제 수단)               |
| **상태**      | 이진(존재/찾을 수 없음)     | 수명 주기(`incomplete` → `completed`) |
| **완료 작업** | 없음                        | 있음                                  |
| **합계**      | 예상치(일부만 있을 수 있음) | 최종 가격                             |

## 장바구니에서 체크아웃으로 전환

장바구니 기능이 협상되었다면, 플랫폼은 체크아웃 생성(Create Checkout) 요청에 `cart_id`를 넣어 장바구니를 체크아웃으로 전환할 수 있습니다. 체크아웃 세션은 장바구니 내용(`line_items`, `context`, `buyer`)으로 초기화됩니다.

```json
{
  "cart_id": "cart_abc123",
  "line_items": []
}
```

비즈니스는 반드시 장바구니 내용을 사용해야 하며(MUST), 체크아웃 페이로드에서 겹치는 필드는 반드시 무시해야 합니다(MUST). `cart_id` 매개변수는 비즈니스 프로필에 장바구니 기능이 선언된 경우에만 사용할 수 있습니다.

**멱등 전환:**

해당 `cart_id`에 완료되지 않은 체크아웃이 이미 있으면, 비즈니스는 새로 만들지 말고 반드시 기존 체크아웃 세션을 반환해야 합니다(MUST). 이렇게 하면 장바구니마다 활성 체크아웃이 하나만 유지되어 세션 충돌을 막을 수 있습니다.

**전환 후 장바구니 수명 주기:**

`cart_id`로 체크아웃을 초기화했다면, 장바구니와 체크아웃 세션은 체크아웃이 진행되는 동안 연결된 상태여야 합니다(SHOULD).

- **체크아웃 진행 중** — 비즈니스는 장바구니를 유지하고, 관련된 체크아웃 수정 사항(수량 변경, 상품 삭제)을 장바구니에 다시 반영해야 합니다(SHOULD). 그래야 구매자가 체크아웃과 스토어프런트를 오가는 스토어프런트 복귀 흐름을 지원할 수 있습니다.
- **체크아웃 완료 후** — 비즈니스는 TTL, 체크아웃 완료, 기타 비즈니스 로직에 따라 장바구니를 비울 수 있습니다(MAY). 비워진 장바구니 ID로 이후 작업을 하면 `not_found`가 반환되며, 플랫폼은 `create_cart`로 새 세션을 시작할 수 있습니다.

## 수량과 판매 기준

장바구니 라인 아이템은 공통 [수량과 단위](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#quantities-and-units) 규약을 따릅니다. 각 `line_items[].quantity`는 상품의 권한 있는 판매 기준에 따른 정수 단계 수입니다. 비즈니스 응답에 `line_items[].item.quantity_unit`이 없으면 기본값인 `each` 기준이라는 뜻이므로 `quantity`는 상품 개수를 나타냅니다.

플랫폼 요청에서 `line_items[].item.quantity_unit`을 생략하면 아무것도 주장하지 않은 것입니다. 비즈니스는 상품의 권한 있는 판매 기준으로 `quantity`를 해석하므로, 계량 단위로 판매되는 상품을 요청할 때도 `each`를 주장하는 일 없이 이 서술자를 생략할 수 있습니다. 판매 기준을 명시적으로 주장하려면 플랫폼은 `item.quantity_unit`을 포함할 수 있습니다(**MAY**).

판매 기준 디스커버리, 주장 일치 확인, 불일치 시 변환 또는 거부, 응답 반영, 주문 증분 처리, 라인 가격 책정은 [체크아웃 — 수량과 판매 기준](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/checkout/#quantity-and-sale-basis)을 따릅니다. 장바구니 합계는 여전히 예상치입니다([합계](#total) 참고). 계량 단위 라인은 장바구니 요약에서 라인 아이템 하나로 계산되며, 이때 `quantity`는 상품 개수가 아니라 양입니다.

## 액션

장바구니는 처리되지 않은 액션(Action) 인스턴스를 응답 전용 `actions` 맵으로 노출합니다. 이 맵은 [개요 — 액션](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#actions)에 정의되어 있습니다.

장바구니에는 상태 수명 주기가 없습니다. 각 액션은 해당 액션 유형에 지정된 장바구니 효과만 제한합니다. 비즈니스는 처리되지 않은 액션이 있다는 이유로 관련 없는 장바구니 작업을 거부해서는 안 됩니다(**MUST NOT**). 플랫폼은 액션이 처리되지 않은 동안에도 상품을 계속 추가, 삭제, 수정할 수 있습니다(**MAY**).

액션을 처리한 뒤 플랫폼은 [장바구니 조회](#get-cart)나 이후의 수정 응답으로 최신 장바구니를 가져와야 합니다(**SHOULD**).

## 스코프

장바구니 기능은 사용자 인증 접근용으로 다음과 같은 잘 알려진 스코프를 정의합니다.

| 스코프                         | 설명                                                                  |
| ------------------------------ | --------------------------------------------------------------------- |
| `dev.ucp.shopping.cart:manage` | 인증된 사용자를 대신하는 모든 장바구니 작업 — 생성, 조회, 수정, 저장. |

스코프 선언과 도출, 사용자 정의 스코프로 이 집합을 확장하는 규칙은 [신원 연결 — 스코프](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/common/identity-linking/#scopes)에 정의되어 있습니다.

## 지침

### 플랫폼

- 구매 전 탐색과 세션 유지에 장바구니를 사용할 수 있습니다(**MAY**).
- 사용자가 구매 의사를 밝히면 장바구니를 체크아웃으로 전환해야 합니다(**SHOULD**).
- 비즈니스 UI로 넘기기 위해 `continue_url`을 표시할 수 있습니다(**MAY**).
- 장바구니가 만료되거나 취소되었을 때 반환되는 `not_found`를 적절히 처리해야 합니다(**SHOULD**).

### 비즈니스

- 장바구니 인계와 세션 복구를 위해 `continue_url`을 제공해야 합니다(**SHOULD**).
- TODO: `continue_url` 목적지 논의 필요 - 장바구니 vs 체크아웃.
- 계산할 수 있으면 예상 합계를 제공해야 합니다(**SHOULD**).
- 주소를 모르면 체크아웃 전까지 풀필먼트 합계를 생략할 수 있습니다(**MAY**).
- 유효성 검사 경고가 있으면 정보성 메시지를 반환해야 합니다(**SHOULD**).
- `expires_at`으로 장바구니 만료 시점을 설정할 수 있습니다(**MAY**).
- `cart_id`로 체크아웃을 초기화한 경우 [장바구니 수명 주기 요구 사항](#cart-to-checkout-conversion)을 따라야 합니다(**SHOULD**).

## 장바구니 스키마 정의

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

## 작업

장바구니 기능에서 정의하는 논리적 작업은 다음과 같습니다.

| 작업                           | 설명                                    |
| ------------------------------ | --------------------------------------- |
| **장바구니 생성(Create Cart)** | 새 장바구니 세션을 만듭니다.            |
| **장바구니 조회(Get Cart)**    | 장바구니 세션의 현재 상태를 가져옵니다. |
| **장바구니 수정(Update Cart)** | 장바구니 세션을 수정합니다.             |
| **장바구니 취소(Cancel Cart)** | 장바구니 세션을 취소합니다.             |

### 장바구니 생성

라인 아이템과 선택적 구매자/컨텍스트 정보로 새 장바구니 세션을 만듭니다. 구매자/컨텍스트 정보는 현지화된 예상 가격을 계산하는 데 쓰입니다.

요청한 상품을 **하나도** 구매할 수 없으면, 비즈니스는 장바구니 리소스를 만드는 대신 오류 응답을 반환할 수 있습니다(MAY). 주 판별 기준은 `ucp.status`이며, `id`가 없다는 점도 일관된 보조 지표가 됩니다.

```json
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

- [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/cart/rest/#create-cart)
- [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/cart/mcp/#create_cart)

### 장바구니 조회

장바구니 세션의 최신 상태를 가져옵니다. 장바구니가 없거나, 만료되었거나, 취소되었으면 `not_found`를 반환합니다.

- [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/cart/rest/#get-cart)
- [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/cart/mcp/#get_cart)

### 장바구니 수정

장바구니 세션을 통째로 교체합니다. 플랫폼은 반드시 장바구니 리소스 전체를 보내야 합니다(**MUST**). 보낸 리소스가 비즈니스 측의 기존 장바구니 상태를 대체합니다.

- [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/cart/rest/#update-cart)
- [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/cart/mcp/#update_cart)

### 장바구니 취소

장바구니 세션을 취소합니다. 비즈니스는 반드시 삭제 전의 장바구니 상태를 반환해야 합니다(MUST). 이후 이 장바구니 ID로 들어오는 작업에는 `not_found`를 반환해야 합니다(SHOULD).

- [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/cart/rest/#cancel-cart)
- [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/cart/mcp/#cancel_cart)

## 엔티티

장바구니는 [체크아웃](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/checkout/index.md)과 같은 엔티티 스키마를 재사용합니다. 덕분에 장바구니를 체크아웃 세션으로 전환해도 데이터 구조가 일관되게 유지됩니다.

### UCP 응답 장바구니

UCP metadata for cart responses. No payment handlers needed pre-checkout.

| 이름             | 타입   | 필수 여부    | 설명                                                                                                                                           |
| ---------------- | ------ | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| version          | string | **Required** | Version identifier in YYYY-MM-DD format.                                                                                                       |
| map_order        | object | Optional     | Preferred key-traversal order for sibling registry fields inside the root `ucp` envelope (`services`, `capabilities`, and `payment_handlers`). |
| status           | string | Optional     | Application-level status of the UCP operation. **Enum:** `success`, `error`                                                                    |
| services         | object | Optional     | Service registry keyed by reverse-domain name.                                                                                                 |
| capabilities     | object | Optional     | Capability registry keyed by reverse-domain name.                                                                                              |
| payment_handlers | object | Optional     | Payment handler registry keyed by reverse-domain name.                                                                                         |
| capabilities     | any    | Optional     |                                                                                                                                                |

### 라인 아이템

#### 라인 아이템 생성 요청

| 이름     | 타입                                                                | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                     |
| -------- | ------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| item     | [Item](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#item) | **Required** |                                                                                                                                                                                                                                                                                                                                                          |
| quantity | integer                                                             | **Required** | Always an integer step count. On Platform requests, steps use the item's Business-authoritative sale basis; omitting `item.quantity_unit` makes no assertion and does not imply `each`. On Business responses, `item.quantity_unit` describes the basis; if absent, it encodes the `each` machine identity (`C62`, 0) and `quantity` counts whole items. |

#### 라인 아이템 수정 요청

| 이름      | 타입                                                                | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                     |
| --------- | ------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id        | string                                                              | Optional     |                                                                                                                                                                                                                                                                                                                                                          |
| item      | [Item](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#item) | **Required** |                                                                                                                                                                                                                                                                                                                                                          |
| quantity  | integer                                                             | **Required** | Always an integer step count. On Platform requests, steps use the item's Business-authoritative sale basis; omitting `item.quantity_unit` makes no assertion and does not imply `each`. On Business responses, `item.quantity_unit` describes the basis; if absent, it encodes the `each` machine identity (`C62`, 0) and `quantity` counts whole items. |
| parent_id | string                                                              | Optional     | Parent line item identifier for any nested structures.                                                                                                                                                                                                                                                                                                   |

#### 라인 아이템

| 이름      | 타입                                                                                    | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                     |
| --------- | --------------------------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id        | string                                                                                  | **Required** |                                                                                                                                                                                                                                                                                                                                                          |
| item      | [Item](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#item)                     | **Required** |                                                                                                                                                                                                                                                                                                                                                          |
| quantity  | integer                                                                                 | **Required** | Always an integer step count. On Platform requests, steps use the item's Business-authoritative sale basis; omitting `item.quantity_unit` makes no assertion and does not imply `each`. On Business responses, `item.quantity_unit` describes the basis; if absent, it encodes the `each` machine identity (`C62`, 0) and `quantity` counts whole items. |
| totals    | Array\[[Total Response](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#total)\] | **Required** | Line item totals breakdown.                                                                                                                                                                                                                                                                                                                              |
| parent_id | string                                                                                  | Optional     | Parent line item identifier for any nested structures.                                                                                                                                                                                                                                                                                                   |

#### 상품

| 이름          | 타입                                                                                  | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------- | ------------------------------------------------------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id            | string                                                                                | **Required** | The product identifier, often the SKU, required to resolve the product details associated with this line item. Should be recognized by both the Platform, and the Business.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| title         | string                                                                                | **Required** | Product title.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| price         | [Amount](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#amount)               | **Required** | Unit price in ISO 4217 minor units. Price is the amount per one whole `quantity_unit.unit` (for example, per lb or per hour); when `quantity_unit` is absent, it is per `each`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| quantity_unit | [Quantity Unit](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#quantity-unit) | Optional     | Sale basis this item's `quantity` is denominated in. On an authoritative Business response, absence encodes the default `each` machine identity (`C62`, 0); the Business MUST include this descriptor for every non-`each` response. On Platform requests, omission makes no assertion: the Business interprets `quantity` using the item's authoritative sale basis. If the Platform includes this descriptor, it asserts the unit-descriptor machine identity. The Business MUST compare that machine identity (`unit`, effective `scale`), ignore `display_text` and `increment`, and resolve a mismatch by conversion surfaced as a visible line revision with a warning, or by rejection with a recoverable business outcome; silent reinterpretation is forbidden. An explicit `C62` descriptor at effective scale 0 matches an authoritative basis represented by an absent descriptor. |
| unit_price    | [Unit Price](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#unit-price)       | Optional     | Pricing basis for this item. On an authoritative Business response, the Business MUST include `unit_price` on every line whose pricing basis differs from its sale basis (for example, priced per pound but sold per `each`); presence on a line marks the rate as transactional rather than display-only. When the pricing basis is the sale basis, `item.price` fully denominates the charge and this field MAY be omitted.                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| image_url     | string                                                                                | Optional     | Product image URI.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |

### 구매자

| 이름         | 타입   | 필수 여부 | 설명                     |
| ------------ | ------ | --------- | ------------------------ |
| first_name   | string | Optional  | First name of the buyer. |
| last_name    | string | Optional  | Last name of the buyer.  |
| email        | string | Optional  | Email of the buyer.      |
| phone_number | string | Optional  | E.164 standard.          |

### 컨텍스트

| 이름            | 타입                                                                                                       | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| --------------- | ---------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| address_country | string                                                                                                     | Optional  | The country, as a 2-letter ISO 3166-1 alpha-2 code (e.g. "US"). A 3-letter alpha-3 code or full country name MAY also be used.                                                                                                                                                                                                                                                                                                                                     |
| address_region  | string                                                                                                     | Optional  | The first-level administrative region within the country (e.g. a state or province such as California).                                                                                                                                                                                                                                                                                                                                                            |
| postal_code     | string                                                                                                     | Optional  | The postal code (e.g. "94043").                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| location        | string                                                                                                     | Optional  | Stable, opaque identifier for a Location in the Business's namespace. This provisional, non-binding hint is distinct from the Buyer's locality. The operation specification or an active capability/extension defines its effects. A common example in retail shopping is the default home store ID selected and saved by the user when purchasing groceries.                                                                                                      |
| intent          | string                                                                                                     | Optional  | Background context describing buyer's intent (e.g., 'looking for a gift under $50', 'need something durable for outdoor use'). Informs relevance, recommendations, and personalization.                                                                                                                                                                                                                                                                            |
| language        | string                                                                                                     | Optional  | Preferred language for content. Use IETF BCP 47 language tags (e.g., 'en', 'fr-CA', 'zh-Hans'). For REST, equivalent to Accept-Language header—platforms SHOULD fall back to Accept-Language when this field is absent; when provided, overrides Accept-Language. Businesses MAY return content in a different language if unavailable.                                                                                                                            |
| currency        | string                                                                                                     | Optional  | Preferred currency (ISO 4217, e.g., 'EUR', 'USD'). Businesses determine presentment currency from context and authoritative signals; this hint MAY inform selection in multi-currency markets. Also serves as the denomination for price filter values — platforms SHOULD include this field when sending price filters. Response prices include explicit currency confirming the resolution.                                                                      |
| eligibility     | Array\[[Reverse Domain Name](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#reverse-domain-name)\] | Optional  | Buyer claims about eligible benefits such as loyalty membership, payment instrument perks, and similar. Recognized claims MAY inform the Business response (e.g., member-only product availability, adjusted pricing in catalog, provisional discounts at cart or checkout). Businesses MUST ignore unrecognized values without error. Values MUST use reverse-domain naming (e.g., 'com.example.loyalty_gold', 'org.school.student') and MUST be non-identifying. |
| payment         | Array[object]                                                                                              | Optional  | Buyer-preferred payment handlers in priority order (most preferred first). Each entry names a handler advertised in the Business profile's `ucp.payment_handlers`, optionally narrowed to preferred instrument types. The Business SHOULD use it to preselect or prioritize the handler (and type, when given) and MAY ignore unavailable or ineligible entries; unrecognized values MUST be ignored without error.                                                |

### 신호

인가와 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터입니다. 신호 값은 구매자가 스스로 주장한 내용이어서는 안 됩니다(MUST NOT). 자세한 내용과 개인정보 보호 요구 사항은 [신호](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#signals)를 참고하세요.

| 이름               | 타입   | 필수 여부 | 설명                                           |
| ------------------ | ------ | --------- | ---------------------------------------------- |
| dev.ucp.buyer_ip   | string | Optional  | Client's IP address (IPv4 or IPv6).            |
| dev.ucp.user_agent | string | Optional  | Client's HTTP User-Agent header or equivalent. |

### 어트리뷰션

플랫폼이 제공하는 유입 경로와 전환 이벤트 컨텍스트입니다. 플랫폼이 전달하는 캠페인 ID, 클릭 식별자, 소스/매체 표식이 여기에 해당합니다. 자세한 내용과 동의 요구 사항은 [어트리뷰션](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#attribution)을 참고하세요.

Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.

### 합계

장바구니와 체크아웃에는 같은 합계 규약이 적용됩니다. 렌더링 규약, 회계 항등식, 잘 알려진 유형, 반복 유형, 하위 라인 의미론은 [체크아웃 합계](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/checkout/#totals)를 참고하세요.

| 이름         | 타입                                                                                  | 필수 여부    | 설명                                                                                                                                                                                                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                                | **Required** | Cost category. Well-known values: subtotal, items_discount, discount, fulfillment, tax, fee, total. Businesses MAY use additional values.                                                                                                                                                   |
| display_text | string                                                                                | Optional     | Text to display against the amount. Should reflect appropriate method (e.g., 'Shipping', 'Delivery').                                                                                                                                                                                       |
| amount       | [Signed Amount](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#signed-amount) | **Required** | Monetary amount in the currency's minor unit as defined by ISO 4217. Refer to the currency's exponent to determine minor-to-major ratio (e.g., 2 for USD, 0 for JPY, 3 for KWD). May be negative — the sign is intrinsic to the value (e.g., discounts are negative, charges are positive). |

세금은 계산할 수 있으면 포함할 수 있습니다(MAY). 플랫폼은 장바구니 합계를 예상치로 보아야 하며(SHOULD), 정확한 세금은 체크아웃에서 계산됩니다.

### 메시지

This object MUST be one of the following types: [Message Error](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message-error), [Message Warning](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message-warning), [Message Info](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#message-info).

#### 메시지 오류

| 이름         | 타입                                                                            | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ------------ | ------------------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                          | **Required** | **Constant = error**. Message type discriminator.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| code         | [Error Code](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#error-code) | **Required** | Error code identifying the type of error. Standard errors are defined in capability specifications (see examples) and have standardized semantics; freeform codes are permitted.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| path         | string                                                                          | Optional     | RFC 9535 JSONPath to the component the message refers to (e.g., $.line_items[0]).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| content_type | string                                                                          | Optional     | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| content      | string                                                                          | **Required** | Human-readable message.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| severity     | string                                                                          | **Required** | Reflects the resource state and recommended action. 'recoverable': platform can resolve the condition in band, for example by modifying inputs or processing a related Action, and submit a new operation when needed. 'requires_buyer_input': merchant requires information their API doesn't support collecting programmatically (checkout incomplete). 'requires_buyer_review': buyer must authorize before order placement due to policy, regulatory, or entitlement rules. 'unrecoverable': no valid resource exists to act on, retry with new resource or inputs. Errors with 'requires\_*' severity contribute to 'status: requires_escalation'.* *Enum:*\* `recoverable`, `requires_buyer_input`, `requires_buyer_review`, `unrecoverable` |

#### 메시지 정보

| 이름         | 타입                                                                          | 필수 여부    | 설명                                                                                                                                                                                           |
| ------------ | ----------------------------------------------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                        | **Required** | **Constant = info**. Message type discriminator.                                                                                                                                               |
| path         | string                                                                        | Optional     | RFC 9535 JSONPath to the component the message refers to (e.g., $.line_items[0]).                                                                                                              |
| code         | [Info Code](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#info-code) | Optional     | Info code identifying the type of informational message. Standard codes are defined in capability specifications (see examples) and have standardized semantics; freeform codes are permitted. |
| content_type | string                                                                        | Optional     | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                                 |
| content      | string                                                                        | **Required** | Human-readable message.                                                                                                                                                                        |

#### 메시지 경고

| 이름         | 타입                                                                                | 필수 여부    | 설명                                                                                                                                                                                                                                                |
| ------------ | ----------------------------------------------------------------------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                              | **Required** | **Constant = warning**. Message type discriminator.                                                                                                                                                                                                 |
| path         | string                                                                              | Optional     | RFC 9535 JSONPath to the component the message refers to (e.g., $.line_items[0]).                                                                                                                                                                   |
| code         | [Warning Code](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#warning-code) | **Required** | Warning code identifying the type of warning. Standard codes are defined in capability specifications (see examples) and have standardized semantics; freeform codes are permitted.                                                                 |
| content      | string                                                                              | **Required** | Human-readable warning message that MUST be displayed.                                                                                                                                                                                              |
| content_type | string                                                                              | Optional     | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                                                                                      |
| presentation | string                                                                              | Optional     | Rendering contract for this warning. 'notice' (default): platform MUST display, MAY dismiss. 'disclosure': platform MUST display in proximity to the path-referenced component, MUST NOT hide or auto-dismiss. See specification for full contract. |
| image_url    | string                                                                              | Optional     | URL to a required visual element (e.g., warning symbol, energy class label).                                                                                                                                                                        |
| url          | string                                                                              | Optional     | Reference URL for more information (e.g., regulatory site, registry entry, policy page).                                                                                                                                                            |

### 링크

| 이름  | 타입   | 필수 여부    | 설명                                                                                                                                                                                                                                 |
| ----- | ------ | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| type  | string | **Required** | Type of link. Well-known values: `privacy_policy`, `terms_of_service`, `refund_policy`, `shipping_policy`, `faq`. Consumers SHOULD handle unknown values gracefully by displaying them using the `title` field or omitting the link. |
| url   | string | **Required** | The actual URL pointing to the content to be displayed.                                                                                                                                                                              |
| title | string | Optional     | Optional display text for the link. When provided, use this instead of generating from type.                                                                                                                                         |

### 정책

이 장바구니의 상품에 적용되는 정책(반품/환불 조건, 보증 등)입니다. `applies_to`의 JSONPath 대상은 이 응답의 루트를 기준으로 합니다(예: `$.line_items[0]`). 전체 모델은 [정책](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#policies)을 참고하세요.

| 이름        | 타입                                                                                              | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ----------- | ------------------------------------------------------------------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type        | [Reverse Domain Name](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#reverse-domain-name) | **Required** | Policy type discriminator. Open reverse-DNS vocabulary. Well-known values: `dev.ucp.shopping.policy.return` (return terms), `dev.ucp.shopping.policy.warranty` (warranty terms). Businesses MAY define custom types in their own domain (e.g., `com.example.policy.price_match`). Platforms MUST tolerate unknown values.                                                                                                                                                                                                                                                                                                                                                          |
| description | [Description](/ucp-ko/2026-08-25/2026-08-25/specification/reference/#description)                 | **Required** | Human-readable policy summary in one or more formats (plain, markdown, html). Required on every policy so a platform can present it without understanding any type-specific fields. This is not the buyer-facing disclosure — display is compelled by a `messages[]` warning (see the Policies section).                                                                                                                                                                                                                                                                                                                                                                           |
| applies_to  | Array[string]                                                                                     | Optional     | RFC 9535 JSONPath expressions identifying the nodes this policy applies to, relative to the embedding response root (e.g., `$.line_items[0]` in cart/checkout, `$.products[2]` in catalog). Each target covers the node it names and everything nested under it, so a target on a product also covers its variants. A singular query (RFC 9535 Section 2.3.5.1; name and index selectors only) names a single node; filters, wildcards, and slices match a set. When omitted, the policy applies to the entire response. When policies of the same `type` contest a node, the narrowest target wins and overrides the rest. See the Policies section for how specificity resolves. |
| url         | string                                                                                            | Optional     | Optional link to the full policy document.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
