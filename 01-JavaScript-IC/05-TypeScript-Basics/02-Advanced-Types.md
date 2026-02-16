# Advanced TypeScript Types

## Overview

Advanced TypeScript features enable powerful type transformations and reusable type patterns. This covers generics, utility types, conditional types, and mapped types.

---

## Generics

### Basic Generics

```typescript
// Generic function
function identity<T>(value: T): T {
  return value;
}

identity<string>('hello');  // Explicit
identity(42);               // Inferred as number

// Generic with arrays
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

const first = firstElement([1, 2, 3]);  // number | undefined

// Multiple type parameters
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

const p = pair('hello', 42);  // [string, number]

// Generic type alias
type Container<T> = {
  value: T;
};

type StringContainer = Container<string>;
const sc: StringContainer = { value: 'hello' };

// Generic interface
interface Response<T> {
  data: T;
  status: number;
  message: string;
}

const userResponse: Response<User> = {
  data: { name: 'John', email: 'john@example.com' },
  status: 200,
  message: 'Success'
};
```

### Generic Constraints

```typescript
// Constrain T to have length property
function logLength<T extends { length: number }>(value: T): T {
  console.log(value.length);
  return value;
}

logLength('hello');      // OK
logLength([1, 2, 3]);    // OK
// logLength(123);       // Error: number doesn't have length

// Constrain to specific type
interface Identifiable {
  id: string | number;
}

function findById<T extends Identifiable>(items: T[], id: T['id']): T | undefined {
  return items.find(item => item.id === id);
}

// keyof constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: 'John', age: 30 };
const name = getProperty(user, 'name');  // string
const age = getProperty(user, 'age');    // number
// getProperty(user, 'email');            // Error
```

### Generic Classes

```typescript
class Queue<T> {
  private items: T[] = [];

  enqueue(item: T): void {
    this.items.push(item);
  }

  dequeue(): T | undefined {
    return this.items.shift();
  }

  peek(): T | undefined {
    return this.items[0];
  }

  get length(): number {
    return this.items.length;
  }
}

const numberQueue = new Queue<number>();
numberQueue.enqueue(1);
numberQueue.enqueue(2);
numberQueue.dequeue();  // 1

// Generic class with constraint
class Repository<T extends { id: string }> {
  private items: Map<string, T> = new Map();

  save(item: T): void {
    this.items.set(item.id, item);
  }

  findById(id: string): T | undefined {
    return this.items.get(id);
  }

  findAll(): T[] {
    return Array.from(this.items.values());
  }
}
```

### Default Type Parameters

```typescript
// Default generic type
interface RequestConfig<T = any> {
  url: string;
  method: 'GET' | 'POST';
  data?: T;
}

const config1: RequestConfig = { url: '/api', method: 'GET' };
const config2: RequestConfig<User> = {
  url: '/api/users',
  method: 'POST',
  data: { name: 'John', email: 'john@example.com' }
};

// Factory function with defaults
function createState<T = string>(initial: T) {
  let state = initial;
  return {
    get: () => state,
    set: (value: T) => { state = value; }
  };
}

const stringState = createState('hello');  // T = string
const numberState = createState(42);       // T = number
```

---

## Built-in Utility Types

### Partial, Required, Readonly

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  age?: number;
}

// Partial - all properties optional
type PartialUser = Partial<User>;
// { id?: string; name?: string; email?: string; age?: number }

function updateUser(id: string, updates: Partial<User>) {
  // Can update any subset of properties
}

// Required - all properties required
type RequiredUser = Required<User>;
// { id: string; name: string; email: string; age: number }

// Readonly - all properties readonly
type ReadonlyUser = Readonly<User>;
const user: ReadonlyUser = { id: '1', name: 'John', email: 'john@example.com' };
// user.name = 'Jane';  // Error
```

### Pick and Omit

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Pick - select specific properties
type UserBasic = Pick<User, 'id' | 'name' | 'email'>;
// { id: string; name: string; email: string }

// Omit - exclude specific properties
type UserWithoutPassword = Omit<User, 'password'>;
// { id: string; name: string; email: string; createdAt: Date }

// Practical use: API response without sensitive data
type PublicUser = Omit<User, 'password'>;

function getPublicProfile(user: User): PublicUser {
  const { password, ...publicData } = user;
  return publicData;
}
```

### Record

```typescript
// Record - object type with specific key and value types
type UserMap = Record<string, User>;

const users: UserMap = {
  user1: { id: '1', name: 'John', email: 'john@example.com' },
  user2: { id: '2', name: 'Jane', email: 'jane@example.com' }
};

// With union keys
type Status = 'pending' | 'approved' | 'rejected';
type StatusCounts = Record<Status, number>;

const counts: StatusCounts = {
  pending: 5,
  approved: 10,
  rejected: 2
};

// Record vs index signature
type IndexSig = { [key: string]: number };
type RecordType = Record<string, number>;
// Both are equivalent for string keys
```

