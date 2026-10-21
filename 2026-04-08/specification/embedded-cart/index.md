# 장바구니 기능 - EP 바인딩

## 소개

임베디드 장바구니 프로토콜(Embedded Cart Protocol, ECaP)은 UCP 임베디드 프로토콜(EP) 전송 바인딩을 장바구니용으로 구현한 것입니다. **호스트**는 ECaP로 **비즈니스**의 장바구니 인터페이스를 임베드하고, 구매자가 장바구니를 조작할 때마다 이벤트를 받습니다. ECaP는 REST와 같은 전송 바인딩입니다. 어떤 데이터가 있는지(**무엇**)가 아니라 **어떻게** 통신하는지를 정의합니다.

## 용어와 행위자

### 커머스 역할

- **비즈니스(Business):** 상품·서비스와 장바구니 구성 경험을 제공하는 판매 주체.
- **구매자(Buyer):** 장바구니 구성을 통해 구매하려는 최종 사용자.

### 기술 구성 요소

- **호스트(Host):** 장바구니를 임베드하는 애플리케이션(예: AI 에이전트 앱, 슈퍼 앱, 브라우저). 사용자 인증(신원 연결 같은 사전 요건 포함)을 담당합니다.
- **임베디드 장바구니(Embedded Cart):** iframe 또는 웹뷰에서 렌더링되는 비즈니스의 장바구니 인터페이스. 장바구니 구성 흐름을 담당하고, 체크아웃 생성 같은 하위 퍼널 단계로 넘어갈 수도 있습니다.

### 디스커버리

ECaP 지원 여부는 서비스 디스커버리로 알립니다. 비즈니스가 `/.well-known/ucp` 프로필에 `embedded` 전송 방식을 선언하면, 모든 장바구니 `continue_url` 값이 임베디드 장바구니 프로토콜을 지원합니다.

**서비스 디스커버리 예시:**

```json
{
    "services": {
        "dev.ucp.shopping": [
            {
                "version": "2026-04-08",
                "transport": "rest",
                "schema": "https://ucp.dev/2026-04-08/services/shopping/rest.openapi.json",
                "endpoint": "https://merchant.example.com/ucp/v1"
            },
            {
                "version": "2026-04-08",
                "transport": "mcp",
                "schema": "https://ucp.dev/2026-04-08/services/shopping/mcp.openrpc.json",
                "endpoint": "https://merchant.example.com/ucp/mcp"
            },
            {
                "version": "2026-04-08",
                "transport": "embedded",
                "schema": "https://ucp.dev/2026-04-08/services/shopping/embedded.openrpc.json"
            }
        ]
    }
}
```

서비스 정의에 `embedded`가 없으면 비즈니스는 `continue_url`로 리디렉션하는 방식의 장바구니 이어 가기만 지원합니다.

#### 장바구니별 구성

서비스 수준 디스커버리는 비즈니스가 ECaP를 지원한다는 사실을 선언할 뿐, 모든 장바구니 세션에서 ECaP를 켠다고 보장하지는 않습니다. 비즈니스는 해당 세션의 ECaP 지원 여부와 허용된 위임 항목을 알리기 위해 장바구니 응답에 `config.delegate`가 포함된 임베디드 서비스 바인딩을 반드시 포함해야 합니다(**MUST**).

**장바구니 응답 예시:**

```json
{
    "id": "cart_123",
    "continue_url": "https://merchant.example.com/cart/cart123",
    "ucp": {
        "version": "2026-04-08",
        "services": {
            "dev.ucp.shopping": [
                {
                    "version": "2026-04-08",
                    "transport": "embedded",
                    "config": {
                        "delegate": []
                    }
                }
            ]
        },
        "capabilities": {...},
        "payment_handlers": {...}
    }
    // ...other cart fields...
}
```

### 임베디드 장바구니 URL 로드

ECaP 지원을 선언한 비즈니스에서 `continue_url`이 담긴 장바구니 응답을 받으면, 호스트는 그 URL을 임베디드 컨텍스트에 로드해 ECaP 세션을 시작할 수 있습니다(**MAY**).

**예시:**

```text
https://example.com/cart/cart123?ep_version=2026-01-23...
```

참고: 모든 쿼리 파라미터 값은 RFC 3986에 따라 올바르게 URL 인코딩해야 합니다.

임베디드 컨텍스트를 로드하기 전에 호스트는 다음을 수행해야 합니다(**SHOULD**).

1. 응답의 `config.delegate`에서 사용할 수 있는 위임 항목을 확인합니다.
1. 비즈니스가 요구하면 인증 절차(예: 신원 연결)를 선택적으로 완료합니다.

