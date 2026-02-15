# Sorting and Searching

## Overview

Sorting and searching are fundamental algorithms that every engineer should know. As a Lead, you should understand the trade-offs between different algorithms and be able to choose the right one for specific scenarios.

---

## Sorting Algorithms Comparison

| Algorithm | Best | Average | Worst | Space | Stable | Notes |
|-----------|------|---------|-------|-------|--------|-------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Educational only |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No | Minimal swaps |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Good for small/nearly sorted |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | Consistent performance |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No | Fastest in practice |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No | In-place, consistent |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes | Integer keys, limited range |
| Radix Sort | O(d(n+k)) | O(d(n+k)) | O(d(n+k)) | O(n+k) | Yes | Fixed-length integers |
| Bucket Sort | O(n+k) | O(n+k) | O(n²) | O(n) | Yes | Uniform distribution |

---

## Core Sorting Algorithms

### Merge Sort

**Python:**
```python
def merge_sort(arr):
    """
    Divide and conquer - O(n log n) always
    Stable, good for linked lists, external sorting
    """
    if len(arr) <= 1:
        return arr

    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])

    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0

    while i < len(left) and j < len(right):
        if left[i] <= right[j]:  # <= for stability
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1

    result.extend(left[i:])
    result.extend(right[j:])
    return result

def merge_sort_in_place(arr, left, right):
    """In-place merge sort (for interviews)"""
    if left < right:
        mid = (left + right) // 2
        merge_sort_in_place(arr, left, mid)
        merge_sort_in_place(arr, mid + 1, right)
        merge_in_place(arr, left, mid, right)

def merge_in_place(arr, left, mid, right):
    left_copy = arr[left:mid + 1]
    right_copy = arr[mid + 1:right + 1]

    i = j = 0
    k = left

    while i < len(left_copy) and j < len(right_copy):
        if left_copy[i] <= right_copy[j]:
            arr[k] = left_copy[i]
            i += 1
        else:
            arr[k] = right_copy[j]
            j += 1
        k += 1

    while i < len(left_copy):
        arr[k] = left_copy[i]
        i += 1
        k += 1

    while j < len(right_copy):
        arr[k] = right_copy[j]
        j += 1
        k += 1
```

**JavaScript:**
```javascript
function mergeSort(arr) {
  // Divide and conquer - O(n log n) always
  // Stable, good for linked lists, external sorting
  if (arr.length <= 1) return arr;

  const mid = Math.floor(arr.length / 2);
  const left = mergeSort(arr.slice(0, mid));
  const right = mergeSort(arr.slice(mid));

  return merge(left, right);
}

function merge(left, right) {
  const result = [];
  let i = 0, j = 0;

  while (i < left.length && j < right.length) {
    if (left[i] <= right[j]) { // <= for stability
      result.push(left[i]);
      i++;
    } else {
      result.push(right[j]);
      j++;
    }
  }

  return [...result, ...left.slice(i), ...right.slice(j)];
}

function mergeSortInPlace(arr, left, right) {
  // In-place merge sort (for interviews)
  if (left < right) {
    const mid = Math.floor((left + right) / 2);
    mergeSortInPlace(arr, left, mid);
    mergeSortInPlace(arr, mid + 1, right);
    mergeInPlace(arr, left, mid, right);
  }
}

function mergeInPlace(arr, left, mid, right) {
  const leftCopy = arr.slice(left, mid + 1);
  const rightCopy = arr.slice(mid + 1, right + 1);

  let i = 0, j = 0, k = left;

  while (i < leftCopy.length && j < rightCopy.length) {
    if (leftCopy[i] <= rightCopy[j]) {
      arr[k] = leftCopy[i];
      i++;
    } else {
      arr[k] = rightCopy[j];
      j++;
    }
    k++;
  }

  while (i < leftCopy.length) {
    arr[k] = leftCopy[i];
    i++;
    k++;
  }

  while (j < rightCopy.length) {
    arr[k] = rightCopy[j];
    j++;
    k++;
  }
}
```

### Quick Sort

