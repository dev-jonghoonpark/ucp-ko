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

# 취소 정책 확장 { #cancellation-policy-extension }

## 개요 { #overview }

취소 정책 확장은 숙박 서비스의 핵심
[`policies[]`](../../overview/index.md#policies) 프리미티브 위에
`dev.ucp.lodging.policy.cancellation` 정책 유형을 정의합니다. 이 유형의
정책에는 구매 전에 확인할 수 있는 기계 판독형 취소 조건이 추가됩니다. 그래서
플랫폼은 외부 정책 페이지를 따로 파싱하지 않고도 "이 예약을 취소할 수 있나요?",
"무료 취소인가요, 수수료가 있나요?", "이 요금은 전액 환불 불가인가요?" 같은
질문에 답할 수 있습니다.

**주요 특징:**

- 취소가 현재 무료인지(`refundable`), 위약금이 발생하는지
  (`partially_refundable`), 허용되지 않는지(`non_refundable`)를 알려 주는
  3단계 환불 가능 여부 분류(`refundability`)
- 숙소별 마감 시각, 시간대 기준 기한, 위약금 일정을 상세히 담은
  사람이 읽을 수 있는 요약(`description`)
- 숙소의 전체 법적 약관으로 바로 연결되는 선택적 링크(`url`)

**의존성:**

- 핵심 `policies[]` 프리미티브([정책](../../overview/index.md#policies) 참고)
- 이 유형이 확장하는 상위 기능: 예약
  (`dev.ucp.lodging.booking`)

## 디스커버리 { #discovery }

비즈니스는 프로필에서 취소 정책 지원 여부를 알립니다. 이 유형은
`policies[]`가 있는 모든 영역을 확장합니다.

<!-- ucp:example schema=profile def=business_schema extract=$.ucp.capabilities target=$.ucp.capabilities -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": {
      "dev.ucp.lodging.policy.cancellation": [
        {
          "version": "{{ ucp_version }}",
          "extends": [
            "dev.ucp.lodging.booking"
          ],
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/lodging/extensions/cancellation-policy",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/lodging/policy_cancellation.json"
        }
      ]
    }
  }
}
```

## 스키마 { #schema }

이 유형이 활성화되면 `type`이 `dev.ucp.lodging.policy.cancellation`인
`policies[]` 항목에는 기본 `type`, `description`, `applies_to`, `url` 외에
추가 속성(예: `refundability`)이 들어갑니다.

{{ extension_schema_fields('policy_cancellation.json#/$defs/cancellation_item', 'lodging/extensions/cancellation-policy') }}

## 취소 조건 { #cancellation-terms }

### 환불 가능 여부 분류 { #refundability-classifications }

`refundability` 필드는 표준화된 상위 수준 분류를 제공합니다.

- **`refundable`**: 현재 무료로 취소할 수 있습니다. 예약자는 `description`에
  명시된 기한 전까지 위약금 없이 취소할 수 있습니다.
- **`partially_refundable`**: 예약을 취소할 수는 있지만 취소 수수료(예: 1박
  객실 요금이나 고정 관리 수수료)가 부과되거나, 예약이 이미 부분 위약금 적용
  기간에 들어와 있습니다. 구체적인 취소 수수료, 위약금 일정, 환불 영향은 정책의
  `description` 필드에 설명합니다.
- **`non_refundable`**: 취소해도 예약 금액을 환불받을 수 없습니다(현지 규정에
  따라 비즈니스가 예약 금액 전액을 보유합니다).

### 환불 불가 예약 { #non-refundable-bookings }

예약이나 요금이 환불 불가임을 알리려면 비즈니스는 반드시 `refundability`를
`"non_refundable"`로 설정해야 합니다(**MUST**).

확정 전에 예약이 환불 불가라는 사실을 예약자에게 보여 줘야 한다면,
비즈니스는 해당 항목을 대상으로 `presentation: "disclosure"`이고 `code`가
`dev.ucp.lodging.policy.cancellation`인 `messages[]` 경고를 보냅니다. 이
고지는 [정책 표시](../../overview/index.md#presenting-policies)에 정의된
대로 해당 노드에 적용되는 취소 정책과 짝을 이룹니다.

### 사람이 읽을 수 있는 설명 { #human-readable-descriptions }

숙박 취소 규칙에는 숙소 현지 기준의 마감 시각(예: "체크인 2일 전 숙소 시간
오후 3시까지")이나 시즌별 규칙이 들어가는 경우가 많습니다. 따라서 비즈니스는
반드시 전체 일정과 조건을 `description`에 명확히 적어야 합니다(**MUST**).
`description`과 `refundability` 필드는 서로 모순되어서는 안 됩니다(**MUST NOT**).

## 대상 지정과 우선순위 { #targeting-and-precedence }

대상 지정과 우선순위는 `policies[]` 프리미티브가 제공하므로 여기서 다시
정의하지 않습니다. 요약하면 `applies_to`가 없는 정책은 응답 전체의 기본값이고,
특정 객실을 대상으로 재정의하는 정책이 있으면 같은 유형 중 대상이 가장 좁은
정책이 우선합니다. 자세한 내용은 [대상 지정](../../overview/index.md#targeting)과
[우선순위](../../overview/index.md#precedence)를 참고하세요.

숙박 예약에서는 비즈니스가 기본 취소 정책을 한 번만 명시하고, 특정
예외(환불 불가 객실 요금이나 프로모션 업그레이드 등)에만 대상 지정
재정의를 추가하는 경우가 흔합니다.

## 책임 { #responsibilities }

취소 정책은 비즈니스가 명시하는 사실입니다. 플랫폼이 제출하지 않는 응답
전용 데이터이며, 사용자가 주장하는 내용이나 PII는 담지 않습니다.

비즈니스는 취소 일정, 적용되는 위약금, 마감 기한을 `description`에 정확히
요약하고 `url`로 전체 정책 약관을 연결해야 합니다(**SHOULD**). 플랫폼은 예약자가
숙소의 전체 정책을 확인할 수 있도록 정책 설명과 함께 `url`을 노출해야
합니다(**SHOULD**).

## 예시 { #examples }

<!-- ucp:example schema=lodging/booking target=$.policies -->
```json
[
  {
    "type": "dev.ucp.lodging.policy.cancellation",
    "description": {
      "plain": "Free cancellation until Dec 20, 2026, 3:00 PM EDT (48 hours before check-in). 1 night penalty thereafter."
    },
    "refundability": "refundable",
    "url": "https://example.com/cancellation-terms"
  },
  {
    "type": "dev.ucp.lodging.policy.cancellation",
    "description": {
      "plain": "Non-refundable promotional rate. This room reservation cannot be cancelled or modified for a refund."
    },
    "applies_to": ["$.stays[0]"],
    "refundability": "non_refundable",
    "url": "https://example.com/cancellation-terms#non-refundable"
  }
]
```
