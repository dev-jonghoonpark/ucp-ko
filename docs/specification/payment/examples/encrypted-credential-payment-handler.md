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

# 암호화된 자격 증명 핸들러 { #encrypted-credential-handler }

* **핸들러 이름:** `com.example.encrypted_credential`
* **유형:** 결제 핸들러 예시

## 소개 { #introduction }

이 예시는 **플랫폼이 비즈니스를 위해 자격 증명을 직접 암호화하는** 결제 핸들러를
보여 준다. 토큰화 패턴과 달리 `/tokenize`나 `/detokenize` 엔드포인트가 없다.
플랫폼의 규정을 준수하는 자격 증명 볼트가 비즈니스의 공개 키로 자격 증명을
암호화하고, 비즈니스는 이를 로컬에서 복호화한다.

이 패턴은 비즈니스가 결제 시점에 토크나이저와의 왕복 지연을 피하고 싶을 때
적합하다.

**참고:** 이 예시는 카드 자격 증명(PCI DSS 준수 필요)을 사용하지만, 암호화 패턴은
**모든 자격 증명 유형**에 적용된다. 준수 요건은 자격 증명 유형에 따라 다르다.

### 주요 이점 { #key-benefits }

* **런타임 왕복 없음:** 비즈니스가 로컬에서 복호화하므로 `/detokenize` 호출이 필요 없다
* **더 단순한 아키텍처:** 토큰 저장소나 토큰-자격 증명 매핑이 없다
* **비즈니스가 관리하는 키:** 비즈니스가 자신의 복호화 키를 직접 관리한다

### 빠른 시작 { #quick-start }

