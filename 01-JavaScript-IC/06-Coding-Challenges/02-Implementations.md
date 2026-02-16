# Common JavaScript Implementations

## Overview

These are common utility functions frequently asked in JavaScript interviews. Understanding these implementations demonstrates practical problem-solving skills.

---

## Debounce and Throttle

### Debounce

```javascript
// Basic debounce - delays execution until after wait period of inactivity
function debounce(fn, delay) {
  let timeoutId;

  return function(...args) {
    clearTimeout(timeoutId);

    timeoutId = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}

// Advanced debounce with leading/trailing options
function debounce(fn, delay, options = {}) {
  const { leading = false, trailing = true } = options;

  let timeoutId;
  let lastArgs;
  let leadingInvoked = false;

  return function(...args) {
    lastArgs = args;

    if (leading && !leadingInvoked) {
      fn.apply(this, args);
      leadingInvoked = true;
    }

    clearTimeout(timeoutId);

    timeoutId = setTimeout(() => {
      if (trailing && lastArgs) {
        fn.apply(this, lastArgs);
      }
      leadingInvoked = false;
      lastArgs = null;
    }, delay);
  };
}

// With cancel and flush
function debounce(fn, delay) {
  let timeoutId;
  let lastThis;
  let lastArgs;

  function debounced(...args) {
    lastThis = this;
    lastArgs = args;

    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => {
      fn.apply(lastThis, lastArgs);
    }, delay);
  }

  debounced.cancel = () => {
    clearTimeout(timeoutId);
    timeoutId = null;
  };

  debounced.flush = () => {
    clearTimeout(timeoutId);
    fn.apply(lastThis, lastArgs);
  };

  return debounced;
}

// Usage
const search = debounce((query) => {
  console.log('Searching:', query);
}, 300);

input.addEventListener('input', (e) => search(e.target.value));
```

### Throttle

```javascript
// Basic throttle - limits execution to once per interval
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

// Advanced throttle with leading/trailing
function throttle(fn, limit, options = {}) {
  const { leading = true, trailing = true } = options;

  let lastArgs;
  let timeoutId;
  let lastCallTime = 0;

  return function(...args) {
    const now = Date.now();
    const remaining = limit - (now - lastCallTime);

    if (remaining <= 0) {
      if (timeoutId) {
        clearTimeout(timeoutId);
        timeoutId = null;
      }

      if (leading) {
        fn.apply(this, args);
        lastCallTime = now;
      }
    } else {
      lastArgs = args;

      if (!timeoutId && trailing) {
        timeoutId = setTimeout(() => {
          fn.apply(this, lastArgs);
          lastCallTime = Date.now();
          timeoutId = null;
        }, remaining);
      }
    }
  };
}

// Usage
const handleScroll = throttle(() => {
  console.log('Scroll position:', window.scrollY);
}, 100);

window.addEventListener('scroll', handleScroll);
```

---

## Deep Clone

### JSON Method (Simple)

```javascript
// Simple but has limitations
function deepClone(obj) {
  return JSON.parse(JSON.stringify(obj));
}

// Limitations:
// - Loses functions, undefined, symbols
// - Doesn't handle circular references
// - Loses Date, RegExp, Map, Set
```

### Recursive Implementation

```javascript
function deepClone(obj, hash = new WeakMap()) {
  // Handle primitives and null
  if (obj === null || typeof obj !== 'object') {
    return obj;
  }

  // Handle circular references
  if (hash.has(obj)) {
    return hash.get(obj);
  }

  // Handle Date
  if (obj instanceof Date) {
    return new Date(obj.getTime());
  }

  // Handle RegExp
  if (obj instanceof RegExp) {
    return new RegExp(obj.source, obj.flags);
  }

  // Handle Map
  if (obj instanceof Map) {
    const clone = new Map();
    hash.set(obj, clone);
    obj.forEach((value, key) => {
      clone.set(deepClone(key, hash), deepClone(value, hash));
    });
    return clone;
  }

  // Handle Set
  if (obj instanceof Set) {
    const clone = new Set();
    hash.set(obj, clone);
    obj.forEach(value => {
      clone.add(deepClone(value, hash));
    });
    return clone;
  }

  // Handle Array
  if (Array.isArray(obj)) {
    const clone = [];
    hash.set(obj, clone);
    obj.forEach((item, index) => {
      clone[index] = deepClone(item, hash);
    });
    return clone;
  }

  // Handle Object
  const clone = Object.create(Object.getPrototypeOf(obj));
  hash.set(obj, clone);

  // Handle symbols
  const symbols = Object.getOwnPropertySymbols(obj);
  for (const sym of symbols) {
    clone[sym] = deepClone(obj[sym], hash);
  }

  // Handle regular properties
  for (const key of Object.keys(obj)) {
    clone[key] = deepClone(obj[key], hash);
  }

  return clone;
}

// Test
const original = {
  name: 'John',
  date: new Date(),
  nested: { a: 1, b: [1, 2, 3] },
  map: new Map([['key', 'value']]),
  set: new Set([1, 2, 3])
};

const cloned = deepClone(original);
console.log(cloned.date instanceof Date);  // true
console.log(cloned.nested !== original.nested);  // true
```

