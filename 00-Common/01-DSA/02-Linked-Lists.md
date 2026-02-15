# Linked Lists

## Overview

Linked lists test your ability to manipulate pointers and handle edge cases. As a Lead engineer, you should solve these problems fluently and explain the trade-offs between arrays and linked lists.

---

## Key Concepts

### Types of Linked Lists

```
Singly Linked List:  [1] -> [2] -> [3] -> null
Doubly Linked List:  null <- [1] <-> [2] <-> [3] -> null
Circular Linked List: [1] -> [2] -> [3] -> [1]
```

### Time Complexity

| Operation | Singly | Doubly |
|-----------|--------|--------|
| Access | O(n) | O(n) |
| Search | O(n) | O(n) |
| Insert at head | O(1) | O(1) |
| Insert at tail | O(n)* | O(1)** |
| Insert at position | O(n) | O(n) |
| Delete at head | O(1) | O(1) |
| Delete at tail | O(n) | O(1) |
| Delete at position | O(n) | O(n) |

*O(1) if tail pointer maintained
**With tail pointer

### When to Use Linked Lists

**Use linked lists when:**
- Frequent insertions/deletions at beginning
- Unknown size that changes frequently
- No random access needed
- Implementing stacks, queues, LRU cache

**Use arrays when:**
- Random access needed
- Memory locality matters (cache performance)
- Size is fixed or rarely changes

---

## Node Implementation

**Python:**
```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

class DoublyListNode:
    def __init__(self, val=0, prev=None, next=None):
        self.val = val
        self.prev = prev
        self.next = next
```

**JavaScript:**
```javascript
class ListNode {
  constructor(val = 0, next = null) {
    this.val = val;
    this.next = next;
  }
}

class DoublyListNode {
  constructor(val = 0, prev = null, next = null) {
    this.val = val;
    this.prev = prev;
    this.next = next;
  }
}
```

---

## Essential Patterns

### 1. Dummy Head (Sentinel Node)

**When to use:** Simplifies edge cases when head might change

**Python:**
```python
def remove_elements(head, val):
    dummy = ListNode(0)
    dummy.next = head
    curr = dummy

    while curr.next:
        if curr.next.val == val:
            curr.next = curr.next.next
        else:
            curr = curr.next

    return dummy.next
```

**JavaScript:**
```javascript
function removeElements(head, val) {
  const dummy = new ListNode(0);
  dummy.next = head;
  let curr = dummy;

  while (curr.next) {
    if (curr.next.val === val) {
      curr.next = curr.next.next;
    } else {
      curr = curr.next;
    }
  }

  return dummy.next;
}
```

---

### 2. Fast and Slow Pointers (Floyd's Algorithm)

**When to use:** Finding middle, detecting cycles, finding nth from end

**Finding Middle:**

**Python:**
```python
def find_middle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow  # Middle node (for odd) or second middle (for even)
```

**JavaScript:**
```javascript
function findMiddle(head) {
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow.next;
    fast = fast.next.next;
  }
  return slow; // Middle node (for odd) or second middle (for even)
}
```

**Detecting Cycle:**

**Python:**
```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False

def find_cycle_start(head):
    """Floyd's cycle detection - find where cycle begins"""
    slow = fast = head

    # Phase 1: Detect cycle
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            break
    else:
        return None  # No cycle

    # Phase 2: Find cycle start
    slow = head
    while slow != fast:
        slow = slow.next
        fast = fast.next

    return slow
```

**JavaScript:**
```javascript
function hasCycle(head) {
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow === fast) return true;
  }
  return false;
}

function findCycleStart(head) {
  // Floyd's cycle detection - find where cycle begins
  let slow = head, fast = head;

  // Phase 1: Detect cycle
  while (fast && fast.next) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow === fast) break;
  }

  if (!fast || !fast.next) return null; // No cycle

  // Phase 2: Find cycle start
  slow = head;
  while (slow !== fast) {
    slow = slow.next;
    fast = fast.next;
  }

  return slow;
}
```

**Nth from End:**

**Python:**
```python
def remove_nth_from_end(head, n):
    dummy = ListNode(0, head)
    slow = fast = dummy

    # Move fast n+1 steps ahead
    for _ in range(n + 1):
        fast = fast.next

    # Move both until fast reaches end
    while fast:
        slow = slow.next
        fast = fast.next

    # Remove nth node
    slow.next = slow.next.next
    return dummy.next
```

