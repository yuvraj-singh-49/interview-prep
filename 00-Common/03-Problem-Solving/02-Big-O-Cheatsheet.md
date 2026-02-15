# Big-O Complexity Cheat Sheet

## Overview

Understanding time and space complexity is essential for choosing the right algorithm and communicating trade-offs during interviews.

---

## Time Complexity Overview

### Growth Rates (Slowest to Fastest)

| Complexity | Name | Example |
|------------|------|---------|
| O(1) | Constant | Array access, hash lookup |
| O(log n) | Logarithmic | Binary search |
| O(n) | Linear | Linear search, single loop |
| O(n log n) | Linearithmic | Merge sort, heap sort |
| O(n²) | Quadratic | Nested loops, bubble sort |
| O(n³) | Cubic | 3 nested loops, Floyd-Warshall |
| O(2ⁿ) | Exponential | Subsets, recursive fibonacci |
| O(n!) | Factorial | Permutations |

### Visual Comparison

```
For n = 1000:

O(1)        = 1
O(log n)    = ~10
O(n)        = 1,000
O(n log n)  = ~10,000
O(n²)       = 1,000,000
O(n³)       = 1,000,000,000
O(2ⁿ)       = 10^301 (impossible)
```

---

## Common Operations Complexity

### Arrays

| Operation | Time | Notes |
|-----------|------|-------|
| Access by index | O(1) | |
| Search (unsorted) | O(n) | |
| Search (sorted) | O(log n) | Binary search |
| Insert at end | O(1) amortized | O(n) worst case |
| Insert at beginning | O(n) | Shift all elements |
| Delete at end | O(1) | |
| Delete at beginning | O(n) | Shift all elements |
| Delete at index | O(n) | |

### Linked Lists

| Operation | Singly | Doubly |
|-----------|--------|--------|
| Access | O(n) | O(n) |
| Search | O(n) | O(n) |
| Insert at head | O(1) | O(1) |
| Insert at tail | O(n) or O(1)* | O(1) |
| Delete at head | O(1) | O(1) |
| Delete at tail | O(n) | O(1) |
| Delete given node | O(n) | O(1) |

*O(1) if tail pointer maintained

### Hash Tables

| Operation | Average | Worst |
|-----------|---------|-------|
| Search | O(1) | O(n) |
| Insert | O(1) | O(n) |
| Delete | O(1) | O(n) |
| Space | O(n) | O(n) |

### Binary Search Tree

| Operation | Average | Worst (unbalanced) |
|-----------|---------|-------------------|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |

### Balanced BST (AVL, Red-Black)

| Operation | Time |
|-----------|------|
| Search | O(log n) |
| Insert | O(log n) |
| Delete | O(log n) |

### Heap

| Operation | Time |
|-----------|------|
| Find min/max | O(1) |
| Insert | O(log n) |
| Delete min/max | O(log n) |
| Build heap | O(n) |

### Stack & Queue

| Operation | Time |
|-----------|------|
| Push/Enqueue | O(1) |
| Pop/Dequeue | O(1) |
| Peek | O(1) |

---

## Sorting Algorithms

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|------|---------|-------|-------|--------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes |
| Radix Sort | O(d(n+k)) | O(d(n+k)) | O(d(n+k)) | O(n+k) | Yes |

---

## Graph Algorithms

| Algorithm | Time | Space |
|-----------|------|-------|
| BFS | O(V + E) | O(V) |
| DFS | O(V + E) | O(V) |
| Dijkstra (binary heap) | O((V + E) log V) | O(V) |
| Bellman-Ford | O(V × E) | O(V) |
| Floyd-Warshall | O(V³) | O(V²) |
| Topological Sort | O(V + E) | O(V) |
| Prim's MST | O((V + E) log V) | O(V) |
| Kruskal's MST | O(E log E) | O(V) |
| Union-Find (optimized) | O(α(n)) ≈ O(1) | O(n) |

---

