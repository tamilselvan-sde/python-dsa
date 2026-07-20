# Linked List in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 07 — Algorithms
> 🔗 Related: [stack.md](./stack.md) · [queue.md](./queue.md) · [hash_map.md](./hash_map.md) · [trees.md](./trees.md)
> OOP: [../05_OOP/classes.md](../05_OOP/classes.md)
> Back to [README](../README.md)

---

## 1. What is it?

A **linked list** is a chain of **nodes**, each carrying a `value` and a `next` pointer. Unlike an array, the elements are **not contiguous in memory** — each node links to the next. You traverse by following `next` pointers.

Variants:
- **Singly linked list** — `next` only, one direction.
- **Doubly linked list** — `next` and `prev`, both directions. Foundation for LRU caches and deque.
- **Circular linked list** — tail links back to head (or to some node).
- **Dummy head / sentinel node** — a `dummy` node placed before `head` so insertion/deletion code can ignore the "head is special" branch.

```python
class Node:
    def __init__(self, val=0, nxt=None):
        self.val = val
        self.next = nxt

class DoublyNode:
    def __init__(self, val=0, prev=None, nxt=None):
        self.val = val
        self.prev = prev
        self.next = nxt
```

**What problem it solves:** Constant-time **insertion/deletion at known positions** (head or middle once you have the pointer). Memory grows one node at a time. Powers LRU caches, hash table buckets (chaining), graph adjacency chains, FIFO queues.

**Real-world analogy:** A treasure hunt where each clue tells you the address of the next clue. To find clue #5 you must visit clues #1, #2, #3, #4 first.

---

## 2. Why do we use it?

- **O(1) head insert / delete** — no shifting.
- **O(1) middle insert / delete**, *if* you have a pointer to the predecessor.
- **Dynamic size** — no preallocation, no over-allocation.
- Foundation for **stacks**, **queues**, **LRU caches** (doubly + dict), **graph adjacency chains**, **polynomial arithmetic**.
- Quick **two-pointer techniques**: cycle detection, middle finding, k-th from end — all O(n) with **O(1) extra space**.

---

## 3. When should I choose it? — Decision Table

| Situation                                        | Best choice                            | Notes                                  |
|--------------------------------------------------|----------------------------------------|----------------------------------------|
| Frequent insert/delete at head or known pointer   | **singly/doubly linked list**           | O(1) vs O(n) for array middle           |
| Need backtrack / iterate both directions          | **doubly** linked list                  | +O(prev) pointer per node                |
| Random access `arr[i]`                            | array / list                            | O(1) vs linked list O(n)                 |
| Need sorted + frequent range queries              | balanced BST / `sortedcontainers`       | -                                       |
| Implement LRU cache                                | doubly list + dict                      | (146)                                   |
| Implement stack / queue                             | list / deque                              | linked is overkill in Python             |
| Cycle detection / nested pointer metaproblem      | **slow/fast two-pointer**                | (141, 142)                              |
| Reverse in-place                                    | singly + iterative pointer technique   | (206)                                   |

---

## 4. Syntax

```python
class Node:
    __slots__ = ("val", "next")    # saves memory per node
    def __init__(self, val=0, nxt=None):
        self.val = val
        self.next = nxt

# Build manually: 1 -> 2 -> 3
head = Node(1, Node(2, Node(3)))

# Traverse
def traverse(h):
    cur = h
    while cur:
        yield cur.val
        cur = cur.next

list(traverse(head))   # [1, 2, 3]

# Insert at head
new = Node(0, head); head = new

# Insert after target node
def insert_after(node, val):
    node.next = Node(val, node.next)

# Delete after target node
def delete_after(node):
    if node.next:
        node.next = node.next.next

# Dummy-head idiom (avoids branching on head change)
def remove_vals(head, target):
    dummy = Node(0, head)
    prev = dummy
    while prev.next:
        if prev.next.val == target:
            prev.next = prev.next.next
        else:
            prev = prev.next
    return dummy.next
```

---

## 5. Basic Example

### LinkedList class with common operations

