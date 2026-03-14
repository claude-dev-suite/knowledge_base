# React Hook Form - Comprehensive Documentation

React Hook Form is a performant, flexible, and extensible forms library for React with easy-to-use validation.

## Table of Contents

1. [Setup and Installation](#setup-and-installation)
2. [useForm Hook](#useform-hook)
3. [register Function](#register-function)
4. [handleSubmit](#handlesubmit)
5. [Form State](#form-state)
6. [Validation](#validation)
7. [Error Messages](#error-messages)
8. [Controller Component](#controller-component)
9. [useWatch Hook](#usewatch-hook)
10. [useFieldArray Hook](#usefieldarray-hook)
11. [setValue and getValues](#setvalue-and-getvalues)
12. [reset and trigger](#reset-and-trigger)
13. [Integration with UI Libraries](#integration-with-ui-libraries)
14. [Integration with TanStack Query](#integration-with-tanstack-query)
15. [File Uploads](#file-uploads)
16. [TypeScript Integration](#typescript-integration)
17. [Performance Optimization](#performance-optimization)
18. [Best Practices](#best-practices)

---

## Setup and Installation

### Basic Installation

```bash
# npm
npm install react-hook-form

# yarn
yarn add react-hook-form

# pnpm
pnpm add react-hook-form
```

### With Validation Libraries

```bash
# Zod (recommended)
npm install zod @hookform/resolvers

# Yup
npm install yup @hookform/resolvers

# Joi
npm install joi @hookform/resolvers

# Superstruct
npm install superstruct @hookform/resolvers
```

### Quick Start Example

```tsx
import { useForm, SubmitHandler } from 'react-hook-form';

type FormInputs = {
  firstName: string;
  lastName: string;
  email: string;
};

function BasicForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormInputs>();

  const onSubmit: SubmitHandler<FormInputs> = (data) => {
    console.log(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('firstName', { required: true })} />
      {errors.firstName && <span>First name is required</span>}

      <input {...register('lastName', { required: true })} />
      {errors.lastName && <span>Last name is required</span>}

      <input {...register('email', { required: true, pattern: /^\S+@\S+$/i })} />
      {errors.email && <span>Valid email is required</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## useForm Hook

The `useForm` hook is the core of React Hook Form. It manages form state and provides methods for form handling.

### All Configuration Options

```tsx
import { useForm } from 'react-hook-form';

const form = useForm({
  // Mode determines when validation triggers
  mode: 'onSubmit', // 'onBlur' | 'onChange' | 'onTouched' | 'all' | 'onSubmit'

  // Revalidation mode after initial submission
  reValidateMode: 'onChange', // 'onBlur' | 'onChange' | 'onSubmit'

  // Default values for all fields
  defaultValues: {
    firstName: '',
    lastName: '',
    email: '',
  },

  // Async default values (fetched from API)
  defaultValues: async () => {
    const response = await fetch('/api/user');
    return await response.json();
  },

  // External validation resolver (Zod, Yup, etc.)
  resolver: zodResolver(schema),

  // Context passed to resolver
  context: { userRole: 'admin' },

  // Focus first error on submit
  shouldFocusError: true,

  // Delay error display in milliseconds
  delayError: 500,

  // Use native validation
  shouldUseNativeValidation: false,

  // Unregister inputs on unmount
  shouldUnregister: false,

  // Validation criteria mode
  criteriaMode: 'firstError', // 'all' | 'firstError'

  // Form values object
  values: externalValues, // Sync form with external state

  // Reset options when values change
  resetOptions: {
    keepDirtyValues: true,
    keepErrors: false,
  },
});
```

### Return Values

```tsx
const {
  // Methods
  register,          // Register an input
  handleSubmit,      // Handle form submission
  watch,             // Watch field values
  setValue,          // Set a field value
  getValues,         // Get field values
  reset,             // Reset form
  resetField,        // Reset specific field
  setError,          // Set an error
  clearErrors,       // Clear errors
  setFocus,          // Focus a field
  trigger,           // Trigger validation
  control,           // Control object for Controller
  unregister,        // Unregister a field
  getFieldState,     // Get field state

  // State
  formState,         // Form state object
} = useForm();
```

### Validation Modes Explained

```tsx
// onSubmit (default) - Validate on form submission
const form = useForm({ mode: 'onSubmit' });

// onBlur - Validate when input loses focus
const form = useForm({ mode: 'onBlur' });

// onChange - Validate on every change (can impact performance)
const form = useForm({ mode: 'onChange' });

// onTouched - Validate on blur first, then on change
const form = useForm({ mode: 'onTouched' });

// all - Validate on blur and change
const form = useForm({ mode: 'all' });
```

---

## register Function

The `register` function connects inputs to the form state.

### Basic Usage

```tsx
<input {...register('fieldName')} />

// With type
<input type="email" {...register('email')} />

// Textarea
<textarea {...register('description')} />

// Select
<select {...register('country')}>
  <option value="us">United States</option>
  <option value="uk">United Kingdom</option>
</select>

// Checkbox
<input type="checkbox" {...register('acceptTerms')} />

// Radio buttons
<input type="radio" value="male" {...register('gender')} />
<input type="radio" value="female" {...register('gender')} />
```

### Register Options

```tsx
register('fieldName', {
  // Required field
  required: true,
  required: 'This field is required',
  required: { value: true, message: 'Required field' },

  // Minimum length
  minLength: 3,
  minLength: { value: 3, message: 'Min 3 characters' },

  // Maximum length
  maxLength: 100,
  maxLength: { value: 100, message: 'Max 100 characters' },

  // Pattern (regex)
  pattern: /^[A-Za-z]+$/,
  pattern: { value: /^[A-Za-z]+$/, message: 'Letters only' },

  // Min value (numbers)
  min: 0,
  min: { value: 0, message: 'Must be positive' },

  // Max value (numbers)
  max: 100,
  max: { value: 100, message: 'Max is 100' },

  // Custom validation
  validate: (value) => value !== 'admin' || 'Username not allowed',

  // Multiple validations
  validate: {
    positive: (v) => parseInt(v) > 0 || 'Must be positive',
    lessThan100: (v) => parseInt(v) < 100 || 'Must be less than 100',
    checkUrl: async (v) => {
      const isValid = await validateUrl(v);
      return isValid || 'URL not valid';
    },
  },

  // Value transformation
  valueAsNumber: true,     // Convert to number
  valueAsDate: true,       // Convert to Date
  setValueAs: (v) => parseInt(v, 10), // Custom transform

  // Disable field
  disabled: true,

  // onChange handler
  onChange: (e) => console.log(e.target.value),

  // onBlur handler
  onBlur: (e) => console.log('Blurred'),

  // Dependencies for validation
  deps: ['otherField'], // Re-validate when otherField changes
});
```

### Register Return Value

```tsx
const { onChange, onBlur, name, ref } = register('fieldName');

// Equivalent to spreading
<input
  onChange={onChange}
  onBlur={onBlur}
  name={name}
  ref={ref}
/>

// Or simply spread
<input {...register('fieldName')} />
```

---

## handleSubmit

The `handleSubmit` function handles form submission and validation.

### Basic Usage

```tsx
const onSubmit = (data) => {
  console.log(data);
};

const onError = (errors) => {
  console.log(errors);
};

<form onSubmit={handleSubmit(onSubmit, onError)}>
  {/* form fields */}
</form>
```

### Async Submit Handler

```tsx
const onSubmit = async (data: FormData) => {
  try {
    const response = await api.createUser(data);
    toast.success('User created!');
    router.push('/users');
  } catch (error) {
    if (error.response?.status === 409) {
      setError('email', { message: 'Email already exists' });
    } else {
      toast.error('Something went wrong');
    }
  }
};
```

### Prevent Double Submission

```tsx
function Form() {
  const { handleSubmit, formState: { isSubmitting } } = useForm();

  const onSubmit = async (data) => {
    await api.submit(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* fields */}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

### Manual Submit Trigger

```tsx
function Form() {
  const { handleSubmit, getValues } = useForm();

  const onSubmit = (data) => console.log(data);

  // Trigger submit programmatically
  const handleManualSubmit = () => {
    handleSubmit(onSubmit)();
  };

  // Or get values without validation
  const handleGetValues = () => {
    const values = getValues();
    console.log(values);
  };

  return (
    <form>
      {/* fields */}
      <button type="button" onClick={handleManualSubmit}>
        Submit Manually
      </button>
    </form>
  );
}
```

---

## Form State

The `formState` object contains information about the form's current state.

### All Form State Properties

```tsx
const {
  formState: {
    // Error state
    errors,           // Object containing field errors

    // Submission state
    isSubmitting,     // Form is currently submitting
    isSubmitted,      // Form has been submitted at least once
    isSubmitSuccessful, // Last submission was successful
    submitCount,      // Number of times form was submitted

    // Validation state
    isValid,          // Form is valid (no errors)
    isValidating,     // Form is currently validating

    // Dirty state (modified from default)
    isDirty,          // Any field has been modified
    dirtyFields,      // Object of modified fields

    // Touched state (user interacted)
    touchedFields,    // Object of fields user has interacted with

    // Loading state
    isLoading,        // Form is loading default values (async)

    // Disabled state
    disabled,         // Form is disabled

    // Default values
    defaultValues,    // Current default values
  },
} = useForm();
```

### Using Form State

```tsx
function Form() {
  const {
    register,
    handleSubmit,
    formState: {
      errors,
      isSubmitting,
      isValid,
      isDirty,
      dirtyFields,
      touchedFields,
    },
  } = useForm({
    mode: 'onChange',
    defaultValues: {
      email: '',
      password: '',
    },
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email', { required: true })} />

      {/* Show error only if touched */}
      {touchedFields.email && errors.email && (
        <span>{errors.email.message}</span>
      )}

      {/* Show dirty indicator */}
      {dirtyFields.email && <span>*</span>}

      <input type="password" {...register('password', { required: true })} />

      {/* Disable submit if not valid or not dirty */}
      <button
        type="submit"
        disabled={isSubmitting || !isValid || !isDirty}
      >
        {isSubmitting ? 'Saving...' : 'Save'}
      </button>

      {/* Show unsaved changes warning */}
      {isDirty && <p>You have unsaved changes</p>}
    </form>
  );
}
```

### Subscribing to Specific State

```tsx
// Only re-render when specific state changes
const { formState: { errors } } = useForm();

// Must read the property to subscribe to it
// This component only re-renders when errors change
function ErrorDisplay() {
  const { formState: { errors } } = useFormContext();
  return errors.email ? <span>{errors.email.message}</span> : null;
}
```

---

## Validation

### Built-in Validation

```tsx
// Required
<input {...register('name', { required: 'Name is required' })} />

// Min/Max length
<input {...register('username', {
  minLength: { value: 3, message: 'Min 3 characters' },
  maxLength: { value: 20, message: 'Max 20 characters' },
})} />

// Pattern
<input {...register('email', {
  pattern: {
    value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
    message: 'Invalid email address',
  },
})} />

// Min/Max value (numbers)
<input type="number" {...register('age', {
  min: { value: 18, message: 'Must be at least 18' },
  max: { value: 120, message: 'Invalid age' },
})} />

// Custom validation
<input {...register('username', {
  validate: (value) => {
    if (value.includes(' ')) return 'No spaces allowed';
    if (value === 'admin') return 'Username not available';
    return true;
  },
})} />

// Async validation
<input {...register('email', {
  validate: async (value) => {
    const exists = await checkEmailExists(value);
    return !exists || 'Email already registered';
  },
})} />
```

### Zod Validation

```tsx
import { z } from 'zod';
import { zodResolver } from '@hookform/resolvers/zod';

// Basic schema
const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Min 8 characters'),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: 'Passwords must match',
  path: ['confirmPassword'],
});

type FormData = z.infer<typeof schema>;

function Form() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}

      <input type="password" {...register('password')} />
      {errors.password && <span>{errors.password.message}</span>}

      <input type="password" {...register('confirmPassword')} />
      {errors.confirmPassword && <span>{errors.confirmPassword.message}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

### Advanced Zod Patterns

```tsx
// Optional fields
const schema = z.object({
  nickname: z.string().optional(),
  bio: z.string().nullable(),
  website: z.string().url().optional().or(z.literal('')),
});

// Enums
const schema = z.object({
  role: z.enum(['admin', 'user', 'moderator']),
  status: z.enum(['active', 'inactive']).default('active'),
});

// Numbers from string inputs
const schema = z.object({
  age: z.coerce.number().min(0).max(120),
  price: z.coerce.number().positive(),
});

// Dates
const schema = z.object({
  birthDate: z.coerce.date(),
  startDate: z.string().datetime(),
});

// Arrays
const schema = z.object({
  tags: z.array(z.string()).min(1, 'At least one tag').max(5, 'Max 5 tags'),
  permissions: z.array(z.enum(['read', 'write', 'delete'])),
});

// Nested objects
const schema = z.object({
  user: z.object({
    name: z.string().min(1),
    email: z.string().email(),
  }),
  address: z.object({
    street: z.string(),
    city: z.string(),
    zipCode: z.string().regex(/^\d{5}$/),
  }),
});

// Conditional validation with discriminated unions
const schema = z.discriminatedUnion('accountType', [
  z.object({
    accountType: z.literal('personal'),
    firstName: z.string().min(1),
    lastName: z.string().min(1),
  }),
  z.object({
    accountType: z.literal('business'),
    companyName: z.string().min(1),
    taxId: z.string().min(1),
  }),
]);

// Custom refinements
const schema = z.object({
  startDate: z.coerce.date(),
  endDate: z.coerce.date(),
}).refine((data) => data.endDate > data.startDate, {
  message: 'End date must be after start date',
  path: ['endDate'],
});

// Transform values
const schema = z.object({
  email: z.string().email().transform((v) => v.toLowerCase()),
  username: z.string().transform((v) => v.trim()),
});
```

### Yup Validation

```tsx
import * as yup from 'yup';
import { yupResolver } from '@hookform/resolvers/yup';

const schema = yup.object({
  email: yup.string().email('Invalid email').required('Email is required'),
  password: yup.string().min(8, 'Min 8 characters').required('Password is required'),
  confirmPassword: yup.string()
    .oneOf([yup.ref('password')], 'Passwords must match')
    .required('Please confirm password'),
}).required();

type FormData = yup.InferType<typeof schema>;

function Form() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: yupResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* form fields */}
    </form>
  );
}
```

---

## Error Messages

### Displaying Errors

```tsx
// Simple error display
{errors.email && <span className="error">{errors.email.message}</span>}

// Error component
function ErrorMessage({ error }: { error?: FieldError }) {
  if (!error) return null;
  return <span className="text-red-500 text-sm">{error.message}</span>;
}

<ErrorMessage error={errors.email} />

// Using ErrorMessage from react-hook-form
import { ErrorMessage } from '@hookform/error-message';

<ErrorMessage
  errors={errors}
  name="email"
  render={({ message }) => <p className="error">{message}</p>}
/>

// Multiple errors (criteriaMode: 'all')
<ErrorMessage
  errors={errors}
  name="password"
  render={({ messages }) =>
    messages &&
    Object.entries(messages).map(([type, message]) => (
      <p key={type} className="error">{message}</p>
    ))
  }
/>
```

### Setting Errors Programmatically

```tsx
const { setError, clearErrors } = useForm();

// Set single error
setError('email', {
  type: 'manual',
  message: 'Email already exists',
});

// Set error with focus
setError('email', { message: 'Invalid email' }, { shouldFocus: true });

// Set root error (form-level)
setError('root', { message: 'Form submission failed' });

// Set root error with custom type
setError('root.serverError', { message: 'Server is unavailable' });

// Clear single error
clearErrors('email');

// Clear multiple errors
clearErrors(['email', 'password']);

// Clear all errors
clearErrors();
```

### Server-Side Error Handling

```tsx
const onSubmit = async (data: FormData) => {
  try {
    await api.createUser(data);
  } catch (error) {
    // Handle field-specific errors
    if (error.response?.data?.errors) {
      const serverErrors = error.response.data.errors;

      Object.entries(serverErrors).forEach(([field, message]) => {
        setError(field as keyof FormData, {
          type: 'server',
          message: message as string,
        });
      });
    }

    // Handle general error
    setError('root.serverError', {
      type: 'server',
      message: error.response?.data?.message || 'Something went wrong',
    });
  }
};

// Display root error
{errors.root?.serverError && (
  <div className="bg-red-100 p-4 rounded">
    {errors.root.serverError.message}
  </div>
)}
```

---

## Controller Component

The `Controller` component wraps controlled components (like UI library inputs) that don't expose a ref.

### Basic Usage

```tsx
import { useForm, Controller } from 'react-hook-form';

function Form() {
  const { control, handleSubmit } = useForm();

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Controller
        name="firstName"
        control={control}
        defaultValue=""
        rules={{ required: 'First name is required' }}
        render={({ field, fieldState, formState }) => (
          <div>
            <input {...field} />
            {fieldState.error && <span>{fieldState.error.message}</span>}
          </div>
        )}
      />
    </form>
  );
}
```

### Controller Props

```tsx
<Controller
  // Required
  name="fieldName"
  control={control}
  render={({ field, fieldState, formState }) => <Component />}

  // Optional
  defaultValue=""
  rules={{ required: true }}
  shouldUnregister={false}
  disabled={false}
/>
```

### Render Props

```tsx
<Controller
  name="email"
  control={control}
  render={({
    field: {
      onChange,    // Send value to hook form
      onBlur,      // Notify when input is touched
      value,       // Current value
      name,        // Field name
      ref,         // Ref to focus input
      disabled,    // Disabled state
    },
    fieldState: {
      invalid,     // Field is invalid
      isTouched,   // Field has been touched
      isDirty,     // Field value differs from default
      error,       // Field error object
    },
    formState,     // Entire form state
  }) => (
    <TextField
      onChange={onChange}
      onBlur={onBlur}
      value={value}
      name={name}
      inputRef={ref}
      error={!!error}
      helperText={error?.message}
    />
  )}
/>
```

### With Third-Party Components

```tsx
// React Select
import Select from 'react-select';

<Controller
  name="country"
  control={control}
  render={({ field }) => (
    <Select
      {...field}
      options={countryOptions}
      onChange={(option) => field.onChange(option?.value)}
      value={countryOptions.find((c) => c.value === field.value)}
    />
  )}
/>

// DatePicker
import { DatePicker } from '@mui/x-date-pickers';

<Controller
  name="birthDate"
  control={control}
  render={({ field }) => (
    <DatePicker
      {...field}
      onChange={(date) => field.onChange(date)}
      slotProps={{
        textField: {
          error: !!errors.birthDate,
          helperText: errors.birthDate?.message,
        },
      }}
    />
  )}
/>

// Checkbox
<Controller
  name="acceptTerms"
  control={control}
  defaultValue={false}
  render={({ field }) => (
    <Checkbox
      checked={field.value}
      onChange={(e) => field.onChange(e.target.checked)}
    />
  )}
/>
```

---

## useWatch Hook

The `useWatch` hook watches specified inputs and returns their values.

### Basic Usage

```tsx
import { useWatch } from 'react-hook-form';

function WatchedValue({ control }) {
  // Watch single field
  const email = useWatch({
    control,
    name: 'email',
    defaultValue: '',
  });

  return <p>Email: {email}</p>;
}

// Watch multiple fields
function WatchedValues({ control }) {
  const [firstName, lastName] = useWatch({
    control,
    name: ['firstName', 'lastName'],
    defaultValue: ['', ''],
  });

  return <p>Full name: {firstName} {lastName}</p>;
}

// Watch all fields
function WatchAllFields({ control }) {
  const formValues = useWatch({ control });
  return <pre>{JSON.stringify(formValues, null, 2)}</pre>;
}
```

### useWatch vs watch

```tsx
// watch - Can cause extra re-renders at form level
const { watch } = useForm();
const email = watch('email'); // Form component re-renders

// useWatch - Isolates re-renders to the component using it
function EmailDisplay({ control }) {
  const email = useWatch({ control, name: 'email' });
  // Only this component re-renders when email changes
  return <p>{email}</p>;
}
```

### Practical Examples

```tsx
// Conditional fields
function Form() {
  const { control, register } = useForm();
  const hasAddress = useWatch({ control, name: 'hasAddress' });

  return (
    <form>
      <input type="checkbox" {...register('hasAddress')} />

      {hasAddress && (
        <>
          <input {...register('street')} placeholder="Street" />
          <input {...register('city')} placeholder="City" />
        </>
      )}
    </form>
  );
}

// Real-time preview
function FormWithPreview() {
  const { control, register } = useForm();

  return (
    <div className="flex gap-4">
      <form>
        <input {...register('title')} />
        <textarea {...register('content')} />
      </form>

      <Preview control={control} />
    </div>
  );
}

function Preview({ control }) {
  const { title, content } = useWatch({ control });

  return (
    <article>
      <h1>{title}</h1>
      <div>{content}</div>
    </article>
  );
}
```

---

## useFieldArray Hook

The `useFieldArray` hook manages dynamic fields/arrays.

### Basic Usage

```tsx
import { useForm, useFieldArray } from 'react-hook-form';

type FormValues = {
  users: { name: string; email: string }[];
};

function DynamicForm() {
  const { control, register, handleSubmit } = useForm<FormValues>({
    defaultValues: {
      users: [{ name: '', email: '' }],
    },
  });

  const { fields, append, remove, prepend, insert, move, swap } = useFieldArray({
    control,
    name: 'users',
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {fields.map((field, index) => (
        <div key={field.id}>
          <input {...register(`users.${index}.name`)} />
          <input {...register(`users.${index}.email`)} />
          <button type="button" onClick={() => remove(index)}>
            Remove
          </button>
        </div>
      ))}

      <button type="button" onClick={() => append({ name: '', email: '' })}>
        Add User
      </button>

      <button type="submit">Submit</button>
    </form>
  );
}
```

### All useFieldArray Methods

```tsx
const {
  fields,    // Array of field objects with 'id' property
  append,    // Add to end
  prepend,   // Add to beginning
  insert,    // Insert at index
  remove,    // Remove at index
  move,      // Move from one index to another
  swap,      // Swap two indices
  replace,   // Replace entire array
  update,    // Update at index
} = useFieldArray({
  control,
  name: 'items',
  keyName: 'id',           // Custom key name (default: 'id')
  shouldUnregister: false, // Keep values when unmounting
  rules: {                 // Validation rules
    minLength: 1,
    maxLength: 10,
    required: true,
  },
});

// Examples
append({ name: '' });                    // Add one item
append([{ name: '' }, { name: '' }]);   // Add multiple items
prepend({ name: '' });                   // Add to beginning
insert(2, { name: '' });                 // Insert at index 2
remove(0);                               // Remove first item
remove([0, 2]);                          // Remove multiple
move(0, 2);                              // Move index 0 to index 2
swap(0, 1);                              // Swap indices 0 and 1
replace([{ name: 'New' }]);              // Replace entire array
update(0, { name: 'Updated' });          // Update at index
```

### Nested Arrays

```tsx
type FormValues = {
  groups: {
    name: string;
    members: { name: string; role: string }[];
  }[];
};

function NestedFieldArray() {
  const { control, register } = useForm<FormValues>();

  const { fields: groupFields, append: appendGroup } = useFieldArray({
    control,
    name: 'groups',
  });

  return (
    <form>
      {groupFields.map((group, groupIndex) => (
        <div key={group.id}>
          <input {...register(`groups.${groupIndex}.name`)} />

          <MembersFieldArray
            control={control}
            groupIndex={groupIndex}
          />
        </div>
      ))}
    </form>
  );
}

function MembersFieldArray({ control, groupIndex }) {
  const { fields, append, remove } = useFieldArray({
    control,
    name: `groups.${groupIndex}.members`,
  });

  return (
    <div>
      {fields.map((member, memberIndex) => (
        <div key={member.id}>
          <input
            {...register(`groups.${groupIndex}.members.${memberIndex}.name`)}
          />
          <button type="button" onClick={() => remove(memberIndex)}>
            Remove
          </button>
        </div>
      ))}
      <button type="button" onClick={() => append({ name: '', role: '' })}>
        Add Member
      </button>
    </div>
  );
}
```

---

## setValue and getValues

### setValue

```tsx
const { setValue, getValues } = useForm();

// Basic usage
setValue('email', 'user@example.com');

// Set multiple values
setValue('firstName', 'John');
setValue('lastName', 'Doe');

// With options
setValue('email', 'user@example.com', {
  shouldValidate: true,    // Trigger validation
  shouldDirty: true,       // Mark as dirty
  shouldTouch: true,       // Mark as touched
});

// Set nested value
setValue('user.address.city', 'New York');

// Set array value
setValue('items.0.name', 'First Item');
setValue('items', [{ name: 'Item 1' }, { name: 'Item 2' }]);
```

### getValues

```tsx
const { getValues } = useForm();

// Get all values
const allValues = getValues();

// Get single value
const email = getValues('email');

// Get multiple values
const [firstName, lastName] = getValues(['firstName', 'lastName']);

// Get nested value
const city = getValues('user.address.city');
```

### Practical Examples

```tsx
// Populate form from API
function EditForm({ userId }) {
  const { setValue, handleSubmit } = useForm();

  useEffect(() => {
    const fetchUser = async () => {
      const user = await api.getUser(userId);
      setValue('firstName', user.firstName);
      setValue('lastName', user.lastName);
      setValue('email', user.email);
      // Or use reset for entire form
    };
    fetchUser();
  }, [userId, setValue]);

  return <form>...</form>;
}

// Copy field value
function Form() {
  const { register, setValue, getValues } = useForm();

  const copyBillingToShipping = () => {
    const billing = getValues('billingAddress');
    setValue('shippingAddress', billing);
  };

  return (
    <form>
      <fieldset>
        <legend>Billing Address</legend>
        <input {...register('billingAddress.street')} />
        <input {...register('billingAddress.city')} />
      </fieldset>

      <button type="button" onClick={copyBillingToShipping}>
        Same as billing
      </button>

      <fieldset>
        <legend>Shipping Address</legend>
        <input {...register('shippingAddress.street')} />
        <input {...register('shippingAddress.city')} />
      </fieldset>
    </form>
  );
}
```

---

## reset and trigger

### reset

```tsx
const { reset } = useForm();

// Reset to default values
reset();

// Reset with new values
reset({
  firstName: 'John',
  lastName: 'Doe',
  email: 'john@example.com',
});

// Reset with options
reset(values, {
  keepErrors: false,           // Keep errors
  keepDirty: false,            // Keep dirty state
  keepDirtyValues: false,      // Keep dirty field values
  keepValues: false,           // Keep current values
  keepDefaultValues: false,    // Keep default values
  keepIsSubmitted: false,      // Keep isSubmitted state
  keepTouched: false,          // Keep touched state
  keepIsValid: false,          // Keep isValid state
  keepSubmitCount: false,      // Keep submit count
});

// Reset single field
const { resetField } = useForm();
resetField('email');
resetField('email', {
  keepError: true,
  keepDirty: true,
  keepTouched: true,
  defaultValue: 'new@email.com',
});
```

### trigger

```tsx
const { trigger } = useForm();

// Validate all fields
await trigger();

// Validate single field
await trigger('email');

// Validate multiple fields
await trigger(['email', 'password']);

// With shouldFocus option
await trigger('email', { shouldFocus: true });

// Example: Validate on step change in wizard
const nextStep = async () => {
  const isValid = await trigger(['firstName', 'lastName', 'email']);
  if (isValid) {
    setStep(step + 1);
  }
};
```

### Practical Examples

```tsx
// Reset after successful submit
const onSubmit = async (data) => {
  await api.submit(data);
  reset(); // Clear form after success
};

// Reset form on modal close
function FormModal({ isOpen, onClose }) {
  const { reset, handleSubmit } = useForm();

  const handleClose = () => {
    reset();
    onClose();
  };

  return (
    <Modal isOpen={isOpen} onClose={handleClose}>
      <form onSubmit={handleSubmit(onSubmit)}>
        {/* fields */}
      </form>
    </Modal>
  );
}

// Multi-step form validation
function WizardForm() {
  const [step, setStep] = useState(1);
  const { register, trigger, handleSubmit } = useForm();

  const stepFields = {
    1: ['firstName', 'lastName'],
    2: ['email', 'phone'],
    3: ['address', 'city', 'zipCode'],
  };

  const nextStep = async () => {
    const isValid = await trigger(stepFields[step]);
    if (isValid) {
      setStep(step + 1);
    }
  };

  const prevStep = () => {
    setStep(step - 1);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {step === 1 && (
        <>
          <input {...register('firstName')} />
          <input {...register('lastName')} />
        </>
      )}

      {step === 2 && (
        <>
          <input {...register('email')} />
          <input {...register('phone')} />
        </>
      )}

      {step > 1 && (
        <button type="button" onClick={prevStep}>Previous</button>
      )}

      {step < 3 ? (
        <button type="button" onClick={nextStep}>Next</button>
      ) : (
        <button type="submit">Submit</button>
      )}
    </form>
  );
}
```

---

## Integration with UI Libraries

### shadcn/ui Integration

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';
import { Checkbox } from '@/components/ui/checkbox';
import { Textarea } from '@/components/ui/textarea';
import { Switch } from '@/components/ui/switch';
import { RadioGroup, RadioGroupItem } from '@/components/ui/radio-group';

const schema = z.object({
  username: z.string().min(3, 'Min 3 characters'),
  email: z.string().email('Invalid email'),
  role: z.enum(['admin', 'user', 'moderator']),
  bio: z.string().max(500).optional(),
  notifications: z.boolean().default(false),
  acceptTerms: z.boolean().refine((v) => v === true, 'You must accept terms'),
  contactMethod: z.enum(['email', 'phone', 'mail']),
});

type FormData = z.infer<typeof schema>;

function ShadcnForm() {
  const form = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: {
      username: '',
      email: '',
      role: 'user',
      bio: '',
      notifications: false,
      acceptTerms: false,
      contactMethod: 'email',
    },
  });

  const onSubmit = async (data: FormData) => {
    console.log(data);
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        {/* Input */}
        <FormField
          control={form.control}
          name="username"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Username</FormLabel>
              <FormControl>
                <Input placeholder="johndoe" {...field} />
              </FormControl>
              <FormDescription>
                Your public display name.
              </FormDescription>
              <FormMessage />
            </FormItem>
          )}
        />

        {/* Select */}
        <FormField
          control={form.control}
          name="role"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Role</FormLabel>
              <Select onValueChange={field.onChange} defaultValue={field.value}>
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="Select a role" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  <SelectItem value="admin">Admin</SelectItem>
                  <SelectItem value="user">User</SelectItem>
                  <SelectItem value="moderator">Moderator</SelectItem>
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />

        {/* Textarea */}
        <FormField
          control={form.control}
          name="bio"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Bio</FormLabel>
              <FormControl>
                <Textarea
                  placeholder="Tell us about yourself"
                  className="resize-none"
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        {/* Switch */}
        <FormField
          control={form.control}
          name="notifications"
          render={({ field }) => (
            <FormItem className="flex items-center justify-between">
              <div>
                <FormLabel>Email Notifications</FormLabel>
                <FormDescription>
                  Receive emails about your account.
                </FormDescription>
              </div>
              <FormControl>
                <Switch
                  checked={field.value}
                  onCheckedChange={field.onChange}
                />
              </FormControl>
            </FormItem>
          )}
        />

        {/* Checkbox */}
        <FormField
          control={form.control}
          name="acceptTerms"
          render={({ field }) => (
            <FormItem className="flex items-start space-x-3 space-y-0">
              <FormControl>
                <Checkbox
                  checked={field.value}
                  onCheckedChange={field.onChange}
                />
              </FormControl>
              <div className="space-y-1 leading-none">
                <FormLabel>Accept terms and conditions</FormLabel>
                <FormDescription>
                  You agree to our Terms of Service and Privacy Policy.
                </FormDescription>
              </div>
              <FormMessage />
            </FormItem>
          )}
        />

        {/* Radio Group */}
        <FormField
          control={form.control}
          name="contactMethod"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Preferred Contact Method</FormLabel>
              <FormControl>
                <RadioGroup
                  onValueChange={field.onChange}
                  defaultValue={field.value}
                  className="flex flex-col space-y-1"
                >
                  <FormItem className="flex items-center space-x-3 space-y-0">
                    <FormControl>
                      <RadioGroupItem value="email" />
                    </FormControl>
                    <FormLabel className="font-normal">Email</FormLabel>
                  </FormItem>
                  <FormItem className="flex items-center space-x-3 space-y-0">
                    <FormControl>
                      <RadioGroupItem value="phone" />
                    </FormControl>
                    <FormLabel className="font-normal">Phone</FormLabel>
                  </FormItem>
                  <FormItem className="flex items-center space-x-3 space-y-0">
                    <FormControl>
                      <RadioGroupItem value="mail" />
                    </FormControl>
                    <FormLabel className="font-normal">Mail</FormLabel>
                  </FormItem>
                </RadioGroup>
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? 'Submitting...' : 'Submit'}
        </Button>
      </form>
    </Form>
  );
}
```

### Material-UI (MUI) Integration

```tsx
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import TextField from '@mui/material/TextField';
import Button from '@mui/material/Button';
import Select from '@mui/material/Select';
import MenuItem from '@mui/material/MenuItem';
import FormControl from '@mui/material/FormControl';
import InputLabel from '@mui/material/InputLabel';
import FormHelperText from '@mui/material/FormHelperText';
import Checkbox from '@mui/material/Checkbox';
import FormControlLabel from '@mui/material/FormControlLabel';

function MuiForm() {
  const {
    control,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: {
      email: '',
      role: '',
      acceptTerms: false,
    },
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Controller
        name="email"
        control={control}
        render={({ field }) => (
          <TextField
            {...field}
            label="Email"
            variant="outlined"
            fullWidth
            margin="normal"
            error={!!errors.email}
            helperText={errors.email?.message}
          />
        )}
      />

      <Controller
        name="role"
        control={control}
        render={({ field }) => (
          <FormControl fullWidth margin="normal" error={!!errors.role}>
            <InputLabel>Role</InputLabel>
            <Select {...field} label="Role">
              <MenuItem value="admin">Admin</MenuItem>
              <MenuItem value="user">User</MenuItem>
            </Select>
            {errors.role && (
              <FormHelperText>{errors.role.message}</FormHelperText>
            )}
          </FormControl>
        )}
      />

      <Controller
        name="acceptTerms"
        control={control}
        render={({ field }) => (
          <FormControlLabel
            control={
              <Checkbox
                checked={field.value}
                onChange={(e) => field.onChange(e.target.checked)}
              />
            }
            label="Accept Terms"
          />
        )}
      />
      {errors.acceptTerms && (
        <FormHelperText error>{errors.acceptTerms.message}</FormHelperText>
      )}

      <Button
        type="submit"
        variant="contained"
        disabled={isSubmitting}
        fullWidth
      >
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </Button>
    </form>
  );
}
```

---

## Integration with TanStack Query

### Create/Update Form Pattern

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { toast } from 'sonner';

const userSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  role: z.enum(['admin', 'user']),
});

type UserFormData = z.infer<typeof userSchema>;

interface UserFormProps {
  userId?: string;
  onSuccess?: () => void;
}

function UserForm({ userId, onSuccess }: UserFormProps) {
  const queryClient = useQueryClient();
  const isEditing = !!userId;

  // Fetch existing user data for editing
  const { data: user, isLoading } = useQuery({
    queryKey: ['users', userId],
    queryFn: () => usersApi.getById(userId!),
    enabled: isEditing,
  });

  const form = useForm<UserFormData>({
    resolver: zodResolver(userSchema),
    defaultValues: {
      name: '',
      email: '',
      role: 'user',
    },
    values: user, // Sync with fetched data
  });

  // Create mutation
  const createMutation = useMutation({
    mutationFn: usersApi.create,
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
      toast.success('User created successfully');
      form.reset();
      onSuccess?.();
    },
    onError: (error: ApiError) => {
      if (error.response?.data?.errors) {
        Object.entries(error.response.data.errors).forEach(([field, msg]) => {
          form.setError(field as keyof UserFormData, { message: msg as string });
        });
      } else {
        toast.error(error.message || 'Failed to create user');
      }
    },
  });

  // Update mutation
  const updateMutation = useMutation({
    mutationFn: (data: UserFormData) => usersApi.update(userId!, data),
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
      queryClient.setQueryData(['users', userId], data);
      toast.success('User updated successfully');
      onSuccess?.();
    },
    onError: (error: ApiError) => {
      toast.error(error.message || 'Failed to update user');
    },
  });

  const mutation = isEditing ? updateMutation : createMutation;

  const onSubmit = (data: UserFormData) => {
    mutation.mutate(data);
  };

  if (isEditing && isLoading) {
    return <div>Loading...</div>;
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
      <FormField
        control={form.control}
        name="name"
        render={({ field }) => (
          <FormItem>
            <FormLabel>Name</FormLabel>
            <FormControl>
              <Input {...field} />
            </FormControl>
            <FormMessage />
          </FormItem>
        )}
      />

      <FormField
        control={form.control}
        name="email"
        render={({ field }) => (
          <FormItem>
            <FormLabel>Email</FormLabel>
            <FormControl>
              <Input type="email" {...field} />
            </FormControl>
            <FormMessage />
          </FormItem>
        )}
      />

      <FormField
        control={form.control}
        name="role"
        render={({ field }) => (
          <FormItem>
            <FormLabel>Role</FormLabel>
            <Select onValueChange={field.onChange} value={field.value}>
              <FormControl>
                <SelectTrigger>
                  <SelectValue />
                </SelectTrigger>
              </FormControl>
              <SelectContent>
                <SelectItem value="admin">Admin</SelectItem>
                <SelectItem value="user">User</SelectItem>
              </SelectContent>
            </Select>
            <FormMessage />
          </FormItem>
        )}
      />

      <div className="flex gap-2">
        <Button type="submit" disabled={mutation.isPending}>
          {mutation.isPending
            ? isEditing ? 'Updating...' : 'Creating...'
            : isEditing ? 'Update User' : 'Create User'
          }
        </Button>

        {form.formState.isDirty && (
          <Button type="button" variant="outline" onClick={() => form.reset()}>
            Reset
          </Button>
        )}
      </div>
    </form>
  );
}
```

### Optimistic Updates

```tsx
const updateMutation = useMutation({
  mutationFn: (data: UserFormData) => usersApi.update(userId!, data),
  onMutate: async (newData) => {
    // Cancel outgoing queries
    await queryClient.cancelQueries({ queryKey: ['users', userId] });

    // Snapshot previous value
    const previousUser = queryClient.getQueryData(['users', userId]);

    // Optimistically update
    queryClient.setQueryData(['users', userId], (old: User) => ({
      ...old,
      ...newData,
    }));

    return { previousUser };
  },
  onError: (err, newData, context) => {
    // Rollback on error
    queryClient.setQueryData(['users', userId], context?.previousUser);
    toast.error('Failed to update user');
  },
  onSettled: () => {
    // Refetch to ensure consistency
    queryClient.invalidateQueries({ queryKey: ['users', userId] });
  },
});
```

---

## File Uploads

### Single File Upload

```tsx
import { useForm } from 'react-hook-form';
import { z } from 'zod';
import { zodResolver } from '@hookform/resolvers/zod';

const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
const ACCEPTED_IMAGE_TYPES = ['image/jpeg', 'image/png', 'image/webp'];

const schema = z.object({
  name: z.string().min(1),
  avatar: z
    .instanceof(FileList)
    .refine((files) => files.length === 1, 'Image is required')
    .refine(
      (files) => files[0]?.size <= MAX_FILE_SIZE,
      'Max file size is 5MB'
    )
    .refine(
      (files) => ACCEPTED_IMAGE_TYPES.includes(files[0]?.type),
      'Only .jpg, .png, and .webp formats are supported'
    ),
});

type FormData = z.infer<typeof schema>;

function FileUploadForm() {
  const {
    register,
    handleSubmit,
    formState: { errors },
    watch,
  } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  const avatar = watch('avatar');
  const preview = avatar?.[0] ? URL.createObjectURL(avatar[0]) : null;

  const onSubmit = async (data: FormData) => {
    const formData = new FormData();
    formData.append('name', data.name);
    formData.append('avatar', data.avatar[0]);

    await fetch('/api/upload', {
      method: 'POST',
      body: formData,
    });
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} />
      {errors.name && <span>{errors.name.message}</span>}

      <input
        type="file"
        accept="image/jpeg,image/png,image/webp"
        {...register('avatar')}
      />
      {errors.avatar && <span>{errors.avatar.message}</span>}

      {preview && (
        <img src={preview} alt="Preview" className="w-24 h-24 object-cover" />
      )}

      <button type="submit">Upload</button>
    </form>
  );
}
```

### Multiple File Upload

```tsx
const schema = z.object({
  documents: z
    .instanceof(FileList)
    .refine((files) => files.length >= 1, 'At least one file is required')
    .refine((files) => files.length <= 5, 'Max 5 files allowed')
    .refine(
      (files) => Array.from(files).every((file) => file.size <= MAX_FILE_SIZE),
      'Each file must be under 5MB'
    ),
});

function MultiFileUpload() {
  const { register, handleSubmit, watch } = useForm();
  const files = watch('documents');

  const onSubmit = async (data) => {
    const formData = new FormData();
    Array.from(data.documents).forEach((file, index) => {
      formData.append(`document_${index}`, file);
    });

    await fetch('/api/upload-multiple', {
      method: 'POST',
      body: formData,
    });
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input type="file" multiple {...register('documents')} />

      {files && files.length > 0 && (
        <ul>
          {Array.from(files).map((file, index) => (
            <li key={index}>{file.name} ({(file.size / 1024).toFixed(2)} KB)</li>
          ))}
        </ul>
      )}

      <button type="submit">Upload</button>
    </form>
  );
}
```

### Drag and Drop with Controller

```tsx
import { useDropzone } from 'react-dropzone';
import { Controller } from 'react-hook-form';

function DropzoneField({ control, name }) {
  return (
    <Controller
      name={name}
      control={control}
      defaultValue={[]}
      render={({ field: { onChange, value } }) => (
        <Dropzone
          files={value}
          onChange={onChange}
        />
      )}
    />
  );
}

function Dropzone({ files, onChange }) {
  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    accept: {
      'image/*': ['.jpeg', '.jpg', '.png', '.webp'],
    },
    maxSize: 5 * 1024 * 1024,
    onDrop: (acceptedFiles) => {
      onChange([...files, ...acceptedFiles]);
    },
  });

  const removeFile = (index: number) => {
    const newFiles = [...files];
    newFiles.splice(index, 1);
    onChange(newFiles);
  };

  return (
    <div>
      <div
        {...getRootProps()}
        className={`border-2 border-dashed p-8 text-center cursor-pointer
          ${isDragActive ? 'border-blue-500 bg-blue-50' : 'border-gray-300'}
        `}
      >
        <input {...getInputProps()} />
        {isDragActive ? (
          <p>Drop files here...</p>
        ) : (
          <p>Drag & drop files here, or click to select</p>
        )}
      </div>

      {files.length > 0 && (
        <ul className="mt-4 space-y-2">
          {files.map((file, index) => (
            <li key={index} className="flex justify-between items-center">
              <span>{file.name}</span>
              <button type="button" onClick={() => removeFile(index)}>
                Remove
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## TypeScript Integration

### Form Types

```tsx
import { useForm, SubmitHandler, FieldErrors } from 'react-hook-form';

// Define form data type
interface LoginFormData {
  email: string;
  password: string;
  rememberMe: boolean;
}

// Or infer from Zod schema
const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  rememberMe: z.boolean(),
});

type LoginFormData = z.infer<typeof loginSchema>;

function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: '',
      password: '',
      rememberMe: false,
    },
  });

  // Typed submit handler
  const onSubmit: SubmitHandler<LoginFormData> = async (data) => {
    // data is fully typed
    console.log(data.email); // string
  };

  // Typed error handler
  const onError = (errors: FieldErrors<LoginFormData>) => {
    console.log(errors.email?.message);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit, onError)}>
      <input {...register('email')} />
      <input type="password" {...register('password')} />
      <input type="checkbox" {...register('rememberMe')} />
      <button type="submit">Login</button>
    </form>
  );
}
```

### Generic Form Component

```tsx
import { UseFormReturn, FieldValues, Path } from 'react-hook-form';

interface FormInputProps<T extends FieldValues> {
  form: UseFormReturn<T>;
  name: Path<T>;
  label: string;
  type?: string;
  placeholder?: string;
}

function FormInput<T extends FieldValues>({
  form,
  name,
  label,
  type = 'text',
  placeholder,
}: FormInputProps<T>) {
  const { register, formState: { errors } } = form;
  const error = errors[name];

  return (
    <div>
      <label htmlFor={name}>{label}</label>
      <input
        id={name}
        type={type}
        placeholder={placeholder}
        {...register(name)}
      />
      {error && <span className="error">{error.message as string}</span>}
    </div>
  );
}

// Usage
function MyForm() {
  const form = useForm<LoginFormData>();

  return (
    <form>
      <FormInput form={form} name="email" label="Email" type="email" />
      <FormInput form={form} name="password" label="Password" type="password" />
    </form>
  );
}
```

### Typed useFieldArray

```tsx
interface TeamMember {
  name: string;
  email: string;
  role: 'lead' | 'member';
}

interface TeamFormData {
  teamName: string;
  members: TeamMember[];
}

function TeamForm() {
  const { control, register, handleSubmit } = useForm<TeamFormData>({
    defaultValues: {
      teamName: '',
      members: [{ name: '', email: '', role: 'member' }],
    },
  });

  const { fields, append, remove } = useFieldArray({
    control,
    name: 'members',
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('teamName')} />

      {fields.map((field, index) => (
        <div key={field.id}>
          {/* Fully typed paths */}
          <input {...register(`members.${index}.name`)} />
          <input {...register(`members.${index}.email`)} />
          <select {...register(`members.${index}.role`)}>
            <option value="lead">Lead</option>
            <option value="member">Member</option>
          </select>
          <button type="button" onClick={() => remove(index)}>Remove</button>
        </div>
      ))}

      {/* Typed append */}
      <button
        type="button"
        onClick={() => append({ name: '', email: '', role: 'member' })}
      >
        Add Member
      </button>
    </form>
  );
}
```

---

## Performance Optimization

### Isolate Re-renders

```tsx
// Bad: Entire form re-renders on every change
function BadForm() {
  const { register, watch } = useForm();
  const email = watch('email'); // Causes re-render on every keystroke

  return (
    <form>
      <input {...register('email')} />
      <input {...register('password')} />
      <p>Email: {email}</p>
    </form>
  );
}

// Good: Only the display component re-renders
function GoodForm() {
  const { register, control } = useForm();

  return (
    <form>
      <input {...register('email')} />
      <input {...register('password')} />
      <EmailDisplay control={control} />
    </form>
  );
}

function EmailDisplay({ control }) {
  const email = useWatch({ control, name: 'email' });
  return <p>Email: {email}</p>;
}
```

### Use FormProvider for Deeply Nested Forms

```tsx
import { useForm, FormProvider, useFormContext } from 'react-hook-form';

function ParentForm() {
  const methods = useForm();

  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        <PersonalInfo />
        <AddressInfo />
        <button type="submit">Submit</button>
      </form>
    </FormProvider>
  );
}

function PersonalInfo() {
  const { register } = useFormContext();
  return (
    <fieldset>
      <input {...register('firstName')} />
      <input {...register('lastName')} />
    </fieldset>
  );
}

function AddressInfo() {
  const { register } = useFormContext();
  return (
    <fieldset>
      <input {...register('address.street')} />
      <input {...register('address.city')} />
    </fieldset>
  );
}
```

### Memoize Components

```tsx
import { memo } from 'react';

// Memoize static parts of the form
const MemoizedFormSection = memo(function FormSection({ register }) {
  return (
    <fieldset>
      <input {...register('field1')} />
      <input {...register('field2')} />
    </fieldset>
  );
});

// Use stable references for callbacks
function Form() {
  const { register, handleSubmit } = useForm();

  const onSubmit = useCallback((data) => {
    console.log(data);
  }, []);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <MemoizedFormSection register={register} />
    </form>
  );
}
```

### Lazy Validation

```tsx
// Validate only on submit (default, most performant)
const form = useForm({ mode: 'onSubmit' });

// Validate on blur (good balance)
const form = useForm({ mode: 'onBlur' });

// Avoid onChange mode for large forms
// const form = useForm({ mode: 'onChange' }); // Can be slow
```

### Debounce Async Validation

```tsx
import { useDebouncedCallback } from 'use-debounce';

function Form() {
  const { register, setError, clearErrors } = useForm();

  const checkUsername = useDebouncedCallback(async (value: string) => {
    if (value.length < 3) return;

    const exists = await api.checkUsername(value);
    if (exists) {
      setError('username', { message: 'Username taken' });
    } else {
      clearErrors('username');
    }
  }, 500);

  return (
    <form>
      <input
        {...register('username', {
          onChange: (e) => checkUsername(e.target.value),
        })}
      />
    </form>
  );
}
```

---

## Best Practices

### 1. Always Provide Default Values

```tsx
// Good: Explicit default values
const form = useForm({
  defaultValues: {
    email: '',
    password: '',
    role: 'user',
  },
});

// Bad: Undefined default values can cause issues
const form = useForm();
```

### 2. Use Zod for Complex Validation

```tsx
// Good: Centralized, reusable schema
const userSchema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Min 8 characters'),
});

const form = useForm({
  resolver: zodResolver(userSchema),
});

// Avoid: Inline validation gets messy
<input {...register('email', {
  required: 'Required',
  pattern: { value: /.../, message: 'Invalid' },
  validate: async (v) => { ... }
})} />
```

### 3. Handle Loading States Properly

```tsx
function Form() {
  const { formState: { isSubmitting }, handleSubmit } = useForm();

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* Disable all inputs during submission */}
      <fieldset disabled={isSubmitting}>
        <input {...register('email')} />
        <input {...register('password')} />
      </fieldset>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

### 4. Use setError for Server-Side Errors

```tsx
const onSubmit = async (data) => {
  try {
    await api.submit(data);
  } catch (error) {
    // Map server errors to form fields
    if (error.response?.data?.field) {
      setError(error.response.data.field, {
        type: 'server',
        message: error.response.data.message,
      });
    } else {
      setError('root.serverError', {
        message: 'Something went wrong',
      });
    }
  }
};
```

### 5. Reset Form After Successful Submission

```tsx
const onSubmit = async (data) => {
  const result = await api.submit(data);
  if (result.success) {
    reset(); // Clear form
    toast.success('Submitted!');
  }
};
```

### 6. Use useWatch Sparingly

```tsx
// Good: Watch only what you need in isolated components
function PriceCalculator({ control }) {
  const [quantity, price] = useWatch({
    control,
    name: ['quantity', 'price'],
  });

  return <p>Total: ${quantity * price}</p>;
}

// Bad: Watching everything in the main form component
function Form() {
  const { watch } = useForm();
  const allValues = watch(); // Re-renders on every change
}
```

### 7. Type Your Forms

```tsx
// Always use TypeScript for better DX
interface FormData {
  email: string;
  password: string;
}

const { register } = useForm<FormData>();

// Now register will autocomplete and type-check field names
register('email'); // OK
register('invalid'); // TypeScript error
```

### 8. Create Reusable Form Components

```tsx
// Create typed, reusable field components
interface TextFieldProps {
  name: Path<FormData>;
  label: string;
  placeholder?: string;
}

function TextField({ name, label, placeholder }: TextFieldProps) {
  const { control } = useFormContext<FormData>();

  return (
    <FormField
      control={control}
      name={name}
      render={({ field }) => (
        <FormItem>
          <FormLabel>{label}</FormLabel>
          <FormControl>
            <Input placeholder={placeholder} {...field} />
          </FormControl>
          <FormMessage />
        </FormItem>
      )}
    />
  );
}
```

### 9. Handle Form Dirty State

```tsx
function Form() {
  const { formState: { isDirty }, handleSubmit, reset } = useForm();

  // Warn before leaving with unsaved changes
  useEffect(() => {
    const handleBeforeUnload = (e: BeforeUnloadEvent) => {
      if (isDirty) {
        e.preventDefault();
        e.returnValue = '';
      }
    };

    window.addEventListener('beforeunload', handleBeforeUnload);
    return () => window.removeEventListener('beforeunload', handleBeforeUnload);
  }, [isDirty]);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* form content */}
      {isDirty && <p className="text-yellow-600">You have unsaved changes</p>}
    </form>
  );
}
```

### 10. Structure Large Forms

```tsx
// Break large forms into logical sections
function CompleteForm() {
  const methods = useForm<CompleteFormData>();
  const [step, setStep] = useState(1);

  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        {step === 1 && <PersonalInfoSection />}
        {step === 2 && <AddressSection />}
        {step === 3 && <PaymentSection />}
        {step === 4 && <ReviewSection />}

        <FormNavigation
          step={step}
          setStep={setStep}
          totalSteps={4}
        />
      </form>
    </FormProvider>
  );
}
```

---

## Quick Reference

| Method/Property | Description |
|----------------|-------------|
| `register` | Connect input to form state |
| `handleSubmit` | Handle form submission |
| `watch` | Watch field values (causes re-render) |
| `useWatch` | Watch values (isolated re-render) |
| `setValue` | Set field value programmatically |
| `getValues` | Get current form values |
| `reset` | Reset form to default values |
| `resetField` | Reset specific field |
| `trigger` | Trigger validation |
| `setError` | Set error manually |
| `clearErrors` | Clear errors |
| `setFocus` | Focus a field |
| `formState.errors` | Current errors |
| `formState.isSubmitting` | Form is submitting |
| `formState.isValid` | Form is valid |
| `formState.isDirty` | Form has been modified |
| `formState.dirtyFields` | Modified fields |
| `formState.touchedFields` | Touched fields |
| `Controller` | Wrapper for controlled components |
| `useFieldArray` | Manage dynamic fields |
| `FormProvider` | Context for nested forms |
| `useFormContext` | Access form in nested components |

---

## Additional Resources

- [Official Documentation](https://react-hook-form.com/)
- [API Reference](https://react-hook-form.com/docs)
- [Examples](https://github.com/react-hook-form/react-hook-form/tree/master/examples)
- [Zod Integration](https://github.com/react-hook-form/resolvers#zod)
- [DevTools](https://react-hook-form.com/dev-tools)
