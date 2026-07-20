# Lists in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 02 — Data Types
> 🔗 Related: [strings.md](./strings.md) · [tuple.md](./tuple.md) · [numbers.md](./numbers.md)
> Algorithms: [two_pointers.md](../07_Algorithms/two_pointers.md) · [sliding_window.md](../07_Algorithms/sliding_window.md)
> Back to [README](../README.md)

---

## 1. What is it?

A **list** is a **mutable, ordered sequence** of arbitrary Python objects, written with square brackets.

```python
l = [1, "two", 3.0, [4]]   # heterogeneous, nested
```

Internally, a list is a **dynamic array of pointers** (PyObject references), so:
- Indexed access is **O(1)** amortized.
- Append is **O(1)** amortized (over-allocated buffer; doubling strategy).
- Insert at front / middle is **O(n)** (shift elements).
- Elements are **references**, not values — copying is shallow by default.

---

## 2. Why do we use it?

- General-purpose **ordered, mutable** collection.
- Default for stacks (`append`/`pop`), queues (use `collections.deque` instead for O(1) popleft), matrices, sliding windows, adjacency lists, etc.
- Supports slicing, comprehensions, and rich methods — fits most algorithm prototypes.

---

## 3. When should I choose it?

| Need                              | Use                       |
|-----------------------------------|---------------------------|
| Ordered, mutable sequence         | `list`                    |
| Immutable fixed sequence           | `tuple` (hashable, safe)  |
| Fast lookup / dedupe               | `set`                     |
| Key -> value                       | `dict`                    |
| Fast queue / deque                 | `collections.deque`       |
| Numeric / scientific              | `numpy.ndarray`           |
| Heterogeneous fixed record         | `tuple` / `dataclass`     |
| Text                               | `str`                     |
| Bytes                             | `bytes` / `bytearray`     |

---

## 4. Syntax

```python
l = []                     # empty
l = [1, 2, 3]
l = list("abc")            # ['a','b','c']
l = list(range(5))         # [0,1,2,3,4]
l = [0] * 5                # [0,0,0,0,0]
l = [x*x for x in range(5)]# comprehension -> [0,1,4,9,16]
l = [[0]*3 for _ in range(3)]  # 2D list (avoid [[0]*3]*3 — shared rows!)
l[0], l[-1], l[1:3], l[::-1]  # index & slice
l + [4]                    # new list
```

---

## 5. Basic Example

```python
nums = [3, 1, 4, 1, 5, 9]
nums.append(2)             # [3,1,4,1,5,9,2]
nums.sort()                # [1,1,2,3,4,5,9]
nums.count(1)              # 2
nums.index(4)              # 3
nums.pop()                 # 9 (returned; removed)
nums.reverse()             # in-place
nums[1:4]                  # [5,4,3]
```

---

## 6. Step-by-Step Dry Run

**Two Sum (1. Two Sum)** returning indices:

```python
def twoSum(nums, target):
    seen = {}
    for i, n in enumerate(nums):
        if target - n in seen:
            return [seen[target - n], i]
        seen[n] = i
```

Dry run `nums=[2,7,11,15]`, `target=9`:

```
i=0 n=2  need=7   seen={}          store seen={2:0}
i=1 n=7  need=2   seen has 2 (idx 0) -> return [0,1]
```

ASCII flow:
```
  nums: [ 2,  7, 11, 15 ]
          i->          seen: {2:0}
             i->       check 7 -> 9-7=2 in seen -> return [0,1]
```

---

## 7. Built-in Methods   (EXHAUSTIVE)

### 7.1 `list.append(x)`
- Adds `x` to the **end**.
- Time: **O(1) amortized**.
- Example: `l=[1]; l.append(2)` -> `[1,2]`.
- Mistakes: `append([1,2])` adds the list as one element vs `extend`.

### 7.2 `list.extend(iterable)`
- Appends each item of iterable.
- Time: **O(k)** where k = len(iterable).
- Example: `l=[1]; l.extend([2,3])` -> `[1,2,3]`; `l.extend("ab")` -> `[1,2,3,'a','b']`.
- Mistakes: `extend` flattens one level only.

### 7.3 `list.insert(i, x)`
- Inserts `x` before index `i` (negative indices supported).
- Time: **O(n)** (shifts elements).
- Example: `[1,2,3].insert(0, 9)` -> `[9,1,2,3]`.
- Mistakes: inserting in a tight loop is O(n^2) — build with `append`+`reverse` if needed.

### 7.4 `list.remove(x)`
- Removes **first** occurrence of value `x`. `ValueError` if absent.
- Time: **O(n)** (search + shift).
- Example: `[1,2,1].remove(1)` -> `[2,1]`.
- Mistakes: iterating + `remove` skips elements.

