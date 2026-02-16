# Error Handling Patterns

## Overview

Proper error handling is crucial for building robust applications. This covers error types, handling strategies, and patterns for managing errors effectively.

---

## JavaScript Error Types

### Built-in Error Types

```javascript
// Base Error
const error = new Error('Something went wrong');
error.name;     // "Error"
error.message;  // "Something went wrong"
error.stack;    // Stack trace

// Specific error types
try {
  // SyntaxError - Invalid syntax (usually at parse time)
  eval('function(');

  // ReferenceError - Invalid reference
  console.log(undeclaredVariable);

  // TypeError - Operation on wrong type
  null.toString();
  const obj = {}; obj.method();

  // RangeError - Value out of range
  new Array(-1);
  (1234).toFixed(200);

  // URIError - Invalid URI
  decodeURIComponent('%');

  // EvalError - Error with eval (legacy)

} catch (error) {
  console.log(error.name, error.message);
}

// Check error type
if (error instanceof TypeError) {
  console.log('Type error occurred');
}
```

### Custom Error Classes

```javascript
// Custom error with additional properties
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;

    // Maintains proper stack trace (V8)
    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, ValidationError);
    }
  }
}

class HttpError extends Error {
  constructor(message, statusCode, responseBody = null) {
    super(message);
    this.name = 'HttpError';
    this.statusCode = statusCode;
    this.responseBody = responseBody;
  }

  get isClientError() {
    return this.statusCode >= 400 && this.statusCode < 500;
  }

  get isServerError() {
    return this.statusCode >= 500;
  }
}

class NotFoundError extends HttpError {
  constructor(resource) {
    super(`${resource} not found`, 404);
    this.name = 'NotFoundError';
    this.resource = resource;
  }
}

class AuthenticationError extends Error {
  constructor(message = 'Authentication required') {
    super(message);
    this.name = 'AuthenticationError';
  }
}

// Usage
function validateUser(data) {
  if (!data.email) {
    throw new ValidationError('Email is required', 'email');
  }
  if (!data.email.includes('@')) {
    throw new ValidationError('Invalid email format', 'email');
  }
}

try {
  validateUser({ email: 'invalid' });
} catch (error) {
  if (error instanceof ValidationError) {
    console.log(`Validation error on ${error.field}: ${error.message}`);
  }
}
```

---

## Try-Catch Patterns

### Basic Pattern

```javascript
// Synchronous error handling
function parseJSON(jsonString) {
  try {
    return JSON.parse(jsonString);
  } catch (error) {
    console.error('Failed to parse JSON:', error.message);
    return null;
  }
}

// With finally
function readFile(path) {
  let handle;
  try {
    handle = openFile(path);
    return handle.read();
  } catch (error) {
    console.error('Failed to read file:', error);
    throw error;  // Re-throw after logging
  } finally {
    // Always runs, even if error is thrown
    if (handle) {
      handle.close();
    }
  }
}

// Selective catching
function processData(data) {
  try {
    return transform(data);
  } catch (error) {
    if (error instanceof ValidationError) {
      // Handle validation errors
      return { error: error.message, field: error.field };
    }
    // Re-throw unexpected errors
    throw error;
  }
}
```

### Async Error Handling

```javascript
// Async/await with try-catch
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);

    if (!response.ok) {
      throw new HttpError('Failed to fetch user', response.status);
    }

    return await response.json();
  } catch (error) {
    if (error instanceof HttpError) {
      console.error(`HTTP ${error.statusCode}: ${error.message}`);
    } else {
      console.error('Network error:', error.message);
    }
    throw error;
  }
}

// Multiple async operations
async function processUserData(userId) {
  try {
    const user = await fetchUser(userId);
    const orders = await fetchOrders(userId);
    const preferences = await fetchPreferences(userId);

    return { user, orders, preferences };
  } catch (error) {
    // Handles errors from any of the above
    console.error('Failed to process user data:', error);
    throw error;
  }
}

// Parallel operations with individual error handling
async function fetchAllData(userIds) {
  const results = await Promise.allSettled(
    userIds.map(id => fetchUser(id))
  );

  return results.map((result, index) => {
    if (result.status === 'fulfilled') {
      return { success: true, data: result.value };
    } else {
      return { success: false, error: result.reason, userId: userIds[index] };
    }
  });
}
```

