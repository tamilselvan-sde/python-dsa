# FAANG Interview Patterns — Pattern Recognition Playbook

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
> Section: 09 — FAANG Patterns
> 🔗 Related: [big_o.md](../08_Time_Complexity/big_o.md) · [prefix_sum.md](../07_Algorithms/prefix_sum.md) · [two_pointers.md](../07_Algorithms/two_pointers.md) · [sliding_window.md](../07_Algorithms/sliding_window.md) · [sorting.md](../07_Algorithms/sorting.md) · [binary_search.md](../07_Algorithms/binary_search.md)
> Back to [README](../README.md)

---

## 1. What is it?

A **FAANG interview pattern** is a reusable **problem-solving archetype** that maps a recognizable problem shape to a known algorithmic technique. After enough practice you start to **read a problem statement, see the keywords, and immediately know which family of solutions to reach for**.

This file is a reference dictionary of the **20 core patterns** that cover ≥ 90% of LeetCode-style interview questions asked at FAANG-tier companies (Meta, Google, Amazon, Microsoft, Apple, Netflix, Uber, Stripe, etc.).

Think of it like a chess opening book — pattern recognition **doesn't solve the problem for you**, but it tells you which 2–3 candidate approaches to try first so you don't waste the first 10 minutes reinventing the wheel.

Each pattern entry below contains:

- **Recognize (signal keywords)** — phrases in the problem statement that *scream* this pattern
- **Key idea** — one-sentence intuition
- **Python skeleton** — a 5–10 line idiomatic template
- **Representative problems** — 2–3 famous LeetCode questions with numbers
- **Common mistakes** — what trips beginners

After the patterns you'll find:

- The **interview process** itself (clarify → examples → brute → optimal → code → test → complexity)
- A **behavioral STAR** template
- The **Top 30** most-asked LeetCode problems (one-line hint each)
- A **pattern frequency table** at FAANG
- The **"First 60 minutes: which pattern?"** decision flowchart
- The **"Stuck on a problem?"** recovery decision tree
- Common **Python interview gotchas**

---

## 2. Why do we use it?

- **Time pressure.** A 45-minute interview penalizes scratch work. Recognizing the pattern in under 2 minutes can be the difference between hire / no-hire.
- **Coverage.** Learn 20 patterns → you can attack thousands of LeetCode problems without memorizing each one.
- **Communication.** Saying *"This is the classic monotonic stack pattern"* to your interviewer signals you've trained — they'll trust your approach.
- **Follow-up agility.** Pattern fluency helps you pivot when the interviewer changes constraints ("what if duplicates are allowed?").
- **Self-diagnosis.** When you fail a problem, you can categorize the miss: *"I didn't recognize the bitmask DP signal"* — and target that pattern next.

---

## 3. When should I choose it? — Pattern Decision Table

| Signal in the problem                                            | Best-fit pattern                                   |
|------------------------------------------------------------------|----------------------------------------------------|
| "Sum of subarray / range query / count subarrays with…"          | **Prefix Sum**                                    |
| "Sorted array / palindromic / pair sum / triplet"               | **Two Pointers**                                  |
| "Longest/shortest substring / subarray with constraint"        | **Sliding Window**                                |
| "Cycle in a sequence / middle of linked list / find k-th"       | **Fast & Slow Pointers**                          |
| "Overlapping intervals / merge / meeting rooms"                | **Merge Intervals**                               |
| "Reverse / reorder / partition linked list"                    | **Linked List Reversal / Mutation**               |
| "Count, group, two-sum-like, anagrams, freq."                  | **Hash Map / Frequency**                          |
| "K-th largest/smallest / top K / K closest"                    | **Top K (Heap)**                                  |
| "Sorted array / search insert / rotated / peak"                | **Binary Search**                                 |
| "All subsets / all permutations / generate all valid…"         | **Backtracking (Subsets/Permutations)**          |
| "Average / right-side view / level-by-level of tree"           | **Tree BFS (level order)**                        |
| "Path sum / depth / ancestor / serialize tree"                | **Tree DFS**                                      |
| "Word ladder / shortest path / maze / min steps in grid"      | **Graph BFS**                                     |
| "Connectivity / cycle / topo order / course schedule"          | **Graph DFS / Topological Sort**                  |
| "Pick-or-skip items / max value with capacity / k items"      | **0/1 Knapsack**                                  |
| "Fib / climb stairs / decode ways / count paths"              | **Fibonacci / 1D DP**                             |
| "Edit distance / LCS / palindrome subsequence"                | **2D String DP**                                  |
| "Prefix match / dictionary / autocomplete / word search"      | **Trie**                                          |
| "Connected components / redundant edge / accounts merge"     | **Union-Find**                                    |

---

## 4. Syntax (Pattern Recognition Grammar)

Each pattern's skeleton in this file uses this template:

```python
# PATTERN: <name>
# Recognize:
#   - <signal keywords>
# Key idea: <one-liner>
def solve(input):
    # data layout
    state = <init>
    for/in/recurrence:
        update state
    return answer
```

Use this format yourself in mock interviews — write the pattern name and key idea as a **comment at the top** of your solution. Interviewers love it.

---

## 5. Basic Example

