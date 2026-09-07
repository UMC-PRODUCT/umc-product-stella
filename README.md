# Stella

UMC PRODUCT 서버의 OpenAPI 스펙과 클라이언트 코드베이스의 Moya Router 연결 상태를 추적하는 SwiftPM 도구입니다. "어떤 API가 아직 앱에 안 붙었는지", "각 엔드포인트의 담당자가 누구인지"를 스냅샷(`coverage.json`)과 HTML 리포트로 보여줍니다.

- 작성자: 제옹(euijjang97)
- 상세 문서: [Wiki](https://github.com/UMC-PRODUCT/umc-product-stella/wiki)

## 왜 별도 레포인가

원래 iOS 레포(`umc-product-iOS`)의 `Stella/` 하위 디렉터리였습니다. macOS 앱(`umc-product-macOS`)이 추가되면서 같은 도구를 두 소비자가 공유해야 했습니다. 그래서 분리했습니다. 이 레포는 **도구만** 소유합니다 — 커버리지 스캔 워크플로·서버 시크릿·GitHub Pages 는 소비자 레포가 각자 가집니다.

## 소비자 레포

| 레포 | 상태 | 비고 |
|------|------|------|
| `UMC-PRODUCT/umc-product-iOS` | 지원 | `--app-product`(레거시 `AppProduct/`) · `--umc-app`(Tuist `UMCApp/`) 두 프로젝트 스캔 |
| `UMC-PRODUCT/umc-product-macOS` | **미지원** | `apicov scan` 의 프로젝트 플래그가 iOS 두 프로젝트로 고정돼 있어(`Sources/apicov/Commands/ScanCommand.swift:24-27`) 플래그 일반화가 선행돼야 합니다. 코어의 `ProjectInput`(`Sources/StellaCore/Pipeline/ScanConfig.swift:3`) 자체는 `key`/`displayName`/`rootPath`/`routerGlobs` 를 받는 범용 구조입니다 |

## 빠른 시작

macOS 15+, Swift 6 툴체인이 필요합니다(`Package.swift`). 레포 루트가 곧 패키지 루트입니다. 소비자 레포는 형제 디렉터리에 클론돼 있다고 가정합니다.

```bash
swift build

export UMC_API_USER=...
export UMC_API_PASS=...

swift run apicov scan \
  --openapi-url https://dev.api.umc.it.kr/docs-json \
  --auth-env UMC_API_USER:UMC_API_PASS \
  --app-product ../umc-product-iOS/AppProduct \
  --umc-app ../umc-product-iOS/UMCApp \
  --blame-root ../umc-product-iOS \
  --authors authors.yml \
  --overrides overrides.yml \
  --owners owners.yml \
  --out coverage.json

swift run apicov report coverage.json --out coverage.html
swift run apicov diff old-coverage.json coverage.json
swift run stella          # 담당자 매핑 편집용 macOS GUI
```

## 매핑 YAML

레포 루트의 세 파일이 운영본이며 git 에 커밋돼 있습니다. `*.yml.example` 은 템플릿, `Fixtures/` 는 테스트 리소스 번들(`Package.swift` 의 `.copy("../../Fixtures")`)이라 운영 매핑이 아닙니다.

| 파일 | 역할 |
|------|------|
| `overrides.yml` | 자동 매칭 실패 Router case 보정 — OpenAPI 키로 강제 매핑하거나 외부 API 를 `ignore` |
| `authors.yml` | git blame 이메일 → 표시명·GitHub username |
| `owners.yml` | 엔드포인트별/태그별 담당자 (이메일은 `authors.yml` 로 표시명 해석) |

## 더 보기

서브커맨드 상세, GUI ↔ `owners.yml` 동작, `.app` 번들 패키징, 소비자 레포 CI 연동은 [Wiki](https://github.com/UMC-PRODUCT/umc-product-stella/wiki) 에 있습니다.
