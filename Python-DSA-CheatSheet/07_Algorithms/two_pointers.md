# Two Pointers in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 07 — Algorithms
> 🔗 Related: [sliding_window.md](./sliding_window.md) · [binary_search.md](./binary_search.md) · [sorting.md](./sorting.md) · [searching.md](./searching.md)
> Data: [list.md](../02_Data_Types/list.md) · [big_o.md](../08_Time_Complexity/big_o.md)
> Back to [README](../README.md)

---

## 1. What is it?

**Two pointers** is a technique where you maintain **two indices** (or iterators) into a sequence and move them under rules that yield O(n) total work, even on problems that naive brute-force solves in O(n²) or O(n³).

Two canonical flavors:

1. **Same direction (slow-fast)** — both pointers start at the same end; the fast one scans ahead while the slow one lags behind, often **writing** the "kept" elements. Classic: **283 Move Zeroes**, **26 Remove Duplicates**, **141 Linked List Cycle**.

2. **Opposite directions (left-right)** — one starts at index `0`, the other at index `n-1`, and they march toward each other. Used on **sorted arrays** for two-sum, three-sum, palindrome check, container with most water.

A third cousin: **two pointers on two independent sequences** — merge two sorted arrays (88), compare strings, intersect arrays (349).

**What problem it solves:** Replace a nested loop with **two monotonic single passes** — typical speedup O(n²) → O(n).

**Real-world analogy:** Finding a matching couple at a dance. The naive approach: try every boy with every girl (O(n²)). The sorted approach: line up dancers by height; one pointer at the shortest boy and one at the tallest girl, move whoever has the next-closer height — O(n) instead.

---

## 2. Why do we use it?

- **O(n) time** with **O(1) extra space** in almost all cases (no hashmap, no extra array).
- When input is **sorted**, two-pointer replaces the hash map needed for two-sum / k-sum.
- Many **in-place** algorithms require two pointers (partition, dedupe, color sort).
- Naturally handles **palindromes**, container-area problems, and water-trapping patterns.
- Powerful when paired with **sort first** (then two-pointer): 3Sum, 4Sum, "closest 3-sum", triangle types.

---

## 3. When should I choose it? — Decision Table