A toy problem attacked via three different patterns so you can see recognition in action.

**Problem (LeetCode 1, Two Sum):** *"Given nums and target, return indices of two numbers that sum to target."*

**Reading for signals:**
- "two numbers" + "sum" with unordered array → **Hash Map / Frequency** is the optimal pattern.
- If sorted → **Two Pointers** would also work.
- Brute → nested loops (O(n²), not a pattern but a baseline).

**Pattern: Hash Map / Complement Lookup**
```python
def twoSum(nums, target):
    seen = {}                          # val -> index
    for i, v in enumerate(nums):
        if (target - v) in seen:
            return [seen[target - v], i]
        seen[v] = i
    return []
```
Time O(n), Space O(n).

**Pattern: Two Pointers** (only valid if `nums` is sorted or we sort first)
```python
def twoSumSorted(nums, target):
    l, r = 0, len(nums) - 1
    while l < r:
        s = nums[l] + nums[r]
        if s == target:  return [l, r]
        if s < target:    l += 1
        else:              r -= 1
```
Time O(n) extra, plus O(n log n) if a sort is needed.

---

## 6. Step-by-Step Dry Run

A canonical "first 90 seconds in an interview" walkthrough:

```
Read problem ──► underline keywords ──► brainstorm keywords → patterns (max 3 candidates)
       │
       ▼
Pick the most promising pattern (lowest Big-O feasible per constraints)
       │
       ▼
State: "I'll use <pattern>. Time O(__), Space O(__). Here is the approach…"
       │
       ▼
Walk through a small example aloud, drawing state changes on the whiteboard
       │
       ▼
Write skeleton: pattern name as comment + variable names matching the example
       │
       ▼
Fill in the body, narrating each branch
       │
       ▼
Run through the example manually; then run a second edge case (empty, single, dup)
       │
       ▼
Confirm Big-O; mention follow-ups (space reduction, extension)
```

---

## 7. Built-in Methods — Python idioms per pattern

Each pattern in Python has a **natural idiom**. Master these; they save lines and bugs.

### 7.1 Prefix Sum
```python
pref = [0]
for x in arr:
    pref.append(pref[-1] + x)         # O(n) build
range_sum = pref[r+1] - pref[l]       # O(1)
```
Idioms: `pref[-1]`, `collections.Counter` for variations, `defaultdict(int)`.

### 7.2 Two Pointers
```python
l, r = 0, len(arr) - 1
while l < r:
    if <condition>:   l += 1
    elif <condition>: r -= 1
    else:             return <answer>
```
Idioms: sorted input, opposite-direction pointers, fast/slow pointers (also for linked lists).

### 7.3 Sliding Window
```python
from collections import Counter, defaultdict
left = 0
freq = defaultdict(int)
for right, x in enumerate(arr):
    freq[x] += 1
    while <window invalid>:
        freq[arr[left]] -= 1
        if freq[arr[left]] == 0: del freq[arr[left]]
        left += 1
    # window arr[left..right] is now valid
```
Idioms: `enumerate`, `defaultdict`, `deque` for fixed-window rotation.

### 7.4 Fast & Slow Pointers
```python
slow = fast = head
while fast and fast.next:
    slow = slow.next
    fast = fast.next.next
# slow is at midpoint or cycle entry (with extra logic)
```

### 7.5 Merge Intervals
```python
intervals.sort(key=lambda x: x[0])
merged = []
for s, e in intervals:
    if not merged or merged[-1][1] < s:
        merged.append([s, e])
    else:
        merged[-1][1] = max(merged[-1][1], e)
```

### 7.6 Hash Map / Frequency / Counter
```python
from collections import Counter, defaultdict
cnt = Counter(arr)                # O(n)
cnt.most_common(k)                # O(n log k)
seen = {}                          # generic complement dict
for x in arr:
    if target - x in seen:  ...
    seen[x] = i
```

### 7.7 Top K — Heap
```python
import heapq
heapq.nlargest(k, arr)                       # O(n log k)
heapq.nsmallest(k, arr)
# Manual:
h = arr[:k]; heapify(h)                       # O(k)
for x in arr[k:]:
    if x > h[0]:
        heapq.heapreplace(h, x)               # O(log k)
```

### 7.8 Binary Search
```python
lo, hi = 0, len(arr) - 1
while lo <= hi:
    mid = (lo + hi) // 2
    if arr[mid] == target: return mid
    elif arr[mid] < target: lo = mid + 1
    else:                     hi = mid - 1
```
Variants: lower bound (`lo < hi`, `hi = mid`), search on answer (max-min), `bisect.bisect_left`.

### 7.9 Backtracking
```python
def bt(state, choices, path, out):
    if <base case>: out.append(path[:]); return
    for c in choices:
        <apply choice>
        bt(state, choices, path + [c], out)
        <un-apply choice>   # or rely on path[:] copy
```

### 7.10 Tree BFS (level order)
```python
from collections import deque
q = deque([root])
while q:
    for _ in range(len(q)):
        node = q.popleft()
        # visit
        if node.left:  q.append(node.left)
        if node.right: q.append(node.right)
```

