# map() in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 04 — Functions
> 🔗 Related: [functions.md](./functions.md) · [lambda.md](./lambda.md) · [filter.md](./filter.md) · [reduce.md](./reduce.md) · [zip.md](./zip.md) · Back to [README](../README.md)

---

## 1. What is it?

`map(fn, *iterables)` is a **built-in higher-order function** that applies `fn` to **each item** of one or more iterables and returns a **lazy iterator** of the results. The original iterables are **not modified**.

```python
nums = [1, 2, 3, 4]
squares = map(lambda x: x ** 2, nums)
list(squares)                  # [1, 4, 9, 16]
```

Key points:

- `map` itself returns a **map object** (an iterator) — call `list()`, `set()`, or iterate to materialize it.
- The function is applied **lazily** — only when you consume the iterator.
- With **multiple iterables**, `fn` must accept that many arguments; iteration stops at the **shortest** iterable.
- `fn = None` with multiple iterables behaves like `zip` — yields tuples of items.

## 2. Why do we use it?

- **Apply a transformation** to every item of an iterable declaratively.
- **Lazy evaluation** — useful for huge streams where you don't want to build a huge list.
- **Avoids an explicit for-loop** with append instructions.
- **Pairs naturally** with `lambda` for one-liners.
- The map object can also be chained: `map(str, map(int, lines))`.

## 3. When should I choose it?

| Goal | Use `map`? | Alternative |
|------|------------|-------------|
| Transform every item of an iterable | ⚠️ OK, but consider list comp | `[f(x) for x in xs]` |
| Apply an **existing** function like `str`, `int`, `len` | ✅ Yes | list comp |
| Lazy infinite/very large stream | ✅ Yes | generator expression `(f(x) for x in xs)` |
| Need to filter, not transform | ❌ Use `filter` | see [filter.md](./filter.md) |
| Need to combine 2+ iterables elementwise | ✅ `map(f, a, b)` or `zip` | see [zip.md](./zip.md) |
| Need access to index | ❌ use `enumerate` | see [enumerate.md](./enumerate.md) |
| Need early break | ❌ use `for` | `next(itertools.islice(...))` |
| One-liner into list with side effects | ❌ | explicit `for` |

## 4. Syntax

```python
map(fn, *iterables)
```

- `fn` — callable applied to each item; pass `None` to skip and just pair.
- `*iterables` — one or more iterables; the result's length equals the **shortest** iterable.
- Returns — `map` object (iterator).

### Variants

```python
map(f, xs)                    # one iterable
map(f, xs, ys, zs)            # multiple iterables, f must take len(iterables) args
map(None, xs, ys)             # deprecated-ish; equivalent to zip in a sense
```

## 5. Basic Example

### 5.1 Single iterable

```python
list(map(str, [1, 2, 3]))                    # ['1', '2', '3']
list(map(len, ["hi", "world", "Python"]))    # [2, 5, 6]
```

### 5.2 Multiple iterables

```python
list(map(lambda a, b: a + b, [1, 2, 3], [10, 20, 30]))
# [11, 22, 33]
```

### 5.3 `None` as fn (zip-like — Python 2 behavior preserved in spirit)

```python
list(map(None, [1, 2, 3], ['a', 'b', 'c']))  # [(1,'a'), (2,'b'), (3,'c')]
```

This is valid in Python 3 (yields tuples) though more idiomatic to use `zip`:

```python
list(zip([1, 2, 3], ['a', 'b', 'c']))         # same result
```

## 6. Step-by-Step Dry Run

```python
nums = [1, 2, 3, 4]
out = list(map(lambda x: x * 2, nums))
```

| Step | index | `x` | `x * 2` | accumulating list |
|------|--------|-----|---------|---------------------|
| 1 | 0 | 1 | 2 | [2] |
| 2 | 1 | 2 | 4 | [2, 4] |
| 3 | 2 | 3 | 6 | [2, 4, 6] |
| 4 | 3 | 4 | 8 | [2, 4, 6, 8] |

Final: `out = [2, 4, 6, 8]`.

`map` itself only built an iterator; calling `list()` drove the iteration.

## 7. Built-in Methods / Idioms

### 7.1 Lazy iteration — consume later

```python
m = map(int, ["1", "2", "3"])
next(m)                  # 1
next(m)                  # 2
list(m)                  # [3]  (rest only)
```

A map object is **single-use**: once exhausted, it's empty.

### 7.2 Chain maps

```python
lines = ["1", "2", "3"]
nums = map(int, lines)
squared = map(lambda x: x ** 2, nums)
list(squared)                       # [1, 4, 9]
```

Each `next()` pulls through the whole chain lazily — no intermediate lists.

### 7.3 Map dict keys / values

```python
prices = {"apple": 2, "pie": 5}
list(map(str, prices.values()))     # ['2', '5']
list(map(lambda k: k.upper(), prices))   # ['APPLE', 'PIE']
```

Be explicit about whether you mean `.values()`, `.keys()`, or default iterators.

