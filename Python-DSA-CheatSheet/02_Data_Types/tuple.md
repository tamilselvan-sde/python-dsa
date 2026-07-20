# Tuples in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 02 — Data Types
> 🔗 Related: [strings.md](./strings.md) · [list.md](./list.md) · [numbers.md](./numbers.md)
> Back to [README](../README.md)

---

## 1. What is it?

A **tuple** is an **immutable, ordered sequence** of arbitrary objects, written with parentheses (or just commas).

```python
t = (1, 2, 3)
t = 1, 2, 3          # parenthesis optional
t = ()               # empty tuple
t = (1,)             # single-element tuple (comma required!)
```

Immutability means: you **cannot** reassign, add, or remove elements after creation. However, **mutable elements inside** (lists, dicts) can still be mutated — that's a common gotcha.

A tuple's "shape" (length + element types, structurally) makes it ideal for **fixed records** and **dictionary keys**.

---

## 2. Why do we use it?

- **Hashable** (if all elements are hashable) → usable as `dict` keys / `set` members.
- **Immutable** → safer shared data, intentional constants.
- **Lightweight** — slightly smaller & faster than lists.
- **Packing/unpacking** for multiple-return functions, coordinate pairs, swap idiom `a,b = b,a`.
- **Fixed schema** records: `(user_id, name, age)` — coordinates, RGB triples, etc.

---

## 3. When should I choose it?

| Need                                  | Use               |
|---------------------------------------|--------------------|
| Fixed record, won't change            | `tuple`            |
| Will grow / mutate                    | `list`             |
| Use as dict key / set element         | `tuple` (hashable) |
| Arbitrary order of strings            | `list` or `tuple`  |
| Numeric arrays                        | `numpy.ndarray`    |
| Streamed/lazy sequence                | generator          |
| Named fields                          | `NamedTuple` / `dataclass` |
| Text                                  | `str`              |
| Bytes                                 | `bytes`            |

---

## 4. Syntax

```python
t = ()                       # empty
t = (1,)                     # single (comma!)
t = 1, 2, 3                  # parenthesis not needed
t = (1, "a", [2,3])          # heterogeneous, can hold mutables
t = tuple("abc")             # ('a','b','c')
t = tuple(range(3))          # (0,1,2)
t = (1, (2, 3), 4)           # nested
t[0], t[-1], t[1:3], t[::-1] # index & slice (return tuples)
+ : (1,2) + (3,)             # -> (1,2,3)  new tuple
* : (0,) * 3                 # -> (0,0,0)
```

---

## 5. Basic Example

```python
point = (3, 4)
x, y = point                 # unpack
print(x, y)                  # 3 4

def min_max(nums):
    return min(nums), max(nums)     # returns tuple

lo, hi = min_max([3, 7, 1, 5])
print(lo, hi)                # 1 7
```

---

## 6. Step-by-Step Dry Run

**Swap without temp** (idiom):
```python
a, b = 1, 2
a, b = b, a          # uses tuple pack/unpack
# Equivalent: tmp = (b, a); a, b = tmp
```
Dry run:
```
Step 1  RHS first:  tmp = (2, 1)
Step 2  a, b = (2, 1)  -> a = 2, b = 1
```

**Returning multiple values from a search:**
```python
def find(nums, target):
    for i, n in enumerate(nums):
        if n == target:
            return True, i        # tuple packing
    return False, -1
```

---

## 7. Built-in Methods

### 7.1 `tuple.count(x)`
- Counts occurrences of value `x`.
- Time: **O(n)**.
- Example: `(1, 2, 1, 1).count(1)` -> `3`.

### 7.2 `tuple.index(x[, start[, stop]])`
- Returns index of first `x`; else `ValueError`.
- Time: **O(n)**.
- Example: `(10, 20, 30).index(20)` -> `1`.

### 7.3 `len(t)`
- Time: **O(1)**.
- Example: `len((1,2,3))` -> `3`.

### 7.4 `x in t` / `x not in t`
- Linear scan.
- Time: **O(n)**.
- Example: `2 in (1,2,3)` -> `True`.

### 7.5 `min`, `max`, `sum`, `sorted`, `reversed`
- Work on tuples just like iterables.
- `sorted(t)` returns a **list**, **not** a tuple.
- Example: `sorted((3,1,2))` -> `[1,2,3]`; `tuple(sorted((3,1,2)))` -> `(1,2,3)`.

