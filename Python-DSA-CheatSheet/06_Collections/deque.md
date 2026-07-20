# collections.deque

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
> Section: 06 — Collections
> 🔗 Related: [counter.md](./counter.md) · [defaultdict.md](./defaultdict.md) · [heapq.md](./heapq.md) · [../07_Algorithms/sliding_window.md](../07_Algorithms/sliding_window.md)
> Back to [README](../README.md)

---

## 1. What is it?

`collections.deque` (pronounced "deck") is a **doubly-ended queue** implemented as a doubly-linked list of **block arrays**. It supports O(1) appends and pops on **either end** — unlike Python lists, which are O(n) at the front.

```python
from collections import deque
q = deque([1, 2, 3])
q.appendleft(0)        # [0,1,2,3]
q.pop()                # 3  →  [0,1,2]
q.popleft()            # 0  →  [1,2]
```

Key facts:
- Stitched together from fixed-size blocks for cache-friendliness.
- Supports an optional `maxlen` for **automatic eviction** (a built-in sliding window!).
- **Indexing is O(n)** — don't use it as a list.
- Thread-safe for **single atomic appends/pops** (the underlying CPython calls hold the GIL).
- Mutable and iterable.

---

## 2. Why do we use it?

Lists are O(n) for `pop(0)` and `insert(0, x)` because they shift every element. Any FIFO queue implemented with `list.pop(0)` degrades to O(n²) for n operations.

`deque` gives you:
- **O(1)** `appendleft` and `popleft` — perfect FIFO queue / stack.
- **O(1)** `append` and `pop` as well — so it's a *double-ended* structure.
- `maxlen` parameter — auto-drops from the other end, ideal for sliding window / rolling buffer / ring-buf.
- `.rotate(n)` — O(k) cyclic shift in one call (versus O(n) with slices).
- Memory savings over a hand-rolled linked list.

Typical interview uses:
- BFS frontiers.
- Palindrome verification.
- Sliding window / max with monotonically-decreasing deque.
- LRU cache (pair with `OrderedDict` or `dict`).
- Undo / history buffers with `maxlen`.

---

## 3. When should I choose it?

| Situation                                          | Best tool                     |
|----------------------------------------------------|-------------------------------|
| FIFO queue                                         | ✅ `deque.popleft`            |
| Stack                                              | `list.pop()` (or `deque.pop`)|
| Sliding window of fixed size                       | ✅ `deque(maxlen=k)`          |
| BFS frontier                                       | ✅ `deque([src])`             |
| Monotonic queue (sliding window max/min)           | ✅ `deque`                    |
| Lazy-creation dict of lists                        | `defaultdict(list)` ([defaultdict.md](./defaultdict.md)) |
| Priority queue / k-th largest                      | `heapq` ([heapq.md](./heapq.md)) |
| Frequency + top-k                                  | `Counter` ([counter.md](./counter.md)) |
| Random-access by index                             | `list` (deque indexing is O(n)) |
| Thread-safe atomic ops                             | `queue.Queue` / `deque`       |
| Need sorted insert                                 | `bisect`                      |

**Decision flow:**

```
      Push and pop from the SAME end?
                  │
                  YES → use list (stack)
                  │ NO
                  │
      Push one end, pop other end?
                  │
                  YES → deque (FIFO queue)
                  │ NO
                  │
      Pop from BOTH ends?
                  │
                  YES → deque
                  │ NO
                  │
      Need O(1) max while doing this? → monotonic deque or heapq
```

---

## 4. Syntax

```python
from collections import deque

# Constructors
deque()                       # empty
deque(iterable)               # copy items
deque(iterable, maxlen=k)    # fixed size — auto-evicts from opposite end
deque(maxlen=k)              # empty with cap

# End operations (all O(1))
q.append(x)                   # push right
q.appendleft(x)              # push left
q.pop()                       # remove from right
q.popleft()                   # remove from left

# Bulk (O(k))
q.extend(iter)
q.extendleft(iter)            # NOTE: reverses order!
q.rotate(n=1)                 # rotate right by n (n<0 rotates left)
q.reverse()

# Search / erase
q.remove(value)               # O(n) — first match
q.count(value)                # O(n)
q.index(value, start, stop)   # O(n)

# Misc
q.copy()                      # shallow copy (keeps maxlen)
q.clear()
q.maxlen                      # attribute (read-only)
q[i]                          # O(n) indexing — read/write
len(q), 'x' in q, iter(q), reversed(q)
q + other                     # new deque
q *= n                        # repeat in place
```

---

## 5. Basic Example