### Promise Error Handling

```javascript
// .catch() method
fetchUser(1)
  .then(user => processUser(user))
  .then(result => saveResult(result))
  .catch(error => {
    console.error('Error in chain:', error);
  });

// Error handling at each step
fetchUser(1)
  .catch(error => {
    console.warn('Failed to fetch, using cached data');
    return getCachedUser(1);
  })
  .then(user => processUser(user))
  .catch(error => {
    console.error('Failed to process user');
    return null;
  });

// Finally
fetchUser(1)
  .then(user => processUser(user))
  .catch(error => handleError(error))
  .finally(() => {
    // Cleanup, runs regardless of success/failure
    hideLoadingSpinner();
  });
```

---

## Result Type Pattern

```javascript
// Type-safe error handling without exceptions
class Result {
  constructor(value, error) {
    this.value = value;
    this.error = error;
  }

  static ok(value) {
    return new Result(value, null);
  }

  static err(error) {
    return new Result(null, error);
  }

  isOk() {
    return this.error === null;
  }

  isErr() {
    return this.error !== null;
  }

  map(fn) {
    if (this.isErr()) return this;
    return Result.ok(fn(this.value));
  }

  mapErr(fn) {
    if (this.isOk()) return this;
    return Result.err(fn(this.error));
  }

  unwrap() {
    if (this.isErr()) {
      throw this.error;
    }
    return this.value;
  }

  unwrapOr(defaultValue) {
    return this.isOk() ? this.value : defaultValue;
  }

  match({ ok, err }) {
    return this.isOk() ? ok(this.value) : err(this.error);
  }
}

// Usage
function divide(a, b) {
  if (b === 0) {
    return Result.err(new Error('Division by zero'));
  }
  return Result.ok(a / b);
}

const result = divide(10, 2);

if (result.isOk()) {
  console.log('Result:', result.value);
} else {
  console.log('Error:', result.error.message);
}

// Chaining
const finalResult = divide(10, 2)
  .map(x => x * 2)
  .map(x => x + 1)
  .match({
    ok: value => `Success: ${value}`,
    err: error => `Failed: ${error.message}`
  });

// Async Result
async function fetchUserResult(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) {
      return Result.err(new HttpError('Not found', response.status));
    }
    const data = await response.json();
    return Result.ok(data);
  } catch (error) {
    return Result.err(error);
  }
}
```

---

## Global Error Handling

### Browser Global Handlers

```javascript
// Catch unhandled errors
window.onerror = function(message, source, lineno, colno, error) {
  console.error('Global error:', { message, source, lineno, colno, error });

  // Send to error tracking service
  reportError({
    message,
    source,
    lineno,
    colno,
    stack: error?.stack
  });

  // Return true to prevent default browser error handling
  return true;
};

// Modern alternative
window.addEventListener('error', (event) => {
  console.error('Error event:', event.error);
  event.preventDefault();  // Prevent default handling
});

// Unhandled promise rejections
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled rejection:', event.reason);

  reportError({
    type: 'unhandled_rejection',
    reason: event.reason
  });

  event.preventDefault();
});

// Resource loading errors (images, scripts)
window.addEventListener('error', (event) => {
  if (event.target !== window) {
    console.error('Resource failed to load:', event.target.src || event.target.href);
  }
}, true);  // Capture phase to catch resource errors
```

### React Error Boundaries

