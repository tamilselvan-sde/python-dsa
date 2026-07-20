# Set

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
> Section: 02 — Data Types
> 🔗 Related: [dictionary.md](./dictionary.md), [../06_Collections/counter.md](../06_Collections/counter.md), [../07_Algorithms/hash_map.md](../07_Algorithms/hash_map.md) · Back to [README](../README.md)

## 1. What is it?

A **set** is an **unordered**, **mutable** collection of **unique**, **hashable** elements. Internally it's a hash table that stores only keys (no values), giving O(1) average membership test, add, and remove.

- Elements must be **hashable** (same rule as dict keys): `int`, `str`, `tuple` of hashables, `frozenset`. **Not** `list`, `dict`, `set`.
- `frozenset` is the **immutable** cousin — hashable, so it can be an element of another set or a dict key.
- A set is essentially a dict with no values.

## 2. Why do we use it?

- **Deduplication** — `list(set(xs))` removes duplicates in O(n).
- **Fast membership** — `x in s` is O(1) vs O(n) for a list.
- **Set algebra** — union, intersection, difference, symmetric difference are first-class operations.
- **Uniqueness constraint** — modeling a "thing can only be counted once".
- **Reachability/visited tracking** in BFS/DFS — the canonical `seen = set()`.

## 3. When should I choose it?

| Need | Choose |
|------|--------|
| Just "is X present?" with no value | **set** |
| Map key → value | **dict** |
| Index-based or ordered data | **list** |
| Sorted unique elements | `sorted(set(xs))` or tree structure |
| Immutable unique collection (hashable) | **frozenset** |
| Count elements with multiplicity | **Counter** / list |
| Insertion-ordered unique items | `dict.fromkeys(xs)` (3.7+) preserves order |

## 4. Syntax

```python
# Empty set — MUST use set(), because {} makes a dict!
s1 = set()

# Literal (one or more elements)
s2 = {1, 2, 3}

# From any iterable
s3 = set([1, 2, 2, 3])         # {1, 2, 3}
s4 = set("hello")              # {"h", "e", "l", "o"}

# Comprehension
s5 = {x*x for x in range(5) if x % 2}

# Frozen (immutable, hashable)
fs = frozenset({1, 2, 3})

# Type annotation
names: set[str] = {"Ada", "Bob"}
```

## 5. Basic Example

```python
s = {1, 2, 3, 4}

s.add(5)            # {1,2,3,4,5}
s.discard(99)       # no error — ok
s.remove(1)         # remove (raises if missing)
3 in s              # True
len(s)              # 4

a = {1, 2, 3}
b = {3, 4, 5}
a | b   # union            {1,2,3,4,5}
a & b   # intersection     {3}
a - b   # difference       {1,2}
a ^ b   # symmetric diff   {1,2,4,5}
```

## 6. Step-by-Step Dry Run

Add `5` to `{1, 2, 3}` (table size 8):

```
Step 1: hash(5) = h
Step 2: index = h % 8
Step 3: bucket[index] empty  → place (5) in bucket
Step 4: set returns None, set is now {1, 2, 3, 5}
```

Membership `3 in {1, 2, 3}`:
```
hash(3) → index → scan bucket → 3 found → True
```

Difference `{1,2,3} - {2,3,4}`:
```
result = empty set
for x in {1,2,3}:
    if x not in {2,3,4}:  → O(1) each
        result.add(x)
result = {1}
```

## 7. Built-in Methods (EXHAUSTIVE)

### Mutating single-element operations

#### `s.add(elem)`
- **Purpose:** Add element; no-op if already present.
- **Syntax:** `s.add(elem)`
- **Input:** hashable elem
- **Output:** `None`
- **Time:** O(1) avg
- **Example:** `s = {1}; s.add(2); s  # {1,2}`
- **Common mistake:** Passing unhashable `list` → `TypeError`.

#### `s.remove(elem)`
- **Purpose:** Remove element; raise `KeyError` if absent.
- **Syntax:** `s.remove(elem)`
- **Time:** O(1) avg
- **Example:** `s = {1,2}; s.remove(1); s  # {2}`
- **Common mistake:** `s.remove(x)` raises if x not in s; use `discard` for safe removal.

#### `s.discard(elem)`
- **Purpose:** Remove element if present; no-op otherwise (no error).
- **Syntax:** `s.discard(elem)`
- **Time:** O(1) avg
- **Example:** `s = {1}; s.discard(99); s  # {1}`
- **Interview use:** Safe removal in visited-set cleanup.

