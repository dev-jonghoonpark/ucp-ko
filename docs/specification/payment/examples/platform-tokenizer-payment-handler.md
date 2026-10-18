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

# 플랫폼 토크나이저 결제 핸들러 { #platform-tokenizer-payment-handler }

* **핸들러 이름:** `com.example.platform_tokenizer`
* **유형:** 결제 핸들러 예시

## 소개 { #introduction }

이 예시는 **플랫폼이 토크나이저 역할을 하는** 토큰화 결제 핸들러를 보여 줍니다.
플랫폼의 **결제 자격 증명 제공자**는 민감한 결제 데이터(예: 사용자 지갑에 저장된 카드)를
안전하게 보관하며, 외부 `/tokenize` 엔드포인트를 호출하지 않고 내부에서 토큰을 생성합니다.

플랫폼의 자격 증명 제공자는 `/detokenize` 엔드포인트를 노출합니다. 비즈니스는 이
엔드포인트를 호출해 결제 처리에 필요한 민감한 결제 수단 정보를 가져옵니다.

이 패턴은 규정을 준수하는 자격 증명 저장소를 갖추고 지갑 제공자로 운영되는 플랫폼에
가장 적합합니다.

**참고:** 이 예시는 카드 자격 증명을 사용하지만, 패턴 자체는 **모든 자격 증명 유형**에
적용됩니다. 규정 준수 요건은 자격 증명 유형에 따라 다릅니다(예: 카드의 경우 PCI DSS).

### 주요 이점 { #key-benefits }

* **사전 전송 없음:** 결제 요청이 최종 확정되기 전까지 플랫폼은 민감한 데이터를 절대 노출하지 않습니다.
* **플랫폼 주도 보안:** 토큰 수명 주기와 바인딩 정책을 플랫폼이 정의합니다.
* **PSP 유연성:** 비즈니스는 역토큰화를 PSP에 위임해 민감한 데이터가 비즈니스 시스템에 아예 들어오지 않게 할 수 있습니다.

### 빠른 시작 { #quickstart }

