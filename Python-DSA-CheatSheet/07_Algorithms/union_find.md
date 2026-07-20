# Union-Find (Disjoint Set Union) in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 07 — Algorithms
> 🔗 Related: [sorting.md](./sorting.md) · [recursion.md](./recursion.md) · [binary_search.md](./binary_search.md)
> Data: [graph.md](../06_Graphs/graph.md) · [big_o.md](../08_Time_Complexity/big_o.md)
> Back to [README](../README.md)

---

## 1. What is it?

**Union-Find**, also called **Disjoint Set Union (DSU)**, is a data structure that maintains a collection of **disjoint (non-overlapping) sets** and supports two lightning-fast operations:

| Operation | Meaning |
|-----------|---------|
| `find(x)`  | Return the **representative (root)** of the set containing `x` |
| `union(x, y)` | **Merge** the sets containing `x` and `y` if they are different |

Optionally:
- `connected(x, y)` → `find(x) == find(y)`
- `count()` → number of distinct sets / connected components

### Real-world analogy
Picture students in a playground forming friendship circles. Initially every student is in their own circle. When two students from different circles become friends, **their entire circles merge** into one bigger circle. Union-Find tracks *who is in which circle* and answers *"are person A and person B in the same circle?"* in near-constant time.

### What problem it solves
Any problem reducible to **"merge groups / detect same-group membership"** with up to ~10⁶ union/find queries — connected components, MST (Kruskal), dynamic connectivity, equivalence classes, cycle detection in undirected graphs.

Cross-link → see [graph.md](../06_Graphs/graph.md) for Kruskal MST and connected-components via BFS/DFS.

---

## 2. Why do we use it?

- **Blazing fast**: with two optimizations (path compression + union-by-rank), each operation is **amortized O(α(n))** ≈ O(1) where α is the inverse Ackermann function — practically a constant for n ≤ 10⁸⁰.
- **Simple**: ~15 lines of OOP code replaces dozens of graph-traversal lines.
- **Online**: handles unions and queries **interleaved** — you don't need the whole graph up front (unlike DFS components).
- **Memory lean**: just two int arrays of size n.
- Beats BFS/DFS for *dynamic* "are these connected?" queries that arrive piecemeal.

---

## 3. When should I choose it?

| Scenario | Use Union-Find? | Why |
|----------|-----------------|-----|
| Connected components in **static** undirected graph | ✅ (or DFS) | Both work; UF is simpler when edges stream in |
| **Dynamic** connectivity (edges added over time) | ✅ | DFS every query = O(V+E) is too slow |
| Detect cycle while building undirected graph | ✅ | Textbook Kruskal usage |
| Minimum Spanning Tree (Kruskal) | ✅ | Sort edges, union endpoints, skip cycle-makers |
| Equations: `a==b`, `c!=d` satisfiability | ✅ | Union equalities, then check inequalities |
| Merge overlapping intervals / accounts by shared email | ✅ | Each shared email = union |
| **Shortest path** between nodes | ❌ | Use BFS / Dijkstra |
| **Cycle in directed graph** | ❌ | Use topological sort / DFS colors |
| **Reachability tree** / DAG ancestors | ❌ | Use DFS / BIT |

**Decision rule of thumb:** "I need to merge groups and answer same-group queries" → **Union-Find**.

---

## 4. Syntax

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))   # each node is its own parent
        self.rank   = [0] * n          # tree height (for union-by-rank)
        self.count  = n                 # # of disjoint sets

    def find(self, x):
        # PATH COMPRESSION: flatten the tree as we walk up
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, x, y):
        rx, ry = self.find(x), self.find(y)
        if rx == ry:                    # already same set → no-op
            return False
        # UNION BY RANK: attach shorter tree under taller one
        if self.rank[rx] < self.rank[ry]:
            rx, ry = ry, rx
        self.parent[ry] = rx
        if self.rank[rx] == self.rank[ry]:
            self.rank[rx] += 1
        self.count -= 1
        return True

    def connected(self, x, y):
        return self.find(x) == self.find(y)
