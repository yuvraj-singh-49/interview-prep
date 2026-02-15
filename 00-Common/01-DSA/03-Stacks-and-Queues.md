# Stacks and Queues

## Overview

Stacks and queues are fundamental data structures that appear in parsing, BFS/DFS, and many optimization problems. Monotonic stacks are particularly important for Lead-level interviews.

---

## Key Concepts

### Stack (LIFO - Last In, First Out)

```
Operations:
- push(x): Add to top - O(1)
- pop(): Remove from top - O(1)
- peek()/top(): View top - O(1)
- isEmpty(): Check if empty - O(1)

Use cases: Undo operations, parsing, DFS, backtracking
```

### Queue (FIFO - First In, First Out)

```
Operations:
- enqueue(x): Add to back - O(1)
- dequeue(): Remove from front - O(1)
- peek()/front(): View front - O(1)
- isEmpty(): Check if empty - O(1)

Use cases: BFS, task scheduling, buffering
```

### Deque (Double-Ended Queue)

```
Operations:
- addFirst(x), addLast(x): O(1)
- removeFirst(), removeLast(): O(1)
- peekFirst(), peekLast(): O(1)

Use cases: Sliding window max/min, palindrome check
```

---

## Implementation

### Stack using Array/List

**Python:**
```python
class Stack:
    def __init__(self):
        self.items = []

    def push(self, item):
        self.items.append(item)

    def pop(self):
        if not self.is_empty():
            return self.items.pop()
        raise IndexError("Stack is empty")

    def peek(self):
        if not self.is_empty():
            return self.items[-1]
        raise IndexError("Stack is empty")

    def is_empty(self):
        return len(self.items) == 0

    def size(self):
        return len(self.items)
```

**JavaScript:**
```javascript
class Stack {
  constructor() {
    this.items = [];
  }

  push(item) {
    this.items.push(item);
  }

  pop() {
    if (!this.isEmpty()) {
      return this.items.pop();
    }
    throw new Error("Stack is empty");
  }

  peek() {
    if (!this.isEmpty()) {
      return this.items[this.items.length - 1];
    }
    throw new Error("Stack is empty");
  }

  isEmpty() {
    return this.items.length === 0;
  }

  size() {
    return this.items.length;
  }
}
```

### Queue using Deque

**Python:**
```python
from collections import deque

class Queue:
    def __init__(self):
        self.items = deque()

    def enqueue(self, item):
        self.items.append(item)

    def dequeue(self):
        if not self.is_empty():
            return self.items.popleft()
        raise IndexError("Queue is empty")

    def front(self):
        if not self.is_empty():
            return self.items[0]
        raise IndexError("Queue is empty")

    def is_empty(self):
        return len(self.items) == 0
```

**JavaScript:**
```javascript
class Queue {
  constructor() {
    this.items = [];
    this.headIndex = 0; // Track front for O(1) dequeue
  }

  enqueue(item) {
    this.items.push(item);
  }

  dequeue() {
    if (!this.isEmpty()) {
      const item = this.items[this.headIndex];
      this.headIndex++;
      // Cleanup when half the array is empty
      if (this.headIndex > this.items.length / 2) {
        this.items = this.items.slice(this.headIndex);
        this.headIndex = 0;
      }
      return item;
    }
    throw new Error("Queue is empty");
  }

  front() {
    if (!this.isEmpty()) {
      return this.items[this.headIndex];
    }
    throw new Error("Queue is empty");
  }

  isEmpty() {
    return this.headIndex >= this.items.length;
  }

  size() {
    return this.items.length - this.headIndex;
  }
}
```

### Queue using Two Stacks

**Python:**
```python
class QueueUsingStacks:
    """Amortized O(1) for all operations"""
    def __init__(self):
        self.stack_in = []   # For enqueue
        self.stack_out = []  # For dequeue

    def enqueue(self, x):
        self.stack_in.append(x)

    def dequeue(self):
        self._transfer()
        return self.stack_out.pop()

    def front(self):
        self._transfer()
        return self.stack_out[-1]

    def _transfer(self):
        if not self.stack_out:
            while self.stack_in:
                self.stack_out.append(self.stack_in.pop())

    def is_empty(self):
        return not self.stack_in and not self.stack_out
```

