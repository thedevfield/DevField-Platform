# DevField — RepoQ 통합 API 명세서

> **Practice. Solve. Grow.**  
> 문서 ID: `DF-API-RQ-ALL-001` (제안) · 버전: **v0.3 Multi-Repository·Private P0 확장 초안** · 최초 작성일: 2026-10-21 · 개정일: 2026-11-05
> 상태: **제품 P0·BR-31~36 확정 반영 / 운영값 결정 전 / 로컬 미게시 계약 초안**
> 기준: [RepoQ PRD v1.2 개정 초안](../../products/repoq/PRD.md) · [서비스 경계 설계서](../../architecture/repoq-service-boundaries.md) · [ADR-001](../../adr/0001-repoq-service-decomposition.md) · [ADR-002](../../adr/0002-devfield-common-authentication.md) · [ADR-003](../../adr/0003-repoq-multi-repository-private-access.md)

## 1. 한눈에 보는 구조

> **Multi-Repository·Private P0 확장 계약:** 기존 P-01~06·X-01~04의 번호와 역사적 단일 공개 Repository 예시는 보존한다. 이 문서 마지막의 **「Multi-Repository·Private P0 확장 계약」**이 복수·비공개 요청과 Snapshot 형태에 우선 적용된다. 아래 21개 합계는 확장 전 기준이며 초기 확장안의 신규 P-07~13·X-05를 포함한 논리 API 합계는 **29개**다. 현재 구현의 연결 상태 GET·동의 GET 및 P-10의 BFF/Backend 경로 분리는 10.2절에 별도로 명시하며, 29개를 실제 구현 Endpoint 수로 사용하지 않는다. BR-31~36은 확정됐으며 운영값만 `[결정 필요]`다.

RepoQ 화면은 **프로젝트 선택·분석 → 자유 연습 또는 모의면접 → 질문·답변·평가·이력** 순서로 동작한다.

| 담당 | 책임 | 이번 문서에 정리한 API |
|---|---|---:|
| Project Service | GitHub 프로젝트 등록·분석·재분석·삭제 | 사용자용 6개 |
| Interview Service | 자유 연습·모의면접·질문·답변·평가·이력 | 사용자용 7개 |
| DevField Auth Service | 공통 User ID·계정 정보·로그인 수단·인증 상태·탈퇴 조정 | 계정 관리 4개 |
| Project ↔ Interview | 분석 자료 전달·프로젝트 삭제 시 기록 정리 | 내부용 2개 |
| Auth → RepoQ | 회원 탈퇴 시 Project·Interview의 사용자 데이터 삭제 | 내부용 2개(서비스별 1개) |
| **기존 합계** | **v0.2 서비스·주소·메서드 조합** | **21개** |

**범위 주의:** 이메일 회원가입·로그인, 소셜 로그인, 이메일 확인, 비밀번호 관리, 로그아웃은 *반드시 제공할 기능*이지만 이 RepoQ 통합 명세의 **21개에 포함하지 않았다.** Auth 상세 Endpoint·콜백은 별도 로컬 v0.1 초안에서 제안하며 기존 21개와 합산하지 않는다. 두 회원 탈퇴 내부 API는 경로 문자열이 같아도 호출받는 서비스가 달라 각각 센다. [ADR-002](../../adr/0002-devfield-common-authentication.md)는 독립 Auth Service, Spring Boot 4.x + Spring Security 7.x, 사용자 API용 RS256 JWT를 결정하며 범용 OAuth/OIDC Provider 구축을 승인하지 않는다.

공통 인증의 **별도 상세 초안**은 [DevField 공통 Auth API 계약서 v0.1](../auth/api-spec.md)과 [Auth 데이터 모델 및 ERD v0.1](../../architecture/auth/data-model.md)에 둔다. 이 문서의 기존 P/I/A/X 21개 번호·의미는 유지하며 신규 Auth 계약 수는 별도로 센다. 두 신규 문서는 로컬 작성 초안으로 GitHub Push 전이다.

### 질문 용어

- **큰 질문(Main Question):** 질문 묶음의 첫 번째 핵심 질문 1개.
- **꼬리질문(Follow-up Question):** 사용자 답변에 따라 나오는 추가 질문 **0~3개**.
- **질문 묶음(Question Bundle):** 큰 질문 **1개 + 꼬리질문 0~3개 + 답변 + 묶음 평가**.
- **큰 질문 묶음:** **모의면접 1회차를 이루는 질문 묶음 5개**. 별도 DB 엔터티가 아닌 사용자 설명용 용어.
- **면접 세션(Interview Session):** 자유 연습 1회 또는 모의면접 1회의 저장·진행 단위.
- **Snapshot:** 특정 GitHub Commit을 분석해 보관한 결과. 최신은 *마지막으로 정상 분석한 버전*이지 GitHub 실시간 최신 코드가 아니다.

## 2. 공통 약속

- 아래 URL은 **서비스별 논리 API 경로**다. 현재 RepoQ Web은 BFF를 통해 Project Service를 호출한다(10절). 다른 미구현 API의 브라우저 경유 방식은 후속 설계 대상이다.
- 모든 사용자용 API는 로그인 필요. Project/Interview는 Auth가 발급한 RS256 JWT를 공개키·JWKS로 검증한다. **서명·허용 알고리즘·`iss`·`aud`·유효기간**을 확인하고 검증된 `sub`를 DevField User ID로 사용한다. 각 서비스는 그 뒤 별도로 리소스 소유권을 확인하며 요청에 임의로 적은 User ID를 믿지 않는다. Auth 소유 계정 관리 API에도 해당 Auth의 인증·본인 확인 정책을 적용한다.
- 사용자 JWT 검증은 X-01~X-04 내부 서비스 호출의 권한 검증을 대체하지 않는다. 서비스 간 인증·사용자 위임, 실제 issuer·audience 값은 후속 계약이다. JWT 서명 검증만으로 로그아웃의 즉시 폐기나 `DELETING` 상태 전파가 해결되지 않는다.
- 목록/상세 조회에는 **허용된 정보만** 반환한다. 코드 전문·비밀 키·다른 사람의 데이터는 반환하지 않는다.
- 질문 생성처럼 시간이 걸리는 작업은 우선 `PREPARING` 등을 반환하고, **기존 상세 조회 API로 상태를 확인**한다. 별도 AI 작업 상태 API는 만들지 않는다.
- 예시 ID와 코드 내용은 설명용이다. 필드명·HTTP 응답 코드는 구현 전 마지막으로 확인한다.

## 3. Project Service — 기존 사용자 API 6개 (P-01~06 ID 유지)

| 번호 | API | 언제 사용하나? | 주로 보내는 값 → 받는 값 | 주요 실패 상황 |
|---|---|---|---|---|
| P-01 | `POST /api/projects` | GitHub 프로젝트 **처음 등록** | `repositories[]`, `name`(선택) → `id`, `analysisStatus`, Repository 상태; 단일 Public `repositoryUrl` 호환 | 잘못된 URL, 무권한 Private, 같은 저장소 중복 |
| P-02 | `GET /api/projects` | RepoQ **내 프로젝트 목록** 화면 | 별도 입력 없음 → 내 프로젝트 ID·이름·분석 상태 목록 | 로그인 만료 |
| P-03 | `GET /api/projects/{id}` | 프로젝트 상세·**분석 진행 상태** 확인 | `id` → 현재 작업 상태, 마지막 성공 Snapshot, 선택 가능 분야·근거 요약 | 프로젝트 없음, 다른 사용자 소유 |
| P-04 | `PATCH /api/projects/{id}` | 프로젝트 **표시 이름 변경** | `name` → 변경된 이름 | 빈 이름, 삭제 중인 프로젝트 |
| P-05 | `POST /api/projects/{id}/reanalyze` | GitHub 코드 수정 후 사용자가 **재분석** 선택 | `id` → 새 분석 접수 상태 | 중복 재분석, 삭제 중, GitHub 오류 |
| P-06 | `DELETE /api/projects/{id}` | **프로젝트 삭제** 선택 | `id` → 삭제 진행 상태 | 권한 없음, 삭제 중 장애 |

