---
title: "[AWS] 사이드 프로젝트 클라우드 인프라 구축기 (Phase 1) — VPC부터 CI/CD까지"
excerpt: "직접 진행 중인 사이드 프로젝트 RePLAN의 AWS 인프라를 처음부터 구축한 과정을 정리합니다. VPC 설계부터 EC2, RDS, ECR, GitHub Actions OIDC 기반 CI/CD까지 단계별로 기록했습니다."
categories:
  - Cloud
tags:
  - AWS
  - VPC
  - EC2
  - RDS
  - ECR
  - GitHub Actions
  - OIDC
  - Docker
  - CI/CD
  - 인프라
date: 2026-05-02 00:00:00 +0900
last_modified_at: 2026-05-02 00:00:00 +0900
toc: true
toc_sticky: true
author_profile: true
comments: true
---

> 사이드 프로젝트 **RePLAN**의 백엔드 서버를 AWS에 올리면서 인프라를 직접 구축한 과정을 기록한 글입니다.  
> 이론으로만 알던 VPC, EC2, RDS, ECR을 처음 손으로 만지면서 각 개념이 왜 필요한지 체감한 내용을 공유합니다.

---

## 전체 로드맵

Phase 1의 목표는 **Spring Boot 애플리케이션을 AWS EC2에 컨테이너로 배포하고, GitHub에 Push하면 자동으로 배포되는 CI/CD 파이프라인**을 구성하는 것입니다.

```
1. 계정 보안 설정
2. VPC + Subnet + IGW + Security Group 구성
3. EC2 띄우고 SSH 접속 확인
4. RDS 띄우고 EC2에서 접속 확인
5. ECR 레포지토리 생성
6. GitHub Actions + OIDC 설정
7. ECR + GitHub Actions CI/CD 연결
```

Phase 1 아키텍처는 아래와 같습니다.

![Phase 1 인프라 아키텍처](/assets/images/replan/infra_phase1.svg)

최종 목표 아키텍처(Phase 3)는 다음과 같습니다.

![최종 인프라 아키텍처](/assets/images/replan/infra_final.svg)

---

## 1. 계정 보안 설정

처음 AWS 계정을 만들면 반드시 세 가지를 먼저 해두는 것이 좋습니다.

**① 루트 계정 MFA 설정**

루트 계정은 모든 권한을 가진 계정입니다. 탈취될 경우 복구가 어렵기 때문에, 콘솔 접속 후 바로 MFA를 활성화해야 합니다.

`AWS 콘솔 → 우상단 계정명 → Security credentials → MFA 활성화`

**② IAM 유저 생성**

루트 계정으로 직접 작업하는 것은 권장되지 않습니다. `AdministratorAccess` 권한을 가진 IAM 유저를 별도로 만들고, 이후 모든 작업은 IAM 유저로 진행합니다.

**③ 결제 알람 설정**

사이드 프로젝트에서 가장 무서운 것은 의도치 않은 과금입니다. Budgets에서 월 예산 초과 시 이메일 알림을 설정해 두면 안심이 됩니다.

`Billing → Budgets → Create Budget → 월 $1 초과 시 이메일 알람`

---

## 2. VPC + Subnet + IGW + Security Group 구성

### VPC (Virtual Private Cloud)

AWS 위에 만드는 **나만의 가상 네트워크**입니다. 회사 서버실에 전용 네트워크를 구축하는 것처럼, AWS 위에 내 프로젝트만을 위한 격리된 네트워크 공간을 만드는 것입니다.

- `IPv4 CIDR`: VPC 안에서 사용할 사설 IP 범위 (예: `10.0.0.0/16`)
- `테넌시`: 기본값(Default)을 사용합니다. 전용 테넌시는 물리 서버를 독점하는 옵션으로 비용이 매우 높습니다.

### Subnet

VPC라는 큰 네트워크를 **용도별로 쪼갠 구역**입니다. 아파트 단지(VPC) 안의 동별 구역에 비유할 수 있습니다.

- `Availability Zone`: AZ는 AWS 리전 안의 물리적으로 분리된 데이터 센터입니다. 서울 리전(ap-northeast-2)에는 a, b, c, d 네 개가 있습니다.
- `IPv4 CIDR`: `10.0.0.0/16` VPC 범위 안에서 `/24`씩 잘라서 Subnet을 구성합니다. (예: `10.0.1.0/24`, `10.0.2.0/24`)
- Subnet 생성 후 **"Enable auto-assign public IPv4"** 옵션을 켜두면 EC2 생성 시 퍼블릭 IP가 자동으로 부여됩니다.
- RDS를 생성하려면 **최소 2개의 Subnet(서로 다른 AZ)**이 필요합니다.

### IGW (Internet Gateway)

