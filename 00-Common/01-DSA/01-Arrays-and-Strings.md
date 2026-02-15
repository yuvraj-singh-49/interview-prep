# Arrays and Strings

## Overview

Arrays and strings are the most fundamental data structures and appear in 40-50% of coding interviews. As a Lead engineer, you're expected to solve these optimally and explain trade-offs clearly.

---

## Key Concepts

### Time Complexity Quick Reference

| Operation | Array | Dynamic Array | String (immutable) |
|-----------|-------|---------------|-------------------|
| Access | O(1) | O(1) | O(1) |
| Search | O(n) | O(n) | O(n) |
| Insert (end) | N/A | O(1) amortized | O(n) |
| Insert (middle) | O(n) | O(n) | O(n) |
| Delete | O(n) | O(n) | O(n) |

### Memory Considerations

```
Array of n integers (32-bit): n * 4 bytes
Array of n integers (64-bit): n * 8 bytes
String of n chars (ASCII): n bytes
String of n chars (UTF-16): n * 2 bytes
```

---

## Essential Patterns

### 1. Two Pointers

**When to use:** Sorted arrays, finding pairs, palindrome checks, removing duplicates

#### Python
```python
def two_pointer_opposite(arr):
    """Pointers moving toward each other"""
    left, right = 0, len(arr) - 1
    while left < right:
        # Process based on condition
        if condition:
            left += 1
        else:
            right -= 1

def two_pointer_same_direction(arr):
    """Fast and slow pointers"""
    slow = 0
    for fast in range(len(arr)):
        if condition:
            arr[slow] = arr[fast]
            slow += 1
    return slow
```

#### JavaScript
```javascript
function twoPointerOpposite(arr) {
  // Pointers moving toward each other
  let left = 0;
  let right = arr.length - 1;

  while (left < right) {
    // Process based on condition
    if (condition) {
      left++;
    } else {
      right--;
    }
  }
}

function twoPointerSameDirection(arr) {
  // Fast and slow pointers
  let slow = 0;

  for (let fast = 0; fast < arr.length; fast++) {
    if (condition) {
      arr[slow] = arr[fast];
      slow++;
    }
  }
  return slow;
}
```

**Classic Problems:**
- Two Sum II (sorted array)
- Container With Most Water
- 3Sum / 4Sum
- Remove Duplicates from Sorted Array
- Trapping Rain Water

---

### 2. Sliding Window

**When to use:** Contiguous subarrays/substrings, finding max/min in window, string matching

#### Python - Fixed Size Window
```python
def fixed_window(arr, k):
    window_sum = sum(arr[:k])
    result = window_sum

    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i - k]  # Slide window
        result = max(result, window_sum)

    return result
```

#### JavaScript - Fixed Size Window
```javascript
function fixedWindow(arr, k) {
  let windowSum = arr.slice(0, k).reduce((a, b) => a + b, 0);
  let result = windowSum;

  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k];  // Slide window
    result = Math.max(result, windowSum);
  }

  return result;
}
```

#### Python - Variable Size Window
```python
def variable_window(arr, target):
    left = 0
    window_sum = 0
    result = float('inf')

    for right in range(len(arr)):
        window_sum += arr[right]  # Expand window

        while window_sum >= target:  # Contract window
            result = min(result, right - left + 1)
            window_sum -= arr[left]
            left += 1

    return result if result != float('inf') else 0
```

#### JavaScript - Variable Size Window
```javascript
function variableWindow(arr, target) {
  let left = 0;
  let windowSum = 0;
  let result = Infinity;

  for (let right = 0; right < arr.length; right++) {
    windowSum += arr[right];  // Expand window

    while (windowSum >= target) {  // Contract window
      result = Math.min(result, right - left + 1);
      windowSum -= arr[left];
      left++;
    }
  }

  return result === Infinity ? 0 : result;
}
```

**Classic Problems:**
- Maximum Sum Subarray of Size K
- Longest Substring Without Repeating Characters
- Minimum Window Substring
- Longest Repeating Character Replacement
- Permutation in String
- Sliding Window Maximum (with deque)