```python
class Node:
    __slots__ = ("val", "next")
    def __init__(self, val=0, nxt=None):
        self.val = val
        self.next = nxt
    def __repr__(self): return f"Node({self.val})"


class LinkedList:
    def __init__(self, items=None):
        self.head = None
        if items:
            for x in items:
                self.append_tail(x)

    def append_tail(self, v):
        if not self.head:
            self.head = Node(v); return
        cur = self.head
        while cur.next:
            cur = cur.next
        cur.next = Node(v)

    def append_head(self, v):
        self.head = Node(v, self.head)

    def pop_head(self):
        if not self.head: raise IndexError("empty")
        v = self.head.val
        self.head = self.head.next
        return v

    def to_list(self):
        out, cur = [], self.head
        while cur:
            out.append(cur.val); cur = cur.next
        return out

    def __len__(self):
        n, cur = 0, self.head
        while cur: n += 1; cur = cur.next
        return n

ll = LinkedList([1, 2, 3, 4])
ll.append_head(0); ll.append_tail(5)
print(ll.to_list())      # [0, 1, 2, 3, 4, 5]
print(ll.pop_head())     # 0
print(len(ll))           # 5
```

**Output:** `[0, 1, 2, 3, 4, 5]` `0` `5`

### Reverse a list iteratively (LC 206)

```python
def reverseList(head):
    prev, cur = None, head
    while cur:
        nxt = cur.next    # remember
        cur.next = prev   # flip
        prev = cur        # advance the ladder
        cur = nxt
    return prev
```

### Cycle detection (LC 141) — Floyd's Tortoise & Hare

```python
def hasCycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast: return True
    return False
```

---

## 6. Step-by-Step Dry Run

### Reverse `1 → 2 → 3 → None`

```
init: prev=None, cur=Node(1)
step 1  nxt=None.next? NO → wait let's re-read.
       1) nxt = cur.next  →  Node(2)
       2) cur.next = prev → 1.next = None   →  list now 1→None
       3) prev = cur       → prev = 1
       4) cur = nxt        → cur = 2
step 2  nxt = 3
       2.next = 1            → 2→1
       prev = 2
       cur = 3
step 3  nxt = None
       3.next = 2            → 3→2
       prev = 3
       cur = None
done:  return prev = Node(3)   → 3 → 2 → 1 → None
```

### Cycle detection on `1 → 2 → 3 → 4 → 2` (cycle: 4→2)

```
slow fast   pos
1   1       -
2   3       slow moves 1, fast moves 2
3   2       (cycles through 3 → 4 → 2)
4   4       meet at Node(4) -> return True
```

### Find middle (slow/fast until fast is at last)

```
1 → 2 → 3 → 4 → 5
slow=1 fast=1
slow=2 fast=3
slow=3 fast=5   fast.next=None → stop → slow at middle (3)

1 → 2 → 3 → 4
slow=1 fast=1
slow=2 fast=3
slow=3 fast=? fast.next.next=None → stop -> slow at the *right* middle (3)
```

### Remove Nth from End (LC 19), `n=2`, on `1→2→3→4→5`

```
1. advance `fast` n=2 steps: fast -> Node(3)
2. advance BOTH slow (head) and fast until fast.next is None
       slow=fake head sentinel, fast=3
       iter: slow=1 fast=4
       iter: slow=2 fast=5  fast.next=None → stop
3. delete slow.next: 2.next = 4   → 1→2→3→4→5  becomes 1→2→3→5
```

---

## 7. Built-in Methods

Python has **no** built-in linked list class. We always implement `Node` by hand — see [../05_OOP/classes.md](../05_OOP/classes.md) for OOP patterns. The closest things in the stdlib are:

| Tool                       | Purpose                                | Where used                                |
|----------------------------|----------------------------------------|--------------------------------------------|
| `collections.deque`         | doubly linked list behind the scenes   | power of numpy-free O(1) queue / stack      |
| `OrderedDict`               | hashmap + doubly linked list           | LRU cache base (146)                        |
| `weakref.WeakValueDictionary`| linked, auto-removed entries           | cache, weak-referenced graphs               |

> `deque`'s internal representation is a doubly linked list of *blocks* — each block holds ~64 PyObject pointers, giving both O(1) ops and good cache locality.

### Hand-rolled `Node` — idiomatic patterns

