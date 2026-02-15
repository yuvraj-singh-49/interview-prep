# Recursion and Backtracking

## Overview

Recursion and backtracking are fundamental techniques for solving combinatorial problems. As a Lead engineer, you should be able to identify when these patterns apply, implement them efficiently, and analyze their complexity.

---

## Recursion Fundamentals

### Anatomy of Recursion

**Python:**
```python
def recursive_function(parameters):
    # 1. Base Case - stops recursion
    if base_condition:
        return base_value

    # 2. Recursive Case - break down problem
    # 3. Process current level
    # 4. Combine results

    return recursive_function(smaller_problem)
```

**JavaScript:**
```javascript
function recursiveFunction(parameters) {
  // 1. Base Case - stops recursion
  if (baseCondition) {
    return baseValue;
  }

  // 2. Recursive Case - break down problem
  // 3. Process current level
  // 4. Combine results

  return recursiveFunction(smallerProblem);
}
```

### Recursion vs Iteration

| Aspect | Recursion | Iteration |
|--------|-----------|-----------|
| Readability | Often cleaner | Can be verbose |
| Space | O(n) stack | O(1) typically |
| Performance | Function call overhead | Usually faster |
| Stack overflow | Risk with deep recursion | No risk |

### Tail Recursion

**Python:**
```python
# Not tail recursive - multiplication after recursive call
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)

# Tail recursive - result built in accumulator
def factorial_tail(n, acc=1):
    if n <= 1:
        return acc
    return factorial_tail(n - 1, n * acc)

# Note: Python doesn't optimize tail recursion, but concept is important
```

**JavaScript:**
```javascript
// Not tail recursive - multiplication after recursive call
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);
}

// Tail recursive - result built in accumulator
function factorialTail(n, acc = 1) {
  if (n <= 1) return acc;
  return factorialTail(n - 1, n * acc);
}

// Note: JS engines may optimize tail calls in strict mode
```

---

## Backtracking Framework

### Template

**Python:**
```python
def backtrack(state, choices, result):
    # Base case: valid solution found
    if is_valid_solution(state):
        result.append(state.copy())  # Save solution
        return

    for choice in choices:
        # Constraint check (pruning)
        if is_valid_choice(choice, state):
            # Make choice
            state.append(choice)  # or modify state

            # Recurse
            backtrack(state, updated_choices, result)

            # Undo choice (backtrack)
            state.pop()  # or restore state
```

**JavaScript:**
```javascript
function backtrack(state, choices, result) {
  // Base case: valid solution found
  if (isValidSolution(state)) {
    result.push([...state]); // Save solution
    return;
  }

  for (const choice of choices) {
    // Constraint check (pruning)
    if (isValidChoice(choice, state)) {
      // Make choice
      state.push(choice); // or modify state

      // Recurse
      backtrack(state, updatedChoices, result);

      // Undo choice (backtrack)
      state.pop(); // or restore state
    }
  }
}
```

### Key Concepts

1. **State Space Tree**: All possible states form a tree
2. **Pruning**: Eliminate invalid branches early
3. **State Management**: Track and restore state properly

---

## Classic Recursion Problems

### Power Function

**Python:**
```python
def power(x, n):
    """Calculate x^n in O(log n)"""
    if n == 0:
        return 1
    if n < 0:
        return 1 / power(x, -n)

    if n % 2 == 0:
        half = power(x, n // 2)
        return half * half
    else:
        return x * power(x, n - 1)
```

**JavaScript:**
```javascript
function power(x, n) {
  // Calculate x^n in O(log n)
  if (n === 0) return 1;
  if (n < 0) return 1 / power(x, -n);

  if (n % 2 === 0) {
    const half = power(x, Math.floor(n / 2));
    return half * half;
  } else {
    return x * power(x, n - 1);
  }
}
```

### Tower of Hanoi

**Python:**
```python
def hanoi(n, source, auxiliary, target):
    """
    Move n disks from source to target using auxiliary
    Time: O(2^n)
    """
    if n == 1:
        print(f"Move disk 1 from {source} to {target}")
        return

    hanoi(n - 1, source, target, auxiliary)
    print(f"Move disk {n} from {source} to {target}")
    hanoi(n - 1, auxiliary, source, target)
```

