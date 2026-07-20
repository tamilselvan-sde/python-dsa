# for Loop — Definite Iteration

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 03 — Control Flow
> 🔗 Related: [if_else.md](./if_else.md) · [while_loop.md](./while_loop.md) · [break_continue.md](./break_continue.md) · Back to [README](../README.md)

## 1. What is it?

A `for` loop iterates over any **iterable** — sequence, set, dict, string, file, generator, etc. —
running the same block once per element without you managing an index.

```python
for item in iterable:
    body
```

Python provides several idioms layered on top of `for`:
- `range(start, stop, step)`         — count loops
- `enumerate(iterable, start=0)`     — index + value
- `zip(*iters)`                      — paired iteration
- `reversed(iterable)`               — backward iteration
- List / Set / Dict **comprehensions** — concise mapping/filtering
- `for ... else:`                    — runs `else` if no `break` exit

## 2. Why do we use it?

- **Definite iteration**: number of steps is known or bound by the iterable.
- **Cleaner** than index juggling — no off-by-one.
- **Reads like English** — "for each element in list, do X".
- Enables **comprehensions**, the most Pythonic transformation primitive.

## 3. When should I choose it? (decision table)

| Need | Use | Avoid |
|------|-----|-------|
| Iterate over each element | `for x in seq:` | manual index while loop |
| Need index alongside value | `for i, x in enumerate(seq):` | `for i in range(len(seq))` when index used too |
| Iterate pairs from 2 lists | `for a, b in zip(l1, l2):` | manual index |
| Iterate backward | `for x in reversed(seq):` | `for i in range(len(seq)-1, -1, -1)` |
| Count loop N times | `for _ in range(N):` | `while i < N: i += 1` |
| Transform items → new list | list comprehension | `append` in loop |
| Iterate dict | `for k, v in d.items():` | `.keys()` when value needed |
| Don't know N, exit condition external | `while` (see [while_loop.md](./while_loop.md)) | — |

## 4. Syntax

```python
# Basic
for item in iterable:
    body

# range with bounds
for i in range(start, stop, step): ...
for i in range(n): ...        # 0..n-1
for i in range(1, n+1): ...   # 1..n

# enumerate (index + value)
for i, v in enumerate(seq, start=0): ...

# zip (parallel)
for a, b in zip(l1, l2): ...

# reversed
for x in reversed(seq): ...

# iterating dict
for k, v in d.items(): ...
for k in d: ...               # same as d.keys()
for v in d.values(): ...

# string
for ch in "hello": ...

# nested
for row in matrix:
    for cell in row: ...

# list comprehension
out = [f(x) for x in seq if cond]

# for ... else
for x in seq:
    if target(x):
        break
else:
    print("not found")         # runs only when loop ends WITHOUT break
```

## 5. Basic Example

```python
nums = [10, 20, 30, 40]

# enumerate
for i, x in enumerate(nums):
    print(i, x)                # 0 10 / 1 20 / 2 30 / 3 40

# range with step
for i in range(0, 10, 2):      # 0 2 4 6 8
    print(i)

# list comprehension (a for inside)
squares = [x*x for x in nums if x >= 20]
# [400, 900, 1600]

# dict iteration
ages = {"Ann": 25, "Bob": 30}
for name, age in ages.items():
    print(name, age)

# for ... else
for n in [2, 4, 6, 8]:
    if n % 2 == 1:
        print("odd found")
        break
else:
    print("all even")          # printed — never broke
```

## 6. Step-by-Step Dry Run

Code:
```python
for i, v in enumerate([10, 20, 30]):
    if v > 15:
        print(i, v)
```

| Iter | `i` | `v` | `v > 15` | Output |
|------|-----|-----|---------|--------|
| 1 | 0 | 10 | False | — |
| 2 | 1 | 20 | True  | `1 20` |
| 3 | 2 | 30 | True  | `2 30` |

`enumerate` lazily zips `range(len)` with the iterable — it does **not** call `len`
if the iterable is a generator, so it works on infinite-ish streams (with manual break).

## 7. Built-in Methods / Idioms (in for-loops)

