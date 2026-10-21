# 분할 결제 확장

- **기능 이름:** `dev.ucp.common.payment.split_payments`

> **예시에 관한 참고:** 이 명세에서 사용하는 결제 수단 `type` 문자열 (`card`, `gift_card`, `store_credit`, `loyalty`)은 예시용입니다. 기본 명세에서 규범적 결제 수단 스키마가 있는 것은 `card` (`card_payment_instrument.json`)뿐이며, 나머지는 핸들러가 정의합니다. 예시에 나오는 자격 증명 형태도 마찬가지로 예시용이며, 실제 핸들러는 반드시 자체 자격 증명 스키마를 공개해야 합니다(MUST). 아래의 각 사용 예시는 해당 결제 수단 조합을 허용하는 비즈니스의 `allowed_combinations` 구성을 전제로 합니다. 이 문서 뒷부분의 "구성 예시" 블록은 가능한 구성 중 하나일 뿐, 유일한 구성은 아닙니다.

## 개요

분할 결제 확장을 사용하면 구매자가 한 번의 체크아웃에서 둘 이상의 결제 수단으로 결제할 수 있습니다. 비즈니스는 지원하는 결제 수단 조합을 `allowed_combinations`에 선언합니다.

각 결제 수단은 다음 두 모드 중 하나로 제출합니다.

- **금액 지정**(`amount` 있음): 플랫폼이 특정 분담액을 요청합니다.
- **금액 미지정**(`amount` 생략): 비즈니스가 처리 시점에 해당 결제 수단의 분담액을 결정합니다(예: 기프트 카드의 사용 가능 잔액을 조회).

응답에서 비즈니스는 성공적으로 처리한 결제 수단마다 실제로 승인 또는 청구한 금액을 같은 `amount` 필드로 보고합니다. 응답의 `amount`는 정보 제공용일 뿐입니다. 아래 "응답: 실제 청구액"을 참고하세요.

모든 `amount` 값은 `checkout.currency`의 최소 화폐 단위(ISO 4217)로 표현합니다. 외화 표시 결제 수단(예: USD 체크아웃에서 CAD 기프트 카드)이나 로열티 포인트 같은 비화폐 결제 수단을 사용하는 핸들러는 반드시 각 결제 수단의 분담액을 체크아웃 통화로 환산해야 합니다(MUST).

결제 수단은 **배분 우선순위 순서**로 제출합니다. 즉, 첫 번째 결제 수단이 체크아웃 총액을 가장 먼저 차지하고, 두 번째가 그다음을 차지하는 식입니다. 비즈니스는 운영상의 이유로 결제 승인을 임의의 순서로 처리할 수 있습니다(MAY)(예: 취소 비용을 줄이려고 오픈 루프 카드보다 기프트 카드를 먼저 처리). 배열 순서가 정하는 것은 처리 순서가 아니라 금액 배분입니다.

## 스키마

### 결제 수단(분할 결제)

이 기능이 활성화되면 `checkout.payment.instruments`의 각 결제 수단에 선택적 `amount` 필드가 추가됩니다.

분할 결제를 위해 결제 수단별 금액(선택 사항)으로 확장된 결제 수단.

| 이름            | 타입    | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                               |
| --------------- | ------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id              | string  | **필수**  | 이 결제 수단 인스턴스의 고유 식별자. 플랫폼이 수집하는 결제 수단은 보통 플랫폼이 할당합니다. 신원 연결된 응답으로 반환되는, 비즈니스가 소유한 저장 결제 수단이라면 비즈니스가 이 식별자를 할당합니다. 플랫폼은 반드시 이를 불투명한 비즈니스 범위 참조로 취급해야 하며(MUST), 구매자가 선택하면 비즈니스가 서버 측에서 해석합니다. |
| handler_id      | string  | **필수**  | 이 결제 수단을 생성한 핸들러 인스턴스의 고유 식별자. 결제 핸들러 정의의 'id' 필드에 해당합니다.                                                                                                                                                                                                                                    |
| type            | string  | **필수**  | 결제 수단의 대분류(예: 'card', 'tokenized_card'). 구체적인 스키마에서 이 값을 상수로 제한합니다.                                                                                                                                                                                                                                   |
| billing_address | object  | 선택      | 이 결제 방법에 연결된 청구지 주소.                                                                                                                                                                                                                                                                                                 |
| credential      | object  | 선택      | 모든 결제 자격 증명의 기본 정의. 구체적인 자격 증명 유형은 핸들러가 정의합니다.                                                                                                                                                                                                                                                    |
| display         | object  | 선택      | 이 결제 수단의 표시 정보. 각 결제 수단 스키마가 결제 핸들러의 설명에 따라 고유한 표시 속성을 정의합니다.                                                                                                                                                                                                                           |
| amount          | integer | 선택      | 이 결제 수단의 분담 금액으로, 이를 포함하는 기능 객체의 `currency` 기준 ISO 4217 최소 단위로 표현합니다. 요청 시: 플랫폼이 요청하는 분담 금액(금액을 정하지 않으면 생략). 응답 시: 실제로 승인되거나 청구된 금액(최종 처리되지 않았으면 생략).                                                                                     |