**등록 요청 예시**

```json
{ "repositoryUrl": "https://github.com/example/demo", "name": "내 프로젝트" }
```

**상세 조회 응답 예시 (재분석 중에도 이전 결과 사용 가능)**

```json
{
  "id": "prj-01",
  "name": "내 프로젝트",
  "projectStatus": "ACTIVE",
  "analysisStatus": "RUNNING",
  "canInterview": true,
  "latestReadySnapshot": {
    "snapshotId": "snap-01",
    "commitSha": "a1b2c3d4",
    "topics": ["DATABASE", "API_BACKEND"]
  },
  "analysisNotice": "이전 분석 결과로 면접을 시작할 수 있습니다."
}
```

**정책:** 최초 등록 시 분석 시작. GitHub에 새 코드가 올라와도 자동 재분석하지 않는다. 사용자가 재분석하여 성공하면 **새 면접부터** 새 Snapshot 사용, 과거 면접은 이전 Snapshot 유지. 재분석 실패 시 이전 정상 분석 결과를 보존한다. `GET /api/projects/{id}`가 분석·삭제 진행 상황을 함께 알려주므로 상태 조회 전용 API는 없다.

**삭제:** P-06은 삭제를 **시작**하고, Project가 내부 삭제 API로 Interview 기록까지 정리한 후 완료한다. 완료되기 전 사용자에게 삭제 완료라고 표시하지 않는다. 다른 사용자가 등록한 같은 공개 GitHub 저장소의 데이터는 지우지 않는다.

## 4. Interview Service — 사용자용 7개 (구성 확정)

| 번호 | API | 언제 사용하나? | 주로 보내는 값 → 받는 값 | 주요 실패 상황 |
|---|---|---|---|---|
| I-01 | `POST /api/interviews` | **자유 연습/모의면접 시작** | `projectId`, `mode`, FREE 첫 `topicKey` → `interviewId`, 준비 상태 | READY 분석 없음, MOCK 5개 질문 근거 부족 |
| I-02 | `GET /api/interviews` | 프로젝트별 **면접 이력 목록** | `projectId`(조회 조건) → 모드·상태·진행 수 목록 | 권한 없음 |
| I-03 | `GET /api/interviews/{id}` | 진행·재접속·복습·**AI 준비 상태 확인** | `id` → 현재 질문·저장된 답변·진행 상태·공개 가능한 평가 | 권한 없음, 삭제된 세션 |
| I-04 | `POST /api/interviews/{id}/answers` | 큰 질문·꼬리질문에 **답변 제출** | `questionId`, `answer`, `submissionId` → 저장/다음 질문 준비 상태 | 빈 답변, 중복/잘못된 진행 순서 |
| I-05 | `POST /api/interviews/{id}/bundles` | FREE에서 평가 확인 후 **다음 질문 묶음 요청** | 이번 묶음의 `topicKey` → 새 질문 준비 상태 | MOCK에서 요청, 진행 중 묶음 미완료 |
| I-06 | `PATCH /api/interviews/{id}` | **일시 중단·재개·자유 연습 종료** | `action`: `PAUSE`/`RESUME`/`END` → 새 상태 | MOCK 중도 `END`, 허용되지 않는 상태 전환 |
| I-07 | `DELETE /api/interviews/{id}` | **개별 이력 삭제** 또는 MOCK **명시적 포기** | `id` → 삭제 결과 | 권한 없음 |

### 4.1 자유 연습(FREE)

1. 시작할 때 사용자가 **첫 질문 묶음**의 분야를 선택한다. `RANDOM`(전체 랜덤)도 선택 가능하다.
2. 큰 질문 1개와 답변에 따른 꼬리질문 0~3개를 진행한다.
3. 묶음이 끝나면 **묶음 평가를 즉시 공개**한다.
4. 계속하고 싶다면 사용자가 **다음 질문 묶음마다** 분야를 다시 고르고 I-05를 호출한다. 이전과 같은 분야도, 다른 분야도 선택할 수 있다.
5. 같은 프로젝트에서 과거 **실제 출제된** 질문을 참고해 같은 의도의 반복을 줄인다. 질문 생성 후보나 미출제 질문은 반복 이력에 넣지 않는다.
6. 더 연습하지 않으려면 I-06의 `END`로 종료한다. 기록은 남아 복습할 수 있다.

**FREE 시작**

```json
{ "projectId": "prj-01", "mode": "FREE", "topicKey": "DATABASE" }
```

**FREE 다음 묶음 선택** — *이전에 DATABASE를 골랐어도 다음에 다시 선택한다.*

```json
{ "topicKey": "DATABASE" }
```

`topicKey`는 해당 Snapshot에서 **실제 코드 근거가 확인되어 선택 가능하다고 내려준 분야**만 허용한다. 네트워크처럼 세부 분야를 별도로 표시할지는 제공할 분야 목록의 설계에 따른다.

### 4.2 모의면접(MOCK)

1. I-01에서 `{ "projectId": "prj-01", "mode": "MOCK" }`로 시작하며 사용자가 분야를 지정하지 않는다.
2. AI가 분석된 실제 프로젝트를 보고 **질문 묶음 5개**에 사용할 분야·질문을 구성한다. 5개가 반드시 서로 다른 분야일 필요는 없다.
3. 질문 묶음마다 큰 질문 1개와 꼬리질문 0~3개를 진행한다. 다음 질문 묶음은 **서버가 자동으로 진행**하므로 I-05를 사용하지 않는다.
4. 현재 회차의 질문·답변은 활용하되 **과거 다른 세션의 질문·답변·평가를 새 MOCK 출제에 이용하지 않는다**.
5. **5묶음 모두 응시하고 평가 5건 및 종합 평가 1건이 준비된 뒤에만** 결과를 한꺼번에 공개한다. 완료 전 I-03 응답에도 부분 평가를 넣지 않는다.
6. `PAUSE`/`RESUME`은 가능하다. 중도 포기는 사용자가 확인한 후 I-07로 **미완료 회차 기록을 삭제**하며 평가를 제공하지 않는다.

### 4.3 질문·답변을 어떻게 기억하나?

**Interview DB**에 세션 → 질문 묶음 → 질문·사용자 답변 → 묶음 평가를 저장한다. MOCK의 종합 평가는 세션에 연결한다.

- 답변 제출 시 **답변을 먼저 저장**하고, 현재 질문 묶음의 질문·답변과 면접 시작 시 확보한 코드 근거를 AI에 전달한다.
- 다음 꼬리질문과 묶음 평가는 생성 후 저장한다. AI가 실패해도 저장된 답변은 지우지 않는다.
- 사용자가 같은 답변을 두 번 전송해도 `submissionId`로 중복 처리하지 않도록 한다. 재시도 시 중복 질문·평가 생성도 막는다.
- FREE의 다음 묶음은 과거 출제 질문을 조회하고, MOCK은 과거 **다른 세션**의 생성 컨텍스트를 읽지 않는다.
- 질문마다 Project Service를 다시 호출하지 않는다. 면접 세션은 시작 시 선택한 Snapshot을 계속 사용한다.

