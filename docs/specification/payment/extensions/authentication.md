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

# 결제 인증 확장 { #payment-authentication-extension }

## 개요 { #overview }

결제 인증 확장은 결제 시도가 진행되는 동안 플랫폼이 처리해야 할 수 있는
브라우저 화면(surface) 상호작용을 정의한다. 이 확장은 두 가지 구체적인
[액션 유형](../../overview/index.md#actions)을 선언한다.

| 액션 유형 | 플랫폼 상호작용 |
| :---------- | :------------------- |
| `dev.ucp.common.payment.device_data_collection` | 브라우저 기능을 갖춘 보이지 않는 기기 데이터 수집 화면을 실행한다. |
| `dev.ucp.common.payment.three_ds_challenge` | 구매자에게 보이는 3DS 챌린지 화면을 표시한다. |

확장 이름은 다음과 같다.

```text
dev.ucp.common.payment.authentication
```

이 확장은 `dev.ucp.shopping.checkout`을 확장한다. 이 확장을 협상하면 두 액션
유형의 전체 계약이 활성화된다. 비즈니스는 체크아웃에 이 확장이 활성화되어 있지
않으면 두 유형 중 어느 것도 내보내서는 안 된다(**MUST NOT**).

이 확장은 플랫폼 측 상호작용만 표준화한다. EMV 3DS를 다시 명세하거나, EMV 3DS
메시지를 전달하거나, 인증 결과를 보고하거나, 결제 핸들러 처리를 대체하지 않는다.
결제사 프로토콜 상태와 권위 있는 결제 결과에 대한 책임은 여전히 비즈니스와 그
결제 핸들러, 결제 대행사에 있다.

## 디스커버리와 협상 { #discovery-and-negotiation }

비즈니스와 플랫폼은 프로필에서 이 확장을 알린다.

<!-- ucp:example schema=profile def=business_schema extract=$.ucp.capabilities target=$.ucp.capabilities -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": {
      "dev.ucp.common.payment.authentication": [
        {
          "version": "{{ ucp_version }}",
          "extends": "dev.ucp.shopping.checkout",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/payment/extensions/authentication",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/common/payment_authentication.json"
        }
      ]
    }
  }
}
```

플랫폼은 보이지 않는 브라우저 화면과 구매자에게 보이는 브라우저 화면을 모두
처리하고, 이 명세의 보안 요구 사항을 지키고, 임베디드 알림을 처리할 수 있을 때만
이 확장을 알린다. 비즈니스는 이 액션을 일으킬 수 있는 모든 결제 핸들러가 필요한
신뢰 및 대체(fallback) 계약을 정의하고 있을 때만 이 확장을 알린다.

확장을 협상했다고 해서 런타임의 모든 URL을 신뢰할 수 있게 되는 것은 아니다. 각
인스턴스는 여전히 자신의 스키마, 결제 시도로 정해진 결제 핸들러와 결제 수단
맥락, 그리고 플랫폼 정책의 적용을 받는다.

## 런타임 형태 { #runtime-shape }

액션은 체크아웃 응답에서 유형을 키로 사용한다. 예를 들어 기기 데이터 수집 단계
다음에 이후 응답에서 챌린지가 이어질 수 있다.

<!-- ucp:example schema=common/payment_authentication def=dev.ucp.shopping.checkout op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "status": "success",
    "capabilities": {
      "dev.ucp.common.payment.authentication": [
        {
          "version": "{{ ucp_version }}",
          "extends": "dev.ucp.shopping.checkout"
        }
      ]
    },
    "payment_handlers": {
      "com.example.card": [
        { "id": "card_handler_1", "version": "{{ ucp_version }}" }
      ]
    }
  },
  "id": "checkout_123",
  "status": "complete_in_progress",
  "currency": "USD",
  "line_items": [],
  "totals": [
    { "type": "subtotal", "amount": 100 },
    { "type": "total", "amount": 100 }
  ],
  "links": [],
  "payment": {
    "instruments": [
      {
        "id": "instrument_1",
        "handler_id": "card_handler_1",
        "type": "card",
        "selected": true
      }
    ]
  },
  "actions": {
    "dev.ucp.common.payment.device_data_collection": [
      {
        "id": "ddc-1",
        "config": {
          "payment_instrument_id": "instrument_1",
          "url": "https://payments.example.com/3ds/method/session_123"
        }
      }
    ]
  }
}
```

