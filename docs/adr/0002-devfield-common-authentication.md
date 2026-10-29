# ADR-002 — DevField 공통 인증을 독립 Auth Service로 구성

> **DevField — Practice. Solve. Grow.**  
> 결정 ID: `ADR-002`  
> 결정일: `2026-10-26`  
> 상태: **Accepted — 인증 아키텍처 결정, 구현·Production 검증 전**  
> 버전: `v1.0 (ADR-002 인증 개정안)`  
> 적용 기준 HEAD: `79efcca35818810caf07c1472f5aed072c6327e6`  
> 게시 상태: **기준 Commit `ad4b505d25b609a681b2726d590881c7cfb7affb`에 GitHub 게시됨**  
> 관련 문서: [RepoQ PRD](../products/repoq/PRD.md) · [서비스 경계](../architecture/repoq-service-boundaries.md) · [ADR-001](0001-repoq-service-decomposition.md) · [통합 API 계약](../contracts/repoq/api-spec.md) · [Auth API v0.1](../contracts/auth/api-spec.md) · [Auth ERD v0.1](../architecture/auth/data-model.md)

## 1. 배경과 결정 범위

> **후속 범위 주석:** `DevField-Auth` 저장소는 현재 존재한다. 이 ADR의 GitHub 소셜 로그인 결정은 로그인 수단에 관한 것이며 RepoQ의 Private 코드 접근·GitHub App 사용자 승인·분석 동의를 부여하지 않는다. 확정된 후속 경계는 [ADR-003](0003-repoq-multi-repository-private-access.md)에 기술한다.

RepoQ의 Project Service와 Interview Service 분리는 [ADR-001](0001-repoq-service-decomposition.md)에서 결정했다. 당시에는 DevField 공통 User ID와 SocialIdentity를 논리적 공통 책임으로 두고 인증 구현 방식과 별도 서비스 여부를 보류했다. 이메일·비밀번호 및 Google·GitHub·Kakao 로그인, 여러 제품에 걸친 동일 사용자 식별과 회원 탈퇴 조정에는 공통 인증 소유자가 필요하다. 이 ADR은 그 보류된 인증 결정을 보완한다. Project/Interview의 분리와 각 데이터 소유권은 그대로 유지한다.

## 2. 결정

1. DevField 공통 인증을 **독립 Auth Service**로 구성하고 **Spring Boot 4.x + Spring Security 7.x**로 구현한다. `DevField-Auth`는 별도 Repository로 생성할 예정이다. **이번 문서 작업에서는 Repository를 생성하거나 서비스를 구현하지 않는다.**
2. **Better Auth는 Production 인증으로 채택하지 않는다.** 이메일·비밀번호와 Google·GitHub·Kakao 로그인 요구는 유지한다. Spring Security 선택을 범용 OAuth/OIDC Provider 구축 승인으로 확대하지 않는다.
3. Auth는 공통 `DevField User ID`, 이메일 로그인 수단, `SocialIdentity`, 인증 상태와 회원 탈퇴 작업 상태·재시도를 소유한다. 동일 이메일은 자동 계정 병합의 근거가 아니다.
4. Auth가 발급한 **RS256 서명 JWT**를 사용자 API 인증에 사용한다. Project와 Interview는 Auth의 **공개키·JWKS**를 사용해 JWT의 서명, 허용 알고리즘, `iss`, `aud`, 유효기간을 검증한다. 검증된 `sub`를 DevField User ID로 사용한다. 실제 issuer·audience 값, 키 운영과 토큰 수명은 후속 계약에서 정한다.
5. Project와 Interview는 검증된 사용자 주체를 받은 뒤에도 각자의 프로젝트·세션·평가 등 **리소스 소유권을 별도로 검증**한다. 사용자 JWT 검증을 X-01~X-04 등 내부 서비스 호출의 권한 검증과 동일시하지 않는다. 서비스 간 인증·사용자 위임 방식은 후속 설계에서 정한다.
6. Auth, Project, Interview는 각자 소유한 데이터만 직접 읽고 변경한다. **다른 서비스의 DB에 직접 접근하지 않는다.** 물리 DB·스키마·마이그레이션·호스팅은 이 결정으로 확정하지 않는다.

## 3. 책임과 기존 계정 정책

| 소유자 | 책임 |
|---|---|
| Auth | 공통 User ID, 이메일 로그인 수단, SocialIdentity의 제공자 고유 ID 연결, 인증 상태, 로그인 수단 관리, 회원 탈퇴 조정 |
| Project | 등록 프로젝트·분석·Snapshot 및 사용자별 Project 데이터의 소유권·삭제 |
| Interview | 세션·질문·답변·평가 및 사용자별 Interview 데이터의 소유권·삭제 |
| DevField-Web | 로그인과 제품 화면 제공. 브라우저 요청 경유 방식은 후속 결정 |

