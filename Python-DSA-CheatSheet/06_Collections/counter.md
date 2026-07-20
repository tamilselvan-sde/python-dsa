# collections.Counter

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 06 — Collections
> 🔗 Related: [defaultdict.md](./defaultdict.md) · [deque.md](./deque.md) · [heapq.md](./heapq.md) · [../02_Data_Types/dictionary.md](../02_Data_Types/dictionary.md)
> Back to [README](../README.md)

---

## 1. What is it?

`collections.Counter` is a **dict subclass** (since Python 2.7) designed for **counting hashable objects**.
It maps each element → its frequency (an integer, can be zero or negative).

Think of it as a **multiset / bag** data structure from mathematics:

```
Counter('aabbbc')  ==  {'a': 2, 'b': 3, 'c': 1}
```

Key facts:
- Stored as a regular dict, so insertion order is **preserved** (Python 3.7+).
- Missing keys return `0` instead of raising `KeyError` — this is the magic.
- Values can go negative (via `.subtract` or arithmetic) — different from a plain dict.
- Located in the `collections` module: `from collections import Counter`.

---

## 2. Why do we use it?

Counting things by hand with a plain dict is verbose:

```python
freq = {}
for ch in s:
    freq[ch] = freq.get(ch, 0) + 1
```

With `Counter` it is a one-liner:

```python
freq = Counter(s)
```

We use it for:
- **Frequency counting** — characters, words, IDs, votes.
- **Anagram / permutation checks** — two things are anagrams iff their Counters are equal.
- **Top-K problems** — `.most_common(k)` returns the k most frequent items.
- **Multiset arithmetic** — union/intersection of frequency maps.
- **Inventory / shopping cart** decrements via `.subtract`.

It saves ~5 lines of boilerplate every time you need a histogram.

---

## 3. When should I choose it?

| Situation                                          | Best tool           |
|----------------------------------------------------|---------------------|
| Count hashable items, need top-k or `0` default    | ✅ `Counter`         |
| Need a dict that auto-creates missing values (list/int) | `defaultdict`   |
| Need FIFO order with O(1) push/pop on both ends    | `deque`             |
| Need k-th smallest / largest, partial sort         | `heapq`             |
| Need order-preserving unique items                 | `dict` (insertion)  |
| Need sorted-by-count output                        | `Counter.most_common`|
| Just need existence tests                          | `set`               |

**Decision flow:**

```
   Need to COUNT things?
           │
   ┌───────┴───────┐
   YES             NO
   │               │
top-k needed?    Default dict
   │               (use defaultdict)
   YES → Counter
   NO  → defaultdict(int) is fine too
```

See [defaultdict.md](./defaultdict.md) for the lazy-creation sibling.

---

## 4. Syntax

```python
from collections import Counter

# Constructors
Counter()                       # empty
Counter(iterable)               # count each item
Counter(mapping)               # copy counts from another dict
Counter(a=2, b=3)              # keyword counts

# Reading / writing
c['a']                          # 0 if missing (no KeyError)
c['a'] = 5                     # set count
del c['a']                      # remove key

# Common methods
c.most_common(n=None)
c.elements()
c.update(iter_or_mapping)
c.subtract(iter_or_mapping)
c.total()                      # Python 3.10+

# Arithmetic & set ops
c1 + c2                        # add counts (union of bags)
c1 - c2                        # subtract counts (keeps only positives)
c1 & c2                        # intersection → min(a,b)
c1 | c2                        # union            → max(a,b)

# Unary
+c                             # drop zero/negative counts
-c                             # negate counts
```

---

## 5. Basic Example

```python
from collections import Counter

c = Counter('abracadabra')
print(c)                       # Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
print(c['a'])                  # 5
print(c['z'])                   # 0  ← no KeyError!
print(c.most_common(2))         # [('a', 5), ('b', 2)]

c.update('aaa')
print(c['a'])                   # 8

d = Counter('abracadabra')
print(c == d)                  # False (we added 3 a's)
```

**Output:**
```
Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
5
0
[('a', 5), ('b', 2)]
8
False
```

---

## 6. Step-by-Step Dry Run

Input string: `"aabbbc"`