```python
from collections import deque

q = deque([1, 2, 3])
q.append(4)
q.appendleft(0)
print(q)               # deque([0, 1, 2, 3, 4])

print(q.pop())         # 4
print(q.popleft())     # 0
print(q)               # deque([1, 2, 3])

q.rotate(1)            # deque([3, 1, 2]) — every item shifted right
q.rotate(-1)           # deque([1, 2, 3]) — shift back left
print(q)               # deque([1, 2, 3])

# maxlen: auto-evict
buf = deque(maxlen=3)
for i in range(5):
    buf.append(i)
print(buf)             # deque([2, 3, 4], maxlen=3)  ← 0,1 evicted
```

---

## 6. Step-by-Step Dry Run

BFS on `graph = {1: [2,3], 2: [4], 3: [], 4: []}` from source `1`.

Initial: `q = deque([1])`, `seen = {1}`

| Step | q (before)  | pop | neighbors added | q (after)    | visited |
|------|-------------|-----|------------------|--------------|---------|
| 1    | [1]         | 1   | 2,3 (both first) | [2, 3]       | [1]     |
| 2    | [2, 3]      | 2   | 4 (unseen)      | [3, 4]       | [1,2]   |
| 3    | [3, 4]      | 3   | none             | [4]          | [1,2,3] |
| 4    | [4]         | 4   | none             | []           | [1,2,3,4]|

`deque.popleft` is O(1) at every step — total BFS = O(V + E).

---

## 7. Built-in Methods   (EXHAUSTIVE)

### `deque(iterable=None, maxlen=None)`
- **Purpose:** Construct a new deque, optionally from an iterable with a size cap.
- **Syntax:** `deque([it,] [maxlen=k])`
- **Input:** iterable + optional int maxlen
- **Output:** new deque
- **Example:**
  ```python
  deque([1,2,3])                # deque([1,2,3])
  deque(maxlen=2)               # deque([], maxlen=2)
  deque([1,2,3,4], maxlen=2)    # deque([3,4], maxlen=2)  ← keeps last 2
  ```
- **Time:** O(n) construction.
- **Interview use:** every BFS / queue initialization.
- **Common mistake:** setting `maxlen` makes pops auto-happen on opposite end during appends — surprising in problems without size caps.
- **Shortcut:** specify `maxlen=k` and never worry about eviction code.

### `.append(x)`
- **Purpose:** Add to the right end. With `maxlen` set: silently drops the leftmost element instead of growing.
- **Syntax:** `q.append(x)`
- **Time:** O(1) amortized.
- **Example:** `q.append(5)` → `[1,2,3]` → `[1,2,3,5]`
- **Interview use:** enqueue (FIFO), top-of-stack.
- **Common mistake:** expecting an error on overflow when `maxlen` is set — there is none.
- **Shortcut:** same API as `list.append`.

### `.appendleft(x)`
- **Purpose:** Add to the left end. With `maxlen`, evicts the rightmost element if needed.
- **Syntax:** `q.appendleft(x)`
- **Time:** O(1)
- **Example:** `q.appendleft(0)` → `[1,2,3]` → `[0,1,2,3]`
- **Interview use:** front-insertion for sliding-window-monotonic-queue; push-to-front in palindrome checks.
- **Common mistake:** for `extendleft`, the order is reversed (see below) — but `appendleft` is one item so no reversal surprise.
- **Shortcut:** gives O(1) front push — impossible with lists.

### `.pop()`
- **Purpose:** Remove and return rightmost element.
- **Syntax:** `q.pop()`
- **Time:** O(1)
- **Example:** `deque([1,2,3]).pop()` → `3`, deque now `[1,2]`
- **Interview use:** undo stack, palindrome from the right.
- **Common mistake:** calling `.pop()` on an empty deque raises `IndexError`.
- **Shortcut:** identical signature to `list.pop()` (without args).

### `.popleft()`
- **Purpose:** Remove and return leftmost element — the **fast** FIFO pop.
- **Syntax:** `q.popleft()`
- **Time:** **O(1)** — the killer feature vs `list.pop(0)` which is O(n).
- **Example:** `deque([1,2,3]).popleft()` → `1`, deque now `[2,3]`
- **Interview use:** BFS queue drains, sliding-window removals, LRU cache eviction.
- **Common mistake:** calling on empty deque → `IndexError`.
- **Shortcut:** — pairs with `append` for FIFO queue, or `appendleft` for a stack.

### `.extend(iterable)`
- **Purpose:** Append every item from iterable to the right (respects maxlen).
- **Time:** O(k) where k = len(iter).
- **Example:** `deque([1]).extend([2,3])` → `deque([1,2,3])`.

