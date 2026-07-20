# collections.defaultdict

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 06 — Collections
> 🔗 Related: [counter.md](./counter.md) · [deque.md](./deque.md) · [heapq.md](./heapq.md) · [../02_Data_Types/dictionary.md](../02_Data_Types/dictionary.md)
> Back to [README](../README.md)

---

## 1. What is it?

`collections.defaultdict` is a **dict subclass** that **automatically creates missing keys** with a default value produced by a `default_factory` callable.

The `default_factory` is invoked (with no arguments) every time a missing key is accessed. After the factory runs, the value is inserted into the dict and then returned — so subsequent lookups are O(1) without any extra setup.

```python
from collections import defaultdict
d = defaultdict(list)
d['a'].append(1)        # no KeyError; d['a'] auto-creates []
print(d['a'])           # [1]
```

The factory can be any zero-arg callable: `int`, `list`, `set`, `dict`, `tuple`, a lambda, or a custom class constructor.

---

## 2. Why do we use it?

To **eliminate the boilerplate** of initializing missing keys before accessing them.

**Without `defaultdict`:**

```python
d = {}
for u, v in edges:
    if u not in d:
        d[u] = []
    d[u].append(v)
```

**With `defaultdict(list)`:**

```python
d = defaultdict(list)
for u, v in edges:
    d[u].append(v)
```

Three lines collapsed to one. Crucial for:
- **Graph adjacency lists** (BFS, DFS, Dijkstra).
- **Grouping / partitioning** items by a key.
- **Accumulator dicts** (`defaultdict(int)`).
- **Multi-map** structures (one key → many values).

---

## 3. When should I choose it?

| Situation                                          | Best tool                  |
|----------------------------------------------------|----------------------------|
| Need a dict + counting integers                    | `Counter` (or `defaultdict(int)`) |
| Need a dict mapping key → list of values           | ✅ `defaultdict(list)`      |
| Need a dict mapping key → set of values            | ✅ `defaultdict(set)`       |
| Need a dict that returns 0 for missing keys        | `defaultdict(int)` or `Counter` |
| Need frequency + top-k                             | `Counter` ([counter.md](./counter.md)) |
| Need FIFO order with O(1) both ends                | `deque` ([deque.md](./deque.md)) |
| Need k-th largest / partial sort                    | `heapq` ([heapq.md](./heapq.md)) |
| Need a plain dict with no default behavior         | `dict`                     |
| Need ordered unique with insertion order          | `dict` (3.7+)              |

**Decision flow:**

```
   Access a missing key?
           │
   ┌───────┴────────┐
   need default value?  YES → defaultdict(factory)
   └────────────────┘
   factory = int    for counters
   factory = list   for adjacency / grouping
   factory = set    for unique grouping
   factory = dict   for nested dicts (rare)
   factory = lambda: X  for non-trivial defaults
```

---

## 4. Syntax

```python
from collections import defaultdict

# Constructor
defaultdict(default_factory=None, /, [*, ...dict kwargs])
defaultdict(list)                  # missing key → []
defaultdict(int)                   # missing key → 0
defaultdict(set)                   # missing key → set()
defaultdict(lambda: 10)           # missing key → 10
defaultdict(None)                  # behaves like plain dict (KeyError on missing)

# Nested
defaultdict(lambda: defaultdict(int))    # 2D counters
defaultdict(lambda: defaultdict(list))   # 2D grouping

# Attributes
d.default_factory                 # callable or None
d.default_factory = None          # turn off defaulting after the fact
d.default_factory = list          # re-enable

# Reading works like dict
d['x']                            # auto-create if missing
d.get('x', 'default')             # 'default' (NEVER auto-creates)
'x' in d                           # False if not present (lazy!)
d.keys(), d.values(), d.items()

# Mutation like dict
del d['x']
d.setdefault('x', []).append(1)
d.update(other)
d.pop('x')
d.clear()
```

---

## 5. Basic Example

```python
from collections import defaultdict

# Group words by their first letter
words = ['ant', 'apple', 'bat', 'bear', 'cat']
groups = defaultdict(list)
for w in words:
    groups[w[0]].append(w)

print(dict(groups))
# {'a': ['ant', 'apple'], 'b': ['bat', 'bear'], 'c': ['cat']}
```

**Counting:**

