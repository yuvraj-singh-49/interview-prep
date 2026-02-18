# Concurrency and Parallelism in Python

## Overview

### Concurrency vs Parallelism

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Concurrency vs Parallelism                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Concurrency: Managing multiple tasks (may not run simultaneously)          │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┐                                │
│  │ T1 │ T2 │ T1 │ T3 │ T2 │ T1 │ T3 │ T2 │  (Interleaved on 1 CPU)        │
│  └────┴────┴────┴────┴────┴────┴────┴────┘                                │
│                                                                             │
│  Parallelism: Actually running multiple tasks simultaneously                │
│  CPU 1: ┌─────────────────────────────────┐                                │
│         │ T1 │ T1 │ T1 │ T1 │ T1 │ T1 │                                    │
│  CPU 2: ├─────────────────────────────────┤                                │
│         │ T2 │ T2 │ T2 │ T2 │ T2 │ T2 │                                    │
│  CPU 3: ├─────────────────────────────────┤                                │
│         │ T3 │ T3 │ T3 │ T3 │ T3 │ T3 │                                    │
│         └─────────────────────────────────┘                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Python Options:
┌─────────────────────────────────────────────────────────────────────────────┐
│ Approach        │ Best For           │ True Parallelism │ GIL Affected     │
├─────────────────────────────────────────────────────────────────────────────┤
│ threading       │ I/O-bound tasks    │ No               │ Yes              │
│ multiprocessing │ CPU-bound tasks    │ Yes              │ No (separate)    │
│ asyncio         │ I/O-bound, many    │ No               │ Yes              │
│                 │ concurrent tasks   │                  │                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The GIL (Global Interpreter Lock)

```python
"""
GIL = Global Interpreter Lock

What: Mutex that protects access to Python objects
Why: Prevents race conditions in CPython's memory management
Effect: Only one thread executes Python bytecode at a time

Implications:
- Threading doesn't help CPU-bound tasks (can't utilize multiple cores)
- Threading DOES help I/O-bound tasks (GIL released during I/O)
- Use multiprocessing for CPU-bound parallelism
"""

# CPU-bound: Threading won't help
import threading
import time

def cpu_bound(n):
    """Count to n (CPU-intensive)"""
    count = 0
    for i in range(n):
        count += 1
    return count

# Single threaded
start = time.perf_counter()
cpu_bound(10_000_000)
cpu_bound(10_000_000)
print(f"Sequential: {time.perf_counter() - start:.2f}s")

# Multi-threaded (won't be faster due to GIL!)
start = time.perf_counter()
t1 = threading.Thread(target=cpu_bound, args=(10_000_000,))
t2 = threading.Thread(target=cpu_bound, args=(10_000_000,))
t1.start(); t2.start()
t1.join(); t2.join()
print(f"Threaded: {time.perf_counter() - start:.2f}s")

# I/O-bound: Threading DOES help
import urllib.request

def io_bound(url):
    """Fetch URL (I/O-intensive)"""
    return urllib.request.urlopen(url).read()

# Threading helps here because GIL is released during I/O wait
```

---

## Threading

### Basic Threading

```python
import threading
import time

# Create a thread
def worker(name, delay):
    print(f"{name} starting")
    time.sleep(delay)
    print(f"{name} finished")

# Method 1: Thread with function
t = threading.Thread(target=worker, args=("Thread-1", 2))
t.start()           # Start the thread
t.join()            # Wait for thread to complete

# Method 2: Thread with lambda
t = threading.Thread(target=lambda: worker("Thread-2", 1))
t.start()

# Method 3: Subclass Thread
class MyThread(threading.Thread):
    def __init__(self, name, delay):
        super().__init__()
        self.name = name
        self.delay = delay

    def run(self):
        print(f"{self.name} starting")
        time.sleep(self.delay)
        print(f"{self.name} finished")

t = MyThread("Thread-3", 1)
t.start()
t.join()

# Thread properties
t.is_alive()        # Check if running
t.name              # Thread name
t.daemon            # Daemon thread?
threading.current_thread()  # Current thread
threading.active_count()    # Number of active threads
threading.enumerate()       # List of active threads
```

