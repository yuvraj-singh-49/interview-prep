# Dynamic Programming

## Overview

Dynamic Programming (DP) is one of the most challenging topics in interviews but also one of the most rewarding. As a Lead engineer, you should be able to identify DP problems, explain the approach clearly, and implement optimal solutions.

---

## Key Concepts

### When to Use DP

1. **Optimal Substructure**: Optimal solution contains optimal solutions to subproblems
2. **Overlapping Subproblems**: Same subproblems are solved multiple times

### DP Characteristics

- Counting problems: "How many ways..."
- Optimization problems: "Minimum/Maximum..."
- Yes/No problems: "Is it possible..."

### Two Approaches

| Approach | Description | Pros | Cons |
|----------|-------------|------|------|
| Top-Down (Memoization) | Recursive + cache | Natural, easier to write | Stack overflow risk |
| Bottom-Up (Tabulation) | Iterative, build from base | No recursion overhead | May compute unnecessary states |

---

## DP Problem-Solving Framework

### 1. Define the State
- What information do we need to uniquely identify a subproblem?
- Usually: `dp[i]`, `dp[i][j]`, `dp[i][j][k]`

### 2. Find the Recurrence Relation
- How does the current state relate to previous states?
- Express `dp[i]` in terms of smaller subproblems

### 3. Identify Base Cases
- What are the simplest subproblems with known answers?

### 4. Determine Computation Order
- Ensure dependencies are computed before use

### 5. Optimize Space (if possible)
- Often can reduce from O(n²) to O(n) or O(1)

---

## Classic DP Patterns

### Pattern 1: Linear DP (1D)

#### Fibonacci / Climbing Stairs

**Python:**
```python
def climb_stairs(n):
    """
    State: dp[i] = ways to reach step i
    Recurrence: dp[i] = dp[i-1] + dp[i-2]
    """
    if n <= 2:
        return n

    # Space optimized - O(1)
    prev2, prev1 = 1, 2
    for i in range(3, n + 1):
        curr = prev1 + prev2
        prev2, prev1 = prev1, curr

    return prev1
```

**JavaScript:**
```javascript
function climbStairs(n) {
  // State: dp[i] = ways to reach step i
  // Recurrence: dp[i] = dp[i-1] + dp[i-2]
  if (n <= 2) return n;

  // Space optimized - O(1)
  let prev2 = 1, prev1 = 2;
  for (let i = 3; i <= n; i++) {
    const curr = prev1 + prev2;
    prev2 = prev1;
    prev1 = curr;
  }

  return prev1;
}
```

#### House Robber

**Python:**
```python
def rob(nums):
    """
    State: dp[i] = max money robbing houses 0 to i
    Recurrence: dp[i] = max(dp[i-2] + nums[i], dp[i-1])
    """
    if not nums:
        return 0
    if len(nums) == 1:
        return nums[0]

    prev2, prev1 = nums[0], max(nums[0], nums[1])

    for i in range(2, len(nums)):
        curr = max(prev2 + nums[i], prev1)
        prev2, prev1 = prev1, curr

    return prev1

def rob_circular(nums):
    """House Robber II - houses in a circle"""
    if len(nums) == 1:
        return nums[0]

    def rob_linear(houses):
        prev2, prev1 = 0, 0
        for num in houses:
            curr = max(prev2 + num, prev1)
            prev2, prev1 = prev1, curr
        return prev1

    # Either rob house 0 to n-2, or house 1 to n-1
    return max(rob_linear(nums[:-1]), rob_linear(nums[1:]))
```

**JavaScript:**
```javascript
function rob(nums) {
  // State: dp[i] = max money robbing houses 0 to i
  // Recurrence: dp[i] = max(dp[i-2] + nums[i], dp[i-1])
  if (nums.length === 0) return 0;
  if (nums.length === 1) return nums[0];

  let prev2 = nums[0], prev1 = Math.max(nums[0], nums[1]);

  for (let i = 2; i < nums.length; i++) {
    const curr = Math.max(prev2 + nums[i], prev1);
    prev2 = prev1;
    prev1 = curr;
  }

  return prev1;
}

function robCircular(nums) {
  // House Robber II - houses in a circle
  if (nums.length === 1) return nums[0];

  function robLinear(houses) {
    let prev2 = 0, prev1 = 0;
    for (const num of houses) {
      const curr = Math.max(prev2 + num, prev1);
      prev2 = prev1;
      prev1 = curr;
    }
    return prev1;
  }

  // Either rob house 0 to n-2, or house 1 to n-1
  return Math.max(
    robLinear(nums.slice(0, -1)),
    robLinear(nums.slice(1))
  );
}
```

