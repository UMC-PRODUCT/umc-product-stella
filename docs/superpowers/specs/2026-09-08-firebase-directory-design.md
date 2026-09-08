# Firebase 기반 접근 통제 · 담당자 원격 관리 설계

- 작성일: 2026-09-08
- 작성자: 제옹(euijjang97)
- 상태: 확정 — 브레인스토밍에서 결정된 내용을 문서화한 것. 새 설계 제안이 아니다.
- 개정: 2026-09-08 — `identifier` 를 `identifiers` 배열로 변경, 미정 10건 중 9건 결정 반영(§15-2), GUI 스캔 경로 공백을 `ScanConfig` 값 주입으로 해소(§10-5 · §16-8)
- 대상: `umc-product-stella` (SwiftPM · macOS 15+ · Swift 6)

## 1. 목적

Stella 는 지금 아무나 실행할 수 있다. 담당자 정보는 레포 루트의 `owners.yml` · `authors.yml` 을 손으로 커밋해서 관리한다. 이 문서는 두 가지를 바꾼다.

1. **접근 통제** — 앱 실행 시 "발급 아이디" 하나를 입력해야 들어올 수 있다. 팀원이 아니면 못 쓴다.
2. **담당자 원격 관리** — `owners.yml` · `authors.yml` 의 역할을 Firestore 로 옮긴다. 두 파일은 생성물로 강등한다.

`CoverageSnapshot` 은 손대지 않는다. `StellaCore` 기존 코드 변경은 `ScanConfig` 옵셔널 필드 2개(`owners` · `authors`) · `Pipeline.generate()` 두 줄 · `StellaError` case 3개가 전부다(§10-5). 매칭 로직은 지금처럼 `Owners` · `AuthorMapper` 값 객체만 받는다. 그 값을 어디서 가져오느냐만 바뀐다.

## 2. 범위 / 비범위

| 범위 | 비범위 |
|------|--------|
| Firebase Auth REST 로그인 · 토큰 갱신 | Firebase SDK 도입 (의존성 추가 0개 원칙) |
| Firestore REST 로 users · assignments · tagAssignments 읽기/쓰기 | `overrides.yml` 원격화 — 매칭 설정이라 그대로 커밋 |
| GUI 로그인 게이트 · 관리자 화면(발급 · 수정 · 회수) · 오프라인 진입 | 비밀번호 재설정 · 이메일 인증 등 일반 계정 기능 |
| `apicov owners pull` (CI 용 yml 생성) | `apicov scan` 변경 — `ScanCommand` 무수정, URL 경로 동작 그대로 |
| 수동 마이그레이션 (2명 · 3건) | `push` 서브커맨드 — 규모상 값어치 없음 |
| `StellaCore` 기존 코드: `ScanConfig` 옵셔널 필드 2개 · `Pipeline.generate()` 두 줄 · `StellaError` case 3개 (§10-5) | 소비자 레포(`umc-product-iOS`) CI 워크플로 수정 — §18 후속 과제 |

## 3. 인증 방식

- 고유 ID 가 곧 Firebase Auth 자격증명이다. 앱은 한 필드(`loginId`)만 받아 이메일 `{loginId}@stella.umc.local`, 비밀번호 `{loginId}` 로 조립한다.
- `loginId` 는 32자 랜덤 문자열. 발급 시 앱이 생성한다.
- Firebase SDK 는 안 쓴다. Auth REST + Firestore REST 만 `URLSession` + `Codable` 로 호출한다.
- `idToken` 은 1시간 만료. `securetoken.googleapis.com` 으로 refresh 하며 만료 5분 전에 자동 갱신한다.

사용하는 엔드포인트 (모두 Google 공개 REST):

| 용도 | 엔드포인트 | 비고 |
|------|-----------|------|
| 로그인 | `POST identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key={apiKey}` | 응답의 `localId` 가 uid |
| 계정 생성(발급) | `POST identitytoolkit.googleapis.com/v1/accounts:signUp?key={apiKey}` | 응답 토큰은 버리고 `localId` 만 취함 |
| 토큰 갱신 | `POST securetoken.googleapis.com/v1/token?key={apiKey}` | `grant_type=refresh_token` |
| 문서 읽기/쓰기 | `firestore.googleapis.com/v1/projects/{projectId}/databases/(default)/documents/...` | `Authorization: Bearer {idToken}` |

로그인 뒤 앱은 곧바로 `users/{uid}` 를 읽어 `role` · `active` 를 확인한다. 문서가 없거나 `active: false` 면 보안 규칙이 403 을 돌려주므로(§5) 로그인은 성공해도 진입은 거부된다.

## 4. Firestore 데이터 모델

```
users/{uid}                       # uid = Firebase Auth uid
  identifiers: [string]           # 파이프라인 식별자 배열 — blame 이메일 또는 GitHub username
                                  #   첫 원소가 primary. 1개 이상. 기존 owners.yml 의 owner 값이 여기 들어간다
  displayName: string
  github:      string
  role:        "admin" | "member"
  active:      bool
  # loginId 는 저장하지 않는다 (§6-2)

assignments/{docId}               # docId 규칙은 아래
  method:    string               # 예) "GET"
  path:      string               # 예) "/api/v1/attendances/available"
  ownerUid:  string               # users/{uid} 참조
  updatedBy: string               # 쓴 사람의 uid — 규칙이 request.auth.uid 와 대조
  updatedAt: timestamp

tagAssignments/{tag}
  ownerUid, updatedBy, updatedAt  # assignments 와 동일
```

### 4-1. `identifiers` 가 배열인 이유

한 사람이 이메일 여러 개로 커밋한다. 이 레포만 봐도 `git log --all --format='%an <%ae>' | sort -u` 에 같은 사람이 두 번 나온다.

```
JEONG <ikejhc159@gmail.com>
JEONG EUI CHAN <80624315+JEONG-J@users.noreply.github.com>
```

그런데 `AuthorMappingEntry` 는 이메일 하나당 엔트리 하나다(`Sources/StellaCore/Authors/AuthorMapper.swift`). 식별자를 문자열 하나로 두면 이메일이 여럿인 사람은 계정을 여러 개 만들어야 한다. 배열로 두고 환원 시점에 펼친다.

- 현재 `owners.yml` 의 owner 값은 이메일(`euijjang97@gmail.com`)과 GitHub username 이 섞여 있다. `Pipeline` 은 이 문자열을 `AuthorMapper.author(forEmail:)` 로 그대로 조회한다(`Sources/StellaCore/Pipeline/Pipeline.swift`, `buildEndpoints`). 이 값을 배열 원소로 보존해야 `Pipeline` 을 건드리지 않는다.
- Firestore 안에서는 `ownerUid` 로 참조하고 `Owners` · `AuthorMapper` 로 환원하는 시점에만 식별자로 바꾼다.

환원 규칙:

| 대상 | 규칙 |
|------|------|
| `AuthorMapper` (`authors.yml`) | **identifier 하나당 `AuthorMappingEntry` 하나.** `displayName` · `github` 는 공유. 현행 `authors.yml` 동작이 그대로 보존된다 |
| `Owners` (`owners.yml`) | owner 값은 **`identifiers` 의 첫 원소(primary)** |
| 중복 검사 (§6-6) | 배열 원소 중 **하나라도** 다른 사용자의 `identifiers` 와 겹치면 경고 |

### 4-2. `assignments` docId 정규화

```
encode(method, path):  method + "|" + path.replacing("/", "|") 에서 선행 "|" 하나를 제거
decode(docId):         첫 "|" 앞 = method, 나머지는 "|" → "/" 로 되돌리고 앞에 "/" 를 붙인다
```

예) `GET` + `/api/v1/attendances/available` → `GET|api|v1|attendances|available`
예) `GET` + `/api/v1/attendances/challenger/{challengerId}/history` → `GET|api|v1|attendances|challenger|{challengerId}|history`

- 역변환 가능하다. 전제는 두 가지다. path 는 항상 `/` 로 시작하고 path 에 `|` 가 없다(OpenAPI 경로에는 등장하지 않는다). 선행 구분자만 제거하므로 경로 중간의 `//` 나 끝의 `/` 도 보존된다.
- 그래도 진실은 `method` · `path` 필드다. docId 는 키일 뿐이고 `decode` 는 테스트와 정합성 검사에 쓴다.

## 5. 보안 규칙

