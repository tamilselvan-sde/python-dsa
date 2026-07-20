# heapq

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
> Section: 06 — Collections
> 🔗 Related: [counter.md](./counter.md) · [defaultdict.md](./defaultdict.md) · [deque.md](./deque.md) · [../07_Algorithms/heap.md](../07_Algorithms/heap.md) · [../07_Algorithms/sliding_window.md](../07_Algorithms/sliding_window.md)
> Back to [README](../README.md)

---

## 1. What is it?

`heapq` is a built-in **module** (not a class) providing heap operations on a regular Python `list`. It maintains the **min-heap invariant**:

```
heap[k] <= heap[2*k + 1]  and  heap[k] <= heap[2*k + 2]
```

So the list, when reinterpreted as a complete binary tree, has every parent ≤ its children. The **smallest element is always at index 0**.

```python
import heapq
h = [5, 3, 8, 1]
heapq.heapify(h)         # h becomes [1, 3, 8, 5]  (a valid min-heap)
heapq.heappush(h, 0)     # h[0] is now 0
heapq.heappop(h)         # 0  → re-heapify to keep min on top
```

Important facts:
- `heapq` does not define a `Heap` class — it operates **in place** on a list.
- The list's order is **NOT sorted** — only the heap invariant holds.
- Always a **min-heap**. For max-heap, negate values (or wrap items).
- **Tuple comparison**: when items are tuples, heap compares element-by-element, so the **first element is the priority**.

---

## 2. Why do we use it?

When we need **efficient access to the smallest (or k smallest) elements** in a stream, without sorting everything.

Operations comparison:

| Operation            | `sorted(lst)`  | `heapq`             |
|----------------------|----------------|---------------------|
| Get min/max          | O(n) lookup    | O(1) (heap[0])      |
| Insert new element   | O(n) re-sort   | O(log n)            |
| Remove min           | O(n)           | O(log n)            |
| Get k smallest       | O(n log n)     | **O(n log k)**      |
| Build heap           | O(n log n) sort| **O(n)** heapify    |

