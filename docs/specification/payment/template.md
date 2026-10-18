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

# {Handler Name} Payment Handler

* **핸들러 이름:** `{reverse-dns.name}`
* **버전:** `{YYYY-MM-DD}`

## 소개 { #introduction }

{이 핸들러로 할 수 있는 일과 지원하는 결제 흐름을 간단히 설명}

### 주요 이점 { #key-benefits }

* {이점 1}
* {이점 2}
* {이점 3}

### 통합 가이드 { #integration-guide }

| 참여자  | 통합 섹션                           |
| :----------- | :-------------------------------------------- |
| **비즈니스** | [비즈니스 통합](#business-integration) |
| **플랫폼** | [플랫폼 통합](#platform-integration) |

---

## 참여자 { #participants }

{이 핸들러의 모든 참여자와 각 역할을 설명합니다.}

> **용어 참고:**
> 이 명세에서는 참여자를 **"비즈니스"**라고 부르지만, 기술적인 스키마 필드에는
> 업계 표준 명칭인 **`merchant_*`**(예: `merchant_id`)가 남아 있을 수 있습니다.
> 대응 관계는 아래에 정리합니다.

| 참여자             | 역할               | 사전 요구 사항                |
| :---------------------- | :----------------- | :--------------------------- |
| **비즈니스**            | {역할 설명} | {예/아니요 — 간단한 설명} |
| **플랫폼**            | {역할 설명} | {예/아니요 — 간단한 설명} |
| **{Other Participant}** | {역할 설명} | {예/아니요 — 간단한 설명} |

{선택 사항: 참여자 간 관계를 보여 주는 ASCII 다이어그램}

```text
+---------+     +---------------+     +------------+
|Platform |     |   {Provider}  |     |  Business  |
+----+----+     +-------+-------+     +------+-----+
     |                  |                    |
     |  {step 1}        |                    |
     |----------------->|                    |
     |                  |                    |
     |  {step 2}        |                    |
     |<-----------------|                    |
     |                  |                    |
     |  {step 3}                             |
     |-------------------------------------->|
```

---

<!--
  PARTICIPANT INTEGRATION SECTIONS

  Include one section per participant. Each section follows the same structure:
  - Prerequisites (onboarding, setup)
  - Configuration or Protocol (what they need to do)
  - Examples

  Number sections starting from 3. Add more sections as needed for additional participants.
-->

## 비즈니스 통합 { #business-integration }

### 사전 요구 사항 { #prerequisites }

이 핸들러를 알리기 전에 비즈니스는 반드시 다음을 완료해야 합니다(**MUST**).

1. {사전 요구 사항 1, 예: "{provider}에 등록해 비즈니스 식별자를 발급받습니다"}
2. {사전 요구 사항 2}

**사전 요구 사항 결과물:**

| 필드                   | 설명                                      |
| :---------------------- | :----------------------------------------------- |
| `identity.access_token` | {할당되는 식별자, 예: business_id} |
| {추가 설정}     | {온보딩에서 받은 추가 설정}   |

### 핸들러 설정 { #handler-configuration }

비즈니스는 UCP 프로필의 `payment_handlers` 레지스트리에서 이 핸들러 지원 여부를
알립니다.

#### 핸들러 스키마 { #handler-schema }

**스키마 URL:** `{schema_url}`

핸들러 스키마는 맥락별로 세 가지 설정 변형을 정의합니다. 전체 패턴은
[결제 핸들러 가이드: 스키마 정의](guide.md#defining-the-schema)를 참고하세요.

| 설정 변형 | 맥락 | 목적 |
| :------------- | :------ | :------ |
| `business_config` | 비즈니스 디스커버리 | {비즈니스별 필드 설명} |
| `platform_config` | 플랫폼 디스커버리 | {플랫폼별 필드 설명} |
| `response_config` | 체크아웃 응답 | {런타임 필드 설명} |

#### 비즈니스 설정 필드 { #business-config-fields }

| 필드   | 유형   | 필수 | 설명   |
| :------ | :----- | :------- | :------------ |
| {필드} | {유형} | {예/아니요} | {설명} |

#### 응답 설정 필드 { #response-config-fields }

| 필드   | 유형   | 필수 | 설명   |
| :------ | :----- | :------- | :------------ |
| {필드} | {유형} | {예/아니요} | {설명} |

#### 핸들러 선언 예시 { #example-handler-declaration }

<!-- ucp:example skip reason="handler config template" -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "payment_handlers": {
      "{handler_name}": [
        {
          "id": "{handler_id}",
          "version": "{version}",
          "spec": "{spec_url}",
          "schema": "{schema_url}",
          "available_instruments": [
            {
              "type": "{instrument_type}",
              "constraints": {
                // Type-specific constraints
              }
            }
          ],
          "config": {
            // Handler-specific configuration
          }
        }
      ]
    }
  }
}
```

### 결제 처리 { #processing-payments }

이 핸들러의 결제 수단으로 결제를 받으면 비즈니스는 반드시 다음을 수행해야
합니다(**MUST**).

1. **핸들러 검증:** `instrument.handler_id`가 알린 핸들러와 일치하는지 확인합니다.
2. **멱등성 보장:** 요청이 재시도라면(이전 `checkout_id` 또는 멱등성 키와
  일치하면) 자금을 다시 처리하지 않고 이전 결과를 즉시 반환합니다.
3. **{Step 3}:** {설명}
4. **{Step 4}:** {설명}
5. **응답 반환:** 최종 확정된 체크아웃 상태로 응답합니다.

{비즈니스가 외부 서비스를 호출한다면 요청/응답 예시를 포함합니다}

---

## 플랫폼 통합 { #platform-integration }

### 사전 요구 사항 { #prerequisites_1 }

이 핸들러를 사용하기 전에 플랫폼은 반드시 다음을 완료해야 합니다(**MUST**).

1. {사전 요구 사항 1, 예: "{provider}에 등록해 플랫폼 식별자를 발급받습니다"}
2. {사전 요구 사항 2}

**사전 요구 사항 결과물:**

| 필드                   | 설명                                    |
| :---------------------- | :--------------------------------------------- |
| `identity.access_token` | {할당되는 식별자}                  |
| {추가 설정}     | {온보딩에서 받은 추가 설정} |

### 핸들러 설정 { #handler-configuration_1 }

플랫폼은 `platform_config`를 사용해 UCP 프로필의 `payment_handlers`
레지스트리에서 이 핸들러 지원 여부를 알립니다.

#### 플랫폼 설정 필드 { #platform-config-fields }

| 필드   | 유형   | 필수 | 설명   |
| :------ | :----- | :------- | :------------ |
| {필드} | {유형} | {예/아니요} | {설명} |

#### 플랫폼 핸들러 선언 예시 { #example-platform-handler-declaration }

<!-- ucp:example skip reason="handler config template" -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "payment_handlers": {
      "{handler_name}": [
        {
          "id": "{handler_id}",
          "version": "{version}",
          "spec": "{spec_url}",
          "schema": "{schema_url}",
          "available_instruments": [
            {
              "type": "{instrument_type}",
              "constraints": {
                // Type-specific constraints the platform supports
              }
            }
          ],
          "config": {
            // Platform-specific configuration
          }
        }
      ]
    }
  }
}
```

### 결제 프로토콜 { #payment-protocol }

플랫폼은 결제 수단을 획득하기 위해 반드시 다음 흐름을 따라야 합니다(**MUST**).

#### 1단계: 핸들러 디스커버리 { #step-1-discover-handler }

플랫폼은 비즈니스 UCP 프로필(`/.well-known/ucp`)의 `payment_handlers`
레지스트리에서 `{handler_name}`을 찾습니다.

<!-- ucp:example skip reason="handler config template" -->
```json
{
  "ucp": {
    "payment_handlers": {
      "{handler_name}": [
        {
          "id": "{handler_id}",
          "version": "{version}",
          "available_instruments": [
            {"type": "{instrument_type}"}
          ],
          "config": {
            // Business's configuration
          }
        }
      ]
    }
  }
}
```

#### Step 2: {Action Name}

{이 단계에서 플랫폼이 하는 일 설명}

{해당하는 경우 코드 예시:}

```javascript
// Example SDK usage or API call
```

#### Step 3: {Action Name}

{나머지 단계도 같은 방식으로 계속...}

#### N단계: 체크아웃 완료 { #step-n-complete-checkout }

플랫폼은 구성한 결제 수단과 함께 체크아웃을 제출합니다.

<!-- ucp:example skip reason="handler config template" -->
```json
POST /checkout-sessions/{checkout_id}/complete
Content-Type: application/json

{
  "payment": {
    "instruments": [
      {
        "id": "{instrument_id}",
        "handler_id": "{handler_id}",
        "type": "{instrument_type}",
        "credential": {
          "type": "{credential_type}",
          // Credential fields
        }
        // Additional instrument fields
      }
    ]
  },
  "signals": {
    // Platform-observed signals (buyer connection and device)
  }
}
```

---

## 결제 액션 [조건부] { #payment-actions-conditional }

{핸들러가 액션을 일으킬 수 있다면 이 섹션을 포함합니다. 액션을 선언하는 협상된
체크아웃 확장을 명시합니다. 기기 데이터 수집과 3DS 챌린지에는 표준
[결제 인증 확장](extensions/authentication.md)을 사용하고, 사용자 정의 액션은 해당
액션을 정의하는 확장 명세에 링크합니다.}

| 액션 유형 | 내보내는 시점 | 차단되는 효과 | 핸들러별 요구 사항 |
| :---------- | :----------- | :----------- | :---------------------------- |
| `{action_type}` | {조건} | {효과} | {결제사별 신뢰, 대체, 재시도, 정리 동작} |

표준 결제 인증 액션이라면 이 핸들러가 일으킬 수 있는 유형, 신뢰할 수 있는 URL
규칙, 액션과 결제 수단의 연결 방식, 비즈니스가 결제사 상태를 관찰하는 방식처럼 이
핸들러에 특화된 동작만 문서화합니다. 사용자 정의 액션은 설정과 플랫폼 처리 계약도
함께 문서화하거나 링크합니다.

---

<!--
  ADDITIONAL PARTICIPANT SECTIONS

  Add one section per additional participant (PSP, Tokenizer, Wallet Provider, etc.)
  following the same pattern as Business and Platform integration.
-->

## {Participant} Integration

### 사전 요구 사항 { #prerequisites_2 }

이 핸들러의 흐름에 참여하기 전에 {participants}는 반드시 다음을 완료해야
합니다(**MUST**).

1. {사전 요구 사항 1}
2. {사전 요구 사항 2}

**사전 요구 사항 결과물:**

| 필드                   | 설명                                    |
| :---------------------- | :--------------------------------------------- |
| `identity.access_token` | {할당되는 식별자}                  |
| {추가 설정}     | {온보딩에서 받은 추가 설정} |

### {Action or Configuration}

{이 참여자가 해야 할 일을 설명합니다.}

{필요하면 예시를 포함합니다.}

---

## 보안 고려 사항 { #security-considerations }

| 요구 사항                  | 설명                                                                                                                                                           |
| :--------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **바인딩 필수**         | 재사용을 막기 위해 자격 증명은 반드시 `binding` 리소스(`type`과 `id`)에 바인딩되어야 하고, 정확히 한 참여자에게만 발급되어야 합니다(**MUST**).                                     |
| **바인딩 위치**        | `binding` 객체는 전송 헤더가 아니라 `credential` 페이로드 안에 포함해 서명 범위에 들어가도록 해야 합니다(**SHOULD**).               |
| **바인딩 검증**         | 처리하는 참여자는 처리 전에 반드시 바인딩이 일치하는지 검증해야 합니다(**MUST**).                                                                                         |
| **토큰 만료**             | {토큰을 사용하는 경우: 토큰은 반드시 {duration} 후 만료되거나 일회용이어야 합니다(**MUST**).}                                                                                             |
| **데이터 보관 위치**           | {현지 법률을 준수하기 위해 PII를 반드시 특정 지역(예: EU, US)에서 처리·저장해야 하는지(**MUST**) 명시합니다.}                                                |
| **{Additional requirement}** | {설명}                                                                                                                                                         |

---

## 참고 자료 { #references }

* **핸들러 명세:** `{spec_url}`
* **핸들러 스키마:** `{schema_url}` (설정, 결제 수단, 자격 증명의 형태를 정의합니다)
