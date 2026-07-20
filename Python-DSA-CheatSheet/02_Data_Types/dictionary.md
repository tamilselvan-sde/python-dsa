# Dictionary (dict)

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 02 — Data Types
> 🔗 Related: [set.md](./set.md), [../06_Collections/defaultdict.md](../06_Collections/defaultdict.md), [../06_Collections/counter.md](../06_Collections/counter.md), [../07_Algorithms/hash_map.md](../07_Algorithms/hash_map.md) · Back to [README](../README.md)

## 1. What is it?

A **dictionary** (`dict`) is a mutable, ordered (since Python 3.7), key-value mapping collection. It stores data as `{key: value}` pairs and uses a **hash table** internally to give O(1) average lookup, insert, and delete by key.

- Keys must be **hashable** (immutable types: `str`, `int`, `tuple` of hashables, `frozenset`).
- Values can be anything (mutable, immutable, nested, even functions).
- Also called a **hash map**, **hash table**, **map**, or **associative array** in other languages.

## 2. Why do we use it?

- **Fast key lookup** — O(1) average vs O(n) for a list.
- **Meaningful key names** — `user["name"]` is more readable than `user[0]`.
- **Mapping relationships** — country→capital, word→count, id→record.
- **Frequency counting** — the backbone of most "count things" problems.
- **Caches / memoization** — store computed results keyed by input.
- **Graph adjacency lists** — `{node: [neighbors]}`.
- **JSON-shaped data** — dictionaries mirror JSON objects natively.

## 3. When should I choose it?

| Need | Choose |
|------|--------|
| Lookup/insert/delete by a key | **dict** |
| Preserve insertion order + key access | **dict** (3.7+) |
| Prevent missing-key crashes | **defaultdict / `.get`** |
| Order matters pre-3.7 or reorder needed | **OrderedDict** |
| Count frequencies | **Counter** |
| Set of unique items, no values | **set** |
| Index-based sequential data | **list** |
| Immutable keys for the lifetime of the program | **MappingProxyType** |

## 4. Syntax

```python
# Empty
d1 = {}
d2 = dict()

# Literal
user = {"name": "Ada", "age": 36}

# From iterable of pairs
d3 = dict([("a", 1), ("b", 2)])

# Keyword arguments
d4 = dict(name="Ada", age=36)

# Comprehension
squares = {n: n*n for n in range(5)}

# Type-annotated (3.9+)
scores: dict[str, int] = {"math": 90}

# Union operator (3.9+) — merge
merged = {"a": 1} | {"b": 2}
```

## 5. Basic Example

```python
person = {"name": "Ada", "age": 36, "city": "London"}

print(person["name"])          # Ada
print(person.get("email", "-"))# -  (safe access)
person["age"] = 37             # update
person["email"] = "a@x.io"     # add
del person["city"]             # remove
print(len(person))             # 3
print("name" in person)        # True

for k, v in person.items():
    print(k, "->", v)
```

## 6. Step-by-Step Dry Run

Insert `"age": 36` into empty dict `d`:

```
Step 1: key = "age"        hash("age") = h
Step 2: index = h % table_size    (say table_size = 8, index = 3)
Step 3: bucket[3] = []  (empty)  →  no collision
Step 4: bucket[3] = [("age", 36)]
```

Lookup `d["age"]`:
```
Step 1: hash("age") = h
Step 2: index = h % 8 = 3
Step 3: scan bucket[3] = [("age", 36)]
Step 4: key match → return 36
```

Resize trigger (CPython):
```
When load factor (used / table_size) > 2/3 → table doubles, all entries re-hashed.
```

## 7. Built-in Methods (EXHAUSTIVE)