`heapq` powers:
- **Priority queues** — Dijkstra, A*.
- **K-th largest / smallest** — Quickselect alternative, robust at small k.
- **Top-K over streams** — store just k items, never hold everything in memory.
- **Merge sorted** streams (`heapq.merge` — generator, O(1) memory advance).
- **Median of stream** (#295) — two-heap trick.

---

## 3. When should I choose it?

| Situation                                          | Best tool                |
|----------------------------------------------------|--------------------------|
| Get k smallest/largest of n elements              | ✅ `heapq.nsmallest/nlargest` |
| Need repeated extract-min over a stream           | ✅ `heapq` priority queue  |
| Dijkstra / A* shortest paths                     | ✅ `heapq`                 |
| Medium of stream                                  | ✅ Two heaps              |
| Need O(1) FIFO                                     | `deque` ([deque.md](./deque.md)) |
| Need top-k by frequency in one shot                | `Counter.most_common` ([counter.md](./counter.md)) |
| Counting/grouping                                  | `Counter` / `defaultdict` |
| Sorted insert / binary search                     | `bisect`                 |
| Need to fully sort                                 | `sorted()`               |
| Need a max-heap                                     | `heapq` + negate         |
| Thread-safe priority queue                        | `queue.PriorityQueue`    |

**Decision flow:**

```
   Need to repeatedly get MIN?
              │
              YES → heapq (store list as heap)
              │ NO
              │
   Get top-k once, all data available?
              │
              YES → heapq.nsmallest/nlargest  (or sorted + slice for tiny k)
              │ NO
              │
   Pure FIFO/LIFO?
              │
              YES → deque
              │
   Sorted (read-only) → bisect / sorted
```

---

## 4. Syntax

```python
import heapq

# In-place operations on a list `h`
heapq.heapify(h)                    # O(n) rearrange into min-heap
heapq.heappush(h, x)                # O(log n) push
heapq.heappop(h)                    # O(log n) pop & return min
heapq.heappushpop(h, x)             # push then pop (efficient single siftup)
heapq.heapreplace(h, x)             # pop then push (always returns previous min)

# Query-only (do NOT mutate)
heapq.nsmallest(k, it, key=None)    # k smallest (lazy iterable OK)
heapq.nlargest(k, it, key=None)     # k largest
heapq.merge(*iterables, key=None, reverse=False)  # merge sorted iterables → iter

# Underlying helpers (usually not for direct use)
heapq._siftdown(h, i)
heapq._siftup(h, i)
```

Max-heap idiom: push `(−value, item)`.

---

## 5. Basic Example

```python
import heapq

nums = [7, 1, 5, 3, 9, 2]
heapq.heapify(nums)
print(nums)           # [1, 3, 2, 7, 9, 5]  ← heap order, NOT sorted

print(heapq.heappop(nums))   # 1
print(heapq.heappop(nums))   # 2
print(heapq.heappop(nums))   # 3

heapq.heappush(nums, 0)
print(nums[0])              # 0   (min always at root)

print(heapq.nsmallest(2, [9, 1, 5, 3, 7]))   # [1, 3]
print(heapq.nlargest(2,  [9, 1, 5, 3, 7]))   # [9, 7]

# Max-heap via negation
maxh = []
for x in [4, 1, 7, 3]:
    heapq.heappush(maxh, -x)
print(-heapq.heappop(maxh))   # 7
```

---

## 6. Step-by-Step Dry Run

Initial list `[5, 3, 8, 1, 2]`, run `heapify`:

```
Step 0:  list = [5, 3, 8, 1, 2]   tree:
                                   5
                                 /   \
                                3     8
                               / \
                              1   2

heapify runs sift-down from parent of last index leftwards:

Last index = 4, parent = (4-1)//2 = 1   (value 3)
   sift-down(1): swap 3 ↔ 1 (its smallest child)
        now:  [5, 1, 8, 3, 2]
                   5
                 /   \
                1     8
               / \
              3   2

Parent 0 = (value 5)
   sift-down(0): swap 5 ↔ 1 (smallest of children 1,8)
        now:  [1, 5, 8, 3, 2]
        then recursively — sift-down(1) of new subtree:
        swap 5 ↔ 2 (smallest child of node 1, which has children 3 and 2)
        now:  [1, 2, 8, 3, 5]

Final heap tree:
                   1
                 /   \
                2     8
               / \
              3   5

list = [1, 2, 8, 3, 5]   ← valid min-heap
```

After `heappush(h, 0)`:

```
append 0 last  → [1, 2, 8, 3, 5, 0]
then sift-up(index 5, value 0):
   parent (5-1)//2 = 2  (value 8)   0 < 8 → swap
   parent (2-1)//2 = 0 (value 1)   0 < 1 → swap
   stop at root
Final: [0, 1, 8, 3, 5, 2]
```

Then `heappop(h)` returns `0`, replaces root with last `2`, sifts-down.

---

## 7. Built-in Methods   (EXHAUSTIVE)

### `heapq.heapify(x)`
- **Purpose:** Transform list `x` in place into a min-heap, in O(n).
- **Syntax:** `heapq.heapify(x)`
- **Input:** list of comparable items
- **Output:** None (mutates)
- **Example:**
  ```python
  h = [4, 1, 3, 2]
  heapq.heapify(h)          # h == [1, 2, 3, 4]
  ```
- **Time:** **O(n)** — faster than n pushes which is O(n log n). Uses sift-down bottom-up.
- **Interview use:** turn any list into a priority queue quickly. Prefer `heapify` over repeated `heappush` for bulk loading.
- **Common mistake:** assuming the list is now sorted — it's just heap-ordered.
- **Shortcut:** `heapify` exists for bulk setup; for streaming, just keep pushing.

### `heapq.heappush(heap, item)`
- **Purpose:** Insert item maintaining heap invariant.
- **Syntax:** `heapq.heappush(heap, item)`
- **Input:** heap (list), item
- **Output:** None (mutates)
- **Example:** `heapq.heappush(h, 0)` on heap `[1,3,2]` → `[0,1,2,3]`
- **Time:** **O(log n)** — sift-up from leaf.
- **Interview use:** streaming priority queue.
- **Common mistake:** pushing tuples whose first element is the data, not the priority — comparisons break.
- **Shortcut:** push `(priority, tie_breaker, value)` to fully resolve ties.

### `heapq.heappop(heap)`
- **Purpose:** Remove and return smallest item.
- **Syntax:** `heapq.heappop(heap)`
- **Input:** heap list
- **Output:** smallest item
- **Example:** `heapq.heappop([1,3,2])` returns `1`; heap becomes `[2,3]`.
- **Time:** **O(log n)** — replaces root with last leaf, sifts down.
- **Interview use:** extract-min in Dijkstra, raw priority queue.
- **Common mistake:** calling on empty heap raises `IndexError`.
- **Shortcut:** to peek min without pop: `heap[0]` (O(1)).

### `heapq.heappushpop(heap, item)`
- **Purpose:** Push `item`, then pop and return the smallest. **More efficient** than separate push+pop because it sifts up and then immediately pops the same node.
- **Syntax:** `heapq.heappushpop(heap, item)`
- **Time:** O(log n) — but in practice faster than `push + pop`.
- **Example:**
  ```python
  h = [1, 3, 2]
  heapq.heappushpop(h, 4)   # returns 1, h becomes [2, 3, 4]
  ```
- **Interview use:** sliding window of size k where each new item replaces the smallest.
- **Common mistake:** if item is smaller than heap min, the result is the item itself — the heap is unchanged.
- **Shortcut:** when item < heap[0], `heappushpop` returns item directly; just use that.

### `heapq.heapreplace(heap, item)`
- **Purpose:** Pop the smallest, then push `item`. Always returns a value, even if item is smaller (unlike `heappushpop`).
- **Syntax:** `heapq.heapreplace(heap, item)`
- **Time:** O(log n).
- **Example:**
  ```python
  h = [1, 3, 2]
  heapq.heapreplace(h, 4)   # returns 1, h becomes [2, 3, 4]
  heapq.heapreplace(h, 0)   # returns 2, h becomes [0, 3, 4]
  ```
- **Interview use:** keep heap size constant (top-k by replacing); LFU cache.
- **Common mistake:** calling on empty heap raises `IndexError`. Differs from `heappushpop` when item < heap[0]: `heapreplace` returns the old min and stores the small item, growing the heap-or-not based on size; `heappushpop` returns the item directly.
- **Shortcut:** for fixed-size top-k, prefer `heapreplace` (pop first, ensures size stays the same).

### `heapq.nsmallest(k, iterable, key=None)`
- **Purpose:** Return the k smallest items (as a sorted list ascending). Lazy: doesn't fully load the iterable.
- **Syntax:** `heapq.nsmallest(k, iterable, key=None)`
- **Time:** **O(n log k)** — uses a max-heap internally of size k.
- **Example:**
  ```python
  heapq.nsmallest(3, [9, 1, 5, 3, 7])   # [1, 3, 5]
  heapq.nsmallest(2, ['apple','dog','cat'], key=len)   # ['cat','dog']
  ```
- **Interview use:** Top-K without mutating the source.
- **Common mistake:** for k close to n, just use `sorted(it)[:k]` (faster).
- **Shortcut:** `nsmallest(1, it)` is equivalent to `min(it)` (slower — use `min` directly).

### `heapq.nlargest(k, iterable, key=None)`
- **Purpose:** Return the k largest items (sorted descending). Lazy.
- **Time:** O(n log k) with min-heap of size k.
- **Example:** `heapq.nlargest(2, [9,1,5,3,7])` → `[9, 7]`
- **Interview use:** leaderboard; "k-th largest element".
- **Common mistake:** confusing order — `nlargest` returns descending (largest first), `nsmallest` returns ascending.
- **Shortcut:** `nlargest(1, it)` == `max(it)` (use min/max instead).

### `heapq.merge(*iterables, key=None, reverse=False)`
- **Purpose:** Merge multiple **already-sorted** iterables into a single sorted iterator. Memory: O(k) where k is number of iterables, NOT their total size.
- **Syntax:** `heapq.merge(it1, it2, ...)`
- **Time:** O(N log k) overall where N total elements, k iterables.
- **Example:**
  ```python
  it1 = [1, 4, 7]; it2 = [2, 5, 8]; it3 = [3, 6, 9]
  list(heapq.merge(it1, it2, it3))   # [1..9]
  ```
- **Interview use:** external merge-sort; merge k sorted lists without holding everything.
- **Common mistake:** passing UN-sorted iterables — output is undefined garbage.
- **Shortcut:** way better than `sorted(it1+it2+it3)` for huge arrays — `merge` is a greedy generator.

### Tuple-comparison heap behavior
- **Purpose:** Heap compares items via `<`; for tuples, comparison is **lexicographic by position**.
- **Consequence:** the **first element is the priority**. To break ties, include a counter or unique id in the tuple.
- **Example:**
  ```python
  h = []
  import itertools; tie = itertools.count()
  heapq.heappush(h, (priority, next(tie), task))
  ```
- **Common mistake:** when priorities tie, heap compares the second tuple element; if those are non-comparable (e.g. dicts), you get `TypeError`. Always include a comparable tie-breaker.
- **Shortcut:** use `(priority, key_or_index, value)` form universally.

### Max-heap trick (negate values)
- **Purpose:** `heapq` only does min-heap. To get max-heap behavior, **negate priorities** (works for numbers).
- **Example:**
  ```python
  import heapq
  h, max_val = [], 0
  for x in [3, 1, 4, 1, 5]:
      heapq.heappush(h, -x)
  while h:
      print(-heapq.heappop(h))   # 5, 4, 3, 1, 1
  ```
- **Common mistake:** forgetting to negate again on `pop` — both push and pop need sign-twisting.
- **Shortcut:** for non-numeric priorities, wrap items in a class with a custom `__lt__` that is reversed.

---

## 8. Interview Example

**Kth Largest Element in an Array (#215):**

```python
import heapq
def findKthLargest(nums, k):
    return heapq.nlargest(k, nums)[-1]

# Equivalent with a min-heap of size k:
def findKthLargest_heap(nums, k):
    h = nums[:k]
    heapq.heapify(h)                     # min-heap of size k
    for x in nums[k:]:
        if x > h[0]:
            heapq.heapreplace(h, x)       # drop smallest, add x
    return h[0]                            # root is the k-th largest
```

**Two-heap Median of Stream (#295):**

```python
import heapq
class MedianFinder:
    def __init__(self):
        self.lo = []   # max-heap (store negated) — biggest of small half on top
        self.hi = []   # min-heap — smallest of large half on top
    def addNum(self, num):
        heapq.heappush(self.lo, -num)
        heapq.heappush(self.hi, -heapq.heappop(self.lo))   # balance
        if len(self.lo) < len(self.hi):
            heapq.heappush(self.lo, -heapq.heappop(self.hi))
    def findMedian(self):
        if len(self.lo) > len(self.hi):
            return -self.lo[0]
        return (-self.lo[0] + self.hi[0]) / 2
```

**Merge K Sorted Lists (#239):**

```python
def mergeKLists(lists):
    import heapq, itertools
    h = []
    tie = itertools.count()
    for i, node in enumerate(lists):
        if node: heapq.heappush(h, (node.val, next(tie), node))
    dummy = cur = ListNode()
    while h:
        _, _, n = heapq.heappop(h)
        cur.next = n
        cur = cur.next
        if n.next:
            heapq.heappush(h, (n.next.val, next(tie), n.next))
    return dummy.next
```

---

## 9. When NOT to use

- You need full sort — `sorted()` is simpler.
- You only need min/max — `min()`, `max()` O(n) is fine.
- You need an **arbitrary-priority queue with decrease-key** — heapq has no built-in `decrease_key`; either rebuild or use `dict` to track indices manually.
- You need a **max-heap on string/custom objects** — yes you can negate numbers, but custom objects need an inverted `__lt__` or wrapping; awkward.
- You need **thread-safe** multi-producer queues — `queue.PriorityQueue` wraps `heapq` with locks.
- You need **O(1) lookup by key** — use `dict`.
- You need FIFO behavior — use `deque` ([deque.md](./deque.md)).

---

## 10. Common Mistakes

| Mistake                                                                    | Fix                                                    |
|----------------------------------------------------------------------------|--------------------------------------------------------|
| Treating the heap list as sorted                                          | it's only `heap[0]` that's guaranteed min.            |
| Popping the **second-smallest** by `heap[1]`                               | `heap[1]` is just one of the children — may not be #2. |
| Pushing tuples where first element is not comparable                        | wrap in `(priority, tie, value)` with comparable tie.   |
| Forgetting to negate for max-heap                                          | both push and pop need `-`.                            |
| Adding tied priorities that throw on next-element comparison               | insert `itertools.count()` as a tie-breaker.          |
| Using `heappushpop` vs `heapreplace` when item < heap[0]                  | revisit the semantics table below.                     |
| Calling `heappop` on empty heap                                            | check `if h:` first.                                    |
| Calling `heapify` after each push expecting it to "fix"                    | unnecessary — `heappush` already maintains invariant. |
| Assuming `nlargest` and `sorted(...)[:k]` are identical                   | tie-breaking may differ; use `key=` consistently.       |
| Holding huge data in memory when only top-k matters                        | use nlargest/nsmallest lazily or maintain size-k heap. |

### heappushpop vs heapreplace quick reference

| Condition              | `heappushpop(h, x)`            | `heapreplace(h, x)`           |
|------------------------|--------------------------------|-------------------------------|
| `x >= heap[0]`         | push x, pop old min            | pop old min, push x (same)    |
| `x < heap[0]`          | returns x; **heap unchanged**   | pops old min, **pushes x**    |

---

## 11. Memory Tricks

- **Min-Heap = "Smallest bubble UP to root."**
- Picture a **pyramid**: small items at top, larger ones sift down — `heap[0]` is the apex (min).
- `heapify` is like **shaking** the pyramid until everything settles — O(n) because half the items barely move.
- `heappush` = **insert at leaf, bubble up**.
- `heappop` = **remove apex, move last leaf to top, bubble down**.
- `nsmallest` uses a **max-heap of size k** to track the smallest k seen (pairs with negation internally).
- **Max-heap by negation**: "flip the pyramid upside-down" by negating priorities.
- **Tie-breaker tuple** pictorially: `(priority, id, payload)` — compare priority, then id, never payload.

---

## 12. Interview Shortcuts

```python
# Top-k of a stream
heapq.nlargest(k, stream)

# K-th largest
heapq.nlargest(k, nums)[-1]
# O(n log k) with manual heap:
h = nums[:k]; heapq.heapify(h)
for x in nums[k:]:
    if x > h[0]: heapq.heapreplace(h, x)
# then h[0] is k-th largest

# Dijkstra — (distance, node)
import heapq
pq = [(0, src)]
while pq:
    d, u = heapq.heappop(pq)
    if d > dist[u]: continue
    for v, w in adj[u]:
        nd = d + w
        if nd < dist[v]:
            dist[v] = nd
            heapq.heappush(pq, (nd, v))

# Max-heap
heapq.heappush(h, -x); -heapq.heappop(h)

# Tie-safe tuples
tie = itertools.count()
heapq.heappush(h, (priority, next(tie), payload))

# Merge sorted streams lazily
for x in heapq.merge(s1, s2, s3): process(x)
```

---

## 13. Cheat Sheet Table

| Function                                    | Returns          | Mutates? | Time         | Notes                                  |
|---------------------------------------------|------------------|----------|--------------|----------------------------------------|
| `heapify(h)`                                | None             | yes      | O(n)         | bulk build                             |
| `heappush(h, x)`                            | None             | yes      | O(log n)     | leaf + sift-up                          |
| `heappop(h)`                                | min item         | yes      | O(log n)     | root + sift-down                        |
| `heappushpop(h, x)`                         | smallest         | yes      | O(log n)     | push-then-pop; faster than separate     |
| `heapreplace(h, x)`                         | old min          | yes      | O(log n)     | pop-then-push; preserves size           |
| `nsmallest(k, it, key=None)`                | list ascending   | no       | O(n log k)   | lazy, doesn't mutate source             |
| `nlargest(k, it, key=None)`                 | list descending  | no       | O(n log k)   | uses min-heap internally                |
| `merge(*sorted_iters, key=None, reverse=False)` | iterator    | no       | O(N log k)   | streaming merge; source MUST be sorted  |
| `h[0]` (peek)                              | smallest         | no       | O(1)         | min at root                             |
| `len(h)`                                    | int              | no       | O(1)         |                                        |
| `in h`                                      | bool             | no       | O(n)         | linear scan                             |

---

## 14. Time Complexity Table

| Operation              | Time       | Notes                                       |
|------------------------|------------|---------------------------------------------|
| `heapify(list)`        | O(n)       | bottom-up sift-down                         |
| `heappush`             | O(log n)   |                                             |
| `heappop`              | O(log n)   |                                             |
| `heappushpop`          | O(log n)   | faster than `push; pop` in practice         |
| `heapreplace`          | O(log n)   | requires non-empty heap                     |
| `nsmallest(k, n)`      | O(n log k) | switches to sort if k ~ n                   |
| `nlargest(k, n)`       | O(n log k) | same                                        |
| `merge` (per element)  | O(log k)   | k iterables                                 |
| peek `h[0]`            | O(1)       |                                             |
| search `x in heap`     | O(n)       | linear scan                                 |
| remove arbitrary       | O(n) + O(log n) | no native; rebuild or use `dict` indices |

---

## 15. Visual Diagram (ASCII)

```
Min-heap as binary tree vs. array layout

Heap:  [1, 2, 8, 3, 5]

             TREE VIEW                ARRAY VIEW
                 1   <- heap[0]        [1, 2, 8, 3, 5]
                / \                       ^  ^  ^  ^  ^
               2   8   <- heap[1..2]      |  |  |  |  |
              / \                         indices
             3   5                       parent = (i-1)//2
                                         left   = 2i+1
                                         right  = 2i+2

Invariant:  heap[parent] <= heap[child]
   For i=1: heap[0]=1 <= heap[1]=2 ✓
   For i=2: heap[0]=1 <= heap[2]=8 ✓
   For i=3: heap[1]=2 <= heap[3]=3 ✓
   For i=4: heap[1]=2 <= heap[4]=5 ✓


heappush 0:

   [1, 2, 8, 3, 5, 0]    ← appended at end
              │
              ▼  sift-up:
   [1, 2, 0, 3, 5, 8]    swap(0, parent@2=8)
   [0, 2, 1, 3, 5, 8]    swap(0, parent@0=1)
   stop — at root


heappop:
   min = h[0] = 0
   move last (8) to root, drop tail:
   [8, 2, 1, 3, 5]
   sift-down(0):
      smallest child of 8 = min(2, 1) = 1 (idx2)
      swap(8, 1): [1, 2, 8, 3, 5]
      8 has no children at new index 2 that are smaller → done
   return 0


Two-heap median trick:

   small half (MAX-HEAP, neg)   │   large half (MIN-HEAP)
   ┌──┬──┬──┬──┬──┐              │   ┌──┬──┬──┬──┬──┐
   │10│ 7│ 5│ 3│ 1│              │   │12│15│18│20│25│
   └──┴──┴──┴──┴──┘              │   └──┴──┴──┴──┴──┘
   top = max of small           │   top = min of large
       (negate to retrieve)     │
                 │                       │
                 └───── median ──────────┘
                  = (lo_max + hi_min) / 2


Sliding-window max with a monotonic deque (deque.md cross-link)
and Top-K via min-heap of size k:

   seen stream: [4, 10, 3, 7, 1], k=3, want k-th largest (i.e., 3rd largest)

   h after each push (kept at size ≤ k):
   after 4:   [4]
   after 10:  [4, 10]               heapify
   after 3:   [3, 10, 4]
   after 7:   7 > 3 → heapreplace:  [4, 10, 7]
   after 1:   1 !> 4 → skip          [4, 10, 7]
   3rd largest = h[0] = 4 ✓
```

---

## 16. Beginner Notes

> **Remember:** `heapq` operates on a *list*, not a custom class. Pass your list, mutate in place.
>
> **Remember:** A heap-ordered list is **NOT sorted** — only `heap[0]` is guaranteed min.
>
> **Remember:** To get top-k efficiently from a stream, maintain a heap of **exactly size k**; push new elements and `heapreplace` if they beat the min.
>
> **Remember:** For max-heap on numbers, **negate** on push and pop.
>
> **Remember:** For tuple items, the **first element is the priority**. Always include a comparable tie-breaker (e.g., `itertools.count()`) to avoid comparing unorderable payloads.
>
> **Remember:** `nsmallest` and `nlargest` lazy-evaluate `iterable`s — they don't load everything into memory.

---

## 17. FAANG Tips

- **Kth Largest** (#215): `nsmallest(len(nums)-k+1)` or `nlargest(k, nums)[-1]`. Interviewers prefer a size-k min-heap to demonstrate understanding of space.
- **Dijkstra's algorithm** — `heapq` is the canonical Python choice for the priority queue. Pair `(distance, node)` tuples. Use a `visited`/`dist` array to skip stale entries.
- **Median of Stream** (#295): two-heap trick, balance sizes each insertion. Watch the invariant `len(lo) >= len(hi)`.
- **Merge k Sorted Lists** (#23): heap of `(node.val, i, node)` so the tie-breaker avoids comparing nodes.
- **Top K Frequent** (#347): `Counter(nums).most_common(k)` is idiomatic, but a heap of size k (`nlargest(k, c, key=c.get)`) shows streaming-style understanding. Cross-link: [counter.md](./counter.md).
- **Sliding Window Median** (#480): two heaps with **lazy deletion** (defer deletes, clean top when needed). Hard; comfort with heapq is essential.
- **Ugly Number II** (#264) / **Super Ugly Number** (#313): min-heap of multiplied candidates; dedupe.
- **Reorganize String** (#767): push `(-count, char)` pairs into a max-heap, pop twice, re-push decremented.
- **Avoid** `heapq` for **sparse random-access priority updates** (decrease-key) — there's no O(log n) native option. Maintain your own index map or use `queue.PriorityQueue` with lazy invalidation.
- See [../07_Algorithms/heap.md](../07_Algorithms/heap.md) for the algorithm-first view.

---

## 18. Practice Problems

**Easy**
- LeetCode 215 — Kth Largest Element in an Array
- LeetCode 703 — Kth Largest Element in a Stream (textbook priority queue)
- LeetCode 1046 — Last Stone Weight
- LeetCode 2336 — Smallest Number in Infinite Set (heap + set)

**Medium**
- LeetCode 347 — Top K Frequent Elements
- LeetCode 264 — Ugly Number II
- LeetCode 378 — Kth Smallest Element in a Sorted Matrix
- LeetCode 973 — K Closest Points to Origin
- LeetCode 767 — Reorganize String
- LeetCode 253 — Meeting Rooms II (premium)

**Hard**
- LeetCode 23 — Merge K Sorted Lists
- LeetCode 295 — Find Median from Data Stream
- LeetCode 480 — Sliding Window Median
- LeetCode 502 — IPO
- LeetCode 778 — Swim in Rising Water (Dijkstra + heap)

---

> End of section 06 — back to **[README](../README.md)**.
> See also: **[counter.md](./counter.md)** · **[defaultdict.md](./defaultdict.md)** · **[deque.md](./deque.md)**.