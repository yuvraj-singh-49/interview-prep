# Memory Management in JavaScript

## Overview

Understanding memory management is crucial for building performant applications and avoiding memory leaks. JavaScript uses automatic garbage collection, but developers must still be aware of memory patterns.

---

## Memory Lifecycle

```
1. Allocate memory    →  2. Use memory    →  3. Release memory
   (automatic)            (read/write)        (garbage collection)
```

### Allocation

```javascript
// Primitives - stored on stack
const number = 42;
const string = 'hello';
const boolean = true;

// Objects - stored on heap (reference on stack)
const obj = { name: 'John' };
const arr = [1, 2, 3];
const func = function() {};

// Function calls allocate memory
function createUser(name) {
  return { name };  // New object allocated
}
```

---

## Garbage Collection

JavaScript uses **mark-and-sweep** algorithm:

1. **Mark**: Starting from roots (global object, local variables), mark all reachable objects
2. **Sweep**: Free memory for unmarked objects

### Roots

```javascript
// Global variables
window.globalVar = { data: 'global' };

// Local variables in executing functions
function foo() {
  const local = { data: 'local' };  // Root while foo executes
}

// DOM references
const element = document.getElementById('app');
```

### Reference Counting (Historical)

```javascript
// Circular reference problem (old IE)
function createCycle() {
  const obj1 = {};
  const obj2 = {};
  obj1.ref = obj2;
  obj2.ref = obj1;
  // Reference count never reaches 0
}
// Modern garbage collectors handle this
```

---

## Memory Leaks

Memory that is no longer needed but not released.

### 1. Accidental Global Variables

```javascript
// Bad - creates global variable
function leak() {
  leakedVar = 'I am global';  // Missing 'const/let'
}

// Bad - 'this' is global in non-strict mode
function leak() {
  this.leakedProp = 'I am on window';
}

// Fix - use strict mode
'use strict';
function noLeak() {
  const localVar = 'I am local';
}
```

### 2. Forgotten Timers and Callbacks

```javascript
// Leak - interval never cleared
const data = loadLargeData();
setInterval(() => {
  process(data);  // data is retained forever
}, 1000);

// Fix - clear when done
const intervalId = setInterval(() => {
  process(data);
}, 1000);
// Later...
clearInterval(intervalId);

// Leak - event listener never removed
const button = document.getElementById('btn');
const handler = () => console.log('clicked');
button.addEventListener('click', handler);
// Button removed from DOM but handler keeps reference

// Fix - remove listener
button.removeEventListener('click', handler);
```

### 3. Closures Holding References

```javascript
// Leak - closure retains large data
function createHandler() {
  const largeData = new Array(1000000).fill('x');

  return function() {
    console.log('Handler');
    // largeData is retained even if unused
  };
}

const handler = createHandler();
// largeData stays in memory

// Fix - only capture what's needed
function createHandler() {
  const largeData = new Array(1000000).fill('x');
  const neededValue = largeData.length;

  return function() {
    console.log('Length:', neededValue);
  };
}
```

### 4. Detached DOM Elements

```javascript
// Leak - reference to removed element
const elements = [];

function addElement() {
  const div = document.createElement('div');
  document.body.appendChild(div);
  elements.push(div);  // Keep reference
}

function removeElements() {
  elements.forEach(el => el.remove());
  // elements array still holds references!
}

// Fix - clear references
function removeElements() {
  elements.forEach(el => el.remove());
  elements.length = 0;  // Clear array
}
```

### 5. Caches Without Bounds

```javascript
// Leak - unbounded cache
const cache = {};

function cacheData(key, data) {
  cache[key] = data;  // Never removed
}

// Fix 1 - LRU cache with max size
class LRUCache {
  constructor(maxSize) {
    this.maxSize = maxSize;
    this.cache = new Map();
  }

  get(key) {
    if (!this.cache.has(key)) return undefined;
    const value = this.cache.get(key);
    // Move to end (most recently used)
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  set(key, value) {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      // Remove oldest (first item)
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }
}

// Fix 2 - WeakMap (auto garbage collected)
const cache = new WeakMap();

function cacheData(obj, data) {
  cache.set(obj, data);  // Removed when obj is garbage collected
}
```

