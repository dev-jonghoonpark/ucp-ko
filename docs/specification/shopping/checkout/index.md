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

# 체크아웃 기능 { #checkout-capability }

* **기능 이름:** `dev.ucp.shopping.checkout`

## 개요 { #overview }

플랫폼이 체크아웃 세션을 진행할 수 있게 한다. AP2 위임(Mandate) 확장이 지원되지
않는 한, 체크아웃은 사용자가 신뢰할 수 있는 UI를 통해 직접 최종 확정해야 한다.

비즈니스는 계속 Merchant of Record(MoR)로 남으며, 이 기능을 통해 카드 결제를 받기
위해 PCI DSS 인증을 받을 필요가 없다.

### 흐름 개요 { #flow-overview }

![상위 수준 체크아웃 흐름 시퀀스 다이어그램](site:specification/images/ucp-checkout-flow.png)

### 결제 { #payments }

결제 핸들러는 `/.well-known/ucp`에 있는 비즈니스의 UCP 프로필과
checkout.ucp.payment_handlers에서 디스커버리된다. 핸들러는 결제 수단(예: Google Pay,
Shop Pay)을 수집하기 위한 처리 사양을 정의한다. 구매자가 결제를 제출하면 플랫폼은
수집한 결제 수단 데이터로 `payment.instruments` 배열을 채운다.

`payment` 객체는 체크아웃 생성 시 선택 사항이며, 결제 처리가 필요 없는 사용 사례
(예: 견적 생성, 장바구니 관리)에서는 생략할 수 있다.

