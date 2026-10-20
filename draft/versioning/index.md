# 버전 관리

이 가이드에서는 UCP의 릴리스 브랜치와 백포트 절차를 설명합니다. UCP는 `YYYY-MM-DD` 형식의 날짜 기반 버전 식별자를 사용합니다. 규범적 릴리스 모델은 [컴포넌트 버전 관리와 릴리스 스냅샷](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#component-versioning-and-release-snapshots)을, 프로필 선택은 [프로토콜 버전](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#protocol-version)을 참고하세요.

새로운 개발은 `main` 브랜치에서 진행합니다. 지원하는 명세 릴리스마다 장기 브랜치를 유지하므로, 게시된 스냅샷 `D`를 참조하고 유지보수하는 데 계속 사용할 수 있습니다. 백포트 자격과 승인 기준은 [컴포넌트 버전 관리와 릴리스 스냅샷](https://dev-jonghoonpark.github.io/ucp-ko/draft/draft/specification/overview/#component-versioning-and-release-snapshots)에서 정의합니다. 기본적으로 승인된 하위 호환 변경만 허용하며, 호환성을 깨는 변경은 Governance Council이 예외적으로 승인한 경우에만 허용합니다. 백포트된 변경은 유지 중인 `release/D` 브랜치에 반영되며, UCP는 갱신된 산출물을 게시하기 전에 해당 스냅샷을 다시 인증합니다.

- Tech Council이 UCP의 새 버전을 승인하면, 현재 `main` 상태에서 바로 `release/YYYY-MM-DD`라는 이름의 새 브랜치를 만듭니다.
  - `release/YYYY-MM-DD` 브랜치를 만드는 즉시 해당 릴리스 브랜치에 코드 프리즈를 적용합니다. 이 기간에는 백포트 정책이 허용하는 변경만 반영해야 합니다.
  - 릴리스 브랜치를 만든 뒤 발견된 승인된 하위 호환 변경은 다음 두 방법 중 하나로 반영해야 합니다.
  - 릴리스 브랜치에서 변경한 뒤 `main`에 병합합니다.
  - `main`에서 변경한 뒤 릴리스 브랜치로 cherry-pick합니다.
- 릴리스가 확정되면 릴리스 브랜치를 `main`에 병합하고 태그를 붙입니다(예: `git tag -a vYYYY-MM-DD`). GitHub Action이 새 태그를 감지해 릴리스 노트 초안을 자동으로 만들고 산출물을 업로드합니다.
- 임시 기능 브랜치와 달리 `release/YYYY-MM-DD` 브랜치는 오래 유지되며, 이력 참조와 유지보수를 위해 명세의 특정 버전에 대응합니다.

## 호환성을 깨는 PR

- 호환성을 깨는 변경은 PR 제목에 `!`를 포함해야 합니다.
- 시기: 호환성을 깨는 변경은 병합 2주 전에 Discussions에 공지합니다.
- 보안에 민감한 수정은 예외입니다. Governance Council이 공개 일정을 사안별로 정하므로, 공개해도 안전해지기 전까지는 수정 사항을 공지하지 않습니다.