**JavaScript:**
```javascript
function hanoi(n, source, auxiliary, target) {
  // Move n disks from source to target using auxiliary
  // Time: O(2^n)
  if (n === 1) {
    console.log(`Move disk 1 from ${source} to ${target}`);
    return;
  }

  hanoi(n - 1, source, target, auxiliary);
  console.log(`Move disk ${n} from ${source} to ${target}`);
  hanoi(n - 1, auxiliary, source, target);
}
```

### Generate All Parentheses

**Python:**
```python
def generate_parentheses(n):
    result = []

    def backtrack(current, open_count, close_count):
        if len(current) == 2 * n:
            result.append(current)
            return

        if open_count < n:
            backtrack(current + '(', open_count + 1, close_count)
        if close_count < open_count:
            backtrack(current + ')', open_count, close_count + 1)

    backtrack('', 0, 0)
    return result
```

**JavaScript:**
```javascript
function generateParentheses(n) {
  const result = [];

  function backtrack(current, openCount, closeCount) {
    if (current.length === 2 * n) {
      result.push(current);
      return;
    }

    if (openCount < n) {
      backtrack(current + '(', openCount + 1, closeCount);
    }
    if (closeCount < openCount) {
      backtrack(current + ')', openCount, closeCount + 1);
    }
  }

  backtrack('', 0, 0);
  return result;
}
```

---

## Permutation Problems

### All Permutations

**Python:**
```python
def permutations(nums):
    """Generate all permutations"""
    result = []

    def backtrack(start):
        if start == len(nums):
            result.append(nums[:])
            return

        for i in range(start, len(nums)):
            nums[start], nums[i] = nums[i], nums[start]
            backtrack(start + 1)
            nums[start], nums[i] = nums[i], nums[start]

    backtrack(0)
    return result

def permutations_with_duplicates(nums):
    """Handle duplicates"""
    result = []
    nums.sort()

    def backtrack(path, used):
        if len(path) == len(nums):
            result.append(path[:])
            return

        for i in range(len(nums)):
            if used[i]:
                continue
            # Skip duplicates
            if i > 0 and nums[i] == nums[i-1] and not used[i-1]:
                continue

            used[i] = True
            path.append(nums[i])
            backtrack(path, used)
            path.pop()
            used[i] = False

    backtrack([], [False] * len(nums))
    return result
```

**JavaScript:**
```javascript
function permutations(nums) {
  // Generate all permutations
  const result = [];

  function backtrack(start) {
    if (start === nums.length) {
      result.push([...nums]);
      return;
    }

    for (let i = start; i < nums.length; i++) {
      [nums[start], nums[i]] = [nums[i], nums[start]];
      backtrack(start + 1);
      [nums[start], nums[i]] = [nums[i], nums[start]];
    }
  }

  backtrack(0);
  return result;
}

function permutationsWithDuplicates(nums) {
  // Handle duplicates
  const result = [];
  nums.sort((a, b) => a - b);

  function backtrack(path, used) {
    if (path.length === nums.length) {
      result.push([...path]);
      return;
    }

    for (let i = 0; i < nums.length; i++) {
      if (used[i]) continue;
      // Skip duplicates
      if (i > 0 && nums[i] === nums[i - 1] && !used[i - 1]) continue;

      used[i] = true;
      path.push(nums[i]);
      backtrack(path, used);
      path.pop();
      used[i] = false;
    }
  }

  backtrack([], new Array(nums.length).fill(false));
  return result;
}
```

### Next Permutation

**Python:**
```python
def next_permutation(nums):
    """
    Modify array to next lexicographically greater permutation
    1. Find first decreasing element from right
    2. Swap with smallest greater element to its right
    3. Reverse suffix
    """
    n = len(nums)

    # Step 1: Find first decreasing
    i = n - 2
    while i >= 0 and nums[i] >= nums[i + 1]:
        i -= 1

    if i >= 0:
        # Step 2: Find smallest greater
        j = n - 1
        while nums[j] <= nums[i]:
            j -= 1
        nums[i], nums[j] = nums[j], nums[i]

    # Step 3: Reverse suffix
    left, right = i + 1, n - 1
    while left < right:
        nums[left], nums[right] = nums[right], nums[left]
        left += 1
        right -= 1
```