### Daemon Threads

```python
import threading
import time

def background_task():
    while True:
        print("Background task running...")
        time.sleep(1)

# Daemon threads are killed when main program exits
daemon = threading.Thread(target=background_task, daemon=True)
daemon.start()

time.sleep(3)
print("Main program exiting - daemon will be killed")
# Program exits, daemon thread is killed
```

### Thread Synchronization

```python
import threading

# Shared resource without synchronization (DANGEROUS)
counter = 0

def increment():
    global counter
    for _ in range(100000):
        counter += 1  # Not atomic! Read-modify-write

threads = [threading.Thread(target=increment) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)  # Expected: 500000, Actual: unpredictable!

# Fix 1: Lock (Mutex)
counter = 0
lock = threading.Lock()

def increment_safe():
    global counter
    for _ in range(100000):
        with lock:  # Acquire lock, auto-release
            counter += 1

# Fix 2: RLock (Reentrant Lock) - can be acquired multiple times by same thread
rlock = threading.RLock()

def recursive_function(n):
    with rlock:
        if n > 0:
            recursive_function(n - 1)  # Same thread can re-acquire

# Semaphore - limit concurrent access
semaphore = threading.Semaphore(3)  # Max 3 threads at once

def limited_access():
    with semaphore:
        print("Accessing resource")
        time.sleep(1)

# Event - thread coordination
event = threading.Event()

def waiter():
    print("Waiting for event...")
    event.wait()  # Block until event is set
    print("Event received!")

def setter():
    time.sleep(2)
    event.set()  # Signal waiting threads

# Condition - complex coordination
condition = threading.Condition()
items = []

def producer():
    with condition:
        items.append("item")
        condition.notify()  # Wake up one waiting thread

def consumer():
    with condition:
        while not items:
            condition.wait()  # Release lock and wait
        return items.pop()

# Barrier - synchronize N threads
barrier = threading.Barrier(3)

def synchronized_task():
    print("Preparing...")
    barrier.wait()  # All 3 threads must reach here
    print("Go!")
```

### ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def task(n):
    time.sleep(n)
    return f"Task {n} completed"

# Basic usage
with ThreadPoolExecutor(max_workers=4) as executor:
    # Submit single task
    future = executor.submit(task, 2)
    result = future.result()  # Blocks until complete

    # Submit multiple tasks
    futures = [executor.submit(task, i) for i in range(5)]

    # Process as completed (not in submission order)
    for future in as_completed(futures):
        print(future.result())

# map - ordered results
with ThreadPoolExecutor(max_workers=4) as executor:
    results = executor.map(task, [1, 2, 3, 4, 5])
    for result in results:  # Results in submission order
        print(result)

# Timeout
future = executor.submit(task, 10)
try:
    result = future.result(timeout=5)  # Raise TimeoutError after 5s
except TimeoutError:
    print("Task timed out")

# Cancel (only if not started)
future = executor.submit(task, 10)
cancelled = future.cancel()  # True if cancelled
```

### Thread-Local Data

```python
import threading

# Thread-local storage
local_data = threading.local()

def process():
    # Each thread has its own copy
    local_data.value = threading.current_thread().name
    time.sleep(1)
    print(f"{threading.current_thread().name}: {local_data.value}")

threads = [threading.Thread(target=process) for _ in range(3)]
for t in threads: t.start()
for t in threads: t.join()
```

---

## Multiprocessing

### Basic Multiprocessing

```python
from multiprocessing import Process, current_process
import os

def worker(name):
    print(f"{name} running in process {os.getpid()}")
    return f"{name} done"

# Create and start process
p = Process(target=worker, args=("Process-1",))
p.start()
p.join()

# Process properties
p.pid               # Process ID
p.is_alive()        # Check if running
p.name              # Process name
p.exitcode          # Exit code (None if still running)
p.daemon            # Daemon process?

# Multiple processes
processes = []
for i in range(4):
    p = Process(target=worker, args=(f"Process-{i}",))
    processes.append(p)
    p.start()

for p in processes:
    p.join()
