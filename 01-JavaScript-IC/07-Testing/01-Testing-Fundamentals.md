# Testing Fundamentals

## Overview

Testing is essential for building reliable applications. Understanding testing principles, patterns, and tools is crucial for Lead-level interviews.

---

## Testing Pyramid

```
         /\
        /  \      E2E Tests (few)
       /----\     - Slow, expensive
      /      \    - Test full user flows
     /--------\
    /          \  Integration Tests (some)
   /------------\ - Test component interactions
  /              \
 /----------------\ Unit Tests (many)
                   - Fast, isolated
                   - Test single units
```

### Types of Tests

```javascript
// Unit Test - Tests single function/component in isolation
test('adds two numbers', () => {
  expect(add(2, 3)).toBe(5);
});

// Integration Test - Tests multiple units working together
test('user service creates and retrieves user', async () => {
  const user = await userService.create({ name: 'John' });
  const retrieved = await userService.findById(user.id);
  expect(retrieved.name).toBe('John');
});

// E2E Test - Tests complete user flow
test('user can login and see dashboard', async () => {
  await page.goto('/login');
  await page.fill('#email', 'user@example.com');
  await page.fill('#password', 'password');
  await page.click('button[type="submit"]');
  await expect(page).toHaveURL('/dashboard');
});
```

---

## Jest Basics

### Setup and Configuration

```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'jsdom',  // or 'node'
  setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '\\.(css|less|scss)$': 'identity-obj-proxy'
  },
  collectCoverageFrom: [
    'src/**/*.{js,jsx,ts,tsx}',
    '!src/**/*.d.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80
    }
  }
};

// jest.setup.js
import '@testing-library/jest-dom';
```

### Basic Test Structure

```javascript
// describe - group related tests
describe('Calculator', () => {
  // beforeAll - run once before all tests
  beforeAll(() => {
    console.log('Starting calculator tests');
  });

  // beforeEach - run before each test
  beforeEach(() => {
    // Reset state
  });

  // afterEach - run after each test
  afterEach(() => {
    // Cleanup
  });

  // afterAll - run once after all tests
  afterAll(() => {
    console.log('Finished calculator tests');
  });

  // test or it - individual test
  test('adds numbers correctly', () => {
    expect(add(1, 2)).toBe(3);
  });

  it('subtracts numbers correctly', () => {
    expect(subtract(5, 3)).toBe(2);
  });

  // Nested describe
  describe('division', () => {
    test('divides numbers', () => {
      expect(divide(10, 2)).toBe(5);
    });

    test('throws on divide by zero', () => {
      expect(() => divide(10, 0)).toThrow('Division by zero');
    });
  });
});
```

### Matchers

```javascript
// Equality
expect(value).toBe(2);              // Strict equality (===)
expect(value).toEqual({ a: 1 });    // Deep equality
expect(value).toStrictEqual(obj);   // Deep equality + type checking

// Truthiness
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeDefined();

// Numbers
expect(value).toBeGreaterThan(3);
expect(value).toBeGreaterThanOrEqual(3);
expect(value).toBeLessThan(5);
expect(value).toBeCloseTo(0.3, 5);  // Floating point

// Strings
expect(string).toMatch(/pattern/);
expect(string).toContain('substring');
expect(string).toHaveLength(5);

// Arrays
expect(array).toContain(item);
expect(array).toContainEqual({ a: 1 });
expect(array).toHaveLength(3);

// Objects
expect(object).toHaveProperty('key');
expect(object).toHaveProperty('key', 'value');
expect(object).toMatchObject({ a: 1 });

// Exceptions
expect(() => fn()).toThrow();
expect(() => fn()).toThrow('error message');
expect(() => fn()).toThrow(ErrorClass);

// Async
await expect(asyncFn()).resolves.toBe(value);
await expect(asyncFn()).rejects.toThrow('error');

// Negation
expect(value).not.toBe(5);
expect(array).not.toContain(item);

// Snapshot
expect(component).toMatchSnapshot();
expect(component).toMatchInlineSnapshot(`"<div>Hello</div>"`);
```

