# 12-Week DSA Study Roadmap for Python Developers

Since you already know Python syntax and have **4 years of experience**, but you're **new to DSA/interview preparation**, don't study like a computer science student. Study like an engineer learning patterns.

## Phase 1 (Week 1–2): Python Fundamentals for DSA

Learn these topics until you can write them without looking up documentation.

### Arrays & Lists

```python
nums = [1, 2, 3]

nums.append(4)
nums.pop()
nums.insert(1, 10)
nums.remove(2)
nums.sort()
nums.reverse()

len(nums)
max(nums)
min(nums)
sum(nums)
```

Practice:

- Reverse array
- Find max/min
- Rotate array
- Remove duplicates

---

### Strings

Learn:

```python
s = "python"

s.lower()
s.upper()
s.strip()
s.split()
" ".join(list)

s.startswith()
s.endswith()

s.replace()

s[::-1]
```

Practice:

- Reverse string
- Count vowels
- Palindrome
- Anagram
- Frequency count

---

### Dictionaries

```python
freq = {}

for x in nums:
    freq[x] = freq.get(x, 0) + 1
```

Practice:

- Two Sum
- Contains Duplicate
- Character Count

---

### Sets

```python
visited = set()

visited.add(10)

10 in visited

visited.remove(10)
```

Practice:

- Unique elements
- Duplicate detection

---

### Collections Module

```python
from collections import Counter
from collections import defaultdict
from collections import deque
```

Must know.

---

## Phase 2 (Week 3)

### Two Pointers

Problems

- Two Sum II
- Move Zeroes
- Valid Palindrome
- Remove Duplicates
- Container With Most Water
- 3Sum

Learn:

```python
left = 0
right = len(nums)-1

while left < right:
```

---

## Phase 3 (Week 4)

### Sliding Window

Fixed window

```python
window_sum = sum(nums[:k])

for right in range(k, len(nums)):
```

Variable window

```python
left = 0

for right in range(len(nums)):
```

Problems

- Maximum Average Subarray
- Maximum Sum Subarray
- Longest Substring Without Repeating Characters
- Character Replacement
- Minimum Window Substring

---

## Phase 4 (Week 5)

### Binary Search

Learn

```python
left = 0
right = len(nums)-1

while left <= right:
```

Problems

- Binary Search
- Search Insert Position
- First Bad Version
- Search Rotated Array

---

## Phase 5 (Week 6)

### Stack

Python

```python
stack = []

stack.append()

stack.pop()
```

Problems

- Valid Parentheses
- Min Stack
- Daily Temperatures
- Next Greater Element

---

## Phase 6 (Week 7)

### Linked List

Learn

- Reverse
- Merge
- Detect Cycle
- Middle Node

---

## Phase 7 (Week 8)

### Trees

Must know

```text
DFS
  - Preorder
  - Inorder
  - Postorder
BFS
```

Problems

- Maximum Depth
- Same Tree
- Invert Tree
- Level Order
- Lowest Common Ancestor

---

## Phase 8 (Week 9)

### Heap

Python

```python
import heapq

heap = []

heapq.heappush(heap, 5)

heapq.heappop(heap)
```

Problems

- Top K Frequent
- K Closest Points
- Merge K Lists

---

## Phase 9 (Week 10)

### Graph

Algorithms

- DFS
- BFS
- Topological Sort
- Union Find

Problems

- Number of Islands
- Clone Graph
- Course Schedule

---

## Phase 10 (Week 11)

### Backtracking

Learn recursion first.

Problems

- Subsets
- Permutations
- Combination Sum
- Word Search

---

## Phase 11 (Week 12)

### Dynamic Programming

Start with

- Climbing Stairs
- House Robber
- Coin Change
- Longest Increasing Subsequence
- Longest Common Subsequence

---

## Daily Schedule (2 Hours)

| Time   | Activity                         |
| ------ | -------------------------------- |
| 15 min | Review yesterday's solution      |
| 30 min | Learn one algorithm/pattern      |
| 45 min | Solve 2–3 easy problems          |
| 20 min | Watch or read one editorial      |
| 10 min | Rewrite the solution from memory |

---

## Python Features You Must Master

### Built-in Functions

```python
len()
sum()
max()
min()
sorted()
enumerate()
zip()
any()
all()
```

---

### Collections

```python
Counter
defaultdict
deque
```

---

### Heap

```python
heapq
```

---

### Sorting

```python
nums.sort()
sorted(nums)
sorted(nums, key=lambda x: x[1])
```

---

### String Operations

```python
split()
join()
replace()
strip()
find()
count()
```

---

### List Comprehension

```python
squares = [x * x for x in range(10)]
```

---

### Dictionary Comprehension

```python
{x: x * x for x in range(5)}
```

---

### Lambda

```python
sorted(arr, key=lambda x: x[1])
```

---

### Enumerate

```python
for index, value in enumerate(nums):
```

---

### Zip

```python
for a, b in zip(arr1, arr2):
```

---

### Deque

```python
from collections import deque

queue = deque()

queue.append()

queue.popleft()
```

---

## Milestones

- **Weeks 1–2:** 40–50 easy problems (arrays, strings, hashing).
- **Weeks 3–6:** 40–50 medium problems (sliding window, stack, binary search, linked lists).
- **Weeks 7–10:** 30–40 medium problems (trees, heaps, graphs).
- **Weeks 11–12:** 20–30 medium DP/backtracking problems.

By the end, you'll have solved **130–170 problems** while covering the major interview patterns.

## Recommended Progression

1. Python basics for DSA
2. Arrays
3. Strings
4. HashMap & Set
5. Two Pointers
6. Sliding Window
7. Stack
8. Binary Search
9. Linked List
10. Trees (DFS/BFS)
11. Heap
12. Graphs
13. Backtracking
14. Dynamic Programming

This order minimizes context switching and builds each new algorithm on top of concepts you've already practiced.
