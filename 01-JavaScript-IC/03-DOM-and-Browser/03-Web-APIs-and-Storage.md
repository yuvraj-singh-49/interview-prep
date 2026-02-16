# Web APIs and Browser Storage

## Overview

Modern browsers provide powerful APIs for network requests, storage, and more. This covers the most important Web APIs for frontend development.

---

## Fetch API

### Basic Usage

```javascript
// GET request
const response = await fetch('https://api.example.com/data');
const data = await response.json();

// With error handling
async function fetchData(url) {
  try {
    const response = await fetch(url);

    if (!response.ok) {
      throw new Error(`HTTP error! Status: ${response.status}`);
    }

    return await response.json();
  } catch (error) {
    console.error('Fetch failed:', error);
    throw error;
  }
}
```

### Request Options

```javascript
// POST with JSON
const response = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer token123'
  },
  body: JSON.stringify({ name: 'John', email: 'john@example.com' })
});

// POST with FormData
const formData = new FormData();
formData.append('file', fileInput.files[0]);
formData.append('name', 'Document');

await fetch('/upload', {
  method: 'POST',
  body: formData  // Content-Type set automatically
});

// All options
fetch(url, {
  method: 'POST',           // GET, POST, PUT, DELETE, PATCH
  headers: {},              // Request headers
  body: data,               // Request body
  mode: 'cors',             // cors, no-cors, same-origin
  credentials: 'include',   // include, same-origin, omit
  cache: 'no-cache',        // default, no-store, reload, no-cache
  redirect: 'follow',       // follow, error, manual
  signal: abortController.signal  // For cancellation
});
```

### Response Handling

```javascript
const response = await fetch(url);

// Response properties
response.ok;          // true if status 200-299
response.status;      // 200, 404, etc.
response.statusText;  // 'OK', 'Not Found', etc.
response.headers;     // Headers object
response.url;         // Final URL (after redirects)

// Parse body (can only read once!)
const json = await response.json();    // Parse as JSON
const text = await response.text();    // Parse as text
const blob = await response.blob();    // Parse as Blob
const buffer = await response.arrayBuffer();  // Parse as ArrayBuffer
const formData = await response.formData();   // Parse as FormData

// Clone for multiple reads
const clone = response.clone();
const json = await response.json();
const text = await clone.text();

// Read headers
response.headers.get('Content-Type');
response.headers.has('Authorization');
for (const [key, value] of response.headers) {
  console.log(key, value);
}
```

### Abort Requests

```javascript
const controller = new AbortController();

// Start fetch
fetch(url, { signal: controller.signal })
  .then(response => response.json())
  .catch(error => {
    if (error.name === 'AbortError') {
      console.log('Request was cancelled');
    }
  });

// Cancel after 5 seconds
setTimeout(() => controller.abort(), 5000);

// Or cancel immediately
controller.abort();

// Timeout helper
async function fetchWithTimeout(url, timeout = 5000) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(url, { signal: controller.signal });
    return response;
  } finally {
    clearTimeout(timeoutId);
  }
}
```

---

## Local Storage and Session Storage

### LocalStorage

```javascript
// Set item
localStorage.setItem('user', JSON.stringify({ name: 'John' }));
localStorage.setItem('theme', 'dark');

// Get item
const user = JSON.parse(localStorage.getItem('user'));
const theme = localStorage.getItem('theme');

// Remove item
localStorage.removeItem('user');

// Clear all
localStorage.clear();

// Get all keys
for (let i = 0; i < localStorage.length; i++) {
  const key = localStorage.key(i);
  console.log(key, localStorage.getItem(key));
}

// Storage event (cross-tab communication)
window.addEventListener('storage', (e) => {
  console.log('Storage changed:', e.key, e.oldValue, e.newValue);
});
```

### SessionStorage

```javascript
// Same API as localStorage
sessionStorage.setItem('sessionId', '123');
const sessionId = sessionStorage.getItem('sessionId');

// Difference: Data cleared when tab closes
```

### Storage Wrapper

```javascript
const storage = {
  get(key, defaultValue = null) {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : defaultValue;
    } catch {
      return defaultValue;
    }
  },

  set(key, value) {
    try {
      localStorage.setItem(key, JSON.stringify(value));
      return true;
    } catch (e) {
      console.error('Storage error:', e);
      return false;
    }
  },

  remove(key) {
    localStorage.removeItem(key);
  },

  clear() {
    localStorage.clear();
  }
};

// Usage
storage.set('user', { name: 'John', age: 30 });
const user = storage.get('user');
```

