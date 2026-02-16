# JavaScript Modules

## Overview

Modules allow you to split code into separate files, manage dependencies, and create reusable components. Understanding different module systems is essential for JavaScript interviews.

---

## ES6 Modules (ESM)

The standard module system for JavaScript.

### Named Exports

```javascript
// math.js
export const PI = 3.14159;

export function add(a, b) {
  return a + b;
}

export function multiply(a, b) {
  return a * b;
}

// Can also export at the end
const subtract = (a, b) => a - b;
const divide = (a, b) => a / b;

export { subtract, divide };

// Rename on export
export { subtract as minus };
```

### Named Imports

```javascript
// app.js
import { add, multiply } from './math.js';

add(2, 3);      // 5
multiply(2, 3); // 6

// Import all as namespace
import * as math from './math.js';
math.add(2, 3);
math.PI;

// Rename on import
import { add as sum } from './math.js';
sum(2, 3);  // 5

// Mix named imports
import { add, PI } from './math.js';
```

### Default Export

```javascript
// logger.js
export default function log(message) {
  console.log(`[LOG] ${message}`);
}

// Can combine with named exports
export const LOG_LEVELS = ['info', 'warn', 'error'];

// app.js
import log from './logger.js';           // Default
import log, { LOG_LEVELS } from './logger.js';  // Both

// Default can be named anything
import logger from './logger.js';
logger('Hello');
```

### Re-exports

```javascript
// utils/index.js - barrel file
export { add, subtract } from './math.js';
export { formatDate } from './date.js';
export { default as Logger } from './logger.js';

// Re-export all
export * from './math.js';

// Re-export with rename
export { add as sum } from './math.js';

// app.js - single import point
import { add, formatDate, Logger } from './utils/index.js';
```

### Dynamic Imports

```javascript
// Lazy loading modules
async function loadModule() {
  const module = await import('./heavy-module.js');
  module.doSomething();
}

// Conditional loading
if (condition) {
  const { feature } = await import('./feature.js');
  feature();
}

// With destructuring
const { default: Logger, LOG_LEVELS } = await import('./logger.js');

// Code splitting in frameworks
// React
const LazyComponent = React.lazy(() => import('./Component'));

// Route-based splitting
const routes = {
  '/dashboard': () => import('./pages/Dashboard'),
  '/profile': () => import('./pages/Profile')
};
```

### Module Characteristics

```javascript
// 1. Strict mode by default
// No need for 'use strict'

// 2. Module-level scope
const privateVar = 'not accessible outside';

// 3. Single evaluation
// Module code runs once, even if imported multiple times

// 4. Live bindings
// counter.js
export let count = 0;
export function increment() {
  count++;
}

// app.js
import { count, increment } from './counter.js';
console.log(count);  // 0
increment();
console.log(count);  // 1 (live binding!)

// 5. Static structure
// Imports/exports must be at top level (not in conditionals)
// This enables tree shaking
```

---

## CommonJS (CJS)

Used in Node.js (before ESM support).

### Exports

```javascript
// math.js
const add = (a, b) => a + b;
const subtract = (a, b) => a - b;

// Named exports
module.exports.add = add;
module.exports.subtract = subtract;

// Or using exports shorthand
exports.add = add;
exports.subtract = subtract;

// Or export object
module.exports = {
  add,
  subtract
};

// Single/default export
module.exports = function logger(msg) {
  console.log(msg);
};
```

### Require

```javascript
// Import entire module
const math = require('./math');
math.add(2, 3);

// Destructure
const { add, subtract } = require('./math');
add(2, 3);

// Single export
const logger = require('./logger');
logger('Hello');
```

### Key Differences from ESM

