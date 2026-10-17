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

# 질문 기능 { #ask-capability }

* **기능 이름:** `dev.ucp.common.ask`

## 개요 { #overview }

질문(Ask) 기능은 비즈니스의 리소스, 정책, 서비스에 관한 자연어 질의응답을
제공한다. 플랫폼은 구매자를 대신해 자유 형식의 질문을 보내고, 텍스트 답변과 선택적인
관련 링크 묶음을 받는다. 질문은 상품, 매장 위치, 주문, 예약처럼 특정 비즈니스
리소스에 근거할 수도 있고(`ids` 사용), 반품 정책 질문처럼 비즈니스 전반에 해당할 수도
있다. `ask`는 UCP의 구조화된 커머스 기능을 보완하는 열린 질문용 기능이다. 다른 기능이
노출하는 구조화된 리소스로는 바로 얻기 어려운 열린 질문과 비즈니스 고유의 사실·지식을
다룬다.

Ask는 공통(Common) 기능이다. 특정 버티컬 서비스가 아니라 `dev.ucp.common` 서비스에
바인딩되며, 쇼핑이든 숙박이든 비즈니스가 노출하는 모든 버티컬의 리소스에 관한 질문에
하나의 기능이 답한다(자세한 내용은
[구조화된 기능과의 관계](#relationship-to-structured-capabilities) 참고).

`ask`는 답할 뿐 행동하지 않는다. 비즈니스는 `ask` 요청에 대한 응답으로 다른 UCP 기능이나
확장이 노출하는 리소스의 상태를 변경해서는 안 된다(**MUST NOT**). 예를 들어 `ask`
요청은 장바구니, 체크아웃, 주문, 예약을 생성하거나 수정하지 않고, 할인을 적용하지
않으며, 재고를 확보하지도 않는다.
`query`의 전부 또는 일부가 다른 기능이나 확장이 담당하는 작업을 요청하는 경우("이거 두
개 장바구니에 담아 줘. 파란색도 있어?"), 비즈니스는 `ask`가 요청된 작업을 수행하지
않았음을 분명히 밝히는 `answer`를 반드시 채워서 반환해야 한다(**MUST**). 비즈니스는
[접근](#access)에 따라 사용이 허용된 데이터로 질문의 정보성 부분에 답할 수
있고(**MAY**), 코드가 `operation_not_performed`인 정보 메시지를 추가할 수
있다(**MAY**)([메시지와 오류 처리](#messages-and-error-handling) 참고).

플랫폼은 `answer`나 해당 메시지를 작업이 실행되었다는 증거로, 또는 작업을 수행하라는
지시로 취급해서는 안 된다(**MUST NOT**). 해당 작업을 담당하는 기능이나 확장을 호출할지는
플랫폼이 스스로 결정한다. 이 결정은 `ask` 응답이 아니라 구매자의 원래 요청을 근거로
내려야 하며, 그 기능이나 확장의 인가, 동의, 멱등성 요구 사항을 따른다
([보안 고려 사항](#security-considerations) 참고).

`ask`는 [접근](#access) 규칙에 따라 공개된 비즈니스 정보와, 호출자가 보유한 전역
ID(GID)나 기타 식별자로 지정할 수 있는 모든 리소스를 활용한다
([질문 범위 지정](#scoping-a-question) 참고).

대표적인 사용 사례는 다음과 같다.

* 상품 질문 — 사이즈 핏, 소재, 호환성, 비교.
* 정책 질문 — 반품, 배송, 보증, 환불.
* FAQ / 사용 방법 — 매장 영업시간, 배송 소요 기간, 사용 가능한 결제 방법.
* `ids`로 근거를 지정한 특정 리소스에 대한 확인 — 구매 전 상품, 매장의 주차나
  접근성, 예약의 취소 조건.

## 작업 { #operation }

| 작업 | 설명 |
| :--- | :--- |
| **Ask** | 자연어로 질문하고, 선택적인 관련 링크 묶음과 함께 답변을 받는다. |

### 요청 { #request }

{{ extension_schema_fields('ask.json#/$defs/ask_request', 'common/ask') }}

### 응답 { #response }

{{ extension_schema_fields('ask.json#/$defs/ask_response', 'common/ask') }}

## 질문 범위 지정 { #scoping-a-question }

요청에는 자연어 `query`와, 선택적으로 질문 대상 리소스를 식별하는 `ids` 묶음이 담긴다.
`ids`가 있으면 답변은 특정 리소스에 근거하고("이거 세탁 가능해?", "이 매장에 주차할 수
있어?"), 없으면 비즈니스 전반에 적용된다("반품 정책이 어떻게 돼?").

`ids`는 **선택 사항**이며, 상품이나 변형(variant), 매장 위치, 장바구니, 주문, 예약 등
비즈니스가 보유한 모든 리소스를 참조할 수 있다. 비즈니스는 해당 리소스의 접근 정책이
달리 정하지 않는 한 자체 GID를 받아들여야 하며(**SHOULD**)([접근](#access) 참고),
SKU, 핸들, URL 같은 인식 가능한 보조 식별자도 받아들일 수 있다(**MAY**). GID는 그것을
발급한 비즈니스에게 스스로를 설명하므로, 플랫폼은 GID를 불투명한(opaque) 근거 참조로
보낼 수 있다. 식별자를 인식하지 못하거나 사용을 거절하는 비즈니스도 답할 수 있는 만큼은
답하며, 해석되지 않은 참조를 정보 메시지로 알릴 수 있다(**MAY**)
([메시지와 오류 처리](#messages-and-error-handling) 참고).

**향후 방향.** 이후 버전에서는 사진을 보여 주며 상품을 묻는 것처럼 멀티모달 근거
지정을 위해 이미지 등을 담는 `attachments` 배열이 추가될 수 있다.

## 접근 { #access }

비즈니스가 `ask`를 통해 공개할 수 있는 내용은 현재 요청에 부여한 접근 권한에 따라
달라진다. `ask`는 공개 정보를 활용할 수 있고, 인가된 경우에는 다른 UCP 기능이 표현하는
데이터를 포함해 개인화되거나 보호된 데이터도 활용할 수 있다. 인가를 받더라도 `ask`의
읽기 전용 경계는 바뀌지 않는다. 어떤 자격 증명이 제시되든 비즈니스는 `ask`에 대한
응답으로 다른 UCP 기능이 노출하는 리소스의 상태를 변경해서는 안 된다(**MUST NOT**).
그런 상태 변경에는 해당 작업을 담당하는 기능에 대한 별도 요청이 필요하다. 보호된
데이터를 답변에 사용했다고 해서 그 답변이 해당 데이터를 표현하는 기능보다 권위를 갖게
되지는 않는다([답변](#answer) 참고).

* **공개** — 자격 증명이 없으면 `ask`는 공개된 비즈니스 정보와 공개 리소스로 답한다.
* **리소스 참조** — `ids`의 식별자는 질문을 특정 리소스에 근거하게 한다. 장바구니나
  체크아웃 같은 보호된 리소스를 사용하기 전에 비즈니스는 반드시 그 리소스에 대한 일반
  접근 정책을 적용해야 한다(**MUST**). 정책에 따라 비즈니스는 식별자 보유만으로
  충분하다고 보거나, 추가 자격 증명을 요구하거나, 답변에 해당 리소스를 사용하지 않을 수
  있다(**MAY**)([메시지와 오류 처리](#messages-and-error-handling) 참고).
* **인증된 구매자** — 호출자가 비즈니스가 인식하는 사용자 신원 토큰을 제시하면
  ([사용자 인증 접근](../identity-linking/index.md#access-levels);
  [스코프](#scopes) 참고), 비즈니스는 정책이 허용하는 범위에서 회원 가격, 이용 권한,
  제한된 재고 정보, 보호된 리소스에서 도출한 정보 등 개인화된 답변을 반환할 수
  있다(**MAY**). 이 등급이 `dev.ucp.common.ask:read` 스코프다.

`dev.ucp.common.ask:read` 스코프는 개인화된 `ask` 응답을 허용할 뿐, 구매자의 모든
데이터에 대한 포괄적 권한이 아니다. 답변에 사용하는 모든 보호된 리소스와 모든 정보 항목을
인가할 책임은 여전히 비즈니스에 있다.

## 대화 { #conversation }

비즈니스는 응답에 불투명한 `id`와 선택적 `expires_at`을 가진 객체인 `conversation`을
반환해 멀티턴 대화를 지원할 수 있다(**MAY**). 플랫폼이 후속 `ask`에 그 `conversation`을
다시 보내면, 비즈니스는 접근 정책이 허용하는 한 대화를 이어 가며, 후속 질문은 현재 요청이
사용할 수 있는 이전 턴을 바탕으로 한다. `conversation`을 생략하면 새 대화가 시작된다.
`id`는 불투명 값이다. 플랫폼은 이를 파싱하거나 직접 만들어서는 안 되며(**MUST NOT**),
비즈니스가 반환한 값만 반드시 그대로 다시 보내야 한다(**MUST**).

`conversation`을 반환하는 비즈니스는 같은 `id`로 들어온 후속 요청이 이전 대화를 이어 갈
수 있도록, 그 대화 이력을 `expires_at`까지(또는 `expires_at`이 생략된 경우 자체 정책에
따라) 보관해야 한다(**SHOULD**). 전달된 `id`를 해석할 수 없으면 비즈니스는 새 대화를
시작하고, 전달된 `conversation`을 찾지 못했다는 정보 메시지를 `messages`에 추가해야
한다(**SHOULD**).

각 후속 요청은 인가 측면에서 새로운 요청이다. 비즈니스는 해당 요청과 함께 제시된 자격
증명(있는 경우)을 사용해 반드시 접근 정책([접근](#access))을 적용해야 한다(**MUST**).
`id`는 맥락을 전달할 뿐 권한을 전달하지 않는다. 비즈니스는 다시 전달된
`conversation`이 이전 턴에서 부여된 접근 권한을 이어받는다고 보거나 현재 호출자의 접근
범위를 넓힌다고 취급해서는 안 되며(**MUST NOT**), 현재 요청이 접근할 권한이 없는 보관된
맥락을 사용하거나 공개해서도 안 된다(**MUST NOT**).

접근 정책이 허용하는 경우, 비즈니스는 `id` 보유만으로 대화를 이어 갈 수 있다(**MAY**).
[멀티턴 예시](rest.md#multi-turn-example)처럼 공개 정보에 대한 익명 대화가 그런 예다.
후속 요청의 인가 상태가 이전 턴과 다르면(예: 한 턴에는 자격 증명을 제시했지만 다음 턴에는
제시하지 않은 경우), 비즈니스는 현재 요청이 사용할 수 있는 맥락만으로 대화를 이어 가거나
접근 정책에 따라 대화 연장을 거절할 수 있다(**MAY**)
([메시지와 오류 처리](#messages-and-error-handling) 참고).

플랫폼은 재시도된 턴이 중복 턴을 추가하지 않고 이전 답변을 반환하도록, `conversation`을
담은 모든 요청에 멱등성 키를 포함해야 한다(**SHOULD**).
[REST](rest.md#http-headers)에서는 `Idempotency-Key`로,
[MCP](mcp.md#request-metadata)에서는 `meta["idempotency-key"]`로 전달한다. 키로 중복
요청을 식별한 비즈니스는 두 번째 턴을 추가해서는 안 된다(**MUST NOT**).

{{ extension_schema_fields('ask.json#/$defs/conversation', 'common/ask') }}

## 답변 { #answer }

답변은 비즈니스가 작성하는 자유 형식 콘텐츠다. 비즈니스는 모든 `answer`에 답변의 일반
텍스트인 `plain`을 반드시 포함해야 하며(**MUST**), 같은 답변의 대체 인코딩으로
`markdown`이나 `html`을 추가로 포함할 수 있다(**MAY**). `plain`은 범용 폴백이다.
플랫폼은 자신이 지원하고 신뢰를 보증할 수 있는 더 풍부한 인코딩을 렌더링할 수
있고(**MAY**), 그렇지 않으면 인식하지 못하는 인코딩은 무시하고 `plain`을 렌더링한다.
`plain`이 없는 `answer`는 스키마상 유효하지 않으며 다른 잘못된 페이로드와 똑같이
거부된다.

답변은 참고용이며 권위 있는 정보가 아니다. `answer`에 언급된 가격, 재고, 합계, 세금,
풀필먼트 예상치, 정책 조건은 확약이 아니다. `answer`가 해당 리소스를 담당하는 기능
(`catalog`, `cart`, `checkout`, `order`, `booking` 등)이 반환한 표현과 충돌하면 그
표현이 권위를 가지며, 플랫폼은 반드시 그것을 우선해야 한다(**MUST**)
([구조화된 기능과의 관계](#relationship-to-structured-capabilities) 참고).
플랫폼은 `answer`를 안전, 알레르기 유발 물질, 규제 관련 주장에 대한 구속력 있는 고지로
취급해서는 안 된다(**MUST NOT**).

답변이 유용하고 신뢰할 수 있도록, 비즈니스는 다음을 따라야 한다(**SHOULD**).

* 답변은 대화에 맞는 분량으로 유지하고, 사이즈 표나 호환성 표 같은 리소스는 본문에
  넣지 말고 링크로 연결한다.
* 플랫폼이 구매자를 안내할 수 있도록 답변의 권위 있는 출처(정책 페이지, 상품 자체)에
  링크한다.
* 안전, 알레르기 유발 물질, 규제 관련 고지는 답변 텍스트에만 넣지 말고
  `presentation: "disclosure"` 경고로 전달한다.
* 답할 수 없는 질문이면 그 사실을 분명히 밝힌다.

{{ extension_schema_fields('ask.json#/$defs/ask_response/properties/answer', 'common/ask') }}

## 컨텍스트 { #context }

질문에 대한 시장 및 현지화 컨텍스트로, 국가, 언어, 의도 등이 해당한다. 이 값들은 잠정적
신호다. 비즈니스는 더 신뢰도 높은 입력이 있을 때 이를 무시하거나 우선순위를 낮출 수
있다(**MAY**). 질문의 *대상*이 되는 리소스는 `context`가 아니라 `ids`에 담긴다.

{{ schema_fields('types/context', 'common/ask') }}

## 신호 { #signals }

인가와 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터다. 신호 값은 구매자가 스스로
주장한 내용이어서는 안 된다(**MUST NOT**). 자세한 내용과 개인정보 보호 요구 사항은
[신호](../../overview/index.md#signals)를 참고한다.

{{ schema_fields('types/signals', 'common/ask') }}

## 링크 { #links }

`links`는 답변이 언급하는 엔터티(상품, 매장 위치, 주문, 정책 페이지 등) 가운데 주소로
지정할 수 있는 리소스나 페이지가 있는 것을 나열한다. 플랫폼이 답변과 답변에 언급된
리소스를 연결할 수 있도록, 비즈니스는 그런 엔터티마다 링크를 하나씩 반환해야
한다(**SHOULD**). 플랫폼은 후속 조치에 링크를 사용할 수 있다(**MAY**).

링크에는 다음이 담긴다.

* `title` — 표시용 텍스트로, 플랫폼이 링크를 자신이 렌더링한 텍스트와 연결할 수 있도록
  링크가 가리키는 리소스(구매자가 방금 들은 상품, 정책, 페이지)를 `answer`에 나타난
  그대로 담아야 한다(**SHOULD**).
* `url` — 플랫폼이 구매자를 안내할 수 있는 페이지. 모든 링크에 필수이며, 링크에 `id`가
  없거나 `id`가 해석되지 않을 때 폴백이 된다.
* `id` — 링크가 주소 지정이 가능한 UCP 리소스를 가리키는 경우, 비즈니스는 `url`과 함께
  해당 리소스의 GID를 포함해야 한다(**SHOULD**). 단, 정책 페이지처럼 리소스에 UCP
  식별자가 없으면 `id`를 생략한다. 플랫폼은 최선의 노력(best-effort)으로 GID를 사용해,
  비즈니스와 협상한 기능이 노출하는 적절한 작업에 리소스를 매칭할 수 있다(**MAY**).
  UCP는 플랫폼이 이 매칭을 수행하는 방식을 규정하지 않으며 모든 GID가 해석된다고
  보장하지도 않는다. `url`은 여전히 폴백으로 남는다. 매칭된 리소스로 무엇을 할지는
  플랫폼이 스스로 결정한다([보안 고려 사항](#security-considerations) 참고).

각 링크에는 `type` 분류자도 담긴다. 잘 알려진 값은 `refund_policy`,
`shipping_policy`, `privacy_policy`, `terms_of_service`, `faq`이며, 비즈니스는 다른
`type` 값(상품, 사이즈 가이드, 매장 찾기 페이지 등)을 제공할 수 있다(**MAY**). `type`은
표시용 힌트일 뿐 라우팅 판별자가 아니며, `id`를 작업에 매칭하는 데는 영향을 주지 않는다.
플랫폼은 인식하지 못하는 `type`을 만나도 응답을 거부하지 말고, 링크를 `title`로
표시하거나 생략하는 식으로 유연하게 처리해야 한다(**SHOULD**).

{{ schema_fields('types/link', 'common/ask') }}

## 구조화된 기능과의 관계 { #relationship-to-structured-capabilities }

`ask`와 UCP의 구조화된 기능(`catalog`, `cart`, `checkout`, `order`, `booking` 등)은
서로 독립적으로 도입할 수 있다. 비즈니스는 `ask`만 제공하거나 이들 중 어느 것과 함께
제공할 수 있으며(**MAY**), 하나를 광고한다고 해서 다른 것을 지원한다는 뜻은 아니다.
`ask`는 `catalog`나 특정 리소스 도메인에 묶여 있지 않다. 질문은 비즈니스가 노출하는 어느
도메인의 리소스에든 근거할 수 있고, 답변도 어느 도메인의 리소스에든 링크할 수 있다.

둘은 서로 다른 필요를 충족한다. `ask`는 사람이 읽을 수 있는 참고용 콘텐츠를 반환하고,
구조화된 기능은 기계가 읽을 수 있는 리소스 표현을 반환하며 후자가 `answer`보다 권위를
갖는다([답변](#answer) 참고).

버티컬 통합 가이드는 이 계약을 바꾸지 않고 버티컬의 관심사를 해당 기능에 대응시킨다.
쇼핑 버티컬은 [쇼핑에서의 Ask](../../shopping/ask/index.md)를 참고한다.

## 메시지와 오류 처리 { #messages-and-error-handling }

비즈니스는 답변에 관한 오류, 경고, 정보성 메모를 담은 `messages` 배열을 포함할 수
있다(**MAY**). 예를 들어 지역별 정책 변형을 근거로 했다는 경고나, 질문의 범위를 재조정했다는
메모가 해당한다. 기본 응답은 여전히 `answer`다.

| 유형 | 사용 시점 | 예시 코드 |
| :--- | :--- | :--- |
| `error` | 비즈니스 수준 오류 — 예: 질문에 호출자 토큰에 없는 스코프가 필요한 경우 | `insufficient_scope` |
| `warning` | 답변에 관한 중요한 조건이나 면책 고지 | `disclaimer` |
| `info` | 추가 맥락이나 진행을 막지 않는 메모 — 예: 참조된 리소스를 해석할 수 없거나, `query`가 `ask`가 수행하지 않는 작업을 요청한 경우 | `not_found`, `operation_not_performed`, `promotion` |

`presentation: "disclosure"`가 지정된 경고는 플랫폼이 숨기거나 닫아서는 안 되는
(**MUST NOT**) 고지를 담는다. 안전, 알레르기 유발 물질, 규제 대상 고지 등이 그 예다.
렌더링 계약은 [경고 표시](../../shopping/checkout/index.md#warning-presentation)를
참고한다.

### 잘 알려진 코드 { #well-known-codes }

메시지 코드는 열린 문자열이어서 공용 메시지 스키마는 어떤 코드든 받아들이며, 잘 알려진
코드는 표준화된 의미를 갖는다. `ask`는 다음 하나를 정의한다.

| 유형 | 코드 | 의미 |
| :--- | :--- | :--- |
| `info` | `operation_not_performed` | `query`가 `ask`의 읽기 전용 경계를 벗어나는 작업을 요청했으며, 비즈니스는 그에 해당하는 상태 변경을 수행하지 않았다. |

이 메시지는 아무것도 바뀌지 않았음을 알린다. 플랫폼에게 다른 기능을 호출하라고 지시하지
않으며, `ask`가 작업을 수행하지 않았다고 밝히는 `answer`를 보완할 뿐 결코 대체하지
않는다([개요](#overview) 참고).

예를 들어 구매자가 "위젯 세 개 더 담고, 그러면 대량 구매 할인 대상이 되는지 알려 줘"라고
묻는다고 하자. 이 `query`에는 `ask`가 수행하지 않는 작업(장바구니에 추가)과 답할 수 있는
질문(할인 조건)이 섞여 있다. 비즈니스는 질문에 답하면서 장바구니를 변경하지 않았다고
밝히고, 같은 사실을 정보 메시지로도 반복한다. 비즈니스는 장바구니 작업을 전혀 수행하지
않으며, 응답의 어떤 내용도 플랫폼을 그런 작업을 수행할 기능으로 유도하지 않는다.

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
    "plain": "I haven't added anything to your cart; it is unchanged. Orders of 10 or more widgets qualify for the volume discount, which takes 15% off each widget."
  },
  "messages": [
    {
      "type": "info",
      "code": "operation_not_performed",
      "content": "No cart operation was performed."
    }
  ]
}
```

### 메시지(오류) { #message-error }

{{ schema_fields('types/message_error', 'common/ask') }}

### 메시지(경고) { #message-warning }

{{ schema_fields('types/message_warning', 'common/ask') }}

### 메시지(정보) { #message-info }

{{ schema_fields('types/message_info', 'common/ask') }}

## 보안 고려 사항 { #security-considerations }

`ask`는 양방향으로 당사자 경계를 넘어 자연어를 전달한다. 구매자의 질문은 비즈니스의
모델에 도달하고, 비즈니스의 답변은 플랫폼의 모델에 도달한다. 둘 다 신뢰할 수 없는
콘텐츠이며, 반드시 지시가 아닌 데이터로 취급해야 한다(**MUST**).

비즈니스는 반드시 `query`를 신뢰할 수 없는 입력으로 취급해야 한다(**MUST**). 질문이 자신의
지시나 인가 결정을 바꾸게 해서는 안 되며(**MUST NOT**), [접근](#access)의 등급은 반드시
모델 바깥에서 강제해야 한다(**MUST**). 호출자가 볼 수 있는 범위는 요청과 함께 제시된 자격
증명으로 결정되며, 질문 내용이나 요청이 다시 전달한 `conversation`으로 결정되어서는
안 된다([대화](#conversation) 참고). 인가는 모델의 동작에 의존해서는 안
된다(**MUST NOT**).

플랫폼은 협상된 확장이 제공한 콘텐츠를 포함해, 비즈니스가 작성한 모든 응답 콘텐츠를
반드시 지시가 아닌 데이터로 취급해야 한다(**MUST**). 플랫폼은 그 콘텐츠를 수신하거나
렌더링하는 것만으로 기능 호출이나 상태 변경이 일어나게 해서는 안 된다(**MUST NOT**).

`answer`, `links[].id`의 식별자, 확장이 정의한 어포던스에 따라 행동할지와 어떻게 행동할지는
플랫폼이 자체 인가 및 구매자 동의 규칙에 따라 결정한다. 플랫폼은 `answer`를 자신의 출력과
구별되는 비즈니스 제공 콘텐츠로 표시해야 한다(**SHOULD**).

## 스코프 { #scopes }

질문 기능은 사용자 인증 접근을 위해 다음과 같은 잘 알려진 스코프를 정의한다.

| 스코프 | 설명 |
| :--- | :--- |
| `dev.ucp.common.ask:read` | 인증된 구매자를 대신해 질문한다 — 회원 가격, 이용 권한, 제한된 재고 정보를 반영한 개인화된 답변. |

스코프 선언, 도출, 그리고 사용자 정의 스코프로 이 집합을 확장하는 규칙은
[신원 연결 — 스코프](../identity-linking/index.md#scopes)에 정의되어 있다.

## 전송 바인딩 { #transport-bindings }

이 기능은 다음 전송 프로토콜에 바인딩된다.

* [REST 바인딩](rest.md): RESTful API 매핑(`POST /ask`).
* [MCP 바인딩](mcp.md): JSON-RPC를 통한 Model Context Protocol 매핑(`ask_business` 도구).
