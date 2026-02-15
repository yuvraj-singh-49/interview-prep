# Essential Coding Patterns

## Overview

Recognizing patterns is the key to solving coding problems efficiently. This guide covers the 15 most important patterns that appear in 90%+ of interview problems.

---

## Pattern Recognition Framework

### When Approaching a Problem:

1. **Read carefully** - Understand inputs, outputs, constraints
2. **Identify keywords** - Trigger pattern recognition
3. **Consider constraints** - Guide algorithm choice
4. **Verify with examples** - Test your understanding

### Constraint-Based Algorithm Selection

| Input Size (n) | Max Complexity | Typical Algorithms |
|----------------|----------------|-------------------|
| n ≤ 10 | O(n!) | Brute force, backtracking |
| n ≤ 20 | O(2^n) | Backtracking with pruning |
| n ≤ 500 | O(n³) | DP, Floyd-Warshall |
| n ≤ 10,000 | O(n²) | DP, nested loops |
| n ≤ 1,000,000 | O(n log n) | Sorting, binary search, heap |
| n ≤ 100,000,000 | O(n) | Hash table, two pointers |
| n > 100,000,000 | O(log n) | Binary search, math |

---

## The 15 Essential Patterns

### 1. Sliding Window

**Keywords:** Subarray, substring, contiguous, window, maximum/minimum sum

**When to use:**
- Finding subarrays with specific properties
- String matching problems
- Maximum/minimum in contiguous sequence

**Templates:**

```python
# Fixed-size window
def fixed_window(arr, k):
    window_sum = sum(arr[:k])
    result = window_sum

    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i-k]
        result = max(result, window_sum)

    return result

# Variable-size window
def variable_window(arr, target):
    left = 0
    window_sum = 0
    result = float('inf')

    for right in range(len(arr)):
        window_sum += arr[right]

        while window_sum >= target:
            result = min(result, right - left + 1)
            window_sum -= arr[left]
            left += 1

    return result if result != float('inf') else 0
```

**Classic Problems:**
- Maximum Sum Subarray of Size K
- Longest Substring Without Repeating Characters
- Minimum Window Substring
- Longest Repeating Character Replacement

---

### 2. Two Pointers

**Keywords:** Sorted array, pair, triplet, in-place, opposite ends

**When to use:**
- Sorted array with target sum
- Removing duplicates in-place
- Comparing elements from both ends

**Templates:**

```python
# Opposite direction (sorted array)
def two_sum_sorted(arr, target):
    left, right = 0, len(arr) - 1

    while left < right:
        curr_sum = arr[left] + arr[right]
        if curr_sum == target:
            return [left, right]
        elif curr_sum < target:
            left += 1
        else:
            right -= 1

    return []

# Same direction (fast/slow)
def remove_duplicates(arr):
    slow = 0
    for fast in range(len(arr)):
        if arr[fast] != arr[slow]:
            slow += 1
            arr[slow] = arr[fast]
    return slow + 1
```

**Classic Problems:**
- Two Sum II (sorted)
- 3Sum, 4Sum
- Container With Most Water
- Remove Duplicates from Sorted Array
- Trapping Rain Water

---

### 3. Fast and Slow Pointers

**Keywords:** Cycle, middle, nth from end, linked list

**When to use:**
- Cycle detection
- Finding middle element
- Finding start of cycle
- Nth element from end

**Templates:**

```python
def find_cycle(head):
    slow = fast = head

    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

        if slow == fast:
            return True  # Cycle exists

    return False

def find_middle(head):
    slow = fast = head

    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

    return slow  # Middle node
```

**Classic Problems:**
- Linked List Cycle I & II
- Middle of Linked List
- Happy Number
- Palindrome Linked List

---

### 4. Merge Intervals

**Keywords:** Overlapping, intervals, meetings, schedule

**When to use:**
- Merging overlapping intervals
- Finding gaps
- Scheduling problems

**Template:**

```python
def merge_intervals(intervals):
    intervals.sort(key=lambda x: x[0])
    merged = []

    for interval in intervals:
        if not merged or merged[-1][1] < interval[0]:
            merged.append(interval)
        else:
            merged[-1][1] = max(merged[-1][1], interval[1])

    return merged
```

**Classic Problems:**
- Merge Intervals
- Insert Interval
- Meeting Rooms I & II
- Non-overlapping Intervals

---

### 5. Cyclic Sort