#### Maximum Subarray (Kadane's)

**Python:**
```python
def max_subarray(nums):
    """
    State: dp[i] = max subarray sum ending at i
    Recurrence: dp[i] = max(nums[i], dp[i-1] + nums[i])
    """
    max_sum = curr_sum = nums[0]

    for num in nums[1:]:
        curr_sum = max(num, curr_sum + num)
        max_sum = max(max_sum, curr_sum)

    return max_sum
```

**JavaScript:**
```javascript
function maxSubarray(nums) {
  // State: dp[i] = max subarray sum ending at i
  // Recurrence: dp[i] = max(nums[i], dp[i-1] + nums[i])
  let maxSum = nums[0], currSum = nums[0];

  for (let i = 1; i < nums.length; i++) {
    currSum = Math.max(nums[i], currSum + nums[i]);
    maxSum = Math.max(maxSum, currSum);
  }

  return maxSum;
}
```

#### Decode Ways

**Python:**
```python
def num_decodings(s):
    """
    State: dp[i] = ways to decode s[0:i]
    """
    if not s or s[0] == '0':
        return 0

    n = len(s)
    prev2, prev1 = 1, 1  # dp[0] = 1, dp[1] = 1

    for i in range(1, n):
        curr = 0

        # Single digit (1-9)
        if s[i] != '0':
            curr += prev1

        # Two digits (10-26)
        two_digit = int(s[i-1:i+1])
        if 10 <= two_digit <= 26:
            curr += prev2

        prev2, prev1 = prev1, curr

    return prev1
```

**JavaScript:**
```javascript
function numDecodings(s) {
  // State: dp[i] = ways to decode s[0:i]
  if (!s || s[0] === '0') return 0;

  const n = s.length;
  let prev2 = 1, prev1 = 1; // dp[0] = 1, dp[1] = 1

  for (let i = 1; i < n; i++) {
    let curr = 0;

    // Single digit (1-9)
    if (s[i] !== '0') {
      curr += prev1;
    }

    // Two digits (10-26)
    const twoDigit = parseInt(s.substring(i - 1, i + 1));
    if (twoDigit >= 10 && twoDigit <= 26) {
      curr += prev2;
    }

    prev2 = prev1;
    prev1 = curr;
  }

  return prev1;
}
```

---

### Pattern 2: Two Sequence DP (2D)

#### Longest Common Subsequence (LCS)

**Python:**
```python
def longest_common_subsequence(text1, text2):
    """
    State: dp[i][j] = LCS length of text1[0:i] and text2[0:j]
    Recurrence:
      - If text1[i-1] == text2[j-1]: dp[i][j] = dp[i-1][j-1] + 1
      - Else: dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    """
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i-1] == text2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])

    return dp[m][n]

def lcs_space_optimized(text1, text2):
    """O(min(m,n)) space"""
    if len(text1) < len(text2):
        text1, text2 = text2, text1

    prev = [0] * (len(text2) + 1)

    for i in range(1, len(text1) + 1):
        curr = [0] * (len(text2) + 1)
        for j in range(1, len(text2) + 1):
            if text1[i-1] == text2[j-1]:
                curr[j] = prev[j-1] + 1
            else:
                curr[j] = max(prev[j], curr[j-1])
        prev = curr

    return prev[-1]
```

**JavaScript:**
```javascript
function longestCommonSubsequence(text1, text2) {
  // State: dp[i][j] = LCS length of text1[0:i] and text2[0:j]
  const m = text1.length, n = text2.length;
  const dp = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (text1[i - 1] === text2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1] + 1;
      } else {
        dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
      }
    }
  }

  return dp[m][n];
}

function lcsSpaceOptimized(text1, text2) {
  // O(min(m,n)) space
  if (text1.length < text2.length) {
    [text1, text2] = [text2, text1];
  }

  let prev = new Array(text2.length + 1).fill(0);

  for (let i = 1; i <= text1.length; i++) {
    const curr = new Array(text2.length + 1).fill(0);
    for (let j = 1; j <= text2.length; j++) {
      if (text1[i - 1] === text2[j - 1]) {
        curr[j] = prev[j - 1] + 1;
      } else {
        curr[j] = Math.max(prev[j], curr[j - 1]);
      }
    }
    prev = curr;
  }

  return prev[prev.length - 1];
}
```

#### Edit Distance (Levenshtein)

