# Event Loop and Asynchronous JavaScript

## Overview

Understanding the event loop is crucial for JavaScript developers. It explains how JavaScript handles asynchronous operations despite being single-threaded.

---

## JavaScript Runtime Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     JavaScript Engine                        │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │   Call Stack    │    │           Heap                  │ │
│  │                 │    │    (Object allocation)          │ │
│  │  ┌───────────┐  │    │                                 │ │
│  │  │  func()   │  │    │                                 │ │
│  │  ├───────────┤  │    │                                 │ │
│  │  │  main()   │  │    │                                 │ │
│  │  └───────────┘  │    │                                 │ │
│  └─────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                        Web APIs                              │
│     (setTimeout, fetch, DOM, addEventListener, etc.)         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      Callback Queue                          │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Macro Tasks: setTimeout, setInterval, I/O, UI rendering ││
│  └─────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Micro Tasks: Promises, queueMicrotask, MutationObserver ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘

                         Event Loop
                     (checks & moves)
```

---

## The Event Loop

The event loop continuously checks:
1. Is the call stack empty?
2. If yes, process all microtasks
3. Then process one macrotask
4. Repeat

```javascript
console.log('Start');

setTimeout(() => {
  console.log('Timeout');
}, 0);

Promise.resolve().then(() => {
  console.log('Promise');
});

console.log('End');

// Output:
// Start
// End
// Promise
// Timeout
```

### Why this order?
1. `console.log('Start')` - synchronous, executes immediately
2. `setTimeout` - registers callback to macrotask queue
3. `Promise.then` - registers callback to microtask queue
4. `console.log('End')` - synchronous, executes immediately
5. Call stack empty, process all microtasks: `Promise`
6. Process one macrotask: `Timeout`

---

## Microtasks vs Macrotasks

### Microtasks (Higher Priority)
- `Promise.then()`, `Promise.catch()`, `Promise.finally()`
- `queueMicrotask()`
- `MutationObserver`
- `async/await` (uses promises internally)

### Macrotasks (Lower Priority)
- `setTimeout()`, `setInterval()`
- `setImmediate()` (Node.js)
- I/O operations
- UI rendering
- Event handlers (click, keypress, etc.)

### Execution Order

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve()
  .then(() => console.log('3'))
  .then(() => console.log('4'));

Promise.resolve().then(() => {
  console.log('5');
  setTimeout(() => console.log('6'), 0);
});

console.log('7');

// Output: 1, 7, 3, 5, 4, 2, 6
```

**Explanation:**
1. Sync: `1`, `7`
2. Microtasks (all processed): `3`, `5`, `4`
3. Macrotask: `2`
4. Macrotask: `6`

---

## Call Stack

```javascript
function multiply(a, b) {
  return a * b;
}

function square(n) {
  return multiply(n, n);
}

function printSquare(n) {
  const result = square(n);
  console.log(result);
}

printSquare(4);

/*
Call Stack progression:
1. [printSquare(4)]
2. [printSquare(4), square(4)]
3. [printSquare(4), square(4), multiply(4, 4)]
4. [printSquare(4), square(4)]  // multiply returns
5. [printSquare(4)]              // square returns
6. []                            // printSquare returns
*/
```

### Stack Overflow

```javascript
function recursive() {
  recursive();
}

recursive();  // Maximum call stack size exceeded
```

---

## setTimeout and setInterval

### setTimeout(callback, delay)

```javascript
console.log('Start');

setTimeout(() => {
  console.log('Timeout 1');
}, 1000);

setTimeout(() => {
  console.log('Timeout 2');
}, 0);  // 0 doesn't mean immediate!

console.log('End');

// Output: Start, End, Timeout 2, Timeout 1
```

**Important:** The delay is the *minimum* time, not guaranteed exact time.

```javascript
// This won't block for 2 seconds total
setTimeout(() => console.log('A'), 1000);
setTimeout(() => console.log('B'), 1000);
// Both fire around the same time (1 second from start)
```

### setInterval(callback, delay)

```javascript
let count = 0;
const intervalId = setInterval(() => {
  count++;
  console.log(count);
  if (count >= 5) {
    clearInterval(intervalId);
  }
}, 1000);
```

**Problem with setInterval:** Can cause stacking if callback takes longer than interval.

```javascript
// Better: Use setTimeout recursively
function poll() {
  fetch('/api/status')
    .then(handleResponse)
    .finally(() => {
      setTimeout(poll, 1000);  // Schedule next after completion
    });
}
```

