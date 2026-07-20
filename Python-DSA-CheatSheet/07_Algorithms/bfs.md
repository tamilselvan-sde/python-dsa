# Breadth-First Search (BFS) in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 07 — Algorithms
> 🔗 Related: [dfs.md](./dfs.md) · [recursion.md](./recursion.md) · [Trees](../05_OOP/classes.md) · [Graphs](./searching.md)
> Data: [list.md](../02_Data_Types/list.md) · [deque](../06_Collections/heapq.md) · [big_o.md](../08_Time_Complexity/big_o.md)
> Back to [README](../README.md)

---

## 1. What is it?

**Breadth-First Search (BFS)** explores a graph or tree **level by level**. Starting from a source node, it visits **all neighbours at distance 1**, then **all neighbours at distance 2**, and so on — like a ripple spreading outward on a pond.

It uses a **FIFO queue** (first-in-first-out): newly discovered nodes are appended to the back, and the node to process next is popped from the front. A **visited** set (or distance array) prevents reprocessing.

```
        1            Level 0  →  {1}
       / \
      2   3          Level 1  →  {2, 3}
     / \   \
    4   5   6        Level 2  →  {4, 5, 6}
```

**Key property:** In an **unweighted** graph, the first time BFS reaches a node, it has found the **shortest path** (in number of edges) to that node. This is the single most important fact about BFS and the reason it appears in dozens of interview problems.

**What problem it solves:** Level-order traversal, shortest path on unweighted graphs, connected components, flood fill, bipartite check, minimum distance in a grid.

**Real-world analogy:** Imagine you are standing at the centre of a city. BFS = "explore every street that is one block away, then every street two blocks away, ..." You find the nearest coffee shop in the fewest number of turns.

---

## 2. Why do we use it?

| Benefit | Detail |
|--------|--------|
| **Shortest path (unweighted)** | BFS guarantees the first arrival = minimum edges. DFS does NOT. |
| **Level-order information** | You naturally know which nodes are at depth k — critical for "minimum depth", "level average", "right-side view". |
| **Early termination** | As soon as you reach the target you can stop — the path found is optimal. |
| **No recursion depth issues** | BFS uses an explicit queue, so it won't blow the Python recursion limit (~1000). |
| **Completeness** | On a finite graph BFS will always find a goal if one exists. |

**Brute force vs BFS:** A brute-force / DFS may also find a path, but it won't be shortest and it may loop forever on cycles without a visited check. BFS is the *right tool* whenever the graph is unweighted and you need the shortest path or level structure.

---

## 3. When should I choose it? (Decision Table)

| Situation | Use BFS | Use DFS | Notes |
|-----------|:------:|:------:|-------|
| Shortest path in **unweighted** graph | ✅ | ❌ | BFS is *the* algorithm here. |
| Shortest path in **weighted** graph | ❌ | ❌ | Use Dijkstra / Bellman-Ford. |
| Level-order / minimum depth | ✅ | ❌ | BFS gives levels naturally. |
| Connected components | ✅ | ✅ | Both work; BFS avoids deep recursion. |
| Cycle detection (directed) | ❌ | ✅ | Colour-state DFS is idiomatic. |
| **Topological sort** | ❌ | ✅ | Or Kahn's BFS variant (in-degree). |
| Bipartite check | ✅ | ✅ | BFS colouring is often easier to reason about. |
| Flood fill / "number of islands" | ✅ | ✅ | DFS is shorter code; BFS avoids recursion limit on 200×200 grids. |
| Word ladder / shortest transformation | ✅ | ❌ | Classic BFS shortest path. |
| "All paths" enumeration | ❌ | ✅ | BFS would blow memory storing all paths. |
| Maze with min steps | ✅ | ❌ | Unweighted → BFS. |

**Rule of thumb:** if the problem says **"shortest", "minimum steps", "nearest", "level"** on an unweighted structure → **BFS**.

---

## 4. Syntax

### Graph BFS template (adjacency list)

```python
from collections import deque

def bfs(graph, start):
    visited = {start}
    queue  = deque([start])
    order  = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for nei in graph[node]:
            if nei not in visited:
                visited.add(nei)
                queue.append(nei)
    return order
```

### Tree BFS (level-order) template

```python
from collections import deque

def level_order(root):
    if not root: return []
    out, q = [], deque([root])
    while q:
        level_size = len(q)
        level = []
        for _ in range(level_size):
            n = q.popleft()
            level.append(n.val)
            if n.left:  q.append(n.left)
            if n.right: q.append(n.right)
        out.append(level)
    return out
```

