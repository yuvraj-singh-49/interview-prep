# Hash Tables

## Overview

Hash tables provide O(1) average-case lookup, insertion, and deletion. As a Lead engineer, you should understand the underlying implementation, collision handling, and when hash tables are the optimal choice.

---

## Key Concepts

### Time Complexity

| Operation | Average | Worst Case |
|-----------|---------|------------|
| Search | O(1) | O(n) |
| Insert | O(1) | O(n) |
| Delete | O(1) | O(n) |
| Space | O(n) | O(n) |

### Hash Function Properties

1. **Deterministic**: Same input always produces same output
2. **Uniform Distribution**: Keys spread evenly across buckets
3. **Fast Computation**: O(1) to compute hash
4. **Minimize Collisions**: Different keys should produce different hashes

### Load Factor

```
Load Factor (α) = n / m
n = number of entries
m = number of buckets

Typical threshold: 0.75
When exceeded: Rehash (double size, redistribute)
```

---

## Collision Resolution

### 1. Chaining (Separate Chaining)

**Python:**
```python
class HashTableChaining:
    def __init__(self, size=16):
        self.size = size
        self.buckets = [[] for _ in range(size)]
        self.count = 0

    def _hash(self, key):
        return hash(key) % self.size

    def put(self, key, value):
        idx = self._hash(key)
        bucket = self.buckets[idx]

        for i, (k, v) in enumerate(bucket):
            if k == key:
                bucket[i] = (key, value)
                return

        bucket.append((key, value))
        self.count += 1

        # Rehash if load factor exceeds 0.75
        if self.count / self.size > 0.75:
            self._rehash()

    def get(self, key):
        idx = self._hash(key)
        for k, v in self.buckets[idx]:
            if k == key:
                return v
        return None

    def remove(self, key):
        idx = self._hash(key)
        bucket = self.buckets[idx]
        for i, (k, v) in enumerate(bucket):
            if k == key:
                bucket.pop(i)
                self.count -= 1
                return True
        return False

    def _rehash(self):
        old_buckets = self.buckets
        self.size *= 2
        self.buckets = [[] for _ in range(self.size)]
        self.count = 0
        for bucket in old_buckets:
            for key, value in bucket:
                self.put(key, value)
```

**JavaScript:**
```javascript
class HashTableChaining {
  constructor(size = 16) {
    this.size = size;
    this.buckets = Array.from({ length: size }, () => []);
    this.count = 0;
  }

  _hash(key) {
    // Simple hash for strings
    let hash = 0;
    const str = String(key);
    for (let i = 0; i < str.length; i++) {
      hash = (hash * 31 + str.charCodeAt(i)) % this.size;
    }
    return hash;
  }

  put(key, value) {
    const idx = this._hash(key);
    const bucket = this.buckets[idx];

    for (let i = 0; i < bucket.length; i++) {
      if (bucket[i][0] === key) {
        bucket[i] = [key, value];
        return;
      }
    }

    bucket.push([key, value]);
    this.count++;

    // Rehash if load factor exceeds 0.75
    if (this.count / this.size > 0.75) {
      this._rehash();
    }
  }

  get(key) {
    const idx = this._hash(key);
    for (const [k, v] of this.buckets[idx]) {
      if (k === key) return v;
    }
    return null;
  }

  remove(key) {
    const idx = this._hash(key);
    const bucket = this.buckets[idx];
    for (let i = 0; i < bucket.length; i++) {
      if (bucket[i][0] === key) {
        bucket.splice(i, 1);
        this.count--;
        return true;
      }
    }
    return false;
  }

  _rehash() {
    const oldBuckets = this.buckets;
    this.size *= 2;
    this.buckets = Array.from({ length: this.size }, () => []);
    this.count = 0;
    for (const bucket of oldBuckets) {
      for (const [key, value] of bucket) {
        this.put(key, value);
      }
    }
  }
}
```

### 2. Open Addressing (Linear Probing)

