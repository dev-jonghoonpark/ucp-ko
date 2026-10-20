# 체크아웃 기능

- **기능 이름:** `dev.ucp.shopping.checkout`

## 개요

플랫폼이 체크아웃 세션을 진행할 수 있게 합니다. AP2 위임(Mandate) 확장을 지원하지 않는 한, 체크아웃은 사용자가 신뢰할 수 있는 UI에서 직접 최종 확정해야 합니다.

비즈니스는 계속 Merchant of Record(MoR)로 남으며, 이 기능으로 카드 결제를 받더라도 PCI DSS 인증을 받을 필요가 없습니다.

### 흐름 개요

### 결제

결제 핸들러는 `/.well-known/ucp`에 있는 비즈니스의 UCP 프로필과 checkout.ucp.payment_handlers에서 찾을 수 있습니다. 핸들러는 결제 수단(예: Google Pay, Shop Pay)을 수집하는 처리 사양을 정의합니다. 구매자가 결제를 제출하면 플랫폼은 수집한 결제 수단 데이터로 `payment.instruments` 배열을 채웁니다.

`payment` 객체는 체크아웃 생성 시 선택 사항입니다. 결제 처리가 필요 없는 사용 사례 (예: 견적 생성, 장바구니 관리)에서는 생략할 수 있습니다.

### 풀필먼트

UCP는 다양한 사용 사례를 수용하기 위해 풀필먼트를 확장으로 모델링합니다.

체크아웃 객체에서 풀필먼트는 선택 사항입니다. 덕분에 플랫폼은 실물 상품에나 필요한 풀필먼트 세부 정보를 제공하지 않고도 디지털 상품을 체크아웃할 수 있습니다.

### 체크아웃 상태 수명 주기

체크아웃의 `status` 필드는 세션의 현재 단계를 나타내며, 다음에 어떤 조치가 필요한지 결정합니다. 상태는 비즈니스가 설정하고, 플랫폼은 진행에 필요한 사항을 알려 주는 메시지를 받습니다.

```text
       +------------+                         +---------------------+
       | incomplete |<----------------------->| requires_escalation |
       +-----+------+                         |   (buyer handoff    |
             |                                |  via continue_url)  |
             | all info collected             +----------+----------+
             v                                           |
    +------------------+                                 |
    |ready_for_complete|                                 |
    |                  |                                 |
    | (platform can    |                                 | continue_url
    | call Complete    |                                 |
    |   Checkout)      |                                 |
    +--------+---------+                                 |
             |                                           |
             | Complete Checkout                         |
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

### 상태 값

- **`incomplete`**: 체크아웃 세션에 필수 정보가 빠졌거나 해결해야 할 문제가 있습니다. 플랫폼은 `messages` 배열에서 맥락을 확인하고 Update Checkout으로 해결을 시도해야 합니다.
- **`requires_escalation`**: 체크아웃 세션에 API로 제공할 수 없는 정보나 구매자의 입력이 필요합니다. 플랫폼은 무엇이 필요한지 파악하려면 `messages`를 확인해야 합니다(아래 에러 처리 참고). `recoverable` 에러가 있으면 먼저 해결합니다. 그런 다음 `continue_url`로 구매자에게 넘깁니다.
- **`ready_for_complete`**: 체크아웃 세션에 필요한 정보가 모두 있어 플랫폼이 프로그래밍 방식으로 최종 확정할 수 있습니다. 플랫폼은 Complete Checkout을 호출할 수 있습니다.
- **`complete_in_progress`**: 비즈니스가 Complete Checkout 요청을 처리하고 있습니다.
- **`completed`**: 주문이 접수되었습니다.
- **`canceled`**: 체크아웃 세션이 유효하지 않거나 만료되었습니다. 필요하면 플랫폼은 새 체크아웃 세션을 시작해야 합니다.

### 에러 처리

`messages` 배열에는 체크아웃 상태에 관한 에러, 경고, 정보 메시지가 담깁니다. `ucp.status`는 형태 판별자입니다. `"success"`는 응답에 기대한 페이로드가 담겨 있다는 뜻이고, `"error"`는 그 대신 에러 정보가 담겨 있다는 뜻입니다. 각 에러 메시지의 `severity` 필드는 권장 조치를 지정합니다.

| 심각도                  | 의미                                        | 플랫폼 조치                                             |
| ----------------------- | ------------------------------------------- | ------------------------------------------------------- |
| `recoverable`           | 플랫폼이 API로 입력을 수정해 해결할 수 있음 | 리소스를 갱신하고 재시도                                |
| `requires_buyer_input`  | 비즈니스가 API로 받을 수 없는 입력을 요구함 | `continue_url`로 넘김                                   |
| `requires_buyer_review` | 구매자의 검토와 승인이 필요함               | `continue_url`로 넘김                                   |
| `unrecoverable`         | 작업 대상 리소스가 존재하지 않음            | 새 리소스나 입력으로 재시도하거나 `continue_url`로 넘김 |

`requires_*` 심각도의 에러가 있으면 `status: requires_escalation`이 됩니다. 둘 다 구매자 넘김으로 이어지지만, 서로 다른 체크아웃 상태를 나타냅니다.

- `requires_buyer_input`은 체크아웃이 **미완료** 상태라는 뜻입니다. 비즈니스가 API로 프로그래밍 방식 수집을 지원하지 않는 정보를 요구하는 경우입니다.
- `requires_buyer_review`는 체크아웃은 **완료** 상태이지만, 정책, 규제, 권한 규칙상 주문 접수 전에 구매자의 승인이 필요하다는 뜻입니다(예: 고액 주문 승인, 첫 구매 정책).

비즈니스가 새 리소스를 생성할 수 없거나 요청한 리소스가 더 이상 존재하지 않으면, 응답에는 실패를 설명하는 `messages`와 함께 `ucp.status: "error"`가 담기고 응답 본문에는 리소스가 포함되지 않습니다. 작업 대상 리소스가 없으면 메시지는 `severity: "unrecoverable"`을 사용해야 합니다(SHOULD). 예를 들어 비즈니스는 모든 상품을 구매할 수 없는 체크아웃 생성 요청을 거부할 수 있습니다.

```json
{
  "ucp": { "version": "2026-01-11", "status": "error" },
  "messages": [
    {
      "type": "error",
      "code": "out_of_stock",
      "content": "All requested items are currently out of stock",
      "severity": "unrecoverable"
    }
  ],
  "continue_url": "https://merchant.com/"
}
```

[REST](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout-rest/#create-checkout) 및 [MCP](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout-mcp/#create_checkout) 바인딩 예시를 참고하세요.

#### 에러 처리 알고리즘

상태가 `incomplete` 또는 `requires_escalation`이면 플랫폼은 에러를 우선순위 스택으로 처리해야 합니다. 아래 예시는 세 가지 에러 유형, 즉 복구 가능한 에러(잘못된 전화번호), 구매자 입력 요구(배송 일정 지정), 검토 요구(고액 주문)가 있는 체크아웃입니다. 뒤의 두 가지는 넘김이 필요하며 플랫폼에 보내는 명시적 신호 역할을 합니다. 비즈니스는 이런 메시지를 가능한 한 일찍 노출해야 하며(**SHOULD**), 플랫폼은 넘김을 시작하기 전에 복구 가능한 에러부터 해결해야 합니다(**SHOULD**).

```json
{
  "status": "requires_escalation",
  "messages": [
    {
      "type": "error",
      "code": "invalid_phone",
      "severity": "recoverable",
      "content": "Phone number format is invalid"
    },
    {
      "type": "error",
      "code": "schedule_delivery",
      "severity": "requires_buyer_input",
      "content": "Select delivery window for your purchase"
    },
    {
      "type": "error",
      "code": "high_value_order",
      "severity": "requires_buyer_review",
      "content": "Orders over $500 require additional verification"
    }
  ]
}
```

에러 처리 알고리즘 예시입니다.

```text
GIVEN response with messages array

