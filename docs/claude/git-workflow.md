# Git Workflow

> 브랜치 전략, 커밋/PR 규칙 상세 레퍼런스.
> 핵심 요약은 `CLAUDE.md` 참고.

단일 `main` 브랜치 + **연속 브랜치 파생** 지원

- 작성자: 제옹(euijjang97)

## 브랜치 전략

- **브랜치명은 `{타입}/{이슈번호}`** — 타입은 이슈 템플릿과 1:1(`feat`/`bug`/`design`/`refac`/`docs`/`chore`),
  base·PR 대상은 `main`. 예: `docs/1203`, `feat/1195`. 설명형 브랜치명 금지.
  - 대응 이슈가 없으면 **브랜치를 만들기 전에 이슈부터 생성**한다 (아래 "이슈 생성 규칙").
  - PR 제목은 `{이모지} [Type] {작업 내용} (#이슈번호)` (아래 "PR 제목 형식"), 본문에 `Closes #이슈번호`.
  - 푸시한 브랜치명을 고쳐야 하면 GitHub 브랜치 rename API가 **열려 있던 PR을 닫아버린다.**
    rename 후 새 PR을 만들고, 닫힌 PR에 후속 PR 번호를 코멘트로 남긴다.
- **연속 브랜치**: feature에서 다음 feature 파생 가능 (티켓 단위 분리)
- **PR 대기 중 작업**: 승인 대기 중 이전 브랜치에서 다음 브랜치 생성 가능
- **동기화**: main에서 merge 대신 `fetch + rebase` 사용

## 커밋 형식

`{이모지} [Type] 한 줄 요약` — PR 제목에서 `(#이슈번호)` 만 뺀 형태입니다. 한국어로 씁니다.
아래 「PR 제목 형식」 표의 이모지·Type 을 그대로 씁니다.

```
📄 [Docs] README 를 독립 레포 기준으로 재작성
🐛 [Bug] 테스트 픽스처 overrides.yml 을 빈 상태로 복원
🔧 [Chore] 운영 매핑 YAML 3종을 레포 루트로 통합
```

본문이 필요하면 제목 한 줄 + 빈 줄 + 변경사항 bullet list 로 씁니다.
`git log --oneline` 으로 최근 커밋을 보고 맞추면 됩니다.

**커밋 메시지에 `Co-Authored-By` 라인을 절대 추가하지 마세요.**
"Generated with Claude Code" 등 AI가 작성했음을 드러내는 문구도 커밋 메시지에 넣지 않습니다.

## PR 제목 형식

`{이모지} [Type] {작업 내용} (#이슈번호)` — **분류는 반드시 `[대괄호]`**, 제목 끝에 이슈번호.

- ✅ `✨ [Feat] Router 스캔 프로젝트 플래그 일반화 — macOS 레포 지원 (#42)`
- ✅ `📄 [Docs] README 를 독립 레포 기준으로 재작성 (#37)`
- ❌ `📄 Docs: README 를 독립 레포 기준으로 재작성 (#37)` — **이슈 제목 형식**(콜론)을 PR에 쓴 경우
- ❌ `📄 [Docs]: …` — 대괄호 뒤 콜론 금지 / ❌ `[Feature]` — Feat로 통일 / ❌ 이슈번호 누락

> **이슈 제목과 PR 제목은 형식이 다르다.**
> 이슈 = `{이모지} {Type}: {내용}` (콜론, 이슈 템플릿의 title prefix 그대로)
> PR = `{이모지} [{Type}] {내용} (#이슈번호)` (대괄호 + 이슈번호)

| PR `[Type]` | 이모지 | 브랜치 | 이슈 제목 접두사 | 라벨 |
|-------------|--------|--------|------------------|------|
| `[Feat]` | ✨ | `feat/{이슈번호}` | `✨ Feature: ` | `:sparkles: Feature` |
| `[Fix]` | 🐛 | `bug/{이슈번호}` | `🐛 Bug: ` | `:bug: Bug` |
| `[Refactor]` | ♻️ | `refac/{이슈번호}` | `♻️ Refactor: ` | `:hammer: Refactor` |
| `[Design]` | 💄 | `design/{이슈번호}` | `🎨 Design: ` | `:lipstick: UI` |
| `[Docs]` | 📄 | `docs/{이슈번호}` | `📄 Docs: ` | `:page_facing_up: Docs` |
| `[Chore]` | 🔧 | `chore/{이슈번호}` | `🍀 ETC: ` | `:wrench: chore` |
| `[Test]` | ✅ | 관련 이슈 타입을 따름 | (전용 템플릿 없음) | (라벨 없음) |

