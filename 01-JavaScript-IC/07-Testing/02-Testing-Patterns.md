# Testing Patterns and Strategies

## Overview

This covers advanced testing patterns, strategies for different scenarios, and best practices for maintaining a healthy test suite.

---

## Component Testing Patterns

### Testing Props and State

```javascript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

// Testing props
describe('Button', () => {
  test('renders with correct label', () => {
    render(<Button label="Click me" />);
    expect(screen.getByRole('button')).toHaveTextContent('Click me');
  });

  test('applies variant styles', () => {
    render(<Button variant="primary" label="Primary" />);
    expect(screen.getByRole('button')).toHaveClass('btn-primary');
  });

  test('is disabled when disabled prop is true', () => {
    render(<Button disabled label="Disabled" />);
    expect(screen.getByRole('button')).toBeDisabled();
  });

  test('calls onClick when clicked', async () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick} label="Click" />);

    await userEvent.click(screen.getByRole('button'));

    expect(handleClick).toHaveBeenCalledTimes(1);
  });
});

// Testing state changes
describe('Counter', () => {
  test('increments count when button clicked', async () => {
    render(<Counter initialCount={0} />);

    expect(screen.getByText('Count: 0')).toBeInTheDocument();

    await userEvent.click(screen.getByRole('button', { name: 'Increment' }));

    expect(screen.getByText('Count: 1')).toBeInTheDocument();
  });
});
```

### Testing Forms

```javascript
describe('LoginForm', () => {
  test('submits form with entered values', async () => {
    const onSubmit = jest.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    await userEvent.type(
      screen.getByLabelText(/email/i),
      'user@example.com'
    );
    await userEvent.type(
      screen.getByLabelText(/password/i),
      'password123'
    );
    await userEvent.click(screen.getByRole('button', { name: /login/i }));

    expect(onSubmit).toHaveBeenCalledWith({
      email: 'user@example.com',
      password: 'password123'
    });
  });

  test('shows validation errors for invalid input', async () => {
    render(<LoginForm onSubmit={jest.fn()} />);

    await userEvent.click(screen.getByRole('button', { name: /login/i }));

    expect(screen.getByText(/email is required/i)).toBeInTheDocument();
    expect(screen.getByText(/password is required/i)).toBeInTheDocument();
  });

  test('disables submit button while submitting', async () => {
    const onSubmit = jest.fn(() => new Promise(r => setTimeout(r, 100)));
    render(<LoginForm onSubmit={onSubmit} />);

    await userEvent.type(screen.getByLabelText(/email/i), 'user@example.com');
    await userEvent.type(screen.getByLabelText(/password/i), 'password123');

    const submitButton = screen.getByRole('button', { name: /login/i });
    await userEvent.click(submitButton);

    expect(submitButton).toBeDisabled();
  });
});
```

### Testing Conditional Rendering

```javascript
describe('UserProfile', () => {
  test('shows loading state initially', () => {
    render(<UserProfile userId="1" />);
    expect(screen.getByText(/loading/i)).toBeInTheDocument();
  });

  test('shows user data when loaded', async () => {
    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });

    expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
  });

  test('shows error state on failure', async () => {
    server.use(
      rest.get('/api/users/:id', (req, res, ctx) => {
        return res(ctx.status(500));
      })
    );

    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText(/error/i)).toBeInTheDocument();
    });
  });

  test('shows different content for admin users', async () => {
    render(<UserProfile userId="admin-1" />);

    await waitFor(() => {
      expect(screen.getByText('Admin Dashboard')).toBeInTheDocument();
    });
  });
});
```

---

## API Mocking with MSW

### Setup

