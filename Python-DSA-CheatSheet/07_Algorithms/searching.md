# Searching Algorithms in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 07 — Algorithms
> 🔗 Related: [binary_search.md](./binary_search.md) · [sorting.md](./sorting.md) · [two_pointers.md](./two_pointers.md)
> Data: [list.md](../02_Data_Types/list.md) · [set.md](../02_Data_Types/set.md) · [big_o.md](../08_Time_Complexity/big_o.md)
> Back to [README](../README.md)

---

## 1. What is it?

**Searching** answers two questions about a collection:

1. **Membership** — *"is `x` in the collection?"*
2. **Location**    — *"at what index/position is `x`?"*

The two extremes of complexity are:

- **Linear search** — scan one element at a time from one end. **O(n)**. Works on **any** sequence — sorted or not, list or string or linked structure.
- **Binary search** — repeatedly halve the search space. **O(log n)**. Requires **sorted** data. See [binary_search.md](./binary_search.md).

A useful variant of linear search is **sentinel search**: place a sentinel value equal to `x` at the end of the array to eliminate the "is index in bounds?" check on every iteration.

Python gives you three ready-to-use entry points:

| Operation              | Built-in                  | Complexity                    | Notes                                  |
|------------------------|---------------------------|-------------------------------|----------------------------------------|
| Membership            | `x in seq`                | O(n) for list/str             | O(1) for set/dict (hash)               |
| First index           | `seq.index(x)`            | O(n)                          | Raises `ValueError` if not found        |
| Count occurrences     | `seq.count(x)`            | O(n)                          | Returns number of occurrences           |
| Boolean membership    | `x in set` / `x in dict`  | O(1) average                  | Hash-based                              |

**What problem it solves:** Find an element (or its location) in a collection as fast as your data structure permits.

**Real-world analogy:** Looking for a lamp in your house. Linear search = open every drawer in order. Sentinel search = put a marker in the last drawer so you always know when you've finished. Binary search = look up "lamps" in the index of an organized catalog (needs the catalog sorted alphabetically).

---

## 2. Why do we use it?

- **Membership check** (`if x in items`) — the single most common thing you do on any sequence.
- **Finding a position** so we can mutate, delete, or peek at the neighbour.
- **Pre-step for many algorithms** — e.g. detect a cycle's start, scan for duplicates, validate.
- **Brute-force baseline** before optimizing — "can I just linear-scan?" is a valid first hatch on any unknown problem.
- **When data isn't sorted** you cannot log-search it; linear is the only option (or pay O(n log n) to sort).

---

## 3. When should I choose it? — Decision Table

| Situation                                            | Best choice                | Why                                |
|------------------------------------------------------|----------------------------|------------------------------------|
| Small list (n ≤ 32) — membership or index            | `x in lst` / `lst.index(x)`| Constant factor dominates; cache friendly |
| Unsorted list — find location                        | `lst.index(x)`             | Linear is the only option when unsorted |
| Repeated membership queries on stable data           | Build a `set` first        | O(n) build, then O(1) per query     |
| Sorted array, just membership/index                  | `bisect_left(a, x)`        | O(log n) — see `binary_search.md`    |
| Find first/last occurrence in sorted                  | `bisect_left`/`bisect_right`| O(log n) — see `binary_search.md`  |
| Custom predicate — first true in sorted              | binary search bisect pattern| See `binary_search.md`              |
| Linked list (no random access)                       | linear scan                | No index, can't binary search         |
| Stream / iterator (one-time scan)                    | linear scan                |                                     |
| Find k closest to a target in sorted array           | binary search + expand two pointers |                          |

**Always favor `set` / `dict` for repeated membership** — O(n)→O(1).

---

## 4. Syntax

