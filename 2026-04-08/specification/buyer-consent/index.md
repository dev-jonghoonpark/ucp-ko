# 구매자 동의 확장

## 개요

구매자 동의(Buyer Consent) 확장을 사용하면 플랫폼이 데이터 사용과 커뮤니케이션 선호에 관한 구매자의 동의 결정을 비즈니스에 전달할 수 있습니다. 구매자는 분석, 마케팅, 데이터 판매 같은 여러 범주에 대한 동의 상태를 전달할 수 있고, 비즈니스는 이를 바탕으로 CCPA, GDPR 같은 개인정보 보호 규정을 준수할 수 있습니다.

이 확장을 지원하면 체크아웃의 `buyer` 객체가 확장되어, boolean 동의 상태를 담은 `consent` 필드가 추가됩니다.

이 확장은 `create_checkout`과 `update_checkout` 작업에 포함할 수 있습니다.

## 디스커버리

비즈니스는 프로필에서 동의 지원 여부를 알립니다.

```json
{
  "capabilities": {
    "dev.ucp.shopping.buyer_consent": [
      {
        "version": "2026-04-08",
        "extends": "dev.ucp.shopping.checkout"
      }
    ]
  }
}
```

## 스키마 구성

동의 확장은 체크아웃 안의 **buyer 객체**를 확장합니다.

- **확장 대상 기본 스키마**: `buyer` 객체를 통한 `checkout`
- **경로**: `checkout.buyer.consent`
- **스키마 참조**: `buyer_consent.json`

## 스키마 정의

### Consent 객체

데이터 처리에 대한 사용자 동의 상태

| 이름         | 타입    | 필수   | 설명                                 |
| ------------ | ------- | ------ | ------------------------------------ |
| analytics    | boolean | 아니요 | 분석 및 성능 추적에 대한 동의.       |
| preferences  | boolean | 아니요 | 사용자 환경설정 저장에 대한 동의.    |
| marketing    | boolean | 아니요 | 마케팅 커뮤니케이션 수신 동의.       |
| sale_of_data | boolean | 아니요 | 제3자에 대한 데이터 판매 동의(CCPA). |

## 사용법

플랫폼은 체크아웃 작업에서 `buyer` 객체 안에 동의 정보를 담아 보냅니다.

### 예시: 동의 정보를 포함한 체크아웃 생성

```json
POST /checkouts

{
  "line_items": [
    {
      "item": {
        "id": "prod_123",
        "title": "Blue T-Shirt",
        "price": 1999
      },
      "id": "li_1",
      "quantity": 1
    }
  ],
  "buyer": {
    "email": "jane.doe@example.com",
    "first_name": "Jane",
    "last_name": "Doe",
    "consent": {
      "analytics": true,
      "preferences": true,
      "marketing": false,
      "sale_of_data": false
    }
  }
}
```

### 예시: 동의 정보가 포함된 체크아웃 응답

```json
{
  "id": "checkout_456",
  "status": "ready_for_complete",
  "currency": "USD",
  "buyer": {
    "email": "jane.doe@example.com",
    "first_name": "Jane",
    "last_name": "Doe",
    "consent": {
      "analytics": true,
      "preferences": true,
      "marketing": false,
      "sale_of_data": false
    }
  },
  "line_items": [...],
  "totals": [...],
  "links": [
    {
      "type": "privacy_policy",
      "url": "https://example.com/privacy"
    }
  ]
}
```

## 보안 및 개인정보 보호 고려 사항

1. **동의는 선언적입니다** - 프로토콜은 동의를 전달할 뿐 강제하지 않습니다.
1. **법규 준수**는 여전히 비즈니스의 책임입니다.
1. **플랫폼은** 사용자의 명시적인 행동 없이 동의를 **가정하지 않아야 합니다**.
1. 동의 정보가 제공되지 않았을 때의 **기본 동작**은 비즈니스마다 다릅니다.