**JavaScript:**
```javascript
function removeNthFromEnd(head, n) {
  const dummy = new ListNode(0, head);
  let slow = dummy, fast = dummy;

  // Move fast n+1 steps ahead
  for (let i = 0; i <= n; i++) {
    fast = fast.next;
  }

  // Move both until fast reaches end
  while (fast) {
    slow = slow.next;
    fast = fast.next;
  }

  // Remove nth node
  slow.next = slow.next.next;
  return dummy.next;
}
```

---

### 3. Reversal Techniques

**Iterative Reversal:**

**Python:**
```python
def reverse_list(head):
    prev = None
    curr = head

    while curr:
        next_temp = curr.next  # Save next
        curr.next = prev       # Reverse link
        prev = curr            # Move prev forward
        curr = next_temp       # Move curr forward

    return prev
```

**JavaScript:**
```javascript
function reverseList(head) {
  let prev = null;
  let curr = head;

  while (curr) {
    const nextTemp = curr.next; // Save next
    curr.next = prev;           // Reverse link
    prev = curr;                // Move prev forward
    curr = nextTemp;            // Move curr forward
  }

  return prev;
}
```

**Recursive Reversal:**

**Python:**
```python
def reverse_list_recursive(head):
    if not head or not head.next:
        return head

    new_head = reverse_list_recursive(head.next)
    head.next.next = head
    head.next = None

    return new_head
```

**JavaScript:**
```javascript
function reverseListRecursive(head) {
  if (!head || !head.next) {
    return head;
  }

  const newHead = reverseListRecursive(head.next);
  head.next.next = head;
  head.next = null;

  return newHead;
}
```

**Reverse Between Positions:**

**Python:**
```python
def reverse_between(head, left, right):
    if not head or left == right:
        return head

    dummy = ListNode(0, head)
    prev = dummy

    # Move to position before left
    for _ in range(left - 1):
        prev = prev.next

    # Reverse from left to right
    curr = prev.next
    for _ in range(right - left):
        next_node = curr.next
        curr.next = next_node.next
        next_node.next = prev.next
        prev.next = next_node

    return dummy.next
```

**JavaScript:**
```javascript
function reverseBetween(head, left, right) {
  if (!head || left === right) return head;

  const dummy = new ListNode(0, head);
  let prev = dummy;

  // Move to position before left
  for (let i = 0; i < left - 1; i++) {
    prev = prev.next;
  }

  // Reverse from left to right
  let curr = prev.next;
  for (let i = 0; i < right - left; i++) {
    const nextNode = curr.next;
    curr.next = nextNode.next;
    nextNode.next = prev.next;
    prev.next = nextNode;
  }

  return dummy.next;
}
```

**Reverse in K-Groups:**

**Python:**
```python
def reverse_k_group(head, k):
    def reverse(head, k):
        prev = None
        curr = head
        for _ in range(k):
            next_temp = curr.next
            curr.next = prev
            prev = curr
            curr = next_temp
        return prev

    # Check if k nodes exist
    count = 0
    curr = head
    while curr and count < k:
        curr = curr.next
        count += 1

    if count < k:
        return head

    # Reverse first k nodes
    new_head = reverse(head, k)

    # Recursively handle rest
    head.next = reverse_k_group(curr, k)

    return new_head
```

**JavaScript:**
```javascript
function reverseKGroup(head, k) {
  function reverse(head, k) {
    let prev = null;
    let curr = head;
    for (let i = 0; i < k; i++) {
      const nextTemp = curr.next;
      curr.next = prev;
      prev = curr;
      curr = nextTemp;
    }
    return prev;
  }

  // Check if k nodes exist
  let count = 0;
  let curr = head;
  while (curr && count < k) {
    curr = curr.next;
    count++;
  }

  if (count < k) return head;

  // Reverse first k nodes
  const newHead = reverse(head, k);

  // Recursively handle rest
  head.next = reverseKGroup(curr, k);

  return newHead;
}
```

---

### 4. Merge Operations

**Merge Two Sorted Lists:**

**Python:**
```python
def merge_two_lists(l1, l2):
    dummy = ListNode(0)
    curr = dummy

    while l1 and l2:
        if l1.val <= l2.val:
            curr.next = l1
            l1 = l1.next
        else:
            curr.next = l2
            l2 = l2.next
        curr = curr.next

    curr.next = l1 or l2
    return dummy.next
```

**JavaScript:**
```javascript
function mergeTwoLists(l1, l2) {
  const dummy = new ListNode(0);
  let curr = dummy;

  while (l1 && l2) {
    if (l1.val <= l2.val) {
      curr.next = l1;
      l1 = l1.next;
    } else {
      curr.next = l2;
      l2 = l2.next;
    }
    curr = curr.next;
  }

  curr.next = l1 || l2;
  return dummy.next;
}
```