---

### 3. Prefix Sum

**When to use:** Range sum queries, subarray sums, finding subarrays with target sum

#### Python
```python
def build_prefix_sum(arr):
    prefix = [0] * (len(arr) + 1)
    for i in range(len(arr)):
        prefix[i + 1] = prefix[i] + arr[i]
    return prefix

def range_sum(prefix, left, right):
    """Sum of arr[left:right+1]"""
    return prefix[right + 1] - prefix[left]
```

#### JavaScript
```javascript
function buildPrefixSum(arr) {
  const prefix = new Array(arr.length + 1).fill(0);
  for (let i = 0; i < arr.length; i++) {
    prefix[i + 1] = prefix[i] + arr[i];
  }
  return prefix;
}

function rangeSum(prefix, left, right) {
  // Sum of arr[left:right+1]
  return prefix[right + 1] - prefix[left];
}
```

**Classic Problems:**
- Subarray Sum Equals K
- Continuous Subarray Sum
- Range Sum Query - Immutable
- Product of Array Except Self

---

### 4. Hash Map for Frequency/Lookup

**When to use:** Finding duplicates, counting occurrences, two-sum variants

#### Python
```python
from collections import Counter, defaultdict

def frequency_count(arr):
    freq = Counter(arr)
    # or
    freq = defaultdict(int)
    for num in arr:
        freq[num] += 1
    return freq

def two_sum(arr, target):
    seen = {}
    for i, num in enumerate(arr):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []
```

#### JavaScript
```javascript
function frequencyCount(arr) {
  const freq = new Map();
  for (const num of arr) {
    freq.set(num, (freq.get(num) || 0) + 1);
  }
  return freq;
}

function twoSum(arr, target) {
  const seen = new Map();

  for (let i = 0; i < arr.length; i++) {
    const complement = target - arr[i];

    if (seen.has(complement)) {
      return [seen.get(complement), i];
    }
    seen.set(arr[i], i);
  }
  return [];
}
```

---

### 5. Kadane's Algorithm (Maximum Subarray)

**When to use:** Finding maximum/minimum sum contiguous subarray

#### Python
```python
def kadane(arr):
    max_sum = curr_sum = arr[0]

    for num in arr[1:]:
        curr_sum = max(num, curr_sum + num)
        max_sum = max(max_sum, curr_sum)

    return max_sum

def kadane_with_indices(arr):
    max_sum = curr_sum = arr[0]
    start = end = temp_start = 0

    for i in range(1, len(arr)):
        if arr[i] > curr_sum + arr[i]:
            curr_sum = arr[i]
            temp_start = i
        else:
            curr_sum += arr[i]

        if curr_sum > max_sum:
            max_sum = curr_sum
            start = temp_start
            end = i

    return max_sum, start, end
```

#### JavaScript
```javascript
function kadane(arr) {
  let maxSum = arr[0];
  let currSum = arr[0];

  for (let i = 1; i < arr.length; i++) {
    currSum = Math.max(arr[i], currSum + arr[i]);
    maxSum = Math.max(maxSum, currSum);
  }

  return maxSum;
}

function kadaneWithIndices(arr) {
  let maxSum = arr[0];
  let currSum = arr[0];
  let start = 0, end = 0, tempStart = 0;

  for (let i = 1; i < arr.length; i++) {
    if (arr[i] > currSum + arr[i]) {
      currSum = arr[i];
      tempStart = i;
    } else {
      currSum += arr[i];
    }

    if (currSum > maxSum) {
      maxSum = currSum;
      start = tempStart;
      end = i;
    }
  }

  return { maxSum, start, end };
}
```

---

### 6. Dutch National Flag (3-way Partition)

**When to use:** Sorting with 3 distinct values, partitioning arrays

#### Python
```python
def dutch_national_flag(arr):
    """Sort array with 0s, 1s, 2s"""
    low, mid, high = 0, 0, len(arr) - 1

    while mid <= high:
        if arr[mid] == 0:
            arr[low], arr[mid] = arr[mid], arr[low]
            low += 1
            mid += 1
        elif arr[mid] == 1:
            mid += 1
        else:
            arr[mid], arr[high] = arr[high], arr[mid]
            high -= 1

    return arr
```