**Python:**
```python
class HashTableOpenAddressing:
    def __init__(self, size=16):
        self.size = size
        self.keys = [None] * size
        self.values = [None] * size
        self.DELETED = object()  # Tombstone marker
        self.count = 0

    def _hash(self, key):
        return hash(key) % self.size

    def _probe(self, key):
        """Linear probing"""
        idx = self._hash(key)
        while self.keys[idx] is not None:
            if self.keys[idx] == key:
                return idx
            idx = (idx + 1) % self.size
        return idx

    def put(self, key, value):
        if self.count / self.size > 0.5:  # Lower threshold for open addressing
            self._rehash()

        idx = self._hash(key)
        while self.keys[idx] is not None and self.keys[idx] != self.DELETED:
            if self.keys[idx] == key:
                self.values[idx] = value
                return
            idx = (idx + 1) % self.size

        self.keys[idx] = key
        self.values[idx] = value
        self.count += 1

    def get(self, key):
        idx = self._hash(key)
        while self.keys[idx] is not None:
            if self.keys[idx] == key:
                return self.values[idx]
            idx = (idx + 1) % self.size
        return None

    def remove(self, key):
        idx = self._hash(key)
        while self.keys[idx] is not None:
            if self.keys[idx] == key:
                self.keys[idx] = self.DELETED
                self.values[idx] = None
                self.count -= 1
                return True
            idx = (idx + 1) % self.size
        return False
```

**JavaScript:**
```javascript
class HashTableOpenAddressing {
  constructor(size = 16) {
    this.size = size;
    this.keys = new Array(size).fill(null);
    this.values = new Array(size).fill(null);
    this.DELETED = Symbol('DELETED'); // Tombstone marker
    this.count = 0;
  }

  _hash(key) {
    let hash = 0;
    const str = String(key);
    for (let i = 0; i < str.length; i++) {
      hash = (hash * 31 + str.charCodeAt(i)) % this.size;
    }
    return hash;
  }

  put(key, value) {
    if (this.count / this.size > 0.5) { // Lower threshold for open addressing
      this._rehash();
    }

    let idx = this._hash(key);
    while (this.keys[idx] !== null && this.keys[idx] !== this.DELETED) {
      if (this.keys[idx] === key) {
        this.values[idx] = value;
        return;
      }
      idx = (idx + 1) % this.size;
    }

    this.keys[idx] = key;
    this.values[idx] = value;
    this.count++;
  }

  get(key) {
    let idx = this._hash(key);
    while (this.keys[idx] !== null) {
      if (this.keys[idx] === key) {
        return this.values[idx];
      }
      idx = (idx + 1) % this.size;
    }
    return null;
  }

  remove(key) {
    let idx = this._hash(key);
    while (this.keys[idx] !== null) {
      if (this.keys[idx] === key) {
        this.keys[idx] = this.DELETED;
        this.values[idx] = null;
        this.count--;
        return true;
      }
      idx = (idx + 1) % this.size;
    }
    return false;
  }

  _rehash() {
    const oldKeys = this.keys;
    const oldValues = this.values;
    this.size *= 2;
    this.keys = new Array(this.size).fill(null);
    this.values = new Array(this.size).fill(null);
    this.count = 0;

    for (let i = 0; i < oldKeys.length; i++) {
      if (oldKeys[i] !== null && oldKeys[i] !== this.DELETED) {
        this.put(oldKeys[i], oldValues[i]);
      }
    }
  }
}
```

---

## Python Collections / JavaScript Equivalents

### dict (Hash Table) / Map & Object

**Python:**
```python
# Creation
d = {}
d = dict()
d = {'a': 1, 'b': 2}
d = dict(a=1, b=2)

# Operations
d['key'] = value          # Set - O(1)
value = d['key']          # Get - O(1), raises KeyError if missing
value = d.get('key', default)  # Get with default - O(1)
del d['key']              # Delete - O(1)
'key' in d                # Check existence - O(1)

# Iteration
for key in d:             # Iterate keys
for key, value in d.items():  # Iterate pairs
for value in d.values():  # Iterate values

# Useful methods
d.setdefault('key', []).append(value)  # Initialize if missing
d.pop('key', default)     # Remove and return
d.update(other_dict)      # Merge dictionaries
```

**JavaScript:**
```javascript
// Using Map (preferred for hash table operations)
const map = new Map();
map.set('key', value);           // Set - O(1)
const value = map.get('key');    // Get - O(1), returns undefined if missing
map.get('key') ?? defaultVal;    // Get with default
map.delete('key');               // Delete - O(1)
map.has('key');                  // Check existence - O(1)

// Iteration
for (const key of map.keys()) {}
for (const [key, value] of map.entries()) {}
for (const value of map.values()) {}

// Using Object (for string keys)
const obj = {};
obj['key'] = value;
const val = obj['key'] ?? defaultVal;
delete obj['key'];
'key' in obj;

// Useful patterns
if (!map.has('key')) map.set('key', []);
map.get('key').push(value);

// Merge objects
const merged = { ...obj1, ...obj2 };
```

