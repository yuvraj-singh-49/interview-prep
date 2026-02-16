# JavaScript Polyfills

## Overview

Polyfills are code implementations that provide functionality not natively supported by older browsers. Understanding how to implement polyfills demonstrates deep understanding of JavaScript fundamentals.

---

## Function Methods

### Function.prototype.bind

```javascript
// bind creates a new function with bound context and partial arguments

Function.prototype.myBind = function(context, ...boundArgs) {
  const fn = this;

  return function(...callArgs) {
    return fn.apply(context, [...boundArgs, ...callArgs]);
  };
};

// With support for new keyword (constructor)
Function.prototype.myBind = function(context, ...boundArgs) {
  const fn = this;

  const bound = function(...callArgs) {
    // If called with new, use this instead of context
    const ctx = this instanceof bound ? this : context;
    return fn.apply(ctx, [...boundArgs, ...callArgs]);
  };

  // Maintain prototype chain for new
  bound.prototype = Object.create(fn.prototype);

  return bound;
};

// Test
const obj = { x: 10 };
function add(a, b) {
  return this.x + a + b;
}

const boundAdd = add.myBind(obj, 5);
console.log(boundAdd(3));  // 18
```

### Function.prototype.call

```javascript
// call invokes function with specified context and arguments

Function.prototype.myCall = function(context, ...args) {
  // Handle null/undefined context
  context = context ?? globalThis;

  // Convert primitives to objects
  if (typeof context !== 'object') {
    context = Object(context);
  }

  // Create unique property to avoid collisions
  const key = Symbol('fn');
  context[key] = this;

  // Call function and get result
  const result = context[key](...args);

  // Clean up
  delete context[key];

  return result;
};

// Test
function greet(greeting) {
  return `${greeting}, ${this.name}!`;
}

const person = { name: 'John' };
console.log(greet.myCall(person, 'Hello'));  // "Hello, John!"
```

### Function.prototype.apply

```javascript
// apply is like call but takes arguments as array

Function.prototype.myApply = function(context, args = []) {
  context = context ?? globalThis;

  if (typeof context !== 'object') {
    context = Object(context);
  }

  const key = Symbol('fn');
  context[key] = this;

  const result = context[key](...args);

  delete context[key];

  return result;
};

// Test
function sum(...nums) {
  return nums.reduce((a, b) => a + this.base + b, 0);
}

const obj = { base: 10 };
console.log(sum.myApply(obj, [1, 2, 3]));  // 36
```

---

## Array Methods

### Array.prototype.map

```javascript
Array.prototype.myMap = function(callback, thisArg) {
  if (typeof callback !== 'function') {
    throw new TypeError(callback + ' is not a function');
  }

  const result = [];
  for (let i = 0; i < this.length; i++) {
    // Check for sparse arrays
    if (i in this) {
      result[i] = callback.call(thisArg, this[i], i, this);
    }
  }
  return result;
};

// Test
const doubled = [1, 2, 3].myMap(x => x * 2);
console.log(doubled);  // [2, 4, 6]
```

### Array.prototype.filter

```javascript
Array.prototype.myFilter = function(callback, thisArg) {
  if (typeof callback !== 'function') {
    throw new TypeError(callback + ' is not a function');
  }

  const result = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this) {
      if (callback.call(thisArg, this[i], i, this)) {
        result.push(this[i]);
      }
    }
  }
  return result;
};

// Test
const evens = [1, 2, 3, 4, 5].myFilter(x => x % 2 === 0);
console.log(evens);  // [2, 4]
```

### Array.prototype.reduce

```javascript
Array.prototype.myReduce = function(callback, initialValue) {
  if (typeof callback !== 'function') {
    throw new TypeError(callback + ' is not a function');
  }

  const len = this.length;
  let accumulator;
  let startIndex;

  // Handle initial value
  if (arguments.length >= 2) {
    accumulator = initialValue;
    startIndex = 0;
  } else {
    // Find first element (handling sparse arrays)
    let found = false;
    for (let i = 0; i < len; i++) {
      if (i in this) {
        accumulator = this[i];
        startIndex = i + 1;
        found = true;
        break;
      }
    }
    if (!found) {
      throw new TypeError('Reduce of empty array with no initial value');
    }
  }

  for (let i = startIndex; i < len; i++) {
    if (i in this) {
      accumulator = callback(accumulator, this[i], i, this);
    }
  }

  return accumulator;
};

// Test
const sum = [1, 2, 3, 4].myReduce((acc, val) => acc + val, 0);
console.log(sum);  // 10
```

### Array.prototype.forEach

