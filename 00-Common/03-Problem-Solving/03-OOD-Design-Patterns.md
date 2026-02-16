# Object-Oriented Design & Design Patterns

## Overview

Lead engineers are expected to design clean, maintainable, and extensible code. This covers SOLID principles and the most commonly asked design patterns in interviews.

---

## SOLID Principles

### 1. Single Responsibility Principle (SRP)

> A class should have only one reason to change.

**Bad:**
```python
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

    def save_to_database(self):
        # Database logic mixed with user entity
        pass

    def send_email(self):
        # Email logic mixed with user entity
        pass
```

**Good:**
```python
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

class UserRepository:
    def save(self, user):
        # Database logic isolated
        pass

class EmailService:
    def send(self, user, message):
        # Email logic isolated
        pass
```

---

### 2. Open/Closed Principle (OCP)

> Open for extension, closed for modification.

**Bad:**
```python
class PaymentProcessor:
    def process(self, payment_type, amount):
        if payment_type == "credit_card":
            # Credit card logic
            pass
        elif payment_type == "paypal":
            # PayPal logic
            pass
        # Adding new payment = modifying this class
```

**Good:**
```python
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    @abstractmethod
    def process(self, amount):
        pass

class CreditCardPayment(PaymentMethod):
    def process(self, amount):
        print(f"Processing ${amount} via credit card")

class PayPalPayment(PaymentMethod):
    def process(self, amount):
        print(f"Processing ${amount} via PayPal")

# Adding new payment = new class, no modification needed
class CryptoPayment(PaymentMethod):
    def process(self, amount):
        print(f"Processing ${amount} via crypto")

class PaymentProcessor:
    def process(self, payment_method: PaymentMethod, amount):
        payment_method.process(amount)
```

**JavaScript:**
```javascript
// Using interfaces via duck typing
class PaymentMethod {
  process(amount) {
    throw new Error("Must implement process()");
  }
}

class CreditCardPayment extends PaymentMethod {
  process(amount) {
    console.log(`Processing $${amount} via credit card`);
  }
}

class PayPalPayment extends PaymentMethod {
  process(amount) {
    console.log(`Processing $${amount} via PayPal`);
  }
}

class PaymentProcessor {
  process(paymentMethod, amount) {
    paymentMethod.process(amount);
  }
}
```

---

### 3. Liskov Substitution Principle (LSP)

> Subtypes must be substitutable for their base types.

**Bad:**
```python
class Bird:
    def fly(self):
        print("Flying")

class Penguin(Bird):
    def fly(self):
        raise Exception("Penguins can't fly!")  # Violates LSP
```

**Good:**
```python
class Bird:
    def move(self):
        pass

class FlyingBird(Bird):
    def move(self):
        print("Flying")

class SwimmingBird(Bird):
    def move(self):
        print("Swimming")

class Sparrow(FlyingBird):
    pass

class Penguin(SwimmingBird):
    pass
```

---

### 4. Interface Segregation Principle (ISP)

> Clients shouldn't depend on interfaces they don't use.

**Bad:**
```python
class Worker(ABC):
    @abstractmethod
    def work(self): pass

    @abstractmethod
    def eat(self): pass

    @abstractmethod
    def sleep(self): pass

class Robot(Worker):
    def work(self):
        print("Working")

    def eat(self):
        pass  # Robots don't eat - forced to implement

    def sleep(self):
        pass  # Robots don't sleep - forced to implement
```

**Good:**
```python
class Workable(ABC):
    @abstractmethod
    def work(self): pass

class Eatable(ABC):
    @abstractmethod
    def eat(self): pass

class Human(Workable, Eatable):
    def work(self):
        print("Working")

    def eat(self):
        print("Eating")

class Robot(Workable):
    def work(self):
        print("Working")
```

---

### 5. Dependency Inversion Principle (DIP)

> Depend on abstractions, not concretions.

**Bad:**
```python
class MySQLDatabase:
    def query(self, sql):
        pass

class UserService:
    def __init__(self):
        self.db = MySQLDatabase()  # Tight coupling to MySQL
```

