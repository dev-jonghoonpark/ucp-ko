# 취소 정책 확장

## 개요

취소 정책 확장은 숙박 서비스의 핵심 [`policies[]`](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#policies) 프리미티브 위에 `dev.ucp.lodging.policy.cancellation` 정책 유형을 정의합니다. 이 유형의 정책에는 구매 전에 확인할 수 있는 기계 판독형 취소 조건이 추가됩니다. 그래서 플랫폼은 외부 정책 페이지를 따로 파싱하지 않고도 "이 예약을 취소할 수 있나요?", "무료 취소인가요, 수수료가 있나요?", "이 요금은 전액 환불 불가인가요?" 같은 질문에 답할 수 있습니다.

**주요 특징:**

- 취소가 현재 무료인지(`refundable`), 위약금이 발생하는지 (`partially_refundable`), 허용되지 않는지(`non_refundable`)를 알려 주는 3단계 환불 가능 여부 분류(`refundability`)
- 숙소별 마감 시각, 시간대 기준 기한, 위약금 일정을 상세히 담은 사람이 읽을 수 있는 요약(`description`)
- 숙소의 전체 법적 약관으로 바로 연결되는 선택적 링크(`url`)

**의존성:**

- 핵심 `policies[]` 프리미티브([정책](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#policies) 참고)
- 이 유형이 확장하는 상위 기능: 예약 (`dev.ucp.lodging.booking`)

## 디스커버리

비즈니스는 프로필에서 취소 정책 지원 여부를 알립니다. 이 유형은 `policies[]`가 있는 모든 영역을 확장합니다.

```json
{
  "ucp": {
    "version": "draft",
    "capabilities": {
      "dev.ucp.lodging.policy.cancellation": [
        {
          "version": "draft",
          "extends": [
            "dev.ucp.lodging.booking"
          ],
          "spec": "https://ucp.dev/draft/specification/lodging/extensions/cancellation-policy",
          "schema": "https://ucp.dev/draft/schemas/lodging/policy_cancellation.json"
        }
      ]
    }
  }
}
```

## 스키마

이 유형이 활성화되면 `type`이 `dev.ucp.lodging.policy.cancellation`인 `policies[]` 항목에는 기본 `type`, `description`, `applies_to`, `url` 외에 추가 속성(예: `refundability`)이 들어갑니다.

| 이름          | 타입                                                                                    | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ------------- | --------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| type          | [Reverse Domain Name](/ucp-ko/draft/draft/specification/reference/#reverse-domain-name) | **필수**  | **상수 = dev.ucp.lodging.policy.cancellation**. 숙박 취소 조건을 나타내는 정책 판별자. 정책 유형 판별자. 열린 역 DNS 어휘입니다. 잘 알려진 정책 유형의 레지스트리는 명세 문서를 참고하세요. 비즈니스는 자체 도메인에 사용자 정의 유형을 정의할 수 있습니다(MAY, 예: `com.example.policy.price_match`). 플랫폼은 알 수 없는 값을 반드시 허용해야 합니다(MUST).                                                                                                                                                                                                                                                            |
| description   | [Description](/ucp-ko/draft/draft/specification/reference/#description)                 | **필수**  | 하나 이상의 형식(plain, markdown, html)으로 된 사람이 읽을 수 있는 정책 요약. 플랫폼이 유형별 필드를 이해하지 못해도 표시할 수 있도록 모든 정책에 필수입니다. 이것은 구매자 대상 고지가 아닙니다. 표시 의무는 `messages[]` 경고로 부과됩니다(정책 섹션 참고).                                                                                                                                                                                                                                                                                                                                                            |
| applies_to    | Array[string]                                                                           | 선택      | 이 정책이 적용되는 노드를 가리키는 RFC 9535 JSONPath 표현식으로, 응답을 포함하는 루트를 기준으로 합니다(예: 장바구니·체크아웃의 `$.line_items[0]`, 카탈로그의 `$.products[2]`). 각 대상은 지정한 노드와 그 아래에 중첩된 모든 것을 포함하므로, 상품을 대상으로 하면 그 변형도 포함됩니다. 단수 쿼리(RFC 9535 Section 2.3.5.1, 이름·인덱스 선택자만 사용)는 노드 하나를 지정하고, 필터·와일드카드·슬라이스는 노드 집합과 일치합니다. 생략하면 정책이 응답 전체에 적용됩니다. 같은 `type`의 정책들이 한 노드를 두고 경합하면 가장 좁은 대상이 이기고 나머지를 덮어씁니다. 구체성을 판정하는 방법은 정책 섹션을 참고하세요. |
| url           | string                                                                                  | 선택      | 전체 정책 문서 링크(선택 사항).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| refundability | string                                                                                  | **필수**  | 상위 수준의 환불 가능 여부 분류. 잘 알려진 값: `refundable`(현재 무료 취소 가능), `partially_refundable`(위약금을 내고 취소 가능하거나 위약금 부과 기간 내), `non_refundable`(취소 시 환불 없음).                                                                                                                                                                                                                                                                                                                                                                                                                        |

## 취소 조건

### 환불 가능 여부 분류

`refundability` 필드는 표준화된 상위 수준 분류를 제공합니다.

- **`refundable`**: 현재 무료로 취소할 수 있습니다. 예약자는 `description`에 명시된 기한 전까지 위약금 없이 취소할 수 있습니다.
- **`partially_refundable`**: 예약을 취소할 수는 있지만 취소 수수료(예: 1박 객실 요금이나 고정 관리 수수료)가 부과되거나, 예약이 이미 부분 위약금 적용 기간에 들어와 있습니다. 구체적인 취소 수수료, 위약금 일정, 환불 영향은 정책의 `description` 필드에 설명합니다.
- **`non_refundable`**: 취소해도 예약 금액을 환불받을 수 없습니다(현지 규정에 따라 비즈니스가 예약 금액 전액을 보유합니다).

### 환불 불가 예약

예약이나 요금이 환불 불가임을 알리려면 비즈니스는 반드시 `refundability`를 `"non_refundable"`로 설정해야 합니다(**MUST**).

확정 전에 예약이 환불 불가라는 사실을 예약자에게 보여 줘야 한다면, 비즈니스는 해당 항목을 대상으로 `presentation: "disclosure"`이고 `code`가 `dev.ucp.lodging.policy.cancellation`인 `messages[]` 경고를 보냅니다. 이 고지는 [정책 표시](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#presenting-policies)에 정의된 대로 해당 노드에 적용되는 취소 정책과 짝을 이룹니다.

### 사람이 읽을 수 있는 설명

숙박 취소 규칙에는 숙소 현지 기준의 마감 시각(예: "체크인 2일 전 숙소 시간 오후 3시까지")이나 시즌별 규칙이 들어가는 경우가 많습니다. 따라서 비즈니스는 반드시 전체 일정과 조건을 `description`에 명확히 적어야 합니다(**MUST**). `description`과 `refundability` 필드는 서로 모순되어서는 안 됩니다(**MUST NOT**).

## 대상 지정과 우선순위

대상 지정과 우선순위는 `policies[]` 프리미티브가 제공하므로 여기서 다시 정의하지 않습니다. 요약하면 `applies_to`가 없는 정책은 응답 전체의 기본값이고, 특정 객실을 대상으로 재정의하는 정책이 있으면 같은 유형 중 대상이 가장 좁은 정책이 우선합니다. 자세한 내용은 [대상 지정](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#targeting)과 [우선순위](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#precedence)를 참고하세요.

숙박 예약에서는 비즈니스가 기본 취소 정책을 한 번만 명시하고, 특정 예외(환불 불가 객실 요금이나 프로모션 업그레이드 등)에만 대상 지정 재정의를 추가하는 경우가 흔합니다.

## 책임

취소 정책은 비즈니스가 명시하는 사실입니다. 플랫폼이 제출하지 않는 응답 전용 데이터이며, 사용자가 주장하는 내용이나 PII는 담지 않습니다.

비즈니스는 취소 일정, 적용되는 위약금, 마감 기한을 `description`에 정확히 요약하고 `url`로 전체 정책 약관을 연결해야 합니다(**SHOULD**). 플랫폼은 예약자가 숙소의 전체 정책을 확인할 수 있도록 정책 설명과 함께 `url`을 노출해야 합니다(**SHOULD**).

## 예시

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