```

### Inter-Process Communication

```python
from multiprocessing import Process, Queue, Pipe, Value, Array

# Queue - thread/process safe FIFO
def producer(q):
    for i in range(5):
        q.put(f"item-{i}")
    q.put(None)  # Sentinel to signal done

def consumer(q):
    while True:
        item = q.get()
        if item is None:
            break
        print(f"Got: {item}")

q = Queue()
p1 = Process(target=producer, args=(q,))
p2 = Process(target=consumer, args=(q,))
p1.start(); p2.start()
p1.join(); p2.join()

# Pipe - bidirectional communication
def sender(conn):
    conn.send("Hello from sender")
    conn.close()

def receiver(conn):
    msg = conn.recv()
    print(f"Received: {msg}")

parent_conn, child_conn = Pipe()
p1 = Process(target=sender, args=(parent_conn,))
p2 = Process(target=receiver, args=(child_conn,))
p1.start(); p2.start()
p1.join(); p2.join()

# Shared Memory - Value and Array
def increment_shared(val, arr):
    val.value += 1
    for i in range(len(arr)):
        arr[i] += 1

shared_val = Value('i', 0)      # 'i' = integer
shared_arr = Array('i', [1, 2, 3, 4, 5])

processes = []
for _ in range(4):
    p = Process(target=increment_shared, args=(shared_val, shared_arr))
    processes.append(p)
    p.start()

for p in processes:
    p.join()

print(shared_val.value)  # 4
print(list(shared_arr))  # [5, 6, 7, 8, 9]

# Note: For complex objects, use Manager
from multiprocessing import Manager

with Manager() as manager:
    shared_list = manager.list([1, 2, 3])
    shared_dict = manager.dict()
```

### ProcessPoolExecutor

```python
from concurrent.futures import ProcessPoolExecutor, as_completed
import math

def cpu_intensive(n):
    """Find sum of prime factors"""
    total = 0
    for i in range(2, int(math.sqrt(n)) + 1):
        while n % i == 0:
            total += i
            n //= i
    if n > 1:
        total += n
    return total

numbers = list(range(10000, 10100))

# Single process
import time
start = time.perf_counter()
results = [cpu_intensive(n) for n in numbers]
print(f"Sequential: {time.perf_counter() - start:.2f}s")

# Multiple processes
start = time.perf_counter()
with ProcessPoolExecutor() as executor:
    results = list(executor.map(cpu_intensive, numbers))
print(f"Parallel: {time.perf_counter() - start:.2f}s")

# With as_completed
with ProcessPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(cpu_intensive, n): n for n in numbers}
    for future in as_completed(futures):
        n = futures[future]
        result = future.result()
        print(f"Sum of prime factors of {n}: {result}")
```

### Pool

```python
from multiprocessing import Pool

def square(x):
    return x ** 2

# Pool of workers
with Pool(processes=4) as pool:
    # map - apply function to each element
    results = pool.map(square, range(10))

    # map_async - non-blocking
    async_result = pool.map_async(square, range(10))
    results = async_result.get(timeout=10)

    # apply - single function call
    result = pool.apply(square, (5,))

    # apply_async - non-blocking single call
    async_result = pool.apply_async(square, (5,))
    result = async_result.get()

    # starmap - map with multiple arguments
    def add(a, b):
        return a + b
    results = pool.starmap(add, [(1, 2), (3, 4), (5, 6)])

    # imap - lazy iterator (memory efficient)
    for result in pool.imap(square, range(10)):
        print(result)

    # imap_unordered - results as they complete
    for result in pool.imap_unordered(square, range(10)):
        print(result)
```

---

## Asyncio

### Async/Await Basics

```python
import asyncio

# Coroutine function
async def say_hello(name, delay):
    await asyncio.sleep(delay)  # Non-blocking sleep
    print(f"Hello, {name}!")
    return f"Greeted {name}"

# Run coroutine
result = asyncio.run(say_hello("Alice", 1))

