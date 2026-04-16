---
title: "[CS스터디] 자료구조 완전 정복 — 배열부터 그래프까지"
excerpt: "배열, 연결 리스트(단일·이중), 스택, 큐, 트리, 힙, 해시 테이블, 그래프, 트라이의 원리·복잡도·구현을 한 번에 정리합니다."
categories:
  - CS스터디
tags:
  - 자료구조
  - 연결리스트
  - 트리
  - 해시테이블
  - 그래프
  - 면접준비
  - 시간복잡도
date: 2026-04-16 00:00:00 +0900
last_modified_at: 2026-04-16 00:00:00 +0900
toc: true
toc_sticky: true
author_profile: true
comments: true
---

> CS 기초 스터디 — 컴퓨터공학과 전공 수준의 핵심 자료구조를 동작 원리, 시간·공간 복잡도, 구현 코드, 실전 활용처 중심으로 정리한 글입니다.

---

## 1. 자료구조란?

**자료구조(Data Structure)**는 데이터를 효율적으로 저장하고 접근·수정·삭제할 수 있도록 조직화하는 방법입니다. 올바른 자료구조를 선택하면 알고리즘의 시간·공간 복잡도를 대폭 줄일 수 있습니다.

### 분류 체계

```
자료구조
├── 선형 (Linear)
│   ├── 배열 (Array)
│   ├── 연결 리스트 (Linked List)
│   │   ├── 단일 연결 리스트 (Singly Linked List)
│   │   ├── 이중 연결 리스트 (Doubly Linked List)
│   │   └── 원형 연결 리스트 (Circular Linked List)
│   ├── 스택 (Stack)
│   └── 큐 (Queue) / 덱 (Deque)
└── 비선형 (Non-linear)
    ├── 트리 (Tree)
    │   ├── 이진 탐색 트리 (BST)
    │   ├── 힙 (Heap)
    │   ├── AVL / Red-Black Tree
    │   └── 트라이 (Trie)
    ├── 해시 테이블 (Hash Table)
    └── 그래프 (Graph)
```

### 핵심 용어 정리

| 용어 | 설명 |
|------|------|
| **시간 복잡도** | 연산(삽입·삭제·탐색)에 필요한 연산 횟수 |
| **공간 복잡도** | 자료구조가 사용하는 메모리 |
| **접근 (Access)** | 특정 위치의 원소를 읽는 연산 |
| **탐색 (Search)** | 특정 값을 가진 원소를 찾는 연산 |
| **삽입 (Insert)** | 새 원소를 추가하는 연산 |
| **삭제 (Delete)** | 원소를 제거하는 연산 |

---

## 2. 한눈에 보는 복잡도 비교표

| 자료구조 | 접근 | 탐색 | 삽입 | 삭제 | 공간 |
|----------|------|------|------|------|------|
| 배열 | O(1) | O(n) | O(n) | O(n) | O(n) |
| 단일 연결 리스트 | O(n) | O(n) | O(1)* | O(1)* | O(n) |
| 이중 연결 리스트 | O(n) | O(n) | O(1)* | O(1)* | O(n) |
| 스택 | O(n) | O(n) | O(1) | O(1) | O(n) |
| 큐 | O(n) | O(n) | O(1) | O(1) | O(n) |
| 이진 탐색 트리 | O(log n)† | O(log n)† | O(log n)† | O(log n)† | O(n) |
| 힙 | — | O(n) | O(log n) | O(log n) | O(n) |
| 해시 테이블 | — | O(1)‡ | O(1)‡ | O(1)‡ | O(n) |
| 그래프 (인접 행렬) | O(1) | O(V) | O(1) | O(1) | O(V²) |
| 그래프 (인접 리스트) | O(V+E) | O(V+E) | O(1) | O(E) | O(V+E) |

> \* 포인터를 이미 알고 있을 때 / † 균형 트리 기준 / ‡ 평균 (최악 O(n))

---

## 3. 선형 자료구조 (Linear Data Structures)