### `d.get(key[, default])`
- **Purpose:** Fetch value without raising `KeyError`.
- **Syntax:** `d.get(key)` or `d.get(key, default)`
- **Input:** key (hashable), optional default
- **Output:** value or default or `None`
- **Example:**
  ```python
  d = {"a": 1}
  d.get("a")        # 1
  d.get("b")        # None
  d.get("b", 0)     # 0   ← does NOT insert "b"
  ```
- **Time:** O(1) avg
- **Interview use:** Frequency counting without `if key in d` checks.
- **Common mistake:** Confusing `get` (read-only) with `setdefault` (writes if missing).
- **Shortcut:** `d.get(k, default)` == `d[k] if k in d else default`.

### `d.setdefault(key[, default])`
- **Purpose:** Return value; if key missing, **insert** `key: default` then return it.
- **Syntax:** `d.setdefault(key, default=None)`
- **Input:** key, optional default (defaults to `None`)
- **Output:** existing value or newly inserted default
- **Example:**
  ```python
  d = {}
  d.setdefault("a", []).append(1)
  d.setdefault("a", []).append(2)
  d  # {"a": [1, 2]}
  ```
- **Time:** O(1) avg
- **Interview use:** Build adjacency lists / group-by without imports.
- **Common mistake:** Using `list` as default `[]` raised at call — each call makes a fresh `[]` (mutable default recreated each call, safe in `setdefault`).
- **Shortcut:** One-liner grouping pattern.

### `d.keys()` / `d.values()` / `d.items()`
- **Purpose:** Live **view objects** of keys/values/pairs.
- **Syntax:** `d.keys() | d.values() | d.items()`
- **Input:** none
- **Output:** dict_keys / dict_values / dict_items view (not a list!)
- **Example:**
  ```python
  d = {"a": 1, "b": 2}
  list(d.keys())        # ["a", "b"]
  list(d.values())      # [1, 2]
  for k, v in d.items():
      ...
  ```
- **Time:** O(1) to create, O(n) to iterate
- **Interview use:** Iterate without materializing a list → memory-efficient.
- **Common mistake:** `d.keys()` ≠ `list(d.keys())`; views are **live** (change when dict changes).
- **Note:** In Python 3, these are dynamic views — `len()` is O(1), membership `k in d.keys()` is O(1).

### `d.update(other)`
- **Purpose:** Merge another mapping/iterable of pairs into `d` (overwrites existing keys).
- **Syntax:** `d.update(other_dict)` or `d.update(iterable)` or `d.update(**kwargs)`
- **Input:** dict / list of pairs / kwargs
- **Output:** `None` (in-place)
- **Example:**
  ```python
  d = {"a": 1}
  d.update({"a": 9, "b": 2})
  d  # {"a": 9, "b": 2}

  # 3.9+ non-mutating merge
  new = {"a": 1} | {"a": 9, "b": 2}  # {"a": 9, "b": 2}
  ```
- **Time:** O(k) for k added
- **Common mistake:** `d.update(...)` returns `None`, can't chain.

### `d.pop(key[, default])`
- **Purpose:** Remove key and return its value.
- **Syntax:** `d.pop(key)` or `d.pop(key, default)`
- **Input:** key, optional default
- **Output:** value (or default; `KeyError` if missing and no default)
- **Example:**
  ```python
  d = {"a": 1, "b": 2}
  d.pop("a")        # 1
  d.pop("z", 0)     # 0
  d                 # {"b": 2}
  ```
- **Time:** O(1) avg

### `d.popitem()`
- **Purpose:** Remove and return the **last** inserted (key, value) pair (LIFO — like a stack).
- **Syntax:** `d.popitem()`
- **Input:** none
- **Output:** (key, value) tuple; `KeyError` if empty
- **Example:**
  ```python
  d = {"a": 1, "b": 2}
  d.popitem()  # ("b", 2)
  d.popitem()  # ("a", 1)
  ```
- **Time:** O(1)
- **Interview use:** LRU-style processing; ordered processing by insertion (reverse).