VPC와 인터넷 사이의 **관문(게이트)**입니다. 아파트 단지의 정문 역할로, IGW 없이는 VPC 안의 EC2가 인터넷과 완전히 단절됩니다.

- IGW 생성 후 반드시 **Attach to VPC**를 해야 합니다.
- VPC 하나당 IGW는 하나만 연결할 수 있습니다.

### Route Table

Subnet이 "인터넷으로 나가는 트래픽은 IGW로 보내라"는 **경로 규칙**을 알아야 합니다. 네비게이션의 경로 설정에 비유할 수 있습니다.

1. Route Table에서 `0.0.0.0/0 → IGW` 경로를 추가합니다.
2. 해당 Route Table을 Subnet에 연결합니다.

### Security Group

EC2나 RDS 앞에 세우는 **가상 방화벽**입니다. 어떤 IP에서, 어떤 포트로 들어오는 트래픽을 허용할지 규칙을 정합니다.

EC2용 Security Group에는 아래 인바운드 규칙을 추가합니다.

| 유형 | 포트 | 소스 |
|------|------|------|
| SSH | 22 | 내 IP (운영 환경에서는 반드시 제한) |
| HTTP | 80 | 0.0.0.0/0 |
| HTTPS | 443 | 0.0.0.0/0 |

> **주의**: 개발 단계에서 편의를 위해 SSH를 `0.0.0.0/0`으로 열어두는 경우가 있는데, 실제 운영 환경에서는 반드시 접속 IP를 제한하거나 AWS Console의 **EC2 Instance Connect**를 활용해야 합니다.

---

## 3. EC2 띄우고 SSH 접속 확인

EC2를 생성하면서 발급받은 `.pem` 키 파일을 이용해 SSH로 접속합니다.

```bash
# 키 파일 권한 설정
chmod 400 my-key.pem

# SSH 접속
ssh -i "my-key.pem" ubuntu@<EC2_PUBLIC_IP>
```

### Docker & Docker Compose 설치

```bash
# 패키지 업데이트
sudo apt update && sudo apt upgrade -y

# Docker 공식 GPG 키 추가
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Docker 레포지토리 추가
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker 설치
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

`sudo` 없이 docker 명령어를 사용하려면:

```bash
sudo usermod -aG docker ubuntu
# 재접속 후 적용됨
```

설치 확인:

```bash
docker --version
docker compose version
```

---

## 4. RDS 띄우고 EC2에서 접속 확인

### RDS 생성 시 주의 사항

- **스토리지 자동 조정**: 비활성화 권장 (활성화 시 자동으로 용량이 늘어나며 과금될 수 있음)
- **성능 개선 도우미**: 비활성화 (사이드 프로젝트 단계에서는 불필요)

### RDS Security Group 설계 포인트

RDS용 Security Group의 인바운드 규칙에서 **소스를 IP 대신 EC2의 Security Group으로 지정**하는 것이 핵심입니다.

```
RDS Security Group 인바운드 규칙:
PostgreSQL(5432) | 소스: replan-ec2-sg (Security Group ID)
```

IP로 지정하면 EC2의 IP가 바뀔 때마다 수정해야 하지만, Security Group을 소스로 지정하면 해당 SG에 속한 EC2에서 오는 트래픽만 허용하므로 더 안전하고 유지보수가 쉽습니다.

### EC2에서 RDS 접속 확인

```bash
# PostgreSQL 클라이언트 설치
sudo apt install -y postgresql-client

# RDS 접속
psql -h <RDS_ENDPOINT> -U <DB_USER> -d <DB_NAME>
```

---

## 5. ECR 레포지토리 생성

ECR(Elastic Container Registry)은 AWS에서 제공하는 Docker 이미지 저장소입니다.

- **이미지 태그 변경 가능성**: `Mutable` 선택
  - 같은 태그로 이미지를 덮어쓸 수 있어 `latest` 태그나 커밋 해시 태그로 계속 업데이트하기 편합니다.

---

## 6. GitHub Actions + OIDC 설정

### 기존 방식 vs OIDC

기존에 많이 쓰던 방식은 `AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY`를 GitHub Secrets에 저장하는 것입니다. 하지만 이 방법은 장기 자격증명(long-lived credentials)을 외부에 보관한다는 보안 리스크가 있습니다.

**OIDC(OpenID Connect)** 방식은 GitHub Actions가 실행될 때 AWS에 임시 자격증명을 요청합니다. 키를 어딘가에 저장할 필요가 없어 훨씬 안전합니다.

```
GitHub Actions 실행
       │
  OIDC 토큰 발급
       │
  AWS STS에 임시 자격증명 요청
       │
  임시 자격증명 발급 (만료 시간 있음)
       │
  ECR push / EC2 배포