| 당신이...                              | 여기서 시작                                    |
| :------------------------------------- | :-------------------------------------------- |
| 이 핸들러를 수락하는 **비즈니스**라면    | [비즈니스 통합](#business-integration) |
| 이 핸들러를 구현하는 **플랫폼**이라면 | [플랫폼 통합](#platform-integration) |

---

## 참여자 { #participants }

| 참여자  | 역할                                                                              | 사전 요건                 |
| :----------- | :-------------------------------------------------------------------------------- | :---------------------------- |
| **비즈니스** | 공개 키를 등록하고, 암호화된 자격 증명을 받아 로컬에서 복호화한다            | 예 — 플랫폼에 등록 |
| **플랫폼** | 규정을 준수하는 자격 증명 볼트를 운영하고, 비즈니스의 공개 키로 암호화한다 | 예 — 암호화 구현   |

### 패턴 흐름 { #pattern-flow }

```text
+-----------------+                              +------------+
|  Platform       |                              |  Business  |
|                 |                              |            |
+--------+--------+                              +------+-----+
         |                                              |
         |  1. Business registers public key (out-of-band)
         |<---------------------------------------------|
         |                                              |
         |  2. Confirmation                             |
         |--------------------------------------------->|
         |                                              |
         |  3. GET ucp.payment_handlers                 |
         |--------------------------------------------->|
         |                                              |
         |  4. Handler with business identity           |
         |<---------------------------------------------|
         |                                              |
         |  5. Platform's vaulting service encrypts     |
         |     credential with business's key           |
         |                                              |
         |  6. POST checkout with EncryptedCredential   |
         |--------------------------------------------->|
         |                                              |
         |       (Business decrypts locally)            |
         |                                              |
         |  7. Checkout complete                        |
         |<---------------------------------------------|
```

---

## 비즈니스 통합 { #business-integration }

### 사전 요건 { #prerequisites }

#### 중요: 카드 자격 증명에는 규정 준수가 필요하다 { #critical-compliance-required-for-card-credentials }

이 핸들러를 수락하기 전에 비즈니스는 자신의 공개 암호화 키를 플랫폼에 등록해야
한다.

비즈니스는 체크아웃 중에 암호화된 `EncryptedCredential` 페이로드만 받지만, 결제
처리를 위해 이 페이로드를 로컬에서 복호화해 원본 자격 증명을 얻는다. **카드 자격
증명의 경우** 비즈니스는 원본 PAN을 다루게 되므로 반드시 **PCI DSS를
준수해야 한다(MUST)**. 여기에는 다음이 포함된다.

* 복호화 키의 안전한 키 관리
* 복호화 후 원본 자격 증명의 안전한 처리
* 카드의 경우: PAN(Primary Account Number) 처리에 관한 모든 PCI DSS 요건 준수

**사전 요건 산출물:**

| 필드                   | 설명                                                |
| :---------------------- | :--------------------------------------------------------- |
| `identity.access_token` | 온보딩 중 플랫폼이 할당한 비즈니스 식별자 |
| 공개 키 등록   | 플랫폼이 암호화에 쓸 비즈니스의 공개 키를 저장한다       |

### 핸들러 구성 { #handler-configuration }

비즈니스는 플랫폼의 핸들러를 공지한다. `business_id` 필드는 비즈니스를 식별하며,
플랫폼은 이를 이용해 암호화에 쓸 올바른 공개 키를 조회한다.

지원되는 결제 수단 스키마는 [CardPaymentInstrument](site:schemas/common/types/card_payment_instrument.json)뿐이고, 지원되는 체크아웃 자격 증명 스키마는 `EncryptedCredential`뿐이며, 지원되는 원천 자격 증명 스키마는 [PanCredential](site:schemas/common/types/pan_credential.json)뿐이다.

**참고:** `EncryptedCredential`의 형태는 핸들러의 스키마(핸들러 선언의 `schema` 필드로 참조)에서 공식적으로 정의된다.

**참고:** `PanCredential`에는 원본 PAN이 담긴다. 카드 자격 증명의 경우 플랫폼의
볼트 서비스는 이 자격 증명을 다룰 때 **PCI DSS를 준수해야 한다**. 비즈니스는
암호화된 페이로드만 받지만, 카드 자격 증명을 로컬에서 복호화하는 순간부터 PCI DSS를
준수해야 한다. 다른 자격 증명 유형에는 각자의 준수 요건이 있다.

#### 비즈니스 구성 (디스커버리) { #business-config-discovery }

| 필드           | 타입   | 필수 | 설명                                         |
| :-------------- | :----- | :------- | :-------------------------------------------------- |
| `environment`   | string | 예      | API 환경(`sandbox` 또는 `production`)         |
| `business_id`   | string | 예      | 플랫폼이 할당한 비즈니스 식별자            |
| `public_key_id` | string | 예      | 비즈니스가 등록한 공개 키의 식별자 |

#### 비즈니스 핸들러 선언 예시 { #example-business-handler-declaration }

<!-- ucp:example schema=profile def=business_schema extract=$.ucp.payment_handlers target=$.ucp.payment_handlers -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "payment_handlers": {
      "com.example.platform_encrypted": [
        {
          "id": "platform_encrypted",
          "version": "{{ ucp_version }}",
          "spec": "https://platform.example.com/ucp/encrypted-handler.json",
          "schema": "https://platform.example.com/ucp/encrypted-handler/schema.json",
          "available_instruments": [
            {
              "type": "card",
              "constraints": {
                "properties": { "brand": { "enum": ["visa", "mastercard"] } }
              }
            }
          ],
          "config": {
            "environment": "production",
            "business_id": "merchant_abc123",
            "public_key_id": "key_2026_01"
          }
        }
      ]
    }
  }
}
```

#### 응답 구성 (체크아웃) { #response-config-checkout }

응답 구성에는 사용된 암호화에 관한 정보가 포함된다.

| 필드                  | 타입   | 필수 | 설명                           |
| :--------------------- | :----- | :------- | :------------------------------------ |
| `environment`          | string | 예      | API 환경                       |
| `business_id`          | string | 예      | 비즈니스 식별자                   |
| `encryption_algorithm` | string | 예      | 사용된 알고리즘(예: `RSA-OAEP-256`) |
| `key_id`               | string | 예      | 암호화에 사용된 키 식별자    |

#### 응답 구성 예시 { #example-response-config }

<!-- ucp:example schema=payment_handler def=response_schema -->
```json
{
  "id": "platform_encrypted",
  "version": "{{ ucp_version }}",
  "available_instruments": [
    {
      "type": "card",
      "constraints": {
        "properties": { "brand": { "enum": ["visa", "mastercard"] } }
      }
    }
  ],
  "config": {
    "environment": "production",
    "business_id": "merchant_abc123",
    "encryption_algorithm": "RSA-OAEP-256",
    "key_id": "key_2026_01"
  }
}
```

### 결제 처리 { #processing-payments }

암호화된 자격 증명이 담긴 체크아웃을 받으면 다음을 수행한다.

1. **핸들러 검증:** `instrument.handler_id`가 예상한 핸들러 ID와 일치하는지 확인한다
2. **자격 증명 복호화:** 비즈니스의 개인 키로 자격 증명을 복호화한다
3. **바인딩 검증:** 복호화된 `binding`의 `type`과 `id`를 비교해 처리 중인 리소스와
  일치하는지 확인한다
4. **결제 처리:** 복호화된 자격 증명으로 결제를 완료한다
5. **응답 반환:** 확정된 체크아웃 상태로 응답한다

---

## 플랫폼 통합 { #platform-integration }

### 사전 요건 { #prerequisites_1 }

이 핸들러는 규정을 준수하는 자격 증명 볼트를 운영하고 비즈니스를 위해 자격 증명을
암호화할 수 있는 플랫폼이 구현한다. 구현하려면 플랫폼은 다음을 해야 한다.

1. 자격 증명 저장·처리에 관한 규정 준수를 유지한다(예: 카드는 PCI DSS)
2. 온보딩 중에 비즈니스 공개 키를 저장한다
3. 핸들러 신원을 기준으로 올바른 비즈니스의 키를 골라 자격 증명을 암호화한다

**구현 요건:**

| 요건 | 설명                                                      |
| :---------- | :--------------------------------------------------------------- |
| 키 저장 | 비즈니스 신원을 해당 공개 키에 매핑한다                     |
| 암호화  | 자격 증명과 바인딩 컨텍스트를 비즈니스의 공개 키로 암호화한다 |

### 핸들러 구성 (플랫폼) { #handler-configuration-platform }

플랫폼은 `platform_config`를 사용해 자신의 UCP 프로필 `payment_handlers`
레지스트리에 이 핸들러를 공지한다.

#### 플랫폼 구성 (디스커버리) { #platform-config-discovery }

| 필드                  | 타입   | 필수 | 설명                                                |
| :--------------------- | :----- | :------- | :--------------------------------------------------------- |
| `environment`          | string | 예      | API 환경(`sandbox` 또는 `production`)                |
| `platform_id`          | string | 예      | 플랫폼 식별자                                        |
| `supported_algorithms` | array  | 예      | 지원하는 암호화 알고리즘(예: `["RSA-OAEP-256"]`) |

#### 플랫폼 핸들러 선언 예시 { #example-platform-handler-declaration }

<!-- ucp:example schema=profile def=business_schema extract=$.ucp.payment_handlers target=$.ucp.payment_handlers -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "payment_handlers": {
      "com.example.platform_encrypted": [
        {
          "id": "platform_encrypted",
          "version": "{{ ucp_version }}",
          "spec": "https://platform.example.com/ucp/encrypted-handler.json",
          "schema": "https://platform.example.com/ucp/encrypted-handler/schema.json",
          "available_instruments": [
            {
              "type": "card",
              "constraints": {
                "properties": { "brand": { "enum": ["visa", "mastercard", "amex", "discover"] } }
              }
            }
          ],
          "config": {
            "environment": "production",
            "platform_id": "platform_abc123",
            "supported_algorithms": ["RSA-OAEP-256", "RSA-OAEP-384"]
          }
        }
      ]
    }
  }
}
```

