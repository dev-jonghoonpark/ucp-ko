# 토큰화 가이드

**OpenAPI:** [Tokenization API](/ucp-ko/2026-04-08/2026-04-08/handlers/tokenization/openapi.json)

## 개요

이 가이드는 **토큰화 결제 핸들러를 구축하는 구현자**를 위한 문서입니다. 모든 토큰화 핸들러가 따르는 공통 API, 보안 요구 사항, 준수 기준을 정의합니다.

**참고:** 이 가이드의 예시는 카드 자격 증명을 사용하지만, 토큰화 패턴은 은행 계좌, 디지털 지갑, 로열티 계정 등 **모든 민감한 자격 증명 유형**에 적용됩니다. 규정 준수 요구 사항(예: 카드의 경우 PCI DSS)은 자격 증명 유형마다 다릅니다.

UCP에서 여러 형태의 토큰화를 활용하는 예시는 다음과 같습니다.

| 예시                                                                                                                                                       | 사용 사례                                       |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| [프로세서 토크나이저](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/examples/processor-tokenizer-payment-handler/index.md) | 비즈니스 또는 PSP가 토큰화와 처리를 수행        |
| [플랫폼 토크나이저](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/examples/platform-tokenizer-payment-handler/index.md)    | 플랫폼이 비즈니스/PSP를 위해 자격 증명을 토큰화 |
| [암호화된 자격 증명 핸들러](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/examples/encrypted-credential-handler/index.md)  | 플랫폼이 토큰화 대신 자격 증명을 암호화         |

______________________________________________________________________

## 핵심 개념

### 자격 증명 흐름

토큰화 핸들러는 자격 증명을 소스 형태와 체크아웃 형태 사이에서 변환합니다.

```text
+-------------------------------------------------------------------------+
|                     Tokenization Payment Flow                           |
+-------------------------------------------------------------------------+
|                                                                         |
|   Platform has:            Tokenizer            Business receives:      |
|   Source Credential    -->  /tokenize  -->         TokenCredential      |
|                                                                         |
|   +-----------------+                      +-------------------------+  |
|   | source_         |                      | checkout_               |  |
|   | credentials     |    What goes IN      | credentials             |  |
|   |                 |<---------------      |                         |  |
|   | * card/fpan     |                      | What comes OUT          |  |
|   | * card/dpan     |                ----->| * token                 |  |
|   |                 |                      |                         |  |
|   +-----------------+                      +-------------------------+  |
|                                                                         |
+-------------------------------------------------------------------------+
```

토큰화 핸들러는 소스 자격 증명(예: FPAN이 담긴 카드)을 받아 체크아웃 자격 증명(예: 토큰)을 생성합니다.

### 토큰 라이프사이클

토큰은 여러 단계를 거칩니다. 핸들러 명세에는 어떤 라이프사이클 정책을 사용하는지 문서화해야 합니다.

```text
+--------------+    +--------------+    +--------------+    +--------------+
|  Generation  |--->|   Storage    |--->| Detokenize   |--->| Invalidation |
|              |    |              |    |              |    |              |
|Platform calls|    | Tokenizer    |    | Business/PSP |    | Token expires|
| /tokenize    |    | holds token  |    | calls        |    | or is used   |
|              |    | -> credential|    | /detokenize  |    |              |
+--------------+    +--------------+    +--------------+    +--------------+
```

| 정책                   | 설명                          | 사용 사례                              |
| ---------------------- | ----------------------------- | -------------------------------------- |
| **일회용(Single-use)** | 첫 디토큰화 후 무효화         | 가장 안전한 권장 기본값                |
| **TTL 기반**           | 정해진 시간(예: 15분) 후 만료 | 일시적 실패 시 재시도 허용             |
| **세션 범위**          | 체크아웃 세션 동안 유효       | 여러 번의 처리 시도가 있는 복잡한 흐름 |

### 바인딩

모든 토큰화 요청에는 토큰을 특정 컨텍스트에 묶는 `binding` 객체가 필요합니다.

