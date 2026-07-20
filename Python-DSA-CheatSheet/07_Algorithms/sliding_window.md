# Sliding Window in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 07 — Algorithms
> 🔗 Related: [prefix_sum.md](./prefix_sum.md) · [two_pointers.md](./two_pointers.md) · [searching.md](./searching.md)
> Data: [list.md](../02_Data_Types/list.md) · [big_o.md](../08_Time_Complexity/big_o.md)
> Back to [README](../README.md)

---

## 1. What is it?

The **sliding window** technique maintains a **contiguous subarray (or substring)** — the "window" — that moves across the input. At each step, you **add** the new right element and **remove** the leftmost element when the window violates a constraint, so most additions happen once and each removal happens once. **Linear-time, no nested loop.**

Two flavors:

1. **Fixed-size window** — window of constant length `k` slides one step at a time; replace the element leaving with the element entering.
2. **Variable-size window** — window grows/shrinks depending on a constraint (sum ≤ target, all distinct, etc.). Right pointer always advances; left pointer jumps forward only when constraint is violated.

**What problem it solves:** Replace an O(n * k) "for each left, for each right" brute force with an **O(n)** single pass.

**Real-world analogy:** A camera sliding across a row of houses. You adjust what's in the frame by moving it one house at a time, instead of stepping back to the start and re-panning every time. The frame moves in **O(n)** total work — start once, slide forward, repeat.

---

## 2. Why do we use it?

- **Massive speedup**: O(n²) brute force → O(n) sliding window.
- **Constant-space** for many variants (only a few pointers, a counter, a dict of size ≤ 26 / set).
- Pattern-matches a huge class of "subarray / substring / contiguous" problems.
- For subarray **sum** problems with **positive ints**, sliding window is the most space-efficient alternative to prefix sums.
- Powers classic String problems (Longest Substring Without Repeating Characters, Minimum Window Substring).

---

## 3. When should I choose it? — Decision Table

| Situation                                                         | Best choice                              | Notes                                            |
|-------------------------------------------------------------------|------------------------------------------|--------------------------------------------------|
| Subarray size is fixed at `k`                                     | **Fixed-size sliding window**            | Sum of max in size-k window                       |
| Window size variable; constraint monotone (sum monotonic in window)| **Variable-size window**                | Longest/shortest subarray with sum ≤ K           |
| All elements positive, "subarray sum == k"                        | **Sliding window** OR prefix + hashmap   | Sliding window needs positivity                  |
| Mix of positive and negative numbers in subarray-sum problems     | **Prefix sum + hashmap**                | Window invariant breaks; see `prefix_sum.md`     |
| Substring with all distinct characters                            | Variable window + `dict` of last index   | "Without repeating characters" (3)                |
| Window max/min                                                    | **Window + monotonic deque**             | `collections.deque` (see 239)                     |
| K-distance with at most K replacements                            | Variable window + counter                | 424 Longest Repeating Char Replacement          |
| Window constrained by characters (must contain all of A..Z)       | Variable window + counter                | 76 Minimum Window Substring                      |
| Need O(log n) element lookup while sliding                         | Sorted container / heap inside window    | SortedList or `heapq`                             |
| Multiple passes (subarrays with sum K and only positives)         | Single pass sliding window — O(n)        | Both ends move forward                            |
| Make window containment based on number of distinct values        | Variable window + `dict` count           | Track count of each character                    |

Compare to prefixes: prefix sum answers **arbitrary range queries**, sliding window is **better** for fixed-size or contiguous queries where total work must be O(n) and queries aren't arbitrary.

---

## 4. Syntax

### Fixed-size window template

```python
def fixed_window_max_sum(a, k):
    win_sum = sum(a[:k])              # initial window
    best = win_sum
    for i in range(k, len(a)):
        win_sum += a[i] - a[i - k]    # slide: new in, old out
        best = max(best, win_sum)
    return best
```

### Variable-size window template (longest subarray with sum ≤ limit)

