---
title: "[데이터 엔지니어링] Reddit 대규모 크롤링 삽질기 — 10만건에서 500만건까지"
excerpt: "GLP-1 비만 치료제 사용자 경험 분석을 위한 Reddit 크롤링 과정에서 IP 차단, API 설계 문제, 댓글 계층 재구성까지 겪은 시행착오를 기록합니다."
categories:
  - 데이터엔지니어링
tags:
  - 크롤링
  - Reddit
  - Python
  - MongoDB
  - ArcticShift
  - 데이터수집
  - 파이프라인
date: 2026-04-24 00:00:00 +0900
last_modified_at: 2026-04-24 00:00:00 +0900
toc: true
toc_sticky: true
author_profile: true
comments: true
---

> GLP-1 계열 비만 치료제(위고비, 마운자로) 사용자 경험 분석 프로젝트에서 Reddit 데이터 수집을 담당하며 겪은 시행착오를 기록한 글입니다.

---

## 들어가며

GLP-1 계열 비만 치료제(위고비, 마운자로)에 대한 실제 사용자 경험을 분석하는 데이터 엔지니어링 프로젝트를 진행했습니다. 임상시험에 기록된 공식 부작용 데이터와 Reddit 커뮤니티에서 실제 복용자들이 공유하는 경험을 비교 분석하는 것이 목표였습니다.

데이터 수집 담당을 맡으면서 Reddit 크롤링을 시도했고, 생각보다 많은 시행착오를 겪었습니다. 이 글은 그 과정을 기록한 것입니다.

---

## 왜 Reddit인가?

Reddit을 선택한 이유는 명확했습니다.

- `r/WegovyWeightLoss`, `r/Mounjaro` 등 **전용 커뮤니티**가 활성화되어 있습니다
- 복용 경험, 부작용, 용량 변화 등 **풍부한 텍스트 데이터**가 존재합니다
- 게시글 → 댓글 → 대댓글의 **계층적 토론 구조**가 명확하게 유지됩니다
- 2021년(약물 출시 시점)부터의 **시계열 데이터** 추적이 가능합니다
- 다수의 학술 논문에서 Reddit을 GLP-1 관련 사용자 경험 분석의 주요 데이터 소스로 활용하고 있습니다

---

## 1차 시도: Reddit JSON API (비공식)

### 방식

Reddit은 모든 페이지 URL 뒤에 `.json`을 붙이면 해당 페이지 데이터를 JSON으로 반환합니다. 별도 API 키나 인증 없이 HTTP GET 요청만으로 데이터를 가져올 수 있는 비공식 방식입니다.

```python
url = "https://www.reddit.com/r/WegovyWeightLoss.json"
params = {"limit": 100, "after": after_token}
response = requests.get(url, headers={"User-Agent": "Mozilla/5.0"})
```

페이지네이션은 커서 기반으로, 응답의 `after` 값을 다음 요청에 넘기는 방식으로 구현했습니다.

### 결과 및 문제

약 **10만건**을 수집하다가 IP 제한(`429 Too Many Requests`)에 걸려 수집이 중단됐습니다. 요청 간격을 늘려도 대량 수집 시 결국 차단됐습니다.

> **문제**: 동일 IP에서 반복 요청 시 Reddit 서버가 차단

---

## 2차 시도: ScraperAPI 프록시 도입 검토

IP 제한 문제를 해결하기 위해 프록시 서비스 도입을 검토했습니다.

ScraperAPI 최저 플랜이 월 $49(약 7만원)인데, Reddit은 JS 렌더링이 필요한 페이지가 많아 크레딧을 **5~10배** 소모합니다. 결과적으로 $49를 쓰고도 기존 10만건보다 적게 수집하는 최악의 상황이 예상됐습니다.

> **문제**: 비용 대비 수집량이 너무 낮음 → 채택하지 않음

---

## 3차 시도: Arctic Shift API 전환

### Arctic Shift란?

