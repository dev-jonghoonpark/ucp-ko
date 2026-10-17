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

# 장바구니 기능 { #cart-capability }

* **기능 이름:** `dev.ucp.shopping.cart`

## 개요 { #overview }

장바구니(Cart) 기능은 체크아웃의 복잡함 없이 장바구니를 구성할 수 있게 한다.
[체크아웃](../checkout/index.md)이 결제 핸들러, 상태 수명 주기, 주문 확정을 관리하는
반면, 장바구니는 구매 의사가 확정되기 전에 상품을 모으기 위한 가벼운 CRUD 인터페이스를
제공한다.

**장바구니와 체크아웃 중 무엇을 쓸까:**

* **장바구니**: 사용자가 상품을 둘러보고, 비교하고, 나중을 위해 저장하는 단계다. 결제
  설정이 필요 없다. 플랫폼/에이전트가 상품을 자유롭게 추가, 삭제, 수정할 수 있다.
* **체크아웃**: 사용자가 구매 의사를 밝힌 단계다. 결제 핸들러가 설정되고 상태 수명
  주기가 시작되며, 세션이 완료를 향해 진행된다.

일반적인 흐름: `cart session` &#8594; `checkout session` &#8594; `order`

장바구니가 지원하는 것:

* **점진적 구성**: 여러 세션에 걸쳐 상품 추가/삭제
* **현지화된 예상 금액**: 전체 체크아웃 부담 없이 컨텍스트를 반영한 가격 책정
* **공유**: `continue_url`로 장바구니 공유와 복구 가능

## 장바구니와 체크아웃 비교 { #cart-vs-checkout }

| 항목 | 장바구니 | 체크아웃 |
| ------ | ---- | -------- |
| **목적** | 구매 전 탐색 | 구매 확정 |
| **결제** | 없음 | 필수(핸들러, 결제 수단) |
| **상태** | 이진(존재/찾을 수 없음) | 수명 주기(`incomplete` → `completed`) |
| **완료 작업** | 없음 | 있음 |
| **합계** | 예상치(일부만 있을 수 있음) | 최종 가격 |

## 장바구니에서 체크아웃으로 전환 { #cart-to-checkout-conversion }

장바구니 기능이 협상된 경우, 플랫폼은 체크아웃 생성(Create Checkout) 요청에 `cart_id`를
넣어 장바구니를 체크아웃으로 전환할 수 있다. 장바구니 내용(`line_items`, `context`,
`buyer`)이 체크아웃 세션을 초기화한다.

<!-- ucp:example schema=shopping/cart def=checkout op=create direction=request -->
```json
{
  "cart_id": "cart_abc123",
  "line_items": []
}
```

장바구니 체크아웃 확장은 기본 체크아웃 스키마 위에 조합되므로 `line_items`는 여전히
필수 요청 필드다. 플랫폼은 반드시 line_items를 포함해야 하며(MUST), 이를 빈 배열
(`"line_items": []`)로 설정해야 한다(SHOULD). 비즈니스는 반드시 장바구니 내용을
사용해야 하며(MUST), 체크아웃 페이로드에서 겹치는 필드는 반드시 무시해야 한다(MUST).
`cart_id` 매개변수는 비즈니스 프로필에 장바구니 기능이 공시된 경우에만 사용할 수 있다.

**멱등 전환:**

주어진 `cart_id`에 대해 미완료 체크아웃이 이미 있으면, 비즈니스는 새로 만드는 대신
반드시 기존 체크아웃 세션을 반환해야 한다(MUST). 이렇게 하면 장바구니당 활성 체크아웃이
하나만 유지되어 세션 충돌을 막을 수 있다.

**전환 후 장바구니 수명 주기:**

`cart_id`로 체크아웃을 초기화한 경우, 장바구니와 체크아웃 세션은 체크아웃이 진행되는
동안 연결되어 있어야 한다(SHOULD).

* **체크아웃 진행 중** — 비즈니스는 장바구니를 유지하고, 관련된 체크아웃 수정 사항(수량
    변경, 상품 삭제)을 장바구니에 다시 반영해야 한다(SHOULD). 이를 통해 구매자가
    체크아웃과 스토어프런트를 오가는 스토어프런트 복귀 흐름을 지원한다.

* **체크아웃 완료 후** — 비즈니스는 TTL, 체크아웃 완료, 기타 비즈니스 로직에 따라
    장바구니를 비울 수 있다(MAY). 비워진 장바구니 ID에 대한 이후 작업은 `not_found`를
    반환하며, 플랫폼은 `create_cart`로 새 세션을 시작할 수 있다.

## 수량과 판매 기준 { #quantity-and-sale-basis }