| Step | char seen | Counter state                      |
|------|-----------|------------------------------------|
| 1    | 'a'       | {'a': 1}                           |
| 2    | 'a'       | {'a': 2}                           |
| 3    | 'b'       | {'a': 2, 'b': 1}                   |
| 4    | 'b'       | {'a': 2, 'b': 2}                   |
| 5    | 'b'       | {'a': 2, 'b': 3}                   |
| 6    | 'c'       | {'a': 2, 'b': 3, 'c': 1}           |

Then `most_common(2)`:
1. Sort items by count desc → `[('b',3),('a',2),('c',1)]`
2. Take first 2 → `[('b',3),('a',2)]`

Time: building the Counter is O(n); `most_common(k)` is O(n log k) internally (uses a heap).

---

## 7. Built-in Methods   (EXHAUSTIVE)

### `Counter(iterable_or_mapping_or_kwargs)`
- **Purpose:** Construct a Counter from any iterable, mapping, or keyword args.
- **Syntax:** `Counter([iter])`
- **Input:** iterable / mapping / keywords
- **Output:** new `Counter` instance
- **Example:**
  ```python
  Counter([1,1,2,3,3,3])        # Counter({3: 3, 1: 2, 2: 1})
  Counter({'x': 4})             # Counter({'x': 4})
  Counter(red=2, blue=3)        # Counter({'blue': 3, 'red': 2})
  ```
- **Time:** O(n) over input length.
- **Interview use:** first line of every frequency problem.
- **Common mistake:** passing a generator twice — works but is wasteful.
- **Shortcut:** `Counter(string)` covers characters; `Counter(string.split())` covers words.

### `.most_common(n=None)`
- **Purpose:** Return the n most common items as `[(item, count), ...]` sorted by count desc (ties broken by insertion order).
- **Syntax:** `c.most_common([n])`
- **Input:** integer n (optional; None = all)
- **Output:** list of (item, count) tuples
- **Example:**
  ```python
  Counter('aabbbcc').most_common(2)   # [('b', 3), ('a', 2)]
  ```
- **Time:** O(n log k) with heap when k<n; O(n log n) when n is large; O(n) when n=None (equivalent to sorted).
- **Interview use:** Top-K Frequent, leaderboards.
- **Common mistake:** forgetting ties are broken by insertion order, not by key.
- **Shortcut:** `c.most_common()[-1]` gives the **least** common item.

### `.elements()`
- **Purpose:** Iterator that yields each element repeated by its count (ignores zeros/negatives, order is insertion-based).
- **Syntax:** `c.elements()`
- **Input:** none
- **Output:** iterator of items
- **Example:**
  ```python
  list(Counter(a=2, b=3).elements())   # ['a','a','b','b','b']
  ```
- **Time:** O(total counts) = O(n).
- **Interview use:** reconstruct string from a Counter (e.g. anti-palindrome).
- **Common mistake:** returns an iterator, not a list — wrap in `list()`.
- **Shortcut:** `sorted(c.elements())` gives a multiset-sorted sequence.

### `.update(iterable_or_mapping)`
- **Purpose:** Add counts (in-place). Existing keys are incremented.
- **Syntax:** `c.update(other)`
- **Input:** iterable / mapping / Counter
- **Output:** None (mutates)
- **Example:**
  ```python
  c = Counter('abb')
  c.update('abc')              # Counter({'b': 3, 'a': 2, 'c': 1})
  ```
- **Time:** O(len other).
- **Interview use:** streaming counters, rolling frequency.
- **Common mistake:** confusing with `+` which returns a new Counter.
- **Shortcut:** `c.update(other)` is in-place `c = c + other`.

### `.subtract(iterable_or_mapping)`
- **Purpose:** In-place subtract counts (can go negative).
- **Syntax:** `c.subtract(other)`
- **Input:** iterable / mapping
- **Output:** None (mutates)
- **Example:**
  ```python
  c = Counter('aabbc')
  c.subtract('aa')            # Counter({'b': 2, 'c': 1, 'a': 0})
  ```
- **Time:** O(len other).
- **Interview use:** inventory, shopping cart, anagram difference (`c - Counter(t)`).
- **Common mistake:** leaves zero/negative keys behind — use `+c` to clean.
- **Shortcut:** `+c` drops count<=0 entries.

### `.total()` (Python 3.10+)
- **Purpose:** Sum of all counts.
- **Syntax:** `c.total()`
- **Input:** none
- **Output:** int
- **Example:**
  ```python
  Counter('aabb').total()      # 4
  ```