| 해당하는 역할                          | 시작 위치                                     |
| :------------------------------------- | :-------------------------------------------- |
| 이 핸들러를 수용하는 **비즈니스**      | [비즈니스 연동](#business-integration) |
| 이 핸들러를 구현하는 **플랫폼**        | [플랫폼 연동](#platform-integration) |
| 비즈니스를 대신해 처리하는 **PSP**     | [PSP 연동](#psp-integration)           |

---

## 참여자 { #participants }

| 참여자       | 역할                                                                                            | 사전 요건                             |
| :----------- | :---------------------------------------------------------------------------------------------- | :------------------------------------ |
| **Business** | 핸들러를 선언하고 토큰을 받으며, 필요하면 PSP에 위임합니다                                      | 있음 — 플랫폼에 온보딩                |
| **Platform** | 토큰을 생성하고 `/detokenize` 엔드포인트를 노출하는 결제 자격 증명 제공자를 운영합니다            | 있음 — 토큰화 서비스 구현             |
| **PSP**      | 필요에 따라 비즈니스를 대신해 역토큰화하고 결제를 처리합니다                                    | 있음 — 플랫폼에 온보딩                |

### 패턴 흐름: 비즈니스가 역토큰화 { #pattern-flow-business-detokenizes }

```text
+-----------------+                              +------------+
|    Platform     |                              |  Business  |
|  (Tokenizer)    |                              |            |
+--------+--------+                              +------+-----+
         |                                              |
         |  1. Business registers with Platform (out-of-band)
         |<---------------------------------------------|
         |                                              |
         |  2. API credentials                          |
         |--------------------------------------------->|
         |                                              |
         |  3. GET ucp.payment_handlers                 |
         |--------------------------------------------->|
         |                                              |
         |  4. Handler with business identity           |
         |<---------------------------------------------|
         |                                              |
         |5. Platforms's Credential Provider generates token
         |                                              |
         |  6. POST checkout with TokenCredential       |
         |--------------------------------------------->|
         |                                              |
         |  7. POST /detokenize (to Credential Provider)|
         |<---------------------------------------------|
         |                                              |
         |  8. Sensitive Data                           |
         |--------------------------------------------->|
         |                                              |
         |  9. Checkout complete                        |
         |<---------------------------------------------|
```

### 패턴 흐름: PSP가 역토큰화 { #pattern-flow-psp-detokenizes }

```text
+-----------------+     +------------+      +---------+
|    Platform     |     |  Business  |      |   PSP   |
|  (Tokenizer)    |     |            |      |         |
+--------+--------+     +------+-----+      +----+----+
         |                     |                 |
         |  1. Business + PSP register with Platform (out-of-band)
         |<--------------------|                 |
         |<--------------------------------------|
         |                     |                 |
         |  2. API credentials |                 |
         |-------------------->|                 |
         |-------------------------------------->|
         |                     |                 |
         |  3. Payment Credential Provider       |
         |     generates token                   |
         |                     |                 |
         |  4. POST checkout with TokenCredential|
         |-------------------->|                 |
         |                     |                 |
         |                     |  5. Forward     |
         |                     |  token to PSP   |
         |                     |---------------->|
         |                     |                 |
         |  6. POST /detokenize (to Credential Provider, with business identity)
         |<--------------------------------------|
         |                     |                 |
         |  7. Sensitive Data  |                 |
         |-------------------------------------->|
         |                     |                 |
         |                     |  8. Payment     |
         |                     |  result         |
         |                     |                  |
         |                     |<----------------|
         |                     |                 |
         |  9. Checkout complete                 |
         |<--------------------|                 |
```

---

## 비즈니스 연동 { #business-integration }

### 사전 요건 { #prerequisites }

#### 중요: 보안 및 규정 준수 필수 { #critical-security-compliance-required }

이 핸들러를 수용하기 전에 비즈니스는 플랫폼에 등록해 `/detokenize` 호출에 필요한
인증 자격 증명을 발급받아야 합니다.

비즈니스는 `/detokenize` 엔드포인트로 민감한 결제 수단 정보를 받는 당사자이므로,
반드시 취급하는 자격 증명 유형에 해당하는 데이터 보안 표준을 준수해야 합니다(**MUST**)
(예: 카드의 경우 PCI DSS). 여기에는 다음이 포함됩니다.

* 안전한 전송(강력한 암호 스위트를 사용하는 HTTPS/TLS)
* 결제 처리 중 민감한 데이터의 안전한 취급
* 금융 결제 수단의 저장 및 처리에 관한 모든 규정 준수

비즈니스는 PSP가 자신을 대신해 역토큰화하도록 설정할 수도 있습니다(PSP 역시
규정을 준수해야 합니다).

**사전 요건 산출물:**

| 필드                       | 설명                                                          |
| :------------------------- | :------------------------------------------------------------ |
| `identity.access_token`    | 온보딩 시 플랫폼이 부여한 비즈니스 식별자                     |
| 인증 자격 증명             | `/detokenize` 호출 인증용 API 키 또는 OAuth 토큰              |

### 핸들러 설정 { #handler-configuration }

비즈니스는 플랫폼의 토큰화 핸들러를 선언합니다. `config`에는 플랫폼에 등록된 비즈니스 신원이
담기며, 이 값은 토큰 바인딩에 쓰입니다. `spec`으로 참조하는 플랫폼의 핸들러 명세에는
플랫폼의 **결제 자격 증명 제공자**가 노출하는 `/detokenize` 엔드포인트 URL이 나와 있습니다.

이 핸들러는 토큰화 입력으로 [PanCredential](site:schemas/common/types/pan_credential.json)과 [NetworkTokenCredential](site:schemas/common/types/network_token_credential.json)을 받고, 체크아웃용으로 [TokenCredential](site:schemas/common/types/token_credential.json)을 생성합니다.

**참고:** `/detokenize`의 결과에는 **민감한 결제 데이터**가 담깁니다. 송신자(플랫폼의
자격 증명 제공자)와 수신자(비즈니스 또는 PSP) 모두 반드시 해당 자격 증명 유형의 관련
표준을 준수해야 합니다(**MUST**)(예: 카드의 경우 PCI DSS).

#### 비즈니스 설정(디스커버리) { #business-config-discovery }

| 필드          | 타입   | 필수     | 설명                                        |
| :------------ | :----- | :------- | :------------------------------------------ |
| `environment` | string | 예       | API 환경(`sandbox` 또는 `production`)       |
| `business_id` | string | 예       | 플랫폼이 부여한 비즈니스 식별자             |

#### 비즈니스 핸들러 선언 예시 { #example-business-handler-declaration }

<!-- ucp:example schema=profile def=business_schema extract=$.ucp.payment_handlers target=$.ucp.payment_handlers -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "payment_handlers": {
      "com.example.platform_tokenizer": [
        {
          "id": "platform_wallet",
          "version": "{{ ucp_version }}",
          "spec": "https://platform.example.com/ucp/handler.json",
          "schema": "https://platform.example.com/ucp/handler/schema.json",
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
            "business_id": "business_abc123"
          }
        }
      ]
    }
  }
}
```

#### 응답 설정(체크아웃) { #response-config-checkout }

응답 설정에는 런타임 토큰 수명 주기 정보가 포함됩니다.

| 필드                | 타입    | 필수     | 설명                          |
| :------------------ | :------ | :------- | :---------------------------- |
| `environment`       | string  | 예       | API 환경                      |
| `business_id`       | string  | 예       | 비즈니스 식별자               |
| `token_ttl_seconds` | integer | 아니요   | 토큰 유효 시간(초)            |

#### 응답 설정 예시 { #example-response-config }

<!-- ucp:example schema=payment_handler def=response_schema -->
```json
{
  "id": "platform_wallet",
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
    "business_id": "business_abc123",
    "token_ttl_seconds": 900
  }
}
```

### 결제 처리 { #processing-payments }

토큰 자격 증명이 담긴 체크아웃을 받으면 다음을 수행합니다.

1. **핸들러 검증:** `instrument.handler_id`가 예상한 핸들러 ID와 일치하는지 확인합니다.
2. **역토큰화 또는 위임:**
   * **옵션 A(직접):** 플랫폼 **자격 증명 제공자**의 `/detokenize` 엔드포인트를 직접 호출한 뒤 결제를 처리합니다.
   * **옵션 B(위임):** 토큰을 PSP에 전달해 역토큰화와 결제 처리를 맡깁니다.
3. **응답 반환:** 확정된 체크아웃 상태로 응답합니다.

옵션 B는 [PSP 연동](#psp-integration) 섹션을 참고하세요.

#### 역토큰화 요청 예시(비즈니스) { #detokenize-request-example-business }

<!-- ucp:example skip reason="tokenization API, not UCP payload" -->
```json
POST https://provider.platform.example.com/ucp/detokenize
Content-Type: application/json
Authorization: Bearer {business_api_key}

