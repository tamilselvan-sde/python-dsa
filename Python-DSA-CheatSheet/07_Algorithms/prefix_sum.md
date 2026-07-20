# Prefix Sum in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 07 — Algorithms
> 🔗 Related: [sliding_window.md](./sliding_window.md) · [two_pointers.md](./two_pointers.md) · [sorting.md](./sorting.md)
> Data: [list.md](../02_Data_Types/list.md) · [big_o.md](../08_Time_Complexity/big_o.md)
> Back to [README](../README.md)

---

## 1. What is it?

A **prefix sum** (aka **cumulative sum**, **running sum**) is an array `pref` where `pref[i]` stores the sum of the first `i` elements of the original array. With it, the **sum of any subarray `a[l..r]`** can be computed in **O(1)** time:

```
sum(a[l..r]) = pref[r+1] - pref[l]
```

This shifts the thinking: instead of recomputing a range sum each time (O(n)), precompute once (O(n)) and answer each query in O(1).

**Definition (one index padding, easiest formula):**

```
pref[0] = 0
pref[i+1] = pref[i] + a[i]         # for i in range(n)
range_sum(l, r) = pref[r+1] - pref[l]    # inclusive l, r
```

**What problem it solves:** Answering many range-sum (or range-count, range-aggregate) queries on a static array in constant time per query.

**Real-world analogy:** A running bank balance. The balance at the end of day 10 is the prefix sum. To know how much you spent between day 3 and day 7, subtract `balance[7] - balance[2]` — no need to re-add every transaction.

---

## 2. Why do we use it?

- Pre-process in **O(n)** once, answer unlimited range-sum queries in **O(1)** each.
- Powerful when paired with**hash maps** — turns many "count subarrays with sum k" problems into O(n).
- Variants generalize to other aggregates: prefix **max/min**, **product**, **XOR**, **count**.
- Foundational for the **difference array** technique (range update in O(1)).
- Forms the basis for **2D prefix sums** (sum of a submatrix in O(1) per query).

Compare to alternatives:
- Brute `sum(a[l:r+1])` is O(n) per query, O(nq) total.
- Sliding window is O(n) total but only works for sliding ranges, not arbitrary ranges.
- A Fenwick / segment tree is overkill if the array is static — prefix sums have zero per-query overhead.

---

## 3. When should I choose it? — Decision Table

| Situation                                                   | Best choice                                                  |
|-------------------------------------------------------------|--------------------------------------------------------------|
| Static array, multiple range-sum queries                    | **1D prefix sum**                                            |
| Static matrix, multiple sub-matrix sum queries               | **2D prefix sum**                                            |
| "Number of subarrays whose sum equals `k`"                  | Prefix sum + **dict of counts**                              |
| "Maximum subarray sum" (Kadane fits too)                    | Kadane (no prefix needed)                                    |
| Range updates (add value to a range) + final array          | **Difference array** (inverse of prefix sum)                 |
| Range sum queries *and* point updates                      | **Fenwick tree / Segment tree**                              |
| Range minimum queries                                       | Sparse table, not prefix sum                                 |
| Sliding subarray sums of a fixed size                       | Sliding window (no prefix sum needed) — see `sliding_window.md` |
| Sequence of running maxes / running products                | Prefix max / prefix product                                  |
| XOR of subarray                                             | Prefix XOR                                                   |

---

## 4. Syntax

```python
# build prefix sum (one-index padding)
pref = [0] * (len(a) + 1)
for i in range(len(a)):
    pref[i + 1] = pref[i] + a[i]

# range sum inclusive [l, r]
def range_sum(pref, l, r):
    return pref[r + 1] - pref[l]

# modulo-variant prefix to detect equal sub-sums
pref_red = [0] * (len(a) + 1)
for i in range(len(a)):
    pref_red[i + 1] = (pref_red[i] + a[i])
```

Itertools alternative:

```python
from itertools import accumulate
pref = list(accumulate(a, initial=0))   # includes leading 0
# range_sum(l, r) = pref[r+1] - pref[l]
```

---

## 5. Basic Example

```python
a = [3, 1, 4, 1, 5, 9, 2, 6]
pref = [0] * (len(a) + 1)
for i in range(len(a)):
    pref[i + 1] = pref[i] + a[i]
print(pref)               # [0, 3, 4, 8, 9, 14, 23, 25, 31]

def range_sum(l, r):
    return pref[r + 1] - pref[l]

print(range_sum(2, 4))    # subarray [4,1,5] = 10
print(range_sum(0, 7))    # whole array = 31
print(range_sum(3, 3))    # single element a[3] = 1
```

