# Hash Map in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 07 — Algorithms
> 🔗 Related: [stack.md](./stack.md) · [queue.md](./queue.md) · [linked_list.md](./linked_list.md) · [trees.md](./trees.md) · [graph.md](./graph.md)
> Data: [dict.md](../02_Data_Types/dict.md) · [counter.md](../06_Collections/counter.md) · [deque.md](../06_Collections/deque.md)
> Back to [README](../README.md)

---

## 1. What is it?

A **hash map** (Python: `dict`) stores **key → value** pairs. A *hash function* converts each key into an integer index into an internal array of "buckets". Insert, lookup, and delete all run in **average O(1)** instead of O(n) scanning.

```
key  →  hash(key)  →  index = hash % len(buckets)  →  bucket[index]  →  entry (key, value)
```

Two keys can land in the same bucket — a **collision**. CPython resolves collisions with **open addressing** (probing sequences). When the load factor exceeds ~2/3, the table is **rehashed** to a larger size to keep operations fast.

Since **Python 3.7**, `dict` **preserves insertion order** as a guaranteed language feature. `set` and the third-party `defaultdict` from `collections` are also hash-based.

**What problem it solves:** Fast lookup / membership / frequency tracking without scanning the whole collection.

**Real-world analogy:** A coat-check room. You hand a ticket (key), the attendant walks to a numbered hook (index) and hands back your coat (value) — no need to inspect every hook.

---

## 2. Why do we use it?

- **O(1) average** insert / lookup / delete — the backbone of most frequency, cache, and index problems.
- **Insertion order preserved** (3.7+), so it doubles as an ordered map.
- **`in` test on keys** is O(1) (vs O(n) for a list).
- Foundation for **Counter**, **defaultdict**, **set**, and most **two-sum / grouping / caching** patterns.
- Python's dict is one of the most heavily optimized hash maps in any language — written in C, integrated into the interpreter core.

---

## 3. When should I choose it? — Decision Table

| Situation                                              | Best choice                              | Notes                                  |
|--------------------------------------------------------|------------------------------------------|----------------------------------------|
| Count frequencies of items                             | `collections.Counter`                    | Hash map specialization                |
| Group items by a key                                   | `dict[k].append(v)` / `defaultdict(list)`| Group Anagrams pattern                 |
| Two elements sum to target                             | `dict` of `value → index`                | One-pass Two Sum (1)                   |
| Look-up "have I seen x before?"                        | `seen = set()`                           | O(1) membership                        |
| Cache the most recent / least recent                   | `dict` + doubly linked list              | LRU Cache (146)                        |
| Ordered keys, range queries on keys                    | **not dict** — sorted list / tree        | Bisection needed                       |
| Keys are 1..n dense integers                           | plain `list`                             | Faster + no hashing                    |
| Keys must stay sorted                                  | `sortedcontainers.SortedDict` / tree map | dict doesn't sort                      |

---

## 4. Syntax

```python
# Creation
d = {}                       # empty
d = {"a": 1, "b": 2}         # literal
d = dict(a=1, b=2)           # constructor
d = dict([("a", 1), ("b", 2)])

# Access
v = d["a"]                   # KeyError if missing
v = d.get("a")               # None if missing
v = d.get("a", 0)            # default if missing

# Mutation
d["c"] = 3                   # insert / update
del d["a"]                   # delete (KeyError if missing)
d.pop("a")                   # delete + return value
d.pop("a", default)          # safe delete
d.update({"x": 9})           # bulk update

# Iteration
for k in d: ...              # keys
for k, v in d.items(): ...   # pairs
list(d.keys()); list(d.values())

# Membership
"a" in d                     # O(1)

# Comprehension
sq = {x: x*x for x in range(5)}

# Setdefault (insert with default, return current)
d.setdefault("z", []).append(1)

# Views are live
d = {"a": 1}; v = d.keys(); d["b"] = 2
# v now reflects {"a", "b"}

# Unhashable keys: use tuple or frozenset (no list/dict!)
# d[ [1,2] ] = 5  ❌ TypeError: unhashable type: 'list'
```