---

## Mocking

### Function Mocks

```javascript
// Create mock function
const mockFn = jest.fn();

// With implementation
const mockAdd = jest.fn((a, b) => a + b);

// With return value
const mockFn = jest.fn().mockReturnValue(42);
const mockFn = jest.fn().mockReturnValueOnce(1).mockReturnValueOnce(2);

// Async mocks
const mockAsync = jest.fn().mockResolvedValue({ data: 'result' });
const mockAsyncError = jest.fn().mockRejectedValue(new Error('failed'));

// Mock implementation
mockFn.mockImplementation((x) => x * 2);
mockFn.mockImplementationOnce((x) => x * 3);

// Assertions on mocks
expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(3);
expect(mockFn).toHaveBeenCalledWith(arg1, arg2);
expect(mockFn).toHaveBeenLastCalledWith(arg);
expect(mockFn).toHaveBeenNthCalledWith(1, arg);
expect(mockFn).toHaveReturnedWith(value);

// Access call information
mockFn.mock.calls;        // [[arg1, arg2], [arg1, arg2]]
mockFn.mock.results;      // [{ type: 'return', value: 42 }]
mockFn.mock.instances;    // [this context]

// Clear/reset
mockFn.mockClear();       // Clear calls and results
mockFn.mockReset();       // Clear + reset implementation
mockFn.mockRestore();     // Restore original (for spies)
```

### Module Mocks

```javascript
// Mock entire module
jest.mock('./api');

// With implementation
jest.mock('./api', () => ({
  fetchUser: jest.fn().mockResolvedValue({ name: 'John' }),
  saveUser: jest.fn().mockResolvedValue({ success: true })
}));

// Partial mock (keep some real implementations)
jest.mock('./utils', () => ({
  ...jest.requireActual('./utils'),
  formatDate: jest.fn().mockReturnValue('2024-01-01')
}));

// Mock node modules
jest.mock('axios');
import axios from 'axios';
axios.get.mockResolvedValue({ data: { users: [] } });

// Manual mocks (__mocks__/moduleName.js)
// src/__mocks__/api.js
export const fetchUser = jest.fn();
export const saveUser = jest.fn();
```

### Spies

```javascript
// Spy on object method
const spy = jest.spyOn(object, 'method');

// Spy with mock implementation
jest.spyOn(object, 'method').mockImplementation(() => 'mocked');

// Spy on module
import * as utils from './utils';
jest.spyOn(utils, 'formatDate').mockReturnValue('2024-01-01');

// Restore original
spy.mockRestore();

// Spy on console
const consoleSpy = jest.spyOn(console, 'log').mockImplementation();
// ... test
expect(consoleSpy).toHaveBeenCalledWith('expected message');
consoleSpy.mockRestore();
```

### Timer Mocks

```javascript
// Enable fake timers
jest.useFakeTimers();

test('debounce function', () => {
  const callback = jest.fn();
  const debounced = debounce(callback, 1000);

  debounced();
  debounced();
  debounced();

  expect(callback).not.toHaveBeenCalled();

  // Fast-forward time
  jest.advanceTimersByTime(1000);

  expect(callback).toHaveBeenCalledTimes(1);
});

// Run all timers
jest.runAllTimers();

// Run only pending timers
jest.runOnlyPendingTimers();

// Advance to next timer
jest.advanceTimersToNextTimer();

// Get current time
jest.now();

// Set system time
jest.setSystemTime(new Date('2024-01-01'));

// Restore real timers
jest.useRealTimers();
```

---

## Testing Async Code

### Promises

```javascript
// Return promise
test('fetches user', () => {
  return fetchUser(1).then(user => {
    expect(user.name).toBe('John');
  });
});

// Async/await
test('fetches user', async () => {
  const user = await fetchUser(1);
  expect(user.name).toBe('John');
});

// Resolves/rejects matchers
test('fetches user', async () => {
  await expect(fetchUser(1)).resolves.toEqual({ name: 'John' });
});

test('handles error', async () => {
  await expect(fetchUser(-1)).rejects.toThrow('Not found');
});
```