> 데이터가 **순서대로 나열**되는 구조입니다. 원소 간에 앞뒤 관계(1:1 관계)가 존재합니다.

---

### 3-1. 배열 (Array)

#### 동작 원리

배열은 **연속된 메모리 공간**에 동일한 타입의 원소를 저장합니다. 인덱스로 O(1) 임의 접근이 가능하며, 가장 기본적인 자료구조입니다.

```
index:  0    1    2    3    4
value: [10] [20] [30] [40] [50]
       ↑
       base address (예: 0x1000)
       arr[i] = base + i × element_size
```

#### 코드 (Python)

```python
# 정적 배열 (고정 크기) 대신 Python의 동적 배열(list) 사용
arr = [10, 20, 30, 40, 50]

# O(1) 접근
print(arr[2])          # 30

# O(n) 삽입 — 중간 삽입 시 뒤 원소들을 모두 밀어야 함
arr.insert(2, 99)      # [10, 20, 99, 30, 40, 50]

# O(1) 끝에 추가 (amortized)
arr.append(60)

# O(n) 삭제 — 중간 삭제 시 뒤 원소들을 당겨야 함
arr.pop(2)             # index 2 제거
```

#### 특징

- **장점**: 임의 접근 O(1), 캐시 지역성 우수, 구조 단순
- **단점**: 삽입·삭제 O(n) (중간), 크기 변경 시 재할당 비용
- **동적 배열(ArrayList/Python list)**: 용량 초과 시 2배 확장 → 삽입 amortized O(1)

---

### 3-2. 단일 연결 리스트 (Singly Linked List)

#### 동작 원리

각 노드가 **데이터 + 다음 노드를 가리키는 포인터(next)**로 구성됩니다. 메모리가 비연속적으로 분산되어 있어 임의 접근은 불가하지만, 포인터 조작으로 삽입·삭제가 O(1)입니다.

```
HEAD
 ↓
[10 | •]→[20 | •]→[30 | •]→[40 | None]
```

#### 코드 (Python)

```python
class Node:
    def __init__(self, data):
        self.data = data
        self.next = None

class SinglyLinkedList:
    def __init__(self):
        self.head = None

    # O(1) 맨 앞 삽입
    def prepend(self, data):
        node = Node(data)
        node.next = self.head
        self.head = node

    # O(n) 맨 뒤 삽입
    def append(self, data):
        node = Node(data)
        if not self.head:
            self.head = node
            return
        cur = self.head
        while cur.next:
            cur = cur.next
        cur.next = node

    # O(n) 탐색
    def search(self, target):
        cur = self.head
        while cur:
            if cur.data == target:
                return cur
            cur = cur.next
        return None

    # O(n) 삭제
    def delete(self, target):
        if not self.head:
            return
        if self.head.data == target:
            self.head = self.head.next
            return
        cur = self.head
        while cur.next and cur.next.data != target:
            cur = cur.next
        if cur.next:
            cur.next = cur.next.next

    def __str__(self):
        result, cur = [], self.head
        while cur:
            result.append(str(cur.data))
            cur = cur.next
        return " → ".join(result) + " → None"
```

#### 특징

- **장점**: 삽입·삭제 O(1) (포인터를 알고 있을 때), 크기 동적 변경 용이
- **단점**: 임의 접근 불가 O(n), 포인터 공간 오버헤드, 역방향 순회 불가

---

### 3-3. 이중 연결 리스트 (Doubly Linked List)

#### 동작 원리

각 노드가 **데이터 + 이전 노드(prev) + 다음 노드(next)** 포인터를 모두 가집니다. 단일 연결 리스트의 단점인 **역방향 순회 불가**와 **삭제 시 이전 노드 탐색 필요** 문제를 해결합니다.

```
      HEAD                         TAIL
       ↓                            ↓
None←[10|•]⇄[20|•]⇄[30|•]⇄[40|None]
     prev next
```

#### 코드 (Python)

