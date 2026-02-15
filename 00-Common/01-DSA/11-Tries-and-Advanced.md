# Tries and Advanced Data Structures

## Overview

Tries are specialized tree structures for string operations. This section also covers advanced data structures like Segment Trees, Fenwick Trees, and LRU/LFU Caches that may appear in Lead-level interviews.

---

## Trie (Prefix Tree)

### Key Concepts

```
Trie Properties:
- Each node represents a character
- Root is empty
- Paths from root to marked nodes form words
- Common prefixes share path

Time Complexity:
- Insert: O(m) where m is word length
- Search: O(m)
- Prefix search: O(m)
- Space: O(alphabet_size * m * n) for n words
```

### Implementation

**Python:**
```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False
        self.count = 0  # Optional: word count
        self.prefix_count = 0  # Optional: prefix count

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        """Insert word into trie - O(m)"""
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
            node.prefix_count += 1
        node.is_end = True
        node.count += 1

    def search(self, word):
        """Check if word exists - O(m)"""
        node = self._find_node(word)
        return node is not None and node.is_end

    def starts_with(self, prefix):
        """Check if any word starts with prefix - O(m)"""
        return self._find_node(prefix) is not None

    def _find_node(self, prefix):
        """Find node for given prefix"""
        node = self.root
        for char in prefix:
            if char not in node.children:
                return None
            node = node.children[char]
        return node

    def delete(self, word):
        """Delete word from trie"""
        def _delete(node, word, index):
            if index == len(word):
                if not node.is_end:
                    return False
                node.is_end = False
                node.count -= 1
                return len(node.children) == 0

            char = word[index]
            if char not in node.children:
                return False

            should_delete = _delete(node.children[char], word, index + 1)

            if should_delete:
                del node.children[char]
                return not node.is_end and len(node.children) == 0

            node.children[char].prefix_count -= 1
            return False

        _delete(self.root, word, 0)

    def get_all_words(self, prefix=""):
        """Get all words with given prefix"""
        result = []
        node = self._find_node(prefix)

        if node is None:
            return result

        def dfs(node, path):
            if node.is_end:
                result.append(path)
            for char, child in node.children.items():
                dfs(child, path + char)

        dfs(node, prefix)
        return result

    def count_words_with_prefix(self, prefix):
        """Count words starting with prefix"""
        node = self._find_node(prefix)
        return node.prefix_count if node else 0
```

**JavaScript:**
```javascript
class TrieNode {
  constructor() {
    this.children = new Map();
    this.isEnd = false;
    this.count = 0;
    this.prefixCount = 0;
  }
}

class Trie {
  constructor() {
    this.root = new TrieNode();
  }

  insert(word) {
    // Insert word into trie - O(m)
    let node = this.root;
    for (const char of word) {
      if (!node.children.has(char)) {
        node.children.set(char, new TrieNode());
      }
      node = node.children.get(char);
      node.prefixCount++;
    }
    node.isEnd = true;
    node.count++;
  }

  search(word) {
    // Check if word exists - O(m)
    const node = this._findNode(word);
    return node !== null && node.isEnd;
  }

  startsWith(prefix) {
    // Check if any word starts with prefix - O(m)
    return this._findNode(prefix) !== null;
  }

  _findNode(prefix) {
    // Find node for given prefix
    let node = this.root;
    for (const char of prefix) {
      if (!node.children.has(char)) {
        return null;
      }
      node = node.children.get(char);
    }
    return node;
  }

  getAllWords(prefix = "") {
    // Get all words with given prefix
    const result = [];
    const node = this._findNode(prefix);

    if (node === null) return result;

    function dfs(node, path) {
      if (node.isEnd) {
        result.push(path);
      }
      for (const [char, child] of node.children) {
        dfs(child, path + char);
      }
    }

    dfs(node, prefix);
    return result;
  }

  countWordsWithPrefix(prefix) {
    // Count words starting with prefix
    const node = this._findNode(prefix);
    return node ? node.prefixCount : 0;
  }
}
```

### Space-Optimized Trie (Array-based)

```python
class TrieArrayBased:
    """More memory efficient for lowercase letters only"""
    def __init__(self):
        self.root = [None] * 27  # 26 letters + is_end flag

    def insert(self, word):
        node = self.root
        for char in word:
            idx = ord(char) - ord('a')
            if node[idx] is None:
                node[idx] = [None] * 27
            node = node[idx]
        node[26] = True  # Mark end

    def search(self, word):
        node = self.root
        for char in word:
            idx = ord(char) - ord('a')
            if node[idx] is None:
                return False
            node = node[idx]
        return node[26] is True
```

---

## Trie Problems

### Word Search II