**Python:**
```python
def quick_sort(arr, low=0, high=None):
    """
    Divide and conquer - O(n log n) average, O(n²) worst
    In-place, cache-friendly
    """
    if high is None:
        high = len(arr) - 1

    if low < high:
        pivot_idx = partition(arr, low, high)
        quick_sort(arr, low, pivot_idx - 1)
        quick_sort(arr, pivot_idx + 1, high)

    return arr

def partition(arr, low, high):
    """Lomuto partition scheme"""
    pivot = arr[high]
    i = low - 1

    for j in range(low, high):
        if arr[j] < pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]

    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1

def partition_hoare(arr, low, high):
    """Hoare partition - more efficient, fewer swaps"""
    pivot = arr[(low + high) // 2]
    i, j = low - 1, high + 1

    while True:
        i += 1
        while arr[i] < pivot:
            i += 1

        j -= 1
        while arr[j] > pivot:
            j -= 1

        if i >= j:
            return j

        arr[i], arr[j] = arr[j], arr[i]

def quick_sort_randomized(arr, low, high):
    """Randomized pivot to avoid worst case"""
    import random

    if low < high:
        # Random pivot
        rand_idx = random.randint(low, high)
        arr[rand_idx], arr[high] = arr[high], arr[rand_idx]

        pivot_idx = partition(arr, low, high)
        quick_sort_randomized(arr, low, pivot_idx - 1)
        quick_sort_randomized(arr, pivot_idx + 1, high)

    return arr
```

**JavaScript:**
```javascript
function quickSort(arr, low = 0, high = arr.length - 1) {
  // Divide and conquer - O(n log n) average, O(n²) worst
  // In-place, cache-friendly
  if (low < high) {
    const pivotIdx = partition(arr, low, high);
    quickSort(arr, low, pivotIdx - 1);
    quickSort(arr, pivotIdx + 1, high);
  }

  return arr;
}

function partition(arr, low, high) {
  // Lomuto partition scheme
  const pivot = arr[high];
  let i = low - 1;

  for (let j = low; j < high; j++) {
    if (arr[j] < pivot) {
      i++;
      [arr[i], arr[j]] = [arr[j], arr[i]];
    }
  }

  [arr[i + 1], arr[high]] = [arr[high], arr[i + 1]];
  return i + 1;
}

function partitionHoare(arr, low, high) {
  // Hoare partition - more efficient, fewer swaps
  const pivot = arr[Math.floor((low + high) / 2)];
  let i = low - 1, j = high + 1;

  while (true) {
    do { i++; } while (arr[i] < pivot);
    do { j--; } while (arr[j] > pivot);

    if (i >= j) return j;

    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
}

function quickSortRandomized(arr, low, high) {
  // Randomized pivot to avoid worst case
  if (low < high) {
    // Random pivot
    const randIdx = low + Math.floor(Math.random() * (high - low + 1));
    [arr[randIdx], arr[high]] = [arr[high], arr[randIdx]];

    const pivotIdx = partition(arr, low, high);
    quickSortRandomized(arr, low, pivotIdx - 1);
    quickSortRandomized(arr, pivotIdx + 1, high);
  }

  return arr;
}
```

### Heap Sort

**Python:**
```python
def heap_sort(arr):
    """
    In-place, O(n log n) guaranteed
    Not stable, not cache-friendly
    """
    n = len(arr)

    # Build max heap
    for i in range(n // 2 - 1, -1, -1):
        heapify(arr, n, i)

    # Extract elements one by one
    for i in range(n - 1, 0, -1):
        arr[0], arr[i] = arr[i], arr[0]
        heapify(arr, i, 0)

    return arr

def heapify(arr, n, i):
    """Maintain max-heap property"""
    largest = i
    left = 2 * i + 1
    right = 2 * i + 2

    if left < n and arr[left] > arr[largest]:
        largest = left

    if right < n and arr[right] > arr[largest]:
        largest = right

    if largest != i:
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)
```

**JavaScript:**
```javascript
function heapSort(arr) {
  // In-place, O(n log n) guaranteed
  // Not stable, not cache-friendly
  const n = arr.length;

  // Build max heap
  for (let i = Math.floor(n / 2) - 1; i >= 0; i--) {
    heapify(arr, n, i);
  }

  // Extract elements one by one
  for (let i = n - 1; i > 0; i--) {
    [arr[0], arr[i]] = [arr[i], arr[0]];
    heapify(arr, i, 0);
  }

  return arr;
}

function heapify(arr, n, i) {
  // Maintain max-heap property
  let largest = i;
  const left = 2 * i + 1;
  const right = 2 * i + 2;

  if (left < n && arr[left] > arr[largest]) {
    largest = left;
  }

  if (right < n && arr[right] > arr[largest]) {
    largest = right;
  }

  if (largest !== i) {
    [arr[i], arr[largest]] = [arr[largest], arr[i]];
    heapify(arr, n, largest);
  }
}
```