```python
class DNode:
    def __init__(self, data):
        self.data = data
        self.prev = None
        self.next = None

class DoublyLinkedList:
    def __init__(self):
        self.head = None
        self.tail = None
        self.size = 0

    # O(1) 맨 앞 삽입
    def prepend(self, data):
        node = DNode(data)
        if not self.head:
            self.head = self.tail = node
        else:
            node.next = self.head
            self.head.prev = node
            self.head = node
        self.size += 1

    # O(1) 맨 뒤 삽입
    def append(self, data):
        node = DNode(data)
        if not self.tail:
            self.head = self.tail = node
        else:
            node.prev = self.tail
            self.tail.next = node
            self.tail = node
        self.size += 1

    # O(1) 특정 노드 뒤에 삽입 (노드 포인터 알고 있을 때)
    def insert_after(self, ref_node, data):
        if ref_node is self.tail:
            self.append(data)
            return
        node = DNode(data)
        node.prev = ref_node
        node.next = ref_node.next
        ref_node.next.prev = node
        ref_node.next = node
        self.size += 1

    # O(1) 특정 노드 삭제 (노드 포인터 알고 있을 때)
    def delete_node(self, node):
        if node.prev:
            node.prev.next = node.next
        else:               # node가 head인 경우
            self.head = node.next
        if node.next:
            node.next.prev = node.prev
        else:               # node가 tail인 경우
            self.tail = node.prev
        self.size -= 1

    # O(n) 값으로 탐색 후 삭제
    def delete(self, target):
        cur = self.head
        while cur:
            if cur.data == target:
                self.delete_node(cur)
                return True
            cur = cur.next
        return False

    # 정방향 순회
    def forward(self):
        result, cur = [], self.head
        while cur:
            result.append(str(cur.data))
            cur = cur.next
        return " ⇄ ".join(result)

    # 역방향 순회 — 이중 연결 리스트의 강점
    def backward(self):
        result, cur = [], self.tail
        while cur:
            result.append(str(cur.data))
            cur = cur.prev
        return " ⇄ ".join(result)
```

#### 단일 vs 이중 연결 리스트 비교

| 항목 | 단일 연결 리스트 | 이중 연결 리스트 |
|------|----------------|----------------|
| 노드 포인터 수 | 1개 (next) | 2개 (prev + next) |
| 역방향 순회 | ❌ 불가 | ✅ O(n) |
| 특정 노드 삭제 | O(n) — 이전 노드 탐색 필요 | O(1) — prev 포인터로 즉시 접근 |
| 메모리 사용 | 더 적음 | next 당 포인터 1개 추가 |
| 맨 뒤 삽입 | O(n) (tail 없을 때) | O(1) (tail 포인터 보유) |
| 구현 복잡도 | 단순 | 약간 복잡 (prev 유지 필요) |

#### 실전 활용

- **브라우저 방문 기록** — 앞으로/뒤로 이동
- **텍스트 에디터 커서** — 양방향 이동
- **LRU 캐시** — 해시맵 + 이중 연결 리스트 조합
- **Python `collections.deque`** 내부 구현

---

### 3-4. 스택 (Stack)

#### 동작 원리

**LIFO(Last In, First Out)** 구조입니다. 가장 나중에 들어온 원소가 가장 먼저 나갑니다. 접시 더미를 생각하면 이해하기 쉽습니다.

```
push(1) → push(2) → push(3)

top → [3]
      [2]
      [1]
      ──

pop() → 3 반환
top → [2]
      [1]
```

#### 코드 (Python)

```python
class Stack:
    def __init__(self):
        self._data = []

    def push(self, item):         # O(1) amortized
        self._data.append(item)

    def pop(self):                # O(1)
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self._data.pop()

    def peek(self):               # O(1) — 제거 없이 top 확인
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self._data[-1]

    def is_empty(self):
        return len(self._data) == 0

    def __len__(self):
        return len(self._data)
```

#### 대표 활용