여기가 유일한 실제 방어선이다. 클라이언트 쪽 검사는 UX 일 뿐 보안이 아니다.

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    function doc()    { return get(/databases/$(db)/documents/users/$(request.auth.uid)).data; }
    function member() { return request.auth != null && doc().active == true; }
    function admin()  { return member() && doc().role == 'admin'; }

    match /users/{uid}        { allow read: if member(); allow write: if admin(); }
    match /assignments/{d}    { allow read: if member();
                                allow write: if member() && request.resource.data.updatedBy == request.auth.uid; }
    match /tagAssignments/{t} { allow read: if member();
                                allow write: if member() && request.resource.data.updatedBy == request.auth.uid; }
  }
}
```

규칙에서 따라 나오는 동작:

| 상황 | 결과 |
|------|------|
| Auth 계정은 있으나 `users/{uid}` 없음 | `doc()` 이 실패 → 모든 read/write 거부 (403) |
| `active: false` | `member()` 거짓 → 모든 접근 거부 |
| member 가 `users` 쓰기 시도 | 거부 — 표시명 · github · `identifiers` 수정 · 발급 · 회수는 admin 전용 |
| member 가 `assignments` 쓰기 | `updatedBy` 를 자기 uid 로 넣었을 때만 허용 |

규칙은 `identifiers` 의 모양(배열 · 최소 1개 · 중복 없음)을 검사하지 않는다. 그건 앱 단 검사다(§6-6).

## 6. 설계 결정과 대가

알고 받아들인 트레이드오프다. 나중에 "왜 이렇게 했지" 가 나오면 여기를 본다.

1. Auth 가입(signUp)을 끌 수 없다. 앱 내 발급 UI 가 `accounts:signUp` REST 를 쓰기 때문이다. 외부인이 API 키로 계정을 만들 수는 있지만 `users/{uid}` 문서가 없으면 규칙이 전부 거부하므로 데이터 접근은 0 이다. 남는 위험은 Auth 사용자 목록 오염과 쿼터 남용뿐이다. API 키는 어차피 바이너리에서 추출 가능한 공개값이다.
2. `loginId` 는 Firestore 에 저장하지 않는다. `users` 문서는 담당자 이름을 보여주기 위해 모든 member 가 읽는다. `loginId` 가 곧 비밀번호라 저장하면 전원이 서로의 계정으로 로그인할 수 있게 된다. 발급 시 화면에 한 번 표시 + 클립보드 복사로 끝낸다.
3. 분실 시 복구 불가 → 재발급. `accounts:update` 로 남의 비밀번호를 바꾸려면 그 사람의 idToken 이 필요해 관리자가 대신 못 한다. 절차는 옛 계정 `active: false` → 새 계정 발급 → 담당 배정을 새 uid 로 이전.
4. 오프라인 캐시 진입의 대가. `active: false` 로 회수해도 네트워크를 끊은 기기는 캐시로 계속 진입한다(§9). 즉시 끊으려면 Firebase 콘솔에서 Auth 계정 자체를 비활성화한다. 그러면 refresh 가 막혀 다음 온라인 복귀 때 세션이 죽는다.
5. 부트스트랩. 제옹의 첫 admin 계정만 Firebase 콘솔에서 수동 생성한다(닭-달걀). 절차는 §12.
6. `identifiers` 중복 방지는 앱 단 검사. 규칙에서는 쿼리를 못 써서 강제할 수 없다. 발급 · 수정 전 `users` 를 조회해 배열 원소 중 하나라도 다른 사용자와 겹치면 경고한다.
7. 미지의 `ownerUid` 는 조용히 버리지 않는다. `users` 에 없는 uid 를 가리키는 배정은 `Owners` 에서 제외하되 제외 목록을 따로 돌려준다(§16-1). 대가는 코드 한 줄이 아니라 반환 타입 하나다. 배정이 "사라진 것처럼" 보이는 대신 경고로 드러난다.
8. 회수된 사람도 `AuthorMapper` 에는 남는다. `active: false` 여도 blame 표시명이 계속 해석돼야 한다(§16-2). 대가는 회수된 사람 이름이 화면에 계속 보이는 것인데 의도된 동작이다. 배정 Picker 에서만 빠진다.

## 7. 관리자 발급 흐름

제옹이 유일한 admin 이다.

1. 관리자 화면에서 표시명 · `identifiers`(1개 이상, 첫 원소가 primary) · GitHub username 을 입력한다.
2. 앱이 `users` 를 조회해 `identifiers` 의 원소 중 하나라도 다른 사용자와 겹치는지 검사한다 (§6-6).
3. 앱이 32자 랜덤 `loginId` 를 생성한다.
4. `accounts:signUp` 으로 계정을 만든다. 응답에서 uid 만 취하고 토큰은 버린다. REST 를 고른 덕에 가능하다. SDK 였으면 `currentUser` 가 갈아치워져 관리자가 자기 세션을 잃는다.
5. 관리자 토큰으로 `users/{uid}` 를 쓴다 (`role: member`, `active: true`).
6. 발급된 `loginId` 를 한 번 표시하고 클립보드에 복사한다. 이후 어디에도 남지 않는다.

4 단계 성공 후 5 단계가 실패하면 Auth 계정만 남는다. 규칙상 접근은 0 이므로(§6-1) 위험은 없지만 화면에는 "문서 작성 실패 — 다시 시도" 를 띄운다. 재시도는 같은 `loginId` 로 하지 않고 처음부터 다시 발급한다.

## 8. 수정 · 회수 · 재발급

모두 `users` 쓰기라 admin 전용이며 `AdminView` 에서 한다. 현재 `ScanFormView` 에 있는 담당자 편집 섹션은 제거되므로(§10-3) 여기가 유일한 수정 경로다.

- **수정**: 사용자 목록에서 인라인으로 표시명 · GitHub · `identifiers` 원소 추가/삭제 · `active` 토글. `identifiers` 는 비울 수 없다(최소 1개). 첫 원소가 primary 라 삭제로 primary 가 바뀔 수 있으니 목록에 primary 를 표시한다. 원소 추가 시에도 §6-6 중복 검사를 한다.
- **회수**: `active` 토글을 끈다(= `false` 로 PATCH). 다음 요청부터 규칙이 거부한다. 문서는 지우지 않는다. blame 표시명 해석(§6-8)과 과거 배정 이력이 남아야 한다.
- **재발급**: §6-3 절차. 옛 uid 로 걸린 `assignments` · `tagAssignments` 는 `EndpointsView` 의 담당자 Picker 로 새 uid 에 다시 배정한다.

## 9. 오프라인 정책

GUI 에만 해당한다. CLI(`apicov owners pull`)는 캐시를 안 쓴다. CI 는 무상태다.

| 조건 | 세션 상태 | 담당자 편집 | 표시 |
|------|----------|------------|------|
| 갱신 성공 | `.authenticated` | 가능 | — |
| 네트워크 실패 + 캐시 있음 | `.offline` | **비활성화** | "오프라인 · 캐시 기준" 배지 |
| 네트워크 실패 + 캐시 없음 | `.failed(사유)` → 로그인 화면 | — | 오류 문구 |
| 갱신 거부 (계정 비활성 · 토큰 폐기) | `.failed(사유)` → 로그인 화면 | — | Keychain 의 refresh token 삭제 |

- 로그인 성공 시 refresh token 을 Keychain 에 보관한다. 기존 `KeychainStore` 를 그대로 쓰고 `account: "firebaseRefreshToken"`.
- 앱 실행 시 조용히 갱신을 시도하고 성공하면 로그인 화면 없이 진입한다.
- 캐시 위치: `~/Library/Application Support/Stella/directory-cache.json`. 마지막으로 성공한 users · assignments · tagAssignments 조회 결과와 조회 시각을 담는다. `coverage.json` 과 무관한 별도 파일이라 `CoverageSnapshot.currentSchemaVersion` 규칙에 걸리지 않는다. 토큰이나 `loginId` 는 담지 않는다.
- 손상된 캐시는 "캐시 없음" 으로 취급한다. 크래시하지 않는다.

## 10. 코드 구조

### 10-1. `Sources/StellaCore/Remote/` (신규)

| 파일 | 역할 | 비고 |
|------|------|------|
| `FirebaseConfig.swift` | `projectId` · `apiKey` 상수 | `Sendable`. 실제 값은 미정(§15-1) |
| `Session.swift` | `idToken` · `refreshToken` · `uid` · `role` · `expiresAt` | `Sendable` 값 타입 |
| `FirebaseAuthClient.swift` | `signIn(loginId:)` · `refresh(_:)` · `signUp(loginId:) -> uid` | 순수 전송 계층. `URLSession` 주입 |
| `FirestoreREST.swift` | 문서 GET / PATCH(`updateMask`) / DELETE / 컬렉션 목록 + Firestore value(`stringValue` · `booleanValue` · `timestampValue` · `arrayValue` · `mapValue`) ↔ Swift 값 매퍼 | 순수 전송 계층. `URLSession` 주입. **목록 조회는 `nextPageToken` 이 빌 때까지 순회한다** |
| `StellaDirectory.swift` | users · assignments · tagAssignments ↔ `Owners` · `[AuthorMappingEntry]` 변환, docId encode/decode, uid ↔ primary identifier 조회 | 네트워크를 모른다. 클라이언트 프로토콜만 안다 |
| `DirectoryCache.swift` | 마지막 성공 조회 저장 · 복구 | 파일 경로는 §9 |

- `FirebaseAuthClient` · `FirestoreREST` 는 `OpenAPILoader` 와 같은 방식으로 `init(session: URLSession = .shared)` 를 받아 `StubURLProtocol` 로 테스트한다.
- 페이지네이션은 YAGNI 가 아니라 정확성 문제다. Firestore REST 목록 조회는 페이지 크기를 서버가 정하므로 `nextPageToken` 을 따라가지 않으면 목록이 조용히 잘린다. 지금 규모(2 · 3)에서는 한 페이지지만 루프는 처음부터 넣는다.
- `StellaDirectory` 가 하는 변환(§4-1 규칙):
  - `users` → `[AuthorMappingEntry]` — 사용자 한 명의 `identifiers` 원소마다 엔트리 하나(`email: 원소, displayName, github` 공유). `active` 와 무관하게 전원.
  - `assignments` → `Owners.endpointOwners[OpenAPIKey(method, path)] = users[ownerUid].identifiers.first`
  - `tagAssignments` → `Owners.tagOwners[tag] = users[ownerUid].identifiers.first`
  - `users` 에 없는 `ownerUid` 는 제외하고 제외 목록을 돌려준다(§16-1)
  - 쓰기: assignment upsert / delete(`updatedBy` = 호출자 uid, `updatedAt` = 현재 시각), user 문서 작성 · 필드 PATCH(`displayName` · `github` · `identifiers` · `active`)
  - `AuthorMappingEntry.email` 에 identifier 원소를 넣는다. 필드명은 legacy 지만 `Pipeline` 이 이 이름으로 조회하므로 바꾸지 않는다.
- 절대 규칙 ①: 이 모듈 어디에도 `SwiftUI` · `AppKit` 이 들어가지 않는다. CLI 가 그대로 쓴다.
- 절대 규칙 ③: 스냅샷을 오가는 건 없지만 `Session` · 디렉터리 값 타입은 모두 `Sendable`.

### 10-2. `StellaError` 추가 (절대 규칙 ②)

```swift
case authenticationFailed(String)   // Auth 400 계열 — EMAIL_NOT_FOUND · INVALID_LOGIN_CREDENTIALS · USER_DISABLED · TOKEN_EXPIRED
case remoteUnavailable(String)      // URLError · 5xx — 오프라인 판정의 입력
case forbidden                      // Firestore 403 — 미등록 · 비활성 · 권한 없음
```

### 10-3. `Sources/StellaApp/` (신규 3개 + 수정)

| 파일 | 상태 | 내용 |
|------|------|------|
| `SessionModel.swift` | 신규 | `@MainActor @Observable`. 로그인 · 실행 시 자동 복구 · 5분 전 갱신 타이머 · 오프라인 판정 · `role` 노출. 상태는 아래 표 |
| `LoginView.swift` | 신규 | 단일 필드 "발급 아이디" + 로그인 버튼 + `.failed` 사유 표시 |
| `AdminView.swift` | 신규 | 발급 · 사용자 목록 · **인라인 수정**(표시명 · GitHub · `identifiers` 추가/삭제 · `active` 토글) · 회수. `role == admin` 일 때만 `SidebarItem` 에 노출 |
| `StellaApp.swift` | 수정 | `body` 에서 세션 상태로 `LoginView` / `ContentView` 분기. "owners.yml 저장…" 메뉴를 "담당자 YAML 내보내기…" 로 |
| `ContentView.swift` | 수정 | `SidebarItem` 에 admin 항목 추가. `allCases` 를 그대로 나열하는 현재 구조에 `role` 필터를 얹는다. `.offline` 배지 |
| `ScanModel.swift` | 수정 | 아래 |
| `ScanFormView.swift` | 수정 | `metadataSection` 의 `authors.yml` · `owners.yml` 행 제거(`overrides.yml` 행은 유지). 담당자 편집 섹션(`upsertOwner` · `deleteOwner`) 제거 — `users` 쓰기라 member 는 항상 403 이므로 `AdminView` 로 옮긴다 |
| `EndpointsView.swift` | 수정 | `ownerPicker` 는 `active` 사용자만, `.offline` 이면 비활성화 |

세션 상태:

| 상태 | 의미 | 화면 |
|------|------|------|
| `.signedOut` | Keychain 에 refresh token 없음 (첫 실행 · 로그아웃) | `LoginView` |
| `.authenticating` | 로그인 또는 실행 시 자동 복구 진행 중 | `LoginView` 에 진행 표시 |
| `.authenticated` | 온라인. 갱신 타이머 동작 | `ContentView`, 편집 가능 |
| `.offline` | 갱신 실패 + 캐시 있음 | `ContentView`, 편집 불가, 배지 |
| `.failed(String)` | 로그인 · 복구 실패 사유. 갱신 거부면 Keychain 을 비운 뒤 진입 | `LoginView` + 사유 문구 |

`ScanModel` 변경은 두 함수 + `buildConfig()` 의 인자 한 곳이다.

| 현재 | 이후 |
|------|------|
| `loadOwnerYAMLFiles()` — `authorsPath` · `ownersPath` 파일을 읽어 `manualOwners` · `ownerAssignments` 채움 | 원격 조회(`StellaDirectory`) → 실패 시 `DirectoryCache` → 같은 두 프로퍼티와, 스캔에 넘길 `Owners` · `AuthorMapper` 값을 함께 보관한다 |
| `saveOwnerYAMLFiles(authors:owners:)` — 파일에 씀 | `assignOwner` 는 `assignments` PATCH/DELETE (`updatedBy` = 내 uid). `upsertOwner` · `deleteOwner` 는 `AdminView` 로 이동 |
| `buildConfig()` — `authorsURL: optionalFile(authorsPath)` · `ownersURL: optionalFile(ownersPath)` | 원격/캐시에서 얻은 `Owners` · `AuthorMapper` 를 `ScanConfig` 의 `owners` · `authors` 에 **직접 주입**. `authorsURL` · `ownersURL` 은 `nil` (§10-5 · §16-8) |

`manualOwners: [EditableOwner]` 는 사용자당 하나, `email` = primary identifier(§16-4). `ownerAssignments: [String: String]`(키 `"METHOD path"`, 값 primary identifier)의 모양도 유지한다. `applyingManualOwners(to:)` · `ownerPicker` 가 그대로 돈다. 배정을 원격에 쓸 때는 `StellaDirectory` 의 primary identifier → uid 조회로 `ownerUid` 를 얻는다.

"담당자 YAML 내보내기…" 메뉴(`saveOwnersYAML()` 개명)는 내보내기 전용으로 존치하며 CLI `pull` 과 맞춰 `owners.yml` · `authors.yml` 두 파일을 쓴다. 저장 위치는 폴더로 고른다(§16-7). 지금은 대상 파일의 기존 `tags` 를 읽어 병합하는데 이후에는 `tags` 도 `tagAssignments` 에서 오므로 병합 단계는 없어진다. `canExportOwnersYAML` 은 배정이나 사용자 중 하나라도 있으면 참.

### 10-4. `Sources/apicov/Commands/OwnersCommand.swift` (신규)

```bash
apicov owners pull \
  --login-id-env STELLA_LOGIN_ID \
  --out-owners owners.yml \
  --out-authors authors.yml
