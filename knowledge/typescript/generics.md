# TypeScript Generics

Generics are one of the most powerful features of TypeScript, enabling the creation of reusable, type-safe components that work with a variety of types rather than a single one. This comprehensive guide covers all aspects of generics in TypeScript.

---

## Table of Contents

1. [Generic Functions](#1-generic-functions)
2. [Generic Constraints](#2-generic-constraints)
3. [Generic Interfaces](#3-generic-interfaces)
4. [Generic Classes](#4-generic-classes)
5. [Generic Type Aliases](#5-generic-type-aliases)
6. [Default Type Parameters](#6-default-type-parameters)
7. [Using Type Parameters in Constraints](#7-using-type-parameters-in-constraints)
8. [Conditional Types with Generics](#8-conditional-types-with-generics)
9. [Mapped Types with Generics](#9-mapped-types-with-generics)
10. [The infer Keyword](#10-the-infer-keyword)
11. [Variadic Tuple Types](#11-variadic-tuple-types)
12. [Higher-Order Generic Patterns](#12-higher-order-generic-patterns)
13. [Generic Utility Functions](#13-generic-utility-functions)
14. [Best Practices](#14-best-practices)
15. [Common Pitfalls](#15-common-pitfalls)

---

## 1. Generic Functions

Generic functions allow you to write functions that work with any type while maintaining type safety.

### Basic Syntax

```typescript
// The identity function - the "Hello World" of generics
function identity<T>(value: T): T {
  return value;
}

// Explicit type argument
const num = identity<number>(42);          // Type: number
const str = identity<string>("hello");     // Type: string

// Type inference - TypeScript infers the type from the argument
const inferredNum = identity(42);          // Type: 42 (literal type)
const inferredStr = identity("hello");     // Type: "hello" (literal type)
```

### Multiple Type Parameters

```typescript
// Function with two type parameters
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

const result = pair("hello", 42);  // Type: [string, number]

// Map function with transformation
function map<T, U>(array: T[], fn: (item: T, index: number) => U): U[] {
  return array.map(fn);
}

const lengths = map(["a", "ab", "abc"], s => s.length);  // Type: number[]
const doubled = map([1, 2, 3], n => n * 2);              // Type: number[]

// Swap function
function swap<T, U>(tuple: [T, U]): [U, T] {
  return [tuple[1], tuple[0]];
}

const swapped = swap(["hello", 42]);  // Type: [number, string]
```

### Generic Arrow Functions

```typescript
// Arrow function syntax
const identity2 = <T>(value: T): T => value;

// In JSX files (.tsx), use this syntax to avoid confusion with JSX
const identity3 = <T,>(value: T): T => value;
// Or use extends
const identity4 = <T extends unknown>(value: T): T => value;

// Multiple parameters in arrow functions
const makePair = <T, U>(first: T, second: U): [T, U] => [first, second];
```

### Generic Function Types

```typescript
// Function type with generic
type GenericFunction<T> = (arg: T) => T;

const numIdentity: GenericFunction<number> = (x) => x;

// Call signature in an object type
type GenericFn = {
  <T>(arg: T): T;
};

// Generic call signature
interface GenericCallable {
  <T>(arg: T): T;
}

const callable: GenericCallable = identity;
```

### Rest Parameters with Generics

```typescript
// Typed rest parameters
function toArray<T>(...args: T[]): T[] {
  return args;
}

const nums = toArray(1, 2, 3);        // Type: number[]
const strs = toArray("a", "b", "c");  // Type: string[]

// Multiple rest types with tuple types
function merge<T extends unknown[], U extends unknown[]>(
  arr1: [...T],
  arr2: [...U]
): [...T, ...U] {
  return [...arr1, ...arr2];
}

const merged = merge([1, 2], ["a", "b"]);  // Type: [number, number, string, string]
```

---

## 2. Generic Constraints

Constraints allow you to limit the types that can be used with a generic, ensuring certain properties or methods are available.

### The extends Keyword

```typescript
// Constraint requiring a length property
function getLength<T extends { length: number }>(item: T): number {
  return item.length;
}

getLength("hello");           // OK - string has length
getLength([1, 2, 3]);         // OK - array has length
getLength({ length: 10 });    // OK - object with length property
// getLength(123);            // Error: number doesn't have length

// Constraint with interface
interface Lengthwise {
  length: number;
}

function logLength<T extends Lengthwise>(item: T): void {
  console.log(`Length: ${item.length}`);
}
```

### Constraining to Specific Types

```typescript
// Constraint to primitive types
function processValue<T extends string | number>(value: T): T {
  console.log(value);
  return value;
}

// Constraint to object types
function clone<T extends object>(obj: T): T {
  return { ...obj };
}

// Constraint to arrays
function first<T extends unknown[]>(arr: T): T[0] | undefined {
  return arr[0];
}

// Constraint to functions
function call<T extends (...args: unknown[]) => unknown>(
  fn: T,
  ...args: Parameters<T>
): ReturnType<T> {
  return fn(...args) as ReturnType<T>;
}
```

### keyof Constraint

```typescript
// Access object properties safely
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const person = { name: "Alice", age: 30 };
const name = getProperty(person, "name");  // Type: string
const age = getProperty(person, "age");    // Type: number
// getProperty(person, "height");          // Error: "height" not in keyof person

// Set property with constraint
function setProperty<T, K extends keyof T>(
  obj: T,
  key: K,
  value: T[K]
): void {
  obj[key] = value;
}

setProperty(person, "age", 31);  // OK
// setProperty(person, "age", "31");  // Error: string not assignable to number
```

### Multiple Constraints

```typescript
// Using intersection types for multiple constraints
interface Named {
  name: string;
}

interface Aged {
  age: number;
}

function greet<T extends Named & Aged>(entity: T): string {
  return `Hello, ${entity.name}! You are ${entity.age} years old.`;
}

// Constraint with generic constraint
function copyFields<T extends U, U>(target: T, source: U): T {
  for (const key in source) {
    target[key] = source[key] as T[Extract<keyof U, string>];
  }
  return target;
}
```

### Constructor Constraints

```typescript
// Constraint for constructable types
interface Constructor<T = object> {
  new (...args: unknown[]): T;
}

function createInstance<T>(ctor: Constructor<T>): T {
  return new ctor();
}

class Animal {
  name = "Animal";
}

const animal = createInstance(Animal);  // Type: Animal

// With constructor parameters
type ConstructorWithArgs<T, Args extends unknown[]> = new (...args: Args) => T;

function createWithArgs<T, Args extends unknown[]>(
  ctor: ConstructorWithArgs<T, Args>,
  ...args: Args
): T {
  return new ctor(...args);
}

class Person {
  constructor(public name: string, public age: number) {}
}

const person2 = createWithArgs(Person, "Bob", 25);  // Type: Person
```

---

## 3. Generic Interfaces

Generic interfaces allow you to define contracts that work with various types.

### Basic Generic Interfaces

```typescript
// Simple generic interface
interface Box<T> {
  value: T;
}

const stringBox: Box<string> = { value: "hello" };
const numberBox: Box<number> = { value: 42 };

// Generic interface with methods
interface Container<T> {
  value: T;
  getValue(): T;
  setValue(value: T): void;
}

const container: Container<string> = {
  value: "initial",
  getValue() {
    return this.value;
  },
  setValue(value) {
    this.value = value;
  },
};
```

### Generic Interface with Multiple Parameters

```typescript
interface KeyValuePair<K, V> {
  key: K;
  value: V;
}

const pair1: KeyValuePair<string, number> = { key: "age", value: 30 };
const pair2: KeyValuePair<number, string> = { key: 1, value: "first" };

// Dictionary-like interface
interface Dictionary<K extends string | number | symbol, V> {
  [key: string]: V;
  get(key: K): V | undefined;
  set(key: K, value: V): void;
  has(key: K): boolean;
}
```

### Generic Interface Extending Other Interfaces

```typescript
interface Entity {
  id: string;
  createdAt: Date;
}

interface Repository<T extends Entity> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(entity: Omit<T, "id" | "createdAt">): Promise<T>;
  update(id: string, entity: Partial<T>): Promise<T>;
  delete(id: string): Promise<boolean>;
}

// Implementation
interface User extends Entity {
  name: string;
  email: string;
}

class UserRepository implements Repository<User> {
  async findById(id: string): Promise<User | null> {
    // Implementation
    return null;
  }
  async findAll(): Promise<User[]> {
    return [];
  }
  async create(entity: Omit<User, "id" | "createdAt">): Promise<User> {
    return { ...entity, id: "1", createdAt: new Date() };
  }
  async update(id: string, entity: Partial<User>): Promise<User> {
    return { id, createdAt: new Date(), name: "", email: "", ...entity };
  }
  async delete(id: string): Promise<boolean> {
    return true;
  }
}
```

### Generic Function Interfaces

```typescript
// Interface describing a generic function
interface GenericIdentityFn {
  <T>(arg: T): T;
}

// Interface with type parameter for the function
interface GenericIdentityFn2<T> {
  (arg: T): T;
}

const myIdentity: GenericIdentityFn = (arg) => arg;
const myNumberIdentity: GenericIdentityFn2<number> = (arg) => arg;

// Callable interface with generics
interface Comparer<T> {
  (a: T, b: T): number;
}

const numberComparer: Comparer<number> = (a, b) => a - b;
const stringComparer: Comparer<string> = (a, b) => a.localeCompare(b);
```

---

## 4. Generic Classes

Generic classes allow you to create reusable class templates that work with different types.

### Basic Generic Class

```typescript
class Box<T> {
  private content: T;

  constructor(value: T) {
    this.content = value;
  }

  getValue(): T {
    return this.content;
  }

  setValue(value: T): void {
    this.content = value;
  }
}

const stringBox = new Box("hello");   // Box<string>
const numberBox = new Box(42);        // Box<number>
```

### Generic Class with Multiple Type Parameters

```typescript
class Pair<T, U> {
  constructor(
    public first: T,
    public second: U
  ) {}

  swap(): Pair<U, T> {
    return new Pair(this.second, this.first);
  }

  map<V, W>(
    mapFirst: (value: T) => V,
    mapSecond: (value: U) => W
  ): Pair<V, W> {
    return new Pair(mapFirst(this.first), mapSecond(this.second));
  }
}

const pair = new Pair("hello", 42);
const swapped = pair.swap();  // Pair<number, string>
```

### Data Structure Examples

```typescript
// Generic Stack
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }

  size(): number {
    return this.items.length;
  }
}

// Generic Queue
class Queue<T> {
  private items: T[] = [];

  enqueue(item: T): void {
    this.items.push(item);
  }

  dequeue(): T | undefined {
    return this.items.shift();
  }

  front(): T | undefined {
    return this.items[0];
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }
}

// Generic Linked List Node
class LinkedListNode<T> {
  constructor(
    public value: T,
    public next: LinkedListNode<T> | null = null
  ) {}
}

class LinkedList<T> {
  private head: LinkedListNode<T> | null = null;
  private tail: LinkedListNode<T> | null = null;

  append(value: T): void {
    const node = new LinkedListNode(value);
    if (!this.head) {
      this.head = node;
      this.tail = node;
    } else {
      this.tail!.next = node;
      this.tail = node;
    }
  }

  *[Symbol.iterator](): Iterator<T> {
    let current = this.head;
    while (current) {
      yield current.value;
      current = current.next;
    }
  }
}
```

### Generic Class with Constraints

```typescript
interface Comparable<T> {
  compareTo(other: T): number;
}

class SortedList<T extends Comparable<T>> {
  private items: T[] = [];

  add(item: T): void {
    const index = this.items.findIndex((i) => item.compareTo(i) < 0);
    if (index === -1) {
      this.items.push(item);
    } else {
      this.items.splice(index, 0, item);
    }
  }

  getAll(): T[] {
    return [...this.items];
  }
}

class NumberWrapper implements Comparable<NumberWrapper> {
  constructor(public value: number) {}

  compareTo(other: NumberWrapper): number {
    return this.value - other.value;
  }
}

const sortedNumbers = new SortedList<NumberWrapper>();
sortedNumbers.add(new NumberWrapper(3));
sortedNumbers.add(new NumberWrapper(1));
sortedNumbers.add(new NumberWrapper(2));
```

### Static Members and Generics

```typescript
// Static members cannot use the class's type parameter
class Factory<T> {
  // This is NOT allowed:
  // static instance: T;

  // But static generic methods are allowed:
  static create<U>(value: U): Factory<U> {
    const factory = new Factory<U>();
    factory.value = value;
    return factory;
  }

  value!: T;
}

const stringFactory = Factory.create("hello");  // Factory<string>
```

---

## 5. Generic Type Aliases

Type aliases with generics provide flexible ways to create reusable type definitions.

### Basic Generic Type Aliases

```typescript
// Simple type alias
type Nullable<T> = T | null;
type Optional<T> = T | undefined;
type Maybe<T> = T | null | undefined;

const nullableString: Nullable<string> = null;
const optionalNumber: Optional<number> = undefined;

// Object type alias
type Result<T, E = Error> = {
  success: boolean;
  data?: T;
  error?: E;
};

const successResult: Result<string> = { success: true, data: "hello" };
const errorResult: Result<string> = { success: false, error: new Error("Failed") };
```

### Union and Intersection Type Aliases

```typescript
// Union types
type StringOrNumber<T> = T extends string ? string : number;

// Either type (discriminated union)
type Either<L, R> =
  | { type: "left"; value: L }
  | { type: "right"; value: R };

function left<L>(value: L): Either<L, never> {
  return { type: "left", value };
}

function right<R>(value: R): Either<never, R> {
  return { type: "right", value };
}

// Tree structure
type TreeNode<T> = {
  value: T;
  children: TreeNode<T>[];
};

const tree: TreeNode<number> = {
  value: 1,
  children: [
    { value: 2, children: [] },
    { value: 3, children: [{ value: 4, children: [] }] },
  ],
};
```

### Recursive Type Aliases

```typescript
// JSON type
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

// Deep partial
type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T;

interface Config {
  server: {
    host: string;
    port: number;
  };
  database: {
    url: string;
    pool: {
      min: number;
      max: number;
    };
  };
}

const partialConfig: DeepPartial<Config> = {
  server: { port: 3000 },
};

// Deep required
type DeepRequired<T> = T extends object
  ? { [P in keyof T]-?: DeepRequired<T[P]> }
  : T;

// Deep readonly
type DeepReadonly<T> = T extends object
  ? { readonly [P in keyof T]: DeepReadonly<T[P]> }
  : T;
```

### Function Type Aliases

```typescript
// Function type with generics
type Mapper<T, U> = (item: T) => U;
type Predicate<T> = (item: T) => boolean;
type Reducer<T, U> = (accumulator: U, current: T) => U;
type Comparator<T> = (a: T, b: T) => number;

// Async function types
type AsyncMapper<T, U> = (item: T) => Promise<U>;
type AsyncPredicate<T> = (item: T) => Promise<boolean>;

// Event handler types
type EventHandler<T> = (event: T) => void;
type AsyncEventHandler<T> = (event: T) => Promise<void>;

// Higher-order function types
type Curried<T, U, R> = (arg1: T) => (arg2: U) => R;
```

---

## 6. Default Type Parameters

Default type parameters allow you to specify fallback types when none are provided.

### Basic Default Parameters

```typescript
// Interface with default type
interface Response<T = unknown> {
  data: T;
  status: number;
  message: string;
}

// Using the default
const genericResponse: Response = {
  data: { any: "value" },
  status: 200,
  message: "OK",
};

// Overriding the default
interface User {
  id: number;
  name: string;
}

const userResponse: Response<User> = {
  data: { id: 1, name: "Alice" },
  status: 200,
  message: "OK",
};
```

### Multiple Default Parameters

```typescript
// Multiple defaults - must follow non-default parameters
interface ApiResponse<T = unknown, E = Error> {
  data?: T;
  error?: E;
  status: number;
}

// Using all defaults
const response1: ApiResponse = { status: 200, data: "hello" };

// Specifying first parameter only
const response2: ApiResponse<User> = { status: 200, data: { id: 1, name: "Bob" } };

// Specifying both parameters
class CustomError extends Error {
  code: string;
  constructor(message: string, code: string) {
    super(message);
    this.code = code;
  }
}

const response3: ApiResponse<User, CustomError> = {
  status: 500,
  error: new CustomError("Failed", "ERR_001"),
};
```

### Default Parameters in Functions

```typescript
// Function with default type parameter
function createArray<T = string>(length: number, value: T): T[] {
  return Array(length).fill(value);
}

const strings = createArray(3, "a");    // string[]
const numbers = createArray(3, 42);     // number[]
const defaulted = createArray<string>(3, "x");  // string[]

// Class with default type parameter
class Container<T = unknown> {
  constructor(public value: T) {}

  map<U = T>(fn: (value: T) => U): Container<U> {
    return new Container(fn(this.value));
  }
}

const container = new Container("hello");  // Container<string>
const unknownContainer = new Container<unknown>(42);  // Container<unknown>
```

### Defaults with Constraints

```typescript
// Default must satisfy the constraint
interface Entity {
  id: string;
}

interface DefaultEntity extends Entity {
  id: string;
  createdAt: Date;
}

// Default type must extend the constraint
class Repository<T extends Entity = DefaultEntity> {
  items: T[] = [];

  add(item: T): void {
    this.items.push(item);
  }

  findById(id: string): T | undefined {
    return this.items.find((item) => item.id === id);
  }
}

// Using default
const defaultRepo = new Repository();  // Repository<DefaultEntity>

// Custom entity
interface Product extends Entity {
  id: string;
  name: string;
  price: number;
}

const productRepo = new Repository<Product>();  // Repository<Product>
```

---

## 7. Using Type Parameters in Constraints

Type parameters can reference each other to create sophisticated type relationships.

### Self-Referencing Constraints

```typescript
// One type parameter constraining another
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// Picking multiple keys
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach((key) => {
    result[key] = obj[key];
  });
  return result;
}

const user = { id: 1, name: "Alice", email: "alice@example.com", age: 30 };
const subset = pick(user, ["name", "email"]);  // { name: string; email: string }
```

### Chained Constraints

```typescript
// T constrains U, U constrains V
function merge<T extends object, U extends T, V extends U>(
  base: T,
  override: U,
  extra: V
): V {
  return { ...base, ...override, ...extra };
}

// Builder pattern with chained generics
class Builder<T extends object> {
  private obj: T;

  constructor(initial: T) {
    this.obj = initial;
  }

  set<K extends keyof T>(key: K, value: T[K]): this {
    this.obj[key] = value;
    return this;
  }

  extend<U extends object>(extra: U): Builder<T & U> {
    return new Builder({ ...this.obj, ...extra });
  }

  build(): T {
    return { ...this.obj };
  }
}

const result = new Builder({ name: "" })
  .set("name", "Alice")
  .extend({ age: 0 })
  .set("age", 30)
  .extend({ email: "" })
  .set("email", "alice@example.com")
  .build();
// Type: { name: string } & { age: number } & { email: string }
```

### Recursive Constraints

```typescript
// Comparable pattern with self-reference
interface Comparable<T extends Comparable<T>> {
  compareTo(other: T): number;
}

class Version implements Comparable<Version> {
  constructor(
    public major: number,
    public minor: number,
    public patch: number
  ) {}

  compareTo(other: Version): number {
    if (this.major !== other.major) return this.major - other.major;
    if (this.minor !== other.minor) return this.minor - other.minor;
    return this.patch - other.patch;
  }
}

// Fluent interface pattern
interface FluentBuilder<T extends FluentBuilder<T>> {
  reset(): T;
  validate(): T;
}

class QueryBuilder implements FluentBuilder<QueryBuilder> {
  private query = "";

  select(fields: string[]): this {
    this.query += `SELECT ${fields.join(", ")} `;
    return this;
  }

  from(table: string): this {
    this.query += `FROM ${table} `;
    return this;
  }

  reset(): this {
    this.query = "";
    return this;
  }

  validate(): this {
    if (!this.query) throw new Error("Empty query");
    return this;
  }

  build(): string {
    return this.query.trim();
  }
}
```

---

## 8. Conditional Types with Generics

Conditional types allow you to create types that depend on type relationships.

### Basic Conditional Types

```typescript
// Simple conditional type
type IsString<T> = T extends string ? true : false;

type A = IsString<"hello">;  // true
type B = IsString<42>;       // false
type C = IsString<string>;   // true

// Conditional with union distribution
type ToArray<T> = T extends unknown ? T[] : never;

type D = ToArray<string | number>;  // string[] | number[]

// Prevent distribution with tuple
type ToArrayNonDist<T> = [T] extends [unknown] ? T[] : never;

type E = ToArrayNonDist<string | number>;  // (string | number)[]
```

### Extracting and Excluding Types

```typescript
// Built-in Extract and Exclude
type T1 = Extract<"a" | "b" | "c", "a" | "f">;  // "a"
type T2 = Exclude<"a" | "b" | "c", "a">;         // "b" | "c"

// NonNullable implementation
type NonNullable<T> = T extends null | undefined ? never : T;

type T3 = NonNullable<string | null | undefined>;  // string

// Custom type filters
type FilterByType<T, U> = T extends U ? T : never;

type Numbers = FilterByType<string | number | boolean, number>;  // number

// Function type conditionals
type FunctionPropertyNames<T> = {
  [K in keyof T]: T[K] extends Function ? K : never;
}[keyof T];

interface Mixed {
  name: string;
  age: number;
  greet(): void;
  calculate(x: number): number;
}

type FuncNames = FunctionPropertyNames<Mixed>;  // "greet" | "calculate"
```

### Nested Conditional Types

```typescript
// Type classification
type TypeName<T> = T extends string
  ? "string"
  : T extends number
  ? "number"
  : T extends boolean
  ? "boolean"
  : T extends undefined
  ? "undefined"
  : T extends Function
  ? "function"
  : "object";

type T4 = TypeName<string>;     // "string"
type T5 = TypeName<() => void>; // "function"
type T6 = TypeName<string[]>;   // "object"

// Unwrap nested types
type Unwrap<T> = T extends Promise<infer U>
  ? Unwrap<U>
  : T extends Array<infer U>
  ? Unwrap<U>
  : T;

type T7 = Unwrap<Promise<Promise<string>>>;  // string
type T8 = Unwrap<string[][]>;                 // string
```

### Distributive Conditional Types

```typescript
// Conditional types distribute over unions
type Nullable<T> = T | null;
type NonNullableKeys<T> = {
  [K in keyof T]: null extends T[K] ? never : K;
}[keyof T];

interface User {
  id: number;
  name: string;
  email: string | null;
  phone: string | null;
}

type RequiredUserKeys = NonNullableKeys<User>;  // "id" | "name"

// Flatten union of arrays
type Flatten<T> = T extends (infer U)[] ? U : T;

type T9 = Flatten<string[] | number[]>;  // string | number

// Map over union members
type MapToPromise<T> = T extends unknown ? Promise<T> : never;

type T10 = MapToPromise<string | number>;  // Promise<string> | Promise<number>
```

---

## 9. Mapped Types with Generics

Mapped types transform existing types by iterating over their properties.

### Basic Mapped Types

```typescript
// Make all properties optional
type Partial<T> = {
  [K in keyof T]?: T[K];
};

// Make all properties required
type Required<T> = {
  [K in keyof T]-?: T[K];
};

// Make all properties readonly
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Make all properties mutable (remove readonly)
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

interface User {
  readonly id: number;
  name: string;
  email?: string;
}

type PartialUser = Partial<User>;      // all optional
type RequiredUser = Required<User>;    // all required
type ReadonlyUser = Readonly<User>;    // all readonly
type MutableUser = Mutable<User>;      // no readonly
```

### Key Remapping

```typescript
// Rename keys using template literals
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

// Filter keys
type FilteredKeys<T, U> = {
  [K in keyof T as T[K] extends U ? K : never]: T[K];
};

interface Mixed {
  id: number;
  name: string;
  active: boolean;
  count: number;
}

type NumberProps = FilteredKeys<Mixed, number>;  // { id: number; count: number }
```

### Property Modifiers in Mapped Types

```typescript
// Add or remove modifiers
type AddOptional<T> = {
  [K in keyof T]+?: T[K];  // + is default, can be omitted
};

type RemoveOptional<T> = {
  [K in keyof T]-?: T[K];
};

type AddReadonly<T> = {
  +readonly [K in keyof T]: T[K];  // + is default
};

type RemoveReadonly<T> = {
  -readonly [K in keyof T]: T[K];
};

// Combine modifiers
type Strict<T> = {
  -readonly [K in keyof T]-?: T[K];  // Remove both optional and readonly
};
```

### Advanced Mapped Type Patterns

```typescript
// Pick implementation
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Omit implementation
type Omit<T, K extends keyof T> = {
  [P in Exclude<keyof T, K>]: T[P];
};

// Record implementation
type Record<K extends keyof any, T> = {
  [P in K]: T;
};

// Deep transformations
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : DeepReadonly<T[K]>
    : T[K];
};

type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : DeepPartial<T[K]>
    : T[K];
};

// Nullable mapped type
type Nullify<T> = {
  [K in keyof T]: T[K] | null;
};

// Promise-wrapped properties
type Promisify<T> = {
  [K in keyof T]: Promise<T[K]>;
};
```

---

## 10. The infer Keyword

The `infer` keyword allows you to extract and capture types within conditional type expressions.

### Basic Inference

```typescript
// Infer return type
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type R1 = ReturnType<() => string>;           // string
type R2 = ReturnType<(x: number) => boolean>; // boolean

// Infer parameter types
type Parameters<T> = T extends (...args: infer P) => any ? P : never;

type P1 = Parameters<(a: string, b: number) => void>;  // [string, number]

// Infer first parameter
type FirstParameter<T> = T extends (first: infer F, ...rest: any[]) => any
  ? F
  : never;

type F1 = FirstParameter<(a: string, b: number) => void>;  // string
```

### Array and Tuple Inference

```typescript
// Infer array element type
type ArrayElement<T> = T extends (infer E)[] ? E : never;

type E1 = ArrayElement<string[]>;       // string
type E2 = ArrayElement<[number, string]>;  // number | string

// Infer first element of tuple
type First<T> = T extends [infer F, ...unknown[]] ? F : never;

type F2 = First<[1, 2, 3]>;  // 1

// Infer last element
type Last<T> = T extends [...unknown[], infer L] ? L : never;

type L1 = Last<[1, 2, 3]>;  // 3

// Infer rest elements
type Rest<T> = T extends [unknown, ...infer R] ? R : never;

type R3 = Rest<[1, 2, 3]>;  // [2, 3]

// Infer initial elements (all but last)
type Initial<T> = T extends [...infer I, unknown] ? I : never;

type I1 = Initial<[1, 2, 3]>;  // [1, 2]
```

### Promise and Async Inference

```typescript
// Unwrap Promise
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;

type A1 = Awaited<Promise<string>>;           // string
type A2 = Awaited<Promise<Promise<number>>>;  // number

// Infer async function return
type AsyncReturnType<T> = T extends (...args: any[]) => Promise<infer R>
  ? R
  : never;

type AR1 = AsyncReturnType<() => Promise<string>>;  // string

// Promisify function return
type PromisifyFn<T> = T extends (...args: infer A) => infer R
  ? (...args: A) => Promise<Awaited<R>>
  : never;
```

### Constructor and Instance Inference

```typescript
// Infer instance type
type InstanceType<T> = T extends new (...args: any[]) => infer R ? R : never;

class MyClass {
  value: string = "";
}

type Instance = InstanceType<typeof MyClass>;  // MyClass

// Infer constructor parameters
type ConstructorParameters<T> = T extends new (...args: infer P) => any
  ? P
  : never;

type CP1 = ConstructorParameters<typeof MyClass>;  // []

class Person {
  constructor(public name: string, public age: number) {}
}

type CP2 = ConstructorParameters<typeof Person>;  // [string, number]
```

### Complex Inference Patterns

```typescript
// Infer generic type parameter
type UnwrapArray<T> = T extends Array<infer U> ? U : T;
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type UnwrapBoth<T> = T extends Promise<Array<infer U>>
  ? U
  : T extends Array<infer V>
  ? V
  : T extends Promise<infer W>
  ? W
  : T;

// Infer function this type
type ThisParameterType<T> = T extends (this: infer U, ...args: any[]) => any
  ? U
  : unknown;

function greet(this: { name: string }): string {
  return `Hello, ${this.name}`;
}

type ThisType = ThisParameterType<typeof greet>;  // { name: string }

// Infer object value types
type ValueOf<T> = T[keyof T];

type PropType<T, K extends keyof T> = T extends { [P in K]: infer V } ? V : never;

interface Config {
  host: string;
  port: number;
  debug: boolean;
}

type HostType = PropType<Config, "host">;  // string
```

---

## 11. Variadic Tuple Types

Variadic tuple types allow working with tuples of variable length and spreading tuple types.

### Basic Variadic Tuples

```typescript
// Spread operator in tuple types
type Concat<T extends unknown[], U extends unknown[]> = [...T, ...U];

type C1 = Concat<[1, 2], [3, 4]>;  // [1, 2, 3, 4]

// Prepend element
type Prepend<T, U extends unknown[]> = [T, ...U];

type P1 = Prepend<0, [1, 2, 3]>;  // [0, 1, 2, 3]

// Append element
type Append<T extends unknown[], U> = [...T, U];

type A1 = Append<[1, 2, 3], 4>;  // [1, 2, 3, 4]
```

### Function Parameter Spreading

```typescript
// Partial application with variadic tuples
type PartiallyApply<
  T extends (...args: any[]) => any,
  Applied extends unknown[]
> = T extends (...args: [...Applied, ...infer Rest]) => infer R
  ? (...args: Rest) => R
  : never;

function sum(a: number, b: number, c: number): number {
  return a + b + c;
}

type SumWith1 = PartiallyApply<typeof sum, [number]>;
// (b: number, c: number) => number

type SumWith2 = PartiallyApply<typeof sum, [number, number]>;
// (c: number) => number

// Curry function type
type Curry<F> = F extends (...args: infer A) => infer R
  ? A extends [infer First, ...infer Rest]
    ? (arg: First) => Curry<(...args: Rest) => R>
    : R
  : never;
```

### Tuple Manipulation

```typescript
// Reverse tuple
type Reverse<T extends unknown[]> = T extends [infer First, ...infer Rest]
  ? [...Reverse<Rest>, First]
  : [];

type R1 = Reverse<[1, 2, 3]>;  // [3, 2, 1]

// Drop first N elements
type Drop<T extends unknown[], N extends number, I extends unknown[] = []> =
  I["length"] extends N
    ? T
    : T extends [unknown, ...infer Rest]
    ? Drop<Rest, N, [...I, unknown]>
    : [];

type D1 = Drop<[1, 2, 3, 4, 5], 2>;  // [3, 4, 5]

// Take first N elements
type Take<T extends unknown[], N extends number, R extends unknown[] = []> =
  R["length"] extends N
    ? R
    : T extends [infer First, ...infer Rest]
    ? Take<Rest, N, [...R, First]>
    : R;

type T1 = Take<[1, 2, 3, 4, 5], 3>;  // [1, 2, 3]

// Flatten tuple one level
type FlattenOnce<T extends unknown[]> = T extends [infer First, ...infer Rest]
  ? First extends unknown[]
    ? [...First, ...FlattenOnce<Rest>]
    : [First, ...FlattenOnce<Rest>]
  : [];

type F1 = FlattenOnce<[[1, 2], [3, 4], 5]>;  // [1, 2, 3, 4, 5]
```

### Generic Function with Variadic Tuples

```typescript
// Zip function
function zip<T extends unknown[], U extends unknown[]>(
  arr1: [...T],
  arr2: [...U]
): { [K in keyof T]: [T[K], K extends keyof U ? U[K] : undefined] } {
  return arr1.map((item, i) => [item, arr2[i]]) as any;
}

const zipped = zip([1, 2, 3], ["a", "b", "c"]);
// [[number, string], [number, string], [number, string]]

// Spread multiple arrays
function concat<T extends unknown[], U extends unknown[]>(
  ...args: [[...T], [...U]]
): [...T, ...U] {
  return [...args[0], ...args[1]] as [...T, ...U];
}

// Type-safe event emitter
type EventMap = {
  click: [x: number, y: number];
  keypress: [key: string];
  load: [];
};

class TypedEmitter<E extends Record<string, unknown[]>> {
  on<K extends keyof E>(event: K, listener: (...args: E[K]) => void): void {
    // Implementation
  }

  emit<K extends keyof E>(event: K, ...args: E[K]): void {
    // Implementation
  }
}

const emitter = new TypedEmitter<EventMap>();
emitter.on("click", (x, y) => console.log(x, y));  // x: number, y: number
emitter.emit("click", 100, 200);
```

---

## 12. Higher-Order Generic Patterns

Advanced patterns for creating flexible, reusable type abstractions.

### Functor Pattern

```typescript
// Functor interface
interface Functor<T> {
  map<U>(fn: (value: T) => U): Functor<U>;
}

// Maybe monad
class Maybe<T> implements Functor<T> {
  private constructor(private value: T | null) {}

  static just<T>(value: T): Maybe<T> {
    return new Maybe(value);
  }

  static nothing<T>(): Maybe<T> {
    return new Maybe<T>(null);
  }

  map<U>(fn: (value: T) => U): Maybe<U> {
    if (this.value === null) {
      return Maybe.nothing<U>();
    }
    return Maybe.just(fn(this.value));
  }

  flatMap<U>(fn: (value: T) => Maybe<U>): Maybe<U> {
    if (this.value === null) {
      return Maybe.nothing<U>();
    }
    return fn(this.value);
  }

  getOrElse(defaultValue: T): T {
    return this.value ?? defaultValue;
  }
}

const result = Maybe.just(5)
  .map((x) => x * 2)
  .map((x) => x.toString())
  .getOrElse("default");  // "10"
```

### Result/Either Pattern

```typescript
// Result type for error handling
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

function mapResult<T, U, E>(
  result: Result<T, E>,
  fn: (value: T) => U
): Result<U, E> {
  if (result.ok) {
    return ok(fn(result.value));
  }
  return result;
}

function flatMapResult<T, U, E>(
  result: Result<T, E>,
  fn: (value: T) => Result<U, E>
): Result<U, E> {
  if (result.ok) {
    return fn(result.value);
  }
  return result;
}

// Usage
function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return err("Division by zero");
  }
  return ok(a / b);
}

const computation = flatMapResult(
  divide(10, 2),
  (result) => divide(result, 2)
);  // { ok: true, value: 2.5 }
```

### Lens Pattern

```typescript
// Lens for functional updates
interface Lens<S, A> {
  get(s: S): A;
  set(a: A, s: S): S;
}

function lens<S, A>(
  get: (s: S) => A,
  set: (a: A, s: S) => S
): Lens<S, A> {
  return { get, set };
}

function composeLens<S, A, B>(
  outer: Lens<S, A>,
  inner: Lens<A, B>
): Lens<S, B> {
  return {
    get: (s) => inner.get(outer.get(s)),
    set: (b, s) => outer.set(inner.set(b, outer.get(s)), s),
  };
}

// Type-safe property lens
function prop<S, K extends keyof S>(key: K): Lens<S, S[K]> {
  return {
    get: (s) => s[key],
    set: (a, s) => ({ ...s, [key]: a }),
  };
}

// Usage
interface Address {
  street: string;
  city: string;
}

interface Person {
  name: string;
  address: Address;
}

const addressLens = prop<Person, "address">("address");
const cityLens = prop<Address, "city">("city");
const personCityLens = composeLens(addressLens, cityLens);

const person: Person = {
  name: "Alice",
  address: { street: "123 Main St", city: "Boston" },
};

const city = personCityLens.get(person);  // "Boston"
const updated = personCityLens.set("New York", person);
```

### Type-Safe Builder Pattern

```typescript
// Builder with type accumulation
type BuilderState = { [key: string]: unknown };

class TypedBuilder<T extends BuilderState> {
  private state: T;

  constructor(initial: T) {
    this.state = initial;
  }

  with<K extends string, V>(
    key: K,
    value: V
  ): TypedBuilder<T & { [P in K]: V }> {
    return new TypedBuilder({ ...this.state, [key]: value } as T & { [P in K]: V });
  }

  build(): T {
    return this.state;
  }
}

const built = new TypedBuilder({})
  .with("name", "Alice")
  .with("age", 30)
  .with("email", "alice@example.com")
  .build();
// Type: { name: string; age: number; email: string }
```

### Middleware Pattern

```typescript
// Type-safe middleware chain
type Context<T = unknown> = { data: T };
type Middleware<T, U> = (ctx: Context<T>, next: () => Promise<Context<U>>) => Promise<Context<U>>;

class Pipeline<T> {
  private middlewares: Middleware<unknown, unknown>[] = [];

  use<U>(middleware: Middleware<T, U>): Pipeline<U> {
    this.middlewares.push(middleware as Middleware<unknown, unknown>);
    return this as unknown as Pipeline<U>;
  }

  async execute(initial: T): Promise<Context<T>> {
    let index = -1;

    const dispatch = async (i: number, ctx: Context<unknown>): Promise<Context<unknown>> => {
      if (i <= index) {
        throw new Error("next() called multiple times");
      }
      index = i;

      if (i === this.middlewares.length) {
        return ctx;
      }

      const middleware = this.middlewares[i];
      return middleware(ctx, () => dispatch(i + 1, ctx));
    };

    return dispatch(0, { data: initial }) as Promise<Context<T>>;
  }
}
```

---

## 13. Generic Utility Functions

Common patterns and utility types that leverage generics for everyday TypeScript development.

### Object Utilities

```typescript
// Deep merge
type DeepMerge<T, U> = {
  [K in keyof T | keyof U]: K extends keyof U
    ? K extends keyof T
      ? T[K] extends object
        ? U[K] extends object
          ? DeepMerge<T[K], U[K]>
          : U[K]
        : U[K]
      : U[K]
    : K extends keyof T
    ? T[K]
    : never;
};

function deepMerge<T extends object, U extends object>(
  target: T,
  source: U
): DeepMerge<T, U> {
  const result = { ...target } as Record<string, unknown>;

  for (const key of Object.keys(source)) {
    const targetValue = (target as Record<string, unknown>)[key];
    const sourceValue = (source as Record<string, unknown>)[key];

    if (
      typeof targetValue === "object" &&
      targetValue !== null &&
      typeof sourceValue === "object" &&
      sourceValue !== null
    ) {
      result[key] = deepMerge(
        targetValue as object,
        sourceValue as object
      );
    } else {
      result[key] = sourceValue;
    }
  }

  return result as DeepMerge<T, U>;
}

// Safe object access
function get<T, K1 extends keyof T>(obj: T, k1: K1): T[K1];
function get<T, K1 extends keyof T, K2 extends keyof T[K1]>(
  obj: T,
  k1: K1,
  k2: K2
): T[K1][K2];
function get<
  T,
  K1 extends keyof T,
  K2 extends keyof T[K1],
  K3 extends keyof T[K1][K2]
>(obj: T, k1: K1, k2: K2, k3: K3): T[K1][K2][K3];
function get(obj: unknown, ...keys: string[]): unknown {
  return keys.reduce(
    (acc, key) => (acc as Record<string, unknown>)?.[key],
    obj
  );
}

// Type-safe Object.keys
function typedKeys<T extends object>(obj: T): (keyof T)[] {
  return Object.keys(obj) as (keyof T)[];
}

// Type-safe Object.entries
function typedEntries<T extends object>(obj: T): [keyof T, T[keyof T]][] {
  return Object.entries(obj) as [keyof T, T[keyof T]][];
}

// Type-safe Object.fromEntries
function typedFromEntries<K extends string, V>(
  entries: [K, V][]
): Record<K, V> {
  return Object.fromEntries(entries) as Record<K, V>;
}
```

### Array Utilities

```typescript
// Group by key
function groupBy<T, K extends keyof T>(
  array: T[],
  key: K
): Record<string, T[]> {
  return array.reduce((acc, item) => {
    const groupKey = String(item[key]);
    (acc[groupKey] = acc[groupKey] || []).push(item);
    return acc;
  }, {} as Record<string, T[]>);
}

// Unique by key
function uniqueBy<T, K extends keyof T>(array: T[], key: K): T[] {
  const seen = new Set<T[K]>();
  return array.filter((item) => {
    const value = item[key];
    if (seen.has(value)) return false;
    seen.add(value);
    return true;
  });
}

// Partition array
function partition<T>(
  array: T[],
  predicate: (item: T) => boolean
): [T[], T[]] {
  const truthy: T[] = [];
  const falsy: T[] = [];

  for (const item of array) {
    (predicate(item) ? truthy : falsy).push(item);
  }

  return [truthy, falsy];
}

// Chunk array
function chunk<T>(array: T[], size: number): T[][] {
  const chunks: T[][] = [];
  for (let i = 0; i < array.length; i += size) {
    chunks.push(array.slice(i, i + size));
  }
  return chunks;
}

// Sort by multiple keys
type SortKey<T> = keyof T | ((item: T) => unknown);

function sortBy<T>(array: T[], ...keys: SortKey<T>[]): T[] {
  return [...array].sort((a, b) => {
    for (const key of keys) {
      const aVal = typeof key === "function" ? key(a) : a[key];
      const bVal = typeof key === "function" ? key(b) : b[key];

      if (aVal < bVal) return -1;
      if (aVal > bVal) return 1;
    }
    return 0;
  });
}
```

### Function Utilities

```typescript
// Debounce with proper typing
function debounce<T extends (...args: unknown[]) => unknown>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timeoutId: ReturnType<typeof setTimeout>;

  return function (this: unknown, ...args: Parameters<T>) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Throttle with proper typing
function throttle<T extends (...args: unknown[]) => unknown>(
  fn: T,
  limit: number
): (...args: Parameters<T>) => void {
  let inThrottle = false;

  return function (this: unknown, ...args: Parameters<T>) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => (inThrottle = false), limit);
    }
  };
}

// Memoize with generic cache
function memoize<T extends (...args: unknown[]) => unknown>(
  fn: T,
  getKey: (...args: Parameters<T>) => string = (...args) => JSON.stringify(args)
): T {
  const cache = new Map<string, ReturnType<T>>();

  return function (this: unknown, ...args: Parameters<T>) {
    const key = getKey(...args);
    if (cache.has(key)) {
      return cache.get(key)!;
    }
    const result = fn.apply(this, args) as ReturnType<T>;
    cache.set(key, result);
    return result;
  } as T;
}

// Compose functions
function compose<T>(...fns: ((arg: T) => T)[]): (arg: T) => T {
  return (arg) => fns.reduceRight((acc, fn) => fn(acc), arg);
}

// Pipe functions
function pipe<T>(...fns: ((arg: T) => T)[]): (arg: T) => T {
  return (arg) => fns.reduce((acc, fn) => fn(acc), arg);
}

// Type-safe pipe with different types
type Pipe = {
  <A, B>(fn1: (a: A) => B): (a: A) => B;
  <A, B, C>(fn1: (a: A) => B, fn2: (b: B) => C): (a: A) => C;
  <A, B, C, D>(fn1: (a: A) => B, fn2: (b: B) => C, fn3: (c: C) => D): (a: A) => D;
};

const typedPipe: Pipe = (...fns: ((arg: unknown) => unknown)[]) => {
  return (arg: unknown) => fns.reduce((acc, fn) => fn(acc), arg);
};
```

### Async Utilities

```typescript
// Retry with exponential backoff
async function retry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    delay?: number;
    backoff?: number;
  } = {}
): Promise<T> {
  const { maxRetries = 3, delay = 1000, backoff = 2 } = options;

  let lastError: Error | undefined;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      if (attempt < maxRetries - 1) {
        await new Promise((resolve) =>
          setTimeout(resolve, delay * Math.pow(backoff, attempt))
        );
      }
    }
  }

  throw lastError;
}

// Timeout wrapper
function withTimeout<T>(
  promise: Promise<T>,
  ms: number,
  message = "Operation timed out"
): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error(message)), ms)
  );

  return Promise.race([promise, timeout]);
}

// Parallel limit
async function parallelLimit<T, R>(
  items: T[],
  limit: number,
  fn: (item: T) => Promise<R>
): Promise<R[]> {
  const results: R[] = [];
  const executing: Promise<void>[] = [];

  for (const item of items) {
    const p = fn(item).then((result) => {
      results.push(result);
    });

    executing.push(p);

    if (executing.length >= limit) {
      await Promise.race(executing);
      executing.splice(
        executing.findIndex((e) => e === p),
        1
      );
    }
  }

  await Promise.all(executing);
  return results;
}
```

---

## 14. Best Practices

### Naming Conventions

```typescript
// Use descriptive type parameter names for complex generics
// Single letter (T, U, K, V) for simple cases
function identity<T>(value: T): T {
  return value;
}

// Descriptive names for complex scenarios
interface Repository<TEntity extends Entity, TId = string> {
  findById(id: TId): Promise<TEntity | null>;
}

// Common conventions:
// T - Type
// K - Key
// V - Value
// E - Element
// R - Return type
// P - Properties/Parameters
// TEntity, TResult, TInput, TOutput - Prefixed descriptive names
```

### Keep Generics Simple

```typescript
// BAD: Overly complex generic
type BadComplexType<
  T extends object,
  K extends keyof T,
  V extends T[K],
  R extends V extends Array<infer U> ? U : V
> = {
  key: K;
  value: R;
};

// GOOD: Break into smaller, composable types
type ExtractArrayElement<T> = T extends Array<infer U> ? U : T;

type SimpleType<T extends object, K extends keyof T> = {
  key: K;
  value: ExtractArrayElement<T[K]>;
};
```

### Use Constraints Appropriately

```typescript
// BAD: No constraint when one is needed
function getLength<T>(item: T): number {
  return (item as { length: number }).length;  // Unsafe cast
}

// GOOD: Proper constraint
function getLength<T extends { length: number }>(item: T): number {
  return item.length;  // Safe access
}

// BAD: Over-constraining
function process<T extends string | number | boolean>(value: T): T {
  return value;  // Could just use the union type
}

// GOOD: Use generics when you need type preservation
function process(value: string | number | boolean): string | number | boolean {
  return value;
}

// Or when you need exact type preservation:
function preserve<T extends string | number | boolean>(value: T): T {
  return value;
}
const x = preserve("hello" as const);  // Type: "hello"
```

### Prefer Inference Over Explicit Types

```typescript
// BAD: Redundant type annotations
const result1 = identity<string>("hello");  // Type is obvious

// GOOD: Let TypeScript infer
const result2 = identity("hello");  // TypeScript infers string

// Exception: When inference gives too narrow a type
const numbers = [1, 2, 3];  // number[]
const tuple = [1, 2, 3] as const;  // readonly [1, 2, 3]

// Or when you want a specific type
const response = {} as Response<User>;  // When needed for setup
```

### Document Complex Generics

```typescript
/**
 * Creates a type-safe event emitter.
 *
 * @template TEvents - A record mapping event names to their argument tuples
 *
 * @example
 * ```typescript
 * type Events = {
 *   'user:login': [userId: string];
 *   'data:change': [oldValue: unknown, newValue: unknown];
 * };
 *
 * const emitter = new Emitter<Events>();
 * emitter.on('user:login', (userId) => console.log(userId));
 * emitter.emit('user:login', 'user123');
 * ```
 */
class Emitter<TEvents extends Record<string, unknown[]>> {
  on<K extends keyof TEvents>(
    event: K,
    handler: (...args: TEvents[K]) => void
  ): void {
    // Implementation
  }

  emit<K extends keyof TEvents>(event: K, ...args: TEvents[K]): void {
    // Implementation
  }
}
```

### Use Built-in Utility Types

```typescript
// TypeScript provides many utility types - use them!

// Instead of manually creating:
type ManualPartial<T> = { [K in keyof T]?: T[K] };

// Use built-in:
type User = {
  id: number;
  name: string;
  email: string;
};

// Partial<T> - make all properties optional
type PartialUser = Partial<User>;

// Required<T> - make all properties required
type RequiredUser = Required<PartialUser>;

// Readonly<T> - make all properties readonly
type ReadonlyUser = Readonly<User>;

// Pick<T, K> - select specific properties
type UserBasics = Pick<User, "id" | "name">;

// Omit<T, K> - exclude specific properties
type UserWithoutEmail = Omit<User, "email">;

// Record<K, V> - create object type with specific keys
type UserRoles = Record<string, "admin" | "user" | "guest">;

// ReturnType<T> - extract function return type
type CreateUserReturn = ReturnType<typeof createUser>;

// Parameters<T> - extract function parameters
type CreateUserParams = Parameters<typeof createUser>;

// Awaited<T> - unwrap Promise types
type UserData = Awaited<Promise<User>>;
```

---

## 15. Common Pitfalls

### Pitfall 1: Type Parameter Not Used

```typescript
// BAD: T is declared but could be inferred or isn't useful
function badExample<T>(value: string): string {
  return value.toUpperCase();
}

// GOOD: Remove unused type parameter
function goodExample(value: string): string {
  return value.toUpperCase();
}

// Or use it meaningfully
function betterExample<T extends string>(value: T): Uppercase<T> {
  return value.toUpperCase() as Uppercase<T>;
}
```

### Pitfall 2: Incorrect Constraint Usage

```typescript
// BAD: Constraint is too loose
function merge<T>(a: T, b: T): T {
  return { ...a, ...b };  // Error: spread only works on objects
}

// GOOD: Add object constraint
function merge<T extends object>(a: T, b: T): T {
  return { ...a, ...b };
}

// BAD: Constraint prevents valid use cases
function getFirst<T extends unknown[]>(arr: T): T[0] {
  return arr[0];
}

// This works but loses type information
const first = getFirst([1, "two", true]);  // Type: string | number | boolean

// GOOD: Use tuple inference
function getFirst<T extends readonly [unknown, ...unknown[]]>(arr: T): T[0] {
  return arr[0];
}

const first2 = getFirst([1, "two", true] as const);  // Type: 1
```

### Pitfall 3: Losing Type Information

```typescript
// BAD: Returns any/unknown unnecessarily
function process<T>(items: T[]): unknown[] {
  return items.map((item) => item);
}

// GOOD: Preserve types
function process<T>(items: T[]): T[] {
  return items.map((item) => item);
}

// BAD: Type assertion loses safety
function unsafeGet<T>(obj: object, key: string): T {
  return (obj as Record<string, T>)[key];  // Unsafe
}

// GOOD: Use proper constraints
function safeGet<T extends object, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

### Pitfall 4: Distributive Conditional Type Surprises

```typescript
// Conditional types distribute over union types
type ToArray<T> = T extends unknown ? T[] : never;

// This distributes!
type Result = ToArray<string | number>;
// Expected: (string | number)[]
// Actual: string[] | number[]

// To prevent distribution, wrap in tuple
type ToArrayNoDist<T> = [T] extends [unknown] ? T[] : never;

type Result2 = ToArrayNoDist<string | number>;
// Now it's: (string | number)[]

// Another example
type IsNever<T> = T extends never ? true : false;

type Test1 = IsNever<never>;  // never (not true!)
// Because never distributes to nothing

// Fix with tuple
type IsNeverFixed<T> = [T] extends [never] ? true : false;

type Test2 = IsNeverFixed<never>;  // true
```

### Pitfall 5: Circular Reference Issues

```typescript
// BAD: Infinite recursion
type BadDeepPartial<T> = {
  [K in keyof T]?: BadDeepPartial<T[K]>;  // Could infinitely recurse
};

// GOOD: Add base case
type DeepPartial<T> = T extends object
  ? T extends Function
    ? T
    : { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

// BAD: Self-referencing type causes issues
type BadTree<T> = {
  value: T;
  children: BadTree<T>[];  // Can cause issues in some cases
};

// GOOD: Use interface for recursive types (better for performance)
interface Tree<T> {
  value: T;
  children: Tree<T>[];
}
```

### Pitfall 6: Generic Function Inference Issues

```typescript
// BAD: TypeScript can't infer from callback parameters alone
function map<T, U>(arr: T[], fn: (item: T) => U): U[] {
  return arr.map(fn);
}

// This works
const result1 = map([1, 2, 3], (n) => n.toString());

// But this doesn't infer T
const result2 = map([], (n) => n.toString());  // n is unknown

// GOOD: Provide explicit types when needed
const result3 = map<number, string>([], (n) => n.toString());

// Or use overloads for better inference
function mapImproved<T, U>(arr: readonly T[], fn: (item: T, index: number) => U): U[];
function mapImproved<T, U>(arr: T[], fn: (item: T, index: number) => U): U[] {
  return arr.map(fn);
}
```

### Pitfall 7: Variance Issues

```typescript
// TypeScript uses structural typing which can cause variance issues

interface Animal {
  name: string;
}

interface Dog extends Animal {
  breed: string;
}

// BAD: Unsafe covariance in arrays
function addAnimal(animals: Animal[]): void {
  animals.push({ name: "Generic Animal" });  // Seems fine
}

const dogs: Dog[] = [{ name: "Buddy", breed: "Labrador" }];
addAnimal(dogs);  // TypeScript allows this!
// dogs[1].breed is now undefined - runtime error!

// GOOD: Use readonly for input arrays
function processAnimals(animals: readonly Animal[]): void {
  // Can't push, so no mutation issues
  animals.forEach((a) => console.log(a.name));
}

// For functions, use in/out modifiers (TypeScript 4.7+)
type Getter<out T> = () => T;  // Covariant
type Setter<in T> = (value: T) => void;  // Contravariant
type GetterSetter<in out T> = {  // Invariant
  get(): T;
  set(value: T): void;
};
```

### Pitfall 8: Object vs object vs {}

```typescript
// These are different!
type A = Object;  // Don't use - matches almost anything
type B = object;  // Non-primitive (no string, number, etc.)
type C = {};      // Any non-nullish value

// BAD: Using Object
function bad<T extends Object>(value: T): T {
  return value;
}
bad("string");  // Works - probably not intended

// GOOD: Using object for "any object"
function good<T extends object>(value: T): T {
  return value;
}
// good("string");  // Error - string is not an object

// For "any non-nullish value", use NonNullable<unknown>
function acceptAnything<T extends NonNullable<unknown>>(value: T): T {
  return value;
}
```

---

## Summary

TypeScript generics are a powerful feature that enables type-safe, reusable code. Key takeaways:

1. **Start simple**: Use basic generics before reaching for advanced patterns
2. **Use constraints wisely**: They provide safety but avoid over-constraining
3. **Leverage inference**: Let TypeScript infer types when possible
4. **Use built-in utility types**: Don't reinvent `Partial`, `Pick`, `Omit`, etc.
5. **Document complex types**: Help others (and future you) understand your code
6. **Be aware of common pitfalls**: Distribution, variance, and circular references
7. **Test your types**: Use tools like `type-testing` libraries to verify type behavior

Generics become more natural with practice. Start with simple use cases and gradually incorporate more advanced patterns as needed.