---

## Promises and the Microtask Queue

```javascript
console.log('sync 1');

setTimeout(() => console.log('timeout'), 0);

Promise.resolve()
  .then(() => {
    console.log('promise 1');
    return Promise.resolve();
  })
  .then(() => {
    console.log('promise 2');
  });

queueMicrotask(() => {
  console.log('microtask');
});

console.log('sync 2');

// Output:
// sync 1
// sync 2
// promise 1
// microtask
// promise 2
// timeout
```

---

## async/await and the Event Loop

`async/await` is syntactic sugar over promises:

```javascript
async function example() {
  console.log('1');
  await Promise.resolve();
  console.log('2');
}

console.log('3');
example();
console.log('4');

// Output: 3, 1, 4, 2
```

**Why?** `await` pauses the async function and adds the rest to microtask queue.

### Detailed Example

```javascript
async function async1() {
  console.log('async1 start');
  await async2();
  console.log('async1 end');
}

async function async2() {
  console.log('async2');
}

console.log('script start');

setTimeout(() => {
  console.log('setTimeout');
}, 0);

async1();

new Promise((resolve) => {
  console.log('promise1');
  resolve();
}).then(() => {
  console.log('promise2');
});

console.log('script end');

// Output:
// script start
// async1 start
// async2
// promise1
// script end
// async1 end
// promise2
// setTimeout
```

---

## requestAnimationFrame

Runs before the next repaint (~60 fps = ~16.67ms):

```javascript
function animate() {
  // Update animation
  element.style.left = position + 'px';
  position++;

  if (position < 200) {
    requestAnimationFrame(animate);
  }
}

requestAnimationFrame(animate);
```

**Priority:** After microtasks, before macrotasks, synced with display refresh.

---

## Common Interview Questions

### Q1: What's the output?

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');
```

**Answer:** `1, 4, 3, 2`

### Q2: What's the output?

```javascript
async function foo() {
  console.log('foo start');
  await bar();
  console.log('foo end');
}

async function bar() {
  console.log('bar');
}

console.log('start');
foo();
console.log('end');
```

**Answer:** `start, foo start, bar, end, foo end`

### Q3: Explain why setTimeout(fn, 0) isn't immediate

```javascript
console.log('start');
setTimeout(() => console.log('timeout'), 0);
console.log('end');
// Output: start, end, timeout
```

**Answer:** Even with 0 delay, setTimeout's callback goes to the macrotask queue. The event loop only processes it after:
1. Current synchronous code completes
2. All microtasks are processed

### Q4: What's the output?

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 1000);
}
```

**Answer:** `3, 3, 3` (all after 1 second)

**Fix:** Use `let` or closure.

### Q5: Implement a sleep function

```javascript
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function demo() {
  console.log('Start');
  await sleep(2000);
  console.log('After 2 seconds');
}
```

### Q6: What's the output?

```javascript
Promise.resolve()
  .then(() => {
    console.log('1');
    throw new Error('error');
  })
  .catch(() => {
    console.log('2');
  })
  .then(() => {
    console.log('3');
  });
```

**Answer:** `1, 2, 3`

The catch handles the error, and execution continues.

---

## Node.js Event Loop Differences

Node.js has additional phases:

```
   ┌───────────────────────────┐
┌─>│           timers          │  (setTimeout, setInterval)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  (I/O callbacks)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  (internal use)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  (retrieve new I/O events)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  (setImmediate)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  (socket.on('close', ...))
   └───────────────────────────┘
```

### setImmediate vs setTimeout

```javascript
// Order is non-deterministic in main module
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));

// Inside I/O callback, setImmediate is always first
const fs = require('fs');
fs.readFile(__filename, () => {
  setImmediate(() => console.log('immediate'));
  setTimeout(() => console.log('timeout'), 0);
});
// Output: immediate, timeout (guaranteed)
```

### process.nextTick

Higher priority than microtasks (Promises):

```javascript
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
// Output: nextTick, promise
```

---

## Key Takeaways

1. JavaScript is **single-threaded** but handles async via event loop
2. **Microtasks** (Promises) have higher priority than **macrotasks** (setTimeout)
3. Call stack must be empty before processing queues
4. **All microtasks** are processed before **one macrotask**
5. `async/await` uses promises internally (microtask queue)
6. `setTimeout(fn, 0)` is not immediate - minimum ~4ms in browsers
7. **Order matters:** sync → microtasks → macrotasks
