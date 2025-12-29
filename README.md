# 🐾 PetClinic Dev

Spring PetClinic Microservices 소스 코드 및 Multi-Cloud CI/CD 파이프라인

## 🏛️ 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway                            │
│                      (Port 8080)                            │
└─────────────────────────┬───────────────────────────────────┘
                          │
    ┌─────────────────────┼─────────────────────┐
    │                     │                     │
    ▼                     ▼                     ▼
┌────────────┐     ┌────────────┐       ┌────────────┐
│ Customers  │     │   Visits   │       │    Vets    │
│  Service   │     │  Service   │       │  Service   │
│ (Port 8081)│     │ (Port 8082)│       │ (Port 8083)│
└────────────┘     └────────────┘       └────────────┘
         │                │                    │
         └────────────────┼────────────────────┘
                          │
                   ┌──────▼──────┐
                   │ MySQL (RDS) │
                   └─────────────┘
```

## 🧩 서비스 구성

| 서비스 | 포트 | 설명 |
|--------|------|------|
| config-server | 8888 | 중앙 설정 관리 |
| discovery-server | 8761 | Eureka 서비스 디스커버리 |
| api-gateway | 8080 | API 라우팅 |
| customers-service | 8081 | 고객/펫 관리 |
| visits-service | 8082 | 방문 기록 관리 |
| vets-service | 8083 | 수의사 정보 관리 |
| admin-server | 9090 | Spring Boot Admin |

## 📁 디렉토리 구조

```
petclinic-dev/
├── .github/
│   └── workflows/
│       └── petclinic-ci.yml          # Multi-Cloud CI/CD 파이프라인
├── spring-petclinic-admin-server/    # Spring Boot Admin (포트 9090)
├── spring-petclinic-api-gateway/     # API Gateway (포트 8080)
├── spring-petclinic-config-server/   # 중앙 설정 서버 (포트 8888)
├── spring-petclinic-customers-service/  # 고객/펫 관리 (포트 8081)
├── spring-petclinic-discovery-server/   # Eureka 서비스 디스커버리 (포트 8761)
├── spring-petclinic-genai-service/   # GenAI 서비스
├── spring-petclinic-vets-service/    # 수의사 정보 (포트 8083)
├── spring-petclinic-visits-service/  # 방문 기록 (포트 8082)
├── docker/
│   ├── Dockerfile                    # 공통 Docker 이미지 빌드
│   ├── grafana/                      # Grafana 대시보드 설정
│   │   ├── dashboards/               # JVM, HTTP 대시보드
│   │   ├── provisioning/             # 데이터소스 프로비저닝
│   │   └── grafana.ini               # Grafana 설정
│   └── prometheus/
│       └── prometheus.yml            # Prometheus 스크랩 설정
├── scripts/
│   ├── chaos/                        # Chaos Engineering 스크립트
│   ├── pushImages.sh                 # 이미지 푸시
│   ├── tagImages.sh                  # 이미지 태깅
│   └── run_all.sh                    # 전체 서비스 실행
├── docs/                             # 문서
├── docker-compose.yml                # 로컬 개발용
└── pom.xml                           # Maven 루트 설정
```

## 🚀 로컬 실행

```bash
# 전체 빌드
./mvnw clean install -DskipTests

# Docker Compose로 실행
docker-compose up -d

# 개별 서비스 실행
./mvnw spring-boot:run -pl spring-petclinic-config-server
```

## ☁️ Multi-Cloud CI/CD (GitHub Actions)

`.github/workflows/petclinic-ci.yml` 파이프라인:

### 이미지 저장 전략 (태그 기반)

| 트리거 | AWS ECR | GCP Artifact Registry | 용도 |
|--------|---------|----------------------|------|
| `git push origin main` | ✅ 저장 | ❌ 스킵 | 개발/테스트 |
| `git push origin v1.0.0` | ✅ 저장 | ✅ 저장 | 프로덕션 릴리스 |

```
일반 커밋 (main)                    태그 푸시 (v1.0.0)
      │                                   │
      ▼                                   ▼
┌─────────────┐                   ┌─────────────┐
│  AWS ECR    │ ✅                │  AWS ECR    │ ✅
│  (개발용)   │                   │  (릴리스)   │
└─────────────┘                   ├─────────────┤
                                  │  GCP AR     │ ✅
                                  │  (프로덕션) │
                                  └─────────────┘
