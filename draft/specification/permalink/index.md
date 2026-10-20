# 퍼머링크 기능

- **기능 이름:** `dev.ucp.shopping.permalink`
- **스키마:** `https://ucp.dev/draft/schemas/shopping/permalink.json`

## 개요

UCP 쇼핑 퍼머링크(permalink)는 **브라우저로 열 수 있는 쇼핑 의도**입니다. 전달받은 입력으로 쇼핑 상태를 초기화한 뒤, 브라우저를 알맞은 구매자용 목적지로 보냅니다.

퍼머링크는 REST API 작업이 아니라 **GET 브라우저 내비게이션**입니다. 비즈니스는 퍼머링크 요청을 받으면 브라우저를 리디렉션하는 방식으로 처리합니다. 초기화된 쇼핑 상태는 장바구니, 체크아웃, 세션 상태 등 어떤 내부 표현으로든 보관할 수 있으며(MAY), 퍼머링크 규약이 특정 UCP 리소스를 요구하지는 않습니다.

비즈니스는 퍼머링크가 열렸다고 해서 이를 주문 생성, 결제 청구, 체크아웃 완료를 승인한 것으로 취급해서는 안 되며(MUST NOT), UCP JSON 리소스를 주 응답으로 반환해서도 안 됩니다(MUST NOT).

이 기능에서 정의하는 내용은 다음과 같습니다.

- 퍼머링크 엔드포인트 디스커버리
- 간결한 상품 경로 문법
- `continue_to` 목적지 선호
- 초기화된 쇼핑 상태와 UCP 필드 경로의 매핑
- 리디렉션 처리 의미론

## 디스커버리

비즈니스는 `dev.ucp.shopping.permalink`로 퍼머링크 지원 여부를 알립니다. 비즈니스 선언에는 반드시 `config.endpoint`가 있어야 합니다(MUST). 엔드포인트는 반드시 authority가 비어 있지 않은 절대 HTTPS 브라우저 엔드포인트여야 하며(MUST), userinfo, 쿼리, 프래그먼트, 공백, 백슬래시, 끝 슬래시가 없어야 합니다.

퍼머링크로 네이티브 앱을 열 수도 있습니다. 네이티브 앱은 엔드포인트의 `https` URL을 처리하는 링크 핸들러로 등록할 수 있습니다(iOS의 Universal Links, Android의 App Links). 앱이 설치되어 있으면 앱이 퍼머링크를 가로채 네이티브 UI에서 처리하고, 설치되어 있지 않으면 웹 브라우저가 내비게이션을 처리합니다.

```json
{
  "ucp": {
    "version": "draft",
    "capabilities": {
      "dev.ucp.shopping.permalink": [
        {
          "version": "draft",
          "spec": "https://ucp.dev/draft/specification/permalink",
          "schema": "https://ucp.dev/draft/schemas/shopping/permalink.json",
          "config": {
            "endpoint": "https://merchant.example/buy"
          }
        }
      ]
    }
  }
}
```

## 스키마

### 설정

Business browser endpoint configuration for shopping permalinks.

| 이름     | 타입   | 필수 여부    | 설명                                                                                                                                                                                                                         |
| -------- | ------ | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| endpoint | string | **Required** | Absolute HTTPS browser endpoint with a non-empty authority and without userinfo, query, fragment, whitespace, backslashes, or trailing slash. Optional compact item path and query parameters are appended to this endpoint. |

## URL 형태와 라우트 바인딩

예를 들어 디스커버리로 찾은 엔드포인트가 다음과 같다고 하겠습니다.

```text
https://merchant.example/buy
```

플랫폼이나 비즈니스는 여기에 간결한 상품 경로와 쿼리 문자열을 선택적으로 덧붙입니다.

```text
https://merchant.example/buy/{items}?{query}
```

각 요소의 정의는 다음과 같습니다.

```text
items     = item_pair *( "," item_pair )
item_pair = item_id_token ":" quantity
quantity  = positive base-10 integer without leading zeros
```

간결한 상품 경로가 없어도, 비즈니스가 지원한다면 엔드포인트에 UCP 필드 경로 쿼리 매개변수를 붙일 수 있습니다(MAY). 예를 들면 다음과 같습니다.

```text
https://merchant.example/buy?context/postal_code=94105&continue_to=/
```

이 URL은 `postal_code`를 `94105`로 설정해 쇼핑 컨텍스트를 초기화하고, 이어서 `/`로 이동하도록 요청합니다.

브라우저 라우트 바인딩은 다음 위치에 따로 문서화되어 있습니다.

```text
https://ucp.dev/draft/services/shopping/permalink.openapi.json
```