비즈니스는 응답의 `payment.instruments`를 자체 사용자 상태에 있는 표시 안전
(display-safe) 결제 수단으로 채울 수도 있다. 예를 들어 인증된 구매자가 해당 비즈니스에
저장해 둔 결제 수단이 이에 해당한다. 이러한 결제 수단에는 원본 자격 증명이 담기지 않는다.
이는 구매자가 선택했을 때 비즈니스가 서버 측에서 해석하는, 표시 안전하고 비즈니스
범위로 한정된 참조(불투명한 `id`와 `display` 필드)다.
[체크아웃 생성 REST 예시](rest.md#create-checkout)는 이미 Shop Pay 결제 수단에 대해
이 형태를 반환한다. 사용자별로 저장된 상태를 반환하려면 사용자 인증 요청이 필요하다.
적용되는 제한 사항은
[신원 연결](../../common/identity-linking/index.md#business-populated-response-values)을
참고한다.

`display` 필드를 통해 플랫폼은 저장된 결제 수단을 표시하고, 구매자는 이를 알아볼 수
있다. 현재 기본 스키마는 결제 수단을 그대로 청구할 수 있는지, 아니면 여전히 자격 증명
수집이 필요한지 구분하지 않는다. 따라서 플랫폼은 `credential`이 없다는 점이나 `id`
명명 방식으로부터 결제 준비 여부를 추론하지 않아야 한다.

### 풀필먼트 { #fulfillment }

UCP에서 풀필먼트는 다양한 사용 사례를 수용하기 위해 확장으로 모델링된다.

체크아웃 객체에서 풀필먼트는 선택 사항이다. 이는 플랫폼이 실물 상품에 더 적합한
풀필먼트 세부 정보를 제공하지 않고도 디지털 상품의 체크아웃을 수행할 수 있게 하기
위함이다.

### 수량과 판매 기준 { #quantity-and-sale-basis }

체크아웃은 기본(`C62`, `0`) 식별자를 포함하여 공통
[수량과 단위](../../overview/index.md#quantities-and-units) 계약을 적용한다. 비즈니스는
거래에 대한 각 상품의 권위 있는 판매 기준을 결정한다. 비즈니스의 응답이 권위를
가지며, 각 `line_items[].quantity`는 해당 기준에서의 정수 스텝 수다.

**판매 기준 디스커버리.** 판매 기준은 상품 데이터다. 플랫폼은 거래 전에
[카탈로그](../catalog/index.md#sale-basis-and-quantity-units)에서 상품의
`quantity_unit`(`scale`과 `increment` 포함)을 디스커버리해야 한다(**SHOULD**).
카탈로그 정보가 없는 플랫폼은 서술자를 생략할 수 있다. 이 경우 비즈니스는 자신의
권위 있는 기준을 적용하고 응답 라인에서 이를 확인해 주며, 플랫폼은 되돌아온 서술자를
검토하고 그 해석이 의도한 것과 다르면 이제 알게 된 기준으로 수량을 다시 제출한다.
요청에 `quantity_unit`을 명시하는 것(아래 참고)은 플랫폼이 이전에 디스커버리한 기준이
여전히 유효한지 확인하는 방법이다. 기준을 모르는 플랫폼은 추측한 값을 명시하지 말고
서술자를 생략한다.

**수량 요청.** 플랫폼 요청에서 `line_items[].item.quantity_unit`을 생략하면,
플랫폼은 판매 기준 식별자에 대해 아무것도 명시하지 않은 것이 된다. 비즈니스는 상품의
권위 있는 판매 기준을 사용해 `quantity`를 해석한다. 따라서 계량 단위로 판매되는 상품에
대한 요청도 기본 식별자를 명시하지 않고 서술자를 생략할 수 있다.

플랫폼은 요청 라인에 `quantity_unit`을 포함하여 자신이 주문한다고 생각하는 기준을
명시할 수 있다(**MAY**). 플랫폼이 이를 포함하면 비즈니스는 반드시 명시된
[기계 식별자](../../overview/index.md#quantities-and-units)와 권위 있는 기계 식별자를
비교해야 한다(**MUST**). `unit: "C62"`와 실효 `scale` `0`을 명시적으로 지정한 것은
서술자가 없을 때 나타내는 기본 식별자와 일치한다. `display_text`의 차이는 불일치가
아니다.

명시된 `quantity_unit`이 상품의 권위 있는 판매 기준과 일치하지 않으면, 요청한 수량은
비즈니스가 판매하지 않는 기준으로 표시된 것이다. 조용한 변환은 양방향 모두 금지된다.
어느 쪽도 상대가 지정한 수량을 변경 사실을 드러내지 않고 재해석해서는 안 된다.
비즈니스는 다음 두 가지 방법 중 하나로 불일치를 해결한다:

* **눈에 보이게 변환한다.** 비즈니스가 명시된 기준을 자신의 권위 있는 기준으로 변환할
    수 있는 경우(예: 파운드를 킬로그램으로), 요청을 눈에 보이는 라인 수정으로 적용할 수
    있다(**MAY**). 라인에는 권위 있는 서술자와 변환된 수량이 담긴다. 변환된 수량은
    비즈니스의 규칙에 따라 권위 있는 `scale`로 한 번 반올림하고, 주문 증분 정책이 있으면
    변환 후에 적용한다. 응답에는 해당 라인에 대한 경고 `messages[]` 항목이 포함된다.
    UCP는 변환 계수나 차원을 정의하지 않으며, 변환 여부는 비즈니스가 스스로 판단한다.
* **복구 가능하게 거부한다.** 변환할 수 없거나 변환하지 않기로 한 경우(예: 리터로
    표시된 체결 부품 수량), 비즈니스는 반드시 권위 있는 기준을 명시한 복구 가능한
    비즈니스 결과로 해당 라인에 대한 요청의 효과를 거부해야 한다(**MUST**). 수정 시에는
    라인이 이전의 권위 있는 상태로 유지된다. 생성 시에는 라인이 생성되지 않으며 메시지의
    `content`에서 해당 상품을 식별한다. 생성할 수 있는 라인이 하나도 없으면 비즈니스는
    대신 에러 응답을 반환할 수 있다(**MAY**).

두 경우 모두 응답은 거부된 입력을 되돌려 주는 것이 아니라 권위 있는 상태를 반영하며,
응답 반향 규칙에 따라 존재하는 모든 라인은 권위 있는 판매 기준을 담는다. 플랫폼은
되돌아온 서술자를 읽거나 카탈로그를 다시 읽은 뒤 재제출하는 방식으로 복구한다. 메시지
`content`는 사람을 위한 설명 텍스트이며, 복구 입력으로 사용해서는 안 된다.

예를 들어 한 산업용 공급업체가 낱개 체결 부품을 킬로그램 단위로 판매하는데, 오래된
데이터 때문에 플랫폼은 이를 파운드 단위로 판매한다고 믿고 있다. 플랫폼은 그 기준을
명시한 수정 요청을 제출한다:

<!-- ucp:example schema=shopping/checkout op=update direction=request -->
```json
{
  "line_items": [
    {
      "id": "li_fasteners",
      "item": {
        "id": "var_fasteners",
        "quantity_unit": { "unit": "LBR", "scale": 2, "display_text": "lb" }
      },
      "quantity": 275
    }
  ]
}
```

변환하는 비즈니스는 수정 요청을 눈에 보이는 수정으로 적용한다. 라인은 권위 있는
기준으로 표시되며(요청한 2.75 lb는 1.25 kg으로 변환되고 `scale` 2로 한 번
반올림된다), 경고가 수정 사실을 표시한다:

<!-- ucp:example schema=shopping/checkout op=read -->
```json
{
  "ucp": { "version": "{{ ucp_version }}", "status": "success", "payment_handlers": {} },
  "id": "chk_fasteners_1",
  "status": "incomplete",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_fasteners",
      "item": {
        "id": "var_fasteners",
        "title": "Stainless Steel Fasteners",
        "price": 1299,
        "quantity_unit": { "unit": "KGM", "scale": 2, "display_text": "kg" }
      },
      "quantity": 125,
      "totals": [
        { "type": "subtotal", "amount": 1624 },
        { "type": "total", "amount": 1624 }
      ]
    }
  ],
  "messages": [
    {
      "type": "warning",
      "code": "quantity_unit_converted",
      "path": "$.line_items[0].quantity",
      "content": "This item is sold by the kilogram. The requested 2.75 lb was converted to 1.25 kg."
    }
  ],
  "totals": [
    { "type": "subtotal", "amount": 1624 },
    { "type": "total", "amount": 1624 }
  ],
  "links": []
}
```

변환하지 않는 비즈니스는 대신 수정 요청을 거부한다. 라인은 변경되지 않으며
(`quantity`는 요청한 `275`를 재해석한 값이 아니라 `150`(1.50 kg)으로 유지된다),
에러는 권위 있는 기준을 명시한다:

<!-- ucp:example schema=shopping/checkout op=read -->
```json
{
  "ucp": { "version": "{{ ucp_version }}", "status": "success", "payment_handlers": {} },
  "id": "chk_fasteners_1",
  "status": "incomplete",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_fasteners",
      "item": {
        "id": "var_fasteners",
        "title": "Stainless Steel Fasteners",
        "price": 1299,
        "quantity_unit": { "unit": "KGM", "scale": 2, "display_text": "kg" }
      },
      "quantity": 150,
      "totals": [
        { "type": "subtotal", "amount": 1949 },
        { "type": "total", "amount": 1949 }
      ]
    }
  ],
  "messages": [
    {
      "type": "error",
      "code": "quantity_unit_mismatch",
      "severity": "recoverable",
      "path": "$.line_items[0].item.quantity_unit",
      "content": "This item is sold by the kilogram, not by the pound. Resubmit the quantity as a count of 0.01-kg steps."
    }
  ],
  "totals": [
    { "type": "subtotal", "amount": 1949 },
    { "type": "total", "amount": 1949 }
  ],
  "links": []
}
```

**주문 증분.** 판매 기준은
[`increment`](../../overview/index.md#ordering-increment), 즉 비즈니스가 판매하는 주문
단위를 스텝 수로 선언할 수 있다(**MAY**). 이 선언을 통해 플랫폼은 수량 스테퍼를 만들고
제출 전에 입력을 검증할 수 있다. 플랫폼이 작성하는 수량은 해당 라인의 실효 증분의
정수배여야 한다(**SHOULD**). 증분은 표현상의 제약이 아니라 판매 정책이며, 스키마
검증으로 강제되지 않는다. 증분에 맞지 않는 수량을 받으면 비즈니스는 이를 수락하거나,
라인을 증분의 배수로 수정하거나, 복구 가능한 비즈니스 결과로 거부할 수 있다(**MAY**).
수정은 반드시 설명용 `messages[]` 항목과 함께 수정된 라인 `quantity`로 반환되어야
한다(**MUST**). 비즈니스는 요청된 수량을 조용히 재해석해서는 안 된다(**MUST NOT**).
비즈니스는 자체적인 이유(예: 재고 부족)로 수량을 수정할 수도 있다(**MAY**). 이러한 수정은
증분 격자 위에 있어야 하며(**SHOULD**), 그래야 수정된 값에서 이어지는 플랫폼의 스테퍼
조작도 격자 위에 머문다.

예를 들어 바나나를 파운드 단위로, 4분의 1파운드 배수로 판매한다고 하자(`scale` `2`에서
`increment` `25`). `quantity` `137`(1.37 lb) 요청은 증분에 맞지 않는다. 이를 맞춰 주는
비즈니스는 라인을 `125`로 수정하고 경고와 함께 반환한다:

<!-- ucp:example schema=shopping/checkout target=$.messages op=read -->
```json
[
  {
    "type": "warning",
    "code": "quantity_increment_revised",
    "path": "$.line_items[0].quantity",
    "content": "This item is sold in 0.25 lb increments. Your requested 1.37 lb was adjusted to 1.25 lb."
  }
]
```

**판매 기준 반향.** 실효 판매 기준 식별자가 기본 식별자인 응답 라인에서는 비즈니스가
`quantity_unit`을 생략할 수 있다(**MAY**). 실효 식별자가 기본 식별자와 다를 때는
비즈니스가 반드시 모든 장바구니, 체크아웃, 주문 라인 응답의 상품에 `quantity_unit`을
포함해야 한다(**MUST**). 그렇지 않으면 생략으로 인해 계량 단위 라인이 낱개 상품 수로
읽히게 된다.

**라인 가격 산정.** `item.price`는 `quantity_unit.unit` 하나(예: kg당 또는 시간당)에
대한 금액이다. `quantity_unit`이 없으면 `each`당 가격이다. 판매 단위의 다른 특성이
제시된 `item.price`에 영향을 줄 수는 있지만, 판매 기준 분모를 바꾸지는 않는다.
비즈니스는 반드시 라인 합계를 `price × quantity × 10^-scale`로 계산하고 라인 단위에서
한 번 반올림해야 한다(**MUST**). 제시된 `totals[]`가 여전히 권위를 가진다
([합계](index.md#totals) 참고). 플랫폼은 소수 수량으로 라인 합계를 다시 계산해 자체
반올림 값으로 대체해서는 안 된다(**MUST NOT**).

**가격 기준.** `item.price`는 항상 판매 기준에 대해 표시되지만, 비즈니스는 다른 측정
기준으로 가격을 제시할 수 있다(MAY). 예를 들어 사과를 파운드당 가격으로 매기고
`each` 단위로 판매하는 경우다. 권위 있는 응답에서 비즈니스는 반드시 가격 기준이 판매
기준과 다른 모든 장바구니, 체크아웃, 주문 라인에 `item.unit_price`를 포함해야
한다(**MUST**). 판매 기준 자체와 마찬가지로, 청구 금액을 결정하는 단가도 라인과 함께
전달된다. 가격 기준이 판매 기준과 같으면 `unit_price`는 카탈로그 표시용 비교값으로
남으며 라인에서 생략할 수 있다(MAY). 라인에 있는지 여부가 표식이 된다. 라인 수준의
`unit_price`는 거래용이고, 카탈로그에만 있는 것은 표시용이다.

예를 들어 비즈니스가 바나나를 `quantity_unit`
`{ "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }`, `price`
`79`($0.79/lb)로 파운드 단위 판매한다고 하자. 구매자가 1.50 lb를 주문하므로 플랫폼은
`quantity` `150`, 즉 100분의 1파운드 스텝 150개를 보낸다. 라인 합계는
`79 × 150 × 10^-2 = 118.5`이며, 가격 규칙에 따라 한 번 반올림하여 `119`($1.19)가 된다:

<!-- ucp:example schema=shopping/checkout op=read -->
```json
{
  "ucp": { "version": "{{ ucp_version }}", "status": "success", "payment_handlers": {} },
  "id": "chk_bananas_1",
  "status": "incomplete",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_bananas",
      "item": {
        "id": "var_bananas",
        "title": "Bananas",
        "price": 79,
        "quantity_unit": { "unit": "LBR", "scale": 2, "display_text": "lb", "increment": 25 }
      },
      "quantity": 150,
      "totals": [
        { "type": "subtotal", "amount": 119 },
        { "type": "total", "amount": 119 }
      ]
    }
  ],
  "totals": [
    { "type": "subtotal", "amount": 119 },
    { "type": "total", "amount": 119 }
  ],
  "links": []
}
```

### 체크아웃 상태 수명 주기 { #checkout-status-lifecycle }

체크아웃의 `status` 필드는 세션의 현재 단계를 나타내며, 다음에 어떤 처리가 필요한지
결정한다. 상태는 비즈니스가 설정하고, 플랫폼은 진행에 필요한 사항을 알려 주는 메시지를
받는다.

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

### 상태 값 { #status-values }

* **`incomplete`**: 체크아웃 세션에 필수 정보가 누락되었거나 해결이 필요한 문제가
    있다. 플랫폼은 `messages` 배열에서 맥락을 확인하고 Update Checkout을 통해 해결을
    시도해야 한다. 활성 확장이 액션(Action)을 노출하는 경우, 그 인스턴스는 비즈니스가
    `ready_for_complete`를 반환하기 전에 완료되어야 하는 작업을 식별할 수 있다.

* **`requires_escalation`**: 체크아웃 세션에 API로 제공할 수 없는 정보가 필요하거나
    구매자의 입력이 필요하다. 플랫폼은 무엇이 필요한지 파악하기 위해 `messages`를
    확인해야 한다(아래 에러 처리 참고). `recoverable` 에러가 있으면 먼저 해결한다.
    그런 다음 `continue_url`을 통해 구매자에게 넘긴다.

* **`ready_for_complete`**: 체크아웃 세션에 필요한 정보가 모두 있으며, 플랫폼이
    프로그래밍 방식으로 최종 확정할 수 있다. 미처리 액션이 남아 있지 않다. 플랫폼은
    Complete Checkout을 호출할 수 있다.

* **`complete_in_progress`**: Complete Checkout 요청이 수락되었고 비즈니스가 주문을
    처리 중이다. 응답에 `order`가 포함되어서는 안 된다(**MUST NOT**). 이 상태에서는
    미처리 액션이 있을 수 있다. 플랫폼의 진행 방법은 [액션](#actions)을 참고한다.

* **`completed`**: 주문이 성공적으로 접수되었다. `order`가 존재하며 미처리 액션이
    남아 있지 않다.

* **`canceled`**: 체크아웃 세션이 유효하지 않거나 만료되었다. 필요하면 플랫폼은 새
    체크아웃 세션을 시작해야 한다. 미처리 액션이 남아 있지 않다.

### 액션 { #actions }

활성 확장에 체크아웃에 대한 미처리 작업이 있으면, 비즈니스는 해당 확장이 응답 전용
`actions` 맵에 선언한 액션 유형 아래에 인스턴스를 노출한다. 공통 규칙은
[개요 — 액션](../../overview/index.md#actions)에 정의되어 있으며, 이 절에서는 체크아웃
상태 수명 주기가 액션을 어떻게 해석하는지만 설명한다. 미처리 액션에 관한 상태 불변
조건은 [상태 값](#status-values)이 권위 있는 기준이다.

모든 액션은 해당 액션 유형에 명시된 효과를 제어한다. `incomplete` 상태에서 액션은
비즈니스가 `ready_for_complete`를 반환하기 전에 완료되어야 하는 작업을 식별할 수 있다.
액션 유형의 계약에 따라 액션을 처리한 후, 플랫폼은 최신 체크아웃을 얻기 위해
[Get Checkout](#get-checkout)을 사용하거나, `complete_in_progress`가 아닌 경우에는
후속 [Update Checkout](#update-checkout) 응답을 사용해야 한다(**SHOULD**).

액션은 체크아웃 상태를 선택하지 않으며, 메시지의 유형이나 심각도는 액션이 자신이
정의한 효과를 제어하는지 여부를 결정하지 않는다. 메시지는 특정 작업의 결과를 보고할 수
있다. 정보 또는 경고 메시지는 실패를 보고하지 않고 미처리 액션을 식별할 수 있다. 복구
가능한 에러 메시지는 액션을 식별하여 그 액션 때문에 요청한 효과가 적용되지 않았음을
보고할 수 있다. 이 경우 비즈니스는 현재 체크아웃을 반환하고,
[개요 — 액션](../../overview/index.md#actions)에 정의된 대로 메시지의 `path`를 정확한 액션
항목으로 설정한다. 메시지가 액션을 관련 없는 체크아웃 작업에 대한 잠금으로 바꾸지는
않는다.

액션 때문에 Complete Checkout을 수락할 수 없는 경우, 비즈니스는 반드시
`status: incomplete`인 현재 체크아웃과 함께, `path`로 해당 액션 항목을 정확히 가리키는
`severity: "recoverable"` 에러 메시지를 반환해야 한다(**MUST**). 액션을 처리한다고
거부된 Complete Checkout 요청이 재시도되지는 않는다. 이후의 Complete Checkout 요청은
기존 멱등성 규칙에 따른 새 작업이다.

#### 수락된 완료 { #accepted-completion }

`complete_in_progress`는 Complete Checkout이 수락되었음을 의미한다. 비즈니스가 보고하는
체크아웃 상태가 권위를 가진다. 체크아웃이 이 상태인 동안에는 다음 작업 계약이
적용된다:

| 작업 | 계약 |
| :-------- | :------- |
| Get Checkout | 플랫폼은 Get Checkout을 호출할 수 있으며(**MAY**), 비즈니스의 응답이 권위를 가진다. 플랫폼은 액션 계약이나 플랫폼 정책이 정한 제한된 백오프로 Get Checkout을 반복할 수 있으며(**MAY**), 반드시 `expires_at`에 반복 요청을 중단해야 한다(**MUST**). |
| Update Checkout | 플랫폼은 새 Update Checkout 작업을 시작해서는 안 된다(**MUST NOT**). 중복 요청은 계속 [재전송 방지](../../signatures.md#replay-protection) 규칙을 따른다. 비즈니스가 새 Update Checkout 요청을 받으면 반드시 체크아웃을 변경하지 않고 복구 가능한 에러 메시지와 함께 현재 체크아웃을 반환해야 한다(**MUST**). |
| Complete Checkout | 플랫폼은 `complete_in_progress` 동안 새 Complete Checkout 작업을 시작해서는 안 된다(**MUST NOT**). 응답 유실 시의 제한적인 복구 재시도는 [Complete Checkout](#complete-checkout)을 참고한다. |
| Cancel Checkout | 플랫폼은 Cancel Checkout을 시도하기 전에 반드시 아래의 대체 경로, 넘김, 취소 경합 규칙을 따라야 한다(**MUST**). |

`complete_in_progress` 체크아웃에 액션이 포함되어 있으면 플랫폼은 해당 액션 유형의
계약에 따라 이를 처리한다. 액션이 없으면 플랫폼은 Get Checkout을 사용해 비즈니스의
처리 상황을 관찰해야 한다(**SHOULD**).

Complete Checkout이 수락된 후에는 Update Checkout을 사용할 수 없으므로, 비즈니스는
반드시 Complete Checkout을 수락하기 전에 Update Checkout을 통한 입력이 필요한 모든
액션을 노출해야 한다(**MUST**).

액션의 확장 정의 처리가 완료된 후에도 비즈니스가 그 결과를 받아 적용하는 데 시간이
필요할 수 있다. 액션 완료 신호는 체크아웃 상태에 대해 권위를 갖지 않는다. 플랫폼은
반드시 [Get Checkout](#get-checkout)을 사용해 비즈니스가 결과를 반영했는지(예: 액션을
제거하거나 체크아웃 상태를 변경했는지) 확인해야 한다(**MUST**).

같은 액션 키와 `id`가 남아 있으면, 플랫폼은 해당 계약이 재시도를 명시적으로 허용하고
안전한 재시도 조건을 모두 충족하지 않는 한 이를 다시 처리해서는 안 된다(**MUST NOT**).
새 `id`로 대체된 것은 새 작업이다. 액션이 사라지면 그 액션은 더 이상 미처리 상태가
아니다.

플랫폼이 액션을 완료할 수 없거나, 비즈니스가 `expires_at` 이전에 액션 계약이나 플랫폼
정책이 정한 제한 시간 내에 체크아웃을 업데이트하지 않으면, 플랫폼은 적용 가능한 액션
대체 경로를 따라야 한다(**SHOULD**). 어떤 대체 경로로도 체크아웃을 계속할 수 없고
`continue_url`을 사용할 수 있으면, 플랫폼은 이를 사용해 체크아웃을 구매자에게 넘겨야
한다(**SHOULD**). 대체 경로나 넘김으로 체크아웃을 계속할 수 있는 동안에는 플랫폼이
Cancel Checkout을 시도해서는 안 된다(**MUST NOT**).

대체 경로와 넘김 모두로 계속할 수 없으면, 플랫폼은 Get Checkout으로 최신 상태를 조회하고
여전히 `complete_in_progress`로 보고되는 경우에만 Cancel Checkout을 시도해야
한다(**SHOULD**). 취소는 완료와 경합할 수 있다. 플랫폼은 비즈니스가 `status: canceled`를
보고하지 않는 한 체크아웃을 취소된 것으로 간주해서는 안 된다(**MUST NOT**).

### 에러 처리 { #error-handling }

`messages` 배열에는 체크아웃 상태에 관한 에러, 경고, 정보 메시지가 담긴다.
`ucp.status`는 형태 판별자다. `"success"`는 응답에 기대한 페이로드가 담겨 있음을,
`"error"`는 대신 에러 정보가 담겨 있음을 의미한다. 각 에러 메시지에는 `type`, `code`,
`severity`, `content`와, 메시지가 가리키는 특정 응답 구성 요소(필드, 라인 아이템, 액션
항목 등)를 식별하는 선택적 `path`가 담긴다(아래 [`path` 필드](#the-path-field) 참고).
`severity` 필드는 권장되는 플랫폼 조치를 지정한다:

| 심각도 | 의미 | 플랫폼 조치 |
| :---------------------- | :-------------------------------------------- | :---------------------------------------------------------------------------- |
| `recoverable`           | 플랫폼이 대역 내에서 상황을 해결할 수 있음 | 입력을 수정하거나 관련 액션을 처리하고, 필요하면 새 작업을 제출 |
| `requires_buyer_input`  | 비즈니스가 API로 받을 수 없는 입력을 요구함 | `continue_url`로 넘김 |
| `requires_buyer_review` | 구매자의 검토와 승인이 필요함 | `continue_url`로 넘김 |
| `unrecoverable`         | 작업 대상 리소스가 존재하지 않음 | 새 리소스나 입력으로 재시도하거나 `continue_url`로 넘김 |

`requires_*` 심각도의 에러는 `status: requires_escalation`의 원인이 된다. 둘 다 구매자
넘김으로 이어지지만, 서로 다른 체크아웃 상태를 나타낸다.

* `requires_buyer_input`은 체크아웃이 **미완료** 상태임을 의미한다. 비즈니스가 API로
프로그래밍 방식 수집을 지원하지 않는 정보를 요구하는 경우다.
* `requires_buyer_review`는 체크아웃이 **완료** 상태이지만, 정책, 규제, 권한 규칙상
주문 접수 전에 구매자의 승인이 필요함을 의미한다(예: 고액 주문 승인, 첫 구매 정책).

비즈니스가 새 리소스를 생성할 수 없거나 요청한 리소스가 더 이상 존재하지 않으면,
응답에는 실패를 설명하는 `messages`와 함께 `ucp.status: "error"`가 담기며 응답 본문에는
리소스가 포함되지 않는다. 작업 대상 리소스가 없으면 메시지는
`severity: "unrecoverable"`을 사용해야 한다(SHOULD).
예를 들어 비즈니스는 모든 상품을 구매할 수 없는 체크아웃 생성 요청을 거부할 수 있다:

<!-- ucp:example schema=common/types/error_response op=read -->
```json
{
  "ucp": { "version": "{{ ucp_version }}", "status": "error" },
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

[REST](rest.md#create-checkout) 및
[MCP](mcp.md#create_checkout) 바인딩 예시를 참고한다.

#### 에러 처리 알고리즘 { #error-processing-algorithm }

상태가 `incomplete` 또는 `requires_escalation`이면 플랫폼은 에러를 우선순위 스택으로
처리해야 한다. 아래 예시는 세 가지 에러 유형, 즉 복구 가능한 에러(잘못된 전화번호),
구매자 입력 요구(배송 일정 지정), 검토 요구(고액 주문)가 있는 체크아웃을 보여 준다.
뒤의 두 가지는 넘김이 필요하며 플랫폼에 대한 명시적 신호 역할을 한다. 비즈니스는
이러한 메시지를 가능한 한 일찍 노출해야 하며(**SHOULD**), 플랫폼은 넘김을 시작하기 전에
복구 가능한 에러를 우선 해결해야 한다(**SHOULD**). 복구 가능한 에러의 `path`가 액션
항목을 가리키면 플랫폼은 [액션](#actions)에 따라 이를 처리하고, 다른 작업을 제출하기
전에 최신 체크아웃을 다시 평가한다. 아래의 예시 알고리즘은 경로가 액션을 가리키지 않는
복구 가능한 입력 에러를 다룬다.

<!-- ucp:example schema=shopping/checkout target=$.messages op=read -->
```json
[
  {
    "type": "error",
    "code": "invalid_phone",
    "severity": "recoverable",
    "path": "$.buyer.phone_number",
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
```

에러 처리 알고리즘 예시:

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
    IF error.path is present
      IDENTIFY the field at error.path in the request payload
      ATTEMPT to fix that field (e.g., reformat phone at $.buyer.phone_number)
    ELSE
      ATTEMPT generic fix based on error.code
  CALL Update Checkout
  RETURN and re-evaluate response

IF requires_buyer_input is not empty
  handoff_context = "incomplete, additional input from buyer is required"
ELSE IF requires_buyer_review is not empty
  handoff_context = "ready for final review by the buyer"
```

#### 표준 에러 { #standard-errors }

표준 에러는 플랫폼이 일반적인 에러 처리 대신 구체적이고 적절한 UX로 처리해야 하는
표준화된 에러 코드다.

| 코드 | 설명 |
| :----------------------- | :------------------------------------------------------------------------- |
| `out_of_stock`           | 특정 상품 또는 변형을 구매할 수 없음 |
| `item_unavailable`       | 상품을 구매할 수 없음(예: 판매 목록에서 제외됨) |
| `address_undeliverable`  | 제공된 주소로 배송할 수 없음 |
| `payment_failed`         | 결제 처리에 실패함 |
| `eligibility_invalid`    | 완료 시점에 자격 주장을 검증할 수 없음 |

비즈니스는 표준 에러에 `severity: recoverable`을 표시하여, 플랫폼이 일반적인 에러
메시지를 보여 주거나 체크아웃 완료 시점까지 미루지 말고 적절한 UX(품절 안내, 주소
확인 프롬프트, 결제 수단 변경)를 제공해야 함을 알려야 한다(**SHOULD**).

예: `out_of_stock`은 사전에 구체적인 UX가 필요한 반면, `payment_required`는 제출
시점에 일반적인 방식으로 처리할 수 있다.

#### `path` 필드 { #the-path-field }

메시지의 선택적 `path` 필드는 메시지를 응답 페이로드의 특정 구성 요소에 연결한다.
플랫폼은 이를 사용해 메시지를 입력 필드, 라인 아이템, 액션 항목과 연관 짓는다. 예를
들어 특정 구매자 필드를 강조하거나, 장바구니 라인에 표시를 하거나, 작업 결과와 관련된
액션을 식별할 수 있다.

`path`는 반드시 UCP 응답 객체의 루트를 기준으로 하는
[RFC 9535](https://www.rfc-editor.org/rfc/rfc9535) JSONPath 표현식이어야 한다(**MUST**).
`path`를 생략하면 메시지는 응답 전체에 적용된다.

**단순 필드 참조:**

<!-- ucp:example skip reason="path schema example" -->
```json
{ "path": "$.buyer.email" }
```

**인덱스로 지정한 배열 요소:**

<!-- ucp:example skip reason="path schema example" -->
```json
{ "path": "$.line_items[0].quantity" }
```

**액션 항목:**

<!-- ucp:example skip reason="path schema example" -->
```json
{
  "path": "$.actions['com.example.identity.student_verification'][0]"
}
```

**구체성 규칙:** 특정 필드에 대한 경로(예: `$.line_items[0].quantity`)가 그 상위
요소에 대한 경로(예: `$.line_items[0]`)보다 우선한다. 같은 필드에 여러 에러가 적용되는
경우, 각 메시지는 적용 가능한 가장 구체적인 경로를 담아야 한다(**SHOULD**).

#### 완료 시 자격 검증 { #eligibility-verification-at-completion }

플랫폼은 `context.eligibility`, 즉 로열티 멤버십, 결제 수단 혜택 등 구매자가 받을 수
있는 혜택에 대한 구매자의 주장을 제공한다. 이는 검증된 사실이 아니라 주장이다.
비즈니스는 세션 중에 인식한 주장에 따라 조치할 수 있지만(**MAY**)(가격 조정, 상품 접근
허용, 잠정 할인 적용), 수락한 모든 주장은 반드시 거래가 완료되기 전에 해소되어야
한다(**MUST**).

인식되지 않았거나 적용할 수 없는 주장이 체크아웃을 막아서는 안 된다(**MUST NOT**).
비즈니스는 주장이 수락되지 않으면 `type: "warning"` `messages`로 구매자에게 알려야
하며(**SHOULD**), 수락된 주장의 효과를 설명하기 위해 `type: "info"`를 사용할 수
있다(**MAY**). 완료 시점에 여전히 검증되지 않은 수락된 주장은 반드시
`code: "eligibility_invalid"`인 `type: "error"`로 이어져야 한다(**MUST**)(아래 참고).

**자격 메시지 코드:**

| 유형 | 코드 | 발생 시점 |
| --------- | -------------------------- | -------------------------------------------------- |
| `warning` | `eligibility_not_accepted` | 주장을 인식하지 못했거나 적용할 수 없음 |
| `info`    | `eligibility_accepted`     | 수락된 주장의 효과 |
| `error`   | `eligibility_invalid`      | 완료 시점에 수락된 주장을 검증할 수 없음 |

주장은 **검증**되거나 **철회**되면 해소된다:

* **검증됨**: 비즈니스가 증명을 바탕으로 주장을 확인한다. UCP는 검증 방식을 규정하지
  않는다. 증명은 결제 자격 증명, 협상된 확장이 노출한 액션, 또는 플랫폼과 비즈니스
  간에 협상된 다른 메커니즘에서 올 수 있다.
* **철회됨**: 플랫폼이 완료 전에 `context.eligibility`에서 주장을 제거한다(예: 구매자가
  결제 수단을 변경하거나 멤버십 주장을 철회함). 제거되면 비즈니스는 해당 주장 없이 다시
  계산한다.

비즈니스는 해소되지 않은 자격 주장이 있는 상태로 거래를 완료해서는 안
된다(**MUST NOT**). 검증되지 않은 주장은 잘못된 가격 책정이나 제한된 상품에 대한 무단
접근으로 이어질 수 있다.

**검증에 실패한 경우:**

비즈니스는 반드시 `code: "eligibility_invalid"`와 `severity: "recoverable"`을 가진
에러를 `messages`에 반환해야 한다(**MUST**). 액션이 요청한 효과를 막는 경우, 비즈니스는
반드시 [액션](#actions)에 명시된 대로 해당 에러 메시지의 `path`가 정확한 액션 항목을
가리키도록 설정해야 한다(**MUST**). 그렇지 않은 경우 비즈니스는 `path`를 사용해 어떤
주장을 검증할 수 없었는지 식별해야 한다(**SHOULD**). 협상된 확장이 기여한 액션이나 할인
상태는 해당 확장의 규칙에 따라 현재 상태로 유지될 수 있다(**MAY**). 그러면 플랫폼은
해당 확장의 검증 흐름을 재시도하거나, 일반적인 체크아웃 변경(예: 자격이 없는 상품 제거)
또는 주장 철회를 위해 [Update Checkout](#update-checkout)을 사용하거나, 시도를 포기할 수
있다(**MAY**).

##### 예시: 검증 액션으로 주장 해소하기 { #example-resolving-a-claim-with-a-verification-action }

이 예시는 설명을 위한 것이다. 여기서는 `actions` 안에 같은 이름의 키로 단일 액션 유형을
선언하고 인스턴스 `config`와 검증 전송 방식을 정의하는 협상된 벤더 확장
`com.example.identity.student_verification`을 사용한다. 이 확장을 `context.eligibility`,
잠정 [할인](../extensions/discount.md), [액션](#actions), `messages`와 조합한다.

잠정 할인 필드(`provisional`, `eligibility`)는
[할인 확장](../extensions/discount.md#eligibility-claims)에 속하며, 해당 확장이 체크아웃에
활성화된 경우에만 사용할 수 있다.

**1. 주장 수락, 잠정 할인, 검증 액션 미처리.**
플랫폼은 Create Checkout에서 `context.eligibility`에 `org.example.student`를 제출한다.
비즈니스는 주장을 수락하고, 할인 확장이 활성화되어 있으므로 잠정 할인, 설명용 `info`
메시지, 검증 액션을 반환한다. 주장이 해소되지 않았으므로 체크아웃은 `incomplete`로
유지된다:

<!-- ucp:example schema=shopping/checkout op=read -->
```json
{
  "ucp": { ... },
  "id": "chk_student_1",
  "status": "incomplete",
  "currency": "USD",
  "line_items": [ ... ],
  "discounts": {
    "applied": [
      {
        "title": "Student 10% Off",
        "amount": 500,
        "automatic": true,
        "provisional": true,
        "eligibility": "org.example.student"
      }
    ]
  },
  "totals": [ ... ],
  "links": [ ... ],
  "actions": {
    "com.example.identity.student_verification": [
      {
        "id": "verify-student-1",
        "config": {
          "verification_url": "https://business.example.com/verify/abc"
        }
      }
    ]
  },
  "messages": [
    {
      "type": "info",
      "code": "eligibility_accepted",
      "content": "Student discount applied provisionally. Verify your student status to keep it.",
      "path": "$.actions['com.example.identity.student_verification'][0]"
    }
  ]
}
```

플랫폼은 확장이 정의한 검증 흐름을 실행한다. 비즈니스는 대역 외(out of band)로 결과를
받는다. 그 흐름이 완료되면 플랫폼은 [Get Checkout](#get-checkout)으로 업데이트된 상태를
조회한다.

**2. 검증됨.** 비즈니스가 주장을 해소한다. 검증 액션이 `actions`에서 제거되고, 할인이
확정되며(더 이상 `provisional`이 아님), 설명용 `info` 메시지가 갱신된다. 주장이
해소되었으므로 체크아웃은 `ready_for_complete`로 진행될 수 있다(**MAY**).

**3. 유효하지 않은 증명.** 제출된 증명이 검증되지 않으면 주장은 해소되지 않은 상태로
남는다. 비즈니스는 `severity: recoverable`인 `eligibility_invalid` 에러를 반환하며
([검증에 실패한 경우](#eligibility-verification-at-completion) 참고), 체크아웃은
`ready_for_complete`에 도달하지 않는다. 플랫폼은 유효한 증명을 제공해 검증 흐름을
재시도하거나 주장을 철회할 수 있다(**MAY**).

**4. 철회.** 플랫폼은 `context.eligibility`에서 주장을 제거하는
[Update Checkout](#update-checkout)으로 주장을 철회한다. 그러면 비즈니스는 잠정 할인
없이 다시 계산한다.

### 경고 표시 방식 { #warning-presentation }

경고 메시지의 `presentation` 필드는 플랫폼이 반드시 따라야 하는(**MUST**) 렌더링 계약을
제어한다. 생략하면 기본값은 `"notice"`다.

| | `notice` (기본값) | `disclosure` |
| :--- | :--- | :--- |
| 내용 표시 | **MUST** | **MUST** |
| `path`와의 근접 배치 | **MAY** | **MUST** |
| 닫기 허용 | **MAY** | **MUST NOT** |
| `image_url` 렌더링 | **MAY** | **MUST** |
| `url` 렌더링 | **MAY** | **SHOULD** |
| 준수할 수 없을 때 에스컬레이션 | — | `continue_url`로 **MUST** |

#### `notice` (기본값) { #notice-default }

경고의 기본 렌더링 계약이다. 플랫폼은 반드시 경고 내용을 구매자에게 표시해야
한다(**MUST**). 플랫폼은 알림을 배너, 트레이, 토스트 형태로 렌더링할 수 있으며(**MAY**),
구매자가 이를 닫도록 허용할 수 있다(**MAY**).

#### `disclosure` { #disclosure }

`presentation: "disclosure"`인 경고는 안전 경고, 알레르기 유발 물질 표시, 규정 준수
내용 등 반드시 아래에 규정된 렌더링 계약을 따라야 하는(**MUST**) 고지를 담는다.

**플랫폼 요구 사항:**

* 반드시 경고 `content`를 구매자에게 표시해야 한다(**MUST**).
* 반드시 고지와 그 대상 간의 연관성이 유지되도록 `path`가 참조하는 구성 요소 가까이에
  경고를 표시해야 한다(**MUST**). `path`를 생략하면 고지는 응답 전체에 적용된다.
* 경고를 숨기거나, 접거나, 자동으로 닫아서는 안 된다(**MUST NOT**).
* `image_url`이 있으면 반드시 렌더링해야 한다(**MUST**)(예: 경고 기호, 에너지 등급
  라벨).
* `url`이 있으면 이동 가능한 참조 링크로 렌더링해야 한다(**SHOULD**).

`presentation: "disclosure"`인 경고는 일반 알림보다 렌더링 우선순위가 높아야
한다(**SHOULD**).

고지 렌더링 계약을 준수할 수 없는 플랫폼은 조용히 일반 알림으로 격하하지 말고, 반드시
`continue_url`을 통해 판매자 UI로 에스컬레이션해야 한다(**MUST**).

**비즈니스 요구 사항:**

* 경고 내용이 특정 구성 요소와 함께 표시되어야 하고 숨기거나 자동으로 닫아서는 안 되는
  경우, 반드시 `presentation: "disclosure"`를 설정해야 한다(**MUST**).
* `path` 필드를 사용해 고지를 응답의 관련 구성 요소와 연관 지어야 한다(**SHOULD**).
* 고지 범주를 식별하는 `code`(예: `prop65`, `allergens`, `energy_label`)를 제공해야
  한다(**SHOULD**). 고지가 구조화된 `policies[]` 항목을 렌더링하는 경우, 둘을 연결하기
  위해 `code`는 반드시 해당 정책의 `type`과 같아야 한다(**MUST**)
  ([정책](../../overview/index.md#policies) 참고).
* 고지에 관련 시각 요소(예: 경고 기호, 에너지 등급 라벨)가 있으면 `image_url`을
  제공해야 한다(**SHOULD**).
* 구매자가 더 자세히 알아볼 수 있는 참조 링크가 있으면 `url`을 제공해야
  한다(**SHOULD**).

#### 고지와 확인 { #disclosure-and-acknowledgment }

`presentation` 필드는 체크아웃 진행 가능 여부가 아니라 경고의 렌더링 방식을 제어한다.
구매자의 명시적인 확인이나 승인도 필요한 경우, 비즈니스는 적절한 구매자 입력을 확실히
받기 위해 고지를
[체크아웃 상태 수명 주기](#checkout-status-lifecycle)에 설명된 에스컬레이션 메커니즘과
결합할 수 있다(**MAY**).

#### 관할권과 적용 범위 { #jurisdiction-and-applicability }

특정 세션에 어떤 고지가 적용되는지 판단하고 관련 있는 고지만 반환하는 것은 비즈니스의
책임이다. 비즈니스는 관할권별 요구 사항을 결정하기 위해 구매자가 제공한 데이터
(`context` 및 기타 입력)와 상품 속성을 사용해야 한다(**SHOULD**). 플랫폼은 고지 적용
범위에 영향을 주거나 이를 결정하지 않으며, 비즈니스로부터 받은 내용을 렌더링한다.

#### 예시 { #example }

라인 아이템에 복구 가능한 에러와 고지 경고가 함께 포함된 체크아웃 응답이다:

<!-- ucp:example schema=shopping/checkout op=read -->
```json
{
  "ucp": { "version": "{{ ucp_version }}", "status": "success", "payment_handlers": { ... } },
  "id": "chk_abc123",
  "status": "incomplete",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_1",
      "item": { "id": "item_456", "title": "Artisan Nut Butter Collection", "price": 1299, "image_url": "https://merchant.com/nut-butter.jpg" },
      "quantity": 1,
      "totals": [
        { "type": "subtotal", "amount": 1299 },
        { "type": "total", "amount": 1299 }
      ]
    }
  ],
  "totals": [
    { "type": "subtotal", "amount": 1299 },
    { "type": "total", "amount": 1299 }
  ],
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

플랫폼은 복구 가능한 에러를 프로그래밍 방식으로 해결하면서, 알레르기 유발 물질 고지를
참조된 라인 아이템 가까이에 렌더링한다.

## Continue URL { #continue-url }

`continue_url` 필드는 플랫폼에서 비즈니스 UI로 체크아웃을 넘길 수 있게 하여, 구매자가
체크아웃 세션을 이어서 진행하고 최종 확정할 수 있도록 한다.

### 제공 조건 { #availability }

비즈니스는 `status` = `requires_escalation`을 반환할 때 반드시 `continue_url`을 제공해야
한다(**MUST**). 그 밖의 모든 비종료 상태(`incomplete`, `ready_for_complete`,
`complete_in_progress`)에서는 비즈니스가 `continue_url`을 제공해야 한다(**SHOULD**).
종료 상태(`completed`, `canceled`)에서는 `continue_url`을 생략해야 한다(**SHOULD**).

### 형식 { #format }

`continue_url`은 반드시 절대 HTTPS URL이어야 하며(**MUST**), 매끄러운 넘김을 위해
체크아웃 상태를 보존해야 한다(**SHOULD**). 비즈니스는 다음 중 한 가지 방법으로 상태 보존을
구현할 수 있다(**MAY**):

#### 서버 측 상태(권장) { #server-side-state-recommended }

서버 측 체크아웃 상태를 기반으로 하는 불투명한 URL:

```text
https://business.example.com/checkout-sessions/{checkout_id}
```

* 서버가 `checkout_id`에 연결된 체크아웃 상태를 유지한다
* 단순하고 안전하며, 대부분의 구현에 권장된다
* URL 수명은 일반적으로 `expires_at`에 연동된다

#### 체크아웃 퍼머링크 { #checkout-permalink }

체크아웃 상태를 직접 인코딩하여 서버 측 저장 없이도 재구성할 수 있게 하는 상태 비저장
URL이다. 비즈니스는 체크아웃 넘김과 빠른 진입을 돕기 위해 이 형식을 지원하도록 구현해야
한다(**SHOULD**). 예를 들어 플랫폼은 바로 구매 흐름을 시작할 때 체크아웃 상태를 미리
채울 수 있다.

> **참고:** 체크아웃 퍼머링크는 [REST 전송 방식 바인딩](rest.md)을 확장하는 REST 전용
> 구성 요소다. 퍼머링크에 접근하면 체크아웃 UI로 리디렉션되거나 체크아웃 페이지가 바로
> 렌더링된다.

## 스코프 { #scopes }

체크아웃 기능은 사용자 인증 접근을 위해 다음과 같은 well-known 스코프를 정의한다:

| 스코프 | 설명 |
| :--- | :--- |
| `dev.ucp.shopping.checkout:manage` | 인증된 사용자를 대신하는 모든 체크아웃 작업. 체크아웃 세션의 생성, 수정, 완료, 취소를 포함한다. |

스코프 선언, 도출, 그리고 사용자 정의 스코프로 이 집합을 확장하는 규칙은
[신원 연결 — 스코프](../../common/identity-linking/index.md#scopes)에 정의되어 있다.

## 가이드라인 { #guidelines }

(전체 가이드라인에 더하여)

### 플랫폼 { #platform }

* 체크아웃 세션 진행을 돕기 위해 에이전트를 활용할 수 있다(**MAY**)(예: 체크아웃
    세션에 상품 추가, 풀필먼트 주소 선택). 단, 에이전트는 사용자가 체크아웃 세부 정보를
    검토하고 주문할 수 있도록 체크아웃 세션을 신뢰할 수 있고 결정론적인 UI로 넘겨야 한다.
* 언제든지 신뢰할 수 있는 결정론적 UI에서 사용자를 다시 에이전트로 보낼 수
    있다(**MAY**). 예를 들어 사용자가 장바구니에 상품을 계속 추가하기 위해 체크아웃
    화면을 나가기로 한 경우다.
* 요청이 에이전트에 의해 수행되었음을 플랫폼이 표시하는 경우 에이전트 컨텍스트를
    제공할 수 있다(**MAY**).
* 체크아웃 상태가 `requires_escalation`이면 반드시 `continue_url`을 사용해야
  한다(**MUST**).
* 다른 상황에서도 `continue_url`을 사용해 비즈니스 UI로 넘길 수 있다(**MAY**).
* 넘김을 수행할 때는 플랫폼이 구성한 체크아웃 퍼머링크보다 비즈니스가 제공한
    `continue_url`을 우선해야 한다(**SHOULD**).

### 비즈니스 { #business }

* 체크아웃이 완료된 후 반드시 확인 이메일을 보내야 한다(**MUST**).
* 정확한 에러 메시지를 제공해야 한다(**SHOULD**).
* 체크아웃 세션을 처리하는 로직은 반드시 결정론적이어야 한다(**MUST**).
* `status` = `requires_escalation`을 반환할 때 반드시 `continue_url`을 제공해야
    한다(**MUST**).
* `status` = `requires_escalation`을 반환할 때 반드시 `severity`가
    `requires_buyer_input` 또는 `requires_buyer_review`인 메시지를 하나 이상
    포함해야 한다(**MUST**).
* 종료 상태가 아닌 모든 체크아웃 응답에 `continue_url`을 제공해야 한다(**SHOULD**).
* 체크아웃 세션이 "completed" 상태에 도달하면 변경할 수 없는 것으로 간주한다.

## 기능 스키마 정의 <span id="checkout"></span> { #capability-schema-definition }

{{ schema_fields('checkout_resp', 'shopping/checkout') }}

## 작업 { #operations }

체크아웃 기능은 다음과 같은 논리적 작업을 정의한다.

| 작업 | 설명 |
| :-------------------- | :--------------------------------------------------------------------------------- |
| **Create Checkout**   | 새 체크아웃 세션을 시작한다. 사용자가 장바구니에 상품을 추가하는 즉시 호출된다. |
| **Get Checkout**      | 체크아웃 세션의 현재 상태를 조회한다. |
| **Update Checkout**   | 체크아웃 세션을 수정한다. |
| **Complete Checkout** | 체크아웃을 최종 확정하고 주문을 접수한다. |
| **Cancel Checkout**   | 체크아웃 세션을 취소한다. |

### 체크아웃 생성(Create Checkout) { #create-checkout }

사용자가 구매 의사를 표시했을 때(예: 구매 버튼 클릭) 플랫폼이 호출하여 상품 세부
정보로 체크아웃 세션을 시작한다.

**권장 사항**: 불일치를 최소화하고 매끄러운 사용자 경험을 제공하기 위해, 비즈니스가
피드로 제공하는 상품 데이터(가격/제목 등)는 응답에서 반환되는 실제 속성과 일치해야
한다(**SHOULD**).

[장바구니](../cart/index.md) 기능이 협상된 경우, 요청 페이로드는 장바구니에서
체크아웃으로의 전환을 위해 추가 `cart_id` 필드를 받아야 한다. 필드 계약은
[장바구니 → 장바구니에서 체크아웃으로 전환](../cart/index.md#cart-to-checkout-conversion)을
참고한다.

{{ method_fields('create_checkout', 'shopping/rest.openapi.json', 'shopping/checkout') }}

### 체크아웃 조회(Get Checkout) { #get-checkout }

체크아웃 리소스의 최신 상태를 제공한다. 취소 또는 완료 후에 무엇을 반환할지는
비즈니스가 결정한다(즉, 오래 유지되는 상태일 수도 있고, 특정 TTL 이후 만료되어 'not
found' 에러가 발생할 수도 있다). 플랫폼 측에서는 체크아웃 TTL을 따로 강제하지 않는다.

플랫폼은 체크아웃 세션 생성 시점에 비즈니스가 `expires_at`으로 제공한 TTL을 준수한다.

{{ method_fields('get_checkout', 'shopping/rest.openapi.json', 'shopping/checkout') }}

### 체크아웃 수정(Update Checkout) { #update-checkout }

체크아웃 리소스를 전체 교체한다. 플랫폼은 반드시 쓰기 전용 데이터 필드에 대한 데이터
갱신을 포함한 체크아웃 리소스 전체를 보내야 한다(**REQUIRED**). 요청에 담긴 리소스가
비즈니스 측의 기존 체크아웃 세션 상태를 대체한다. 이 일반 교체 규칙은
`complete_in_progress` 동안에는 Update Checkout이 허용되지 않으므로 적용되지 않는다.
고정된 작업 계약은 [수락된 완료](#accepted-completion)를 참고한다.

{{ method_fields('update_checkout', 'shopping/rest.openapi.json', 'shopping/checkout') }}

### 체크아웃 완료(Complete Checkout) { #complete-checkout }

체크아웃의 최종 주문 접수 호출이다. 사용자가 선택한 상품에 대해 결제하고 주문하기로
확정했을 때 호출한다. Complete Checkout은 `ready_for_complete` 상태에서 체크아웃을
완료하고 주문을 접수한다.

응답은 체크아웃 객체다:

* 완료가 동기적으로 끝나면 `status`는 `completed`이고 `order` 필드가 존재하며, 접수된
    주문의 전체 상태를 참조하는 데 사용할 수 있는 `id`, `permalink_url` 같은 필수
    식별자를 제공한다.
* 완료가 비동기 처리로 수락되면 `status`는 `complete_in_progress`이고 `order`는 없다.
    플랫폼은 비즈니스 처리와 미처리 액션에 대해 [수락된 완료](#accepted-completion)를
    따른다.
* 그 밖의 상태는 핵심 [체크아웃 상태 수명 주기](#checkout-status-lifecycle)의 의미를
    그대로 유지한다. 예를 들어 복구 가능한 변경에는 `incomplete`, 구매자 넘김에는
    `requires_escalation`이다.

플랫폼은 수락된 처리를 폴링하거나 재개하기 위해 Complete Checkout을 사용해서는 안
된다(**MUST NOT**). Complete Checkout에 대한 응답을 받지 못하면, 플랫폼은 제한된
백오프로 Get Checkout을 사용해 현재 상태를 확인해야 하며(**SHOULD**), 반드시
`expires_at`에 중단해야 한다(**MUST**). Get Checkout으로도 비즈니스가 원래 요청을
처리했는지 확인할 수 없는 경우에만, 플랫폼은 같은 멱등성 키로 원래와 동일한 Complete
Checkout 요청을 다시 제출할 수 있다(**MAY**). 플랫폼은 이 재시도에 새 멱등성 키를
사용해서는 안 된다(**MUST NOT**). 체크아웃이 `ready_for_complete`로 돌아간 뒤의 실제로
새로운 Complete Checkout 작업은 페이로드가 바뀌지 않았더라도 반드시 새 멱등성 키를
사용해야 한다(**MUST**).

주문을 저장하는 시점에 `Checkout`의 필드를 사용해 주문 표현을 구성할 수 있다(**MAY**)
(즉, `line_items`, `fulfillment` 같은 정보가 초기 주문 표현을 만드는 데 사용된다).

주문이 접수된 후에는 주문과 관련 상품이 공급망을 거쳐 이동함에 따라 다른 세부 정보가
후속 이벤트를 통해 갱신된다.

{{ method_fields('complete_checkout', 'shopping/rest.openapi.json', 'shopping/checkout') }}

### 체크아웃 취소(Cancel Checkout) { #cancel-checkout }

이 작업은 취소 가능한 체크아웃 세션을 취소하는 데 사용된다. 체크아웃 세션을 취소할 수
없는 경우(예: 체크아웃 세션이 이미 취소되었거나 완료됨), 비즈니스는 작업이 허용되지
않음을 나타내는 에러를 돌려보내야 한다(**SHOULD**). 상태가 `completed` 또는 `canceled`가
아닌 모든 체크아웃 세션은 취소할 수 있어야 한다(**SHOULD**).

{{ method_fields('cancel_checkout', 'shopping/rest.openapi.json', 'shopping/checkout') }}

## 전송 방식 바인딩 { #transport-bindings }

위의 추상 작업은 아래에 정의된 대로 특정 전송 프로토콜에 바인딩된다:

* [REST 바인딩](rest.md): 표준 HTTP 메서드와 JSON 페이로드를 사용하는 RESTful API 매핑.
* [MCP 바인딩](mcp.md): 에이전트 상호작용을 위한 Model Context Protocol 매핑.
* [A2A 바인딩](a2a.md): 에이전트 상호작용을 위한 Agent-to-Agent Protocol 매핑.
* [임베디드 체크아웃 바인딩](embedded.md): 임베디드 체크아웃을 구동하기 위한 JSON-RPC.

## 엔티티 { #entities }

### 구매자 { #buyer }

{{ schema_fields('buyer', 'shopping/checkout') }}

### 컨텍스트 { #context }

컨텍스트 신호는 잠정적인 것이며 권위 있는 데이터가 아니다. 비즈니스는 검증된 입력(예:
배송 주소)이 없을 때 이 값을 사용해야 하며(SHOULD), 더 신뢰도 높은 신호(인증된 계정,
위험 탐지)나 규제 제약(수출 통제)과 일치하지 않으면 무시하거나 우선순위를 낮출 수
있다(MAY). 자격 및 정책 적용은 반드시 체크아웃 시점에 구속력 있는 거래 데이터를 사용해
이루어져야 한다(MUST).

{{ schema_fields('context', 'shopping/checkout') }}

### 신호 { #signals }

승인과 악용 방지를 지원하기 위해 플랫폼이 제공하는 환경 데이터다. `context`(구매자가
주장한 선호)나 `buyer`(자기 신고 신원)와 달리, 신호 값은 구매자가 주장한 내용이어서는
안 된다(MUST NOT). 플랫폼은 직접 관찰한 내용을 바탕으로 하거나, 독립적으로 검증 가능한
제3자 증명을 전달하는 방식으로 신호를 제공한다. 자세한 내용과 프라이버시 요구 사항은
[신호](../../overview/index.md#signals)를 참고한다.

{{ schema_fields('types/signals', 'shopping/checkout') }}

### 어트리뷰션 { #attribution }

플랫폼이 제공하는 유입 경로 및 전환 이벤트 컨텍스트로, 플랫폼이 전달하는 캠페인 ID,
클릭 식별자, 소스/매체 표식을 말한다. 자세한 내용과 동의 요구 사항은
[어트리뷰션](../../overview/index.md#attribution)을 참고한다.

{{ schema_fields('types/attribution', 'shopping/checkout') }}

### 상품 { #item }

#### 상품 생성 요청 { #item-create-request }

{{ schema_fields('types/item_create_req', 'shopping/checkout') }}

#### 상품 수정 요청 { #item-update-request }

{{ schema_fields('types/item_update_req', 'shopping/checkout') }}

#### 상품 { #item_1 }

{{ schema_fields('types/item_resp', 'shopping/checkout') }}

### 라인 아이템 { #line-item }

#### 라인 아이템 생성 요청 { #line-item-create-request }

{{ schema_fields('types/line_item_create_req', 'shopping/checkout') }}

#### 라인 아이템 수정 요청 { #line-item-update-request }

{{ schema_fields('types/line_item_update_req', 'shopping/checkout') }}

#### 라인 아이템 { #line-item_1 }

{{ schema_fields('types/line_item_resp', 'shopping/checkout') }}

### 링크 { #link }

{{ schema_fields('types/link', 'shopping/checkout') }}

#### Well-Known 링크 유형 { #well-known-link-types }

비즈니스는 거래와 관련된 모든 링크를 제공해야 한다(**SHOULD**). 권장되는 well-known
유형은 다음과 같다:

| 유형 | 설명 |
| :----------------- | :------------------------------------------------ |
| `privacy_policy`   | 비즈니스의 개인정보 처리방침 링크 |
| `terms_of_service` | 비즈니스의 서비스 약관 링크 |
| `refund_policy`    | 비즈니스의 환불 정책 링크 |
| `shipping_policy`  | 비즈니스의 배송 정책 링크 |
| `faq`              | 비즈니스의 자주 묻는 질문 링크 |

비즈니스는 도메인별 요구에 맞게 사용자 정의 유형을 정의할 수 있다(**MAY**). 플랫폼은
알 수 없는 유형을 `title` 필드로 표시하거나 생략하는 방식으로 적절히 처리해야
한다(**SHOULD**).

### 정책 { #policy }

이 체크아웃의 상품에 적용되는 정책(반품/환불 조건, 보증 등)이다. `applies_to`의
JSONPath 대상은 이 응답의 루트를 기준으로 한다(예: `$.line_items[0]`). 전체 모델은
[정책](../../overview/index.md#policies)을 참고한다.

{{ schema_fields('types/policy', 'shopping/checkout') }}

### 메시지 { #message }

{{ schema_fields('message', 'shopping/checkout') }}

### 에러 메시지 { #message-error }

{{ schema_fields('types/message_error', 'shopping/checkout') }}

#### 에러 코드 { #error-code }

{{ schema_fields('types/error_code', 'shopping/checkout') }}

### 정보 메시지 { #message-info }

{{ schema_fields('types/message_info', 'shopping/checkout') }}

### 경고 메시지 { #message-warning }

{{ schema_fields('types/message_warning', 'shopping/checkout') }}

### 결제 { #payment }

{{ schema_fields('payment', 'shopping/checkout') }}

#### 선택된 결제 수단 { #selected-payment-instrument }

{{ extension_schema_fields('../common/types/payment_instrument.json#/$defs/selected_payment_instrument', 'shopping/checkout') }}

### 결제 자격 증명 { #payment-credential }

{{ schema_fields('payment_credential', 'shopping/checkout') }}

### 우편 주소 { #postal-address }

{{ schema_fields('postal_address', 'shopping/checkout') }}

### 응답 { #response }

{{ extension_schema_fields('capability.json#/$defs/response_schema', 'shopping/checkout') }}

### 합계 {: #totals }

{{ schema_fields('types/total_resp', 'shopping/checkout') }}

#### 렌더링 계약 { #rendering-contract }

제시되는 합계의 내용과 순서는 비즈니스가 권위 있는 출처다. 올바른 표시 방식은
비즈니스가 충족해야 하는 지역, 상품, 규제 요구 사항(예: 다중 관할권 세금 항목화, 필수
수수료 고지)의 적용을 받기 때문이다.

플랫폼은 반드시 모든 최상위 항목을 제공된 순서대로 렌더링해야 한다(MUST):

```python
for entry in totals:
    render_line(entry.display_text, entry.amount)
```

플랫폼은 하위 라인을 보조 세부 정보로 렌더링할 수 있다(MAY):

```python
for entry in totals:
    render_line(entry.display_text, entry.amount)
    if entry.lines:
        for sub in entry.lines:
            render_detail_line(sub.display_text, sub.amount)
```

플랫폼은 항목을 해석, 필터링, 재정렬, 집계하거나 자체 표시 로직을 적용해서는 안
된다(MUST NOT).

`totals[]`의 불변 조건:

* 모든 항목에는 `type`과 `amount`가 있다. 플랫폼은 `display_text`가 제공되면 이를
  사용해야 한다(SHOULD). well-known 유형에는 대체용 기본 표시 라벨이 있다(아래 표
  참고). 알 수 없는 유형은 반드시 `display_text`를 포함해야 한다(MUST).
* 금액은 부호 있는 정수다. 음수 값은 차감(예: 할인), 양수 값은 가산을 의미한다. 부호가
  곧 방향이다.
* `type: "subtotal"`은 반드시 정확히 하나 있어야 한다(MUST).
* `type: "total"`은 반드시 정확히 하나 있어야 한다(MUST).

#### 검증 { #verification }

플랫폼은 비즈니스의 값을 자체 계산한 합계로 대체해서는 안 된다(MUST NOT). 플랫폼은
제공된 합계를 검증할 수 있다(MAY):

```python
assert sum(e.amount for e in totals if e.type != "total") == total_entry.amount
```

계산한 합이 `type: "total"` 항목과 일치하지 않더라도 플랫폼은 렌더링 결과를 변경해서는
안 된다(MUST NOT). 표시에 대해서는 비즈니스가 제시한 합계가 권위를 가진다. 그러나
플랫폼은 합계가 일치하지 않는 체크아웃을 자율적으로 완료해서는 안 된다(MUST NOT).
플랫폼은 체크아웃을 거부하거나, 에스컬레이션하여 `continue_url`을 통해 구매자의 검토를
요청해야 한다(SHOULD).

#### Well-Known 유형 { #well-known-types }

| 유형 | 부호 | 기본 라벨 | 의미 |
| ----------------- | ---- | ---------------- | ----------------------------------------- |
| `subtotal`        | +    | Subtotal         | 라인 아이템 가격의 합 |
| `discount`        | −    | Discount         | 주문 또는 라인 아이템 수준 할인 |
| `items_discount`  | −    | Item Discounts   | 라인 아이템 할인의 합산 |
| `fulfillment`     | +    | Shipping         | 배송, 배달, 픽업 요금 |
| `tax`             | +    | Tax              | 세금 |
| `fee`             | +    | Fee              | 수수료 및 추가 요금 |
| `total`           | =    | Total            | 권위 있는 최종 합계(정확히 하나) |

`display_text`가 제공되면 플랫폼은 반드시 이를 사용해야 한다(MUST). well-known 유형에서
생략된 경우 플랫폼은 위의 기본 라벨을 사용해야 한다(SHOULD). well-known 유형의 부호
규칙은 스키마로 강제된다. 차감 유형(discount, items_discount)은 반드시 음수 금액이어야
하고(MUST), 가산 유형(subtotal, fulfillment, tax, fee)은 반드시 음수가 아닌 금액이어야
한다(MUST).

`type` 필드는 개방형 문자열이므로 비즈니스는 well-known 집합 외의 값도 사용할 수
있다(MAY). 알 수 없는 유형은 반드시 `display_text`를 포함해야 하며(MUST)(스키마로 강제됨),
금액의 부호가 스스로 의미를 나타낸다.

#### 반복 유형 { #repeating-types }

`subtotal`과 `total`을 제외한 모든 유형은 여러 번 나타날 수 있다(MAY). 예를 들어 다중
관할권 세금 라인이나 항목별 수수료가 이에 해당한다.

#### 하위 라인(`lines`) { #sub-lines-lines }

각 최상위 항목은 `lines` 배열을 포함할 수 있다(MAY). 하위 라인은 최상위 항목과 같은
기본 형태(`display_text`와 `amount`)를 가지며, 상위 항목 아래에 항목별 내역을 제공한다.

**불변 조건:** `sum(lines[].amount)`는 반드시 상위 항목의 `amount`와 같아야 한다(MUST).

반드시 렌더링해야 하는 것(MUST, 최상위 항목)과 선택적으로 노출할 수 있는 것(MAY, 하위
라인)은 비즈니스가 결정한다. 플랫폼은 하위 라인이 제공되면 이를 렌더링해야 한다(SHOULD).

#### 예시 { #examples }

**최상위에서 항목화한 분할 세금:**

<!-- ucp:example schema=shopping/checkout target=$.totals op=read -->
```json
[
  { "type": "subtotal",    "display_text": "Subtotal",    "amount": 5750 },
  { "type": "fulfillment", "display_text": "Shipping",    "amount": 899 },
  { "type": "tax",         "display_text": "Federal Tax", "amount": 332 },
  { "type": "tax",         "display_text": "State Tax",   "amount": 465 },
  { "type": "total",       "display_text": "Total",       "amount": 7446 }
]
```

**선택적 내역이 포함된 통합 수수료:**

<!-- ucp:example schema=shopping/checkout target=$.totals op=read -->
```json
[
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

<!-- ucp:example schema=shopping/checkout target=$.totals op=read -->
```json
[
  { "type": "subtotal",       "display_text": "Subtotal",       "amount": 10000 },
  { "type": "discount",       "display_text": "Summer Sale",    "amount": -1500 },
  { "type": "tax",            "display_text": "Tax",            "amount": 680 },
  { "type": "account_credit", "display_text": "Account Credit", "amount": -2500 },
  { "type": "total",          "display_text": "Amount Due",     "amount": 6680 }
]
```

### UCP 체크아웃 응답 {: #ucp-response-checkout-schema }

{{ extension_schema_fields('ucp.json#/$defs/response_checkout_schema', 'shopping/checkout') }}

### 주문 확인 { #order-confirmation }

{{ schema_fields('order_confirmation', 'shopping/checkout') }}

### 에러 응답 <span id="error-response"></span> { #error-response }

{{ schema_fields('types/error_response', 'shopping/checkout') }}