```

### 파이프라인 흐름

```
Push to main / Tag (v*.*.*)
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 변경 감지                                                │
│    └── git diff로 수정된 서비스만 식별                      │
├─────────────────────────────────────────────────────────────┤
│ 2. Maven 빌드 (변경된 서비스만)                             │
│    └── Java 17, 테스트 스킵                                 │
├─────────────────────────────────────────────────────────────┤
│ 3. Docker Build & Multi-Cloud Push                         │
│    ├── AWS ECR Push (항상)                                 │
│    └── GCP Artifact Registry Push (태그 푸시 시에만)       │
├─────────────────────────────────────────────────────────────┤
│ 4. 🔔 Slack 알림 - 승인 요청                                │
│    └── 팀장에게 배포 승인 요청 알림 전송                    │
├─────────────────────────────────────────────────────────────┤
│ 5. ✅ 팀장 승인 (GitHub Environment)                        │
│    └── production 환경의 Required Reviewer 승인 대기       │
├─────────────────────────────────────────────────────────────┤
│ 6. GitOps 업데이트                                          │
│    ├── petclinic-gitops 클론                               │
│    ├── overlays/aws/kustomization.yaml 태그 수정           │
│    ├── overlays/gcp/kustomization.yaml 태그 수정           │
│    └── Commit & Push                                        │
├─────────────────────────────────────────────────────────────┤
│ 7. 🔔 Slack 알림 - 배포 완료                                │
│    └── 배포 완료 알림 전송                                  │
├─────────────────────────────────────────────────────────────┤
│ 8. ArgoCD 자동 배포                                         │
│    ├── AWS EKS 배포 (Primary)                              │
│    └── GCP GKE 배포 (DR)                                   │
└─────────────────────────────────────────────────────────────┘
```

### 🔔 Slack 알림 + 승인 프로세스

배포 전 팀장 승인이 필요한 워크플로우가 적용되어 있습니다.

#### 워크플로우 흐름

1. **코드 Push** → 빌드 및 Docker 이미지 생성
2. **Slack 알림** → 팀장에게 승인 요청 알림
3. **승인 대기** → GitHub Actions에서 승인 버튼 클릭 대기
4. **승인 완료** → GitOps 업데이트 및 ArgoCD 배포
5. **완료 알림** → Slack으로 배포 완료 알림

#### GitHub Environment 설정 (필수)

1. Repository → Settings → Environments
2. **New environment** → `production` 생성
3. **Required reviewers** 체크 → 승인자 GitHub 계정 추가
4. **Prevent self-review** 체크 (선택) → 본인이 Push한 경우 본인 승인 불가
5. **Save protection rules** 클릭

> **참고**: Prevent self-review 체크 시, Push한 사람과 다른 승인자가 필요합니다.

#### GitHub Secrets 설정

| Secret | 용도 |
|--------|------|
| `AWS_ROLE_ARN` | AWS OIDC 인증용 IAM Role |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | GCP Workload Identity Provider |
| `GCP_SERVICE_ACCOUNT` | GCP 서비스 계정 |
| `GITOPS_TOKEN` | petclinic-gitops 레포 접근용 PAT |
| `SLACK_WEBHOOK_URL` | Slack 알림용 Incoming Webhook URL |

#### Slack 알림 예시

**승인 요청 알림:**
```
🚀 빌드(CI) 성공! 배포 승인이 필요합니다.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Repository: ParkSeJin0514/petclinic-dev
Image Tag: abc1234
Built Services: api-gateway, customers-service
Author: developer-name
[배포 승인하러 가기] 버튼
```

**배포 완료 알림:**
```
🎉 GitOps 레포 업데이트 완료! ArgoCD가 자동으로 배포합니다.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 🐳 이미지 레지스트리

| 클라우드 | 레지스트리 | 리전 | 저장 조건 |
|---------|-----------|------|----------|
| **AWS** | ECR | ap-northeast-2 | 모든 커밋 |
| **GCP** | Artifact Registry | asia-northeast3 | 태그 푸시만 (v*) |

```bash
# AWS ECR (모든 이미지)
946775837287.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic-msa/petclinic-*

# GCP Artifact Registry (릴리스 이미지만)
asia-northeast3-docker.pkg.dev/kdt2-final-project-t1/petclinic-msa/petclinic-*
```

### 🏷️ 릴리스 방법 (Git 태그)

```bash
# 1. 일반 개발 커밋 (AWS ECR만 저장)
git add .
git commit -m "feat: 새로운 기능 추가"
git push origin main

# 2. 프로덕션 릴리스 (AWS ECR + GCP 저장)
git tag -a v1.0.0 -m "Release v1.0.0 - 첫 번째 정식 릴리스"
git push origin v1.0.0
```