이 문서에서 정의하는 라우트는 다음과 같습니다.

```text
GET /
GET /{items}
```

OpenAPI 문서는 라우트 형태와 리디렉션 응답을 기술합니다. 상품 토큰 인코딩, 쿼리 분류, UCP 필드 경로 의미론, 리디렉션 의미론, 안전 규칙은 이 명세가 규범적 기준입니다.

## 간결한 상품 경로

간결한 상품 경로는 쇼핑 `line_items`를 초기화합니다.

간결한 상품 식별자가 구매 가능한 상품 ID라면, 다음 URL은

```text
/buy/sku_123:2,sku_456:1
```

아래 초기화 데이터와 같습니다.

```json
{
  "line_items": [
    { "item": { "id": "sku_123" }, "quantity": 2 },
    { "item": { "id": "sku_456" }, "quantity": 1 }
  ]
}
```

### 구매 가능 상품 해석

간결한 상품 경로는 비즈니스가 요청된 쇼핑 흐름에 넣어야 할 상품을 가리킵니다. 플랫폼과 비즈니스는 가능하면 카탈로그 작업이 반환한 구매 가능 변형(variant) ID나 판매 단위 ID를 사용해야 합니다(SHOULD).

비즈니스는 다른 상품 식별자(제품 ID 포함)라도 요청 컨텍스트에서 구매 가능한 판매 단위 하나로 모호함 없이 해석할 수 있다면 받아들일 수 있습니다(MAY). 식별자가 구매 가능한 단위로 하나도 해석되지 않거나 여러 개로 해석되면, 비즈니스는 요청을 쇼핑 오류로 처리하거나 구매자를 선택 페이지 또는 문제 해결 페이지로 안내해야 합니다(SHOULD).

비즈니스가 퍼머링크로 UCP 데이터를 만들 때 `line_items[].item.id`는 반드시 해석된 구매 가능 판매 단위를 가리켜야 합니다(MUST).

### 상품 ID 토큰

`item_id_token`은 간결한 상품 식별자를 경로에 안전하게, 원래대로 되돌릴 수 있게 인코딩한 값입니다. 별도의 두 번째 식별자가 아니라 URL에 담긴 식별자를 그대로 보존한 것입니다.

상품 식별자가 다음 패턴과 일치하면

```text
^[A-Za-z0-9._-]+$
```

플랫폼이나 비즈니스는 그대로 쓸 수 있습니다(MAY).

```text
/buy/sku_123:2
```

일치하지 않으면 상품 식별자는 반드시 다음과 같이 인코딩해야 합니다(MUST).

```text
"~" + base64url_no_padding(utf8(item_identifier))
```

상품 식별자를 간결한 경로에 넣으려고 퍼센트 인코딩해서는 안 됩니다(MUST NOT). 상품 식별자에 원시 토큰 문법에 없는 문자(`/`, `:`, `,`, `%`, `~`, 공백, 비ASCII 문자 등)가 하나라도 있으면 반드시 `~` 인코딩 형식을 써야 합니다(MUST). 이렇게 하면 `%2F`처럼 퍼센트 인코딩된 경로 구분자를 피할 수 있습니다. 이런 구분자는 HTTP 서버, 프록시, 라우터가 해석 전에 거부하거나 디코딩하거나 정규화할 수 있습니다.

예를 들어

```text
gid://shopify/ProductVariant/70881412
```

는 다음과 같이 바뀝니다.

```text
/buy/~Z2lkOi8vc2hvcGlmeS9Qcm9kdWN0VmFyaWFudC83MDg4MTQxMg:1
```

비즈니스는 반드시 맨 앞의 `~`를 인코딩 표식으로 보고 나머지를 패딩 없는 base64url로 디코딩해야 합니다(MUST). 나머지 부분이 정규 형식의 패딩 없는 base64url이 아닌 토큰은 반드시 거부해야 하며(MUST), 디코딩한 값이 유효한 UTF-8이 아니거나 제어 문자를 포함해도 반드시 거부해야 합니다(MUST). 식별자의 원시 형식과 `~` 인코딩 형식은 반드시 같은 디코딩 결과로 해석되어야 합니다(MUST). URI처럼 보이더라도 식별자는 식별자일 뿐입니다. 비즈니스는 자신의 카탈로그에서 이를 해석하며, 자신이 소유한 형식이라면 파싱할 수도 있습니다(MAY)(예: `gid://…` 안의 숫자 키).

## 쿼리 처리

