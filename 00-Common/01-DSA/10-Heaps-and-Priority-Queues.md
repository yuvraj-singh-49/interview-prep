# Heaps and Priority Queues

## Overview

Heaps are essential for problems involving finding min/max elements, scheduling, and streaming data. As a Lead engineer, you should understand heap operations and when to apply them.

---

## Key Concepts

### Heap Properties

```
Min Heap: Parent <= Children (root is minimum)
Max Heap: Parent >= Children (root is maximum)

Complete Binary Tree Structure:
- All levels filled except possibly last
- Last level filled left to right

Array Representation (0-indexed):
- Parent of i: (i - 1) // 2
- Left child of i: 2 * i + 1
- Right child of i: 2 * i + 2
```

### Time Complexity

| Operation | Time |
|-----------|------|
| Insert (push) | O(log n) |
| Extract min/max | O(log n) |
| Peek min/max | O(1) |
| Build heap | O(n) |
| Heapify | O(log n) |
| Search | O(n) |

---

## Python heapq Module

```python
import heapq

# Python only has min-heap

# Create heap from list - O(n)
nums = [3, 1, 4, 1, 5, 9]
heapq.heapify(nums)

# Push - O(log n)
heapq.heappush(nums, 2)

# Pop minimum - O(log n)
min_val = heapq.heappop(nums)

# Peek minimum - O(1)
min_val = nums[0]

# Push and pop in one operation
val = heapq.heappushpop(nums, 6)  # Push then pop
val = heapq.heapreplace(nums, 6)  # Pop then push

# Get n smallest/largest - O(n log k)
smallest = heapq.nsmallest(3, nums)
largest = heapq.nlargest(3, nums)
```

### Max Heap Trick

```python
# Negate values for max-heap behavior
max_heap = []
heapq.heappush(max_heap, -val)
max_val = -heapq.heappop(max_heap)

# With tuples (priority, value)
heapq.heappush(max_heap, (-priority, value))
```

### Custom Comparators

```python
# Using tuples
heap = []
heapq.heappush(heap, (priority, index, item))

# Using wrapper class
class Item:
    def __init__(self, val, priority):
        self.val = val
        self.priority = priority

    def __lt__(self, other):
        return self.priority < other.priority

heap = []
heapq.heappush(heap, Item("task", 5))
```

---

## Heap Implementation

**Python:**
```python
class MinHeap:
    def __init__(self):
        self.heap = []

    def parent(self, i):
        return (i - 1) // 2

    def left_child(self, i):
        return 2 * i + 1

    def right_child(self, i):
        return 2 * i + 2

    def swap(self, i, j):
        self.heap[i], self.heap[j] = self.heap[j], self.heap[i]

    def push(self, val):
        self.heap.append(val)
        self._bubble_up(len(self.heap) - 1)

    def _bubble_up(self, i):
        while i > 0 and self.heap[self.parent(i)] > self.heap[i]:
            self.swap(i, self.parent(i))
            i = self.parent(i)

    def pop(self):
        if not self.heap:
            return None
        if len(self.heap) == 1:
            return self.heap.pop()

        min_val = self.heap[0]
        self.heap[0] = self.heap.pop()
        self._bubble_down(0)
        return min_val

    def _bubble_down(self, i):
        min_idx = i
        left = self.left_child(i)
        right = self.right_child(i)

        if left < len(self.heap) and self.heap[left] < self.heap[min_idx]:
            min_idx = left
        if right < len(self.heap) and self.heap[right] < self.heap[min_idx]:
            min_idx = right

        if min_idx != i:
            self.swap(i, min_idx)
            self._bubble_down(min_idx)

    def peek(self):
        return self.heap[0] if self.heap else None

    def size(self):
        return len(self.heap)

    @staticmethod
    def heapify(arr):
        """Build heap in O(n)"""
        heap = MinHeap()
        heap.heap = arr[:]
        # Start from last non-leaf node
        for i in range(len(arr) // 2 - 1, -1, -1):
            heap._bubble_down(i)
        return heap
```