FILTER errors FROM messages WHERE type = "error"

PARTITION errors INTO
  recoverable           WHERE severity = "recoverable"
  requires_buyer_input  WHERE severity = "requires_buyer_input"
  requires_buyer_review WHERE severity = "requires_buyer_review"
  unrecoverable         WHERE severity = "unrecoverable"

IF unrecoverable is not empty
  RETRY with new resource or inputs, or hand off via continue_url
  RETURN

IF recoverable is not empty
  FOR EACH error IN recoverable
    ATTEMPT to fix error (e.g., reformat phone number)
  CALL Update Checkout
  RETURN and re-evaluate response

IF requires_buyer_input is not empty
  handoff_context = "incomplete, additional input from buyer is required"
ELSE IF requires_buyer_review is not empty
  handoff_context = "ready for final review by the buyer"
```

#### 표준 에러

표준 에러는 플랫폼이 일반적인 에러 처리 대신 구체적이고 적절한 UX로 처리해야 하는 표준화된 에러 코드입니다.

| 코드                    | 설명                                            |
| ----------------------- | ----------------------------------------------- |
| `out_of_stock`          | 특정 상품 또는 변형을 구매할 수 없음            |
| `item_unavailable`      | 상품을 구매할 수 없음(예: 판매 목록에서 제외됨) |
| `address_undeliverable` | 제공된 주소로 배송할 수 없음                    |
| `payment_failed`        | 결제 처리에 실패함                              |
| `eligibility_invalid`   | 완료 시점에 자격 주장을 검증할 수 없음          |

비즈니스는 표준 에러에 `severity: recoverable`을 표시해야 합니다(**SHOULD**). 그러면 플랫폼은 일반적인 에러 메시지를 보여 주거나 체크아웃 완료 시점까지 미루지 않고 적절한 UX(품절 안내, 주소 확인 프롬프트, 결제 수단 변경)를 제공해야 한다는 것을 알 수 있습니다.

예: `out_of_stock`은 미리 구체적인 UX가 필요하지만, `payment_required`는 제출 시점에 일반적인 방식으로 처리해도 됩니다.

#### 완료 시 자격 검증

플랫폼은 `context.eligibility`로 구매자의 주장을 제공합니다. 로열티 멤버십, 결제 수단 혜택처럼 구매자가 자신이 받을 수 있다고 내세우는 혜택입니다. 이는 검증된 사실이 아니라 주장입니다. 비즈니스는 세션 중에 인식한 주장에 따라 조치할 수 있지만(**MAY**)(가격 조정, 상품 접근 허용, 잠정 할인 적용), 수락한 주장은 반드시 거래가 완료되기 전에 모두 해소해야 합니다(**MUST**).

인식하지 못했거나 적용할 수 없는 주장 때문에 체크아웃이 막혀서는 안 됩니다(**MUST NOT**). 주장을 수락하지 않으면 비즈니스는 `type: "warning"` `messages`로 구매자에게 알려야 하며(**SHOULD**), 수락한 주장의 효과를 설명하는 데 `type: "info"`를 사용할 수 있습니다(**MAY**). 완료 시점까지 검증되지 않은 수락된 주장은 반드시 `code: "eligibility_invalid"`인 `type: "error"`로 처리해야 합니다(**MUST**)(아래 참고).

**자격 메시지 코드:**

| 유형      | 코드                       | 발생 시점                                |
| --------- | -------------------------- | ---------------------------------------- |
| `warning` | `eligibility_not_accepted` | 주장을 인식하지 못했거나 적용할 수 없음  |
| `info`    | `eligibility_accepted`     | 수락된 주장의 효과                       |
| `error`   | `eligibility_invalid`      | 완료 시점에 수락된 주장을 검증할 수 없음 |

주장은 **검증**되거나 **철회**되면 해소됩니다.

- **검증됨**: 비즈니스가 완료 시점에 제공된 증명을 바탕으로 주장을 확인합니다. UCP는 검증 방식을 규정하지 않습니다. 증명은 결제 자격 증명, 신원 검증 기능, 또는 플랫폼과 비즈니스가 협상한 다른 메커니즘에서 올 수 있습니다.
- **철회됨**: 플랫폼이 완료 전에 `context.eligibility`에서 주장을 제거합니다(예: 구매자가 결제 수단을 바꾸거나 멤버십 주장을 철회한 경우). 주장이 제거되면 비즈니스는 그 주장 없이 다시 계산합니다.

비즈니스는 해소되지 않은 자격 주장이 남은 상태로 거래를 완료해서는 안 됩니다(**MUST NOT**). 검증되지 않은 주장은 잘못된 가격 책정이나 제한된 상품에 대한 무단 접근으로 이어질 수 있습니다.

**검증에 실패한 경우:**

검증 실패는 반드시 `messages` 배열에만 영향을 주어야 합니다(**MUST**). 비즈니스는 반드시 `code: "eligibility_invalid"`, `severity: "recoverable"`인 에러를 `messages`에 반환해야 합니다(**MUST**). 메시지는 `path` 필드로 어떤 주장을 검증할 수 없었는지 알려야 합니다(**SHOULD**). 그러면 플랫폼은 유효한 증명을 제공해 다시 제출하거나, 체크아웃을 재구성하거나(예: 자격이 없는 상품 제거, 주장 갱신), 시도를 포기할 수 있습니다(**MAY**).

예를 들어 플랫폼이 `context.eligibility`로 스토어 카드 혜택을 주장하고, 비즈니스는 세션 동안 회원 가격을 적용합니다. 그런데 완료 시점에 결제 자격 증명이 주장한 결제 수단과 일치하지 않습니다.

```json
{
  "ucp": { "version": "2026-01-11", "status": "success" },
  "id": "checkout_abc",
  "status": "ready_for_complete",
  "line_items": [ "..." ],
  "totals": [ "..." ],
  "messages": [
    {
      "type": "error",
      "code": "eligibility_invalid",
      "severity": "recoverable",
      "content": "Payment credential does not match the claimed store card benefit.",
      "path": "$.context.eligibility[0]"
    }
  ]
}
```

플랫폼은 구매자가 자격 요건을 충족하는 결제 수단으로 바꾸게 하거나, `context.eligibility`에서 주장을 제거해 체크아웃을 다시 협상한 다음(갱신된 가격, 재고 등을 받음) 완료를 다시 요청하는 방식으로 이 문제를 해결할 수 있습니다.

### 경고 표시 방식

경고 메시지의 `presentation` 필드는 플랫폼이 반드시 따라야 하는(**MUST**) 렌더링 계약을 지정합니다. 생략하면 기본값은 `"notice"`입니다.

|                                | `notice` (기본값) | `disclosure`              |
| ------------------------------ | ----------------- | ------------------------- |
| 내용 표시                      | **MUST**          | **MUST**                  |
| `path`와의 근접 배치           | **MAY**           | **MUST**                  |
| 닫기 허용                      | **MAY**           | **MUST NOT**              |
| `image_url` 렌더링             | **MAY**           | **MUST**                  |
| `url` 렌더링                   | **MAY**           | **SHOULD**                |
| 준수할 수 없을 때 에스컬레이션 | —                 | `continue_url`로 **MUST** |

#### `notice` (기본값)

경고의 기본 렌더링 계약입니다. 플랫폼은 반드시 경고 내용을 구매자에게 표시해야 합니다(**MUST**). 플랫폼은 알림을 배너, 트레이, 토스트 형태로 렌더링할 수 있으며(**MAY**), 구매자가 알림을 닫도록 허용할 수 있습니다(**MAY**).

#### `disclosure`

`presentation: "disclosure"`인 경고에는 안전 경고, 알레르기 유발 물질 표시, 규정 준수 내용처럼 반드시 아래 렌더링 계약을 따라야 하는(**MUST**) 고지가 담깁니다.

**플랫폼 요구 사항:**

- 반드시 경고 `content`를 구매자에게 표시해야 합니다(**MUST**).
- 고지와 그 대상의 연관성이 유지되도록 반드시 `path`가 참조하는 구성 요소 가까이에 경고를 표시해야 합니다(**MUST**). `path`를 생략하면 고지는 응답 전체에 적용됩니다.
- 경고를 숨기거나, 접거나, 자동으로 닫아서는 안 됩니다(**MUST NOT**).
- `image_url`이 있으면 반드시 렌더링해야 합니다(**MUST**)(예: 경고 기호, 에너지 등급 라벨).
- `url`이 있으면 이동 가능한 참조 링크로 렌더링해야 합니다(**SHOULD**).

`presentation: "disclosure"`인 경고는 일반 알림보다 렌더링 우선순위가 높아야 합니다(**SHOULD**).

고지 렌더링 계약을 지킬 수 없는 플랫폼은 슬그머니 일반 알림으로 낮춰 표시하지 말고, 반드시 `continue_url`로 판매자 UI에 에스컬레이션해야 합니다(**MUST**).

**비즈니스 요구 사항:**

- 경고 내용을 특정 구성 요소와 함께 표시해야 하고 숨기거나 자동으로 닫아서는 안 되는 경우, 반드시 `presentation: "disclosure"`를 설정해야 합니다(**MUST**).
- `path` 필드로 고지를 응답의 관련 구성 요소와 연결해야 합니다(**SHOULD**).
- 고지 범주를 식별하는 `code`(예: `prop65`, `allergens`, `energy_label`)를 제공해야 합니다(**SHOULD**).
- 고지에 관련 시각 요소(예: 경고 기호, 에너지 등급 라벨)가 있으면 `image_url`을 제공해야 합니다(**SHOULD**).
- 구매자가 더 자세히 알아볼 수 있는 참조 링크가 있으면 `url`을 제공해야 합니다(**SHOULD**).

#### 고지와 확인

`presentation` 필드는 체크아웃을 진행할 수 있는지가 아니라 경고를 어떻게 렌더링할지를 제어합니다. 구매자의 명시적인 확인이나 승인도 필요하다면, 비즈니스는 적절한 구매자 입력을 확실히 받을 수 있도록 고지를 [체크아웃 상태 수명 주기](#checkout-status-lifecycle)에 설명된 에스컬레이션 메커니즘과 함께 사용할 수 있습니다(**MAY**).

#### 관할권과 적용 범위

특정 세션에 어떤 고지가 적용되는지 판단하고 관련 있는 고지만 반환하는 일은 비즈니스가 책임집니다. 비즈니스는 관할권별 요구 사항을 판단할 때 구매자가 제공한 데이터 (`context` 및 기타 입력)와 상품 속성을 사용해야 합니다(**SHOULD**). 플랫폼은 고지 적용 범위에 관여하거나 이를 결정하지 않고, 비즈니스에게서 받은 내용을 렌더링합니다.

#### 예시

라인 아이템에 복구 가능한 에러와 고지 경고가 함께 담긴 체크아웃 응답입니다.

```json
{
  "ucp": { "version": "2026-04-08", "status": "success" },
  "id": "chk_abc123",
  "status": "incomplete",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_1",
      "item": { "id": "item_456", "title": "Artisan Nut Butter Collection", "image_url": "https://merchant.com/nut-butter.jpg" },
      "quantity": 1,
      "totals": [{ "type": "subtotal", "amount": 1299 }]
    }
  ],
  "totals": [{ "type": "total", "amount": 1299 }],
  "messages": [
    {
      "type": "error",
      "code": "field_required",
      "path": "$.buyer.email",
      "content": "Buyer email is required",
      "severity": "recoverable"
    },
    {
      "type": "warning",
      "code": "allergens",
      "path": "$.line_items[0]",
      "content": "**Contains: tree nuts.** Produced in a facility that also processes peanuts, milk, and soy.",
      "content_type": "markdown",
      "presentation": "disclosure",
      "image_url": "https://merchant.com/allergen-tree-nuts.svg",
      "url": "https://merchant.com/allergen-info"
    }
  ],
  "links": []
}
```

플랫폼은 복구 가능한 에러를 프로그래밍 방식으로 해결하면서, 알레르기 유발 물질 고지를 참조된 라인 아이템 가까이에 렌더링합니다.

## Continue URL

`continue_url` 필드를 사용하면 플랫폼에서 비즈니스 UI로 체크아웃을 넘길 수 있으므로, 구매자가 체크아웃 세션을 이어서 진행하고 최종 확정할 수 있습니다.

### 제공 조건

비즈니스는 `status` = `requires_escalation`을 반환할 때 반드시 `continue_url`을 제공해야 합니다(**MUST**). 그 밖의 비종료 상태(`incomplete`, `ready_for_complete`, `complete_in_progress`)에서는 모두 `continue_url`을 제공해야 합니다(**SHOULD**). 종료 상태(`completed`, `canceled`)에서는 `continue_url`을 생략해야 합니다(**SHOULD**).

### 형식

`continue_url`은 반드시 절대 HTTPS URL이어야 하며(**MUST**), 매끄럽게 넘길 수 있도록 체크아웃 상태를 보존해야 합니다(**SHOULD**). 비즈니스는 다음 중 한 가지 방법으로 상태 보존을 구현할 수 있습니다(**MAY**).

#### 서버 측 상태(권장)

서버 측 체크아웃 상태를 기반으로 하는 불투명한 URL입니다.

```text
https://business.example.com/checkout-sessions/{checkout_id}
```

- 서버가 `checkout_id`에 연결된 체크아웃 상태를 유지합니다.
- 단순하고 안전하며, 대부분의 구현에 권장합니다.
- URL 수명은 보통 `expires_at`에 연동됩니다.

#### 체크아웃 퍼머링크

체크아웃 상태를 직접 인코딩하여 서버 측 저장 없이도 재구성할 수 있는 상태 비저장 URL입니다. 비즈니스는 체크아웃 넘김과 빠른 진입을 돕기 위해 이 형식을 지원해야 합니다(**SHOULD**). 예를 들어 플랫폼은 바로 구매 흐름을 시작할 때 체크아웃 상태를 미리 채울 수 있습니다.

> **참고:** 체크아웃 퍼머링크는 [REST 전송 방식 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout-rest/index.md)을 확장하는 REST 전용 구성 요소입니다. 퍼머링크에 접근하면 체크아웃 UI로 리디렉션되거나 체크아웃 페이지가 바로 렌더링됩니다.

## 스코프

체크아웃 기능은 사용자 인증 접근을 위해 다음 well-known 스코프를 정의합니다.

| 스코프                             | 설명                                                                                              |
| ---------------------------------- | ------------------------------------------------------------------------------------------------- |
| `dev.ucp.shopping.checkout:manage` | 인증된 사용자를 대신하는 모든 체크아웃 작업. 체크아웃 세션의 생성, 수정, 완료, 취소를 포함합니다. |

스코프 선언과 도출 방법, 사용자 정의 스코프로 이 집합을 확장하는 규칙은 [신원 연결 — 스코프](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/identity-linking/#scopes)에 정의되어 있습니다.

## 가이드라인

(전체 가이드라인에 더하여)

### 플랫폼

- 체크아웃 세션 진행을 돕는 데 에이전트를 활용할 수 있습니다(**MAY**)(예: 체크아웃 세션에 상품 추가, 풀필먼트 주소 선택). 단, 에이전트는 사용자가 체크아웃 세부 정보를 검토하고 주문할 수 있도록 체크아웃 세션을 신뢰할 수 있는 결정론적 UI로 넘겨야 합니다.
- 언제든지 신뢰할 수 있는 결정론적 UI에서 사용자를 다시 에이전트로 보낼 수 있습니다(**MAY**). 사용자가 장바구니에 상품을 더 담으려고 체크아웃 화면을 나가기로 한 경우가 그 예입니다.
- 요청을 에이전트가 수행했다고 플랫폼이 표시하는 경우 에이전트 컨텍스트를 제공할 수 있습니다(**MAY**).
- 체크아웃 상태가 `requires_escalation`이면 반드시 `continue_url`을 사용해야 합니다(**MUST**).
- 다른 상황에서도 `continue_url`로 비즈니스 UI에 넘길 수 있습니다(**MAY**).
- 넘길 때는 플랫폼이 구성한 체크아웃 퍼머링크보다 비즈니스가 제공한 `continue_url`을 우선해야 합니다(**SHOULD**).

### 비즈니스

- 체크아웃이 완료되면 반드시 확인 이메일을 보내야 합니다(**MUST**).
- 정확한 에러 메시지를 제공해야 합니다(**SHOULD**).
- 체크아웃 세션을 처리하는 로직은 반드시 결정론적이어야 합니다(**MUST**).
- `status` = `requires_escalation`을 반환할 때 반드시 `continue_url`을 제공해야 합니다(**MUST**).
- `status` = `requires_escalation`을 반환할 때 반드시 `severity`가 `requires_buyer_input` 또는 `requires_buyer_review`인 메시지를 하나 이상 포함해야 합니다(**MUST**).
- 종료 상태가 아닌 모든 체크아웃 응답에 `continue_url`을 제공해야 합니다(**SHOULD**).
- 체크아웃 세션이 "completed" 상태에 도달하면 변경할 수 없는 것으로 간주합니다.

## 기능 스키마 정의

| 이름         | 타입                                                                                            | 필수    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ----------------------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | any                                                                                             | **Yes** | UCP metadata for checkout responses.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| id           | string                                                                                          | **Yes** | Unique identifier of the checkout session.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| line_items   | Array\[[Line Item Response](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#line-item)\] | **Yes** | List of line items being checked out.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| buyer        | [Buyer](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#buyer)                           | No      | Representation of the buyer.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| context      | [Context](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#context)                       | No      | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals      | [Signals](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#signals)                       | No      | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| attribution  | [Attribution](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#attribution)               | No      | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| status       | string                                                                                          | **Yes** | Checkout state indicating the current phase and required action. See Checkout Status lifecycle documentation for state transition details. **Enum:** `incomplete`, `requires_escalation`, `ready_for_complete`, `complete_in_progress`, `completed`, `canceled`                                                                                                                                                                                                                                                                                                                                                                                                         |
| currency     | string                                                                                          | **Yes** | ISO 4217 currency code reflecting the merchant's market determination. Derived from address, context, and geo IP—buyers provide signals, merchants determine currency.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| totals       | [Totals](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#totals)                         | **Yes** | Different cart totals.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| messages     | Array\[[Message](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#message)\]              | No      | List of messages with error and info about the checkout session state.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| links        | Array\[[Link](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#link)\]                    | **Yes** | Links to be displayed by the platform (Privacy Policy, TOS). Mandatory for legal compliance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| expires_at   | string                                                                                          | No      | RFC 3339 expiry timestamp. Default TTL is 6 hours from creation if not sent.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| continue_url | string                                                                                          | No      | URL for checkout handoff and session recovery. MUST be provided when status is requires_escalation. See specification for format and availability requirements.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| payment      | [Payment](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#payment)                        | No      | Payment configuration containing handlers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| order        | [Order Confirmation](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#order-confirmation) | No      | Details about an order created for this checkout session.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |

## 작업

체크아웃 기능은 다음 논리적 작업을 정의합니다.

| 작업                  | 설명                                                                                |
| --------------------- | ----------------------------------------------------------------------------------- |
| **Create Checkout**   | 새 체크아웃 세션을 시작합니다. 사용자가 장바구니에 상품을 추가하는 즉시 호출합니다. |
| **Get Checkout**      | 체크아웃 세션의 현재 상태를 조회합니다.                                             |
| **Update Checkout**   | 체크아웃 세션을 수정합니다.                                                         |
| **Complete Checkout** | 체크아웃을 최종 확정하고 주문을 접수합니다.                                         |
| **Cancel Checkout**   | 체크아웃 세션을 취소합니다.                                                         |

### 체크아웃 생성(Create Checkout)

사용자가 구매 의사를 밝혔을 때(예: 구매 버튼 클릭) 플랫폼이 호출해 상품 세부 정보로 체크아웃 세션을 시작합니다.

**권장 사항**: 불일치를 줄이고 매끄러운 사용자 경험을 제공하려면, 비즈니스가 피드로 제공하는 상품 데이터(가격/제목 등)가 응답으로 반환되는 실제 속성과 일치해야 합니다(**SHOULD**).

**Inputs**

| 이름        | 타입                                                                                   | 필수    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | -------------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| line_items  | Array\[[Line Item](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#line-item)\] | **Yes** | List of line items being checked out.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| buyer       | [Buyer](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#buyer)                  | No      | Representation of the buyer.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| context     | [Context](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#context)              | No      | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals     | [Signals](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#signals)              | No      | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| attribution | [Attribution](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#attribution)      | No      | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| payment     | [Payment](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#payment)               | No      | Payment configuration containing handlers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

**Output**

This object MUST be one of the following types: [Checkout](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#checkout), [Error Response](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#error-response).

### 체크아웃 조회(Get Checkout)

체크아웃 리소스의 최신 상태를 제공합니다. 취소나 완료 후에 무엇을 반환할지는 비즈니스가 결정합니다(오래 유지되는 상태를 반환할 수도 있고, 일정 TTL이 지나 만료되어 'not found' 에러를 반환할 수도 있습니다). 플랫폼 측에서는 체크아웃 TTL을 따로 강제하지 않습니다.

플랫폼은 체크아웃 세션을 생성할 때 비즈니스가 `expires_at`으로 알려 준 TTL을 지킵니다.

**Inputs**

| 이름 | 타입   | 필수    | 설명                                                           |
| ---- | ------ | ------- | -------------------------------------------------------------- |
| id   | string | **Yes** | The unique identifier of the checkout session.Defined in path. |

**Output**

This object MUST be one of the following types: [Checkout](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#checkout), [Error Response](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#error-response).

### 체크아웃 수정(Update Checkout)

체크아웃 리소스를 통째로 교체합니다. 플랫폼은 반드시 쓰기 전용 데이터 필드의 갱신 내용을 포함한 체크아웃 리소스 전체를 보내야 합니다(**REQUIRED**). 요청에 담긴 리소스가 비즈니스 측의 기존 체크아웃 세션 상태를 대체합니다.

**Inputs**

| 이름        | 타입                                                                                   | 필수    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | -------------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id          | string                                                                                 | **Yes** | The unique identifier of the checkout session.Defined in path.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| line_items  | Array\[[Line Item](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#line-item)\] | No      | List of line items being checked out.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| buyer       | [Buyer](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#buyer)                  | No      | Representation of the buyer.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| context     | [Context](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#context)              | No      | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals     | [Signals](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#signals)              | No      | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| attribution | [Attribution](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#attribution)      | No      | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| payment     | [Payment](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#payment)               | No      | Payment configuration containing handlers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

**Output**

This object MUST be one of the following types: [Checkout](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#checkout), [Error Response](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#error-response).

### 체크아웃 완료(Complete Checkout)

체크아웃에서 주문을 최종 접수하는 호출입니다. 사용자가 선택한 상품을 결제하고 주문하기로 확정했을 때 호출합니다. 이 호출의 응답은 `order` 필드가 채워진 체크아웃 객체입니다. 반환된 `order`는 접수된 주문의 전체 상태를 참조하는 데 쓸 수 있는 `id`, `permalink_url` 같은 필수 식별자를 제공합니다. 주문을 저장할 때 `Checkout`의 필드로 주문 표현을 구성할 수 있습니다(**MAY**) (예: `line_items`, `fulfillment` 같은 정보로 초기 주문 표현을 만듭니다).

이 호출 이후에는 주문과 관련 상품이 공급망을 거쳐 이동하면서 다른 세부 정보가 후속 이벤트로 갱신됩니다.

**Inputs**

| 이름        | 타입                                                                              | 필수    | 설명                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------- | --------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id          | string                                                                            | **Yes** | The unique identifier of the checkout session.Defined in path.                                                                                                                                                                                                                                                                                                                                          |
| signals     | [Signals](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#signals)         | No      | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace. |
| attribution | [Attribution](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#attribution) | No      | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                        |
| payment     | [Payment](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#payment)          | No      | Payment configuration containing handlers.                                                                                                                                                                                                                                                                                                                                                              |

**Output**

This object MUST be one of the following types: [Checkout](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#checkout), [Error Response](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#error-response).

### 체크아웃 취소(Cancel Checkout)

이 작업으로 취소 가능한 체크아웃 세션을 취소합니다. 체크아웃 세션을 취소할 수 없으면(예: 체크아웃 세션이 이미 취소되었거나 완료됨) 비즈니스는 작업이 허용되지 않는다는 에러를 반환해야 합니다(**SHOULD**). 상태가 `completed` 또는 `canceled`가 아닌 체크아웃 세션은 모두 취소할 수 있어야 합니다(**SHOULD**).

**Inputs**

| 이름 | 타입   | 필수    | 설명                                                           |
| ---- | ------ | ------- | -------------------------------------------------------------- |
| id   | string | **Yes** | The unique identifier of the checkout session.Defined in path. |

**Output**

This object MUST be one of the following types: [Checkout](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#checkout), [Error Response](/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#error-response).

## 전송 방식 바인딩

위의 추상 작업은 아래와 같이 특정 전송 프로토콜에 바인딩됩니다.

- [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout-rest/index.md): 표준 HTTP 메서드와 JSON 페이로드를 사용하는 RESTful API 매핑.
- [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout-mcp/index.md): 에이전트 상호작용을 위한 Model Context Protocol 매핑.
- [A2A 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout-a2a/index.md): 에이전트 상호작용을 위한 Agent-to-Agent Protocol 매핑.
- [임베디드 체크아웃 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/embedded-checkout/index.md): 임베디드 체크아웃을 구동하기 위한 JSON-RPC.

## 엔티티

### 구매자

| 이름         | 타입   | 필수 | 설명                     |
| ------------ | ------ | ---- | ------------------------ |
| first_name   | string | No   | First name of the buyer. |
| last_name    | string | No   | Last name of the buyer.  |
| email        | string | No   | Email of the buyer.      |
| phone_number | string | No   | E.164 standard.          |

### 컨텍스트

컨텍스트 신호는 잠정적인 값이며 권위 있는 데이터가 아닙니다. 비즈니스는 검증된 입력(예: 배송 주소)이 없을 때 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정, 위험 탐지)나 규제 제약(수출 통제)과 맞지 않으면 무시하거나 우선순위를 낮출 수 있습니다(MAY). 자격 및 정책 적용은 반드시 체크아웃 시점의 구속력 있는 거래 데이터를 기준으로 해야 합니다(MUST).

| 이름            | 타입                                                                                                       | 필수 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| --------------- | ---------------------------------------------------------------------------------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| address_country | string                                                                                                     | No   | The country. Recommended to be in 2-letter ISO 3166-1 alpha-2 format, for example "US". For backward compatibility, a 3-letter ISO 3166-1 alpha-3 country code such as "SGP" or a full country name such as "Singapore" can also be used. Optional hint for market context (currency, availability, pricing)—higher-resolution data (e.g., shipping address) supersedes this value.                                                                                |
| address_region  | string                                                                                                     | No   | The region in which the locality is, and which is in the country. For example, California or another appropriate first-level Administrative division. Optional hint for progressive localization—higher-resolution data (e.g., shipping address) supersedes this value.                                                                                                                                                                                            |
| postal_code     | string                                                                                                     | No   | The postal code. For example, 94043. Optional hint for regional refinement—higher-resolution data (e.g., shipping address) supersedes this value.                                                                                                                                                                                                                                                                                                                  |
| intent          | string                                                                                                     | No   | Background context describing buyer's intent (e.g., 'looking for a gift under $50', 'need something durable for outdoor use'). Informs relevance, recommendations, and personalization.                                                                                                                                                                                                                                                                            |
| language        | string                                                                                                     | No   | Preferred language for content. Use IETF BCP 47 language tags (e.g., 'en', 'fr-CA', 'zh-Hans'). For REST, equivalent to Accept-Language header—platforms SHOULD fall back to Accept-Language when this field is absent; when provided, overrides Accept-Language. Businesses MAY return content in a different language if unavailable.                                                                                                                            |
| currency        | string                                                                                                     | No   | Preferred currency (ISO 4217, e.g., 'EUR', 'USD'). Businesses determine presentment currency from context and authoritative signals; this hint MAY inform selection in multi-currency markets. Also serves as the denomination for price filter values — platforms SHOULD include this field when sending price filters. Response prices include explicit currency confirming the resolution.                                                                      |
| eligibility     | Array\[[Reverse Domain Name](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#reverse-domain-name)\] | No   | Buyer claims about eligible benefits such as loyalty membership, payment instrument perks, and similar. Recognized claims MAY inform the Business response (e.g., member-only product availability, adjusted pricing in catalog, provisional discounts at cart or checkout). Businesses MUST ignore unrecognized values without error. Values MUST use reverse-domain naming (e.g., 'com.example.loyalty_gold', 'org.school.student') and MUST be non-identifying. |

### 신호

승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터입니다. `context`(구매자가 주장한 선호)나 `buyer`(자기 신고 신원)와 달리, 신호 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰한 내용을 바탕으로 하거나, 독립적으로 검증할 수 있는 제3자 증명을 전달하는 방식으로 신호를 제공합니다. 자세한 내용과 프라이버시 요구 사항은 [신호](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/#signals)를 참고하세요.

| 이름               | 타입   | 필수 | 설명                                           |
| ------------------ | ------ | ---- | ---------------------------------------------- |
| dev.ucp.buyer_ip   | string | No   | Client's IP address (IPv4 or IPv6).            |
| dev.ucp.user_agent | string | No   | Client's HTTP User-Agent header or equivalent. |

### 어트리뷰션

플랫폼이 제공하는 유입 경로 및 전환 이벤트 컨텍스트입니다. 플랫폼이 전달하는 캠페인 ID, 클릭 식별자, 소스/매체 표식이 여기에 해당합니다. 자세한 내용과 동의 요구 사항은 [어트리뷰션](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/#attribution)을 참고하세요.

Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.

### 풀필먼트 옵션

A fulfillment option within a group (e.g., Standard Shipping $5, Express $15).

| 이름                      | 타입          | 필수    | 설명                                                                       |
| ------------------------- | ------------- | ------- | -------------------------------------------------------------------------- |
| id                        | string        | **Yes** | Unique fulfillment option identifier.                                      |
| title                     | string        | **Yes** | Short label (e.g., 'Express Shipping', 'Curbside Pickup').                 |
| description               | string        | No      | Complete context for buyer decision (e.g., 'Arrives Dec 12-15 via FedEx'). |
| carrier                   | string        | No      | Carrier name (for shipping).                                               |
| earliest_fulfillment_time | string        | No      | Earliest fulfillment date.                                                 |
| latest_fulfillment_time   | string        | No      | Latest fulfillment date.                                                   |
| totals                    | Array[object] | **Yes** | Fulfillment option totals breakdown.                                       |

### 상품

#### 상품 생성 요청

| 이름 | 타입   | 필수    | 설명                                                                                                                                                                        |
| ---- | ------ | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id   | string | **Yes** | The product identifier, often the SKU, required to resolve the product details associated with this line item. Should be recognized by both the Platform, and the Business. |

#### 상품 수정 요청

| 이름 | 타입   | 필수    | 설명                                                                                                                                                                        |
| ---- | ------ | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id   | string | **Yes** | The product identifier, often the SKU, required to resolve the product details associated with this line item. Should be recognized by both the Platform, and the Business. |

#### 상품

| 이름      | 타입                                                                    | 필수    | 설명                                                                                                                                                                        |
| --------- | ----------------------------------------------------------------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id        | string                                                                  | **Yes** | The product identifier, often the SKU, required to resolve the product details associated with this line item. Should be recognized by both the Platform, and the Business. |
| title     | string                                                                  | **Yes** | Product title.                                                                                                                                                              |
| price     | [Amount](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#amount) | **Yes** | Unit price in ISO 4217 minor units.                                                                                                                                         |
| image_url | string                                                                  | No      | Product image URI.                                                                                                                                                          |

### 라인 아이템

#### 라인 아이템 생성 요청

| 이름     | 타입                                                                | 필수    | 설명                                  |
| -------- | ------------------------------------------------------------------- | ------- | ------------------------------------- |
| item     | [Item](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#item) | **Yes** |                                       |
| quantity | integer                                                             | **Yes** | Quantity of the item being purchased. |

#### 라인 아이템 수정 요청

| 이름      | 타입                                                                | 필수    | 설명                                                   |
| --------- | ------------------------------------------------------------------- | ------- | ------------------------------------------------------ |
| id        | string                                                              | No      |                                                        |
| item      | [Item](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#item) | **Yes** |                                                        |
| quantity  | integer                                                             | **Yes** | Quantity of the item being purchased.                  |
| parent_id | string                                                              | No      | Parent line item identifier for any nested structures. |

#### 라인 아이템

| 이름      | 타입                                                                           | 필수    | 설명                                                   |
| --------- | ------------------------------------------------------------------------------ | ------- | ------------------------------------------------------ |
| id        | string                                                                         | **Yes** |                                                        |
| item      | [Item](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#item)            | **Yes** |                                                        |
| quantity  | integer                                                                        | **Yes** | Quantity of the item being purchased.                  |
| totals    | Array\[[Total](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#total)\] | **Yes** | Line item totals breakdown.                            |
| parent_id | string                                                                         | No      | Parent line item identifier for any nested structures. |

### 링크

| 이름  | 타입   | 필수    | 설명                                                                                                                                                                                                                                 |
| ----- | ------ | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| type  | string | **Yes** | Type of link. Well-known values: `privacy_policy`, `terms_of_service`, `refund_policy`, `shipping_policy`, `faq`. Consumers SHOULD handle unknown values gracefully by displaying them using the `title` field or omitting the link. |
| url   | string | **Yes** | The actual URL pointing to the content to be displayed.                                                                                                                                                                              |
| title | string | No      | Optional display text for the link. When provided, use this instead of generating from type.                                                                                                                                         |

#### Well-Known 링크 유형

비즈니스는 거래와 관련된 링크를 모두 제공해야 합니다(**SHOULD**). 권장하는 well-known 유형은 다음과 같습니다.

| 유형               | 설명                              |
| ------------------ | --------------------------------- |
| `privacy_policy`   | 비즈니스의 개인정보 처리방침 링크 |
| `terms_of_service` | 비즈니스의 서비스 약관 링크       |
| `refund_policy`    | 비즈니스의 환불 정책 링크         |
| `shipping_policy`  | 비즈니스의 배송 정책 링크         |
| `faq`              | 비즈니스의 자주 묻는 질문 링크    |

비즈니스는 도메인별 필요에 맞게 사용자 정의 유형을 정의할 수 있습니다(**MAY**). 플랫폼은 알 수 없는 유형을 `title` 필드로 표시하거나 생략하는 식으로 적절히 처리해야 합니다(**SHOULD**).

### 메시지

This object MUST be one of the following types: [Message Error](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#message-error), [Message Warning](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#message-warning), [Message Info](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#message-info).

### 에러 메시지

| 이름         | 타입                                                                            | 필수    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------ | ------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                          | **Yes** | **Constant = error**. Message type discriminator.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| code         | [Error Code](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#error-code) | **Yes** | Error code identifying the type of error. Standard errors are defined in specification (see examples), and have standardized semantics; freeform codes are permitted.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| path         | string                                                                          | No      | RFC 9535 JSONPath to the component the message refers to (e.g., $.items[1]).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| content_type | string                                                                          | No      | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| content      | string                                                                          | **Yes** | Human-readable message.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| severity     | string                                                                          | **Yes** | Reflects the resource state and recommended action. 'recoverable': platform can resolve by modifying inputs and retrying via API. 'requires_buyer_input': merchant requires information their API doesn't support collecting programmatically (checkout incomplete). 'requires_buyer_review': buyer must authorize before order placement due to policy, regulatory, or entitlement rules. 'unrecoverable': no valid resource exists to act on, retry with new resource or inputs. Errors with 'requires\_*' severity contribute to 'status: requires_escalation'.* *Enum:*\* `recoverable`, `requires_buyer_input`, `requires_buyer_review`, `unrecoverable` |

#### 에러 코드

Error code identifying the type of error. Standard errors are defined in specification (see examples), and have standardized semantics; freeform codes are permitted.

### 정보 메시지

| 이름         | 타입                                                                          | 필수    | 설명                                                                                                                                                                                   |
| ------------ | ----------------------------------------------------------------------------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                        | **Yes** | **Constant = info**. Message type discriminator.                                                                                                                                       |
| path         | string                                                                        | No      | RFC 9535 JSONPath to the component the message refers to.                                                                                                                              |
| code         | [Info Code](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#info-code) | No      | Info code identifying the type of informational message. Standard codes are defined in capability specs (see examples), and have standardized semantics; freeform codes are permitted. |
| content_type | string                                                                        | No      | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                         |
| content      | string                                                                        | **Yes** | Human-readable message.                                                                                                                                                                |

### 경고 메시지

| 이름         | 타입                                                                                | 필수    | 설명                                                                                                                                                                                                                                                |
| ------------ | ----------------------------------------------------------------------------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                              | **Yes** | **Constant = warning**. Message type discriminator.                                                                                                                                                                                                 |
| path         | string                                                                              | No      | JSONPath (RFC 9535) to related field (e.g., $.line_items[0]).                                                                                                                                                                                       |
| code         | [Warning Code](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#warning-code) | **Yes** | Warning code identifying the type of warning. Standard codes are defined in capability specs (see examples), and have standardized semantics; freeform codes are permitted.                                                                         |
| content      | string                                                                              | **Yes** | Human-readable warning message that MUST be displayed.                                                                                                                                                                                              |
| content_type | string                                                                              | No      | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                                                                                      |
| presentation | string                                                                              | No      | Rendering contract for this warning. 'notice' (default): platform MUST display, MAY dismiss. 'disclosure': platform MUST display in proximity to the path-referenced component, MUST NOT hide or auto-dismiss. See specification for full contract. |
| image_url    | string                                                                              | No      | URL to a required visual element (e.g., warning symbol, energy class label).                                                                                                                                                                        |
| url          | string                                                                              | No      | Reference URL for more information (e.g., regulatory site, registry entry, policy page).                                                                                                                                                            |

### 결제

| 이름        | 타입                                                                                                                                                             | 필수 | 설명                                                                                                                                                                                                                       |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| instruments | Array\[[Payment Instrument Selected Payment Instrument](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#payment-instrument-selected-payment-instrument)\] | No   | The payment instruments available for this payment. Each instrument is associated with a specific handler via the handler_id field. Handlers can extend the base payment_instrument schema to add handler-specific fields. |

#### 선택된 결제 수단

A payment instrument with selection state.

| 이름            | 타입    | 필수    | 설명                                                                                                                                                         |
| --------------- | ------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| id              | string  | **Yes** | A unique identifier for this instrument instance, assigned by the platform.                                                                                  |
| handler_id      | string  | **Yes** | The unique identifier for the handler instance that produced this instrument. This corresponds to the 'id' field in the Payment Handler definition.          |
| type            | string  | **Yes** | The broad category of the instrument (e.g., 'card', 'tokenized_card'). Specific schemas will constrain this to a constant value.                             |
| billing_address | object  | No      | The billing address associated with this payment method.                                                                                                     |
| credential      | object  | No      | The base definition for any payment credential. Handlers define specific credential types.                                                                   |
| display         | object  | No      | Display information for this payment instrument. Each payment instrument schema defines its specific display properties, as outlined by the payment handler. |
| selected        | boolean | No      | Whether this instrument is selected by the user.                                                                                                             |

### 결제 자격 증명

| 이름 | 타입   | 필수    | 설명                                                                                         |
| ---- | ------ | ------- | -------------------------------------------------------------------------------------------- |
| type | string | **Yes** | The credential type discriminator. Specific schemas will constrain this to a constant value. |

### 우편 주소

| 이름             | 타입   | 필수 | 설명                                                                                                                                                                                                                                      |
| ---------------- | ------ | ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| extended_address | string | No   | An address extension such as an apartment number, C/O or alternative name.                                                                                                                                                                |
| street_address   | string | No   | The street address.                                                                                                                                                                                                                       |
| address_locality | string | No   | The locality in which the street address is, and which is in the region. For example, Mountain View.                                                                                                                                      |
| address_region   | string | No   | The region in which the locality is, and which is in the country. Required for applicable countries (i.e. state in US, province in CA). For example, California or another appropriate first-level Administrative division.               |
| address_country  | string | No   | The country. Recommended to be in 2-letter ISO 3166-1 alpha-2 format, for example "US". For backward compatibility, a 3-letter ISO 3166-1 alpha-3 country code such as "SGP" or a full country name such as "Singapore" can also be used. |
| postal_code      | string | No   | The postal code. For example, 94043.                                                                                                                                                                                                      |
| first_name       | string | No   | Optional. First name of the contact associated with the address.                                                                                                                                                                          |
| last_name        | string | No   | Optional. Last name of the contact associated with the address.                                                                                                                                                                           |
| phone_number     | string | No   | Optional. Phone number of the contact associated with the address.                                                                                                                                                                        |

### 응답

Capability reference in responses. Only name/version required to confirm active capabilities.

| 이름    | 타입    | 필수    | 설명                                                                                                                            |
| ------- | ------- | ------- | ------------------------------------------------------------------------------------------------------------------------------- |
| version | string  | **Yes** | Entity version in YYYY-MM-DD format.                                                                                            |
| spec    | string  | No      | URL to human-readable specification document.                                                                                   |
| schema  | string  | No      | URL to JSON Schema defining this entity's structure and payloads.                                                               |
| id      | string  | No      | Unique identifier for this entity instance. Used to disambiguate when multiple instances exist.                                 |
| config  | object  | No      | Entity-specific configuration. Structure defined by each entity's schema.                                                       |
| extends | OneOf[] | No      | Parent capability(s) this extends. Present for extensions, absent for root capabilities. Use array for multi-parent extensions. |

### 합계

| 이름         | 타입                                                                                  | 필수    | 설명                                                                                                                                                                                                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                                | **Yes** | Cost category. Well-known values: subtotal, items_discount, discount, fulfillment, tax, fee, total. Businesses MAY use additional values.                                                                                                                                                   |
| display_text | string                                                                                | No      | Text to display against the amount. Should reflect appropriate method (e.g., 'Shipping', 'Delivery').                                                                                                                                                                                       |
| amount       | [Signed Amount](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#signed-amount) | **Yes** | Monetary amount in the currency's minor unit as defined by ISO 4217. Refer to the currency's exponent to determine minor-to-major ratio (e.g., 2 for USD, 0 for JPY, 3 for KWD). May be negative — the sign is intrinsic to the value (e.g., discounts are negative, charges are positive). |

#### 렌더링 계약

제시되는 합계의 내용과 순서에 대한 권한은 비즈니스에 있습니다. 올바른 표시 방식은 비즈니스가 지켜야 하는 지역, 상품, 규제 요구 사항(예: 다중 관할권 세금 항목화, 필수 수수료 고지)에 따라 달라지기 때문입니다.

플랫폼은 반드시 모든 최상위 항목을 제공된 순서대로 렌더링해야 합니다(MUST).

```python
for entry in totals:
    render_line(entry.display_text, entry.amount)