### Shortest-path BFS (distance driven)

```python
from collections import deque
def shortest_path(graph, src, dst):
    dist = {src: 0}
    q = deque([src])
    while q:
        u = q.popleft()
        if u == dst: return dist[u]
        for v in graph[u]:
            if v not in dist:
                dist[v] = dist[u] + 1
                q.append(v)
    return -1
```

---

## 5. Basic Example

```python
from collections import deque

graph = {
    1: [2, 3],
    2: [1, 4, 5],
    3: [1, 6],
    4: [2],
    5: [2],
    6: [3],
}

def bfs(g, s):
    seen = {s}
    q = deque([s])
    out = []
    while q:
        u = q.popleft()
        out.append(u)
        for v in g[u]:
            if v not in seen:
                seen.add(v)
                q.append(v)
    return out

print(bfs(graph, 1))     # [1, 2, 3, 4, 5, 6]
```

**Output:**
```
[1, 2, 3, 4, 5, 6]
```

---

## 6. Step-by-Step Dry Run

Graph:
```
    1 — 2
    |   |
    3   4
    |
    5
```
Adjacency list: `1:[2,3], 2:[1,4], 3:[1,5], 4:[2], 5:[3]`

| Step | queue (front → back) | visited | popped | enqueue new |
|-----:|----------------------|---------|--------|-------------|
| 0    | [1]                  | {1}     | —      | —           |
| 1    | [2, 3]              | {1,2,3} | 1      | 2, 3        |
| 2    | [3, 4]              | {1,2,3,4}| 2     | 4           |
| 3    | [4, 5]              | {1,2,3,4,5}| 3  | 5           |
| 4    | [5]                 | —       | 4      | —           |
| 5    | []                   | —       | 5      | —           |

**Visit order:** `1 → 2 → 3 → 4 → 5`

**Wave view:**
```
Level 0:  1
Level 1:  2  3
Level 2:  4  5
```

---

## 7. Built-in Methods (deque)

`collections.deque` is the right queue for BFS — `popleft()` is **O(1)**, whereas `list.pop(0)` is O(n).

| Method / Syntax | Purpose | Example | Complex-ity | Interview use | Mistakes |
|-----------------|---------|---------|-------------|----------------|----------|
| `deque([1,2,3])` | create a double-ended queue | `q = deque([root])` | O(n) | initialise BFS queue | using `deque(1)` — needs iterable |
| `q.append(x)` | enqueue to the right (rear) | `q.append(neighbour)` | O(1) | push newly discovered node | appending to the left by mistake |
| `q.popleft()` | dequeue from the front (FIFO) | `u = q.popleft()` | O(1) | get next node to process | using `pop()` → LIFO = DFS! |
| `q.appendleft(x)` | push to front (rare) | undo / re-prioritise | O(1) | 0-1 BFS (push 0-cost to front) | confusing with plain BFS |
| `q.pop()` | pop from right (LIFO) | backtracking | O(1) | rarely in pure BFS | turns it into DFS if used as main pop |
| `len(q)` | queue size | `for _ in range(len(q))` for level slice | O(1) | level-order traversal | calling `len` then mutating inside |
| `q[0]` | peek front without removing | front element | O(1) | 0-1 BFS frontier | mutates if you assign |
| `list(q)` | snapshot | debugging | O(n) | print queue state | — |

**Mnemonic:** `append` to the **back**, `popleft` from the **front** → FIFO.

**Shortcut:** For level-order, snapshot `len(q)` *before* the inner loop; otherwise the loop keeps processing newly added children of the current level.

```python
while q:
    k = len(q)                    # freeze size of THIS level
    for _ in range(k):            # process exactly k nodes
        u = q.popleft()
        ...
        q.append(...)             # children belong to next level
```

**Mistake:** writing `for _ in q:` — iterating over a mutating queue skips/expands unexpectedly.

---

## 8. Interview Example — 102. Binary Tree Level Order Traversal

```python
from collections import deque

class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val  = val
        self.left = left
        self.right = right

def levelOrder(root):
    if not root: return []
    out, q = [], deque([root])
    while q:
        k = len(q)
        level = []
        for _ in range(k):
            n = q.popleft()
            level.append(n.val)
            if n.left:  q.append(n.left)
            if n.right: q.append(n.right)
        out.append(level)
    return out

#        3
#       / \
#      9  20
#         / \
#        15  7
t = TreeNode(3, TreeNode(9), TreeNode(20, TreeNode(15), TreeNode(7)))
print(levelOrder(t))
```
**Output:**
```
[[3], [9, 20], [15, 7]]
```

