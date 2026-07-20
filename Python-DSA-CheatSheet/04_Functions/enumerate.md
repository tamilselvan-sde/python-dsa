# enumerate() in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
> Section: 04 — Functions
> 🔗 Related: [functions.md](./functions.md) · [lambda.md](./lambda.md) · [map.md](./map.md) · [filter.md](./filter.md) · [zip.md](./zip.md) · Back to [README](../README.md)

---

## 1. What is it?

`enumerate(iterable, start=0)` is a **built-in function** that pairs each item of `iterable` with a running index, yielding `(index, item)` tuples. It produces a **lazy iterator**, so it's memory-light even on huge streams.

```python
for i, name in enumerate(["Tamil", "Ana", "Bob"]):
    print(i, name)
# 0 Tamil
# 1 Ana
# 2 Bob
```

Key points:

- Returns an **iterator of 2-tuples** `(index, item)`.
- `start` parameter customizes the **first index** (default `0`).
- Lazy — produces one tuple at a time, no list built.
- Replaces the un-Pythonic `range(len(xs))` idiom when you need both index and value.

## 2. Why do we use it?

- **Read both index and value** in a single `for` loop — without `xs[i]`.
- **Map style**: build dicts `(index → item)`, count occurrences with positions, etc.
- Output `start=1` for "human-friendly" reporting numbers (1-based ranks).
- **Avoids off-by-one errors** that come from inventing your own counter.
- Works with any iterable (lists, strings, files, generators) — not just indexable ones.

## 3. When should I choose it?

| Goal | Use `enumerate`? | Alternative |
|------|--------------------|-------------|
| Loop with both index and value | ✅ Yes | manual counter |
| Loop with only value | ❌ just `for x in xs` | plain loop |
| Loop with only index | ❌ `range(len(xs))` | `range(len(xs))` |
| Parallel loop over 2+ lists | ❌ `zip` | see [zip.md](./zip.md) |
| Need value counts/positions | ✅ Yes | `dict(enumerate(...))` |
| Output a numbered list | ✅ Yes | manual counter |
| Need **start = 1** for ranks/data | ✅ Yes `start=1` | manual +1 |
| Need to know both `i` and `s[i]` in a string | ✅ Yes | `for c in s` if only char |
| You only need **count** | ❌ `len(xs)` or `Counter` | — |

## 4. Syntax

```python
enumerate(iterable, start=0)
```

- `iterable` — any iterable (list, str, file, generator, dict).
- `start` — int; index of the first item (default 0).
- Returns: an `enumerate` object — iterator yielding `(int, item)` tuples.

### Common unpacking patterns

```python
for i, x in enumerate(xs):           # default start=0
    ...

for i, x in enumerate(xs, start=1):  # 1-based ranks
    ...

list(enumerate("Hello"))             # [(0,'H'),(1,'e'),(2,'l'),(3,'l'),(4,'o')]
dict(enumerate("Hello"))             # {0:'H',1:'e',2:'l',3:'l',4:'o'}
```

## 5. Basic Example

### 5.1 With default start

```python
fruits = ["apple", "banana", "cherry"]
for i, f in enumerate(fruits):
    print(i, f)
# 0 apple
# 1 banana
# 2 cherry
```

### 5.2 With `start=1`

```python
for rank, name in enumerate(["Tamil", "Ana", "Bob"], start=1):
    print(rank, name)
# 1 Tamil
# 2 Ana
# 3 Bob
```

### 5.3 Build an indexed dict

```python
dict(enumerate(["a", "b", "c"]))
# {0: 'a', 1: 'b', 2: 'c'}
dict(enumerate("abc", start=1))
# {1: 'a', 2: 'b', 3: 'c'}
```

### 5.4 Strings — index + char

```python
for i, c in enumerate("hello"):
    if c == "l":
        print(i)
# 2
# 3
```

### 5.5 Filtered index list

```python
[ i for i, c in enumerate("hello") if c == "l" ]   # [2, 3]
```

## 6. Step-by-Step Dry Run

`enumerate(["Tamil", "Ana", "Bob"], start=1)`:

| Step | internal counter | item | tuple emitted |
|------|------------------|------|----------------|
| 1 | 1 | "Tamil" | (1, "Tamil") |
| 2 | 2 | "Ana" | (2, "Ana") |
| 3 | 3 | "Bob" | (3, "Bob") |
| 4 | StopIteration | — | end |

After the loop, you can build pairing (rank, name) for each person.

## 7. Built-in Methods / Idioms

### 7.1 Find first index matching predicate

```python
def first_match(xs, pred):
    return next((i for i, x in enumerate(xs) if pred(x)), -1)

first_match([1, 3, 5, 4], lambda x: x % 2 == 0)   # 3
```

### 7.2 Find **all** indices

```python
[i for i, x in enumerate("hello") if x == "l"]   # [2, 3]
```

### 7.3 Build dict from index

```python
dict(enumerate(["alpha", "beta"]))   # {0: 'alpha', 1: 'beta'}
```

