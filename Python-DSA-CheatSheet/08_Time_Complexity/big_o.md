# Big-O Time & Space Complexity in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 08 — Time Complexity
> 🔗 Related: [prefix_sum.md](../07_Algorithms/prefix_sum.md) · [two_pointers.md](../07_Algorithms/two_pointers.md) · [sliding_window.md](../07_Algorithms/sliding_window.md) · [sorting.md](../07_Algorithms/sorting.md) · [binary_search.md](../07_Algorithms/binary_search.md) · [interview_patterns.md](../09_FAANG_Patterns/interview_patterns.md)
> Back to [README](../README.md)

---

## 1. What is it?

**Big-O** is a mathematical notation that describes the **upper bound on how an algorithm's runtime (or memory) grows as the input size `n` grows toward infinity**.

Key idea you must internalize:

- It is about **growth rate**, *not* actual seconds.
- Constants are dropped: `O(2n)`, `O(n + 5)`, `O(100n)` are all written **O(n)**.
- Lower-order terms are dropped: `O(n² + n + log n)` becomes **O(n²)**.
- We care about **large `n`** (asymptotic behavior). For small `n`, the constant factor matters more in practice.

Formally, `f(n) = O(g(n))` means there exist positive constants `c` and `n₀` such that `0 ≤ f(n) ≤ c·g(n)` for all `n ≥ n₀`. In interviews we use it loosely to mean "grows no faster than g".

**Real-world analogy:** Big-O is like estimating how long a road trip will take based only on the distance and the type of road (highway vs. city streets), ignoring traffic lights and weather.

---

## 2. Why do we use it?

- **Compare algorithms without running them.** Two sorting routines might both be O(n²) or one O(n²) and one O(n log n) — the difference is visible without benchmarking.
- **Predict scaling.** If your algorithm passes tests with `n = 1000` in 0.01 s, Big-O tells you whether it will pass `n = 10⁶` (rough rule: a Python solution does ~10⁷–10⁸ simple ops/sec).
- **Communicate tradeoffs.** "O(n) time, O(n) space" tells the interviewer immediately which resource you're trading.
- **Spot bugs early.** If you wrote O(n²) for an `n = 10⁶` pipeline, Big-O screams "this won't finish".
- **It's the gatekeeper.** Most FAANG interviews end the round with *"What's the time and space complexity?"* — getting this wrong often means "no hire".

---

## 3. When should I choose it? — Decision Table

| Situation                                                            | Which notation to report               |
|----------------------------------------------------------------------|----------------------------------------|
| Pure random access on a list / dict lookup                          | **O(1)**                               |
| Halving the search space each iteration (binary search)              | **O(log n)**                           |
| Single pass over an array / sliding window                          | **O(n)**                               |
| Sorting, merge sort, heap operations, divide & conquer splits        | **O(n log n)**                         |
| Two nested loops over `n`                                            | **O(n²)**                              |
| Three nested loops (`n³`)                                            | **O(n³)**                              |
| Generating all subsets / combinations                                | **O(2ⁿ)**                              |
| Generating all permutations                                           | **O(n!)**                              |
| Trial division loop up to √n                                         | **O(√n)**                              |
| Recursive series `T(n) = T(n-1) + O(1)`                              | **O(n)**                               |
| Recursive series `T(n) = T(n-1) + O(n)`                              | **O(n²)**                              |
| Recursive series `T(n) = 2·T(n/2) + O(n)`                           | **O(n log n)**                         |
| Recursive series `T(n) = 2·T(n/2) + O(1)`                           | **O(n)**                               |
| Recursive series `T(n) = T(n-1) + T(n-2)` (Fib recursion)            | **O(2ⁿ)** (more precisely O(φⁿ))       |

---

## 4. Syntax

Big-O is *written notation*, not code. The conventions:

```
O(1)            # constant
O(log n)        # logarithmic  (base irrelevant — log₂ n and log₁₀ n differ by a constant factor)
O(n)            # linear
O(n log n)      # linearithmic
O(n²)           # quadratic
O(n³)           # cubic
O(2ⁿ)           # exponential
O(n!)           # factorial
O(√n)           # square root
```

When writing complexity, separate **time** from **space**:

```
Time:  O(n log n)
Space: O(n)        # extra space, NOT counting the input
```

If the recursion stack is meaningful, include it in space:

```
Space: O(h) recursion stack   # h = height of recursion tree
```

---

## 5. Basic Example

```python
# Example 1: O(1)  — constant time
def first_two(nums):
    return nums[0] + nums[1]           # always 2 ops regardless of n

# Example 2: O(log n) — halving
def count_bits(n):
    count = 0
    while n:
        n >>= 1                        # halve n each iteration
        count += 1
    return count

# Example 3: O(n) — single pass
def sum_list(nums):
    total = 0
    for x in nums:                     # n iterations
        total += x
    return total

# Example 4: O(n log n) — Python's TimSort
def sort_then_return(nums):
    nums.sort()                        # O(n log n)
    return nums

# Example 5: O(n²) — nested loops
def all_pairs(nums):
    out = []
    for i in range(len(nums)):          # n outer
        for j in range(len(nums)):      # n inner
            out.append((nums[i], nums[j]))
    return out                          # n*n = n² appends

# Example 6: O(2ⁿ) — recursion produces 2ⁿ leaves
def subsets(s, i=0, acc=None):
    if acc is None: acc = []
    if i == len(s):
        yield acc
        return
    yield from subsets(s, i+1, acc)                    # exclude
    yield from subsets(s, i+1, acc + [s[i]])            # include
```

---

## 6. Step-by-Step Dry Run

**Problem:** Analyze the following snippet.

```python
def mystery(nums):
    n = len(nums)                  # L1
    s = set(nums)                 # L2
    result = []                    # L3
    for x in nums:                 # L4
        if x * 2 in s:            # L5
            result.append(x)       # L6
    result.sort()                  # L7
    return result                  # L8
```

| Line | Operation                            | Cost        | Why                                |
|------|--------------------------------------|-------------|------------------------------------|
| L1   | `len(nums)`                          | O(1)        | Python list stores its size        |
| L2   | Build `set` from list                | O(n)        | n hashes + n inserts (avg O(1))    |
| L3   | empty list init                      | O(1)        |                                    |
| L4   | outer loop runs n times             | —           | multiplier                         |
| L5   | `x * 2 in s` (set lookup)           | O(1) avg    | hash table                          |
| L6   | `append`                            | O(1) amrt   |                                     |
| L7   | `result.sort()` (≤ n elements)      | O(n log n)  | TimSort                             |
| L8   | return reference                     | O(1)        |                                     |

**Totals:**
- Time:  O(n) [L2] + O(n)·O(1) [L4-L6] + O(n log n) [L7] = **O(n log n)**
- Space: O(n) [the set s] + O(n) [result] = **O(n)**

**Worst-case refinement:** If every element pairs with some other element, `result` has n entries → sort remains O(n log n). If `result` were at most 2 entries, sort would be O(1) and total would be O(n). Always state assumptions.

---

## 7. Built-in Methods — Common Python Operation Costs

### List operations

| Operation                       | Average       | Worst        | Notes                                  |
|---------------------------------|---------------|--------------|----------------------------------------|
| `lst[i]` indexing               | O(1)          | O(1)         |                                        |
| `lst[i] = v` assignment         | O(1)          | O(1)         |                                        |
| `lst.append(v)`                 | O(1) amortized| O(n) (rare)  | doubling capacity, occasional realloc   |
| `lst.pop()`                    | O(1)          | O(1)         | last element                           |
| `lst.pop(0)` / `lst.insert(0,v)` | O(n)        | O(n)         | shifts all elements                    |
| `lst.insert(i, v)`             | O(n)          | O(n)         | shifts n-i elements                    |
| `del lst[i]`                   | O(n)          | O(n)         | shifts                                 |
| `lst.remove(v)`                | O(n)          | O(n)         | search + shift                         |
| `lst.index(v)` / `v in lst`    | O(n)          | O(n)         | linear scan                            |
| `lst.count(v)`                 | O(n)          | O(n)         |                                        |
| `lst.sort()`                   | O(n log n)    | O(n log n)   | TimSort                                |
| `sorted(lst)`                  | O(n log n)    | O(n log n)   | new list                               |
| `lst + lst2`                   | O(n + m)      | O(n + m)     | new list                               |
| `lst * k`                      | O(nk)         | O(nk)        | repeats references                     |
| slicing `lst[i:j]`             | O(k)          | O(k)         | k = slice length                       |
| reversing `lst[::-1]`          | O(n)          | O(n)         |                                        |

