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

# 카탈로그 기능 { #catalog-capability }

## 개요 { #overview }

카탈로그 기능(Capability)은 플랫폼이 비즈니스의 상품 카탈로그를 검색하고 탐색할 수
있게 한다. 이를 통해 체크아웃 전에 상품을 찾을 수 있으며, 다음과 같은 사용 사례를
지원한다.

* 자유 텍스트 상품 검색
* 카테고리 및 필터 기반 탐색
* 식별자로 상품/변형 일괄 조회
* 변형 간 가격 비교

## 기능 { #capabilities }

| 기능 | 설명 |
| :--- | :--- |
| [`dev.ucp.shopping.catalog.search`](search.md) | 쿼리 텍스트와 필터로 상품을 검색한다. |
| [`dev.ucp.shopping.catalog.lookup`](lookup.md) | 식별자로 상품 또는 변형을 조회한다. |

## 핵심 개념 { #key-concepts }

* **상품(Product)**: 제목, 설명, 미디어, 하나 이상의 변형을 가진 카탈로그 항목.
* **변형(Variant)**: 특정 옵션 선택(예: "Blue / Large"), 가격, 재고 상태를 가진 구매
  가능한 항목.
* **가격(Price)**: 가격 값에는 금액(통화의 최소 단위)과 통화 코드가 모두 포함되어
  다중 통화 카탈로그를 지원한다.
