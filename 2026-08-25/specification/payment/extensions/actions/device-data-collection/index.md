# 결제 기기 데이터 수집 액션

이 명세는 [결제 인증 확장](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/payment/extensions/authentication/index.md)이 선언하는 기기 데이터 수집 액션 유형을 정의합니다.

```text
dev.ucp.common.payment.device_data_collection
```

이 액션은 기기 데이터 수집을 마칠 수 있도록 보이지 않는 결제 인증 화면(surface)을 마운트해 달라고 플랫폼에 요청합니다.

## 목적과 범위

기기 데이터 수집을 이용하면 비즈니스, 결제 핸들러, PSP 또는 3DS 제공자가 결제 처리를 계속하기 전에 기기나 브라우저 데이터를 수집할 수 있습니다. 수집한 데이터는 UCP로 전달되지 않습니다. 로드된 화면이 자체 연동으로 제공자별 데이터를 비즈니스나 결제 제공자에게 직접 보냅니다.

EMV 3DS에서 이 액션은 플랫폼에 수집 화면을 마운트하라고 지시하는 의미만 있습니다. 비즈니스가 ACS 메서드 제공 여부, 메서드 타임아웃, 수집 결과를 `threeDSCompInd` 같은 EMVCo 상태로 어떻게 매핑하는지는 UCP가 정의하지 않습니다. 제공자나 비즈니스가 수집을 건너뛸 때(ACS가 메서드 URL을 제공하지 않는 경우 포함)는 비즈니스가 이 액션을 내보내지 않습니다.

## 차단 액션

비즈니스는 수집을 결제 수단 및 협상된 결제 핸들러와 연결할 수 있게 되면 체크아웃 생성(Create), 수정(Update), 완료(Complete) 응답에서 이 유형을 내보낼 수 있습니다(**MAY**). `config.payment_instrument_id`로 해당 결제 수단을 식별합니다.

권위 있는(authoritative) 체크아웃에 이 액션 항목이 미해결 상태로 남아 있는 동안에는 연결된 결제 시도가 진행되지 않습니다. 화면의 종료 이벤트는 조정(reconciliation)을 시작시킬 뿐, 그 자체로 차단을 풀거나 수집이 성공했다고 확정하지 않습니다. 비즈니스가 기다리지 않을 단순 최적화 목적의 수집이라면 비즈니스는 이 액션을 내보내서는 안 됩니다(**MUST NOT**).

## 런타임 형태

액션은 해당 유형 키 아래에 담겨 내보내집니다.

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

config 형태는 [결제 인증 확장 스키마](/ucp-ko/2026-08-25/2026-08-25/schemas/common/payment_authentication.json)에 인라인으로 정의되어 있습니다.

| 필드                    | 타입   | 필수 | 비고                                              |
| ----------------------- | ------ | ---- | ------------------------------------------------- |
| `payment_instrument_id` | string | ✓    | 이 액션이 담긴 체크아웃에서 연결된 결제 수단의 ID |
| `url`                   | string | ✓    | 보이지 않는 수집 화면의 절대 HTTPS URL            |

`config.url`은 비즈니스나 제공자가 운영해야 하며(**SHOULD**), ACS 메서드 URL로 `threeDSMethodData`를 POST하는 것처럼 제공자별 POST도 직접 처리해야 합니다. 플랫폼은 이 URL로 일반적인 내비게이션만 수행합니다.

## 플랫폼 동작

플랫폼은 반드시 다음을 수행해야 합니다(**MUST**).

1. 결제 인증 계약, 결제 핸들러 및 결제 수단 컨텍스트, 플랫폼 정책에 따라 URL을 검증합니다.
1. 격리되어 있고 보이지 않으며 브라우저 기능을 갖춘 화면에 URL을 로드합니다.
1. 수집을 시작하도록 허용하기 전에 `action.ready` 핸드셰이크를 완료합니다.
1. 임베디드 알림을 이 액션 항목, 그리고 마운트된 화면 양쪽과 연관 짓습니다.
1. `action.done`, `action.error`, 로드 실패, 타임아웃 중 하나가 발생하면 화면을 언마운트합니다.