#### JavaScript
```javascript
function dutchNationalFlag(arr) {
  // Sort array with 0s, 1s, 2s
  let low = 0;
  let mid = 0;
  let high = arr.length - 1;

  while (mid <= high) {
    if (arr[mid] === 0) {
      [arr[low], arr[mid]] = [arr[mid], arr[low]];
      low++;
      mid++;
    } else if (arr[mid] === 1) {
      mid++;
    } else {
      [arr[mid], arr[high]] = [arr[high], arr[mid]];
      high--;
    }
  }

  return arr;
}
```

---

## String-Specific Patterns

### 1. Character Frequency with Fixed Array

#### Python
```python
def char_frequency(s):
    """For lowercase letters only - O(1) space"""
    freq = [0] * 26
    for c in s:
        freq[ord(c) - ord('a')] += 1
    return freq

def are_anagrams(s1, s2):
    return char_frequency(s1) == char_frequency(s2)
```

#### JavaScript
```javascript
function charFrequency(s) {
  // For lowercase letters only - O(1) space
  const freq = new Array(26).fill(0);
  for (const c of s) {
    freq[c.charCodeAt(0) - 'a'.charCodeAt(0)]++;
  }
  return freq;
}

function areAnagrams(s1, s2) {
  if (s1.length !== s2.length) return false;
  const freq1 = charFrequency(s1);
  const freq2 = charFrequency(s2);
  return freq1.every((val, i) => val === freq2[i]);
}
```

### 2. StringBuilder Pattern

#### Python
```python
# Python strings are immutable, use list and join
def build_string(parts):
    result = []
    for part in parts:
        result.append(part)
    return ''.join(result)  # O(n) instead of O(n^2)
```

#### JavaScript
```javascript
// JS strings are also immutable, but += is optimized in modern engines
// For clarity and large strings, use array + join
function buildString(parts) {
  const result = [];
  for (const part of parts) {
    result.push(part);
  }
  return result.join('');  // O(n) instead of O(n^2)
}
```

### 3. Palindrome Checks

#### Python
```python
def is_palindrome(s):
    left, right = 0, len(s) - 1
    while left < right:
        if s[left] != s[right]:
            return False
        left += 1
        right -= 1
    return True

def longest_palindromic_substring(s):
    """Expand around center - O(n^2)"""
    def expand(left, right):
        while left >= 0 and right < len(s) and s[left] == s[right]:
            left -= 1
            right += 1
        return s[left + 1:right]

    result = ""
    for i in range(len(s)):
        odd = expand(i, i)
        even = expand(i, i + 1)
        result = max(result, odd, even, key=len)

    return result
```

#### JavaScript
```javascript
function isPalindrome(s) {
  let left = 0;
  let right = s.length - 1;

  while (left < right) {
    if (s[left] !== s[right]) {
      return false;
    }
    left++;
    right--;
  }
  return true;
}

function longestPalindromicSubstring(s) {
  // Expand around center - O(n^2)
  function expand(left, right) {
    while (left >= 0 && right < s.length && s[left] === s[right]) {
      left--;
      right++;
    }
    return s.substring(left + 1, right);
  }

  let result = "";
  for (let i = 0; i < s.length; i++) {
    const odd = expand(i, i);
    const even = expand(i, i + 1);

    if (odd.length > result.length) result = odd;
    if (even.length > result.length) result = even;
  }

  return result;
}
```

### 4. KMP Pattern Matching (Advanced)

#### Python
```python
def kmp_search(text, pattern):
    """O(n + m) pattern matching"""
    def build_lps(pattern):
        lps = [0] * len(pattern)
        length = 0
        i = 1

        while i < len(pattern):
            if pattern[i] == pattern[length]:
                length += 1
                lps[i] = length
                i += 1
            elif length > 0:
                length = lps[length - 1]
            else:
                lps[i] = 0
                i += 1

        return lps

    lps = build_lps(pattern)
    i = j = 0
    results = []

    while i < len(text):
        if text[i] == pattern[j]:
            i += 1
            j += 1
            if j == len(pattern):
                results.append(i - j)
                j = lps[j - 1]
        elif j > 0:
            j = lps[j - 1]
        else:
            i += 1

    return results
```

