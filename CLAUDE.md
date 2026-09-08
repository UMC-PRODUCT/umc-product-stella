# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **구조 안내**: 이 파일은 **핵심 요약 + 절대 규칙 + 레퍼런스 인덱스**만 담는 허브입니다.
> 상세는 `CONTRIBUTING.md` · `docs/claude/` · [Wiki](https://github.com/UMC-PRODUCT/umc-product-stella/wiki) 로 나눠져 있고,
> **필요할 때 해당 파일을 `Read` 로 열어** 봅니다. (컨텍스트 절약을 위해 `@import` 로 인라인하지 않습니다.)

## Project Overview

**Stella** — SwiftPM 도구 · macOS 15+ · Swift 6

- **목적**: UMC PRODUCT 서버 OpenAPI 스펙과 클라이언트의 Moya Router 연결 상태를 추적 —
  "어떤 API 가 아직 앱에 안 붙었는지", "각 엔드포인트 담당자가 누구인지"
- **소비자 레포**: `umc-product-iOS` (지원) · `umc-product-macOS` (미지원 — 프로젝트 플래그 일반화 선행 필요)
- **작성자**: 제옹(euijjang97)

## 구조 한눈에

```
OpenAPI 스펙 ─┐
              ├─→ StellaCore (파싱 · Router 스캔 · 매칭 · blame) ─→ coverage.json ─→ report / diff / GUI
Router 소스 ──┘
```

| 산출물 | 종류 | 역할 |
|--------|------|------|
| `StellaCore` | 라이브러리 | OpenAPI 파싱 · Router 스캔 · 매칭 · blame · 스냅샷 |
| `StellaTestSupport` | 라이브러리 | 테스트 헬퍼 (샌드박스 git 레포, URL 스텁) |
| `apicov` | 실행 파일 | CLI — `scan` · `report` · `diff` |
| `stella` | 실행 파일 | macOS GUI (`Sources/StellaApp`) |

CLI 와 GUI 는 둘 다 `StellaCore` 를 얇게 감쌉니다. 로직은 한 곳에만 둡니다.

## 절대 규칙 (항상 적용)

1. **`StellaCore` 는 UI 를 모른다** — `import SwiftUI`/`import AppKit` 이 코어에 들어가면 CLI 가 그 코드를 못 씁니다.
   로직은 코어에, 표시만 `apicov`/`StellaApp` 에.
2. **에러는 `StellaError` 로 모은다** (`Sources/StellaCore/Errors/StellaError.swift`).
3. **Swift 6 동시성** — 스냅샷을 오가는 타입은 `Sendable`, GUI 모델은 `@MainActor`.
4. **스냅샷 형식을 바꾸면 읽는 쪽 셋을 같이 고친다** — `coverage.json` 은 HTML 리포트 · `diff` · GUI 가 함께 읽습니다.
   옵셔널 필드 추가는 그대로 디코딩되므로 버전 유지, 필수 필드 추가·의미 변경은
   `CoverageSnapshot.currentSchemaVersion` 을 올립니다.
5. **`Fixtures/overrides.yml` 은 비어 있는 상태가 정상** — 규칙을 넣으면 매칭 테스트 기대값이 어긋납니다.
   운영 규칙은 레포 루트 `overrides.yml` 에 넣습니다.
6. **파일 헤더** — Swift 소스에는 붙이지 않습니다. 스크립트(`.sh`/`.swift` 스크립트)와 워크플로 `.yml` 에만
   `Created by euijjang97` 헤더를 `#` 주석으로 답니다. 문서(`.md`)·위키의 작성자 필드는 `제옹(euijjang97)`.
7. **식별자에 의미 없는 숫자 접미사 금지** — `text1`/`value2` 등 금지, 역할이 드러나는 이름을 씁니다.
   (상세: `docs/claude/coding-style.md`)
8. **커밋·PR·이슈에 AI 작성 흔적(attribution) 절대 금지** — `Co-Authored-By` 라인,
   `🤖 Generated with [Claude Code](...)` 푸터 등 일체 금지.
9. **작업 브랜치명은 `{타입}/{이슈번호}` — 이슈를 먼저 만들고 그 번호를 쓴다.**
   타입은 이슈 템플릿과 1:1: `feat` · `bug` · `design` · `refac` · `docs` · `chore`. base 는 `main`.
   설명형 브랜치명(`docs/repo-rename-links` 등) 금지.
   - 대응 이슈가 없으면 **브랜치를 만들기 전에 이슈부터 생성**합니다 (제목 접두사·라벨·Type·Priority/Effort까지).
   - **PR 제목은 `{이모지} [Type] {작업 내용} (#이슈번호)`** — 분류는 `[대괄호]`, 끝에 이슈번호.
     이슈 제목 형식(`📄 Docs: …` — 콜론)을 PR 제목에 쓰지 않습니다. `[Docs]:` 처럼 대괄호 뒤 콜론도 금지.
   - **PR 생성 시 Assignee 와 라벨을 반드시 지정** — `gh pr create` 에 `--assignee "@me"` 와
     `[Type]` 대응 라벨을 같이 넘깁니다. 라벨명은 `gh label list` 출력과 정확히 일치해야 합니다.
   - **PR 본문은 `.github/pull_request_template.md` 섹션 구조를 그대로 따른다.** 임의 목차 금지.
     `Closes #이슈번호` 는 `## 🔗 관련 이슈` 섹션에 넣습니다.
     ⚠️ `gh pr create --body "..."` 는 템플릿을 불러오지 않습니다 — 템플릿을 복사해 채운 뒤 `--body-file` 로 넘깁니다.
   - `main` 직접 푸시 금지, Squash and Merge, 최소 1인 Approve.
10. **커밋 메시지는 `{이모지} [Type] 한 줄 요약`** — PR 제목에서 `(#이슈번호)` 만 뺀 형태, 한국어.

## 빌드 · 테스트

레포 루트가 곧 패키지 루트입니다.

```bash
swift build
swift test                                    # 전체 (XCTest)
swift test --filter PathNormalizerTests       # 하나만
swift run apicov scan --help                  # CLI
swift run stella                              # macOS GUI
scripts/build-app.sh                          # dist/Stella.app 패키징 (ad-hoc 서명, 로컬 실행용)
```

테스트는 XCTest 를 쓰고 디렉터리가 소스 구조를 그대로 따라갑니다 —
`Sources/StellaCore/Matcher/PathNormalizer.swift` ↔ `Tests/StellaCoreTests/Matcher/PathNormalizerTests.swift`.
파싱·매칭 규칙을 바꿨으면 `Fixtures/MiniRepo` 에 케이스를 추가해 회귀를 잡아둡니다.

## 매핑 YAML

레포 루트의 세 파일이 운영본이며 **git 에 커밋합니다**. `*.yml.example` 은 템플릿,
`Fixtures/` 는 테스트 리소스 번들이라 운영 매핑이 아닙니다.

| 파일 | 역할 |
|------|------|
| `overrides.yml` | 자동 매칭 실패 Router case 보정 — 강제 매핑하거나 외부 API 를 `ignore` |
| `authors.yml` | git blame 이메일 → 표시명 · GitHub username |
| `owners.yml` | 엔드포인트별/태그별 담당자 |

## 상세 레퍼런스 (필요 시 Read)

| 주제 | 위치 | 언제 읽나 |
|------|------|----------|
| 기여 가이드 (개발 환경 · 코드 규칙 · 테스트 · 스냅샷) | `CONTRIBUTING.md` | Stella 자체를 고칠 때 먼저 |
| Git Workflow | `docs/claude/git-workflow.md` | 브랜치/커밋/PR/이슈 (템플릿 · Type · Priority) |
| 코딩 스타일 & 네이밍 | `docs/claude/coding-style.md` | 네이밍 판단이 필요할 때 |
| 이슈 · PR 템플릿 | `.github/` | 이슈·PR 생성 시 |
| 사용법 · CLI 레퍼런스 | `README.md` | 스캔을 돌리는 방법 |

산문 문서는 [Wiki](https://github.com/UMC-PRODUCT/umc-product-stella/wiki) 에 있습니다 —
[CLI 레퍼런스](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/CLI-Reference) ·
[매칭 파이프라인](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Matching-Pipeline) ·
[스냅샷 스키마](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Snapshot-Schema) ·
[트러블슈팅](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Troubleshooting).
위키는 별도 git 레포(`...stella.wiki.git`)이므로 따로 클론해야 합니다.
동작을 바꿨으면 해당 위키 페이지도 같이 고칩니다.
