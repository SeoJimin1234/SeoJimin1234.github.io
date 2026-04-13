---
title: "[AWS] 클라우드 스토리지, Amazon S3, CloudFront 핵심 개념 정리"
excerpt: "ACC EWHA 세션을 바탕으로 클라우드 스토리지의 종류(File/Block/Object Storage), Amazon S3의 구조와 보안, Amazon CloudFront CDN의 동작 방식까지 핵심 개념을 정리했습니다."
categories:
  - Cloud
tags:
  - AWS
  - S3
  - CloudFront
  - Storage
  - 인프라
date: 2026-04-13 00:00:00 +0900
last_modified_at: 2026-04-13 00:00:00 +0900
toc: true
toc_sticky: true
author_profile: true
comments: true
---

> AWS Cloud Clubs (ACC EWHA 4기) 세션 내용을 바탕으로 정리한 글입니다.

---

## 클라우드 스토리지란?

클라우드 스토리지는 클라우드 컴퓨팅 제공업체를 통해 **데이터와 파일을 인터넷에 저장할 수 있는 클라우드 컴퓨팅 모델**입니다.  
사용자는 퍼블릭 인터넷 또는 전용 프라이빗 네트워크 연결을 통해 스토리지에 액세스할 수 있습니다.

### 클라우드 스토리지의 장점

- **비용 효율성** : 물리 장비 구매 없이 사용한 만큼만 비용 지불
- **민첩성 향상** : 빠른 프로비저닝으로 개발 속도 증가
- **더 빠른 배포** : 필요한 시점에 즉시 스토리지 확보
- **효율적인 데이터 관리** : 중앙화된 관리 및 자동화 지원
- **확장성** : 수요에 따라 유연하게 용량 확장/축소

---

## 클라우드 스토리지의 종류

클라우드 스토리지는 크게 세 가지 유형으로 나뉩니다.

### 1. Block Storage

**물리적 하드웨어(HDD, SSD)와 비슷한 역할**을 하는 스토리지입니다.