화면을 마운트할 때는 공통 [결제 인증 렌더링 계약](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/payment/extensions/authentication/#surface-rendering-and-notifications)과 [임베디드 프로토콜 보안 요구 사항](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/embedded-protocol/#security)을 반드시 따라야 합니다(**MUST**).

웹에서는 보통 숨겨진 iframe을 이 화면으로 씁니다. 네이티브 플랫폼은 격리된 웹뷰나 그에 준하는 브라우저 화면을 쓸 수 있습니다. 이 화면은 구매자에게 보여서도, 구매자에게 상호작용을 요청해서도 안 됩니다(**MUST NOT**).

플랫폼은 자기 화면이 끝났는지, 아니면 계속할 수 없었는지만 구분합니다. 기기 데이터를 제대로 수집했는지, ACS 메서드가 타임아웃되었는지, 3DS 인증을 진행해야 하는지는 판단하지 않습니다.

## 임베디드 알림

공통 [`action.ready` 핸드셰이크](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/payment/extensions/authentication/#ready-handshake)를 완료한 뒤, 화면은 [결제 액션 임베디드 계약](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/payment/extensions/authentication/#surface-rendering-and-notifications)에 정의된 JSON-RPC 2.0 알림을 보냅니다.

### 완료

플랫폼 쪽 수집 단계가 끝나면 화면은 반드시 다음을 보내야 합니다(**MUST**).

```json
{
  "jsonrpc": "2.0",
  "method": "action.done",
  "params": {
    "id": "ddc-1"
  }
}
```

이 단계가 언제 끝나는지는 핸들러 쪽 로직이 결정합니다. 이 알림만으로는 수집이 성공했는지, 사용할 수 없었는지, 건너뛰었는지, 특정 제공자의 완료 지표로 매핑되었는지 알 수 없습니다.

### 에러

플랫폼과의 전송 관점에서 화면이 더 이상 계속할 수 없으면 다음을 보낼 수 있습니다(**MAY**).

```json
{
  "jsonrpc": "2.0",
  "method": "action.error",
  "params": {
    "id": "ddc-1",
    "error": {
      "ucp": { "version": "2026-08-25", "status": "error" },
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

화면 수준 코드는 공통 [잘 알려진 `action.error` 코드](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/payment/extensions/authentication/#surface-rendering-and-notifications)를 따릅니다. 이 코드는 EMV 3DS 결과나 결제 결과가 아닙니다. 제공자 도메인에서 발생한 타임아웃이나 사용할 수 없는 메서드처럼 핸들러가 안전하게 넘어갈 수 있는 상황이라면 보통 `action.done`으로 이어져야 하며, 도메인 결과는 비즈니스와 제공자가 서버 측에 기록합니다. 어느 알림을 받든 그 뒤에는 플랫폼이 공통 결제 인증 조정 계약을 따릅니다.

## 마감 시간과 대체 경로

[EMV 3-D Secure Protocol and Core Functions v2.3.1.1](https://www.emvco.com/specifications/emv-3-d-secure-protocol-and-core-functions-specification-6/)에 따르면 3DS Method가 5초 안에 끝나지 않으면 3DS Server가 `threeDSCompInd`를 `N`으로 설정해야 합니다(Req 315). 이 제공자 도메인 타이머는 메서드가 호출될 때 시작되며 여전히 비즈니스와 제공자가 관리합니다. 플랫폼은 바깥 화면의 로드 이벤트를 기준으로 이 타이머를 계산해서는 안 됩니다(**MUST NOT**).

플랫폼은 핸들러나 플랫폼 정책이 정한 상한 안에서 바깥 화면 마감 시간을 적용합니다. 이 마감 시간은 내비게이션 시점부터 재며, 체크아웃의 `expires_at`을 넘지 않습니다. 이 마감 시간은 플랫폼이 UCP 화면을 얼마나 오래 마운트해 둘지만 정할 뿐, `threeDSCompInd`나 다른 제공자 결과를 설정하지 않습니다.

바깥 마감 시간 전에 유효한 `action.done`이나 `action.error`가 오지 않으면 플랫폼은 반드시 화면을 언마운트하고, 상호작용을 타임아웃으로 처리하고, 실패한 화면에 대한 조정 경로를 따라야 합니다(**MUST**). 로드에 실패했거나 유효한 종료 알림을 받았다면 더 일찍 언마운트해도 됩니다. 플랫폼은 같은 액션 유형과 `id`를 다시 마운트해서는 안 됩니다(**MUST NOT**). 그대로 남아 있는 항목은 상위 체크아웃 액션의 대체 경로, 핸드오프, 취소 규칙을 따릅니다.

## 보안

공통 [결제 인증 보안 요구 사항](https://dev-jonghoonpark.github.io/ucp-ko/2026-08-25/2026-08-25/specification/payment/extensions/authentication/#security-and-data-handling)이 적용되며, 추가로 다음을 따릅니다.

- 플랫폼이 초기 출처와 리다이렉트 출처를 로드하려면 먼저 핸들러의 신뢰 정책이 반드시 그 출처를 승인해야 합니다(**MUST**).
- 화면은 플랫폼 자격 증명을 받거나 플랫폼 스토리지에 접근해서는 안 됩니다(**MUST NOT**).
- 제공자 페이로드와 수집한 기기 데이터를 액션, 알림 params, 체크아웃 또는 결제 수단에 복사해서는 안 됩니다(**MUST NOT**).
- `action.done`과 `action.error`는 참고용 화면 신호입니다. 권위 있는 쪽은 여전히 비즈니스/제공자 상태를 바탕으로 한 이후의 체크아웃 응답입니다.
