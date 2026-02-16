# JavaScript Design Patterns

## Overview

Design patterns are reusable solutions to common problems in software design. Understanding patterns helps write maintainable, scalable code and is essential for Lead-level interviews.

---

## Creational Patterns

### Singleton

```javascript
// Ensures only one instance exists

// Classic approach
class Database {
  constructor() {
    if (Database.instance) {
      return Database.instance;
    }

    this.connection = this.connect();
    Database.instance = this;
  }

  connect() {
    console.log('Connecting to database...');
    return { connected: true };
  }

  query(sql) {
    return `Executing: ${sql}`;
  }
}

const db1 = new Database();
const db2 = new Database();
console.log(db1 === db2);  // true

// Module pattern (ES6 modules are singletons)
// database.js
let connection = null;

export function getConnection() {
  if (!connection) {
    connection = createConnection();
  }
  return connection;
}

// Modern approach with closure
const Singleton = (() => {
  let instance;

  function createInstance() {
    return {
      data: [],
      add(item) { this.data.push(item); }
    };
  }

  return {
    getInstance() {
      if (!instance) {
        instance = createInstance();
      }
      return instance;
    }
  };
})();

const s1 = Singleton.getInstance();
const s2 = Singleton.getInstance();
console.log(s1 === s2);  // true
```

### Factory

```javascript
// Creates objects without specifying exact class

// Simple Factory
class UserFactory {
  static create(type, data) {
    switch (type) {
      case 'admin':
        return new AdminUser(data);
      case 'moderator':
        return new ModeratorUser(data);
      default:
        return new RegularUser(data);
    }
  }
}

class RegularUser {
  constructor({ name, email }) {
    this.name = name;
    this.email = email;
    this.permissions = ['read'];
  }
}

class AdminUser {
  constructor({ name, email }) {
    this.name = name;
    this.email = email;
    this.permissions = ['read', 'write', 'delete', 'admin'];
  }
}

const admin = UserFactory.create('admin', { name: 'John', email: 'john@example.com' });
const user = UserFactory.create('regular', { name: 'Jane', email: 'jane@example.com' });

// Abstract Factory
class UIFactory {
  createButton() { throw new Error('Must implement'); }
  createInput() { throw new Error('Must implement'); }
}

class MaterialUIFactory extends UIFactory {
  createButton() { return new MaterialButton(); }
  createInput() { return new MaterialInput(); }
}

class BootstrapUIFactory extends UIFactory {
  createButton() { return new BootstrapButton(); }
  createInput() { return new BootstrapInput(); }
}

function createUI(factory) {
  const button = factory.createButton();
  const input = factory.createInput();
  return { button, input };
}

const materialUI = createUI(new MaterialUIFactory());
```

### Builder

```javascript
// Constructs complex objects step by step

class RequestBuilder {
  constructor() {
    this.request = {
      method: 'GET',
      headers: {},
      body: null
    };
  }

  setUrl(url) {
    this.request.url = url;
    return this;
  }

  setMethod(method) {
    this.request.method = method;
    return this;
  }

  setHeader(key, value) {
    this.request.headers[key] = value;
    return this;
  }

  setBody(body) {
    this.request.body = JSON.stringify(body);
    this.setHeader('Content-Type', 'application/json');
    return this;
  }

  setTimeout(ms) {
    this.request.timeout = ms;
    return this;
  }

  build() {
    if (!this.request.url) {
      throw new Error('URL is required');
    }
    return { ...this.request };
  }
}

// Fluent interface
const request = new RequestBuilder()
  .setUrl('/api/users')
  .setMethod('POST')
  .setHeader('Authorization', 'Bearer token')
  .setBody({ name: 'John' })
  .setTimeout(5000)
  .build();

// Query Builder example
class QueryBuilder {
  constructor(table) {
    this.table = table;
    this.conditions = [];
    this.orderByClause = null;
    this.limitValue = null;
  }

  select(...columns) {
    this.columns = columns;
    return this;
  }

  where(column, operator, value) {
    this.conditions.push({ column, operator, value });
    return this;
  }

  orderBy(column, direction = 'ASC') {
    this.orderByClause = { column, direction };
    return this;
  }

  limit(n) {
    this.limitValue = n;
    return this;
  }

  toSQL() {
    const cols = this.columns?.join(', ') || '*';
    let sql = `SELECT ${cols} FROM ${this.table}`;

    if (this.conditions.length) {
      const where = this.conditions
        .map(c => `${c.column} ${c.operator} '${c.value}'`)
        .join(' AND ');
      sql += ` WHERE ${where}`;
    }

    if (this.orderByClause) {
      sql += ` ORDER BY ${this.orderByClause.column} ${this.orderByClause.direction}`;
    }

    if (this.limitValue) {
      sql += ` LIMIT ${this.limitValue}`;
    }

    return sql;
  }
}

const query = new QueryBuilder('users')
  .select('id', 'name', 'email')
  .where('status', '=', 'active')
  .where('role', '=', 'admin')
  .orderBy('created_at', 'DESC')
  .limit(10)
  .toSQL();
```

