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

# Ask - REST 바인딩 { #ask-rest-binding }

이 문서는 [Ask 기능(Capability)](index.md)의 HTTP/REST 바인딩을 정의한다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`에 있는 UCP 프로필을 통해 Common 서비스와
Ask 기능의 REST 전송 방식 지원 여부를 알린다.

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
      "dev.ucp.common.ask": [{
        "version": "{{ ucp_version }}",
        "spec": "https://ucp.dev/{{ ucp_version }}/specification/common/ask",
        "schema": "https://ucp.dev/{{ ucp_version }}/schemas/common/ask.json"
      }]
    },
    "payment_handlers": {}
  }
}
```

## 엔드포인트 { #endpoints }

| 엔드포인트 | 메서드 | 기능 | 설명 |
| :--- | :--- | :--- | :--- |
| `/ask` | POST | [Ask](index.md) | 다른 기능으로는 답할 수 없는 질문을 포함해 비즈니스의 리소스, 정책, 서비스에 대해 질문한다. |

### `POST /ask` { #post-ask }

[Ask](index.md) 기능에 대응한다.

{{ method_fields('ask_business', 'common/rest.openapi.json', 'common/ask/rest') }}

#### 예시 { #example }

구매자가 특정 상품에 대한 정책을 질문한다. 여기서는 상품을 페이지 URL로 식별한다.
`ids`는 비즈니스가 발급한 전역 ID(GID)나 비즈니스가 인식하는 보조 식별자(SKU,
핸들, URL)로 비즈니스의 어떤 리소스든 질문의 근거로 지정한다. 따라서 플랫폼은 이미
가지고 있는 식별자를 무엇이든 전달할 수 있다([질문 범위 지정](index.md#scoping-a-question) 참고).

=== "요청"

    <!-- ucp:example schema=common/ask def=ask_request op=create direction=request -->
    ```json
    POST /ask HTTP/1.1
    Host: business.example.com
    Content-Type: application/json

    {
      "query": "What is your return policy for these on sale?",
      "ids": ["https://business.example.com/products/winter-jacket"],
      "context": {
        "address_country": "US",
        "language": "en",
        "intent": "buying a discounted winter jacket as a gift"
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=common/ask def=ask_response op=read -->
    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.common.ask": [
            {"version": "{{ ucp_version }}"}
          ]
        }
      },
      "answer": {
        "plain": "Sale items can be returned within 14 days of delivery for store credit. Items returned to a different region may be subject to local return rules — see the refund policy for details.",
        "markdown": "Sale items can be returned within **14 days** of delivery for store credit. Items returned to a different region may be subject to local return rules — see the refund policy for details."
      },
      "links": [
        {
          "type": "refund_policy",
          "url": "https://business.example.com/policies/refunds",
          "title": "Refund Policy"
        }
      ]
    }
    ```

#### 멀티턴 예시 { #multi-turn-example }

멀티턴 대화를 지원하는 비즈니스는 `conversation`을 반환하며, 플랫폼은 그 안의 불투명한
`id`를 다음 턴에 다시 보낸다. 1턴에서는 `conversation`을 생략하고(새 대화), 2턴에서는
식별자를 다시 보내 대화를 이어 간다.

1턴 요청 — 새 대화(`conversation` 없음):

<!-- ucp:example schema=common/ask def=ask_request op=create direction=request -->
```json
POST /ask HTTP/1.1
Host: business.example.com
Content-Type: application/json

{
  "query": "Is this jacket warm enough for winter?",
  "ids": ["https://business.example.com/products/winter-jacket"]
}
```

1턴 응답 — 비즈니스가 대화 식별자를 발급한다:

<!-- ucp:example schema=common/ask def=ask_response op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": {
      "dev.ucp.common.ask": [
        {"version": "{{ ucp_version }}"}
      ]
    }
  },
  "answer": {
    "plain": "Yes — it's insulated for sub-zero conditions and rated for harsh winter use."
  },
  "conversation": { "id": "conv_9a3f2e7b", "expires_at": "2026-06-22T18:30:00Z" }
}
```

2턴 요청 — 식별자를 다시 보내 대화를 이어 간다:

<!-- ucp:example schema=common/ask def=ask_request op=create direction=request -->
```json
POST /ask HTTP/1.1
Host: business.example.com
Content-Type: application/json
Idempotency-Key: 7c9e6679-7425-40de-944b-e07fc1f90ae7

{
  "query": "And is it waterproof?",
  "conversation": { "id": "conv_9a3f2e7b" }
}
```

2턴 응답:

<!-- ucp:example schema=common/ask def=ask_response op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": {
      "dev.ucp.common.ask": [
        {"version": "{{ ucp_version }}"}
      ]
    }
  },
  "answer": {
    "plain": "It's water-resistant with a durable water-repellent finish — good for snow and light rain, though not fully waterproof."
  },
  "conversation": { "id": "conv_9a3f2e7b", "expires_at": "2026-06-22T18:30:00Z" }
}
```

#### "답변 불가" 예시 { #cant-answer-example }

비즈니스가 질문에 답할 수 없을 때도 응답은 HTTP 200이며, 한계를 밝히는 `answer`가
채워진다.

