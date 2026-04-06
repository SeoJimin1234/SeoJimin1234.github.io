---
title: "[AWS] VPC와 Route 53 핵심 개념 정리"
excerpt: "ACC EWHA 핸즈온 세션을 바탕으로 VPC, Subnet, IGW, Route Table, NAT Gateway, NACL, VPC Peering, VPC Endpoint, Route 53까지 AWS 네트워킹 핵심 개념을 정리했습니다."
categories:
  - Cloud
tags:
  - AWS
  - VPC
  - Route53
  - 네트워크
  - 인프라
date: 2026-04-06 00:00:00 +0900
last_modified_at: 2026-04-06 00:00:00 +0900
toc: true
toc_sticky: true
author_profile: true
comments: true
---

> AWS Cloud Clubs (ACC EWHA 4기) 핸즈온 세션 내용을 바탕으로 정리한 글입니다.

---

## IP (Internet Protocol)

인터넷에 연결된 모든 장치를 식별하기 위해 각 장비에 부여되는 **고유한 주소**입니다.
IPv4 기준으로 총 32비트로 구성됩니다. (예: `255.255.255.255`)

### 분류

| 구분 | 설명 |
|------|------|
| 고정 IP | 변하지 않는 IP |
| 유동 IP | 변경될 수 있는 IP |
| 공인 IP | 인터넷에서 사용하는 주소 |
| 사설 IP | 내부 네트워크에서 사용하는 주소 |

> **확인 방법**
> 공인 IP: https://ip.pe.kr/
> 사설 IP: `ipconfig` 명령어

---

## CIDR (Classless Inter-Domain Routing)

IP 주소는 **네트워크 ID + 호스트 ID** 로 구성됩니다.

- **네트워크 ID**: 네트워크를 구분하는 부분 (큰 단위)
- **호스트 ID**: 호스트를 개별적으로 구분하는 부분

CIDR는 이 두 영역의 범위를 표기하는 방식입니다.

```
192.168.0.0/24
```

위 표기는 IP 주소 중 **앞 24비트를 네트워크 ID**로 사용하고, 나머지 8비트를 호스트 ID로 사용한다는 의미입니다.
→ 2⁸ = **256개의 IP 주소** 사용 가능

> AWS에서는 `/16 ~ /28` 범위의 서브넷 마스크만 허용합니다.

---

## DNS (Domain Name System)

도메인 이름(`aws.amazon.com`)을 IP 주소(`192.168.1.0`)로 변환하거나, 그 반대 역할을 수행하는 **분산형 데이터베이스 시스템**입니다.

DNS가 분산형인 이유는 `Root DNS → TLD DNS → Authoritative DNS` 순서로 질의하며, 각 서버가 일부 정보만 담당하기 때문입니다.

### 통신 흐름
```
DHCP → ARP → DNS → TCP → HTTP/HTTPS
```

### DNS 레코드 종류

DNS 서버가 요청을 받았을 때 어떻게 응답할지 정의한 규칙입니다.
기본 형식: `RR(Name, Value, Type, TTL)`

| 레코드 | 설명 |
|--------|------|
| **A** | 도메인 이름을 IPv4 주소에 매핑 |
| **CNAME** | 하나의 도메인을 다른 도메인 이름에 매핑 |
| **NS** | 해당 도메인을 관리하는 네임서버를 나타냄 |
| **Alias** | Route 53 특수 레코드, AWS 리소스와 연결 시 사용 |

---

## Route 53

Route 53은 AWS의 DNS 서비스로, 단순한 DNS를 넘어 다양한 기능을 제공합니다.

- **DNS (네임서버)**: IP와 도메인 연결
- **Health Check**: 포트 모니터링
- **Failover**: L4 기능
- **라우팅 정책**: GSLB 역할

> 라우팅이란 네트워크에서 경로를 찾는 행위입니다.

### 라우팅 정책 종류

| 정책 | 설명 |
|------|------|
| 단순 | 표준 DNS 기능 |
| 가중치 기반 | 각 리소스에 보낼 트래픽을 비율로 지정 |
| 지리적 위치 | 사용자 위치 기반 라우팅 |
| 지리적 근접 | 리소스 위치 기반 라우팅 |
| 지연 시간 | 지연시간이 가장 짧은 리전으로 라우팅 |
| 장애조치 (Failover) | 정상 상태일 때만 트래픽 라우팅 |
| 다중 응답 | 무작위 트래픽 라우팅 |

