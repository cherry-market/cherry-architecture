# ERD

```mermaid
erDiagram
  USERS ||--o{ PRODUCTS : sells
  CATEGORIES ||--o{ PRODUCTS : categorizes
  PRODUCTS ||--o{ PRODUCT_IMAGES : has
  PRODUCTS ||--o{ PRODUCT_TAGS : tagged
  TAGS ||--o{ PRODUCT_TAGS : tags
  USERS ||--o{ PRODUCT_LIKES : likes
  PRODUCTS ||--o{ PRODUCT_LIKES : liked_by
  PRODUCTS ||--o{ CHAT_ROOMS : discussed_in
  USERS ||--o{ CHAT_ROOMS : "buys/sells"
  CHAT_ROOMS ||--o{ CHAT_MESSAGES : contains
  USERS ||--o{ CHAT_MESSAGES : sends
  CHAT_ROOMS ||--o{ CHAT_READ_POSITIONS : tracks

  USERS {
    bigint id PK
    varchar email
    varchar nickname
    varchar password
    varchar profile_image_url
    datetime created_at
    datetime updated_at
  }

  CATEGORIES {
    bigint id PK
    varchar code
    varchar display_name
    boolean is_active
    int sort_order
    datetime created_at
    datetime updated_at
  }

  PRODUCTS {
    bigint id PK
    bigint seller_user_id FK
    varchar title
    text description
    int price
    enum status "PENDING|SELLING|RESERVED|SOLD"
    enum trade_type "DIRECT|DELIVERY|BOTH"
    bigint category_id FK
    datetime deleted_at
    datetime created_at
    datetime updated_at
  }

  PRODUCT_IMAGES {
    bigint id PK
    bigint product_id FK
    varchar original_url
    varchar image_url
    varchar thumbnail_url
    int image_order
    boolean is_thumbnail
    datetime created_at
    datetime updated_at
  }

  TAGS {
    bigint id PK
    varchar name
    datetime created_at
    datetime updated_at
  }

  PRODUCT_TAGS {
    bigint id PK
    bigint product_id FK
    bigint tag_id FK
    datetime created_at
    datetime updated_at
  }

  PRODUCT_LIKES {
    bigint id PK
    bigint user_id FK
    bigint product_id FK
    datetime created_at
    datetime updated_at
  }

  CHAT_ROOMS {
    bigint id PK
    bigint product_id FK
    bigint buyer_id FK
    bigint seller_id FK
    datetime last_message_at
    datetime buyer_left_at
    datetime seller_left_at
    datetime created_at
    datetime updated_at
  }

  CHAT_MESSAGES {
    bigint id PK
    bigint room_id FK
    bigint sender_id FK
    enum message_type "TEXT|IMAGE|SYSTEM"
    text content
    varchar client_message_id
    datetime created_at
    datetime updated_at
  }

  CHAT_READ_POSITIONS {
    bigint id PK
    bigint room_id FK
    bigint user_id FK
    bigint last_read_message_id
    datetime created_at
    datetime updated_at
  }
```

---

## 테이블 설명

### 사용자 및 인증

| 테이블 | 역할 | 비고 |
|--------|------|------|
| **USERS** | 회원 정보 (이메일, 닉네임, BCrypt 비밀번호) | email/nickname UNIQUE |

### 상품

| 테이블 | 역할 | 비고 |
|--------|------|------|
| **PRODUCTS** | 상품 정보 (제목, 설명, 가격, 상태, 거래방식) | Soft Delete (`deleted_at`), 상태: PENDING→SELLING→RESERVED→SOLD |
| **CATEGORIES** | 상품 카테고리 마스터 (포토카드, 앨범 등) | `code`로 필터링, `sort_order`로 정렬 |
| **PRODUCT_IMAGES** | 상품 이미지 (원본/리사이즈/썸네일 URL) | Lambda 처리 전 `image_url=null`, `is_thumbnail`로 대표 이미지 지정 |
| **TAGS** | 태그 마스터 (상품 등록 시 자동 생성) | name UNIQUE |
| **PRODUCT_TAGS** | 상품-태그 N:M 매핑 | |
| **PRODUCT_LIKES** | 찜(위시) 기록 | `UNIQUE(user_id, product_id)` 중복 방지 |

### 채팅

| 테이블 | 역할 | 비고 |
|--------|------|------|
| **CHAT_ROOMS** | 1:1 채팅방 (상품 기반) | `UNIQUE(product_id, buyer_id)`, `buyer_left_at`/`seller_left_at`으로 나가기 관리 |
| **CHAT_MESSAGES** | 채팅 메시지 (TEXT/IMAGE/SYSTEM) | `UNIQUE(room_id, client_message_id)` 멱등성 보장 |
| **CHAT_READ_POSITIONS** | 읽음 위치 추적 | `last_read_message_id` 단조 증가 |

## 주요 관계

- **1:N** — USERS → PRODUCTS (판매자), PRODUCTS → PRODUCT_IMAGES, CHAT_ROOMS → CHAT_MESSAGES
- **N:M** — PRODUCTS ↔ TAGS (PRODUCT_TAGS 중간 테이블)
- **1:1 (논리적)** — CHAT_ROOMS는 `UNIQUE(product_id, buyer_id)`로 상품-구매자 쌍당 1개 채팅방

## 설계 특징

- 모든 테이블에 `created_at`/`updated_at` 존재 (`BaseTimeEntity` 상속)
- PRODUCTS의 `deleted_at`으로 Soft Delete 구현 (30일 유예 후 물리 삭제)
- ENUM 타입은 MySQL ENUM으로 매핑 (`@Enumerated(STRING)`)
- AUTO_INCREMENT BIGINT PK 사용 (UUID 미사용, 성능 우선)
