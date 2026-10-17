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

# Ask - MCP 바인딩 { #ask-mcp-binding }

이 문서는 [Ask 기능(Capability)](index.md)의 Model Context Protocol(MCP) 바인딩을
명세한다.

## 프로토콜 기본 사항 { #protocol-fundamentals }

### 디스커버리 { #discovery }

비즈니스는 `/.well-known/ucp`의 UCP 프로필을 통해 Common 서비스와 Ask 기능에 대한
MCP 전송 방식 지원 여부를 알린다.

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

### 요청 메타데이터 { #request-metadata }

MCP를 사용하는 플랫폼은 모든 요청에 반드시
`meta["ucp-agent"].profile`이 포함된 `meta` 객체를 넣어야 한다(**MUST**). 이 필드는
버전 호환성 확인과 기능 협상에 쓰이는 플랫폼의 UCP 프로필을 식별한다.
프로토콜 메타데이터는 `ask`의 도메인 요청과 분리되어 `meta`에 남는다.

<!-- ucp:example schema=common/ask def=ask_request op=create direction=request extract=$.params.arguments.ask -->
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "ask_business",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/agent.json"
        }
      },
      "ask": {
        "query": "What is your return policy for sale items?"
      }
    }
  }
}
```

Common 서비스의 `meta`는 HTTP `Idempotency-Key` 헤더에 대응하는 MCP 필드인 선택적
`idempotency-key`도 정의한다. 플랫폼은 `conversation`을 담은 모든 요청에
`meta["idempotency-key"]`를 포함해야 하며(**SHOULD**), 그 밖의 요청에도 포함할 수
있다(**MAY**). 재시도 규약은 [대화](index.md#conversation)를 참고한다.

## 도구 { #tools }

| 도구 | 기능 | 설명 |
| :--- | :--- | :--- |
| `ask_business` | [Ask](index.md) | 다른 기능으로는 답할 수 없는 질문을 포함해, 비즈니스의 리소스·정책·서비스에 대해 비즈니스에 질문한다. |

### `ask_business` { #ask_business }

[Ask](index.md) 기능에 대응한다.

#### Ask 요청 { #ask-request }

{{ extension_schema_fields(
  'ask.json#/$defs/ask_request', 'common/ask/mcp'
) }}

#### Ask 응답 { #ask-response }

{{ extension_schema_fields(
  'ask.json#/$defs/ask_response', 'common/ask/mcp'
) }}

#### Ask 예시 { #ask-example }

구매자가 특정 상품에 대한 정책 질문을 한다. 여기서 상품은 페이지 URL로 식별한다.
`ids`는 비즈니스가 발급한 Global ID(GID)나 비즈니스가 인식하는 보조 식별자(SKU,
핸들, URL)로 질문을 비즈니스 리소스에 연결한다. 따라서 플랫폼은 이미 가지고 있는
식별자 중 무엇이든 전달할 수 있다([질문 범위 지정](index.md#scoping-a-question) 참고).

=== "요청"

    <!-- ucp:example schema=common/ask def=ask_request op=create direction=request extract=$.params.arguments.ask -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "method": "tools/call",
      "params": {
        "name": "ask_business",
        "arguments": {
          "meta": {
            "ucp-agent": {
              "profile": "https://platform.example/profiles/v2026-01/agent.json"
            }
          },
          "ask": {
            "query": "What is your return policy for these on sale?",
            "ids": ["https://business.example.com/products/winter-jacket"],
            "context": {
              "address_country": "US",
              "language": "en",
              "intent": "buying a discounted winter jacket as a gift"
            }
          }
        }
      }
    }
    ```

=== "응답"

    <!-- ucp:example schema=common/ask def=ask_response op=read direction=response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
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
      }
    }
    ```

#### 멀티턴 예시 { #multi-turn-example }

멀티턴 대화를 지원하는 비즈니스는 `conversation`을 반환하며, 플랫폼은 그 불투명한
`id`를 다음 턴에 다시 보낸다. 턴 1은 `conversation`을 생략하고(새 대화), 턴 2는 그
식별자를 다시 보내 대화를 이어간다.

턴 1, 요청 — 새 대화(`conversation` 없음):

<!-- ucp:example schema=common/ask def=ask_request op=create direction=request extract=$.params.arguments.ask -->
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "ask_business",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/agent.json"
        }
      },
      "ask": {
        "query": "Is this jacket warm enough for winter?",
        "ids": ["https://business.example.com/products/winter-jacket"]
      }
    }
  }
}
```

