# ADR-003: RepoQ Multi-Repository와 Private GitHub 접근

- 상태: **승인** (제품 P0 방향과 BR-31~36 확정; 운영값은 미결정)
- 작성일: 2026-10-29
- 관련: [ADR-001](0001-repoq-service-decomposition.md), [ADR-002](0002-devfield-common-authentication.md), [PRD](../products/repoq/PRD.md), [API](../contracts/repoq/api-spec.md), [Project 데이터 모델](../architecture/repoq/project-data-model.md)

## 배경

정식 RepoQ MVP는 Project 하나에 여러 Public/Private Repository를 연결하고 Repository별 근거와 관계를 통합한 면접 Snapshot을 제공해야 한다. 기존 RepoQ A 단계는 단일 Public Repository 중심이며 V1 Flyway가 이미 적용되었다. Auth의 GitHub 소셜 로그인은 로그인 수단일 뿐 코드 접근 승인이나 DevField의 분석 동의가 아니다.

## 결정과 서비스 경계

ADR-001의 Project/Interview 독립 서비스와 ADR-002의 Auth 공통 인증·User ID 소유권을 유지한다. **BR-36 확정:** MVP에서는 별도 GitHub Integration Service를 신설하지 않고 thedevfield Organization 소유 GitHub App의 연결·권한·분석 동의와 Repository 분석·Snapshot을 Project Service가 소유한다. Auth는 연결 token·Private 코드·동의를 소유하지 않는다. Interview는 Project의 X-01로 유효한 통합 Snapshot의 허용된 근거만 받아 질문·답변·평가를 소유한다. 서비스 사이 DB 직접 접근·FK는 금지한다.

GitHub App은 Metadata: Read와 Contents: Read만 요청하고 Public 등록에는 설치를 강제하지 않는다. Private 접근은 DevField User ID↔GitHub User ID 연결, App 설치 범위, **GitHub App User Access Token이 가진 사용자·App 권한의 교집합**, Repository별 수집 동의를 확인한다. Installation Access Token만으로 특정 DevField 회원의 접근을 허가하지 않는다. 사용자 승인이 필요한 App 연결 flow에서 state를 로그인 세션·User ID와 결합하고 일회용·만료로 관리한다. 조직 저장소는 관리자 설치 승인과 해당 사용자 접근권을 모두 요구할 수 있다. 각 GitHub 호출 직전에 권한·동의를 재검사한다. Webhook은 `X-Hub-Signature-256` HMAC과 delivery ID 중복을 검증하고 설치 제거·Repository 선택 변경·권한 철회를 차단 상태로 반영한다.

GitHub App의 **사용자 승인 callback**(`code/state`, PKCE)과 **설치 후 setup redirect**(`installation_id/setup_action`)는 서로 다른 사건이다. 후자의 `installation_id`는 위조 가능하므로 세션에 기록한 대기 설치와 GitHub API에서 확인한 사용자·설치·Repository 권한의 교집합 없이는 연결을 확정하지 않는다. P-09/P-10은 두 사건을 한 논리 API 묶음으로 제공하되 상태를 혼동하지 않는다. GitHub App의 “설치 중 OAuth 승인 요청” 설정과 실제 callback/setup URL은 `[결정 필요]`다.

User Access Token/refresh token은 만료·갱신·암호화·키 교체를 지원하고 응답·로그에 노출하지 않는다. 원본 코드의 장기 보관을 피하고 근거는 Repository ID·SHA·경로·요약으로 남긴다. 외부 AI 전송 직전에 별도 `EXTERNAL_AI` 유효 동의를 검사하며 App 설치나 수집 동의를 외부 전송 동의로 대체하지 않는다. 저장소 콘텐츠는 비신뢰 입력으로 취급하고 AI 출력의 출처·구조를 검증한다.

외부 AI 동의가 없으면 Private 코드를 외부 LLM에 보내지 않고 Private 근거 기반 AI 면접을 비활성화한다(BR-33). 통합 Snapshot이 유효하고 Public 근거가 있으면 X-01의 `PUBLIC_ONLY` 투영에서 Private 근거·관계·분야를 서버 측에서 제외해 Public 면접을 계속 허용한다. Private 권한·동의 철회 시에는 해당 통합 Snapshot 전체와 파생 면접을 BR-32에 따라 차단·삭제하고, Public 연결의 후속 분석은 계속 가능하다.

