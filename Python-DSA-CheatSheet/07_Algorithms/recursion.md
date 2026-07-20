# Recursion in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 07 — Algorithms
> 🔗 Related: [sorting.md](./sorting.md) · [binary_search.md](./binary_search.md) · [searching.md](./searching.md)
> Data: [list.md](../02_Data_Types/list.md) · [big_o.md](../08_Time_Complexity/big_o.md)
> Back to [README](../README.md)

---

## 1. What is it?

**Recursion** is a function that **calls itself** to solve a smaller instance of the same problem, until it reaches a **base case** that can be answered directly.

Every recursive solution has two essential pieces:

1. **Base case** — the smallest, trivial version of the problem; returns immediately, no recursive call.
2. **Recursive case** — reduces the problem toward the base case and calls itself on the reduction.

A classical example:

```python
def factorial(n):
    if n <= 1:                       # base case
        return 1
    return n * factorial(n - 1)      # recursive case
```

Recursion uses the **call stack** automatically: each call pushes a new frame (locals, return address); when the base case is hit, frames unwind and combine results.

**What problem it solves:** problems with a **self-similar sub-structure** — trees, graphs, divide-and-conquer (merge sort, quick sort), combinations/permutations, backtracking, dynamic programming (top-down), expression parsing.

**Real-world analogy:** To find out how tall a stack of coins is, measure one coin and ask your friend "how tall is the rest of the stack?" — your friend asks the next friend, and so on, until one coin is left (base case), then they each add their measurement on the way back up.

---

## 2. Why do we use it?

- **Cleanest expression of self-similar problems**: trees, nested structures, divide-and-conquer.
- **Replaces explicit stack management**: the call stack _is_ your stack data structure.
- **Backtracking** is naturally expressed recursively (combinations, permutations, N-Queens).
- **Divide-and-conquer** algorithms (merge sort, quick sort, binary search recursion variant) read more clearly recursive.
- **Top-down DP** = recursion + memoization — define the recurrence, let cache prune recomputation.

---

## 3. When should I choose it? — Decision Table

| Situation                                                | Recursion?                | Reason                                     |
|----------------------------------------------------------|---------------------------|--------------------------------------------|
| Tree/graph traversal                                     | Yes                       | Structure is inherently recursive          |
| Divide & conquer (merge sort, quick sort, Fibonacci)    | Yes                       | Recurrence naturally recursive              |
| Combinations / permutations / subsets                   | Yes (backtracking)        | Recursive choices + undo                  |
| Top-down dynamic programming                             | Yes + memoization         | Recurrence + caching                       |
| Simple linear iteration (`for i in range(n)`)            | No                        | Iteration is clearer and cheaper           |
| Pipeline of stateful transformations                     | No                        | Use loops + state vars                     |
| Depth could exceed 1000                                  | No (or increase limit)    | Python default recursion limit is 1000     |
| Performance-critical hot path                            | Often no                  | Function call overhead; unnecessary stack  |
| In-order traversal of a balanced tree                    | Either                    | Iterative with explicit stack avoids limit |

See also [sorting.md](./sorting.md) for merge-sort and quick-sort recursion trees.

---

## 4. Syntax

```python
# canonical shape
def rec(args):
    if <base condition>:
        return <trivial answer>     # NO recursive call here
    # reduce the problem
    return <combine>(rec(<smaller args>))

# Python recursion limit
import sys
print(sys.getrecursionlimit())     # 1000 default
sys.setrecursionlimit(5000)        # raise if you know your tree depth

# memoization via functools.lru_cache
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)
```

---

## 5. Basic Example

### Factorial (linear recursion)

```python
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)

print(factorial(5))     # 120
```

### Fibonacci (naive — exponential!) vs memoized

```python
def fib_naive(n):
    if n < 2:
        return n
    return fib_naive(n-1) + fib_naive(n-2)

from functools import lru_cache

@lru_cache(maxsize=None)
def fib_memo(n):
    if n < 2:
        return n
    return fib_memo(n-1) + fib_memo(n-2)

print(fib_naive(10))   # 55
print(fib_memo(50))   # 12586269025 (memoized makes 50 fast)
```