#### `s.pop()`
- **Purpose:** Remove and return an **arbitrary** element (sets are unordered).
- **Syntax:** `s.pop()`
- **Output:** an element; `KeyError` if empty
- **Time:** O(1) avg
- **Common mistake:** Assuming it returns "first" or "smallest" — it does not.

#### `s.clear()`
- **Purpose:** Empty the set.
- **Time:** O(n)
- **Example:** `s.clear(); s  # set()`

### Non-mutating algebraic operations

#### Union  `s | t` or `s.union(*others)`
- **Purpose:** New set with elements in **s OR t** (or more).
- **Syntax:** `s | t` / `s.union(t1, t2, ...)`
- **Input:** set(s) or any iterable
- **Output:** new set
- **Time:** O(len(s) + len(t))
- **Example:** `{1,2} | {2,3}  # {1,2,3}`
- **Shortcut:** `|` requires both sets; `union` accepts any iterable.

#### Intersection  `s & t` or `s.intersection(*others)`
- **Purpose:** New set with elements in **s AND t**.
- **Time:** O(min(len(s), len(t)))
- **Example:** `{1,2,3} & {2,3,4}  # {2,3}`
- **Shortcut:** Pair the smaller set on the left for speed: `small.intersection(big)`.

#### Difference  `s - t` or `s.difference(*others)`
- **Purpose:** Elements in **s but NOT in t**.
- **Time:** O(len(s))
- **Example:** `{1,2,3} - {2,3}  # {1}`
- **Note:** `s - t ≠ t - s` (asymmetric).

#### Symmetric Difference  `s ^ t` or `s.symmetric_difference(t)`
- **Purpose:** Elements in **s XOR t** (in exactly one).
- **Time:** O(len(s) + len(t))
- **Example:** `{1,2,3} ^ {3,4}  # {1,2,4}`
- **Property:** `s ^ t == (s - t) | (t - s) == (s | t) - (s & t)`.

### In-place mutating algebraic operations

#### `s.update(t)` / `s |= t`
- **Purpose:** Add all elements of t into s (in-place union).
- **Time:** O(len(t))
- **Example:** `s={1}; s.update({2,3}); s  # {1,2,3}`

#### `s.intersection_update(t)` / `s &= t`
- **Purpose:** Keep only elements in both s and t.
- **Time:** O(min(len(s), len(t)))
- **Example:** `s={1,2,3}; s &= {2,3,4}; s  # {2,3}`

#### `s.difference_update(t)` / `s -= t`
- **Purpose:** Remove from s all elements found in t.
- **Time:** O(len(t))
- **Example:** `s={1,2,3}; s -= {2}; s  # {1,3}`

#### `s.symmetric_difference_update(t)` / `s ^= t`
- **Purpose:** In-place XOR with t.
- **Time:** O(len(s) + len(t))
- **Example:** `s={1,2}; s ^= {2,3}; s  # {1,3}`

### Sub/superset tests

#### `s.issuperset(t)` or `s >= t`
- **Purpose:** True if every element of t is in s.
- **Time:** O(len(t))
- **Example:** `{1,2,3} >= {1,2}  # True`

#### `s.issubset(t)` or `s <= t`
- **Purpose:** True if every element of s is in t.
- **Time:** O(len(s))
- **Example:** `{1,2} <= {1,2,3}  # True`

#### `s.isdisjoint(t)`
- **Purpose:** True if s and t share **no** elements (= empty intersection).
- **Time:** O(min(len(s), len(t)))
- **Example:** `{1,2}.isdisjoint({3,4})  # True`
- **Interview use:** Faster than `s & t == set()` because it can short-circuit on first common element.

### Copying & freezing

#### `s.copy()`
- **Purpose:** Shallow copy (safe — elements are immutable/hashable anyway).
- **Time:** O(n)
- **Example:** `s2 = s.copy()`

#### `frozenset(iterable)`
- **Purpose:** Immutable set; hashable, so usable as dict key or set element.
- **Example:**
  ```python
  fs = frozenset({1, 2, 3})
  fs.add(4)             # AttributeError
  {fs, frozenset({4})} # OK — frozenset is hashable
  d = {fs: "meta"}      # OK as dict key
  ```
- **Time:** O(n) construction
- **Common mistake:** Trying to add/remove from a frozenset.

### Other built-ins
- `len(s)` → O(1)
- `x in s` → O(1) avg
- `for x in s:` → O(n) (unordered iteration)
- `set()` supports `__or__`, `__and__`, etc. to interop with other iterables via the named methods (not operators).

## 8. Interview Example — Contains Duplicate (LeetCode 217)

```python
def containsDuplicate(nums):
    seen = set()
    for n in nums:
        if n in seen:      # O(1)
            return True
        seen.add(n)
    return False
```

