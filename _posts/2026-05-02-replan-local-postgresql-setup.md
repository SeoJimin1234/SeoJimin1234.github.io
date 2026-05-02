---
title: "[RePLAN] Docker로 로컬 PostgreSQL 환경 구축하기"
excerpt: "Spring Boot 프로젝트에서 Docker Compose로 PostgreSQL을 로컬에 띄우고, application-local.yaml로 개발 환경을 안전하게 분리하는 과정을 정리합니다."
categories:
  - Spring
tags:
  - SpringBoot
  - PostgreSQL
  - Docker
  - 개발환경
  - RePLAN
date: 2026-05-02 00:00:00 +0900
last_modified_at: 2026-05-02 00:00:00 +0900
toc: true
toc_sticky: true
author_profile: true
comments: true
---

> Spring Boot + PostgreSQL 로컬 개발 환경을 Docker Compose로 구성하고, 팀 협업을 위한 설정 파일 분리 전략까지 다룹니다.

---

## 환경

| 항목 | 버전 |
|------|------|
| OS | 로컬 개발 환경 |
| DB | PostgreSQL 16 (Docker) |
| 프레임워크 | Spring Boot 17 (Gradle) |
| 프로파일 | `local` |

---

## 1. docker-compose.local.yml 작성

프로젝트 루트에 `docker-compose.local.yml`을 작성한다.

```yaml
services:
  postgres:
    image: postgres:16
    container_name: replan-postgres
    environment:
      POSTGRES_DB: replan
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: 1234
    ports:
      - "5432:5432"
    volumes:
      - replan_postgres_data:/var/lib/postgresql/data

volumes:
  replan_postgres_data:
```

### 볼륨 네이밍 전략

Docker Named Volume은 Docker 엔진 전역에서 관리된다. 여러 프로젝트를 동시에 개발하다 보면 볼륨 이름이 충돌할 수 있으므로, 프로젝트 이름을 prefix로 붙여(`replan_postgres_data`) 다른 프로젝트의 볼륨과 구분한다.

| 명령 | 볼륨 삭제 여부 |
|------|--------------|
| `docker compose down` | ❌ 볼륨 유지 |
| `docker compose down -v` | ✅ 볼륨 삭제 |

`docker compose down`만으로는 볼륨이 삭제되지 않는다. 컨테이너를 내려도 데이터가 보존되므로, 의도적으로 초기화할 때만 `-v` 플래그를 사용한다.

### Git 관리 전략

실제 파일은 `.gitignore`에 등록하고, example 파일만 Git에 포함한다.

```
docker-compose.local.yml          ← .gitignore에 추가
docker-compose.local.yml.example  ← Git에 포함 (팀 온보딩용)
```

팀원이 레포를 클론한 뒤 example 파일을 보고 자신의 환경에 맞게 복사해서 사용할 수 있도록 한다.

---

## 2. .gitignore 등록

민감한 값이 담긴 파일은 반드시 `.gitignore`에 등록해야 한다.

```gitignore
# 로컬 환경 설정 파일 (실제 값 포함)
application-local.yaml
docker-compose.local.yml
```

팀원 온보딩을 위해 아래 example 파일은 Git에 포함한다.

```
application-local.yaml.example
docker-compose.local.yml.example
```

example 파일에는 실제 비밀번호 대신 플레이스홀더(`<your-password>`)나 팀에서 공유받을 값이라는 안내를 남긴다.

---

## 3. application-local.yaml 작성

Spring Boot는 아래 순서로 설정 파일을 로드한다.

```
application.yaml 로드 → application-local.yaml로 오버라이드
```

따라서 `application-local.yaml`에는 `application.yaml`과 **다른 값만** 작성한다. 값이 동일한 항목을 중복 작성하면 변경 사항 추적이 어려워진다.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/replan
    username: postgres
    password: 1234
  jpa:
    show-sql: true        # base는 false
  data:
    redis:
      host: localhost
      port: 6379

jwt:
  secret: <팀에서 공유받은 값>
  access-expiration: 3600000  # base는 900000
```

### 제거한 중복 항목

`application.yaml`과 값이 동일하여 `application-local.yaml`에서 작성하지 않은 항목:

- `driver-class-name`
- `jpa.properties.hibernate.dialect`
- `jwt.refresh-expiration`

---

## 4. IntelliJ 실행 구성

Run Configuration 편집 > **Active profiles** 항목에 `local` 입력.

환경변수는 `application-local.yaml`에서 직접 값을 지정하므로 IntelliJ Run Configuration의 Environment variables 항목은 별도로 설정하지 않는다.

---

## 5. PostgreSQL 컨테이너 실행

```bash
docker compose -f docker-compose.local.yml up -d
```

PostgreSQL 공식 이미지는 **첫 실행 시 `POSTGRES_DB`에 지정한 이름으로 DB를 자동 생성**한다. 즉, 컨테이너 실행 후 `replan` DB는 이미 존재한다. 테이블은 아직 없는 상태.

이후 실행부터는 볼륨에 데이터가 남아 있으므로 `POSTGRES_DB` 환경 변수를 다시 읽지 않는다. 볼륨을 초기화하려면 `docker compose down -v` 후 재실행한다.

---

## 6. 테이블 생성

`ddl-auto: update`로 설정되어 있으므로 Spring Boot 실행 시 **Hibernate가 Entity 클래스를 읽고 자동으로 테이블을 생성**한다.

```
Entity 클래스 작성 → Spring Boot 실행 → Hibernate가 테이블 자동 생성
```

`ddl-auto: update`는 로컬 개발 환경에서만 사용한다. 서버 환경에서는 `validate`로 설정하고 Flyway로 스키마를 명시적으로 관리한다.

---

## 7. DB 연결 확인

### IntelliJ Database 탭

우측 세로 탭의 Database 패널에서 아래 정보로 연결한다.

| 항목 | 값 |
|------|-----|
| Host | localhost |
| Port | 5432 |
| Database | replan |
| User | postgres |
| Password | 1234 |

### 터미널로 직접 확인

```bash
# 컨테이너 내부 psql 접속
docker exec -it replan-postgres psql -U postgres -d replan

# 테이블 목록 확인
\dt
```

Spring Boot를 한 번 실행한 뒤 `\dt`를 입력하면 Hibernate가 생성한 테이블 목록을 확인할 수 있다.

---

## 참고: 로컬 vs 서버 환경 비교

서버(AWS RDS) 환경과의 전략 차이를 정리한다.

| 항목 | 로컬 | 서버 |
|------|------|------|
| DB | Docker PostgreSQL | AWS RDS PostgreSQL |
| 데이터 영속성 | Named Volume | RDS 자체 관리 |
| 스키마 관리 | `ddl-auto: update` | Flyway 마이그레이션 |
| `ddl-auto` | `update` | `validate` |
| 시크릿 관리 | `application-local.yaml` (gitignore) | SSM Parameter Store |

로컬에서는 빠른 개발 사이클을 위해 Hibernate의 자동 DDL을 허용하지만, 서버에서는 스키마 변경 이력이 명확히 추적되어야 하므로 Flyway로 마이그레이션을 관리한다.
