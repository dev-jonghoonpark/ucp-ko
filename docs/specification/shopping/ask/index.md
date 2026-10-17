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

# Ask 기능 { #ask-capability }

Shopping은 자연어 질문에 Common [Ask 기능](../../common/ask/index.md)인
`dev.ucp.common.ask`를 사용한다. 규범적 계약은 Common 명세가 정의하며, 이
페이지는 Shopping의 관심사를 각각을 담당하는 기능(Capability)에 대응시킨다.

`ask`는 구매자가 쇼핑 중에 갖는 열린 질문(치수감, 소재, 호환성, 반품·배송
정책, 매장 영업시간 등)에 답한다. 필요하면 `ids`에 지정된 Shopping 리소스를
근거로 삼는다. 답변은 참고용이며, 해당 리소스를 담당하는 구조화된 기능이 여전히
그 리소스에 대한 권위를 가진다([답변](../../common/ask/index.md#answer) 참고).

Shopping에서 `ask`는 비즈니스와 플랫폼이 협상한 기능과 확장을 보완한다.
[카탈로그](../catalog/index.md)가 표현하는 상품 정보,
[풀필먼트](../extensions/fulfillment.md)가 표현하는 배송 선택지,
[할인](../extensions/discount.md)이 표현하는 프로모션, 또는
[장바구니](../cart/index.md), [체크아웃](../checkout/index.md),
[주문](../order/index.md)이 표현하는 상태를 설명할 수 있다. 이는 예시일 뿐
라우팅 레지스트리가 아니다. 구조화된 필드나 작업을 정의하는 기능 또는 확장이
여전히 그에 대한 권위를 가진다.

구조화된 상태를 바꾸는 요청(상품 추가, 할인 적용, 풀필먼트 선택, 체크아웃 완료,
주문 취소 등)은 이를 담당하는 기능 또는 확장이 별도 작업으로 처리한다. `ask`는
그 작업을 수행하지 않는다. 답변은 아무것도 바뀌지 않았음을 밝히며, 선택적으로
`operation_not_performed` 메시지를 함께 보낸다([개요](../../common/ask/index.md#overview)와
[잘 알려진 코드](../../common/ask/index.md#well-known-codes) 참고).

## 예시 { #example }

겨울 재킷을 보던 구매자가 사이즈가 작게 나오는지, 세일 상품도 반품할 수 있는지
묻는다.

### 요청 { #request }

<!-- ucp:example schema=common/ask def=ask_request op=create direction=request -->
```json
{
  "query": "Does this jacket run small, and can sale items be returned?",
  "ids": ["gid://business.example/Product/alpine-shell"]
}
```

### 응답 { #response }

<!-- ucp:example schema=common/ask def=ask_response op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": {
      "dev.ucp.common.ask": [
        {"version": "{{ ucp_version }}"}
      ]
    }
  },
  "answer": {
    "plain": "The Alpine Shell runs true to size. Sale items can be returned within 14 days for store credit."
  },
  "links": [
    {
      "type": "product",
      "url": "https://business.example.com/products/alpine-shell",
      "title": "Alpine Shell Jacket",
      "id": "gid://business.example/Product/alpine-shell"
    },
    {
      "type": "refund_policy",
      "url": "https://business.example.com/policies/refunds",
      "title": "Refund Policy"
    }
  ]
}
```

이후 재킷을 장바구니에 담는 요청은 Cart 기능을 사용한다. `ask`는 그 작업을
수행하지 않는다.