**Pattern:** snapshot `len(q)`, process exactly that many, their children go to the next level.

---

## 9. When NOT to use

- **Weighted graphs.** Use Dijkstra (weights ≥ 0) or Bellman-Ford (negative weights). BFS treats every edge as weight 1.
- **Deep / huge state space where memory is tight.** BFS keeps the *entire frontier* in memory — exponential blow-up at depth d in a tree (b^d nodes). DFS with backtracking uses O(d) memory.
- **You need ALL paths, not just the shortest.** Enumerate with DFS + backtracking; BFS stores too many partial paths.
- **Topological sort of a DAG.** Use post-order DFS or Kahn's algorithm (in-degree BFS — which *is* BFS-like but with the in-degree condition, not a pure distance BFS).
- **Strongly / weakly connected components.** Tarjan / Kosaraju are DFS-based.
- **Binary tree height.** BFS computes *level-order*, but for the *height* a DFS is cleaner and O(h) space vs O(w).

---

## 10. Common Mistakes

1. **Using `list.pop(0)` instead of `deque.popleft()`.** O(n) per pop → O(n²) overall. Always import `deque`.
2. **Forgetting `visited` / marking a node visited too late.** Push into queue → set visited **before** the loop, or you may enqueue the same node multiple times.
3. **Marking visited on dequeue instead of enqueue.** Causes the same node to be enqueued many times (once per incoming edge), blowing up the queue to O(E).
4. **Mutating the queue while iterating `for u in q`.** Either snapshot `len(q)` or collect results, then mutate.
5. **Level-order: wrong loop nesting.** If you forget the inner `for _ in range(len(q))`, you lose the level grouping.
6. **Not checking boundaries in grid BFS.** `0 <= r < R and 0 <= c < C` before enqueueing neighbours.
7. **2D grid: using riches of `visited` set vs colouring visited nodes in-place.** Both fine, but keep them consistent.
8. **Wrong starting state.** BFS from multiple sources? Enqueue *all* sources at distance 0 (rotting oranges, walls-and-gates).
9. **`deque` imported from `queue` (`queue.Queue`) instead of `collections.deque`.** `queue.Queue` is for threading, slow and overkill.
10. **Treating undirected graph as directed.** Build adjacency both ways; otherwise BFS can't traverse the reverse.

---

## 11. Memory Tricks

- **"Lake ripple"** — BFS is the ripple growing outward from the start. Each level = one ring further out.
- **Queue = line at a coffee shop.** First person in line is served first (FIFO). New people join at the back.
- **`popleft` reads like "pop-left = FIFO"** — pick the side whose name you remember; the *opposite* side is where you `append`.
- **Distance = level number.** The level at which you first see a node is the shortest distance.
- **Multi-source BFS = many ripples starting together** → imagine dropping many stones at once; they meet at the boundary.

---

## 12. Interview Shortcuts

- Start every BFS solution with `from collections import deque` — interviewers expect it.
- Use a **distance dict/array instead of a plain visited set** — gives you the answer for free in shortest-path problems.
- Multi-source BFS: push **all starting nodes** onto the queue with distance 0; the BFS property still holds.
- 0-1 BFS: when edge weights ∈ {0, 1}, use `deque`; push 0-weight edges to the **front** (`appendleft`) and 1-weight edges to the **back**. Avoids Dijkstra's log factor.
- Grid = graph. Neighbours of `(r,c)` are the 4 (or 8) adjacent cells. Encode as `(r±1, c)`, `(r, c±1)`.
- Level-order pattern: `for _ in range(len(q))` is the interview cheat code.

---

## 13. Cheat Sheet Table

| Concept | BFS detail |
|--------|------------|
| Data structure | FIFO queue (`collections.deque`) |
| Traversal order | level-by-level (wave-front) |
| Visit time | first reach = shortest path (unweighted) |
| Memory | O(V) frontier — wide trees blow up |
| Typical use cases | shortest path (unweighted), level-order, flood fill, bipartite, word ladder |
| Cycles handled by | `visited` set / distance array |
| Multi-source variant | enqueue all sources at distance 0 |
| 0-1 BFS | `deque` + `appendleft` for 0-weight edges |
| When to avoid | weighted graphs, enumerating all paths |

---

## 14. Time Complexity Table

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Build adjacency list | O(V + E) | input pre-processing |
| Single-source BFS | **O(V + E)** | each node + edge processed once |
| Level-order of tree (n nodes) | O(n) | same as traversal |
| Shortest path BFS | O(V + E) | unweighted only |
| Grid BFS R×C | **O(R·C)** | 4 neighbours each |
| Multi-source BFS | O(V + E) | same as single source |
| Space (queue + visited) | **O(V)** | worst case: star graph keeps V/2 nodes on frontier |