| Pattern                          | Code sketch                                  | Use                            |
|----------------------------------|----------------------------------------------|--------------------------------|
| Sentin / dummy head              | `dummy = Node(0, head); ... return dummy.next`| avoid `head is None` branch      |
| Two-pointer slow/fast            | `slow = fast = head`                          | cycle, middle, kth-from-end     |
| Reverse with three pointers      | `prev, cur = None, head; while cur: ...`      | (206)                           |
| Recursive reverse                | `def rev(head): tail = rev(head.next); ...`   | concise but uses O(n) stack     |
| Doubly list + dict for LRU       | dict key → node, node carries prev/next       | (146) — see hash_map.md         |

---

## 8. Interview Example

### LC 21 — Merge Two Sorted Lists (Easy)

```python
def mergeTwoLists(a, b):
    dummy = Node(); tail = dummy
    while a and b:
        if a.val <= b.val: tail.next = a; a = a.next
        else:              tail.next = b; b = b.next
        tail = tail.next
    tail.next = a or b
    return dummy.next
```

### LC 143 — Reorder List (Medium)

Fold `L0 → L1 → ... → Ln` into `L0 → Ln → L1 → Ln-1 → ...`

```python
def reorderList(head):
    if not head or not head.next: return
    # 1) find middle (right one)
    slow, fast = head, head
    while fast.next and fast.next.next:
        slow = slow.next; fast = fast.next.next
    # 2) cut + reverse second half
    second = reverse(slow.next); slow.next = None
    # 3) weave
    first = head
    while second:
        fn, sn = first.next, second.next
        first.next = second; second.next = fn
        first, second = fn, sn

def reverse(h):
    prev, cur = None, h
    while cur:
        nxt = cur.next; cur.next = prev; prev = cur; cur = nxt
    return prev
```

### LC 25 — Reverse Nodes in k-Group (Hard)

```python
def reverseKGroup(head, k):
    def reverse(h, k):
        prev, cur = None, h
        while cur and k:
            nxt = cur.next; cur.next = prev; prev = cur; cur = nxt; k -= 1
        h.next = cur
        return prev
    dummy = Node(0, head)
    group_prev = dummy
    while True:
        kth = group_prev
        for _ in range(k):
            kth = kth.next
            if not kth: return dummy.next
        group_next = kth.next
        new_head = reverse(group_prev.next, k)
        # reverse() already re-linked old head → group_next
        group_prev.next = new_head
        group_prev = group_prev.next.next if group_prev.next else group_prev  # jump to kth of new
        # simpler: advance group_prev twice; the clearest version uses saved group_prev = old_head
```

(Most interviewers accept this with helper `reverse(h, k)`.)

---

## 9. When NOT to use

- **Random access** (`arr[i]`): O(n) traversal in linked list vs O(1) in array.
- **Search by value / position repeatedly** — hash map or sorted array wins.
- **CPU-cache-sensitive workloads** — pointer chasing misses cache; arrays breathe cache lines.
- **`len()` O(1)** needed — Python's `list` gives this; you must maintain a counter on a linked list.
- **Persistent storage / serialization** — pointer addresses don't survive serialization; use arrays.
- **Small/medium data** where the O(n) shift of `list.insert` is dwarfed by Python's C-level optimizations.

---

## 10. Common Mistakes

1. **Forgetting `prev` / `dummy`**: trying to mutate `head` without a sentinel leads to ugly `if head is None` branches.
2. **`cur = cur.next` after reassignment**: in reversal must capture `nxt = cur.next` **before** mutating `cur.next`.
3. **Cycle = address equality**, not value equality. Always use `slow is fast`, never `slow.val == fast.val` — values collide; identity doesn't.
4. **Two-pointer termination**: middle-of-list loop conditions differ for odd vs even length. Use `while fast and fast.next` (left middle) vs `while fast.next and fast.next.next` (right middle).
5. **Recursive reverse stack overflow** for very large lists (Python recursion limit ~1000).
6. **Modifying head without dummy** — sentinel-free code mutates `head` mid-traversal and loses pointers.
7. **Garbage reference loops**: cyclically linked structures can leak memory; use `weakref` or break the cycle before drop.
8. **Returning the wrong node in `reverseKGroup`**: returning `head` instead of `prev` after the flip is the classic bug.

---

## 11. Memory Tricks