수집은 챌린지가 나오기 전에 완료되거나 시간 초과되므로, 비즈니스는 결제 시도마다
한 번에 최대 하나의 결제 인증 액션만 미처리 상태로 두고, 플랫폼이 처리할 수 있는
다음 상호작용만 내보내야 한다(**SHOULD**). 수집이 챌린지보다 먼저 이루어져야
하면, 비즈니스는 기기 데이터 수집 액션을 먼저 내보내고 챌린지는 이후 체크아웃
응답에서만 `dev.ucp.common.payment.three_ds_challenge` 아래에 내보낸다.

`dev.ucp.common.payment.split_payments`가 활성화되어 있으면 체크아웃은 결제
수단마다 하나씩 결제 시도를 가질 수 있고, 각 시도에 미처리 인증 액션이 따로 있을
수 있다. 이들을 구분하는 것이 `config.payment_instrument_id`다.

각 액션의 `config.payment_instrument_id`는 반드시 그 액션을 담은 체크아웃의 결제
수단 하나를 식별해야 한다(**MUST**). 플랫폼은 해당 결제 수단의 `handler_id`를
`ucp.payment_handlers`에서 찾아 그 핸들러의 신뢰 및 대체 규칙을 적용한다. 결제
수단이나 핸들러 연결을 모호함 없이 확정할 수 없는 액션은 플랫폼이 반드시 거절해야
한다(**MUST**).

## 체크아웃 수명 주기 { #checkout-lifecycle }