**JavaScript:**
```javascript
class QueueUsingStacks {
  // Amortized O(1) for all operations
  constructor() {
    this.stackIn = [];  // For enqueue
    this.stackOut = []; // For dequeue
  }

  enqueue(x) {
    this.stackIn.push(x);
  }

  dequeue() {
    this._transfer();
    return this.stackOut.pop();
  }

  front() {
    this._transfer();
    return this.stackOut[this.stackOut.length - 1];
  }

  _transfer() {
    if (this.stackOut.length === 0) {
      while (this.stackIn.length > 0) {
        this.stackOut.push(this.stackIn.pop());
      }
    }
  }

  isEmpty() {
    return this.stackIn.length === 0 && this.stackOut.length === 0;
  }
}
```

### Stack using Two Queues

**Python:**
```python
from collections import deque

class StackUsingQueues:
    def __init__(self):
        self.q1 = deque()
        self.q2 = deque()

    def push(self, x):
        self.q2.append(x)
        while self.q1:
            self.q2.append(self.q1.popleft())
        self.q1, self.q2 = self.q2, self.q1

    def pop(self):
        return self.q1.popleft()

    def top(self):
        return self.q1[0]

    def is_empty(self):
        return len(self.q1) == 0
```

**JavaScript:**
```javascript
class StackUsingQueues {
  constructor() {
    this.q1 = [];
    this.q2 = [];
  }

  push(x) {
    this.q2.push(x);
    while (this.q1.length > 0) {
      this.q2.push(this.q1.shift());
    }
    [this.q1, this.q2] = [this.q2, this.q1];
  }

  pop() {
    return this.q1.shift();
  }

  top() {
    return this.q1[0];
  }

  isEmpty() {
    return this.q1.length === 0;
  }
}
```

---

## Essential Patterns

### 1. Parentheses Matching

**Valid Parentheses:**

**Python:**
```python
def is_valid(s):
    stack = []
    mapping = {')': '(', '}': '{', ']': '['}

    for char in s:
        if char in mapping:
            if not stack or stack[-1] != mapping[char]:
                return False
            stack.pop()
        else:
            stack.append(char)

    return len(stack) == 0
```

**JavaScript:**
```javascript
function isValid(s) {
  const stack = [];
  const mapping = { ')': '(', '}': '{', ']': '[' };

  for (const char of s) {
    if (char in mapping) {
      if (stack.length === 0 || stack[stack.length - 1] !== mapping[char]) {
        return false;
      }
      stack.pop();
    } else {
      stack.push(char);
    }
  }

  return stack.length === 0;
}
```

**Longest Valid Parentheses:**

**Python:**
```python
def longest_valid_parentheses(s):
    stack = [-1]  # Base for valid substring length
    max_len = 0

    for i, char in enumerate(s):
        if char == '(':
            stack.append(i)
        else:
            stack.pop()
            if not stack:
                stack.append(i)  # New base
            else:
                max_len = max(max_len, i - stack[-1])

    return max_len
```

**JavaScript:**
```javascript
function longestValidParentheses(s) {
  const stack = [-1]; // Base for valid substring length
  let maxLen = 0;

  for (let i = 0; i < s.length; i++) {
    if (s[i] === '(') {
      stack.push(i);
    } else {
      stack.pop();
      if (stack.length === 0) {
        stack.push(i); // New base
      } else {
        maxLen = Math.max(maxLen, i - stack[stack.length - 1]);
      }
    }
  }

  return maxLen;
}
```

**Generate All Valid Parentheses:**

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

### 2. Monotonic Stack

**When to use:** Next greater/smaller element, histogram problems, stock span

**Monotonic Decreasing Stack (Next Greater Element):**

**Python:**
```python
def next_greater_element(nums):
    """For each element, find the next greater element to the right"""
    n = len(nums)
    result = [-1] * n
    stack = []  # Stack of indices

    for i in range(n):
        while stack and nums[i] > nums[stack[-1]]:
            idx = stack.pop()
            result[idx] = nums[i]
        stack.append(i)

    return result
```