### 7.4 Reverse mapping

```python
{v: i for i, v in enumerate(["alpha", "beta"])}   # {'alpha': 0, 'beta': 1}
```

Use this for **lookup-by-value** dictionaries.

### 7.5 `enumerate` on a dict

`enumerate(d)` yields `(i, key)` — index + key. For `(i, (k, v))` use `enumerate(d.items())`.

```python
d = {"x": 1, "y": 2}
for i, key in enumerate(d):                        # iterates keys
    print(i, key)
for i, (k, v) in enumerate(d.items()):             # iterate pairs
    print(i, k, v)
```

### 7.6 `enumerate` with `zip`

For **index along paired lists**:

```python
for i, (a, b) in enumerate(zip(xs, ys)):
    print(i, a, b)
```

### 7.7 Pair `enumerate` and a custom `start`

```python
for row_num, line in enumerate(file, start=1):
    print(f"line {row_num}: {line.rstrip()}")
```

### 7.8 Unary tuple unpacking (1-based ranks output)

```python
for i, x in enumerate(sorted(scores, reverse=True), 1):
    medal = {1: "🥇", 2: "🥈", 3: "🥉"}.get(i, "")
    print(i, medal, x)
```

### 7.9 Compare to `range(len(xs))`

```python
# ❌ un-Pythonic
for i in range(len(xs)):
    print(i, xs[i])

# ✅ Pythonic
for i, x in enumerate(xs):
    print(i, x)
```

The second avoids index lookups and reads cleaner.

### 7.10 Lazy on a file — lines counter

```python
with open("data.csv") as f:
    for line_no, row in enumerate(f, 1):
        if line_no > 100:
            break
        process(row)
```

Lazy iteration over the file — no giant string read into memory.

## 8. Interview Example

**Problem (Meta-style):** Given a list, return indices of the **two** items that sum to target (Two Sum variant w/o dict).

```python
def two_sum(nums, target):
    for i, a in enumerate(nums):
        for j, b in enumerate(nums[i+1:], start=i+1):
            if a + b == target:
                return [i, j]
```

Better: hash-based O(n):

```python
def two_sum(nums, target):
    seen = {}
    for i, n in enumerate(nums):
        if target - n in seen:
            return [seen[target - n], i]
        seen[n] = i
```

`enumerate` provides both the value (`n`) and the index (`i`) to record in the seen dict.

**Top K strings:**

```python
[ w for i, w in enumerate(sorted(words, key=lambda w: (-len(w), w))) if i < k ]
```

`enumerate` gives a positional rank for slicing.

## 9. When NOT to use

- **Loop doesn't need the index** — just `for x in xs`.
- **Only need a count** — `len(xs)` or `collections.Counter` are clearer.
- **You want **all** pair combinations** — `itertools.combinations`, not nested `enumerate`.
- **Pairing two lists** — `zip` (see [zip.md](./zip.md)).
- **Sorting by index** — `sorted(enumerate(xs), key=lambda t: t[1])` is more readable without `enumerate` (just `sorted(xs)` and `argsort` later).

## 10. Common Mistakes