```python
def find_words(board, words):
    """Find all words from dictionary in the board"""
    # Build trie from words
    trie = Trie()
    for word in words:
        trie.insert(word)

    rows, cols = len(board), len(board[0])
    result = set()

    def backtrack(r, c, node, path):
        char = board[r][c]

        if char not in node.children:
            return

        node = node.children[char]
        path += char

        if node.is_end:
            result.add(path)
            # Don't return - might have longer words

        # Mark visited
        board[r][c] = '#'

        for dr, dc in [(0, 1), (0, -1), (1, 0), (-1, 0)]:
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and board[nr][nc] != '#':
                backtrack(nr, nc, node, path)

        # Restore
        board[r][c] = char

    for r in range(rows):
        for c in range(cols):
            backtrack(r, c, trie.root, "")

    return list(result)
```

### Design Add and Search Words

```python
class WordDictionary:
    """Support wildcard '.' in search"""
    def __init__(self):
        self.root = TrieNode()

    def addWord(self, word):
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_end = True

    def search(self, word):
        def dfs(node, index):
            if index == len(word):
                return node.is_end

            char = word[index]

            if char == '.':
                for child in node.children.values():
                    if dfs(child, index + 1):
                        return True
                return False
            else:
                if char not in node.children:
                    return False
                return dfs(node.children[char], index + 1)

        return dfs(self.root, 0)
```

### Autocomplete / Search Suggestions

```python
class AutocompleteSystem:
    def __init__(self, sentences, times):
        self.trie = {}
        self.current_input = ""

        for sentence, count in zip(sentences, times):
            self._insert(sentence, count)

    def _insert(self, sentence, count):
        node = self.trie
        for char in sentence:
            if char not in node:
                node[char] = {}
            node = node[char]
        node['#'] = node.get('#', 0) + count

    def _search(self, prefix):
        """Get all sentences with prefix and their counts"""
        node = self.trie
        for char in prefix:
            if char not in node:
                return []
            node = node[char]

        # DFS to find all sentences
        result = []
        def dfs(node, path):
            if '#' in node:
                result.append((path, node['#']))
            for char, child in node.items():
                if char != '#':
                    dfs(child, path + char)

        dfs(node, prefix)
        return result

    def input(self, c):
        if c == '#':
            self._insert(self.current_input, 1)
            self.current_input = ""
            return []

        self.current_input += c
        suggestions = self._search(self.current_input)

        # Sort by frequency (desc) then lexicographically
        suggestions.sort(key=lambda x: (-x[1], x[0]))

        return [s[0] for s in suggestions[:3]]
```

---

## Segment Tree

```python
class SegmentTree:
    """
    Range queries and point updates in O(log n)
    Used for: range sum, range min/max, range GCD, etc.
    """
    def __init__(self, arr):
        self.n = len(arr)
        self.tree = [0] * (4 * self.n)
        self._build(arr, 0, 0, self.n - 1)

    def _build(self, arr, node, start, end):
        if start == end:
            self.tree[node] = arr[start]
        else:
            mid = (start + end) // 2
            self._build(arr, 2*node + 1, start, mid)
            self._build(arr, 2*node + 2, mid + 1, end)
            self.tree[node] = self.tree[2*node + 1] + self.tree[2*node + 2]

    def update(self, idx, val):
        """Update arr[idx] to val - O(log n)"""
        self._update(0, 0, self.n - 1, idx, val)

    def _update(self, node, start, end, idx, val):
        if start == end:
            self.tree[node] = val
        else:
            mid = (start + end) // 2
            if idx <= mid:
                self._update(2*node + 1, start, mid, idx, val)
            else:
                self._update(2*node + 2, mid + 1, end, idx, val)
            self.tree[node] = self.tree[2*node + 1] + self.tree[2*node + 2]

    def query(self, left, right):
        """Sum of arr[left:right+1] - O(log n)"""
        return self._query(0, 0, self.n - 1, left, right)

    def _query(self, node, start, end, left, right):
        if right < start or left > end:
            return 0  # Out of range
        if left <= start and end <= right:
            return self.tree[node]  # Fully in range

        mid = (start + end) // 2
        left_sum = self._query(2*node + 1, start, mid, left, right)
        right_sum = self._query(2*node + 2, mid + 1, end, left, right)
        return left_sum + right_sum
```

### Segment Tree with Lazy Propagation

```python
class SegmentTreeLazy:
    """Support range updates in O(log n)"""
    def __init__(self, arr):
        self.n = len(arr)
        self.tree = [0] * (4 * self.n)
        self.lazy = [0] * (4 * self.n)
        self._build(arr, 0, 0, self.n - 1)

    def _build(self, arr, node, start, end):
        if start == end:
            self.tree[node] = arr[start]
        else:
            mid = (start + end) // 2
            self._build(arr, 2*node + 1, start, mid)
            self._build(arr, 2*node + 2, mid + 1, end)
            self.tree[node] = self.tree[2*node + 1] + self.tree[2*node + 2]

    def _propagate(self, node, start, end):
        if self.lazy[node] != 0:
            self.tree[node] += self.lazy[node] * (end - start + 1)
            if start != end:
                self.lazy[2*node + 1] += self.lazy[node]
                self.lazy[2*node + 2] += self.lazy[node]
            self.lazy[node] = 0

    def range_update(self, left, right, val):
        """Add val to all elements in [left, right]"""
        self._range_update(0, 0, self.n - 1, left, right, val)

    def _range_update(self, node, start, end, left, right, val):
        self._propagate(node, start, end)

        if right < start or left > end:
            return
        if left <= start and end <= right:
            self.lazy[node] += val
            self._propagate(node, start, end)
            return

        mid = (start + end) // 2
        self._range_update(2*node + 1, start, mid, left, right, val)
        self._range_update(2*node + 2, mid + 1, end, left, right, val)
        self.tree[node] = self.tree[2*node + 1] + self.tree[2*node + 2]
```

