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

# 구매자 동의 확장 { #buyer-consent-extension }

## 개요 { #overview }

구매자 동의(Buyer Consent) 확장을 사용하면 비즈니스는 자신이 제공하는 동의 옵션을
알리고, 플랫폼은 데이터 사용과 커뮤니케이션 선호에 대한 구매자의 동의 결정을 전달할 수
있다. 대상 목적에는 마케팅, 분석, 선호 설정, 데이터 판매 또는 공유 등이 있다.

동의는 2단계 구조로 모델링한다.

- **목적(Purpose)** — 동의의 대상. 최상위 수준에서 역DNS 식별자
    (예: `dev.ucp.consent.marketing`)를 키로 사용한다. 각 목적은 `granted` 상태,
    그 상태를 누가 지정했는지 나타내는 `source`, 사람이 읽을 수 있는 `description`,
    선택 사항인 `links`를 가진다.
- **세그먼트(Segment)** — 상위 목적의 범위를 특정 채널(예: 이메일, SMS), 벤더
    (예: 측정 제공업체), 또는 프로그램으로 좁히는 선택적 세분화 단위. 각 세그먼트는
    같은 형태(`granted`, `source`, `description`, 선택 사항인 `links`)를 가지며,
    해당 범위에서는 상위 목적보다 우선한다.

중첩은 한 단계로 제한된다. 목적은 세그먼트를 가지지만, 세그먼트는 더 이상 중첩되지
않는다.

## 범위 { #scope }

이 확장은 비즈니스와 플랫폼이 구매자 동의 상태를 확정하고 주고받는 메커니즘을
정의한다. 그 상태에 따라 각 당사자가 어떻게 행동해야 하는지는 정하지 않는다.
비즈니스가 동의 상태에 따라 어떻게 행동하는지(예: 이메일 발송, 데이터 공유), 플랫폼이
기존 결정을 구매자에게 어떻게 보여 주는지(예: 현재 구독 상태와 구독 해지 수단 표시)는
이 명세가 아니라 비즈니스 정책과 관련 규정이 정한다.

비즈니스는 동의 옵션을 알리기 전에 적용되는 정책에 따라 초기 `granted` 값을 정할
책임이 있다. `source: "business"`로 알린 값은 비즈니스의 권위 있는 기본값이며,
구매자는 플랫폼을 통해 이 값을 변경할 수 있다. 정책상의 판단은 계속 비즈니스의
책임이다.

## 디스커버리 { #discovery }

비즈니스는 프로필에서 동의 지원 여부를 알린다. 이 기능은 장바구니, 체크아웃, 또는
둘 다를 확장할 수 있다.

<!-- ucp:example schema=profile def=business_schema extract=$.capabilities target=$.ucp.capabilities -->

```json
{
  "capabilities": {
    "dev.ucp.shopping.buyer_consent": [
      {
        "version": "{{ ucp_version }}",
        "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/buyer-consent",
        "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/buyer_consent.json",
        "extends": [
          "dev.ucp.shopping.cart",
          "dev.ucp.shopping.checkout"
        ]
      }
    ]
  }
}
```

## 스키마 구성 { #schema-composition }

이 기능이 활성화되면 장바구니 및/또는 체크아웃 안의 **buyer 객체**에 `consent`
필드가 추가된다.

- **장바구니 경로**: `cart.buyer.consent`
- **체크아웃 경로**: `checkout.buyer.consent`

`buyer` 객체(따라서 `consent`도)는 모든 장바구니·체크아웃 작업, 즉 장바구니
`create` / `update`와 체크아웃 `create` / `update` / `complete`에서 선택 사항이다.
플랫폼은 수집한 동의를 결제 정보와 함께 `complete_checkout`에서 제출할 수
있다(MAY).

## 스키마 정의 { #schema-definition }

### Consent 객체 { #consent-object }

