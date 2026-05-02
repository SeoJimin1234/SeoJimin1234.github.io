---
title: "[AWS] GitHub Actions + AWS OIDC — 키 없이 배포하는 임시 자격증명"
excerpt: "AWS_ACCESS_KEY를 GitHub Secrets에 저장하지 않고, OIDC(OpenID Connect)를 이용해 GitHub Actions 실행 시 AWS 임시 자격증명을 발급받는 방식과 그 원리를 정리합니다."
categories:
  - Cloud
tags:
  - AWS
  - GitHub Actions
  - OIDC
  - IAM
  - CI/CD
  - 보안
date: 2026-05-02 00:00:00 +0900
last_modified_at: 2026-05-02 00:00:00 +0900
toc: true
toc_sticky: true
author_profile: true
comments: true
---

> 사이드 프로젝트 **RePLAN**의 CI/CD 파이프라인을 구성하면서, 기존 AWS Access Key 방식 대신 OIDC 기반 임시 자격증명으로 전환했습니다.
> 이 글에서는 왜 전환했는지, OIDC가 무엇인지, 실제로 어떻게 동작하는지를 정리합니다.

---

## 문제 인식 — Long-lived Key의 리스크

GitHub Actions에서 AWS 리소스(ECR, S3 등)에 접근하려면 자격증명이 필요합니다. 가장 흔하게 쓰이는 방법은 아래처럼 IAM Access Key를 GitHub Secrets에 저장하는 것입니다.

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ap-northeast-2
```

이 방식의 문제는 **장기 자격증명(long-lived credentials)**을 외부 시스템에 저장한다는 점입니다.

- 키가 **만료되지 않습니다.** 한 번 탈취되면 직접 무효화하기 전까지 계속 유효합니다.
- **키 로테이션**을 주기적으로 직접 해줘야 합니다.
- 팀이 커지면 Secrets에 접근 가능한 사람이 늘어나며, **키 관리가 복잡**해집니다.
- GitHub Secrets 자체는 안전하게 암호화되어 있지만, 결국 외부 시스템에 민감한 정보를 **위탁**하는 구조입니다.

---

## OIDC란 무엇인가

**OIDC(OpenID Connect)**는 OAuth 2.0 위에서 동작하는 인증 프로토콜입니다. 핵심은 **"A가 B에게 내가 누구인지 증명할 수 있도록 신뢰된 제3자(IdP)가 서명한 토큰을 발급해 주는 것"**입니다.

### 왜 GitHub이 JWT를 발급하는가

"GitHub Actions가 AWS에 접근 요청을 보낼 때, AWS는 그게 진짜 GitHub Actions에서 온 건지 어떻게 믿을 수 있을까?"라는 질문에서 시작합니다.

누군가 AWS에 HTTP 요청을 보내며 "나 GitHub Actions야"라고 말한다고 AWS가 믿을 수는 없습니다. **검증 가능한 증거**가 필요합니다.

JWT는 단순한 데이터 덩어리가 아닙니다. **GitHub의 개인키로 서명된 문서**입니다.

```
JWT = header.payload.signature

signature = GitHub_개인키로_서명(header + payload)
```

AWS는 OIDC 공급자 등록 시 GitHub의 **공개키**를 가져와 저장합니다. 이후 JWT를 받으면 공개키로 서명을 검증해 "이 토큰은 진짜 GitHub이 만든 것"임을 수학적으로 확인합니다. GitHub에 다시 물어볼 필요도 없습니다.

**왜 Access Key처럼 미리 공유하지 않나**

Access Key 방식은 결국 **공유된 비밀(shared secret)** 구조입니다. 키를 어딘가에 저장해야 하고, 저장된 곳이 뚫리면 끝입니다.

반면 비대칭 키 방식은 다릅니다.

```
GitHub: 개인키 보유 (외부 노출 없음)
AWS:    공개키 보유 (노출돼도 무관)
→ 키를 어디에도 "저장"할 필요가 없음
```

**여권**에 비유하면 이렇습니다. 정부(GitHub)가 서명해서 발급하고, 입국 심사관(AWS)은 공개된 방법으로 진위를 확인합니다. 심사관이 정부에 전화할 필요도 없고, 여권을 어딘가에 맡겨둘 필요도 없습니다. 게다가 여권에는 "이 사람은 ○○레포, main 브랜치에서 왔음"이라는 claim이 적혀 있어서 세밀한 접근 제어도 가능합니다.

GitHub Actions에서의 OIDC 흐름을 단순화하면 이렇습니다.

```
GitHub (IdP)         AWS (Service Provider)
    │                        │
    │  1. OIDC 토큰 발급      │
    │◄────────────────────── │ (Actions 워크플로우 실행)
    │                        │
    │  2. 토큰을 AWS STS에 제시
    │ ──────────────────────►│
    │                        │
    │  3. 토큰 서명 검증       │
    │  (GitHub의 공개키로)    │
    │                        │
    │  4. 임시 자격증명 발급   │
    │ ◄──────────────────────│
    │                        │
