# DOM Manipulation

## Overview

The Document Object Model (DOM) is the browser's representation of HTML as a tree of objects. Understanding DOM manipulation is essential for frontend development.

---

## Selecting Elements

### Modern Methods

```javascript
// Single element
const element = document.querySelector('.class');
const element = document.querySelector('#id');
const element = document.querySelector('div.class > p');
const element = document.querySelector('[data-id="123"]');

// Multiple elements (NodeList)
const elements = document.querySelectorAll('.class');
const elements = document.querySelectorAll('div, p, span');

// Convert NodeList to Array
const array = Array.from(elements);
const array = [...elements];

// Iterate NodeList
elements.forEach(el => console.log(el));
```

### Classic Methods

```javascript
// By ID (single element)
const element = document.getElementById('myId');

// By class (HTMLCollection - live)
const elements = document.getElementsByClassName('myClass');

// By tag (HTMLCollection - live)
const elements = document.getElementsByTagName('div');

// By name attribute
const elements = document.getElementsByName('myName');
```

### NodeList vs HTMLCollection

```javascript
// NodeList (querySelectorAll) - static snapshot
const nodeList = document.querySelectorAll('.item');
document.body.appendChild(document.createElement('div'));
// nodeList still has original elements

// HTMLCollection (getElementsByClassName) - live
const collection = document.getElementsByClassName('item');
// collection updates when DOM changes

// Both are array-like but not arrays
nodeList.forEach(el => {});  // Works
// collection.forEach(el => {});  // Doesn't work

// Convert to array for full array methods
Array.from(collection).forEach(el => {});
```

### Traversing DOM

```javascript
const element = document.querySelector('.target');

// Parent
element.parentElement;
element.parentNode;
element.closest('.ancestor');  // Finds nearest ancestor matching selector

// Children
element.children;              // HTMLCollection of child elements
element.childNodes;            // NodeList including text nodes
element.firstElementChild;
element.lastElementChild;
element.firstChild;            // Might be text node
element.lastChild;

// Siblings
element.nextElementSibling;
element.previousElementSibling;
element.nextSibling;           // Might be text node
element.previousSibling;

// Find within element
element.querySelector('.child');
element.querySelectorAll('.child');
```

---

## Creating and Modifying Elements

### Creating Elements

```javascript
// Create element
const div = document.createElement('div');
div.id = 'myDiv';
div.className = 'container active';
div.textContent = 'Hello World';

// Create text node
const text = document.createTextNode('Hello');

// Create from HTML string
const template = document.createElement('template');
template.innerHTML = '<div class="card"><h2>Title</h2></div>';
const element = template.content.firstElementChild;

// Clone element
const clone = element.cloneNode(true);   // Deep clone (with children)
const shallow = element.cloneNode(false); // Shallow clone

// Document fragment (efficient for multiple insertions)
const fragment = document.createDocumentFragment();
for (let i = 0; i < 100; i++) {
  const li = document.createElement('li');
  li.textContent = `Item ${i}`;
  fragment.appendChild(li);
}
document.querySelector('ul').appendChild(fragment);
```

### Inserting Elements

```javascript
const parent = document.querySelector('.parent');
const child = document.createElement('div');
const reference = document.querySelector('.reference');

// Append (at end)
parent.appendChild(child);
parent.append(child, 'text', anotherChild);  // Multiple items

// Prepend (at start)
parent.prepend(child);

// Insert before reference
parent.insertBefore(child, reference);

// Insert at specific position
reference.insertAdjacentElement('beforebegin', child); // Before reference
reference.insertAdjacentElement('afterbegin', child);  // First child of reference
reference.insertAdjacentElement('beforeend', child);   // Last child of reference
reference.insertAdjacentElement('afterend', child);    // After reference

// Insert HTML
element.insertAdjacentHTML('beforeend', '<span>HTML</span>');

// Insert text
element.insertAdjacentText('afterbegin', 'Text');

// Replace
parent.replaceChild(newChild, oldChild);
oldChild.replaceWith(newChild);
```

### Removing Elements

```javascript
// Remove element
element.remove();

// Remove from parent
parent.removeChild(child);

// Remove all children
parent.innerHTML = '';
// Or more efficiently:
while (parent.firstChild) {
  parent.removeChild(parent.firstChild);
}
// Or using replaceChildren:
parent.replaceChildren();
```

---

## Modifying Content

### Text Content