---

## 5. Basic Example

```python
from collections import Counter

text = "abracadabra"
freq = Counter(text)
print(freq)            # Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
print(freq.most_common(2))  # [('a', 5), ('b', 2)]
print(freq['z'])            # 0  (Counter returns 0 not KeyError for missing)

# Two Sum (one-pass)
def two_sum(nums, target):
    seen = {}              # value -> index
    for i, x in enumerate(nums):
        if (need := target - x) in seen:
            return [seen[need], i]
        seen[x] = i
    return []

print(two_sum([2, 7, 11, 15], 9))   # [0, 1]
```

**Output:**
```
Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
[('a', 5), ('b', 2)]
0
[0, 1]
```

---

## 6. Step-by-Step Dry Run

**Two Sum** `nums = [2, 7, 11, 15]`, `target = 9`

```
i=0  x=2   need=7  seen={  }           → miss, seen={2:0}
i=1  x=7   need=2  seen={2:0}          → HIT  seen[2]=0  → return [0, 1]
```

**Group Anagrams** `strs = ["eat","tea","tan","ate","nat","bat"]`

```
word="eat" key='aet'  groups['aet'] = ["eat"]
word="tea" key='aet'  groups['aet'] = ["eat","tea"]
word="tan" key='ant'  groups['ant'] = ["tan"]
word="ate" key='aet'  groups['aet'] = ["eat","tea","ate"]
word="nat" key='ant'  groups['ant'] = ["tan","nat"]
word="bat" key='abt'  groups['abt'] = ["bat"]
result = [["eat","tea","ate"],["tan","nat"],["bat"]]
```

**Hash table bucket walk-through** for inserting keys `"a", "b", "c"` into an 8-bucket table using `hash(key) % 8`:

```
buckets:  [ _ _ _ _ _ _ _ _ ]
hash('a')= 1 %8 =1  → bucket[1]="a"
buckets:  [ _ a _ _ _ _ _ _ ]
hash('b')= 2 %8 =2  → bucket[2]="b"
hash('c')= 9 %8 =1  → COLLISION @1  → probe bucket[2] (occupied) → bucket[3] empty
buckets:  [ _ a b c _ _ _ _ ]
```

---

## 7. Built-in Methods

### `dict` — core operations

| Method              | Purpose                                | Syntax            | Example              | Complexity        | Interview use                    | Mistakes                                  | Shortcut                            |
|---------------------|----------------------------------------|-------------------|----------------------|-------------------|----------------------------------|-------------------------------------------|-------------------------------------|
| `d[k]`              | get/set value by key                   | `d[k]`            | `d["a"]`             | O(1) avg           | everywhere                      | raises `KeyError` if missing              | use `get` instead if missing possible |
| `d.get(k, default)` | safe read                              | `d.get(k, None)`  | `dget("z", 0) = 0`   | O(1) avg           | counter patterns                | forgetting default                        | `d.get(k, 0) += 1` needs setdefault   |
| `d.setdefault(k, d)`| insert+default, return current          | `d.setdefault(k, [])` | group buckets      | O(1) avg           | defaultdict alternative         | confused with `get` (read-only)           | `d.setdefault(k, []).append(v)`       |
| `d[k] = v`          | insert / overwrite                     | -                 | -                    | O(1) avg           | building maps                   | overwrites silently                       | -                                   |
| `del d[k]`          | delete                                 | `del d[k]`        | -                    | O(1) avg           | pruning                         | KeyError if missing                       | use `pop(k, None)`                  |
| `d.pop(k, default)` | delete + return                        | `d.pop("a", None)`| -                    | O(1) avg           | caching / LRU                   | default arg is *evaluated eagerly*        | -                                   |
| `d.update(other)`   | bulk merge                             | `d.update({...})` | -                    | O(m)               | merging entities                | mutates in place                          | `{**a, **b}` non-mutating merge       |
| `k in d`            | membership                             | -                 | -                    | O(1) avg           | seen-set checks                 | `in d.keys()` is slower & redundant       | -                                   |
| `d.items()`         | (k, v) pairs view                      | `for k,v in d.items()`  | -              | O(n)               | iterate together                | modifying during iteration breaks         | -                                   |
| `{k:v for ...}`     | comprehension                          | -                 | -                    | O(n)               | transform maps                  | duplicate keys collapse silently          | -                                   |

