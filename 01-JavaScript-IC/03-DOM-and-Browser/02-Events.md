# JavaScript Events

## Overview

Events are the foundation of interactive web applications. Understanding event handling, propagation, and delegation is crucial for frontend development.

---

## Adding Event Listeners

### addEventListener

```javascript
const button = document.querySelector('#btn');

// Basic usage
button.addEventListener('click', function(event) {
  console.log('Clicked!', event);
});

// Arrow function
button.addEventListener('click', (e) => {
  console.log('Clicked!', e.target);
});

// Named function (can be removed)
function handleClick(event) {
  console.log('Clicked!');
}
button.addEventListener('click', handleClick);

// With options
button.addEventListener('click', handleClick, {
  once: true,      // Remove after first trigger
  passive: true,   // Won't call preventDefault (performance)
  capture: true    // Capture phase instead of bubble
});

// Third parameter as boolean = useCapture
button.addEventListener('click', handleClick, true);  // Capture phase
```

### Removing Event Listeners

```javascript
// Must use same function reference
function handleClick(e) {
  console.log('Clicked!');
}

button.addEventListener('click', handleClick);
button.removeEventListener('click', handleClick);

// Arrow functions can't be removed (no reference)
button.addEventListener('click', () => console.log('Click'));
// Can't remove this!

// AbortController (modern approach)
const controller = new AbortController();

button.addEventListener('click', handleClick, {
  signal: controller.signal
});

// Remove all listeners with this signal
controller.abort();
```

### Inline Events (Avoid)

```html
<!-- Old way - avoid -->
<button onclick="handleClick()">Click</button>

<!-- Problems:
  - Global scope only
  - Can't remove
  - Security risks
  - Mixing HTML and JS
-->
```

---

## Event Object

```javascript
element.addEventListener('click', function(event) {
  // Common properties
  event.type;           // 'click'
  event.target;         // Element that triggered event
  event.currentTarget;  // Element with listener attached
  event.timeStamp;      // When event occurred

  // Position (mouse events)
  event.clientX;        // X relative to viewport
  event.clientY;        // Y relative to viewport
  event.pageX;          // X relative to document
  event.pageY;          // Y relative to document
  event.screenX;        // X relative to screen
  event.screenY;        // Y relative to screen
  event.offsetX;        // X relative to target element
  event.offsetY;        // Y relative to target element

  // Keyboard events
  event.key;            // 'Enter', 'a', 'Shift', etc.
  event.code;           // 'KeyA', 'Enter', 'ShiftLeft'
  event.keyCode;        // Deprecated, use key/code

  // Modifier keys
  event.altKey;         // Alt/Option pressed
  event.ctrlKey;        // Ctrl pressed
  event.shiftKey;       // Shift pressed
  event.metaKey;        // Cmd (Mac) / Win (Windows)

  // Mouse buttons
  event.button;         // 0=left, 1=middle, 2=right
  event.buttons;        // Bitmask of buttons

  // Propagation control
  event.stopPropagation();      // Stop bubbling/capturing
  event.stopImmediatePropagation(); // Stop all handlers
  event.preventDefault();        // Prevent default action

  // Check if default prevented
  event.defaultPrevented;

  // Bubbles/cancelable
  event.bubbles;
  event.cancelable;
});
```

---

## Event Propagation

### Three Phases

```
1. Capture Phase: Window → Document → ... → Target's parent
2. Target Phase: Target element
3. Bubble Phase: Target's parent → ... → Document → Window
```

```javascript
// Bubble phase (default)
parent.addEventListener('click', (e) => {
  console.log('Parent clicked (bubble)');
});

// Capture phase
parent.addEventListener('click', (e) => {
  console.log('Parent clicked (capture)');
}, true);

// Order for click on child:
// 1. Parent capture
// 2. Child handlers
// 3. Parent bubble
```

### stopPropagation vs stopImmediatePropagation

```javascript
// stopPropagation - stops going to parent/child
element.addEventListener('click', (e) => {
  e.stopPropagation();
  console.log('First handler');
});

element.addEventListener('click', (e) => {
  console.log('Second handler');  // Still runs!
});

// stopImmediatePropagation - stops all handlers
element.addEventListener('click', (e) => {
  e.stopImmediatePropagation();
  console.log('First handler');
});

element.addEventListener('click', (e) => {
  console.log('Second handler');  // Doesn't run!
});
```

### preventDefault

```javascript
// Prevent default browser action
link.addEventListener('click', (e) => {
  e.preventDefault();  // Don't navigate
  console.log('Link clicked but not followed');
});

form.addEventListener('submit', (e) => {
  e.preventDefault();  // Don't submit
  // Handle form manually
});

input.addEventListener('keydown', (e) => {
  if (e.key === 'Enter') {
    e.preventDefault();  // Don't submit form on Enter
  }
});

// Check if cancelable
if (event.cancelable) {
  event.preventDefault();
}
```

---

## Event Delegation

Handle events on parent for multiple children.

