# Promises and Async/Await

## Overview

Promises and async/await are fundamental to modern JavaScript. They provide a clean way to handle asynchronous operations.

---

## Promises

### Creating a Promise

```javascript
const promise = new Promise((resolve, reject) => {
  // Asynchronous operation
  setTimeout(() => {
    const success = true;

    if (success) {
      resolve('Data fetched successfully');
    } else {
      reject(new Error('Failed to fetch data'));
    }
  }, 1000);
});
```

### Promise States

1. **Pending** - Initial state, neither fulfilled nor rejected
2. **Fulfilled** - Operation completed successfully
3. **Rejected** - Operation failed

Once settled (fulfilled/rejected), a promise cannot change state.

### Consuming Promises

```javascript
promise
  .then((result) => {
    console.log('Success:', result);
    return result.toUpperCase();
  })
  .then((upperResult) => {
    console.log('Upper:', upperResult);
  })
  .catch((error) => {
    console.error('Error:', error.message);
  })
  .finally(() => {
    console.log('Cleanup - runs regardless');
  });
```

---

## Promise Methods

### Promise.all()

Waits for all promises to resolve (fails fast on first rejection):

```javascript
const p1 = Promise.resolve(1);
const p2 = Promise.resolve(2);
const p3 = Promise.resolve(3);

Promise.all([p1, p2, p3])
  .then(([r1, r2, r3]) => {
    console.log(r1, r2, r3);  // 1, 2, 3
  });

// If any rejects, entire Promise.all rejects
const p4 = Promise.reject('Error');

Promise.all([p1, p4, p3])
  .catch((error) => {
    console.log(error);  // 'Error'
  });
```

### Promise.allSettled()