### 7.11 Tree DFS
```python
def dfs(node):
    if not node: return <base>
    L = dfs(node.left)
    R = dfs(node.right)
    return <combine node.val, L, R>
```
Pre/in/post-order by reordering `node.val`, `L`, `R`.

### 7.12 Graph BFS
```python
from collections import deque
q = deque([start]); seen = {start}
while q:
    u = q.popleft()
    for v in adj[u]:
        if v not in seen:
            seen.add(v)
            q.append(v)
```

### 7.13 Graph DFS / Topo Sort
```python
def dfs(u):
    if u in visited: return
    visited.add(u)
    for v in adj[u]:
        dfs(v)
    order.append(u)                  # for topo sort: reverse later

# Iterative post-order for topo:
order = []; visited = set(); stack = [(start, False)]
while stack:
    u, processed = stack.pop()
    if processed: order.append(u); continue
    if u in visited: continue
    visited.add(u); stack.append((u, True))
    for v in adj[u]: stack.append((v, False))
```

### 7.14 0/1 Knapsack DP
```python
dp = [0] * (capacity + 1)                # space rolled
for val, wt in items:
    for c in range(capacity, wt - 1, -1):   # reverse to avoid reuse
        dp[c] = max(dp[c], dp[c - wt] + val)
```

### 7.15 1D / Fibonacci DP
```python
dp = [0] * (n + 2)
dp[0], dp[1] = 0, 1
for i in range(2, n + 1):
    dp[i] = dp[i-1] + dp[i-2]
# Often: rolling vars to save space to O(1)
```

### 7.16 2D String DP
```python
dp = [[0] * (m + 1) for _ in range(n + 1)]
for i in range(1, n + 1):
    for j in range(1, m + 1):
        if s1[i-1] == s2[j-1]:
            dp[i][j] = dp[i-1][j-1] + 1
        else:
            dp[i][j] = max(dp[i-1][j], dp[i][j-1])
```

### 7.17 Trie
```python
class Trie:
    def __init__(self):  self.children = {}; self.end = False
    def insert(self, w):
        node = self
        for c in w:
            node = node.children.setdefault(c, Trie())
        node.end = True
    def search(self, w):
        node = self
        for c in w:
            if c not in node.children: return False
            node = node.children[c]
        return node.end
```

### 7.18 Union-Find
```python
parent = list(range(n))
rank = [0] * n
def find(x):
    while parent[x] != x:
        parent[x] = parent[parent[x]]      # path compression
        x = parent[x]
    return x
def union(a, b):
    ra, rb = find(a), find(b)
    if ra == rb: return False
    if rank[ra] < rank[rb]: ra, rb = rb, ra
    parent[rb] = ra
    if rank[ra] == rank[rb]: rank[ra] += 1
    return True
```

---

## 8. Interview Example — Pattern in Action

**Problem (LeetCode 49 — Group Anagrams):** Group the anagrams in a list of strings.

**Signals:**
- "anagram" → **Hash Map / Frequency**.
- We need to bucket strings by their letter signature.

**Pattern: Hash Map with canonical key**
```python
def groupAnagrams(strs):
    buckets = defaultdict(list)
    for s in strs:
        key = tuple(sorted(s))                # O(k log k) per string
        buckets[key].append(s)
    return list(buckets.values())
```
Time: O(N · K log K) where N = #strings, K = max length.  
Space: O(N·K).

**Optimization with counter key:**
```python
def groupAnagrams(strs):
    buckets = defaultdict(list)
    for s in strs:
        cnt = [0] * 26
        for c in s: cnt[ord(c) - 97] += 1
        buckets[tuple(cnt)].append(s)
    return list(buckets.values())
```
Time: O(N·K), Space: O(N·K). Pattern recognizes → cleaner and faster.

---

## 9. When NOT to use

- **Don't pattern-match before understanding the example.** Spend 60 seconds tracing the sample input. A wrong pattern wastes 10 minutes.
- **Don't insist on an optimal pattern if a brute will fit constraints.** If n ≤ 20 and 2ⁿ works, go for it. Pattern overfitting is real.
- **Don't carry the same pattern into follow-ups.** When the interviewer changes constraints, re-read signals — the new pattern may be different.
- **Don't avoid patterns just because they're "fancy".** Trie, Union-Find, Monotonic Stack are short and expected at FAANG.
- **Don't over-template.** A custom skeleton reused blindly will fail edge cases the pattern doesn't cover (empty input, duplicates, negatives).

---

## 10. Common Mistakes

1. **Misreading the pattern.** "Subarray sum" looks like sliding window, but "any subarray (not contiguous)" is actually subset / knapsack.
2. **Sliding window with a `for left` instead of `while`.** Right pointer moves 1× per loop; left must `while` to shrink until valid.
3. **Two pointers on unsorted array.** Without sorting, two pointers solve only a few problems (palindrome, in-place removal).
4. **Backtracking without state restoration.** When you append to a shared `path`, you MUST `pop()` it after recursion — or pass copies (slower but safer).
5. **Heap of size n instead of k.** "`nlargest` is O(n log k)" not O(n log n) — keep heap size k.
6. **Union-Find without path compression.** Naive Union-Find is O(n) per op → entire problem O(n²). Always compress.
7. **DP with wrong indexing.** Mismatched off-by-one between "DP table" and the original string is the #1 bug in 2D string DP.
8. **BFS not setting `visited` at enqueue time.** If you wait till dequeue, duplicates pile up → O(V + E) becomes O(V²).
9. **Trie without lowercase sanity.** Indexing `ord(c) - 97` on uppercase or non-letters throws index errors. Normalize input first.
10. **Forgetting monotonic-stack "pop while top < current".** Wrong comparator direction reverses the result.
11. **Recursive DFS blowing Python's stack** at depth > 1000 → rewrite iterative.
12. **Knapsack rolling array left-to-right** → reuses the same item. Iterate capacity **descending**.

