# Trees and Binary Search Trees

## Overview

Trees are hierarchical data structures that appear in 30-40% of interviews. As a Lead engineer, you should be fluent in tree traversals, BST operations, and be able to solve complex tree problems efficiently.

---

## Key Concepts

### Terminology

```
        1          <- Root (depth 0, height 2)
       / \
      2   3        <- Internal nodes (depth 1)
     / \   \
    4   5   6      <- Leaves (depth 2)

Height: Longest path from root to leaf
Depth: Distance from root to node
Level: Depth + 1
```

### Types of Trees

| Type | Definition |
|------|------------|
| Binary Tree | Each node has at most 2 children |
| BST | Left < Root < Right for all nodes |
| Complete Binary Tree | All levels filled except possibly last (filled left to right) |
| Full Binary Tree | Every node has 0 or 2 children |
| Perfect Binary Tree | All leaves at same level, all internal nodes have 2 children |
| Balanced Tree | Height difference of subtrees ≤ 1 |

### Properties

```
For a tree with n nodes:
- Min height: floor(log₂(n))
- Max height: n - 1 (degenerate/skewed)

For perfect binary tree with height h:
- Number of nodes: 2^(h+1) - 1
- Number of leaves: 2^h

For complete binary tree (array representation):
- Parent of node i: (i-1) // 2
- Left child of node i: 2*i + 1
- Right child of node i: 2*i + 2
```

---

## Node Implementation

**Python:**
```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

class NaryNode:
    def __init__(self, val=0, children=None):
        self.val = val
        self.children = children if children else []
```

**JavaScript:**
```javascript
class TreeNode {
  constructor(val = 0, left = null, right = null) {
    this.val = val;
    this.left = left;
    this.right = right;
  }
}

class NaryNode {
  constructor(val = 0, children = []) {
    this.val = val;
    this.children = children;
  }
}
```

---

## Tree Traversals

### DFS Traversals

**Python - Recursive:**
```python
# Recursive
def preorder(root):
    """Root -> Left -> Right"""
    if not root:
        return []
    return [root.val] + preorder(root.left) + preorder(root.right)

def inorder(root):
    """Left -> Root -> Right (BST: sorted order)"""
    if not root:
        return []
    return inorder(root.left) + [root.val] + inorder(root.right)

def postorder(root):
    """Left -> Right -> Root"""
    if not root:
        return []
    return postorder(root.left) + postorder(root.right) + [root.val]
```

**JavaScript - Recursive:**
```javascript
// Recursive
function preorder(root) {
  // Root -> Left -> Right
  if (!root) return [];
  return [root.val, ...preorder(root.left), ...preorder(root.right)];
}

function inorder(root) {
  // Left -> Root -> Right (BST: sorted order)
  if (!root) return [];
  return [...inorder(root.left), root.val, ...inorder(root.right)];
}

function postorder(root) {
  // Left -> Right -> Root
  if (!root) return [];
  return [...postorder(root.left), ...postorder(root.right), root.val];
}
```

**Python - Iterative (More important for interviews):**
```python
# Iterative (More important for interviews)
def preorder_iterative(root):
    if not root:
        return []
    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.val)
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)

    return result

def inorder_iterative(root):
    result = []
    stack = []
    curr = root

    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        result.append(curr.val)
        curr = curr.right

    return result

def postorder_iterative(root):
    if not root:
        return []
    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.val)
        if node.left:
            stack.append(node.left)
        if node.right:
            stack.append(node.right)

    return result[::-1]  # Reverse at end
```