### Sum of a list (uses recursive structure)

```python
def sum_list(a):
    if not a:
        return 0
    return a[0] + sum_list(a[1:])

print(sum_list([1,2,3,4]))   # 10
```

### Power of a binary tree — count nodes

```python
class Node:
    def __init__(self, val, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def count_nodes(root):
    if root is None:
        return 0
    return 1 + count_nodes(root.left) + count_nodes(root.right)
```

### Binary recursion — classic merge sort (see sorting.md)

```python
def merge_sort(a):
    if len(a) <= 1:           # base
        return a
    mid = len(a) // 2
    return merge(merge_sort(a[:mid]), merge_sort(a[mid:]))

def merge(L, R):
    out, i, j = [], 0, 0
    while i < len(L) and j < len(R):
        if L[i] <= R[j]:
            out.append(L[i]); i += 1
        else:
            out.append(R[j]); j += 1
    out.extend(L[i:]); out.extend(R[j:])
    return out
```

---

## 6. Step-by-Step Dry Run

### `factorial(4)` — recursion and unwind

```
Call stack grows downward; unwinds back up.

  factorial(4)            # n=4, not base; call factorial(3), then *
    factorial(3)          # n=3, not base; call factorial(2), then *
      factorial(2)        # n=2, not base; call factorial(1), then *
        factorial(1)      # BASE CASE -> returns 1
      2 * 1 = 2           # unwinds -> factorial(2) returns 2
    3 * 2 = 6             # unwinds -> factorial(3) returns 6
  4 * 6 = 24              # unwinds -> factorial(4) returns 24
```

ASCII call-stack pushing and popping:

```
  push:  [fact(1)]                  (top of stack)
         [fact(2)]
         [fact(3)]
         [fact(4)]
         [main]
  pop:   fact(1) -> 1
         fact(2) -> 2 * 1 = 2
         fact(3) -> 3 * 2 = 6
         fact(4) -> 4 * 6 = 24
         main prints 24
```

### Fibonacci `fib_naive(5)` recursion tree

```
                    f(5)
                  /      \
              f(4)         f(3)
             /    \        /   \
          f(3)   f(2)  f(2)  f(1)
         /  \    / \    / \   (=1)
      f(2) f(1) f(1) f(0) f(1) f(0)
      (=1) (=1) (=1) (=0) (=1) (=0)

  Notice f(3) computed twice, f(2) thrice — exponential blowup.
  With memoization, each fib(k) computed once -> O(n).
```

### `sum_list([1,2,3,4])`

```
sum_list([1,2,3,4]) = 1 + sum_list([2,3,4])
                       = 1 + 2 + sum_list([3,4])
                                  = 1 + 2 + 3 + sum_list([4])
                                              = 1 + 2 + 3 + 4 + sum_list([])
                                                              = 1+2+3+4+0 = 10
```

---

## 7. Built-in Methods   (relevant to recursion)

### 7.1 `functools.lru_cache(maxsize=128, typed=False)`
- **Purpose**: memoize a function based on its arguments; O(1) lookup in an LRU dict.
- **Syntax**: `@lru_cache(maxsize=None)` (None = unlimited).
- **Example**: `fib(50)` returns instantly; naive version takes minutes.
- **Complexity**: turns `fib` from O(2^n) to O(n) time, O(n) space.
- **Interview use**: top-down DP — knapsack, coin change, decode ways, climb stairs.
- **Mistakes**: arguments must be **hashable** (no lists/tuples-of-lists unless frozen). Cache can be inspected with `.cache_info()`.
- **Shortcut**: for unhashable inputs, use a manual `dict` cache keyed by indices.

### 7.2 `functools.cache`
- Equivalent to `lru_cache(maxsize=None)`, available since Python 3.9.
- Slightly faster than `lru_cache` because it doesn't track LRU ordering.

### 7.3 `sys.setrecursionlimit(n)`
- **Purpose**: increase max recursion depth — Python default is **1000**.
- **Caveats**: only raises the *Python interpreter limit*; the C stack is finite. Setting very high values (e.g. 100000) can crash Python with a segfault.
- **Interview use**: occasionally needed for deep recursion on balanced trees with n ~ 10⁴.
- **Mistakes**: thinking it makes deep recursion safe — better to convert to iteration.

