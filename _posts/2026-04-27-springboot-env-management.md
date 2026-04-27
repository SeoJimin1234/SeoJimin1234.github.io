---
title: "[Spring Boot] .env를 공식 지원하지 않는 Spring Boot, 개발 환경 변수는 어떻게 관리할까?"
excerpt: "Spring Boot가 .env 파일을 공식적으로 지원하지 않는 이유와, 개발 환경에서 환경 변수를 안전하게 관리하는 다양한 방법들을 정리합니다."
categories:
  - Spring
tags:
  - SpringBoot
  - 환경변수
  - 보안
  - 개발환경
  - application.yml
date: 2026-04-27 00:00:00 +0900
last_modified_at: 2026-04-27 00:00:00 +0900
toc: true
toc_sticky: true
author_profile: true
comments: true
---

> Spring Boot 프로젝트를 시작하면서 `.env` 파일로 환경 변수를 관리하려다 막혔던 경험이 있다면, 이 글이 도움이 될 것입니다.

---

## 왜 Spring Boot는 .env를 공식 지원하지 않나?

Node.js 생태계에서 개발을 해봤다면 `.env` 파일은 너무나 당연한 존재입니다. `dotenv` 라이브러리 하나면 로컬에서 DB 비밀번호, API 키를 손쉽게 주입할 수 있으니까요.

그런데 Spring Boot는 공식적으로 `.env` 파일을 읽는 기능이 없습니다.

이유는 Spring Boot의 설계 철학에 있습니다. Spring Boot는 **12-Factor App** 원칙을 따르며, 환경 변수는 OS 레벨에서 주입받는 것을 기본으로 삼습니다. 즉, 파일로 환경 변수를 관리하는 것보다 운영 환경(Kubernetes, ECS, EC2 등)의 환경 변수 주입 메커니즘을 그대로 활용하는 것이 원칙입니다.

또한 Spring Boot는 이미 `application.yml` / `application.properties`라는 강력한 설정 파일 체계를 갖추고 있어서, 별도의 `.env` 지원이 굳이 필요하지 않다는 판단도 있습니다.

하지만 문제는 **개발 환경**입니다. `application.yml`에 DB 비밀번호나 API 키를 직접 적으면 GitHub에 올라갈 위험이 생깁니다. `.gitignore`로 막으면 팀원들이 설정 파일을 공유받지 못하고요.

---

## 개발 환경 변수 관리 방법들

### 방법 1: `application-local.yml` + `.gitignore`

가장 단순하고 직관적인 방법입니다.

**구조:**
```
src/main/resources/
├── application.yml          ← Git에 올림 (공통 설정)
└── application-local.yml    ← Git에서 제외 (민감한 설정)
```

**application.yml:**
```yaml
spring:
  profiles:
    active: local  # 기본 프로파일을 local로 지정
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

**application-local.yml:**
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: mysecretpassword
```

**.gitignore:**
```
application-local.yml
```

Spring Boot는 `spring.profiles.active=local`이 설정되면 `application-local.yml`을 `application.yml` 위에 덮어씌워서 적용합니다. 민감한 값은 `application-local.yml`에만 두고, 이 파일을 `.gitignore`에 추가하면 됩니다.

**장점:**
- 별도 라이브러리 없이 Spring Boot 기본 기능만 사용
- 팀원 간 공유는 별도 채널(노션, 슬랙 등)로 전달

**단점:**
- 파일을 직접 공유해야 하므로 팀 규모가 커지면 관리가 번거로움
- 새 팀원 온보딩 시 설정 파일 전달을 놓칠 수 있음

---

### 방법 2: OS 환경 변수 직접 주입

Spring Boot는 OS 환경 변수를 자동으로 읽습니다. `application.yml`에서 `${DB_PASSWORD}` 형태로 참조하면, OS에 설정된 환경 변수 값을 주입받습니다.

**application.yml:**
```yaml
spring:
  datasource:
    password: ${DB_PASSWORD}
```

**macOS / Linux:**
```bash
export DB_PASSWORD=mysecretpassword
```

**IntelliJ IDEA에서 설정하는 방법:**

Run/Debug Configurations → Edit Configurations → Environment variables에 직접 추가할 수 있습니다.

```
DB_URL=jdbc:mysql://localhost:3306/mydb;DB_USERNAME=root;DB_PASSWORD=mysecretpassword
```

**장점:**
- 파일 없이 환경 변수만으로 깔끔하게 관리
- 운영 환경과 동일한 방식이라 일관성 확보

**단점:**
- 터미널 세션이 닫히면 `export`로 설정한 값이 사라짐 (영구 설정 필요 시 `.zshrc`나 `.bashrc`에 추가해야 함)
- 여러 프로젝트가 같은 변수명을 사용하면 충돌 가능성

---

### 방법 3: `spring-dotenv` 라이브러리