**JavaScript - Iterative:**
```javascript
function preorderIterative(root) {
  if (!root) return [];
  const result = [];
  const stack = [root];

  while (stack.length > 0) {
    const node = stack.pop();
    result.push(node.val);
    if (node.right) stack.push(node.right);
    if (node.left) stack.push(node.left);
  }

  return result;
}

function inorderIterative(root) {
  const result = [];
  const stack = [];
  let curr = root;

  while (curr || stack.length > 0) {
    while (curr) {
      stack.push(curr);
      curr = curr.left;
    }
    curr = stack.pop();
    result.push(curr.val);
    curr = curr.right;
  }

  return result;
}

function postorderIterative(root) {
  if (!root) return [];
  const result = [];
  const stack = [root];

  while (stack.length > 0) {
    const node = stack.pop();
    result.push(node.val);
    if (node.left) stack.push(node.left);
    if (node.right) stack.push(node.right);
  }

  return result.reverse(); // Reverse at end
}
```

### BFS Traversal (Level Order)

**Python:**
```python
from collections import deque

def level_order(root):
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level = []
        level_size = len(queue)

        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(level)

    return result

def zigzag_level_order(root):
    if not root:
        return []

    result = []
    queue = deque([root])
    left_to_right = True

    while queue:
        level = deque()
        for _ in range(len(queue)):
            node = queue.popleft()
            if left_to_right:
                level.append(node.val)
            else:
                level.appendleft(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(list(level))
        left_to_right = not left_to_right

    return result
```

**JavaScript:**
```javascript
function levelOrder(root) {
  if (!root) return [];

  const result = [];
  const queue = [root];

  while (queue.length > 0) {
    const level = [];
    const levelSize = queue.length;

    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift();
      level.push(node.val);
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }

    result.push(level);
  }

  return result;
}

function zigzagLevelOrder(root) {
  if (!root) return [];

  const result = [];
  const queue = [root];
  let leftToRight = true;

  while (queue.length > 0) {
    const level = [];
    const levelSize = queue.length;

    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift();
      if (leftToRight) {
        level.push(node.val);
      } else {
        level.unshift(node.val);
      }
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }

    result.push(level);
    leftToRight = !leftToRight;
  }

  return result;
}
```

---

## Tree Properties

### Height and Depth

**Python:**
```python
def max_depth(root):
    """Height of tree"""
    if not root:
        return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))

def min_depth(root):
    """Minimum depth to any leaf"""
    if not root:
        return 0
    if not root.left:
        return 1 + min_depth(root.right)
    if not root.right:
        return 1 + min_depth(root.left)
    return 1 + min(min_depth(root.left), min_depth(root.right))
```

**JavaScript:**
```javascript
function maxDepth(root) {
  // Height of tree
  if (!root) return 0;
  return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}

function minDepth(root) {
  // Minimum depth to any leaf
  if (!root) return 0;
  if (!root.left) return 1 + minDepth(root.right);
  if (!root.right) return 1 + minDepth(root.left);
  return 1 + Math.min(minDepth(root.left), minDepth(root.right));
}
```

### Balanced Check

**Python:**
```python
def is_balanced(root):
    """Check if height-balanced (heights differ by at most 1)"""
    def check(node):
        if not node:
            return 0

        left = check(node.left)
        if left == -1:
            return -1

        right = check(node.right)
        if right == -1:
            return -1

        if abs(left - right) > 1:
            return -1

        return 1 + max(left, right)

    return check(root) != -1
```

**JavaScript:**
```javascript
function isBalanced(root) {
  // Check if height-balanced (heights differ by at most 1)
  function check(node) {
    if (!node) return 0;

    const left = check(node.left);
    if (left === -1) return -1;

    const right = check(node.right);
    if (right === -1) return -1;

    if (Math.abs(left - right) > 1) return -1;

    return 1 + Math.max(left, right);
  }

  return check(root) !== -1;
}
```

### Symmetric/Mirror

**Python:**
```python
def is_symmetric(root):
    def is_mirror(t1, t2):
        if not t1 and not t2:
            return True
        if not t1 or not t2:
            return False
        return (t1.val == t2.val and
                is_mirror(t1.left, t2.right) and
                is_mirror(t1.right, t2.left))

    return is_mirror(root, root)
```

