# 예약 기능

초안 - 작업 중

이 기능은 **초안(draft) 상태**입니다. 데이터 모델, 프로토콜 바인딩, 운영 흐름이 활발히 개선되고 있으며, 명세가 발전하면서 호환성을 깨는 변경이 생길 수 있으므로 구현할 때 주의하세요.

- **기능 이름:** `dev.ucp.lodging.booking`

## 개요

숙박 예약 기능을 사용하면 플랫폼은 비즈니스와의 숙박 예약 세션을 처음부터 끝까지 지원하고 관리할 수 있습니다.

비즈니스는 계속 기록상 판매자(MoR, Merchant of Record)로 남으며, 이 기능으로 카드 결제를 받으려고 PCI DSS 인증을 받을 필요는 없습니다. AP2 위임(Mandate) 확장을 지원하지 않는 한, 예약은 사용자가 신뢰할 수 있는 UI에서 직접 확정해야 합니다.

### 흐름 개요

예약은 단계적으로 진행되는 세션 수명 주기를 따릅니다.

1. **세션 시작**: 플랫폼은 상위 퍼널 검색에서 찾은 숙소, 숙박, 요금제, 숙박 날짜 정보로 예약 세션을 시작합니다.
1. **단계적 정보 보강**: 플랫폼은 하나 이상의 작업에 걸쳐 투숙객 프로필, 숙박별 투숙객 배정, 예약자 정보, 결제 정보로 세션을 업데이트합니다.
1. **세션 완료**: 플랫폼은 예약을 확정해 변경할 수 없는 확정 예약을 만듭니다.

```text
        +------------+                         +---------------------+
        | incomplete |<----------------------->| requires_escalation |
        +-----+------+                         |   (user handoff     |
              |                                |   via continue_url) |
              | all info collected             +----------+----------+
              v                                           |
     +------------------+                                 |
     |ready_for_complete|                                 |
     |                  |                                 |
     | (platform can    |                                 | continue_url
     |  call Complete   |                                 |
     | Booking Session) |                                 |
     +--------+---------+                                 |
              |                                           |
              | Complete Booking                          |
              v                                           |
    +--------------------+                                |
    |complete_in_progress|                                |
    +---------+----------+                                |
              |                                           |
              +-----------------------+-------------------+
                                      v
                                +-------------+
                                |  completed  |
                                +-------------+

                                +-------------+
                                |  canceled   |
                                +-------------+
           (session invalid/expired - can occur from any state)
```

## 핵심 개념

- **복합 숙박 바인딩(`stay`)**: 숙박 예약은 하나 이상의 숙박 단위로 구성됩니다. `stay`는 물리적 객실 유형(`accommodation_type`), 상업적 요금 계약(`rate_plan`), 숙박 날짜(`stay_dates`), 투숙 인원 요건(`occupancy`), 투숙객 배정 (`guest_assignments`)을 연결하는 복합 바인딩입니다.
- **플랫폼이 생성하는 투숙객 식별자(`guest.id`)**: 비즈니스 범위의 카탈로그·숙소·숙박 식별자와 달리, 투숙객 식별자는 플랫폼이 자신의 네임스페이스 안에서 생성·할당· 관리합니다. 비즈니스는 `guest.id`를 안정적이고 불투명한 참조로 취급합니다. 투숙객 식별자는 개별 예약 세션 범위로 엄격히 한정된 불투명 문자열입니다(예: `"gst_01"`, `"gst_02"`). 플랫폼은 여러 판매자에 걸쳐 사용자를 지속적으로 추적하는 식별자를 사용하거나 `guest.id`에 개인 식별 정보(PII)를 노출해서는 안 됩니다(**MUST NOT**). 비즈니스는 `guest.id`를 근거로 서로 다른 예약 세션 사이의 신원을 추론하거나 연결해서는 안 됩니다(**MUST NOT**).
  - **시점**: 플랫폼은 예약이 `ready_for_complete`에 이르기 전에는 `id` 외의 `guests[]` 신원 필드를 보내지 않아야 하며(**SHOULD NOT**), 비즈니스가 `messages[]`로 요청한 필드만 보내야 합니다(**SHOULD**).
- **투숙객 풀과 숙박 배정 모델**: 투숙객 데이터는 2단계 관계형 모델로 구성됩니다.
  - **루트 투숙객 풀(`guests[]`)**: 예약 전체와 관련된 모든 개별 투숙객 프로필을 담은 평면 컬렉션입니다.
  - **숙박 배정(`stays[].guest_assignments[]`)**: `guest_id`로 특정 숙박 단위와 루트 풀의 투숙객을 연결하고 투숙 역할(`primary`, `accompanying` 등)을 지정하는 세부 매핑입니다.
- **예약자와 투숙객의 분리**: 데이터 모델은 법적 구매자와 실제 숙소 투숙객을 엄격히 구분합니다.
  - **`booker`**: 결제, 연락 의무, 예약 소유권을 책임지는 법적 계약 당사자입니다.
  - **`guests`**: 실제로 숙소에 투숙할 개인입니다. 예약자를 루트 풀의 투숙객으로도 등록할 수 있지만(**MAY**), 법인 예약, 대리 예약, 다중 숙박 예약을 지원하기 위해 두 엔티티는 분리해 둡니다.
- **잠정적 디스커버리와 권위 있는 예약**:
  - *디스커버리 단계(잠정)*: 검색, 견적, 객실 조회 응답은 검색 파라미터를 기반으로 잠정 요금, 이용 가능한 객실 유형, 정책 요약을 제공합니다.
  - *예약 세션(권위)*: 예약 세션을 생성하면 잠정적 디스커버리에서 권위 있는 상태로 전환됩니다. 비즈니스는 실시간 재고 가용성을 검증하고(또는 자체 정책에 따라 임시 소프트 홀드를 설정하고), 구속력 있는 요금 규칙을 확정하고, 객실 단위의 수용 인원 한도를 적용하고, 권위 있는 합계(`totals[]`)를 계산하고, 구속력 있는 취소 조건(`policies[]`)을 첨부합니다. 확정 예약은 Complete Booking Session으로 세션이 확정되기 전까지 생성되지 않습니다.

### 가격 범위

숙박 예약은 소비자 보호 규정(FTC, EU 가격 투명성 지침 등)을 준수하기 위해 엄격한 총액 표시(all-in pricing) 규칙을 따릅니다. 숙박에서는 한 번의 숙박에 드는 전체 금전적 부담이 예약 확정 시점에 선결제하는 요금과, 체크인 또는 체크아웃 때 숙소가 직접 징수하는 요금(예: 리조트 요금, 지방 숙박세, 남은 숙박 잔액)으로 나뉘는 경우가 많습니다.

#### 가격 구조와 범위 지침

- **권위 있는 루트 합계(`totals`)**: 최상위 `totals` 배열은 요청된 모든 숙박 단위를 포함한 예약 전체의 구속력 있고 권위 있는 가격 내역과 총 금전적 부담을 나타냅니다.
- **숙박 단위 합계(`stays[].totals`)**: `stays[].totals`의 각 항목은 해당 숙박 단위의 전체 숙박 기간(`stay_dates`, 즉 체크인부터 체크아웃까지)의 총 요금을 나타내며, 1박 기준 금액이 **아닙니다**.
- **항목별 소계와 1박별 내역(`lines`)**: 합계 항목 아래의 `lines` 배열은 보조적인 항목별 상세 내역을 제공합니다.
  - `subtotal` 합계 항목에는 1박별 숙박 요금 내역을 나타내는 `lines`를 둘 수 있습니다(**MAY**).
  - `tax` 합계 항목에는 과세 주체별 내역(예: 주 판매세와 지방 숙박세 또는 관광세)을 구분하는 `lines`를 둘 수 있습니다(**MAY**).
  - `fee` 합계 항목에는 필수 요금(예: 1일 리조트 요금, 청소비)을 자세히 나타내는 `lines`를 둘 수 있습니다(**MAY**).
- **가격 투명성과 전체 포함 비용**: 플랫폼과 비즈니스는 반드시 예약 확정 전에 투숙객에게 숙박에 따른 전체 부담액을 보여 줘야 합니다(**MUST**). 숨겨진 요금이나 고지되지 않은 숙소 요금은 가격 투명성 기준을 위반합니다.

#### 결제 시점과 조건(`dev.ucp.common.payment.terms`)

- **권위 있는 숙박 부담액(`total`)**: 핵심 `totals.json` 계약에 따라 표준 `type: "total"` 항목은 요청된 모든 단위를 포함한 예약 전체의 권위 있는 총 숙박 부담액만 나타냅니다. 모든 예약 세션은 반드시 `total` 항목을 정확히 하나 포함해야 합니다(**MUST**). 그래야 플랫폼이 관련 소비자 가격 표시 법규에 따라 숙박의 전체 비용을 처음부터 투명하게 표시할 수 있습니다.
- **가격 내역과 결제 시점**: 숙박 예약에서 가격 내역과 결제 시점은 분리되어 있습니다.
  - `totals[]`는 **가격 내역**(`subtotal`, `fee`, `tax`, 선택적 `discount`)을 정의합니다. total이 아닌 모든 항목은 더하면 `total`이 되는 가격 구성 요소입니다 (`sum(non-total entries) == total`).
  - `payment.terms[]`(`dev.ucp.common.payment.terms`)는 **결제 시점**과 자금 이동 일정을 정의합니다.