### Using itertools.accumulate

```python
from itertools import accumulate
pref = list(accumulate([3,1,4,1,5,9], initial=0))
# [0, 3, 4, 8, 9, 14, 23]
print(pref[5] - pref[1])   # range [1..4] -> a[1]+a[2]+a[3]+a[4] = 1+4+1+5 = 11
```

### Hash for counting "subarrays with sum exactly k"

```python
def subarraySum(nums, k):
    seen = {0: 1}
    total = 0
    ans = 0
    for x in nums:
        total += x
        ans += seen.get(total - k, 0)
        seen[total] = seen.get(total, 0) + 1
    return ans
```

### 2D prefix sum

```python
def build_2d_prefix(mat):
    m, n = len(mat), len(mat[0])
    pref = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(m):
        for j in range(n):
            pref[i+1][j+1] = (mat[i][j]
                               + pref[i][j+1]
                               + pref[i+1][j]
                               - pref[i][j])
    return pref

def submat_sum(pref, r1, c1, r2, c2):       # inclusive bounds
    return (pref[r2+1][c2+1]
            - pref[r1][c2+1]
            - pref[r2+1][c1]
            + pref[r1][c1])
```

---

## 6. Step-by-Step Dry Run

### Build prefix for `a = [3, 1, 4, 1, 5]`

```
pref[0] = 0
i=0: pref[1] = 0 + 3 = 3
i=1: pref[2] = 3 + 1 = 4
i=2: pref[3] = 4 + 4 = 8
i=3: pref[4] = 8 + 1 = 9
i=4: pref[5] = 9 + 5 = 14
pref = [0, 3, 4, 8, 9, 14]
```

ASCII table:

```
  Index i:    0   1   2   3   4
  Original:   3   1   4   1   5
  Prefix  :   3   4   8   9  14
  pref[0] : 0  (sentinel for sum-from-0)
```

### Range query `sum(a[1..3])` = `a[1]+a[2]+a[3]` = `1 + 4 + 1` = `6`

Using formula: `range_sum(1, 3) = pref[4] - pref[1] = 9 - 3 = 6`. ✅

```
  a = [3, 1, 4, 1, 5]
  pref = [0, 3, 4, 8, 9, 14]
  
       l=1                 r=3
       v                   v
       a[1] + a[2] + a[3] = 1 + 4 + 1 = 6
  
  pref[r+1] - pref[l] = pref[4] - pref[1] = 9 - 3 = 6
```

### Subarray sum equals k (LeetCode 560) dry run

`a = [1, 1, 1]`, `k = 2`

```
total=0  seen={0:1}
i=0 : total=1  need=-1   seen has -1? no    ans=0   seen={0:1, 1:1}
i=1 : total=2  need=0    seen has 0 (cnt=1) ans=1   seen={0:1, 1:1, 2:1}
i=2 : total=3  need=1    seen has 1 (cnt=1) ans=2
return 2   (subarrays: [1,1] at indices 0-1 and 1-2)
```

---

## 7. Built-in Methods

### 7.1 `itertools.accumulate(iterable, func=add, *, initial=None)`
- **Purpose**: produce a running aggregate (`+` by default; can be `mul`, `max`, `min`, `xor`, custom).
- **Syntax**: `list(accumulate(a, initial=0))`.
- **Example**: `list(accumulate([1,2,3,4]))` → `[1, 3, 6, 10]`; `list(accumulate([1,2,3,4], initial=0))` → `[0, 1, 3, 6, 10]`; `list(accumulate([1,2,3,4], func=mul))` → `[1, 2, 6, 24]`.
- **Complexity**: O(n) time, O(n) space.
- **Interview use**: shortest prefix-sum construction; usable for prefix XOR, prefix product.
- **Mistakes**: forgetting `initial=0` (Python ≥ 3.8) → array is one shorter and `range_sum` formula needs adjusting.
- **Shortcut**: prefer `accumulate` for clean LeetCode submissions; manual loop for pedagogy.