---

## 11. Memory Tricks

- **"Sum → prefix. Pair/triplet sorted → two pointers. Window → sliding. K → heap. Path → BFS/DFS."**
- **"DP is recursion + memory. Greedy is La-Z-Boy DP (only the current good state)."**
- **"Tree BFS = level queue. Graph BFS = shortest path on unweighted."**
- **"LCSubstr vs LCSubseq:** `str` ends with same char → substr; subseq can skip.
- **"Backtracking = DFS on a choices tree."** Same code, different domain.
- **"Heap = priority queue. Top K → keep a small heap."**
- **"Trie = nested dicts."** If you can write `defaultdict(Trie)`, you can write a trie.
- **"Two-sum-family → complement dict."** Two-sum, three-sum, four-sum all reduce.

---

## 12. Interview Shortcuts

- Always read constraints first — they whisper the complexity class, which whispers the pattern.
- Constraints ≤ ~20 ⇒ bitmask / backtracking. ≤ 100 ⇒ O(n³). ≤ 10⁵ ⇒ O(n log n) or O(n).
- Sorted == binary search. Sorted + pair == two pointers.
- K-th == heap. K smallest == max-heap of size K.
- "Number of ways / min cost / max profit" with overlapping subproblems ⇒ DP.
- "Cycle / middle / find k-th of linked list" ⇒ fast & slow pointers.
- "All subsets/permutations/combinations" ⇒ backtracking template.
- "Path between two nodes with shortest distance" ⇒ BFS (unweighted) or Dijkstra (weighted).
- Time/space before code. Speak at the whiteboard.
- After the optimal, mention a follow-up ("if duplicates / negative numbers / streaming") — interviewers love initiative.

---

## 13. Cheat Sheet Table — The 20 Patterns

| # | Pattern                         | Recognize signal                                  | Skeleton complexity    |
|---|---------------------------------|---------------------------------------------------|------------------------|
| 1 | Prefix Sum                      | range-sum / range-count, count subarrays           | O(n) build / O(1) query  |
| 2 | Two Pointers                    | sorted pair / triplet / palindrome                 | O(n)                    |
| 3 | Sliding Window                  | longest/shortest constraint subarray               | O(n)                    |
| 4 | Fast & Slow Pointers            | cycle / midpoint / k-th-from-end (LL)             | O(n)                    |
| 5 | Merge Intervals                 | overlapping intervals / meeting rooms              | O(n log n)              |
| 6 | Reverse Linked List             | reverse k / reverse between / rotate LL            | O(n)                   |
| 7 | In-place LL Mutation            | partition / reorder / dedupe in place                | O(n)                   |
| 8 | Hash Map / Frequency            | complement / count / two-sum family / anagrams     | O(n)                    |
| 9 | Top K / Heap                    | k-th largest/smallest, top K, K closest             | O(n log k)              |
| 10| Binary Search                   | sorted/search/rotated/peak/search on answer       | O(log n)                |
| 11| Subset/Permutation Backtracking | "all subsets / permutations / combinations"        | O(2ⁿ) / O(n!)           |
| 12| Tree BFS (level order)          | "level by level" / "right side view"               | O(n)                    |
| 13| Tree DFS / Subtree              | "path sum" / "depth" / serialize subtree             | O(n)                    |
| 14| Graph BFS                       | shortest unweighted path / maze                     | O(V + E)                |
| 15| Graph DFS / Topo Sort           | connectivity / course schedule / island grid        | O(V + E)                |
| 16| 0/1 Knapsack DP                 | pick-or-skip items with weight/value / capacity    | O(n · W)                |
| 17| Fibonacci / 1D DP               | count ways / max subset / climb / decode ways      | O(n)                    |
| 18| 2D String DP                    | edit distance / LCS / palindrome subseq             | O(n · m)                |
| 19| Trie / String Search            | prefix match / autocomplete / dictionary            | O(L) per op             |
| 20| Union-Find                      | redundant edge / connected components / merge       | O(α(n)) per op         |

---

## 14. Time Complexity Table — Per Pattern

