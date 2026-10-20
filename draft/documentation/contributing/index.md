# UCP에 기여하기

[Universal Commerce Protocol(UCP) GitHub 저장소](https://github.com/Universal-Commerce-Protocol/ucp)에서는 표준을 함께 만들고 발전시켜 나갈 커뮤니티의 기여, 패치, 피드백을 환영합니다. 오타 수정, 문서 개선, SDK 코드 작성, 프로토콜 확장 제안 등 어떤 형태로든 참여해 주시면 에이전트 커머스(agentic commerce)를 위한 개방형 표준을 세우는 데 큰 힘이 됩니다.

______________________________________________________________________

## 첫 기여

어떤 작업부터 시작할 수 있는지 살펴보세요.

- **문서 개선**: 코드 스니펫을 다듬고, 명세 문구를 명확히 하고, 문서를 개선하거나 깨진 링크를 고칩니다.
- **입문용 이슈**: `good first issue` 또는 `help wanted` 태그가 붙은 [열린 이슈](https://github.com/Universal-Commerce-Protocol/ucp/issues)를 살펴봅니다.
- **스키마 예시**: 프로토콜 기능을 보여 주는 스키마 예시 페이로드(`source/schemas/` 또는 `docs/specification/`)를 만들거나 갱신합니다.
- **SDK 및 적합성 테스트 개선**: [`python-sdk`](https://github.com/Universal-Commerce-Protocol/python-sdk), [`js-sdk`](https://github.com/Universal-Commerce-Protocol/js-sdk), [`conformance`](https://github.com/Universal-Commerce-Protocol/conformance)의 버그를 고치거나 테스트 커버리지를 넓힙니다.

### 작은 변경(바로 PR)

일상적인 수정이나 사소한 업데이트는 바로 풀 리퀘스트로 제출하면 됩니다.

- 버그 수정
- 문서나 스키마 속성 설명의 명확화
- 스키마 예시와 튜토리얼 추가

### 큰 변경(개선 제안 필요)

프로토콜을 크게 바꾸는 변경은 모두 [개선 제안 템플릿](https://github.com/Universal-Commerce-Protocol/ucp/issues/new?template=enhancement-proposal.yml)으로 정식 개선 제안(Enhancement Proposal, EP)을 제출하고 Domain Tech Council(DTC)의 승인을 받아야 합니다.

큰 변경에는 다음이 포함됩니다.

- **코어 스키마 수정**: JSON 스키마의 필드나 설명을 추가, 삭제, 수정하는 경우
- **프로토콜 변경**: 통신 흐름이나 기능(체크아웃, 장바구니, 카탈로그, 주문, 신원 연결(Identity Linking))의 기대 동작을 바꾸는 경우
- **새 API 엔드포인트 / 전송 방식**: 새 전송 방식 바인딩(REST, MCP, A2A, 임베디드)이나 새 프로토콜 엔드포인트를 도입하는 경우
- **하위 호환성 깨짐**: 기존 기능이나 스키마의 호환성을 깨는 모든 변경(Governing Council 승인과 Discussions에서 2주 전 사전 공지가 필요합니다. [버전 관리](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/versioning/index.md) 참고)

**벤더 확장 vs. 코어(`dev.ucp.*`)**: UCP는 코어 프로토콜을 가볍게 유지하도록 설계되었습니다. 특정 벤더만 쓰거나 새로 떠오르는 사용 사례를 위한 기능이나 확장은 먼저 벤더 네임스페이스(예: `com.{vendor}.*`, [네임스페이스 거버넌스](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#namespace-governance) 참고) 아래에서 자체 호스팅해야 합니다. 여러 주체가 참여하는 생태계 전반에서 널리 채택된 뒤에 `dev.ucp.*`로 편입하자고 제안하세요.

**기능 요청 vs 개선 제안**: 구현 계획 없이 새 기능만 제안하려면 일반 "Feature Request" 이슈를 제출합니다. 기술 설계까지 함께 제안한다면 "Enhancement Proposal" 이슈를 제출하고, 풀 리퀘스트에서 그 이슈를 참조해 구현합니다.

______________________________________________________________________

## 개선 제안 생명주기

개선 제안은 프로토콜 변경이 세 단계 생명주기를 거치는 과정을 추적합니다.

```
flowchart LR
    P["Proposal<br/>Public debate & feedback"] -->|DTC Majority Vote| PR["Provisional<br/>Working draft & prototyping"]
    PR -->|DTC Majority Vote| IMP["Implemented<br/>Merged & finalized"]
```

| 단계            | 상태 및 설명                                                                                     |
| --------------- | ------------------------------------------------------------------------------------------------ |
| **Proposal**    | 커뮤니티 구성원 누구나 제출할 수 있으며, 커뮤니티와 Domain Tech Council이 공개적으로 논의합니다. |
| **Provisional** | DTC 과반수 투표로 승인되면 작업 초안을 반복하며 프로토타이핑하는 단계에 들어갑니다.              |
| **Implemented** | DTC 과반수 투표로 확정되며, 코드·스키마·문서가 완성되어 병합된 상태입니다.                       |

모든 개선 제안 이슈는 다음 항목을 채우도록 되어 있는 표준 템플릿을 따라야 합니다.

- **Summary**: 변경 사항의 상위 수준 요약
- **Motivation**: 문제 정의와 사용자 가치
- **Goals & Non-Goals**: 구체적이고 측정 가능한 결과, 그리고 범위가 불어나지 않도록 명시한 범위 밖 경계
- **Detailed Design**: 기술 명세, 스키마 영향, API 표면, 엣지 케이스
- **Risks and Mitigations**: 호환성, 마이그레이션 고려 사항, 성능 및 보안 영향
- **Test Plan**: 여러 전송 방식과 적합성 테스트 스위트 전반의 검증 전략
- **Graduation Criteria**: Candidate 및 Stable 단계로 올라가기 위한 요건

### 개선 제안 이슈와 풀 리퀘스트의 협업 방식

개선 제안(EP)은 서로 연결된 두 GitHub 산출물로 **설계 합의**와 **코드·스키마 제공**을 분리합니다.

- **EP 이슈(설계 및 생명주기 추적)**: [개선 제안 템플릿](https://github.com/Universal-Commerce-Protocol/ucp/issues/new?template=enhancement-proposal.yml)으로 엽니다(`enhancement-proposal` 라벨). *왜*와 *무엇*에 관한 단일 진실 공급원으로, 동기, 상세 설계, 위험, 테스트 계획, 졸업 기준, 그리고 계속 갱신되는 `Implementation History` 로그를 담습니다. EP 이슈는 기능이 **Stable**(`Implemented`)로 졸업할 때까지 생명주기 내내 열어 둡니다.
- **풀 리퀘스트(구체적인 구현)**: JSON 스키마(`source/`), 명세 문서(`docs/specification/`), SDK 모델, 적합성 테스트, 샘플 등 *어떻게*에 해당하는 부분을 제공합니다. EP를 구현하는 모든 PR은 PR 설명의 `## Related Issues` 아래에 추적 이슈를 연결해야 합니다. 점진적인 PR에는 `Part of #<issue-number>`를, 최종 졸업을 마무리하는 PR에는 `Fixes #<issue-number>`를 씁니다.

```
sequenceDiagram
    actor Author as Contributor
    participant Issue as EP Issue
    participant DTC as Domain Tech Council
    participant PR as Pull Requests

    Author->>Issue: 1. Open EP Issue (Proposal)
    Note over Issue,DTC: Triage & community design discussion<br/>(Optional Draft PR to illustrate diffs)
    DTC->>Issue: 2. Vote to accept as Provisional
    Note over Issue: Implementation History updated<br/>Capability enters Working Draft
    Author->>PR: 3. Open PR linked to EP Issue
    DTC->>PR: 4. Schema, spec & governance review
    PR->>Issue: 5. PR merged & progress logged in EP Issue
    Note over Issue,PR: Additional PRs (SDKs, conformance, feedback)<br/>advance capability to Candidate and Stable
    DTC->>Issue: 6. Final graduation vote (Implemented) & close EP Issue
```

#### 단계별 워크플로

1. **EP 이슈부터 엽니다(`Proposal` 단계)**: 전체 구현 코드를 작성하거나 병합할 수 있는 PR을 열기 전에 EP 이슈를 먼저 제출합니다. *(선택 사항: 구체적인 JSON 스키마 diff나 프로토타입 코드가 설계를 설명하는 데 도움이 된다면 EP 이슈에 연결된* *Draft PR\*\*을 열어도 됩니다. 다만 EP가 `Provisional` 상태가 되기 전에는 구현 PR을 병합하지 않습니다.)*
1. **설계 리뷰와 Provisional 승인(`Proposal` → `Provisional`)**: 트리아저가 EP 이슈에 기능 도메인 라벨(예: `area:shopping`, `area:payments`, `area:lodging`, `area:food`, `area:common`)을 붙이고 관련 **Domain Working Group**(DWG)과 **Domain Tech Council**(DTC)에 넘깁니다. 열려 있던 설계 쟁점이 해결되면 DTC가 제안을 **Provisional**로 받아들일지 투표하고, 그 결정을 이슈의 `Implementation History` 섹션에 기록합니다.
1. **구현 PR 제출(`Provisional` 단계)**: EP 이슈가 **Provisional**로 승인되면(기능 성숙도는 `Working Draft` 단계에 들어갑니다) `Part of #<EP-issue-number>`를 참조하는 구현 PR을 엽니다(또는 *Ready for Review*로 표시합니다). 규모가 큰 제안은 여러 저장소에 걸친 여러 PR로 나뉘는 경우가 많습니다. 예를 들어 `ucp`에 코어 스키마와 명세를 반영한 뒤 `ucp-schema`에 검증 규칙을, `python-sdk`/`js-sdk`에 SDK 지원을, `conformance`에 테스트를 추가합니다.
1. **졸업을 향한 반복(`Provisional` → `Implemented`)**: 구현 PR이 병합되고 실제 도입 피드백이 반영되어 기능이 `Working Draft`에서 `Candidate`, `Stable`로 올라가면 메인테이너가 EP 이슈의 `Implementation History`를 갱신합니다. 졸업 기준과 적합성 테스트를 모두 충족하면 DTC가 최종 투표로 EP를 **Implemented**로 표시하고(기능을 **Stable**로 승격) EP 이슈를 닫습니다.

______________________________________________________________________

## 기능 성숙도 단계

개선 제안이 Provisional 상태가 되면 새 기능은 세 가지 성숙도 단계를 거칩니다.

```
flowchart LR
    WD["Working Draft<br/>Breaking changes expected"] -->|DTC Vote| CD["Candidate<br/>API frozen; pilot testing"]
    CD -->|DTC Vote| ST["Stable (YYYY-MM-DD)<br/>Production ready & backward compatible"]
```

| 단계              | 버전 태그       | 안정성 보장                                                                                                     | 목적                                 | 종료 기준                        |
| ----------------- | --------------- | --------------------------------------------------------------------------------------------------------------- | ------------------------------------ | -------------------------------- |
| **Working Draft** | `Working Draft` | 호환성을 깨는 변경이 있을 수 있음                                                                               | 프로토타이핑, 피드백 수집, 설계 반복 | DTC 과반수 투표로 다음 단계 진행 |
| **Candidate**     | `Candidate`     | API 표면은 안정적이나 구현 세부 사항은 바뀔 수 있음                                                             | 얼리 어답터 구현, 프로덕션 파일럿    | DTC 과반수 투표로 다음 단계 진행 |
| **Stable**        | `YYYY-MM-DD`    | 완전한 하위 호환성([버전 관리](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/versioning/index.md) 참고) | 프로덕션 배포                        | 날짜 기반 버전 부여              |

______________________________________________________________________

## 저장소 구조

Universal Commerce Protocol 생태계는 [Universal-Commerce-Protocol](https://github.com/Universal-Commerce-Protocol) GitHub 조직 아래의 여러 전용 Git 저장소로 이루어져 있습니다.

- **🏠 [`/ucp`](https://github.com/Universal-Commerce-Protocol/ucp)**: 코어 명세 저장소. 프로토콜 정의, 웹사이트(ucp.dev), 명세 사이트 생성기
- **📜 [`/ucp-schema`](https://github.com/Universal-Commerce-Protocol/ucp-schema)**: JSON 스키마와 Rust 기반 검증·생성 도구.
- **🧪 [`/conformance`](https://github.com/Universal-Commerce-Protocol/conformance)**: pytest로 작성된 프로토콜 적합성 테스트 스위트.
- **⚙️ [`/python-sdk`](https://github.com/Universal-Commerce-Protocol/python-sdk) & [`/js-sdk`](https://github.com/Universal-Commerce-Protocol/js-sdk)**: 프로토콜 스키마로부터 생성된 공식 언어별 SDK.
- **🔎 [`/samples`](https://github.com/Universal-Commerce-Protocol/samples)**: 레퍼런스 판매자 구현(FastAPI, Node.js).
- **♥️ [`/.github`](https://github.com/Universal-Commerce-Protocol/.github)**: 조직 전체에 적용되는 상태(health) 파일, 이슈 템플릿(`enhancement-proposal.yml`, `bug-report.yml`), 커뮤니티 거버넌스 기본값.

______________________________________________________________________

## 첫 PR을 열기 전 체크리스트

- **CLA**: [Google 기여자 라이선스 계약(CLA)](https://cla.developers.google.com/)에 서명했나요?
- **사전 확인**: 이 버그나 기능을 추적하는 기존 이슈가 있고, 그 이슈가 본인에게 할당되어 있나요?
- **브랜치**: `main`에서 브랜치를 따고, 커밋을 원자적으로 유지하고 있나요?
- **검증**: 로컬 린터(`pre-commit`), 스키마 검증기(`ucp-schema lint source/`), 적합성 테스트 스위트를 실행했나요?
- **PR 제목 및 설명**: PR 제목이 **Conventional Commits**(예: `feat: ...`, `fix: ...`, `docs: ...`, 호환성을 깨는 변경에는 `feat!: ...`) 형식을 따르나요? 설명에 **왜**, **무엇**이 바뀌었는지 적고 관련 이슈 링크를 넣었나요?

______________________________________________________________________

## 시작하기

바로 시작하고 싶다면 [UCP GitHub 조직](https://github.com/Universal-Commerce-Protocol)에서 저장소를 둘러보고, [열린 이슈](https://github.com/Universal-Commerce-Protocol/ucp/issues)를 살펴보거나, 첫 풀 리퀘스트를 제출해 보세요.

프로토콜이 처음이라면 [핵심 개념](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/documentation/core-concepts/index.md)에서 아키텍처를 익히고, [샘플](https://github.com/Universal-Commerce-Protocol/samples)에서 레퍼런스 구현을 살펴보고, [GitHub Discussions](https://github.com/Universal-Commerce-Protocol/ucp/discussions)에서 대화에 참여해 보세요.
