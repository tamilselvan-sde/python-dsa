# zip() in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 04 — Functions
> 🔗 Related: [functions.md](./functions.md) · [lambda.md](./lambda.md) · [map.md](./map.md) · [filter.md](./filter.md) · [enumerate.md](./enumerate.md) · Back to [README](../README.md)

---

## 1. What is it?

`zip(*iterables)` is a **built-in higher-order function** that aggregates one or more iterables element by element, producing a **lazy iterator of tuples**. The `i`-th tuple contains the `i`-th item from each iterable. Iteration **stops at the shortest** iterable by default.

```python
list(zip([1, 2, 3], ['a', 'b', 'c']))
# [(1, 'a'), (2, 'b'), (3, 'c')]
```

Key points:

- Returns a `zip` object — call `list()`, `dict()`, `set()` to materialize.
- Stops at **shortest** iterable unless `strict=True` is passed (Python 3.10+).
- `strict=True` raises `ValueError` when iterables have unequal lengths — great for safer code.
- Called with **zero arguments**, `zip()` returns an empty iterator.
- Called with **one argument**, `zip(xs)` yields 1-tuples — odd but valid.
- **Unzip** with `zip(*pairs)`.

## 2. Why do we use it?

- **Pair up parallel lists** — names + scores, keys + values, x + y coordinates.
- **Build a dict from keys+values** — `dict(zip(keys, values))`.
- **Transpose a matrix** — `list(zip(*matrix))`.
- **Loop multiple sequences together** — `for n, c in zip(nums, chars):`.
- **Unzip** pairs back into separate lists — `xs, ys = zip(*pairs)`.
- Replaces manual index bookkeeping (`for i in range(len(a))`).

## 3. When should I choose it?

| Goal | Use `zip`? | Alternative |
|------|-------------|-------------|
| Pair 2+ iterables elementwise | ✅ Yes | `for` with index |
| Build dict from keys+values | ✅ `dict(zip(k, v))` | comprehension `{k: v for k, v in zip(...)}` |
| Loop over 2 lists in parallel | ✅ Yes | range(len) |
| Iterate with index + value | ❌ Use `enumerate` | see [enumerate.md](./enumerate.md) |
| Apply fn to many iterables | ⚠️ `map(f, *iters)` (zip equivalent) | `zip` + comp |
| Unequal-length pairing | ❌ use `itertools.zip_longest` | — |
| Strict length check | ✅ `zip(a, b, strict=True)` (3.10+) | explicit length check |
| Transpose matrix rows↔cols | ✅ `zip(*m)` | nested loop |

## 4. Syntax

```python
zip(*iterables, strict=False)            # keyword-only `strict` (3.10+)
```

- `*iterables` — zero or more iterables to fuse.
- `strict` (keyword-only, Python 3.10+) — when `True`, raises `ValueError` if iterables differ in length.
- Returns a `zip` iterator yielding tuples.

## 5. Basic Example

### 5.1 Pair two lists

```python
list(zip([1, 2, 3], ['a', 'b', 'c']))
# [(1, 'a'), (2, 'b'), (3, 'c')]
```

### 5.2 Build dict

```python
dict(zip(['a', 'b', 'c'], [1, 2, 3]))
# {'a': 1, 'b': 2, 'c': 3}
```

### 5.3 Loop in parallel

```python
names = ["Tamil", "Ana", "Bob"]
ages = [30, 25, 35]
for name, age in zip(names, ages):
    print(name, age)
```

### 5.4 unequal lengths — stops at shortest

```python
list(zip([1, 2, 3, 4], ['a', 'b']))
# [(1, 'a'), (2, 'b')]
```

### 5.5 strict mode (3.10+)

```python
list(zip([1, 2, 3], ['a', 'b'], strict=True))
# ValueError: zip() argument 2 is shorter than argument 1
```

### 5.6 Three (or more) iterables