**Keywords:** Numbers 1 to n, missing, duplicate, range [0, n]

**When to use:**
- Array with numbers in range [1, n] or [0, n-1]
- Finding missing or duplicate numbers

**Template:**

```python
def cyclic_sort(nums):
    i = 0
    while i < len(nums):
        correct_idx = nums[i] - 1  # For 1-indexed

        if nums[i] != nums[correct_idx]:
            nums[i], nums[correct_idx] = nums[correct_idx], nums[i]
        else:
            i += 1

    return nums

def find_missing(nums):
    cyclic_sort(nums)
    for i, num in enumerate(nums):
        if num != i + 1:
            return i + 1
    return len(nums) + 1
```

**Classic Problems:**
- Find Missing Number
- Find All Missing Numbers
- Find Duplicate Number
- First Missing Positive

---

### 6. In-place Linked List Reversal

**Keywords:** Reverse, k-group, swap pairs

**When to use:**
- Reversing entire or part of linked list
- Reversing in groups

**Template:**

```python
def reverse_list(head):
    prev = None
    curr = head

    while curr:
        next_temp = curr.next
        curr.next = prev
        prev = curr
        curr = next_temp

    return prev

def reverse_between(head, left, right):
    dummy = ListNode(0, head)
    prev = dummy

    for _ in range(left - 1):
        prev = prev.next

    curr = prev.next
    for _ in range(right - left):
        next_node = curr.next
        curr.next = next_node.next
        next_node.next = prev.next
        prev.next = next_node

    return dummy.next
```

**Classic Problems:**
- Reverse Linked List
- Reverse Linked List II
- Reverse Nodes in k-Group
- Swap Nodes in Pairs

---

### 7. Tree BFS (Level Order)

**Keywords:** Level by level, breadth-first, layer, minimum depth

**When to use:**
- Level-order traversal
- Finding minimum depth/path
- Zigzag traversal
- Connecting level nodes

**Template:**

```python
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
```

**Classic Problems:**
- Binary Tree Level Order Traversal
- Zigzag Level Order
- Minimum Depth of Binary Tree
- Connect Level Order Siblings

---

### 8. Tree DFS

**Keywords:** Root to leaf, path sum, depth-first, recursion

**When to use:**
- Path-related problems
- Sum along paths
- Checking tree properties

**Template:**

```python
def dfs_template(root):
    def dfs(node, state):
        if not node:
            return  # Base case

        # Process current node

        # Recurse
        dfs(node.left, new_state)
        dfs(node.right, new_state)

        # Backtrack if needed

    dfs(root, initial_state)
```

**Classic Problems:**
- Path Sum I, II, III
- Maximum Path Sum
- Diameter of Binary Tree
- Validate BST

---

### 9. Two Heaps

**Keywords:** Median, balance, largest/smallest halves

**When to use:**
- Finding median in stream
- Balancing two groups
- Scheduling with priorities

**Template:**

```python
import heapq

class MedianFinder:
    def __init__(self):
        self.small = []  # Max heap
        self.large = []  # Min heap

    def add_num(self, num):
        heapq.heappush(self.small, -num)

        # Balance
        if self.small and self.large and -self.small[0] > self.large[0]:
            heapq.heappush(self.large, -heapq.heappop(self.small))

        # Size balance
        if len(self.small) > len(self.large) + 1:
            heapq.heappush(self.large, -heapq.heappop(self.small))
        elif len(self.large) > len(self.small):
            heapq.heappush(self.small, -heapq.heappop(self.large))

    def find_median(self):
        if len(self.small) > len(self.large):
            return -self.small[0]
        return (-self.small[0] + self.large[0]) / 2
```

**Classic Problems:**
- Find Median from Data Stream
- Sliding Window Median
- IPO

---

### 10. Subsets / Backtracking

**Keywords:** All combinations, subsets, permutations, generate all

**When to use:**
- Generating all possibilities
- Combinatorial problems
- Permutation/combination

**Template:**

```python
def backtrack(result, path, choices, start):
    result.append(path[:])  # Add current subset

    for i in range(start, len(choices)):
        path.append(choices[i])
        backtrack(result, path, choices, i + 1)
        path.pop()

def subsets(nums):
    result = []
    backtrack(result, [], nums, 0)
    return result
```

**Classic Problems:**
- Subsets I & II
- Permutations I & II
- Combination Sum I, II, III
- Letter Combinations of Phone Number

