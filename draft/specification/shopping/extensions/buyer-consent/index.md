# 구매자 동의 확장

## 개요

구매자 동의(Buyer Consent) 확장을 사용하면 비즈니스는 제공하는 동의 옵션을 알리고, 플랫폼은 데이터 사용과 커뮤니케이션 선호에 관한 구매자의 동의 결정을 전달할 수 있습니다. 대상 목적에는 마케팅, 분석, 선호 설정, 데이터 판매 또는 공유 등이 있습니다.

동의는 두 단계 구조로 모델링합니다.

- **목적(Purpose)** — 동의의 대상입니다. 최상위 수준에서 역DNS 식별자 (예: `dev.ucp.consent.marketing`)를 키로 사용합니다. 각 목적에는 `granted` 상태, 그 상태를 누가 지정했는지 나타내는 `source`, 사람이 읽을 수 있는 `description`, 선택 사항인 `links`가 있습니다.
- **세그먼트(Segment)** — 상위 목적의 범위를 특정 채널(예: 이메일, SMS), 벤더 (예: 측정 제공업체), 프로그램으로 좁히는 선택적 세분화 단위입니다. 각 세그먼트도 같은 형태(`granted`, `source`, `description`, 선택 사항인 `links`)이며, 해당 범위에서는 상위 목적보다 우선합니다.

중첩은 한 단계까지만 허용합니다. 목적 아래에는 세그먼트를 둘 수 있지만, 세그먼트 아래에는 더 중첩할 수 없습니다.

## 범위

이 확장은 비즈니스와 플랫폼이 구매자 동의 상태를 확정하고 주고받는 메커니즘을 정의합니다. 그 상태에 따라 각 당사자가 어떻게 행동해야 하는지는 정하지 않습니다. 비즈니스가 동의 상태에 따라 어떻게 행동할지(예: 이메일 발송, 데이터 공유), 플랫폼이 기존 결정을 구매자에게 어떻게 보여 줄지(예: 현재 구독 상태와 구독 해지 수단 표시)는 이 명세가 아니라 비즈니스 정책과 관련 규정에 따릅니다.

비즈니스는 동의 옵션을 알리기 전에, 적용되는 정책에 따라 초기 `granted` 값을 정할 책임이 있습니다. `source: "business"`로 알린 값은 비즈니스가 정한 권위 있는 기본값이며, 구매자는 플랫폼에서 이 값을 변경할 수 있습니다. 정책상의 판단은 여전히 비즈니스의 책임입니다.

## 디스커버리

비즈니스는 프로필에서 동의 지원 여부를 알립니다. 이 기능은 장바구니, 체크아웃 중 하나 또는 둘 다를 확장할 수 있습니다.

```json
{
  "capabilities": {
    "dev.ucp.shopping.buyer_consent": [
      {
        "version": "draft",
        "spec": "https://ucp.dev/draft/specification/shopping/extensions/buyer-consent",
        "schema": "https://ucp.dev/draft/schemas/shopping/buyer_consent.json",
        "extends": [
          "dev.ucp.shopping.cart",
          "dev.ucp.shopping.checkout"
        ]
      }
    ]
  }
}
```

## 스키마 구성

이 기능을 활성화하면 장바구니나 체크아웃(또는 둘 다)의 **buyer 객체**에 `consent` 필드가 추가됩니다.

- **장바구니 경로**: `cart.buyer.consent`
- **체크아웃 경로**: `checkout.buyer.consent`

`buyer` 객체는 모든 장바구니·체크아웃 작업, 즉 장바구니 `create` / `update`와 체크아웃 `create` / `update` / `complete`에서 선택 사항이며, 따라서 `consent`도 선택 사항입니다. 플랫폼은 수집한 동의를 `complete_checkout`에서 결제 정보와 함께 제출할 수 있습니다(MAY).

## 스키마 정의

### Consent 객체