```

---

## 5. Basic Example

```python
uf = UnionFind(6)
uf.union(0, 1)
uf.union(2, 3)
uf.union(1, 2)          # merges {0,1} and {2,3}
print(uf.connected(0, 3))   # True
print(uf.connected(0, 4))   # False
print(uf.count)              # 3   (sets: {0,1,2,3}, {4}, {5})
```

**Output:**
```
True
False
3
```

---

## 6. Step-by-Step Dry Run

Operations on 7 nodes (0..6):

```
init:   parent=[0,1,2,3,4,5,6]  rank=[0,0,0,0,0,0,0]  count=7

union(1,2):  find(1)=1  find(2)=2  rank equal → parent[2]=1, rank[1]=1   count=6
                 1
                /
               2

union(3,4):  parent[4]=3                                          count=5
                 1              3
                /              /
               2              4

union(2,4):  find(2)=1 (root) find(4)=3 (root)
             rank[1]=1, rank[3]=0 → parent[3]=1 (shorter under taller)  count=4
                     1
                    /|\
                   2 3 4   ← 3's child 4 stays attached

union(5,6):  parent[6]=5                                          count=3
                     1           5
                    /|\\          |
                   2 3 4          6

find(4):     walks 4→3→1, path-compress: parent[4]=1, parent[3]=1
                     1
                    /|\\
                   2 3 4     (flat!)

connected(2,4):  find(2)=1 == find(4)=1  → True
connected(0,6):  find(0)=0  != find(6)=5 → False
```

---

## 7. Built-in Methods / Idioms

Python has **no built-in Union-Find** (unlike C++ DSU or Java's `DisjointSet`). You hand-roll it. Relevant idioms:

| Idiom | Purpose | Syntax | Complexity | Interview use | Mistakes |
|-------|---------|--------|-----------|---------------|----------|
| `parent = list(range(n))` | init each node its own parent | `list(range(n))` | O(n) | Always at setup | Using `[0]*n` (all share root 0 — wrong!) |
| `self.parent[x] = self.find(self.parent[x])` | **Path compression** — point node straight to root | inside `find` | amortized O(α) | Must-have | Forgetting to assign return → no compression |
| Union by **rank** | keep trees shallow | compare `rank[rx] vs rank[ry]` | amortized O(α) | Combine with compression | Using `size` array name for rank → logic bug |
| Union by **size** | alternative: attach smaller set under larger | `if size[rx] < size[ry]: swap; size[rx]+=size[ry]` | amortized O(α) | Good when you need set sizes | Forgetting to add sizes when merging |
| `count = n; count -= 1 each merge` | track # of components | member field | O(1) | LC 547, 323 | Decrementing on failed union |
| **Remove node** trick | DSU can't really delete | soft-delete flag | n/a | Rare | Trying to "un-union" — not supported efficiently |

**Shortcut:** `find` with compression can be written as a one-liner:
```python
def find(self, x):
    return x if self.parent[x]==x else (self.parent[x] := self.find(self.parent[x]))
```
(walrus `:=` writes back the compressed root — Python 3.8+).

---

## 8. Interview Example

**LeetCode 547 — Number of Provinces:** Given `isConnected[i][j]==1` means city i is directly connected to city j, return number of provinces (connected components).

```python
def findCircleNum(isConnected):
    n = len(isConnected)
    uf = UnionFind(n)
    for i in range(n):
        for j in range(i+1, n):        # symmetric matrix → upper triangle
            if isConnected[i][j]:
                uf.union(i, j)
    return uf.count
