# Prototypes and Inheritance

## Overview

JavaScript uses prototypal inheritance, which is different from classical inheritance in languages like Java or C++. Understanding prototypes is crucial for senior-level JavaScript interviews.

---

## The Prototype Chain

Every JavaScript object has an internal link to another object called its **prototype**. When accessing a property, JavaScript looks up the prototype chain.

```javascript
const animal = {
  eats: true,
  walk() {
    console.log("Animal walks");
  }
};

const rabbit = {
  jumps: true,
  __proto__: animal  // rabbit inherits from animal
};

console.log(rabbit.jumps);  // true (own property)
console.log(rabbit.eats);   // true (inherited from animal)
rabbit.walk();              // "Animal walks" (inherited method)

// Check prototype chain
console.log(rabbit.__proto__ === animal);  // true
console.log(animal.__proto__ === Object.prototype);  // true
console.log(Object.prototype.__proto__);  // null (end of chain)
```

### Visual Representation

```
rabbit
  │
  └──> animal
        │
        └──> Object.prototype
              │
              └──> null
```

---

## Object.prototype

```javascript
const obj = {};

// Object.prototype methods available on all objects
obj.hasOwnProperty('key');
obj.toString();
obj.valueOf();
obj.isPrototypeOf(other);

// Check if property exists (own or inherited)
'toString' in obj;  // true

// Check if property is object's own
obj.hasOwnProperty('toString');  // false
```

---

## Constructor Functions

Before ES6 classes, constructor functions were the primary way to create objects:

```javascript
function Person(name, age) {
  this.name = name;
  this.age = age;
}

// Methods on prototype (shared across instances)
Person.prototype.greet = function() {
  return `Hi, I'm ${this.name}`;
};

Person.prototype.species = 'Homo sapiens';

const john = new Person('John', 30);
const jane = new Person('Jane', 25);

console.log(john.greet());  // "Hi, I'm John"
console.log(jane.greet());  // "Hi, I'm Jane"

// Both share the same prototype
console.log(john.__proto__ === Person.prototype);  // true
console.log(jane.__proto__ === Person.prototype);  // true
console.log(john.greet === jane.greet);  // true (same function)

// Constructor property
console.log(john.constructor === Person);  // true
```

### What `new` Does

```javascript
function Person(name) {
  // 1. Create empty object: {}
  // 2. Set its [[Prototype]] to Person.prototype
  // 3. Bind 'this' to the new object
  this.name = name;
  // 4. Return 'this' (unless function returns an object)
}

// Equivalent manual process:
function createPerson(name) {
  const obj = Object.create(Person.prototype);
  obj.name = name;
  return obj;
}
```

---

## Inheritance with Constructor Functions

```javascript
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function() {
  console.log(`${this.name} makes a sound`);
};

function Dog(name, breed) {
  Animal.call(this, name);  // Call parent constructor
  this.breed = breed;
}

// Set up inheritance
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;  // Fix constructor reference

// Override method
Dog.prototype.speak = function() {
  console.log(`${this.name} barks`);
};

// Add new method
Dog.prototype.fetch = function() {
  console.log(`${this.name} fetches the ball`);
};

const dog = new Dog('Rex', 'German Shepherd');
dog.speak();  // "Rex barks"
dog.fetch();  // "Rex fetches the ball"

console.log(dog instanceof Dog);     // true
console.log(dog instanceof Animal);  // true
```

---

## ES6 Classes

Classes are syntactic sugar over prototypes:

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    console.log(`${this.name} makes a sound`);
  }

  // Static method (on class, not instances)
  static isAnimal(obj) {
    return obj instanceof Animal;
  }

  // Getter
  get info() {
    return `Animal: ${this.name}`;
  }

  // Setter
  set nickname(value) {
    this._nickname = value;
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);  // Must call super before using 'this'
    this.breed = breed;
  }

  speak() {
    console.log(`${this.name} barks`);
  }

  // Call parent method
  speakPolitely() {
    super.speak();
    console.log('(but quietly)');
  }
}

const dog = new Dog('Rex', 'German Shepherd');
dog.speak();         // "Rex barks"
dog.speakPolitely(); // "Rex makes a sound" + "(but quietly)"

console.log(Animal.isAnimal(dog));  // true
console.log(dog.info);              // "Animal: Rex"
```

### Classes Are Still Functions

```javascript
class Person {
  constructor(name) {
    this.name = name;
  }
}

console.log(typeof Person);  // "function"
console.log(Person.prototype.constructor === Person);  // true

const john = new Person('John');
console.log(john.__proto__ === Person.prototype);  // true
```

---

## Object.create()

Create an object with a specific prototype:

```javascript
const animal = {
  speak() {
    console.log('Animal speaks');
  }
};

// Create object with animal as prototype
const dog = Object.create(animal);
dog.bark = function() {
  console.log('Dog barks');
};

dog.speak();  // "Animal speaks" (inherited)
dog.bark();   // "Dog barks" (own method)

// Create object with no prototype
const plainObj = Object.create(null);
console.log(plainObj.toString);  // undefined (no Object.prototype)

// Create with property descriptors
const cat = Object.create(animal, {
  name: {
    value: 'Whiskers',
    writable: true,
    enumerable: true,
    configurable: true
  }
});
```

