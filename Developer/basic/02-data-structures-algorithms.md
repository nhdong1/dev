# Data Structures & Algorithms - Interview Questions

## Question 1: Giải thích Big O Notation và phân tích complexity của các operations phổ biến?

**Answer:**

**Big O Notation** mô tả upper bound của algorithm's growth rate khi input size tăng.

**Common Complexities (từ tốt → xấu):**

| Complexity | Name | Example |
|------------|------|---------|
| O(1) | Constant | Array access, hash table lookup |
| O(log n) | Logarithmic | Binary search |
| O(n) | Linear | Linear search, array traversal |
| O(n log n) | Linearithmic | Merge sort, heap sort |
| O(n²) | Quadratic | Bubble sort, nested loops |
| O(2^n) | Exponential | Recursive fibonacci, subset generation |
| O(n!) | Factorial | Permutations |

**Data Structure Operations:**

```
Array:
- Access: O(1)
- Search: O(n), O(log n) if sorted
- Insert/Delete at end: O(1) amortized
- Insert/Delete at index: O(n)

Hash Table:
- Average: O(1) for insert, delete, search
- Worst case: O(n) với poor hash function

Binary Search Tree (balanced):
- Search, Insert, Delete: O(log n)

Heap:
- Insert: O(log n)
- Extract min/max: O(log n)
- Get min/max: O(1)
```

**Amortized Analysis:**
```
Dynamic Array resize:
- Most inserts: O(1)
- Occasional resize: O(n)
- Amortized: O(1) per operation
```

**Space Complexity:**
Đo lường extra memory algorithm sử dụng (không tính input).
```
In-place sort (quicksort): O(log n) stack space
Merge sort: O(n) auxiliary space
```

---

## Question 2: Implement và giải thích các Tree Traversal algorithms?

**Answer:**

**Binary Tree Node:**
```python
class TreeNode:
    def __init__(self, val):
        self.val = val
        self.left = None
        self.right = None
```

**1. Depth-First Search (DFS) Traversals:**

```python
# Inorder: Left → Root → Right (cho BST: sorted order)
def inorder(node):
    if node:
        inorder(node.left)
        print(node.val)
        inorder(node.right)

# Preorder: Root → Left → Right (copy tree, prefix expression)
def preorder(node):
    if node:
        print(node.val)
        preorder(node.left)
        preorder(node.right)

# Postorder: Left → Right → Root (delete tree, postfix expression)
def postorder(node):
    if node:
        postorder(node.left)
        postorder(node.right)
        print(node.val)
```

**Iterative Inorder (using stack):**
```python
def inorder_iterative(root):
    stack = []
    current = root
    result = []

    while stack or current:
        while current:
            stack.append(current)
            current = current.left
        current = stack.pop()
        result.append(current.val)
        current = current.right

    return result
```

**2. Breadth-First Search (BFS) - Level Order:**
```python
from collections import deque

def level_order(root):
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)
        level = []

        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(level)

    return result
```

**Time & Space Complexity:**
- All traversals: O(n) time
- Recursive DFS: O(h) space (h = height, worst O(n))
- BFS: O(w) space (w = max width)

---

## Question 3: Giải thích Graph algorithms: BFS, DFS, Dijkstra?

**Answer:**

**Graph Representation:**
```python
# Adjacency List (preferred for sparse graphs)
graph = {
    'A': [('B', 1), ('C', 4)],
    'B': [('C', 2), ('D', 5)],
    'C': [('D', 1)],
    'D': []
}
```

**1. BFS (Breadth-First Search):**
```python
from collections import deque

def bfs(graph, start):
    visited = set([start])
    queue = deque([start])

    while queue:
        node = queue.popleft()
        print(node)

        for neighbor, _ in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
```
- **Use cases**: Shortest path (unweighted), level-order traversal
- **Complexity**: O(V + E)

**2. DFS (Depth-First Search):**
```python
def dfs(graph, node, visited=None):
    if visited is None:
        visited = set()

    visited.add(node)
    print(node)

    for neighbor, _ in graph[node]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
```
- **Use cases**: Topological sort, cycle detection, connected components
- **Complexity**: O(V + E)