### 7.2 `collections.Counter` — counting prefix sums
- **Purpose**: in "subarrays with sum k" (560), use `Counter` to track how many times each prefix value was seen.
- **Example**: `from collections import Counter; c = Counter([0]); c[total - k]` returns 0 for missing keys.
- **Complexity**: O(n) build + O(1) per increment/read.
- **Mistakes**: forgetting to seed with `{0: 1}` — the "empty prefix sum" is needed for subarrays starting at index 0.

### 7.3 Difference array (inverse of prefix sum)
- **Purpose**: apply range updates (add value v to every element in `a[l..r]`) in **O(1)**, then build the final array in one O(n) prefix pass.
- **Syntax**:
  ```python
  diff = [0] * (n + 1)
  for l, r, v in updates:
      diff[l] += v
      diff[r + 1] -= v
  # one pass to expand
  cur = 0
  for i in range(n):
      cur += diff[i]
      a[i] += cur
  ```
- **Complexity**: O(1) per update, O(n) finalization.
- **Interview use**: "range addition" (370), corporate flight bookings (1109).
- **Mistakes**: forgetting `diff[r + 1] -= v` (must decrement after the range to stop the addition).
- **Shortcut**: visualize the difference array as step-up at `l`, step-down at `r+1`.

---

## 8. Interview Example

### LeetCode 303 — Range Sum Query — Immutable

```python
class NumArray:
    def __init__(self, nums):
        self.pref = [0] * (len(nums) + 1)
        for i, v in enumerate(nums):
            self.pref[i + 1] = self.pref[i] + v

    def sumRange(self, l, r):
        return self.pref[r + 1] - self.pref[l]
```

### LeetCode 560 — Subarray Sum Equals K

```python
def subarraySum(nums, k):
    cnt = {0: 1}
    total = 0
    ans = 0
    for x in nums:
        total += x
        ans += cnt.get(total - k, 0)
        cnt[total] = cnt.get(total, 0) + 1
    return ans
```

### LeetCode 238 — Product Except Self (variant — prefix & suffix products)

```python
def productExceptSelf(nums):
    n = len(nums)
    out = [1] * n
    pre = 1
    for i in range(n):
        out[i] = pre
        pre *= nums[i]
    suf = 1
    for i in range(n - 1, -1, -1):
        out[i] *= suf
        suf *= nums[i]
    return out
```

### LeetCode 1732 — Find the Highest Altitude

```python
def largestAltitude(gain):
    cur = 0
    ans = 0
    for g in gain:
        cur += g
        ans = max(ans, cur)
    return ans
```

### LeetCode 525 — Contiguous Array (longest equal 0 and 1)

Treat 0 as -1; prefix sum = 0 means equal counts.

```python
def findMaxLength(nums):
    idx = {0: -1}
    total = 0
    ans = 0
    for i, x in enumerate(nums):
        total += 1 if x == 1 else -1
        if total in idx:
            ans = max(ans, i - idx[total])
        else:
            idx[total] = i
    return ans
```

---

## 9. When NOT to use

- **Update-heavy range-sum**: when elements mutate, pref gets stale on every update; use a Fenwick (BIT) tree.
- **Range minimum/max queries**: prefix min works only for queries from `[0..i]`, not arbitrary `[l..r]`. Use a sparse table or segment tree.
- **"Sub-sum of a fixed-size subarray"**: a sliding window is O(n) total and avoids extra O(n) space — see `sliding_window.md`.
- **Min subarray sum or max subarray sum**: Kadane's algorithm is O(n) without prefix sums.
- **Subarray **≥ K** with negative values**: trickier; use subarray sum near-K with sorted prefix sums + deque, or Kadane variant.

---

## 10. Common Mistakes

1. **Wrong formula**: `range_sum(l, r) = pref[r] - pref[l]` only works if `pref[i] = a[0]+...+a[i]` (no leading 0). Easier to always prepend 0.
2. **Index off-by-one with prefix**: `pref[l]` not `pref[l - 1]` when `pref[0]=0`.
3. **Forgetting to seed `{0:1}`** in "sum equals k" — misses subarrays starting at index 0.
4. **2D formula wrong signed**: submatrix sum = `pref[r2+1][c2+1] - pref[r1][c2+1] - pref[r2+1][c1] + pref[r1][c1]`. Don't forget the **+ pref[r1][c1]** (inclusion-exclusion).
5. **Difference array forgetting `diff[r+1] -= v`** — addition leaks past the range end.
6. **Using prefix sum on a mutable array** without rebuilding — out-of-date prefix.
7. **Integer overflow** in other languages — not an issue in Python but worth noting.
8. **Confusing "subsequence" (can skip) vs "subarray" (contiguous)** — prefix sums apply to **contiguous** only.
9. **Building the prefix array in-place over `a`** while still indexing `a` from a later stage — make a separate buffer.
10. **Modulo variants**: forgetting modulus during overflow or using wrong modulus.