```python
list(zip([1, 2], ['x', 'y'], [10, 20]))
# [(1, 'x', 10), (2, 'y', 20)]
```

## 6. Step-by-Step Dry Run

```python
out = list(zip([1, 2, 3], ['a', 'b', 'c']))
```

| Step | `iter1` next | `iter2` next | tuple produced |
|------|--------------|--------------|-----------------|
| 1 | 1 | 'a' | (1, 'a') |
| 2 | 2 | 'b' | (2, 'b') |
| 3 | 3 | 'c' | (3, 'c') |
| 4 | StopIteration/StopIteration | — | end |

Final: `[(1, 'a'), (2, 'b'), (3, 'c')]`.

If one list is shorter:

```python
list(zip([1, 2, 3], ['a', 'b']))
```

Stops after step 2 → `[(1, 'a'), (2, 'b')]`. The third item from the first list is silently dropped.

## 7. Built-in Methods / Idioms

### 7.1 `zip()` (no args)

```python
list(zip())   # []
```

### 7.2 `zip(xs)` (single arg)

```python
list(zip([1, 2, 3]))   # [(1,), (2,), (3,)]
```

### 7.3 Unzip with `zip(*pairs)`

```python
pairs = [(1, 'a'), (2, 'b'), (3, 'c')]
nums, chars = zip(*pairs)
nums   # (1, 2, 3)
chars  # ('a', 'b', 'c')
```

### 7.4 `dict(zip(k, v))`

```python
keys = ['id', 'name', 'age']
vals = [1, 'Tamil', 30]
dict(zip(keys, vals))
# {'id': 1, 'name': 'Tamil', 'age': 30}
```

### 7.5 Matrix transpose

```python
matrix = [[1, 2, 3],
          [4, 5, 6],
          [7, 8, 9]]
list(zip(*matrix))
# [(1, 4, 7), (2, 5, 8), (3, 6, 9)]
```

Each row becomes a column.

### 7.6 `zip` + `enumerate`

```python
for i, (a, b) in enumerate(zip(xs, ys)):
    print(i, a, b)
```

### 7.7 `itertools.zip_longest` for unequal lengths

```python
from itertools import zip_longest
list(zip_longest([1, 2, 3], ['a', 'b'], fillvalue='?'))
# [(1, 'a'), (2, 'b'), (3, '?')]
```

### 7.8 `zip` + `map`

```python
list(map(lambda ab: ab[0] + ab[1], zip([1, 2, 3], [10, 20, 30])))
# [11, 22, 33]
```

Or with `operator.add`:

```python
from operator import add
list(map(add, [1, 2, 3], [10, 20, 30]))   # [11, 22, 33]
```

### 7.9 Strict mode (3.10+) — defensive pairing

```python
fields = ["id", "name"]
values = [1, "Tamil", 30]
dict(zip(fields, values))                       # {'id': 1, 'name': 'Tamil'}  ← silent truncation
dict(zip(fields, values, strict=True))         # ValueError: 3 vs 2  ← loud, caught
```

Always prefer `strict=True` in code where an unequal length implies a bug rather than intended truncation.

### 7.10 Pairwise (windowing)

```python
xs = [1, 2, 3, 4, 5]
list(zip(xs, xs[1:]))
# [(1, 2), (2, 3), (3, 4), (4, 5)]
```

Or in Python 3.10+ use `itertools.pairwise`.

## 8. Interview Example

**Problem:** Zigzag merge two strings.

```python
def zigzag(a, b):
    out = []
    for c1, c2 in zip(a, b):
        out.append(c1)
        out.append(c2)
    out.append(a[len(b):])
    out.append(b[len(a):])
    return "".join(out)

zigzag("ace", "bdfghi")     # "abcdefghi"
```

**Transpose matrix** (1-liner, often asked):

```python
def transpose(m):
    return [list(row) for row in zip(*m)]

transpose([[1,2,3],[4,5,6]])
# [[1, 4], [2, 5], [3, 6]]
```