세션을 시작하려면 호스트는 반드시 `continue_url`에 지원되는 ECaP 쿼리 파라미터를 추가해야 합니다(**MUST**).

여러 임베딩 환경에서 호환성을 최대한 보장하기 위해 모든 ECaP 파라미터는 HTTP 헤더가 아니라 URL 쿼리 문자열로 전달합니다. 네임스페이스 오염을 피하고 비즈니스 고유의 쿼리 파라미터와 명확히 구분할 수 있도록, ECaP 파라미터에는 `ep` 또는 `ep_cart` 접두사를 붙여야 합니다(**SHOULD**).

- `ep_version` (string, **필수(REQUIRED)**): 이 세션의 UCP 버전 (형식: `YYYY-MM-DD`). 서비스 디스커버리의 버전과 일치해야 합니다.
- `ep_auth` (string, **선택(OPTIONAL)**): 비즈니스가 정의한 형식의 인증 토큰.
- `ep_color_scheme` (string, **선택(OPTIONAL)**): 장바구니 UI의 색상 테마 선호. 유효한 값: `light`, `dark`. 지정하지 않으면 임베디드 장바구니는 시스템 설정을 따릅니다.
- `ep_cart_delegate` (string, **선택(OPTIONAL)**): 호스트가 처리하려는 위임 항목의 쉼표 구분 목록. 위임이 필요 없으면 비워 둘 수 있습니다(**MAY**). 임베디드 서비스 바인딩 `config.delegate`의 부분 집합이어야 합니다(**SHOULD**).

## 전송과 메시징