**JavaScript:**
```javascript
function nextGreaterElement(nums) {
  // For each element, find the next greater element to the right
  const n = nums.length;
  const result = new Array(n).fill(-1);
  const stack = []; // Stack of indices

  for (let i = 0; i < n; i++) {
    while (stack.length > 0 && nums[i] > nums[stack[stack.length - 1]]) {
      const idx = stack.pop();
      result[idx] = nums[i];
    }
    stack.push(i);
  }

  return result;
}
```

**Monotonic Increasing Stack (Next Smaller Element):**

**Python:**
```python
def next_smaller_element(nums):
    n = len(nums)
    result = [-1] * n
    stack = []

    for i in range(n):
        while stack and nums[i] < nums[stack[-1]]:
            idx = stack.pop()
            result[idx] = nums[i]
        stack.append(i)

    return result
```

**JavaScript:**
```javascript
function nextSmallerElement(nums) {
  const n = nums.length;
  const result = new Array(n).fill(-1);
  const stack = [];

  for (let i = 0; i < n; i++) {
    while (stack.length > 0 && nums[i] < nums[stack[stack.length - 1]]) {
      const idx = stack.pop();
      result[idx] = nums[i];
    }
    stack.push(i);
  }

  return result;
}
```

**Previous Greater/Smaller:**

**Python:**
```python
def previous_smaller_element(nums):
    n = len(nums)
    result = [-1] * n
    stack = []

    for i in range(n):
        while stack and nums[stack[-1]] >= nums[i]:
            stack.pop()
        if stack:
            result[i] = nums[stack[-1]]
        stack.append(i)

    return result
```

**JavaScript:**
```javascript
function previousSmallerElement(nums) {
  const n = nums.length;
  const result = new Array(n).fill(-1);
  const stack = [];

  for (let i = 0; i < n; i++) {
    while (stack.length > 0 && nums[stack[stack.length - 1]] >= nums[i]) {
      stack.pop();
    }
    if (stack.length > 0) {
      result[i] = nums[stack[stack.length - 1]];
    }
    stack.push(i);
  }

  return result;
}
```

---

### 3. Largest Rectangle in Histogram

**Python:**
```python
def largest_rectangle_area(heights):
    """Classic monotonic stack problem"""
    stack = []  # Stack of indices
    max_area = 0

    for i, h in enumerate(heights + [0]):  # Add 0 to flush remaining
        while stack and heights[stack[-1]] > h:
            height = heights[stack.pop()]
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)

    return max_area
```

**JavaScript:**
```javascript
function largestRectangleArea(heights) {
  // Classic monotonic stack problem
  const stack = []; // Stack of indices
  let maxArea = 0;

  // Add 0 to flush remaining
  heights.push(0);

  for (let i = 0; i < heights.length; i++) {
    while (stack.length > 0 && heights[stack[stack.length - 1]] > heights[i]) {
      const height = heights[stack.pop()];
      const width = stack.length === 0 ? i : i - stack[stack.length - 1] - 1;
      maxArea = Math.max(maxArea, height * width);
    }
    stack.push(i);
  }

  heights.pop(); // Restore original array
  return maxArea;
}
```

**Maximal Rectangle in Matrix:**

**Python:**
```python
def maximal_rectangle(matrix):
    if not matrix:
        return 0

    n = len(matrix[0])
    heights = [0] * n
    max_area = 0

    for row in matrix:
        for i in range(n):
            heights[i] = heights[i] + 1 if row[i] == '1' else 0
        max_area = max(max_area, largest_rectangle_area(heights))

    return max_area
```

**JavaScript:**
```javascript
function maximalRectangle(matrix) {
  if (!matrix || matrix.length === 0) return 0;

  const n = matrix[0].length;
  const heights = new Array(n).fill(0);
  let maxArea = 0;

  for (const row of matrix) {
    for (let i = 0; i < n; i++) {
      heights[i] = row[i] === '1' ? heights[i] + 1 : 0;
    }
    maxArea = Math.max(maxArea, largestRectangleArea([...heights]));
  }

  return maxArea;
}
```

---

### 4. Daily Temperatures / Stock Span