| 필드          | 필수   | 설명                                                               |
| ------------- | ------ | ------------------------------------------------------------------ |
| `checkout_id` | 예     | 이 토큰이 유효한 체크아웃 세션                                     |
| `identity`    | 조건부 | 바인딩할 참여자 신원. 호출자가 다른 참여자를 대신해 동작할 때 필요 |

토크나이저는 `/detokenize`에서 반드시 바인딩 일치 여부를 검증해야 합니다(**MUST**). [바인딩 스키마](/ucp-ko/2026-04-08/2026-04-08/schemas/shopping/types/binding.json)를 참고하세요.

______________________________________________________________________

## OpenAPI

토큰화 핸들러가 구현하는 엔드포인트는 두 개입니다. 핸들러는 아키텍처에 따라 하나 또는 둘 다 구현할 수 있습니다(**MAY**). 자체 암호화 메커니즘을 정의하는 암호화 페이로드 예시처럼 하나도 구현하지 않을 수도 있습니다.

### POST /tokenize

원시 자격 증명을 체크아웃과 신원에 바인딩된 토큰으로 변환합니다.

**구현 시점:** 내부적으로 토큰을 생성하는 에이전트가 아니라면 항상 구현합니다.

```json
POST /tokenize
Content-Type: application/json

{
  "credential": {
    "type": "card",
    "card_number_type": "fpan",
    "number": "4111111111111111",
    "expiry_month": 12,
    "expiry_year": 2026,
    "cvc": "123"
  },
  "binding": {
    "checkout_id": "abc123",
    "identity": {
      "access_token": "merchant_001"
    }
  }
}
```

**응답:**

```json
{
  "token": "tok_abc123xyz789"
}
```

### POST /detokenize

유효한 토큰을 받으면 원래의 자격 증명을 반환합니다. 바인딩이 일치해야 합니다.

**구현 시점:** 디토큰화를 처리와 결합하지 않는 한 항상 구현합니다(PSP 예시 참고).

```json
POST /detokenize
Content-Type: application/json
Authorization: Bearer {caller_access_token}

{
  "token": "tok_abc123xyz789",
  "binding": {
    "checkout_id": "abc123"
  }
}
```

**응답:**

```json
{
  "type": "card",
  "card_number_type": "fpan",
  "number": "4111111111111111",
  "expiry_month": 12,
  "expiry_year": 2026,
  "cvc": "123"
}
```

**참고:** 인증된 호출자가 바인딩 대상이라면 `binding.identity`를 생략합니다. 다른 참여자를 대신해 동작할 때(예: PSP가 비즈니스를 위해 디토큰화하는 경우)는 포함합니다.

전체 요청/응답 스키마는 [OpenAPI 명세](/ucp-ko/2026-04-08/2026-04-08/handlers/tokenization/openapi.json) 전문을 참고하세요.

______________________________________________________________________

## 보안 요구 사항

| 요구 사항         | 설명                                                                                                    |
| ----------------- | ------------------------------------------------------------------------------------------------------- |
| **바인딩 필수**   | 재사용을 막기 위해 자격 증명은 반드시 `checkout_id`와 참여자 `identity`에 바인딩되어야 합니다(**MUST**) |
| **바인딩 검증**   | 토크나이저는 자격 증명을 반환하기 전에 반드시 바인딩 일치 여부를 검증해야 합니다(**MUST**)              |
| **암호학적 난수** | 안전한 난수 생성기를 사용합니다. 토큰은 추측할 수 없어야 합니다                                         |
| **충분한 길이**   | 최소 128비트의 엔트로피                                                                                 |
| **비가역성**      | 토큰에서 자격 증명을 도출할 수 없어야 합니다                                                            |
| **범위 제한**     | 토큰은 해당 토크나이저에서만 동작해야 합니다                                                            |
| **시간 제한**     | 사용 사례에 맞는 TTL을 적용합니다(일반적으로 5~30분)                                                    |
| **일회용 권장**   | 가능하면 첫 디토큰화 후 무효화합니다                                                                    |

______________________________________________________________________

## 핸들러 명세 요구 사항

핸들러를 게시할 때 명세 문서에는 반드시 다음 내용을 포함해야 합니다(**MUST**).