```javascript
// src/mocks/handlers.js
import { rest } from 'msw';

export const handlers = [
  rest.get('/api/users', (req, res, ctx) => {
    return res(
      ctx.json([
        { id: 1, name: 'John' },
        { id: 2, name: 'Jane' }
      ])
    );
  }),

  rest.get('/api/users/:id', (req, res, ctx) => {
    const { id } = req.params;
    return res(
      ctx.json({ id, name: 'John Doe' })
    );
  }),

  rest.post('/api/users', async (req, res, ctx) => {
    const body = await req.json();
    return res(
      ctx.status(201),
      ctx.json({ id: '3', ...body })
    );
  }),

  rest.delete('/api/users/:id', (req, res, ctx) => {
    return res(ctx.status(204));
  })
];

// src/mocks/server.js
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// jest.setup.js
import { server } from './mocks/server';

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### Using in Tests

```javascript
import { server } from '../mocks/server';
import { rest } from 'msw';

describe('UserList', () => {
  test('renders users from API', async () => {
    render(<UserList />);

    await waitFor(() => {
      expect(screen.getByText('John')).toBeInTheDocument();
      expect(screen.getByText('Jane')).toBeInTheDocument();
    });
  });

  test('handles error response', async () => {
    // Override handler for this test
    server.use(
      rest.get('/api/users', (req, res, ctx) => {
        return res(ctx.status(500), ctx.json({ message: 'Server error' }));
      })
    );

    render(<UserList />);

    await waitFor(() => {
      expect(screen.getByText(/error/i)).toBeInTheDocument();
    });
  });

  test('handles empty response', async () => {
    server.use(
      rest.get('/api/users', (req, res, ctx) => {
        return res(ctx.json([]));
      })
    );

    render(<UserList />);

    await waitFor(() => {
      expect(screen.getByText('No users found')).toBeInTheDocument();
    });
  });

  test('handles network error', async () => {
    server.use(
      rest.get('/api/users', (req, res, ctx) => {
        return res.networkError('Failed to connect');
      })
    );

    render(<UserList />);

    await waitFor(() => {
      expect(screen.getByText(/network error/i)).toBeInTheDocument();
    });
  });
});
```

---

## Testing Async Patterns

### Testing Data Fetching

```javascript
describe('useUsers hook', () => {
  test('fetches and returns users', async () => {
    const { result } = renderHook(() => useUsers());

    expect(result.current.loading).toBe(true);

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.users).toHaveLength(2);
    expect(result.current.error).toBeNull();
  });

  test('handles fetch error', async () => {
    server.use(
      rest.get('/api/users', (req, res, ctx) => {
        return res(ctx.status(500));
      })
    );

    const { result } = renderHook(() => useUsers());

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.error).toBeTruthy();
    expect(result.current.users).toEqual([]);
  });
});
```

### Testing Debounced Functions

```javascript
describe('SearchInput', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  test('debounces search requests', async () => {
    const onSearch = jest.fn();
    render(<SearchInput onSearch={onSearch} debounceMs={300} />);

    const input = screen.getByRole('textbox');

    // Type quickly
    await userEvent.type(input, 'test', { delay: null });

    // Search not called yet
    expect(onSearch).not.toHaveBeenCalled();

    // Fast-forward past debounce time
    act(() => {
      jest.advanceTimersByTime(300);
    });

    // Now search is called with final value
    expect(onSearch).toHaveBeenCalledTimes(1);
    expect(onSearch).toHaveBeenCalledWith('test');
  });
});
```

### Testing Polling

```javascript
describe('StatusPoller', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  test('polls for status updates', async () => {
    let pollCount = 0;
    server.use(
      rest.get('/api/status', (req, res, ctx) => {
        pollCount++;
        return res(ctx.json({ status: pollCount < 3 ? 'pending' : 'complete' }));
      })
    );

    render(<StatusPoller interval={1000} />);

    // Initial fetch
    await waitFor(() => {
      expect(screen.getByText('pending')).toBeInTheDocument();
    });

    // Advance timer for next poll
    act(() => {
      jest.advanceTimersByTime(1000);
    });

    // Still pending
    await waitFor(() => {
      expect(screen.getByText('pending')).toBeInTheDocument();
    });

    // Advance timer again
    act(() => {
      jest.advanceTimersByTime(1000);
    });

    // Now complete
    await waitFor(() => {
      expect(screen.getByText('complete')).toBeInTheDocument();
    });
  });
});
```

---

## Testing Error Boundaries

```javascript
// Suppress console.error for error boundary tests
const originalError = console.error;