{
  "token": "ptok_x9y8z7w6v5u4",
  "binding": {
    "type": "dev.ucp.shopping.checkout",
    "id": "checkout_789"
  }
}
```

참고: 비즈니스가 직접 인증하는 경우에는 `identity`가 필요 없습니다. 플랫폼이 API 키로
비즈니스를 식별할 수 있기 때문입니다.

---

## 플랫폼 연동 { #platform-integration }

### 사전 요건 { #prerequisites_1 }

이 핸들러는 **규정을 준수하는 결제 자격 증명 제공자**나 지갑 서비스를 운영하는 플랫폼이
구현합니다. 민감한 데이터를 다루고 `/detokenize` 엔드포인트를 노출하는 쪽은 메인 플랫폼
애플리케이션이 아니라 결제 자격 증명 제공자입니다. 이 핸들러를 구현하려면 플랫폼은 다음을 해야 합니다.

1. 자격 증명 저장과 취급에 관한 규정 준수를 유지하는 **규정 준수 결제 자격 증명 제공자**를 배포합니다.
2. 자격 증명 제공자에서 API 패턴을 따르는 `/detokenize` 엔드포인트를 노출합니다.
3. 자격 증명 제공자의 `/detokenize` 엔드포인트를 호출할 비즈니스와 PSP를 온보딩합니다.

**구현 요건:**

| 요건                   | 설명                                                                                            |
| :--------------------- | :---------------------------------------------------------------------------------------------- |
| `/detokenize` endpoint | 플랫폼 애플리케이션이 아니라 규정을 준수하는 결제 자격 증명 제공자가 노출합니다                 |
| 토큰 저장소            | 자격 증명 제공자 안에서 토큰을 바인딩 메타데이터와 함께 자격 증명에 매핑합니다                    |
| 참여자 허용 목록       | 온보딩된 비즈니스/PSP만 자격 증명 제공자의 `/detokenize`를 호출할 수 있습니다                       |
| 바인딩 검증            | 역토큰화 시 결제 자격 증명 제공자가 `binding`과 요청 참여자를 검증합니다                          |

### 핸들러 설정(플랫폼) { #handler-configuration-platform }

플랫폼은 `platform_config`를 사용해 UCP 프로필의 `payment_handlers` 레지스트리에 이
핸들러를 선언합니다.

#### 플랫폼 설정(디스커버리) { #platform-config-discovery }

| 필드                        | 타입    | 필수     | 설명                                        |
| :-------------------------- | :------ | :------- | :------------------------------------------ |
| `environment`               | string  | 예       | API 환경(`sandbox` 또는 `production`)       |
| `platform_id`               | string  | 예       | 플랫폼 식별자                               |
| `default_token_ttl_seconds` | integer | 아니요   | 비즈니스에 제공하는 기본 토큰 TTL           |

#### 플랫폼 핸들러 선언 예시 { #example-platform-handler-declaration }

<!-- ucp:example schema=profile def=business_schema extract=$.ucp.payment_handlers target=$.ucp.payment_handlers -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "payment_handlers": {
      "com.example.platform_tokenizer": [
        {
          "id": "platform_wallet",
          "version": "{{ ucp_version }}",
          "spec": "https://platform.example.com/ucp/handler.json",
          "schema": "https://platform.example.com/ucp/handler/schema.json",
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
            "default_token_ttl_seconds": 900
          }
        }
      ]
    }
  }
}
```