```

- 자격증명은 환경변수 이름만 받는다 (`--auth-env` 와 같은 관례). Keychain 은 CLI 바이너리의 서명이 GUI 와 달라 프롬프트가 뜨므로 쓰지 않는다.
- 흐름: env 읽기 → `FirebaseAuthClient.signIn` → `FirestoreREST` 로 세 컬렉션 조회(페이지 순회 포함) → `StellaDirectory` 변환 → `OwnersLoader.serialize` · `AuthorMapper.serialize` → 파일 쓰기.
- 산출물: `owners.yml` 의 owner 값은 primary identifier, `authors.yml` 은 identifier 원소마다 엔트리 하나(§4-1).
- 실패하면 `StellaError` 를 그대로 던지고 0 이 아닌 종료 코드. 캐시 없음.
- `ScanCommand` 는 무수정이다. URL 경로로 계속 읽는다(§10-5). CI 는 두 줄이다:

```bash
apicov owners pull --login-id-env STELLA_LOGIN_ID --out-owners owners.yml --out-authors authors.yml
apicov scan ... --owners owners.yml --authors authors.yml
```

`Apicov.swift` 의 `subcommands` 에 `OwnersCommand.self` 를 추가한다.

### 10-5. `ScanConfig` · `Pipeline` (최소 변경)

GUI 가 파일 경로를 없애면 `Pipeline` 이 읽을 `authors.yml` · `owners.yml` 이 없어 스캔 결과의 blame 표시명 · owner 가 빈다(§16-8). 해법은 이미 로드된 값을 `ScanConfig` 가 직접 받는 것이다.

```swift
// Sources/StellaCore/Pipeline/ScanConfig.swift — 기본값 nil 인 추가 필드. 기존 호출부(ScanCommand · ScanModel · PipelineTests) 전부 무수정
public let owners: Owners?
public let authors: AuthorMapper?
```

```swift
// Sources/StellaCore/Pipeline/Pipeline.swift — generate() 안 두 줄 교체
let authors = try config.authors ?? config.authorsURL.map { try AuthorMapper.load(from: $0) } ?? .empty
let owners  = try config.owners  ?? config.ownersURL.map  { try OwnersLoader.load(from: $0) }  ?? .empty
```

- 값이 주어지면 그걸 쓰고 없으면 기존대로 URL 에서 읽는다. CLI `scan` 은 URL 경로 그대로라 동작이 바뀌지 않는다. GUI 만 값을 넘긴다.
- `try` 는 문장 앞에 둔다. Swift 가 `??` 오른쪽에 `try` 를 허용하지 않아서다. 현행 26~28행과 같은 꼴이다.
- 이게 최소 diff 인 이유. 메모리에 있는 값을 파일로 썼다가 `Pipeline` 이 다시 읽는 왕복이 사라진다. 같은 폴더의 `directory-cache.json` 과 같은 데이터가 두 표현으로 중복될 일도 없다.
- 전제(코드 확인): `ScanConfig: Sendable` (`ScanConfig.swift:35`), `Owners: Sendable, Equatable` (`OwnersLoader.swift:4`), `AuthorMapper: Sendable` (`AuthorMapper.swift:16`). 필드를 추가해도 `ScanConfig: Sendable` 이 유지된다. `ownersURL` 이 이미 `= nil` 기본값이라 같은 방식으로 init 끝에 붙이면 호출부가 안 깨진다.
- 절대 규칙 ①(코어는 UI 를 모른다): `Owners` · `AuthorMapper` 는 코어 타입. ③(Sendable): 위 전제. ④(스냅샷 스키마): `coverage.json` 무관, `currentSchemaVersion` 유지.

## 11. 현행 코드와의 접점 요약

| 건드리는 것 | 안 건드리는 것 |
|------------|---------------|
| `StellaError` (case 3개 추가) · `ScanConfig` (옵셔널 필드 2개) · `Pipeline.generate()` (두 줄) | `ProjectInput` · `Pipeline` 의 URL 로드 경로(CLI 가 쓰는 길) |
| `StellaApp.body` 분기 · 메뉴 문구 | `CoverageSnapshot` · `SnapshotEncoder` · schema version |
| `ContentView.SidebarItem` | `OwnersLoader` · `AuthorMapper` (읽기 · 직렬화 로직 그대로) |
| `ScanModel` 두 함수 + `buildConfig()` 인자 · `ScanFormView` 두 섹션 · `EndpointsView.ownerPicker` 필터 | `applyingManualOwners` · `EditableOwner` |
| `Apicov.subcommands` | `ScanCommand` · `DiffCommand` · `ReportCommand` |
| `.gitignore` · `CLAUDE.md` · `README.md` · `CONTRIBUTING.md` | `overrides.yml` · `*.yml.example` · `Fixtures/` |

## 12. 마이그레이션

현행 데이터 중 사용자 2명 · 배정 3건만 이관한다. `push` 서브커맨드를 만들 값어치가 없으므로 관리자 화면에서 수동으로 한다.

### 12-1. 부트스트랩 (Firebase 콘솔, 코드 무관)

1. Firebase 프로젝트 · Firestore(Native mode, 리전 `asia-northeast3` 서울) 준비. projectId 는 이때 확정(§15-1)
2. Authentication → Sign-in method → 이메일/비밀번호 활성화
3. Authentication 사용자 추가: 이메일 `{loginId}@stella.umc.local`, 비밀번호 `{loginId}` (32자 랜덤을 직접 만든다)
4. Firestore 에 `users/{uid}` 수동 작성:
   `identifiers: ["euijjang97@gmail.com"]` · `displayName: 제옹` · `github: JEONG-J` · `role: admin` · `active: true`
5. §5 규칙 배포
6. `FirebaseConfig.swift` 에 projectId · apiKey 기입

### 12-2. 사용자 이관 (관리자 화면)

| 표시명 | `identifiers` | github | role | 방법 |
|--------|--------------|--------|------|------|
| 제옹 | `["euijjang97@gmail.com"]` | `JEONG-J` | **admin** | 콘솔 부트스트랩 (12-1) |
| 원 | `["One@One"]` | `One` | member | 앱 발급 |

- 원의 `One@One` 은 유효한 blame 이메일이 아니라 blame 매칭이 안 된다. 지금 막을 필요는 없다. 배열이라 실제 커밋 이메일을 알게 되면 `AdminView` 에서 원소를 추가하면 된다.
- 제옹도 이메일이 여럿이다. 이 레포에서 `ikejhc159@gmail.com` · `80624315+JEONG-J@users.noreply.github.com` 으로 커밋한 이력이 있다. 소비자 레포(`umc-product-iOS`)에서 `git log --format='%ae' | sort -u` 로 실제 blame 이메일을 확인하고 필요하면 `identifiers` 에 추가한다.
- CI 계정 1개를 앱에서 발급한다: `identifiers: ["stella-ci"]` · `role: member` · `active: true`. 배정을 받지 않으니 유일하기만 하면 되고 표시명 · github 는 표시될 일이 없다. 발급된 `loginId` 를 소비자 레포 secret `STELLA_LOGIN_ID` 로 전달한다(§18). `active` member 라 배정 Picker 에 나타나는데 배정하지 않으면 그만이다.

### 12-3. 배정 이관 (관리자 화면, 현행 `owners.yml` 기준)

| method | path | 이관 후 담당자 |
|--------|------|---------------|
| GET | `/api/v1/attendances/available` | 제옹 |
| GET | `/api/v1/attendances/challenger/{challengerId}/history` | 제옹 |
| GET | `/api/v1/attendances/pending` | 제옹 |
| POST | `/api/v1/attendances/check` | **없음** — 현행 담당 소피(`LeeYeJi546`)를 이관하지 않으므로 비워 두고, 제옹이 GUI 에서 재배정 |

`tags` 배정은 현재 없다.

### 12-4. 빠지는 데이터

소피(`LeeYeJi546`) · 도도(`dodo@example.com`) 계정은 만들지 않는다. 나중에 실제 팀원으로 확인되면 관리자 화면에서 발급하면 된다.

### 12-5. 검증

`apicov owners pull` 로 내려받은 `owners.yml` · `authors.yml` 이 §12-2 · §12-3 표와 일치하는지 본다. 이관 전 커밋본과 diff 하면 차이가 정확히 다음뿐이어야 한다.

- `owners.yml`: `POST /api/v1/attendances/check` 항목 부재(재배정 전) 또는 제옹(재배정 후)
- `authors.yml`: 소피 · 도도 엔트리 부재, `stella-ci` 엔트리 추가, 제옹 `identifiers` 를 늘렸다면 그만큼 엔트리 추가

## 13. 파일 운명 · 문서 갱신

| 파일 | 이후 |
|------|------|
| `owners.yml` · `authors.yml` | 생성물로 강등 → `.gitignore` 추가, 커밋 중단. git 에서 제거(`git rm --cached`) |
| `*.yml.example` | 유지 |
| `overrides.yml` | 그대로 커밋 (매칭 설정, 원격화 대상 아님) |
| `Fixtures/owners.yml` · `Fixtures/authors.yml` | 테스트 리소스 — **유지**. `.gitignore` 패턴은 루트 고정(`/owners.yml` · `/authors.yml`)으로 써서 픽스처를 삼키지 않게 한다 |

갱신할 문서:

- `CLAUDE.md` "매핑 YAML" 절 — 세 파일이 아니라 `overrides.yml` 하나만 운영본. 나머지 둘은 `apicov owners pull` 생성물
- `README.md` — 빠른 시작에 `owners pull` 추가, 매핑 YAML 표 · 레포 구조 수정
- `CONTRIBUTING.md` — "커밋하지 않는 것" 절 수정
- Wiki `Mapping-YAML` · `CLI-Reference` · `Stella-App` · `Consumer-Integration` — 별도 레포이므로 이슈 본문에 체크 항목으로만 둔다

## 14. 테스트

`StubURLProtocol` 을 재사용한다. 디렉터리는 소스 구조를 따라간다.

| 테스트 파일 | 케이스 |
|------------|--------|
| `Tests/StellaCoreTests/Remote/FirebaseAuthClientTests.swift` | 로그인 성공 / 실패(400) / 토큰 갱신 / 만료 판정(`expiresAt` - 5분) |
| `Tests/StellaCoreTests/Remote/FirestoreRESTTests.swift` | value 인코딩 · 디코딩 왕복 (string · bool · timestamp · **array** · 중첩 map) / 403 → `.forbidden` / **페이지 경계** — 첫 응답에 `nextPageToken`, 둘째에 없음 → 두 페이지 문서가 모두 모인다 |
| `Tests/StellaCoreTests/Remote/StellaDirectoryTests.swift` | users + assignments + tagAssignments → `Owners` · `[AuthorMappingEntry]` 변환 / **`identifiers` 2개인 사용자 → 엔트리 2개, owner 값은 첫 원소** / 미지의 `ownerUid` 제외 + 제외 목록 / **docId encode → decode 왕복** (`{param}` 포함 경로) |
| `Tests/StellaCoreTests/Remote/DirectoryCacheTests.swift` | 저장 · 복구 · 손상 파일 |
| `Tests/StellaCoreTests/Pipeline/PipelineTests.swift` (기존 파일에 추가) | ① `config.owners` · `config.authors` 를 주고 `authorsURL` · `ownersURL` 에 존재하지 않는 경로를 넣어도 throw 없이 값 쪽이 쓰인다 ② 둘 다 `nil` 이면 기존 URL 경로로 읽는다(회귀) — 기존 `Fixtures/MiniRepo` 케이스가 수정 없이 통과 |
| `Tests/apicovTests/OwnersCommandTests.swift` | `pull` 산출물이 `OwnersLoader.parse` · `AuthorMapper.parse` 로 왕복되는지 / `identifiers` 2개 사용자가 `authors.yml` 에 2줄로 나오는지 |

`URLSession` 주입 지점은 구현 재량이다. `ScanCommandTests` 처럼 `parse([...]).run()` 으로 돌린다.

## 15. 미정 · 결정 이력

### 15-1. 남은 미정 (1건)

| # | 항목 | 왜 미정인가 |
|---|------|------------|
| 1 | Firebase `projectId` · `apiKey` 실제 값 | Firebase 프로젝트 생성 전. 이슈 G 에서 채운다 |

### 15-2. 결정된 항목 (2026-09-08)

| # | 항목 | 결정 | 반영 절 |
|---|------|------|--------|
| 2 | Firestore 리전 | `asia-northeast3` (서울) | §12-1 · 이슈 G |
| 3 | docId 선행 `\|\|` | 선행 구분자 제거 → `GET\|api\|v1\|…`. 역변환 규칙 명시 | §4-2 · §14 · 이슈 B |
| 4 | 페이지네이션 | `nextPageToken` 루프 구현. 서버가 페이지 크기를 정하므로 안 따라가면 목록이 조용히 잘린다 — 정확성 문제 | §10-1 · §14 · 이슈 A |
| 5 | `dodo` · `One` 이관 | 제옹 · 원만 이관. 소피 · 도도는 만들지 않는다 | §12 · 이슈 G |
| 6 | CI 계정 identifier | `identifiers: ["stella-ci"]` | §12-2 · 이슈 G |
| 7 | AdminView 수정 UI | 포함. 인라인 수정(표시명 · github · `identifiers` 추가/삭제 · `active` 토글). 없으면 기능 퇴행 | §8 · §10-3 · 이슈 D |
| 8 | GUI 내보내기 | `authors.yml` 도 함께. 메뉴 문구 "담당자 YAML 내보내기…" | §10-3 · 이슈 E |
| 9 | 세션 상태 enum | `.signedOut` · `.authenticating` · `.authenticated` · `.offline` · `.failed(String)` | §9 · §10-3 · 이슈 C |
| 10 | 소비자 레포 CI | 범위 밖 → §18 후속 과제 | §2 · §18 |

## 16. 보충 결정

AC 를 검증 가능하게 만들려고 이 문서에서 보탠 세부. 1~8 모두 승인됨. 7 · 8 은 이번 개정에서 보탰다.

1. 미지의 `ownerUid`. `users` 에 없는 uid 를 가리키는 assignment 는 `Owners` 에서 제외하고 `StellaDirectory` 가 제외 목록을 따로 돌려준다. 조용히 버리지 않는다. (§6-7)
2. `AuthorMapper` 변환은 `active` 와 무관하게 전원을 포함한다. 회수된 사람의 blame 표시명이 계속 해석돼야 한다. 배정 Picker(`ownerPicker`)는 `active` 만 보여준다. (§6-8)
3. 로그인 직후 `users/{uid}` 조회가 403 이면 `.forbidden` → `.failed("등록되지 않았거나 비활성화된 계정")`. Keychain 에 refresh token 을 남기지 않는다.
4. `EditableOwner.email` · `ownerAssignments` 의 키 구조는 유지한다. 이름만 legacy 다. `manualOwners` 는 사용자당 하나이며 `email` 에 primary identifier 를 넣는다(원소마다 펼치면 Picker 에 같은 이름이 여러 번 뜬다). 바꾸면 `Pipeline` 은 안 건드려도 뷰 여러 곳이 흔들린다.
5. `ScanFormView` 담당자 편집 섹션 제거. `users` 쓰기라 member 에겐 항상 403 이다. `AdminView` 로 옮긴다.
6. CLI `pull` 은 캐시 미사용. 오프라인 정책은 GUI 전용이다.
7. GUI 내보내기는 폴더를 고른다 (신규). 파일 두 개를 쓰므로 `NSSavePanel` 대신 폴더 선택 후 `owners.yml` · `authors.yml` 을 그 안에 쓴다.
8. GUI 스캔이 `Pipeline` 에 넘기는 담당자 값 (신규). `ScanModel.buildConfig()` 는 지금 `authorsPath` · `ownersPath` 를 `ScanConfig` 에 넘기고 `Pipeline` 이 파일에서 읽는다. 두 경로를 없애면 GUI 스캔의 blame 표시명 · owner 가 비게 되는데 원안이 이를 놓쳤다. 결정: `ScanConfig` 에 `owners: Owners?` · `authors: AuthorMapper?` 를 추가하고 GUI 는 원격/캐시에서 얻은 값을 직접 넣는다. `Pipeline.generate()` 는 값이 있으면 쓰고 없으면 URL 에서 읽는다(§10-5). 폐기한 대안은 디렉터리를 `~/Library/Application Support/Stella/owners.yml` · `authors.yml` 로 직렬화해 그 URL 을 넘기는 방식이었다. 같은 폴더의 `directory-cache.json` 과 데이터가 두 표현으로 중복된다. 직렬화 실패 처리와 두 표현 사이 동기화 부담도 붙어서 버렸다.

---

## 17. 작업 분할 — GitHub 이슈 7개 초안

- 이슈 제목 형식: `{이모지} {Type}: {내용}` (콜론). PR 은 `{이모지} [Type] {내용} (#이슈번호)`.
- 브랜치: `feat/{이슈번호}` (A~F) · `chore/{이슈번호}` (G). base `main`.
- 라벨: 이 레포에는 `:page_facing_up: Docs` 만 있다. A~G 생성 전에 아래를 먼저 만든다 (`docs/claude/git-workflow.md` 에 있는 명령 그대로).
  ```bash
  gh label create ":sparkles: Feature" --color a2eeef --description "새 기능을 추가합니다."
  gh label create ":wrench: chore"     --color ededed --description "기타 작업입니다."
  ```
- 이슈 Type: A~F `Feature`, G `Task`. 우선순위는 아래 제안값, Effort 는 생성 시 스킬로 지정(옵션명 미확인).
- 선행 관계: A → B → {C, F} · {B, C} → D · {B, C} → E · {D, E, F} → G. 연속 브랜치 파생이 허용되므로 선행 PR 머지 전이라도 다음 브랜치를 딸 수 있다.
- 이 초안은 실제 생성하지 않았다. 생성 시 `--body-file` 로 넘긴다.

```
A  ✨ Feature: StellaCore/Remote 전송 계층 — FirebaseAuthClient · FirestoreREST · Session
B  ✨ Feature: StellaDirectory · DirectoryCache — Firestore 문서를 Owners · AuthorMapper 로 변환하고 캐시
C  ✨ Feature: GUI 로그인 게이트 — SessionModel · LoginView · 세션 상태로 진입 분기
D  ✨ Feature: GUI 관리자 화면 — 계정 발급 · 사용자 목록 · 인라인 수정 · 회수 (AdminView)
E  ✨ Feature: GUI 담당자 소스를 Firestore 로 전환 — ScanConfig 에 Owners · AuthorMapper 직접 주입, 담당자 YAML 은 내보내기 전용
F  ✨ Feature: apicov owners pull — Firestore 담당자를 owners.yml · authors.yml 로 내려받기
G  🍀 ETC: Firestore 규칙 배포 · 담당자 수동 이관 · 매핑 YAML 문서 갱신
```

### A — `✨ Feature: StellaCore/Remote 전송 계층 — FirebaseAuthClient · FirestoreREST · Session`

라벨 `:sparkles: Feature` · Type `Feature` · 우선순위 높음 · 브랜치 `feat/{번호}` · 선행 없음

````markdown
## 📌 작업 목적
Stella 에 Firebase 기반 접근 통제를 넣기 위한 첫 단계. Firebase SDK 없이 Auth REST 와 Firestore REST 를 `URLSession` + `Codable` 로 감싸는 순수 전송 계층을 `StellaCore` 에 만든다. UI 를 모르고, CLI 와 GUI 가 같이 쓴다.

설계 문서: `docs/superpowers/specs/2026-09-08-firebase-directory-design.md` §3 · §10-1 · §10-2

## 🛠️ 작업 내용
- `Sources/StellaCore/Remote/FirebaseConfig.swift` — `projectId` · `apiKey` 상수 (`Sendable`). 실제 값은 미정이므로 플레이스홀더로 두고 이슈 G 에서 채운다
- `Sources/StellaCore/Remote/Session.swift` — `idToken` · `refreshToken` · `uid` · `role` · `expiresAt` (`Sendable`)
- `Sources/StellaCore/Remote/FirebaseAuthClient.swift` — `signIn(loginId:)` · `refresh(_:)` · `signUp(loginId:) -> uid`. `loginId` 하나로 이메일 `{loginId}@stella.umc.local` · 비밀번호 `{loginId}` 를 조립한다. `signUp` 은 응답에서 uid 만 돌려주고 토큰은 버린다
- `Sources/StellaCore/Remote/FirestoreREST.swift`
  - 문서 GET / PATCH(`updateMask` 지원) / DELETE
  - 컬렉션 목록 조회 — **`nextPageToken` 이 빌 때까지 순회해 전체를 모은다.** 서버가 페이지 크기를 정하므로 한 번만 부르면 조용히 잘린다
  - Firestore value(`stringValue` · `booleanValue` · `timestampValue` · `arrayValue` · `mapValue`) ↔ Swift 값 매퍼. `identifiers: [string]` 때문에 `arrayValue` 가 필요하다
- `StellaError` 에 `.authenticationFailed(String)` · `.remoteUnavailable(String)` · `.forbidden` 추가
- 두 클라이언트 모두 `init(session: URLSession = .shared)` 로 세션 주입 (`OpenAPILoader` 와 같은 방식)
- 테스트: `Tests/StellaCoreTests/Remote/FirebaseAuthClientTests.swift` · `FirestoreRESTTests.swift` (`StubURLProtocol` 사용)

## ✅ 완료 조건
- [ ] `import SwiftUI` · `import AppKit` 이 `Sources/StellaCore/Remote/` 어디에도 없다
- [ ] `FirebaseAuthClient.signIn(loginId:)` 가 `accounts:signInWithPassword` 에 `{loginId}@stella.umc.local` / `{loginId}` 를 보내고 `Session` 을 돌려준다 (스텁 테스트)
- [ ] Auth 400 응답(`INVALID_LOGIN_CREDENTIALS` 등)이 `StellaError.authenticationFailed` 로, `URLError` · 5xx 가 `.remoteUnavailable` 로, Firestore 403 이 `.forbidden` 으로 매핑된다 (각각 테스트)
- [ ] `refresh(_:)` 가 `securetoken.googleapis.com/v1/token` 에 `grant_type=refresh_token` 으로 호출하고 새 `Session` 을 돌려준다
- [ ] `Session` 에 "만료 5분 전" 판정이 있고 경계값 테스트가 있다
- [ ] `signUp(loginId:)` 가 uid 만 돌려주고 응답의 idToken · refreshToken 은 어디에도 저장하지 않는다
- [ ] 컬렉션 목록 조회가 `nextPageToken` 이 빌 때까지 순회한다 — 첫 응답에 `nextPageToken` 이 있고 둘째 응답에 없는 2페이지 스텁에서 두 페이지의 문서가 모두 모이고, 둘째 요청에 `pageToken` 이 실려 간다 (테스트)
- [ ] `FirestoreREST` value 인코딩 → 디코딩 왕복 테스트가 string · bool · timestamp · array · 중첩 map 을 덮는다
- [ ] `Package.swift` 에 새 의존성이 없다
- [ ] `swift build` · `swift test` 통과

## 📂 관련 파일 / 영역
- `Sources/StellaCore/Remote/` (신규)
- `Sources/StellaCore/Errors/StellaError.swift`
- `Sources/StellaCore/OpenAPI/OpenAPILoader.swift` — 세션 주입 패턴 참고
- `Sources/StellaTestSupport/StubURLProtocol.swift`

## ⚠️ 영향 범위 / 고려사항
- 기존 코드 호출 경로 없음 — 순수 추가. `Pipeline` · `ScanConfig` 무수정
- `apiKey` 는 바이너리에서 추출 가능한 공개값이다. 방어선은 Firestore 규칙(이슈 G)이지 키 은닉이 아니다
- 스냅샷 스키마 무관 — `CoverageSnapshot.currentSchemaVersion` 유지

## 🔗 관련 정보
- 설계 문서 §3 · §6-1 · §10-1 · §10-2 · §14 · §15-2(4)
- 후속: 이슈 B (StellaDirectory · DirectoryCache)
````

### B — `✨ Feature: StellaDirectory · DirectoryCache — Firestore 문서를 Owners · AuthorMapper 로 변환하고 캐시`

라벨 `:sparkles: Feature` · Type `Feature` · 우선순위 높음 · 브랜치 `feat/{번호}` · **blocked by A**

````markdown
## 📌 작업 목적
Firestore 의 `users` · `assignments` · `tagAssignments` 를 파이프라인이 이미 받는 `Owners` · `[AuthorMappingEntry]` 로 바꾸는 변환 계층과, 오프라인 진입용 캐시를 만든다. `Pipeline` 은 이 계층이 있는지도 모른다.

설계 문서: `docs/superpowers/specs/2026-09-08-firebase-directory-design.md` §4 · §9 · §10-1 · §16

## 🛠️ 작업 내용
- `Sources/StellaCore/Remote/StellaDirectory.swift`
  - 네트워크를 모른다. 이슈 A 의 클라이언트를 프로토콜로 받는다
  - `users` → `[AuthorMappingEntry]` — **사용자 한 명의 `identifiers` 원소마다 엔트리 하나**(`email: 원소`, `displayName` · `github` 공유). `active` 와 무관하게 전원 포함
  - `assignments` → `Owners.endpointOwners[OpenAPIKey(method, path)] = users[ownerUid].identifiers.first` (primary)
  - `tagAssignments` → `Owners.tagOwners[tag] = users[ownerUid].identifiers.first`
  - `users` 에 없는 `ownerUid` 는 `Owners` 에서 제외하고 제외 목록을 따로 돌려준다
  - uid ↔ primary identifier 조회 (GUI 배정 쓰기용)
  - docId encode / decode — §4-2 규칙. `encode`: `method + "|" + path.replacing("/", "|")` 에서 선행 `|` 하나 제거. `decode`: 첫 `|` 앞이 method, 나머지는 `|` → `/` 되돌리고 앞에 `/`
  - 쓰기: assignment upsert / delete (`updatedBy` = 호출자 uid, `updatedAt` = 현재 시각), user 문서 작성 · 필드 PATCH(`displayName` · `github` · `identifiers` · `active`)
- `Sources/StellaCore/Remote/DirectoryCache.swift`
  - 경로 `~/Library/Application Support/Stella/directory-cache.json`
  - 마지막 성공 조회(users · assignments · tagAssignments · 조회 시각) 저장 · 복구
  - 토큰 · loginId 는 담지 않는다. 손상 파일은 "캐시 없음" 으로 취급
- 테스트: `Tests/StellaCoreTests/Remote/StellaDirectoryTests.swift` · `DirectoryCacheTests.swift`

## ✅ 완료 조건
- [ ] `users` 3건(그중 1명은 `identifiers` 2개) + `assignments` 3건 + `tagAssignments` 1건 픽스처를 넣으면 기대한 `Owners` · `[AuthorMappingEntry]` 가 나온다 (테스트)
- [ ] `identifiers` 가 `["a@x", "b@y"]` 인 사용자는 `AuthorMappingEntry` 2개로 펼쳐지고 두 엔트리의 `displayName` · `github` 가 같다 (테스트)
- [ ] 그 사용자에게 걸린 assignment 의 `Owners` 값은 `"a@x"`(첫 원소)다 (테스트)
- [ ] `ownerUid` 가 `users` 에 없는 assignment 는 `Owners.endpointOwners` 에 들어가지 않고 제외 목록에 그 docId 가 들어 있다 (테스트)
- [ ] `active: false` 사용자도 `[AuthorMappingEntry]` 에 포함된다 (테스트)
- [ ] `encode(GET, /api/v1/attendances/available)` 가 `GET|api|v1|attendances|available` 이고, `{challengerId}` 가 든 경로를 포함해 `decode(encode(key)) == key` 다 (테스트)
- [ ] 변환 결과를 `OwnersLoader.serialize` → `OwnersLoader.parse`, `AuthorMapper.serialize` → `AuthorMapper.parse` 로 돌리면 같은 값이 나온다 (테스트)
- [ ] `DirectoryCache.save` 후 `load` 가 같은 값을 돌려준다 (테스트)
- [ ] 임의 바이트를 쓴 캐시 파일을 `load` 하면 `nil` 을 돌려주고 throw · crash 하지 않는다 (테스트)
- [ ] 캐시 파일 안에 `idToken` · `refreshToken` · `loginId` 문자열이 없다 (테스트에서 파일 내용 검사)
- [ ] `Sources/StellaCore/Remote/` 에 `SwiftUI` · `AppKit` import 없음
- [ ] `swift test` 통과

## 📂 관련 파일 / 영역
- `Sources/StellaCore/Remote/StellaDirectory.swift` · `DirectoryCache.swift` (신규)
- `Sources/StellaCore/Owners/OwnersLoader.swift` · `Sources/StellaCore/Authors/AuthorMapper.swift` — 변환 대상 타입, 무수정
- `Sources/StellaCore/Pipeline/Pipeline.swift` — 이 이슈에서는 무수정 (`generate()` 두 줄 변경은 이슈 E)

## ⚠️ 영향 범위 / 고려사항
- 캐시 파일은 `coverage.json` 과 무관한 별도 파일 — 스냅샷 스키마 버전 규칙에 걸리지 않는다
- `AuthorMappingEntry.email` 에 identifier 원소를 넣는다. 필드명은 legacy 지만 `Pipeline` 이 이 이름으로 조회하므로 바꾸지 않는다
- 페이지 순회는 이슈 A 의 `FirestoreREST` 가 책임진다. 여기서는 전체 목록을 받는다고 가정

## 🔗 관련 정보
- 설계 문서 §4-1 · §4-2 · §9 · §10-1 · §16-1 · §16-2
- 선행: 이슈 A / 후속: 이슈 C · E · F
````

### C — `✨ Feature: GUI 로그인 게이트 — SessionModel · LoginView · 세션 상태로 진입 분기`

라벨 `:sparkles: Feature` · Type `Feature` · 우선순위 높음 · 브랜치 `feat/{번호}` · **blocked by B**

````markdown
## 📌 작업 목적
Stella 앱을 열면 "발급 아이디" 를 넣어야 들어올 수 있게 한다. 한 번 로그인하면 refresh token 을 Keychain 에 두고 다음 실행부터는 조용히 복구한다. 네트워크가 없으면 캐시로 읽기 전용 진입한다.

설계 문서: `docs/superpowers/specs/2026-09-08-firebase-directory-design.md` §3 · §9 · §10-3

## 🛠️ 작업 내용
- `Sources/StellaApp/SessionModel.swift` — `@MainActor @Observable`
  - 상태: `.signedOut` · `.authenticating` · `.authenticated` · `.offline` · `.failed(String)` (설계 문서 §10-3 표)
  - 로그인: `FirebaseAuthClient.signIn` → `users/{uid}` 조회로 `role` · `active` 확인 → 성공 시 refresh token 을 `KeychainStore.write(_, account: "firebaseRefreshToken")`
  - 실행 시 자동 복구: Keychain 의 refresh token 으로 `refresh` → 성공하면 `.authenticated`
  - 만료 5분 전 자동 갱신
  - 네트워크 실패 + 캐시 있음 → `.offline`. 갱신 거부(`authenticationFailed`) → Keychain 비우고 `.failed(사유)`. 네트워크 실패 + 캐시 없음 → `.failed(사유)`
  - `role` 노출 (이슈 D 가 사용)
- `Sources/StellaApp/LoginView.swift` — 단일 필드 "발급 아이디" + 로그인 버튼 + `.failed` 사유 표시. `.authenticating` 이면 진행 표시. 403 이면 "등록되지 않았거나 비활성화된 계정"
- `Sources/StellaApp/StellaApp.swift` — `body` 에서 세션 상태로 `LoginView` / `ContentView` 분기
- `Sources/StellaApp/ContentView.swift` — `.offline` 이면 "오프라인 · 캐시 기준" 배지 표시

## ✅ 완료 조건
- [ ] 앱을 처음 실행하면 `.signedOut` 이고 `LoginView` 만 보이며 `ContentView` 는 마운트되지 않는다
- [ ] 유효한 발급 아이디로 로그인하면 `.authenticating` 을 거쳐 `.authenticated` 가 되고 `ContentView` 로 전환되며 Keychain `firebaseRefreshToken` 에 값이 들어간다
- [ ] 잘못된 아이디는 `.failed` 에 `authenticationFailed` 사유가, Auth 계정은 있으나 `users/{uid}` 가 없거나 `active: false` 인 경우는 "등록되지 않았거나 비활성화된 계정" 이 담기고 `LoginView` 에 보인다
- [ ] 로그인된 상태로 앱을 재실행하면 로그인 화면 없이 `ContentView` 가 뜬다
- [ ] 네트워크를 끊고 재실행하면 (캐시 있음) `.offline` 로 진입하고 "오프라인 · 캐시 기준" 배지가 보인다
- [ ] 네트워크를 끊고 캐시를 지운 뒤 재실행하면 `.failed` 로 로그인 화면이 뜬다
- [ ] 콘솔에서 Auth 계정을 비활성화한 뒤 재실행하면 refresh 가 거부되고 `.failed` 로 로그인 화면에 떨어지며 Keychain 값이 비워진다
- [ ] `SessionModel` 은 `@MainActor`, 세션 값 타입은 `Sendable`
- [ ] `swift build` 통과

## 📂 관련 파일 / 영역
- `Sources/StellaApp/SessionModel.swift` · `LoginView.swift` (신규)
- `Sources/StellaApp/StellaApp.swift` · `ContentView.swift` · `KeychainStore.swift`

## ⚠️ 영향 범위 / 고려사항
- `ScanModel` 은 이 이슈에서 건드리지 않는다 — 담당자 소스 전환은 이슈 E
- 오프라인 캐시 진입은 의도된 트레이드오프(설계 문서 §6-4). `active: false` 회수가 오프라인 기기에 즉시 반영되지 않는다
- 수동 검증 항목이 많다. 부트스트랩 admin 계정(이슈 G §12-1)이 먼저 있어야 끝까지 확인 가능 — 콘솔 작업이라 G 의 코드 변경과 무관하게 먼저 해도 된다

## 🔗 관련 정보
- 설계 문서 §3 · §6-4 · §9 · §10-3 · §15-2(9) · §16-3
- 선행: 이슈 B / 후속: 이슈 D · E
````

### D — `✨ Feature: GUI 관리자 화면 — 계정 발급 · 사용자 목록 · 인라인 수정 · 회수 (AdminView)`

라벨 `:sparkles: Feature` · Type `Feature` · 우선순위 보통 · 브랜치 `feat/{번호}` · **blocked by B, C**

````markdown
## 📌 작업 목적
관리자(제옹)가 팀원 계정을 발급하고, 표시명 · GitHub · `identifiers` 를 고치고, 회수하는 화면. `role == admin` 일 때만 사이드바에 나타난다. 발급된 아이디는 한 번만 보여주고 어디에도 저장하지 않는다. `ScanFormView` 의 담당자 편집 섹션이 이슈 E 에서 사라지므로 여기가 유일한 수정 경로다.

설계 문서: `docs/superpowers/specs/2026-09-08-firebase-directory-design.md` §4-1 · §6 · §7 · §8 · §10-3

## 🛠️ 작업 내용
- `Sources/StellaApp/AdminView.swift`
  - 발급 폼: 표시명 · `identifiers`(1개 이상, 첫 원소가 primary) · GitHub username
  - 발급 흐름: `users` 조회로 중복 검사(배열 원소 중 하나라도 다른 사용자와 겹치면 경고) → 32자 랜덤 loginId 생성 → `signUp` (uid 만 취함) → 관리자 토큰으로 `users/{uid}` 작성(`role: member`, `active: true`) → loginId 한 번 표시 + 클립보드 복사
  - 사용자 목록: 표시명 · `identifiers`(primary 표시) · GitHub · role · active
  - **인라인 수정**: 표시명 · GitHub · `identifiers` 원소 추가/삭제 · `active` 토글. 각각 `users/{uid}` 필드 PATCH. `identifiers` 는 비울 수 없다. 원소 추가 시에도 중복 검사
  - 회수 = `active` 토글 off. 문서는 지우지 않는다
- `Sources/StellaApp/ContentView.swift` — `SidebarItem` 에 admin 항목 추가, `role == admin` 일 때만 노출

## ✅ 완료 조건
- [ ] member 로 로그인하면 사이드바에 관리자 항목이 없다. admin 이면 있다
- [ ] 발급 완료 화면에 32자 loginId 가 표시되고 클립보드에 같은 값이 들어 있다
- [ ] 발급 후 Firestore `users/{uid}` 에 입력한 `identifiers` 배열 · displayName · github 와 `role: member` · `active: true` 가 들어 있고 `loginId` 필드는 없다
- [ ] 발급 도중 관리자 자신의 세션이 유지된다 (발급 직후 관리자 토큰으로 다른 요청이 성공)
- [ ] 다른 사용자가 이미 가진 값을 `identifiers` 에 넣어 발급을 시도하면 signUp 전에 경고가 뜨고 계정이 만들어지지 않는다
- [ ] `signUp` 성공 후 `users/{uid}` 작성이 실패하면 "문서 작성 실패 — 다시 시도" 가 뜨고 화면에 loginId 가 노출되지 않는다
- [ ] 목록에서 표시명을 고치면 `users/{uid}.displayName` 만 바뀌고 다른 필드는 그대로다 (`updateMask`)
- [ ] 목록에서 `identifiers` 에 원소를 추가하면 문서 배열이 늘어나고, `apicov owners pull` 산출 `authors.yml` 에 그 사람 엔트리가 하나 더 생긴다
- [ ] 첫 원소를 삭제하면 둘째 원소가 primary 가 되어 목록의 primary 표시가 바뀐다. 마지막 남은 원소는 삭제 버튼이 비활성화된다
- [ ] 이미 다른 사용자가 가진 값을 원소로 추가하려 하면 경고가 뜨고 PATCH 가 나가지 않는다
- [ ] 회수(`active` off)하면 문서의 `active` 가 `false` 가 되고 그 계정으로 새로 로그인하면 거부된다
- [ ] 회수된 사용자가 목록에 비활성 표시로 남아 있다 (삭제되지 않음)
- [ ] `swift build` 통과

## 📂 관련 파일 / 영역
- `Sources/StellaApp/AdminView.swift` (신규)
- `Sources/StellaApp/ContentView.swift` — `SidebarItem`
- `Sources/StellaApp/SessionModel.swift` — `role`
- `Sources/StellaCore/Remote/StellaDirectory.swift` — user 문서 작성 · 필드 PATCH

## ⚠️ 영향 범위 / 고려사항
- 사이드바의 admin 항목 숨김은 UX 다. 실제 방어는 Firestore 규칙(`users` write: admin only)
- `signUp` 을 앱에서 쓰는 이상 Auth 가입을 끌 수 없다 — 의도된 트레이드오프(설계 문서 §6-1)
- `identifiers` 중복 검사는 앱 단이다(§6-6). 규칙으로 강제되지 않으니 이 화면 밖(콘솔 직접 편집)에서 중복이 생길 수 있다
- 분실 시 복구 불가. 재발급 = 옛 계정 회수 → 새 계정 발급 → 배정 이전(§6-3). 배정 이전 UI 는 이슈 E 의 `ownerPicker` 로 한다

## 🔗 관련 정보
- 설계 문서 §4-1 · §6-1 · §6-2 · §6-3 · §6-6 · §7 · §8 · §15-2(7)
- 선행: 이슈 B · C / 후속: 이슈 G (마이그레이션이 이 화면을 쓴다)
````

### E — `✨ Feature: GUI 담당자 소스를 Firestore 로 전환 — ScanConfig 에 Owners · AuthorMapper 직접 주입, 담당자 YAML 은 내보내기 전용`

라벨 `:sparkles: Feature` · Type `Feature` · 우선순위 높음 · 브랜치 `feat/{번호}` · **blocked by B, C**

````markdown
## 📌 작업 목적
GUI 의 담당자 데이터가 `authors.yml` · `owners.yml` 파일 대신 Firestore(실패 시 캐시)에서 오게 한다. 배정 변경은 원격에 쓴다. 담당자 YAML 저장은 두 파일 내보내기 전용으로 남긴다. GUI 가 더 이상 파일 경로를 넘길 수 없으므로 `ScanConfig` 가 로드된 값을 직접 받도록 코어를 최소 수정한다.

설계 문서: `docs/superpowers/specs/2026-09-08-firebase-directory-design.md` §9 · §10-3 · §10-5 · §16

## 🛠️ 작업 내용
- `Sources/StellaCore/Pipeline/ScanConfig.swift` — `owners: Owners?` · `authors: AuthorMapper?` 필드 추가, 기본값 `nil`. 기존 `ownersURL: URL? = nil` 과 같은 방식으로 init 끝에 붙인다. 기존 호출부 `ScanCommand` · `ScanModel` · `PipelineTests` 는 무수정
- `Sources/StellaCore/Pipeline/Pipeline.swift` — `generate()` 의 `authors` · `owners` 로딩 두 줄을 `config.authors ?? URL 로드 ?? .empty` / `config.owners ?? URL 로드 ?? .empty` 로 교체 (설계 문서 §10-5)
- `Tests/StellaCoreTests/Pipeline/PipelineTests.swift` — 값 주입 시 URL 무시 / 둘 다 `nil` 이면 URL 경로 회귀, 케이스 2개 추가
- `Sources/StellaApp/ScanModel.swift`
  - `loadOwnerYAMLFiles()` → 원격 조회(`StellaDirectory`) → 실패 시 `DirectoryCache` → `manualOwners` · `ownerAssignments` 를 채운다. `manualOwners` 는 **사용자당 하나**, `email` = primary identifier. 키 `"METHOD path"` 구조 유지
  - 같은 자리에서 스캔에 넘길 `Owners` · `AuthorMapper` 값을 보관한다
  - `buildConfig()` — 그 값을 `ScanConfig(owners:authors:)` 에 직접 주입. `authorsURL` · `ownersURL` 은 `nil` (설계 문서 §16-8)
  - `saveOwnerYAMLFiles(authors:owners:)` → `assignOwner` 는 assignment PATCH / DELETE (`updatedBy` = 내 uid, `ownerUid` 는 primary identifier → uid 조회). `upsertOwner` · `deleteOwner` 는 제거 (이슈 D 의 AdminView 로)
  - `authorsPath` · `ownersPath` 프로퍼티와 UserDefaults 키 제거
  - `saveOwnersYAML()` → 폴더 선택 후 `owners.yml` · `authors.yml` 두 파일을 쓴다(§16-7). `tags` 는 `tagAssignments` 에서 오므로 대상 파일의 기존 `tags` 병합 단계 제거. `canExportOwnersYAML` 은 배정이나 사용자 중 하나라도 있으면 참
- `Sources/StellaApp/StellaApp.swift` — 메뉴 문구 "owners.yml 저장…" → "담당자 YAML 내보내기…"
- `Sources/StellaApp/ScanFormView.swift`
  - `metadataSection` 에서 `authors.yml` · `owners.yml` 행 제거 (`overrides.yml` 행 유지)
  - 담당자 편집 섹션(`새 담당자` · `저장` · `삭제`) 제거
- `Sources/StellaApp/EndpointsView.swift` — `ownerPicker` 는 `active` 사용자만 나열. `.offline` 이면 비활성화

## ✅ 완료 조건
- [ ] 로그인 후 설정 화면에 `authors.yml` · `owners.yml` 경로 입력이 없고 `overrides.yml` 행은 남아 있다
- [ ] 설정 화면에 담당자 추가 · 수정 · 삭제 UI 가 없다
- [ ] 스캔을 돌리면 디스크에 yml 을 쓰지 않고도 스냅샷의 `owner` 와 connection `author.displayName` 이 디렉터리 값으로 채워진다 (`~/Library/Application Support/Stella/` 에 `directory-cache.json` 외 파일이 생기지 않는다)
- [ ] `identifiers` 가 2개인 사용자는 `ownerPicker` 에 한 번만 나오고, 두 이메일 중 어느 쪽으로 커밋한 라인이든 blame 표시명이 그 사람으로 해석된다
- [ ] 온라인에서 엔드포인트 담당자를 바꾸면 Firestore `assignments/{docId}` 에 `ownerUid` · `updatedBy`(내 uid) · `updatedAt` 이 기록되고 docId 가 `GET|api|…` 형태다. "미지정" 으로 바꾸면 문서가 삭제된다
- [ ] 다른 기기(또는 재로그인)에서 같은 배정이 보인다
- [ ] `.offline` 진입 시 `ownerPicker` 가 비활성화되고 캐시의 배정이 표시된다
- [ ] `ownerPicker` 목록에 `active: false` 사용자가 없다. 단 그 사용자에게 이미 걸린 배정은 표시명으로 계속 해석된다
- [ ] "담당자 YAML 내보내기…" 로 폴더를 고르면 `owners.yml` · `authors.yml` 두 파일이 생기고, 각각 `OwnersLoader.parse` · `AuthorMapper.parse` 결과가 현재 디렉터리와 일치한다 (`tags` 는 `tagAssignments` 값, `authors.yml` 은 identifier 원소마다 한 엔트리)
- [ ] `ScanConfig` 에 `owners: Owners?` · `authors: AuthorMapper?` 가 기본값 `nil` 로 추가되고, `ScanCommand.swift` · `PipelineTests.swift` 의 기존 `ScanConfig(...)` 호출은 diff 가 0 줄이다
- [ ] `ScanConfig` 가 여전히 `Sendable` 이다 (Swift 6 strict concurrency 빌드 경고 없음)
- [ ] `PipelineTests`: `owners` · `authors` 값을 주고 `authorsURL` · `ownersURL` 에 존재하지 않는 경로를 넣어도 throw 없이 값 쪽이 결과에 반영된다
- [ ] `PipelineTests`: 둘 다 `nil` 이면 기존처럼 URL 에서 읽는다 — 기존 `Fixtures/MiniRepo` 회귀 케이스가 수정 없이 통과한다
- [ ] `Pipeline.swift` 변경이 `generate()` 안 두 줄에 그친다. `CoverageSnapshot` · `SnapshotEncoder` diff 는 0 줄
- [ ] `Tests/apicovTests/ScanCommandTests.swift` 가 수정 없이 통과한다 (CLI 회귀)
- [ ] `Tests/StellaAppTests/ScanModelTemplateTests.swift` 통과, `swift test` 통과

## 📂 관련 파일 / 영역
- `Sources/StellaCore/Pipeline/ScanConfig.swift` — 필드 2개 추가
- `Sources/StellaCore/Pipeline/Pipeline.swift` — `generate()` 두 줄
- `Tests/StellaCoreTests/Pipeline/PipelineTests.swift` — 케이스 2개 추가
- `Sources/StellaApp/ScanModel.swift` — `loadOwnerYAMLFiles` · `saveOwnerYAMLFiles` · `buildConfig` · `saveOwnersYAML` · `exportOwnersYAML` · `exportAuthorsYAML`
- `Sources/StellaApp/ScanFormView.swift` — `metadataSection` · 담당자 편집 섹션
- `Sources/StellaApp/EndpointsView.swift` — `ownerPicker`
- `Sources/StellaApp/StellaApp.swift` — 커맨드 메뉴

## ⚠️ 영향 범위 / 고려사항
- `applyingManualOwners(to:)` 와 `ownerPicker` 의 데이터 모양을 유지하므로 렌더링 경로는 안 바뀐다
- `buildConfig()` 가 넘길 파일 경로가 없어지면 GUI 스캔의 blame 표시명 · owner 가 빈다 — 원안이 놓쳤던 부분. `ScanConfig` 값 주입으로 메운다(§16-8)
- **코어 공용 타입(`ScanConfig`) · 공용 진입점(`Pipeline.generate()`) 변경이라 CLI 회귀 테스트가 통과해야 한다** — `ScanCommandTests` · `PipelineTests` 기존 케이스 무수정 통과가 머지 조건
- 옵셔널 필드 + 기본값 `nil` 이라 소스 호환. `ScanConfig` 는 `Sendable` 유지(`Owners` · `AuthorMapper` 모두 `Sendable`)
- GUI 의 `bearerToken` · `basicUser` 등 API 테스트용 설정은 이 이슈와 무관 — 건드리지 않는다
- 이 이슈가 머지되면 GUI 는 더 이상 레포의 yml 을 읽지 않는다. 이슈 G 의 마이그레이션이 끝나기 전에 배포하면 GUI 담당자가 비어 보인다 — 머지 순서 주의

## 🔗 관련 정보
- 설계 문서 §9 · §10-3 · §10-5 · §11 · §14 · §15-2(8) · §16-2 · §16-4 · §16-5 · §16-7 · §16-8
- 선행: 이슈 B · C / 후속: 이슈 G
````

### F — `✨ Feature: apicov owners pull — Firestore 담당자를 owners.yml · authors.yml 로 내려받기`

라벨 `:sparkles: Feature` · Type `Feature` · 우선순위 보통 · 브랜치 `feat/{번호}` · **blocked by B**

````markdown
## 📌 작업 목적
CI 가 `owners.yml` · `authors.yml` 을 레포에서 읽는 대신 Firestore 에서 내려받게 한다. `scan` 은 건드리지 않고, CI 는 `pull` → `scan` 두 줄이 된다.

설계 문서: `docs/superpowers/specs/2026-09-08-firebase-directory-design.md` §10-4

## 🛠️ 작업 내용
- `Sources/apicov/Commands/OwnersCommand.swift` — `owners` 상위 커맨드 + `pull` 서브커맨드
  ```
  apicov owners pull --login-id-env STELLA_LOGIN_ID --out-owners owners.yml --out-authors authors.yml
  ```
  - `--login-id-env` 는 자격증명 값이 아니라 **환경변수 이름** (`--auth-env` 관례)
  - 흐름: env 읽기 → `FirebaseAuthClient.signIn` → 세 컬렉션 조회(페이지 순회는 `FirestoreREST` 가 함) → `StellaDirectory` 변환 → `OwnersLoader.serialize` · `AuthorMapper.serialize` → 파일 쓰기
  - 산출물: `owners.yml` owner 값 = primary identifier, `authors.yml` = identifier 원소마다 엔트리 하나
  - 실패 시 `StellaError` 그대로 throw, 0 이 아닌 종료 코드. Keychain · 캐시 미사용
- `Sources/apicov/Apicov.swift` — `subcommands` 에 `OwnersCommand.self` 추가
- 테스트: `Tests/apicovTests/OwnersCommandTests.swift` — `StubURLProtocol` 로 Auth · Firestore 응답 스텁, 산출물 왕복 검증. `URLSession` 주입 지점은 구현 재량

## ✅ 완료 조건
- [ ] `swift run apicov owners pull --help` 가 세 옵션을 보여준다
- [ ] 환경변수가 없으면 `ValidationError` 로 종료하고 네트워크 요청을 보내지 않는다 (테스트)
- [ ] 스텁 응답으로 `pull` 을 돌리면 `--out-owners` 파일을 `OwnersLoader.parse` 한 결과와 `--out-authors` 파일을 `AuthorMapper.parse` 한 결과가 스텁 데이터와 일치한다 (테스트)
- [ ] 스텁에 `identifiers` 2개인 사용자를 넣으면 `authors.yml` 에 그 사람 엔트리가 2개 나오고 `owners.yml` 의 owner 값은 첫 원소다 (테스트)
- [ ] 로그인 실패(400) 시 종료 코드가 0 이 아니고 출력 파일이 만들어지지 않는다 (테스트)
- [ ] 생성된 `owners.yml` · `authors.yml` 을 `apicov scan --owners --authors` 에 그대로 넘겨 `ScanCommandTests` 와 같은 방식으로 스냅샷의 `owner` 가 채워진다 (테스트)
- [ ] `ScanCommand.swift` diff 가 0 줄이다
- [ ] `swift test` 통과

## 📂 관련 파일 / 영역
- `Sources/apicov/Commands/OwnersCommand.swift` (신규)
- `Sources/apicov/Apicov.swift`
- `Tests/apicovTests/OwnersCommandTests.swift` (신규)
- `Tests/apicovTests/ScanCommandTests.swift` — 패턴 참고

## ⚠️ 영향 범위 / 고려사항
- CLI 는 오프라인 캐시를 쓰지 않는다 — CI 는 무상태
- 소비자 레포(`umc-product-iOS`) 워크플로에 `pull` 단계와 `STELLA_LOGIN_ID` secret 추가가 필요하다. 이 레포 범위 밖(설계 문서 §18)
- CI 용 member 계정(`identifiers: ["stella-ci"]`) 발급은 이슈 G

## 🔗 관련 정보
- 설계 문서 §4-1 · §10-4 · §14 · §16-6
- 선행: 이슈 B / 후속: 이슈 G
````

### G — `🍀 ETC: Firestore 규칙 배포 · 담당자 수동 이관 · 매핑 YAML 문서 갱신`

라벨 `:wrench: chore` + `:page_facing_up: Docs` · Type `Task` · 우선순위 높음 · 브랜치 `chore/{번호}` · **blocked by D, E, F** (이관 · 문서 부분). 부트스트랩 · 규칙 배포는 콘솔 작업이라 C 착수 전에 먼저 해도 된다

````markdown
## 📌 작업 목적
Firebase 콘솔 설정 · 보안 규칙 배포 · 첫 admin 부트스트랩 · 담당자 2명 / 배정 3건 수동 이관, 그리고 `owners.yml` · `authors.yml` 을 생성물로 강등하는 레포 · 문서 정리.

설계 문서: `docs/superpowers/specs/2026-09-08-firebase-directory-design.md` §5 · §12 · §13

## 🤔 배경
코드(이슈 A~F)만으로는 아무도 못 들어온다. 규칙이 없으면 방어선이 없고, admin 이 없으면 발급을 못 하고, 이관을 안 하면 담당자가 비어 보인다. 이 이슈가 전체를 "켜는" 단계다.

## 🛠️ 작업 내용
### 콘솔 (코드 무관, 먼저 해도 됨)
- Firebase 프로젝트 · Firestore(Native mode, 리전 `asia-northeast3`) 준비 — projectId 는 이때 확정
- Authentication → 이메일/비밀번호 활성화
- 제옹 admin 계정 수동 생성: `{loginId}@stella.umc.local` / `{loginId}` (32자 랜덤)
- `users/{uid}` 수동 작성: `identifiers: ["euijjang97@gmail.com"]` · `displayName: 제옹` · `github: JEONG-J` · `role: admin` · `active: true`
- 설계 문서 §5 의 규칙을 그대로 배포
- `FirebaseConfig.swift` 의 projectId · apiKey 플레이스홀더를 실제 값으로 교체

### 이관 (AdminView 사용, 이슈 D · E 머지 후)
- 원 발급: `identifiers: ["One@One"]` · 표시명 원 · github `One` · member. `One@One` 은 blame 이메일이 아니라 매칭이 안 되는데, 실제 커밋 이메일을 알게 되면 원소로 추가한다
- 제옹 `identifiers` 보강: 소비자 레포에서 `git log --format='%ae' | sort -u` 로 실제 blame 이메일을 확인해 필요하면 추가 (`ikejhc159@gmail.com` · noreply 주소 후보)
- CI 계정 발급: `identifiers: ["stella-ci"]` · member → loginId 를 소비자 레포 secret `STELLA_LOGIN_ID` 로 전달
- 배정 3건을 제옹으로: `GET /api/v1/attendances/available` · `GET /api/v1/attendances/challenger/{challengerId}/history` · `GET /api/v1/attendances/pending`
- `POST /api/v1/attendances/check` 는 비워 두고 제옹이 GUI 에서 재배정
- 소피(`LeeYeJi546`) · 도도(`dodo@example.com`) 는 만들지 않는다
- `apicov owners pull` 결과를 §12-5 기준으로 검증

### 레포 · 문서
- `.gitignore` 에 `/owners.yml` · `/authors.yml` (루트 고정 — `Fixtures/` 는 유지)
- `git rm --cached owners.yml authors.yml`
- `CLAUDE.md` "매핑 YAML" 절 — 운영본은 `overrides.yml` 하나. 나머지 둘은 `apicov owners pull` 생성물
- `README.md` — 빠른 시작에 `owners pull` 추가, 매핑 YAML 표 · 레포 구조 수정
- `CONTRIBUTING.md` — "커밋하지 않는 것" 절 수정
- Wiki `Mapping-YAML` · `CLI-Reference` · `Stella-App` · `Consumer-Integration` (별도 레포)

## ✅ 완료 조건
- [ ] 규칙 배포 후 미등록 Auth 계정으로 `users` GET 이 403, member 로 `users` PATCH 가 403, member 가 `updatedBy` 를 남의 uid 로 넣은 assignment PATCH 가 403 이다 (콘솔 규칙 시뮬레이터 또는 실제 호출로 확인)
- [ ] Firestore 데이터베이스 리전이 `asia-northeast3` 다
- [ ] 제옹 계정으로 로그인하면 사이드바에 관리자 항목이 보인다
- [ ] `users` 에 제옹(admin) · 원(member) · `stella-ci`(member) 세 문서가 있고 소피 · 도도 문서는 없다
- [ ] `apicov owners pull` 산출 `owners.yml` 에 제옹 배정 3건이 있고, `authors.yml` 에 제옹 `identifiers` 원소 수만큼 + 원 1 + `stella-ci` 1 엔트리가 있다
- [ ] 이관 전 커밋본과의 diff 가 §12-5 에 적힌 차이뿐이다
- [ ] `git ls-files owners.yml authors.yml` 이 비어 있고, `git status` 에서 두 파일이 untracked 로도 보이지 않는다
- [ ] `git ls-files Fixtures/owners.yml Fixtures/authors.yml` 은 여전히 두 파일을 보여준다
- [ ] `swift test` 통과 (픽스처가 살아 있음을 겸해서 확인)
- [ ] `CLAUDE.md` · `README.md` · `CONTRIBUTING.md` 에 "세 파일을 커밋한다" 는 문장이 남아 있지 않다
- [ ] `FirebaseConfig.swift` 에 플레이스홀더가 남아 있지 않다

## 📂 관련 파일 / 영역
- Firebase 콘솔 (Authentication · Firestore · Rules)
- `Sources/StellaCore/Remote/FirebaseConfig.swift`
- `.gitignore` · `owners.yml` · `authors.yml`
- `CLAUDE.md` · `README.md` · `CONTRIBUTING.md`
- Wiki (별도 레포 `umc-product-stella.wiki.git`)

## ⚠️ 영향 범위 / 고려사항
- 이슈 E 가 먼저 배포되고 이관이 늦으면 GUI 담당자가 비어 보인다. 이관을 E 배포 직후에 붙여서 한다
- `overrides.yml` 은 그대로 커밋 — 이 이슈에서 건드리지 않는다
- 남은 미정은 projectId · apiKey 한 건 — 콘솔 단계에서 확정된다
- 소비자 레포 CI 워크플로 수정은 별도(§18) — 이 이슈에서는 secret 값 전달까지만

## 🔗 관련 정보
- 설계 문서 §5 · §6-5 · §12 · §13 · §15
- 선행: 이슈 D · E · F (이관 · 문서) / 없음 (콘솔 · 규칙 · 부트스트랩)
````

## 18. 후속 과제 (이 레포 범위 밖)

- **소비자 레포(`umc-product-iOS`) CI** — 워크플로에 `apicov owners pull` 단계 추가, `STELLA_LOGIN_ID` secret 등록. 값은 이슈 G 에서 발급한 `stella-ci` 계정의 loginId.
- **`stella-ci` 를 배정 Picker 에서 숨기기** — 지금은 `active` member 라 Picker 에 나온다. 배정하지 않으면 해가 없어 당장은 두고 거슬리면 그때 다룬다.
- **소피 · 도도 계정** — 실제 팀원으로 확인되면 `AdminView` 에서 발급. 이관 대상이 아니다.