```python
# Membership
x in seq             # True/False
x not in seq

# First index (raises ValueError if missing)
seq.index(x)         # whole-sequence search
seq.index(x, start)  # from start index
seq.index(x, start, end)

# Count
seq.count(x)

# Manual linear search
def linear_search(seq, target):
    for i, v in enumerate(seq):
        if v == target:
            return i
    return -1
```

---

## 5. Basic Example

```python
nums = [4, 2, 7, 1, 9, 3]

# built-in membership
print(7 in nums)        # True
print(5 in nums)        # False
print(nums.index(9))    # 4
print(nums.count(2))    # 1

# manual linear search
def linear_search(seq, target):
    for i, v in enumerate(seq):
        if v == target:
            return i
    return -1

print(linear_search(nums, 1))   # 3
print(linear_search(nums, 5))   # -1

# sentinel search (eliminate boundary check)
def sentinel_search(seq, target):
    seq = list(seq) + [target]   # don't mutate caller's list
    i = 0
    while seq[i] != target:
        i += 1
    return i if i < len(seq) - 1 else -1

print(sentinel_search(nums, 7))   # 2
print(sentinel_search(nums, 99))  # -1
```

Output:
```
True
False
4
1
3
-1
2
-1
```

---

## 6. Step-by-Step Dry Run

### Linear search `nums = [4, 2, 7, 1, 9, 3]`, `target = 1`

```
i=0  v=4 4 != 1 -> continue
i=1  v=2 2 != 1 -> continue
i=2  v=7 7 != 1 -> continue
i=3  v=1 1 == 1 -> return 3   ✔
```

### Sentential search over `[4,2,7,1,9,3]` for `1`

1. Build `[4,2,7,1,9,3,1]` — sentinel appended at the end.
2. Walk `i` while `seq[i] != 1`: i=0,1,2 → stops at i=3.
3. `3 < 6` → real match, return 3.

Dry run for missing target `99`:

1. Build `[4,2,7,1,9,3,99]`.
2. Walk i=0..5 — none equal 99.
3. i=6 hits sentinel `99`; `6 == len(seq)-1` → return `-1`.

ASCII:
```
  index:    0   1   2   3   4   5  |6|
  array:   [4,  2,  7,  1,  9,  3, |1|]   (sentinel=1)
  scan:     ^ ---^ ---^ hit!
                     return 3
```

### .index() dry run on `"hello world"` searching `'o'`

```
index: 0 1 2 3 4 5 6 7 8 9 10 11
chars: h e l l o   w o r l  d
i=0..3 (h,l,l,o first at 4)
returns 4 — first match only
```

---

## 7. Built-in Methods

### 7.1 `x in seq`
- **Purpose**: membership test, returns bool.
- **Syntax**: `target in iterable`.
- **Example**: `3 in [1,2,3]` → `True`; `"a" in "abc"` → `True`.
- **Complexity**: O(n) on list/str/tuple; O(1) average on set/dict.
- **Interview use**: dedup guard `if n not in seen:`; verifying constraints.
- **Mistakes**: using a list for fast membership — convert to `set` first when possible.
- **Shortcut**: `seen = set(); for x in a: if x in seen: ...`.

### 7.2 `seq.index(x[, start[, end]])`
- **Purpose**: first index position of `x`, with optional slice bounds.
- **Example**: `[5,3,5].index(5)` → 0; `"abc".index("b")` → 1; `[1,2,2].index(2, 2)` → finds after index 2 → returns 2.
- **Complexity**: O(n).
- **Interview use**: position lookup; first occurrence of a flag character.
- **Mistakes**: raises `ValueError` if missing — wrap with `if x in seq` or catch the exception. Also: index only returns the FIRST match.
- **Shortcut**: `i = seq.index(x) if x in seq else -1` (one-liner).

### 7.3 `seq.count(x)`
- **Purpose**: number of occurrences.
- **Example**: `[1,2,1,1].count(1)` → 3.
- **Complexity**: O(n).
- **Interview use**: counting duplicate, "find majority element" sanity check.
- **Mistakes**: `count(len)` of substring can be quadratic; consider `str.split`.
- **Shortcut**: `bool(count)` instead of `count > 0`.