**Python:**
```python
def min_distance(word1, word2):
    """
    State: dp[i][j] = min operations to convert word1[0:i] to word2[0:j]
    Operations: insert, delete, replace
    """
    m, n = len(word1), len(word2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    # Base cases
    for i in range(m + 1):
        dp[i][0] = i  # Delete all from word1
    for j in range(n + 1):
        dp[0][j] = j  # Insert all to word1

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i-1] == word2[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(
                    dp[i-1][j],      # Delete
                    dp[i][j-1],      # Insert
                    dp[i-1][j-1]     # Replace
                )

    return dp[m][n]
```

**JavaScript:**
```javascript
function minDistance(word1, word2) {
  // State: dp[i][j] = min operations to convert word1[0:i] to word2[0:j]
  // Operations: insert, delete, replace
  const m = word1.length, n = word2.length;
  const dp = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));

  // Base cases
  for (let i = 0; i <= m; i++) dp[i][0] = i; // Delete all from word1
  for (let j = 0; j <= n; j++) dp[0][j] = j; // Insert all to word1

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (word1[i - 1] === word2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1];
      } else {
        dp[i][j] = 1 + Math.min(
          dp[i - 1][j],     // Delete
          dp[i][j - 1],     // Insert
          dp[i - 1][j - 1]  // Replace
        );
      }
    }
  }

  return dp[m][n];
}
```

#### Longest Increasing Subsequence (LIS)

**Python:**
```python
def length_of_lis(nums):
    """
    O(n²) DP approach
    State: dp[i] = length of LIS ending at index i
    """
    n = len(nums)
    dp = [1] * n

    for i in range(1, n):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)

    return max(dp)

def length_of_lis_optimized(nums):
    """
    O(n log n) using binary search
    """
    from bisect import bisect_left

    tails = []

    for num in nums:
        pos = bisect_left(tails, num)
        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num

    return len(tails)
```

**JavaScript:**
```javascript
function lengthOfLIS(nums) {
  // O(n²) DP approach
  // State: dp[i] = length of LIS ending at index i
  const n = nums.length;
  const dp = new Array(n).fill(1);

  for (let i = 1; i < n; i++) {
    for (let j = 0; j < i; j++) {
      if (nums[j] < nums[i]) {
        dp[i] = Math.max(dp[i], dp[j] + 1);
      }
    }
  }

  return Math.max(...dp);
}

function lengthOfLISOptimized(nums) {
  // O(n log n) using binary search
  const tails = [];

  function bisectLeft(arr, target) {
    let left = 0, right = arr.length;
    while (left < right) {
      const mid = Math.floor((left + right) / 2);
      if (arr[mid] < target) left = mid + 1;
      else right = mid;
    }
    return left;
  }

  for (const num of nums) {
    const pos = bisectLeft(tails, num);
    if (pos === tails.length) {
      tails.push(num);
    } else {
      tails[pos] = num;
    }
  }

  return tails.length;
}
```

---

### Pattern 3: Knapsack Problems

#### 0/1 Knapsack

**Python:**
```python
def knapsack_01(weights, values, capacity):
    """
    Each item can be taken at most once
    State: dp[i][w] = max value using items 0 to i-1 with capacity w
    """
    n = len(weights)
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for w in range(capacity + 1):
            # Don't take item i-1
            dp[i][w] = dp[i-1][w]

            # Take item i-1 if possible
            if weights[i-1] <= w:
                dp[i][w] = max(dp[i][w],
                              dp[i-1][w - weights[i-1]] + values[i-1])

    return dp[n][capacity]

def knapsack_01_optimized(weights, values, capacity):
    """Space optimized - O(capacity)"""
    dp = [0] * (capacity + 1)

    for i in range(len(weights)):
        # Traverse backwards to avoid using same item twice
        for w in range(capacity, weights[i] - 1, -1):
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])

    return dp[capacity]
```

**JavaScript:**
```javascript
function knapsack01(weights, values, capacity) {
  // Each item can be taken at most once
  // State: dp[i][w] = max value using items 0 to i-1 with capacity w
  const n = weights.length;
  const dp = Array.from({ length: n + 1 }, () => new Array(capacity + 1).fill(0));

  for (let i = 1; i <= n; i++) {
    for (let w = 0; w <= capacity; w++) {
      // Don't take item i-1
      dp[i][w] = dp[i - 1][w];

      // Take item i-1 if possible
      if (weights[i - 1] <= w) {
        dp[i][w] = Math.max(dp[i][w],
          dp[i - 1][w - weights[i - 1]] + values[i - 1]);
      }
    }
  }

  return dp[n][capacity];
}

function knapsack01Optimized(weights, values, capacity) {
  // Space optimized - O(capacity)
  const dp = new Array(capacity + 1).fill(0);

  for (let i = 0; i < weights.length; i++) {
    // Traverse backwards to avoid using same item twice
    for (let w = capacity; w >= weights[i]; w--) {
      dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
    }
  }

  return dp[capacity];
}
```

