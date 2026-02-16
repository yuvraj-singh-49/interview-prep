# Rendering Performance

## Overview

Understanding how browsers render content is essential for building performant web applications. This covers the critical rendering path, reflows, repaints, and optimization techniques.

---

## Critical Rendering Path

### How Browsers Render Pages

```
1. Parse HTML → DOM Tree
2. Parse CSS → CSSOM Tree
3. Combine → Render Tree
4. Layout (Reflow) → Calculate positions/sizes
5. Paint → Draw pixels
6. Composite → Layer composition
```

### DOM Construction

```javascript
// HTML parsing is incremental
// Scripts can block parsing

// Blocking script (bad)
<script src="app.js"></script>

// Async - downloads in parallel, executes when ready
<script async src="analytics.js"></script>

// Defer - downloads in parallel, executes after DOM ready
<script defer src="app.js"></script>

// When to use:
// async: Independent scripts (analytics, ads)
// defer: Scripts that need DOM or each other
```

### CSS and Rendering

```html
<!-- CSS blocks rendering (render-blocking) -->
<link rel="stylesheet" href="styles.css">

<!-- Media queries for conditional loading -->
<link rel="stylesheet" href="print.css" media="print">
<link rel="stylesheet" href="mobile.css" media="(max-width: 768px)">

<!-- Preload critical CSS -->
<link rel="preload" href="critical.css" as="style">
<link rel="stylesheet" href="critical.css">
```

---

## Reflow and Repaint

### What Triggers Reflow (Layout)

```javascript
// Reflow: Recalculating layout (expensive)
// Triggers when geometry changes

// These trigger reflow:
element.offsetWidth;           // Reading layout properties
element.offsetHeight;
element.offsetTop;
element.offsetLeft;
element.clientWidth;
element.clientHeight;
element.scrollTop;
element.scrollLeft;
element.getComputedStyle();
element.getBoundingClientRect();

// Writing layout-affecting properties:
element.style.width = '100px';
element.style.height = '100px';
element.style.padding = '10px';
element.style.margin = '10px';
element.style.border = '1px solid';
element.style.display = 'block';
element.style.position = 'absolute';
element.style.top = '10px';
element.style.left = '10px';
element.style.fontSize = '16px';
element.style.fontFamily = 'Arial';

// DOM changes:
element.appendChild(child);
element.removeChild(child);
element.innerHTML = 'content';
element.textContent = 'text';
```

### What Triggers Repaint Only

```javascript
// Repaint: Redrawing pixels (less expensive than reflow)
// Triggers when visual properties change without layout change

element.style.color = 'red';
element.style.backgroundColor = 'blue';
element.style.visibility = 'hidden';  // (not display: none)
element.style.outline = '1px solid';
element.style.boxShadow = '0 0 5px';
element.style.borderRadius = '5px';
```

### Layout Thrashing

```javascript
// Bad - forced synchronous layout (layout thrashing)
const elements = document.querySelectorAll('.item');

elements.forEach(el => {
  // Read (forces layout)
  const width = el.offsetWidth;
  // Write (invalidates layout)
  el.style.width = (width * 2) + 'px';
  // Next read forces layout again!
});

// Good - batch reads, then batch writes
const widths = [];

// Batch reads
elements.forEach(el => {
  widths.push(el.offsetWidth);
});

// Batch writes
elements.forEach((el, i) => {
  el.style.width = (widths[i] * 2) + 'px';
});

// Better - use requestAnimationFrame
function updateLayout() {
  // Reads
  const widths = Array.from(elements).map(el => el.offsetWidth);

  // Writes in next frame
  requestAnimationFrame(() => {
    elements.forEach((el, i) => {
      el.style.width = (widths[i] * 2) + 'px';
    });
  });
}
```

---

## Optimizing Rendering

### CSS Containment

```css
/* Tell browser element's internals don't affect outside */
.widget {
  contain: layout;    /* Layout is independent */
  contain: paint;     /* Painting is independent */
  contain: size;      /* Size is fixed */
  contain: style;     /* Styles don't escape */
  contain: content;   /* layout + paint */
  contain: strict;    /* layout + paint + size */
}

/* Will-change: Hint to browser for optimization */
.animated-element {
  will-change: transform;
  will-change: opacity;
}

/* Remove after animation */
.animated-element.done {
  will-change: auto;
}
```

### Compositor-Only Properties

```css
/* These properties can be animated on compositor thread
   (no reflow or repaint on main thread) */

/* Transform */
.efficient {
  transform: translateX(100px);
  transform: scale(1.5);
  transform: rotate(45deg);
  transform: translate3d(0, 0, 0);  /* Force GPU layer */
}

/* Opacity */
.fade {
  opacity: 0.5;
}

/* Filter */
.blur {
  filter: blur(5px);
}

/* Bad - triggers layout/paint */
.inefficient {
  left: 100px;      /* Use transform instead */
  top: 100px;       /* Use transform instead */
  width: 200px;     /* Use transform: scale instead */
  height: 200px;    /* Use transform: scale instead */
}
```