```
Walkthrough: each `isConnected[i][j]==1` is an edge → union the two endpoints. After processing all edges, `uf.count` answers in O(n² · α) ≈ O(n²).

**Variations:**
- **LC 684 Redundant Connection:** union edges; first edge that connects two already-connected nodes is the answer.
- **LC 990 Satisfiability of Equality Equations:** two passes — union all `==`, then verify all `!=`.

---

## 9. When NOT to use

- **Shortest paths**: UF gives connectivity, not distance. Use BFS/Dijkstra/Bellman-Ford.
- **Directed cycles / topological order**: UF only knows undirected equivalence. Use DFS-Colors / Kahn.
- **Delete an edge**: Union-Find cannot efficiently remove a union. Use link-cut trees (overkill for interviews).
- **Update vertex weights**: UF structures by identity, not value. A Fenwick / Segment Tree fits better.
- **Order between sets matters**: UF has no notion of "set A comes before set B". You'd want topological sort.

---

## 10. Common Mistakes

1. `parent = [0] * n` — every node thinks root is `0`. **Always** `list(range(n))`.
2. Forgetting to compress in `find` — recursion returns root but you don't write it back, so trees stay tall → O(n) find.
3. Comparing `rank` but finishing with `parent[ry]=rx` even when you swapped — double-check swap consistency.
4. `union` returns nothing → caller can't tell if a merge happened (needed for cycle detection / Kruskal).
5. Calling `union(x,y)` then `find(x)` immediately and expecting it equals `find(y)` → it does, but only *after* the union call finishes; don't read `parent` array manually to check.
6. With **1-indexed inputs** (LC 323 gives `n=5` labeled `1..n`), allocate `n+1` cells and ignore index 0.
7. In **Kruskal**, sorting edges but forgetting to break on the successful `(V-1)`-th union → wasted passes.
8. Using `rank` to *answer* queries — `rank` is an internal heuristic, NOT the tree depth after path compression (compression doesn't update rank).

---

## 11. Memory Tricks

- 🌲 **"Tall trees are bad; flatten them"** — that's path compression.
- 🏔️ **"Shorter hill goes under taller hill"** — union-by-rank.
- 🧩 **"Merge circles, don't split"** — only operation is union; never delete.
- 🔑 **`find` = "who's the boss?"**, `union` = "make them share a boss".
- ⚡ **α ≈ 4** for any realistic n — treat as O(1).

---

## 12. Interview Shortcuts

- Allocate `parent`, `rank` arrays of size `n` (or `n+1` for 1-indexed).
- `count` field → instant answer for "number of components".
- `union` returning `bool` (True if merged) → cycle detection and Kruskal in 3 lines.
- **Two-pass pattern** for `==`/`!=` problems (LC 990): union all equals, then check all not-equals.
- **Account merge** (LC 721): hash email → id, union ids sharing an account, then group.
- **Connect nodes on a grid** (LC 200 alt): flatten 2D cells to `r*cols+c`; union with right & down neighbors only.

---

## 13. Cheat Sheet Table

| Component | Snippet / Concept | Notes |
|-----------|-------------------|-------|
| Init | `parent=list(range(n)); rank=[0]*n; count=n` | O(n) |
| Find + compression | recurse, assign back | O(α) amortized |
| Union by rank | compare rank, attach shorter | O(α) amortized |
| Connected | `find(x)==find(y)` | O(α) |
| Components | read `count` field | O(1) |
| Kruskal | sort edges, union if different | O(E log E) total |
| Path compression | `parent[x]=find(parent[x])` | flattens tree |
| Id mapping | `dict[str,int]` for strings | for LC 721, 990 |

---

## 14. Time Complexity Table

| Operation | Naïve | + Union by Rank | + Path Compression | + Both |
|-----------|-------|------------------|--------------------|----|
| `find`    | O(n)  | O(log n)         | amortized O(α)     | **amortized O(α)** |
| `union`   | O(n)  | O(log n)         | amortized O(α)     | **amortized O(α)** |
| `connected` | O(n) | O(log n)       | amortized O(α)     | **amortized O(α)** |
| Build     | O(n)  | O(n)              | O(n)               | O(n) |

α(n) ≤ 4 for all n ≤ 10⁸⁰ — effectively constant. Full bounds: O((n+m)·α(n)) for n elements, m operations.

Space: **O(n)** for the two arrays.

---

## 15. Visual Diagram (ASCII)

**Two trees being merged with union-by-rank:**

```
BEFORE union(C, E):                     AFTER (rank[B]=2 > rank[D]=1, attach D under B):

     B (rank 2)         D (rank 1)            B (rank 2)
    / \                / \                  / | | \
   A   C              E   F                A  C D  E F   ← D's subtree came along
                                              |
                                              F?  (actually F stays under D;
                                                   path-compressed later on find)