### 토큰 생성 { #token-generation }

플랫폼 애플리케이션은 결제 흐름을 조율하지만 **민감한 결제 데이터에는 절대 접근하지
않습니다**. 대신 다음과 같이 동작합니다.

1. 플랫폼의 **결제 자격 증명 제공자**가 결제 자격 증명을 안전하게 보관합니다.
2. 결제가 필요하면 플랫폼 애플리케이션이 자격 증명 제공자에게 토큰을 요청합니다.
3. 자격 증명 제공자는 `binding` 리소스에 바인딩되고 (핸들러 선언에 있는) 비즈니스의 `identity`에 발급되는 토큰을 생성합니다.
4. 자격 증명 제공자가 플랫폼 애플리케이션에 토큰을 반환합니다.
5. 플랫폼 애플리케이션은 체크아웃을 제출할 때 이 토큰을 포함합니다.

이렇게 분리하면 플랫폼 애플리케이션 자체는 민감한 결제 수단 정보를 다루거나 접근할 일이
전혀 없습니다.

### 체크아웃 제출 { #submitting-checkout }

플랫폼 애플리케이션은 (자체 결제 자격 증명 제공자에게서 받은) 토큰과 함께 체크아웃을
제출합니다.

<!-- ucp:example schema=shopping/checkout op=complete direction=request -->
```json
POST /checkout-sessions/{checkout_id}/complete
Content-Type: application/json

{
  "payment": {
    "instruments": [
      {
        "id": "instr_1",
        "handler_id": "platform_wallet",
        "type": "card",
        "selected": true,
        "display": {
          "brand": "visa",
          "last_digits": "4242"
        },
        "credential": {
          "type": "token",
          "token": "ptok_x9y8z7w6v5u4"
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

## 런타임 결제 인증 액션 { #runtime-payment-authentication-actions }

이 핸들러는 협상된 [결제 인증 확장](../extensions/authentication.md)의 두 가지 액션
유형을 모두 지원합니다.

| 액션 유형 | 용도 |
| :---------- | :-- |
| `dev.ucp.common.payment.device_data_collection` | 선택된 토큰화 결제 수단의 기기/브라우저 데이터를 수집합니다. |
| `dev.ucp.common.payment.three_ds_challenge` | 결제 완료 중 구매자에게 3DS 챌린지를 표시합니다. |

두 단계 중 하나가 필요하면 비즈니스는 선택된 결제 수단을 처리하는 도중 해당 액션을
반환합니다. 액션의 `config.payment_instrument_id`가 그 결제 수단을 식별하며, 플랫폼은 이 값으로
이 핸들러와 핸들러의 신뢰 정책을 확인합니다. 플랫폼은 확장에 따라 액션을 처리하고
`action.done` 이후 체크아웃을 조회합니다. 제출된 토큰과 결제 수단은 바뀌지 않으며, 비즈니스는
자신의 제공자 상태를 바탕으로 결제 결과를 결정합니다.

---

## PSP 연동 { #psp-integration }

### 사전 요건 { #prerequisites_2 }

#### 중요: 보안 및 규정 준수 필수 { #critical-security-compliance-required_1 }

비즈니스를 대신해 역토큰화하기 전에 PSP는 자신이 처리를 맡은 비즈니스 목록을 제출하고
플랫폼에 등록해야 합니다.

PSP는 `/detokenize` 엔드포인트로 민감한 결제 수단 정보를 받는 당사자이므로, 반드시
취급하는 자격 증명 유형에 해당하는 보안 표준을 **준수해야** 합니다(**MUST**)(예: 카드의 경우
PCI DSS). 여기에는 다음이 포함됩니다.

* 안전한 전송(강력한 암호 스위트를 사용하는 HTTPS/TLS)
* 결제 처리 중 민감한 데이터의 안전한 취급
* 금융 결제 수단의 저장 및 처리에 관한
  모든 규정 준수

**사전 요건 산출물:**

| 필드                       | 설명                                                          |
| :------------------------- | :------------------------------------------------------------ |
| 인증 자격 증명             | `/detokenize` 호출 인증용 API 키 또는 OAuth 토큰              |
| 비즈니스 연결              | 이 PSP가 대신 역토큰화할 수 있는 비즈니스 신원 목록           |

### 역토큰화 흐름 { #detokenization-flow }

비즈니스가 PSP에 토큰을 전달하면 PSP는 다음을 수행합니다.

1. 결제 수단에서 토큰을 추출합니다.
2. `binding`과 함께 비즈니스의 `identity`를 담아 플랫폼 **결제 자격 증명 제공자**의
   `/detokenize` 엔드포인트를 호출합니다.
3. 반환된 자격 증명으로 결제를 처리합니다.

#### 역토큰화 요청 예시(PSP) { #detokenize-request-example-psp }

<!-- ucp:example skip reason="tokenization API, not UCP payload" -->
```json
POST https://provider.platform.example.com/ucp/detokenize
Content-Type: application/json
Authorization: Bearer {psp_api_key}