**JavaScript:**
```javascript
function nextPermutation(nums) {
  // Modify array to next lexicographically greater permutation
  // 1. Find first decreasing element from right
  // 2. Swap with smallest greater element to its right
  // 3. Reverse suffix
  const n = nums.length;

  // Step 1: Find first decreasing
  let i = n - 2;
  while (i >= 0 && nums[i] >= nums[i + 1]) {
    i--;
  }

  if (i >= 0) {
    // Step 2: Find smallest greater
    let j = n - 1;
    while (nums[j] <= nums[i]) {
      j--;
    }
    [nums[i], nums[j]] = [nums[j], nums[i]];
  }

  // Step 3: Reverse suffix
  let left = i + 1, right = n - 1;
  while (left < right) {
    [nums[left], nums[right]] = [nums[right], nums[left]];
    left++;
    right--;
  }
}
```

### Permutation Sequence (Kth Permutation)

**Python:**
```python
def get_permutation(n, k):
    """Get kth permutation of 1 to n"""
    import math

    numbers = list(range(1, n + 1))
    k -= 1  # Convert to 0-indexed
    result = []

    for i in range(n, 0, -1):
        factorial = math.factorial(i - 1)
        index = k // factorial
        result.append(str(numbers[index]))
        numbers.pop(index)
        k %= factorial

    return ''.join(result)
```

**JavaScript:**
```javascript
function getPermutation(n, k) {
  // Get kth permutation of 1 to n
  function factorial(num) {
    let result = 1;
    for (let i = 2; i <= num; i++) result *= i;
    return result;
  }

  const numbers = Array.from({ length: n }, (_, i) => i + 1);
  k--; // Convert to 0-indexed
  let result = '';

  for (let i = n; i > 0; i--) {
    const fact = factorial(i - 1);
    const index = Math.floor(k / fact);
    result += numbers[index];
    numbers.splice(index, 1);
    k %= fact;
  }

  return result;
}
```

---

## Combination Problems

### All Combinations (n choose k)

**Python:**
```python
def combinations(n, k):
    result = []

    def backtrack(start, path):
        if len(path) == k:
            result.append(path[:])
            return

        # Pruning: need k - len(path) more elements
        remaining = k - len(path)
        for i in range(start, n - remaining + 2):
            path.append(i)
            backtrack(i + 1, path)
            path.pop()

    backtrack(1, [])
    return result
```

**JavaScript:**
```javascript
function combinations(n, k) {
  const result = [];

  function backtrack(start, path) {
    if (path.length === k) {
      result.push([...path]);
      return;
    }

    // Pruning: need k - path.length more elements
    const remaining = k - path.length;
    for (let i = start; i <= n - remaining + 1; i++) {
      path.push(i);
      backtrack(i + 1, path);
      path.pop();
    }
  }

  backtrack(1, []);
  return result;
}
```

### Combination Sum (Can Reuse Elements)

**Python:**
```python
def combination_sum(candidates, target):
    """Elements can be used multiple times"""
    result = []
    candidates.sort()

    def backtrack(start, target, path):
        if target == 0:
            result.append(path[:])
            return

        for i in range(start, len(candidates)):
            if candidates[i] > target:
                break  # Pruning

            path.append(candidates[i])
            backtrack(i, target - candidates[i], path)  # i, not i+1
            path.pop()

    backtrack(0, target, [])
    return result
```

**JavaScript:**
```javascript
function combinationSum(candidates, target) {
  // Elements can be used multiple times
  const result = [];
  candidates.sort((a, b) => a - b);

  function backtrack(start, target, path) {
    if (target === 0) {
      result.push([...path]);
      return;
    }

    for (let i = start; i < candidates.length; i++) {
      if (candidates[i] > target) break; // Pruning

      path.push(candidates[i]);
      backtrack(i, target - candidates[i], path); // i, not i+1
      path.pop();
    }
  }

  backtrack(0, target, []);
  return result;
}
```

