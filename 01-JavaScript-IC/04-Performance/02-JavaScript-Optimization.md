# JavaScript Optimization

## Overview

Optimizing JavaScript execution is crucial for responsive web applications. This covers algorithmic efficiency, memory optimization, and techniques to prevent blocking the main thread.

---

## Algorithm Optimization

### Time Complexity Awareness

```javascript
// O(n²) - Avoid for large datasets
function findDuplicatesNaive(arr) {
  const duplicates = [];
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j] && !duplicates.includes(arr[i])) {
        duplicates.push(arr[i]);
      }
    }
  }
  return duplicates;
}

// O(n) - Using Set
function findDuplicatesOptimized(arr) {
  const seen = new Set();
  const duplicates = new Set();

  for (const item of arr) {
    if (seen.has(item)) {
      duplicates.add(item);
    }
    seen.add(item);
  }

  return [...duplicates];
}

// Use Map/Set for O(1) lookups
const map = new Map();
const set = new Set();

// O(1) operations
map.set(key, value);
map.get(key);
map.has(key);
set.add(value);
set.has(value);
```

### Efficient Data Structures

```javascript
// Array operations complexity
const arr = [1, 2, 3, 4, 5];

arr[0];              // O(1) - index access
arr.push(6);         // O(1) - add to end
arr.pop();           // O(1) - remove from end
arr.unshift(0);      // O(n) - add to start (shifts all)
arr.shift();         // O(n) - remove from start (shifts all)
arr.includes(3);     // O(n) - linear search
arr.indexOf(3);      // O(n) - linear search
arr.splice(2, 1);    // O(n) - may shift elements

// Object vs Map performance
// Object - good for small, string-keyed data
const obj = { name: 'John' };

// Map - better for frequent add/delete, any key type
const map = new Map();
map.set(objKey, 'value');  // Objects as keys
map.set(1, 'number');      // Number keys
map.delete(key);           // Frequent deletions are faster

// Set for unique values
const set = new Set(array);  // O(n) to create
set.has(value);              // O(1) lookup
```

### Loop Optimization

```javascript
// Cache array length
const arr = getLargeArray();

// Bad - length evaluated each iteration
for (let i = 0; i < arr.length; i++) {}

// Good - length cached
for (let i = 0, len = arr.length; i < len; i++) {}

// Better - use for...of for readability (similar performance)
for (const item of arr) {}

// Fastest for performance-critical code - while loop
let i = arr.length;
while (i--) {
  // Process arr[i]
}

// Array methods are often clearer and fast enough
arr.forEach(item => process(item));
const mapped = arr.map(item => transform(item));
const filtered = arr.filter(item => condition(item));

// Reduce for single-pass operations
const sum = arr.reduce((acc, val) => acc + val, 0);
```

---

## Debouncing and Throttling

### Debounce

```javascript
// Execute after pause in calls
function debounce(fn, delay) {
  let timeoutId;

  return function(...args) {
    clearTimeout(timeoutId);

    timeoutId = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}

// With leading edge option
function debounce(fn, delay, { leading = false } = {}) {
  let timeoutId;
  let leadingCall = true;

  return function(...args) {
    clearTimeout(timeoutId);

    if (leading && leadingCall) {
      fn.apply(this, args);
      leadingCall = false;
    }

    timeoutId = setTimeout(() => {
      if (!leading) {
        fn.apply(this, args);
      }
      leadingCall = true;
    }, delay);
  };
}

// Usage: Search input
const search = debounce(async (query) => {
  const results = await fetchResults(query);
  displayResults(results);
}, 300);

searchInput.addEventListener('input', (e) => {
  search(e.target.value);
});
```

### Throttle

```javascript
// Execute at most once per interval
function throttle(fn, limit) {
  let inThrottle = false;

  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

// With trailing call option
function throttle(fn, limit, { trailing = true } = {}) {
  let lastArgs = null;
  let inThrottle = false;

  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;

      setTimeout(() => {
        inThrottle = false;
        if (trailing && lastArgs) {
          fn.apply(this, lastArgs);
          lastArgs = null;
        }
      }, limit);
    } else if (trailing) {
      lastArgs = args;
    }
  };
}

// Usage: Scroll handler
const handleScroll = throttle(() => {
  updateScrollPosition();
  checkLazyImages();
}, 100);

window.addEventListener('scroll', handleScroll);

// Usage: Resize handler
const handleResize = throttle(() => {
  recalculateLayout();
}, 200);

window.addEventListener('resize', handleResize);
```