**JavaScript:**
```javascript
function isSymmetric(root) {
  function isMirror(t1, t2) {
    if (!t1 && !t2) return true;
    if (!t1 || !t2) return false;
    return t1.val === t2.val &&
           isMirror(t1.left, t2.right) &&
           isMirror(t1.right, t2.left);
  }

  return isMirror(root, root);
}
```

### Same Tree

**Python:**
```python
def is_same_tree(p, q):
    if not p and not q:
        return True
    if not p or not q:
        return False
    return (p.val == q.val and
            is_same_tree(p.left, q.left) and
            is_same_tree(p.right, q.right))
```

**JavaScript:**
```javascript
function isSameTree(p, q) {
  if (!p && !q) return true;
  if (!p || !q) return false;
  return p.val === q.val &&
         isSameTree(p.left, q.left) &&
         isSameTree(p.right, q.right);
}
```

---

## Path Problems

### Path Sum

**Python:**
```python
def has_path_sum(root, target_sum):
    """Check if root-to-leaf path exists with given sum"""
    if not root:
        return False
    if not root.left and not root.right:  # Leaf
        return root.val == target_sum
    return (has_path_sum(root.left, target_sum - root.val) or
            has_path_sum(root.right, target_sum - root.val))

def path_sum_all_paths(root, target_sum):
    """Return all root-to-leaf paths with given sum"""
    result = []

    def dfs(node, remaining, path):
        if not node:
            return
        path.append(node.val)

        if not node.left and not node.right and remaining == node.val:
            result.append(path[:])
        else:
            dfs(node.left, remaining - node.val, path)
            dfs(node.right, remaining - node.val, path)

        path.pop()

    dfs(root, target_sum, [])
    return result

def path_sum_iii(root, target_sum):
    """Count paths (any start/end) with given sum"""
    def count_paths(node, prefix_sum, target, prefix_sums):
        if not node:
            return 0

        prefix_sum += node.val
        count = prefix_sums.get(prefix_sum - target, 0)

        prefix_sums[prefix_sum] = prefix_sums.get(prefix_sum, 0) + 1

        count += count_paths(node.left, prefix_sum, target, prefix_sums)
        count += count_paths(node.right, prefix_sum, target, prefix_sums)

        prefix_sums[prefix_sum] -= 1

        return count

    return count_paths(root, 0, target_sum, {0: 1})
```

**JavaScript:**
```javascript
function hasPathSum(root, targetSum) {
  // Check if root-to-leaf path exists with given sum
  if (!root) return false;
  if (!root.left && !root.right) { // Leaf
    return root.val === targetSum;
  }
  return hasPathSum(root.left, targetSum - root.val) ||
         hasPathSum(root.right, targetSum - root.val);
}

function pathSumAllPaths(root, targetSum) {
  // Return all root-to-leaf paths with given sum
  const result = [];

  function dfs(node, remaining, path) {
    if (!node) return;
    path.push(node.val);

    if (!node.left && !node.right && remaining === node.val) {
      result.push([...path]);
    } else {
      dfs(node.left, remaining - node.val, path);
      dfs(node.right, remaining - node.val, path);
    }

    path.pop();
  }

  dfs(root, targetSum, []);
  return result;
}

function pathSumIII(root, targetSum) {
  // Count paths (any start/end) with given sum
  function countPaths(node, prefixSum, target, prefixSums) {
    if (!node) return 0;

    prefixSum += node.val;
    let count = prefixSums.get(prefixSum - target) ?? 0;

    prefixSums.set(prefixSum, (prefixSums.get(prefixSum) ?? 0) + 1);

    count += countPaths(node.left, prefixSum, target, prefixSums);
    count += countPaths(node.right, prefixSum, target, prefixSums);

    prefixSums.set(prefixSum, prefixSums.get(prefixSum) - 1);

    return count;
  }

  return countPaths(root, 0, targetSum, new Map([[0, 1]]));
}
```

### Maximum Path Sum