### Using structuredClone (Modern)

```javascript
// Built-in deep clone (Node 17+, modern browsers)
const clone = structuredClone(original);

// Handles circular references, most built-in types
// Doesn't handle functions, DOM nodes, symbols
```

---

## Deep Equal

```javascript
function deepEqual(a, b) {
  // Same reference or both primitives and equal
  if (a === b) return true;

  // Handle null
  if (a === null || b === null) return false;

  // Different types
  if (typeof a !== typeof b) return false;

  // Handle primitives
  if (typeof a !== 'object') return false;

  // Handle Date
  if (a instanceof Date && b instanceof Date) {
    return a.getTime() === b.getTime();
  }

  // Handle RegExp
  if (a instanceof RegExp && b instanceof RegExp) {
    return a.source === b.source && a.flags === b.flags;
  }

  // Handle Array
  if (Array.isArray(a) !== Array.isArray(b)) return false;

  if (Array.isArray(a)) {
    if (a.length !== b.length) return false;
    return a.every((item, index) => deepEqual(item, b[index]));
  }

  // Handle Map
  if (a instanceof Map && b instanceof Map) {
    if (a.size !== b.size) return false;
    for (const [key, value] of a) {
      if (!b.has(key) || !deepEqual(value, b.get(key))) return false;
    }
    return true;
  }

  // Handle Set
  if (a instanceof Set && b instanceof Set) {
    if (a.size !== b.size) return false;
    for (const value of a) {
      if (!b.has(value)) return false;
    }
    return true;
  }

  // Handle Object
  const keysA = Object.keys(a);
  const keysB = Object.keys(b);

  if (keysA.length !== keysB.length) return false;

  return keysA.every(key => deepEqual(a[key], b[key]));
}

// Test
console.log(deepEqual({ a: 1, b: { c: 2 } }, { a: 1, b: { c: 2 } }));  // true
console.log(deepEqual([1, [2, 3]], [1, [2, 3]]));  // true
console.log(deepEqual(new Date('2024-01-01'), new Date('2024-01-01')));  // true
```

---

## Flatten Array

```javascript
// Recursive
function flatten(arr, depth = Infinity) {
  const result = [];

  for (const item of arr) {
    if (Array.isArray(item) && depth > 0) {
      result.push(...flatten(item, depth - 1));
    } else {
      result.push(item);
    }
  }

  return result;
}

// Using reduce
function flatten(arr, depth = 1) {
  return depth > 0
    ? arr.reduce((acc, val) =>
        acc.concat(Array.isArray(val) ? flatten(val, depth - 1) : val),
        []
      )
    : arr.slice();
}

// Iterative with stack
function flatten(arr, depth = Infinity) {
  const stack = arr.map(item => [item, depth]);
  const result = [];

  while (stack.length) {
    const [item, d] = stack.pop();

    if (Array.isArray(item) && d > 0) {
      for (let i = item.length - 1; i >= 0; i--) {
        stack.push([item[i], d - 1]);
      }
    } else {
      result.unshift(item);
    }
  }

  return result;
}

// Test
console.log(flatten([1, [2, [3, [4]]]], 2));  // [1, 2, 3, [4]]
```

---

## Flatten Object

```javascript
function flattenObject(obj, prefix = '', result = {}) {
  for (const key in obj) {
    if (Object.prototype.hasOwnProperty.call(obj, key)) {
      const newKey = prefix ? `${prefix}.${key}` : key;

      if (typeof obj[key] === 'object' &&
          obj[key] !== null &&
          !Array.isArray(obj[key])) {
        flattenObject(obj[key], newKey, result);
      } else {
        result[newKey] = obj[key];
      }
    }
  }

  return result;
}

// Unflatten
function unflattenObject(obj) {
  const result = {};

  for (const key in obj) {
    const keys = key.split('.');
    let current = result;

    for (let i = 0; i < keys.length - 1; i++) {
      if (!(keys[i] in current)) {
        current[keys[i]] = {};
      }
      current = current[keys[i]];
    }

    current[keys[keys.length - 1]] = obj[key];
  }

  return result;
}

// Test
const nested = { a: { b: { c: 1 }, d: 2 }, e: 3 };
console.log(flattenObject(nested));
// { 'a.b.c': 1, 'a.d': 2, 'e': 3 }

console.log(unflattenObject({ 'a.b.c': 1, 'a.d': 2 }));
// { a: { b: { c: 1 }, d: 2 } }
```