### Combination Sum II (Each Element Once)

**Python:**
```python
def combination_sum2(candidates, target):
    """Each element used at most once, avoid duplicates"""
    result = []
    candidates.sort()

    def backtrack(start, target, path):
        if target == 0:
            result.append(path[:])
            return

        for i in range(start, len(candidates)):
            # Skip duplicates at same level
            if i > start and candidates[i] == candidates[i-1]:
                continue

            if candidates[i] > target:
                break

            path.append(candidates[i])
            backtrack(i + 1, target - candidates[i], path)
            path.pop()

    backtrack(0, target, [])
    return result
```

**JavaScript:**
```javascript
function combinationSum2(candidates, target) {
  // Each element used at most once, avoid duplicates
  const result = [];
  candidates.sort((a, b) => a - b);

  function backtrack(start, target, path) {
    if (target === 0) {
      result.push([...path]);
      return;
    }

    for (let i = start; i < candidates.length; i++) {
      // Skip duplicates at same level
      if (i > start && candidates[i] === candidates[i - 1]) continue;

      if (candidates[i] > target) break;

      path.push(candidates[i]);
      backtrack(i + 1, target - candidates[i], path);
      path.pop();
    }
  }

  backtrack(0, target, []);
  return result;
}
```

---

## Subset Problems

### All Subsets (Power Set)

**Python:**
```python
def subsets(nums):
    """Generate all 2^n subsets"""
    result = []

    def backtrack(start, path):
        result.append(path[:])

        for i in range(start, len(nums)):
            path.append(nums[i])
            backtrack(i + 1, path)
            path.pop()

    backtrack(0, [])
    return result

def subsets_iterative(nums):
    """Iterative approach"""
    result = [[]]

    for num in nums:
        result += [subset + [num] for subset in result]

    return result

def subsets_bitmask(nums):
    """Using bit manipulation"""
    n = len(nums)
    result = []

    for mask in range(1 << n):
        subset = []
        for i in range(n):
            if mask & (1 << i):
                subset.append(nums[i])
        result.append(subset)

    return result
```

**JavaScript:**
```javascript
function subsets(nums) {
  // Generate all 2^n subsets
  const result = [];

  function backtrack(start, path) {
    result.push([...path]);

    for (let i = start; i < nums.length; i++) {
      path.push(nums[i]);
      backtrack(i + 1, path);
      path.pop();
    }
  }

  backtrack(0, []);
  return result;
}

function subsetsIterative(nums) {
  // Iterative approach
  let result = [[]];

  for (const num of nums) {
    result = [...result, ...result.map(subset => [...subset, num])];
  }

  return result;
}

function subsetsBitmask(nums) {
  // Using bit manipulation
  const n = nums.length;
  const result = [];

  for (let mask = 0; mask < (1 << n); mask++) {
    const subset = [];
    for (let i = 0; i < n; i++) {
      if (mask & (1 << i)) {
        subset.push(nums[i]);
      }
    }
    result.push(subset);
  }

  return result;
}
```

### Subsets with Duplicates

**Python:**
```python
def subsets_with_dup(nums):
    result = []
    nums.sort()

    def backtrack(start, path):
        result.append(path[:])

        for i in range(start, len(nums)):
            # Skip duplicates at same level
            if i > start and nums[i] == nums[i-1]:
                continue

            path.append(nums[i])
            backtrack(i + 1, path)
            path.pop()

    backtrack(0, [])
    return result
```

**JavaScript:**
```javascript
function subsetsWithDup(nums) {
  const result = [];
  nums.sort((a, b) => a - b);

  function backtrack(start, path) {
    result.push([...path]);

    for (let i = start; i < nums.length; i++) {
      // Skip duplicates at same level
      if (i > start && nums[i] === nums[i - 1]) continue;

      path.push(nums[i]);
      backtrack(i + 1, path);
      path.pop();
    }
  }

  backtrack(0, []);
  return result;
}
```

---

## Classic Backtracking Problems

### N-Queens