**답변 제출 예시**

```json
{ "questionId": "q-02", "answer": "인덱스는 조회 성능 개선을 위해 추가했습니다.", "submissionId": "submit-001" }
```

**면접 상세 조회 응답 예시(진행 중 MOCK)**

```json
{
  "interviewId": "int-01",
  "mode": "MOCK",
  "status": "PAUSED",
  "snapshotId": "snap-01",
  "completedBundles": 1,
  "currentQuestion": { "id": "q-02", "text": "인덱스를 사용한 이유는 무엇인가요?" }
}
```

*실제 응답에는 필요한 질문·답변 이력과 사용자에게 보여 줄 상태를 포함한다. 진행 중 MOCK 평가 정보는 반환하지 않는다. AI 생성 중이면 `PREPARING` 또는 평가 대기 상태를 보여주고 I-03을 다시 조회한다.*

## 5. DevField Auth Service — 공통 계정 관리 API 4개 (구성 확정)

| 번호 | API (경로 제안) | 언제 사용하나? | 보내는 값 → 받는 값 | 필수 제한 |
|---|---|---|---|---|
| A-01 | `GET /api/account/me` | 마이페이지·탈퇴 진행 상태 확인 | 별도 입력 없음 → 본인 계정·이메일 로그인 가능 여부·연결 소셜 목록·`deletionStatus` | 다른 사람 정보 조회 금지 |
| A-02 | `POST /api/account/social-links/{provider}` | 로그인 상태에서 **소셜 수단 연결 시작** | `provider` → 제공자 본인 인증 진행 정보 | 제공자 인증 완료 전 연결 처리 금지 |
| A-03 | `DELETE /api/account/social-links/{provider}` | 로그인 수단 연결 해제 | `provider` → 해제 결과 | **마지막 사용 가능한 로그인 수단** 해제 금지 |
| A-04 | `DELETE /api/account/me` | **회원 탈퇴 신청·상태 재확인** | 본인 재확인 → 진행 중 `202`와 `DELETING`, 전 서비스 삭제 완료 후 `204` | 서비스별 완료 확인 전 계정 최종 삭제·완료 표시 금지 |

- `provider`는 `KAKAO` / `GOOGLE` / `GITHUB`를 사용한다. 같은 이메일이라는 이유로 계정을 자동 병합하지 않는다.
- 다른 DevField User ID에 이미 연결된 소셜 계정은 자동 연결·데이터 병합하지 않는다.
- Auth는 공통 User ID·이메일 로그인 수단·SocialIdentity·인증 상태를 소유한다. 로그인한 **기존 계정**에서 본인 확인과 제공자 인증 후 명시적으로 연결하며, 다른 User ID에 연결된 로그인 수단을 자동 이전하지 않는다. Kakao 이메일 미제공 문제를 가짜 이메일로 해결하지 않는다.
- 이메일·비밀번호로 이미 로그인할 수 있다면 소셜 로그인 수단 하나를 해제할 수 있지만, **로그인 방법이 0개가 되는 해제는 차단**한다.
- 소셜로 가입한 사용자도 이메일 확인·비밀번호 설정 후 **기존 User ID에 이메일 로그인 수단을 추가**할 수 있어야 한다(구체적인 화면/경로는 Auth 상세 계약에서 결정).
- 계정 연결·해제·탈퇴는 중요 작업이므로 필요하면 비밀번호 재입력 또는 최근 로그인 등으로 본인 여부를 다시 확인한다.

### 5.1 회원 탈퇴 연동 — Auth가 조정

1. A-04는 로그인한 **본인의 재확인**을 거쳐 공통 User ID를 서버에서 얻는다. 요청자가 임의의 User ID를 지정해 다른 계정을 삭제할 수 없다. 공통 계정 소유자는 계정을 `DELETING`으로 기록하고 새 로그인·프로젝트·면접·답변·AI 결과 저장을 차단한다. 진행 중에는 A-01과 동일 사용자의 A-04 재요청만 탈퇴 상태 확인·재시도로 허용한다.
2. 공통 계정 소유자는 X-03과 X-04를 각 소유 서비스에 요청한다. Project는 해당 User ID의 모든 등록 프로젝트·Snapshot·분석 자료를, Interview는 해당 User ID의 모든 면접 세션·질문·답변·평가·면접용 Snapshot 자료를 각자 삭제한다. 두 서비스는 삭제 차단 상태를 먼저 남기고 진행 중 분석·AI 작업의 늦은 저장을 막는다. 서로의 DB를 직접 읽거나 삭제하지 않는다.
3. 각 서비스는 **해당 사용자의 소유 데이터가 정리되고 늦은 재생성이 차단된 경우에만 `204 No Content`**를 반환한다. 이미 처리한 동일 요청에도 `204`를 반환한다. 일부 실패·타임아웃이면 공통 계정은 `DELETING`을 유지하고 안전하게 재시도한다. 한 서비스의 완료만으로 탈퇴 완료를 알리지 않는다.
4. 현재 MVP에서 공통 계정은 **Project와 Interview 양쪽의 완료 응답**을 확인한 뒤 연결된 소셜 Identity와 이메일 로그인 수단·세션을 정리하고 탈퇴를 완료한다. 완료 전에는 A-04가 `202`를 반환하며 A-01에 `deletionStatus=DELETING`을 표시한다. 완료된 A-04는 `204`를 반환하고 기존 세션은 종료된다. 완료 후 별도 상태 조회 API는 두지 않는다.
5. 이후 AlgoField·InfraField 등 다른 제품이 실제 사용자 데이터를 소유하게 되면 해당 제품도 탈퇴 완료 확인 대상에 등록한다. 탈퇴 작업의 상태·재시도 기록은 개인정보를 최소화해 관리하고, 백업·법정 보관의 구체적 범위와 시점은 별도 정책에서 정한다.

**프로젝트 삭제와 구분:** P-06은 Project가 X-02로 해당 프로젝트의 Interview 기록을 정리한다. A-04는 공통 계정 소유자가 **사용자의 모든 프로젝트와 면접 기록**을 X-03·X-04로 각각 정리한다. 두 흐름은 반복 호출과 동시 요청에도 다른 사용자 데이터에 영향을 주지 않아야 한다.

이 문서에서 **공통 계정 소유자**는 [ADR-002](../../adr/0002-devfield-common-authentication.md)의 **Auth Service**다. A-04·X-03·X-04의 기존 행과 완료 조건은 유지한다. Auth는 `DELETING`과 재시도를 조정하고 양쪽의 완료 전 탈퇴 완료를 표시하지 않는다. Project/Interview는 각자의 사용자 데이터 삭제와 신규·늦은 쓰기 차단을 담당한다. 유효하게 서명된 오래된 JWT와 계정 상태 변경의 관계, 즉시 폐기·상태 전파는 후속 계약 및 실제 통합 검증 대상이다.

### Auth 상세 계약에서 정할 기능 (반드시 구현 또는 연동)

