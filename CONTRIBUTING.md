# 기여 가이드

Stella 자체를 고칠 때 읽으세요. 스캔을 돌리는 방법만 필요하다면 [README](README.md) 로 충분합니다.

- 작성자: 제옹(euijjang97)

## 준비

| 항목 | 값 |
|------|-----|
| Swift | 6.0 이상 |
| macOS | 15 이상 |

```bash
git clone git@github.com:UMC-PRODUCT/umc-product-stella.git
cd umc-product-stella
swift build
swift test
```

레포 루트가 곧 패키지 루트입니다. Xcode 로 열어도 되고 `swift build` 로 끝내도 됩니다.

동작을 직접 확인하려면 소비자 레포를 형제 디렉터리에 클론해두면 편합니다.

```
~/Work/
├── umc-product-iOS/
└── umc-product-stella/
```

## 변경 흐름

1. `main` 에서 브랜치를 땁니다
2. 고치고, 테스트를 붙이고, `swift test` 를 통과시킵니다
3. PR 을 올립니다 — 무엇을 왜 바꿨는지, 어떻게 확인했는지 세 줄이면 충분합니다

`main` 에 직접 푸시하지 않습니다.

## 커밋 메시지

`이모지 [타입] 한 줄 요약` 형식입니다.

```
📄 [Docs] README 를 독립 레포 기준으로 재작성
🐛 [Bug] 테스트 픽스처 overrides.yml 을 빈 상태로 복원
🔧 [Chore] 운영 매핑 YAML 3종을 레포 루트로 통합
✨ [Feat] 스터디 그룹 Command API 정합화
```

한국어로 씁니다. `git log --oneline` 으로 최근 것들을 보고 맞추면 됩니다.

## 코드 규칙

- **`StellaCore` 는 UI 를 모릅니다.** AppKit·SwiftUI import 가 들어가면 CLI 가 그 코드를 못 씁니다. 로직은 코어에, 표시만 `apicov`/`StellaApp` 에 둡니다
- **에러는 `StellaError` 로 모읍니다** (`Sources/StellaCore/Errors/StellaError.swift`)
- **Swift 6 동시성** — 스냅샷을 오가는 타입은 `Sendable`, GUI 모델은 `@MainActor` 입니다
- **파일 헤더** — Swift 소스에는 붙이지 않습니다. 스크립트와 워크플로 YAML 에만 `Created by euijjang97` 헤더가 있습니다
- 포매터·린터는 따로 두지 않았습니다. 주변 코드 스타일에 맞추세요

## 테스트

XCTest 를 쓰고, 테스트 디렉터리는 소스 구조를 그대로 따라갑니다. `Sources/StellaCore/Matcher/PathNormalizer.swift` 를 고쳤으면 `Tests/StellaCoreTests/Matcher/PathNormalizerTests.swift` 를 봅니다.

```bash
swift test
swift test --filter PathMatcherTests
```

파싱이나 매칭 규칙을 바꿨다면 `Fixtures/MiniRepo` 에 케이스를 하나 추가해 회귀를 잡아두세요.

**`Fixtures/overrides.yml` 은 비어 있는 상태가 정상입니다.** 여기에 규칙을 넣으면 매칭 테스트의 기대값이 어긋납니다. 운영 규칙은 레포 루트의 `overrides.yml` 에 넣으세요.

## 스냅샷 형식을 바꿀 때

`coverage.json` 은 HTML 리포트·`diff`·Stella 앱 셋이 함께 읽습니다.

- 옵셔널 필드를 더하는 정도면 기존 스냅샷이 그대로 디코딩되므로 `schemaVersion` 을 올리지 않아도 됩니다
- 필수 필드를 추가하거나 의미를 바꾸면 `CoverageSnapshot.currentSchemaVersion` 을 올리고 읽는 쪽 셋을 같이 손봅니다

필드 정의는 [스냅샷 스키마](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Snapshot-Schema) 에 정리돼 있습니다.

## 커밋하지 않는 것

`.gitignore` 가 `.build/` · `.swiftpm/` · `dist/` · `.claude/` · `_workspace/` 를 걸러줍니다.

레포 루트의 `authors.yml` · `owners.yml` · `overrides.yml` 은 운영 데이터라 **커밋합니다**. 새로 시작할 때는 옆의 `*.yml.example` 을 복사해 쓰세요.

## 문서

산문 문서는 [Wiki](https://github.com/UMC-PRODUCT/umc-product-stella/wiki) 에 있습니다. 위키는 별도 git 레포라 따로 클론해야 합니다.

```bash
git clone git@github.com:UMC-PRODUCT/umc-product-stella.wiki.git
```

동작을 바꿨으면 해당 위키 페이지도 같이 고쳐주세요. 특히 [CLI 레퍼런스](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/CLI-Reference) 와 [스냅샷 스키마](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Snapshot-Schema) 는 코드와 어긋나면 바로 티가 납니다.