**Good:**
```python
class Database(ABC):
    @abstractmethod
    def query(self, sql): pass

class MySQLDatabase(Database):
    def query(self, sql):
        print(f"MySQL: {sql}")

class PostgresDatabase(Database):
    def query(self, sql):
        print(f"Postgres: {sql}")

class UserService:
    def __init__(self, database: Database):  # Depends on abstraction
        self.db = database

# Dependency injection
service = UserService(MySQLDatabase())
service = UserService(PostgresDatabase())  # Easy to switch
```

**JavaScript:**
```javascript
// Dependency Injection in JS
class Database {
  query(sql) {
    throw new Error("Must implement query()");
  }
}

class MySQLDatabase extends Database {
  query(sql) {
    console.log(`MySQL: ${sql}`);
  }
}

class PostgresDatabase extends Database {
  query(sql) {
    console.log(`Postgres: ${sql}`);
  }
}

class UserService {
  constructor(database) {
    this.db = database; // Injected dependency
  }

  getUser(id) {
    return this.db.query(`SELECT * FROM users WHERE id = ${id}`);
  }
}

// Usage
const service = new UserService(new MySQLDatabase());
```

---

## Creational Patterns

### Singleton

> Ensure a class has only one instance with global access.

**Use cases:** Configuration, logging, connection pools, caches

**Python:**
```python
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Thread-safe version
import threading

class ThreadSafeSingleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:  # Double-check
                    cls._instance = super().__new__(cls)
        return cls._instance

# Pythonic way - module-level
# config.py
class _Config:
    def __init__(self):
        self.settings = {}

config = _Config()  # Module instance acts as singleton
```

**JavaScript:**
```javascript
// Using closure
const Singleton = (function() {
  let instance;

  function createInstance() {
    return {
      settings: {},
      getSetting(key) {
        return this.settings[key];
      },
      setSetting(key, value) {
        this.settings[key] = value;
      }
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

// Usage
const config1 = Singleton.getInstance();
const config2 = Singleton.getInstance();
console.log(config1 === config2); // true

// ES6 Module (simpler - modules are singletons)
// config.js
class Config {
  constructor() {
    this.settings = {};
  }
}
export const config = new Config();
```

---

### Factory

> Create objects without specifying exact class.

**Use cases:** Creating objects based on input, abstracting object creation

**Python:**
```python
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def speak(self): pass

class Dog(Animal):
    def speak(self):
        return "Woof!"

class Cat(Animal):
    def speak(self):
        return "Meow!"

class AnimalFactory:
    @staticmethod
    def create(animal_type: str) -> Animal:
        animals = {
            "dog": Dog,
            "cat": Cat
        }
        if animal_type not in animals:
            raise ValueError(f"Unknown animal: {animal_type}")
        return animals[animal_type]()

# Usage
dog = AnimalFactory.create("dog")
print(dog.speak())  # Woof!
```

**JavaScript:**
```javascript
class Dog {
  speak() {
    return "Woof!";
  }
}

class Cat {
  speak() {
    return "Meow!";
  }
}

class AnimalFactory {
  static create(animalType) {
    const animals = {
      dog: Dog,
      cat: Cat
    };

    const AnimalClass = animals[animalType];
    if (!AnimalClass) {
      throw new Error(`Unknown animal: ${animalType}`);
    }
    return new AnimalClass();
  }
}

// Usage
const dog = AnimalFactory.create("dog");
console.log(dog.speak()); // Woof!
```

---

### Builder

> Construct complex objects step by step.

**Use cases:** Objects with many optional parameters, immutable objects