| 용도 | 설명 |
|------|------|
| **함수 호출 스택** | 재귀 호출, 지역 변수 관리 |
| **괄호 유효성 검사** | `()[]{}`가 올바르게 닫히는지 확인 |
| **DFS** | 그래프 깊이 우선 탐색 (반복 구현) |
| **역순 출력** | 문자열·수열 뒤집기 |
| **수식 계산** | 후위 표기법(Postfix) 계산 |

---

### 3-5. 큐 (Queue) & 덱 (Deque)

#### 동작 원리

**큐**: **FIFO(First In, First Out)** 구조. 먼저 들어온 원소가 먼저 나갑니다. 줄 서기와 동일합니다.

```
enqueue(1) → enqueue(2) → enqueue(3)

FRONT → [1][2][3] ← REAR

dequeue() → 1 반환
FRONT → [2][3] ← REAR
```

**덱(Deque, Double-Ended Queue)**: 양쪽 끝에서 삽입·삭제가 모두 가능합니다.

#### 코드 (Python)

```python
from collections import deque

# 큐
queue = deque()
queue.append(1)       # enqueue — O(1)
queue.append(2)
queue.append(3)
queue.popleft()       # dequeue → 1, O(1)

# 덱
dq = deque()
dq.append(1)          # 오른쪽 추가 — O(1)
dq.appendleft(0)      # 왼쪽 추가 — O(1)
dq.pop()              # 오른쪽 제거 — O(1)
dq.popleft()          # 왼쪽 제거 — O(1)
```

> Python의 `list`로 큐를 구현하면 `pop(0)`이 O(n)입니다. 반드시 `collections.deque`를 사용하세요.

### 순환 큐 (Circular Queue)

배열로 큐를 구현할 때 앞 공간이 낭비되는 문제를 해결합니다.

```python
class CircularQueue:
    def __init__(self, capacity):
        self.capacity = capacity + 1   # 빈 칸 구분용
        self.data = [None] * self.capacity
        self.front = 0
        self.rear = 0

    def is_empty(self):
        return self.front == self.rear

    def is_full(self):
        return (self.rear + 1) % self.capacity == self.front

    def enqueue(self, item):          # O(1)
        if self.is_full():
            raise OverflowError("Queue is full")
        self.data[self.rear] = item
        self.rear = (self.rear + 1) % self.capacity

    def dequeue(self):                # O(1)
        if self.is_empty():
            raise IndexError("Queue is empty")
        item = self.data[self.front]
        self.front = (self.front + 1) % self.capacity
        return item
```

#### 대표 활용

- **BFS** — 그래프 너비 우선 탐색
- **프로세스 스케줄링** — CPU 라운드 로빈
- **프린터 스풀링** — 작업 순서 보장
- **슬라이딩 윈도우** — 덱을 활용한 최솟값/최댓값 O(n) 계산

---

## 4. 비선형 자료구조 (Non-linear Data Structures)

> 데이터 간에 **계층적·망형 관계(1:N 또는 N:M)**가 존재합니다. 순서 대신 구조적 관계를 표현합니다.

---

### 4-1. 트리 (Tree)

#### 기본 개념

트리는 **계층적(hierarchical) 구조**를 가진 비선형 자료구조입니다. 사이클이 없는 연결 그래프로, 루트(root)에서 자식(child) 방향으로 뻗어 나갑니다.

```
         [1]          ← 루트 (Root)
        /   \
      [2]   [3]       ← 내부 노드 (Internal Node)
     /   \     \
   [4]   [5]   [6]    ← 리프 (Leaf)
```

| 용어 | 설명 |
|------|------|
| **높이 (Height)** | 루트에서 가장 먼 리프까지의 거리 |
| **깊이 (Depth)** | 루트에서 해당 노드까지의 거리 |
| **차수 (Degree)** | 노드의 자식 수 |
| **완전 이진 트리** | 마지막 레벨 제외 모두 가득, 마지막은 왼쪽부터 채움 |
| **포화 이진 트리** | 모든 리프가 같은 깊이이고 모든 노드가 자식 2개 |

#### 이진 탐색 트리 (BST)

**BST 규칙**: 왼쪽 자식 < 현재 노드 < 오른쪽 자식

