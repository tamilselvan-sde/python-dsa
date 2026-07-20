# Algorithm Learning Plan — When & How to Use Each Pattern

> Use this as a **decision guide**. When you read a problem, scan the "When to Use" column first — it tells you which pattern fits based on the problem's signals.

---

## How to Identify the Right Pattern

| Signal in Problem | Pattern to Try |
|--------------------|----------------|
| "sorted array" + find target | Binary Search |
| "pair"/"triplet" + sorted | Two Pointers |
| "subarray"/"substring" + size or longest/shortest | Sliding Window |
| "next greater"/"valid pairs"/"nested" | Stack |
| "all permutations"/"all combinations" | Backtracking |
| "min/max k elements"/"top k" | Heap |
| "grid"/"connected"/"islands" | Graph DFS/BFS |
| "overlapping subproblems"/"count ways" | Dynamic Programming |
| "lookup"/"frequency"/"pair with sum" | HashMap / Set |
| "reverse"/"cycle"/"merge" lists | Linked List pointers |
| "level by level"/"depth"/"height" | Tree DFS/BFS |

---

## 1. HashMap / Set

### When to Use
- You need **O(1) lookup** of values, indices, or frequencies.
- Problem asks: "find pair", "contains duplicate", "two sum", "count occurrences".
- You want to **trade space for time** (cache results).

### How to Use
1. Decide what the **key** and **value** represent.
2. Iterate once, storing info as you go.
3. Check the map before storing to find complements/matches.

```python
# Template: Frequency map
freq = {}
for x in items:
    freq[x] = freq.get(x, 0) + 1

# Template: Seen set
seen = set()
for x in items:
    if x in seen:
        # duplicate found
        pass
    seen.add(x)

# Template: Index map (Two Sum style)
seen = {}
for i, num in enumerate(nums):
    if target - num in seen:
        return [seen[target - num], i]
    seen[num] = i
```

### Common Pitfalls
- Using a list instead of a set for lookup → O(n) instead of O(1).
- Forgetting to store the index **before** checking (order matters in Two Sum).

---

## 2. Two Pointers

### When to Use
- **Sorted array** + find pair/triplet.
- Need to compare elements from **both ends** or find a meeting point.
- "in-place" removal/move operations.
- Palindrome validation.

### How to Use
1. Sort the array (if not already sorted).
2. Place one pointer at the **start**, another at the **end** (or both at start for slow/fast).
3. Move pointers based on comparison until they meet.

```python
# Template: Opposite ends
left, right = 0, len(nums) - 1
while left < right:
    if condition(nums[left], nums[right]):
        # process
        left += 1
    else:
        right -= 1

# Template: Slow / fast (remove duplicates in-place)
slow = 0
for fast in range(1, len(nums)):
    if nums[fast] != nums[slow]:
        slow += 1
        nums[slow] = nums[fast]
```

### Common Pitfalls
- Forgetting to sort first.
- Infinite loop if both pointers don't move on every iteration.

---

## 3. Sliding Window

### When to Use
- Problem asks for **longest/shortest/maximum/minimum** subarray or substring.
- Contains words like "contiguous", "consecutive", "window of size k".
- You need to track a **running sum/product/count** over a range.

### How to Use
1. **Fixed window**: compute initial window of size k, then slide by adding next element and removing the first.
2. **Variable window**: expand right, shrink left when condition breaks, track best answer.

```python
# Template: Fixed window size k
window_sum = sum(nums[:k])
for right in range(k, len(nums)):
    window_sum += nums[right] - nums[right - k]
    # update max/min

# Template: Variable window
left = 0
for right in range(len(nums)):
    # add nums[right] to window
    while window_invalid:
        # remove nums[left], shrink
        left += 1
    # update answer with current window size (right - left + 1)
```

### Common Pitfalls
- Forgetting to **shrink** the left side (window grows unbounded).
- Using max on an empty answer variable (initialize with -inf / 0).

---

## 4. Binary Search

### When to Use
- **Sorted array** or **monotonic condition** (search space can be split).
- Problem asks for "first/last position", "insert position", or search in rotated sorted array.
- Answer has **binary property**: everything left of answer is True, right is False (or vice versa).

### How to Use
1. Define `left` and `right` bounds.
2. Compute `mid = (left + right) // 2`.
3. Decide which half to discard based on comparison.
4. Loop until `left > right` (standard) or `left >= right` (first-true pattern).

```python
# Template: Exact match
left, right = 0, len(nums) - 1
while left <= right:
    mid = (left + right) // 2
    if nums[mid] == target:
        return mid
    elif nums[mid] < target:
        left = mid + 1
    else:
        right = mid - 1
return -1

# Template: Find first True ("leftmost")
left, right = 0, n
while left < right:
    mid = (left + right) // 2
    if condition(mid):
        right = mid
    else:
        left = mid + 1
return left

# Template: Find last True ("rightmost")
left, right = 0, n
while left < right:
    mid = (left + right + 1) // 2
    if condition(mid):
        left = mid
    else:
        right = mid - 1
return left
```