```javascript
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log error to service
    console.error('Error caught by boundary:', error);
    console.error('Component stack:', errorInfo.componentStack);

    reportError({
      error,
      componentStack: errorInfo.componentStack
    });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="error-fallback">
          <h2>Something went wrong</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
function App() {
  return (
    <ErrorBoundary fallback={<ErrorPage />}>
      <Routes />
    </ErrorBoundary>
  );
}

// Functional component with error boundary hook
function useErrorBoundary() {
  const [error, setError] = useState(null);

  const resetError = () => setError(null);

  const captureError = useCallback((error) => {
    setError(error);
    reportError(error);
  }, []);

  if (error) {
    throw error;  // Re-throw to nearest error boundary
  }

  return { captureError, resetError };
}
```

---

## API Error Handling

### HTTP Error Handling

```javascript
// Centralized API error handling
class ApiClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
  }

  async request(endpoint, options = {}) {
    const url = `${this.baseUrl}${endpoint}`;

    try {
      const response = await fetch(url, {
        ...options,
        headers: {
          'Content-Type': 'application/json',
          ...options.headers
        }
      });

      // Handle HTTP errors
      if (!response.ok) {
        const errorBody = await this.parseErrorBody(response);
        throw new HttpError(
          errorBody.message || `HTTP ${response.status}`,
          response.status,
          errorBody
        );
      }

      return await response.json();

    } catch (error) {
      // Handle network errors
      if (error instanceof TypeError) {
        throw new NetworkError('Network request failed');
      }

      // Re-throw HttpError
      if (error instanceof HttpError) {
        throw error;
      }

      // Wrap unknown errors
      throw new ApiError('Request failed', { cause: error });
    }
  }

  async parseErrorBody(response) {
    try {
      return await response.json();
    } catch {
      return { message: response.statusText };
    }
  }

  // Convenience methods
  get(endpoint) {
    return this.request(endpoint);
  }

  post(endpoint, data) {
    return this.request(endpoint, {
      method: 'POST',
      body: JSON.stringify(data)
    });
  }
}

// Usage with error handling
const api = new ApiClient('/api');

async function createUser(userData) {
  try {
    return await api.post('/users', userData);
  } catch (error) {
    if (error instanceof HttpError) {
      switch (error.statusCode) {
        case 400:
          throw new ValidationError(error.responseBody.message);
        case 401:
          throw new AuthenticationError();
        case 404:
          throw new NotFoundError('User');
        case 409:
          throw new ConflictError('User already exists');
        default:
          throw error;
      }
    }
    throw error;
  }
}
```

### Retry Logic

```javascript
async function withRetry(fn, options = {}) {
  const {
    maxRetries = 3,
    delay = 1000,
    backoff = 2,
    retryOn = (error) => true
  } = options;

  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      // Check if we should retry this error
      if (!retryOn(error) || attempt === maxRetries) {
        throw error;
      }

      // Wait before retrying
      const waitTime = delay * Math.pow(backoff, attempt);
      await new Promise(resolve => setTimeout(resolve, waitTime));
    }
  }

  throw lastError;
}

// Usage
const result = await withRetry(
  () => api.get('/users'),
  {
    maxRetries: 3,
    delay: 1000,
    retryOn: (error) => {
      // Only retry on network errors or 5xx
      return error instanceof NetworkError ||
             (error instanceof HttpError && error.isServerError);
    }
  }
);
```

---

## Error Logging and Reporting