---

## VPC (Virtual Private Cloud)

### VPC가 등장한 이유

VPC 개념이 없던 시절에는 클라우드 리소스들을 격리할 방법이 없었습니다. EC2 인스턴스 하나를 추가하려면 모든 리소스 관계를 수정해야 하는 번거로움이 있었습니다.

2011년, **나만의 독립된 가상 네트워크망**인 VPC가 등장했습니다. VPC 단위로 네트워크를 구성하면서 시스템 의존도가 낮아지고 유지보수가 훨씬 편해졌습니다.

### VPC 특징

- **리전 단위**로 생성되며, 리전 당 최대 5개 생성 가능
- VPC 당 최대 5개의 CIDR 설정 가능
- **사설 IPv4 범위**만 할당 가능
- VPC CIDR는 다른 VPC나 네트워크와 **겹치면 안됨**
- AWS 계정 생성 시 **Default VPC**가 자동으로 생성됨

---

## Subnet

VPC의 IP 주소를 나눠 리소스가 배치되는 **물리적인 주소 범위**입니다. IP 네트워크의 논리적 영역을 쪼개서 만든 하위 네트워크망으로, 이 과정을 **Subnetting**이라고 합니다.

- **Public Subnet**: 인터넷과 직접 통신 가능
- **Private Subnet**: 내부망에서만 통신

VPC가 하나의 리전에 존재하는 것처럼, **서브넷은 하나의 AZ(가용영역)에만 존재**합니다.

### VPC를 왜 쪼개야 할까?

IP 주소를 **낭비 없이 효율적으로 사용**하기 위해서입니다.
예를 들어 `192.168.10.0/24` (256개 주소)를 실제 50개만 필요한 네트워크에 사용하면 낭비가 생기므로, 필요한 크기로 나눠 사용합니다.

---

## Internet Gateway (IGW)

VPC는 기본적으로 격리된 네트워크 환경이라 리소스들이 인터넷과 연결되어 있지 않습니다.
IGW는 **VPC의 리소스를 인터넷에 연결하는 통로** 역할을 합니다.

> IGW를 생성했다고 바로 인터넷에 연결되지는 않습니다. IGW와 Subnet을 연결하는 Route Table 설정이 추가로 필요합니다.

---

## Route Table

서브넷 또는 게이트웨이의 네트워크 트래픽이 **어디로 전송되어야 할지** 정의한 라우팅 규칙 모음입니다. 트래픽의 이정표 역할을 합니다.

```
[VPC 내부 리소스]
       ↓
[Route Table]
Destination: 0.0.0.0/0
Target: IGW
       ↓
[Internet Gateway]
       ↓
[Internet]
```

각 서브넷에는 Route Table이 연결됩니다.

| 서브넷 | Route Table 설정 |
|--------|-----------------|
| Public Subnet | Target이 IGW → 인터넷 연결 가능 |
| Private Subnet | Target이 Local → 내부 통신만 가능 |

---

## Private Subnet이 인터넷과 통신하는 방법

Private Subnet에 있는 리소스가 인터넷에 접근해야 하는 상황이 생길 수 있습니다.
하지만 IGW에 직접 연결하면 외부에서도 접근이 가능해져 보안에 취약해집니다.
이를 해결하는 두 가지 방법이 있습니다.

### NAT Gateway

Public Subnet에 NAT Gateway를 배치하고, Private Subnet이 외부 인터넷이 필요할 때 NAT Gateway를 거치도록 라우팅을 추가합니다.

```
Private Subnet 리소스 → NAT Gateway (Public Subnet) → IGW → 외부 인터넷
```

- 외부 → Private Subnet으로의 **인바운드 연결은 차단** (단방향)
- 특정 AZ에 생성되며, Elastic IP를 사용
- 여러 AZ에 각각 NAT Gateway를 생성하면 장애 발생 시에도 내결함성(Fault Tolerance) 유지 가능
- **시간당 비용 + 데이터 처리량 비용** 발생