**Why a set?** We only care "have we seen n before?" — no value to attach, so a dict would waste space.

Longer example — Intersection of Two Arrays (LeetCode 349):

```python
def intersection(a, b):
    return list(set(a) & set(b))           # one-liner
```

## 9. When NOT to use

- **Order matters** → use list / dict (3.7+ ordered).
- **Duplicates matter** → use list or `Counter`.
- **Need key→value** → use dict.
- **Need smallest/largest efficiently** → use `heapq` or a sorted structure (set has no min/max in O(1)).
- **Tiny data with one lookup** → overhead of hashing may exceed linear scan; for n < ~5, list is fine.
- **You need to mutate elements in place** → sets can't hold unhashable elements.

## 10. Common Mistakes

1. **`{}` is an empty dict, not an empty set.** Use `set()`.
2. **`remove` raises `KeyError`; `discard` doesn't.** Pick the right one.
3. **Trying to put a `list` in a set** → `TypeError: unhashable type: 'list'`. Convert to `tuple`.
4. **`set.pop()` is not "first element"** — it's arbitrary. Never rely on order.
5. **`s | t` requires both sets** — use `s.union(t)` if t is a list/iterable.
6. **Modifying a set while iterating it** → `RuntimeError: Set changed size during iteration`. Copy first or collect-then-update.
7. **`frozenset` is not subscriptable** — no indexing `[0]`.
8. **Assuming `subset` order** → `{1,2} <= {2,1}` is True (order irrelevant).
9. **`{1, 2, 3} == {3, 2, 1}` is True** — sets compare by membership, not order.
10. **Negation pitfall** — `not x in s` is valid but reads worse than `x not in s`.

## 11. Memory Tricks

- **"ADD ACCEPTS, REMOVE RAISES, DISCARD QUIETLY"** — `add`/`remove`/`discard` triplet.
- **"Pipe OR, Amp AND, Minus NOT"** — `|` (union), `&` (intersection), `-` (difference).
- **"Hat XOR = exactly one"** — `s ^ t` is the "exclusive or" of membership.
- **"Frozen for keys"** — `frozenset` is the only set you can put inside another set.
- **"Empty? Use `set()`"** — `{}` is a dict trap.
- **"HASH = HASHABLE"** — anything in a set must pass `hash(...)`.

## 12. Interview Shortcuts

- **Dedup one-shot**: `list(set(xs))` (3.7+ guarantees dict/set iteration ≈ insertion order for `dict.fromkeys` if order matters).
- **Intersection two arrays**: `set(a) & set(b)`.
- **Count distinct**: `len(set(xs))`.
- **BFS/DFS visited**: `seen = set()`; `if nxt in seen: continue`.
- **Common elements among K lists**: `set.intersection(*map(set, lists))`.
- **"In any of"**: `x in set(all_targets)` once, then O(1) checks.
- **Anagram signatures**: `frozenset` of char counts, `tuple(sorted(word))` works too.
- **Find all duplicates**: `seen_once`, `seen_twice`; or `list -> Counter`.
- **Set difference for "missing from A"**: `set(want) - set(have)`.
- **All subsets / power set**: `itertools.combinations` or recursive `"bits"` with `frozenset`.

## 13. Cheat Sheet Table

| Method / Op | Use |
|-------------|-----|
| `s.add(e)` | Insert element if absent |
| `s.remove(e)` | Remove (raises if absent) |
| `s.discard(e)` | Remove (safe) |
| `s.pop()` | Remove arbitrary element |
| `s.clear()` | Empty the set |
| `s.copy()` | Shallow copy |
| `s \| t` / `s.union` | Union |
| `s & t` / `s.intersection` | Intersection |
| `s - t` / `s.difference` | Difference (s not in t) |
| `s ^ t` / `s.symmetric_difference` | XOR |
| `s.update(t)` / `s \|= t` | In-place union |
| `s.intersection_update(t)` / `s &= t` | In-place intersection |
| `s.difference_update(t)` / `s -= t` | In-place difference |
| `s.symmetric_difference_update(t)` / `s ^= t` | In-place XOR |
| `s >= t` / `s.issuperset(t)` | s includes all of t |
| `s <= t` / `s.issubset(t)` | s inside t |
| `s.isdisjoint(t)` | No common elements |
| `frozenset(it)` | Immutable, hashable set |
| `e in s` | Membership O(1) |
| `len(s)` | Count elements O(1) |

## 14. Time Complexity Table

