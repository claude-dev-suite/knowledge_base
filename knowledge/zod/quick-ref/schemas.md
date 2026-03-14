# Zod Schema Quick Reference

## Basic Types

```typescript
import { z } from 'zod';

// Primitives
const string = z.string();
const number = z.number();
const boolean = z.boolean();
const date = z.date();
const bigint = z.bigint();
const symbol = z.symbol();
const undefined = z.undefined();
const null = z.null();
const void = z.void();
const any = z.any();
const unknown = z.unknown();
const never = z.never();
```

## String Validations

```typescript
z.string()
  .min(5, "Too short")
  .max(100, "Too long")
  .email("Invalid email")
  .url("Invalid URL")
  .uuid("Invalid UUID")
  .regex(/^\d+$/, "Only digits")
  .trim()
  .toLowerCase()
  .toUpperCase()
  .startsWith("https://")
  .endsWith(".com")
  .includes("@")
  .datetime() // ISO datetime
  .ip() // IP address
```

## Number Validations

```typescript
z.number()
  .int("Must be integer")
  .positive("Must be positive")
  .negative("Must be negative")
  .min(0, "Min 0")
  .max(100, "Max 100")
  .multipleOf(5, "Must be multiple of 5")
  .finite()
  .safe() // Within JS safe integer range
```

## Objects

```typescript
const userSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(),
  age: z.number().optional(),
  active: z.boolean().default(true)
});

type User = z.infer<typeof userSchema>;
```

## Arrays

```typescript
z.array(z.string())
  .min(1, "At least 1")
  .max(5, "Max 5")
  .nonempty("Cannot be empty")
  .length(3, "Exactly 3");

// Non-empty array
z.string().array().nonempty();
```

## Enums

```typescript
const roleSchema = z.enum(['admin', 'user', 'guest']);

// Native enum
enum Status {
  Active = 'ACTIVE',
  Inactive = 'INACTIVE'
}
const statusSchema = z.nativeEnum(Status);
```

## Union & Intersection

```typescript
// Union
const idSchema = z.union([z.string(), z.number()]);
// or
const idSchema = z.string().or(z.number());

// Discriminated union
const eventSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('click'), x: z.number() }),
  z.object({ type: z.literal('keypress'), key: z.string() })
]);

// Intersection
const withTimestamp = z.object({ createdAt: z.date() });
const userWithTimestamp = userSchema.and(withTimestamp);
```

## Optional & Nullable

```typescript
z.string().optional()     // string | undefined
z.string().nullable()     // string | null
z.string().nullish()      // string | null | undefined
```

## Default Values

```typescript
z.string().default('default')
z.number().default(0)
z.boolean().default(true)
z.date().default(() => new Date())
```

## Transform

```typescript
z.string()
  .transform(val => val.trim())
  .transform(val => val.toLowerCase());

z.string()
  .transform(val => parseInt(val, 10));

z.coerce.number()  // Coerce to number
z.coerce.date()    // Coerce to date
```

## Refine (Custom Validation)

```typescript
z.string().refine(val => val.length > 0, {
  message: "Cannot be empty"
});

z.object({
  password: z.string(),
  confirm: z.string()
}).refine(data => data.password === data.confirm, {
  message: "Passwords don't match",
  path: ['confirm']
});
```

## Parsing

```typescript
// Throws on error
const user = userSchema.parse(data);

// Safe parse (no throw)
const result = userSchema.safeParse(data);
if (result.success) {
  console.log(result.data);
} else {
  console.log(result.error);
}

// Async
await userSchema.parseAsync(data);
```

## React Hook Form

```typescript
import { zodResolver } from '@hookform/resolvers/zod';

const schema = z.object({
  name: z.string().min(2),
  email: z.string().email()
});

const { register, handleSubmit } = useForm({
  resolver: zodResolver(schema)
});
```

## Partial & Pick

```typescript
const userSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string()
});

// All optional
userSchema.partial();

// Pick fields
userSchema.pick({ name: true, email: true });

// Omit fields
userSchema.omit({ id: true });

// Extend
userSchema.extend({
  phone: z.string()
});

// Merge
const merged = schema1.merge(schema2);
```

## Error Handling

```typescript
try {
  schema.parse(data);
} catch (error) {
  if (error instanceof z.ZodError) {
    console.log(error.issues);
    console.log(error.flatten());
  }
}
```

## Common Patterns

```typescript
// Email
z.string().email().toLowerCase()

// Password
z.string()
  .min(8)
  .regex(/[A-Z]/, "Need uppercase")
  .regex(/[0-9]/, "Need number")

// Phone
z.string().regex(/^\+?[1-9]\d{1,14}$/)

// URL
z.string().url().startsWith("https://")

// Date range
z.object({
  start: z.date(),
  end: z.date()
}).refine(data => data.end > data.start, {
  message: "End must be after start"
})
```
