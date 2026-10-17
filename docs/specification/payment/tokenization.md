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

# 토큰화 가이드 { #tokenization-guide }

**OpenAPI:** [Tokenization API](site:handlers/tokenization/openapi.json)

## 개요 { #overview }

이 가이드는 **토큰화 결제 핸들러를 구축하는 구현자**를 위한 것이다. 모든 토큰화
핸들러가 따르는 공통 API, 보안 요구 사항, 준수 기준을 정의한다.

**참고:** 이 가이드의 예시는 카드 자격 증명을 사용하지만, 토큰화 패턴은 은행 계좌,
디지털 지갑, 로열티 계정 등 **모든 민감한 자격 증명 유형**에 적용된다. 규정 준수
요구 사항(예: 카드의 경우 PCI DSS)은 자격 증명 유형에 따라 다르다.

UCP에서 다양한 형태의 토큰화를 활용하는 여러 예시를 제공한다.

| 예시 | 사용 사례 |
| :------ | :------- |
| [프로세서 토크나이저](examples/processor-tokenizer-payment-handler.md) | 비즈니스 또는 PSP가 토큰화와 처리를 수행 |
| [플랫폼 토크나이저](examples/platform-tokenizer-payment-handler.md) | 플랫폼이 비즈니스/PSP를 위해 자격 증명을 토큰화 |
| [암호화된 자격 증명 핸들러](examples/encrypted-credential-payment-handler.md) | 플랫폼이 토큰화 대신 자격 증명을 암호화 |

---

## 핵심 개념 { #core-concepts }

### 자격 증명 흐름 { #credential-flow }

토큰화 핸들러는 자격 증명을 소스 형태와 체크아웃 형태 사이에서 변환한다.

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
|   | * pan           |                      | What comes OUT          |  |
|   | * network_token |                ----->| * token                 |  |
|   |                 |                      |                         |  |
|   +-----------------+                      +-------------------------+  |
|                                                                         |
+-------------------------------------------------------------------------+
```

토큰화 핸들러는 소스 자격 증명(예: PAN 자격 증명)을 받아 체크아웃 자격 증명(예: 토큰)을
생성한다.

네트워크 토큰은 이 왕복 과정이 항상 필요한 것은 아니다. 네트워크 토큰은 일치하는
크립토그램 없이는 사용할 수 없으므로, 핸들러는 `network_token_credential.json`을
토큰화용 소스 자격 증명으로 받을 수도 있고(**MAY**), 체크아웃 자격 증명으로 바로 받을
수도 있다(**MAY**). 핸들러 명세는 둘 중 어느 쪽을 받는지 반드시 명시해야 한다(**MUST**).

### 토큰 라이프사이클 { #token-lifecycle }

토큰은 서로 다른 단계를 거친다. 핸들러 명세에는 어떤 라이프사이클 정책을 사용하는지
문서화해야 한다.

```text
+--------------+    +--------------+    +--------------+    +--------------+
|  Generation  |--->|   Storage    |--->| Detokenize   |--->| Invalidation |
|              |    |              |    |              |    |              |
|Platform calls|    | Tokenizer    |    | Business/PSP |    | Token expires|
| /tokenize    |    | holds token  |    | calls        |    | or is used   |
|              |    | -> credential|    | /detokenize  |    |              |
+--------------+    +--------------+    +--------------+    +--------------+
```

| 정책 | 설명 | 사용 사례 |
| :----------------- | :------------------------------------------ | :---------------------------------------------- |
| **일회용(Single-use)** | 첫 디토큰화 후 무효화 | 가장 안전함. 권장 기본값 |
| **TTL 기반** | 정해진 시간(예: 15분) 후 만료 | 일시적 실패 시 재시도 허용 |
| **세션 범위** | 체크아웃 세션 동안 유효 | 여러 번의 처리 시도가 있는 복잡한 흐름 |

### 바인딩 { #binding }

모든 토큰화 요청에는 토큰을 특정 기능(Capability) 리소스에 묶는 `binding` 객체가
필요하다.

| 필드 | 필수 | 설명 |
| :----- | :------- | :---------------------------------------------------------------------------------------------------------- |
| `type` | 예 | 바인딩된 리소스를 소유한 기능. 예: `dev.ucp.shopping.checkout` |
| `id` | 예 | 해당 기능 내에서 바인딩된 리소스의 불투명 식별자. 호출자는 반드시 비어 있지 않은 문자열을 보내야 한다(**MUST**) |

[바인딩 스키마](site:schemas/common/types/binding.json)를 참고한다.

리소스 범위와 참여자 범위는 별개다. 요청은 `identity`를 `binding` 안이 아니라 그 옆에
담는다. `binding`은 토큰이 어떤 리소스를 위한 것인지, `identity`는 어떤 참여자를 위한
것인지를 나타낸다. `identity`는 호출자가 다른 참여자를 대신해 동작할 때 필요하며,
인증된 호출자가 바로 그 참여자일 때는 생략한다.
[결제 신원 스키마](site:schemas/common/types/payment_identity.json)를 참고한다.

바인딩은 리소스 참조가 아니라 재전송(replay) 방지 장치다. 모든 토크나이저에 다음 규칙이
적용된다.

1. 토크나이저는 반드시 `type`과 `id`의 정확한 일치로 바인딩을 검증해야
   한다(**MUST**). 토크나이저는 부분 일치를 허용해서는 안 되며(**MUST NOT**),
   `binding`에 인식하지 못하는 멤버가 있다는 이유로 요청을 거부해서도 안
   된다(**MUST NOT**). 토크나이저는 비교 시 인식하지 못하는 멤버를 반드시 무시해야
   한다(**MUST**). 추가 `binding` 멤버를 정의하는 확장은 해당 멤버가 비교에 참여하는지
   반드시 명시해야 하며(**MUST**), 그 확장을 구현하는 토크나이저는 반드시 확장의 정의를
   따라야 한다(**MUST**).
2. 토크나이저는 반드시 `binding.id`를 불투명한 값으로 취급해야 한다(**MUST**). 즉,
   값이 해석 가능할 것을 요구해서는 안 되며(**MUST NOT**), 식별된 리소스의 존재를
   확인할 수 없다는 이유로 요청을 거부해서도 안 된다(**MUST NOT**).
3. 토크나이저는 `binding.type`을 인식하지 못한다는 이유만으로 요청을 거부해서는 안
   된다(**MUST NOT**). 따라서 지금 하나의 기능을 지원하는 토크나이저도 구현을 바꾸지
   않고 나중에 정의되는 기능에서 계속 사용할 수 있다.
4. 모든 토큰은 정확히 하나의 참여자에게 발급된다. 토크나이저는 `/tokenize` 시점에 그
   참여자를 반드시 기록해야 한다(**MUST**). `identity`가 있으면 그 값을, 없으면 인증된
   호출자를 기록한다. `/detokenize`에서 토크나이저는 요청한 참여자를 같은 방식으로
   반드시 결정해야 하고(**MUST**), 그 참여자가 발급 시 기록된 참여자와 일치하는지
   반드시 검증해야 하며(**MUST**), 인증된 호출자가 그 참여자 본인이거나 그 참여자를
   대신해 동작할 권한이 있는지 반드시 검증해야 한다(**MUST**). 두 검사 중 하나라도
   실패하면 토크나이저는 자격 증명을 반환해서는 안 된다(**MUST NOT**). `identity`는
   자격 증명이 아니라 참여자 식별자이며, 토크나이저는 이를 인증 수단으로 받아들여서는
   안 된다(**MUST NOT**). 한 참여자가 다른 참여자를 대신해 동작할 권한을 얻는 방식은
   핸들러가 정의하며 이 명세의 범위 밖이다.

---

## OpenAPI { #openapi }

토큰화 핸들러는 두 개의 엔드포인트를 구현한다. 핸들러는 아키텍처에 따라 하나 또는 둘
다 구현할 수 있다(**MAY**). 자체 암호화 메커니즘을 정의하는 암호화 페이로드 예시처럼
하나도 구현하지 않을 수도 있다.

### POST /tokenize { #post-tokenize }

원시 자격 증명을 기능 리소스에 바인딩되고 참여자에게 발급되는 토큰으로 변환한다.

**구현 시점:** 내부적으로 토큰을 생성하는 에이전트가 아니라면 항상 구현한다.

<!-- ucp:example skip reason="tokenization API, not UCP payload" -->
```json
POST /tokenize
Content-Type: application/json