- 인증된 **기존 계정**에서 본인 확인과 제공자 인증을 거친 **명시적 연결**만 허용한다. 같은 이메일을 가진 별도 User ID를 자동 병합하지 않으며, 다른 User ID에 연결된 로그인 수단을 자동 이전하지 않는다.
- 마지막 사용 가능한 로그인 수단의 연결 해제를 차단한다. 소셜 계정에 이메일 로그인 수단을 추가할 때에도 이메일 확인과 본인 확인을 거쳐 기존 User ID를 유지한다.
- Kakao가 이메일을 제공하지 않는 상황을 가짜 이메일 생성으로 해결하지 않는다. 제공자 고유 ID를 기준으로 식별하고, 이메일이 필요한 절차의 본인 확인·입력 정책은 후속 설계에서 정의한다.

## 4. 회원 탈퇴와 기존 계약 보존

기존 [통합 API 계약](../contracts/repoq/api-spec.md)의 **A-04·X-03·X-04** 행과 의미를 유지한다. Auth가 A-04에서 본인을 재확인하고 계정을 `DELETING`으로 기록하며 재시도를 조정한다. Auth는 X-03으로 Project, X-04로 Interview에 해당 User ID 데이터 삭제를 요청한다. 두 서비스는 각자의 데이터만 삭제하고 신규·늦은 분석, 답변, AI 결과 등의 쓰기를 차단한다. 반복 요청은 멱등 처리하고 타 사용자 데이터를 보존한다.

Project와 Interview **양쪽의 삭제 완료 확인 전**에는 Auth가 탈퇴를 완료로 표시하거나 로그인 수단·세션 정리를 최종 완료하지 않는다. 한쪽 실패·타임아웃이면 `DELETING`을 유지하고 안전하게 재시도한다. 사용자 JWT의 서명 검증만으로 즉시 폐기 또는 계정 상태 전파가 해결되는 것은 아니다. `DELETING` 사용자에 대한 신규 접근과 늦은 쓰기를 막을 상태 확인·전파 및 토큰 무효화 계약은 후속 설계와 통합 검증이 필요하다. 프로젝트 하나를 삭제하는 P-06/X-02 흐름은 Project가 조정하는 기존 계약을 유지한다.

## 5. 선택의 결과와 트레이드오프

독립 Auth는 공통 계정 정책과 User ID의 진실 원본을 제품 밖에 명시한다. Project/Interview는 공개키로 사용자 JWT를 검증할 수 있어 매 사용자 요청마다 Auth에 서명 검증을 위임할 필요가 없다. 그 대신 별도 서비스와 Repository의 빌드·배포·관측, 키 배포와 교체, 토큰·상태의 시간차, 탈퇴 시 분산 삭제·재시도 비용이 생긴다. JWKS 조회 실패·키 교체 중 불일치·오래된 유효 JWT·계정 상태 변경 지연을 구현과 검증에서 다뤄야 한다. 구체적 캐시, 재조회, 실패 처리와 즉시 폐기 수단은 아직 정하지 않는다.

## 6. PoC 보고와 Production 경계

**이전 보고 기준이며 이번 작업에서 원시 로그를 재검증하거나 테스트를 재실행하지 않았다.** 이전 보고에는 HTTP **41 PASS / 0 FAIL**, Java 정책 **2 PASS / 0 FAIL**이 적혀 있다. 보고된 범위는 이메일 흐름·BCrypt·RS256/JWKS, 위조·만료 JWT 거부, 로그아웃 및 `DELETING` 접근 차단, Google/GitHub/Kakao 흐름·SocialIdentity·세션 DB 기록 확인이다.

다음은 **미검증** 상태로 유지한다.

- 실제 동일 이메일 자동 병합 방지 E2E
- 실제 소셜 계정 연결·해제 E2E
- Kakao 이메일 미제공 E2E
- 제공자별 콜백부터 JWT까지 추적
- Project/Interview 실제 탈퇴 데이터 삭제 통합 검증
- Production HTTPS·Cookie·BFF·키 운영·남용 방어·전체 E2E

PoC 통과는 Production 인증 또는 RepoQ MVP 완료를 의미하지 않는다. PoC의 임시 키·토큰 수명·세션 설정을 운영 정책으로 확정하지 않는다.

## 7. 아직 확정하지 않은 계약

- BFF 도입 여부, 브라우저의 서비스 호출 경유 방식
- Production Cookie·domain·CORS·CSRF 상세 구성
- JWT·세션 수명, refresh 및 즉시 폐기·상태 전파 계약
- 키 저장·교체·JWKS 게시·캐시·장애 운영
- 서비스 간 인증과 사용자 위임 방식
- 인증 상세 Endpoint와 실제 `iss`·`aud` 값
- 물리 DB·스키마·마이그레이션·호스팅
- RepoQ의 나머지 기술 스택

## 8. 구현 수용 기준