```python
class BSTNode:
    def __init__(self, key):
        self.key = key
        self.left = None
        self.right = None

class BST:
    def __init__(self):
        self.root = None

    # O(log n) 평균, O(n) 최악
    def insert(self, key):
        self.root = self._insert(self.root, key)

    def _insert(self, node, key):
        if not node:
            return BSTNode(key)
        if key < node.key:
            node.left = self._insert(node.left, key)
        elif key > node.key:
            node.right = self._insert(node.right, key)
        return node

    # O(log n) 평균, O(n) 최악
    def search(self, key):
        return self._search(self.root, key)

    def _search(self, node, key):
        if not node or node.key == key:
            return node
        if key < node.key:
            return self._search(node.left, key)
        return self._search(node.right, key)

    # 중위 순회 — BST에서는 오름차순 출력
    def inorder(self, node=None, first=True):
        if first:
            node = self.root
        if not node:
            return []
        return (self.inorder(node.left, False)
                + [node.key]
                + self.inorder(node.right, False))
```

#### 트리 순회 (Tree Traversal)

```
        [4]
       /   \
     [2]   [6]
    /  \   /  \
  [1] [3] [5] [7]
```

| 순회 방식 | 방문 순서 | 결과 | 주요 용도 |
|-----------|----------|------|-----------|
| **전위 (Preorder)** | 루트 → 왼 → 오 | 4,2,1,3,6,5,7 | 트리 복사, 직렬화 |
| **중위 (Inorder)** | 왼 → 루트 → 오 | 1,2,3,4,5,6,7 | BST 정렬 출력 |
| **후위 (Postorder)** | 왼 → 오 → 루트 | 1,3,2,5,7,6,4 | 트리 삭제, 수식 계산 |
| **레벨 순서 (BFS)** | 레벨 단위 좌→우 | 4,2,6,1,3,5,7 | 최단 경로, 레벨 탐색 |

#### BST의 한계 — 불균형 문제

```
편향 트리 (최악 케이스):
[1]
  \
  [2]
    \
    [3]       ← 탐색 O(n)
      \
      [4]
```

정렬된 데이터를 순서대로 삽입하면 편향 트리가 됩니다. 이를 해결하기 위해 **AVL 트리**, **Red-Black 트리** 같은 자기 균형 BST를 사용합니다.

---

### 4-2. 힙 (Heap)

#### 동작 원리

**완전 이진 트리** 기반의 자료구조로, **힙 속성(Heap Property)**을 유지합니다.

- **최대 힙(Max Heap)**: 부모 노드 ≥ 자식 노드 → 루트가 최댓값
- **최소 힙(Min Heap)**: 부모 노드 ≤ 자식 노드 → 루트가 최솟값

```
Max Heap:
         [50]
        /    \
      [30]  [40]
      / \   /
    [10][20][35]

배열 표현: [50, 30, 40, 10, 20, 35]
인덱스 i의 자식: 2i+1 (왼쪽), 2i+2 (오른쪽)
인덱스 i의 부모: (i-1) // 2
```

#### 코드 (Python)

```python
import heapq  # Python은 최소 힙 내장

# 최소 힙
heap = []
heapq.heappush(heap, 5)    # O(log n) 삽입
heapq.heappush(heap, 1)
heapq.heappush(heap, 3)
print(heapq.heappop(heap)) # O(log n) 최솟값 추출 → 1
print(heap[0])             # O(1) 최솟값 확인 → 3

# 최대 힙 — 값에 음수 부호 적용
max_heap = []
for v in [5, 1, 3]:
    heapq.heappush(max_heap, -v)
print(-heapq.heappop(max_heap))  # → 5
```

#### 직접 구현 (Min Heap)