### 7.4 `any()` / `all()` — short-circuit search-y helpers
- `any(iterable)` returns `True` if *any* element is truthy — **short-circuits** on first truthy.
- `all(iterable)` returns `True` if *all* elements are truthy — **short-circuits** on first falsy.
- Effective: a linear search that stops early on a predicate.
- Example: `any(x > 5 for x in nums)` — equivalent to scanning for the first match.
- Complexity: O(n) worst case, but stops as soon as possible.

### 7.5 `next()` with a generator — "first match with a default"

```python
first_even = next((x for x in nums if x % 2 == 0), None)
first_idx  = next((i for i, x in enumerate(nums) if x > 5), -1)
```
- Returns the first match or the default if none.
- Comparison: cleaner than a for-loop assignment.
- Caveat: still O(n).

---

## 8. Interview Example

### 1. Two Sum (powers a linear scan inside the canonical hashmap solution)

```python
def twoSum(nums, target):
    seen = {}
    for i, n in enumerate(nums):
        if target - n in seen:        # 'in' on dict = O(1) avg
            return [seen[target - n], i]
        seen[n] = i
    return []
```

### LeetCode 278 — First Bad Version (when version list isn't precomputed, you call `isBadVersion(v)`)

```python
def firstBadVersion(n):
    for v in range(1, n + 1):       # linear baseline — replaced by binary search in real interview
        if isBadVersion(v):
            return v
    return n
```

The above is the brute-force linear version. Real interview expects O(log n) via binary search — see [binary_search.md](./binary_search.md).

### LeetCode 35 — Search Insert Position (linear introduction)

```python
def searchInsert(nums, target):
    for i, v in enumerate(nums):
        if v >= target:
            return i
    return len(nums)
```

Linear answer: scan until first element ≥ target. The interview-grading answer is O(log n) binary search.

---

## 9. When NOT to use

- **Sorted data + repeated queries** — use binary search (`bisect`) instead.
- **Millions of membership tests** — turn the list into a `set` once, then O(1) lookups.
- **Need first/last occurrence in sorted array** — use `bisect_left`/`bisect_right`.
- **Counting many occurrences** — convert to `collections.Counter` (O(n) build, then O(1) per query).
- **Use a heap** for "smallest/largest k" — don't scan k times.
- **`.index()` in a tight loop** — O(n²) trap: convert to a dict of value→index once.
- **Searching a *sorted* list of 10⁷ items** — linear is 10⁷ ops; binary search is ~23.

---

## 10. Common Mistakes

1. **Forgetting `.index()` raises `ValueError`** — wrap with a guard or try/except.
2. **Searching in a list when the data is hashable** — convert to `set` / `dict` first.
3. **Building the same dict every time** — move it out of the loop.
4. **`if x in lst:` in a python list comprehension that re-creates the list** — O(n²) trap.
5. **Off-by-one in sentinel bounds** — append sentinel at index `n`, return `-1` when `i == n`.
6. **`"a" in "abc"` works** but `"a" in ["a","b","c"]` is element-match, not substring — be careful with strings of length > 1.
7. **Comparing types**: `3 in ["3"]` is False — Python doesn't coerce.
8. **Assuming `index()` returns last occurrence** — wrap with reverse slicing or use `rindex` for strings (`str.rindex`).
9. **Searching a custom object's `key` attribute** — use `next((o for o in items if o.key == k), None)`, not `lst.index(k)`.
10. **Linear search inside a double loop** — pre-compute with a dict.

---

## 11. Memory Tricks