```javascript
// 1. Synchronous loading
const module = require('./module');  // Blocks until loaded

// 2. Dynamic - can be conditional
if (condition) {
  const feature = require('./feature');
}

// 3. Copy, not live binding
// counter.js
let count = 0;
module.exports = {
  count,
  increment() { count++; }
};

// app.js
const counter = require('./counter');
console.log(counter.count);  // 0
counter.increment();
console.log(counter.count);  // 0 (still! it's a copy)

// 4. Can modify exports
const mod = require('./module');
mod.newProp = 'added';  // Works
```

---

## ESM vs CommonJS

| Feature | ESM | CommonJS |
|---------|-----|----------|
| Syntax | import/export | require/module.exports |
| Loading | Async | Sync |
| Structure | Static | Dynamic |
| Bindings | Live | Copy |
| Top-level await | Yes | No |
| Tree shaking | Yes | Limited |
| Browser support | Native | Needs bundler |

### Interoperability

```javascript
// ESM importing CJS (Node.js)
import cjsModule from './cjs-module.cjs';  // Gets default
import { named } from './cjs-module.cjs';  // May work

// CJS importing ESM (complex)
// Must use dynamic import
async function loadEsm() {
  const esmModule = await import('./esm-module.mjs');
}

// package.json configuration
{
  "type": "module",  // .js files are ESM
  // or
  "type": "commonjs"  // .js files are CJS (default)
}

// File extensions
// .mjs - always ESM
// .cjs - always CommonJS
```

---

## Module Patterns (Pre-ES6)

### IIFE Module Pattern

```javascript
const Module = (function() {
  // Private
  let privateVar = 0;

  function privateMethod() {
    return privateVar;
  }

  // Public API
  return {
    increment() {
      privateVar++;
    },
    getValue() {
      return privateMethod();
    }
  };
})();

Module.increment();
Module.getValue();  // 1
Module.privateVar;  // undefined
```

### Revealing Module Pattern

```javascript
const Calculator = (function() {
  let result = 0;

  function add(x) {
    result += x;
    return this;
  }

  function subtract(x) {
    result -= x;
    return this;
  }

  function getResult() {
    return result;
  }

  function reset() {
    result = 0;
    return this;
  }

  // Reveal public API
  return {
    add,
    subtract,
    getResult,
    reset
  };
})();

Calculator.add(5).subtract(2).getResult();  // 3
```

---

## AMD (Asynchronous Module Definition)

Used with RequireJS (historical).

```javascript
// Define module
define('math', [], function() {
  return {
    add: function(a, b) { return a + b; }
  };
});

// Define with dependencies
define('app', ['math', 'logger'], function(math, logger) {
  return {
    run: function() {
      logger.log(math.add(2, 3));
    }
  };
});

// Use module
require(['app'], function(app) {
  app.run();
});
```

---

## UMD (Universal Module Definition)

Works in multiple environments.

```javascript
(function(root, factory) {
  if (typeof define === 'function' && define.amd) {
    // AMD
    define(['dependency'], factory);
  } else if (typeof module === 'object' && module.exports) {
    // CommonJS
    module.exports = factory(require('dependency'));
  } else {
    // Browser global
    root.MyModule = factory(root.Dependency);
  }
}(typeof self !== 'undefined' ? self : this, function(dependency) {
  // Module code
  return {
    doSomething: function() {}
  };
}));
```

---

## Tree Shaking

Eliminating unused exports.

```javascript
// utils.js
export function used() {
  return 'used';
}

export function unused() {
  return 'unused';
}

// app.js
import { used } from './utils.js';
used();

// After tree shaking, 'unused' is removed from bundle

// Things that prevent tree shaking:
// 1. Side effects
export const config = setupConfig();  // Runs on import

// 2. Dynamic access
import * as utils from './utils.js';
utils[dynamicKey]();  // Can't statically analyze

// 3. CommonJS
const { used } = require('./utils');  // Can't tree shake

// package.json sideEffects
{
  "sideEffects": false,  // All files are pure
  // or
  "sideEffects": ["*.css", "*.scss"]  // These have side effects
}
```

---

## Circular Dependencies

