# 버전 관리 { #versioning }

이 가이드는 UCP의 릴리스 브랜치와 백포트 절차를 정의한다. UCP는 `YYYY-MM-DD`
형식의 날짜 기반 버전 식별자를 사용한다. 규범적 릴리스 모델은
[컴포넌트 버전 관리와 릴리스 스냅샷](specification/overview/index.md#component-versioning-and-release-snapshots)을,
프로필 선택은
[프로토콜 버전](specification/overview/index.md#protocol-version)을
참고한다.

새로운 개발은 `main` 브랜치에서 진행한다. 지원하는 모든 명세 릴리스마다
장기 브랜치를 유지하여, 게시된 스냅샷 `D`를 참조와 유지보수 목적으로 계속
사용할 수 있게 한다. 백포트 자격과 승인 기준(기본적으로는 승인된 하위 호환
변경만 허용하고, 호환성을 깨는 변경은 Governance Council의 예외적 승인을
거쳐야만 허용)은
[컴포넌트 버전 관리와 릴리스 스냅샷](specification/overview/index.md#component-versioning-and-release-snapshots)에서
정의한다. 백포트된 변경은 유지 중인 `release/D` 브랜치에 반영되며, UCP는
갱신된 산출물을 게시하기 전에 해당 스냅샷을 다시 인증한다.

* Tech Council이 UCP의 새 버전을 승인하면, 현재 `main` 상태에서 바로
  `release/YYYY-MM-DD`라는 이름의 새 브랜치를 만든다.
    * `release/YYYY-MM-DD` 브랜치를 만드는 즉시 해당 릴리스 브랜치에 코드
      프리즈를 적용한다. 이 기간에는 백포트 정책이 허용하는 변경만
      반영해야 한다.
    * 릴리스 브랜치를 만든 뒤 발견된, 승인된 하위 호환 변경은 다음 두 방법
      중 하나로 반영해야 한다.
      1. 릴리스 브랜치에서 변경한 뒤 `main`에 병합한다.
      2. `main`에서 변경한 뒤 릴리스 브랜치로 cherry-pick한다.
* 릴리스가 확정되면 릴리스 브랜치를 `main`에 병합하고 태그를 붙인다(예:
  `git tag -a vYYYY-MM-DD`).  GitHub Action이 새 태그를 감지해 릴리스 노트
  초안을 자동으로 생성하고 산출물을 업로드한다.
* 임시 기능 브랜치와 달리 `release/YYYY-MM-DD` 브랜치는 장기간 유지되며,
  이력 참조와 유지보수를 위해 명세의 특정 버전에 대응한다.

## 호환성을 깨는 PR { #breaking-prs }

* 호환성을 깨는 변경은 PR 제목에 `!`를 포함해야 한다
* 시기: 호환성을 깨는 변경은 병합 2주 전에 Discussions에 공지한다.
* 보안에 민감한 수정은 예외다. Governance Council이 공개 일정을 사안별로
  정하므로, 수정 사항은 공개해도 안전해지기 전까지 공지하지 않는다.
