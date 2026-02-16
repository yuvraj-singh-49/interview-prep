# Network Performance

## Overview

Network performance is critical for web applications. This covers caching strategies, resource optimization, and techniques to minimize load times.

---

## Resource Loading

### Preload, Prefetch, Preconnect

```html
<!-- Preload: High-priority resource needed for current page -->
<link rel="preload" href="critical.css" as="style">
<link rel="preload" href="hero.jpg" as="image">
<link rel="preload" href="font.woff2" as="font" crossorigin>
<link rel="preload" href="app.js" as="script">

<!-- Prefetch: Resource likely needed for future navigation -->
<link rel="prefetch" href="next-page.js">
<link rel="prefetch" href="/api/user-data">

<!-- Preconnect: Establish early connection to origin -->
<link rel="preconnect" href="https://api.example.com">
<link rel="preconnect" href="https://fonts.googleapis.com" crossorigin>

<!-- DNS Prefetch: Resolve DNS early (less aggressive than preconnect) -->
<link rel="dns-prefetch" href="https://analytics.example.com">

<!-- Modulepreload: Preload ES modules -->
<link rel="modulepreload" href="./module.js">
```

### Resource Hints in JavaScript

```javascript
// Dynamically add resource hints
function preloadResource(url, as) {
  const link = document.createElement('link');
  link.rel = 'preload';
  link.href = url;
  link.as = as;
  document.head.appendChild(link);
}

// Preload on user intent
button.addEventListener('mouseenter', () => {
  preloadResource('/api/data', 'fetch');
});

// Prefetch on idle
requestIdleCallback(() => {
  const link = document.createElement('link');
  link.rel = 'prefetch';
  link.href = '/next-page.js';
  document.head.appendChild(link);
});
```

### Priority Hints

```html
<!-- fetchpriority attribute (Chrome) -->
<img src="hero.jpg" fetchpriority="high" alt="Hero">
<img src="below-fold.jpg" fetchpriority="low" alt="Below fold">

<script src="critical.js" fetchpriority="high"></script>
<script src="analytics.js" fetchpriority="low"></script>

<!-- In fetch API -->
<script>
fetch('/api/critical-data', { priority: 'high' });
fetch('/api/analytics', { priority: 'low' });
</script>
```

---

## Caching Strategies

### Browser Cache Headers

```javascript
// Cache-Control header examples

// Public cache, max 1 year (for versioned/hashed files)
// Cache-Control: public, max-age=31536000, immutable

// Private cache (user-specific data)
// Cache-Control: private, max-age=3600

// No cache (must revalidate)
// Cache-Control: no-cache

// No store (sensitive data)
// Cache-Control: no-store

// ETag for conditional requests
// ETag: "abc123"
// If-None-Match: "abc123"  →  304 Not Modified

// Last-Modified
// Last-Modified: Wed, 15 Jan 2025 12:00:00 GMT
// If-Modified-Since: Wed, 15 Jan 2025 12:00:00 GMT
```

### Service Worker Caching

```javascript
// service-worker.js

const CACHE_NAME = 'v1';
const STATIC_ASSETS = [
  '/',
  '/styles.css',
  '/app.js',
  '/offline.html'
];

// Install - cache static assets
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(STATIC_ASSETS))
  );
});

// Activate - clean old caches
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys => {
      return Promise.all(
        keys.filter(key => key !== CACHE_NAME)
            .map(key => caches.delete(key))
      );
    })
  );
});

// Fetch - serve from cache, fallback to network
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then(cached => cached || fetch(event.request))
  );
});
```

### Caching Strategies

```javascript
// 1. Cache First (Static Assets)
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  const response = await fetch(request);
  const cache = await caches.open(CACHE_NAME);
  cache.put(request, response.clone());
  return response;
}

// 2. Network First (API Data)
async function networkFirst(request) {
  try {
    const response = await fetch(request);
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, response.clone());
    return response;
  } catch {
    return caches.match(request);
  }
}

// 3. Stale While Revalidate (Balance)
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME);
  const cached = await cache.match(request);

  const fetchPromise = fetch(request).then(response => {
    cache.put(request, response.clone());
    return response;
  });

  return cached || fetchPromise;
}

// 4. Network Only (Real-time data)
async function networkOnly(request) {
  return fetch(request);
}

// 5. Cache Only (Offline-first)
async function cacheOnly(request) {
  return caches.match(request);
}

// Strategy selection in fetch handler
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(request));
  } else if (url.pathname.match(/\.(js|css|png|jpg)$/)) {
    event.respondWith(cacheFirst(request));
  } else {
    event.respondWith(staleWhileRevalidate(request));
  }
});
```

