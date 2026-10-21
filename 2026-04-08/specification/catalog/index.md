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

| 이름            | 타입                                                                                                       | 필수   | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| --------------- | ---------------------------------------------------------------------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| address_country | string                                                                                                     | 아니요 | 국가. 2자리 ISO 3166-1 alpha-2 형식(예: "US")을 권장합니다. 하위 호환성을 위해 "SGP" 같은 3자리 ISO 3166-1 alpha-3 국가 코드나 "Singapore" 같은 전체 국가명도 사용할 수 있습니다. 시장 맥락(통화, 구매 가능 여부, 가격)을 위한 선택적 힌트이며, 더 정밀한 데이터(예: 배송지 주소)가 있으면 이 값보다 우선합니다.                                                                                                                                |
| address_region  | string                                                                                                     | 아니요 | 시·군·구가 속하고 해당 국가 안에 있는 지역. 예: California 또는 그 밖의 적절한 1차 행정 구역. 점진적 현지화를 위한 선택적 힌트이며, 더 정밀한 데이터(예: 배송지 주소)가 있으면 이 값보다 우선합니다.                                                                                                                                                                                                                                            |
| postal_code     | string                                                                                                     | 아니요 | 우편번호. 예: 94043. 지역 단위 세분화를 위한 선택적 힌트이며, 더 정밀한 데이터(예: 배송지 주소)가 있으면 이 값보다 우선합니다.                                                                                                                                                                                                                                                                                                                  |
| intent          | string                                                                                                     | 아니요 | 구매자의 의도를 설명하는 배경 맥락(예: 'looking for a gift under $50', 'need something durable for outdoor use'). 관련성, 추천, 개인화에 활용됩니다.                                                                                                                                                                                                                                                                                            |
| language        | string                                                                                                     | 아니요 | 콘텐츠의 선호 언어입니다. IETF BCP 47 언어 태그(예: 'en', 'fr-CA', 'zh-Hans')를 사용합니다. REST에서는 Accept-Language 헤더에 해당합니다. 이 필드가 없으면 플랫폼은 Accept-Language로 대체해야 하며(SHOULD), 값이 있으면 Accept-Language보다 우선합니다. 해당 언어를 제공할 수 없으면 비즈니스는 다른 언어로 콘텐츠를 반환할 수 있습니다(MAY).                                                                                                  |
| currency        | string                                                                                                     | 아니요 | 선호 통화(ISO 4217, 예: 'EUR', 'USD'). 비즈니스는 맥락과 신뢰할 수 있는 신호로 표시 통화를 결정하며, 다중 통화 시장에서는 이 힌트를 선택에 참고할 수 있습니다(MAY). 가격 필터 값의 통화 단위로도 쓰이므로, 플랫폼은 가격 필터를 보낼 때 이 필드를 포함해야 합니다(SHOULD). 응답 가격에는 결정된 통화를 확인할 수 있도록 통화가 명시됩니다.                                                                                                      |
| eligibility     | Array\[[Reverse Domain Name](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#reverse-domain-name)\] | 아니요 | 로열티 멤버십, 결제 수단 혜택 등 적격 혜택에 대한 구매자 주장. 인식된 주장은 비즈니스 응답에 반영될 수 있습니다(MAY)(예: 회원 전용 상품 구매 가능 여부, 카탈로그의 조정 가격, 장바구니나 체크아웃의 잠정 할인). 비즈니스는 인식하지 못한 값을 오류 없이 반드시 무시해야 합니다(MUST). 값은 반드시 역 도메인 이름 규칙을 사용해야 하며(MUST)(예: 'com.example.loyalty_gold', 'org.school.student'), 반드시 개인을 식별할 수 없어야 합니다(MUST). |

### 신호

인가와 악용 방지에 도움이 되도록 플랫폼이 제공하는 환경 데이터입니다. 신호 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 자세한 내용과 개인정보 보호 요건은 [신호](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/#signals)를 참고하세요.

| 이름               | 타입   | 필수   | 설명                                                     |
| ------------------ | ------ | ------ | -------------------------------------------------------- |
| dev.ucp.buyer_ip   | string | 아니요 | 클라이언트의 IP 주소(IPv4 또는 IPv6).                    |
| dev.ucp.user_agent | string | 아니요 | 클라이언트의 HTTP User-Agent 헤더 또는 그에 상응하는 값. |

### 어트리뷰션

플랫폼이 제공하는 유입 및 전환 이벤트 컨텍스트로, 플랫폼이 전달하는 캠페인 ID, 클릭 식별자, 소스/매체 표식 등이 해당합니다. 자세한 내용과 동의 요건은 [어트리뷰션](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/#attribution)을 참고하세요.

플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.

### 상품

판매 가능한 항목을 나타내는 카탈로그 항목으로, 구매 가능한 변형이 하나 이상 있습니다.

`media`와 `variants`는 순서가 있는 배열입니다. 비즈니스는 가장 관련성 높은 변형과 이미지를 맨 앞에 반환해야 합니다(SHOULD). 조회에서는 기본값을, 검색에서는 쿼리와 컨텍스트에 가장 잘 맞는 항목을 앞에 둡니다. 플랫폼은 첫 번째 요소를 대표 항목으로 취급해야 합니다(SHOULD).

| 이름             | 타입                                                                                             | 필수   | 설명                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------ | ------ | ------------------------------------------------------------------------------------------------------ |
| id               | string                                                                                           | **예** | 이 상품을 고유하게 식별하는 전역 ID(GID).                                                              |
| handle           | string                                                                                           | 아니요 | SEO 친화적인 URL을 위한 URL 안전 슬러그(예: 'blue-runner-pro'). 안정적인 API 참조에는 id를 사용하세요. |
| title            | string                                                                                           | **예** | 상품 제목.                                                                                             |
| description      | [Description](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#description)                | **예** | 하나 이상의 형식으로 작성된 상품 설명.                                                                 |
| url              | string                                                                                           | 아니요 | 표준 상품 페이지 URL.                                                                                  |
| categories       | Array\[[Category](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#category)\]             | 아니요 | 선택적 분류 체계 식별자가 포함된 상품 카테고리.                                                        |
| price_range      | [Price Range](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price-range)                | **예** | 모든 변형에 걸친 가격 범위.                                                                            |
| list_price_range | [Price Range](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price-range)                | 아니요 | 할인 전 정가 범위(취소선 표시용).                                                                      |
| media            | Array\[[Media](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#media)\]                   | 아니요 | 상품 미디어(이미지, 동영상, 3D 모델). 첫 번째 항목은 목록에 표시되는 대표 미디어입니다.                |
| options          | Array\[[Product Option](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#product-option)\] | 아니요 | 상품 옵션(Size, Color 등).                                                                             |
| variants         | Array\[[Variant](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#variant)\]               | **예** | 이 상품의 구매 가능한 변형입니다. 첫 번째 항목은 목록에 표시되는 대표 변형입니다.                      |
| rating           | [Rating](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#rating)                          | 아니요 | 상품 종합 평점.                                                                                        |
| tags             | Array[string]                                                                                    | 아니요 | 분류와 검색에 쓰는 상품 태그.                                                                          |
| metadata         | object                                                                                           | 아니요 | 표준 상품 모델을 확장하는 비즈니스 정의 사용자 데이터.                                                 |

### 변형

특정 옵션 선택, 가격, 재고 상태가 정해진 구매 가능한 항목입니다.

조회 응답의 각 변형에는 대응 관계를 파악할 수 있도록 `inputs` 배열이 있습니다. 어떤 요청 식별자가 이 변형으로 해석되었는지, 일치 방식이 `exact`였는지 `featured`(서버가 선택)였는지가 담깁니다. 자세한 내용은 [클라이언트 대응](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/catalog/lookup/#client-correlation)을 참고하세요.

`media`는 순서가 있는 배열입니다. 비즈니스는 변형의 대표 이미지를 첫 번째 요소로 반환해야 합니다(SHOULD). 플랫폼은 첫 번째 요소를 대표 항목으로 취급해야 합니다(SHOULD).

| 이름         | 타입                                                                                               | 필수   | 설명                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------- | ------ | --------------------------------------------------------------------------------------- |
| id           | string                                                                                             | **예** | 이 변형을 고유하게 식별하는 전역 ID(GID). 체크아웃에서 item.id로 사용합니다.            |
| sku          | string                                                                                             | 아니요 | 재고와 풀필먼트를 위해 비즈니스가 할당한 식별자.                                        |
| barcodes     | Array[object]                                                                                      | 아니요 | 상호 참조와 연계를 위한 업계 표준 상품 식별자.                                          |
| handle       | string                                                                                             | 아니요 | URL 안전 변형 핸들/슬러그.                                                              |
| title        | string                                                                                             | **예** | 변형 표시 제목(예: 'Blue / Large').                                                     |
| description  | [Description](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#description)                  | **예** | 하나 이상의 형식으로 된 변형 설명.                                                      |
| url          | string                                                                                             | 아니요 | 표준 변형(variant) 페이지 URL.                                                          |
| categories   | Array\[[Category](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#category)\]               | 아니요 | 선택적 분류 체계 식별자가 포함된 변형 카테고리.                                         |
| price        | [Price](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price)                              | **예** | 현재 판매 가격.                                                                         |
| list_price   | [Price](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price)                              | 아니요 | 할인 전 정가(취소선 표시용).                                                            |
| unit_price   | object                                                                                             | 아니요 | 표준 측정 단위당 가격입니다. 단위 가격이 적용되지 않으면 생략할 수 있습니다(MAY).       |
| availability | object                                                                                             | 아니요 | 변형의 구매 가능 여부.                                                                  |
| options      | Array\[[Selected Option](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#selected-option)\] | 아니요 | 이 변형을 정의하는 옵션 값(예: Color: Blue, Size: Large).                               |
| media        | Array\[[Media](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#media)\]                     | 아니요 | 변형 미디어(이미지, 동영상, 3D 모델). 첫 번째 항목이 목록에 노출되는 대표 미디어입니다. |
| rating       | [Rating](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#rating)                            | 아니요 | 변형 평점.                                                                              |
| tags         | Array[string]                                                                                      | 아니요 | 분류와 검색을 위한 변형 태그.                                                           |
| metadata     | object                                                                                             | 아니요 | 표준 변형 모델을 확장하는 비즈니스 정의 사용자 데이터.                                  |
| seller       | object                                                                                             | 아니요 | 이 변형의 판매자 맥락(선택 사항).                                                       |

### 가격

| 이름     | 타입                                                                    | 필수   | 설명                                                        |
| -------- | ----------------------------------------------------------------------- | ------ | ----------------------------------------------------------- |
| amount   | [Amount](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#amount) | **예** | ISO 4217 최소 통화 단위의 금액. 무료 항목은 0을 사용합니다. |
| currency | string                                                                  | **예** | ISO 4217 통화 코드(예: 'USD', 'EUR', 'GBP').                |

### 가격 범위

| 이름 | 타입                                                                  | 필수   | 설명              |
| ---- | --------------------------------------------------------------------- | ------ | ----------------- |
| min  | [Price](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price) | **예** | 범위의 최저 가격. |
| max  | [Price](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#price) | **예** | 범위의 최고 가격. |

### 미디어

| 이름     | 타입    | 필수   | 설명                                                     |
| -------- | ------- | ------ | -------------------------------------------------------- |
| type     | string  | **예** | 미디어 유형. 잘 알려진 값: `image`, `video`, `model_3d`. |
| url      | string  | **예** | 미디어 리소스의 URL.                                     |
| alt_text | string  | 아니요 | 미디어를 설명하는 접근성 텍스트.                         |
| width    | integer | 아니요 | 픽셀 단위 너비(이미지/동영상용).                         |
| height   | integer | 아니요 | 높이(픽셀, 이미지/동영상용).                             |

### 상품 옵션

| 이름   | 타입                                                                                         | 필수   | 설명                            |
| ------ | -------------------------------------------------------------------------------------------- | ------ | ------------------------------- |
| name   | string                                                                                       | **예** | 옵션 이름(예: 'Size', 'Color'). |
| values | Array\[[Option Value](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#option-value)\] | **예** | 이 옵션에 사용할 수 있는 값.    |

### 옵션 값

| 이름  | 타입   | 필수   | 설명                                                                                                                             |
| ----- | ------ | ------ | -------------------------------------------------------------------------------------------------------------------------------- |
| id    | string | 아니요 | 서버가 할당한 이 옵션 값의 식별자(선택 사항). selected_option에 있으면 서버는 label 대신 이 값을 매칭에 사용해야 합니다(SHOULD). |
| label | string | **예** | 이 옵션 값의 표시 텍스트(예: 'Small', 'Blue').                                                                                   |

### 선택된 옵션

| 이름  | 타입   | 필수   | 설명                                                                                                                                                |
| ----- | ------ | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| name  | string | **예** | 옵션 이름(예: 'Size').                                                                                                                              |
| id    | string | 아니요 | option_value.id에서 가져온 옵션 값 식별자(선택 사항). 있으면 서버는 이를 매칭에 사용해야 하며(SHOULD), name과 label은 표시용으로 여전히 필요합니다. |
| label | string | **예** | 선택한 옵션 레이블(예: 'Large').                                                                                                                    |

### 평점

| 이름      | 타입    | 필수   | 설명                                     |
| --------- | ------- | ------ | ---------------------------------------- |
| value     | number  | **예** | 평균 평점.                               |
| scale_min | number  | 아니요 | 평점 척도의 최솟값(예: 1~5점 척도면 1).  |
| scale_max | number  | **예** | 평점 척도의 최댓값(예: 별 5개 척도면 5). |
| count     | integer | 아니요 | 평점 산정에 반영된 리뷰 수.              |

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

| 이름         | 타입                                                                            | 필수   | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------ | ------------------------------------------------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                          | **예** | **상수 = error**. 메시지 유형 판별자.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| code         | [Error Code](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#error-code) | **예** | 오류 유형을 식별하는 오류 코드. 표준 오류는 명세에 정의되어 있고(예시 참고) 의미가 표준화되어 있으며, 자유 형식 코드도 허용합니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| path         | string                                                                          | 아니요 | 메시지가 가리키는 구성 요소의 RFC 9535 JSONPath(예: $.items[1]).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| content_type | string                                                                          | 아니요 | 콘텐츠 형식. 기본값은 plain. **허용 값:** `plain`, `markdown`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| content      | string                                                                          | **예** | 사람이 읽을 수 있는 메시지.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| severity     | string                                                                          | **예** | 리소스 상태와 권장 조치를 나타냅니다. 'recoverable': 플랫폼이 입력을 수정하고 API로 재시도해 해결할 수 있습니다. 'requires_buyer_input': 판매자의 API로는 프로그래밍 방식으로 수집할 수 없는 정보가 필요합니다(체크아웃 미완료). 'requires_buyer_review': 정책, 규제, 자격 규칙 때문에 주문 전에 구매자가 승인해야 합니다. 'unrecoverable': 조치할 유효한 리소스가 없으므로 새 리소스나 입력으로 재시도해야 합니다. 'requires\_*' 심각도의 오류는 'status: requires_escalation'의 원인이 됩니다.* *허용 값:*\* `recoverable`, `requires_buyer_input`, `requires_buyer_review`, `unrecoverable` |

#### 메시지 (경고)

| 이름         | 타입                                                                                | 필수   | 설명                                                                                                                                                                                                                                                                            |
| ------------ | ----------------------------------------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                              | **예** | **상수 = warning**. 메시지 유형 판별자.                                                                                                                                                                                                                                         |
| path         | string                                                                              | 아니요 | 관련 필드를 가리키는 JSONPath(RFC 9535)(예: $.line_items[0]).                                                                                                                                                                                                                   |
| code         | [Warning Code](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#warning-code) | **예** | 경고 유형을 식별하는 경고 코드. 표준 코드는 기능 명세에 정의되어 있고(예시 참고) 표준화된 의미를 지닙니다. 자유 형식 코드도 허용됩니다.                                                                                                                                         |
| content      | string                                                                              | **예** | 반드시 표시해야 하는(MUST) 사람이 읽을 수 있는 경고 메시지.                                                                                                                                                                                                                     |
| content_type | string                                                                              | 아니요 | 콘텐츠 형식. 기본값은 plain. **허용 값:** `plain`, `markdown`                                                                                                                                                                                                                   |
| presentation | string                                                                              | 아니요 | 이 경고의 렌더링 규약입니다. 'notice'(기본값): 플랫폼은 반드시 표시해야 하며(MUST), 닫을 수 있습니다(MAY). 'disclosure': 플랫폼은 반드시 path가 가리키는 구성 요소 가까이에 표시해야 하며(MUST), 숨기거나 자동으로 닫아서는 안 됩니다(MUST NOT). 전체 규약은 명세를 참고하세요. |
| image_url    | string                                                                              | 아니요 | 필수 시각 요소(예: 경고 기호, 에너지 등급 라벨)의 URL.                                                                                                                                                                                                                          |
| url          | string                                                                              | 아니요 | 추가 정보를 위한 참조 URL(예: 규제 기관 사이트, 등록부 항목, 정책 페이지).                                                                                                                                                                                                      |

#### 메시지 (정보)

| 이름         | 타입                                                                          | 필수   | 설명                                                                                                                                           |
| ------------ | ----------------------------------------------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                        | **예** | **상수 = info**. 메시지 유형 판별자.                                                                                                           |
| path         | string                                                                        | 아니요 | 메시지가 가리키는 구성 요소의 RFC 9535 JSONPath.                                                                                               |
| code         | [Info Code](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#info-code) | 아니요 | 정보 메시지의 유형을 식별하는 정보 코드. 표준 코드는 기능 명세에 정의되어 있고(예시 참고) 표준화된 의미를 가지며, 자유 형식 코드도 허용됩니다. |
| content_type | string                                                                        | 아니요 | 콘텐츠 형식. 기본값은 plain. **허용 값:** `plain`, `markdown`                                                                                  |
| content      | string                                                                        | **예** | 사람이 읽을 수 있는 메시지.                                                                                                                    |

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
