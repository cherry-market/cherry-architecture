# Cherry Architecture

> K-POP 팬덤 굿즈 C2C 플랫폼의 시스템 아키텍처 및 설계 결정 기록

[![Service](https://img.shields.io/badge/Service-cheryi.com-FF2E88?style=for-the-badge&logo=safari&logoColor=white)](https://cheryi.com)

> [!IMPORTANT]
> **현재 공개 배포는 중단된 상태입니다** (AWS 무료 계정 종료). 문서와 코드는 그대로 유지되며, 로컬 실행은 가능합니다.
> 서비스 규모에 맞춰 인프라 비용을 낮추는 저비용 구조로의 전환은 설계 단계입니다(미구현). → [저비용 인프라 전환 설계](architecture/low-cost-migration.md)

---

## Tech Stack

| Layer | Stack |
|-------|-------|
| **Frontend** | React 19, TypeScript, Vite, Zustand, Tailwind CSS |
| **Backend** | Spring Boot 3, Java 21, JPA/Hibernate, Spring Security (JWT) |
| **Real-time** | STOMP over WebSocket, SimpleBroker |
| **Database** | MySQL 8 (RDS), Redis (Cache + Ranking + Rate Limit) |
| **Infra** | AWS EC2, S3, Lambda, CloudFront, ALB, Docker Compose |
| **AI** | Google Gemini (gemini-2.0-flash) — 상품 설명 자동 생성 |

---

## Documents

### Architecture

시스템 구성, 배포 구조, 데이터 흐름을 다룹니다.

- [System Overview](architecture/system-overview.md) — 전체 인프라 구성 (Mermaid)
- [저비용 인프라 전환 설계](architecture/low-cost-migration.md) — 기존 AWS 구조와 저비용 전환 후보 구조 비교 (설계 단계, 미구현)
- [ERD](architecture/erd.md) — 데이터 모델 설계
- [Use Cases](architecture/use-cases.md) — 핵심 유스케이스 다이어그램
- [CI/CD + Runtime](architecture/cicd.md) — 배포 파이프라인 및 런타임 구조
- [Auth Domain](architecture/auth-domain.md) — 인증/회원가입 도메인 (JWT, Spring Security, 토큰 흐름)
- [Image Pipeline Domain](architecture/image-pipeline-domain.md) — 이미지 업로드 파이프라인 (Presigned URL, S3, Lambda)
- [Product Domain](architecture/product-domain.md) — 상품 도메인 (CRUD, 캐싱, 트렌딩, 찜, AI 설명 생성)
- [Chat Domain](architecture/chat-domain.md) — 채팅 도메인 아키텍처 (메시지 흐름, 보안, 확장 포인트)

### Architecture Decision Records

설계 과정에서 내린 주요 의사결정과 그 근거를 기록합니다.

- [ADR-001: Soft Delete 전략](decisions/ADR-001-soft-delete-strategy.md)
- [ADR-002: 이미지 수정 범위](decisions/ADR-002-image-edit-scope.md)
- [ADR-003: 상태 전환 규칙](decisions/ADR-003-status-transition-rules.md)
- [ADR-004: S3 이미지 정리](decisions/ADR-004-s3-image-cleanup.md)
- [ADR-005: 채팅 메시지 저장 전략](decisions/ADR-005-chat-message-storage.md)
- [ADR-006: 채팅 목록 실시간 갱신](decisions/ADR-006-chat-list-realtime-update.md)
- [ADR-007: 실패 메시지 영구 저장소](decisions/ADR-007-failed-message-storage.md)
- [ADR-008: 채팅 이미지 첨부 전략](decisions/ADR-008-chat-image-strategy.md)
- [ADR-009: 상품 검색 전략](decisions/ADR-009-search-strategy.md)
- [ADR-010: 인증 사용자 열거 방지 및 타이밍 사이드채널 차단](decisions/ADR-010-auth-user-enumeration-hardening.md)

### Performance

Redis Cache-Aside 패턴 적용 전후 성능 비교 분석입니다.

- [Caching Optimization Report](performance/caching-optimization.md) — wrk 벤치마크 기반 Before/After

### Engineering

- [AI-Augmented Development](engineering/ai-augmented-development.md) — AI 에이전트 협업 방식과 활용 기록
- [Security Review (AI Collaboration)](engineering/security-review-ai-collaboration.md) — Phase 10 인증 보안 감사 기록 (AI-인간 협업으로 찾은 사용자 열거 취약점과 수정 프로세스)

---

## Repositories

| Repository | Role |
|------------|------|
| [cherry-client](https://github.com/cherry-market/cherry-client) | Frontend (React + FSD) |
| [cherry-server](https://github.com/cherry-market/cherry-server) | Backend (Spring Boot) |
| [cherry-architecture](https://github.com/cherry-market/cherry-architecture) | Architecture Docs (this repo) |

---

<div align="center">
  <sub>Cherry Market &mdash; Human Architect + AI Collaboration</sub>
</div>