**Daily Temperatures:**

**Python:**
```python
def daily_temperatures(temperatures):
    n = len(temperatures)
    result = [0] * n
    stack = []  # Stack of indices

    for i in range(n):
        while stack and temperatures[i] > temperatures[stack[-1]]:
            idx = stack.pop()
            result[idx] = i - idx
        stack.append(i)

    return result
```

**JavaScript:**
```javascript
function dailyTemperatures(temperatures) {
  const n = temperatures.length;
  const result = new Array(n).fill(0);
  const stack = []; // Stack of indices

  for (let i = 0; i < n; i++) {
    while (stack.length > 0 && temperatures[i] > temperatures[stack[stack.length - 1]]) {
      const idx = stack.pop();
      result[idx] = i - idx;
    }
    stack.push(i);
  }

  return result;
}
```

**Stock Span:**

**Python:**
```python
class StockSpanner:
    def __init__(self):
        self.stack = []  # (price, span)

    def next(self, price):
        span = 1
        while self.stack and self.stack[-1][0] <= price:
            span += self.stack.pop()[1]
        self.stack.append((price, span))
        return span
```

**JavaScript:**
```javascript
class StockSpanner {
  constructor() {
    this.stack = []; // [price, span]
  }

  next(price) {
    let span = 1;
    while (this.stack.length > 0 && this.stack[this.stack.length - 1][0] <= price) {
      span += this.stack.pop()[1];
    }
    this.stack.push([price, span]);
    return span;
  }
}
```

---

### 5. Sliding Window Maximum (Monotonic Deque)

**Python:**
```python
from collections import deque

def max_sliding_window(nums, k):
    """O(n) using monotonic decreasing deque"""
    result = []
    dq = deque()  # Stores indices

    for i in range(len(nums)):
        # Remove elements outside window
        while dq and dq[0] <= i - k:
            dq.popleft()

        # Maintain decreasing order
        while dq and nums[dq[-1]] < nums[i]:
            dq.pop()

        dq.append(i)

        # Add to result once window is full
        if i >= k - 1:
            result.append(nums[dq[0]])

    return result
```

**JavaScript:**
```javascript
function maxSlidingWindow(nums, k) {
  // O(n) using monotonic decreasing deque
  const result = [];
  const dq = []; // Stores indices

  for (let i = 0; i < nums.length; i++) {
    // Remove elements outside window
    while (dq.length > 0 && dq[0] <= i - k) {
      dq.shift();
    }

    // Maintain decreasing order
    while (dq.length > 0 && nums[dq[dq.length - 1]] < nums[i]) {
      dq.pop();
    }

    dq.push(i);

    // Add to result once window is full
    if (i >= k - 1) {
      result.push(nums[dq[0]]);
    }
  }

  return result;
}
```

**Sliding Window Minimum:**

**Python:**
```python
def min_sliding_window(nums, k):
    result = []
    dq = deque()

    for i in range(len(nums)):
        while dq and dq[0] <= i - k:
            dq.popleft()

        while dq and nums[dq[-1]] > nums[i]:  # Change to >
            dq.pop()

        dq.append(i)

        if i >= k - 1:
            result.append(nums[dq[0]])

    return result
```

**JavaScript:**
```javascript
function minSlidingWindow(nums, k) {
  const result = [];
  const dq = []; // Stores indices

  for (let i = 0; i < nums.length; i++) {
    while (dq.length > 0 && dq[0] <= i - k) {
      dq.shift();
    }

    while (dq.length > 0 && nums[dq[dq.length - 1]] > nums[i]) { // Change to >
      dq.pop();
    }

    dq.push(i);

    if (i >= k - 1) {
      result.push(nums[dq[0]]);
    }
  }

  return result;
}
```

---

### 6. Min Stack

**Python:**
```python
class MinStack:
    def __init__(self):
        self.stack = []
        self.min_stack = []

    def push(self, val):
        self.stack.append(val)
        if not self.min_stack or val <= self.min_stack[-1]:
            self.min_stack.append(val)

    def pop(self):
        val = self.stack.pop()
        if val == self.min_stack[-1]:
            self.min_stack.pop()

    def top(self):
        return self.stack[-1]

    def getMin(self):
        return self.min_stack[-1]
```

