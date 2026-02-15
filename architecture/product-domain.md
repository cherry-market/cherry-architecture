# 상품 도메인 아키텍처

- **Phase**: 2~5
- **상태**: 완료

---

## 1. 개요

Cherry 플랫폼의 핵심 도메인. 상품 등록/수정/삭제, 리스트 조회, 캐싱, 트렌딩, 찜, AI 설명 생성을 포함한다.

**핵심 기능:**
- 상품 CRUD (생성/수정/삭제)
- 동적 필터링 + 커서 기반 페이지네이션
- Redis 캐싱 (Cache-Aside 패턴)
- 트렌딩 시스템 (조회수 기반 랭킹)
- 찜(위시) 기능
- AI 상품 설명 생성 (Gemini)
- 상품 상태 관리 (PENDING → SELLING → RESERVED → SOLD)

---

## 2. 기술 스택

| 구성 요소 | 기술 | 역할 |
|-----------|------|------|
| 동적 쿼리 | JPA Criteria API | 필터 조합 처리 (status, category, price, tradeType) |
| 캐싱 | Redis Cache-Aside | 리스트 조회 성능 최적화 (TTL 5분) |
| 트렌딩 | Redis Sorted Set | 조회수 집계 + 상위 상품 추출 |
| 찜 | 복합 유니크 인덱스 | 중복 방지 (user_id + product_id) |
| AI | Google Gemini API | 상품 설명 자동 생성 |
| 삭제 | Soft Delete (deleted_at) | 데이터 보존 (30일 유예) |
| 상태 관리 | 화이트리스트 맵 | 유효하지 않은 상태 전환 방지 |

---

## 3. 데이터 모델

```
products
├── id (PK, BIGINT AUTO_INCREMENT)
├── seller_user_id (FK → users)
├── title (VARCHAR, NOT NULL)
├── description (TEXT)
├── price (INT)
├── status (ENUM: PENDING, SELLING, RESERVED, SOLD)
├── trade_type (ENUM: DIRECT, DELIVERY, BOTH)
├── category_id (FK → categories)
├── deleted_at (DATETIME, nullable) — Soft Delete
├── created_at / updated_at (DATETIME)

categories
├── id (PK)
├── code (VARCHAR, UNIQUE) — 필터링 키
├── display_name (VARCHAR) — 화면 표시명
├── is_active (BOOLEAN)
├── sort_order (INT)
├── created_at / updated_at

tags
├── id (PK)
├── name (VARCHAR, UNIQUE) — 자동 생성
├── created_at / updated_at

product_tags (N:M)
├── product_id (FK → products)
├── tag_id (FK → tags)

product_likes
├── id (PK)
├── user_id (FK → users)
├── product_id (FK → products)
├── created_at / updated_at
└── UNIQUE(user_id, product_id)

product_images
├── id (PK)
├── product_id (FK → products)
├── original_url (VARCHAR) — S3 원본
├── image_url (VARCHAR) — Lambda 리사이즈 결과
├── thumbnail_url (VARCHAR) — Lambda 썸네일
├── image_order (INT) — 정렬 순서
├── is_thumbnail (BOOLEAN) — 대표 이미지 여부
└── created_at / updated_at
```

---

## 4. 상품 조회 + 캐싱

### 4.1 리스트 조회 흐름

```mermaid
sequenceDiagram
    participant C as Client
    participant API as ProductController
    participant SVC as ProductService
    participant Redis as Redis Cache
    participant DB as MySQL
    participant Repo as ProductRepository

    C->>API: GET /products?cursor&status&categoryCode&minPrice&maxPrice&tradeType&sortBy&limit
    API->>SVC: getProducts(cursor, limit, userId, condition, sortBy)
    SVC->>SVC: buildCacheKey(cursor + filters + sortBy + limit)
    SVC->>Redis: GET 리스트 캐시 키
    alt Cache Hit
        Redis-->>SVC: ProductListResponse (isLiked=false)
        SVC->>SVC: applyLikedFlags(userId) — 실시간 찜 여부 적용
        SVC-->>API: ProductListResponse
    else Cache Miss
        SVC->>Repo: Criteria API 동적 쿼리
        Repo->>DB: SELECT (Fetch Join: seller, category)
        DB-->>Repo: Product[]
        SVC->>DB: 배치 조회 (태그, 찜 여부, 찜 개수)
        SVC->>SVC: ProductSummaryResponse[] 생성
        SVC->>Redis: SET 리스트 캐시 키, TTL 5분 (isLiked=false로 정규화)
        SVC-->>API: ProductListResponse
    end
    API-->>C: {items: [], nextCursor: "..."}
```

