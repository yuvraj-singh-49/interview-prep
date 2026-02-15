# The `this` Keyword

## Overview

The `this` keyword is one of the most confusing aspects of JavaScript. Its value is determined by **how a function is called**, not where it's defined (except for arrow functions).

---

## The Four Rules of `this`

### Rule 1: Default Binding (Global/undefined)

When a function is called without any context:

```javascript
function showThis() {
  console.log(this);
}

showThis();
// Non-strict mode: window (browser) or global (Node)
// Strict mode: undefined
```

```javascript
"use strict";

function strictShowThis() {
  console.log(this);
}

strictShowThis();  // undefined
```

### Rule 2: Implicit Binding (Object Method)

When a function is called as a method of an object:

```javascript
const person = {
  name: "John",
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  }
};

person.greet();  // "Hello, I'm John"
// this = person (the object before the dot)
```

**Implicit binding is lost:**

```javascript
const person = {
  name: "John",
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  }
};

const greetFunc = person.greet;
greetFunc();  // "Hello, I'm undefined"
// this = window/global (function called without object context)

// Also lost in callbacks
setTimeout(person.greet, 100);  // "Hello, I'm undefined"

// Fix with bind
setTimeout(person.greet.bind(person), 100);  // "Hello, I'm John"
```

### Rule 3: Explicit Binding (call, apply, bind)

Explicitly set `this` using `call`, `apply`, or `bind`:

```javascript
function greet(greeting, punctuation) {
  console.log(`${greeting}, I'm ${this.name}${punctuation}`);
}

const person = { name: "John" };

// call - arguments passed individually
greet.call(person, "Hello", "!");  // "Hello, I'm John!"

// apply - arguments passed as array
greet.apply(person, ["Hi", "?"]);  // "Hi, I'm John?"

// bind - returns new function with bound this
const boundGreet = greet.bind(person);
boundGreet("Hey", ".");  // "Hey, I'm John."

const boundWithArgs = greet.bind(person, "Hello");
boundWithArgs("!!!");  // "Hello, I'm John!!!"
```

### Rule 4: new Binding (Constructor)

When a function is called with `new`:

```javascript
function Person(name) {
  // 1. A new empty object is created
  // 2. this is bound to that new object
  this.name = name;
  // 3. The new object is returned (unless function returns an object)
}

const john = new Person("John");
console.log(john.name);  // "John"
```

**What `new` does:**
1. Creates a new empty object
2. Sets the object's prototype to the constructor's prototype
3. Binds `this` to the new object
4. Returns the new object (unless the constructor returns an object)

```javascript
function Person(name) {
  this.name = name;
  return { name: "Overridden" };  // Returning an object overrides
}

const person = new Person("John");
console.log(person.name);  // "Overridden"
```

---

## Priority of Rules

From highest to lowest priority:

1. **new binding** - `new Foo()`
2. **Explicit binding** - `call`, `apply`, `bind`
3. **Implicit binding** - `obj.method()`
4. **Default binding** - `func()`

```javascript
function foo() {
  console.log(this.a);
}

const obj1 = { a: 1, foo };
const obj2 = { a: 2 };

// Implicit vs Explicit
obj1.foo();              // 1 (implicit)
obj1.foo.call(obj2);     // 2 (explicit wins)

// Explicit vs new
const BoundFoo = foo.bind(obj1);
BoundFoo();              // 1 (bound to obj1)

const newObj = new BoundFoo();  // undefined
// new wins over bind, this = newly created object (no 'a' property)
```

---

## Arrow Functions

Arrow functions **do NOT have their own `this`**. They inherit `this` from the enclosing scope (lexical `this`):

```javascript
const person = {
  name: "John",

  // Regular function
  greetRegular: function() {
    console.log(this.name);
  },

  // Arrow function
  greetArrow: () => {
    console.log(this.name);  // undefined - inherits from global
  },

  // Arrow function inside regular function
  greetDelayed: function() {
    setTimeout(() => {
      console.log(this.name);  // "John" - inherits from greetDelayed
    }, 100);
  },

  // Regular function inside regular function (problem)
  greetDelayedProblem: function() {
    setTimeout(function() {
      console.log(this.name);  // undefined - new this binding
    }, 100);
  }
};

person.greetRegular();       // "John"
person.greetArrow();         // undefined
person.greetDelayed();       // "John" (after 100ms)
person.greetDelayedProblem();// undefined (after 100ms)
```

**Arrow functions cannot be:**
- Used as constructors (`new`)
- Bound to a different `this` using `call`, `apply`, `bind`
- Used as methods that need dynamic `this`

```javascript
const obj = {
  name: "Object",
  // Bad: arrow function as method
  getName: () => this.name  // undefined - inherits from where obj is defined
};

// Good: regular function
const obj2 = {
  name: "Object",
  getName() { return this.name; }  // "Object"
};
```

---

## `this` in Different Contexts

### In Event Handlers

```javascript
// Regular function: this = element that triggered event
button.addEventListener('click', function() {
  console.log(this);  // button element
  this.classList.add('clicked');
});