**Merge K Sorted Lists (Divide & Conquer):**

**Python:**
```python
def merge_k_lists(lists):
    if not lists:
        return None

    def merge_two(l1, l2):
        dummy = ListNode(0)
        curr = dummy
        while l1 and l2:
            if l1.val <= l2.val:
                curr.next = l1
                l1 = l1.next
            else:
                curr.next = l2
                l2 = l2.next
            curr = curr.next
        curr.next = l1 or l2
        return dummy.next

    while len(lists) > 1:
        merged = []
        for i in range(0, len(lists), 2):
            l1 = lists[i]
            l2 = lists[i + 1] if i + 1 < len(lists) else None
            merged.append(merge_two(l1, l2))
        lists = merged

    return lists[0]
```

**JavaScript:**
```javascript
function mergeKLists(lists) {
  if (!lists || lists.length === 0) return null;

  function mergeTwo(l1, l2) {
    const dummy = new ListNode(0);
    let curr = dummy;
    while (l1 && l2) {
      if (l1.val <= l2.val) {
        curr.next = l1;
        l1 = l1.next;
      } else {
        curr.next = l2;
        l2 = l2.next;
      }
      curr = curr.next;
    }
    curr.next = l1 || l2;
    return dummy.next;
  }

  while (lists.length > 1) {
    const merged = [];
    for (let i = 0; i < lists.length; i += 2) {
      const l1 = lists[i];
      const l2 = i + 1 < lists.length ? lists[i + 1] : null;
      merged.push(mergeTwo(l1, l2));
    }
    lists = merged;
  }

  return lists[0];
}
```

**Merge K Sorted Lists (Heap):**

**Python:**
```python
import heapq

def merge_k_lists_heap(lists):
    dummy = ListNode(0)
    curr = dummy

    heap = []
    for i, node in enumerate(lists):
        if node:
            heapq.heappush(heap, (node.val, i, node))

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
// Using a MinHeap implementation (or use a library like 'heap-js')
class MinHeap {
  constructor(compareFn = (a, b) => a - b) {
    this.heap = [];
    this.compare = compareFn;
  }

  push(val) {
    this.heap.push(val);
    this._bubbleUp(this.heap.length - 1);
  }

  pop() {
    if (this.heap.length === 0) return null;
    const top = this.heap[0];
    const last = this.heap.pop();
    if (this.heap.length > 0) {
      this.heap[0] = last;
      this._bubbleDown(0);
    }
    return top;
  }

  size() { return this.heap.length; }

  _bubbleUp(idx) {
    while (idx > 0) {
      const parent = Math.floor((idx - 1) / 2);
      if (this.compare(this.heap[idx], this.heap[parent]) >= 0) break;
      [this.heap[idx], this.heap[parent]] = [this.heap[parent], this.heap[idx]];
      idx = parent;
    }
  }

  _bubbleDown(idx) {
    while (true) {
      const left = 2 * idx + 1, right = 2 * idx + 2;
      let smallest = idx;
      if (left < this.heap.length && this.compare(this.heap[left], this.heap[smallest]) < 0) smallest = left;
      if (right < this.heap.length && this.compare(this.heap[right], this.heap[smallest]) < 0) smallest = right;
      if (smallest === idx) break;
      [this.heap[idx], this.heap[smallest]] = [this.heap[smallest], this.heap[idx]];
      idx = smallest;
    }
  }
}

function mergeKListsHeap(lists) {
  const dummy = new ListNode(0);
  let curr = dummy;

  const heap = new MinHeap((a, b) => a.val - b.val);

  for (const node of lists) {
    if (node) heap.push(node);
  }

  while (heap.size() > 0) {
    const node = heap.pop();
    curr.next = node;
    curr = curr.next;
    if (node.next) {
      heap.push(node.next);
    }
  }

  return dummy.next;
}
```

---

### 5. Intersection and Union

**Find Intersection Point:**

**Python:**
```python
def get_intersection_node(headA, headB):
    if not headA or not headB:
        return None

    pA, pB = headA, headB

    while pA != pB:
        pA = pA.next if pA else headB
        pB = pB.next if pB else headA

    return pA
```

**JavaScript:**
```javascript
function getIntersectionNode(headA, headB) {
  if (!headA || !headB) return null;

  let pA = headA, pB = headB;

  while (pA !== pB) {
    pA = pA ? pA.next : headB;
    pB = pB ? pB.next : headA;
  }

  return pA;
}
```

---

### 6. Palindrome Check