목적별 동의 결정을 담은 맵입니다. 키는 `dev.ucp.consent.marketing`이나 `com.example.purpose.loyalty` 같은 역DNS 목적 식별자이고, 값은 [Consent Purpose](#consent-purpose) 객체입니다. 목적 안에는 채널, 벤더, 프로그램 범위별 [Consent Segment](#consent-segment) 세분화를 중첩할 수 있습니다.

### Consent Purpose

A buyer's consent decision for a purpose (e.g., marketing, analytics). Carries the current binary state, its source (business default or platform-captured buyer decision), human-readable context, and optional refinements scoping the decision to specific channels, vendors, or programs.

| 이름        | 타입                                                               | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                      |
| ----------- | ------------------------------------------------------------------ | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| granted     | boolean                                                            | **Required** | Whether consent has been granted for this purpose. The `source` field identifies who asserted this state (business default or platform-captured buyer preference).                                                                                                                                                                                        |
| source      | string                                                             | **Required** | Identifies the party that asserted the current `granted` value. `business` means the value reflects the business's default policy; `platform` means the value reflects an explicit buyer decision captured by the platform. **Enum:** `business`, `platform`                                                                                              |
| description | string                                                             | **Required** | Human-readable description of what the buyer is consenting to (e.g., 'Promotional communications across all channels').                                                                                                                                                                                                                                   |
| links       | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\] | Optional     | Optional links providing context (e.g., privacy policy, terms).                                                                                                                                                                                                                                                                                           |
| segments    | object                                                             | Optional     | Optional refinements scoping this purpose to specific channels, vendors, or programs. Keys are reverse-DNS identifiers. UCP currently defines two well-known segment identifiers under `dev.ucp.consent.marketing`: `dev.ucp.consent.marketing.email`, `dev.ucp.consent.marketing.sms`. Other segments follow vendor or merchant reverse-DNS conventions. |

### Consent Segment

A buyer's consent decision for a specific refinement of a parent purpose (e.g., email marketing under the marketing purpose). Overrides the parent's `granted` value for this scope. Segments do not nest further.

| 이름        | 타입                                                               | 필수 여부    | 설명                                                                                                                                                                                                                                                                          |
| ----------- | ------------------------------------------------------------------ | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| granted     | boolean                                                            | **Required** | Whether consent has been granted for this segment. Overrides the parent purpose's `granted` value for this specific scope.                                                                                                                                                    |
| source      | string                                                             | **Required** | Identifies the party that asserted the current `granted` value for this segment. `business` means the value reflects the business's default policy; `platform` means the value reflects an explicit buyer decision captured by the platform. **Enum:** `business`, `platform` |
| description | string                                                             | **Required** | Human-readable description of what the buyer is consenting to within this segment (e.g., 'Promotional emails and exclusive offers').                                                                                                                                          |
| links       | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\] | Optional     | Optional segment-specific links (e.g., channel terms or privacy disclosures).                                                                                                                                                                                                 |

## 잘 알려진 목적

이 확장은 `dev.ucp.consent.*` 네임스페이스 아래에 잘 알려진 목적 식별자 네 가지를 정의합니다. 비즈니스는 직접 관리하는 역DNS 네임스페이스 아래에 목적을 추가로 정의할 수 있습니다. 이 식별자는 운영상의 정의일 뿐 법적 분류 체계가 아닙니다.

| 식별자                            | 목적                                    |
| --------------------------------- | --------------------------------------- |
| `dev.ucp.consent.analytics`       | 분석 및 성능 측정                       |
| `dev.ucp.consent.marketing`       | 마케팅 커뮤니케이션                     |
| `dev.ucp.consent.preferences`     | 구매자 선호 기억                        |
| `dev.ucp.consent.sale_or_sharing` | 제3자에 대한 개인 데이터 판매 또는 공유 |

## 잘 알려진 세그먼트

이 확장은 `dev.ucp.consent.marketing` 목적에 쓸 잘 알려진 채널 세그먼트 식별자를 다음과 같이 정의합니다.

| 식별자                            | 채널   |
| --------------------------------- | ------ |
| `dev.ucp.consent.marketing.email` | 이메일 |
| `dev.ucp.consent.marketing.sms`   | SMS    |

## 알림과 확인

같은 형태의 맵으로 서로 보완하는 두 방향의 정보를 전달합니다.