- 데이터를 **Block 단위**로 Read/Write
- 빠른 I/O를 위해 각 Block에 **고유한 식별자(ID#)** 부여
- 백업 기능 제공 → **SAN(Storage Area Network)** 역할 수행 가능
- AWS 서비스: **Amazon EBS (Elastic Block Store)**

> Block Storage는 OS, 데이터베이스처럼 낮은 지연 시간과 높은 성능이 필요한 워크로드에 적합합니다.

---

### 2. File Storage

**데이터를 파일 단위의 계층 구조로 저장**하는 스토리지입니다.

- 애플리케이션에 가장 **널리 사용되는 유형**
- **NFS, NAS** 기술 활용 가능
- 로컬 하드 드라이브와 유사한 접근 방식 (`/home/users/data/name.txt`)
- AWS 서비스: **Amazon EFS (Elastic File System)**

**사용 사례**

- 여러 서버나 사용자가 동일한 파일 시스템에 **동시 접근**하는 경우
- 파일과 디렉토리 구조로 데이터를 관리하고 싶은 경우
- 네트워크를 통한 파일 공유가 필요한 경우

---

### 3. Object Storage

**대용량 미디어 파일, 이미지, 백업 등의 비정형 데이터**를 저장하기 위한 스토리지입니다.

- 전송된 형식 그대로를 **객체(Object)** 데이터로 저장
- 사용자가 직접 **메타데이터** 지정 가능
- 객체는 **보안 버킷(Bucket)** 이라는 저장 공간에 보관
- **HTTP 프로토콜 기반 REST API** 호출을 통해 접근
- AWS 서비스: **Amazon S3**

**사용 사례**

- 정적 웹 컨텐츠(이미지, 비디오, HTML)를 호스팅하는 경우
- 전 세계 여러 데이터센터에 데이터를 분산시키는 경우
- 저렴한 비용으로 데이터를 오래 보관하는 경우

---

## Amazon S3 (Simple Storage Service)

Amazon S3는 **데이터를 버킷 내 객체로 저장하는 객체 스토리지 서비스**입니다.

- 파일을 하나의 객체로 저장 → **Key**를 통해 관리
- **확장성**: 저장 용량 제한 없음
- **데이터 보호**: 버전 관리, 암호화, 복제 기능 제공
- **비용 효율성**: 사용한 만큼만 비용 지불
- 비즈니스 요구에 맞게 **ACL, IAM 정책**으로 접근 제어

### S3 구성: 버킷과 객체

S3는 **버킷(Bucket)** 과 **객체(Object)** 로 구성됩니다.

| 구분 | 설명 |
|------|------|
| 버킷 (Bucket) | 최상위 디렉토리, 객체의 컨테이너 |
| 객체 (Object) | 버킷 내에 저장되는 파일 |

---

### 버킷 (Bucket)

버킷은 S3에 저장된 객체에 대한 컨테이너로, 폴더에 비유할 수 있습니다.

- 버킷에는 객체를 **무제한으로 저장** 가능
- 한 AWS 계정당 **최대 100개의 버킷** 생성 가능
- AWS 전역에서 **단 하나만 존재**, 리전과 관계없이 **전역적으로 유일한 이름** 사용

---

### 객체 (Object)

객체는 S3에 저장되는 기본 단위로, 파일에 비유할 수 있습니다.

- 각 객체의 크기는 **최대 5TB**
- **객체 데이터(Data)** 와 **메타데이터(Metadata)** 로 구성
- **Key** 와 **버전 ID(Version ID)** 로 버킷 내에서 고유하게 식별

**객체 구성요소**

| 요소 | 설명 |
|------|------|
| Key | 객체 이름 (경로 포함, ex. `/img/cat.png`) |
| Value | 실제 데이터 |
| Version ID | 버전 관리 활성화 시 부여되는 고유 ID |
| Metadata | 최종 수정일, 파일 타입, 소유자, 사이즈 등 |
| 태그 | 접근 제어 및 분류 목적 |
| 액세스 제어 정보 | 해당 객체에 대한 권한 설정 |

**객체 URL 형식**

```
https://{버킷이름}.s3.{리전}.amazonaws.com/{Key이름}
```

예시: `https://awsinaction.s3.ap-southeast-2.amazonaws.com/img/cat.png`

**태그를 이용한 접근 제어 예시**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Principal": { "AWS": ["arn:aws:iam::111122223333:role/JohnDoe"] },
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion"],
      "Resource": "arn:aws:s3:::amzn-s3-demo-bucket/*",
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/environment": "production"
        }
      }
    }
  ]
}
```

> `environment: production` 태그가 있는 객체에만 접근을 허용하는 정책 예시입니다.

---

## Amazon S3 보안

### 데이터 암호화

#### 서버 측 암호화 (SSE)

서버가 데이터를 **자동으로 보호**해주는 기능입니다.

- **업로드 시**: AWS가 데이터를 대신 암호화하여 저장
- **다운로드 시**: 자동으로 복호화하여 원래 상태로 반환
- 종류: `SSE-S3`, `SSE-KMS`, `DSSE-KMS`, `SSE-C`
- 현재는 **SSE-S3**가 모든 S3 버킷의 기본 암호화 수준으로 적용

요청 헤더 설정:
```
"x-amz-server-side-encryption": "AES256"
```

#### 클라이언트 측 암호화

전송 및 저장 시 보안을 보장하기 위해 **로컬에서 직접 데이터를 암호화**합니다.

- AWS를 포함한 제3자에게 객체 노출 방지 가능
- S3 암호화 클라이언트가 사용자와 S3 간 **중개자 역할** 수행
- 높은 보안을 보장하지만 **관리 및 구현 복잡성 증가**

---

### 액세스 제어

#### IAM (Identity and Access Management)

- **사용자/역할 기준** → "누가 무엇을 할 수 있나?"
- 자신의 AWS 계정 내 `IAM User`, `IAM Group`, `IAM Role`에 적용
- 출발지 IP 주소나 도메인명에 의한 액세스 제어도 가능

#### ACL (Access Control List)

- **계정 기준** → "어떤 계정이 접근할 수 있나?"
- AWS 계정 단위로 액세스 권한 설정
- 다른 AWS 계정에 대해 객체 또는 버킷의 읽기/쓰기 허용

#### 버킷 정책 (Bucket Policy)

- **버킷/객체 기준** → "이 버킷에 누가 접근할 수 있나?"
- 단일 S3 버킷 내 모든 객체에 대한 권한을 세부적으로 구성
- 자신의 계정 IAM 사용자 및 다른 AWS 계정 사용자 모두에 적용 가능
- IP 주소, 도메인명 기반 액세스 제어도 가능

#### CORS (Cross-Origin Resource Sharing)

- 교차 출처 리소스 공유: **서로 다른 출처 간의 리소스 공유 허용 정책**
- `Access-Control-Allow-Origin` 헤더를 통해 허용 출처 지정
- 웹 브라우저가 Preflight Request로 서버에 허가 여부를 먼저 확인

#### Pre-signed URL

- S3 접근 권한이 없는 이용자를 대상으로 **임시 URL 발급**
- 일정 시간이 지나면 자동으로 public 접근 불가 (timeout)
- URL 생성자가 유효한 보안 자격 증명과 해당 작업 권한을 보유해야 함

---

### 스토리지 관리

#### 버전 관리 (Versioning)

- 한 버킷에 **여러 버전의 객체** 보관
- 실수로 삭제되거나 덮어써진 객체 복원 가능
- 버전 관리 활성화 시 각 객체에 **고유한 Version ID 자동 생성**
- **한 번 활성화하면 비활성화 불가** (일시중단은 가능)
- 각 버전은 독립적으로 존재

#### 객체 복제 (Replication)

하나 이상의 대상 버킷에 대한 객체를 **비동기적으로 자동 복제**합니다.

| 유형 | 설명 | 사용 사례 |
|------|------|----------|
| SRR (Same-Region Replication) | 동일 리전 내 버킷 간 복제 | 로그 집계, 계정 간 라이브 복제, 액세스 제어 |
| CRR (Cross-Region Replication) | 서로 다른 리전 버킷 간 복제 | 대기시간 최소화, 운영 효율 향상 |
| S3 배치 복제 | 기존 객체를 온디맨드로 다른 버킷에 복제 | 대량 기존 데이터 마이그레이션 |

#### 스토리지 클래스 (Storage Class)

접근 빈도에 따라 적절한 스토리지 클래스를 선택해 비용을 최적화할 수 있습니다.

| 클래스 | 특징 | 사용 시기 |
|--------|------|----------|
| S3 Standard | 높은 내구성, 가용성, 성능 | 자주 접근하는 데이터 |
| S3 Intelligent-Tiering | 접근 패턴에 따라 자동으로 티어 이동 | 접근 패턴이 불확실한 경우 |
| S3 Standard-IA | 저비용, 빠른 조회 가능 | 가끔 접근하지만 빠른 조회가 필요한 경우 |
| S3 One Zone-IA | Standard-IA보다 저렴, 단일 AZ 저장 | 복구 가능하며 비용을 줄이고 싶은 경우 |
| S3 Glacier Instant Retrieval | 아카이브 저장, 즉시 조회 가능 | 거의 안 쓰지만 즉시 조회가 필요한 경우 |
| S3 Glacier Flexible Retrieval | 몇 분 ~ 몇 시간 내 조회 | 몇 분~몇 시간 기다려도 되는 경우 |
| S3 Glacier Deep Archive | 최저 비용 | 거의 접근하지 않는 장기 보관 데이터 |

#### 수명 주기 (Lifecycle)

객체를 **효율적으로 저장/관리**하기 위해 S3 객체 그룹에 적용할 작업을 정의하는 구성입니다.

- **전환 작업**: 객체가 다른 스토리지 클래스로 전환되는 시기 정의
  - 예: 30일 지나면 S3 Standard-IA로 이동, 60일 지나면 S3 Glacier Instant Retrieval로 이동
- **만료 작업**: 객체가 만료되는 시기 정의 → 만료되면 S3가 해당 객체 자동 삭제

---

## Amazon CloudFront

### CloudFront란?

Amazon CloudFront는 **CDN(Content Delivery Network, 글로벌 콘텐츠 전송 네트워크)** 서비스입니다.

- 짧은 지연 시간과 빠른 전송 속도로 데이터, 동영상, 애플리케이션 및 API를 전 세계 고객에게 안전하게 전송
- 기본 보안 기능(DDoS 방어 등) 제공

### CDN이란?

CDN은 **콘텐츠를 효율적으로 전달하기 위해 여러 서버에 데이터를 분산 캐싱**하는 네트워크입니다.

- 원본 서버의 위치와 상관없이, **사용자에게 더 가까운 서버**로부터 캐싱된 콘텐츠 배포
- 예: 영국에 있는 웹사이트에 미국 사용자가 접근할 때 → 미국 근처 캐시 서버에서 배포

---

### Edge Location

CloudFront가 콘텐츠를 제공하는 방식의 핵심입니다.

- 사용자에게 더 **가까운 지리적 위치**에 콘텐츠를 저장하는 Origin Server의 캐싱 서버
- 사용자 요청 발생 시 → **지연 시간이 가장 적은 엣지 로케이션**으로 라우팅

### Regional Edge Caches (REC)

- **Origin Server와 Edge Location 사이에 존재**하는 중간 캐시 계층
- Edge Location에서 캐시하지 않는 콘텐츠 저장
- Edge Location에서 **캐시 Miss** 발생 시 → Origin Server 대신 **REC으로 요청** → 빠른 응답

---

### CloudFront 동작 방식

```
1. 클라이언트 → Edge Server로 콘텐츠 요청
2. Edge Server: 해당 데이터 캐싱 여부 확인
   ├── 캐시 Hit: 캐싱된 콘텐츠를 바로 사용자에게 응답
   └── 캐시 Miss: Origin Server로 포워딩
                  → Edge Server에 캐싱 데이터 생성
                  → 사용자에게 응답
