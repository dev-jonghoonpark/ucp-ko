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

# 결제 기기 데이터 수집 액션 { #payment-device-data-collection-action }

이 명세는 [결제 인증 확장](../authentication.md)이 선언하는 기기 데이터 수집 액션
유형을 정의한다.

```text
dev.ucp.common.payment.device_data_collection
```

이 액션은 기기 데이터 수집을 완료하기 위해 보이지 않는 결제 인증 화면(surface)을
마운트하도록 플랫폼에 요청한다.

## 목적과 범위 { #purpose-and-scope }

기기 데이터 수집을 통해 비즈니스, 결제 핸들러, PSP 또는 3DS 제공자는 결제 처리를
계속하기 전에 기기나 브라우저 데이터를 수집할 수 있다. 수집된 데이터는 UCP가 전달하지
않는다. 로드된 화면이 자체 연동을 통해 제공자별 데이터를 비즈니스나 결제 제공자에게
직접 보낸다.

EMV 3DS에서 이 액션은 수집 화면을 마운트하라는, 플랫폼을 향한 지시만을 나타낸다.
비즈니스가 ACS 메서드 제공 여부, 메서드 타임아웃, 수집 결과를 `threeDSCompInd` 같은
EMVCo 상태로 어떻게 매핑하는지는 UCP가 정의하지 않는다. 제공자나 비즈니스가 수집을
건너뛰는 경우(ACS가 메서드 URL을 제공하지 않는 경우 포함)에는 비즈니스가 이 액션을
내보내지 않는다.

## 차단 액션 { #blocking-action }

비즈니스는 수집을 결제 수단 및 협상된 결제 핸들러와 연결할 수 있게 된 뒤, 체크아웃
생성(Create), 수정(Update), 완료(Complete) 응답에서 이 유형을 내보낼 수
있다(**MAY**). `config.payment_instrument_id`가 해당 결제 수단을 식별한다.

이 액션은 권위 있는(authoritative) 체크아웃에 해당 항목이 미해결 상태로 남아 있는
동안, 연결된 결제 시도가 진행되지 못하도록 막는다. 화면의 종료 이벤트는 조정
(reconciliation)을 시작할 뿐, 그 자체로 차단을 해제하거나 수집이 성공했다고 단언하지
않는다. 수집이 비즈니스가 기다리지 않을 단순한 최적화에 불과하다면, 비즈니스는 액션을
내보내서는 안 된다(**MUST NOT**).

## 런타임 형태 { #runtime-shape }

액션은 해당 유형 키 아래에 내보내진다.

<!-- ucp:example schema=common/payment_authentication def=dev.ucp.shopping.checkout extract=$.actions target=$.actions op=read -->
```json
{
  "actions": {
    "dev.ucp.common.payment.device_data_collection": [
      {
        "id": "ddc-1",
        "config": {
          "payment_instrument_id": "instrument_1",
          "url": "https://payments.example.com/ucp/payment/ddc/session_123"
        }
      }
    ]
  }
}
```

config 형태는
[결제 인증 확장 스키마](site:schemas/common/payment_authentication.json)에 인라인으로
정의되어 있다.

| 필드 | 타입 | 필수 | 비고 |
| :---- | :--- | :------- | :---- |
| `payment_instrument_id` | string | ✓ | 이 액션을 담고 있는 체크아웃에서 연결된 결제 수단의 ID. |
| `url` | string | ✓ | 보이지 않는 수집 화면의 절대 HTTPS URL. |

`config.url`은 비즈니스나 제공자가 운영해야 하며(**SHOULD**), ACS 메서드 URL로
`threeDSMethodData`를 POST하는 것처럼 제공자별 POST를 직접 담당해야 한다. 플랫폼은
이 URL로 일반적인 내비게이션을 수행한다.

## 플랫폼 동작 { #platform-behavior }

플랫폼은 반드시 다음을 수행해야 한다(**MUST**).

1. 결제 인증 계약, 결제 핸들러 및 결제 수단 컨텍스트, 플랫폼 정책에 따라 URL을
   검증한다.
2. 격리되고 보이지 않으며 브라우저 기능을 갖춘 화면에 URL을 로드한다.
3. 수집이 시작되도록 허용하기 전에 `action.ready` 핸드셰이크를 완료한다.
4. 임베디드 알림을 이 액션 항목 및 마운트된 화면과 모두 연관 짓는다.
5. `action.done`, `action.error`, 로드 실패 또는 타임아웃 이후 화면을
   언마운트한다.