### 7.6 `tuple(iterable)`
- Constructor — converts any iterable to a tuple.

### 7.7 Packing & Unpacking
```python
# Pack
t = 1, 2, 3
# Unpack
a, b, c = t
a, *rest = t            # a = 1, rest = [2, 3]
*first, last = t        # first = [1, 2], last = 3
a, *mid, z = (1, 2, 3, 4, 5)   # a=1, mid=[2,3,4], z=5
```

### 7.8 Nested unpacking
```python
( (x, y), z ) = ((1, 2), 3)
```

### 7.9 Single-element tuple
```python
(5)         # int 5  -- NOT a tuple
(5,)        # tuple (5,)
5,          # tuple (5,)  (trailing comma)
```
This is why the trailing comma is the **real** tuple marker.

### 7.10 `collections.namedtuple`
```python
from collections import namedtuple
Point = namedtuple("Point", "x y")
p = Point(3, 4)
p.x              # 3
p.y              # 4
p[0]             # 3
p._asdict()      # {'x':3,'y':4}
p._replace(x=10) # Point(x=10, y=4)  (returns new namedtuple)
```

### 7.11 Is there "tuple comprehension"?
**No.** Parentheses with `for` create a **generator expression**:
```python
(x*x for x in range(5))   # generator -> lazy, 1-elem memory
tuple(x*x for x in range(5))  # -> (0,1,4,9,16)
```
A list comprehension uses `[]`:
```python
[x*x for x in range(5)]   # [0,1,4,9,16]  eager list
```
Use generator when streaming / memory-conscious; use `tuple(...)` only when you need hashability/immutability (and it fits in memory).

---

## 8. Interview Example

**Two Sum variant — return indices in tuple, use tuple as dict key for memoized DFS:**

```python
def climb(n, memo={}):
    if n <= 2: return n
    if n in memo: return memo[n]
    memo[n] = climb(n-1) + climb(n-2)
    return memo[n]
```

Tuples as **dictionary keys** (immutable graph memo):
```python
seen = {(r, c): True for r, c in coords}
```

---

## 9. When NOT to use