**Python:**
```python
class User:
    def __init__(self, builder):
        self.name = builder.name
        self.email = builder.email
        self.age = builder.age
        self.phone = builder.phone
        self.address = builder.address

    def __str__(self):
        return f"User({self.name}, {self.email})"

class UserBuilder:
    def __init__(self, name, email):  # Required params
        self.name = name
        self.email = email
        self.age = None
        self.phone = None
        self.address = None

    def with_age(self, age):
        self.age = age
        return self

    def with_phone(self, phone):
        self.phone = phone
        return self

    def with_address(self, address):
        self.address = address
        return self

    def build(self):
        return User(self)

# Usage
user = (UserBuilder("John", "john@email.com")
        .with_age(30)
        .with_phone("123-456-7890")
        .build())
```

**JavaScript:**
```javascript
class User {
  constructor(builder) {
    this.name = builder.name;
    this.email = builder.email;
    this.age = builder.age;
    this.phone = builder.phone;
    this.address = builder.address;
  }
}

class UserBuilder {
  constructor(name, email) {
    this.name = name;
    this.email = email;
    this.age = null;
    this.phone = null;
    this.address = null;
  }

  withAge(age) {
    this.age = age;
    return this;
  }

  withPhone(phone) {
    this.phone = phone;
    return this;
  }

  withAddress(address) {
    this.address = address;
    return this;
  }

  build() {
    return new User(this);
  }
}

// Usage
const user = new UserBuilder("John", "john@email.com")
  .withAge(30)
  .withPhone("123-456-7890")
  .build();
```

---

## Structural Patterns

### Adapter

> Convert interface of a class to another interface clients expect.

**Use cases:** Legacy code integration, third-party library wrapping

**Python:**
```python
# Legacy payment system with different interface
class LegacyPaymentSystem:
    def make_payment(self, amount_in_cents):
        print(f"Legacy payment: {amount_in_cents} cents")

# New interface expected by our app
class PaymentGateway(ABC):
    @abstractmethod
    def pay(self, amount_in_dollars): pass

# Adapter
class LegacyPaymentAdapter(PaymentGateway):
    def __init__(self, legacy_system):
        self.legacy = legacy_system

    def pay(self, amount_in_dollars):
        amount_in_cents = int(amount_in_dollars * 100)
        self.legacy.make_payment(amount_in_cents)

# Usage
legacy = LegacyPaymentSystem()
adapter = LegacyPaymentAdapter(legacy)
adapter.pay(10.50)  # Works with our dollar-based interface
```

**JavaScript:**
```javascript
// Legacy system
class LegacyPaymentSystem {
  makePayment(amountInCents) {
    console.log(`Legacy payment: ${amountInCents} cents`);
  }
}

// Adapter
class LegacyPaymentAdapter {
  constructor(legacySystem) {
    this.legacy = legacySystem;
  }

  pay(amountInDollars) {
    const amountInCents = Math.round(amountInDollars * 100);
    this.legacy.makePayment(amountInCents);
  }
}

// Usage
const legacy = new LegacyPaymentSystem();
const adapter = new LegacyPaymentAdapter(legacy);
adapter.pay(10.50);
```

---

### Decorator

> Add behavior to objects dynamically without altering their structure.

**Use cases:** Adding features at runtime, cross-cutting concerns (logging, caching)

**Python:**
```python
# Function decorator (common in Python)
def log_calls(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished {func.__name__}")
        return result
    return wrapper

@log_calls
def process_data(data):
    return data.upper()

# Class-based decorator pattern
class Coffee(ABC):
    @abstractmethod
    def cost(self): pass

    @abstractmethod
    def description(self): pass

class SimpleCoffee(Coffee):
    def cost(self):
        return 2.0

    def description(self):
        return "Simple coffee"

class CoffeeDecorator(Coffee):
    def __init__(self, coffee):
        self._coffee = coffee

    def cost(self):
        return self._coffee.cost()

    def description(self):
        return self._coffee.description()

class MilkDecorator(CoffeeDecorator):
    def cost(self):
        return self._coffee.cost() + 0.5

    def description(self):
        return self._coffee.description() + ", milk"

class SugarDecorator(CoffeeDecorator):
    def cost(self):
        return self._coffee.cost() + 0.25

    def description(self):
        return self._coffee.description() + ", sugar"

# Usage
coffee = SimpleCoffee()
coffee = MilkDecorator(coffee)
coffee = SugarDecorator(coffee)
print(f"{coffee.description()}: ${coffee.cost()}")
# Simple coffee, milk, sugar: $2.75
```