| Pattern                          | Time                  | Space                  |
|----------------------------------|-----------------------|------------------------|
| Prefix Sum                       | O(n) build, O(1) q    | O(n)                   |
| Two Pointers                     | O(n)                  | O(1)                   |
| Sliding Window                   | O(n)                  | O(k) (window state)    |
| Fast & Slow Pointers             | O(n)                  | O(1)                   |
| Merge Intervals                  | O(n log n)            | O(n) (output)          |
| Reverse / In-place LL            | O(n)                  | O(1)                   |
| Hash Map / Frequency             | O(n)                  | O(n)                   |
| Top K (heap)                     | O(n log k)            | O(k)                   |
| Binary Search                    | O(log n)              | O(1) / O(log n) stack  |
| Subset Backtracking              | O(n · 2ⁿ)             | O(n) stack              |
| Permutation Backtracking         | O(n · n!)             | O(n) stack              |
| Tree BFS                         | O(n)                  | O(width) queue          |
| Tree DFS                         | O(n)                  | O(h) stack              |
| Graph BFS                        | O(V + E)              | O(V)                    |
| Graph DFS / Toposort             | O(V + E)              | O(V)                    |
| 0/1 Knapsack DP                  | O(n · W)              | O(W) (rolled)           |
| Fibonacci / 1D DP                | O(n)                  | O(n) or O(1)            |
| 2D String DP                     | O(n · m)              | O(n · m) or O(min)      |
| Trie                             | O(L)                  | O(Σ · L)                |
| Union-Find (path compression + rank) | O(α(n))           | O(n)                    |

---

## 15. Visual Diagram (ASCII)

### 15.1 "First 60 minutes: which pattern?" decision flowchart

```
                      ┌─────────────────────────────────────────┐
                      │  Read constraints: max n? type of data? │
                      └────────────────────┬────────────────────┘
                                           ▼
                  Is the input already SORTED or sortable cheaply?
                              │                       │
                          YES │                       │ NO
                              ▼                       ▼
                         Pair / range           ┌─────────────────────┐
                         / position             │ Is the data a tree  │
                         question?              │ / graph / LL?        │
                              │ YES             └──────┬──────────────┘
                              ▼                          │
                        Two Pointers                   YES NO
                        OR Binary Search                ├─────────────────────┐
                        (search-shaped)                ▼                     ▼
                                                  LL: cycle ← Fast/Slow   Is it DP-shaped?
                                                  Graph path ← BFS/DFS    │
                                                  Tree level ← BFS         YES NO
                                                  Tree depth ← DFS          │   │
                                                                            ▼   ▼
                                                                     2D string DP,    Need buckets /
                                                                     0/1 knapsack,    counts /
                                                                     Fibonacci-style  matching?
                                                                                            │
                                                                                       YES  ▼
                                                                                   Hash Map / Frequency
                                                                                   OR Trie (prefix shaped)
                                                                                   OR Heap (top K phrased)
                                                                                   OR Backtracking
                                                                                      (generator style)

                ◄─── if nothing fits yet ──►  default: brute force, then optimize
```

### 15.2 "Stuck on a problem?" recovery decision tree

```
                ┌──────── You're stuck. ────┐
                                            ▼
                  Yelp brute force (≤ O(n²)? ≤ O(n³)?) — get SOMETHING working
                                            │
                                            ▼
                  Read signal keywords again, underline anything you missed
                                            │
                ┌───────────────────────────┴────────────────────────────┐
                ▼                                                          ▼
            Subarray / "sum/constraint over a window"           "Two numbers" / pair / triplet
                      │                                                 │
                      ▼                                                 ▼
            Sliding Window or Prefix Sum                          Two Pointers or Hash Map
                      │
                      ▼
            "Longest/shortest" type → Sliding Window
            "Count / subarray sum = k" → Prefix Sum + counter

                ┌───────────────────────────┴────────────────────────────┐
                ▼                                                          ▼
            "All subsets / permutations / combinations"                "K-th", "top/bottom K"
                      │                                                 │
                      ▼                                                 ▼
                  Backtracking                                       Heap / Quickselect

                ┌───────────────────────────┴────────────────────────────┐
                ▼                                                          ▼
            "Tree or graph path / depth / level"               ufe "Pick-or-skip / count ways / max profit"
                      │                                                 │
                      ▼                                                 ▼
                   BFS or DFS (pick by edge weight)              DP (figure state, transition, base)

                ┌───────────────────────────┴────────────────────────────┐
                ▼                                                          ▼
            "Connected / merge / redundant"                    "Prefix match / dictionary"
                      │                                                 │
                      ▼                                                 ▼
                   Union-Find                                          Trie

                ◄─── last resort: try the LeetCode Discuss editorial hints one paragraph deep ──►
```

### 15.3 Pattern → Keyword map (mental cheat sheet)

```
sum over range      ─► prefix sum
two/three numbers   ─► two pointers / hash complement
longest substring   ─► sliding window
kth / top K          ─► heap
sorted & search     ─► binary search
cycle in LL         ─► fast & slow
overlapping ranges  ─► merge intervals
all combinations    ─► backtracking
min steps in grid   ─► BFS
course schedule     ─► topo / dfs
shortest weighted   ─► Dijkstra
connect components  ─► union find
climb / decode / ways ─► 1D DP
edit distance / LCS ─► 2D DP
autocomplete / prefix ─► trie
```

---

## 16. Beginner Notes