### Application-Level Caching

```javascript
// In-memory cache with TTL
class Cache {
  constructor(ttl = 60000) {
    this.cache = new Map();
    this.ttl = ttl;
  }

  get(key) {
    const item = this.cache.get(key);
    if (!item) return undefined;

    if (Date.now() > item.expiry) {
      this.cache.delete(key);
      return undefined;
    }

    return item.value;
  }

  set(key, value, ttl = this.ttl) {
    this.cache.set(key, {
      value,
      expiry: Date.now() + ttl
    });
  }

  delete(key) {
    this.cache.delete(key);
  }

  clear() {
    this.cache.clear();
  }
}

// Usage with fetch
const apiCache = new Cache(5 * 60 * 1000);  // 5 minutes

async function cachedFetch(url) {
  const cached = apiCache.get(url);
  if (cached) return cached;

  const response = await fetch(url);
  const data = await response.json();

  apiCache.set(url, data);
  return data;
}
```

---

## Compression and Bundling

### Gzip and Brotli

```javascript
// Server-side compression (Express example)
const compression = require('compression');
app.use(compression());

// Nginx configuration
// gzip on;
// gzip_types text/plain application/json application/javascript text/css;
// gzip_min_length 1000;

// Brotli (better compression)
// brotli on;
// brotli_types text/plain application/json application/javascript text/css;
```

### Code Splitting

```javascript
// Webpack - dynamic imports
const Dashboard = () => import('./Dashboard');
const Profile = () => import('./Profile');

// Route-based splitting (React Router)
const routes = [
  {
    path: '/dashboard',
    component: React.lazy(() => import('./Dashboard'))
  },
  {
    path: '/profile',
    component: React.lazy(() => import('./Profile'))
  }
];

// Vendor splitting (webpack.config.js)
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all'
        }
      }
    }
  }
};
```

### Tree Shaking

```javascript
// Ensure ES modules for tree shaking

// Good - named exports are tree-shakeable
export function used() { }
export function unused() { }

import { used } from './utils';  // unused is eliminated

// Bad - default export of object
export default {
  used: () => {},
  unused: () => {}
};

// package.json sideEffects
{
  "sideEffects": false,  // All files are pure
  // or
  "sideEffects": ["*.css"]  // CSS has side effects
}
```

---

## Image Optimization

### Modern Formats

```html
<!-- Picture element for format fallback -->
<picture>
  <source srcset="image.avif" type="image/avif">
  <source srcset="image.webp" type="image/webp">
  <img src="image.jpg" alt="Fallback">
</picture>

<!-- Responsive images -->
<img
  src="small.jpg"
  srcset="small.jpg 400w, medium.jpg 800w, large.jpg 1200w"
  sizes="(max-width: 400px) 400px, (max-width: 800px) 800px, 1200px"
  alt="Responsive image"
>
```

### Lazy Loading

```html
<!-- Native lazy loading -->
<img src="image.jpg" loading="lazy" alt="Lazy">

<!-- With intersection observer -->
<img data-src="image.jpg" class="lazy" alt="Lazy">
```

```javascript
const lazyImages = document.querySelectorAll('img.lazy');

const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;
      img.classList.remove('lazy');
      observer.unobserve(img);
    }
  });
}, { rootMargin: '200px' });

lazyImages.forEach(img => observer.observe(img));
```

### Image CDN

```javascript
// Using image CDN for on-the-fly optimization
// Cloudinary, imgix, Cloudflare Images

// Transform URL
const imageUrl = (src, { width, height, format, quality }) => {
  return `https://cdn.example.com/${src}?w=${width}&h=${height}&f=${format}&q=${quality}`;
};

// Usage
imageUrl('photo.jpg', {
  width: 400,
  height: 300,
  format: 'webp',
  quality: 80
});
```

---

## HTTP/2 and HTTP/3

### HTTP/2 Benefits

```javascript
// HTTP/2 features:
// - Multiplexing: Multiple requests over single connection
// - Server Push: Proactive resource delivery
// - Header compression (HPACK)
// - Stream prioritization

// No need for:
// - Domain sharding (multiple CDN domains)
// - Bundling many small files (can be counterproductive)
// - Sprite sheets (individual files are fine)