| 필수 사용자 기능 | 기본 동작 | 현재 경로 |
|---|---|---|
| 이메일·비밀번호 회원가입 | 이메일 소유권 확인, 최초 DevField User ID 생성 | 미정 |
| 이메일·비밀번호 로그인 | 비밀번호 확인 후 세션 시작 | 미정 |
| Kakao·Google·GitHub 로그인 | 제공자 인증 결과를 DevField User ID에 연결 | 미정 |
| 이메일 확인 | 일회용 인증 링크/코드 | 미정 |
| 비밀번호 분실·재설정 | 본인 확인 후 안전하게 재설정 | 미정 |
| 로그인 중 비밀번호 변경 / 소셜 계정에 이메일 로그인 추가 | 재인증·이메일 확인 후 반영 | 미정 |
| 로그아웃 | 현재 DevField 로그인 세션 종료 | 미정 |

**인증 보안:** 비밀번호 원문을 DB·로그에 저장하지 않고 안전한 해시 방식을 사용한다. 이메일만 일치한다고 다른 계정을 합치지 않는다. 로그인 반복 실패, 비밀번호 재설정 및 OAuth 콜백 공격을 방어한다. Auth는 Spring Boot 4.x + Spring Security 7.x로 구현하며 Better Auth를 Production에 채택하지 않는다. JWT·세션 수명, refresh·즉시 폐기, 키 저장·교체·JWKS 운영과 Production 브라우저 보안 구성은 아직 결정하지 않았다.

## 6. 서비스 간 내부 API — 기존 4개 (X-01~04 ID 유지)

| 번호 | 호출 방향 · API | 목적 | 핵심 동작 |
|---|---|---|---|
| X-01 | Interview → Project `GET /internal/projects/{projectId}/interview-data` | 새 면접 시작 시 **분석 자료 받기** | 사용자 소유권·접근/동의 상태·최근 유효 통합 Snapshot 검증 후 `snapshotId`, 구성원별 `commitSha`, 분야·근거 반환 |
| X-02 | Project → Interview `DELETE /internal/projects/{projectId}/interviews` | 프로젝트 삭제 시 **관련 면접 기록 지우기** | 새 세션·답변·평가 저장 차단 → 관련 기록·캐시 정리 → 완료 시 `204 No Content` |
| X-03 | 공통 계정 → Project `DELETE /internal/users/{userId}/data` | 회원 탈퇴 시 **해당 사용자의 모든 Project 데이터 삭제** | 신규·늦은 쓰기 차단 → 그 사용자 프로젝트·Snapshot·분석 자료 정리 → 완료 시 `204 No Content` |
| X-04 | 공통 계정 → Interview `DELETE /internal/users/{userId}/data` | 회원 탈퇴 시 **해당 사용자의 모든 Interview 데이터 삭제** | 신규·늦은 쓰기 차단 → 그 사용자 세션·질문·답변·평가·면접용 자료 정리 → 완료 시 `204 No Content` |

- X-01에서 새 면접은 **마지막 정상 Snapshot**을 사용한다. 재분석 진행/실패 시 이전 정상 Snapshot이 있다면 활용하고, 기존 면접의 버전은 바꾸지 않는다. Private 근거를 포함한 세션은 이후 민감 읽기/쓰기 전에도 X-01로 해당 Snapshot의 현재 접근 유효성을 재검증한다(10.3절).
- X-02는 **여러 번 호출해도 결과가 같아야** 한다(멱등). 타임아웃 시 Project는 `DELETING`을 유지하고 안전하게 재시도한다. 삭제 완료 후 늦게 온 AI 결과가 데이터를 복구하면 안 된다.
- 내부 API는 **서비스 간 인증과 호출 권한 검증**을 요구한다. 사용자 ID를 임의로 전달한 요청을 그대로 신뢰해서는 안 된다. 사용자 API의 Auth JWT 검증만으로 내부 호출 주체 또는 위임된 사용자 권한이 증명되지는 않는다. 구체적인 서비스 간 인증·사용자 위임 방식은 미정이다.
- 별도 내부 Snapshot 재조회·삭제 상태 조회 API는 MVP에서 만들지 않는다.
- X-03·X-04는 공통 계정 소유자의 **인증된 서비스 호출**만 허용하며, 경로의 User ID가 실제 탈퇴 대상인지 검증한다. 반복 요청은 멱등이고 일부 실패 시 `5xx` 등 재시도 가능 상태를 반환한다. 각 서비스는 다른 사용자 소유 데이터를 삭제하지 않는다. 회원 탈퇴의 완료 판단과 재시도 조정은 공통 계정 소유자가 담당한다.

## 7. 공통 상태·오류 (간단 규칙)

| 응답 | 의미 · 예시 |
|---|---|
| `200` / `201` | 정상 조회·변경 / 새 프로젝트·세션 생성 |
| `202` | 분석·질문 생성·프로젝트 삭제·회원 탈퇴 등이 **접수됐지만 진행 중**; 탈퇴는 `DELETING` 안내 |
| `204` | 삭제 등 처리 완료, 반환할 본문 없음; 회원 탈퇴는 모든 소유 서비스의 완료 확인 후 |
| `400` | 필수 값 누락, 잘못된 입력 |
| `401` / `403` | 로그인 또는 서비스 인증 없음 / 허용되지 않는 요청 |
| `404` | 대상 없음 또는 타인 소유 대상(존재 유무 노출 방지) |
| `409` | 중복 등록, 분석·삭제 중, 면접 진행 상태 충돌 |
| `5xx` | GitHub·LLM·서버 장애. 답변·진행 상태를 보존하고 안전하게 재시도 |

**주의:** 위 코드는 대표 예시다. `POST`가 202를 반환하더라도 요청의 실제 등록·저장 성공 여부는 상태/ID를 통해 구분해야 한다. LLM 실패로 답변을 잃거나, MOCK 진행 중 평가가 새어 나가거나, 삭제 중 기록이 재생성되는 일은 허용하지 않는다.

## 8. 출시 전에 반드시 확인할 사용자 흐름

1. 가입(이메일 및 세 소셜) → 로그인 → 본인 프로젝트 목록만 확인.
2. GitHub URL 등록 → 분석 중 표시 → READY → FREE/MOCK 선택.
3. 재분석 중/실패에도 이전 Snapshot 보존 → 새 면접은 성공한 최신 버전, 기존 세션은 이전 버전 유지.
4. FREE에서 매 질문 묶음마다 같은/다른 분야 선택 → 중복 질문 최소화 → 묶음별 평가 즉시 확인.
5. MOCK에서 AI 자동 분야 선택 → 질문 묶음 5개 → 완료 전 평가 비공개 → 끝나면 평가 5개 + 종합 평가 1개.
6. 답변 제출 뒤 오류·새로고침·일시 중단 → 답변 보존·중복 처리 없음·정확한 위치에서 재개.
7. 면접 포기·개별 삭제와 프로젝트 전체 삭제를 구분하고, 삭제 중 늦은 기록 재생성 차단.
8. 소셜 계정 명시적 연결·해제, 마지막 로그인 수단 제거 거부, 이메일 기반 자동 병합 없음.
9. 회원 탈퇴 중 Project/Interview 신규·늦은 저장 차단 → 각 서비스별 완료 확인 → 그 뒤 계정·로그인 수단 정리; 한쪽 실패 시 진행 상태·재시도, 타 사용자 데이터 보존.
10. Vercel Production에서 브라우저 로그인 → 분석 → 면접 → 평가·이력까지 실제 E2E 확인.

