# TypeScript Types - Comprehensive Guide

This comprehensive guide covers TypeScript's type system in depth, based on official TypeScript documentation. It serves as both a learning resource and a reference for everyday development.

---

## Table of Contents

1. [Primitive Types](#1-primitive-types)
2. [Special Types](#2-special-types)
3. [Literal Types](#3-literal-types)
4. [Arrays and Tuples](#4-arrays-and-tuples)
5. [Object Types](#5-object-types)
6. [Union and Intersection Types](#6-union-and-intersection-types)
7. [Type Narrowing](#7-type-narrowing)
8. [Type Assertions](#8-type-assertions)
9. [Type Guards](#9-type-guards)
10. [Conditional Types](#10-conditional-types)
11. [Template Literal Types](#11-template-literal-types)
12. [typeof and keyof Operators](#12-typeof-and-keyof-operators)
13. [Indexed Access Types](#13-indexed-access-types)
14. [Best Practices and Common Patterns](#14-best-practices-and-common-patterns)
15. [Common Pitfalls](#15-common-pitfalls)

---

## 1. Primitive Types

TypeScript includes all JavaScript primitive types with full type safety.

### String

```typescript
// String type for textual data
let name: string = "John";
let greeting: string = `Hello, ${name}!`;  // Template literals supported
let empty: string = "";

// String methods are fully typed
const upper: string = name.toUpperCase();
const length: number = name.length;
const char: string = name.charAt(0);

// Important: String (capital S) is the wrapper object type - avoid it
let primitiveString: string = "hello";     // Correct
let objectString: String = new String("hello");  // Avoid - wrapper object
```

### Number

```typescript
// Number type for all numeric values (integer and floating point)
let integer: number = 42;
let float: number = 3.14;
let negative: number = -100;
let hex: number = 0xff;        // Hexadecimal
let binary: number = 0b1010;   // Binary
let octal: number = 0o744;     // Octal
let exponent: number = 2.5e10; // Exponential notation

// Special numeric values
let inf: number = Infinity;
let negInf: number = -Infinity;
let notANumber: number = NaN;  // NaN is still type number

// Number precision considerations
console.log(0.1 + 0.2 === 0.3);  // false (floating point precision)
console.log(Number.MAX_SAFE_INTEGER);  // 9007199254740991
```

### BigInt

```typescript
// BigInt for arbitrarily large integers (ES2020+)
let big: bigint = 9007199254740991n;
let anotherBig: bigint = BigInt("9007199254740991");
let negative: bigint = -100n;

// BigInt arithmetic
let sum: bigint = 100n + 200n;
let product: bigint = 10n * 20n;

// Cannot mix BigInt and number without explicit conversion
// let mixed = 100n + 50;  // Error!
let converted: bigint = 100n + BigInt(50);  // OK

// BigInt does not support decimals
// let decimal: bigint = 3.14n;  // Error!
```

### Boolean

```typescript
// Boolean for true/false values
let isActive: boolean = true;
let isDisabled: boolean = false;

// Boolean expressions
let hasPermission: boolean = user.role === "admin";
let canEdit: boolean = isActive && hasPermission;

// Truthy/falsy behavior in conditionals
// TypeScript allows any type in conditionals, narrowing as needed
if (someValue) {
  // someValue could be truthy (non-empty string, non-zero number, etc.)
}
```

### Symbol

```typescript
// Symbol for unique identifiers (ES2015+)
let id: symbol = Symbol("id");
let anotherId: symbol = Symbol("id");
console.log(id === anotherId);  // false - symbols are always unique

// Using symbols as object keys
const KEY: unique symbol = Symbol("key");  // unique symbol type
interface HasKey {
  [KEY]: string;
}

// Well-known symbols are typed
const iterable = {
  [Symbol.iterator]: function* () {
    yield 1;
    yield 2;
  }
};

// unique symbol - a subtype of symbol for constant declarations
const sym1: unique symbol = Symbol();
// const sym2: unique symbol = sym1;  // Error - each unique symbol is distinct
```

### Null and Undefined

```typescript
// Null and undefined are distinct types
let nothing: null = null;
let notDefined: undefined = undefined;

// With strictNullChecks enabled (recommended):
let name: string = "John";
// name = null;       // Error - null not assignable to string
// name = undefined;  // Error - undefined not assignable to string

// Explicitly allow null or undefined with union types
let nullableName: string | null = "John";
nullableName = null;  // OK

let optionalValue: string | undefined = undefined;
optionalValue = "value";  // OK

// Checking for null/undefined
function greet(name: string | null | undefined) {
  if (name === null || name === undefined) {
    return "Hello, stranger!";
  }
  return `Hello, ${name}!`;
}

// Nullish coalescing operator (??)
const displayName = name ?? "Anonymous";  // Uses "Anonymous" if name is null/undefined

// Optional chaining (?.)
const length = name?.length;  // undefined if name is null/undefined
```

---

## 2. Special Types

### any

```typescript
// any - disables type checking (use sparingly)
let anything: any = "hello";
anything = 42;
anything = { foo: "bar" };
anything.nonExistentMethod();  // No error at compile time, crashes at runtime

// any is contagious - it propagates through operations
let result: any = anything + 1;  // result is also any

// Use cases for any (limited):
// 1. Gradual migration from JavaScript
// 2. Working with third-party libraries without types
// 3. Truly dynamic content where type is unknowable

// Implicit any (when noImplicitAny is disabled)
function process(input) {  // input is implicitly any
  return input.something;
}

// Better: Use unknown instead of any for unknown types
```

### unknown

```typescript
// unknown - the type-safe counterpart of any
let value: unknown = "hello";
value = 42;
value = { foo: "bar" };

// Unlike any, you cannot use unknown directly
// value.foo;           // Error - Object is of type 'unknown'
// value.toUpperCase(); // Error - Object is of type 'unknown'

// Must narrow the type before using
if (typeof value === "string") {
  console.log(value.toUpperCase());  // OK - value is string here
}

if (typeof value === "number") {
  console.log(value.toFixed(2));  // OK - value is number here
}

// unknown with type assertions (use with caution)
const str = value as string;  // Asserts value is string

// unknown in function parameters
function processUnknown(input: unknown): string {
  if (typeof input === "string") {
    return input;
  }
  if (typeof input === "number") {
    return input.toString();
  }
  return String(input);
}

// unknown is assignable only to itself and any
let a: unknown = value;  // OK
let b: any = value;      // OK
// let c: string = value;  // Error
```

### never

```typescript
// never - represents values that never occur
// Used for:
// 1. Functions that never return
// 2. Impossible type intersections
// 3. Exhaustiveness checking

// Functions that throw
function fail(message: string): never {
  throw new Error(message);
}

// Infinite loops
function infiniteLoop(): never {
  while (true) {
    // Do something forever
  }
}

// never is the bottom type - assignable to everything but nothing assigns to it
// let x: never = "hello";  // Error - string not assignable to never

// Exhaustiveness checking with never
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "triangle":
      return (shape.base * shape.height) / 2;
    default:
      // If all cases are handled, shape is never here
      const _exhaustiveCheck: never = shape;
      return _exhaustiveCheck;
  }
}

// If you add a new shape type, TypeScript will error on the default case
// This ensures all cases are handled

// never in conditional types
type NonNullable<T> = T extends null | undefined ? never : T;
type Result = NonNullable<string | null>;  // string
```

### void

```typescript
// void - absence of any type, typically for functions without return
function log(message: string): void {
  console.log(message);
  // No return statement, or return undefined/void
}

// void vs undefined
function returnVoid(): void {
  return;  // OK
  // return undefined;  // Also OK
}

function returnUndefined(): undefined {
  return undefined;  // Must explicitly return undefined
}

// void in callback types
type Callback = () => void;

// Important: void return type allows returning values (they're ignored)
const callback: Callback = () => {
  return 42;  // No error! Return value is ignored
};

// This is intentional to allow passing functions that return values
// to callbacks that don't use the return value
const arr = [1, 2, 3];
arr.forEach((n) => arr.push(n));  // push returns number, but forEach expects void
```

### object

```typescript
// object - any non-primitive type
let obj: object = { name: "John" };
obj = [1, 2, 3];      // Arrays are objects
obj = () => {};       // Functions are objects
// obj = "string";    // Error - primitives not allowed
// obj = 42;          // Error
// obj = null;        // Error (with strictNullChecks)

// object vs Object vs {}
// object: non-primitive types only
// Object: matches anything with toString, valueOf, etc. (avoid)
// {}: matches anything except null/undefined (avoid)

// Use more specific types when possible
interface Person {
  name: string;
  age: number;
}

let person: Person = { name: "John", age: 30 };  // Preferred over object

// Object.keys and object type
function getKeys(obj: object): string[] {
  return Object.keys(obj);
}
```

---

## 3. Literal Types

Literal types allow you to specify exact values a variable can hold.

### String Literal Types

```typescript
// Single string literal
let direction: "north" = "north";
// direction = "south";  // Error - only "north" is allowed

// Union of string literals
type Direction = "north" | "south" | "east" | "west";
let heading: Direction = "north";
heading = "east";  // OK
// heading = "up";  // Error - not in the union

// String literals in function parameters
function move(direction: "up" | "down" | "left" | "right"): void {
  console.log(`Moving ${direction}`);
}

move("up");    // OK
// move("diagonal");  // Error

// String literals for discriminated unions
type SuccessResult = { status: "success"; data: string };
type ErrorResult = { status: "error"; message: string };
type Result = SuccessResult | ErrorResult;

function handleResult(result: Result) {
  if (result.status === "success") {
    console.log(result.data);  // TypeScript knows this is SuccessResult
  } else {
    console.log(result.message);  // TypeScript knows this is ErrorResult
  }
}
```

### Numeric Literal Types

```typescript
// Single numeric literal
type Zero = 0;
let zero: Zero = 0;
// zero = 1;  // Error

// Union of numeric literals
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;
let roll: DiceRoll = 4;
// roll = 7;  // Error

// Numeric literals in APIs
type HttpSuccessCode = 200 | 201 | 204;
type HttpErrorCode = 400 | 401 | 403 | 404 | 500;
type HttpStatusCode = HttpSuccessCode | HttpErrorCode;

function handleStatus(code: HttpStatusCode) {
  if (code === 200 || code === 201 || code === 204) {
    console.log("Success!");
  }
}

// Numeric literals for array indices
type Bit = 0 | 1;
type Byte = [Bit, Bit, Bit, Bit, Bit, Bit, Bit, Bit];
```

### Boolean Literal Types

```typescript
// Boolean literals: true and false
type True = true;
type False = false;

let alwaysTrue: True = true;
// alwaysTrue = false;  // Error

// Useful in generic constraints
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false

// Boolean literals in discriminated unions
type LoadingState = { loading: true };
type LoadedState = { loading: false; data: string };
type State = LoadingState | LoadedState;

function render(state: State) {
  if (state.loading === true) {
    return "Loading...";
  }
  return state.data;  // TypeScript knows data exists when loading is false
}
```

### const Assertions

```typescript
// const assertions make literals immutable and narrow their types
const config = {
  endpoint: "/api",
  method: "GET",
} as const;

// Without as const:
// config.method is type string

// With as const:
// config.method is type "GET" (literal)
// config is readonly { readonly endpoint: "/api"; readonly method: "GET" }

// const assertions on arrays
const colors = ["red", "green", "blue"] as const;
// type: readonly ["red", "green", "blue"]
// colors[0] is type "red", not string

// colors.push("yellow");  // Error - readonly array

// Extracting types from const assertions
type Color = typeof colors[number];  // "red" | "green" | "blue"

// const assertions with object methods
const routes = {
  home: "/",
  about: "/about",
  users: "/users",
} as const;

type Route = typeof routes[keyof typeof routes];  // "/" | "/about" | "/users"

// Nested const assertions
const deepConfig = {
  api: {
    baseUrl: "https://api.example.com",
    version: "v1",
  },
  features: ["auth", "logging"],
} as const;
// All properties are deeply readonly
```

---

## 4. Arrays and Tuples

### Basic Arrays

```typescript
// Two syntaxes for arrays
let numbers: number[] = [1, 2, 3];
let strings: Array<string> = ["a", "b", "c"];

// Mixed types with unions
let mixed: (string | number)[] = [1, "two", 3, "four"];

// Empty arrays need type annotation
let empty: string[] = [];
empty.push("item");

// Array methods are fully typed
const doubled = numbers.map(n => n * 2);  // number[]
const found = numbers.find(n => n > 2);   // number | undefined
const filtered = numbers.filter(n => n > 1);  // number[]

// Array of objects
interface User {
  id: number;
  name: string;
}
let users: User[] = [
  { id: 1, name: "Alice" },
  { id: 2, name: "Bob" },
];
```

### Readonly Arrays

```typescript
// Readonly array - cannot be modified
const readonlyNumbers: readonly number[] = [1, 2, 3];
// readonlyNumbers.push(4);     // Error
// readonlyNumbers[0] = 10;     // Error
// readonlyNumbers.length = 0;  // Error

// ReadonlyArray generic type
const readonlyStrings: ReadonlyArray<string> = ["a", "b", "c"];

// Readonly arrays in function parameters (prevent mutation)
function processItems(items: readonly string[]): void {
  // items.push("new");  // Error - cannot modify
  items.forEach(item => console.log(item));  // OK - reading is fine
}

// Converting mutable to readonly
let mutable: string[] = ["a", "b"];
let immutable: readonly string[] = mutable;  // OK

// Cannot convert readonly to mutable without assertion
// let backToMutable: string[] = immutable;  // Error
let backToMutable: string[] = immutable as string[];  // Requires assertion
```

### Tuples

```typescript
// Tuples - fixed length arrays with typed positions
let tuple: [string, number] = ["age", 30];
let first: string = tuple[0];
let second: number = tuple[1];
// let third = tuple[2];  // Error - tuple only has 2 elements

// Labeled tuples (TypeScript 4.0+)
type Point2D = [x: number, y: number];
type Point3D = [x: number, y: number, z: number];

let point: Point2D = [10, 20];
// Labels appear in IDE tooltips and error messages

// Optional tuple elements
type OptionalTuple = [string, number?];
let withOptional: OptionalTuple = ["hello"];       // OK
let withBoth: OptionalTuple = ["hello", 42];       // OK

// Rest elements in tuples
type StringNumberBooleans = [string, number, ...boolean[]];
let snb: StringNumberBooleans = ["hello", 42, true, false, true];

// Rest elements at any position (TypeScript 4.2+)
type Strings = [boolean, ...string[], number];
let s: Strings = [true, "a", "b", "c", 42];

// Readonly tuples
type ReadonlyPair = readonly [string, number];
let pair: ReadonlyPair = ["key", 1];
// pair[0] = "newKey";  // Error
```

### Tuple Use Cases

```typescript
// Function that returns multiple values
function getCoordinates(): [number, number] {
  return [10, 20];
}
const [x, y] = getCoordinates();

// React useState pattern
type UseStateReturn<T> = [T, (newValue: T) => void];

// Named tuple for better documentation
type HttpResponse = [
  status: number,
  statusText: string,
  body: unknown
];

function parseResponse(response: HttpResponse) {
  const [status, statusText, body] = response;
  // Labels help with understanding
}

// Variadic tuple types (TypeScript 4.0+)
type Concat<T extends unknown[], U extends unknown[]> = [...T, ...U];
type Result = Concat<[1, 2], [3, 4]>;  // [1, 2, 3, 4]

// Spreading tuples in function parameters
function call<T extends unknown[], R>(
  fn: (...args: T) => R,
  ...args: T
): R {
  return fn(...args);
}
```

---

## 5. Object Types

### Interfaces

```typescript
// Basic interface
interface User {
  id: number;
  name: string;
  email: string;
}

// Optional properties
interface Config {
  host: string;
  port?: number;  // Optional
  secure?: boolean;
}

// Readonly properties
interface Point {
  readonly x: number;
  readonly y: number;
}

let p: Point = { x: 10, y: 20 };
// p.x = 5;  // Error - readonly

// Extending interfaces
interface Person {
  name: string;
  age: number;
}

interface Employee extends Person {
  employeeId: string;
  department: string;
}

// Multiple inheritance
interface Manager extends Employee {
  reports: Employee[];
}

// Implementing interfaces
class UserImpl implements User {
  constructor(
    public id: number,
    public name: string,
    public email: string
  ) {}
}

// Interface merging (declaration merging)
interface Window {
  customProperty: string;
}
// This merges with the global Window interface
```

### Type Aliases

```typescript
// Type alias for objects
type User = {
  id: number;
  name: string;
  email?: string;
};

// Type aliases can define any type, not just objects
type ID = string | number;
type Callback = (data: string) => void;
type Nullable<T> = T | null;

// Intersection types with type aliases
type Person = {
  name: string;
  age: number;
};

type Employee = Person & {
  employeeId: string;
};

// Recursive type aliases
type TreeNode<T> = {
  value: T;
  children?: TreeNode<T>[];
};

// Conditional type aliases
type NonNullable<T> = T extends null | undefined ? never : T;

// Type aliases vs interfaces
// - Type aliases cannot be merged (redeclared)
// - Type aliases can represent unions, tuples, primitives
// - Interfaces can only represent objects
// - Interfaces support extends, type aliases use intersections
// - Use interfaces for object shapes, type aliases for everything else
```

### Index Signatures

```typescript
// String index signature
interface StringDictionary {
  [key: string]: string;
}

let dict: StringDictionary = {};
dict["key1"] = "value1";
dict["key2"] = "value2";
// dict["key3"] = 42;  // Error - must be string

// Number index signature
interface NumberArray {
  [index: number]: string;
}

let arr: NumberArray = ["a", "b", "c"];
let first: string = arr[0];

// Combining index signatures with known properties
interface User {
  name: string;
  email: string;
  [key: string]: string;  // All properties must be string
}

// Index signature with union types
interface FlexibleDictionary {
  [key: string]: string | number | boolean;
  id: number;  // Must be compatible with index signature
  name: string;
}

// Readonly index signatures
interface ReadonlyDictionary {
  readonly [key: string]: string;
}

let roDict: ReadonlyDictionary = { key: "value" };
// roDict["key"] = "new";  // Error - readonly

// Template literal index signatures (TypeScript 4.4+)
interface DataAttributes {
  [key: `data-${string}`]: string;
}

let attrs: DataAttributes = {
  "data-id": "123",
  "data-name": "test",
  // "other": "value"  // Error - must start with "data-"
};
```

### Mapped Types

```typescript
// Basic mapped type
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};

// Make all properties optional
type Partial<T> = {
  [P in keyof T]?: T[P];
};

// Make all properties required
type Required<T> = {
  [P in keyof T]-?: T[P];  // -? removes optional
};

// Make all properties mutable
type Mutable<T> = {
  -readonly [P in keyof T]: T[P];  // -readonly removes readonly
};

// Pick specific properties
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Omit specific properties
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

// Record utility type
type Record<K extends keyof any, T> = {
  [P in K]: T;
};

type PageInfo = {
  title: string;
};
type Pages = Record<"home" | "about" | "contact", PageInfo>;

// Key remapping in mapped types (TypeScript 4.1+)
type Getters<T> = {
  [P in keyof T as `get${Capitalize<string & P>}`]: () => T[P];
};

interface Person {
  name: string;
  age: number;
}

type PersonGetters = Getters<Person>;
// { getName: () => string; getAge: () => number; }

// Filtering properties with key remapping
type OnlyStrings<T> = {
  [P in keyof T as T[P] extends string ? P : never]: T[P];
};
```

---

## 6. Union and Intersection Types

### Union Types

```typescript
// Basic union
type StringOrNumber = string | number;
let value: StringOrNumber = "hello";
value = 42;  // OK

// Union with literals
type Status = "pending" | "approved" | "rejected";
let status: Status = "pending";

// Union in function parameters
function format(input: string | number): string {
  if (typeof input === "string") {
    return input.toUpperCase();
  }
  return input.toFixed(2);
}

// Union with objects
type Cat = { kind: "cat"; meow: () => void };
type Dog = { kind: "dog"; bark: () => void };
type Pet = Cat | Dog;

function interact(pet: Pet) {
  if (pet.kind === "cat") {
    pet.meow();
  } else {
    pet.bark();
  }
}

// Distributive behavior in conditional types
type ToArray<T> = T extends any ? T[] : never;
type StrOrNumArray = ToArray<string | number>;  // string[] | number[]
```

### Intersection Types

```typescript
// Combine multiple types
type Person = {
  name: string;
  age: number;
};

type Employee = {
  employeeId: string;
  department: string;
};

type PersonEmployee = Person & Employee;
// Has all properties from both types

const emp: PersonEmployee = {
  name: "John",
  age: 30,
  employeeId: "E123",
  department: "Engineering",
};

// Intersection with interfaces
interface Printable {
  print(): void;
}

interface Loggable {
  log(): void;
}

type PrintableAndLoggable = Printable & Loggable;

// Intersection of incompatible types
type Impossible = string & number;  // never

// Intersection to add properties
function extend<T, U>(first: T, second: U): T & U {
  return { ...first, ...second };
}

const result = extend({ name: "John" }, { age: 30 });
// result is { name: string } & { age: number }
```

### Discriminated Unions

```typescript
// Discriminated union pattern
type Circle = {
  kind: "circle";
  radius: number;
};

type Square = {
  kind: "square";
  side: number;
};

type Triangle = {
  kind: "triangle";
  base: number;
  height: number;
};

type Shape = Circle | Square | Triangle;

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "triangle":
      return (shape.base * shape.height) / 2;
  }
}

// Discriminated unions with string literals
type Success<T> = { success: true; data: T };
type Failure = { success: false; error: string };
type Result<T> = Success<T> | Failure;

function handleResult<T>(result: Result<T>) {
  if (result.success) {
    console.log(result.data);  // TypeScript knows data exists
  } else {
    console.log(result.error);  // TypeScript knows error exists
  }
}

// State machine with discriminated unions
type IdleState = { status: "idle" };
type LoadingState = { status: "loading" };
type SuccessState = { status: "success"; data: string };
type ErrorState = { status: "error"; message: string };

type State = IdleState | LoadingState | SuccessState | ErrorState;

function transition(state: State): string {
  switch (state.status) {
    case "idle":
      return "Ready to start";
    case "loading":
      return "Please wait...";
    case "success":
      return `Data: ${state.data}`;
    case "error":
      return `Error: ${state.message}`;
  }
}
```

---

## 7. Type Narrowing

Type narrowing is the process of refining types to more specific types.

### typeof Guards

```typescript
// typeof works with primitives
function process(value: string | number | boolean) {
  if (typeof value === "string") {
    // value is string
    return value.toUpperCase();
  }
  if (typeof value === "number") {
    // value is number
    return value.toFixed(2);
  }
  // value is boolean
  return value ? "yes" : "no";
}

// typeof with null handling
function handleNullable(value: string | null) {
  if (typeof value === "string") {
    return value.length;
  }
  // value is null
  return 0;
}

// typeof limitations - objects and arrays
function checkType(value: object | null) {
  if (typeof value === "object") {
    // value is still object | null (typeof null === "object")
    if (value !== null) {
      // Now value is object
    }
  }
}
```

### instanceof Guards

```typescript
// instanceof for class instances
class Dog {
  bark() { console.log("Woof!"); }
}

class Cat {
  meow() { console.log("Meow!"); }
}

function makeSound(animal: Dog | Cat) {
  if (animal instanceof Dog) {
    animal.bark();  // animal is Dog
  } else {
    animal.meow();  // animal is Cat
  }
}

// instanceof with built-in types
function processValue(value: Date | string) {
  if (value instanceof Date) {
    return value.toISOString();
  }
  return value;
}

// instanceof with Error types
function handleError(error: Error | string) {
  if (error instanceof TypeError) {
    return `Type error: ${error.message}`;
  }
  if (error instanceof Error) {
    return `Error: ${error.message}`;
  }
  return error;
}
```

### in Operator

```typescript
// in operator checks for property existence
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    animal.swim();  // animal is Fish
  } else {
    animal.fly();  // animal is Bird
  }
}

// in with optional properties
interface Admin {
  name: string;
  privileges: string[];
}

interface User {
  name: string;
  email: string;
}

function printInfo(person: Admin | User) {
  console.log(person.name);  // Common property

  if ("privileges" in person) {
    console.log(person.privileges);  // person is Admin
  }

  if ("email" in person) {
    console.log(person.email);  // person is User
  }
}
```

### Type Predicates

```typescript
// Custom type predicate function
function isString(value: unknown): value is string {
  return typeof value === "string";
}

function process(value: unknown) {
  if (isString(value)) {
    // value is string
    console.log(value.toUpperCase());
  }
}

// Type predicate with complex types
interface Cat {
  meow(): void;
  purr(): void;
}

interface Dog {
  bark(): void;
  wagTail(): void;
}

function isCat(pet: Cat | Dog): pet is Cat {
  return "meow" in pet;
}

function interact(pet: Cat | Dog) {
  if (isCat(pet)) {
    pet.meow();
    pet.purr();
  } else {
    pet.bark();
    pet.wagTail();
  }
}

// Type predicate with arrays
function isStringArray(arr: unknown[]): arr is string[] {
  return arr.every(item => typeof item === "string");
}
```

### Assertion Functions

```typescript
// Assertion function that throws if condition is false
function assert(condition: boolean, message: string): asserts condition {
  if (!condition) {
    throw new Error(message);
  }
}

function process(value: string | null) {
  assert(value !== null, "Value must not be null");
  // After assertion, value is string
  console.log(value.toUpperCase());
}

// Assertion function with type predicate
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error("Expected string");
  }
}

function processUnknown(value: unknown) {
  assertIsString(value);
  // value is string after assertion
  console.log(value.toUpperCase());
}

// Assertion function for non-nullable
function assertDefined<T>(value: T | null | undefined): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error("Value must be defined");
  }
}
```

### Control Flow Analysis

```typescript
// TypeScript tracks types through control flow
function example(value: string | number | null) {
  if (value === null) {
    return;  // Early return narrows type
  }

  // value is string | number here

  if (typeof value === "string") {
    // value is string
    console.log(value.toUpperCase());
  } else {
    // value is number
    console.log(value.toFixed(2));
  }
}

// Narrowing with assignments
let value: string | number;
value = "hello";
// value is string here

value = 42;
// value is number here

// Truthiness narrowing
function printLength(value: string | null | undefined) {
  if (value) {
    // value is string (truthy check excludes null, undefined, empty string)
    console.log(value.length);
  }
}

// Equality narrowing
function compare(a: string | number, b: string | boolean) {
  if (a === b) {
    // Both must be string (only common type)
    console.log(a.toUpperCase());
    console.log(b.toUpperCase());
  }
}
```

---

## 8. Type Assertions

### Basic Assertions (as syntax)

```typescript
// Type assertion with as
let value: unknown = "hello";
let strLength: number = (value as string).length;

// Assertion on DOM elements
const input = document.getElementById("myInput") as HTMLInputElement;
input.value = "Hello";

// Double assertion (use sparingly)
// When TypeScript doesn't allow direct assertion
const num = "hello" as unknown as number;  // Generally avoid this

// Assertion in JSX
const element = <div>{value as string}</div>;
```

### const Assertions

```typescript
// const assertion creates literal types
let point = { x: 10, y: 20 } as const;
// Type: { readonly x: 10; readonly y: 20 }

// Array const assertion
let tuple = [1, 2, 3] as const;
// Type: readonly [1, 2, 3]

// Use case: Creating enum-like objects
const Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT",
} as const;

type Direction = typeof Direction[keyof typeof Direction];
// "UP" | "DOWN" | "LEFT" | "RIGHT"

// Function with const assertion
function getConfig() {
  return {
    endpoint: "/api",
    timeout: 5000,
  } as const;
}
// Return type: { readonly endpoint: "/api"; readonly timeout: 5000 }
```

### satisfies Operator (TypeScript 4.9+)

```typescript
// satisfies validates type without changing inferred type
type Colors = "red" | "green" | "blue";
type RGB = [number, number, number];

// Without satisfies - loses literal types
const palette1: Record<Colors, string | RGB> = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: [0, 0, 255],
};
// palette1.green is string | RGB

// With satisfies - keeps literal types
const palette2 = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: [0, 0, 255],
} satisfies Record<Colors, string | RGB>;
// palette2.green is string (more specific)
// palette2.red is [number, number, number]

// satisfies with validation
type Route = { path: string; element: JSX.Element };
const routes = {
  home: { path: "/", element: <Home /> },
  about: { path: "/about", element: <About /> },
  // missing: { path: "/missing" },  // Error: element is required
} satisfies Record<string, Route>;

// Keys are still inferred
type RouteKeys = keyof typeof routes;  // "home" | "about"

// Combining satisfies with as const
const config = {
  apiUrl: "https://api.example.com",
  timeout: 5000,
} as const satisfies { apiUrl: string; timeout: number };
```

### Non-null Assertion

```typescript
// Non-null assertion operator (!)
function processElement(id: string) {
  const element = document.getElementById(id);
  // element is HTMLElement | null

  element!.innerHTML = "Hello";  // Assert element is not null
  // Use only when you're certain the value is not null/undefined
}

// Non-null assertion on optional properties
interface User {
  name: string;
  address?: {
    city: string;
  };
}

function getCity(user: User): string {
  return user.address!.city;  // Assert address exists
  // Will throw at runtime if address is undefined
}

// Prefer null checks over non-null assertion
function safeGetCity(user: User): string | undefined {
  return user.address?.city;  // Optional chaining is safer
}
```

---

## 9. Type Guards

### Custom Type Guards

```typescript
// Basic type guard function
function isNumber(value: unknown): value is number {
  return typeof value === "number" && !isNaN(value);
}

// Type guard for objects
interface Car {
  drive(): void;
  honk(): void;
}

interface Bicycle {
  pedal(): void;
  ring(): void;
}

function isCar(vehicle: Car | Bicycle): vehicle is Car {
  return (vehicle as Car).drive !== undefined;
}

// Type guard for arrays
function isStringArray(value: unknown): value is string[] {
  return Array.isArray(value) && value.every(item => typeof item === "string");
}

// Type guard with validation
interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
}

function isSuccessResponse<T>(
  response: ApiResponse<T>
): response is Required<Pick<ApiResponse<T>, "success" | "data">> & { success: true } {
  return response.success === true && response.data !== undefined;
}

// Narrowing with type guards
function processResponse<T>(response: ApiResponse<T>) {
  if (isSuccessResponse(response)) {
    console.log(response.data);  // data is definitely present
  } else {
    console.log(response.error);
  }
}
```

### Assertion Functions

```typescript
// Assertion function - throws if condition fails
function assertNonNull<T>(value: T | null | undefined): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error("Value is null or undefined");
  }
}

// Usage
function process(value: string | null) {
  assertNonNull(value);
  // value is string after this line
  console.log(value.toUpperCase());
}

// Assertion function with condition
function assert(condition: boolean, msg?: string): asserts condition {
  if (!condition) {
    throw new Error(msg ?? "Assertion failed");
  }
}

function divide(a: number, b: number): number {
  assert(b !== 0, "Cannot divide by zero");
  return a / b;
}

// Assertion function for specific types
function assertIsHTMLElement(element: Element): asserts element is HTMLElement {
  if (!(element instanceof HTMLElement)) {
    throw new Error("Element is not an HTMLElement");
  }
}

// Class-based assertion
class ValidationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "ValidationError";
  }
}

function assertValidEmail(email: string): asserts email is `${string}@${string}` {
  if (!email.includes("@")) {
    throw new ValidationError("Invalid email format");
  }
}
```

### Discriminated Union Guards

```typescript
// Type guard based on discriminant property
type LoadingState = { state: "loading" };
type ErrorState = { state: "error"; message: string };
type SuccessState = { state: "success"; data: unknown };
type AppState = LoadingState | ErrorState | SuccessState;

function isErrorState(state: AppState): state is ErrorState {
  return state.state === "error";
}

function isSuccessState(state: AppState): state is SuccessState {
  return state.state === "success";
}

// Usage in components
function renderState(state: AppState) {
  if (isErrorState(state)) {
    return `Error: ${state.message}`;
  }
  if (isSuccessState(state)) {
    return `Data: ${JSON.stringify(state.data)}`;
  }
  return "Loading...";
}
```

---

## 10. Conditional Types

### Basic Conditional Types

```typescript
// Basic syntax: T extends U ? X : Y
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false

// Conditional type with infer
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type FuncReturn = ReturnType<() => string>;  // string
type FuncReturn2 = ReturnType<(x: number) => boolean>;  // boolean

// Extract parameter types
type Parameters<T> = T extends (...args: infer P) => any ? P : never;

type Params = Parameters<(a: string, b: number) => void>;  // [string, number]

// First parameter type
type FirstParameter<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;

type First = FirstParameter<(name: string, age: number) => void>;  // string
```

### Distributive Conditional Types

```typescript
// Conditional types distribute over unions
type ToArray<T> = T extends any ? T[] : never;

type StrOrNumArray = ToArray<string | number>;
// Distributes to: (string extends any ? string[] : never) | (number extends any ? number[] : never)
// Result: string[] | number[]

// Preventing distribution with tuple wrapper
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;

type StrOrNumArrayNonDist = ToArrayNonDist<string | number>;
// Result: (string | number)[]

// Filter types from union
type FilterString<T> = T extends string ? T : never;

type OnlyStrings = FilterString<string | number | boolean>;  // string

// Exclude and Extract utility types
type Exclude<T, U> = T extends U ? never : T;
type Extract<T, U> = T extends U ? T : never;

type T1 = Exclude<string | number | boolean, number>;  // string | boolean
type T2 = Extract<string | number | boolean, number | boolean>;  // number | boolean
```

### Inferring Within Conditional Types

```typescript
// Extract element type from array
type ElementType<T> = T extends (infer E)[] ? E : T;

type Elem = ElementType<string[]>;  // string
type NotArray = ElementType<number>;  // number

// Extract promise result type
type Awaited<T> = T extends Promise<infer R> ? Awaited<R> : T;

type ResolvedType = Awaited<Promise<string>>;  // string
type NestedResolved = Awaited<Promise<Promise<number>>>;  // number

// Infer function return type
type GetReturnType<T> = T extends (...args: never[]) => infer Return
  ? Return
  : never;

type Num = GetReturnType<() => number>;  // number

// Infer object property types
type PropertyType<T, K extends keyof T> = T extends { [P in K]: infer V } ? V : never;

interface User {
  id: number;
  name: string;
}

type IdType = PropertyType<User, "id">;  // number

// Multiple infer positions
type Func<T> = T extends (arg: infer A) => infer R ? [A, R] : never;

type FuncTypes = Func<(x: string) => number>;  // [string, number]
```

### Practical Conditional Type Examples

```typescript
// Type-safe event handlers
type EventHandler<T extends keyof HTMLElementEventMap> =
  (event: HTMLElementEventMap[T]) => void;

type ClickHandler = EventHandler<"click">;  // (event: MouseEvent) => void
type KeyHandler = EventHandler<"keydown">;  // (event: KeyboardEvent) => void

// Deep partial type
type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T;

interface Config {
  server: {
    host: string;
    port: number;
  };
  debug: boolean;
}

type PartialConfig = DeepPartial<Config>;
// { server?: { host?: string; port?: number }; debug?: boolean }

// Type-safe object paths
type PathImpl<T, K extends keyof T> =
  K extends string
    ? T[K] extends Record<string, any>
      ? K | `${K}.${PathImpl<T[K], keyof T[K]>}`
      : K
    : never;

type Path<T> = PathImpl<T, keyof T>;

type ConfigPaths = Path<Config>;  // "server" | "debug" | "server.host" | "server.port"
```

---

## 11. Template Literal Types

### Basic Template Literals

```typescript
// Template literal types (TypeScript 4.1+)
type Greeting = `Hello, ${string}!`;

let greeting: Greeting = "Hello, World!";  // OK
// greeting = "Hi, World!";  // Error - doesn't match pattern

// Union types in template literals
type Locale = "en" | "es" | "fr";
type MessageKey = "welcome" | "goodbye";

type LocalizedKey = `${Locale}_${MessageKey}`;
// "en_welcome" | "en_goodbye" | "es_welcome" | "es_goodbye" | "fr_welcome" | "fr_goodbye"

// Combining multiple unions
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type ApiVersion = "v1" | "v2";

type ApiEndpoint = `/${ApiVersion}/${string}`;
type ApiCall = `${HttpMethod} ${ApiEndpoint}`;
```

### String Manipulation Types

```typescript
// Built-in string manipulation types
type Upper = Uppercase<"hello">;  // "HELLO"
type Lower = Lowercase<"HELLO">;  // "hello"
type Capital = Capitalize<"hello">;  // "Hello"
type Uncapital = Uncapitalize<"Hello">;  // "hello"

// Using with template literals
type EventName = "click" | "focus" | "blur";
type HandlerName = `on${Capitalize<EventName>}`;  // "onClick" | "onFocus" | "onBlur"

// Getter/setter generation
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

interface Person {
  name: string;
  age: number;
}

type PersonGetters = Getters<Person>;
// { getName: () => string; getAge: () => number }

type PersonSetters = Setters<Person>;
// { setName: (value: string) => void; setAge: (value: number) => void }
```

### Inferring with Template Literals

```typescript
// Extract parts from template literal
type ExtractRouteParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<Rest>
    : T extends `${string}:${infer Param}`
      ? Param
      : never;

type Params = ExtractRouteParams<"/users/:userId/posts/:postId">;
// "userId" | "postId"

// Parse event names
type EventConfig<T extends string> =
  T extends `${infer Event}:${infer Handler}`
    ? { event: Event; handler: Handler }
    : never;

type Config = EventConfig<"click:handleClick">;
// { event: "click"; handler: "handleClick" }

// CSS unit parser
type CSSUnit = "px" | "em" | "rem" | "%";
type CSSValue = `${number}${CSSUnit}`;

function setWidth(width: CSSValue) {
  // ...
}

setWidth("100px");  // OK
setWidth("2em");    // OK
// setWidth("100");  // Error - missing unit
```

### Advanced Template Literal Patterns

```typescript
// Type-safe SQL-like query builder
type Column = "id" | "name" | "email" | "createdAt";
type Table = "users" | "posts" | "comments";

type SelectQuery = `SELECT ${Column | "*"} FROM ${Table}`;
type WhereClause = `WHERE ${Column} = ?`;

type FullQuery = `${SelectQuery} ${WhereClause}`;

// Valid queries
const query1: SelectQuery = "SELECT * FROM users";
const query2: FullQuery = "SELECT name FROM posts WHERE id = ?";

// Dot notation paths
type DotPrefix<T extends string> = T extends "" ? "" : `.${T}`;

type DotNestedKeys<T> = (
  T extends object
    ? {
        [K in Exclude<keyof T, symbol>]: `${K}${DotPrefix<DotNestedKeys<T[K]>>}`;
      }[Exclude<keyof T, symbol>]
    : ""
) extends infer D
  ? Extract<D, string>
  : never;

interface NestedObject {
  user: {
    name: string;
    address: {
      city: string;
      country: string;
    };
  };
}

type Keys = DotNestedKeys<NestedObject>;
// "user" | "user.name" | "user.address" | "user.address.city" | "user.address.country"
```

---

## 12. typeof and keyof Operators

### typeof Operator

```typescript
// typeof in type context
const config = {
  apiUrl: "https://api.example.com",
  timeout: 5000,
  retries: 3,
};

type ConfigType = typeof config;
// { apiUrl: string; timeout: number; retries: number }

// typeof with functions
function createUser(name: string, age: number) {
  return { id: Math.random(), name, age };
}

type CreateUserFn = typeof createUser;
// (name: string, age: number) => { id: number; name: string; age: number }

type User = ReturnType<typeof createUser>;
// { id: number; name: string; age: number }

// typeof with arrays
const colors = ["red", "green", "blue"] as const;
type Colors = typeof colors;           // readonly ["red", "green", "blue"]
type Color = typeof colors[number];    // "red" | "green" | "blue"

// typeof with class
class Point {
  x: number = 0;
  y: number = 0;

  static origin = new Point();

  constructor(x?: number, y?: number) {
    if (x) this.x = x;
    if (y) this.y = y;
  }
}

type PointInstance = Point;           // Instance type
type PointClass = typeof Point;       // Class constructor type

function createPoint(ctor: typeof Point, x: number, y: number): Point {
  return new ctor(x, y);
}
```

### keyof Operator

```typescript
// keyof extracts property keys as union
interface Person {
  name: string;
  age: number;
  email: string;
}

type PersonKeys = keyof Person;  // "name" | "age" | "email"

// keyof with index signatures
interface Dictionary {
  [key: string]: unknown;
}

type DictKeys = keyof Dictionary;  // string | number (number indexes to string)

// keyof with arrays
type ArrayKeys = keyof string[];
// number | "length" | "push" | "pop" | ... (all array properties)

// keyof in generic constraints
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const person: Person = { name: "John", age: 30, email: "john@example.com" };
const name = getProperty(person, "name");     // string
const age = getProperty(person, "age");       // number
// getProperty(person, "invalid");  // Error

// Combining keyof and typeof
const routes = {
  home: "/",
  about: "/about",
  users: "/users",
};

type RouteKey = keyof typeof routes;  // "home" | "about" | "users"
type RoutePath = typeof routes[RouteKey];  // "/" | "/about" | "/users"
```

### Practical Applications

```typescript
// Type-safe object picking
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach(key => {
    result[key] = obj[key];
  });
  return result;
}

const user = { id: 1, name: "John", email: "john@example.com", password: "secret" };
const publicUser = pick(user, ["id", "name", "email"]);
// { id: number; name: string; email: string }

// Type-safe object mapping
function mapObject<T extends object, U>(
  obj: T,
  fn: (value: T[keyof T], key: keyof T) => U
): { [K in keyof T]: U } {
  const result = {} as { [K in keyof T]: U };
  (Object.keys(obj) as (keyof T)[]).forEach(key => {
    result[key] = fn(obj[key], key);
  });
  return result;
}

// Creating type-safe event emitters
type Events = {
  click: { x: number; y: number };
  keypress: { key: string };
  scroll: { position: number };
};

class TypedEventEmitter<T extends Record<string, any>> {
  private handlers: { [K in keyof T]?: ((data: T[K]) => void)[] } = {};

  on<K extends keyof T>(event: K, handler: (data: T[K]) => void): void {
    if (!this.handlers[event]) {
      this.handlers[event] = [];
    }
    this.handlers[event]!.push(handler);
  }

  emit<K extends keyof T>(event: K, data: T[K]): void {
    this.handlers[event]?.forEach(handler => handler(data));
  }
}

const emitter = new TypedEventEmitter<Events>();
emitter.on("click", (data) => console.log(data.x, data.y));  // Fully typed
emitter.emit("click", { x: 10, y: 20 });  // Fully typed
```

---

## 13. Indexed Access Types

### Basic Indexed Access

```typescript
// Access property type by key
interface Person {
  name: string;
  age: number;
  address: {
    city: string;
    country: string;
  };
}

type PersonName = Person["name"];        // string
type PersonAge = Person["age"];          // number
type PersonAddress = Person["address"];  // { city: string; country: string }

// Nested access
type PersonCity = Person["address"]["city"];  // string

// Access multiple properties with union
type NameOrAge = Person["name" | "age"];  // string | number

// Access all property types
type PersonValues = Person[keyof Person];
// string | number | { city: string; country: string }
```

### Array and Tuple Indexed Access

```typescript
// Access array element type
type StringArrayElement = string[][number];  // string

// Access tuple element types
type Tuple = [string, number, boolean];
type First = Tuple[0];   // string
type Second = Tuple[1];  // number
type Third = Tuple[2];   // boolean

// Access union of tuple elements
type TupleElement = Tuple[number];  // string | number | boolean

// With const assertion
const colors = ["red", "green", "blue"] as const;
type Color = typeof colors[number];  // "red" | "green" | "blue"

// Extracting types from object arrays
const users = [
  { id: 1, name: "John" },
  { id: 2, name: "Jane" },
] as const;

type User = typeof users[number];
// { readonly id: 1; readonly name: "John" } | { readonly id: 2; readonly name: "Jane" }
```

### Advanced Indexed Access Patterns

```typescript
// Dynamic key access in generics
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// Deep property access type
type DeepPropertyType<T, Path extends string> =
  Path extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T
      ? DeepPropertyType<T[Key], Rest>
      : never
    : Path extends keyof T
      ? T[Path]
      : never;

interface Config {
  database: {
    host: string;
    port: number;
    credentials: {
      username: string;
      password: string;
    };
  };
}

type Host = DeepPropertyType<Config, "database.host">;  // string
type Username = DeepPropertyType<Config, "database.credentials.username">;  // string

// Indexed access with conditional types
type OptionalKeys<T> = {
  [K in keyof T]: {} extends Pick<T, K> ? K : never;
}[keyof T];

type RequiredKeys<T> = {
  [K in keyof T]: {} extends Pick<T, K> ? never : K;
}[keyof T];

interface Example {
  required: string;
  optional?: number;
}

type OptKeys = OptionalKeys<Example>;  // "optional"
type ReqKeys = RequiredKeys<Example>;  // "required"
```

---

## 14. Best Practices and Common Patterns

### Type-First Development

```typescript
// Define types before implementation
interface UserService {
  getUser(id: string): Promise<User>;
  createUser(data: CreateUserInput): Promise<User>;
  updateUser(id: string, data: UpdateUserInput): Promise<User>;
  deleteUser(id: string): Promise<void>;
}

interface User {
  id: string;
  email: string;
  name: string;
  createdAt: Date;
  updatedAt: Date;
}

type CreateUserInput = Pick<User, "email" | "name">;
type UpdateUserInput = Partial<CreateUserInput>;

// Implement against the interface
class UserServiceImpl implements UserService {
  async getUser(id: string): Promise<User> {
    // Implementation
  }
  // ...
}
```

### Discriminated Unions for State Management

```typescript
// Use discriminated unions for application state
type RequestState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

function handleState<T>(state: RequestState<T>): string {
  switch (state.status) {
    case "idle":
      return "Ready";
    case "loading":
      return "Loading...";
    case "success":
      return `Data: ${JSON.stringify(state.data)}`;
    case "error":
      return `Error: ${state.error.message}`;
  }
}

// Exhaustive checking with never
function assertNever(x: never): never {
  throw new Error("Unexpected value: " + x);
}
```

### Branded Types for Type Safety

```typescript
// Create nominal types using branding
type Brand<K, T> = K & { __brand: T };

type UserId = Brand<string, "UserId">;
type PostId = Brand<string, "PostId">;

function createUserId(id: string): UserId {
  return id as UserId;
}

function createPostId(id: string): PostId {
  return id as PostId;
}

function getUser(id: UserId): void {
  // ...
}

function getPost(id: PostId): void {
  // ...
}

const userId = createUserId("user-123");
const postId = createPostId("post-456");

getUser(userId);  // OK
// getUser(postId);  // Error - PostId is not assignable to UserId

// Branded primitives for validation
type Email = Brand<string, "Email">;
type PositiveNumber = Brand<number, "PositiveNumber">;

function validateEmail(email: string): Email {
  if (!email.includes("@")) {
    throw new Error("Invalid email");
  }
  return email as Email;
}

function validatePositive(n: number): PositiveNumber {
  if (n <= 0) {
    throw new Error("Number must be positive");
  }
  return n as PositiveNumber;
}
```

### Builder Pattern with Types

```typescript
// Type-safe builder pattern
interface QueryBuilder<T extends object = {}> {
  select<K extends string>(columns: K[]): QueryBuilder<T & { select: K[] }>;
  from(table: string): QueryBuilder<T & { from: string }>;
  where(condition: string): QueryBuilder<T & { where: string }>;
  build(): T extends { select: infer S; from: string } ? string : never;
}

// Result type inference
type BuilderResult<T> = T extends { select: infer S; from: string } ? string : never;
```

### Utility Type Combinations

```typescript
// Common utility type combinations
type Nullable<T> = T | null;
type Optional<T> = T | undefined;
type Maybe<T> = T | null | undefined;

// Deep versions
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

type DeepRequired<T> = {
  [P in keyof T]-?: T[P] extends object ? DeepRequired<T[P]> : T[P];
};

// Mutable - removes readonly
type Mutable<T> = {
  -readonly [P in keyof T]: T[P];
};

// Concrete - removes optional and readonly
type Concrete<T> = {
  -readonly [P in keyof T]-?: T[P];
};

// Merge two types (second overrides first)
type Merge<T, U> = Omit<T, keyof U> & U;

// Require specific keys
type RequireKeys<T, K extends keyof T> = T & Required<Pick<T, K>>;

// Make specific keys optional
type OptionalKeys<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;
```

### Function Overloading Best Practices

```typescript
// Use function overloads for different return types based on input
function parse(input: string): string;
function parse(input: number): number;
function parse(input: string | number): string | number {
  if (typeof input === "string") {
    return input.trim();
  }
  return Math.round(input);
}

// Prefer unions over overloads when return type is the same
function format(value: string | number | Date): string {
  if (typeof value === "string") return value;
  if (typeof value === "number") return value.toString();
  return value.toISOString();
}
```

---

## 15. Common Pitfalls

### Structural vs Nominal Typing

```typescript
// TypeScript uses structural typing - shape matters, not name
interface Point {
  x: number;
  y: number;
}

interface Coordinate {
  x: number;
  y: number;
}

function printPoint(p: Point) {
  console.log(p.x, p.y);
}

const coord: Coordinate = { x: 1, y: 2 };
printPoint(coord);  // OK - same structure

// This can lead to unexpected compatibility
interface User {
  id: number;
  name: string;
}

interface Product {
  id: number;
  name: string;
}

// User and Product are compatible! Use branded types for distinction
```

### Object Type Widening

```typescript
// Object literals are widened by default
const config = {
  method: "GET",  // Type is string, not "GET"
};

// Use const assertion for literal types
const configLiteral = {
  method: "GET" as const,  // Type is "GET"
};

// Or const assertion on the whole object
const configConst = {
  method: "GET",
} as const;
// Type is { readonly method: "GET" }
```

### Array Type Pitfalls

```typescript
// Empty arrays default to any[] without annotation
const items = [];  // any[] (if noImplicitAny is off)
const items2: string[] = [];  // Proper annotation

// Array methods can lose type information
const numbers = [1, 2, 3];
const filtered = numbers.filter(n => n > 1);  // number[]
const found = numbers.find(n => n > 1);  // number | undefined

// Type assertion in filter
interface Cat { type: "cat"; meow(): void }
interface Dog { type: "dog"; bark(): void }

const pets: (Cat | Dog)[] = [];
const cats = pets.filter((pet): pet is Cat => pet.type === "cat");
// Without type predicate, cats would be (Cat | Dog)[]
```

### Union Type Narrowing Gotchas

```typescript
// Type narrowing doesn't persist across callbacks
function processValue(value: string | number) {
  if (typeof value === "string") {
    // value is string here
    setTimeout(() => {
      // value is still string | number in callback
      // because value could have been reassigned
    }, 1000);
  }
}

// Solution: use const or a new variable
function processValueFixed(value: string | number) {
  if (typeof value === "string") {
    const str = value;  // str is always string
    setTimeout(() => {
      console.log(str.toUpperCase());  // OK
    }, 1000);
  }
}
```

### Excess Property Checking

```typescript
// TypeScript only checks excess properties on object literals
interface Config {
  host: string;
  port: number;
}

// Direct assignment - excess property error
// const config: Config = { host: "localhost", port: 3000, debug: true };  // Error

// Indirect assignment - no error (structural typing)
const obj = { host: "localhost", port: 3000, debug: true };
const config: Config = obj;  // OK - obj has required properties

// Use satisfies for validation without type widening
const configChecked = {
  host: "localhost",
  port: 3000,
  // debug: true,  // Would error with satisfies
} satisfies Config;
```

### this Type Issues

```typescript
// this can lose its type in callbacks
class Counter {
  count = 0;

  increment() {
    this.count++;
  }
}

const counter = new Counter();
const increment = counter.increment;
// increment();  // Runtime error - this is undefined

// Solutions:
// 1. Arrow function property
class CounterFixed1 {
  count = 0;
  increment = () => {
    this.count++;
  };
}

// 2. Bind in constructor
class CounterFixed2 {
  count = 0;

  constructor() {
    this.increment = this.increment.bind(this);
  }

  increment() {
    this.count++;
  }
}

// 3. Use this parameter
class CounterFixed3 {
  count = 0;

  increment(this: CounterFixed3) {
    this.count++;
  }
}
```

### Generic Type Inference Limits

```typescript
// TypeScript can struggle with complex generic inference
function createPair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

// Inference works well here
const pair = createPair("hello", 42);  // [string, number]

// But can fail with nested generics
function mapArray<T, U>(arr: T[], fn: (item: T) => U): U[] {
  return arr.map(fn);
}

// May need explicit type parameters
const result = mapArray<number, string>([1, 2, 3], n => n.toString());

// Use constraints to help inference
function firstElement<T extends unknown[]>(arr: T): T[0] | undefined {
  return arr[0];
}
```

### Type Assertion Overuse

```typescript
// Avoid excessive type assertions - they bypass type safety
const value: unknown = getData();

// Bad - assertions hide potential errors
const user = value as User;
user.name;  // No type error, but might crash

// Better - use type guards
function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "name" in value &&
    "email" in value
  );
}

if (isUser(value)) {
  value.name;  // Safe
}

// Non-null assertions should be rare
const element = document.getElementById("app")!;  // Risky

// Prefer optional chaining or null checks
const elementSafe = document.getElementById("app");
if (elementSafe) {
  elementSafe.innerHTML = "Hello";  // Safe
}
```

### enum Pitfalls

```typescript
// Numeric enums allow invalid values
enum Status {
  Active = 1,
  Inactive = 2,
}

function setStatus(status: Status) {
  console.log(status);
}

setStatus(Status.Active);  // OK
setStatus(999);  // No error! TypeScript allows any number

// Use const assertions or union types instead
const StatusConst = {
  Active: "active",
  Inactive: "inactive",
} as const;

type StatusType = typeof StatusConst[keyof typeof StatusConst];
// "active" | "inactive"

function setStatusSafe(status: StatusType) {
  console.log(status);
}

setStatusSafe("active");  // OK
// setStatusSafe("invalid");  // Error
```

---

## Summary

TypeScript's type system is powerful and expressive. Key takeaways:

1. **Use strict mode** - Enable `strict: true` in tsconfig.json for maximum type safety
2. **Prefer type inference** - Let TypeScript infer types when obvious
3. **Use unknown over any** - unknown is the type-safe alternative to any
4. **Leverage discriminated unions** - Great for state management and domain modeling
5. **Use type guards** - Custom type predicates for complex narrowing
6. **Combine utility types** - Build complex types from simpler ones
7. **Avoid assertions** - Use type guards instead of type assertions when possible
8. **Consider branded types** - For nominal typing needs
9. **Use const assertions** - For literal types and immutability
10. **Leverage satisfies** - For validation without type widening (TypeScript 4.9+)

For more information, refer to the official TypeScript documentation at https://www.typescriptlang.org/docs/
