---
title: "[CS스터디] 정렬 알고리즘 완전 정복 — 버블부터 팀 정렬까지"
excerpt: "Bubble, Selection, Insertion, Merge, Quick, Heap, Counting, Radix, Tim Sort의 원리·시간복잡도·공간복잡도·안정성을 한 번에 정리합니다."
categories:
  - CS스터디
tags:
  - 정렬
  - 알고리즘
  - 자료구조
  - 면접준비
  - 시간복잡도
date: 2026-04-15 00:00:00 +0900
last_modified_at: 2026-04-15 00:00:00 +0900
toc: true
toc_sticky: true
author_profile: true
comments: true
---

> CS 기초 스터디 — 대표 정렬 알고리즘의 동작 원리, 시간·공간 복잡도, 안정성, 실전 사용처를 비교·정리한 글입니다.

---

## 1. 정렬 알고리즘이란?

**정렬(Sorting)**은 데이터를 특정 기준(오름차순, 내림차순 등)에 따라 순서대로 나열하는 연산입니다.  
실전에서는 탐색 성능을 높이거나, 중복 제거, 우선순위 처리 등 거의 모든 알고리즘의 전처리 단계로 쓰입니다.

### 핵심 용어 정리

| 용어 | 설명 |
|------|------|
| **시간 복잡도** | 입력 크기 n에 따른 연산 횟수 |
| **공간 복잡도** | 추가로 사용하는 메모리 |
| **안정 정렬 (Stable)** | 같은 키를 가진 원소의 상대적 순서가 보존됨 |
| **제자리 정렬 (In-place)** | O(1) 추가 메모리만 사용 |
| **비교 기반 정렬** | 원소 간 비교를 통해 순서를 결정 — 하한이 O(n log n) |

---

## 2. 한눈에 보는 복잡도 비교표

| 알고리즘 | 최선 | 평균 | 최악 | 공간 | 안정성 |
|----------|------|------|------|------|--------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | ❌ |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ |
| Counting Sort | O(n + k) | O(n + k) | O(n + k) | O(k) | ✅ |
| Radix Sort | O(nk) | O(nk) | O(nk) | O(n + k) | ✅ |
| Tim Sort | O(n) | O(n log n) | O(n log n) | O(n) | ✅ |

> k = 값의 범위(Counting), 자릿수(Radix)

---

## 3. 버블 정렬 (Bubble Sort)

### 동작 원리

인접한 두 원소를 비교해 큰 값을 오른쪽으로 밀어 올립니다. 마치 거품이 수면 위로 떠오르는 것처럼 가장 큰 값이 끝으로 이동합니다.

```
[5, 3, 8, 1]
→ [3, 5, 8, 1]  (5↔3)
→ [3, 5, 8, 1]  (5<8, 유지)
→ [3, 5, 1, 8]  (8↔1) ← 1 pass 완료, 8 확정
→ [3, 1, 5, 8]  (5↔1)
→ ...
```

### 코드 (Python)

```python
def bubble_sort(arr):
    n = len(arr)
    for i in range(n):
        swapped = False
        for j in range(n - 1 - i):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
                swapped = True
        if not swapped:   # 교환이 없으면 이미 정렬됨 → O(n) 조기 종료
            break
    return arr
```

### 특징

- 구현이 가장 단순하지만 실전에서는 거의 쓰이지 않음
- `swapped` 플래그로 최선 케이스를 O(n)으로 개선 가능
- 데이터가 거의 정렬된 경우 유용

---

## 4. 선택 정렬 (Selection Sort)

### 동작 원리

전체 배열에서 **최솟값을 선택**해 맨 앞과 교환합니다. 이 과정을 반복하며 앞에서부터 확정된 구간을 늘려 갑니다.

```
[5, 3, 8, 1]
→ [1, 3, 8, 5]  (최솟값 1을 index 0과 교환)
→ [1, 3, 8, 5]  (최솟값 3은 제자리)
→ [1, 3, 5, 8]  (최솟값 5를 index 2와 교환)
```

