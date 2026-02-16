# Concurrency & Multithreading Fundamentals

## Overview

Lead engineers must understand concurrency for designing scalable systems. This covers threading, synchronization primitives, async patterns, and common pitfalls.

---

## Key Concepts

### Process vs Thread

```
Process:
- Independent memory space
- Heavier to create/switch
- True parallelism
- Inter-process communication (IPC) needed

Thread:
- Shared memory within process
- Lighter weight
- Can run concurrently/parallel
- Easier communication, but needs synchronization
```

### Concurrency vs Parallelism

```
Concurrency: Dealing with multiple things at once (structure)
- Single core can handle via time-slicing
- I/O bound tasks benefit

Parallelism: Doing multiple things at once (execution)
- Requires multiple cores
- CPU bound tasks benefit
```

---

## Python Concurrency

### Threading

**Python:** GIL (Global Interpreter Lock) limits true parallelism for CPU-bound tasks.

```python
import threading
import time

# Basic thread
def worker(name, delay):
    print(f"{name} starting")
    time.sleep(delay)
    print(f"{name} finished")

# Create and start threads
threads = []
for i in range(3):
    t = threading.Thread(target=worker, args=(f"Thread-{i}", i))
    threads.append(t)
    t.start()

# Wait for completion
for t in threads:
    t.join()

print("All threads done")
```

### Thread with Return Value

```python
import threading
from concurrent.futures import ThreadPoolExecutor

# Using ThreadPoolExecutor (preferred)
def fetch_data(url):
    time.sleep(1)  # Simulate I/O
    return f"Data from {url}"

with ThreadPoolExecutor(max_workers=5) as executor:
    urls = ["url1", "url2", "url3"]
    futures = [executor.submit(fetch_data, url) for url in urls]

    for future in futures:
        print(future.result())

# Or using map
with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(fetch_data, urls))
```

### Locks (Mutex)

```python
import threading

class BankAccount:
    def __init__(self, balance):
        self.balance = balance
        self.lock = threading.Lock()

    def withdraw(self, amount):
        # Without lock - race condition possible
        # if self.balance >= amount:
        #     self.balance -= amount

        # With lock - thread-safe
        with self.lock:
            if self.balance >= amount:
                self.balance -= amount
                return True
            return False

    def deposit(self, amount):
        with self.lock:
            self.balance += amount

# Multiple threads accessing same account
account = BankAccount(1000)

def make_withdrawals():
    for _ in range(100):
        account.withdraw(10)

threads = [threading.Thread(target=make_withdrawals) for _ in range(10)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(f"Final balance: {account.balance}")
```

### RLock (Reentrant Lock)

```python
import threading

# Regular lock would deadlock if same thread tries to acquire twice
lock = threading.RLock()

def outer():
    with lock:
        print("Outer acquired")
        inner()  # Same thread can re-acquire

def inner():
    with lock:  # Works with RLock, deadlocks with Lock
        print("Inner acquired")

outer()
```

### Semaphore

```python
import threading
import time

# Limit concurrent access to resource
class ConnectionPool:
    def __init__(self, max_connections):
        self.semaphore = threading.Semaphore(max_connections)
        self.connections = []

    def get_connection(self):
        self.semaphore.acquire()
        return f"Connection-{threading.current_thread().name}"

    def release_connection(self, conn):
        self.semaphore.release()

pool = ConnectionPool(3)

def worker():
    conn = pool.get_connection()
    print(f"Using {conn}")
    time.sleep(1)
    pool.release_connection(conn)
    print(f"Released {conn}")

threads = [threading.Thread(target=worker) for _ in range(10)]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

### Condition Variables

```python
import threading

class BoundedBuffer:
    def __init__(self, capacity):
        self.capacity = capacity
        self.buffer = []
        self.lock = threading.Lock()
        self.not_full = threading.Condition(self.lock)
        self.not_empty = threading.Condition(self.lock)

    def put(self, item):
        with self.not_full:
            while len(self.buffer) >= self.capacity:
                self.not_full.wait()  # Release lock, wait for signal
            self.buffer.append(item)
            self.not_empty.notify()  # Signal consumers

    def get(self):
        with self.not_empty:
            while len(self.buffer) == 0:
                self.not_empty.wait()
            item = self.buffer.pop(0)
            self.not_full.notify()  # Signal producers
            return item

