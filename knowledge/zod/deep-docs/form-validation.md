# Zod Form Validation Patterns

## React Hook Form Integration

### Complete Form Example

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Schema
const createUserSchema = z.object({
  name: z.string()
    .min(2, "Name must be at least 2 characters")
    .max(50, "Name must be at most 50 characters"),
  email: z.string()
    .email("Invalid email address")
    .toLowerCase(),
  password: z.string()
    .min(8, "Password must be at least 8 characters")
    .regex(/[A-Z]/, "Password must contain uppercase letter")
    .regex(/[a-z]/, "Password must contain lowercase letter")
    .regex(/[0-9]/, "Password must contain number"),
  confirmPassword: z.string(),
  age: z.coerce.number()
    .int("Must be integer")
    .min(18, "Must be at least 18")
    .max(100, "Must be at most 100"),
  role: z.enum(['admin', 'user', 'guest']),
  terms: z.boolean().refine(val => val === true, {
    message: "You must accept the terms"
  })
}).refine(data => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ['confirmPassword']
});

type CreateUserForm = z.infer<typeof createUserSchema>;

// Component
export function CreateUserForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting }
  } = useForm<CreateUserForm>({
    resolver: zodResolver(createUserSchema),
    defaultValues: {
      role: 'user'
    }
  });

  const onSubmit = async (data: CreateUserForm) => {
    try {
      await createUser(data);
    } catch (error) {
      console.error(error);
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label htmlFor="name">Name</label>
        <input {...register('name')} />
        {errors.name && <span>{errors.name.message}</span>}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input {...register('email')} type="email" />
        {errors.email && <span>{errors.email.message}</span>}
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input {...register('password')} type="password" />
        {errors.password && <span>{errors.password.message}</span>}
      </div>

      <div>
        <label htmlFor="confirmPassword">Confirm Password</label>
        <input {...register('confirmPassword')} type="password" />
        {errors.confirmPassword && <span>{errors.confirmPassword.message}</span>}
      </div>

      <div>
        <label htmlFor="age">Age</label>
        <input {...register('age')} type="number" />
        {errors.age && <span>{errors.age.message}</span>}
      </div>

      <div>
        <label htmlFor="role">Role</label>
        <select {...register('role')}>
          <option value="user">User</option>
          <option value="admin">Admin</option>
          <option value="guest">Guest</option>
        </select>
        {errors.role && <span>{errors.role.message}</span>}
      </div>

      <div>
        <input {...register('terms')} type="checkbox" />
        <label htmlFor="terms">Accept Terms</label>
        {errors.terms && <span>{errors.terms.message}</span>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

## Async Validation

### Email Uniqueness Check

```typescript
const emailSchema = z.string()
  .email("Invalid email")
  .refine(
    async (email) => {
      const response = await fetch(`/api/check-email?email=${email}`);
      const { exists } = await response.json();
      return !exists;
    },
    { message: "Email already exists" }
  );

// Usage with React Hook Form
const schema = z.object({
  email: emailSchema
});

const { register, handleSubmit } = useForm({
  resolver: zodResolver(schema, {
    async: true // Enable async validation
  })
});
```

## Multi-Step Forms

### Step-by-Step Validation

```typescript
// Step 1: Personal Info
const step1Schema = z.object({
  firstName: z.string().min(2),
  lastName: z.string().min(2),
  email: z.string().email()
});

// Step 2: Address
const step2Schema = z.object({
  street: z.string().min(5),
  city: z.string().min(2),
  zipCode: z.string().regex(/^\d{5}$/)
});

// Step 3: Preferences
const step3Schema = z.object({
  newsletter: z.boolean(),
  notifications: z.boolean()
});

// Complete form schema
const completeSchema = step1Schema
  .merge(step2Schema)
  .merge(step3Schema);

type CompleteForm = z.infer<typeof completeSchema>;

// Multi-step component
export function MultiStepForm() {
  const [step, setStep] = useState(1);
  const [formData, setFormData] = useState<Partial<CompleteForm>>({});

  const step1Form = useForm({
    resolver: zodResolver(step1Schema)
  });

  const step2Form = useForm({
    resolver: zodResolver(step2Schema)
  });

  const step3Form = useForm({
    resolver: zodResolver(step3Schema)
  });

  const handleStep1Submit = (data: z.infer<typeof step1Schema>) => {
    setFormData(prev => ({ ...prev, ...data }));
    setStep(2);
  };

  const handleStep2Submit = (data: z.infer<typeof step2Schema>) => {
    setFormData(prev => ({ ...prev, ...data }));
    setStep(3);
  };

  const handleStep3Submit = async (data: z.infer<typeof step3Schema>) => {
    const complete = { ...formData, ...data };
    // Validate complete form
    const validated = completeSchema.parse(complete);
    await submitForm(validated);
  };

  return (
    <div>
      {step === 1 && (
        <form onSubmit={step1Form.handleSubmit(handleStep1Submit)}>
          {/* Step 1 fields */}
        </form>
      )}
      {step === 2 && (
        <form onSubmit={step2Form.handleSubmit(handleStep2Submit)}>
          {/* Step 2 fields */}
        </form>
      )}
      {step === 3 && (
        <form onSubmit={step3Form.handleSubmit(handleStep3Submit)}>
          {/* Step 3 fields */}
        </form>
      )}
    </div>
  );
}
```

## Dynamic Fields

### Array Fields Validation

```typescript
const itemSchema = z.object({
  name: z.string().min(1),
  quantity: z.number().int().min(1),
  price: z.number().positive()
});

const orderSchema = z.object({
  customerName: z.string(),
  items: z.array(itemSchema)
    .min(1, "At least one item required")
    .max(10, "Maximum 10 items"),
  total: z.number()
}).refine(
  data => {
    const calculatedTotal = data.items.reduce(
      (sum, item) => sum + (item.quantity * item.price),
      0
    );
    return Math.abs(calculatedTotal - data.total) < 0.01;
  },
  { message: "Total doesn't match items", path: ['total'] }
);

// Component with useFieldArray
export function OrderForm() {
  const { register, control, handleSubmit } = useForm({
    resolver: zodResolver(orderSchema),
    defaultValues: {
      items: [{ name: '', quantity: 1, price: 0 }]
    }
  });

  const { fields, append, remove } = useFieldArray({
    control,
    name: 'items'
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {fields.map((field, index) => (
        <div key={field.id}>
          <input {...register(`items.${index}.name`)} />
          <input {...register(`items.${index}.quantity`)} type="number" />
          <input {...register(`items.${index}.price`)} type="number" />
          <button type="button" onClick={() => remove(index)}>
            Remove
          </button>
        </div>
      ))}
      <button type="button" onClick={() => append({ name: '', quantity: 1, price: 0 })}>
        Add Item
      </button>
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Conditional Validation

### Dependent Fields

```typescript
const shippingSchema = z.object({
  shippingMethod: z.enum(['pickup', 'delivery']),
  address: z.string().optional(),
  city: z.string().optional(),
  zipCode: z.string().optional()
}).superRefine((data, ctx) => {
  if (data.shippingMethod === 'delivery') {
    if (!data.address || data.address.length < 5) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: "Address is required for delivery",
        path: ['address']
      });
    }
    if (!data.city) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: "City is required for delivery",
        path: ['city']
      });
    }
    if (!data.zipCode || !/^\d{5}$/.test(data.zipCode)) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: "Valid zip code is required for delivery",
        path: ['zipCode']
      });
    }
  }
});
```

## File Upload Validation

```typescript
const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
const ACCEPTED_IMAGE_TYPES = ['image/jpeg', 'image/png', 'image/webp'];