### Bastion Host

Public Subnet에 위치한 EC2 인스턴스(Bastion Host)를 통해 **외부 → Private Subnet**으로의 SSH 접근을 허용합니다.

```
외부 사용자 → SSH → Bastion Host (Public Subnet) → SSH → Private Subnet EC2
```

NAT Gateway가 Private → 외부 방향이라면, Bastion Host는 **외부 → Private** 방향입니다.

---

## NACL (Network Access Control List)

**Subnet 단위**로 트래픽을 제어하는 보안 계층입니다. Subnet을 오고가는 모든 트래픽에 대해 허용(Allow) 또는 거부(Deny)를 설정합니다.

- 서브넷 생성 시 자동으로 **기본 NACL(Default NACL)** 에 연결됨
- 기본 NACL은 인바운드/아웃바운드 트래픽을 **모두 허용**
- 커스텀 NACL은 **모두 차단**된 상태에서 시작
- 규칙에는 **1 ~ 32766** 사이의 번호를 부여하며, **낮은 번호가 우선순위 높음**
- 실무에서는 규칙을 **100 단위**로 설정하는 것이 일반적

---

## Security Group vs NACL

| 비교 항목 | Security Group | NACL |
|-----------|----------------|------|
| 적용 단위 | 인스턴스 단위 | Subnet 단위 |
| 상태 | Stateful (이전 요청 기억) | Stateless (매번 재평가) |
| 규칙 평가 | 모든 규칙 평가 후 결정 | 우선순위 높은 규칙부터 평가 |
| 적용 범위 | EC2 인스턴스 하나 | Subnet 내 모든 EC2 |
| 허용/거부 | 허용(Allow) 규칙만 가능 | 허용 + 거부 모두 가능 |

---

## VPC Peering

원래 VPC들은 서로 독립적이지만, **두 VPC 간 트래픽을 라우팅**할 수 있도록 연결하는 서비스입니다. 마치 하나의 VPC에 있는 것처럼 통신이 가능해집니다.

### 특징

- AWS 네트워크를 통해 연결되므로 **안전**
- VPC의 **CIDR가 서로 겹치면 안됨**
- 피어링 관계는 **전이되지 않음** (A-B 피어링, B-C 피어링 → A-C 직접 통신 불가)
- 피어링 후 **라우팅 테이블도 업데이트** 필요
- **서로 다른 계정, 서로 다른 리전**에서도 가능

---

## VPC Endpoint

S3, CloudWatch, DynamoDB, API Gateway 같은 AWS 서비스들은 VPC 내부에 위치하지 않고, **외부에서 Public IP로 접근**합니다.

Private Subnet의 EC2가 S3에 접근하려면 기본적으로 `NAT Gateway → IGW → 외부` 경로를 거칩니다. 이는 **비용과 보안 이슈**가 발생합니다.

**VPC Endpoint**를 사용하면 인터넷을 거치지 않고 **AWS 내부 네트워크를 통해 직접 연결**할 수 있습니다.

```
Private Subnet EC2 → VPC Endpoint → AWS 서비스 (S3, DynamoDB 등)
```

---

## 전체 구조 요약

```
인터넷
  ↓
IGW (Internet Gateway)
  ↓
Public Subnet
  ├── EC2 (Web Server)
  ├── NAT Gateway
  └── Bastion Host

Private Subnet
  ├── EC2 (App Server)
  └── RDS (Database)

  → VPC Endpoint로 AWS 관리형 서비스 접근 (S3 등)
  → Route Table로 트래픽 경로 제어
  → Security Group + NACL로 이중 보안
```

---

## 마치며

이번 세션을 통해 AWS 네트워킹의 기본이 되는 VPC와 Route 53의 핵심 개념들을 정리했습니다. 처음에는 복잡하게 느껴질 수 있지만, 각 컴포넌트의 역할을 하나씩 이해하다 보면 전체 그림이 보이기 시작합니다.

다음 포스트에서는 직접 AWS 콘솔에서 VPC를 생성하고 EC2와 연결하는 실습 내용을 다뤄볼 예정입니다.