> Remember:
>
> 1. **Pattern recognition >> memorization.** Understand *why* a pattern fits each signal.
> 2. **Always 1 brute + 1 optimal.** Even if you see the optimal immediately, speak the brute out loud first.
> 3. **State time and space before writing code.** The interviewer wants to know you sized it.
> 4. **Off-by-one is the #1 bug.** Prefix sum, sliding window, binary search, DP table — every pattern has an index bug.
> 5. **Constrain yourself to speaking up.** A silent candidate looks stuck even when they aren't. Narrate: *"I'm thinking it might be sliding window because..."*
> 6. **Hand-trace examples.** Use the sample input *before* typing; design your variable names from the example.
> 7. **Edge cases always:** empty / single / duplicates / negatives / sorted / unsorted / overflow. Write tests after code.
> 8. **Don't summarize algorithms.** Briefly articulate approach, then code. Wasted time in monologues costs you coding minutes.
> 9. **One pattern rarely covers all the follow-ups.** Be ready to pivot to a heap, prefix sum, or DP table.
> 10. **Clean code trumps clever code.** In a 45-min interview, clarity wins. Save one-liners for the bar.

---

## 17. FAANG Tips

- **Meta likes graph traversal, sliding window, and intervals.** They move fast; narrate before code.
- **Google loves DP, math, and "search on answer".** They'll push you past the obvious pattern with a constraint twist.
- **Amazon loves Leadership Principles.** Code-as-a-skill question is followed by behavioral. Practice STAR narratives (template below).
- **Apple emphasizes fundamentals.** Big-O matters; they'll ask explicitly. Don't fudge space.
- **Netflix values performances.** Expect an errand about scaling: "What if the input is 1 TB?" Mention streaming, bloom filters, sampling.
- **Microsoft often asks tree/LL/string questions** with two-pointer or DP follow-ups.
- **Uber / Stripe focus on real systems** — edge cases are king. Mention idempotency, currency rounding, time-zones.
- **Always ask clarifying questions:**
  - Duplicates possible? Negatives? Empty array? Sorted?
  - Output ordered or any order is OK?
  - 1-indexed or 0-indexed?
  - Modify in place or new data structure?
- **End each round with a question.** "What does your typical week look like?" beats silence.

### 17.1 Behavioral STAR Anecdote Template

```
Situation:           ─ context, project, role           (~20 sec)
Task:                 ─ what *you* were responsible for (~15 sec)
Action:               ─ concrete steps YOU took         (~60 sec, the meat)
Result:               ─ measurable outcome + metric     (~15 sec)
+ Reflection (optional): what you'd do differently
```

Prepare 5–7 stories mapping to: **Customer Obsession, Ownership, Deliver Results, Invent & Simplify, Dive Deep, Bias for Action, Disagree & Commit, Earn Trust.**

Example flood-proof outline:

> **S:** Last quarter I owned the orders ingestion pipeline for our checkout service.  
> **T:** I had to cut end-to-end event latency below 200 ms. The legacy pipeline averaged 800 ms.  
> **A:** I profiled the hot path, found a redundant O(n²) dedupe loop, switched to a hash-set-based filter, and parallelized the consumer using a thread pool of size = CPU count.  
> **R:** P99 dropped 73% to 140 ms. Throughput went from 500 → 1800 ops/sec. Promoted to next IC level at the next cycle.

---

## 18. Practice Problems

### The Top 30 most-asked LeetCode problems (ranked by interview frequency)

| #  | Problem                                | LC    | Pattern                | One-line hint                              |
|----|----------------------------------------|-------|------------------------|--------------------------------------------|
| 1  | Two Sum                                | 1     | Hash Map               | complement dict while iterating            |
| 2  | Reverse Linked List                   | 206   | LL Reversal            | iterative prev/cur/next                    |
| 3  | Valid Parentheses                      | 20    | Stack                  | open-close pair stack                      |
| 4  | Merge Two Sorted Lists                 | 21    | Two Pointers (LL)      | dummy head, compare & connect              |
| 5  | Best Time to Buy/Sell Stock            | 121   | DP / Prefix min        | track min-so-far                           |
| 6  | Binary Tree Inorder Traversal          | 94    | Tree DFS               | iterative stack or Morris                  |
| 7  | Maximum Subarray (Kadane)              | 53    | Kadane / 1D DP         | rolling max ending here                    |
| 8  | Longest Substring No Repeating         | 3     | Sliding Window         | seen dict + left pointer                   |
| 9  | Linked List Cycle                      | 141   | Fast & Slow Pointers   | Floyd cycle detection                       |
| 10 | Number of Islands                       | 200   | Graph DFS / BFS        | flood fill                                  |
| 11 | Product of Array Except Self          | 238   | Prefix/Suffix          | left then right pass                        |
| 12 | Climbing Stairs                        | 70    | 1D DP (Fib)            | rolling two vars                            |
| 13 | Three Sum                              | 15    | Two Pointers           | sort + two-pointer for each i                |
| 14 | Merge Intervals                         | 56    | Merge Intervals        | sort by start, fuse                         |
| 15 | LRU Cache                              | 146   | Hash + DLL             | dict + doubly linked list                    |
| 16 | Course Schedule                         | 207   | Topo Sort / Cycle DFS  | Kahn's BFS indegree                         |
| 17 | Word Break                              | 139   | 1D DP                  | dp[i] = reachable from any j with dict       |
| 18 | Longest Palindromic Substring          | 5     | Expand Around Center    | odd/even expand                              |
| 19 | Combination Sum                         | 39    | Backtracking           | pick/skip with rollback                      |
| 20 | Kth Largest Element in an Array        | 215   | Quickselect / Heap      | min-heap of size k                           |
| 21 | Subtree of Another Tree                | 572   | Tree DFS + serialize    | compare serialized subtree                    |
| 22 | Coin Change                            | 322   | 1D DP                  | dp[amt] = 1 + min(dp[amt - c])              |
| 23 | Binary Tree Level Order Traversal     | 102   | Tree BFS               | deque + for-loop per level                   |
| 24 | Word Search                             | 79    | Backtracking + Grid    | DFS with visited mark                         |
| 25 | Trie (Implement)                        | 208   | Trie                   | nested dict, end flag                        |
| 26 | Top K Frequent Elements                | 347   | Heap / Bucket          | heapify counts, nlargest k                    |
| 27 | Longest Palindromic Subsequence         | 516   | 2D DP                  | dp[i][j] palindrome subseq                  |
| 28 | Edit Distance                           | 72    | 2D DP                  | dp[i][j] = min of insert/delete/replace     |
| 29 | Median of Two Sorted Arrays            | 4     | Binary Search          | partition pointers                            |
| 30 | Group Anagrams                          | 49    | Hash Map               | sorted-string-or-count key                   |