### Dict operations

| Operation              | Average | Worst                                            |
|------------------------|---------|--------------------------------------------------|
| `d[k]` read            | O(1)    | O(n) (rare hash collisions, pathological inputs) |
| `d[k] = v`             | O(1)    | O(n)                                            |
| `del d[k]`             | O(1)    | O(n)                                            |
| `k in d`               | O(1)    | O(n)                                            |
| iterate `for k in d`   | O(n)    | O(n)                                            |
| `len(d)`               | O(1)    | O(1)                                            |

### Set operations

| Operation                       | Average | Worst    |
|---------------------------------|---------|----------|
| `s.add(v)`                       | O(1)    | O(n)      |
| `s.discard(v)` / `s.remove(v)`  | O(1)    | O(n)      |
| `v in s`                         | O(1)    | O(n)      |
| `s & t`, `s | t`, `s - t`       | O(min(s,t)) / O(len(s)+len(t)) | varies |
| `len(s)`                         | O(1)    | O(1)      |

### String operations

| Operation                       | Cost                  | Notes                                                  |
|---------------------------------|-----------------------|--------------------------------------------------------|
| `s[i]`                          | O(1)                  | strings are immutable arrays                           |
| `len(s)`                        | O(1)                  |                                                        |
| `s + t`                         | O(n + m)              | builds a NEW string each time                          |
| `"".join(parts)`                | O(total length)       | one allocation — always preferred over `+=` in loop    |
| `s += t`                        | O(n + m)              | repeated `+=` in loop → **O(n²)** gotcha               |
| slicing `s[i:j]`                | O(k)                  | k = slice length                                       |
| `s.find(sub)` / `sub in s`      | O(n·m) worst, often O(n) | naive substring search                              |
| `s.split()`                     | O(n)                  |                                                        |
| `s.replace(a, b)`               | O(n)                  |                                                        |
| `s[::-1]`                       | O(n)                  |                                                        |

### Heap operations (`heapq`)

| Operation                              | Cost       |
|----------------------------------------|------------|
| `heapify(lst)`                          | O(n)        | uses Floyd's algorithm, NOT O(n log n)              |
| `heappush(h, v)`                       | O(log n)    |                                                  |
| `heappop(h)`                           | O(log n)    |                                                  |
| `heappushpop(h, v)`                    | O(log n)    | slightly cheaper than push then pop              |
| `heapreplace(h, v)`                    | O(log n)    | pop then push atomically                         |
| building by repeated `heappush`        | O(n log n)  | use `heapify` instead for O(n)                   |

### Deque (`collections.deque`)

| Operation         | Cost       |
|-------------------|------------|
| `d.append` / `d.appendleft` | O(1) |
| `d.pop` / `d.popleft`      | O(1) |
| `d[i]`                     | O(n)  |
| `d.rotate(k)`              | O(k)  |

---

## 8. Interview Example

**Problem (LeetCode 1 — Two Sum):** Given an array of integers `nums` and an integer `target`, return indices of the two numbers that add up to `target`.

### Brute force — O(n²) time, O(1) space

```python
def twoSum(nums, target):
    for i in range(len(nums)):
        for j in range(i+1, len(nums)):
            if nums[i] + nums[j] == target:
                return [i, j]
```

### Optimal — O(n) time, O(n) space

```python
def twoSum(nums, target):
    seen = {}                          # val -> index
    for i, v in enumerate(nums):       # O(n)
        need = target - v
        if need in seen:               # O(1) avg
            return [seen[need], i]
        seen[v] = i                    # O(1) avg
```

**Stating complexity to the interviewer:**

> "Time is O(n) — we make a single pass over the array; each `in` and assignment on the dict is O(1) amortized.  
> Space is O(n) — in the worst case we store nearly every element in `seen` before finding the match."

---

## 9. When NOT to use

- **Don't micro-optimize constants on a non-competitive algorithm.** O(n²) at n=20 beats O(n log n) + huge constant.
- **Don't conflate worst case with average case.** Hash maps are O(1) *amortized average*; adversarial inputs can degrade to O(n). State which you mean.
- **Don't pick the lowest Big-O if memory pressure forbids it.** An O(n) space solution may be impossible at n=10⁸.
- **Don't use Big-O to compare two algorithms of the **same class** with very different constants** — benchmark instead (e.g. dict vs. list of tuples for small n).
- **Don't mislabel amortized as worst case.** `list.append` is O(1) amortized even though individual ops can be O(n).