```python
def longest_subarray_sum_le(a, limit):
    left = 0
    cur = 0
    best = 0
    for right, x in enumerate(a):
        cur += x                       # EXPAND to the right
        while cur > limit:             # CONTRACT from the left while invalid
            cur -= a[left]
            left += 1
        best = max(best, right - left + 1)
    return best
```

### Window with a frequency dict (Longest substring without repeating chars)

```python
def lengthOfLongestSubstring(s):
    last = {}
    left = 0
    best = 0
    for right, ch in enumerate(s):
        if ch in last and last[ch] >= left:
            left = last[ch] + 1        # jump left past the duplicate
        last[ch] = right
        best = max(best, right - left + 1)
    return best
```

### Sliding-window max via deque (LeetCode 239)

```python
from collections import deque
def maxSlidingWindow(nums, k):
    dq = deque()                  # holds indices, values decreasing
    res = []
    for i, x in enumerate(nums):
        while dq and dq[0] <= i - k:        # remove out-of-window
            dq.popleft()
        while dq and nums[dq[-1]] <= x:     # remove smaller from back
            dq.pop()
        dq.append(i)
        if i >= k - 1:
            res.append(nums[dq[0]])         # front is the maximum
    return res
```

---

## 5. Basic Example

### Fixed window — maximum sum of three consecutive elements

```python
def max_sum_k(a, k):
    cur = sum(a[:k])
    best = cur
    for i in range(k, len(a)):
        cur += a[i] - a[i - k]
        best = max(best, cur)
    return best

print(max_sum_k([2, 1, 5, 1, 3, 2], 3))    # 9   (subarray [5,1,3])
print(max_sum_k([2, 3, 4, 1, 5], 2))       # 7   (subarray [3,4])
```

### Variable window — longest subarray with sum ≤ 7

```python
def longest_le_k(a, k=7):
    left, cur, best = 0, 0, 0
    for right, x in enumerate(a):
        cur += x
        while cur > k:
            cur -= a[left]
            left += 1
        best = max(best, right - left + 1)
    return best

print(longest_le_k([2, 3, 1, 2, 4, 3], 7))   # 4   (subarray [3,1,2,4] sum=10 > 7? No actually: 3+1+2=6, +4=10 so 4-long [1,2,4] sum=7 -> 3)
```

Wait, let's recompute: `a = [2,3,1,2,4,3]` and `k=7`. Start at left=0:
- right=0: cur=2, valid, window [2], best=1
- right=1: cur=5, valid, window [2,3], best=2
- right=2: cur=6, valid, window [2,3,1], best=3
- right=3: cur=8 > 7, contract: cur-=2 -> cur=6, left=1. window [3,1,2], best=3
- right=4: cur=10 > 7, contract: cur-=3 -> 7, left=2. window [1,2,4], best=3
- right=5: cur=10 > 7, contract: cur-=1 -> 9, left=3. cur=2... wait need to re-loop: cur=9 still > 7, contract cur-=2 -> 7, left=4. window [4,3], best=3

So output = 3.

### Longest substring without repeating characters (LeetCode 3)

```python
def lengthOfLongestSubstring(s):
    last, left, best = {}, 0, 0
    for right, ch in enumerate(s):
        if ch in last and last[ch] >= left:
            left = last[ch] + 1
        last[ch] = right
        best = max(best, right - left + 1)
    return best

print(lengthOfLongestSubstring("abcabcbb"))   # 3
print(lengthOfLongestSubstring("pwwkew"))     # 3  ("wke" or "kew")
```

### Longest Repeating Character Replacement (LeetCode 424)

```python
def characterReplacement(s, k):
    cnt = {}
    max_freq = 0
    left = 0
    best = 0
    for right, ch in enumerate(s):
        cnt[ch] = cnt.get(ch, 0) + 1
        max_freq = max(max_freq, cnt[ch])
        # window length - max_freq <= k means we can swap to all-same
        while (right - left + 1) - max_freq > k:
            cnt[s[left]] -= 1
            left += 1
        best = max(best, right - left + 1)
    return best
```