const fileSchema = z.object({
  avatar: z.custom<FileList>()
    .refine(files => files?.length === 1, "Avatar is required")
    .refine(
      files => files?.[0]?.size <= MAX_FILE_SIZE,
      `Max file size is 5MB`
    )
    .refine(
      files => ACCEPTED_IMAGE_TYPES.includes(files?.[0]?.type),
      "Only .jpg, .png and .webp formats are supported"
    )
    .transform(files => files[0])
});

// Component
const { register, handleSubmit } = useForm({
  resolver: zodResolver(fileSchema)
});

<input {...register('avatar')} type="file" accept="image/*" />
```

## Error Messages Customization

```typescript
const customErrorMap: z.ZodErrorMap = (issue, ctx) => {
  if (issue.code === z.ZodIssueCode.invalid_type) {
    if (issue.expected === 'string') {
      return { message: 'This field must be text' };
    }
  }
  if (issue.code === z.ZodIssueCode.too_small) {
    if (issue.type === 'string') {
      return { message: `Minimum ${issue.minimum} characters required` };
    }
  }
  return { message: ctx.defaultError };
};

z.setErrorMap(customErrorMap);

// Or per-schema
const schema = z.object({
  name: z.string()
}, { errorMap: customErrorMap });
```

## Best Practices

1. ✅ Define schemas at module level, not in components
2. ✅ Use `z.infer` to derive TypeScript types
3. ✅ Provide clear, user-friendly error messages
4. ✅ Use `.transform()` for data normalization (trim, lowercase)
5. ✅ Use `.refine()` for cross-field validation
6. ✅ Use `.superRefine()` for complex conditional logic
7. ✅ Enable async validation when checking uniqueness
8. ✅ Use `.partial()` for update forms
9. ✅ Validate file uploads (size, type)
10. ✅ Test edge cases (empty, null, max length)

## Common Patterns

```typescript
// Update form (partial schema)
const updateUserSchema = createUserSchema.partial();

// Required field in update
const updateWithRequired = updateUserSchema.required({ email: true });

// Transform on submit
const schema = z.object({
  email: z.string().email().transform(v => v.toLowerCase()),
  name: z.string().transform(v => v.trim())
});

// Optional with default
const schema = z.object({
  role: z.enum(['user', 'admin']).default('user')
});
```

## References

- [Zod Documentation](https://zod.dev/)
- [React Hook Form](https://react-hook-form.com/)
- [Form Validation Best Practices](https://web.dev/sign-in-form-best-practices/)
