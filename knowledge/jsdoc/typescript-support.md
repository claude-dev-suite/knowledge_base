# JSDoc TypeScript Support

> Source: https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html

## Type Checking JavaScript with JSDoc

TypeScript can type-check JavaScript files using JSDoc comments.

### Enable in tsconfig.json

```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,
    "noEmit": true
  }
}
```

### Or use // @ts-check

```javascript
// @ts-check

/** @type {string} */
let name = 123; // Error: Type 'number' is not assignable to type 'string'
```

## Supported JSDoc Tags

### @type

```javascript
/** @type {string} */
let str;

/** @type {Window} */
let win;

/** @type {PromiseLike<string>} */
let promise;

/** @type {HTMLElement} */
let element;
```

### @param and @returns

```javascript
/**
 * @param {string} p1 - String param
 * @param {string=} p2 - Optional param (syntax 1)
 * @param {string} [p3] - Optional param (syntax 2)
 * @param {string} [p4="default"] - Optional with default
 * @returns {string}
 */
function fn(p1, p2, p3, p4 = "default") {
  return p1 + (p2 || "") + (p3 || "") + p4;
}
```

### @typedef and @property

```javascript
/**
 * @typedef {Object} SpecialType
 * @property {string} prop1 - Required property
 * @property {number} [prop2] - Optional property
 * @property {number=} prop3 - Optional (alternate syntax)
 */

/** @type {SpecialType} */
const obj = { prop1: "hello" };
```

### @template (Generics)

```javascript
/**
 * @template T
 * @param {T} x
 * @returns {T}
 */
function identity(x) {
  return x;
}

const num = identity(42); // type: number
const str = identity("hi"); // type: string
```

### Generic Constraints

```javascript
/**
 * @template {string} K - Must extend string
 * @template {{ name: string }} T - Must have name property
 * @param {K} key
 * @param {T} obj
 */
function getProperty(key, obj) {
  return obj[key];
}
```

### @satisfies

```javascript
/** @satisfies {Record<string, unknown>} */
const config = {
  name: "app",
  version: 1
};
// config.name is typed as string, not unknown
```

### @enum

```javascript
/** @enum {number} */
const Status = {
  Pending: 0,
  Active: 1,
  Completed: 2
};

/** @type {Status} */
let status = Status.Active;
```

### @this

```javascript
/**
 * @this {HTMLElement}
 */
function handleClick() {
  this.classList.add('clicked');
}
```

### @overload

```javascript
/**
 * @overload
 * @param {string} x
 * @returns {string}
 */
/**
 * @overload
 * @param {number} x
 * @returns {number}
 */
/**
 * @param {string | number} x
 * @returns {string | number}
 */
function process(x) {
  return x;
}
```

## Importing Types

### From TypeScript Files

```javascript
/** @typedef {import('./types').User} User */

/** @type {User} */
const user = { id: 1, name: "John" };

// Inline import
/** @type {import('./types').Config} */
const config = {};
```

### From npm Packages

```javascript
/** @typedef {import('express').Request} Request */
/** @typedef {import('express').Response} Response */

/**
 * @param {Request} req
 * @param {Response} res
 */
function handler(req, res) {}
```

## Type Assertions

```javascript
/** @type {any} */
let value = getValue();

// Cast to specific type
const user = /** @type {User} */ (value);

// Const assertion
const arr = /** @type {const} */ ([1, 2, 3]);
// Type: readonly [1, 2, 3]
```

## Class Patterns

```javascript
/**
 * @template T
 */
class Container {
  /**
   * @param {T} value
   */
  constructor(value) {
    /** @type {T} */
    this.value = value;
  }

  /**
   * @returns {T}
   */
  getValue() {
    return this.value;
  }
}

/** @type {Container<string>} */
const strContainer = new Container("hello");
```

## Extending Types

```javascript
/**
 * @typedef {Object} BaseConfig
 * @property {string} name
 */

/**
 * @typedef {BaseConfig & { port: number }} ServerConfig
 */

/** @type {ServerConfig} */
const config = { name: "server", port: 3000 };
```

## Utility Types

```javascript
/** @type {Partial<User>} */
let partialUser;

/** @type {Required<User>} */
let requiredUser;

/** @type {Readonly<User>} */
let readonlyUser;

/** @type {Pick<User, 'id' | 'name'>} */
let pickedUser;

/** @type {Omit<User, 'password'>} */
let safeUser;

/** @type {Record<string, number>} */
let scores;

/** @type {ReturnType<typeof fn>} */
let result;

/** @type {Parameters<typeof fn>} */
let params;
```

## Compiler Directives

```javascript
// @ts-check - Enable type checking
// @ts-nocheck - Disable type checking
// @ts-ignore - Ignore next line
// @ts-expect-error - Expect error on next line
```

## Best Practices

### 1. Use Type Imports for Complex Types

```javascript
// Good: Import from .d.ts or .ts files
/** @typedef {import('./types').User} User */

// Avoid: Redefining types that exist elsewhere
```

### 2. Prefer @typedef for Reusable Types

```javascript
// types.js
/**
 * @typedef {Object} ApiResponse
 * @property {boolean} success
 * @property {any} data
 * @property {string} [error]
 */

// usage.js
/** @typedef {import('./types').ApiResponse} ApiResponse */
```

### 3. Use .d.ts Files for Complex Type Definitions

```typescript
// types.d.ts
export interface User {
  id: string;
  name: string;
  email: string;
}

export type UserRole = 'admin' | 'user' | 'guest';
```

```javascript
// app.js
/** @typedef {import('./types').User} User */
/** @typedef {import('./types').UserRole} UserRole */
```
