아키텍처 설명 문서 (에이전트용)

프로젝트: 토이 프로젝트 - 아이돌 굿즈 중고 장터 (Cherry)

목적

- 이 문서는 현재 운영/배포 환경의 “사실”을 정리한 문서다.
- AI 에이전트는 구현/수정 시 이 문서를 전제로 한다.
- 문서에 없는 인프라 요소(예: EKS, Terraform 등)는 존재하지 않는 것으로 간주한다.

도메인

- Front: https://cheryi.com
- API: https://api.cheryi.com

머메이드 다이어그램

```mermaid
flowchart LR
  U[User Browser] -->|HTTPS 443| CF[CloudFront - Front CDN]
  CF -->|GET static assets| S3[S3 - Static Website Bucket]

  U -->|HTTPS 443 / API calls| ALB[Application Load Balancer - api.cheryi.com]
  ALB -->|Forward HTTP 8080| EC2[EC2 Instance (Docker Compose)]
  EC2 -->|JDBC 3306| RDS[(RDS MySQL)]
  EC2 -->|TCP 6379| Redis[(Redis Container)]

  subgraph VPC[AWS VPC (ap-northeast-2)]
    ALB
    EC2
    RDS
  end
```

트래픽 흐름 (런타임)

1. 사용자가 브라우저에서 https://cheryi.com 접속

- CloudFront가 S3에 업로드된 정적 파일(React/Vite 빌드 결과)을 서빙한다.
- 브라우저는 동일 도메인에서 JS/CSS/이미지를 로드한다.

2. 프론트엔드가 API 호출

- 브라우저에서 https://api.cheryi.com 으로 API 요청을 보낸다.
  - 요청은 ALB(HTTPS 443)로 들어간다.
  - ALB는 Target Group을 통해 EC2의 애플리케이션 포트(HTTP 8080)로 포워딩한다.
- Spring Boot는 비즈니스 로직 처리 중 RDS(MySQL)와 Redis를 사용한다.
- 응답은 ALB를 통해 브라우저로 반환된다.

이미지 업로드 파이프라인 (S3 + Lambda)

흐름

```mermaid
sequenceDiagram
    participant C as Client
    participant B as Backend
    participant S3 as S3
    participant L as Lambda
    participant DB as Database

    C->>B: POST /api/upload/images (요청: 파일 개수, 확장자)
    B->>B: 유효성 검증 (최대 10장, 확장자, 크기)
    B->>S3: Generate Presigned URL (PutObject)
    B-->>C: Presigned URL 목록 반환
    C->>S3: PUT (원본 이미지 직접 업로드)
    S3->>L: S3 Event (ObjectCreated)
    L->>S3: GetObject (원본)
    L->>L: Resize (detail: 1280px, thumb: 256x256)
    L->>S3: PutObject (detail, thumb)
    L->>B: POST /internal/images/complete (콜백)
    B->>DB: ProductImage 저장/업데이트
    B-->>L: 200 OK
```

Backend Presigned URL 발급