| Operation | Average | Worst |
|-----------|---------|-------|
| `s.add(e)` | O(1) | O(n) (resize/rehash) |
| `s.remove(e)` / `discard` | O(1) | O(n) |
| `s.pop()` | O(1) | O(1) |
| `e in s` | O(1) | O(n) |
| `len(s)` | O(1) | O(1) |
| `s \| t` union | O(len(s)+len(t)) | O(len(s)+len(t)) |
| `s & t` intersection | O(min(len(s), len(t))) | O(len(s)*len(t)) worst w/ collisions |
| `s - t` difference | O(len(s)) | O(len(s)) |
| `s ^ t` symmetric diff | O(len(s)+len(t)) | O(len(s)+len(t)) |
| `s.issuperset(t)` | O(len(t)) | O(len(t)) |
| `s.isdisjoint(t)` | O(min) early-exit | O(min) |
| `s.copy()` | O(n) | O(n) |
| iterate all | O(n) | O(n) |

### Worst case note
Adversarial hash collisions (theoretically) collapse O(1) → O(n). Python mitigates with randomized hashing (3.3+) per process for `str`/`bytes`, so practical damage is rare. Mention this when an interviewer probes DoS.

## 15. Visual Diagram (ASCII)

### Venn diagrams for set operations

```
   UNION  s | t                INTERSECTION  s & t
      _____                      _____
    /   V   \                  /       \
   /  A  B   \                /  only s∩t  \   ← overlap
  /  s     t  \              /              \
  \___________/              \______________/

   DIFFERENCE  s - t           SYMMETRIC DIFF  s ^ t
      _____                      _____
    /   ⊂     \                 /   V   \
   /  A only   \              /  A      B  \   ← both shaded
  /  s     t    \            /  s     t    \   but not overlap
  \____________/              \____________/
```

(set-diff shaded = s minus the part overlapping t; symmetric = everything except the middle.)

### Hash table bucket view (chaining)

```
   set s = {3, "a", 9, "b"}
   hash & mod ── bucket index

       index   bucket (linked list of entries)
       ─────   ────────────────────────────────
        0      ─
        1      ("a") ──┐            ← collision
        2      (3)     ┐            (same bucket index)
        3      (9)   ──┘            ← none
        4      ─
        5      ("b")
        ...
```

### Membership lookup flow

```
        x in s
           │
           ▼
   ┌──────────────┐
   │  hash(x)     │──────►  index = h % capacity
   └──────────────┘
           │
           ▼
   scan bucket[index]:
       ┌─ match found  ──▶ True
       └─ chain end    ──▶ False
```

## 16. Beginner Notes

> **Remember:**
> - `{}` is a **dict**; `set()` is empty set.
> - All elements must be **hashable** (immutable).
> - **No order, no indexing** — you can't do `s[0]`.
> - Use `discard` for safe removal; `remove` for strict removal.
> - `frozenset` is the immutable, hashable set.
> - `|`, `&`, `-`, `^` are the **operators**; methods accept any iterable.
> - In-place ops end in `_update` or use `|=`, `&=`, `-=`, `^=`.
> - `set == set` ignores order; only membership matters.

## 17. FAANG Tips

1. **First instinct: "deduplicate or check existence?" → reach for a set.**
2. **For pair / triplet / "two arrays overlap" problems**, convert both to sets and use `&`/`-`/`^`.
3. **In graph traversal**, `seen` as a set avoids accidental re-visits in O(1).
4. **Subarray with distinct constraint** — sliding window + `set` of in-window elements (or `dict` if you need indices).
5. **Power set / combinations** — `frozenset` makes a clean, hashable signature for memoization.
6. **Watch immutability** — store `tuple(sorted(item))` not `list` inside sets.
7. **Large intersection**: iterate the smaller set and check `in` on the larger to minimize work.
8. **Don't `sort` a thinking-it's-ordered set** — `sorted(s)` returns a list (sets have no inherent sort).
9. **When you need smallest/largest often**, prefer `heapq` not `min(s)`/`max(s)` (each is O(n)).
10. **Profile if hashing becomes slow** — sets of floats with hash collisions degrade; prefer ints or strings.

## 18. Practice Problems

### Easy
1. **Contains Duplicate** — LeetCode 217
2. **Intersection of Two Arrays** — LeetCode 349
3. **Happy Number** — LeetCode 202
4. **Single Number** — LeetCode 136

### Medium
5. **Longest Consecutive Sequence** — LeetCode 128
6. **Set Matrix Zeroes** — LeetCode 73
7. **Find the Duplicate Number** — LeetCode 287
8. **RandomizedSet** (insert/delete/get-random O(1)) — LeetCode 380

### Hard
9. **Word Ladder** — LeetCode 126
10. **Smallest Range Covering Elements from K Lists** — LeetCode 632
11. **Suffix Structures / Substring Set Pairs** (custom) — used in string-matching interviews