## Common Algorithm Patterns

### Recursion

```python
# Without memoization: O(2^n)
def fib(n):
    if n <= 1:
        return n
    return fib(n-1) + fib(n-2)

# With memoization: O(n)
def fib_memo(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib_memo(n-1, memo) + fib_memo(n-2, memo)
    return memo[n]
```

### Nested Loops

```python
# O(n²)
for i in range(n):
    for j in range(n):
        # O(1) operation

# O(n²) even though j starts at i
for i in range(n):
    for j in range(i, n):
        # Still O(n²), specifically n(n+1)/2

# O(n log n)
for i in range(n):
    j = 1
    while j < n:
        j *= 2  # log n iterations
```

### Binary Search

```python
# O(log n)
def binary_search(arr, target):
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = (left + right) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1
```

---

## Space Complexity

### What Counts

1. **Input space** - Usually not counted
2. **Auxiliary space** - Additional space used
3. **Total space** - Input + Auxiliary

### Common Space Patterns

| Pattern | Space |
|---------|-------|
| Fixed variables | O(1) |
| Array of size n | O(n) |
| 2D matrix n×m | O(n×m) |
| Recursion depth | O(depth) |
| Hash set/map of n items | O(n) |

### Examples

```python
# O(1) space
def sum_array(arr):
    total = 0
    for num in arr:
        total += num
    return total

# O(n) space
def copy_array(arr):
    return arr[:]

# O(n) space (recursion stack)
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)

# O(1) space (tail recursion concept, though Python doesn't optimize)
def factorial_iter(n):
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

---

## Amortized Analysis

Some operations are occasionally expensive but average out:

### Dynamic Array (ArrayList)

```
Insert at end:
- Usually O(1)
- Occasionally O(n) when resizing
- Amortized: O(1)
```

### Hash Table

```
Insert:
- Usually O(1)
- Occasionally O(n) when rehashing
- Amortized: O(1)
```

---

## Interview Tips

### Analyzing Your Solution

1. **Count loops**: Each nested loop usually multiplies
2. **Check recursion**: Draw recursion tree
3. **Consider input variations**: Best, average, worst case
4. **Don't forget space**: Stack frames, data structures

### Common Mistakes

| Mistake | Reality |
|---------|---------|
| "It's O(1) because hash lookup" | Could be O(n) worst case |
| "Recursion is always O(n)" | Depends on branching factor |
| "Sort is O(n)" | Comparison sort is O(n log n) min |
| "Binary search on unsorted" | Must sort first: O(n log n) |

### Optimization Hints

| Current | Optimization | How |
|---------|--------------|-----|
| O(n²) | O(n log n) | Sorting + binary search |
| O(n²) | O(n) | Hash table |
| O(n) multiple passes | O(n) single pass | Combine operations |
| O(n) space | O(1) space | In-place algorithms |

---

## Quick Reference Card

```
Instant lookup?           → O(1) - Hash table
Sorted + search?          → O(log n) - Binary search
Linear scan?              → O(n) - Single loop
Nested loops?             → O(n²) - Two loops
Divide & conquer?         → O(n log n) - Merge sort
All subsets?              → O(2ⁿ) - Backtracking
All permutations?         → O(n!) - Backtracking

Space:
- Variables only          → O(1)
- New array/list          → O(n)
- 2D matrix               → O(n²)
- Recursion               → O(depth)
```

---

## Practice Problems by Complexity

### O(1)
- Get minimum from stack
- Check if number is power of 2

### O(log n)
- Binary search
- Find peak element
- Search in rotated array

### O(n)
- Two Sum (with hash)
- Maximum subarray
- Linked list cycle

### O(n log n)
- Merge sort
- Meeting rooms
- Top K frequent

### O(n²)
- Longest palindromic substring
- DP problems (many)
- 3Sum

### O(2ⁿ)
- Subsets
- Combination sum

### O(n!)
- Permutations
- N-Queens