---

## WeakMap and WeakSet

Keys are weakly held - if no other reference exists, garbage collection can occur.

### WeakMap

```javascript
const weakMap = new WeakMap();

let obj = { name: 'John' };
weakMap.set(obj, 'metadata');

console.log(weakMap.get(obj));  // 'metadata'

obj = null;  // Object can now be garbage collected
// weakMap entry is also removed

// Use cases:
// 1. Private data
const privateData = new WeakMap();

class User {
  constructor(name, password) {
    this.name = name;
    privateData.set(this, { password });
  }

  checkPassword(pwd) {
    return privateData.get(this).password === pwd;
  }
}

// 2. Caching computed values
const cache = new WeakMap();

function computeExpensive(obj) {
  if (cache.has(obj)) {
    return cache.get(obj);
  }
  const result = /* expensive computation */;
  cache.set(obj, result);
  return result;
}

// 3. Tracking DOM elements
const elementData = new WeakMap();

function storeData(element, data) {
  elementData.set(element, data);
  // Automatically cleaned up when element is removed
}
```

### WeakSet

```javascript
const weakSet = new WeakSet();

let obj = { id: 1 };
weakSet.add(obj);

console.log(weakSet.has(obj));  // true

obj = null;  // Object can be garbage collected

// Use case: Tracking objects without preventing GC
const processed = new WeakSet();

function processOnce(obj) {
  if (processed.has(obj)) {
    return;  // Already processed
  }
  processed.add(obj);
  // Process object...
}
```

### Limitations

```javascript
// WeakMap/WeakSet:
// - Keys must be objects (not primitives)
// - Not iterable (no forEach, keys, values, entries)
// - No size property
// - Cannot clear all entries

// WeakRef and FinalizationRegistry (advanced)
let obj = { name: 'John' };
const weakRef = new WeakRef(obj);

// Later - might be undefined if GC ran
const maybeObj = weakRef.deref();
if (maybeObj) {
  console.log(maybeObj.name);
}
```

---

## Memory Profiling

### Chrome DevTools

```javascript
// 1. Memory tab → Take heap snapshot
// 2. Compare snapshots to find leaks
// 3. Look for "Detached" DOM trees

// Performance timeline
// - Record memory usage over time
// - Look for steadily increasing memory

// allocation timeline
// - Shows when allocations happen
// - Identify allocation patterns
```

### Programmatic Memory Info

```javascript
// Chrome/Node.js
if (performance.memory) {
  console.log('Used JS Heap:', performance.memory.usedJSHeapSize);
  console.log('Total JS Heap:', performance.memory.totalJSHeapSize);
  console.log('Heap Limit:', performance.memory.jsHeapSizeLimit);
}

// Node.js
const used = process.memoryUsage();
console.log('Heap Used:', used.heapUsed);
console.log('Heap Total:', used.heapTotal);
console.log('External:', used.external);
console.log('RSS:', used.rss);
```

---

## Memory Optimization Patterns

### 1. Object Pooling

```javascript
// Reuse objects instead of creating new ones
class ObjectPool {
  constructor(createFn, resetFn, initialSize = 10) {
    this.createFn = createFn;
    this.resetFn = resetFn;
    this.pool = Array.from({ length: initialSize }, createFn);
  }

  acquire() {
    return this.pool.pop() || this.createFn();
  }

  release(obj) {
    this.resetFn(obj);
    this.pool.push(obj);
  }
}

// Example: Particle system
const particlePool = new ObjectPool(
  () => ({ x: 0, y: 0, vx: 0, vy: 0, active: false }),
  (p) => { p.x = 0; p.y = 0; p.vx = 0; p.vy = 0; p.active = false; }
);

function spawnParticle(x, y) {
  const particle = particlePool.acquire();
  particle.x = x;
  particle.y = y;
  particle.active = true;
  return particle;
}

function despawnParticle(particle) {
  particlePool.release(particle);
}
```

