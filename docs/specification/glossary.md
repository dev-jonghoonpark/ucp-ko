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

# 용어집 { #glossary }

이 용어집에는 UCP 명세 전반에서 쓰는 약어와 용어를 최대한 모아 두었습니다.
새 항목은 해당 범주 안에서 알파벳 순서에 맞춰 추가해야 합니다.

**참고:** 용어집이 있더라도 각 명세 Markdown 파일에서 약어를 처음 쓸 때는
전체 용어를 풀어 쓰는 것이 좋습니다(예: "Payment Card Industry Data Security
Standard (PCI-DSS)").

## 프로토콜 { #protocol }

| 용어 | 약어 | 정의 |
| :--- | :--- | :--- |
| **Action(액션)** | - | 확장이 정의하고 플랫폼이 처리해야 하는 미처리 작업. 액션을 채택한 기능의 응답 전용 `actions` 맵에만 나타납니다. |
| **Agent Payments Protocol(에이전트 결제 프로토콜)** | AP2 | AI 에이전트가 안전하게 상호 운용하며 자율적으로 결제를 완료할 수 있도록 설계된 개방형 프로토콜. UCP는 안전한 결제 위임(Mandate)에 AP2를 활용합니다. |
| **Agent2Agent Protocol(에이전트 간 프로토콜)** | A2A | 여러 AI 에이전트가 안전하게 협업하며 통신하기 위한 개방형 표준. UCP는 A2A를 전송 계층으로 사용할 수 있습니다. |
| **Capability(기능)** | - | 비즈니스가 지원하는 독립적인 핵심 기능(예: Checkout, Identity Linking). 기능은 UCP의 기본 "동사"입니다. |
| **Credential Provider(자격 증명 제공자)** | CP | 사용자의 결제 및 신원 자격 증명을 안전하게 관리하고 실행하는 신뢰할 수 있는 주체(디지털 지갑 등). |
| **Extension(확장)** | - | `extends` 필드로 다른 기능을 보강하는 선택적 기능. 확장은 핵심 기능과 함께 `ucp.capabilities[]`에 나타납니다. |
| **Model Context Protocol(모델 컨텍스트 프로토콜)** | MCP | AI 모델이 외부 데이터와 도구에 연결하는 방식을 표준화한 프로토콜. UCP 기능은 MCP 도구와 1:1로 대응합니다. |
| **Profile(프로필)** | - | 비즈니스와 플랫폼이 well-known URI에 호스팅하는 JSON 문서. 자신의 신원, 지원 기능, 엔드포인트를 선언합니다. |
| **Protocol Namespace(프로토콜 네임스페이스)** | - | 스키마로 필드가 정의된 구조화된 UCP 객체에서 프로토콜 메타데이터와 주석용으로 예약한 `ucp` 멤버. 딕셔너리는 제외합니다. 딕셔너리에서 `ucp`는 그냥 키 데이터이지만, 값으로 들어 있는 구조화된 객체는 여전히 대상입니다. 최상위 엔벨로프가 그 루트입니다. |
| **Universal Commerce Protocol(유니버설 커머스 프로토콜)** | UCP | 이 문서에서 정의하는 표준. 표준화된 기능과 디스커버리로 커머스 주체 간 상호 운용성을 보장합니다. |

## 커머스 { #commerce }

| 용어 | 약어 | 정의 |
| :--- | :--- | :--- |
| **Business(비즈니스)** | - | 상품이나 서비스를 판매하는 주체. UCP에서 비즈니스는 **Merchant of Record(MoR)** 역할을 하며 재무적 책임과 주문 소유권을 집니다. |
| **Buyer(구매자)** | - | UCP로 커머스 행위를 완료하는 거래 고객 또는 최종 소비자. `user`, `consumer` 같은 용어와 같은 뜻입니다. |
| **Location(위치)** | - | 비즈니스의 네임스페이스 안에서 안정적이고 불투명한(opaque) 문자열로 식별되는 물리적 장소(예: 매장, 식당, 숙소, 픽업 지점). |
| **Increment(증분)** | - | 판매 기준의 선택적 [주문 단위](overview/index.md#ordering-increment)로, 스텝 수로 표현합니다. 플랫폼이 작성하는 수량에는 권고 사항으로만 적용됩니다. |
| **Merchant of Record(공식 판매자)** | MoR | 재무적 책임과 주문 소유권을 포함하여 판매에 대한 책임을 지는 법적 주체. |
| **Payment Service Provider(결제 서비스 제공자)** | PSP | 비즈니스를 대신하여 결제, 승인, 정산을 처리하는 금융 인프라 제공자. |
| **Platform(플랫폼)** | - | 사용자를 대신하여 비즈니스를 탐색하고 커머스를 지원하는 소비자 대면 접점(AI 에이전트, 앱, 웹사이트). |
| **Quantity Unit(수량 단위)** | - | 기계가 식별할 수 있는 정보(`unit`, 실효 `scale`)로 이루어진 [판매 기준 서술자](overview/index.md#quantities-and-units). `display_text`/`increment`는 제외합니다. |
| **Step(스텝)** | - | 권위 있는 판매 기준에서의 [정수 증분](overview/index.md#quantities-and-units) 하나. `unit`의 `10^-scale`에 해당합니다. |

## 결제 { #payments }

| 용어 | 약어 | 정의 |
| :--- | :--- | :--- |
| **Card Verification Value(카드 인증 값)** | CVV | 결제 카드에 적힌 3자리 또는 4자리 보안 코드. 카드 비대면 거래를 검증하는 데 씁니다. |
| **Payment Card Industry Data Security Standard(지불 카드 산업 데이터 보안 표준)** | PCI-DSS | 신용카드 정보를 수락, 처리, 저장, 전송하는 모든 기업이 안전한 환경을 유지하도록 설계된 보안 표준 모음. |
| **Primary Account Number(기본 계좌 번호)** | PAN | 카드 발급사와 카드 소지자 계좌를 식별하는 고유한 결제 카드 번호(일반적으로 13~19자리). |
| **Strong Customer Authentication(강력한 고객 인증)** | SCA | 결제 서비스 제공자가 전자 결제에 다중 인증을 적용하도록 하는 PSD2상의 요구 사항. |
| **3D Secure** | 3DS | 카드 소지자 인증으로 온라인 신용카드 및 직불카드 거래에 보안 계층을 추가하도록 설계된 프로토콜. |

## 규정 준수 및 규제 { #compliance-regulatory }

| 용어 | 약어 | 정의 |
| :--- | :--- | :--- |
| **California Consumer Privacy Act(캘리포니아 소비자 프라이버시법)** | CCPA | 미국 캘리포니아주 거주자의 프라이버시 권리와 소비자 보호를 강화하기 위한 주 법률. |
| **General Data Protection Regulation(일반 데이터 보호 규정)** | GDPR | 유럽연합(EU)과 유럽경제지역(EEA)의 데이터 보호 및 프라이버시에 관한 EU 법률상의 규정. |
| **Know Your Customer(고객 확인)** | KYC | 사기, 자금 세탁, 테러 자금 조달을 방지하기 위해 고객의 신원을 확인하는 절차. |

## 표준 및 명세 { #standards-specifications }

| 용어 | 약어 | 정의 |
| :--- | :--- | :--- |
| **International Organization for Standardization(국제표준화기구)** | ISO | 여러 국가 표준 기관의 대표로 구성된 국제 표준 제정 기구. UCP는 국가 코드(ISO 3166-1), 통화 코드(ISO 4217), 날짜 형식(ISO 8601)에서 ISO 표준을 참조합니다. |
| **Verifiable Digital Credential(검증 가능한 디지털 자격 증명)** | VDC | 발급자(Issuer)가 서명한 자격 증명(클레임 집합)으로, 진위를 암호학적으로 검증할 수 있습니다. UCP에서는 안전한 결제 승인에 사용합니다. |