### Counting Sort

**Python:**
```python
def counting_sort(arr, max_val=None):
    """
    O(n + k) where k is range of values
    Works for non-negative integers
    """
    if not arr:
        return arr

    if max_val is None:
        max_val = max(arr)

    count = [0] * (max_val + 1)

    # Count occurrences
    for num in arr:
        count[num] += 1

    # Build sorted array
    result = []
    for num, cnt in enumerate(count):
        result.extend([num] * cnt)

    return result

def counting_sort_stable(arr, max_val):
    """Stable version - preserves relative order"""
    count = [0] * (max_val + 1)

    for num in arr:
        count[num] += 1

    # Convert to cumulative count
    for i in range(1, len(count)):
        count[i] += count[i - 1]

    # Build output (traverse backwards for stability)
    output = [0] * len(arr)
    for i in range(len(arr) - 1, -1, -1):
        num = arr[i]
        output[count[num] - 1] = num
        count[num] -= 1

    return output
```

**JavaScript:**
```javascript
function countingSort(arr, maxVal = null) {
  // O(n + k) where k is range of values
  // Works for non-negative integers
  if (arr.length === 0) return arr;

  if (maxVal === null) {
    maxVal = Math.max(...arr);
  }

  const count = new Array(maxVal + 1).fill(0);

  // Count occurrences
  for (const num of arr) {
    count[num]++;
  }

  // Build sorted array
  const result = [];
  for (let num = 0; num < count.length; num++) {
    for (let i = 0; i < count[num]; i++) {
      result.push(num);
    }
  }

  return result;
}

function countingSortStable(arr, maxVal) {
  // Stable version - preserves relative order
  const count = new Array(maxVal + 1).fill(0);

  for (const num of arr) {
    count[num]++;
  }

  // Convert to cumulative count
  for (let i = 1; i < count.length; i++) {
    count[i] += count[i - 1];
  }

  // Build output (traverse backwards for stability)
  const output = new Array(arr.length);
  for (let i = arr.length - 1; i >= 0; i--) {
    const num = arr[i];
    output[count[num] - 1] = num;
    count[num]--;
  }

  return output;
}
```

### Radix Sort

**Python:**
```python
def radix_sort(arr):
    """
    O(d * (n + k)) where d is number of digits
    Works for non-negative integers
    """
    if not arr:
        return arr

    max_val = max(arr)
    exp = 1

    while max_val // exp > 0:
        counting_sort_by_digit(arr, exp)
        exp *= 10

    return arr

def counting_sort_by_digit(arr, exp):
    n = len(arr)
    output = [0] * n
    count = [0] * 10

    for num in arr:
        digit = (num // exp) % 10
        count[digit] += 1

    for i in range(1, 10):
        count[i] += count[i - 1]

    for i in range(n - 1, -1, -1):
        digit = (arr[i] // exp) % 10
        output[count[digit] - 1] = arr[i]
        count[digit] -= 1

    for i in range(n):
        arr[i] = output[i]
```

**JavaScript:**
```javascript
function radixSort(arr) {
  // O(d * (n + k)) where d is number of digits
  // Works for non-negative integers
  if (arr.length === 0) return arr;

  const maxVal = Math.max(...arr);
  let exp = 1;

  while (Math.floor(maxVal / exp) > 0) {
    countingSortByDigit(arr, exp);
    exp *= 10;
  }

  return arr;
}

function countingSortByDigit(arr, exp) {
  const n = arr.length;
  const output = new Array(n);
  const count = new Array(10).fill(0);

  for (const num of arr) {
    const digit = Math.floor(num / exp) % 10;
    count[digit]++;
  }

  for (let i = 1; i < 10; i++) {
    count[i] += count[i - 1];
  }

  for (let i = n - 1; i >= 0; i--) {
    const digit = Math.floor(arr[i] / exp) % 10;
    output[count[digit] - 1] = arr[i];
    count[digit]--;
  }

  for (let i = 0; i < n; i++) {
    arr[i] = output[i];
  }
}
```

---

## Binary Search

### Basic Template

**Python:**
```python
def binary_search(arr, target):
    """
    Standard binary search - O(log n)
    Returns index if found, -1 otherwise
    """
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = left + (right - left) // 2  # Avoid overflow

        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1
```