```python
class MinHeap:
    def __init__(self):
        self.heap = []

    def push(self, val):              # O(log n)
        self.heap.append(val)
        self._sift_up(len(self.heap) - 1)

    def pop(self):                    # O(log n)
        if len(self.heap) == 1:
            return self.heap.pop()
        root = self.heap[0]
        self.heap[0] = self.heap.pop()
        self._sift_down(0)
        return root

    def _sift_up(self, i):
        parent = (i - 1) // 2
        if i > 0 and self.heap[i] < self.heap[parent]:
            self.heap[i], self.heap[parent] = self.heap[parent], self.heap[i]
            self._sift_up(parent)

    def _sift_down(self, i):
        n = len(self.heap)
        smallest = i
        left, right = 2 * i + 1, 2 * i + 2
        if left < n and self.heap[left] < self.heap[smallest]:
            smallest = left
        if right < n and self.heap[right] < self.heap[smallest]:
            smallest = right
        if smallest != i:
            self.heap[i], self.heap[smallest] = self.heap[smallest], self.heap[i]
            self._sift_down(smallest)
```

#### 대표 활용

- **우선순위 큐(Priority Queue)** — 항상 최솟값/최댓값에 O(1) 접근
- **힙 정렬** — O(n log n), 추가 메모리 O(1)
- **다익스트라(Dijkstra)** 최단 경로 알고리즘
- **k번째 최솟값/최댓값** 추출

---

### 4-3. 그래프 (Graph)

#### 기본 개념

**정점(Vertex, V)**과 **간선(Edge, E)**으로 이루어진 자료구조입니다. 트리는 그래프의 특수한 경우입니다.

```
방향 그래프 (Directed):       무방향 그래프 (Undirected):
A → B                         A ─ B
↓   ↓                         |   |
C → D                         C ─ D
```

#### 표현 방법

#### 인접 행렬 (Adjacency Matrix)

```python
# V×V 크기 행렬, matrix[i][j] = 1이면 i→j 간선 존재
V = 4
matrix = [[0] * V for _ in range(V)]
# A=0, B=1, C=2, D=3
matrix[0][1] = 1  # A→B
matrix[0][2] = 1  # A→C
matrix[1][3] = 1  # B→D
```

#### 인접 리스트 (Adjacency List)

```python
from collections import defaultdict

graph = defaultdict(list)
graph[0].append(1)  # A→B
graph[0].append(2)  # A→C
graph[1].append(3)  # B→D
```

| 항목 | 인접 행렬 | 인접 리스트 |
|------|----------|-----------|
| 공간 | O(V²) | O(V + E) |
| 간선 확인 | O(1) | O(차수) |
| 이웃 탐색 | O(V) | O(차수) |
| 적합한 경우 | 밀집 그래프 | 희소 그래프 |

#### BFS & DFS 구현

```python
from collections import deque

def bfs(graph, start):
    visited = set([start])
    queue = deque([start])
    order = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return order

def dfs(graph, start, visited=None):
    if visited is None:
        visited = set()
    visited.add(start)
    order = [start]
    for neighbor in graph[start]:
        if neighbor not in visited:
            order.extend(dfs(graph, neighbor, visited))
    return order
```

#### 대표 알고리즘

| 알고리즘 | 시간 복잡도 | 용도 |
|----------|-----------|------|
| **BFS** | O(V + E) | 최단 경로 (가중치 없는), 레벨 탐색 |
| **DFS** | O(V + E) | 사이클 감지, 위상 정렬, 연결 요소 |
| **다익스트라** | O((V+E) log V) | 단일 출발 최단 경로 (양수 가중치) |
| **벨만-포드** | O(VE) | 최단 경로 (음수 가중치 허용) |
| **플로이드-워셜** | O(V³) | 모든 쌍 최단 경로 |
| **크루스칼/프림** | O(E log E) | 최소 신장 트리 (MST) |

---

### 4-4. 트라이 (Trie)

#### 동작 원리

**문자열 검색에 특화**된 트리 자료구조입니다. 각 노드가 문자(character)를 나타내며, 루트에서 리프까지의 경로가 하나의 문자열을 표현합니다.

```
삽입: "cat", "car", "card", "care", "bat"

         [root]
        /      \
      [c]      [b]
       |         |
      [a]       [a]
     /   \        \
   [t*]  [r]      [t*]
         / \
       [d*][e*]

* = 단어 종료 표시
```