beforeAll(() => {
  console.error = jest.fn();
});

afterAll(() => {
  console.error = originalError;
});

describe('ErrorBoundary', () => {
  const ThrowError = () => {
    throw new Error('Test error');
  };

  test('renders fallback UI when child throws', () => {
    render(
      <ErrorBoundary fallback={<div>Something went wrong</div>}>
        <ThrowError />
      </ErrorBoundary>
    );

    expect(screen.getByText('Something went wrong')).toBeInTheDocument();
  });

  test('renders children when no error', () => {
    render(
      <ErrorBoundary fallback={<div>Something went wrong</div>}>
        <div>Normal content</div>
      </ErrorBoundary>
    );

    expect(screen.getByText('Normal content')).toBeInTheDocument();
    expect(screen.queryByText('Something went wrong')).not.toBeInTheDocument();
  });

  test('calls onError callback', () => {
    const onError = jest.fn();

    render(
      <ErrorBoundary onError={onError} fallback={<div>Error</div>}>
        <ThrowError />
      </ErrorBoundary>
    );

    expect(onError).toHaveBeenCalled();
  });
});
```

---

## Testing Redux

### Testing Reducers

```javascript
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },
    decrement: (state) => { state.value -= 1; },
    incrementByAmount: (state, action) => { state.value += action.payload; }
  }
});

describe('counterSlice', () => {
  test('should return initial state', () => {
    expect(counterSlice.reducer(undefined, { type: 'unknown' })).toEqual({
      value: 0
    });
  });

  test('should handle increment', () => {
    const actual = counterSlice.reducer({ value: 0 }, increment());
    expect(actual.value).toEqual(1);
  });

  test('should handle decrement', () => {
    const actual = counterSlice.reducer({ value: 5 }, decrement());
    expect(actual.value).toEqual(4);
  });

  test('should handle incrementByAmount', () => {
    const actual = counterSlice.reducer({ value: 0 }, incrementByAmount(10));
    expect(actual.value).toEqual(10);
  });
});
```

### Testing Components with Redux

```javascript
import { configureStore } from '@reduxjs/toolkit';
import { Provider } from 'react-redux';

const renderWithRedux = (
  ui,
  {
    preloadedState = {},
    store = configureStore({
      reducer: { counter: counterReducer },
      preloadedState
    }),
    ...renderOptions
  } = {}
) => {
  const Wrapper = ({ children }) => (
    <Provider store={store}>{children}</Provider>
  );

  return { store, ...render(ui, { wrapper: Wrapper, ...renderOptions }) };
};

describe('Counter component', () => {
  test('renders with initial state', () => {
    renderWithRedux(<Counter />, {
      preloadedState: { counter: { value: 5 } }
    });

    expect(screen.getByText('5')).toBeInTheDocument();
  });

  test('increments value on button click', async () => {
    renderWithRedux(<Counter />);

    await userEvent.click(screen.getByRole('button', { name: /increment/i }));

    expect(screen.getByText('1')).toBeInTheDocument();
  });
});
```

### Testing Async Thunks

```javascript
import { createAsyncThunk } from '@reduxjs/toolkit';

const fetchUser = createAsyncThunk(
  'users/fetchById',
  async (userId, { rejectWithValue }) => {
    try {
      const response = await api.fetchUser(userId);
      return response.data;
    } catch (err) {
      return rejectWithValue(err.response.data);
    }
  }
);

