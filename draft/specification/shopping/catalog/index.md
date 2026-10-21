# 카탈로그 기능

## 개요

카탈로그 기능(Capability)을 사용하면 플랫폼이 비즈니스의 상품 카탈로그를 검색하고 탐색할 수 있습니다. 체크아웃 전에 상품을 찾는 단계로, 다음과 같은 사용 사례를 지원합니다.

- 자유 텍스트 상품 검색
- 카테고리 및 필터 기반 탐색
- 식별자로 상품/변형 일괄 조회
- 변형 간 가격 비교

## 기능

| 기능                                                                                                                                      | 설명                                    |
| ----------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| [`dev.ucp.shopping.catalog.search`](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/search/index.md) | 쿼리 텍스트와 필터로 상품을 검색합니다. |
| [`dev.ucp.shopping.catalog.lookup`](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/index.md) | 식별자로 상품 또는 변형을 조회합니다.   |

## 핵심 개념

- **상품(Product)**: 제목, 설명, 미디어와 하나 이상의 변형으로 구성된 카탈로그 항목입니다.
- **변형(Variant)**: 특정 옵션 선택(예: "Blue / Large"), 가격, 재고 상태가 정해진 구매 가능한 항목입니다.
- **가격(Price)**: 가격 값에는 금액(통화의 최소 단위)과 통화 코드가 모두 들어 있어 다중 통화 카탈로그를 지원합니다.
- **판매 기준(Sale basis)**: 수량을 표시하는 방식입니다. 개수 단위(`each`, 기본값)이거나 무게, 길이, 면적, 부피, 시간 같은 측정 단위입니다. [수량과 단위](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#quantities-and-units)를 참고하세요.

### 체크아웃과의 관계

카탈로그 작업은 체크아웃의 `line_items[].item.id`에 바로 쓸 수 있는 상품 ID와 변형 ID를 반환합니다. 카탈로그 조회로 얻은 변형 ID는 체크아웃이 기대하는 아이템 ID와 일치해야 합니다.

카탈로그 응답(가격, 재고 상태 등)은 해당 요청 시점의 비즈니스 조건을 반영하지만 거래상의 확약은 아닙니다. 권위는 체크아웃에 있습니다. 응답은 세션마다 다를 수 있으므로, 재검증 없이 다른 세션에서 재사용하지 않는 것이 좋습니다(**SHOULD NOT**).

## 판매 기준과 수량 단위

`variants[].quantity_unit`은 거래 전에 변형의 판매 기준을 알려 줍니다. 이 서술자는 공통 [수량과 단위](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#quantities-and-units) 계약을 따릅니다. 서술자가 없으면 기본값인 `each` 기준이라는 뜻이며, `each`가 아닌 기준은 비즈니스가 서술자를 포함해 알립니다.

플랫폼은 거래 전에 카탈로그에서 판매 기준을 파악합니다. `unit`과 `scale`은 수량을 표시하는 방식을 정의하고, 선택 사항인 [`increment`](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#ordering-increment)는 비즈니스가 판매하는 주문 단위를 알려 줍니다. 플랫폼은 이를 바탕으로 수량 조절 UI를 만들고 제출 전에 입력을 검증할 수 있습니다. 바나나를 파운드 단위로, 1/4파운드 배수로 판매하는 비즈니스라면 `{ "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }`를 알립니다.

판매 기준이 물리적 측정에만 국한되지는 않습니다. 분 단위 주차, 시간 단위 인건비 같은 계량형 상품도 같은 계약에 따라 같은 서술자(`MIN`, `HUR`)를 사용합니다.

`variants[].id`는 구매 가능한 변형을 식별하고, `quantity_unit`은 그 변형을 주문할 때 쓰는 단위와 세분성을 정의합니다. `quantity_unit`이 `{ "unit": "LBR", "scale": 2, "display_text": "lb" }`인 바나나는 1/100파운드 단위로 판매되므로, `quantity`가 `150`이면 1.50 lb입니다.

### 단가와의 구분

`quantity_unit`과 변형의 [`unit_price`](#variant)는 서로 다른 질문에 답하며, 각각 독립적으로 설정합니다.

- `quantity_unit`은 **판매 기준**입니다. `quantity`를 세는 단위이자 `price`를 매기는 단위입니다.
- `unit_price`는 **표시용 비교 지표**입니다. 매대에서처럼 비교할 수 있도록 파생한 "표준 측정량당 가격"(예: 100 mL당)입니다. `measure`는 변형의 포장 또는 내용량이고, `reference`는 비교 기준이 되는 분모입니다.

`quantity_unit`, `unit_price.measure`, `unit_price.reference`의 단위 필드는 공통 [수량과 단위](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#quantities-and-units) 계약을 따릅니다. 두 단가 필드는 공통 측정 타입을 사용하므로 정수 `value` 필드는 단계 수입니다. 비즈니스는 변형에 `quantity_unit`이나 `unit_price` 중 하나만 제공하거나, 둘 다 제공하거나, 둘 다 제공하지 않을 수 있습니다(**MAY**).

표시용 비교 지표가 의미를 잃지 않도록, 비즈니스는 반드시 `unit_price.measure.value`와 `unit_price.reference.value` 모두에 양의 정수를 사용해야 합니다(**MUST**). 비즈니스는 반드시 `unit_price.currency`를 `price.currency`와 같게 설정해야 합니다(**MUST**). `unit_price` 안에서 비즈니스는 반드시 `measure.unit`과 `reference.unit`에 같은 값을 사용해야 합니다(**MUST**). 두 측정값의 `scale` 값은 서로 달라도 됩니다(**MAY**). 비즈니스는 단가를 계산하면서 단위 간 변환이나 통화 변환을 해서는 안 됩니다(**MUST NOT**). 각 측정값은 공통 단위 기준으로 정수 `value × 10^-scale`를 나타냅니다. 비즈니스는 반드시 `price.amount`와 이 스케일 적용 값으로 비교 지표를 계산해야 합니다(**MUST**).

```text
(price.amount / (measure.value × 10^-measure.scale)) × (reference.value × 10^-reference.scale)
```

비즈니스는 반드시 가격 규칙에 따라 결과를 통화의 최소 단위로 한 번 반올림해 `unit_price.amount`로 반환해야 합니다(**MUST**). 반환된 `unit_price.amount`가 권위를 가집니다. 플랫폼은 이 값을 다시 계산하거나 자체 결과로 대체해서는 안 됩니다(**MUST NOT**).

동일 단위·동일 통화 규칙은 의미상의 불변 조건입니다. JSON Schema는 해당 필드를 각각 독립적으로 검증할 뿐, 두 동일성 중 어느 것도 강제하지 않습니다.

예를 들어 `each`로 판매하는 50 m 케이블 스풀은 `quantity_unit`을 생략하고 미터당 `unit_price`를 둘 수 있습니다. 이때 `measure`는 `{ "value": 5000, "unit": "MTR", "scale": 2, "display_text": "m" }`, `reference`는 `{ "value": 1, "unit": "MTR", "display_text": "m" }`가 될 수 있습니다. 두 측정값 모두 `MTR`을 사용하며, 단위 간 변환 없이 각각 50 m와 1 m를 나타냅니다.

거래 라인에서는 필드가 있느냐 없느냐로 역할이 드러납니다. 가격 기준이 판매 기준과 다른 라인에는 비즈니스가 반드시 `unit_price`를 빠짐없이 담아 보내야 합니다(**MUST**)([체크아웃 — 가격 기준](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#quantity-and-sale-basis) 참고). 이 스풀의 미터당 가격처럼 카탈로그에만 있는 `unit_price`는 표시용 비교 지표이고, 라인 수준의 `unit_price`에는 청구 금액을 계산하는 기준 요율이 담깁니다.

### 예시: 무게 단위로 판매하는 상품

파운드 단위로 판매하는 바나나의 `get_product` 응답입니다. 변형은 `quantity_unit`으로 `{ "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }`를, `price`로 `79`를 알립니다. 즉 1파운드당 $0.79이며 0.25 lb 배수로 판매합니다.

```json
{
  "ucp": { "version": "draft" },
  "product": {
    "id": "prod_bananas",
    "title": "Bananas",
    "description": { "plain": "Fresh bananas sold by the pound." },
    "price_range": {
      "min": { "amount": 79, "currency": "USD" },
      "max": { "amount": 79, "currency": "USD" }
    },
    "variants": [
      {
        "id": "var_bananas",
        "title": "Bananas",
        "description": { "plain": "Fresh bananas sold by the pound." },
        "price": { "amount": 79, "currency": "USD" },
        "quantity_unit": { "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 },
        "availability": { "available": true }
      }
    ]
  }
}
```

## 공통 엔터티

### 컨텍스트

카탈로그 작업에 쓰는 위치 및 시장 컨텍스트입니다. 모든 필드는 관련성과 현지화를 위한 선택적 힌트입니다. 플랫폼은 요청 헤더로 컨텍스트를 지리적으로 감지할 수 있습니다(MAY).

컨텍스트 신호는 잠정적인 값일 뿐 권위 있는 데이터가 아닙니다. 비즈니스는 검증된 입력(예: 배송 주소)이 없을 때 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제상 제약(수출 통제)과 맞지 않으면 무시하거나 순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 이루어져야 합니다(MUST).

비즈니스는 컨텍스트 신호를 바탕으로 통화를 포함한 시장 배정을 결정합니다. 가격 필터 값은 `context.currency` 기준이며, 표시 통화가 다르면 비즈니스는 적용 전에 변환해야 합니다(SHOULD)([가격 필터](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/search/#price-filter) 참고). 응답 가격에는 결정 결과를 확인할 수 있도록 통화 코드가 명시됩니다.

`context.eligibility` 주장이 있고 비즈니스가 이를 수락했다면, 취소선 표시를 위해 `price` / `list_price`를 직접 조정할 수 있으며(**MAY**), `code: "eligibility_benefit"`이 담긴 `messages`로 조정 내용이 어떤 주장에 따른 것인지 밝힐 수 있습니다(**MAY**).

| 이름            | 타입                                                                                             | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| --------------- | ------------------------------------------------------------------------------------------------ | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| address_country | string                                                                                           | 선택      | 2자리 ISO 3166-1 alpha-2 코드로 표기한 국가(예: "US"). 3자리 alpha-3 코드나 전체 국가명을 사용할 수도 있습니다(MAY).                                                                                                                                                                                                                                                                                                                                           |
| address_region  | string                                                                                           | 선택      | 국가 안의 1차 행정 구역(예: California 같은 주 또는 도).                                                                                                                                                                                                                                                                                                                                                                                                       |
| postal_code     | string                                                                                           | 선택      | 우편번호(예: "94043").                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| location        | string                                                                                           | 선택      | 비즈니스 네임스페이스 안에서 위치(Location)를 가리키는 안정적이고 불투명한 식별자. 이 값은 잠정적이고 구속력 없는 힌트이며 구매자의 지역 정보와는 별개입니다. 효과는 작업 명세나 활성화된 기능/확장이 정의합니다. 소매 쇼핑의 흔한 예로, 사용자가 식료품을 구매할 때 선택해 저장한 기본 단골 매장 ID가 있습니다.                                                                                                                                               |
| intent          | string                                                                                           | 선택      | 구매자의 의도를 설명하는 배경 맥락(예: 'looking for a gift under $50', 'need something durable for outdoor use'). 관련성, 추천, 개인화에 활용됩니다.                                                                                                                                                                                                                                                                                                           |
| language        | string                                                                                           | 선택      | 콘텐츠의 선호 언어입니다. IETF BCP 47 언어 태그(예: 'en', 'fr-CA', 'zh-Hans')를 사용합니다. REST에서는 Accept-Language 헤더에 해당합니다. 이 필드가 없으면 플랫폼은 Accept-Language로 대체해야 하며(SHOULD), 값이 있으면 Accept-Language보다 우선합니다. 해당 언어를 제공할 수 없으면 비즈니스는 다른 언어로 콘텐츠를 반환할 수 있습니다(MAY).                                                                                                                 |
| currency        | string                                                                                           | 선택      | 선호 통화(ISO 4217, 예: 'EUR', 'USD'). 비즈니스는 맥락과 신뢰할 수 있는 신호로 표시 통화를 결정하며, 다중 통화 시장에서는 이 힌트를 선택에 참고할 수 있습니다(MAY). 가격 필터 값의 통화 단위로도 쓰이므로, 플랫폼은 가격 필터를 보낼 때 이 필드를 포함해야 합니다(SHOULD). 응답 가격에는 결정된 통화를 확인할 수 있도록 통화가 명시됩니다.                                                                                                                     |
| eligibility     | Array\[[Reverse Domain Name](/ucp-ko/draft/draft/specification/reference/#reverse-domain-name)\] | 선택      | 로열티 멤버십, 결제 수단 혜택 등 적격 혜택에 대한 구매자 주장. 인식된 주장은 비즈니스 응답에 반영될 수 있습니다(MAY)(예: 회원 전용 상품 구매 가능 여부, 카탈로그의 조정 가격, 장바구니나 체크아웃의 잠정 할인). 비즈니스는 인식하지 못한 값을 오류 없이 반드시 무시해야 합니다(MUST). 값은 반드시 역 도메인 이름 규칙을 사용해야 하며(MUST)(예: 'com.example.loyalty_gold', 'org.school.student'), 반드시 개인을 식별할 수 없어야 합니다(MUST).                |
| payment         | Array[object]                                                                                    | 선택      | 구매자가 선호하는 결제 핸들러를 우선순위 순서(가장 선호하는 것이 먼저)로 나열합니다. 각 항목은 비즈니스 프로필의 `ucp.payment_handlers`에 공개된 핸들러를 가리키며, 선호하는 결제 수단 유형으로 범위를 좁힐 수도 있습니다. 비즈니스는 이를 사용해 핸들러(유형이 주어지면 유형까지)를 미리 선택하거나 우선순위를 정해야 하며(SHOULD), 사용할 수 없거나 자격이 없는 항목은 무시할 수 있습니다(MAY). 인식하지 못하는 값은 오류 없이 반드시 무시해야 합니다(MUST). |

### 신호

인가와 악용 방지에 도움이 되도록 플랫폼이 제공하는 환경 데이터입니다. 신호 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 자세한 내용과 개인정보 보호 요건은 [신호](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#signals)를 참고하세요.

| 이름               | 타입   | 필수 여부 | 설명                                                     |
| ------------------ | ------ | --------- | -------------------------------------------------------- |
| dev.ucp.buyer_ip   | string | 선택      | 클라이언트의 IP 주소(IPv4 또는 IPv6).                    |
| dev.ucp.user_agent | string | 선택      | 클라이언트의 HTTP User-Agent 헤더 또는 그에 상응하는 값. |

### 어트리뷰션

플랫폼이 제공하는 유입 및 전환 이벤트 컨텍스트로, 플랫폼이 전달하는 캠페인 ID, 클릭 식별자, 소스/매체 표식 등이 해당합니다. 자세한 내용과 동의 요건은 [어트리뷰션](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#attribution)을 참고하세요.

플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.

### 상품

판매 가능한 항목을 나타내는 카탈로그 항목으로, 구매 가능한 변형이 하나 이상 있습니다.

`media`와 `variants`는 순서가 있는 배열입니다. 비즈니스는 가장 관련성 높은 변형과 미디어 항목을 맨 앞에 반환해야 합니다(**SHOULD**). 조회에서는 기본값을, 검색에서는 쿼리와 컨텍스트에 가장 잘 맞는 항목을 앞에 둡니다. 플랫폼은 자신이 표시하는 첫 번째 미디어 항목을 대표 항목으로 취급해야 합니다(**SHOULD**).

| 이름             | 타입                                                                                   | 필수 여부 | 설명                                                                                                   |
| ---------------- | -------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------ |
| id               | string                                                                                 | **필수**  | 이 상품을 고유하게 식별하는 전역 ID(GID).                                                              |
| handle           | string                                                                                 | 선택      | SEO 친화적인 URL을 위한 URL 안전 슬러그(예: 'blue-runner-pro'). 안정적인 API 참조에는 id를 사용하세요. |
| title            | string                                                                                 | **필수**  | 상품 제목.                                                                                             |
| description      | [Description](/ucp-ko/draft/draft/specification/reference/#description)                | **필수**  | 하나 이상의 형식으로 작성된 상품 설명.                                                                 |
| url              | string                                                                                 | 선택      | 표준 상품 페이지 URL.                                                                                  |
| categories       | Array\[[Category](/ucp-ko/draft/draft/specification/reference/#category)\]             | 선택      | 선택적 분류 체계 식별자가 포함된 상품 카테고리.                                                        |
| price_range      | [Price Range](/ucp-ko/draft/draft/specification/reference/#price-range)                | **필수**  | 모든 변형에 걸친 가격 범위.                                                                            |
| list_price_range | [Price Range](/ucp-ko/draft/draft/specification/reference/#price-range)                | 선택      | 할인 전 정가 범위(취소선 표시용).                                                                      |
| media            | Array\[[Media](/ucp-ko/draft/draft/specification/reference/#media)\]                   | 선택      | 상품 미디어(이미지, 동영상, 3D 모델). 첫 번째 항목은 목록에 표시되는 대표 미디어입니다.                |
| options          | Array\[[Product Option](/ucp-ko/draft/draft/specification/reference/#product-option)\] | 선택      | 상품 옵션(Size, Color 등).                                                                             |
| variants         | Array\[[Variant](/ucp-ko/draft/draft/specification/reference/#variant)\]               | **필수**  | 이 상품의 구매 가능한 변형입니다. 첫 번째 항목은 목록에 표시되는 대표 변형입니다.                      |
| rating           | [Rating](/ucp-ko/draft/draft/specification/reference/#rating)                          | 선택      | 상품 종합 평점.                                                                                        |
| tags             | Array[string]                                                                          | 선택      | 분류와 검색에 쓰는 상품 태그.                                                                          |
| metadata         | object                                                                                 | 선택      | 표준 상품 모델을 확장하는 비즈니스 정의 사용자 데이터.                                                 |

### 변형

특정 옵션 선택, 가격, 재고 상태가 정해진 구매 가능한 항목입니다.

조회 응답의 각 변형에는 대응 관계를 파악할 수 있도록 `inputs` 배열이 있습니다. 어떤 요청 식별자가 이 변형으로 해석되었는지, 일치 방식이 `exact`였는지 `featured`(서버가 선택)였는지가 담깁니다. 자세한 내용은 [클라이언트 대응](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/#client-correlation)을 참고하세요.

`media`는 순서가 있는 배열입니다. 비즈니스는 변형의 대표 미디어 항목을 첫 번째 요소로 반환해야 합니다(**SHOULD**). 플랫폼은 자신이 표시하는 첫 번째 미디어 항목을 대표 항목으로 취급해야 합니다(**SHOULD**).

| 이름          | 타입                                                                                     | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                             |
| ------------- | ---------------------------------------------------------------------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id            | string                                                                                   | **필수**  | 이 변형을 고유하게 식별하는 전역 ID(GID). 체크아웃에서 item.id로 사용합니다.                                                                                                                                                                                                                                                     |
| sku           | string                                                                                   | 선택      | 재고와 풀필먼트를 위해 비즈니스가 할당한 식별자.                                                                                                                                                                                                                                                                                 |
| barcodes      | Array[object]                                                                            | 선택      | 상호 참조와 연계를 위한 업계 표준 상품 식별자.                                                                                                                                                                                                                                                                                   |
| handle        | string                                                                                   | 선택      | URL 안전 변형 핸들/슬러그.                                                                                                                                                                                                                                                                                                       |
| title         | string                                                                                   | **필수**  | 변형 표시 제목(예: 'Blue / Large').                                                                                                                                                                                                                                                                                              |
| description   | [Description](/ucp-ko/draft/draft/specification/reference/#description)                  | **필수**  | 하나 이상의 형식으로 된 변형 설명.                                                                                                                                                                                                                                                                                               |
| url           | string                                                                                   | 선택      | 표준 변형(variant) 페이지 URL.                                                                                                                                                                                                                                                                                                   |
| categories    | Array\[[Category](/ucp-ko/draft/draft/specification/reference/#category)\]               | 선택      | 선택적 분류 체계 식별자가 포함된 변형 카테고리.                                                                                                                                                                                                                                                                                  |
| price         | [Price](/ucp-ko/draft/draft/specification/reference/#price)                              | **필수**  | 현재 판매 가격. 가격은 `quantity_unit.unit` 한 단위(예: lb당, 시간당) 기준 금액이며, `quantity_unit`이 없으면 `each` 기준입니다. 라인 합계는 `price × quantity × 10^-scale`이며 비즈니스가 한 번 계산하고 반올림합니다. 권위 있는 값은 여전히 `totals`입니다.                                                                    |
| quantity_unit | [Quantity Unit](/ucp-ko/draft/draft/specification/reference/#quantity-unit)              | 선택      | 이 변형의 `quantity`가 표시되는 판매 기준입니다. 기본 판매 기준은 `each`이며, 기계적 정체성은 (`C62`, 0)입니다. `C62`는 one/each를 뜻하는 UN/CEFACT Rec20 코드입니다. 카탈로그 기술자가 없으면 이 기본값을 뜻합니다. `increment`는 스텝 단위의 주문 단위를 알립니다(예: `scale` 2에 `increment` 25이면 0.25 단위의 배수로 판매). |
| list_price    | [Price](/ucp-ko/draft/draft/specification/reference/#price)                              | 선택      | 할인 전 정가(취소선 표시용).                                                                                                                                                                                                                                                                                                     |
| unit_price    | [Unit Price](/ucp-ko/draft/draft/specification/reference/#unit-price)                    | 선택      | 선반 표시 방식의 가격 비교에 쓰는 표준 측정 단위당 가격입니다. 단위 가격이 적용되지 않으면 생략할 수 있습니다(MAY).                                                                                                                                                                                                              |
| availability  | [Availability](/ucp-ko/draft/draft/specification/reference/#availability)                | 선택      | 변형의 구매 가능 여부.                                                                                                                                                                                                                                                                                                           |
| options       | Array\[[Selected Option](/ucp-ko/draft/draft/specification/reference/#selected-option)\] | 선택      | 이 변형을 정의하는 옵션 값(예: Color: Blue, Size: Large).                                                                                                                                                                                                                                                                        |
| media         | Array\[[Media](/ucp-ko/draft/draft/specification/reference/#media)\]                     | 선택      | 변형 미디어(이미지, 동영상, 3D 모델). 첫 번째 항목이 목록에 노출되는 대표 미디어입니다.                                                                                                                                                                                                                                          |
| rating        | [Rating](/ucp-ko/draft/draft/specification/reference/#rating)                            | 선택      | 변형 평점.                                                                                                                                                                                                                                                                                                                       |
| tags          | Array[string]                                                                            | 선택      | 분류와 검색을 위한 변형 태그.                                                                                                                                                                                                                                                                                                    |
| metadata      | object                                                                                   | 선택      | 표준 변형 모델을 확장하는 비즈니스 정의 사용자 데이터.                                                                                                                                                                                                                                                                           |
| seller        | object                                                                                   | 선택      | 이 변형의 판매자 맥락(선택 사항).                                                                                                                                                                                                                                                                                                |

### 가격

| 이름     | 타입                                                          | 필수 여부 | 설명                                                        |
| -------- | ------------------------------------------------------------- | --------- | ----------------------------------------------------------- |
| amount   | [Amount](/ucp-ko/draft/draft/specification/reference/#amount) | **필수**  | ISO 4217 최소 통화 단위의 금액. 무료 항목은 0을 사용합니다. |
| currency | string                                                        | **필수**  | ISO 4217 통화 코드(예: 'USD', 'EUR', 'GBP').                |

### 가격 범위

| 이름 | 타입                                                        | 필수 여부 | 설명              |
| ---- | ----------------------------------------------------------- | --------- | ----------------- |
| min  | [Price](/ucp-ko/draft/draft/specification/reference/#price) | **필수**  | 범위의 최저 가격. |
| max  | [Price](/ucp-ko/draft/draft/specification/reference/#price) | **필수**  | 범위의 최고 가격. |

### 미디어

상품이나 변형에 속한 하나의 논리적 미디어 항목으로, 이미지, 동영상, 3D 모델 등이 있습니다. 미디어 항목은 플랫폼이 렌더링하는 리소스입니다. `url`은 항목을 식별하고, `sources`는 대체 렌디션을 나열합니다. `preview`는 플랫폼이 항목 대신 또는 항목보다 먼저 보여 줄 수 있는 정지 이미지이고, `width`와 `height`는 고유 픽셀 크기입니다. `preview`와 각 `sources` 항목의 `width`와 `height`는 해당 정지 이미지나 렌디션의 픽셀 크기입니다.

`type`은 개방형 문자열이며, 잘 알려진 값으로 `image`, `video`, `external_video`, `model_3d` 네 가지가 있습니다. 플랫폼은 미디어 항목의 `type`을 인식하지 못한다는 이유로 상품이나 응답을 거부해서는 안 됩니다(**MUST NOT**). 그런 항목은 `preview`와 `name`으로 표시하거나 생략할 수 있습니다(**MAY**). `type`을 인식하면 타입에 맞게 표시할 수 있다는 점이 다를 뿐입니다.

플랫폼은 `https` 스킴을 사용하지 않는 미디어 URL의 로드나 임베드를 거부할 수 있습니다(**MAY**).

플랫폼은 `mime_type`과 크기를 기준으로 `sources`에서 렌디션을 고릅니다. `sources`를 제공하는 비즈니스는 `url`에 해당하는 렌디션도 포함해야 합니다(**SHOULD**).

| 이름     | 타입                                                                               | 필수 여부 | 설명                                                                                                                                                                                                                                                               |
| -------- | ---------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| type     | string                                                                             | **필수**  | 미디어 유형. 잘 알려진 값: `image`, `video`, `external_video`, `model_3d`. 플랫폼은 인식하지 못한 값이 있다고 해서 이를 포함한 객체를 거부해서는 안 됩니다(MUST NOT). 대신 `preview`와 `name`으로 항목을 표시하거나 생략할 수 있습니다(MAY).                       |
| url      | string                                                                             | **필수**  | 플랫폼이 렌더링하는 미디어 리소스의 URL. 해석 방식은 `type`에 따라 달라집니다.                                                                                                                                                                                     |
| name     | string                                                                             | 선택      | 미디어의 사람이 읽을 수 있는 제목 또는 라벨로, `alt_text`(접근성 텍스트)와는 별개입니다. `video`와 `model_3d`에 자주 있습니다.                                                                                                                                     |
| alt_text | string                                                                             | 선택      | 미디어를 설명하는 접근성 텍스트.                                                                                                                                                                                                                                   |
| width    | integer                                                                            | 선택      | 미디어 항목의 고유 픽셀 너비로, 이미지·동영상 유형용입니다. 각 렌디션의 실제 크기는 해당 `sources` 항목에 있습니다.                                                                                                                                                |
| height   | integer                                                                            | 선택      | 미디어 항목의 고유 픽셀 높이로, 이미지·동영상 유형용입니다. 각 렌디션의 실제 크기는 해당 `sources` 항목에 있습니다.                                                                                                                                                |
| duration | integer                                                                            | 선택      | `video`, `external_video` 같은 시간 기반 미디어의 재생 시간(초).                                                                                                                                                                                                   |
| preview  | object                                                                             | 선택      | 플랫폼이 기본 리소스보다 먼저 또는 그 대신 렌더링하는 포스터/썸네일 정지 이미지. 비즈니스는 적절한 정지 이미지가 없는 경우가 아니면 `video`, `external_video`, `model_3d`에 이를 제공해야 하며(SHOULD), `image`에도 저해상도 자리표시자로 제공할 수 있습니다(MAY). |
| sources  | Array\[[Media Source](/ucp-ko/draft/draft/specification/reference/#media-source)\] | 선택      | `url`에 있는 리소스의 대체 렌디션. 비즈니스는 `url`의 렌디션도 포함해야 합니다(SHOULD).                                                                                                                                                                            |

#### 이미지

`url`은 바로 표시할 수 있는 이미지입니다. `sources`가 있으면 해당 이미지의 대체 인코딩과 크기를 나열합니다. 예를 들면 `image/avif`, `image/webp`, `image/jpeg`이나 반응형 레이아웃을 위한 너비별 변형입니다. `sources`에서 고르지 않는 플랫폼은 `url`을 바로 로드하므로, 비즈니스는 `url`을 가장 폭넓게 디코딩할 수 있는 렌디션으로 지정해야 합니다(**SHOULD**).

| 이름    | 타입                                                                               | 필수 여부 | 설명                                                                                                                                                                         |
| ------- | ---------------------------------------------------------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| url     | string                                                                             | **필수**  | 바로 표시할 수 있는 이미지.                                                                                                                                                  |
| sources | Array\[[Media Source](/ucp-ko/draft/draft/specification/reference/#media-source)\] | 선택      | 이미지의 대체 인코딩과 크기(예: `image/avif`, `image/webp`, `image/jpeg`, 반응형 레이아웃을 위한 너비 변형). `url`은 가장 널리 디코딩할 수 있는 렌디션이어야 합니다(SHOULD). |

#### 동영상

`url`은 동영상의 재생 가능한 렌디션으로, 프로그레시브 파일이거나 적응형 스트리밍 매니페스트입니다. `sources`가 있으면 대체 렌디션과 매니페스트를 나열합니다.

| 이름    | 타입                                                                               | 필수 여부 | 설명                                                                             |
| ------- | ---------------------------------------------------------------------------------- | --------- | -------------------------------------------------------------------------------- |
| url     | string                                                                             | **필수**  | 재생 가능한 동영상 렌디션: 프로그레시브 파일 또는 적응형 스트리밍 매니페스트.    |
| width   | integer                                                                            | 선택      | 동영상의 고유 픽셀 너비. 각 렌디션의 실제 크기는 해당 `sources` 항목에 있습니다. |
| height  | integer                                                                            | 선택      | 동영상의 고유 픽셀 높이. 각 렌디션의 실제 크기는 해당 `sources` 항목에 있습니다. |
| sources | Array\[[Media Source](/ucp-ko/draft/draft/specification/reference/#media-source)\] | 선택      | 대체 렌디션(MP4/WebM)과 적응형 매니페스트(HLS/DASH).                             |

#### 외부 동영상

서드파티 플레이어로 표시하는 동영상입니다. `url`은 플랫폼이 임베드하는 플레이어입니다. 비즈니스가 CDN을 포함한 임의의 오리진에서 파일로 제공하는 동영상은 `video`에 해당합니다. 서드파티 플레이어를 임베드하지 않는 플랫폼은 `preview`를 표시하거나 항목을 생략합니다.

플레이어를 임베드하는 플랫폼은 반드시 플레이어를 플랫폼의 권한과 격리해야 합니다(**MUST**). 플랫폼 자격 증명, 플랫폼 API나 에이전트 도구 접근, 프로토콜 메시징 권한, 호스트 애플리케이션 내비게이션 권한 중 어느 것도 주지 않습니다. 임베드하기 전에 `url`이 userinfo가 없고 플랫폼과 same-site가 아닌 절대 `https` URL이 아니라면, 반드시 항목을 거부해야 합니다(**MUST**). 웹에서 플레이어는 `src`가 `url`인 iframe 하나이며, 샌드박스와 credentialless를 적용하고 플레이어에 필요한 권한만 부여합니다.

```html
<iframe sandbox="allow-scripts allow-same-origin" allow="fullscreen" credentialless src="https://videos.example.com/embed/123"></iframe>
```

[Iframe 샌드박스 속성](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#iframe-sandbox-attributes)과 [Credentialless Iframe](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#credentialless-iframes)을 참고하세요.

| 이름    | 타입                                                                               | 필수 여부 | 설명                                                                                                                                             |
| ------- | ---------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| url     | string                                                                             | **필수**  | 플랫폼이 임베드하는 서드파티 플레이어. 비즈니스가 CDN을 포함한 어떤 출처에서든 파일로 제공하는 동영상은 `external_video`가 아니라 `video`입니다. |
| sources | Array\[[Media Source](/ucp-ko/draft/draft/specification/reference/#media-source)\] | 선택      | 사용하지 않음: 플레이어로 표시되는 동영상에는 렌디션이 없습니다.                                                                                 |

#### 3D 모델

`url`은 GLB 같은 기본 모델 파일입니다. `sources`가 있으면 glTF, USDZ 같은 대체 형식을 나열합니다.

| 이름    | 타입                                                                               | 필수 여부 | 설명                                         |
| ------- | ---------------------------------------------------------------------------------- | --------- | -------------------------------------------- |
| url     | string                                                                             | **필수**  | 기본 모델 파일(예: GLB).                     |
| width   | integer                                                                            | 선택      | 해당 없음: 3D 모델에는 픽셀 크기가 없습니다. |
| height  | integer                                                                            | 선택      | 해당 없음: 3D 모델에는 픽셀 크기가 없습니다. |
| sources | Array\[[Media Source](/ucp-ko/draft/draft/specification/reference/#media-source)\] | 선택      | 모델의 대체 파일 형식(glTF/GLB/USDZ).        |

### 상품 옵션

| 이름   | 타입                                                                               | 필수 여부 | 설명                            |
| ------ | ---------------------------------------------------------------------------------- | --------- | ------------------------------- |
| name   | string                                                                             | **필수**  | 옵션 이름(예: 'Size', 'Color'). |
| values | Array\[[Option Value](/ucp-ko/draft/draft/specification/reference/#option-value)\] | **필수**  | 이 옵션에 사용할 수 있는 값.    |

### 옵션 값

| 이름  | 타입   | 필수 여부 | 설명                                                                                                                             |
| ----- | ------ | --------- | -------------------------------------------------------------------------------------------------------------------------------- |
| id    | string | 선택      | 서버가 할당한 이 옵션 값의 식별자(선택 사항). selected_option에 있으면 서버는 label 대신 이 값을 매칭에 사용해야 합니다(SHOULD). |
| label | string | **필수**  | 이 옵션 값의 표시 텍스트(예: 'Small', 'Blue').                                                                                   |

### 선택된 옵션

| 이름  | 타입   | 필수 여부 | 설명                                                                                                                                                |
| ----- | ------ | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| name  | string | **필수**  | 옵션 이름(예: 'Size').                                                                                                                              |
| id    | string | 선택      | option_value.id에서 가져온 옵션 값 식별자(선택 사항). 있으면 서버는 이를 매칭에 사용해야 하며(SHOULD), name과 label은 표시용으로 여전히 필요합니다. |
| label | string | **필수**  | 선택한 옵션 레이블(예: 'Large').                                                                                                                    |

### 평점

| 이름      | 타입    | 필수 여부 | 설명                                     |
| --------- | ------- | --------- | ---------------------------------------- |
| value     | number  | **필수**  | 평균 평점.                               |
| scale_min | number  | 선택      | 평점 척도의 최솟값(예: 1~5점 척도면 1).  |
| scale_max | number  | **필수**  | 평점 척도의 최댓값(예: 별 5개 척도면 5). |
| count     | integer | 선택      | 평점 산정에 반영된 리뷰 수.              |

### 정책

카탈로그 응답의 상품에 적용되는 정책(반품·환불 조건, 보증 등)입니다. `applies_to`의 JSONPath 대상은 응답 루트를 기준으로 합니다. 검색과 일괄 조회에서는 `$.products[N]`, get_product에서는 `$.product`입니다. 전체 모델은 [정책](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#policies)을 참고하세요.

| 이름        | 타입                                                                                    | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ----------- | --------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| type        | [Reverse Domain Name](/ucp-ko/draft/draft/specification/reference/#reverse-domain-name) | **필수**  | 정책 유형 판별자. 열린 역 DNS 어휘입니다. 잘 알려진 정책 유형의 레지스트리는 명세 문서를 참고하세요. 비즈니스는 자체 도메인에 사용자 정의 유형을 정의할 수 있습니다(MAY, 예: `com.example.policy.price_match`). 플랫폼은 알 수 없는 값을 반드시 허용해야 합니다(MUST).                                                                                                                                                                                                                                                                                                                                                   |
| description | [Description](/ucp-ko/draft/draft/specification/reference/#description)                 | **필수**  | 하나 이상의 형식(plain, markdown, html)으로 된 사람이 읽을 수 있는 정책 요약. 플랫폼이 유형별 필드를 이해하지 못해도 표시할 수 있도록 모든 정책에 필수입니다. 이것은 구매자 대상 고지가 아닙니다. 표시 의무는 `messages[]` 경고로 부과됩니다(정책 섹션 참고).                                                                                                                                                                                                                                                                                                                                                            |
| applies_to  | Array[string]                                                                           | 선택      | 이 정책이 적용되는 노드를 가리키는 RFC 9535 JSONPath 표현식으로, 응답을 포함하는 루트를 기준으로 합니다(예: 장바구니·체크아웃의 `$.line_items[0]`, 카탈로그의 `$.products[2]`). 각 대상은 지정한 노드와 그 아래에 중첩된 모든 것을 포함하므로, 상품을 대상으로 하면 그 변형도 포함됩니다. 단수 쿼리(RFC 9535 Section 2.3.5.1, 이름·인덱스 선택자만 사용)는 노드 하나를 지정하고, 필터·와일드카드·슬라이스는 노드 집합과 일치합니다. 생략하면 정책이 응답 전체에 적용됩니다. 같은 `type`의 정책들이 한 노드를 두고 경합하면 가장 좁은 대상이 이기고 나머지를 덮어씁니다. 구체성을 판정하는 방법은 정책 섹션을 참고하세요. |
| url         | string                                                                                  | 선택      | 전체 정책 문서 링크(선택 사항).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |

## 액션

카탈로그 검색, 일괄 조회, 성공한 Get Product 응답에는 확장이 정의한 액션(Action)이 포함될 수 있습니다. 카탈로그에서 액션은 해당 타입이 정의하는 효과를 막고 있는 미처리 작업으로, 비즈니스가 어떤 상품을 반환하는지나 플랫폼이 상품을 어떻게 다루는지에 영향을 줄 수 있습니다.

검색, 일괄 조회, 성공한 Get Product는 서로 독립된 카탈로그 작업이며, 각 응답이 상위 리소스의 수명을 공유하지 않습니다. 비즈니스는 서로 다른 응답에서 같은 액션 `id`를 사용할 수 있으며(**MAY**), 값이 같다고 해서 같은 작업을 가리키지는 않습니다. 구체적인 액션 타입 계약에서 자신의 인스턴스에 대해 더 강한 대응 관계를 정의할 수는 있습니다(**MAY**).

검색과 일괄 조회에서 비즈니스는 액션 타입 계약과 자체 정책에 따라, 원래라면 관련 있었을 상품을 하나도 반환하지 않을지, 일부만 반환할지, 모두 반환할지 결정합니다. 메시지에서 액션을 가리켜 응답을 설명할 수 있습니다. 성공한 Get Product에는 여전히 `product`가 포함되며, 기존 에러 응답은 바뀌지 않습니다.

액션을 처리한 뒤 플랫폼은 카탈로그 작업을 새로 수행하며, 그 뒤의 비즈니스 응답이 권위를 가집니다. 카탈로그는 액션의 수명 주기, 폴링, 재개 동작을 정의하지 않습니다. 구체적인 액션 타입 계약에서 자신의 인스턴스 처리에 관한 그런 동작을 정의할 수 있습니다(**MAY**). 공통 형태와 규칙은 [개요 — 액션](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#actions)에 정의되어 있습니다.

예를 들어 다음 검색 응답은 상품을 반환하지 않고, 연령 확인이 결과에 영향을 줄 수 있다고 설명합니다.

```json
{
  "ucp": {...},
  "products": [],
  "actions": {
    "com.example.identity.age_verification": [
      {
        "id": "age-check-1"
      }
    ]
  },
  "messages": [
    {
      "type": "info",
      "code": "age_verification_required",
      "content": "Complete age verification to see age-restricted products matching your search.",
      "path": "$.actions['com.example.identity.age_verification'][0]"
    }
  ]
}
```

상품을 하나도 반환하지 않는 것은 비즈니스가 고를 수 있는 방법 중 하나일 뿐이며, 일부 또는 전체 결과를 반환해도 명세에 부합합니다. 액션 타입과 메시지 코드는 예시이며, 카탈로그는 둘 다 정의하지 않습니다.

## 메시지와 에러 처리

모든 카탈로그 응답에는 선택 사항인 `messages` 배열이 있어, 비즈니스가 에러, 경고, 정보성 알림의 맥락을 전달할 수 있습니다.

### 메시지 타입

메시지는 비즈니스의 처리 결과를 전달하고 맥락을 제공합니다.

| 타입      | 사용 시점                      | 예시 코드                                        |
| --------- | ------------------------------ | ------------------------------------------------ |
| `error`   | 비즈니스 수준 에러             | `NOT_FOUND`, `OUT_OF_STOCK`, `REGION_RESTRICTED` |
| `warning` | 구매에 영향을 주는 중요한 조건 | `DELAYED_FULFILLMENT`, `FINAL_SALE`              |
| `info`    | 문제없이 덧붙이는 추가 맥락    | `PROMOTIONAL_PRICING`, `LIMITED_AVAILABILITY`    |

`presentation: "disclosure"`가 있는 경고에는 플랫폼이 숨기거나 닫아서는 안 되는 고지(예: 알레르기 유발 성분 표시, 안전 경고)가 담깁니다. 전체 렌더링 계약은 [경고 표시](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#warning-presentation)를 참고하세요.

**참고**: 대부분의 카탈로그 에러는 `severity: "recoverable"`을 사용하며, 에이전트가 이를 프로그램으로 처리합니다(재시도, 사용자에게 알림, 대안 제시). `get_product`는 식별자를 해석할 수 없으면 `severity: "unrecoverable"`을 반환하며, 이때 에이전트는 같은 `id`로 재시도해서는 안 됩니다(MUST NOT)([REST](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/rest/#product-not-found)와 [MCP](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/mcp/#product-not-found) 예시 참고).

#### 메시지 (에러)

| 이름         | 타입                                                                  | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------ | --------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                | **필수**  | **상수 = error**. 메시지 유형 판별자.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| code         | [Error Code](/ucp-ko/draft/draft/specification/reference/#error-code) | **필수**  | 오류 유형을 식별하는 오류 코드. 표준 오류는 기능 명세에 정의되어 있고(예시 참고) 의미가 표준화되어 있으며, 자유 형식 코드도 허용합니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| path         | string                                                                | 선택      | 메시지가 가리키는 구성 요소의 RFC 9535 JSONPath(예: $.line_items[0]).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| content_type | string                                                                | 선택      | 콘텐츠 형식. 기본값은 plain. **허용 값:** `plain`, `markdown`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| content      | string                                                                | **필수**  | 사람이 읽을 수 있는 메시지.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| severity     | string                                                                | **필수**  | 리소스 상태와 권장 조치를 나타냅니다. 'recoverable': 플랫폼이 입력을 수정하거나 관련 Action을 처리하는 등 대역 내에서 상황을 해결하고, 필요하면 새 작업을 제출할 수 있습니다. 'requires_buyer_input': 판매자의 API로는 프로그래밍 방식으로 수집할 수 없는 정보가 필요합니다(체크아웃 미완료). 'requires_buyer_review': 정책, 규제, 자격 규칙 때문에 주문 전에 구매자가 승인해야 합니다. 'unrecoverable': 조치할 유효한 리소스가 없으므로 새 리소스나 입력으로 재시도해야 합니다. 'requires\_*' 심각도의 오류는 'status: requires_escalation'의 원인이 됩니다.* *허용 값:*\* `recoverable`, `requires_buyer_input`, `requires_buyer_review`, `unrecoverable` |

#### 메시지 (경고)

| 이름         | 타입                                                                      | 필수 여부 | 설명                                                                                                                                                                                                                                                                            |
| ------------ | ------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                    | **필수**  | **상수 = warning**. 메시지 유형 판별자.                                                                                                                                                                                                                                         |
| path         | string                                                                    | 선택      | 메시지가 가리키는 구성 요소의 RFC 9535 JSONPath(예: $.line_items[0]).                                                                                                                                                                                                           |
| code         | [Warning Code](/ucp-ko/draft/draft/specification/reference/#warning-code) | **필수**  | 경고 유형을 식별하는 경고 코드. 표준 코드는 기능 명세에 정의되어 있고(예시 참고) 표준화된 의미를 지닙니다. 자유 형식 코드도 허용됩니다.                                                                                                                                         |
| content      | string                                                                    | **필수**  | 반드시 표시해야 하는(MUST) 사람이 읽을 수 있는 경고 메시지.                                                                                                                                                                                                                     |
| content_type | string                                                                    | 선택      | 콘텐츠 형식. 기본값은 plain. **허용 값:** `plain`, `markdown`                                                                                                                                                                                                                   |
| presentation | string                                                                    | 선택      | 이 경고의 렌더링 규약입니다. 'notice'(기본값): 플랫폼은 반드시 표시해야 하며(MUST), 닫을 수 있습니다(MAY). 'disclosure': 플랫폼은 반드시 path가 가리키는 구성 요소 가까이에 표시해야 하며(MUST), 숨기거나 자동으로 닫아서는 안 됩니다(MUST NOT). 전체 규약은 명세를 참고하세요. |
| image_url    | string                                                                    | 선택      | 필수 시각 요소(예: 경고 기호, 에너지 등급 라벨)의 URL.                                                                                                                                                                                                                          |
| url          | string                                                                    | 선택      | 추가 정보를 위한 참조 URL(예: 규제 기관 사이트, 등록부 항목, 정책 페이지).                                                                                                                                                                                                      |

#### 메시지 (정보)

| 이름         | 타입                                                                | 필수 여부 | 설명                                                                                                                                           |
| ------------ | ------------------------------------------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                              | **필수**  | **상수 = info**. 메시지 유형 판별자.                                                                                                           |
| path         | string                                                              | 선택      | 메시지가 가리키는 구성 요소의 RFC 9535 JSONPath(예: $.line_items[0]).                                                                          |
| code         | [Info Code](/ucp-ko/draft/draft/specification/reference/#info-code) | 선택      | 정보 메시지의 유형을 식별하는 정보 코드. 표준 코드는 기능 명세에 정의되어 있고(예시 참고) 표준화된 의미를 가지며, 자유 형식 코드도 허용됩니다. |
| content_type | string                                                              | 선택      | 콘텐츠 형식. 기본값은 plain. **허용 값:** `plain`, `markdown`                                                                                  |
| content      | string                                                              | **필수**  | 사람이 읽을 수 있는 메시지.                                                                                                                    |

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

에이전트는 각 변형의 `inputs` 배열로 결과의 대응 관계를 파악합니다. [클라이언트 대응](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/lookup/#client-correlation)을 참고하세요.

#### 상품 고지

상품에 고지(예: 알레르기 유발 성분 안내, 안전 경고)가 필요하면 `presentation: "disclosure"`가 있는 경고로 반환합니다. `path` 필드는 응답에서 관련 구성 요소를 가리키며, 상품을 가리키면 고지가 그 상품의 모든 변형에 적용됩니다.

```json
{
  "ucp": {...},
  "products": [
    {
      "id": "prod_nut_butter",
      "title": "Artisan Nut Butter Collection",
      "description": { "plain": "Assorted artisan nut butters." },
      "price_range": {
        "min": { "amount": 1299, "currency": "USD" },
        "max": { "amount": 1499, "currency": "USD" }
      },
      "variants": [
        {
          "id": "var_almond",
          "title": "Almond Butter",
          "description": { "plain": "Smooth almond butter." },
          "price": { "amount": 1299, "currency": "USD" },
          "availability": { "available": true }
        },
        {
          "id": "var_cashew",
          "title": "Cashew Butter",
          "description": { "plain": "Creamy cashew butter." },
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

전체 렌더링 계약은 [경고 표시](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#warning-presentation)를 참고하세요.

## 스코프

카탈로그 검색과 카탈로그 조회 기능은 사용자 인증 접근을 위해 다음과 같은 잘 알려진 스코프를 정의합니다.

| 스코프                                 | 설명                                                                                             |
| -------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `dev.ucp.shopping.catalog.search:read` | 인증된 사용자를 대신해 검색합니다. 개인화된 결과, 회원 가격, 접근이 제한된 재고 등이 해당합니다. |
| `dev.ucp.shopping.catalog.lookup:read` | 인증된 사용자를 대신해 조회합니다. 특정 상품의 개인화된 가격이나 재고 상태 등이 해당합니다.      |

스코프 선언과 도출, 사용자 정의 스코프로 이 집합을 확장하는 규칙은 [신원 연결 — 스코프](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/identity-linking/#scopes)에 정의되어 있습니다.

## 전송 바인딩

위 기능은 다음 전송 프로토콜에 바인딩됩니다.

- [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/rest/index.md): RESTful API 매핑
- [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/mcp/index.md): JSON-RPC 기반 Model Context Protocol 매핑