# Producer-Consumer example
buffer = BoundedBuffer(5)

def producer():
    for i in range(10):
        buffer.put(i)
        print(f"Produced {i}")

def consumer():
    for _ in range(10):
        item = buffer.get()
        print(f"Consumed {item}")

threading.Thread(target=producer).start()
threading.Thread(target=consumer).start()
```

### Event

```python
import threading

# Simple signaling mechanism
event = threading.Event()

def waiter():
    print("Waiting for event...")
    event.wait()  # Blocks until event is set
    print("Event received!")

def setter():
    time.sleep(2)
    print("Setting event")
    event.set()

threading.Thread(target=waiter).start()
threading.Thread(target=setter).start()
```

---

## Python Async/Await

Best for I/O-bound operations (network, file I/O).

```python
import asyncio

# Basic async function
async def fetch_data(url):
    print(f"Fetching {url}")
    await asyncio.sleep(1)  # Simulated I/O
    return f"Data from {url}"

# Running async code
async def main():
    # Sequential
    data1 = await fetch_data("url1")
    data2 = await fetch_data("url2")

    # Concurrent (better!)
    results = await asyncio.gather(
        fetch_data("url1"),
        fetch_data("url2"),
        fetch_data("url3")
    )
    print(results)

asyncio.run(main())
```

### Async with Timeout

```python
import asyncio

async def slow_operation():
    await asyncio.sleep(10)
    return "Done"

async def main():
    try:
        result = await asyncio.wait_for(slow_operation(), timeout=2.0)
    except asyncio.TimeoutError:
        print("Operation timed out!")

asyncio.run(main())
```

### Async Queue

```python
import asyncio

async def producer(queue):
    for i in range(5):
        await asyncio.sleep(0.5)
        await queue.put(i)
        print(f"Produced {i}")
    await queue.put(None)  # Sentinel

async def consumer(queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        print(f"Consumed {item}")
        queue.task_done()

async def main():
    queue = asyncio.Queue()
    await asyncio.gather(producer(queue), consumer(queue))

asyncio.run(main())
```

### Async Lock

```python
import asyncio

class AsyncBankAccount:
    def __init__(self, balance):
        self.balance = balance
        self.lock = asyncio.Lock()

    async def withdraw(self, amount):
        async with self.lock:
            if self.balance >= amount:
                await asyncio.sleep(0.1)  # Simulate processing
                self.balance -= amount
                return True
            return False

    async def deposit(self, amount):
        async with self.lock:
            await asyncio.sleep(0.1)
            self.balance += amount
```

---

## JavaScript Concurrency

### Event Loop

JavaScript is single-threaded with an event loop for async operations.

```javascript
// Event loop order: Call Stack -> Microtasks -> Macrotasks
console.log("1"); // Call stack

setTimeout(() => console.log("2"), 0); // Macrotask

Promise.resolve().then(() => console.log("3")); // Microtask

console.log("4"); // Call stack

// Output: 1, 4, 3, 2
```

### Promises

```javascript
// Creating a promise
function fetchData(url) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (url) {
        resolve(`Data from ${url}`);
      } else {
        reject("No URL provided");
      }
    }, 1000);
  });
}

// Using promises
fetchData("api/users")
  .then(data => console.log(data))
  .catch(err => console.error(err))
  .finally(() => console.log("Done"));

// Chaining
fetchData("api/users")
  .then(users => fetchData("api/posts"))
  .then(posts => console.log(posts));
```

### Async/Await

```javascript
// Async function
async function getData() {
  try {
    const data = await fetchData("api/users");
    console.log(data);
  } catch (error) {
    console.error(error);
  }
}