```javascript
// textContent - all text (including hidden)
element.textContent = 'New text';
const text = element.textContent;

// innerText - visible text only (triggers reflow)
element.innerText = 'New text';
const visibleText = element.innerText;

// innerHTML - HTML content
element.innerHTML = '<strong>Bold</strong> text';
const html = element.innerHTML;

// outerHTML - includes the element itself
element.outerHTML = '<div class="new">Replaced</div>';

// Security warning: Never use innerHTML with user input
// element.innerHTML = userInput;  // XSS vulnerability!

// Safe alternative
element.textContent = userInput;  // Escapes HTML
```

### Attributes

```javascript
// Get/Set/Remove attributes
element.getAttribute('data-id');
element.setAttribute('data-id', '123');
element.removeAttribute('data-id');
element.hasAttribute('data-id');

// Direct property access
element.id = 'newId';
element.className = 'class1 class2';
element.title = 'Tooltip';
element.hidden = true;

// Data attributes
element.dataset.userId = '123';  // Sets data-user-id="123"
const userId = element.dataset.userId;

// Boolean attributes
element.disabled = true;
element.checked = true;
element.readonly = true;

// All attributes
const attrs = element.attributes;  // NamedNodeMap
for (const attr of attrs) {
  console.log(attr.name, attr.value);
}
```

### Classes

```javascript
const element = document.querySelector('.target');

// classList API
element.classList.add('active');
element.classList.add('class1', 'class2');
element.classList.remove('active');
element.classList.toggle('active');
element.classList.toggle('active', condition);  // Force add/remove
element.classList.contains('active');
element.classList.replace('old', 'new');

// Get all classes
const classes = [...element.classList];

// className (full string)
element.className = 'class1 class2';
element.className += ' class3';
```

### Styles

```javascript
// Inline styles
element.style.color = 'red';
element.style.backgroundColor = 'blue';  // camelCase
element.style.fontSize = '16px';
element.style.cssText = 'color: red; font-size: 16px;';

// Get computed styles (actual rendered values)
const styles = getComputedStyle(element);
const color = styles.color;
const fontSize = styles.fontSize;

// CSS custom properties (variables)
element.style.setProperty('--my-color', 'red');
const value = element.style.getPropertyValue('--my-color');
element.style.removeProperty('--my-color');

// Get from computed
const customProp = getComputedStyle(element).getPropertyValue('--my-color');
```

---

## Element Dimensions and Position

### Dimensions

```javascript
// Content dimensions (no padding, border, scrollbar)
element.clientWidth;
element.clientHeight;

// Full dimensions (including padding, border, scrollbar)
element.offsetWidth;
element.offsetHeight;

// Scrollable content dimensions
element.scrollWidth;
element.scrollHeight;

// getBoundingClientRect (relative to viewport)
const rect = element.getBoundingClientRect();
rect.top;     // Distance from viewport top
rect.left;    // Distance from viewport left
rect.bottom;  // Distance from viewport top to element bottom
rect.right;   // Distance from viewport left to element right
rect.width;   // Element width
rect.height;  // Element height
rect.x;       // Same as left
rect.y;       // Same as top
```

### Position

```javascript
// Offset from positioned parent
element.offsetTop;
element.offsetLeft;
element.offsetParent;  // Nearest positioned ancestor

// Scroll position
element.scrollTop;
element.scrollLeft;

// Scroll container
document.documentElement.scrollTop;  // Page scroll
window.scrollY;  // Same as above
window.pageYOffset;  // Legacy alias

// Scroll element into view
element.scrollIntoView();
element.scrollIntoView({ behavior: 'smooth', block: 'center' });

// Check if in viewport
function isInViewport(element) {
  const rect = element.getBoundingClientRect();
  return (
    rect.top >= 0 &&
    rect.left >= 0 &&
    rect.bottom <= (window.innerHeight || document.documentElement.clientHeight) &&
    rect.right <= (window.innerWidth || document.documentElement.clientWidth)
  );
}
```

---

## Forms

```javascript
// Access form elements
const form = document.querySelector('form');
form.elements;                    // All form elements
form.elements.username;           // By name
form.elements['user-email'];      // By name (with hyphen)

// Get/Set values
const input = form.elements.username;
input.value;                      // Current value
input.defaultValue;               // Original value
input.value = 'new value';

// Checkbox/Radio
const checkbox = form.elements.agree;
checkbox.checked;                 // Boolean
checkbox.checked = true;

// Select
const select = form.elements.country;
select.value;                     // Selected value
select.selectedIndex;             // Index of selected option
select.options;                   // All options
select.options[select.selectedIndex].text;  // Selected text

// Multiple select
const multiSelect = form.elements.colors;
const selectedValues = [...multiSelect.selectedOptions].map(opt => opt.value);

// Form data
const formData = new FormData(form);
formData.get('username');
formData.getAll('colors');  // For multiple values
formData.set('username', 'newValue');
formData.append('extra', 'value');

// Convert to object
const data = Object.fromEntries(formData);

// Submit
form.submit();                    // No submit event
form.requestSubmit();             // Triggers submit event

// Reset
form.reset();

// Validation
input.validity;                   // ValidityState object
input.validity.valid;
input.validity.valueMissing;
input.validity.typeMismatch;
input.checkValidity();            // Returns boolean
input.reportValidity();           // Shows validation message
input.setCustomValidity('Custom error message');
```