---

## 10. Common Mistakes

1. **`O(n + n) = O(n)`, not `O(n²)`.** Two sequential passes are linear, not quadratic. Quadratic needs *nested* iteration over the same data.
2. **Confusing nested loops over different-sized inputs.** `for x in A: for y in B:` is `O(|A|·|B|)`, not `O(n²)` unless `|A| = |B| = n`.
3. **Hidden loops in string concatenation.**
   ```python
   s = ""
   for c in parts: s += c       # O(n²)  — each += copies the entire string
   ```
   Use `"".join(parts)` → O(n).
4. **Hidden loops in slicing.** `for i in range(n): x = s[:i]` is O(n²), not O(n) — each slice is O(i).
5. **Ignoring the recursion cost.** Naive Fibonacci `f(n) = f(n-1) + f(n-2)` has `T(n) = T(n-1) + T(n-2) + O(1)` which is **O(2ⁿ)**, not O(n).
6. **`T(n) = T(n-1) + O(n)` is O(n²), not O(n).** The +O(n) inside the recurrence means each level does O(n) work over n levels.
7. **Treating Python's `in` as always O(1).** On a list it is O(n); on a set/dict it is O(1) average.
8. **Forgetting the recursion stack in space.** Recursive DFS on a tree is O(h) space even if you allocate no extra data.
9. **Counting heap build as O(n log n).** It's O(n) via `heapify`; only repeated `heappush` is O(n log n).
10. **Confusing input size.** If your input is a number `k` **not** an array of size `k`, then `O(k)` algorithms may actually be `O(2^bits)` (where bits = log k). Classic trap: trial division up to √k is **O(2^(bits/2))** from a complexity theory POV, which is **exponential in input length**.
11. **`O(1)` ≠ "instant".** It means constant w.r.t. n. The constant might be a billion.
12. **Best case misrepresented as worst case.** Quicksort's best case is O(n log n), worst case is O(n²). Always say which you're reporting.

---

## 11. Memory Tricks

- **"Drop the constants, drop the sidekicks."** Big-O keeps only the *fastest-growing term* and the *first* constant multiplier.
- **"Nested loops = multiply. Sequential steps = add."** `for …: for …:` → ×. `for …:` then `for …:` → +.
- **Halving each step → log n.** Doubling each step → 2ⁿ.
- **Master Theorem cheat sheet:**
  - `T(n) = a·T(n/b) + f(n)` with `f(n) = O(n^d)`
  - if `a > b^d` → O(n^(log_b a))
  - if `a = b^d` → O(n^d · log n)
  - if `a < b^d` → O(n^d)
- **Sort once = O(n log n).** Sort twice in a row is still O(n log n).
- **Python line counts:** ~10⁷ simple ops/sec → useful for sanity-checking LeetCode time limits.

---

## 12. Interview Shortcuts

- **Always state complexity BEFORE coding.** Say: *"I'll aim for O(n) time and O(1) space — here's the plan…"* Then implement to match the promise.
- **Best / worst / average.** When the interviewer asks "what's the complexity?" answer with three if they differ. Otherwise state the worst.
- **Trade space for time, not the reverse**, unless explicitly asked. Most FAANG follow-ups are "can you do it in less space?"
- **LeetCode N → complexity table (since servers cap ~10⁷ ops):**

| Max `n`           | Expected complexity                       |
|-------------------|-------------------------------------------|
| `≤ 10`            | O(n!) (permutations)                       |
| `≤ 20`            | O(2ⁿ)  (subsets, bitmask DP)             |
| `≤ 100`           | O(n³) (Floyd-Warshall, all 3-tuples)     |
| `≤ 1,000`         | O(n²)                                     |
| `≤ 10⁵`           | O(n log n) or O(n)                        |
| `≤ 10⁶`           | O(n) strictly                              |
| `≤ 10⁹`           | O(log n) or O(1) (binary search, math)   |

- **One-word meaning of each class:**
  - O(1) → constant  · O(log n) → halving  · O(n) → linear scan  · O(n log n) → sort  · O(n²) → pairwise  · O(2ⁿ) → yes/no choices  · O(n!) → arrangements