### Cookies

```javascript
// Set cookie
document.cookie = 'username=John';
document.cookie = 'theme=dark; max-age=86400';  // 1 day
document.cookie = 'session=abc; path=/; secure; samesite=strict';

// All options
document.cookie = `
  name=value;
  expires=${new Date(Date.now() + 86400000).toUTCString()};
  max-age=86400;
  path=/;
  domain=example.com;
  secure;
  samesite=strict
`;

// Get cookies
const cookies = document.cookie;  // "name1=value1; name2=value2"

// Parse cookies
function getCookie(name) {
  const match = document.cookie.match(new RegExp(`(^| )${name}=([^;]+)`));
  return match ? match[2] : null;
}

// Cookie utility
const cookieUtil = {
  get(name) {
    const match = document.cookie.match(new RegExp(`(^| )${name}=([^;]+)`));
    return match ? decodeURIComponent(match[2]) : null;
  },

  set(name, value, days = 7) {
    const expires = new Date(Date.now() + days * 864e5).toUTCString();
    document.cookie = `${name}=${encodeURIComponent(value)}; expires=${expires}; path=/`;
  },

  remove(name) {
    document.cookie = `${name}=; max-age=0; path=/`;
  }
};
```

---

## IndexedDB

```javascript
// Open database
const request = indexedDB.open('MyDatabase', 1);

request.onerror = (event) => {
  console.error('Database error:', event.target.error);
};

request.onupgradeneeded = (event) => {
  const db = event.target.result;

  // Create object store
  const store = db.createObjectStore('users', { keyPath: 'id', autoIncrement: true });
  store.createIndex('email', 'email', { unique: true });
  store.createIndex('name', 'name');
};

request.onsuccess = (event) => {
  const db = event.target.result;

  // Add data
  const transaction = db.transaction(['users'], 'readwrite');
  const store = transaction.objectStore('users');
  store.add({ name: 'John', email: 'john@example.com' });

  // Read data
  const getRequest = store.get(1);
  getRequest.onsuccess = () => console.log(getRequest.result);

  // Query by index
  const index = store.index('email');
  const indexRequest = index.get('john@example.com');

  // Get all
  const allRequest = store.getAll();

  // Delete
  store.delete(1);

  // Clear all
  store.clear();
};

// Promise wrapper
class DB {
  constructor(name, version) {
    this.name = name;
    this.version = version;
  }

  open() {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.name, this.version);
      request.onerror = () => reject(request.error);
      request.onsuccess = () => resolve(request.result);
      request.onupgradeneeded = (e) => this.onUpgrade?.(e.target.result);
    });
  }

  async add(storeName, data) {
    const db = await this.open();
    return new Promise((resolve, reject) => {
      const transaction = db.transaction([storeName], 'readwrite');
      const store = transaction.objectStore(storeName);
      const request = store.add(data);
      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }
}
```

---

## History API

```javascript
// Navigate
history.pushState({ page: 1 }, 'Title', '/page1');  // Add to history
history.replaceState({ page: 2 }, 'Title', '/page2');  // Replace current

// Navigate back/forward
history.back();
history.forward();
history.go(-2);  // Go back 2 pages
history.go(1);   // Go forward 1 page

// Current state
console.log(history.state);  // { page: 1 }
console.log(history.length); // Number of entries

// Listen for navigation
window.addEventListener('popstate', (event) => {
  console.log('Navigation:', event.state);
  // Handle state change
  renderPage(event.state);
});

// SPA router example
class Router {
  constructor() {
    this.routes = {};
    window.addEventListener('popstate', () => this.handleRoute());
  }

  addRoute(path, handler) {
    this.routes[path] = handler;
  }

  navigate(path, state = {}) {
    history.pushState(state, '', path);
    this.handleRoute();
  }

  handleRoute() {
    const path = window.location.pathname;
    const handler = this.routes[path];
    if (handler) handler();
  }
}
```

---

## Geolocation API