---

## 6. Step-by-Step Dry Run

### Fixed window — `max_sum_k([2, 1, 5, 1, 3, 2], 3)`

```
Initial: window [2, 1, 5], cur = 8, best = 8
i=3: cur += 1 - 2 = 7   window [1,5,1]   best=8
i=4: cur += 3 - 1 = 9   window [5,1,3]  best=9
i=5: cur += 2 - 5 = 6   window [1,3,2]  best=9
return 9
```

ASCII window sliding:

```
  i= 0 1 2 3 4 5
  a= [2,1,5,1,3,2]
  [2 1 5]                       initial  sum=8
     [1 5 1]                          sum=7
        [5 1 3]                          sum=9  <-- max
           [1 3 2]                          sum=6
```

### Variable window — `longest_le_k([2,3,1,2,4,3], 7)`

```
right=0 x=2  cur=2  window=[2]        best=1
right=1 x=3  cur=5  window=[2,3]      best=2
right=2 x=1  cur=6  window=[2,3,1]   best=3
right=3 x=2  cur=8 > 7  CONTRACT: cur-=2, left=1, cur=6  window=[3,1,2]  best=3
right=4 x=4  cur=10 > 7 CONTRACT: cur-=3, left=2, cur=7  window=[1,2,4]  best=3
right=5 x=3  cur=10 > 7 CONTRACT: cur-=1, left=3, cur=9 > 7 CONTRACT: cur-=2, left=4, cur=7  window=[4,3]  best=3
return 3
```

ASCII: `[L,R]` is the window.

```
  step    L R    window       cur   note
   0      0 0   [2]             2
   1      0 1   [2,3]           5
   2      0 2   [2,3,1]         6       best = 3
   3      1 3   [3,1,2]         6  ← contract  after violation
   4      2 4   [1,2,4]         7
   5      4 5   [4,3]           7  ← contract twice (best still 3)
```

### Longest substring without repeating — `abcabcbb`

```
right=0 'a' last={}              left=0 window "a"      best=1
right=1 'b' last={'a':0}         left=0 window "ab"     best=2
right=2 'c' last={'a':0,'b':1}   left=0 window "abc"    best=3
right=3 'a' 'a' in last(0) >= left(0) -> left=1  window "bca"  best=3
right=4 'b' 'b' in last(1) >= left(1) -> left=2  window "cab"  best=3
right=5 'c' 'c' in last(2) >= left(2) -> left=3  window "abc"  best=3
right=6 'b' 'b' in last(4) >= left(3) -> left=5  window "cb"   best=3
right=7 'b' 'b' in last(6) >= left(5) -> left=7  window "b"    best=3
return 3
```

### Sliding window maximum — `maxSlidingWindow([1,3,-1,-3,5,3,6,7], 3)`

```
i=0 x=1  dq=[0]
i=1 x=3  pop 0 (smaller), dq=[1]
i=2 x=-1 dq=[1,2]                       window=[1,3,-1]  res.append(nums[1]=3)
i=3 x=-3 dq=[1,2,3]                     window=[3,-1,-3] res.append(3)
i=4 x=5  pop 3(-3),2(-1),1(3) dq=[4]    window=[-1,-3,5] res.append(5)
i=5 x=3  dq=[4,5]                       window=[-3,5,3]  res.append(5)
i=6 x=6  pop 5(3),4(5) dq=[6]           window=[5,3,6]   res.append(6)
i=7 x=7  pop 6(6) dq=[7]                window=[3,6,7]   res.append(7)
res = [3, 3, 5, 5, 6, 7]
```

---

## 7. Built-in Methods

### 7.1 `collections.deque` — for monotonic window max/min
- **Purpose**: O(1) append/pop on both ends. The deque stores **indices** whose corresponding values are **decreasing**. The front is always the current window's maximum.
- **Syntax**: `from collections import deque; dq = deque()`.
- **Example**:
  ```python
  while dq and nums[dq[0]] <= i - k: dq.popleft()   # remove stale
  while dq and nums[dq[-1]] <= nums[i]: dq.pop()     # pop smaller
  dq.append(i)
  ```