### `.extendleft(iterable)`
- **Purpose:** Append every item from iterable to the left, **reversing their order**.
- **Time:** O(k)
- **Example:** `deque([1]).extendleft([2,3])` → `deque([3,2,1])` — because each appendleft puts the next item at the front.
- **Common mistake:** NOT reversing mentally — surprisingly easy to get wrong on a whiteboard.
- **Shortcut:** `extendleft` is the inverse-order twin of `extend`.

### `.rotate(n=1)`
- **Purpose:** Rotate the deque n steps to the **right** (positive n) or **left** (negative n). Wraps around.
- **Time:** O(k) where k=abs(n) (rotating past `len(q)` is min(|n|,len) effectively).
- **Example:**
  ```python
  q = deque([1,2,3,4,5])
  q.rotate(2)      # deque([4,5,1,2,3])
  q.rotate(-1)     # deque([5,1,2,3,4])
  ```
- **Interview use:** circular arrays, round-robin scheduling, "rotate string/rotated-array" problems.
- **Common mistake:** thinking `rotate(1)` shifts left — it does NOT; it shifts right (last items become first).
- **Shortcut:** `rotate(-n)` == rotate-left-by-n.

### `.remove(value)`
- **Purpose:** Remove first occurrence of value.
- **Time:** O(n) — must scan.
- **Example:** `deque([1,2,1]).remove(1)` → `deque([2,1])`.
- **Common mistake:** raises `ValueError` if value not present.
- **Interview use:** avoid — usually better to use a different structure if you need O(1) arbitrary removal.

### `.clear()`
- **Purpose:** Empty the deque (keeps `maxlen`).
- **Time:** O(n) (needs to free nodes) but logically O(1) feel.
- **Example:** `q.clear()` → `deque([])`.

### `.copy()`
- **Purpose:** Shallow copy. Preserves `maxlen`.
- **Time:** O(n).
- **Example:** `q2 = q.copy()`.
- **Common mistake:** it is a shallow copy — nested objects are shared.

### `.count(value)`
- **Purpose:** Count occurrences. O(n) scan.
- **Example:** `deque([1,2,1]).count(1)` → `2`.

### `.index(value[, start[, stop]])`
- **Purpose:** First index of value within bounds. O(n) scan.
- **Raises:** `ValueError` if not found.

### `.reverse()`
- **Purpose:** Reverse the deque in place.
- **Time:** O(n).
- **Example:** `deque([1,2,3]).reverse()` → `deque([3,2,1])`.

### Indexing `q[i]`, slice-free
- **Purpose:** Read/write element at position i. **O(n)** — implemented by walking the linked blocks.
- **Common mistake:** using deque in `for i in range(len(q)): ... q[i]` becomes O(n²). Convert to list first or iterate `for x in q`. Slicing (`q[1:3]`) is **unsupported**.
- **Shortcut:** don't index deques; iterate them.

### Operators `+`, `+=`, `*=`
- **`q + other`** returns a **new** deque (concatenation).
- **`q += other`** extends in place.
- **`q *= n`** repeats elements in place (new length capped by maxlen).
- **Common mistake:** `q + q2` doesn't mutate either operand.

---

## 8. Interview Example