- Need to mutate / append / sort in place → use `list`.
- Need default mutable args (avoid `def f(x=())` only if you'll mutate).
- Very large streamed data → prefer generator.
- Need named attributes / methods → use `dataclass` or `NamedTuple`.
- "I want a tuple but with one optional field" → tuple is positional; use `dataclass` instead.

---

## 10. Common Mistakes

```python
# 1. Single element without comma
t = (1)            # int, not tuple
t = (1,)           # tuple

# 2. Trying to mutate
t = (1, 2, 3)
t[0] = 9           # TypeError: tuple immutable

# 3. Mutable elements still mutable
t = (1, [2, 3])
t[1].append(4)     # works!  (the list inside is mutable)
t = (1, [2,3,4])

# 4. Unhashable tuple can't be a dict key
{(1, [2]): "x"}    # TypeError: unhashable type: 'list'

# 5. sorted returns a list, not a tuple
type(sorted((1,2,3)))   # <class 'list'>

# 6. Mixed elements prevent comparisons
(1, "a") < (1, 2)  # TypeError in Py3: < between int and str
```

---

## 11. Memory Tricks

- **Tuple = "locked list"** — read-only after creation.
- **Trailing comma** is the real tuple marker; `(5)` is just `5`.
- **Packing** = "make a bundle"; **unpacking** = "open the bundle".
- **Immutable wrapper ≠ immutable contents** — mutables inside can change.
- Use `namedtuple` when **positional indices hurt readability**.

---

## 12. Interview Shortcuts

- `a, b = b, a` — swap with tuple packing.
- Return `False, -1` etc. for multi-result functions.
- Use tuples as **dict keys** for memoization (`@lru_cache` even auto-handles this when args are hashable).
- `tuple(iterable)` is the way to lock down a mutable list snapshot immutably.
- `(x for x in xs)` is a lazy generator — saves memory.
- `namedtuple` gives attribute access AND keeps hashability/positional.

---

## 13. Cheat Sheet Table

| Method / Op            | Use                                  |
|------------------------|--------------------------------------|
| `t.count(x)`           | count occurrences                    |
| `t.index(x)`           | first index of x                     |
| `len(t)`               | length                               |
| `x in t`               | membership                           |
| `t[i]`, `t[a:b:c]`     | index/slice (returns new tuple)      |
| `t1 + t2`              | concatenation (new tuple)            |
| `t * k`                | repetition                           |
| `a, b, c = t`          | unpack                               |
| `a, *rest = t`         | extended unpack                      |
| `(1,)`                 | single-element tuple                 |
| `tuple(iter)`          | from iterable                        |
| `namedtuple(...)`      | named fields                         |

---

## 14. Time Complexity Table

| Operation                  | Complexity | Notes                      |
|----------------------------|------------|----------------------------|
| `t[i]`                     | O(1)       |                            |
| `t[a:b]` (slice)           | O(k)       | new tuple (copies refs)    |
| `len(t)`                   | O(1)       | cached                     |
| `x in t` / `t.index` / `t.count` | O(n) | linear scan                |
| `t1 + t2`                  | O(n+m)     | new tuple                  |
| `t * k`                    | O(n*k)     |                            |
| `tuple(iter)`              | O(n)       |                            |
| `hash(t)`                  | O(n) (cached after first) | elements hashed once |
| `min/max/sum`              | O(n)       |                            |
| `sorted(t)`                | O(n log n) | returns list               |

---

## 15. Visual Diagram (ASCII)

### Tuple as immutable record
```
Index:     0   1   2
         +---+---+---+
t:       | 1 | 2 | 3 |    (length fixed; frozen)
         +---+---+---+
Neg:     -3  -2  -1

t[1] = 9  -> TypeError (cannot assign)

t[1] = 9
         ╳╳╳  frozen
```

### Mutable element inside immutable tuple
```
t = (1, [2,3])
         ref--->[ mutable list: 2 3 ]
                  │
                  v
t[1].append(4)   -> list grows to [2,3,4]
                  tuple still has 2 elements (its container shape is fixed)
```

### Packing / unpacking
```
Pack:    (1, 2, 3)           pack 3 values into bundle
Unpack:  a, b, c = (1,2,3)   open bundle -> a=1 b=2 c=3
Swap:    a, b = b, a          via temp (b, a)
```

### Two-pointer swap with tuple syntax
```
arr = [1, 2, 3, 4]
arr[i], arr[j] = arr[j], arr[i]
   i=0, j=3
       ┌───── swap ─────┐
before: [1, 2, 3, 4]
after:  [4, 2, 3, 1]
```

### Generator vs list comprehension
```
( x for x in range(5) )        -> <generator> lazy, 1 object at a time
[ x for x in range(5) ]        -> [0,1,2,3,4]    eager list
tuple( x for x in range(5) )  -> (0,1,2,3,4)    eager, immutable
```

---

## 16. Beginner Notes

> Remember:
> - Tuples are **immutable**; lists are **mutable**.
> - `(5)` is just `5`; `(5,)` is a 1-tuple.
> - Mutable elements inside a tuple **can** be changed — outer shape can't.
> - Tuples are **hashable** if all their contents are hashable — usable as `dict` keys.
> - Use `namedtuple` to give names instead of indexes for readability.
> - There is **no tuple comprehension** — `(x for x in xs)` is a generator.
> - `sorted(tuple)` returns a **list**, not a tuple.

---

## 17. FAANG Tips

- Use tuples as **coordinate / state keys** in DP memo tables: `@lru_cache` on `f(r, c, k)` hashes the arg tuple automatically.
- `namedtuple` is great for return values from parsing / network — like a tiny struct.
- **For performance**: tuple iteration is **slightly faster** than list iteration; tiny constant-factor, but matters in hot loops.
- **For memory**: small tuples are smaller (no over-allocation buffer like list).
- Default tuple sentinel: `def f(x=None)` — use `None`, not `()`, as sentinel.
- When you need immutable behavior but with defaults/clear, prefer `frozenset`, `NamedTuple`, or `dataclass(frozen=True)`.

---

## 18. Practice Problems

**Easy**
1. LeetCode 1 – Two Sum (return tuple of indices)
2. LeetCode 448 – Find All Numbers Disappeared in an Array
3. LeetCode 263 – Ugly Number (tuple-based memo)

**Medium**
4. LeetCode 62 – Unique Paths (DP key as `(r, c)`)
5. LeetCode 49 – Group Anagrams (use sorted tuple as key)
6. LeetCode 621 – Task Scheduler

**Hard**
7. LeetCode 336 – Palindrome Pairs (tuple keys + trie)
8. LeetCode 115 – Distinct Subsequences

---

> Next: [numbers.md](./numbers.md) · Back to [README](../README.md)