### 코드 (Python)

```python
def selection_sort(arr):
    n = len(arr)
    for i in range(n):
        min_idx = i
        for j in range(i + 1, n):
            if arr[j] < arr[min_idx]:
                min_idx = j
        arr[i], arr[min_idx] = arr[min_idx], arr[i]
    return arr
```

### 특징

- 교환 횟수가 O(n)으로 적어 **쓰기 비용이 큰 매체(플래시 메모리 등)** 에서 고려 가능
- 안정 정렬이 아님 — 같은 값의 원소 순서가 바뀔 수 있음

---

## 5. 삽입 정렬 (Insertion Sort)

### 동작 원리

**카드를 손에 쥐고 정렬**하는 방식과 동일합니다. 새 원소를 이미 정렬된 구간의 올바른 위치에 끼워 넣습니다.

```
[5, 3, 8, 1]
→ [3, 5, 8, 1]  (3을 5 앞에 삽입)
→ [3, 5, 8, 1]  (8은 제자리)
→ [1, 3, 5, 8]  (1을 맨 앞에 삽입)
```

### 코드 (Python)

```python
def insertion_sort(arr):
    for i in range(1, len(arr)):
        key = arr[i]
        j = i - 1
        while j >= 0 and arr[j] > key:
            arr[j + 1] = arr[j]
            j -= 1
        arr[j + 1] = key
    return arr
```

### 특징

- **거의 정렬된 데이터**에서 O(n)에 가까운 성능 — Tim Sort의 핵심 구성 요소
- 작은 배열(n ≤ 10~20)에서는 오버헤드 없이 빠름
- 안정 정렬

---

## 6. 합병 정렬 (Merge Sort)

### 동작 원리

**분할 정복(Divide & Conquer)** 전략을 사용합니다. 배열을 반으로 나누고, 각각을 재귀적으로 정렬한 뒤 두 정렬된 배열을 합칩니다.

```
[5, 3, 8, 1]
분할: [5, 3] | [8, 1]
분할: [5] [3] | [8] [1]
합병: [3, 5] | [1, 8]
합병: [1, 3, 5, 8]
```

### 코드 (Python)

```python
def merge_sort(arr):
    if len(arr) <= 1:
        return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i]); i += 1
        else:
            result.append(right[j]); j += 1
    result.extend(left[i:])
    result.extend(right[j:])
    return result
```

### 특징

- 최악 케이스도 O(n log n) 보장 — **안정성이 중요한 경우의 표준 선택**
- 추가 메모리 O(n) 필요 — 메모리가 제한된 환경에서는 단점
- **Linked List 정렬**에 특히 적합 (포인터 조작으로 추가 공간 절약 가능)

---

## 7. 퀵 정렬 (Quick Sort)

### 동작 원리

**피벗(pivot)**을 기준으로 배열을 두 파티션으로 나눕니다. 피벗보다 작은 값은 왼쪽, 큰 값은 오른쪽에 배치하고 재귀적으로 반복합니다.

```
[5, 3, 8, 1, 9, 2]  (pivot = 5)
→ [3, 1, 2] | 5 | [8, 9]
→ [1, 2, 3, 5, 8, 9]
```

### 코드 (Python — Lomuto 파티션)

```python
def quick_sort(arr, low=0, high=None):
    if high is None:
        high = len(arr) - 1
    if low < high:
        pivot_idx = partition(arr, low, high)
        quick_sort(arr, low, pivot_idx - 1)
        quick_sort(arr, pivot_idx + 1, high)
    return arr

def partition(arr, low, high):
    pivot = arr[high]
    i = low - 1
    for j in range(low, high):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1
```

### 피벗 선택 전략

| 전략 | 설명 | 최악 케이스 발생 조건 |
|------|------|----------------------|
| 첫 번째 원소 | 단순 | 이미 정렬된 배열 |
| 마지막 원소 | 단순 | 이미 정렬된 배열 |
| 중앙값 (Median-of-3) | 세 값의 중앙값 | 드묾 |
| 랜덤 | 무작위 선택 | 사실상 없음 |