**JavaScript:**
```javascript
// Function decorator
function logCalls(fn) {
  return function(...args) {
    console.log(`Calling ${fn.name}`);
    const result = fn.apply(this, args);
    console.log(`Finished ${fn.name}`);
    return result;
  };
}

const processData = logCalls(function processData(data) {
  return data.toUpperCase();
});

// Class-based decorator
class SimpleCoffee {
  cost() {
    return 2.0;
  }

  description() {
    return "Simple coffee";
  }
}

class MilkDecorator {
  constructor(coffee) {
    this.coffee = coffee;
  }

  cost() {
    return this.coffee.cost() + 0.5;
  }

  description() {
    return this.coffee.description() + ", milk";
  }
}

class SugarDecorator {
  constructor(coffee) {
    this.coffee = coffee;
  }

  cost() {
    return this.coffee.cost() + 0.25;
  }

  description() {
    return this.coffee.description() + ", sugar";
  }
}

// Usage
let coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);
console.log(`${coffee.description()}: $${coffee.cost()}`);
```

---

## Behavioral Patterns

### Strategy

> Define a family of algorithms, encapsulate each one, and make them interchangeable.

**Use cases:** Payment methods, sorting algorithms, validation strategies

**Python:**
```python
from abc import ABC, abstractmethod

class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data): pass

class BubbleSort(SortStrategy):
    def sort(self, data):
        print("Bubble sorting")
        return sorted(data)  # Simplified

class QuickSort(SortStrategy):
    def sort(self, data):
        print("Quick sorting")
        return sorted(data)  # Simplified

class MergeSort(SortStrategy):
    def sort(self, data):
        print("Merge sorting")
        return sorted(data)  # Simplified

class Sorter:
    def __init__(self, strategy: SortStrategy):
        self._strategy = strategy

    def set_strategy(self, strategy: SortStrategy):
        self._strategy = strategy

    def sort(self, data):
        return self._strategy.sort(data)

# Usage
sorter = Sorter(QuickSort())
sorter.sort([3, 1, 4, 1, 5])

sorter.set_strategy(MergeSort())
sorter.sort([3, 1, 4, 1, 5])
```

**JavaScript:**
```javascript
class BubbleSort {
  sort(data) {
    console.log("Bubble sorting");
    return [...data].sort((a, b) => a - b);
  }
}

class QuickSort {
  sort(data) {
    console.log("Quick sorting");
    return [...data].sort((a, b) => a - b);
  }
}

class Sorter {
  constructor(strategy) {
    this.strategy = strategy;
  }

  setStrategy(strategy) {
    this.strategy = strategy;
  }

  sort(data) {
    return this.strategy.sort(data);
  }
}

// Usage
const sorter = new Sorter(new QuickSort());
sorter.sort([3, 1, 4, 1, 5]);

sorter.setStrategy(new BubbleSort());
sorter.sort([3, 1, 4, 1, 5]);
```

---

### Observer

> Define one-to-many dependency so when one object changes state, all dependents are notified.

**Use cases:** Event systems, pub/sub, UI updates

**Python:**
```python
from abc import ABC, abstractmethod

class Observer(ABC):
    @abstractmethod
    def update(self, message): pass

class Subject:
    def __init__(self):
        self._observers = []

    def attach(self, observer: Observer):
        self._observers.append(observer)

    def detach(self, observer: Observer):
        self._observers.remove(observer)

    def notify(self, message):
        for observer in self._observers:
            observer.update(message)

class EmailSubscriber(Observer):
    def __init__(self, email):
        self.email = email

    def update(self, message):
        print(f"Email to {self.email}: {message}")

class SMSSubscriber(Observer):
    def __init__(self, phone):
        self.phone = phone

    def update(self, message):
        print(f"SMS to {self.phone}: {message}")

class NewsPublisher(Subject):
    def publish(self, news):
        print(f"Publishing: {news}")
        self.notify(news)

# Usage
publisher = NewsPublisher()
publisher.attach(EmailSubscriber("user@email.com"))
publisher.attach(SMSSubscriber("123-456-7890"))
publisher.publish("Breaking news!")
```