### 7.4 `sys.getrecursionlimit()` — current limit (default 1000).

### 7.5 Generator-based recursion — `yield from`

For recursion that produces many items (subsets, permutations), `yield from` lets you delegate to a recursive generator cleanly:

```python
def subsets(nums):
    def rec(i, path):
        if i == len(nums):
            yield path
            return
        yield from rec(i+1, path)
        yield from rec(i+1, path + [nums[i]])
    yield from rec(0, [])
```

---

## 8. Interview Example

### LeetCode 21 — Merge Two Sorted Lists (recursive)

```python
def mergeTwoLists(l1, l2):
    if not l1: return l2
    if not l2: return l1
    if l1.val < l2.val:
        l1.next = mergeTwoLists(l1.next, l2)
        return l1
    else:
        l2.next = mergeTwoLists(l1, l2.next)
        return l2
```

### LeetCode 206 — Reverse Linked List (recursive)

```python
def reverseList(head):
    if not head or not head.next:
        return head
    p = reverseList(head.next)
    head.next.next = head      # reverse the link
    head.next = None            # break old forward link
    return p
```

### LeetCode 77 — Combinations (backtracking)

```python
def combine(n, k):
    res = []
    def rec(start, path):
        if len(path) == k:
            res.append(path[:])
            return
        for i in range(start, n + 1):
            path.append(i)
            rec(i + 1, path)
            path.pop()         # backtrack
    rec(1, [])
    return res
```

### LeetCode 46 — Permutations (backtracking)

```python
def permute(nums):
    res = []
    def rec(path,used):
        if len(path) == len(nums):
            res.append(path[:])
            return
        for i in range(len(nums)):
            if used[i]:
                continue
            used[i] = True
            path.append(nums[i])
            rec(path, used)
            path.pop()
            used[i] = False
    rec([], [False]*len(nums))
    return res
```

Dry run `permute([1,2,3])` (top branch):

```
rec([], [F F F])
  -> i=0 use 1: rec([1], [T F F])
       -> i=1 use 2: rec([1,2], [T T F])
            -> i=2 use 3: rec([1,2,3], [T T T]) -> add [1,2,3]
            pop 3, pop 2, ... unwind
```

---

## 9. When NOT to use

- **Iteration is obvious**: avoid recursion for `for i in range(n)` style problems; the cost is a stack frame per step.
- **Need > 1000 frames in production**: Python's recursion limit will trip unless you raise it; safer to iterate.
- **Performance critical** and stack depth unknown: every function call adds overhead (~µs each).
- **Tail-recursion hope**: Python does **NOT** optimize tail calls, so converting ML-style recursion is meaningless.
- **Recursion + state mutation** may be unintuitive — e.g. modifying globals from a recursive function is bug-prone.
- **Dynamic programming with iterative bottom-up**: bottom-up moves O(n) space into a flat list with no call overhead.

---

## 10. Common Mistakes

1. **Missing or wrong base case** → infinite recursion → `RecursionError`.
2. **No progress toward base case**: `def f(n): return f(n)` loops forever.
3. **Naive Fibonacci** — O(2ⁿ). Add `@lru_cache` or use iterative.
4. **Mutable default arguments**: `def f(n, path=[]):` shares path across recursive calls. Pass `path=None` and initialize inside.
5. **Mutating a list that's also referenced by caller**: if you `path.append` without `path[:]` snapshot, your `res.append(path)` will see the mutated version.
6. **Forgetting to undo state on backtrack**: especially in permutations/combinations.
7. **Iterating `for child in children:` recursively and forgetting to pop**.
8. **Confusing base case with edge case**: e.g. `factorial(0)` should return `1`, not raise.
9. **Closing over loop variable in nested recursion**: classic Python late-binding; pass as argument.
10. **Raising recursion limit too aggressively**: setting 10⁶ frames will crash the interpreter before reaching the limit.

---

## 11. Memory Tricks