### 특징

- **평균적으로 가장 빠른 정렬** — 캐시 지역성이 좋아 실전 성능이 뛰어남
- 최악 케이스 O(n²) — 피벗을 잘 선택하면 거의 발생하지 않음
- 재귀 스택 공간 O(log n) 사용
- 안정 정렬 아님

---

## 8. 힙 정렬 (Heap Sort)

### 동작 원리

**최대 힙(Max Heap)**을 구성한 뒤, 루트(최댓값)를 배열 끝과 교환하고 힙 크기를 줄이며 반복합니다.

```
1단계: 배열 → Max Heap 구성 (heapify)
2단계: 루트(최댓값)를 끝으로 이동 → 힙 크기 감소 → 다시 heapify
반복 → 오름차순 정렬 완성
```

### 코드 (Python)

```python
def heap_sort(arr):
    n = len(arr)
    # 최대 힙 구성
    for i in range(n // 2 - 1, -1, -1):
        heapify(arr, n, i)
    # 원소 하나씩 추출
    for i in range(n - 1, 0, -1):
        arr[0], arr[i] = arr[i], arr[0]
        heapify(arr, i, 0)
    return arr

def heapify(arr, n, i):
    largest = i
    left, right = 2 * i + 1, 2 * i + 2
    if left < n and arr[left] > arr[largest]:
        largest = left
    if right < n and arr[right] > arr[largest]:
        largest = right
    if largest != i:
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)
```

### 특징

- 최악 케이스도 O(n log n) + 추가 메모리 O(1) — **메모리 효율 + 성능 보장**
- 캐시 지역성이 나빠 실전에서는 Quick Sort보다 느린 경우가 많음
- 안정 정렬 아님
- 우선순위 큐 구현의 기반

---

## 9. 계수 정렬 (Counting Sort)

### 동작 원리

비교를 하지 않고 **각 값의 등장 횟수를 배열에 기록**한 뒤, 누적합을 이용해 원소를 올바른 위치에 배치합니다.

```
입력: [4, 2, 2, 8, 3, 3, 1]  (값 범위 1~8)
count: [0, 1, 2, 2, 1, 0, 0, 0, 1]
출력: [1, 2, 2, 3, 3, 4, 8]
```

### 코드 (Python)

```python
def counting_sort(arr):
    if not arr:
        return arr
    max_val = max(arr)
    count = [0] * (max_val + 1)
    for x in arr:
        count[x] += 1
    result = []
    for val, cnt in enumerate(count):
        result.extend([val] * cnt)
    return result
```

### 특징

- **비교 기반이 아니므로 O(n log n) 하한을 깸** — O(n + k)
- 값의 범위 k가 n보다 훨씬 크면 메모리와 시간 낭비
- **정수, 한정된 범위의 값**에만 사용 가능
- Radix Sort의 서브루틴으로 활용

---

## 10. 기수 정렬 (Radix Sort)

### 동작 원리

**자릿수(Digit)** 단위로 낮은 자리부터 높은 자리 순으로 안정 정렬을 반복 적용합니다. 각 자리에서 Counting Sort를 서브루틴으로 사용합니다.

```
입력: [170, 45, 75, 90, 802, 24, 2, 66]

1의 자리 정렬: [170, 90, 802, 2, 24, 45, 75, 66]
10의 자리 정렬: [802, 2, 24, 45, 66, 170, 75, 90]
100의 자리 정렬: [2, 24, 45, 66, 75, 90, 170, 802]
```

### 코드 (Python)