| 방향                       | 채우는 주체 | 채우는 필드                                                        |
| -------------------------- | ----------- | ------------------------------------------------------------------ |
| **알림(Advertise)** (응답) | 비즈니스    | `granted`, `source`, `description`; 있는 경우 `links`와 `segments` |
| **확인(Confirm)** (요청)   | 플랫폼      | `granted`, `source`; 있는 경우 `segments`                          |

`description`과 `links`는 응답에만 쓰며, 플랫폼은 확인할 때 이 값을 되돌려 보내지 않습니다. `granted`와 `source`는 양방향 모두 필수입니다. 비즈니스는 알릴 때, 플랫폼은 확인할 때 각각 현재 상태와 그 출처를 보냅니다.

`source` 필드는 현재 `granted` 값을 누가 정했는지 나타냅니다.

- `source: "business"`는 값이 비즈니스의 기본값이며, 적용되는 구매자 선호가 없다는 뜻입니다.
- `source: "platform"`은 값이 플랫폼이 수집한 구매자의 명시적 선호라는 뜻입니다.

플랫폼은 source를 보고 현재 값을 어떻게 다룰지 판단합니다. `source: "business"`는 구매자에게 선택지를 제시해 참여를 유도하라는 신호이고, `source: "platform"`은 기록된 구매자 선호가 있다는 뜻입니다.

### 예시: 목적과 세그먼트

가장 단순한 경우는 목적 수준에서 동의를 수집하는 것입니다. 이 예시의 `analytics`는 구매자의 동의가 상위 목적에 기록된 경우입니다. 더 세밀하게 제어하려면 목적 아래에 세그먼트를 둘 수도 있습니다. `marketing`은 꺼져 있지만 구매자는 SMS 마케팅에 동의했습니다. 해당 범위에서는 세그먼트의 `granted: true`가 상위의 `granted: false`보다 우선합니다(더 구체적인 값이 우선).

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

구매자가 직접 응답한 두 선택(분석 전반, SMS 마케팅 개별)은 모두 `source: "platform"`입니다. 나머지 값(상위 `marketing`, `email` 세그먼트)은 `source: "business"`입니다.

### 알림 예시

비즈니스는 제공하는 목적과 세그먼트를 현재 동의 상태(구매자의 이전 결정 또는 비즈니스 기본값)와 함께 알립니다.

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

이 예시에서 구매자는 이전에 프로모션 이메일 수신에 동의했고(`marketing.email`이 `source: "platform"`), 데이터 판매는 거부했습니다(`sale_or_sharing`이 `source: "platform"`). 나머지 선택은 모두 비즈니스가 지정한 기본값 그대로이므로, 플랫폼이 구매자에게 제시해 명시적으로 동의를 받을 후보입니다.

### 확인 예시

플랫폼은 이후의 `update_checkout`이나 `complete_checkout` 요청에서 알림받은 모든 목적과 세그먼트의 현재 상태를 제출합니다. 적용되는 구매자 선호가 없는 선택은 알림받은 `granted`와 `source`를 그대로 되돌려 보내고, 구매자 선호가 있는 선택은 `source: "platform"`과 구매자가 밝힌 `granted` 값으로 설정합니다.

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

이 예시에서 구매자는 이메일과 SMS 마케팅 수신에 동의했습니다. 나머지 선택은 알림받은 값 그대로 되돌려 보냈습니다. 이전에 플랫폼이 수집한 선택을 그대로 유지하는 `sale_or_sharing`도 여기에 포함됩니다.

## 데이터 의존성