---

## Prototype Methods

### Object.getPrototypeOf()

```javascript
const proto = Object.getPrototypeOf(dog);
console.log(proto === animal);  // true
```

### Object.setPrototypeOf()

```javascript
Object.setPrototypeOf(dog, newPrototype);
// Note: This is slow - avoid in performance-critical code
```

### isPrototypeOf()

```javascript
console.log(animal.isPrototypeOf(dog));  // true
console.log(Object.prototype.isPrototypeOf(dog));  // true
```

### instanceof

```javascript
console.log(dog instanceof Object);  // true
console.log([] instanceof Array);    // true
console.log([] instanceof Object);   // true

// Custom instanceof behavior
class MyArray {
  static [Symbol.hasInstance](obj) {
    return Array.isArray(obj);
  }
}

console.log([] instanceof MyArray);  // true
```

---

## Property Lookup and Shadowing

```javascript
const parent = {
  name: 'Parent',
  greet() {
    return `Hello from ${this.name}`;
  }
};

const child = Object.create(parent);
child.name = 'Child';  // Shadows parent's name

console.log(child.greet());  // "Hello from Child"
console.log(parent.greet()); // "Hello from Parent"

// Accessing parent's property
console.log(child.__proto__.name);  // "Parent"
```

### hasOwnProperty vs in

```javascript
const child = Object.create(parent);
child.age = 10;

console.log('age' in child);      // true (own property)
console.log('name' in child);     // true (inherited)
console.log(child.hasOwnProperty('age'));   // true
console.log(child.hasOwnProperty('name'));  // false (inherited)

// Safe hasOwnProperty (in case it's overridden)
Object.prototype.hasOwnProperty.call(child, 'age');
```

---

## Property Descriptors

```javascript
const obj = {};

Object.defineProperty(obj, 'readOnly', {
  value: 42,
  writable: false,      // Can't change value
  enumerable: true,     // Shows in for...in
  configurable: false   // Can't delete or reconfigure
});

obj.readOnly = 100;  // Fails silently (or throws in strict mode)
console.log(obj.readOnly);  // 42

// Get descriptor
console.log(Object.getOwnPropertyDescriptor(obj, 'readOnly'));
// { value: 42, writable: false, enumerable: true, configurable: false }

// Define multiple properties
Object.defineProperties(obj, {
  prop1: { value: 1, writable: true },
  prop2: { value: 2, writable: true }
});
```

---

## Common Interview Questions

### Q1: What's the difference between __proto__ and prototype?

```javascript
function Person(name) {
  this.name = name;
}

const john = new Person('John');

// prototype is a property on constructor functions
console.log(Person.prototype);  // { constructor: Person }

// __proto__ is the actual prototype of an instance
console.log(john.__proto__ === Person.prototype);  // true

// __proto__ is deprecated; use Object.getPrototypeOf
console.log(Object.getPrototypeOf(john) === Person.prototype);  // true
```

### Q2: Implement classical inheritance

```javascript
function inherit(Child, Parent) {
  Child.prototype = Object.create(Parent.prototype);
  Child.prototype.constructor = Child;
}

function Animal(name) {
  this.name = name;
}
Animal.prototype.speak = function() {
  console.log(this.name + ' speaks');
};

function Dog(name, breed) {
  Animal.call(this, name);
  this.breed = breed;
}
inherit(Dog, Animal);

Dog.prototype.bark = function() {
  console.log(this.name + ' barks');
};
```

### Q3: Implement Object.create

```javascript
function objectCreate(proto, propertiesObject) {
  if (typeof proto !== 'object' && typeof proto !== 'function') {
    throw new TypeError('Object prototype may only be an Object or null');
  }

  function F() {}
  F.prototype = proto;
  const obj = new F();

  if (propertiesObject !== undefined) {
    Object.defineProperties(obj, propertiesObject);
  }

  return obj;
}
```

### Q4: Implement instanceof

```javascript
function myInstanceOf(obj, constructor) {
  if (obj === null || typeof obj !== 'object') {
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
```

### Q5: What will be logged?

```javascript
function Animal() {}
Animal.prototype.speak = function() { return 'Animal'; };

function Dog() {}
Dog.prototype = new Animal();
Dog.prototype.speak = function() { return 'Dog'; };

const dog = new Dog();
console.log(dog.speak());

delete dog.speak;
console.log(dog.speak());

delete Dog.prototype.speak;
console.log(dog.speak());
```

**Answer:** `Dog`, `Dog`, `Animal`

---

## Key Takeaways

1. **Prototype chain** is how JavaScript implements inheritance
2. Every object has a **[[Prototype]]** (accessed via `__proto__` or `Object.getPrototypeOf`)
3. **Constructor functions** use `prototype` property for shared methods
4. ES6 **classes** are syntactic sugar over prototypes
5. Use `Object.create()` for explicit prototype setting
6. Property lookup goes up the prototype chain until null
7. `hasOwnProperty` checks only own properties
8. `instanceof` checks the prototype chain
