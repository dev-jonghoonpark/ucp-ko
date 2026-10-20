# Ask - REST 바인딩

이 문서는 [Ask 기능(Capability)](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/index.md)의 HTTP/REST 바인딩을 정의합니다.

## 프로토콜 기본 사항

### 디스커버리

비즈니스는 `/.well-known/ucp`의 UCP 프로필에서 Common 서비스와 Ask 기능의 REST 전송 방식 지원 여부를 알립니다.

```json
{
  "ucp": {
    "version": "draft",
    "services": {
      "dev.ucp.common": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/overview",
          "transport": "rest",
          "schema": "https://ucp.dev/draft/services/common/rest.openapi.json",
          "endpoint": "https://business.example.com/ucp"
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

## 엔드포인트

| 엔드포인트 | 메서드 | 기능                                                                                           | 설명                                                                                                   |
| ---------- | ------ | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `/ask`     | POST   | [Ask](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/index.md) | 비즈니스의 리소스, 정책, 서비스에 관해 질문합니다. 다른 기능으로는 답할 수 없는 질문도 할 수 있습니다. |

### `POST /ask`

[Ask](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/index.md) 기능에 대응합니다.

**Inputs**

| 이름         | 타입                                                                            | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| query        | string                                                                          | **Required** | Natural-language question.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| conversation | [Conversation](/ucp-ko/draft/draft/specification/common/ask/rest/#conversation) | Optional     | Provide a conversation to continue it on a follow-up; omit to start a new one.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ids          | Array[string]                                                                   | Optional     | Optional identifiers that ground the question in any Business resource. A Business SHOULD accept its own Global IDs (GIDs) and MAY accept recognized secondary identifiers such as SKU, handle, or URL. GIDs self-describe to the Business; a Platform may send them as opaque grounding references. Examples include a product, location, cart, order, or booking.                                                                                                                                                                                                                                                                                                     |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                 | Optional     | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                 | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |

**Output**

| 이름         | 타입                                                                            | 필수 여부    | 설명                                                                                                                                                                                                                                                                                |
| ------------ | ------------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | UCP Ask Response Schema                                                         | **Required** | UCP metadata for ask responses.                                                                                                                                                                                                                                                     |
| answer       | [Description](/ucp-ko/draft/draft/specification/reference/#description)         | **Required** | Answer provided by the Business. Plain text is required; Markdown and HTML are optional alternative encodings.                                                                                                                                                                      |
| conversation | [Conversation](/ucp-ko/draft/draft/specification/common/ask/rest/#conversation) | Optional     | Business-issued conversation for multi-turn continuation. When returned, the business SHOULD retain its history until `expires_at` (or per its own policy) so a follow-up with the same `id` builds on it. An unresolved `id` starts a new conversation and is noted in `messages`. |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]              | Optional     | Optional links and references related to the answer. When a link refers to an addressable UCP resource, the Business SHOULD include its GID in the link's `id`.                                                                                                                     |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]        | Optional     | Errors, warnings, or informational messages about the answer or its grounding.                                                                                                                                                                                                      |

#### 예시

구매자가 특정 상품의 정책을 질문하는 예시입니다. 여기서는 상품을 페이지 URL로 식별합니다. `ids`에는 비즈니스가 발급한 전역 ID(GID)나 비즈니스가 인식하는 보조 식별자(SKU, 핸들, URL)를 넣어 비즈니스의 어떤 리소스든 질문의 근거로 지정할 수 있습니다. 그래서 플랫폼은 이미 알고 있는 식별자를 무엇이든 전달하면 됩니다([질문 범위 지정](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/#scoping-a-question) 참고).

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

```json
{
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
```

#### 멀티턴 예시

멀티턴 대화를 지원하는 비즈니스는 `conversation`을 반환하고, 플랫폼은 그 안의 불투명한 `id`를 다음 턴에 다시 보냅니다. 1턴에서는 `conversation`을 생략하고(새 대화), 2턴에서는 식별자를 다시 보내 대화를 이어 갑니다.

1턴 요청 — 새 대화(`conversation` 없음):

```json
POST /ask HTTP/1.1
Host: business.example.com
Content-Type: application/json

{
  "query": "Is this jacket warm enough for winter?",
  "ids": ["https://business.example.com/products/winter-jacket"]
}
```

1턴 응답 — 비즈니스가 대화 식별자를 발급합니다.

```json
{
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
```

2턴 요청 — 식별자를 다시 보내 대화를 이어 갑니다.

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

```json
{
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
```

#### "답변 불가" 예시

비즈니스가 질문에 답할 수 없을 때도 응답은 HTTP 200이며, `answer`에는 답할 수 없는 이유와 한계를 밝힌 내용이 들어갑니다.

```json
{
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
```

## HTTP 헤더

HTTP 바인딩에서 정의하는 헤더는 다음과 같습니다.

**Request Headers**

| 헤더              | 필수    | 설명                                                                                                                                  |
| ----------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `Authorization`   | No      | Contains an OAuth token representing Platform or Buyer credentials.                                                                   |
| `X-API-Key`       | No      | Reusable API key allocated to the Platform by the Business.                                                                           |
| `Signature`       | No      | RFC 9421 HTTP Message Signature. Format: `sig1=:<base64-signature>:`.                                                                 |
| `Signature-Input` | No      | RFC 9421 Signature-Input header. Format: `sig1=("@method" "@path" ...);created=<timestamp>;keyid="<key-id>"`.                         |
| `Content-Digest`  | No      | Body digest per RFC 9530. Format: `sha-256=:<base64-digest>:`.                                                                        |
| `Idempotency-Key` | No      | Ensures duplicate operations don't happen during retries.                                                                             |
| `Request-Id`      | **Yes** | Unique UUID for tracing requests across components.                                                                                   |
| `User-Agent`      | No      | Identifies the user agent string making the call.                                                                                     |
| `UCP-Agent`       | **Yes** | Identifies the UCP agent making the call, containing the signer's profile URI. Format: profile="https://example.com/.well-known/ucp". |
| `Content-Type`    | No      | Representation Metadata describing the body content.                                                                                  |
| `Accept`          | No      | Content Negotiation, indicating accepted formats.                                                                                     |
| `Accept-Language` | No      | Preferred natural languages for localization.                                                                                         |
| `Accept-Encoding` | No      | Supported content-codings (compression).                                                                                              |

**Response Headers**

| 헤더              | 필수 | 설명                                          |
| ----------------- | ---- | --------------------------------------------- |
| `Signature`       | No   | RFC 9421 HTTP Message Signature for response. |
| `Signature-Input` | No   | RFC 9421 Signature-Input header for response. |
| `Content-Digest`  | No   | Body digest per RFC 9530 for response.        |

### 헤더별 요구 사항

- **UCP-Agent**: All requests **MUST** include the `UCP-Agent` header containing the platform profile URI using Dictionary Structured Field syntax ([RFC 8941](https://datatracker.ietf.org/doc/html/rfc8941)). Format: `profile="https://platform.example/profile"`.

- **Idempotency-Key**: Common 서비스는 `Idempotency-Key`를 선택적 요청 헤더로 선언합니다. 플랫폼은 `conversation`을 담은 모든 요청에 이 헤더를 포함해야 하며(**SHOULD**), 그 밖의 요청에도 포함할 수 있습니다(**MAY**)([대화](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/#conversation) 참고). 이 헤더가 있으면 비즈니스는 반드시 다음을 지켜야 합니다(**MUST**).

  1. 키를 결과와 함께 최소 24시간 동안 저장합니다.
  1. 중복 키의 요청 본문이 원래 요청과 같으면 두 번째 턴을 추가하지 않고 캐시된 결과를 반환합니다.
  1. 같은 키가 다른 본문과 함께 재사용되면 `409 Conflict`를 반환합니다. 페이로드 일치에 관한 전체 계약은 [메시지 서명 — 재전송 방지](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/signatures/#replay-protection)를 참고하세요.

## 에러 처리

UCP는 전송 계층 에러와 비즈니스 결과를 분리하는 2계층 에러 모델을 사용합니다.

### 전송 에러

요청을 처리할 수 없게 만드는 프로토콜 수준의 문제에는 HTTP 상태 코드를 사용합니다.

| 상태 | 의미                                                     |
| ---- | -------------------------------------------------------- |
| 400  | Bad Request - 잘못된 형식의 JSON 또는 필수 파라미터 누락 |
| 401  | Unauthorized - 인증 정보 누락 또는 유효하지 않은 인증    |
| 429  | Too Many Requests - 요청 빈도 제한 초과                  |
| 500  | Internal Server Error                                    |

### 비즈니스 결과

애플리케이션 수준의 결과는 모두 UCP 엔벨로프와 값이 채워진 `answer`를 담아 HTTP 200으로 반환합니다. 답변에 딸린 경고나 안내는 선택적 `messages` 배열에 담습니다. 메시지의 의미는 [Ask 개요](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/#messages-and-error-handling)를 참고하세요.

#### 예시: 고지 경고가 포함된 답변

답변이 규제 대상 고지(알레르기 유발 성분, 안전, 법률)와 관련되면, 구속력 있는 고지는 `presentation: "disclosure"`를 지정한 경고로 참조합니다. 답변 자체에서는 한계를 밝히고 구속력 있는 출처를 확인하도록 안내합니다.

```json
{
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
```

## 엔티티

### UCP 응답 Ask(엔벨로프)

UCP metadata for ask responses.

| 이름             | 타입   | 필수 여부    | 설명                                                                                                                                           |
| ---------------- | ------ | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| version          | string | **Required** | Version identifier in YYYY-MM-DD format.                                                                                                       |
| map_order        | object | Optional     | Preferred key-traversal order for sibling registry fields inside the root `ucp` envelope (`services`, `capabilities`, and `payment_handlers`). |
| status           | string | Optional     | Application-level status of the UCP operation. **Enum:** `success`, `error`                                                                    |
| services         | object | Optional     | Service registry keyed by reverse-domain name.                                                                                                 |
| capabilities     | object | Optional     | Capability registry keyed by reverse-domain name.                                                                                              |
| payment_handlers | object | Optional     | Payment handler registry keyed by reverse-domain name.                                                                                         |

### 대화

| 이름       | 타입   | 필수 여부    | 설명                                                                                |
| ---------- | ------ | ------------ | ----------------------------------------------------------------------------------- |
| id         | string | **Required** | Opaque, business-issued conversation identifier.                                    |
| expires_at | string | Optional     | RFC 3339 timestamp after which the conversation context may be discarded. Optional. |

### 에러 응답

| 이름         | 타입                                                                     | 필수 여부    | 설명                                                              |
| ------------ | ------------------------------------------------------------------------ | ------------ | ----------------------------------------------------------------- |
| ucp          | UCP Error                                                                | **Required** | UCP protocol metadata. Status MUST be 'error' for error response. |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\] | **Required** | Array of messages describing why the operation failed.            |
| continue_url | string                                                                   | Optional     | URL for buyer handoff or session recovery.                        |

## 적합성

REST 전송 방식 구현이 이 명세를 준수하려면 반드시 다음을 지켜야 합니다(**MUST**).

1. 비즈니스의 UCP 프로필에 `dev.ucp.common.ask`가 선언되어 있으면 `dev.ucp.common` 서비스의 REST `endpoint` 아래에 `POST /ask`를 노출합니다.
1. 요청 본문을 [Ask 스키마](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/index.md)에 따라 검증합니다.
1. 형식이 올바르고 인가되었으며 한도 내에 있는 요청에는 모두 HTTP 200을 반환합니다. 비즈니스 결과는 `answer`와 `messages` 배열로 전달하고, HTTP 에러 상태 코드는 전송 수준의 문제(인증, 요청 빈도 제한, 서비스 불가, 잘못된 형식의 입력)에만 사용합니다.
1. 성공 응답에는 항상 값이 채워진 `answer`를 반환합니다. "답변 불가" 결과도 한계를 분명히 밝힌 `answer`로 전달하며, 필드 생략, 에러 코드, 2xx가 아닌 상태로 표현하지 **않습니다**.