- 🔑 **Every recursive function** must answer: *"How do I shrink the input?"* and *"When do I stop?"*.
- 🔑 **B.R.B.** = **B**ase case **R**ecursive case **B**uild up the answer.
- 🔑 "Recursion is just a loop with a stack frame per iteration" — when in doubt, ask if a `while` would do.
- 🔑 Memoization = **M**emo = **M**emory + recurrence. Cache the recurrences.
- 🔑 `@lru_cache` is the lazy top-down DP tool of choice.
- 🔑 Tail recursion isn't optimized in Python — don't write Scala-style tail recursions expecting speedup.
- 🔑 Think of the call stack as **a stack of partially done tasks**, resuming when its child returns.

---

## 12. Interview Shortcuts

- Default memoization pattern: `@lru_cache(None)` on a clean recurrence.
- Convert recursion → iteration by maintaining an explicit stack: `stack = [root]; while stack: node = stack.pop()`.
- Backtracking template: pick → recurse → undo (always in a triplet).
- For tree problems, try recursion first; for graph problems with potential cycles, use an iterative BFS/DFS with a `visited` set.
- "Recurrence relation ==  recursion function" — write the recurrence, the rest is bookkeeping.
- Depth too large? Convert to iterative with `while` and a manual stack — no recursion limit issues.
- For top-down DP, change the Я of `return rec(state)` with `cache[state] = rec(state)`.

---

## 13. Cheat Sheet Table

| Pattern               | Code snippet                              | Complexity                                  |
|----------------------|-------------------------------------------|---------------------------------------------|
| Linear recursion     | `return n * f(n-1)`                       | O(n) time, O(n) stack                       |
| Binary recursion     | `return f(L) + f(R)` (tree)               | O(nodes)                                    |
| Divide-and-conquer   | `return merge(f(L), f(R))`                | O(n log n) mergesort pattern                |
| Backtracking         | pick -> recurse -> undo                   | O(2^n) subsets; O(n!) perms                 |
| Memoized recursion  | `@lru_cache(None)` on a recurrence       | O(states * branches)                        |
| Tail recursion       | `return f(n, acc)`                        | (NOT optimized in Python)                   |
| Recursion → iteration| manual stack: `stack = [root]`           | same complexity, more lines, no limit issue |

---

## 14. Time Complexity Table

| Recursion type               | Time                | Space (call stack)   | Notes                                   |
|------------------------------|---------------------|----------------------|-----------------------------------------|
| Linear (factorial)           | O(n)                | O(n)                 | Python limit is 1000                    |
| Binary balanced tree (depth) | O(log n)            | O(log n)             | Safe for n up to ~10^300                |
| Fibonacci naive              | O(2ⁿ)               | O(n)                 | Exponential — never acceptable          |
| Fibonacci + memoization      | O(n)                | O(n)                 | Top-down DP                              |
| Merge sort                   | O(n log n)          | O(log n) stack       | Build output O(n) extra                  |
| Quick sort (avg)             | O(n log n)          | O(log n)             | Worst O(n²) time, O(n) stack             |
| Binary search recursive      | O(log n)            | O(log n)             | Iterative version makes it O(1) space   |
| Subsets                      | O(2ⁿ * n)           | O(n)                 | Output size dominates                    |
| Permutations                 | O(n! * n)           | O(n)                 | Backtracking                              |
| Tower of Hanoi               | O(2ⁿ)               | O(n)                 | Classic recursion example                |

---

## 15. Visual Diagram (ASCII)

### Call stack for `factorial(4)`

```
       push (grows down)          pop (grows up)
                
   ┌──────────────────┐
   │  fact(1) ret=1   │  base case triggers unwind
   ├──────────────────┤
   │  fact(2)  n=2    │  ↓ 2 * 1 = 2
   ├──────────────────┤
   │  fact(3)  n=3    │  ↓ 3 * 2 = 6
   ├──────────────────┤
   │  fact(4)  n=4    │  ↓ 4 * 6 = 24
   ├──────────────────┤
   │  main            │
   └──────────────────┘
```

### Recursion tree (Fibonacci 5) — show overlapping subproblems

