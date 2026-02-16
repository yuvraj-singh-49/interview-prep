# ES6+ Features

## Overview

Modern JavaScript (ES6/ES2015 and beyond) introduced significant features that are essential for Lead-level interviews. This covers the most important additions.

---

## Destructuring

### Array Destructuring

```javascript
const arr = [1, 2, 3, 4, 5];

// Basic
const [first, second] = arr;  // 1, 2

// Skip elements
const [, , third] = arr;  // 3

// Rest pattern
const [head, ...tail] = arr;  // 1, [2, 3, 4, 5]

// Default values
const [a, b, c = 10] = [1, 2];  // 1, 2, 10

// Swap variables
let x = 1, y = 2;
[x, y] = [y, x];  // x = 2, y = 1

// Nested
const nested = [1, [2, 3]];
const [one, [two, three]] = nested;  // 1, 2, 3

// Function return
function getCoords() {
  return [10, 20];
}
const [xCoord, yCoord] = getCoords();
```

### Object Destructuring

```javascript
const person = { name: 'John', age: 30, city: 'NYC' };

// Basic
const { name, age } = person;  // 'John', 30

// Rename
const { name: personName } = person;  // personName = 'John'

// Default values
const { country = 'USA' } = person;  // 'USA'

// Rename with default
const { country: nation = 'USA' } = person;  // nation = 'USA'

// Nested
const user = {
  info: { firstName: 'John', lastName: 'Doe' }
};
const { info: { firstName } } = user;  // 'John'

// Rest pattern
const { name: n, ...rest } = person;  // rest = { age: 30, city: 'NYC' }

// Function parameters
function greet({ name, age = 0 }) {
  return `${name} is ${age}`;
}
greet({ name: 'John', age: 30 });

// Computed property names
const key = 'name';
const { [key]: value } = person;  // value = 'John'
```

---

## Spread and Rest Operators

### Spread (...)

```javascript
// Array spread
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];
const combined = [...arr1, ...arr2];  // [1, 2, 3, 4, 5, 6]

// Copy array (shallow)
const copy = [...arr1];

// Convert iterable to array
const chars = [..."hello"];  // ['h', 'e', 'l', 'l', 'o']
const set = new Set([1, 2, 3]);
const arrFromSet = [...set];  // [1, 2, 3]

// Object spread
const obj1 = { a: 1, b: 2 };
const obj2 = { c: 3 };
const merged = { ...obj1, ...obj2 };  // { a: 1, b: 2, c: 3 }

// Override properties (last wins)
const updated = { ...obj1, b: 10 };  // { a: 1, b: 10 }

// Shallow copy
const copyObj = { ...obj1 };

// Function arguments
const numbers = [1, 2, 3];
Math.max(...numbers);  // 3
```

### Rest (...)

```javascript
// Function parameters
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3, 4);  // 10

// With other params (must be last)
function greet(greeting, ...names) {
  return names.map(n => `${greeting}, ${n}`);
}
greet('Hello', 'John', 'Jane');  // ['Hello, John', 'Hello, Jane']

// Destructuring (arrays)
const [first, ...rest] = [1, 2, 3, 4];  // first = 1, rest = [2, 3, 4]

// Destructuring (objects)
const { a, ...others } = { a: 1, b: 2, c: 3 };  // a = 1, others = { b: 2, c: 3 }
```

---

## Arrow Functions

```javascript
// Basic syntax
const add = (a, b) => a + b;

// Single parameter (parentheses optional)
const double = x => x * 2;

// No parameters
const getRandom = () => Math.random();

// Multi-line (needs braces and return)
const calculate = (a, b) => {
  const sum = a + b;
  return sum * 2;
};

// Return object literal (wrap in parentheses)
const createPerson = (name, age) => ({ name, age });

// Key differences from regular functions:
// 1. No own 'this' (inherits from enclosing scope)
// 2. No 'arguments' object
// 3. Cannot be used as constructors (no 'new')
// 4. No 'prototype' property

// this behavior
const obj = {
  name: 'Object',
  regular: function() {
    return this.name;  // 'Object'
  },
  arrow: () => {
    return this.name;  // undefined (inherits from global)
  },
  delayed: function() {
    setTimeout(() => {
      console.log(this.name);  // 'Object' (inherits from delayed)
    }, 100);
  }
};
```

---

## Template Literals

```javascript
// Basic interpolation
const name = 'John';
const greeting = `Hello, ${name}!`;

// Multi-line strings
const html = `
  <div>
    <h1>${title}</h1>
    <p>${content}</p>
  </div>
`;

// Expressions
const price = 10;
const quantity = 3;
const total = `Total: $${price * quantity}`;

// Nested templates
const items = ['a', 'b', 'c'];
const list = `
  <ul>
    ${items.map(item => `<li>${item}</li>`).join('')}
  </ul>
