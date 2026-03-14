# React Forms Reference

## Controlled Components

```tsx
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log({ email, password });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={e => setEmail(e.target.value)}
      />
      <input
        type="password"
        value={password}
        onChange={e => setPassword(e.target.value)}
      />
      <button type="submit">Login</button>
    </form>
  );
}
```

## Uncontrolled Components

```tsx
function ContactForm() {
  const nameRef = useRef<HTMLInputElement>(null);
  const emailRef = useRef<HTMLInputElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log({
      name: nameRef.current?.value,
      email: emailRef.current?.value,
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={nameRef} defaultValue="" />
      <input ref={emailRef} type="email" defaultValue="" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

## React 19 Form Actions

```tsx
async function createUser(formData: FormData) {
  'use server';
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  await db.users.create({ name, email });
  revalidatePath('/users');
}

function CreateUserForm() {
  return (
    <form action={createUser}>
      <input name="name" required />
      <input name="email" type="email" required />
      <button type="submit">Create</button>
    </form>
  );
}
```

## useActionState (React 19)

```tsx
const [state, action, isPending] = useActionState(
  async (prevState, formData: FormData) => {
    const email = formData.get('email') as string;

    if (!email.includes('@')) {
      return { error: 'Invalid email' };
    }

    await submitForm(formData);
    return { success: true };
  },
  { error: null, success: false }
);

<form action={action}>
  <input name="email" type="email" />
  {state.error && <p className="error">{state.error}</p>}
  <button disabled={isPending}>
    {isPending ? 'Submitting...' : 'Submit'}
  </button>
</form>
```

## useFormStatus

```tsx
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

// Must be inside a form
<form action={submitAction}>
  <input name="email" />
  <SubmitButton />
</form>
```

## useOptimistic

```tsx
function TodoList({ todos }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (state, newTodo) => [...state, { ...newTodo, pending: true }]
  );

  async function addTodo(formData: FormData) {
    const text = formData.get('text') as string;
    addOptimistic({ id: Date.now(), text });
    await saveTodo(text);
  }

  return (
    <form action={addTodo}>
      <input name="text" />
      <button type="submit">Add</button>
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id} className={todo.pending ? 'opacity-50' : ''}>
            {todo.text}
          </li>
        ))}
      </ul>
    </form>
  );
}
```

## React Hook Form

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Min 8 characters'),
});

type FormData = z.infer<typeof schema>;

function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data: FormData) => {
    await login(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}

      <input type="password" {...register('password')} />
      {errors.password && <span>{errors.password.message}</span>}

      <button disabled={isSubmitting}>
        {isSubmitting ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
}
```

## Dynamic Fields

```tsx
import { useFieldArray } from 'react-hook-form';

function OrderForm() {
  const { control, register, handleSubmit } = useForm({
    defaultValues: { items: [{ name: '', quantity: 1 }] },
  });

  const { fields, append, remove } = useFieldArray({
    control,
    name: 'items',
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {fields.map((field, index) => (
        <div key={field.id}>
          <input {...register(`items.${index}.name`)} />
          <input type="number" {...register(`items.${index}.quantity`)} />
          <button type="button" onClick={() => remove(index)}>Remove</button>
        </div>
      ))}
      <button type="button" onClick={() => append({ name: '', quantity: 1 })}>
        Add Item
      </button>
      <button type="submit">Submit</button>
    </form>
  );
}
```

## File Upload

```tsx
function FileUploadForm() {
  const [file, setFile] = useState<File | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!file) return;

    const formData = new FormData();
    formData.append('file', file);
    await uploadFile(formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="file"
        onChange={e => setFile(e.target.files?.[0] || null)}
        accept="image/*"
      />
      {file && <p>Selected: {file.name}</p>}
      <button type="submit" disabled={!file}>Upload</button>
    </form>
  );
}
```

## Custom Validation

```tsx
const { register } = useForm();

<input
  {...register('username', {
    required: 'Username is required',
    minLength: { value: 3, message: 'Min 3 characters' },
    maxLength: { value: 20, message: 'Max 20 characters' },
    pattern: { value: /^[a-z0-9]+$/, message: 'Lowercase alphanumeric only' },
    validate: {
      notAdmin: v => v !== 'admin' || 'Cannot be admin',
      unique: async v => (await checkUsername(v)) || 'Already taken',
    },
  })}
/>
```

## Form State Comparison

| Feature | Controlled | Uncontrolled | React Hook Form |
|---------|------------|--------------|-----------------|
| Re-renders | Every change | None | Minimal |
| Validation | Manual | Manual | Built-in |
| Complexity | Medium | Simple | Low |
| Best for | Simple forms | Quick forms | Complex forms |