**Python:**
```python
def solve_n_queens(n):
    result = []
    board = [['.'] * n for _ in range(n)]

    # Track attacked columns and diagonals
    cols = set()
    diag1 = set()  # row - col
    diag2 = set()  # row + col

    def backtrack(row):
        if row == n:
            result.append([''.join(r) for r in board])
            return

        for col in range(n):
            if col in cols or (row - col) in diag1 or (row + col) in diag2:
                continue

            # Place queen
            board[row][col] = 'Q'
            cols.add(col)
            diag1.add(row - col)
            diag2.add(row + col)

            backtrack(row + 1)

            # Remove queen
            board[row][col] = '.'
            cols.remove(col)
            diag1.remove(row - col)
            diag2.remove(row + col)

    backtrack(0)
    return result
```

**JavaScript:**
```javascript
function solveNQueens(n) {
  const result = [];
  const board = Array.from({ length: n }, () => new Array(n).fill('.'));

  // Track attacked columns and diagonals
  const cols = new Set();
  const diag1 = new Set(); // row - col
  const diag2 = new Set(); // row + col

  function backtrack(row) {
    if (row === n) {
      result.push(board.map(r => r.join('')));
      return;
    }

    for (let col = 0; col < n; col++) {
      if (cols.has(col) || diag1.has(row - col) || diag2.has(row + col)) {
        continue;
      }

      // Place queen
      board[row][col] = 'Q';
      cols.add(col);
      diag1.add(row - col);
      diag2.add(row + col);

      backtrack(row + 1);

      // Remove queen
      board[row][col] = '.';
      cols.delete(col);
      diag1.delete(row - col);
      diag2.delete(row + col);
    }
  }

  backtrack(0);
  return result;
}
```

### Sudoku Solver

**Python:**
```python
def solve_sudoku(board):
    def is_valid(row, col, num):
        # Check row
        if num in board[row]:
            return False

        # Check column
        if any(board[r][col] == num for r in range(9)):
            return False

        # Check 3x3 box
        box_row, box_col = 3 * (row // 3), 3 * (col // 3)
        for r in range(box_row, box_row + 3):
            for c in range(box_col, box_col + 3):
                if board[r][c] == num:
                    return False

        return True

    def solve():
        for row in range(9):
            for col in range(9):
                if board[row][col] == '.':
                    for num in '123456789':
                        if is_valid(row, col, num):
                            board[row][col] = num
                            if solve():
                                return True
                            board[row][col] = '.'
                    return False
        return True

    solve()
```

**JavaScript:**
```javascript
function solveSudoku(board) {
  function isValid(row, col, num) {
    // Check row
    if (board[row].includes(num)) return false;

    // Check column
    for (let r = 0; r < 9; r++) {
      if (board[r][col] === num) return false;
    }

    // Check 3x3 box
    const boxRow = 3 * Math.floor(row / 3);
    const boxCol = 3 * Math.floor(col / 3);
    for (let r = boxRow; r < boxRow + 3; r++) {
      for (let c = boxCol; c < boxCol + 3; c++) {
        if (board[r][c] === num) return false;
      }
    }

    return true;
  }

  function solve() {
    for (let row = 0; row < 9; row++) {
      for (let col = 0; col < 9; col++) {
        if (board[row][col] === '.') {
          for (let num = 1; num <= 9; num++) {
            const numStr = String(num);
            if (isValid(row, col, numStr)) {
              board[row][col] = numStr;
              if (solve()) return true;
              board[row][col] = '.';
            }
          }
          return false;
        }
      }
    }
    return true;
  }

  solve();
}
```

### Word Search

**Python:**
```python
def exist(board, word):
    rows, cols = len(board), len(board[0])

    def backtrack(row, col, index):
        if index == len(word):
            return True

        if (row < 0 or row >= rows or col < 0 or col >= cols or
            board[row][col] != word[index]):
            return False

        # Mark as visited
        temp = board[row][col]
        board[row][col] = '#'

        # Explore neighbors
        found = (backtrack(row + 1, col, index + 1) or
                backtrack(row - 1, col, index + 1) or
                backtrack(row, col + 1, index + 1) or
                backtrack(row, col - 1, index + 1))

        # Restore
        board[row][col] = temp
        return found

    for r in range(rows):
        for c in range(cols):
            if backtrack(r, c, 0):
                return True

    return False
```

