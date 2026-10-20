# 결제 3DS 챌린지 액션

이 명세는 [결제 인증 확장](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/payment/extensions/authentication/index.md)이 선언하는 3DS 챌린지 액션 (Action) 타입을 정의합니다.

```text
dev.ucp.common.payment.three_ds_challenge
```

이 액션은 구매자가 3DS 챌린지를 완료할 수 있도록, 구매자에게 보이는 결제 인증 화면(surface)을 표시해 달라고 플랫폼에 요청합니다.

## 목적과 범위

비즈니스와 결제 핸들러는 3DS 챌린지 액션으로 발급사(issuer)나 제공업체의 챌린지를 구매자에게 표시해 달라고 플랫폼에 요청할 수 있습니다. UCP는 CReq/CRes, AReq/ARes, RReq/RRes, `transStatus` 같은 EMV 3DS 페이로드를 전달하지 않습니다. 이런 세부 정보는 로드된 화면, 그리고 비즈니스·결제 핸들러·PSP·3DS 서버·발급사·ACS 연동 내부에서만 다룹니다.

플랫폼은 화면을 표시하고, 플랫폼 쪽 상호작용이 끝났는지 또는 더 진행할 수 없는지만 관찰합니다. 인증과 결제의 결과는 비즈니스가 서버 측 제공업체 상태를 바탕으로 판단합니다.

## 차단 액션

3DS 챌린지 액션은 해당 항목(occurrence)이 해결되지 않은 동안, 연관된 결제 시도로 체크아웃이 완료되지 못하도록 막습니다. 이 액션이 체크아웃 완료(Complete Checkout) 작업이 거부되었다는 뜻은 아닙니다. 또한 화면에서 종료 이벤트가 발생했다고 해서 그것만으로 차단이 풀리지도 않습니다.

비즈니스는 체크아웃 완료 요청을 수락하고 결제 시도를 시작한 뒤, 챌린지 액션과 함께 `complete_in_progress`를 반환해야 합니다(**SHOULD**). 챌린지 전에 완료 요청을 수락할 수 없다면 비즈니스는 핵심 체크아웃 규칙을 따릅니다. 즉, `path`가 챌린지 항목을 가리키는 복구 가능한 오류 메시지와 함께 `incomplete`를 반환합니다.

## 런타임 형태

액션은 타입 키 아래에 담겨 전달됩니다.

```json
{
  "actions": {
    "dev.ucp.common.payment.three_ds_challenge": [
      {
        "id": "three-ds-challenge-1",
        "config": {
          "payment_instrument_id": "instrument_1",
          "url": "https://payments.example.com/ucp/payment/3ds/session_456"
        }
      }
    ]
  }
}
```

config 형태는 [결제 인증 확장 스키마](/ucp-ko/draft/draft/schemas/common/payment_authentication.json)에 인라인으로 정의되어 있습니다.

| 필드                    | 타입   | 필수 | 비고                                           |
| ----------------------- | ------ | ---- | ---------------------------------------------- |
| `payment_instrument_id` | string | ✓    | 이 체크아웃에 포함된 연관 결제 수단의 ID       |
| `url`                   | string | ✓    | 구매자에게 보이는 챌린지 화면의 절대 HTTPS URL |

`config.url`은 비즈니스나 제공업체가 운영해야 하며, ACS 챌린지 URL로 CReq를 제출하는 것 같은 제공업체 고유의 POST는 모두 이 URL에서 처리해야 합니다(**SHOULD**). 플랫폼은 이 URL로 일반적인 내비게이션만 수행합니다. 플랫폼이 제공업체 요청을 구성하거나, 챌린지 페이로드를 파싱하거나, 3DS 프로토콜 데이터를 중계해서는 안 됩니다(**MUST NOT**).

## 플랫폼 동작

플랫폼은 반드시 다음을 수행해야 합니다(**MUST**).