---

## Fenwick Tree (Binary Indexed Tree)

```python
class FenwickTree:
    """
    More space efficient than segment tree
    Point update + prefix sum in O(log n)
    """
    def __init__(self, n):
        self.n = n
        self.tree = [0] * (n + 1)  # 1-indexed

    @classmethod
    def from_array(cls, arr):
        """Build from array in O(n)"""
        bit = cls(len(arr))
        for i, val in enumerate(arr):
            bit.update(i, val)
        return bit

    def update(self, idx, delta):
        """Add delta to arr[idx] - O(log n)"""
        idx += 1  # 1-indexed
        while idx <= self.n:
            self.tree[idx] += delta
            idx += idx & (-idx)  # Add LSB

    def prefix_sum(self, idx):
        """Sum of arr[0:idx+1] - O(log n)"""
        idx += 1  # 1-indexed
        total = 0
        while idx > 0:
            total += self.tree[idx]
            idx -= idx & (-idx)  # Remove LSB
        return total

    def range_sum(self, left, right):
        """Sum of arr[left:right+1]"""
        return self.prefix_sum(right) - (self.prefix_sum(left - 1) if left > 0 else 0)
```

### Count Smaller Numbers After Self

```python
def count_smaller(nums):
    """Count elements smaller than each element to its right"""
    # Coordinate compression
    sorted_nums = sorted(set(nums))
    rank = {num: i + 1 for i, num in enumerate(sorted_nums)}

    bit = FenwickTree(len(sorted_nums))
    result = []

    # Process from right to left
    for num in reversed(nums):
        r = rank[num]
        result.append(bit.prefix_sum(r - 1))
        bit.update(r - 1, 1)

    return result[::-1]
```

---

## LRU Cache

```python
class LRUCache:
    """
    O(1) get and put using OrderedDict
    or Hash Map + Doubly Linked List
    """
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)

# Manual implementation with doubly linked list
class LRUCacheManual:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}

        # Dummy head and tail
        self.head = DListNode(0, 0)
        self.tail = DListNode(0, 0)
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node):
        prev, next = node.prev, node.next
        prev.next, next.prev = next, prev

    def _add_to_head(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key):
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._add_to_head(node)
        return node.val

    def put(self, key, value):
        if key in self.cache:
            self._remove(self.cache[key])
        node = DListNode(key, value)
        self.cache[key] = node
        self._add_to_head(node)

        if len(self.cache) > self.capacity:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]

class DListNode:
    def __init__(self, key, val):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None
```

---

## LFU Cache

```python
from collections import defaultdict, OrderedDict

class LFUCache:
    """Least Frequently Used Cache - O(1) operations"""
    def __init__(self, capacity):
        self.capacity = capacity
        self.min_freq = 0
        self.key_to_val = {}
        self.key_to_freq = {}
        self.freq_to_keys = defaultdict(OrderedDict)

    def get(self, key):
        if key not in self.key_to_val:
            return -1

        freq = self.key_to_freq[key]
        del self.freq_to_keys[freq][key]

        if not self.freq_to_keys[freq]:
            del self.freq_to_keys[freq]
            if self.min_freq == freq:
                self.min_freq += 1

        self.key_to_freq[key] = freq + 1
        self.freq_to_keys[freq + 1][key] = None

        return self.key_to_val[key]

    def put(self, key, value):
        if self.capacity <= 0:
            return

        if key in self.key_to_val:
            self.key_to_val[key] = value
            self.get(key)  # Update frequency
            return

        if len(self.key_to_val) >= self.capacity:
            # Evict LFU (and LRU among ties)
            lfu_key, _ = self.freq_to_keys[self.min_freq].popitem(last=False)
            del self.key_to_val[lfu_key]
            del self.key_to_freq[lfu_key]

        self.key_to_val[key] = value
        self.key_to_freq[key] = 1
        self.freq_to_keys[1][key] = None
        self.min_freq = 1
```

---

## Must-Know Problems

### Trie
1. Implement Trie
2. Add and Search Words
3. Word Search II
4. Replace Words
5. Design Search Autocomplete System
6. Palindrome Pairs

### Advanced DS
1. LRU Cache
2. LFU Cache
3. Range Sum Query - Mutable (Segment Tree/BIT)
4. Count of Smaller Numbers After Self
5. Range Module

---

## Practice Checklist

- [ ] Can implement Trie from scratch
- [ ] Understand when to use Trie vs Hash
- [ ] Know Segment Tree for range queries
- [ ] Can implement LRU Cache both ways
- [ ] Understand time/space trade-offs
