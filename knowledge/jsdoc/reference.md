# JSDoc Reference

> Source: https://jsdoc.app/

## Basic Syntax

```javascript
/**
 * Description of the function.
 * @param {string} name - The name parameter
 * @returns {string} The greeting
 */
function greet(name) {
  return `Hello, ${name}!`;
}
```

## Common Tags

### @param

```javascript
/**
 * @param {string} name - Simple parameter
 * @param {string} [name] - Optional parameter
 * @param {string} [name="default"] - Optional with default
 * @param {Object} options - Object parameter
 * @param {string} options.id - Property of object
 * @param {number} [options.count=1] - Optional property
 * @param {...string} names - Rest parameter
 */
```

### @returns / @return

```javascript
/**
 * @returns {string} Description of return value
 * @returns {Promise<User>} Async return
 * @returns {string|null} Multiple types
 * @returns {void} No return value
 */
```

### @type

```javascript
/** @type {string} */
let name;

/** @type {Array<number>} */
let numbers;

/** @type {Object.<string, number>} */
let scores;

/** @type {function(string): boolean} */
let validator;
```

### @typedef

```javascript
/**
 * @typedef {Object} User
 * @property {string} id - User ID
 * @property {string} name - User name
 * @property {string} [email] - Optional email
 * @property {number} age - User age
 */

/**
 * @param {User} user
 */
function saveUser(user) {}
```

### @callback

```javascript
/**
 * @callback RequestCallback
 * @param {Error|null} error - Error object
 * @param {Response} response - Response object
 * @returns {void}
 */

/**
 * @param {string} url
 * @param {RequestCallback} callback
 */
function fetch(url, callback) {}
```

### @template (Generics)

```javascript
/**
 * @template T
 * @param {T[]} array
 * @returns {T}
 */
function first(array) {
  return array[0];
}

/**
 * @template {string} K
 * @template V
 * @param {K} key
 * @param {V} value
 * @returns {Record<K, V>}
 */
function createRecord(key, value) {
  return { [key]: value };
}
```

### @throws

```javascript
/**
 * @throws {TypeError} If name is not a string
 * @throws {RangeError} If age is negative
 */
function validate(name, age) {}
```

### @example

```javascript
/**
 * Adds two numbers.
 * @param {number} a - First number
 * @param {number} b - Second number
 * @returns {number} Sum of a and b
 * @example
 * // Basic usage
 * add(1, 2); // returns 3
 *
 * @example
 * // With negative numbers
 * add(-1, 5); // returns 4
 */
function add(a, b) {
  return a + b;
}
```

### @deprecated

```javascript
/**
 * @deprecated Use newFunction() instead
 */
function oldFunction() {}

/**
 * @deprecated Since v2.0. Will be removed in v3.0
 */
```

### @see / @link

```javascript
/**
 * @see {@link https://example.com} External link
 * @see OtherClass
 * @see OtherClass#method
 */

/**
 * Similar to {@link Array.prototype.map}
 */
```

## Class Documentation

```javascript
/**
 * Represents a user in the system.
 * @class
 * @extends BaseEntity
 * @implements {Serializable}
 */
class User extends BaseEntity {
  /**
   * Create a user.
   * @param {string} name - The user's name
   * @param {number} age - The user's age
   */
  constructor(name, age) {
    super();
    /**
     * The user's name
     * @type {string}
     * @public
     */
    this.name = name;

    /**
     * The user's age
     * @type {number}
     * @private
     */
    this._age = age;
  }

  /**
   * Get user's display name.
   * @returns {string} Display name
   * @readonly
   */
  get displayName() {
    return this.name.toUpperCase();
  }

  /**
   * Save user to database.
   * @async
   * @returns {Promise<void>}
   * @fires User#saved
   */
  async save() {}

  /**
   * Create user from JSON.
   * @static
   * @param {Object} json - JSON object
   * @returns {User} New user instance
   */
  static fromJSON(json) {}
}
```

## Module Documentation

```javascript
/**
 * Utility functions for string manipulation.
 * @module utils/string
 */

/**
 * @exports capitalize
 */
export function capitalize(str) {}

/**
 * @exports
 * @default
 */
export default class StringUtils {}
```

## Complex Types

### Union Types

```javascript
/** @type {string|number} */
let id;

/** @param {string|string[]} input */
function process(input) {}
```

### Nullable Types

```javascript
/** @type {?string} */
let nullable;

/** @type {!string} */
let nonNullable;
```

### Array Types

```javascript
/** @type {string[]} */
let strings;

/** @type {Array<string>} */
let alsoStrings;

/** @type {Array<Array<number>>} */
let matrix;
```

### Object Types

```javascript
/** @type {{name: string, age: number}} */
let person;

/** @type {Object<string, User>} */
let userMap;
```

### Function Types

```javascript
/** @type {function(): void} */
let noArgs;

/** @type {function(string, number): boolean} */
let withArgs;

/** @type {function(...string): void} */
let restArgs;
```

## Access Modifiers

```javascript
/**
 * @public - Accessible everywhere
 * @private - Only within class
 * @protected - Within class and subclasses
 * @package - Within same package/module
 * @access private - Alternative syntax
 */
```

## Configuration (jsdoc.json)

```json
{
  "source": {
    "include": ["src"],
    "includePattern": ".+\\.js(doc|x)?$",
    "excludePattern": "(^|\\/|\\\\)_"
  },
  "opts": {
    "destination": "./docs",
    "recurse": true,
    "template": "templates/default"
  },
  "plugins": ["plugins/markdown"],
  "templates": {
    "cleverLinks": true,
    "monospaceLinks": true
  },
  "tags": {
    "allowUnknownTags": true
  }
}
```

## Generate Documentation

```bash
# Install JSDoc
npm install --save-dev jsdoc

# Generate docs
npx jsdoc src -r -d docs

# With config file
npx jsdoc -c jsdoc.json
```