**Python:**
```python
def is_palindrome(head):
    # Find middle
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

    # Reverse second half
    prev = None
    while slow:
        next_temp = slow.next
        slow.next = prev
        prev = slow
        slow = next_temp

    # Compare halves
    left, right = head, prev
    while right:
        if left.val != right.val:
            return False
        left = left.next
        right = right.next

    return True
```

**JavaScript:**
```javascript
function isPalindrome(head) {
  // Find middle
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow.next;
    fast = fast.next.next;
  }

  // Reverse second half
  let prev = null;
  while (slow) {
    const nextTemp = slow.next;
    slow.next = prev;
    prev = slow;
    slow = nextTemp;
  }

  // Compare halves
  let left = head, right = prev;
  while (right) {
    if (left.val !== right.val) return false;
    left = left.next;
    right = right.next;
  }

  return true;
}
```

---

### 7. Reorder List

**Python:**
```python
def reorder_list(head):
    """Reorder: L0 → Ln → L1 → Ln-1 → L2 → Ln-2 → ..."""
    if not head or not head.next:
        return

    # Find middle
    slow = fast = head
    while fast.next and fast.next.next:
        slow = slow.next
        fast = fast.next.next

    # Reverse second half
    prev = None
    curr = slow.next
    slow.next = None
    while curr:
        next_temp = curr.next
        curr.next = prev
        prev = curr
        curr = next_temp

    # Merge two halves
    first, second = head, prev
    while second:
        tmp1, tmp2 = first.next, second.next
        first.next = second
        second.next = tmp1
        first = tmp1
        second = tmp2
```

**JavaScript:**
```javascript
function reorderList(head) {
  // Reorder: L0 → Ln → L1 → Ln-1 → L2 → Ln-2 → ...
  if (!head || !head.next) return;

  // Find middle
  let slow = head, fast = head;
  while (fast.next && fast.next.next) {
    slow = slow.next;
    fast = fast.next.next;
  }

  // Reverse second half
  let prev = null;
  let curr = slow.next;
  slow.next = null;
  while (curr) {
    const nextTemp = curr.next;
    curr.next = prev;
    prev = curr;
    curr = nextTemp;
  }

  // Merge two halves
  let first = head, second = prev;
  while (second) {
    const tmp1 = first.next, tmp2 = second.next;
    first.next = second;
    second.next = tmp1;
    first = tmp1;
    second = tmp2;
  }
}
```

---

### 8. Copy List with Random Pointer

**Python:**
```python
def copy_random_list(head):
    if not head:
        return None

    # Method 1: Hash Map - O(n) space
    old_to_new = {}
    curr = head
    while curr:
        old_to_new[curr] = Node(curr.val)
        curr = curr.next

    curr = head
    while curr:
        old_to_new[curr].next = old_to_new.get(curr.next)
        old_to_new[curr].random = old_to_new.get(curr.random)
        curr = curr.next

    return old_to_new[head]

def copy_random_list_o1_space(head):
    """O(1) space - interleave nodes"""
    if not head:
        return None

    # Step 1: Create interleaved list
    curr = head
    while curr:
        new_node = Node(curr.val, curr.next)
        curr.next = new_node
        curr = new_node.next

    # Step 2: Copy random pointers
    curr = head
    while curr:
        if curr.random:
            curr.next.random = curr.random.next
        curr = curr.next.next

    # Step 3: Separate lists
    dummy = Node(0)
    new_curr = dummy
    curr = head
    while curr:
        new_curr.next = curr.next
        new_curr = new_curr.next
        curr.next = curr.next.next
        curr = curr.next

    return dummy.next
```

**JavaScript:**
```javascript
// Node with random pointer
class NodeWithRandom {
  constructor(val = 0, next = null, random = null) {
    this.val = val;
    this.next = next;
    this.random = random;
  }
}

// Method 1: Hash Map - O(n) space
function copyRandomList(head) {
  if (!head) return null;

  const oldToNew = new Map();
  let curr = head;

  // First pass: create all new nodes
  while (curr) {
    oldToNew.set(curr, new NodeWithRandom(curr.val));
    curr = curr.next;
  }

  // Second pass: set next and random pointers
  curr = head;
  while (curr) {
    oldToNew.get(curr).next = oldToNew.get(curr.next) || null;
    oldToNew.get(curr).random = oldToNew.get(curr.random) || null;
    curr = curr.next;
  }

  return oldToNew.get(head);
}

// Method 2: O(1) space - interleave nodes
function copyRandomListO1Space(head) {
  if (!head) return null;

  // Step 1: Create interleaved list
  let curr = head;
  while (curr) {
    const newNode = new NodeWithRandom(curr.val, curr.next);
    curr.next = newNode;
    curr = newNode.next;
  }

  // Step 2: Copy random pointers
  curr = head;
  while (curr) {
    if (curr.random) {
      curr.next.random = curr.random.next;
    }
    curr = curr.next.next;
  }

  // Step 3: Separate lists
  const dummy = new NodeWithRandom(0);
  let newCurr = dummy;
  curr = head;
  while (curr) {
    newCurr.next = curr.next;
    newCurr = newCurr.next;
    curr.next = curr.next.next;
    curr = curr.next;
  }

  return dummy.next;
}
```

