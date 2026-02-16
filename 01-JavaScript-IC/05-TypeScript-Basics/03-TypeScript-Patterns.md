# TypeScript Patterns and Best Practices

## Overview

This covers practical TypeScript patterns commonly used in production applications, including type-safe patterns, common pitfalls, and best practices.

---

## Type-Safe Patterns

### Result Type (Error Handling)

```typescript
// Type-safe error handling without exceptions
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return { success: false, error: 'Division by zero' };
  }
  return { success: true, data: a / b };
}

// Usage
const result = divide(10, 2);
if (result.success) {
  console.log(result.data);  // TypeScript knows data exists
} else {
  console.log(result.error); // TypeScript knows error exists
}

// More comprehensive Result type
type Result<T, E = Error> = Ok<T> | Err<E>;

interface Ok<T> {
  readonly ok: true;
  readonly value: T;
}

interface Err<E> {
  readonly ok: false;
  readonly error: E;
}

function ok<T>(value: T): Ok<T> {
  return { ok: true, value };
}

function err<E>(error: E): Err<E> {
  return { ok: false, error };
}
```

### Builder Pattern

```typescript
class RequestBuilder<
  TMethod extends string = never,
  TUrl extends string = never,
  TBody = never
> {
  private config: Partial<{
    method: string;
    url: string;
    body: unknown;
    headers: Record<string, string>;
  }> = {};

  method<M extends string>(method: M): RequestBuilder<M, TUrl, TBody> {
    this.config.method = method;
    return this as any;
  }

  url<U extends string>(url: U): RequestBuilder<TMethod, U, TBody> {
    this.config.url = url;
    return this as any;
  }

  body<B>(body: B): RequestBuilder<TMethod, TUrl, B> {
    this.config.body = body;
    return this as any;
  }

  headers(headers: Record<string, string>): this {
    this.config.headers = headers;
    return this;
  }

  // Only available when all required fields are set
  build(
    this: RequestBuilder<string, string, any>
  ): { method: string; url: string; body?: unknown } {
    return this.config as any;
  }
}

// Usage - type-safe builder
const request = new RequestBuilder()
  .method('POST')
  .url('/api/users')
  .body({ name: 'John' })
  .build();  // Only compiles when method and url are set
```

### State Machine Pattern

```typescript
// Type-safe state machines
type OrderState = 'pending' | 'processing' | 'shipped' | 'delivered' | 'cancelled';

type OrderTransitions = {
  pending: 'processing' | 'cancelled';
  processing: 'shipped' | 'cancelled';
  shipped: 'delivered';
  delivered: never;
  cancelled: never;
};

class Order<S extends OrderState = 'pending'> {
  constructor(
    public readonly id: string,
    public readonly state: S
  ) {}

  transition<NS extends OrderTransitions[S]>(
    newState: NS
  ): Order<NS> {
    return new Order(this.id, newState);
  }
}

// Usage
const order = new Order('123', 'pending');
const processing = order.transition('processing');  // OK
const shipped = processing.transition('shipped');    // OK
// processing.transition('delivered');  // Error! Can't go directly to delivered
```

### Branded Types

```typescript
// Create distinct types from primitives
type Brand<T, B> = T & { __brand: B };

type UserId = Brand<string, 'UserId'>;
type PostId = Brand<string, 'PostId'>;

function createUserId(id: string): UserId {
  return id as UserId;
}

function createPostId(id: string): PostId {
  return id as PostId;
}

function getUser(id: UserId) {
  // ...
}

function getPost(id: PostId) {
  // ...
}

const userId = createUserId('user-123');
const postId = createPostId('post-456');

getUser(userId);  // OK
// getUser(postId);  // Error! PostId is not assignable to UserId

// Numeric branded types
type PositiveNumber = Brand<number, 'Positive'>;
type Percentage = Brand<number, 'Percentage'>;

function createPositive(n: number): PositiveNumber {
  if (n <= 0) throw new Error('Must be positive');
  return n as PositiveNumber;
}

function createPercentage(n: number): Percentage {
  if (n < 0 || n > 100) throw new Error('Must be 0-100');
  return n as Percentage;
}
```

