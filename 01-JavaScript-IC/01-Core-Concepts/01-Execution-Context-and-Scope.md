# Execution Context and Scope

## Overview

Understanding execution context, scope, and hoisting is fundamental to mastering JavaScript. These concepts explain how JavaScript code is executed and how variables are accessed.

---

## Execution Context

### What is Execution Context?

The execution context is the environment in which JavaScript code is evaluated and executed. It contains:
- Variable Environment
- Scope Chain
- `this` binding

### Types of Execution Context

```javascript
// 1. Global Execution Context (GEC)
// Created when script first runs
// In browsers: window object
// In Node.js: global object

// 2. Function Execution Context (FEC)
// Created when a function is invoked
function greet() {
  // New execution context created here
  const message = "Hello";
  console.log(message);
}

// 3. Eval Execution Context
// Created inside eval() - avoid using eval
```

### Execution Context Phases

#### Phase 1: Creation Phase

```javascript
console.log(name);  // undefined (hoisted)
console.log(greet); // function greet() {...} (hoisted)
console.log(age);   // ReferenceError (let/const not hoisted same way)

var name = "John";
let age = 30;

function greet() {
  return "Hello";
}
```

During creation:
1. `this` binding is determined
2. Outer environment reference is created
3. Variables are hoisted:
   - `var`: initialized as `undefined`
   - `let`/`const`: in "temporal dead zone" (TDZ)
   - Functions: fully hoisted with body

#### Phase 2: Execution Phase

Code is executed line by line, variables are assigned values.

---

## Call Stack

The call stack manages execution contexts using LIFO (Last In, First Out):

```javascript
function first() {
  console.log("First");
  second();
  console.log("First done");
}

function second() {
  console.log("Second");
  third();
  console.log("Second done");
}

function third() {
  console.log("Third");
}

first();

/*
Call Stack:
1. Global Context (pushed first)
2. first() context pushed
3. second() context pushed
4. third() context pushed
5. third() pops off
6. second() pops off
7. first() pops off
*/
```

### Stack Overflow

```javascript
function recursive() {
  recursive();  // No base case
}

recursive();  // Maximum call stack size exceeded
```

---

## Hoisting

### Variable Hoisting

```javascript
// var is hoisted and initialized as undefined
console.log(x);  // undefined
var x = 5;

// Equivalent to:
var x;
console.log(x);  // undefined
x = 5;
```

```javascript
// let and const are hoisted but NOT initialized
// They exist in Temporal Dead Zone (TDZ)
console.log(y);  // ReferenceError: Cannot access 'y' before initialization
let y = 10;

console.log(z);  // ReferenceError
const z = 15;
```

### Function Hoisting

```javascript
// Function declarations are fully hoisted
greet();  // "Hello!" - works

function greet() {
  console.log("Hello!");
}

// Function expressions are NOT hoisted
sayHi();  // TypeError: sayHi is not a function

var sayHi = function() {
  console.log("Hi!");
};

// Arrow functions behave like function expressions
sayBye();  // TypeError

const sayBye = () => {
  console.log("Bye!");
};
```

### Class Hoisting

```javascript
// Classes are hoisted but NOT initialized (like let/const)
const p = new Person();  // ReferenceError

class Person {
  constructor() {
    this.name = "John";
  }
}
```

---

## Scope

### Types of Scope

#### 1. Global Scope

```javascript
var globalVar = "I'm global";
let globalLet = "I'm also global";

function test() {
  console.log(globalVar);  // Accessible
  console.log(globalLet);  // Accessible
}

// In browser: attached to window
console.log(window.globalVar);  // "I'm global"
console.log(window.globalLet);  // undefined (let/const not on window)
```

#### 2. Function Scope

```javascript
function outer() {
  var functionScoped = "Only in this function";
  let alsoFunctionScoped = "Same here";

  function inner() {
    console.log(functionScoped);  // Accessible (closure)
  }
}

console.log(functionScoped);  // ReferenceError
```

#### 3. Block Scope

