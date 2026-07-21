# Trees

## Overview

Trees are hierarchical data structures with a root node and child nodes. Binary trees are the most common in interviews. Recursive solutions dominate tree problems due to their naturally recursive structure.

```mermaid
mindmap
  root((Trees))
    Binary Tree
      Traversal: inorder, preorder, postorder
      Level order (BFS)
      Depth first (DFS)
    Binary Search Tree
      Search: O(log n) average
      Insert & Delete
      Validate BST
    Tree Properties
      Height & Depth
      Diameter
      Max path sum
      LCA (Lowest Common Ancestor)
    Tree Construction
      Serialization / Deserialization
      Build from traversals
```

## When to Use

- Hierarchical data representation
- Binary search tree operations
- Problems involving levels, ancestors, descendants
- Recursive divide and conquer
- Expression trees, decision trees

## How to Identify

- Input is a tree (TreeNode) or can be represented as tree
- "Binary tree", "BST", "binary search tree" mentioned
- "Level order", "inorder", "preorder" traversal
- "LCA", "diameter", "path sum"
- Problems with hierarchical relationships

## Template/Skeleton

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

# DFS Traversals (Recursive)
def inorder(root):
    return inorder(root.left) + [root.val] + inorder(root.right) if root else []

def preorder(root):
    return [root.val] + preorder(root.left) + preorder(root.right) if root else []

def postorder(root):
    return postorder(root.left) + postorder(root.right) + [root.val] if root else []

# BFS Traversal (Iterative)
from collections import deque
def level_order(root):
    if not root:
        return []
    result = []
    queue = deque([root])
    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)
    return result

# Validate BST
def is_valid_bst(root, lo=float('-inf'), hi=float('inf')):
    if not root:
        return True
    if not (lo < root.val < hi):
        return False
    return is_valid_bst(root.left, lo, root.val) and \
           is_valid_bst(root.right, root.val, hi)
```

## ASCII Diagram: Binary Tree

```
       1
      / \
     2   3
    / \ / \
   4  5 6  7

Inorder:   4 → 2 → 5 → 1 → 6 → 3 → 7
Preorder:  1 → 2 → 4 → 5 → 3 → 6 → 7
Postorder: 4 → 5 → 2 → 6 → 7 → 3 → 1

BST Example:
       8
      / \
     3   10
    / \    \
   1   6    14
      / \   /
     4  7  13

          8
         / \
        3   10
       / \    \
      1   6    14
         / \   /
        4  7  13

Inorder traversal of BST is sorted:
1, 3, 4, 6, 7, 8, 10, 13, 14

Tree Terminology:
   - Root = topmost node (8)
   - Leaf = node with no children (1, 4, 7, 13)
   - Height = longest path from root to leaf
   - Depth = distance from root