### Callbacks

```javascript
// Using done callback
test('calls callback with data', (done) => {
  fetchData((error, data) => {
    try {
      expect(error).toBeNull();
      expect(data).toBe('result');
      done();
    } catch (e) {
      done(e);
    }
  });
});
```

### Waiting for Elements

```javascript
// waitFor - wait for assertion to pass
import { waitFor } from '@testing-library/react';

test('shows loading then data', async () => {
  render(<DataComponent />);

  expect(screen.getByText('Loading...')).toBeInTheDocument();

  await waitFor(() => {
    expect(screen.getByText('Data loaded')).toBeInTheDocument();
  });
});

// findBy - queries that wait
const element = await screen.findByText('Data loaded');
const elements = await screen.findAllByRole('listitem');
```

---

## React Testing Library

### Rendering Components

```javascript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('renders component', () => {
  render(<Button label="Click me" />);

  expect(screen.getByRole('button')).toHaveTextContent('Click me');
});

// With providers
const renderWithProviders = (ui, options = {}) => {
  const Wrapper = ({ children }) => (
    <ThemeProvider>
      <AuthProvider>
        {children}
      </AuthProvider>
    </ThemeProvider>
  );

  return render(ui, { wrapper: Wrapper, ...options });
};

test('renders with context', () => {
  renderWithProviders(<UserProfile />);
  expect(screen.getByText('John')).toBeInTheDocument();
});
```

### Queries

```javascript
// Query types:
// getBy    - throws if not found (sync)
// queryBy  - returns null if not found (sync)
// findBy   - returns promise, waits (async)
// getAllBy - returns array, throws if empty
// queryAllBy - returns array (empty if none)
// findAllBy - returns promise of array

// By role (preferred)
screen.getByRole('button');
screen.getByRole('button', { name: 'Submit' });
screen.getByRole('textbox', { name: /email/i });
screen.getByRole('heading', { level: 1 });

// By label text (forms)
screen.getByLabelText('Email');
screen.getByLabelText(/email/i);

// By placeholder
screen.getByPlaceholderText('Enter email');

// By text content
screen.getByText('Hello World');
screen.getByText(/hello/i);

// By display value (inputs)
screen.getByDisplayValue('current value');

// By alt text (images)
screen.getByAltText('Profile picture');

// By title
screen.getByTitle('Close');

// By test id (last resort)
screen.getByTestId('custom-element');
```

### User Interactions

```javascript
import userEvent from '@testing-library/user-event';

test('form submission', async () => {
  const user = userEvent.setup();
  const onSubmit = jest.fn();

  render(<LoginForm onSubmit={onSubmit} />);

  // Type in inputs
  await user.type(screen.getByLabelText('Email'), 'test@example.com');
  await user.type(screen.getByLabelText('Password'), 'password123');

  // Click button
  await user.click(screen.getByRole('button', { name: 'Login' }));

  expect(onSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'password123'
  });
});

// Other interactions
await user.clear(input);              // Clear input
await user.selectOptions(select, ['option1', 'option2']);
await user.deselectOptions(select, 'option1');
await user.upload(fileInput, file);
await user.hover(element);
await user.unhover(element);
await user.tab();                     // Tab to next element
await user.keyboard('{Enter}');       // Press key
await user.keyboard('Hello');         // Type text
await user.pointer('[MouseLeft]');    // Mouse click
await user.copy();                    // Copy selected
await user.paste();                   // Paste clipboard
```

### Testing Hooks

```javascript
import { renderHook, act } from '@testing-library/react';

test('useCounter hook', () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => {
    result.current.increment();
  });

  expect(result.current.count).toBe(1);
});

// With wrapper
const { result } = renderHook(() => useAuth(), {
  wrapper: ({ children }) => (
    <AuthProvider>{children}</AuthProvider>
  )
});

// Rerender with new props
const { result, rerender } = renderHook(
  ({ initialCount }) => useCounter(initialCount),
  { initialProps: { initialCount: 0 } }
);

rerender({ initialCount: 10 });
```