---

## 11. Memory Tricks

- 🔑 **Prefix sum** = running total. The earliest definition most of us learned.
- 🔑 Formula rhythm: `sum(l..r) = pref[r+1] - pref[l]`. Always `r+1` minus `l` (both one-indexed).
- 🔑 `pref = [0] + accumulate` — the sentinel zero is your secret weapon.
- 🔑 "Two prefix sums were equal ⇒ elements between them sum to 0." — base of "subarray sum equals k" pattern.
- 🔑 Difference array is the *inverse*: prefix sum of a diff array is the original.
- 🔑 2D inclusion-exclusion: subtract the two rect overlaps, add back their intersection.

---

## 12. Interview Shortcuts

- `itertools.accumulate(a, initial=0)` is the fastest constructor.
- For "subarray sum == k", **prefix sum + hashmap** is the standard recipe — O(n) time, O(n) space.
- For "subarray sum == k" **with negative numbers**, you **still** can use the hashmap; if only positives, sliding window also works (saves space).
- For "range update + final read" use a **difference array**.
- "Subarray with **sum 0**" → look for repeated prefix values.
- "Subarray sum divisible by k" → store `pref[i] % k` instead of `pref[i]`.

---

## 13. Cheat Sheet Table

| Operation                                | Code / formula                                         | Time   | Space    |
|------------------------------------------|--------------------------------------------------------|--------|----------|
| Build prefix (1D)                        | `pref[i+1] = pref[i] + a[i]`                          | O(n)   | O(n)     |
| Build via accumulate                     | `list(accumulate(a, initial=0))`                     | O(n)   | O(n)     |
| Range sum                                | `pref[r+1] - pref[l]`                                 | O(1)   | O(1)     |
| Count subarrays sum == k                 | prefix sum + `dict` counts                            | O(n)   | O(n)     |
| Range update (difference array)           | `diff[l] += v; diff[r+1] -= v;` then prefix pass     | O(1)/update, O(n) final | O(n) |
| Build prefix 2D                          | `pref[i+1][j+1] = mat[i][j] + ...`                    | O(mn)  | O(mn)    |
| Submatrix sum                            | inclusion-exclusion                                   | O(1)   | O(1)     |
| Prefix product                           | `pref[i+1] = pref[i] * a[i]`                          | O(n)   | O(n)     |
| Prefix XOR                               | `pref[i+1] = pref[i] ^ a[i]`                          | O(n)   | O(n)     |

---

## 14. Time Complexity Table

| Operation                              | Time       | Space            |
|----------------------------------------|------------|------------------|
| Build prefix array (size n)            | O(n)       | O(n)             |
| Single range-sum query                  | O(1)       | O(1)             |
| All pairs range-sum analysis (n²)      | O(n²)      | O(n)             |
| Count subarrays with sum k             | O(n)       | O(n)             |
| 2D prefix build (m × n)               | O(mn)      | O(mn)            |
| Submatrix sum query (2D)               | O(1)       | O(1)             |
| Difference array — q range updates     | O(q + n)   | O(n)             |
| Fenwick tree (for point updates)       | O(log n) build/query | O(n)  |

---

## 15. Visual Diagram (ASCII)

### Original vs prefix

```
  Index:    0   1   2   3   4   5   6   7
  Original: 3   1   4   1   5   9   2   6
  Prefix :  3   4   8   9  14  23  25  31
  pref[0]: 0  (sentinel)

  range [2..4] = a[2]+a[3]+a[4] = 4+1+5 = 10
              = pref[5]-pref[2] = 14 - 4 = 10
```

### Formula visual — "stretched shadow"

```
  pref[r+1] covers a[0..r]   (everything up to r)
  pref[l]   covers a[0..l-1] (everything before l)
  subtract  -> covers a[l..r] only
                   [l         ]
                              [r]
   [0.........l-1] [l..........r]
   ^^^^^^^^^^^^^^^   ^^^^^^^^^^^^^^
     pref[l]          range_sum(l,r)
                                  -> pref[r+1] = pref[l] + range_sum(l,r)
   => range_sum(l, r) = pref[r+1] - pref[l]
```

