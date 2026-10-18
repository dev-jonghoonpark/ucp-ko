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

# 체크아웃 기능 - A2A 바인딩 { #checkout-capability-a2a-binding }

이 문서는 [체크아웃 기능(Checkout Capability)](index.md)의
Agent2Agent Protocol(A2A) 바인딩을 정의합니다.

## 전송 방식 디스커버리 { #transport-discovery }

A2A 전송 방식을 지원하는 비즈니스는 `/.well-known/ucp`의 UCP 프로필 `services`에
에이전트 카드 엔드포인트를 명시해야 합니다. 그래야 A2A를 지원하는 플랫폼이
A2A 프로토콜로 비즈니스 서비스와 상호작용할 수 있습니다.

<!-- ucp:example schema=profile def=business_schema extract=$.ucp.services target=$.ucp.services -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview",
          "transport": "a2a",
          "endpoint": "https://example-business.com/.well-known/agent-card.json"
        }
      ]
    }
  }
}
```

## 쇼핑 에이전트 프로필 알림 { #shopping-agent-profile-advertisement }

비즈니스 에이전트와 상호작용하는 쇼핑 플랫폼은 모든 요청에 자신의 프로필 URI를
`UCP-Agent` 요청 헤더로 보내야 합니다.

```http
UCP-Agent: profile="https://agent.example/profiles/v2025-11/shopping-agent.json"
Content-Type: application/json
```

### 헤더 매핑 참조 { #header-mapping-reference }

다음 표에 A2A 에이전트가 플랫폼과 UCP 데이터 타입을 주고받는 데 필요한 헤더를
정리했습니다.

| 헤더 이름          | 설명                                       |
| :----------------- | :----------------------------------------- |
| `UCP-Agent`        | 쇼핑 플랫폼 애플리케이션 프로필 URI |
| `X-A2A-Extensions` | UCP 확장 URI(아래 참고)                    |

## A2A 상호작용 { #a2a-interactions }

A2A 프로토콜은 에이전트 간 통신을 위한 탄탄한 기반을 제공합니다.
[A2A 확장](https://a2a-protocol.org/latest/topics/extensions/)을 사용하면 에이전트끼리
구조화된 데이터 타입으로 통신할 수 있습니다. 덕분에 비즈니스는 UCP 데이터 타입으로
플랫폼과 통신하는 AI 애플리케이션을 만들 수 있습니다.

UCP A2A 확장의 URI:
`https://ucp.dev/{{ ucp_version }}/specification/reference`

UCP를 지원하는 비즈니스는 플랫폼이 확장을 활성화할 수 있도록 A2A 에이전트 카드에
확장과 선택 기능을 선언해야 합니다.

예시:

<!-- ucp:example schema=transports/a2a_message def=agent_card -->
```json
{
  "extensions": [
    {
      "uri": "https://ucp.dev/{{ ucp_version }}/specification/reference",
      "description": "Business agent supporting UCP",
      "params": {
        "capabilities": {
          "dev.ucp.shopping.checkout": [
            {"version": "{{ ucp_version }}"}
          ],
          "dev.ucp.shopping.fulfillment": [
            {
              "version": "{{ ucp_version }}",
              "extends": "dev.ucp.shopping.checkout"
            }
          ]
        }
      }
    }
  ]
}
```

### Agent2Agent 협상 { #agent2agent-negotiation }

비즈니스 에이전트는 A2A `Message` 객체로 쇼핑 에이전트/플랫폼과 상호작용할
수 있습니다. 에이전트가 반환하는 A2A `Message` 객체는 메시지 안의 `DataPart` 객체에
구조화된 데이터를 담습니다. 플랫폼은 현재 컨텍스트를 유지하려면 세션의 이후 턴에서
비즈니스 에이전트가 생성한 `contextId`를 전달해야 합니다.

적용 가능한 시나리오에서는 비즈니스 에이전트가 A2A `Task` 객체를 사용할 수도 있습니다.
이 경우 비즈니스 에이전트는 플랫폼과 상호작용하는 데 필요한 페이로드를 담은 `Task`
객체를 반환합니다. 플랫폼은 작업이 완료될 때까지 이후 턴에서 `contextId`와 함께 서버가
생성한 `taskId`를 전달해야 합니다.

플랫폼은 작업이 종료 상태에 이른 뒤에도 같은 세션에서 추가 협상을 처리할 수 있어야
합니다(예: 사용자가 주문한 뒤 같은 컨텍스트에서 다시 주문하려는 경우, 또는 예외 때문에
작업이 실패 상태가 된 경우). 작업이 종료 상태에 이르면 플랫폼은 에이전트와 계속
상호작용할 수 있도록 `taskId`를 초기화해야 합니다. 다만 현재 `contextId`는 이후
상호작용에서 재사용할 수 있습니다.

