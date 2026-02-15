# CI/CD & Runtime Architecture (Frontend + Backend)

```mermaid
flowchart LR
    subgraph CI_CD [GitHub Actions]
        GHA_FE[FE Build & S3 Upload]
        GHA_BE[BE Docker Build & Deploy]
    end

    subgraph User_Interaction [User]
        User((User))
    end

    subgraph AWS_Global [AWS Cloud - Global]
        Route53[Route 53]
        CloudFront[Amazon CloudFront]
        S3Static[(Amazon S3 - Static)]
        S3Media[(Amazon S3 - Product Images)]
    end

    subgraph AWS_Region [AWS Cloud - ap-northeast-2]
        subgraph VPC [VPC]
            subgraph ALB [ALB: cheryi-api-alb]
                L80[Listener: HTTP 80]
                L443[Listener: HTTPS 443\nACM: *.cheryi.com]
            end

            subgraph Target_Group [Target Group: cheryi-api-tg]
                EC2[EC2 Instance\nDocker App: 8080\nHealth Check: /health]
            end

            RDS[(Amazon RDS - MySQL)]
            Redis[(Redis)]
        end
    end

    %% CI/CD Flow
    GHA_FE --> S3Static
    GHA_BE --> EC2

    %% Frontend Flow
    User -- "HTTPS 443\nhttps://cheryi.com" --> CloudFront
    CloudFront -- "Origin Access" --> S3Static

    %% Backend Flow
    User -- "https://api.cheryi.com" --> Route53
    Route53 -- "HTTPS 443" --> L443
    User -- "HTTP 80" --> L80
    L80 -- "301 Redirect" --> L443
    L443 -- "Forward (HTTP 8080)" --> EC2
    EC2 -- "JDBC 3306" --> RDS
    EC2 -- "TCP 6379" --> Redis

    %% Media Flow
    User -- "Image GET" --> S3Media
```

---

## 배포 구조 설명

### Frontend (React + Vite)

| 단계 | 도구 | 동작 |
|------|------|------|
| 빌드 | GitHub Actions | `npm run build` → 정적 파일 생성 |
| 업로드 | AWS CLI | S3 Static 버킷에 업로드 |
| 서빙 | CloudFront | S3를 Origin으로 CDN 서빙 |
| 도메인 | Route 53 | `cheryi.com` → CloudFront 연결 |

**특징:**
- Vite 빌드 결과를 S3에 직접 업로드 (서버 불필요)
- CloudFront로 글로벌 CDN 서빙 (Edge Location 캐싱)
- HTTPS는 CloudFront에서 ACM 인증서로 처리

### Backend (Spring Boot + Docker)

| 단계 | 도구 | 동작 |
|------|------|------|
| 빌드 | GitHub Actions | `./gradlew build` → JAR 생성 → Docker 이미지 빌드 |
| 레지스트리 | GHCR | Docker 이미지 Push (GitHub Container Registry) |
| 배포 | SSH + Docker Compose | EC2에서 이미지 Pull → 컨테이너 재시작 |
| 서빙 | ALB | HTTPS 443 → EC2 HTTP 8080 포워딩 |
| 도메인 | Route 53 | `api.cheryi.com` → ALB 연결 |

**특징:**
- Docker Compose로 Spring Boot 앱 + Redis 컨테이너 관리
- ALB에서 HTTPS 종료 (ACM 인증서, `*.cheryi.com`)
- HTTP 80 → HTTPS 443 자동 리다이렉트
- Health Check: `/health` 엔드포인트

## 인프라 구성

### 네트워크

| 구성 요소 | 역할 |
|-----------|------|
| **Route 53** | DNS 관리 (`cheryi.com`, `api.cheryi.com`) |
| **CloudFront** | 프론트엔드 CDN + HTTPS |
| **ALB** | 백엔드 로드밸런서 + HTTPS 종료 |
| **VPC** | 백엔드 인프라 격리 (ap-northeast-2) |

### 컴퓨팅 및 스토리지

| 구성 요소 | 역할 | 비고 |
|-----------|------|------|
| **EC2** | Spring Boot 앱 실행 | Docker Compose 기반 |
| **RDS MySQL** | 관계형 데이터 저장 | VPC 내부, JDBC 3306 |
| **Redis** | 캐싱 + Rate Limit + 트렌딩 | Docker 컨테이너, TCP 6379 |
| **S3 Static** | 프론트엔드 정적 파일 | CloudFront Origin |
| **S3 Media** | 상품 이미지 저장 | Presigned URL 업로드, Lambda 리사이즈 |

### 보안

- ALB에서 HTTPS 종료 (ACM 인증서)
- Frontend/Backend 다른 Origin → CORS 정책 적용
- 원본 이미지 경로 비공개, 리사이즈 결과만 공개
- VPC 내부 통신 (EC2 ↔ RDS, EC2 ↔ Redis)
