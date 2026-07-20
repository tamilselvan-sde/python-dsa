# while Loop — Indefinite Iteration

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 03 — Control Flow
> 🔗 Related: [if_else.md](./if_else.md) · [for_loop.md](./for_loop.md) · [break_continue.md](./break_continue.md) · Back to [README](../README.md)

## 1. What is it?

A `while` loop repeatedly executes its body **as long as** a boolean condition remains `True`.
It's used when the **number of iterations is unknown** beforehand.

```python
while condition:
    body    # must progress toward condition=False
```

Variants:
- **Sentinel pattern** — read until a special value (`"-1"`, `None`).
- **Infinite loop** — `while True:` paired with `break`.
- **`while ... else`** — `else` runs when condition becomes `False` naturally (not via `break`).
- **Multi-variable condition** — `while left < right and arr[left] < target:`.

## 2. Why do we use it?

- Iteration count is governed by a **dynamic condition**, not a fixed size.
- Useful for **simulation**, **search**, **numerical refinement** (Newton's method), **streaming**.
- Powers classic algorithms: binary search, two-pointer, BFS level loop, Sliding Window.

## 3. When should I choose it? (decision table)

| Need | Use | Avoid |
|------|-----|-------|
| Known count of iterations | `for i in range(n):` | `while i < n` |
| Iterate until condition met | `while cond:` | `for` + break |
| Wait for stream signal | `while True:` + `break` | for-loop |
| Binary search | `while lo <= hi:` | recursive call |
| Two-pointer | `while left < right:` | for |
| Numerical refinement | `while abs(x - prev) > eps:` | fixed ranges |
| Sentinels | `while x != SENTINEL:` | for |

## 4. Syntax

```python
# Basic
while cond:
    body

# Infinite + exit
while True:
    if cond_met: break

# Sentinel
SENTINEL = -1
val = int(input())
while val != SENTINEL:
    process(val)
    val = int(input())

# while-else (rare)
i = 0
while i < 10:
    if i == 5: break
    i += 1
else:
    print("exhausted without break")   # runs only if condition became False normally

# Multi-variable
while left < right and arr[left] <= target:
    left += 1
```

## 5. Basic Example

```python
# Count digits in an integer
n = 12345
count = 0
while n > 0:
    n //= 10
    count += 1
print(count)        # 5

# Sentinel input loop
SENTINEL = 0
total = 0
value = int(input())
while value != SENTINEL:
    total += value
    value = int(input())
print("sum:", total)

# while-else: search
def find_first_odd_pair(arr):
    i = 0
    while i < len(arr) - 1:
        if arr[i] % 2 == 1 and arr[i+1] % 2 == 1:
            return i
        i += 1
    else:
        return -1
```

## 6. Step-by-Step Dry Run

Code:
```python
n = 7
power = 1
while power * 2 <= n:
    power *= 2
print(power)
```

| Iter | `power` | `power*2 <= n` | Action |
|------|---------|---------------|--------|
| — | 1 | 2 ≤ 7 → True | power ← 2 |
| 1 | 2 | 4 ≤ 7 → True | power ← 4 |
| 2 | 4 | 8 ≤ 7 → False | stop |

Output: `4`

## 7. Built-in Methods / Idioms (in while-loops)

Python `while` itself has no methods — but a few idioms pair with it constantly.

### Sentinel value loop
- **Purpose:** read until a "stop" token.
- **Syntax:** `while x != SENTINEL: ... ; x = next()`.
- **Example:** Interactive input: run until user types `q`.
- **Complexity:** depends on stream length — often unknown.
- **Interview use:** reading tokens from a list/string cursor.
- **Mistakes:** forgetting to advance `x` inside the body → infinite loop.
- **Shortcut:** Sentinel = "magic stop value"; pick one unlikely to occur in real data.

### `while True:` + `break`
- **Purpose:** exit from middle of body when multiple exit conditions.
- **Syntax:** `while True: ... if exit_condition: break`.
- **Example:** BFS levels: `while queue: ... continue outer`.
- **Complexity:** O(iterations × body cost).
- **Interview use:** allows clean exits when condition not at the top of the loop.
- **Mistakes:** forgetting the `break` → infinite loop (CPU pinned at 100%).
- **Shortcut:** `while True:` is idiomatic in Python when the loop has multiple exit points.

### Multi-pointer while
- **Purpose:** two-pointer / three-pointer problems.
- **Syntax:** `while left <= right:` or `while fast and fast.next:`.
- **Example:** Sliding Window, binary search, partition.
- **Complexity:** O(n) typical.
- **Interview use:** Daily. Linked-list cycle detection (fast/slow), quicksort, etc.
- **Mistakes:** off-by-one at boundaries; forgetting to advance a pointer inside body.
- **Shortcut:** trace loop invariants on paper before coding.

### `while ... else:`
- **Purpose:** runs `else` only when condition naturally turns `False` (not via `break`).
- **Syntax:** `while cond: ... break else: ...`.
- **Example:** primality test (see [break_continue.md](./break_continue.md)).
- **Complexity:** O(loop count).
- **Interview use:** replaces `found = False` flag.
- **Mistakes:** confusing with `if-else` semantics.
- **Shortcut:** Read as "while-without-break".

### Two-pointer / fast-slow
- **Purpose:** O(n) single sweep with two moving indices.
- **Syntax:** `slow = fast = head; while fast and fast.next: slow=slow.next; fast=fast.next.next`.
- **Example:** Find middle of linked list.
- **Complexity:** O(n) time, O(1) space.
- **Interview use:** cycle detection (Floyd's Tortoise & Hare), remove duplicates in sorted array.
- **Mistakes:** not advancing `fast` correctly → infinite loop; off-by-one when reporting `slow`.
- **Shortcut:** Use fast/slow for "middle" and slow/fast for "k-th from end".

## 8. Interview Example

**LeetCode 704. Binary Search**

```python
def search(nums, target):
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1
```

Classic `while lo <= hi:` pattern — paired with `if/elif/else` (see [if_else.md](./if_else.md))
and somewhere `mid` updates with `lo = mid + 1` and `hi = mid - 1` to avoid infinite loop.

See [../07_Algorithms/binary_search.md](../07_Algorithms/binary_search.md) for full algorithm write-up.

## 9. When NOT to use

- **Fixed iterations known ahead** → use `for` (see [for_loop.md](./for_loop.md)).
- **Iterating over a sequence's items** → `for x in seq:`.
- **Recursion suits the problem better** (divide-and-conquer on naturally nested data).
- **You can't prove termination** — every `while` body must reduce the "distance to False".
- **Performance critical inner loop** — `for` over numpy/optimized / itertools is often faster.

## 10. Common Mistakes

| # | Mistake | Fix |
|---|---------|-----|
| 1 | Forgetting to advance state in body → infinite loop | ensure body mutates condition-relevant variable(s) |
| 2 | `while x = get_next():` (assignment in cond — SyntaxError) | use walrus `while (x := get_next()):` (3.8+) |
| 3 | Off-by-one termination (`<` vs `<=`) | choose based on inclusive vs exclusive bounds |
| 4 | `while True:` with no `break` reachable | always have an exit |
| 5 | Forgetting `break` after sentinel received | check sentinel before body and break |
| 6 | Mutating iterable while streaming | consume via iterator pattern using `next()` |
| 7 | `while i < len: i+=1` feeling like for | use `for i in range(n):` instead |
| 8 | `while-else` confusion | read docs; only fires when condition becomes False naturally |
| 9 | Comparing floats with `==` in exit condition | use `abs(a - b) > eps` |
| 10 | Side-effect induction (modify state outside loop) | keep mutation inside body |

## 11. Memory Tricks

- **"WHILE = While Honest Invariance Loops Elsewhere"** — every body must restore/move invariants.
- **Sentinel** sounds like a guard watching for a stop sign.
- **`while True:` is just a "until"** — bail with `break` when condition met.
- **`while-else`** = "while-without-break".
- **`<` vs `<=`**: `<=` when boundary matters (binary search with single midpoint inclusive).
- **Walrus `:=` is the Pythontic sentinel reader**: `while (line := f.readline()):`.

## 12. Interview Shortcuts

- **Binary search**: `while lo <= hi:` (inclusive bounds) and update `lo = mid+1` / `hi = mid-1`.
- **Sliding window**: `while right < len:` then shrink inner `while` with cost check.
- **BFS**: `while queue:` (or `while q: cur = q.popleft()` — see [../02_Data_Types/list.md](../02_Data_Types/list.md)).
- **Floyd's cycle**: `while fast and fast.next` — known by heart.
- **Walrus operator** (`:=`, 3.8+): best friend for sentinel reads and accumulator one-liners.
- **Tracing loop invariants on paper** before coding helps interview nerves far more than typing speed.

## 13. Cheat Sheet Table

| Pattern | Use When | Risk |
|---------|----------|------|
| `while cond:` | single condition | infinite loop if no progress |
| `while True:` + break | multiple exit points | no break = infinity |
| `while x != SENT:` | sentinel-stream | advance `x` |
| `while lo <= hi:` | binary search | mid update off-by-one |
| `while fast and fast.next:` | linked list traversal | EOF / cycle care |
| `while (x := next()):` | walrus read pattern | needs Python 3.8+ |
| `while cond: ... else:` | "lopped naturally" hook | rare — often misunderstood |
| Two-pointer | shrink left/right or slow/fast | mark invariants on paper |

## 14. Time Complexity Table

| Pattern | Time | Space | Example |
|---------|------|-------|---------|
| Sentinel input loop | O(N) — based on stream length N | O(1) | input streaming |
| Binary search `while lo<=hi` | O(log n) | O(1) | sorted array search |
| Two-pointer (single sweep) | O(n) | O(1) | reverse, partition |
| Sliding window | O(n) | O(k) for ring buffer | max in subarrays |
| Floyd's cycle | O(n) | O(1) | linked-list cycle |
| Newton's method (numerical) | O(log (1/eps)) iterations to fixed point | O(1) | sqrt, root finds |
| Nested `while` outer+inner | O(n × m) worst | O(1) | matrix scan |

## 15. Visual Diagram (ASCII)

```
        ┌────────────────────────────────┐
        │     evaluate condition         │
        │  (initialized before, mutated  │
        │   inside body)                  │
        └───────────────┬────────────────┘
                        │
                  ┌─────▼─────┐
                  │ cond True │
                  └───┬───┬───┘
                    yes  no
                     │   │
        ┌────────────▼┐ ┌▼─────────────┐
        │  run body   │ │  exit:       │
        │  (advance   │ │  fall to:    │
        │   state)    │ │   - end      │
        └──────┬──────┘ │   - else     │
               │        │     (only    │
               │        │      natural)│
               │        └──────────────┘
               │
      ┌─break? │
      │ yes    │ no ───── loop back to evaluate cond
      │        ▼
      ▼   ┌──────────┐
   ┌─────────┐  ▲
   │ break   │  └─ continue? (skip rest of body, re-evaluate cond)
   │ exits   │
   │ loop    │
   └─────────┘
```

## 16. Beginner Notes

> Remember:
> - A `while` body **must mutate** a variable used in the condition — else infinite loop.
> - `while True:` requires a reachable `break` — never assume input will arrive.
> - **`while-else`** is rare but powerful: it's like a `for-else` "no-break hook".
> - The **walrus operator `:=`** is great for combining read + test in one line (3.8+).
> - Use **`while`** when count is unknown; **`for`** when count is known (see [for_loop.md](./for_loop.md)).

## 17. FAANG Tips

- **Binary search**: `while lo <= hi:` with `mid = (lo + hi) // 2` may overflow in other languages, but Python ints are arbitrary precision — still, prefer `mid = lo + (hi - lo) // 2` for cross-language muscle memory.
- **Two-pointer invariants**: write them as a comment; explain to interviewer before coding.
- **Sliding window with two pointers**: usually `while right < n:` outer, `while cost_violation: ...; left += 1` inner.
- **BFS**: `while queue:` dequeues FIFO — use `collections.deque` for O(1) `popleft`, not list pop(0).
- **Floyd's slow/fast loop**: `while fast and fast.next` is THE phrase, memoize it.
- **Walrus `:=` reads elegantly**: `while (line := f.readline()): process(line)`.
- Always have a **terminating proof** ready in your mind — say it aloud during interviews.

## 18. Practice Problems

| Difficulty | Problem | Notes |
|-----------|---------|-------|
| Easy | [LeetCode 704. Binary Search](https://leetcode.com/problems/binary-search/) | classic while pattern |
| Easy | [LeetCode 283. Move Zeroes](https://leetcode.com/problems/move-zeroes/) | two-pointer while |
| Medium | [LeetCode 3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/) | sliding window with while loops |
| Medium | [LeetCode 75. Sort Colors](https://leetcode.com/problems/sort-colors/) | three-pointer while |
| Medium | [LeetCode 328. Odd Even Linked List](https://leetcode.com/problems/odd-even-linked-list/) | pointers + while |
| Medium | [LeetCode 141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/) | slow/fast while |
| Hard | [LeetCode 4. Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/) | binary-search-style while |
| Hard | [LeetCode 432. All O`one Data Structure](https://leetcode.com/problems/all-oone-data-structure/) | frequent use of while invariants |
| Hard | [LeetCode 658. Find K Closest Elements](https://leetcode.com/problems/find-k-closest-elements/) | binary search + expand via while |

---
Next up: [break_continue.md](./break_continue.md) · Related: [if_else.md](./if_else.md) · [for_loop.md](./for_loop.md) · [../07_Algorithms/binary_search.md](../07_Algorithms/binary_search.md)