```javascript
// Get current position
navigator.geolocation.getCurrentPosition(
  (position) => {
    console.log('Latitude:', position.coords.latitude);
    console.log('Longitude:', position.coords.longitude);
    console.log('Accuracy:', position.coords.accuracy);
  },
  (error) => {
    switch (error.code) {
      case error.PERMISSION_DENIED:
        console.log('User denied location');
        break;
      case error.POSITION_UNAVAILABLE:
        console.log('Position unavailable');
        break;
      case error.TIMEOUT:
        console.log('Request timeout');
        break;
    }
  },
  {
    enableHighAccuracy: true,
    timeout: 5000,
    maximumAge: 0
  }
);

// Watch position (continuous updates)
const watchId = navigator.geolocation.watchPosition(
  (position) => updateMap(position.coords),
  (error) => handleError(error)
);

// Stop watching
navigator.geolocation.clearWatch(watchId);

// Promise wrapper
function getCurrentPosition(options = {}) {
  return new Promise((resolve, reject) => {
    navigator.geolocation.getCurrentPosition(resolve, reject, options);
  });
}

const position = await getCurrentPosition();
```

---

## Intersection Observer

```javascript
// Create observer
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      console.log('Element is visible:', entry.target);
      entry.target.classList.add('visible');
    }
  });
}, {
  root: null,          // viewport
  rootMargin: '0px',   // margin around root
  threshold: 0.5       // 50% visible (can be array [0, 0.5, 1])
});

// Observe elements
document.querySelectorAll('.lazy').forEach(el => observer.observe(el));

// Stop observing
observer.unobserve(element);
observer.disconnect();  // Stop all

// Lazy loading images
const imageObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;
      img.classList.remove('lazy');
      imageObserver.unobserve(img);
    }
  });
});

document.querySelectorAll('img.lazy').forEach(img => imageObserver.observe(img));

// Infinite scroll
const sentinel = document.querySelector('.sentinel');
const infiniteObserver = new IntersectionObserver((entries) => {
  if (entries[0].isIntersecting) {
    loadMoreItems();
  }
});
infiniteObserver.observe(sentinel);
```

---

## Web Workers

```javascript
// main.js
const worker = new Worker('worker.js');

// Send message to worker
worker.postMessage({ type: 'calculate', data: [1, 2, 3] });

// Receive message from worker
worker.onmessage = (event) => {
  console.log('Result from worker:', event.data);
};

worker.onerror = (error) => {
  console.error('Worker error:', error);
};

// Terminate worker
worker.terminate();

// worker.js
self.onmessage = (event) => {
  const { type, data } = event.data;

  if (type === 'calculate') {
    const result = data.reduce((sum, n) => sum + n, 0);
    self.postMessage(result);
  }
};

// Inline worker (from string)
const code = `
  self.onmessage = (e) => {
    self.postMessage(e.data * 2);
  };
`;
const blob = new Blob([code], { type: 'application/javascript' });
const worker = new Worker(URL.createObjectURL(blob));
```

---

## Storage Comparison

| Feature | LocalStorage | SessionStorage | Cookies | IndexedDB |
|---------|-------------|----------------|---------|-----------|
| Capacity | ~5MB | ~5MB | ~4KB | Large (quota) |
| Persistence | Forever | Tab session | Configurable | Forever |
| Accessible from | Same origin | Same origin/tab | Server + client | Same origin |
| API | Sync | Sync | String | Async |
| Data type | String | String | String | Any |
| Use case | Settings | Temp data | Auth tokens | Large data |

---

## Interview Questions

### Q1: Difference between localStorage and sessionStorage?

```javascript
// localStorage: Persists after browser closes
// sessionStorage: Cleared when tab closes
// Both: ~5MB limit, same origin only, sync API
```

### Q2: How would you implement request caching with fetch?

```javascript
const cache = new Map();

async function cachedFetch(url, options = {}, ttl = 60000) {
  const key = url + JSON.stringify(options);

  if (cache.has(key)) {
    const { data, timestamp } = cache.get(key);
    if (Date.now() - timestamp < ttl) {
      return data;
    }
  }

  const response = await fetch(url, options);
  const data = await response.json();

  cache.set(key, { data, timestamp: Date.now() });
  return data;
}
```

### Q3: When would you use IndexedDB over localStorage?

```javascript
// IndexedDB for:
// - Large amounts of data
// - Complex queries
// - Binary data (files, blobs)
// - Structured data with indexes

// localStorage for:
// - Simple key-value pairs
// - Small amounts of data
// - Configuration/settings
```

---

## Key Takeaways

1. **Fetch API** is the modern way to make HTTP requests
2. **AbortController** for cancelling requests
3. **localStorage** persists; **sessionStorage** is tab-only
4. **IndexedDB** for large/complex data
5. **History API** for SPA routing
6. **IntersectionObserver** for visibility detection
7. **Web Workers** for CPU-intensive tasks
8. Always handle errors and edge cases