### `range(start, stop, step)`
- **Purpose:** generate an arithmetic sequence, lazy.
- **Syntax:** `range(stop)`, `range(start, stop)`, `range(start, stop, step)`.
- **Example:** `range(2, 10, 3)` → 2, 5, 8.
- **Complexity:** O(1) creation, O(n) iteration; constant memory.
- **Interview use:** index-driven iteration, reverse with `range(n-1, -1, -1)`.
- **Mistakes:** `range(10)` stops at 9; `range(1, 10, -1)` is empty (must swap bounds).
- **Shortcut:** `range(n)` is the simplest loop counter; use `range(n-1, -1, -1)` to go backward.

### `enumerate(iterable, start=0)`
- **Purpose:** produces (index, value) tuples while iterating.
- **Syntax:** `for i, v in enumerate(seq, start=1):`.
- **Example:** `for i, ch in enumerate(word): if ch == 'a': positions.append(i)`.
- **Complexity:** O(n).
- **Interview use:** replaces ugly `for i in range(len(seq))`.
- **Mistakes:** unrolling as `i = 0; for v in seq: ...; i += 1`.
- **Shortcut:** `start=` param is great for human-readable ranks ("Player 1, Player 2…").

### `zip(*iterables, strict=False)`
- **Purpose:** iterate several iterables in lock-step.
- **Syntax:** `for a, b, c in zip(l1, l2, l3):`.
- **Example:** `for matrix_row, kernel_row in zip(A, K):`.
- **Complexity:** O(min length).
- **Interview use:** pairing two lists; matrix transpose `list(zip(*matrix))`.
- **Mistakes:** silent truncation if lengths differ — use `zip(a, b, strict=True)` (3.10+) to raise on mismatch.
- **Shortcut:** transpose = `list(map(list, zip(*matrix)))`.

### `reversed(iterable)`
- **Purpose:** iterate backward without slicing-copy.
- **Syntax:** `for x in reversed(seq):`.
- **Example:** `for ch in reversed("abc"): print(ch)`.
- **Complexity:** O(1) creation; O(n) iteration.
- **Interview use:** palindrome checks; processing latest-first.
- **Mistakes:** `reversed([1,2,3])` returns an iterator — materialize with `list()` or iterate directly.
- **Shortcut:** prefer `reversed` over `seq[::-1]` when only iterating, to avoid a copy.

### `dict.items() / .keys() / .values()`
- **Purpose:** iterate dicts as views keyed to dict shape.
- **Syntax:** `for k, v in d.items():`.
- **Example:** `for node, neighbors in graph.items(): ...`.
- **Complexity:** O(n) iteration.
- **Interview use:** graph traversal, counter visits.
- **Mistakes:** modifying dict size during iteration → `RuntimeError`; build a list first.
- **Shortcut:** `dict.get(k, default)` inside a `for` replaces "guard" `if k in d`.

### List / Dict / Set comprehension
- **Purpose:** one-line map/filter loops.
- **Syntax:** `[f(x) for x in seq if p(x)]`, `{k: v for ...}`, `{x for ...}`.
- **Example:** `evens = [x for x in nums if x % 2 == 0]`.
- **Complexity:** O(n) time + O(out length) space.
- **Interview use:** replaces verbose `append` loops; chain comparisons over sequences.
- **Mistakes:** nesting 3+ comprehensions becomes unreadable.
- **Shortcut:** "filter" → keep the `if` at the **end**; "if-else mapping" → put `if` at the **start**:
  `[x if x%2==0 else -x for x in nums]`.

### `for ... else:`
- **Purpose:** runs the `else` block **only when no `break` exit**.
- **Syntax:** `for x in seq: ... break else: "exhausted"`.
- **Example:** prime test (see [break_continue.md](./break_continue.md)).
- **Complexity:** O(n).
- **Interview use:** "not found" sentinel without a `found` flag.
- **Mistakes:** confusing `for-else` with `if-else`.
- **Shortcut:** read it as "for-with-no-break".

## 8. Interview Example