### Extract and Exclude

```typescript
type AllTypes = string | number | boolean | null | undefined;

// Extract - keep types assignable to union
type Primitives = Extract<AllTypes, string | number | boolean>;
// string | number | boolean

// Exclude - remove types assignable to union
type NonNullable = Exclude<AllTypes, null | undefined>;
// string | number | boolean

// Practical example
type Response = { type: 'success'; data: User } | { type: 'error'; message: string };

type SuccessResponse = Extract<Response, { type: 'success' }>;
// { type: 'success'; data: User }

type ErrorResponse = Extract<Response, { type: 'error' }>;
// { type: 'error'; message: string }
```

### ReturnType and Parameters

```typescript
function getUser(id: string, includeDetails: boolean): User {
  return {} as User;
}

// ReturnType - extract function return type
type UserReturn = ReturnType<typeof getUser>;
// User

// Parameters - extract parameter types as tuple
type UserParams = Parameters<typeof getUser>;
// [id: string, includeDetails: boolean]

// ConstructorParameters - for class constructors
class UserService {
  constructor(private apiUrl: string, private timeout: number) {}
}

type ServiceParams = ConstructorParameters<typeof UserService>;
// [apiUrl: string, timeout: number]

// InstanceType - instance type of constructor
type ServiceInstance = InstanceType<typeof UserService>;
// UserService
```

### NonNullable

```typescript
type MaybeString = string | null | undefined;

type DefinitelyString = NonNullable<MaybeString>;
// string

// Useful in strictNullChecks mode
function processValue(value: string | null) {
  if (value !== null) {
    const nonNull: NonNullable<typeof value> = value;
  }
}
```

---

## Conditional Types

### Basic Conditional Types

```typescript
// T extends U ? X : Y
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;   // true
type B = IsString<number>;   // false

// Nested conditionals
type TypeName<T> =
  T extends string ? 'string' :
  T extends number ? 'number' :
  T extends boolean ? 'boolean' :
  T extends undefined ? 'undefined' :
  T extends Function ? 'function' :
  'object';

type T1 = TypeName<string>;    // 'string'
type T2 = TypeName<() => void>; // 'function'
type T3 = TypeName<string[]>;   // 'object'
```

### Distributive Conditional Types

```typescript
// Conditional types distribute over unions
type ToArray<T> = T extends any ? T[] : never;

type StrOrNumArray = ToArray<string | number>;
// string[] | number[]  (distributed!)

// Prevent distribution with tuple
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;

type Combined = ToArrayNonDist<string | number>;
// (string | number)[]  (not distributed)
```

### infer Keyword

```typescript
// Extract types from other types
type GetReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type FnReturn = GetReturnType<() => string>;  // string

// Extract array element type
type ElementType<T> = T extends (infer E)[] ? E : never;

type Elem = ElementType<string[]>;  // string

// Extract Promise value
type UnwrapPromise<T> = T extends Promise<infer V> ? V : T;

type Unwrapped = UnwrapPromise<Promise<string>>;  // string
type Same = UnwrapPromise<number>;                 // number

// Recursive unwrap
type DeepUnwrap<T> = T extends Promise<infer V> ? DeepUnwrap<V> : T;

type Deep = DeepUnwrap<Promise<Promise<Promise<string>>>>;  // string

// Extract function arguments
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;

type First = FirstArg<(a: string, b: number) => void>;  // string
```

---

## Mapped Types

### Basic Mapped Types

```typescript
// Create new type by mapping over keys
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};

type Partial<T> = {
  [K in keyof T]?: T[K];
};

// Custom mapped type
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

interface User {
  name: string;
  age: number;
}

type NullableUser = Nullable<User>;
// { name: string | null; age: number | null }
```

### Key Remapping

```typescript
// Remap keys with 'as' clause
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface Person {
  name: string;
  age: number;
}

type PersonGetters = Getters<Person>;
// { getName: () => string; getAge: () => number }

// Filter keys
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

interface Mixed {
  name: string;
  age: number;
  email: string;
}

type StringProps = OnlyStrings<Mixed>;
// { name: string; email: string }

// Remove specific keys
type OmitByKey<T, K extends keyof T> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};
```

### Modifiers

```typescript
// Remove modifiers with -
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

type Required<T> = {
  [K in keyof T]-?: T[K];
};

// Add modifiers with + (implicit)
type ReadonlyPartial<T> = {
  +readonly [K in keyof T]+?: T[K];
};
```

---

## Template Literal Types

### Basic Template Literals

```typescript
type Greeting = `Hello, ${string}`;
let greeting: Greeting = 'Hello, World';

type EmailLocale = `${string}@${string}.${string}`;

// With unions
type Size = 'small' | 'medium' | 'large';
type Color = 'red' | 'blue' | 'green';
type ColoredSize = `${Color}-${Size}`;
// 'red-small' | 'red-medium' | ... (9 combinations)

// HTTP methods
type Method = 'GET' | 'POST' | 'PUT' | 'DELETE';
type Endpoint = '/users' | '/posts' | '/comments';
type Route = `${Method} ${Endpoint}`;
```