### Animation Performance

```javascript
// Bad - using setInterval
setInterval(() => {
  element.style.left = (pos++) + 'px';
}, 16);

// Good - using requestAnimationFrame
function animate() {
  element.style.transform = `translateX(${pos++}px)`;

  if (pos < target) {
    requestAnimationFrame(animate);
  }
}
requestAnimationFrame(animate);

// Best - use CSS animations/transitions
// CSS animation
.slide {
  animation: slideIn 0.3s ease-out;
}

@keyframes slideIn {
  from { transform: translateX(-100%); }
  to { transform: translateX(0); }
}

// CSS transition
.element {
  transition: transform 0.3s ease-out;
}
.element.moved {
  transform: translateX(100px);
}

// Web Animations API
element.animate([
  { transform: 'translateX(0)' },
  { transform: 'translateX(100px)' }
], {
  duration: 300,
  easing: 'ease-out',
  fill: 'forwards'
});
```

---

## Virtual DOM Concepts

### How Virtual DOM Works

```javascript
// Virtual DOM is a lightweight copy of real DOM
// Used by React, Vue, etc.

// 1. Create virtual DOM representation
const vdom = {
  type: 'div',
  props: { className: 'container' },
  children: [
    { type: 'h1', props: {}, children: ['Hello'] },
    { type: 'p', props: {}, children: ['World'] }
  ]
};

// 2. Render to real DOM
function render(vnode, container) {
  if (typeof vnode === 'string') {
    container.appendChild(document.createTextNode(vnode));
    return;
  }

  const element = document.createElement(vnode.type);

  // Set properties
  Object.entries(vnode.props || {}).forEach(([key, value]) => {
    if (key === 'className') {
      element.className = value;
    } else if (key.startsWith('on')) {
      element.addEventListener(key.slice(2).toLowerCase(), value);
    } else {
      element.setAttribute(key, value);
    }
  });

  // Render children
  (vnode.children || []).forEach(child => render(child, element));

  container.appendChild(element);
}

// 3. Diff algorithm (simplified)
function diff(oldVNode, newVNode) {
  // Different types - replace entirely
  if (oldVNode.type !== newVNode.type) {
    return { type: 'REPLACE', newVNode };
  }

  // Same type - update props
  const propPatches = diffProps(oldVNode.props, newVNode.props);

  // Diff children
  const childPatches = diffChildren(oldVNode.children, newVNode.children);

  return { type: 'UPDATE', propPatches, childPatches };
}

// 4. Apply patches (minimal DOM updates)
function patch(element, patches) {
  switch (patches.type) {
    case 'REPLACE':
      const newElement = createElement(patches.newVNode);
      element.parentNode.replaceChild(newElement, element);
      break;
    case 'UPDATE':
      applyProps(element, patches.propPatches);
      patchChildren(element, patches.childPatches);
      break;
  }
}
```

### React Reconciliation Keys

```javascript
// Keys help React identify which items changed

// Bad - no keys (entire list re-renders)
{items.map(item => <li>{item.name}</li>)}

// Bad - index as key (problems with reordering)
{items.map((item, index) => <li key={index}>{item.name}</li>)}

// Good - stable unique key
{items.map(item => <li key={item.id}>{item.name}</li>)}

// Why keys matter:
// Without proper keys, React may:
// - Re-render unchanged items
// - Lose component state on reorder
// - Incorrectly reuse DOM elements
```

---

## Lazy Loading and Code Splitting

### Image Lazy Loading

```html
<!-- Native lazy loading -->
<img src="image.jpg" loading="lazy" alt="Lazy loaded">

<!-- With Intersection Observer -->
<img data-src="image.jpg" class="lazy" alt="Lazy loaded">
```

```javascript
// Intersection Observer for lazy loading
const lazyImages = document.querySelectorAll('img.lazy');

const imageObserver = new IntersectionObserver((entries, observer) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;
      img.classList.remove('lazy');
      observer.unobserve(img);
    }
  });
}, {
  rootMargin: '100px'  // Load 100px before entering viewport
});

lazyImages.forEach(img => imageObserver.observe(img));
```

### Component Lazy Loading (React)