```

핵심 용어를 짚고 넘어가겠습니다.

| 용어 | 설명 |
|------|------|
| **IdP (Identity Provider)** | 신원을 증명해 주는 발급자. 여기서는 GitHub |
| **OIDC 토큰 (JWT)** | GitHub이 서명한 JSON Web Token. 워크플로우 정보(레포, 브랜치, 커밋 등)를 claim으로 포함 |
| **STS (Security Token Service)** | AWS에서 임시 자격증명을 발급해 주는 서비스 |
| **AssumeRoleWithWebIdentity** | STS API. 외부 IdP의 토큰을 받아 IAM Role을 임시로 위임해 줌 |

---

## 동작 원리 상세

### 1단계 — GitHub이 OIDC 토큰 발급

GitHub Actions 워크플로우가 실행되면, GitHub의 OIDC 공급자(`token.actions.githubusercontent.com`)가 해당 실행 컨텍스트를 담은 **JWT 토큰**을 발급합니다.

이 토큰의 payload(claim)에는 아래와 같은 정보가 담깁니다.

```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "sub": "repo:myorg/myrepo:ref:refs/heads/main",
  "aud": "sts.amazonaws.com",
  "repository": "myorg/myrepo",
  "ref": "refs/heads/main",
  "sha": "abc123...",
  "workflow": "Deploy to EC2",
  "actor": "seojimin"
}
```

- `iss`: 이 토큰을 발급한 주체 (GitHub OIDC 공급자)
- `sub`: 이 토큰의 주인공 (어떤 레포의 어떤 브랜치에서 실행됐는지)
- `aud`: 이 토큰을 소비할 대상 (AWS STS)

### 2단계 — AWS STS에 임시 자격증명 요청

GitHub Actions의 `aws-actions/configure-aws-credentials` 액션이 위 JWT를 들고 AWS STS의 `AssumeRoleWithWebIdentity` API를 호출합니다.

```
POST https://sts.amazonaws.com/
Action=AssumeRoleWithWebIdentity
&RoleArn=arn:aws:iam::123456789012:role/GitHubActionsRole
&WebIdentityToken=<JWT>
&RoleSessionName=GitHubActions
```

### 3단계 — AWS가 토큰 유효성 검증

AWS는 토큰을 검증하기 위해 두 가지를 확인합니다.

1. **서명 검증**: GitHub OIDC 공급자의 공개키(JWKS)로 JWT 서명이 유효한지 확인합니다.
2. **신뢰 정책(Trust Policy) 검증**: IAM Role에 설정된 조건과 토큰의 `sub` claim이 일치하는지 확인합니다.

IAM Role의 신뢰 정책은 아래와 같이 생겼습니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

`Condition`의 `sub` 조건이 핵심입니다. `myorg/myrepo`의 `main` 브랜치에서 실행된 Actions만 이 Role을 위임받을 수 있습니다. fork된 레포나 다른 브랜치에서 실행된 Actions은 조건에 맞지 않아 거부됩니다.

### 4단계 — 임시 자격증명 발급 및 사용

모든 검증을 통과하면 STS는 아래 형태의 임시 자격증명을 반환합니다.

```json
{
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "...",
  "SessionToken": "...",
  "Expiration": "2026-05-02T12:30:00Z"
}
```

이 자격증명은 기본 **1시간 후 자동 만료**됩니다. GitHub Actions 워크플로우 실행 시간 내에만 유효하므로, 탈취되더라도 짧은 시간 내에 자동으로 무효화됩니다.

---

## Long-lived Key vs OIDC 비교

| 항목 | Long-lived Key | OIDC |
|------|---------------|------|
| 자격증명 저장 위치 | GitHub Secrets (외부) | 없음 (매 실행마다 발급) |
| 만료 | 없음 (직접 삭제해야 함) | 기본 1시간 자동 만료 |
| 키 로테이션 | 수동 필요 | 불필요 |
| 탈취 시 피해 범위 | 무기한 | 만료까지 최대 1시간 |
| 접근 제어 세분화 | 어려움 | 레포·브랜치·환경 단위 가능 |

---

## AWS 설정 방법

### OIDC 공급자 등록

`IAM → 자격 증명 공급자 → 공급자 추가`

| 항목 | 값 |
|------|-----|
| 공급자 유형 | OpenID Connect |
| 공급자 URL | `https://token.actions.githubusercontent.com` |
| 대상(Audience) | `sts.amazonaws.com` |