## 9. 아직 정해야 하는 것

- **Auth 상세 계약:** 독립 Auth Service·Spring Boot 4.x + Spring Security 7.x·RS256 사용자 JWT는 결정했다. 실제 회원가입·로그인·OAuth 콜백·비밀번호 관리 Endpoint, 실제 issuer·audience, JWT·세션 수명·refresh·즉시 폐기, 키 저장·교체·JWKS 운영은 후속 설계다. 현재 A-01~04 경로 제안의 최종 필드도 후속 계약 대상이다.
- **브라우저 및 내부 권한 경계:** 현재 RepoQ Web의 BFF·Origin/CSRF 경계는 구현되어 있다(10절). Production 배포 설정 검증과 미구현 내부 서비스 간 인증·사용자 위임은 후속 대상이다.
- **회원 탈퇴 연동:** X-03·X-04와 완료 확인 원칙은 정의했다. Auth의 재시도 간격·작업 상태 저장 및 `DELETING` 전파·늦은 JWT 접근 차단 방식, 백업·법정 보관 범위는 후속 상세 설계가 필요하다. 향후 제품이 사용자 데이터를 소유하면 별도 삭제 계약과 완료 확인을 추가한다.
- **AI 및 DB 설계:** 질문 생성과 평가의 백그라운드 실행·재시도, 요청 중복 방지의 실제 구현, 데이터 보관/삭제, Snapshot 보존 범위, 물리 DB 테이블.
- **물리 운영:** Auth와 RepoQ의 물리 DB·스키마·마이그레이션·호스팅 및 RepoQ의 나머지 기술 스택은 미정이다.
- **제품 정책 문서:** 이메일·비밀번호 회원가입과 회원 탈퇴 연동은 [PRD v1.1 개정 초안](../../products/repoq/PRD.md)에 반영했다. 기존 승인 기준선 v1.0은 Git 이력에서 확인할 수 있다.
- **문서 통합:** 이 문서는 기존 Project–Interview API v0.3의 X-01·X-02를 포함한 통합 검토 초안이다. 기존 파일은 중복을 줄이기 위해 제거했으며 과거 내용은 Git 이력에서 확인할 수 있다.

**설계 원칙:** 불필요한 API를 늘리지 않고, 검증된 사용자 흐름부터 구현한다. 미확정 기술·예외 처리는 필요한 순간에 구체화하며, 기존 PRD의 FREE/MOCK·Snapshot·데이터 소유권을 훼손하지 않는다.

**게시 상태:** 이 `v0.2 + ADR-002 인증 개정안`은 기준 Commit `ad4b505d25b609a681b2726d590881c7cfb7affb`에 GitHub 게시됐다. 이 문서에서 연결한 Auth API·ERD v0.1은 아직 로컬 작성 초안이다.

## 10. Multi-Repository·Private P0 확장 계약 (2026-10-29)

이 절은 제품 설계와 현재 구현을 구분한다. **2026-11-05 로컬 기준:** RepoQ `fix/private-consent-revocation-contract` / `91d02d8`, DevField-Web `feature/repoq-private-repository-web` / `ca5fcc8`. 원격 main 반영 여부나 실제 GitHub Private E2E 통과를 뜻하지 않는다. 기존 단일 공개 저장소 예시는 호환 요청이다.

현재 Backend 사용자 API는 JWT/JWKS 검증·AUTH-15 세션 확인·DevField User ID 소유권 검사를 요구한다. Web BFF는 HttpOnly 세션에서 서버용 JWT를 얻고 변경 요청에 Origin/CSRF를 검증한다. 브라우저 callback/setup GET은 일회용 Attempt Cookie·원 세션 결합으로 검사하며 Backend에는 인증된 완료 POST로 전달한다. 타인 소유 Project·연결 항목은 `404`로 숨긴다. GitHub 소셜 로그인은 코드 접근 권한이 아니다.

**현재 구현 범위:** 등록·추가·제거, 안전한 목록/상세, GitHub App 연결·동의 조회/철회·연결 해제와 접근 차단이다. 분석 Worker·통합 Snapshot·Interview 데이터 삭제 및 X-05 전파는 미구현이며 아래 관련 동작은 향후 설계다. 실제 GitHub Private E2E는 미검증이다. `PROJECT_PRIVATE_REGISTRATION_ENABLED`와 `WEB_PRIVATE_REGISTRATION_ENABLED`는 기본 비활성이고 Public 등록은 App 연결·Private 동의를 요구하지 않는다. 등록 Gate와 기존 동의 철회·연결 해제는 분리한다.

### 10.1 기존 P API 변경

| API | 확장 계약 |
|---|---|
| P-01 `POST /api/projects` | `name?`, 비어 있지 않은 `repositories:[{repositoryUrl 또는 githubRepositoryId, consents?}]`를 받는다(BR-35). Public은 URL, Private은 P-11에서 선택한 안정 ID를 사용한다. `consents`에 `PRIVATE_COLLECTION`과 `EXTERNAL_AI`의 **각각 명시적** 결정·정책 버전을 함께 제출할 수 있어 별도 P-12 선행 호출이 필수는 아니다. Private 수집 동의·사용자/App 권한을 확인하고 동의 기록·Project·연결·최초 Job을 원자 저장한다. `201 {id,analysisStatus:"QUEUED",repositories:[{id,githubRepositoryId,visibility,analysisStatus}]}`. 기존 `{repositoryUrl,name?}`는 단일 Public 요청으로 정규화하고 기존 `id`,`analysisStatus`를 유지한다. |
| P-02 `GET /api/projects` | 본인 Project만 반환한다. 기존 `id`,`name`,`analysisStatus`와 `repositoryCount`, `privateRepositoryCount`, `canInterview`를 제공한다. Private 코드·경로는 목록에 싣지 않는다. |
| P-03 `GET /api/projects/{id}` | 별도 분석 상태 API 없이 Project의 통합 `analysisStatus`·Job 상태와 `repositories:[{id,githubRepositoryId,fullName,visibility,accessStatus,analysisStatus,commitSha?}]`를 함께 반환한다. 최신 유효 결과는 `latestReadySnapshot:{snapshotId,members:[{repositoryId,githubRepositoryId,commitSha}],topics,relations}`. 단일 최상위 `commitSha`는 단일 구성 호환 응답에서만 사용한다. 기존 `canInterview`는 **허용된 범위에서 면접 가능 여부**, 신규 `canPrivateInterview`는 Private 근거 포함 면접 가능 여부다. Private 상세·Snapshot 제공 직전 GitHub 권한과 동의 재검사에 실패하면 민감 필드를 제외하고 fail closed한다. 철회·구성 불일치 후 유효 Snapshot이 없으면 둘 다 false; A 단계는 `QUEUED`, Snapshot `null`. |
| P-05 `POST /api/projects/{id}/reanalyze` | 현재 연결 구성 버전을 고정한 **하나의** Project 분석 Job을 `202`로 접수한다. Repository별 진행은 그 Job의 상태에서 P-03으로 조회한다. 활성 Job 중복은 `409`, 같은 `Idempotency-Key` 재시도는 기존 Job을 반환한다. 일부 Repository 실패 시 새 통합 READY를 게시하지 않는다(BR-34). |
| P-06 `DELETE /api/projects/{id}` | 기존 Project 삭제 계약을 유지하며 모든 연결·Snapshot·동의·GitHub 연결 참조 및 Interview 자료를 소유권 범위에서 정리한다. 다른 Project가 공유하는 GitHubConnection을 임의 삭제하지 않는다. `202` 진행, 반복 요청은 같은 삭제 상태 또는 완료 `204`. |