**JavaScript:**
```javascript
class MinStack {
  constructor() {
    this.stack = [];
    this.minStack = [];
  }

  push(val) {
    this.stack.push(val);
    if (this.minStack.length === 0 || val <= this.minStack[this.minStack.length - 1]) {
      this.minStack.push(val);
    }
  }

  pop() {
    const val = this.stack.pop();
    if (val === this.minStack[this.minStack.length - 1]) {
      this.minStack.pop();
    }
  }

  top() {
    return this.stack[this.stack.length - 1];
  }

  getMin() {
    return this.minStack[this.minStack.length - 1];
  }
}
```

**Min Stack with O(1) Space:**

**Python:**
```python
class MinStackO1Space:
    def __init__(self):
        self.stack = []
        self.min_val = float('inf')

    def push(self, val):
        if val <= self.min_val:
            self.stack.append(self.min_val)  # Push old min
            self.min_val = val
        self.stack.append(val)

    def pop(self):
        val = self.stack.pop()
        if val == self.min_val:
            self.min_val = self.stack.pop()  # Restore previous min

    def top(self):
        return self.stack[-1]

    def getMin(self):
        return self.min_val
```

**JavaScript:**
```javascript
class MinStackO1Space {
  constructor() {
    this.stack = [];
    this.minVal = Infinity;
  }

  push(val) {
    if (val <= this.minVal) {
      this.stack.push(this.minVal); // Push old min
      this.minVal = val;
    }
    this.stack.push(val);
  }

  pop() {
    const val = this.stack.pop();
    if (val === this.minVal) {
      this.minVal = this.stack.pop(); // Restore previous min
    }
  }

  top() {
    return this.stack[this.stack.length - 1];
  }

  getMin() {
    return this.minVal;
  }
}
```

---

### 7. Expression Evaluation

**Basic Calculator (with +, -, parentheses):**

**Python:**
```python
def calculate(s):
    stack = []
    result = 0
    num = 0
    sign = 1

    for char in s:
        if char.isdigit():
            num = num * 10 + int(char)
        elif char == '+':
            result += sign * num
            num = 0
            sign = 1
        elif char == '-':
            result += sign * num
            num = 0
            sign = -1
        elif char == '(':
            stack.append(result)
            stack.append(sign)
            result = 0
            sign = 1
        elif char == ')':
            result += sign * num
            num = 0
            result *= stack.pop()  # Sign before parenthesis
            result += stack.pop()  # Result before parenthesis

    return result + sign * num
```

**JavaScript:**
```javascript
function calculate(s) {
  const stack = [];
  let result = 0;
  let num = 0;
  let sign = 1;

  for (const char of s) {
    if (char >= '0' && char <= '9') {
      num = num * 10 + parseInt(char);
    } else if (char === '+') {
      result += sign * num;
      num = 0;
      sign = 1;
    } else if (char === '-') {
      result += sign * num;
      num = 0;
      sign = -1;
    } else if (char === '(') {
      stack.push(result);
      stack.push(sign);
      result = 0;
      sign = 1;
    } else if (char === ')') {
      result += sign * num;
      num = 0;
      result *= stack.pop(); // Sign before parenthesis
      result += stack.pop(); // Result before parenthesis
    }
  }

  return result + sign * num;
}
```

**Evaluate Reverse Polish Notation:**

**Python:**
```python
def eval_rpn(tokens):
    stack = []
    operators = {
        '+': lambda a, b: a + b,
        '-': lambda a, b: a - b,
        '*': lambda a, b: a * b,
        '/': lambda a, b: int(a / b)  # Truncate toward zero
    }

    for token in tokens:
        if token in operators:
            b, a = stack.pop(), stack.pop()
            stack.append(operators[token](a, b))
        else:
            stack.append(int(token))

    return stack[0]
```