#### JavaScript
```javascript
function kmpSearch(text, pattern) {
  // O(n + m) pattern matching
  function buildLPS(pattern) {
    const lps = new Array(pattern.length).fill(0);
    let length = 0;
    let i = 1;

    while (i < pattern.length) {
      if (pattern[i] === pattern[length]) {
        length++;
        lps[i] = length;
        i++;
      } else if (length > 0) {
        length = lps[length - 1];
      } else {
        lps[i] = 0;
        i++;
      }
    }

    return lps;
  }

  const lps = buildLPS(pattern);
  let i = 0, j = 0;
  const results = [];

  while (i < text.length) {
    if (text[i] === pattern[j]) {
      i++;
      j++;
      if (j === pattern.length) {
        results.push(i - j);
        j = lps[j - 1];
      }
    } else if (j > 0) {
      j = lps[j - 1];
    } else {
      i++;
    }
  }

  return results;
}
```

---

## Common Interview Problems with Solutions

### Two Sum

#### Python
```python
def two_sum(nums, target):
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []
```

#### JavaScript
```javascript
function twoSum(nums, target) {
  const seen = new Map();

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];

    if (seen.has(complement)) {
      return [seen.get(complement), i];
    }
    seen.set(nums[i], i);
  }
  return [];
}
```

### Container With Most Water

#### Python
```python
def max_area(height):
    left, right = 0, len(height) - 1
    max_water = 0

    while left < right:
        width = right - left
        h = min(height[left], height[right])
        max_water = max(max_water, width * h)

        if height[left] < height[right]:
            left += 1
        else:
            right -= 1

    return max_water
```

#### JavaScript
```javascript
function maxArea(height) {
  let left = 0;
  let right = height.length - 1;
  let maxWater = 0;

  while (left < right) {
    const width = right - left;
    const h = Math.min(height[left], height[right]);
    maxWater = Math.max(maxWater, width * h);

    if (height[left] < height[right]) {
      left++;
    } else {
      right--;
    }
  }

  return maxWater;
}
```

### Longest Substring Without Repeating Characters

#### Python
```python
def length_of_longest_substring(s):
    char_index = {}
    left = 0
    max_length = 0

    for right, char in enumerate(s):
        if char in char_index and char_index[char] >= left:
            left = char_index[char] + 1

        char_index[char] = right
        max_length = max(max_length, right - left + 1)

    return max_length
```

#### JavaScript
```javascript
function lengthOfLongestSubstring(s) {
  const charIndex = new Map();
  let left = 0;
  let maxLength = 0;

  for (let right = 0; right < s.length; right++) {
    const char = s[right];

    if (charIndex.has(char) && charIndex.get(char) >= left) {
      left = charIndex.get(char) + 1;
    }

    charIndex.set(char, right);
    maxLength = Math.max(maxLength, right - left + 1);
  }

  return maxLength;
}
```

### 3Sum

#### Python
```python
def three_sum(nums):
    nums.sort()
    result = []

    for i in range(len(nums) - 2):
        if i > 0 and nums[i] == nums[i - 1]:
            continue

        left, right = i + 1, len(nums) - 1

        while left < right:
            total = nums[i] + nums[left] + nums[right]

            if total == 0:
                result.append([nums[i], nums[left], nums[right]])
                while left < right and nums[left] == nums[left + 1]:
                    left += 1
                while left < right and nums[right] == nums[right - 1]:
                    right -= 1
                left += 1
                right -= 1
            elif total < 0:
                left += 1
            else:
                right -= 1

    return result
```