- 여러 성격이 섞이면 변경 비중이 가장 큰 Type을 제목에 쓰고, 부수 라벨을 추가로 붙인다.

> ⚠️ **이 레포에는 `:page_facing_up: Docs` 하나만 만들어져 있다.** 나머지는 쓰기 전에 먼저 만든다 —
> 없는 라벨을 넘기면 `gh pr create` 가 실패한다.
>
> ```bash
> gh label create ":sparkles: Feature" --color a2eeef --description "새 기능을 추가합니다."
> gh label create ":bug: Bug"          --color d73a4a --description "버그를 수정합니다."
> gh label create ":hammer: Refactor"  --color fbca04 --description "코드 구조를 개선합니다."
> gh label create ":lipstick: UI"      --color f9d0c4 --description "UI/디자인을 반영합니다."
> gh label create ":wrench: chore"     --color ededed --description "기타 작업입니다."
> ```

## PR 본문 형식

**PR 본문은 `.github/pull_request_template.md` 의 섹션 구조를 그대로 따른다.**
자기만의 목차(`## 무엇을`, `## 검증`, `## 넘기는 것` 등)를 새로 만들지 않는다.

| 섹션 | 내용 |
|------|------|
| `## 🔗 관련 이슈` | `Closes #이슈번호` 를 한 줄에 하나씩 |
| `## ✨ PR 유형` | 변경 성격 한두 줄 (제목의 `[Type]` 과 일치) |
| `## 📷 스크린샷 or 영상(UI 변경 시)` | UI 변경이면 필수. 아니면 `UI 변경 없음` 한 줄 |
| `## 🛠️ 작업내용` | 실제 작업. 이슈가 여러 건이면 `### #이슈번호 — 제목` 으로 나눈다 |
| `## 📋 추후 진행 상황` | 후속 작업·다음 PR로 넘기는 것·미완료 항목과 사유 |
| `## 📌 리뷰 포인트` | 리뷰어가 집중해서 볼 지점. 빌드·테스트 검증 결과도 여기 |
| `## ✅ Checklist` | 템플릿 항목을 그대로 두고 `[x]` 로 체크 |

- 섹션은 **빠뜨리지 않는다.** 해당 없으면 지우지 말고 `해당 없음` 을 적는다.
- 템플릿의 `<!-- 안내 주석 -->` 은 내용을 채우면서 지운다.

> ⚠️ **`gh pr create --body "..."` 는 템플릿을 자동으로 불러오지 않는다.**
> 템플릿은 GitHub 웹 UI 에서만 자동 적용된다. CLI 로 PR 을 만들 때는 반드시 템플릿을 직접 채워야 한다:
>
> ```bash
> # 템플릿을 복사해 채운 뒤 --body-file 로 넘긴다
> cp .github/pull_request_template.md /tmp/pr-body.md
> # ... /tmp/pr-body.md 편집 (섹션 유지, 주석 제거, 내용 작성) ...
> gh pr create --base main \
>   --title "📄 [Docs] 작업 내용 (#37)" \
>   --assignee "@me" \
>   --label ":page_facing_up: Docs" \
>   --body-file /tmp/pr-body.md
> ```
>
> `--assignee`·`--label` 은 **생략하지 않는다** (위 「PR 규칙」 첫 항목).
>
> 이미 만든 PR 의 본문을 고칠 때도 같다: `gh pr edit <번호> --body-file /tmp/pr-body.md`

## PR 규칙

