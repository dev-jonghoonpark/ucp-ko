# Universal Commerce Protocol

플랫폼, 에이전트, 비즈니스를 위한 공통 언어.

UCP는 디스커버리와 구매부터 구매 이후 경험까지 에이전틱 커머스에 필요한 구성 요소를 정의합니다. 덕분에 생태계 전체가 맞춤 개발 없이 하나의 표준으로 상호 운용할 수 있습니다.

### 알아보기

프로토콜 개요, 핵심 개념, 설계 원칙

[문서 읽기](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/index.md)

### 구현하기

GitHub 저장소, 기술 명세, SDK, 레퍼런스 구현

[GitHub에서 보기](https://github.com/Universal-Commerce-Protocol/ucp)

## 업계 선도 기업들이 공동 개발하고 채택

UCP는 업계가 업계를 위해 만들었습니다. 장바구니 이탈과 쇼핑객의 불만으로 이어지는 단절된 커머스 여정 문제를 해결하고, 에이전틱 커머스를 가능하게 합니다.

Google

Shopify

Etsy

Wayfair

Target

Walmart

## 유연성, 보안, 확장성을 고려한 설계

에이전틱 커머스에는 상호 운용성이 필요합니다. UCP는 업계 표준 위에 만들어졌습니다. REST와 JSON-RPC 전송 방식을 사용하고, [Agent Payments Protocol (AP2)](https://ap2-protocol.org/), [Agent2Agent (A2A)](https://a2a-protocol.org/latest/), [Model Context Protocol (MCP)](https://modelcontextprotocol.io/docs/getting-started/intro) 지원을 기본으로 내장하고 있어 서로 다른 시스템이 맞춤 통합 없이 함께 동작할 수 있습니다.

### 확장 가능하고 보편적

특정 화면에 얽매이지 않도록 설계해, 소규모 비즈니스부터 엔터프라이즈까지 모든 커머스 주체를 채팅, 비주얼 커머스, 음성 등 모든 모달리티에서 지원할 수 있습니다.

### 비즈니스 중심

원활한 커머스를 위해 설계되었습니다. 리테일러는 비즈니스 규칙에 대한 통제권을 유지하고 기록상 판매자(Merchant of Record)로 남아 고객 관계를 온전히 소유합니다.

### 개방적이고 확장 가능

설계 단계부터 개방적이고 확장 가능하게 만들어, 여러 분야에 걸쳐 커뮤니티가 주도해 기능과 확장을 개발할 수 있습니다.

### 안전하고 프라이빗

계정 연결(OAuth 2.0)에도, 결제 위임(Mandate)과 검증 가능한 자격 증명을 활용한 안전한 결제(AP2)에도 검증된 보안 표준을 사용합니다.

### 마찰 없는 결제

제공업체 간에 상호 운용되는 개방형 월렛 생태계로, 구매자는 원하는 결제 수단을 사용할 수 있습니다.

## 실제 동작 살펴보기

UCP는 초기 상품 디스커버리와 검색부터 최종 판매, 구매 후 지원까지 커머스 수명 주기 전체를 원활하게 이어 주도록 설계되었습니다. 프로토콜의 첫 출시는 체크아웃, 신원 연결, 주문 관리라는 세 가지 핵심 기능에 집중합니다.

체크아웃 신원 연결 주문

실제 동작 살펴보기

### 체크아웃

통합된 체크아웃 세션 하나로 수백만 비즈니스의 복잡한 장바구니 로직, 동적 가격 책정, 세금 계산 등을 지원합니다.

[자세히 알아보기](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout-rest/index.md)

```json
{
  "ucp": { ... },
  "id": "chk_123456789",
  "status": "ready_for_complete",
  "currency": "USD",
  "buyer": {
    "email": "e.beckett@example.com",
    "first_name": "Elisa",
    "last_name": "Beckett"
  },
  "line_items": [
    {
      "id": "li_1",
      "item": {
        "id": "item_123",
        "title": "Monos Carry-On Pro suitcase",
        "price": 26550
      },
      "quantity": 1,
      ...
    }
  ],
  "totals": [ ... ],
  "links": [ ... ],
  "payment": { ... },
  "fulfillment": {
    "methods": [
      {
        "id": "method_1",
        "type": "shipping",
        "line_item_ids": ["li_1"],
        "selected_destination_id": "dest_1",
        "destinations": [
          {
            "id": "dest_1",
            "first_name": "Elisa",
            "last_name": "Beckett",
            "street_address": "1600 Amphitheatre Pkwy",
            "address_locality": "Mountain View",
            "address_region": "CA",
            "postal_code": "94043",
            "address_country": "US"
          }
        ],
        "groups": [
          {
            "id": "group_1",
            "line_item_ids": ["li_1"],
            "selected_option_id": "free-shipping",
            "options": [
              {
                "id": "free-shipping",
                "title": "Free Shipping",
                "totals": [ {"type": "total", "amount": 0} ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

실제 동작 살펴보기

### 신원 연결

에이전트는 OAuth 2.0 표준을 사용해 자격 증명을 공유하지 않고도 안전하게 인가된 관계를 유지할 수 있습니다.

[자세히 알아보기](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/identity-linking/index.md)

```json
Sample of /.well-known/oauth-authorization-server

{
  "issuer": "https://example.com",
  "authorization_endpoint": "https://example.com/oauth2/authorize",
  "token_endpoint": "https://example.com/oauth2/token",
  "revocation_endpoint": "https://example.com/oauth2/revoke",
  "scopes_supported": [
    "dev.ucp.shopping.checkout"
  ],
  "response_types_supported": [
    "code"
  ],
  "grant_types_supported": [
    "authorization_code",
    "refresh_token"
  ],
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic"
  ],
  "service_documentation": "https://example.com/docs/oauth2"
}
```

실제 동작 살펴보기

### 주문

구매 확인부터 배송까지. 실시간 웹훅으로 모든 채널에서 상태 업데이트, 배송 추적, 반품 처리를 지원합니다.

[자세히 알아보기](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/order/index.md)

```json
{
  "ucp": { ... },
  "id": "order_123456789",
  "checkout_id": "chk_123456789",
  "permalink_url": ...,
  "line_items": [ ... ],
  "fulfillment": {
    "expectations": [
      {
        "id": "exp_1",
        "line_items": [{ "id": "li_1", "quantity": 1 }],
        "method_type": "shipping",
        "destination": {
          "first_name": "Elisa",
          "last_name": "Beckett",
          "street_address": "1600 Amphitheatre Pkwy",
          "address_locality": "Mountain View",
          "address_region": "CA",
          "postal_code": "94043",
          "address_country": "US"
        },
        "description": "Arrives in 2-3 business days",
        "fulfillable_on": "now"
      }
      ...
    ],
    "events": [
      {
        "id": "evt_1",
        "occurred_at": "2026-01-11T10:30:00Z",
        "type": "delivered",
        "line_items": [{ "id": "li_1", "quantity": 1 }],
        "tracking_number": "123456789",
        "tracking_url": "https://fedex.com/track/123456789",
        "description": "Delivered to front door"
      }
    ]
  },
  "adjustments": [
    {
      "id": "adj_1",
      "type": "refund",
      "occurred_at": "2026-01-12T14:30:00Z",
      "status": "completed",
      "line_items": [{ "id": "li_1", "quantity": -1 }],
      "totals": [{ "type": "total", "amount": -26550 }],
      "description": "Defective item"
    }
  ],
  "totals": [ ... ]
}
```

### 네이티브 체크아웃 구현

판매자의 체크아웃 API와 직접 통합하고 협상해, 플랫폼에 맞는 네이티브 UI와 워크플로를 구현합니다.

[작동 방식 보기](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/checkout-rest/index.md)

### 비즈니스 체크아웃 임베드

비즈니스의 체크아웃 UI를 임베드하고 렌더링해 복잡한 체크아웃 흐름을 지원합니다. 양방향 통신, 결제 및 배송지 위임 같은 고급 기능을 제공합니다.

[자세히 알아보기](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/embedded-checkout/index.md)

## 커머스 생태계 전체를 위한 설계

### 개발자

개방된 기반 위에서 커머스의 미래를 만들어 보세요. 차세대 디지털 커머스를 위해 설계된 오픈소스 표준을 함께 발전시켜 나갈 커뮤니티에 참여하세요.

[기술 명세 보기](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/index.md)

### 비즈니스

UCP를 사용하면 리테일러는 채널마다 체크아웃을 새로 만들지 않고도 AI 어시스턴트, 쇼핑 에이전트, 임베디드 경험 등 고객이 있는 곳 어디서든 고객을 만날 수 있습니다. 비즈니스는 계속 기록상 판매자로 남고, 비즈니스 로직도 그대로 유지됩니다.

[UCP와 통합하기](https://developers.google.com/merchant/ucp/)

### AI 플랫폼

표준화된 API로 비즈니스 온보딩을 간소화하고, 사용자에게 통합된 쇼핑 경험을 제공합니다. MCP, A2A, 기존 에이전트 프레임워크와 호환됩니다.

[UCP 핵심 개념 자세히 알아보기](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/documentation/core-concepts/index.md)

### 결제 제공업체

증명 가능한 범용 결제. 모든 승인은 사용자 동의를 입증하는 암호학적 증명으로 뒷받침됩니다. 개방적이고 모듈화된 결제 핸들러 설계로 열린 상호 운용성과 결제 수단 선택권을 제공합니다.

[UCP와 AP2 자세히 알아보기](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/documentation/ucp-and-ap2/index.md)

## 생태계 전반의 지지

Adyen

Affirm

Amex

Ant International

Best Buy

Block

Carrefour

Checkout.com

Chewy

Commerce

Fiserv

Flipkart

Gap

Klarna

Kroger

Lowe's

Macy's

Mastercard

Paypal

Salesforce

Sephora

Shopee

Splitit

Stripe

The Home Depot

Ulta

Visa

Worldpay

Zalando

Adyen

Affirm

Amex

Ant International

Best Buy

Block

Carrefour

Checkout.com

Chewy

Commerce

Fiserv

Flipkart

Gap

Klarna

Kroger

Lowe's

Macy's

Mastercard

Paypal

Salesforce

Sephora

Shopee

Splitit

Stripe

The Home Depot

Ulta

Visa

Worldpay

Zalando

## 지금 시작하기

UCP는 AI 에이전트, 앱, 비즈니스, 결제 제공업체가 연결마다 일회성 맞춤 통합을 만들지 않고도 매끄럽게 상호작용하도록 설계된 개방형 표준입니다. 커머스의 미래를 함께 만들어 갈 여러분의 피드백과 기여를 기다립니다.

전체 기술 명세, 문서, 레퍼런스 구현은 공개 GitHub 저장소에서 확인할 수 있습니다.

### [다운로드](https://github.com/Universal-Commerce-Protocol/samples)

코드 샘플을 다운로드해 실행해 보세요

### [실험](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/playground/index.md)

프로토콜과 다양한 에이전트 역할을 직접 실험해 보세요

### [기여](https://github.com/Universal-Commerce-Protocol/.github/blob/main/CONTRIBUTING.md)

공개 저장소에 피드백과 코드를 기여해 주세요

[GitHub 저장소 방문하기](https://github.com/Universal-Commerce-Protocol/ucp)
