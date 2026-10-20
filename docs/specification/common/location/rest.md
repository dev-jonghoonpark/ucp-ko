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

# 위치(Location) - REST 바인딩 { #location-rest-binding }

이 문서는 [위치 기능(Location Capability)](index.md)의 HTTP/REST 바인딩을 정의합니다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필에서 Common 서비스와
위치 기능의 REST 전송 방식 지원 여부를 알립니다.

<!-- ucp:example schema=profile def=business_schema -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "services": {
      "dev.ucp.common": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview",
          "transport": "rest",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/common/rest.openapi.json",
          "endpoint": "https://business.example.com/ucp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.common.location.search": [{
        "version": "{{ ucp_version }}",
        "spec": "https://ucp.dev/{{ ucp_version }}/specification/common/location/search",
        "schema": "https://ucp.dev/{{ ucp_version }}/schemas/common/location_search.json"
      }],
      "dev.ucp.common.location.lookup": [{
        "version": "{{ ucp_version }}",
        "spec": "https://ucp.dev/{{ ucp_version }}/specification/common/location/lookup",
        "schema": "https://ucp.dev/{{ ucp_version }}/schemas/common/location_lookup.json"
      }]
    },
    "payment_handlers": {}
  }
}
```

## 엔드포인트 { #endpoints }

| 엔드포인트 | 메서드 | 기능 | 설명 |
| :--- | :--- | :--- | :--- |
| `/locations/search` | POST | [검색](search.md) | 물리적 위치를 검색합니다. |
| `/locations/lookup` | POST | [조회](lookup.md) | 식별자로 위치를 조회합니다. 명시적인 공간 관계와 필터로 결과를 좁힐 수 있습니다. |

### `POST /locations/search` { #post-locationssearch }

[위치 검색](search.md) 기능에 대응합니다.
전체 예시는 [전송 방식과 무관한 검색 예시](search.md#examples)를 참고하세요.

{{ method_fields('search_locations', 'common/rest.openapi.json', 'common/location/rest') }}

#### 바인딩 엔벨로프 예시 { #binding-envelope-example }

=== "요청"

    <!-- ucp:example schema=common/location_search op=search direction=request -->
    ```json
    POST /locations/search HTTP/1.1
    Host: business.example.com
    Content-Type: application/json
    Request-Id: 8ef9b0c2-78d1-4e4b-91c2-3e2ef0d3ab9f
    UCP-Agent: profile="https://platform.example/profiles/v2026-01/agent.json"

    {
      "query": "grocery store"
    }
    ```

=== "응답"

    <!-- ucp:example schema=common/location_search op=search direction=response -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json
    Content-Digest: sha-256=:yG9a8bC7...:

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.common.location.search": [
            {"version": "{{ ucp_version }}"}
          ]
        }
      },
      "locations": [
        {
          "id": "loc_valley_grocers",
          "name": "Valley Grocers"
        }
      ]
    }
    ```

### `POST /locations/lookup` { #post-locationslookup }