---

## LRU Cache Implementation

**Python:**
```python
class LRUCache:
    """Lead-level: Implement LRU Cache using doubly linked list + hash map"""

    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key -> node

        # Dummy head and tail
        self.head = DoublyListNode(0)
        self.tail = DoublyListNode(0)
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node):
        prev_node = node.prev
        next_node = node.next
        prev_node.next = next_node
        next_node.prev = prev_node

    def _add_to_head(self, node):
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def get(self, key):
        if key in self.cache:
            node = self.cache[key]
            self._remove(node)
            self._add_to_head(node)
            return node.val
        return -1

    def put(self, key, value):
        if key in self.cache:
            self._remove(self.cache[key])

        node = DoublyListNode(key, value)  # Modified to store key too
        self._add_to_head(node)
        self.cache[key] = node

        if len(self.cache) > self.capacity:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```

**JavaScript:**
```javascript
class LRUNode {
  constructor(key, val) {
    this.key = key;
    this.val = val;
    this.prev = null;
    this.next = null;
  }
}

class LRUCache {
  // Lead-level: Implement LRU Cache using doubly linked list + hash map

  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map(); // key -> node

    // Dummy head and tail
    this.head = new LRUNode(0, 0);
    this.tail = new LRUNode(0, 0);
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  _remove(node) {
    const prevNode = node.prev;
    const nextNode = node.next;
    prevNode.next = nextNode;
    nextNode.prev = prevNode;
  }

  _addToHead(node) {
    node.prev = this.head;
    node.next = this.head.next;
    this.head.next.prev = node;
    this.head.next = node;
  }

  get(key) {
    if (this.cache.has(key)) {
      const node = this.cache.get(key);
      this._remove(node);
      this._addToHead(node);
      return node.val;
    }
    return -1;
  }

  put(key, value) {
    if (this.cache.has(key)) {
      this._remove(this.cache.get(key));
    }

    const node = new LRUNode(key, value);
    this._addToHead(node);
    this.cache.set(key, node);

    if (this.cache.size > this.capacity) {
      const lru = this.tail.prev;
      this._remove(lru);
      this.cache.delete(lru.key);
    }
  }
}

// Usage example:
// const lru = new LRUCache(2);
// lru.put(1, 1);
// lru.put(2, 2);
// lru.get(1);    // returns 1
// lru.put(3, 3); // evicts key 2
// lru.get(2);    // returns -1
```

---

## Must-Know Problems

### Easy
1. Reverse Linked List
2. Merge Two Sorted Lists
3. Linked List Cycle
4. Remove Duplicates from Sorted List
5. Intersection of Two Linked Lists

### Medium
1. Add Two Numbers
2. Remove Nth Node From End
3. Swap Nodes in Pairs
4. Reorder List
5. Sort List (Merge Sort)
6. Copy List with Random Pointer
7. Linked List Cycle II (find start)
8. Odd Even Linked List

### Hard
1. Reverse Nodes in k-Group
2. Merge k Sorted Lists
3. LRU Cache
4. LFU Cache

---

## Common Edge Cases

1. **Empty list** - `head is None`
2. **Single node** - `head.next is None`
3. **Two nodes** - Important for reversal problems
4. **Cycle at head** - Entire list is a cycle
5. **Cycle at tail** - Last node points to itself

---

## Interview Tips

1. **Always ask:** Can I modify the input? Do I need to restore it?
2. **Draw it out:** Visualize pointer changes
3. **Use dummy head:** Simplifies edge cases
4. **Handle null checks:** Prevent null pointer exceptions
5. **Test incrementally:** Walk through with simple example

---

## Practice Checklist

- [ ] Can reverse a linked list in < 3 minutes
- [ ] Understand Floyd's cycle detection proof
- [ ] Can implement LRU Cache from scratch
- [ ] Know when to use dummy head
- [ ] Can handle all edge cases without bugs
