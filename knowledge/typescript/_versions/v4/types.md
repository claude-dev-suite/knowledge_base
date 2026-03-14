# TypeScript 4 → Types Delta

## Not Available in TypeScript 4

- `satisfies` operator (TS 5.0+)
- `const` type parameters (TS 5.0+)
- `export type *` syntax (TS 5.0+)
- `--moduleResolution bundler` (TS 5.0+)
- All enums as union enums (TS 5.0+)

## Syntax Differences

### satisfies Operator

```typescript
// TypeScript 5 - satisfies for type checking without widening
type Colors = 'red' | 'green' | 'blue'
type RGB = [number, number, number]

const palette = {
  red: [255, 0, 0],
  green: '#00ff00',
  blue: [0, 0, 255]
} satisfies Record<Colors, RGB | string>

// Type is preserved: palette.green is string, palette.red is [number, number, number]
palette.green.toUpperCase()  // OK - knows it's a string
palette.red.map(x => x)      // OK - knows it's a tuple

// TypeScript 4 - Use type annotation (loses specific types)
const palette: Record<Colors, RGB | string> = {
  red: [255, 0, 0],
  green: '#00ff00',
  blue: [0, 0, 255]
}

// Type is widened
palette.green.toUpperCase()  // Error - could be RGB | string
```

### const Type Parameters

```typescript
// TypeScript 5 - const type parameter
function createPair<const T extends readonly unknown[]>(arr: T): T {
  return arr
}

const pair = createPair(['a', 1] as const)
// Type: readonly ["a", 1]

// Without const:
function createPairNoConst<T extends readonly unknown[]>(arr: T): T {
  return arr
}
const pairWide = createPairNoConst(['a', 1])
// Type: readonly (string | number)[]


// TypeScript 4 - Use 'as const' at call site
const pair = createPairNoConst(['a', 1] as const)
```

### Decorators (Stage 3)

```typescript
// TypeScript 5 - Standard decorators (Stage 3)
function log<This, Args extends any[], Return>(
  target: (this: This, ...args: Args) => Return,
  context: ClassMethodDecoratorContext
) {
  return function(this: This, ...args: Args): Return {
    console.log(`Calling ${String(context.name)}`)
    return target.call(this, ...args)
  }
}

class Calculator {
  @log
  add(a: number, b: number) {
    return a + b
  }
}

// TypeScript 4 - Experimental decorators (different API)
// tsconfig.json: "experimentalDecorators": true
function log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value
  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${propertyKey}`)
    return original.apply(this, args)
  }
}
```

### Module Resolution: bundler

```json
// TypeScript 5 - tsconfig.json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "noEmit": true
  }
}

// TypeScript 4 - Use "node" or "node16"
{
  "compilerOptions": {
    "moduleResolution": "node16"
  }
}
```

### Export Type Star

```typescript
// TypeScript 5
export type * from './types'
export type * as Types from './types'

// TypeScript 4 - Must export explicitly
export type { TypeA, TypeB, TypeC } from './types'
// Or re-export with namespace
import * as Types from './types'
export { Types }
```

## Still Current in TypeScript 4

- Basic types (string, number, boolean, etc.)
- Interfaces and type aliases
- Generics
- Union and intersection types
- Type guards
- Mapped types
- Conditional types
- Template literal types (4.1+)
- Variadic tuple types (4.0+)

## Recommendations for TypeScript 4 Users

1. **Use `as const`** - For literal type inference
2. **Explicit type annotations** - Where `satisfies` would help
3. **Experimental decorators** - With flag, different API
4. **node16 resolution** - For ESM projects
5. **Plan upgrade to TS 5** - For better DX features