## 구성

비즈니스는 프로필에 분할 결제 구성을 선언합니다.

### 비즈니스 프로필

| 이름                 | 타입         | 필수 여부 | 설명                                                                                                            |
| -------------------- | ------------ | --------- | --------------------------------------------------------------------------------------------------------------- |
| allowed_combinations | Array[array] | **필수**  | 유효한 결제 수단 조합의 배열. 각 조합은 결제 수단 그룹의 배열입니다. 결제가 어느 한 조합과 일치하면 유효합니다. |

#### `allowed_combinations`

유효한 결제 수단 조합의 배열입니다. 각 조합은 **결제 수단 그룹**의 배열이며, 그룹들이 모여 결제를 분할하는 유효한 방식 하나를 정의하는 제약 조건이 됩니다.

결제 수단 집합이 배열의 조합 중 **하나라도** 일치하면 유효합니다.

#### 결제 수단 그룹

조합 내의 각 그룹은 특정 결제 수단 유형을 받아들이는 "슬롯"을 정의합니다.

| 이름  | 타입          | 필수 여부 | 설명                                                                                              |
| ----- | ------------- | --------- | ------------------------------------------------------------------------------------------------- |
| types | Array[string] | **필수**  | 이 그룹이 받는 결제 수단 유형(OR 논리). 나열된 유형 중 하나면 충족합니다.                         |
| min   | integer       | 선택      | 이 그룹에서 필요한 최소 결제 수단 개수. 기본값은 0(선택 사항)입니다.                              |
| max   | integer       | 선택      | 이 그룹에서 허용되는 최대 결제 수단 개수. 기본값은 1입니다. 반드시 `min` 이상이어야 합니다(MUST). |

**일치 알고리즘:** 제출된 각 결제 수단을 정확히 하나의 그룹에 할당하되 다음 조건을 만족하는 할당이 있으면 해당 제출은 그 조합과 일치합니다.

- 결제 수단의 `type`이 할당된 그룹의 `types`에 포함됩니다.
- 모든 그룹의 결제 수단 개수가 해당 그룹의 `min`과 `max` 범위(경계값 포함) 안에 있습니다.

비즈니스는 이러한 할당을 찾기 위해 어떤 알고리즘이든 사용할 수 있습니다(MAY). 유효한 할당이 하나라도 있으면 그 조합은 일치합니다.

#### 구성 예시

(a) 카드 1장과 최대 2개의 교환형 결제 수단, (b) 최대 5장의 기프트 카드 단독, (c) 신용카드 2장을 지원하는 비즈니스의 예는 다음과 같습니다.

```json
{
  "capabilities": [{
    "dev.ucp.common.payment.split_payments": [
      {
        "version": "2026-01-23",
        "config": {
          "allowed_combinations": [
            [
              { "types": ["card"], "min": 1, "max": 1 },
              { "types": ["gift_card", "store_credit"], "max": 2 }
            ],
            [
              { "types": ["gift_card"], "min": 1, "max": 5 }
            ],
            [
              { "types": ["card"], "min": 2, "max": 2 }
            ]
          ]
        }
      }
    ]
  }]
}
```

각 조합의 의미는 다음과 같습니다.

1. **카드 + 교환형 결제 수단**: 정확히 1장의 카드(필수)와, 기프트 카드 또는 스토어 크레딧 중 최대 2개(선택). 유효한 결제: 카드 단독, 카드 + 기프트 카드, 카드 + 스토어 크레딧, 카드 + 기프트 카드 2장 등.
1. **기프트 카드만**: 다른 결제 수단 유형 없이 기프트 카드 1~5장.
1. **카드 2장**: 정확히 2장의 신용/체크 카드.