```javascript
Array.prototype.myForEach = function(callback, thisArg) {
  if (typeof callback !== 'function') {
    throw new TypeError(callback + ' is not a function');
  }

  for (let i = 0; i < this.length; i++) {
    if (i in this) {
      callback.call(thisArg, this[i], i, this);
    }
  }
};

// Test
[1, 2, 3].myForEach((x, i) => console.log(i, x));
```

### Array.prototype.find / findIndex

```javascript
Array.prototype.myFind = function(callback, thisArg) {
  if (typeof callback !== 'function') {
    throw new TypeError(callback + ' is not a function');
  }

  for (let i = 0; i < this.length; i++) {
    if (i in this) {
      if (callback.call(thisArg, this[i], i, this)) {
        return this[i];
      }
    }
  }
  return undefined;
};

Array.prototype.myFindIndex = function(callback, thisArg) {
  if (typeof callback !== 'function') {
    throw new TypeError(callback + ' is not a function');
  }

  for (let i = 0; i < this.length; i++) {
    if (i in this) {
      if (callback.call(thisArg, this[i], i, this)) {
        return i;
      }
    }
  }
  return -1;
};

// Test
const users = [{ id: 1, name: 'John' }, { id: 2, name: 'Jane' }];
console.log(users.myFind(u => u.id === 2));  // { id: 2, name: 'Jane' }
console.log(users.myFindIndex(u => u.id === 2));  // 1
```

### Array.prototype.some / every

```javascript
Array.prototype.mySome = function(callback, thisArg) {
  if (typeof callback !== 'function') {
    throw new TypeError(callback + ' is not a function');
  }

  for (let i = 0; i < this.length; i++) {
    if (i in this) {
      if (callback.call(thisArg, this[i], i, this)) {
        return true;
      }
    }
  }
  return false;
};

Array.prototype.myEvery = function(callback, thisArg) {
  if (typeof callback !== 'function') {
    throw new TypeError(callback + ' is not a function');
  }

  for (let i = 0; i < this.length; i++) {
    if (i in this) {
      if (!callback.call(thisArg, this[i], i, this)) {
        return false;
      }
    }
  }
  return true;
};

// Test
console.log([1, 2, 3].mySome(x => x > 2));   // true
console.log([1, 2, 3].myEvery(x => x > 0));  // true
```

### Array.prototype.flat

```javascript
Array.prototype.myFlat = function(depth = 1) {
  const result = [];

  const flatten = (arr, d) => {
    for (const item of arr) {
      if (Array.isArray(item) && d > 0) {
        flatten(item, d - 1);
      } else {
        result.push(item);
      }
    }
  };

  flatten(this, depth);
  return result;
};

// Non-recursive version
Array.prototype.myFlat = function(depth = 1) {
  let result = [...this];

  for (let i = 0; i < depth; i++) {
    const temp = [];
    let hasNested = false;

    for (const item of result) {
      if (Array.isArray(item)) {
        temp.push(...item);
        hasNested = true;
      } else {
        temp.push(item);
      }
    }

    result = temp;
    if (!hasNested) break;
  }

  return result;
};

// Test
console.log([1, [2, [3, [4]]]].myFlat(2));  // [1, 2, 3, [4]]
```

### Array.prototype.flatMap

```javascript
Array.prototype.myFlatMap = function(callback, thisArg) {
  return this.myMap(callback, thisArg).myFlat(1);
};

// Test
const arr = [1, 2, 3];
console.log(arr.myFlatMap(x => [x, x * 2]));  // [1, 2, 2, 4, 3, 6]
```

---

## Promise Polyfills

### Promise.all

```javascript
Promise.myAll = function(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let completed = 0;
    const total = promises.length;

    if (total === 0) {
      resolve([]);
      return;
    }

    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(value => {
          results[index] = value;
          completed++;

          if (completed === total) {
            resolve(results);
          }
        })
        .catch(reject);
    });
  });
};

// Test
Promise.myAll([
  Promise.resolve(1),
  Promise.resolve(2),
  Promise.resolve(3)
]).then(console.log);  // [1, 2, 3]
```

### Promise.race

```javascript
Promise.myRace = function(promises) {
  return new Promise((resolve, reject) => {
    for (const promise of promises) {
      Promise.resolve(promise)
        .then(resolve)
        .catch(reject);
    }
  });
};

// Test
Promise.myRace([
  new Promise(r => setTimeout(() => r('slow'), 100)),
  new Promise(r => setTimeout(() => r('fast'), 50))
]).then(console.log);  // 'fast'
```

### Promise.allSettled