Reddit 전체 데이터를 아카이빙하는 오픈 프로젝트입니다. Pushshift의 후계자 격으로, 2025년 12월까지의 Reddit 데이터를 무료로 제공합니다.

| 항목 | 내용 |
|---|---|
| Base URL | `https://arctic-shift.photon-reddit.com` |
| 인증 | 불필요 |
| IP 제한 | 없음 |
| 비용 | 무료 |

### 게시글 수집 전략

`/api/posts/search` 엔드포인트로 서브레딧 단위 수집을 구현했습니다.

Reddit 공식 API와 달리 키워드 단독 검색이 지원되지 않아, **서브레딧을 직접 지정**하는 방식을 채택했습니다. 서브레딧 성격에 따라 두 가지 전략을 적용했습니다.

**전용 서브레딧 — 전체 수집**

`WegovyWeightLoss`, `Mounjaro`, `Semaglutide`, `ozempic` 등 12개 약물 전용 커뮤니티는 전체 게시글을 수집했습니다.

**범용 서브레딧 — 키워드 필터링**

`diabetes`, `loseit`, `obesity` 등 범용 커뮤니티는 제목에 관련 키워드가 포함된 게시글만 수집했습니다.

```python
params = {
    "subreddit": "WegovyWeightLoss",
    "after": "2021-01-01",
    "before": "2025-12-31",
    "limit": 100,
    "sort": "asc",
}
```

페이지네이션은 마지막 게시글의 `created_utc + 1`을 다음 `after`로 설정하는 방식으로 구현했습니다.

### 결과

게시글 수집은 빠르게 완료됐습니다. 기존 10만건과 중복 제거 후 **신규 47만건** 수집에 성공했습니다.

---

## 댓글 수집의 함정: 47만번 API 호출

### 1차 방식: 게시글별 tree API

게시글마다 `/api/comments/tree`를 호출해서 댓글 트리를 가져오는 방식을 구현했습니다. 계층 구조를 완벽하게 보존할 수 있다는 장점이 있었습니다.

```python
# 게시글 하나당 API 1번 호출
r = requests.get(f"{BASE_URL}/api/comments/tree", params={"link_id": f"t3_{post_id}"})
```

그런데 치명적인 문제가 있었습니다. 게시글이 **47만건**이니 API를 47만번 호출해야 합니다. 실제로 돌려보니 **10시간에 1만건** 처리가 고작이었습니다. 단순 계산으로 **19일**이 걸리는 수준이었습니다.

> **문제**: 게시글 수 × API 호출 = 현실적으로 불가능한 수집 시간

### 2차 방식: 서브레딧별 flat 수집 + 계층 후처리

발상을 전환했습니다. 게시글별로 댓글을 가져오는 대신, **서브레딧 단위**로 댓글을 한 번에 긁어온 뒤 `parent_id`를 활용해 계층 구조를 재구성하는 방식으로 변경했습니다.

```python
# 서브레딧 전체 댓글을 시간순으로 수집
params = {
    "subreddit": "WegovyWeightLoss",
    "after": "2021-01-01",
    "limit": 100,
    "sort": "asc",
    "fields": "id,body,author,created_utc,score,link_id,parent_id"
}
```

수집된 flat 댓글 리스트에서 `parent_id`를 따라가며 `depth_level`과 `path`를 계산합니다.

```python
def build_hierarchy(comments):
    id_map = {c["id"]: c for c in comments}
    
    for c in comments:
        depth = 1
        path_parts = [link_id]
        current = clean_parent
        
        while current and current != link_id:
            path_parts.append(current)
            current = id_map[current]["parent_id"]
            depth += 1
        
        path_parts.append(comment_id)
        # depth_level = depth
        # path = "루트ID/부모ID/.../자신ID"
```

API 호출 횟수가 **47만번 → 수천번**으로 대폭 감소했습니다.

---

## 최종 데이터 스키마

