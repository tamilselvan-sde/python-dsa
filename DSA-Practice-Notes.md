# DSA Practice Notes — Python Examples

## 1. Arrays & Lists

```python
# Reverse array
def reverse_array(nums):
    left, right = 0, len(nums) - 1
    while left < right:
        nums[left], nums[right] = nums[right], nums[left]
        left += 1
        right -= 1
    return nums

# Find max/min
def find_max_min(nums):
    return max(nums), min(nums)

# Rotate array (right by k)
def rotate_array(nums, k):
    k %= len(nums)
    nums.reverse()
    nums[:k] = reversed(nums[:k])
    nums[k:] = reversed(nums[k:])
    return nums

# Remove duplicates (sorted array)
def remove_duplicates(nums):
    if not nums:
        return 0
    write = 1
    for read in range(1, len(nums)):
        if nums[read] != nums[read - 1]:
            nums[write] = nums[read]
            write += 1
    return write
```

## 2. Strings

```python
# Reverse string
def reverse_string(s):
    return s[::-1]

# Count vowels
def count_vowels(s):
    return sum(1 for c in s.lower() if c in "aeiou")

# Palindrome check
def is_palindrome(s):
    s = "".join(c.lower() for c in s if c.isalnum())
    return s == s[::-1]

# Anagram check
def is_anagram(s, t):
    return sorted(s) == sorted(t)

# Frequency count
def char_frequency(s):
    freq = {}
    for c in s:
        freq[c] = freq.get(c, 0) + 1
    return freq
```

## 3. Dictionaries & HashMap

```python
# Two Sum
def two_sum(nums, target):
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []

# Contains Duplicate
def contains_duplicate(nums):
    return len(nums) != len(set(nums))

# Character Count
def char_count(s):
    return {c: s.count(c) for c in set(s)}
```

## 4. Sets

```python
# Unique elements
def unique_elements(nums):
    return list(set(nums))

# Intersection of two arrays
def intersection(nums1, nums2):
    return list(set(nums1) & set(nums2))
```

## 5. Collections Module

```python
from collections import Counter, defaultdict, deque

# Counter
freq = Counter("hello")  # {'h': 1, 'e': 1, 'l': 2, 'o': 1}

# defaultdict
groups = defaultdict(list)
groups["a"].append(1)  # no KeyError

# deque (double-ended queue)
dq = deque([1, 2, 3])
dq.append(4)        # [1,2,3,4]
dq.appendleft(0)    # [0,1,2,3,4]
dq.pop()            # 4
dq.popleft()        # 0
```

## 6. Two Pointers

```python
# Two Sum II (sorted array)
def two_sum_sorted(numbers, target):
    left, right = 0, len(numbers) - 1
    while left < right:
        s = numbers[left] + numbers[right]
        if s == target:
            return [left + 1, right + 1]
        elif s < target:
            left += 1
        else:
            right -= 1
    return []

# Move Zeroes
def move_zeroes(nums):
    pos = 0
    for i in range(len(nums)):
        if nums[i] != 0:
            nums[pos], nums[i] = nums[i], nums[pos]
            pos += 1
    return nums

# Container With Most Water
def max_area(height):
    left, right = 0, len(height) - 1
    max_water = 0
    while left < right:
        h = min(height[left], height[right])
        max_water = max(max_water, h * (right - left))
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
    return max_water

# 3Sum
def three_sum(nums):
    nums.sort()
    res = []
    for i in range(len(nums) - 2):
        if i > 0 and nums[i] == nums[i - 1]:
            continue
        left, right = i + 1, len(nums) - 1
        while left < right:
            s = nums[i] + nums[left] + nums[right]
            if s == 0:
                res.append([nums[i], nums[left], nums[right]])
                while left < right and nums[left] == nums[left + 1]:
                    left += 1
                while left < right and nums[right] == nums[right - 1]:
                    right -= 1
                left += 1
                right -= 1
            elif s < 0:
                left += 1
            else:
                right -= 1
    return res
```

## 7. Sliding Window

```python
# Fixed window — Maximum Average Subarray
def find_max_average(nums, k):
    window_sum = sum(nums[:k])
    max_sum = window_sum
    for right in range(k, len(nums)):
        window_sum += nums[right] - nums[right - k]
        max_sum = max(max_sum, window_sum)
    return max_sum / k

# Variable window — Longest Substring Without Repeating Characters
def length_of_longest_substring(s):
    seen = {}
    left = max_len = 0
    for right, c in enumerate(s):
        if c in seen and seen[c] >= left:
            left = seen[c] + 1
        seen[c] = right
        max_len = max(max_len, right - left + 1)
    return max_len

# Minimum Window Substring
def min_window(s, t):
    if not s or not t:
        return ""
    need = Counter(t)
    have = {}
    left = 0
    formed, required = 0, len(need)
    min_len, min_start = float("inf"), 0
    for right, c in enumerate(s):
        have[c] = have.get(c, 0) + 1
        if c in need and have[c] == need[c]:
            formed += 1
        while formed == required:
            if right - left + 1 < min_len:
                min_len = right - left + 1
                min_start = left
            left_char = s[left]
            have[left_char] -= 1
            if left_char in need and have[left_char] < need[left_char]:
                formed -= 1
            left += 1
    return s[min_start:min_start + min_len] if min_len != float("inf") else ""
```