**Python:**
```python
def max_path_sum(root):
    """Max sum path between any two nodes"""
    max_sum = float('-inf')

    def dfs(node):
        nonlocal max_sum
        if not node:
            return 0

        left = max(0, dfs(node.left))   # Ignore negative paths
        right = max(0, dfs(node.right))

        # Path through current node
        max_sum = max(max_sum, left + node.val + right)

        # Return max single path (for parent)
        return node.val + max(left, right)

    dfs(root)
    return max_sum
```

**JavaScript:**
```javascript
function maxPathSum(root) {
  // Max sum path between any two nodes
  let maxSum = -Infinity;

  function dfs(node) {
    if (!node) return 0;

    const left = Math.max(0, dfs(node.left));   // Ignore negative paths
    const right = Math.max(0, dfs(node.right));

    // Path through current node
    maxSum = Math.max(maxSum, left + node.val + right);

    // Return max single path (for parent)
    return node.val + Math.max(left, right);
  }

  dfs(root);
  return maxSum;
}
```

### Binary Tree Diameter

**Python:**
```python
def diameter_of_binary_tree(root):
    """Longest path between any two nodes (in edges)"""
    diameter = 0

    def depth(node):
        nonlocal diameter
        if not node:
            return 0

        left = depth(node.left)
        right = depth(node.right)

        diameter = max(diameter, left + right)

        return 1 + max(left, right)

    depth(root)
    return diameter
```

**JavaScript:**
```javascript
function diameterOfBinaryTree(root) {
  // Longest path between any two nodes (in edges)
  let diameter = 0;

  function depth(node) {
    if (!node) return 0;

    const left = depth(node.left);
    const right = depth(node.right);

    diameter = Math.max(diameter, left + right);

    return 1 + Math.max(left, right);
  }

  depth(root);
  return diameter;
}
```

---

## Lowest Common Ancestor (LCA)

**Python:**
```python
def lowest_common_ancestor(root, p, q):
    """LCA in binary tree"""
    if not root or root == p or root == q:
        return root

    left = lowest_common_ancestor(root.left, p, q)
    right = lowest_common_ancestor(root.right, p, q)

    if left and right:
        return root
    return left or right

def lca_bst(root, p, q):
    """LCA in BST - more efficient"""
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left
        elif p.val > root.val and q.val > root.val:
            root = root.right
        else:
            return root
    return None
```

**JavaScript:**
```javascript
function lowestCommonAncestor(root, p, q) {
  // LCA in binary tree
  if (!root || root === p || root === q) return root;

  const left = lowestCommonAncestor(root.left, p, q);
  const right = lowestCommonAncestor(root.right, p, q);

  if (left && right) return root;
  return left || right;
}

function lcaBST(root, p, q) {
  // LCA in BST - more efficient
  while (root) {
    if (p.val < root.val && q.val < root.val) {
      root = root.left;
    } else if (p.val > root.val && q.val > root.val) {
      root = root.right;
    } else {
      return root;
    }
  }
  return null;
}
```

---

## BST Operations

### Search

**Python:**
```python
def search_bst(root, val):
    if not root or root.val == val:
        return root
    if val < root.val:
        return search_bst(root.left, val)
    return search_bst(root.right, val)

def search_bst_iterative(root, val):
    while root and root.val != val:
        root = root.left if val < root.val else root.right
    return root
```

**JavaScript:**
```javascript
function searchBST(root, val) {
  if (!root || root.val === val) return root;
  if (val < root.val) return searchBST(root.left, val);
  return searchBST(root.right, val);
}

function searchBSTIterative(root, val) {
  while (root && root.val !== val) {
    root = val < root.val ? root.left : root.right;
  }
  return root;
}
```

### Insert

**Python:**
```python
def insert_into_bst(root, val):
    if not root:
        return TreeNode(val)
    if val < root.val:
        root.left = insert_into_bst(root.left, val)
    else:
        root.right = insert_into_bst(root.right, val)
    return root
```

