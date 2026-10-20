# 질문 기능

- **기능 이름:** `dev.ucp.common.ask`

## 개요

질문(Ask) 기능은 비즈니스의 리소스, 정책, 서비스에 관한 자연어 질의응답을 제공합니다. 플랫폼은 구매자를 대신해 자유 형식의 질문을 보내고, 텍스트 답변과 선택적인 관련 링크 묶음을 받습니다. 질문은 상품, 매장 위치, 주문, 예약처럼 특정 비즈니스 리소스를 근거로 할 수도 있고(`ids` 사용), 반품 정책 질문처럼 비즈니스 전반에 관한 것일 수도 있습니다. `ask`는 UCP의 구조화된 커머스 기능을 보완하는 열린 질문용 기능입니다. 다른 기능이 노출하는 구조화된 리소스만으로는 바로 답하기 어려운 열린 질문과 비즈니스 고유의 사실·지식을 다룹니다.

Ask는 공통(Common) 기능입니다. 특정 버티컬 서비스가 아니라 `dev.ucp.common` 서비스에 바인딩되므로, 쇼핑이든 숙박이든 비즈니스가 노출하는 모든 버티컬의 리소스에 관한 질문에 이 기능 하나로 답합니다(자세한 내용은 [구조화된 기능과의 관계](#relationship-to-structured-capabilities) 참고).

`ask`는 답하기만 하고 행동하지 않습니다. 비즈니스는 `ask` 요청에 응답하면서 다른 UCP 기능이나 확장이 노출하는 리소스의 상태를 변경해서는 안 됩니다(**MUST NOT**). 예를 들어 `ask` 요청으로 장바구니, 체크아웃, 주문, 예약을 생성하거나 수정하지 않고, 할인을 적용하지도 않으며, 재고를 확보하지도 않습니다. `query`의 전부 또는 일부가 다른 기능이나 확장이 담당하는 작업을 요청하면("이거 두 개 장바구니에 담아 줘. 파란색도 있어?") 비즈니스는 `ask`가 요청된 작업을 수행하지 않았다고 분명히 밝히는 `answer`를 반드시 채워서 반환해야 합니다(**MUST**). 비즈니스는 [접근](#access) 규칙상 사용이 허용된 데이터로 질문의 정보성 부분에 답할 수 있고(**MAY**), 코드가 `operation_not_performed`인 정보 메시지를 추가할 수 있습니다(**MAY**)([메시지와 오류 처리](#messages-and-error-handling) 참고).

플랫폼은 `answer`나 해당 메시지를 작업이 실행되었다는 증거나 작업을 수행하라는 지시로 받아들여서는 안 됩니다(**MUST NOT**). 해당 작업을 담당하는 기능이나 확장을 호출할지는 플랫폼이 스스로 결정합니다. 이 결정은 `ask` 응답이 아니라 구매자의 원래 요청을 근거로 내려야 하며, 그 기능이나 확장의 인가, 동의, 멱등성 요구 사항을 따릅니다 ([보안 고려 사항](#security-considerations) 참고).

`ask`는 [접근](#access) 규칙에 따라 공개된 비즈니스 정보와, 호출자가 가진 전역 ID(GID)나 기타 식별자로 지정할 수 있는 모든 리소스를 활용합니다 ([질문 범위 지정](#scoping-a-question) 참고).

대표적인 사용 사례는 다음과 같습니다.

- 상품 질문 — 사이즈 핏, 소재, 호환성, 비교.
- 정책 질문 — 반품, 배송, 보증, 환불.
- FAQ / 사용 방법 — 매장 영업시간, 배송 소요 기간, 사용 가능한 결제 방법.
- `ids`로 근거를 지정한 특정 리소스 확인 — 구매 전 상품, 매장의 주차나 접근성, 예약의 취소 조건.

## 작업

| 작업    | 설명                                                               |
| ------- | ------------------------------------------------------------------ |
| **Ask** | 자연어로 질문하고, 선택적인 관련 링크 묶음과 함께 답변을 받습니다. |

### 요청

| 이름         | 타입                                                            | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | --------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| query        | string                                                          | **Required** | Natural-language question.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| conversation | object                                                          | Optional     | Provide a conversation to continue it on a follow-up; omit to start a new one.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ids          | Array[string]                                                   | Optional     | Optional identifiers that ground the question in any Business resource. A Business SHOULD accept its own Global IDs (GIDs) and MAY accept recognized secondary identifiers such as SKU, handle, or URL. GIDs self-describe to the Business; a Platform may send them as opaque grounding references. Examples include a product, location, cart, order, or booking.                                                                                                                                                                                                                                                                                                     |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context) | Optional     | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals) | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |

### 응답

| 이름         | 타입                                                                     | 필수 여부    | 설명                                                                                                                                                                                                                                                                                |
| ------------ | ------------------------------------------------------------------------ | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | UCP Ask Response Schema                                                  | **Required** | UCP metadata for ask responses.                                                                                                                                                                                                                                                     |
| answer       | object                                                                   | **Required** | Answer provided by the Business. Plain text is required; Markdown and HTML are optional alternative encodings.                                                                                                                                                                      |
| conversation | object                                                                   | Optional     | Business-issued conversation for multi-turn continuation. When returned, the business SHOULD retain its history until `expires_at` (or per its own policy) so a follow-up with the same `id` builds on it. An unresolved `id` starts a new conversation and is noted in `messages`. |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]       | Optional     | Optional links and references related to the answer. When a link refers to an addressable UCP resource, the Business SHOULD include its GID in the link's `id`.                                                                                                                     |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\] | Optional     | Errors, warnings, or informational messages about the answer or its grounding.                                                                                                                                                                                                      |

## 질문 범위 지정

요청에는 자연어 `query`가 담기고, 선택적으로 질문 대상 리소스를 식별하는 `ids` 묶음이 함께 담깁니다. `ids`가 있으면 답변은 특정 리소스를 근거로 하고("이거 세탁 가능해?", "이 매장에 주차할 수 있어?"), 없으면 비즈니스 전반에 적용됩니다("반품 정책이 어떻게 돼?").

`ids`는 **선택 사항**이며, 상품이나 변형(variant), 매장 위치, 장바구니, 주문, 예약 등 비즈니스가 보유한 모든 리소스를 참조할 수 있습니다. 해당 리소스의 접근 정책이 달리 정하지 않는 한 비즈니스는 자체 GID를 받아들여야 하며(**SHOULD**)([접근](#access) 참고), SKU, 핸들, URL처럼 알아볼 수 있는 보조 식별자도 받아들일 수 있습니다(**MAY**). GID는 발급한 비즈니스가 스스로 해석할 수 있으므로, 플랫폼은 GID를 불투명한(opaque) 근거 참조로 보낼 수 있습니다. 식별자를 인식하지 못하거나 사용을 거절하는 비즈니스도 답할 수 있는 만큼은 답하며, 해석되지 않은 참조는 정보 메시지로 알릴 수 있습니다(**MAY**) ([메시지와 오류 처리](#messages-and-error-handling) 참고).

**향후 방향.** 이후 버전에서는 사진을 보여 주며 상품을 묻는 식의 멀티모달 근거 지정을 위해 이미지 등을 담는 `attachments` 배열이 추가될 수 있습니다.

## 접근

비즈니스가 `ask`로 공개할 수 있는 내용은 현재 요청에 부여된 접근 권한에 따라 달라집니다. `ask`는 공개 정보를 활용할 수 있고, 인가된 경우에는 다른 UCP 기능이 표현하는 데이터를 포함해 개인화되거나 보호된 데이터도 활용할 수 있습니다. 인가를 받더라도 `ask`의 읽기 전용 경계는 그대로입니다. 어떤 자격 증명이 제시되든 비즈니스는 `ask`에 응답하면서 다른 UCP 기능이 노출하는 리소스의 상태를 변경해서는 안 됩니다(**MUST NOT**). 그런 상태 변경은 해당 작업을 담당하는 기능에 따로 요청해야 합니다. 보호된 데이터를 답변에 썼다고 해서 그 답변이 해당 데이터를 표현하는 기능보다 더 권위 있는 정보가 되지는 않습니다([답변](#answer) 참고).

- **공개** — 자격 증명이 없으면 `ask`는 공개된 비즈니스 정보와 공개 리소스로 답합니다.
- **리소스 참조** — `ids`의 식별자로 질문의 근거가 되는 특정 리소스를 지정합니다. 장바구니나 체크아웃 같은 보호된 리소스를 사용하기 전에 비즈니스는 반드시 그 리소스의 일반 접근 정책을 적용해야 합니다(**MUST**). 정책에 따라 비즈니스는 식별자를 가진 것만으로 충분하다고 보거나, 추가 자격 증명을 요구하거나, 답변에 해당 리소스를 쓰지 않을 수 있습니다(**MAY**)([메시지와 오류 처리](#messages-and-error-handling) 참고).
- **인증된 구매자** — 호출자가 비즈니스가 인식하는 사용자 신원 토큰을 제시하면 ([사용자 인증 접근](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/identity-linking/#access-levels); [스코프](#scopes) 참고), 비즈니스는 정책이 허용하는 범위에서 회원 가격, 이용 권한, 제한된 재고 정보, 보호된 리소스에서 도출한 정보 등 개인화된 답변을 반환할 수 있습니다(**MAY**). 이 등급에 해당하는 것이 `dev.ucp.common.ask:read` 스코프입니다.

`dev.ucp.common.ask:read` 스코프는 개인화된 `ask` 응답을 허용할 뿐, 구매자의 모든 데이터에 대한 포괄적 권한이 아닙니다. 답변에 쓰는 모든 보호된 리소스와 모든 정보 항목을 인가할 책임은 여전히 비즈니스에 있습니다.

## 대화

비즈니스는 응답에 불투명한 `id`와 선택적 `expires_at`이 담긴 `conversation` 객체를 반환해 멀티턴 대화를 지원할 수 있습니다(**MAY**). 플랫폼이 후속 `ask`에 그 `conversation`을 다시 보내면 비즈니스는 접근 정책이 허용하는 한 대화를 이어 가고, 후속 질문은 현재 요청이 사용할 수 있는 이전 턴을 바탕으로 처리됩니다. `conversation`을 생략하면 새 대화가 시작됩니다. `id`는 불투명 값입니다. 플랫폼은 이 값을 파싱하거나 직접 만들어서는 안 되며(**MUST NOT**), 비즈니스가 반환한 값을 반드시 그대로 다시 보내야 합니다(**MUST**).

`conversation`을 반환하는 비즈니스는 같은 `id`로 들어온 후속 요청이 이전 대화를 이어 갈 수 있도록, 그 대화 이력을 `expires_at`까지(`expires_at`이 없으면 자체 정책에 따라) 보관해야 합니다(**SHOULD**). 전달된 `id`를 해석할 수 없으면 비즈니스는 새 대화를 시작하고, 전달된 `conversation`을 찾지 못했다는 정보 메시지를 `messages`에 추가해야 합니다(**SHOULD**).

후속 요청은 인가 측면에서 매번 새로운 요청입니다. 비즈니스는 해당 요청과 함께 제시된 자격 증명(있는 경우)으로 반드시 접근 정책([접근](#access))을 적용해야 합니다(**MUST**). `id`는 맥락만 전달할 뿐 권한은 전달하지 않습니다. 비즈니스는 다시 전달된 `conversation`이 이전 턴에서 부여된 접근 권한을 이어받는다고 보거나 현재 호출자의 접근 범위를 넓힌다고 취급해서는 안 되며(**MUST NOT**), 현재 요청에 접근 권한이 없는 보관된 맥락을 사용하거나 공개해서도 안 됩니다(**MUST NOT**).

접근 정책이 허용하면 비즈니스는 `id`를 가진 것만으로 대화를 이어 갈 수 있습니다(**MAY**). [멀티턴 예시](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/rest/#multi-turn-example)처럼 공개 정보에 관한 익명 대화가 그런 예입니다. 후속 요청의 인가 상태가 이전 턴과 다르면(예: 한 턴에는 자격 증명을 제시했지만 다음 턴에는 제시하지 않은 경우) 비즈니스는 현재 요청이 사용할 수 있는 맥락만으로 대화를 이어 가거나 접근 정책에 따라 대화 연장을 거절할 수 있습니다(**MAY**) ([메시지와 오류 처리](#messages-and-error-handling) 참고).

재시도된 턴이 중복 턴을 추가하지 않고 이전 답변을 반환하도록, 플랫폼은 `conversation`을 담은 모든 요청에 멱등성 키를 포함해야 합니다(**SHOULD**). [REST](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/rest/#http-headers)에서는 `Idempotency-Key`로, [MCP](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/mcp/#request-metadata)에서는 `meta["idempotency-key"]`로 전달합니다. 키로 중복 요청을 식별한 비즈니스는 두 번째 턴을 추가해서는 안 됩니다(**MUST NOT**).

| 이름       | 타입   | 필수 여부    | 설명                                                                                |
| ---------- | ------ | ------------ | ----------------------------------------------------------------------------------- |
| id         | string | **Required** | Opaque, business-issued conversation identifier.                                    |
| expires_at | string | Optional     | RFC 3339 timestamp after which the conversation context may be discarded. Optional. |

## 답변

답변은 비즈니스가 작성하는 자유 형식 콘텐츠입니다. 비즈니스는 모든 `answer`에 답변의 일반 텍스트인 `plain`을 반드시 포함해야 하며(**MUST**), 같은 답변을 다른 형식으로 인코딩한 `markdown`이나 `html`을 추가로 포함할 수 있습니다(**MAY**). `plain`은 어디서나 쓸 수 있는 폴백입니다. 플랫폼은 자신이 지원하고 신뢰를 보증할 수 있는 더 풍부한 인코딩을 렌더링할 수 있고(**MAY**), 그렇지 않으면 인식하지 못하는 인코딩을 무시하고 `plain`을 렌더링합니다. `plain`이 없는 `answer`는 스키마상 유효하지 않으므로 다른 잘못된 페이로드와 똑같이 거부됩니다.

답변은 참고용이며 권위 있는 정보가 아닙니다. `answer`에 언급된 가격, 재고, 합계, 세금, 풀필먼트 예상치, 정책 조건은 확약이 아닙니다. `answer`가 해당 리소스를 담당하는 기능 (`catalog`, `cart`, `checkout`, `order`, `booking` 등)이 반환한 표현과 충돌하면 그 표현이 우선하며, 플랫폼은 반드시 그 표현을 따라야 합니다(**MUST**) ([구조화된 기능과의 관계](#relationship-to-structured-capabilities) 참고). 플랫폼은 `answer`를 안전, 알레르기 유발 물질, 규제 관련 주장에 대한 구속력 있는 고지로 취급해서는 안 됩니다(**MUST NOT**).

답변이 유용하고 신뢰할 수 있도록 비즈니스는 다음을 따라야 합니다(**SHOULD**).

- 답변은 대화에 맞는 분량으로 유지하고, 사이즈 표나 호환성 표 같은 리소스는 본문에 넣지 말고 링크로 연결합니다.
- 플랫폼이 구매자를 안내할 수 있도록 답변의 권위 있는 출처(정책 페이지, 상품 자체)를 링크합니다.
- 안전, 알레르기 유발 물질, 규제 관련 고지는 답변 텍스트에만 넣지 말고 `presentation: "disclosure"` 경고로 전달합니다.
- 답할 수 없는 질문이면 그렇다고 분명히 밝힙니다.

Answer provided by the Business. Plain text is required; Markdown and HTML are optional alternative encodings.

| 이름     | 타입   | 필수 여부    | 설명                                                                                                                                                                      |
| -------- | ------ | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| plain    | string | **Required** | Plain text content.                                                                                                                                                       |
| html     | string | Optional     | HTML-formatted content. Security: Platforms MUST sanitize before rendering—strip scripts, event handlers, and untrusted elements. Treat all rich text as untrusted input. |
| markdown | string | Optional     | Markdown-formatted content.                                                                                                                                               |

## 컨텍스트

질문의 시장·현지화 컨텍스트로, 국가, 언어, 의도 등이 여기에 해당합니다. 이 값들은 잠정적인 신호입니다. 더 신뢰할 만한 입력이 있으면 비즈니스는 이 값을 무시하거나 우선순위를 낮출 수 있습니다(**MAY**). 질문의 *대상*이 되는 리소스는 `context`가 아니라 `ids`에 담습니다.

| 이름            | 타입                                                                                             | 필수 여부 | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| --------------- | ------------------------------------------------------------------------------------------------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| address_country | string                                                                                           | Optional  | The country, as a 2-letter ISO 3166-1 alpha-2 code (e.g. "US"). A 3-letter alpha-3 code or full country name MAY also be used.                                                                                                                                                                                                                                                                                                                                     |
| address_region  | string                                                                                           | Optional  | The first-level administrative region within the country (e.g. a state or province such as California).                                                                                                                                                                                                                                                                                                                                                            |
| postal_code     | string                                                                                           | Optional  | The postal code (e.g. "94043").                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| location        | string                                                                                           | Optional  | Stable, opaque identifier for a Location in the Business's namespace. This provisional, non-binding hint is distinct from the Buyer's locality. The operation specification or an active capability/extension defines its effects. A common example in retail shopping is the default home store ID selected and saved by the user when purchasing groceries.                                                                                                      |
| intent          | string                                                                                           | Optional  | Background context describing buyer's intent (e.g., 'looking for a gift under $50', 'need something durable for outdoor use'). Informs relevance, recommendations, and personalization.                                                                                                                                                                                                                                                                            |
| language        | string                                                                                           | Optional  | Preferred language for content. Use IETF BCP 47 language tags (e.g., 'en', 'fr-CA', 'zh-Hans'). For REST, equivalent to Accept-Language header—platforms SHOULD fall back to Accept-Language when this field is absent; when provided, overrides Accept-Language. Businesses MAY return content in a different language if unavailable.                                                                                                                            |
| currency        | string                                                                                           | Optional  | Preferred currency (ISO 4217, e.g., 'EUR', 'USD'). Businesses determine presentment currency from context and authoritative signals; this hint MAY inform selection in multi-currency markets. Also serves as the denomination for price filter values — platforms SHOULD include this field when sending price filters. Response prices include explicit currency confirming the resolution.                                                                      |
| eligibility     | Array\[[Reverse Domain Name](/ucp-ko/draft/draft/specification/reference/#reverse-domain-name)\] | Optional  | Buyer claims about eligible benefits such as loyalty membership, payment instrument perks, and similar. Recognized claims MAY inform the Business response (e.g., member-only product availability, adjusted pricing in catalog, provisional discounts at cart or checkout). Businesses MUST ignore unrecognized values without error. Values MUST use reverse-domain naming (e.g., 'com.example.loyalty_gold', 'org.school.student') and MUST be non-identifying. |
| payment         | Array[object]                                                                                    | Optional  | Buyer-preferred payment handlers in priority order (most preferred first). Each entry names a handler advertised in the Business profile's `ucp.payment_handlers`, optionally narrowed to preferred instrument types. The Business SHOULD use it to preselect or prioritize the handler (and type, when given) and MAY ignore unavailable or ineligible entries; unrecognized values MUST be ignored without error.                                                |

## 신호

인가와 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터입니다. 신호 값은 구매자가 스스로 주장한 내용이어서는 안 됩니다(**MUST NOT**). 자세한 내용과 개인정보 보호 요구 사항은 [신호](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#signals)를 참고하세요.

| 이름               | 타입   | 필수 여부 | 설명                                           |
| ------------------ | ------ | --------- | ---------------------------------------------- |
| dev.ucp.buyer_ip   | string | Optional  | Client's IP address (IPv4 or IPv6).            |
| dev.ucp.user_agent | string | Optional  | Client's HTTP User-Agent header or equivalent. |

## 링크

`links`에는 답변이 언급하는 엔터티(상품, 매장 위치, 주문, 정책 페이지 등) 가운데 주소로 가리킬 수 있는 리소스나 페이지가 있는 것을 나열합니다. 플랫폼이 답변과 답변에 언급된 리소스를 연결할 수 있도록 비즈니스는 그런 엔터티마다 링크를 하나씩 반환해야 합니다(**SHOULD**). 플랫폼은 후속 조치에 링크를 사용할 수 있습니다(**MAY**).

링크에 담기는 항목은 다음과 같습니다.

- `title` — 표시용 텍스트. 플랫폼이 링크를 자신이 렌더링한 텍스트와 연결할 수 있도록, 링크가 가리키는 리소스(구매자가 방금 들은 상품, 정책, 페이지)를 `answer`에 나온 그대로 담아야 합니다(**SHOULD**).
- `url` — 플랫폼이 구매자를 안내할 수 있는 페이지. 모든 링크에 필수이며, 링크에 `id`가 없거나 `id`가 해석되지 않을 때 폴백으로 쓰입니다.
- `id` — 링크가 주소 지정 가능한 UCP 리소스를 가리키면 비즈니스는 `url`과 함께 해당 리소스의 GID를 포함해야 합니다(**SHOULD**). 단, 정책 페이지처럼 리소스에 UCP 식별자가 없으면 `id`를 생략합니다. 플랫폼은 최선의 노력(best-effort)으로 GID를 사용해 비즈니스와 협상한 기능이 노출하는 적절한 작업에 리소스를 매칭할 수 있습니다(**MAY**). UCP는 플랫폼이 이 매칭을 어떻게 하는지 규정하지 않으며, 모든 GID가 해석된다고 보장하지도 않습니다. `url`은 여전히 폴백으로 남습니다. 매칭된 리소스로 무엇을 할지는 플랫폼이 스스로 결정합니다([보안 고려 사항](#security-considerations) 참고).

각 링크에는 `type` 분류자도 담깁니다. 잘 알려진 값은 `refund_policy`, `shipping_policy`, `privacy_policy`, `terms_of_service`, `faq`이며, 비즈니스는 다른 `type` 값(상품, 사이즈 가이드, 매장 찾기 페이지 등)을 제공할 수 있습니다(**MAY**). `type`은 표시용 힌트일 뿐 라우팅 판별자가 아니며, `id`를 작업에 매칭하는 데는 영향을 주지 않습니다. 플랫폼은 인식하지 못하는 `type`을 만나도 응답을 거부하지 말고, 링크를 `title`로 표시하거나 생략하는 식으로 유연하게 처리해야 합니다(**SHOULD**).

| 이름  | 타입   | 필수 여부    | 설명                                                                                                                                                                                                                                                         |
| ----- | ------ | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| type  | string | **Required** | Type of link. Well-known values: `privacy_policy`, `terms_of_service`, `refund_policy`, `shipping_policy`, `faq`. Consumers SHOULD handle unknown values gracefully by displaying them using the `title` field or omitting the link.                         |
| url   | string | **Required** | The actual URL pointing to the content to be displayed.                                                                                                                                                                                                      |
| title | string | Optional     | Optional display text for the link. When provided, use this instead of generating from type.                                                                                                                                                                 |
| id    | string | Optional     | Optional identifier for the linked resource when it is an addressable UCP resource (for example, a product or variant), so a platform can act on it through the capability that owns it. Omit for resources without a UCP identifier, such as a policy page. |

## 구조화된 기능과의 관계

`ask`와 UCP의 구조화된 기능(`catalog`, `cart`, `checkout`, `order`, `booking` 등)은 서로 독립적으로 도입할 수 있습니다. 비즈니스는 `ask`만 제공할 수도 있고 이들 중 어느 것과든 함께 제공할 수도 있으며(**MAY**), 하나를 지원한다고 선언했다고 해서 다른 것도 지원한다는 뜻은 아닙니다. `ask`는 `catalog`나 특정 리소스 도메인에 묶여 있지 않습니다. 질문은 비즈니스가 노출하는 어느 도메인의 리소스든 근거로 삼을 수 있고, 답변도 어느 도메인의 리소스든 링크할 수 있습니다.

둘이 채우는 필요는 서로 다릅니다. `ask`는 사람이 읽는 참고용 콘텐츠를 반환하고, 구조화된 기능은 기계가 읽는 리소스 표현을 반환합니다. 둘이 다르면 구조화된 기능의 표현이 `answer`보다 우선합니다([답변](#answer) 참고).

버티컬 통합 가이드는 이 계약을 바꾸지 않고 버티컬의 관심사를 해당 기능에 대응시킵니다. 쇼핑 버티컬은 [쇼핑에서의 Ask](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/ask/index.md)를 참고하세요.

## 메시지와 오류 처리

비즈니스는 답변에 관한 오류, 경고, 정보성 메모를 담은 `messages` 배열을 포함할 수 있습니다(**MAY**). 지역별 정책 변형을 근거로 했다는 경고나 질문 범위를 재조정했다는 메모가 그 예입니다. 기본 응답은 여전히 `answer`입니다.

| 유형      | 사용 시점                                                                                                                      | 예시 코드                                           |
| --------- | ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------- |
| `error`   | 비즈니스 수준 오류 — 예: 질문에 호출자 토큰에 없는 스코프가 필요한 경우                                                        | `insufficient_scope`                                |
| `warning` | 답변에 관한 중요한 조건이나 면책 고지                                                                                          | `disclaimer`                                        |
| `info`    | 추가 맥락이나 진행을 막지 않는 메모 — 예: 참조된 리소스를 해석할 수 없거나, `query`가 `ask`가 수행하지 않는 작업을 요청한 경우 | `not_found`, `operation_not_performed`, `promotion` |

`presentation: "disclosure"`가 지정된 경고에는 플랫폼이 숨기거나 닫아서는 안 되는 (**MUST NOT**) 고지가 담깁니다. 안전, 알레르기 유발 물질, 규제 대상 고지 등이 그 예입니다. 렌더링 계약은 [경고 표시](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/#warning-presentation)를 참고하세요.

### 잘 알려진 코드

메시지 코드는 열린 문자열이어서 공용 메시지 스키마는 어떤 코드든 받아들이며, 잘 알려진 코드에는 표준화된 의미가 있습니다. `ask`가 정의하는 코드는 다음 하나입니다.

| 유형   | 코드                      | 의미                                                                                                                   |
| ------ | ------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `info` | `operation_not_performed` | `query`가 `ask`의 읽기 전용 경계를 벗어나는 작업을 요청했고, 비즈니스는 그에 해당하는 상태 변경을 수행하지 않았습니다. |

이 메시지는 아무것도 바뀌지 않았다는 사실을 알립니다. 플랫폼에 다른 기능을 호출하라고 지시하지 않으며, `ask`가 작업을 수행하지 않았다고 밝히는 `answer`를 보완할 뿐 결코 대체하지 않습니다([개요](#overview) 참고).

예를 들어 구매자가 "위젯 세 개 더 담고, 그러면 대량 구매 할인 대상이 되는지 알려 줘"라고 물었다고 합시다. 이 `query`에는 `ask`가 수행하지 않는 작업(장바구니에 추가)과 답할 수 있는 질문(할인 조건)이 섞여 있습니다. 비즈니스는 질문에 답하면서 장바구니를 변경하지 않았다고 밝히고, 같은 사실을 정보 메시지로도 한 번 더 알립니다. 비즈니스는 장바구니 작업을 전혀 수행하지 않으며, 응답의 어떤 내용도 플랫폼이 그런 작업을 수행할 기능을 호출하도록 유도하지 않습니다.

```json
{
  "ucp": {
    "version": "draft",
    "capabilities": {
      "dev.ucp.common.ask": [
        {"version": "draft"}
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

### 메시지(오류)

| 이름         | 타입                                                                  | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ------------ | --------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                | **Required** | **Constant = error**. Message type discriminator.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| code         | [Error Code](/ucp-ko/draft/draft/specification/reference/#error-code) | **Required** | Error code identifying the type of error. Standard errors are defined in capability specifications (see examples) and have standardized semantics; freeform codes are permitted.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| path         | string                                                                | Optional     | RFC 9535 JSONPath to the component the message refers to (e.g., $.line_items[0]).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| content_type | string                                                                | Optional     | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| content      | string                                                                | **Required** | Human-readable message.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| severity     | string                                                                | **Required** | Reflects the resource state and recommended action. 'recoverable': platform can resolve the condition in band, for example by modifying inputs or processing a related Action, and submit a new operation when needed. 'requires_buyer_input': merchant requires information their API doesn't support collecting programmatically (checkout incomplete). 'requires_buyer_review': buyer must authorize before order placement due to policy, regulatory, or entitlement rules. 'unrecoverable': no valid resource exists to act on, retry with new resource or inputs. Errors with 'requires\_*' severity contribute to 'status: requires_escalation'.* *Enum:*\* `recoverable`, `requires_buyer_input`, `requires_buyer_review`, `unrecoverable` |

### 메시지(경고)

| 이름         | 타입                                                                      | 필수 여부    | 설명                                                                                                                                                                                                                                                |
| ------------ | ------------------------------------------------------------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                                    | **Required** | **Constant = warning**. Message type discriminator.                                                                                                                                                                                                 |
| path         | string                                                                    | Optional     | RFC 9535 JSONPath to the component the message refers to (e.g., $.line_items[0]).                                                                                                                                                                   |
| code         | [Warning Code](/ucp-ko/draft/draft/specification/reference/#warning-code) | **Required** | Warning code identifying the type of warning. Standard codes are defined in capability specifications (see examples) and have standardized semantics; freeform codes are permitted.                                                                 |
| content      | string                                                                    | **Required** | Human-readable warning message that MUST be displayed.                                                                                                                                                                                              |
| content_type | string                                                                    | Optional     | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                                                                                      |
| presentation | string                                                                    | Optional     | Rendering contract for this warning. 'notice' (default): platform MUST display, MAY dismiss. 'disclosure': platform MUST display in proximity to the path-referenced component, MUST NOT hide or auto-dismiss. See specification for full contract. |
| image_url    | string                                                                    | Optional     | URL to a required visual element (e.g., warning symbol, energy class label).                                                                                                                                                                        |
| url          | string                                                                    | Optional     | Reference URL for more information (e.g., regulatory site, registry entry, policy page).                                                                                                                                                            |

### 메시지(정보)

| 이름         | 타입                                                                | 필수 여부    | 설명                                                                                                                                                                                           |
| ------------ | ------------------------------------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type         | string                                                              | **Required** | **Constant = info**. Message type discriminator.                                                                                                                                               |
| path         | string                                                              | Optional     | RFC 9535 JSONPath to the component the message refers to (e.g., $.line_items[0]).                                                                                                              |
| code         | [Info Code](/ucp-ko/draft/draft/specification/reference/#info-code) | Optional     | Info code identifying the type of informational message. Standard codes are defined in capability specifications (see examples) and have standardized semantics; freeform codes are permitted. |
| content_type | string                                                              | Optional     | Content format, default = plain. **Enum:** `plain`, `markdown`                                                                                                                                 |
| content      | string                                                              | **Required** | Human-readable message.                                                                                                                                                                        |

## 보안 고려 사항

`ask`는 당사자 경계를 넘어 양방향으로 자연어를 전달합니다. 구매자의 질문은 비즈니스의 모델에 닿고, 비즈니스의 답변은 플랫폼의 모델에 닿습니다. 둘 다 신뢰할 수 없는 콘텐츠이므로 반드시 지시가 아닌 데이터로 취급해야 합니다(**MUST**).

비즈니스는 반드시 `query`를 신뢰할 수 없는 입력으로 취급해야 합니다(**MUST**). 질문 때문에 자신의 지시나 인가 결정이 바뀌어서는 안 되며(**MUST NOT**), [접근](#access)의 등급은 반드시 모델 바깥에서 강제해야 합니다(**MUST**). 호출자가 볼 수 있는 범위는 요청과 함께 제시된 자격 증명으로 정해지며, 질문 내용이나 요청에 다시 담긴 `conversation`으로 정해져서는 안 됩니다([대화](#conversation) 참고). 인가가 모델의 동작에 의존해서는 안 됩니다(**MUST NOT**).

플랫폼은 협상된 확장이 제공한 콘텐츠를 포함해 비즈니스가 작성한 모든 응답 콘텐츠를 반드시 지시가 아닌 데이터로 취급해야 합니다(**MUST**). 플랫폼은 그 콘텐츠를 받거나 렌더링했다는 이유만으로 기능을 호출하거나 상태를 변경해서는 안 됩니다(**MUST NOT**).

`answer`, `links[].id`의 식별자, 확장이 정의한 어포던스를 보고 행동할지, 어떻게 행동할지는 플랫폼이 자체 인가 및 구매자 동의 규칙에 따라 결정합니다. 플랫폼은 `answer`를 자신의 출력과 구별되는 비즈니스 제공 콘텐츠로 표시해야 합니다(**SHOULD**).

## 스코프

질문 기능은 사용자 인증 접근에 쓰는 다음 잘 알려진 스코프를 정의합니다.

| 스코프                    | 설명                                                                                         |
| ------------------------- | -------------------------------------------------------------------------------------------- |
| `dev.ucp.common.ask:read` | 인증된 구매자를 대신해 질문 — 회원 가격, 이용 권한, 제한된 재고 정보를 반영한 개인화된 답변. |

스코프 선언과 도출, 사용자 정의 스코프로 이 집합을 확장하는 규칙은 [신원 연결 — 스코프](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/identity-linking/#scopes)에 정의되어 있습니다.

## 전송 바인딩

이 기능은 다음 전송 프로토콜에 바인딩됩니다.

- [REST 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/rest/index.md): RESTful API 매핑(`POST /ask`).
- [MCP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/ask/mcp/index.md): JSON-RPC를 통한 Model Context Protocol 매핑(`ask_business` 도구).
