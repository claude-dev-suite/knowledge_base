# TypeScript Utility Types

A comprehensive guide to TypeScript's built-in utility types for type transformations.

## Table of Contents

1. [Partial\<T\> and Required\<T\>](#1-partialt-and-requiredt)
2. [Readonly\<T\> and ReadonlyArray\<T\>](#2-readonlyt-and-readonlyarrayt)
3. [Record\<K, V\>](#3-recordk-v)
4. [Pick\<T, K\> and Omit\<T, K\>](#4-pickt-k-and-omitt-k)
5. [Exclude\<T, U\> and Extract\<T, U\>](#5-excludet-u-and-extractt-u)
6. [NonNullable\<T\>](#6-nonnullablet)
7. [Parameters\<T\> and ReturnType\<T\>](#7-parameterst-and-returntypet)
8. [ConstructorParameters\<T\> and InstanceType\<T\>](#8-constructorparameterst-and-instancetypet)
9. [ThisParameterType\<T\> and OmitThisParameter\<T\>](#9-thisparametertypet-and-omitthisparametert)
10. [Awaited\<T\>](#10-awaitedt)
11. [String Manipulation Types](#11-string-manipulation-types)
12. [NoInfer\<T\>](#12-noinfert-typescript-54)
13. [Custom Utility Types](#13-custom-utility-types)
14. [Combining Utility Types](#14-combining-utility-types)
15. [Real-world Patterns and Examples](#15-real-world-patterns-and-examples)
16. [Best Practices](#16-best-practices)

---

## 1. Partial\<T\> and Required\<T\>

### Partial\<T\>

Constructs a type with all properties of `T` set to optional. This utility returns a type that represents all subsets of a given type.

**Definition:**
```typescript
type Partial<T> = {
  [P in keyof T]?: T[P];
};
```

**Basic Usage:**
```typescript
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
}

type PartialUser = Partial<User>;
// Result:
// {
//   id?: string;
//   name?: string;
//   email?: string;
//   age?: number;
// }
```

**Common Use Cases:**

```typescript
// 1. Update functions - only pass fields to update
interface Product {
  id: string;
  name: string;
  price: number;
  description: string;
  inventory: number;
}

function updateProduct(id: string, updates: Partial<Product>): Product {
  const existing = getProductById(id);
  return { ...existing, ...updates };
}

// Can update any subset of fields
updateProduct('123', { price: 29.99 });
updateProduct('123', { name: 'New Name', inventory: 50 });

// 2. Optional configuration objects
interface DatabaseConfig {
  host: string;
  port: number;
  database: string;
  username: string;
  password: string;
  ssl: boolean;
}

function createConnection(config: Partial<DatabaseConfig> = {}): Connection {
  const defaults: DatabaseConfig = {
    host: 'localhost',
    port: 5432,
    database: 'mydb',
    username: 'root',
    password: '',
    ssl: false,
  };
  return connect({ ...defaults, ...config });
}

// 3. Form state management
interface FormFields {
  username: string;
  email: string;
  password: string;
  confirmPassword: string;
}

interface FormState {
  values: Partial<FormFields>;
  errors: Partial<Record<keyof FormFields, string>>;
  touched: Partial<Record<keyof FormFields, boolean>>;
}
```

### Required\<T\>

Constructs a type consisting of all properties of `T` set to required. The opposite of `Partial<T>`.

**Definition:**
```typescript
type Required<T> = {
  [P in keyof T]-?: T[P];
};
```

The `-?` modifier removes the optional modifier from properties.

**Basic Usage:**
```typescript
interface Config {
  host?: string;
  port?: number;
  timeout?: number;
}

type RequiredConfig = Required<Config>;
// Result:
// {
//   host: string;
//   port: number;
//   timeout: number;
// }
```

**Common Use Cases:**

```typescript
// 1. Ensuring all config values are provided after defaults
interface AppConfig {
  apiUrl?: string;
  timeout?: number;
  retries?: number;
  debug?: boolean;
}

function initializeApp(userConfig: AppConfig): Required<AppConfig> {
  const defaults: Required<AppConfig> = {
    apiUrl: 'https://api.example.com',
    timeout: 5000,
    retries: 3,
    debug: false,
  };
  return { ...defaults, ...userConfig };
}

// 2. Validating complete objects
interface UserInput {
  name?: string;
  email?: string;
  age?: number;
}

function validateUser(input: UserInput): input is Required<UserInput> {
  return input.name !== undefined
    && input.email !== undefined
    && input.age !== undefined;
}

function processUser(input: UserInput) {
  if (validateUser(input)) {
    // input is now Required<UserInput>
    console.log(input.name.toUpperCase()); // Safe access
  }
}

// 3. Database entity definitions
interface BaseEntity {
  id?: string;
  createdAt?: Date;
  updatedAt?: Date;
}

interface UserEntity extends BaseEntity {
  name: string;
  email: string;
}

// After saving to database, all fields are required
type SavedUserEntity = Required<UserEntity>;
```

---

## 2. Readonly\<T\> and ReadonlyArray\<T\>

### Readonly\<T\>

Constructs a type with all properties of `T` set to `readonly`, meaning the properties cannot be reassigned.

**Definition:**
```typescript
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

**Basic Usage:**
```typescript
interface Todo {
  title: string;
  description: string;
  completed: boolean;
}

type ReadonlyTodo = Readonly<Todo>;

const todo: ReadonlyTodo = {
  title: 'Learn TypeScript',
  description: 'Study utility types',
  completed: false,
};

// Error: Cannot assign to 'completed' because it is a read-only property
// todo.completed = true;
```

**Common Use Cases:**

```typescript
// 1. Immutable state in Redux/state management
interface AppState {
  user: User | null;
  posts: Post[];
  loading: boolean;
}

type ImmutableState = Readonly<AppState>;

function reducer(state: ImmutableState, action: Action): ImmutableState {
  switch (action.type) {
    case 'SET_USER':
      return { ...state, user: action.payload };
    default:
      return state;
  }
}

// 2. Configuration objects that shouldn't change
interface ServerConfig {
  host: string;
  port: number;
  environment: 'development' | 'production';
}

const config: Readonly<ServerConfig> = {
  host: 'localhost',
  port: 3000,
  environment: 'development',
};

// Prevents accidental modifications
// config.port = 8080; // Error!

// 3. Function parameters that shouldn't be mutated
function processData(data: Readonly<User[]>): Report {
  // Cannot modify the input array
  // data.push(newUser); // Error!
  return generateReport(data);
}
```

**Important Note - Shallow Readonly:**
```typescript
interface Nested {
  data: {
    value: number;
  };
}

const obj: Readonly<Nested> = {
  data: { value: 1 },
};

// This still works! Readonly is shallow
obj.data.value = 2; // No error

// For deep readonly, you need a custom type (see Custom Utility Types section)
```

### ReadonlyArray\<T\>

A type representing an array that cannot be mutated. No push, pop, or index assignment.

**Definition:**
```typescript
interface ReadonlyArray<T> {
  readonly length: number;
  readonly [n: number]: T;
  // ... iterator methods that return new arrays
}
```

**Basic Usage:**
```typescript
const numbers: ReadonlyArray<number> = [1, 2, 3, 4, 5];

// Error: Property 'push' does not exist on type 'readonly number[]'
// numbers.push(6);

// Error: Index signature in type 'readonly number[]' only permits reading
// numbers[0] = 10;

// These work - they return new arrays
const doubled = numbers.map(n => n * 2);
const filtered = numbers.filter(n => n > 2);
```

**Alternative Syntax:**
```typescript
// These are equivalent
const arr1: ReadonlyArray<number> = [1, 2, 3];
const arr2: readonly number[] = [1, 2, 3];
```

**Common Use Cases:**

```typescript
// 1. Immutable function parameters
function calculateSum(numbers: readonly number[]): number {
  return numbers.reduce((sum, n) => sum + n, 0);
}

// 2. Tuple types as readonly
type Point = readonly [number, number];
const origin: Point = [0, 0];
// origin[0] = 1; // Error!

// 3. Preventing array mutation in class properties
class UserList {
  private _users: User[] = [];

  get users(): ReadonlyArray<User> {
    return this._users;
  }

  addUser(user: User): void {
    this._users.push(user);
  }
}

const list = new UserList();
// list.users.push(newUser); // Error - cannot modify the exposed array
```

---

## 3. Record\<K, V\>

Constructs an object type whose property keys are `K` and whose property values are `V`. This utility is useful for mapping properties of a type to another type.

**Definition:**
```typescript
type Record<K extends keyof any, T> = {
  [P in K]: T;
};
```

**Basic Usage:**
```typescript
// String literal union as keys
type Role = 'admin' | 'user' | 'guest';

type RolePermissions = Record<Role, string[]>;

const permissions: RolePermissions = {
  admin: ['read', 'write', 'delete', 'manage'],
  user: ['read', 'write'],
  guest: ['read'],
};

// Every key in the union must be present
// Missing 'guest' would cause an error
```

**Various Key Types:**

```typescript
// 1. String keys - creates index signature
type StringMap = Record<string, number>;
const counts: StringMap = {
  apples: 5,
  oranges: 3,
  // Can add any string key
};

// 2. Number keys
type NumberMap = Record<number, string>;
const names: NumberMap = {
  1: 'one',
  2: 'two',
  3: 'three',
};

// 3. Enum keys
enum Status {
  Pending = 'pending',
  Active = 'active',
  Completed = 'completed',
}

type StatusColors = Record<Status, string>;
const colors: StatusColors = {
  [Status.Pending]: 'yellow',
  [Status.Active]: 'green',
  [Status.Completed]: 'blue',
};

// 4. Template literal keys
type EventName = `on${Capitalize<'click' | 'hover' | 'focus'>}`;
// 'onClick' | 'onHover' | 'onFocus'

type EventHandlers = Record<EventName, () => void>;
```

**Common Use Cases:**

```typescript
// 1. Lookup tables
type HttpStatusCode = 200 | 201 | 400 | 401 | 404 | 500;

const statusMessages: Record<HttpStatusCode, string> = {
  200: 'OK',
  201: 'Created',
  400: 'Bad Request',
  401: 'Unauthorized',
  404: 'Not Found',
  500: 'Internal Server Error',
};

// 2. Feature flags
type FeatureFlag = 'darkMode' | 'betaFeatures' | 'analytics';

type FeatureConfig = Record<FeatureFlag, {
  enabled: boolean;
  rolloutPercentage: number;
}>;

const features: FeatureConfig = {
  darkMode: { enabled: true, rolloutPercentage: 100 },
  betaFeatures: { enabled: true, rolloutPercentage: 10 },
  analytics: { enabled: false, rolloutPercentage: 0 },
};

// 3. Caching/memoization
type CacheKey = string;
type CacheEntry<T> = {
  value: T;
  timestamp: number;
  ttl: number;
};

type Cache<T> = Record<CacheKey, CacheEntry<T>>;

// 4. Localization/i18n
type Locale = 'en' | 'es' | 'fr' | 'de';
type TranslationKey = 'greeting' | 'farewell' | 'error';

type Translations = Record<Locale, Record<TranslationKey, string>>;

const translations: Translations = {
  en: { greeting: 'Hello', farewell: 'Goodbye', error: 'Error' },
  es: { greeting: 'Hola', farewell: 'Adiós', error: 'Error' },
  fr: { greeting: 'Bonjour', farewell: 'Au revoir', error: 'Erreur' },
  de: { greeting: 'Hallo', farewell: 'Auf Wiedersehen', error: 'Fehler' },
};
```

---

## 4. Pick\<T, K\> and Omit\<T, K\>

### Pick\<T, K\>

Constructs a type by picking a set of properties `K` from `T`.

**Definition:**
```typescript
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};
```

**Basic Usage:**
```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
  updatedAt: Date;
}

// Select only public-facing properties
type PublicUser = Pick<User, 'id' | 'name' | 'email'>;
// Result:
// {
//   id: string;
//   name: string;
//   email: string;
// }
```

**Common Use Cases:**

```typescript
// 1. API response shaping
interface FullProduct {
  id: string;
  name: string;
  description: string;
  price: number;
  cost: number;        // Internal - don't expose
  supplierId: string;  // Internal - don't expose
  inventory: number;
  createdAt: Date;
}

type ProductListItem = Pick<FullProduct, 'id' | 'name' | 'price'>;
type ProductDetail = Pick<FullProduct, 'id' | 'name' | 'description' | 'price' | 'inventory'>;

// 2. Form field subsets
interface RegistrationForm {
  username: string;
  email: string;
  password: string;
  confirmPassword: string;
  firstName: string;
  lastName: string;
  phone: string;
  address: string;
}

type LoginCredentials = Pick<RegistrationForm, 'email' | 'password'>;
type BasicInfo = Pick<RegistrationForm, 'firstName' | 'lastName' | 'email'>;

// 3. Component props from larger interfaces
interface ButtonProps {
  onClick: () => void;
  disabled: boolean;
  loading: boolean;
  variant: 'primary' | 'secondary';
  size: 'small' | 'medium' | 'large';
  children: React.ReactNode;
}

type IconButtonProps = Pick<ButtonProps, 'onClick' | 'disabled' | 'size'> & {
  icon: React.ReactNode;
};
```

### Omit\<T, K\>

Constructs a type by picking all properties from `T` and then removing `K`. The opposite of `Pick`.

**Definition:**
```typescript
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

**Basic Usage:**
```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
}

// Remove sensitive field
type UserWithoutPassword = Omit<User, 'password'>;
// Result:
// {
//   id: string;
//   name: string;
//   email: string;
// }

// Remove multiple fields
type CreateUserDto = Omit<User, 'id' | 'createdAt' | 'updatedAt'>;
```

**Common Use Cases:**

```typescript
// 1. DTOs (Data Transfer Objects)
interface DatabaseUser {
  id: string;
  name: string;
  email: string;
  passwordHash: string;
  salt: string;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

// For creating new users - no id or timestamps
type CreateUserDto = Omit<DatabaseUser, 'id' | 'createdAt' | 'updatedAt' | 'deletedAt'>;

// For API responses - no sensitive data
type UserResponse = Omit<DatabaseUser, 'passwordHash' | 'salt' | 'deletedAt'>;

// 2. Extending base types with modifications
interface BaseEvent {
  id: string;
  type: string;
  timestamp: Date;
  payload: unknown;
}

// Replace the payload type
type UserEvent = Omit<BaseEvent, 'payload'> & {
  payload: {
    userId: string;
    action: string;
  };
};

// 3. React component prop manipulation
type InputProps = React.InputHTMLAttributes<HTMLInputElement>;

// Custom input with controlled value prop
type CustomInputProps = Omit<InputProps, 'value' | 'onChange'> & {
  value: string;
  onChange: (value: string) => void;
};

// 4. Removing index signatures
interface ApiResponse {
  data: unknown;
  status: number;
  [key: string]: unknown; // Index signature
}

// This is tricky - Omit doesn't remove index signatures
// You need a custom utility for that (see Custom Utility Types)
```

**Pick vs Omit - When to Use Which:**

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  role: string;
  permissions: string[];
  createdAt: Date;
  updatedAt: Date;
}

// Use Pick when you want a small subset
type UserCredentials = Pick<User, 'email' | 'password'>; // 2 of 8 fields

// Use Omit when you want most fields minus a few
type SafeUser = Omit<User, 'password'>; // 7 of 8 fields

// Rule of thumb: Use whichever results in listing fewer properties
```

---

## 5. Exclude\<T, U\> and Extract\<T, U\>

These utilities work on **union types**, not object types.

### Exclude\<T, U\>

Constructs a type by excluding from `T` all union members that are assignable to `U`.

**Definition:**
```typescript
type Exclude<T, U> = T extends U ? never : T;
```

**Basic Usage:**
```typescript
type Status = 'pending' | 'approved' | 'rejected' | 'cancelled';

// Remove 'cancelled' from the union
type ActiveStatus = Exclude<Status, 'cancelled'>;
// Result: 'pending' | 'approved' | 'rejected'

// Remove multiple types
type PositiveStatus = Exclude<Status, 'rejected' | 'cancelled'>;
// Result: 'pending' | 'approved'
```

**Common Use Cases:**

```typescript
// 1. Filtering out null/undefined (similar to NonNullable)
type MaybeValues = string | number | null | undefined;
type DefiniteValues = Exclude<MaybeValues, null | undefined>;
// Result: string | number

// 2. Removing specific types from unions
type Primitive = string | number | boolean | symbol | bigint;
type NumericPrimitive = Exclude<Primitive, string | boolean | symbol>;
// Result: number | bigint

// 3. Event type filtering
type MouseEvent = 'click' | 'dblclick' | 'mousedown' | 'mouseup' | 'mousemove';
type ClickEvents = Exclude<MouseEvent, 'mousemove'>;
// Result: 'click' | 'dblclick' | 'mousedown' | 'mouseup'

// 4. Permission filtering
type Permission = 'read' | 'write' | 'delete' | 'admin';
type UserPermission = Exclude<Permission, 'admin'>;
// Result: 'read' | 'write' | 'delete'

// 5. Function type filtering
type AllTypes = string | number | (() => void) | { name: string };
type NonFunctionTypes = Exclude<AllTypes, Function>;
// Result: string | number | { name: string }
```

### Extract\<T, U\>

Constructs a type by extracting from `T` all union members that are assignable to `U`. The opposite of `Exclude`.

**Definition:**
```typescript
type Extract<T, U> = T extends U ? T : never;
```

**Basic Usage:**
```typescript
type AllTypes = string | number | boolean | (() => void);

// Extract only function types
type FunctionTypes = Extract<AllTypes, Function>;
// Result: () => void

// Extract string and number
type TextOrNumber = Extract<AllTypes, string | number>;
// Result: string | number
```

**Common Use Cases:**

```typescript
// 1. Finding common types between unions
type SetA = 'a' | 'b' | 'c' | 'd';
type SetB = 'b' | 'd' | 'e' | 'f';

type CommonTypes = Extract<SetA, SetB>;
// Result: 'b' | 'd'

// 2. Extracting specific categories
type AllEvents =
  | { type: 'CLICK'; x: number; y: number }
  | { type: 'KEYPRESS'; key: string }
  | { type: 'SCROLL'; offset: number }
  | { type: 'RESIZE'; width: number; height: number };

// Extract events with specific type
type ClickEvent = Extract<AllEvents, { type: 'CLICK' }>;
// Result: { type: 'CLICK'; x: number; y: number }

// 3. Extracting callable types
type Mixed = string | number | (() => string) | ((x: number) => void);
type Callables = Extract<Mixed, (...args: any[]) => any>;
// Result: (() => string) | ((x: number) => void)

// 4. Type guards with discriminated unions
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string };

type SuccessResult<T> = Extract<Result<T>, { success: true }>;
// Result: { success: true; data: T }

type FailureResult = Extract<Result<unknown>, { success: false }>;
// Result: { success: false; error: string }

// 5. Filtering object types from unions
type Values = string | number | { id: string } | { name: string };
type ObjectValues = Extract<Values, object>;
// Result: { id: string } | { name: string }
```

**Combining Exclude and Extract:**

```typescript
// Get all keys of T that have values assignable to V
type KeysOfType<T, V> = {
  [K in keyof T]: T[K] extends V ? K : never;
}[keyof T];

interface User {
  id: string;
  name: string;
  age: number;
  isActive: boolean;
  score: number;
}

type StringKeys = KeysOfType<User, string>;
// Result: 'id' | 'name'

type NumberKeys = KeysOfType<User, number>;
// Result: 'age' | 'score'
```

---

## 6. NonNullable\<T\>

Constructs a type by excluding `null` and `undefined` from `T`.

**Definition:**
```typescript
type NonNullable<T> = T & {};
// Or equivalently:
// type NonNullable<T> = T extends null | undefined ? never : T;
```

**Basic Usage:**
```typescript
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>;
// Result: string

type MaybeUser = User | null | undefined;
type DefiniteUser = NonNullable<MaybeUser>;
// Result: User
```

**Common Use Cases:**

```typescript
// 1. Working with optional values
interface Config {
  apiKey?: string;
  timeout?: number;
}

function getApiKey(config: Config): NonNullable<Config['apiKey']> {
  if (!config.apiKey) {
    throw new Error('API key is required');
  }
  return config.apiKey;
}

// 2. Array filtering
const values: (string | null | undefined)[] = ['a', null, 'b', undefined, 'c'];

// Filter removes nullish values, but TypeScript doesn't know that
const filtered = values.filter((v): v is NonNullable<typeof v> => v != null);
// filtered is now string[]

// 3. Map/dictionary lookups
type UserMap = Record<string, User | undefined>;

function getUser(map: UserMap, id: string): NonNullable<UserMap[string]> {
  const user = map[id];
  if (!user) {
    throw new Error(`User ${id} not found`);
  }
  return user;
}

// 4. Promise resolution
async function fetchData(): Promise<Data | null> {
  // ...
}

async function requireData(): Promise<NonNullable<Awaited<ReturnType<typeof fetchData>>>> {
  const data = await fetchData();
  if (!data) {
    throw new Error('Data not found');
  }
  return data;
}

// 5. Type assertions after validation
function processValue<T>(value: T | null | undefined): NonNullable<T> {
  if (value == null) {
    throw new Error('Value is required');
  }
  return value as NonNullable<T>;
}
```

---

## 7. Parameters\<T\> and ReturnType\<T\>

### Parameters\<T\>

Constructs a tuple type from the types used in the parameters of a function type `T`.

**Definition:**
```typescript
type Parameters<T extends (...args: any) => any> =
  T extends (...args: infer P) => any ? P : never;
```

**Basic Usage:**
```typescript
function greet(name: string, age: number): string {
  return `Hello ${name}, you are ${age} years old`;
}

type GreetParams = Parameters<typeof greet>;
// Result: [name: string, age: number]

// Access individual parameters
type FirstParam = Parameters<typeof greet>[0];  // string
type SecondParam = Parameters<typeof greet>[1]; // number
```

**Common Use Cases:**

```typescript
// 1. Creating wrapper functions
function originalFn(a: string, b: number, c: boolean): void {
  // ...
}

function wrapper(...args: Parameters<typeof originalFn>): void {
  console.log('Before');
  originalFn(...args);
  console.log('After');
}

// 2. Event handler types
type EventHandler = (event: MouseEvent, data: { id: string }) => void;
type EventParams = Parameters<EventHandler>;
// Result: [event: MouseEvent, data: { id: string }]

// 3. Testing - matching function signatures
function apiCall(endpoint: string, options: RequestInit): Promise<Response> {
  return fetch(endpoint, options);
}

// In tests, ensure mock matches signature
const mockApiCall = jest.fn<
  ReturnType<typeof apiCall>,
  Parameters<typeof apiCall>
>();

// 4. Partial application / currying
function add(a: number, b: number, c: number): number {
  return a + b + c;
}

type AddParams = Parameters<typeof add>;

function partialAdd(a: AddParams[0]): (b: AddParams[1], c: AddParams[2]) => number {
  return (b, c) => add(a, b, c);
}

// 5. Extracting callback parameters
type Callback = (error: Error | null, result: string) => void;
type CallbackError = Parameters<Callback>[0];   // Error | null
type CallbackResult = Parameters<Callback>[1];  // string
```

### ReturnType\<T\>

Constructs a type consisting of the return type of function `T`.

**Definition:**
```typescript
type ReturnType<T extends (...args: any) => any> =
  T extends (...args: any) => infer R ? R : any;
```

**Basic Usage:**
```typescript
function createUser() {
  return {
    id: '123',
    name: 'John',
    createdAt: new Date(),
  };
}

type User = ReturnType<typeof createUser>;
// Result: { id: string; name: string; createdAt: Date }
```

**Common Use Cases:**

```typescript
// 1. Inferring types from factory functions
function createStore() {
  return {
    state: { count: 0 },
    increment() { this.state.count++; },
    decrement() { this.state.count--; },
    getCount() { return this.state.count; },
  };
}

type Store = ReturnType<typeof createStore>;

// 2. API response types
async function fetchUsers(): Promise<{ users: User[]; total: number }> {
  // ...
}

type FetchUsersResponse = ReturnType<typeof fetchUsers>;
// Result: Promise<{ users: User[]; total: number }>

// For the unwrapped type, combine with Awaited
type UsersData = Awaited<ReturnType<typeof fetchUsers>>;
// Result: { users: User[]; total: number }

// 3. Redux action creators
function createAction(type: string, payload: unknown) {
  return { type, payload, timestamp: Date.now() };
}

type Action = ReturnType<typeof createAction>;

// 4. Builder pattern return types
class QueryBuilder {
  where(condition: string): this { /* ... */ return this; }
  orderBy(field: string): this { /* ... */ return this; }
  limit(n: number): this { /* ... */ return this; }
  execute(): Promise<unknown[]> { /* ... */ }
}

type QueryResult = ReturnType<QueryBuilder['execute']>;
// Result: Promise<unknown[]>

// 5. Higher-order function return types
function createLogger(prefix: string) {
  return (message: string) => console.log(`${prefix}: ${message}`);
}

type Logger = ReturnType<typeof createLogger>;
// Result: (message: string) => void
```

---

## 8. ConstructorParameters\<T\> and InstanceType\<T\>

### ConstructorParameters\<T\>

Constructs a tuple type from the types of a constructor function type's parameters.

**Definition:**
```typescript
type ConstructorParameters<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: infer P) => any ? P : never;
```

**Basic Usage:**
```typescript
class User {
  constructor(
    public id: string,
    public name: string,
    public email: string
  ) {}
}

type UserConstructorParams = ConstructorParameters<typeof User>;
// Result: [id: string, name: string, email: string]
```

**Common Use Cases:**

```typescript
// 1. Factory functions
class Product {
  constructor(
    public name: string,
    public price: number,
    public category: string
  ) {}
}

function createProduct(...args: ConstructorParameters<typeof Product>): Product {
  return new Product(...args);
}

// 2. Dependency injection containers
type Constructor<T = any> = new (...args: any[]) => T;

class Container {
  private instances = new Map<Constructor, any>();

  register<T>(
    ctor: Constructor<T>,
    ...args: ConstructorParameters<Constructor<T>>
  ): void {
    this.instances.set(ctor, new ctor(...args));
  }

  resolve<T>(ctor: Constructor<T>): T {
    return this.instances.get(ctor);
  }
}

// 3. Testing - mocking constructors
class DatabaseConnection {
  constructor(
    host: string,
    port: number,
    credentials: { user: string; password: string }
  ) {}
}

type DBParams = ConstructorParameters<typeof DatabaseConnection>;
// [string, number, { user: string; password: string }]

function createMockConnection(...args: DBParams): DatabaseConnection {
  // Create mock with same signature
  return {} as DatabaseConnection;
}

// 4. Abstract class parameters
abstract class BaseService {
  constructor(protected apiUrl: string, protected timeout: number) {}
  abstract fetch(): Promise<unknown>;
}

type BaseServiceParams = ConstructorParameters<typeof BaseService>;
// [apiUrl: string, timeout: number]
```

### InstanceType\<T\>

Constructs a type consisting of the instance type of a constructor function type `T`.

**Definition:**
```typescript
type InstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : any;
```

**Basic Usage:**
```typescript
class User {
  id: string;
  name: string;

  constructor(id: string, name: string) {
    this.id = id;
    this.name = name;
  }

  greet(): string {
    return `Hello, ${this.name}`;
  }
}

type UserInstance = InstanceType<typeof User>;
// Result: User (the instance type, not the constructor)
```

**Common Use Cases:**

```typescript
// 1. Generic factory functions
function createInstance<T extends new (...args: any[]) => any>(
  ctor: T,
  ...args: ConstructorParameters<T>
): InstanceType<T> {
  return new ctor(...args);
}

const user = createInstance(User, '1', 'John');
// user is User

// 2. Type-safe service locator
type ServiceConstructor = new (...args: any[]) => any;

class ServiceLocator {
  private services = new Map<ServiceConstructor, any>();

  register<T extends ServiceConstructor>(
    ctor: T,
    instance: InstanceType<T>
  ): void {
    this.services.set(ctor, instance);
  }

  get<T extends ServiceConstructor>(ctor: T): InstanceType<T> {
    const instance = this.services.get(ctor);
    if (!instance) {
      throw new Error(`Service not registered: ${ctor.name}`);
    }
    return instance;
  }
}

// 3. Mixin patterns
type AnyConstructor = new (...args: any[]) => any;

function Timestamped<T extends AnyConstructor>(Base: T) {
  return class extends Base {
    createdAt = new Date();
    updatedAt = new Date();
  };
}

class BaseUser {
  constructor(public name: string) {}
}

const TimestampedUser = Timestamped(BaseUser);
type TimestampedUserInstance = InstanceType<typeof TimestampedUser>;

// 4. Abstract factory pattern
abstract class Animal {
  abstract speak(): string;
}

class Dog extends Animal {
  speak() { return 'Woof!'; }
}

class Cat extends Animal {
  speak() { return 'Meow!'; }
}

type AnimalConstructor = new () => Animal;

function createAnimal<T extends AnimalConstructor>(ctor: T): InstanceType<T> {
  return new ctor();
}
```

---

## 9. ThisParameterType\<T\> and OmitThisParameter\<T\>

### ThisParameterType\<T\>

Extracts the type of the `this` parameter from a function type, or `unknown` if the function has no `this` parameter.

**Definition:**
```typescript
type ThisParameterType<T> =
  T extends (this: infer U, ...args: never) => any ? U : unknown;
```

**Basic Usage:**
```typescript
function toHex(this: Number) {
  return this.toString(16);
}

type ThisType = ThisParameterType<typeof toHex>;
// Result: Number

// Function without this parameter
function add(a: number, b: number): number {
  return a + b;
}

type NoThis = ThisParameterType<typeof add>;
// Result: unknown
```

**Common Use Cases:**

```typescript
// 1. Method extraction with correct this binding
interface Calculator {
  value: number;
  add(this: Calculator, n: number): number;
  multiply(this: Calculator, n: number): number;
}

const calculator: Calculator = {
  value: 10,
  add(n) { return this.value + n; },
  multiply(n) { return this.value * n; },
};

type AddThis = ThisParameterType<Calculator['add']>;
// Result: Calculator

// 2. Event handler contexts
interface EventEmitter {
  on(this: EventEmitter, event: string, callback: Function): this;
  emit(this: EventEmitter, event: string, ...args: any[]): boolean;
}

type EmitterThis = ThisParameterType<EventEmitter['on']>;
// Result: EventEmitter

// 3. jQuery-style chaining
interface ChainableQuery {
  find(this: ChainableQuery, selector: string): ChainableQuery;
  addClass(this: ChainableQuery, className: string): ChainableQuery;
  removeClass(this: ChainableQuery, className: string): ChainableQuery;
}
```

### OmitThisParameter\<T\>

Removes the `this` parameter from a function type. Creates a new function type without `this`.

**Definition:**
```typescript
type OmitThisParameter<T> =
  unknown extends ThisParameterType<T>
    ? T
    : T extends (...args: infer A) => infer R
      ? (...args: A) => R
      : T;
```

**Basic Usage:**
```typescript
function toHex(this: Number) {
  return this.toString(16);
}

type ToHexWithoutThis = OmitThisParameter<typeof toHex>;
// Result: () => string

// Can now use without binding
const boundToHex: ToHexWithoutThis = toHex.bind(42);
boundToHex(); // "2a"
```

**Common Use Cases:**

```typescript
// 1. Creating standalone functions from methods
class Logger {
  prefix: string = '[LOG]';

  log(this: Logger, message: string): void {
    console.log(`${this.prefix} ${message}`);
  }
}

const logger = new Logger();

// Extract method type without this
type LogFn = OmitThisParameter<typeof logger.log>;
// Result: (message: string) => void

// Create bound version
const boundLog: LogFn = logger.log.bind(logger);

// 2. Callback registration
interface Service {
  onData(this: Service, handler: (data: unknown) => void): void;
}

type DataHandler = OmitThisParameter<Service['onData']>;
// Result: (handler: (data: unknown) => void) => void

// 3. Function composition without this constraints
interface MathOps {
  double(this: MathOps, n: number): number;
  square(this: MathOps, n: number): number;
}

type PureMathFn = OmitThisParameter<MathOps['double']>;
// Result: (n: number) => number

function compose(
  f: PureMathFn,
  g: PureMathFn
): PureMathFn {
  return (n) => f(g(n));
}

// 4. React event handlers
interface ButtonComponent {
  handleClick(this: ButtonComponent, event: MouseEvent): void;
}

type ClickHandler = OmitThisParameter<ButtonComponent['handleClick']>;
// Result: (event: MouseEvent) => void

// Can be used as a regular callback
const onClick: ClickHandler = (event) => {
  console.log('Clicked at', event.clientX, event.clientY);
};
```

---

## 10. Awaited\<T\>

Unwraps the type that a `Promise` resolves to. Works recursively for nested promises.

**Definition (simplified):**
```typescript
type Awaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any } ?
    F extends ((value: infer V, ...args: infer _) => any) ?
      Awaited<V> :
      never :
  T;
```

**Basic Usage:**
```typescript
type StringPromise = Promise<string>;
type ResolvedString = Awaited<StringPromise>;
// Result: string

// Works with nested promises
type NestedPromise = Promise<Promise<number>>;
type ResolvedNumber = Awaited<NestedPromise>;
// Result: number

// Works with deeply nested
type DeepPromise = Promise<Promise<Promise<boolean>>>;
type ResolvedBoolean = Awaited<DeepPromise>;
// Result: boolean
```

**Common Use Cases:**

```typescript
// 1. Extracting async function return types
async function fetchUser(id: string): Promise<{ id: string; name: string }> {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

type User = Awaited<ReturnType<typeof fetchUser>>;
// Result: { id: string; name: string }

// 2. Promise.all result types
async function fetchData() {
  const results = await Promise.all([
    fetchUser('1'),
    fetchPosts(),
    fetchComments(),
  ]);
  return results;
}

type DataResults = Awaited<ReturnType<typeof fetchData>>;
// Result: [User, Post[], Comment[]]

// 3. Generic async utilities
async function retry<T>(
  fn: () => Promise<T>,
  attempts: number
): Promise<Awaited<T>> {
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (e) {
      if (i === attempts - 1) throw e;
    }
  }
  throw new Error('Unreachable');
}

// 4. Thenable types (not just Promise)
interface CustomThenable<T> {
  then<R>(
    onfulfilled: (value: T) => R | PromiseLike<R>
  ): CustomThenable<R>;
}

type ResolvedCustom = Awaited<CustomThenable<string>>;
// Result: string

// 5. Type-safe async wrappers
type AsyncFunction<T> = () => Promise<T>;

function createCachedFetcher<T extends AsyncFunction<any>>(
  fn: T
): () => Promise<Awaited<ReturnType<T>>> {
  let cache: Awaited<ReturnType<T>> | null = null;

  return async () => {
    if (cache === null) {
      cache = await fn();
    }
    return cache;
  };
}

// 6. Working with Promise.race
type RaceResult<T extends readonly Promise<any>[]> = Awaited<T[number]>;

const promises = [
  Promise.resolve(1),
  Promise.resolve('hello'),
  Promise.resolve(true),
] as const;

type Result = RaceResult<typeof promises>;
// Result: string | number | boolean
```

---

## 11. String Manipulation Types

TypeScript provides built-in types for manipulating string literal types.

### Uppercase\<S\>

Converts string literal type to uppercase.

```typescript
type Greeting = 'hello world';
type LoudGreeting = Uppercase<Greeting>;
// Result: 'HELLO WORLD'

type Status = 'pending' | 'active' | 'completed';
type UpperStatus = Uppercase<Status>;
// Result: 'PENDING' | 'ACTIVE' | 'COMPLETED'
```

### Lowercase\<S\>

Converts string literal type to lowercase.

```typescript
type Header = 'CONTENT-TYPE';
type LowerHeader = Lowercase<Header>;
// Result: 'content-type'

type Methods = 'GET' | 'POST' | 'PUT' | 'DELETE';
type LowerMethods = Lowercase<Methods>;
// Result: 'get' | 'post' | 'put' | 'delete'
```

### Capitalize\<S\>

Converts first character to uppercase.

```typescript
type Name = 'john';
type ProperName = Capitalize<Name>;
// Result: 'John'

type Events = 'click' | 'hover' | 'focus';
type CapitalizedEvents = Capitalize<Events>;
// Result: 'Click' | 'Hover' | 'Focus'
```

### Uncapitalize\<S\>

Converts first character to lowercase.

```typescript
type ClassName = 'UserService';
type InstanceName = Uncapitalize<ClassName>;
// Result: 'userService'

type Components = 'Button' | 'Input' | 'Select';
type CamelComponents = Uncapitalize<Components>;
// Result: 'button' | 'input' | 'select'
```

### Common Use Cases

```typescript
// 1. Event handler naming conventions
type DOMEvent = 'click' | 'focus' | 'blur' | 'change';
type EventHandler = `on${Capitalize<DOMEvent>}`;
// Result: 'onClick' | 'onFocus' | 'onBlur' | 'onChange'

type EventHandlers = {
  [K in EventHandler]?: (event: Event) => void;
};

// 2. CSS-in-JS property mapping
type CSSProperty = 'margin' | 'padding' | 'border';
type Direction = 'Top' | 'Right' | 'Bottom' | 'Left';

type DirectionalCSS = `${CSSProperty}${Direction}`;
// Result: 'marginTop' | 'marginRight' | ... | 'borderLeft'

// 3. HTTP methods
type HTTPMethod = 'get' | 'post' | 'put' | 'delete' | 'patch';
type MethodHandler = `handle${Capitalize<HTTPMethod>}`;
// Result: 'handleGet' | 'handlePost' | 'handlePut' | 'handleDelete' | 'handlePatch'

// 4. Getters and setters
type Property = 'name' | 'age' | 'email';
type Getter = `get${Capitalize<Property>}`;
type Setter = `set${Capitalize<Property>}`;

interface Accessors {
  getName(): string;
  setName(value: string): void;
  getAge(): number;
  setAge(value: number): void;
  // ...
}

// 5. Environment variables
type EnvVar = 'apiUrl' | 'dbHost' | 'logLevel';
type EnvKey = `VITE_${Uppercase<EnvVar>}`;
// Result: 'VITE_APIURL' | 'VITE_DBHOST' | 'VITE_LOGLEVEL'

// 6. Action type constants (Redux-style)
type Action = 'fetchUser' | 'updateUser' | 'deleteUser';
type ActionType = Uppercase<Action>;
// Result: 'FETCHUSER' | 'UPDATEUSER' | 'DELETEUSER'

// Better with template literals
type BetterActionType = `USER_${Uppercase<'fetch' | 'update' | 'delete'>}`;
// Result: 'USER_FETCH' | 'USER_UPDATE' | 'USER_DELETE'

// 7. Combining multiple transformations
type DatabaseTable = 'users' | 'posts' | 'comments';
type ModelName = Capitalize<DatabaseTable>;
type TableConstant = `TABLE_${Uppercase<DatabaseTable>}`;
// ModelName: 'Users' | 'Posts' | 'Comments'
// TableConstant: 'TABLE_USERS' | 'TABLE_POSTS' | 'TABLE_COMMENTS'
```

---

## 12. NoInfer\<T\> (TypeScript 5.4+)

Prevents TypeScript from inferring a type parameter from a specific location. This is useful when you want inference to happen from one argument but not another.

**Basic Usage:**
```typescript
// Without NoInfer - T is inferred from both arguments
function createState<T>(initial: T, defaultValue: T): T {
  return initial ?? defaultValue;
}

// T is inferred as string | number (union of both)
const state = createState('hello', 42);

// With NoInfer - T is only inferred from the first argument
function createStateBetter<T>(initial: T, defaultValue: NoInfer<T>): T {
  return initial ?? defaultValue;
}

// T is inferred as string, second argument must be string
const state2 = createStateBetter('hello', 42); // Error!
const state3 = createStateBetter('hello', 'default'); // OK
```

**Common Use Cases:**

```typescript
// 1. Default value patterns
function useState<T>(initialValue: T, options?: { default: NoInfer<T> }): T {
  return initialValue;
}

// T is inferred from initialValue only
useState(0, { default: 'not a number' }); // Error!
useState(0, { default: 100 }); // OK

// 2. Callback inference control
function process<T>(
  items: T[],
  transform: (item: NoInfer<T>) => NoInfer<T>
): T[] {
  return items.map(transform);
}

// T is inferred from items array only
process([1, 2, 3], (x) => x.toString()); // Error! T is number
process([1, 2, 3], (x) => x * 2); // OK

// 3. Event handlers with specific types
function on<T extends string>(
  event: T,
  handler: (data: NoInfer<EventData<T>>) => void
): void {
  // ...
}

type EventData<T> = T extends 'click' ? MouseEvent :
                    T extends 'keypress' ? KeyboardEvent :
                    Event;

// Event type is inferred from first argument
on('click', (data) => {
  // data is MouseEvent
  console.log(data.clientX);
});

// 4. Configuration objects
interface Config<T> {
  value: T;
  validate: (v: NoInfer<T>) => boolean;
  transform?: (v: NoInfer<T>) => NoInfer<T>;
}

function defineConfig<T>(config: Config<T>): Config<T> {
  return config;
}

defineConfig({
  value: 42,
  validate: (v) => typeof v === 'number', // v is number
  transform: (v) => v * 2, // v is number
});

// 5. Builder patterns
class QueryBuilder<T> {
  where(condition: (item: NoInfer<T>) => boolean): this {
    return this;
  }

  select<K extends keyof NoInfer<T>>(keys: K[]): Pick<T, K>[] {
    return [];
  }
}

// T should be set explicitly or from another source
interface User { id: string; name: string; age: number; }
const query = new QueryBuilder<User>();
query.where(user => user.age > 18); // user is User

// 6. Preventing widening in generics
function createPair<T>(first: T, second: NoInfer<T>): [T, T] {
  return [first, second];
}

createPair(1, 2);           // OK: [number, number]
createPair(1, 'two');       // Error! second must be number
createPair([1] as const, [2]); // OK: [readonly [1], readonly [1]]
```

---

## 13. Custom Utility Types

Learn how to create your own utility types for common patterns.

### Deep Readonly

Make all properties readonly recursively:

```typescript
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object
    ? T[P] extends Function
      ? T[P]
      : DeepReadonly<T[P]>
    : T[P];
};

interface NestedConfig {
  server: {
    host: string;
    port: number;
    ssl: {
      enabled: boolean;
      cert: string;
    };
  };
}

type ImmutableConfig = DeepReadonly<NestedConfig>;
// All nested properties are readonly
```

### Deep Partial

Make all properties optional recursively:

```typescript
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object
    ? T[P] extends Function
      ? T[P]
      : DeepPartial<T[P]>
    : T[P];
};

interface FormData {
  user: {
    name: string;
    address: {
      street: string;
      city: string;
    };
  };
}

type PartialFormData = DeepPartial<FormData>;
// Can provide any subset of nested properties
```

### Nullable

Make a type nullable:

```typescript
type Nullable<T> = T | null;

type NullableString = Nullable<string>;
// Result: string | null

// Make all properties nullable
type NullableProps<T> = {
  [P in keyof T]: Nullable<T[P]>;
};
```

### Mutable (Remove Readonly)

Remove readonly modifier from all properties:

```typescript
type Mutable<T> = {
  -readonly [P in keyof T]: T[P];
};

interface ReadonlyUser {
  readonly id: string;
  readonly name: string;
}

type MutableUser = Mutable<ReadonlyUser>;
// { id: string; name: string; } - no readonly
```

### Optional Keys

Get keys that are optional:

```typescript
type OptionalKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? K : never;
}[keyof T];

interface User {
  id: string;
  name: string;
  email?: string;
  phone?: string;
}

type UserOptionalKeys = OptionalKeys<User>;
// Result: 'email' | 'phone'
```

### Required Keys

Get keys that are required:

```typescript
type RequiredKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? never : K;
}[keyof T];

type UserRequiredKeys = RequiredKeys<User>;
// Result: 'id' | 'name'
```

### Keys Of Type

Get keys whose values are of a specific type:

```typescript
type KeysOfType<T, V> = {
  [K in keyof T]: T[K] extends V ? K : never;
}[keyof T];

interface User {
  id: string;
  name: string;
  age: number;
  isActive: boolean;
  score: number;
}

type StringKeys = KeysOfType<User, string>;
// Result: 'id' | 'name'

type NumberKeys = KeysOfType<User, number>;
// Result: 'age' | 'score'

type BooleanKeys = KeysOfType<User, boolean>;
// Result: 'isActive'
```

### PickByType

Pick properties of a specific type:

```typescript
type PickByType<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K];
};

type StringProps = PickByType<User, string>;
// Result: { id: string; name: string; }

type NumberProps = PickByType<User, number>;
// Result: { age: number; score: number; }
```

### OmitByType

Omit properties of a specific type:

```typescript
type OmitByType<T, V> = {
  [K in keyof T as T[K] extends V ? never : K]: T[K];
};

type NonStringProps = OmitByType<User, string>;
// Result: { age: number; isActive: boolean; score: number; }
```

### Prettify

Flatten intersection types for better readability:

```typescript
type Prettify<T> = {
  [K in keyof T]: T[K];
} & {};

type Combined = { a: string } & { b: number } & { c: boolean };
// Shows as: { a: string } & { b: number } & { c: boolean }

type PrettyCombined = Prettify<Combined>;
// Shows as: { a: string; b: number; c: boolean; }
```

### ValueOf

Get union of all value types:

```typescript
type ValueOf<T> = T[keyof T];

interface Config {
  host: string;
  port: number;
  debug: boolean;
}

type ConfigValue = ValueOf<Config>;
// Result: string | number | boolean
```

### Entries

Create tuple type for Object.entries():

```typescript
type Entries<T> = {
  [K in keyof T]: [K, T[K]];
}[keyof T];

interface User {
  name: string;
  age: number;
}

type UserEntries = Entries<User>;
// Result: ['name', string] | ['age', number]
```

### UnionToIntersection

Convert union type to intersection:

```typescript
type UnionToIntersection<U> =
  (U extends any ? (k: U) => void : never) extends ((k: infer I) => void)
    ? I
    : never;

type Union = { a: string } | { b: number } | { c: boolean };
type Intersection = UnionToIntersection<Union>;
// Result: { a: string } & { b: number } & { c: boolean }
```

---

## 14. Combining Utility Types

Real-world scenarios often require combining multiple utility types.

### API Response Types

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

// Create: omit system-managed fields
type CreateUserDto = Omit<User, 'id' | 'createdAt' | 'updatedAt' | 'deletedAt'>;

// Update: partial without id and system fields
type UpdateUserDto = Partial<Omit<User, 'id' | 'createdAt' | 'updatedAt' | 'deletedAt'>>;

// Response: omit sensitive and internal fields
type UserResponse = Omit<User, 'password' | 'deletedAt'>;

// List response: pick only essential fields
type UserListItem = Pick<User, 'id' | 'name' | 'email'>;
```

### Form State Management

```typescript
interface FormFields {
  username: string;
  email: string;
  password: string;
  confirmPassword: string;
  age: number;
  acceptTerms: boolean;
}

// Form values can be partial during editing
type FormValues = Partial<FormFields>;

// Errors are optional for each field
type FormErrors = Partial<Record<keyof FormFields, string>>;

// Track which fields have been touched
type FormTouched = Partial<Record<keyof FormFields, boolean>>;

// Validation result for a specific field
type FieldValidation<K extends keyof FormFields> = {
  field: K;
  value: FormFields[K];
  error: string | null;
};

// Complete form state
interface FormState {
  values: FormValues;
  errors: FormErrors;
  touched: FormTouched;
  isSubmitting: boolean;
  isValid: boolean;
}
```

### Redux/State Management

```typescript
// Base state shape
interface AppState {
  user: User | null;
  posts: Post[];
  comments: Record<string, Comment[]>;
  ui: {
    theme: 'light' | 'dark';
    sidebarOpen: boolean;
  };
}

// Immutable state
type ImmutableState = DeepReadonly<AppState>;

// Action creators return type
type Action<T extends string, P = undefined> = P extends undefined
  ? { type: T }
  : { type: T; payload: P };

// Extract state slice types
type UserState = NonNullable<AppState['user']>;
type UIState = AppState['ui'];

// Selector return types
type SelectUser = (state: AppState) => AppState['user'];
type SelectPosts = (state: AppState) => AppState['posts'];
```

### Component Props

```typescript
// Base HTML button attributes
type ButtonHTMLProps = React.ButtonHTMLAttributes<HTMLButtonElement>;

// Custom button props - replace onClick signature
type ButtonProps = Omit<ButtonHTMLProps, 'onClick' | 'disabled'> & {
  onClick: (event: React.MouseEvent<HTMLButtonElement>) => Promise<void> | void;
  loading?: boolean;
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
};

// Icon button - pick specific props and add icon
type IconButtonProps = Pick<ButtonProps, 'onClick' | 'loading' | 'size'> & {
  icon: React.ReactNode;
  'aria-label': string;
};

// Link-styled button - combine with anchor props
type LinkButtonProps = Omit<ButtonProps, 'onClick'> & {
  href: string;
  target?: '_blank' | '_self';
};

// Polymorphic component props
type PolymorphicProps<E extends React.ElementType, P = {}> = P &
  Omit<React.ComponentPropsWithoutRef<E>, keyof P> & {
    as?: E;
  };
```

### Database Entity Types

```typescript
// Base entity with timestamps
interface BaseEntity {
  id: string;
  createdAt: Date;
  updatedAt: Date;
}

// Full user entity
interface UserEntity extends BaseEntity {
  email: string;
  passwordHash: string;
  name: string;
  role: 'admin' | 'user';
}

// Input for creating (no auto-generated fields)
type CreateUserInput = Omit<UserEntity, keyof BaseEntity | 'passwordHash'> & {
  password: string;
};

// Input for updating (partial, exclude immutable fields)
type UpdateUserInput = Partial<Omit<UserEntity, keyof BaseEntity | 'email'>>;

// Safe output (no sensitive data)
type SafeUserOutput = Omit<UserEntity, 'passwordHash'>;

// Query filters
type UserFilters = Partial<Pick<UserEntity, 'email' | 'name' | 'role'>>;

// Sortable fields
type UserSortField = keyof Pick<UserEntity, 'name' | 'email' | 'createdAt'>;
```

### API Client Types

```typescript
// Base request config
interface RequestConfig {
  headers: Record<string, string>;
  timeout: number;
  retries: number;
  cache: boolean;
}

// Optional config - all fields optional
type RequestOptions = Partial<RequestConfig>;

// Response wrapper
type ApiResponse<T> = {
  data: T;
  status: number;
  headers: Record<string, string>;
};

// Error response
type ApiError = {
  code: string;
  message: string;
  details?: Record<string, unknown>;
};

// Combined result type
type ApiResult<T> =
  | { success: true; response: ApiResponse<T> }
  | { success: false; error: ApiError };

// Typed fetch function
type TypedFetch = <T>(
  url: string,
  options?: RequestOptions
) => Promise<ApiResult<T>>;

// Extract response data type
type ResponseData<T extends (...args: any) => Promise<ApiResult<any>>> =
  Awaited<ReturnType<T>> extends { success: true; response: ApiResponse<infer D> }
    ? D
    : never;
```

---

## 15. Real-world Patterns and Examples

### Pattern 1: Type-Safe Event Emitter

```typescript
type EventMap = {
  'user:login': { userId: string; timestamp: Date };
  'user:logout': { userId: string };
  'data:update': { entity: string; id: string; changes: Record<string, unknown> };
  'error': { code: string; message: string };
};

type EventName = keyof EventMap;
type EventPayload<E extends EventName> = EventMap[E];
type EventHandler<E extends EventName> = (payload: EventPayload<E>) => void;

class TypedEventEmitter {
  private handlers: {
    [E in EventName]?: EventHandler<E>[]
  } = {};

  on<E extends EventName>(event: E, handler: EventHandler<E>): void {
    if (!this.handlers[event]) {
      this.handlers[event] = [];
    }
    this.handlers[event]!.push(handler);
  }

  emit<E extends EventName>(event: E, payload: EventPayload<E>): void {
    this.handlers[event]?.forEach(handler => handler(payload));
  }

  off<E extends EventName>(event: E, handler: EventHandler<E>): void {
    const handlers = this.handlers[event];
    if (handlers) {
      const index = handlers.indexOf(handler);
      if (index > -1) handlers.splice(index, 1);
    }
  }
}

// Usage
const emitter = new TypedEventEmitter();

emitter.on('user:login', (payload) => {
  // payload is { userId: string; timestamp: Date }
  console.log(`User ${payload.userId} logged in at ${payload.timestamp}`);
});

emitter.emit('user:login', { userId: '123', timestamp: new Date() });
```

### Pattern 2: Type-Safe Builder Pattern

```typescript
interface QueryConfig {
  table: string;
  select: string[];
  where: Record<string, unknown>;
  orderBy: { field: string; direction: 'asc' | 'desc' };
  limit: number;
  offset: number;
}

type QueryBuilderState<T extends Partial<QueryConfig>> = T;

class QueryBuilder<State extends Partial<QueryConfig> = {}> {
  constructor(private state: State = {} as State) {}

  from<T extends string>(
    table: T
  ): QueryBuilder<State & { table: T }> {
    return new QueryBuilder({ ...this.state, table });
  }

  select<T extends string[]>(
    ...fields: T
  ): QueryBuilder<State & { select: T }> {
    return new QueryBuilder({ ...this.state, select: fields });
  }

  where<T extends Record<string, unknown>>(
    conditions: T
  ): QueryBuilder<State & { where: T }> {
    return new QueryBuilder({ ...this.state, where: conditions });
  }

  orderBy<F extends string, D extends 'asc' | 'desc'>(
    field: F,
    direction: D
  ): QueryBuilder<State & { orderBy: { field: F; direction: D } }> {
    return new QueryBuilder({ ...this.state, orderBy: { field, direction } });
  }

  limit(n: number): QueryBuilder<State & { limit: number }> {
    return new QueryBuilder({ ...this.state, limit: n });
  }

  // Only allow execute if table and select are set
  execute(
    this: QueryBuilder<State & { table: string; select: string[] }>
  ): Promise<unknown[]> {
    console.log('Executing query:', this.state);
    return Promise.resolve([]);
  }
}

// Usage
const query = new QueryBuilder()
  .from('users')
  .select('id', 'name', 'email')
  .where({ active: true })
  .orderBy('name', 'asc')
  .limit(10);

query.execute(); // OK - has table and select

// This would error:
// new QueryBuilder().execute(); // Error - missing table and select
```

### Pattern 3: Type-Safe Form Validation

```typescript
type ValidationRule<T> = {
  validate: (value: T) => boolean;
  message: string;
};

type FormSchema<T extends Record<string, unknown>> = {
  [K in keyof T]: {
    required?: boolean;
    rules?: ValidationRule<T[K]>[];
  };
};

type FormErrors<T extends Record<string, unknown>> = {
  [K in keyof T]?: string[];
};

function validateForm<T extends Record<string, unknown>>(
  data: Partial<T>,
  schema: FormSchema<T>
): { valid: boolean; errors: FormErrors<T> } {
  const errors: FormErrors<T> = {};
  let valid = true;

  for (const key in schema) {
    const field = schema[key];
    const value = data[key];
    const fieldErrors: string[] = [];

    if (field.required && (value === undefined || value === '')) {
      fieldErrors.push(`${key} is required`);
      valid = false;
    }

    if (value !== undefined && field.rules) {
      for (const rule of field.rules) {
        if (!rule.validate(value as T[typeof key])) {
          fieldErrors.push(rule.message);
          valid = false;
        }
      }
    }

    if (fieldErrors.length > 0) {
      errors[key] = fieldErrors;
    }
  }

  return { valid, errors };
}

// Usage
interface RegistrationForm {
  email: string;
  password: string;
  age: number;
}

const schema: FormSchema<RegistrationForm> = {
  email: {
    required: true,
    rules: [
      {
        validate: (v) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v),
        message: 'Invalid email format',
      },
    ],
  },
  password: {
    required: true,
    rules: [
      {
        validate: (v) => v.length >= 8,
        message: 'Password must be at least 8 characters',
      },
    ],
  },
  age: {
    required: true,
    rules: [
      {
        validate: (v) => v >= 18,
        message: 'Must be at least 18 years old',
      },
    ],
  },
};

const result = validateForm(
  { email: 'test@example.com', password: '12345', age: 16 },
  schema
);
// { valid: false, errors: { password: [...], age: [...] } }
```

### Pattern 4: Type-Safe API Routes

```typescript
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';

interface RouteDefinition<
  Params extends Record<string, string> = {},
  Query extends Record<string, string> = {},
  Body = unknown,
  Response = unknown
> {
  params: Params;
  query: Query;
  body: Body;
  response: Response;
}

interface ApiRoutes {
  'GET /users': RouteDefinition<{}, { page?: string; limit?: string }, never, User[]>;
  'GET /users/:id': RouteDefinition<{ id: string }, {}, never, User>;
  'POST /users': RouteDefinition<{}, {}, CreateUserDto, User>;
  'PUT /users/:id': RouteDefinition<{ id: string }, {}, UpdateUserDto, User>;
  'DELETE /users/:id': RouteDefinition<{ id: string }, {}, never, { success: boolean }>;
}

type RouteKey = keyof ApiRoutes;

type ExtractMethod<R extends RouteKey> = R extends `${infer M} ${string}` ? M : never;
type ExtractPath<R extends RouteKey> = R extends `${string} ${infer P}` ? P : never;

type RouteParams<R extends RouteKey> = ApiRoutes[R]['params'];
type RouteQuery<R extends RouteKey> = ApiRoutes[R]['query'];
type RouteBody<R extends RouteKey> = ApiRoutes[R]['body'];
type RouteResponse<R extends RouteKey> = ApiRoutes[R]['response'];

// Type-safe fetch wrapper
async function apiRequest<R extends RouteKey>(
  route: R,
  options: {
    params?: RouteParams<R>;
    query?: RouteQuery<R>;
    body?: RouteBody<R>;
  }
): Promise<RouteResponse<R>> {
  // Implementation
  throw new Error('Not implemented');
}

// Usage
const users = await apiRequest('GET /users', {
  query: { page: '1', limit: '10' },
});
// users is User[]

const user = await apiRequest('GET /users/:id', {
  params: { id: '123' },
});
// user is User

const created = await apiRequest('POST /users', {
  body: { email: 'test@example.com', name: 'Test', password: 'secret' },
});
// created is User
```

### Pattern 5: Type-Safe State Machine

```typescript
type StateMachineConfig<
  States extends string,
  Events extends string
> = {
  initial: States;
  states: {
    [S in States]: {
      on?: {
        [E in Events]?: States;
      };
    };
  };
};

type InferStates<T> = T extends StateMachineConfig<infer S, any> ? S : never;
type InferEvents<T> = T extends StateMachineConfig<any, infer E> ? E : never;

function createStateMachine<
  Config extends StateMachineConfig<string, string>
>(config: Config) {
  type States = InferStates<Config>;
  type Events = InferEvents<Config>;

  let currentState = config.initial as States;

  return {
    getState(): States {
      return currentState;
    },

    transition(event: Events): States {
      const stateConfig = config.states[currentState];
      const nextState = stateConfig.on?.[event];

      if (nextState) {
        currentState = nextState as States;
      }

      return currentState;
    },

    canTransition(event: Events): boolean {
      const stateConfig = config.states[currentState];
      return event in (stateConfig.on ?? {});
    },
  };
}

// Usage
const orderMachine = createStateMachine({
  initial: 'pending',
  states: {
    pending: {
      on: {
        CONFIRM: 'confirmed',
        CANCEL: 'cancelled',
      },
    },
    confirmed: {
      on: {
        SHIP: 'shipped',
        CANCEL: 'cancelled',
      },
    },
    shipped: {
      on: {
        DELIVER: 'delivered',
      },
    },
    delivered: {},
    cancelled: {},
  },
} as const);

orderMachine.transition('CONFIRM'); // 'confirmed'
orderMachine.transition('SHIP');    // 'shipped'
orderMachine.transition('DELIVER'); // 'delivered'
```

---

## 16. Best Practices

### 1. Prefer Built-in Utility Types

Use TypeScript's built-in utility types when possible. They are well-tested, optimized, and widely understood.

```typescript
// Good - uses built-in utilities
type UpdateUser = Partial<Omit<User, 'id'>>;

// Avoid - custom implementation of the same thing
type UpdateUser = {
  [K in keyof User as K extends 'id' ? never : K]?: User[K];
};
```

### 2. Name Derived Types Meaningfully

Give derived types meaningful names that describe their purpose.

```typescript
// Good - clear intent
type CreateUserDto = Omit<User, 'id' | 'createdAt'>;
type UserResponse = Omit<User, 'password'>;
type UserListItem = Pick<User, 'id' | 'name'>;

// Avoid - unclear purpose
type User1 = Omit<User, 'id' | 'createdAt'>;
type User2 = Omit<User, 'password'>;
type User3 = Pick<User, 'id' | 'name'>;
```

### 3. Use Type Aliases for Complex Combinations

When combining multiple utility types, create intermediate aliases for clarity.

```typescript
// Good - clear steps
type UserWithoutSystemFields = Omit<User, 'id' | 'createdAt' | 'updatedAt'>;
type UpdateUserDto = Partial<UserWithoutSystemFields>;

// Harder to read
type UpdateUserDto = Partial<Omit<User, 'id' | 'createdAt' | 'updatedAt'>>;
```

### 4. Avoid Over-Engineering

Don't create utility types for one-off uses. Simple inline types are often clearer.

```typescript
// Good for one-off use
function process(data: { name: string; value: number }): void { }

// Over-engineered for single use
type ProcessData = Pick<SomeInterface, 'name' | 'value'>;
function process(data: ProcessData): void { }
```

### 5. Document Complex Utility Types

Add JSDoc comments to explain complex custom utility types.

```typescript
/**
 * Makes all properties of T deeply readonly, including nested objects.
 * Functions are preserved as-is.
 *
 * @example
 * type Config = DeepReadonly<{ server: { port: number } }>;
 * // All nested properties are readonly
 */
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object
    ? T[P] extends Function
      ? T[P]
      : DeepReadonly<T[P]>
    : T[P];
};
```

### 6. Consider Readability in IDE

Use `Prettify` for complex intersections to improve IDE hover information.

```typescript
// IDE shows: { a: string } & { b: number } & { c: boolean }
type Ugly = { a: string } & { b: number } & { c: boolean };

// IDE shows: { a: string; b: number; c: boolean }
type Pretty = Prettify<{ a: string } & { b: number } & { c: boolean }>;
```

### 7. Test Utility Types

For custom utility types, write type-level tests to ensure correctness.

```typescript
// Type-level testing utilities
type Expect<T extends true> = T;
type Equal<X, Y> = (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2)
  ? true
  : false;

// Tests for custom utility type
type TestKeysOfType = Expect<Equal<
  KeysOfType<{ a: string; b: number; c: string }, string>,
  'a' | 'c'
>>;

type TestPickByType = Expect<Equal<
  PickByType<{ a: string; b: number }, string>,
  { a: string }
>>;
```

### 8. Avoid Excessive Nesting

Deeply nested utility types become hard to understand and debug.

```typescript
// Hard to understand
type Complex = Partial<Required<Omit<Pick<User, keyof BaseEntity>, 'id'>>>;

// Better - break it down
type UserBaseFields = Pick<User, keyof BaseEntity>;
type UserFieldsWithoutId = Omit<UserBaseFields, 'id'>;
type RequiredUserFields = Required<UserFieldsWithoutId>;
type PartialRequiredUserFields = Partial<RequiredUserFields>;
```

### 9. Use Conditional Types Sparingly

Conditional types are powerful but can be hard to debug. Prefer simpler alternatives when possible.

```typescript
// Simple union manipulation - good
type NonString = Exclude<string | number | boolean, string>;

// Complex conditional - use when necessary
type UnwrapPromise<T> = T extends Promise<infer U>
  ? U extends Promise<any>
    ? UnwrapPromise<U>
    : U
  : T;
```

### 10. Keep Utility Types Focused

Each utility type should do one thing well. Combine them for complex transformations.

```typescript
// Good - focused utilities
type Nullable<T> = T | null;
type Optional<T> = T | undefined;
type Mutable<T> = { -readonly [P in keyof T]: T[P] };

// Combine as needed
type MutableNullableUser = Mutable<Nullable<User>>;

// Avoid - utility that does too much
type ComplexTransform<T> = {
  -readonly [P in keyof T]?: T[P] | null | undefined;
}; // Does mutable + optional + nullable
```

---

## Quick Reference

| Utility Type | Purpose | Example |
|-------------|---------|---------|
| `Partial<T>` | All props optional | `Partial<User>` |
| `Required<T>` | All props required | `Required<Config>` |
| `Readonly<T>` | All props readonly | `Readonly<State>` |
| `Pick<T, K>` | Select specific props | `Pick<User, 'id' \| 'name'>` |
| `Omit<T, K>` | Remove specific props | `Omit<User, 'password'>` |
| `Record<K, V>` | Object with K keys and V values | `Record<string, number>` |
| `Exclude<T, U>` | Remove types from union | `Exclude<Status, 'cancelled'>` |
| `Extract<T, U>` | Keep matching types | `Extract<AllTypes, Function>` |
| `NonNullable<T>` | Remove null/undefined | `NonNullable<string \| null>` |
| `Parameters<T>` | Function param types | `Parameters<typeof fn>` |
| `ReturnType<T>` | Function return type | `ReturnType<typeof fn>` |
| `ConstructorParameters<T>` | Constructor param types | `ConstructorParameters<typeof Class>` |
| `InstanceType<T>` | Instance type of class | `InstanceType<typeof Class>` |
| `Awaited<T>` | Unwrap Promise type | `Awaited<Promise<string>>` |
| `Uppercase<S>` | String to uppercase | `Uppercase<'hello'>` |
| `Lowercase<S>` | String to lowercase | `Lowercase<'HELLO'>` |
| `Capitalize<S>` | Capitalize first char | `Capitalize<'hello'>` |
| `Uncapitalize<S>` | Lowercase first char | `Uncapitalize<'Hello'>` |
| `NoInfer<T>` | Prevent type inference | `NoInfer<T>` |

---

## Further Reading

- [TypeScript Handbook - Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)
- [TypeScript Handbook - Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html)
- [TypeScript Handbook - Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html)
- [TypeScript Handbook - Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)
