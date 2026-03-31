# Admin Domain Architecture

> 관리자 시스템: 신고 처리, 사용자 제재, 상품 관리, 감사 로그

## 도메인 개요

```mermaid
flowchart TB
    subgraph Admin["관리자 시스템"]
        Dashboard[대시보드]
        ReportMgmt[신고 관리]
        UserMgmt[사용자 관리]
        ProductMgmt[상품 관리]
        AuditLog[감사 로그]
    end

    subgraph UserApp["사용자 앱"]
        Report[신고 접수]
        Block[사용자 차단]
    end

    Report -->|신고 데이터| ReportMgmt
    ReportMgmt -->|제재 부여| UserMgmt
    ReportMgmt -->|상품 숨김| ProductMgmt
    UserMgmt -->|로그인 차단| Auth[인증 시스템]
    Block -->|차단 필터링| ProductFeed[상품 피드]
    Block -->|채팅 차단| Chat[채팅 시스템]

    Dashboard -->|통계 조회| ReportMgmt
    Dashboard -->|통계 조회| UserMgmt
    Dashboard -->|통계 조회| ProductMgmt

    ReportMgmt -->|기록| AuditLog
    UserMgmt -->|기록| AuditLog
    ProductMgmt -->|기록| AuditLog
```

## 신고 시스템

사용자가 부적절한 상품, 사용자, 채팅을 신고하면 관리자가 검토 후 처리한다.

### 신고 플로우

```mermaid
stateDiagram-v2
    [*] --> PENDING: 사용자 신고 접수
    PENDING --> REVIEWING: 관리자 확인
    REVIEWING --> APPROVED: 신고 인정 (제재 조치)
    REVIEWING --> DISMISSED: 신고 기각
    PENDING --> APPROVED: 즉시 처리
    PENDING --> DISMISSED: 즉시 기각
```

### 신고 유형

| 유형 | 진입점 | 피신고자 결정 |
|------|--------|-------------|
| 상품 신고 | 상품 상세 메뉴 | 자동 (판매자) |
| 사용자 신고 | 프로필 메뉴 | 직접 지정 |
| 채팅 신고 | 채팅방 메뉴 | 자동 (채팅 상대방) |

### 신고 처리 액션

| 액션 | 효과 |
|------|------|
| 기각 | 신고 기각, 추가 조치 없음 |
| 경고 | 사용자에게 경고 기록 (영구 누적) |
| 상품 숨김 | 해당 상품을 피드/검색에서 제거 |
| 임시 정지 | 로그인 차단 (1일/3일/7일/30일) |
| 영구 차단 | 로그인 영구 차단 |

## 제재 시스템

사용자 상태는 별도 테이블에서 파생한다. User 테이블에 status 컬럼을 두지 않고, 제재 이력 테이블에서 활성 제재 여부를 계산한다.

### 상태 판별

```mermaid
flowchart TD
    Login[로그인 요청] --> Check{활성 제재 조회}
    Check -->|PERMANENT_BAN 있음| Banned[영구 차단 - 로그인 거부]
    Check -->|TEMPORARY_BAN 있고 기간 내| Suspended[임시 정지 - 로그인 거부 + 해제일 안내]
    Check -->|TEMPORARY_BAN 있고 기간 만료| AutoRelease[자동 해제 → 정상 로그인]
    Check -->|활성 제재 없음| Active[정상 로그인]
```

### 제재 유형

| 유형 | 해제 | 누적 |
|------|------|------|
| 경고 (WARNING) | 해제 불가 | 영구 누적 |
| 임시 정지 (TEMPORARY_BAN) | 기간 만료 시 자동 해제 | - |
| 영구 차단 (PERMANENT_BAN) | 관리자 수동 해제만 가능 | - |

## 차단 시스템

사용자 간 차단 기능. 단방향 차단 (차단한 측에게만 영향).

### 차단 시 필터링 범위

| 영역 | 동작 |
|------|------|
| 상품 피드/검색/트렌딩 | 차단한 사용자의 상품 제외 |
| 좋아요 목록 | 차단한 사용자의 상품 제외 |
| 채팅 | 신규 생성 불가 + 기존 채팅 메시지 송수신 차단 |
| 상품 상세 | 접근 허용 (거래 이력 확인 보장) |
| 프로필 | 접근 허용 + 차단 상태 표시 (차단 해제 경로 보장) |

### 차단 정책

- **단방향**: A가 B를 차단 → A에게만 B가 숨겨짐. B는 변화 없음.
- **채팅**: 양방향 차단 (MVP). 어느 한 쪽이라도 차단하면 메시지 송수신 불가.
- **차단당한 측**: 차단 사실을 알 수 없음.

## 관리자 보안 아키텍처

### 4중 방어

```mermaid
flowchart LR
    Request[관리자 요청] --> L1[네트워크\nWAF IP 화이트리스트]
    L1 --> L2[인증\nJWT + clientType 교차 검증]
    L2 --> L3[인가\nSpring Security hasRole ADMIN]
    L3 --> L4[감사\nAdminActionLog 기록]
    L4 --> Action[관리 액션 실행]
```

| 레이어 | 우회 시나리오 차단 |
|--------|------------------|
| 네트워크 | 외부에서 관리 페이지/API 접근 자체 불가 |
| 인증 | 일반 사용자가 관리자로 로그인 불가 (clientType 교차 검증) |
| 인가 | JWT를 조작해도 서버에서 역할 강제 검증 |
| 감사 | 모든 관리 행위를 IP + 기기 정보와 함께 추적 |

### 감사 로그

모든 관리자 액션은 자동 기록된다:

- **기록 대상**: 로그인, 신고 처리, 제재 부여/해제, 상품 숨김/삭제
- **기록 항목**: 관리자 ID, 액션 유형, 대상 ID, IP, User-Agent (기기/브라우저/OS 파싱)
- **조회 불가 액션은 기록하지 않음** (목록 조회, 상세 조회 등)

## 데이터 모델

```mermaid
erDiagram
    User ||--o{ Report : "신고자/피신고자"
    User ||--o{ UserBlock : "차단자/피차단자"
    User ||--o{ UserSanction : "제재 대상"
    Product ||--o{ Report : "피신고 상품"
    ChatRoom ||--o{ Report : "피신고 채팅방"
    Report ||--o| UserSanction : "연관 제재"
    User ||--o{ AdminActionLog : "관리자 행위"

    Report {
        Long id PK
        ReportType reportType
        ReportReason reportReason
        ReportStatus status
        String description
        String contentSnapshot
        String adminNote
        LocalDateTime processedAt
    }

    UserBlock {
        Long id PK
        Long blockerId FK
        Long blockedId FK
    }

    UserSanction {
        Long id PK
        Long userId FK
        SanctionType sanctionType
        String reason
        LocalDateTime startAt
        LocalDateTime endAt
        boolean isActive
        Long processedBy FK
    }

    AdminActionLog {
        Long id PK
        Long adminUserId
        AdminActionType actionType
        Long targetId
        String clientIp
        String deviceType
        String browser
        String os
    }
```