## 8. Binary Search

```python
# Binary Search
def binary_search(nums, target):
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

# Search Insert Position
def search_insert(nums, target):
    left, right = 0, len(nums)
    while left < right:
        mid = (left + right) // 2
        if nums[mid] >= target:
            right = mid
        else:
            left = mid + 1
    return left

# First Bad Version
def first_bad_version(n, is_bad_version):
    left, right = 1, n
    while left < right:
        mid = (left + right) // 2
        if is_bad_version(mid):
            right = mid
        else:
            left = mid + 1
    return left

# Search in Rotated Sorted Array
def search_rotated(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = (left + right) // 2
        if nums[mid] == target:
            return mid
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        else:
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1
    return -1
```

## 9. Stack

```python
# Valid Parentheses
def is_valid_parentheses(s):
    pairs = {")": "(", "}": "{", "]": "["}
    stack = []
    for c in s:
        if c in pairs:
            if not stack or stack.pop() != pairs[c]:
                return False
        else:
            stack.append(c)
    return not stack

# Min Stack
class MinStack:
    def __init__(self):
        self.stack = []
        self.min_stack = []

    def push(self, val):
        self.stack.append(val)
        if not self.min_stack or val <= self.min_stack[-1]:
            self.min_stack.append(val)

    def pop(self):
        if self.stack.pop() == self.min_stack[-1]:
            self.min_stack.pop()

    def top(self):
        return self.stack[-1]

    def get_min(self):
        return self.min_stack[-1]

# Daily Temperatures
def daily_temperatures(temps):
    res = [0] * len(temps)
    stack = []
    for i, t in enumerate(temps):
        while stack and t > temps[stack[-1]]:
            prev = stack.pop()
            res[prev] = i - prev
        stack.append(i)
    return res

# Next Greater Element
def next_greater_element(nums1, nums2):
    stack = []
    nge = {}
    for num in nums2:
        while stack and num > stack[-1]:
            nge[stack.pop()] = num
        stack.append(num)
    return [nge.get(n, -1) for n in nums1]
```

## 10. Linked List

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

# Reverse Linked List
def reverse_list(head):
    prev = None
    curr = head
    while curr:
        next_temp = curr.next
        curr.next = prev
        prev = curr
        curr = next_temp
    return prev

# Merge Two Sorted Lists
def merge_two_lists(l1, l2):
    dummy = ListNode()
    curr = dummy
    while l1 and l2:
        if l1.val < l2.val:
            curr.next = l1
            l1 = l1.next
        else:
            curr.next = l2
            l2 = l2.next
        curr = curr.next
    curr.next = l1 or l2
    return dummy.next

# Detect Cycle
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False