### 7.5 `list.pop([i])`
- Removes and returns element at index `i` (default last).
- Time: **O(1)** at end; **O(n)** at front/middle.
- Example: `l=[1,2,3]; l.pop()` -> `3`; `l` -> `[1,2]`.
- Interview use: stack via `append`/`pop`.

### 7.6 `list.clear()`
- Empties the list in place.
- Time: **O(n)** (drops refs).
- Example: `l=[1,2]; l.clear()` -> `[]`.

### 7.7 `list.index(x[, start[, stop]])`
- Returns index of first `x`; else `ValueError`.
- Time: **O(n)**.
- Example: `''.join` style: `[1, 2, 3].index(2)` -> `1`.

### 7.8 `list.count(x)`
- Counts occurrences of value.
- Time: **O(n)**.
- Example: `[1,2,1].count(1)` -> `2`.

### 7.9 `list.sort(*, key=None, reverse=False)`
- In-place stable sort.
- Time: **O(n log n)**; space **O(n)** (Timsort).
- Example:
  ```python
  words = ["bb","a","ccc"]
  words.sort(key=len)          # ['a','bb','ccc']
  words.sort(key=lambda s: (len(s), s))  # multi-key
  words.sort(reverse=True)
  ```
- Interview use: sort by freq `sort(key=lambda x:(-cnt[x], x))`.
- Mistakes: `sort` returns `None`; use `sorted(l)` for new list.

### 7.10 `list.reverse()`
- Reverses in place.
- Time: **O(n)**.
- Example: `[1,2,3].reverse()` -> `[3,2,1]`; returns `None`.

### 7.11 `list.copy()`
- **Shallow copy** (new list, same element refs).
- Time: **O(n)**.
- Example: `l2 = l.copy()` equals `l2 = l[:]`.
- Mistakes: nested mutables still shared.

### 7.12 `del list[i]` / `del list[a:b]`
- Statement (not method) — removes by index/slice.
- Example:
  ```python
  l = [1,2,3,4]
  del l[0]      # [2,3,4]
  del l[::2]    # remove every other
  ```
- Time: O(n).

### 7.13 `len(l)`
- Built-in (not method) — returns element count.
- Time: **O(1)**.

### 7.14 `x in l` / `x not in l`
- Membership test (linear scan).
- Time: **O(n)**. Use sets for O(1).

### 7.15 `sorted(iterable, *, key=None, reverse=False)`
- Returns a **new** sorted list.
- Example: `sorted("bca", key=lambda c:-ord(c))` -> `['c','b','a']`.

### 7.16 `reversed(seq)`
- Returns iterator. `list(reversed([1,2,3]))` -> `[3,2,1]`.

### 7.17 Slicing `l[start:stop:step]`
- Negatives allowed. **Returns new list.**
- Examples:
  ```python
  l=[0,1,2,3,4,5]
  l[1:4]        # [1,2,3]
  l[::-1]       # reverse
  l[::2]        # [0,2,4]
  l[-2:]        # last two
  l[1:5:2]      # [1,3]
  ```
- Time: **O(k)** where k = slice length.

### 7.18 List Comprehension
```python
[x for x in range(10) if x % 2 == 0]
[(x, y) for x in range(3) for y in range(3) if x != y]
[len(s) for s in ["a","bb","ccc"]]
```
Time: O(n * cost-body).

### 7.19 Nested Lists
```python
matrix = [[1,2,3],
          [4,5,6],
          [7,8,9]]
matrix[1][2] = 6
```
Tip: avoid `[[0]*n]*m` (shared rows). Use `[[0]*n for _ in range(m)]`.

### 7.20 Sorting: key, reverse, lambda
```python
pairs = [(1,'b'),(2,'a')]
pairs.sort(key=lambda p: p[1])              # by string
pairs.sort(key=lambda p: (-p[0], p[1]))    # numeric desc, str asc
students = [{'age':20,'gpa':3.5}]
students.sort(key=lambda s: (-s['gpa'], s['age']))
```

### 7.21 `copy` vs `deepcopy`
```python
import copy
a = [[1,2],[3,4]]
b = a                 # same object
b = a[:]              # shallow copy (rows shared)
b = a.copy()          # shallow copy (rows shared)
b = copy.deepcopy(a)  # deep copy (rows independent)
b[0][0] = 99
```

ASCII of shallow vs deep:
```
shallow copy            deep copy
a -> [ row0, row1 ]     a -> [ row0, row1 ]
b -> [ row0, row1 ]     b -> [ row0', row1' ]  (separate)
        ^^^^^                                ^^^^^^
        shared                                distinct
```

