# 저비용 인프라 전환 설계

> 기존 AWS 인프라를 **최소 비용으로 다시 구성**하기 위한 설계 기록. 같은 서비스를 더 낮은 비용으로 어떻게 다시 올릴지 고민하고, 무엇을 유지하고 무엇을 바꿀지 판단한 과정을 정리한다.

---

## 1. 개요

이 문서는 "서비스를 다시 배포했다"는 결과 보고가 아니라, **기존 설계 의도를 유지하면서 인프라 비용을 최소화하도록 다시 구성하는 방법을 고민·설계한 기록**이다.

- 기존 AWS 무료 계정 종료로 공개 배포가 중단되면서, 같은 서비스를 더 낮은 비용으로 다시 올리는 방법을 검토했다.
- 프로젝트를 폐기한 것은 아니며, 데이터 계층과 이미지 처리 구조 같은 핵심 설계는 유지한 채 비용이 큰 리소스만 대체하는 방향으로 잡았다.
- 상용 서비스 수준의 상시 인프라 비용이 아니라, 열람·데모 트래픽을 기준으로 비용 최소화를 우선한다.
- 아래 전환 후보 구조는 설계 단계이며 실제 이전은 아직 진행하지 않았다 (구현 상태는 [8장](#8-구현-상태-구분) 참조).

관련 기존 문서:

- [System Overview](./system-overview.md) — 기존 AWS 인프라 구성
- [CI/CD + Runtime](./cicd.md) — 기존 배포 파이프라인
- [Image Pipeline Domain](./image-pipeline-domain.md) — 기존 이미지 파이프라인 상세

---

## 2. 현재 상태

| 항목 | 상태 |
|------|------|
| 기존 AWS 인프라 | 계정 종료로 **내려간 상태** |
| 공개 배포 | **중단** |
| 로컬 실행 | 가능 |
| 포트폴리오 증빙 | 필요 시 로컬에서 주요 화면 재캡처로 보완 가능 |
| S3 이미지 자산 | 로컬 백업이 없다면 복구 대상으로 두지 않음 |

당시의 아키텍처·성능 개선 문서는 공개되어 있으나, 실제 서비스 화면에는 현재 접근할 수 없다.

---

## 3. 기존 AWS 구성 *(Implemented in previous AWS version)*

실제로 구현·운영했던 구조다. 상세는 [System Overview](./system-overview.md), [CI/CD](./cicd.md) 참조.

```mermaid
flowchart TD
    A["React / Vite"] --> B["S3 + CloudFront<br/>(정적 배포)"]
    A -->|API / WSS| C["Spring Boot / EC2<br/>(Docker Compose)"]
    C --> D["RDS MySQL"]
    C --> E["Redis<br/>(Cache-Aside)"]

    subgraph IMG["이미지 처리 (비동기)"]
        F["S3 Original"] --> G["S3 ObjectCreated Event"]
        G --> H["Lambda + sharp"]
        H --> I["Detail 1280px / Thumb 256px"]
        I --> J["S3 (리사이즈 결과)"]
        J --> K["Spring Boot Callback"]
    end

    C -->|Presigned URL 발급| F
    K --> C
```

| 레이어 | 구현 기술 |
|--------|-----------|
| Frontend | React / Vite |
| 정적 배포 | S3 + CloudFront |
| Backend | Spring Boot / EC2 (Docker Compose) |
| DB | RDS MySQL |
| Cache | Redis (Cache-Aside) |
| Image Storage | S3 |
| Image Processing | S3 Event → Lambda(sharp) → 리사이즈 → Callback |

---

## 4. 기존 이미지 처리 흐름 *(Implemented)*

```mermaid
sequenceDiagram
    participant C as Client
    participant B as Spring Boot
    participant S3 as S3
    participant L as Lambda(sharp)

    B->>S3: Presigned URL 발급
    C->>S3: PUT 원본 이미지 직접 업로드
    S3->>L: ObjectCreated 이벤트
    L->>L: Detail(1280px) / Thumb(256px) 생성
    L->>S3: 리사이즈 결과 저장
    L->>B: 처리 완료 Callback
    B->>B: 이미지 메타데이터 반영 (PENDING → SELLING)
```

핵심은 **원본 저장과 이미지 후처리를 분리**하고, **업로드 이벤트를 기준으로 비동기 처리**한다는 점이다. 이 구조 덕분에 리사이징 부하가 API 서버와 경쟁하지 않는다.

---

## 5. 전환 원칙 *(Designed)*

전환의 목표는 "AWS 서비스를 1:1로 재현"하는 것이 아니라, **아래 설계 의도를 유지하면서 상시 비용을 최소화**하는 것이다.

| 원칙 | 유지하려는 이유 |
|------|-----------------|
| **MySQL/JPA 구조 유지** | 기존 DB 설계·인덱스·조회 성능 개선 기록과 실제 코드의 일관성 유지 |
| **Cache-Aside 구조 유지** | Redis 또는 호환 가능한 Valkey 계열로 기존 캐시 전략 보존 |
| **이미지 처리를 API 서버와 분리** | 리사이징이 API 서버 CPU/메모리와 경쟁하지 않도록 책임 분리 유지 |
| **비동기 이미지 처리 유지** | Object Storage 이벤트 이후 별도 Worker가 처리하는 구조 유지 |
| **포트폴리오 트래픽 기준 설계** | 상용 수준 고정 비용보다 열람·데모 목적의 비용 최소화 우선 |

> [!NOTE]
> 무료 플랜의 조건·제한(용량, 요청 수, 슬립 정책 등)은 시점에 따라 바뀌므로, **실제 이전 시점에 다시 조사한 뒤 확정**해야 한다.

---

## 6. 저비용 전환 후보 구조 *(Designed / Not Implemented)*

```mermaid
flowchart TD
    A["React / Vite"] --> B["무료 Static Hosting"]
    A -->|API| C["Spring Boot / 무료 Runtime"]
    C --> D["Managed MySQL Free"]
    C --> E["Managed Redis / Valkey Free"]

    subgraph IMG["이미지 처리 (비동기, 후보)"]
        F["Cloudflare R2<br/>(Original)"] --> G["Object Created Event"]
        G --> H["Queue"]
        H --> I["Image Worker"]
        I --> J["Resize / Transform"]
        J --> K["R2 Detail / Thumb"]
        K --> L["Spring Boot Callback"]
    end

    C -->|업로드 URL 발급| F
    L --> C
```

| 역할 | 후보 | 전환 방향 |
|------|------|-----------|
| Frontend | Cloudflare Pages / Firebase Hosting 등 무료 정적 호스팅 | React 정적 배포 |
| Backend | 무료 Container / Web Runtime | Spring Boot 유지 |
| DB | Managed MySQL Free | 기존 MySQL/JPA 코드 최대한 유지 |
| Cache | Managed Redis / Valkey Free | 기존 Cache-Aside 구조 유지 |
| Image Storage | Cloudflare R2 | S3 대체 Object Storage |
| Image Processing | Queue + Worker + Transform | 기존 S3 Event + Lambda 책임 분리 보존 |

> 위 서비스명은 모두 **전환 후보**이며 확정된 선택이 아니다. 실제 이전 시점에 무료 플랜 조건을 재조사해 최종 결정한다.

---

## 7. 이미지 파이프라인 전환 방향 *(Designed)*

구현 기술은 바뀌어도 **설계 의도는 그대로 유지**한다. 아래는 기존과 전환안의 구조 비교다.

```mermaid
flowchart LR
    subgraph AWS["기존 (Implemented)"]
        direction TB
        A1["S3 Original"] --> A2["S3 Event"]
        A2 --> A3["Lambda + sharp"]
        A3 --> A4["S3 Detail / Thumb"]
        A4 --> A5["Backend Callback"]
    end

    subgraph LOW["전환안 (Designed)"]
        direction TB
        B1["R2 Original"] --> B2["Object Event"]
        B2 --> B3["Queue"]
        B3 --> B4["Worker (Resize)"]
        B4 --> B5["R2 Detail / Thumb"]
        B5 --> B6["Backend Callback"]
    end

    AWS -.전환.-> LOW
```

이 구조를 유지하면 구현 기술이 달라져도 다음 특성이 보존된다.

- 이미지 처리 실패가 일반 API 요청에 직접 영향을 주지 않는다.
- 이미지 리사이징 작업을 별도 실행 환경으로 격리한다.
- 비동기 처리 및 재처리(재시도) 구조로 확장할 수 있다.
- 원본 / Detail / Thumbnail 저장 규칙을 유지할 수 있다.

전환안이 큐(Queue)를 명시적으로 두는 점은 기존 `S3 Event → Lambda` 대비 재처리·백프레셔 제어 지점을 드러내려는 의도이며, 이 역시 실제 구현 시점에 검증할 대상이다.

---

## 8. 구현 상태 구분

읽는 사람이 "이미 만든 것"과 "앞으로 옮길 설계안"을 혼동하지 않도록 상태를 명확히 구분한다.

| 상태 | 대상 |
|------|------|
| ✅ **Implemented (previous AWS version)** | React/Vite, S3+CloudFront, Spring Boot/EC2, RDS MySQL, Redis, S3 이미지 저장, S3 Event → Lambda(sharp) → 리사이즈 → Callback 파이프라인 |
| 📐 **Designed for low-cost migration** | 전환 원칙, 저비용 후보 구조, 이미지 파이프라인 전환 방향(R2 + Queue + Worker) |
| ⛔ **Not implemented yet** | 무료 Static Hosting / Runtime 배포, Managed MySQL·Redis 이전, Cloudflare R2 이전, Queue·Worker 파이프라인 구현 |

---

## 9. 포트폴리오 보완 계획 *(Not Implemented)*

실제 인프라 이전보다 아래 작업을 먼저 수행한다.

- [ ] 로컬에서 체리를 실행해 대표 화면 캡처
- [ ] 상품 목록 / 상세 / 필터·정렬 / 마이페이지 / 관리자 화면 확보
- [ ] 가능하면 이미지 업로드 → 처리 완료 → Thumbnail 표시 흐름 캡처
- [ ] 기존 공개 아키텍처 문서에 배포 중단 이유 명시

---

## 10. 추후 구현 순서 *(Not Implemented)*

1. [ ] 로컬 실행 및 주요 기능 정상 동작 확인
2. [ ] 샘플 데이터와 이미지 자산 재구성
3. [ ] 실제 이전 시점의 무료 플랜 / 제약 재조사
4. [ ] MySQL 이전 및 Spring Boot 연결
5. [ ] Redis / Valkey 연결
6. [ ] Backend / Frontend 배포
7. [ ] Object Storage(R2) 이전
8. [ ] 이미지 이벤트 → Queue → Worker 파이프라인 구현
9. [ ] 전체 기능 및 포트폴리오 링크 검증

---

## 11. 마무리 메모

이 문서는 재배포 성과를 자랑하기 위한 것이 아니라, **무엇을 유지하고 무엇을 바꿀지 판단한 과정**을 남기기 위한 것이다.

- 유지: MySQL/JPA 데이터 계층, Cache-Aside 전략, 이미지 후처리의 책임 분리와 비동기 구조
- 변경: 상시 비용이 드는 관리형·컴퓨팅 리소스를 무료 플랜 기반 후보로 대체

포트폴리오용 프로젝트라도, 단순히 다시 띄우는 것보다 **비용 제약 안에서 설계 의도를 어떻게 지킬지 고민한 기록**을 남기는 데 의미가 있다고 판단했다.
