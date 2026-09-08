# Stella

**서버 OpenAPI 스펙과 클라이언트 Moya Router 사이의 연결 상태를 추적하는 SwiftPM 도구입니다.**

서버는 API 를 계속 내놓는데 앱이 그중 무엇을 붙였는지는 아무도 한눈에 모릅니다. Stella 는 OpenAPI 스펙에서 엔드포인트 목록을 뽑고, 클라이언트 레포의 Moya Router 소스를 파싱해 실제로 선언된 경로와 맞춰 봅니다. 그래서 다음 두 질문에 답합니다.

- **어떤 API 가 아직 앱에 안 붙었나** — 스펙에는 있는데 Router 에 없는 엔드포인트
- **각 엔드포인트 담당자가 누구인가** — git blame 으로 Router case 를 마지막에 만진 사람을 찾고, 매핑 YAML 로 표시명·GitHub username 을 붙임

### 산출물

| 산출물 | 만드는 명령 | 쓰는 곳 |
|--------|------------|---------|
| `coverage.json` 스냅샷 | `apicov scan` | 다른 모든 산출물의 입력. CI 아티팩트로 보관 |
| HTML 리포트 | `apicov report` | 의존성 없는 단일 페이지. GitHub Pages 에 그대로 올림 |
| 스냅샷 diff | `apicov diff` | 두 스냅샷 사이에 새로 붙은/떨어져 나간 엔드포인트 |
| 담당자 매핑 | `authors.yml` · `owners.yml` (GUI 로 편집) | 리포트의 담당자 열 |