---

## Structural Patterns

### Adapter

```javascript
// Converts interface of one class to another

// Old API
class OldPaymentSystem {
  processPayment(amount, cardNumber) {
    return { success: true, transactionId: '123' };
  }
}

// New API expected by app
class PaymentProcessor {
  pay(paymentDetails) {
    throw new Error('Must implement');
  }
}

// Adapter
class PaymentAdapter extends PaymentProcessor {
  constructor() {
    super();
    this.oldSystem = new OldPaymentSystem();
  }

  pay({ amount, card }) {
    const result = this.oldSystem.processPayment(amount, card.number);
    return {
      successful: result.success,
      id: result.transactionId
    };
  }
}

// Usage
const processor = new PaymentAdapter();
processor.pay({ amount: 100, card: { number: '4111111111111111' } });

// Adapter for third-party library
class LocalStorageAdapter {
  get(key) {
    const value = localStorage.getItem(key);
    return value ? JSON.parse(value) : null;
  }

  set(key, value) {
    localStorage.setItem(key, JSON.stringify(value));
  }

  remove(key) {
    localStorage.removeItem(key);
  }
}

class SessionStorageAdapter {
  get(key) {
    const value = sessionStorage.getItem(key);
    return value ? JSON.parse(value) : null;
  }

  set(key, value) {
    sessionStorage.setItem(key, JSON.stringify(value));
  }

  remove(key) {
    sessionStorage.removeItem(key);
  }
}

// Use same interface for different storage
function createCache(adapter) {
  return {
    get: (key) => adapter.get(key),
    set: (key, value) => adapter.set(key, value),
    remove: (key) => adapter.remove(key)
  };
}
```

### Decorator

```javascript
// Adds behavior to objects dynamically

// Function decorator
function withLogging(fn) {
  return function(...args) {
    console.log(`Calling ${fn.name} with`, args);
    const result = fn.apply(this, args);
    console.log(`Result:`, result);
    return result;
  };
}

function add(a, b) {
  return a + b;
}

const loggedAdd = withLogging(add);
loggedAdd(2, 3);

// Class method decorator (TypeScript style)
function log(target, key, descriptor) {
  const original = descriptor.value;

  descriptor.value = function(...args) {
    console.log(`Calling ${key}`);
    return original.apply(this, args);
  };

  return descriptor;
}

// Object decorator
function withTimestamp(obj) {
  return {
    ...obj,
    createdAt: new Date(),
    updatedAt: new Date(),
    touch() {
      this.updatedAt = new Date();
    }
  };
}

const user = withTimestamp({ name: 'John', email: 'john@example.com' });

// Decorator pattern for HTTP client
class HttpClient {
  async request(url, options) {
    const response = await fetch(url, options);
    return response.json();
  }
}

class AuthDecorator {
  constructor(client, getToken) {
    this.client = client;
    this.getToken = getToken;
  }

  async request(url, options = {}) {
    const token = this.getToken();
    return this.client.request(url, {
      ...options,
      headers: {
        ...options.headers,
        Authorization: `Bearer ${token}`
      }
    });
  }
}

class RetryDecorator {
  constructor(client, maxRetries = 3) {
    this.client = client;
    this.maxRetries = maxRetries;
  }

  async request(url, options) {
    let lastError;
    for (let i = 0; i < this.maxRetries; i++) {
      try {
        return await this.client.request(url, options);
      } catch (error) {
        lastError = error;
        await new Promise(r => setTimeout(r, 1000 * (i + 1)));
      }
    }
    throw lastError;
  }
}

// Compose decorators
let client = new HttpClient();
client = new AuthDecorator(client, () => getToken());
client = new RetryDecorator(client, 3);
```

