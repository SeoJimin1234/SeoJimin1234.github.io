---
title: "[RePLAN] Spring Boot 4.0에서 Flyway가 실행되지 않는 문제 트러블슈팅"
excerpt: "prod 배포 시 Flyway가 완전히 무시되는 현상을 6단계에 걸쳐 추적한 과정. starter vs 직접 의존성의 차이, CONDITIONS EVALUATION REPORT 활용법, 두 JAR 문제까지 다룹니다."
categories:
  - Spring
tags:
  - SpringBoot
  - Flyway
  - PostgreSQL
  - Docker
  - 트러블슈팅
  - RePLAN
date: 2026-05-15 00:00:00 +0900
last_modified_at: 2026-05-15 00:00:00 +0900
toc: true
toc_sticky: true
author_profile: true
comments: true
---

> Spring Boot 4.0 + Flyway 11에서 배포할 때마다 Flyway가 전혀 실행되지 않는 현상을 6단계에 걸쳐 추적했습니다. 근본 원인은 `flyway-core` 직접 의존성이 자동 구성 등록을 트리거하지 않는다는 점이었습니다.

---

## 환경

| 항목 | 버전 |
|------|------|
| Spring Boot | 4.0.4 |
| Flyway | 11.14.1 |
| DB | PostgreSQL 16 |
| 배포 | AWS EC2 + Docker Compose |

---

## 증상

서버에 배포할 때마다 Flyway가 전혀 실행되지 않았다.

- 앱 시작 로그에 Flyway 관련 출력이 한 줄도 없음
- `flyway_schema_history` 테이블이 DB에 생성되지 않음
- 스키마 불일치로 인해 `ddl-auto: validate` 설정이 앱을 즉시 종료

---

## 프로젝트 구조

```text
src/main/resources/
├── application.yaml          # 공통 base 설정
├── application-local.yaml    # 로컬 프로파일
└── application-prod.yaml     # 프로덕션 프로파일
```

배포 파이프라인은 GitHub Actions → AWS ECR → EC2 Docker Compose이고, `.env` 파일에 `SPRING_PROFILES_ACTIVE=prod`를 주입한다.

---

## 트러블슈팅 과정

### 1단계: application-prod 설정 파일 점검

처음 확인한 `application-prod.yaml`:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate        # ← Flyway 미실행 시 앱이 죽는 원인
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    baseline-version: "1"
```

`ddl-auto: validate`는 Flyway가 정상 실행된다는 전제 하에서만 안전하다. Flyway가 실행되지 않은 상태에서는 스키마 불일치로 앱이 종료된다.

**조치:** `ddl-auto: validate` → `none`으로 임시 변경.

---

### 2단계: base 설정 파일 점검

`application.yaml`에 다음 설정이 있었다:

```yaml
spring:
  flyway:
    enabled: false