// Parallel execution
async function getMultipleData() {
  // Sequential (slow)
  const data1 = await fetchData("url1"); // Waits
  const data2 = await fetchData("url2"); // Then waits

  // Parallel (fast)
  const [result1, result2] = await Promise.all([
    fetchData("url1"),
    fetchData("url2")
  ]);

  // Parallel with error handling (first error rejects all)
  const results = await Promise.all([
    fetchData("url1"),
    fetchData("url2"),
    fetchData("url3")
  ]);

  // Parallel, settle all (get all results even with errors)
  const settled = await Promise.allSettled([
    fetchData("url1"),
    fetchData("invalid")
  ]);
  // [{status: "fulfilled", value: ...}, {status: "rejected", reason: ...}]
}
```

### Promise.race and Promise.any

```javascript
// First to resolve/reject wins
const fastest = await Promise.race([
  fetchData("server1"),
  fetchData("server2")
]);

// First to resolve wins (ignores rejections)
const firstSuccess = await Promise.any([
  fetchData("server1"),
  fetchData("server2")
]);
```

### Web Workers (True Parallelism)

```javascript
// main.js
const worker = new Worker("worker.js");

worker.postMessage({ data: [1, 2, 3, 4, 5] });

worker.onmessage = function(e) {
  console.log("Result from worker:", e.data);
};

worker.onerror = function(error) {
  console.error("Worker error:", error);
};

// worker.js
self.onmessage = function(e) {
  const data = e.data.data;
  const result = data.map(x => x * 2); // CPU-intensive work
  self.postMessage(result);
};
```

### Async Patterns

```javascript
// Rate limiting with async queue
class AsyncQueue {
  constructor(concurrency) {
    this.concurrency = concurrency;
    this.running = 0;
    this.queue = [];
  }

  async push(task) {
    if (this.running >= this.concurrency) {
      await new Promise(resolve => this.queue.push(resolve));
    }
    this.running++;
    try {
      return await task();
    } finally {
      this.running--;
      if (this.queue.length > 0) {
        const next = this.queue.shift();
        next();
      }
    }
  }
}

// Usage
const queue = new AsyncQueue(3); // Max 3 concurrent

const tasks = Array(10).fill(null).map((_, i) =>
  queue.push(async () => {
    console.log(`Starting task ${i}`);
    await delay(1000);
    console.log(`Finished task ${i}`);
    return i;
  })
);

