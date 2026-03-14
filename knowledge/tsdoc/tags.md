# TSDoc Tags Reference

> Source: https://tsdoc.org/pages/tags/

## Tag Categories

| Category | Tags |
|----------|------|
| **Block** | @deprecated, @example, @param, @remarks, @returns, @see, @throws, @typeParam, @defaultValue |
| **Modifier** | @alpha, @beta, @internal, @public, @readonly, @sealed, @virtual, @override |
| **Inline** | @link, @inheritDoc, @label |
| **Special** | @packageDocumentation, @eventProperty |

## Block Tags

### @param

Documents a function parameter.

```typescript
/**
 * @param name - User's full name
 * @param options - Configuration object
 * @param options.format - Output format
 */
function greet(name: string, options?: { format?: string }): string;
```

### @returns

Documents the return value.

```typescript
/**
 * @returns The sum of all numbers, or 0 for empty arrays
 */
function sum(numbers: number[]): number;
```

### @typeParam

Documents a type parameter (generic).

```typescript
/**
 * @typeParam T - The element type
 * @typeParam K - The key type, defaults to string
 */
class Map<T, K = string> {}
```

### @throws

Documents exceptions that may be thrown.

```typescript
/**
 * @throws {@link Error} - When input is invalid
 * @throws {@link NetworkError} - When connection fails
 */
function fetchData(url: string): Promise<Data>;
```

### @example

Provides usage examples with code.

```typescript
/**
 * @example
 * Simple usage:
 * ```ts
 * add(1, 2); // Returns 3
 * ```
 *
 * @example
 * With negative numbers:
 * ```ts
 * add(-1, 1); // Returns 0
 * ```
 */
function add(a: number, b: number): number;
```

### @remarks

Extended discussion and implementation notes.

```typescript
/**
 * Compresses data using gzip.
 *
 * @remarks
 * This method uses Node.js zlib under the hood.
 * For browser usage, consider using a polyfill.
 *
 * Memory usage scales with input size. For files
 * larger than 100MB, use streamCompress() instead.
 */
function compress(data: Buffer): Buffer;
```

### @see

References related documentation.

```typescript
/**
 * @see {@link OtherClass}
 * @see {@link OtherClass.method}
 * @see {@link https://example.com/docs | External Docs}
 */
```

### @deprecated

Marks as deprecated with migration guidance.

```typescript
/**
 * @deprecated Use `newMethod()` instead. Will be removed in v3.0.
 */
function oldMethod(): void;
```

### @defaultValue

Documents default values for optional parameters.

```typescript
interface Config {
  /**
   * Server port.
   * @defaultValue 3000
   */
  port?: number;

  /**
   * Enable SSL.
   * @defaultValue false
   */
  ssl?: boolean;
}
```

## Modifier Tags

### @public

Indicates stable public API.

```typescript
/**
 * @public
 */
export function stableFunction(): void;
```

### @beta

Preview API, may have breaking changes.

```typescript
/**
 * @beta
 * This API is in preview. Feedback welcome.
 */
export function previewFeature(): void;
```

### @alpha

Early preview, expect significant changes.

```typescript
/**
 * @alpha
 * Experimental. Not ready for production.
 */
export function experimentalApi(): void;
```

### @internal

Excluded from public documentation.

```typescript
/**
 * @internal
 * Used internally by the library.
 */
export function _internalHelper(): void;
```

### @readonly

Indicates read-only property or value.

```typescript
class Config {
  /**
   * @readonly
   * The application version.
   */
  readonly version: string;
}
```

### @sealed

Class should not be extended.

```typescript
/**
 * @sealed
 */
class FinalClass {}
```

### @virtual

Method can be overridden by subclasses.

```typescript
class Base {
  /**
   * @virtual
   */
  process(): void {}
}
```

### @override

Method overrides a base class method.

```typescript
class Derived extends Base {
  /**
   * @override
   */
  process(): void {}
}
```

## Inline Tags

### @link

Creates links to other symbols or URLs.

```typescript
/**
 * Returns data like {@link Array.prototype.map}.
 *
 * See the {@link https://example.com | API docs} for details.
 *
 * Works with {@link Config} objects.
 */
```

### @inheritDoc

Copies documentation from another symbol.

```typescript
interface IService {
  /**
   * Initializes the service.
   * @param config - Service configuration
   */
  init(config: Config): void;
}

class MyService implements IService {
  /**
   * {@inheritDoc IService.init}
   */
  init(config: Config): void {}
}
```

### @label

Creates a named reference point.

```typescript
/**
 * {@label MAIN_ENTRY}
 * Main entry point.
 */
function main(): void;

// Reference elsewhere: {@link main:MAIN_ENTRY}
```

## Special Tags

### @packageDocumentation

Documents the entire package (entry file only).

```typescript
/**
 * # My Package
 *
 * A utility library for data processing.
 *
 * ## Installation
 * ```bash
 * npm install my-package
 * ```
 *
 * ## Quick Start
 * ```typescript
 * import { process } from 'my-package';
 * process(data);
 * ```
 *
 * @packageDocumentation
 */

export * from './processor';
```

### @eventProperty

Documents an event property.

```typescript
class WebSocket {
  /**
   * @eventProperty
   * Fired when a message is received.
   */
  onMessage: ((data: string) => void) | null;

  /**
   * @eventProperty
   * Fired when connection opens.
   */
  onOpen: (() => void) | null;
}
```

## Tag Combinations

### Common Patterns

```typescript
/**
 * Fetches user data from the API.
 *
 * @remarks
 * This method caches results for 5 minutes.
 *
 * @example
 * ```typescript
 * const user = await fetchUser('123');
 * console.log(user.name);
 * ```
 *
 * @param id - The user ID
 * @returns The user object
 *
 * @throws {@link NotFoundError} - User doesn't exist
 * @throws {@link NetworkError} - API unavailable
 *
 * @see {@link updateUser} to modify user data
 *
 * @public
 */
async function fetchUser(id: string): Promise<User>;
```

### Interface Documentation

```typescript
/**
 * Configuration options for the logger.
 *
 * @remarks
 * All properties are optional and have sensible defaults.
 *
 * @public
 */
interface LoggerOptions {
  /**
   * Log level threshold.
   * @defaultValue "info"
   */
  level?: 'debug' | 'info' | 'warn' | 'error';

  /**
   * Output destination.
   * @defaultValue process.stdout
   */
  output?: NodeJS.WriteStream;

  /**
   * Include timestamps.
   * @defaultValue true
   */
  timestamps?: boolean;
}
```