- **Complexity**: O(n) total — each index appended once and removed at most once.
- **Interview use**: 239 Sliding Window Maximum, 862 Shortest Subarray with Sum at Least K (monotonic deque + prefix).
- **Mistakes**: storing values instead of indices — you need the index to detect when a max has slid out of the window.
- **Shortcut**: keep deque decreasing for max, increasing for min.

### 7.2 `collections.Counter` — frequency tracking in window
- **Purpose**: count how many times each character/value appears in the current window.
- **Syntax**: `cnt = collections.Counter(); cnt[ch] += 1; cnt[ch] -= 1; if cnt[ch] == 0: del cnt[ch]`.
- **Example**: Minimum Window Substring (76) — maintain counts of `t`, decrement when a char enters the window, count of "missing" chars.
- **Complexity**: O(1) operations on bounded alphabets (e.g. ASCII).
- **Mistakes**: forgetting to `del` keys with zero count — `len(cnt)` will lie about distinct count.

### 7.3 `set` for distinct windows
- **Purpose**: track unique members of a variable window.
- **Mistakes**: easy to forget removal when the left pointer moves, leaving stale entries — use `dict` of positions per value to disambiguate duplicates.

### 7.4 `sortedcontainers.SortedList` (third-party)
- **Purpose**: keep a sorted multiset of window contents; supports O(log k) insert/delete/max/min.
- **Use case**: k-something window with arbitrary ordering not expressible as a monotonic deque.
- **Not in stdlib** — most interview panels prefer not requiring it.

### 7.5 Built-in slicing & sum
- `sum(a[i:i+k])` is O(k) — don't use inside a tight loop, except for one-off queries.
- For the initial window you may use `sum(a[:k])` — that's a single O(k) call, fine.

---

## 8. Interview Example

### LeetCode 121 — Best Time to Buy and Sell Stock (modified window)

```python
def maxProfit(prices):
    min_price = float("inf")
    best = 0
    for p in prices:
        min_price = min(min_price, p)
        best = max(best, p - min_price)
    return best
```

This is a window where left = min so far & right = current day.

### LeetCode 643 — Maximum Average Subarray I (fixed window)

```python
def findMaxAverage(nums, k):
    cur = sum(nums[:k])
    best = cur
    for i in range(k, len(nums)):
        cur += nums[i] - nums[i - k]
        best = max(best, cur)
    return best / k
```

### LeetCode 3 — Longest Substring Without Repeating Characters (variable window)

```python
def lengthOfLongestSubstring(s):
    last = {}
    left = 0
    best = 0
    for right, ch in enumerate(s):
        if ch in last and last[ch] >= left:
            left = last[ch] + 1
        last[ch] = right
        best = max(best, right - left + 1)
    return best
```

### LeetCode 424 — Longest Repeating Character Replacement

```python
def characterReplacement(s, k):
    cnt = {}
    max_freq = 0
    left = 0
    best = 0
    for right, ch in enumerate(s):
        cnt[ch] = cnt.get(ch, 0) + 1
        max_freq = max(max_freq, cnt[ch])
        while (right - left + 1) - max_freq > k:
            cnt[s[left]] -= 1
            left += 1
        best = max(best, right - left + 1)
    return best
```

### LeetCode 76 — Minimum Window Substring

```python
from collections import Counter
def minWindow(s, t):
    need = Counter(t)
    missing = len(t)
    left = 0
    best = (float("inf"), 0, 0)
    for right, ch in enumerate(s):
        if need[ch] > 0:
            missing -= 1
        need[ch] -= 1
        while missing == 0:
            if right - left + 1 < best[0]:
                best = (right - left + 1, left, right)
            need[s[left]] += 1
            if need[s[left]] > 0:
                missing += 1
            left += 1
    return s[best[1]:best[2] + 1] if best[0] != float("inf") else ""
```