### `d.clear()`
- **Purpose:** Empty the dict.
- **Syntax:** `d.clear()`
- **Output:** `None`; dict now `{}`
- **Time:** O(n) (must decref all entries)
- **Example:**
  ```python
  d = {"a": 1}; d.clear(); d  # {}
  ```

### `d.copy()`
- **Purpose:** **Shallow** copy of the dict.
- **Syntax:** `d.copy()`
- **Output:** new dict with same key→value references
- **Example:**
  ```python
  d  = {"a": [1,2]}
  d2 = d.copy()
  d2["a"].append(3)
  d["a"]            # [1,2,3]  ← shared list!
  ```
- **Time:** O(n)
- **Common mistake:** Shallow copy shares nested mutable values — use `copy.deepcopy(d)` for independent nested structures.

### `dict.fromkeys(iterable[, value])`
- **Purpose:** Build dict with keys from iterable, all initialized to same value.
- **Syntax:** `dict.fromkeys(["a","b"], 0)`
- **Input:** iterable of keys, optional default value (`None`)
- **Output:** new dict
- **Example:**
  ```python
  dict.fromkeys(["a","b"], 0)        # {"a": 0, "b": 0}
  dict.fromkeys("abc")               # {"a":None,"b":None,"c":None}
  ```
- **Time:** O(n)
- **Common mistake:** Mutable default (like `[]`) is **shared** across all keys.

### `key in d` (membership)
- **Purpose:** Test key presence.
- **Syntax:** `key in d` / `key not in d`
- **Output:** bool
- **Time:** O(1) avg
- **Example:**
  ```python
  "a" in {"a": 1}     # True
  "z" in {"a": 1}     # False
  ```
- **Common mistake:** Membership tests **keys only**, not values. `"Ada" in person` checks if `"Ada"` is a key. For values use `x in d.values()` (O(n)).

### `del d[key]`
- **Purpose:** Delete a key (statement, not expression).
- **Syntax:** `del d[key]`
- **Output:** None (raises `KeyError` if missing)
- **Time:** O(1)
- **Example:**
  ```python
  d = {"a": 1}; del d["a"]; d  # {}
  ```
- **Common mistake:** Can't chain; returns nothing. Use `pop` when you need the value.

### Other built-ins
- `len(d)` → number of items, O(1)
- `reversed(d)` → reverse-iteration view (3.8+)
- `d | other` / `d |= other` (3.9+) → merge / in-place merge
- `dict.__missing__(key)` → hook subclass to customize missing-key behavior (used by `defaultdict`).

## 8. Interview Example — Two Sum (LeetCode 1)

```python
def twoSum(nums, target):
    seen = {}                       # value -> index
    for i, num in enumerate(nums):
        need = target - num
        if need in seen:            # O(1) lookup
            return [seen[need], i]
        seen[num] = i
```

**Why dict?** We need "have I seen X?" in O(1) and to remember each number's index — a dict keyed by value is the optimal structure.

## 9. When NOT to use

- **Order sensitive + duplicate keys** → keys must be unique; use a list of tuples.
- **Memory-constrained tiny data** → dict has ~72 bytes overhead per entry; for ≤5 items a list of tuples may be cheaper.
- **Need numeric indexing** → list is the right tool.
- **Hashable-key restriction** → if keys are mutable (lists), use a list of pairs or convert to `tuple` first.
- **Heavy numeric/scientific work** → use NumPy arrays or pandas.

## 10. Common Mistakes

1. **`KeyError` on direct access** — prefer `d.get(k, default)`.
2. **Mutating while iterating** — `for k in d: del d[k]` raises `RuntimeError`. Iterate over `list(d.keys())` or use comprehension.
3. **Mutable default in `fromkeys`** — all keys share the same list object.
4. **`d.keys()` is not a list** — call `list(d.keys())` when indexing needed.
5. **Shallow copy surprises** — `d.copy()` shares nested objects.
6. **Order assumed pre-3.7** — only insertion-ordered since 3.7 (CPython 3.6 implementation detail).
7. **Using a mutable (list/set) as a key** → `TypeError: unhashable type`.
8. **Confusing `|` (dict union) with `|` (set union)** — works only on dicts 3.9+.
9. **`d.update()` returns None** — can't chain: `d.update(x)[k]` ❌.
10. **Comparing identity (`is`) instead of equality (`==`)** for value checks.

