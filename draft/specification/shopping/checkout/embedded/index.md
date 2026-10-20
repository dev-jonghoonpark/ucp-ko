# 체크아웃 기능 - EP 바인딩

## 소개

임베디드 체크아웃 프로토콜(Embedded Checkout Protocol, ECP)은 UCP 임베디드 프로토콜(Embedded Protocol, EP) 전송 바인딩을 체크아웃용으로 구현한 것입니다. ECP를 사용하면 **호스트**가 **비즈니스**의 체크아웃 인터페이스를 임베드하고, 구매자가 체크아웃과 상호작용할 때 이벤트를 받으며, 주소·결제 수단 선택 같은 주요 사용자 동작을 위임받을 수 있습니다. ECP는 (REST처럼) 전송 바인딩입니다. 즉 **어떤** 데이터가 있는지가 아니라 **어떻게** 통신하는지를 정의합니다.

### W3C Payment Request와의 개념적 대응

ECP는 \*\*[W3C Payment Request API](https://www.w3.org/TR/payment-request/)\*\*에서 착안해, 그 멘탈 모델을 임베디드 체크아웃 시나리오에 맞게 조정했습니다. Payment Request에 익숙한 개발자라면 비슷한 패턴을 알아볼 수 있지만, 실행 모델은 다릅니다.

**W3C Payment Request:** 브라우저가 제어합니다. 비즈니스가 `show()`를 호출하면 브라우저가 네이티브 결제 시트를 렌더링합니다. 이벤트는 결제 핸들러에서 비즈니스로 흐릅니다.

**임베디드 체크아웃:** 비즈니스가 제어합니다. 호스트는 비즈니스의 체크아웃 UI를 iframe/webview에 임베드합니다. 이벤트는 양방향으로 흐르며, 호스트는 선택적으로 위임을 받아 특정 상호작용을 네이티브로 처리할 수 있습니다.

| 개념               | W3C Payment Request             | 임베디드 체크아웃                                                  |
| ------------------ | ------------------------------- | ------------------------------------------------------------------ |
| **초기화**         | `new PaymentRequest()`          | `continue_url`로 임베디드 컨텍스트 로드                            |
| **UI 준비**        | `show()`가 Promise 반환         | `ec.start` 알림                                                    |
| **결제 수단 변경** | `paymentmethodchange` 이벤트    | `ec.payment.change` 알림                                           |
| **주소 변경**      | `shippingaddresschange` 이벤트  | `ec.fulfillment.change` 및 `ec.fulfillment.address_change_request` |
| **결제 제출**      | 사용자 수락 → `PaymentResponse` | 위임된 `ec.payment.credential_request`                             |
| **완료**           | `response.complete()`           | `ec.complete` 알림                                                 |
| **에러/메시지**    | Promise 거부                    | `ec.messages.change` 알림                                          |

**핵심 차이:** W3C Payment Request에서는 브라우저가 결제 흐름을 조율합니다. 임베디드 체크아웃에서는 비즈니스가 임베디드 컨텍스트 안에서 흐름을 조율하며, 네이티브 경험을 위해 특정 UI(결제 수단 선택, 주소 선택기)를 선택적으로 호스트에 위임합니다.

## 용어 및 행위자

### 커머스 역할

- **비즈니스:** 상품/서비스와 체크아웃 경험을 제공하는 판매 주체.
- **구매자:** 구매를 하는 최종 사용자.

### 기술 구성 요소

- **호스트(Host):** 체크아웃을 임베드하는 애플리케이션(예: AI 에이전트 앱, 슈퍼 앱, 브라우저). **결제 핸들러**와 사용자 인증을 담당합니다.
- **임베디드 체크아웃(Embedded Checkout):** iframe이나 webview에 렌더링되는 비즈니스의 체크아웃 인터페이스. 체크아웃 흐름과 주문 생성을 담당합니다.
- **결제 핸들러(Payment Handler):** 사용자 인증(생체 인식/PIN)과 자격 증명 발급을 수행하는 보안 구성 요소.

## 요구 사항

### 디스커버리

ECP 지원 여부는 두 단계로 알립니다. 서비스 수준 디스커버리는 지원 능력을 선언하고, 체크아웃 응답은 실제 사용 가능 여부와 세션별로 허용된 설정을 확정합니다.

#### 서비스 수준 디스커버리

비즈니스는 `/.well-known/ucp` 프로필에 `embedded` 전송 방식을 넣어 임베디드 체크아웃 프로토콜 지원을 선언합니다.

**서비스 디스커버리 예시:**

```json
{
    "services": {
        "dev.ucp.shopping": [
            {
                "version": "draft",
                "transport": "rest",
                "schema": "https://ucp.dev/draft/services/shopping/rest.openapi.json",
                "endpoint": "https://merchant.example.com/ucp/v1"
            },
            {
                "version": "draft",
                "transport": "mcp",
                "schema": "https://ucp.dev/draft/services/shopping/mcp.openrpc.json",
                "endpoint": "https://merchant.example.com/ucp/mcp"
            },
            {
                "version": "draft",
                "transport": "embedded",
                "schema": "https://ucp.dev/draft/services/shopping/embedded.openrpc.json"
            }
        ]
    }
}
```

서비스 정의에 `embedded`가 없으면 비즈니스는 `continue_url`을 이용한 리디렉션 기반 체크아웃 이어가기만 지원합니다.

#### 체크아웃별 설정

서비스 수준 디스커버리는 비즈니스가 ECP를 지원한다고 선언할 뿐, 모든 체크아웃 세션에서 ECP가 활성화된다고 보장하지는 않습니다. 비즈니스는 특정 세션의 ECP 사용 가능 여부와 허용된 위임을 나타내기 위해 반드시 체크아웃 응답에 `config.delegate`가 포함된 임베디드 서비스 바인딩을 넣어야 합니다(**MUST**).

**체크아웃 응답 예시:**

```json
{
    "id": "checkout_abc123",
    "status": "open",
    "continue_url": "https://merchant.example.com/checkout/abc123",
    "ucp": {
        "version": "draft",
        "services": {
            "dev.ucp.shopping": [
                {
                    "version": "draft",
                    "transport": "embedded",
                    "config": {
                        "delegate": ["payment.credential", "fulfillment.address_change", "window.open"]
                    }
                }
            ]
        },
        "capabilities": {...},
        "payment_handlers": {...}
    }
}
```

`config.delegate` 배열은 이 체크아웃 세션에서 비즈니스가 수락한 위임, 즉 호스트가 `ec_delegate`로 요청한 것과 비즈니스가 허용하는 것의 교집합을 확정합니다. 이 값은 다음에 따라 달라질 수 있습니다.

- **장바구니 내용**: 일부 상품은 비즈니스가 직접 처리하는 결제 흐름이 필요할 수 있습니다
- **에이전트 권한**: 인증된 에이전트는 더 많은 위임을 받을 수 있습니다
- **비즈니스 정책**: 위험 규칙, 지역 제한 등

`config.delegate`가 포함된 임베디드 서비스 바인딩이 있으면 다음을 뜻합니다.

- 이 체크아웃은 `continue_url`로 ECP를 사용할 수 있습니다
- `config.delegate`는 비즈니스가 수락한 위임을 확정합니다
- 이는 `ec.ready` 핸드셰이크의 `delegate` 필드와 대응합니다

체크아웃 응답에 임베디드 서비스 바인딩이 없으면 (서비스 수준 디스커버리에서 임베디드 지원을 선언했더라도) 그 체크아웃은 `continue_url`을 이용한 리디렉션 기반 이어가기만 지원합니다.

### 임베디드 체크아웃 URL 로드

임베디드 서비스 바인딩이 포함된 체크아웃 응답을 받은 호스트는 임베디드 컨텍스트에서 `continue_url`을 로드해 ECP 세션을 시작할 수 있습니다(**MAY**).

임베디드 컨텍스트를 로드하기 전에 호스트는 다음을 해야 합니다(**SHOULD**).

1. `config.delegate`에서 사용 가능한 위임을 확인합니다
1. 호스트가 지원하려는 위임의 핸들러를 준비합니다
1. 비즈니스가 요구하면 선택적으로 인증 자격 증명을 준비합니다

세션을 시작하려면 호스트는 반드시 `continue_url`에 `ec_` 접두사가 붙은 ECP 쿼리 파라미터를 덧붙여야 합니다(**MUST**).

다양한 임베딩 환경에서 최대한 호환되도록 모든 ECP 파라미터는 HTTP 헤더가 아닌 URL 쿼리 문자열로 전달합니다. 네임스페이스 오염을 피하고 비즈니스 고유 쿼리 파라미터와 ECP 파라미터를 명확히 구분하기 위해 파라미터에는 `ec_` 접두사를 붙입니다.

- `ec_version` (string, **REQUIRED**): 이 세션의 UCP 버전 (형식: `YYYY-MM-DD`). 체크아웃 응답의 버전과 일치해야 합니다. 버전은 세션 초기화 시 협상되며 반드시 ECP 세션이 유지되는 동안 변하지 않아야 합니다(**MUST**). 핸드셰이크 이후에는 어느 쪽도 버전을 바꿀 수 없습니다.
- `ec_auth` (string, **OPTIONAL**): 비즈니스가 정의한 형식의 인증 토큰
- `ec_delegate` (string, **OPTIONAL**): 호스트가 처리하려는 위임을 쉼표로 구분한 목록. 임베디드 서비스 바인딩의 `config.delegate`의 부분집합이어야 합니다(**SHOULD**).
- `ec_color_scheme` (string, **OPTIONAL**): 체크아웃 UI의 색 구성표 선호값. 유효한 값: `light`, `dark`. 제공되지 않으면 임베디드 체크아웃은 시스템 설정을 따릅니다.

#### 인증

**토큰 형식:**

- `auth` 파라미터 형식은 전적으로 비즈니스가 정의합니다
- 일반적인 형식으로는 JWT, OAuth 토큰, API 키, 세션 식별자 등이 있습니다
- 비즈니스는 반드시 기대하는 토큰 형식과 검증 절차를 문서화해야 합니다(**MUST**)

**예시(참고용 - JWT 기반):**

```text
// One possible implementation using JWT
{
  "alg": "HS256",
  "typ": "JWT"
}
{
  "iat": 1234567890,
  "exp": 1234568190,
  "jti": "unique-id",
  // ... business-specific claims ...
}
```

비즈니스는 반드시 자체 보안 요구 사항에 따라 인증을 검증해야 합니다(**MUST**).

**인증을 포함한 초기화 예시:**

```text
https://example.com/checkout/abc123?ec_version=2026-01-11&ec_auth=eyJ...
```

참고: 모든 쿼리 파라미터 값은 RFC 3986에 따라 올바르게 URL 인코딩해야 합니다.

#### 위임

선택 파라미터 `ec_delegate`는 구매자가 임베디드 체크아웃 UI에서 처리하는 대신 호스트가 네이티브로 처리하려는 오퍼레이션을 선언합니다. 각 위임 식별자는 일관된 패턴 `ec.{delegation}_request`를 따르는 대응 `_request` 메시지에 매핑됩니다.

**위임 식별자 예시:**

| `ec_delegate` 값             | 대응 메시지                             |
| ---------------------------- | --------------------------------------- |
| `payment.instruments_change` | `ec.payment.instruments_change_request` |
| `payment.credential`         | `ec.payment.credential_request`         |
| `fulfillment.address_change` | `ec.fulfillment.address_change_request` |
| `window.open`                | `ec.window.open_request`                |

확장은 자체 위임 식별자를 정의합니다. 사용 가능한 옵션은 각 확장의 명세를 참고하세요.

```text
?ec_version=2026-01-11&ec_delegate=payment.instruments_change,payment.credential,fulfillment.address_change,window.open
```

#### 색 구성표

선택 파라미터 `ec_color_scheme`을 사용하면 호스트가 임베디드 체크아웃이 쓸 색 구성표를 지정할 수 있어, 호스트 애플리케이션과 체크아웃 UI의 시각적 일관성을 맞출 수 있습니다.

**유효한 값:**

| 값      | 설명                                            |
| ------- | ----------------------------------------------- |
| `light` | 밝은 색 구성표 사용(밝은 배경, 어두운 텍스트)   |
| `dark`  | 어두운 색 구성표 사용(어두운 배경, 밝은 텍스트) |

**기본 동작:**

`ec_color_scheme`이 제공되지 않으면 임베디드 체크아웃은 [`prefers-color-scheme`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme) 미디어 쿼리나 [`Sec-CH-Prefers-Color-Scheme`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Sec-CH-Prefers-Color-Scheme) HTTP 클라이언트 힌트로 구매자의 시스템 설정을 사용할 수 있으며, 변경을 감지해 그에 맞게 갱신해야 합니다(**SHOULD**).

**구현 참고 사항:**

- 기본적으로 임베디드 체크아웃은 구매자의 시스템 색 구성표 설정을 따르고, 변경을 감지해 그에 맞게 갱신해야 합니다(**SHOULD**)
- `ec_color_scheme`이 명시적으로 제공되면 반드시 시스템 설정보다 우선하고, 로드 즉시 적용되며, 세션이 유지되는 동안 계속 적용되어야 합니다(**MUST**).
- 비즈니스는 지원하지 않는 값을 무시할 수 있습니다(**MAY**)

**예시:**

```text
https://example.com/checkout/abc123?ec_version=2026-01-11&ec_color_scheme=dark
```

#### 위임 협상

위임은 비즈니스 정책에서 최종 수락까지 범위가 점점 좁아지는 연쇄를 따릅니다.

```text
config.delegate ⊇ ec_delegate ⊇ ec.ready delegate
```

1. **비즈니스 허용**(체크아웃 응답의 `config.delegate`): 이 체크아웃 세션에서 비즈니스가 허용하는 위임 집합
1. **호스트 요청**(`ec_delegate` URL 파라미터): 호스트가 네이티브로 처리하려는 부분집합
1. **ECP 수락**(`ec.ready`의 `delegate`): 임베디드 체크아웃이 실제로 위임할 최종 부분집합

각 단계는 이전 단계의 부분집합입니다.

- 호스트는 `config.delegate`에 있는 위임만 요청해야 합니다(**SHOULD**)
- 비즈니스는 `config.delegate`에 없는 위임을 수락하지 않아야 하며(**SHOULD NOT**), 반드시 수락한 위임을 `ec.ready`에서 확정해야 합니다(**MUST**)

### 위임 계약

위임은 호스트와 임베디드 체크아웃 사이에 구속력 있는 계약을 만듭니다. 다만 임베디드 체크아웃은 비즈니스 정책에 따라 인증되었거나 승인된 호스트에만 위임을 허용할 수 있습니다(**MAY**).

#### 위임 수락

임베디드 체크아웃은 다음을 근거로 어떤 위임을 받아들일지 결정합니다.

- 인증 상태(`ec_auth` 파라미터)
- 호스트 권한 수준
- 비즈니스 정책

임베디드 체크아웃은 반드시 `ec.ready` 요청의 `delegate` 필드로 수락한 위임을 표시해야 합니다(**MUST**)([`ec.ready`](#ecready) 참고). 요청된 위임을 수락하지 않으면 임베디드 체크아웃은 반드시 해당 동작을 자체 UI로 처리해야 합니다(**MUST**).

#### 구속 요건

**위임이 수락되면** 양측은 구속력 있는 계약을 맺습니다.

**임베디드 체크아웃의 책임:**

1. 해당 동작이 트리거되면 반드시 적절한 `{action}_request` 메시지를 보내야 합니다(**MUST**)
1. 진행하기 전에 반드시 호스트의 응답을 기다려야 합니다(**MUST**)
1. 위임된 동작에 대해 자체 UI를 보여서는 안 됩니다(**MUST NOT**)

**호스트의 책임:**

1. 받은 모든 `{action}_request` 메시지에 반드시 응답해야 합니다(**MUST**)
1. 사용자가 취소하면 반드시 적절한 에러로 응답해야 합니다(**MUST**)
1. 위임을 처리하는 동안 로딩/처리 중 상태를 보여 주어야 합니다(**SHOULD**)

#### 3.3.3 위임 흐름

1. **요청**: 임베디드 체크아웃이 현재 상태를 담아 `ec.{domain}.{action}_request` 메시지를 보냅니다(`id` 포함)
1. **네이티브 UI**: 호스트가 위임된 동작에 대한 네이티브 UI를 표시합니다
1. **응답**: 호스트가 일치하는 `id`와 `result` 또는 `error`를 담은 JSON-RPC 응답을 돌려보냅니다
1. **갱신**: 임베디드 체크아웃이 상태를 갱신하고, 이어서 변경 알림을 보낼 수 있습니다

도메인별 위임 세부 사항은 [결제 확장](#payment-extension), [풀필먼트 확장](#fulfillment-extension), [윈도 확장](#window-extension)을 참고하세요.

### 탐색 제약

체크아웃이 임베디드 모드로 렌더링될 때 구현은 체크아웃에 집중된 경험을 유지하기 위해 체크아웃 밖으로의 탐색을 막아야 합니다(**SHOULD**). 임베디드 뷰는 범용 브라우저가 아니라 체크아웃 흐름을 제공하기 위한 것입니다.

**탐색 요건:**

- 임베디드 체크아웃은 체크아웃 흐름 밖의 URL로 이동하려는 시도를 차단하거나 가로채야 합니다(**SHOULD**)
- 임베디드 체크아웃은 체크아웃을 벗어나게 하는 UI 요소(예: 외부 링크, 내비게이션 바)를 제거하거나 비활성화해야 합니다(**SHOULD**)
- 임베드하는 쪽은 컨테이너 수준에서 추가 탐색 제한을 구현할 수 있습니다(**MAY**)

**허용되는 예외:** 체크아웃 완료에 필요한 경우 다음 탐색 시나리오는 허용할 수 있습니다(**MAY**).

- 결제 제공자 리디렉션: 외부 사이트 결제 흐름
- 3D Secure 인증: 카드 인증 프레임과 리디렉션
- 은행 승인: 오픈 뱅킹 또는 유사한 승인 흐름
- 신원 확인: 필요한 경우 KYC/AML 규정 준수 확인

이러한 예외 흐름은 완료되면 사용자를 체크아웃 흐름으로 되돌려야 합니다(**SHOULD**).

## 전송 및 메시징

ECP는 EP 공통 전송 계층을 사용합니다. 메시지 형식, 메시지 유형, 응답 처리 규칙은 [임베디드 프로토콜 — 전송 및 메시징](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#transport-messaging)을 참고하세요.

모든 응답의 `ucp.version`은 반드시 세션 초기화 시 협상되고 호스트가 `ec.ready` 응답에서 확정한 `ec_version`을 그대로 반환해야 합니다(**MUST**). 버전은 세션에 묶여 있으며, ECP 세션이 유지되는 동안 바뀌어서는 안 됩니다(**MUST NOT**).

### 통신 채널

ECP는 EP 공통 통신 채널 모델을 따릅니다. 일반적인 패턴은 [임베디드 프로토콜 — 통신 채널](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#communication-channels)을 참고하세요.

네이티브 호스트의 경우 체크아웃 전용 전역 객체는 다음과 같습니다.

- `window.EmbeddedCheckoutProtocolConsumer`(권장)
- `window.webkit.messageHandlers.EmbeddedCheckoutProtocolConsumer`
- `window.EmbeddedCheckoutProtocol`(호스트 → 임베디드 체크아웃)

## 메시지 API 참조

### 메시지 범주

#### 핵심 메시지

핵심 메시지는 ECP 명세에서 정의하며, 모든 구현은 반드시 이를 지원해야 합니다(**MUST**). 모든 메시지는 임베디드 체크아웃에서 호스트로 전송됩니다.

| 범주           | 목적                                               | 패턴 | 핵심 메시지                                                                                              |
| -------------- | -------------------------------------------------- | ---- | -------------------------------------------------------------------------------------------------------- |
| **핸드셰이크** | 호스트와 임베디드 체크아웃 사이의 연결 수립        | 요청 | `ec.ready`                                                                                               |
| **인증**       | 임베디드 체크아웃과 호스트 사이의 인증 데이터 교환 | 요청 | `ec.auth`                                                                                                |
| **수명 주기**  | 체크아웃 상태 전환 알림                            | 알림 | `ec.start`, `ec.complete`                                                                                |
| **상태 변경**  | 체크아웃 필드 변경 알림                            | 알림 | `ec.line_items.change`, `ec.buyer.change`, `ec.payment.change`, `ec.messages.change`, `ec.totals.change` |
| **세션 에러**  | 체크아웃 리소스와 무관한 세션 수준 에러 신호       | 알림 | `ec.error`                                                                                               |

#### 확장 메시지

확장은 추가 메시지를 정의해 임베디드 프로토콜을 확장할 수 있습니다(**MAY**). 확장 메시지는 반드시 다음 명명 규칙을 따라야 합니다(**MUST**).

- **알림**: `ec.{domain}.change` — 상태 변경 알림(`id` 없음)
- **위임 요청**: `ec.{domain}.{action}_request` — 응답이 필요함(`id` 있음)

여기서:

- `{domain}`은 디스커버리의 도메인 식별자와 일치합니다(예: `payment`, `fulfillment`, `window`)
- `{action}`은 위임되는 구체적인 동작을 나타냅니다(예: `instruments_change`, `address_change`)
- `_request` 접미사는 응답이 필요한 위임 지점임을 나타냅니다

### 핸드셰이크 메시지

#### `ec.ready`

렌더링되면 임베디드 체크아웃은 반드시 `ec.ready` 메시지로 부모 컨텍스트에 준비 완료를 알려야 합니다(**MUST**). 이 메시지는 호스트와 임베디드 체크아웃 사이의 보안 통신 채널을 초기화하고, 어떤 위임이 수락되었는지와 추가 인증 교환이 필요한지를 전달하며, UCP 체크아웃 동작으로 전달되지 않은 표시 전용 체크아웃 상태를 호스트가 추가로 제공할 수 있게 합니다.

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 요청
- **페이로드:**
  - `delegate` (array of strings, **REQUIRED**): 임베디드 체크아웃이 수락한 위임 식별자 목록. 반드시 `ec_delegate`(호스트가 요청한 것)와 체크아웃 응답의 `config.delegate`(비즈니스가 허용하는 것) 모두의 부분집합이어야 합니다(**MUST**). 빈 배열은 수락된 위임이 없다는 뜻입니다.
  - `auth` (object, **OPTIONAL**): 추가 고려 사항 때문에 `ec_auth` URL 파라미터로 충분하지 않거나 적용할 수 없을 때, 비즈니스는 이 객체 안에 `type` 문자열을 지정해 최초 핸드셰이크 중에 인증을 요청할 수 있습니다. 이 `type` 문자열 값은 [`ec.auth`](#ecauth)에 포함되는 페이로드 내용과 대응합니다.

**메시지 예시(수락된 위임 없음):**

```json
{
    "jsonrpc": "2.0",
    "id": "ready_1",
    "method": "ec.ready",
    "params": {
        "delegate": [],
        "auth": {
            "type": "oauth"
        }
    }
}
```

**메시지 예시(위임 수락됨):**

```json
{
    "jsonrpc": "2.0",
    "id": "ready_1",
    "method": "ec.ready",
    "params": {
        "delegate": ["payment.credential", "fulfillment.address_change", "window.open"],
        "auth": {
            "type": "oauth"
        }
    }
}
```

`ec.ready` 메시지는 요청이므로, 호스트는 핸드셰이크를 완료하기 위해 반드시 응답해야 합니다(**MUST**).

- **방향:** 호스트 → 임베디드 체크아웃
- **유형:** 응답
- **결과 페이로드:**
  - `ucp` (object, **REQUIRED**): UCP 프로토콜 메타데이터. `version`은 협상된 `ec_version`을 확정하며, `status`는 반드시 `"success"`여야 합니다(**MUST**). 이 버전은 세션에 묶여 있습니다. 호스트가 여기서 프로토콜 버전을 명시적으로 확정하며, 세션이 유지되는 동안 바뀌어서는 안 됩니다(**MUST NOT**).
  - `upgrade` (object, **OPTIONAL**): 임베디드 체크아웃이 호스트와 통신하는 데 쓰는 통신 채널을 어떻게 갱신할지 기술하는 객체. 이 필드가 있으면 호스트는 `credential`을 포함해서는 안 됩니다(**MUST NOT**). 채널이 다시 수립되므로 여기서 보낸 자격 증명은 폐기됩니다.
  - `credential` (string, **OPTIONAL**): 요청된 인증 데이터로, OAuth 토큰, JWT, API 키 등의 형태일 수 있습니다. 요청에 `auth`가 있으면 반드시 설정해야 합니다(**MUST**). `upgrade`가 있으면 설정해서는 안 됩니다(**MUST NOT**).
  - `checkout` (object, **OPTIONAL**): UCP 체크아웃 동작으로 전달되지 않은 표시 전용 추가 체크아웃 상태. 체크아웃 UI를 채우는 데 쓰이며, 특정 조건에서 다음 필드를 채우는 용도로만 사용할 수 있습니다.
    - `payment.instruments`: 호스트와 임베디드 체크아웃이 모두 `payment.instruments_change` 위임을 수락한 경우 덮어쓸 수 있습니다.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "ready_1",
    "result": {
        "ucp": { "version": "draft", "status": "success" },
        "credential": "fake_identity_linking_oauth_token"
    }
}
```

호스트는 `upgrade` 필드로 응답해 호스트와 임베디드 체크아웃 사이의 통신 채널을 갱신할 수 있습니다(**MAY**). 현재 이 객체는 `port` 필드만 지원합니다. 이 필드는 반드시 `MessagePort` 객체여야 하며(**MUST**), 반드시 임베디드 체크아웃 컨텍스트로 전달되어야 합니다(**MUST**) (예: 호스트의 `iframe.contentWindow.postMessage()` 호출에서 `{transfer: [port2]}` 사용).

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "ready_1",
    "result": {
        "ucp": { "version": "draft", "status": "success" },
        "upgrade": {
            "port": "[Transferable MessagePort]"
        }
    }
}
```

호스트가 `upgrade` 객체로 응답하면 임베디드 체크아웃은 반드시 메시지의 다른 정보를 모두 폐기하고, 갱신된 통신 채널로 새 `ec.ready` 메시지를 보낸 뒤 새 응답을 기다려야 합니다(**MUST**). 이후의 모든 메시지는 반드시 갱신된 통신 채널로만 보내야 합니다(**MUST**).

호스트는 `checkout` 객체로 응답할 수도 있으며(**MAY**), 이 객체는 호스트와 비즈니스 사이의 위임 계약에 따라 체크아웃 UI를 채우는 데 쓰입니다.

**메시지 예시: 표시 정보를 포함해 결제 수단 제공:**

```json
{
    "jsonrpc": "2.0",
    "id": "ready_1",
    "result": {
        "ucp": { "version": "draft", "status": "success" },
        "checkout": {
            "payment": {
                // The instrument structure is defined by the handler's instrument schema
                "instruments": [
                    {
                        "id": "payment_instrument_123",
                        "handler_id": "merchant_psp_handler_123",
                        "type": "card",
                        "selected": true,
                        "display": {
                            "brand": "visa",
                            "expiry_month": 12,
                            "expiry_year": 2026,
                            "last_digits": "1111",
                            "description": "Visa •••• 1111",
                            "card_art": "https://host.com/cards/visa-gold.png"
                        }
                    }
                ]
            }
        }
    }
}
```

**에러 응답 예시:**

호스트가 핸드셰이크를 완료할 수 없으면(예: 출처 검증 실패, 프로토콜 상태 위반) 반드시 `error_response` 결과로 응답해야 합니다(**MUST**).

```json
{
    "jsonrpc": "2.0",
    "id": "ready_1",
    "result": {
        "ucp": { "version": "draft", "status": "error" },
        "messages": [
            {
                "type": "error",
                "code": "security_error",
                "content": "Host origin validation failed.",
                "severity": "unrecoverable"
            }
        ]
    }
}
```

호스트가 에러로 응답하면 세션은 진행될 수 없습니다. 호스트는 반드시 임베디드 컨텍스트를 해제해야 하며(**MUST**), `continue_url`이 있으면 구매자를 그곳으로 리디렉션할 수 있습니다(**MAY**). 임베디드 체크아웃은 핸드셰이크 에러를 받은 뒤 더 이상 메시지를 보내서는 안 됩니다(**MUST NOT**).

### 인증

#### `ec.auth`

`ec.auth`는 EP 공통 인증 패턴을 구현합니다. 요청/응답 규약, 예시, 에러 에스컬레이션 흐름은 [임베디드 프로토콜 — 인증](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#authentication)을 참고하세요.

- **메서드:** `ec.auth`
- **방향:** 임베디드 체크아웃 → 호스트(요청), 호스트 → 임베디드 체크아웃(응답)

에러 에스컬레이션이 필요하면 임베디드 체크아웃은 반드시 [세션 에러 패턴](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#session-error)에 따라 `ec.error` 알림을 보내야 합니다(**MUST**).

### 수명 주기 메시지

수명 주기 알림은 EP 공통 패턴을 따릅니다. [임베디드 프로토콜 — 수명 주기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#lifecycle)를 참고하세요. 모든 수명 주기 알림은 전체 `checkout` 객체를 페이로드로 담습니다.

#### `ec.start`

체크아웃이 화면에 표시되어 상호작용할 준비가 되었음을 알립니다. `ec.ready` 핸드셰이크가 성공한 뒤 전송됩니다.

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 알림
- **페이로드:**
  - `checkout` (object, **REQUIRED**): 체크아웃의 현재 전체 상태.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ec.start",
    "params": {
        "checkout": {
            "id": "checkout_123",
            "status": "incomplete",
            "messages": [
                {
                    "type": "error",
                    "code": "missing",
                    "path": "$.buyer.shipping_address",
                    "content": "Shipping address is required",
                    "severity": "recoverable"
                }
            ],
            "totals": [ ... ],
            "line_items": [ ... ],
            "buyer": { ... },
            "payment": { ... }
            // ... other checkout fields
        }
    }
}
```

#### `ec.complete`

체크아웃이 성공적으로 완료되었음을 나타냅니다.

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 알림
- **페이로드:**
  - `checkout` (object, **REQUIRED**): 결과 `order` 객체를 포함한 체크아웃의 최종 상태.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ec.complete",
    "params": {
        "checkout": {
            "id": "checkout_123",
            // ... other checkout fields
            "order": {
                "id": "ord_99887766",
                "permalink_url": "https://merchant.com/orders/ord_99887766"
            }
        }
    }
}
```

### 상태 변경 메시지

상태 변경 알림은 EP 공통 패턴을 따릅니다. [임베디드 프로토콜 — 상태 변경](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#state-change)을 참고하세요. 모든 상태 변경 알림은 임베디드 체크아웃에서 호스트로 전송되며 전체 `checkout` 객체를 페이로드로 담습니다.

#### `ec.line_items.change`

라인 아이템이 수정되었습니다(수량 변경, 상품 추가/제거).

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 알림
- **페이로드:**
  - `checkout` (object, **REQUIRED**): 체크아웃의 현재 전체 상태.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ec.line_items.change",
    "params": {
        "checkout": {
            "id": "checkout_123",
            // The entire checkout object is provided, including the updated line items and totals
            "totals": [ ... ],
            "line_items": [ ... ]
            // ...
        }
    }
}
```

#### `ec.buyer.change`

구매자 정보가 갱신되었습니다(이메일, 전화번호, 주소).

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 알림
- **페이로드:**
  - `checkout` (object, **REQUIRED**): 체크아웃의 현재 전체 상태.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ec.buyer.change",
    "params": {
        "checkout": {
            "id": "checkout_123",
            // The entire checkout object is provided, including the updated buyer information
            "buyer": { ... }
            // ...
        }
    }
}
```

#### `ec.messages.change`

체크아웃 메시지가 갱신되었습니다. 메시지에는 체크아웃 상태에 관한 에러, 경고, 정보성 안내가 포함됩니다.

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 알림
- **페이로드:**
  - `checkout` (object, **REQUIRED**): 체크아웃의 현재 전체 상태.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ec.messages.change",
    "params": {
        "checkout": {
            "id": "checkout_123",
            "messages": [
                {
                    "type": "error",
                    "code": "invalid_address",
                    "path": "$.buyer.shipping_address",
                    "content": "We cannot ship to this address",
                    "severity": "recoverable"
                },
                {
                    "type": "info",
                    "code": "free_shipping",
                    "content": "Free shipping applied!"
                }
            ]
            // ...
        }
    }
}
```

#### `ec.totals.change`

체크아웃 합계가 갱신되었습니다. 이 메시지는 세금, 수수료, 할인, 풀필먼트 비용을 포함한 모든 합계 항목 변경을 다루며, 이 중 상당수는 별도의 도메인별 변경 메시지가 없습니다. 비즈니스는 `checkout.totals`가 어떤 이유로든 바뀔 때마다 반드시 이 메시지를 보내야 합니다(**MUST**).

변경이 도메인별 메시지(예: `ec.line_items.change`, `ec.buyer.change`, `ec.payment.change`)도 트리거하면, 비즈니스는 반드시 도메인별 메시지를 먼저 보내고 이어서 `ec.totals.change`를 보내야 합니다(**MUST**).

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 알림
- **페이로드:**
  - `checkout` (object, **REQUIRED**): 체크아웃의 현재 전체 상태.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ec.totals.change",
    "params": {
        "checkout": {
            "id": "checkout_123",
            // The entire checkout object is provided, including the updated totals
            "totals": [
                {
                    "type": "subtotal",
                    "display_text": "Subtotal",
                    "amount": 4000
                },
                {
                    "type": "fulfillment",
                    "display_text": "Shipping",
                    "amount": 599
                },
                {
                    "type": "tax",
                    "display_text": "Tax",
                    "amount": 382
                },
                {
                    "type": "total",
                    "display_text": "Total",
                    "amount": 4981
                }
            ]
            // ...
        }
    }
}
```

#### `ec.payment.change`

결제 상태가 갱신되었습니다. 전체 문서는 [결제 확장](#payment-extension)을 참고하세요.

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 알림
- **페이로드:**
  - `checkout` (object, **REQUIRED**): 체크아웃의 현재 전체 상태.

### 세션 에러 메시지

#### `ec.error`

`ec.error`는 EP 공통 세션 에러 패턴을 구현합니다. 페이로드 명세와 호스트 처리 요건은 [임베디드 프로토콜 — 세션 에러](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#session-error)를 참고하세요.

## 결제 확장

결제 확장은 호스트가 상태 변경 알림과 위임 요청을 사용해 사용자 에스컬레이션 흐름을 조율하는 방법을 정의합니다. 체크아웃 URL에 `ec_delegate=payment.instruments_change,payment.credential`이 포함되면 호스트가 결제 수단 선택과 토큰 획득을 제어하게 되며, 그 결과로 임베디드 체크아웃에 상태 갱신을 제공합니다.

### 결제 개요 및 호스트 선택

결제 위임에서는 호스트와 임베디드 체크아웃을 조율하는 두 가지 패턴을 쓸 수 있습니다.

**옵션 A: 호스트가 임베디드 체크아웃에 맡김** 호스트는 URL에 결제 위임을 포함하지 **않습니다**. 임베디드 체크아웃이 자체 UI와 결제 흐름으로 결제 수단 선택과 처리를 담당합니다. 이것이 위임하지 않는 표준 흐름입니다.

**옵션 B: 호스트가 제어** 호스트는 체크아웃 URL에 `ec_delegate=payment.instruments_change,payment.credential`을 포함해, 결제 UI와 토큰 획득을 호스트에 위임하라고 임베디드 체크아웃에 알립니다. 위임된 경우:

- **임베디드 체크아웃의 책임**:
  - 현재 결제 수단을 변경 진입점(예: "결제 수단 변경" 버튼)과 함께 표시합니다
  - 결제를 제출하기 전에 `ec.payment.credential_request` 메시지에 대한 응답을 기다립니다
- **호스트의 책임**:
  - `ec.payment.instruments_change_request`에 응답해 구매자가 다른 결제 수단을 선택할 수 있는 네이티브 UI를 렌더링하고, 선택된 수단으로 응답합니다
  - `ec.payment.credential_request`에 응답해 선택된 결제 수단의 결제 토큰을 획득하고, 그 토큰을 임베디드 체크아웃에 보냅니다

### 결제 메시지 API 참조

#### `ec.payment.change`

새 결제 수단이 선택되는 등 체크아웃 UI의 결제 영역에서 무언가 바뀌었음을 호스트에 알립니다.

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 알림
- **페이로드:**
  - `checkout`: 체크아웃의 최신 상태

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ec.payment.change",
    "params": {
        "checkout": {
            "id": "checkout_123",
            // The entire checkout object is provided, including the updated payment details
            "payment": {
                "instruments": [
                    {
                        "id": "payment_instrument_123",
                        "selected": true
                        // ... additional instrument fields
                    }
                ]
            }
            // ...
        }
    }
}
```

#### `ec.payment.instruments_change_request`

호스트에게 결제 수단 선택 UI를 표시하도록 요청합니다.

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 요청
- **페이로드:**
  - `checkout`: 체크아웃의 최신 상태

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "payment_instruments_change_request_1",
    "method": "ec.payment.instruments_change_request",
    "params": {
        "checkout": {
            "id": "checkout_123",
            // The entire checkout object is provided, including the current payment details
            "payment": { ... }
            // ...
        }
    }
}
```

호스트는 반드시 에러 또는 새로 선택된 결제 수단으로 응답해야 합니다(**MUST**). 성공 응답에서 호스트는 반드시 `payment.instruments` 필드만 갱신한 `checkout` 객체의 부분 업데이트로 응답해야 합니다(**MUST**). 임베디드 체크아웃은 반드시 이 업데이트를 PUT 방식 변경으로 취급해, 새 데이터를 기존 상태와 병합하려 하지 말고 제공된 필드의 기존 상태를 완전히 교체해야 합니다(**MUST**).

- **방향:** 호스트 → 임베디드 체크아웃
- **유형:** 응답
- **페이로드:**
  - `ucp`: `status: "success"`를 담은 UCP 프로토콜 메타데이터
  - `checkout`: 체크아웃 객체에 적용할 업데이트

**성공 응답 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "payment_instruments_change_request_1",
    "result": {
        "ucp": { "version": "draft", "status": "success" },
        "checkout": {
            "payment": {
                // The instrument structure is defined by the handler's instrument schema
                "instruments": [
                    {
                        "id": "payment_instrument_123",
                        "handler_id": "merchant_psp_handler_123",
                        "type": "card",
                        "selected": true,
                        "display": {
                            "brand": "visa",
                            "expiry_month": 12,
                            "expiry_year": 2026,
                            "last_digits": "1111",
                            "description": "Visa •••• 1111",
                            "card_art": "https://host.com/cards/visa-gold.png"
                        }
                        // No `credential` yet; it will be attached in the `ec.payment.credential_request` response
                    }
                ]
            }
        }
    }
}
```

**에러 응답 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "payment_instruments_change_request_1",
    "result": {
        "ucp": { "version": "draft", "status": "error" },
        "messages": [
            {
                "type": "error",
                "code": "abort_error",
                "content": "User closed the payment sheet without authorizing.",
                "severity": "recoverable"
            }
        ]
    }
}
```

#### `ec.payment.credential_request`

체크아웃 제출 중 선택된 결제 수단의 자격 증명을 요청합니다.

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 요청
- **페이로드:**
  - `checkout`: 체크아웃의 최신 상태

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "payment_credential_request_1",
    "method": "ec.payment.credential_request",
    "params": {
        "checkout": {
            "id": "checkout_123",
            // The entire checkout object is provided, including the current payment details
            "payment": {
                "instruments": [
                    {
                        "id": "payment_instrument_123",
                        "selected": true
                        // ... additional instrument fields
                    }
                ]
            }
            // ...
        }
    }
}
```

호스트는 반드시 에러 또는 선택된 결제 수단의 자격 증명으로 응답해야 합니다(**MUST**). 성공 응답에서 호스트는 반드시 `checkout` 객체의 부분 업데이트를 제공해, `selected: true`인 결제 수단에 새 `credentials` 필드를 넣어 갱신해야 합니다(**MUST**). 임베디드 체크아웃은 반드시 이 업데이트를 PUT 방식 변경으로 취급해, 새 데이터를 기존 상태와 병합하려 하지 말고 `payment.instruments`의 기존 상태를 완전히 교체해야 합니다(**MUST**).

- **방향:** 호스트 → 임베디드 체크아웃
- **유형:** 응답
- **페이로드:**
  - `ucp`: `status: "success"`를 담은 UCP 프로토콜 메타데이터
  - `checkout`: 체크아웃 객체에 적용할 업데이트

**성공 응답 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "payment_credential_request_1",
    "result": {
        "ucp": { "version": "draft", "status": "success" },
        "checkout": {
            "payment": {
                "instruments": [
                    {
                        "id": "payment_instrument_123",
                        "handler_id": "merchant_psp_handler_123",
                        "type": "card",
                        "selected": true,
                        "display": {
                            "brand": "visa",
                            "expiry_month": 12,
                            "expiry_year": 2026,
                            "last_digits": "1111",
                            "description": "Visa •••• 1111",
                            "card_art": "https://host.com/cards/visa-gold.png"
                        },
                        // The credential structure is defined by the handler's instrument schema
                        "credential": {
                            "type": "token",
                            "token": "tok_123"
                        }
                    }
                ]
            }
        }
    }
}
```

**에러 응답 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "payment_credential_request_1",
    "result": {
        "ucp": { "version": "draft", "status": "error" },
        "messages": [
            {
                "type": "error",
                "code": "abort_error",
                "content": "User closed the payment sheet without authorizing.",
                "severity": "recoverable"
            }
        ]
    }
}
```

**결제 토큰 위임 중 호스트의 책임:**

1. **확인:** 호스트는 신뢰할 수 있는 결제 UI(결제 시트 / 생체 인식 프롬프트)를 표시합니다. 호스트는 메시지만으로 토큰을 조용히 내주어서는 안 됩니다(**MUST NOT**).
1. **인증:** 호스트는 결제 핸들러로 사용자 승인을 받습니다.
1. **AP2 연동(선택):** `ucp.ap2_mandate`가 활성화되어 있으면 (**[AP2 확장](https://ap2-extension.org/)** 참고) 호스트는 신뢰할 수 있는 사용자 인터페이스로 이 시점에 `payment_mandate`를 생성합니다.

## 풀필먼트 확장

풀필먼트 확장은 호스트가 주소 선택을 위임받아 네이티브 주소 선택기 경험을 제공하는 방법을 정의합니다. 체크아웃 URL에 `ec_delegate=fulfillment.address_change`가 포함되면 호스트가 배송 주소 선택을 제어하게 되며, 그 결과로 임베디드 체크아웃에 주소 갱신을 제공합니다.

### 풀필먼트 개요 및 호스트 선택

풀필먼트 위임에서는 두 가지 패턴을 쓸 수 있습니다.

**옵션 A: 호스트가 임베디드 체크아웃에 맡김** 호스트는 URL에 풀필먼트 위임을 포함하지 **않습니다**. 임베디드 체크아웃이 자체 UI와 주소 입력 양식으로 주소 입력을 처리합니다. 이것이 위임하지 않는 표준 흐름입니다.

**옵션 B: 호스트가 제어** 호스트는 체크아웃 URL에 `ec_delegate=fulfillment.address_change`를 포함해, 주소 선택 UI를 호스트에 위임하라고 임베디드 체크아웃에 알립니다. 위임된 경우:

**임베디드 체크아웃의 책임**:

- 현재 배송 주소를 변경 진입점(예: "주소 변경" 버튼)과 함께 표시합니다
- 구매자가 주소 변경을 시작하면 `ec.fulfillment.address_change_request`를 보냅니다
- 호스트가 반환한 주소를 바탕으로 배송 옵션을 갱신합니다

**호스트의 책임**:

- `ec.fulfillment.address_change_request`에 응답해 구매자가 배송 주소를 선택하거나 입력할 수 있는 네이티브 UI를 렌더링합니다
- 선택된 주소를 UCP PostalAddress 형식으로 응답합니다

### 풀필먼트 메시지 API 참조

#### `ec.fulfillment.change`

체크아웃 UI에서 풀필먼트 세부 정보가 바뀌었음을 호스트에 알립니다.

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 알림
- **페이로드:**
  - `checkout`: 체크아웃의 최신 상태

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ec.fulfillment.change",
    "params": {
        "checkout": {
            "id": "checkout_123",
            // The entire checkout object is provided, including the updated fulfillment details
            "fulfillment": { ... }
            // ...
        }
    }
}
```

#### `ec.fulfillment.address_change_request`

배송 풀필먼트 방식에 대한 주소 선택 UI를 표시하도록 호스트에 요청합니다.

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 요청
- **페이로드:**
  - `checkout`: 체크아웃의 최신 상태

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "fulfillment_address_change_request_1",
    "method": "ec.fulfillment.address_change_request",
    "params": {
        "checkout": {
            "id": "checkout_123",
            // The entire checkout object is provided, including the current fulfillment details
            "fulfillment": {
                "methods": [
                    {
                        "id": "method_1",
                        "type": "shipping",
                        "line_item_ids": ["li_1"],
                        "selected_destination_id": "address_123",
                        "destinations": [
                            {
                                "id": "address_123",
                                "street_address": "456 Old Street"
                                // ...
                            }
                        ]
                        // ...
                    }
                ]
            }
            // ...
        }
    }
}
```

호스트는 반드시 에러 또는 새로 선택된 주소로 응답해야 합니다(**MUST**). 성공 응답에서 호스트는 반드시 갱신된 `fulfillment.methods` 객체로 응답해야 하며(**MUST**), 풀필먼트 방식의 `selected_destination_id`와 `destinations` 필드를 갱신하고 나머지는 기존 상태를 유지합니다. 임베디드 체크아웃은 반드시 이 업데이트를 PUT 방식 변경으로 취급해, 새 데이터를 기존 상태와 병합하려 하지 말고 `fulfillment.methods`의 기존 상태를 완전히 교체해야 합니다(**MUST**).

- **방향:** 호스트 → 임베디드 체크아웃
- **유형:** 응답
- **페이로드:**
  - `ucp`: `status: "success"`를 담은 UCP 프로토콜 메타데이터
  - `checkout`: 체크아웃 객체에 적용할 업데이트

**성공 응답 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "fulfillment_address_change_request_1",
    "result": {
        "ucp": { "version": "draft", "status": "success" },
        "checkout": {
            "fulfillment": {
                "methods": [
                    {
                        "id": "method_1",
                        "type": "shipping",
                        "line_item_ids": ["li_1"],
                        "selected_destination_id": "address_789",
                        "destinations": [
                            {
                                "id": "address_789",
                                "first_name": "John",
                                "last_name": "Doe",
                                "street_address": "123 New Street"
                            }
                        ]
                    }
                ]
            }
        }
    }
}
```

**에러 응답 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "fulfillment_address_change_request_1",
    "result": {
        "ucp": { "version": "draft", "status": "error" },
        "messages": [
            {
                "type": "error",
                "code": "abort_error",
                "content": "User cancelled address selection.",
                "severity": "recoverable"
            }
        ]
    }
}
```

### 주소 형식

주소 객체는 UCP [PostalAddress](/ucp-ko/draft/draft/specification/reference/#postal-address) 형식을 사용합니다.

### 우편 주소

| 이름             | 타입   | 필수 여부 | 설명                                                                                                                                                                                                                                      |
| ---------------- | ------ | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| extended_address | string | Optional  | An address extension such as an apartment number, C/O or alternative name.                                                                                                                                                                |
| street_address   | string | Optional  | The street address.                                                                                                                                                                                                                       |
| address_locality | string | Optional  | The locality in which the street address is, and which is in the region. For example, Mountain View.                                                                                                                                      |
| address_region   | string | Optional  | The region in which the locality is, and which is in the country. Required for applicable countries (i.e. state in US, province in CA). For example, California or another appropriate first-level Administrative division.               |
| address_country  | string | Optional  | The country. Recommended to be in 2-letter ISO 3166-1 alpha-2 format, for example "US". For backward compatibility, a 3-letter ISO 3166-1 alpha-3 country code such as "SGP" or a full country name such as "Singapore" can also be used. |
| postal_code      | string | Optional  | The postal code. For example, 94043.                                                                                                                                                                                                      |
| first_name       | string | Optional  | Optional. First name of the contact associated with the address.                                                                                                                                                                          |
| last_name        | string | Optional  | Optional. Last name of the contact associated with the address.                                                                                                                                                                           |
| phone_number     | string | Optional  | Optional. Phone number of the contact associated with the address.                                                                                                                                                                        |

## 윈도 확장

윈도 확장은 구매자가 비즈니스가 제시한 링크를 활성화했을 때 임베디드 체크아웃이 호스트에 알리는 방법을 정의합니다. 체크아웃 URL에 `ec_delegate=window.open`이 포함되면 호스트는 반드시 모든 `ec.window.open_request`를 처리하고 요청을 확인 응답해야 합니다(**MUST**).

이는 임베디드 체크아웃이 관련 없는 페이지로의 탐색을 막기 위해 무조건 적용하는 [탐색 제약](#navigation-constraints)과는 별개입니다.

### 윈도 개요 및 호스트 선택

윈도 위임에서는 두 가지 패턴을 쓸 수 있습니다.

**옵션 A: 호스트가 임베디드 체크아웃에 맡김** 호스트는 `ec_delegate`에 `window.open`을 포함하지 **않습니다**. 임베디드 체크아웃이 자체 인라인 UI로 링크 표시를 처리합니다. 이것이 위임하지 않는 표준 흐름입니다.

**옵션 B: 호스트가 제어** 호스트는 체크아웃 URL에 `ec_delegate=window.open`을 포함해, 구매자가 링크를 활성화하면 `ec.window.open_request`를 보내라고 임베디드 체크아웃에 알립니다. 위임된 경우:

**임베디드 체크아웃의 책임**:

- 구매자가 비즈니스가 제시한 링크를 활성화하면 반드시 `ec.window.open_request`를 보내야 합니다(**MUST**)

**호스트의 책임**:

- 요청된 URL이 `https` 스킴을 쓰는지 반드시 검증해야 합니다(**MUST**)
- 추가 호스트 보안 정책(예: 출처 검증)을 적용해야 합니다(**SHOULD**)
- 승인한 모든 요청에 대해 반드시 콘텐츠를 구매자에게 보여 주어야 합니다(**MUST**) (예: 모달, 새 탭 등)
- 요청을 처리했으면 JSON-RPC 성공 결과로, 호스트 정책 때문에 탐색을 막았으면 `window_open_rejected_error` 에러로 반드시 응답해야 합니다(**MUST**)
- 요청이 거부되었으면 구매자에게 알릴 수 있습니다(**MAY**)

`window.open` 위임을 수락하면 호스트는 구매자의 링크 상호작용을 처리할 책임을 집니다. 임베디드 체크아웃은 해당 링크에 대해 자체 UI를 표시해서는 안 됩니다(**MUST NOT**).

`ec.window.open_request` 페이로드에는 URL만 담깁니다. 더 풍부한 맥락(예: 링크 유형이나 레이블)이 필요한 호스트는 요청된 URL을 체크아웃 세션의 `checkout.links` 배열과 대조해 추가 메타데이터를 얻을 수 있습니다(**MAY**).

### 윈도 메시지 API 참조

#### `ec.window.open_request`

체크아웃 안에서 구매자가 활성화한 링크를 처리하도록 호스트에 요청합니다.

- **방향:** 임베디드 체크아웃 → 호스트
- **유형:** 요청
- **페이로드:**
  - `url` (string, uri, **REQUIRED**): 표시할 리소스의 URL.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "window_1",
    "method": "ec.window.open_request",
    "params": {
        "url": "https://merchant.com/privacy-policy"
    }
}
```

- **방향:** 호스트 → 임베디드 체크아웃
- **유형:** 응답
- **페이로드:**
  - `ucp`: `status: "success"`를 담은 UCP 프로토콜 메타데이터

**성공 응답 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "window_1",
    "result": {
        "ucp": { "version": "draft", "status": "success" }
    }
}
```

**에러 응답 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "window_1",
    "result": {
        "ucp": { "version": "draft", "status": "error" },
        "messages": [
            {
                "type": "error",
                "code": "window_open_rejected_error",
                "content": "Window open rejected by host.",
                "severity": "unrecoverable"
            }
        ]
    }
}
```

## 보안 및 에러 처리

### 에러 코드

공통 에러 코드는 [임베디드 프로토콜 — 에러 코드](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#error-codes)를 참고하세요. 임베디드 체크아웃은 위임 관련 시나리오를 위해 다음 코드를 추가로 정의합니다.

| 코드                         | 심각도          | 설명                                                                                                                                     |
| ---------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `not_allowed_error`          | `recoverable`   | 요청에 유효한 사용자 활성화(User Activation)가 없었습니다([원치 않는 결제 요청 방지](#prevention-of-unsolicited-payment-requests) 참고). |
| `window_open_rejected_error` | `unrecoverable` | 호스트 정책 때문에 탐색이 막혔습니다. 호스트는 구매자에게 요청이 거부되었음을 알릴 수 있습니다(**MAY**).                                 |

`not_allowed_error`에서 복구하려면 위임을 다시 시도하기 전에 새로운 [사용자 활성화](https://html.spec.whatwg.org/multipage/interaction.html#activation) 제스처가 필요합니다.

### 웹 기반 호스트의 보안

ECP는 CSP, iframe 샌드박싱, credentialless iframe, 엄격한 출처 검증에 관한 EP 공통 보안 요건을 그대로 따릅니다. 전체 명세는 [임베디드 프로토콜 — 보안](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#security)을 참고하세요.

### 원치 않는 결제 요청 방지

**취약점:** 악의적이거나 침해된 비즈니스가 사용자 상호작용 없이 프로그래밍 방식으로 `ec.payment.credential_request`를 트리거할 수 있습니다.

**완화책(호스트 제어 실행):** 이 위험을 없애기 위해 호스트를 결제 실행의 유일한 신뢰 개시자로 지정합니다. 호스트는 토큰을 내주기 전에 사용자 확인 UI를 표시해야 합니다(SHOULD). 트리거가 임베디드 체크아웃에서 시작된 경우 사용자 모르게 토큰화하는 것은 엄격히 금지됩니다(PROHIBITED).

## 스키마 정의

다음 스키마는 임베디드 체크아웃 프로토콜과 그 확장에서 사용하는 데이터 구조를 정의합니다.

### 체크아웃

라인 아이템, 합계, 구매자 정보를 포함해 거래의 현재 상태를 나타내는 핵심 객체.

| 이름         | 타입                                                                                                      | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | --------------------------------------------------------------------------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | [UCP Checkout Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-checkout-schema) | **Required** | UCP metadata for checkout responses.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| id           | string                                                                                                    | **Required** | Unique identifier of the checkout session.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| line_items   | Array\[[Line Item Response](/ucp-ko/draft/draft/specification/reference/#line-item)\]                     | **Required** | List of line items being checked out.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| buyer        | [Buyer](/ucp-ko/draft/draft/specification/reference/#buyer)                                               | Optional     | Representation of the buyer.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| context      | [Context](/ucp-ko/draft/draft/specification/reference/#context)                                           | Optional     | Provisional buyer signals for relevance and localization—not authoritative data. Businesses SHOULD use these values when verified inputs (e.g., shipping address) are absent, and MAY ignore or down-rank them if inconsistent with higher-confidence signals (authenticated account, risk detection) or regulatory constraints (export controls). Eligibility and policy enforcement MUST occur at checkout time using binding transaction data. Context SHOULD be non-identifying and can be disclosed progressively—coarse signals early, finer resolution as the session progresses. Higher-resolution data (shipping address, billing address) supersedes context. |
| signals      | [Signals](/ucp-ko/draft/draft/specification/reference/#signals)                                           | Optional     | Environment data provided by the platform to support authorization and abuse prevention. Values MUST NOT be buyer-asserted claims — platforms provide signals based on direct observation or independently verifiable third-party attestations. All signal keys MUST use reverse-domain naming to ensure provenance and prevent collisions when multiple extensions contribute to the shared namespace.                                                                                                                                                                                                                                                                 |
| attribution  | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                                   | Optional     | Platform-emitted referral and conversion-event context — campaign identifiers, click IDs, source/medium markers, etc. The same parameters platforms communicate via URL query parameters in browser-based flows.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| status       | string                                                                                                    | **Required** | Checkout state indicating the current phase and required processing. See Checkout Status lifecycle documentation for state transition details. **Enum:** `incomplete`, `requires_escalation`, `ready_for_complete`, `complete_in_progress`, `completed`, `canceled`                                                                                                                                                                                                                                                                                                                                                                                                     |
| currency     | string                                                                                                    | **Required** | ISO 4217 currency code reflecting the merchant's market determination. Derived from address, context, and geo IP—buyers provide signals, merchants determine currency.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| totals       | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                             | **Required** | Different cart totals.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| actions      | [Actions](/ucp-ko/draft/draft/specification/reference/#actions)                                           | Optional     | Outstanding extension-defined Actions for this checkout.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| messages     | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                                  | Optional     | List of messages with error and info about the checkout session state.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| links        | Array\[[Link](/ucp-ko/draft/draft/specification/reference/#link)\]                                        | **Required** | Links to be displayed by the platform (Privacy Policy, TOS). Mandatory for legal compliance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| policies     | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                                    | Optional     | Policies (e.g., return/refund terms) that apply to the items in this checkout. `applies_to` targets are relative to the response root; when absent or empty, refer to the URLs in `links[]`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| expires_at   | string                                                                                                    | Optional     | RFC 3339 expiry timestamp. Default TTL is 6 hours from creation if not sent.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| continue_url | string                                                                                                    | Optional     | URL for checkout handoff and session recovery. MUST be provided when status is requires_escalation. See specification for format and availability requirements.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| payment      | [Payment](/ucp-ko/draft/draft/specification/reference/#payment)                                           | Optional     | Payment configuration containing handlers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| order        | [Order Confirmation](/ucp-ko/draft/draft/specification/reference/#order-confirmation)                     | Optional     | Details about an order created for this checkout session.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |

### 주문

체크아웃이 성공적으로 완료되면 반환되는 객체로, 확인 세부 정보를 담습니다.

| 이름          | 타입                                                                                                | 필수 여부    | 설명                                                                                                                                                               |
| ------------- | --------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ucp           | [UCP Order Response Schema](/ucp-ko/draft/draft/specification/reference/#ucp-response-order-schema) | **Required** | UCP metadata for order responses. No payment handlers needed post-purchase.                                                                                        |
| id            | string                                                                                              | **Required** | Unique order identifier.                                                                                                                                           |
| label         | string                                                                                              | Optional     | Human-readable label for identifying the order. MUST only be provided by the business.                                                                             |
| checkout_id   | string                                                                                              | **Required** | Associated checkout ID for reconciliation.                                                                                                                         |
| permalink_url | string                                                                                              | **Required** | Permalink to access the order on merchant site.                                                                                                                    |
| line_items    | Array\[[Order Line Item](/ucp-ko/draft/draft/specification/reference/#order-line-item)\]            | **Required** | Line items representing what was purchased — can change post-order via edits or exchanges.                                                                         |
| fulfillment   | object                                                                                              | **Required** | Fulfillment data: buyer expectations and what actually happened.                                                                                                   |
| adjustments   | Array\[[Adjustment](/ucp-ko/draft/draft/specification/reference/#adjustment)\]                      | Optional     | Post-order events (refunds, returns, credits, disputes, cancellations, etc.) that exist independently of fulfillment.                                              |
| currency      | string                                                                                              | **Required** | ISO 4217 currency code. MUST match the currency from the originating checkout session.                                                                             |
| totals        | [Totals](/ucp-ko/draft/draft/specification/reference/#totals)                                       | **Required** | Different totals for the order.                                                                                                                                    |
| policies      | Array\[[Policy](/ucp-ko/draft/draft/specification/reference/#policy)\]                              | Optional     | Snapshot of the policies that applied to the items at checkout, captured on the order as a durable record. `applies_to` targets are relative to the response root. |
| messages      | Array\[[Message](/ucp-ko/draft/draft/specification/reference/#message)\]                            | Optional     | Business outcome messages (errors, warnings, informational). Present when the business needs to communicate status or issues to the platform.                      |
| attribution   | [Attribution](/ucp-ko/draft/draft/specification/reference/#attribution)                             | Optional     | Snapshot of the attribution associated with the originating checkout. Read-only on the order.                                                                      |

### 결제

| 이름        | 타입                                                                                                                                                   | 필수 여부 | 설명                                                                                                                                                                                                                       |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| instruments | Array\[[Payment Instrument Selected Payment Instrument](/ucp-ko/draft/draft/specification/reference/#payment-instrument-selected-payment-instrument)\] | Optional  | The payment instruments available for this payment. Each instrument is associated with a specific handler via the handler_id field. Handlers can extend the base payment_instrument schema to add handler-specific fields. |

### 결제 수단

구매자가 사용할 수 있는 특정 결제 방법(예: 특정 신용카드, 은행 계좌, 지갑 자격 증명)을 나타냅니다.

| 이름            | 타입                                                                                  | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                  |
| --------------- | ------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id              | string                                                                                | **Required** | A unique identifier for this instrument instance. Typically assigned by the platform for instruments it collects. For a business-owned saved instrument returned on an identity-linked response, this identifier is assigned by the business; the platform MUST treat it as an opaque, business-scoped reference, and the business resolves it server-side when the buyer selects it. |
| handler_id      | string                                                                                | **Required** | The unique identifier for the handler instance that produced this instrument. This corresponds to the 'id' field in the Payment Handler definition.                                                                                                                                                                                                                                   |
| type            | string                                                                                | **Required** | The broad category of the instrument (e.g., 'card', 'tokenized_card'). Specific schemas will constrain this to a constant value.                                                                                                                                                                                                                                                      |
| billing_address | [Postal Address](/ucp-ko/draft/draft/specification/reference/#postal-address)         | Optional     | The billing address associated with this payment method.                                                                                                                                                                                                                                                                                                                              |
| credential      | [Payment Credential](/ucp-ko/draft/draft/specification/reference/#payment-credential) | Optional     | The base definition for any payment credential. Handlers define specific credential types.                                                                                                                                                                                                                                                                                            |
| display         | object                                                                                | Optional     | Display information for this payment instrument. Each payment instrument schema defines its specific display properties, as outlined by the payment handler.                                                                                                                                                                                                                          |

#### 선택된 결제 수단

A payment instrument with selection state.

| 이름            | 타입                                                                                  | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                  |
| --------------- | ------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id              | string                                                                                | **Required** | A unique identifier for this instrument instance. Typically assigned by the platform for instruments it collects. For a business-owned saved instrument returned on an identity-linked response, this identifier is assigned by the business; the platform MUST treat it as an opaque, business-scoped reference, and the business resolves it server-side when the buyer selects it. |
| handler_id      | string                                                                                | **Required** | The unique identifier for the handler instance that produced this instrument. This corresponds to the 'id' field in the Payment Handler definition.                                                                                                                                                                                                                                   |
| type            | string                                                                                | **Required** | The broad category of the instrument (e.g., 'card', 'tokenized_card'). Specific schemas will constrain this to a constant value.                                                                                                                                                                                                                                                      |
| billing_address | [Postal Address](/ucp-ko/draft/draft/specification/reference/#postal-address)         | Optional     | The billing address associated with this payment method.                                                                                                                                                                                                                                                                                                                              |
| credential      | [Payment Credential](/ucp-ko/draft/draft/specification/reference/#payment-credential) | Optional     | The base definition for any payment credential. Handlers define specific credential types.                                                                                                                                                                                                                                                                                            |
| display         | object                                                                                | Optional     | Display information for this payment instrument. Each payment instrument schema defines its specific display properties, as outlined by the payment handler.                                                                                                                                                                                                                          |
| selected        | boolean                                                                               | Optional     | Whether this instrument is selected by the user.                                                                                                                                                                                                                                                                                                                                      |

### 카드 결제 수단

| 이름            | 타입                                                                                  | 필수 여부    | 설명                                                                                                                                                                                                                                                                                                                                                                                  |
| --------------- | ------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| id              | string                                                                                | **Required** | A unique identifier for this instrument instance. Typically assigned by the platform for instruments it collects. For a business-owned saved instrument returned on an identity-linked response, this identifier is assigned by the business; the platform MUST treat it as an opaque, business-scoped reference, and the business resolves it server-side when the buyer selects it. |
| handler_id      | string                                                                                | **Required** | The unique identifier for the handler instance that produced this instrument. This corresponds to the 'id' field in the Payment Handler definition.                                                                                                                                                                                                                                   |
| type            | string                                                                                | **Required** | **Constant = card**. Indicates this is a card payment instrument. The broad category of the instrument (e.g., 'card', 'tokenized_card'). Specific schemas will constrain this to a constant value.                                                                                                                                                                                    |
| billing_address | [Postal Address](/ucp-ko/draft/draft/specification/reference/#postal-address)         | Optional     | The billing address associated with this payment method.                                                                                                                                                                                                                                                                                                                              |
| credential      | [Payment Credential](/ucp-ko/draft/draft/specification/reference/#payment-credential) | Optional     | The base definition for any payment credential. Handlers define specific credential types.                                                                                                                                                                                                                                                                                            |
| display         | object                                                                                | Optional     | Display information for this card payment instrument. Display information for this payment instrument. Each payment instrument schema defines its specific display properties, as outlined by the payment handler.                                                                                                                                                                    |
| network         | string                                                                                | Optional     | Card network elected for this transaction, typically a co-badged selection. When present, the business MAY decline if the card cannot route over it and MUST NOT substitute another.                                                                                                                                                                                                  |

### 결제 자격 증명

| 이름 | 타입   | 필수 여부    | 설명                                                                                         |
| ---- | ------ | ------------ | -------------------------------------------------------------------------------------------- |
| type | string | **Required** | The credential type discriminator. Specific schemas will constrain this to a constant value. |

### 토큰 자격 증명

| 이름 | 타입   | 필수 여부    | 설명                                                                                                                                                                    |
| ---- | ------ | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type | string | **Required** | The specific type of token produced by the handler (e.g., 'stripe_token'). The credential type discriminator. Specific schemas will constrain this to a constant value. |

### 카드 자격 증명

| 이름             | 타입    | 필수 여부    | 설명                                                                                                                                                                                                                 |
| ---------------- | ------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| type             | string  | **Required** | **Constant = card**. The credential type identifier for card credentials. The credential type discriminator. Specific schemas will constrain this to a constant value.                                               |
| card_number_type | string  | **Required** | Deprecated: the credential type now carries this distinction. The type of card number. Network tokens are preferred with fallback to FPAN. See PCI Scope for more details. **Enum:** `fpan`, `network_token`, `dpan` |
| number           | string  | Optional     | Card number.                                                                                                                                                                                                         |
| expiry_month     | integer | Optional     | The month of the card's expiration date (1-12).                                                                                                                                                                      |
| expiry_year      | integer | Optional     | The year of the card's expiration date.                                                                                                                                                                              |
| name             | string  | Optional     | Cardholder name.                                                                                                                                                                                                     |
| cvc              | string  | Optional     | Card CVC number.                                                                                                                                                                                                     |
| cryptogram       | string  | Optional     | Cryptogram provided with network tokens.                                                                                                                                                                             |
| eci_value        | string  | Optional     | Electronic Commerce Indicator / Security Level Indicator provided with network tokens.                                                                                                                               |

### 결제 핸들러

특정 결제 수단의 인증과 처리를 담당하는 처리자 또는 지갑 제공자(예: Google Pay, Stripe, 은행 앱)를 나타냅니다.

Handler reference in responses. May include full config state for runtime usage of the handler.

| 이름                  | 타입                                                                                                               | 필수 여부    | 설명                                                                                                                             |
| --------------------- | ------------------------------------------------------------------------------------------------------------------ | ------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| version               | string                                                                                                             | **Required** | Entity version in YYYY-MM-DD format.                                                                                             |
| spec                  | string                                                                                                             | Optional     | URL to human-readable specification document.                                                                                    |
| schema                | string                                                                                                             | Optional     | URL to JSON Schema defining this entity's structure and payloads.                                                                |
| id                    | string                                                                                                             | **Required** | Unique identifier for this entity instance. Used to disambiguate when multiple instances exist.                                  |
| config                | object                                                                                                             | Optional     | Entity-specific configuration. Structure defined by each entity's schema.                                                        |
| available_instruments | Array\[[Available Payment Instrument](/ucp-ko/draft/draft/specification/reference/#available-payment-instrument)\] | Optional     | Instrument types this handler supports, with optional constraints. When absent, every instrument should be considered available. |