```javascript
Promise.myAllSettled = function(promises) {
  return new Promise((resolve) => {
    const results = [];
    let completed = 0;
    const total = promises.length;

    if (total === 0) {
      resolve([]);
      return;
    }

    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(value => {
          results[index] = { status: 'fulfilled', value };
        })
        .catch(reason => {
          results[index] = { status: 'rejected', reason };
        })
        .finally(() => {
          completed++;
          if (completed === total) {
            resolve(results);
          }
        });
    });
  });
};

// Test
Promise.myAllSettled([
  Promise.resolve(1),
  Promise.reject('error'),
  Promise.resolve(3)
]).then(console.log);
// [{ status: 'fulfilled', value: 1 }, { status: 'rejected', reason: 'error' }, { status: 'fulfilled', value: 3 }]
```

### Promise.any

```javascript
Promise.myAny = function(promises) {
  return new Promise((resolve, reject) => {
    const errors = [];
    let rejected = 0;
    const total = promises.length;

    if (total === 0) {
      reject(new AggregateError([], 'All promises were rejected'));
      return;
    }

    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(resolve)
        .catch(error => {
          errors[index] = error;
          rejected++;

          if (rejected === total) {
            reject(new AggregateError(errors, 'All promises were rejected'));
          }
        });
    });
  });
};

// Test
Promise.myAny([
  Promise.reject('error1'),
  Promise.resolve('success'),
  Promise.reject('error2')
]).then(console.log);  // 'success'
```

### Simple Promise Implementation

```javascript
class MyPromise {
  constructor(executor) {
    this.state = 'pending';
    this.value = undefined;
    this.handlers = [];

    const resolve = (value) => {
      if (this.state !== 'pending') return;

      if (value instanceof MyPromise) {
        value.then(resolve, reject);
        return;
      }

      this.state = 'fulfilled';
      this.value = value;
      this.handlers.forEach(h => h.onFulfilled(value));
    };

    const reject = (reason) => {
      if (this.state !== 'pending') return;

      this.state = 'rejected';
      this.value = reason;
      this.handlers.forEach(h => h.onRejected(reason));
    };

    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const handle = (handler, fallback) => (value) => {
        try {
          if (typeof handler === 'function') {
            resolve(handler(value));
          } else {
            fallback(value);
          }
        } catch (error) {
          reject(error);
        }
      };

      const handler = {
        onFulfilled: handle(onFulfilled, resolve),
        onRejected: handle(onRejected, reject)
      };

      if (this.state === 'pending') {
        this.handlers.push(handler);
      } else if (this.state === 'fulfilled') {
        queueMicrotask(() => handler.onFulfilled(this.value));
      } else {
        queueMicrotask(() => handler.onRejected(this.value));
      }
    });
  }

  catch(onRejected) {
    return this.then(null, onRejected);
  }

  finally(onFinally) {
    return this.then(
      value => MyPromise.resolve(onFinally()).then(() => value),
      reason => MyPromise.resolve(onFinally()).then(() => { throw reason; })
    );
  }

  static resolve(value) {
    return new MyPromise(resolve => resolve(value));
  }

  static reject(reason) {
    return new MyPromise((_, reject) => reject(reason));
  }
}
```

---

## Object Methods

### Object.create

```javascript
Object.myCreate = function(proto, propertiesObject) {
  if (typeof proto !== 'object' && proto !== null) {
    throw new TypeError('Object prototype may only be an Object or null');
  }

  function F() {}
  F.prototype = proto;
  const obj = new F();

  if (propertiesObject !== undefined) {
    Object.defineProperties(obj, propertiesObject);
  }

  return obj;
};

// Test
const person = {
  greet() { return `Hello, ${this.name}`; }
};

const john = Object.myCreate(person);
john.name = 'John';
console.log(john.greet());  // "Hello, John"
```

### Object.assign

```javascript
Object.myAssign = function(target, ...sources) {
  if (target == null) {
    throw new TypeError('Cannot convert undefined or null to object');
  }

  const to = Object(target);

  for (const source of sources) {
    if (source != null) {
      for (const key of Object.keys(source)) {
        to[key] = source[key];
      }

      // Handle symbols
      for (const sym of Object.getOwnPropertySymbols(source)) {
        if (Object.prototype.propertyIsEnumerable.call(source, sym)) {
          to[sym] = source[sym];
        }
      }
    }
  }

  return to;
};

// Test
const target = { a: 1 };
const source = { b: 2, c: 3 };
console.log(Object.myAssign(target, source));  // { a: 1, b: 2, c: 3 }
```

### Object.keys / values / entries