**JavaScript:**
```javascript
function insertIntoBST(root, val) {
  if (!root) return new TreeNode(val);
  if (val < root.val) {
    root.left = insertIntoBST(root.left, val);
  } else {
    root.right = insertIntoBST(root.right, val);
  }
  return root;
}
```

### Delete

**Python:**
```python
def delete_node(root, key):
    if not root:
        return None

    if key < root.val:
        root.left = delete_node(root.left, key)
    elif key > root.val:
        root.right = delete_node(root.right, key)
    else:
        # Node to delete found
        if not root.left:
            return root.right
        if not root.right:
            return root.left

        # Two children: replace with inorder successor
        successor = root.right
        while successor.left:
            successor = successor.left
        root.val = successor.val
        root.right = delete_node(root.right, successor.val)

    return root
```

**JavaScript:**
```javascript
function deleteNode(root, key) {
  if (!root) return null;

  if (key < root.val) {
    root.left = deleteNode(root.left, key);
  } else if (key > root.val) {
    root.right = deleteNode(root.right, key);
  } else {
    // Node to delete found
    if (!root.left) return root.right;
    if (!root.right) return root.left;

    // Two children: replace with inorder successor
    let successor = root.right;
    while (successor.left) {
      successor = successor.left;
    }
    root.val = successor.val;
    root.right = deleteNode(root.right, successor.val);
  }

  return root;
}
```

### Validate BST

**Python:**
```python
def is_valid_bst(root):
    def validate(node, min_val, max_val):
        if not node:
            return True
        if node.val <= min_val or node.val >= max_val:
            return False
        return (validate(node.left, min_val, node.val) and
                validate(node.right, node.val, max_val))

    return validate(root, float('-inf'), float('inf'))
```

**JavaScript:**
```javascript
function isValidBST(root) {
  function validate(node, minVal, maxVal) {
    if (!node) return true;
    if (node.val <= minVal || node.val >= maxVal) return false;
    return validate(node.left, minVal, node.val) &&
           validate(node.right, node.val, maxVal);
  }

  return validate(root, -Infinity, Infinity);
}
```

### Kth Smallest/Largest

**Python:**
```python
def kth_smallest(root, k):
    """Inorder traversal gives sorted order"""
    stack = []
    curr = root

    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        k -= 1
        if k == 0:
            return curr.val
        curr = curr.right

    return -1
```

**JavaScript:**
```javascript
function kthSmallest(root, k) {
  // Inorder traversal gives sorted order
  const stack = [];
  let curr = root;

  while (curr || stack.length > 0) {
    while (curr) {
      stack.push(curr);
      curr = curr.left;
    }
    curr = stack.pop();
    k--;
    if (k === 0) return curr.val;
    curr = curr.right;
  }

  return -1;
}
```

---

## Tree Construction

### From Preorder and Inorder

**Python:**
```python
def build_tree_pre_in(preorder, inorder):
    if not preorder or not inorder:
        return None

    inorder_map = {val: i for i, val in enumerate(inorder)}

    def build(pre_start, pre_end, in_start, in_end):
        if pre_start > pre_end:
            return None

        root_val = preorder[pre_start]
        root = TreeNode(root_val)

        root_idx = inorder_map[root_val]
        left_size = root_idx - in_start

        root.left = build(pre_start + 1, pre_start + left_size,
                         in_start, root_idx - 1)
        root.right = build(pre_start + left_size + 1, pre_end,
                          root_idx + 1, in_end)

        return root

    return build(0, len(preorder) - 1, 0, len(inorder) - 1)
```

