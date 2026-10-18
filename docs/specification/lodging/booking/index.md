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

# 예약 기능 { #booking-capability }

!!! note "초안 - 작업 중"
    이 기능은 **초안(draft) 상태**입니다. 데이터 모델, 프로토콜 바인딩, 운영 흐름이
    활발히 개선되고 있으며, 명세가 발전하면서 호환성을 깨는 변경이 생길 수
    있으므로 구현할 때 주의하세요.

* **기능 이름:** `dev.ucp.lodging.booking`

## 개요 { #overview }

숙박 예약 기능을 사용하면 플랫폼은 비즈니스와의 숙박 예약 세션을 처음부터 끝까지
지원하고 관리할 수 있습니다.

비즈니스는 계속 기록상 판매자(MoR, Merchant of Record)로 남으며, 이 기능으로 카드
결제를 받으려고 PCI DSS 인증을 받을 필요는 없습니다. AP2 위임(Mandate) 확장을
지원하지 않는 한, 예약은 사용자가 신뢰할 수 있는 UI에서 직접 확정해야 합니다.

### 흐름 개요 { #flow-overview }

예약은 단계적으로 진행되는 세션 수명 주기를 따릅니다.

1. **세션 시작**: 플랫폼은 상위 퍼널 검색에서 찾은 숙소, 숙박, 요금제, 숙박 날짜
   정보로 예약 세션을 시작합니다.
2. **단계적 정보 보강**: 플랫폼은 하나 이상의 작업에 걸쳐 투숙객 프로필, 숙박별
   투숙객 배정, 예약자 정보, 결제 정보로 세션을 업데이트합니다.
3. **세션 완료**: 플랫폼은 예약을 확정해 변경할 수 없는 확정 예약을 만듭니다.

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

## 핵심 개념 { #key-concepts }

* **복합 숙박 바인딩(`stay`)**: 숙박 예약은 하나 이상의 숙박 단위로 구성됩니다.
  `stay`는 물리적 객실 유형(`accommodation_type`), 상업적 요금 계약(`rate_plan`),
  숙박 날짜(`stay_dates`), 투숙 인원 요건(`occupancy`), 투숙객 배정
  (`guest_assignments`)을 연결하는 복합 바인딩입니다.
* **플랫폼이 생성하는 투숙객 식별자(`guest.id`)**: 비즈니스 범위의 카탈로그·숙소·숙박
  식별자와 달리, 투숙객 식별자는 플랫폼이 자신의 네임스페이스 안에서 생성·할당·
  관리합니다. 비즈니스는 `guest.id`를 안정적이고 불투명한 참조로 취급합니다. 투숙객
  식별자는 개별 예약 세션 범위로 엄격히 한정된 불투명 문자열입니다(예: `"gst_01"`,
  `"gst_02"`). 플랫폼은 여러 판매자에 걸쳐 사용자를 지속적으로 추적하는 식별자를
  사용하거나 `guest.id`에 개인 식별 정보(PII)를 노출해서는 안 됩니다(**MUST NOT**).
  비즈니스는 `guest.id`를 근거로 서로 다른 예약 세션 사이의 신원을 추론하거나 연결해서는
  안 됩니다(**MUST NOT**).
    * **시점**: 플랫폼은 예약이 `ready_for_complete`에 이르기 전에는 `id` 외의
      `guests[]` 신원 필드를 보내지 않아야 하며(**SHOULD NOT**), 비즈니스가
      `messages[]`로 요청한 필드만 보내야 합니다(**SHOULD**).
* **투숙객 풀과 숙박 배정 모델**: 투숙객 데이터는 2단계 관계형 모델로 구성됩니다.
    * **루트 투숙객 풀(`guests[]`)**: 예약 전체와 관련된 모든 개별 투숙객 프로필을
      담은 평면 컬렉션입니다.
    * **숙박 배정(`stays[].guest_assignments[]`)**: `guest_id`로 특정 숙박 단위와
      루트 풀의 투숙객을 연결하고 투숙 역할(`primary`, `accompanying` 등)을 지정하는
      세부 매핑입니다.
* **예약자와 투숙객의 분리**: 데이터 모델은 법적 구매자와 실제 숙소 투숙객을 엄격히
  구분합니다.
    * **`booker`**: 결제, 연락 의무, 예약 소유권을 책임지는 법적 계약 당사자입니다.
    * **`guests`**: 실제로 숙소에 투숙할 개인입니다. 예약자를 루트 풀의 투숙객으로도
      등록할 수 있지만(**MAY**), 법인 예약, 대리 예약, 다중 숙박 예약을 지원하기 위해
      두 엔티티는 분리해 둡니다.
