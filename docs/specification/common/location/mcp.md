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

# Location - MCP 바인딩 { #location-mcp-binding }

이 문서는 [Location 기능(Capability)](index.md)의 Model Context Protocol(MCP) 바인딩을 설명합니다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`의 UCP 프로필에서 Common 서비스와 Location 기능의 MCP 전송 방식 지원 여부를 알립니다.

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
          "transport": "mcp",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/common/mcp.openrpc.json",
          "endpoint": "https://business.example.com/ucp/mcp"
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

### 요청 메타데이터 { #request-metadata }

MCP를 사용하는 플랫폼은 모든 요청에 반드시
`meta["ucp-agent"].profile`이 포함된 `meta` 객체를 넣어야 합니다(**MUST**). 이 필드는
버전 호환성 확인과 기능 협상에 쓰이는 플랫폼의 UCP 프로필을 식별합니다.
프로토콜 메타데이터는 `location`의 도메인 요청과 분리해 `meta`에
둡니다.

## 도구 { #tools }

| 도구 | 기능 | 설명 |
| :--- | :--- | :--- |
| `search_locations` | [Search](search.md) | 텍스트, 명시적 공간 관계, 필터로 Location을 검색합니다. |
| `lookup_locations` | [Lookup](lookup.md) | 식별자로 Location을 일괄 조회하며, 선택적으로 명시적 공간 관계와 필터로 범위를 좁힐 수 있습니다. |

### `search_locations` { #search_locations }

[Location Search](search.md) 기능에 대응합니다.
[전송 방식에 중립적인 Search 전체 예시](search.md#examples)를 참고하세요.

#### 요청 인자 { #request-arguments }

{{ extension_schema_fields('location_search.json#/$defs/search_request', 'common/location/mcp') }}

#### 응답 스키마 { #response-schema }

{{ extension_schema_fields('location_search.json#/$defs/search_response', 'common/location/mcp') }}

#### 바인딩 엔벌로프 예시 { #binding-envelope-example }

=== "요청"

    <!-- ucp:example schema=common/location_search op=search direction=request extract=$.params.arguments.location -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "method": "tools/call",
      "params": {
        "name": "search_locations",
        "arguments": {
          "meta": {
            "ucp-agent": {
              "profile": "https://platform.example/profiles/v2026-01/agent.json"
            }
          },
          "location": {
            "query": "grocery store"
          }
        }
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=common/location_search op=search direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
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
      }
    }
    ```

### `lookup_locations` { #lookup_locations }

[Location Lookup](lookup.md) 기능에 대응합니다.
[전송 방식에 중립적인 Lookup 전체 예시](lookup.md#examples)를 참고하세요.

#### 요청 인자 { #request-arguments_1 }

{{ extension_schema_fields('location_lookup.json#/$defs/lookup_request', 'common/location/mcp') }}

#### 응답 스키마 { #response-schema_1 }

{{ extension_schema_fields('location_lookup.json#/$defs/lookup_response', 'common/location/mcp') }}

#### 바인딩 엔벌로프 예시 { #binding-envelope-example_1 }

=== "요청"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=request extract=$.params.arguments.location -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 2,
      "method": "tools/call",
      "params": {
        "name": "lookup_locations",
        "arguments": {
          "meta": {
            "ucp-agent": {
              "profile": "https://platform.example/profiles/v2026-01/agent.json"
            }
          },
          "location": {
            "ids": ["loc_downtown"]
          }
        }
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=common/location_lookup op=lookup direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 2,
      "result": {
        "structuredContent": {
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
      }
    }
    ```

## 오류 처리 { #error-handling }

UCP는 전송 수준 오류와 비즈니스 결과를 구분하는 2계층 오류 모델을 사용합니다.

### 전송 오류 { #transport-errors }

요청 처리를 막는 전송 수준 실패(인증, 요청 한도 제한, 잘못된 파라미터)는 JSON-RPC `error`로 반환합니다. 자세한 내용은 [핵심 명세](../../overview/index.md#error-codes)를 참고하세요.

### 비즈니스 결과 { #business-outcomes }

모든 애플리케이션 수준 결과는 UCP 엔벌로프와 선택적 `messages` 배열을 담은 성공 JSON-RPC result로 반환합니다. 메시지의 의미는 [Location 개요](index.md#messages-and-error-handling)를 참고하세요.

## 엔티티 { #entities }

### Amenity Type { #amenity-type }

{{ schema_fields('types/amenity_type', 'common/location/mcp') }}

### Amenity {: #amenity }

{{ extension_schema_fields('types/location.json#/$defs/amenity', 'common/location/mcp') }}

### Lookup Location { #lookup-location }

{{ extension_schema_fields('location_lookup.json#/$defs/lookup_location', 'common/location/mcp') }}

### Location {: #location-entity }

{{ schema_fields('types/location', 'common/location/mcp') }}

### Location Filter {: #location-filter-schema }

{{ schema_fields('types/location_filter', 'common/location/mcp') }}

### Location Distance {: #location-distance-schema }

{{ schema_fields('types/location_distance', 'common/location/mcp') }}

### Location Serves {: #location-serves-schema }

{{ schema_fields('types/location_serves', 'common/location/mcp') }}

### Error Response {: #error-response }

{{ schema_fields('types/error_response', 'common/location/mcp') }}

## 적합성 { #conformance }

적합한 MCP 전송 방식 구현은 반드시 다음을 지켜야 합니다(**MUST**).

1. JSON-RPC 2.0 프로토콜을 올바르게 구현합니다.
2. 비즈니스의 UCP 프로필에 알린 각 Location 기능의 도구를 해당 기능의
    요구 사항([Search](search.md), [Lookup](lookup.md))에 따라 구현합니다.
    각 기능은 독립적으로 채택할 수 있습니다(**MAY**).
3. `distance`, `serves` 관계와 `filters.items` 재고 조건은 플랫폼이
    명시적으로 제공한 피연산자로만 평가합니다. `context`, `signals`, IP 주소에서
    피연산자를 유도해서는 안 되며, 입력이 없을 때 서비스 가능 여부나 상품
    재고 여부를 암묵적으로 확인해서도 안 됩니다.
4. `distance`, `serves`, 그리고 제공된 모든 `filters` 조건을
    논리곱(AND)으로 적용합니다.
5. 공통 페이지네이션 규약에 따라 Search에서 커서 기반 페이지네이션을
    지원합니다([페이지네이션](search.md#pagination) 참고).
6. Lookup 요청에는 성공 JSON-RPC result를 반환합니다. 알 수 없는 식별자가 있으면 반환되는 Location이 줄거나
    없을 뿐입니다(`messages` 배열에 정보성 `not_found` 메시지를 포함할 수 있습니다(**MAY**)).
7. Lookup 요청이 비즈니스의 배치 최대치를 초과하면 성공 JSON-RPC result를
    반환하고, 요청 순서대로 고유 식별자를 최대치만큼 앞에서부터 처리하며,
    정보성 `batch_limit_applied` 메시지를 포함합니다.
8. 도구 입력을 UCP 스키마로 검증합니다.