#### 코드 (Python)

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    # O(L) — L: 문자열 길이
    def insert(self, word):
        node = self.root
        for ch in word:
            if ch not in node.children:
                node.children[ch] = TrieNode()
            node = node.children[ch]
        node.is_end = True

    # O(L)
    def search(self, word):
        node = self.root
        for ch in word:
            if ch not in node.children:
                return False
            node = node.children[ch]
        return node.is_end

    # O(L) — 해당 접두사로 시작하는 단어 존재 여부
    def starts_with(self, prefix):
        node = self.root
        for ch in prefix:
            if ch not in node.children:
                return False
            node = node.children[ch]
        return True

    # 해당 접두사로 시작하는 모든 단어 반환
    def autocomplete(self, prefix):
        node = self.root
        for ch in prefix:
            if ch not in node.children:
                return []
            node = node.children[ch]
        results = []
        self._dfs(node, list(prefix), results)
        return results

    def _dfs(self, node, path, results):
        if node.is_end:
            results.append("".join(path))
        for ch, child in node.children.items():
            path.append(ch)
            self._dfs(child, path, results)
            path.pop()
```

#### 특징

| 항목 | 해시맵 | 트라이 |
|------|--------|-------|
| 단어 정확 탐색 | O(L) 평균 | O(L) |
| 접두사 탐색 | O(n×L) | O(L) |
| 공간 | O(n×L) | O(알파벳크기×L×n) |
| 정렬된 순회 | 불가 | O(n×L) |

- **자동완성(Autocomplete)**, **맞춤법 검사**, **IP 라우팅** 테이블에 활용
- 메모리 사용이 크다는 단점 → **압축 트라이(Compressed Trie)** 로 개선 가능

---

## 5. 해시 기반 자료구조 (Hash-based Data Structures)

> **해시 함수**를 이용해 키를 인덱스로 변환하여 O(1) 평균 접근을 달성합니다. 선형도 비선형도 아닌 독립적인 분류로 다루는 것이 일반적입니다.

---

### 5-1. 해시 테이블 (Hash Table)

#### 동작 원리

**해시 함수(Hash Function)**로 키(Key)를 정수 인덱스로 변환하고, 그 인덱스에 값을 저장합니다. 평균 O(1) 삽입·탐색·삭제가 가능한 강력한 자료구조입니다.

```
key = "alice"
hash("alice") % 8 = 3

index: 0   1   2   3        4   5   6   7
       [ ] [ ] [ ] ["alice":25] [ ] [ ] [ ] [ ]
```

#### 충돌 해결 방법

**충돌(Collision)**: 서로 다른 키가 같은 인덱스로 매핑되는 경우

#### 1. 체이닝 (Chaining) — 연결 리스트로 연결

```python
class HashTableChaining:
    def __init__(self, size=8):
        self.size = size
        self.buckets = [[] for _ in range(size)]

    def _hash(self, key):
        return hash(key) % self.size

    def put(self, key, value):        # O(1) 평균
        idx = self._hash(key)
        for i, (k, v) in enumerate(self.buckets[idx]):
            if k == key:
                self.buckets[idx][i] = (key, value)
                return
        self.buckets[idx].append((key, value))

    def get(self, key):               # O(1) 평균
        idx = self._hash(key)
        for k, v in self.buckets[idx]:
            if k == key:
                return v
        return None

    def delete(self, key):            # O(1) 평균
        idx = self._hash(key)
        self.buckets[idx] = [(k, v) for k, v in self.buckets[idx] if k != key]
```

#### 2. 개방 주소법 (Open Addressing) — 빈 슬롯 탐색

| 방법 | 탐색 순서 | 특징 |
|------|----------|------|
| **선형 탐사** | h+1, h+2, … | 구현 단순, 클러스터링 발생 |
| **이차 탐사** | h+1², h+2², … | 클러스터링 감소 |
| **이중 해싱** | h + i×h₂(k) | 클러스터링 최소화 |

#### 부하율 (Load Factor)

```
부하율 α = 저장된 원소 수 / 테이블 크기