```python
votes = ['a','b','a','c','a','b']
tally = defaultdict(int)
for v in votes:
    tally[v] += 1
print(dict(tally))      # {'a': 3, 'b': 2, 'c': 1}
```

---

## 6. Step-by-Step Dry Run

Input edges: `[(1,2),(1,3),(2,3),(3,1)]` with `d = defaultdict(list)`.

| Step | Edge   | Access        | Factory called? | State after                          |
|------|--------|---------------|----------------|--------------------------------------|
| 1    | (1,2)  | `d[1].append(2)`  | yes         | `{1: [2]}`                           |
| 2    | (1,3)  | `d[1].append(3)`  | no          | `{1: [2,3]}`                         |
| 3    | (2,3)  | `d[2].append(3)`  | yes         | `{1:[2,3],2:[3]}`                    |
| 4    | (3,1)  | `d[3].append(1)`  | yes         | `{1:[2,3],2:[3],3:[1]}`              |

**Note:** `d.get(99)` returns `None` and **does not** create the key. Only `d[99]` (subscript) triggers the factory.

---

## 7. Built-in Methods   (EXHAUSTIVE)

### `defaultdict(default_factory=None)`
- **Purpose:** Construct a dict that calls `default_factory()` for missing keys.
- **Syntax:** `defaultdict(factory[, **kwargs])`
- **Input:** zero-arg callable
- **Output:** new defaultdict
- **Example:**
  ```python
  d = defaultdict(lambda: 'NA')
  d['missing']              # 'NA'  (factory runs, key inserted)
  ```
- **Time:** O(1) per missing-key access (plus factory cost).
- **Interview use:** adjacency lists, grouping, multi-maps.
- **Common mistake:** passing `list()` (with parens) instead of `list` — you'd insert the SAME shared list everywhere. Always pass the callable itself.
- **Shortcut:** `defaultdict(int)` is the cheapest counter; `defaultdict(list)` is the cheapest grouping.

### `.default_factory` (attribute)
- **Purpose:** Get/set/remove the factory callable. Setting to `None` makes it behave like a plain dict.
- **Syntax:** `d.default_factory = list` / `None`
- **Example:**
  ```python
  d = defaultdict(list)
  d.default_factory = int          # switch behavior mid-stream
  d['x']                            # now 0
  ```
- **Time:** O(1).
- **Interview use:** switch from lazy-create to KeyError-strict after setup phase.

### `.setdefault(key, default)` (inherited from dict)
- **Purpose:** Insert key with `default` if missing, **then** return its value. Bypasses the factory.
- **Syntax:** `d.setdefault(k, default)`
- **Example:**
  ```python
  d = defaultdict(list)
  d.setdefault('x', []).append(1)   # works; factory not called
  ```
- **Time:** O(1) average.
- **Common mistake:** calling `setdefault` on a defaultdict is usually redundant — just index it. Use setdefault only when the default differs from the factory's default.
- **Shortcut:** `setdefault` is the plain-dict equivalent of `defaultdict`.

### Mutating methods (inherited from dict): `.pop`, `.update`, `.popitem`, `.clear`, `.copy`, `del`
- All behave as in `dict`; mutation never triggers the factory.
- `.copy()` returns a plain **dict**, not a defaultdict!  You lose default behavior.

### Lookup methods: `d[k]`, `d.get(k, default)`, `k in d`, `d.keys/values/items`
- Only `d[k]` (subscript) triggers the factory. `get`, `in`, `keys` do not.
- **Common mistake:** expecting `d.get('missing')` to insert the key — it does **not**.

---

## 8. Interview Example

**Adjacency list + BFS** (graph basics):

```python
from collections import defaultdict, deque

def build_graph(edges):
    g = defaultdict(list)
    for u, v in edges:
        g[u].append(v)
        g[v].append(u)         # if undirected
    return g

def bfs(g, src):
    seen, q = {src}, deque([src])
    order = []
    while q:
        n = q.popleft()
        order.append(n)
        for nb in g[n]:
            if nb not in seen:
                seen.add(nb)
                q.append(nb)
    return order
```