### 자격 증명 암호화 { #credential-encryption }

플랫폼 애플리케이션은 결제 흐름을 조율하지만 **원본 자격 증명에는 절대 접근하지
않는다**. 대신 다음과 같이 처리한다.

1. 플랫폼의 **규정을 준수하는 볼트 서비스**가 사용자로부터 원본 자격 증명을
   받는다
2. 볼트 서비스가 비즈니스의 공개 키로 자격 증명과 바인딩 컨텍스트를 함께
   암호화한다
3. 볼트 서비스가 암호화된 페이로드를 플랫폼 애플리케이션에 반환한다
4. 플랫폼 애플리케이션이 이 암호화된 페이로드를 체크아웃 제출에 포함한다

이렇게 분리하면 플랫폼 애플리케이션 자체는 원본 PAN을 다루거나 접근하지 않게
된다.

### 체크아웃 제출 { #submitting-checkout }

플랫폼 애플리케이션은 (자신의 볼트 서비스에서 받은) 암호화된 자격 증명과 함께
체크아웃을 제출한다.

<!-- ucp:example schema=shopping/checkout op=complete direction=request -->
```json
POST /checkout-sessions/{checkout_id}/complete
UCP-Agent: profile="https://platform.example/profile"
Content-Type: application/json

{
  "payment": {
    "instruments": [
      {
        "id": "instr_1",
        "handler_id": "platform_encrypted",
        "type": "card",
        "selected": true,
        "display": {
          "brand": "visa",
          "last_digits": "1111",
          "expiry_month": 12,
          "expiry_year": 2026
        },
        "credential": {
          "type": "encrypted",
          "encrypted_data": "base64-encoded-encrypted-payload..."
        }
      }
    ]
  },
  "signals": {
    "dev.ucp.buyer_ip": "203.0.113.42",
    "dev.ucp.user_agent": "Mozilla/5.0 ..."
  }
}
```