## 분할 결제 사용하기

### 결제 수단 처리 모델

1. 플랫폼은 `payment.instruments` 배열에 배분 우선순위 순서로 결제 수단을 제출합니다.
1. 비즈니스는 반드시 배열 순서에 따라 각 결제 수단의 분담액을 산출해야 합니다(MUST).
1. **금액 지정**: 비즈니스는 반드시 명시된 `amount`만큼 승인해야 합니다(MUST).
1. **금액 미지정**: 비즈니스가 분담액을 결정합니다. 일반적으로 앞선 모든 분담액을 뺀 남은 체크아웃 총액을 한도로, 해당 결제 수단의 사용 가능 잔액 전부를 사용합니다. 사용 가능 잔액이 0이면 실패가 아니라 유효한 $0 분담입니다.

### 오류 처리

분할 결제는 완전히 완료되거나, 아니면 아무런 금전적 효과도 남기지 않습니다. 비즈니스가 금액이 지정된 결제 수단을 처리할 수 없거나 최종 총액을 채울 수 없으면, 반드시 `messages[]`에 `payment_failed`를 반환하고(MUST) 자신이 수행한 모든 승인을 반드시 무효화하거나 취소해야 합니다(MUST). 취소는 일시적으로 재시도할 수 있지만(MAY)(예: 매입사의 요청 한도 제한이나 실패하는 무효화 호출을 우회하기 위해), 미완료된 분할 결제로 구매자에게 청구된 상태가 남아서는 안 됩니다(MUST NOT).

프로토콜 계층에서 각 요청은 독립적으로 처리되며, 분할 결제 상태는 요청 간에 유지되지 않습니다. 비즈니스는 반드시 이전 요청이나 응답을 참조하지 않고 각 요청을 새로운 전체 제출로 처리해야 합니다(MUST).