# Middle of Linked List
def middle_node(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow
```

## 11. Trees

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

# Maximum Depth
def max_depth(root):
    if not root:
        return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))

# Same Tree
def is_same_tree(p, q):
    if not p and not q:
        return True
    if not p or not q or p.val != q.val:
        return False
    return is_same_tree(p.left, q.left) and is_same_tree(p.right, q.right)

# Invert Tree
def invert_tree(root):
    if not root:
        return None
    root.left, root.right = root.right, root.left
    invert_tree(root.left)
    invert_tree(root.right)
    return root

# Level Order Traversal
def level_order(root):
    if not root:
        return []
    res, queue = [], deque([root])
    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        res.append(level)
    return res

# Lowest Common Ancestor (BST)
def lowest_common_ancestor(root, p, q):
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left
        elif p.val > root.val and q.val > root.val:
            root = root.right
        else:
            return root

# Lowest Common Ancestor (Binary Tree)
def lca_bt(root, p, q):
    if not root or root == p or root == q:
        return root
    left = lca_bt(root.left, p, q)
    right = lca_bt(root.right, p, q)
    if left and right:
        return root
    return left or right
```

## 12. Heap

```python
import heapq

# Top K Frequent Elements
def top_k_frequent(nums, k):
    freq = Counter(nums)
    return [num for num, _ in heapq.nlargest(k, freq.items(), key=lambda x: x[1])]

# Alternative with min-heap
def top_k_frequent_heap(nums, k):
    freq = Counter(nums)
    heap = []
    for num, count in freq.items():
        heapq.heappush(heap, (count, num))
        if len(heap) > k:
            heapq.heappop(heap)
    return [num for _, num in heap]

# K Closest Points to Origin
def k_closest(points, k):
    return heapq.nsmallest(k, points, key=lambda p: p[0]**2 + p[1]**2)

# Merge K Sorted Lists
def merge_k_lists(lists):
    heap = []
    for i, node in enumerate(lists):
        if node:
            heapq.heappush(heap, (node.val, i, node))
    dummy = curr = ListNode()
    while heap:
        val, i, node = heapq.heappop(heap)
        curr.next = ListNode(val)
        curr = curr.next
        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))
    return dummy.next
```

## 13. Graph

```python
from collections import deque

# DFS (adjacency list)
def dfs(graph, start, visited=None):
    if visited is None:
        visited = set()
    visited.add(start)
    for neighbor in graph[start]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
    return visited

# BFS (adjacency list)
def bfs(graph, start):
    visited = {start}
    queue = deque([start])
    while queue:
        node = queue.popleft()
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return visited

# Number of Islands
def num_islands(grid):
    def dfs(r, c):
        if r < 0 or r >= len(grid) or c < 0 or c >= len(grid[0]) or grid[r][c] == "0":
            return
        grid[r][c] = "0"
        for dr, dc in [(1,0), (-1,0), (0,1), (0,-1)]:
            dfs(r + dr, c + dc)
    count = 0
    for r in range(len(grid)):
        for c in range(len(grid[0])):
            if grid[r][c] == "1":
                count += 1
                dfs(r, c)
    return count

# Clone Graph
def clone_graph(node):
    if not node:
        return None
    clones = {}
    def dfs(n):
        if n in clones:
            return clones[n]
        copy = Node(n.val)
        clones[n] = copy
        for neighbor in n.neighbors:
            copy.neighbors.append(dfs(neighbor))
        return copy
    return dfs(node)

# Course Schedule (Topological Sort / Cycle Detection)
def can_finish(num_courses, prerequisites):
    graph = [[] for _ in range(num_courses)]
    in_degree = [0] * num_courses
    for course, prereq in prerequisites:
        graph[prereq].append(course)
        in_degree[course] += 1
    queue = deque([i for i in range(num_courses) if in_degree[i] == 0])
    count = 0
    while queue:
        node = queue.popleft()
        count += 1
        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)
    return count == num_courses

# Union Find
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
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

## 14. Backtracking

```python
# Subsets
def subsets(nums):
    res = []
    def backtrack(start, path):
        res.append(path[:])
        for i in range(start, len(nums)):
            path.append(nums[i])
            backtrack(i + 1, path)
            path.pop()
    backtrack(0, [])
    return res

# Permutations
def permute(nums):
    res = []
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
    backtrack([], [False] * len(nums))
    return res

# Combination Sum
def combination_sum(candidates, target):
    res = []
    def backtrack(start, path, total):
        if total == target:
            res.append(path[:])
            return
        if total > target:
            return
        for i in range(start, len(candidates)):
            path.append(candidates[i])
            backtrack(i, path, total + candidates[i])
            path.pop()
    backtrack(0, [], 0)
    return res

# Word Search
def exist(board, word):
    def dfs(r, c, i):
        if i == len(word):
            return True
        if r < 0 or r >= len(board) or c < 0 or c >= len(board[0]) or board[r][c] != word[i]:
            return False
        temp, board[r][c] = board[r][c], "#"
        found = (dfs(r+1, c, i+1) or dfs(r-1, c, i+1) or
                 dfs(r, c+1, i+1) or dfs(r, c-1, i+1))
        board[r][c] = temp
        return found
    for r in range(len(board)):
        for c in range(len(board[0])):
            if dfs(r, c, 0):
                return True
    return False
```

## 15. Dynamic Programming

```python
# Climbing Stairs
def climb_stairs(n):
    if n <= 2:
        return n
    prev, curr = 1, 2
    for _ in range(3, n + 1):
        prev, curr = curr, prev + curr
    return curr

# House Robber
def rob(nums):
    prev = curr = 0
    for num in nums:
        prev, curr = curr, max(curr, prev + num)
    return curr

# Coin Change (minimum coins)
def coin_change(coins, amount):
    dp = [float("inf")] * (amount + 1)
    dp[0] = 0
    for coin in coins:
        for x in range(coin, amount + 1):
            dp[x] = min(dp[x], dp[x - coin] + 1)
    return dp[amount] if dp[amount] != float("inf") else -1

# Longest Increasing Subsequence
def length_of_lis(nums):
    tails = []
    for num in nums:
        i = 0
        j = len(tails)
        while i < j:
            mid = (i + j) // 2
            if tails[mid] < num:
                i = mid + 1
            else:
                j = mid
        if i == len(tails):
            tails.append(num)
        else:
            tails[i] = num
    return len(tails)

# Longest Common Subsequence
def longest_common_subsequence(text1, text2):
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i - 1] == text2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])
    return dp[m][n]
```

## Quick Reference — Time Complexities

| Pattern | Time | Space |
|---------|------|-------|
| Two Pointers | O(n) | O(1) |
| Sliding Window | O(n) | O(k) or O(1) |
| Binary Search | O(log n) | O(1) |
| Stack | O(n) | O(n) |
| Linked List ops | O(n) | O(1) |
| Tree traversal | O(n) | O(h) |
| Heap (push/pop) | O(log n) | O(n) |
| Graph DFS/BFS | O(V+E) | O(V) |
| Backtracking | O(2^n) or O(n!) | O(n) |
| DP | O(n) to O(n²) | O(n) to O(n²) |