---

## Test Patterns

### Arrange-Act-Assert (AAA)

```javascript
test('user can add item to cart', () => {
  // Arrange - Set up test data and conditions
  const cart = new ShoppingCart();
  const item = { id: 1, name: 'Widget', price: 10 };

  // Act - Perform the action being tested
  cart.addItem(item);

  // Assert - Verify the expected outcome
  expect(cart.items).toHaveLength(1);
  expect(cart.total).toBe(10);
});
```

### Given-When-Then

```javascript
describe('Shopping Cart', () => {
  describe('given an empty cart', () => {
    describe('when adding an item', () => {
      it('then cart should contain the item', () => {
        const cart = new ShoppingCart();
        cart.addItem({ id: 1, price: 10 });
        expect(cart.items).toHaveLength(1);
      });

      it('then total should equal item price', () => {
        const cart = new ShoppingCart();
        cart.addItem({ id: 1, price: 10 });
        expect(cart.total).toBe(10);
      });
    });
  });
});
```

### Test Data Builders

```javascript
// Builder pattern for test data
class UserBuilder {
  constructor() {
    this.user = {
      id: '1',
      name: 'John Doe',
      email: 'john@example.com',
      role: 'user'
    };
  }

  withId(id) {
    this.user.id = id;
    return this;
  }

  withName(name) {
    this.user.name = name;
    return this;
  }

  withRole(role) {
    this.user.role = role;
    return this;
  }

  asAdmin() {
    this.user.role = 'admin';
    return this;
  }

  build() {
    return { ...this.user };
  }
}

// Usage
const user = new UserBuilder().withName('Jane').asAdmin().build();

// Factory function alternative
const createUser = (overrides = {}) => ({
  id: '1',
  name: 'John Doe',
  email: 'john@example.com',
  role: 'user',
  ...overrides
});

const admin = createUser({ role: 'admin', name: 'Admin User' });
```

---

## Interview Questions

### Q1: What's the difference between unit, integration, and E2E tests?

```javascript
// Unit: Test single function/component in isolation
// - Fast, many tests, mock dependencies
// - Example: Testing a pure function or single component

// Integration: Test multiple units working together
// - Medium speed, test real interactions
// - Example: Testing API route with database

// E2E: Test complete user flows
// - Slow, few tests, real environment
// - Example: Testing login flow in browser
```

### Q2: When should you use mocks?

```javascript
// Use mocks for:
// - External services (APIs, databases)
// - Non-deterministic code (Date, Math.random)
// - Slow operations
// - Side effects (console.log, localStorage)

// Avoid mocking:
// - Implementation details
// - Everything (over-mocking)
// - When integration test is more valuable
```

### Q3: How do you test async code?

```javascript
// 1. Return promise
test('async test', () => {
  return fetchData().then(data => expect(data).toBe('result'));
});

// 2. Async/await
test('async test', async () => {
  const data = await fetchData();
  expect(data).toBe('result');
});

// 3. resolves/rejects
test('async test', async () => {
  await expect(fetchData()).resolves.toBe('result');
});
```

### Q4: What makes a good test?

```javascript
// Good tests are:
// - Fast: Run quickly
// - Isolated: Don't depend on other tests
// - Repeatable: Same result every time
// - Self-validating: Clear pass/fail
// - Timely: Written with the code

// Follow FIRST principles:
// Fast, Isolated, Repeatable, Self-validating, Timely
```

---

## Key Takeaways

1. **Testing Pyramid**: Many unit tests, some integration, few E2E
2. **Jest matchers**: Know common matchers and when to use each
3. **Mocking**: Mock external dependencies, not implementation details
4. **React Testing Library**: Query by role/accessibility, test behavior not implementation
5. **Async testing**: Use async/await, waitFor, findBy queries
6. **Test patterns**: AAA pattern, test data builders
7. **Good tests**: Fast, isolated, repeatable, clear assertions