> [!NOTE] 각 분할 결제 제출은 그 자체로 완결된 요청으로 처리됩니다. 결제 수단 집합을 바꾸면 새 제출이 되며, 새로운 [멱등성 키](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/signatures/#replay-protection)가 필요합니다. 승인을 포함한 분할 결제 상태는 제출 간에 유지되지 않으며, 앞에서 설명한 실패 시 원상 복구 요구 사항이 이를 보장합니다.

**결제 수단별 보고:** 분할 결제가 미완료이거나 실패하면, 비즈니스는 반드시 실패한 각 결제 수단마다 해당 결제 수단을 가리키는 `path`와 함께 `payment_failed` 오류를 내보내야 합니다(MUST). 또한 비즈니스는 성공한 결제 수단에 `info` 메시지를 내보내, 플랫폼이 구매자에게 보여 줄 수 있는 긍정적인 맥락(예: "Gift card authorized for $10.00")을 전달할 수 있습니다(MAY).

```json
{
  "messages": [
    {
      "type": "info",
      "path": "$.payment.instruments[0]",
      "content": "Gift card authorized for $10.00."
    },
    {
      "type": "error",
      "code": "payment_failed",
      "path": "$.payment.instruments[1]",
      "severity": "recoverable",
      "content": "Card declined — insufficient funds."
    }
  ]
}
```

플랫폼의 API로 대체 결제 수단을 수집해 다시 제출할 수 있으므로 심각도는 `recoverable`입니다. 판매자에게 필요한 정보를 API가 프로그래밍 방식으로 수집할 수 없는 경우에 쓰는 `requires_buyer_input`과는 다릅니다.

오류 조건:

- 처리할 수 없는 결제 수단이 하나라도 있으면(잘못된 자격 증명, 사기 의심 플래그, 하드 거절, 만료, 잔액 부족) 비즈니스는 반드시 오류를 반환해야 합니다(MUST). 금액 미지정 결제 수단의 사용 가능 잔액이 0인 것은 실패가 아닙니다.
- 제출된 모든 결제 수단을 적용한 뒤에도 체크아웃 총액에 도달하지 못하면 비즈니스는 반드시 오류를 반환해야 합니다(MUST).
- 지정된 모든 `amount` 값의 합이 체크아웃 총액을 초과하면 비즈니스는 반드시 오류를 반환해야 합니다(MUST).
- 제출된 결제 수단이 유효한 `allowed_combinations` 중 어느 것과도 일치하지 않으면 비즈니스는 반드시 오류를 반환해야 합니다(MUST).

### 응답: 실제 청구액

체크아웃 응답에서 비즈니스는 반드시 승인 또는 청구된 모든 결제 수단에 실제 분담액을 반영한 `amount`를 설정해야 합니다(MUST). 금액 미지정 결제 수단(`amount` 없이 제출된 결제 수단)은 비즈니스가 반드시 실제 분담액을 산출해 보고해야 합니다(MUST). 그 밖의 모든 결제 수단(예: 이후 결제 수단이 실패해 무효화된 임시 승인, 또는 비즈니스가 시도조차 하지 않은 결제 수단)에서는 반드시 `amount`를 생략해야 합니다(MUST).

응답의 `amount`는 정보 제공용입니다. 구매자 대상 UX와 감사를 위해 비즈니스가 처리한 내용을 전달합니다(예: "기프트 카드로 $10.00 청구됨"). 플랫폼은 이전 응답의 `amount` 값을 이후 요청에서 유지되는 상태로 취급해서는 안 되며(MUST NOT), 비즈니스는 플랫폼이 이를 그대로 되돌려 보낸다고 가정해서는 안 됩니다(MUST NOT). 각 요청은 새로운 의도입니다. 플랫폼은 이전 응답과 상관없이 분담액을 지정하려면 `amount`를 설정하고, 금액을 지정하지 않으려면 생략합니다.

## 예시

### 기프트 카드 + 신용카드

> "기프트 카드로 먼저 결제하고, 나머지는 신용카드로 해 줘."

**인바운드(구매자의 선택):**

```json
{
  "payment": {
    "instruments": [
      {
        "id": "pi_gc_1",
        "handler_id": "example_handler_1",
        "type": "gift_card",
        "credential": { "type": "gift_card", "token": "gc_abc123" }
      },
      {
        "id": "pi_card_1",
        "handler_id": "example_handler_1",
        "type": "card",
        "credential": { "type": "card", "token": "tok_visa_xxxx" }
      }
    ]
  }
}
```

두 결제 수단 모두 `amount`를 포함하지 않으므로 비즈니스가 두 금액을 모두 결정합니다.

**아웃바운드(완료된 체크아웃, $50 주문):**

```json
{
  "payment": {
    "instruments": [
      {
        "id": "pi_gc_1",
        "handler_id": "example_handler_1",
        "type": "gift_card",
        "credential": { "type": "gift_card", "token": "gc_abc123" },
        "amount": 1000
      },
      {
        "id": "pi_card_1",
        "handler_id": "example_handler_1",
        "type": "card",
        "credential": { "type": "card", "token": "tok_visa_xxxx" },
        "amount": 4000
      }
    ]
  }
}
```

비즈니스는 기프트 카드 잔액($10)을 조회해 전액을 청구하고, 남은 $40을 신용카드로 청구했습니다.

### 로열티 포인트 + 신용카드

> "로열티 포인트 2000점 중 500점($5 상당)을 쓰고, 나머지는 신용카드로 해 줘."

**인바운드(구매자의 선택):**

```json
{
  "payment": {
    "instruments": [
      {
        "id": "pi_lp_1",
        "handler_id": "example_handler_1",
        "type": "loyalty",
        "credential": { "type": "loyalty", "token": "lp_abc123" },
        "amount": 500
      },
      {
        "id": "pi_card_1",
        "handler_id": "example_handler_1",
        "type": "card",
        "credential": { "type": "card", "token": "tok_visa_xxxx" }
      }
    ]
  }
}
```

플랫폼은 로열티 결제 수단에 `amount: 500`을 지정합니다(고객이 정확히 500점을 쓰기로 선택했습니다). 나머지는 신용카드가 부담합니다.

**아웃바운드(완료된 체크아웃, $50 주문):**

```json
{
  "payment": {
    "instruments": [
      {
        "id": "pi_lp_1",
        "handler_id": "example_handler_1",
        "type": "loyalty",
        "credential": { "type": "loyalty", "token": "lp_abc123" },
        "amount": 500
      },
      {
        "id": "pi_card_1",
        "handler_id": "example_handler_1",
        "type": "card",
        "credential": { "type": "card", "token": "tok_visa_xxxx" },
        "amount": 4500
      }
    ]
  }
}
```

비즈니스는 요청대로 로열티 포인트로 $5를 청구했고, 남은 $45는 신용카드가 부담합니다.

### 기프트 카드 + 기프트 카드 + 신용카드(혼합 금액)

> "기프트 카드 두 장을 모두 쓰고, 나머지는 신용카드로 해 줘."

**인바운드:**

```json
{
  "payment": {
    "instruments": [
      {
        "id": "pi_gc_1",
        "handler_id": "handler_gc",
        "type": "gift_card",
        "credential": { "type": "gift_card", "token": "gc_abc123" }
      },
      {
        "id": "pi_gc_2",
        "handler_id": "handler_gc",
        "type": "gift_card",
        "credential": { "type": "gift_card", "token": "gc_def456" }
      },
      {
        "id": "pi_card_1",
        "handler_id": "handler_card",
        "type": "card",
        "credential": { "type": "card", "token": "tok_visa_xxxx" }
      }
    ]
  }
}
```

**아웃바운드(완료된 체크아웃, $100 주문):**

```json
{
  "payment": {
    "instruments": [
      {
        "id": "pi_gc_1",
        "handler_id": "handler_gc",
        "type": "gift_card",
        "credential": { "type": "gift_card", "token": "gc_abc123" },
        "amount": 2500
      },
      {
        "id": "pi_gc_2",
        "handler_id": "handler_gc",
        "type": "gift_card",
        "credential": { "type": "gift_card", "token": "gc_def456" },
        "amount": 0
      },
      {
        "id": "pi_card_1",
        "handler_id": "handler_card",
        "type": "card",
        "credential": { "type": "card", "token": "tok_visa_xxxx" },
        "amount": 7500
      }
    ]
  }
}
```

첫 번째 기프트 카드는 잔액이 $25였습니다(전액 청구). 두 번째 기프트 카드는 잔액이 $0이었는데, 이는 오류가 아니며 아무것도 분담하지 않을 뿐입니다. 남은 $75는 신용카드가 부담합니다.

### 부분 실패와 복구

> "기프트 카드로 먼저 결제하고, 나머지는 신용카드로 해 줘." 그런데 신용카드가 거절됩니다. 비즈니스는 기프트 카드 승인을 무효화하고, 대체 카드로 체크아웃을 재시도할 수 있다고 플랫폼에 알립니다.

**아웃바운드(미완료 체크아웃, $50 주문. 원자성 불변 조건에 따라 기프트 카드 승인은 무효화됨):**

```json
{
  "status": "incomplete",
  "payment": {
    "instruments": [
      {
        "id": "pi_gc_1",
        "handler_id": "example_handler_1",
        "type": "gift_card",
        "credential": { "type": "gift_card", "token": "gc_abc123" }
      },
      {
        "id": "pi_card_1",
        "handler_id": "example_handler_1",
        "type": "card",
        "credential": { "type": "card", "token": "tok_visa_xxxx" }
      }
    ]
  },
  "messages": [
    {
      "type": "info",
      "path": "$.payment.instruments[0]",
      "content": "Gift card has $10.00 available balance."
    },
    {
      "type": "error",
      "code": "payment_failed",
      "path": "$.payment.instruments[1]",
      "severity": "recoverable",
      "content": "Card declined — insufficient funds."
    }
  ]
}
```

두 결제 수단 모두 자금을 분담하지 않았으므로 `amount`를 생략합니다. 기프트 카드는 임시 승인되었다가 카드가 거절되면서 무효화되었고, 카드는 처음부터 거절되었습니다. `info` 메시지는 확인된 기프트 카드 잔액을 전달해 플랫폼이 정확한 구매자 UX를 보여 줄 수 있게 하며, `payment_failed` 오류는 `severity: recoverable`과 함께 실패한 결제 수단을 가리킵니다.

**인바운드(플랫폼이 대체 카드로 다시 제출, $50 주문):**

```json
{
  "payment": {
    "instruments": [
      {
        "id": "pi_gc_1",
        "handler_id": "example_handler_1",
        "type": "gift_card",
        "credential": { "type": "gift_card", "token": "gc_abc123" }
      },
      {
        "id": "pi_card_2",
        "handler_id": "example_handler_1",
        "type": "card",
        "credential": { "type": "card", "token": "tok_visa_yyyy" }
      }
    ]
  }
}
```

플랫폼이 새 요청을 제출합니다. 기프트 카드는 금액 미지정으로 다시 제출되고, 실패한 카드는 교체됩니다. 비즈니스는 이 요청을 새로운 의도로 처리합니다.
