# Functional Programming in JavaScript

## Overview

Functional programming (FP) is a programming paradigm that treats computation as the evaluation of mathematical functions. JavaScript supports FP concepts, and understanding them is crucial for modern JavaScript development.

---

## Core Principles

### 1. Pure Functions

A pure function:
- Given the same inputs, always returns the same output
- Has no side effects (doesn't modify external state)

```javascript
// Pure function
function add(a, b) {
  return a + b;
}

// Impure - depends on external state
let tax = 0.1;
function calculateTotal(price) {
  return price + price * tax;  // Depends on external 'tax'
}

// Impure - has side effects
function addToCart(cart, item) {
  cart.push(item);  // Modifies input
  return cart;
}

// Pure version
function addToCart(cart, item) {
  return [...cart, item];  // Returns new array
}

// Impure - side effects
let count = 0;
function increment() {
  count++;  // Modifies external state
  return count;
}

// Pure version
function increment(count) {
  return count + 1;
}
```

### 2. Immutability

Never modify data; create new copies instead.

```javascript
// Mutable (bad)
const person = { name: 'John', age: 30 };
person.age = 31;  // Mutates original

// Immutable (good)
const updatedPerson = { ...person, age: 31 };  // New object

// Array mutations (bad)
const arr = [1, 2, 3];
arr.push(4);      // Mutates
arr.splice(0, 1); // Mutates

// Immutable array operations (good)
const arr = [1, 2, 3];
const added = [...arr, 4];           // [1, 2, 3, 4]
const removed = arr.slice(1);        // [2, 3]
const updated = arr.map((x, i) => i === 0 ? 10 : x);  // [10, 2, 3]

// Deep immutability
const user = {
  name: 'John',
  address: { city: 'NYC', zip: '10001' }
};

// Wrong - still mutates nested object
const user2 = { ...user };
user2.address.city = 'LA';  // Also changes user.address.city!

// Correct - deep clone nested objects
const user2 = {
  ...user,
  address: { ...user.address, city: 'LA' }
};

// Using structuredClone (modern browsers/Node 17+)
const deepCopy = structuredClone(user);
```

### 3. First-Class Functions

Functions are values that can be:
- Assigned to variables
- Passed as arguments
- Returned from other functions

```javascript
// Assign to variable
const greet = function(name) {
  return `Hello, ${name}`;
};

// Pass as argument
function executeOperation(operation, a, b) {
  return operation(a, b);
}
executeOperation((x, y) => x + y, 5, 3);  // 8

// Return from function
function createMultiplier(factor) {
  return function(number) {
    return number * factor;
  };
}
const double = createMultiplier(2);
double(5);  // 10
```

---

## Higher-Order Functions

Functions that take functions as arguments or return functions.

### map

Transform each element.

```javascript
const numbers = [1, 2, 3, 4, 5];

// Transform to doubled values
const doubled = numbers.map(x => x * 2);  // [2, 4, 6, 8, 10]

// Extract properties
const users = [
  { name: 'John', age: 30 },
  { name: 'Jane', age: 25 }
];
const names = users.map(user => user.name);  // ['John', 'Jane']

// With index
const indexed = numbers.map((num, i) => `${i}: ${num}`);

// Implement map
function map(arr, fn) {
  const result = [];
  for (let i = 0; i < arr.length; i++) {
    result.push(fn(arr[i], i, arr));
  }
  return result;
}
```

### filter

Select elements that match a condition.

```javascript
const numbers = [1, 2, 3, 4, 5, 6];

// Filter even numbers
const evens = numbers.filter(x => x % 2 === 0);  // [2, 4, 6]

// Filter by property
const adults = users.filter(user => user.age >= 18);

// Implement filter
function filter(arr, predicate) {
  const result = [];
  for (let i = 0; i < arr.length; i++) {
    if (predicate(arr[i], i, arr)) {
      result.push(arr[i]);
    }
  }
  return result;
}
```

### reduce

Accumulate values into a single result.

```javascript
const numbers = [1, 2, 3, 4, 5];

// Sum
const sum = numbers.reduce((acc, curr) => acc + curr, 0);  // 15

// Max
const max = numbers.reduce((acc, curr) => Math.max(acc, curr), -Infinity);

// Count occurrences
const items = ['a', 'b', 'a', 'c', 'a', 'b'];
const counts = items.reduce((acc, item) => {
  acc[item] = (acc[item] || 0) + 1;
  return acc;
}, {});
// { a: 3, b: 2, c: 1 }

// Group by property
const people = [
  { name: 'John', department: 'Sales' },
  { name: 'Jane', department: 'Engineering' },
  { name: 'Bob', department: 'Sales' }
];

const byDepartment = people.reduce((acc, person) => {
  const dept = person.department;
  acc[dept] = acc[dept] || [];
  acc[dept].push(person);
  return acc;
}, {});

// Flatten array
const nested = [[1, 2], [3, 4], [5]];
const flat = nested.reduce((acc, arr) => [...acc, ...arr], []);
// [1, 2, 3, 4, 5]

// Pipe functions
const pipe = (...fns) => (value) =>
  fns.reduce((acc, fn) => fn(acc), value);

const addOne = x => x + 1;
const double = x => x * 2;
const pipeline = pipe(addOne, double, addOne);
pipeline(5);  // ((5 + 1) * 2) + 1 = 13

// Implement reduce
function reduce(arr, fn, initial) {
  let acc = initial;
  let startIndex = 0;

  if (initial === undefined) {
    acc = arr[0];
    startIndex = 1;
  }

  for (let i = startIndex; i < arr.length; i++) {
    acc = fn(acc, arr[i], i, arr);
  }

  return acc;
}
```

### find and findIndex

```javascript
const users = [
  { id: 1, name: 'John' },
  { id: 2, name: 'Jane' },
  { id: 3, name: 'Bob' }
];

// Find first match
const user = users.find(u => u.id === 2);  // { id: 2, name: 'Jane' }

// Find index of first match
const index = users.findIndex(u => u.id === 2);  // 1
```

### some and every

```javascript
const numbers = [1, 2, 3, 4, 5];

// At least one matches
numbers.some(x => x > 4);   // true
numbers.some(x => x > 10);  // false

// All match
numbers.every(x => x > 0);  // true
numbers.every(x => x > 3);  // false
```

---

## Function Composition

Combining simple functions to build complex ones.

### compose (right to left)

```javascript
const compose = (...fns) => (value) =>
  fns.reduceRight((acc, fn) => fn(acc), value);

const addOne = x => x + 1;
const double = x => x * 2;
const square = x => x * x;

const composed = compose(square, double, addOne);
composed(2);  // square(double(addOne(2))) = square(double(3)) = square(6) = 36
```

### pipe (left to right)

```javascript
const pipe = (...fns) => (value) =>
  fns.reduce((acc, fn) => fn(acc), value);

const piped = pipe(addOne, double, square);
piped(2);  // square(double(addOne(2))) = square(double(3)) = square(6) = 36

// Same result, but reads left-to-right
```

### Practical Example

```javascript
// Data processing pipeline
const users = [
  { name: 'John', age: 30, active: true },
  { name: 'Jane', age: 25, active: false },
  { name: 'Bob', age: 35, active: true }
];

const getActiveUsers = users => users.filter(u => u.active);
const getNames = users => users.map(u => u.name);
const sortNames = names => [...names].sort();
const formatList = names => names.join(', ');

const getActiveUserList = pipe(
  getActiveUsers,
  getNames,
  sortNames,
  formatList
);

getActiveUserList(users);  // "Bob, John"
```

---

## Currying

Converting a function with multiple arguments into a sequence of functions with single arguments.

```javascript
// Regular function
const add = (a, b, c) => a + b + c;
add(1, 2, 3);  // 6

// Curried version
const curriedAdd = a => b => c => a + b + c;
curriedAdd(1)(2)(3);  // 6

// Partial application
const add1 = curriedAdd(1);
const add1and2 = add1(2);
add1and2(3);  // 6

// Generic curry function
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

const curriedSum = curry((a, b, c) => a + b + c);
curriedSum(1)(2)(3);     // 6
curriedSum(1, 2)(3);     // 6
curriedSum(1)(2, 3);     // 6
curriedSum(1, 2, 3);     // 6

// Practical example: Event handler factory
const handleEvent = curry((eventType, element, callback) => {
  element.addEventListener(eventType, callback);
});

const onClick = handleEvent('click');
const onClickButton = onClick(document.getElementById('btn'));
onClickButton(() => console.log('Clicked!'));

// Configuration
const fetchWithBase = curry((baseUrl, endpoint, options) =>
  fetch(`${baseUrl}${endpoint}`, options)
);

const apiClient = fetchWithBase('https://api.example.com');
const getUsers = apiClient('/users');
const postUser = apiClient('/users', { method: 'POST' });
```

---

## Partial Application

Fixing some arguments of a function.

```javascript
// Using bind
function greet(greeting, name) {
  return `${greeting}, ${name}!`;
}

const sayHello = greet.bind(null, 'Hello');
sayHello('John');  // "Hello, John!"

// Generic partial function
function partial(fn, ...presetArgs) {
  return function(...laterArgs) {
    return fn(...presetArgs, ...laterArgs);
  };
}

const multiply = (a, b, c) => a * b * c;
const multiplyBy2 = partial(multiply, 2);
multiplyBy2(3, 4);  // 24

// Partial with placeholders
const _ = Symbol('placeholder');

function partialWithPlaceholders(fn, ...presetArgs) {
  return function(...laterArgs) {
    const args = presetArgs.map(arg =>
      arg === _ ? laterArgs.shift() : arg
    );
    return fn(...args, ...laterArgs);
  };
}

const divide = (a, b) => a / b;
const divideBy = partialWithPlaceholders(divide, _, 2);
divideBy(10);  // 5
```

---

## Common Functional Utilities

### debounce

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
  console.log('Searching:', query);
}, 300);
```

### throttle

```javascript
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