describe('fetchUser thunk', () => {
  test('dispatches fulfilled on success', async () => {
    const store = configureStore({
      reducer: { users: usersReducer }
    });

    await store.dispatch(fetchUser('1'));

    const state = store.getState();
    expect(state.users.currentUser).toEqual({ id: '1', name: 'John' });
    expect(state.users.status).toBe('succeeded');
  });

  test('dispatches rejected on failure', async () => {
    server.use(
      rest.get('/api/users/:id', (req, res, ctx) => {
        return res(ctx.status(404), ctx.json({ message: 'Not found' }));
      })
    );

    const store = configureStore({
      reducer: { users: usersReducer }
    });

    await store.dispatch(fetchUser('999'));

    const state = store.getState();
    expect(state.users.status).toBe('failed');
    expect(state.users.error).toBe('Not found');
  });
});
```

---

## Snapshot Testing

```javascript
// Basic snapshot
test('renders correctly', () => {
  const tree = render(<Button label="Click me" />);
  expect(tree).toMatchSnapshot();
});

// Inline snapshot
test('renders correctly', () => {
  const { container } = render(<Button label="Click me" />);
  expect(container.firstChild).toMatchInlineSnapshot(`
    <button
      class="btn"
    >
      Click me
    </button>
  `);
});

// When to use snapshots:
// - Detecting unintended changes
// - Quick coverage for stable components
// - Output verification (CLI tools, serialized data)

// When NOT to use:
// - Testing behavior
// - Large/complex output
// - Frequently changing components
```

---

## Test Organization

### File Structure

```
src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx
│   │   └── Button.stories.tsx
│   └── Form/
│       ├── Form.tsx
│       ├── Form.test.tsx
│       └── __snapshots__/
├── hooks/
│   ├── useAuth.ts
│   └── useAuth.test.ts
├── utils/
│   ├── formatters.ts
│   └── formatters.test.ts
└── __tests__/
    └── integration/
        └── checkout.test.tsx
```

### Test Naming Conventions

```javascript
// Describe the unit being tested
describe('Button', () => {
  // Describe the scenario/context
  describe('when disabled', () => {
    // Describe expected behavior
    it('should not call onClick handler', () => {});
    it('should have disabled attribute', () => {});
    it('should have reduced opacity', () => {});
  });

  describe('when loading', () => {
    it('should show spinner', () => {});
    it('should disable click', () => {});
  });
});

// Or use "should" pattern
describe('formatCurrency', () => {
  it('should format positive numbers with dollar sign', () => {});
  it('should handle negative numbers', () => {});
  it('should round to two decimal places', () => {});
});
```

---

## Interview Questions

### Q1: How do you decide what to test?

```javascript
// Focus on:
// 1. User interactions and flows
// 2. Business logic
// 3. Edge cases and error states
// 4. Integration points

// Don't test:
// - Implementation details
// - Third-party libraries
// - Constants/static values
// - Getters/setters without logic
```

### Q2: How do you handle flaky tests?

```javascript
// Common causes and solutions:
// 1. Timing issues - Use proper async patterns (waitFor, findBy)
// 2. Shared state - Isolate tests, reset between tests
// 3. External dependencies - Mock consistently
// 4. Race conditions - Ensure deterministic order
// 5. Date/time - Mock Date, use fake timers
```

### Q3: What's the difference between mocking and stubbing?

```javascript
// Stub: Provides canned answers to calls
const stub = jest.fn().mockReturnValue(42);

// Mock: Stubs + ability to verify calls
const mock = jest.fn();
mock(1, 2);
expect(mock).toHaveBeenCalledWith(1, 2);

// Spy: Wraps real implementation, can track calls
const spy = jest.spyOn(object, 'method');
```

---

## Key Takeaways

1. **Test behavior, not implementation** - Focus on what users see and do
2. **Use the right queries** - Prefer accessible queries (role, label)
3. **Mock at the right level** - Prefer MSW for API mocking
4. **Keep tests isolated** - Each test should be independent
5. **Make tests readable** - Clear names, AAA pattern
6. **Don't over-test** - Focus on value, not coverage numbers
7. **Maintain test quality** - Refactor tests like production code