### Pattern → Practice Problems (Easy / Medium / Hard)

#### Prefix Sum
| Difficulty | Problem                  | LC  |
|------------|--------------------------|-----|
| Easy       | Running Sum of 1D Array  | 1480|
| Medium     | Subarray Sum Equals K    | 560 |
| Medium     | Continuous Subarray Sum  | 523 |
| Hard       | Count Submatrices Sum    | 1074|

#### Two Pointers
| Easy    | Move Zeroes                       | 283 |
| Medium  | Three Sum                         | 15  |
| Medium  | Container With Most Water         | 11  |
| Hard    | Trapping Rain Water               | 42  |

#### Sliding Window
| Easy    | Maximum Average Subarray I        | 643 |
| Medium  | Longest Substring No Repeating    | 3   |
| Medium  | Minimum Window Substring          | 76  |
| Hard    | Minimum Window Subsequence        | 727 |

#### Fast & Slow Pointers
| Easy    | Middle of the Linked List         | 876 |
| Medium  | Linked List Cycle II              | 142 |
| Medium  | Happy Number                      | 202 |
| Hard    | Find the Duplicate Number        | 287 |

#### Merge Intervals
| Medium  | Merge Intervals                   | 56  |
| Medium  | Insert Interval                   | 57  |
| Medium  | Meeting Rooms II                  | 253 |
| Hard    | Employee Free Time                | 759 |

#### Reverse / In-place LL Mutation
| Easy    | Reverse Linked List               | 206 |
| Medium  | Reverse Linked List II            | 92  |
| Medium  | Rotate List                        | 61  |
| Hard    | Reverse Nodes in K-Group           | 25  |

#### Hash Map / Frequency
| Easy    | Two Sum                            | 1   |
| Easy    | Contains Duplicate                | 217 |
| Medium  | Group Anagrams                    | 49  |
| Hard    | Longest Consecutive Sequence      | 128 |

#### Top K / Heap
| Easy    | Kth Largest Element in a Stream    | 703 |
| Medium  | Top K Frequent Elements           | 347 |
| Medium  | Kth Largest Element in an Array   | 215 |
| Hard    | Find Median from Data Stream      | 295 |

#### Binary Search
| Easy    | Binary Search                      | 704 |
| Medium  | Search in Rotated Sorted Array    | 33  |
| Hard    | Median of Two Sorted Arrays       | 4   |
| Hard    | Aggressive Cows / Split Array     | 410 |

#### Subset / Permutation Backtracking
| Medium  | Subsets                            | 78  |
| Medium  | Permutations                       | 46  |
| Medium  | Combination Sum                    | 39  |
| Hard    | N-Queens                           | 51  |
| Hard    | Word Search II                     | 212 |

#### Tree BFS (level order)
| Easy    | Same Tree                          | 100 |
| Medium  | Binary Tree Level Order Traversal  | 102 |
| Hard    | Serialize and Deserialize Binary Tree | 297 |

#### Tree DFS / Subtree
| Easy    | Maximum Depth of Binary Tree       | 104 |
| Medium  | Lowest Common Ancestor             | 236 |
| Medium  | Path Sum II                        | 113 |
| Hard    | Binary Tree Maximum Path Sum       | 124 |

#### Graph BFS
| Medium  | Number of Islands                  | 200 |
| Medium  | Rotting Oranges                    | 994 |
| Hard    | Word Ladder                        | 127 |

#### Graph DFS / Topo Sort
| Medium  | Course Schedule                    | 207 |
| Medium  | Clone Graph                        | 133 |
| Hard    | Alien Dictionary                    | 269 |

#### 0/1 Knapsack
| Medium  | Partition Equal Subset Sum         | 416 |
| Medium  | Coin Change 0/1 (variant)          | 416 |
| Hard    | Last Stone Weight II               | 1049|