α > 0.75 → 테이블 크기를 2배로 재해싱 (Rehashing)
```

부하율이 높아질수록 충돌이 증가해 성능이 O(n)에 가까워집니다.

#### Python `dict`

Python의 `dict`는 해시 테이블로 구현되어 있으며 개방 주소법(이중 해싱 변형)을 사용합니다.

```python
# Python dict = 해시 테이블
d = {}
d["alice"] = 25        # O(1) 삽입
print(d["alice"])      # O(1) 탐색
del d["alice"]         # O(1) 삭제
print("alice" in d)    # O(1) 존재 여부
```

---

## 6. 자료구조 선택 가이드

```
상황에 따른 자료구조 선택:

빠른 임의 접근이 필요하다?
  → 배열 (Array) — O(1) 접근

중간 삽입·삭제가 빈번하다?
  → 이중 연결 리스트 (Doubly Linked List) — O(1) 삽입/삭제

LIFO 구조가 필요하다? (재귀, 괄호 검증)
  → 스택 (Stack)

FIFO 구조가 필요하다? (BFS, 스케줄링)
  → 큐 (Queue) / 덱 (Deque)

항상 최솟값/최댓값에 빠르게 접근해야 한다?
  → 힙 (Heap) — O(log n) 삽입/삭제, O(1) 최솟값

키-값 매핑이 필요하고 탐색이 빨라야 한다?
  → 해시 테이블 (Hash Table) — O(1) 평균

정렬된 순서로 탐색·삽입·삭제가 모두 O(log n) 필요?
  → 이진 탐색 트리 (BST) / AVL / Red-Black Tree

문자열 접두사 탐색 또는 자동완성이 필요하다?
  → 트라이 (Trie)

관계(경로, 연결) 표현이 필요하다?
  → 그래프 (Graph)
    - 간선이 적으면 인접 리스트
    - 간선이 많으면 인접 행렬
```

---

## 7. 면접 핵심 질문 정리

**Q. 배열과 연결 리스트의 차이는 무엇인가요?**
> 배열은 연속 메모리에 저장되어 임의 접근 O(1)이 가능하지만 삽입·삭제가 O(n)입니다. 연결 리스트는 포인터로 연결된 분산 메모리로 임의 접근이 O(n)이지만, 포인터를 알고 있을 때 삽입·삭제가 O(1)입니다.

**Q. 이중 연결 리스트는 언제 단일 연결 리스트보다 유리한가요?**
> 역방향 순회가 필요하거나, 포인터를 알고 있는 특정 노드를 O(1)에 삭제해야 할 때 유리합니다. LRU 캐시 구현이 대표적인 예로, 해시맵으로 노드 위치를 O(1)에 찾고 이중 연결 리스트로 O(1)에 삭제·이동합니다.

**Q. 해시 테이블의 최악 케이스는 왜 O(n)인가요?**
> 모든 키가 같은 버킷으로 해시될 경우(충돌이 최대인 경우) 체이닝 구조에서 탐색이 O(n)으로 저하됩니다. 좋은 해시 함수 설계와 부하율 관리(재해싱)로 평균 O(1)을 유지합니다.

**Q. 스택과 큐를 각각 다른 하나로 구현하려면?**
> 큐 2개로 스택 구현: push 시 빈 큐에 추가 후 나머지 큐 전체를 이동 (O(n)). 스택 2개로 큐 구현: enqueue는 스택1에 push(O(1)), dequeue 시 스택1이 비면 전체를 스택2로 이동 (amortized O(1)).

**Q. BST vs 해시 테이블 — 언제 무엇을 선택하나요?**
> 해시 테이블은 정확한 키 탐색에 O(1)로 최적이지만, 범위 탐색·정렬된 순회가 불가합니다. BST(균형)는 O(log n)이지만 범위 탐색, 순서 기반 연산이 가능합니다.