### collections.defaultdict / Custom DefaultMap

**Python:**
```python
from collections import defaultdict

# Auto-initialize missing keys
word_count = defaultdict(int)
word_count['hello'] += 1  # No KeyError

# Group items
groups = defaultdict(list)
for item in items:
    groups[item.category].append(item)

# Nested defaultdict
nested = defaultdict(lambda: defaultdict(int))
nested['a']['b'] += 1
```

**JavaScript:**
```javascript
// Custom DefaultMap class
class DefaultMap extends Map {
  constructor(defaultFactory) {
    super();
    this.defaultFactory = defaultFactory;
  }

  get(key) {
    if (!this.has(key)) {
      this.set(key, this.defaultFactory());
    }
    return super.get(key);
  }
}

// Usage - auto-initialize missing keys
const wordCount = new DefaultMap(() => 0);
wordCount.set('hello', wordCount.get('hello') + 1);

// Group items
const groups = new DefaultMap(() => []);
for (const item of items) {
  groups.get(item.category).push(item);
}

// Alternative: Using plain objects with nullish coalescing
const count = {};
count['hello'] = (count['hello'] ?? 0) + 1;

// Group items without DefaultMap
const groups2 = {};
for (const item of items) {
  (groups2[item.category] ??= []).push(item);
}
```

### collections.Counter / Frequency Map

**Python:**
```python
from collections import Counter

# Frequency counting
counter = Counter(['a', 'b', 'a', 'c', 'a', 'b'])
# Counter({'a': 3, 'b': 2, 'c': 1})

counter = Counter("abracadabra")
# Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})

# Common operations
counter.most_common(2)    # [('a', 5), ('b', 2)]
counter['a']              # 5
counter['z']              # 0 (not KeyError)
counter.update(['a', 'b'])  # Add more elements

# Arithmetic
counter1 + counter2       # Combine counts
counter1 - counter2       # Subtract (keeps positive)
counter1 & counter2       # Intersection (min)
counter1 | counter2       # Union (max)
```

**JavaScript:**
```javascript
// Counter class implementation
class Counter extends Map {
  constructor(iterable) {
    super();
    if (iterable) {
      for (const item of iterable) {
        this.set(item, (this.get(item) ?? 0) + 1);
      }
    }
  }

  get(key) {
    return super.get(key) ?? 0;
  }

  mostCommon(n) {
    return [...this.entries()]
      .sort((a, b) => b[1] - a[1])
      .slice(0, n);
  }

  update(iterable) {
    for (const item of iterable) {
      this.set(item, this.get(item) + 1);
    }
  }
}

// Usage
const counter = new Counter(['a', 'b', 'a', 'c', 'a', 'b']);
// Map { 'a' => 3, 'b' => 2, 'c' => 1 }

const strCounter = new Counter("abracadabra");
counter.mostCommon(2);    // [['a', 5], ['b', 2]]
counter.get('z');         // 0 (not undefined)

// Simple frequency counting without class
function countFrequency(arr) {
  const freq = new Map();
  for (const item of arr) {
    freq.set(item, (freq.get(item) ?? 0) + 1);
  }
  return freq;
}
```

### collections.OrderedDict / Map (maintains insertion order)

**Python:**
```python
from collections import OrderedDict

od = OrderedDict()
od['a'] = 1
od['b'] = 2
od['c'] = 3

# Maintains insertion order (Python 3.7+ dict does too)
# But has special methods:
od.move_to_end('a')       # Move 'a' to end
od.move_to_end('c', last=False)  # Move 'c' to beginning
od.popitem(last=True)     # Pop last item
od.popitem(last=False)    # Pop first item
```

