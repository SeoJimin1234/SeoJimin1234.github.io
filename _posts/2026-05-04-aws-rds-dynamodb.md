---
title: "[AWS] RDS와 DynamoDB 핵심 개념 및 실습 정리"
excerpt: "AWS RDS(관계형 데이터베이스 서비스)와 DynamoDB(NoSQL)의 핵심 개념을 비교하고, PostgreSQL DB 인스턴스 생성 및 DynamoDB 테이블 실습 과정을 정리합니다."
categories:
  - Cloud
tags:
  - AWS
  - RDS
  - DynamoDB
  - PostgreSQL
  - NoSQL
  - 데이터베이스
  - 인프라
date: 2026-05-04 00:00:00 +0900
last_modified_at: 2026-05-04 00:00:00 +0900
toc: true
toc_sticky: true
author_profile: true
comments: true
---

> AWS Cloud Clubs (ACC EWHA 4기) 핸즈온 3회차 세션 내용을 바탕으로 정리한 글입니다.

---

## 데이터베이스란?

데이터베이스는 **구조화된 정보 또는 데이터의 조직화된 모음**으로, DBMS(Database Management System)에 의해 제어됩니다.

데이터베이스는 크게 두 가지 유형으로 나뉩니다.

| 구분 | Relational Database | Non-Relational Database |
|------|---------------------|-------------------------|
| 데이터 저장 방식 | 구조화된 데이터, 테이블 형태 | 비정형 데이터 |
| 스키마 | 엄격한 스키마 적용 | 스키마 없음 |
| 쿼리 언어 | SQL | 다양한 형태 |
| 특징 | 데이터 무결성, 관계 표현 | 대량 분산 데이터, 고속 처리 |
| 대표 예시 | MySQL, PostgreSQL | MongoDB, Redis |

---

## AWS RDS란?

### DB 운영 방식 비교

AWS에서 관계형 데이터베이스를 운영하는 방법은 세 가지가 있습니다.

| 방식 | 설명 |
|------|------|
| **On-Premise** | 사용자가 직접 서버를 구축해서 DB 관리 |
| **AWS EC2** | EC2 인스턴스 위에 사용자가 직접 DB를 설치하고 관리 |
| **AWS RDS** | AWS의 완전관리형 DB 서비스 |

**RDS(Relational Database Service)** 는 AWS 클라우드에서 관계형 데이터베이스를 더 쉽게 설치, 운영, 확장할 수 있는 웹 서비스입니다.

### RDS의 장점

- **간편한 관리** : 프로비저닝, 패치, 백업 등을 AWS가 자동으로 처리
- **가용성(Availability) 및 안정성(Durability)**
  - 자동 백업 vs. 스냅샷
  - Multi AZs 지원
- **보안성(Security)** : VPC 내 격리, 암호화 지원
- **확장성(Scalability)** : Read Replica를 통한 Scale Out
- **비용 효율성(Cost Effective)** : 사용한 만큼만 지불

---

## RDS 핵심 기능

### Multi AZ

Multi AZ는 고가용성을 위한 핵심 기능입니다.

- 데이터베이스의 복사본을 **다른 가용 영역(Availability Zone)** 에 자동으로 생성하고 동기화
- 복사본은 평소에는 **대기(Standby)** 상태로 유지
- 장애 감지 시 자동으로 대기 인스턴스로 **Failover(장애 조치)** 수행

```
[AZ 1] Primary DB  --동기복제-->  [AZ 2] Standby DB
        ↑                                   ↑
    읽기/쓰기                          장애 시 자동 전환
```

### Read Replica

Read Replica는 **읽기 전용 복제본**으로, 읽기 쿼리의 성능 향상과 분산 처리를 위해 사용합니다.

- **비동기적** 복제 방식으로 운영
- Read Replica 인스턴스를 추가해 **Scale Out** 구현
- 읽기 중심 워크로드(통계 조회, 리포팅 등)의 처리량 향상

**실제 시나리오**  
RDS를 사용하는 서비스에서 하루/한 달에 한 번씩 통계 데이터를 읽어오는 배치 작업으로 인해 성능 문제가 발생한다면, **Read Replica**를 추가해 메인 DB의 부하를 분리하는 것이 정답입니다.

---

## RDS 핸즈온 실습

### 아키텍처

실습 구성은 다음과 같습니다.

```
인터넷 → Internet Gateway → EC2(Public Subnet) → PostgreSQL RDS(Private Subnet)
```

EC2를 점프 서버(Bastion Host)처럼 활용해 Private Subnet에 있는 RDS에 접근하는 구조입니다.

### 1단계: EC2 인스턴스 생성

1. EC2 서비스 → **인스턴스 시작** 클릭
2. OS 이미지: **Amazon Linux 2023 AMI**, 인스턴스 유형: **t2.micro** (기본값 유지)
3. 키 페어: **키 페어 없이 계속 진행** (EC2 Instance Connect로 접속할 것이기 때문)
4. 네트워크 설정: SSH 트래픽 허용 → **내 IP** 로 제한
5. **인스턴스 시작** 클릭