### Facade

```javascript
// Provides simplified interface to complex subsystem

// Complex subsystems
class VideoDecoder {
  decode(file) { return { decoded: true, format: 'raw' }; }
}

class AudioDecoder {
  decode(file) { return { decoded: true, format: 'pcm' }; }
}

class MediaEncoder {
  encode(video, audio, format) {
    return { encoded: true, format };
  }
}

class FileSystem {
  read(path) { return { data: 'binary' }; }
  write(path, data) { return { success: true }; }
}

// Facade
class MediaConverter {
  constructor() {
    this.videoDecoder = new VideoDecoder();
    this.audioDecoder = new AudioDecoder();
    this.encoder = new MediaEncoder();
    this.fs = new FileSystem();
  }

  convert(inputPath, outputPath, format) {
    // Complex operations hidden behind simple interface
    const file = this.fs.read(inputPath);
    const video = this.videoDecoder.decode(file);
    const audio = this.audioDecoder.decode(file);
    const encoded = this.encoder.encode(video, audio, format);
    this.fs.write(outputPath, encoded);

    return { success: true, output: outputPath };
  }
}

// Simple usage
const converter = new MediaConverter();
converter.convert('video.avi', 'video.mp4', 'mp4');

// API Facade
class ApiFacade {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
  }

  async getUsers() {
    const response = await fetch(`${this.baseUrl}/users`);
    return this.handleResponse(response);
  }

  async getUser(id) {
    const response = await fetch(`${this.baseUrl}/users/${id}`);
    return this.handleResponse(response);
  }

  async createUser(data) {
    const response = await fetch(`${this.baseUrl}/users`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    return this.handleResponse(response);
  }

  async handleResponse(response) {
    if (!response.ok) {
      throw new Error(`HTTP error: ${response.status}`);
    }
    return response.json();
  }
}
```

### Proxy

```javascript
// Controls access to another object

// Lazy initialization proxy
class HeavyObject {
  constructor() {
    this.data = this.loadData();  // Expensive operation
  }

  loadData() {
    console.log('Loading heavy data...');
    return new Array(1000000).fill('data');
  }

  process() {
    return this.data.length;
  }
}

class HeavyObjectProxy {
  constructor() {
    this.realObject = null;
  }

  getRealObject() {
    if (!this.realObject) {
      this.realObject = new HeavyObject();
    }
    return this.realObject;
  }

  process() {
    return this.getRealObject().process();
  }
}

// ES6 Proxy for validation
const userValidator = {
  set(obj, prop, value) {
    if (prop === 'age') {
      if (!Number.isInteger(value) || value < 0 || value > 150) {
        throw new Error('Invalid age');
      }
    }
    if (prop === 'email') {
      if (!value.includes('@')) {
        throw new Error('Invalid email');
      }
    }
    obj[prop] = value;
    return true;
  }
};

const user = new Proxy({}, userValidator);
user.email = 'john@example.com';  // OK
// user.age = -5;  // Throws Error

// Caching proxy
function createCachingProxy(target) {
  const cache = new Map();

  return new Proxy(target, {
    apply(target, thisArg, args) {
      const key = JSON.stringify(args);

      if (cache.has(key)) {
        console.log('Cache hit');
        return cache.get(key);
      }

      const result = target.apply(thisArg, args);
      cache.set(key, result);
      return result;
    }
  });
}

const expensiveFunction = (n) => {
  console.log('Computing...');
  return n * 2;
};

const cachedFunction = createCachingProxy(expensiveFunction);
cachedFunction(5);  // Computing... 10
cachedFunction(5);  // Cache hit: 10
```