**JavaScript:**
```javascript
class MinHeap {
  constructor() {
    this.heap = [];
  }

  parent(i) { return Math.floor((i - 1) / 2); }
  leftChild(i) { return 2 * i + 1; }
  rightChild(i) { return 2 * i + 2; }
  swap(i, j) { [this.heap[i], this.heap[j]] = [this.heap[j], this.heap[i]]; }

  push(val) {
    this.heap.push(val);
    this._bubbleUp(this.heap.length - 1);
  }

  _bubbleUp(i) {
    while (i > 0 && this.heap[this.parent(i)] > this.heap[i]) {
      this.swap(i, this.parent(i));
      i = this.parent(i);
    }
  }

  pop() {
    if (this.heap.length === 0) return null;
    if (this.heap.length === 1) return this.heap.pop();

    const minVal = this.heap[0];
    this.heap[0] = this.heap.pop();
    this._bubbleDown(0);
    return minVal;
  }

  _bubbleDown(i) {
    let minIdx = i;
    const left = this.leftChild(i);
    const right = this.rightChild(i);

    if (left < this.heap.length && this.heap[left] < this.heap[minIdx]) {
      minIdx = left;
    }
    if (right < this.heap.length && this.heap[right] < this.heap[minIdx]) {
      minIdx = right;
    }

    if (minIdx !== i) {
      this.swap(i, minIdx);
      this._bubbleDown(minIdx);
    }
  }

  peek() { return this.heap.length > 0 ? this.heap[0] : null; }
  size() { return this.heap.length; }

  static heapify(arr) {
    const heap = new MinHeap();
    heap.heap = [...arr];
    // Start from last non-leaf node
    for (let i = Math.floor(arr.length / 2) - 1; i >= 0; i--) {
      heap._bubbleDown(i);
    }
    return heap;
  }
}
```

---

## Classic Heap Problems

### Kth Largest Element

**Python:**
```python
def find_kth_largest(nums, k):
    """
    Using min-heap of size k - O(n log k)
    """
    heap = nums[:k]
    heapq.heapify(heap)

    for num in nums[k:]:
        if num > heap[0]:
            heapq.heapreplace(heap, num)

    return heap[0]

def find_kth_largest_max_heap(nums, k):
    """Using max-heap - O(n + k log n)"""
    max_heap = [-x for x in nums]
    heapq.heapify(max_heap)

    for _ in range(k - 1):
        heapq.heappop(max_heap)

    return -max_heap[0]
```

**JavaScript:**
```javascript
function findKthLargest(nums, k) {
  // Using min-heap of size k - O(n log k)
  const heap = new MinHeap();

  for (const num of nums) {
    heap.push(num);
    if (heap.size() > k) {
      heap.pop();
    }
  }

  return heap.peek();
}

function findKthLargestMaxHeap(nums, k) {
  // Using max-heap - O(n + k log n)
  // Create max heap by negating values
  const heap = new MinHeap();
  for (const num of nums) {
    heap.push(-num);
  }

  for (let i = 0; i < k - 1; i++) {
    heap.pop();
  }

  return -heap.peek();
}
```

### Top K Frequent Elements

**Python:**
```python
def top_k_frequent(nums, k):
    """O(n log k) using min-heap"""
    from collections import Counter

    count = Counter(nums)
    heap = []

    for num, freq in count.items():
        heapq.heappush(heap, (freq, num))
        if len(heap) > k:
            heapq.heappop(heap)

    return [num for freq, num in heap]

def top_k_frequent_bucket(nums, k):
    """O(n) using bucket sort"""
    from collections import Counter

    count = Counter(nums)
    buckets = [[] for _ in range(len(nums) + 1)]

    for num, freq in count.items():
        buckets[freq].append(num)

    result = []
    for freq in range(len(buckets) - 1, -1, -1):
        for num in buckets[freq]:
            result.append(num)
            if len(result) == k:
                return result

    return result
```

**JavaScript:**
```javascript
function topKFrequent(nums, k) {
  // O(n) using bucket sort
  const count = new Map();
  for (const num of nums) {
    count.set(num, (count.get(num) || 0) + 1);
  }

  const buckets = Array.from({ length: nums.length + 1 }, () => []);
  for (const [num, freq] of count.entries()) {
    buckets[freq].push(num);
  }

  const result = [];
  for (let freq = buckets.length - 1; freq >= 0; freq--) {
    for (const num of buckets[freq]) {
      result.push(num);
      if (result.length === k) return result;
    }
  }

  return result;
}
```