### Common Pitfalls
- Off-by-one: `left <= right` vs `left < right` changes the meaning.
- Integer overflow in other languages (use `left + (right - left) // 2` if needed).
- Not recognizing a problem as binary searchable (e.g., "minimum capacity to ship packages").

---

## 5. Stack

### When to Use
- Problem involves **nesting** (parentheses, tags).
- Need to find **next greater/smaller** element.
- Function call simulation, expression evaluation.
- Monotonic increasing/decreasing stack problems.

### How to Use
1. Push elements onto the stack.
2. Before pushing, pop all elements that violate your invariant.
3. Process what you popped.

```python
# Template: Next Greater Element
stack = []
res = [-1] * len(nums)
for i in range(len(nums)):
    while stack and nums[i] > nums[stack[-1]]:
        res[stack.pop()] = nums[i]
    stack.append(i)

# Template: Valid Parentheses
pairs = {")": "(", "}": "{", "]": "["}
stack = []
for c in s:
    if c in pairs:
        if not stack or stack.pop() != pairs[c]:
            return False
    else:
        stack.append(c)
return not stack
```

### Common Pitfalls
- Forgetting to check if stack is **empty** before pop.
- Not returning True if stack is non-empty at end (unmatched opens).

---

## 6. Linked List

### When to Use
- Problem explicitly involves a linked list.
- Need to reverse, merge, detect cycles, or find middle.
- Often combined with **slow/fast pointers** (Floyd's algorithm).

### How to Use
1. Use a **dummy node** before the head to simplify edge cases.
2. Use two pointers (prev/curr) for reversal.
3. Use slow/fast for cycle detection and middle-finding.

```python
# Template: Reverse
prev, curr = None, head
while curr:
    next_temp = curr.next
    curr.next = prev
    prev = curr
    curr = next_temp
return prev

# Template: Floyd's Cycle Detection
slow = fast = head
while fast and fast.next:
    slow = slow.next
    fast = fast.next.next
    if slow == fast:
        return True  # cycle
return False

# Template: Merge with dummy
dummy = curr = ListNode()
while l1 and l2:
    if l1.val < l2.val:
        curr.next = l1
        l1 = l1.next
    else:
        curr.next = l2
        l2 = l2.next
    curr = curr.next
curr.next = l1 or l2
```

### Common Pitfalls
- Forgetting the dummy node (forces special handling for head).
- Not saving `curr.next` before reassigning (loses rest of list).

---

## 7. Trees (DFS / BFS)

### When to Use
- Problem involves a tree or tree-shaped structure (organizational charts, file systems).
- Need depth, paths, level-order traversal, ancestors.
- DFS: explore/validate; BFS: shortest path / level-by-level.

### How to Use
1. **DFS** for depth, path tracking, validation (recursive or explicit stack).
2. **BFS** for level-order / shortest distance (queue).
3. Always handle the `None` base case first.

```python
# Template: DFS preorder
def dfs(node):
    if not node:
        return
    # process node (preorder)
    dfs(node.left)
    # process node (inorder)
    dfs(node.right)
    # process node (postorder)

# Template: BFS level order
queue = deque([root])
while queue:
    level_size = len(queue)
    for _ in range(level_size):
        node = queue.popleft()
        # process node
        if node.left:
            queue.append(node.left)
        if node.right:
            queue.append(node.right)

# Template: Path sum / depth
def max_depth(root):
    if not root:
        return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))
```

### Common Pitfalls
- Forgetting base case → infinite recursion.
- Modifying the tree while traversing (use return values, not mutation).

---

## 8. Heap / Priority Queue

### When to Use
- Need **top k** or **bottom k** elements.
- Need repeated access to the **min/max** element.
- Merging k sorted sequences.
- Any problem where you keep popping the "best" element.

### How to Use
1. Python `heapq` is a **min-heap**. For max-heap, negate values.
2. Push all candidates, pop until you have what you need.
3. For top-k, use a **min-heap of size k** — keep popping the smallest.

```python
import heapq

# Template: Top k largest (min-heap of size k)
heap = []
for num in nums:
    heapq.heappush(heap, num)
    if len(heap) > k:
        heapq.heappop(heap)
# heap contains k largest elements

# Template: Max-heap via negation
heap = []
for num in nums:
    heapq.heappush(heap, -num)
max_val = -heapq.heappop(heap)

# Template: K closest points
return heapq.nsmallest(k, points, key=lambda p: p[0]**2 + p[1]**2)
```

### Common Pitfalls
- Using `heappop` on an empty heap → IndexError.
- Forgetting that tuples compare lexicographically (first element breaks ties).

---

## 9. Graph (DFS / BFS / Topological Sort / Union Find)

### When to Use
- Grid / matrix problems ("islands", "flood fill").
- Dependencies between nodes ("course schedule", "build order").
- Connectivity ("are these in the same group").
- Shortest path in unweighted graph → BFS.

### How to Use
1. **DFS** for exploring connected components, reachability.
2. **BFS** for shortest path in unweighted graphs.
3. **Topological sort** (Kahn's algorithm) when order matters — count in-degrees, peel off zeros.
4. **Union Find** for dynamic connectivity / grouping.

```python
# Template: Grid DFS
def dfs(r, c):
    if out_of_bounds(r, c) or grid[r][c] == "0":
        return
    grid[r][c] = "0"  # mark visited
    for dr, dc in [(1,0), (-1,0), (0,1), (0,-1)]:
        dfs(r + dr, c + dc)

# Template: BFS shortest path
queue = deque([start])
visited = {start}
distance = 0
while queue:
    for _ in range(len(queue)):
        node = queue.popleft()
        if node == target:
            return distance
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    distance += 1

# Template: Topological sort (Kahn)
in_degree = [0] * n
for u in range(n):
    for v in graph[u]:
        in_degree[v] += 1
queue = deque([i for i in range(n) if in_degree[i] == 0])
order = []
while queue:
    node = queue.popleft()
    order.append(node)
    for v in graph[node]:
        in_degree[v] -= 1
        if in_degree[v] == 0:
            queue.append(v)
# if len(order) != n: cycle exists

# Template: Union Find
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # path compression
        return self.parent[x]
    def union(self, x, y):
        px, py = self.find(x), self.find(y)
        if px == py:
            return False
        if self.rank[px] < self.rank[py]:
            self.parent[px] = py
        elif self.rank[px] > self.rank[py]:
            self.parent[py] = px
        else:
            self.parent[py] = px
            self.rank[px] += 1
        return True
```

### Common Pitfalls
- Not marking visited when enqueueing (marks too late → duplicates in queue).
- Confusing adjacency list vs adjacency matrix setup.
- Forgetting path compression in Union Find (slows down to O(log n) per op).

---

## 10. Backtracking

### When to Use
- Problem asks for **all** combinations, permutations, subsets, or arrangements.
- "Generate all possible", "find any valid configuration", sudoku solver.
- Decision tree where each level picks one option.

### How to Use
1. Define the **state** (what's chosen so far).
2. At each step, try every valid option.
3. **Recurse**, then **undo** the choice (backtrack).
4. Add a copy of the state to results when it's complete.

```python
# Template: Subsets
def backtrack(start, path):
    res.append(path[:])  # add current state
    for i in range(start, len(nums)):
        path.append(nums[i])      # choose
        backtrack(i + 1, path)    # explore
        path.pop()                 # undo

# Template: Permutations
def backtrack(path, used):
    if len(path) == len(nums):
        res.append(path[:])
        return
    for i in range(len(nums)):
        if used[i]:
            continue
        used[i] = True
        path.append(nums[i])
        backtrack(path, used)
        path.pop()
        used[i] = False

# Template: Combination Sum (with reuse)
def backtrack(start, path, remaining):
    if remaining == 0:
        res.append(path[:])
        return
    if remaining < 0:
        return
    for i in range(start, len(candidates)):
        path.append(candidates[i])
        backtrack(i, path, remaining - candidates[i])  # i not i+1 (reuse)
        path.pop()
```

### Common Pitfalls
- Forgetting to **copy** the path before appending to `res` (`path[:]`).
- Forgetting to **undo** the choice (pop) after recursion.
- Not pruning early (e.g., return when remaining < 0).

---

## 11. Dynamic Programming

### When to Use
- Problem has **overlapping subproblems** and **optimal substructure**.
- "Count the number of ways", "minimum/maximum cost", "is it possible".
- Brute force recursion would recompute the same state many times.

### How to Use
1. Define the **state** clearly: what does `dp[i]` (or `dp[i][j]`) represent?
2. Write the **base case**.
3. Write the **recurrence**: how does `dp[i]` depend on earlier states?
4. Decide iteration order (smaller subproblems first).
5. Return the final answer.

```python
# Template: 1D DP (Climbing Stairs)
dp = [0] * (n + 1)
dp[0] = dp[1] = 1
for i in range(2, n + 1):
    dp[i] = dp[i - 1] + dp[i - 2]
return dp[n]

# Template: 1D DP space-optimized
prev, curr = 1, 1
for i in range(2, n + 1):
    prev, curr = curr, prev + curr
return curr

# Template: 2D DP (Longest Common Subsequence)
dp = [[0] * (n + 1) for _ in range(m + 1)]
for i in range(1, m + 1):
    for j in range(1, n + 1):
        if text1[i-1] == text2[j-1]:
            dp[i][j] = dp[i-1][j-1] + 1
        else:
            dp[i][j] = max(dp[i-1][j], dp[i][j-1])
return dp[m][n]

# Template: Unbounded knapsack (Coin Change — min coins)
dp = [float("inf")] * (amount + 1)
dp[0] = 0
for coin in coins:
    for x in range(coin, amount + 1):
        dp[x] = min(dp[x], dp[x - coin] + 1)
```

### Common Pitfalls
- Wrong state definition (the most common DP bug — revisit what `dp[i]` means).
- Off-by-one in array size (`amount + 1` not `amount`).
- Forgetting to initialize base cases before the loop.
- Using a 2D table when 1D rolling array would suffice (wastes space).

---

## 12. Greedy

### When to Use
- A **local optimal** choice always leads to the **global optimal**.
- Problem involves intervals, scheduling, or "minimum number of operations".
- You can prove swapping doesn't make things worse.

### How to Use
1. Sort by some key (start time, end time, value).
2. Make the locally best choice at each step.
3. Verify with examples that greedy actually works.

```python
# Template: Interval scheduling (max non-overlapping)
intervals.sort(key=lambda x: x[1])  # sort by end
count, end = 0, float("-inf")
for s, e in intervals:
    if s >= end:
        count += 1
        end = e
return count
```

### Common Pitfalls
- Greedy **fails** on many problems where it seems to apply — always test edge cases.
- Not sorting by the right key.

---

## Quick Pattern Recognition Cheat Sheet

| If problem says... | Use this pattern |
|--------------------|------------------|
| "sorted" + "find" | Binary Search |
| "two sum"/"pair"/"triplet" | Two Pointers or HashMap |
| "subarray"/"substring" + "max/min/longest" | Sliding Window |
| "next greater"/"balanced"/"nested" | Stack |
| "top k"/"k closest"/"kth largest" | Heap |
| "grid"/"connected"/"islands" | Graph DFS/BFS |
| "all permutations"/"all subsets" | Backtracking |
| "number of ways"/"minimum cost to reach" | Dynamic Programming |
| "schedule"/"non-overlapping intervals" | Greedy |
| "find cycle"/"reverse"/"merge" list | Linked List pointers |
| "depth"/"level order"/"ancestor" | Tree DFS/BFS |
| "are these connected"/"group"/"merge accounts" | Union Find |
| "course prerequisites"/"build order" | Topological Sort |

---

## Study Plan — 12 Week Schedule

| Week | Pattern | Practice Problems | Goal |
|------|---------|--------------------|------|
| 1 | Arrays, Strings, HashMap | 10 easy each | Master Python idioms |
| 2 | Sets, Collections, basic problems | 10 easy | Frequency counting fluency |
| 3 | Two Pointers | 6 problems | Opposite-end + slow/fast |
| 4 | Sliding Window | 5 problems | Fixed + variable window |
| 5 | Binary Search | 4 problems | Exact + first-true pattern |
| 6 | Stack | 4 problems | Monotonic stack |
| 7 | Linked List | 5 problems | Reverse/merge/cycle/middle |
| 8 | Trees (DFS/BFS) | 5 problems | Traversals + LCA |
| 9 | Heap | 3 problems | Top-k patterns |
| 10 | Graph | 4 problems | DFS/BFS/topo/union-find |
| 11 | Backtracking | 4 problems | Recursive decision tree |
| 12 | Dynamic Programming | 5 problems | 1D + 2D + knapsack |

---

## Daily Workflow (2 Hours)

| Time | Activity |
|------|----------|
| 15 min | Review yesterday's solution from memory |
| 20 min | Read this guide's pattern section |
| 10 min | Write the template by hand |
| 45 min | Solve 2–3 practice problems |
| 20 min | Read an editorial or watch a video explanation |
| 10 min | Re-solve one problem without notes |

---

## Final Tips

- **Recognize the pattern first**, then code. Spend 2–3 minutes identifying the pattern before writing anything.
- **Templates are training wheels.** After 5 problems, write from memory.
- **Debug by state inspection**: print `dp[i]`, print the stack, print the window bounds.
- **Edge cases to always check**: empty input, single element, all duplicates, negative numbers, integer overflow.
- **When stuck**: ask "what subproblem looks smaller but identical?" → that's DP or recursion.
- **Time limit**: if a O(n²) solution times out, look for a hash map or two-pointer optimization to make it O(n).