**JavaScript:**
```javascript
function binarySearch(arr, target) {
  // Standard binary search - O(log n)
  // Returns index if found, -1 otherwise
  let left = 0, right = arr.length - 1;

  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2); // Avoid overflow

    if (arr[mid] === target) {
      return mid;
    } else if (arr[mid] < target) {
      left = mid + 1;
    } else {
      right = mid - 1;
    }
  }

  return -1;
}
```

### Lower Bound (First >= Target)

**Python:**
```python
def lower_bound(arr, target):
    """
    Find first index where arr[i] >= target
    bisect.bisect_left equivalent
    """
    left, right = 0, len(arr)

    while left < right:
        mid = left + (right - left) // 2

        if arr[mid] < target:
            left = mid + 1
        else:
            right = mid

    return left
```

**JavaScript:**
```javascript
function lowerBound(arr, target) {
  // Find first index where arr[i] >= target
  // bisect_left equivalent
  let left = 0, right = arr.length;

  while (left < right) {
    const mid = left + Math.floor((right - left) / 2);

    if (arr[mid] < target) {
      left = mid + 1;
    } else {
      right = mid;
    }
  }

  return left;
}
```

### Upper Bound (First > Target)

**Python:**
```python
def upper_bound(arr, target):
    """
    Find first index where arr[i] > target
    bisect.bisect_right equivalent
    """
    left, right = 0, len(arr)

    while left < right:
        mid = left + (right - left) // 2

        if arr[mid] <= target:
            left = mid + 1
        else:
            right = mid

    return left
```

**JavaScript:**
```javascript
function upperBound(arr, target) {
  // Find first index where arr[i] > target
  // bisect_right equivalent
  let left = 0, right = arr.length;

  while (left < right) {
    const mid = left + Math.floor((right - left) / 2);

    if (arr[mid] <= target) {
      left = mid + 1;
    } else {
      right = mid;
    }
  }

  return left;
}
```

### Search Insert Position

**Python:**
```python
def search_insert(nums, target):
    """Find index to insert target to keep array sorted"""
    left, right = 0, len(nums)

    while left < right:
        mid = (left + right) // 2
        if nums[mid] < target:
            left = mid + 1
        else:
            right = mid

    return left
```

**JavaScript:**
```javascript
function searchInsert(nums, target) {
  // Find index to insert target to keep array sorted
  let left = 0, right = nums.length;

  while (left < right) {
    const mid = Math.floor((left + right) / 2);
    if (nums[mid] < target) {
      left = mid + 1;
    } else {
      right = mid;
    }
  }

  return left;
}
```

### First and Last Position

**Python:**
```python
def search_range(nums, target):
    """Find first and last index of target"""
    def find_first():
        left, right = 0, len(nums) - 1
        result = -1

        while left <= right:
            mid = (left + right) // 2
            if nums[mid] == target:
                result = mid
                right = mid - 1  # Keep searching left
            elif nums[mid] < target:
                left = mid + 1
            else:
                right = mid - 1

        return result

    def find_last():
        left, right = 0, len(nums) - 1
        result = -1

        while left <= right:
            mid = (left + right) // 2
            if nums[mid] == target:
                result = mid
                left = mid + 1  # Keep searching right
            elif nums[mid] < target:
                left = mid + 1
            else:
                right = mid - 1

        return result

    return [find_first(), find_last()]
```

**JavaScript:**
```javascript
function searchRange(nums, target) {
  // Find first and last index of target
  function findFirst() {
    let left = 0, right = nums.length - 1;
    let result = -1;

    while (left <= right) {
      const mid = Math.floor((left + right) / 2);
      if (nums[mid] === target) {
        result = mid;
        right = mid - 1; // Keep searching left
      } else if (nums[mid] < target) {
        left = mid + 1;
      } else {
        right = mid - 1;
      }
    }

    return result;
  }

  function findLast() {
    let left = 0, right = nums.length - 1;
    let result = -1;

    while (left <= right) {
      const mid = Math.floor((left + right) / 2);
      if (nums[mid] === target) {
        result = mid;
        left = mid + 1; // Keep searching right
      } else if (nums[mid] < target) {
        left = mid + 1;
      } else {
        right = mid - 1;
      }
    }

    return result;
  }

  return [findFirst(), findLast()];
}
```

---

## Binary Search Variations