const throttledScroll = throttle(() => {
  console.log('Scroll event');
}, 100);
```

### memoize

```javascript
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

const expensiveOperation = memoize((n) => {
  console.log('Computing...');
  return n * 2;
});

expensiveOperation(5);  // Computing... 10
expensiveOperation(5);  // 10 (cached)
```

### once

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

const initialize = once(() => {
  console.log('Initializing...');
  return 'initialized';
});

initialize();  // Initializing... 'initialized'
initialize();  // 'initialized' (no log)
```

### negate

```javascript
function negate(predicate) {
  return function(...args) {
    return !predicate.apply(this, args);
  };
}

const isEven = n => n % 2 === 0;
const isOdd = negate(isEven);

isOdd(3);  // true
isOdd(4);  // false
```

### flip

```javascript
function flip(fn) {
  return function(...args) {
    return fn.apply(this, args.reverse());
  };
}

const divide = (a, b) => a / b;
const flippedDivide = flip(divide);

divide(10, 2);        // 5
flippedDivide(10, 2); // 0.2 (2 / 10)
```

---

## Recursion in FP

### Factorial

```javascript
// Recursive
const factorial = n =>
  n <= 1 ? 1 : n * factorial(n - 1);

// Tail recursive (can be optimized)
const factorialTail = (n, acc = 1) =>
  n <= 1 ? acc : factorialTail(n - 1, n * acc);
```