`;

// Tagged templates
function highlight(strings, ...values) {
  return strings.reduce((result, str, i) => {
    const value = values[i] !== undefined ? `<mark>${values[i]}</mark>` : '';
    return result + str + value;
  }, '');
}

const user = 'John';
const action = 'logged in';
highlight`User ${user} has ${action}`;
// "User <mark>John</mark> has <mark>logged in</mark>"

// Raw strings
String.raw`Line1\nLine2`;  // 'Line1\\nLine2' (escape preserved)
```

---

## Enhanced Object Literals

```javascript
const name = 'John';
const age = 30;

// Shorthand property names
const person = { name, age };  // Same as { name: name, age: age }

// Shorthand methods
const obj = {
  greet() {  // Same as greet: function() {}
    return 'Hello';
  },
  // Computed property names
  ['prop_' + 42]: 'value',
  [Symbol.iterator]() {
    // ...
  }
};

// Getter and setter
const account = {
  _balance: 0,
  get balance() {
    return this._balance;
  },
  set balance(value) {
    if (value < 0) throw new Error('Invalid balance');
    this._balance = value;
  }
};
```

---

## Classes

```javascript
class Person {
  // Private field (ES2022)
  #ssn;

  // Static property
  static species = 'Homo sapiens';

  // Static private field
  static #count = 0;

  constructor(name, age) {
    this.name = name;
    this.age = age;
    this.#ssn = Math.random().toString(36);
    Person.#count++;
  }

  // Instance method
  greet() {
    return `Hi, I'm ${this.name}`;
  }

  // Private method
  #getSSN() {
    return this.#ssn;
  }

  // Getter
  get info() {
    return `${this.name}, ${this.age}`;
  }

  // Setter
  set info(value) {
    [this.name, this.age] = value.split(', ');
  }

  // Static method
  static getCount() {
    return Person.#count;
  }

  // Static block (ES2022)
  static {
    console.log('Class initialized');
  }
}

// Inheritance
class Employee extends Person {
  constructor(name, age, role) {
    super(name, age);  // Must call before using 'this'
    this.role = role;
  }

  greet() {
    return `${super.greet()}, I'm a ${this.role}`;
  }
}

const emp = new Employee('John', 30, 'Developer');
emp.greet();  // "Hi, I'm John, I'm a Developer"
```

---

## Symbols

```javascript
// Create unique symbol
const sym1 = Symbol('description');
const sym2 = Symbol('description');
console.log(sym1 === sym2);  // false (always unique)

// Use as object key
const id = Symbol('id');
const user = {
  name: 'John',
  [id]: 12345
};

// Symbols are not enumerable
Object.keys(user);  // ['name']
Object.getOwnPropertySymbols(user);  // [Symbol(id)]

// Global symbol registry
const globalSym = Symbol.for('app.id');
const sameSym = Symbol.for('app.id');
console.log(globalSym === sameSym);  // true

Symbol.keyFor(globalSym);  // 'app.id'

// Well-known symbols
class Collection {
  constructor(items) {
    this.items = items;
  }

  // Make iterable
  [Symbol.iterator]() {
    let index = 0;
    return {
      next: () => {
        if (index < this.items.length) {
          return { value: this.items[index++], done: false };
        }
        return { done: true };
      }
    };
  }

  // Custom string representation
  get [Symbol.toStringTag]() {
    return 'Collection';
  }
}