### Merge K Sorted Lists

**Python:**
```python
def merge_k_lists(lists):
    """O(n log k) where n is total elements, k is number of lists"""
    heap = []

    # Add first element from each list
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, (lst.val, i, lst))

    dummy = ListNode(0)
    curr = dummy

    while heap:
        val, i, node = heapq.heappop(heap)
        curr.next = node
        curr = curr.next

        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))

    return dummy.next
```

**JavaScript:**
```javascript
function mergeKLists(lists) {
  // O(n log k) where n is total elements, k is number of lists
  const heap = new MinHeap();

  // Add first element from each list (store as [val, listIdx, node])
  for (let i = 0; i < lists.length; i++) {
    if (lists[i]) {
      heap.push([lists[i].val, i, lists[i]]);
    }
  }

  // Custom compare for heap - need to modify MinHeap to compare [0] index
  const dummy = new ListNode(0);
  let curr = dummy;

  while (heap.size() > 0) {
    const [val, i, node] = heap.pop();
    curr.next = node;
    curr = curr.next;

    if (node.next) {
      heap.push([node.next.val, i, node.next]);
    }
  }

  return dummy.next;
}
```

### Kth Smallest Element in Sorted Matrix

```python
def kth_smallest(matrix, k):
    """Matrix sorted row-wise and column-wise"""
    import heapq

    n = len(matrix)
    heap = [(matrix[0][0], 0, 0)]  # (value, row, col)
    visited = {(0, 0)}

    for _ in range(k):
        val, row, col = heapq.heappop(heap)

        # Add right neighbor
        if col + 1 < n and (row, col + 1) not in visited:
            heapq.heappush(heap, (matrix[row][col + 1], row, col + 1))
            visited.add((row, col + 1))

        # Add bottom neighbor
        if row + 1 < n and (row + 1, col) not in visited:
            heapq.heappush(heap, (matrix[row + 1][col], row + 1, col))
            visited.add((row + 1, col))

    return val
```

---

## Two Heaps Pattern

### Find Median from Data Stream

**Python:**
```python
class MedianFinder:
    """
    Two heaps:
    - max_heap for smaller half
    - min_heap for larger half
    """
    def __init__(self):
        self.small = []  # Max heap (negated)
        self.large = []  # Min heap

    def addNum(self, num):
        # Add to max heap
        heapq.heappush(self.small, -num)

        # Ensure max of small <= min of large
        if self.small and self.large and -self.small[0] > self.large[0]:
            heapq.heappush(self.large, -heapq.heappop(self.small))

        # Balance sizes (small can have at most 1 more)
        if len(self.small) > len(self.large) + 1:
            heapq.heappush(self.large, -heapq.heappop(self.small))
        elif len(self.large) > len(self.small):
            heapq.heappush(self.small, -heapq.heappop(self.large))

    def findMedian(self):
        if len(self.small) > len(self.large):
            return -self.small[0]
        return (-self.small[0] + self.large[0]) / 2
```

**JavaScript:**
```javascript
class MedianFinder {
  // Two heaps: max_heap for smaller half, min_heap for larger half
  constructor() {
    this.small = new MaxHeap(); // For smaller half
    this.large = new MinHeap(); // For larger half
  }

  addNum(num) {
    // Add to max heap
    this.small.push(num);

    // Ensure max of small <= min of large
    if (this.small.size() > 0 && this.large.size() > 0 &&
        this.small.peek() > this.large.peek()) {
      this.large.push(this.small.pop());
    }

    // Balance sizes (small can have at most 1 more)
    if (this.small.size() > this.large.size() + 1) {
      this.large.push(this.small.pop());
    } else if (this.large.size() > this.small.size()) {
      this.small.push(this.large.pop());
    }
  }

  findMedian() {
    if (this.small.size() > this.large.size()) {
      return this.small.peek();
    }
    return (this.small.peek() + this.large.peek()) / 2;
  }
}

// MaxHeap can be implemented by negating values or extending MinHeap
class MaxHeap extends MinHeap {
  _bubbleUp(i) {
    while (i > 0 && this.heap[this.parent(i)] < this.heap[i]) {
      this.swap(i, this.parent(i));
      i = this.parent(i);
    }
  }

  _bubbleDown(i) {
    let maxIdx = i;
    const left = this.leftChild(i);
    const right = this.rightChild(i);

    if (left < this.heap.length && this.heap[left] > this.heap[maxIdx]) {
      maxIdx = left;
    }
    if (right < this.heap.length && this.heap[right] > this.heap[maxIdx]) {
      maxIdx = right;
    }

    if (maxIdx !== i) {
      this.swap(i, maxIdx);
      this._bubbleDown(maxIdx);
    }
  }
}
```