- **일정과 `totals`의 정합성**:
  - 결제 조건은 하나 이상의 `schedules[]`로 구성됩니다.
  - 선택된 조건에 속한 모든 `schedules[].amount`의 합은 반드시 `totals[].type: "total"`과 같아야 합니다(**MUST**)(핵심 `common/payment_terms.json` 불변 조건 충족).
  - `type: "immediate"`인 일정은 예약 완료 시점에 결제할 금액을 나타냅니다.
  - `type: "deferred"`인 일정은 지정된 미래 날짜나 사건 시점에 결제할 금액을 나타냅니다(예: 체크인 때 결제하는 잔액, `description` 또는 `due_at`에 명시된 보증금 결제 예정일).
  - `type: "at_property"`인 일정은 숙소가 직접 징수하는 금액을 나타냅니다(예: 도착· 출발 시 내는 지방 관광세나 필수 리조트 요금).
- **결제 조건과 요금제**: 숙박 유통에서 즉시 결제 할인이나 선결제 혜택은 보통 같은 요금제의 대체 결제 조건이 아니라 별도의 요금제(예: 환불 불가 사전 구매 요금과 유연한 최저 가용 요금)로 모델링합니다. 한 요금제에 유연한 결제 조건(보증금과 잔액 등)을 제공한다면 비즈니스는 해당하는 일정 내역을 `payment.terms[]`에 제공합니다.

#### 현지 세금과 요금 고지

필수 세금이나 요금을 숙소가 현지에서 징수하며 예약 시점에 납부할 수 없는 경우, 비즈니스는 `presentation: "disclosure"`이고 `path`가 해당 결제 일정(예: `$.payment.terms[0].schedules[1]`)을 직접 가리키는 경고 메시지를 `messages[]`에 제공해야 합니다(**SHOULD**). 이 고지 안내에는 반드시 적용되는 현지 세율, 면제 조건, 결제 방법을 명시해야 하며(**MUST**), `links[]`의 공식 정책 링크로 보완합니다.

다음 예시는 대표적인 숙박 가격 패턴에서 `totals[]`가 결제 조건과 함께 어떻게 구성되는지 보여 줍니다.

도쿄 호텔 3박 숙박으로, 기본 객실 요금(`64,000`)과 선결제 서비스 요금(`6,400`)은 즉시 청구되고, 필수 리조트 요금(`6,000`)과 도쿄 숙박세(`1,200`)는 숙소에서 징수합니다. `totals[]`에서 total이 아닌 구성 요소의 합은 총 숙박 비용(`77,600`)과 정확히 같습니다.

```json
[
  {
    "type": "subtotal",
    "display_text": "Room Rate (3 nights for 1 room, includes 10% consumption tax)",
    "amount": 64000
  },
  {
    "type": "fee",
    "display_text": "Service Fee (10% prepaid)",
    "amount": 6400
  },
  {
    "type": "fee",
    "display_text": "Resort & Facility Amenity Fee (Pay at hotel, ¥2,000/night)",
    "amount": 6000
  },
  {
    "type": "tax",
    "display_text": "Tokyo Accommodation Tax (Pay at hotel, ~¥200/guest/night)",
    "amount": 1200
  },
  {
    "type": "total",
    "display_text": "Total Stay Cost",
    "amount": 77600
  }
]
```

보증금·잔액 조건으로 3박을 숙박하며, `subtotal.lines`에 1박별 내역이 들어 있습니다. 선결제 보증금은 첫날 숙박 요금(`40,000`)과 서비스 요금(`3,000`), 첫 세금(`4,000`)을 포함하고, 남은 잔액(`88,000`)은 체크인 때까지 미뤄집니다. `totals[]`에서 모든 숙박 요금, 요금, 세금을 합하면 전체 숙박 부담액(`135,000`)이 됩니다.

```json
[
  {
    "type": "subtotal",
    "display_text": "Room Rate (3 nights @ $400)",
    "amount": 120000,
    "lines": [
      {
        "display_text": "Night 1 (Deposit): Sep 1, 2026",
        "amount": 40000
      },
      {
        "display_text": "Night 2: Sep 2, 2026",
        "amount": 40000
      },
      {
        "display_text": "Night 3: Sep 3, 2026",
        "amount": 40000
      }
    ]
  },
  {
    "type": "fee",
    "display_text": "Service Fee",
    "amount": 3000
  },
  {
    "type": "tax",
    "display_text": "State Lodging Tax (10%)",
    "amount": 12000
  },
  {
    "type": "total",
    "display_text": "Total Stay Cost",
    "amount": 135000
  }
]
```