**LeetCode 1. Two Sum (variant with enumerate)**

```python
def twoSum(nums, target):
    seen = {}
    for i, x in enumerate(nums):
        partner = target - x
        if partner in seen:
            return [seen[partner], i]
        seen[x] = i
    return []
```

Skill of note: `enumerate` + dict guard via `if partner in seen`. A classic O(n) time, O(n) space pattern.

## 9. When NOT to use

- Unknown number of iterations, condition external  → `while` (see [while_loop.md](./while_loop.md))
- Variable collection mutating length during iteration → snap a copy first: `for x in list(items):`.
- Deep nested triple-comprehension → use a plain nested `for`, much more readable.
- Performance hot inner loop over container → consider numpy / optimized libs.

## 10. Common Mistakes

| # | Mistake | Fix |
|---|---------|-----|
| 1 | Mutating a list while iterating it | iterate a copy `for x in list(seq):` |
| 2 | Using `for i in range(len(seq))` and re-fetching `seq[i]` | `for i, v in enumerate(seq)` |
| 3 | Wrong zip-truncation (lists of unequal length) silent | `zip(a, b, strict=True)` (3.10+) |
| 4 | Comprehension as a loop side-effects list | use a plain `for` for side-effects |
| 5 | `range(10, 0, -1)` start/stop mismatch | remember `stop` is **exclusive** |
| 6 | Iterating dict and deleting keys live | collect keys first, then mutate |
| 7 | Mutating copied string → ignored | strings immutable; build new ones |
| 8 | Overusing nested comprehensions | flatten to plain `for`s |
| 9 | Forgetting `for-else` only fires w/o `break` | always re-read whether `break` exists |
| 10 | Modifying matrix rows `for row in matrix` mutates row in place | sometimes intended, often surprising — be deliberate |

## 11. Memory Tricks

- **`range` stops at `stop - 1`** — "Python stops short of the stop".
- **`enumerate` = e-numerate = "give me the number"** — index alongside.
- **`zip` zips** — closes zippers together; stops when shorter ends.
- **`reversed` for backward when you want lazy; `[::-1]` when you want a copy**.
- **Dict iteration = `.items()` for both, `.keys()` for keys-only (rarely needed; default already keys), `.values()` for values-only**.
- **`for-else` runs only when the loop completes naturally**.
- **Comprehension order**: `expr` (map) → `for` (iterate) → `if` (filter chain as many as needed).

## 12. Interview Shortcuts

- Enumerate is **free** vs `range(len)` — use it.
- `zip(*matrix)` — instantly transpose a grid (very common in DP / matrix problems).
- `list(reversed(seq))` cheaper than `seq[::-1]` when only iterating once.
- Comprehension with `if/else` up front: `[x if x%2==0 else 0 for x in nums]`.
- `for-else` cleanly detects "loop exhausted without finding" without a flag.
- For graph traversal, `for neighbor in graph[node]:` reads easier than `range(len(graph[node]))`.

## 13. Cheat Sheet Table

| Construct | Purpose | Returns | Stops When |
|-----------|---------|---------|------------|
| `for x in iter:` | iterate items | item | iterable exhausted |
| `range(s, st, step)` | count loop | int | reaches `st` |
| `enumerate(iter)` | index + item | (i, x) | iterable exhausted |
| `zip(*iters, strict=)` | parallel iterate | tuple | when shortest ends (`strict=True` → raise) |
| `reversed(iter)` | backward iterate | item | iterable exhausted |
| `dict.items()` | k, v pairs | (k, v) | dict exhausted |
| `dict.keys()` | keys only | k | dict exhausted |
| `dict.values()` | values only | v | dict exhausted |
| `[expr for x in .. if cond]` | map/filter into list | list | iterable exhausted |
| `for ... else:` | post-loop hook | nothing | `else` runs only if no `break` |
| nested `for` | multi-D iteration | items | inner exhausted, outer continues |

## 14. Time Complexity Table