```javascript
// Error reporting service
class ErrorReporter {
  constructor(options = {}) {
    this.endpoint = options.endpoint;
    this.environment = options.environment || 'production';
    this.queue = [];
    this.batchSize = options.batchSize || 10;
    this.flushInterval = options.flushInterval || 5000;

    // Flush periodically
    setInterval(() => this.flush(), this.flushInterval);
  }

  report(error, context = {}) {
    const errorData = {
      timestamp: new Date().toISOString(),
      environment: this.environment,
      message: error.message,
      name: error.name,
      stack: error.stack,
      context: {
        url: window.location.href,
        userAgent: navigator.userAgent,
        ...context
      }
    };

    this.queue.push(errorData);

    if (this.queue.length >= this.batchSize) {
      this.flush();
    }
  }

  async flush() {
    if (this.queue.length === 0) return;

    const batch = this.queue.splice(0, this.batchSize);

    try {
      await fetch(this.endpoint, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(batch)
      });
    } catch (e) {
      // Put errors back in queue
      this.queue.unshift(...batch);
    }
  }
}

// Initialize
const errorReporter = new ErrorReporter({
  endpoint: '/api/errors',
  environment: process.env.NODE_ENV
});

// Global error handler
window.addEventListener('error', (event) => {
  errorReporter.report(event.error);
});

window.addEventListener('unhandledrejection', (event) => {
  errorReporter.report(event.reason);
});
```

---

## Best Practices

### Error Messages

```javascript
// Good error messages are:
// - Specific about what went wrong
// - Include relevant context
// - Suggest resolution if possible

// BAD
throw new Error('Error');
throw new Error('Invalid input');

// GOOD
throw new Error('User email is required');
throw new Error(`Failed to parse config file: ${filePath}`);
throw new Error(
  `API rate limit exceeded. Retry after ${retryAfter} seconds.`
);

// Include error codes for programmatic handling
class AppError extends Error {
  constructor(message, code, details = {}) {
    super(message);
    this.code = code;
    this.details = details;
  }
}

throw new AppError(
  'Payment failed',
  'PAYMENT_DECLINED',
  { reason: 'insufficient_funds' }
);
```

### Error Recovery

```javascript
// Graceful degradation
async function fetchWithFallback(url, fallbackData) {
  try {
    return await fetch(url).then(r => r.json());
  } catch (error) {
    console.warn(`Failed to fetch ${url}, using fallback`);
    return fallbackData;
  }
}

// Partial failure handling
async function fetchDashboardData() {
  const [usersResult, statsResult, alertsResult] = await Promise.allSettled([
    fetchUsers(),
    fetchStats(),
    fetchAlerts()
  ]);

  return {
    users: usersResult.status === 'fulfilled'
      ? usersResult.value
      : { error: 'Failed to load users' },
    stats: statsResult.status === 'fulfilled'
      ? statsResult.value
      : { error: 'Failed to load stats' },
    alerts: alertsResult.status === 'fulfilled'
      ? alertsResult.value
      : []
  };
}
```

---

## Interview Questions

### Q1: How do you handle errors in async code?

```javascript
// 1. try/catch with async/await
async function getData() {
  try {
    const data = await fetchData();
    return data;
  } catch (error) {
    handleError(error);
  }
}

// 2. .catch() with promises
fetchData()
  .then(processData)
  .catch(handleError);

// 3. Global unhandledrejection handler
window.addEventListener('unhandledrejection', handler);
```

### Q2: What are Error Boundaries in React?

```javascript
// Class components that catch errors in child component tree
// - Catch errors during rendering, lifecycle methods, constructors
// - Don't catch: event handlers, async code, server rendering, own errors

// Use getDerivedStateFromError to update state
// Use componentDidCatch to log errors
```

### Q3: When would you use Result type vs throwing errors?

```javascript
// Throw errors:
// - Exceptional, unexpected conditions
// - Programming errors (bugs)
// - When you want stack traces

// Result type:
// - Expected failure cases (validation, not found)
// - When caller must handle both success/failure
// - Type-safe error handling
// - Functional programming style
```

---

## Key Takeaways

1. **Use custom error classes** for domain-specific errors
2. **Handle errors at appropriate level** - not too early, not too late
3. **Always catch async errors** - use try/catch or .catch()
4. **Implement global error handlers** - for unhandled errors
5. **Use Result type** for expected failures
6. **Error boundaries** for React component errors
7. **Log and report errors** to monitoring services
8. **Provide meaningful error messages** with context
9. **Implement retry logic** for transient failures
10. **Fail gracefully** - degrade functionality, don't crash