이 단계에서 AWS는 GitHub OIDC 공급자의 공개키를 가져와 등록합니다. 이후 토큰 검증 시 이 공개키를 사용합니다.

### IAM Role 생성

`IAM → 역할 → 역할 만들기 → 웹 자격 증명`

- **자격 증명 공급자**: `token.actions.githubusercontent.com`
- **Audience**: `sts.amazonaws.com`
- **GitHub 조직/레포/브랜치**: 허용할 범위 지정

필요한 권한 정책을 연결합니다. 최소 권한 원칙에 따라 필요한 것만 부여합니다.

```
예시)
AmazonEC2ContainerRegistryFullAccess  →  ECR push/pull
AmazonSSMReadOnlyAccess               →  Parameter Store 읽기
```

Role 생성 후 **ARN을 복사**해서 GitHub Secrets에 `AWS_ROLE_ARN`으로 저장합니다.

---

## GitHub Actions 워크플로우 설정

```yaml
name: Deploy to EC2

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # OIDC 토큰 발급 권한 (필수)
      contents: read

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}

      # 이후 단계에서 임시 자격증명으로 AWS 리소스에 접근
      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2
```

`permissions.id-token: write`가 핵심입니다. 이 설정이 없으면 GitHub Actions가 OIDC 토큰을 발급받을 수 없습니다.

`role-to-assume`에 `aws-access-key-id` 대신 IAM Role ARN을 지정하는 것만으로 전환이 완료됩니다.

---

## 신뢰 정책 조건 세분화

`sub` claim을 이용하면 접근을 세밀하게 제한할 수 있습니다.

```json
// 특정 레포의 특정 브랜치만 허용
"token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"

// 특정 레포의 모든 브랜치 허용 (와일드카드 불가, StringLike 사용)
"StringLike": {
  "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:*"
}

// GitHub Environments를 사용하는 경우
"token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:environment:production"
```

`StringLike`를 사용하면 와일드카드(`*`)를 쓸 수 있어 유연하게 조건을 설정할 수 있습니다.

> **주의**: `sub` 조건을 너무 넓게 열면 의도치 않은 Actions 실행이 AWS 리소스에 접근할 수 있습니다. Pull Request에서 실행되는 Actions도 포함될 수 있으므로, 가능하면 브랜치나 Environment 단위로 좁게 설정하는 것이 좋습니다.

---

## 마치며

OIDC 방식을 도입하면서 AWS Access Key를 어디에도 저장하지 않아도 된다는 것이 가장 큰 장점으로 느껴졌습니다. 키 로테이션 같은 운영 부담도 사라지고, 임시 자격증명이라 탈취되더라도 피해가 제한적입니다.

설정 과정도 생각보다 간단했습니다. OIDC 공급자 등록, IAM Role 생성, 워크플로우에서 `id-token: write` 권한 추가가 전부입니다.

장기 자격증명을 외부에 보관하는 방식이라면 OIDC 전환을 적극 권장합니다.