```javascript
// Without delegation - inefficient
document.querySelectorAll('.item').forEach(item => {
  item.addEventListener('click', handleClick);
});
// Problem: Many listeners, doesn't work for dynamic elements

// With delegation - efficient
document.querySelector('.list').addEventListener('click', (e) => {
  // Check if clicked element matches
  if (e.target.matches('.item')) {
    handleClick(e);
  }

  // Or find closest matching ancestor
  const item = e.target.closest('.item');
  if (item) {
    handleClick(e, item);
  }
});

// Benefits:
// 1. Single listener
// 2. Works with dynamic elements
// 3. Less memory
// 4. Cleaner code

// Practical example: Todo list
const todoList = document.querySelector('.todo-list');

todoList.addEventListener('click', (e) => {
  // Delete button
  if (e.target.matches('.delete-btn')) {
    const todo = e.target.closest('.todo-item');
    todo.remove();
  }

  // Edit button
  if (e.target.matches('.edit-btn')) {
    const todo = e.target.closest('.todo-item');
    editTodo(todo);
  }

  // Checkbox
  if (e.target.matches('.todo-checkbox')) {
    const todo = e.target.closest('.todo-item');
    todo.classList.toggle('completed');
  }
});
```

---

## Common Events

### Mouse Events

```javascript
element.addEventListener('click', fn);       // Click (mousedown + mouseup)
element.addEventListener('dblclick', fn);    // Double click
element.addEventListener('mousedown', fn);   // Button pressed
element.addEventListener('mouseup', fn);     // Button released
element.addEventListener('mouseenter', fn);  // Pointer enters (no bubble)
element.addEventListener('mouseleave', fn);  // Pointer leaves (no bubble)
element.addEventListener('mouseover', fn);   // Pointer enters (bubbles)
element.addEventListener('mouseout', fn);    // Pointer leaves (bubbles)
element.addEventListener('mousemove', fn);   // Pointer moves
element.addEventListener('contextmenu', fn); // Right click

// Drag events
element.addEventListener('dragstart', fn);
element.addEventListener('drag', fn);
element.addEventListener('dragend', fn);
element.addEventListener('dragenter', fn);
element.addEventListener('dragover', fn);
element.addEventListener('dragleave', fn);
element.addEventListener('drop', fn);
```

### Keyboard Events

```javascript
element.addEventListener('keydown', fn);   // Key pressed
element.addEventListener('keyup', fn);     // Key released
element.addEventListener('keypress', fn);  // Deprecated

// Example: Handle keyboard shortcuts
document.addEventListener('keydown', (e) => {
  // Ctrl/Cmd + S
  if ((e.ctrlKey || e.metaKey) && e.key === 's') {
    e.preventDefault();
    save();
  }

  // Escape
  if (e.key === 'Escape') {
    closeModal();
  }

  // Arrow keys
  if (e.key === 'ArrowUp') {
    navigate('up');
  }
});
```

### Form Events

```javascript
form.addEventListener('submit', fn);      // Form submitted
input.addEventListener('input', fn);      // Value changed (immediate)
input.addEventListener('change', fn);     // Value changed (on blur)
input.addEventListener('focus', fn);      // Element focused
input.addEventListener('blur', fn);       // Element lost focus
input.addEventListener('focusin', fn);    // Focus (bubbles)
input.addEventListener('focusout', fn);   // Blur (bubbles)
select.addEventListener('change', fn);    // Selection changed
form.addEventListener('reset', fn);       // Form reset

// Example: Form validation
form.addEventListener('submit', (e) => {
  e.preventDefault();

  const formData = new FormData(form);
  const data = Object.fromEntries(formData);

  if (validate(data)) {
    submitForm(data);
  }
});

// Real-time validation
input.addEventListener('input', (e) => {
  const isValid = validateField(e.target.value);
  e.target.classList.toggle('invalid', !isValid);
});
```

### Window/Document Events

```javascript
// Load events
window.addEventListener('load', fn);           // All resources loaded
window.addEventListener('DOMContentLoaded', fn); // DOM ready (faster)
window.addEventListener('beforeunload', fn);   // Before leaving page
window.addEventListener('unload', fn);         // Page unloading

// Visibility
document.addEventListener('visibilitychange', fn);  // Tab visibility
window.addEventListener('focus', fn);
window.addEventListener('blur', fn);

// Scroll
window.addEventListener('scroll', fn);
element.addEventListener('scroll', fn);

// Resize
window.addEventListener('resize', fn);

// Error
window.addEventListener('error', fn);
window.addEventListener('unhandledrejection', fn);  // Unhandled Promise

// Example: Lazy loading
window.addEventListener('scroll', () => {
  const images = document.querySelectorAll('img[data-src]');
  images.forEach(img => {
    if (isInViewport(img)) {
      img.src = img.dataset.src;
      img.removeAttribute('data-src');
    }
  });
});

// Better: IntersectionObserver (see DOM file)
```

### Touch Events