```

**Path compression on `find(F)`:**
```
walk F→D→B, return B, and on the way back:
    parent[F] = B
    parent[D] = B            ← tree flattens

Result:        B
             / | | \
            A  C D  F   (height-1 leaves)
```

**Flowchart of a single `union(x, y)`:**

```
   union(x, y)
       │
       ▼
   rx = find(x);  ry = find(y)
       │
       ├── rx == ry ──► return False (already same set)
       │
       ▼
   rank[rx] < rank[ry] ? ──yes──► swap(rx, ry)   # ensure rx is the taller
       │ no
       ▼
   parent[ry] = rx
   rank[rx] += (rank[rx] == rank[ry])            # bump only on tie
   count -= 1
   return True
```

---

## 16. Beginner Notes

> **Remember:**
> - DSU = "merge groups, ask same-group": `find` (root) + `union` (merge).
> - **Two magic optimizations**: path compression + union-by-rank ⇒ O(α(n)) ≈ O(1).
> - `parent[x] = x` self-loop means "I am the root".
> - After path compression, **rank is inaccurate** — don't use it for anything except the union decision.
> - UF cannot delete edges. *"Union is forever."*
> - For grid connectivity, flatten `(r,c)` → `r*cols + c`.
> - Cross-link → [graph.md](../06_Graphs/graph.md) (Kruskal/BFS-vs-UF) and [big_o.md](../08_Time_Complexity/big_o.md).

---

## 17. FAANG Tips

- **Mention α** when asked about complexity — interviewers love it.
- When given strings instead of ints (LC 721, 990), map them with a dict and a `next_id` counter, then run UF.
- **Kruskal**: sort edges by weight; union edges in order; stop after V−1 successful unions (spanning tree).
- **Redundant Connection (LC 684)**: union each edge; the first edge whose endpoints are already connected is the answer.
- **Number of Islands II (LC 305, hard variant)**: dynamic land appearance — UF excels here where BFS per query would TLE.
- **Satisfiability of Equality Equations (LC 990)**: classic two-pass. Mention it as a UF showcase.
- For **large n** (>10⁶) and few unions, prefer **iterative find** to avoid Python recursion limit — or `sys.setrecursionlimit`.
- Ask: "Are edges only ever **added**?" If yes → almost certainly UF.

---

## 18. Practice Problems

| # | Title | Difficulty | LeetCode |
|---|-------|-----------|----------|
| 547 | Number of Provinces | Medium | [link](https://leetcode.com/problems/number-of-provinces/) |
| 200 | Number of Islands (DSU alt) | Medium | [link](https://leetcode.com/problems/number-of-islands/) |
| 323 | Number of Connected Components in Undirected Graph | Medium | [link](https://leetcode.com/problems/number-of-connected-components-in-an-undirected-graph/) |
| 684 | Redundant Connection | Medium | [link](https://leetcode.com/problems/redundant-connection/) |
| 685 | Redundant Connection II (directed) | Hard | [link](https://leetcode.com/problems/redundant-connection-ii/) |
| 990 | Satisfiability of Equality Equations | Medium | [link](https://leetcode.com/problems/satisfiability-of-equality-equations/) |
| 721 | Accounts Merge | Medium | [link](https://leetcode.com/problems/accounts-merge/) |
| 1319 | Number of Operations to Make Network Connected | Medium | [link](https://leetcode.com/problems/number-of-operations-to-make-network-connected/) |
| 305 | Number of Islands II | Hard | [link](https://leetcode.com/problems/number-of-islands-ii/) |
| 1101 | The Earliest Moment When Everyone Become Friends | Medium | [link](https://leetcode.com/problems/the-earliest-moment-when-everyone-become-friends/) |

---

**Cross-links:** [graph.md](../06_Graphs/graph.md) · [big_o.md](../08_Time_Complexity/big_o.md) · [recursion.md](./recursion.md) · [sorting.md](./sorting.md) · Back to [README](../README.md)