### 4.2 동적 필터링 (Criteria API)

**필터 파라미터:**
- `status` (SELLING, RESERVED, SOLD)
- `categoryCode` (PHOTOCARD, ALBUM, etc.)
- `minPrice` / `maxPrice`
- `tradeType` (DIRECT, DELIVERY, BOTH)
- `sortBy` (LATEST, LOW_PRICE, HIGH_PRICE)

**동적 Predicate 생성:**
- 각 필터가 null이 아닌 경우에만 Predicate 추가
- `TradeType.BOTH` 특수 처리: DIRECT/DELIVERY 요청에도 매칭
- Fetch Join (seller, category)으로 N+1 방지
- 배치 조회로 태그, 찜 여부 N+1 완전 제거

### 4.3 커서 기반 페이지네이션

**커서 형식:** `{sortValue}_{productId}`

| 정렬 기준 | 1차 정렬 | 2차 정렬 | 커서 예시 |
|-----------|----------|----------|-----------|
| LATEST | createdAt DESC | id DESC | `2026-02-15T10:30:00_123` |
| LOW_PRICE | price ASC | id DESC | `15000_456` |
| HIGH_PRICE | price DESC | id DESC | `50000_789` |

**2차 정렬(id DESC)로 정렬 안정성 보장** — 동일 가격/시간에도 일관된 순서

### 4.4 캐시 무효화

```mermaid
flowchart TD
    A[Product 변경 이벤트] --> B{이벤트 타입}
    B -->|생성| C["@PostPersist"]
    B -->|수정| D["@PostUpdate"]
    B -->|삭제| E["@PostRemove"]
    C --> F[ProductEntityListener]
    D --> F
    E --> F
    F --> G[ProductCacheInvalidator]
    G --> H[Redis 패턴 매칭으로 리스트 캐시 탐색]
    H --> I[전체 캐시 삭제]
```

**isLiked 분리 전략:**
- 캐시에는 `isLiked=false`로 저장 (사용자 무관)
- 조회 시 실시간으로 `isLiked` 적용
- 캐시 재사용률 극대화 (사용자별 캐시 불필요)

### 4.5 성능 개선 결과

**wrk 벤치마크** (동시성 50, 60초, HikariCP maxPool=10):

| 구분 | Avg Latency | Req/sec | 개선 |
|------|-------------|---------|------|
| Before (Redis OFF) | 1.71s | 24.28 | - |
| After (Redis ON) | 78.48ms | 847.36 | **RPS 34.9배**, 지연시간 95.4% 감소 |

---

## 5. 트렌딩 시스템

### 5.1 조회수 추적

```mermaid
sequenceDiagram
    participant C as Client
    participant API as ProductController
    participant SVC as ProductService
    participant Trending as TrendingRepository
    participant Redis as Redis (Sorted Set)

    C->>API: GET /products/{id}
    API->>SVC: getProduct(productId, userId, clientIp)
    SVC->>SVC: IP 추출 (X-Forwarded-For 대응)
    SVC->>Trending: tryIncrementViewCount(productId, clientIp)
    Trending->>Redis: EXISTS IP별 중복 방지 키 (TTL 체크)
    alt 신규 IP
        Trending->>Redis: SET IP별 중복 방지 키, TTL
        Trending->>Redis: ZINCRBY 트렌딩 Sorted Set {productId} 1
    else 중복 IP
        Note over Trending: 무시 (조회수 미증가)
    end
    SVC-->>API: ProductDetailResponse
```

**중복 방지 로직:**
- IP별 중복 방지 키 사용 (TTL 설정)
- 동일 IP에서 재조회 시 조회수 미증가

### 5.2 트렌딩 조회

```mermaid
sequenceDiagram
    participant C as Client
    participant API as ProductController
    participant SVC as ProductService
    participant Trending as TrendingRepository
    participant Redis as Redis
    participant DB as MySQL

    C->>API: GET /products/trending
    API->>SVC: getTrending(userId)
    SVC->>Trending: getTopTrendingProductIds(10)
    Trending->>Redis: ZREVRANGE 트렌딩 Sorted Set 0 9 (상위 10개)
    Redis-->>Trending: [productId1, productId2, ...]
    Trending-->>SVC: List<Long>
    SVC->>DB: 배치 조회 (IN 쿼리, Fetch Join)
    SVC->>SVC: PENDING 상품 필터링
    SVC->>DB: 찜 정보 배치 조회
    SVC-->>API: ProductListResponse
    API-->>C: {items: [], nextCursor: null}
```

---

## 6. 찜(위시) 기능

### 6.1 찜 흐름

