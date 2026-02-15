# Closures

## Overview

Closures are one of the most important and frequently asked concepts in JavaScript interviews. A closure is the combination of a function and the lexical environment within which that function was declared.

---

## What is a Closure?

```javascript
function outer() {
  const message = "Hello";  // Variable in outer scope

  function inner() {
    console.log(message);   // Accesses outer's variable
  }

  return inner;
}

const myFunc = outer();
myFunc();  // "Hello" - still has access to message!
```

**Definition:** A closure gives you access to an outer function's scope from an inner function, even after the outer function has returned.

---

## How Closures Work

### Lexical Environment

Every function maintains a reference to its lexical environment:

```javascript
function createCounter() {
  let count = 0;  // Enclosed variable

  return {
    increment: function() {
      count++;
      return count;
    },
    decrement: function() {
      count--;
      return count;
    },
    getCount: function() {
      return count;
    }
  };
}

const counter = createCounter();
console.log(counter.increment());  // 1
console.log(counter.increment());  // 2
console.log(counter.decrement());  // 1
console.log(counter.getCount());   // 1

// count is private - cannot access directly
console.log(counter.count);  // undefined
```

### Multiple Closures

Each invocation creates a new closure with its own environment:

```javascript
function createMultiplier(multiplier) {
  return function(number) {
    return number * multiplier;
  };
}

const double = createMultiplier(2);
const triple = createMultiplier(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15
// Each has its own multiplier value
```

---

## Common Use Cases

### 1. Data Privacy / Encapsulation

```javascript
function createBankAccount(initialBalance) {
  let balance = initialBalance;  // Private variable

  return {
    deposit(amount) {
      if (amount > 0) {
        balance += amount;
        return `Deposited ${amount}. New balance: ${balance}`;
      }
      return "Invalid amount";
    },

    withdraw(amount) {
      if (amount > 0 && amount <= balance) {
        balance -= amount;
        return `Withdrew ${amount}. New balance: ${balance}`;
      }
      return "Invalid amount or insufficient funds";
    },

    getBalance() {
      return balance;
    }
  };
}

const account = createBankAccount(100);
console.log(account.deposit(50));     // "Deposited 50. New balance: 150"
console.log(account.withdraw(30));    // "Withdrew 30. New balance: 120"
console.log(account.balance);         // undefined - can't access directly
```

### 2. Function Factories

```javascript
function createGreeter(greeting) {
  return function(name) {
    return `${greeting}, ${name}!`;
  };
}

const sayHello = createGreeter("Hello");
const sayHi = createGreeter("Hi");

console.log(sayHello("John"));  // "Hello, John!"
console.log(sayHi("Jane"));     // "Hi, Jane!"
```

### 3. Memoization

```javascript
function memoize(fn) {
  const cache = {};  // Closure over cache

  return function(...args) {
    const key = JSON.stringify(args);

    if (key in cache) {
      console.log("From cache");
      return cache[key];
    }

    console.log("Computing");
    const result = fn.apply(this, args);
    cache[key] = result;
    return result;
  };
}

const slowFib = (n) => {
  if (n <= 1) return n;
  return slowFib(n - 1) + slowFib(n - 2);
};

const fastFib = memoize(slowFib);
console.log(fastFib(10));  // Computing... 55
console.log(fastFib(10));  // From cache... 55
```

### 4. Event Handlers

```javascript
function setupButton(buttonId, message) {
  const button = document.getElementById(buttonId);

  button.addEventListener('click', function() {
    // Closure over message
    alert(message);
  });
}

setupButton('btn1', 'Button 1 clicked');
setupButton('btn2', 'Button 2 clicked');
```

### 5. Module Pattern

```javascript
const Calculator = (function() {
  // Private variables and functions
  let result = 0;

  function validate(num) {
    return typeof num === 'number' && !isNaN(num);
  }

  // Public API (returned object)
  return {
    add(num) {
      if (validate(num)) result += num;
      return this;
    },
    subtract(num) {
      if (validate(num)) result -= num;
      return this;
    },
    multiply(num) {
      if (validate(num)) result *= num;
      return this;
    },
    getResult() {
      return result;
    },
    reset() {
      result = 0;
      return this;
    }
  };
})();

Calculator.add(10).subtract(2).multiply(3);
console.log(Calculator.getResult());  // 24
```

### 6. Partial Application / Currying

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return function(...moreArgs) {
      return curried.apply(this, args.concat(moreArgs));
    };
  };
}

function add(a, b, c) {
  return a + b + c;
}

const curriedAdd = curry(add);
console.log(curriedAdd(1)(2)(3));     // 6
console.log(curriedAdd(1, 2)(3));     // 6
console.log(curriedAdd(1)(2, 3));     // 6
```

### 7. setTimeout in Loops

```javascript
// Problem: All print 3
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}

// Solution 1: Use let (block scope)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}

// Solution 2: IIFE creating closure
for (var i = 0; i < 3; i++) {
  ((j) => {
    setTimeout(() => console.log(j), 100);
  })(i);
}