# Multiple coroutines concurrently
async def main():
    # Method 1: gather - run concurrently, get results in order
    results = await asyncio.gather(
        say_hello("Alice", 2),
        say_hello("Bob", 1),
        say_hello("Charlie", 3)
    )
    print(results)  # ['Greeted Alice', 'Greeted Bob', 'Greeted Charlie']

    # Method 2: create_task - schedule coroutines
    task1 = asyncio.create_task(say_hello("Alice", 2))
    task2 = asyncio.create_task(say_hello("Bob", 1))
    await task1
    await task2

asyncio.run(main())
```

### Task Management

```python
import asyncio

async def long_running_task(name, duration):
    print(f"{name} starting")
    await asyncio.sleep(duration)
    print(f"{name} completed")
    return f"{name} result"

async def main():
    # Create tasks
    task1 = asyncio.create_task(long_running_task("Task 1", 2))
    task2 = asyncio.create_task(long_running_task("Task 2", 1))

    # Task properties
    print(task1.done())      # False
    print(task1.cancelled()) # False

    # Cancel task
    task1.cancel()
    try:
        await task1
    except asyncio.CancelledError:
        print("Task 1 was cancelled")

    # Wait with timeout
    try:
        await asyncio.wait_for(
            long_running_task("Task 3", 10),
            timeout=2
        )
    except asyncio.TimeoutError:
        print("Task 3 timed out")

    # Wait for first completion
    done, pending = await asyncio.wait(
        [task2],
        return_when=asyncio.FIRST_COMPLETED
    )

asyncio.run(main())
```

### Async Context Managers and Iterators

```python
import asyncio

# Async context manager
class AsyncResource:
    async def __aenter__(self):
        print("Acquiring resource")
        await asyncio.sleep(1)
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("Releasing resource")
        await asyncio.sleep(0.5)

async def use_resource():
    async with AsyncResource() as resource:
        print("Using resource")

# Async iterator
class AsyncCounter:
    def __init__(self, stop):
        self.current = 0
        self.stop = stop

    def __aiter__(self):
        return self

    async def __anext__(self):
        if self.current >= self.stop:
            raise StopAsyncIteration
        await asyncio.sleep(0.1)
        self.current += 1
        return self.current

async def iterate():
    async for num in AsyncCounter(5):
        print(num)

# Async generator
async def async_range(stop):
    for i in range(stop):
        await asyncio.sleep(0.1)
        yield i

async def use_async_gen():
    async for num in async_range(5):
        print(num)

asyncio.run(iterate())
```

### Synchronization Primitives

```python
import asyncio

# Lock
lock = asyncio.Lock()

async def critical_section(name):
    async with lock:
        print(f"{name} acquired lock")
        await asyncio.sleep(1)
        print(f"{name} releasing lock")

# Semaphore
semaphore = asyncio.Semaphore(2)  # Max 2 concurrent

async def limited_task(name):
    async with semaphore:
        print(f"{name} running")
        await asyncio.sleep(1)

# Event
event = asyncio.Event()

async def waiter(name):
    print(f"{name} waiting")
    await event.wait()
    print(f"{name} proceeding")

async def setter():
    await asyncio.sleep(2)
    event.set()

# Queue
queue = asyncio.Queue()

async def producer(q):
    for i in range(5):
        await q.put(i)
        await asyncio.sleep(0.5)

async def consumer(q):
    while True:
        item = await q.get()
        print(f"Consumed: {item}")
        q.task_done()

async def main():
    q = asyncio.Queue()
    producer_task = asyncio.create_task(producer(q))
    consumer_task = asyncio.create_task(consumer(q))
    await producer_task
    await q.join()  # Wait for all items to be processed
    consumer_task.cancel()
```

### Real-World Async Example

```python
import asyncio
import aiohttp  # pip install aiohttp

async def fetch_url(session, url):
    """Fetch a URL asynchronously."""
    async with session.get(url) as response:
        return await response.text()

async def fetch_all(urls):
    """Fetch multiple URLs concurrently."""
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        return results

async def main():
    urls = [
        "https://api.github.com",
        "https://httpbin.org/get",
        "https://jsonplaceholder.typicode.com/posts/1"
    ]
    results = await fetch_all(urls)
    for url, result in zip(urls, results):
        if isinstance(result, Exception):
            print(f"{url}: Error - {result}")
        else:
            print(f"{url}: {len(result)} bytes")

