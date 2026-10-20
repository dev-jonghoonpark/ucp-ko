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

# 스키마 레퍼런스 { #schema-reference }

이 페이지에서는 UCP에서 사용하는 모든 기능 데이터 모델과 타입의 레퍼런스를
제공합니다.

## 기능 스키마 { #capability-schemas }

{{ auto_generate_schema_reference('.', 'reference', include_extensions=False) }}

## 타입 스키마 { #type-schemas }

{{ auto_generate_schema_reference('types', 'reference', include_extensions=False) }}

### 선택된 결제 수단 {: #payment-instrument-selected-payment-instrument }

{{ extension_schema_fields('types/payment_instrument.json#/$defs/selected_payment_instrument', 'reference') }}

### 페이지네이션 요청 {: #pagination-request }

{{ extension_schema_fields('types/pagination.json#/$defs/request', 'reference') }}

### 페이지네이션 응답 {: #pagination-response }

{{ extension_schema_fields('types/pagination.json#/$defs/response', 'reference') }}

### 에러 코드 {: #error-code }

{{ schema_fields('types/error_code', 'reference') }}

### 경고 코드 {: #warning-code }

{{ schema_fields('types/warning_code', 'reference') }}

### 정보 코드 {: #info-code }

{{ schema_fields('types/info_code', 'reference') }}

## 확장 스키마 { #extension-schemas }

{{ auto_generate_schema_reference('.', 'reference', include_capability=False) }}

## UCP 메타데이터 <span id="services"></span> <span id="ap2-checkout-response"></span> <span id="ap2-complete-request"></span> { #ucp-metadata }

다음 스키마는 디스커버리와 응답에서 사용하는 UCP 메타데이터의 구조를 정의합니다.

### 플랫폼 디스커버리 프로필 { #platform-discovery-profile }

플랫폼 프로필 문서(플랫폼이 알린 URI에 호스팅됨)의 최상위 구조입니다.

{{ extension_schema_fields('ucp.json#/$defs/platform_schema', 'reference') }}

### 비즈니스 디스커버리 프로필 { #business-discovery-profile }

비즈니스 디스커버리 문서(`/.well-known/ucp`)의 최상위 구조입니다.

{{ extension_schema_fields('ucp.json#/$defs/business_schema', 'reference') }}

### 체크아웃 응답 메타데이터 {: #ucp-response-checkout-schema }

체크아웃 응답에 포함되는 `ucp` 객체입니다.

{{ extension_schema_fields('ucp.json#/$defs/response_checkout_schema', 'reference') }}

### 장바구니 응답 메타데이터 {: #ucp-response-cart-schema }

장바구니 응답에 포함되는 `ucp` 객체입니다.

{{ extension_schema_fields('ucp.json#/$defs/response_cart_schema', 'reference') }}

### 주문 응답 메타데이터 {: #ucp-response-order-schema }

주문 응답이나 이벤트에 포함되는 `ucp` 객체입니다.

{{ extension_schema_fields('ucp.json#/$defs/response_order_schema', 'reference') }}

### 기능(Capability) { #capability }

이 객체는 기능 또는 확장 하나를 설명합니다. 디스커버리 프로필과 응답의
`capabilities` 배열에 나타나며, 맥락마다 필수 필드가 조금씩 다릅니다.

#### 기능(디스커버리) {: #discovery }

디스커버리 프로필에 나타나는 형태입니다.

{{ extension_schema_fields('capability.json#/$defs/platform_schema', 'reference') }}

#### 기능(응답) {: #response }

응답 메시지에 나타나는 형태입니다.

{{ extension_schema_fields('capability.json#/$defs/response_schema', 'reference') }}