**JavaScript:**
```javascript
function exist(board, word) {
  const rows = board.length, cols = board[0].length;

  function backtrack(row, col, index) {
    if (index === word.length) return true;

    if (row < 0 || row >= rows || col < 0 || col >= cols ||
        board[row][col] !== word[index]) {
      return false;
    }

    // Mark as visited
    const temp = board[row][col];
    board[row][col] = '#';

    // Explore neighbors
    const found = backtrack(row + 1, col, index + 1) ||
                  backtrack(row - 1, col, index + 1) ||
                  backtrack(row, col + 1, index + 1) ||
                  backtrack(row, col - 1, index + 1);

    // Restore
    board[row][col] = temp;
    return found;
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (backtrack(r, c, 0)) return true;
    }
  }

  return false;
}
```

### Letter Combinations of Phone Number

**Python:**
```python
def letter_combinations(digits):
    if not digits:
        return []

    mapping = {
        '2': 'abc', '3': 'def', '4': 'ghi', '5': 'jkl',
        '6': 'mno', '7': 'pqrs', '8': 'tuv', '9': 'wxyz'
    }

    result = []

    def backtrack(index, path):
        if index == len(digits):
            result.append(''.join(path))
            return

        for char in mapping[digits[index]]:
            path.append(char)
            backtrack(index + 1, path)
            path.pop()

    backtrack(0, [])
    return result
```

**JavaScript:**
```javascript
function letterCombinations(digits) {
  if (!digits) return [];

  const mapping = {
    '2': 'abc', '3': 'def', '4': 'ghi', '5': 'jkl',
    '6': 'mno', '7': 'pqrs', '8': 'tuv', '9': 'wxyz'
  };

  const result = [];

  function backtrack(index, path) {
    if (index === digits.length) {
      result.push(path.join(''));
      return;
    }

    for (const char of mapping[digits[index]]) {
      path.push(char);
      backtrack(index + 1, path);
      path.pop();
    }
  }

  backtrack(0, []);
  return result;
}
```

### Palindrome Partitioning

**Python:**
```python
def partition(s):
    result = []

    def is_palindrome(string):
        return string == string[::-1]

    def backtrack(start, path):
        if start == len(s):
            result.append(path[:])
            return

        for end in range(start + 1, len(s) + 1):
            substring = s[start:end]
            if is_palindrome(substring):
                path.append(substring)
                backtrack(end, path)
                path.pop()

    backtrack(0, [])
    return result
```

**JavaScript:**
```javascript
function partition(s) {
  const result = [];

  function isPalindrome(str) {
    return str === str.split('').reverse().join('');
  }

  function backtrack(start, path) {
    if (start === s.length) {
      result.push([...path]);
      return;
    }

    for (let end = start + 1; end <= s.length; end++) {
      const substring = s.substring(start, end);
      if (isPalindrome(substring)) {
        path.push(substring);
        backtrack(end, path);
        path.pop();
      }
    }
  }

  backtrack(0, []);
  return result;
}
```

### Restore IP Addresses

**Python:**
```python
def restore_ip_addresses(s):
    result = []

    def is_valid(segment):
        if len(segment) > 1 and segment[0] == '0':
            return False
        return 0 <= int(segment) <= 255

    def backtrack(start, path):
        if len(path) == 4:
            if start == len(s):
                result.append('.'.join(path))
            return

        remaining = len(s) - start
        remaining_parts = 4 - len(path)

        # Pruning
        if remaining < remaining_parts or remaining > remaining_parts * 3:
            return

        for length in range(1, 4):
            if start + length > len(s):
                break

            segment = s[start:start + length]
            if is_valid(segment):
                path.append(segment)
                backtrack(start + length, path)
                path.pop()

    backtrack(0, [])
    return result
```