### Search in Rotated Sorted Array

**Python:**
```python
def search_rotated(nums, target):
    """Array rotated at some pivot"""
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = (left + right) // 2

        if nums[mid] == target:
            return mid

        # Left half is sorted
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        # Right half is sorted
        else:
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1

    return -1

def find_minimum_rotated(nums):
    """Find minimum in rotated sorted array"""
    left, right = 0, len(nums) - 1

    while left < right:
        mid = (left + right) // 2

        if nums[mid] > nums[right]:
            left = mid + 1
        else:
            right = mid

    return nums[left]
```

**JavaScript:**
```javascript
function searchRotated(nums, target) {
  // Array rotated at some pivot
  let left = 0, right = nums.length - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);

    if (nums[mid] === target) return mid;

    // Left half is sorted
    if (nums[left] <= nums[mid]) {
      if (nums[left] <= target && target < nums[mid]) {
        right = mid - 1;
      } else {
        left = mid + 1;
      }
    // Right half is sorted
    } else {
      if (nums[mid] < target && target <= nums[right]) {
        left = mid + 1;
      } else {
        right = mid - 1;
      }
    }
  }

  return -1;
}

function findMinimumRotated(nums) {
  // Find minimum in rotated sorted array
  let left = 0, right = nums.length - 1;

  while (left < right) {
    const mid = Math.floor((left + right) / 2);

    if (nums[mid] > nums[right]) {
      left = mid + 1;
    } else {
      right = mid;
    }
  }

  return nums[left];
}
```

### Search in 2D Matrix

**Python:**
```python
def search_matrix(matrix, target):
    """
    Matrix where rows and columns are sorted
    Each row's first element > last element of previous row
    """
    if not matrix or not matrix[0]:
        return False

    m, n = len(matrix), len(matrix[0])
    left, right = 0, m * n - 1

    while left <= right:
        mid = (left + right) // 2
        mid_val = matrix[mid // n][mid % n]

        if mid_val == target:
            return True
        elif mid_val < target:
            left = mid + 1
        else:
            right = mid - 1

    return False

def search_matrix_ii(matrix, target):
    """
    Matrix where rows are sorted left to right
    And columns are sorted top to bottom
    Start from top-right or bottom-left
    """
    if not matrix or not matrix[0]:
        return False

    m, n = len(matrix), len(matrix[0])
    row, col = 0, n - 1

    while row < m and col >= 0:
        if matrix[row][col] == target:
            return True
        elif matrix[row][col] > target:
            col -= 1
        else:
            row += 1

    return False
```

**JavaScript:**
```javascript
function searchMatrix(matrix, target) {
  // Matrix where each row's first element > last element of previous row
  if (!matrix || matrix.length === 0 || matrix[0].length === 0) {
    return false;
  }

  const m = matrix.length, n = matrix[0].length;
  let left = 0, right = m * n - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    const midVal = matrix[Math.floor(mid / n)][mid % n];

    if (midVal === target) {
      return true;
    } else if (midVal < target) {
      left = mid + 1;
    } else {
      right = mid - 1;
    }
  }

  return false;
}

function searchMatrixII(matrix, target) {
  // Matrix where rows sorted left to right, columns top to bottom
  // Start from top-right or bottom-left
  if (!matrix || matrix.length === 0 || matrix[0].length === 0) {
    return false;
  }

  const m = matrix.length, n = matrix[0].length;
  let row = 0, col = n - 1;

  while (row < m && col >= 0) {
    if (matrix[row][col] === target) {
      return true;
    } else if (matrix[row][col] > target) {
      col--;
    } else {
      row++;
    }
  }

  return false;
}
```

### Peak Element

**Python:**
```python
def find_peak_element(nums):
    """Find any peak (greater than neighbors)"""
    left, right = 0, len(nums) - 1

    while left < right:
        mid = (left + right) // 2

        if nums[mid] < nums[mid + 1]:
            left = mid + 1
        else:
            right = mid

    return left
```

**JavaScript:**
```javascript
function findPeakElement(nums) {
  // Find any peak (greater than neighbors)
  let left = 0, right = nums.length - 1;

  while (left < right) {
    const mid = Math.floor((left + right) / 2);

    if (nums[mid] < nums[mid + 1]) {
      left = mid + 1;
    } else {
      right = mid;
    }
  }

  return left;
}
```

### Find Kth Element