> [!TIP] 숙박 바인딩, 대표 투숙객 배정, 메시지, 결제 조건을 모두 포함한 전체 예약 세션 페이로드는 [가격 및 결제 조건 예시](#pricing-examples)를 참고하세요.

### 결제

결제 핸들러는 `/.well-known/ucp`에 있는 비즈니스의 UCP 프로필에서 찾습니다. 핸들러는 결제 수단(예: Google Pay, Shop Pay)을 수집하기 위한 처리 명세를 정의합니다. 사용자가 결제 정보를 제출하면 플랫폼은 수집한 결제 수단 데이터로 `payment.instruments` 배열을 채웁니다.

`payment` 객체는 예약 세션 생성과 수정 작업에서는 선택 사항입니다. 완료 (`complete_booking_session`) 시에는 구속력 있는 결제 합의를 확정해야 하므로 `payment`가 \*\*필수(REQUIRED)\*\*입니다.

- **즉시 카드 결제**: 즉시 청구나 선결제 카드 보증이라면 수집한 결제 수단 데이터로 `payment.instruments`를 채웁니다.
- **결제 조건 확인**: Complete는 이미 선택된 결제 조건을 확인할 뿐, 최종 조건을 전송하지 **않습니다**. 조건 선택은 세션이 `ready_for_complete`에 이르기 전에 가격과 파생 상태를 바꿀 수 있는 Update(`update_booking_session`)로 이루어지므로, Complete에서는 `selected_term_id`를 의도적으로 생략합니다.
- **결제 수단 불필요**: 결제 수단이 필요 없는 경우(예: 선결제 카드 보증이 필요 없는 후불 또는 현장 결제 예약)에는 빈 결제 객체(`payment: {}`)로 완료 요건을 충족합니다.

### 예약 상태 수명 주기

예약 `status` 필드는 세션의 현재 단계를 나타내며, 다음에 어떤 조치가 필요한지 결정합니다. 상태는 비즈니스가 설정하고, 플랫폼은 진행에 무엇이 필요한지 알려 주는 메시지를 받습니다.

#### 상태 값

- **`incomplete`**: 예약 세션에 필수 정보가 없거나 해결해야 할 문제가 있습니다. 플랫폼은 `messages` 배열에서 맥락을 확인하고 Update Booking Session으로 해결을 시도해야 합니다.
- **`requires_escalation`**: 예약 세션에 API로 제공할 수 없는 정보나 사용자 입력이 필요합니다. 플랫폼은 `messages`를 확인해 무엇이 필요한지 파악해야 합니다. `recoverable` 오류가 있으면 먼저 해결한 다음 `continue_url`로 사용자에게 넘깁니다.
- **`ready_for_complete`**: 예약 세션에 필요한 정보(확정된 가격 합계, 유효한 숙박 날짜, 필요한 경우 수집된 결제 수단, `booker` 또는 대표 투숙객 배정 (`role: "primary"`)을 통한 대표 투숙객 식별, 차단 중인 모든 미처리 액션의 해결)가 모두 있으며, 플랫폼이 프로그래밍 방식으로 확정할 수 있습니다. 플랫폼은 Complete Booking Session을 호출할 수 있습니다.
- **`complete_in_progress`**: 비즈니스가 Complete Booking 요청을 처리하고 있습니다. 응답에는 `confirmation` 필드가 있어서는 안 됩니다(**MUST NOT**). 허용되는 작업은 [수락된 완료](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#accepted-completion)를 참고하세요.
- **`completed`**: 예약이 성공적으로 확정되었습니다.
- **`canceled`**: 예약 세션이 유효하지 않거나 만료되었습니다. 플랫폼은 필요하면 새 예약 세션을 시작해야 합니다.

#### 세션 만료와 재고 관리

- **일시적인 세션 수명(`expires_at`)**: 예약 세션은 확정 전 단계에서 진행 중인 리소스입니다. 비즈니스는 세션 상태, 제시된 가격, 임시 재고 홀드가 얼마나 오래 유효한지 나타내는 `expires_at` 타임스탬프를 제공할 수 있습니다(**MAY**).
- **재고 홀드 전략**: 비즈니스는 `expires_at` 기간 동안 재고에 임시 소프트 홀드를 걸 수도 있고(**MAY**), 세션이 업데이트될 때마다 재고 가용성을 실시간으로 확인하고 Complete Booking Session 시점에 최종 배정을 수행하는 낙관적 동시성 방식을 사용할 수도 있습니다(**MAY**).
- **해제 방식**: 예약 흐름이 중단되면 플랫폼은 Cancel Booking Session을 호출해 홀드된 재고와 세션 상태를 명시적으로 해제할 수 있습니다(**MAY**). 그렇지 않으면 홀드된 리소스는 `expires_at`이 지나면 자동으로 해제됩니다.

### 액션

활성화된 기능이나 확장에 예약 세션의 미처리 추가 인증(step-up) 작업(PSD2 / 3D Secure 강력한 고객 인증, 생체 인증 추가 확인, 신원 확인 등)이 있으면, 비즈니스는 응답 전용 `actions` 맵에 그 인스턴스를 노출합니다. 공통 규칙은 [개요 — 액션](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#actions)에 정의되어 있으며, 이 섹션에서는 예약 상태 수명 주기에서 액션을 어떻게 해석하는지 설명합니다.

모든 액션은 해당 액션 유형에 지정된 효과를 차단합니다. `incomplete` 상태의 액션은 비즈니스가 `ready_for_complete`를 반환하기 전에 완료되어야 하는 작업을 나타낼 수 있습니다. 액션 유형 계약에 따라 액션을 처리한 뒤(예: `url_redirect`로 구매자를 리디렉션) 플랫폼은 Get Booking Session이나 이후의 Update Booking Session으로 최신 예약 상태를 가져와야 합니다(**SHOULD**).

액션 때문에 Complete Booking Session을 수락할 수 없다면, 비즈니스는 반드시 현재 예약 세션을 `status: incomplete`(사용자 인계가 필요하면 `status: requires_escalation`)로 반환하고, `path`가 해당 액션 인스턴스를 정확히 가리키는 `severity: "recoverable"` 오류 메시지를 함께 반환해야 합니다(**MUST**).

### 오류 처리

`messages` 배열에는 예약 상태와 관련된 오류, 경고, 안내 메시지가 들어 있습니다. `ucp.status`는 응답 형태를 구분하는 값으로, `"success"`는 응답에 기대한 페이로드가 있다는 뜻이고 `"error"`는 그 대신 오류 정보가 있다는 뜻입니다. 각 오류 메시지의 `severity` 필드는 권장 조치를 지정합니다.

| 심각도                  | 의미                                        | 플랫폼 조치                                             |
| ----------------------- | ------------------------------------------- | ------------------------------------------------------- |
| `recoverable`           | 플랫폼이 API로 입력을 수정해 해결할 수 있음 | 리소스를 업데이트하고 재시도                            |
| `requires_buyer_input`  | 비즈니스가 API로 받을 수 없는 입력을 요구함 | `continue_url`로 인계                                   |
| `requires_buyer_review` | 사용자의 검토와 승인이 필요함               | `continue_url`로 인계                                   |
| `unrecoverable`         | 조치할 리소스가 없음                        | 새 리소스나 입력으로 재시도하거나 `continue_url`로 인계 |

`requires_*` 심각도의 오류는 `status: requires_escalation`으로 이어집니다. 둘 다 사용자 인계로 끝나지만 서로 다른 예약 세션 상태를 나타냅니다.

- `requires_buyer_input`은 예약 세션이 **미완성**이라는 뜻입니다. 비즈니스에 필요한 정보를 그 API로는 프로그래밍 방식으로 수집할 수 없습니다.
- `requires_buyer_review`는 예약 세션이 **완성**되었지만, 정책·규제·자격 규칙상 완료 전에 사용자 승인이 필요하다는 뜻입니다.

#### 표준 오류

| 코드                          | 설명                                                  |
| ----------------------------- | ----------------------------------------------------- |
| `inventory_exhausted`         | 선택한 객실 단위나 재고 홀드를 더 이상 이용할 수 없음 |
| `occupancy_exceeded_capacity` | 배정된 투숙객 수가 물리적 수용 한도를 초과함          |
| `payment_failed`              | 결제 처리 실패                                        |
| `eligibility_invalid`         | 완료 시점에 자격 주장을 검증하지 못함                 |

### 경고 표시

경고 메시지의 `presentation` 필드는 플랫폼이 반드시 따라야 하는(**MUST**) 렌더링 계약(예: `notice`와 `disclosure`)을 결정합니다. 권위 있는 렌더링 규칙은 [체크아웃 — 경고 표시](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#warning-presentation)를 참고하세요.

#### 합계 변경

진행 중인 예약 세션에서 숙박 요금, 세금, 필수 요금이 바뀌면 비즈니스는 `totals[]`를 다시 계산한 세션을 반환하고, 반드시 `code: "totals_changed"`와 `path: "$.totals"`가 설정된 `messages[]` 경고 메시지로 그 변경을 알려야 합니다(**MUST**).

구매자가 변경된 금액을 확실히 인지해야 한다면 비즈니스는 반드시 `presentation: "disclosure"`를 설정해야 합니다(**MUST**). 플랫폼은 반드시 업데이트된 합계와 경고 내용을 구매자에게 보여 줘야 하며(**MUST**), 고지를 자동으로 닫거나 숨겨서는 안 됩니다(**MUST NOT**).

## Continue URL

`continue_url` 필드를 사용하면 예약을 플랫폼에서 비즈니스 UI로 넘겨, 사용자가 예약 세션을 이어서 진행하고 확정할 수 있습니다.

### 제공 여부

비즈니스는 `status` = `requires_escalation`을 반환할 때 반드시 `continue_url`을 제공해야 합니다(**MUST**). 그 밖의 모든 비종료 상태(`incomplete`, `ready_for_complete`, `complete_in_progress`)에서는 `continue_url`을 제공해야 합니다(**SHOULD**). 종료 상태(`completed`, `canceled`)에서는 `continue_url`을 생략해야 합니다(**SHOULD**).

## 가이드라인

### 플랫폼

- 예약 세션을 생성할 때는 반드시 유효한 `property.id`와 함께, 상위 퍼널 디스커버리 메커니즘에서 얻은 미리 구성된 `stay.id` 또는 `accommodation_type.id`와 `rate_plan.id` 식별자 둘 다를 제공해야 합니다(**MUST**).
- 반드시 각 `stays[]` 항목을 하나의 단위로 표시해야 하며(**MUST**) (`accommodation_type.title`, `rate_plan.title`, `rate_plan.description`(있는 경우), `occupancy`, 그리고 합계 렌더링 계약에 따른 해당 항목의 `totals`), `accommodation_type`이나 `rate_plan`이 같은 항목을 병합하거나 중복 제거해서는 안 됩니다(**MUST NOT**).
- Complete Booking Session을 호출하기 전에 반드시 `booker` 정보를 제공하거나 적어도 한 명의 투숙객을 `role: "primary"`(법적 성명 전체와 연락처 포함)로 지정해 대표 투숙객을 식별해야 합니다(**MUST**). 대표 투숙객이 식별되지 않은 상태로 `ready_for_complete`를 받은 플랫폼은 Complete Booking Session을 호출해서는 안 되며 (**MUST NOT**), Update Booking Session으로 세션을 바로잡거나 가능하면 `continue_url`로 에스컬레이션해야 합니다(**SHOULD**).
- 반드시 루트 `guests[]` 배열의 각 항목마다 플랫폼 네임스페이스 안에서 고유하고 안정적이며 세션 범위로 한정된 문자열 식별자(예: `"gst_01"`, `"gst_02"`)를 생성해야 합니다(**MUST**).
- 반드시 모든 `guest_assignments[].guest_id`가 루트 `guests[]` 풀에 있는 유효한 `id`를 참조하도록 보장해야 합니다(**MUST**).
- 예약이 `ready_for_complete`에 이르기 전에는 `id` 외의 `guests[]` 개인 신원 필드를 보내지 않아야 하며(**SHOULD NOT**), 비즈니스가 `messages[]`로 요청한 필드만 보내야 합니다(**SHOULD**).
- 예약 세션 진행을 돕는 데 에이전트를 활용할 수 있습니다(**MAY**)(예: 숙박과 날짜 선택, 투숙객 정보 수집). 다만 에이전트는 사용자가 예약 정보를 검토하고 예약을 완료할 수 있도록 신뢰할 수 있고 결정적인 UI에 예약 세션을 넘겨야 합니다.
- 신뢰할 수 있고 결정적인 UI에서 언제든 사용자를 에이전트로 되돌려 보낼 수 있습니다(**MAY**).
- 플랫폼이 에이전트가 보낸 요청임을 표시하는 경우 에이전트 맥락을 제공할 수 있습니다(**MAY**).
- 예약 상태가 `requires_escalation`이면 반드시 `continue_url`을 사용해야 합니다(**MUST**).
- 그 밖의 상황에서도 `continue_url`을 사용해 비즈니스 UI로 인계할 수 있습니다(**MAY**).
- 인계할 때는 비즈니스가 제공한 `continue_url`을 우선해야 합니다(**SHOULD**).

### 비즈니스

- 반드시 요청된 `stay.id` 또는 복합 `accommodation_type.id`·`rate_plan.id` 바인딩을 실시간 가용성과 재고 제약에 비추어 평가하고, 권위 있는 메타데이터, 가격 합계, 정책 조건을 반환해야 합니다(**MUST**).
- 반드시 `rate_plan.title`로 요금제를 구별할 수 있게 하고, `rate_plan.description`이 있다면 두 필드만으로 구매자가 `policies[]`까지 읽지 않아도 주요 상업 조건을 이해할 수 있을 만큼 충분히 작성해야 합니다(**MUST**). 두 필드는 `policies[]`와 모순되어서는 안 됩니다(**MUST NOT**).
- 반드시 플랫폼이 제공한 `guest.id` 식별자를 세션 업데이트와 응답 전반에서 재매핑, 이름 변경, 수정 없이 그대로 유지해야 합니다(**MUST**).
- 반드시 모든 `stays[].guest_assignments[].guest_id` 참조가 루트 `guests[]` 배열의 기존 항목과 일치하는지 검증해야 합니다(**MUST**).
- 식별된 대표자 없이 `ready_for_complete`, `complete_in_progress`, `completed` 상태의 예약 세션을 반환해서는 안 됩니다(**MUST NOT**). 대표자란 법적 성명 전체와 연락 수단을 갖춘 `booker`, 법적 성명 전체와 연락 수단을 갖추고 어떤 `stays[].guest_assignments[]`에서 `primary`로 배정된 `guests[]`의 투숙객, 또는 법인·대리 예약의 경우 연락 수단을 제공하는 `booker`와 함께 이름이 지정된 대표 투숙객을 말합니다. 비즈니스에 필요한 추가 신원 필드(여권 번호, 국적, 생년월일 등)는 반드시 비즈니스별로 `messages[]`로 협상해야 합니다(**MUST**).
- 반드시 배정된 전체 투숙 인원과 투숙객 나이를 기준으로 물리적 `capacity` 한도를 적용해야 합니다(**MUST**).
  - 투숙객 나이는 숙박 체크인 날짜 기준 만 나이로 평가합니다.
  - 나이가 정의된 `child_age_ranges[].ages` 구간에 속하는 투숙객은 어린이로 분류하고, 어떤 어린이 구간에도 속하지 않는 투숙객은 성인으로 분류합니다.
  - 요청된 투숙 인원은 반드시 `occupancy.adults <= capacity.adults`, `occupancy.children <= capacity.children`, `occupancy.total <= capacity.total`을 충족해야 합니다(**MUST**).
  - `limit`이 있는 `child_age_ranges[]`의 각 항목마다, 나이가 그 구간에 속하는 어린이 수는 `limit`을 초과해서는 안 됩니다(**MUST NOT**).
- `booker`나 대표 투숙객 정보에 유효한 이메일 주소가 있으면, 반드시 예약 완료 후 확인 이메일을 보내야 합니다(**MUST**).
- 정확한 오류 및 경고 메시지를 제공해야 합니다(**SHOULD**).
- 예약 세션을 처리하는 로직은 반드시 결정적이어야 합니다(**MUST**).
- `status` = `requires_escalation`을 반환할 때는 반드시 `continue_url`을 제공해야 합니다(**MUST**).
- `status` = `requires_escalation`을 반환할 때는 반드시 `severity`가 `requires_buyer_input` 또는 `requires_buyer_review`인 메시지를 하나 이상 포함해야 합니다(**MUST**).
- 모든 비종료 예약 응답에 `continue_url`을 제공해야 합니다(**SHOULD**).
- 예약 세션이 "completed" 상태에 이르면 변경할 수 없는 것으로 간주합니다.

## 기능 스키마 정의

| 이름           | 타입                                                                                                    | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| -------------- | ------------------------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp            | [UCP Booking Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-booking-schema) | **필수**  | 예약 응답용 UCP 메타데이터.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| id             | string                                                                                                  | **필수**  | 예약의 고유 식별자.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| status         | string                                                                                                  | **필수**  | 현재 단계와 필요한 작업을 나타내는 예약 상태. 상태 전이에 대한 자세한 내용은 Booking Status 생명주기 문서를 참고하세요. **허용 값:** `incomplete`, `requires_escalation`, `ready_for_complete`, `complete_in_progress`, `completed`, `canceled`                                                                                                                                                                                                                                                                                                                                                                       |
| property       | [Property](/ucp-ko/draft/draft/specification/reference/#property)                                       | **필수**  | 숙박 시설이 위치한 실제 시설 또는 지리적 위치(예: 호텔, 리조트, 빌라 단지, 캐빈 파크).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| stays          | Array\[[Stay Response](/ucp-ko/draft/draft/specification/reference/#stay)\]                             | **필수**  | 예약의 숙박 라인 아이템 목록.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| booker         | [Booker](/ucp-ko/draft/draft/specification/reference/#booker)                                           | 선택      | 예약하는 법적 계약 당사자 정보.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| guests         | Array\[[Guest](/ucp-ko/draft/draft/specification/reference/#guest)\]                                    | 선택      | 예약에 연결된 투숙객 프로필 목록.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| travel_purpose | string                                                                                                  | 선택      | 플랫폼이 제공하는 여행 또는 예약의 목적입니다. 잘 알려진 값: `business`, `leisure`. 플랫폼은 다른 값을 보낼 수 있으며(MAY), 비즈니스는 인식하지 못하는 값을 반드시 무시하고 목적이 지정되지 않은 것으로 처리해야 합니다(MUST).                                                                                                                                                                                                                                                                                                                                                                                        |
| context        | [Context](/ucp-ko/draft/draft/specification/reference/#context)                                         | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals        | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                                         | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |
| currency       | string                                                                                                  | **필수**  | 비즈니스가 판단한 시장을 반영하는 ISO 4217 통화 코드. 주소, 컨텍스트, geo IP로 결정합니다. 구매자는 신호를 제공하고 통화는 비즈니스가 결정합니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| totals         | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                           | **필수**  | 예약의 권위 있는 가격 내역. `total` 항목은 요청한 모든 단위에 걸친 권위 있는 총 숙박 비용을 나타냅니다. total이 아닌 항목은 모두 합산하면 `total`이 되는 가격 가산 항목입니다. 결제 시점이 여러 일정으로 나뉘거나 숙박 시설에서 직접 징수하는 요금이 있으면, 시점별 내역은 `totals`가 아니라 결제 조건(`payment.terms[]`) 일정으로 전달합니다.                                                                                                                                                                                                                                                                        |
| actions        | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                         | 선택      | 이 예약에 대해 확장이 정의한 미해결 Action.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| messages       | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                | 선택      | 예약 세션 상태에 대한 오류·정보 메시지 목록.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| links          | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]                                      | **필수**  | 플랫폼이 표시할 링크(개인정보처리방침, 이용약관). 법적 준수를 위해 필수입니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| policies       | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                  | 선택      | 예약 세션에 적용되는 정책(예: 취소 조건). `applies_to` 대상은 응답 루트를 기준으로 하며, 없거나 비어 있으면 `links[]`의 URL을 참조합니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| expires_at     | string                                                                                                  | 선택      | 이 시점 이후로 예약 세션이 더 이상 유효하지 않은 RFC 3339 타임스탬프입니다. 세션 상태, 견적 가격, 임시 재고 보류가 얼마나 오래 유효한지 나타냅니다. 비즈니스는 업데이트된 합계를 먼저 플랫폼에 반환하지 않고 변경된 가격으로 예약을 완료해서는 안 됩니다(MUST NOT).                                                                                                                                                                                                                                                                                                                                                   |
| continue_url   | string                                                                                                  | 선택      | 예약 핸드오프와 세션 복구용 URL. status가 requires_escalation이면 반드시 제공해야 합니다(MUST). 형식과 가용성 요구 사항은 명세를 참고하세요.                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| payment        | [Payment](/ucp-ko/draft/draft/specification/reference/#payment)                                         | 선택      | 핸들러를 포함한 결제 구성.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| confirmation   | [Booking Confirmation](/ucp-ko/draft/draft/specification/reference/#booking-confirmation)               | 선택      | 이 세션에서 생성된 예약의 세부 정보.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |

## 작업

예약 기능은 다음과 같은 논리적 작업을 정의합니다.

| 작업                         | 설명                                                                      |
| ---------------------------- | ------------------------------------------------------------------------- |
| **Create Booking Session**   | 새 예약 세션을 시작합니다. 사용자가 예약 의사를 표시하는 즉시 호출합니다. |
| **Get Booking Session**      | 예약 세션의 현재 상태를 조회합니다.                                       |
| **Update Booking Session**   | 리소스 전체를 교체하는 방식으로 예약 세션을 수정합니다.                   |
| **Complete Booking Session** | 예약을 확정하고 예약 내역을 확인합니다.                                   |
| **Cancel Booking Session**   | 예약 세션을 취소합니다.                                                   |

### 예약 세션 생성

사용자가 예약 의사를 표시하면 플랫폼이 상위 퍼널의 숙소, 숙박, 숙박 날짜 파라미터로 세션을 시작할 때 호출합니다.

**Inputs**

| 이름           | 타입                                                                 | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| -------------- | -------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| property       | [Property](/ucp-ko/draft/draft/specification/reference/#property)    | **필수**  | 숙박 시설이 위치한 실제 시설 또는 지리적 위치(예: 호텔, 리조트, 빌라 단지, 캐빈 파크).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| stays          | Array\[[Stay](/ucp-ko/draft/draft/specification/reference/#stay)\]   | **필수**  | 예약의 숙박 라인 아이템 목록.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| booker         | [Booker](/ucp-ko/draft/draft/specification/reference/#booker)        | 선택      | 예약하는 법적 계약 당사자 정보.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| guests         | Array\[[Guest](/ucp-ko/draft/draft/specification/reference/#guest)\] | 선택      | 예약에 연결된 투숙객 프로필 목록.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| travel_purpose | string                                                               | 선택      | 플랫폼이 제공하는 여행 또는 예약의 목적입니다. 잘 알려진 값: `business`, `leisure`. 플랫폼은 다른 값을 보낼 수 있으며(MAY), 비즈니스는 인식하지 못하는 값을 반드시 무시하고 목적이 지정되지 않은 것으로 처리해야 합니다(MUST).                                                                                                                                                                                                                                                                                                                                                                                        |
| context        | [Context](/ucp-ko/draft/draft/specification/reference/#context)      | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals        | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)      | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |
| payment        | [Payment](/ucp-ko/draft/draft/specification/reference/#payment)      | 선택      | 핸들러를 포함한 결제 구성.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |

**Output**

This object MUST be one of the following types: [Booking](/ucp-ko/draft/draft/specification/lodging/booking/#booking), [Error Response](/ucp-ko/draft/draft/specification/reference/#error-response).

### 예약 세션 조회

예약 세션 리소스의 최신 상태를 조회합니다.

**Inputs**

| 이름 | 타입   | 필수 여부 | 설명                                  |
| ---- | ------ | --------- | ------------------------------------- |
| id   | string | **필수**  | 예약의 고유 식별자.경로에 정의됩니다. |

**Output**

This object MUST be one of the following types: [Booking](/ucp-ko/draft/draft/specification/lodging/booking/#booking), [Error Response](/ucp-ko/draft/draft/specification/reference/#error-response).

### 예약 세션 수정

예약 세션 리소스 전체를 교체합니다. 플랫폼은 반드시 변경된 데이터(예: 투숙객 프로필, 투숙객 배정, 예약자 정보)를 포함한 전체 예약 상태를 보내야 합니다(**REQUIRED**).

**Inputs**

| 이름           | 타입                                                                 | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| -------------- | -------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id             | string                                                               | **필수**  | 예약의 고유 식별자.경로에 정의됩니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| property       | [Property](/ucp-ko/draft/draft/specification/reference/#property)    | **필수**  | 숙박 시설이 위치한 실제 시설 또는 지리적 위치(예: 호텔, 리조트, 빌라 단지, 캐빈 파크).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| stays          | Array\[[Stay](/ucp-ko/draft/draft/specification/reference/#stay)\]   | **필수**  | 예약의 숙박 라인 아이템 목록.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| booker         | [Booker](/ucp-ko/draft/draft/specification/reference/#booker)        | 선택      | 예약하는 법적 계약 당사자 정보.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| guests         | Array\[[Guest](/ucp-ko/draft/draft/specification/reference/#guest)\] | 선택      | 예약에 연결된 투숙객 프로필 목록.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| travel_purpose | string                                                               | 선택      | 플랫폼이 제공하는 여행 또는 예약의 목적입니다. 잘 알려진 값: `business`, `leisure`. 플랫폼은 다른 값을 보낼 수 있으며(MAY), 비즈니스는 인식하지 못하는 값을 반드시 무시하고 목적이 지정되지 않은 것으로 처리해야 합니다(MUST).                                                                                                                                                                                                                                                                                                                                                                                        |
| context        | [Context](/ucp-ko/draft/draft/specification/reference/#context)      | 선택      | 관련성 판단과 현지화를 위한 잠정적 구매자 신호로, 권위 있는 데이터가 아닙니다. 검증된 입력(예: 배송 주소)이 없으면 비즈니스는 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 판단과 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터로 수행해야 합니다(MUST). 컨텍스트는 개인을 식별할 수 없어야 하며(SHOULD), 단계적으로 공개할 수 있습니다. 초기에는 대략적인 신호를, 세션이 진행되면서 더 세밀한 정보를 제공합니다. 더 세밀한 데이터(배송 주소, 청구 주소)가 컨텍스트보다 우선합니다. |
| signals        | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)      | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST).                                                                                                                                                                                                                                                               |
| payment        | [Payment](/ucp-ko/draft/draft/specification/reference/#payment)      | 선택      | 핸들러를 포함한 결제 구성.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |

**Output**

This object MUST be one of the following types: [Booking](/ucp-ko/draft/draft/specification/lodging/booking/#booking), [Error Response](/ucp-ko/draft/draft/specification/reference/#error-response).

### 예약 세션 완료

예약을 최종 접수하는 호출입니다. 결제 정보가 수집되고 사용자가 예약 확정을 결정했을 때 호출합니다.

**Inputs**

| 이름    | 타입                                                            | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ------- | --------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id      | string                                                          | **필수**  | 예약의 고유 식별자.경로에 정의됩니다.                                                                                                                                                                                                                                                                                                                   |
| signals | [Signals](/ucp-ko/draft/draft/specification/reference/#signals) | 선택      | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| payment | [Payment](/ucp-ko/draft/draft/specification/reference/#payment) | **필수**  | 핸들러를 포함한 결제 구성.                                                                                                                                                                                                                                                                                                                              |

**Output**

This object MUST be one of the following types: [Booking](/ucp-ko/draft/draft/specification/lodging/booking/#booking), [Error Response](/ucp-ko/draft/draft/specification/reference/#error-response).

### 예약 세션 취소

완료 전에 진행 중인 예약 세션을 취소합니다.

**Inputs**

| 이름 | 타입   | 필수 여부 | 설명                                  |
| ---- | ------ | --------- | ------------------------------------- |
| id   | string | **필수**  | 예약의 고유 식별자.경로에 정의됩니다. |

**Output**

This object MUST be one of the following types: [Booking](/ucp-ko/draft/draft/specification/lodging/booking/#booking), [Error Response](/ucp-ko/draft/draft/specification/reference/#error-response).

## 전송 바인딩

위의 추상 작업은 다음과 같은 구체적인 전송 프로토콜에 바인딩됩니다.

- [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/lodging/booking/rest/index.md): 표준 HTTP 메서드와 JSON 페이로드를 사용하는 RESTful API 매핑
- [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/lodging/booking/mcp/index.md): 에이전트 상호작용을 위한 Model Context Protocol 매핑

## 예시

### 가격 예시

다음 예시는 숙소 징수 요금과 유연한 결제 조건을 통합해 보여 주는 완전하고 권위 있는 예약 세션입니다.

도쿄 호텔 3박 예약의 전체 예시로, 기본 객실 요금과 서비스 요금은 즉시 선결제하고 도쿄 숙박세와 리조트 요금은 체크인 때 숙소가 직접 징수합니다. 결제 시점은 `dev.ucp.common.payment.terms` 기능의 `payment.terms[]`로 명시적으로 모델링하며, `messages[]`의 세금 고지 경고는 숙소 징수 일정을 참조합니다.

```json
{
  "ucp": {
    "version": "draft",
    "capabilities": {
      "dev.ucp.lodging.booking": [
        {
          "version": "draft"
        }
      ],
      "dev.ucp.common.payment.terms": [
        {
          "version": "draft"
        }
      ]
    },
    "payment_handlers": {
      "com.example.card_handler": [
        {
          "id": "card_handler",
          "version": "draft",
          "available_instruments": [
            {
              "type": "card"
            }
          ]
        }
      ]
    }
  },
  "id": "bks_example_01",
  "status": "incomplete",
  "currency": "JPY",
  "property": {
    "id": "pp_ginza_hotel",
    "name": "Ginza Grand Hotel"
  },
  "stays": [
    {
      "id": "stay_dlx_king_std",
      "accommodation_type": {
        "id": "at_dlx_king",
        "title": "Deluxe King Room"
      },
      "rate_plan": {
        "id": "rp_standard",
        "title": "Standard Flexible Rate"
      },
      "occupancy": {
        "adults": 2,
        "total": 2
      },
      "stay_dates": {
        "start_date": "2026-10-01",
        "end_date": "2026-10-04"
      },
      "totals": [
        {
          "type": "subtotal",
          "amount": 64000
        },
        {
          "type": "total",
          "amount": 64000
        }
      ]
    }
  ],
  "totals": [
    {
      "type": "subtotal",
      "display_text": "Room Rate (3 nights for 1 room, includes 10% consumption tax)",
      "amount": 64000
    },
    {
      "type": "fee",
      "display_text": "Service Fee (10% prepaid)",
      "amount": 6400
    },
    {
      "type": "fee",
      "display_text": "Resort & Facility Amenity Fee (Pay at hotel, ¥2,000/night)",
      "amount": 6000
    },
    {
      "type": "tax",
      "display_text": "Tokyo Accommodation Tax (Pay at hotel, ~¥200/guest/night)",
      "amount": 1200
    },
    {
      "type": "total",
      "display_text": "Total Stay Cost",
      "amount": 77600
    }
  ],
  "messages": [
    {
      "type": "warning",
      "code": "local_tax",
      "path": "$.payment.terms[0].schedules[1]",
      "presentation": "disclosure",
      "content": "**Tokyo Accommodation Tax Notice**: In accordance with Tokyo Metropolitan Government regulations, a local accommodation tax of JPY 200 per guest per night applies to room rates of JPY 15,000 or higher. This tax is collected directly by the property upon check-in.",
      "content_type": "markdown",
      "url": "https://hotel.example.com/policies/tokyo-accommodation-tax"
    }
  ],
  "links": [
    {
      "type": "tax_policy",
      "title": "Tokyo Local Accommodation Tax Schedule",
      "url": "https://hotel.example.com/policies/tokyo-accommodation-tax"
    },
    {
      "type": "terms_of_service",
      "title": "Hotel Booking Terms & Conditions",
      "url": "https://hotel.example.com/terms"
    },
    {
      "type": "refund_policy",
      "title": "Cancellation and Refund Policy",
      "url": "https://hotel.example.com/cancellation-policy"
    }
  ],
  "payment": {
    "selected_term_id": "pt_standard",
    "terms": [
      {
        "id": "pt_standard",
        "title": "Standard Settlement",
        "description": {
          "plain": "Pay room rate and service fee today; mandatory resort fee and Tokyo Accommodation Tax are paid at check-in."
        },
        "schedules": [
          {
            "id": "sched_room",
            "type": "immediate",
            "description": {
              "plain": "Due today when you book (Room rate + service fee)."
            },
            "amount": 70400
          },
          {
            "id": "sched_property",
            "type": "at_property",
            "description": {
              "plain": "Due at check-in on October 1, 2026 (Resort fee + Tokyo Accommodation Tax)."
            },
            "amount": 7200
          }
        ]
      }
    ]
  }
}
```

보증금·잔액 결제 조건(`pt_deposit_balance`)을 제공하는 호텔 3박 예약의 전체 예시입니다. 첫날 숙박 요금과 선결제 요금, 첫 세금은 즉시 청구하고, 나머지 2박의 숙박 요금과 남은 세금은 체크인 때까지 미룹니다.

```json
{
  "ucp": {
    "version": "draft",
    "capabilities": {
      "dev.ucp.lodging.booking": [
        {
          "version": "draft"
        }
      ],
      "dev.ucp.common.payment.terms": [
        {
          "version": "draft"
        }
      ]
    },
    "payment_handlers": {
      "com.example.card_handler": [
        {
          "id": "card_handler",
          "version": "draft",
          "available_instruments": [
            {
              "type": "card"
            }
          ]
        }
      ]
    }
  },
  "id": "bks_example_02",
  "status": "incomplete",
  "currency": "USD",
  "property": {
    "id": "pp_grand_hotel",
    "name": "Grand Hotel"
  },
  "stays": [
    {
      "id": "stay_king_std",
      "accommodation_type": {
        "id": "at_king",
        "title": "King Room"
      },
      "rate_plan": {
        "id": "rp_flex",
        "title": "Flexible Rate"
      },
      "occupancy": {
        "adults": 2,
        "total": 2
      },
      "stay_dates": {
        "start_date": "2026-09-01",
        "end_date": "2026-09-04"
      },
      "totals": [
        {
          "type": "subtotal",
          "amount": 120000
        },
        {
          "type": "total",
          "amount": 120000
        }
      ]
    }
  ],
  "totals": [
    {
      "type": "subtotal",
      "display_text": "Room Rate (3 nights @ $400)",
      "amount": 120000,
      "lines": [
        {
          "display_text": "Night 1 (Deposit): Sep 1, 2026",
          "amount": 40000
        },
        {
          "display_text": "Night 2: Sep 2, 2026",
          "amount": 40000
        },
        {
          "display_text": "Night 3: Sep 3, 2026",
          "amount": 40000
        }
      ]
    },
    {
      "type": "fee",
      "display_text": "Service Fee (Prepaid)",
      "amount": 3000
    },
    {
      "type": "tax",
      "display_text": "State Lodging Tax (10%)",
      "amount": 12000
    },
    {
      "type": "total",
      "display_text": "Total Stay Cost",
      "amount": 135000
    }
  ],
  "links": [
    {
      "type": "terms_of_service",
      "title": "Terms of Service",
      "url": "https://example.com/tos"
    }
  ],
  "payment": {
    "selected_term_id": "pt_deposit_balance",
    "terms": [
      {
        "id": "pt_deposit_balance",
        "title": "First night now, balance at check-in",
        "description": {
          "plain": "Hold your room with one night's rate plus initial taxes & fees."
        },
        "schedules": [
          {
            "id": "sched_first_night",
            "type": "immediate",
            "description": {
              "plain": "Due today when you book (Deposit: First night + service fee + tax)."
            },
            "amount": 47000
          },
          {
            "id": "sched_balance",
            "type": "deferred",
            "description": {
              "plain": "Due at check-in on September 1, 2026 at 3:00 PM PDT (Remaining 2 nights + remaining tax)."
            },
            "due_at": "2026-09-01T15:00:00-07:00",
            "amount": 88000
          }
        ]
      }
    ]
  }
}
```

## 엔티티

### 액션

예약 세션을 진행하는 데 필요한 추가 인증(step-up) 액션 지시(예: 3D Secure / PSD2 구매자 리디렉션이나 신원 확인).

확장이 정의한 미해결 Action 인스턴스. 키는 확장 이름이 아니라 역 도메인 형식의 Action 유형입니다.

### 숙소

숙박 시설이 있는 물리적 시설이나 지리적 위치(예: 호텔, 리조트, 빌라 단지, 캐빈 파크).

| 이름    | 타입                                                                          | 필수 여부 | 설명                                                                           |
| ------- | ----------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------ |
| id      | string                                                                        | **필수**  | 비즈니스 범위의 안정적이고 불투명한 위치 식별자.                               |
| name    | string                                                                        | **필수**  | 구매자에게 표시되는, 비즈니스가 소유한 표시 이름.                              |
| address | [Postal Address](/ucp-ko/draft/draft/specification/reference/#postal-address) | 선택      | 장소의 실제 주소.                                                              |
| media   | Array\[[Media](/ucp-ko/draft/draft/specification/reference/#media)\]          | 선택      | 숙소 미디어(이미지, 동영상, 3D 모델). 첫 번째 항목은 표시용 대표 미디어입니다. |

### 예약 확인

예약이 성공적으로 완료되면 반환되는 확인 및 예약 조회 정보.

| 이름          | 타입   | 필수 여부 | 설명                                                                                                                       |
| ------------- | ------ | --------- | -------------------------------------------------------------------------------------------------------------------------- |
| id            | string | **필수**  | 고유 예약 식별자.                                                                                                          |
| label         | string | 선택      | 고유 id와 다를 경우 예약을 식별하기 위한 사람이 읽을 수 있는 라벨(예: 확인 번호). 반드시 비즈니스만 제공해야 합니다(MUST). |
| pincode       | string | 선택      | 예약에 안전하게 접근하기 위해 제공되는 추가 코드. 고유 식별자나 확인 번호와 함께 사용해야 합니다(SHOULD).                  |
| permalink_url | string | 선택      | 비즈니스 웹사이트에서 예약에 바로 접근하는 퍼머링크.                                                                       |

### 예약자

예약을 하는 법적 계약 당사자이자 주 연락 창구.

| 이름         | 타입                                                                          | 필수 여부 | 설명                                                                                     |
| ------------ | ----------------------------------------------------------------------------- | --------- | ---------------------------------------------------------------------------------------- |
| first_name   | string                                                                        | 선택      | 예약자의 이름.                                                                           |
| last_name    | string                                                                        | 선택      | 예약자의 성.                                                                             |
| email        | string                                                                        | 선택      | 예약자의 이메일 주소.                                                                    |
| phone_number | string                                                                        | 선택      | E.164 표준. 예약자의 전화번호.                                                           |
| birthdate    | string                                                                        | 선택      | ISO 8601 형식의 예약자 생년월일.                                                         |
| company      | string                                                                        | 선택      | 예약자의 회사 또는 조직 이름으로, 법인 청구서 발행에 사용됩니다.                         |
| address      | [Postal Address](/ucp-ko/draft/draft/specification/reference/#postal-address) | 선택      | 법적 계약, 규제 준수, 인보이스 발행에 사용하는 예약자 또는 회사의 실제 주소나 등록 주소. |

### 수용 인원

물리적 객실 유형이 지원하는 투숙 인원 한도와 어린이 나이 기준.

| 이름             | 타입          | 필수 여부 | 설명                                                                                                                              |
| ---------------- | ------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------- |
| adults           | integer       | 선택      | 최대 성인 투숙 인원.                                                                                                              |
| children         | integer       | 선택      | 최대 아동 투숙 인원.                                                                                                              |
| child_age_ranges | Array[object] | 선택      | 어린이 나이 구간과 구간별 수용 인원 한도(선택). 나이가 구간에 속하는 투숙객은 어린이로, 그렇지 않으면 성인으로 계산합니다.        |
| total            | integer       | 선택      | 최대 총 투숙 인원(성인 또는 아동). adults 및/또는 children이 있으면 이 값은 반드시 max(adults, children) 이상이어야 합니다(MUST). |

### 컨텍스트

구매자 위치와 시장 맥락에 관한 힌트.

| 이름            | 타입                                                                                             | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| --------------- | ------------------------------------------------------------------------------------------------ | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| address_country | string                                                                                           | 선택      | 2자리 ISO 3166-1 alpha-2 코드로 표기한 국가(예: "US"). 3자리 alpha-3 코드나 전체 국가명을 사용할 수도 있습니다(MAY).                                                                                                                                                                                                                                                                                                                                           |
| address_region  | string                                                                                           | 선택      | 국가 안의 1차 행정 구역(예: California 같은 주 또는 도).                                                                                                                                                                                                                                                                                                                                                                                                       |
| postal_code     | string                                                                                           | 선택      | 우편번호(예: "94043").                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| location        | string                                                                                           | 선택      | 비즈니스 네임스페이스 안에서 위치(Location)를 가리키는 안정적이고 불투명한 식별자. 이 값은 잠정적이고 구속력 없는 힌트이며 구매자의 지역 정보와는 별개입니다. 효과는 작업 명세나 활성화된 기능/확장이 정의합니다. 소매 쇼핑의 흔한 예로, 사용자가 식료품을 구매할 때 선택해 저장한 기본 단골 매장 ID가 있습니다.                                                                                                                                               |
| intent          | string                                                                                           | 선택      | 구매자의 의도를 설명하는 배경 맥락(예: 'looking for a gift under $50', 'need something durable for outdoor use'). 관련성, 추천, 개인화에 활용됩니다.                                                                                                                                                                                                                                                                                                           |
| language        | string                                                                                           | 선택      | 콘텐츠의 선호 언어입니다. IETF BCP 47 언어 태그(예: 'en', 'fr-CA', 'zh-Hans')를 사용합니다. REST에서는 Accept-Language 헤더에 해당합니다. 이 필드가 없으면 플랫폼은 Accept-Language로 대체해야 하며(SHOULD), 값이 있으면 Accept-Language보다 우선합니다. 해당 언어를 제공할 수 없으면 비즈니스는 다른 언어로 콘텐츠를 반환할 수 있습니다(MAY).                                                                                                                 |
| currency        | string                                                                                           | 선택      | 선호 통화(ISO 4217, 예: 'EUR', 'USD'). 비즈니스는 맥락과 신뢰할 수 있는 신호로 표시 통화를 결정하며, 다중 통화 시장에서는 이 힌트를 선택에 참고할 수 있습니다(MAY). 가격 필터 값의 통화 단위로도 쓰이므로, 플랫폼은 가격 필터를 보낼 때 이 필드를 포함해야 합니다(SHOULD). 응답 가격에는 결정된 통화를 확인할 수 있도록 통화가 명시됩니다.                                                                                                                     |
| eligibility     | Array\[[Reverse Domain Name](/ucp-ko/draft/draft/specification/reference/#reverse-domain-name)\] | 선택      | 로열티 멤버십, 결제 수단 혜택 등 적격 혜택에 대한 구매자 주장. 인식된 주장은 비즈니스 응답에 반영될 수 있습니다(MAY)(예: 회원 전용 상품 구매 가능 여부, 카탈로그의 조정 가격, 장바구니나 체크아웃의 잠정 할인). 비즈니스는 인식하지 못한 값을 오류 없이 반드시 무시해야 합니다(MUST). 값은 반드시 역 도메인 이름 규칙을 사용해야 하며(MUST)(예: 'com.example.loyalty_gold', 'org.school.student'), 반드시 개인을 식별할 수 없어야 합니다(MUST).                |
| payment         | Array[object]                                                                                    | 선택      | 구매자가 선호하는 결제 핸들러를 우선순위 순서(가장 선호하는 것이 먼저)로 나열합니다. 각 항목은 비즈니스 프로필의 `ucp.payment_handlers`에 공개된 핸들러를 가리키며, 선호하는 결제 수단 유형으로 범위를 좁힐 수도 있습니다. 비즈니스는 이를 사용해 핸들러(유형이 주어지면 유형까지)를 미리 선택하거나 우선순위를 정해야 하며(SHOULD), 사용할 수 없거나 자격이 없는 항목은 무시할 수 있습니다(MAY). 인식하지 못하는 값은 오류 없이 반드시 무시해야 합니다(MUST). |

### 날짜 구간

숙박의 체크인(`start_date`)과 체크아웃(`end_date`) 날짜 범위.

| 이름       | 타입   | 필수 여부 | 설명                                                                                |
| ---------- | ------ | --------- | ----------------------------------------------------------------------------------- |
| start_date | string | **필수**  | ISO 8601 형식의 기간 시작일.                                                        |
| end_date   | string | **필수**  | ISO 8601 형식의 기간 종료일. 반드시 `start_date`와 같거나 그 이후여야 합니다(MUST). |

### 투숙객

개별 투숙객 프로필. `id`는 예약 세션 안에서 투숙객을 고유하게 식별할 수 있도록 플랫폼이 생성해 제공합니다.

| 이름         | 타입    | 필수 여부 | 설명                                                                                  |
| ------------ | ------- | --------- | ------------------------------------------------------------------------------------- |
| id           | string  | **필수**  | 플랫폼 네임스페이스 안에서 투숙객을 가리키는, 세션 범위의 안정적이고 불투명한 식별자. |
| first_name   | string  | 선택      | 투숙객의 이름.                                                                        |
| last_name    | string  | 선택      | 투숙객의 성.                                                                          |
| email        | string  | 선택      | 투숙객의 이메일.                                                                      |
| phone_number | string  | 선택      | E.164 표준. 투숙객의 전화번호.                                                        |
| age          | integer | 선택      | 체크인과 숙박 인원에 대한 연령 제한을 적용하는 데 쓰이는 투숙객 나이.                 |

### 투숙객 배정

`guest_id`로 숙박 단위와 루트 `guests[]` 풀의 투숙객을 연결하고 역할을 지정하는 관계 링크.

| 이름     | 타입   | 필수 여부 | 설명                                                                                                                                                                                                                                            |
| -------- | ------ | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| guest_id | string | **필수**  | 배정된 투숙객의 안정적이고 불투명한 식별자. 반드시 루트 `guests` 풀에서 플랫폼이 제공한 식별자와 일치해야 합니다(MUST).                                                                                                                         |
| role     | string | 선택      | 이번 숙박에서 투숙객의 역할입니다. 잘 알려진 값: `primary`(예약이 연결된 투숙객), `accompanying`(그 밖의 모든 투숙객). 플랫폼은 다른 값을 보낼 수 있으며(MAY), 비즈니스는 인식하지 못하는 값을 반드시 `accompanying`으로 처리해야 합니다(MUST). |

### 링크

규정 준수 및 법적 링크(예: 개인정보 처리방침, 서비스 약관).

| 이름  | 타입   | 필수 여부 | 설명                                                                                                                                                                                                                      |
| ----- | ------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type  | string | **필수**  | 링크 유형. 잘 알려진 값: `privacy_policy`, `terms_of_service`, `refund_policy`, `shipping_policy`, `faq`. 소비자는 알 수 없는 값을 `title` 필드로 표시하거나 링크를 생략하는 식으로 적절히 처리해야 합니다(SHOULD).       |
| url   | string | **필수**  | 표시할 콘텐츠를 가리키는 실제 URL.                                                                                                                                                                                        |
| title | string | 선택      | 링크의 표시 텍스트(선택 사항). 제공되면 type에서 생성하는 대신 이 값을 사용합니다.                                                                                                                                        |
| id    | string | 선택      | 연결된 리소스가 주소 지정 가능한 UCP 리소스(예: 상품 또는 변형)일 때의 식별자(선택 사항). 플랫폼은 이를 이용해 해당 리소스를 소유한 기능으로 작업할 수 있습니다. 정책 페이지처럼 UCP 식별자가 없는 리소스에는 생략합니다. |

### 오류 메시지

| 이름         | 타입                                                                  | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------ | --------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                | **필수**  | **상수 = error**. 메시지 유형 판별자.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| code         | [Error Code](/ucp-ko/draft/draft/specification/reference/#error-code) | **필수**  | 오류 유형을 식별하는 오류 코드. 표준 오류는 기능 명세에 정의되어 있고(예시 참고) 의미가 표준화되어 있으며, 자유 형식 코드도 허용합니다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| path         | string                                                                | 선택      | 메시지가 가리키는 구성 요소의 RFC 9535 JSONPath(예: $.line_items[0]).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| content_type | string                                                                | 선택      | 콘텐츠 형식. 기본값은 plain. **허용 값:** `plain`, `markdown`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| content      | string                                                                | **필수**  | 사람이 읽을 수 있는 메시지.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| severity     | string                                                                | **필수**  | 리소스 상태와 권장 조치를 나타냅니다. 'recoverable': 플랫폼이 입력을 수정하거나 관련 Action을 처리하는 등 대역 내에서 상황을 해결하고, 필요하면 새 작업을 제출할 수 있습니다. 'requires_buyer_input': 판매자의 API로는 프로그래밍 방식으로 수집할 수 없는 정보가 필요합니다(체크아웃 미완료). 'requires_buyer_review': 정책, 규제, 자격 규칙 때문에 주문 전에 구매자가 승인해야 합니다. 'unrecoverable': 조치할 유효한 리소스가 없으므로 새 리소스나 입력으로 재시도해야 합니다. 'requires\_*' 심각도의 오류는 'status: requires_escalation'의 원인이 됩니다.* *허용 값:*\* `recoverable`, `requires_buyer_input`, `requires_buyer_review`, `unrecoverable` |

### 안내 메시지

| 이름         | 타입                                                                | 필수 여부 | 설명                                                                                                                                           |
| ------------ | ------------------------------------------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                              | **필수**  | **상수 = info**. 메시지 유형 판별자.                                                                                                           |
| path         | string                                                              | 선택      | 메시지가 가리키는 구성 요소의 RFC 9535 JSONPath(예: $.line_items[0]).                                                                          |
| code         | [Info Code](/ucp-ko/draft/draft/specification/reference/#info-code) | 선택      | 정보 메시지의 유형을 식별하는 정보 코드. 표준 코드는 기능 명세에 정의되어 있고(예시 참고) 표준화된 의미를 가지며, 자유 형식 코드도 허용됩니다. |
| content_type | string                                                              | 선택      | 콘텐츠 형식. 기본값은 plain. **허용 값:** `plain`, `markdown`                                                                                  |
| content      | string                                                              | **필수**  | 사람이 읽을 수 있는 메시지.                                                                                                                    |

### 경고 메시지

| 이름         | 타입                                                                      | 필수 여부 | 설명                                                                                                                                                                                                                                                                            |
| ------------ | ------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                    | **필수**  | **상수 = warning**. 메시지 유형 판별자.                                                                                                                                                                                                                                         |
| path         | string                                                                    | 선택      | 메시지가 가리키는 구성 요소의 RFC 9535 JSONPath(예: $.line_items[0]).                                                                                                                                                                                                           |
| code         | [Warning Code](/ucp-ko/draft/draft/specification/reference/#warning-code) | **필수**  | 경고 유형을 식별하는 경고 코드. 표준 코드는 기능 명세에 정의되어 있고(예시 참고) 표준화된 의미를 지닙니다. 자유 형식 코드도 허용됩니다.                                                                                                                                         |
| content      | string                                                                    | **필수**  | 반드시 표시해야 하는(MUST) 사람이 읽을 수 있는 경고 메시지.                                                                                                                                                                                                                     |
| content_type | string                                                                    | 선택      | 콘텐츠 형식. 기본값은 plain. **허용 값:** `plain`, `markdown`                                                                                                                                                                                                                   |
| presentation | string                                                                    | 선택      | 이 경고의 렌더링 규약입니다. 'notice'(기본값): 플랫폼은 반드시 표시해야 하며(MUST), 닫을 수 있습니다(MAY). 'disclosure': 플랫폼은 반드시 path가 가리키는 구성 요소 가까이에 표시해야 하며(MUST), 숨기거나 자동으로 닫아서는 안 됩니다(MUST NOT). 전체 규약은 명세를 참고하세요. |
| image_url    | string                                                                    | 선택      | 필수 시각 요소(예: 경고 기호, 에너지 등급 라벨)의 URL.                                                                                                                                                                                                                          |
| url          | string                                                                    | 선택      | 추가 정보를 위한 참조 URL(예: 규제 기관 사이트, 등록부 항목, 정책 페이지).                                                                                                                                                                                                      |

### 투숙 인원

숙박에 요청된 성인과 어린이 투숙객 수 내역.

| 이름       | 타입           | 필수 여부 | 설명                                                                                                        |
| ---------- | -------------- | --------- | ----------------------------------------------------------------------------------------------------------- |
| adults     | integer        | 선택      | 숙박 유닛에 투숙하는 성인 수.                                                                               |
| children   | integer        | 선택      | 숙박 유닛에 투숙하는 아동 수.                                                                               |
| child_ages | Array[integer] | 선택      | 숙박 단위에 머무는 어린이들의 나이. 값이 있으면 length(child_ages)는 반드시 children과 같아야 합니다(MUST). |
| total      | integer        | **필수**  | 객실의 총 투숙 인원. adults와/또는 children 필드가 지정되면 반드시 adults + children과 같아야 합니다(MUST). |

### 결제

결제 정보와 수집된 결제 수단.

| 이름        | 타입                                                                                                                                                   | 필수 여부 | 설명                                                                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| instruments | Array\[[Payment Instrument Selected Payment Instrument](/ucp-ko/draft/draft/specification/reference/#payment-instrument-selected-payment-instrument)\] | 선택      | 이 결제에 사용할 수 있는 결제 수단. 각 결제 수단은 handler_id 필드로 특정 핸들러와 연결됩니다. 핸들러는 기본 payment_instrument 스키마를 확장해 핸들러별 필드를 추가할 수 있습니다. |

### 정책

예약 세션이나 숙박에 적용되는 정책(취소 조건, 숙소 이용 규칙 등). `applies_to`의 JSONPath 대상은 이 응답의 루트를 기준으로 합니다(예: `$.stays[0]`). 전체 모델은 [정책](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#policies)을 참고하세요.

| 이름        | 타입                                                                                    | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ----------- | --------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| type        | [Reverse Domain Name](/ucp-ko/draft/draft/specification/reference/#reverse-domain-name) | **필수**  | 정책 유형 판별자. 열린 역 DNS 어휘입니다. 잘 알려진 정책 유형의 레지스트리는 명세 문서를 참고하세요. 비즈니스는 자체 도메인에 사용자 정의 유형을 정의할 수 있습니다(MAY, 예: `com.example.policy.price_match`). 플랫폼은 알 수 없는 값을 반드시 허용해야 합니다(MUST).                                                                                                                                                                                                                                                                                                                                                   |
| description | [Description](/ucp-ko/draft/draft/specification/reference/#description)                 | **필수**  | 하나 이상의 형식(plain, markdown, html)으로 된 사람이 읽을 수 있는 정책 요약. 플랫폼이 유형별 필드를 이해하지 못해도 표시할 수 있도록 모든 정책에 필수입니다. 이것은 구매자 대상 고지가 아닙니다. 표시 의무는 `messages[]` 경고로 부과됩니다(정책 섹션 참고).                                                                                                                                                                                                                                                                                                                                                            |
| applies_to  | Array[string]                                                                           | 선택      | 이 정책이 적용되는 노드를 가리키는 RFC 9535 JSONPath 표현식으로, 응답을 포함하는 루트를 기준으로 합니다(예: 장바구니·체크아웃의 `$.line_items[0]`, 카탈로그의 `$.products[2]`). 각 대상은 지정한 노드와 그 아래에 중첩된 모든 것을 포함하므로, 상품을 대상으로 하면 그 변형도 포함됩니다. 단수 쿼리(RFC 9535 Section 2.3.5.1, 이름·인덱스 선택자만 사용)는 노드 하나를 지정하고, 필터·와일드카드·슬라이스는 노드 집합과 일치합니다. 생략하면 정책이 응답 전체에 적용됩니다. 같은 `type`의 정책들이 한 노드를 두고 경합하면 가장 좁은 대상이 이기고 나머지를 덮어씁니다. 구체성을 판정하는 방법은 정책 섹션을 참고하세요. |
| url         | string                                                                                  | 선택      | 전체 정책 문서 링크(선택 사항).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |

### 요금제

상업적 요금제 계약, 취소 정책 규칙, 요금 포함 사항. `id`는 상위 퍼널 검색에서 얻습니다.

| 이름        | 타입   | 필수 여부 | 설명                             |
| ----------- | ------ | --------- | -------------------------------- |
| id          | string | **필수**  | 요금제의 고유 식별자.            |
| title       | string | **필수**  | 사람이 읽을 수 있는 요금제 이름. |
| description | string | 선택      | 요금제에 대한 자세한 설명.       |

### 숙박

물리적 객실(`accommodation_type.id`), 상업적 요금 조건(`rate_plan.id`), 숙박 날짜(`stay_dates`), 투숙 인원, 투숙객 배정을 하나로 묶는 복합 바인딩.

| 이름               | 타입                                                                                       | 필수 여부 | 설명                                                                            |
| ------------------ | ------------------------------------------------------------------------------------------ | --------- | ------------------------------------------------------------------------------- |
| id                 | string                                                                                     | **필수**  | 숙박 바인딩의 안정적이고 불투명한 고유 식별자.                                  |
| stay_dates         | [Date Interval](/ucp-ko/draft/draft/specification/reference/#date-interval)                | **필수**  | 숙소 현지 시간 기준으로 이 객실의 숙박 기간(체크인 및 체크아웃 날짜).           |
| accommodation_type | [Accommodation Type](/ucp-ko/draft/draft/specification/reference/#accommodation-type)      | **필수**  | 임대 가능한 물리적 공간의 범주 또는 사양(예: room, apartment, villa, campsite). |
| rate_plan          | [Rate Plan](/ucp-ko/draft/draft/specification/reference/#rate-plan)                        | **필수**  | 상업적 요금 정책 계약.                                                          |
| occupancy          | [Occupancy](/ucp-ko/draft/draft/specification/reference/#occupancy)                        | 선택      | 이 숙박의 투숙 인원 구성과 투숙객 수.                                           |
| guest_assignments  | Array\[[Guest Assignment](/ucp-ko/draft/draft/specification/reference/#guest-assignment)\] | 선택      | 이 숙박의 투숙객 배정 목록.                                                     |
| totals             | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                              | **필수**  | 이 숙박의 합계 내역.                                                            |

### 객실 유형

임대 가능한 물리적 공간의 종류나 사양(예: 객실, 아파트, 빌라, 캠핑 사이트)과 수용 인원 한도.

| 이름        | 타입                                                                 | 필수 여부 | 설명                                                                                |
| ----------- | -------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------------- |
| id          | string                                                               | **필수**  | 객실 유형의 고유 식별자.                                                            |
| title       | string                                                               | **필수**  | 사람이 읽을 수 있는 객실 유형 제목.                                                 |
| description | string                                                               | 선택      | 객실 유형에 대한 자세한 설명.                                                       |
| capacity    | [Capacity](/ucp-ko/draft/draft/specification/reference/#capacity)    | 선택      | 객실 유형의 수용 인원 정보.                                                         |
| media       | Array\[[Media](/ucp-ko/draft/draft/specification/reference/#media)\] | 선택      | 숙박 유형 미디어(이미지, 동영상, 3D 모델). 첫 번째 항목이 표시용 대표 미디어입니다. |

### 신호

플랫폼이 제공하는 사기 방지 및 보안 맥락.

| 이름               | 타입   | 필수 여부 | 설명                                                     |
| ------------------ | ------ | --------- | -------------------------------------------------------- |
| dev.ucp.buyer_ip   | string | 선택      | 클라이언트의 IP 주소(IPv4 또는 IPv6).                    |
| dev.ucp.user_agent | string | 선택      | 클라이언트의 HTTP User-Agent 헤더 또는 그에 상응하는 값. |

### 합계

권위 있는 항목별 가격 구성 요소와 예약 총액.

| 이름         | 타입                                                                        | 필수 여부 | 설명                                                                                                                                                                                                                        |
| ------------ | --------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                      | **필수**  | 비용 범주. 잘 알려진 값: subtotal, items_discount, discount, fulfillment, tax, fee, total. 비즈니스는 추가 값을 사용할 수 있습니다(MAY).                                                                                    |
| display_text | string                                                                      | 선택      | 금액 옆에 표시할 텍스트. 적절한 방식을 반영해야 합니다(예: 'Shipping', 'Delivery').                                                                                                                                         |
| amount       | [Signed Amount](/ucp-ko/draft/draft/specification/reference/#signed-amount) | **필수**  | ISO 4217에 정의된 통화의 최소 단위로 나타낸 금액. 최소 단위와 기본 단위의 비율은 통화의 지수를 참고합니다(예: USD는 2, JPY는 0, KWD는 3). 음수일 수 있으며, 부호는 값 자체에 포함됩니다(예: 할인은 음수, 청구 금액은 양수). |