### LeetCode 239 — Sliding Window Maximum

```python
from collections import deque
def maxSlidingWindow(nums, k):
    dq = deque()
    res = []
    for i, x in enumerate(nums):
        while dq and dq[0] <= i - k:
            dq.popleft()
        while dq and nums[dq[-1]] <= x:
            dq.pop()
        dq.append(i)
        if i >= k - 1:
            res.append(nums[dq[0]])
    return res
```

---

## 9. When NOT to use

- **Negative values in subarray-sum problems**: window invariant (`sum ≤ limit`) no longer means "shrink left makes sum smaller". Use prefix sums + hashmap (see `prefix_sum.md`).
- **Non-contiguous subsequences**: sliding window requires contiguous — for subsequences, use DP or other techniques.
- **Window constraint not monotonic**: if removing a left element can sometimes make the window *more* invalid, sliding window cannot guarantee progress.
- **Needing all subarrays** (not just max/longest): brute force or different formulation (prefix sum enumeration).
- **2D / matrix windows** are doable but expensive: consider 2D prefix sum instead.
- **Frequent element updates outside window**: use a heap or balanced tree per window.

---

## 10. Common Mistakes

1. **Forgetting to contract after expanding**: must `while condition_violated: shrink` immediately after `cur += a[right]`.
2. **Shrinking when condition isn't strictly violated**: typical off-by-one (`>` vs `>=`) gives a window of wrong size.
3. **`left` may not always move by 1**: in "without repeating characters", `left = last[ch] + 1` jumps. In "max sum ≤ K", `left++` repeatedly.
4. **stale `last` dict entries**: when checking `if ch in last and last[ch] >= left`, the `>= left` part matters — old positions before `left` are irrelevant.
5. **Updating `max_freq` in 424**: the "lazy" version never decreases `max_freq` even when stale, but works because we only require window length - max_freq ≤ k as an upper bound.
6. **Sliding window max deque storing indices vs values**: storing values won't tell you when an element has slid off.
7. **Initial window for fixed-size**: must sum `a[:k]` *before* the loop, not inside it.
8. **Using sliding window with negative numbers for `sum == k`** — fails silently. Switch to prefix sum + hashmap.
9. **Off-by-one when computing window length**: `right - left + 1`, not `right - left` (you want inclusive count).
10. **Deque `.popleft` not `.pop`**: the front of the deque is the live max; `.pop()` removes from back. Choose carefully.

---

## 11. Memory Tricks

- 🔑 **Two pointers, both moving right** — that's the sliding window invariant. Left never overtakes right.
- 🔑 **Expand on each iter, then contract while invalid.** Pronounce it like a worm: "extend, fix, extend".
- 🔑 For fixed-size windows: **"slide by one, subtract out, add in."**
- 🔑 For variable windows: **"R pushes forward, L follows when violated."**
- 🔑 For max-in-window: monotonic deque — front always carries the current max.
- 🔑 For longest/shortest without repeating: `last[ch]` stores the **most recent** index — `left = last[ch] + 1` jumps to skip the duplicate instantly.

---

## 12. Interview Shortcuts

- **O(n) time, O(1) space** is the typical complexity of sliding window — say it before coding.
- For "longest subarray with sum at most K and all positives": plain variable window with `while cur > K: ...`.
- For "subarray sum exactly K with positives": variable window; with negatives: prefix + hashmap.
- "Longest substring without repeating": use `dict` to remember last index, jump left.
- "Most-replaceable string equal to one repeating char (424)": track `max_freq` and window length.
- **Distinct-k** windows (340 Longest Substring with At Most K Distinct Chars): use `Counter` and shrink while `len(cnt) > k`.
- **Sliding window maximum (239)**: monotonic deque — say the invariant out loud before coding.
- For most substring problems: "window + a map of frequencies" is the universal skeleton.

---

## 13. Cheat Sheet Table