* **잠정적 디스커버리와 권위 있는 예약**:
    * *디스커버리 단계(잠정)*: 검색, 견적, 객실 조회 응답은 검색 파라미터를 기반으로
      잠정 요금, 이용 가능한 객실 유형, 정책 요약을 제공합니다.
    * *예약 세션(권위)*: 예약 세션을 생성하면 잠정적 디스커버리에서 권위 있는 상태로
      전환됩니다. 비즈니스는 실시간 재고 가용성을 검증하고(또는 자체 정책에 따라 임시
      소프트 홀드를 설정하고), 구속력 있는 요금 규칙을 확정하고, 객실 단위의 수용 인원
      한도를 적용하고, 권위 있는 합계(`totals[]`)를 계산하고, 구속력 있는 취소
      조건(`policies[]`)을 첨부합니다. 확정 예약은 Complete Booking Session으로 세션이
      확정되기 전까지 생성되지 않습니다.

### 가격 범위 { #pricing-scope }

숙박 예약은 소비자 보호 규정(FTC, EU 가격 투명성 지침 등)을 준수하기 위해 엄격한
총액 표시(all-in pricing) 규칙을 따릅니다. 숙박에서는 한 번의 숙박에 드는 전체 금전적
부담이 예약 확정 시점에 선결제하는 요금과, 체크인 또는 체크아웃 때 숙소가 직접 징수하는
요금(예: 리조트 요금, 지방 숙박세, 남은 숙박 잔액)으로 나뉘는 경우가 많습니다.

#### 가격 구조와 범위 지침 { #pricing-architecture-scope-guidelines }

* **권위 있는 루트 합계(`totals`)**: 최상위 `totals` 배열은 요청된 모든 숙박 단위를
  포함한 예약 전체의 구속력 있고 권위 있는 가격 내역과 총 금전적 부담을 나타냅니다.
* **숙박 단위 합계(`stays[].totals`)**: `stays[].totals`의 각 항목은 해당 숙박 단위의
  전체 숙박 기간(`stay_dates`, 즉 체크인부터 체크아웃까지)의 총 요금을 나타내며,
  1박 기준 금액이 **아닙니다**.
* **항목별 소계와 1박별 내역(`lines`)**: 합계 항목 아래의 `lines` 배열은 보조적인
  항목별 상세 내역을 제공합니다.
    * `subtotal` 합계 항목에는 1박별 숙박 요금 내역을 나타내는 `lines`를 둘 수
      있습니다(**MAY**).
    * `tax` 합계 항목에는 과세 주체별 내역(예: 주 판매세와 지방 숙박세 또는 관광세)을
      구분하는 `lines`를 둘 수 있습니다(**MAY**).
    * `fee` 합계 항목에는 필수 요금(예: 1일 리조트 요금, 청소비)을 자세히 나타내는
      `lines`를 둘 수 있습니다(**MAY**).
* **가격 투명성과 전체 포함 비용**: 플랫폼과 비즈니스는 반드시 예약 확정 전에
  투숙객에게 숙박에 따른 전체 부담액을 보여 줘야 합니다(**MUST**). 숨겨진 요금이나
  고지되지 않은 숙소 요금은 가격 투명성 기준을 위반합니다.

#### 결제 시점과 조건(`dev.ucp.common.payment.terms`) { #payment-timing-terms-devucpcommonpaymentterms }

* **권위 있는 숙박 부담액(`total`)**: 핵심 `totals.json` 계약에 따라 표준
  `type: "total"` 항목은 요청된 모든 단위를 포함한 예약 전체의 권위 있는 총
  숙박 부담액만 나타냅니다. 모든 예약 세션은 반드시 `total` 항목을 정확히 하나
  포함해야 합니다(**MUST**). 그래야 플랫폼이 관련 소비자 가격 표시 법규에 따라
  숙박의 전체 비용을 처음부터 투명하게 표시할 수 있습니다.
* **가격 내역과 결제 시점**: 숙박 예약에서 가격 내역과 결제 시점은 분리되어 있습니다.
    * `totals[]`는 **가격 내역**(`subtotal`, `fee`, `tax`, 선택적 `discount`)을
      정의합니다. total이 아닌 모든 항목은 더하면 `total`이 되는 가격 구성 요소입니다
      (`sum(non-total entries) == total`).
    * `payment.terms[]`(`dev.ucp.common.payment.terms`)는 **결제 시점**과 자금 이동
      일정을 정의합니다.
* **일정과 `totals`의 정합성**:
    * 결제 조건은 하나 이상의 `schedules[]`로 구성됩니다.
    * 선택된 조건에 속한 모든 `schedules[].amount`의 합은 반드시
      `totals[].type: "total"`과 같아야 합니다(**MUST**)(핵심
      `common/payment_terms.json` 불변 조건 충족).
    * `type: "immediate"`인 일정은 예약 완료 시점에 결제할 금액을 나타냅니다.
    * `type: "deferred"`인 일정은 지정된 미래 날짜나 사건 시점에 결제할 금액을
      나타냅니다(예: 체크인 때 결제하는 잔액, `description` 또는 `due_at`에 명시된
      보증금 결제 예정일).
    * `type: "at_property"`인 일정은 숙소가 직접 징수하는 금액을 나타냅니다(예: 도착·
      출발 시 내는 지방 관광세나 필수 리조트 요금).