- 관리자: 제옹(euijjang97)
- 지식 재산권: 제옹(euijjang97)
- 라이선스: [MIT](LICENSE)
- 상세 문서: [Wiki](https://github.com/UMC-PRODUCT/umc-product-stella/wiki)

## 소비자 레포

| 레포 | 상태 | 비고 |
|------|------|------|
| `UMC-PRODUCT/umc-product-iOS` | 지원 | 프로젝트 2종 스캔 |
| `UMC-PRODUCT/umc-product-macOS` | 미지원 | 플래그 일반화 선행 |

iOS 레포는 `--app-product`(레거시 `AppProduct/`) 와 `--umc-app`(Tuist `UMCApp/`) 두 프로젝트를 함께 스캔합니다.

macOS 레포를 아직 지원하지 않는 것은 코어가 아니라 CLI 쪽 제약 때문입니다.
코어의 `ProjectInput`(`Sources/StellaCore/Pipeline/ScanConfig.swift:3`) 은
`key`·`displayName`·`rootPath`·`routerGlobs` 를 받는 범용 구조라 프로젝트 종류를 가리지 않습니다.
반면 `apicov scan` 의 프로젝트 플래그는 iOS 두 프로젝트로 고정돼 있어서
(`Sources/apicov/Commands/ScanCommand.swift:24-27`) macOS 를 붙이려면 이 플래그를 먼저 일반화해야 합니다.

## 실행법

### 전제

- **macOS 15+ · Swift 6 툴체인** (`Package.swift` 의 `swift-tools-version: 6.0`, `platforms: [.macOS(.v15)]`)
- 레포 루트가 곧 패키지 루트라 별도 설정 없이 `swift build` 가 돕니다
- **소비자 레포가 형제 디렉터리에 클론돼 있다고 가정합니다** — 아래 예시의 `../umc-product-iOS` 가 그 가정입니다

```
~/work/
├── umc-product-stella/     ← 여기서 명령을 실행
└── umc-product-iOS/        ← 스캔 대상
```

### 1. 빌드

```bash
swift build
```

`.build/debug/` 에 `apicov` 와 `stella` 실행 파일이 생깁니다. 아래에서는 `swift run` 으로 부릅니다.

### 2. `apicov scan` — 스냅샷 만들기

**입력** OpenAPI 스펙(URL 또는 로컬 파일) + 소비자 레포의 Router 소스 + 매핑 YAML 3종
**출력** `coverage.json` (`--out` 생략 시 stdout)

```bash
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
```

### 3. `apicov report` — HTML 리포트

**입력** `coverage.json` **출력** 의존성 없는 단일 HTML (`--out` 생략 시 stdout)

```bash
swift run apicov report coverage.json --out coverage.html
open coverage.html
```

### 4. `apicov diff` — 두 스냅샷 비교

**입력** 스냅샷 두 개(옛것, 새것) **출력** 텍스트 diff (stdout)

```bash
swift run apicov diff old-coverage.json coverage.json
```

새로 붙은 엔드포인트와 떨어져 나간 엔드포인트를 보여줍니다. CI 에서 직전 스캔 결과와 비교할 때 씁니다.

### 5. `swift run stella` — macOS GUI

```bash
swift run stella
```

같은 스캔을 폼에서 돌리고 결과를 화면에서 훑습니다. 매칭 안 된 Router case 를 보면서 `overrides.yml` · `owners.yml` 을 편집할 때 주로 씁니다. 서버 자격증명은 Keychain 에 둡니다.

## `apicov scan` 플래그

| 플래그 | 설명 |
|--------|------|
| `--openapi-url <URL>` | 스펙을 HTTP 로 받습니다. `--openapi-file` 과 **둘 중 하나는 필수** |
| `--openapi-file <경로>` | 로컬 OpenAPI JSON 을 씁니다. 이쪽을 주면 `--openapi-url`·`--auth-env` 는 무시됩니다 |
| `--auth-env <USER_VAR>:<PASS_VAR>` | **자격증명 값이 아니라 환경변수 이름 두 개**를 콜론으로 이어 받습니다. `--auth-env UMC_API_USER:UMC_API_PASS` 라고 쓰면 두 환경변수를 읽어 Basic Auth 를 만듭니다. 값을 그대로 적으면 셸 히스토리에 비밀번호가 남습니다 |
| `--app-product <경로>` | 레거시 `AppProduct/` 프로젝트 루트. Router 글롭은 `**/Router/*Router.swift` |
| `--umc-app <경로>` | Tuist `UMCApp/` 프로젝트 루트. Router 글롭은 `**/Data/Sources/*Router.swift`. `--app-product` 와 **최소 하나는 필수** |
| `--blame-root <경로>` | git blame 을 돌릴 레포 루트. 생략하면 현재 디렉터리라, 소비자 레포를 스캔할 때는 **반드시 지정해야** 담당자가 붙습니다 |
| `--authors <경로>` | `authors.yml` — blame 이메일을 표시명으로 옮기는 표 |
| `--overrides <경로>` | `overrides.yml` — 자동 매칭이 놓친 Router case 보정 |
| `--owners <경로>` | `owners.yml` — 엔드포인트별/태그별 담당자 |
| `--out`, `-o <경로>` | 스냅샷 출력 경로. 생략하면 stdout |

경로 해석은 두 갈래입니다. `/` 나 `.` 으로 시작하면 **현재 디렉터리** 기준이고(위 예시의 `../umc-product-iOS/AppProduct` 가 여기 해당), 그냥 이름만 주면 `--blame-root` 기준입니다.

## 매핑 YAML

레포 루트의 세 파일이 운영본이며 git 에 커밋돼 있습니다. `*.yml.example` 은 템플릿, `Fixtures/` 는 테스트 리소스 번들(`Package.swift` 의 `.copy("../../Fixtures")`)이라 운영 매핑이 아닙니다.

| 파일 | 역할 |
|------|------|
| `overrides.yml` | 자동 매칭 실패 Router case 보정 — OpenAPI 키로 강제 매핑하거나 외부 API 를 `ignore` |
| `authors.yml` | git blame 이메일 → 표시명·GitHub username |
| `owners.yml` | 엔드포인트별/태그별 담당자 (이메일은 `authors.yml` 로 표시명 해석) |

## 산출 타깃

| 이름 | 종류 | 역할 |
|------|------|------|
| `StellaCore` | 라이브러리 | OpenAPI 파싱 · Router 스캔 · 매칭 · blame · 스냅샷 |
| `StellaTestSupport` | 라이브러리 | 테스트 헬퍼 — 샌드박스 git 레포, URL 스텁 |
| `apicov` | 실행 파일 | CLI — `scan` · `report` · `diff` |
| `stella` | 실행 파일 | macOS GUI |

CLI 와 GUI 는 둘 다 `StellaCore` 를 얇게 감쌉니다. 로직은 한 곳에만 있습니다.

## 레포 구조

```
Package.swift                              SwiftPM 매니페스트 (레포 루트)
Sources/StellaCore/                        스캐너·매처·OpenAPI 파서·스냅샷
Sources/apicov/                            CLI
Sources/StellaApp/                         macOS GUI
Sources/StellaTestSupport/                 테스트 헬퍼
Tests/                                     StellaCoreTests · apicovTests · StellaAppTests
Fixtures/                                  테스트 리소스 (MiniOpenAPI.json · MiniRepo · 픽스처 yml)
authors.yml · owners.yml · overrides.yml   운영 매핑 (git 커밋)
*.yml.example                              매핑 파일 템플릿
scripts/build-app.sh                       `.app` 번들 패키징
```

## 개발

```bash
swift test                                    # 전체 (XCTest)
swift test --filter PathNormalizerTests       # 하나만
scripts/build-app.sh                          # dist/Stella.app 패키징
```

`.app` 번들은 ad-hoc 서명이라 로컬 실행용입니다. 자세한 내용은 [개발 가이드][development]와 [CONTRIBUTING.md](CONTRIBUTING.md) 에 있습니다.

## 더 보기

| 문서 | 내용 |
|------|------|
| [설치와 첫 스캔](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Getting-Started) | 빌드부터 첫 `coverage.json` 까지, 경로 해석 규칙 |
| [CLI 레퍼런스](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/CLI-Reference) | `scan` · `report` · `diff` 플래그 전체 |
| [Stella 앱](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Stella-App) | GUI 화면 구성, 단축키, YAML 동기화 |
| [매핑 YAML](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Mapping-YAML) | 세 파일의 스키마와 예시 |
| [매칭 파이프라인](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Matching-Pipeline) | 스캔이 실제로 하는 일 |
| [스냅샷 스키마](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Snapshot-Schema) | `coverage.json` 필드 정의 |
| [소비자 레포 연동](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Consumer-Integration) | CI 워크플로, 새 레포 붙이기 |
| [트러블슈팅](https://github.com/UMC-PRODUCT/umc-product-stella/wiki/Troubleshooting) | 증상별 원인과 조치 |

## 라이선스

MIT 라이선스를 따릅니다. 전문은 [LICENSE](LICENSE) 를 보세요.

지식 재산권은 제옹(euijjang97) 에게 있습니다.