```javascript
// React.lazy for code splitting
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <HeavyComponent />
    </Suspense>
  );
}

// Route-based code splitting
const routes = {
  '/dashboard': React.lazy(() => import('./pages/Dashboard')),
  '/profile': React.lazy(() => import('./pages/Profile')),
  '/settings': React.lazy(() => import('./pages/Settings'))
};

// Preload on hover
function NavLink({ to, children }) {
  const preload = () => {
    routes[to]?.preload?.();
  };

  return (
    <Link to={to} onMouseEnter={preload}>
      {children}
    </Link>
  );
}
```

### Dynamic Import

```javascript
// Load module on demand
async function loadFeature() {
  const { feature } = await import('./feature.js');
  feature();
}

// Conditional loading
if (userNeedsAdvancedFeatures) {
  const { AdvancedEditor } = await import('./AdvancedEditor.js');
  renderEditor(AdvancedEditor);
}

// Webpack magic comments
const Component = await import(
  /* webpackChunkName: "my-chunk" */
  /* webpackPreload: true */
  './Component'
);
```

---

## Measuring Performance

### Performance API

```javascript
// Mark and measure
performance.mark('start-operation');
// ... operation ...
performance.mark('end-operation');

performance.measure('operation', 'start-operation', 'end-operation');

// Get measurements
const measures = performance.getEntriesByType('measure');
console.log(measures[0].duration);

// Navigation timing
const timing = performance.getEntriesByType('navigation')[0];
console.log('DOM Content Loaded:', timing.domContentLoadedEventEnd);
console.log('Load Complete:', timing.loadEventEnd);
console.log('First Paint:', performance.getEntriesByType('paint')[0].startTime);

// Resource timing
const resources = performance.getEntriesByType('resource');
resources.forEach(resource => {
  console.log(resource.name, resource.duration);
});
```

### Core Web Vitals

```javascript
// LCP - Largest Contentful Paint (< 2.5s good)
new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lastEntry = entries[entries.length - 1];
  console.log('LCP:', lastEntry.startTime);
}).observe({ entryTypes: ['largest-contentful-paint'] });

// FID - First Input Delay (< 100ms good)
new PerformanceObserver((list) => {
  const firstInput = list.getEntries()[0];
  const delay = firstInput.processingStart - firstInput.startTime;
  console.log('FID:', delay);
}).observe({ entryTypes: ['first-input'] });

// CLS - Cumulative Layout Shift (< 0.1 good)
let clsScore = 0;
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      clsScore += entry.value;
    }
  }
  console.log('CLS:', clsScore);
}).observe({ entryTypes: ['layout-shift'] });
```

---

## Interview Questions

### Q1: What is the critical rendering path?

```javascript
// The sequence browser uses to render a page:
// 1. Parse HTML → DOM
// 2. Parse CSS → CSSOM
// 3. Combine DOM + CSSOM → Render Tree
// 4. Layout (calculate positions)
// 5. Paint (draw pixels)
// 6. Composite (combine layers)

// Optimization: Minimize render-blocking resources
// - Defer non-critical JS
// - Inline critical CSS
// - Preload important resources
```

### Q2: Difference between reflow and repaint?

```javascript
// Reflow (Layout): Recalculates geometry
// - More expensive
// - Triggers when layout changes (size, position)
// - Affects children and sometimes ancestors

// Repaint: Redraws pixels without layout change
// - Less expensive
// - Triggers for visual changes (color, shadow)
// - Only affects the element

// Avoid layout thrashing: batch reads, then batch writes
```

### Q3: How would you optimize an animation?

```javascript
// 1. Use compositor-only properties
element.style.transform = 'translateX(100px)';  // Good
element.style.left = '100px';                    // Bad

// 2. Use requestAnimationFrame
requestAnimationFrame(() => {
  element.style.transform = `translateX(${pos}px)`;
});

// 3. Use CSS animations/transitions when possible
// 4. Use will-change hint (sparingly)
// 5. Avoid animating layout properties
```

### Q4: What are Core Web Vitals?

```javascript
// LCP (Largest Contentful Paint)
// - Time until largest content element is rendered
// - Target: < 2.5 seconds

// FID (First Input Delay) / INP (Interaction to Next Paint)
// - Time from user interaction to browser response
// - Target: < 100ms

// CLS (Cumulative Layout Shift)
// - Visual stability score
// - Target: < 0.1
```

---

## Key Takeaways

1. **Critical Rendering Path**: Understand HTML → DOM → CSSOM → Render → Layout → Paint
2. **Reflow is expensive**: Batch DOM reads/writes to avoid layout thrashing
3. **Use compositor-only properties**: transform, opacity for animations
4. **requestAnimationFrame**: Sync with browser rendering cycle
5. **Virtual DOM**: Minimizes real DOM operations through diffing
6. **Lazy loading**: Defer non-critical resources
7. **Measure with Performance API**: Use Core Web Vitals metrics
8. **CSS containment**: Isolate components for better performance
