# Large-Scale Frontend Architecture

## Frontend System Design Interview Focus

```
Unlike backend system design, frontend focuses on:
• Component architecture
• State management
• Performance optimization
• Bundle size and loading
• Offline support
• Real-time updates
• Accessibility
```

---

## Architecture Patterns

### Micro-Frontends

```
┌─────────────────────────────────────────────────────────────┐
│                     Container App                            │
│  (Shell: routing, auth, shared UI)                          │
├─────────────────┬─────────────────┬─────────────────────────┤
│   Product MFE   │   Cart MFE      │   Checkout MFE          │
│   (Team A)      │   (Team B)      │   (Team C)              │
│   React 18      │   Vue 3         │   React 18              │
│   npm/yarn      │   pnpm          │   npm/yarn              │
└─────────────────┴─────────────────┴─────────────────────────┘
```

```javascript
// Module Federation (Webpack 5)
// Container app
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'container',
      remotes: {
        productApp: 'productApp@http://products.example.com/remoteEntry.js',
        cartApp: 'cartApp@http://cart.example.com/remoteEntry.js',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
      },
    }),
  ],
};

// Product MFE
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'productApp',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductList': './src/components/ProductList',
        './ProductDetail': './src/components/ProductDetail',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
      },
    }),
  ],
};
```

### Component-Based Architecture

```
src/
├── components/           # Shared UI components
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx
│   │   ├── Button.styles.ts
│   │   └── index.ts
│   └── ...
├── features/            # Feature-based modules
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   ├── store/
│   │   └── types/
│   ├── products/
│   └── checkout/
├── shared/              # Shared utilities
│   ├── hooks/
│   ├── utils/
│   └── constants/
├── pages/               # Route pages
└── app/                 # App-level concerns
    ├── store/
    ├── router/
    └── providers/
```

---

## State Management at Scale

### State Categories

```
┌─────────────────────────────────────────────────────────────┐
│                    State Types                               │
├─────────────────┬───────────────────────────────────────────┤
│ Server State    │ Data from APIs (users, products, etc.)   │
│                 │ → React Query, SWR, RTK Query            │
├─────────────────┼───────────────────────────────────────────┤
│ Client State    │ UI state (modals, forms, filters)        │
│                 │ → useState, Zustand, Jotai               │
├─────────────────┼───────────────────────────────────────────┤
│ URL State       │ Current route, query params              │
│                 │ → React Router, Next.js Router           │
├─────────────────┼───────────────────────────────────────────┤
│ Form State      │ Form inputs, validation                   │
│                 │ → React Hook Form, Formik                │
└─────────────────┴───────────────────────────────────────────┘
```

### Server State with React Query

```javascript
// Queries (read)
function useProducts(filters) {
  return useQuery({
    queryKey: ['products', filters],
    queryFn: () => fetchProducts(filters),
    staleTime: 5 * 60 * 1000,  // 5 minutes
    cacheTime: 30 * 60 * 1000, // 30 minutes
  });
}

// Mutations (write)
function useCreateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createProduct,
    onSuccess: () => {
      queryClient.invalidateQueries(['products']);
    },
    // Optimistic updates
    onMutate: async (newProduct) => {
      await queryClient.cancelQueries(['products']);
      const previous = queryClient.getQueryData(['products']);
      queryClient.setQueryData(['products'], old => [...old, newProduct]);
      return { previous };
    },
    onError: (err, newProduct, context) => {
      queryClient.setQueryData(['products'], context.previous);
    },
  });
}
```

### Global State with Zustand

```javascript
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
  totalPrice: () => number;
}

const useCartStore = create<CartStore>()(
  devtools(
    persist(
      (set, get) => ({
        items: [],

        addItem: (item) =>
          set((state) => ({
            items: [...state.items, item],
          })),

        removeItem: (id) =>
          set((state) => ({
            items: state.items.filter((i) => i.id !== id),
          })),

        clearCart: () => set({ items: [] }),

        totalPrice: () =>
          get().items.reduce((sum, item) => sum + item.price, 0),
      }),
      { name: 'cart-storage' }
    )
  )
);
```

---

## Performance Patterns

### Code Splitting

```javascript
// Route-based splitting
const ProductPage = lazy(() => import('./pages/ProductPage'));
const CheckoutPage = lazy(() => import('./pages/CheckoutPage'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/products" element={<ProductPage />} />
        <Route path="/checkout" element={<CheckoutPage />} />
      </Routes>
    </Suspense>
  );
}

// Component-based splitting
const HeavyChart = lazy(() => import('./components/HeavyChart'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}
```

### Virtualization

```javascript
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }) {
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50, // Estimated row height
    overscan: 5, // Render 5 extra items above/below viewport
  });

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              transform: `translateY(${virtualItem.start}px)`,
              height: `${virtualItem.size}px`,
            }}
          >
            {items[virtualItem.index]}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Memoization

```javascript
// Memoize expensive components
const ExpensiveList = memo(function ExpensiveList({ items, onSelect }) {
  return items.map(item => (
    <ExpensiveItem key={item.id} item={item} onSelect={onSelect} />
  ));
});

// Memoize callbacks
function Parent() {
  const [filter, setFilter] = useState('');

  const handleSelect = useCallback((id) => {
    console.log('Selected:', id);
  }, []);

  const filteredItems = useMemo(
    () => items.filter(item => item.name.includes(filter)),
    [items, filter]
  );

  return <ExpensiveList items={filteredItems} onSelect={handleSelect} />;
}
```

---

## Real-Time Updates

### WebSocket Integration

```javascript
import { useEffect, useRef, useCallback } from 'react';
import { useQueryClient } from '@tanstack/react-query';