### `collections.Counter`

```python
from collections import Counter
c = Counter("abracadabra")
c.most_common(2)     # [('a', 5), ('b', 2)]
c.total()            # 11  (3.10+)
c += Counter(a=2)    # arithmetic works
c2 = Counter(a=4, b=2)
c - c2 ; c | c2 ; c & c2  # subset / union / intersection
```

### `collections.defaultdict`

```python
from collections import defaultdict
g = defaultdict(list)
g["a"].append(1)         # no manual initialization
counts = defaultdict(int)
counts["x"] += 1
```

> **`defaultdict` vs `dict.setdefault`**: `defaultdict` calls the factory *only when needed and silently*; `setdefault` evaluates the default eagerly *every call*. For `[]` / `0` it doesn't matter, but it matters for expensive factories.

---

## 8. Interview Example

### LC 49 — Group Anagrams (Medium)

```python
from collections import defaultdict

def groupAnagrams(strs):
    groups = defaultdict(list)
    for w in strs:
        key = "".join(sorted(w))     # O(k log k) signature
        groups[key].append(w)
    return list(groups.values())

print(groupAnagrams(["eat","tea","tan","ate","nat","bat"]))
# [['eat','tea','ate'], ['tan','nat'], ['bat']]
```

### LC 1 — Two Sum (Easy)

```python
def twoSum(nums, target):
    seen = {}
    for i, x in enumerate(nums):
        if target - x in seen:
            return [seen[target - x], i]
        seen[x] = i
```

### LC 560 — Subarray Sum Equals K (Medium)

Prefix-sum + hashmap trick: number of subarrays ending at `i` with sum `k` equals `count[curr_sum - k]`.

```python
from collections import defaultdict

def subarraySum(nums, k):
    count = defaultdict(int, {0: 1})
    curr = res = 0
    for x in nums:
        curr += x
        res += count[curr - k]
        count[curr] += 1
    return res
```

---

## 9. When NOT to use

- **Order statistic queries** ("k-th smallest key", "keys in range [lo, hi]") → use `sortedcontainers.SortedDict` or a balanced BST; `dict` is unsorted.
- **Keys are dense, small-range integers** — a `list[None] * (n+1)` is faster (no hashing cost) and uses less per-key memory.
- **Storing huge sparse bitsets** → use a real `set` or bit-array; dict is overkill.
- **Range-scanning is the dominant operation** rather than point lookup.
- **Keys are unhashable** (lists, dicts, sets) — convert to `tuple`/`frozenset` first; otherwise use a list of pairs.
- **Worst-case latency matters** (e.g., realtime systems) — pathological hash collisions can degrade dict to O(n). Most apps don't care; high-frequency trading and embedded do.

---

## 10. Common Mistakes

