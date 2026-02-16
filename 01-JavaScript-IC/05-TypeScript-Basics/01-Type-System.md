# TypeScript Type System

## Overview

TypeScript adds static typing to JavaScript, catching errors at compile time and improving developer experience. Understanding the type system is essential for modern frontend development.

---

## Basic Types

### Primitive Types

```typescript
// String
let name: string = 'John';
let template: string = `Hello, ${name}`;

// Number (includes integers and floats)
let age: number = 30;
let price: number = 19.99;
let hex: number = 0xff;
let binary: number = 0b1010;

// Boolean
let isActive: boolean = true;

// Null and Undefined
let nullValue: null = null;
let undefinedValue: undefined = undefined;

// BigInt
let bigNumber: bigint = 9007199254740991n;

// Symbol
let sym: symbol = Symbol('key');
let uniqueSym: unique symbol = Symbol('unique');
```

### Special Types

```typescript
// Any - opt out of type checking
let anything: any = 'hello';
anything = 42;  // No error
anything.nonExistent();  // No error (runtime error!)

// Unknown - type-safe any
let uncertain: unknown = 'hello';
// uncertain.length;  // Error: Object is of type 'unknown'

// Type guard required
if (typeof uncertain === 'string') {
  console.log(uncertain.length);  // OK
}

// Never - never returns
function throwError(msg: string): never {
  throw new Error(msg);
}

function infiniteLoop(): never {
  while (true) {}
}

// Void - no return value
function logMessage(msg: string): void {
  console.log(msg);
}

// Object - any non-primitive
let obj: object = { name: 'John' };
let arr: object = [1, 2, 3];
let fn: object = () => {};
```

---

## Type Annotations vs Inference

### Type Inference

```typescript
// TypeScript infers types when possible
let inferredString = 'hello';  // string
let inferredNumber = 42;       // number
let inferredArray = [1, 2, 3]; // number[]
let inferredObject = { name: 'John', age: 30 };  // { name: string; age: number }

// Function return type inferred
function add(a: number, b: number) {
  return a + b;  // Return type inferred as number
}

// Complex inference
const users = [
  { name: 'John', age: 30 },
  { name: 'Jane', age: 25 }
];
// Type: { name: string; age: number }[]
```

### When to Annotate

```typescript
// Always annotate function parameters
function greet(name: string): string {
  return `Hello, ${name}`;
}

// Annotate when inference is wrong
let id: string | number = 123;  // Without annotation, would be just number

// Annotate return types for complex functions
function getUser(id: number): User | undefined {
  return users.find(u => u.id === id);
}

// Annotate when declaring without initializing
let name: string;
name = 'John';

// Don't over-annotate - let inference work
// Redundant:
const x: number = 5;
// Better:
const x = 5;
```

---

## Arrays and Tuples

### Arrays

```typescript
// Array types (two syntaxes)
let numbers: number[] = [1, 2, 3];
let strings: Array<string> = ['a', 'b', 'c'];

// Mixed arrays
let mixed: (string | number)[] = [1, 'two', 3];

// Readonly arrays
const readonlyNums: readonly number[] = [1, 2, 3];
// readonlyNums.push(4);  // Error

const readonlyArr: ReadonlyArray<string> = ['a', 'b'];
```

### Tuples

```typescript
// Fixed-length array with specific types per position
let tuple: [string, number] = ['hello', 42];
let name = tuple[0];   // string
let value = tuple[1];  // number

// Named tuple elements
type NameAge = [name: string, age: number];
const person: NameAge = ['John', 30];

// Optional elements
type OptionalTuple = [string, number?];
const withOptional: OptionalTuple = ['only string'];

// Rest elements
type StringNumberBooleans = [string, number, ...boolean[]];
const snb: StringNumberBooleans = ['hello', 1, true, false, true];

// Readonly tuples
const readonlyTuple: readonly [string, number] = ['hello', 42];
// readonlyTuple[0] = 'world';  // Error
```

---

## Objects and Interfaces

### Object Types

```typescript
// Inline object type
function printUser(user: { name: string; age: number }): void {
  console.log(`${user.name} is ${user.age}`);
}

// Optional properties
type Config = {
  host: string;
  port?: number;  // Optional
};

// Readonly properties
type Point = {
  readonly x: number;
  readonly y: number;
};

const p: Point = { x: 10, y: 20 };
// p.x = 5;  // Error: Cannot assign to 'x'

// Index signatures
type StringMap = {
  [key: string]: string;
};

type NumberMap = {
  [key: number]: boolean;
};

// Combining with known properties
type Dictionary = {
  name: string;
  [key: string]: string;  // All values must be string
};
```

### Interfaces

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age?: number;  // Optional
  readonly createdAt: Date;  // Readonly
}