화면 마운트는 공통
[결제 인증 렌더링 계약](../authentication.md#surface-rendering-and-notifications)과
[임베디드 프로토콜 보안 요구 사항](../../../embedded-protocol.md#security)을 반드시
따라야 한다(**MUST**).

웹에서는 보통 숨겨진 iframe이 이 화면이 된다. 네이티브 플랫폼은 격리된 웹뷰나 그에
준하는 브라우저 화면을 사용할 수 있다. 이 화면은 구매자에게 보이거나 구매자의
상호작용을 요청해서는 안 된다(**MUST NOT**).

플랫폼은 자신의 화면이 끝났는지, 아니면 계속할 수 없었는지만 구분한다. 기기 데이터가
성공적으로 수집되었는지, ACS 메서드가 타임아웃되었는지, 3DS 인증을 진행해야 하는지는
판단하지 않는다.

## 임베디드 알림 { #embedded-notifications }

공통 [`action.ready` 핸드셰이크](../authentication.md#ready-handshake)를 완료한 뒤,
화면은 [결제 액션 임베디드 계약](../authentication.md#surface-rendering-and-notifications)에
정의된 JSON-RPC 2.0 알림을 보낸다.

### 완료 { #done }

플랫폼을 향한 수집 단계가 끝나면 화면은 반드시 다음을 보내야 한다(**MUST**).

<!-- ucp:example schema=transports/jsonrpc def=request op=read -->
```json
{
  "jsonrpc": "2.0",
  "method": "action.done",
  "params": {
    "id": "ddc-1"
  }
}
```

그 단계가 언제 끝나는지는 핸들러가 소유한 로직이 결정한다. 이 알림은 수집이
성공했는지, 사용할 수 없었는지, 건너뛰었는지, 특정 제공자 완료 지표로 매핑되었는지를
드러내지 않는다.

### 에러 { #error }

플랫폼을 향한 전송 관점에서 화면이 더 이상 계속할 수 없으면, 다음을 보낼 수
있다(**MAY**).

<!-- ucp:example schema=transports/jsonrpc def=request op=read -->
```json
{
  "jsonrpc": "2.0",
  "method": "action.error",
  "params": {
    "id": "ddc-1",
    "error": {
      "ucp": { "version": "{{ ucp_version }}", "status": "error" },
      "messages": [
        {
          "type": "error",
          "code": "action_unavailable",
          "content": "Device data collection could not start.",
          "severity": "recoverable"
        }
      ]
    }
  }
}
```

화면 수준 코드는 공통
[잘 알려진 `action.error` 코드](../authentication.md#surface-rendering-and-notifications)를
따른다. 이 코드들은 EMV 3DS 결과나 결제 결과가 아니다. 핸들러가 안전하게 넘어갈 수 있는
제공자 도메인의 타임아웃이나 사용할 수 없는 메서드는 보통 `action.done`으로 이어져야
하며, 도메인 결과는 비즈니스와 제공자가 서버 측에 기록한다. 어느 알림이든 받은 뒤에는
플랫폼이 공통 결제 인증 조정 계약을 따른다.

## 마감 시간과 대체 경로 { #deadline-and-fallback }

[EMV 3-D Secure Protocol and Core Functions v2.3.1.1](https://www.emvco.com/specifications/emv-3-d-secure-protocol-and-core-functions-specification-6/){ target="_blank" }은
3DS Method가 5초 안에 완료되지 않으면 3DS Server가 `threeDSCompInd`를 `N`으로
설정하도록 요구한다(Req 315). 이 제공자 도메인 타이머는 메서드가 호출될 때 시작되며
계속 비즈니스와 제공자가 소유한다. 플랫폼은 이 타이머를 바깥 화면의 로드 이벤트에서
도출해서는 안 된다(**MUST NOT**).

플랫폼은 핸들러나 플랫폼 정책이 정한, 상한이 있는 바깥 화면 마감 시간을 적용한다. 이
마감 시간은 내비게이션 시점부터 측정하며 체크아웃의 `expires_at`을 넘지 않는다. 이
마감 시간은 플랫폼이 UCP 화면을 얼마나 오래 마운트해 둘지만 제어할 뿐,
`threeDSCompInd`나 다른 제공자 결과를 설정하지 않는다.

바깥 마감 시간 전에 유효한 `action.done`이나 `action.error`가 도착하지 않으면,
플랫폼은 반드시 화면을 언마운트하고, 상호작용을 타임아웃으로 처리하고, 성공하지 못한
화면에 대한 조정 경로를 따라야 한다(**MUST**). 로드 실패나 유효한 종료 알림 이후에는
더 일찍 언마운트할 수 있다. 플랫폼은 같은 액션 유형과 `id`를 다시 마운트해서는 안
된다(**MUST NOT**). 변경되지 않은 항목은 상위 체크아웃 액션의 대체 경로, 핸드오프,
취소 규칙을 따른다.

## 보안 { #security }

공통 [결제 인증 보안 요구 사항](../authentication.md#security-and-data-handling)이
적용된다. 추가로 다음을 따른다.

- 핸들러의 신뢰 정책은 플랫폼이 초기 출처와 리다이렉트 출처를 로드하기 전에 반드시
  이를 승인해야 한다(**MUST**).
- 화면은 플랫폼 자격 증명을 받거나 플랫폼 스토리지에 접근해서는 안 된다(**MUST NOT**).
- 제공자 페이로드와 수집된 기기 데이터를 액션, 알림 params, 체크아웃 또는 결제 수단에
  복사해서는 안 된다(**MUST NOT**).
- `action.done`과 `action.error`는 참고용 화면 신호다. 비즈니스/제공자 상태에 기반한
  이후의 체크아웃 응답이 여전히 권위를 가진다.