### 7.22 Other helpers
| Function         | Purpose                       |
|------------------|-------------------------------|
| `min(l), max(l)` | smallest / largest            |
| `sum(l[, start])`| sum of elements               |
| `any(l), all(l)` | any/all truthy                |
| `enumerate(l)`   | (index, val) pairs            |
| `zip(a, b)`      | paired tuples                 |
| `map(f, l)`      | iterator of f(items)          |
| `filter(p, l)`   | iterator of items where p     |
| `len(l)`         | count                         |

---

## 8. Interview Example

**LeetCode 1 – Two Sum** (already shown). Concurrency bonus:

**LeetCode 15 – 3Sum** via sorting + two pointers:
```python
def threeSum(nums):
    nums.sort()
    res = []
    for i in range(len(nums)-2):
        if i > 0 and nums[i] == nums[i-1]: continue  # skip duplicates
        l, r = i+1, len(nums)-1
        while l < r:
            s = nums[i] + nums[l] + nums[r]
            if s == 0:
                res.append([nums[i], nums[l], nums[r]])
                while l < r and nums[l] == nums[l+1]: l += 1
                while l < r and nums[r] == nums[r-1]: r -= 1
                l += 1; r -= 1
            elif s < 0: l += 1
            else: r -= 1
    return res
```

---

## 9. When NOT to use

- Need O(1) membership / dedupe → `set`.
- Need O(1) key lookup → `dict`.
- Need O(1) front inserts / popleft → `collections.deque`.
- Fixed size & immutable → `tuple`.
- Heavy numeric math → `numpy.ndarray`.
- Multithreaded shared buffer → `queue.Queue` / `multiprocessing.Queue`.
- Memory-tight huge dataset → `array.array` or `numpy`.

---

## 10. Common Mistakes

```python
# 1. [[0]*n]*m — shared rows
m = [[0]*3] * 3
m[0][0] = 1
print(m)            # [[1,0,0],[1,0,0],[1,0,0]]  (all rows same!)
# Fix:  m = [[0]*3 for _ in range(3)]

# 2. Modifying while iterating
l = [1,2,3,4]
for x in l:
    if x % 2 == 0: l.remove(x)        # SKIPS items!

# 3. Default mutable args
def f(x, cache=[]):
    cache.append(x); return cache      # shared across calls!
# Fix: cache=None then if cache is None: cache = []

# 4. sort vs sorted
l = [3,1,2]
new = l.sort()      # new is None

# 5. Shallow copy gotcha
a = [[1,2]]
b = a.copy()
b[0].append(3)
print(a)            # [[1,2,3]] rows shared!

# 6. Insert at front in a loop -> O(n^2)
# Use deque for queues.

# 7. extend vs append
l = [1]; l.append([2,3])  # [1,[2,3]]
l = [1]; l.extend([2,3])  # [1,2,3]

# 8. remove with duplicates during iteration
# Safest: build a new list (comprehension), then reassign.
```

---

## 11. Memory Tricks

- **List = "order matters"**: when index / order is essential.
- **append = "tail"**; **insert = "costly"**.
- **copy is shallow**; **deepcopy is the clone capsule**.
- **`[[0]*n]*m` is evil twin**; prefer `[[0]*n for _ in range(m)]`.
- **sort returns None**; **sorted returns new**.
- **`extend` flattens one level.**

---

## 12. Interview Shortcuts

- Sliding window: keep list-slice(); avoid slicing — track pointers & a counter dict.
- Two pointers: `sort()` first, then expand inward.
- Frequency: `collections.Counter(l)`; or `[0]*26` for lowercase letters.
- Reverse without mutating: `l[::-1]`.
- Pair / group: `zip(a, b)` -> iterator of tuples.
- Stack: `append`+`pop`; queue: **don't use list** — use `deque`.
- Index-value swap: `l[i], l[j] = l[j], l[i]` (no temp).
- Dedupe preserving order: `list(dict.fromkeys(l))` (3.7+ dict ordered).

---

## 13. Cheat Sheet Table

| Method / Op          | Use                              |
|----------------------|----------------------------------|
| `append(x)`          | add to end                       |
| `extend(it)`         | add many                         |
| `insert(i,x)`        | insert at i                      |
| `remove(x)`          | remove first equal               |
| `pop([i])`           | remove + return at i / last      |
| `clear()`            | empty                            |
| `index(x)`           | index of first x                 |
| `count(x)`           | count x                          |
| `sort(key,reverse)`  | in-place sort                    |
| `reverse()`          | in-place reverse                 |
| `copy()`             | shallow copy                     |
| `del l[i]` / `del l[:]` | delete by index/slice         |
| `len(l)`             | length                           |
| `x in l`             | membership                       |
| `sorted(l)`          | new sorted list                  |
| `l[a:b:c]`           | slice (new)                      |
| `l + r`              | concatenate (new)                |
| `l * k`              | repeat                           |
| unpack `a,b,*rest=l` | spread                           |
| `[expr for x in iter]` | comprehension                   |
| `copy.deepcopy(l)`   | deep copy                        |