asyncio.run(main())
```

---

## Choosing the Right Approach

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Decision Guide                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Is your task I/O-bound or CPU-bound?                                       │
│                                                                             │
│  I/O-bound (network, disk, database):                                       │
│  ├── Few tasks? → threading                                                 │
│  ├── Many concurrent tasks? → asyncio                                       │
│  └── Need existing sync code? → threading + ThreadPoolExecutor              │
│                                                                             │
│  CPU-bound (computation, data processing):                                  │
│  ├── Need parallelism? → multiprocessing                                    │
│  ├── NumPy/Pandas? → They release GIL, threading can help                   │
│  └── Need to share state? → multiprocessing + Manager/Queue                 │
│                                                                             │
│  Mixed workload:                                                            │
│  └── Use asyncio + ProcessPoolExecutor for CPU tasks                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Performance Comparison

```python
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# I/O-bound task simulation
def io_task(n):
    time.sleep(0.1)
    return n

async def async_io_task(n):
    await asyncio.sleep(0.1)
    return n

# CPU-bound task
def cpu_task(n):
    return sum(i*i for i in range(n))

# Benchmark
def benchmark():
    N = 100

    # Sequential I/O
    start = time.perf_counter()
    [io_task(i) for i in range(N)]
    print(f"Sequential I/O: {time.perf_counter() - start:.2f}s")

    # Threaded I/O
    start = time.perf_counter()
    with ThreadPoolExecutor(max_workers=10) as e:
        list(e.map(io_task, range(N)))
    print(f"Threaded I/O: {time.perf_counter() - start:.2f}s")

    # Async I/O
    async def run_async():
        return await asyncio.gather(*[async_io_task(i) for i in range(N)])

    start = time.perf_counter()
    asyncio.run(run_async())
    print(f"Async I/O: {time.perf_counter() - start:.2f}s")

    # Sequential CPU
    start = time.perf_counter()
    [cpu_task(100000) for _ in range(N)]
    print(f"Sequential CPU: {time.perf_counter() - start:.2f}s")

    # Multiprocess CPU
    start = time.perf_counter()
    with ProcessPoolExecutor() as e:
        list(e.map(cpu_task, [100000] * N))
    print(f"Multiprocess CPU: {time.perf_counter() - start:.2f}s")

benchmark()
```

---

## Interview Questions

```python
# Q1: Explain the GIL and its implications
"""
GIL = Global Interpreter Lock
- Only one thread executes Python bytecode at a time
- Protects CPython's reference counting
- Threading doesn't help CPU-bound tasks
- Threading helps I/O-bound (GIL released during I/O)
- Use multiprocessing for CPU parallelism
"""

# Q2: When would you use threading vs multiprocessing vs asyncio?
"""
Threading: I/O-bound, existing sync code, simpler mental model
Multiprocessing: CPU-bound, need true parallelism
Asyncio: Many concurrent I/O tasks, high concurrency requirements
"""

# Q3: What is a race condition and how do you prevent it?
"""
Race condition: Multiple threads/processes access shared data
concurrently, leading to unpredictable results.

Prevention:
- Locks/Mutexes
- Thread-safe data structures (Queue)
- Atomic operations
- Immutable data
- Thread-local storage
"""

# Q4: Implement a thread-safe counter
import threading

class ThreadSafeCounter:
    def __init__(self):
        self._value = 0
        self._lock = threading.Lock()

    def increment(self):
        with self._lock:
            self._value += 1

    @property
    def value(self):
        with self._lock:
            return self._value

# Q5: Implement async rate limiter
class RateLimiter:
    def __init__(self, rate, per):
        self.rate = rate
        self.per = per
        self.semaphore = asyncio.Semaphore(rate)
        self._tokens = rate
        self._last_update = time.monotonic()

    async def acquire(self):
        await self.semaphore.acquire()
        asyncio.create_task(self._release_later())

    async def _release_later(self):
        await asyncio.sleep(self.per)
        self.semaphore.release()
```