- **"Arrow flip in a corridor"**: For reversal, picture walking from left to right; at every node, you turn its forward arrow backward.
- **Floyd's Tortoise & Hare**: tortoise speed 1, hare speed 2. If they meet, there's a loop.
- **Find-middle-fast**: stop the hare at `fast.next is None` (odd length) or `fast is None` (even length).
- **Dummy head** = "parking spot before the start"; never lose `head`.
- **Two-pointer = race on a runway**: slow pointer lets fast pointer measure geometry.
- **k-th from end**: send a runner k nodes ahead; when it hits None, you stop at the target.
- **LRU = doubly + dict**: dict gives O(1) find, doubly gives O(1) move-to-front.

---

## 12. Interview Shortcuts

- Always create `dummy = Node(0, head)` so you can `return dummy.next` without headache.
- Use `slow, fast = head, head` + `while fast and fast.next` to find the left middle, ending exactly at `slow`.
- Merge two sorted lists with `dummy` + tail pointer; the one-liner `tail.next = a or b` handles leftover.
- Detect cycle: Floyd → True/False (LC 141). To find the cycle *entry* (LC 142): once `slow == fast`, reset `slow` to head and walk both 1-step until they meet — that's the entry.
- Find kth-from-end by sending `fast` k ahead, then locking both at pace 1.
- **Reverse K group** is the integration test — if you can write this in 7 minutes, you've mastered linked lists.
- For LRU: `OrderedDict.move_to_end(key)` is one line; build a doubly list manually only when the interviewer forbids it.

---

## 13. Cheat Sheet Table

| Operation                  | Singly           | Doubly           | Array (`list`)      |
|----------------------------|------------------|------------------|---------------------|
| Access by index            | O(n)             | O(n)             | **O(1)**            |
| Search by value            | O(n)             | O(n)             | O(n)                |
| Insert at head             | **O(1)**         | **O(1)**         | O(n)                |
| Insert at tail             | O(n) (or O(1) with tail pointer) | **O(1)** (with tail) | O(1) amortized   |
| Insert at middle (have pred)| **O(1)**         | **O(1)**         | O(n)                |
| Delete head                | O(1)             | O(1)             | O(n)                |
| Delete tail                | O(n) (singly)    | **O(1)**         | O(1)                |
| Delete middle (have pred)  | O(1)             | O(1)             | O(n)                |
| Reverse in place           | O(n)             | O(n)             | O(n) (`[::-1]`)     |
| Cycle detection            | O(n), O(1) mem   | O(n), O(1) mem   | (rare — see hash)   |
| Memory (per node)          | 1 ptr + val      | 2 ptrs + val     | contiguous block    |

---

## 14. Time Complexity Table

| Operation                          | Time           | Space      | Notes                                 |
|------------------------------------|----------------|------------|---------------------------------------|
| Traverse to index / find value     | O(n)           | O(1)       | -                                     |
| Insert at head                      | O(1)           | O(1)       | no dummy needed                       |
| Insert at tail (with tail ptr)     | O(1)           | O(1)       | otherwise O(n)                         |
| Insert at middle (have pointer)     | O(1)           | O(1)       | -                                     |
| Delete by pointer                   | O(1) (doubly) / O(n) (singly, need prev) | O(1) | -     |
| Reverse iterative                   | O(n)           | O(1)       | recommended                            |
| Reverse recursive                   | O(n)           | O(n)       | stack overhead, hit limit on long lists|
| Cycle detect Floyd                  | O(n)           | O(1)       | most space-efficient                   |
| Find middle                         | O(n)           | O(1)       | -                                     |
| Merge two sorted                    | O(n + m)       | O(1)       | in-place with dummy                    |
| Reverse K groups                     | O(n)           | O(1)       | -                                     |

---

## 15. Visual Diagram (ASCII + Mermaid)

### Singly linked list

```mermaid
flowchart LR
    Head["head"]
    N1["[1]"] --> N2["[2]"] --> N3["[3]"] --> N4["[4]"] --> N5["[5]"] --> None["None"]
    Head --> N1
```

```
    head                                              tail
      ▼                                                 ▼
   ┌────┐    ┌────┐    ┌────┐    ┌────┐    ┌────┐
   │ 1  │───►│ 2  │───►│ 3  │───►│ 4  │───►│ 5  │───► None
   └────┘    └────┘    └────┘    └────┘    └────┘
```