장바구니 라인 아이템은 공통
[수량과 단위](../../overview/index.md#quantities-and-units) 규약을 따른다. 각
`line_items[].quantity`는 상품의 권한 있는 판매 기준에 따른 정수 단계 수다. 비즈니스
응답에서 `line_items[].item.quantity_unit`이 없으면 기본값인 `each` 기준을 뜻하므로
`quantity`는 상품 개수를 센다.

플랫폼 요청에서 `line_items[].item.quantity_unit`을 생략하는 것은 아무것도 주장하지
않는다. 비즈니스는 상품의 권한 있는 판매 기준으로 `quantity`를 해석하므로, 계량 단위로
판매되는 상품에 대한 요청은 `each`를 주장하지 않고도 이 서술자를 생략할 수 있다.
플랫폼은 판매 기준을 명시적으로 주장하기 위해 `item.quantity_unit`을 포함할 수
있다(**MAY**).

판매 기준 디스커버리, 주장 일치 확인, 불일치 시 변환 또는 거부, 응답 반영, 주문 증분
처리, 라인 가격 책정은
[체크아웃 — 수량과 판매 기준](../checkout/index.md#quantity-and-sale-basis)을 따른다.
장바구니 합계는 여전히 예상치다([합계](#total) 참고). 계량 단위 라인은 장바구니 요약에서
라인 아이템 하나로 계산되며, 이때 `quantity`는 상품 개수가 아니라 양이다.

## 액션 { #actions }

장바구니는 응답 전용 `actions` 맵에 처리되지 않은 액션(Action) 인스턴스를 노출한다.
이 맵은 [개요 — 액션](../../overview/index.md#actions)에 정의되어 있다.

장바구니에는 상태 수명 주기가 없다. 각 액션은 해당 액션 유형에 지정된 장바구니 효과만
제한한다. 비즈니스는 처리되지 않은 액션을 이유로 관련 없는 장바구니 작업을 거부해서는
안 된다(**MUST NOT**). 플랫폼은 액션이 처리되지 않은 동안에도 상품을 계속 추가, 삭제,
수정할 수 있다(**MAY**).

액션을 처리한 후 플랫폼은 [장바구니 조회](#get-cart) 또는 이후의 수정 응답으로 최신
장바구니를 가져와야 한다(**SHOULD**).

## 스코프 { #scopes }

장바구니 기능은 사용자 인증 접근을 위해 다음의 잘 알려진 스코프를 정의한다.

| 스코프 | 설명 |
| :--- | :--- |
| `dev.ucp.shopping.cart:manage` | 인증된 사용자를 대신하는 모든 장바구니 작업 — 생성, 조회, 수정, 저장. |

스코프 선언, 도출, 그리고 사용자 정의 스코프로 이 집합을 확장하는 규칙은
[신원 연결 — 스코프](../../common/identity-linking/index.md#scopes)에 정의되어 있다.

## 지침 { #guidelines }

### 플랫폼 { #platform }

* 구매 전 탐색과 세션 유지에 장바구니를 사용할 수 있다(**MAY**).
* 사용자가 구매 의사를 밝히면 장바구니를 체크아웃으로 전환해야 한다(**SHOULD**).
* 비즈니스 UI로 넘기기 위해 `continue_url`을 표시할 수 있다(**MAY**).
* 장바구니가 만료되거나 취소되었을 때 `not_found`를 적절히 처리해야 한다(**SHOULD**).

### 비즈니스 { #business }

* 장바구니 인계와 세션 복구를 위해 `continue_url`을 제공해야 한다(**SHOULD**).
* TODO: `continue_url` 목적지 논의 필요 - 장바구니 vs 체크아웃.
* 계산할 수 있으면 예상 합계를 제공해야 한다(**SHOULD**).
* 주소를 모르면 체크아웃 전까지 풀필먼트 합계를 생략할 수 있다(**MAY**).
* 유효성 검사 경고에 대해 정보성 메시지를 반환해야 한다(**SHOULD**).
* `expires_at`으로 장바구니 만료를 설정할 수 있다(**MAY**).
* `cart_id`로 체크아웃을 초기화한 경우 [장바구니 수명 주기 요구 사항](#cart-to-checkout-conversion)을
    따라야 한다(**SHOULD**).

## 장바구니 스키마 정의 { #cart-schema-definition }

{{ schema_fields('cart_resp', 'shopping/cart') }}

## 작업 { #operations }

장바구니 기능은 다음 논리적 작업을 정의한다.

| 작업 | 설명 |
| :--- | :--- |
| **장바구니 생성(Create Cart)** | 새 장바구니 세션을 만든다. |
| **장바구니 조회(Get Cart)** | 장바구니 세션의 현재 상태를 가져온다. |
| **장바구니 수정(Update Cart)** | 장바구니 세션을 수정한다. |
| **장바구니 취소(Cancel Cart)** | 장바구니 세션을 취소한다. |

### 장바구니 생성 { #create-cart }

라인 아이템과, 현지화된 예상 가격 책정을 위한 선택적 구매자/컨텍스트 정보로 새 장바구니
세션을 만든다.

요청한 상품이 **모두** 구매 불가능하면, 비즈니스는 장바구니 리소스를 만드는 대신 오류
응답을 반환할 수 있다(MAY). `ucp.status`가 주된 판별자이며, `id`가 없다는 점이 일관된
보조 지표가 된다.

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

* [REST 바인딩](rest.md#create-cart)
* [MCP 바인딩](mcp.md#create_cart)

### 장바구니 조회 { #get-cart }

장바구니 세션의 최신 상태를 가져온다. 장바구니가 없거나, 만료되었거나, 취소되었으면
`not_found`를 반환한다.

* [REST 바인딩](rest.md#get-cart)
* [MCP 바인딩](mcp.md#get_cart)

### 장바구니 수정 { #update-cart }

장바구니 세션을 통째로 교체한다. 플랫폼은 반드시 장바구니 리소스 전체를 보내야
한다(**MUST**). 제공된 리소스가 비즈니스 측의 기존 장바구니 상태를 대체한다.

* [REST 바인딩](rest.md#update-cart)
* [MCP 바인딩](mcp.md#update_cart)

### 장바구니 취소 { #cancel-cart }

장바구니 세션을 취소한다. 비즈니스는 반드시 삭제 전의 장바구니 상태를 반환해야 한다(MUST).
이 장바구니 ID에 대한 이후 작업은 `not_found`를 반환해야 한다(SHOULD).

* [REST 바인딩](rest.md#cancel-cart)
* [MCP 바인딩](mcp.md#cancel_cart)

## 엔티티 { #entities }

장바구니는 [체크아웃](../checkout/index.md)과 같은 엔티티 스키마를 재사용한다. 덕분에
장바구니를 체크아웃 세션으로 전환할 때 데이터 구조가 일관되게 유지된다.

### UCP 응답 장바구니 {: #ucp-response-cart-schema }

{{ extension_schema_fields('ucp.json#/$defs/response_cart_schema', 'shopping/cart') }}

### 라인 아이템 { #line-item }

#### 라인 아이템 생성 요청 { #line-item-create-request }

{{ schema_fields('types/line_item_create_req', 'shopping/checkout') }}

#### 라인 아이템 수정 요청 { #line-item-update-request }

{{ schema_fields('types/line_item_update_req', 'shopping/checkout') }}

#### 라인 아이템 { #line-item_1 }

{{ schema_fields('types/line_item_resp', 'shopping/cart') }}

#### 상품 { #item }

{{ schema_fields('types/item_resp', 'shopping/cart') }}

### 구매자 { #buyer }

{{ schema_fields('buyer', 'shopping/checkout') }}

### 컨텍스트 { #context }

{{ schema_fields('context', 'shopping/checkout') }}

### 신호 { #signals }

인가와 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터다. 신호 값은 구매자가 스스로
주장한 내용이어서는 안 된다(MUST NOT). 자세한 내용과 개인정보 보호 요구 사항은
[신호](../../overview/index.md#signals)를 참고한다.

{{ schema_fields('types/signals', 'shopping/checkout') }}

### 어트리뷰션 { #attribution }

플랫폼이 제공하는 유입 경로 및 전환 이벤트 컨텍스트다. 플랫폼이 전달하는 캠페인 ID, 클릭
식별자, 소스/매체 표식이 여기에 해당한다. 자세한 내용과 동의 요구 사항은
[어트리뷰션](../../overview/index.md#attribution)을 참고한다.

{{ schema_fields('types/attribution', 'shopping/checkout') }}

### 합계 { #total }

장바구니와 체크아웃에는 같은 합계 규약이 적용된다. 렌더링 규약, 회계 항등식, 잘 알려진
유형, 반복 유형, 하위 라인 의미론은 [체크아웃 합계](../checkout/index.md#totals)를 참고한다.

{{ schema_fields('types/total_resp', 'shopping/checkout') }}

세금은 계산할 수 있는 경우 포함할 수 있다(MAY). 플랫폼은 장바구니 합계를 예상치로 간주해야
하며(SHOULD), 정확한 세금은 체크아웃에서 계산된다.

### 메시지 { #message }

{{ schema_fields('message', 'shopping/checkout') }}

#### 메시지 오류 { #message-error }

{{ schema_fields('types/message_error', 'shopping/checkout') }}

#### 메시지 정보 { #message-info }

{{ schema_fields('types/message_info', 'shopping/checkout') }}

#### 메시지 경고 { #message-warning }

{{ schema_fields('types/message_warning', 'shopping/checkout') }}

### 링크 { #link }

{{ schema_fields('types/link', 'shopping/checkout') }}

### 정책 { #policy }

이 장바구니의 상품에 적용되는 정책(반품/환불 조건, 보증 등)이다. `applies_to`의 JSONPath
대상은 이 응답의 루트를 기준으로 한다(예: `$.line_items[0]`). 전체 모델은
[정책](../../overview/index.md#policies)을 참고한다.

{{ schema_fields('types/policy', 'shopping/cart') }}