## 요청 멱등성 { #request-idempotency }

비즈니스 에이전트는 A2A `Message`에 포함되어 전송되는 `messageId`로 플랫폼
재시도 때문에 생긴 중복 메시지를 감지해야 합니다.

## 체크아웃 기능 동작 { #checkout-functionality }

체크아웃 기능을 사용하면 소비자가 체크아웃 세션의 상품을 관리하고 구매 절차를 완료할
수 있습니다. 비즈니스 에이전트는 보통 비즈니스의 체크아웃 API와 연동해 이 기능을 제공합니다.

이 확장은 비즈니스 에이전트가 체크아웃 관련 동작, 체크아웃 완료나 취소를 처리할 때
체크아웃 기능을 표현하는 데이터 스키마를 정의합니다. `Checkout` 엔티티는 A2A `Message`의
프로필입니다. 비즈니스 에이전트는 UCP-A2A 확장을 활성화한 플랫폼에 A2A `Message`의
`DataPart`로 Checkout 엔티티를 반환해야 합니다. 체크아웃 객체는 반드시
`a2a.ucp.checkout` 키가 있는 `DataPart` 객체의 일부로 반환해야 합니다(**MUST**).

**요청 형식:** 에이전트형 애플리케이션은 에이전트와 상호작용하는 사용자로부터 자연어
입력을 받아 사용자의 의도를 파악하고, 필요한 정보를 얻기 위해 사용자와 협상한 다음,
적절한 도구를 호출해 작업을 수행할 수 있습니다. 플랫폼의 입력은 A2A `Message`로 원격
비즈니스 에이전트에 보낼 수 있습니다.

예시:

- 자연어 입력

<!-- ucp:example schema=transports/a2a_message def=message_request direction=request -->
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "type": "text",
          "text": "add Pixel 10 Pro to my checkout"
        }
      ],
      "messageId": "69da8f87-991b-479e-80dc-ed92fcb57cbe",
      "kind": "message",
      "contextId": "aad14abc-4082-4748-84ca-4afff85aedfa"
    }
  }
}
```

- 사용자 동작에 따른 구조화된 입력

<!-- ucp:example schema=transports/a2a_message def=message_request direction=request -->
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "type": "data",
          "data": {
            "action": "add_to_checkout",
            "product_id": "PIXEL-10-PRO",
            "quantity": 1
          }
        }
      ],
      "messageId": "e94a8c10-69f4-4c4c-b988-21a298302da6",
      "kind": "message",
      "contextId": "aad14abc-4082-4748-84ca-4afff85aedfa"
    }
  }
}
```

**응답 형식:** 다음은 체크아웃 기능을 구현한 비즈니스 에이전트의 응답 예시입니다.

<!-- ucp:example schema=transports/a2a_message def=message_response -->
```json
{
  "id": 33,
  "jsonrpc": "2.0",
  "result": {
    "contextId": "4629ea79-7201-4ece-bc7a-ce19fff76e61",
    "kind": "message",
    "messageId": "8e8566e0-6d7c-4f29-bd90-26a132385baa",
    "parts": [
      {
        "data": {
          "a2a.ucp.checkout": { ... }
        },
        "kind": "data"
      }
    ],
    "role": "agent"
  }
}
```

### 체크아웃 완료 { #checkout-completion }

사용자가 결제할 준비가 되면 체크아웃 절차를 완료하기 위해 `payment`를 비즈니스
에이전트에 제출해야 합니다. `payment`는 UCP에 정의된 구조화된 데이터 타입입니다. 체크아웃을
완료하려고 결제를 처리할 때는 `payment`를 속성 이름이 `a2a.ucp.checkout.payment`인
`DataPart`로 비즈니스 에이전트에 제출해야 합니다. 관련 신호는 속성 이름
`a2a.ucp.checkout.signals`로 보내야 합니다.

체크아웃 절차가 완료되면 비즈니스 에이전트는 `id`와 `permalink_url`을 담은 `order`
속성이 포함된 체크아웃 객체를 반환해야 합니다.

### 요청 형식 { #request-format }