---

## Curry

```javascript
// Basic curry
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }

    return function(...moreArgs) {
      return curried.apply(this, [...args, ...moreArgs]);
    };
  };
}

// With placeholder support
const _ = Symbol('placeholder');

function curryWithPlaceholder(fn) {
  return function curried(...args) {
    // Check if we have enough non-placeholder args
    const realArgs = args.filter(arg => arg !== _);

    if (realArgs.length >= fn.length) {
      return fn.apply(this, args.map(arg => arg === _ ? undefined : arg));
    }

    return function(...moreArgs) {
      const combined = [];
      let moreIndex = 0;

      // Replace placeholders with new args
      for (const arg of args) {
        if (arg === _ && moreIndex < moreArgs.length) {
          combined.push(moreArgs[moreIndex++]);
        } else {
          combined.push(arg);
        }
      }

      // Add remaining new args
      while (moreIndex < moreArgs.length) {
        combined.push(moreArgs[moreIndex++]);
      }

      return curried.apply(this, combined);
    };
  };
}

// Test
const add = (a, b, c) => a + b + c;
const curriedAdd = curry(add);

console.log(curriedAdd(1)(2)(3));     // 6
console.log(curriedAdd(1, 2)(3));     // 6
console.log(curriedAdd(1)(2, 3));     // 6
console.log(curriedAdd(1, 2, 3));     // 6
```

---

## Compose and Pipe

```javascript
// Compose - right to left
function compose(...fns) {
  return function(x) {
    return fns.reduceRight((acc, fn) => fn(acc), x);
  };
}

// Pipe - left to right
function pipe(...fns) {
  return function(x) {
    return fns.reduce((acc, fn) => fn(acc), x);
  };
}

// Async compose
function composeAsync(...fns) {
  return function(x) {
    return fns.reduceRight(
      (acc, fn) => acc.then(fn),
      Promise.resolve(x)
    );
  };
}

// Test
const add10 = x => x + 10;
const multiply2 = x => x * 2;
const subtract5 = x => x - 5;

const composed = compose(subtract5, multiply2, add10);
console.log(composed(5));  // (5 + 10) * 2 - 5 = 25

const piped = pipe(add10, multiply2, subtract5);
console.log(piped(5));  // (5 + 10) * 2 - 5 = 25
```

---

## Memoize

```javascript
// Basic memoization
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

// With custom key function
function memoize(fn, keyFn = JSON.stringify) {
  const cache = new Map();

  return function(...args) {
    const key = keyFn(args);

    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// LRU memoization
function memoizeLRU(fn, maxSize = 100) {
  const cache = new Map();

  return function(...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      const value = cache.get(key);
      // Move to end (most recently used)
      cache.delete(key);
      cache.set(key, value);
      return value;
    }

    const result = fn.apply(this, args);

    if (cache.size >= maxSize) {
      // Remove oldest entry
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }

    cache.set(key, result);
    return result;
  };
}

// Test
const fibonacci = memoize((n) => {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
});

console.log(fibonacci(40));  // Fast with memoization
```

---

## Event Emitter

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
  }

  on(event, listener) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(listener);

    // Return unsubscribe function
    return () => this.off(event, listener);
  }

  off(event, listener) {
    if (!this.events[event]) return;

    this.events[event] = this.events[event].filter(l => l !== listener);
  }

  emit(event, ...args) {
    if (!this.events[event]) return;

    this.events[event].forEach(listener => {
      listener.apply(this, args);
    });
  }

  once(event, listener) {
    const wrapper = (...args) => {
      listener.apply(this, args);
      this.off(event, wrapper);
    };

    this.on(event, wrapper);
  }

  removeAllListeners(event) {
    if (event) {
      delete this.events[event];
    } else {
      this.events = {};
    }
  }

  listenerCount(event) {
    return this.events[event]?.length || 0;
  }
}

// Test
const emitter = new EventEmitter();

const unsubscribe = emitter.on('message', (msg) => {
  console.log('Received:', msg);
});

emitter.emit('message', 'Hello!');  // "Received: Hello!"
unsubscribe();
emitter.emit('message', 'World');   // (nothing)
```

---

## Rate Limiter

```javascript
// Token bucket rate limiter
class RateLimiter {
  constructor(capacity, refillRate) {
    this.capacity = capacity;
    this.tokens = capacity;
    this.refillRate = refillRate;
    this.lastRefill = Date.now();
  }

  refill() {
    const now = Date.now();
    const timePassed = now - this.lastRefill;
    const tokensToAdd = (timePassed / 1000) * this.refillRate;

    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }

  tryAcquire(tokens = 1) {
    this.refill();

    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;
    }

    return false;
  }
}

// Sliding window rate limiter
class SlidingWindowRateLimiter {
  constructor(limit, windowMs) {
    this.limit = limit;
    this.windowMs = windowMs;
    this.requests = [];
  }

  tryAcquire() {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    // Remove old requests
    this.requests = this.requests.filter(time => time > windowStart);

    if (this.requests.length < this.limit) {
      this.requests.push(now);
      return true;
    }

    return false;
  }
}

// Test
const limiter = new RateLimiter(10, 1);  // 10 capacity, 1 token/sec

for (let i = 0; i < 15; i++) {
  console.log(`Request ${i}: ${limiter.tryAcquire()}`);
}
```

---

## Retry with Backoff

```javascript
async function retry(fn, options = {}) {
  const {
    retries = 3,
    delay = 1000,
    backoff = 2,
    maxDelay = 30000,
    onRetry = () => {}
  } = options;

  let lastError;
  let currentDelay = delay;

  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      if (attempt < retries) {
        onRetry(error, attempt + 1);
        await new Promise(resolve => setTimeout(resolve, currentDelay));
        currentDelay = Math.min(currentDelay * backoff, maxDelay);
      }
    }
  }

  throw lastError;
}

// With jitter
async function retryWithJitter(fn, options = {}) {
  const {
    retries = 3,
    baseDelay = 1000,
    maxDelay = 30000
  } = options;

  let lastError;

  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      if (attempt < retries) {
        // Exponential backoff with jitter
        const delay = Math.min(
          maxDelay,
          baseDelay * Math.pow(2, attempt) * (0.5 + Math.random() * 0.5)
        );
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError;
}

// Test
const unreliableFetch = async () => {
  if (Math.random() < 0.7) throw new Error('Failed');
  return 'Success';
};

retry(unreliableFetch, { retries: 5, onRetry: (e, n) => console.log(`Retry ${n}`) })
  .then(console.log)
  .catch(console.error);
```

---

## Parallel Limit

```javascript
async function parallelLimit(tasks, limit) {
  const results = [];
  const executing = [];

  for (const [index, task] of tasks.entries()) {
    const promise = Promise.resolve().then(() => task()).then(result => {
      results[index] = result;
    });

    executing.push(promise);

    if (executing.length >= limit) {
      await Promise.race(executing);
      // Remove completed promises
      for (let i = executing.length - 1; i >= 0; i--) {
        if (executing[i].settled) {
          executing.splice(i, 1);
        }
      }
    }
  }

  await Promise.all(executing);
  return results;
}

// Better implementation
async function parallelLimit(tasks, limit) {
  const results = new Array(tasks.length);
  let index = 0;

  async function worker() {
    while (index < tasks.length) {
      const currentIndex = index++;
      results[currentIndex] = await tasks[currentIndex]();
    }
  }

  await Promise.all(Array.from({ length: limit }, worker));
  return results;
}

// Test
const tasks = Array.from({ length: 10 }, (_, i) => async () => {
  await new Promise(r => setTimeout(r, 100));
  return i;
});

parallelLimit(tasks, 3).then(console.log);  // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

---

## Interview Questions

### Q1: Implement a function that limits concurrent promises

```javascript
function promisePool(tasks, limit) {
  return new Promise((resolve) => {
    const results = [];
    let index = 0;
    let running = 0;

    function runNext() {
      while (running < limit && index < tasks.length) {
        const currentIndex = index++;
        running++;

        tasks[currentIndex]()
          .then(result => { results[currentIndex] = result; })
          .finally(() => {
            running--;
            if (index < tasks.length) {
              runNext();
            } else if (running === 0) {
              resolve(results);
            }
          });
      }
    }

    runNext();
  });
}
```

### Q2: Implement a scheduler that executes tasks with dependencies

```javascript
async function taskScheduler(tasks) {
  const completed = new Set();
  const results = {};

  async function runTask(name) {
    if (completed.has(name)) return;

    const task = tasks[name];

    // Wait for dependencies
    if (task.deps) {
      await Promise.all(task.deps.map(runTask));
    }

    results[name] = await task.fn();
    completed.add(name);
  }

  await Promise.all(Object.keys(tasks).map(runTask));
  return results;
}
```

---

## Key Takeaways

1. **Debounce vs Throttle**: Debounce for final value, throttle for rate limiting
2. **Deep operations**: Handle circular references, special types
3. **Memoization**: Consider cache size limits
4. **Event emitters**: Return unsubscribe functions
5. **Async patterns**: Handle concurrency limits, retries with backoff
6. **Composition**: Right-to-left (compose) vs left-to-right (pipe)