**3. Dijkstra's Algorithm (Single Source Shortest Path):**
```python
import heapq

def dijkstra(graph, start):
    distances = {node: float('inf') for node in graph}
    distances[start] = 0
    pq = [(0, start)]  # (distance, node)

    while pq:
        curr_dist, curr_node = heapq.heappop(pq)

        if curr_dist > distances[curr_node]:
            continue

        for neighbor, weight in graph[curr_node]:
            distance = curr_dist + weight

            if distance < distances[neighbor]:
                distances[neighbor] = distance
                heapq.heappush(pq, (distance, neighbor))

    return distances
```
- **Complexity**: O((V + E) log V) với min-heap
- **Limitation**: Không hoạt động với negative weights

**Other Important Algorithms:**
- **Bellman-Ford**: Handles negative weights, O(VE)
- **Floyd-Warshall**: All pairs shortest path, O(V³)
- **Topological Sort**: DAG ordering, O(V + E)
- **Kruskal/Prim**: Minimum Spanning Tree

---

## Question 4: Giải thích Hash Table implementation và collision resolution?

**Answer:**

**Hash Table Components:**

1. **Hash Function**: Maps key → array index
   ```python
   def hash(key, size):
       return hash(key) % size
   ```

2. **Array**: Stores key-value pairs at computed indices

**Collision Resolution Methods:**

**1. Chaining (Separate Chaining):**
```python
class HashTable:
    def __init__(self, size=10):
        self.size = size
        self.buckets = [[] for _ in range(size)]

    def put(self, key, value):
        index = hash(key) % self.size
        bucket = self.buckets[index]

        for i, (k, v) in enumerate(bucket):
            if k == key:
                bucket[i] = (key, value)
                return
        bucket.append((key, value))

    def get(self, key):
        index = hash(key) % self.size
        bucket = self.buckets[index]

        for k, v in bucket:
            if k == key:
                return v
        raise KeyError(key)
```
- **Pros**: Simple, handles high load factor
- **Cons**: Extra memory for linked lists

**2. Open Addressing (Linear Probing):**
```python
class HashTableOpenAddressing:
    def __init__(self, size=10):
        self.size = size
        self.keys = [None] * size
        self.values = [None] * size

    def put(self, key, value):
        index = hash(key) % self.size

        while self.keys[index] is not None:
            if self.keys[index] == key:
                self.values[index] = value
                return
            index = (index + 1) % self.size  # Linear probing

        self.keys[index] = key
        self.values[index] = value
```

**Other Open Addressing Methods:**
- **Quadratic Probing**: index = (h + c₁i + c₂i²) % size
- **Double Hashing**: index = (h₁ + i * h₂) % size

**Load Factor & Resizing:**
```
Load Factor = n / size (n = number of entries)
When load factor > 0.75 → resize (double size)
Rehash all existing entries
```

**Good Hash Function Properties:**
- Deterministic
- Uniform distribution
- Fast to compute

---

## Question 5: Dynamic Programming: Giải thích approach và solve Longest Common Subsequence?

**Answer:**

**Dynamic Programming** là technique giải problems bằng cách:
1. Breaking down thành overlapping subproblems
2. Storing solutions (memoization/tabulation)
3. Building up to final solution

**Two Approaches:**

**1. Top-Down (Memoization):**
```python
def fib_memo(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib_memo(n-1, memo) + fib_memo(n-2, memo)
    return memo[n]
```

**2. Bottom-Up (Tabulation):**
```python
def fib_tab(n):
    if n <= 1:
        return n
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i-1] + dp[i-2]
    return dp[n]
```

**Longest Common Subsequence (LCS) Problem:**

Given: `text1 = "abcde"`, `text2 = "ace"`
Output: `3` (LCS = "ace")

**Solution:**
```python
def lcs(text1, text2):
    m, n = len(text1), len(text2)

    # dp[i][j] = LCS length of text1[0:i] and text2[0:j]
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i-1] == text2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])

    return dp[m][n]
```

**Recurrence Relation:**
```
if text1[i] == text2[j]:
    dp[i][j] = dp[i-1][j-1] + 1
else:
    dp[i][j] = max(dp[i-1][j], dp[i][j-1])
```

**Complexity:**
- Time: O(m × n)
- Space: O(m × n), có thể optimize thành O(min(m, n))

**Common DP Problems:**
- Knapsack (0/1, unbounded)
- Longest Increasing Subsequence
- Edit Distance
- Coin Change
- Matrix Chain Multiplication

---

## Question 6: Implement và phân tích QuickSort vs MergeSort?