```

### AWS OIDC Provider 등록

`IAM → 자격 증명 공급자 → 공급자 추가`

- 공급자 유형: `OpenID Connect`
- 공급자 URL: `https://token.actions.githubusercontent.com`
- 대상(Audience): `sts.amazonaws.com`

### IAM Role 생성

- 신뢰할 수 있는 엔터티 유형: `웹 자격 증명`
- 자격 증명 공급자: `token.actions.githubusercontent.com`
- GitHub organization 및 Repository 지정

**권한 정책:**

| 정책 | 용도 |
|------|------|
| `AmazonEC2ContainerRegistryFullAccess` | ECR push/pull |
| `AmazonSSMReadOnlyAccess` | Parameter Store 읽기 |

### GitHub Secrets 설정

| Secret 이름 | 값 |
|-------------|-----|
| `AWS_ROLE_ARN` | 생성한 IAM Role의 ARN |
| `AWS_REGION` | `ap-northeast-2` |
| `ECR_REGISTRY` | `<AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com` |
| `ECR_REPOSITORY` | 레포지토리 이름 |
| `EC2_HOST` | EC2 퍼블릭 IP |
| `EC2_SSH_KEY` | `.pem` 파일 전체 내용 |

---

## 7. ECR + GitHub Actions CI/CD 연결

### deploy.yml

```yaml
name: Deploy to EC2

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # OIDC 토큰 발급에 필요
      contents: read

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        env:
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:$IMAGE_TAG .
          docker push ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:$IMAGE_TAG
          docker tag ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:$IMAGE_TAG \
                     ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:latest
          docker push ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:latest

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            aws ecr get-login-password --region ap-northeast-2 | \
              docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
            docker compose pull
            docker compose up -d
            docker image prune -f

      - name: Health check
        run: |
          sleep 30
          curl -f http://${{ secrets.EC2_HOST }}/health || exit 1
```

> 커밋 해시를 이미지 태그로 사용해, 어느 커밋에서 만들어진 이미지인지 추적할 수 있게 했습니다.

### docker-compose.yml (EC2)

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app
    restart: always

  app:
    image: ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
    environment:
      - DB_HOST=${DB_HOST}
      - DB_PORT=5432
      - DB_NAME=${DB_NAME}
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      - redis
    restart: always

  redis:
    image: redis:alpine
    restart: always
```

### nginx.conf

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /health {
        proxy_pass http://app:8080/actuator/health;
    }
}
```

### Dockerfile (Spring Boot)

```dockerfile
# 1단계: 빌드
FROM gradle:8.5-jdk17 AS build

WORKDIR /app

# 의존성 캐싱을 위해 gradle 파일 먼저 복사
COPY build.gradle settings.gradle ./
COPY gradle ./gradle
RUN gradle dependencies --no-daemon

COPY src ./src
RUN gradle bootJar --no-daemon

# 2단계: 실행
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -q -O- http://localhost:8080/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

멀티 스테이지 빌드를 사용해 빌드 도구(Gradle, JDK)는 최종 이미지에 포함되지 않으므로 이미지 크기를 크게 줄일 수 있습니다.

### Spring Boot Actuator 설정

Health check 엔드포인트를 위해 `spring-boot-starter-actuator`를 추가합니다.

**build.gradle:**
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

**application.yml:**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      show-details: never
```

### EC2에 IAM Role 연결

EC2에서 `aws ecr get-login-password` 명령어가 동작하려면, EC2 자체에도 IAM Role이 필요합니다.

`IAM → 역할 생성 → AWS 서비스 → EC2`

| 권한 | 용도 |
|------|------|
| `AmazonEC2ContainerRegistryReadOnly` | ECR 이미지 pull |
| `AmazonSSMReadOnlyAccess` | Parameter Store 읽기 |

생성 후 EC2 인스턴스에 Attach합니다.

---

## 마치며

Security Group 설계에서 소스에 IP 대신 SG를 참조하는 방식, 그리고 long-lived key 없이 임시 자격증명을 사용하는 OIDC 흐름이 특히 인상 깊었습니다.

### Phase 2 예정

- **SSM Parameter Store 마이그레이션**: `.env` 파일에 남아 있는 DB 접속 정보 등 환경변수를 AWS Parameter Store로 이전
- **CloudWatch 적용**: EC2 및 애플리케이션 로그 수집, 알람 설정
- **Terraform 적용**: 현재 콘솔로 구성한 인프라를 코드로 관리

### Phase 3 (Final) 예정

- **Route 53 적용**: 도메인 연결 및 DNS 관리
- **NAT 인스턴스 적용**: Private Subnet에 RDS를 배치하고 NAT를 통해 외부 통신 제어
- **Let's Encrypt 적용**: HTTPS 인증서 발급 및 Nginx SSL 설정