게시글과 댓글을 동일한 스키마로 통일해서 MongoDB에 저장했습니다.

| 컬럼 | 설명 |
|---|---|
| `document_id` | 고유 식별자 |
| `type` | `post` / `comment` |
| `platform` | `Reddit` |
| `drug_type` | 본문 키워드 기반 추론 (`wegovy` / `mounjaro` / `unknown`) |
| `subreddit` | 서브레딧명 |
| `text_body` | 본문 텍스트 |
| `author_id` | 작성자 ID |
| `timestamp` | 작성 시간 (Unix epoch) |
| `engagement_metrics` | 추천수 |
| `thread_id` | 루트 게시글 ID |
| `parent_id` | 부모 문서 ID |
| `depth_level` | 계층 깊이 (게시글=0, 직접 댓글=1, 대댓글=2...) |
| `path` | `루트ID/부모ID/.../자신ID` |
| `post_title` | 게시글 제목 |

---

## 중복 제거 전략

수집 과정에서 중복 데이터가 발생할 수 있는 지점이 여러 곳이었습니다.

1. **기존 10만건과의 중복**: 수집 시작 시 기존 CSV의 `document_id`를 메모리(set)에 올려두고, 새로 수집한 데이터와 비교해 실시간 필터링
2. **MongoDB 업로드 중복**: `$setOnInsert` + `upsert=True` 조합으로 동일 `document_id`가 이미 있으면 스킵
3. **수집 중단 후 재시작**: `checkpoint.json`에 서브레딧별 진행 상황을 저장해 이어받기 구현

---

## 저장소: MongoDB Atlas → Azure Cosmos DB

처음에는 MongoDB Atlas 프리티어(512MB)를 사용했는데, 게시글 47만건 업로드 도중 용량 초과로 막혔습니다. 팀원이 Azure Cosmos DB로 이전하면서 해결됐습니다.

Azure Cosmos DB는 MongoDB 프로토콜과 호환되어 **기존 pymongo 코드를 그대로 사용**할 수 있었습니다.

---

## 최종 수집 현황

| 항목 | 내용 |
|---|---|
| 수집 기간 | 2021-01-01 ~ 2026-12-31 |
| 수집 서브레딧 | 전용 12개 + 범용 4개 |
| 게시글 | 약 47만건 |
| 댓글 | 수백만건 (수집 중) |
| 총 DB 문서 수 | 약 470만건 |
| 저장소 | Azure Cosmos DB (MongoDB 호환) |

---

## 마치며

단순해 보이는 크롤링 작업도 막상 해보면 IP 제한, API 구조 파악, 대용량 처리, 계층 구조 재구성, 중복 제거, 용량 문제 등 예상치 못한 장애물이 계속 나옵니다.

처음 JSON API로 시작했을 때는 "그냥 요청 보내면 되는 거 아닌가"라고 생각했습니다. 하지만 10만건에서 IP 차단을 맞고, 프록시 비용을 계산하고, 아카이브 서비스를 찾아내고, 댓글 수집이 19일짜리 작업이라는 걸 깨닫고, 서브레딧 단위로 전략을 바꾸는 과정을 거치면서 데이터 수집이 단순한 작업이 아니라는 것을 체감했습니다.

특히 **"어떻게 수집할까"보다 "얼마나 효율적으로 수집할까"가 훨씬 중요한 문제**라는 것을 배웠습니다. 게시글별 API 호출 방식은 기술적으로 정확했지만 현실적으로 쓸 수 없었고, 서브레딧 단위 수집 + 후처리 방식은 약간의 트레이드오프를 감수하는 대신 **실제로 동작하는 파이프라인**을 만들어줬습니다.

데이터 엔지니어링에서 완벽한 설계보다 **작동하는 파이프라인**이 먼저라는 것, 그리고 막혔을 때 문제를 우회하는 방법을 찾는 능력이 중요하다는 것을 이번 프로젝트에서 직접 느꼈습니다.