{
  "credential": {
    "type": "pan",
    "number": "4111111111111111",
    "expiry_month": 12,
    "expiry_year": 2026,
    "cvc": "123"
  },
  "binding": {
    "type": "dev.ucp.shopping.checkout",
    "id": "abc123"
  },
  "identity": {
    "access_token": "merchant_001"
  }
}
```

**응답:**

<!-- ucp:example skip reason="tokenization API, not UCP payload" -->
```json
{
  "token": "tok_abc123xyz789"
}
```

### POST /detokenize { #post-detokenize }

유효한 토큰에 대해 원래의 자격 증명을 반환한다. 바인딩이 일치해야 한다.

**구현 시점:** 디토큰화를 처리와 결합하지 않는 한 항상 구현한다(PSP 예시 참고).

<!-- ucp:example skip reason="tokenization API, not UCP payload" -->
```json
POST /detokenize
Content-Type: application/json
Authorization: Bearer {caller_access_token}

{
  "token": "tok_abc123xyz789",
  "binding": {
    "type": "dev.ucp.shopping.checkout",
    "id": "abc123"
  }
}
```

**응답:**

<!-- ucp:example skip reason="tokenization API, not UCP payload" -->
```json
{
  "type": "pan",
  "number": "4111111111111111",
  "expiry_month": 12,
  "expiry_year": 2026,
  "cvc": "123"
}
```

**참고:** 인증된 호출자가 토큰을 발급받은 참여자라면 `identity`를 생략한다. 다른
참여자를 대신해 동작할 때(예: PSP가 비즈니스를 위해 디토큰화하는 경우)는 포함한다.

전체 요청/응답 스키마는 [OpenAPI 명세](site:handlers/tokenization/openapi.json) 전문을 참고한다.

---

## 보안 요구 사항 { #security-requirements }

| 요구 사항 | 설명 |
| :--------------------------- | :----------------------------------------------------------------------------------------------------------------------------- |
| **바인딩 필수** | 재사용을 막기 위해 자격 증명은 반드시 `binding` 리소스(`type`과 `id`)에 바인딩되고 정확히 하나의 참여자에게 발급되어야 한다(**MUST**) |
| **바인딩 검증** | 토크나이저는 자격 증명을 반환하기 전에 반드시 바인딩 일치 여부를 검증해야 한다(**MUST**) |
| **암호학적 난수** | 안전한 난수 생성기를 사용한다. 토큰은 추측할 수 없어야 한다 |
| **충분한 길이** | 최소 128비트의 엔트로피 |
| **비가역성** | 토큰으로부터 자격 증명을 도출할 수 없다 |
| **범위 제한** | 토큰은 해당 토크나이저에서만 동작해야 한다 |
| **시간 제한** | 사용 사례에 맞는 TTL을 적용한다(일반적으로 5~30분) |
| **일회용 권장** | 가능하면 첫 디토큰화 후 무효화한다 |

---

## 핸들러 명세 요구 사항 { #handler-specification-requirements }

핸들러를 게시할 때 명세 문서에는 반드시 다음이 포함되어야 한다(**MUST**).

| 요구 사항 | 예시 |
| :------------------------------ | :---------------------------------------------------------------- |
| **고유한 핸들러 이름** | `com.example.tokenization_payment` (역 DNS 형식) |
| **엔드포인트 URL** | 프로덕션 및 샌드박스 기본 URL |
| **인증 요구 사항** | OAuth 2.0, API 키 등 |
| **온보딩 절차** | 참여자가 등록하고 신원을 발급받는 방법 |
| **허용되는 자격 증명** | 토큰화에 허용되는 자격 증명 유형 |
| **토큰 라이프사이클 정책** | 일회용, TTL, 또는 세션 범위 |
| **보안 책임 동의** | 원시 자격 증명을 받는 참여자는 책임을 수락해야 한다 |

### 명세 개요 예시 { #example-specification-outline }

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
| `card`     | `pan`, `network_token`       | `token`              |

**Token Lifecycle:** Single-use (invalidated after detokenization)

**Authentication:** OAuth 2.0 client credentials

**Onboarding:** Register at portal.acme.com. Businesses receive `access_token` for handler identity.
```