턴 1, 응답 — 비즈니스가 대화 식별자를 발급한다:

<!-- ucp:example schema=common/ask def=ask_response op=read direction=response extract=$.result.structuredContent -->
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
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
  }
}
```

턴 2, 요청 — 식별자를 다시 보내 대화를 이어간다:

<!-- ucp:example schema=common/ask def=ask_request op=create direction=request extract=$.params.arguments.ask -->
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "ask_business",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://platform.example/profiles/v2026-01/agent.json"
        },
        "idempotency-key": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
      },
      "ask": {
        "query": "And is it waterproof?",
        "conversation": { "id": "conv_9a3f2e7b" }
      }
    }
  }
}
```

턴 2, 응답:

<!-- ucp:example schema=common/ask def=ask_response op=read direction=response extract=$.result.structuredContent -->
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "structuredContent": {
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
  }
}
```

#### "답변 불가" 예시 { #cant-answer-example }

비즈니스가 질문에 답할 수 없을 때도 응답은 성공 결과이며, 한계를 밝히는 `answer`가
채워져 있다.

<!-- ucp:example schema=common/ask def=ask_response op=read direction=response extract=$.result.structuredContent -->
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "structuredContent": {
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
  }
}
```

## 오류 처리 { #error-handling }

UCP는 전송 오류와 비즈니스 결과를 구분하는 2계층 오류 모델을 사용한다.

### 전송 오류 { #transport-errors }

요청 처리를 막는 전송 수준 실패(인증, 요청 한도 제한, 서비스 불가)는 JSON-RPC
`error`로 반환한다. 전체 오류 코드 레지스트리와 JSON-RPC 오류 코드 매핑은
[핵심 명세](../../overview/index.md#error-codes)를 참고한다.

### 비즈니스 결과 { #business-outcomes }

모든 애플리케이션 수준 결과는 UCP 엔벌로프와 채워진 `answer`를 담은 성공 JSON-RPC
result로 반환한다. 답변별 경고나 정보성 메모는 선택적 `messages` 배열에 담긴다.
메시지의 의미는 [Ask 개요](index.md#messages-and-error-handling)를 참고한다.

#### 예시: 고지 경고가 포함된 답변 { #example-answer-with-a-disclosure-warning }

답변이 규제 대상 고지(알레르기 유발 성분, 안전, 법률)와 관련될 때는 구속력 있는
고지를 `presentation: "disclosure"` 경고로 참조한다. 답변 자체는 한계를 밝히고
구속력 있는 출처를 따르도록 안내한다.

<!-- ucp:example schema=common/ask def=ask_response op=read direction=response extract=$.result.structuredContent -->
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "structuredContent": {
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
  }
}
```

## 엔티티 { #entities }

### UCP Response Ask {: #ucp-response-ask-schema }

{{ extension_schema_fields('ucp.json#/$defs/response_ask_schema', 'common/ask/mcp') }}

### 에러 응답 {: #error-response }

{{ schema_fields('types/error_response', 'common/ask/mcp') }}

## 준수 { #conformance }

준수하는 MCP 전송 방식 구현은 반드시 다음을 지켜야 한다(**MUST**).

1. JSON-RPC 2.0을 구현한다.
2. 비즈니스의 UCP 프로필에 `dev.ucp.common.ask`가 공시되어 있으면,
   `dev.ucp.common` 서비스의 MCP `endpoint`에서 `ask_business` 도구를 제공한다.
3. [Ask 스키마](index.md)에 따라 도구 입력을 검증한다.
4. 형식이 올바르고, 권한이 있으며, 한도 내에 있는 모든 요청에 성공 JSON-RPC
   result를 반환한다. 비즈니스 결과는 `answer`와 `messages` 배열로 전달하고,
   JSON-RPC 오류는 전송 수준 문제(인증, 요청 한도 제한, 서비스 불가, 잘못된 입력)에만
   사용한다.
5. 모든 성공 응답에 채워진 `answer`를 반환한다. "답변 불가" 결과는 한계를 분명히
   밝히는 채워진 `answer`이며, 필드 생략·오류 코드·JSON-RPC 오류가 **아니다**.