// Solution 3: Function factory
for (var i = 0; i < 3; i++) {
  setTimeout(createLogger(i), 100);
}

function createLogger(num) {
  return function() {
    console.log(num);
  };
}
```

---

## Common Interview Questions

### Q1: What will be logged?

```javascript
function createFunctions() {
  var result = [];

  for (var i = 0; i < 3; i++) {
    result.push(function() {
      return i;
    });
  }

  return result;
}

var funcs = createFunctions();
console.log(funcs[0]());
console.log(funcs[1]());
console.log(funcs[2]());
```

**Answer:** `3, 3, 3`

All functions share the same `i` (var is function-scoped), and `i` is 3 after the loop.

**Fix:**
```javascript
function createFunctions() {
  var result = [];

  for (let i = 0; i < 3; i++) {  // Use let
    result.push(function() {
      return i;
    });
  }

  return result;
}
```

### Q2: Implement a function that can only be called once

```javascript
function once(fn) {
  let called = false;
  let result;

  return function(...args) {
    if (!called) {
      called = true;
      result = fn.apply(this, args);
    }
    return result;
  };
}

const initOnce = once(() => {
  console.log("Initializing...");
  return "initialized";
});

console.log(initOnce());  // "Initializing..." then "initialized"
console.log(initOnce());  // "initialized" (no log, returns cached result)
```

### Q3: Implement debounce

```javascript
function debounce(fn, delay) {
  let timeoutId;

  return function(...args) {
    clearTimeout(timeoutId);

    timeoutId = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}

const debouncedSearch = debounce((query) => {
  console.log("Searching for:", query);
}, 300);

// Only last call executes after 300ms of inactivity
debouncedSearch("a");
debouncedSearch("ab");
debouncedSearch("abc");  // Only this executes
```

### Q4: Implement throttle

```javascript
function throttle(fn, limit) {
  let inThrottle = false;

  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;

      setTimeout(() => {
        inThrottle = false;
      }, limit);
    }
  };
}

const throttledScroll = throttle(() => {
  console.log("Scroll event");
}, 100);
```

### Q5: Create a private counter

```javascript
function createCounter() {
  let count = 0;

  return {
    increment() { return ++count; },
    decrement() { return --count; },
    reset() { count = 0; return count; },
    getCount() { return count; }
  };
}

const counter = createCounter();
console.log(counter.increment());  // 1
console.log(counter.increment());  // 2
console.log(counter.getCount());   // 2
counter.count = 100;  // Does nothing to actual count
console.log(counter.getCount());   // 2
```

### Q6: What will be logged?

```javascript
var name = "Global";

const obj = {
  name: "Object",
  getName: function() {
    return function() {
      return this.name;
    };
  }
};

console.log(obj.getName()());
```

**Answer:** `"Global"` (or `undefined` in strict mode)

The inner function loses `this` context. The returned function is called without object context.

**Fix with closure:**
```javascript
const obj = {
  name: "Object",
  getName: function() {
    const self = this;  // Capture this in closure
    return function() {
      return self.name;
    };
  }
};

// Or use arrow function
const obj2 = {
  name: "Object",
  getName: function() {
    return () => this.name;  // Arrow inherits this
  }
};
```

---

## Memory Considerations

### Memory Leaks with Closures

```javascript
// Potential memory leak
function createHandler() {
  const largeData = new Array(1000000).fill('x');

  return function() {
    // Even if we don't use largeData, it's retained
    console.log("Handler called");
  };
}

const handler = createHandler();
// largeData is retained in memory as long as handler exists

// Better: Only capture what you need
function createHandlerBetter() {
  const largeData = new Array(1000000).fill('x');
  const neededValue = largeData.length;

  return function() {
    console.log("Length was:", neededValue);
  };
  // largeData can be garbage collected
}
```

### Cleaning Up Event Listeners

```javascript
function setupTracking() {
  const data = { /* large object */ };

  const handler = function() {
    console.log(data);
  };

  document.addEventListener('click', handler);

  // Return cleanup function
  return function cleanup() {
    document.removeEventListener('click', handler);
  };
}

const cleanup = setupTracking();
// Later...
cleanup();  // Removes handler, allows garbage collection
```

---

## Key Takeaways

1. **Closures** preserve access to outer scope variables
2. Each closure has its **own environment** (separate instances)
3. Common uses: **privacy**, **state**, **callbacks**, **factories**
4. Be aware of **memory implications** - closures retain references
5. Understanding closures is key to understanding **JavaScript patterns**
6. **Arrow functions** inherit `this` from enclosing scope (lexical `this`)

---

## Closure Mental Model

```
┌─────────────────────────────────────┐
│         Global Scope                │
│   ┌─────────────────────────────┐   │
│   │     outer() Scope           │   │
│   │  ┌───────────────────────┐  │   │
│   │  │   inner() Scope       │  │   │
│   │  │                       │  │   │
│   │  │  Can access:          │  │   │
│   │  │  - own variables      │  │   │
│   │  │  - outer's variables ◄┼──┼── Closure!
│   │  │  - global variables   │  │   │
│   │  └───────────────────────┘  │   │
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘
```