일부 동의 상태는 추가 구매자 데이터나 체크아웃 데이터가 있어야 성립합니다. 예를 들어 SMS 마케팅 동의에는 구매자 전화번호가 필요합니다. 플랫폼이 의존 데이터 없이 동의 값을 확인하면, 비즈니스는 표준 [체크아웃 상태 수명 주기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#checkout-status-lifecycle)와 [에러 처리](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#error-handling) 메커니즘으로 누락 사실을 알립니다.

`create_cart`, `update_cart`, `create_checkout`, `update_checkout`에서 비즈니스는 누락된 의존 데이터를 `warning` 메시지로 알려, 플랫폼이 이후 작업에서 해당 데이터를 수집할 수 있게 해야 합니다(SHOULD). 이때 알린 동의 결정은 여전히 유효하며, 경고는 참고용입니다.

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

`complete_checkout`에서 비즈니스는 확인된 동의 결정의 데이터 의존성이 충족되지 않은 상태로 체크아웃을 `completed`로 전환해서는 안 됩니다(MUST NOT). 누락된 의존 데이터는 반드시 표준 [에러 처리](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#error-handling) 흐름으로 알려야 합니다(MUST).

## 규범적 요구 사항

1. **알린 설정을 사용합니다.** 비즈니스는 반드시 지원하는 동의 옵션 전체를 알려야 합니다(MUST). 어떤 선택지를 언제 구매자에게 제시할지는 플랫폼이 정합니다. 선택지를 제시할 때 플랫폼은 반드시 알림받은 `description`, `links`, `granted` 값, 목적/세그먼트 묶음을 사용해야 합니다(MUST). `source` 필드는 플랫폼이 판단할 때 참고하는 정보이며([알림과 확인](#advertise-and-confirm) 참고) 사용자에게 보여 주는 콘텐츠가 아닙니다. `dev.ucp.consent.*` 네임스페이스 밖의 식별자는 불투명한 핸들이므로, 플랫폼은 경로만 보고 의미를 추론해서는 안 됩니다(MUST NOT). UCP가 정의한 식별자의 의미는 이 명세를 따릅니다.
1. **확인의 의미.** 요청에서 `consent` 필드는 선택 사항입니다. 이 필드를 생략하면 동의 업데이트가 없는 것으로 보고, 비즈니스는 이전 상태를 유지합니다. `consent`를 제출할 때 플랫폼은 반드시 알림받은 모든 목적과 세그먼트 키를 포함하고, 키마다 `granted`와 `source`를 모두 담아야 합니다(MUST). 제출한 `consent` 맵에서 알림받은 키를 빼는 방식으로 값을 표현해서는 안 됩니다(MUST NOT).
1. **알린 집합이 기준입니다.** 이전 응답에서 알리지 않은 목적과 세그먼트가 요청에 들어 있으면 비즈니스는 반드시 무시해야 합니다(MUST). 플랫폼은 비즈니스가 알리지 않은 목적이나 세그먼트를 구매자에게 묻거나 전송해서는 안 됩니다(MUST NOT).
1. **더 구체적인 값이 우선합니다.** 알린 세그먼트에서는 세그먼트의 `granted` 값이 해당 세그먼트에 관한 상위 목적의 `granted`보다 우선합니다.
1. **source를 platform으로 지정하려면 구매자가 밝힌 선호가 있어야 합니다.** 플랫폼은 `granted` 값이 구매자가 밝힌 선호를 반영할 때만 `source: "platform"`으로 설정해야 합니다(MUST). 적용되는 구매자 선호가 없는 선택은 반드시 알림받은 `source`를 바꾸지 않고 그대로 되돌려 보내야 합니다(MUST). 무엇을 구매자가 밝힌 선호로 볼지는 프로토콜이 정하지 않으며, 플랫폼이 적용되는 정책에 따라 판단합니다.
1. **비즈니스별 귀속.** 특정 비즈니스와 상호작용하며 수집한 동의 결정은 그 비즈니스에만 귀속됩니다. 플랫폼은 이 결정을 다른 비즈니스에 보내는 요청에서 `source: "platform"`의 근거로 삼아서는 안 됩니다(MUST NOT). 특정 비즈니스에 묶이지 않은 더 넓은 범위의 구매자 선호는 비즈니스별 결정과 별개이며, 각 비즈니스가 알린 선택지에 독자적인 근거로 적용할 수 있습니다.
1. **유지와 번복 가능성.** 플랫폼은 같은 비즈니스와 여러 번 상호작용하는 동안 구매자의 이전 선호(`source: "platform"`)를 유지할 수 있고(MAY), 바뀌지 않은 값은 다시 제시하지 않을 수 있습니다(MAY). 선호를 유지한다면 플랫폼은 구매자가 이를 변경할 수 있는 수단을 제공해야 합니다(SHOULD).