function useWebSocket(url) {
  const ws = useRef(null);
  const queryClient = useQueryClient();

  useEffect(() => {
    ws.current = new WebSocket(url);

    ws.current.onmessage = (event) => {
      const message = JSON.parse(event.data);

      switch (message.type) {
        case 'PRODUCT_UPDATED':
          queryClient.invalidateQueries(['products', message.productId]);
          break;
        case 'ORDER_STATUS_CHANGED':
          queryClient.setQueryData(
            ['orders', message.orderId],
            (old) => ({ ...old, status: message.status })
          );
          break;
      }
    };

    return () => ws.current?.close();
  }, [url, queryClient]);

  const send = useCallback((message) => {
    ws.current?.send(JSON.stringify(message));
  }, []);

  return { send };
}
```

### Optimistic Updates

```javascript
function useUpdateTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updateTodo,

    // Optimistically update before server responds
    onMutate: async (updatedTodo) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries(['todos', updatedTodo.id]);

      // Snapshot previous value
      const previous = queryClient.getQueryData(['todos', updatedTodo.id]);

      // Optimistically update
      queryClient.setQueryData(['todos', updatedTodo.id], updatedTodo);

      return { previous };
    },

    // Rollback on error
    onError: (err, updatedTodo, context) => {
      queryClient.setQueryData(
        ['todos', updatedTodo.id],
        context.previous
      );
    },

    // Refetch after success or error
    onSettled: (data, error, variables) => {
      queryClient.invalidateQueries(['todos', variables.id]);
    },
  });
}
```

---

## Offline Support

### Service Worker Caching

```javascript
// service-worker.js
const CACHE_NAME = 'app-v1';
const STATIC_ASSETS = [
  '/',
  '/index.html',
  '/static/js/main.js',
  '/static/css/main.css',
];

// Cache static assets on install
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(STATIC_ASSETS);
    })
  );
});

// Network-first with cache fallback
self.addEventListener('fetch', (event) => {
  if (event.request.method !== 'GET') return;

  event.respondWith(
    fetch(event.request)
      .then((response) => {
        const clone = response.clone();
        caches.open(CACHE_NAME).then((cache) => {
          cache.put(event.request, clone);
        });
        return response;
      })
      .catch(() => {
        return caches.match(event.request);
      })
  );
});
```

### Offline Data Sync

```javascript
// Queue mutations when offline
const offlineQueue = [];

async function mutate(mutation) {
  if (navigator.onLine) {
    return executeMutation(mutation);
  }

  // Queue for later
  offlineQueue.push({
    ...mutation,
    timestamp: Date.now(),
  });

  // Store in IndexedDB
  await saveToIndexedDB('offlineQueue', offlineQueue);

  // Optimistically update local state
  updateLocalState(mutation);
}

// Sync when back online
window.addEventListener('online', async () => {
  const queue = await getFromIndexedDB('offlineQueue');

  for (const mutation of queue) {
    try {
      await executeMutation(mutation);
    } catch (error) {
      handleSyncError(mutation, error);
    }
  }

  await clearIndexedDB('offlineQueue');
});
```

---

## Design System Implementation

```javascript
// Theme tokens
const theme = {
  colors: {
    primary: {
      50: '#eff6ff',
      500: '#3b82f6',
      900: '#1e3a8a',
    },
    // ...
  },
  spacing: {
    xs: '4px',
    sm: '8px',
    md: '16px',
    lg: '24px',
    xl: '32px',
  },
  typography: {
    fontFamily: 'Inter, system-ui, sans-serif',
    fontSize: {
      sm: '0.875rem',
      base: '1rem',
      lg: '1.125rem',
    },
  },
};

// Component with variants
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'ghost';
  size: 'sm' | 'md' | 'lg';
}

const Button = styled.button<ButtonProps>`
  font-family: ${({ theme }) => theme.typography.fontFamily};

  ${({ variant, theme }) => {
    switch (variant) {
      case 'primary':
        return `
          background: ${theme.colors.primary[500]};
          color: white;
        `;
      case 'secondary':
        return `
          background: transparent;
          border: 1px solid ${theme.colors.primary[500]};
        `;
    }
  }}

  ${({ size, theme }) => {
    switch (size) {
      case 'sm':
        return `padding: ${theme.spacing.xs} ${theme.spacing.sm};`;
      case 'md':
        return `padding: ${theme.spacing.sm} ${theme.spacing.md};`;
      case 'lg':
        return `padding: ${theme.spacing.md} ${theme.spacing.lg};`;
    }
  }}
`;
```

---

## Interview Questions

### Q: How would you design a complex form with validation?

1. Use form library (React Hook Form) for performance
2. Schema validation (Zod, Yup)
3. Field-level and form-level validation
4. Async validation for uniqueness checks
5. Proper error handling and display
6. Accessibility (ARIA labels, focus management)

### Q: How do you handle state in a large application?

1. Separate server state (React Query) from client state
2. Keep state as local as possible
3. Use URL state for shareable state
4. Global store only for truly global state
5. Consider co-location of state with components

### Q: How do you optimize a slow React application?

1. Profile with React DevTools
2. Memoize expensive components (memo, useMemo)
3. Virtualize long lists
4. Code split by routes
5. Lazy load heavy components
6. Optimize re-renders (avoid context abuse)