**JavaScript:**
```javascript
function buildTreePreIn(preorder, inorder) {
  if (!preorder.length || !inorder.length) return null;

  const inorderMap = new Map();
  inorder.forEach((val, i) => inorderMap.set(val, i));

  function build(preStart, preEnd, inStart, inEnd) {
    if (preStart > preEnd) return null;

    const rootVal = preorder[preStart];
    const root = new TreeNode(rootVal);

    const rootIdx = inorderMap.get(rootVal);
    const leftSize = rootIdx - inStart;

    root.left = build(preStart + 1, preStart + leftSize, inStart, rootIdx - 1);
    root.right = build(preStart + leftSize + 1, preEnd, rootIdx + 1, inEnd);

    return root;
  }

  return build(0, preorder.length - 1, 0, inorder.length - 1);
}
```

### From Inorder and Postorder

**Python:**
```python
def build_tree_in_post(inorder, postorder):
    inorder_map = {val: i for i, val in enumerate(inorder)}

    def build(in_start, in_end, post_start, post_end):
        if in_start > in_end:
            return None

        root_val = postorder[post_end]
        root = TreeNode(root_val)

        root_idx = inorder_map[root_val]
        left_size = root_idx - in_start

        root.left = build(in_start, root_idx - 1,
                         post_start, post_start + left_size - 1)
        root.right = build(root_idx + 1, in_end,
                          post_start + left_size, post_end - 1)

        return root

    return build(0, len(inorder) - 1, 0, len(postorder) - 1)
```

**JavaScript:**
```javascript
function buildTreeInPost(inorder, postorder) {
  const inorderMap = new Map();
  inorder.forEach((val, i) => inorderMap.set(val, i));

  function build(inStart, inEnd, postStart, postEnd) {
    if (inStart > inEnd) return null;

    const rootVal = postorder[postEnd];
    const root = new TreeNode(rootVal);

    const rootIdx = inorderMap.get(rootVal);
    const leftSize = rootIdx - inStart;

    root.left = build(inStart, rootIdx - 1, postStart, postStart + leftSize - 1);
    root.right = build(rootIdx + 1, inEnd, postStart + leftSize, postEnd - 1);

    return root;
  }

  return build(0, inorder.length - 1, 0, postorder.length - 1);
}
```

### Serialize and Deserialize

**Python:**
```python
class Codec:
    def serialize(self, root):
        """Encodes a tree to a single string."""
        if not root:
            return "null"

        result = []
        queue = deque([root])

        while queue:
            node = queue.popleft()
            if node:
                result.append(str(node.val))
                queue.append(node.left)
                queue.append(node.right)
            else:
                result.append("null")

        # Remove trailing nulls
        while result and result[-1] == "null":
            result.pop()

        return ",".join(result)

    def deserialize(self, data):
        """Decodes your encoded data to tree."""
        if data == "null":
            return None

        values = data.split(",")
        root = TreeNode(int(values[0]))
        queue = deque([root])
        i = 1

        while queue and i < len(values):
            node = queue.popleft()

            if i < len(values) and values[i] != "null":
                node.left = TreeNode(int(values[i]))
                queue.append(node.left)
            i += 1

            if i < len(values) and values[i] != "null":
                node.right = TreeNode(int(values[i]))
                queue.append(node.right)
            i += 1

        return root
```

**JavaScript:**
```javascript
class Codec {
  serialize(root) {
    // Encodes a tree to a single string
    if (!root) return "null";

    const result = [];
    const queue = [root];

    while (queue.length > 0) {
      const node = queue.shift();
      if (node) {
        result.push(String(node.val));
        queue.push(node.left);
        queue.push(node.right);
      } else {
        result.push("null");
      }
    }

    // Remove trailing nulls
    while (result.length > 0 && result[result.length - 1] === "null") {
      result.pop();
    }

    return result.join(",");
  }

  deserialize(data) {
    // Decodes your encoded data to tree
    if (data === "null") return null;

    const values = data.split(",");
    const root = new TreeNode(parseInt(values[0]));
    const queue = [root];
    let i = 1;

    while (queue.length > 0 && i < values.length) {
      const node = queue.shift();

      if (i < values.length && values[i] !== "null") {
        node.left = new TreeNode(parseInt(values[i]));
        queue.push(node.left);
      }
      i++;

      if (i < values.length && values[i] !== "null") {
        node.right = new TreeNode(parseInt(values[i]));
        queue.push(node.right);
      }
      i++;
    }

    return root;
  }
}
```