**Build dict from pairs**:

```python
def make_dict(keys, values):
    return dict(zip(keys, values, strict=True))
```

`strict=True` ensures you don't silently lose data when keys/values mismatch.

## 9. When NOT to use

- **Need index + value** in same loop → use `enumerate` ([enumerate.md](./enumerate.md)).
- **Need scalar transform of one iterable** → use `map` ([map.md](./map.md)) or comprehension.
- **Iterables are patently unequal in length and you want all of both** → `itertools.zip_longest`.
- **You want to fail loud on length mismatch** but you're on Python <3.10 → manually compare `len(...)` first.
- **One iterable is huge and the other small** — `zip` silently truncates; consider explicit `islice`.

## 10. Common Mistakes

1. **Forgetting to materialize** — `zip` returns an iterator; `print(zip(...))` shows `<zip object at 0x...>`. Wrap in `list()`.
2. **Assuming it iterates to the longest** — by default `zip` stops at the **shortest**. Use `zip_longest` if needed.
3. **Confuse `strict=True` availability** — only Python 3.10+; older code will raise `TypeError`.
4. **Forgetting argument unpacking for unzip** — `[zip(pairs)]` ≠ `zip(*pairs)`. Use `*`.
5. **Trying to `list()` after consuming** — `zip` object is single-use.
6. **Mis-ordering unzip** — `zip(*[(1,'a'),(2,'b')])` returns `(1, 2), ('a', 'b')`; the numbers come first.
7. **Trying to bulk-`len()` a zip object** — it's an iterator; no `len`.
8. **Not using `strict=True`** when bugs hide in unequal lengths — turn on by default for system code.

## 11. Memory Tricks

- `zip` **zips up pairs** like a jacket zipper merging two teeth rows.
- "Shortest wins" unless you ask `strict=True`.
- "Unzip = `zip(*pairs)`" — pass the pairs back through `zip` to split.
- "zip into dict": `dict(zip(keys, values))` — quick construction.
- Matrix transpose: `list(zip(*matrix))` — rows ↔ cols.

## 12. Interview Shortcuts

- `dict(zip(keys, values))` is **the canonical** dict-from-two-lists one-liner.
- `list(zip(*matrix))` — one-liner transpose; surprising interview win.
- `zip(xs, xs[1:])` — **pairwise** sliding window; common in array problems.
- `dict(zip(..., strict=True))` — explicitly mention "this would crash on length mismatch; safer than silent truncation."
- `for x, y in zip(a, b)` reads clearer than `for i in range(len(a)): a[i], b[i]`.
- Mention `itertools.zip_longest` for filling-merging sequences.

## 13. Cheat Sheet Table

| Form | Returns | Equivalent |
|------|---------|------------|
| `zip()` | empty iter | `[]` |
| `zip(xs)` | 1-tuples | `[(x,) for x in xs]` |
| `zip(a, b)` | 2-tuples | shortens to shortest |
| `zip(a, b, c)` | 3-tuples | (zip triple) |
| `zip(a, b, strict=True)` | tuples or ValueError | 3.10+ |
| `zip(*pairs)` | separate tuples | unzip |
| `dict(zip(k, v))` | dict | quick build |
| `list(zip(*matrix))` | transposed rows | matrix transpose |
| `itertools.zip_longest(...)` | tuples padded with `fillvalue` | — |

## 14. Time Complexity Table

| Operation | Cost |
|-----------|------|
| `zip()` creation | O(1) — lazy |
| `next(zip_obj)` | O(1) (one `next()` per iterable) |
| `list(zip(...))` | O(min length × number of iterables) for creating tuples |
| `zip(*m)` matrix transpose | O(rows × cols) |
| `dict(zip(k, v))` | O(min length) |
| memory: zip object | O(1) — only one tuple in flight |

## 15. Visual Diagram (ASCII)

### zip — pairing