**Monotonic deque for Sliding Window Maximum (#239):**

```python
from collections import deque

def maxSlidingWindow(nums, k):
    res, dq = [], deque()
    for i, n in enumerate(nums):
        while dq and nums[dq[-1]] <= n:
            dq.pop()                       # remove smaller from right
        dq.append(i)
        if dq[0] <= i - k:
            dq.popleft()                    # evict expired front
        if i >= k - 1:
            res.append(nums[dq[0]])
    return res
```

**LRU cache building block:**

```python
from collections import deque, OrderedDict
# Or just use OrderedDict directly
class LRUCache:
    def __init__(self, cap):
        self.cap = cap
        self.d = OrderedDict()
    def get(self, key):
        if key not in self.d: return -1
        self.d.move_to_end(key)            # mark recently used
        return self.d[key]
    def put(self, key, val):
        if key in self.d: del self.d[key]
        self.d[key] = val
        if len(self.d) > self.cap:
            self.d.popitem(last=False)     # evict oldest from front
```

---

## 9. When NOT to use

- **Random indexing patterns** — use `list` (deque is O(n) at indexing).
- **Sorted-order insertions** — use `bisect` on a list, or `heapq`.
- **Need a min/max** — `heapq`.
- **Need O(1) lookup** — use a dict or set.
- **Heavy slicing** — unsupported in deque; use list.
- **Immutable** — there's no frozen deque; use `tuple`.

---

## 10. Common Mistakes

| Mistake                                                          | Fix                                           |
|------------------------------------------------------------------|-----------------------------------------------|
| Using `list.pop(0)` as a queue (O(n) per op                     | use `deque.popleft` for O(1).                |
| Indexing a deque in a tight loop expecting O(1)                   | convert to list first or iterate `for x in q`.|
| Slicing a deque: `q[1:3]`                                          | not supported — `list(q)[1:3]` works.         |
| Forgetting `extendleft` reverses order                            | reverse the iterable first if order matters. |
| Confusing direction of `rotate(n)`                                | positive rotates RIGHT; negative rotates LEFT.|
| Surprised when `maxlen` silently drops items                       | that's the feature, not a bug; document it.   |
| Storing mutable objects and expecting `.copy()` to deep-copy     | use `copy.deepcopy`.                          |
| Expecting `queue.Queue` semantics (blocking, threads)             | `deque` is non-blocking; `queue.Queue` for producer/consumer. |
| `deque + deque` not modifying                                     | arithmetic returns new deque.                |
| Expecting thread-safety across multiple ops                       | single ops only — multiple ops still race.    |

---

## 11. Memory Tricks

- **deque = "deck of cards you can draw from top OR bottom."**
- Picture **heads and tails**: you can pull from both ends, but the middle requires "flipping through" each card.
- `maxlen` = **a tray that holds exactly k items**; new one in → oldest falls off the far end.
- `popleft` is the "FIFO pop"; `pop` is the "stack pop".
- `extendleft` is like **stacking** plates onto the left side — last stacked is on top (so order looks reversed).
- `rotate(n)` = spin the tray n steps clockwise; `rotate(-n)` anticlockwise.

---

## 12. Interview Shortcuts

```python
# FIFO queue (one producer, one consumer)
q = deque(); q.append(x); front = q.popleft()

# Stack
q.append(x); top = q.pop()

# Sliding window of last k values
buf = deque(maxlen=k)

# Round-robin (circular schedule)
queue.rotate(1)        # back goes to front

# Palindrome check
def is_pal(s):
    q = deque(s)
    while len(q) > 1:
        if q.popleft() != q.pop(): return False
    return True

# BFS
from collections import deque
q = deque([src])
while q:
    node = q.popleft()
    for nb in graph[node]:
        q.append(nb)
```

---

## 13. Cheat Sheet Table

| Operation              | Returns    | Mutates | Time     | Notes                          |
|------------------------|------------|---------|----------|--------------------------------|
| `deque(it, maxlen=k)`  | deque      | —       | O(n)     | constructor                    |
| `.append(x)`           | None       | yes     | O(1)     | right push                     |
| `.appendleft(x)`       | None       | yes     | O(1)     | left push                      |
| `.pop()`               | x          | yes     | O(1)     | right pop                      |
| `.popleft()`           | x          | yes     | O(1)     | left pop (the killer feature)  |
| `.extend(it)`          | None       | yes     | O(k)     | right extend                   |
| `.extendleft(it)`      | None       | yes     | O(k)     | left extend (reverses order)   |
| `.rotate(n)`           | None       | yes     | O(\|n\|) | right rotation; neg = left     |
| `.reverse()`           | None       | yes     | O(n)     |                                |
| `.remove(x)`           | None       | yes     | O(n)     | first match                    |
| `.count(x)`            | int        | no      | O(n)     |                                |
| `.index(x, i, j)`      | int        | no      | O(n)     |                                |
| `.clear()`             | None       | yes     | O(n)     | keeps `maxlen`                 |
| `.copy()`              | deque      | no      | O(n)     | shallow; preserves maxlen      |
| `q[i]`                 | item       | —       | **O(n)** | read/write; **no slicing**     |
| `len(q)`               | int        | no      | O(1)     |                                |
| `q + q2`               | deque      | no      | O(n+m)   | new deque (drops maxlen?)      |
| `q += it`              | None       | yes     | O(k)     | in place                       |
| `q *= n`               | None       | yes     | O(n\*k)  | repeat                         |
| `q.maxlen`             | int/None   | —       | O(1)     | read-only attr                 |

---

## 14. Time Complexity Table

| Operation           | `list`       | `deque`         |
|---------------------|--------------|-----------------|
| `append`            | O(1) amortized | O(1)        |
| `pop()`             | O(1)         | O(1)            |
| `appendleft`        | O(n)         | **O(1)**        |
| `pop(0)` / `popleft` | O(n)        | **O(1)**        |
| `q[i]`              | O(1)         | **O(n)**        |
| iterate             | O(n)         | O(n)            |
| insert middle       | O(n)         | O(n)            |
| remove by value     | O(n)         | O(n)            |
| extend              | O(k)         | O(k)            |
| memory per element  | ~8 bytes (ptr) | ~ larger    |

---

## 15. Visual Diagram (ASCII)

```
   Doubly-linked list of block arrays

      ┌────┐  ⇄  ┌────┐  ⇄  ┌────┐
   leftmost    blocks      rightmost
      └────┘      └────┘      └────┘
      ↑                            ↑
  appendleft / popleft        append / pop


   deque as double-ended buffer (maxlen=4):

   append(5) ──→  [1|2|3|4]   ←  maxlen cap reached
                       │
                       ▼
                  [2|3|4|5]   ← '1' silently evicted from LEFT

   appendleft(0) ──→  [2|3|4|5]
                   │
                   ▼
                  [0|2|3|4]   ← '5' silently evicted from RIGHT


   rotate(n=1)   — rotate right by one          rotate(-1)  — rotate left by one

   [a b c d]                                [a b c d]
       │                                       │
       ▼                                       ▼
   [d a b c]                                [b c d a]


   BFS queue visualization:

   push ───►  ┌──┬──┬──┬──┐  ───► popleft
              │n4│n3│n2│n1│            (FIFO)
              └──┴──┴──┴──┘
   appendleft      push to back        pop from front
```

---

## 16. Beginner Notes

> **Remember:** `deque.popleft()` is O(1); `list.pop(0)` is O(n). This is the #1 reason `deque` exists.
>
> **Remember:** Indexing a deque is **O(n)**. Don't use a deque like a list.
>
> **Remember:** `extendleft` reverses order — each new item is pushed to the front, so the last one ends up leftmost.
>
> **Remember:** `rotate(n)` rotates RIGHT for positive n. `rotate(-1)` is left-rotate.
>
> **Remember:** Setting `maxlen` makes `append`/`appendleft` silently evict from the opposite end — no exception is raised.
>
> **Remember:** A deque is iterable; never index when you can iterate `for x in q`.

---

## 17. FAANG Tips

- **BFS anywhere** (graphs, grids, 0-1 BFS) — `deque([src])`, `q.popleft()`, `q.append(nb)`.
- **Sliding Window Maximum** (#239) — maintain a **monotonic deque**: store indices whose values are decreasing; pop from the back, evict from the front by index diff. This is the canonical deque interview problem. See [../07_Algorithms/sliding_window.md](../07_Algorithms/sliding_window.md).
- **LRU Cache** (#146) — `OrderedDict.move_to_end` + `popitem(last=False)` is the easiest Python solution; you can build it on a `deque + dict` pair instead.
- **Monotonic stacks** for "Next Greater Element" — a `deque` (or even a plain list as a stack) works; use deque when you need both-end operations.
- **Sliding-window median** (#480) — much harder than max; use two heaps or a SortedList, not a deque.
- **Verify palindrome** by pulling `popleft()` + `pop()` and comparing.
- **Producer/consumer threading**? Use `queue.Queue` (blocks on get/put), NOT deque.
- **Rotate Array** (#189) — `deque(nums).rotate(k); list(q)` is ultra-pythonic, but interviewers usually want in-place list rotation.
- Use `deque(maxlen=k)` for **rolling buffers / moving averages** — no manual `pop` needed.

---

## 18. Practice Problems

**Easy**
- LeetCode 232 — Implement Queue using Stacks (solve with deques)
- LeetCode 225 — Implement Stack using Queues (deque for the queue)
- LeetCode 9 — Palindrome Number (verify with deque)
- LeetCode 346 — Moving Average from Data Stream (maxlen!)

**Medium**
- LeetCode 239 — Sliding Window Maximum (monotonic deque — must-know!)
- LeetCode 207 — Course Schedule (BFS Kahn's algorithm)
- LeetCode 200 — Number of Islands (BFS variant)
- LeetCode 143 — Reorder List (use deque to weave)
- LeetCode 933 — Number of Recent Calls (maxlen deque)

**Hard**
- LeetCode 146 — LRU Cache
- LeetCode 460 — LFU Cache
- LeetCode 862 — Shortest Subarray with Sum at Least K (monotonic deque + prefix sum)
- LeetCode 1425 — Constrained Subset Sum (monotonic deque + DP)

---

> Next: **[heapq.md](./heapq.md)** — the priority-queue / k-th-largest tool.
> Back to **[defaultdict.md](./defaultdict.md)** or **[counter.md](./counter.md)**.