# ADR-009: 상품 검색 전략

- **상태**: 확정
- **일자**: 2026-02-28

---

## 맥락 (Context)

Phase 7에서 상품 키워드 검색 기능을 구현한다. 기존 `GET /products` API는 필터와 정렬만 지원하며, 키워드 검색은 미구현 상태였다.

- DB: MySQL 8.x (RDS)
- 검색 대상: 상품 제목(`title`), 태그(`tags`)
- 검색어 특성: 대부분 한국어 (아이돌 이름, 굿즈명)
- 상품 규모: MVP (수천~수만 건)

---

## 주요 의사결정

### 1. 검색 엔진

| 선택지 | 장점 | 단점 |
|--------|------|------|
| LIKE '%keyword%' | 구현 단순 | 인덱스 미사용, 풀스캔 |
| FULLTEXT + 기본 파서 | MySQL 내장 전문 검색 | 한국어 지원 제한 (공백 기준 토큰화) |
| **FULLTEXT + ngram parser** | **한국어 부분 매칭 지원** | **ngram_token_size 설정 필요** |
| OpenSearch | 형태소 분석, 자동완성 | 추가 인프라 비용, MVP에 과도 |

**결정: FULLTEXT + ngram parser** — 추가 인프라 $0, 한국어 2글자 부분 매칭 지원.

### 2. API 설계

| 선택지 | 장점 | 단점 |
|--------|------|------|
| **기존 GET /products에 q 추가** | **필터/정렬 통합, 프론트 변경 최소** | **검색 로직 혼합** |
| GET /products/search 분리 | 로직 격리 | 엔드포인트 분리, 프론트 분기 필요 |

**결정: 기존 API에 q 파라미터 추가** — 내부적으로 검색 로직을 분리하여 추후 OpenSearch 전환 대비.

### 3. 정렬 전략

| 선택지 | 장점 | 단점 |
|--------|------|------|
| 기존 sortBy 유지 | 변경 없음 | 검색 relevance 무시 |
| **RELEVANCE 기본 + sortBy 전환** | **검색 UX 자연스러움** | **페이징 복잡도 증가** |
| RELEVANCE 고정 | 구현 단순 | 가격순 전환 불가 |

**결정: RELEVANCE 기본 + sortBy 전환 가능** — 검색 시 관련도순 기본, 사용자가 가격순 등으로 전환 가능.

### 4. 페이징 전략

| 정렬 | 페이징 방식 | 근거 |
|------|-----------|------|
| RELEVANCE | offset 기반 (page, limit) | FULLTEXT score는 동적 값이라 커서 불가 |
| LATEST / 가격순 | 기존 커서 기반 유지 | 안정적 커서 값 존재 |

Elasticsearch도 기본은 from+size(offset)이며, 10,000건 초과 시에만 search_after로 전환한다. MVP 규모에서 offset 성능 문제 없음.

### 5. 캐싱 전략

**결정: 캐싱하지 않음** — 검색어 조합이 다양하여 히트율 낮고, 상품 변경 시 무효화 복잡. FULLTEXT 인덱스 사용 시 조회 비용은 허용 범위.

---

## 트레이드오프

### 감수하는 것
- ngram FULLTEXT 인덱스의 추가 저장 공간 → MVP 규모에서 무시 가능
- RELEVANCE 정렬 시 offset 방식의 deep pagination 성능 저하 → 검색 결과 수십~수백 건이라 발생 확률 극히 낮음
- 태그 검색 시 JOIN 쿼리 복잡도 → MVP 규모에서 무시 가능

### 얻는 것
- 추가 인프라 비용 $0 (MySQL 단독)
- 한국어 부분 매칭 (ngram)
- 기존 API/프론트 구조 최소 변경으로 통합
- OpenSearch 전환 경로 확보

---

## 전환 기준 (Scaling Trigger)

아래 조건에 해당할 때 OpenSearch 도입을 재검토한다:
- 상품 수 10만 건 이상으로 FULLTEXT 성능 저하 체감 시
- 한국어 형태소 분석(nori plugin)이 필요할 때
- 자동완성, 유사 상품 추천 기능 요구 시

전환 경로: MySQL FULLTEXT → OpenSearch (API 변경 없이 내부 구현만 교체)