### Sliding Window Median

```python
def median_sliding_window(nums, k):
    """O(n log k) with two heaps"""
    from sortedcontainers import SortedList

    window = SortedList()
    result = []

    for i, num in enumerate(nums):
        window.add(num)

        if len(window) > k:
            window.remove(nums[i - k])

        if len(window) == k:
            if k % 2 == 1:
                result.append(window[k // 2])
            else:
                result.append((window[k // 2 - 1] + window[k // 2]) / 2)

    return result
```

### IPO (Maximize Capital)

```python
def find_maximized_capital(k, w, profits, capital):
    """
    Select at most k projects to maximize capital
    Two heaps approach
    """
    n = len(profits)
    # Min heap: projects sorted by capital required
    projects = [(capital[i], profits[i]) for i in range(n)]
    heapq.heapify(projects)

    # Max heap: available projects sorted by profit
    available = []

    for _ in range(k):
        # Add all affordable projects to available heap
        while projects and projects[0][0] <= w:
            cap, profit = heapq.heappop(projects)
            heapq.heappush(available, -profit)

        if not available:
            break

        # Select most profitable
        w += -heapq.heappop(available)

    return w
```

---

## K-way Merge Pattern

### Find K Pairs with Smallest Sums

```python
def k_smallest_pairs(nums1, nums2, k):
    """Find k pairs with smallest sums"""
    if not nums1 or not nums2:
        return []

    heap = [(nums1[0] + nums2[0], 0, 0)]
    visited = {(0, 0)}
    result = []

    while heap and len(result) < k:
        total, i, j = heapq.heappop(heap)
        result.append([nums1[i], nums2[j]])

        if i + 1 < len(nums1) and (i + 1, j) not in visited:
            heapq.heappush(heap, (nums1[i + 1] + nums2[j], i + 1, j))
            visited.add((i + 1, j))

        if j + 1 < len(nums2) and (i, j + 1) not in visited:
            heapq.heappush(heap, (nums1[i] + nums2[j + 1], i, j + 1))
            visited.add((i, j + 1))

    return result
```

### Smallest Range Covering Elements from K Lists

```python
def smallest_range(nums):
    """
    Find smallest range that includes at least one element from each list
    """
    heap = []
    max_val = float('-inf')

    # Initialize with first element from each list
    for i, lst in enumerate(nums):
        heapq.heappush(heap, (lst[0], i, 0))
        max_val = max(max_val, lst[0])

    result = [float('-inf'), float('inf')]

    while heap:
        min_val, list_idx, elem_idx = heapq.heappop(heap)

        # Update result if current range is smaller
        if max_val - min_val < result[1] - result[0]:
            result = [min_val, max_val]

        # If any list is exhausted, we're done
        if elem_idx + 1 >= len(nums[list_idx]):
            break

        # Add next element from same list
        next_val = nums[list_idx][elem_idx + 1]
        heapq.heappush(heap, (next_val, list_idx, elem_idx + 1))
        max_val = max(max_val, next_val)

    return result
```

---

## Scheduling Problems

### Task Scheduler

**Python:**
```python
def least_interval(tasks, n):
    """
    Minimum time to complete all tasks with cooldown n
    """
    from collections import Counter

    count = Counter(tasks)
    max_heap = [-cnt for cnt in count.values()]
    heapq.heapify(max_heap)

    time = 0
    queue = []  # (available_time, remaining_count)

    while max_heap or queue:
        time += 1

        if max_heap:
            remaining = -heapq.heappop(max_heap) - 1
            if remaining > 0:
                queue.append((time + n, remaining))

        if queue and queue[0][0] == time:
            _, cnt = queue.pop(0)
            heapq.heappush(max_heap, -cnt)

    return time

def least_interval_formula(tasks, n):
    """O(n) formula approach"""
    from collections import Counter

    count = Counter(tasks)
    max_count = max(count.values())
    num_max = sum(1 for c in count.values() if c == max_count)

    # (max_count - 1) * (n + 1) + num_max
    return max(len(tasks), (max_count - 1) * (n + 1) + num_max)
```