- 🔑 **Linear** = "line" — walk in a straight line, one item per step.
- 🔑 **Sentinel** = a guard placed at the end so you never check the *end* of the line yourself.
- 🔑 Membership mnemonic: **`in` on list = O(n), `in` on set = O(1).**
- 🔑 Don't sort then linear search — that's wasted work. Either sort then **binary** search, or skip sorting.
- 🔑 "Find position → `.index()`; find existence → `in`; first match w/default → `next(gen, default)`."

---

## 12. Interview Shortcuts

- `x in sequence` for short sequences (n ≤ ~50).
- Build a `set` when you'll membership-check many times.
- `next((match for x in a if cond(x)), default)` for "first occurrence with default".
- `any(...)` and `all(...)` short-circuit linear search — perfect for boolean checks.
- `Counter(a)` for counting then `if cnt[x] >= 2`.
- For ordered queries with sorted data, jump to [binary_search.md](./binary_search.md).
- `dict.setdefault(k, []).append(v)` — group-and-search idiom.

---

## 13. Cheat Sheet Table

| Need                            | Best                             | Complexity         |
|---------------------------------|----------------------------------|--------------------|
| `x` in list                     | `x in lst`                       | O(n)               |
| `x` in set / dict               | `x in s`                         | O(1) avg           |
| Find first index of `x`         | `lst.index(x)`                   | O(n)               |
| First index with default        | `next((i for i,x in ... if x == t), -1)` | O(n)        |
| First satisfying predicate      | `next((x for x in a if p(x)), None)` | O(n)              |
| Exists satisfying predicate     | `any(p(x) for x in a)`           | O(n) worst, short-circuits |
| All satisfy predicate           | `all(p(x) for x in a)`           | O(n) worst, short-circuits |
| Count occurrences in list       | `lst.count(x)` or `Counter(lst)[x]` | O(n)            |
| Repeated membership on growing data | `set` membership            | O(1) avg           |
| Sorted array membership         | `bisect_left(a, x)` (binary)     | O(log n)           |
| Educational — linear (manual)   | `for i, v in enumerate(a):`      | O(n)               |
| Educational — sentinel          | (see code above)                 | O(n), 1 less branch|

---

## 14. Time Complexity Table

| Algorithm / Operation       | Best     | Average  | Worst    | Space    | Sorted required? |
|-----------------------------|----------|----------|----------|----------|------------------|
| `x in list` (membership)    | O(1)     | O(n)     | O(n)     | O(1)     | No               |
| `x in set` / `x in dict`    | O(1)     | O(1)     | O(n)*    | O(n)     | No               |
| `list.index(x)`             | O(1)     | O(n)     | O(n)     | O(1)     | No               |
| `list.count(x)`             | O(n)     | O(n)     | O(n)     | O(1)     | No               |
| Linear search (manual)      | O(1)     | O(n)     | O(n)     | O(1)     | No               |
| Sentinel search             | O(1)     | O(n)     | O(n)     | O(1)     | No               |
| `any()`/`all()` on iterable | O(1)     | O(n)     | O(n)     | O(1)     | No               |
| Binary search (see dedicated file) | O(1) | O(log n) | O(log n) | O(1) iter | Yes             |

`*` Hash collisions degrade `set`/`dict` to O(n) worst; rare in practice for non-adversarial inputs.

---

## 15. Visual Diagram (ASCII)

### Linear search versus binary search

```
  Linear search (O(n)):                            Binary search (O(log n), needs sorted):
  [3, 1, 4, 1, 5, 9, 2, 6]   target=5            [1,1,2,3,4,5,6,9]   target=5
   ^--i=0 no                                       mid=4 -> [1,1,2,3,|4|,5,6,9]
      ^--i=1 no                                     5 > 4 -> right
         ^--i=2 no                                  new range [5,6,9] mid=6
            ^--i=3 no                              5 < 6 -> left
               ^--i=4 hit! 4=>5? no                range [5] -> 5 == 5 hit!
                  ^--i=5 hit! return 5              -> total 2 comparisons vs 5
```

### Sentinel search

