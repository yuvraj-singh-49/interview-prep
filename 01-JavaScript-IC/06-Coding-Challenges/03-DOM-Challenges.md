# DOM Manipulation Challenges

## Overview

DOM manipulation challenges test practical frontend skills. These are common interview problems involving element creation, event handling, and interactive UI components.

---

## Virtual DOM Implementation

### Simple Virtual DOM

```javascript
// Virtual node structure
function createElement(type, props, ...children) {
  return {
    type,
    props: props || {},
    children: children.flat()
  };
}

// Render virtual node to real DOM
function render(vnode) {
  // Text node
  if (typeof vnode === 'string' || typeof vnode === 'number') {
    return document.createTextNode(vnode);
  }

  // Element node
  const element = document.createElement(vnode.type);

  // Set properties
  Object.entries(vnode.props).forEach(([key, value]) => {
    if (key.startsWith('on')) {
      const event = key.slice(2).toLowerCase();
      element.addEventListener(event, value);
    } else if (key === 'className') {
      element.className = value;
    } else if (key === 'style' && typeof value === 'object') {
      Object.assign(element.style, value);
    } else {
      element.setAttribute(key, value);
    }
  });

  // Render children
  vnode.children.forEach(child => {
    element.appendChild(render(child));
  });

  return element;
}

// Diff algorithm
function diff(oldVNode, newVNode) {
  // Different types - replace
  if (typeof oldVNode !== typeof newVNode) {
    return { type: 'REPLACE', newVNode };
  }

  // Text nodes
  if (typeof newVNode === 'string' || typeof newVNode === 'number') {
    if (oldVNode !== newVNode) {
      return { type: 'TEXT', content: newVNode };
    }
    return null;
  }

  // Different element types
  if (oldVNode.type !== newVNode.type) {
    return { type: 'REPLACE', newVNode };
  }

  // Same type - diff props and children
  const propPatches = diffProps(oldVNode.props, newVNode.props);
  const childPatches = diffChildren(oldVNode.children, newVNode.children);

  if (propPatches.length === 0 && childPatches.length === 0) {
    return null;
  }

  return { type: 'UPDATE', propPatches, childPatches };
}

function diffProps(oldProps, newProps) {
  const patches = [];

  // Check for changed/new props
  Object.keys(newProps).forEach(key => {
    if (oldProps[key] !== newProps[key]) {
      patches.push({ type: 'SET', key, value: newProps[key] });
    }
  });

  // Check for removed props
  Object.keys(oldProps).forEach(key => {
    if (!(key in newProps)) {
      patches.push({ type: 'REMOVE', key });
    }
  });

  return patches;
}

function diffChildren(oldChildren, newChildren) {
  const patches = [];
  const maxLen = Math.max(oldChildren.length, newChildren.length);

  for (let i = 0; i < maxLen; i++) {
    patches.push(diff(oldChildren[i], newChildren[i]));
  }

  return patches;
}

// Apply patches to DOM
function patch(element, patches) {
  if (!patches) return element;

  switch (patches.type) {
    case 'REPLACE':
      const newElement = render(patches.newVNode);
      element.parentNode.replaceChild(newElement, element);
      return newElement;

    case 'TEXT':
      element.textContent = patches.content;
      return element;

    case 'UPDATE':
      // Apply prop patches
      patches.propPatches.forEach(p => {
        if (p.type === 'SET') {
          if (p.key.startsWith('on')) {
            // Event handlers need special handling
          } else {
            element.setAttribute(p.key, p.value);
          }
        } else {
          element.removeAttribute(p.key);
        }
      });

      // Apply child patches
      const childNodes = Array.from(element.childNodes);
      patches.childPatches.forEach((childPatch, i) => {
        if (childPatch) {
          if (childNodes[i]) {
            patch(childNodes[i], childPatch);
          } else if (childPatch.type === 'REPLACE') {
            element.appendChild(render(childPatch.newVNode));
          }
        }
      });

      return element;
  }
}

// Usage
const vdom1 = createElement('div', { className: 'container' },
  createElement('h1', null, 'Hello'),
  createElement('p', null, 'World')
);

const container = document.getElementById('app');
const dom = render(vdom1);
container.appendChild(dom);
```

---

## Infinite Scroll