#### Fibonacci / 1D DP
| Easy    | Climbing Stairs                    | 70  |
| Medium  | House Robber                       | 198 |
| Medium  | Coin Change                        | 322 |
| Medium  | Decode Ways                        | 91  |
| Hard    | Best Time to Buy/Sell IV           | 188 |

#### 2D String DP
| Medium  | Longest Common Subsequence         | 1143|
| Medium  | Longest Palindromic Subsequence    | 516 |
| Hard    | Edit Distance                      | 72  |
| Hard    | Distinct Subsequences              | 115 |

#### Trie / String Search
| Easy    | Implement Trie (Prefix Tree)       | 208 |
| Medium  | Word Search II                     | 212 |
| Hard    | Word Search II (with reuse forbidding) | 212 |

#### Union-Find / Disjoint Set
| Medium  | Number of Connected Components     | 323 |
| Medium  | Graph Valid Tree                   | 261 |
| Hard    | Accounts Merge                     | 721 |
| Hard    | Redundant Connection II            | 685 |

### FAANG Pattern Frequency Table

| Pattern                  | Meta | Google | Amazon | Apple | Microsoft | Netflix |
|--------------------------|------|--------|--------|-------|-----------|---------|
| Two Pointers             | High | Med    | Med    | High  | High      | Med     |
| Sliding Window           | High | High   | High   | Med   | Med       | High    |
| Hash Map                 | High | High   | High   | High  | High      | High    |
| Heap / Top K             | Med  | High   | Med    | Med   | Med       | Med     |
| Binary Search            | Med  | High   | Med    | High  | High      | Med     |
| Backtracking             | Med  | High   | Low    | Med   | Med       | Med     |
| Tree BFS/DFS             | High | Med    | High   | High  | High      | Med     |
| Graph BFS/DFS            | High | High   | High   | Med   | Med       | High    |
| DP (1D and 2D)           | High | High   | Med    | Med   | High      | High    |
| Trie                     | Low  | Med    | Low    | Low   | Low       | Med     |
| Union-Find               | Low  | Med    | Low    | Low   | Med       | Low     |
| Intervals                | Med  | Med    | High   | Low   | Med       | Med     |

(High = appears in ≥ 50% of reported rounds, Med = 15–50%, Low = <15%)

### Common Python Interview Gotchas

1. **List aliasing:** `a = [0]*n; b = a` shares references. `[[0]*m]*n` creates a list where every row is the **same** object. Use `[[0]*m for _ in range(n)]` instead.
2. **Mutable default args:** `def f(x, acc=[]):` — the list persists across calls. Use `acc=None` + `acc = [] if acc is None else acc`.
3. **Recursion limit:** Python's default ~1000. For deep trees/graphs, rewrite iterative or `sys.setrecursionlimit(10**6)`.
4. **No integer overflow** in Python 3 — `2**100` is fine. (Other languages' overflow gotchas aren't.)
5. **Floor division rounds toward negative infinity** — `(-1) // 2 == -1`. Use `math.floor` explicitly or `int(a/b)` for round-toward-zero.
6. **`copy()` vs `deepcopy()`:** for nested lists, `list.copy()` is shallow.
7. **`range` exclusive end:** off-by-one errors come from mixing inclusive pseudocode with Python's exclusive bounds.
8. **`collections.Counter.most_common(k)` is O(n log k)**, not O(n) — still faster than `sorted().[:k]`.
9. **`dict` preserves insertion order** (since 3.7). Useful for "first unique char" tricks.
10. **`heapq` is a min-heap.** For a max-heap, negate values or use `heapq.nlargest`.
11. **`str` is immutable.** Repeated `+=` on a string in a loop is O(n²). Use `"".join(parts)`.
12. **`is` vs `==`:** Never compare strings/numbers with `is` outside CPython's internal cache (small ints / interned strings). Use `==`.
13. **`sorted` returns a NEW list; `.sort()` sorts in place.** Mixing these up causes hours of debugging.
14. **`set` ordering is arbitrary.** Never depend on iteration order of a set.
15. **`a, b = b, a` is a true swap** thanks to tuple packing — but inside a loop with side effects (like increment) order matters.

---

### 🔗 Related Pattern Files (deep-dive per technique)

- [prefix_sum.md](../07_Algorithms/prefix_sum.md) — Pattern 1
- [two_pointers.md](../07_Algorithms/two_pointers.md) — Patterns 2, 4
- [sliding_window.md](../07_Algorithms/sliding_window.md) — Pattern 3
- [sorting.md](../07_Algorithms/sorting.md) — Patterns 5, 11 (used by merge intervals, binary search)
- [binary_search.md](../07_Algorithms/binary_search.md) — Pattern 10
- [searching.md](../07_Algorithms/searching.md) — Pattern 10 (linear + binary search)
- [recursion.md](../07_Algorithms/recursion.md) — Patterns 11 (backtracking), 13 (tree DFS), 15 (graph DFS)
- [big_o.md](../08_Time_Complexity/big_o.md) — Complexity reasoning behind every pattern

> Back to [README](../README.md)