---

## 14. Time Complexity Table

| Operation                  | Complexity        | Notes                          |
|----------------------------|-------------------|--------------------------------|
| `l[i]` / `l[i] = v`        | O(1)              |                                |
| `l.append(x)`              | O(1) amortized    | occasional resize              |
| `l.pop()` (last)           | O(1)              |                                |
| `l.pop(0)` / `insert(0,x)`| O(n)              | shifts all elements            |
| `del l[i]`                 | O(n)              |                                |
| `l.remove(x)`              | O(n)              | search + shift                 |
| `l.extend(it)`             | O(k)              | k = len(it)                    |
| `l.count(x)` / `l.index(x)`| O(n)              |                                |
| `x in l`                   | O(n)              | linear scan                    |
| `len(l)`                   | O(1)              | cached                         |
| `l[a:b:c]`                 | O(k)              | k = slice size                 |
| `l + r`                    | O(n+m)            | new list copy                  |
| `l * k`                    | O(n*k)            |                                |
| `l.sort()`                 | O(n log n)        | Timsort; stable                |
| `sorted(l)`                | O(n log n)        | new list                       |
| `l.reverse()`              | O(n)              | in place                        |
| `max/min/sum`              | O(n)              |                                |
| `copy.deepcopy(l)`         | O(n)              | of structure                   |

---

## 15. Visual Diagram (ASCII)

### List memory + indices
```
Index (pos):    0   1   2   3   4
              +---+---+---+---+---+
list:         | 10| 20| 30| 40| 50|
              +---+---+---+---+---+
Neg index:   -5  -4  -3  -2  -1

append ->     [10 20 30 40 50 60]    O(1) amortized  (capacity doubling)
insert(0,9)-> [9 10 20 30 40 50]      O(n) shift
pop() ->      [10 20 30 40]           O(1) (returns removed value)
pop(0) ->     [20 30 40 50]           O(n)
```

### Underlying array of references
```
list object   [ obj*, obj*, obj*, obj*, obj* ]   (over-allocated)
                 │    │    │    │
                 v    v    v    v
                10   20   30 "ab"  <-- heterogeneous objects
```

### 2D list (rectangle)
```
matrix:  row0 -> [1,2,3]
         row1 -> [4,5,6]
         row2 -> [7,8,9]
matrix[1][2] -> 6
```

### Stack via append/pop                                 
```
push: 3  -> [3]
push 5  -> [3,5]
peek      -> 5 (top)
pop      -> 5 returned; [3]
```

---

## 16. Beginner Notes

> Remember:
> - Lists are **mutable, ordered, heterogeneous**.
> - Indexing/slicing returns views for numpy, **copies** for list.
> - `append` adds one item; `extend` / `+=` add many.
> - `sort()` mutates and returns `None`; `sorted()` returns a new list.
> - Default mutable args (`def f(x=[])`) are shared — pass `None`.
> - To clone nested lists use `copy.deepcopy`.
> - Use `deque` for queue / front inserts.

---

## 17. FAANG Tips

- Sliding window: maintain `[left, right]` pointers and an auxiliary dict/array; **don't re-slice** each step.
- For two pointers, **sort first** for O(n log n) — beats nested loops.
- For permutations / combinations, use `itertools` (`permutations`, `combinations`, `accumulate`).
- For monotonic queue patterns (LeetCode 239 Sliding Window Max), use `deque` with stored indices.
- `heapq` turns a list into a heap in place — `heapify`, `heappush`, `heappop` — O(n), O(log n), O(log n).
- `Counter` + `most_common(k)` for top-k.
- For huge datasets prefer generator pipelines over list materialization.

---

## 18. Practice Problems

**Easy**
1. LeetCode 217 – Contains Duplicate (set)
2. LeetCode 26 – Remove Duplicates from Sorted Array (two pointers)
3. LeetCode 1 – Two Sum
4. LeetCode 121 – Best Time to Buy and Sell Stock
5. LeetCode 283 – Move Zeroes

**Medium**
6. LeetCode 15 – 3Sum (sort + two pointers)
7. LeetCode 11 – Container With Most Water
8. LeetCode 238 – Product of Array Except Self
9. LeetCode 56 – Merge Intervals
10. LeetCode 75 – Sort Colors (Dutch flag)

**Hard**
11. LeetCode 239 – Sliding Window Maximum (deque)
12. LeetCode 42 – Trapping Rain Water
13. LeetCode 84 – Largest Rectangle in Histogram (monotonic stack)

---

> Next: [tuple.md](./tuple.md) · [numbers.md](./numbers.md) · Back to [README](../README.md)