```javascript
class InfiniteScroll {
  constructor(options) {
    this.container = options.container;
    this.loadMore = options.loadMore;
    this.threshold = options.threshold || 200;
    this.loading = false;
    this.hasMore = true;

    this.init();
  }

  init() {
    // Create sentinel element
    this.sentinel = document.createElement('div');
    this.sentinel.className = 'sentinel';
    this.container.appendChild(this.sentinel);

    // Use Intersection Observer
    this.observer = new IntersectionObserver(
      entries => this.handleIntersect(entries),
      { rootMargin: `${this.threshold}px` }
    );

    this.observer.observe(this.sentinel);
  }

  async handleIntersect(entries) {
    if (!entries[0].isIntersecting || this.loading || !this.hasMore) {
      return;
    }

    this.loading = true;
    this.showLoader();

    try {
      const { items, hasMore } = await this.loadMore();
      this.hasMore = hasMore;
      this.renderItems(items);
    } catch (error) {
      console.error('Failed to load more:', error);
    }

    this.hideLoader();
    this.loading = false;
  }

  renderItems(items) {
    const fragment = document.createDocumentFragment();

    items.forEach(item => {
      const element = this.createItemElement(item);
      fragment.appendChild(element);
    });

    this.container.insertBefore(fragment, this.sentinel);
  }

  createItemElement(item) {
    const div = document.createElement('div');
    div.className = 'item';
    div.textContent = item.text;
    return div;
  }

  showLoader() {
    if (!this.loader) {
      this.loader = document.createElement('div');
      this.loader.className = 'loader';
      this.loader.textContent = 'Loading...';
    }
    this.container.insertBefore(this.loader, this.sentinel);
  }

  hideLoader() {
    this.loader?.remove();
  }

  destroy() {
    this.observer.disconnect();
    this.sentinel.remove();
    this.loader?.remove();
  }
}

// Usage
let page = 0;

const scroll = new InfiniteScroll({
  container: document.getElementById('list'),
  loadMore: async () => {
    const response = await fetch(`/api/items?page=${++page}`);
    const data = await response.json();
    return {
      items: data.items,
      hasMore: data.hasNextPage
    };
  }
});
```

---

## Drag and Drop

```javascript
class DragAndDrop {
  constructor(container) {
    this.container = container;
    this.dragged = null;
    this.placeholder = null;

    this.init();
  }

  init() {
    this.container.addEventListener('dragstart', this.handleDragStart.bind(this));
    this.container.addEventListener('dragend', this.handleDragEnd.bind(this));
    this.container.addEventListener('dragover', this.handleDragOver.bind(this));
    this.container.addEventListener('drop', this.handleDrop.bind(this));

    // Make items draggable
    this.container.querySelectorAll('.draggable').forEach(item => {
      item.setAttribute('draggable', 'true');
    });
  }

  handleDragStart(e) {
    if (!e.target.classList.contains('draggable')) return;

    this.dragged = e.target;
    this.dragged.classList.add('dragging');

    // Create placeholder
    this.placeholder = document.createElement('div');
    this.placeholder.className = 'placeholder';
    this.placeholder.style.height = `${this.dragged.offsetHeight}px`;

    e.dataTransfer.effectAllowed = 'move';
    e.dataTransfer.setData('text/plain', e.target.id);

    // Delay to allow drag image to be captured
    setTimeout(() => {
      this.dragged.style.display = 'none';
    }, 0);
  }

  handleDragEnd(e) {
    this.dragged.style.display = '';
    this.dragged.classList.remove('dragging');
    this.placeholder?.remove();
    this.dragged = null;
    this.placeholder = null;
  }

  handleDragOver(e) {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'move';

    const target = this.getDragTarget(e.target);
    if (!target || target === this.dragged) return;

    const rect = target.getBoundingClientRect();
    const midY = rect.top + rect.height / 2;

    if (e.clientY < midY) {
      target.parentNode.insertBefore(this.placeholder, target);
    } else {
      target.parentNode.insertBefore(this.placeholder, target.nextSibling);
    }
  }

  handleDrop(e) {
    e.preventDefault();

    if (this.placeholder && this.dragged) {
      this.placeholder.parentNode.insertBefore(this.dragged, this.placeholder);
    }
  }

  getDragTarget(element) {
    while (element && element !== this.container) {
      if (element.classList.contains('draggable')) {
        return element;
      }
      element = element.parentNode;
    }
    return null;
  }
}

// Usage
const dnd = new DragAndDrop(document.getElementById('sortable-list'));
```

---

## Modal Component

