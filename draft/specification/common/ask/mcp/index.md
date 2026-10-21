# Ask - MCP 바인딩

이 문서는 [Ask 기능(Capability)](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/index.md)의 Model Context Protocol(MCP) 바인딩을 설명합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`의 UCP 프로필에서 Common 서비스와 Ask 기능의 MCP 전송 방식 지원 여부를 알립니다.

```json
{
  "ucp": {
    "version": "draft",
    "services": {
      "dev.ucp.common": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/overview",
          "transport": "mcp",
          "schema": "https://ucp.dev/draft/services/common/mcp.openrpc.json",
          "endpoint": "https://business.example.com/ucp/mcp"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.common.ask": [{
        "version": "draft",
        "spec": "https://ucp.dev/draft/specification/common/ask",
        "schema": "https://ucp.dev/draft/schemas/common/ask.json"
      }]
    },
    "payment_handlers": {}
  }
}
```

### 요청 메타데이터

MCP를 사용하는 플랫폼은 모든 요청에 반드시 `meta["ucp-agent"].profile`이 포함된 `meta` 객체를 넣어야 합니다(**MUST**). 이 필드는 버전 호환성 확인과 기능 협상에 쓰는 플랫폼의 UCP 프로필을 식별합니다. 프로토콜 메타데이터는 `ask`의 도메인 요청과 분리해 `meta`에 둡니다.

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

Common 서비스의 `meta`는 HTTP `Idempotency-Key` 헤더에 대응하는 MCP 필드인 선택적 `idempotency-key`도 정의합니다. 플랫폼은 `conversation`을 담은 모든 요청에 `meta["idempotency-key"]`를 포함해야 하며(**SHOULD**), 그 밖의 요청에도 포함할 수 있습니다(**MAY**). 재시도 규약은 [대화](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/#conversation)를 참고하세요.

## 도구

| 도구           | 기능                                                                                           | 설명                                                                                                    |
| -------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `ask_business` | [Ask](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/index.md) | 다른 기능으로는 답할 수 없는 질문을 포함해, 비즈니스의 리소스·정책·서비스에 대해 비즈니스에 질문합니다. |

### `ask_business`

[Ask](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/index.md) 기능에 대응합니다.

#### Ask 요청

| 이름         | 타입                                                            | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ------------ | --------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| query        | string                                                          | **필수**  | 자연어 질문.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| conversation | object                                                          | 선택      | 후속 요청에서 대화를 이어가려면 conversation을 제공하고, 새 대화를 시작하려면 생략합니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ids          | Array[string]                                                   | 선택      | 질문의 근거가 되는 비즈니스 리소스의 식별자(선택 사항). 비즈니스는 자체 전역 ID(GID)를 받아들여야 하며(SHOULD), SKU, 핸들, URL 같은 인식 가능한 보조 식별자도 받을 수 있습니다(MAY). GID는 비즈니스가 스스로 해석할 수 있으므로 플랫폼은 이를 불투명한 근거 참조로 보낼 수 있습니다. 예로는 상품, 장소, 장바구니, 주문, 예약이 있습니다.                                                                                                                                                                                                                                                                              |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context) | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals) | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |

#### Ask 응답

| 이름         | 타입                                                                     | 필수 여부 | 설명                                                                                                                                                                                                                                                                                              |
| ------------ | ------------------------------------------------------------------------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | UCP Ask Response Schema                                                  | **필수**  | ask 응답용 UCP 메타데이터.                                                                                                                                                                                                                                                                        |
| answer       | object                                                                   | **필수**  | 비즈니스가 제공한 답변. 일반 텍스트는 필수이며, Markdown과 HTML은 선택적인 대체 인코딩입니다.                                                                                                                                                                                                     |
| conversation | object                                                                   | 선택      | 여러 차례 이어지는 대화를 위해 비즈니스가 발급한 대화. 반환한 경우 비즈니스는 같은 `id`로 이어지는 후속 요청이 기존 대화를 이어받을 수 있도록 `expires_at`까지(또는 자체 정책에 따라) 기록을 보관해야 합니다(SHOULD). 해석되지 않는 `id`는 새 대화를 시작하며, 이 사실은 `messages`에 기록됩니다. |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]       | 선택      | 답변과 관련된 링크와 참조(선택 사항). 링크가 주소 지정 가능한 UCP 리소스를 가리키면 비즈니스는 링크의 `id`에 해당 GID를 포함해야 합니다(SHOULD).                                                                                                                                                  |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\] | 선택      | 답변 또는 그 근거에 관한 오류, 경고, 정보 메시지.                                                                                                                                                                                                                                                 |

#### Ask 예시

구매자가 특정 상품의 정책을 묻는 예시입니다. 여기서는 상품을 페이지 URL로 식별합니다. `ids`는 비즈니스가 발급한 Global ID(GID)나 비즈니스가 인식하는 보조 식별자(SKU, 핸들, URL)로 질문을 비즈니스 리소스에 연결합니다. 따라서 플랫폼은 이미 알고 있는 식별자를 무엇이든 전달할 수 있습니다([질문 범위 지정](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/#scoping-a-question) 참고).

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

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "draft",
        "capabilities": {
          "dev.ucp.common.ask": [
            {"version": "draft"}
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

#### 멀티턴 예시

멀티턴 대화를 지원하는 비즈니스는 `conversation`을 반환하고, 플랫폼은 그 불투명한 `id`를 다음 턴에 다시 보냅니다. 턴 1은 `conversation`을 생략하고(새 대화), 턴 2는 그 식별자를 다시 보내 대화를 이어갑니다.

턴 1, 요청 — 새 대화(`conversation` 없음):

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

턴 1, 응답 — 비즈니스가 대화 식별자를 발급합니다:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "draft",
        "capabilities": {
          "dev.ucp.common.ask": [
            {"version": "draft"}
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

턴 2, 요청 — 식별자를 다시 보내 대화를 이어갑니다:

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

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "draft",
        "capabilities": {
          "dev.ucp.common.ask": [
            {"version": "draft"}
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

#### "답변 불가" 예시

비즈니스가 질문에 답할 수 없을 때도 응답은 성공 결과이며, 한계를 밝히는 `answer`를 채워서 보냅니다.

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "draft",
        "capabilities": {
          "dev.ucp.common.ask": [
            {"version": "draft"}
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

## 오류 처리

UCP는 전송 오류와 비즈니스 결과를 구분하는 2계층 오류 모델을 사용합니다.

### 전송 오류

요청 처리를 막는 전송 수준 실패(인증, 요청 한도 제한, 서비스 불가)는 JSON-RPC `error`로 반환합니다. 전체 오류 코드 레지스트리와 JSON-RPC 오류 코드 매핑은 [핵심 명세](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#error-codes)를 참고하세요.

### 비즈니스 결과

모든 애플리케이션 수준 결과는 UCP 엔벌로프와 채워진 `answer`를 담은 성공 JSON-RPC result로 반환합니다. 답변별 경고나 정보성 메모는 선택적 `messages` 배열에 담습니다. 메시지의 의미는 [Ask 개요](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/#messages-and-error-handling)를 참고하세요.

#### 예시: 고지 경고가 포함된 답변

답변이 규제 대상 고지(알레르기 유발 성분, 안전, 법률)와 관련될 때는 구속력 있는 고지를 `presentation: "disclosure"` 경고로 참조합니다. 답변 자체는 한계를 밝히고 구속력 있는 출처를 확인하도록 안내합니다.

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "draft",
        "capabilities": {
          "dev.ucp.common.ask": [
            {"version": "draft"}
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

## 엔티티

### UCP Response Ask

ask 응답용 UCP 메타데이터.

| 이름             | 타입   | 필수 여부 | 설명                                                                                                                    |
| ---------------- | ------ | --------- | ----------------------------------------------------------------------------------------------------------------------- |
| version          | string | **필수**  | YYYY-MM-DD 형식의 버전 식별자.                                                                                          |
| map_order        | object | 선택      | 루트 `ucp` 엔벌로프 안에 있는 형제 레지스트리 필드(`services`, `capabilities`, `payment_handlers`)의 선호 키 순회 순서. |
| status           | string | 선택      | UCP 작업의 애플리케이션 수준 상태. **허용 값:** `success`, `error`                                                      |
| services         | object | 선택      | 역도메인 이름을 키로 하는 서비스 레지스트리.                                                                            |
| capabilities     | object | 선택      | 역도메인 이름을 키로 하는 기능 레지스트리.                                                                              |
| payment_handlers | object | 선택      | 역 도메인 이름을 키로 하는 결제 핸들러 레지스트리.                                                                      |

### 에러 응답

| 이름         | 타입                                                                     | 필수 여부 | 설명                                                                               |
| ------------ | ------------------------------------------------------------------------ | --------- | ---------------------------------------------------------------------------------- |
| ucp          | UCP Error                                                                | **필수**  | UCP 프로토콜 메타데이터. 오류 응답에서는 status가 반드시 'error'여야 합니다(MUST). |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\] | **필수**  | 작업이 실패한 이유를 설명하는 메시지 배열.                                         |
| continue_url | string                                                                   | 선택      | 구매자 핸드오프 또는 세션 복구용 URL.                                              |

## 준수

준수하는 MCP 전송 방식 구현은 반드시 다음을 지켜야 합니다(**MUST**).

1. JSON-RPC 2.0을 구현합니다.
1. 비즈니스의 UCP 프로필에 `dev.ucp.common.ask`가 선언되어 있으면, `dev.ucp.common` 서비스의 MCP `endpoint`에서 `ask_business` 도구를 제공합니다.
1. [Ask 스키마](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/index.md)에 따라 도구 입력을 검증합니다.
1. 형식이 올바르고, 권한이 있으며, 한도 내에 있는 모든 요청에 성공 JSON-RPC result를 반환합니다. 비즈니스 결과는 `answer`와 `messages` 배열로 전달하고, JSON-RPC 오류는 전송 수준 문제(인증, 요청 한도 제한, 서비스 불가, 잘못된 입력)에만 사용합니다.
1. 모든 성공 응답에 채워진 `answer`를 반환합니다. "답변 불가" 결과도 한계를 분명히 밝히는 채워진 `answer`로 표현하며, 필드 생략·오류 코드·JSON-RPC 오류로 표현하지 **않습니다**.