```python
def radix_sort(arr):
    max_val = max(arr)
    exp = 1
    while max_val // exp > 0:
        counting_sort_by_digit(arr, exp)
        exp *= 10
    return arr

def counting_sort_by_digit(arr, exp):
    n = len(arr)
    output = [0] * n
    count = [0] * 10
    for i in arr:
        count[(i // exp) % 10] += 1
    for i in range(1, 10):
        count[i] += count[i - 1]
    for i in range(n - 1, -1, -1):
        idx = (arr[i] // exp) % 10
        output[count[idx] - 1] = arr[i]
        count[idx] -= 1
    for i in range(n):
        arr[i] = output[i]
```

### 특징

- **자릿수 k가 고정되면 사실상 O(n)** — 대용량 정수 정렬에서 매우 효율적
- 문자열, 고정 길이 키 정렬에도 응용 가능
- 부동소수점, 음수에는 추가 처리 필요

---

## 11. 팀 정렬 (Tim Sort)

### 동작 원리

**Insertion Sort + Merge Sort**의 하이브리드입니다. 실세계 데이터에 자연스럽게 존재하는 **런(run, 부분 정렬된 구간)**을 활용합니다.

1. 배열에서 자연 런(오름차순 또는 내림차순 구간)을 찾음
2. 런이 최소 크기(minrun, 보통 32~64)보다 작으면 Insertion Sort로 확장
3. 런들을 스택에 쌓으면서 균형 조건을 만족하도록 Merge

### 특징

- **Python의 `sorted()`, `list.sort()`** 및 **Java의 `Arrays.sort(Object[])`** 에서 기본 정렬로 채택
- 실세계 데이터(거의 정렬된 경우)에서 탁월한 성능
- 최선 O(n) — 이미 정렬된 배열에서 런 발견 후 즉시 완료
- 안정 정렬, 추가 메모리 O(n)

---

## 12. 정렬 알고리즘 선택 가이드

```
데이터 특성에 따른 선택:

n이 매우 작다 (n ≤ 20)?
  → Insertion Sort

데이터가 거의 정렬되어 있다?
  → Insertion Sort 또는 Tim Sort

안정 정렬이 필요하다?
  → Merge Sort 또는 Tim Sort

메모리가 매우 제한되어 있다?
  → Heap Sort (O(1) 추가 메모리, O(n log n) 보장)

평균 성능이 최우선이다?
  → Quick Sort (랜덤 피벗 또는 Median-of-3)

값의 범위가 정수이고 범위가 좁다?
  → Counting Sort

자릿수가 고정된 정수 대용량 데이터?
  → Radix Sort

언어 기본 라이브러리를 쓸 수 있다?
  → 그냥 쓰세요 (Python: Tim Sort, C++: Intro Sort)
```

---

## 13. 인트로 정렬 (Introsort) — 보너스

C++ STL의 `std::sort`에서 사용하는 알고리즘으로, 세 가지 정렬을 상황에 따라 조합합니다.

| 상황 | 사용 알고리즘 |
|------|--------------|
| 일반 케이스 | Quick Sort |
| 재귀 깊이 > 2 log n | Heap Sort (O(n²) 방지) |
| 남은 원소 ≤ 16 | Insertion Sort |

Quick Sort의 평균 성능, Heap Sort의 최악 케이스 보장, Insertion Sort의 소규모 효율을 모두 취한 실용적 설계입니다.

---

## 정리

정렬 알고리즘 하나를 고를 때 **"가장 빠른 것"** 을 찾기보다 데이터의 특성과 제약 조건을 먼저 파악하는 것이 중요합니다. 대부분의 경우 언어 표준 라이브러리가 최선의 선택이지만, 각 알고리즘의 동작 원리와 복잡도를 이해하면 병목이 발생했을 때 더 나은 판단을 내릴 수 있습니다.

> **면접 팁**: "Quick Sort의 최악 케이스는 O(n²)인데 왜 실전에서 자주 쓰나요?"  
> → 평균 케이스가 O(n log n)이고, 캐시 지역성이 좋아 상수 계수가 작습니다. 랜덤 피벗이나 Median-of-3으로 최악 케이스 발생을 사실상 막을 수 있습니다.