## 11. Memory Tricks

- **"KVV"** — **K**ey must be hashable, **V**alue can be anything, **V**iews are live (`keys/values/items`).
- **"Get reads, Setdefault writes"** — `get` never mutates; `setdefault` does.
- **"Pop returns, del doesn't"** — use `pop` when you need the value back.
- **"3.7 = Ordered"** — from 3.7 on, insertion order is guaranteed.
- **"Hash → Bucket → Chain"** — the three steps of every dict lookup.
- Mnemonic for ordering: "Insertion Prevails" → IP → 3.7+ keeps Insertion Persistence.

## 12. Interview Shortcuts

- **O(1) lookup answer** → use a dict as a hash map.
- **Frequency count** → `Counter(nums)` or `d.get(x, 0) + 1` loop.
- **Group anagrams/LRU** → dict of lists; pair with `OrderedDict` for LRU.
- **Memoization** → `cache = {}` keyed by argument tuple.
- **Adjacency list** → `graph.setdefault(u, []).append(v)` — one line.
- **Distinct seen-tracking** → `seen = set()` is faster than dict if you only need existence.
- **Two-sum** → dict value→index in one pass.
- **Sliding window/max substring** → dict char→last_index for jumps.
- **Top-K** → `Counter.most_common(k)` (Counter subclasses dict).
- **Python 3.9+ merge** → `{**a, **b}` or `a | b` (b wins on collisions).

## 13. Cheat Sheet Table

| Method / Op | Use |
|-------------|-----|
| `d[k]` | Get/set/replace by key (raises if missing on get) |
| `d.get(k, def)` | Safe read, no mutation |
| `d.setdefault(k, def)` | Get or insert default |
| `d.keys()` | Live view of keys |
| `d.values()` | Live view of values |
| `d.items()` | Live view of (k,v) pairs |
| `d.update(other)` | Merge in-place |
| `d \| other` (3.9+) | New merged dict |
| `d.pop(k, def)` | Remove + return value |
| `d.popitem()` | Remove + return last pair (LIFO) |
| `d.clear()` | Empty dict |
| `d.copy()` | Shallow copy |
| `dict.fromkeys(iter, v)` | Build dict from keys |
| `k in d` | Key membership O(1) |
| `del d[k]` | Delete key (statement) |
| `len(d)` | Count entries |
| `reversed(d)` | Reverse-iteration view (3.8+) |

## 14. Time Complexity Table

| Operation | Average | Worst (adversarial hash / resize) |
|-----------|---------|----------------------------------|
| `d[k]` get | O(1) | O(n) |
| `d[k] = v` set | O(1) | O(n) |
| `del d[k]` | O(1) | O(n) |
| `d.get`, `setdefault`, `pop` | O(1) | O(n) |
| `popitem` | O(1) | O(1) |
| `k in d` | O(1) | O(n) |
| `len(d)` | O(1) | O(1) |
| `d.copy()` | O(n) | O(n) |
| `d.update(other)` | O(k) | O(k+n) |
| `d.keys/values/items` create | O(1) | O(1) |
| iterate all items | O(n) | O(n) |
| `list in` (lookup) | **O(n)** | O(n) |
| `dict in` (lookup) | **O(1)** | O(n) |

> **Key insight:** dict lookup O(1) vs list lookup O(n). For ≥ ~10 elements and frequent lookups, dict dominates.

## 15. Visual Diagram (ASCII)

### Hash table buckets with chaining