**JavaScript:**
```javascript
function restoreIpAddresses(s) {
  const result = [];

  function isValid(segment) {
    if (segment.length > 1 && segment[0] === '0') return false;
    const num = parseInt(segment);
    return num >= 0 && num <= 255;
  }

  function backtrack(start, path) {
    if (path.length === 4) {
      if (start === s.length) {
        result.push(path.join('.'));
      }
      return;
    }

    const remaining = s.length - start;
    const remainingParts = 4 - path.length;

    // Pruning
    if (remaining < remainingParts || remaining > remainingParts * 3) {
      return;
    }

    for (let length = 1; length <= 3; length++) {
      if (start + length > s.length) break;

      const segment = s.substring(start, start + length);
      if (isValid(segment)) {
        path.push(segment);
        backtrack(start + length, path);
        path.pop();
      }
    }
  }

  backtrack(0, []);
  return result;
}
```

---

## Advanced Backtracking

### Expression Add Operators

**Python:**
```python
def add_operators(num, target):
    """Add +, -, * between digits to reach target"""
    result = []

    def backtrack(index, expr, value, prev_operand):
        if index == len(num):
            if value == target:
                result.append(expr)
            return

        for i in range(index, len(num)):
            # Skip leading zeros
            if i > index and num[index] == '0':
                break

            curr_str = num[index:i+1]
            curr_num = int(curr_str)

            if index == 0:
                backtrack(i + 1, curr_str, curr_num, curr_num)
            else:
                # Addition
                backtrack(i + 1, expr + '+' + curr_str,
                         value + curr_num, curr_num)
                # Subtraction
                backtrack(i + 1, expr + '-' + curr_str,
                         value - curr_num, -curr_num)
                # Multiplication (need to handle precedence)
                backtrack(i + 1, expr + '*' + curr_str,
                         value - prev_operand + prev_operand * curr_num,
                         prev_operand * curr_num)

    backtrack(0, '', 0, 0)
    return result
```

**JavaScript:**
```javascript
function addOperators(num, target) {
  // Add +, -, * between digits to reach target
  const result = [];

  function backtrack(index, expr, value, prevOperand) {
    if (index === num.length) {
      if (value === target) {
        result.push(expr);
      }
      return;
    }

    for (let i = index; i < num.length; i++) {
      // Skip leading zeros
      if (i > index && num[index] === '0') break;

      const currStr = num.substring(index, i + 1);
      const currNum = parseInt(currStr);

      if (index === 0) {
        backtrack(i + 1, currStr, currNum, currNum);
      } else {
        // Addition
        backtrack(i + 1, expr + '+' + currStr,
          value + currNum, currNum);
        // Subtraction
        backtrack(i + 1, expr + '-' + currStr,
          value - currNum, -currNum);
        // Multiplication (need to handle precedence)
        backtrack(i + 1, expr + '*' + currStr,
          value - prevOperand + prevOperand * currNum,
          prevOperand * currNum);
      }
    }
  }

  backtrack(0, '', 0, 0);
  return result;
}
```

---

## Complexity Analysis

### Time Complexity Patterns

| Problem Type | Time Complexity |
|--------------|-----------------|
| Permutations | O(n! × n) |
| Combinations (n choose k) | O(C(n,k) × k) |
| Subsets | O(2^n × n) |
| N-Queens | O(n!) |
| Sudoku | O(9^(empty cells)) |

### Space Complexity

- Recursion stack: O(depth of recursion)
- Path storage: O(path length)
- Result storage: O(number of solutions × solution size)

---

## Must-Know Problems

### Easy
1. Letter Combinations of Phone Number

### Medium
1. Permutations I & II
2. Combinations
3. Combination Sum I, II, III
4. Subsets I & II
5. Generate Parentheses
6. Palindrome Partitioning
7. Word Search
8. Restore IP Addresses

### Hard
1. N-Queens I & II
2. Sudoku Solver
3. Expression Add Operators
4. Word Search II (with Trie)
5. Permutation Sequence

---

## Practice Checklist

- [ ] Can write backtracking template from memory
- [ ] Know how to handle duplicates
- [ ] Understand pruning strategies
- [ ] Can analyze time/space complexity
- [ ] Can convert between recursive and iterative