---

### 11. Modified Binary Search

**Keywords:** Sorted, rotated, search, find position

**When to use:**
- Searching in sorted/rotated array
- Finding boundaries
- Search on answer space

**Template:**

```python
def binary_search_on_answer(nums):
    left, right = min_possible, max_possible

    while left < right:
        mid = (left + right) // 2

        if condition(mid):
            right = mid  # or left = mid + 1
        else:
            left = mid + 1  # or right = mid

    return left
```

**Classic Problems:**
- Search in Rotated Sorted Array
- Find Minimum in Rotated Sorted Array
- Search a 2D Matrix
- Koko Eating Bananas
- Capacity to Ship Packages

---

### 12. Top K Elements

**Keywords:** K largest, K smallest, K most frequent, K closest

**When to use:**
- Finding K extreme elements
- Priority-based selection

**Template:**

```python
import heapq

def top_k_frequent(nums, k):
    from collections import Counter

    count = Counter(nums)
    return heapq.nlargest(k, count.keys(), key=count.get)

def k_closest_points(points, k):
    return heapq.nsmallest(k, points,
                           key=lambda p: p[0]**2 + p[1]**2)
```

**Classic Problems:**
- Top K Frequent Elements
- Kth Largest Element
- K Closest Points to Origin
- Reorganize String

---

### 13. K-way Merge

**Keywords:** K sorted, merge, smallest among K

**When to use:**
- Merging K sorted arrays/lists
- Finding smallest among multiple sources

**Template:**

```python
import heapq

def merge_k_lists(lists):
    heap = []

    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, (lst[0], i, 0))

    result = []
    while heap:
        val, list_idx, elem_idx = heapq.heappop(heap)
        result.append(val)

        if elem_idx + 1 < len(lists[list_idx]):
            next_val = lists[list_idx][elem_idx + 1]
            heapq.heappush(heap, (next_val, list_idx, elem_idx + 1))

    return result
```

**Classic Problems:**
- Merge K Sorted Lists
- Kth Smallest in Sorted Matrix
- Find K Pairs with Smallest Sums
- Smallest Range Covering K Lists

---

### 14. Topological Sort

**Keywords:** Dependencies, prerequisites, ordering, DAG

**When to use:**
- Task scheduling with dependencies
- Course prerequisites
- Build order

**Template:**

```python
from collections import deque

def topological_sort(n, edges):
    graph = [[] for _ in range(n)]
    in_degree = [0] * n

    for u, v in edges:
        graph[u].append(v)
        in_degree[v] += 1

    queue = deque([i for i in range(n) if in_degree[i] == 0])
    order = []

    while queue:
        node = queue.popleft()
        order.append(node)

        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    return order if len(order) == n else []  # Empty if cycle
```

**Classic Problems:**
- Course Schedule I & II
- Alien Dictionary
- Minimum Height Trees
- Sequence Reconstruction

---

### 15. Union Find

**Keywords:** Connected, groups, components, union, disjoint

**When to use:**
- Finding connected components
- Detecting cycles in undirected graph
- Grouping elements

**Template:**

```python
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
            px, py = py, px
        self.parent[py] = px
        if self.rank[px] == self.rank[py]:
            self.rank[px] += 1

        return True
```

**Classic Problems:**
- Number of Islands (alternative)
- Number of Provinces
- Redundant Connection
- Accounts Merge
- Longest Consecutive Sequence

---

## Pattern Recognition Cheat Sheet

| If you see... | Think... |
|---------------|----------|
| Contiguous subarray/substring | Sliding Window |
| Sorted array, find pair | Two Pointers |
| Linked list, cycle/middle | Fast & Slow Pointers |
| Overlapping ranges | Merge Intervals |
| Numbers 1 to n, missing | Cyclic Sort |
| Reverse linked list | In-place Reversal |
| Tree level-by-level | BFS |
| Tree path problems | DFS |
| Find median/balance | Two Heaps |
| All combinations | Backtracking |
| Sorted + search | Binary Search |
| K largest/smallest | Heap |
| K sorted sources | K-way Merge |
| Dependencies/ordering | Topological Sort |
| Connected components | Union Find |

---

## Practice Strategy

1. **Learn one pattern at a time**
2. **Solve 3-5 problems per pattern**
3. **Identify pattern before coding**
4. **Time yourself (25-35 min for medium)**
5. **Review and refine solutions**