### Doubly linked list

```
   ┌────┐ ⇄ ┌────┐ ⇄ ┌────┐ ⇄ ┌────┐
   │ 1  │←→│ 2  │←→│ 3  │←→│ 4  │←→ None (both ends)
   └────┘   └────┘   └────┘   └────┘
```

### Insert at head

```
   new=Node(0)
   before: head → [1] → [2] → None
   after : new   → [1] → [2] → None  then head = new
   new → [1] → [2] → None   head=new (val=0)
```

### Delete after target

```
   prev → A → B → C          prev → A → C  ; B is reclaimed
```

### Reverse iteratively — arrow flips

```
   BEFORE:  None ← 1 ← 2 ← 3    (cur points to 1)
   DURING:  None ← 1 ← 2     3  (cur=3, prev=2, after step 3)
   AFTER :  None ← 1 ← 2 ← 3    (prev = 3 is new head)
```

### Floyd's cycle detection

```
   1 → 2 → 3 → 4 → 5
            ▲           │
            └───────────┘     (5.next = 3, cycle)

   slow pos:  1  2  3  4  5  3  4  5 ...
   fast pos:  1  3  5  3  5  3  5 ...
   they meet at 5 (after a few iterations) → cycle exists
```

### Algorithm flowchart — reverse a list

```
          ┌─────────────┐
          │ prev = None │
          │ cur  = head │
          └──────┬──────┘
                 ▼
           while cur:
              ┌── nxt = cur.next ──┐
              ├── cur.next = prev ─┤
              ├── prev = cur       ┤
              └── cur = nxt        ┘
                 ▼
           return prev
```

---

## 16. Beginner Notes

> **Remember:**
> - A linked list is just `Node` objects with `.next` pointers.
> - `Node` should store `val` and `next` (and `prev` for doubly). Use `__slots__` to save memory.
> - **Always use a `dummy = Node(0, head)`** for head-mutating operations; return `dummy.next`.
> - **Two-pointer slow/fast** is the meta-skill: cycle, middle, kth-from-end, palindrome split.
> - **Dummy-driven reverse** flips arrows one-by-one. Practice until muscle memory.
> - Linked lists trade **O(1) random access** for **O(1) insert/delete at known positions**.
> - `collections.deque` is your stdlib Swiss-army linked list.

---

## 17. FAANG Tips

- **Sketch with dummy sentinels first** — eliminates special-case branches in deletion problems.
- **Never compare by `.val` for identity** — use `is` when checking `slow is fast`.
- **Lay out the 4-step reversal** with the interviewer before coding; it pays off under pressure.
- **Find middle fast-finger**: `slow=head, fast=head; while fast and fast.next: slow=slow.next; fast=fast.next.next` gives the **left** middle when length is even.
- For **Add Two Numbers** / **Merge Two Sorted Lists** type problems, start with `dummy = Node(); tail = dummy` and never let go of `tail`.
- For **LRU (146)**, the cleanest pattern: dict stores `key → node` and a doubly list tracks recency. `move_to_front` and `remove_node` are two helpers that share the pointer math.
- Watch recursion limit on big lists: iterative reversal is safer.
- **Palindromic linking** (LC 234): mid + reverse-half + compare halves. Restore it after if asked.
- Talk about **memory locality** when interviewer asks trade-offs: arrays win, linked wins on insert/delete.

---

## 18. Practice Problems

### Easy
- **LC 206** — Reverse Linked List
- **LC 21** — Merge Two Sorted Lists
- **LC 141** — Linked List Cycle
- **LC 83** — Remove Duplicates from Sorted List
- **LC 160** — Intersection of Two Linked Lists

### Medium
- **LC 19** — Remove Nth Node From End
- **LC 143** — Reorder List
- **LC 142** — Linked List Cycle II (find cycle entry)
- **LC 2** — Add Two Numbers
- **LC 92** — Reverse Linked List II (subrange reverse)

### Hard
- **LC 25** — Reverse Nodes in k-Group
- **LC 146** — LRU Cache (doubly list + dict)
- **LC 23** — Merge k Sorted Lists (heap-based merges), see [heap.md](./heap.md)