**JavaScript:**
```javascript
// Map maintains insertion order by default
const map = new Map();
map.set('a', 1);
map.set('b', 2);
map.set('c', 3);

// OrderedMap with move operations
class OrderedMap extends Map {
  moveToEnd(key) {
    if (this.has(key)) {
      const value = this.get(key);
      this.delete(key);
      this.set(key, value);
    }
  }

  moveToBeginning(key) {
    if (this.has(key)) {
      const value = this.get(key);
      const entries = [...this.entries()];
      this.clear();
      this.set(key, value);
      for (const [k, v] of entries) {
        if (k !== key) this.set(k, v);
      }
    }
  }

  popLast() {
    const keys = [...this.keys()];
    if (keys.length === 0) return undefined;
    const lastKey = keys[keys.length - 1];
    const value = this.get(lastKey);
    this.delete(lastKey);
    return [lastKey, value];
  }

  popFirst() {
    const firstKey = this.keys().next().value;
    if (firstKey === undefined) return undefined;
    const value = this.get(firstKey);
    this.delete(firstKey);
    return [firstKey, value];
  }
}
```

---

## Essential Patterns

### 1. Two Sum Pattern

**Python:**
```python
def two_sum(nums, target):
    """Find indices of two numbers that sum to target"""
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []

def three_sum(nums):
    """Find all unique triplets that sum to zero"""
    nums.sort()
    result = []

    for i in range(len(nums) - 2):
        if i > 0 and nums[i] == nums[i-1]:
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

**JavaScript:**
```javascript
function twoSum(nums, target) {
  // Find indices of two numbers that sum to target
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

function threeSum(nums) {
  // Find all unique triplets that sum to zero
  nums.sort((a, b) => a - b);
  const result = [];

  for (let i = 0; i < nums.length - 2; i++) {
    if (i > 0 && nums[i] === nums[i - 1]) continue;

    let left = i + 1, right = nums.length - 1;
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

### 2. Frequency Counting

**Python:**
```python
def top_k_frequent(nums, k):
    """Find k most frequent elements - O(n)"""
    count = Counter(nums)

    # Bucket sort by frequency
    buckets = [[] for _ in range(len(nums) + 1)]
    for num, freq in count.items():
        buckets[freq].append(num)

    result = []
    for i in range(len(buckets) - 1, -1, -1):
        for num in buckets[i]:
            result.append(num)
            if len(result) == k:
                return result

    return result

def first_unique_char(s):
    """Find first non-repeating character"""
    count = Counter(s)
    for i, char in enumerate(s):
        if count[char] == 1:
            return i
    return -1
```

**JavaScript:**
```javascript
function topKFrequent(nums, k) {
  // Find k most frequent elements - O(n)
  const count = new Map();
  for (const num of nums) {
    count.set(num, (count.get(num) ?? 0) + 1);
  }

  // Bucket sort by frequency
  const buckets = Array.from({ length: nums.length + 1 }, () => []);
  for (const [num, freq] of count) {
    buckets[freq].push(num);
  }

  const result = [];
  for (let i = buckets.length - 1; i >= 0; i--) {
    for (const num of buckets[i]) {
      result.push(num);
      if (result.length === k) return result;
    }
  }

  return result;
}

function firstUniqueChar(s) {
  // Find first non-repeating character
  const count = new Map();
  for (const char of s) {
    count.set(char, (count.get(char) ?? 0) + 1);
  }
  for (let i = 0; i < s.length; i++) {
    if (count.get(s[i]) === 1) return i;
  }
  return -1;
}
```

### 3. Grouping / Anagrams

**Python:**
```python
def group_anagrams(strs):
    """Group strings that are anagrams of each other"""
    groups = defaultdict(list)

    for s in strs:
        # Method 1: Sort as key
        key = tuple(sorted(s))

        # Method 2: Character count as key (faster for long strings)
        # count = [0] * 26
        # for c in s:
        #     count[ord(c) - ord('a')] += 1
        # key = tuple(count)

        groups[key].append(s)

    return list(groups.values())
```

**JavaScript:**
```javascript
function groupAnagrams(strs) {
  // Group strings that are anagrams of each other
  const groups = new Map();

  for (const s of strs) {
    // Method 1: Sort as key
    const key = [...s].sort().join('');

    // Method 2: Character count as key (faster for long strings)
    // const count = new Array(26).fill(0);
    // for (const c of s) {
    //   count[c.charCodeAt(0) - 97]++;
    // }
    // const key = count.join('#');

    if (!groups.has(key)) {
      groups.set(key, []);
    }
    groups.get(key).push(s);
  }

  return [...groups.values()];
}
```

### 4. Subarray Sum

**Python:**
```python
def subarray_sum_equals_k(nums, k):
    """Count subarrays with sum equal to k"""
    count = 0
    prefix_sum = 0
    seen = {0: 1}  # Empty subarray has sum 0

    for num in nums:
        prefix_sum += num
        if prefix_sum - k in seen:
            count += seen[prefix_sum - k]
        seen[prefix_sum] = seen.get(prefix_sum, 0) + 1

    return count

def longest_subarray_sum_k(nums, k):
    """Find longest subarray with sum k"""
    prefix_sum = 0
    first_occurrence = {0: -1}
    max_len = 0

    for i, num in enumerate(nums):
        prefix_sum += num
        if prefix_sum - k in first_occurrence:
            max_len = max(max_len, i - first_occurrence[prefix_sum - k])
        if prefix_sum not in first_occurrence:
            first_occurrence[prefix_sum] = i

    return max_len
```

**JavaScript:**
```javascript
function subarraySumEqualsK(nums, k) {
  // Count subarrays with sum equal to k
  let count = 0;
  let prefixSum = 0;
  const seen = new Map([[0, 1]]); // Empty subarray has sum 0

  for (const num of nums) {
    prefixSum += num;
    if (seen.has(prefixSum - k)) {
      count += seen.get(prefixSum - k);
    }
    seen.set(prefixSum, (seen.get(prefixSum) ?? 0) + 1);
  }

  return count;
}

function longestSubarraySumK(nums, k) {
  // Find longest subarray with sum k
  let prefixSum = 0;
  const firstOccurrence = new Map([[0, -1]]);
  let maxLen = 0;

  for (let i = 0; i < nums.length; i++) {
    prefixSum += nums[i];
    if (firstOccurrence.has(prefixSum - k)) {
      maxLen = Math.max(maxLen, i - firstOccurrence.get(prefixSum - k));
    }
    if (!firstOccurrence.has(prefixSum)) {
      firstOccurrence.set(prefixSum, i);
    }
  }

  return maxLen;
}
```

### 5. Duplicate Detection

**Python:**
```python
def contains_duplicate(nums):
    """Check if array contains duplicates"""
    return len(nums) != len(set(nums))

def contains_nearby_duplicate(nums, k):
    """Check if duplicate exists within k indices"""
    seen = {}
    for i, num in enumerate(nums):
        if num in seen and i - seen[num] <= k:
            return True
        seen[num] = i
    return False

def contains_nearby_almost_duplicate(nums, k, t):
    """Check if |nums[i] - nums[j]| <= t and |i - j| <= k"""
    if t < 0:
        return False

    buckets = {}
    bucket_size = t + 1

    for i, num in enumerate(nums):
        bucket_id = num // bucket_size

        # Check same bucket
        if bucket_id in buckets:
            return True

        # Check adjacent buckets
        if bucket_id - 1 in buckets and abs(num - buckets[bucket_id - 1]) <= t:
            return True
        if bucket_id + 1 in buckets and abs(num - buckets[bucket_id + 1]) <= t:
            return True

        buckets[bucket_id] = num

        # Remove old entries
        if i >= k:
            old_bucket = nums[i - k] // bucket_size
            del buckets[old_bucket]

    return False
```

**JavaScript:**
```javascript
function containsDuplicate(nums) {
  // Check if array contains duplicates
  return nums.length !== new Set(nums).size;
}

function containsNearbyDuplicate(nums, k) {
  // Check if duplicate exists within k indices
  const seen = new Map();
  for (let i = 0; i < nums.length; i++) {
    if (seen.has(nums[i]) && i - seen.get(nums[i]) <= k) {
      return true;
    }
    seen.set(nums[i], i);
  }
  return false;
}

function containsNearbyAlmostDuplicate(nums, k, t) {
  // Check if |nums[i] - nums[j]| <= t and |i - j| <= k
  if (t < 0) return false;

  const buckets = new Map();
  const bucketSize = t + 1;

  const getBucketId = (num) => Math.floor(num / bucketSize);

  for (let i = 0; i < nums.length; i++) {
    const bucketId = getBucketId(nums[i]);

    // Check same bucket
    if (buckets.has(bucketId)) return true;

    // Check adjacent buckets
    if (buckets.has(bucketId - 1) && Math.abs(nums[i] - buckets.get(bucketId - 1)) <= t) {
      return true;
    }
    if (buckets.has(bucketId + 1) && Math.abs(nums[i] - buckets.get(bucketId + 1)) <= t) {
      return true;
    }

    buckets.set(bucketId, nums[i]);

    // Remove old entries
    if (i >= k) {
      const oldBucket = getBucketId(nums[i - k]);
      buckets.delete(oldBucket);
    }
  }

  return false;
}
```

### 6. LRU Cache

**Python:**
```python
from collections import OrderedDict

class LRUCache:
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
```

**JavaScript:**
```javascript
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map(); // Map maintains insertion order
  }

  get(key) {
    if (!this.cache.has(key)) return -1;
    // Move to end (most recently used)
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  put(key, value) {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    }
    this.cache.set(key, value);
    if (this.cache.size > this.capacity) {
      // Delete oldest (first) item
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
  }
}
```

### 7. Set Operations

**Python:**
```python
def intersection(nums1, nums2):
    """Find common elements"""
    return list(set(nums1) & set(nums2))

def union(nums1, nums2):
    """Find all unique elements"""
    return list(set(nums1) | set(nums2))

def difference(nums1, nums2):
    """Elements in nums1 but not nums2"""
    return list(set(nums1) - set(nums2))

def is_subset(nums1, nums2):
    """Check if nums1 is subset of nums2"""
    return set(nums1).issubset(set(nums2))
```

**JavaScript:**
```javascript
function intersection(nums1, nums2) {
  // Find common elements
  const set2 = new Set(nums2);
  return [...new Set(nums1)].filter(x => set2.has(x));
}

function union(nums1, nums2) {
  // Find all unique elements
  return [...new Set([...nums1, ...nums2])];
}

function difference(nums1, nums2) {
  // Elements in nums1 but not nums2
  const set2 = new Set(nums2);
  return [...new Set(nums1)].filter(x => !set2.has(x));
}

function isSubset(nums1, nums2) {
  // Check if nums1 is subset of nums2
  const set2 = new Set(nums2);
  return nums1.every(x => set2.has(x));
}
```

### 8. String/Array Matching

**Python:**
```python
def is_isomorphic(s, t):
    """Check if characters can be mapped one-to-one"""
    if len(s) != len(t):
        return False

    s_to_t = {}
    t_to_s = {}

    for c1, c2 in zip(s, t):
        if c1 in s_to_t and s_to_t[c1] != c2:
            return False
        if c2 in t_to_s and t_to_s[c2] != c1:
            return False
        s_to_t[c1] = c2
        t_to_s[c2] = c1

    return True

def word_pattern(pattern, s):
    """Check if words follow the pattern"""
    words = s.split()
    if len(pattern) != len(words):
        return False

    p_to_w = {}
    w_to_p = {}

    for p, w in zip(pattern, words):
        if p in p_to_w and p_to_w[p] != w:
            return False
        if w in w_to_p and w_to_p[w] != p:
            return False
        p_to_w[p] = w
        w_to_p[w] = p

    return True
```

**JavaScript:**
```javascript
function isIsomorphic(s, t) {
  // Check if characters can be mapped one-to-one
  if (s.length !== t.length) return false;

  const sToT = new Map();
  const tToS = new Map();

  for (let i = 0; i < s.length; i++) {
    const c1 = s[i], c2 = t[i];
    if (sToT.has(c1) && sToT.get(c1) !== c2) return false;
    if (tToS.has(c2) && tToS.get(c2) !== c1) return false;
    sToT.set(c1, c2);
    tToS.set(c2, c1);
  }

  return true;
}

function wordPattern(pattern, s) {
  // Check if words follow the pattern
  const words = s.split(' ');
  if (pattern.length !== words.length) return false;

  const pToW = new Map();
  const wToP = new Map();

  for (let i = 0; i < pattern.length; i++) {
    const p = pattern[i], w = words[i];
    if (pToW.has(p) && pToW.get(p) !== w) return false;
    if (wToP.has(w) && wToP.get(w) !== p) return false;
    pToW.set(p, w);
    wToP.set(w, p);
  }

  return true;
}
```

---

## Hashing Techniques

### Rolling Hash (Rabin-Karp)

**Python:**
```python
def rabin_karp(text, pattern):
    """Pattern matching using rolling hash"""
    if len(pattern) > len(text):
        return []

    base = 26
    mod = 10**9 + 7
    n, m = len(text), len(pattern)

    # Compute hash of pattern and first window
    pattern_hash = 0
    window_hash = 0
    power = 1

    for i in range(m):
        pattern_hash = (pattern_hash * base + ord(pattern[i])) % mod
        window_hash = (window_hash * base + ord(text[i])) % mod
        if i < m - 1:
            power = (power * base) % mod

    results = []
    for i in range(n - m + 1):
        if window_hash == pattern_hash and text[i:i+m] == pattern:
            results.append(i)

        # Slide window
        if i < n - m:
            window_hash = ((window_hash - ord(text[i]) * power) * base + ord(text[i + m])) % mod
            window_hash = (window_hash + mod) % mod  # Handle negative

    return results
```

**JavaScript:**
```javascript
function rabinKarp(text, pattern) {
  // Pattern matching using rolling hash
  if (pattern.length > text.length) return [];

  const base = 26;
  const mod = 1e9 + 7;
  const n = text.length, m = pattern.length;

  // Compute hash of pattern and first window
  let patternHash = 0;
  let windowHash = 0;
  let power = 1;

  for (let i = 0; i < m; i++) {
    patternHash = (patternHash * base + pattern.charCodeAt(i)) % mod;
    windowHash = (windowHash * base + text.charCodeAt(i)) % mod;
    if (i < m - 1) {
      power = (power * base) % mod;
    }
  }

  const results = [];
  for (let i = 0; i <= n - m; i++) {
    if (windowHash === patternHash && text.slice(i, i + m) === pattern) {
      results.push(i);
    }

    // Slide window
    if (i < n - m) {
      windowHash = ((windowHash - text.charCodeAt(i) * power) * base + text.charCodeAt(i + m)) % mod;
      windowHash = ((windowHash % mod) + mod) % mod; // Handle negative
    }
  }

  return results;
}
```

### Consistent Hashing

```python
import bisect
import hashlib

class ConsistentHash:
    """For distributed systems - Lead level knowledge"""
    def __init__(self, nodes=None, replicas=100):
        self.replicas = replicas
        self.ring = []
        self.ring_map = {}

        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node):
        for i in range(self.replicas):
            key = self._hash(f"{node}:{i}")
            self.ring.append(key)
            self.ring_map[key] = node
        self.ring.sort()

    def remove_node(self, node):
        for i in range(self.replicas):
            key = self._hash(f"{node}:{i}")
            self.ring.remove(key)
            del self.ring_map[key]

    def get_node(self, key):
        if not self.ring:
            return None
        h = self._hash(key)
        idx = bisect.bisect_right(self.ring, h)
        if idx == len(self.ring):
            idx = 0
        return self.ring_map[self.ring[idx]]
```

---

## Must-Know Problems

### Easy
1. Two Sum
2. Valid Anagram
3. Contains Duplicate
4. Single Number
5. Intersection of Two Arrays

### Medium
1. Group Anagrams
2. Top K Frequent Elements
3. Subarray Sum Equals K
4. Longest Consecutive Sequence
5. LRU Cache
6. 4Sum II
7. Find All Anagrams in String
8. Encode and Decode TinyURL

### Hard
1. LFU Cache
2. Minimum Window Substring
3. Substring with Concatenation of All Words
4. First Missing Positive (uses array as hash)

---

## Interview Discussion Points

### When to Use Hash Table vs Other DS

| Use Case | Hash Table | Alternative |
|----------|------------|-------------|
| Lookup by key | Yes | - |
| Sorted data needed | No | TreeMap |
| Range queries | No | TreeMap/BST |
| Memory constrained | Maybe | Trie (for strings) |
| Duplicate handling | Counter | Multiset |
| Order matters | OrderedDict | LinkedList + Map |

### Common Pitfalls

1. **Mutable keys**: Never use lists/sets as dict keys
2. **Hash collisions**: Can degrade to O(n) in worst case
3. **Memory overhead**: ~30% overhead vs arrays
4. **Not thread-safe**: Use `threading.Lock` or `concurrent.futures`

---

## Practice Checklist

- [ ] Can implement hash table from scratch
- [ ] Understand chaining vs open addressing trade-offs
- [ ] Know when Counter/defaultdict is appropriate
- [ ] Can recognize prefix sum + hash map pattern
- [ ] Familiar with rolling hash concept