---

## 보안 고려 사항 { #security-considerations }

| 요건 | 설명 |
| :---------- | :---------- |
| **규정 준수 (플랫폼)** | 플랫폼 볼트 서비스는 반드시 자격 증명 유형에 해당하는 표준을 준수해야 한다(MUST)(예: 원본 PAN을 다루는 카드는 PCI DSS) |
| **규정 준수 (비즈니스)** | 비즈니스는 반드시 원본 자격 증명의 로컬 복호화·처리에 해당하는 표준을 준수해야 한다(MUST)(예: 카드는 PCI DSS) |
| **플랫폼 앱의 자격 증명 접근 금지** | 플랫폼 애플리케이션은 원본 자격 증명을 다뤄서는 안 된다(MUST NOT). 규정을 준수하는 볼트 서비스만 다룬다 |
| **비대칭 암호화** | 플랫폼의 자격 증명 볼트는 비즈니스의 공개 키로 암호화하며, 비즈니스만 복호화할 수 있다 |
| **바인딩 포함** | 재전송 공격을 막기 위해 `binding` 객체(`type`과 `id`)를 반드시 암호화된 페이로드에 포함해야 한다(MUST) |
| **키 교체** | 비즈니스는 주기적으로 키를 교체해야 하며(SHOULD), 플랫폼은 키 갱신을 지원해야 한다 |
| **자격 증명 저장 없음** | 플랫폼은 암호화된 자격 증명을 저장하지 않는다. 암호화는 단방향이다 |
| **HTTPS 필수** | 모든 체크아웃 제출은 TLS를 사용해야 한다 |

---

## 참고 자료 { #references }

* **신원 스키마:** [schemas/common/types/payment_identity.json](site:schemas/common/types/payment_identity.json)
* **결제 수단 스키마:** [schemas/common/types/card_payment_instrument.json](site:schemas/common/types/card_payment_instrument.json)