#### Unbounded Knapsack (Coin Change)

**Python:**
```python
def coin_change(coins, amount):
    """
    Minimum coins needed
    State: dp[a] = min coins to make amount a
    """
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0

    for a in range(1, amount + 1):
        for coin in coins:
            if coin <= a and dp[a - coin] != float('inf'):
                dp[a] = min(dp[a], dp[a - coin] + 1)

    return dp[amount] if dp[amount] != float('inf') else -1

def coin_change_ways(coins, amount):
    """
    Number of ways to make amount (Coin Change 2)
    """
    dp = [0] * (amount + 1)
    dp[0] = 1

    for coin in coins:  # Iterate coins first to avoid counting permutations
        for a in range(coin, amount + 1):
            dp[a] += dp[a - coin]

    return dp[amount]
```

**JavaScript:**
```javascript
function coinChange(coins, amount) {
  // Minimum coins needed
  // State: dp[a] = min coins to make amount a
  const dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;

  for (let a = 1; a <= amount; a++) {
    for (const coin of coins) {
      if (coin <= a && dp[a - coin] !== Infinity) {
        dp[a] = Math.min(dp[a], dp[a - coin] + 1);
      }
    }
  }

  return dp[amount] !== Infinity ? dp[amount] : -1;
}

function coinChangeWays(coins, amount) {
  // Number of ways to make amount (Coin Change 2)
  const dp = new Array(amount + 1).fill(0);
  dp[0] = 1;

  for (const coin of coins) { // Iterate coins first to avoid counting permutations
    for (let a = coin; a <= amount; a++) {
      dp[a] += dp[a - coin];
    }
  }

  return dp[amount];
}
```

#### Subset Sum / Partition Equal Subset Sum

**Python:**
```python
def can_partition(nums):
    """
    Check if array can be partitioned into two equal sum subsets
    """
    total = sum(nums)
    if total % 2 != 0:
        return False

    target = total // 2
    dp = [False] * (target + 1)
    dp[0] = True

    for num in nums:
        for t in range(target, num - 1, -1):
            dp[t] = dp[t] or dp[t - num]

    return dp[target]

def count_subsets_with_sum(nums, target):
    """Count subsets that sum to target"""
    dp = [0] * (target + 1)
    dp[0] = 1

    for num in nums:
        for t in range(target, num - 1, -1):
            dp[t] += dp[t - num]

    return dp[target]
```

**JavaScript:**
```javascript
function canPartition(nums) {
  // Check if array can be partitioned into two equal sum subsets
  const total = nums.reduce((a, b) => a + b, 0);
  if (total % 2 !== 0) return false;

  const target = total / 2;
  const dp = new Array(target + 1).fill(false);
  dp[0] = true;

  for (const num of nums) {
    for (let t = target; t >= num; t--) {
      dp[t] = dp[t] || dp[t - num];
    }
  }

  return dp[target];
}

function countSubsetsWithSum(nums, target) {
  // Count subsets that sum to target
  const dp = new Array(target + 1).fill(0);
  dp[0] = 1;

  for (const num of nums) {
    for (let t = target; t >= num; t--) {
      dp[t] += dp[t - num];
    }
  }

  return dp[target];
}
```

---

### Pattern 4: String DP

#### Palindrome Problems

**Python:**
```python
def longest_palindromic_substring(s):
    """
    State: dp[i][j] = True if s[i:j+1] is palindrome
    """
    n = len(s)
    dp = [[False] * n for _ in range(n)]
    start, max_len = 0, 1

    # All single chars are palindromes
    for i in range(n):
        dp[i][i] = True

    # Check substrings of length 2 to n
    for length in range(2, n + 1):
        for i in range(n - length + 1):
            j = i + length - 1

            if length == 2:
                dp[i][j] = (s[i] == s[j])
            else:
                dp[i][j] = (s[i] == s[j] and dp[i+1][j-1])

            if dp[i][j] and length > max_len:
                start, max_len = i, length

    return s[start:start + max_len]

def count_palindromic_substrings(s):
    """Count all palindromic substrings"""
    n = len(s)
    count = 0

    # Expand around center approach - O(n²) time, O(1) space
    def expand(left, right):
        cnt = 0
        while left >= 0 and right < n and s[left] == s[right]:
            cnt += 1
            left -= 1
            right += 1
        return cnt

    for i in range(n):
        count += expand(i, i)      # Odd length
        count += expand(i, i + 1)  # Even length

    return count

def min_cuts_palindrome_partition(s):
    """Minimum cuts to partition into palindromes"""
    n = len(s)

    # Precompute palindrome table
    is_palindrome = [[False] * n for _ in range(n)]
    for i in range(n - 1, -1, -1):
        for j in range(i, n):
            if s[i] == s[j] and (j - i <= 2 or is_palindrome[i+1][j-1]):
                is_palindrome[i][j] = True

    # dp[i] = min cuts for s[0:i+1]
    dp = [float('inf')] * n
    for i in range(n):
        if is_palindrome[0][i]:
            dp[i] = 0
        else:
            for j in range(i):
                if is_palindrome[j+1][i]:
                    dp[i] = min(dp[i], dp[j] + 1)

    return dp[n-1]
```