이 액션 유형들은 [체크아웃 — 액션](../../shopping/checkout/index.md#actions)에
정의된 상위 체크아웃 수명 주기를 따른다.

- 내보낸 모든 결제 인증 액션은 권위 있는 체크아웃에 같은 액션 유형과 `id`가
  미처리 상태로 남아 있는 동안 연결된 결제 시도의 진행을 막는다. 화면의 종료
  이벤트는 플랫폼의 현재 처리 시도를 끝낼 뿐, 그 차단을 풀지 않는다.
- 비즈니스는 결제 완료 전에 기기 데이터 수집을 수행하기 위해 Create 또는 Update
  Checkout에서 이를 내보낼 수 있다(**MAY**). 체크아웃이 `incomplete`인 동안 이
  액션은 해결될 때까지 `ready_for_complete`를 막지만, 관련 없는 Update는 계속
  허용된다.
- 결제 처리 중에 필요한 결제 인증 상호작용을 시작하기 전에 비즈니스는 Complete
  Checkout을 수락하고 액션과 함께 `complete_in_progress`를 반환해야
  한다(**SHOULD**). 이는 수락된 결제 시도가 플랫폼 작업을 기다리고 있음을
  나타낸다.
- 반대로 액션 때문에 Complete Checkout을 수락할 수 없다면, 비즈니스는 반드시
  `status: incomplete`와 함께 `path`가 해당 액션 인스턴스를 정확히 가리키는
  `recoverable` 오류 메시지를 반환해야 한다(**MUST**).
- 체크아웃이 `complete_in_progress`인 동안 플랫폼은 다른 Update나 Complete
  Checkout 작업을 시작해서는 안 된다(**MUST NOT**). 플랫폼은 Get Checkout으로
  비즈니스의 처리 상황을 관찰하고, 상위의 대체·인계·취소 규칙을 따른다.

결제 인증 액션 인스턴스는 한 번만 사용한다. 화면을 시작한 후에는 `action.done`,
`action.error`, 로드 실패, 시간 초과, 구매자의 닫기 이후를 포함해 플랫폼이 같은
액션 유형과 `id`를 다시 처리해서는 안 된다(**MUST NOT**). 비즈니스가 플랫폼에
상호작용을 재시도하게 하려면 반드시 새 `id`를 가진 인스턴스로 교체해야
한다(**MUST**).

화면이 종료 결과에 이르면 플랫폼은 화면을 마운트 해제하고 권위 있는 체크아웃과
상태를 맞춘다.

| 화면 결과 | 상태 맞추기 |
| :-------------- | :------------- |
| `action.done` | Get Checkout을 사용하며, 제한된 백오프로 최대 30초 동안 반복하고 `expires_at`에 이르면 그보다 일찍 멈춘다. |
| `action.error`, 로드 실패, 시간 초과, 구매자의 닫기 | 대체 동작을 적용하기 전에 Get Checkout을 한 번 사용한다. 액션 유형이나 핸들러가 제한된 추가 상태 맞추기를 정의할 수 있다. |

이후 응답에서 액션이 제거되거나, 새 `id`로 교체되거나, 체크아웃이 진행되면 그
응답이 권위를 가진다. 결제사의 최종 결과를 알게 된 비즈니스는 반드시 원래
인스턴스를 제거하거나, 새 `id`를 가진 새 작업으로 교체하거나, 체크아웃을
진행시켜야 한다(**MUST**). 더 이상 처리할 수 있는 작업을 나타내지 않는 인스턴스를
계속 반환해서는 안 된다(**MUST NOT**).

상태 맞추기 기간이 지난 후에도 같은 인스턴스가 남아 있으면, 플랫폼은 그것을 다시
열어서는 안 되며(**MUST NOT**) 상위 체크아웃 액션의 대체·인계·취소 규칙을 따른다.

`action.done` 알림은 플랫폼 측 화면이 끝났다는 뜻일 뿐이다. 수집이나 인증의 성공
여부는 비즈니스가 서버 측 결제사 상태를 보고 판단한다.

## 거절, 실패, 중단 { #decline-failure-and-abandonment }

플랫폼은 자신의 런타임 정책을 위반하는 인스턴스를 거절할 수 있다(**MAY**).
인스턴스를 처리할 수 없다면, 유효한 `continue_url`이 있을 때 이를 사용해 세션을
비즈니스에 넘겨야 한다(**SHOULD**).

`incomplete` 체크아웃의 액션이라면 플랫폼은 대신 일반 Update로 다른 결제 수단을
고른 뒤 나중에 새 Complete 작업을 제출할 수도 있다. Update Checkout이 동결되는
`complete_in_progress` 동안에는 이 방법을 쓸 수 없다. 비즈니스는 이전 결제 수단과
연결된 작업을 대체된 것으로 취급하고 새 결제 시도에는 사용하지 않는다. 대체된
작업은 더 이상 미처리 상태가 아니므로 해당 액션 인스턴스는 이후 체크아웃 응답에
나타나지 않으며, 비즈니스는 체크아웃에 더 이상 없는 결제 수단의 액션을 반드시
제거해야 한다(**MUST**).

유형별 시간 초과와 대체 동작은 개별 액션 명세에서 정의한다.

- [결제 기기 데이터 수집](actions/device-data-collection.md)
- [결제 3DS 챌린지](actions/three-ds-challenge.md)

## 화면 렌더링과 알림 { #surface-rendering-and-notifications }

결제 인증 액션을 반환하는 것은 플랫폼에 해당 `config.url`을 렌더링하라고 요청하는
것이다. 결제 핸들러 명세는 액션 화면을 불러올 출처(origin)를 지정하며, 이것이 그
인스턴스의 **허용 출처**다. 허용 출처는 결제사가 운영할 수도, 비즈니스가 운영할
수도, 둘 다일 수도 있다. 따라서 이 명세는 어느 쪽이 그 신뢰를 갖는지 정하지 않고,
액션을 내보내기 전에 그런 기대가 정해져 있어야 한다는 점만 요구한다. 플랫폼은
`config.url`의 출처가 허용 출처에 없는 인스턴스를 반드시 거절해야 하며(**MUST**),
즉시 완료되는 화면이 초기화와 경합하지 않도록 반드시 탐색 전에 메시지 수신기를
설치해야 한다(**MUST**).

`postMessage`와 네이티브 웹뷰 동작 방식은
[임베디드 프로토콜 — 통신 채널](../../embedded-protocol.md#communication-channels)을
따른다. 결제 인증은 아래의 액션 메서드를 사용한다. 기능별 임베디드 프로토콜
`ready` 메서드나 `MessageChannel` 업그레이드는 사용하지 않으며, `action.ready`가
범위가 한정된 핸드셰이크 역할을 한다.

각 액션 인스턴스는 새로운 격리된 브라우징 컨텍스트를 사용한다. 기기 데이터
수집은 숨겨져 있고 상호작용이 없다. 3DS 챌린지는 화면에 보이고 체크아웃의 나머지
부분과의 상호작용을 막으며, 콘텐츠가 렌더링될 때까지 로딩 상태를 보여 줘야
한다(**SHOULD**). 웹 플랫폼은 iframe이나 핸들러가 승인한 다른 브라우저
컨텍스트를 사용한다. 네이티브 화면에서 플랫폼으로 보내는 메시지를 위해 네이티브
플랫폼은 반드시 `window.EmbeddedActionProtocolConsumer`(Android) 또는
`window.webkit.messageHandlers.EmbeddedActionProtocolConsumer`(iOS)를 통해 JSON
문자열 브리지를 노출해야 한다(**MUST**). 플랫폼에서 화면으로 보내는 ready 응답은
플랫폼이 JSON 문자열로 변환한 JSON-RPC 응답을 인자로
`window.EmbeddedActionProtocol.postMessage()`를 호출해 전달한다.

### Ready 핸드셰이크 { #ready-handshake }

액션별 작업을 시작하기 전에 모든 화면은 반드시 `action.ready` JSON-RPC 요청을
보내야 한다(**MUST**). `params.id`는 미처리 액션 인스턴스를 식별하고, JSON-RPC
봉투의 `id`는 ready 응답과의 대응 관계를 나타낸다. `params.version`은 반드시
그 액션을 담은 체크아웃에서 활성화된 결제 인증 확장 버전과 같아야
한다(**MUST**).

<!-- ucp:example schema=transports/jsonrpc def=request op=read -->
```json
{
  "jsonrpc": "2.0",
  "id": "ready-1",
  "method": "action.ready",
  "params": {
    "id": "ddc-1",
    "version": "{{ ucp_version }}"
  }
}
```

플랫폼은 응답하기 전에 반드시 메시지 출처(source), 허용 출처, 액션 인스턴스,
버전을 검증해야 한다(**MUST**). 출처나 origin 검증에 실패한 메시지는 신뢰할 수
없으므로 응답하지 않는다. 신뢰할 수 있는 화면이 알 수 없는 액션 인스턴스나 활성
버전이 아닌 버전을 지정하면, 플랫폼은 오류 결과를 반환하고 화면을 마운트
해제한 뒤 실패 시의 상태 맞추기 절차를 따른다.

유효한 요청에는 플랫폼이 빈 JSON-RPC 결과를 반환한다. 이 성공 응답은 요청한 결제
인증 확장 버전이 수락되었음을 확인해 준다.

<!-- ucp:example schema=transports/jsonrpc def=success_response op=read -->
```json
{
  "jsonrpc": "2.0",
  "id": "ready-1",
  "result": {}
}
```

화면은 기기 데이터 수집을 시작하거나, 챌린지를 표시하거나, `action.done` 또는
`action.error`를 보내기 전에 반드시 이 성공 응답을 기다려야 한다(**MUST**). 버전은
해당 액션 인스턴스가 유지되는 동안 고정된다.

핸드셰이크 후 화면은
[결제 액션 OpenRPC](site:services/payment-actions/embedded.openrpc.json)에 정의된
`action.done` 또는 `action.error` 알림을 보낸다. 웹에서 플랫폼은 반드시
`event.source`가 마운트된 화면과 일치하는지, `event.origin`이 인스턴스의 허용
출처에 속하는지 검증해야 한다(**MUST**). 유효한 종료 알림을 받은 뒤에는 화면을
마운트 해제하고, 그 컨텍스트에서 오는 중복 알림이나 늦은 알림은 무시한다.

이 확장은 다음과 같은 잘 알려진 `action.error` 코드를 정의한다.

| 코드 | 의미 |
| :--- | :------ |
| `action_unavailable` | 화면을 초기화할 수 없었다. |
| `action_expired` | 화면의 세션이 만료되었다. |
| `action_failed` | 그 밖의 화면 종료 오류. |

이 코드들은 화면 진단용 코드이며, 인증이나 결제 결과를 나타내지 않는다.
[Error Code](site:schemas/common/types/error_code.json)에 따라 이 집합은 열려 있다.
화면은 다른 값을 보낼 수 있으며(**MAY**), 플랫폼은 인식하지 못한 값도 반드시
허용하고 같은 방식으로 상태를 맞춰야 한다(**MUST**).

### 구매자의 닫기 { #buyer-dismissal }

3DS 화면 안에서 이루어진 취소는 화면 오류가 아니라 결제사 결과다. 화면은 비즈니스를
위해 그 결과를 기록한 뒤 `action.done`을 보내야 한다(**SHOULD**). 반면 구매자가
종료 알림 전에 플랫폼 소유의 컨테이너를 닫으면, 플랫폼은 컨테이너를 마운트
해제하고 앞에서 설명한 실패 시의 화면 상태 맞추기 절차를 따른다.

## 보안과 데이터 처리 { #security-and-data-handling }

웹 플랫폼은 CSP, iframe 샌드박싱, credentialless iframe 평가, 엄격한 출처 검증에
관해 공통 [임베디드 프로토콜 보안 요구 사항](../../embedded-protocol.md#security)을
반드시 따라야 한다(**MUST**). [임베디드 체크아웃 보안](../../shopping/checkout/embedded.md#security-for-web-based-hosts)은
UCP 기능이 이 요구 사항을 어떻게 적용하는지 보여 준다. 결제 인증은 자체
`action.ready` 핸드셰이크와 종료 알림 메서드를 사용하며, 임베디드 프로토콜의 기능
수명 주기나 위임 메시지는 채택하지 않는다.

또한 플랫폼은 반드시 다음을 해야 한다(**MUST**).

- `config.url`을 표준을 준수하는 URL 파서로 파싱한다. 이 URL은 반드시 절대
  URL이어야 하고, `https` 스킴을 사용해야 하며, userinfo 구성 요소가 없어야
  한다(**MUST**).
- 화면을 허용 출처 안에 머물게 한다. 웹에서는 허용 출처만 지정한 `frame-src`
  지시어를 사용하면 교차 출처 리디렉션이 신뢰 범위를 조용히 넓히는 대신 로드에
  실패하게 된다.
- 핸들러가 요구하는 프레임 또는 웹뷰 기능만 부여한다. 핸들러는 공통 샌드박스
  기준에서 벗어나는 부분을 반드시 문서화해야 한다(**MUST**).
- 세션 URL을 로그에 남기거나 리퍼러를 통해 유출하지 않는다.
- 모든 완료 알림과 진단 알림을 참고용으로만 취급한다.