1. 결제 인증 계약, 결제 핸들러와 결제 수단 컨텍스트, 플랫폼 정책에 따라 URL을 검증합니다.
1. 사용자에게 보이고 격리된, 브라우저 기능을 갖춘 화면에 URL을 로드합니다.
1. 챌린지를 표시하기 전에 `action.ready` 핸드셰이크를 완료합니다.
1. 챌린지가 해결되지 않은 동안에는 체크아웃의 나머지 부분과 상호작용하지 못하게 막습니다.
1. 임베디드 알림을 이 액션 항목과 마운트된 화면 양쪽에 대조해 연관시킵니다.
1. `action.done`, `action.error`, 복구할 수 없는 로드 실패, 이탈이 발생하면 화면을 닫습니다.

화면을 마운트할 때는 공통 [결제 인증 렌더링 계약](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/payment/extensions/authentication/#surface-rendering-and-notifications)과 [임베디드 프로토콜 보안 요구 사항](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/embedded-protocol/#security)을 반드시 따라야 합니다(**MUST**).

웹에서는 협상된 핸들러가 해당 표시 방식과 보안 정책을 정의했다면 화면으로 전체 페이지 프레임, 모달 프레임, 별도 창 중 하나를 사용할 수 있습니다. 네이티브 플랫폼은 격리된 웹뷰, 브라우저 뷰, 시스템 브라우저를 사용할 수 있습니다. 최상위(top-level) 또는 시스템 브라우저로 넘기는 방식은, 플랫폼이 인증하고 액션 항목과 연관시킬 수 있는 완료 채널을 핸들러가 정의한 경우에만 허용됩니다.

플랫폼이 프레임 내용, URL 변경, 알림 진단 정보로 인증 결과를 추론해서는 안 됩니다(**MUST NOT**).

## 임베디드 알림

공통 [`action.ready` 핸드셰이크](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/payment/extensions/authentication/#ready-handshake)를 완료하면 화면은 [결제 액션 임베디드 계약](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/payment/extensions/authentication/#surface-rendering-and-notifications)에 정의된 JSON-RPC 2.0 알림을 보냅니다.

### 완료

플랫폼 쪽 챌린지 상호작용이 끝나면 화면은 반드시 다음을 보내야 합니다(**MUST**).

```json
{
  "jsonrpc": "2.0",
  "method": "action.done",
  "params": {
    "id": "three-ds-challenge-1"
  }
}
```

`action.done`은 플랫폼이 챌린지 화면을 닫고 체크아웃을 조회하면 된다는 뜻일 뿐입니다. 인증이나 결제가 성공했다는 뜻은 아닙니다. 플랫폼은 화면을 닫은 뒤 처리 상태를 확인하려면 체크아웃 조회(Get Checkout)를 호출해야 합니다. 체크아웃 조회는 읽기 전용 관찰이며, 호출한다고 해서 결제 시도가 진행되지는 않습니다. 비즈니스는 플랫폼이 체크아웃 조회를 언제 호출하든 상관없이, 서버 측 제공업체 상태(보통 대역 외(out-of-band) 제공업체 푸시/콜백)를 바탕으로 체크아웃 상태를 진행시킵니다.

핸들러가 소유한 화면은 상관관계 식별자나 SDK 결과 같은 제공업체 고유 params나 진단용 params를 포함할 수 있습니다(**MAY**). 플랫폼은 협상된 핸들러가 정의하지 않은 알 수 없는 params를 UCP 처리에서 반드시 무시해야 합니다(**MUST**). 이런 params는 기본적으로 권위 있는 결제 결과로 취급되지 않습니다.

### 오류

화면이 더 진행할 수 없으면 다음을 보낼 수 있습니다(**MAY**).

```json
{
  "jsonrpc": "2.0",
  "method": "action.error",
  "params": {
    "id": "three-ds-challenge-1",
    "error": {
      "ucp": { "version": "draft", "status": "error" },
      "messages": [
        {
          "type": "error",
          "code": "action_failed",
          "content": "The challenge surface could not continue.",
          "severity": "recoverable"
        }
      ]
    }
  }
}
```

화면 수준 코드는 공통 [잘 알려진 `action.error` 코드](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/payment/extensions/authentication/#surface-rendering-and-notifications)를 따릅니다. 화면은 구매자 이탈 여부나 제공업체 인증 결과를 알 수 없을 수도 있고, 이 정보는 제공업체 백엔드만 알고 있을 수도 있습니다. 이런 구분은 비즈니스와 결제 제공업체가 권위를 가지고 판단합니다. 어느 알림을 받든 플랫폼은 이후 공통 결제 인증 조정(reconciliation) 계약을 따릅니다.

## 결합된 제공업체 흐름

결제 제공업체는 하나의 `config.url`에서 기기 데이터 수집과 챌린지를 함께 제공할 수 있습니다. 발행되는 액션 타입은 플랫폼 쪽에서 어떻게 동작할지를 나타냅니다.

- 화면이 끝까지 보이지 않고 구매자 상호작용을 전혀 요청하지 않으면 비즈니스는 기기 데이터 수집 액션을 발행합니다.
- 화면이 구매자 챌린지를 표시할 수 있으면 비즈니스는 3DS 챌린지 액션을 발행합니다.

비즈니스는 보이지 않는 DDC 액션을 발행해 놓고 그 화면을 보이게 만들면 안 됩니다. 수집이 먼저 끝나면, 이후 체크아웃 응답에서 DDC 항목을 제거하고 새 챌린지 항목을 발행하는 흐름을 권장합니다. 두 항목은 `config.url` 오리진이 같더라도 액션 타입과 ID가 서로 다릅니다.

## 기한과 폴백

[EMV 3-D Secure Protocol and Core Functions v2.3.1.1](https://www.emvco.com/specifications/emv-3-d-secure-protocol-and-core-functions-specification-6/) 에 따르면 ACS는 각 챌린지 인터페이스를 챌린지 iframe으로 보내는 데 성공한 뒤 10분(600초)을 쓸 수 있습니다(Req 226). 이 인터페이스별 타이머는 화면과 제공업체가 관리합니다. 이 규정이 최초 내비게이션 시점부터 시작되는 단일 UCP 기한을 정의하는 것은 아닙니다.

플랫폼은 핸들러나 플랫폼 정책이 정한, 상한이 있는 외부 화면 기한을 적용합니다. 이 기한은 내비게이션 시점부터 재며 체크아웃의 `expires_at`을 넘을 수 없습니다. 비즈니스는 핸들러가 정의한 시간 범위를 수용할 수 있도록 만료 시각을 반드시 정해야 합니다(**MUST**). 외부 기한은 플랫폼이 소유한 컨테이너에만 적용되며, ACS 타이머를 대체하거나 재설정하지 않습니다.

플랫폼은 유효한 종료 알림, 로드 실패, 구매자의 닫기, 체크아웃 종료 상태가 발생하면 화면을 더 일찍 닫을 수 있습니다. 외부 기한이 지나면 플랫폼은 반드시 화면을 닫고, 상호작용을 이탈로 간주하고, 성공하지 못한 화면의 조정 경로를 따라야 합니다(**MUST**). 같은 액션 타입과 `id`를 다시 마운트해서는 안 되며(**MUST NOT**), 이후 처리는 상위 체크아웃 액션의 폴백, 핸드오프, 취소 규칙을 따릅니다.

## 보안

공통 [결제 인증 보안 요구 사항](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/payment/extensions/authentication/#security-and-data-handling)이 적용됩니다. 그 밖에 다음을 따릅니다.

- 플랫폼이 최초 오리진과 리디렉션 오리진을 표시하려면 먼저 핸들러의 신뢰 정책이 해당 오리진을 반드시 승인해야 합니다(**MUST**).
- 플랫폼은 메시지 출처(source)와 오리진을 모두 검증합니다.
- 서버 측 제공업체 상태를 바탕으로 한 이후 체크아웃 응답은 화면 알림과 관계없이 권위를 유지합니다.