---

## Advanced Tree Problems

### Flatten Binary Tree to Linked List

**Python:**
```python
def flatten(root):
    """Flatten to linked list (preorder) in-place"""
    curr = root
    while curr:
        if curr.left:
            # Find rightmost of left subtree
            rightmost = curr.left
            while rightmost.right:
                rightmost = rightmost.right

            # Rewire connections
            rightmost.right = curr.right
            curr.right = curr.left
            curr.left = None

        curr = curr.right
```

**JavaScript:**
```javascript
function flatten(root) {
  // Flatten to linked list (preorder) in-place
  let curr = root;
  while (curr) {
    if (curr.left) {
      // Find rightmost of left subtree
      let rightmost = curr.left;
      while (rightmost.right) {
        rightmost = rightmost.right;
      }

      // Rewire connections
      rightmost.right = curr.right;
      curr.right = curr.left;
      curr.left = null;
    }

    curr = curr.right;
  }
}
```

### Vertical Order Traversal

**Python:**
```python
from collections import defaultdict

def vertical_order(root):
    if not root:
        return []

    column_table = defaultdict(list)
    queue = deque([(root, 0)])  # (node, column)

    while queue:
        node, col = queue.popleft()
        column_table[col].append(node.val)

        if node.left:
            queue.append((node.left, col - 1))
        if node.right:
            queue.append((node.right, col + 1))

    return [column_table[col] for col in sorted(column_table.keys())]
```

**JavaScript:**
```javascript
function verticalOrder(root) {
  if (!root) return [];

  const columnTable = new Map();
  const queue = [[root, 0]]; // [node, column]

  while (queue.length > 0) {
    const [node, col] = queue.shift();

    if (!columnTable.has(col)) {
      columnTable.set(col, []);
    }
    columnTable.get(col).push(node.val);

    if (node.left) queue.push([node.left, col - 1]);
    if (node.right) queue.push([node.right, col + 1]);
  }

  const sortedCols = [...columnTable.keys()].sort((a, b) => a - b);
  return sortedCols.map(col => columnTable.get(col));
}
```

### Right Side View

**Python:**
```python
def right_side_view(root):
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)
        for i in range(level_size):
            node = queue.popleft()
            if i == level_size - 1:  # Rightmost node
                result.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

    return result
```

**JavaScript:**
```javascript
function rightSideView(root) {
  if (!root) return [];

  const result = [];
  const queue = [root];

  while (queue.length > 0) {
    const levelSize = queue.length;
    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift();
      if (i === levelSize - 1) { // Rightmost node
        result.push(node.val);
      }
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
  }

  return result;
}
```

---

## Must-Know Problems

### Easy
1. Maximum Depth of Binary Tree
2. Invert Binary Tree
3. Same Tree
4. Symmetric Tree
5. Path Sum
6. Subtree of Another Tree

### Medium
1. Binary Tree Level Order Traversal
2. Validate Binary Search Tree
3. Kth Smallest Element in BST
4. LCA of Binary Tree
5. Construct Binary Tree from Preorder and Inorder
6. Binary Tree Right Side View
7. Flatten Binary Tree to Linked List
8. Count Good Nodes in Binary Tree
9. Binary Tree Zigzag Level Order

### Hard
1. Binary Tree Maximum Path Sum
2. Serialize and Deserialize Binary Tree
3. Binary Tree Cameras
4. Vertical Order Traversal
5. Recover BST (swap two nodes)

---

## Practice Checklist

- [ ] Can implement all traversals iteratively
- [ ] Know when to use BFS vs DFS
- [ ] Can validate BST in one pass
- [ ] Understand LCA algorithm and its applications
- [ ] Can serialize/deserialize trees
- [ ] Can construct trees from traversals