* **결제 조건과 요금제**: 숙박 유통에서 즉시 결제 할인이나 선결제 혜택은 보통 같은
  요금제의 대체 결제 조건이 아니라 별도의 요금제(예: 환불 불가 사전 구매 요금과
  유연한 최저 가용 요금)로 모델링합니다. 한 요금제에 유연한 결제 조건(보증금과 잔액
  등)을 제공한다면 비즈니스는 해당하는 일정 내역을 `payment.terms[]`에 제공합니다.

#### 현지 세금과 요금 고지 { #local-tax-fee-disclosures }

필수 세금이나 요금을 숙소가 현지에서 징수하며 예약 시점에 납부할 수 없는 경우,
비즈니스는 `presentation: "disclosure"`이고 `path`가 해당 결제 일정(예:
`$.payment.terms[0].schedules[1]`)을 직접 가리키는 경고 메시지를 `messages[]`에
제공해야 합니다(**SHOULD**). 이 고지 안내에는 반드시 적용되는 현지 세율, 면제 조건,
결제 방법을 명시해야 하며(**MUST**), `links[]`의 공식 정책 링크로 보완합니다.

다음 예시는 대표적인 숙박 가격 패턴에서 `totals[]`가 결제 조건과 함께 어떻게
구성되는지 보여 줍니다.

=== "패턴 1: 숙소 징수 요금"

    도쿄 호텔 3박 숙박으로, 기본 객실 요금(`64,000`)과 선결제 서비스 요금(`6,400`)은 즉시 청구되고, 필수 리조트 요금(`6,000`)과 도쿄 숙박세(`1,200`)는 숙소에서 징수합니다. `totals[]`에서 total이 아닌 구성 요소의 합은 총 숙박 비용(`77,600`)과 정확히 같습니다.

    <!-- ucp:example schema=lodging/booking target=$.totals op=read -->
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

=== "패턴 2: 보증금과 체크인 시 잔액"

    보증금·잔액 조건으로 3박을 숙박하며, `subtotal.lines`에 1박별 내역이 들어 있습니다. 선결제 보증금은 첫날 숙박 요금(`40,000`)과 서비스 요금(`3,000`), 첫 세금(`4,000`)을 포함하고, 남은 잔액(`88,000`)은 체크인 때까지 미뤄집니다. `totals[]`에서 모든 숙박 요금, 요금, 세금을 합하면 전체 숙박 부담액(`135,000`)이 됩니다.

    <!-- ucp:example schema=lodging/booking target=$.totals op=read -->
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