```javascript
class Modal {
  constructor(options = {}) {
    this.onClose = options.onClose;
    this.closeOnOverlay = options.closeOnOverlay !== false;
    this.closeOnEscape = options.closeOnEscape !== false;

    this.create();
    this.attachEvents();
  }

  create() {
    // Overlay
    this.overlay = document.createElement('div');
    this.overlay.className = 'modal-overlay';

    // Modal
    this.modal = document.createElement('div');
    this.modal.className = 'modal';
    this.modal.setAttribute('role', 'dialog');
    this.modal.setAttribute('aria-modal', 'true');

    // Close button
    this.closeBtn = document.createElement('button');
    this.closeBtn.className = 'modal-close';
    this.closeBtn.innerHTML = '&times;';
    this.closeBtn.setAttribute('aria-label', 'Close modal');

    // Content container
    this.content = document.createElement('div');
    this.content.className = 'modal-content';

    this.modal.appendChild(this.closeBtn);
    this.modal.appendChild(this.content);
    this.overlay.appendChild(this.modal);
  }

  attachEvents() {
    this.closeBtn.addEventListener('click', () => this.close());

    if (this.closeOnOverlay) {
      this.overlay.addEventListener('click', (e) => {
        if (e.target === this.overlay) {
          this.close();
        }
      });
    }

    if (this.closeOnEscape) {
      this.handleKeyDown = (e) => {
        if (e.key === 'Escape') {
          this.close();
        }
      };
    }
  }

  open(content) {
    if (typeof content === 'string') {
      this.content.innerHTML = content;
    } else {
      this.content.innerHTML = '';
      this.content.appendChild(content);
    }

    document.body.appendChild(this.overlay);
    document.body.style.overflow = 'hidden';

    if (this.handleKeyDown) {
      document.addEventListener('keydown', this.handleKeyDown);
    }

    // Focus trap
    this.previousActiveElement = document.activeElement;
    this.closeBtn.focus();

    // Animate in
    requestAnimationFrame(() => {
      this.overlay.classList.add('open');
    });
  }

  close() {
    this.overlay.classList.remove('open');

    this.overlay.addEventListener('transitionend', () => {
      this.overlay.remove();
      document.body.style.overflow = '';

      if (this.handleKeyDown) {
        document.removeEventListener('keydown', this.handleKeyDown);
      }

      this.previousActiveElement?.focus();
      this.onClose?.();
    }, { once: true });
  }
}

// Usage
const modal = new Modal({
  onClose: () => console.log('Modal closed')
});

document.getElementById('open-modal').addEventListener('click', () => {
  modal.open('<h2>Hello!</h2><p>This is a modal.</p>');
});
```

---

## Autocomplete / Typeahead

