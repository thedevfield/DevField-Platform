 <div align="center">

<img src="https://github.com/thedevfield.png" alt="DevField Logo" width="110" />

# DevField Platform

**Practice. Solve. Grow.**

개발자의 기술 이해도와 실무 문제 해결 역량을 훈련하는 **DevField**의 공통 제품 요구사항, 아키텍처 및 서비스 간 계약을 관리하는 저장소입니다.

</div>

---

## Overview

DevField는 개발자가 기술을 이해하고, 선택의 근거를 설명하며, 실제 문제를 해결하는 능력을 훈련할 수 있도록 만드는 개발자 역량 강화 플랫폼입니다.

이 저장소인 **`DevField-Platform`**은 각 제품의 소스코드를 모아 두는 곳이 아니라, 플랫폼 전반에서 공유하는 **요구사항·설계 결정·서비스 계약·통합 가이드의 기준점**입니다.

## Products

| 서비스 | 핵심 기능 | 개발 단계 |
|---|---|---|
| **RepoQ** | GitHub 프로젝트 기반 AI 기술면접, 동적 꼬리질문 및 평가 | **1차 개발 대상** |
| **AlgoField** | 알고리즘·자료구조 기반 코딩테스트 및 풀이 훈련 | 향후 확장 |
| **InfraField** | 인프라·시스템 장애 분석과 대응 판단 훈련 | 향후 확장 |

각 제품은 독립적으로 이용할 수 있으며, 공통 계정과 일관된 사용자 경험을 공유하는 방향으로 설계합니다.

## Repository Responsibilities

`DevField-Platform`에서 관리할 영역은 다음과 같습니다.

- **Product Requirements** — 제품별 PRD, MVP 범위, 인수 기준 및 변경 이력
- **Architecture & ADR** — 플랫폼 구조, 서비스 책임·데이터 소유권, 주요 기술적 의사결정
- **Service Contracts** — 제품·서비스 간 API 계약과 호환성 기준
- **Integration Guides** — 로컬 통합 실행, 환경 구성, 서비스 연동 및 검증 가이드

실제 웹·백엔드 애플리케이션의 구현 코드와 개별 빌드·배포 설정은 해당 제품 또는 애플리케이션 저장소에서 관리합니다.

## Documentation

| 문서 | 설명 | 상태 |
|---|---|---|
| [RepoQ MVP PRD v1.1 개정 초안](docs/products/repoq/PRD.md) | 승인된 v1.0 기준선에 이메일 가입·회원 탈퇴 연동을 반영한 제품 요구사항 | **개정 초안 · 구현 전** |
| [RepoQ 서비스 경계 설계서](docs/architecture/repoq-service-boundaries.md) | Project/Interview 책임과 데이터 소유권 | **상위 설계 승인** |
| [ADR-001](docs/adr/0001-repoq-service-decomposition.md) | RepoQ 서비스 분리 결정 | **Accepted** |
| [RepoQ 통합 API 명세서 v0.2](docs/contracts/repoq/api-spec.md) | 사용자용·서비스 간 계약 및 회원 탈퇴 연동 | **통합 검토 초안** |

현재 저장소에 존재하는 문서 구조는 다음과 같습니다.

```text
DevField-Platform/
├── README.md
└── docs/
    ├── products/
    │   └── repoq/
    │       └── PRD.md
    ├── architecture/
    │   └── repoq-service-boundaries.md
    ├── adr/
    │   └── 0001-repoq-service-decomposition.md
    └── contracts/
        └── repoq/
            └── api-spec.md
```

추가 디렉터리는 실제 설계와 문서가 필요해지는 시점에 구성합니다.

## Architecture Principles

DevField는 **MSA(Microservices Architecture)**와 **Multi-Repository** 운영을 지향합니다.

1. **Business Boundaries** — 비즈니스 책임과 데이터 소유권에 따라 서비스 경계를 정의합니다.
2. **Independent Delivery** — 서비스별 독립 빌드·테스트·배포를 목표로 합니다.
3. **Explicit Contracts** — 서비스 간 연동은 명시된 API·이벤트 계약을 사용하며, 다른 서비스의 DB에 직접 접근하지 않습니다.
4. **Pragmatic Complexity** — 초기 MVP에 불필요한 인프라나 마이크로서비스를 도입하지 않습니다.
5. **Traceability & Quality** — 요구사항, 설계 결정, 테스트 및 변경 이유를 추적할 수 있도록 관리합니다.

제품별 저장소 안에 여러 독립 서비스가 공존할 수 있습니다. RepoQ의 **Project Service와 Interview Service 분리**는 ADR-001에서 승인했습니다. 인증 제품·기술 스택 및 인프라 제품은 아직 확정하지 않았습니다.

## Planned Repository Organization

GitHub Organization: [thedevfield](https://github.com/thedevfield)

| Repository | 책임 | 상태 |
|---|---|---|
| [`.github`](https://github.com/thedevfield/.github) | Organization 프로필 | 생성됨 |
| [`DevField-Platform`](https://github.com/thedevfield/DevField-Platform) | 공통 요구사항·아키텍처·계약 | **현재 저장소** |
| `DevField-Web` | 통합 웹 프론트엔드 · Vercel | 생성 예정 |
| `RepoQ` | RepoQ 애플리케이션 및 내부 서비스 | 생성 예정 |
| `AlgoField` | 코딩테스트 제품 | 향후 |
| `InfraField` | 인프라 훈련 제품 | 향후 |

## Current Status & Next Steps

**현재 상태: RepoQ MVP PRD v1.0 승인 기준선을 바탕으로 v1.1 개정 초안을 작성했고, 서비스 경계·ADR-001은 승인됐습니다. 통합 API v0.2는 검토 초안이며 구현·배포 전입니다.**

다음 단계에서는 통합 API의 세부 필드·인증 연동을 검토하고 기술 스택을 설계한 후 필요한 애플리케이션 저장소와 통합 개발 환경을 구성합니다. RepoQ 웹은 **Vercel Production 배포 및 실제 E2E 검증**을 MVP 완료 조건으로 계획하고 있습니다.

> 이 저장소는 현재 문서 중심의 플랫폼 저장소이며, 별도의 실행 가능한 애플리케이션이나 배포 결과를 포함한다고 주장하지 않습니다.

---

<div align="center">

**DevField — Practice. Solve. Grow.**

[GitHub Organization](https://github.com/thedevfield) · [RepoQ MVP Requirements](docs/products/repoq/PRD.md)

</div>