**JavaScript:**
```javascript
function longestPalindromicSubstring(s) {
  // State: dp[i][j] = true if s[i:j+1] is palindrome
  const n = s.length;
  const dp = Array.from({ length: n }, () => new Array(n).fill(false));
  let start = 0, maxLen = 1;

  // All single chars are palindromes
  for (let i = 0; i < n; i++) {
    dp[i][i] = true;
  }

  // Check substrings of length 2 to n
  for (let length = 2; length <= n; length++) {
    for (let i = 0; i <= n - length; i++) {
      const j = i + length - 1;

      if (length === 2) {
        dp[i][j] = s[i] === s[j];
      } else {
        dp[i][j] = s[i] === s[j] && dp[i + 1][j - 1];
      }

      if (dp[i][j] && length > maxLen) {
        start = i;
        maxLen = length;
      }
    }
  }

  return s.substring(start, start + maxLen);
}

function countPalindromicSubstrings(s) {
  // Count all palindromic substrings
  const n = s.length;
  let count = 0;

  // Expand around center approach - O(n²) time, O(1) space
  function expand(left, right) {
    let cnt = 0;
    while (left >= 0 && right < n && s[left] === s[right]) {
      cnt++;
      left--;
      right++;
    }
    return cnt;
  }

  for (let i = 0; i < n; i++) {
    count += expand(i, i);     // Odd length
    count += expand(i, i + 1); // Even length
  }

  return count;
}

function minCutsPalindromePartition(s) {
  // Minimum cuts to partition into palindromes
  const n = s.length;

  // Precompute palindrome table
  const isPalindrome = Array.from({ length: n }, () => new Array(n).fill(false));
  for (let i = n - 1; i >= 0; i--) {
    for (let j = i; j < n; j++) {
      if (s[i] === s[j] && (j - i <= 2 || isPalindrome[i + 1][j - 1])) {
        isPalindrome[i][j] = true;
      }
    }
  }

  // dp[i] = min cuts for s[0:i+1]
  const dp = new Array(n).fill(Infinity);
  for (let i = 0; i < n; i++) {
    if (isPalindrome[0][i]) {
      dp[i] = 0;
    } else {
      for (let j = 0; j < i; j++) {
        if (isPalindrome[j + 1][i]) {
          dp[i] = Math.min(dp[i], dp[j] + 1);
        }
      }
    }
  }

  return dp[n - 1];
}
```

#### Word Break

**Python:**
```python
def word_break(s, word_dict):
    """
    Can string be segmented into dictionary words?
    State: dp[i] = True if s[0:i] can be segmented
    """
    word_set = set(word_dict)
    n = len(s)
    dp = [False] * (n + 1)
    dp[0] = True

    for i in range(1, n + 1):
        for j in range(i):
            if dp[j] and s[j:i] in word_set:
                dp[i] = True
                break

    return dp[n]

def word_break_ii(s, word_dict):
    """Return all possible sentences"""
    word_set = set(word_dict)
    memo = {}

    def backtrack(start):
        if start in memo:
            return memo[start]

        if start == len(s):
            return [""]

        sentences = []
        for end in range(start + 1, len(s) + 1):
            word = s[start:end]
            if word in word_set:
                for rest in backtrack(end):
                    if rest:
                        sentences.append(word + " " + rest)
                    else:
                        sentences.append(word)

        memo[start] = sentences
        return sentences

    return backtrack(0)
```