```javascript
class Autocomplete {
  constructor(input, options) {
    this.input = input;
    this.fetchSuggestions = options.fetchSuggestions;
    this.onSelect = options.onSelect;
    this.minChars = options.minChars || 2;
    this.debounceMs = options.debounceMs || 300;

    this.selectedIndex = -1;
    this.suggestions = [];

    this.create();
    this.attachEvents();
  }

  create() {
    // Wrapper
    this.wrapper = document.createElement('div');
    this.wrapper.className = 'autocomplete-wrapper';
    this.input.parentNode.insertBefore(this.wrapper, this.input);
    this.wrapper.appendChild(this.input);

    // Dropdown
    this.dropdown = document.createElement('ul');
    this.dropdown.className = 'autocomplete-dropdown';
    this.dropdown.setAttribute('role', 'listbox');
    this.wrapper.appendChild(this.dropdown);

    // ARIA
    this.input.setAttribute('role', 'combobox');
    this.input.setAttribute('aria-autocomplete', 'list');
    this.input.setAttribute('aria-expanded', 'false');
  }

  attachEvents() {
    // Debounced input handler
    this.debouncedFetch = this.debounce(async (value) => {
      if (value.length < this.minChars) {
        this.hideSuggestions();
        return;
      }

      this.suggestions = await this.fetchSuggestions(value);
      this.renderSuggestions();
    }, this.debounceMs);

    this.input.addEventListener('input', (e) => {
      this.debouncedFetch(e.target.value);
    });

    this.input.addEventListener('keydown', (e) => {
      this.handleKeyDown(e);
    });

    this.input.addEventListener('blur', () => {
      // Delay to allow click on suggestion
      setTimeout(() => this.hideSuggestions(), 200);
    });

    this.dropdown.addEventListener('click', (e) => {
      const item = e.target.closest('.autocomplete-item');
      if (item) {
        this.selectItem(parseInt(item.dataset.index));
      }
    });
  }

  handleKeyDown(e) {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        this.selectedIndex = Math.min(
          this.selectedIndex + 1,
          this.suggestions.length - 1
        );
        this.highlightItem();
        break;

      case 'ArrowUp':
        e.preventDefault();
        this.selectedIndex = Math.max(this.selectedIndex - 1, -1);
        this.highlightItem();
        break;

      case 'Enter':
        if (this.selectedIndex >= 0) {
          e.preventDefault();
          this.selectItem(this.selectedIndex);
        }
        break;

      case 'Escape':
        this.hideSuggestions();
        break;
    }
  }

  renderSuggestions() {
    if (this.suggestions.length === 0) {
      this.hideSuggestions();
      return;
    }

    this.dropdown.innerHTML = this.suggestions
      .map((item, i) => `
        <li class="autocomplete-item" data-index="${i}" role="option">
          ${this.highlightMatch(item.text, this.input.value)}
        </li>
      `)
      .join('');

    this.selectedIndex = -1;
    this.dropdown.classList.add('open');
    this.input.setAttribute('aria-expanded', 'true');
  }

  highlightMatch(text, query) {
    const regex = new RegExp(`(${this.escapeRegex(query)})`, 'gi');
    return text.replace(regex, '<mark>$1</mark>');
  }

  escapeRegex(str) {
    return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  }

  highlightItem() {
    const items = this.dropdown.querySelectorAll('.autocomplete-item');

    items.forEach((item, i) => {
      item.classList.toggle('highlighted', i === this.selectedIndex);
    });
  }

  selectItem(index) {
    const item = this.suggestions[index];
    this.input.value = item.text;
    this.hideSuggestions();
    this.onSelect?.(item);
  }

  hideSuggestions() {
    this.dropdown.classList.remove('open');
    this.dropdown.innerHTML = '';
    this.input.setAttribute('aria-expanded', 'false');
  }

  debounce(fn, delay) {
    let timeoutId;
    return (...args) => {
      clearTimeout(timeoutId);
      timeoutId = setTimeout(() => fn(...args), delay);
    };
  }
}

// Usage
const autocomplete = new Autocomplete(document.getElementById('search'), {
  fetchSuggestions: async (query) => {
    const response = await fetch(`/api/search?q=${query}`);
    return response.json();
  },
  onSelect: (item) => {
    console.log('Selected:', item);
  }
});
```

---

## Form Validation

```javascript
class FormValidator {
  constructor(form, rules) {
    this.form = form;
    this.rules = rules;
    this.errors = {};

    this.init();
  }

  init() {
    this.form.addEventListener('submit', (e) => {
      e.preventDefault();
      if (this.validate()) {
        this.onSubmit?.(this.getValues());
      }
    });

    // Real-time validation
    Object.keys(this.rules).forEach(field => {
      const input = this.form.elements[field];
      if (input) {
        input.addEventListener('blur', () => this.validateField(field));
        input.addEventListener('input', () => this.clearError(field));
      }
    });
  }

  validate() {
    this.errors = {};

    Object.keys(this.rules).forEach(field => {
      this.validateField(field);
    });

    return Object.keys(this.errors).length === 0;
  }

  validateField(field) {
    const rules = this.rules[field];
    const input = this.form.elements[field];
    const value = input?.value?.trim() ?? '';

    for (const rule of rules) {
      const error = rule(value, this.getValues());
      if (error) {
        this.setError(field, error);
        return false;
      }
    }

    this.clearError(field);
    return true;
  }

  setError(field, message) {
    this.errors[field] = message;
    const input = this.form.elements[field];
    const errorEl = this.getOrCreateErrorElement(input);

    input.classList.add('error');
    errorEl.textContent = message;
  }

  clearError(field) {
    delete this.errors[field];
    const input = this.form.elements[field];
    const errorEl = input?.parentNode.querySelector('.error-message');

    input?.classList.remove('error');
    if (errorEl) errorEl.textContent = '';
  }

  getOrCreateErrorElement(input) {
    let errorEl = input.parentNode.querySelector('.error-message');
    if (!errorEl) {
      errorEl = document.createElement('span');
      errorEl.className = 'error-message';
      input.parentNode.appendChild(errorEl);
    }
    return errorEl;
  }

  getValues() {
    const formData = new FormData(this.form);
    return Object.fromEntries(formData);
  }
}

// Validation rules
const required = (field) => (value) =>
  !value ? `${field} is required` : null;

const minLength = (min) => (value) =>
  value.length < min ? `Must be at least ${min} characters` : null;

const email = (value) =>
  value && !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)
    ? 'Invalid email address'
    : null;

const matches = (field) => (value, values) =>
  value !== values[field] ? `Must match ${field}` : null;

// Usage
const validator = new FormValidator(document.getElementById('signup-form'), {
  username: [required('Username'), minLength(3)],
  email: [required('Email'), email],
  password: [required('Password'), minLength(8)],
  confirmPassword: [required('Confirm password'), matches('password')]
});

validator.onSubmit = (values) => {
  console.log('Form submitted:', values);
};
```