| Problem statement / property                                  | Best choice                              | Notes                                              |
|---------------------------------------------------------------|------------------------------------------|----------------------------------------------------|
| "Given a sorted array, find two indices summing to target"   | **Two-pointer left/right**                | 167 Two Sum II Sorted                             |
| "Find pairs with a value ≤/≥/== threshold in sorted array"   | Two-pointer left/right                    | Adjust rules based on target                      |
| "Three distinct elements summing to target"                   | Sort + two-pointer per first index        | 15 3Sum                                            |
| Move zeros to end / partition by predicate                    | **Slow-fast same direction**              | 283 Move Zeroes; 75 Sort Colors                  |
| "Remove duplicates in-place" (sorted)                         | Slow-fast same direction                  | 26 Remove Duplicates from Sorted Array            |
| "Detect cycle in a linked list"                               | Slow-fast (Floyd's tortoise & hare)       | 141 Linked List Cycle                              |
| "Find the middle of a linked list"                            | Slow-fast                                  | 876 Middle of the Linked List                     |
| "Is the string a palindrome"                                  | Two-pointer left/right                    | 125 Valid Palindrome                               |
| Max area between two vertical lines (unsorted)               | **Two-pointer left/right** (greedy)       | 11 Container With Most Water                     |
| "Trap rain water" — bounded by min of 2 bars                  | Two-pointer left/right (min-of-2 trick)   | 42 Trapping Rain Water                            |
| Subarray with a constraint (contiguous sliding)              | **Sliding window** instead of two-pointer| See `sliding_window.md`                            |
| Unsorted + many membership checks                             | Hash map / two-sum (1)                    | Not two pointers — see `searching.md`              |
| Sorted + membership                                           | Binary search (single element)           | See `binary_search.md`                              |
| Merge two sorted arrays                                        | Two pointers parallel                     | 88 Merge Sorted Array                              |
| K-Sorted fusion                                                | Sort then two-pointer per dimension       | Generalizes to 3Sum/4Sum                           |

---

## 4. Syntax

### Same direction (slow-fast in-place)

```python
def remove_zeros(nums):
    slow = 0
    for fast in range(len(nums)):
        if nums[fast] != 0:                  # predicate: keep non-zeros
            nums[slow], nums[fast] = nums[fast], nums[slow]
            slow += 1
    return slow                              # new "length"
```

### Opposite directions (sorted two-sum)

```python
def two_sum_sorted(a, target):
    lo, hi = 0, len(a) - 1
    while lo < hi:
        s = a[lo] + a[hi]
        if s == target:
            return [lo, hi]
        elif s < target:
            lo += 1
        else:
            hi -= 1
    return []
```

### Linked-list slow-fast (Floyd's cycle detection)

```python
def hasCycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False
```

### Palindrome

```python
def isPalindrome(s):
    l, r = 0, len(s) - 1
    while l < r:
        if s[l] != s[r]:
            return False
        l += 1
        r -= 1
    return True
```

---

## 5. Basic Example

### Move Zeroes (LeetCode 283)

```python
def moveZeroes(nums):
    slow = 0
    for fast in range(len(nums)):
        if nums[fast] != 0:
            nums[slow], nums[fast] = nums[fast], nums[slow]
            slow += 1
    return nums

print(moveZeroes([0, 1, 0, 3, 12]))   # [1, 3, 12, 0, 0]
```

### Two Sum II — Sorted (LeetCode 167)

```python
def twoSum(numbers, target):
    lo, hi = 0, len(numbers) - 1
    while lo < hi:
        s = numbers[lo] + numbers[hi]
        if s == target:
            return [lo + 1, hi + 1]        # 1-indexed
        elif s < target:
            lo += 1
        else:
            hi -= 1
    return []

print(twoSum([2, 7, 11, 15], 9))    # [1, 2]
```

### 3Sum (LeetCode 15) — sort + two-pointer per first element

```python
def threeSum(nums):
    nums.sort()
    n = len(nums)
    res = []
    for i in range(n - 2):
        if i > 0 and nums[i] == nums[i - 1]:    # skip duplicates
            continue
        lo, hi = i + 1, n - 1
        while lo < hi:
            s = nums[i] + nums[lo] + nums[hi]
            if s == 0:
                res.append([nums[i], nums[lo], nums[hi]])
                lo += 1
                hi -= 1
                while lo < hi and nums[lo] == nums[lo - 1]:
                    lo += 1
                while lo < hi and nums[hi] == nums[hi + 1]:
                    hi -= 1
            elif s < 0:
                lo += 1
            else:
                hi -= 1
    return res

print(threeSum([-1, 0, 1, 2, -1, -4]))   # [[-1, -1, 2], [-1, 0, 1]]
```

### Container With Most Water (LeetCode 11) — opposite directions greedy

```python
def maxArea(height):
    lo, hi = 0, len(height) - 1
    best = 0
    while lo < hi:
        h = min(height[lo], height[hi])
        best = max(best, h * (hi - lo))
        if height[lo] < height[hi]:
            lo += 1
        else:
            hi -= 1
    return best

print(maxArea([1, 8, 6, 2, 5, 4, 8, 3, 7]))   # 49
```

### Sort Colors (LeetCode 75) — three pointers (Dutch flag)

```python
def sortColors(nums):
    lo, mid, hi = 0, 0, len(nums) - 1
    while mid <= hi:
        if nums[mid] == 0:
            nums[lo], nums[mid] = nums[mid], nums[lo]
            lo += 1; mid += 1
        elif nums[mid] == 1:
            mid += 1
        else:                                # nums[mid] == 2
            nums[mid], nums[hi] = nums[hi], nums[mid]
            hi -= 1
    return nums
```

### Trapping Rain Water (LeetCode 42)

```python
def trap(height):
    lo, hi = 0, len(height) - 1
    left_max = right_max = 0
    ans = 0
    while lo < hi:
        if height[lo] < height[hi]:
            if height[lo] >= left_max:
                left_max = height[lo]
            else:
                ans += left_max - height[lo]
            lo += 1
        else:
            if height[hi] >= right_max:
                right_max = height[hi]
            else:
                ans += right_max - height[hi]
            hi -= 1
    return ans
```

---

## 6. Step-by-Step Dry Run

### Two Sum II Sorted — `numbers = [2, 7, 11, 15]`, `target = 9`

```
lo=0 hi=3   sum = 2 + 15 = 17  > 9  -> hi--
lo=0 hi=2   sum = 2 + 11 = 13  > 9  -> hi--
lo=0 hi=1   sum = 2 + 7  = 9   == 9 -> return [1, 2]   (1-indexed)
```

ASCII:
```
  index:   0   1   2   3
  nums:    2   7  11  15
           ^         ^
           lo,       hi       sum=17 > 9 → hi--
           ^     ^
            lo,   hi          sum=13 > 9 → hi--
           ^ ^
           lo,hi             sum= 9 == 9 → MATCH [lo+1, hi+1] = [1, 2]
```

### 3Sum — `nums = [-1, 0, 1, 2, -1, -4]`, `target = 0`

```
sort:    [-4, -1, -1, 0, 1, 2]
i=0 -4:  lo=1 hi=5  -4 + -1 + 2 = -3 < 0 -> lo++
         lo=2 hi=5  -4 + -1 + 2 = -3 -> lo++
         lo=3 hi=5  -4 + 0 + 2  = -2 -> lo++
         lo=4 hi=5  -4 + 1 + 2  = -1 -> lo++
         lo=5 (lo==hi) stop
i=1 -1 (skip dupes? nums[1]==nums[0]? no, but -1 != -4):
  lo=2 hi=5  -1 + -1 + 2 = 0  -> add [-1, -1, 2]; skip dups lo=3 hi=4
  lo=3 hi=4  -1 + 0 + 1  = 0  -> add [-1, 0, 1];  lo=4 hi=3 stop
i=2 -1 (skip because nums[2] == nums[1] == -1)
i=3  0...
result: [[-1, -1, 2], [-1, 0, 1]]
```

### Container With Most Water — `height = [1,8,6,2,5,4,8,3,7]`

```
lo=0 hi=8  h=1 area=1*8=8 best=8   h[lo]<h[hi] -> lo++
lo=1 hi=8  h=7 area=7*7=49 best=49  h[lo]<h[hi]? 8 > 7 -> hi--
lo=1 hi=7  h=3 area=3*6=18  best=49 hi--
lo=1 hi=6  h=8 area=8*5=40  best=49 8 < 8 false -> hi-- (lo side: 8 < 8 -> false)
   ... actually h[1]=8, h[6]=8, equal, default branch hi--, hi=5
lo=1 hi=5  h=4 area=4*4=16 best=49 hi--
lo=1 hi=4  h=5 area=5*3=15 best=49 8 > 5 hi--? actually height[lo]=8 height[hi]=5; 8 < 5 false so hi--; hi=3
   ...
   keep contracting, max stays 49
return 49
```

### Move Zeroes — `nums = [0, 1, 0, 3, 12]`

```
slow=0 fast=0 nums[0]=0 -> skip (no swap)
slow=0 fast=1 nums[1]=1 !=0 -> swap -> [1, 0, 0, 3, 12], slow=1
slow=1 fast=2 nums[2]=0 -> skip
slow=1 fast=3 nums[3]=3 !=0 -> swap -> [1, 3, 0, 0, 12], slow=2
slow=2 fast=4 nums[4]=12 !=0 -> swap -> [1, 3, 12, 0, 0], slow=3
done. Output: [1, 3, 12, 0, 0]
```

### Sort Colors — `nums = [2, 0, 2, 1, 1, 0]`

```
lo=0 mid=0 hi=5 nums[mid]=2 -> swap with hi=[5]=0 -> [0,0,2,1,1,2] hi=4
lo=0 mid=0 hi=4 nums[mid]=0 -> swap lo=mid=0 -> [0,0,2,1,1,2] lo=1 mid=1
lo=1 mid=1 hi=4 nums[mid]=0 -> swap lo=1 -> [0,0,2,1,1,2] lo=2 mid=2
lo=2 mid=2 hi=4 nums[mid]=2 -> swap hi=4 -> [0,0,1,1,2,2] hi=3
lo=2 mid=2 hi=3 nums[mid]=1 -> mid++
lo=2 mid=3 hi=3 nums[mid]=1 -> mid++ -> exit
Output: [0, 0, 1, 1, 2, 2]
```

### Floyd cycle detection on linked list — list `1 -> 2 -> 3 -> 4 -> 2` (cycle)

```
slow: 1 -> 2 -> 3 -> 4 -> 2 -> 3 -> 4 -> 2 ... (cycle)
fast: 1 -> 3 -> 2 -> 4 -> 3 -> 2 -> 4 -> ...
they meet inside the cycle when `slow is fast`
```

---

## 7. Built-in Methods

### 7.1 Swift hacks not needed — but companion tools
Two pointers operates only on indexing/slicing — no built-in methods. Sometimes you combine it with:

### 7.2 `sorted()` and `list.sort()`
- Many two-pointer problems require **sorted input first** (3Sum, 4Sum, two-sum sorted, triangle types).
- **Sort → two pointers** is the canonical recipe: time O(n log n) for sort + O(n) for two-pointer scan.
- See [sorting.md](./sorting.md) for details.

### 7.3 `enumerate` — for pair iteration alongside one pointer

```python
for i, x in enumerate(nums):
    # i is the slow pointer position
```

### 7.4 `zip` — for parallel two-pointer iteration over two sequences

```python
for x, y in zip(A, B):
    # x is from A, y from B
```

### 7.5 `reversed()` — flip the order so left/right pointers travel from the same end

```python
for x in reversed(nums):
    ...
```

### 7.6 `collections.deque` — for two-ended pops when emulating from both ends of a list

- `popleft`/`pop` for opposite-direction removals.
- See [sliding_window.md](./sliding_window.md) for deque usage.

---

## 8. Interview Example

### LeetCode 15 — 3Sum

```python
def threeSum(nums):
    nums.sort()
    res = []
    n = len(nums)
    for i in range(n - 2):
        if i > 0 and nums[i] == nums[i - 1]:
            continue
        lo, hi = i + 1, n - 1
        while lo < hi:
            s = nums[i] + nums[lo] + nums[hi]
            if s == 0:
                res.append([nums[i], nums[lo], nums[hi]])
                lo += 1
                hi -= 1
                while lo < hi and nums[lo] == nums[lo - 1]:
                    lo += 1
                while lo < hi and nums[hi] == nums[hi + 1]:
                    hi -= 1
            elif s < 0:
                lo += 1
            else:
                hi -= 1
    return res
```

### LeetCode 11 — Container With Most Water

```python
def maxArea(height):
    lo, hi = 0, len(height) - 1
    best = 0
    while lo < hi:
        h = min(height[lo], height[hi])
        best = max(best, h * (hi - lo))
        if height[lo] < height[hi]:
            lo += 1
        else:
            hi -= 1
    return best
```

### LeetCode 42 — Trapping Rain Water

```python
def trap(height):
    lo, hi = 0, len(height) - 1
    left_max = right_max = 0
    ans = 0
    while lo < hi:
        if height[lo] < height[hi]:
            if height[lo] >= left_max:
                left_max = height[lo]
            else:
                ans += left_max - height[lo]
            lo += 1
        else:
            if height[hi] >= right_max:
                right_max = height[hi]
            else:
                ans += right_max - height[hi]
            hi -= 1
    return ans
```

### LeetCode 75 — Sort Colors (Dutch flag)

```python
def sortColors(nums):
    lo, mid, hi = 0, 0, len(nums) - 1
    while mid <= hi:
        if nums[mid] == 0:
            nums[lo], nums[mid] = nums[mid], nums[lo]
            lo += 1; mid += 1
        elif nums[mid] == 1:
            mid += 1
        else:
            nums[mid], nums[hi] = nums[hi], nums[mid]
            hi -= 1
```

### LeetCode 88 — Merge Sorted Array (back-to-front)

```python
def merge(nums1, m, nums2, n):
    i, j, k = m - 1, n - 1, m + n - 1
    while i >= 0 and j >= 0:
        if nums1[i] > nums2[j]:
            nums1[k] = nums1[i]
            i -= 1
        else:
            nums1[k] = nums2[j]
            j -= 1
        k -= 1
    while j >= 0:                 # if any nums2 elements remain
        nums1[k] = nums2[j]
        j -= 1; k -= 1
```

---

## 9. When NOT to use

- **Sort breaks the requirement** — if the problem needs original indices preserved, sort may break ordering; in 1. Two Sum you must return indices, hence the hashmap solution.
- **Subarray contiguity with negatives & sum constraints**: prefer prefix sum + hashmap (see `prefix_sum.md`).
- **You need order-independent lookup**: use a hash set rather than two pointers (when array is unsorted & can't be sorted).
- **You're searching for a single element** in sorted data: use binary search (see `binary_search.md`), not two pointers.
- **Constraint is not monotonic on shrinking**: container-with-water is monotonic (always hold the higher wall) but some variants aren't — be careful.
- **Window contiguity matters more than paired values**: sliding window is the better tool.

---

## 10. Common Mistakes

1. **Forgetting to sort first** for two-sum/three-sum variants — without sorting the two-pointer invariants break.
2. **Duplicate triples in 3Sum**: skip duplicates after each match `while lo < hi and nums[lo] == nums[lo - 1]: lo += 1` and same on `hi`.
3. **Off-by-one in opposite-direction loop**: `while lo < hi` (NOT `<=`). Both pointers should not cross.
4. **Wrong pointer movement: `"sum < target -> hi--"`** is reversed — too small, increase `lo`.
5. **Linked list slow-fast with empty list**: `if not head or not head.next: return False/None`.
6. **Modifying the slow value before storing**: in Move Zeroes you must **swap** to preserve the value being kicked out; just storing the new value loses it.
7. **In Merge Sorted Array, going front-to-back**: overwrites `nums1` elements. Always go back-to-front.
8. **Container with Most Water**: contracting the **taller** wall (you want to keep the wall with potential) is wrong; contract the **shorter** because the bottleneck is the shorter wall.
9. **Trapping Rain Water**: forgetting to update `left_max`/`right_max` *before* comparing.
10. **Comparing references** `slow is fast` — needed for `Optional[ListNode]` linked-list cycle (objects), not values.

---

## 11. Memory Tricks

- 🔑 **Slow-fast pointer** = the slow one writes; the fast one reads — one of "two pointers, one writer".
- 🔑 **Two-pointer sum**: if `sum < target`, raise `lo` (need bigger); if `sum > target`, drop `hi` (need smaller). "Smaller lo → smaller; bigger lo → bigger."
- 🔑 **Sort + two-pointer** is the **k-sum** pattern, generalized: outer loop on the "first" element, two-pointer on the rest, total O(n^(k-1)).
- 🔑 **Palindrome**: always symmetric — `l=0, r=n-1; s[l] == s[r]; l++, r--`. Done if `l >= r`.
- 🔑 **Container with water**: drop the shorter wall — by moving the smaller-height pointer inward, you *give up* the smallest chance of improvement.
- 🔑 **Trapping rain water**: drop the smaller wall — the trapped water is bounded by the **minimum** of the two outer maxes, so keep running maximums.

---

## 12. Interview Shortcuts

- **"Sorted"** + **"pair/triplet"** in the problem → **sort + two pointers** (or 3Sum, 4Sum).
- **"In-place"** + **"reorder / dedupe / partition"** → slow-fast two pointers.
- **"Detect cycle"** + linked list → Floyd's tortoise & hare.
- **"Linked list middle"** → slow-fast (slow goes one, fast goes two; slow is in the middle when fast hits end).
- **"Container / area / max with two boundaries"** → opposite-direction greedy.
- **"Remove duplicates from sorted"** → slow-fast same direction.
- For Sorted Linked List merge: use a **dummy head** with `tail` pointer, append whichever is smaller.
- For "trapping rain water" alt approach: dynamic prefix max arrays (left_max, right_max), 2-pointer is the O(1) space version.

---

## 13. Cheat Sheet Table

| Pattern                              | Code skeleton                                            | Time       | Space    |
|-------------------------------------|----------------------------------------------------------|------------|----------|
| Slow-fast in-place dedupe/partition | `if predicate: nums[slow]=nums[fast]; slow++`           | O(n)       | O(1)     |
| Floyd's cycle detection              | `slow=slow.next; fast=fast.next.next`                    | O(n)       | O(1)     |
| Sorted two-sum (167)                 | `while lo < hi: s=...; if< t: lo++; elif >: hi--`       | O(n)       | O(1)     |
| 3Sum (15)                            | sort + `for i; lo=i+1; hi=n-1; while lo<hi: ...`         | O(n²)      | O(1)     |
| Container (11)                       | opposite dirs, contract shorter wall                     | O(n)       | O(1)     |
| Trapping rain (42)                   | opposite dirs with left_max/right_max                    | O(n)       | O(1)     |
| Sort Colors (75) — Dutch flag        | three pointers lo/mid/hi                                 | O(n)       | O(1)     |
| Merge Sorted Array (88)              | back-to-front two pointers                               | O(n + m)   | O(1)     |
| Valid Palindrome (125)              | left/right symmetric                                     | O(n)       | O(1)     |

---

## 14. Time Complexity Table

| Algorithm                       | Time       | Time after sort | Space    |
|--------------------------------|------------|-----------------|----------|
| Slow-fast in-place partition   | O(n)       | —               | O(1)     |
| Floyd's tortoise & hare        | O(n)       | —               | O(1)     |
| Sorted two-sum                 | O(n)       | O(n log n) (incl. sort) | O(1) |
| 3Sum                            | O(n²)      | O(n²)+O(n log n) sort | O(1) |
| 4Sum                            | O(n³)      | —               | O(1)     |
| Container with water          | O(n)       | —               | O(1)     |
| Trapping rain water (2-pointer)| O(n)       | —               | O(1)     |
| Sort Colors                    | O(n)       | —               | O(1)     |
| Merge sorted array             | O(n + m)   | —               | O(1)     |
| Two-pointer palindrome         | O(n)       | —               | O(1)     |

---

## 15. Visual Diagram (ASCII)

### Opposite-direction pointers on a sorted array (two-sum)

```
  a = [-4, -1, -1, 0, 1, 2]   target = 0
  
  Step 1:  [-4, -1, -1, 0, 1, 2]
             ^                ^
             lo                hi    sum -4+2=-2 <0 -> lo++
  Step 2:  [-4, -1, -1, 0, 1, 2]
                  ^            ^
                  lo           hi    sum -1+2=1 >0 -> hi--
  Step 3:  [-4, -1, -1, 0, 1, 2]
                  ^         ^
                  lo         hi      sum -1+1=0 -> MATCH (i depends on the outer)
```

### Same-direction pointers (slow-fast in-place partition)

```
  nums = [0, 1, 0, 3, 12]
  slow = 0 (writes here)
  fast scans right:
  
   [0, 1, 0, 3, 12]
    s/f        (slow stays; nums[fast]==0, skip)
       f       swap with slow=0 -> [1, 0, 0, 3, 12], slow=1
          f    skip (nums[2]==0)
             f swap with slow=1 -> [1, 3, 0, 0, 12], slow=2
                f swap with slow=2 -> [1, 3, 12, 0, 0], slow=3
  Output: [1, 3, 12, 0, 0]
```

### Floyd's tortoise and hare on a cycled list

```
      1 -> 2 -> 3 -> 4
                ^       \
                |       |
                +-------+
  
   slow: 1, 2, 3, 4, 3, 4, 3, 4, ...
   fast: 1, 3, 3, 3, ...   meet at 3 — false example
   (actual meeting depends on the parity of cycle length; trust the algorithm)
```

### Container with most water (greedy contracting)

```
  height = [1, 8, 6, 2, 5, 4, 8, 3, 7]
   lo=0                          hi=8
   area = min(1,7)*8 = 8 -> advance lo (shorter)
       lo=1                     hi=8
   area = min(8,7)*7 = 49 -> advance hi (shorter)
       lo=1                  hi=7
   area = min(8,3)*6 = 18 -> advance hi
       ...
   best = 49
```

### Trapping rain water heights

```
  index:   0   1   2   3   4
  heights: 0   1   0   2   1
           |   |   v   |   v
                            water stalls on the lower of left_max and right_max
  
  Two-pointer: lo starts at 0, hi at 4.
   h[lo]=0 < h[hi]=1 -> process left side, update left_max=0; lo=1
   h[lo]=1 < h[hi]=1? no -> process right side, update right_max=1; hi=3
   h[lo]=1 < h[hi]=2 -> process left side: left_max=1; ans+=1-1=0; lo=2
   h[lo]=0 < h[hi]=2 -> process left side: ans += left_max - h[lo] = 1 - 0 = 1; lo=3
   lo==hi stop, ans = 1
```

### Flowchart — choosing flavor

```
              start: need pair / dedup / palindrome / cycle?
                              |
              sorted or sortable?
                 /                   \
              yes                    no
               |                      |
        pair/sum problem?         in-place dedup/partition?
         /               \           /                \
        k=2            k≥3       slow-fast         other
         |                |        (283, 75)
       opposite-dir      sort + (k-1) two-pointer
       two-pointer
```

---

## 16. Beginner Notes — Remember block

```
Remember:
- Two pointers means two indices that move under rules; both move forward (slow-fast) or toward each other (left-right).
- Opposite-direction pointers need SORTED input.
- Slow-fast pointers work in-place: slow writes, fast reads.
- Floyd's tortoise & hare for cycle detection in linked lists.
- Always `while lo < hi` (not `<=`) for opposite-direction pointers.
- For k-sum with k≥3, sort first and pointer-loop from outer inward.
- Container/water problems: contract the shorter side.
- In-place merge sorted array: go from the BACK to avoid overwrite.
```

---

## 17. FAANG Tips

1. **Recognize "sorted + pair/triplet" within 10 seconds** — sort + two pointers is the answer; you don't need a hash map.
2. **"In-place"** in the problem statement is a strong hint for slow-fast two-pointer (remove duplicates, move zeros, sort colors).
3. **Skipping duplicates** in 3Sum/4Sum: skip same values for both `i` (outer) and `lo`/`hi` after a successful match.
4. **Container With Most Water** is **unsorted** — both pointers work because area is `min(h[lo], h[hi]) * (hi - lo)` and contracting the smaller side never misses a better option.
5. **Trapping Rain Water (42)** is a 2-pointer optimization of the array's left_max/right_max precomputed DP.
6. **Floyd's cycle detection** is a must-know for linked lists: easy to code, harder to invent during interview.
7. **Merge Sorted Array (88)** from the back lets you merge `nums1` in place without overwriting unprocessed values.
8. **Sort Colors (75)** — Dutch national flag with three pointers: maintain `[0..lo)=0`, `[lo..mid)=1`, `(hi..n)=2`. Always increment `mid` only in the 0 and 1 cases; for the 2 case leave `mid` because the swapped element from `hi` is unknown.
9. **Palindrome verification** can be done after filtering non-alphanumerics; lowercase first.
10. **Two-pointer + sorting → O(n log n) total**; if the problem doesn't require in-place, **hash map** is O(n) and worth comparing.

---

## 18. Practice Problems

| Difficulty | Problem                                                                                              | Hint                                                |
|-----------|-------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Easy      | [283 Move Zeroes](https://leetcode.com/problems/move-zeroes/)                                        | Slow-fast same direction                            |
| Easy      | [26 Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array/) | Slow-fast dedup                          |
| Easy      | [125 Valid Palindrome](https://leetcode.com/problems/valid-palindrome/)                              | Opposite-direction                                  |
| Easy      | [167 Two Sum II — Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/)           | Classic opposite-direction                          |
| Easy      | [88 Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array/)                           | Back-to-front two pointers                          |
| Medium    | [11 Container With Most Water](https://leetcode.com/problems/container-with-most-water/)             | Greedy opposite-direction                           |
| Medium    | [15 3Sum](https://leetcode.com/problems/3sum/)                                                       | Sort + two-pointer per first index                  |
| Medium    | [75 Sort Colors](https://leetcode.com/problems/sort-colors/)                                          | Three-pointer Dutch flag                            |
| Hard      | [42 Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/)                          | 2-pointer with left_max/right_max                   |
| Hard      | [30 Substring with Concatenation of All Words](https://leetcode.com/problems/substring-with-concatenation-of-all-words/) | Sliding window + two-pointer                |

---

**Cross-links**: [sliding_window.md](./sliding_window.md) (var overlaps) · [sorting.md](./sorting.md) (always sort first for k-sum) · [binary_search.md](./binary_search.md) (single-element search vs pair search) · [searching.md](./searching.md) (hash-based pair alternatives).