**JavaScript:**
```javascript
function wordBreak(s, wordDict) {
  // Can string be segmented into dictionary words?
  // State: dp[i] = true if s[0:i] can be segmented
  const wordSet = new Set(wordDict);
  const n = s.length;
  const dp = new Array(n + 1).fill(false);
  dp[0] = true;

  for (let i = 1; i <= n; i++) {
    for (let j = 0; j < i; j++) {
      if (dp[j] && wordSet.has(s.substring(j, i))) {
        dp[i] = true;
        break;
      }
    }
  }

  return dp[n];
}

function wordBreakII(s, wordDict) {
  // Return all possible sentences
  const wordSet = new Set(wordDict);
  const memo = new Map();

  function backtrack(start) {
    if (memo.has(start)) {
      return memo.get(start);
    }

    if (start === s.length) {
      return [""];
    }

    const sentences = [];
    for (let end = start + 1; end <= s.length; end++) {
      const word = s.substring(start, end);
      if (wordSet.has(word)) {
        for (const rest of backtrack(end)) {
          if (rest) {
            sentences.push(word + " " + rest);
          } else {
            sentences.push(word);
          }
        }
      }
    }

    memo.set(start, sentences);
    return sentences;
  }

  return backtrack(0);
}
```

#### Regular Expression Matching

**Python:**
```python
def is_match(s, p):
    """
    '.' matches any single character
    '*' matches zero or more of the preceding element
    """
    m, n = len(s), len(p)
    dp = [[False] * (n + 1) for _ in range(m + 1)]
    dp[0][0] = True

    # Handle patterns like a*, a*b*, etc. that can match empty string
    for j in range(2, n + 1):
        if p[j-1] == '*':
            dp[0][j] = dp[0][j-2]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if p[j-1] == '*':
                # Zero occurrences of preceding char
                dp[i][j] = dp[i][j-2]
                # One or more occurrences
                if p[j-2] == '.' or p[j-2] == s[i-1]:
                    dp[i][j] = dp[i][j] or dp[i-1][j]
            elif p[j-1] == '.' or p[j-1] == s[i-1]:
                dp[i][j] = dp[i-1][j-1]

    return dp[m][n]
```

**JavaScript:**
```javascript
function isMatch(s, p) {
  // '.' matches any single character
  // '*' matches zero or more of the preceding element
  const m = s.length, n = p.length;
  const dp = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(false));
  dp[0][0] = true;

  // Handle patterns like a*, a*b*, etc. that can match empty string
  for (let j = 2; j <= n; j++) {
    if (p[j - 1] === '*') {
      dp[0][j] = dp[0][j - 2];
    }
  }

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (p[j - 1] === '*') {
        // Zero occurrences of preceding char
        dp[i][j] = dp[i][j - 2];
        // One or more occurrences
        if (p[j - 2] === '.' || p[j - 2] === s[i - 1]) {
          dp[i][j] = dp[i][j] || dp[i - 1][j];
        }
      } else if (p[j - 1] === '.' || p[j - 1] === s[i - 1]) {
        dp[i][j] = dp[i - 1][j - 1];
      }
    }
  }

  return dp[m][n];
}
```

---

### Pattern 5: Grid DP

#### Unique Paths

**Python:**
```python
def unique_paths(m, n):
    """Count paths from top-left to bottom-right"""
    dp = [[1] * n for _ in range(m)]

    for i in range(1, m):
        for j in range(1, n):
            dp[i][j] = dp[i-1][j] + dp[i][j-1]

    return dp[m-1][n-1]

def unique_paths_with_obstacles(grid):
    """With obstacles (1 = obstacle)"""
    m, n = len(grid), len(grid[0])
    if grid[0][0] == 1:
        return 0

    dp = [[0] * n for _ in range(m)]
    dp[0][0] = 1

    for i in range(m):
        for j in range(n):
            if grid[i][j] == 1:
                dp[i][j] = 0
            else:
                if i > 0:
                    dp[i][j] += dp[i-1][j]
                if j > 0:
                    dp[i][j] += dp[i][j-1]

    return dp[m-1][n-1]
```

**JavaScript:**
```javascript
function uniquePaths(m, n) {
  // Count paths from top-left to bottom-right
  const dp = Array.from({ length: m }, () => new Array(n).fill(1));

  for (let i = 1; i < m; i++) {
    for (let j = 1; j < n; j++) {
      dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
    }
  }

  return dp[m - 1][n - 1];
}

function uniquePathsWithObstacles(grid) {
  // With obstacles (1 = obstacle)
  const m = grid.length, n = grid[0].length;
  if (grid[0][0] === 1) return 0;

  const dp = Array.from({ length: m }, () => new Array(n).fill(0));
  dp[0][0] = 1;

  for (let i = 0; i < m; i++) {
    for (let j = 0; j < n; j++) {
      if (grid[i][j] === 1) {
        dp[i][j] = 0;
      } else {
        if (i > 0) dp[i][j] += dp[i - 1][j];
        if (j > 0) dp[i][j] += dp[i][j - 1];
      }
    }
  }

  return dp[m - 1][n - 1];
}
```