---

## DOM Tree Traversal

```javascript
// Find all text nodes
function getTextNodes(element) {
  const textNodes = [];

  function traverse(node) {
    if (node.nodeType === Node.TEXT_NODE) {
      if (node.textContent.trim()) {
        textNodes.push(node);
      }
    } else {
      node.childNodes.forEach(traverse);
    }
  }

  traverse(element);
  return textNodes;
}

// Find element by path
function getElementByPath(root, path) {
  return path.reduce((el, index) => {
    return el?.children[index];
  }, root);
}

// Get path to element
function getPathToElement(root, target) {
  const path = [];

  function traverse(element) {
    if (element === target) return true;

    for (let i = 0; i < element.children.length; i++) {
      path.push(i);
      if (traverse(element.children[i])) return true;
      path.pop();
    }

    return false;
  }

  traverse(root);
  return path;
}

// Find common ancestor
function findCommonAncestor(element1, element2) {
  const ancestors = new Set();

  let current = element1;
  while (current) {
    ancestors.add(current);
    current = current.parentElement;
  }

  current = element2;
  while (current) {
    if (ancestors.has(current)) {
      return current;
    }
    current = current.parentElement;
  }

  return null;
}

// Serialize DOM to JSON
function serializeDOM(element) {
  if (element.nodeType === Node.TEXT_NODE) {
    return element.textContent.trim() || null;
  }

  const result = {
    tag: element.tagName.toLowerCase(),
    attributes: {},
    children: []
  };

  // Get attributes
  for (const attr of element.attributes) {
    result.attributes[attr.name] = attr.value;
  }

  // Get children
  for (const child of element.childNodes) {
    const serialized = serializeDOM(child);
    if (serialized) {
      result.children.push(serialized);
    }
  }

  return result;
}

// Deserialize JSON to DOM
function deserializeDOM(json) {
  if (typeof json === 'string') {
    return document.createTextNode(json);
  }

  const element = document.createElement(json.tag);

  Object.entries(json.attributes).forEach(([key, value]) => {
    element.setAttribute(key, value);
  });

  json.children.forEach(child => {
    element.appendChild(deserializeDOM(child));
  });

  return element;
}
```

---

## Interview Questions

### Q1: Implement a function to highlight text in DOM

```javascript
function highlightText(root, searchText) {
  const regex = new RegExp(`(${searchText})`, 'gi');

  function traverse(node) {
    if (node.nodeType === Node.TEXT_NODE) {
      if (regex.test(node.textContent)) {
        const span = document.createElement('span');
        span.innerHTML = node.textContent.replace(regex, '<mark>$1</mark>');

        const fragment = document.createDocumentFragment();
        while (span.firstChild) {
          fragment.appendChild(span.firstChild);
        }

        node.parentNode.replaceChild(fragment, node);
      }
    } else {
      Array.from(node.childNodes).forEach(traverse);
    }
  }

  traverse(root);
}
```

### Q2: Implement a tooltip component

```javascript
class Tooltip {
  constructor() {
    this.tooltip = document.createElement('div');
    this.tooltip.className = 'tooltip';
    document.body.appendChild(this.tooltip);

    document.addEventListener('mouseover', this.show.bind(this));
    document.addEventListener('mouseout', this.hide.bind(this));
  }

  show(e) {
    const text = e.target.dataset.tooltip;
    if (!text) return;

    this.tooltip.textContent = text;
    this.tooltip.classList.add('visible');

    const rect = e.target.getBoundingClientRect();
    this.tooltip.style.left = `${rect.left + rect.width / 2}px`;
    this.tooltip.style.top = `${rect.top - 10}px`;
  }

  hide() {
    this.tooltip.classList.remove('visible');
  }
}
```

---

## Key Takeaways

1. **Event delegation**: Handle events on parent for dynamic content
2. **Intersection Observer**: Efficient scroll-based loading
3. **ARIA attributes**: Ensure accessibility
4. **Debounce input**: Prevent excessive API calls
5. **Focus management**: Trap focus in modals
6. **requestAnimationFrame**: Smooth animations
7. **Document fragments**: Batch DOM insertions
