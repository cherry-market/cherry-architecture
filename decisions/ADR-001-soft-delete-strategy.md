# ADR-001: 상품 삭제 전략

- **상태**: 확정
- **일자**: 2026-02-09
- **참여**: Claude (코드베이스 분석) + ChatGPT (리서치/사례 조사)

---

## 맥락 (Context)

Phase 5.5에서 상품 삭제 기능을 구현하면서, 삭제 방식의 전략을 결정해야 했다.

현재 구조:
- `Product` 테이블에 `status` 컬럼이 MySQL ENUM 타입으로 존재 (PENDING, SELLING, RESERVED, SOLD)
- Product에 연관된 테이블: ProductImage, ProductTag, ProductLike(찜)
- 상품 목록 조회 시 Redis 캐시 사용 중 (Cache-Aside, TTL 5분)

---

## 선택지 (Options)

| 선택지 | 장점 | 단점 |
|--------|------|------|
| **1. Soft Delete (`deleted_at` 컬럼)** | status와 분리되어 명확, 복구 가능, 운영/디버깅 용이 | 모든 조회에 조건 추가 필요 |
| **2. Hard Delete (실제 삭제)** | 구현이 단순해 보임 | FK 제약/삭제 순서 처리, S3 정리, 복구 불가, 캐시 유령 상품 |
| **3. 상태 기반 (DELETED ENUM 추가)** | 기존 status 흐름에 통합 | 거래 상태와 존재/노출 상태가 혼재, ALTER TABLE 필요, 상태 전이 복잡화 |

### 비용 비교

| 선택지 | 개발 공수 | 유지보수 복잡도 | DB 변경 비용 |
|--------|-----------|-----------------|-------------|
| Soft Delete | 낮음 (컬럼 추가 + 조회 조건) | 낮음 | ALTER TABLE ADD COLUMN (가벼움) |
| Hard Delete | 중간 (FK cascade, S3 정리) | 높음 (복구 불가) | 없음 |
| 상태 기반 | 중간 | 높음 (상태 전이 복잡) | ALTER TABLE MODIFY ENUM (무거움) |

---

## 결정 (Decision)

**Soft Delete + `deleted_at` DATETIME NULL 컬럼**을 채택한다.

- `deleted_at`이 NULL이면 미삭제, 값이 있으면 삭제된 상태
- `boolean deleted` 대신 `DATETIME deleted_at`을 사용

### `deleted_at`이 `boolean`보다 나은 이유

1. "언제 삭제했는지" 추적 가능 — 운영/디버깅 편의
2. 하나의 컬럼으로 두 가지 정보 (삭제 여부 + 삭제 시점)
3. 배치 정리 기준으로 바로 사용 가능 (`WHERE deleted_at < NOW() - INTERVAL 30 DAY`)

---

## 트레이드오프 (Trade-offs)

### 감수하는 것
- 모든 조회 쿼리에 `WHERE deleted_at IS NULL` 조건이 추가됨
  - 완화 방안: JPA `@Where` 어노테이션 또는 공통 Repository 레벨에서 강제
- 삭제된 데이터가 DB에 남아 스토리지를 차지함
  - 완화 방안: 30일 후 배치로 물리 삭제 (ADR-004 참고)

### 얻는 것
- status 흐름(PENDING/SELLING/RESERVED/SOLD)의 의미가 순수하게 유지됨
- 실수 삭제 시 복구 가능 (30일 이내)
- 분쟁/CS 대응을 위한 데이터 보존 (실무 관행과 일치)
- 기존 BaseTimeEntity의 `createdAt`/`updatedAt` 패턴과 자연스럽게 어울림

---

## 참고 (References)

- 당근마켓: 개인정보 처리방침에서 거래기록(판매 게시물 등)을 일정 기간 보유하는 내용을 명시
- 중고거래/마켓플레이스는 사용자에겐 삭제처럼 보여도, 내부적으로는 일정 기간 보관하는 것이 업계 관행
- ChatGPT 리서치 (2026-02-09)