---

## API and Data Patterns

### API Response Types

```typescript
// Generic API response wrapper
interface ApiResponse<T> {
  data: T;
  meta: {
    timestamp: string;
    requestId: string;
  };
}

interface PaginatedResponse<T> extends ApiResponse<T[]> {
  pagination: {
    page: number;
    pageSize: number;
    totalPages: number;
    totalItems: number;
  };
}

// Error response
interface ApiError {
  code: string;
  message: string;
  details?: Record<string, string[]>;
}

// Union type for all possible responses
type ApiResult<T> =
  | { success: true; response: ApiResponse<T> }
  | { success: false; error: ApiError };

// Type-safe fetch wrapper
async function fetchApi<T>(url: string): Promise<ApiResult<T>> {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      const error: ApiError = await response.json();
      return { success: false, error };
    }
    const data: ApiResponse<T> = await response.json();
    return { success: true, response: data };
  } catch (e) {
    return {
      success: false,
      error: { code: 'NETWORK_ERROR', message: 'Network request failed' }
    };
  }
}
```

### Form Types

```typescript
// Form state management
interface FormField<T> {
  value: T;
  error: string | null;
  touched: boolean;
  dirty: boolean;
}

type FormState<T> = {
  [K in keyof T]: FormField<T[K]>;
};

interface UserFormData {
  name: string;
  email: string;
  age: number;
}

type UserFormState = FormState<UserFormData>;
// {
//   name: FormField<string>;
//   email: FormField<string>;
//   age: FormField<number>;
// }

// Form validation
type ValidationRule<T> = (value: T) => string | null;

type FormValidation<T> = {
  [K in keyof T]?: ValidationRule<T[K]>[];
};

const userValidation: FormValidation<UserFormData> = {
  name: [
    (v) => v.length < 2 ? 'Name too short' : null,
    (v) => v.length > 50 ? 'Name too long' : null,
  ],
  email: [
    (v) => !v.includes('@') ? 'Invalid email' : null,
  ],
  age: [
    (v) => v < 0 ? 'Age must be positive' : null,
    (v) => v > 150 ? 'Invalid age' : null,
  ],
};
```

### DTOs and Mappers

```typescript
// Database entity
interface UserEntity {
  id: string;
  name: string;
  email: string;
  password_hash: string;
  created_at: Date;
  updated_at: Date;
  deleted_at: Date | null;
}

// API response DTO
interface UserDto {
  id: string;
  name: string;
  email: string;
  createdAt: string;
}

// Create DTO type
interface CreateUserDto {
  name: string;
  email: string;
  password: string;
}

// Update DTO (partial)
type UpdateUserDto = Partial<Omit<CreateUserDto, 'password'>> & {
  password?: string;
};

// Type-safe mapper
function toUserDto(entity: UserEntity): UserDto {
  return {
    id: entity.id,
    name: entity.name,
    email: entity.email,
    createdAt: entity.created_at.toISOString(),
  };
}
```

---

## React TypeScript Patterns

### Component Props

```typescript
// Basic props
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
  variant?: 'primary' | 'secondary';
}

const Button: React.FC<ButtonProps> = ({
  label,
  onClick,
  disabled = false,
  variant = 'primary'
}) => (
  <button
    onClick={onClick}
    disabled={disabled}
    className={variant}
  >
    {label}
  </button>
);

// Props with children
interface CardProps {
  title: string;
  children: React.ReactNode;
}

// Props extending HTML elements
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

const Input = React.forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, ...props }, ref) => (
    <div>
      <label>{label}</label>
      <input ref={ref} {...props} />
      {error && <span>{error}</span>}
    </div>
  )
);

// Polymorphic components
type PolymorphicProps<E extends React.ElementType> = {
  as?: E;
} & React.ComponentPropsWithoutRef<E>;

function Box<E extends React.ElementType = 'div'>({
  as,
  ...props
}: PolymorphicProps<E>) {
  const Component = as || 'div';
  return <Component {...props} />;
}

// Usage
<Box as="a" href="/link">Link</Box>
<Box as="button" onClick={() => {}}>Button</Box>
```