**Answer:**

**QuickSort:**
```python
def quicksort(arr, low=0, high=None):
    if high is None:
        high = len(arr) - 1

    if low < high:
        pivot_idx = partition(arr, low, high)
        quicksort(arr, low, pivot_idx - 1)
        quicksort(arr, pivot_idx + 1, high)

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

**MergeSort:**
```python
def mergesort(arr):
    if len(arr) <= 1:
        return arr

    mid = len(arr) // 2
    left = mergesort(arr[:mid])
    right = mergesort(arr[mid:])

    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0

    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1

    result.extend(left[i:])
    result.extend(right[j:])
    return result
```

**Comparison:**

| Aspect | QuickSort | MergeSort |
|--------|-----------|-----------|
| **Average Time** | O(n log n) | O(n log n) |
| **Worst Time** | O(n²) - sorted array | O(n log n) - always |
| **Space** | O(log n) in-place | O(n) auxiliary |
| **Stable** | No | Yes |
| **Cache** | Better locality | Worse (random access) |
| **Use case** | General purpose | Stable sort, linked lists |

**Optimizations:**

QuickSort:
- Random pivot selection
- Three-way partition (Dutch National Flag)
- Switch to insertion sort for small subarrays

MergeSort:
- Bottom-up iterative version
- Natural merge sort (detect sorted runs)

---

## Question 7: Giải thích Heap data structure và implement Priority Queue?

**Answer:**

**Heap** là complete binary tree thỏa mãn heap property:
- **Min-heap**: Parent ≤ children
- **Max-heap**: Parent ≥ children

**Array Representation:**
```
Parent of i: (i-1) // 2
Left child of i: 2*i + 1
Right child of i: 2*i + 2
```

**Min-Heap Implementation:**
```python
class MinHeap:
    def __init__(self):
        self.heap = []

    def push(self, val):
        self.heap.append(val)
        self._bubble_up(len(self.heap) - 1)

    def pop(self):
        if not self.heap:
            raise IndexError("Empty heap")

        if len(self.heap) == 1:
            return self.heap.pop()

        root = self.heap[0]
        self.heap[0] = self.heap.pop()
        self._bubble_down(0)
        return root

    def peek(self):
        return self.heap[0] if self.heap else None

    def _bubble_up(self, idx):
        parent = (idx - 1) // 2
        while idx > 0 and self.heap[idx] < self.heap[parent]:
            self.heap[idx], self.heap[parent] = self.heap[parent], self.heap[idx]
            idx = parent
            parent = (idx - 1) // 2

    def _bubble_down(self, idx):
        size = len(self.heap)
        while True:
            smallest = idx
            left = 2 * idx + 1
            right = 2 * idx + 2

            if left < size and self.heap[left] < self.heap[smallest]:
                smallest = left
            if right < size and self.heap[right] < self.heap[smallest]:
                smallest = right

            if smallest == idx:
                break

            self.heap[idx], self.heap[smallest] = self.heap[smallest], self.heap[idx]
            idx = smallest
```

**Complexity:**
- Push: O(log n)
- Pop: O(log n)
- Peek: O(1)
- Heapify (build heap): O(n)

**Priority Queue Use Cases:**
- Dijkstra's algorithm
- Task scheduling
- Median finding (two heaps)
- K largest/smallest elements

**Python's heapq (Min-heap by default):**
```python
import heapq

# For max-heap, negate values
heap = []
heapq.heappush(heap, -5)  # Push negated
val = -heapq.heappop(heap)  # Pop and negate back
```

---

## Question 8: Concurrency: Giải thích các synchronization primitives và deadlock conditions?

**Answer:**

**Synchronization Primitives:**

**1. Mutex (Mutual Exclusion)**
```python
import threading

mutex = threading.Lock()

def critical_section():
    with mutex:
        # Only one thread can execute this
        shared_resource.update()
```

**2. Semaphore**
```python
# Allows N threads to access resource simultaneously
semaphore = threading.Semaphore(3)  # Max 3 concurrent

def limited_access():
    with semaphore:
        # Up to 3 threads can be here
        access_resource()
```

**3. Condition Variable**
```python
condition = threading.Condition()
queue = []

def producer():
    with condition:
        queue.append(item)
        condition.notify()  # Wake up one waiting consumer

def consumer():
    with condition:
        while not queue:
            condition.wait()  # Release lock and wait
        return queue.pop(0)