```

---

### Origin Server

CloudFront에서 **데이터의 원본이 위치한 곳**으로, 정적/동적 콘텐츠를 모두 처리합니다.

| 콘텐츠 유형 | 설명 | Origin |
|------------|------|--------|
| 정적 콘텐츠 | EC2 서버가 불필요한 콘텐츠 (이미지 등) | S3 Bucket |
| 동적 콘텐츠 | 서버가 필요한 콘텐츠 (로그인 정보, 실시간 데이터 등) | EC2 Instance, ELB |

> 동적 콘텐츠를 정적 캐싱하면 **TTL(Time To Live)** 시간 동안 사용자는 새로 추가/수정된 데이터를 볼 수 없습니다.

---

### CloudFront 주요 기능

#### 1. HTTPS 지원

- Origin에서 HTTPS를 지원하지 않아도 CloudFront가 **HTTPS 통신을 자동 처리**
- 인증키 설치 등 복잡한 작업 불필요 → 간편!

#### 2. 지리적 접근 제한 (Geo Restriction)

- 특정 국가/지역의 사용자에 대해 콘텐츠 접근 제한 가능
- 예: 라이선스 계약이나 회사 정책에 따라 특정 지역 차단

#### 3. Signed URL

- S3 Pre-signed URL과 유사한 기능
- **허용된 사용자에게만** 접근할 수 있는 서명된 URL 제공
- 단, **하나의 콘텐츠에만** 사용 가능
- 예: 유료 결제를 한 사용자에게만 특정 동영상 URL 제공
- 동작 흐름: 접근 허용 사용자 정의 → Signed URL 생성 → 사용자 검증 → URL 제공

#### 4. Signed Cookie

- **다수의 파일에 대한 액세스**를 하나의 Signed Cookie로 제공
- 만료 시간, IP 주소 범위 등 세부 조건 지정 가능
- 예: ID/PW로 로그인한 유료 회원에게 모든 유료 콘텐츠 제공
- 동작 흐름: 로그인 → 인증 정보 기반 Cookie 발급 → Cookie 포함 요청 시 인증 완료

---

## 정리

| 서비스 | 유형 | 핵심 특징 |
|--------|------|----------|
| Amazon EBS | Block Storage | EC2 인스턴스에 연결하는 고성능 블록 스토리지 |
| Amazon EFS | File Storage | NFS 기반 공유 파일 스토리지, 다중 EC2 동시 접근 |
| Amazon S3 | Object Storage | 무제한 확장, 버킷/객체 구조, 다양한 보안·관리 기능 |
| Amazon CloudFront | CDN | 엣지 로케이션 기반 전 세계 빠른 콘텐츠 배포 |

AWS의 스토리지 서비스는 각각의 특성과 사용 목적이 명확히 구분되어 있습니다.  
워크로드 특성에 맞는 스토리지를 선택하고, S3의 스토리지 클래스와 수명 주기 정책을 적절히 활용하면 **비용 효율적이고 안전한 데이터 관리**가 가능합니다.  
CloudFront와 S3를 함께 사용하면 정적 콘텐츠를 전 세계 사용자에게 빠르고 안전하게 제공할 수 있습니다.
