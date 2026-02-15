# ADR-003: 상태 전환 규칙

- **상태**: 확정
- **일자**: 2026-02-09
- **참여**: Claude (코드베이스 분석) + ChatGPT (리서치/사례 조사)

---

## 맥락 (Context)

상품 수정/삭제 기능과 함께, 사용자가 상품 상태를 직접 변경할 수 있는 규칙을 정의해야 했다.

현재 상태:
- `ProductStatus` ENUM: PENDING, SELLING, RESERVED, SOLD
- PENDING → SELLING은 Lambda 콜백에 의해 자동 전환
- 사용자가 수동으로 변경할 수 있는 전환 규칙이 없었음
- 삭제는 Soft Delete(`deleted_at`)로 status와 분리 확정 (ADR-001)

---

## 선택지 (Options)

### 1. 상태 전환 허용 범위

| 전환 | 허용 여부 | 사유 |
|------|-----------|------|
| SELLING → RESERVED | 허용 | 예약 처리 (기본) |
| RESERVED → SELLING | 허용 | 예약 취소/거래 무산 |
| SELLING → SOLD | 허용 | 판매 완료 |
| RESERVED → SOLD | 허용 | 예약 후 거래 완료 |
| SOLD → SELLING | **논쟁점** | 재판매(거래 무산 복구) |

**SOLD → SELLING 허용 여부:**

| 선택지 | 장점 | 단점 |
|--------|------|------|
| 허용 | 거래 무산 시 복구 가능, 당근마켓 관행 일치 | 상태 전이 방향이 양방향 |
| 금지 | 상태 흐름이 단방향으로 단순 | 거래 무산 시 삭제 후 재등록 필요 |

### 2. 상태 전환 API 설계

| 선택지 | 설명 | 비용 |
|--------|------|------|
| **수정 API에 status 포함** | 상품 수정 API에 status 필드 포함 | 검증 로직 혼재, 유지보수 복잡 |
| **전용 API 분리** | 상태 전환 전용 API | 관심사 분리, 검증 독립 |

### 3. 상태별 수정/삭제 허용

| 선택지 | PENDING 텍스트 수정 | SOLD 텍스트 수정 |
|--------|---------------------|-----------------|
| 관대 정책 | 허용 | 허용 |
| **엄격 정책 (권장)** | 금지 | 금지 (재판매 후 수정) |

비용 비교:
- 관대 정책: PENDING에서 수정 시 Lambda 콜백과의 동시성 고려 필요 (개발 비용 증가)
- 엄격 정책: 조건 분기가 단순, PENDING은 수 초 이내 자동 전환이라 UX 손해 미미

---

## 결정 (Decision)

### 허용되는 상태 전환 (화이트리스트)

```
SELLING  → RESERVED  ✅
RESERVED → SELLING   ✅
SELLING  → SOLD      ✅
RESERVED → SOLD      ✅
SOLD     → SELLING   ✅ (재판매/거래 무산 복구)
```

구현: `Map<ProductStatus, Set<ProductStatus>>` 화이트리스트

### 상태별 수정/삭제 매트릭스

| 상태 | 텍스트 수정 | 이미지 수정 | 삭제(soft) |
|------|:-----------:|:-----------:|:----------:|
| PENDING | 금지 | 금지 | 허용 |
| SELLING | 허용 | 허용 | 허용 |
| RESERVED | 허용 | 허용 | 허용 |
| SOLD | 금지 | 금지 | 허용 |

SOLD 상태에서 수정이 필요하면 → SOLD → SELLING 전환 후 수정

### API 설계

```
상태 전환 전용 API   ← PATCH, 화이트리스트 검증
상품 수정 API        ← PUT, status 필드 미포함
상품 삭제 API        ← DELETE, soft delete (deleted_at)
```

### 동시성 처리

```sql
UPDATE product SET status = ? WHERE id = ? AND status = ? AND deleted_at IS NULL
```

낙관적 전이 방식으로 영향 범위 최소화

---

## 트레이드오프 (Trade-offs)

### 감수하는 것
- SOLD → SELLING 허용으로 상태 흐름이 단방향이 아님
  - 완화: 화이트리스트로 허용 전환만 명시적으로 제한
- PENDING/SOLD에서 텍스트 수정 불가 → 사용자가 불편할 수 있음
  - 완화: PENDING은 수 초 이내 자동 해소, SOLD는 재판매 전환 후 수정 가능

### 얻는 것
- "내용 수정"과 "상태 전환"의 관심사 분리 → 검증 로직 독립, 유지보수 용이
- 당근마켓 관행과 일치하는 UX (재판매 허용)
- PENDING 상태에서의 동시성 문제 원천 차단
- 상태머신을 분리해 안전하게 운영하는 설계 구조

---

## 참고 (References)

- 당근마켓 고객센터: 예약중/거래완료를 판매중으로 되돌리는 동작 안내 (https://cs.kr.karrotmarket.com/wv/faqs/87)
- SOLD → SELLING의 실제 사용 사례: 거래 무산(노쇼, 미입금), 실수로 완료 처리, 반품 후 재등록
- ChatGPT 리서치 (2026-02-09)
