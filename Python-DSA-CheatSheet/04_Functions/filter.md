# filter() in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 04 — Functions
> 🔗 Related: [functions.md](./functions.md) · [lambda.md](./lambda.md) · [map.md](./map.md) · [reduce.md](./reduce.md) · [zip.md](./zip.md) · Back to [README](../README.md)

---

## 1. What is it?

`filter(predicate, iterable)` is a **built-in higher-order function** that returns a **lazy iterator** containing only the items of `iterable` for which `predicate(item)` is truthy. The original iterable is **not modified**.

```python
nums = [1, 2, 3, 4, 5, 6]
list(filter(lambda x: x % 2 == 0, nums))   # [2, 4, 6]
```

Key points:

- `predicate` — a function returning a truthy/falsy value; called once per item.
- If `predicate` is `None`, `filter` keeps items that are themselves **truthy** (drops zeros, empty strings, `None`, etc.).
- Returns a `filter` object — call `list()`, `set()`, or iterate to materialize it.
- Iteration is **lazy** — items are tested only when consumed.

## 2. Why do we use it?

- **Declaratively remove unwanted items** from a sequence.
- **Lazy evaluation** — useful for very large streams or when chaining with `map`.
- **Pairs naturally** with `lambda` for one-line predicates: `lambda x: x > 5`.
- **Eliminates explicit** `if` checks inside a `for` loop.

## 3. When should I choose it?

| Goal | Use `filter`? | Alternative |
|------|----------------|-------------|
| Keep items where predicate is true | ✅ Yes | list comp `[x for x in xs if p(x)]` |
| Pass an existing predicate like `bool`, `None` | ✅ Yes (`None` keeps truthy) | list comp |
| Lazy stream filtering | ✅ Yes | genexpr `(x for x in xs if p(x))` |
| Need to transform items, not select | ❌ Use `map` | see [map.md](./map.md) |
| Need to fold to one value | ❌ Use `reduce` | see [reduce.md](./reduce.md) |
| Need access to index while filtering | ❌ Use `enumerate` + comp | see [enumerate.md](./enumerate.md) |
| Need early break | ❌ `for` + `break` | — |

## 4. Syntax

```python
filter(predicate, iterable)
```

- `predicate` — callable returning truthy/falsy; pass `None` to filter on truthiness of item directly.
- `iterable` — a single iterable (no multiple like `map`).
- Returns — `filter` object (iterator).

### With `None`

```python
filter(None, iterable)         # keeps truthy items, drops falsy ones
list(filter(None, [0, 1, 2, "", "x", None, []]))   # [1, 2, 'x']
```

## 5. Basic Example

### 5.1 Keep evens

```python
list(filter(lambda x: x % 2 == 0, range(10)))
# [0, 2, 4, 6, 8]
```

### 5.2 Keep non-empty strings

```python
words = ["", "hi", None, "world", 0, False, [], "x"]
list(filter(None, words))            # ['hi', 'world', 'x']
```

### 5.3 Custom predicate — filter by attribute

```python
people = [{"name": "Tamil", "age": 30}, {"name": "Ana", "age": 22}]
list(filter(lambda p: p["age"] >= 25, people))
# [{'name': 'Tamil', 'age': 30}]
```

### 5.4 Negated filter — remove falsy

```python
list(filter(bool, [0, 1, "", "x", None, [], [1]]))   # [1, 'x', [1]]
```

## 6. Step-by-Step Dry Run

```python
nums = [1, 2, 3, 4, 5, 6]
out = list(filter(lambda x: x > 3, nums))
```

| Step | item | `x > 3`? | accumulate | running list |
|------|------|-----------|------------|----------------|
| 1 | 1 | False | skip | [] |
| 2 | 2 | False | skip | [] |
| 3 | 3 | False | skip | [] |
| 4 | 4 | True | keep | [4] |
| 5 | 5 | True | keep | [4, 5] |
| 6 | 6 | True | keep | [4, 5, 6] |

Final: `out = [4, 5, 6]`.

## 7. Built-in Methods / Idioms

### 7.1 `None` predicate — keep truthy

```python
list(filter(None, [0, 1, "", "x", None, [], 0.0, False]))   # [1, 'x']
```

`0`, `0.0`, `False`, `None`, `""`, `[]`, `{}`, `set()` are all falsy — `filter(None, xs)` drops them.

### 7.2 Predicate design — clear single-purpose

```python
is_even = lambda x: x % 2 == 0
is_positive = lambda x: x > 0
list(filter(is_even, filter(is_positive, [-2, -1, 0, 1, 2, 3, 4])))
# [2, 4]
```

But the comprehension is clearer:

```python
[x for x in [-2,-1,0,1,2,3,4] if x > 0 and x % 2 == 0]   # [2, 4]
```

### 7.3 Chained `filter` then `map`