`.env` 방식이 꼭 필요하다면 서드파티 라이브러리를 사용할 수 있습니다.

**의존성 추가 (build.gradle):**
```groovy
implementation 'me.paulschwarz:spring-dotenv:4.0.0'
```

**프로젝트 루트에 `.env` 파일 생성:**
```
DB_URL=jdbc:mysql://localhost:3306/mydb
DB_USERNAME=root
DB_PASSWORD=mysecretpassword
```

**application.yml에서 참조:**
```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

**.gitignore에 추가:**
```
.env
```

`spring-dotenv`는 `.env` 파일을 읽어서 Spring의 `Environment`에 등록해줍니다. 내부적으로 Spring의 `PropertySource` 메커니즘을 사용하므로 `application.yml`의 `${}` 문법과 그대로 호환됩니다.

**장점:**
- Node.js 개발자에게 익숙한 `.env` 방식 그대로 사용 가능
- 설정이 한 파일에 모여서 관리하기 편리

**단점:**
- 공식 지원이 아닌 서드파티 라이브러리 의존
- Spring Boot 버전 업그레이드 시 호환성 문제가 생길 수 있음

---

### 방법 4: IntelliJ IDEA의 EnvFile 플러그인

IntelliJ를 사용한다면 **EnvFile** 플러그인으로 `.env` 파일을 런타임에 주입할 수 있습니다.

1. IntelliJ → Plugins → `EnvFile` 설치
2. Run/Debug Configurations → `EnvFile` 탭 활성화
3. `.env` 파일 경로 지정

코드나 빌드 설정 변경 없이 IDE 레벨에서만 설정이 가능하므로, 라이브러리 의존성을 추가하고 싶지 않을 때 유용합니다.

**장점:**
- 코드베이스에 변경 없이 IDE에서만 설정
- `.env` 파일 자체는 `.gitignore`로 제외하면 됨

**단점:**
- IntelliJ에 종속됨 (VS Code, CLI 환경에서는 동작하지 않음)
- CI/CD 파이프라인에서는 별도 처리 필요

---

### 방법 5: Jasypt로 `application.yml` 값 암호화

`.gitignore`로 파일을 제외하지 않고, 민감한 값 자체를 암호화해서 `application.yml`에 포함시키는 방법입니다.

**의존성 추가:**
```groovy
implementation 'com.github.ulisesbocchio:jasypt-spring-boot-starter:3.0.5'
```

**application.yml:**
```yaml
spring:
  datasource:
    password: ENC(암호화된_문자열)

jasypt:
  encryptor:
    password: ${JASYPT_MASTER_PASSWORD}  # 마스터 키는 환경 변수로 주입
```

암호화된 값은 `ENC(...)` 형태로 감싸면 Jasypt가 자동으로 복호화해서 주입합니다. 마스터 비밀번호 하나만 안전하게 관리하면, 나머지 설정은 암호화된 채로 Git에 올릴 수 있습니다.

**장점:**
- `application.yml`을 Git에 올릴 수 있어 팀원 간 설정 공유가 쉬움
- 마스터 키 하나만 안전하게 전달하면 됨

**단점:**
- 암호화/복호화 과정이 추가되어 설정 변경이 번거로움
- 마스터 키가 노출되면 모든 값이 위험해짐

---

## 방법 비교 요약

| 방법 | 난이도 | 팀 협업 | 추가 라이브러리 | 권장 상황 |
|------|--------|---------|----------------|-----------|
| `application-local.yml` | 낮음 | 보통 | 없음 | 소규모 팀, 빠른 시작 |
| OS 환경 변수 | 낮음 | 낮음 | 없음 | 개인 프로젝트, 운영 환경 통일 |
| `spring-dotenv` | 낮음 | 보통 | 있음 | `.env` 방식 선호 시 |
| EnvFile 플러그인 | 낮음 | 낮음 | 없음 | IntelliJ 전용 팀 |
| Jasypt 암호화 | 높음 | 높음 | 있음 | 설정 파일을 Git에 올려야 할 때 |

---

## 마치며

Spring Boot가 `.env`를 공식 지원하지 않는다는 것이 처음엔 불편하게 느껴졌지만, 이미 `application.yml` 프로파일 분리와 OS 환경 변수 주입이라는 견고한 대안이 있다는 것을 알게 되었습니다.

개인적으로는 **`application-local.yml` + `.gitignore`** 조합을 기본으로 사용하고, 팀 프로젝트에서 설정 파일 공유가 필요할 때는 **Jasypt**나 **Vault** 같은 시크릿 관리 도구를 검토하는 것이 좋다고 생각합니다.

보안 이슈는 나중에 고치기 어렵습니다. 프로젝트 초기부터 민감한 값이 절대 Git에 올라가지 않도록 환경 변수 관리 전략을 잡아두는 습관을 들이는 것이 중요합니다.