### Intrinsic String Types

```typescript
// Built-in string manipulation types
type Upper = Uppercase<'hello'>;      // 'HELLO'
type Lower = Lowercase<'HELLO'>;      // 'hello'
type Cap = Capitalize<'hello'>;       // 'Hello'
type Uncap = Uncapitalize<'Hello'>;   // 'hello'

// Use in mapped types
type EventHandlers<T> = {
  [K in keyof T as `on${Capitalize<string & K>}`]: (value: T[K]) => void;
};

interface State {
  count: number;
  name: string;
}

type StateHandlers = EventHandlers<State>;
// { onCount: (value: number) => void; onName: (value: string) => void }
```

### Pattern Matching with Template Literals

```typescript
// Extract parts from string literal types
type ExtractId<T> = T extends `user-${infer Id}` ? Id : never;

type UserId = ExtractId<'user-123'>;  // '123'
type Invalid = ExtractId<'post-456'>; // never

// Parse route parameters
type ParseRoute<T> = T extends `${string}/:${infer Param}/${infer Rest}`
  ? Param | ParseRoute<`/${Rest}`>
  : T extends `${string}/:${infer Param}`
  ? Param
  : never;

type Params = ParseRoute<'/users/:userId/posts/:postId'>;
// 'userId' | 'postId'
```

---

## Custom Utility Types

### DeepPartial

```typescript
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object
    ? DeepPartial<T[K]>
    : T[K];
};

interface Config {
  server: {
    host: string;
    port: number;
  };
  database: {
    connection: string;
  };
}

type PartialConfig = DeepPartial<Config>;
// All nested properties are optional
```

### DeepReadonly

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepReadonly<T[K]>
    : T[K];
};
```

### Paths and PathValue

```typescript
// Get all paths in an object
type Paths<T> = T extends object
  ? {
      [K in keyof T]: K extends string
        ? T[K] extends object
          ? K | `${K}.${Paths<T[K]>}`
          : K
        : never;
    }[keyof T]
  : never;

interface User {
  name: string;
  address: {
    city: string;
    country: string;
  };
}

type UserPaths = Paths<User>;
// 'name' | 'address' | 'address.city' | 'address.country'
```

### Awaited (Deep Promise Unwrap)

```typescript
// Built-in since TS 4.5
type Awaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F): any } ?
    F extends ((value: infer V) => any) ? Awaited<V> : never :
  T;

type A = Awaited<Promise<string>>;           // string
type B = Awaited<Promise<Promise<number>>>; // number
type C = Awaited<string>;                    // string
```

---

## Interview Questions

### Q1: Implement a DeepPartial type

```typescript
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

// Usage
interface Config {
  server: { host: string; port: number };
}

type PartialConfig = DeepPartial<Config>;
// { server?: { host?: string; port?: number } }
```

### Q2: How do conditional types with infer work?

```typescript
// infer declares a type variable that TypeScript infers

// Extract return type
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// Extract array element type
type ArrayElement<T> = T extends (infer E)[] ? E : never;

// Extract promise value
type Awaited<T> = T extends Promise<infer V> ? V : T;

// The inferred type is available in the true branch
```

### Q3: Explain mapped types with key remapping

```typescript
// Mapped types iterate over keys of a type
// Key remapping (as clause) allows transforming keys

type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

// K in keyof T: iterate over all keys
// as `get${...}`: transform the key name
// () => T[K]: transform the value type
```

### Q4: Create a type that makes specific properties required

```typescript
type RequireFields<T, K extends keyof T> = T & Required<Pick<T, K>>;

// Or using mapped types
type RequireFields<T, K extends keyof T> = Omit<T, K> & {
  [P in K]-?: T[P];
};

interface User {
  name?: string;
  email?: string;
  age?: number;
}

type UserWithName = RequireFields<User, 'name'>;
// name is required, others optional
```

### Q5: What's the difference between `extends` in generics vs conditional types?

```typescript
// In generics: constraint (T must be assignable to constraint)
function process<T extends { id: string }>(item: T) {
  return item.id;
}

// In conditional types: type test (is T assignable to test type?)
type IsString<T> = T extends string ? true : false;

// Key difference:
// - Generics: restricts what T can be
// - Conditional: checks what T is and returns different types
```

---

## Key Takeaways

1. **Generics**: Enable reusable, type-safe code
2. **Constraints**: Use `extends` to limit generic types
3. **Utility types**: Partial, Required, Pick, Omit, Record are essential
4. **Conditional types**: Enable type-level programming
5. **infer**: Extract types from complex structures
6. **Mapped types**: Transform object types systematically
7. **Template literals**: Create string-based types
8. **Combine techniques**: Build powerful custom utilities