---

## Behavioral Patterns

### Observer / Pub-Sub

```javascript
// Defines one-to-many dependency between objects

// Observer pattern
class Subject {
  constructor() {
    this.observers = [];
  }

  subscribe(observer) {
    this.observers.push(observer);
    return () => this.unsubscribe(observer);
  }

  unsubscribe(observer) {
    this.observers = this.observers.filter(obs => obs !== observer);
  }

  notify(data) {
    this.observers.forEach(observer => observer(data));
  }
}

// Usage
const subject = new Subject();
const unsubscribe = subject.subscribe(data => console.log('Observer 1:', data));
subject.subscribe(data => console.log('Observer 2:', data));

subject.notify({ message: 'Hello' });
unsubscribe();
subject.notify({ message: 'World' });

// Pub/Sub (decoupled)
class EventBus {
  constructor() {
    this.events = {};
  }

  subscribe(event, callback) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(callback);

    return () => {
      this.events[event] = this.events[event].filter(cb => cb !== callback);
    };
  }

  publish(event, data) {
    if (this.events[event]) {
      this.events[event].forEach(callback => callback(data));
    }
  }

  once(event, callback) {
    const unsubscribe = this.subscribe(event, (data) => {
      callback(data);
      unsubscribe();
    });
  }
}

// Global event bus
const bus = new EventBus();
bus.subscribe('user:login', user => console.log('User logged in:', user));
bus.publish('user:login', { id: 1, name: 'John' });

// React-like state management
function createStore(initialState) {
  let state = initialState;
  const listeners = new Set();

  return {
    getState() {
      return state;
    },
    setState(newState) {
      state = typeof newState === 'function'
        ? newState(state)
        : { ...state, ...newState };
      listeners.forEach(listener => listener(state));
    },
    subscribe(listener) {
      listeners.add(listener);
      return () => listeners.delete(listener);
    }
  };
}
```

### Strategy

```javascript
// Defines family of algorithms, encapsulates each one

// Payment strategies
const paymentStrategies = {
  creditCard: {
    pay(amount) {
      console.log(`Paid ${amount} using Credit Card`);
      return { success: true, method: 'credit_card' };
    },
    validate(details) {
      return details.cardNumber && details.cvv;
    }
  },
  paypal: {
    pay(amount) {
      console.log(`Paid ${amount} using PayPal`);
      return { success: true, method: 'paypal' };
    },
    validate(details) {
      return details.email;
    }
  },
  crypto: {
    pay(amount) {
      console.log(`Paid ${amount} using Cryptocurrency`);
      return { success: true, method: 'crypto' };
    },
    validate(details) {
      return details.walletAddress;
    }
  }
};

class PaymentProcessor {
  constructor(strategy) {
    this.strategy = strategy;
  }

  setStrategy(strategy) {
    this.strategy = strategy;
  }

  checkout(amount, details) {
    if (!this.strategy.validate(details)) {
      throw new Error('Invalid payment details');
    }
    return this.strategy.pay(amount);
  }
}

// Usage
const processor = new PaymentProcessor(paymentStrategies.creditCard);
processor.checkout(100, { cardNumber: '4111...', cvv: '123' });

processor.setStrategy(paymentStrategies.paypal);
processor.checkout(100, { email: 'user@example.com' });

// Validation strategies
const validators = {
  required: (value) => value != null && value !== '',
  email: (value) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value),
  minLength: (min) => (value) => value.length >= min,
  maxLength: (max) => (value) => value.length <= max,
  pattern: (regex) => (value) => regex.test(value)
};

function validate(value, strategies) {
  return strategies.every(strategy => {
    if (typeof strategy === 'function') {
      return strategy(value);
    }
    return validators[strategy](value);
  });
}

validate('test@example.com', ['required', validators.email]);
```

### Command

