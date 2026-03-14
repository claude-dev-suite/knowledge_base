# TSDoc Syntax

> Source: https://tsdoc.org/

## What is TSDoc?

TSDoc is a standardized syntax for TypeScript doc comments. It's used by:
- API Extractor
- TypeDoc
- VS Code IntelliSense
- Documentation generators

## Basic Syntax

```typescript
/**
 * Summary paragraph - first paragraph is the summary.
 *
 * @remarks
 * Additional details go here in the remarks section.
 * This can span multiple paragraphs.
 *
 * @example
 * ```typescript
 * const result = myFunction('hello');
 * ```
 *
 * @param input - Description of input parameter
 * @returns Description of return value
 */
function myFunction(input: string): string {
  return input.toUpperCase();
}
```

## Core Tags

### @param

```typescript
/**
 * @param name - The user's name (required)
 * @param age - The user's age
 * @param options - Configuration options
 * @param options.timeout - Timeout in milliseconds
 */
function createUser(
  name: string,
  age: number,
  options?: { timeout?: number }
): User {}
```

### @returns

```typescript
/**
 * @returns The computed hash value
 */
function hash(input: string): string {}

/**
 * @returns A promise that resolves to the user data
 */
async function fetchUser(id: string): Promise<User> {}
```

### @remarks

```typescript
/**
 * Calculates the factorial of a number.
 *
 * @remarks
 * This implementation uses recursion. For large numbers,
 * consider using an iterative approach to avoid stack overflow.
 *
 * The function throws for negative inputs.
 */
function factorial(n: number): number {}
```

### @example

```typescript
/**
 * Formats a date string.
 *
 * @example
 * Basic usage:
 * ```typescript
 * formatDate(new Date()); // "2024-01-15"
 * ```
 *
 * @example
 * With custom format:
 * ```typescript
 * formatDate(new Date(), 'MM/DD/YYYY'); // "01/15/2024"
 * ```
 */
function formatDate(date: Date, format?: string): string {}
```

### @throws

```typescript
/**
 * @throws {@link InvalidArgumentError}
 * Thrown if the input is negative
 *
 * @throws {@link NetworkError}
 * Thrown if the API request fails
 */
function processData(input: number): void {}
```

### @see

```typescript
/**
 * @see {@link OtherClass} for related functionality
 * @see {@link https://example.com/docs | External Documentation}
 */
```

### @deprecated

```typescript
/**
 * @deprecated Use {@link newFunction} instead.
 * This will be removed in version 3.0.
 */
function oldFunction(): void {}
```

## Modifier Tags

### @public, @internal, @alpha, @beta

```typescript
/**
 * @public
 * Stable API for external use.
 */
export function stableApi(): void {}

/**
 * @beta
 * API is in preview and may change.
 */
export function previewApi(): void {}

/**
 * @alpha
 * Early preview, expect breaking changes.
 */
export function earlyApi(): void {}

/**
 * @internal
 * Not part of public API.
 */
export function internalHelper(): void {}
```

### @readonly

```typescript
/**
 * @readonly
 * The current version number.
 */
export const VERSION: string = '1.0.0';

class Config {
  /**
   * @readonly
   */
  readonly name: string;
}
```

### @sealed, @virtual, @override

```typescript
/**
 * @sealed
 * This class should not be extended.
 */
class FinalClass {}

class Base {
  /**
   * @virtual
   * Subclasses can override this method.
   */
  process(): void {}
}

class Derived extends Base {
  /**
   * @override
   */
  process(): void {}
}
```

### @packageDocumentation

```typescript
// At the top of index.ts
/**
 * A library for working with dates.
 *
 * @remarks
 * This package provides utilities for date formatting,
 * parsing, and manipulation.
 *
 * @packageDocumentation
 */

export * from './format';
export * from './parse';
```

## Inline Tags

### @link

```typescript
/**
 * Similar to {@link Array.prototype.map}.
 *
 * See {@link https://example.com | the docs} for more info.
 *
 * Uses the {@link Config} object for settings.
 */
```

### @inheritDoc

```typescript
interface IProcessor {
  /**
   * Processes the input data.
   * @param data - The data to process
   * @returns The processed result
   */
  process(data: string): string;
}

class MyProcessor implements IProcessor {
  /** {@inheritDoc IProcessor.process} */
  process(data: string): string {
    return data.toUpperCase();
  }
}
```

### @label

```typescript
/**
 * {@label MAIN_FUNCTION}
 * The main entry point.
 */
function main(): void {}

// Reference: {@link main:MAIN_FUNCTION}
```

## Block Tags

### @typeParam

```typescript
/**
 * A generic container class.
 *
 * @typeParam T - The type of items in the container
 * @typeParam K - The key type for lookups
 */
class Container<T, K extends string = string> {
  /**
   * @typeParam U - Transform output type
   */
  map<U>(fn: (item: T) => U): Container<U, K> {}
}
```

### @defaultValue

```typescript
/**
 * @param timeout - Request timeout
 * @defaultValue 30000
 */
function request(url: string, timeout = 30000): Promise<Response> {}

interface Options {
  /**
   * Enable debug mode.
   * @defaultValue false
   */
  debug?: boolean;
}
```

### @eventProperty

```typescript
class EventEmitter {
  /**
   * @eventProperty
   * Fired when data is received.
   */
  onData: (data: string) => void;
}
```

## Documentation Structure

### Summary Section

```typescript
/**
 * First paragraph is the summary. Keep it brief.
 *
 * Second paragraph is additional details but still
 * part of the general description.
 *
 * @remarks
 * The remarks section is for implementation notes,
 * caveats, and extended discussion.
 */
```

### Complete Example

```typescript
/**
 * Parses a JSON configuration file.
 *
 * Reads the specified file, parses it as JSON, and validates
 * it against the expected schema.
 *
 * @remarks
 * The parser supports both JSON and JSON5 formats. Comments
 * are allowed in JSON5 mode.
 *
 * For large files, consider using {@link parseConfigStream}
 * which processes the file incrementally.
 *
 * @example
 * ```typescript
 * const config = await parseConfig('./config.json');
 * console.log(config.database.host);
 * ```
 *
 * @param filePath - Path to the configuration file
 * @param options - Parser options
 * @param options.strict - Enable strict validation
 * @param options.json5 - Enable JSON5 parsing
 *
 * @returns The parsed configuration object
 *
 * @throws {@link FileNotFoundError}
 * Thrown if the file doesn't exist
 *
 * @throws {@link ValidationError}
 * Thrown if validation fails
 *
 * @see {@link validateConfig} for validation details
 * @see {@link https://json5.org | JSON5 Spec}
 *
 * @public
 */
async function parseConfig(
  filePath: string,
  options?: ParseOptions
): Promise<Config> {}
```