통합 `AnalysisSnapshot`은 구성 버전과 `SnapshotMember`의 Repository별 SHA를 고정한다. 관계·역할의 구조화 결과는 통합 Snapshot에 두고 근거는 Repository ID·SHA·경로를 가진 `EvidenceReference`로 분리한다. 관계는 양측 근거를 요구하고 코드 사실·문서 주장·AI 가설을 구분한다. 기존 `analysis_job` 한 행의 제한된 Repository별 진행 상태로 충분하므로 별도 Repository Job/Snapshot/Relation 테이블을 MVP 필수로 만들지 않는다. `SnapshotMember`는 FK·철회 대상 검색·삭제 범위 때문에 유지한다. 구성 변경·삭제·철회 이후 오래된 Job이 READY를 게시하지 못하도록 USER/PROJECT 장벽·구성 버전·활성 Job UNIQUE를 사용한다. **BR-32 확정:** Private 권한·동의 철회 시 영향을 받은 통합 Snapshot과 해당 면접 세션·질문·답변·평가의 민감 접근을 즉시 차단하고 X-05로 멱등 삭제한다. 백업 삭제 기한은 아직 결정하지 않는다.

## 대안 평가

| 대안 | 장점 | 비용·위험 | MVP 판단 |
|---|---|---|---|
| A. Project가 GitHub 연결·권한 소유 | 수집·동의·Snapshot의 동일 트랜잭션 경계, 서비스 통신 최소 | Project의 보안 책임 증가; token 운영 필요 | **채택(BR-36)** |
| B. Auth가 GitHub 연결 통합 관리 | 공통 계정 UI와 가까움 | Auth에 코드 접근·분석 동의 도메인 유입, 매 호출 교차 서비스 의존 | MVP 비권장 |
| C. 별도 GitHub Integration Service | 다제품 재사용 가능 | 신규 배포·인증·장애 경계·삭제 계약 증가 | 실제 공통 수요가 확인되면 재검토 |
| OAuth App 또는 PAT | 구현 시작은 단순할 수 있음 | 사용자 토큰의 권한 범위·조직 승인·회수·최소 권한 통제가 불리 | Private MVP 기본안에서 제외 |

## 확정 정책과 운영 미결정

- **BR-31 확정:** 동일 GitHub Repository는 같은 DevField 사용자의 활성 Project 하나에만 연결한다. 다른 사용자의 동일 Public 등록은 허용한다.
- **BR-32 확정:** Private 접근·동의 철회 시 민감 접근을 즉시 차단하고 영향받은 통합 Snapshot과 면접 기록을 안전하게 삭제한다.
- **BR-33 확정:** Private 수집·분석과 외부 AI 전송 동의는 독립이다. 외부 AI 거부 시 Private 코드 전송·Private 기반 AI 면접은 금지하고 Public 사용은 허용한다.
- **BR-34 확정:** Repository 일부 실패 시 새 통합 Snapshot READY 게시 금지; 기존 유효 Snapshot은 권한·동의가 유지될 때만 재사용한다.
- **BR-35 확정:** 하나 이상 Repository를 확정한 후 Project를 생성하고 영속 DRAFT Project를 필수로 만들지 않는다.
- **BR-36 확정:** GitHub App은 thedevfield Organization 소유, 연결·권한·동의는 Project Service 소유다.
- `[결정 필요]` 외부 AI 제공자·전송 범위, 원본/파생 데이터·백업 보관기간, Secret/KMS 운영값, 조직 App 승인 운영, callback 배포 주소. 기존 Private 근거가 포함된 면접 기록 삭제 자체는 BR-32로 확정됐다.

## 결과 및 검증

새 API는 기존 P-01 단일 Public 요청과 201/QUEUED를 호환한다. Private 동의는 P-01/P-07에 함께 제출하거나 P-12에서 변경·철회할 수 있어 생성 시 불필요한 선행 호출이 없다. 별도 분석 상태 API 없이 P-03이 통합·Repository별 진행을 보여 준다. V1을 수정하지 않고 확장 Migration·백필·제약 전환을 거친다. Private 권한/동의 없는 수집, 타 사용자 연결, Webhook 위조, 오래된 Job 게시, 철회 후 X-01/면접 재접근, 기존 데이터 손실을 수용 테스트에서 거부해야 한다. 이 ADR의 정책 승인으로 GitHub App 생성·설치 또는 SQL 실행이 승인된 것은 아니다.

GitHub 근거: [App 권한과 사용자 토큰의 교집합](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app), [사용자 승인·state·PKCE](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-user-access-token-for-a-github-app), [설치 setup URL의 위조 가능 ID](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/about-the-setup-url), [Webhook 서명 검증](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries).
