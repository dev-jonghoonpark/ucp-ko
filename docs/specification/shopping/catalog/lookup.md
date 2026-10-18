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

# 카탈로그 조회 기능(Capability) { #catalog-lookup-capability }

* **기능 이름:** `dev.ucp.shopping.catalog.lookup`

식별자로 상품이나 변형(variant)을 조회합니다. 식별자를 이미 알고 있을 때 사용합니다
(예: 저장된 목록, 딥 링크, 장바구니 검증, 상세 화면에 표시하려고 선택한 상품).

## 작업 { #operations }

| 작업 | 도구 / 엔드포인트 | 설명 |
| :--- | :--- | :--- |
| **일괄 조회** | `lookup_catalog` / `POST /catalog/lookup` | 식별자로 여러 상품을 조회합니다. |
| **상품 조회** | `get_product` / `POST /catalog/product` | 단일 상품의 전체 상세 정보를 조회합니다. |

`lookup_catalog`는 식별자를 상품으로 해석하고, `get_product`는 알고 있는 상품 ID나
변형 ID의 전체 상세 정보를 가져옵니다.

| 구분 | `lookup_catalog` | `get_product` |
| :--- | :--- | :--- |
| **입력** | `ids[]` — 상품/변형 ID. SKU, 핸들, URL 등을 지원할 수 있음(MAY) | `id` — 상품 또는 변형 ID |
| **목적** | 식별자를 상품으로 해석 | 대화형 옵션 선택을 포함해 구매 결정에 필요한 전체 상품 상세 정보 |
| **변형** | 상품당 대표(featured) 변형 하나 | 대표 변형과 관련 하위 집합. 옵션 선택에 따라 필터링 |

해석하거나 목록에 표시할 식별자가 있다면 `lookup_catalog`를 사용합니다.
상품이 이미 정해졌고, 에이전트가 구매 결정을 내리려고 대화형 변형 선택을 포함한 전체
상세 정보가 필요하다면 `get_product`를 사용합니다.

---

## 일괄 조회 (`lookup_catalog`) { #batch-lookup-lookup_catalog }

### 지원 식별자 { #supported-identifiers }

`ids` 파라미터는 식별자 배열을 받습니다. 구현체는 반드시 상품 ID와 변형 ID로 조회를
지원해야 합니다(MUST). SKU나 핸들 같은 보조 식별자도 추가로 지원할 수 있지만(MAY),
그 식별자가 반환되는 상품 객체의 필드이기도 해야 합니다.

요청에 중복된 식별자가 있으면 반드시 중복을 제거해야 합니다(MUST). 식별자 하나가
여러 상품과 일치하면(예: 여러 변형이 공유하는 SKU) 구현체는 일치하는 상품을 반환하며,
결과 수를 제한할 수 있습니다(MAY). 여러 식별자가 같은 상품으로 해석되면 그 상품은
반드시 한 번만 반환해야 합니다(MUST).

### 클라이언트 대응 관계 { #client-correlation }

응답은 순서를 보장하지 않습니다. 각 변형에는 어떤 요청 식별자가 어떤 방식으로 그
변형으로 해석되었는지 알려 주는 `inputs` 배열이 있습니다.

{{ schema_fields('types/input_correlation', 'shopping/catalog') }}

여러 요청 식별자가 같은 변형으로 해석될 수도 있습니다(예: 상품 ID와 그 상품의 변형 ID
중 하나). 이때 변형의 `inputs` 배열에는 해석된 식별자마다 항목이 하나씩 들어가며, 항목마다
각자의 일치 유형이 있습니다. `inputs` 항목이 없는 변형은 조회 응답에 나타나서는
안 됩니다(MUST NOT).

### 일괄 처리 크기 { #batch-size }

구현체는 요청당 식별자를 10개 이상 받아야 합니다(SHOULD). 최대 일괄 처리 크기를
강제할 수 있으며(MAY), 한도를 넘는 요청은 반드시 적절한 에러(REST는 HTTP 400
`request_too_large`, MCP는 JSON-RPC `-32602`)로 거부해야 합니다(MUST).

### 해석 동작 { #resolution-behavior }

`match`는 식별자의 유형이 아니라 식별자가 어느 수준으로 해석되었는지를 나타냅니다.

* **`exact`**: 식별자가 이 변형으로 바로 해석되었습니다
  (예: 변형 ID, SKU, 바코드).
* **`featured`**: 식별자가 상위 상품으로 해석되었고, 서버가 이 변형을 대표로
  골랐습니다(예: 상품 ID, 핸들).

### 필터 { #filters }