```javascript
// Encapsulates request as object

class Command {
  execute() { throw new Error('Must implement'); }
  undo() { throw new Error('Must implement'); }
}

class AddTextCommand extends Command {
  constructor(editor, text) {
    super();
    this.editor = editor;
    this.text = text;
  }

  execute() {
    this.previousText = this.editor.content;
    this.editor.content += this.text;
  }

  undo() {
    this.editor.content = this.previousText;
  }
}

class DeleteTextCommand extends Command {
  constructor(editor, count) {
    super();
    this.editor = editor;
    this.count = count;
  }

  execute() {
    this.deletedText = this.editor.content.slice(-this.count);
    this.editor.content = this.editor.content.slice(0, -this.count);
  }

  undo() {
    this.editor.content += this.deletedText;
  }
}

class Editor {
  constructor() {
    this.content = '';
    this.history = [];
    this.redoStack = [];
  }

  executeCommand(command) {
    command.execute();
    this.history.push(command);
    this.redoStack = [];  // Clear redo stack
  }

  undo() {
    const command = this.history.pop();
    if (command) {
      command.undo();
      this.redoStack.push(command);
    }
  }

  redo() {
    const command = this.redoStack.pop();
    if (command) {
      command.execute();
      this.history.push(command);
    }
  }
}

// Usage
const editor = new Editor();
editor.executeCommand(new AddTextCommand(editor, 'Hello '));
editor.executeCommand(new AddTextCommand(editor, 'World'));
console.log(editor.content);  // "Hello World"

editor.undo();
console.log(editor.content);  // "Hello "

editor.redo();
console.log(editor.content);  // "Hello World"
```

### State

```javascript
// Allows object to alter behavior when state changes

class TrafficLight {
  constructor() {
    this.states = {
      green: new GreenState(this),
      yellow: new YellowState(this),
      red: new RedState(this)
    };
    this.currentState = this.states.green;
  }

  setState(state) {
    this.currentState = this.states[state];
  }

  change() {
    this.currentState.change();
  }

  getColor() {
    return this.currentState.color;
  }
}

class GreenState {
  constructor(light) {
    this.light = light;
    this.color = 'green';
  }

  change() {
    console.log('Green -> Yellow');
    this.light.setState('yellow');
  }
}

class YellowState {
  constructor(light) {
    this.light = light;
    this.color = 'yellow';
  }

  change() {
    console.log('Yellow -> Red');
    this.light.setState('red');
  }
}

class RedState {
  constructor(light) {
    this.light = light;
    this.color = 'red';
  }

  change() {
    console.log('Red -> Green');
    this.light.setState('green');
  }
}

// Usage
const light = new TrafficLight();
console.log(light.getColor());  // green
light.change();  // Green -> Yellow
light.change();  // Yellow -> Red
light.change();  // Red -> Green
```

---

## Interview Questions

### Q1: When would you use Factory vs Builder pattern?

```javascript
// Factory: Create objects of different types with same interface
// - When you need to create objects based on a condition
// - When object creation is complex but follows patterns

// Builder: Construct complex objects step by step
// - When object has many optional parameters
// - When you want fluent/chainable construction
// - When construction involves multiple steps
```

### Q2: Explain the Observer pattern

```javascript
// Observer defines one-to-many relationship
// When subject changes, all observers are notified

// Use cases:
// - Event systems
// - State management (Redux)
// - Real-time updates
// - UI data binding
```

### Q3: What's the difference between Decorator and Proxy?

```javascript
// Decorator: Adds new behavior/responsibilities
// - Enhances functionality
// - Can stack multiple decorators

// Proxy: Controls access to object
// - Same interface as original
// - Lazy loading, caching, access control, logging
```

---

## Key Takeaways

1. **Singleton**: One instance, global access
2. **Factory**: Create objects without specifying class
3. **Builder**: Step-by-step complex object construction
4. **Adapter**: Convert interfaces
5. **Decorator**: Add behavior dynamically
6. **Facade**: Simplify complex subsystems
7. **Proxy**: Control object access
8. **Observer**: One-to-many event notification
9. **Strategy**: Interchangeable algorithms
10. **Command**: Encapsulate actions for undo/redo
