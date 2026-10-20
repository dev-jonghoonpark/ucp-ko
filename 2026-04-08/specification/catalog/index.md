# 카탈로그 기능

## 개요

카탈로그 기능(Capability)을 사용하면 플랫폼이 비즈니스의 상품 카탈로그를 검색하고 탐색할 수 있습니다. 체크아웃 전에 상품을 찾는 단계로, 다음과 같은 사용 사례를 지원합니다.

- 자유 텍스트 상품 검색
- 카테고리 및 필터 기반 탐색
- 식별자로 상품/변형 일괄 조회
- 변형 간 가격 비교

## 기능

| 기능                                                                                                                                       | 설명                                    |
| ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------- |
| [`dev.ucp.shopping.catalog.search`](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/search/index.md) | 쿼리 텍스트와 필터로 상품을 검색합니다. |
| [`dev.ucp.shopping.catalog.lookup`](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/lookup/index.md) | 식별자로 상품 또는 변형을 조회합니다.   |

## 핵심 개념

- **상품(Product)**: 제목, 설명, 미디어와 하나 이상의 변형으로 구성된 카탈로그 항목입니다.
- **변형(Variant)**: 특정 옵션 선택(예: "Blue / Large"), 가격, 재고 상태가 정해진 구매 가능한 항목입니다.
- **가격(Price)**: 가격 값에는 금액(통화의 최소 단위)과 통화 코드가 모두 들어 있어 다중 통화 카탈로그를 지원합니다.

### 체크아웃과의 관계

카탈로그 작업은 체크아웃의 `line_items[].item.id`에 바로 쓸 수 있는 상품 ID와 변형 ID를 반환합니다. 카탈로그 조회로 얻은 변형 ID는 체크아웃이 기대하는 아이템 ID와 일치해야 합니다.

카탈로그 응답(가격, 재고 상태 등)은 해당 요청 시점의 비즈니스 조건을 반영하지만 거래상의 확약은 아닙니다. 권위는 체크아웃에 있습니다. 응답은 세션마다 다를 수 있으므로, 재검증 없이 다른 세션에서 재사용하지 않는 것이 좋습니다(**SHOULD NOT**).

## 공통 엔터티

### 컨텍스트

카탈로그 작업에 쓰는 위치 및 시장 컨텍스트입니다. 모든 필드는 관련성과 현지화를 위한 선택적 힌트입니다. 플랫폼은 요청 헤더로 컨텍스트를 지리적으로 감지할 수 있습니다(MAY).

컨텍스트 신호는 잠정적인 값일 뿐 권위 있는 데이터가 아닙니다. 비즈니스는 검증된 입력(예: 배송 주소)이 없을 때 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제상 제약(수출 통제)과 맞지 않으면 무시하거나 순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 이루어져야 합니다(MUST).