<!-- ucp:example schema=transports/a2a_message def=message_request direction=request -->
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "type": "data",
          "data": {"action":"complete_checkout"}
        },
        {
          "kind": "data",
          "data": {
            "a2a.ucp.checkout.payment": { ... },
            "a2a.ucp.checkout.signals": {
              "dev.ucp.buyer_ip": "203.0.113.42",
              "dev.ucp.user_agent": "Mozilla/5.0 ..."
            }
          }
        }
      ],
      "messageId": "e94a8c10-69f4-4c4c-b988-21a298302da6",
      "kind": "message",
      "contextId": "aad14abc-4082-4748-84ca-4afff85aedfa"
    }
  }
}

```

**응답 형식:** 다음은 체크아웃 기능을 구현한 비즈니스 에이전트의 응답 예시입니다.

<!-- ucp:example schema=transports/a2a_message def=message_response -->
```json
{
  "id": 33,
  "jsonrpc": "2.0",
  "result": {
    "contextId": "4629ea79-7201-4ece-bc7a-ce19fff76e61",
    "kind": "message",
    "messageId": "8e8566e0-6d7c-4f29-bd90-26a132385baa",
    "parts": [
      {
        "data": {
          "a2a.ucp.checkout": { ... }
        },
        "kind": "data"
      }
    ],
    "role": "agent"
  }
}

```

#### AP2 기반 체크아웃 완료 { #ap2-based-checkout-completion }

비즈니스 에이전트는 에이전트 간 결제 상호작용에서 사용자 의도와 승인을 안전하게
교환할 수 있게 해 주는 AP2 위임(Mandate) 확장을 구현할 수 있습니다. UCP용 AP2 위임 확장을
지원하는 비즈니스는 UCP 디스커버리 문서와 A2A 에이전트 카드에 이를 명시해야 합니다.
플랫폼과 비즈니스 에이전트가 각자의 프로필에 AP2 위임 확장을 선언하면 이 확장은
암묵적으로 활성화된 것으로 간주합니다.

AP2 위임 확장이 활성화되면 비즈니스 에이전트는 체크아웃 객체의 detached JWS를
생성하고, 생성된 서명을 `DataPart`의 `ap2.merchant_authorization`으로 반환해야 합니다.
그러면 플랫폼은 비즈니스의 공개 키로 체크아웃 페이로드를 암호학적으로 검증할 수 있습니다.

<!-- ucp:example schema=transports/a2a_message def=message_response -->
```json
{
  "id": 33,
  "jsonrpc": "2.0",
  "result": {
    "contextId": "4629ea79-7201-4ece-bc7a-ce19fff76e61",
    "kind": "message",
    "messageId": "8e8566e0-6d7c-4f29-bd90-26a132385baa",
    "parts": [
      {
        "data": {
          "a2a.ucp.checkout": {
            "ap2": {
              "merchant_authorization": "<detached jws signature>"
            }
          }
        },
        "kind": "data"
      }
    ],
    "role": "agent"
  }
}
```

사용자가 플랫폼에서 결제를 확인하면, 체크아웃을 완료하기 위해 사용자가 서명한
체크아웃 위임 객체와 결제 위임 객체를 `DataPart`로 비즈니스 에이전트에 보내야 합니다.
결제 위임을 포함하는 `payment`는 속성 이름이 `a2a.ucp.checkout.payment`인 `DataPart`의
일부로 제출해야 합니다. 서명된 체크아웃 위임은 `DataPart`에 `ap2.checkout_mandate`로
지정해야 합니다. 결제 위임은 `payment`의 `token` 속성에 담깁니다. 체크아웃 완료를 위한
위임 검증과 처리에 관한 자세한 내용은
[AP2 위임 확장](../../payment/extensions/ap2-mandates.md) 문서를 참고하세요.

#### 요청 형식 { #request-format_1 }

<!-- ucp:example schema=transports/a2a_message def=message_request direction=request -->
```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "type": "data",
          "data": {
            "action": "complete_checkout"
          }
        },
        {
          "kind": "data",
          "data": {
            "a2a.ucp.checkout.payment": {
              "id": "instr_1",
              "handler_id": "gpay",
              "type": "card",
              "description": "Visa •••• 1234",
              "billing_address": {
                "street_address": "123 Main St",
                "address_locality": "Anytown",
                "address_region": "CA",
                "address_country": "US",
                "postal_code": "12345"
              },
              "credential": {
                "type": "PAYMENT_GATEWAY",
                "token": "examplePaymentMethodToken"
              }
            },
            "ap2": {
              "checkout_mandate": "eyJhbGciOiJFUz..."
            }
          }
        }
      ],
      "messageId": "e94a8c10-69f4-4c4c-b988-21a298302da6",
      "kind": "message",
      "contextId": "aad14abc-4082-4748-84ca-4afff85aedfa"
    }
  }
}
```