**Group Anagrams (#49)** using `defaultdict(list)`:

```python
from collections import defaultdict
def groupAnagrams(strs):
    groups = defaultdict(list)
    for s in strs:
        key = ''.join(sorted(s))
        groups[key].append(s)
    return list(groups.values())
```

---

## 9. When NOT to use

- You want *counting* with `.most_common`/arithmetic — use `Counter` ([counter.md](./counter.md)).
- You want defaults that depend on the key (factory takes the key) — `defaultdict` can't; write a custom dict subclass with `__missing__`.
- You need a **zero-mutation read** like `d.get(k)` to NOT insert — `defaultdict[k]` does insert. Use plain `dict` + `.get`.
- You need thread-safe creation — `defaultdict` has no lock.
- You need immutable / hashable structures — defaultdict is mutable.

---

## 10. Common Mistakes

| Mistake                                            | Fix                                                  |
|----------------------------------------------------|------------------------------------------------------|
| `defaultdict(list())` — passing instance, not class | pass `list` (the class).                            |
| Thinking `d[k]` lookup doesn't insert               | it DOES insert; check `k in d` first if you don't want side effects. |
| `d.get(k)` to trigger factory                       | it never does; subscript does.                       |
| Nested defaultdicts getting tangled                | prefer `defaultdict(lambda: defaultdict(int))` once. |
| `.copy()` losing default behavior                   | rebuild with `defaultdict(factory, old.copy())`.      |
| Forgetting `int` factory starts at 0 (not 1)        | fine for counters; not for ID assignment.            |
| Comparing defaultdict vs dict with `==`             | works because defaultdict IS a dict; equal if same key/vals. |
| Using `setdefault` redundantly on a defaultdict     | just index it.                                        |
| Mutating the factory's shared default                | dangerous with non-callable defaults — that's why factory must be callable. |
| Setting `default_factory = None` and forgetting it  | subsequent missing keys raise KeyError.              |

---

## 11. Memory Tricks

- **defaultdict = "dict with a desk drawer that fills itself."**
- The **factory** is the *thing you stamp out* copies of: `list` stamps out `[]`, `int` stamps out `0`, `set` stamps out `set()`.
- **`int` → counter** (zero start).
- **`list` → adjacency list / grouping**.
- **`set` → unique grouping**.
- Picture a vending machine: every time you ask for a snack (key) that's missing, the machine **makes** a fresh one and stores it.
- Turning the factory off (set to `None`) = closing the vending machine.

ASCII of lazy value creation:

```
       key 'x'  ── not in d?
                       │
                       ▼
              ┌─────────────────┐
              │  default_factory() │   ← list()  →  []
              └─────────────────┘
                       │
                       ▼
                d['x'] = []
                       │
                       ▼
              return that same []
                       │
                       ▼
        caller can append/extend/etc.
```

---

## 12. Interview Shortcuts

```python
# Adjacency list
g = defaultdict(list)
for u, v in edges: g[u].append(v)

# Counter that integrates naturally
tally = defaultdict(int)
for x in stream: tally[x] += 1

# Group items
groups = defaultdict(list)
for item in items: groups[key(item)].append(item)

# Unique grouping
groups = defaultdict(set)

# Bidirectional / weighted graph
g = defaultdict(lambda: defaultdict(int))   # g[u][v] = weight

# 2D frequency grid (infinite grid)
grid = defaultdict(lambda: defaultdict(int))

# Disable factory after init
g.default_factory = None
```

---

## 13. Cheat Sheet Table

| Operation                          | Returns   | Mutates? | Notes                                  |
|------------------------------------|-----------|----------|----------------------------------------|
| `defaultdict(factory)`             | dict      | —        | constructor                            |
| `d[k]` (missing)                   | new value | yes      | calls factory, inserts, returns        |
| `d[k]` (present)                   | value     | no       | normal dict get                        |
| `d.get(k[, default])`              | value     | **no**   | never inserts                          |
| `k in d`                           | bool      | **no**   | never inserts                          |
| `d.setdefault(k, default)`         | value     | maybe    | bypasses factory                       |
| `d.default_factory = None`         | None      | yes      | disables auto-create                   |
| `del d[k]` / `d.pop(k)`            | value     | yes      | removes key                            |
| `d.copy()`                         | dict      | no       | **plain dict** — factory lost           |
| `d.update(other)`                  | None      | yes      | merge another mapping                  |
| Arithmetic                                         | —        | —        | NOT supported like Counter             |

---

## 14. Time Complexity Table

| Operation                | Time    | Notes                                |
|--------------------------|---------|--------------------------------------|
| `d[k]` (present)         | O(1)    |                                      |
| `d[k]` (missing)         | O(1)+F  | F = cost of factory (`list()` is O(1))|
| `k in d` / `.get`        | O(1)    |                                      |
| `.setdefault(k, v)`      | O(1)    |                                      |
| `del` / `.pop`           | O(1)    |                                      |
| `.update(it)`            | O(len it)|                                     |
| `.copy()`                | O(n)    |                                      |
| Iteration                | O(n)    |                                      |

---

## 15. Visual Diagram (ASCII)

```
   defaultdict(list)
            │
   ┌────────┴──────────┐
   │   Internal dict    │  {'a':[1,2,3]}
   │   (real storage)   │  {'b':[7]}
   └────────┬──────────┘
            │  on miss
            ▼
       ┌──────────────┐
       │   factory()   │  → list()  →  []
       └──────────────┘
            │
            ▼
       inserted into dict at key
            │
            ▼
   returned to caller → caller appends

         Lazy creation flow:

       ┌─────────────┐         ┌─────────────┐
       │  d['new']    │ ──miss─▶│  list()    │
       └─────────────┘         └─────────────┘
                                       │
            ┌──────────────────────────┘
            ▼
        dict['new'] = []
            │
            ▼
        return []
            │
            ▼
       .append(item)  ← caller now mutates the stored list
```

---

## 16. Beginner Notes

> **Remember:** Only `d['x']` (subscript) triggers the factory. `d.get('x')` and `'x' in d` do **not**.
>
> **Remember:** Pass the *class* (`list`, `int`), not the *instance* (`list()`, `int()`).
>
> **Remember:** `defaultdict(int)` is the simplest counter — but `Counter` gives you `.most_common` for free.
>
> **Remember:** `.copy()` returns a plain dict, losing the factory.
>
> **Remember:** Set `d.default_factory = None` to "lock" the dict (no more auto-creation).

---

## 17. FAANG Tips

- **Adjacency lists** (`defaultdict(list)`) are the textbook graph representation; pair with BFS/DFS. Pair with the `deque` in [deque.md](./deque.md) for O(1) BFS.
- **Group Anagrams** (#49) is the canonical `defaultdict(list)` problem — sort and group.
- **Graph with weights**: `defaultdict(lambda: defaultdict(int))` — clean double-level counter.
- **Avoid accidental insertion when just reading**: switch to `.get()` or `'k' in d`.
- For **unique grouping** (e.g., donors per project) use `defaultdict(set)` and `.add(...)`.
- For 2D infinite grids (e.g. cellular automata / sparse matrix), `defaultdict(lambda: defaultdict(int))` saves hours of bounds-checking.
- **Counting overlap with Counter**: `defaultdict(int)` and `Counter` are interchangeable for simple counting; reach for Counter when you'll need `most_common`.
- In **BFS islands / number-of-islands (#200)** variants, `defaultdict(set)` for visited cells can be slightly slower than `set` of tuples — measure before committing.

---

## 18. Practice Problems

**Easy**
- LeetCode 49 — Group Anagrams (use `defaultdict(list)`)
- LeetCode 706 — Design HashMap (`defaultdict(int)` skeleton)
- LeetCode 389 — Find the Difference (use `defaultdict(int)`)
- LeetCode 205 — Isomorphic Strings

**Medium**
- LeetCode 133 — Clone Graph (build adjacency, BFS)
- LeetCode 207 — Course Schedule (adjacency list + topological sort)
- LeetCode 310 — Minimum Height Trees
- LeetCode 811 — Subdomain Visit Count (counter via defaultdict)
- LeetCode 3 — Longest Substring Without Repeating Characters (lazy dict + last-seen)

**Hard**
- LeetCode 146 — LRU Cache (combined with OrderedDict)
- LeetCode 432 — All O'one Data Structure (defaultdict(set) of counts)
- LeetCode 711 — Number of Distinct Islands II (grouping with set of normalized shapes)

---

> Next: **[deque.md](./deque.md)** — the fast two-ended queue.
> Back to **[counter.md](./counter.md)** or jump to **[heapq.md](./heapq.md)**.