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

| 이름         | 타입                                                                            | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ------------ | ------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| query        | string                                                                          | **필수**  | 자연어 질문.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| conversation | [Conversation](/ucp-ko/draft/draft/specification/common/ask/rest/#conversation) | 선택      | 후속 요청에서 대화를 이어가려면 conversation을 제공하고, 새 대화를 시작하려면 생략합니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ids          | Array[string]                                                                   | 선택      | 질문의 근거가 되는 비즈니스 리소스의 식별자(선택 사항). 비즈니스는 자체 전역 ID(GID)를 받아들여야 하며(SHOULD), SKU, 핸들, URL 같은 인식 가능한 보조 식별자도 받을 수 있습니다(MAY). GID는 비즈니스가 스스로 해석할 수 있으므로 플랫폼은 이를 불투명한 근거 참조로 보낼 수 있습니다. 예로는 상품, 장소, 장바구니, 주문, 예약이 있습니다.                                                                                                                                                                                                                                                                              |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                 | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                 | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |

**Output**

| 이름         | 타입                                                                            | 필수 여부 | 설명                                                                                                                                                                                                                                                                                              |
| ------------ | ------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | UCP Ask Response Schema                                                         | **필수**  | ask 응답용 UCP 메타데이터.                                                                                                                                                                                                                                                                        |
| answer       | [Description](/ucp-ko/draft/draft/specification/reference/#description)         | **필수**  | 비즈니스가 제공한 답변. 일반 텍스트는 필수이며, Markdown과 HTML은 선택적인 대체 인코딩입니다.                                                                                                                                                                                                     |
| conversation | [Conversation](/ucp-ko/draft/draft/specification/common/ask/rest/#conversation) | 선택      | 여러 차례 이어지는 대화를 위해 비즈니스가 발급한 대화. 반환한 경우 비즈니스는 같은 `id`로 이어지는 후속 요청이 기존 대화를 이어받을 수 있도록 `expires_at`까지(또는 자체 정책에 따라) 기록을 보관해야 합니다(SHOULD). 해석되지 않는 `id`는 새 대화를 시작하며, 이 사실은 `messages`에 기록됩니다. |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]              | 선택      | 답변과 관련된 링크와 참조(선택 사항). 링크가 주소 지정 가능한 UCP 리소스를 가리키면 비즈니스는 링크의 `id`에 해당 GID를 포함해야 합니다(SHOULD).                                                                                                                                                  |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]        | 선택      | 답변 또는 그 근거에 관한 오류, 경고, 정보 메시지.                                                                                                                                                                                                                                                 |

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

| 헤더              | 필수   | 설명                                                                                                                   |
| ----------------- | ------ | ---------------------------------------------------------------------------------------------------------------------- |
| `Authorization`   | 아니요 | 플랫폼 또는 구매자 자격 증명을 나타내는 OAuth 토큰을 담습니다.                                                         |
| `X-API-Key`       | 아니요 | 비즈니스가 플랫폼에 발급한 재사용 가능한 API 키.                                                                       |
| `Signature`       | 아니요 | RFC 9421 HTTP Message Signature입니다. 형식: `sig1=:<base64-signature>:`.                                              |
| `Signature-Input` | 아니요 | RFC 9421 Signature-Input 헤더입니다. 형식: `sig1=("@method" "@path" ...);created=<timestamp>;keyid="<key-id>"`.        |
| `Content-Digest`  | 아니요 | RFC 9530 본문 다이제스트. 형식: `sha-256=:<base64-digest>:`.                                                           |
| `Idempotency-Key` | 아니요 | 재시도 중에 같은 작업이 중복 실행되지 않도록 보장합니다.                                                               |
| `Request-Id`      | **예** | 컴포넌트 간 요청을 추적하기 위한 고유 UUID.                                                                            |
| `User-Agent`      | 아니요 | 호출하는 사용자 에이전트 문자열을 식별합니다.                                                                          |
| `UCP-Agent`       | **예** | 호출하는 UCP 에이전트를 식별하며, 서명자의 프로필 URI를 담습니다. 형식: profile="https://example.com/.well-known/ucp". |
| `Content-Type`    | 아니요 | 본문 콘텐츠를 기술하는 표현 메타데이터(Representation Metadata).                                                       |
| `Accept`          | 아니요 | 콘텐츠 협상. 허용하는 형식을 나타냅니다.                                                                               |
| `Accept-Language` | 아니요 | 현지화에 사용할 선호 자연어.                                                                                           |
| `Accept-Encoding` | 아니요 | 지원하는 콘텐츠 코딩(압축).                                                                                            |

**Response Headers**

| 헤더              | 필수   | 설명                                         |
| ----------------- | ------ | -------------------------------------------- |
| `Signature`       | 아니요 | 응답에 대한 RFC 9421 HTTP Message Signature. |
| `Signature-Input` | 아니요 | 응답에 대한 RFC 9421 Signature-Input 헤더.   |
| `Content-Digest`  | 아니요 | 응답에 대한 RFC 9530 본문 다이제스트.        |

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

ask 응답용 UCP 메타데이터.

| 이름             | 타입   | 필수 여부 | 설명                                                                                                                    |
| ---------------- | ------ | --------- | ----------------------------------------------------------------------------------------------------------------------- |
| version          | string | **필수**  | YYYY-MM-DD 형식의 버전 식별자.                                                                                          |
| map_order        | object | 선택      | 루트 `ucp` 엔벌로프 안에 있는 형제 레지스트리 필드(`services`, `capabilities`, `payment_handlers`)의 선호 키 순회 순서. |
| status           | string | 선택      | UCP 작업의 애플리케이션 수준 상태. **허용 값:** `success`, `error`                                                      |
| services         | object | 선택      | 역도메인 이름을 키로 하는 서비스 레지스트리.                                                                            |
| capabilities     | object | 선택      | 역도메인 이름을 키로 하는 기능 레지스트리.                                                                              |
| payment_handlers | object | 선택      | 역 도메인 이름을 키로 하는 결제 핸들러 레지스트리.                                                                      |

### 대화

| 이름       | 타입   | 필수 여부 | 설명                                                                                      |
| ---------- | ------ | --------- | ----------------------------------------------------------------------------------------- |
| id         | string | **필수**  | 비즈니스가 발급한 불투명 대화 식별자.                                                     |
| expires_at | string | 선택      | 이 시점 이후로 대화 컨텍스트를 폐기할 수 있는 RFC 3339 타임스탬프입니다. 선택 사항입니다. |

### 에러 응답

| 이름         | 타입                                                                     | 필수 여부 | 설명                                                                               |
| ------------ | ------------------------------------------------------------------------ | --------- | ---------------------------------------------------------------------------------- |
| ucp          | UCP Error                                                                | **필수**  | UCP 프로토콜 메타데이터. 오류 응답에서는 status가 반드시 'error'여야 합니다(MUST). |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\] | **필수**  | 작업이 실패한 이유를 설명하는 메시지 배열.                                         |
| continue_url | string                                                                   | 선택      | 구매자 핸드오프 또는 세션 복구용 URL.                                              |

## 적합성

REST 전송 방식 구현이 이 명세를 준수하려면 반드시 다음을 지켜야 합니다(**MUST**).

1. 비즈니스의 UCP 프로필에 `dev.ucp.common.ask`가 선언되어 있으면 `dev.ucp.common` 서비스의 REST `endpoint` 아래에 `POST /ask`를 노출합니다.
1. 요청 본문을 [Ask 스키마](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/index.md)에 따라 검증합니다.
1. 형식이 올바르고 인가되었으며 한도 내에 있는 요청에는 모두 HTTP 200을 반환합니다. 비즈니스 결과는 `answer`와 `messages` 배열로 전달하고, HTTP 에러 상태 코드는 전송 수준의 문제(인증, 요청 빈도 제한, 서비스 불가, 잘못된 형식의 입력)에만 사용합니다.
1. 성공 응답에는 항상 값이 채워진 `answer`를 반환합니다. "답변 불가" 결과도 한계를 분명히 밝힌 `answer`로 전달하며, 필드 생략, 에러 코드, 2xx가 아닌 상태로 표현하지 **않습니다**.