**Python:**
```python
def find_kth_largest(nums, k):
    """
    QuickSelect - O(n) average, O(n²) worst
    """
    import random

    def partition(left, right, pivot_idx):
        pivot = nums[pivot_idx]
        nums[pivot_idx], nums[right] = nums[right], nums[pivot_idx]

        store_idx = left
        for i in range(left, right):
            if nums[i] > pivot:  # For kth largest
                nums[store_idx], nums[i] = nums[i], nums[store_idx]
                store_idx += 1

        nums[right], nums[store_idx] = nums[store_idx], nums[right]
        return store_idx

    def quickselect(left, right, k_idx):
        if left == right:
            return nums[left]

        pivot_idx = random.randint(left, right)
        pivot_idx = partition(left, right, pivot_idx)

        if k_idx == pivot_idx:
            return nums[k_idx]
        elif k_idx < pivot_idx:
            return quickselect(left, pivot_idx - 1, k_idx)
        else:
            return quickselect(pivot_idx + 1, right, k_idx)

    return quickselect(0, len(nums) - 1, k - 1)
```

**JavaScript:**
```javascript
function findKthLargest(nums, k) {
  // QuickSelect - O(n) average, O(n²) worst

  function partition(left, right, pivotIdx) {
    const pivot = nums[pivotIdx];
    [nums[pivotIdx], nums[right]] = [nums[right], nums[pivotIdx]];

    let storeIdx = left;
    for (let i = left; i < right; i++) {
      if (nums[i] > pivot) { // For kth largest
        [nums[storeIdx], nums[i]] = [nums[i], nums[storeIdx]];
        storeIdx++;
      }
    }

    [nums[right], nums[storeIdx]] = [nums[storeIdx], nums[right]];
    return storeIdx;
  }

  function quickselect(left, right, kIdx) {
    if (left === right) return nums[left];

    let pivotIdx = left + Math.floor(Math.random() * (right - left + 1));
    pivotIdx = partition(left, right, pivotIdx);

    if (kIdx === pivotIdx) {
      return nums[kIdx];
    } else if (kIdx < pivotIdx) {
      return quickselect(left, pivotIdx - 1, kIdx);
    } else {
      return quickselect(pivotIdx + 1, right, kIdx);
    }
  }

  return quickselect(0, nums.length - 1, k - 1);
}
```

### Median of Two Sorted Arrays

**Python:**
```python
def find_median_sorted_arrays(nums1, nums2):
    """O(log(min(m, n)))"""
    if len(nums1) > len(nums2):
        nums1, nums2 = nums2, nums1

    m, n = len(nums1), len(nums2)
    left, right = 0, m
    half_len = (m + n + 1) // 2

    while left <= right:
        partition1 = (left + right) // 2
        partition2 = half_len - partition1

        max_left1 = float('-inf') if partition1 == 0 else nums1[partition1 - 1]
        min_right1 = float('inf') if partition1 == m else nums1[partition1]
        max_left2 = float('-inf') if partition2 == 0 else nums2[partition2 - 1]
        min_right2 = float('inf') if partition2 == n else nums2[partition2]

        if max_left1 <= min_right2 and max_left2 <= min_right1:
            if (m + n) % 2 == 0:
                return (max(max_left1, max_left2) +
                       min(min_right1, min_right2)) / 2
            else:
                return max(max_left1, max_left2)
        elif max_left1 > min_right2:
            right = partition1 - 1
        else:
            left = partition1 + 1

    return 0
```

**JavaScript:**
```javascript
function findMedianSortedArrays(nums1, nums2) {
  // O(log(min(m, n)))
  if (nums1.length > nums2.length) {
    [nums1, nums2] = [nums2, nums1];
  }

  const m = nums1.length, n = nums2.length;
  let left = 0, right = m;
  const halfLen = Math.floor((m + n + 1) / 2);

  while (left <= right) {
    const partition1 = Math.floor((left + right) / 2);
    const partition2 = halfLen - partition1;

    const maxLeft1 = partition1 === 0 ? -Infinity : nums1[partition1 - 1];
    const minRight1 = partition1 === m ? Infinity : nums1[partition1];
    const maxLeft2 = partition2 === 0 ? -Infinity : nums2[partition2 - 1];
    const minRight2 = partition2 === n ? Infinity : nums2[partition2];

    if (maxLeft1 <= minRight2 && maxLeft2 <= minRight1) {
      if ((m + n) % 2 === 0) {
        return (Math.max(maxLeft1, maxLeft2) +
                Math.min(minRight1, minRight2)) / 2;
      } else {
        return Math.max(maxLeft1, maxLeft2);
      }
    } else if (maxLeft1 > minRight2) {
      right = partition1 - 1;
    } else {
      left = partition1 + 1;
    }
  }

  return 0;
}
```