{
  "token": "ptok_x9y8z7w6v5u4",
  "binding": {
    "type": "dev.ucp.shopping.checkout",
    "id": "checkout_789"
  },
  "identity": {
    "access_token": "business_abc123"
  }
}
```

참고: 여기서는 `identity`가 **필수입니다**. PSP는 비즈니스를 대신해 호출하므로 어느
비즈니스의 토큰을 조회하는지 명시해야 합니다.

플랫폼의 결제 자격 증명 제공자는 다음을 검증합니다.

* PSP가 `identity`로 지정된 비즈니스를 대신할 권한이 있는지
* `binding`이 최초 토큰화 시점과 일치하고, `identity`가 토큰을 발급받은 참여자로
  확인되는지
* 토큰이 만료되거나 이미 사용되지 않았는지

---

## 보안 고려 사항 { #security-considerations }

| 요건 | 설명 |
| :---------- | :---------- |
| **규정 준수(자격 증명 제공자)** | 플랫폼의 자격 증명 제공자는 민감한 결제 수단 정보를 취급·저장할 때 반드시 해당 자격 증명 유형의 관련 표준(예: 카드의 경우 PCI DSS)을 준수해야 합니다(**MUST**). |
| **규정 준수(수신자)** | `/detokenize`를 호출하는 비즈니스/PSP는 민감한 데이터 페이로드를 받을 때 반드시 해당 자격 증명 유형의 관련 표준을 준수해야 합니다(**MUST**). |
| **안전한 전송** | `/detokenize`로 데이터를 전송할 때는 반드시 강력한 암호 스위트를 사용하는 HTTPS/TLS를 써야 합니다(**MUST**). |
| **플랫폼 앱 접근 금지** | 플랫폼 애플리케이션은 민감한 데이터를 다뤄서는 안 됩니다(**MUST NOT**). 민감한 데이터는 규정을 준수하는 결제 자격 증명 제공자만 다룹니다. |
| **엔드포인트 분리** | `/detokenize` 엔드포인트는 반드시 플랫폼 애플리케이션이 아닌 결제 자격 증명 제공자가 노출해야 합니다(**MUST**). |
| **참여자 인증** | 플랫폼의 자격 증명 제공자는 `/detokenize` 호출을 받기 전에 반드시 비즈니스/PSP를 인증해야 합니다(**MUST**). |
| **참여자에게 발급** | 토큰은 반드시 핸들러 선언에 있는 비즈니스의 `identity`에 발급해야 합니다(**MUST**). |
| **리소스 바인딩** | 토큰은 반드시 특정 `binding` 리소스에 바인딩해야 합니다(**MUST**). |
| **호출자 검증** | 플랫폼은 반드시 인증된 호출자가 토큰을 발급받은 참여자(또는 권한 있는 PSP)인지 검증해야 합니다(**MUST**). |
| **일회용** | 토큰은 역토큰화 후 무효화해야 합니다(**SHOULD**). |
| **짧은 TTL** | 토큰은 짧은 시간 안에 만료되어야 합니다(**SHOULD**). |
| **HTTPS 필수** | 모든 `/detokenize` 호출은 TLS를 사용해야 합니다. |

---

## 참고 자료 { #references }

* **패턴:** [토큰화 결제 핸들러](../guide.md)
* **API 패턴:** [handlers/tokenization/openapi.json](site:handlers/tokenization/openapi.json)
* **신원 스키마:** [schemas/common/types/payment_identity.json](site:schemas/common/types/payment_identity.json)