ECaP는 공용 EP 전송 계층을 사용합니다. 메시지 형식, 메시지 유형, 응답 처리 규칙은 [임베디드 프로토콜 — 전송과 메시징](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/embedded-protocol/#transport-messaging)을 참고하세요.

모든 응답의 `ucp.version`은 세션 초기화 때 협상되고 호스트가 `ep.cart.ready` 응답에서 확인한 `ep_version`과 반드시 같아야 합니다(**MUST**). 버전은 세션에 고정되며, ECaP 세션이 이어지는 동안 바뀌어서는 안 됩니다(**MUST NOT**).

### 통신 채널

ECaP는 공용 EP 통신 채널 모델을 따릅니다. 일반적인 패턴은 [임베디드 프로토콜 — 통신 채널](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/embedded-protocol/#communication-channels)을 참고하세요.

네이티브 호스트에서 쓰는 장바구니 전용 전역 객체는 다음과 같습니다.

- `window.EmbeddedCartProtocolConsumer` (권장)
- `window.webkit.messageHandlers.EmbeddedCartProtocolConsumer`
- `window.EmbeddedCartProtocol` (호스트 → 임베디드 장바구니)

## 메시지 API 레퍼런스

### 메시지 분류

#### 핵심 메시지

핵심 메시지는 ECaP 명세에서 정의하며, 모든 구현이 반드시 지원해야 합니다(**MUST**).

| 분류           | 목적                                           | 패턴 | 핵심 메시지                                                                    |
| -------------- | ---------------------------------------------- | ---- | ------------------------------------------------------------------------------ |
| **핸드셰이크** | 호스트와 임베디드 장바구니 간 연결 수립        | 요청 | `ep.cart.ready`                                                                |
| **인증**       | 임베디드 장바구니와 호스트 간 인증 데이터 교환 | 요청 | `ep.cart.auth`                                                                 |
| **수명 주기**  | 임베디드 장바구니의 장바구니 상태 알림         | 알림 | `ep.cart.start`, `ep.cart.complete`                                            |
| **상태 변경**  | 장바구니 필드 변경 알림                        | 알림 | `ep.cart.line_items.change`, `ep.cart.buyer.change`, `ep.cart.messages.change` |
| **세션 오류**  | 장바구니 리소스와 무관한 세션 수준 오류 알림   | 알림 | `ep.cart.error`                                                                |

### 핸드셰이크 메시지

#### `ep.cart.ready`

렌더링이 끝나면 임베디드 장바구니는 `ep.cart.ready` 메시지로 상위 컨텍스트에 준비가 끝났음을 반드시 알려야 합니다(**MUST**). 이 메시지는 호스트와 임베디드 장바구니 사이의 보안 통신 채널을 초기화하고, 추가 인증 교환이 필요한지 알리며, 호스트가 요청받은 인가 데이터를 임베디드 장바구니에 돌려줄 수 있게 합니다.

- **방향:** 임베디드 장바구니 → 호스트
- **유형:** 요청
- **페이로드:**
  - `delegate` (array of strings, **필수(REQUIRED)**): 임베디드 장바구니가 수락한 위임 식별자 목록. `ep_cart_delegate`(호스트가 요청한 항목)와 장바구니 응답의 `config.delegate`(비즈니스가 허용한 항목) 양쪽의 부분 집합이어야 합니다(**MUST**). 빈 배열이면 수락된 위임이 없다는 뜻입니다.
  - `auth` (object, **선택(OPTIONAL)**): 추가 고려 사항 때문에 `ep_auth` URL 파라미터로 충분하지 않거나 쓸 수 없으면, 비즈니스는 이 객체에 `type` 문자열을 지정해 초기 핸드셰이크 중에 인가를 요청할 수 있습니다. 이 `type` 문자열 값은 [`ep.cart.auth`](#epcartauth)에 담기는 페이로드 내용과 같습니다.

**메시지 예시(수락된 위임 없음):**

```json
{
    "jsonrpc": "2.0",
    "id": "ready_1",
    "method": "ep.cart.ready",
    "params": {
        "delegate": [],
        "auth": {
            "type": "oauth"
        }
    }
}
```

`ep.cart.ready` 메시지는 요청이므로, 호스트는 핸드셰이크를 완료하려면 반드시 응답해야 합니다(**MUST**).

- **방향:** 호스트 → 임베디드 장바구니
- **유형:** 응답
- **결과 페이로드:**
  - `ucp` (object, **필수(REQUIRED)**): UCP 프로토콜 메타데이터. `version`은 협상된 `ep_version`을 확인하며, `status`는 반드시 `"success"`여야 합니다(**MUST**).
  - `upgrade` (object, **선택(OPTIONAL)**): 임베디드 장바구니가 호스트와 주고받는 통신 채널을 어떻게 갱신할지 설명하는 객체. 이 필드가 있으면 호스트는 `credential`을 포함해서는 안 됩니다(**MUST NOT**). 채널을 다시 수립하므로 여기서 보낸 자격 증명은 버려집니다.
  - `credential` (string, **선택(OPTIONAL)**): 요청된 인가 데이터. OAuth 토큰, JWT, API 키 등의 형태일 수 있습니다. 요청에 `auth`가 있으면 반드시 설정해야 합니다(**MUST**). `upgrade`가 있으면 설정해서는 안 됩니다(**MUST NOT**).

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "ready_1",
    "result": {
        "ucp": { "version": "2026-04-08", "status": "success" },
        "credential": "fake_identity_linking_oauth_token"
    }
}
```

호스트는 호스트와 임베디드 장바구니 사이의 통신 채널을 갱신하려고 `upgrade` 필드로 응답할 수 있습니다(**MAY**). 현재 이 객체는 `port` 필드만 지원합니다. 이 필드는 반드시 `MessagePort` 객체여야 하고(**MUST**), 반드시 임베디드 장바구니 컨텍스트로 전달(transfer)해야 합니다(**MUST**)(예: 호스트의 `iframe.contentWindow.postMessage()` 호출에서 `{transfer: [port2]}` 사용).

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "id": "ready_1",
    "result": {
        "ucp": { "version": "2026-04-08", "status": "success" },
        "upgrade": {
            "port": "[Transferable MessagePort]"
        }
    }
}
```

호스트가 `upgrade` 객체로 응답하면 임베디드 장바구니는 메시지의 다른 정보를 모두 버리고, 갱신된 통신 채널로 새 `ep.cart.ready` 메시지를 보낸 뒤 새 응답을 기다려야 합니다(**MUST**). 이후 모든 메시지는 반드시 갱신된 통신 채널로만 보내야 합니다(**MUST**).

호스트가 핸드셰이크를 완료할 수 없으면(예: 출처(origin) 검증 실패, 프로토콜 상태 위반) 반드시 `error_response` 결과로 응답해야 합니다(**MUST**). 호스트가 오류로 응답하면 세션은 더 진행할 수 없습니다. 호스트는 반드시 임베디드 컨텍스트를 해제해야 하며(**MUST**), `continue_url`이 있으면 구매자를 그곳으로 리디렉션할 수 있습니다(**MAY**). 임베디드 장바구니는 핸드셰이크 오류를 받은 뒤 더 이상 메시지를 보내서는 안 됩니다(**MUST NOT**).

### 인증

#### `ep.cart.auth`

`ep.cart.auth`는 공용 EP 인증 패턴을 구현합니다. 요청/응답 계약, 예시, 오류 에스컬레이션 흐름은 [임베디드 프로토콜 — 인증](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/embedded-protocol/#authentication)을 참고하세요.

- **메서드:** `ep.cart.auth`
- **방향:** 임베디드 장바구니 → 호스트(요청), 호스트 → 임베디드 장바구니(응답)

오류 에스컬레이션이 필요하면 임베디드 장바구니는 [세션 오류 패턴](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/embedded-protocol/#session-error)에 따라 반드시 `ep.cart.error` 알림을 보내야 합니다(**MUST**).

### 수명 주기 메시지

수명 주기 알림은 공용 EP 패턴을 따릅니다. [임베디드 프로토콜 — 수명 주기](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/embedded-protocol/#lifecycle)를 참고하세요. 모든 수명 주기 알림은 전체 `cart` 객체를 페이로드로 담습니다.

#### `ep.cart.start`

장바구니가 화면에 표시되어 조작할 준비가 되었음을 알립니다. `ep.cart.ready` 핸드셰이크가 성공한 뒤에 보냅니다.

- **방향:** 임베디드 장바구니 → 호스트
- **유형:** 알림
- **페이로드:**
  - `cart` (object, **필수(REQUIRED)**): 장바구니의 현재 전체 상태.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ep.cart.start",
    "params": {
        "cart": {
            "id": "cart_123",
            "currency": "USD",
            "totals": [/* ... */],
            "line_items": [/* ... */],
            "buyer": {/* ... */},
            // ...other cart fields...
        }
    }
}
```

#### `ep.cart.complete`

장바구니 구성이 끝나 구매자가 구매 여정의 다음 단계로 넘어갈 준비가 되었음을 알립니다.

이 메시지로 임베디드 장바구니가 완료됩니다. 서비스 디스커버리에서 협상된 기능에 `dev.ucp.shopping.checkout`이 있으면 호스트는 [체크아웃 생성](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout/#create-checkout) 작업을 호출해 완료된 장바구니로 체크아웃 세션을 시작할 수 있습니다(**MAY**).

- **방향:** 임베디드 장바구니 → 호스트
- **유형:** 알림
- **페이로드:**
  - `cart` (object, **필수(REQUIRED)**): 장바구니의 최종 상태.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ep.cart.complete",
    "params": {
        "cart": {
            "id": "cart_123",
            "currency": "USD",
            "totals": [/* ... */],
            "line_items": [/* ... */],
            "buyer": {/* ... */},
            // ...other cart fields...
        }
    }
}
```

### 상태 변경 메시지

상태 변경 알림은 공용 EP 패턴을 따릅니다. [임베디드 프로토콜 — 상태 변경](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/embedded-protocol/#state-change)을 참고하세요. 모든 상태 변경 알림은 임베디드 장바구니에서 호스트로 보내며, 전체 `cart` 객체를 페이로드로 담습니다.

#### `ep.cart.line_items.change`

라인 아이템이 수정되었습니다(수량 변경, 항목 추가/삭제).

- **방향:** 임베디드 장바구니 → 호스트
- **유형:** 알림
- **페이로드:**
  - `cart` (object, **필수(REQUIRED)**): 장바구니의 현재 전체 상태.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ep.cart.line_items.change",
    "params": {
        "cart": {
            "id": "cart_123",
            // The entire cart object is provided, including the updated line items and estimated totals
            "totals": [
                /* ... */
            ],
            "line_items": [
                /* ... */
            ]
            // ...
        }
    }
}
```

#### `ep.cart.buyer.change`

구매자 정보(이메일, 전화번호, 이름)가 갱신되었습니다.

- **방향:** 임베디드 장바구니 → 호스트
- **유형:** 알림
- **페이로드:**
  - `cart` (object, **필수(REQUIRED)**): 장바구니의 현재 전체 상태.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ep.cart.buyer.change",
    "params": {
        "cart": {
            "id": "cart_123",
            // The entire cart object is provided, including the updated buyer information
            "buyer": {
                /* ... */
            }
            // ...
        }
    }
}
```

#### `ep.cart.messages.change`

장바구니 메시지가 갱신되었습니다. 메시지에는 장바구니 상태에 관한 오류, 경고, 정보성 알림이 담깁니다.

- **방향:** 임베디드 장바구니 → 호스트
- **유형:** 알림
- **페이로드:**
  - `cart` (object, **필수(REQUIRED)**): 장바구니의 현재 전체 상태.

**메시지 예시:**

```json
{
    "jsonrpc": "2.0",
    "method": "ep.cart.messages.change",
    "params": {
        "cart": {
            "id": "cart_123",
            // The entire cart object is provided, including any updated messages
            "messages": [
                {
                    "type": "error",
                    "code": "invalid_quantity",
                    "path": "$.line_items[0].quantity",
                    "content": "Quantity must be at least 1",
                    "severity": "recoverable"
                }
            ]
            // ...
        }
    }
}
```

### 세션 오류 메시지

#### `ep.cart.error`

`ep.cart.error`는 공용 EP 세션 오류 패턴을 구현합니다. 페이로드 명세와 호스트 처리 요구 사항은 [임베디드 프로토콜 — 세션 오류](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/embedded-protocol/#session-error)를 참고하세요.

## 보안과 오류 처리

### 오류 코드

ECaP는 공용 EP 오류 코드 집합을 사용합니다. [임베디드 프로토콜 — 오류 코드](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/embedded-protocol/#error-codes)를 참고하세요.

### 웹 기반 호스트의 보안

ECaP는 CSP, iframe 샌드박싱, credentialless iframe, 엄격한 출처 검증에 관한 공용 EP 보안 요구 사항을 그대로 따릅니다. 전체 명세는 [임베디드 프로토콜 — 보안](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/embedded-protocol/#security)을 참고하세요.

## 스키마 정의

다음 스키마는 임베디드 장바구니 프로토콜에서 쓰는 데이터 구조를 정의합니다.

### 장바구니

라인 아이템, 합계, 구매자 정보를 포함해 장바구니의 현재 상태를 나타내는 핵심 객체입니다.

| 이름         | 타입                                                                                            | 필수   | 설명                                                                                                                                                                                                                                                                                                                                                    |
| ------------ | ----------------------------------------------------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ucp          | any                                                                                             | **예** | 장바구니 응답용 UCP 메타데이터. 체크아웃 전에는 결제 핸들러가 필요 없습니다.                                                                                                                                                                                                                                                                            |
| id           | string                                                                                          | **예** | 고유 장바구니 식별자.                                                                                                                                                                                                                                                                                                                                   |
| line_items   | Array\[[Line Item Response](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#line-item)\] | **예** | 장바구니 라인 아이템. 체크아웃과 구조가 같습니다. 업데이트 시 전체를 교체합니다.                                                                                                                                                                                                                                                                        |
| context      | [Context](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#context)                       | 아니요 | 현지화를 위한 구매자 신호(country, region, postal_code). 판매자는 이를 가격, 재고, 통화를 정하는 데 사용합니다. 생략하면 geo-IP로 대체합니다.                                                                                                                                                                                                           |
| signals      | [Signals](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#signals)                       | 아니요 | 승인과 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터. 값은 구매자가 주장한 내용이어서는 안 됩니다(MUST NOT). 플랫폼은 직접 관찰했거나 독립적으로 검증 가능한 제3자 증명에 근거한 신호를 제공합니다. 여러 확장이 공유 네임스페이스에 값을 추가할 때 출처를 보장하고 충돌을 막기 위해, 모든 신호 키는 반드시 역도메인 이름을 사용해야 합니다(MUST). |
| attribution  | [Attribution](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#attribution)               | 아니요 | 플랫폼이 내보내는 추천 및 전환 이벤트 맥락: 캠페인 식별자, 클릭 ID, source/medium 마커 등. 브라우저 기반 흐름에서 플랫폼이 URL 쿼리 매개변수로 전달하는 것과 같은 매개변수입니다.                                                                                                                                                                       |
| buyer        | [Buyer](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#buyer)                           | 아니요 | 개인화된 견적을 위한 구매자 정보(선택 사항).                                                                                                                                                                                                                                                                                                            |
| currency     | string                                                                                          | **예** | ISO 4217 통화 코드. 판매자가 컨텍스트나 geo-IP를 바탕으로 결정합니다.                                                                                                                                                                                                                                                                                   |
| totals       | [Totals](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#totals)                         | **예** | 예상 비용 내역. 배송비·세금을 아직 계산할 수 없으면 일부만 포함될 수 있습니다.                                                                                                                                                                                                                                                                          |
| messages     | Array\[[Message](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#message)\]              | 아니요 | 검증 메시지, 경고 또는 안내성 알림.                                                                                                                                                                                                                                                                                                                     |
| links        | Array\[[Link](/ucp-ko/2026-04-08/2026-04-08/specification/reference/#link)\]                    | 아니요 | 판매자 링크(선택 사항, 정책, FAQ).                                                                                                                                                                                                                                                                                                                      |
| continue_url | string                                                                                          | 아니요 | 장바구니 핸드오프와 세션 복구용 URL. 공유와 사람이 개입하는(human-in-the-loop) 흐름을 지원합니다.                                                                                                                                                                                                                                                       |
| expires_at   | string                                                                                          | 아니요 | 장바구니 만료 타임스탬프(RFC 3339). 선택 사항.                                                                                                                                                                                                                                                                                                          |