```

Spring Boot는 base 설정 위에 프로파일 설정을 병합한다. 그러나 base에서 `enabled: false`가 선언되면 prod 프로파일에서 `enabled: true`로 재선언해도 예상대로 덮어써지지 않을 수 있다.

**조치:** base 설정에서 `spring.flyway` 블록 전체 제거.

---

### 3단계: `.yml` vs `.yaml` 확장자 의심

```text
application.yaml      ← .yaml
application-prod.yml  ← .yml  (불일치)
```

Spring Boot는 두 확장자를 모두 지원하지만, 혼용 시 예상치 못한 동작이 생길 수 있다는 의심이 들었다.

배포된 JAR를 직접 확인했다. JRE 전용 이미지(`eclipse-temurin:17-jre-alpine`)에는 `jar` 명령어가 없으므로 `unzip`을 사용해야 한다:

```bash
unzip -l /app/app.jar | grep "application"
```

```text
BOOT-INF/classes/application.yaml
BOOT-INF/classes/application-prod.yaml   ← 이미 .yaml로 변경된 상태
```

두 파일 모두 JAR에 정상 포함되어 있었다.

**조치:** 일관성을 위해 `application-prod.yml` → `application-prod.yaml`로 이름 변경 (근본 원인은 아님).

---

### 4단계: 두 개의 JAR 파일 문제

`build.gradle`에 `jar { enabled = false }` 설정이 없어서 `build/libs/` 에 두 개의 JAR가 생성되고 있었다:

```text
build/libs/
├── replan-0.0.1-SNAPSHOT.jar        # Spring Boot 실행 가능 JAR (bootJar)
└── replan-0.0.1-SNAPSHOT-plain.jar  # 일반 JAR (jar task)
```

`Dockerfile`의 `COPY` 명령이 와일드카드를 사용하고 있었다:

```dockerfile
COPY --from=build /app/build/libs/*.jar app.jar
```

`*.jar`가 두 파일에 매칭되어 `COPY` 명령이 실패하거나 잘못된 JAR를 복사할 수 있다.

**조치:** `build.gradle`에 추가:

```gradle
jar {
    enabled = false
}
```

---

### 5단계: Flyway JAR가 app.jar에 포함되어 있는지 확인

서버에서 다음 명령을 실행했다:

```bash
docker exec -it app-app-1 jar tf /app/app.jar | grep flyway
```

아무것도 출력되지 않았다. 처음에는 Flyway 의존성이 빠진 것으로 판단했지만, 이는 잘못된 결론이었다.

**원인:** `eclipse-temurin:17-jre-alpine`은 JRE만 포함하므로 `jar` 명령어가 존재하지 않는다. 올바른 방법은 `unzip`을 사용하는 것이다:

```bash
docker exec -it app-app-1 unzip -l /app/app.jar | grep flyway
```

```text
47946   BOOT-INF/lib/flyway-database-postgresql-11.14.1.jar
737727  BOOT-INF/lib/flyway-core-11.14.1.jar
```

Flyway JAR는 정상적으로 포함되어 있었다.

---

### 6단계: CONDITIONS EVALUATION REPORT 분석

`application-prod.yaml`에 `debug: true`를 임시로 추가해서 자동 구성 조건 리포트를 출력했다.

```yaml
debug: true
```

리포트를 확인한 결과, `FlywayAutoConfiguration`이 **Positive, Negative, Unconditional 어디에도 나타나지 않았다.**

이것이 핵심 단서였다.

`@ConditionalOnClass`가 실패하면 해당 자동 구성 클래스는 Negative matches에 기록된다. 리포트에 아예 등록되지 않는다는 것은 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 파일에 `FlywayAutoConfiguration`이 존재하지 않는다는 의미다.

---

## 근본 원인

**`flyway-core`를 직접 추가하면 Spring Boot 4.0의 자동 구성 메커니즘이 `FlywayAutoConfiguration`을 등록하지 않는다.**

기존 `build.gradle`:

```gradle
implementation 'org.flywaydb:flyway-core'
implementation 'org.flywaydb:flyway-database-postgresql'
```

Spring Boot의 `FlywayAutoConfiguration`은 `spring-boot-autoconfigure`의 `AutoConfiguration.imports`에 등록되어 있다. 이 자동 구성이 활성화되려면 Spring Boot가 인식하는 starter를 통해 의존성을 가져와야 한다.

`flyway-core`를 직접 추가하면 Flyway 라이브러리 자체는 클래스패스에 올라오지만, `FlywayAutoConfiguration`이 조건 평가 대상에 포함되지 않아 Bean이 생성되지 않는다.

---

## 해결책

`flyway-core`를 `spring-boot-starter-flyway`로 교체한다.

```gradle
// 변경 전
implementation 'org.flywaydb:flyway-core'
implementation 'org.flywaydb:flyway-database-postgresql'

// 변경 후
implementation 'org.springframework.boot:spring-boot-starter-flyway'
implementation 'org.flywaydb:flyway-database-postgresql'
```

`spring-boot-starter-flyway`는 내부적으로 `flyway-core`를 포함하며, Spring Boot의 자동 구성 등록 절차를 올바르게 트리거한다.

---

## 로컬 검증

배포 전 로컬에서 Docker로 빈 PostgreSQL을 띄우고 prod 프로파일로 앱을 실행해 Flyway가 정상 작동하는지 검증했다.

```bash
# 빈 PostgreSQL 컨테이너 실행
docker run -d --name flyway-test-pg \
  -e POSTGRES_DB=replan \
  -e POSTGRES_USER=testuser \
  -e POSTGRES_PASSWORD=testpass \
  -p 5433:5432 \
  postgres:16-alpine

# prod 프로파일로 앱 실행
DB_HOST=localhost DB_PORT=5433 DB_NAME=replan \
DB_USER=testuser DB_PASSWORD=testpass \
SPRING_PROFILES_ACTIVE=prod \
JWT_SECRET=this-is-a-test-secret-key-must-be-32chars \
GOOGLE_WEB_CLIENT_ID=test \
REDIS_HOST=localhost REDIS_PORT=6379 \
AWS_REGION=ap-northeast-2 AWS_S3_BUCKET=test AWS_CLOUDFRONT_DOMAIN=test \
./gradlew bootRun
```

확인된 로그:

```text
INFO  FlywayExecutor : Database: jdbc:postgresql://localhost:5433/replan (PostgreSQL 16.14)
INFO  JdbcTableSchemaHistory : Schema history table "public"."flyway_schema_history" does not exist yet
INFO  JdbcTableSchemaHistory : Creating Schema History table "public"."flyway_schema_history" ...
INFO  DbMigrate : Migrating schema "public" to version "1 - init"
INFO  DbMigrate : Migrating schema "public" to version "2 - add created at and set is pinned default"
INFO  DbMigrate : Successfully applied 2 migrations to schema "public", now at version v2
```

---

## 최종 변경 요약

| 파일 | 변경 내용 |
|------|----------|
| `build.gradle` | `flyway-core` → `spring-boot-starter-flyway` 교체, `jar { enabled = false }` 추가 |
| `application.yaml` | `spring.flyway.enabled: false` 블록 제거 |
| `application-prod.yaml` | `ddl-auto: none`으로 변경 |
| `application-prod.yml` | `application-prod.yaml`로 이름 변경 (확장자 통일) |

---

## 교훈

### 1. Spring Boot starter vs 직접 의존성

Spring Boot 생태계의 라이브러리는 가능하면 `spring-boot-starter-*`를 통해 추가해야 한다. `starter`는 단순한 의존성 묶음이 아니라 자동 구성 등록, 버전 관리, 조건부 Bean 생성까지 책임진다. 라이브러리를 직접 추가하면 Spring Boot가 해당 기능을 자동으로 인식하지 못할 수 있다.

### 2. CONDITIONS EVALUATION REPORT는 강력한 디버깅 도구다

자동 구성이 의심될 때 `debug: true`를 추가해 CONDITIONS EVALUATION REPORT를 확인하면 어떤 자동 구성이 활성화됐고 왜 비활성화됐는지 명확히 알 수 있다. 특히 **리포트에 해당 클래스가 아예 없다면** 의존성 자체가 아니라 자동 구성 등록 단계부터 문제임을 바로 알 수 있다.

### 3. JRE-only 컨테이너에서는 jar 명령어가 없다

`eclipse-temurin:17-jre-alpine` 같은 JRE 전용 이미지에는 `jar` 명령어가 포함되지 않는다. 컨테이너 내부에서 JAR 내용을 확인할 때는 `unzip -l`을 사용해야 한다.

```bash
# JRE 환경에서 JAR 내용 확인
unzip -l /app/app.jar | grep <검색어>
```

### 4. 배포 전 로컬 검증은 Docker로 간단히 가능하다

서버 환경(prod 프로파일)을 로컬에서 재현하려면 Docker로 빈 DB를 띄우고 환경변수만 주입하면 된다. 로컬 개발 DB를 건드리지 않고 prod 설정이 정상 작동하는지 안전하게 검증할 수 있다.