### 10.2 최소 신규 사용자 API

| ID | 메서드·경로 | 요청·성공·주요 실패 |
|---|---|---|
| P-07 | `POST /api/projects/{id}/repositories` | P-01과 같은 `repositories:[{repositoryUrl 또는 githubRepositoryId,consents?}]`로 여러 연결을 한 번에 추가한다. Private 동의를 요청에 함께 기록할 수 있다. 기존 Job이 `QUEUED`이면 `CANCELLED` 후 새 구성의 `QUEUED` Job을 생성하고, `RUNNING`이면 `409`, 종료 상태이면 새 `QUEUED` Job을 생성한다. 성공 시 `201` 연결 항목·새 구성 버전·QUEUED Job, 같은 `Idempotency-Key` 재시도는 동일 응답. 중복·구성 충돌 `409`, 무권한 Private `403`, 타인 Project `404`. |
| P-08 | `DELETE /api/projects/{id}/repositories/{memberId}` | `memberId`는 P-03의 Project Repository 연결 UUID이며 GitHub Repository ID가 아니다. 현재 본인 연결 제거 성공 및 이미 제거된 본인 연결의 반복 요청은 **`204 No Content`**, 본문 없음. 마지막 활성 연결 제거는 `409 last_repository_required`. 일반 Public 제거에서 기존 Job이 `QUEUED`이면 `CANCELLED` 후 새 구성의 `QUEUED` Job을 생성하고, `RUNNING`이면 `409`, 종료 상태이면 새 `QUEUED` Job을 생성한다. Private 연결 제거는 `RUNNING`이어도 관련 활성 Job을 DB에서 취소하고 구성 버전을 갱신한다. 남은 Private 접근이 유효할 때만 새 `QUEUED` Job을 생성한다. 미래 Snapshot·Interview 비동기 정리의 `202`는 현재 구현 완료 응답이 아니다. |
| P-09 | `POST /api/github/connections/start` | 본문 없이 시작. Backend `200 {nextAction,url,proof,expiresAt}`; `nextAction=AUTHORIZE_USER/INSTALL_APP`. Web BFF는 Attempt Cookie를 저장하고 브라우저에 `200 {nextAction,url,expiresAt}`만 반환한다. 사용자 승인·App 설치는 별도 상태이며 Public에는 불필요하다. |
| P-10 | Web BFF `GET /api/github/connections/callback`, `GET /api/github/connections/setup` | callback은 `code,state`(선택 `iss`), setup은 `installation_id,setup_action`을 받는다. BFF가 원 세션·Attempt를 확인한 뒤 아래 인증된 완료 POST로 전달한다. 브라우저 응답은 결과를 표시할 연결 관리 화면으로 **`303` 리다이렉트**다. |
| P-11 | `GET /api/github/repositories` | Backend `page`(기본 1, 1~10000), `perPage`(기본 30, 1~100). `200 {status,repositories:[{id,fullName,name,isPrivate}],page,hasNext}`; `id`는 숫자. Web은 `page`만 받고 `perPage=30`으로 호출하며 `isPrivate`를 `private`로 정규화한다. 별도 연결 상태 GET은 아래에 명시한다. |
| P-12 | `PUT /api/github/repositories/{githubRepositoryId}/consents` | `consents:[{purpose,decision,policyVersion}]`. 두 목적 제출을 유지하고 **한 목적만 보내는 경우 REVOKED만 허용**한다. 일반 갱신은 `200 {revocationId:null,status:"UPDATED"}`, 새 철회 또는 반복 철회를 포함하면 `202 {revocationId:<UUID>,status:"BLOCKED"}`. 목적별 전이·버전·멱등성은 아래 상세 계약을 따른다. |
| P-13 | `DELETE /api/github/connections/{connectionId}` | 본인 연결 UUID, 본문 없음. `202 {connectionId:<UUID>,status:"BLOCKED"}`; 반복 요청도 같은 ID·상태. 관련 Private 접근 차단·활성 Job DB 취소·구성 버전 갱신·로컬 자격증명 제거 및 연결 generation 증가를 원자 처리한다(반복 해제는 generation을 다시 증가시키지 않음). GitHub App 자체 제거 또는 DevField 소셜 로그인 해제가 아니다. X-05/Interview 삭제 완료를 의미하지 않는다. |

P-07/P-08의 Job 취소·구성 버전 변경·새 Job 생성은 원자적 DB 트랜잭션으로 처리한다. P-05의 기존 활성 Job(`QUEUED`/`RUNNING`) `409` 정책은 유지한다. 일반 Public 연결 제거의 A 단계 `RUNNING` Job `409`는 Private 연결 제거·권한·동의 철회에 적용하지 않는다. Private 연결 제거·철회 시 진행 중 Job으로 요청을 거부하지 않고 관련 활성 Job을 DB에서 취소하며 민감정보 접근을 즉시 차단한다. Worker 취소 확인, Snapshot 무효화, X-05 연동은 후속 구현 범위다.

Private 생성·추가 시 `consents`를 요청에 포함하면 별도 P-12를 먼저 호출할 필요가 없다. 수집 동의 `GRANTED`가 없으면 Private 연결을 거부한다. 외부 AI 동의는 독립적으로 `GRANTED` 또는 거부할 수 있으며, 거부 시 Private 코드 외부 전송과 그 근거를 사용하는 AI 면접을 차단한다(BR-33). Public 전용 분석·면접은 계속 사용 가능하다. P-12는 이후 동의 변경·철회와 Project 생성 전 선택적 동의에 사용한다. 실제 외부 AI 제공자·보관기간은 `[결정 필요]`다.

#### P-09~P-11 완료·상태 조회

| 구분 | Backend 메서드·경로 | 요청 / `200` 응답 |
|---|---|---|
| 사용자 승인 완료 | `POST /api/github/connections/callback/complete` | `{code,proof}` → `{userAuthorization,installation,connectionId}` |
| 설치 완료 | `POST /api/github/connections/setup/complete` | `{proof,installationId,setupAction}` (`installationId`: 숫자, `setupAction`: `install/update`) → 같은 연결 상태 DTO |
| 연결 상태 조회 | `GET /api/github/connections` | 본문 없음 → 같은 연결 상태 DTO; Web BFF도 같은 GET 경로·필드 제공 |

완료 POST는 JWT·세션 인증이 필요하며 Backend에서 일회성·만료·사용자/세션·연결 generation을 확인한다. 사용자 승인에는 PKCE를 사용하고, 설치 redirect 값만으로 설치를 승인하지 않고 GitHub 사용자/App 접근을 검증한다. Web GET callback/setup을 Backend의 비인증 완료 API로 해석하지 않는다. 경로는 구현됐으며 Production 호스트/Provider 설정·실제 E2E는 별도 검증 대상이다.