<!-- ucp:example schema=common/ask def=ask_response op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": {
      "dev.ucp.common.ask": [
        {"version": "{{ ucp_version }}"}
      ]
    }
  },
  "answer": {
    "plain": "I don't have information about competitor pricing. For questions about this store's products, I can help with policies, materials, fit, and availability."
  }
}
```

## HTTP 헤더 { #http-headers }

HTTP 바인딩에는 다음 헤더가 정의되어 있다.

{{ header_fields('ask_business', 'common/rest.openapi.json') }}

### 헤더별 요구 사항 { #specific-header-requirements }

{{ header_requirements('ucp_agent') }}

* **Idempotency-Key**: Common 서비스는 `Idempotency-Key`를 선택적 요청 헤더로
    선언한다. 플랫폼은 `conversation`을 담은 모든 요청에 이 헤더를 포함해야 하며(**SHOULD**),
    그 밖의 요청에도 포함할 수 있다(**MAY**)([대화](index.md#conversation) 참고). 이
    헤더가 있으면 비즈니스는 반드시 다음을 지켜야 한다(**MUST**).
    1. 키를 결과와 함께 최소 24시간 동안 저장한다.
    2. 요청 본문이 원래 요청과 일치하는 중복 키에 대해서는 두 번째 턴을 추가하지 않고
       캐시된 결과를 반환한다.
    3. 키가 다른 본문과 함께 재사용되면 `409 Conflict`를 반환한다.
    페이로드 일치에 관한 전체 계약은 [메시지 서명 — 재전송 방지](../../signatures.md#replay-protection)를
    참고한다.

## 에러 처리 { #error-handling }

UCP는 전송 계층 에러와 비즈니스 결과를 분리하는 2계층 에러 모델을 사용한다.

### 전송 에러 { #transport-errors }

요청 처리를 가로막는 프로토콜 수준의 문제에는 HTTP 상태 코드를 사용한다.

| 상태 | 의미 |
| :--- | :--- |
| 400 | Bad Request - 잘못된 형식의 JSON 또는 필수 파라미터 누락 |
| 401 | Unauthorized - 인증 정보 누락 또는 유효하지 않은 인증 |
| 429 | Too Many Requests - 요청 빈도 제한 초과 |
| 500 | Internal Server Error |

### 비즈니스 결과 { #business-outcomes }

모든 애플리케이션 수준 결과는 UCP 엔벨로프와 채워진 `answer`를 담아 HTTP 200으로
반환한다. 답변별 경고나 정보성 안내는 선택적 `messages` 배열에 담긴다. 메시지의
의미는 [Ask 개요](index.md#messages-and-error-handling)를 참고한다.

#### 예시: 고지 경고가 포함된 답변 { #example-answer-with-a-disclosure-warning }

답변이 규제 대상 고지(알레르기 유발 성분, 안전, 법률)와 관련되면, 구속력 있는 고지를
`presentation: "disclosure"`를 지정한 경고로 참조한다. 답변 자체는 한계를 밝히고
구속력 있는 출처를 따르도록 안내한다.

<!-- ucp:example schema=common/ask def=ask_response op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": {
      "dev.ucp.common.ask": [
        {"version": "{{ ucp_version }}"}
      ]
    }
  },
  "answer": {
    "plain": "This nut butter is made with almonds. For complete allergen information, please consult the on-product allergen disclosure, which is the authoritative source.",
    "markdown": "This nut butter is made with almonds. For complete allergen information, please consult the on-product allergen disclosure, which is the authoritative source."
  },
  "links": [
    {
      "type": "faq",
      "url": "https://business.example.com/faq/allergens",
      "title": "Allergen FAQ"
    }
  ],
  "messages": [
    {
      "type": "warning",
      "code": "allergens",
      "content": "**Contains: tree nuts.** Produced in a facility that also processes peanuts, milk, and soy.",
      "content_type": "markdown",
      "presentation": "disclosure"
    }
  ]
}
```

## 엔티티 { #entities }

### UCP 응답 Ask(엔벨로프) {: #ucp-response-ask-schema }

{{ extension_schema_fields('ucp.json#/$defs/response_ask_schema', 'common/ask/rest') }}

### 대화 {: #conversation }

{{ extension_schema_fields('ask.json#/$defs/conversation', 'common/ask/rest') }}

### 에러 응답 {: #error-response }

{{ schema_fields('types/error_response', 'common/ask/rest') }}

## 적합성 { #conformance }

적합한 REST 전송 방식 구현은 반드시 다음을 지켜야 한다(**MUST**).

1. 비즈니스의 UCP 프로필에 `dev.ucp.common.ask`가 광고되어 있으면
   `dev.ucp.common` 서비스의 REST `endpoint` 아래에 `POST /ask`를 노출한다.
2. 요청 본문을 [Ask 스키마](index.md)에 따라 검증한다.
3. 형식이 올바르고, 인가되었으며, 한도 내에 있는 모든 요청에 HTTP 200을 반환한다.
   비즈니스 결과는 `answer`와 `messages` 배열로 전달하고, HTTP 에러 상태 코드는 전송
   수준의 문제(인증, 요청 빈도 제한, 서비스 불가, 잘못된 형식의 입력)에만 사용한다.
4. 모든 성공 응답에 채워진 `answer`를 반환한다. "답변 불가" 결과는 한계를 분명히 밝히는
   채워진 `answer`이며, 필드 생략, 에러 코드, 2xx가 아닌 상태가 **아니다**.