### requestAnimationFrame Throttle

```javascript
// Throttle to animation frames (usually 60fps)
function rafThrottle(fn) {
  let rafId = null;

  return function(...args) {
    if (rafId) return;

    rafId = requestAnimationFrame(() => {
      fn.apply(this, args);
      rafId = null;
    });
  };
}

// Perfect for visual updates
const updatePosition = rafThrottle((x, y) => {
  element.style.transform = `translate(${x}px, ${y}px)`;
});

document.addEventListener('mousemove', (e) => {
  updatePosition(e.clientX, e.clientY);
});
```

---

## Memoization

### Basic Memoization

```javascript
// Cache function results
function memoize(fn) {
  const cache = new Map();

  return function(...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// Usage
const expensiveCalculation = memoize((n) => {
  console.log('Computing...');
  return n * n;
});

expensiveCalculation(5);  // Computing... 25
expensiveCalculation(5);  // 25 (cached)
```

### Memoization with Max Size (LRU)

```javascript
function memoizeLRU(fn, maxSize = 100) {
  const cache = new Map();

  return function(...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      // Move to end (most recently used)
      const value = cache.get(key);
      cache.delete(key);
      cache.set(key, value);
      return value;
    }

    const result = fn.apply(this, args);

    // Evict oldest if at capacity
    if (cache.size >= maxSize) {
      const oldestKey = cache.keys().next().value;
      cache.delete(oldestKey);
    }

    cache.set(key, result);
    return result;
  };
}
```

### Memoization with WeakMap (for object arguments)

```javascript
function memoizeWeak(fn) {
  const cache = new WeakMap();

  return function(obj, ...rest) {
    if (!cache.has(obj)) {
      cache.set(obj, new Map());
    }

    const objCache = cache.get(obj);
    const key = JSON.stringify(rest);

    if (objCache.has(key)) {
      return objCache.get(key);
    }

    const result = fn.call(this, obj, ...rest);
    objCache.set(key, result);
    return result;
  };
}

// Object can be garbage collected
// and cache entry goes with it
```

### React useMemo and useCallback

```javascript
// useMemo - memoize computed values
function ExpensiveComponent({ items, filter }) {
  const filteredItems = useMemo(() => {
    return items.filter(item => item.category === filter);
  }, [items, filter]);  // Only recompute when these change

  return <List items={filteredItems} />;
}

// useCallback - memoize functions
function ParentComponent({ id }) {
  const handleClick = useCallback(() => {
    fetchData(id);
  }, [id]);  // New function only when id changes

  return <ChildComponent onClick={handleClick} />;
}

// React.memo - memoize component
const MemoizedComponent = React.memo(function Component({ value }) {
  return <div>{value}</div>;
});

// With custom comparison
const MemoizedComponent = React.memo(Component, (prevProps, nextProps) => {
  return prevProps.id === nextProps.id;
});
```

---

## Web Workers

### Basic Web Worker

```javascript
// main.js
const worker = new Worker('worker.js');

// Send data to worker
worker.postMessage({ type: 'calculate', data: largeArray });

// Receive results
worker.onmessage = (event) => {
  console.log('Result:', event.data);
};

worker.onerror = (error) => {
  console.error('Worker error:', error);
};

// Terminate worker
worker.terminate();

// worker.js
self.onmessage = (event) => {
  const { type, data } = event.data;

  if (type === 'calculate') {
    const result = heavyCalculation(data);
    self.postMessage(result);
  }
};

function heavyCalculation(arr) {
  // CPU-intensive work here
  return arr.reduce((sum, n) => sum + n, 0);
}
```

### Inline Worker (No Separate File)

```javascript
function createWorker(fn) {
  const blob = new Blob(
    [`self.onmessage = function(e) { self.postMessage((${fn})(e.data)); }`],
    { type: 'application/javascript' }
  );
  return new Worker(URL.createObjectURL(blob));
}

// Usage
const sortWorker = createWorker((arr) => {
  return arr.sort((a, b) => a - b);
});

sortWorker.postMessage([3, 1, 4, 1, 5, 9, 2, 6]);
sortWorker.onmessage = (e) => console.log('Sorted:', e.data);
```