**JavaScript:**
```javascript
function leastInterval(tasks, n) {
  // O(n) formula approach
  const count = new Map();
  for (const task of tasks) {
    count.set(task, (count.get(task) || 0) + 1);
  }

  const maxCount = Math.max(...count.values());
  let numMax = 0;
  for (const cnt of count.values()) {
    if (cnt === maxCount) numMax++;
  }

  // (maxCount - 1) * (n + 1) + numMax
  return Math.max(tasks.length, (maxCount - 1) * (n + 1) + numMax);
}
```

### Meeting Rooms II

```python
def min_meeting_rooms(intervals):
    """Minimum number of meeting rooms required"""
    if not intervals:
        return 0

    intervals.sort(key=lambda x: x[0])
    heap = []  # End times

    for start, end in intervals:
        # If earliest ending meeting is over, reuse room
        if heap and heap[0] <= start:
            heapq.heappop(heap)
        heapq.heappush(heap, end)

    return len(heap)
```

### Reorganize String

```python
def reorganize_string(s):
    """Rearrange so no two adjacent chars are same"""
    from collections import Counter

    count = Counter(s)
    max_heap = [(-cnt, char) for char, cnt in count.items()]
    heapq.heapify(max_heap)

    result = []
    prev = None

    while max_heap or prev:
        if prev and not max_heap:
            return ""  # Can't place prev

        cnt, char = heapq.heappop(max_heap)
        result.append(char)
        cnt += 1  # Used one

        if prev:
            heapq.heappush(max_heap, prev)
            prev = None

        if cnt < 0:
            prev = (cnt, char)

    return ''.join(result)
```

---

## Streaming Data

### Sliding Window Maximum

```python
from collections import deque

def max_sliding_window(nums, k):
    """O(n) using monotonic deque"""
    result = []
    dq = deque()  # Indices

    for i, num in enumerate(nums):
        # Remove indices outside window
        while dq and dq[0] <= i - k:
            dq.popleft()

        # Remove smaller elements
        while dq and nums[dq[-1]] < num:
            dq.pop()

        dq.append(i)

        if i >= k - 1:
            result.append(nums[dq[0]])

    return result
```

### Stream of Integers - Kth Largest

**Python:**
```python
class KthLargest:
    """Maintain kth largest from stream"""
    def __init__(self, k, nums):
        self.k = k
        self.heap = []

        for num in nums:
            self.add(num)

    def add(self, val):
        if len(self.heap) < self.k:
            heapq.heappush(self.heap, val)
        elif val > self.heap[0]:
            heapq.heapreplace(self.heap, val)

        return self.heap[0] if len(self.heap) == self.k else None
```

**JavaScript:**
```javascript
class KthLargest {
  // Maintain kth largest from stream
  constructor(k, nums) {
    this.k = k;
    this.heap = new MinHeap();

    for (const num of nums) {
      this.add(num);
    }
  }

  add(val) {
    if (this.heap.size() < this.k) {
      this.heap.push(val);
    } else if (val > this.heap.peek()) {
      this.heap.pop();
      this.heap.push(val);
    }

    return this.heap.size() === this.k ? this.heap.peek() : null;
  }
}
```

---

## Must-Know Problems

### Easy
1. Kth Largest Element in a Stream
2. Last Stone Weight
3. Relative Ranks

### Medium
1. Kth Largest Element in Array
2. Top K Frequent Elements
3. Sort Characters by Frequency
4. K Closest Points to Origin
5. Task Scheduler
6. Reorganize String
7. Kth Smallest Element in Sorted Matrix
8. Find K Pairs with Smallest Sums

### Hard
1. Merge K Sorted Lists
2. Find Median from Data Stream
3. Sliding Window Median
4. IPO
5. Smallest Range Covering Elements from K Lists
6. Trapping Rain Water II

---

## Practice Checklist

- [ ] Can implement heap from scratch
- [ ] Know min-heap vs max-heap usage in Python
- [ ] Understand two heaps pattern
- [ ] Can apply K-way merge
- [ ] Know when heap is better than sorting