| 요구 사항                  | 예시                                                  |
| -------------------------- | ----------------------------------------------------- |
| **고유한 핸들러 이름**     | `com.example.tokenization_payment` (역 DNS 형식)      |
| **엔드포인트 URL**         | 프로덕션 및 샌드박스 기본 URL                         |
| **인증 요구 사항**         | OAuth 2.0, API 키 등                                  |
| **온보딩 절차**            | 참여자가 등록하고 신원을 발급받는 방법                |
| **허용되는 자격 증명**     | 토큰화에 허용되는 자격 증명 유형                      |
| **토큰 라이프사이클 정책** | 일회용, TTL, 또는 세션 범위                           |
| **보안 책임 동의**         | 원시 자격 증명을 받는 참여자는 책임을 수락해야 합니다 |

### 명세 개요 예시

```markdown
**Handler Name:** `com.acme.tokenization_payment`
**OpenAPI:** [Tokenization API](site:handlers/tokenization/openapi.json)

| Environment | Base URL                           |
| :---------- | :--------------------------------- |
| Production  | `https://api.acme.com/ucp`         |
| Sandbox     | `https://sandbox.api.acme.com/ucp` |

**Supported Instruments:**

| Instrument | Source Credentials           | Checkout Credentials |
| :--------- | :--------------------------- | :------------------- |
| `card`     | `card` (fpan, network_token) | `token`              |

**Token Lifecycle:** Single-use (invalidated after detokenization)

**Authentication:** OAuth 2.0 client credentials

**Onboarding:** Register at portal.acme.com. Businesses receive `access_token` for handler identity.
```

______________________________________________________________________

## 준수 체크리스트

토크나이저 핸들러가 다음을 모두 충족하면 이 패턴을 준수하는 것입니다.

- 고유한 역 DNS 형식의 `handler_name`과 함께 안정적인 URL에 핸들러 명세를 게시합니다
- OpenAPI에 따라 `/tokenize`와 `/detokenize` 중 하나 또는 둘 다를 구현합니다
- 인증 및 온보딩 요구 사항을 정의합니다
- 소스 형태와 체크아웃 형태 사이의 자격 증명 변환을 문서화합니다
- `TokenCredential` 스키마와 호환되는 토큰을 생성합니다
- 토큰 라이프사이클 정책(TTL, 일회용 등)을 명시합니다
- 토큰화 요청에 `checkout_id`를 포함한 `binding`을 요구합니다
- 참여자 식별에 `PaymentIdentity`를 사용합니다
- 디토큰화 요청에서 `binding` 일치 여부를 검증합니다
- 원시 자격 증명을 받는 참여자에게 보안 책임 동의를 요구합니다

______________________________________________________________________

## 참고 자료

| 리소스                | URL                                                                                                                                      |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 토큰화 OpenAPI        | [handlers/tokenization/openapi.json](/ucp-ko/2026-04-08/2026-04-08/handlers/tokenization/openapi.json)                                   |
| 신원 스키마           | [schemas/shopping/types/payment_identity.json](/ucp-ko/2026-04-08/2026-04-08/schemas/shopping/types/payment_identity.json)               |
| 바인딩 스키마         | [schemas/shopping/types/binding.json](/ucp-ko/2026-04-08/2026-04-08/schemas/shopping/types/binding.json)                                 |
| 토큰 자격 증명 스키마 | [schemas/shopping/types/token_credential.json](/ucp-ko/2026-04-08/2026-04-08/schemas/shopping/types/token_credential.json)               |
| 카드 결제 수단 스키마 | [schemas/shopping/types/card_payment_instrument.json](/ucp-ko/2026-04-08/2026-04-08/schemas/shopping/types/card_payment_instrument.json) |

______________________________________________________________________

## 함께 보기

- **[암호화된 자격 증명 핸들러](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/examples/encrypted-credential-handler/index.md)** — 토큰화/디토큰화 왕복 대신 암호화를 사용하는 대안 패턴
- **[AP2 위임(Mandate) 확장](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/ap2-mandates/index.md)** — PSP가 검증할 수 있도록 체크아웃 합의에 대한 암호학적 증명을 추가합니다