### Transferable Objects

```javascript
// Transfer ownership instead of copying (much faster for large data)
const buffer = new ArrayBuffer(1024 * 1024);  // 1MB

// Slow - copies data
worker.postMessage({ buffer });

// Fast - transfers ownership (buffer becomes unusable in main thread)
worker.postMessage({ buffer }, [buffer]);
console.log(buffer.byteLength);  // 0 (transferred)

// With typed arrays
const float32Array = new Float32Array(1000000);
worker.postMessage({ data: float32Array }, [float32Array.buffer]);
```

### Worker Pool

```javascript
class WorkerPool {
  constructor(workerScript, poolSize = navigator.hardwareConcurrency) {
    this.workers = [];
    this.queue = [];
    this.activeWorkers = new Map();

    for (let i = 0; i < poolSize; i++) {
      const worker = new Worker(workerScript);
      worker.onmessage = (e) => this.handleMessage(worker, e);
      this.workers.push(worker);
    }
  }

  handleMessage(worker, event) {
    const { resolve } = this.activeWorkers.get(worker);
    resolve(event.data);
    this.activeWorkers.delete(worker);
    this.workers.push(worker);
    this.processQueue();
  }

  processQueue() {
    if (this.queue.length === 0 || this.workers.length === 0) return;

    const worker = this.workers.pop();
    const { data, resolve, reject } = this.queue.shift();

    this.activeWorkers.set(worker, { resolve, reject });
    worker.postMessage(data);
  }

  exec(data) {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject });
      this.processQueue();
    });
  }

  terminate() {
    this.workers.forEach(w => w.terminate());
    this.workers = [];
  }
}

// Usage
const pool = new WorkerPool('worker.js', 4);

const results = await Promise.all([
  pool.exec({ task: 'process', data: chunk1 }),
  pool.exec({ task: 'process', data: chunk2 }),
  pool.exec({ task: 'process', data: chunk3 }),
  pool.exec({ task: 'process', data: chunk4 })
]);
```

---

## Avoiding Main Thread Blocking

### requestIdleCallback

```javascript
// Schedule work during idle periods
function processItems(items) {
  let index = 0;

  function processChunk(deadline) {
    // Process while we have idle time
    while (index < items.length && deadline.timeRemaining() > 0) {
      processItem(items[index]);
      index++;
    }

    // More work? Schedule next chunk
    if (index < items.length) {
      requestIdleCallback(processChunk);
    }
  }

  requestIdleCallback(processChunk);
}

// With timeout (fallback)
requestIdleCallback(callback, { timeout: 2000 });
```

### Chunking Long Tasks

```javascript
// Break long task into chunks
async function processLargeArray(items, chunkSize = 1000) {
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);

    // Process chunk
    chunk.forEach(item => processItem(item));

    // Yield to main thread
    await new Promise(resolve => setTimeout(resolve, 0));
  }
}

// Using generator
function* chunkGenerator(items, chunkSize = 1000) {
  for (let i = 0; i < items.length; i += chunkSize) {
    yield items.slice(i, i + chunkSize);
  }
}

async function processWithChunks(items) {
  for (const chunk of chunkGenerator(items, 1000)) {
    chunk.forEach(processItem);
    await scheduler.yield();  // New API (if available)
  }
}
```

### Scheduler API (Experimental)

```javascript
// Priority-based scheduling
if ('scheduler' in window) {
  // User-blocking - highest priority
  scheduler.postTask(() => handleUserInput(), { priority: 'user-blocking' });

  // User-visible - medium priority
  scheduler.postTask(() => updateUI(), { priority: 'user-visible' });

  // Background - lowest priority
  scheduler.postTask(() => analytics(), { priority: 'background' });

  // With abort
  const controller = new TaskController();
  scheduler.postTask(task, {
    priority: 'background',
    signal: controller.signal
  });
  controller.abort();
}
```

---

## Memory Efficient Patterns

### Object Pooling