```

플랫폼은 하위 라인을 보조 세부 정보로 렌더링할 수 있습니다(MAY).

```python
for entry in totals:
    render_line(entry.display_text, entry.amount)
    if entry.lines:
        for sub in entry.lines:
            render_detail_line(sub.display_text, sub.amount)
```

플랫폼은 항목을 해석, 필터링, 재정렬, 집계하거나 자체 표시 로직을 적용해서는 안 됩니다(MUST NOT).

`totals[]`의 불변 조건은 다음과 같습니다.

- 모든 항목에는 `type`과 `amount`가 있습니다. 플랫폼은 `display_text`가 제공되면 이를 사용해야 합니다(SHOULD). well-known 유형에는 대체용 기본 표시 라벨이 있습니다(아래 표 참고). 알 수 없는 유형은 반드시 `display_text`를 포함해야 합니다(MUST).
- 금액은 부호 있는 정수입니다. 음수 값은 차감(예: 할인), 양수 값은 가산을 뜻합니다. 부호가 곧 방향입니다.
- `type: "subtotal"`은 반드시 정확히 하나 있어야 합니다(MUST).
- `type: "total"`은 반드시 정확히 하나 있어야 합니다(MUST).

#### 검증

플랫폼은 비즈니스의 값을 자체 계산한 합계로 바꿔서는 안 됩니다(MUST NOT). 플랫폼은 제공된 합계를 검증할 수 있습니다(MAY).

```python
assert sum(e.amount for e in totals if e.type != "total") == total_entry.amount
```

계산한 합이 `type: "total"` 항목과 맞지 않더라도 플랫폼은 렌더링 결과를 바꿔서는 안 됩니다(MUST NOT). 표시에서는 비즈니스가 제시한 합계가 권위를 가집니다. 그러나 플랫폼은 합계가 맞지 않는 체크아웃을 자율적으로 완료해서는 안 됩니다(MUST NOT). 플랫폼은 체크아웃을 거부하거나, 에스컬레이션해서 `continue_url`로 구매자의 검토를 요청해야 합니다(SHOULD).

#### Well-Known 유형

| 유형             | 부호 | 기본 라벨      | 의미                             |
| ---------------- | ---- | -------------- | -------------------------------- |
| `subtotal`       | +    | Subtotal       | 라인 아이템 가격의 합            |
| `discount`       | −    | Discount       | 주문 또는 라인 아이템 수준 할인  |
| `items_discount` | −    | Item Discounts | 라인 아이템 할인의 합산          |
| `fulfillment`    | +    | Shipping       | 배송, 배달, 픽업 요금            |
| `tax`            | +    | Tax            | 세금                             |
| `fee`            | +    | Fee            | 수수료 및 추가 요금              |
| `total`          | =    | Total          | 권위 있는 최종 합계(정확히 하나) |

`display_text`가 제공되면 플랫폼은 반드시 이를 사용해야 합니다(MUST). well-known 유형에서 생략되었다면 플랫폼은 위의 기본 라벨을 사용해야 합니다(SHOULD). well-known 유형의 부호 규칙은 스키마가 강제합니다. 차감 유형(discount, items_discount)은 반드시 음수 금액이어야 하고(MUST), 가산 유형(subtotal, fulfillment, tax, fee)은 반드시 음수가 아닌 금액이어야 합니다(MUST).

`type` 필드는 개방형 문자열이므로 비즈니스는 well-known 집합에 없는 값도 사용할 수 있습니다(MAY). 알 수 없는 유형은 반드시 `display_text`를 포함해야 하며(MUST)(스키마가 강제함), 금액의 부호가 의미를 나타냅니다.

#### 반복 유형

`subtotal`과 `total`을 제외한 모든 유형은 여러 번 나타날 수 있습니다(MAY). 다중 관할권 세금 라인이나 항목별 수수료가 그 예입니다.

#### 하위 라인(`lines`)

각 최상위 항목에는 `lines` 배열이 있을 수 있습니다(MAY). 하위 라인은 최상위 항목과 같은 기본 형태(`display_text`와 `amount`)로, 상위 항목 아래에 항목별 내역을 제공합니다.

**불변 조건:** `sum(lines[].amount)`는 반드시 상위 항목의 `amount`와 같아야 합니다(MUST).

반드시 렌더링해야 하는 것(MUST, 최상위 항목)과 선택적으로 노출할 수 있는 것(MAY, 하위 라인)은 비즈니스가 결정합니다. 플랫폼은 하위 라인이 제공되면 렌더링해야 합니다(SHOULD).

#### 예시

**최상위에서 항목화한 분할 세금:**

```json
"totals": [
  { "type": "subtotal",    "display_text": "Subtotal",    "amount": 5750 },
  { "type": "fulfillment", "display_text": "Shipping",    "amount": 899 },
  { "type": "tax",         "display_text": "Federal Tax", "amount": 332 },
  { "type": "tax",         "display_text": "State Tax",   "amount": 465 },
  { "type": "total",       "display_text": "Total",       "amount": 7446 }
]
```

**선택적 내역이 포함된 통합 수수료:**

```json
"totals": [
  { "type": "subtotal", "display_text": "Subtotal", "amount": 4999 },
  {
    "type": "fee", "display_text": "Fees", "amount": 549,
    "lines": [
      { "display_text": "Service Fee", "amount": 399 },
      { "display_text": "Recycling Fee", "amount": 150 }
    ]
  },
  { "type": "tax",   "display_text": "Tax",   "amount": 444 },
  { "type": "total", "display_text": "Total", "amount": 5992 }
]
```

**할인과 계정 크레딧 — 음수 금액:**

```json
"totals": [
  { "type": "subtotal",       "display_text": "Subtotal",       "amount": 10000 },
  { "type": "discount",       "display_text": "Summer Sale",    "amount": -1500 },
  { "type": "tax",            "display_text": "Tax",            "amount": 680 },
  { "type": "account_credit", "display_text": "Account Credit", "amount": -2500 },
  { "type": "total",          "display_text": "Amount Due",     "amount": 6680 }
]
```

### UCP 체크아웃 응답

UCP metadata for checkout responses.

| 이름             | 타입   | 필수    | 설명                                                                        |
| ---------------- | ------ | ------- | --------------------------------------------------------------------------- |
| version          | string | **Yes** | UCP version in YYYY-MM-DD format.                                           |
| status           | string | No      | Application-level status of the UCP operation. **Enum:** `success`, `error` |
| services         | object | No      | Service registry keyed by reverse-domain name.                              |
| capabilities     | object | No      | Capability registry keyed by reverse-domain name.                           |
| payment_handlers | object | **Yes** | Payment handler registry keyed by reverse-domain name.                      |
| services         | any    | No      |                                                                             |
| capabilities     | any    | No      |                                                                             |
| payment_handlers | any    | **Yes** |                                                                             |

### 주문 확인

| 이름          | 타입   | 필수    | 설명                                                                                   |
| ------------- | ------ | ------- | -------------------------------------------------------------------------------------- |
| id            | string | **Yes** | Unique order identifier.                                                               |
| label         | string | No      | Human-readable label for identifying the order. MUST only be provided by the business. |
| permalink_url | string | **Yes** | Permalink to access the order on merchant site.                                        |

### 에러 응답

| 이름         | 타입                                                                               | 필수    | 설명                                                              |
| ------------ | ---------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------------- |
| ucp          | any                                                                                | **Yes** | UCP protocol metadata. Status MUST be 'error' for error response. |
| messages     | Array\[[Message](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#message)\] | **Yes** | Array of messages describing why the operation failed.            |
| continue_url | string                                                                             | No      | URL for buyer handoff or session recovery.                        |