1. **`d[k]` on a missing key** → `KeyError` . Use `d.get(k)` or `defaultdict`.
2. **`if k in d` after `d[k]` access** — does the lookup twice; store the value or use `get`.
3. **Mutating dict during iteration** → `RuntimeError: dictionary changed size`. Build a list of changes first.
4. **Unhashable key attempt**: `d[ [1,2] ] = 5` → `TypeError`. Use `tuple` instead.
5. **Confusing `defaultdict` default with `get` default**: `defaultdict(int)['z']` returns `0` *and inserts* the key; `get('z', 0)` returns `0` *without* inserting.
6. **Reading `.keys()` view as a list** — `d.keys()` is a live view; materialize with `list(d.keys())` if you mutate.
7. **Assuming default arguments are re-evaluated** (Python's notorious gotcha):

   ```python
   def f(items={}):           # ❌ one shared dict across calls
       items["x"]=1
   ```
   → Use `def f(items=None): items = items or {}`.
8. **Order dependence before 3.7**: don't rely on insertion order in legacy code.
9. **Using a list as a key by accident** — clearly list/dict keys fail loudly, but a `datetime` works while a `numpy.int64` may not be hashable-compatible.

---

## 11. Memory Tricks

- **"Ticket → hook → coat"** = key → index → value.
- **Diamond rule for collisions**: Python keeps two collisions rare by **rehashing at 2/3 load**.
- **`Counter` is a `dict` subclass** → every dict method works; it just adds `most_common`, arithmetic, and a `__missing__` returning `0`.
- **`set` is a "valueless dict"**: think `set(x)` ≈ `dict.fromkeys(x, True)`.
- **`defaultdict(list)` ≈ "auto-bucketing dict"**: the one-liner every grouping problem wants.

<details><summary>Mnemonic for `get` vs <code>setdefault</code></summary>

- **g**et → **g**lance only (read)
- **s**etdefault → **s**et-if-absent, return current
</details>

---

## 12. Interview Shortcuts

- **Two-sum one-pass**: maintain `value → index`, check `target - x in seen` *before* inserting (avoids matching the same element twice).
- **Frequency counting**: `Counter(iter)` is your opener — one-liner, supports arithmetic.
- **Grouping**: `defaultdict(list)` + `groups[key].append(item)`.
- **Subarray sum tricks**: `prefix[0] = 1` initialized to handle "subarray from index 0".
- **Pair presence check**: prefer `set(nums)` to `if x in nums` (O(n) → O(1)).
- **`set` intersection / difference**: `a & b`, `a - b` use hash tables internally — O(min(m, n)) average.
- **LRU Cache recipe**: `dict` for O(1) lookup + `OrderedDict.move_to_end` (or doubly linked list by hand). See `LinkedHashMap`-style interview patterns.
- **Frozen keys**: hash `tuple(sorted(freq.items()))` for a quick composite signature.

---

## 13. Cheat Sheet Table

| Operation                       | Syntax                          | Avg time | Notes                              |
|---------------------------------|---------------------------------|----------|-------------------------------------|
| Build from iterable             | `dict(pairs)`                   | O(n)     | -                                   |
| Insert / update                 | `d[k] = v`                      | O(1)     | overwrites silently                 |
| Lookup                          | `d.get(k, def)`                 | O(1)     | no KeyError                         |
| Membership                      | `k in d`                        | O(1)     | faster than keys() check           |
| Delete                          | `d.pop(k, def)`                 | O(1)     | safe pop                            |
| Iteration                       | `for k in d` / `.items()`       | O(n)     | insertion-ordered (3.7+)            |
| Length                          | `len(d)`                        | O(1)     | -                                   |
| Merge                           | `{**a, **b}`, `a\|b` (3.9+)     | O(len a + len b) | rightmost wins                  |
| Counter most_common(k)          | `c.most_common(k)`              | O(n log k)| heap-based                          |
| Sort by value                   | `sorted(d.items(), key=lambda x: -x[1])` | O(n log n) | post-process |

---

## 14. Time Complexity Table

| Operation                          | Average | Worst  | Notes                           |
|------------------------------------|---------|--------|---------------------------------|
| `d[k] = v`                         | O(1)    | O(n)   | worst = pathological collisions |
| `d[k]` / `get` / `in`              | O(1)    | O(n)   | same                            |
| `del d[k]` / `pop`                 | O(1)    | O(n)   | -                               |
| `len(d)`                           | O(1)    | O(1)   | stored on dict object           |
| `iter` over keys/values/items      | O(n)    | O(n)   | -                               |
| `update(other)`                    | O(m)    | O(m)   | m = len(other)                  |
| `Counter(iterable)`                | O(n)    | O(n)   | -                               |
| `Counter.most_common()`            | O(n log n) | O(n log n) | full sort                  |
| `Counter.most_common(k)`           | O(n log k) | O(n log n) | heap-based                 |

**Space:** O(n) for n entries; load factor ~2/3 keeps memory lower than many maps, but expect 50–70 bytes per (key, value) entry including the entry struct, the key, and the value pointers.

---

## 15. Visual Diagram (ASCII)

### Hash table buckets

```
            +----------+----------+
buckets[0]  |  None    |  --      |
            +----------+----------+
buckets[1]  |  ("a",1) |  ──next──┼──>  ("c",3)     # collision (separate chaining
            +----------+----------+                  sketch; CPython uses open addressing)
buckets[2]  |  ("b",2) |  --      |
            +----------+----------+
buckets[3]  |  None    |          |
            +----------+----------+
              ...        ...
buckets[7]  |  None    |          |
            +----------+----------+

key → hash(key) → idx = hash % len(buckets) → walk bucket chain
```

### Frequency map build flow

```
                +-----------+   for each item x:
   "items" ───> |  iterate |───────────────────┐
                +-----------+                   │
                                                ▼
                                    +-----------------------+
                                    | seen[x] = seen.get(x,0)+1 |
                                    +-----------------------+
                                                │
                                                ▼
                                       output: Counter / dict
```

### Two Sum flow

```
 nums = [2, 7, 11, 15],  target = 9
   i=0 x=2  need=7  seen={}        → miss; seen={2:0}
   i=1 x=7  need=2  seen={2:0}     → HIT!  return [0, 1]
```

---

## 16. Beginner Notes

> **Remember:**
> - Hash map = `dict` in Python — **order-preserving** since 3.7.
> - **Average O(1)** for the three core ops; **worst O(n)** when collisions pile up.
> - Keys must be **hashable** (immutable + `__hash__`): `int`, `str`, `tuple`, `frozenset` ✓ ; `list`, `dict`, `set` ✗.
> - Use **`get`** to avoid `KeyError`, **`defaultdict`** to auto-initialize buckets, **`Counter`** for tallies.
> - Insertion-order preservation lets you treat a dict as an **LRU-ish structure** with `move_to_end` on `OrderedDict`.
> - `d.update()` mutates; **`{**a, **b}`** does not — pick based on intent.

---

## 17. FAANG Tips

- **Frequency counting is the #1 hash map pattern**; reach for `Counter` before any manual loop.
- **Two-pointer pair sums** → `dict` of `value → index`, **check before insert** to avoid double-counting.
- **Prefix-sum + hashmap** turns "count subarrays with sum k" from O(n²) to **O(n)** (LC 560).
- **Cycle / repeat detection** (Linked List Cycle II, Happy Number) can use a dict as a `seen` set; a slow/fast pointer beats it on memory though — see [linked_list.md](./linked_list.md).
- **String signatures**: `tuple(sorted(c.replace(" ","").lower()))` or `Counter` tuples make great grouping keys.
- **LRU**: dict for O(1) lookup + doubly linked list (or `OrderedDict`) for O(1) MRU move. Sketch the structure *before* code.
- **`__hash__` matters**: dict keys rely on hash + `__eq__` equality;`numpy.int64` and `bool`/`int` collisions can bite. Cast to `int`.
- **Keep iteration safe** — never `del` from a dict while iterating it; collect keys or use a comprehension to build a fresh dict.
- **Leverage `set` arithmetic**: `a & b`, `a - b`, `a ^ b` are hash-based and almost-never beatable for the matching problems.
- **Ask the interviewer**: "Are keys always hashable? expected cardinality?" before designing around a dict — guides collision / capacity decisions.

---

## 18. Practice Problems

### Easy
- **LC 1** — Two Sum
- **LC 217** — Contains Duplicate
- **LC 383** — Ransom Note
- **LC 205** — Isomorphic Strings
- **LC 290** — Word Pattern

### Medium
- **LC 49** — Group Anagrams
- **LC 128** — Longest Consecutive Sequence
- **LC 560** — Subarray Sum Equals K
- **LC 347** — Top K Frequent Elements
- **LC 739 ─ (next greater see stack.md)** daily freq hashmap support

### Hard
- **LC 146** — LRU Cache (hashmap + doubly linked list)
- **LC 432** — All O`one Data Structure
- **LC 76** — Minimum Window Substring (sliding window + freq maps)