- IAM 사용자: cheryi-backend-s3-uploader
- 권한: s3:PutObject (products/original/* 경로만)
- 환경변수 (GitHub Secrets → EC2 .env):
  - AWS_ACCESS_KEY_ID
  - AWS_SECRET_ACCESS_KEY
  - AWS_REGION
  - AWS_S3_BUCKET
- Presigned URL 만료: 5분 (300초)

업로드 제한

- 최대 이미지 수: 10장
- 장당 최대 크기: 10MB
- 허용 확장자: jpg, jpeg, png, webp
- 허용 Content-Type: image/jpeg, image/png, image/webp

S3 버킷/리전

- 버킷: cheryi-product-images-prod
- 리전: ap-northeast-2

경로 규칙

- 원본 업로드(prefix): products/original/
- 리사이즈 결과:
  - products/detail/
  - products/thumb/
- 기존 mock 경로(레거시/테스트):
  - mock/products/images/detail/
  - mock/products/images/thumb/

보안/접근 정책

- products/original/ 은 공개 접근 불가
- products/detail/ 및 products/thumb/ 은 공개 GET 허용
- mock 경로도 퍼블릭 GET 허용(기존 연결용)

Lambda 동작(런타임: Node.js 20.x)

- S3 이벤트 key 디코딩: decodeURIComponent(key.replace(/\+/g, ' '))
- 재귀 방지 가드
  - products/original/ 이외는 즉시 return
  - products/detail/ 또는 products/thumb/ 는 즉시 return
- 리사이즈 규칙
  - detail: 긴 변 1280px, 비율 유지, withoutEnlargement: true
  - thumb: resize(256, 256, { fit: 'cover', position: 'centre' })
- S3 업로드 시 CacheControl: public, max-age=31536000, immutable
- 업로드 보호: HeadObject(ContentLength)로 20MB 상한 체크
  - IAM에 s3:GetObject 권한이 포함되어야 HeadObject 사용 가능
- 콜백 호출
  - URL: https://api.cheryi.com/internal/images/complete
  - Header: X-Internal-Token: INTERNAL_TOKEN
  - Body: imageKey, detailUrl, thumbnailUrl, imageOrder(0), isThumbnail(false)
- 콜백 실패 시 CloudWatch 로그에 awsRequestId, originalKey, detailKey, thumbKey, callbackStatus, callbackBody(1KB truncate) 포함 후 throw
- 실패는 throw로 남기고, 필요 시 DLQ/SQS 연동 여지를 둔다
- 상품 수정 시 이미지 재처리
  - 신규 이미지 추가 시 상품 상태가 PENDING으로 전환 (revertToPending)
  - 모든 이미지 처리 완료 시 자동으로 SELLING으로 전환
- 삭제된 상품 가드
  - Lambda 콜백 시 deleted_at이 설정된 상품은 처리하지 않고 스킵
  - S3 배치 정리: 매일 03시 스케줄러, 삭제 후 30일 유예기간 경과한 이미지 삭제

URL 정책

- 현재는 S3 public URL 사용
- 추후 CloudFront로 전환 가능(옵션)

포트/프로토콜

- Front 도메인 (cheryi.com)
  - 사용자 → CloudFront: HTTPS 443
  - CloudFront → S3: AWS 내부 통신(정적 파일 GET)
- API 도메인 (api.cheryi.com)
  - 사용자 → ALB: HTTPS 443
  - ALB → EC2 app 컨테이너: HTTP 8080
  - EC2 app → RDS MySQL: TCP 3306
  - app 컨테이너 → redis 컨테이너: TCP 6379 (docker network)

보안 경계/인증서

- 인증서(ACM)는 api.cheryi.com에 대해 발급되어 ALB(HTTPS Listener)에 연결된다.
- 프론트(cheryi.com)는 CloudFront/도메인 쪽에서 HTTPS 제공(별도 인증서 관리).
- API는 프론트 도메인과 다른 Origin이므로, prod에서는 CORS를 https://cheryi.com만 허용한다.

배포 흐름
Front 배포 - GitHub Actions로 프론트 빌드 - 빌드 결과물을 S3에 업로드 - CloudFront가 S3의 정적 파일을 캐시/서빙

Backend 배포 - GitHub Actions로 백엔드 Docker 이미지 빌드 → GHCR push - EC2에서 docker compose로 이미지 pull 후 컨테이너 재시작 - 환경변수는 GitHub Secrets에서 전달되어 EC2에 .env 생성 후 docker compose –env-file로 주입 - 컨테이너는 8080 포트로 구동

이미지 파이프라인 검증 시나리오 (운영 체크)

- products/original/<uuid>.jpg 업로드
- Lambda 실행 로그(CloudWatch) 확인
- products/detail/, products/thumb/ 생성 확인
- detail/thumb URL 200 확인
- 콜백 응답 200/204 확인
- 실패 케이스
  - original 아닌 prefix 업로드 시 즉시 return 로그
  - INTERNAL_TOKEN 누락 시 401/403 로그
  - ContentLength 상한 초과 시 실패 로그

운영 환경의 “사실” 체크리스트 - EC2 내부: curl http://localhost:8080/health 는 200이어야 한다. - 외부: curl https://api.cheryi.com/health 는 200이어야 한다. - 브라우저에서 https://cheryi.com은 정적 화면을 정상 렌더링해야 한다. - 브라우저에서 API 호출 시 CORS 정책은 prod에서 https://cheryi.com만 허용해야 한다.

에이전트 작업 규칙 - 인프라 변경(ALB/CloudFront/S3/Route53/ACM)은 별도 지시 없으면 하지 않는다. - 백엔드 관련 수정은 기본적으로 “EC2 Docker Compose + ALB 443→8080”을 전제로 한다. - 프론트 관련 수정은 기본적으로 “S3 + CloudFront 정적 배포”를 전제로 한다. - API Base URL(prod)은 https://api.cheryi.com 이다. - Front URL(prod)은 https://cheryi.com 이다.