**JavaScript:**
```javascript
class Subject {
  constructor() {
    this.observers = [];
  }

  attach(observer) {
    this.observers.push(observer);
  }

  detach(observer) {
    this.observers = this.observers.filter(o => o !== observer);
  }

  notify(message) {
    this.observers.forEach(observer => observer.update(message));
  }
}

class EmailSubscriber {
  constructor(email) {
    this.email = email;
  }

  update(message) {
    console.log(`Email to ${this.email}: ${message}`);
  }
}

class SMSSubscriber {
  constructor(phone) {
    this.phone = phone;
  }

  update(message) {
    console.log(`SMS to ${this.phone}: ${message}`);
  }
}

class NewsPublisher extends Subject {
  publish(news) {
    console.log(`Publishing: ${news}`);
    this.notify(news);
  }
}

// Usage
const publisher = new NewsPublisher();
publisher.attach(new EmailSubscriber("user@email.com"));
publisher.attach(new SMSSubscriber("123-456-7890"));
publisher.publish("Breaking news!");

// Modern JS: EventEmitter / CustomEvent
class EventEmitter {
  constructor() {
    this.events = {};
  }

  on(event, listener) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(listener);
  }

  off(event, listener) {
    if (this.events[event]) {
      this.events[event] = this.events[event].filter(l => l !== listener);
    }
  }

  emit(event, data) {
    if (this.events[event]) {
      this.events[event].forEach(listener => listener(data));
    }
  }
}
```

---

### Command

> Encapsulate a request as an object, allowing parameterization and queuing.

**Use cases:** Undo/redo, task queues, macro recording

**Python:**
```python
from abc import ABC, abstractmethod

class Command(ABC):
    @abstractmethod
    def execute(self): pass

    @abstractmethod
    def undo(self): pass

class TextEditor:
    def __init__(self):
        self.text = ""

    def write(self, text):
        self.text += text

    def delete(self, count):
        deleted = self.text[-count:]
        self.text = self.text[:-count]
        return deleted

class WriteCommand(Command):
    def __init__(self, editor, text):
        self.editor = editor
        self.text = text

    def execute(self):
        self.editor.write(self.text)

    def undo(self):
        self.editor.delete(len(self.text))

class CommandManager:
    def __init__(self):
        self.history = []
        self.redo_stack = []

    def execute(self, command):
        command.execute()
        self.history.append(command)
        self.redo_stack.clear()

    def undo(self):
        if self.history:
            command = self.history.pop()
            command.undo()
            self.redo_stack.append(command)

    def redo(self):
        if self.redo_stack:
            command = self.redo_stack.pop()
            command.execute()
            self.history.append(command)

# Usage
editor = TextEditor()
manager = CommandManager()

manager.execute(WriteCommand(editor, "Hello "))
manager.execute(WriteCommand(editor, "World"))
print(editor.text)  # Hello World

manager.undo()
print(editor.text)  # Hello

manager.redo()
print(editor.text)  # Hello World
```

**JavaScript:**
```javascript
class TextEditor {
  constructor() {
    this.text = "";
  }

  write(text) {
    this.text += text;
  }

  delete(count) {
    const deleted = this.text.slice(-count);
    this.text = this.text.slice(0, -count);
    return deleted;
  }
}

class WriteCommand {
  constructor(editor, text) {
    this.editor = editor;
    this.text = text;
  }

  execute() {
    this.editor.write(this.text);
  }

  undo() {
    this.editor.delete(this.text.length);
  }
}

class CommandManager {
  constructor() {
    this.history = [];
    this.redoStack = [];
  }

  execute(command) {
    command.execute();
    this.history.push(command);
    this.redoStack = [];
  }

  undo() {
    if (this.history.length > 0) {
      const command = this.history.pop();
      command.undo();
      this.redoStack.push(command);
    }
  }

  redo() {
    if (this.redoStack.length > 0) {
      const command = this.redoStack.pop();
      command.execute();
      this.history.push(command);
    }
  }
}

// Usage
const editor = new TextEditor();
const manager = new CommandManager();

manager.execute(new WriteCommand(editor, "Hello "));
manager.execute(new WriteCommand(editor, "World"));
console.log(editor.text); // Hello World

manager.undo();
console.log(editor.text); // Hello

manager.redo();
console.log(editor.text); // Hello World
```