* **판매 기준(Sale basis)**: 수량을 표시하는 방식. 개수 단위(`each`, 기본값)이거나
  무게, 길이, 면적, 부피, 시간 같은 측정 단위다.
  [수량과 단위](../../overview/index.md#quantities-and-units)를 참고한다.

### 체크아웃과의 관계 { #relationship-to-checkout }

카탈로그 작업은 체크아웃의 `line_items[].item.id`에 바로 쓸 수 있는 상품 ID와 변형
ID를 반환한다. 카탈로그 조회로 얻은 변형 ID는 체크아웃이 기대하는 아이템 ID와 일치해야
한다.

카탈로그 응답(가격, 재고 상태 등)은 해당 요청에 대한 비즈니스의 현재 조건을 반영하지만
거래상의 확약은 아니다. 권위는 체크아웃에 있다. 응답은 세션별로 다를 수 있으며, 재검증
없이 다른 세션에서 재사용하지 않아야 한다(**SHOULD NOT**).

## 판매 기준과 수량 단위 { #sale-basis-and-quantity-units }

`variants[].quantity_unit`은 거래 전에 변형의 판매 기준을 알린다. 이 서술자는 공통
[수량과 단위](../../overview/index.md#quantities-and-units) 계약을 따른다. 서술자가
없으면 기본값인 `each` 기준을 뜻하며, 비즈니스는 서술자를 포함해 `each`가 아닌 기준을
알린다.

플랫폼은 카탈로그에서 거래 전에 판매 기준을 파악한다. `unit`과 `scale`은 수량을
표시하는 방식을 정의하고, 선택 사항인
[`increment`](../../overview/index.md#ordering-increment)는 비즈니스가 판매하는 주문
단위를 알린다. 이를 바탕으로 플랫폼은 수량 조절 UI를 만들고 제출 전에 입력을 검증할 수
있다. 바나나를 파운드 단위로, 1/4파운드 배수로 판매하는 비즈니스는
`{ "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }`를 알린다.

판매 기준은 물리적 측정에만 국한되지 않는다. 분 단위 주차, 시간 단위 인건비 같은 계량형
상품도 같은 계약에 따라 같은 서술자(`MIN`, `HUR`)를 사용한다.

`variants[].id`는 구매 가능한 변형을 식별하고, `quantity_unit`은 그 변형을 주문할 때
쓰는 단위와 세분성을 정의한다. `quantity_unit`이
`{ "unit": "LBR", "scale": 2, "display_text": "lb" }`인 바나나는 1/100파운드 단위로
판매되므로, `quantity`가 `150`이면 1.50 lb를 뜻한다.

### 단가와의 구분 { #distinction-from-unit-price }

`quantity_unit`과 변형의 [`unit_price`](#variant)는 서로 다른 질문에 답하며 독립적으로
설정된다.

* `quantity_unit`은 **판매 기준**이다. `quantity`를 세는 단위이자 `price`를 매기는
  단위다.
* `unit_price`는 **표시용 비교 지표**다. 매대식 비교를 위해 파생한 "표준 측정량당
  가격"(예: 100 mL당)이다. `measure`는 변형의 포장 또는 내용량이고, `reference`는 비교
  기준 분모다.

`quantity_unit`, `unit_price.measure`, `unit_price.reference`의 단위 필드는 공통
[수량과 단위](../../overview/index.md#quantities-and-units) 계약을 따른다. 두 단가
필드는 공통 측정 타입을 사용하므로 정수 `value` 필드는 단계 수다. 비즈니스는 변형에
`quantity_unit`, `unit_price`, 둘 다, 또는 둘 다 제공하지 않을 수 있다(**MAY**).

표시용 비교 지표가 정의된 상태로 유지되도록, 비즈니스는 반드시
`unit_price.measure.value`와 `unit_price.reference.value` 모두에 양의 정수를 사용해야
한다(**MUST**). 비즈니스는 반드시 `unit_price.currency`를 `price.currency`와 같게
설정해야 한다(**MUST**). `unit_price` 안에서 비즈니스는 반드시 `measure.unit`과
`reference.unit`에 동일한 값을 사용해야 한다(**MUST**). 두 측정값의 `scale` 값은 서로
달라도 된다(**MAY**). 비즈니스는 단가 계산의 일부로 단위 간 변환이나 통화 변환을 해서는
안 된다(**MUST NOT**). 각 측정값은 공통 단위로 정수 `value × 10^-scale`를 나타낸다.
비즈니스는 반드시 `price.amount`와 이 스케일 적용 값으로 비교 지표를 계산해야
한다(**MUST**).

```text
(price.amount / (measure.value × 10^-measure.scale)) × (reference.value × 10^-reference.scale)
```

비즈니스는 반드시 가격 규칙에 따라 결과를 통화의 최소 단위로 한 번 반올림해
`unit_price.amount`로 반환해야 한다(**MUST**). 반환된 `unit_price.amount`가 권위를
가진다. 플랫폼은 이를 다시 계산하거나 자체 결과로 대체해서는 안 된다(**MUST NOT**).

동일 단위·동일 통화 규칙은 의미상의 불변 조건이다. JSON Schema는 해당 필드를 각각
독립적으로 검증하며, 두 동일성 중 어느 것도 강제하지 않는다.

예를 들어 `each`로 판매하는 50 m 케이블 스풀은 `quantity_unit`을 생략하고 미터당
`unit_price`를 가질 수 있다. `measure`는
`{ "value": 5000, "unit": "MTR", "scale": 2, "display_text": "m" }`, `reference`는
`{ "value": 1, "unit": "MTR", "display_text": "m" }`일 수 있다. 두 측정값 모두 `MTR`을
사용하며, 단위 간 변환 없이 각각 50 m와 1 m를 나타낸다.

거래 라인에서는 존재 여부가 역할을 나타낸다. 비즈니스는 가격 기준이 판매 기준과 다른
모든 라인에 반드시 `unit_price`를 그대로 담아 보내야 한다(**MUST**)([체크아웃 — 가격
기준](../checkout/index.md#quantity-and-sale-basis) 참고). 이 스풀의 미터당 가격처럼
카탈로그에만 있는 `unit_price`는 표시용 비교 지표이고, 라인 수준의 `unit_price`는
청구 금액 계산의 기준 요율을 담는다.

### 예시: 무게 단위로 판매하는 상품 { #example-a-good-sold-by-weight }

파운드 단위로 판매하는 바나나에 대한 `get_product` 응답이다. 변형은 `quantity_unit`으로
`{ "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }`를, `price`로
`79`를 알린다. 즉 1파운드당 $0.79이며 0.25 lb 배수로 판매한다.

<!-- ucp:example schema=shopping/catalog_lookup op=get_product -->
```json
{
  "ucp": { "version": "{{ ucp_version }}" },
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

## 공통 엔터티 { #shared-entities }

### 컨텍스트 { #context }

카탈로그 작업을 위한 위치 및 시장 컨텍스트다. 모든 필드는 관련성과 현지화를 위한 선택적
힌트다. 플랫폼은 요청 헤더로 컨텍스트를 지리적으로 감지할 수 있다(MAY).

컨텍스트 신호는 잠정적인 값이며 권위 있는 데이터가 아니다. 비즈니스는 검증된 입력(예:
배송 주소)이 없을 때 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정,
위험 탐지)나 규제상 제약(수출 통제)과 일치하지 않으면 무시하거나 순위를 낮출 수
있다(MAY). 자격과 정책 적용은 반드시 구속력 있는 거래 데이터를 사용해 체크아웃 시점에
이루어져야 한다(MUST).

비즈니스는 컨텍스트 신호를 바탕으로 통화를 포함한 시장 배정을 결정한다. 가격 필터 값은
`context.currency` 기준으로 표시되며, 표시 통화가 다르면 비즈니스는 적용 전에 변환해야
한다(SHOULD)([가격 필터](search.md#price-filter) 참고). 응답 가격에는 결정 결과를
확인해 주는 명시적인 통화 코드가 포함된다.

`context.eligibility` 주장이 있을 때, 이를 수락한 비즈니스는 취소선 표시를 위해
`price` / `list_price`를 직접 조정할 수 있으며(**MAY**), `code: "eligibility_benefit"`을
가진 `messages`로 조정 내용을 특정 주장에 귀속시킬 수 있다(**MAY**).

{{ schema_fields('types/context', 'shopping/catalog') }}

### 신호 { #signals }

인가와 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터다. 신호 값은 구매자가
주장한 내용이어서는 안 된다(MUST NOT). 자세한 내용과 개인정보 보호 요건은
[신호](../../overview/index.md#signals)를 참고한다.

{{ schema_fields('types/signals', 'shopping/catalog') }}

### 어트리뷰션 { #attribution }

플랫폼이 제공하는 유입 및 전환 이벤트 컨텍스트로, 플랫폼이 전달하는 캠페인 ID, 클릭
식별자, 소스/매체 표식 등이다. 자세한 내용과 동의 요건은
[어트리뷰션](../../overview/index.md#attribution)을 참고한다.

{{ schema_fields('types/attribution', 'shopping/catalog') }}

### 상품 { #product }

하나 이상의 구매 가능한 변형을 가진, 판매 가능한 항목을 나타내는 카탈로그 항목이다.

`media`와 `variants`는 순서가 있는 배열이다. 비즈니스는 가장 관련성 높은 변형과 미디어
항목을 맨 앞에 반환해야 한다(**SHOULD**). 조회에서는 기본값을, 검색에서는 쿼리와
컨텍스트에 가장 잘 맞는 항목을 앞에 둔다. 플랫폼은 자신이 표시하는 첫 번째 미디어 항목을
대표 항목으로 취급해야 한다(**SHOULD**).

{{ schema_fields('types/product', 'shopping/catalog') }}

### 변형 { #variant }

특정 옵션 선택, 가격, 재고 상태를 가진 구매 가능한 항목이다.

조회 응답에서 각 변형은 대응 관계 파악을 위한 `inputs` 배열을 가진다. 어떤 요청
식별자가 이 변형으로 해석되었는지, 그리고 일치가 `exact`였는지 `featured`(서버가
선택)였는지를 담는다. 자세한 내용은 [클라이언트 대응](lookup.md#client-correlation)을
참고한다.

`media`는 순서가 있는 배열이다. 비즈니스는 변형의 대표 미디어 항목을 첫 번째 요소로
반환해야 한다(**SHOULD**). 플랫폼은 자신이 표시하는 첫 번째 미디어 항목을 대표 항목으로
취급해야 한다(**SHOULD**).

{{ schema_fields('types/variant', 'shopping/catalog') }}

### 가격 { #price }

{{ schema_fields('types/price', 'shopping/catalog') }}

### 가격 범위 { #price-range }

{{ schema_fields('types/price_range', 'shopping/catalog') }}

### 미디어 { #media }

상품 또는 변형에 속한 하나의 논리적 미디어 항목으로, 이미지, 동영상, 3D 모델 등이다.
미디어 항목은 플랫폼이 렌더링하는 리소스다. `url`은 항목을 식별하고, `sources`는 대체
렌디션을 나열하며, `preview`는 플랫폼이 항목 대신 또는 항목보다 먼저 보여 줄 수 있는
정지 이미지이고, `width`와 `height`는 고유 픽셀 크기다. `preview`와 각 `sources` 항목의
`width`와 `height`는 해당 정지 이미지 또는 렌디션의 픽셀 크기다.

`type`은 개방형 문자열이며 잘 알려진 값으로 `image`, `video`, `external_video`,
`model_3d` 네 가지가 있다. 플랫폼은 미디어 항목의 `type`을 인식하지 못한다는 이유로
상품이나 응답을 거부해서는 안 된다(**MUST NOT**). 그런 항목은 `preview`와 `name`으로
표시하거나 생략할 수 있다(**MAY**). `type`을 인식하면 타입별 표시가 가능해질 뿐이다.

플랫폼은 `https` 스킴을 사용하지 않는 미디어 URL의 로드나 임베드를 거부할 수
있다(**MAY**).

플랫폼은 `mime_type`과 크기를 기준으로 `sources`에서 렌디션을 선택한다. `sources`를
제공하는 비즈니스는 `url`에 해당하는 렌디션도 포함해야 한다(**SHOULD**).

{{ schema_fields('types/media', 'shopping/catalog') }}

#### 이미지 { #image }

`url`은 바로 표시할 수 있는 이미지다. `sources`가 있으면 해당 이미지의 대체 인코딩과
크기를 나열한다. 예를 들어 `image/avif`, `image/webp`, `image/jpeg`이나 반응형
레이아웃을 위한 너비별 변형이다. `sources`에서 선택하지 않는 플랫폼은 `url`을 바로
로드하므로, 비즈니스는 `url`을 가장 폭넓게 디코딩 가능한 렌디션으로 지정해야
한다(**SHOULD**).

{{ schema_fields('types/media_image', 'shopping/catalog') }}

#### 동영상 { #video }

`url`은 동영상의 재생 가능한 렌디션으로, 프로그레시브 파일이거나 적응형 스트리밍
매니페스트다. `sources`가 있으면 대체 렌디션과 매니페스트를 나열한다.

{{ schema_fields('types/media_video', 'shopping/catalog') }}

#### 외부 동영상 { #external-video }

서드파티 플레이어를 통해 표시하는 동영상이다. `url`은 플랫폼이 임베드하는 플레이어다.
비즈니스가 CDN을 포함한 임의의 오리진에서 파일로 제공하는 동영상은 `video`다. 서드파티
플레이어를 임베드하지 않는 플랫폼은 `preview`를 표시하거나 항목을 생략한다.

플레이어를 임베드하는 플랫폼은 반드시 플레이어를 플랫폼의 권한으로부터 격리해야
한다(**MUST**). 플랫폼 자격 증명도, 플랫폼 API나 에이전트 도구에 대한 접근도, 프로토콜
메시징 권한도, 호스트 애플리케이션 내비게이션 권한도 주지 않는다. 임베드하기 전에,
`url`이 userinfo가 없고 플랫폼과 same-site가 아닌 절대 `https` URL이 아니라면 반드시
항목을 거부해야 한다(**MUST**). 웹에서 플레이어는 `src`가 `url`인 iframe 하나이며,
샌드박스와 credentialless를 적용하고 플레이어에 필요한 권한만 부여한다.

```html
<iframe sandbox="allow-scripts allow-same-origin" allow="fullscreen" credentialless src="https://videos.example.com/embed/123"></iframe>
```

[Iframe 샌드박스 속성](../../embedded-protocol.md#iframe-sandbox-attributes)과
[Credentialless Iframe](../../embedded-protocol.md#credentialless-iframes)을
참고한다.

{{ schema_fields('types/media_external_video', 'shopping/catalog') }}

#### 3D 모델 { #3d-model }

`url`은 GLB 같은 기본 모델 파일이다. `sources`가 있으면 glTF, USDZ 같은 대체 형식을
나열한다.

{{ schema_fields('types/media_model_3d', 'shopping/catalog') }}

### 상품 옵션 { #product-option }

{{ schema_fields('types/product_option', 'shopping/catalog') }}

### 옵션 값 { #option-value }

{{ schema_fields('types/option_value', 'shopping/catalog') }}

### 선택된 옵션 { #selected-option }

{{ schema_fields('types/selected_option', 'shopping/catalog') }}

### 평점 { #rating }

{{ schema_fields('types/rating', 'shopping/catalog') }}

### 정책 { #policy }

카탈로그 응답의 상품에 적용되는 정책(반품·환불 조건, 보증 등)이다. `applies_to`의
JSONPath 대상은 응답 루트를 기준으로 한다. 검색과 일괄 조회에서는 `$.products[N]`,
get_product에서는 `$.product`다. 전체 모델은 [정책](../../overview/index.md#policies)을 참고한다.

{{ schema_fields('types/policy', 'shopping/catalog') }}

## 액션 { #actions }

카탈로그 검색, 일괄 조회, 성공한 Get Product 응답에는 확장이 정의한 액션(Action)이
포함될 수 있다. 카탈로그에서 액션은 그 타입이 정의하는 효과를 가로막는 미처리 작업이며,
비즈니스가 어떤 상품을 반환하는지 또는 플랫폼이 상품을 어떻게 다루는지에 영향을 줄 수
있다.

검색, 일괄 조회, 성공한 Get Product는 서로 독립된 카탈로그 작업이며, 그 응답들은 상위
리소스의 수명을 공유하지 않는다. 비즈니스는 서로 다른 응답에서 같은 액션 `id`를 사용할
수 있으며(**MAY**), 그렇게 값이 같다고 해서 같은 작업을 가리키지는 않는다. 구체적인 액션
타입 계약은 자신의 인스턴스에 대해 더 강한 대응 관계를 정의할 수 있다(**MAY**).

검색과 일괄 조회에서 비즈니스는 액션 타입 계약과 자체 정책에 따라, 원래라면 관련 있었을
상품을 하나도 반환하지 않을지, 일부만 반환할지, 모두 반환할지 결정한다. 메시지는 액션을
가리켜 응답을 설명할 수 있다. 성공한 Get Product에는 여전히 `product`가 포함되며, 기존
에러 응답은 바뀌지 않는다.

액션을 처리한 뒤 플랫폼은 새로운 카탈로그 작업을 수행하며, 이후의 비즈니스 응답이
권위를 가진다. 카탈로그는 액션의 수명 주기, 폴링, 재개 동작을 정의하지 않는다. 구체적인
액션 타입 계약이 자신의 인스턴스 처리에 대해 그러한 동작을 정의할 수 있다(**MAY**).
공통 형태와 규칙은 [개요 — 액션](../../overview/index.md#actions)에 정의되어 있다.

예를 들어 다음 검색 응답은 상품을 반환하지 않으며, 연령 확인이 결과에 영향을 줄 수
있음을 설명한다.

<!-- ucp:example schema=shopping/catalog_search op=search -->
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

상품을 하나도 반환하지 않는 것은 비즈니스의 선택지 중 하나일 뿐이며, 일부 또는 전체
결과를 반환해도 명세에 부합한다. 액션 타입과 메시지 코드는 예시이며, 카탈로그는 둘 다
정의하지 않는다.

## 메시지와 에러 처리 { #messages-and-error-handling }

모든 카탈로그 응답에는 선택 사항인 `messages` 배열이 있어, 비즈니스가 에러, 경고, 정보성
알림에 관한 맥락을 제공할 수 있다.

### 메시지 타입 { #message-types }

메시지는 비즈니스 처리 결과를 전달하고 맥락을 제공한다.

| 타입 | 사용 시점 | 예시 코드 |
| :--- | :--- | :--- |
| `error` | 비즈니스 수준 에러 | `NOT_FOUND`, `OUT_OF_STOCK`, `REGION_RESTRICTED` |
| `warning` | 구매에 영향을 주는 중요한 조건 | `DELAYED_FULFILLMENT`, `FINAL_SALE` |
| `info` | 문제없이 덧붙이는 추가 맥락 | `PROMOTIONAL_PRICING`, `LIMITED_AVAILABILITY` |

`presentation: "disclosure"`가 있는 경고는 플랫폼이 숨기거나 닫아서는 안 되는 고지(예:
알레르기 유발 성분 표시, 안전 경고)를 담는다. 전체 렌더링 계약은
[경고 표시](../checkout/index.md#warning-presentation)를 참고한다.

**참고**: 대부분의 카탈로그 에러는 `severity: "recoverable"`을 사용하며, 에이전트는 이를
프로그램적으로 처리한다(재시도, 사용자에게 알림, 대안 제시). `get_product`는 식별자를
해석할 수 없을 때 `severity: "unrecoverable"`을 반환하며, 에이전트는 같은 `id`로
재시도해서는 안 된다(MUST NOT)([REST](rest.md#product-not-found)와
[MCP](mcp.md#product-not-found) 예시 참고).

#### 메시지 (에러) { #message-error }

{{ schema_fields('types/message_error', 'shopping/catalog') }}

#### 메시지 (경고) { #message-warning }

{{ schema_fields('types/message_warning', 'shopping/catalog') }}

#### 메시지 (정보) { #message-info }

{{ schema_fields('types/message_info', 'shopping/catalog') }}

### 일반적인 시나리오 { #common-scenarios }

#### 빈 검색 결과 { #empty-search }

검색 결과가 없으면 메시지 없이 빈 배열을 반환한다.

<!-- ucp:example schema=shopping/catalog_search op=search -->
```json
{
  "ucp": {...},
  "products": []
}
```

이는 에러가 아니다. 쿼리는 유효했지만 결과가 없었을 뿐이다.

#### 이월 주문 경고 { #backorder-warning }

상품은 구매 가능하지만 풀필먼트가 지연되는 경우, 경고 메시지와 함께 상품을 반환한다.
특정 변형을 가리키려면 `path` 필드를 사용한다.

<!-- ucp:example schema=shopping/catalog_search op=search -->
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

에이전트는 해당 옵션을 보여 주면서 사용자에게 지연 사실을 알릴 수 있다. `path` 필드는
RFC 9535 JSONPath로 특정 구성 요소를 가리킨다.

#### 식별자를 찾을 수 없음 { #identifiers-not-found }

요청한 식별자가 존재하지 않으면, 찾은 상품(있다면)과 함께 성공으로 반환한다. 응답에는
찾지 못한 식별자를 알리는 정보성 메시지를 포함할 수 있다(MAY).

<!-- ucp:example schema=shopping/catalog_lookup op=lookup -->
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

에이전트는 각 변형의 `inputs` 배열로 결과의 대응 관계를 파악한다.
[클라이언트 대응](lookup.md#client-correlation)을 참고한다.

#### 상품 고지 { #product-disclosure }

상품에 고지(예: 알레르기 유발 성분 안내, 안전 경고)가 필요하면
`presentation: "disclosure"`를 가진 경고로 반환한다. `path` 필드는 응답에서 관련 구성
요소를 가리키며, 상품을 가리키는 경우 고지는 그 상품의 모든 변형에 적용된다.

<!-- ucp:example schema=shopping/catalog_search op=search -->
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

전체 렌더링 계약은 [경고 표시](../checkout/index.md#warning-presentation)를 참고한다.

## 스코프 { #scopes }

카탈로그 검색과 카탈로그 조회 기능은 사용자 인증 접근을 위해 다음과 같은 잘 알려진
스코프를 정의한다.

| 스코프 | 설명 |
| :--- | :--- |
| `dev.ucp.shopping.catalog.search:read` | 인증된 사용자를 대신해 검색한다. 개인화된 결과, 회원 가격, 접근이 제한된 재고 등. |
| `dev.ucp.shopping.catalog.lookup:read` | 인증된 사용자를 대신해 조회한다. 특정 상품에 대한 개인화된 가격이나 재고 상태 등. |

스코프 선언, 도출, 그리고 사용자 정의 스코프로 이 집합을 확장하는 규칙은
[신원 연결 — 스코프](../../common/identity-linking/index.md#scopes)에 정의되어 있다.

## 전송 바인딩 { #transport-bindings }

위 기능들은 특정 전송 프로토콜에 바인딩된다.

* [REST 바인딩](rest.md): RESTful API 매핑.
* [MCP 바인딩](mcp.md): JSON-RPC를 통한 Model Context Protocol 매핑.