```

**4. Read-Write Lock**
```python
# Multiple readers OR one writer
# Python: threading.RLock() for reentrant lock
```

**Deadlock Conditions (All 4 must be present):**

1. **Mutual Exclusion**: Resource held exclusively
2. **Hold and Wait**: Holding one resource, waiting for another
3. **No Preemption**: Cannot forcibly take resource
4. **Circular Wait**: A waits for B, B waits for A

**Deadlock Prevention:**
```python
# Strategy: Always acquire locks in same order
lock_a = threading.Lock()
lock_b = threading.Lock()

def process():
    with lock_a:      # Always acquire A before B
        with lock_b:
            do_work()
```

**Deadlock Example:**
```python
# Thread 1              # Thread 2
lock_a.acquire()        lock_b.acquire()
lock_b.acquire()   →    lock_a.acquire()  # DEADLOCK!
```

**Other Concurrency Issues:**
- **Race Condition**: Output depends on timing
- **Livelock**: Threads change state but make no progress
- **Starvation**: Thread never gets resource access

---

## Question 9: Implement LRU Cache với O(1) operations?

**Answer:**

**LRU (Least Recently Used) Cache** evicts least recently accessed item khi cache full.

**Approach:** Hash Map + Doubly Linked List

```python
class Node:
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}  # key -> Node

        # Dummy head and tail
        self.head = Node()
        self.tail = Node()
        self.head.next = self.tail
        self.tail.prev = self.head

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1

        node = self.cache[key]
        self._move_to_front(node)
        return node.value

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            node = self.cache[key]
            node.value = value
            self._move_to_front(node)
        else:
            if len(self.cache) >= self.capacity:
                self._evict_lru()

            node = Node(key, value)
            self.cache[key] = node
            self._add_to_front(node)

    def _add_to_front(self, node):
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def _remove_node(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _move_to_front(self, node):
        self._remove_node(node)
        self._add_to_front(node)

    def _evict_lru(self):
        lru = self.tail.prev
        self._remove_node(lru)
        del self.cache[lru.key]
```

**Complexity:**
- Get: O(1)
- Put: O(1)
- Space: O(capacity)

**Data Structure Visualization:**
```
HEAD ↔ [Most Recent] ↔ [Recent] ↔ ... ↔ [LRU] ↔ TAIL
         ↑
    Hash Map points to nodes
```

**Python Alternative with OrderedDict:**
```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)
```

---

## Question 10: Giải thích Trie data structure và use cases?

**Answer:**

**Trie (Prefix Tree)** là tree structure để store strings, mỗi edge represents một character.

**Implementation:**
```python
class TrieNode:
    def __init__(self):
        self.children = {}  # char -> TrieNode
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_end = True

    def search(self, word: str) -> bool:
        node = self._find_node(word)
        return node is not None and node.is_end

    def starts_with(self, prefix: str) -> bool:
        return self._find_node(prefix) is not None

    def _find_node(self, prefix: str) -> TrieNode:
        node = self.root
        for char in prefix:
            if char not in node.children:
                return None
            node = node.children[char]
        return node

    # Autocomplete: Get all words with prefix
    def autocomplete(self, prefix: str) -> list:
        node = self._find_node(prefix)
        if not node:
            return []

        results = []
        self._dfs(node, prefix, results)
        return results

    def _dfs(self, node, path, results):
        if node.is_end:
            results.append(path)

        for char, child in node.children.items():
            self._dfs(child, path + char, results)
```

**Complexity:**
- Insert: O(m) where m = word length
- Search: O(m)
- Prefix search: O(m)
- Space: O(n × m) worst case

**Use Cases:**

1. **Autocomplete/Type-ahead**
   ```
   User types "pro" → suggest ["program", "promise", "project"]
   ```

2. **Spell Checker**
   - Check if word exists in dictionary
   - Suggest corrections

3. **IP Routing (Longest Prefix Match)**
   - Router tables use tries for IP lookup

4. **Word Games**
   - Boggle, Scrabble word validation

5. **T9 Predictive Text**

**Optimizations:**

**Compressed Trie (Radix Tree):**
```
Instead of: a → p → p → l → e
Compress:   "apple" (single edge with string)
```

**Memory Efficient:**
```python
# Use array instead of dict for lowercase letters
self.children = [None] * 26
index = ord(char) - ord('a')
```

---