- `userAuthorization`: `NOT_AUTHORIZED / AUTHORIZED / REAUTHORIZE / DISCONNECTED`.
- `installation`: `NOT_INSTALLED / PENDING_APPROVAL / INSTALLED`.
- `connectionId`: UUID 문자열 또는 연결 이력이 없으면 `null`. P-13 후에는 같은 ID와 `DISCONNECTED`, `NOT_INSTALLED`를 반환한다.
- P-11 목록 `status`: `CONNECTED / USER_AUTHORIZATION_REQUIRED / INSTALLATION_REQUIRED / INSTALLATION_PENDING`. `CONNECTED`가 아니면 `repositories:[]`, `hasNext:false`. 목록은 연결 상태 GET을 대체하지 않는다. Repository 이름은 현재 GitHub 권한으로 확인된 목록에만 제공하며 동의 GET에는 포함하지 않는다.

#### P-12 목적별 결정·철회

- 목적은 `PRIVATE_COLLECTION`, `EXTERNAL_AI`; 결정은 `GRANTED / DENIED / REVOKED`. 두 목적 요청은 중복 없이 각 목적 하나씩, 부분 요청은 **단일 REVOKED만** 가능하다. 단일 GRANTED/DENIED는 `400 invalid_request`다.
- 새 GRANTED/DENIED 이벤트는 서버의 `PROJECT_PRIVATE_CONSENT_POLICY_VERSION`과 일치하는 버전이 필요하다. 미설정·불일치면 Backend `400 invalid_request`(Web 사전 검사에서는 `409 consent_policy_unavailable`). 사용자가 명시적으로 선택한 새 GRANTED는 현재 GitHub 사용자·App 설치·대상 Repository 권한과 기존 연결/Project 장벽을 재검증한다. 과거 GRANTED를 현재 버전으로 자동 인정하지 않는다.
- 동일 목적·동일 결정·동일 정책 버전 반복은 이벤트를 추가하지 않는다. 따라서 두 목적 요청에서 한쪽 철회와 다른 쪽의 동일 결정·동일 버전을 함께 보내도 다른 쪽은 재승인하지 않는다. **동일 GRANTED라도 정책 버전이 달라지면** 새 버전에 대한 명시적 동의와 검증 후 이벤트를 추가한다. Web은 두 목적 결정을 각각 선택하게 하며 과거 결정을 새 버전으로 자동 갱신하지 않는다.
- REVOKED는 본인 해당 목적의 최신 GRANTED 또는 REVOKED 이력이 있어야 한다. 이력이 없거나 DENIED인 경우 `403 consent_not_owned`. 현재 정책 미설정 또는 GitHub 권한 상실 상태에서도 철회할 수 있다. Web은 `policyVersion:null`로 보내며 Backend는 호출자 버전 대신 **저장된 이전 이벤트의 정책 버전**을 사용한다. 기존 두 목적 요청의 REVOKED에 버전 문자열이 있어도 이를 신뢰하지 않는다.
- 단일 목적 철회는 다른 목적의 결정·정책 버전·이벤트를 변경하지 않는다. PRIVATE_COLLECTION 철회는 관련 Private 접근을 BLOCKED로 바꾸고 활성 Job을 DB에서 취소하며 구성 버전을 갱신한다. EXTERNAL_AI 철회는 외부 AI 동의를 무효화하지만 유효한 PRIVATE_COLLECTION까지 철회하지 않는다. Public 기능은 유지된다.
- 반복 REVOKED는 새 이벤트 없이 기존 `revocationId`를 반환한다. 응답은 요청 단위의 ID 하나이며 목적별 상태는 동의 GET으로 확인한다. `202 BLOCKED`는 파생 자료 삭제 완료나 해당 Repository의 모든 목적이 철회됐다는 뜻이 아니다.
- 최신 REVOKED의 정리 상태가 **NO_DERIVED_DATA일 때만** 새 결정 절차를 진행할 수 있다. PENDING 등 미완료·알 수 없는 상태는 `409 revocation_in_progress`. 기존 GRANTED/REVOKED를 DENIED로 덮는 전이는 `409 revocation_required`다. 정리 상태만으로 권한이 열리지 않으며, 재동의는 새 명시적 GRANTED·현재 정책·현재 GitHub 권한·기존 보안 장벽을 모두 확인한다. 과거 이벤트는 삭제/덮어쓰기하지 않는다.
- 동의 이벤트와 접근 차단은 USER → PROJECT 잠금 순서를 유지한 단일 트랜잭션으로 처리한다. Worker·Snapshot·Interview 삭제는 아직 없고 현재 새 철회 이벤트는 `cleanupState=NO_DERIVED_DATA`를 기록한다.

단일 목적 철회 요청 예시:

```json
{"consents":[{"purpose":"EXTERNAL_AI","decision":"REVOKED","policyVersion":null}]}
```

#### 동의 상태 조회 (P-12와 같은 경로의 GET)

`GET /api/github/repositories/{githubRepositoryId}/consents` → `200`. ID는 양의 십진수 문자열(`[1-9][0-9]{0,18}`); 잘못된 형식은 `400`. Project에 등록하기 전에도 조회 가능하다.

```json
{
  "githubRepositoryId": "123456",
  "currentPolicyVersion": null,
  "consentSubmissionAvailable": false,
  "consents": [
    {"purpose":"PRIVATE_COLLECTION","decision":null,"policyVersion":null,"cleanupState":null},
    {"purpose":"EXTERNAL_AI","decision":null,"policyVersion":null,"cleanupState":null}
  ]
}
```

- `currentPolicyVersion`: 현재 서버 정책 문자열, 미설정이면 `null`. `consentSubmissionAvailable`은 **현재 정책 설정 여부만** 나타내며 Gate·GitHub 권한·정리 완료·실제 GRANTED 허용 여부를 보장하지 않는다. 예시의 `false`는 새 결정을 제출할 수 없음을 뜻한다.
- `consents`: 두 목적의 최신 이벤트를 본인 DevField User ID 범위에서 반환한다. 미결정은 `decision/policyVersion/cleanupState` 모두 `null`. GRANTED/DENIED는 기록 당시 버전과 `cleanupState:null`; REVOKED는 기록된 이전 정책 버전과 실제 저장된 정리 상태 문자열을 반환한다. 과거 버전을 현재 버전으로 치환하지 않는다.
- `NO_DERIVED_DATA`는 현재 파생 데이터가 없다는 의미이며 재동의 절차를 시작할 수 있는 상태다. 삭제 작업을 수행했다거나 접근 권한을 부여했다는 뜻이 아니다. 미완료·알 수 없는 정리 상태에서는 재동의 가능으로 판단하지 않는다. Web은 잘못된 DTO를 거부하고 조회 실패 시 이전 상태로 Private 제출하지 않는다.
- 본인 동의 이력이 있으면 현재 GitHub 접근 권한 없이도 과거 상태를 조회할 수 있다. 이력도 접근 권한도 없으면 존재 여부를 숨기는 `404 repository_not_found`; GitHub 확인 장애는 `503` 등 오류로 닫는다. 권한이 있는 다른 사용자에게는 그 사용자의 미결정 상태만 제공하며 타인의 이력은 조회하지 않는다. 이름·경로·코드·자격증명은 반환하지 않는다.

#### P-01/P-07 등록 시 철회 장벽

Private ID는 P-11 선택 결과만으로 승인하지 않고 저장 직전 사용자/App 권한·독립 동의·Repository ID를 재확인한다. Public도 GitHub 안정 ID를 확인한다. `(owner_user_id,github_repository_id)` 활성 연결 UNIQUE와 트랜잭션으로 BR-31 경합을 막고 다른 사용자 Public 등록은 허용한다. Private 입력은 현재 버전의 두 목적 GRANTED/DENIED를 요구하고 PRIVATE_COLLECTION은 반드시 GRANTED다. 별도 P-12 선행 호출은 필수가 아니다.