| Pattern                                              | Code skeleton                                                | Complexity        |
|-----------------------------------------------------|--------------------------------------------------------------|-------------------|
| Fixed size-k sum/max                               | `cur += a[i] - a[i-k]`                                      | O(n), O(1)        |
| Variable size (longest with sum ≤ K)                | `while cur > K: cur -= a[left]; left += 1`                  | O(n), O(1)        |
| Variable size (longest distinct chars)              | `dict[ch]=index; left = max(left, last[ch]+1)`              | O(n), O(min(n,Σ)) |
| K distinct chars                                    | `Counter` + `len(cnt) > k` shrinking                         | O(n), O(k)        |
| Replacement cost ≤ K (424)                          | `max_freq`; window - max_freq ≤ k                            | O(n), O(Σ)        |
| Window maximum (239)                                | Monotonic deque                                              | O(n), O(k)        |
| Window minimum                                       | Monotonic deque increasing                                   | O(n), O(k)        |
| Minimum window substring                             | `Counter(t)`, missing counter, shrink to expand             | O(n), O(Σ)        |
| "Subarray sum == K, all positives"                  | Variable window                                              | O(n), O(1)        |
| "Subarray sum == K, may have negatives"             | Prefix sum + hashmap                                         | O(n), O(n)        |

---

## 14. Time Complexity Table