- **Assignee 와 라벨은 PR 생성 시점에 반드시 지정한다** — 나중에 붙이는 게 아니라 `gh pr create` 플래그로 같이 넘긴다.
  라벨이 없으면 보드·라벨 필터에서 PR 이 새어나가고, Assignee 가 없으면 담당자 추적이 끊긴다.
  - **Assignee**: 기본 `--assignee "@me"`(PR 작성자). 다른 사람이 이어받으면 `--assignee <github-login>`,
    여러 명이면 콤마로 구분(`--assignee "alice,bob"`).
  - **라벨**: 제목의 `[Type]` 에 대응하는 값 — 위 「PR 제목 형식」 표의 `라벨` 열을 그대로 쓴다.
    여러 성격이 섞이면 `--label` 을 반복해 복수 지정한다.
  - 라벨 이름은 `gh label list` 출력 문자열(`:page_facing_up: Docs` 처럼 이모지 코드 포함)과
    **정확히 일치**해야 한다. 틀리면 `gh pr create` 자체가 실패한다.
  - 이미 만든 PR 에 누락됐다면: `gh pr edit <번호> --add-assignee "@me" --add-label ":page_facing_up: Docs"`
- 최소 1인 Approve 필수
- `main` 직접 푸시 금지
- Squash and Merge 사용
- **PR 제목·본문에 AI 작성 흔적 금지** — `🤖 Generated with [Claude Code](...)` 푸터, `Co-Authored-By` 크레딧 등
  attribution 문구를 절대 넣지 않는다

## 이슈 생성 규칙

이슈는 **제목 접두사 + 라벨 + 이슈 Type + 보드(#3)·`우선순위` + 네이티브 `Priority`·`Effort`** 를
**기본으로 모두 채워서** 생성한다. 날짜(Start/Target date)만 팀 일정이 있을 때 채운다. (`/create-issue` 스킬이 자동화)
이슈 제목·본문에도 "Generated with Claude Code" 등 **AI 작성 흔적(attribution) 문구를 절대 넣지 않는다.**

| 템플릿 | 제목 접두사 | 라벨 | 이슈 Type |
|--------|------------|------|-----------|
| 버그 수정 | `🐛 Bug: ` | `:bug: Bug` | `Bug` |
| 기능 추가 | `✨ Feature: ` | `:sparkles: Feature` | `Feature` |
| 디자인 반영 | `🎨 Design: ` | `:lipstick: UI` | `Task` |
| 리팩토링 | `♻️ Refactor: ` | `:hammer: Refactor` | `Task` |
| 문서 작업 | `📄 Docs: ` | `:page_facing_up: Docs` | `Task` |
| 기타 작업 | `🍀 ETC: ` | `:wrench: chore` | `Task` |

### Type / Priority / Projects

- **이슈 Type** (조직 레벨): 현재 `Task` / `Bug` / `Feature` 3종만 존재 → Bug/Feature 외 템플릿은 `Task`로 매핑.
  `gh issue create`엔 `--type` 플래그가 없으므로(gh 2.83.1) **생성 직후 REST로 설정**한다:
  ```bash
  gh api --method PATCH repos/UMC-PRODUCT/umc-product-stella/issues/{번호} -f type=Feature
  ```
- **보드 #3 + `우선순위`(Projects v2)**: 생성 시 **기본으로 보드 추가 + 우선순위 설정**. 단 `project` 스코프 필요 —
  없을 때만 이 부분을 건너뛰고(Type/라벨은 적용) `gh auth refresh -s project` 후 재적용.
  - iOS 보드 = 조직 Projects **#3 `iOS 개발 프로젝트 템플릿`**, 우선순위 필드명은 영어가 아닌 한글 **`우선순위`**.
  - `gh project item-list` JSON은 한글 단일선택 필드를 노출하지 않음 → 설정 검증은 GraphQL `fieldValueByName("우선순위")` 사용.
- **네이티브 이슈 Fields**(베타, 사이드바 "Fields"): 보드와 별개인 레포 이슈 자체 필드 `Priority`/`Effort`/`Start date`/`Target date`.
  생성 시 **`Priority`·`Effort` 기본 설정**(날짜는 비움). `gh` 명령 없이 GraphQL `setIssueFieldValue` mutation 사용 (`/create-issue` 스킬 4-4·4-5에 id 캐싱).
  - 보드 #3의 한글 `우선순위`와 **다른 필드**(영어 `Priority`)지만, **같은 '우선순위 레벨'로 일치**시켜 채운다: 최고→Urgent/🔥, 높음→High/🔨, 보통→Medium/🤔, 낮음→Low/💬.
- 기본 assignee는 `@me`. 다른 담당자 지정 또는 미지정은 명시적으로 요청된 경우만.