[위치 조회](lookup.md) 기능에 대응합니다.
전체 예시는 [전송 방식과 무관한 조회 예시](lookup.md#examples)를 참고하세요.

{{ method_fields('lookup_locations', 'common/rest.openapi.json', 'common/location/rest') }}

#### 바인딩 엔벨로프 예시 { #binding-envelope-example_1 }

=== "요청"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=request -->
    ```json
    POST /locations/lookup HTTP/1.1
    Host: business.example.com
    Content-Type: application/json
    Request-Id: 2c9b0c2a-18d1-4e4b-91c2-3e2ef0d3ab9f
    UCP-Agent: profile="https://platform.example/profiles/v2026-01/agent.json"

    {
      "ids": ["loc_downtown"]
    }
    ```

=== "응답"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=response -->
    ```json
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.common.location.lookup": [
            {"version": "{{ ucp_version }}"}
          ]
        }
      },
      "locations": [
        {
          "id": "loc_downtown",
          "inputs": [
            {"id": "loc_downtown"}
          ],
          "name": "Downtown Store"
        }
      ]
    }
    ```

## 에러 처리 { #error-handling }

UCP는 전송 계층 에러와 비즈니스 결과를 분리하는 2계층 에러 모델을 사용합니다.

### 전송 에러 { #transport-errors }

요청 처리를 가로막는 프로토콜 수준의 문제에는 HTTP 상태 코드를 사용합니다.

| 상태 | 의미 |
| :--- | :--- |
| 400 | Bad Request - 잘못된 형식의 JSON 또는 필수 파라미터 누락 |
| 401 | Unauthorized - 인증 정보 누락 또는 유효하지 않은 인증 |
| 429 | Too Many Requests - 요청 빈도 제한 초과 |
| 500 | Internal Server Error |

### 비즈니스 결과 { #business-outcomes }

모든 애플리케이션 수준 결과는 UCP 엔벨로프와 선택적 `messages` 배열을 담아 HTTP 200으로 반환합니다. 메시지의 의미는 [위치 개요](index.md#messages-and-error-handling)를 참고하세요.

## 엔티티 { #entities }

### UCP 응답 위치(엔벨로프) {: #ucp-response-location-schema }

{{ extension_schema_fields('ucp.json#/$defs/response_location_schema', 'common/location/rest') }}

### 편의 시설 유형 { #amenity-type }

{{ schema_fields('types/amenity_type', 'common/location/rest') }}

### 편의 시설 {: #amenity }

{{ extension_schema_fields('types/location.json#/$defs/amenity', 'common/location/rest') }}

### 위치 {: #location-entity }

{{ schema_fields('types/location', 'common/location/rest') }}

### 조회 위치 {: #lookup-location }

{{ extension_schema_fields('location_lookup.json#/$defs/lookup_location', 'common/location/rest') }}

### 위치 필터 {: #location-filter-schema }

{{ schema_fields('types/location_filter', 'common/location/rest') }}

### 위치 거리 {: #location-distance-schema }

{{ schema_fields('types/location_distance', 'common/location/rest') }}

### 위치 서비스 지역 {: #location-serves-schema }

{{ schema_fields('types/location_serves', 'common/location/rest') }}

### 에러 응답 {: #error-response }

{{ schema_fields('types/error_response', 'common/location/rest') }}

## 적합성 { #conformance }

적합한 REST 전송 방식 구현은 반드시 다음을 지켜야 합니다(**MUST**).

1. 비즈니스의 UCP 프로필에 선언된 각 위치 기능의 엔드포인트를 해당 기능의
    요구 사항([검색](search.md), [조회](lookup.md))에 따라 구현합니다.
    각 기능은 독립적으로 도입할 수 있습니다(**MAY**).
2. `distance`·`serves` 관계와 `filters.items` 재고 여부 조건은 플랫폼이
    명시적으로 제공한 피연산자로만 평가합니다. `context`, `signals`, IP 주소에서
    피연산자를 유추하지 않으며, 입력이 없을 때 서비스 가능 여부나 상품 재고 여부를
    암묵적으로 확인하지 않습니다.
3. `distance`, `serves`, 그리고 제공된 모든 `filters` 조건을
    논리곱(AND)으로 적용합니다.
4. 공통 페이지네이션 규약에 따라 검색에서 커서 기반 페이지네이션을 지원합니다
    ([페이지네이션](search.md#pagination) 참고).
5. 조회 요청에는 HTTP 200을 반환합니다. 알 수 없는 식별자가 있으면 반환되는 위치가
    줄어들거나 없을 수 있습니다(정보성 `not_found` 메시지를 포함할 수 있습니다(**MAY**)).
6. 조회 요청이 비즈니스의 일괄 처리 최대치를 넘으면 HTTP 200을 반환하고,
    요청 순서상 앞에서부터 최대치만큼의 고유 식별자를 처리하며,
    정보성 `batch_limit_applied` 메시지를 포함합니다.