### 7.4 `None` as fn — pairing iterables (`zip`-like)

```python
list(map(None, [1,2,3], ['a','b']))   # [(1,'a'), (2,'b'), (3,None)]
```

Note the **third position is `None`** because `map(None, ...)` pairs even if lengths mismatch (extends with `None`) — different from `zip` which stops at the shortest. For the longest-with-fill behavior prefer `itertools.zip_longest`.

### 7.5 `map` vs list comprehension

```python
list(map(str, xs))     == [str(x) for x in xs]
list(map(f, xs, ys))   == [f(x, y) for x, y in zip(xs, ys)]
list(filter(p, xs))    == [x for x in xs if p(x)]
```

### 7.6 Materialize as set / dict

```python
set(map(len, words))                         # unique word lengths
dict(map(lambda w: (w, len(w)), ["hi","yo"]))# {'hi':2,'yo':2}
```

### 7.7 Read all lines and strip

```python
with open("data.txt") as f:
    lines = list(map(str.strip, f))          # all lines whitespace-stripped
```

### 7.8 Common interview — parse then transform

```python
data = "1,2,3,4,5"
nums = list(map(int, data.split(",")))       # [1, 2, 3, 4, 5]
total = sum(map(int, data.split(",")))       # 15
```

## 8. Interview Example

**Problem (Stripe-style):** Given a CSV string with header `"id,name,score"`, build list of dicts.

```python
text = "id,name,score\n1,Tamil,90\n2,Ana,85"
lines = text.splitlines()
header = lines[0].split(",")
rows = [dict(zip(header, map(str.strip, line.split(",")))) for line in lines[1:]]
# [{'id':'1','name':'Tamil','score':'90'}, {'id':'2','name':'Ana','score':'85'}]
```

Here `map(str.strip, ...)` cleans each cell, then `zip` merges with the header row.

**Classic — square every even number:**

```python
nums = [1, 2, 3, 4, 5, 6]
list(map(lambda x: x ** 2, filter(lambda x: x % 2 == 0, nums)))
# [4, 16, 36]
```

But the comprehension `[x**2 for x in nums if x % 2 == 0]` is clearer — interviewers may ask you to write both and compare readability.

## 9. When NOT to use

- **When a list comprehension is more readable** — Prefer list comp when the transform is significantly clearer with the loop variable named (almost always).
- **When you need early termination** — `map` runs through the whole iterator; use a `for` loop with `break`.
- **When you need conditional logic in the body** — only a single function call fits `map`; complex transforms belong in a `def` or a comprehension.
- **When you need to mutate the original list in place** — `list[:] = [f(x) for x in xs]` or a `for i` loop.
- **When `fn` is itself a heavy custom function** — name it and call it from a comprehension for stack traceability.

## 10. Common Mistakes

1. **Forgetting to materialize**: `map` returns an iterator — `type(map(.))` is `<class 'map'>`. `print(map(...))` shows `<map object at 0x...>`. Always wrap in `list()` when you need a real list.
2. **Iterating twice**: a map object is exhausted after the first iteration. Re-create it or store with `list()`.
3. **Wrong arity** with multiple iterables: `map(lambda x: x*2, a, b)` → `TypeError` because the lambda takes 1 arg.
4. **Assuming list length = longest**: it's the **shortest**.
5. **Confusing lazy evaluation timing**: side effects in `fn` happen lazily as you consume the map object — bugs in `fn` may not surface until much later.
6. **Comparing performance wrong**: `list(map(str, xs))` often slightly beats `[str(x) for x in xs]` because `str` is a C-level call, but they're close. Readability > micro-optimization.
7. **Using `map` for side effects only**: `map(print, xs)` discards results. Use a `for` loop.

## 11. Memory Tricks

- `map(f, xs)` reads as **"map f over xs"** — apply `f` to every item.
- Think of it as **column-wise** in a spreadsheet: every row gets the same formula.
- Remember it's **lazy** — picture a conveyor belt producing items on demand.
- "Map makes new values from old" vs **"Filter keeps some old values"** (see [filter.md](./filter.md)).
- Multiple iterables ⇒ picture parallel conveyors feeding one worker.

## 12. Interview Shortcuts

- `list(map(int, strings))` is the canonical "convert a string list to ints" — fast and idiomatic.
- `sum(map(int, line.split()))` parses and sums in one line.
- `dict(map(lambda kv: (kv[0], kv[1]*2), prices.items()))` — dict transform one-liner.
- For chained transforms, **chain `map` calls** to avoid intermediate lists and preserve laziness.
- When asked "compare with list comprehension" — say: same result, comprehensions are usually clearer, `map` is shorter when passing an existing (named) function like `str` / `int` / `len`.

## 13. Cheat Sheet Table