```javascript
Object.myKeys = function(obj) {
  if (obj == null) {
    throw new TypeError('Cannot convert undefined or null to object');
  }

  const result = [];
  for (const key in obj) {
    if (Object.prototype.hasOwnProperty.call(obj, key)) {
      result.push(key);
    }
  }
  return result;
};

Object.myValues = function(obj) {
  return Object.myKeys(obj).map(key => obj[key]);
};

Object.myEntries = function(obj) {
  return Object.myKeys(obj).map(key => [key, obj[key]]);
};

// Test
const obj = { a: 1, b: 2 };
console.log(Object.myKeys(obj));    // ['a', 'b']
console.log(Object.myValues(obj));  // [1, 2]
console.log(Object.myEntries(obj)); // [['a', 1], ['b', 2]]
```

---

## Other Polyfills

### String.prototype.trim

```javascript
String.prototype.myTrim = function() {
  return this.replace(/^\s+|\s+$/g, '');
};

String.prototype.myTrimStart = function() {
  return this.replace(/^\s+/, '');
};

String.prototype.myTrimEnd = function() {
  return this.replace(/\s+$/, '');
};

// Test
console.log('  hello  '.myTrim());  // 'hello'
```

### Array.from

```javascript
Array.myFrom = function(arrayLike, mapFn, thisArg) {
  const result = [];
  const len = arrayLike.length;

  for (let i = 0; i < len; i++) {
    const value = arrayLike[i];
    result[i] = mapFn ? mapFn.call(thisArg, value, i) : value;
  }

  return result;
};

// Test
console.log(Array.myFrom('hello'));  // ['h', 'e', 'l', 'l', 'o']
console.log(Array.myFrom([1, 2, 3], x => x * 2));  // [2, 4, 6]
```

### Array.isArray

```javascript
Array.myIsArray = function(value) {
  return Object.prototype.toString.call(value) === '[object Array]';
};

// Test
console.log(Array.myIsArray([1, 2, 3]));  // true
console.log(Array.myIsArray('hello'));    // false
```

### instanceof

```javascript
function myInstanceof(obj, constructor) {
  if (obj == null || typeof obj !== 'object') {
    return false;
  }

  let proto = Object.getPrototypeOf(obj);

  while (proto !== null) {
    if (proto === constructor.prototype) {
      return true;
    }
    proto = Object.getPrototypeOf(proto);
  }

  return false;
}

// Test
class Animal {}
class Dog extends Animal {}

const dog = new Dog();
console.log(myInstanceof(dog, Dog));     // true
console.log(myInstanceof(dog, Animal));  // true
console.log(myInstanceof(dog, Array));   // false
```

### new Keyword

```javascript
function myNew(Constructor, ...args) {
  // Create object with constructor's prototype
  const obj = Object.create(Constructor.prototype);

  // Call constructor with new object as context
  const result = Constructor.apply(obj, args);

  // Return object or constructor result if it's an object
  return result instanceof Object ? result : obj;
}

// Test
function Person(name, age) {
  this.name = name;
  this.age = age;
}

const john = myNew(Person, 'John', 30);
console.log(john.name, john.age);  // 'John', 30
```

---

## Interview Questions

### Q1: Implement Array.prototype.reduce

```javascript
Array.prototype.myReduce = function(callback, initialValue) {
  let accumulator = arguments.length >= 2 ? initialValue : this[0];
  const startIndex = arguments.length >= 2 ? 0 : 1;

  for (let i = startIndex; i < this.length; i++) {
    accumulator = callback(accumulator, this[i], i, this);
  }

  return accumulator;
};
```

### Q2: Implement Function.prototype.bind

```javascript
Function.prototype.myBind = function(context, ...boundArgs) {
  const fn = this;
  return function(...args) {
    return fn.apply(context, [...boundArgs, ...args]);
  };
};
```

### Q3: Implement Promise.all

```javascript
Promise.myAll = function(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let count = 0;

    promises.forEach((p, i) => {
      Promise.resolve(p).then(value => {
        results[i] = value;
        if (++count === promises.length) resolve(results);
      }).catch(reject);
    });

    if (promises.length === 0) resolve([]);
  });
};
```

---

## Key Takeaways

1. **Understand the spec**: Know edge cases (null context, sparse arrays)
2. **Handle thisArg**: Most array methods accept optional this context
3. **Symbol properties**: Consider symbols in object operations
4. **Prototype chain**: Understand how prototypes work for instanceof
5. **Promise microtasks**: Callbacks should be async (queueMicrotask)
6. **Error handling**: Validate inputs, throw appropriate errors
7. **Sparse arrays**: Check `i in array` for sparse array support
