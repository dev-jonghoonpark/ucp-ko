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

# 카탈로그 검색 기능(Capability) { #catalog-search-capability }

* **기능 이름:** `dev.ucp.shopping.catalog.search`

비즈니스의 상품 카탈로그에서 상품을 검색합니다. 자유 텍스트 쿼리, 카테고리·가격
필터링, 페이지네이션을 지원합니다.

## 작업 { #operation }

| 작업 | 설명 |
| :--- | :--- |
| **카탈로그 검색** | 주어진 입력과 필터로 상품을 검색합니다. |

### 요청 { #request }

{{ extension_schema_fields('catalog_search.json#/$defs/search_request', 'shopping/catalog') }}

### 응답 { #response }

{{ extension_schema_fields('catalog_search.json#/$defs/search_response', 'shopping/catalog') }}

## 검색 입력 { #search-inputs }

유효한 검색 요청에는 `query` 문자열, 하나 이상의 `filters`, 확장이 정의한 입력 중
적어도 하나가 반드시 들어 있어야 합니다(MUST). `query`가 없는 요청은 둘러보기(browse)
작업으로 간주하며, 이때 비즈니스는 텍스트 관련도 순위를 매기지 않고 주어진 필터에
맞는 상품을 반환합니다. 확장은 시각적 유사도나 상품 참조 같은 입력을 추가로 정의할
수 있습니다(MAY).

구현체는 들어오는 요청에 인식할 수 있는 입력이 적어도 하나 있는지 반드시 검증해야
하며(MUST), 비어 있거나 유효하지 않은 요청은 적절한 에러와 함께 거부해야
합니다(SHOULD). 어떤 입력이 있어야 하고 내용이 어때야 하는지는 구현체가 직접 정해서
적용합니다. 예를 들어 `query`를 필수로 요구하거나, 빈 `query` 문자열을 거부하거나,
카테고리 둘러보기용으로 필터만 있는 요청을 허용할 수 있습니다.

## 검색 필터 { #search-filters }

검색 결과를 좁히는 필터 조건입니다. 표준 필터는 아래에 정의되어 있으며, 판매자는
`additionalProperties`로 사용자 정의 필터를 추가로 지원할 수 있습니다(MAY).

{{ schema_fields('types/search_filters', 'shopping/catalog') }}

### 가격 필터 { #price-filter }

{{ schema_fields('types/price_filter', 'shopping/catalog') }}

## 페이지네이션 { #pagination }

목록 작업에서 쓰는 커서 기반 페이지네이션입니다. 커서는 불투명(opaque) 문자열입니다.
비즈니스는 커서를 상태 없는 키셋(keyset) 토큰으로 인코딩할 수 있습니다(**MAY**).

### 페이지 크기 { #page-size }

`limit` 파라미터는 원하는 페이지 크기를 요청할 뿐, 결과 개수를 보장하지 않습니다.
`limit`가 없으면 비즈니스는 반드시 기본 페이지 크기를 적용해야 합니다(**MUST**).
기본값은 10을 권장하지만(**RECOMMENDED**), 비즈니스가 다른 값을 정할 수도
있습니다(**MAY**).

비즈니스는 요청된 페이지 크기나 기본 페이지 크기보다 적은 결과를 반환할 수
있으며(**MAY**), 최대 페이지 크기를 적용하는 경우도 여기에 해당합니다. 플랫폼은 응답에
담긴 결과 개수가 두 값 중 어느 하나와 같다고 가정해서는 안 됩니다(**MUST NOT**).

### 페이지네이션 요청 { #pagination-request }

{{ extension_schema_fields('types/pagination.json#/$defs/request', 'shopping/catalog') }}

### 페이지네이션 응답 { #pagination-response }

{{ extension_schema_fields('types/pagination.json#/$defs/response', 'shopping/catalog') }}

## 액션 { #actions }

검색 응답은 응답 전용 `actions` 맵을 사용합니다. 필수 항목인 `products` 배열은 비어
있을 수 있습니다. 관련 상품을 하나도 넣지 않을지, 일부만 넣을지, 전부 넣을지는 액션
유형 계약에 따라 비즈니스가 결정합니다. 상위 계약과 예시는
[카탈로그 — 액션](index.md#actions)을, 공통 규칙은
[개요 — 액션](../../overview/index.md#actions)을 참고하세요.

## 전송 방식 바인딩 { #transport-bindings }

* [REST 바인딩](rest.md#post-catalogsearch): `POST /catalog/search`
* [MCP 바인딩](mcp.md#search_catalog): `search_catalog` 도구