```javascript
element.addEventListener('touchstart', fn);  // Touch starts
element.addEventListener('touchmove', fn);   // Touch moves
element.addEventListener('touchend', fn);    // Touch ends
element.addEventListener('touchcancel', fn); // Touch cancelled

// Touch event properties
element.addEventListener('touchstart', (e) => {
  const touch = e.touches[0];  // First touch point
  touch.clientX;
  touch.clientY;

  e.touches;        // All current touches
  e.targetTouches;  // Touches on target element
  e.changedTouches; // Touches that changed
});

// Pointer Events (unified mouse + touch + pen)
element.addEventListener('pointerdown', fn);
element.addEventListener('pointermove', fn);
element.addEventListener('pointerup', fn);
element.addEventListener('pointercancel', fn);
element.addEventListener('pointerenter', fn);
element.addEventListener('pointerleave', fn);
```

---

## Custom Events

```javascript
// Create custom event
const event = new CustomEvent('myEvent', {
  detail: { message: 'Hello', data: 123 },
  bubbles: true,
  cancelable: true
});

// Dispatch event
element.dispatchEvent(event);

// Listen for custom event
element.addEventListener('myEvent', (e) => {
  console.log(e.detail.message);  // 'Hello'
});

// Practical example: Component communication
class CartWidget {
  addItem(item) {
    // Add to cart...

    // Notify other components
    const event = new CustomEvent('cart:updated', {
      detail: { item, count: this.items.length },
      bubbles: true
    });
    this.element.dispatchEvent(event);
  }
}

class HeaderWidget {
  constructor(element) {
    document.addEventListener('cart:updated', (e) => {
      this.updateCartCount(e.detail.count);
    });
  }
}
```

---

## Event Patterns

### Debounced Events

```javascript
function debounce(fn, delay) {
  let timeoutId;
  return function(...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Search input
const searchInput = document.querySelector('#search');
const debouncedSearch = debounce((query) => {
  fetch(`/api/search?q=${query}`);
}, 300);

searchInput.addEventListener('input', (e) => {
  debouncedSearch(e.target.value);
});
```

### Throttled Events

```javascript
function throttle(fn, limit) {
  let inThrottle;
  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

// Scroll handler
const throttledScroll = throttle(() => {
  console.log('Scroll position:', window.scrollY);
}, 100);

window.addEventListener('scroll', throttledScroll);
```

### Once Pattern

```javascript
// Using options
element.addEventListener('click', handler, { once: true });

// Manual implementation
function once(fn) {
  let called = false;
  return function(...args) {
    if (!called) {
      called = true;
      fn.apply(this, args);
    }
  };
}

element.addEventListener('click', once(handler));
```

### Event Bus / PubSub

```javascript
class EventBus {
  constructor() {
    this.listeners = {};
  }

  on(event, callback) {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event].push(callback);

    // Return unsubscribe function
    return () => this.off(event, callback);
  }

  off(event, callback) {
    if (this.listeners[event]) {
      this.listeners[event] = this.listeners[event]
        .filter(cb => cb !== callback);
    }
  }

  emit(event, data) {
    if (this.listeners[event]) {
      this.listeners[event].forEach(callback => callback(data));
    }
  }

  once(event, callback) {
    const wrapper = (data) => {
      callback(data);
      this.off(event, wrapper);
    };
    this.on(event, wrapper);
  }
}

const bus = new EventBus();
const unsubscribe = bus.on('user:login', (user) => {
  console.log('User logged in:', user);
});
bus.emit('user:login', { name: 'John' });
unsubscribe();
```

---

## Interview Questions

### Q1: What's event delegation and why use it?

```javascript
// Delegation: Handle events on parent for children
// Benefits: Fewer listeners, works with dynamic elements, less memory

list.addEventListener('click', (e) => {
  if (e.target.matches('.item')) {
    // Handle item click
  }
});
```

### Q2: Difference between target and currentTarget?

```javascript
// target: Element that triggered the event
// currentTarget: Element with the listener attached

parent.addEventListener('click', (e) => {
  // Click on child element
  e.target;        // child (where click happened)
  e.currentTarget; // parent (where listener is)
});
```

### Q3: Implement a custom event emitter

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
  }

  on(event, listener) {
    (this.events[event] ||= []).push(listener);
    return () => this.off(event, listener);
  }

  off(event, listener) {
    this.events[event] = this.events[event]?.filter(l => l !== listener);
  }

  emit(event, ...args) {
    this.events[event]?.forEach(listener => listener(...args));
  }
}
```

### Q4: Capture vs Bubble phase?

```javascript
// Capture: Top-down (window → target)
// Bubble: Bottom-up (target → window)
// Default is bubble phase

// Use capture
element.addEventListener('click', handler, true);
element.addEventListener('click', handler, { capture: true });
```

---

## Key Takeaways

1. **Use addEventListener** over inline handlers
2. **Event delegation** for dynamic elements
3. **stopPropagation** stops bubbling; **preventDefault** stops default action
4. **target** = where event happened; **currentTarget** = where listener is
5. **Debounce** for input events; **throttle** for scroll/resize
6. **Custom events** for component communication
7. Use **AbortController** for easy listener cleanup
8. Prefer **pointer events** over mouse/touch for cross-device support