#### Minimum Path Sum

**Python:**
```python
def min_path_sum(grid):
    m, n = len(grid), len(grid[0])

    for i in range(m):
        for j in range(n):
            if i == 0 and j == 0:
                continue
            elif i == 0:
                grid[i][j] += grid[i][j-1]
            elif j == 0:
                grid[i][j] += grid[i-1][j]
            else:
                grid[i][j] += min(grid[i-1][j], grid[i][j-1])

    return grid[m-1][n-1]
```

**JavaScript:**
```javascript
function minPathSum(grid) {
  const m = grid.length, n = grid[0].length;

  for (let i = 0; i < m; i++) {
    for (let j = 0; j < n; j++) {
      if (i === 0 && j === 0) {
        continue;
      } else if (i === 0) {
        grid[i][j] += grid[i][j - 1];
      } else if (j === 0) {
        grid[i][j] += grid[i - 1][j];
      } else {
        grid[i][j] += Math.min(grid[i - 1][j], grid[i][j - 1]);
      }
    }
  }

  return grid[m - 1][n - 1];
}
```

#### Maximal Square

**Python:**
```python
def maximal_square(matrix):
    """Find largest square of 1s"""
    if not matrix:
        return 0

    m, n = len(matrix), len(matrix[0])
    dp = [[0] * n for _ in range(m)]
    max_side = 0

    for i in range(m):
        for j in range(n):
            if matrix[i][j] == '1':
                if i == 0 or j == 0:
                    dp[i][j] = 1
                else:
                    dp[i][j] = min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]) + 1
                max_side = max(max_side, dp[i][j])

    return max_side * max_side
```

**JavaScript:**
```javascript
function maximalSquare(matrix) {
  // Find largest square of 1s
  if (!matrix || matrix.length === 0) return 0;

  const m = matrix.length, n = matrix[0].length;
  const dp = Array.from({ length: m }, () => new Array(n).fill(0));
  let maxSide = 0;

  for (let i = 0; i < m; i++) {
    for (let j = 0; j < n; j++) {
      if (matrix[i][j] === '1') {
        if (i === 0 || j === 0) {
          dp[i][j] = 1;
        } else {
          dp[i][j] = Math.min(dp[i - 1][j], dp[i][j - 1], dp[i - 1][j - 1]) + 1;
        }
        maxSide = Math.max(maxSide, dp[i][j]);
      }
    }
  }

  return maxSide * maxSide;
}
```

---

### Pattern 6: Interval DP

#### Matrix Chain Multiplication

**Python:**
```python
def matrix_chain_multiplication(dims):
    """
    dims[i] represents dimensions: matrix i is dims[i-1] x dims[i]
    Find minimum operations to multiply all matrices
    """
    n = len(dims) - 1  # Number of matrices
    dp = [[0] * n for _ in range(n)]

    # length = chain length
    for length in range(2, n + 1):
        for i in range(n - length + 1):
            j = i + length - 1
            dp[i][j] = float('inf')

            for k in range(i, j):
                cost = (dp[i][k] + dp[k+1][j] +
                       dims[i] * dims[k+1] * dims[j+1])
                dp[i][j] = min(dp[i][j], cost)

    return dp[0][n-1]
```

**JavaScript:**
```javascript
function matrixChainMultiplication(dims) {
  // dims[i] represents dimensions: matrix i is dims[i-1] x dims[i]
  // Find minimum operations to multiply all matrices
  const n = dims.length - 1; // Number of matrices
  const dp = Array.from({ length: n }, () => new Array(n).fill(0));

  // length = chain length
  for (let length = 2; length <= n; length++) {
    for (let i = 0; i <= n - length; i++) {
      const j = i + length - 1;
      dp[i][j] = Infinity;

      for (let k = i; k < j; k++) {
        const cost = dp[i][k] + dp[k + 1][j] +
          dims[i] * dims[k + 1] * dims[j + 1];
        dp[i][j] = Math.min(dp[i][j], cost);
      }
    }
  }

  return dp[0][n - 1];
}
```

#### Burst Balloons

**Python:**
```python
def max_coins(nums):
    """Burst balloons to maximize coins"""
    nums = [1] + nums + [1]
    n = len(nums)
    dp = [[0] * n for _ in range(n)]

    for length in range(2, n):
        for left in range(n - length):
            right = left + length

            for i in range(left + 1, right):
                coins = (nums[left] * nums[i] * nums[right] +
                        dp[left][i] + dp[i][right])
                dp[left][right] = max(dp[left][right], coins)

    return dp[0][n-1]
```

