# 임베디드 프로토콜(EP)

## 소개

임베디드 프로토콜(Embedded Protocol, EP)은 **호스트**가 **비즈니스**의 인터페이스를 iframe이나 웹뷰에 임베드하고 그 채널로 구조화된 메시지를 주고받을 수 있게 하는 UCP 전송 바인딩입니다. EP는 **어떻게** 통신하는지, 즉 메시지 형식, 채널 설정, 인증, 오류 처리, 보안 제약을 정의합니다. **어떤** 데이터가 오가는지는 정의하지 않으며, 그 부분은 EP에 바인딩되는 각 기능(Capability)이 맡습니다.

각 기능은 자체 메서드 접두사 아래에 EP 메서드를 구현합니다. 예를 들어 체크아웃은 `ec.*` 접두사를, 장바구니는 `ep.cart.*` 접두사를 사용합니다. 이 문서에서 설명하는 공통 패턴은 모든 EP 기능에 똑같이 적용됩니다.

디스커버리, URL 파라미터, 위임 계약, 메시지 페이로드, 스키마 정의 같은 기능별 세부 사항은 각 기능의 EP 바인딩 명세에서 정의합니다.

- [체크아웃 기능 — EP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/checkout/embedded/index.md)
- [장바구니 기능 — EP 바인딩](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/shopping/cart/embedded/index.md)

## 용어와 행위자

### 커머스 역할

- **비즈니스:** 상품이나 서비스를 제공하는 판매 주체.
- **구매자:** 구매를 진행하는 최종 사용자.

### 기술 구성 요소

- **호스트:** 비즈니스의 인터페이스를 임베드하는 애플리케이션(예: AI 에이전트 앱, 슈퍼 앱, 브라우저). 사용자 인증을 담당하며, 위임받은 경우 결제나 주소 선택 같은 작업을 위한 네이티브 UI도 담당합니다.
- **임베디드 컨텍스트:** iframe이나 웹뷰에 렌더링된 비즈니스의 인터페이스. 기능별 흐름(예: 체크아웃, 장바구니 구성)을 담당합니다.

## 전송과 메시징

### 메시지 형식