```mermaid
sequenceDiagram
    participant C as Client
    participant API as WishController
    participant SVC as WishService
    participant Cache as ProductCacheInvalidator
    participant Redis as Redis
    participant DB as MySQL

    Note over C,DB: 찜 추가
    C->>API: POST /products/{id}/like
    API->>SVC: addLike(userId, productId)
    SVC->>DB: 존재 여부 확인 + 저장 (유니크 제약 위반 catch)
    SVC->>Cache: invalidate(productId)
    Cache->>Redis: 리스트 캐시 + 찜 캐시 무효화
    SVC-->>API: 200 OK

    Note over C,DB: 찜 취소
    C->>API: DELETE /products/{id}/like
    API->>SVC: removeLike(userId, productId)
    SVC->>DB: DELETE WHERE user_id AND product_id
    SVC->>Cache: invalidate(productId)
    Cache->>Redis: 리스트 캐시 + 찜 캐시 무효화
    SVC-->>API: 204 No Content

    Note over C,DB: 찜 여부 조회
    C->>API: GET /products/{id}/like-status
    API->>SVC: isLiked(userId, productId)
    SVC->>DB: SELECT EXISTS
    SVC-->>API: Boolean (true/false)

    Note over C,DB: 내 찜 목록
    C->>API: GET /me/likes?cursor&limit=20
    API->>SVC: getMyLikes(userId, cursor, limit)
    SVC->>Redis: GET 찜 목록 캐시 키
    alt Cache Hit
        Redis-->>SVC: ProductListResponse
    else Cache Miss
        SVC->>DB: SELECT (커서 기반 페이지네이션)
        SVC->>Redis: SET 찜 목록 캐시 키, TTL
    end
    SVC-->>API: ProductListResponse
```

### 6.2 데이터 구조

**ProductLike 테이블:**
- `UNIQUE(user_id, product_id)` 복합 인덱스로 중복 방지
- 존재 여부 확인 + 저장 + 유니크 제약 위반 catch (방어적 패턴)
- 찜 개수: 집계 쿼리 (`countByProductIds`)
- 찜 목록: 커서 기반 페이지네이션 (`created_at DESC, id DESC`)
- 찜 목록 캐싱: Redis로 조회 성능 최적화

---

## 7. AI 상품 설명 생성

### 7.1 흐름

```mermaid
sequenceDiagram
    participant C as Client
    participant API as AiController
    participant SVC as AiService
    participant Redis as Redis
    participant Gemini as Gemini API

    C->>API: POST AI 설명 생성 API
    Note right of C: {keywords, category, personality, tone}
    API->>SVC: generate(userId, request)
    SVC->>Redis: 일일 제한 카운터 체크
    alt 횟수 초과
        SVC-->>API: 429 Too Many Requests
    end
    SVC->>Redis: 쿨다운 키 체크
    alt 쿨다운 중
        SVC-->>API: 429 Too Many Requests
    end

    alt API 키 설정됨
        SVC->>SVC: buildPrompt(키워드/카테고리/성격/말투)
        SVC->>Gemini: Gemini API 호출
        Gemini-->>SVC: {candidates: [{content: {parts: [{text: "..."}]}}]}
        SVC->>SVC: JSON 파싱 → text 추출
    else API 키 미설정
        SVC->>SVC: callFallback() — 기본 메시지
    end

    SVC->>Redis: 일일 제한 카운터 +1, 쿨다운 키 설정
    SVC-->>API: {generatedDescription: "...", remainingCount: 4}
    API-->>C: 200 OK
```

### 7.2 설계 포인트

**Rate Limiting:**
- 일일 제한 카운터 (TTL 24시간)
- 쿨다운 키 (TTL 설정)
- 요청 간 쿨다운 적용 (연속 요청 방지)

**Fallback:**
- API 키 미설정 시 기본 메시지 반환
- Gemini API 오류 시 fallback 동작

**프롬프트 엔지니어링:**
- 카테고리별 맞춤 톤 (`personality`, `tone`)
- K-POP 팬 커뮤니티 용어 활용 (양도, 쿨거, 컨디션, 택포)
- 실제 판매자가 쓴 것처럼 자연스럽게 (AI 티 방지)

**모델 설정:**
- Model: Gemini API (flash 계열)
- Temperature: 0.8 (창의성)
- Max Tokens: 500

---

## 8. 상품 상태 관리

### 8.1 상태 전환