Waits for all promises to settle (doesn't fail fast):

```javascript
const promises = [
  Promise.resolve('Success'),
  Promise.reject('Error'),
  Promise.resolve('Another success')
];

Promise.allSettled(promises)
  .then((results) => {
    console.log(results);
    /*
    [
      { status: 'fulfilled', value: 'Success' },
      { status: 'rejected', reason: 'Error' },
      { status: 'fulfilled', value: 'Another success' }
    ]
    */
  });
```

### Promise.race()

Returns first settled promise (either fulfilled or rejected):

```javascript
const slow = new Promise(resolve => setTimeout(() => resolve('Slow'), 2000));
const fast = new Promise(resolve => setTimeout(() => resolve('Fast'), 100));

Promise.race([slow, fast])
  .then(result => console.log(result));  // 'Fast'

// Useful for timeouts
function fetchWithTimeout(url, timeout) {
  return Promise.race([
    fetch(url),
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Timeout')), timeout)
    )
  ]);
}
```

### Promise.any()

Returns first fulfilled promise (ignores rejections until all reject):

```javascript
const promises = [
  Promise.reject('Error 1'),
  Promise.resolve('Success'),
  Promise.reject('Error 2')
];

Promise.any(promises)
  .then(result => console.log(result));  // 'Success'

// All rejections = AggregateError
Promise.any([
  Promise.reject('Error 1'),
  Promise.reject('Error 2')
])
  .catch(error => {
    console.log(error.errors);  // ['Error 1', 'Error 2']
  });
```

### Promise.resolve() / Promise.reject()

```javascript
// Create resolved promise
const resolved = Promise.resolve('value');

// Create rejected promise
const rejected = Promise.reject(new Error('Failed'));

// Useful for returning consistent promise interface
function maybeAsync(value) {
  if (cached) {
    return Promise.resolve(cached);
  }
  return fetchFromServer(value);
}
```

---

## Promise Chaining

```javascript
fetch('/api/user')
  .then(response => response.json())
  .then(user => fetch(`/api/posts/${user.id}`))
  .then(response => response.json())
  .then(posts => {
    console.log('User posts:', posts);
  })
  .catch(error => {
    console.error('Any error in chain:', error);
  });
```

### Returning Values

```javascript
Promise.resolve(1)
  .then(x => x + 1)           // 2
  .then(x => x * 2)           // 4
  .then(x => Promise.resolve(x + 10))  // 14 (returning promise)
  .then(x => { console.log(x); });     // logs 14
```

### Error Handling in Chains

```javascript
Promise.resolve()
  .then(() => {
    throw new Error('Step 1 failed');
  })
  .then(() => {
    console.log('This is skipped');
  })
  .catch(error => {
    console.log('Caught:', error.message);
    return 'Recovered';  // Recovery
  })
  .then(value => {
    console.log('Continued with:', value);  // 'Recovered'
  });
```

---

## async/await

### Basic Syntax

```javascript
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    const user = await response.json();
    return user;
  } catch (error) {
    console.error('Failed to fetch user:', error);
    throw error;
  }
}

// Usage
const user = await fetchUser(1);
```

### async Functions Always Return Promises

```javascript
async function example() {
  return 42;
}

example().then(value => console.log(value));  // 42

// Equivalent to:
function example() {
  return Promise.resolve(42);
}
```

### await with Promise.all

```javascript
async function fetchAllData() {
  // Sequential (slow)
  const user = await fetchUser();
  const posts = await fetchPosts();
  const comments = await fetchComments();

  // Parallel (fast)
  const [user, posts, comments] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchComments()
  ]);

  return { user, posts, comments };
}
```

### Error Handling

```javascript
// try/catch
async function fetchData() {
  try {
    const data = await fetch('/api/data');
    return await data.json();
  } catch (error) {
    console.error('Error:', error);
    return null;
  }
}

// .catch() on the promise
async function fetchData() {
  const data = await fetch('/api/data')
    .catch(error => {
      console.error('Error:', error);
      return null;
    });

  return data ? await data.json() : null;
}

// Error wrapper utility
async function safeAwait(promise) {
  try {
    const result = await promise;
    return [result, null];
  } catch (error) {
    return [null, error];
  }
}

// Usage
const [user, error] = await safeAwait(fetchUser(1));
if (error) {
  console.log('Handle error');
}
```

---

## Implement Promises

### Basic Promise Implementation

```javascript
class MyPromise {
  constructor(executor) {
    this.state = 'pending';
    this.value = undefined;
    this.handlers = [];

    const resolve = (value) => {
      if (this.state !== 'pending') return;
      this.state = 'fulfilled';
      this.value = value;
      this.handlers.forEach(h => h.onFulfilled(value));
    };

    const reject = (error) => {
      if (this.state !== 'pending') return;
      this.state = 'rejected';
      this.value = error;
      this.handlers.forEach(h => h.onRejected(error));
    };

    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const handle = () => {
        try {
          if (this.state === 'fulfilled') {
            const result = onFulfilled
              ? onFulfilled(this.value)
              : this.value;
            resolve(result);
          } else if (this.state === 'rejected') {
            if (onRejected) {
              const result = onRejected(this.value);
              resolve(result);
            } else {
              reject(this.value);
            }
          }
        } catch (error) {
          reject(error);
        }
      };

      if (this.state === 'pending') {
        this.handlers.push({
          onFulfilled: () => handle(),
          onRejected: () => handle()
        });
      } else {
        queueMicrotask(handle);
      }
    });
  }

  catch(onRejected) {
    return this.then(null, onRejected);
  }

  finally(callback) {
    return this.then(
      value => {
        callback();
        return value;
      },
      error => {
        callback();
        throw error;
      }
    );
  }

  static resolve(value) {
    return new MyPromise(resolve => resolve(value));
  }

  static reject(error) {
    return new MyPromise((_, reject) => reject(error));
  }

  static all(promises) {
    return new MyPromise((resolve, reject) => {
      const results = [];
      let completed = 0;

      promises.forEach((promise, index) => {
        Promise.resolve(promise).then(
          value => {
            results[index] = value;
            completed++;
            if (completed === promises.length) {
              resolve(results);
            }
          },
          reject
        );
      });
    });
  }

  static race(promises) {
    return new MyPromise((resolve, reject) => {
      promises.forEach(promise => {
        Promise.resolve(promise).then(resolve, reject);
      });
    });
  }
}
```

---

## Common Patterns

### Sequential Execution

```javascript
async function processSequentially(items) {
  const results = [];

  for (const item of items) {
    const result = await processItem(item);
    results.push(result);
  }

  return results;
}

// Using reduce
function processSequentially(items) {
  return items.reduce((promise, item) => {
    return promise.then(results => {
      return processItem(item).then(result => [...results, result]);
    });
  }, Promise.resolve([]));
}
```

### Parallel with Concurrency Limit

```javascript
async function parallelLimit(items, limit, fn) {
  const results = [];
  const executing = [];

  for (const item of items) {
    const promise = fn(item).then(result => {
      executing.splice(executing.indexOf(promise), 1);
      return result;
    });

    results.push(promise);
    executing.push(promise);

    if (executing.length >= limit) {
      await Promise.race(executing);
    }
  }

  return Promise.all(results);
}

// Usage: Process 100 items, max 5 at a time
await parallelLimit(items, 5, processItem);
```

### Retry Logic

```javascript
async function retry(fn, retries = 3, delay = 1000) {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === retries - 1) throw error;
      await new Promise(r => setTimeout(r, delay));
    }
  }
}

// With exponential backoff
async function retryWithBackoff(fn, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === retries - 1) throw error;
      await new Promise(r => setTimeout(r, Math.pow(2, i) * 1000));
    }
  }
}
```

### Cancellable Promise

```javascript
function cancellable(promise) {
  let cancelled = false;

  const wrappedPromise = new Promise((resolve, reject) => {
    promise.then(
      value => cancelled ? reject({ cancelled: true }) : resolve(value),
      error => cancelled ? reject({ cancelled: true }) : reject(error)
    );
  });

  return {
    promise: wrappedPromise,
    cancel() {
      cancelled = true;
    }
  };
}

// Usage
const { promise, cancel } = cancellable(fetchData());
cancel();  // Cancel the operation
```

---

## Interview Questions

### Q1: What's the output?

```javascript
Promise.resolve('1')
  .then(res => {
    console.log(res);
    return '2';
  })
  .then(res => {
    console.log(res);
  });
console.log('3');
```

**Answer:** `3, 1, 2`

### Q2: What's the output?

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
async1();
console.log('script end');
```

**Answer:** `script start, async1 start, async2, script end, async1 end`

### Q3: Implement Promise.all

```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (!promises.length) {
      return resolve([]);
    }

    const results = [];
    let completed = 0;

    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(value => {
          results[index] = value;
          completed++;
          if (completed === promises.length) {
            resolve(results);
          }
        })
        .catch(reject);
    });
  });
}
```

### Q4: Convert callback to promise

```javascript
function readFilePromise(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, 'utf8', (err, data) => {
      if (err) reject(err);
      else resolve(data);
    });
  });
}

// Generic promisify
function promisify(fn) {
  return function(...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (err, result) => {
        if (err) reject(err);
        else resolve(result);
      });
    });
  };
}
```

---

## Key Takeaways

1. Promises represent eventual completion/failure of async operations
2. **States:** pending → fulfilled/rejected (immutable once settled)
3. **Chaining:** `.then()` returns a new promise
4. **async/await** is syntactic sugar over promises
5. Use `Promise.all` for parallel, loop with `await` for sequential
6. Always handle errors with `.catch()` or `try/catch`
7. `Promise.allSettled` when you need all results regardless of errors