// Still important:
// - Minification
// - Compression
// - Caching
```

### Server Push (HTTP/2)

```javascript
// Express with HTTP/2
const http2 = require('http2');
const fs = require('fs');

const server = http2.createSecureServer({
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
});

server.on('stream', (stream, headers) => {
  // Push CSS before HTML
  const pushStream = stream.pushStream({ ':path': '/styles.css' }, (err, push) => {
    if (err) throw err;
    push.respond({ 'content-type': 'text/css' });
    push.end(fs.readFileSync('styles.css'));
  });

  // Respond with HTML
  stream.respond({ 'content-type': 'text/html' });
  stream.end(fs.readFileSync('index.html'));
});
```

---

## API Performance

### Request Batching

```javascript
// Batch multiple requests
class RequestBatcher {
  constructor(batchFn, delay = 50) {
    this.batchFn = batchFn;
    this.delay = delay;
    this.queue = [];
    this.timeout = null;
  }

  add(request) {
    return new Promise((resolve, reject) => {
      this.queue.push({ request, resolve, reject });

      if (!this.timeout) {
        this.timeout = setTimeout(() => this.flush(), this.delay);
      }
    });
  }

  async flush() {
    const batch = this.queue;
    this.queue = [];
    this.timeout = null;

    try {
      const results = await this.batchFn(batch.map(b => b.request));
      batch.forEach((b, i) => b.resolve(results[i]));
    } catch (error) {
      batch.forEach(b => b.reject(error));
    }
  }
}

// Usage
const userBatcher = new RequestBatcher(async (ids) => {
  const response = await fetch(`/api/users?ids=${ids.join(',')}`);
  return response.json();
});

// These become a single request
const user1 = userBatcher.add(1);
const user2 = userBatcher.add(2);
const user3 = userBatcher.add(3);
```

### Request Deduplication

```javascript
// Deduplicate concurrent identical requests
class RequestDeduplicator {
  constructor() {
    this.pending = new Map();
  }

  async fetch(url, options = {}) {
    const key = `${options.method || 'GET'}:${url}`;

    if (this.pending.has(key)) {
      return this.pending.get(key);
    }

    const promise = fetch(url, options)
      .then(response => response.json())
      .finally(() => this.pending.delete(key));

    this.pending.set(key, promise);
    return promise;
  }
}

const deduplicator = new RequestDeduplicator();

// Multiple calls become single request
const p1 = deduplicator.fetch('/api/user/1');
const p2 = deduplicator.fetch('/api/user/1');
const p3 = deduplicator.fetch('/api/user/1');
// Only one network request made
```

### Pagination and Infinite Scroll

```javascript
// Cursor-based pagination (more efficient than offset)
async function fetchPage(cursor = null, limit = 20) {
  const params = new URLSearchParams({ limit });
  if (cursor) params.set('cursor', cursor);

  const response = await fetch(`/api/items?${params}`);
  return response.json();
}

// Infinite scroll
class InfiniteScroll {
  constructor(container, fetchFn) {
    this.container = container;
    this.fetchFn = fetchFn;
    this.cursor = null;
    this.loading = false;
    this.hasMore = true;

    this.observer = new IntersectionObserver(
      entries => this.handleIntersect(entries),
      { rootMargin: '200px' }
    );

    this.sentinel = document.createElement('div');
    container.appendChild(this.sentinel);
    this.observer.observe(this.sentinel);
  }

  async handleIntersect(entries) {
    if (!entries[0].isIntersecting || this.loading || !this.hasMore) return;

    this.loading = true;
    const { items, nextCursor } = await this.fetchFn(this.cursor);

    items.forEach(item => this.renderItem(item));

    this.cursor = nextCursor;
    this.hasMore = !!nextCursor;
    this.loading = false;
  }

  renderItem(item) {
    const el = document.createElement('div');
    el.textContent = item.name;
    this.container.insertBefore(el, this.sentinel);
  }
}
```

### GraphQL Query Optimization

```javascript
// Only request needed fields
const query = `
  query GetUser($id: ID!) {
    user(id: $id) {
      name
      email
      # Don't request unnecessary fields
    }
  }
`;

// Use fragments for reusability
const USER_FRAGMENT = `
  fragment UserFields on User {
    id
    name
    email
  }
`;