### 2. Lazy Initialization

```javascript
// Don't allocate until needed
class HeavyResource {
  constructor() {
    this._data = null;
  }

  get data() {
    if (this._data === null) {
      this._data = this._loadHeavyData();
    }
    return this._data;
  }

  _loadHeavyData() {
    // Expensive operation
    return new Array(1000000).fill('data');
  }
}
```

### 3. Streaming and Chunking

```javascript
// Process large data in chunks
async function processLargeArray(items, chunkSize = 1000) {
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    await processChunk(chunk);

    // Allow GC between chunks
    await new Promise(resolve => setTimeout(resolve, 0));
  }
}

// Use generators for lazy evaluation
function* processItems(items) {
  for (const item of items) {
    yield processItem(item);
  }
}

// Process one at a time
const processor = processItems(largeArray);
for (const result of processor) {
  // Memory efficient - one at a time
}
```

### 4. String Interning

```javascript
// JavaScript automatically interns string literals
const str1 = 'hello';
const str2 = 'hello';
console.log(str1 === str2);  // true (same memory location)

// Be careful with dynamic strings
const strings = [];
for (let i = 0; i < 10000; i++) {
  strings.push('prefix_' + i);  // Each is unique
}

// Consider using symbols for known string sets
const Status = {
  PENDING: Symbol('pending'),
  SUCCESS: Symbol('success'),
  ERROR: Symbol('error')
};
```

---

## Common Interview Questions

### Q1: How does garbage collection work in JavaScript?

```javascript
// JavaScript uses mark-and-sweep:
// 1. Starting from roots (global, local variables), traverse all reachable objects
// 2. Mark all reachable objects
// 3. Free memory of unmarked objects

// Modern engines also use:
// - Generational GC (young vs old generation)
// - Incremental marking (avoid long pauses)
// - Concurrent/parallel collection
```

### Q2: What causes memory leaks?

```javascript
// 1. Accidental globals
// 2. Forgotten timers/intervals
// 3. Closures holding unnecessary references
// 4. Detached DOM nodes with references
// 5. Unbounded caches
// 6. Event listeners not removed
```

### Q3: When would you use WeakMap?

```javascript
// 1. Private data for objects
// 2. Caching computed values
// 3. Storing metadata for DOM elements
// 4. Any case where you want automatic cleanup when key object is GC'd
```

### Q4: How would you detect a memory leak?

```javascript
// 1. Take heap snapshots at different times
// 2. Compare snapshots to see growing objects
// 3. Use allocation timeline to see allocation patterns
// 4. Look for "Detached" DOM trees
// 5. Monitor memory usage over time in production
```

### Q5: Implement a memory-efficient cache

```javascript
class MemoryEfficientCache {
  constructor(maxSize = 100) {
    this.maxSize = maxSize;
    this.cache = new Map();
  }

  get(key) {
    if (!this.cache.has(key)) return undefined;

    // Move to end (LRU)
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  set(key, value) {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      // Remove oldest
      const oldest = this.cache.keys().next().value;
      this.cache.delete(oldest);
    }
    this.cache.set(key, value);
  }

  clear() {
    this.cache.clear();
  }
}
```

---

## Key Takeaways

1. JavaScript uses **automatic garbage collection** (mark-and-sweep)
2. Memory leaks still occur when references prevent GC
3. Common leaks: **globals, timers, closures, detached DOM, unbounded caches**
4. Use **WeakMap/WeakSet** for data that shouldn't prevent GC
5. **Profile with DevTools** to identify leaks
6. Use **object pooling** for frequent allocations
7. **Lazy initialization** delays allocation until needed
8. **Clear references** when objects are no longer needed