```mermaid
stateDiagram-v2
    [*] --> PENDING: 상품 등록 (이미지 처리 중)
    PENDING --> SELLING: Lambda 리사이즈 완료

    SELLING --> RESERVED: 예약 처리
    RESERVED --> SELLING: 예약 취소

    SELLING --> SOLD: 판매 완료
    RESERVED --> SOLD: 예약 후 거래 완료

    SOLD --> SELLING: 재판매 (거래 무산 복구)

    note right of PENDING
        이미지 없이 등록 시 즉시 SELLING
        이미지 있으면 Lambda 처리 대기
        PENDING 상품은 판매자 본인만 조회 가능 (타인 404)
    end note

    note right of SOLD
        SOLD → SELLING 허용 (당근마켓 관행)
        거래 무산, 노쇼, 반품 후 재판매
    end note
```

**허용되는 상태 전환 (화이트리스트):**
```
SELLING  → RESERVED  ✅
RESERVED → SELLING   ✅
SELLING  → SOLD      ✅
RESERVED → SOLD      ✅
SOLD     → SELLING   ✅
```

**상태별 수정/삭제 매트릭스:**

| 상태 | 텍스트 수정 | 이미지 수정 | 삭제(soft) |
|------|:-----------:|:-----------:|:----------:|
| PENDING | 금지 | 금지 | 허용 |
| SELLING | 허용 | 허용 (신규 이미지 추가 시 PENDING 전환) | 허용 |
| RESERVED | 허용 | 허용 (신규 이미지 추가 시 PENDING 전환) | 허용 |
| SOLD | 금지 | 금지 | 허용 |

SOLD 상태에서 수정 필요 시 → SOLD → SELLING 전환 후 수정

### 8.2 Soft Delete

```mermaid
flowchart TD
    A[사용자 삭제 요청] --> B[deleted_at = NOW]
    B --> C[전체 쿼리에 deleted_at IS NULL 조건]
    C --> D[목록/상세에서 숨김]
    D --> E[30일 유예 기간]
    E --> F{Spring @Scheduled 배치}
    F --> G[WHERE deleted_at < NOW - INTERVAL 30 DAY]
    G --> H[물리 삭제 (DELETE)]
```

**정책:**
- `deleted_at` DATETIME(0) 컬럼 (NULL = 미삭제, 값 = 삭제 시점)
- 모든 조회 쿼리에 `deleted_at IS NULL` 조건
- 30일 유예 후 물리 삭제 (Spring @Scheduled로 구현됨)
- `boolean deleted` 대신 `DATETIME deleted_at` 사용 이유:
  - 삭제 시점 추적 가능 (운영/디버깅)
  - 하나의 컬럼으로 두 정보 (삭제 여부 + 시점)
  - 배치 정리 기준으로 바로 사용 가능

---

## 9. 설계 결정 요약

| 결정 | 선택 | 근거 | ADR |
|------|------|------|-----|
| 삭제 전략 | Soft Delete (deleted_at) | 데이터 보존, 복구 가능, 운영 편의 | [ADR-001](../decisions/ADR-001-soft-delete-strategy.md) |
| 상태 전환 | 화이트리스트 맵 | 유효하지 않은 전환 방지, 명시적 규칙 | [ADR-003](../decisions/ADR-003-status-transition-rules.md) |
| 캐싱 | Cache-Aside + TTL + Write Invalidation | 성능 vs 일관성 균형, 추가 인프라 불필요 | - |
| 동적 쿼리 | JPA Criteria API | 의존성 최소화, QueryDSL 불필요 | - |
| 커서 페이지네이션 | {sortValue}_{id} | 대용량 안정적 페이징, offset 방식 문제 회피 | - |
| 전체 캐시 무효화 | KEYS 패턴 삭제 | 단순성 우선 (필터 조합 다양, 선택적 무효화 복잡) | - |
| isLiked 분리 | 캐시=false, 조회 시 실시간 적용 | 사용자별 캐시 불필요, 재사용률 극대화 | - |
| SOLD → SELLING | 허용 | 당근마켓 관행, 거래 무산 시 복구 | [ADR-003](../decisions/ADR-003-status-transition-rules.md) |

---

## 10. 확장 포인트

| 항목 | 현재 | 확장 시 |
|------|------|---------|
| 검색 | 미구현 | Elasticsearch/OpenSearch (전문 검색) |
| 캐시 무효화 | 전체 삭제 | 선택적 무효화 (캐시 태그, 필터별 TTL) |
| 동적 쿼리 | Criteria API | QueryDSL (타입 안정성, 가독성) |
| AI | Gemini 텍스트 생성 | 이미지 분석 기반 자동 태깅 |
| 트렌딩 | 24시간 단순 조회수 | 시간 가중치 + 찜 수 복합 랭킹 |
| Rate Limit | 고정 임계값 | 사용자 등급별 차등 적용 |