---

## Common Patterns

### Safe HTML Insertion

```javascript
// Never do this with user input
element.innerHTML = userInput;  // XSS!

// Safe alternatives
function escapeHtml(str) {
  const div = document.createElement('div');
  div.textContent = str;
  return div.innerHTML;
}

// Or use textContent
element.textContent = userInput;

// For complex HTML, use template + textContent for data
function createCard(title, content) {
  const card = document.createElement('div');
  card.className = 'card';

  const h2 = document.createElement('h2');
  h2.textContent = title;  // Safe

  const p = document.createElement('p');
  p.textContent = content;  // Safe

  card.append(h2, p);
  return card;
}
```

### Efficient DOM Updates

```javascript
// Bad - multiple reflows
for (let i = 0; i < 1000; i++) {
  document.body.appendChild(document.createElement('div'));
}

// Good - batch with fragment
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  fragment.appendChild(document.createElement('div'));
}
document.body.appendChild(fragment);

// Good - batch with innerHTML (careful with XSS)
const html = items.map(item => `<div>${escapeHtml(item)}</div>`).join('');
container.innerHTML = html;

// Good - modify detached element
const element = parent.removeChild(child);
// Make many changes...
parent.appendChild(element);
```

### Observer Patterns

```javascript
// MutationObserver - watch for DOM changes
const observer = new MutationObserver((mutations) => {
  mutations.forEach(mutation => {
    if (mutation.type === 'childList') {
      console.log('Children changed');
    }
    if (mutation.type === 'attributes') {
      console.log('Attribute changed:', mutation.attributeName);
    }
  });
});

observer.observe(element, {
  childList: true,     // Watch for child additions/removals
  attributes: true,    // Watch for attribute changes
  subtree: true,       // Watch all descendants
  attributeFilter: ['class', 'style'],  // Only these attributes
  characterData: true  // Watch text content changes
});

// Stop observing
observer.disconnect();

// IntersectionObserver - visibility detection
const intersectionObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      console.log('Element is visible');
      entry.target.classList.add('visible');
    }
  });
}, {
  threshold: 0.5,  // 50% visible
  rootMargin: '100px'  // Trigger 100px before entering viewport
});

intersectionObserver.observe(element);

// ResizeObserver - size changes
const resizeObserver = new ResizeObserver((entries) => {
  entries.forEach(entry => {
    console.log('New size:', entry.contentRect.width, entry.contentRect.height);
  });
});

resizeObserver.observe(element);
```

---

## Interview Questions

### Q1: Difference between innerHTML and textContent?

```javascript
// innerHTML: Parses and renders HTML, XSS risk
// textContent: Plain text, escapes HTML, safer and faster
// innerText: Like textContent but only visible text (triggers reflow)
```

### Q2: Implement element creation from HTML string

```javascript
function createElementFromHTML(htmlString) {
  const template = document.createElement('template');
  template.innerHTML = htmlString.trim();
  return template.content.firstChild;
}

const element = createElementFromHTML('<div class="card">Content</div>');
```

### Q3: How would you efficiently add 1000 items to a list?

```javascript
function addItems(items, container) {
  const fragment = document.createDocumentFragment();

  items.forEach(item => {
    const li = document.createElement('li');
    li.textContent = item;
    fragment.appendChild(li);
  });

  container.appendChild(fragment);
}
```

### Q4: Implement a function to check if element is visible

```javascript
function isVisible(element) {
  const style = getComputedStyle(element);

  return (
    style.display !== 'none' &&
    style.visibility !== 'hidden' &&
    style.opacity !== '0' &&
    element.offsetParent !== null
  );
}
```

---

## Key Takeaways

1. **querySelector/querySelectorAll** are preferred for selection
2. Use **DocumentFragment** for batch insertions
3. **textContent** is safer than innerHTML
4. **classList** API for class manipulation
5. **getBoundingClientRect** for element dimensions
6. **MutationObserver** for watching DOM changes
7. **IntersectionObserver** for visibility detection
8. Minimize DOM access - cache references