> [!TIP]
> 숙박 바인딩, 대표 투숙객 배정, 메시지, 결제 조건을 모두 포함한 전체 예약 세션
> 페이로드는 [가격 및 결제 조건 예시](#pricing-examples)를 참고하세요.

### 결제 { #payments }

결제 핸들러는 `/.well-known/ucp`에 있는 비즈니스의 UCP 프로필에서 찾습니다. 핸들러는
결제 수단(예: Google Pay, Shop Pay)을 수집하기 위한 처리 명세를 정의합니다. 사용자가
결제 정보를 제출하면 플랫폼은 수집한 결제 수단 데이터로 `payment.instruments` 배열을
채웁니다.

`payment` 객체는 예약 세션 생성과 수정 작업에서는 선택 사항입니다. 완료
(`complete_booking_session`) 시에는 구속력 있는 결제 합의를 확정해야 하므로 `payment`가
**필수(REQUIRED)**입니다.

* **즉시 카드 결제**: 즉시 청구나 선결제 카드 보증이라면 수집한 결제 수단 데이터로
  `payment.instruments`를 채웁니다.
* **결제 조건 확인**: Complete는 이미 선택된 결제 조건을 확인할 뿐, 최종 조건을
  전송하지 **않습니다**. 조건 선택은 세션이 `ready_for_complete`에 이르기 전에 가격과
  파생 상태를 바꿀 수 있는 Update(`update_booking_session`)로 이루어지므로,
  Complete에서는 `selected_term_id`를 의도적으로 생략합니다.
* **결제 수단 불필요**: 결제 수단이 필요 없는 경우(예: 선결제 카드 보증이 필요 없는
  후불 또는 현장 결제 예약)에는 빈 결제 객체(`payment: {}`)로 완료 요건을 충족합니다.

### 예약 상태 수명 주기 { #booking-status-lifecycle }

예약 `status` 필드는 세션의 현재 단계를 나타내며, 다음에 어떤 조치가 필요한지 결정합니다.
상태는 비즈니스가 설정하고, 플랫폼은 진행에 무엇이 필요한지 알려 주는 메시지를 받습니다.

#### 상태 값 { #status-values }

* **`incomplete`**: 예약 세션에 필수 정보가 없거나 해결해야 할 문제가 있습니다.
    플랫폼은 `messages` 배열에서 맥락을 확인하고 Update Booking Session으로 해결을
    시도해야 합니다.
* **`requires_escalation`**: 예약 세션에 API로 제공할 수 없는 정보나
    사용자 입력이 필요합니다. 플랫폼은 `messages`를 확인해 무엇이 필요한지 파악해야
    합니다. `recoverable` 오류가 있으면 먼저 해결한 다음 `continue_url`로 사용자에게
    넘깁니다.
* **`ready_for_complete`**: 예약 세션에 필요한 정보(확정된 가격 합계, 유효한 숙박
    날짜, 필요한 경우 수집된 결제 수단, `booker` 또는 대표 투숙객 배정
    (`role: "primary"`)을 통한 대표 투숙객 식별, 차단 중인 모든 미처리 액션의 해결)가
    모두 있으며, 플랫폼이 프로그래밍 방식으로 확정할 수 있습니다. 플랫폼은 Complete Booking
    Session을 호출할 수 있습니다.
* **`complete_in_progress`**: 비즈니스가 Complete Booking 요청을 처리하고 있습니다.
    응답에는 `confirmation` 필드가 있어서는 안 됩니다(**MUST NOT**). 허용되는 작업은
    [수락된 완료](../../shopping/checkout/index.md#accepted-completion)를 참고하세요.
* **`completed`**: 예약이 성공적으로 확정되었습니다.
* **`canceled`**: 예약 세션이 유효하지 않거나 만료되었습니다. 플랫폼은 필요하면 새 예약
    세션을 시작해야 합니다.

#### 세션 만료와 재고 관리 { #session-expiry-inventory-management }

* **일시적인 세션 수명(`expires_at`)**: 예약 세션은 확정 전 단계에서 진행 중인
  리소스입니다. 비즈니스는 세션 상태, 제시된 가격, 임시 재고 홀드가 얼마나 오래 유효한지
  나타내는 `expires_at` 타임스탬프를 제공할 수 있습니다(**MAY**).
* **재고 홀드 전략**: 비즈니스는 `expires_at` 기간 동안 재고에 임시 소프트 홀드를 걸
  수도 있고(**MAY**), 세션이 업데이트될 때마다 재고 가용성을 실시간으로 확인하고
  Complete Booking Session 시점에 최종 배정을 수행하는 낙관적 동시성 방식을 사용할
  수도 있습니다(**MAY**).
* **해제 방식**: 예약 흐름이 중단되면 플랫폼은 Cancel Booking Session을 호출해 홀드된
  재고와 세션 상태를 명시적으로 해제할 수 있습니다(**MAY**). 그렇지 않으면 홀드된 리소스는
  `expires_at`이 지나면 자동으로 해제됩니다.

### 액션 { #actions }

활성화된 기능이나 확장에 예약 세션의 미처리 추가 인증(step-up) 작업(PSD2 /
3D Secure 강력한 고객 인증, 생체 인증 추가 확인, 신원 확인 등)이 있으면, 비즈니스는
응답 전용 `actions` 맵에 그 인스턴스를 노출합니다. 공통 규칙은
[개요 — 액션](../../overview/index.md#actions)에 정의되어 있으며, 이 섹션에서는 예약
상태 수명 주기에서 액션을 어떻게 해석하는지 설명합니다.

모든 액션은 해당 액션 유형에 지정된 효과를 차단합니다. `incomplete` 상태의 액션은
비즈니스가 `ready_for_complete`를 반환하기 전에 완료되어야 하는 작업을 나타낼 수
있습니다. 액션 유형 계약에 따라 액션을 처리한 뒤(예: `url_redirect`로 구매자를
리디렉션) 플랫폼은 Get Booking Session이나 이후의 Update Booking Session으로 최신
예약 상태를 가져와야 합니다(**SHOULD**).

액션 때문에 Complete Booking Session을 수락할 수 없다면, 비즈니스는 반드시 현재 예약
세션을 `status: incomplete`(사용자 인계가 필요하면 `status: requires_escalation`)로
반환하고, `path`가 해당 액션 인스턴스를 정확히 가리키는 `severity: "recoverable"`
오류 메시지를 함께 반환해야 합니다(**MUST**).

### 오류 처리 { #error-handling }

`messages` 배열에는 예약 상태와 관련된 오류, 경고, 안내 메시지가 들어 있습니다.
`ucp.status`는 응답 형태를 구분하는 값으로, `"success"`는 응답에 기대한 페이로드가
있다는 뜻이고 `"error"`는 그 대신 오류 정보가 있다는 뜻입니다. 각 오류 메시지의
`severity` 필드는 권장 조치를 지정합니다.

| 심각도                | 의미                                          | 플랫폼 조치                                                   |
| :---------------------- | :----------------------------------------------- | :---------------------------------------------------------------- |
| `recoverable`           | 플랫폼이 API로 입력을 수정해 해결할 수 있음 | 리소스를 업데이트하고 재시도                                         |
| `requires_buyer_input`  | 비즈니스가 API로 받을 수 없는 입력을 요구함    | `continue_url`로 인계                                       |
| `requires_buyer_review` | 사용자의 검토와 승인이 필요함        | `continue_url`로 인계                                       |
| `unrecoverable`         | 조치할 리소스가 없음                     | 새 리소스나 입력으로 재시도하거나 `continue_url`로 인계 |

`requires_*` 심각도의 오류는 `status: requires_escalation`으로 이어집니다. 둘 다 사용자
인계로 끝나지만 서로 다른 예약 세션 상태를 나타냅니다.

* `requires_buyer_input`은 예약 세션이 **미완성**이라는 뜻입니다. 비즈니스에 필요한
  정보를 그 API로는 프로그래밍 방식으로 수집할 수 없습니다.
* `requires_buyer_review`는 예약 세션이 **완성**되었지만, 정책·규제·자격 규칙상
  완료 전에 사용자 승인이 필요하다는 뜻입니다.

#### 표준 오류 { #standard-errors }

| 코드                          | 설명                                                              |
| :---------------------------- | :----------------------------------------------------------------------- |
| `inventory_exhausted`         | 선택한 객실 단위나 재고 홀드를 더 이상 이용할 수 없음 |
| `occupancy_exceeded_capacity` | 배정된 투숙객 수가 물리적 수용 한도를 초과함               |
| `payment_failed`              | 결제 처리 실패                                                |
| `eligibility_invalid`         | 완료 시점에 자격 주장을 검증하지 못함                    |

### 경고 표시 { #warning-presentation }

경고 메시지의 `presentation` 필드는 플랫폼이 반드시 따라야 하는(**MUST**) 렌더링
계약(예: `notice`와 `disclosure`)을 결정합니다. 권위 있는 렌더링 규칙은
[체크아웃 — 경고 표시](../../shopping/checkout/index.md#warning-presentation)를
참고하세요.

#### 합계 변경 { #totals-changes }

진행 중인 예약 세션에서 숙박 요금, 세금, 필수 요금이 바뀌면 비즈니스는 `totals[]`를
다시 계산한 세션을 반환하고, 반드시 `code: "totals_changed"`와 `path: "$.totals"`가
설정된 `messages[]` 경고 메시지로 그 변경을 알려야 합니다(**MUST**).

구매자가 변경된 금액을 확실히 인지해야 한다면 비즈니스는 반드시
`presentation: "disclosure"`를 설정해야 합니다(**MUST**). 플랫폼은 반드시 업데이트된
합계와 경고 내용을 구매자에게 보여 줘야 하며(**MUST**), 고지를 자동으로 닫거나 숨겨서는
안 됩니다(**MUST NOT**).

## Continue URL { #continue-url }

`continue_url` 필드를 사용하면 예약을 플랫폼에서 비즈니스 UI로 넘겨, 사용자가 예약
세션을 이어서 진행하고 확정할 수 있습니다.

### 제공 여부 { #availability }

비즈니스는 `status` = `requires_escalation`을 반환할 때 반드시 `continue_url`을
제공해야 합니다(**MUST**). 그 밖의 모든 비종료 상태(`incomplete`,
`ready_for_complete`, `complete_in_progress`)에서는 `continue_url`을 제공해야
합니다(**SHOULD**). 종료 상태(`completed`, `canceled`)에서는 `continue_url`을 생략해야
합니다(**SHOULD**).

## 가이드라인 { #guidelines }

### 플랫폼 { #platform }

* 예약 세션을 생성할 때는 반드시 유효한 `property.id`와 함께, 상위 퍼널 디스커버리
  메커니즘에서 얻은 미리 구성된 `stay.id` 또는 `accommodation_type.id`와
  `rate_plan.id` 식별자 둘 다를 제공해야 합니다(**MUST**).
* 반드시 각 `stays[]` 항목을 하나의 단위로 표시해야 하며(**MUST**)
  (`accommodation_type.title`, `rate_plan.title`, `rate_plan.description`(있는 경우),
  `occupancy`, 그리고 합계 렌더링 계약에 따른 해당 항목의 `totals`),
  `accommodation_type`이나 `rate_plan`이 같은 항목을 병합하거나 중복 제거해서는 안
  됩니다(**MUST NOT**).
* Complete Booking Session을 호출하기 전에 반드시 `booker` 정보를 제공하거나 적어도
  한 명의 투숙객을 `role: "primary"`(법적 성명 전체와 연락처 포함)로 지정해 대표
  투숙객을 식별해야 합니다(**MUST**). 대표 투숙객이 식별되지 않은 상태로
  `ready_for_complete`를 받은 플랫폼은 Complete Booking Session을 호출해서는 안 되며
  (**MUST NOT**), Update Booking Session으로 세션을 바로잡거나 가능하면
  `continue_url`로 에스컬레이션해야 합니다(**SHOULD**).
* 반드시 루트 `guests[]` 배열의 각 항목마다 플랫폼 네임스페이스 안에서 고유하고
  안정적이며 세션 범위로 한정된 문자열 식별자(예: `"gst_01"`, `"gst_02"`)를 생성해야
  합니다(**MUST**).
* 반드시 모든 `guest_assignments[].guest_id`가 루트 `guests[]` 풀에 있는 유효한
  `id`를 참조하도록 보장해야 합니다(**MUST**).
* 예약이 `ready_for_complete`에 이르기 전에는 `id` 외의 `guests[]` 개인 신원 필드를
  보내지 않아야 하며(**SHOULD NOT**), 비즈니스가 `messages[]`로 요청한 필드만 보내야
  합니다(**SHOULD**).
* 예약 세션 진행을 돕는 데 에이전트를 활용할 수 있습니다(**MAY**)(예: 숙박과 날짜
  선택, 투숙객 정보 수집). 다만 에이전트는 사용자가 예약 정보를 검토하고 예약을
  완료할 수 있도록 신뢰할 수 있고 결정적인 UI에 예약 세션을 넘겨야 합니다.
* 신뢰할 수 있고 결정적인 UI에서 언제든 사용자를 에이전트로 되돌려 보낼 수
  있습니다(**MAY**).
* 플랫폼이 에이전트가 보낸 요청임을 표시하는 경우 에이전트 맥락을 제공할
  수 있습니다(**MAY**).
* 예약 상태가 `requires_escalation`이면 반드시 `continue_url`을 사용해야
  합니다(**MUST**).
* 그 밖의 상황에서도 `continue_url`을 사용해 비즈니스 UI로 인계할 수 있습니다(**MAY**).
* 인계할 때는 비즈니스가 제공한 `continue_url`을 우선해야 합니다(**SHOULD**).

### 비즈니스 { #business }

* 반드시 요청된 `stay.id` 또는 복합 `accommodation_type.id`·`rate_plan.id` 바인딩을
  실시간 가용성과 재고 제약에 비추어 평가하고, 권위 있는 메타데이터, 가격 합계, 정책
  조건을 반환해야 합니다(**MUST**).
* 반드시 `rate_plan.title`로 요금제를 구별할 수 있게 하고, `rate_plan.description`이
  있다면 두 필드만으로 구매자가 `policies[]`까지 읽지 않아도 주요 상업 조건을 이해할 수
  있을 만큼 충분히 작성해야 합니다(**MUST**). 두 필드는 `policies[]`와 모순되어서는
  안 됩니다(**MUST NOT**).
* 반드시 플랫폼이 제공한 `guest.id` 식별자를 세션 업데이트와 응답 전반에서 재매핑,
  이름 변경, 수정 없이 그대로 유지해야 합니다(**MUST**).
* 반드시 모든 `stays[].guest_assignments[].guest_id` 참조가 루트 `guests[]` 배열의
  기존 항목과 일치하는지 검증해야 합니다(**MUST**).
* 식별된 대표자 없이 `ready_for_complete`, `complete_in_progress`, `completed`
  상태의 예약 세션을 반환해서는 안 됩니다(**MUST NOT**). 대표자란 법적 성명 전체와
  연락 수단을 갖춘 `booker`, 법적 성명 전체와 연락 수단을 갖추고 어떤
  `stays[].guest_assignments[]`에서 `primary`로 배정된 `guests[]`의 투숙객, 또는
  법인·대리 예약의 경우 연락 수단을 제공하는 `booker`와 함께 이름이 지정된 대표
  투숙객을 말합니다. 비즈니스에 필요한 추가 신원 필드(여권 번호, 국적, 생년월일 등)는
  반드시 비즈니스별로 `messages[]`로 협상해야 합니다(**MUST**).
* 반드시 배정된 전체 투숙 인원과 투숙객 나이를 기준으로 물리적 `capacity` 한도를
  적용해야 합니다(**MUST**).
    * 투숙객 나이는 숙박 체크인 날짜 기준 만 나이로 평가합니다.
    * 나이가 정의된 `child_age_ranges[].ages` 구간에 속하는 투숙객은 어린이로 분류하고, 어떤 어린이 구간에도 속하지 않는 투숙객은 성인으로 분류합니다.
    * 요청된 투숙 인원은 반드시 `occupancy.adults <= capacity.adults`, `occupancy.children <= capacity.children`, `occupancy.total <= capacity.total`을 충족해야 합니다(**MUST**).
    * `limit`이 있는 `child_age_ranges[]`의 각 항목마다, 나이가 그 구간에 속하는 어린이 수는 `limit`을 초과해서는 안 됩니다(**MUST NOT**).
* `booker`나 대표 투숙객 정보에 유효한 이메일 주소가 있으면, 반드시 예약 완료 후
  확인 이메일을 보내야 합니다(**MUST**).
* 정확한 오류 및 경고 메시지를 제공해야 합니다(**SHOULD**).
* 예약 세션을 처리하는 로직은 반드시 결정적이어야 합니다(**MUST**).
* `status` = `requires_escalation`을 반환할 때는 반드시 `continue_url`을 제공해야
  합니다(**MUST**).
* `status` = `requires_escalation`을 반환할 때는 반드시 `severity`가
  `requires_buyer_input` 또는 `requires_buyer_review`인 메시지를 하나 이상 포함해야
  합니다(**MUST**).
* 모든 비종료 예약 응답에 `continue_url`을 제공해야 합니다(**SHOULD**).
* 예약 세션이 "completed" 상태에 이르면 변경할 수 없는 것으로 간주합니다.

## 기능 스키마 정의 <span id="booking"></span> { #capability-schema-definition }

{{ schema_fields('booking_resp', 'lodging/booking') }}

## 작업 { #operations }

예약 기능은 다음과 같은 논리적 작업을 정의합니다.

| 작업                    | 설명                                                                         |
| :--------------------------- | :---------------------------------------------------------------------------------- |
| **Create Booking Session**   | 새 예약 세션을 시작합니다. 사용자가 예약 의사를 표시하는 즉시 호출합니다. |
| **Get Booking Session**      | 예약 세션의 현재 상태를 조회합니다.                                   |
| **Update Booking Session**   | 리소스 전체를 교체하는 방식으로 예약 세션을 수정합니다.                            |
| **Complete Booking Session** | 예약을 확정하고 예약 내역을 확인합니다.                                 |
| **Cancel Booking Session**   | 예약 세션을 취소합니다.                                                          |

### 예약 세션 생성 { #create-booking-session }

사용자가 예약 의사를 표시하면 플랫폼이 상위 퍼널의 숙소, 숙박, 숙박 날짜 파라미터로
세션을 시작할 때 호출합니다.

{{ method_fields('create_booking_session', 'lodging/rest.openapi.json', 'lodging/booking') }}

### 예약 세션 조회 { #get-booking-session }

예약 세션 리소스의 최신 상태를 조회합니다.

{{ method_fields('get_booking_session', 'lodging/rest.openapi.json', 'lodging/booking') }}

### 예약 세션 수정 { #update-booking-session }

예약 세션 리소스 전체를 교체합니다. 플랫폼은 반드시 변경된 데이터(예: 투숙객 프로필,
투숙객 배정, 예약자 정보)를 포함한 전체 예약 상태를 보내야 합니다(**REQUIRED**).

{{ method_fields('update_booking_session', 'lodging/rest.openapi.json', 'lodging/booking') }}

### 예약 세션 완료 { #complete-booking-session }

예약을 최종 접수하는 호출입니다. 결제 정보가 수집되고 사용자가 예약 확정을 결정했을 때
호출합니다.

{{ method_fields('complete_booking_session', 'lodging/rest.openapi.json', 'lodging/booking') }}

### 예약 세션 취소 { #cancel-booking-session }

완료 전에 진행 중인 예약 세션을 취소합니다.

{{ method_fields('cancel_booking_session', 'lodging/rest.openapi.json', 'lodging/booking') }}

## 전송 바인딩 { #transport-bindings }

위의 추상 작업은 다음과 같은 구체적인 전송 프로토콜에 바인딩됩니다.

* [REST 바인딩](rest.md): 표준 HTTP 메서드와 JSON 페이로드를 사용하는 RESTful API 매핑
* [MCP 바인딩](mcp.md): 에이전트 상호작용을 위한 Model Context Protocol 매핑

## 예시 {: #examples }

### 가격 예시 { #pricing-examples }

다음 예시는 숙소 징수 요금과 유연한 결제 조건을 통합해 보여 주는 완전하고 권위 있는
예약 세션입니다.

=== "숙소 징수 세금 및 요금"

    도쿄 호텔 3박 예약의 전체 예시로, 기본 객실 요금과 서비스 요금은 즉시 선결제하고 도쿄 숙박세와 리조트 요금은 체크인 때 숙소가 직접 징수합니다. 결제 시점은 `dev.ucp.common.payment.terms` 기능의 `payment.terms[]`로 명시적으로 모델링하며, `messages[]`의 세금 고지 경고는 숙소 징수 일정을 참조합니다.

    <!-- ucp:example schema=lodging/booking op=read -->
    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.lodging.booking": [
            {
              "version": "{{ ucp_version }}"
            }
          ],
          "dev.ucp.common.payment.terms": [
            {
              "version": "{{ ucp_version }}"
            }
          ]
        },
        "payment_handlers": {
          "com.example.card_handler": [
            {
              "id": "card_handler",
              "version": "{{ ucp_version }}",
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

=== "유연한 결제 조건 — 보증금과 잔액"

    보증금·잔액 결제 조건(`pt_deposit_balance`)을 제공하는 호텔 3박 예약의 전체 예시입니다. 첫날 숙박 요금과 선결제 요금, 첫 세금은 즉시 청구하고, 나머지 2박의 숙박 요금과 남은 세금은 체크인 때까지 미룹니다.

    <!-- ucp:example schema=lodging/booking op=read -->
    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "capabilities": {
          "dev.ucp.lodging.booking": [
            {
              "version": "{{ ucp_version }}"
            }
          ],
          "dev.ucp.common.payment.terms": [
            {
              "version": "{{ ucp_version }}"
            }
          ]
        },
        "payment_handlers": {
          "com.example.card_handler": [
            {
              "id": "card_handler",
              "version": "{{ ucp_version }}",
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

## 엔티티 { #entities }

### 액션 { #actions_1 }

예약 세션을 진행하는 데 필요한 추가 인증(step-up) 액션 지시(예: 3D Secure / PSD2 구매자 리디렉션이나 신원 확인).

{{ schema_fields('types/actions', 'lodging/booking') }}

### 숙소 { #property }

숙박 시설이 있는 물리적 시설이나 지리적 위치(예: 호텔, 리조트, 빌라 단지, 캐빈 파크).

{{ schema_fields('types/property_resp', 'lodging/booking') }}

### 예약 확인 { #booking-confirmation }

예약이 성공적으로 완료되면 반환되는 확인 및 예약 조회 정보.

{{ schema_fields('types/booking_confirmation', 'lodging/booking') }}

### 예약자 { #booker }

예약을 하는 법적 계약 당사자이자 주 연락 창구.

{{ schema_fields('types/booker', 'lodging/booking') }}

### 수용 인원 { #capacity }

물리적 객실 유형이 지원하는 투숙 인원 한도와 어린이 나이 기준.

{{ schema_fields('types/capacity', 'lodging/booking') }}

### 컨텍스트 { #context }

구매자 위치와 시장 맥락에 관한 힌트.

{{ schema_fields('types/context', 'lodging/booking') }}

### 날짜 구간 { #date-interval }

숙박의 체크인(`start_date`)과 체크아웃(`end_date`) 날짜 범위.

{{ schema_fields('types/date_interval', 'lodging/booking') }}

### 투숙객 { #guest }

개별 투숙객 프로필. `id`는 예약 세션 안에서 투숙객을 고유하게 식별할 수 있도록 플랫폼이
생성해 제공합니다.

{{ schema_fields('types/guest', 'lodging/booking') }}

### 투숙객 배정 { #guest-assignment }

`guest_id`로 숙박 단위와 루트 `guests[]` 풀의 투숙객을 연결하고 역할을 지정하는 관계
링크.

{{ schema_fields('types/guest_assignment', 'lodging/booking') }}

### 링크 { #link }

규정 준수 및 법적 링크(예: 개인정보 처리방침, 서비스 약관).

{{ schema_fields('types/link', 'lodging/booking') }}

### 오류 메시지 { #message-error }

{{ schema_fields('types/message_error', 'lodging/booking') }}

### 안내 메시지 { #message-info }

{{ schema_fields('types/message_info', 'lodging/booking') }}

### 경고 메시지 { #message-warning }

{{ schema_fields('types/message_warning', 'lodging/booking') }}

### 투숙 인원 { #occupancy }

숙박에 요청된 성인과 어린이 투숙객 수 내역.

{{ schema_fields('types/occupancy', 'lodging/booking') }}

### 결제 { #payment }

결제 정보와 수집된 결제 수단.

{{ schema_fields('payment', 'lodging/booking') }}

### 정책 { #policy }

예약 세션이나 숙박에 적용되는 정책(취소 조건, 숙소 이용 규칙 등).
`applies_to`의 JSONPath 대상은 이 응답의 루트를 기준으로 합니다(예: `$.stays[0]`).
전체 모델은 [정책](../../overview/index.md#policies)을 참고하세요.

{{ schema_fields('types/policy', 'lodging/booking') }}

### 요금제 { #rate-plan }

상업적 요금제 계약, 취소 정책 규칙, 요금 포함 사항.
`id`는 상위 퍼널 검색에서 얻습니다.

{{ schema_fields('types/rate_plan', 'lodging/booking') }}

### 숙박 { #stay }

물리적 객실(`accommodation_type.id`), 상업적 요금 조건(`rate_plan.id`), 숙박
날짜(`stay_dates`), 투숙 인원, 투숙객 배정을 하나로 묶는 복합 바인딩.

{{ schema_fields('types/stay', 'lodging/booking') }}

### 객실 유형 { #accommodation-type }

임대 가능한 물리적 공간의 종류나 사양(예: 객실, 아파트, 빌라, 캠핑 사이트)과 수용
인원 한도.

{{ schema_fields('types/accommodation_type', 'lodging/booking') }}

### 신호 { #signals }

플랫폼이 제공하는 사기 방지 및 보안 맥락.

{{ schema_fields('types/signals', 'lodging/booking') }}

### 합계 { #total }

권위 있는 항목별 가격 구성 요소와 예약 총액.

{{ schema_fields('types/total_resp', 'lodging/booking') }}