```
  Before:  [3, 1, 4, 1, 5]            target=5
  Append target sentinel -> [3, 1, 4, 1, 5, |5|]
  Scan without bound check:
   i=0  3 != 5
   i=1  1 != 5
   i=2  4 != 5
   i=3  1 != 5
   i=4  5 == 5  i < 5 -> return 4
```

### Flowchart — choosing a search

```
                 need to find x
                       |
              is the collection sorted?
              /                       \
            yes                         no
             |                           |
        is it hashable?              linear scan / set
        /            \              (if many queries, build set first)
       yes            no
        |              |
   use set/dict    binary search (bisect)
       O(1)

```

---

## 16. Beginner Notes — Remember block

```
Remember:
- x in lst is O(n). x in set / dict is O(1).
- lst.index(x) raises ValueError if x not present.
- Linear works on EVERY sequence; binary needs sorted.
- Sentinel search adds a guard element to save the bounds check.
- any()/all() short-circuit linear scans with a predicate.
- When in doubt and n is small, just use `in` — simplicity > speed.
- Need repeated find? Build a set or use a dict mapping value -> index.
- 278 First Bad Version: linear is the brute force; upgrade to binary search.
```

---

## 17. FAANG Tips

1. **Avoid `.index()` in a loop** — common O(n²) pitfall in interviews.
2. **Convert list → set `else` member-tested** especially when validating constraints against a known pool (e.g. `'(', '{', '['` for valid parentheses).
3. **First match with default**: `next((x for x in a if p(x)), None)` reads cleanly and short-circuits.
4. **For "first occurrence of a property"** on a non-sorted sequence, linear is the answer — don't be afraid to call it boring.
5. **Sentinel search mostly appears as a textbook trick** — interviewers rarely expect you to write it; just mention it as an optimization on linear scans.
6. Many "find X" problems on unsorted data can be re-read as "build a map, then look up" — turn `O(n)` per query into `O(1)`.
7. If sorted but you're not sure if duplicates exist, use `bisect_left` then check if `a[idx] == x`.
8. **Linear vs Binary** scan decision is always worth verbalizing in the interview — "data isn't sorted so O(n) here is the best we can do without preprocessing."
9. **'First match' vs 'all matches'**: `list.index` for the first, `(i for i,x in enumerate(a) if p(x))` for an iterator over all.
10. **Strings**: `str.find(sub)` returns `-1` (not raise) — different convention from `list.index`.

---

## 18. Practice Problems

| Difficulty | Problem                                                                               | Hint                                                |
|-----------|---------------------------------------------------------------------------------------|-----------------------------------------------------|
| Easy      | [35 Search Insert Position](https://leetcode.com/problems/search-insert-position/)    | Linear intro; upgrade to binary search             |
| Easy      | [278 First Bad Version](https://leetcode.com/problems/first-bad-version/)             | Linear baseline → O(log n) binary                  |
| Easy      | [1 Two Sum](https://leetcode.com/problems/two-sum/)                                   | Use dict membership for `target - n`               |
| Easy      | [217 Contains Duplicate](https://leetcode.com/problems/contains-duplicate/)          | `len(set(a)) != len(a)`                            |
| Easy      | [349 Intersection of Two Arrays](https://leetcode.com/problems/intersection-of-two-arrays/) | `set & set` is set-based search              |
| Medium    | [287 Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number/) | Cycle detection or set membership          |
| Medium    | [442 Find All Duplicates in an Array](https://leetcode.com/problems/find-all-duplicates-in-an-array/) | Index sign trick or Counter                |
| Hard      | [41 First Missing Positive](https://leetcode.com/problems/first-missing-positive/)   | Index remapping as linear search                   |

---

**Cross-links**: [binary_search.md](./binary_search.md) exhausts log-time search · [sorting.md](./sorting.md) is the precondition for binary search · [two_pointers.md](./two_pointers.md) often searches sorted arrays pair-wise.