```
                f(5)
              /       \
           f(4)        f(3)         <- f(3) computed twice
          /    \        /  \
        f(3)  f(2)   f(2) f(1)
       /  \
     f(2) f(1)
     / \
   f(1) f(0)
```

### Flowchart — recursive function design

```
        start rec(input)
              |
       is input base case?
        /            \
      yes            no
       |              |
    return trivial   shrink input -> rec(smaller)
                    combine result -> return
```

### Backtracking template diagram

```
   start path = []
   choose element  -> push onto path
   recurse         -> rec(next path)
   undo            -> pop from path
   try other choices
```

---

## 16. Beginner Notes — Remember block

```
Remember:
- A recursive function needs a base case + recursive case.
- Each call adds a stack frame; Python's default limit is 1000.
- Tail recursion is NOT optimized in Python — prefer iteration.
- Naive Fibonacci is O(2^n); always memoize with @lru_cache.
- Backtracking = choose -> recurse -> undo.
- Use a manual stack to convert deep recursion into iteration.
- Top-down DP = recursion + cache; bottom-up DP = loops + table.
- Recursion is best when the problem structure is recursive: trees, divide-and-conquer, combinations.
```

---

## 17. FAANG Tips

1. **Always say your base case out loud** at the start of a recursive function — interviewers look for it.
2. **Draw the recursion tree** before coding — it tells you complexity and where memoization helps.
3. **Match the recurrence to a known pattern**:
   - `f(n) = f(n-1) + f(n-2)` → Fibonacci-like → try memoization or matrix exponentiation.
   - `f(n) = min(f(n - c_i)) + 1` → coin change DP.
   - `f(i, j) = f(i+1, j-1) + cond` → palindromic substring DP.
4. **Convert recursion → iteration when depth is risky**: trees with n > 1000, etc.
5. **Watch Python's recursion limit** in merge sort on 10⁵ elements — log₂(10⁵) ≈ 17, so safe; but depth-first on a long chain of 5000 nodes will crash.
6. **Backtracking template** for subsets/permutations/N-Queens:
   ```
   def rec(i, path):
       if base: res.append(path[:])  # snapshot!
       for choice in choices(i):
           path.append(choice); rec(next, path); path.pop()
   ```
7. **Top-down DP first**: define the recurrence, add `@lru_cache(None)`, then convert to bottom-up if space is tight. Easier to write correctly.
8. **Iterative DFS using a stack**: `stack = [(root, state)]; while stack: node, state = stack.pop()` — drop-in replacement for recursion when avoiding limits.
9. **Be explicit about mutations**: pass new objects (`path + [x]`) for clarity; pass references (with undo) for performance.
10. **Recursion vs DP**: ask "are subproblems repeated?" — if yes, memoize or use a table.

---

## 18. Practice Problems

| Difficulty | Problem                                                                          | Hint                                                |
|-----------|----------------------------------------------------------------------------------|-----------------------------------------------------|
| Easy      | [509 Fibonacci Number](https://leetcode.com/problems/fibonacci-number/)         | `@lru_cache` or iterative                          |
| Easy      | [206 Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)   | Recursion unwinds, reverses links                   |
| Easy      | [21 Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/) | Pick smaller, recurse                            |
| Medium    | [77 Combinations](https://leetcode.com/problems/combinations/)                   | Backtracking template                              |
| Medium    | [46 Permutations](https://leetcode.com/problems/permutations/)                  | Backtracking with used[] array                     |
| Medium    | [22 Generate Parentheses](https://leetcode.com/problems/generate-parentheses/)  | Backtracking with `open < n` and `close < open`   |
| Medium    | [98 Validate BST](https://leetcode.com/problems/validate-binary-search-tree/)   | Recursive bounds `(lo, hi)`                        |
| Hard      | [51 N-Queens](https://leetcode.com/problems/n-queens/)                           | Backtracking with column/diagonal sets              |
| Hard      | [10 Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching/) | 2D DP recursion + memo                 |

---

**Cross-links**: [sorting.md](./sorting.md) for merge/quick sort recursion · [binary_search.md](./binary_search.md) for recursive binary search · [searching.md](./searching.md) for when not to recurse.