```
        hash(key)  ──mod──>  bucket index
                                 │
   ┌──hash table (size 8)──┐    │
   │  0  │ []               │    ▼
   │  1  │ []               │   index = h % 8
   │  2  │ [ ("name","Ada") ]│
   │  3  │ [ ("age", 36)  ] │   ← no collision
   │  4  │ []               │
   │  5  │ [ ("b",1),      ]│   ← collision! two keys, same bucket
   │     │   ("r",9)        │     resolved by linked list (CHAINING)
   │  6  │ []               │
   │  7  │ [ ("x", 99)     ]│
   └─────┴──────────────────┘
```

### Resize (rehashing) flow
```
load_factor = used / capacity
   used=5, capacity=8 → 5/8 = 0.62   (under 2/3 threshold, OK)
   used=6, capacity=8 → 0.75 > 0.66   → RESIZE

   old capacity (8)  ──x2──>  new capacity (16)
   re-hash every key, re-bucket each entry
   amortized O(1) inserts over a sequence of n inserts = O(n) total
```

### Lookup flow
```
                 ┌──────────────┐
   d["age"] ───▶ │ hash("age")  │──── h = 42 (example)
                 └──────────────┘
                         │
                         ▼
                 ┌──────────────────┐
                 │ index = h % 8 = 2│
                 └──────────────────┘
                         │
                         ▼
              ┌─────────────────┐
              │ scan bucket[2]: │
              │  ("age", 36)?   │─key match─▶ return 36 ✓
              └─────────────────┘
                          │ no match (collision case)
                          ▼
                   move to next node in chain…
```

## 16. Beginner Notes

> **Remember:**
> - Dict = `{}` with `:` between key and value.
> - Keys are **unique** & **hashable**; values can repeat or be anything.
> - Insertion order is preserved since Python 3.7.
> - Use `d.get(k, default)` to avoid `KeyError`.
> - Views (`keys`/`values`/`items`) are **live**, not copies — change the dict, the view reflects it.
> - `==` compares contents; `is` compares identity.
> - Nested dicts are just dicts whose values are also dicts — access with chained `[]`.

## 17. FAANG Tips

1. **Default to dict when you need O(1) lookup by a key** — the #1 interview move.
2. **For "count/group" problems**, reach for `Counter` or `defaultdict` first.
3. **LRU cache** — `OrderedDict` with `move_to_end` + `popitem(last=False)`.
4. **Two-pointer + dict** = classic for subarray/substring problems (`{char: last_seen_idx}`).
5. **State your dict's key/value semantics out loud** before coding — interviewers love clarity.
6. **Beware hash collisions on adversarial input** — Python 3.3+ randomizes hashes by default per process for `str`/`bytes`, so the worst case is rare in practice but theoretically O(n). Mention this when asked about DoS.
7. **Memoization dict with tuple keys** is the go-to for recursion → DP transitions.
8. **`d.setdefault(k, []).append(v)`** is a one-liner group-by when you can't import `defaultdict`.
9. **Don't iterate and mutate**: collect keys to delete first or rebuild via comprehension.
10. **For sorted output**, sort `d.items()` by `key=lambda x: x[1]` — never assume dict order equals sorted order.

## 18. Practice Problems

### Easy
1. **Two Sum** — LeetCode 1
2. **Roman to Integer** — LeetCode 13
3. **Majority Element** — LeetCode 169
4. **Contains Duplicate II** — LeetCode 219

### Medium
5. **Group Anagrams** — LeetCode 49
6. **Longest Substring Without Repeating Characters** — LeetCode 3
7. **Copy List with Random Pointer** — LeetCode 138
8. **Top K Frequent Elements** — LeetCode 347

### Hard
9. **LRU Cache** — LeetCode 146
10. **Substring with Concatenation of All Words** — LeetCode 30
11. **Smallest Range Covering Elements from K Lists** — LeetCode 632