| Form | Returns | Equivalent |
|------|---------|------------|
| `map(f, xs)` | iterator | `(f(x) for x in xs)` |
| `list(map(f, xs))` | list | `[f(x) for x in xs]` |
| `tuple(map(f, xs))` | tuple | `tuple(f(x) for x in xs)` |
| `set(map(f, xs))` | set | `{f(x) for x in xs}` |
| `dict(map(f, pairs))` | dict | by-pair mapping |
| `map(f, a, b)` | iterator | `(f(x, y) for x, y in zip(a, b))` |
| `map(None, a, b)` | iterator of tuples | `zip`-like (extends with `None`) |

## 14. Time Complexity Table

| Operation | Cost |
|-----------|------|
| `map(...)` creation | O(1) — lazy, no work done |
| `next(map_obj)` | O(1) + one call to `fn` |
| `list(map(f, xs))` | O(n) calls to `fn` + O(n) list build |
| `map(f, xs, ys)` | O(min(len(xs), len(ys))) |
| memory (lazy) | O(1) — only one item buffered |
| memory (list) | O(n) after materialization |

## 15. Visual Diagram (ASCII)

### map pipeline

```
   ┌──┐  ┌──┐  ┌──┐  ┌──┐
   │1 │  │2 │  │3 │  │4 │    ← input iterable
   └─┬┘  └─┬┘  └─┬┘  └─┬┘
     │      │      │      │
     ▼      ▼      ▼      ▼
   ┌────────────────────────┐
   │  f(x) = x * 2  (lambda)│  ← function applied to each
   └─────┬───┬───┬───┬──────┘
         ▼   ▼   ▼   ▼
       ┌──┐┌──┐┌──┐┌──┐
       │2 ││4 ││6 ││8 │    ← map object (lazy iterator)
       └──┘└──┘└──┘└──┘

  list(map(...)) forces full materialization → [2, 4, 6, 8]
```

### Multiple iterables

```
   [1, 2, 3]      [10, 20, 30]
       │              │
       └────┬─────────┘
            ▼
       f(a, b) = a + b
            ▼
       [11, 22, 33]
   Stops at the shortest iterable.
```

### Chained maps (lazy pipeline)

```
   "1","2","3"     →     int → →  1,2,3   →  x**2 →  1,4,9
   ─────────┬─────────────────┬──────────────────┬──────────
            ▼                 ▼                  ▼
        map(int,...)     map(lambda x: x**2, ...)
            (lazy → no intermediate list created)
```

## 16. Beginner Notes

> Remember:
> - `map(f, xs)` applies `f` to every item of `xs` and returns a **lazy iterator**.
> - Call `list(map(...))` to actually get a list out.
> - With multiple iterables, `f` takes one arg per iterable; iteration stops at the **shortest**.
> - `map` doesn't modify the original — it builds a new (lazy) iterator.
> - A list comprehension `[f(x) for x in xs]` does the same and is often clearer.
> - Use `map` when passing an existing function (`str`, `int`, `len`) — short and clean.

## 17. FAANG Tips

- **Lazy `map`** fits streaming pipelines — `map(parse, file.readlines())` without holding everything in memory.
- For interviews, demonstrate you know **both map and list comp**, and pick whichever communicates intent best.
- `map` + `filter` + `reduce` chain is the **functional style** — pair with [`lambda`](./lambda.md). Talk about that explicitly when asked "functional programming in Python."
- When the function is already imported, **avoid wrapping it in a lambda** — `map(str, xs)` is better than `map(lambda x: str(x), xs)`.
- Materialize only once at the end with `list()` — keep everything lazy up to that point.
- For **big-data** style code, mention you'd use generators (yield) to stay memory-efficient.

## 18. Practice Problems

**Easy**
1. [LeetCode 9 — Palindrome Number](https://leetcode.com/problems/palindrome-number/) — `map(int, str(n))` to get digits.
2. [LeetCode 258 — Add Digits](https://leetcode.com/problems/add-digits/) — `sum(map(int, str(n)))`.
3. [LeetCode 1365 — How Many Numbers Are Smaller Than the Current Number](https://leetcode.com/problems/how-many-numbers-are-smaller-than-the-current-number/) — `map` compare.

**Medium**
4. [LeetCode 412 — Fizz Buzz](https://leetcode.com/problems/fizz-buzz/) — `map` a function over `range(1, n+1)`.
5. [LeetCode 1431 — Kids With the Greatest Number of Candies](https://leetcode.com/problems/kids-with-the-greatest-number-of-candies/) — `map` comparing each to `max - extra`.
6. [LeetCode 8 — String to Integer (atoi)](https://leetcode.com/problems/string-to-integer-atoi/) — parse map-style.

**Hard**
7. [LeetCode 41 — First Missing Positive](https://leetcode.com/problems/first-missing-positive/) — index placing uses `map` style transforms.

---

🧭 Next: [filter.md](./filter.md) — keeping items that pass a predicate. Then [reduce.md](./reduce.md), [zip.md](./zip.md), [enumerate.md](./enumerate.md).

Related loops: [../03_Control_Flow/for_loop.md](../03_Control_Flow/for_loop.md) · Lists: [../02_Data_Types/list.md](../02_Data_Types/list.md)