```javascript
if (true) {
  var varVariable = "var";
  let letVariable = "let";
  const constVariable = "const";
}

console.log(varVariable);    // "var" - var ignores block scope
console.log(letVariable);    // ReferenceError - block scoped
console.log(constVariable);  // ReferenceError - block scoped
```

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 3, 3, 3 (var is function scoped)

for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log(j), 100);
}
// Prints: 0, 1, 2 (let creates new binding each iteration)
```

---

## Scope Chain

```javascript
const global = "global";

function outer() {
  const outerVar = "outer";

  function inner() {
    const innerVar = "inner";

    console.log(innerVar);   // "inner" - own scope
    console.log(outerVar);   // "outer" - parent scope
    console.log(global);     // "global" - global scope
  }

  inner();
}

outer();

/*
Scope Chain for inner():
1. inner's own scope
2. outer's scope
3. Global scope
*/
```

### Variable Shadowing

```javascript
const name = "Global";

function outer() {
  const name = "Outer";  // Shadows global

  function inner() {
    const name = "Inner";  // Shadows outer
    console.log(name);     // "Inner"
  }

  inner();
  console.log(name);  // "Outer"
}

outer();
console.log(name);  // "Global"
```

### Illegal Shadowing

```javascript
function test() {
  var a = 10;
  let b = 20;

  {
    let a = 30;   // OK - var can be shadowed by let
    // var b = 40;  // SyntaxError - let cannot be shadowed by var in same scope
  }
}
```

---

## Lexical Scope (Static Scope)

JavaScript uses lexical scoping - scope is determined by where functions are defined, not where they're called:

```javascript
const x = 10;

function outer() {
  const x = 20;

  function inner() {
    console.log(x);  // 20 - uses x from where inner is defined
  }

  return inner;
}

const innerFunc = outer();

function other() {
  const x = 30;
  innerFunc();  // Still prints 20, not 30
}

other();
```

---

## var vs let vs const

| Feature | var | let | const |
|---------|-----|-----|-------|
| Scope | Function | Block | Block |
| Hoisting | Yes (undefined) | Yes (TDZ) | Yes (TDZ) |
| Re-declaration | Yes | No | No |
| Re-assignment | Yes | Yes | No |
| Global object | Yes | No | No |

### Best Practices

```javascript
// Modern JavaScript: prefer const, use let when needed
const config = { api: "https://api.example.com" };
const numbers = [1, 2, 3];

// Use let only when reassignment is needed
let count = 0;
count++;

// Avoid var in modern code
// var x = 10;  // Avoid
```

---

## Interview Questions

### Q1: What will be logged?

```javascript
var a = 1;
function foo() {
  console.log(a);
  var a = 2;
}
foo();
```

**Answer:** `undefined`

Due to hoisting, `var a` is hoisted within `foo()`, so local `a` exists but is `undefined` when logged.

### Q2: What will be logged?

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

**Answer:** `3, 3, 3`

`var` is function-scoped, so all callbacks share the same `i`, which is 3 after the loop.

### Q3: Fix the above to print 0, 1, 2

```javascript
// Solution 1: Use let
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}

// Solution 2: IIFE
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 0);
  })(i);
}

// Solution 3: setTimeout third parameter
for (var i = 0; i < 3; i++) {
  setTimeout((j) => console.log(j), 0, i);
}
```

### Q4: What's the output?

```javascript
function test() {
  console.log(typeof x);
  console.log(typeof y);
  var x = 1;
  let y = 2;
}
test();
```

**Answer:** `undefined`, then `ReferenceError`

`var x` is hoisted as `undefined`, but accessing `y` before declaration causes ReferenceError due to TDZ.

---

## Key Takeaways

1. **Execution context** has creation and execution phases
2. **Hoisting** moves declarations to top (but differently for var/let/const/functions)
3. **var** is function-scoped; **let/const** are block-scoped
4. **Scope chain** follows lexical (where defined) not dynamic (where called) scoping
5. **Prefer const**, use **let** when needed, **avoid var**
