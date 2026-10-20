UCP가 리테일을 넘어 확장됩니다. 숙박 명세 초안을 지금 확인해 보세요. **푸드도 곧 공개됩니다.**

# Universal Commerce Protocol

플랫폼, 에이전트, 비즈니스를 위한 공통 언어.

UCP는 디스커버리부터 체크아웃, 그 이후 단계까지 산업 전반의 에이전틱 커머스에 필요한 구성 요소를 제공합니다. 덕분에 생태계 전체가 맞춤 개발 없이 하나의 표준으로 움직일 수 있습니다.

### 알아보기

프로토콜 개요, 핵심 개념, 설계 원칙

[시작하기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/index.md)

### 구현하기

GitHub 저장소, 기술 명세, SDK, 레퍼런스 구현

[GitHub에서 보기](https://github.com/Universal-Commerce-Protocol/ucp)

## 업계 선도 기업들이 공동 개발

UCP는 매끄러운 에이전틱 경험을 위해 업계가 함께 만듭니다. 사용자를 불편하게 하고 전환율을 떨어뜨리는 단절된 사용자 여정 문제를 해결합니다.

쇼핑 숙박 푸드

Google

Shopify

Etsy

Wayfair

Target

Walmart

Amazon

Microsoft

Meta

Salesforce

Stripe

Amadeus

Booking.com

Expedia Group

Google

Hilton

Marriott

Trip.com

DoorDash

Google

Square

Toast

Uber Eats

## 유연성, 보안, 확장성을 고려한 설계

에이전틱 커머스에는 상호 운용성이 필요합니다. UCP는 업계 표준 위에 만들어졌습니다. REST와 JSON-RPC 전송 방식을 사용하고, [Agent Payments Protocol (AP2)](https://ap2-protocol.org/), [Agent2Agent (A2A)](https://a2a-protocol.org/latest/), [Model Context Protocol (MCP)](https://modelcontextprotocol.io/docs/getting-started/intro) 지원을 기본으로 내장하고 있어 서로 다른 시스템이 맞춤 통합 없이 함께 동작할 수 있습니다.

### 확장 가능하고 보편적

특정 화면에 얽매이지 않도록 설계해, 소규모부터 엔터프라이즈까지 모든 산업의 비즈니스를 채팅, 비주얼 커머스, 음성 등 어떤 모달리티에서든 지원할 수 있습니다.

### 비즈니스 중심

원활한 커머스를 위해 설계되었습니다. 비즈니스는 통제권을 유지하고 기록상 판매자(Merchant of Record)로 남아 고객 관계를 온전히 소유합니다.

### 개방적이고 확장 가능

설계 단계부터 개방적이고 확장 가능하게 만들어, 산업 전반에서 커뮤니티가 주도해 기능과 확장을 개발할 수 있습니다.

### 안전하고 프라이빗

계정 연결(OAuth 2.0)에도, 결제 위임(Mandate)과 검증 가능한 자격 증명을 활용한 안전한 결제(AP2)에도 검증된 보안 표준을 사용합니다.

### 마찰 없는 결제

제공업체 간에 상호 운용되는 개방형 월렛 생태계로, 구매자는 원하는 결제 수단을 사용할 수 있습니다.

## 실제 동작 살펴보기

UCP는 디스커버리와 검색부터 최종 판매, 구매 후 지원까지 커머스 수명 주기 전체를 매끄럽게 이어 주도록 설계되었습니다. 카탈로그 검색과 조회, 장바구니 구성, 신원 연결, 체크아웃, 주문 관리 등 다양한 핵심 기능을 지원합니다.

쇼핑 숙박 푸드

카탈로그 장바구니 체크아웃 신원 연결 주문

### 카탈로그

플랫폼은 자유 텍스트 검색, 카테고리 필터링, 일괄 조회로 비즈니스의 상품 카탈로그를 검색하고 탐색할 수 있습니다. 이를 바탕으로 가격 비교, 할인가 알림, 재입고·재고 부족 알림 같은 사용자 여정을 만들 수 있습니다.

[시작하기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/catalog/index.md)

```json
{
  "ucp": { ... },
  "products": [
    {
      "id": "prod_abc123",
      "handle": "blue-runner-pro",
      "title": "Blue Runner Pro",
      "description": {
        "plain": "Lightweight running shoes with responsive cushioning."
      },
      "url": "https://business.example.com/products/blue-runner-pro",
      "categories": [
        { "value": "187", "taxonomy": "google_product_category" },
        { "value": "aa-8-1", "taxonomy": "shopify" },
        { "value": "Footwear > Running", "taxonomy": "merchant" }
      ],
      "price_range": {
        "min": { "amount": 12000, "currency": "USD" },
        "max": { "amount": 12000, "currency": "USD" }
      },
      "media": [
        {
          "type": "image",
          "url": "https://cdn.example.com/products/blue-runner-pro.jpg",
          "alt_text": "Blue Runner Pro running shoes"
        }
      ],
      "options": [
        {
          "name": "Size",
          "values": [
            {"label": "8"},
            {"label": "9"},
            {"label": "10"},
            {"label": "11"},
            {"label": "12"}
          ]
        }
      ],
      "variants": [
        {
          "id": "prod_abc123_size10",
          "sku": "BRP-BLU-10",
          "title": "Size 10",
          "description": { "plain": "Size 10 variant" },
          "price": { "amount": 12000, "currency": "USD" },
          "availability": { "available": true },
          "options": [
            { "name": "Size", "label": "10" }
          ],
          "tags": ["running", "road", "neutral"],
          "seller": {
            "name": "Example Store",
            "links": [
              {
                "type": "refund_policy",
                "url": "https://business.example.com/refunds"
              }
            ]
          }
        }
      ],
      "rating": {
        "value": 4.5,
        "scale_max": 5,
        "count": 128
      },
      "metadata": {
        "collection": "Winter 2026",
        "technology": {
          "midsole": "React foam",
          "outsole": "Continental rubber"
        }
      }
    }
  ],
  "pagination": {
    "cursor": "eyJwYWdlIjoxfQ==",
    "has_next_page": true,
    "total_count": 47
  }
}
```

### 장바구니

여러 AI 화면에서 다양한 상품을 담은 장바구니를 구성하고 관리할 수 있습니다. 쇼핑객은 장바구니를 판매자 사이트로 바로 옮기거나, 통합된 다중 상품 체크아웃으로 그 자리에서 구매를 마칠 수 있습니다.

[시작하기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/cart/index.md)

```json
{
  "ucp": { ... },
  "id": "cart_88392A",
  "line_items": [
    {
      "id": "li_1",
      "item": {
        "id": "item_luggage_01",
        "title": "Monos Carry-On Pro Suitcase",
        "price": 26500,
        "image_url": "https://example.com/images/carry-on-pro.jpg"
      },
      "quantity": 1,
      "totals": [ ... ]
    },
    {
      "id": "li_2",
      "item": {
        "id": "item_backpack_02",
        "title": "Monos Metro Backpack",
        "price": 18000,
        "image_url": "https://example.com/images/metro-backpack.jpg"
      },
      "quantity": 1,
      "totals": [ ... ]
    }
  ],
  "context": {
    "address_country": "US",
    "address_region": "CA",
    "postal_code": "94105",
    "intent": "travel gear",
    "language": "en-US",
    "currency": "USD"
  },
  "signals": {
    "dev.ucp.buyer_ip": "198.51.100.42",
    "dev.ucp.user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/128.0.0.0 Safari/537.36"
  },
  "attribution": {
    "source": "google_search",
    "medium": "ai_mode",
    "campaign": "spring_travel_2026",
    "click_id": "gclid_987654321_abc",
    "referrer_url": "https://www.google.com/search"
  },
  "buyer": {
    "first_name": "Elisa",
    "last_name": "Beckett",
    "email": "e.beckett@example.com",
    "phone_number": "+16505550199"
  },
  "currency": "USD",
  "totals": [ ... ],
  "messages": [
    {
      "type": "info",
      "content": "You qualify for free standard delivery on orders over $100."
    }
  ],
  "links": [
    {
      "type": "privacy_policy",
      "url": "https://example.com/policies/privacy",
      "title": "Privacy Policy"
    },
    {
      "type": "terms_of_service",
      "url": "https://example.com/terms",
      "title": "Terms of Service"
    },
    {
      "type": "faq",
      "url": "https://example.com/help/faq",
      "title": "Frequently Asked Questions"
    }
  ],
  "policies": [
    {
      "type": "dev.ucp.shopping.policy.return",
      "description": {
        "plain": "Free returns and exchanges within 30 days of delivery on unused items in original packaging."
      },
      "url": "https://example.com/policies/refunds",
      "applies_to": [
        "$.line_items[0]",
        "$.line_items[1]"
      ]
    }
  ],
  "continue_url": "https://example.com/cart?cart_id=cart_88392A",
  "expires_at": "2026-08-31T12:00:00Z"
}
```

### 체크아웃

통합된 체크아웃 세션 하나로 수백만 비즈니스의 복잡한 장바구니 로직, 동적 가격 책정, 세금 계산 등을 지원합니다.

[시작하기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/index.md)

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
            "type": "shipping_address",
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

### 체크아웃

통합된 체크아웃 세션 하나로 수백만 비즈니스의 복잡한 장바구니 로직, 동적 가격 책정, 세금 계산 등을 지원합니다.

[시작하기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/index.md)

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
            "type": "shipping_address",
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

### 신원 연결

에이전트는 OAuth 2.0 표준을 사용해 자격 증명을 공유하지 않고도 안전하게 인가된 관계를 유지할 수 있습니다.

[시작하기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/common/identity-linking/index.md)

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

### 주문

구매 확인부터 배송까지. 실시간 웹훅으로 모든 채널에서 상태 업데이트, 배송 추적, 반품 처리를 지원합니다.

[시작하기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/order/index.md)

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

### 예약

AI 화면 안에서 완성도 높은 예약 흐름을 구현할 수 있습니다. 객실의 실시간 가격과 예약 가능 여부 확인, 간편한 투숙객 등록, 복잡한 결제 일정까지 지원하는 안전한 체크아웃을 제공합니다.

[시작하기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/lodging/booking/index.md)

```json
{
  "ucp": { ... },
  "id": "bk_fb_88392A",
  "status": "ready_for_complete",
  "booker": {
    "first_name": "Jane",
    "last_name": "Doe",
    "email": "jane.doe@example.com",
    "phone_number": "+14085550199"
  },
  "property": {
    "id": "hotel_juniper_cupt",
    "name": "Juniper Hotel Cupertino",
    "address": {
      "street_address": "123 De Anza Blvd",
      "address_locality": "Cupertino",
      "address_region": "CA",
      "address_country": "US",
      "postal_code": "95014"
    },
    "media": [
      {
        "type": "image",
        "url": "https://example.com/juniper-cupertino.jpg"
      }
    ]
  },
  "stays": [
    {
      "id": "rt_luxury_queen__rp_avg_base_rate",
      "accommodation_type": {
        "id": "rt_luxury_queen",
        "title": "Deluxe Room, 2 Queens",
        "capacity": {
          "adults": 2,
          "total": 2
        }
      },
      "rate_plan": {
        "id": "rp_avg_base_rate",
        "title": "Best Available Rate"
      },
      "occupancy": {
        "adults": 2,
        "total": 2
      },
      "stay_dates": {
        "start_date": "2026-05-12",
        "end_date": "2026-05-17"
      },
      "totals": [ ... ]
    }
  ],
  "currency": "USD",
  "totals": [ ... ],
  "policies": [
    {
      "type": "dev.ucp.lodging.policy.cancellation",
      "refundability": "non_refundable",
      "description": {
        "plain": "Non-refundable rate. If you cancel or modify this reservation, or in the event of a no-show, no refund will be issued and the full booking amount is retained."
      },
      "url": "https://hotel.example.com/terms#non-refundable"
    }
  ],
  "messages": [
    {
      "type": "warning",
      "code": "dev.ucp.lodging.policy.cancellation",
      "path": "$.stays[0]",
      "presentation": "disclosure",
      "content": "This reservation is non-refundable. Cancellations or changes will not be refunded."
    }
  ],
  "payment": {
    "selected_term_id": "pt_deposit_balance",
    "terms": [
      {
        "id": "pt_deposit_balance",
        "title": "Deposit today, balance at check-in",
        "description": {
          "plain": "Pay 2 night deposit ($1200.00) today and the remaining balance upon arrival."
        },
        "schedules": [
          {
            "id": "sched_deposit",
            "type": "immediate",
            "description": {
              "plain": "Due today upon booking confirmation."
            },
            "amount": 120000
          },
          {
            "id": "sched_balance_checkin",
            "type": "deferred",
            "description": {
              "plain": "FlightBulb will charge you the remaining on May 12, 2026."
            },
            "due_at": "2026-05-12T15:00:00-07:00",
            "amount": 8445
          }
        ]
      }
    ],
    "instruments": [
      {
        "id": "pi_card_88392A",
        "handler_id": "handler_1",
        "type": "card",
        "selected": true,
        "display": {
          "brand": "amex",
          "last_digits": "1234",
          "description": "Amex ending in 1234"
        },
        "billing_address": {
          "street_address": "123 De Anza Blvd",
          "address_locality": "Cupertino",
          "address_region": "CA",
          "address_country": "US",
          "postal_code": "95014"
        }
      }
    ]
  },
  "links": [
    {
      "type": "privacy_policy",
      "url": "https://hotel.example.com/privacy"
    },
    {
      "type": "terms_of_service",
      "url": "https://hotel.example.com/terms"
    },
    {
      "type": "refund_policy",
      "title": "Cancellation Policy",
      "url": "https://hotel.example.com/cancellation"
    }
  ],
  "continue_url": "https://business.example.com/booking-sessions/bk_fb_88392A",
  "expires_at": "2026-05-12T06:30:00Z"
}
```

### 푸드

AI 화면의 확장 가능한 체크아웃 경험으로 대화형 음식 주문 여정을 지원합니다. 세밀한 메뉴 맞춤 설정, 실시간 주문 가능 여부와 할인, 팁, 배달 요청 사항까지 매끄럽게 처리합니다.

[자세히 알아보기](https://developers.google.com/actions-center/verticals/ordering/ucp)

Detailed specifications coming soon

### 네이티브 체크아웃 구현

판매자의 체크아웃 API와 직접 통합하고 협상해, 플랫폼에 맞는 네이티브 UI와 워크플로를 구현합니다.

[시작하기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/rest/index.md)

### 비즈니스 체크아웃 임베드

비즈니스의 체크아웃 UI를 임베드하고 렌더링해 복잡한 체크아웃 흐름을 지원합니다. 양방향 통신, 결제 및 배송지 위임 같은 고급 기능을 제공합니다.

[작동 방식 보기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/checkout/embedded/index.md)

## 커머스 생태계 전체를 위한 설계

### 개발자

개방된 기반 위에서 커머스의 미래를 만들어 보세요. 차세대 디지털 커머스를 위해 설계된 오픈소스 표준을 함께 발전시켜 나갈 커뮤니티에 참여하세요.

[기술 명세 보기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/index.md)

### 비즈니스

UCP를 사용하면 채널마다 체크아웃을 새로 만들지 않고도 AI 어시스턴트, 에이전트, 임베디드 경험 등 고객이 있는 곳 어디서든 고객을 만날 수 있습니다. 비즈니스는 계속 기록상 판매자로 남고, 비즈니스 로직도 그대로 유지됩니다.

[UCP와 통합하기](https://developers.google.com/merchant/ucp/)

### AI 플랫폼

표준화된 API로 비즈니스 온보딩을 간소화하고, 사용자에게 통합된 에이전틱 커머스 경험을 제공합니다. MCP, A2A, 기존 에이전트 프레임워크와 호환됩니다.

[UCP 핵심 개념 자세히 알아보기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/documentation/core-concepts/index.md)

### 결제 제공업체

증명 가능한 범용 결제. 모든 승인은 사용자 동의를 입증하는 암호학적 증명으로 뒷받침됩니다. 개방적이고 모듈화된 결제 핸들러 설계로 열린 상호 운용성과 결제 수단 선택권을 제공합니다.

[UCP와 AP2 자세히 알아보기](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/documentation/ucp-and-ap2/index.md)

## 생태계 전반의 지지

Accor

Adore Beauty

Adyen

Affirm

Amadeus

Amex

Ant International

Aven

Best Buy

Block

Booking.com

Bunnings

Carrefour

Checkout.com

Chewy

Choice Hotels

Commerce

DoorDash

Expedia Group

Fiserv

Flipkart

Gap

Hilton

Iconic

IHG

Klarna

Kogan

Kroger

Lowe's

Macy's

Marriott

Mastercard

Paypal

Petbarn

Salesforce

SAP

Sephora

Shopee

Splitit

Square

Stripe

The Home Depot

Toast

Trip.com

Uber Eats

Ulta

Visa

VTEX

Worldpay

Wyndham

Zalando

Accor

Adore Beauty

Adyen

Affirm

Amadeus

Amex

Ant International

Aven

Best Buy

Block

Booking.com

Bunnings

Carrefour

Checkout.com

Chewy

Choice Hotels

Commerce

DoorDash

Expedia Group

Fiserv

Flipkart

Gap

Hilton

Iconic

IHG

Klarna

Kogan

Kroger

Lowe's

Macy's

Marriott

Mastercard

Paypal

Petbarn

Salesforce

SAP

Sephora

Shopee

Splitit

Square

Stripe

The Home Depot

Toast

Trip.com

Uber Eats

Ulta

Visa

VTEX

Worldpay

Wyndham

Zalando

## 지금 시작하기

UCP는 AI 에이전트, 앱, 비즈니스, 결제 제공업체가 연결마다 일회성 맞춤 통합을 만들지 않고도 매끄럽게 상호작용하도록 설계된 개방형 표준입니다. 커머스의 미래를 함께 만들어 갈 여러분의 피드백과 기여를 기다립니다.

전체 기술 명세, 문서, 레퍼런스 구현은 공개 GitHub 저장소에서 확인할 수 있습니다.

### [다운로드](https://github.com/Universal-Commerce-Protocol/samples)

코드 샘플을 다운로드해 실행해 보세요

### [실험](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/shopping/playground/index.md)

프로토콜과 다양한 에이전트 역할을 직접 실험해 보세요

### [기여](https://github.com/Universal-Commerce-Protocol/.github/blob/main/CONTRIBUTING.md)

공개 저장소에 피드백과 코드를 기여해 주세요

[GitHub 저장소 방문하기](https://github.com/Universal-Commerce-Protocol/ucp)