---

## 준수 체크리스트 { #conformance-checklist }

토크나이저 핸들러가 다음을 모두 충족하면 이 패턴을 준수한다.

- [ ] 고유한 역 DNS 형식의 `handler_name`과 함께 안정적인 URL에 핸들러 명세를 게시한다
- [ ] OpenAPI에 따라 `/tokenize`와 `/detokenize` 중 하나 또는 둘 다를 구현한다
- [ ] 인증 및 온보딩 요구 사항을 정의한다
- [ ] 소스 형태와 체크아웃 형태 사이의 자격 증명 변환을 문서화한다
- [ ] `TokenCredential` 스키마와 호환되는 토큰을 생성한다
- [ ] 토큰 라이프사이클 정책(TTL, 일회용 등)을 명시한다
- [ ] 토큰화 요청에 `type`과 `id`를 포함한 `binding`을 요구한다
- [ ] 참여자 식별에 `PaymentIdentity`를 사용한다
- [ ] 디토큰화 요청에서 `binding`이 정확히 일치하는지 검증한다
- [ ] 각 토큰을 발급받은 참여자를 기록하고, 디토큰화 시 그 참여자와 호출자의 대리 권한을 모두 검증한다
- [ ] 인식하지 못하는 바인딩 유형도 허용한다
- [ ] 원시 자격 증명을 받는 참여자에게 보안 책임 동의를 요구한다

---

## 참고 자료 { #references }

| 리소스 | URL                                                                                                             |
| :---------------------- | :-------------------------------------------------------------------------------------------------------------- |
| 토큰화 OpenAPI | [handlers/tokenization/openapi.json](site:handlers/tokenization/openapi.json)                                   |
| 신원 스키마 | [schemas/common/types/payment_identity.json](site:schemas/common/types/payment_identity.json)                   |
| 바인딩 스키마 | [schemas/common/types/binding.json](site:schemas/common/types/binding.json)                                     |
| 토큰 자격 증명 스키마 | [schemas/common/types/token_credential.json](site:schemas/common/types/token_credential.json)                   |
| 카드 결제 수단 스키마 | [schemas/common/types/card_payment_instrument.json](site:schemas/common/types/card_payment_instrument.json)     |

---

## 함께 보기 { #see-also }

- **[암호화된 자격 증명 핸들러](examples/encrypted-credential-payment-handler.md)** — 토큰화/디토큰화 왕복 대신 암호화를 사용하는 대안 패턴
- **[AP2 위임(Mandate) 확장](extensions/ap2-mandates.md)** — PSP 검증을 위해 체크아웃 합의에 대한 암호학적 증명을 추가한다