```python
nums = [1, 2, 3, 4, 5, 6]
list(map(lambda x: x ** 2, filter(lambda x: x % 2 == 0, nums)))
# [4, 16, 36]
```

### 7.4 Materialize to set / dict keys

```python
set(filter(lambda x: x > 2, [3, 3, 1, 1, 5]))     # {3, 5}
dict(filter(lambda kv: kv[1] > 0, {"a": 0, "b": 3, "c": -1}.items()))
# {'b': 3}
```

### 7.5 Filter dict by value

```python
prices = {"a": 10, "b": 0, "c": 5, "d": 0}
dict(filter(lambda kv: kv[1] > 0, prices.items()))   # {'a': 10, 'c': 5}
```

### 7.6 `filter` on strings

```python
text = "hello world"
list(filter(str.isalpha, text))   # ['h','e','l','l','o','w','o','r','l','d']
"".join(filter(str.isalpha, "h3ll0 w0rld"))   # "hll wrld"
```

### 7.7 `itertools.filterfalse` — keep items where predicate is **false**

```python
from itertools import filterfalse
list(filterfalse(lambda x: x % 2 == 0, range(6)))   # [1, 3, 5]
```

When you'd otherwise write `not`, consider this built-in.

### 7.8 `filter` vs list comprehension

| Use | Functional style | Pythonic list comp |
|-----|------------------|---------------------|
| Keep items | `list(filter(p, xs))` | `[x for x in xs if p(x)]` |
| Keep truthy | `list(filter(None, xs))` | `[x for x in xs if x]` |
| Combined with transform | `list(map(f, filter(p, xs)))` | `[f(x) for x in xs if p(x)]` |

## 8. Interview Example

**Problem (Amazon-style):** Given a list of integers, return only those that are perfect squares.

```python
def is_square(n):
    if n < 0:
        return False
    r = int(n ** 0.5)
    return r * r == n

nums = [4, 9, 11, 16, 25, 27]
list(filter(is_square, nums))     # [4, 9, 16, 25]
```

**Top-K problem — keep non-zero freqs:**

```python
from collections import Counter
counts = Counter("aabbbcccc")
list(filter(lambda kv: kv[1] >= 2, counts.items()))
# [('a', 2), ('b', 3), ('c', 4)]
```

**Filter unique patterns (anagram reducer):**

```python
words = ["eat", "tea", "tan", "ate"]
seen = set()
def is_first(s):
    key = "".join(sorted(s))
    if key in seen: return False
    seen.add(key); return True

list(filter(is_first, words))   # ['eat', 'tan']
```

## 9. When NOT to use

- **A list comprehension reads clearer** — `[x for x in xs if p(x)]` is usually more Pythonic, especially when the predicate is a one-off.
- **When you need early break** — `filter` traverses the whole iterator; use `takewhile` from itertools or a `for` loop with `break`.
- **When the predicate is just `bool`** — `filter(None, xs)` is the conventional idiom, but `_ for _ in xs if _` is just as good.
- **When the predicate is complex with multiple branches** — write a `def`, not a `lambda`.
- **When the data is huge and you need batch processing** — use a lazy generator function instead.

## 10. Common Mistakes

1. **Forgetting to materialize** — `filter` returns an iterator; `print(filter(...))` shows `<filter object at 0x...>`. Wrap with `list()`.
2. **Using multiple iterables** — `filter` accepts only one (unlike `map`).
3. **Confusing `filter(None, xs)` with `filter(bool, xs)`** — they're equivalent, but `None` is the documented idiom.
4. **Using `not` in predicate** when `itertools.filterfalse` is clearer.
5. **Reusing a filter object** — it's exhausted after one consumption; recreate or materialize first.
6. **Forgetting laziness causes deferred errors** — exceptions in the predicate surface only when iterating.
7. **Side effects in predicate** — `filter(lambda x: print(x) or True, xs)` is hacky; prefer a `for` loop.
8. **Mixing `filter` and comprehension styles in same codebase** harms readability.

## 11. Memory Tricks

- `filter(p, xs)` — **"filter xs through p"** — only items passing p come out.
- Visualize a **sieve**: items pour in, holes let only some through.
- `filter(None, xs)` — "no filter means drop falsy" — strips out empties & zeros.
- Filter keeps items; Map changes them; Reduce collapses them.

## 12. Interview Shortcuts

- `list(filter(None, xs))` is the canonical **non-empty cleaner** for nested lists/strings.
- For **sorted keepers**: `sorted(filter(p, xs))` — filter first, sort the shorter.
- For **dict filtering**: `dict(filter(lambda kv: kv[1] > N, d.items()))` — common map-reduce pattern.
- Combine with `map` via comprehension when predicate + transform both needed: `[f(x) for x in xs if p(x)]`.
- When asked to remove duplicates: `list(dict.fromkeys(xs))` is cleaner than a `filter`.

## 13. Cheat Sheet Table