- **Time:** O(1) (cached).
- **Interview use:** check whether two Counters cover the same total.
- **Common mistake:** before 3.10 you must do `sum(c.values())`.
- **Shortcut:** `sum(c.values())` is the pre-3.10 equivalent.

### `+` (add)
- **Purpose:** Union of multisets: counts add up.
- **Syntax:** `c1 + c2`
- **Output:** new Counter
- **Example:** `Counter(a=2) + Counter(a=1, b=3)` → `Counter({'a': 3, 'b': 3})`
- **Time:** O(len(c1) + len(c2)).
- **Common mistake:** drops negative/zero entries on output.
- **Shortcut:** combine two frequency tables.

### `-` (subtract)
- **Purpose:** Difference of multisets; **only positive counts survive**.
- **Syntax:** `c1 - c2`
- **Output:** new Counter
- **Example:** `Counter(a=3, b=2) - Counter(a=1, b=3)` → `Counter({'a': 2})` (b dropped)
- **Time:** O(len(c1) + len(c2)).
- **Interview use:** find chars in `s` missing from `t`, anagram difference, ransom-note problems.
- **Common mistake:** drops zero counts (vs `.subtract` which keeps them).
- **Shortcut:** `c - c` is always empty — sanity check.

### `&` (intersection — min)
- **Purpose:** Element-wise **minimum** count; only items in BOTH.
- **Syntax:** `c1 & c2`
- **Output:** new Counter
- **Example:** `Counter(a=3,b=2) & Counter(a=1,b=5)` → `Counter({'a': 1, 'b': 2})`
- **Time:** O(min(len(c1), len(c2))).
- **Interview use:** longest common subsequence of multiset, shared characters.
- **Common mistake:** not the set-intersection of keys — min of counts.
- **Shortcut:** `c1 & c2` = "what's the maximal bag that fits both?"

### `|` (union — max)
- **Purpose:** Element-wise **maximum** count; items in EITHER.
- **Syntax:** `c1 | c2`
- **Output:** new Counter
- **Example:** `Counter(a=1) | Counter(a=3,b=2)` → `Counter({'a':3,'b':2})`
- **Time:** O(len(c1) + len(c2)).
- **Interview use:** merge two frequency maps keeping the "bigger bag".
- **Common mistake:** summing counts instead of taking max — that's `+`.
- **Shortcut:** `c | c` == `c`.

### Unary `+c` and `-c`
- **Purpose:** `+c` keeps only positive entries; `-c` negates and keeps positives.
- **Syntax:** `+c`, `-c`
- **Example:**
  ```python
  c = Counter(a=2, b=-1, c=0)
  +c                         # Counter({'a': 2})
  -c                         # Counter({'b': 1})
  ```
- **Time:** O(len(c)).
- **Shortcut:** use `+c` after `.subtract` to clean up zero/negatives.

### `Counter.fromkeys(iterable, value=0)` (inherited from dict)
- **Purpose:** Create a Counter with keys from iterable, all set to `value`.
- **Example:** `Counter.fromkeys('abc', 1)` → `Counter({'a':1,'b':1,'c':1})`
- **Common mistake:** default `value` is `None`, not `0`!  Pass `0` explicitly.

---

## 8. Interview Example