`lookup_catalog`와 `get_product` 모두 반환할 상품과 변형을 좁히는 선택적
`filters`를 받습니다. 필터는 [검색 필터](search.md#search-filters)와 같은 스키마와 AND
의미를 따릅니다. 예를 들어 가격 필터는 지정한 범위를 벗어난 변형을 제외합니다.

필터는 식별자 해석(lookup)이나 옵션 선택(get_product) *이후에* 적용됩니다. 식별자가
가리키는 상품의 변형이 모두 가격 필터 범위를 벗어나면 그 상품은 응답에서
제외됩니다.

### 요청 { #request }

{{ extension_schema_fields('catalog_lookup.json#/$defs/lookup_request', 'shopping/catalog') }}

### 응답 { #response }

{{ extension_schema_fields('catalog_lookup.json#/$defs/lookup_response', 'shopping/catalog') }}

---

## 상품 조회 (`get_product`) { #get-product-get_product }

식별자 하나로 현재 상품 상태를 조회하며, 대화형 변형 선택과 실시간 재고 신호를
지원합니다. 구매 결정의 권위 있는 출처입니다.

### 지원 식별자 { #supported-identifiers_1 }

`id` 파라미터는 상품 ID나 변형 ID 하나를 받습니다.

### 해석 동작 { #resolution-behavior_1 }

응답은 전체 맥락(제목, 설명, 미디어, 옵션)을 담은 상품과 **[`product.selected`](#option-selection)와
일치하는 변형의 하위 집합**을 반환합니다.

* **상품 ID**: `variants`에는 대표 변형과, `product.selected`와 일치하는 다른 변형이
  들어가야 합니다(SHOULD). 요청에 `selected` 옵션이 있으면 하위 집합은 클라이언트의
  선택과 일치하는 변형으로 좁혀집니다.
* **변형 ID**: 요청한 변형이 반드시 첫 번째 요소(대표)여야 합니다(MUST).
  `product.selected`에는 그 변형의 옵션이 반영되고, 나머지 변형도 같은 유효 선택과
  일치합니다. 요청의 `selected` 옵션이 변형 자체의 옵션과 충돌하면 변형의 옵션이
  우선합니다. 즉 변형 ID만으로 선택 상태가 결정되며 `selected`는 무시됩니다.

### 응답 형태 { #response-shape }

이 작업은 단일 리소스를 다루므로, 응답에는 배열이 아닌 `product` 객체 하나가
들어갑니다. 식별자를 찾지 못하면 서버는 `ucp.status: "error"`와 함께
에러 상세 정보를 담은 `messages` 배열을 반환합니다. 이는 전송 에러가 아니라
애플리케이션 결과입니다. 핸들러가 실행되었고 그 결과를 UCP 엔벨로프로 보고한 것입니다.

### 옵션 선택 { #option-selection }

`selected`와 `preferences` 파라미터로 변형을 대화형으로 좁힐 수 있습니다. 사용자가
옵션(색상, 사이즈 등)을 하나씩 고르면 UI가 실시간으로 재고 여부를 갱신하는, 상품 상세
페이지의 핵심 상호작용입니다.

#### 입력 { #input }

* **`selected`**: 옵션 선택 배열(예: `[{"name": "Color", "label": "Red"}]`).
  일부만 선택한 상태도 유효하며, 클라이언트는 사용자가 지금까지 고른 값을 그대로 보냅니다.
  각 옵션 이름은 반드시 한 번 이하로만 나타나야 합니다(MUST).
* **`preferences`**: 완화 우선순위 순서로 나열한 옵션 이름(예:
  `["Color", "Size"]`). 모든 선택과 일치하는 변형이 없으면 서버는 이 목록의 **끝**에
  있는 옵션부터 제외하고, 우선순위가 높은 선택은 유지합니다. 선택 사항이며, 생략하면
  서버가 자체 완화 휴리스틱을 사용합니다.

#### 출력: 유효 선택 { #output-effective-selections }

상품에 설정 가능한 옵션이 있으면 응답에는 반드시 `product.selected`가 포함되어야
합니다(MUST). 요청에 `selected`가 있으면 완화를 거친 뒤의 유효 선택을, 없으면 대표
변형의 기본 선택을 담습니다. 상품에 설정 가능한 옵션이 없으면 `selected`는 비워 두거나
생략할 수 있습니다(MAY).

`selected`를 보낸 클라이언트는 보낸 요청과 `product.selected`를 비교해 완화 여부를
판단합니다.

* **완화 없음**: 응답의 `selected`가 요청과 같습니다. 모든 선택이 하나 이상의 변형으로
  해석되었습니다.
* **완화 발생**: 응답의 `selected`가 요청의 하위 집합입니다. 서버가 `preferences`
  우선순위에 따라 해석할 수 없는 옵션을 제외했습니다.

#### 출력: 재고 신호 { #output-availability-signals }

응답의 옵션 값에는 `product.selected`를 기준으로 한 재고 신호가 포함되어야
합니다(SHOULD).

| `available` | `exists` | 의미 | UI 처리 |
| :--- | :--- | :--- | :--- |
| `true` | `true` | 재고 있음 — 구매 가능 | 선택 가능 |
| `false` | `true` | 품절 — 변형은 있지만 구매 불가 | 비활성화 / 취소선 |
| `false` | `false` | 이 조합에 해당하는 변형 없음 | 숨김 또는 시각적으로 구분 |

이 필드는 `product.options[].values[]`의 각 옵션 값에 들어 있습니다. 값은 **유효
선택을 기준으로 한** 재고 여부이므로, 선택을 바꾸면 재고 맵도 바뀝니다.

### 요청 { #request_1 }

{{ extension_schema_fields('catalog_lookup.json#/$defs/get_product_request', 'shopping/catalog') }}

### 응답 { #response_1 }

{{ extension_schema_fields('catalog_lookup.json#/$defs/get_product_response', 'shopping/catalog') }}

---

## 액션 { #actions }

일괄 조회 응답과 성공한 상품 조회 응답에는 응답 전용 `actions` 맵이 들어갑니다. 일괄
조회는 필수 `products` 배열(비어 있을 수 있음)을, 성공한 상품 조회는 필수 `product`를
그대로 유지하며, 기존 에러 응답은 바뀌지 않습니다. 상위 계약은
[카탈로그 — 액션](index.md#actions)을, 공통 규칙은
[개요 — 액션](../../overview/index.md#actions)을 참고하세요.

## 전송 바인딩 { #transport-bindings }

* [REST 바인딩](rest.md#post-cataloglookup): `POST /catalog/lookup` (일괄)
* [REST 바인딩](rest.md#post-catalogproduct): `POST /catalog/product` (단일)
* [MCP 바인딩](mcp.md#lookup_catalog): `lookup_catalog` 도구 (일괄)
* [MCP 바인딩](mcp.md#get_product): `get_product` 도구 (단일)