**JavaScript:**
```javascript
function evalRPN(tokens) {
  const stack = [];
  const operators = {
    '+': (a, b) => a + b,
    '-': (a, b) => a - b,
    '*': (a, b) => a * b,
    '/': (a, b) => Math.trunc(a / b) // Truncate toward zero
  };

  for (const token of tokens) {
    if (token in operators) {
      const b = stack.pop();
      const a = stack.pop();
      stack.push(operators[token](a, b));
    } else {
      stack.push(parseInt(token));
    }
  }

  return stack[0];
}
```

---

### 8. Decode String

**Python:**
```python
def decode_string(s):
    """k[encoded_string] -> repeat encoded_string k times"""
    stack = []
    current_num = 0
    current_str = ""

    for char in s:
        if char.isdigit():
            current_num = current_num * 10 + int(char)
        elif char == '[':
            stack.append((current_str, current_num))
            current_str = ""
            current_num = 0
        elif char == ']':
            prev_str, num = stack.pop()
            current_str = prev_str + current_str * num
        else:
            current_str += char

    return current_str
```

**JavaScript:**
```javascript
function decodeString(s) {
  // k[encoded_string] -> repeat encoded_string k times
  const stack = [];
  let currentNum = 0;
  let currentStr = "";

  for (const char of s) {
    if (char >= '0' && char <= '9') {
      currentNum = currentNum * 10 + parseInt(char);
    } else if (char === '[') {
      stack.push([currentStr, currentNum]);
      currentStr = "";
      currentNum = 0;
    } else if (char === ']') {
      const [prevStr, num] = stack.pop();
      currentStr = prevStr + currentStr.repeat(num);
    } else {
      currentStr += char;
    }
  }

  return currentStr;
}
```

---

### 9. Trapping Rain Water

**Python:**
```python
def trap(height):
    """Using stack - O(n) time, O(n) space"""
    stack = []
    water = 0

    for i, h in enumerate(height):
        while stack and height[stack[-1]] < h:
            bottom = stack.pop()
            if not stack:
                break
            width = i - stack[-1] - 1
            bounded_height = min(h, height[stack[-1]]) - height[bottom]
            water += width * bounded_height
        stack.append(i)

    return water

def trap_two_pointers(height):
    """O(n) time, O(1) space"""
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

**JavaScript:**
```javascript
// Using stack - O(n) time, O(n) space
function trap(height) {
  const stack = [];
  let water = 0;

  for (let i = 0; i < height.length; i++) {
    while (stack.length > 0 && height[stack[stack.length - 1]] < height[i]) {
      const bottom = stack.pop();
      if (stack.length === 0) break;
      const width = i - stack[stack.length - 1] - 1;
      const boundedHeight = Math.min(height[i], height[stack[stack.length - 1]]) - height[bottom];
      water += width * boundedHeight;
    }
    stack.push(i);
  }

  return water;
}