1. **`range(len(xs))` instead of `enumerate`** — the classic Pythonic mistake; reviewers love flagging this.
2. **Wrong tuple order**: `enumerate` returns `(index, item)`, not `(item, index)`. Swapping in the loop var order inverts indexing.
3. **Forgetting to materialize**: `list(enumerate(xs))` when you need a real list (it's a lazy iterator otherwise).
4. **Modifying the underlying list during enumeration** — index won't reflect insertions/deletions; iterate over a copy.
5. **Reusing an `enumerate` object** — single-use. Create fresh or materialize first.
6. **Confusing `start` semantics**: `start=1` only offsets the index `i`, the underlying iterable still starts at its first item.
7. **Using `enumerate` for "every-Nth":** `range(0, len(xs), N)` is more idiomatic than `enumerate` + skip.
8. **Reassigning `i`** inside the loop — Python silently resets each iteration but readers may be confused; introduce a separate counter.

## 11. Memory Tricks

- `enumerate(xs)` → **"pair each item with a counter"** = enumerate each.
- Visualize **a pass with a chip** where you hand each learner a numbered ticket.
- `(i, x)` — index comes **first**; order matters when unpacking.
- `start=1` ⇒ "rank-me" idiom — output that reads naturally.
- "Enumerate everything one at a time" → lazy, lazy, lazy.

## 12. Interview Shortcuts

- Replace any `for i in range(len(xs))` immediately with `for i, x in enumerate(xs)` — interviewers love this signal.
- `dict(enumerate(xs))` — build {0: x₀, 1: x₁, …} in one line.
- `start=1` for **ranked outputs** ("player 1, player 2").
- For **lookup tables**, reverse-`enumerate` builds `{value: index}`:
  ```python
  lookup = { c: i for i, c in enumerate(letters) }
  ```
- For "find first index" pairs naturally: `next((i for i, x in enumerate(xs) if pred(x)), -1)`.
- For "groups with positions": `dict(enumerate(str.split()))`.

## 13. Cheat Sheet Table

| Form | Yields | Use case |
|------|--------|----------|
| `enumerate(xs)` | `(i, x)` | both index + value |
| `enumerate(xs, 1)` | `(i, x)` with i starting at 1 | ranks, line numbers |
| `list(enumerate(xs))` | `[(0,x),…]` | materialize |
| `dict(enumerate(xs))` | `{0: x, …}` | index → value map |
| `{v: i for i, v in enumerate(xs)}` | `{value: index}` | reverse lookup |
| `next((i for i, x in enumerate(xs) if p(x)), -1)` | first index | find-first helper |
| `enumerate(string)` | `(i, char)` | character position |
| `enumerate(file, 1)` | `(line_no, line)` | line numbering |

## 14. Time Complexity Table

| Operation | Cost |
|-----------|------|
| `enumerate(...)` creation | O(1) — lazy |
| `next(enumerate_obj)` | O(1) — counter + next of iterable |
| `list(enumerate(xs))` | O(n) |
| `dict(enumerate(xs))` | O(n) — n tuples → dict |
| memory (lazy) | O(1) |
| materialized memory | O(n) |

## 15. Visual Diagram (ASCII)

### enumerate pipeline

```
   list:      ["Tamil",  "Ana",  "Bob"]
                │       │       │
                ▼       ▼       ▼
   enumerate (start=1):
                ┌──────┬──────┬──────┐
                │(1,..)│(2,..)│(3,..)│   ← lazy 2-tuples
                └──────┴──────┴──────┘
                  ▼      ▼      ▼
   For loop:
       1 "Tamil"
       2 "Ana"
       3 "Bob"

   Compare to bad pattern:

   for i in range(len(list)):
       i, list[i]        ← ugly, calls __getitem__ each time, slow on dicts
```

### Two Sum dict-build with enumerate

```
   nums:        [2, 7, 11, 15]      target: 9
                  │  │   │    │
                  ▼  ▼   ▼    ▼
   enumerate:
                  (0,2)(1,7)(2,11)(3,15)
                  │  │
                  ▼  ▼
   seen:        {}→  {2:0}→ complement(9-7=2)?? YES return [0,1]
                  Not in    found
                  seen
                  → put 7:1
                                                       (next iteration)
   Result: [0, 1]
```

## 16. Beginner Notes

> Remember:
> - `enumerate(xs)` yields `(index, value)` pairs; **index comes first**.
> - Returns a lazy iterator — wrap `list()` if you need an actual list.
> - `start=` sets the first index — useful for 1-based ranks, line numbers, etc.
> - Use `enumerate` instead of `range(len(xs))` whenever you need both `i` and `x`.
> - `dict(enumerate(xs))` builds {index → value}; swap with `{v: i for i, v in enumerate(xs)}` for reverse.
> - For **paired lists**, use `zip`, NOT `enumerate`+`range`.

## 17. FAANG Tips

- **Replace `for i in range(len(xs))`** with `enumerate` immediately — interviewers will see "you write Pythonic code."
- For **counted output** use `start=1` (ranks, line numbers, page numbers).
- Use **generator expression**: `next((i for i, x in enumerate(xs) if pred(x)), -1)` is a premier one-liner "find first index."
- **Reverse lookup dicts**: `{v: i for i, v in enumerate(values)}` are common for **string→index** maps (variable indices in interpreters, e.g. variable→register).
- For **two-sum-like** problems enumerate gives both the value and index for the seen-dict in one line.
- When processing **file lines** for compiler / parser-style code, use `enumerate(file, start=1)` to track line numbers for error messages.
- For sorted ranking — `sorted(...)` followed by `enumerate(..., start=1)` is a clean combo.

## 18. Practice Problems

**Easy**
1. [LeetCode 1 — Two Sum](https://leetcode.com/problems/two-sum/) — `enumerate` supplies the (value, index) pairs.
2. [LeetCode 26 — Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array/) — `enumerate` iterates in-place slots.
3. [LeetCode 28 — Find the Index of the First Occurrence in a String](https://leetcode.com/problems/find-the-index-of-the-first-occurrence-in-a-string/) — `enumerate(s)` sliding index.

**Medium**
4. [LeetCode 49 — Group Anagrams](https://leetcode.com/problems/group-anagrams/) — enumerate over sorted strings.
5. [LeetCode 152 — Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray/) — enumerate tracks last_pos / first_neg positions.
6. [LeetCode 238 — Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/) — enumerate for left/right product sweeps.

**Hard**
7. [LeetCode 84 — Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) — use enumerate with a stack-based index tracking.

---

🧭 Back: [zip.md](./zip.md) · Earlier: [reduce.md](./reduce.md) · [filter.md](./filter.md) · [map.md](./map.md) · [lambda.md](./lambda.md) · [functions.md](./functions.md).

Related loops: [../03_Control_Flow/for_loop.md](../03_Control_Flow/for_loop.md) · Lists: [../02_Data_Types/list.md](../02_Data_Types/list.md)