#### 태그 네이밍 컨벤션 (Semantic Versioning)

| 태그 | 설명 | 예시 |
|------|------|------|
| `vX.Y.Z` | 정식 릴리스 | v1.0.0, v1.1.0, v2.0.0 |
| `vX.Y.Z-rc.N` | Release Candidate | v1.0.0-rc.1 |
| `vX.Y.Z` (패치) | 핫픽스 | v1.0.1, v1.0.2 |

#### 태그 관리 명령어

```bash
# 태그 목록 확인
git tag -l "v*"

# 태그 상세 정보
git show v1.0.0

# 태그 삭제 (로컬 + 원격)
git tag -d v1.0.0
git push origin --delete v1.0.0
```

### 📝 선택적 빌드

| 변경 파일 | 빌드 대상 |
|----------|----------|
| `spring-petclinic-api-gateway/*` | api-gateway만 |
| `spring-petclinic-customers-service/*` | customers-service만 |
| `pom.xml` 또는 `.github/workflows/*` | 전체 서비스 (7개) |

### ⚙️ OIDC 인증 (키 없음)

```yaml
# AWS OIDC
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: ap-northeast-2

# GCP Workload Identity
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
    service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
```

## 🐳 Docker 이미지

### 레이어드 빌드 (Dockerfile)
```dockerfile
FROM eclipse-temurin:17-jdk-alpine AS build
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/dependencies/ ./
COPY --from=build /app/spring-boot-loader/ ./
COPY --from=build /app/snapshot-dependencies/ ./
COPY --from=build /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

## 📊 모니터링 (Prometheus + Grafana)

### Actuator & Micrometer 설정

모든 서비스에 Prometheus 메트릭 엔드포인트가 활성화되어 있습니다.

```yaml
# application.yml (모든 서비스 공통)
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    tags:
      application: petclinic    # 모든 메트릭에 application 레이블 추가
    export:
      prometheus:
        enabled: true
```

### 메트릭 엔드포인트

| 서비스 | Prometheus 엔드포인트 |
|--------|----------------------|
| config-server | http://config-server:8888/actuator/prometheus |
| discovery-server | http://discovery-server:8761/actuator/prometheus |
| api-gateway | http://api-gateway:8080/actuator/prometheus |
| customers-service | http://customers-service:8081/actuator/prometheus |
| visits-service | http://visits-service:8082/actuator/prometheus |
| vets-service | http://vets-service:8083/actuator/prometheus |
| admin-server | http://admin-server:9090/actuator/prometheus |

### Grafana 대시보드

- **JVM (Micrometer)**: Heap/Non-Heap 메모리, GC, 스레드 모니터링
- **HTTP 요청**: 요청률, 에러율, 응답 시간

### Prometheus 설정 (docker/prometheus/prometheus.yml)

```yaml
scrape_configs:
  - job_name: 'customers-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['customers-service:8081']
  # ... 다른 서비스들
```

## 🛠️ 기술 스택

| 분류 | 기술 |
|------|------|
| Language | Java 17 |
| Framework | Spring Boot 3.x, Spring Cloud |
| Build | Maven |
| Container | Docker |
| Database | MySQL 8.0 (AWS RDS) |
| CI/CD | GitHub Actions |
| GitOps | ArgoCD + Kustomize |
| Monitoring | Prometheus + Grafana + Micrometer |
| AWS 인증 | OIDC (IRSA) |
| GCP 인증 | Workload Identity |

## 🔧 트러블슈팅

### Maven 빌드 실패
```bash
# 캐시 삭제 후 재빌드
./mvnw clean install -U -DskipTests
```

### Docker 빌드 시 메모리 부족
```bash
MAVEN_OPTS="-Xmx512m" ./mvnw package
```

### ECR Push 권한 오류
```bash
# OIDC Role에 ECR 권한 확인
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin {account}.dkr.ecr.ap-northeast-2.amazonaws.com
```

### Artifact Registry Push 권한 오류
```bash
# Workload Identity 바인딩 확인
gcloud iam service-accounts get-iam-policy github-actions@PROJECT_ID.iam.gserviceaccount.com
```

## 🔗 관련 저장소

| 저장소 | 설명 |
|--------|------|
| **petclinic-gitops** | 애플리케이션 GitOps (Kustomize, AWS/GCP overlay) |
| **platform-gitops-last** | 플랫폼 컴포넌트 (ArgoCD, External Secrets 등) |
| **platform-dev-last** | Terraform 인프라 (EKS, GKE, VPC) |