| Form | Returns | Equivalent |
|------|---------|------------|
| `filter(p, xs)` | iterator | `(x for x in xs if p(x))` |
| `list(filter(p, xs))` | list | `[x for x in xs if p(x)]` |
| `filter(None, xs)` | iterator of truthy | `(x for x in xs if x)` |
| `itertools.filterfalse(p, xs)` | iterator of false | `(x for x in xs if not p(x))` |
| `sorted(filter(p, xs))` | sorted list | sorted then filtered |
| `set(filter(p, xs))` | set | `{x for x in xs if p(x)}` |
| `dict(filter(p, d.items()))` | dict | `{k:v for k,v in d.items() if p((k,v))}` |

## 14. Time Complexity Table

| Operation | Cost |
|-----------|------|
| `filter(...)` creation | O(1) — lazy |
| `next(filter_obj)` | O(1) + O(predicate) |
| `list(filter(p, xs))` | O(n · predicate cost) |
| `filter(None, xs)` per item | O(1) truthiness check |
| memory (lazy) | O(1) |
| memory (list) | O(k) where k ≤ n items kept |

## 15. Visual Diagram (ASCII)

### filter pipeline

```
   ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐
   │1 │ │2 │ │3 │ │4 │ │5 │ │6 │    ← input iterable
   └─┬┘ └─┬┘ └─┬┘ └─┬┘ └─┬┘ └─┬┘
     │    │    │    │    │    │
     ▼    ▼    ▼    ▼    ▼    ▼
   ┌───────────────────────────┐
   │  predicate: x > 3         │  ← test each
   └────┬─────┬─────┬─────┬─────┘
        │     │     │     │
        ❌    ❌    ❌    ✓     ✓     ✓
            dropped  kept  kept  kept
                       │     │     │
                       ▼     ▼     ▼
                       ┌──┐ ┌──┐ ┌──┐
                       │4 │ │5 │ │6 │    ← filter object (lazy)
                       └──┘ └──┘ └──┘
   list(... → [4, 5, 6]
```

### Nested filter then map (lazy)

```
   [1,2,3,4,5,6]
       │
       ▼ filter (even)
   [2,4,6]
       │
       ▼ map (square)
   [4,16,36]
   
   Pure pipeline — no intermediate lists if consumed lazily.
```

## 16. Beginner Notes

> Remember:
> - `filter(p, xs)` keeps items where `p(item)` is truthy; drops the rest.
> - `filter(None, xs)` keeps truthy items directly (no predicate needed).
> - Returns an iterator — call `list()` to get a real list.
> - Original `xs` is **never modified**.
> - Equivalent one-liner: `[x for x in xs if p(x)]` — usually clearer for short predicates.
> - Pair with `map`: `map(f, filter(p, xs))` — but a single comprehension reads better.

## 17. FAANG Tips

- Show both forms — `filter(p, xs)` vs the equivalent comprehension — and **pick the clearer one** in the moment.
- `filter(None, xs)` is a clean way to mention "drop falsy" in **one word**.
- When dealing with **huge streams**, keep `filter` lazy — materialize only at the last step.
- For **dict filtering**, `dict(filter(..., d.items()))` is a snippet interviewers love seeing written in one breathe.
- `itertools.filterfalse` is the "drop items where p is true" — useful when reading-not-leaves vs reading-leaves.
- Functional programming coding style: emphasize `filter` + `map` + `reduce` as the core trio; mention [`lambda`](./lambda.md) ties them together.

## 18. Practice Problems

**Easy**
1. [LeetCode 1 — Two Sum](https://leetcode.com/problems/two-sum/) — filter candidates by complement.
2. [LeetCode 283 — Move Zeroes](https://leetcode.com/problems/move-zeroes/) — `filter(None, nums)` drops zeros (then pad).
3. [LeetCode 9 — Palindrome Number](https://leetcode.com/problems/palindrome-number/) — `filter(str.isalnum, s)`.

**Medium**
4. [LeetCode 1456 — Maximum Number of Vowels in a Substring of Given Length](https://leetcode.com/problems/maximum-number-of-vowels-in-a-substring-of-given-length/) — `filter` vowels approach.
5. [LeetCode 15 — 3Sum](https://leetcode.com/problems/3sum/) — filter duplicates from candidate triples.
6. [LeetCode 49 — Group Anagrams](https://leetcode.com/problems/group-anagrams/) — filter empty groups.

**Hard**
7. [LeetCode 30 — Substring with Concatenation of All Words](https://leetcode.com/problems/substring-with-concatenation-of-all-words/) — filter valid start indices by Counter equality.

---

🧭 Next: [reduce.md](./reduce.md) — collapse one iterable into a single value. Then [zip.md](./zip.md), [enumerate.md](./enumerate.md).

Related loops: [../03_Control_Flow/for_loop.md](../03_Control_Flow/for_loop.md) · Lists: [../02_Data_Types/list.md](../02_Data_Types/list.md)