- [ ] Auth가 독립 서비스로 공통 User ID, 이메일·소셜 로그인 수단, 인증 상태를 소유하고 Project/Interview는 다른 서비스 DB에 직접 접근하지 않는다.
- [ ] 이메일·비밀번호 및 Google·GitHub·Kakao 로그인이 동작하며 동일 이메일 자동 병합, 타 User ID의 로그인 수단 자동 이전, 마지막 로그인 수단 해제가 차단된다. Kakao 이메일 미제공 흐름을 검증한다.
- [ ] Auth가 RS256 JWT를 발급하고 Project/Interview가 공개키·JWKS로 서명·허용 알고리즘·`iss`·`aud`·유효기간을 검증하여 `sub`를 DevField User ID로 사용한다.
- [ ] Project/Interview가 각 리소스 소유권을 따로 검증하고 내부 서비스 호출 권한을 사용자 JWT와 별도 계약으로 검증한다.
- [ ] A-04·X-03·X-04에 따라 Auth가 `DELETING`·재시도를 조정하고 양쪽 삭제 완료 전 탈퇴 완료를 표시하지 않는다. 신규·늦은 쓰기와 타 사용자 오삭제를 막는다.
- [ ] 토큰 폐기·상태 전파, 실제 제공자 콜백, 키 운영과 Production HTTPS·브라우저 경계에 대한 계약 및 통합 E2E를 검증한다.

위 체크리스트는 **향후 구현 수용 기준**이며 현재 통과로 표시하지 않는다.

### 공통 계정 관리 보안 결정 보완 (2026-10-28)

A-02/A-03은 활성 세션만으로 실행하지 않는다. 이메일 수단은 현재 비밀번호를 검증하고, 소셜 전용 계정은 이미 연결된 Provider의 별도 OAuth 승인과 같은 고유 ID를 확인한다. 재확인 증명은 User ID와 AuthSession ID에 결합하며 최대 5분 동안 변경 작업에 한 번만 사용한다. 로그인·연결·재확인 의도는 구분한다.

GitHub OAuth 재승인은 비밀번호 재입력을 보장하지 않는다. GitHub 소셜 전용 계정의 일반 계정 관리 허용은 사용자 최종 승인 전 보안 절충안이다. 강한 보증이 필요한 운영 기능과 유료화 이전에는 재검토하며 Production 보증 완료로 표시하지 않는다.

### LOGIN/SIGNUP 의도 분리 보완 (2026-10-29)

OAuth 시작·state·handoff 목적을 LOGIN, SIGNUP, LINK, REAUTH로 구분한다. 제공자가 검증한 고유 ID로만 가입 여부를 판정한다. LOGIN은 기존 회원에게만 새 세션을 발급하며 미가입 계정은 회원가입을 안내한다. SIGNUP은 미가입 고유 ID에만 새 User와 세션을 생성하며 기존 회원은 자동 로그인 없이 로그인 화면으로 안내한다. LINK와 REAUTH는 시작 User ID·AuthSession에 묶이고 기존 세션을 유지한다. Google REAUTH의 기본 모드는 OIDC `claims`로 `auth_time`을 요구하며, 응답 ID Token에서 실제 인증 시각을 확인한다. 실제 Provider E2E가 수행되기 전 Google 400 해결을 운영 검증 완료로 표시하지 않는다.

2026-10-29 사용자 승인: **로컬 MVP에 한해** `AUTH_GOOGLE_REAUTH_IDENTITY_ONLY_ENABLED=true`를 설정하면 Google REAUTH는 `auth_time` 요구 없이 새 OAuth 결과의 검증된 Google `sub`와 기존 연결 ID를 비교한다. 설정 기본값은 `false`이며 클라이언트가 변경할 수 없다. 이 절충안은 동일 Google 계정 접근 확인이며 비밀번호 재입력·MFA·최근 강한 인증을 보증하지 않는다. GitHub·Kakao 정책과 LINK/REAUTH 목적, 시작 User ID·AuthSession ID, state·nonce·브라우저 결합, 5분 일회용 증명은 유지한다. 유료화 및 Production의 민감한 계정 변경 전 별도 강한 재확인 수단과 보안 검토가 필요하다. 실제 Provider E2E를 운영 검증 완료로 표시하지 않는다.
## 이메일 수단과 Provider 계정 선택 후속 결정 (2026-10-29)

기존 User ID와 Auth 소유 EmailCredential·ActionToken·AuthSession을 재사용한다. 소셜 계정에 이메일을 추가할 때 최근 본인 확인 증명을 소모해 예약을 시작하고, 요청 세션에 묶인 메일 링크로 주소 소유권을 확인한 뒤 비밀번호를 최초 설정한다. 다른 User ID의 이메일 또는 Provider 고유 ID를 자동 병합·이전하지 않는다. 비밀번호 변경은 현재 세션만 유지하고, 분실 재설정은 모든 세션을 폐기한다. 마지막으로 사용 가능한 로그인 수단 해제는 계속 거부한다.

LOGIN·SIGNUP·LINK·REAUTH는 별개 OAuth 의도로 유지한다. Google과 GitHub에는 공식 `prompt=select_account`를 요청하고 Kakao에는 로그인 화면 재진입을 위한 `prompt=login`을 요청한다. Provider 화면·브라우저 세션에 따른 실제 선택 기회는 Provider가 결정하며, 계정 선택 자체는 강한 본인 재인증의 근거가 아니다. Google의 엄격한 REAUTH는 기존 `auth_time` 검증 정책을 유지한다.