```
   ┌─┐ ┌─┐ ┌─┐ ┌─┐
   │1│ │2│ │3│ │4│    ← list A
   └┬┘ └┬┘ └┬┘ └┬┘
    │   │   │   │
    ▼   ▼   ▼   ▼
   ┌─┐ ┌─┐ ┌─┐
   │a│ │b│ │c│       ← list B (shorter)
   └┬┘ └┬┘ └┬┘
    │   │   │
    ▼   ▼   ▼   ▼   ← zip stops after shortest exhausts
  ┌───┐┌───┐┌───┐
  │(1,││(2,││(3,│
  │ a)││ b)││ c)│   ← zipped iterator
  └───┘└───┘└───┘
   Note: 4 dropped silently.
```

### Unzip with `zip(*pairs)`

```
   pairs = [(1,'a'),(2,'b'),(3,'c')]
              │    │    │
              ▼    ▼    ▼
          zip(*pairs)         # "spread the pairs into zip"
              │
              ▼
   nums   = (1, 2, 3)
   chars  = ('a', 'b', 'c')
```

### Matrix transpose

```
   matrix  →  zip(*matrix)  →  transposed
   [[1,2,3],
    [4,5,6],
    [7,8,9]]
       │       ▼
       │   [(1,4,7),
       │    (2,5,8),
       │    (3,6,9)]
   (rows ↔ cols swap)
```

## 16. Beginner Notes

> Remember:
> - `zip(a, b)` pairs items of `a` and `b` into tuples; **stops at the shortest**.
> - Returns an iterator — call `list()` to get a real list.
> - `dict(zip(keys, values))` quickly builds a dict from two lists.
> - `zip(*pairs)` is the **unzip idiom** — splits pairs back into separate tuples.
> - `zip(*matrix)` transposes a 2D matrix (rows → cols).
> - For unequal lengths use `itertools.zip_longest`.
> - Use `strict=True` (3.10+) when length mismatch is a bug.
> - `zip(xs, xs[1:])` builds a sliding window of pairs.

## 17. FAANG Tips

- `dict(zip(keys, values))` — show interviewer this one-liner for **dict from parallel lists**.
- `list(zip(*matrix))` for matrix transpose wins style points and replaces nested loops.
- **Pairwise iteration** like `for a, b in zip(xs, xs[1:])` is a common interview idiom for two-pointer style problems.
- **`strict=True`** in 3.10+ shows awareness of safer APIs.
- `zip_longest` is the answer to "zip unequal length without losing data."
- For **sliding windows** of size 3+, generalise with `zip(*[xs[i:] for i in range(k)])`.
- For DP, `dict(zip(keys, [None] * len(keys)))` initializes a memo dict cleanly.

## 18. Practice Problems

**Easy**
1. [LeetCode 1 — Two Sum](https://leetcode.com/problems/two-sum/) — build `dict(zip(nums, indices))`.
2. [LeetCode 344 — Reverse String](https://leetcode.com/problems/reverse-string/) — zip halves to swap pairs.
3. [LeetCode 242 — Valid Anagram](https://leetcode.com/problems/valid-anagram/) — `zip(sorted(a), sorted(b))` for char-wise check.

**Medium**
4. [LeetCode 14 — Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix/) — `zip(*strs)` char columns.
5. [LeetCode 49 — Group Anagrams](https://leetcode.com/problems/group-anagrams/) — zip chars to compare.
6. [LeetCode 844 — Backspace String Compare](https://leetcode.com/problems/backspace-string-compare/) — zip processed stacks.

**Hard**
7. [LeetCode 4 — Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/) — binary search equivalent to zip-based merging (and stretch thinking).

---

🧭 Next: [enumerate.md](./enumerate.md) — index + value helper.

Related loops: [../03_Control_Flow/for_loop.md](../03_Control_Flow/for_loop.md) · Lists: [../02_Data_Types/list.md](../02_Data_Types/list.md)