#### JavaScript
```javascript
function threeSum(nums) {
  nums.sort((a, b) => a - b);
  const result = [];

  for (let i = 0; i < nums.length - 2; i++) {
    if (i > 0 && nums[i] === nums[i - 1]) continue;

    let left = i + 1;
    let right = nums.length - 1;

    while (left < right) {
      const total = nums[i] + nums[left] + nums[right];

      if (total === 0) {
        result.push([nums[i], nums[left], nums[right]]);

        while (left < right && nums[left] === nums[left + 1]) left++;
        while (left < right && nums[right] === nums[right - 1]) right--;

        left++;
        right--;
      } else if (total < 0) {
        left++;
      } else {
        right--;
      }
    }
  }

  return result;
}
```

### Trapping Rain Water

#### Python
```python
def trap(height):
    if not height:
        return 0

    left, right = 0, len(height) - 1
    left_max = right_max = 0
    water = 0

    while left < right:
        if height[left] < height[right]:
            if height[left] >= left_max:
                left_max = height[left]
            else:
                water += left_max - height[left]
            left += 1
        else:
            if height[right] >= right_max:
                right_max = height[right]
            else:
                water += right_max - height[right]
            right -= 1

    return water
```

#### JavaScript
```javascript
function trap(height) {
  if (!height.length) return 0;

  let left = 0;
  let right = height.length - 1;
  let leftMax = 0;
  let rightMax = 0;
  let water = 0;

  while (left < right) {
    if (height[left] < height[right]) {
      if (height[left] >= leftMax) {
        leftMax = height[left];
      } else {
        water += leftMax - height[left];
      }
      left++;
    } else {
      if (height[right] >= rightMax) {
        rightMax = height[right];
      } else {
        water += rightMax - height[right];
      }
      right--;
    }
  }

  return water;
}
```

---

## Lead-Level Problem-Solving Approach

### 1. Clarify Requirements
- Input size and constraints
- Edge cases (empty, single element, all same)
- Expected time/space complexity
- Can we modify input array?

### 2. Discuss Multiple Approaches
```
Approach 1: Brute Force - O(n^2) time, O(1) space
Approach 2: Sorting - O(n log n) time, O(1) space
Approach 3: Hash Map - O(n) time, O(n) space
Approach 4: Two Pointers - O(n) time, O(1) space (if sorted)
```

### 3. Justify Your Choice
- "Given the constraint of n = 10^5, O(n^2) would be too slow"
- "We need O(n) or O(n log n) solution"
- "Trading space for time is acceptable here"

---

## Must-Know Problems (By Difficulty)

### Easy (Warm-up)
1. Two Sum
2. Best Time to Buy and Sell Stock
3. Contains Duplicate
4. Valid Anagram
5. Valid Palindrome

### Medium (Interview Standard)
1. 3Sum
2. Container With Most Water
3. Longest Substring Without Repeating Characters
4. Product of Array Except Self
5. Maximum Subarray
6. Merge Intervals
7. Group Anagrams
8. Subarray Sum Equals K
9. Minimum Window Substring
10. Longest Palindromic Substring

### Hard (Lead-Level)
1. Trapping Rain Water
2. Sliding Window Maximum
3. Minimum Window Substring
4. Longest Valid Parentheses
5. First Missing Positive
6. Median of Two Sorted Arrays

---

## Common Mistakes to Avoid

1. **Off-by-one errors** - Be careful with indices and boundaries
2. **Integer overflow** - Use long for sums, check constraints
3. **Not handling empty arrays** - Always check `if not arr` / `if (!arr.length)`
4. **Modifying array while iterating** - Use two-pointer or iterate backwards
5. **String immutability** - Don't concatenate in a loop (use join)

---

## Practice Checklist

- [ ] Can solve Two Sum in < 5 minutes
- [ ] Can explain sliding window pattern clearly
- [ ] Know when to use each pattern instinctively
- [ ] Can identify optimal approach by looking at constraints
- [ ] Can code without IDE autocomplete
- [ ] Can trace through code on whiteboard

---

## Resources

- [LeetCode Array Problems](https://leetcode.com/tag/array/)
- [NeetCode Arrays & Hashing](https://neetcode.io/roadmap)
- [Tech Interview Handbook - Arrays](https://www.techinterviewhandbook.org/algorithms/array/)