// Extending interfaces
interface Employee extends User {
  department: string;
  salary: number;
}

// Multiple inheritance
interface Manager extends Employee {
  directReports: Employee[];
}

// Interface merging (declaration merging)
interface Window {
  myCustomProperty: string;
}
// Extends built-in Window interface

// Implementing interfaces (classes)
class UserImpl implements User {
  id: number;
  name: string;
  email: string;
  readonly createdAt: Date;

  constructor(id: number, name: string, email: string) {
    this.id = id;
    this.name = name;
    this.email = email;
    this.createdAt = new Date();
  }
}
```

### Type vs Interface

```typescript
// Both can describe object shapes
type UserType = {
  name: string;
  age: number;
};

interface UserInterface {
  name: string;
  age: number;
}

// Differences:

// 1. Interface can be extended/merged
interface A { x: number; }
interface A { y: number; }  // Merges with above
// const a: A = { x: 1, y: 2 };

// 2. Type can use unions, intersections, primitives
type ID = string | number;
type Combined = TypeA & TypeB;
type StringAlias = string;

// 3. Type can use mapped types, conditional types
type Keys = keyof User;
type Partial<T> = { [P in keyof T]?: T[P] };

// General guidance:
// - Interface for object shapes, especially public APIs
// - Type for unions, complex types, utility types
```

---

## Union and Intersection Types

### Union Types

```typescript
// Value can be one of multiple types
type ID = string | number;
let userId: ID = 'abc123';
userId = 123;

// Discriminated unions
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rectangle'; width: number; height: number }
  | { kind: 'square'; size: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius ** 2;
    case 'rectangle':
      return shape.width * shape.height;
    case 'square':
      return shape.size ** 2;
  }
}

// Exhaustiveness checking
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius ** 2;
    case 'rectangle':
      return shape.width * shape.height;
    case 'square':
      return shape.size ** 2;
    default:
      return assertNever(shape);  // Compile error if case missing
  }
}
```

### Intersection Types

```typescript
// Combine multiple types
type WithID = { id: string };
type WithName = { name: string };
type WithID_Name = WithID & WithName;

const item: WithID_Name = { id: '1', name: 'Item' };

// Practical example: Mixin pattern
type Timestamped = {
  createdAt: Date;
  updatedAt: Date;
};

type Authored = {
  authorId: string;
  authorName: string;
};

type BlogPost = {
  title: string;
  content: string;
} & Timestamped & Authored;

const post: BlogPost = {
  title: 'Hello',
  content: 'World',
  createdAt: new Date(),
  updatedAt: new Date(),
  authorId: '1',
  authorName: 'John'
};
```

---

## Type Narrowing

### Type Guards

```typescript
// typeof guard
function padLeft(value: string, padding: string | number) {
  if (typeof padding === 'number') {
    return ' '.repeat(padding) + value;
  }
  return padding + value;
}

// instanceof guard
function processDate(date: Date | string) {
  if (date instanceof Date) {
    return date.toISOString();
  }
  return new Date(date).toISOString();
}

// in operator
interface Bird { fly(): void }
interface Fish { swim(): void }

function move(animal: Bird | Fish) {
  if ('fly' in animal) {
    animal.fly();
  } else {
    animal.swim();
  }
}

// Equality narrowing
function example(x: string | number, y: string | boolean) {
  if (x === y) {
    // x and y are both string here
    x.toUpperCase();
    y.toLowerCase();
  }
}

// Truthiness narrowing
function processValue(value: string | null | undefined) {
  if (value) {
    // value is string here
    console.log(value.length);
  }
}
```

### Custom Type Guards

```typescript
// Type predicate
function isString(value: unknown): value is string {
  return typeof value === 'string';
}

function isUser(obj: unknown): obj is User {
  return (
    typeof obj === 'object' &&
    obj !== null &&
    'name' in obj &&
    'email' in obj
  );
}

// Usage
function processInput(input: unknown) {
  if (isString(input)) {
    console.log(input.toUpperCase());  // input is string
  }

  if (isUser(input)) {
    console.log(input.name, input.email);  // input is User
  }
}

// Assertion function
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== 'string') {
    throw new Error('Value is not a string');
  }
}

function processValue(value: unknown) {
  assertIsString(value);
  // value is string after this point
  console.log(value.toUpperCase());
}
```

---

## Literal Types

### String Literal Types

```typescript
// Specific string values
type Direction = 'north' | 'south' | 'east' | 'west';
let direction: Direction = 'north';
// direction = 'up';  // Error

// Template literal types
type EventName = `on${string}`;
type Handler: EventName = 'onClick';  // OK
// type Invalid: EventName = 'click';  // Error

type HTTPMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type APIEndpoint = `/api/${string}`;
type APIRoute = `${HTTPMethod} ${APIEndpoint}`;
// 'GET /api/users' is valid
```

### Numeric Literal Types

```typescript
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;
let roll: DiceRoll = 3;
// roll = 7;  // Error

type HTTPStatus = 200 | 201 | 400 | 401 | 404 | 500;
```

### Const Assertions

```typescript
// Without const assertion
const config = {
  endpoint: '/api',
  retries: 3
};
// Type: { endpoint: string; retries: number }

// With const assertion
const config = {
  endpoint: '/api',
  retries: 3
} as const;
// Type: { readonly endpoint: '/api'; readonly retries: 3 }

// Arrays as tuples
const tuple = [1, 2, 3] as const;
// Type: readonly [1, 2, 3]

// Function that needs literal type
function route(method: 'GET' | 'POST', path: string) {}

const req = { method: 'GET', path: '/api' };
// route(req.method, req.path);  // Error: string not assignable

const req = { method: 'GET', path: '/api' } as const;
route(req.method, req.path);  // OK
```

---

## Function Types

### Function Type Expressions

```typescript
// Function type
type GreetFunction = (name: string) => string;
type AddFunction = (a: number, b: number) => number;
type VoidFunction = () => void;

const greet: GreetFunction = (name) => `Hello, ${name}`;
const add: AddFunction = (a, b) => a + b;

// Optional and default parameters
function greet(name: string, greeting?: string): string {
  return `${greeting || 'Hello'}, ${name}`;
}

function greetWithDefault(name: string, greeting: string = 'Hello'): string {
  return `${greeting}, ${name}`;
}

// Rest parameters
function sum(...numbers: number[]): number {
  return numbers.reduce((a, b) => a + b, 0);
}

// Function overloads
function getValue(key: 'name'): string;
function getValue(key: 'age'): number;
function getValue(key: string): string | number {
  // Implementation
  if (key === 'name') return 'John';
  if (key === 'age') return 30;
  return '';
}

const name = getValue('name');  // string
const age = getValue('age');     // number
```

### Call Signatures

```typescript
// Object with call signature
type Logger = {
  (message: string): void;
  level: string;
};

const logger: Logger = (message) => console.log(message);
logger.level = 'info';
logger('Hello');

// Construct signatures (for classes)
type Constructor = {
  new (name: string): User;
};

class User {
  constructor(public name: string) {}
}

function createUser(ctor: Constructor, name: string): User {
  return new ctor(name);
}
```

---

## Interview Questions

### Q1: What is the difference between `any` and `unknown`?

```typescript
// any: Disables type checking completely
// - Can assign any value
// - Can call any method/property
// - No compile-time safety

// unknown: Type-safe any
// - Can assign any value
// - Must narrow type before use
// - Provides compile-time safety

let a: any = 'hello';
a.foo();  // No error (runtime error!)

let u: unknown = 'hello';
// u.foo();  // Error: Object is of type 'unknown'

if (typeof u === 'string') {
  u.toUpperCase();  // OK after narrowing
}
```

### Q2: When to use interface vs type?

```typescript
// Interface: Object shapes, public APIs, can be extended/merged
interface User {
  name: string;
}
interface User {  // Declaration merging
  age: number;
}

// Type: Unions, intersections, mapped types, primitives
type ID = string | number;
type UserWithID = User & { id: ID };

// General rule:
// - Interface for defining contracts (classes, objects)
// - Type for complex type transformations
```

### Q3: How does type narrowing work?

```typescript
// TypeScript narrows types based on control flow

function process(value: string | number) {
  // value is string | number

  if (typeof value === 'string') {
    // value is string here
    return value.toUpperCase();
  }

  // value is number here
  return value.toFixed(2);
}

// Custom type guards for complex types
function isUser(obj: unknown): obj is User {
  return typeof obj === 'object' && obj !== null && 'name' in obj;
}
```

### Q4: What are discriminated unions?

```typescript
// Unions with a common discriminant property
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string };

function handleResult<T>(result: Result<T>) {
  if (result.success) {
    // TypeScript knows result.data exists
    console.log(result.data);
  } else {
    // TypeScript knows result.error exists
    console.log(result.error);
  }
}
```

---

## Key Takeaways

1. **Type inference**: Let TypeScript infer when possible
2. **unknown over any**: Use unknown for type-safe dynamic values
3. **Union types**: Use for values that can be multiple types
4. **Intersection types**: Use to combine types
5. **Type narrowing**: Use type guards to refine types
6. **Literal types**: For specific string/number values
7. **const assertions**: Preserve literal types
8. **Interface for objects**: Type for complex types