### 2단계: PostgreSQL DB 인스턴스 생성

1. RDS 서비스 → **데이터베이스 생성** 클릭
2. 생성 방식: **손쉬운 생성(Easy Create)**
3. 구성 설정:
   - 엔진 유형: **PostgreSQL** (Aurora가 아닌 기본 PostgreSQL)
   - DB 인스턴스 크기: **프리 티어**
   - 자격 증명 관리: **자체 관리**
   - 암호: 자동 생성 또는 직접 입력
4. EC2 연결 설정: **EC2 컴퓨팅 리소스에 연결** → 앞서 만든 EC2 인스턴스 선택
5. **데이터베이스 생성** 클릭

> 암호 자동 생성을 선택한 경우, 생성 후 바로 **자격 증명 세부 정보 보기** 버튼을 클릭해 암호를 복사해두세요. 이후에는 다시 확인할 수 없습니다.

### 3단계: EC2에서 RDS 연결

EC2 인스턴스에 접속 후 psql을 설치하고 DB에 연결합니다.

```bash
# 최신 버전으로 업데이트
sudo dnf update -y

# psql 설치
sudo dnf install postgresql15

# DB 인스턴스에 연결
psql --host={endpoint} --port=5432 --dbname=postgres --username=postgres
```

DB 인스턴스의 엔드포인트는 **RDS 콘솔 → 해당 DB 인스턴스 → 연결 및 보안 탭**에서 확인할 수 있습니다.

아래와 같이 프롬프트가 나타나면 연결 성공입니다.

```
psql (15.6, server 16.2)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

postgres=> \d
```

### 4단계: 실습 후 리소스 삭제

비용 발생 방지를 위해 실습 후 반드시 리소스를 삭제하세요.

- EC2 인스턴스 종료
- RDS DB 인스턴스 삭제

> DB 인스턴스 삭제 시 **최종 스냅샷 생성**과 **자동 백업 보존** 옵션의 체크를 해제해야 완전히 삭제됩니다.

---

## AWS DynamoDB란?

### SQL vs NoSQL 비교

| 구분 | SQL | NoSQL |
|------|-----|-------|
| DB 유형 | 관계형 | 비관계형 |
| 스키마 | 정해진 스키마 (테이블) | 스키마 없음 |
| 데이터 형태 | 행과 열 (표 형태) | 문서, Key-Value, 그래프 등 |
| 특징 | 데이터 무결성, 복잡한 조인 | 가용성·확장성 높음, 고성능 |
| 대표 예시 | Oracle, MySQL, PostgreSQL | MongoDB, AWS DynamoDB |

NoSQL은 Instagram, Facebook 같이 **대용량 트래픽과 유연한 데이터 구조가 필요한 서비스**에서 널리 사용됩니다.

### DynamoDB 핵심 특징

Amazon DynamoDB는 **모든 규모에서 10밀리초 미만의 성능을 제공하는 서버리스 NoSQL 완전관리형 데이터베이스**입니다.

- **서버리스(Serverless)** : 서버 관리 불필요
- **완전관리형** : 장비 운영부터 DB 솔루션 설치 및 운영까지 AWS가 전담
- **높은 가용성과 내구성**
  - 대부분 10ms 이내 데이터 처리
  - 모든 데이터를 SSD에 저장, AWS 리전의 여러 AZ에 자동 복제
- **Auto-Scaling** : 요청 트래픽에 따라 읽기/쓰기 용량을 자동으로 조정

---

## DynamoDB 핵심 구성 요소

### 테이블(Table), 항목(Item), 속성(Attribute)

DynamoDB의 구성 요소는 RDBMS와 다음과 같이 대응됩니다.

| DynamoDB | RDBMS |
|----------|-------|
| Table | Table |
| Item | Row(행) |
| Attribute | Column(열) |

NoSQL의 특성상 **기본 키를 제외하면 스키마가 없습니다.** 각 Item은 서로 다른 속성을 가질 수 있으며, 중첩 속성(Nested Attribute)도 허용됩니다.

```json
// People 테이블의 Item 예시
{
    "PersonID": 101,
    "LastName": "Smith",
    "FirstName": "Fred",
    "Phone": "555-4321"
}

// 같은 테이블의 다른 Item (속성이 달라도 무방)
{
    "PersonID": 102,
    "LastName": "Jones",
    "FirstName": "Mary",
    "Address": {
        "Street": "123 Main",
        "City": "Anytown",
        "State": "OH",
        "ZIPCode": 12345
    }
}
```

### Partition Key

- RDBMS의 **Primary Key** 와 동일한 역할
- 테이블에서 **반드시 유일**해야 하는 값
- **파티션(Partition)** 을 결정하는 해시 값으로 사용됨
- DynamoDB는 테이블 크기가 10GB를 초과하면 자동으로 파티션을 분할
- 같은 Partition Key를 가진 Item은 같은 파티션에 저장됨
- **일치 검색(Equal)만 지원**