쿼리 문자열에는 초기화할 쇼핑 상태, 목적지 선호, UCP가 아닌 쿼리 매개변수가 담깁니다. 플랫폼은 UCP 필드 이름을 맨 앞의 `/`를 뺀 JSON Pointer 경로로 구성합니다(예: `buyer/email`, `line_items/0/quantity`).

비즈니스는 디코딩한 쿼리 매개변수 이름을 하나씩 다음 순서로 분류합니다.

1. 이름이 `continue_to`이면 목적지 선호로 처리합니다.
1. 그 밖의 이름은 맨 앞에 `/`가 정확히 하나 오도록 정규 JSON Pointer로 정규화합니다. 예를 들어 `buyer/email`과 `/buyer/email`은 모두 `/buyer/email`이 되고, `buyer`는 `/buyer`가 됩니다.
1. 정규화한 포인터를 JSON Pointer 토큰으로 파싱합니다. 첫 번째 토큰이 UCP 루트 후보입니다.
1. 루트 후보가 비즈니스가 지원하는 UCP 장바구니 또는 체크아웃 스키마(활성화된 프로필과 확장 포함)의 최상위 필드나 이 기능이 정의한 필드와 일치하면, 해당 매개변수를 UCP 필드 경로 쿼리 매개변수로 처리합니다.
1. 일치하지 않으면 UCP가 아닌 쿼리 매개변수로 처리합니다.