모든 EP 메시지는 반드시 JSON-RPC 2.0 형식 ([RFC 7159](https://datatracker.ietf.org/doc/html/rfc7159))을 사용해야 합니다(**MUST**). 각 메시지는 반드시 다음을 포함해야 합니다(**MUST**).

- `jsonrpc`: 반드시 `"2.0"`이어야 함(**MUST**)
- `method`: 메시지 이름(예: `"ec.start"`, `"ep.cart.start"`)
- `params`: 메시지별 페이로드(빈 객체일 수 있음)
- `id`: (선택) 응답을 기대하는 요청에만 존재

### 메시지 유형

**요청**(`id` 필드 있음):

- 수신자의 응답이 필요합니다.
- 반드시 고유한 `id` 필드를 포함해야 합니다(**MUST**).
- 수신자는 반드시 일치하는 `id`로 응답해야 합니다(**MUST**).
- 응답은 반드시 `result` 또는 `error_response` 중 하나여야 합니다(**MUST**).
- 확인 응답이나 데이터가 필요한 작업에 사용합니다.

**알림**(`id` 필드 없음):

- 정보 제공용이며 응답을 기대하지 않습니다.
- `id` 필드를 포함해서는 안 됩니다(**MUST NOT**).
- 수신자는 응답을 보내서는 안 됩니다(**MUST NOT**).
- 상태 업데이트와 정보성 이벤트에 사용합니다.

### 응답 처리

요청(`id`가 있는 메시지)을 받으면 수신자는 반드시 `result`로 응답해야 합니다(**MUST**). UCP의 2계층 오류 모델에 따라 성공 결과와 오류 결과 모두 반드시 `result` 필드로 반환해야 합니다(**MUST**). JSON-RPC `error` 필드는 전송 수준 실패(파싱 오류, 메서드 없음, 잘못된 파라미터) 전용입니다. 구현체는 애플리케이션 수준 오류 코드에 JSON-RPC `error` 필드를 사용해서는 안 됩니다(**MUST NOT**).

**성공 응답:**

```json
{
  "jsonrpc": "2.0",
  "id": "req_1",
  "result": {
    "ucp": { "version": "2026-08-25", "status": "success" }
    // ... result fields
  }
}
```

**오류 응답:**

```json
{
  "jsonrpc": "2.0",
  "id": "...",
  "result": {
    "ucp": { "version": "2026-08-25", "status": "error" },
    "messages": [...]
  }
}
```

두 경우 모두 `result.ucp.status`가 성공과 오류 결과를 구분하는 판별자 역할을 합니다. 모든 UCP 전송 방식에서 사용하는 것과 같은 패턴입니다.

### 전송 오류

전송 오류는 요청 처리를 막는 프로토콜 수준의 실패입니다. 표준 JSON-RPC 오류 코드를 사용해 JSON-RPC `error`로 반환합니다. 이는 비즈니스 로직이 실행되어 오류 결과를 냈다는 뜻이 아니라, 메시지 자체가 유효하지 않거나 처리할 수 없었다는 뜻입니다. 전체 오류 코드 레지스트리는 [핵심 명세](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/overview/#error-codes)를 참고하세요.

예를 들어 요청을 처리할 수 없으면(알 수 없는 메서드, 형식이 잘못된 파라미터) 호스트는 반드시 JSON-RPC `error`로 응답해야 합니다(**MUST**).

```json
{
    "jsonrpc": "2.0",
    "id": "req_1",
    "error": {
        "code": -32601,
        "message": "Method not found."
    }
}
```

## 통신 채널

### 웹 기반 호스트

호스트가 웹 애플리케이션이면, 통신은 호스트와 임베디드 컨텍스트의 window 사이에서 `postMessage`로 시작합니다. 호스트는 반드시 임베디드 window에서 오는 `postMessage` 호출을 수신 대기해야 하며(**MUST**), 메시지를 받으면 반드시 그 origin이 세션을 시작할 때 사용한 `continue_url`과 일치하는지 검증해야 합니다(**MUST**).

검증이 끝나면 호스트는 `MessageChannel`을 만들고 그 포트 중 하나를 `ready` 응답에 담아 전달할 수 있습니다(**MAY**). 호스트가 `MessagePort`로 응답하면 이후의 모든 메시지는 반드시 그 채널로 보내야 합니다(**MUST**). 그렇지 않으면 호스트와 비즈니스는 origin 검증을 포함해 각자의 `window` 객체 사이에서 반드시 `postMessage()`를 계속 사용해야 합니다(**MUST**).

### 네이티브 호스트

호스트가 네이티브 애플리케이션이면, 웹과 네이티브 환경 사이에서 `postMessage` 통신할 수 있도록 반드시 임베디드 컨텍스트에 전역 객체를 주입해야 합니다(**MUST**). 각 기능은 다음 패턴에 따라 자체 전역 객체 이름을 정의합니다.

- `window.Embedded{Capability}ProtocolConsumer` (권장)
- `window.webkit.messageHandlers.Embedded{Capability}ProtocolConsumer`

이 객체는 반드시 다음 인터페이스를 구현해야 합니다(**MUST**).

```javascript
{
  postMessage(message: string): void
}
```

여기서 `message`는 JSON 문자열로 직렬화된 JSON-RPC 2.0 메시지입니다. 호스트는 반드시 처리하기 전에 JSON 문자열을 파싱해야 합니다(**MUST**).

호스트에서 임베디드 컨텍스트로 메시지를 보낼 때 호스트는 반드시 JSON-RPC 메시지를 인자로 `window.Embedded{Capability}Protocol.postMessage()`를 호출하는 JavaScript를 웹뷰에 주입해야 합니다(**MUST**). 임베디드 컨텍스트는 반드시 기능의 `ready` 메시지를 보내기 전에 이 전역 객체를 초기화하고 `postMessage()` 호출 수신 대기를 시작해야 합니다(**MUST**).

구체적인 전역 객체 이름은 각 기능의 EP 바인딩을 참고하세요.

- **체크아웃:** `EmbeddedCheckoutProtocolConsumer` / `EmbeddedCheckoutProtocol`
- **장바구니:** `EmbeddedCartProtocolConsumer` / `EmbeddedCartProtocol`

## 메시지 패턴

### 핸드셰이크

렌더링되면 임베디드 컨텍스트는 반드시 기능의 `ready` 메시지로 호스트에 준비 상태를 알려야 합니다(**MUST**). 이 메시지로 호스트와 임베디드 컨텍스트 사이의 보안 통신 채널을 초기화하고, 어떤 위임이 수락되었는지 전달하며, 추가 인가 교환이 필요한지 알립니다.

**일반 흐름:**

1. 임베디드 컨텍스트가 `delegate` 목록과 선택적 `auth` 요청을 담아 `ready` 요청을 보냅니다.
1. 호스트가 다음을 담아 응답합니다.
   - 협상된 프로토콜 버전을 확인하는 `ucp` 엔벌로프
   - 채널 업그레이드를 위한 `MessagePort`가 담긴 선택적 `upgrade`
   - 요청된 인가 데이터가 담긴 선택적 `credential`
   - 선택적인 기능별 상태(예: 체크아웃 위임 상태)
1. 호스트가 `upgrade`를 포함하면, 임베디드 컨텍스트는 반드시 응답의 다른 모든 필드를 버리고 업그레이드된 채널로 전환한 뒤 그 채널로 새 `ready` 메시지를 보내야 합니다(**MUST**).
1. 호스트가 `error_response` result로 응답하면 세션을 진행할 수 없습니다. 호스트는 반드시 임베디드 컨텍스트를 해제해야 하며(**MUST**), `continue_url`이 있으면 구매자를 그곳으로 리디렉션할 수 있습니다(**MAY**). 임베디드 컨텍스트는 핸드셰이크 오류를 받은 뒤 더 이상 메시지를 보내서는 안 됩니다(**MUST NOT**).

기능별 파라미터와 result 필드를 포함한 `ready` 메서드의 전체 명세는 각 기능이 정의합니다.

### 인증

임베디드 컨텍스트는 초기 핸드셰이크 이후 호스트에 인가를 요청할 수 있습니다(**MAY**). 만료된 OAuth 토큰을 갱신하는 경우가 그 예입니다. 이 교환에는 기능의 auth 메서드(예: 체크아웃은 `ec.auth`, 장바구니는 `ep.cart.auth`)를 사용합니다.

**요청:**

- **방향:** 임베디드 컨텍스트 → 호스트
- **유형:** 요청
- **페이로드:**
  - `type` (string, 필수(**REQUIRED**)): 요청하는 인가 유형(예: `"oauth"`, `"api_key"`, `"jwt"`).

**성공 응답:**

- **방향:** 호스트 → 임베디드 컨텍스트
- **유형:** 응답
- **Result 페이로드:**
  - `ucp` (object, 필수(**REQUIRED**)): `status: "success"`를 담은 UCP 프로토콜 메타데이터.
  - `credential` (string, 필수(**REQUIRED**)): 요청된 인가 데이터.

**오류 응답:**

호스트는 반드시 인가 데이터 또는 `error_response` 중 하나를 담은 `result`로 응답해야 합니다(**MUST**).

**성공 응답 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "auth_1",
    "result": {
        "ucp": { "version": "2026-08-25", "status": "success" },
        "credential": "eyJhbGciOiJSUzI1NiIs..."
    }
}
```

**오류 응답 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "auth_1",
    "result": {
        "ucp": { "version": "2026-08-25", "status": "error" },
        "messages": [
            {
                "type": "error",
                "code": "timeout_error",
                "content": "An internal service timed out when fetching the required authorization data.",
                "severity": "recoverable"
            }
        ]
    }
}
```

**오류 에스컬레이션:**

오류가 일시적이면(`recoverable` 심각도로 표시) 임베디드 컨텍스트는 auth 요청을 다시 시작할 수 있습니다(**MAY**). 그렇지 않으면 임베디드 컨텍스트는 반드시 `unrecoverable` 오류 응답을 담은 세션 오류 알림([세션 오류](#session-error) 참고)을 보내야 합니다(**MUST**). 이 에스컬레이션은 호스트가 제공한 자격 증명을 임베디드 컨텍스트가 처리할 수 없는 경우(예: 자격 증명이 손상된 경우)에도 적용됩니다. 세션 오류에는 구매자 핸드오프를 위해 `continue_url`을 포함해야 합니다(**SHOULD**).

**예시 — 세션 오류로 에스컬레이션된 auth 실패:**

```json
{
    "jsonrpc": "2.0",
    "method": "ec.error",
    "params": {
        "error": {
            "ucp": { "version": "2026-08-25", "status": "error" },
            "messages": [
                {
                    "type": "error",
                    "code": "not_supported_error",
                    "content": "Requested auth credential type is not supported",
                    "severity": "unrecoverable"
                }
            ],
            "continue_url": "https://merchant.example.com"
        }
    }
}
```

호스트가 이 알림을 받으면 반드시 임베디드 컨텍스트를 해제해야 하며(**MUST**), 적절한 오류 상태를 표시해야 합니다(**SHOULD**). `continue_url`이 있으면 호스트는 반드시 이 URL로 구매자를 핸드오프해 세션을 복구할 수 있게 해야 합니다(**MUST**).

### 세션 오류

세션 오류는 기능의 리소스와 무관한 치명적 상황을 알립니다. 세션을 더 이상 진행할 수 없게 만드는 최종 auth 실패가 그 예입니다. 각 기능은 자체 세션 오류 알림 메서드(예: 체크아웃은 `ec.error`, 장바구니는 `ep.cart.error`)를 정의합니다.

**알림 페이로드:**

- `error` (object, 필수(**REQUIRED**)): 세션 수준 오류 응답.
  - `ucp` (object, 필수(**REQUIRED**)): UCP 프로토콜 메타데이터. `status`는 반드시 `"error"`여야 합니다(**MUST**).
  - `messages` (array, 필수(**REQUIRED**)): 실패를 설명하는 하나 이상의 메시지.
  - `continue_url` (string, 선택(**OPTIONAL**)): 구매자 핸드오프 또는 세션 복구를 위한 URL.

**예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ec.error",
    "params": {
        "error": {
            "ucp": { "version": "2026-08-25", "status": "error" },
            "messages": [
                {
                    "type": "error",
                    "code": "not_supported_error",
                    "content": "Requested auth credential type is not supported.",
                    "severity": "unrecoverable"
                }
            ],
            "continue_url": "https://merchant.example.com/checkout/abc123"
        }
    }
}
```

**호스트 처리:**

호스트가 세션 오류 알림을 받으면 반드시 임베디드 컨텍스트를 해제해야 하며(**MUST**), 구매자에게 적절한 오류 상태를 표시해야 합니다(**SHOULD**). `continue_url`이 있으면 호스트는 반드시 이 URL로 구매자를 핸드오프해 세션을 복구할 수 있게 해야 합니다(**MUST**).

### 생명주기

각 기능은 공통 계약을 따르는 `start`와 `complete` 알림 메서드를 정의합니다.

- **`start`**: 임베디드 컨텍스트가 UI가 표시되어 상호작용할 준비가 되었음을 호스트에 알립니다. 알림에는 반드시 기능 리소스(예: 체크아웃 또는 장바구니 객체)의 현재 전체 상태를 포함해야 합니다(**MUST**).
- **`complete`**: 임베디드 컨텍스트가 기능 흐름이 성공적으로 끝났음을 호스트에 알립니다. 알림에는 반드시 리소스의 최종 상태를 포함해야 합니다(**MUST**).

둘 다 알림이므로 호스트는 응답해서는 안 됩니다(**MUST NOT**).

**예시 — start 알림(장바구니):**

```json
{
    "jsonrpc": "2.0",
    "method": "ep.cart.start",
    "params": {
        "cart": {
            "id": "cart_123",
            "currency": "USD",
            "totals": [ ... ],
            "line_items": [ ... ],
            "buyer": { ... }
        }
    }
}
```

**예시 — complete 알림(체크아웃):**

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

### 상태 변경

상태 변경 알림은 임베디드 컨텍스트에서 이미 일어난 변경을 호스트에 알립니다. 이 알림은 정보 제공용일 뿐이며, 임베디드 컨텍스트는 이미 변경을 적용하고 갱신된 UI를 렌더링한 상태입니다. 호스트는 상태 변경 알림에 응답해서는 안 됩니다(**MUST NOT**).

각 기능은 관련 리소스 필드(예: 라인 아이템, 구매자 정보, 총액)를 다루는 자체 상태 변경 메서드 집합을 정의합니다. 상태 변경 알림에는 변경된 필드만이 아니라 반드시 기능 리소스의 현재 전체 상태를 포함해야 합니다(**MUST**).

**예시 — 라인 아이템 변경(체크아웃):**

```json
{
    "jsonrpc": "2.0",
    "method": "ec.line_items.change",
    "params": {
        "checkout": {
            "id": "checkout_123",
            "totals": [ ... ],
            "line_items": [ ... ]
            // ... other checkout fields
        }
    }
}
```

**예시 — 메시지 변경(장바구니):**

```json
{
    "jsonrpc": "2.0",
    "method": "ep.cart.messages.change",
    "params": {
        "cart": {
            "id": "cart_123",
            "line_items": [ ... ],
            "messages": [
                {
                    "type": "error",
                    "code": "invalid_quantity",
                    "path": "$.line_items[0].quantity",
                    "content": "Quantity must be at least 1",
                    "severity": "recoverable"
                }
            ]
            // ... other cart fields
        }
    }
}
```

## 오류 코드

임베디드 프로토콜은 공통 오류 코드 집합을 정의합니다. 각 기능은 기능별 시나리오에 맞게 추가 오류 코드를 정의할 수 있습니다(**MAY**).

| 코드                  | 심각도          | 설명                                                            |
| --------------------- | --------------- | --------------------------------------------------------------- |
| `abort_error`         | `recoverable`   | 사용자가 상호작용을 취소했습니다(예: 시트를 닫음).              |
| `security_error`      | `unrecoverable` | 호스트 origin 검증에 실패했습니다.                              |
| `invalid_state_error` | `unrecoverable` | 핸드셰이크가 순서에 맞지 않게 시도되었습니다.                   |
| `not_supported_error` | `unrecoverable` | 요청한 작업 또는 인가 유형을 지원하지 않습니다.                 |
| `timeout_error`       | `recoverable`   | 내부 서비스가 시간 초과되었습니다. 작업을 재시도할 수 있습니다. |

오류에는 `recoverable`과 `unrecoverable` 심각도만 사용해야 합니다(**SHOULD**). `recoverable`은 작업을 다시 시도할 수 있다는 뜻입니다. `unrecoverable`은 현재 세션에서 작업이 성공할 수 없다는 뜻입니다.

## 보안

### 콘텐츠 보안 정책(CSP)

보안을 위해 양측은 반드시 적절한 **[콘텐츠 보안 정책(CSP)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)** 지시어를 구현해야 합니다(**MUST**).

- **비즈니스:** 신뢰할 수 있는 호스트만 임베드할 수 있도록 반드시 `frame-ancestors <host_origin>;`를 설정해야 합니다(**MUST**).

- **호스트:**

  - **직접 임베딩:** 호스트가 비즈니스의 페이지를 직접 임베드하는 경우, 특히 비즈니스가 많다면 가능한 모든 비즈니스 origin을 나열한 `frame-src` 지시어를 지정하기가 현실적으로 어려울 수 있습니다. 이 경우 엄격한 `frame-src`가 이상적이긴 하지만, [iframe 샌드박스 속성](#iframe-sandbox-attributes)과 [Credentialless iframe](#credentialless-iframes)에서 설명하는 것과 같은 다른 보안 조치가 매우 중요합니다.
  - **중간 iframe:** 호스트는 비즈니스의 페이지를 임베드하기 위해 중간 iframe(예: 호스트가 제어하는 서브도메인)을 사용할 수 있습니다(**MAY**). 이렇게 하면 더 세밀하게 제어할 수 있습니다.
    - 호스트의 메인 페이지는 `frame-src`에 중간 iframe의 origin만 허용하면 됩니다(예: `frame-src <intermediate_iframe_origin>;`).
    - 중간 iframe은 반드시 현재 임베디드 세션의 특정 `<merchant_origin>`*만* 허용하도록 동적으로 설정되는 엄격한 `frame-src` 정책을 구현해야 합니다(**MUST**)(예: `frame-src <merchant_origin>;`). 이 정책은 중간 iframe 콘텐츠를 제공할 때 HTTP 헤더로 설정할 수 있습니다.

### iframe 샌드박스 속성

모든 비즈니스 iframe은 권한을 제한하기 위해 반드시 샌드박스 처리해야 합니다(**MUST**). 다음 샌드박스 속성을 적용해야 하지만(**SHOULD**), 호스트와 비즈니스는 추가 권한을 협상할 수 있습니다(**MAY**).

```html
<iframe sandbox="allow-scripts allow-forms allow-same-origin"></iframe>
```

### Credentialless iframe

호스트는 iframe에 `credentialless` 속성을 사용해 새로운 일회성 컨텍스트에서 로드해야 합니다(**SHOULD**). 이렇게 하면 비즈니스가 여러 컨텍스트에 걸쳐 사용자 활동을 연결하거나 기존 세션에 접근하지 못하므로 사용자 개인정보가 보호됩니다.

```html
<iframe credentialless src="https://business.example.com/..."></iframe>
```

### 엄격한 origin 검증

프레임 간의 모든 `postMessage` 통신에서 `origin`을 엄격하게 검증합니다.