| Operation | Time | Space |
|-----------|------|-------|
| `for x in seq` | O(n) | O(1) extra |
| `range(...)` | O(1) creation, O(n) iterate | O(1) |
| `enumerate(seq)` | O(n) | O(1) extra |
| `zip(*iters)` | O(min len) | O(1) extra |
| `reversed(seq)` | O(n) iterate | O(1) |
| `dict.items()` | O(n) | O(1) |
| List comprehension | O(n) | O(out len) |
| Nested `for`s over m×n matrix | O(m·n) | O(1) |
| `for-else` overhead | O(1) extra | O(1) |

## 15. Visual Diagram (ASCII)

```
            ┌──────────────────────────────────────────┐
            │ for item in iterable: (get next item)    │
            └────────────────────┬─────────────────────┘
                                 │
                          ┌──────▼──────┐
                          │ items left? │
                          └──┬───┬──────┘
                            yes  no
                             │   │
                  ┌──────────▼┐ ┌▼─────────┐
                  │ run body  │ │ run else  │ ← only if NO break
                  │ (item)    │ │ block     │
                  └──────┬────┘ └────┬──────┘
                         │           │
                  break? │           │
              ┌─yes──────┴────no─────┘
              │                         │
              ▼                         │
        ┌──────────┐                    │
        │ break    │◀──────── loop next │
        │ exits    │     iteration
        └──────────┘
              │
              ▼
         ┌──────────┐
         │ continue │ (not shown above — skips body, jumps to "next item")
         └──────────┘
```

## 16. Beginner Notes

> Remember:
> - A `for` loop visits each element exactly once in iteration order.
> - Strings are iterables — `for ch in "abc":`.
> - Use **`enumerate`** instead of `range(len(seq))` when you need both the index and the value.
> - For **dicts**, default `for k in d:` iterates **keys** — use `.items()` for both.
> - **`for-else`** is rare but powerful: think "no-break hook".
> - **Comprehensions** are just compact `for` loops — but only use them when mapping/filtering.

## 17. FAANG Tips

- Use **`enumerate` ALWAYS** when index + value needed — `range(len(...))` is a tell for juniors.
- **`zip` transpose** = `list(zip(*matrix))` is a common trick in matrix problems.
- **Comprehensions with chained `if`** can replace filter chains: `[x for x in xs if x > 0 if x < 10]`.
- For ordered traversal problems, prefer **`range`** with explicit bounds — easier to reason about indices.
- **`for-else`** is your "no break" hook; pair with `else` clause on the for to handle "not found".
- Use **dict iteration** over index iteration for graph problems — much cleaner.
- For huge streams, `zip`, `enumerate`, `map` are all **lazy** iterators — no buffer cost up front.

## 18. Practice Problems

| Difficulty | Problem | Notes |
|-----------|---------|-------|
| Easy | [LeetCode 1. Two Sum](https://leetcode.com/problems/two-sum/) | enumerate + dict guard |
| Easy | [LeetCode 1920. Build Array from Permutation](https://leetcode.com/problems/build-array-from-permutation/) | list comprehension |
| Easy | [LeetCode 9. Palindrome Number](https://leetcode.com/problems/palindrome-number/) | iterate from both ends |
| Easy | [LeetCode 1929. Concatenation of Array](https://leetcode.com/problems/concatenation-of-array/) | comprehension |
| Medium | [LeetCode 54. Spiral Matrix](https://leetcode.com/problems/spiral-matrix/) | nested for + shrinking bounds |
| Medium | [LeetCode 345. Reverse Vowels of a String](https://leetcode.com/problems/reverse-vowels-of-a-string/) | enumerate + zip |
| Medium | [LeetCode 152. Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray/) | single-pass for-loop DP |
| Hard | [LeetCode 76. Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/) | iterate with two-pointer + for body |
| Hard | [LeetCode 84. Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) | iterate index forces over bars |

---
Next up: [while_loop.md](./while_loop.md) · [break_continue.md](./break_continue.md) · Related: [../02_Data_Types/list.md](../02_Data_Types/list.md) · [../04_Functions/enumerate.md](../04_Functions/enumerate.md) · [../07_Algorithms/binary_search.md](../07_Algorithms/binary_search.md)