For trees: O(n) time, O(w) space where w = max width.

---

## 15. Visual Diagram (ASCII)

### Wave-by-wave BFS on a graph

```
          source
            ●───── Level 0  (distance 0)
           / \
          ●   ● ─────── Level 1  (distance 1)
         /|   |\
        ● ●   ● ● ───── Level 2  (distance 2)
         \|   |/
          ●   ● ─────── Level 3  (distance 3)
```

### Queue evolution

```
Step 0:  [ A ]                       visited: A
Step 1:  [ B, C ]                    visited: A B C        (popped A)
Step 2:  [ C, D, E ]                visited: A B C D E    (popped B)
Step 3:  [ D, E, F ]                visited: ... F         (popped C)
Step 4:  [ E, F ]                   (popped D, no new)
Step 5:  [ F ]                      (popped E)
Step 6:  [ ]                        (popped F)
```

### BFS vs DFS traversal style

```
BFS:  A → B → C → D → E → F        (horizontal waves)
DFS:  A → B → D → E → C → F        (vertical plunge)

         A
        / \
       B   C
      / \   \
     D   E   F
```

---

## 16. Beginner Notes (Remember block)

> **Remember:**
> - BFS = **queue (FIFO)**, `deque.popleft()`.
> - First time you reach a node in an **unweighted** graph ⇒ shortest path.
> - **Mark visited on ENQUEUE**, not on dequeue (prevents duplicates in the queue).
> - Level-order trick: `for _ in range(len(q))`.
> - Multi-source BFS = push *all* starts at distance 0.
> - Import `deque` from **`collections`**, never use `list.pop(0)`.
> - Grid = graph: cells = nodes, 4-neighbours = edges.
> - Time **O(V+E)**, space **O(V)**.

---

## 17. FAANG Tips

- **Rotting oranges / walls-and-gates** = multi-source BFS. Push all rotten/empty cells first. Classic FAANG problem pattern.
- **Word ladder (127)** = BFS where nodes = words, edges = 1-letter diffs. Build adjacency with wildcard patterns (`h*t` → `hit, hot, hat`) to avoid O(n²) edge enumeration.
- **Cheapest Flights K Stops (787)** = BFS layered by hop count, *not* pure shortest path — keep the best price *within k+1 layers*. Bellman-Ford for k+1 iterations.
- **01 Matrix (542)** = multi-source BFS from all zeroes — distances ripple outward.
- **Clone Graph (133)** = BFS with a `visited` map (node → cloned node). Clone upon first discovery.
- **Number of Islands (200)** = grid BFS iterating all cells, launching BFS only on unvisited land — each BFS marks one island.
- **Bipartite check** = BFS with two-colour tag; if a neighbour already has the same colour as the current node → not bipartite.
- **0-1 BFS** = use `deque` and `appendleft` for zero-weight edges. Classic trick to avoid Dijkstra log factor.

---

## 18. Practice Problems

### Easy
1. **637. Average of Levels in Binary Tree** — level-order BFS, average each level.
2. **111. Minimum Depth of Binary Tree** — BFS finds the first leaf (shortest depth).
3. **429. N-ary Tree Level Order Traversal** — minor variant of level-order template.
4. **733. Flood Fill** — BFS/DFS on grid; simpler with BFS for colour swap.
5. **559. Maximum Depth of N-ary Tree** — BFS count levels.

### Medium
1. **200. Number of Islands** — classic grid BFS.
2. **102. Binary Tree Level Order Traversal** — bread-and-butter BFS.
3. **994. Rotting Oranges** — multi-source BFS, return minutes until all fresh rot.
4. **542. 01 Matrix** — multi-source BFS from zeroes.
5. **127. Word Ladder** — BFS on word graph with wildcard adjacency.
6. **133. Clone Graph** — BFS + hashmap clone.
7. **787. Cheapest Flights Within K Stops** — layered BFS / Bellman-Ford iter.

### Hard
1. **126. Word Ladder II** — BFS to build shortest-path DAG, then DFS to reconstruct paths.
2. **1293. Shortest Path in a Grid with Obstacles Elimination** — BFS with state `(r, c, k)` — obstacles left.
3. **773. Sliding Puzzle** — BFS on the state space of board configurations.
4. **815. Bus Routes** — BFS over bus routes treated as super-nodes.

---

> Next: [dfs.md](./dfs.md) — the depth counterpart.