await Promise.all(tasks);
```

### Debounce and Throttle

```javascript
// Debounce: Execute after delay with no new calls
function debounce(fn, delay) {
  let timeoutId;
  return function(...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Throttle: Execute at most once per interval
function throttle(fn, interval) {
  let lastTime = 0;
  return function(...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}

// Usage
const debouncedSearch = debounce(query => fetch(`/search?q=${query}`), 300);
const throttledScroll = throttle(() => console.log("scroll"), 100);
```

---

## Common Concurrency Problems

### 1. Race Condition

```python
# Problem: Multiple threads modify shared state
counter = 0

def increment():
    global counter
    for _ in range(100000):
        counter += 1  # Not atomic!

# Solution: Use lock
lock = threading.Lock()

def safe_increment():
    global counter
    for _ in range(100000):
        with lock:
            counter += 1
```

### 2. Deadlock

```python
# Problem: Circular wait for resources
lock_a = threading.Lock()
lock_b = threading.Lock()

def thread1():
    with lock_a:
        time.sleep(0.1)
        with lock_b:  # Waits for lock_b
            pass

def thread2():
    with lock_b:
        time.sleep(0.1)
        with lock_a:  # Waits for lock_a - DEADLOCK!
            pass

# Solution 1: Lock ordering (always acquire in same order)
def safe_thread1():
    with lock_a:
        with lock_b:
            pass

def safe_thread2():
    with lock_a:  # Same order
        with lock_b:
            pass

# Solution 2: Try-lock with timeout
def trylock_thread():
    acquired_a = lock_a.acquire(timeout=1)
    if acquired_a:
        acquired_b = lock_b.acquire(timeout=1)
        if acquired_b:
            try:
                pass  # Do work
            finally:
                lock_b.release()
        lock_a.release()
```

### 3. Starvation

```python
# Problem: Low priority threads never get resources
# Solution: Fair locks, priority inheritance, or timeout-based approach

# Using a fair semaphore
import queue

class FairLock:
    def __init__(self):
        self.queue = queue.Queue()
        self.locked = False
        self.lock = threading.Lock()

    def acquire(self):
        event = threading.Event()
        with self.lock:
            if not self.locked:
                self.locked = True
                return
            self.queue.put(event)
        event.wait()  # Fair ordering via queue

    def release(self):
        with self.lock:
            if not self.queue.empty():
                event = self.queue.get()
                event.set()
            else:
                self.locked = False
```

### 4. Livelock

```python
# Problem: Threads repeatedly respond to each other, making no progress
# Example: Two people in a hallway, both keep stepping aside

# Solution: Add randomization/backoff
import random

def polite_thread(name, resource):
    while True:
        resource.lock.acquire()
        if resource.available:
            print(f"{name} got resource")
            resource.available = False
            resource.lock.release()
            break
        resource.lock.release()
        # Random backoff to break livelock
        time.sleep(random.uniform(0.01, 0.1))
```

---

## Thread-Safe Data Structures

### Python

```python
import queue
from collections import deque
import threading

# Thread-safe queue
q = queue.Queue()
q.put(item)
item = q.get()

# Thread-safe with blocking
q.put(item, block=True, timeout=5)
item = q.get(block=True, timeout=5)

# Thread-safe counter
class AtomicCounter:
    def __init__(self):
        self.value = 0
        self.lock = threading.Lock()

    def increment(self):
        with self.lock:
            self.value += 1
            return self.value

    def decrement(self):
        with self.lock:
            self.value -= 1
            return self.value
```

### JavaScript

```javascript
// SharedArrayBuffer for shared memory between workers
const sharedBuffer = new SharedArrayBuffer(4);
const sharedArray = new Int32Array(sharedBuffer);

// Atomic operations
Atomics.add(sharedArray, 0, 1); // Atomic increment
Atomics.sub(sharedArray, 0, 1); // Atomic decrement
Atomics.load(sharedArray, 0);   // Atomic read
Atomics.store(sharedArray, 0, 5); // Atomic write

// Atomic wait/notify (for coordination)
Atomics.wait(sharedArray, 0, 0); // Wait while value is 0
Atomics.notify(sharedArray, 0, 1); // Wake one waiter
```

---

## Multiprocessing (Python)

For CPU-bound tasks, use multiprocessing to bypass GIL.

```python
from multiprocessing import Pool, Process, Queue, Manager
import os

# Basic process
def worker(name):
    print(f"Worker {name}, PID: {os.getpid()}")

if __name__ == "__main__":
    processes = []
    for i in range(4):
        p = Process(target=worker, args=(i,))
        processes.append(p)
        p.start()

    for p in processes:
        p.join()

# Process pool
def cpu_intensive(x):
    return x ** 2

if __name__ == "__main__":
    with Pool(4) as pool:
        results = pool.map(cpu_intensive, range(10))
        print(results)

# Shared state with Manager
def increment_shared(counter, lock):
    for _ in range(1000):
        with lock:
            counter.value += 1

if __name__ == "__main__":
    with Manager() as manager:
        counter = manager.Value('i', 0)
        lock = manager.Lock()

        processes = [Process(target=increment_shared, args=(counter, lock))
                    for _ in range(4)]
        for p in processes:
            p.start()
        for p in processes:
            p.join()

        print(f"Final: {counter.value}")
```

---

## When to Use What

| Scenario | Python | JavaScript |
|----------|--------|------------|
| **I/O bound** | `asyncio`, `threading` | `async/await`, Promises |
| **CPU bound** | `multiprocessing` | Web Workers |
| **Simple parallelism** | `ThreadPoolExecutor` | `Promise.all` |
| **Shared state** | Locks, Queues | Atomics, SharedArrayBuffer |
| **Rate limiting** | Semaphore | Async Queue |

---

## Interview Tips

1. **Know the difference** between concurrency and parallelism
2. **Understand GIL** in Python and its implications
3. **Identify deadlock conditions**: Mutual exclusion, hold and wait, no preemption, circular wait
4. **Always consider**: What happens if this code runs concurrently?
5. **Prefer higher-level abstractions**: ThreadPoolExecutor, async/await over raw threads
6. **Test concurrent code**: Use stress tests, race condition detectors