두 목적 각각의 최신 이벤트를 본인 USER Write Fence 안에서 INSERT 전에 검사한다. REVOKED + 미완료/알 수 없는 정리 상태는 `409 revocation_in_progress`로 거부하며 EXTERNAL_AI를 DENIED로 덮어 철회 상태를 숨길 수도 없다. NO_DERIVED_DATA 이후에도 기존 GRANTED/REVOKED → DENIED는 `409 revocation_required`다. 따라서 외부 AI 철회를 유지한 신규 Private 등록은 현재 등록 요청으로 표현할 수 없으며 Web은 이를 안내한다. 기존 유효한 로컬 수집 동의까지 철회하는 것은 아니다.

동의·Repository·Project·Job 저장은 원자적이며 혼합 요청 실패 시 Public 항목도 부분 저장하지 않는다. USER → PROJECT 잠금으로 동시 철회와 직렬화한다. P-01/P-07의 Idempotency-Key는 사용자·경로·본문에 결합하며 다른 본문 재사용은 `409`. 기존 응답 재생도 현재 Private 접근을 재검사·가림 처리하고 철회 장벽을 복구하지 않는다. P-05 Job 멱등성은 향후 설계다. Public만 사용하는 등록·추가는 Private 동의나 App 연결을 요구하지 않는다. 외부 AI 제공자·보관기간은 미확정이고 실제 분석·면접 처리는 미구현이다.

P-01의 혼합 요청 예시(P-07은 `name` 없이 같은 `repositories` 배열 사용; 동의 버전 값은 예시이며 운영 버전 확정값이 아님):

```json
{
  "name": "내 프로젝트",
  "repositories": [
    {"repositoryUrl": "https://github.com/example/public-api"},
    {"githubRepositoryId": "123456", "consents": [
      {"purpose": "PRIVATE_COLLECTION", "decision": "GRANTED", "policyVersion": "example-version"},
      {"purpose": "EXTERNAL_AI", "decision": "DENIED", "policyVersion": "example-version"}
    ]}
  ]
}
```

### 10.3 X-01 확장·민감 데이터 철회

**향후 설계(미구현):** 아래 통합 Snapshot·Interview 민감 읽기/쓰기·X-05 삭제 계약은 현재 P-08/P-12/P-13 응답으로 완료를 보장하지 않는다.

`X-01`은 Project가 **현재 사용 가능한 통합** `AnalysisSnapshot`을 검증한 뒤 `snapshotId`, 구성 버전, `members:[{repositoryId,githubRepositoryId,commitSha}]`, 활성 분야, Repository ID·SHA·경로가 붙은 최소 근거와 양측 근거가 있는 관계를 반환한다. Interview는 서비스 자격과 원 사용자 JWT를 함께 전달하고 Project는 소유권·사용자/Project 장벽·모든 Private 접근 권한·수집 동의를 재검사한다. 선택 query `evidenceScope=PUBLIC_ONLY/FULL`의 **기본값은 `PUBLIC_ONLY`**다. `FULL`은 Private 외부 AI 동의까지 확인하고, `PUBLIC_ONLY`는 Private에서 유래한 멤버·역할·요약·근거·관계·분야를 **서버에서 완전히 제외**한다. 일부 Repository 분석 실패 시 새 READY는 없고, 이전 Snapshot은 그 구성·권한·수집 동의가 유효할 때만 사용한다. Private 외부 AI 동의가 없으면 Private 기반 AI 면접을 비활성화하되 Public 전용 면접은 `PUBLIC_ONLY`로 허용한다.

Private 근거를 가진 진행·과거 면접의 **민감 읽기·쓰기 직전**에도 Interview는 X-01의 선택 query `snapshotId`에 세션의 고정 ID를 지정해 접근 유효성을 확인한다. 이 경우 최신 Snapshot으로 자동 대체하지 않는다. Project/Auth/GitHub 확인 장애나 철회 상태이면 fail closed한다. X-05는 Interview 로컬 차단을 빠르게 전파·정리하지만 전달 지연만으로 접근 허용을 결정하지 않는다. 매 요청 재검사의 지연·호출량은 구현 시 제한하고 민감 데이터의 무효 권한 캐시를 허용하지 않는다. Public 전용 면접은 기존 고정 Snapshot 경로를 유지한다.

| ID | 호출 방향 · API | 계약 |
|---|---|---|
| X-05 | Project → Interview `POST /internal/projects/{projectId}/snapshot-revocations` | `{revocationId,affectedSnapshotIds,reasonCode}`. 인증된 Project 서비스만 호출. Interview는 대상 Snapshot의 신규·진행·과거 세션 민감 읽기/쓰기를 차단하고 해당 Snapshot에서 파생된 면접 질문·답변·평가·캐시를 안전하게 삭제한다(BR-32). `202`는 차단 완료·삭제 진행, `204`는 삭제 완료, `5xx`는 재시도. 같은 `revocationId`는 멱등이며 타 사용자·다른 Snapshot은 삭제하지 않는다. 백업 삭제 기한은 `[결정 필요]`. |

**현재 구현된 Webhook:** 사용자 API 번호에 포함하지 않는다. Project의 별도 수신 경로에서 `X-Hub-Signature-256` HMAC을 검증하고 delivery ID 중복을 제거하며 설치 제거·Repository 접근 변경을 차단 이벤트로 처리한다. Webhook 지연·실패에 대비해 GitHub 호출 직전 권한 재검사도 수행한다. Auth의 X-03/X-04와 Project 삭제 X-02는 기존 소유권과 완료 확인 계약을 유지한다.

### 10.4 오류·상태·미결정

현재 Backend `ApiErrors`와 Web BFF의 오류 본문은 `{code}`다. `message/requestId`는 현재 필수 응답 필드가 아니다. 오류에는 JWT, GitHub token, 서비스 키, GitHub/AI 원문 및 Private 코드·경로를 넣지 않는다. `400` 입력/동의 버전 오류, `401` 로그인·세션 실패, `403` 본인에게 허용되지 않은 GitHub 접근/동의 부족, `404` 타인 소유 Project/연결, `409` 중복·구성 버전·진행 Job 충돌, `503 github_rate_limited` GitHub 한도(외부 한도 응답도 안전한 오류로 변환), `503` Auth/GitHub 장애로 구분한다. 외부 장애 시 보호 API는 fail closed다. 현재 Private 접근 상태는 `accessStatus=BLOCKED`로 반환하며 해당 항목의 `githubRepositoryId/fullName/commitSha`는 `null`로 가린다. 동의의 `REVOKED`와 접근 상태 `BLOCKED`를 혼동하지 않는다. 현재 `latestReadySnapshot=null`, `canInterview=false`, `canPrivateInterview=false`이며 분석/면접 가능 상태로 표시하지 않는다. P-01/P-07/P-12의 철회 충돌은 `409 revocation_in_progress / revocation_required`로 구분한다.

`[결정 필요]` GitHub App callback/setup/BFF Production 배포 호스트·설정 검증, AI 제공자·원본/파생 데이터 보관기간, 백업 삭제 기한, Secret/KMS 운영값, 삭제 완료 SLA, 멱등 키 보관기간. BR-31~36의 보안·삭제·중복 원칙은 확정됐으며 운영값이 미정이라는 이유로 적용을 미루지 않는다.