**Top K Frequent Elements (#347)** — return the k most frequent elements in O(n log k):

```python
from collections import Counter
import heapq

def topKFrequent(nums, k):
    c = Counter(nums)
    return [item for item, _ in c.most_common(k)]

# Variants using an explicit heap (more flexible):
def topKFrequent_heap(nums, k):
    c = Counter(nums)
    return heapq.nlargest(k, c.keys(), key=c.get)
```

**Anagram check:**

```python
from collections import Counter
def isAnagram(s, t):
    return Counter(s) == Counter(t)
```

**Ransom Note (#383):**

```python
from collections import Counter
def canConstruct(ransom, mag):
    rc, mc = Counter(ransom), Counter(mag)
    return not (rc - mc)   # if anything left over → impossible
```

---

## 9. When NOT to use

- You need **insertion-order utility AND counts aren't needed** → just use `dict` or `OrderedDict`.
- You need **auto-creation of new keys with non-integer defaults** (lists, sets) → use `defaultdict` ([defaultdict.md](./defaultdict.md)).
- You need **sorted data / partial sorting** → use `heapq` ([heapq.md](./heapq.md)) or `sorted()`.
- You only need **unique membership tests** → `set` is faster and clearer.
- Tiny data (3 items) → a plain dict is fine; Counter is overkill.
- You need **per-key locking / thread safety** → wrap in a lock, Counter is not atomic.

---

## 10. Common Mistakes

| Mistake                                                  | Fix                                              |
|----------------------------------------------------------|--------------------------------------------------|
| `c['missing']` raises KeyError? Actually returns 0      | ✅ behavior is correct — but `c.get('missing')` returns `None`. |
| Forgetting `most_common(n)` returns `(item, count)` tuples, not bare items | use `[k for k,_ in ...]`. |
| Using `&` thinking it's set intersection of KEYS         | it's element-wise **min of counts**.             |
| `c.subtract` keeps zero-count keys — iterating them out | wrap with `+c`.                                  |
| Treating Counter as immutable after `c + c2`            | arithmetic returns a **new** Counter.            |
| Comparing Counters ignoring zeros                        | ok—zero entries don't affect equality.           |
| Expecting missing key to raise                           | use `in` if you really want a KeyError-style test.|
| Mutating while iterating                                  | copy: `for k in list(c): ...`.                   |
| Using Counter for ordered-with-duplicates data           | use a list or `deque`.                           |
| Sorting keys alphabetically instead of by frequency      | sort with `key=c.get` or use `most_common`.      |

---

## 11. Memory Tricks

- **Counter = "bag of things"** — like a real bag, you toss items in; counts are how many.
- `most_common` → **"most"** = highest counts first.
- `&` and `|` mirror **set ops** but on counts: intersection = min, union = max.
- `+` adds bags (multiset union), `-` removes a bag (only positives survive).
- Missing key → 0 because **a bag simply has zero of that item**.
- `.elements()` literally **expands the bag** back to a list.
- `+c` is like **shaking the bag** to drop the empties.

---

## 12. Interview Shortcuts

```python
# Top-k frequency
Counter(nums).most_common(k)

# Equal multisets? (anagram)
Counter(s) == Counter(t)

# Items in s missing from t (positive only)
Counter(s) - Counter(t)

# Char frequency table ready for sliding window
Counter(s[:k])            # window seed

# Convert Counter → ranked list (no ties issue)
sorted(c, key=c.get, reverse=True)

# Tally a list of votes
Counter(votes).most_common(1)[0][0]

# Build a Counter from a 2D matrix
Counter(tuple(row) for row in matrix)

# Pre-3.10 total
sum(c.values())
```

---

## 13. Cheat Sheet Table

| Operation                         | Returns        | Mutates? | Notes                                |
|-----------------------------------|----------------|----------|--------------------------------------|
| `Counter(it)`                     | Counter        | —        | O(n)                                 |
| `c[k]`                            | int            | no       | 0 if missing                         |
| `c.most_common(k)`                | list of pairs  | no       | O(n log k)                           |
| `c.elements()`                    | iterator       | no       | sorted-by-insertion order            |
| `c.update(it)`                    | None           | yes      | adds counts                          |
| `c.subtract(it)`                  | None           | yes      | counts may go negative               |
| `c.total()` (3.10+)               | int            | no       | O(1) cached                          |
| `c1 + c2`                         | Counter        | no       | multiset sum                         |
| `c1 - c2`                         | Counter        | no       | drops ≤0                             |
| `c1 & c2`                         | Counter        | no       | min counts                           |
| `c1 \| c2`                         | Counter       | no       | max counts                           |
| `+c` / `-c`                       | Counter        | no       | keep positive / negate               |
| `del c[k]`                        | None           | yes      | removes key                          |
| `c.pop(k)`                        | int            | yes      | removes key, default 0               |
| `c.clear()`                       | None           | yes      | inherited from dict                  |

---

## 14. Time Complexity Table

| Operation              | Time              | Notes                              |
|------------------------|-------------------|------------------------------------|
| Construction           | O(n)              | iterate input once                 |
| `c[k]`                 | O(1)              | dict lookup                         |
| `c[k] = v`             | O(1)              |                                    |
| `.update(it)`          | O(len it)         |                                    |
| `.subtract(it)`        | O(len it)         |                                    |
| `.elements()`          | O(Σ counts)       | lazily generated                    |
| `.most_common(k)`      | O(n log k)        | internally uses `heapq.nlargest`    |
| `.most_common()`       | O(n log n)        | full sort                          |
| `+` `-` `&` `\|`       | O(|c1| + |c2|)    | new Counter                        |
| `+c` / `-c`            | O(|c|)            |                                    |
| `c.total()`            | O(1)              | cached since 3.10                  |
| `in c`                 | O(1)              | hash lookup                         |

---

## 15. Visual Diagram (ASCII)

```
      Input:  "aabbbc"
            ↓   ↓   ↓   ↓   ↓   ↓
         ┌──────────────────────────┐
         │      Counter engine       │  (hashes each item,
         │   (dict of key → count)   │   increments value)
         └──────────────────────────┘
                    ↓
        ┌──────────────────────────┐
        │  {'a':2, 'b':3, 'c':1}   │
        └──────────────────────────┘
                    ↓
       most_common(2) sorts by count desc:
                    ↓
           [('b', 3),
            ('a', 2)]


Multiset arithmetic (think of bags):

  Counter(a=3,b=2)   ●●●  ●●
  Counter(a=1,b=4)     ●  ●●●●

  + (add)        →  Counter(a=4, b=6)        ●●●●  ●●●●●●
  - (sub)        →  Counter(a=2)            ●●          (b dropped: 2-4 <0)
  & (min)        →  Counter(a=1, b=2)        ●   ●●
  | (max)        →  Counter(a=3, b=4)       ●●●  ●●●●
```

---

## 16. Beginner Notes

> **Remember:** A `Counter` is just a dict that returns `0` for missing keys instead of raising `KeyError`. That's 90% of its magic. The other 10% is `.most_common`, arithmetic, and `.elements`.
>
> **Remember:** `most_common(k)` returns `(item, count)` tuples — not items alone.
>
> **Remember:** `&` is **min** (intersection), `|` is **max** (union). These behave like set operations but on counts.
>
> **Remember:** `.subtract` may leave zero/negative keys; use `+c` to clean them.
>
> **Remember:** `Counter(s) == Counter(t)` is the cleanest anagram check in Python.

---

## 17. FAANG Tips

- **Top-K Frequent** (#347): `Counter(nums).most_common(k)` is Python's idiomatic one-liner. Interviewers love/hate it.
- **Sliding-window frequency**: build `Counter(s[:k])`, then slide — `add s[i]`, `remove s[i-k]`. Pre-computes a hash-map of the window in O(k); each slide is O(1) amortized. See [../07_Algorithms/sliding_window.md](../07_Algorithms/sliding_window.md).
- **Find All Anagrams in a String** (#438): comparing two Counters each step is O(unique chars) — fine for ASCII, but a "match count" trick is faster: track how many characters currently have the right frequency.
- **Ransom Note** (#383): `Counter(ransom) - Counter(mag)` empty? → possible.
- **Hash map problems** where order of counts matters: convert to a list of pairs and sort. See [../07_Algorithms/hash_map.md](../07_Algorithms/hash_map.md).
- Don't reinvent `Counter` with `defaultdict(int)` — Counter gives you `most_common` for free.
- Negative-count trick: `Counter(int) - Counter(int)` after streaming to see "what's missing".
- For huge data use `collections.Counter` + chunked (`.update(chunk)`) accumulation.

---

## 18. Practice Problems

**Easy**
- LeetCode 242 — Valid Anagram
- LeetCode 383 — Ransom Note
- LeetCode 266 — Palindrome Permutation (premium)
- LeetCode 409 — Longest Palindrome

**Medium**
- LeetCode 347 — Top K Frequent Elements
- LeetCode 438 — Find All Anagrams in a String
- LeetCode 451 — Sort Characters By Frequency
- LeetCode 1636 — Sort Array by Increasing Frequency

**Hard**
- LeetCode 992 — Subarrays with K Different Integers
- LeetCode 480 — Sliding Window Median (Counter trick variant)
- LeetCode 295 — Find Median from Data Stream (counter-based bucket approach)

---

> Next: **[defaultdict.md](./defaultdict.md)** — the sibling that auto-creates missing values.
> Then: **[deque.md](./deque.md)** and **[heapq.md](./heapq.md)**.