원시 키가 달라도 같은 포인터로 정규화될 수 있고(예: `buyer/email`과 `/buyer/email`), `continue_to`가 여러 번 나올 수도 있습니다. 비즈니스는 정규화한 뒤 이런 충돌을 반드시 감지해야 하며(MUST), 쿼리 매개변수 순서로 충돌을 해소해서는 안 됩니다(MUST NOT). 이런 경우는 모호한 요청으로 처리합니다([병합 규칙](#merge-rules)과 [오류 처리](#error-handling) 참고).

UCP 필드 경로 쿼리 매개변수라면 비즈니스는 나머지 포인터 토큰을 적용 대상 스키마에 맞춰 해석합니다. 퍼머링크 입력은 신뢰할 수 없습니다. 비즈니스는 다른 UCP 요청과 마찬가지로 자신의 장바구니 또는 체크아웃 스키마가 입력으로 받는 필드에만 값을 적용합니다. 따라서 서버 소유 필드나 응답 전용 필드(합계, 표시 통화, 상태, 식별자, 주문)는 URL로 절대 설정할 수 없습니다. 전체 필드 경로를 인식할 수 있고 적용할 수 있으면 비즈니스는 이를 초기화된 쇼핑 상태에 적용합니다. 인식하거나 적용할 수 없으면 기본적으로 소비하고 제거해야 합니다(SHOULD). 비즈니스는 어트리뷰션 보존처럼 정해진 목적이 있을 때, 특정 필드 경로를 명시적으로 골라 필요하면 다시 작성한 경우에만 목적지 URL에 노출할 수 있습니다(MAY). 민감한 값은 전달해서는 안 됩니다(MUST NOT).

### 목적지 선호(`continue_to`)

`continue_to`는 쇼핑 상태 데이터가 아니라 목적지 선호입니다. 비즈니스가 퍼머링크 입력을 적용한 뒤 구매자가 이어서 이동할 동일 출처(same-origin) 경로를 요청합니다.

```text
/buy/sku_123:1?continue_to=/collections/spring
```

`continue_to`는 외부에서 들어온 신뢰할 수 없는 입력이므로, 비즈니스는 반드시 서버 측에서 다음 순서대로 검증해야 합니다(MUST).

1. 값을 한 번 퍼센트 디코딩합니다(RFC 3986 퍼센트 디코딩 1회).
1. 디코딩한 값이 단일 `/`로 시작하지 않거나(`//`는 불가), URL 스킴, 백슬래시, 공백, 제어 문자(TAB, CR, LF 포함)를 포함하면 거부합니다.
1. 디코딩한 값을 퍼머링크 엔드포인트의 출처 기준 상대 참조로 해석하고, RFC 3986 점 세그먼트 제거를 적용합니다.
1. 그 결과로 나온 절대 URL이 엔드포인트와 동일 출처가 아니면 거부합니다.
1. 다시 인코딩한 정규 경로를 `Location`에 내보냅니다. 비즈니스는 원시 값을 그대로 반영해서는 안 되며(MUST NOT), 내보내는 `Location`에는 CR, LF, 기타 제어 문자가 들어가서는 안 됩니다(MUST NOT).

이 검증을 통과하지 못한 값이면 비즈니스는 반드시 기본 목적지로 대체해야 합니다(MUST).

### UCP 필드

UCP 필드 경로는 초기화된 쇼핑 상태를 가리킵니다. 이 상태는 해당 흐름에 쓰이는 비즈니스의 장바구니 또는 체크아웃 스키마(활성화된 프로필과 확장 포함)를 기준으로 해석합니다. 그렇다고 퍼머링크 규약이 비즈니스에게 해당 UCP 리소스를 실제로 생성하라고 요구하지는 않습니다. 어떤 URL 필드에 UCP 의미가 있는지는 비즈니스의 스키마가 정합니다.

```text
context/postal_code=94105
context/payment/0/handler=com.example.wallet
buyer/email=alice%40example.com
attribution/utm_source=social
```

위 필드는 다음과 같이 매핑됩니다.

```json
{
  "context": {
    "postal_code": "94105",
    "payment": [{ "handler": "com.example.wallet" }]
  },
  "buyer": { "email": "alice@example.com" },
  "attribution": { "utm_source": "social" }
}
```

인식되는 필드는 기본 UCP, 프로필, 확장 어디에서든 올 수 있습니다. 플랫폼과 비즈니스는 비즈니스가 해당 의미론을 지원한다는 사실을 알지 못한다면 프로필이나 확장 고유의 필드에 의존하지 않는 것이 좋습니다(SHOULD NOT).

### 슬래시 없는 쿼리 매개변수

슬래시가 없는 쿼리 매개변수도 UCP 필드 경로 쿼리 매개변수가 될 수 있습니다. 예를 들어 `buyer`는 `/buyer`로 정규화됩니다. `buyer`가 비즈니스가 인식하는 UCP 루트라면 UCP 형식의 매개변수이고, 그렇지 않으면 UCP가 아닌 쿼리 매개변수입니다.

### 확장 정의 필드

확장과 프로필은 UCP 필드 경로를 추가로 정의할 수 있습니다. 퍼머링크는 이런 필드에도 같은 JSON Pointer 경로 문법을 쓰므로, 확장 전용 별칭 문법이 따로 필요 없습니다.

예를 들어 의도한 쇼핑 흐름에서 `dev.ucp.shopping.discount`가 활성화되어 있으면, 다음 URL은

```text
/buy/sku_123:1?discounts/codes/0=SAVE10&discounts/codes/1=WELCOME
```

아래 내용을 기록합니다.

```json
{ "discounts": { "codes": ["SAVE10", "WELCOME"] } }
```

비즈니스가 해당 흐름에서 할인 확장을 지원하지 않으면 이 필드 경로들은 할인 입력으로 적용되지 않습니다. 이때 비즈니스는 위의 쿼리 처리 규칙에 따라 처리합니다.

프로필 정의 필드도 같은 방식을 씁니다.

```text
/buy/sku_123:1?line_items/0/selling_plan=plan_monthly
```

기본 UCP에는 `line_items[].selling_plan`이 정의되어 있지 않습니다. 플랫폼과 비즈니스는 이 필드를 정의하는 프로필이나 확장을 비즈니스가 지원한다는 사실을 알 때만 이 필드를 사용해야 합니다(SHOULD).

필드 경로는 장바구니 또는 체크아웃의 최상위 필드와 그 안의 위치 기반 항목을 가리킵니다. 값으로는 변형 ID나 목적지 ID처럼 구매자가 이미 알고 있는 식별자를 참조할 수 있습니다(MAY). 하지만 라인 아이템 ID나 풀필먼트 그룹 ID처럼 비즈니스가 구매 가능 상태를 구성하면서 비로소 생성하는 식별자를 키로 쓰는 항목은 가리킬 수 없습니다. 링크를 만드는 시점에는 그런 식별자가 아직 없기 때문입니다.

### UCP가 아닌 쿼리 매개변수

UCP는 UCP가 아닌 쿼리 매개변수를 초기화된 쇼핑 데이터에 넣지 않습니다.

```text
utm_source=social&ref=creator_42&gclid=...
```

어트리뷰션을 UCP 객체에 넣어야 한다면 명시적인 UCP 필드를 사용합니다.

```text
attribution/ref=creator_42&attribution/utm_source=social
```

UCP가 아닌 쿼리 매개변수를 보존하고 걸러내는 규칙은 [리디렉션 처리](#redirect-resolution)를 참고하세요.

## 병합 규칙

라인 아이템 ID와 수량은 간결한 상품 경로가 우선합니다. 같은 라인 아이템을 대상으로 하는 쿼리 필드가 이 값을 바꿔서는 안 됩니다(MUST NOT).

예를 들어

```text
/buy/sku_123:2?line_items/0/quantity=5
```

는 간결한 경로에 따라 `line_items/0/quantity`를 `2`로 해석합니다.

프로필이나 확장 의미론에서 해당 필드를 정의하고 경로 필드를 덮어쓰지 않는다면, 쿼리 필드로 경로가 만든 라인 아이템을 확장할 수 있습니다(MAY). 예를 들어 확장에서 정의한 `line_items/0/selling_plan` 필드는 간결한 경로의 상품 ID나 수량을 바꾸지 않고 판매 플랜 상태를 추가할 수 있습니다.

플랫폼은 같은 UCP 포인터로 정규화되는 쿼리 매개변수를 여러 개 만들어서는 안 됩니다(MUST NOT). 값이 여러 개 필요하면 플랫폼은 반드시 인덱스를 붙인 배열 경로처럼 서로 다른 필드 경로를 사용해야 합니다(MUST).

```text
/buy/sku_123:1?discounts/codes/0=SAVE10&discounts/codes/1=WELCOME
```

비즈니스는 중복 쓰기를 해소하려고 쿼리 매개변수 순서에 기대서는 안 됩니다(MUST NOT). 두 쿼리 매개변수가 같은 UCP 포인터로 정규화되면, 해당 필드에 순서와 무관한 정책을 문서화해 두지 않은 한 비즈니스는 반드시 요청을 모호한 것으로 보고 [오류 처리](#error-handling)에 따라 처리해야 합니다(MUST).

## 리디렉션 처리

비즈니스는 처리한 퍼머링크 요청에 반드시 `303 See Other`와 구매자용 목적지를 가리키는 `Location` 헤더로 응답해야 합니다(MUST). 응답에는 반드시 `Cache-Control: no-store`가 있어야 하며(MUST), 퍼머링크 URL이 제3자에게 유출되지 않게 하는 리퍼러 정책(예: `Referrer-Policy: no-referrer`)도 반드시 있어야 합니다(MUST).

```http
HTTP/1.1 303 See Other
Location: https://merchant.example/checkout/session/chk_123
Cache-Control: no-store
Referrer-Policy: no-referrer
```

`Location`은 비즈니스가 정하며, 다른 출처일 수도 있습니다(MAY)(예: 외부 사이트나 호스팅형 체크아웃). 비즈니스는 신뢰할 수 없는 입력이 목적지에 영향을 주기 전에 반드시 검증하고 정규화해야 하며(MUST), 그런 입력에서 나온 다른 출처의 목적지는 명시적 정책이나 허용 목록으로 통제해야 합니다(SHOULD).

비즈니스는 퍼머링크 처리에 `301`이나 `308` 같은 영구 리디렉션을 써서는 안 됩니다(MUST NOT).

입력이 충분하면 비즈니스는 구매 가능한 장바구니, 체크아웃, 또는 그에 준하는 서버 측 상태를 구성해야 합니다(SHOULD). 구매 가능 상태를 구성할 수 있고 `continue_to`가 없으면 기본 구매 목적지로 리디렉션해야 합니다(SHOULD). 구매 가능 상태를 구성할 수 없을 때는, 유효한 `continue_to`가 있으면 그 목적지로 구매자를 안내해야 하고(SHOULD), 없으면 스토어프런트 루트, 장바구니, 구매자용 문제 해결 페이지 같은 안전한 대체 목적지로 안내해야 합니다(SHOULD).

그 상태를 어떻게 구성할지는 비즈니스가 정합니다. 새 장바구니를 만들 수도, 별도의 체크아웃을 준비할 수도, 퍼머링크 상품을 기존 장바구니에 병합할 수도 있으며(MAY), 구매자에게 선택하게 하거나 자동화된 정책을 적용할 수도 있습니다(MAY). 다만 인증된 구매자의 기존 장바구니에 퍼머링크 상태를 병합하기 전에는 화면에 보여 준 퍼머링크 상태에 대해 반드시 구매자의 명시적인 행동을 받아야 합니다(MUST). 페이지 로드나 리디렉션 시점에 자동으로 병합하거나, 미리 채워진 제출로 병합하는 것은 이 요구 사항을 충족하지 않습니다. 퍼머링크는 구매자의 의도를 표현하고, 응답은 그 결과 상태를 전달합니다. 비즈니스는 구매 가능 상태를 구성하기 전에 인증, 자격 확인이나 연령 확인, 로그인 같은 추가 단계를 요구할 수도 있으며(MAY), 그에 맞게 구매자를 안내합니다.

비즈니스는 자신이 이해하는 쿼리 매개변수를 적용해야 합니다(SHOULD). 적용된 매개변수는 서버 측 상태나 목적지 선택에 영향을 줄 수 있으며, 리디렉션 URL에 나타날 필요는 없습니다. UCP 필드 경로 쿼리 매개변수는 기본적으로 소비하고 제거해야 합니다(SHOULD).

클라이언트 측 분석, 라우팅, 호환성, 프런트엔드 동작이 계속 작동하도록 비즈니스는 기본적으로 UCP가 아닌 쿼리 매개변수를 리디렉션 목적지에 보존해야 합니다(SHOULD). 보존하기에 안전하지 않거나, 목적지와 호환되지 않거나, 비즈니스 정책상 허용되지 않으면 UCP가 아닌 쿼리 매개변수를 삭제하거나 다시 작성할 수 있습니다(MAY). 골라 둔 UCP 필드 경로를 목적지 URL에 노출할 때는 알맞은 비UCP 쿼리 매개변수로 다시 작성해야 합니다(SHOULD). 어트리뷰션 필드는 보통 `attribution/` 접두사를 떼는 방식으로 다시 작성합니다.

예를 들어 다음 퍼머링크 요청은

```text
/buy/sku_123:1?continue_to=/collections/spring&buyer/email=alice%40example.com&buyer/unknown=foo&attribution/utm_source=email&utm_medium=sms&color=black&access_token=secret
```

다음과 같이 처리될 수 있습니다.

```http
HTTP/1.1 303 See Other
Location: https://merchant.example/collections/spring?utm_source=email&utm_medium=sms&color=black
Cache-Control: no-store
Referrer-Policy: no-referrer
```

이 예시에서 각 매개변수는 다음과 같이 처리됩니다.

- 소비됨: `sku_123:1`은 서버 측 쇼핑 상태를 초기화하고, `continue_to`는 목적지 경로를 정하며, `buyer/email`은 서버 측 상태에 적용됩니다.
- 삭제됨: `buyer`가 UCP 루트이므로 `buyer/unknown`은 UCP 필드 경로 쿼리 매개변수이지만, 전체 필드 경로를 인식하거나 적용할 수 없습니다.
- 다시 작성됨: `attribution/utm_source`는 클라이언트 측 분석용으로 골라 `utm_source`로 내보냅니다.
- 그대로 전달됨: `utm_medium`과 `color`는 UCP가 아닌 쿼리 매개변수로, 클라이언트 측 분석과 스토어프런트 동작을 위해 보존됩니다.
- 삭제됨: `access_token`은 UCP가 아닌 쿼리 매개변수이지만 민감한 값입니다.

비즈니스는 민감한 값([개인정보 보호](#privacy) 참고)을 리디렉션 목적지에 반영해서는 안 됩니다(MUST NOT).

## 오류 처리

처리된 쇼핑 오류는 URL은 파싱할 수 있지만 입력을 모두 적용할 수는 없을 때 생깁니다. 상품 구매 불가, 수량 상한 적용, 유효하지 않은 할인 코드, 사용할 수 없는 결제 핸들러, 무시된 구매자 필드, 지원하지 않는 확장 필드, 체크아웃 자격 미달 등이 여기에 해당합니다.

처리된 쇼핑 오류라면 비즈니스는 `303` 리디렉션 모델을 그대로 유지하고, 목적지에서 구매자에게 문제 해결 방법을 보여 줘야 합니다(SHOULD). 유효한 `continue_to` 값이 있으면 안전한 입력을 적용한 뒤 그 목적지로 안내해야 하고(SHOULD), 없으면 실패 유형에 맞는 구매, 쇼핑, 문제 해결 목적지로 안내해야 합니다(SHOULD).

잘못되었거나 안전하지 않은 요청에는 형식이 잘못된 상품 토큰, 유효하지 않은 수량, 유효하지 않은 base64url 토큰, 안전하지 않은 `continue_to` 값, 파싱할 수 없는 쿼리, 제어 문자가 해당합니다. 비즈니스는 가능하면 잘못된 브라우저 요청을 안전한 구매자용 대체 목적지로 리디렉션해야 하지만(SHOULD), 요청을 안전하게 해석할 수 없으면 `4xx`를 반환할 수 있습니다(MAY).

오류는 서버 측 세션 상태나 목적지 페이지 상태로 표시하는 것을 우선해야 합니다(SHOULD).

## 예시

### 단일 상품 기본 구매

```text
https://merchant.example/buy/sku_123:1
```

초기화 데이터:

```json
{
  "line_items": [
    {
      "item": { "id": "sku_123" },
      "quantity": 1
    }
  ]
}
```

처리 결과 예:

```http
HTTP/1.1 303 See Other
Location: https://checkout.merchant.example/session/chk_123
Cache-Control: no-store
Referrer-Policy: no-referrer
```

### 할인과 이어가기가 포함된 캠페인 링크

```text
https://merchant.example/buy/sku_123:1,sku_456:2?continue_to=/collections/spring&discounts/codes/0=SPRING10&attribution/utm_source=email
```

초기화 데이터:

```json
{
  "line_items": [
    {
      "item": { "id": "sku_123" },
      "quantity": 1
    },
    {
      "item": { "id": "sku_456" },
      "quantity": 2
    }
  ],
  "discounts": {
    "codes": ["SPRING10"]
  },
  "attribution": {
    "utm_source": "email"
  }
}
```

`continue_to`는 비즈니스가 퍼머링크 입력을 적용한 뒤 이어질 브라우저 내비게이션을 요청합니다. 비즈니스는 `attribution/utm_source`를 소비하고, 클라이언트 측 분석을 위해 리디렉션 목적지에서 이를 `utm_source`로 다시 작성할 수 있습니다.

처리 결과 예:

```http
HTTP/1.1 303 See Other
Location: https://merchant.example/collections/spring?utm_source=email
Cache-Control: no-store
Referrer-Policy: no-referrer
```

### 구매자 맞춤 구매 링크

```text
https://merchant.example/buy/sku_kit:3,~Z2lkOi8vc2hvcGlmeS9Qcm9kdWN0VmFyaWFudC83MDg4MTQxMg:2?discounts/codes/0=VIP20&discounts/codes/1=WELCOME&buyer/email=alice%40foo.com&buyer/phone_number=123-456-7890&context/address_country=US&context/postal_code=94105&context/language=en-US&attribution/ref=creator_42&attribution/utm_source=social&context/payment/0/handler=com.example.wallet
```

초기화 데이터:

```json
{
  "line_items": [
    {
      "item": { "id": "sku_kit" },
      "quantity": 3
    },
    {
      "item": {
        "id": "gid://shopify/ProductVariant/70881412"
      },
      "quantity": 2
    }
  ],
  "discounts": {
    "codes": ["VIP20", "WELCOME"]
  },
  "buyer": {
    "email": "alice@foo.com",
    "phone_number": "123-456-7890"
  },
  "context": {
    "address_country": "US",
    "postal_code": "94105",
    "language": "en-US",
    "payment": [{ "handler": "com.example.wallet" }]
  },
  "attribution": {
    "ref": "creator_42",
    "utm_source": "social"
  }
}
```

`context/payment`는 선호 힌트입니다. 비즈니스는 이 값으로 핸들러(와 유형)를 미리 선택하거나 우선순위를 높여야 하며(SHOULD), 사용할 수 없거나 자격이 없는 값은 무시할 수 있습니다(MAY).

처리 결과 예:

```http
HTTP/1.1 303 See Other
Location: https://checkout.merchant.example/session/chk_123
Cache-Control: no-store
Referrer-Policy: no-referrer
```

### 목적지를 미리 선택한 픽업

```text
https://merchant.example/buy/sku_123:1?fulfillment/methods/0/type=pickup&fulfillment/methods/0/selected_destination_id=loc_1375
```

초기화 데이터:

```json
{
  "line_items": [{ "item": { "id": "sku_123" }, "quantity": 1 }],
  "fulfillment": {
    "methods": [{ "type": "pickup", "selected_destination_id": "loc_1375" }]
  }
}
```

해당 흐름에서 풀필먼트 확장이 활성화되어 있으면, 퍼머링크로 풀필먼트 방식과 구매자가 이미 알고 있는 목적지(예: 매장 찾기에서 고른 매장)를 미리 선택할 수 있습니다. 다른 확장과 같은 필드 경로 방식을 씁니다. `methods/0`이 장바구니 수준 풀필먼트를 위치로 가리키는 이유는, 비즈니스가 링크를 해석하기 전까지는 장바구니도 없고, 서버가 생성한 ID를 키로 쓰는 라인 아이템별·그룹별 풀필먼트도 없기 때문입니다.

처리 결과 예:

```http
HTTP/1.1 303 See Other
Location: https://checkout.merchant.example/session/chk_123
Cache-Control: no-store
Referrer-Policy: no-referrer
```

## 보안 고려 사항

퍼머링크 엔드포인트는 인증 없이 받는 브라우저 GET입니다. URL의 모든 구성 요소는 외부에서 들어온 신뢰할 수 없는 값이며, 브라우저, 프리페처, 링크 미리보기 봇, 보안 스캐너는 구매자의 의도와 상관없이 퍼머링크 URL을 엽니다.

비즈니스는 반드시 모든 입력을 신뢰할 수 없는 값으로 보고 검증해야 합니다(MUST). 여기에는 간결한 경로와 `~` 인코딩 상품 토큰([상품 ID 토큰](#item-id-token) 참고), `continue_to` ([목적지 선호](#destination-preference-continue_to) 참고), 필드 경로 쿼리 매개변수 ([쿼리 처리](#query-processing) 참고)가 포함됩니다. 이 규칙들은 오픈 리디렉션, 응답 헤더 주입, 서버 소유 필드의 대량 할당(mass-assignment)을 막고, 디코딩된 토큰과 값에 섞인 제어 문자로 인한 로그 주입이나 파서 혼동도 막습니다.

퍼머링크 URL에는 그 안에 담긴 입력이 그대로 드러납니다. 링크를 건 페이지의 `Referer`로 URL 자체가 유출될 수 있으며, `303` 응답의 `Cache-Control`과 리퍼러 정책([리디렉션 처리](#redirect-resolution) 참고)은 URL이 캐시되거나 목적지로 전달되지 않게 막습니다.

### 개인정보 보호

퍼머링크 URL은 브라우저 기록, 로그, 분석, 리퍼러, 스크린샷, 링크 미리보기, 복사된 메시지로 유출됩니다. 플랫폼과 비즈니스는 해당 채널에 적합한 데이터만 넣어야 합니다(SHOULD).

널리 뿌리는 캠페인 링크에는 상품 ID, 수량, 대략적인 컨텍스트, 할인 코드 같은 간결한 확장 필드, `continue_to`, `context/payment` 선호, UCP가 아닌 캠페인 쿼리 매개변수를 우선 사용해야 합니다(SHOULD).

구매자 맞춤 링크에는 채널과 활성 스키마/프로필 의미론에 맞는다면 다음과 같이 최소한의 구매자 정보를 미리 채워 넣을 수 있습니다(MAY).

```text
buyer/email=alice%40example.com
buyer/first_name=Alice
```

플랫폼과 비즈니스는 널리 공유되는 링크에 개인 식별 정보(PII)를 넣지 않아야 합니다(SHOULD). 비즈니스는 원래 구매자 값이 빠진 URL로 리디렉션하고, 가능한 범위에서 로그의 PII를 가리고, 구매자가 미리 채워진 값을 고칠 수 있게 해야 합니다(SHOULD).

퍼머링크에는 결제 자격 증명, 결제 수단 토큰, 고객 액세스 토큰, 세션 쿠키, AP2 위임(Mandate), API 키, 베어러 토큰, 일회용 비밀 값이 들어가서는 안 됩니다(MUST NOT).

## URL 크기 예산

플랫폼과 비즈니스는 URL 인코딩까지 마쳐 완전히 직렬화한 절대 URL의 크기를 UTF-8 옥텟 단위로 산정해야 합니다(SHOULD). 브라우저, 메신저, 스캐너, CDN, 오리진 서버마다 실제 제한이 다르므로, 널리 배포하는 퍼머링크는 의도적으로 작게 만들어야 합니다(SHOULD).

권장 URL 크기 예산:

| 채널                              | 권장 최대 크기 |
| --------------------------------- | -------------- |
| QR 코드, 인쇄물, 짧은 소셜 게시물 | 512옥텟        |
| SMS 및 널리 공유되는 캠페인 링크  | 1,024옥텟      |
| 이메일 및 인증된 인계             | 2,048옥텟      |

플랫폼과 비즈니스는 이 예산 안에서 링크를 만들어야 하며(SHOULD), QR 예산은 의도한 인쇄 크기, 거리, 대비, 스캐너 환경을 테스트한 뒤에만 넘어야 합니다(SHOULD).

링크가 알맞은 예산을 넘을 것 같으면 [개인정보 보호](#privacy)에서 설명한 간결한 하위 집합으로 줄입니다. 비공개 상태나 고도로 개인화된 상태는 서버 측에서 소비해야 하며(SHOULD), 리디렉션 목적지로 넘겨서는 안 됩니다(MUST NOT).

비즈니스는 2,048옥텟 이상의 절대 퍼머링크 URL을 받아들여야 합니다(SHOULD). URL 길이, 간결한 상품 개수, 쿼리 매개변수 개수, 포인터 깊이, 디코딩된 값 길이에는 문서화된 제한을 둘 수 있습니다(MAY). 이 제한을 넘는 요청은 구매자용 문제 해결 페이지로 리디렉션하거나 알맞은 `4xx` 응답으로 거부할 수 있습니다(MAY).