When modules depend on each other.

```javascript
// a.js
import { b } from './b.js';
export const a = 'A';
console.log('a.js:', b);

// b.js
import { a } from './a.js';
export const b = 'B';
console.log('b.js:', a);

// ESM handles this with live bindings
// But can cause issues with initialization order

// Solutions:
// 1. Restructure to avoid circles
// 2. Use dependency injection
// 3. Lazy evaluation

// Lazy evaluation fix
// a.js
import { getB } from './b.js';
export const a = 'A';
export function getA() { return a; }
console.log('a.js:', getB());

// b.js
import { getA } from './a.js';
export const b = 'B';
export function getB() { return b; }
console.log('b.js:', getA());
```

---

## Best Practices

### 1. Prefer Named Exports

```javascript
// Good - explicit and tree-shakeable
export function processData(data) {}
export function formatData(data) {}

// Avoid - default exports can't be tree-shaken
export default {
  processData,
  formatData
};
```

### 2. Use Barrel Files Carefully

```javascript
// utils/index.js
export * from './string.js';
export * from './number.js';
export * from './array.js';

// Can hurt tree shaking if not careful
// Better: direct imports for large modules
import { capitalize } from './utils/string.js';
```

### 3. Consistent Module Structure

```javascript
// 1. Imports at top
import { dep1 } from './dep1.js';
import { dep2 } from './dep2.js';

// 2. Constants
const CONFIG = {};

// 3. Main implementation
function mainFunction() {}

// 4. Exports at bottom (or with declarations)
export { mainFunction };
```

### 4. Avoid Circular Dependencies

```javascript
// Use dependency injection
class Service {
  setDependency(dep) {
    this.dep = dep;
  }
}

// Or event-based communication
import { EventEmitter } from 'events';
export const bus = new EventEmitter();
```

---

## Interview Questions

### Q1: Difference between ESM and CommonJS?

```javascript
// ESM: import/export, async, static, live bindings, tree-shakeable
// CJS: require/module.exports, sync, dynamic, copy values
```

### Q2: What is tree shaking?

```javascript
// Removal of unused exports during bundling
// Requires: ESM, static analysis, pure functions
// Enabled by sideEffects: false in package.json
```

### Q3: How do you handle circular dependencies?

```javascript
// 1. Restructure code to eliminate cycles
// 2. Use lazy evaluation (functions instead of values)
// 3. Dependency injection
// 4. Event-based communication
```

### Q4: Default vs named exports - when to use which?

```javascript
// Named exports:
// - Multiple exports per file
// - Better tree shaking
// - Explicit imports
// - IDE autocomplete

// Default exports:
// - Single main export (class, function)
// - Cleaner import syntax
// - Common in React components
```

### Q5: Implement a simple module system

```javascript
const ModuleSystem = (function() {
  const modules = {};

  function define(name, deps, factory) {
    modules[name] = {
      deps,
      factory,
      exports: null
    };
  }

  function require(name) {
    const module = modules[name];

    if (!module) {
      throw new Error(`Module ${name} not found`);
    }

    if (!module.exports) {
      const deps = module.deps.map(require);
      module.exports = module.factory(...deps);
    }

    return module.exports;
  }

  return { define, require };
})();

// Usage
ModuleSystem.define('math', [], function() {
  return { add: (a, b) => a + b };
});

ModuleSystem.define('app', ['math'], function(math) {
  return { result: math.add(2, 3) };
});

const app = ModuleSystem.require('app');
console.log(app.result);  // 5
```

---

## Key Takeaways

1. **ESM** is the standard - use it for new projects
2. **Named exports** are preferred for tree shaking
3. **Dynamic imports** enable code splitting
4. **Circular dependencies** should be avoided/managed carefully
5. **CommonJS** still used in Node.js ecosystem
6. **Tree shaking** requires static structure and pure modules
7. **Barrel files** simplify imports but use carefully