```

## Common Problems

### Problem 1: Maximum Depth of Binary Tree

- **Problem:** Find maximum depth (height) of binary tree.
- **Approach:** Recursively compute max depth of left and right.
- **Python Solution:**
  ```python
  def max_depth(root):
      if not root:
          return 0
      return 1 + max(max_depth(root.left), max_depth(root.right))
  ```
- **Complexity:** O(n) time, O(h) space where h is height

### Problem 2: Validate Binary Search Tree

- **Problem:** Determine if tree is a valid BST.
- **Approach:** Recursive range checking with min/max bounds.
- **Python Solution:**
  ```python
  def is_valid_bst(root, lo=float('-inf'), hi=float('inf')):
      if not root:
          return True
      if not (lo < root.val < hi):
          return False
      return is_valid_bst(root.left, lo, root.val) and \
             is_valid_bst(root.right, root.val, hi)
  ```
- **Complexity:** O(n) time, O(h) space

### Problem 3: Lowest Common Ancestor of BST

- **Problem:** Find LCA of two nodes in BST.
- **Approach:** Compare node values — LCA is first node between p and q values.
- **Python Solution:**
  ```python
  def lowest_common_ancestor(root, p, q):
      while root:
          if root.val > p.val and root.val > q.val:
              root = root.left
          elif root.val < p.val and root.val < q.val:
              root = root.right
          else:
              return root
      return None
  ```
- **Complexity:** O(h) time, O(1) space

### Problem 4: Binary Tree Level Order Traversal

- **Problem:** Return level-by-level traversal.
- **Approach:** BFS with queue, process level by level.
- **Python Solution:**
  ```python
  def level_order(root):
      if not root:
          return []
      result = []
      queue = deque([root])
      while queue:
          level = []
          for _ in range(len(queue)):
              node = queue.popleft()
              level.append(node.val)
              if node.left:
                  queue.append(node.left)
              if node.right:
                  queue.append(node.right)
          result.append(level)
      return result
  ```
- **Complexity:** O(n) time, O(n) space

### Problem 5: Diameter of Binary Tree

- **Problem:** Find longest path between any two nodes.
- **Approach:** DFS computes height, tracks max diameter seen.
- **Python Solution:**
  ```python
  def diameter_of_binary_tree(root):
      diameter = 0
      def height(node):
          nonlocal diameter
          if not node:
              return 0
          left = height(node.left)
          right = height(node.right)
          diameter = max(diameter, left + right)
          return 1 + max(left, right)
      height(root)
      return diameter
  ```
- **Complexity:** O(n) time, O(h) space

### Problem 6: Serialize and Deserialize Binary Tree

- **Problem:** Convert tree to string and back.
- **Approach:** Preorder traversal with '#' for null markers.
- **Python Solution:**
  ```python
  class Codec:
      def serialize(self, root):
          def dfs(node):
              if not node:
                  return ['#']
              return [str(node.val)] + dfs(node.left) + dfs(node.right)
          return ','.join(dfs(root))

      def deserialize(self, data):
          def dfs():
              val = next(vals)
              if val == '#':
                  return None
              node = TreeNode(int(val))
              node.left = dfs()
              node.right = dfs()
              return node
          vals = iter(data.split(','))
          return dfs()
  ```
- **Complexity:** O(n) time, O(n) space

## Complexity Analysis Table

| Problem | Time | Space | Difficulty |
|---------|------|-------|-----------|
| Maximum Depth | O(n) | O(h) | Easy |
| Validate BST | O(n) | O(h) | Medium |
| LCA of BST | O(h) | O(1) | Medium |
| Level Order | O(n) | O(n) | Medium |
| Diameter | O(n) | O(h) | Easy |
| Serialize/Deserialize | O(n) | O(n) | Hard |

## Quick Notes

- Recursive solutions are natural for trees — think about base case (null node) first
- BFS (level order) uses queue, DFS uses stack (or recursion)
- Inorder traversal of BST gives sorted values
- For problems involving "longest path" in trees, DFS tracking height + diameter works
- BST properties: left < root < right (for all nodes, not just immediate children)
- Serialization patterns: preorder with null markers is most common

## Common Mistakes

- Not checking for null/None before accessing node.left or node.right
- Confusing inorder/preorder/postorder traversal order
- Forgetting that BST validation requires min/max bounds, not just comparing with children
- Using global variables in recursive functions without proper scoping (use nonlocal or closure)
- Not handling the case where tree is empty (root is None)
- Incorrectly computing diameter: it's left height + right height, not left diameter + right diameter

## Remember

- Trees are recursive by nature — most solutions are recursive
- If a tree problem seems hard, think about traversal order (inorder, preorder, postorder)
- BFS is for level-related problems, DFS is for path/height/comparison problems
- The recursive height function is a building block for many tree problems
- BST problems reduce to comparing values — no need for parent pointers
- Always draw the tree — tree problems are visual problems

---
Author: Tamilselvan S
LinkedIn: https://www.linkedin.com/in/tamilselvan-ai/
GitHub: `your-github-username`
---