### Subarray-sum-equals-k ladder

```
  indices:   0   1   2   3   4   5
  a:         1   1   1   1   1
  pref:  0   1   2   3   4   5
  k = 2  (find total - k in seen)
  Each "earlier prefix with value (total - 2)" yields a valid subarray ending at the current index.
```

### 2D prefix diagram

```
  mat =  [(r1,c1)............(r2,c2)]
  
  pref[r2+1][c2+1] = entire (0..r2, 0..c2)
  pref[r1][c2+1]   = upper part to remove
  pref[r2+1][c1]    = left part to remove
  pref[r1][c1]       = added back (was subtracted twice)
```

### Flowchart

```
  start
   |
  need many range sums on static array?
       yes -> build pref (O(n)). query each in O(1).
       no  -> use brute sum or different structure
```

---

## 16. Beginner Notes — Remember block

```
Remember:
- pref[i+1] = pref[i] + a[i]; query = pref[r+1] - pref[l].
- Always prepend pref[0] = 0 to handle index 0 ranges seamlessly.
- Subarray sum == k: keep dict of prefix counts; seed {0:1}.
- Difference array for range updates in O(1).
- For 2D, use inclusion-exclusion: pref[r2+1][c2+1] - pref[r1][c2+1] - pref[r2+1][c1] + pref[r1][c1].
- Sliding window beats prefix sum only if ranges are fixed length or all-positive-with-non-shrinking windows.
- Update array => re-build prefix, or use Fenwick tree.
```

---

## 17. FAANG Tips

1. **Identify "range sum" / "subarray sum"** in the problem statement → prefix sum is a candidate.
2. **Counting subarrays** with sum `== k` or sum `% k == 0`: combine prefix sum with hash map.
3. **Sliding window needs positive numbers only** for the typical "subarray sum == k" — if negatives exist, prefix + hashmap is the right answer.
4. **2D prefix sum** (304 Range Sum Query 2D Immutable): pre-compute the prefix matrix once, answer each query in O(1).
5. **Difference array** for "range add q times then output" problems — saves O(qn) → O(q + n).
6. **Product except self (238)**: think prefix product + suffix product in O(n) without division.
7. **Contiguous array (525)**: regroup 0 into -1 so prefix = 0 means equal counts.
8. **Make Sum Divisible by P (1590)**: prefix sum + hashmap of mod to latest index.
9. **Path Sum IV / matrix path prefix**: 2D prefix is often the unblocker.
10. **Cumulative max/min puzzles**: prefix max or prefix product (think "trap rain water" variations).

---

## 18. Practice Problems

| Difficulty | Problem                                                                              | Hint                                              |
|-----------|--------------------------------------------------------------------------------------|---------------------------------------------------|
| Easy      | [303 Range Sum Query Immutable](https://leetcode.com/problems/range-sum-query-immutable/) | Classic 1D prefix                         |
| Easy      | [1732 Find the Highest Altitude](https://leetcode.com/problems/find-the-highest-altitude/) | Running max of prefix                          |
| Easy      | [1413 Minimum Value to Get Positive Step by Step Sum](https://leetcode.com/problems/minimum-value-to-get-positive-step-by-step-sum/) | Track min prefix                 |
| Medium    | [560 Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/)   | Prefix + hashmap `seen[total - k]`             |
| Medium    | [525 Contiguous Array](https://leetcode.com/problems/contiguous-array/)              | Treat 0 as -1; hashmap of prefix to index       |
| Medium    | [238 Product Except Self](https://leetcode.com/problems/product-except-self/)      | Prefix product + suffix product                  |
| Medium    | [304 Range Sum Query 2D Immutable](https://leetcode.com/problems/range-sum-query-2d-immutable/) | 2D prefix                             |
| Medium    | [1109 Corporate Flight Bookings](https://leetcode.com/problems/corporate-flight-bookings/) | Difference array                       |
| Hard      | [1590 Make Sum Divisible by P](https://leetcode.com/problems/make-sum-divisible-by-p/) | Prefix mod + hashmap                  |

---

**Cross-links**: [sliding_window.md](./sliding_window.md) for fixed-size / variable-size range problems · [two_pointers.md](./two_pointers.md) for pair-based subarray problems · [sorting.md](./sorting.md) for sorting-based preprocessing before prefix queries.