// Arrow function: this = enclosing scope's this
button.addEventListener('click', () => {
  console.log(this);  // window/undefined (not the button!)
});

// If you need both element and enclosing this:
const controller = {
  handleClick(event) {
    console.log(event.target);  // element
    console.log(this);          // controller (if bound)
  },

  init() {
    button.addEventListener('click', this.handleClick.bind(this));
  }
};
```

### In Classes

```javascript
class Person {
  constructor(name) {
    this.name = name;
  }

  // Regular method
  greet() {
    console.log(`Hello, ${this.name}`);
  }

  // Arrow function as class property (bound automatically)
  greetArrow = () => {
    console.log(`Hello, ${this.name}`);
  }
}

const person = new Person("John");

// Regular method loses this
const greet = person.greet;
greet();  // undefined

// Arrow property preserves this
const greetArrow = person.greetArrow;
greetArrow();  // "Hello, John"
```

### In Callbacks

```javascript
const user = {
  name: "John",
  friends: ["Jane", "Bob"],

  // Problem: this lost in callback
  showFriendsWrong() {
    this.friends.forEach(function(friend) {
      console.log(`${this.name} is friends with ${friend}`);
      // this.name is undefined
    });
  },

  // Solution 1: Arrow function
  showFriendsArrow() {
    this.friends.forEach((friend) => {
      console.log(`${this.name} is friends with ${friend}`);
    });
  },

  // Solution 2: bind
  showFriendsBind() {
    this.friends.forEach(function(friend) {
      console.log(`${this.name} is friends with ${friend}`);
    }.bind(this));
  },

  // Solution 3: forEach's thisArg
  showFriendsThisArg() {
    this.friends.forEach(function(friend) {
      console.log(`${this.name} is friends with ${friend}`);
    }, this);  // Second argument is thisArg
  },

  // Solution 4: Store reference
  showFriendsRef() {
    const self = this;
    this.friends.forEach(function(friend) {
      console.log(`${self.name} is friends with ${friend}`);
    });
  }
};
```

---

## Common Interview Questions

### Q1: What will be logged?

```javascript
const obj = {
  name: "Object",
  getName: function() {
    return this.name;
  }
};

const getName = obj.getName;
console.log(getName());
```

**Answer:** `undefined` (or `""` if global `name` exists)

The function is called without object context.

### Q2: What will be logged?

```javascript
const obj = {
  name: "Object",
  inner: {
    name: "Inner",
    getName: function() {
      return this.name;
    }
  }
};

console.log(obj.inner.getName());
```

**Answer:** `"Inner"`

`this` refers to the immediate object before the method (`inner`).

### Q3: What will be logged?

```javascript
function foo() {
  console.log(this.bar);
}

const obj1 = { bar: "obj1", foo };
const obj2 = { bar: "obj2" };

const bound = foo.bind(obj1);
bound.call(obj2);
```

**Answer:** `"obj1"`

`bind` creates a permanently bound function. `call` cannot override it.

### Q4: Implement a bind polyfill

```javascript
Function.prototype.myBind = function(context, ...boundArgs) {
  const fn = this;

  return function(...args) {
    return fn.apply(context, [...boundArgs, ...args]);
  };
};

// Usage
function greet(greeting, punct) {
  return `${greeting}, ${this.name}${punct}`;
}

const person = { name: "John" };
const boundGreet = greet.myBind(person, "Hello");
console.log(boundGreet("!"));  // "Hello, John!"
```

### Q5: Implement call

```javascript
Function.prototype.myCall = function(context, ...args) {
  context = context || globalThis;
  const fnSymbol = Symbol('fn');

  context[fnSymbol] = this;
  const result = context[fnSymbol](...args);
  delete context[fnSymbol];

  return result;
};
```

### Q6: Implement apply

```javascript
Function.prototype.myApply = function(context, args = []) {
  context = context || globalThis;
  const fnSymbol = Symbol('fn');

  context[fnSymbol] = this;
  const result = context[fnSymbol](...args);
  delete context[fnSymbol];

  return result;
};
```

---

## `this` Cheat Sheet

| Call Type | `this` Value |
|-----------|-------------|
| `func()` | `window`/`global` or `undefined` (strict) |
| `obj.method()` | `obj` |
| `func.call(ctx)` | `ctx` |
| `func.apply(ctx)` | `ctx` |
| `func.bind(ctx)()` | `ctx` (permanent) |
| `new Func()` | New object |
| Arrow function | Inherited from enclosing scope |
| Event handler | Element (regular fn) or inherited (arrow) |

---

## Key Takeaways

1. `this` is determined by **how** a function is called, not where defined
2. Priority: `new` > explicit (`call`/`apply`/`bind`) > implicit (method) > default
3. Arrow functions **inherit** `this` from enclosing scope
4. Lost `this` is a common bug - watch callbacks and event handlers
5. Use `bind` or arrow functions to preserve `this`
6. Classes use `this` like constructor functions