---

## Binary Search on Answer

### Square Root

**Python:**
```python
def sqrt(x):
    """Binary search for floor(sqrt(x))"""
    if x < 2:
        return x

    left, right = 1, x // 2

    while left <= right:
        mid = (left + right) // 2
        square = mid * mid

        if square == x:
            return mid
        elif square < x:
            left = mid + 1
        else:
            right = mid - 1

    return right
```

**JavaScript:**
```javascript
function mySqrt(x) {
  // Binary search for floor(sqrt(x))
  if (x < 2) return x;

  let left = 1, right = Math.floor(x / 2);

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    const square = mid * mid;

    if (square === x) {
      return mid;
    } else if (square < x) {
      left = mid + 1;
    } else {
      right = mid - 1;
    }
  }

  return right;
}
```

### Koko Eating Bananas

**Python:**
```python
def min_eating_speed(piles, h):
    """Minimum speed to eat all bananas in h hours"""
    import math

    def can_finish(speed):
        hours = sum(math.ceil(pile / speed) for pile in piles)
        return hours <= h

    left, right = 1, max(piles)

    while left < right:
        mid = (left + right) // 2

        if can_finish(mid):
            right = mid
        else:
            left = mid + 1

    return left
```

**JavaScript:**
```javascript
function minEatingSpeed(piles, h) {
  // Minimum speed to eat all bananas in h hours

  function canFinish(speed) {
    let hours = 0;
    for (const pile of piles) {
      hours += Math.ceil(pile / speed);
    }
    return hours <= h;
  }

  let left = 1, right = Math.max(...piles);

  while (left < right) {
    const mid = Math.floor((left + right) / 2);

    if (canFinish(mid)) {
      right = mid;
    } else {
      left = mid + 1;
    }
  }

  return left;
}
```

### Capacity to Ship Packages

**Python:**
```python
def ship_within_days(weights, days):
    """Minimum capacity to ship all packages in given days"""
    def can_ship(capacity):
        day_count = 1
        current_load = 0

        for weight in weights:
            if current_load + weight > capacity:
                day_count += 1
                current_load = weight
            else:
                current_load += weight

        return day_count <= days

    left = max(weights)  # Must carry at least the heaviest
    right = sum(weights)  # Ship everything in one day

    while left < right:
        mid = (left + right) // 2

        if can_ship(mid):
            right = mid
        else:
            left = mid + 1

    return left
```

**JavaScript:**
```javascript
function shipWithinDays(weights, days) {
  // Minimum capacity to ship all packages in given days

  function canShip(capacity) {
    let dayCount = 1;
    let currentLoad = 0;

    for (const weight of weights) {
      if (currentLoad + weight > capacity) {
        dayCount++;
        currentLoad = weight;
      } else {
        currentLoad += weight;
      }
    }

    return dayCount <= days;
  }

  let left = Math.max(...weights); // Must carry at least the heaviest
  let right = weights.reduce((a, b) => a + b, 0); // Ship everything in one day

  while (left < right) {
    const mid = Math.floor((left + right) / 2);

    if (canShip(mid)) {
      right = mid;
    } else {
      left = mid + 1;
    }
  }

  return left;
}
```

### Split Array Largest Sum

**Python:**
```python
def split_array(nums, k):
    """Minimize the largest sum among k subarrays"""
    def can_split(max_sum):
        count = 1
        current_sum = 0

        for num in nums:
            if current_sum + num > max_sum:
                count += 1
                current_sum = num
            else:
                current_sum += num

        return count <= k

    left = max(nums)
    right = sum(nums)

    while left < right:
        mid = (left + right) // 2

        if can_split(mid):
            right = mid
        else:
            left = mid + 1

    return left
```