### Hooks Types

```typescript
// useState with explicit type
const [user, setUser] = useState<User | null>(null);

// useReducer with discriminated unions
type CounterAction =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset'; payload: number };

interface CounterState {
  count: number;
}

function counterReducer(state: CounterState, action: CounterAction): CounterState {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return { count: action.payload };
  }
}

// Custom hook with generics
function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    const item = localStorage.getItem(key);
    return item ? JSON.parse(item) : initialValue;
  });

  const setValue = (value: T | ((val: T) => T)) => {
    const valueToStore = value instanceof Function ? value(storedValue) : value;
    setStoredValue(valueToStore);
    localStorage.setItem(key, JSON.stringify(valueToStore));
  };

  return [storedValue, setValue] as const;
}
```

### Context Types

```typescript
interface AuthContextType {
  user: User | null;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
  isLoading: boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

// Type-safe hook with error if used outside provider
function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}

// Provider component
function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(false);

  const login = async (credentials: Credentials) => {
    setIsLoading(true);
    // ... login logic
    setIsLoading(false);
  };

  const logout = () => {
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, isLoading }}>
      {children}
    </AuthContext.Provider>
  );
}
```

---

## Utility Patterns

### Type-Safe Event Emitter

```typescript
type EventMap = Record<string, any>;

interface TypedEventEmitter<Events extends EventMap> {
  on<K extends keyof Events>(event: K, listener: (payload: Events[K]) => void): void;
  off<K extends keyof Events>(event: K, listener: (payload: Events[K]) => void): void;
  emit<K extends keyof Events>(event: K, payload: Events[K]): void;
}

class EventEmitter<Events extends EventMap> implements TypedEventEmitter<Events> {
  private listeners: { [K in keyof Events]?: ((payload: Events[K]) => void)[] } = {};

  on<K extends keyof Events>(event: K, listener: (payload: Events[K]) => void) {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(listener);
  }

  off<K extends keyof Events>(event: K, listener: (payload: Events[K]) => void) {
    this.listeners[event] = this.listeners[event]?.filter(l => l !== listener);
  }

  emit<K extends keyof Events>(event: K, payload: Events[K]) {
    this.listeners[event]?.forEach(listener => listener(payload));
  }
}

// Usage
interface AppEvents {
  userLogin: { userId: string; timestamp: Date };
  userLogout: { userId: string };
  error: { message: string; code: number };
}

const emitter = new EventEmitter<AppEvents>();

emitter.on('userLogin', ({ userId, timestamp }) => {
  console.log(`User ${userId} logged in at ${timestamp}`);
});

emitter.emit('userLogin', { userId: '123', timestamp: new Date() });
```

### Type-Safe Object.keys and Object.entries

```typescript
// Object.keys returns string[], not keyof T
// Create type-safe versions

function typedKeys<T extends object>(obj: T): (keyof T)[] {
  return Object.keys(obj) as (keyof T)[];
}

function typedEntries<T extends object>(obj: T): [keyof T, T[keyof T]][] {
  return Object.entries(obj) as [keyof T, T[keyof T]][];
}

// Usage
const user = { name: 'John', age: 30 };

Object.keys(user);     // string[]
typedKeys(user);       // ('name' | 'age')[]

for (const [key, value] of typedEntries(user)) {
  console.log(key, value);  // key is 'name' | 'age'
}
```

### Exhaustive Type Checking

```typescript
// Ensure all cases are handled
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}

type Status = 'pending' | 'approved' | 'rejected';

function getStatusColor(status: Status): string {
  switch (status) {
    case 'pending':
      return 'yellow';
    case 'approved':
      return 'green';
    case 'rejected':
      return 'red';
    default:
      return assertNever(status);  // Error if case missing
  }
}

// Adding new status without handling it causes compile error
// type Status = 'pending' | 'approved' | 'rejected' | 'cancelled';
// Now getStatusColor shows error at assertNever
```