// O(n) time, O(1) space
function trapTwoPointers(height) {
  let left = 0, right = height.length - 1;
  let leftMax = 0, rightMax = 0;
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

## Priority Queue / Heap

### Min Heap Operations

**Python:**
```python
import heapq

# Python has min-heap by default
heap = []
heapq.heappush(heap, 3)
heapq.heappush(heap, 1)
heapq.heappush(heap, 2)

min_val = heapq.heappop(heap)  # Returns 1
peek = heap[0]  # View min without removing

# For max-heap, negate values
max_heap = []
heapq.heappush(max_heap, -3)
heapq.heappush(max_heap, -1)
max_val = -heapq.heappop(max_heap)  # Returns 3

# Heapify existing list - O(n)
arr = [3, 1, 4, 1, 5, 9]
heapq.heapify(arr)
```

**JavaScript:**
```javascript
// JavaScript doesn't have a built-in heap, implement MinHeap class
class MinHeap {
  constructor() {
    this.heap = [];
  }

  push(val) {
    this.heap.push(val);
    this._bubbleUp(this.heap.length - 1);
  }

  pop() {
    if (this.heap.length === 0) return null;
    const min = this.heap[0];
    const last = this.heap.pop();
    if (this.heap.length > 0) {
      this.heap[0] = last;
      this._bubbleDown(0);
    }
    return min;
  }

  peek() {
    return this.heap[0];
  }

  size() {
    return this.heap.length;
  }

  _bubbleUp(idx) {
    while (idx > 0) {
      const parent = Math.floor((idx - 1) / 2);
      if (this.heap[idx] >= this.heap[parent]) break;
      [this.heap[idx], this.heap[parent]] = [this.heap[parent], this.heap[idx]];
      idx = parent;
    }
  }

  _bubbleDown(idx) {
    while (true) {
      const left = 2 * idx + 1;
      const right = 2 * idx + 2;
      let smallest = idx;

      if (left < this.heap.length && this.heap[left] < this.heap[smallest]) {
        smallest = left;
      }
      if (right < this.heap.length && this.heap[right] < this.heap[smallest]) {
        smallest = right;
      }
      if (smallest === idx) break;

      [this.heap[idx], this.heap[smallest]] = [this.heap[smallest], this.heap[idx]];
      idx = smallest;
    }
  }
}

// For max-heap, extend or modify comparison
class MaxHeap extends MinHeap {
  _bubbleUp(idx) {
    while (idx > 0) {
      const parent = Math.floor((idx - 1) / 2);
      if (this.heap[idx] <= this.heap[parent]) break;
      [this.heap[idx], this.heap[parent]] = [this.heap[parent], this.heap[idx]];
      idx = parent;
    }
  }

  _bubbleDown(idx) {
    while (true) {
      const left = 2 * idx + 1;
      const right = 2 * idx + 2;
      let largest = idx;

      if (left < this.heap.length && this.heap[left] > this.heap[largest]) {
        largest = left;
      }
      if (right < this.heap.length && this.heap[right] > this.heap[largest]) {
        largest = right;
      }
      if (largest === idx) break;

      [this.heap[idx], this.heap[largest]] = [this.heap[largest], this.heap[idx]];
      idx = largest;
    }
  }
}
```

### Top K Elements

**Python:**
```python
def find_k_largest(nums, k):
    """Using min-heap of size k - O(n log k)"""
    heap = nums[:k]
    heapq.heapify(heap)

    for num in nums[k:]:
        if num > heap[0]:
            heapq.heapreplace(heap, num)

    return heap

def find_k_smallest(nums, k):
    """Using max-heap (negated) of size k"""
    heap = [-x for x in nums[:k]]
    heapq.heapify(heap)

    for num in nums[k:]:
        if num < -heap[0]:
            heapq.heapreplace(heap, -num)

    return [-x for x in heap]
```

**JavaScript:**
```javascript
// Using min-heap of size k - O(n log k)
function findKLargest(nums, k) {
  const minHeap = new MinHeap();

  for (let i = 0; i < k; i++) {
    minHeap.push(nums[i]);
  }

  for (let i = k; i < nums.length; i++) {
    if (nums[i] > minHeap.peek()) {
      minHeap.pop();
      minHeap.push(nums[i]);
    }
  }

  return minHeap.heap;
}

// Using max-heap of size k
function findKSmallest(nums, k) {
  const maxHeap = new MaxHeap();

  for (let i = 0; i < k; i++) {
    maxHeap.push(nums[i]);
  }

  for (let i = k; i < nums.length; i++) {
    if (nums[i] < maxHeap.peek()) {
      maxHeap.pop();
      maxHeap.push(nums[i]);
    }
  }

  return maxHeap.heap;
}
```

---

## Must-Know Problems

### Easy
1. Valid Parentheses
2. Min Stack
3. Implement Queue using Stacks
4. Implement Stack using Queues

### Medium
1. Daily Temperatures
2. Next Greater Element I, II
3. Evaluate Reverse Polish Notation
4. Decode String
5. Simplify Path
6. Asteroid Collision
7. Remove K Digits
8. 132 Pattern

### Hard
1. Largest Rectangle in Histogram
2. Maximal Rectangle
3. Sliding Window Maximum
4. Trapping Rain Water
5. Basic Calculator I, II, III
6. Longest Valid Parentheses

---

## Practice Checklist

- [ ] Can identify when to use monotonic stack
- [ ] Understand the difference between increasing/decreasing stack
- [ ] Can implement sliding window max/min with deque
- [ ] Can evaluate expressions with parentheses
- [ ] Know stack vs queue use cases instinctively