**JavaScript:**
```javascript
function maxCoins(nums) {
  // Burst balloons to maximize coins
  nums = [1, ...nums, 1];
  const n = nums.length;
  const dp = Array.from({ length: n }, () => new Array(n).fill(0));

  for (let length = 2; length < n; length++) {
    for (let left = 0; left < n - length; left++) {
      const right = left + length;

      for (let i = left + 1; i < right; i++) {
        const coins = nums[left] * nums[i] * nums[right] +
          dp[left][i] + dp[i][right];
        dp[left][right] = Math.max(dp[left][right], coins);
      }
    }
  }

  return dp[0][n - 1];
}
```

---

### Pattern 7: State Machine DP

#### Best Time to Buy and Sell Stock with Cooldown

**Python:**
```python
def max_profit_with_cooldown(prices):
    """
    States: hold, sold, rest
    """
    if len(prices) < 2:
        return 0

    hold = -prices[0]  # Holding stock
    sold = 0           # Just sold
    rest = 0           # Cooldown/not holding

    for price in prices[1:]:
        prev_hold = hold
        hold = max(hold, rest - price)
        rest = max(rest, sold)
        sold = prev_hold + price

    return max(sold, rest)
```

**JavaScript:**
```javascript
function maxProfitWithCooldown(prices) {
  // States: hold, sold, rest
  if (prices.length < 2) return 0;

  let hold = -prices[0]; // Holding stock
  let sold = 0;          // Just sold
  let rest = 0;          // Cooldown/not holding

  for (let i = 1; i < prices.length; i++) {
    const prevHold = hold;
    hold = Math.max(hold, rest - prices[i]);
    rest = Math.max(rest, sold);
    sold = prevHold + prices[i];
  }

  return Math.max(sold, rest);
}
```

#### Best Time to Buy and Sell Stock with K Transactions

**Python:**
```python
def max_profit_k_transactions(k, prices):
    if not prices:
        return 0

    n = len(prices)

    # Optimization for large k
    if k >= n // 2:
        return sum(max(0, prices[i] - prices[i-1]) for i in range(1, n))

    # dp[t][d] = max profit with at most t transactions up to day d
    dp = [[0] * n for _ in range(k + 1)]

    for t in range(1, k + 1):
        max_diff = -prices[0]
        for d in range(1, n):
            dp[t][d] = max(dp[t][d-1], prices[d] + max_diff)
            max_diff = max(max_diff, dp[t-1][d] - prices[d])

    return dp[k][n-1]
```

**JavaScript:**
```javascript
function maxProfitKTransactions(k, prices) {
  if (!prices || prices.length === 0) return 0;

  const n = prices.length;

  // Optimization for large k
  if (k >= Math.floor(n / 2)) {
    let profit = 0;
    for (let i = 1; i < n; i++) {
      profit += Math.max(0, prices[i] - prices[i - 1]);
    }
    return profit;
  }

  // dp[t][d] = max profit with at most t transactions up to day d
  const dp = Array.from({ length: k + 1 }, () => new Array(n).fill(0));

  for (let t = 1; t <= k; t++) {
    let maxDiff = -prices[0];
    for (let d = 1; d < n; d++) {
      dp[t][d] = Math.max(dp[t][d - 1], prices[d] + maxDiff);
      maxDiff = Math.max(maxDiff, dp[t - 1][d] - prices[d]);
    }
  }

  return dp[k][n - 1];
}
```

---

## Must-Know Problems

### Easy
1. Climbing Stairs
2. House Robber
3. Maximum Subarray
4. Best Time to Buy and Sell Stock

### Medium
1. Coin Change
2. Longest Increasing Subsequence
3. Longest Common Subsequence
4. Word Break
5. Unique Paths
6. House Robber II
7. Partition Equal Subset Sum
8. Decode Ways
9. Longest Palindromic Substring
10. Jump Game I & II

### Hard
1. Edit Distance
2. Regular Expression Matching
3. Wildcard Matching
4. Burst Balloons
5. Palindrome Partitioning II
6. Distinct Subsequences
7. Interleaving String
8. Best Time to Buy/Sell Stock IV

---

## Practice Checklist

- [ ] Can identify DP problems quickly
- [ ] Know when to use 1D vs 2D DP
- [ ] Understand memoization vs tabulation trade-offs
- [ ] Can optimize space in common patterns
- [ ] Can explain state transitions clearly