// Batch queries with aliases
const batchQuery = `
  query {
    user1: user(id: "1") { ...UserFields }
    user2: user(id: "2") { ...UserFields }
    user3: user(id: "3") { ...UserFields }
  }
  ${USER_FRAGMENT}
`;

// DataLoader for batching (server-side)
const DataLoader = require('dataloader');

const userLoader = new DataLoader(async (ids) => {
  const users = await User.findByIds(ids);
  return ids.map(id => users.find(u => u.id === id));
});

// Multiple calls batched automatically
const user1 = userLoader.load(1);
const user2 = userLoader.load(2);
```

---

## Measuring Network Performance

### Navigation Timing API

```javascript
// Page load metrics
const timing = performance.getEntriesByType('navigation')[0];

const metrics = {
  // DNS lookup
  dns: timing.domainLookupEnd - timing.domainLookupStart,

  // TCP connection
  tcp: timing.connectEnd - timing.connectStart,

  // SSL handshake
  ssl: timing.secureConnectionStart
    ? timing.connectEnd - timing.secureConnectionStart
    : 0,

  // Time to First Byte
  ttfb: timing.responseStart - timing.requestStart,

  // Content download
  download: timing.responseEnd - timing.responseStart,

  // DOM parsing
  domParsing: timing.domInteractive - timing.responseEnd,

  // DOM Content Loaded
  domContentLoaded: timing.domContentLoadedEventEnd - timing.fetchStart,

  // Full page load
  pageLoad: timing.loadEventEnd - timing.fetchStart
};
```

### Resource Timing API

```javascript
// Analyze individual resources
const resources = performance.getEntriesByType('resource');

resources.forEach(resource => {
  console.log({
    name: resource.name,
    type: resource.initiatorType,
    duration: resource.duration,
    size: resource.transferSize,
    cached: resource.transferSize === 0
  });
});

// Find slow resources
const slowResources = resources
  .filter(r => r.duration > 1000)
  .sort((a, b) => b.duration - a.duration);
```

---

## Interview Questions

### Q1: Explain caching strategies

```javascript
// 1. Cache-First: Serve from cache, fallback to network
//    Use for: Static assets, fonts, images

// 2. Network-First: Try network, fallback to cache
//    Use for: API data, user content

// 3. Stale-While-Revalidate: Serve cache, update in background
//    Use for: News feeds, frequently updated content

// 4. Cache-Only: Only serve from cache
//    Use for: Offline-first apps

// 5. Network-Only: Only use network
//    Use for: Real-time data, sensitive info
```

### Q2: How would you optimize initial page load?

```javascript
// 1. Critical CSS inline, defer non-critical
// 2. Defer/async non-critical JavaScript
// 3. Preload critical resources
// 4. Use modern image formats (WebP, AVIF)
// 5. Lazy load below-fold content
// 6. Code splitting by route
// 7. Enable compression (Brotli/Gzip)
// 8. Use CDN for static assets
// 9. HTTP/2 for multiplexing
// 10. Service worker for repeat visits
```

### Q3: What is TTFB and how to improve it?

```javascript
// Time To First Byte: Time from request to first byte received

// Improvements:
// 1. Use CDN (closer servers)
// 2. Server-side caching (Redis)
// 3. Database query optimization
// 4. Connection reuse (keep-alive)
// 5. Reduce server processing time
// 6. Use HTTP/2 (header compression)
// 7. DNS prefetch for external origins
// 8. Preconnect to API servers
```

### Q4: How does Service Worker caching work?

```javascript
// Service Worker lifecycle:
// 1. Register SW from main thread
// 2. Install event: cache static assets
// 3. Activate event: clean old caches
// 4. Fetch event: intercept requests, serve from cache/network

// Key points:
// - Runs in separate thread
// - Only works on HTTPS
// - Can work offline
// - Updates require new SW version
// - Cache API for storage
```

---

## Key Takeaways

1. **Resource hints**: Use preload, prefetch, preconnect appropriately
2. **Service workers**: Enable offline support and caching strategies
3. **Code splitting**: Load only what's needed for current route
4. **Image optimization**: Modern formats, responsive images, lazy loading
5. **Compression**: Gzip/Brotli for text, optimize images
6. **HTTP/2**: Multiplexing eliminates need for domain sharding
7. **Request batching**: Combine multiple API calls
8. **Measure**: Use Navigation/Resource Timing APIs
9. **CDN**: Serve static assets from edge locations
10. **Cache headers**: Set appropriate max-age and caching policies