| Variant                                  | Time   | Space       | Notes                                                    |
|------------------------------------------|--------|-------------|----------------------------------------------------------|
| Fixed-size window sum/max                | O(n)   | O(1)        | Each index enters and leaves the window exactly once.   |
| Variable window (sum-constraint)        | O(n)   | O(1)        | Both pointers move monotonically right.                  |
| Variable window (distinct chars)        | O(n)   | O(min(n,Σ)) | Hash map / array of size alphabet                        |
| K-replacement (424)                      | O(n)   | O(Σ)        | Counter                                                  |
| Min window substring                     | O(n + m) | O(Σ)      | m = size of t                                            |
| Sliding window maximum (deque)          | O(n)   | O(k)        | Each index pushed/popped at most once                    |
| Subarray sum == K with negatives        | O(n)   | O(n)        | Prefix sum + hashmap (sliding window won't work)         |

`Σ` is the size of the alphabet (26 for English lower-case; 128 for ASCII).

---

## 15. Visual Diagram (ASCII)

### Fixed window sliding over an array

```
  Array: [ 2, 1, 5, 1, 3, 2 ]    k = 3
  Step 1:  [2 1 5]                    sum=8
  Step 2:    [1 5 1]                  sum=7
  Step 3:      [5 1 3]                sum=9  <-- max
  Step 4:        [1 3 2]              sum=6
  
  ASCII movement:
  
   [2 1 5]                                |
        [1 5 1]                           | window slides right
             [5 1 3]                      |
                  [1 3 2]                 |
```

### Variable window expanding & contracting

```
   Pointer L, R both start at 0
   Expand: R++
   Contract: while invalid: cur -= a[L]; L++

        L                                 (R expands)
        ---->----
   L>|          |R    when invalid, L chases R
        shrink
   ASCII:
                 |valid...|
                              |invalid|
                                     L jumps to here
```

### Longest substring without repeating characters

```
  s = a b c a b c b b
   L=0 R=0:  "a"          last={a:0}
   L=0 R=1:  "ab"  no rep
   L=0 R=2:  "abc" no rep
   L=0 R=3:  repeat a at idx 3 (last[a]=0, >= L=0)
              -> L jumps to last[a]+1 = 1
   L=1 R=3:  "bca"          last={a:3, b:1, c:2}
   ... and so on, best=3 ("abc")
```

### Sliding window max monotonic deque

```
  nums = [1, 3, -1, -3, 5, 3, 6, 7], k = 3

   i=0 (1) dq=[0]
   i=1 (3) dq=[1]                       smaller popped
   i=2(-1) dq=[1,2]                     -> max=3
   i=3(-3) dq=[1,2,3]                   -> max=3   window [3,-1,-3]
   i=4 (5) dq=[4]                       smaller popped -> max=5
   i=5 (3) dq=[4,5]                     -> max=5
   i=6 (6) dq=[6]                       -> max=6
   i=7 (7) dq=[7]                       -> max=7
   result: [3, 3, 5, 5, 6, 7]
```

### Flowchart — choosing a flavor

```
            Does the problem require
            a contiguous subarray/substring?
                       |
                  yes — sort the array?
                  /            \
              yes (sorted)     no (preserve order)
               |                |
           two pointers         sliding window
                                    |
                    Window size fixed?
                       /         \
                   yes           no
                    |             |
              fixed-window    variable-window
              (maintain sum   (expand, then
              and update)      shrink while invalid)
```

---

## 16. Beginner Notes — Remember block

```
Remember:
- Sliding window works on CONTIGUOUS arrays/substrings.
- Right pointer always advances; left pointer jumps/shrinks after violation.
- Fixed-size window: replace outgoing with incoming element.
- Variable window: while invalid, shrink from the left.
- Use deque (monotonic) for window max/min in O(1) amortized.
- For negative numbers, switch to prefix sum + hashmap.
- Maintaining a frequency Counter/dict is the standard "state" of a window.
- Longest: maximize (right - left + 1) over valid windows.
- Shortest: minimize (right - left + 1) over valid windows; shrink aggressively while valid.
```

---

## 17. FAANG Tips

1. **Mention "sliding window" explicitly** when the problem mentions any of: "subarray", "substring", "contiguous", "longest", "shortest", "max sum of size k".
2. **Variable vs fixed** is the first decision — write which one you'll use, then code.
3. **For "at most K"/"exactly K"** with all positives, the variable window is clean. With negatives, ask "do elements ever become negative?" if so → prefix + hashmap.
4. **State** of the window is one of: a single sum, a frequency dict, a Counter, a monotonic deque. Decide before coding.
5. **Longest substring templates are interview classics** (3, 424, 340, 76) — memorize the dict skeleton.
6. **Sliding window maximum (239)**: state the deque invariant: "values decreasing, indices increasing".
7. **"Smallest subarray with sum ≥ K"** with all positives: simple variable window. With negatives: monotonic deque + prefix sum (862, harder).
8. **Always dry-run on small cases** with both pointers visualized as `[L, R]` to catch off-by-one window size errors.
9. **Time complexity reflex**: "each element enters once, leaves once" → O(n).
10. **Frequency-Counter bookkeeping**: never forget to `del cnt[k]` when count drops to zero — `len` otherwise lies.

---

## 18. Practice Problems

| Difficulty | Problem                                                                                         | Hint                                                  |
|-----------|--------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Easy      | [643 Maximum Average Subarray I](https://leetcode.com/problems/maximum-average-subarray-i/)   | Fixed window k                                        |
| Easy      | [121 Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/) | Window with min-so-far                          |
| Easy      | [219 Contains Duplicate II](https://leetcode.com/problems/contains-duplicate-ii/)              | Sliding hash of last index                            |
| Medium    | [3 Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/) | Variable window + dict          |
| Medium    | [424 Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement/) | max_freq + window size      |
| Medium    | [209 Minimum Size Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum/)      | Variable window, shrink while sum ≥ target           |
| Medium    | [340 Longest Substring with At Most K Distinct Characters](https://leetcode.com/problems/longest-substring-with-at-most-k-distinct-characters/) | Counter + shrink while distinct > K |
| Hard      | [239 Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/)            | Monotonic deque                                       |
| Hard      | [76 Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/)        | Counter + missing tracker                              |
| Hard      | [862 Shortest Subarray with Sum at Least K](https://leetcode.com/problems/shortest-subarray-with-sum-at-least-k/) | Monotonic deque + prefix sums  |

---

**Cross-links**: [prefix_sum.md](./prefix_sum.md) for clicks where negatives break the window invariant · [two_pointers.md](./two_pointers.md) for the cousin technique on sorted arrays · [searching.md](./searching.md) for membership and locating.