비즈니스는 컨텍스트 신호를 바탕으로 통화를 포함한 시장 배정을 결정합니다. 가격 필터 값은 `context.currency` 기준이며, 표시 통화가 다르면 비즈니스는 적용 전에 변환해야 합니다(SHOULD)([가격 필터](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/search/#price-filter) 참고). 응답 가격에는 결정 결과를 확인할 수 있도록 통화 코드가 명시됩니다.

`context.eligibility` 주장이 있고 비즈니스가 이를 수락했다면, 취소선 표시를 위해 `price` / `list_price`를 직접 조정할 수 있으며(**MAY**), `code: "eligibility_benefit"`이 담긴 `messages`로 조정 내용이 어떤 주장에 따른 것인지 밝힐 수 있습니다(**MAY**).

| 이름            | 타입                                                                                                       | 필수 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| --------------- | ---------------------------------------------------------------------------------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| address_country | string                                                                                                     | No   | The country. Recommended to be in 2-letter ISO 3166-1 alpha-2 format, for example "US". For backward compatibility, a 3-letter ISO 3166-1 alpha-3 country code such as "SGP" or a full country name such as "Singapore" can also be used. Optional hint for market context (currency, availability, pricing)—higher-resolution data (e.g., shipping address) supersedes this value.                                                                                |
| address_region  | string                                                                                                     | No   | The region in which the locality is, and which is in the country. For example, California or another appropriate first-level Administrative division. Optional hint for progressive localization—higher-resolution data (e.g., shipping address) supersedes this value.                                                                                                                                                                                            |
| postal_code     | string                                                                                                     | No   | The postal code. For example, 94043. Optional hint for regional refinement—higher-resolution data (e.g., shipping address) supersedes this value.                                                                                                                                                                                                                                                                                                                  |
| intent          | string                                                                                                     | No   | Background context describing buyer's intent (e.g., 'looking for a gift under $50', 'need something durable for outdoor use'). Informs relevance, recommendations, and personalization.                                                                                                                                                                                                                                                                            |
| language        | string                                                                                                     | No   | Preferred language for content. Use IETF BCP 47 language tags (e.g., 'en', 'fr-CA', 'zh-Hans'). For REST, equivalent to Accept-Language header—platforms SHOULD fall back to Accept-Language when this field is absent; when provided, overrides Accept-Language. Businesses MAY return content in a different language if unavailable.                                                                                                                            |
| currency        | string                                                                                                     | No   | Preferred currency (ISO 4217, e.g., 'EUR', 'USD'). Businesses determine presentment currency from context and authoritative signals; this hint MAY inform selection in multi-currency markets. Also serves as the denomination for price filter values — platforms SHOULD include this field when sending price filters. Response prices include explicit currency confirming the resolution.                                                                      |
| eligibility     | Array\[[Reverse Domain Name](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#reverse-domain-name)\] | No   | Buyer claims about eligible benefits such as loyalty membership, payment instrument perks, and similar. Recognized claims MAY inform the Business response (e.g., member-only product availability, adjusted pricing in catalog, provisional discounts at cart or checkout). Businesses MUST ignore unrecognized values without error. Values MUST use reverse-domain naming (e.g., 'com.example.loyalty_gold', 'org.school.student') and MUST be non-identifying. |

### 신호

인가와 악용 방지에 도움이 되도록 플랫폼이 제공하는 환경 데이터입니다. 신호 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 자세한 내용과 개인정보 보호 요건은 [신호](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/#signals)를 참고하세요.

| 이름               | 타입   | 필수 | 설명                                           |
| ------------------ | ------ | ---- | ---------------------------------------------- |
| dev.ucp.buyer_ip   | string | No   | Client's IP address (IPv4 or IPv6).            |
| dev.ucp.user_agent | string | No   | Client's HTTP User-Agent header or equivalent. |

### 어트리뷰션

플랫폼이 제공하는 유입 및 전환 이벤트 컨텍스트로, 플랫폼이 전달하는 캠페인 ID, 클릭 식별자, 소스/매체 표식 등이 해당합니다. 자세한 내용과 동의 요건은 [어트리뷰션](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/#attribution)을 참고하세요.

Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.

### 상품

판매 가능한 항목을 나타내는 카탈로그 항목으로, 구매 가능한 변형이 하나 이상 있습니다.

`media`와 `variants`는 순서가 있는 배열입니다. 비즈니스는 가장 관련성 높은 변형과 이미지를 맨 앞에 반환해야 합니다(SHOULD). 조회에서는 기본값을, 검색에서는 쿼리와 컨텍스트에 가장 잘 맞는 항목을 앞에 둡니다. 플랫폼은 첫 번째 요소를 대표 항목으로 취급해야 합니다(SHOULD).

| 이름             | 타입                                                                                             | 필수    | 설명                                                                                             |
| ---------------- | ------------------------------------------------------------------------------------------------ | ------- | ------------------------------------------------------------------------------------------------ |
| id               | string                                                                                           | **Yes** | Global ID (GID) uniquely identifying this product.                                               |
| handle           | string                                                                                           | No      | URL-safe slug for SEO-friendly URLs (e.g., 'blue-runner-pro'). Use id for stable API references. |
| title            | string                                                                                           | **Yes** | Product title.                                                                                   |
| description      | [Description](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#description)                | **Yes** | Product description in one or more formats.                                                      |
| url              | string                                                                                           | No      | Canonical product page URL.                                                                      |
| categories       | Array\[[Category](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#category)\]             | No      | Product categories with optional taxonomy identifiers.                                           |
| price_range      | [Price Range](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price-range)                | **Yes** | Price range across all variants.                                                                 |
| list_price_range | [Price Range](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price-range)                | No      | List price range before discounts (for strikethrough display).                                   |
| media            | Array\[[Media](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#media)\]                   | No      | Product media (images, videos, 3D models). First item is the featured media for listings.        |
| options          | Array\[[Product Option](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#product-option)\] | No      | Product options (Size, Color, etc.).                                                             |
| variants         | Array\[[Variant](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#variant)\]               | **Yes** | Purchasable variants of this product. First item is the featured variant for listings.           |
| rating           | [Rating](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#rating)                          | No      | Aggregate product rating.                                                                        |
| tags             | Array[string]                                                                                    | No      | Product tags for categorization and search.                                                      |
| metadata         | object                                                                                           | No      | Business-defined custom data extending the standard product model.                               |

### 변형

특정 옵션 선택, 가격, 재고 상태가 정해진 구매 가능한 항목입니다.

조회 응답의 각 변형에는 대응 관계를 파악할 수 있도록 `inputs` 배열이 있습니다. 어떤 요청 식별자가 이 변형으로 해석되었는지, 일치 방식이 `exact`였는지 `featured`(서버가 선택)였는지가 담깁니다. 자세한 내용은 [클라이언트 대응](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/lookup/#client-correlation)을 참고하세요.

`media`는 순서가 있는 배열입니다. 비즈니스는 변형의 대표 이미지를 첫 번째 요소로 반환해야 합니다(SHOULD). 플랫폼은 첫 번째 요소를 대표 항목으로 취급해야 합니다(SHOULD).

| 이름         | 타입                                                                                               | 필수    | 설명                                                                                      |
| ------------ | -------------------------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------------------------------------- |
| id           | string                                                                                             | **Yes** | Global ID (GID) uniquely identifying this variant. Used as item.id in checkout.           |
| sku          | string                                                                                             | No      | Business-assigned identifier for inventory and fulfillment.                               |
| barcodes     | Array[object]                                                                                      | No      | Industry-standard product identifiers for cross-reference and correlation.                |
| handle       | string                                                                                             | No      | URL-safe variant handle/slug.                                                             |
| title        | string                                                                                             | **Yes** | Variant display title (e.g., 'Blue / Large').                                             |
| description  | [Description](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#description)                  | **Yes** | Variant description in one or more formats.                                               |
| url          | string                                                                                             | No      | Canonical variant page URL.                                                               |
| categories   | Array\[[Category](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#category)\]               | No      | Variant categories with optional taxonomy identifiers.                                    |
| price        | [Price](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price)                              | **Yes** | Current selling price.                                                                    |
| list_price   | [Price](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price)                              | No      | List price before discounts (for strikethrough display).                                  |
| unit_price   | object                                                                                             | No      | Price per standard unit of measurement. MAY be omitted when unit pricing does not apply.  |
| availability | object                                                                                             | No      | Variant availability for purchase.                                                        |
| options      | Array\[[Selected Option](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#selected-option)\] | No      | Option values that define this variant (e.g., Color: Blue, Size: Large).                  |
| media        | Array\[[Media](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#media)\]                     | No      | Variant media (images, videos, 3D models). First item is the featured media for listings. |
| rating       | [Rating](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#rating)                            | No      | Variant rating.                                                                           |
| tags         | Array[string]                                                                                      | No      | Variant tags for categorization and search.                                               |
| metadata     | object                                                                                             | No      | Business-defined custom data extending the standard variant model.                        |
| seller       | object                                                                                             | No      | Optional seller context for this variant.                                                 |

### 가격

| 이름     | 타입                                                                    | 필수    | 설명                                                  |
| -------- | ----------------------------------------------------------------------- | ------- | ----------------------------------------------------- |
| amount   | [Amount](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#amount) | **Yes** | Amount in ISO 4217 minor units. Use 0 for free items. |
| currency | string                                                                  | **Yes** | ISO 4217 currency code (e.g., 'USD', 'EUR', 'GBP').   |

### 가격 범위

| 이름 | 타입                                                                  | 필수    | 설명                        |
| ---- | --------------------------------------------------------------------- | ------- | --------------------------- |
| min  | [Price](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price) | **Yes** | Minimum price in the range. |
| max  | [Price](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price) | **Yes** | Maximum price in the range. |

### 미디어

| 이름     | 타입    | 필수    | 설명                                                         |
| -------- | ------- | ------- | ------------------------------------------------------------ |
| type     | string  | **Yes** | Media type. Well-known values: `image`, `video`, `model_3d`. |
| url      | string  | **Yes** | URL to the media resource.                                   |
| alt_text | string  | No      | Accessibility text describing the media.                     |
| width    | integer | No      | Width in pixels (for images/video).                          |
| height   | integer | No      | Height in pixels (for images/video).                         |

### 상품 옵션

| 이름   | 타입                                                                                         | 필수    | 설명                                 |
| ------ | -------------------------------------------------------------------------------------------- | ------- | ------------------------------------ |
| name   | string                                                                                       | **Yes** | Option name (e.g., 'Size', 'Color'). |
| values | Array\[[Option Value](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#option-value)\] | **Yes** | Available values for this option.    |

### 옵션 값

| 이름  | 타입   | 필수    | 설명                                                                                                                                                  |
| ----- | ------ | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| id    | string | No      | Optional server-assigned identifier for this option value. When present in a selected_option, the server SHOULD use it for matching instead of label. |
| label | string | **Yes** | Display text for this option value (e.g., 'Small', 'Blue').                                                                                           |

### 선택된 옵션

| 이름  | 타입   | 필수    | 설명                                                                                                                                                    |
| ----- | ------ | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| name  | string | **Yes** | Option name (e.g., 'Size').                                                                                                                             |
| id    | string | No      | Optional option value identifier from option_value.id. When present, the server SHOULD use it for matching; name and label remain required for display. |
| label | string | **Yes** | Selected option label (e.g., 'Large').                                                                                                                  |

### 평점

| 이름      | 타입    | 필수    | 설명                                                       |
| --------- | ------- | ------- | ---------------------------------------------------------- |
| value     | number  | **Yes** | Average rating value.                                      |
| scale_min | number  | No      | Minimum value on the rating scale (e.g., 1 for 1-5 stars). |
| scale_max | number  | **Yes** | Maximum value on the rating scale (e.g., 5 for 5-star).    |
| count     | integer | No      | Number of reviews contributing to the rating.              |

## 메시지와 에러 처리

모든 카탈로그 응답에는 선택 사항인 `messages` 배열이 있어, 비즈니스가 에러, 경고, 정보성 알림의 맥락을 전달할 수 있습니다.

### 메시지 타입

메시지는 비즈니스의 처리 결과를 전달하고 맥락을 제공합니다.

| 타입      | 사용 시점                      | 예시 코드                                        |
| --------- | ------------------------------ | ------------------------------------------------ |
| `error`   | 비즈니스 수준 에러             | `NOT_FOUND`, `OUT_OF_STOCK`, `REGION_RESTRICTED` |
| `warning` | 구매에 영향을 주는 중요한 조건 | `DELAYED_FULFILLMENT`, `FINAL_SALE`              |
| `info`    | 문제없이 덧붙이는 추가 맥락    | `PROMOTIONAL_PRICING`, `LIMITED_AVAILABILITY`    |

`presentation: "disclosure"`가 있는 경고에는 플랫폼이 숨기거나 닫아서는 안 되는 고지(예: 알레르기 유발 성분 표시, 안전 경고)가 담깁니다. 전체 렌더링 계약은 [경고 표시](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#warning-presentation)를 참고하세요.

**참고**: 모든 카탈로그 에러는 `severity: "recoverable"`을 사용하며, 에이전트가 이를 프로그램으로 처리합니다(재시도, 사용자에게 알림, 대안 제시).

#### 메시지 (에러)

| 이름         | 타입                                                                            | 필수    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------ | ------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                          | **Yes** | **Constant = error**. Message type discriminator.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| code         | [Error Code](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#error-code) | **Yes** | Error code identifying the type of error. Standard errors are defined in specification (see examples), and have standardized semantics; freeform codes are permitted.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| path         | string                                                                          | No      | RFC 9535 JSONPath to the component the message refers to (e.g., $.items[1]).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| content_type | string                                                                          | No      | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| content      | string                                                                          | **Yes** | Human-readable message.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| severity     | string                                                                          | **Yes** | Reflects the resource state and recommended action. 'recoverable': platform can resolve by modifying inputs and retrying via API. 'requires_buyer_input': merchant requires information their API doesn't support collecting programmatically (checkout incomplete). 'requires_buyer_review': buyer must authorize before order placement due to policy, regulatory, or entitlement rules. 'unrecoverable': no valid resource exists to act on, retry with new resource or inputs. Errors with 'requires\_*' severity contribute to 'status: requires_escalation'.* *Enum:*\* `recoverable`, `requires_buyer_input`, `requires_buyer_review`, `unrecoverable` |

#### 메시지 (경고)

| 이름         | 타입                                                                                | 필수    | 설명                                                                                                                                                                                                                                                |
| ------------ | ----------------------------------------------------------------------------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                              | **Yes** | **Constant = warning**. Message type discriminator.                                                                                                                                                                                                 |
| path         | string                                                                              | No      | JSONPath (RFC 9535) to related field (e.g., $.line_items[0]).                                                                                                                                                                                       |
| code         | [Warning Code](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#warning-code) | **Yes** | Warning code identifying the type of warning. Standard codes are defined in capability specs (see examples), and have standardized semantics; freeform codes are permitted.                                                                         |
| content      | string                                                                              | **Yes** | Human-readable warning message that MUST be displayed.                                                                                                                                                                                              |
| content_type | string                                                                              | No      | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                                                                                      |
| presentation | string                                                                              | No      | Rendering contract for this warning. 'notice' (default): platform MUST display, MAY dismiss. 'disclosure': platform MUST display in proximity to the path-referenced component, MUST NOT hide or auto-dismiss. See specification for full contract. |
| image_url    | string                                                                              | No      | URL to a required visual element (e.g., warning symbol, energy class label).                                                                                                                                                                        |
| url          | string                                                                              | No      | Reference URL for more information (e.g., regulatory site, registry entry, policy page).                                                                                                                                                            |

#### 메시지 (정보)

| 이름         | 타입                                                                          | 필수    | 설명                                                                                                                                                                                   |
| ------------ | ----------------------------------------------------------------------------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                        | **Yes** | **Constant = info**. Message type discriminator.                                                                                                                                       |
| path         | string                                                                        | No      | RFC 9535 JSONPath to the component the message refers to.                                                                                                                              |
| code         | [Info Code](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#info-code) | No      | Info code identifying the type of informational message. Standard codes are defined in capability specs (see examples), and have standardized semantics; freeform codes are permitted. |
| content_type | string                                                                        | No      | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                         |
| content      | string                                                                        | **Yes** | Human-readable message.                                                                                                                                                                |

### 일반적인 시나리오

#### 빈 검색 결과

검색 결과가 없으면 메시지 없이 빈 배열을 반환합니다.

```json
{
  "ucp": {...},
  "products": []
}
```

이는 에러가 아닙니다. 쿼리는 유효했지만 결과가 없었을 뿐입니다.

#### 이월 주문 경고

상품은 구매할 수 있지만 풀필먼트가 지연되는 경우, 경고 메시지와 함께 상품을 반환합니다. 특정 변형을 가리키려면 `path` 필드를 사용합니다.

```json
{
  "ucp": {...},
  "products": [
    {
      "id": "prod_xyz789",
      "title": "Professional Chef Knife Set",
      "description": { "plain": "Complete professional knife collection." },
      "price_range": {
        "min": { "amount": 29900, "currency": "USD" },
        "max": { "amount": 29900, "currency": "USD" }
      },
      "variants": [
        {
          "id": "var_abc",
          "title": "12-piece Set",
          "description": { "plain": "Complete professional knife collection." },
          "price": { "amount": 29900, "currency": "USD" },
          "availability": { "available": true }
        }
      ]
    }
  ],
  "messages": [
    {
      "type": "warning",
      "code": "delayed_fulfillment",
      "path": "$.products[0].variants[0]",
      "content": "12-piece set on backorder, ships in 2-3 weeks"
    }
  ]
}
```

에이전트는 해당 옵션을 보여 주면서 사용자에게 지연 사실을 알릴 수 있습니다. `path` 필드는 RFC 9535 JSONPath로 특정 구성 요소를 가리킵니다.

#### 식별자를 찾을 수 없음

요청한 식별자가 존재하지 않아도, 찾은 상품(있다면)과 함께 성공으로 반환합니다. 응답에 찾지 못한 식별자를 알리는 정보성 메시지를 포함할 수 있습니다(MAY).

```json
{
  "ucp": {...},
  "products": [],
  "messages": [
    {
      "type": "info",
      "code": "not_found",
      "content": "prod_invalid"
    }
  ]
}
```

에이전트는 각 변형의 `inputs` 배열로 결과의 대응 관계를 파악합니다. [클라이언트 대응](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/lookup/#client-correlation)을 참고하세요.

#### 상품 고지

상품에 고지(예: 알레르기 유발 성분 안내, 안전 경고)가 필요하면 `presentation: "disclosure"`가 있는 경고로 반환합니다. `path` 필드는 응답에서 관련 구성 요소를 가리키며, 상품을 가리키면 고지가 그 상품의 모든 변형에 적용됩니다.

```json
{
  "ucp": {...},
  "products": [
    {
      "id": "prod_nut_butter",
      "title": "Artisan Nut Butter Collection",
      "variants": [
        {
          "id": "var_almond",
          "title": "Almond Butter",
          "price": { "amount": 1299, "currency": "USD" },
          "availability": { "available": true }
        },
        {
          "id": "var_cashew",
          "title": "Cashew Butter",
          "price": { "amount": 1499, "currency": "USD" },
          "availability": { "available": true }
        }
      ]
    }
  ],
  "messages": [
    {
      "type": "warning",
      "code": "allergens",
      "path": "$.products[0]",
      "content": "**Contains: tree nuts.** Produced in a facility that also processes peanuts, milk, and soy.",
      "content_type": "markdown",
      "presentation": "disclosure",
      "image_url": "https://merchant.com/allergen-tree-nuts.svg",
      "url": "https://merchant.com/allergen-info"
    }
  ]
}
```

전체 렌더링 계약은 [경고 표시](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#warning-presentation)를 참고하세요.

## 스코프

카탈로그 검색과 카탈로그 조회 기능은 사용자 인증 접근을 위해 다음과 같은 잘 알려진 스코프를 정의합니다.

| 스코프                                 | 설명                                                                                             |
| -------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `dev.ucp.shopping.catalog.search:read` | 인증된 사용자를 대신해 검색합니다. 개인화된 결과, 회원 가격, 접근이 제한된 재고 등이 해당합니다. |
| `dev.ucp.shopping.catalog.lookup:read` | 인증된 사용자를 대신해 조회합니다. 특정 상품의 개인화된 가격이나 재고 상태 등이 해당합니다.      |

스코프 선언과 도출, 사용자 정의 스코프로 이 집합을 확장하는 규칙은 [신원 연결 — 스코프](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/identity-linking/#scopes)에 정의되어 있습니다.

## 전송 바인딩

위 기능은 다음 전송 프로토콜에 바인딩됩니다.

- [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/rest/index.md): RESTful API 매핑
- [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/mcp/index.md): JSON-RPC 기반 Model Context Protocol 매핑