- **Space shorthand:** auxiliary space + recursion stack. Do **not** count input or output unless the interviewer says so.

---

## 13. Cheat Sheet Table

| Class          | Growth examples                                  | Recognizable from                            |
|----------------|--------------------------------------------------|----------------------------------------------|
| O(1)           | direct access, math                              | `lst[i]`, `d[k]`, `if-else`                  |
| O(log n)       | binary search, bit iteration                     | `n //= 2`, `l, r = mid+1, r`                 |
| O(√n)          | trial division prime check                       | `while i*i <= n`                             |
| O(n)           | single pass, two pointers, sliding window        | `for x in arr:`                              |
| O(n log n)     | sort, heap ops, D&C merges                       | `.sort()`, `heappush` n times                 |
| O(n²)          | two nested loops, naive DP table                 | `for i: for j:`                              |
| O(n³)          | triple nested, matrix multiplication naive        | `for i: for j: for k:`                       |
| O(2ⁿ)          | subset recursion, bitmask brute                  | `solve(i+1, include/exclude)`                |
| O(n!)          | permutation recursion                            | `for i in remaining: recurse(rest)`          |

---

## 14. Time Complexity Table — Common Algorithms

| Algorithm                                       | Best            | Average          | Worst            | Space            |
|-------------------------------------------------|-----------------|------------------|------------------|------------------|
| Linear search (unsorted list)                   | O(1)            | O(n)             | O(n)             | O(1)             |
| Binary search (sorted array)                    | O(1)            | O(log n)         | O(log n)         | O(1) iter / O(log n) rec |
| Bubble sort                                     | O(n)            | O(n²)            | O(n²)            | O(1)             |
| Insertion sort                                  | O(n)            | O(n²)            | O(n²)            | O(1)             |
| Selection sort                                  | O(n²)           | O(n²)            | O(n²)            | O(1)             |
| Merge sort                                      | O(n log n)      | O(n log n)       | O(n log n)       | O(n)             |
| Quick sort                                      | O(n log n)      | O(n log n)       | O(n²)            | O(log n) stack   |
| TimSort (Python's `.sort()`)                    | O(n)            | O(n log n)       | O(n log n)       | O(n)             |
| Heap sort                                       | O(n log n)      | O(n log n)       | O(n log n)       | O(1)             |
| BFS / DFS on graph with V vertices, E edges     | O(V + E)        | O(V + E)         | O(V + E)         | O(V)             |
| Dijkstra (binary heap)                          | O((V + E) log V)| O((V + E) log V) | O((V + E) log V) | O(V)             |
| Bellman-Ford                                    | O(V·E)          | O(V·E)           | O(V·E)           | O(V)             |
| Floyd-Warshall                                  | O(V³)           | O(V³)            | O(V³)            | O(V²)            |
| Union-Find (with path compression + rank)       | O(α(n)) ≈ O(1)  |                  |                  | O(n)             |
| Kadane (max subarray)                           | O(n)            | O(n)             | O(n)             | O(1)             |
| 1D dynamic programming (LIS, Kadane, fib)       | O(n)            | O(n)             | O(n)             | O(n)             |
| 2D DP (LCS, edit distance, knapsack)            | O(n·m)          | O(n·m)           | O(n·m)           | O(n·m) or O(min(n,m)) |
| Matrix chain multiplication                    | O(n³)           | O(n³)            | O(n³)            | O(n²)            |
| Backtracking subsets                            | O(2ⁿ·n)         | —                | O(2ⁿ·n)         | O(n) stack       |
| Backtracking permutations                       | O(n!·n)         | —                | O(n!·n)         | O(n) stack       |
| Trie insert/search                              | O(L)            | O(L)             | O(L)             | O(Σ·L) alphabet size × length |

---

## 15. Visual Diagram (ASCII)

### 15.1 Growth comparison chart (relative heights)

```
n          O(1)   O(log n) O(√n)  O(n)   O(n log n) O(n²)  O(2ⁿ)   O(n!)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1          ▏      ▏        ▏      ▏      ▏          ▏      ▏        ▏
4          ▏      ▎        ▐      ▐      ▐         ▐      ▐        ▐
16         ▏      ▍        ▎      ▍      ▍         ▍      ▎        ▐
64         ▏      ▌        ▌      ▌      ▌         █▌     ▌        █▌
256        ▏      ▋        ▌      ▋      ▋         ██▌    ▋        █████
1024       ▏      ▊        ▊      ▊      ▊         ████▌  ▊        ███ (overflow)
10⁴        ▏      ▉        ▉      ▉      ▉         ████▌  ▉        ███
10⁵        ▏      ▉        ▉      ▉      ▉         █████▌ █        ███
10⁶        ▏      █        █      █      █▌        ███████ ▌        ███
```
(`▏` < 1, `▎` ~ log n, `▐` linear, `█` ≥ quadratic/exp)

### 15.2 "Is my algorithm fast enough?" flowchart (by LeetCode n)

```
                    ┌── What's the max n in constraints? ──┐
                                                              │
   n ≤ 10          ─► O(n!)  / O(2ⁿ) allowed   (backtracking, perms)
   n ≤ 20          ─► O(2ⁿ)  or  O(n²·2ⁿ) bitmask DP
   n ≤ 100         ─► O(n³) allowed            (Floyd, 3-sum-ish DP)
   n ≤ 1,000       ─► O(n²) allowed            (pairwise, naive DP)
   n ≤ 10⁵         ─► O(n log n) or O(n)       (sort, two ptr, sliding)
   n ≤ 10⁶         ─► O(n) strictly             (one-pass, prefix sum)
   n ≤ 10⁹         ─► O(log n) or O(1)          (binary search, math)
                    └────────────────────────────┘
```

### 15.3 Deciding complexity of a code block (flow)

```
                 ┌──── Are there loops? ────┐
                 │                          │
              NO │                          │ YES
                 ▼                          ▼
            Constant?                  How many nested loops?
              O(1)                          │
                                  1 nested  → × n   →  is inner O(1)?  →  O(n)
                                              │ NO ─ ratio of inner sizes (m, k) → O(n·m)
                                  2 nested   → × n × n  →  O(n²)
                                  k nested   → × n ᵏ       →  O(nᵏ)
                ▼
       Does the loop HALVE / divide n per step?
              YES → O(log n)
              NO  → linear over the divided set (e.g. O(√n) for step *=2)
                ▼
       Is there RECURSION?
              ├─ branches b, each 1/b of input → Master Theorem
              ├─ T(n) = T(n-1) + O(1)        → O(n)
              ├─ T(n) = T(n-1) + O(n)        → O(n²)
              ├─ T(n) = 2·T(n/2) + O(1)      → O(n)
              └─ T(n) = 2·T(n/2) + O(n)      → O(n log n)
```

### 15.4 Time vs Space trade-off mental model

```
       Time                                                          Space
  ┌────────────┐                                                  ┌────────────┐
  │   O(n²)    │  ─── trade space for speed ──►  ┌────────────┐  │   O(n)     │
  │  (brute)   │                                  │   O(n)     │  │ (memo/dict)│
  └────────────┘                                  └────────────┘  └────────────┘
                                                       ▲
                                                       │  memoization / hash map
                                                       │  / prefix sum / deque
                                                       │
  The universal FAANG pattern: O(n²)→O(n) trade space,
  or O(n)→O(log n) trade data structure (heap/segment tree).
```

---

## 16. Beginner Notes

> Remember:
> 
> 1. Big-O is **growth rate**, not seconds. Don't trust benches, trust the math.
> 2. **Drop constants**, **drop lower-order terms**. `O(3n² + 7n + 99)` = `O(n²)`.
> 3. **Nested loops multiply, sequential steps add.**
> 4. **`in` on a list is O(n); `in` on a set/dict is O(1) avg.** This single fact unlocks hash-map optimizations for dozens of problems.
> 5. **`for x in arr: s += x`** on a list is fine (amortized O(1)), but on a **string** it is O(n²). Always use `"".join()`.
> 6. **Sort is O(n log n)** — when you see sorted input, your algorithm should beat or match it; don't undo the work.
> 7. **Recursion costs stack space** — O(h) where h is recursion depth. Python's default recursion limit is ~1000.
> 8. **State tbefore coding.** Always tell the interviewer: *"This will be O(__) time and O(__) space."*
> 9. **`n` in constraints lies** — read them carefully. Constraints `nums.length ≤ 20` is your signal to consider `O(2ⁿ)`.
> 10. **Best ≠ Average ≠ Worst.** Say which you're describing.

---

## 17. FAANG Tips

- **Always announce complexity before writing code.** Interviewers want to hear: *"I'm going to do it in O(n) time and O(n) space using a hashmap. Brute would be O(n²), but…"*
- **Optimize iteratively.** Walk from brute → improved → optimal. Show the interviewer why each step is faster.
- **Idle moments at the end → mention a follow-up.** *"If the array were sorted I could do two pointers for O(1) space."*
- **Watch for tricks:**
  - Sorted input → binary search / two pointer saves a factor.
  - "K largest/smallest" → heap (don't sort the whole array).
  - "Subarray sum/product/count" → prefix sum + hash map.
  - "Path/sequence/min-cost" → graph DP.
  - "Tree, all roots → leaves" → think twice about whether space is height or width.
- **Space gotcha:** Recursive DFS on a degenerate linked list is O(n) stack — same as iterative. Always ask the interviewer if they consider the recursion stack as "space".
- **Hash collisions:** When asked "what's the worst case for dict?" answer *"O(n) if many collisions; in practice Python's hash randomization makes adversarial collision hard."*
- **Inputs large**: At n=10⁷ and beyond, prefer in-place algorithms. Page-cache misses on a huge aux array can dominate.
- **Use `n` consistently.** Don't say "O(m)" if the interviewer introduced only `n`. Use their variable names.

---

## 18. Practice Problems

### Easy
| Problem                                   | LeetCode | Hint                                  |
|-------------------------------------------|----------|---------------------------------------|
| Contains Duplicate                        | 217      | set insert O(n); check len diff       |
| Two Sum                                   | 1        | hashmap complement lookup             |
| Best Time to Buy and Sell Stock           | 121      | one pass O(n), O(1)                   |
| Valid Anagram                             | 242      | counter or sorted, both acceptable    |
| Palindrome Number                         | 9        | reverse half — O(log n) time          |
| Single Number                             | 136      | XOR, O(n) O(1)                        |
| Merge Sorted Array                        | 88       | fill from end O(m+n)                  |
| Majority Element                           | 169      | Boyer-Moore majority vote O(n)        |

### Medium
| Problem                                   | LeetCode | Hint                                  |
|-------------------------------------------|----------|---------------------------------------|
| Top K Frequent Elements                   | 347      | heap or bucket sort O(n log k)        |
| Longest Palindromic Substring             | 5        | expand around center O(n²), O(1)     |
| Word Break                                | 139      | DP O(n²·m)                            |
| Longest Substring Without Repeating Char  | 3        | sliding window + dict O(n)            |
| Product of Array Except Self              | 238      | prefix/suffix O(n), O(1) extra        |
| Coin Change                               | 322      | DP O(n·amount)                        |
| Kth Largest Element in an Array           | 215      | heap n log k or quickselect avg O(n)  |

### Hard
| Problem                                   | LeetCode | Hint                                  |
|-------------------------------------------|----------|---------------------------------------|
| Median of Two Sorted Arrays               | 4        | binary search O(log(min(n,m)))        |
| Minimum Window Substring                  | 76       | sliding window O(m+n)                |
| Maximum Profit with K Transactions        | 188      | DP O(n·k), space O(k) rolling         |
| Cherry Pickup                             | 741      | DP O(n³)                              |
| Word Ladder                               | 127      | bidirectional BFS O(L·min(|A|,|B|))   |
| Word Break II                             | 140      | memoized DFS O(2ⁿ) worst              |
| Number of Ways to Form a Target String     | 1639     | DP with prefix sums O(n·m)            |

---

### 🔗 Related Algorithm Files

- [prefix_sum.md](../07_Algorithms/prefix_sum.md) — O(n) precompute, O(1) per query
- [two_pointers.md](../07_Algorithms/two_pointers.md) — typically O(n)
- [sliding_window.md](../07_Algorithms/sliding_window.md) — O(n)
- [sorting.md](../07_Algorithms/sorting.md) — O(n log n) bounds
- [searching.md](../07_Algorithms/searching.md) — linear vs binary search
- [binary_search.md](../07_Algorithms/binary_search.md) — O(log n)
- [recursion.md](../07_Algorithms/recursion.md) — recursion tree analysis
- [interview_patterns.md](../09_FAANG_Patterns/interview_patterns.md) — pattern → complexity cheat-sheet

> Back to [README](../README.md)