```javascript
class ObjectPool {
  constructor(factory, reset, initialSize = 10) {
    this.factory = factory;
    this.reset = reset;
    this.pool = Array.from({ length: initialSize }, factory);
  }

  acquire() {
    return this.pool.length > 0 ? this.pool.pop() : this.factory();
  }

  release(obj) {
    this.reset(obj);
    this.pool.push(obj);
  }
}

// Example: Particle system
const particlePool = new ObjectPool(
  () => ({ x: 0, y: 0, vx: 0, vy: 0, life: 0 }),
  (p) => { p.x = 0; p.y = 0; p.vx = 0; p.vy = 0; p.life = 0; }
);

function createParticle(x, y) {
  const p = particlePool.acquire();
  p.x = x;
  p.y = y;
  p.vx = Math.random() - 0.5;
  p.vy = Math.random() - 0.5;
  p.life = 100;
  return p;
}

function removeParticle(p) {
  particlePool.release(p);
}
```

### Avoid Creating Objects in Loops

```javascript
// Bad - creates new object each iteration
function processCoordinates(points) {
  points.forEach(point => {
    const normalized = { x: point.x / 100, y: point.y / 100 };
    draw(normalized);
  });
}

// Good - reuse object
function processCoordinates(points) {
  const normalized = { x: 0, y: 0 };

  points.forEach(point => {
    normalized.x = point.x / 100;
    normalized.y = point.y / 100;
    draw(normalized);
  });
}

// Bad - creates new array each iteration
for (let i = 0; i < 1000; i++) {
  doSomething([a, b, c]);
}

// Good - reuse array
const args = [null, null, null];
for (let i = 0; i < 1000; i++) {
  args[0] = a; args[1] = b; args[2] = c;
  doSomething(args);
}
```

---

## Interview Questions

### Q1: Implement debounce and throttle

```javascript
// Debounce - delay until pause
function debounce(fn, delay) {
  let timeoutId;
  return function(...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Throttle - limit frequency
function throttle(fn, limit) {
  let inThrottle;
  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}
```

### Q2: How would you optimize a slow rendering list?

```javascript
// 1. Virtual scrolling - only render visible items
// 2. Use keys for React reconciliation
// 3. Memoize items with React.memo
// 4. Debounce filter/search inputs
// 5. Use windowing libraries (react-window, react-virtualized)

// Virtual list concept
function VirtualList({ items, rowHeight, visibleRows }) {
  const [scrollTop, setScrollTop] = useState(0);

  const startIndex = Math.floor(scrollTop / rowHeight);
  const endIndex = Math.min(startIndex + visibleRows, items.length);
  const visibleItems = items.slice(startIndex, endIndex);

  return (
    <div onScroll={e => setScrollTop(e.target.scrollTop)}>
      <div style={{ height: items.length * rowHeight }}>
        <div style={{ transform: `translateY(${startIndex * rowHeight}px)` }}>
          {visibleItems.map(item => <Row key={item.id} item={item} />)}
        </div>
      </div>
    </div>
  );
}
```

### Q3: When would you use Web Workers?

```javascript
// Use Web Workers for:
// - Heavy computations (data processing, image manipulation)
// - Parsing large JSON
// - Complex algorithms (sorting large arrays, pathfinding)
// - Encryption/decryption

// Don't use for:
// - DOM manipulation (not accessible)
// - Simple operations (overhead not worth it)
// - Frequent small tasks (use requestIdleCallback instead)
```

### Q4: How do you prevent long tasks from blocking the UI?

```javascript
// 1. Break into chunks with setTimeout
// 2. Use requestIdleCallback for non-critical work
// 3. Use Web Workers for heavy computation
// 4. Use requestAnimationFrame for visual updates
// 5. Virtualize long lists

async function processWithoutBlocking(items) {
  const chunkSize = 100;
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    processChunk(chunk);
    await new Promise(r => setTimeout(r, 0));  // Yield
  }
}
```

---

## Key Takeaways

1. **Algorithm complexity matters**: Use Map/Set for O(1) lookups
2. **Debounce** for input events, **throttle** for continuous events
3. **Memoization** prevents redundant calculations
4. **Web Workers** offload CPU-intensive work
5. **Chunk long tasks** to keep UI responsive
6. **requestIdleCallback** for non-critical background work
7. **Object pooling** reduces garbage collection pressure
8. **Virtual scrolling** for long lists