### Deep operations

```javascript
// Deep clone
const deepClone = obj => {
  if (obj === null || typeof obj !== 'object') {
    return obj;
  }

  if (Array.isArray(obj)) {
    return obj.map(deepClone);
  }

  return Object.fromEntries(
    Object.entries(obj).map(([k, v]) => [k, deepClone(v)])
  );
};

// Deep freeze
const deepFreeze = obj => {
  Object.freeze(obj);

  Object.values(obj).forEach(value => {
    if (typeof value === 'object' && value !== null) {
      deepFreeze(value);
    }
  });

  return obj;
};

// Flatten nested arrays
const flatten = arr =>
  arr.reduce((acc, val) =>
    Array.isArray(val) ? [...acc, ...flatten(val)] : [...acc, val],
    []
  );

flatten([1, [2, [3, [4]]]]); // [1, 2, 3, 4]
```

---

## Functors and Monads (Simplified)

### Functor (Mappable container)

```javascript
// Array is a functor
[1, 2, 3].map(x => x * 2);

// Custom functor
class Box {
  constructor(value) {
    this.value = value;
  }

  map(fn) {
    return new Box(fn(this.value));
  }

  fold(fn) {
    return fn(this.value);
  }
}

const box = new Box(5)
  .map(x => x + 1)
  .map(x => x * 2)
  .fold(x => x);  // 12
```

### Maybe (handles null)

```javascript
class Maybe {
  constructor(value) {
    this.value = value;
  }

  static of(value) {
    return new Maybe(value);
  }

  isNothing() {
    return this.value === null || this.value === undefined;
  }

  map(fn) {
    return this.isNothing()
      ? Maybe.of(null)
      : Maybe.of(fn(this.value));
  }

  fold(onNothing, onJust) {
    return this.isNothing()
      ? onNothing()
      : onJust(this.value);
  }
}

// Safe property access
const getNestedValue = obj =>
  Maybe.of(obj)
    .map(o => o.user)
    .map(u => u.address)
    .map(a => a.city)
    .fold(
      () => 'Unknown',
      city => city
    );

getNestedValue({ user: { address: { city: 'NYC' } } });  // 'NYC'
getNestedValue({ user: {} });  // 'Unknown'
getNestedValue(null);  // 'Unknown'
```

---

## Interview Questions

### Q1: Implement map using reduce

```javascript
function mapWithReduce(arr, fn) {
  return arr.reduce((acc, item, index) => {
    acc.push(fn(item, index, arr));
    return acc;
  }, []);
}
```

### Q2: Implement filter using reduce

```javascript
function filterWithReduce(arr, predicate) {
  return arr.reduce((acc, item, index) => {
    if (predicate(item, index, arr)) {
      acc.push(item);
    }
    return acc;
  }, []);
}
```

### Q3: Implement compose

```javascript
const compose = (...fns) => (value) =>
  fns.reduceRight((acc, fn) => fn(acc), value);
```

### Q4: Implement curry

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return (...more) => curried.apply(this, [...args, ...more]);
  };
}
```

### Q5: What makes a function pure?

```javascript
// 1. Same input always produces same output
// 2. No side effects (doesn't modify external state)
// 3. Doesn't depend on external mutable state
```

---

## Key Takeaways

1. **Pure functions** are predictable and testable
2. **Immutability** prevents bugs from shared state
3. **Higher-order functions** (map, filter, reduce) are core FP tools
4. **Composition** builds complex logic from simple functions
5. **Currying** enables partial application and reusable functions
6. **Memoization** optimizes expensive computations
7. FP leads to more **declarative, readable** code