**JavaScript:**
```javascript
function splitArray(nums, k) {
  // Minimize the largest sum among k subarrays

  function canSplit(maxSum) {
    let count = 1;
    let currentSum = 0;

    for (const num of nums) {
      if (currentSum + num > maxSum) {
        count++;
        currentSum = num;
      } else {
        currentSum += num;
      }
    }

    return count <= k;
  }

  let left = Math.max(...nums);
  let right = nums.reduce((a, b) => a + b, 0);

  while (left < right) {
    const mid = Math.floor((left + right) / 2);

    if (canSplit(mid)) {
      right = mid;
    } else {
      left = mid + 1;
    }
  }

  return left;
}
```

---

## Sorting Applications

### Merge Intervals

**Python:**
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

**JavaScript:**
```javascript
function mergeIntervals(intervals) {
  intervals.sort((a, b) => a[0] - b[0]);
  const merged = [];

  for (const interval of intervals) {
    if (merged.length === 0 || merged[merged.length - 1][1] < interval[0]) {
      merged.push(interval);
    } else {
      merged[merged.length - 1][1] = Math.max(merged[merged.length - 1][1], interval[1]);
    }
  }

  return merged;
}
```

### Meeting Rooms

**Python:**
```python
def can_attend_meetings(intervals):
    """Can attend all meetings (no overlap)?"""
    intervals.sort()

    for i in range(1, len(intervals)):
        if intervals[i][0] < intervals[i-1][1]:
            return False

    return True

def min_meeting_rooms(intervals):
    """Minimum rooms needed"""
    import heapq

    if not intervals:
        return 0

    intervals.sort()
    heap = []  # End times

    for start, end in intervals:
        if heap and heap[0] <= start:
            heapq.heappop(heap)
        heapq.heappush(heap, end)

    return len(heap)
```

**JavaScript:**
```javascript
function canAttendMeetings(intervals) {
  // Can attend all meetings (no overlap)?
  intervals.sort((a, b) => a[0] - b[0]);

  for (let i = 1; i < intervals.length; i++) {
    if (intervals[i][0] < intervals[i - 1][1]) {
      return false;
    }
  }

  return true;
}

function minMeetingRooms(intervals) {
  // Minimum rooms needed
  if (intervals.length === 0) return 0;

  // Using two arrays approach (simpler without heap in JS)
  const starts = intervals.map(i => i[0]).sort((a, b) => a - b);
  const ends = intervals.map(i => i[1]).sort((a, b) => a - b);

  let rooms = 0, endPtr = 0;

  for (let i = 0; i < starts.length; i++) {
    if (starts[i] < ends[endPtr]) {
      rooms++;
    } else {
      endPtr++;
    }
  }

  return rooms;
}
```

### Sort Colors (Dutch National Flag)

**Python:**
```python
def sort_colors(nums):
    """Sort array with 0, 1, 2 in-place"""
    low, mid, high = 0, 0, len(nums) - 1

    while mid <= high:
        if nums[mid] == 0:
            nums[low], nums[mid] = nums[mid], nums[low]
            low += 1
            mid += 1
        elif nums[mid] == 1:
            mid += 1
        else:
            nums[mid], nums[high] = nums[high], nums[mid]
            high -= 1
```

**JavaScript:**
```javascript
function sortColors(nums) {
  // Sort array with 0, 1, 2 in-place (Dutch National Flag)
  let low = 0, mid = 0, high = nums.length - 1;

  while (mid <= high) {
    if (nums[mid] === 0) {
      [nums[low], nums[mid]] = [nums[mid], nums[low]];
      low++;
      mid++;
    } else if (nums[mid] === 1) {
      mid++;
    } else {
      [nums[mid], nums[high]] = [nums[high], nums[mid]];
      high--;
    }
  }
}
```

---

## Must-Know Problems

### Sorting
1. Sort Colors
2. Merge Intervals
3. Meeting Rooms I & II
4. Top K Frequent Elements
5. Kth Largest Element in Array

### Binary Search
1. Binary Search
2. Search in Rotated Sorted Array I & II
3. Find Minimum in Rotated Sorted Array
4. Search a 2D Matrix I & II
5. Find Peak Element
6. First and Last Position in Sorted Array

### Binary Search on Answer
1. Sqrt(x)
2. Koko Eating Bananas
3. Capacity to Ship Packages Within D Days
4. Split Array Largest Sum
5. Median of Two Sorted Arrays

---

## Practice Checklist

- [ ] Can implement merge sort and quick sort from memory
- [ ] Understand when to use each sorting algorithm
- [ ] Can write binary search without bugs
- [ ] Know lower_bound vs upper_bound difference
- [ ] Can apply binary search on answer pattern