### Sort Key

- Partition Key로 파티션을 결정한 후, 같은 파티션 내에서 **Sort Key 값 기준으로 정렬**되어 저장
- 일치, 부등호, 포함(begins_with) 등 **범위 검색 지원**
- Partition Key + Sort Key를 합쳐 **복합 기본 키(Composite Primary Key)** 를 구성

```
Partition Key (Artist)  |  Sort Key (songTitle)
------------------------|-----------------------
NewJeans                |  Attention
NewJeans                |  ETA
NewJeans                |  Hype Boy
aespa                   |  Black Mamba
aespa                   |  Supernova
```

---

## DynamoDB 핸즈온 실습

### 1단계: NoSQL 테이블 생성

1. AWS 콘솔 → **DynamoDB** 검색 후 진입
2. 대시보드 → **테이블 생성** 클릭
3. 설정 값 입력:
   - **테이블 이름**: `Music`
   - **파티션 키(Partition Key)**: `Artist` (문자열)
   - **정렬 키(Sort Key)**: `songTitle` (문자열)
4. 테이블 설정: **설정 사용자 지정** 선택 (Auto Scaling 활성화를 위해)
5. 테이블 클래스: **DynamoDB Standard** 선택
6. 읽기/쓰기 용량:
   - 용량 모드: **프로비저닝**
   - Auto Scaling: **켜기** (최소 1, 최대 10, 목표 70%)

### 2단계: 데이터 추가

항목 탐색 → Music 테이블 선택 → **항목 생성** 클릭

| Artist (파티션 키) | songTitle (정렬 키) |
|-------------------|---------------------|
| NewJeans | Attention |
| NewJeans | ETA |
| NewJeans | Hype Boy |
| aespa | Black Mamba |
| aespa | Supernova |

### 3단계: 쿼리(Query) 실행

**Partition Key만 사용한 쿼리:**

- Artist = `aespa` 로 쿼리하면 aespa의 모든 곡이 반환됩니다.

```
반환된 항목 (2)
aespa | Black Mamba
aespa | Supernova
```

**Sort Key를 함께 사용한 쿼리:**

- Artist = `NewJeans`, songTitle이 `A`로 시작하는 항목 조회

```
반환된 항목 (1)
NewJeans | Attention
```

Sort Key를 이용하면 범위 기반 검색으로 원하는 데이터를 더 정밀하게 필터링할 수 있습니다.

---

## Lambda로 DynamoDB 대량 데이터 삽입하기

수동으로 항목을 하나씩 추가하는 것은 비효율적입니다. **AWS Lambda**를 활용하면 이 문제를 해결할 수 있습니다.

### BatchWriteItem API

DynamoDB는 **Batch 쓰기**를 지원합니다.

- **BatchWriteItem API**: 한 번에 최대 **25개**의 항목을 DynamoDB 테이블에 삽입 가능
- Lambda 함수에서 `batch.put_item`을 사용해 여러 데이터를 한 번에 처리

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Music')

with table.batch_writer() as batch:
    batch.put_item(Item={'Artist': 'BTS', 'songTitle': 'Dynamite'})
    batch.put_item(Item={'Artist': 'BTS', 'songTitle': 'Butter'})
    batch.put_item(Item={'Artist': 'IU', 'songTitle': 'Blueming'})
    # ... 최대 25개까지 한 번에 처리
```

### CloudWatch Events로 주기적 실행

**CloudWatch Events(EventBridge)** 를 활용하면 특정 시간 또는 주기적으로 Lambda 함수를 자동 실행할 수 있습니다. 예를 들어, 매일 자정 데이터 동기화 작업을 자동화할 수 있습니다.

### NoSQL에서 Join 처리

DynamoDB는 NoSQL 특성상 **Join을 기본 지원하지 않습니다.** Join이 반드시 필요한 경우, Lambda 함수 내에서 여러 테이블을 각각 조회하고 애플리케이션 레벨에서 데이터를 병합하는 방식으로 구현할 수 있습니다.

---

## 정리

| 항목 | AWS RDS | AWS DynamoDB |
|------|---------|--------------|
| 유형 | 관계형(SQL) | 비관계형(NoSQL) |
| 관리 방식 | 완전관리형 | 완전관리형(서버리스) |
| 스키마 | 엄격한 스키마 | 유연한 스키마 |
| 성능 | 복잡한 쿼리에 강함 | 단순 조회/대용량에 강함 |
| 확장 방법 | Read Replica, Multi AZ | Auto Scaling, 자동 파티셔닝 |
| 적합한 사용 사례 | 금융, ERP, 복잡한 관계 데이터 | SNS, 게임, IoT, 실시간 서비스 |

RDS는 **복잡한 관계형 데이터와 트랜잭션**이 필요한 경우에, DynamoDB는 **대규모 트래픽과 유연한 데이터 구조**가 필요한 경우에 적합합니다. 서비스의 특성과 요구사항에 맞는 데이터베이스를 선택하는 것이 중요합니다.
