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

# UCP와 AP2 { #ucp-and-ap2 }

UCP는
[Agent Payments Protocol(AP2)](https://ap2-protocol.org/){ target="_blank" }와
완벽하게 호환됩니다. AP2는 에이전트가 사용자를 대신해 수행하는 거래의 신뢰
계층입니다. 플랫폼과 비즈니스가 의도와 승인을 안전하고 검증 가능한 방식으로
주고받도록 강제합니다. AP2는 검증 가능한 디지털 자격 증명(Verifiable Digital
Credentials, VDC)을 사용하므로 중간에서 신뢰를 보증해 줄 "심판"이 필요하지
않습니다. 비즈니스는 서명된 체크아웃 약정을 받으므로 흐름 도중에 가격과 조건이
바뀌지 않는다는 것을 보장받습니다. 플랫폼은 장바구니의 특정 상태에 수학적으로
묶인, 암호학적으로 서명된 결제 승인을 제공할 수 있습니다.

## 주요 이점 { #key-benefits }

* **구속력 있는 증명:** 무엇을 제안했고 무엇에 합의했는지에 대한 암호학적
    증거를 양측이 모두 보유하므로 거래의 최종성과 진정성이 보장됩니다.
* **사기 감소:** 결제 위임(Mandate)은 특정 체크아웃 해시에만 유효하므로
    "토큰 재사용(token replay)"이나 금액 조작을 막을 수 있습니다.
* **에이전트 대응:** 자율 AI 에이전트가 미리 정해 둔 검증 가능한 범위 안에서
    사용자를 대신해 거래할 수 있습니다.

## 프로토콜 흐름 { #protocol-flow }

1. **디스커버리:** 비즈니스가 디스커버리 문서를 게시하고 AP2 확장을 지원한다고
    선언합니다.
2. **세션 활성화:** 플랫폼이 체크아웃 세션을 생성하거나 업데이트할 때 AP2를
    활성화하겠다고 알립니다.
3. **서명(비즈니스):** 비즈니스는 `checkoutSignature`(체크아웃 상태에 서명한
    분리형(detached) JWT)로 응답하고, 지원하는 검증 가능한 프레젠테이션 형식(예:
    `sd-jwt`)을 나열합니다.
4. **승인:** 사용자가 동의하면 플랫폼이 자격 증명 두 개를 생성합니다.
    * **CheckoutMandate:** `CheckoutObject`의 해시를 담습니다.
    * **PaymentMandate:** 결제 승인을 담은 SD-JWT-VC입니다.
5. **완료:** 플랫폼이 두 위임을 모두 비즈니스의 `/complete` 엔드포인트로
    제출합니다.
6. **검증:**
    * 비즈니스가 `CheckoutMandate`를 검증합니다.
    * 결제 처리업체(Payment Processor)가 `PaymentMandate`를 검증합니다.
7. **확정:** 검증에 성공하면 결제가 처리되고 주문이 확정됩니다.

**의존성:** 체크아웃 기능(Capability)

[AP2 위임 확장 전체 보기](../specification/ap2-mandates.md)

[AP2 자세히 알아보기](https://ap2-protocol.org){target="_blank"}