---

## Common Interview Design Problems

### 1. Design a Parking Lot

```python
from enum import Enum
from abc import ABC, abstractmethod

class VehicleSize(Enum):
    MOTORCYCLE = 1
    COMPACT = 2
    LARGE = 3

class Vehicle(ABC):
    def __init__(self, license_plate):
        self.license_plate = license_plate

    @abstractmethod
    def get_size(self): pass

class Motorcycle(Vehicle):
    def get_size(self):
        return VehicleSize.MOTORCYCLE

class Car(Vehicle):
    def get_size(self):
        return VehicleSize.COMPACT

class Truck(Vehicle):
    def get_size(self):
        return VehicleSize.LARGE

class ParkingSpot:
    def __init__(self, spot_id, size):
        self.spot_id = spot_id
        self.size = size
        self.vehicle = None

    def can_fit(self, vehicle):
        return self.vehicle is None and vehicle.get_size().value <= self.size.value

    def park(self, vehicle):
        if self.can_fit(vehicle):
            self.vehicle = vehicle
            return True
        return False

    def remove(self):
        self.vehicle = None

class ParkingLot:
    def __init__(self):
        self.spots = []
        self.vehicle_to_spot = {}

    def add_spot(self, spot):
        self.spots.append(spot)

    def park(self, vehicle):
        for spot in self.spots:
            if spot.park(vehicle):
                self.vehicle_to_spot[vehicle.license_plate] = spot
                return spot.spot_id
        return None  # No available spot

    def leave(self, license_plate):
        if license_plate in self.vehicle_to_spot:
            spot = self.vehicle_to_spot.pop(license_plate)
            spot.remove()
            return True
        return False
```

---

### 2. Design a Rate Limiter

```python
import time
from collections import deque

class RateLimiter:
    """Sliding window rate limiter"""
    def __init__(self, max_requests, window_seconds):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = {}  # user_id -> deque of timestamps

    def is_allowed(self, user_id):
        now = time.time()

        if user_id not in self.requests:
            self.requests[user_id] = deque()

        user_requests = self.requests[user_id]

        # Remove old requests outside window
        while user_requests and user_requests[0] <= now - self.window_seconds:
            user_requests.popleft()

        # Check if under limit
        if len(user_requests) < self.max_requests:
            user_requests.append(now)
            return True

        return False

# Token bucket implementation
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.tokens = capacity
        self.last_refill = time.time()

    def is_allowed(self):
        self._refill()

        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False

    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        new_tokens = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now
```

---

## Quick Reference

| Pattern | Purpose | When to Use |
|---------|---------|-------------|
| **Singleton** | Single instance | Config, logging, pools |
| **Factory** | Object creation | Unknown types at compile time |
| **Builder** | Complex construction | Many optional params |
| **Adapter** | Interface conversion | Legacy integration |
| **Decorator** | Add behavior | Features without subclassing |
| **Strategy** | Interchangeable algorithms | Multiple algorithms |
| **Observer** | Event notification | Pub/sub systems |
| **Command** | Encapsulate actions | Undo/redo, queues |

---

## Interview Tips

1. **Start with SOLID** - Mention which principles apply
2. **Explain trade-offs** - Every pattern has downsides
3. **Keep it simple** - Don't over-engineer
4. **Use real examples** - Show you've used patterns in practice
5. **Consider testability** - How does the pattern affect testing?
