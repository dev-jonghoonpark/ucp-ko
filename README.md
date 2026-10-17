<!-- markdownlint-disable-file MD041 -->

<p align="center">
  <h1 align="center">Universal Commerce Protocol (UCP) 한국어 번역</h1>
</p>

<p align="center">
  <b>UCP 공식 문서와 명세를 한국어로 옮긴 비공식 번역입니다.</b>
</p>

<p align="center">
  <a href="https://dev-jonghoonpark.github.io/ucp-ko/">번역 사이트</a> |
  <a href="https://ucp.dev">공식 문서</a> |
  <a href="https://ucp.dev/latest/specification/overview/">공식 명세(latest)</a> |
  <a href="https://github.com/Universal-Commerce-Protocol/ucp">공식 저장소</a>
</p>

> [!IMPORTANT]
> 이 번역은 원문 저장소
> [`Universal-Commerce-Protocol/ucp`](https://github.com/Universal-Commerce-Protocol/ucp)의
> main 브랜치 커밋 [`3ab4104`](https://github.com/Universal-Commerce-Protocol/ucp/tree/3ab41043b79373df33a6e2c377bc82a806175f78)
> (2026-10-16)을 기준으로 합니다. main 브랜치는 공식 사이트의 **초안(draft)** 에 해당하므로,
> 최신 안정 버전([ucp.dev/latest](https://ucp.dev/latest/specification/overview/))이나
> 이후에 갱신된 공식 문서와 내용이 다를 수 있습니다. 구현할 때는 반드시 공식 문서를 확인하세요.

## 무엇이 들어 있나

- `docs/` — 한국어로 번역한 문서(개요, 핵심 개념, 전체 명세)
- `source/` — 원문 그대로의 JSON 스키마. 스키마 표에 나오는 필드 설명은 이 스키마에서 생성되므로 영어로 남아 있습니다.
- `overrides/main.html` — 상단 공지와 페이지마다 붙는 "공식 원문 보기" 링크

사이트 스타일(MkDocs Material 테마, 색상, 레이아웃)은 원문 사이트를 그대로 따릅니다.
원문 사이트와 달리 버전별(mike) 배포 없이 개요와 명세를 한 사이트로 빌드합니다.

## 로컬에서 빌드하기

[uv](https://docs.astral.sh/uv/)와 Rust(cargo)가 필요합니다.

```bash
cargo install ucp-schema --version 1.4.1 --locked
uv sync
DOCS_MODE=all uv run mkdocs serve
```

`DOCS_MODE=all`은 이 번역 저장소에서 추가한 모드로, 개요와 명세를 한 사이트에 함께 빌드합니다.

## 라이선스

원문과 같은 [Apache License 2.0](LICENSE)을 따릅니다. 원저작권은 UCP Authors에게 있으며,
이 저장소는 원문을 한국어로 번역하고 빌드 설정을 일부 수정한 2차 저작물입니다.