---

## Common Pitfalls

### Type Assertions vs Type Guards

```typescript
// Bad - type assertion can be wrong
const user = data as User;  // Might not actually be User

// Good - type guard validates at runtime
function isUser(data: unknown): data is User {
  return (
    typeof data === 'object' &&
    data !== null &&
    'name' in data &&
    'email' in data
  );
}

if (isUser(data)) {
  console.log(data.name);  // Safe
}
```

### Optional vs Undefined

```typescript
// These are different!
interface A {
  x?: number;  // Property may not exist
}

interface B {
  x: number | undefined;  // Property must exist, can be undefined
}

const a: A = {};          // OK
// const b: B = {};       // Error! x is required

const a2: A = { x: undefined };  // OK
const b2: B = { x: undefined };  // OK
```

### Index Signature Pitfalls

```typescript
interface StringMap {
  [key: string]: string;
}

const map: StringMap = { name: 'John' };
const value = map.nonexistent;  // string (not string | undefined!)

// Solution: Enable noUncheckedIndexedAccess in tsconfig
// Or be explicit:
interface SafeStringMap {
  [key: string]: string | undefined;
}
```

### Readonly Arrays and Objects

```typescript
// readonly doesn't prevent deep mutations
interface User {
  name: string;
  address: { city: string };
}

const user: Readonly<User> = {
  name: 'John',
  address: { city: 'NYC' }
};

// user.name = 'Jane';     // Error
user.address.city = 'LA';  // OK! Deep mutation still works

// Use DeepReadonly for full immutability
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

---

## Interview Questions

### Q1: How do you type a function that accepts any object with an id?

```typescript
// Using interface constraint
interface HasId {
  id: string | number;
}

function processItem<T extends HasId>(item: T): T {
  console.log(item.id);
  return item;
}

// Or inline constraint
function processItem<T extends { id: string | number }>(item: T): T {
  console.log(item.id);
  return item;
}
```

### Q2: How do you make TypeScript infer literal types?

```typescript
// Use const assertion
const config = {
  endpoint: '/api',
  method: 'GET'
} as const;
// Type: { readonly endpoint: '/api'; readonly method: 'GET' }

// Or use const type parameter (TS 5.0+)
function createConfig<const T>(config: T): T {
  return config;
}

const config = createConfig({
  endpoint: '/api',
  method: 'GET'
});
// Type: { endpoint: '/api'; method: 'GET' }
```

### Q3: How do you type a higher-order function?

```typescript
// Function that returns a function
function createMultiplier(factor: number) {
  return (value: number): number => value * factor;
}

// Generic HOF
function compose<A, B, C>(
  f: (b: B) => C,
  g: (a: A) => B
): (a: A) => C {
  return (a) => f(g(a));
}

// With explicit types
type Middleware<T> = (value: T, next: () => T) => T;

function applyMiddleware<T>(
  value: T,
  middlewares: Middleware<T>[]
): T {
  // Implementation
}
```

### Q4: What's the difference between `type` and `interface` for function types?

```typescript
// Both work for simple cases
type FnType = (x: number) => string;
interface FnInterface {
  (x: number): string;
}

// Interface can have additional properties
interface FnWithProps {
  (x: number): string;
  defaultValue: number;
}

// Type can use union
type MaybeFn = ((x: number) => string) | null;

// Interface can be augmented
interface Fn { (x: number): string; }
interface Fn { description: string; }  // Merged
```

---

## Key Takeaways

1. **Use discriminated unions** for type-safe state
2. **Branded types** prevent mixing similar primitives
3. **Result types** for explicit error handling
4. **Generic constraints** for flexible yet type-safe functions
5. **Type guards** over type assertions for runtime safety
6. **Exhaustive checking** with never type
7. **as const** for literal type inference
8. **DeepReadonly/DeepPartial** for nested objects