const coll = new Collection([1, 2, 3]);
[...coll];  // [1, 2, 3]
Object.prototype.toString.call(coll);  // '[object Collection]'
```

---

## Iterators and Generators

### Iterators

```javascript
// Iterator protocol
const iterable = {
  data: [1, 2, 3],
  [Symbol.iterator]() {
    let index = 0;
    return {
      next: () => {
        if (index < this.data.length) {
          return { value: this.data[index++], done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
};

for (const item of iterable) {
  console.log(item);  // 1, 2, 3
}

// Manual iteration
const iterator = iterable[Symbol.iterator]();
iterator.next();  // { value: 1, done: false }
iterator.next();  // { value: 2, done: false }
iterator.next();  // { value: 3, done: false }
iterator.next();  // { value: undefined, done: true }
```

### Generators

```javascript
// Generator function
function* generateNumbers() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = generateNumbers();
gen.next();  // { value: 1, done: false }
gen.next();  // { value: 2, done: false }
gen.next();  // { value: 3, done: false }
gen.next();  // { value: undefined, done: true }

// Spread and for...of work with generators
[...generateNumbers()];  // [1, 2, 3]

// Infinite sequence
function* infiniteSequence() {
  let i = 0;
  while (true) {
    yield i++;
  }
}

// Generator with values passed in
function* calculator() {
  const a = yield 'Enter first number';
  const b = yield 'Enter second number';
  return a + b;
}

const calc = calculator();
calc.next();       // { value: 'Enter first number', done: false }
calc.next(10);     // { value: 'Enter second number', done: false }
calc.next(5);      // { value: 15, done: true }

// Delegating generators
function* gen1() {
  yield 1;
  yield 2;
}

function* gen2() {
  yield* gen1();  // Delegate to gen1
  yield 3;
}

[...gen2()];  // [1, 2, 3]

// Async generator (ES2018)
async function* asyncGenerator() {
  yield await fetch('/api/1').then(r => r.json());
  yield await fetch('/api/2').then(r => r.json());
}

for await (const data of asyncGenerator()) {
  console.log(data);
}
```

---

## Map and Set

### Map

```javascript
const map = new Map();

// Set values (any type as key)
map.set('string', 'value1');
map.set(42, 'value2');
map.set({}, 'value3');

// Get values
map.get('string');  // 'value1'
map.get(42);        // 'value2'

// Check existence
map.has('string');  // true

// Size
map.size;  // 3

// Delete
map.delete('string');

// Clear all
map.clear();

// Initialize with entries
const map2 = new Map([
  ['a', 1],
  ['b', 2]
]);

// Iteration
for (const [key, value] of map2) {
  console.log(key, value);
}

map2.forEach((value, key) => console.log(key, value));

// Convert to array
[...map2.keys()];    // ['a', 'b']
[...map2.values()];  // [1, 2]
[...map2.entries()]; // [['a', 1], ['b', 2]]

// Object vs Map
// - Map allows any key type
// - Map maintains insertion order
// - Map has size property
// - Map is directly iterable
// - Map performs better for frequent additions/removals
```

### Set

```javascript
const set = new Set([1, 2, 3, 3, 3]);  // Duplicates removed

set.add(4);
set.has(2);     // true
set.delete(2);
set.size;       // 3

// Clear all
set.clear();

// Iteration
for (const item of set) {
  console.log(item);
}

set.forEach(item => console.log(item));

// Convert to array
[...set];
Array.from(set);

// Set operations
const a = new Set([1, 2, 3]);
const b = new Set([2, 3, 4]);

// Union
const union = new Set([...a, ...b]);  // {1, 2, 3, 4}

// Intersection
const intersection = new Set([...a].filter(x => b.has(x)));  // {2, 3}

// Difference
const difference = new Set([...a].filter(x => !b.has(x)));  // {1}

// Remove duplicates from array
const unique = [...new Set([1, 2, 2, 3, 3, 3])];  // [1, 2, 3]
```

### WeakMap and WeakSet

```javascript
// WeakMap - keys must be objects, garbage collected when no other reference
const weakMap = new WeakMap();
let obj = { name: 'John' };
weakMap.set(obj, 'metadata');
weakMap.get(obj);  // 'metadata'

obj = null;  // Object can now be garbage collected

// Use case: Store private data
const privateData = new WeakMap();

class Person {
  constructor(name) {
    privateData.set(this, { name });
  }

  getName() {
    return privateData.get(this).name;
  }
}

// WeakSet - values must be objects, garbage collected
const weakSet = new WeakSet();
let user = { name: 'John' };
weakSet.add(user);
weakSet.has(user);  // true

user = null;  // Object can be garbage collected

// Use case: Track visited objects without preventing GC
const visited = new WeakSet();

function process(obj) {
  if (visited.has(obj)) {
    return;  // Already processed
  }
  visited.add(obj);
  // Process object...
}
```

---

## Optional Chaining and Nullish Coalescing

### Optional Chaining (?.)

```javascript
const user = {
  name: 'John',
  address: {
    city: 'NYC'
  }
};

// Property access
user?.address?.city;        // 'NYC'
user?.contact?.email;       // undefined (no error)

// Method calls
user.getName?.();           // undefined if method doesn't exist

// Array access
const arr = [1, 2, 3];
arr?.[0];                   // 1
arr?.[100];                 // undefined

// Dynamic properties
const key = 'name';
user?.[key];                // 'John'

// Short-circuit evaluation
let count = 0;
null?.prop[count++];        // count stays 0
```

### Nullish Coalescing (??)

```javascript
// Returns right side only if left is null or undefined
const value1 = null ?? 'default';     // 'default'
const value2 = undefined ?? 'default'; // 'default'
const value3 = '' ?? 'default';        // '' (empty string is not nullish)
const value4 = 0 ?? 'default';         // 0 (zero is not nullish)
const value5 = false ?? 'default';     // false (false is not nullish)

// Compare with ||
const a = '' || 'default';  // 'default' (|| considers '' falsy)
const b = '' ?? 'default';  // '' (?? only checks null/undefined)

// Combining with optional chaining
const city = user?.address?.city ?? 'Unknown';

// Cannot mix with && or || without parentheses
// (a && b) ?? c  // OK
// a && b ?? c    // SyntaxError
```

---

## ES2020+ Features

### BigInt

```javascript
// For integers larger than Number.MAX_SAFE_INTEGER
const big = 9007199254740991n;
const big2 = BigInt('9007199254740991');

big + 1n;     // 9007199254740992n
big * 2n;     // 18014398509481982n

// Cannot mix with regular numbers
// big + 1;   // TypeError
big + BigInt(1);  // OK

typeof big;  // 'bigint'
```

### Promise.allSettled

```javascript
const promises = [
  Promise.resolve(1),
  Promise.reject('error'),
  Promise.resolve(3)
];

const results = await Promise.allSettled(promises);
// [
//   { status: 'fulfilled', value: 1 },
//   { status: 'rejected', reason: 'error' },
//   { status: 'fulfilled', value: 3 }
// ]
```

### globalThis

```javascript
// Works in all environments (browser, Node, Web Workers)
globalThis.setTimeout;  // Same setTimeout everywhere
```

### String Methods

```javascript
// replaceAll (ES2021)
'aabbcc'.replaceAll('b', 'x');  // 'aaxxcc'

// at() - negative indexing (ES2022)
'hello'.at(-1);   // 'o'
[1, 2, 3].at(-1); // 3
```

### Array Methods

```javascript
// flat (ES2019)
[1, [2, [3, [4]]]].flat();     // [1, 2, [3, [4]]]
[1, [2, [3, [4]]]].flat(2);    // [1, 2, 3, [4]]
[1, [2, [3, [4]]]].flat(Infinity); // [1, 2, 3, 4]

// flatMap (ES2019)
[1, 2, 3].flatMap(x => [x, x * 2]);  // [1, 2, 2, 4, 3, 6]

// Array.from with mapFn
Array.from({ length: 5 }, (_, i) => i * 2);  // [0, 2, 4, 6, 8]

// at() (ES2022)
const arr = [1, 2, 3, 4, 5];
arr.at(-1);   // 5
arr.at(-2);   // 4

// findLast, findLastIndex (ES2023)
const nums = [1, 2, 3, 2, 1];
nums.findLast(x => x === 2);      // 2 (last occurrence)
nums.findLastIndex(x => x === 2); // 3

// toSorted, toReversed, toSpliced, with (ES2023) - immutable versions
const sorted = arr.toSorted((a, b) => b - a);  // New sorted array
const reversed = arr.toReversed();  // New reversed array
const spliced = arr.toSpliced(1, 1, 'new');  // New array with splice
const changed = arr.with(0, 'new');  // New array with element changed
```

### Object Methods

```javascript
// Object.fromEntries (ES2019)
const entries = [['a', 1], ['b', 2]];
Object.fromEntries(entries);  // { a: 1, b: 2 }

// Object.hasOwn (ES2022) - safer than hasOwnProperty
Object.hasOwn({ a: 1 }, 'a');  // true
Object.hasOwn({ a: 1 }, 'b');  // false
```

---

## Key Interview Questions

### Q1: What's the difference between var, let, and const?

```javascript
// var: function-scoped, hoisted, can be redeclared
// let: block-scoped, TDZ, cannot be redeclared
// const: block-scoped, TDZ, cannot be reassigned (but object properties can change)
```

### Q2: Explain spread vs rest

```javascript
// Spread: Expands an iterable into individual elements
const arr = [...[1, 2], 3];  // [1, 2, 3]

// Rest: Collects multiple elements into an array
const [first, ...rest] = [1, 2, 3];  // rest = [2, 3]
```

### Q3: When would you use a Symbol?

```javascript
// 1. Unique property keys that won't conflict
// 2. Implementing well-known protocols (Symbol.iterator)
// 3. Private-ish properties (not truly private but not enumerable)
```

### Q4: Map vs Object?

```javascript
// Use Map when:
// - Keys are not strings
// - Need to preserve insertion order
// - Frequently adding/deleting keys
// - Need to know the size

// Use Object when:
// - Simple string keys
// - Need JSON serialization
// - Need prototype chain
```

---

## Key Takeaways

1. **Destructuring** simplifies extracting values from arrays/objects
2. **Spread** expands; **Rest** collects
3. **Arrow functions** have lexical `this`
4. **Classes** are syntactic sugar over prototypes
5. **Symbols** provide unique, non-enumerable keys
6. **Generators** enable lazy iteration
7. **Map/Set** provide better collection types
8. **Optional chaining** (?.) safely navigates nested properties
9. **Nullish coalescing** (??) only defaults for null/undefined