목적별 동의 결정을 담은 맵이다. 키는 `dev.ucp.consent.marketing`이나
`com.example.purpose.loyalty` 같은 역DNS 목적 식별자이고, 값은
[Consent Purpose](#consent-purpose) 객체다. 목적 안에는 채널, 벤더, 프로그램 범위에
대한 [Consent Segment](#consent-segment) 세분화가 중첩될 수 있다.

### Consent Purpose { #consent-purpose }

{{ extension_schema_fields('buyer_consent.json#/$defs/consent_purpose', 'shopping/extensions/buyer-consent') }}

### Consent Segment { #consent-segment }

{{ extension_schema_fields('buyer_consent.json#/$defs/consent_segment', 'shopping/extensions/buyer-consent') }}

## 잘 알려진 목적 { #well-known-purposes }

이 확장은 `dev.ucp.consent.*` 네임스페이스 아래에 잘 알려진 목적 식별자 네 가지를
정의한다. 비즈니스는 자신이 관리하는 역DNS 네임스페이스 아래에 목적을 추가로 정의할 수
있다. 이 식별자들은 운영상의 정의이며 법적 분류 체계가 아니다.

| 식별자                            | 목적                                                      |
| --------------------------------- | --------------------------------------------------------- |
| `dev.ucp.consent.analytics`       | 분석 및 성능 측정                                         |
| `dev.ucp.consent.marketing`       | 마케팅 커뮤니케이션                                       |
| `dev.ucp.consent.preferences`     | 구매자 선호 기억                                          |
| `dev.ucp.consent.sale_or_sharing` | 제3자에 대한 개인 데이터 판매 또는 공유                   |

## 잘 알려진 세그먼트 { #well-known-segments }

`dev.ucp.consent.marketing` 목적에 대해 이 확장은 다음과 같은 잘 알려진 채널
세그먼트 식별자를 정의한다.

| 식별자                            | 채널    |
| --------------------------------- | ------- |
| `dev.ucp.consent.marketing.email` | 이메일  |
| `dev.ucp.consent.marketing.sms`   | SMS     |

## 알림과 확인 { #advertise-and-confirm }

같은 맵 형태가 서로 보완적인 두 방향의 정보를 전달한다.

| 방향                     | 채우는 주체   | 채우는 필드                                                               |
| ------------------------ | ------------- | ------------------------------------------------------------------------- |
| **알림(Advertise)** (응답) | 비즈니스    | `granted`, `source`, `description`; 있는 경우 `links`와 `segments`        |
| **확인(Confirm)** (요청) | 플랫폼        | `granted`, `source`; 있는 경우 `segments`                                 |

`description`과 `links`는 응답에만 쓰이며, 플랫폼은 확인할 때 이를 되돌려 보내지
않는다. `granted`와 `source`는 양방향 모두 필수다. 비즈니스는 알릴 때 현재 상태와 그
출처를 보내고, 플랫폼은 확인할 때 현재 상태와 그 출처를 보낸다.

`source` 필드는 현재 `granted` 값을 누가 정했는지를 나타낸다.

- `source: "business"`는 값이 비즈니스의 기본값을 반영하며, 적용되는 구매자 선호가
  없다는 뜻이다.
- `source: "platform"`은 값이 플랫폼이 수집한 구매자의 명시적 선호를 반영한다는
  뜻이다.

source는 플랫폼이 현재 값을 어떻게 다뤄야 하는지 알려 준다.
`source: "business"`는 플랫폼이 구매자에게 선택지를 제시해 참여를 유도하라는 의미이고,
`source: "platform"`은 기록된 구매자 선호가 있다는 의미다.

### 예시: 목적과 세그먼트 { #example-purposes-and-segments }

가장 단순한 경우는 목적 수준에서 동의를 수집하는 것이다. 여기서 `analytics`는 상위
목적에 기록된 구매자의 동의를 보여 준다. 목적은 더 세밀한 제어를 위해 세그먼트를 가질
수도 있다. `marketing`은 꺼져 있지만 구매자는 SMS 마케팅에 동의했다. 해당 범위에서는
세그먼트의 `granted: true`가 상위의 `granted: false`보다 우선한다(더 구체적인 값이
우선).

<!-- ucp:example schema=shopping/buyer_consent def=consent op=read extract=$.buyer.consent -->

```json
{
  "ucp": { ... },
  "id": "checkout_456",
  "status": "ready_for_complete",
  "currency": "USD",
  "buyer": {
    "consent": {
      "dev.ucp.consent.analytics": {
        "granted": true,
        "source": "platform",
        "description": "Site analytics and performance measurement"
      },
      "dev.ucp.consent.marketing": {
        "granted": false,
        "source": "business",
        "description": "Promotional communications",
        "segments": {
          "dev.ucp.consent.marketing.email": {
            "granted": false,
            "source": "business",
            "description": "Promotional emails"
          },
          "dev.ucp.consent.marketing.sms": {
            "granted": true,
            "source": "platform",
            "description": "Marketing text messages"
          }
        }
      }
    }
  }
}
```

구매자는 두 가지 선택(분석 전반, SMS 마케팅 개별)에 직접 응답했고, 둘 다
`source: "platform"`을 가진다. 나머지 값(상위 `marketing`, `email` 세그먼트)은
`source: "business"`를 가진다.

### 알림 예시 { #advertise-example }

비즈니스는 제공하는 목적과 세그먼트를 현재 동의 상태(구매자의 이전 결정 또는
비즈니스 기본값)와 함께 알린다.

<!-- ucp:example schema=shopping/buyer_consent def=consent op=read extract=$.buyer.consent -->

```json
{
  "ucp": { ... },
  "id": "checkout_456",
  "status": "ready_for_complete",
  "currency": "USD",
  "buyer": {
    "consent": {
      "dev.ucp.consent.marketing": {
        "granted": false,
        "source": "business",
        "description": "Promotional communications across all channels",
        "links": [{ "type": "privacy_policy", "url": "https://example.com/privacy" }],
        "segments": {
          "dev.ucp.consent.marketing.email": {
            "granted": true,
            "source": "platform",
            "description": "Promotional emails and exclusive offers"
          },
          "dev.ucp.consent.marketing.sms": {
            "granted": false,
            "source": "business",
            "description": "Marketing text messages",
            "links": [{ "type": "terms_of_service", "url": "https://example.com/sms-terms" }]
          },
          "com.example.channel.marketing": {
            "granted": false,
            "source": "business",
            "description": "Marketing messages via a third-party channel"
          }
        }
      },
      "dev.ucp.consent.analytics": {
        "granted": true,
        "source": "business",
        "description": "Site analytics and performance measurement",
        "segments": {
          "com.example.analytics": {
            "granted": false,
            "source": "business",
            "description": "Third-party analytics measurement",
            "links": [{ "type": "privacy_policy", "url": "https://example.com/analytics-privacy" }]
          }
        }
      },
      "dev.ucp.consent.preferences": {
        "granted": true,
        "source": "business",
        "description": "Remember preferences and personalize the shopping experience"
      },
      "dev.ucp.consent.sale_or_sharing": {
        "granted": false,
        "source": "platform",
        "description": "Sale or sharing of personal data with third parties",
        "links": [{ "type": "privacy_policy", "url": "https://example.com/privacy" }]
      }
    }
  }
}
```

이 예시에서 구매자는 이전에 프로모션 이메일 수신에 동의했고(`marketing.email`이
`source: "platform"`), 데이터 판매는 이전에 거부했다(`sale_or_sharing`이
`source: "platform"`). 나머지 선택은 모두 비즈니스가 지정한 기본값 그대로이며,
플랫폼이 구매자에게 제시해 명시적으로 수집할 후보가 된다.

### 확인 예시 { #confirm-example }

플랫폼은 이후의 `update_checkout` 또는 `complete_checkout` 요청에서 알림받은 모든
목적과 세그먼트의 현재 상태를 제출한다. 각 선택에 대해, 적용되는 구매자 선호가 없으면
알림받은 `granted`와 `source`를 그대로 되돌려 보내고, 구매자 선호가 있으면
`source: "platform"`과 구매자가 밝힌 `granted` 값을 설정한다.

<!-- ucp:example schema=shopping/buyer_consent def=consent op=complete direction=request extract=$.buyer.consent -->

```json
{
  "buyer": {
    "consent": {
      "dev.ucp.consent.marketing": {
        "granted": true,
        "source": "platform",
        "segments": {
          "dev.ucp.consent.marketing.email": { "granted": true,  "source": "platform" },
          "dev.ucp.consent.marketing.sms":   { "granted": true,  "source": "platform" },
          "com.example.channel.marketing":   { "granted": false, "source": "business" }
        }
      },
      "dev.ucp.consent.analytics": {
        "granted": true,
        "source": "business",
        "segments": {
          "com.example.analytics": { "granted": false, "source": "business" }
        }
      },
      "dev.ucp.consent.preferences":     { "granted": true,  "source": "business" },
      "dev.ucp.consent.sale_or_sharing": { "granted": false, "source": "platform" }
    }
  }
}
```

여기서 구매자는 이메일과 SMS 마케팅 수신에 동의했다. 나머지 선택은 알림받은 값
그대로 되돌려 보냈으며, 여기에는 이전에 플랫폼이 수집한 선택을 유지하는
`sale_or_sharing`도 포함된다.

## 데이터 의존성 { #data-dependencies }

일부 동의 상태는 추가적인 구매자 데이터나 체크아웃 데이터에 의존한다. 예를 들어 SMS
마케팅 동의에는 구매자 전화번호가 필요하다. 플랫폼이 필요한 의존 데이터가 없는 동의
값을 확인하면, 비즈니스는 표준 [체크아웃 상태
수명 주기](../checkout/index.md#checkout-status-lifecycle)와 [에러
처리](../checkout/index.md#error-handling) 메커니즘을 통해 누락 사실을 알린다.

`create_cart`, `update_cart`, `create_checkout`, `update_checkout`에서 비즈니스는
누락된 의존 데이터를 `warning` 메시지로 알려, 플랫폼이 이후 작업에서 해당 데이터를
수집할 수 있게 해야 한다(SHOULD). 알린 동의 결정은 여전히 유효하며, 경고는 정보 제공
목적이다.

<!-- ucp:example schema=shopping/checkout target=$.messages op=read -->

```json
[
  {
    "type": "warning",
    "code": "missing_consent_data",
    "content": "Phone number is required for SMS marketing.",
    "path": "$.buyer.phone_number"
  }
]
```

`complete_checkout`에서 비즈니스는 확인된 동의 결정에 충족되지 않은 데이터 의존성이
있는 동안 체크아웃을 `completed`로 전환해서는 안 된다(MUST NOT). 누락된 의존
데이터는 반드시 표준 [에러
처리](../checkout/index.md#error-handling) 흐름으로 알려야 한다(MUST).

## 규범적 요구 사항 { #normative-requirements }

1. **알린 설정을 사용한다.** 비즈니스는 반드시 자신이 지원하는 동의 옵션 전체를
   알려야 한다(MUST). 어떤 선택지를 언제 구매자에게 제시할지는 플랫폼이 정한다.
   선택지를 제시할 때 플랫폼은 반드시 알림받은 `description`, `links`, `granted` 값,
   목적/세그먼트 묶음을 사용해야 한다(MUST). `source` 필드는 플랫폼의 판단에 참고하는
   정보이며([알림과 확인](#advertise-and-confirm) 참고) 사용자에게 보여 주는 콘텐츠가
   아니다. `dev.ucp.consent.*` 네임스페이스 밖의 식별자는 불투명한 핸들이며, 플랫폼은
   경로만 보고 그 의미를 추론해서는 안 된다(MUST NOT). UCP가 정의한 식별자는 이
   명세에서 정한 의미를 가진다.

2. **확인의 의미.** 요청에서 `consent` 필드는 선택 사항이다. 이 필드를 생략하면 동의
   업데이트가 없는 것으로 보고, 비즈니스는 이전 상태를 유지한다. `consent`를 제출할
   때 플랫폼은 반드시 알림받은 모든 목적과 세그먼트 키를 포함하고, 각 키마다
   `granted`와 `source`를 모두 담아야 한다(MUST). 제출한 `consent` 맵에서 알림받은
   키를 생략하는 방식으로 어떤 값을 표현해서는 안 된다(MUST NOT).

3. **알린 집합이 권위를 가진다.** 비즈니스는 이전 응답에서 알리지 않은 목적과
   세그먼트가 요청에 있으면 반드시 무시해야 한다(MUST). 플랫폼은 비즈니스가 알리지
   않은 목적이나 세그먼트를 구매자에게 묻거나 전송해서는 안 된다(MUST NOT).

4. **더 구체적인 값이 우선한다.** 알린 세그먼트에 대해서는 세그먼트의 `granted` 값이
   해당 세그먼트에 대한 상위 목적의 `granted`보다 우선한다.

5. **source 귀속에는 구매자가 밝힌 선호가 필요하다.** 플랫폼은 `granted` 값이
   구매자가 밝힌 선호를 반영하는 경우에만 `source: "platform"`을 설정해야 한다(MUST).
   적용되는 구매자 선호가 없는 선택에 대해서는 플랫폼이 반드시 알림받은 `source`를
   바꾸지 않고 그대로 되돌려 보내야 한다(MUST). 무엇을 구매자가 밝힌 선호로 볼지는
   프로토콜이 정하지 않으며, 플랫폼이 적용되는 정책에 따라 판단한다.

6. **비즈니스별 귀속.** 특정 비즈니스와의 상호작용에서 수집한 동의 결정은 그
   비즈니스에만 귀속된다. 플랫폼은 이를 다른 비즈니스의 요청에서
   `source: "platform"`의 근거로 전파해서는 안 된다(MUST NOT). 특정 비즈니스에
   묶이지 않은 더 넓은 범위의 구매자 선호는 비즈니스별 결정과 독립적이며, 각
   비즈니스가 알린 선택지에 그 자체의 근거로 적용될 수 있다.

7. **유지와 번복 가능성.** 플랫폼은 같은 비즈니스와의 상호작용 전반에 걸쳐 구매자의
   이전 선호(`source: "platform"`)를 유지할 수 있으며(MAY), 변경되지 않은 값을 다시
   제시하지 않을 수 있다(MAY). 선호를 유지하는 경우 플랫폼은 구매자가 이를 변경할 수
   있는 수단을 제공해야 한다(SHOULD).
