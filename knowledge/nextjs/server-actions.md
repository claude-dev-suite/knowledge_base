# Next.js Server Actions

Server Actions are asynchronous functions that run on the server. They can be called from both Server and Client Components to handle form submissions, data mutations, and other server-side operations in Next.js applications.

## Table of Contents

1. [Server Action Basics](#server-action-basics)
2. [Form Handling](#form-handling)
3. [Validation with Zod](#validation-with-zod)
4. [useFormState Hook](#useformstate-hook)
5. [useFormStatus Hook](#useformstatus-hook)
6. [useOptimistic Hook](#useoptimistic-hook)
7. [Programmatic Invocation](#programmatic-invocation)
8. [Redirects and Revalidation](#redirects-and-revalidation)
9. [Error Handling](#error-handling)
10. [Return Values and State](#return-values-and-state)
11. [File Uploads](#file-uploads)
12. [Authentication in Server Actions](#authentication-in-server-actions)
13. [Rate Limiting](#rate-limiting)
14. [Server Actions with Prisma](#server-actions-with-prisma)
15. [Testing Server Actions](#testing-server-actions)
16. [Security Considerations](#security-considerations)
17. [Best Practices](#best-practices)

---

## Server Action Basics

### The 'use server' Directive

Server Actions are defined using the `'use server'` directive. You can place it at the top of an async function to mark it as a Server Action, or at the top of a separate file to mark all exports as Server Actions.

#### File-Level Declaration

```tsx
// app/actions.ts
'use server';

// All exports from this file are Server Actions
export async function createUser(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  // Server-side logic here
  await db.user.create({ data: { name, email } });
}

export async function deleteUser(userId: string) {
  await db.user.delete({ where: { id: userId } });
}

export async function updateUser(userId: string, data: Partial<User>) {
  await db.user.update({ where: { id: userId }, data });
}
```

#### Inline Declaration in Server Components

```tsx
// app/page.tsx (Server Component)
export default function Page() {
  // Inline Server Action
  async function handleSubmit(formData: FormData) {
    'use server';

    const title = formData.get('title') as string;
    await db.post.create({ data: { title } });
  }

  return (
    <form action={handleSubmit}>
      <input name="title" required />
      <button type="submit">Create</button>
    </form>
  );
}
```

### Importing Server Actions in Client Components

```tsx
// app/components/CreatePostForm.tsx
'use client';

import { createPost } from '@/app/actions';

export function CreatePostForm() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Post title" required />
      <textarea name="content" placeholder="Content" required />
      <button type="submit">Create Post</button>
    </form>
  );
}
```

### Server Action Arguments

Server Actions can receive various types of arguments:

```tsx
'use server';

// FormData argument (from forms)
export async function handleFormSubmit(formData: FormData) {
  const data = Object.fromEntries(formData);
  // Process data...
}

// Bound arguments using .bind()
export async function updateItem(itemId: string, formData: FormData) {
  const name = formData.get('name') as string;
  await db.item.update({ where: { id: itemId }, data: { name } });
}

// Multiple arguments (when called programmatically)
export async function processData(userId: string, action: string, payload: object) {
  // Process with multiple args...
}
```

### Binding Arguments

```tsx
// app/components/EditForm.tsx
'use client';

import { updateItem } from '@/app/actions';

export function EditForm({ itemId }: { itemId: string }) {
  // Bind the itemId to the action
  const updateItemWithId = updateItem.bind(null, itemId);

  return (
    <form action={updateItemWithId}>
      <input name="name" required />
      <button type="submit">Update</button>
    </form>
  );
}
```

---

## Form Handling

### Basic Form with Server Action

```tsx
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;
  const category = formData.get('category') as string;
  const tags = formData.getAll('tags') as string[];

  await db.post.create({
    data: {
      title,
      content,
      category,
      tags,
    },
  });

  revalidatePath('/posts');
}

// app/posts/new/page.tsx
import { createPost } from '@/app/actions';

export default function NewPostPage() {
  return (
    <form action={createPost} className="space-y-4">
      <div>
        <label htmlFor="title">Title</label>
        <input
          type="text"
          id="title"
          name="title"
          required
          minLength={3}
          maxLength={100}
        />
      </div>

      <div>
        <label htmlFor="content">Content</label>
        <textarea
          id="content"
          name="content"
          required
          rows={10}
        />
      </div>

      <div>
        <label htmlFor="category">Category</label>
        <select id="category" name="category" required>
          <option value="">Select category</option>
          <option value="tech">Technology</option>
          <option value="lifestyle">Lifestyle</option>
          <option value="business">Business</option>
        </select>
      </div>

      <div>
        <label>Tags</label>
        <label>
          <input type="checkbox" name="tags" value="featured" /> Featured
        </label>
        <label>
          <input type="checkbox" name="tags" value="trending" /> Trending
        </label>
      </div>

      <button type="submit">Create Post</button>
    </form>
  );
}
```

### Handling Multiple Forms

```tsx
// app/actions.ts
'use server';

export async function createComment(postId: string, formData: FormData) {
  const content = formData.get('content') as string;

  await db.comment.create({
    data: {
      content,
      postId,
    },
  });

  revalidatePath(`/posts/${postId}`);
}

export async function likePost(postId: string) {
  await db.post.update({
    where: { id: postId },
    data: { likes: { increment: 1 } },
  });

  revalidatePath(`/posts/${postId}`);
}

// app/posts/[id]/page.tsx
import { createComment, likePost } from '@/app/actions';

export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await db.post.findUnique({ where: { id: params.id } });

  const createCommentWithId = createComment.bind(null, params.id);
  const likePostWithId = likePost.bind(null, params.id);

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>

      {/* Like Form */}
      <form action={likePostWithId}>
        <button type="submit">Like ({post.likes})</button>
      </form>

      {/* Comment Form */}
      <form action={createCommentWithId}>
        <textarea name="content" required />
        <button type="submit">Add Comment</button>
      </form>
    </article>
  );
}
```

### Hidden Form Fields

```tsx
// Using hidden fields to pass additional data
export default function ProductForm({ productId, userId }: Props) {
  return (
    <form action={updateProduct}>
      <input type="hidden" name="productId" value={productId} />
      <input type="hidden" name="userId" value={userId} />
      <input type="hidden" name="timestamp" value={Date.now().toString()} />

      <input name="name" required />
      <input name="price" type="number" step="0.01" required />

      <button type="submit">Update Product</button>
    </form>
  );
}
```

---

## Validation with Zod

### Basic Schema Validation

```tsx
// app/lib/schemas.ts
import { z } from 'zod';

export const CreatePostSchema = z.object({
  title: z
    .string()
    .min(3, 'Title must be at least 3 characters')
    .max(100, 'Title must be less than 100 characters'),
  content: z
    .string()
    .min(10, 'Content must be at least 10 characters')
    .max(10000, 'Content must be less than 10,000 characters'),
  category: z.enum(['tech', 'lifestyle', 'business'], {
    errorMap: () => ({ message: 'Please select a valid category' }),
  }),
  published: z.coerce.boolean().default(false),
});

export const UpdateUserSchema = z.object({
  name: z.string().min(2).max(50).optional(),
  email: z.string().email('Invalid email address').optional(),
  bio: z.string().max(500).optional(),
  website: z.string().url('Invalid URL').optional().or(z.literal('')),
});

export type CreatePostInput = z.infer<typeof CreatePostSchema>;
export type UpdateUserInput = z.infer<typeof UpdateUserSchema>;
```

### Server Action with Validation

```tsx
// app/actions.ts
'use server';

import { z } from 'zod';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { CreatePostSchema } from '@/app/lib/schemas';

export type FormState = {
  success: boolean;
  message: string;
  errors?: {
    title?: string[];
    content?: string[];
    category?: string[];
    _form?: string[];
  };
  data?: any;
};

export async function createPost(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  // Parse and validate the form data
  const validatedFields = CreatePostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
    category: formData.get('category'),
    published: formData.get('published'),
  });

  // Return early if validation fails
  if (!validatedFields.success) {
    return {
      success: false,
      message: 'Validation failed',
      errors: validatedFields.error.flatten().fieldErrors,
    };
  }

  try {
    const post = await db.post.create({
      data: validatedFields.data,
    });

    revalidatePath('/posts');

    return {
      success: true,
      message: 'Post created successfully',
      data: { id: post.id },
    };
  } catch (error) {
    return {
      success: false,
      message: 'Failed to create post',
      errors: {
        _form: ['An unexpected error occurred. Please try again.'],
      },
    };
  }
}
```

### Complex Validation Schemas

```tsx
// app/lib/schemas.ts
import { z } from 'zod';

// Schema with refinements
export const RegistrationSchema = z
  .object({
    username: z
      .string()
      .min(3)
      .max(20)
      .regex(/^[a-zA-Z0-9_]+$/, 'Username can only contain letters, numbers, and underscores'),
    email: z.string().email(),
    password: z
      .string()
      .min(8, 'Password must be at least 8 characters')
      .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
      .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
      .regex(/[0-9]/, 'Password must contain at least one number'),
    confirmPassword: z.string(),
    terms: z.literal(true, {
      errorMap: () => ({ message: 'You must accept the terms and conditions' }),
    }),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: 'Passwords do not match',
    path: ['confirmPassword'],
  });

// Schema with async validation
export const CreateUserSchema = z.object({
  email: z.string().email(),
  username: z.string().min(3).max(20),
});

// Async validation in server action
export async function createUser(prevState: FormState, formData: FormData) {
  const parsed = CreateUserSchema.safeParse({
    email: formData.get('email'),
    username: formData.get('username'),
  });

  if (!parsed.success) {
    return {
      success: false,
      errors: parsed.error.flatten().fieldErrors,
    };
  }

  // Check for existing user
  const existingUser = await db.user.findFirst({
    where: {
      OR: [
        { email: parsed.data.email },
        { username: parsed.data.username },
      ],
    },
  });

  if (existingUser) {
    return {
      success: false,
      errors: {
        _form: ['A user with this email or username already exists'],
      },
    };
  }

  // Create user...
}
```

### Transforming and Coercing Data

```tsx
import { z } from 'zod';

export const ProductSchema = z.object({
  name: z.string().min(1).transform((val) => val.trim()),
  price: z.coerce.number().positive().multipleOf(0.01),
  quantity: z.coerce.number().int().min(0).default(0),
  sku: z.string().toUpperCase(),
  tags: z
    .string()
    .transform((val) => val.split(',').map((t) => t.trim()).filter(Boolean)),
  releaseDate: z.coerce.date(),
  discount: z.coerce.number().min(0).max(100).optional(),
});

// Server action
export async function createProduct(prevState: FormState, formData: FormData) {
  const validatedFields = ProductSchema.safeParse({
    name: formData.get('name'),
    price: formData.get('price'),
    quantity: formData.get('quantity'),
    sku: formData.get('sku'),
    tags: formData.get('tags'), // "tag1, tag2, tag3" -> ["tag1", "tag2", "tag3"]
    releaseDate: formData.get('releaseDate'),
    discount: formData.get('discount') || undefined,
  });

  if (!validatedFields.success) {
    return {
      success: false,
      errors: validatedFields.error.flatten().fieldErrors,
    };
  }

  // validatedFields.data now has properly typed and transformed values
  await db.product.create({ data: validatedFields.data });
}
```

---

## useFormState Hook

Note: In React 19, `useFormState` has been renamed to `useActionState`. Both work similarly.

### Basic useFormState Usage

```tsx
// app/actions.ts
'use server';

export type ActionState = {
  success: boolean;
  message: string;
  errors?: Record<string, string[]>;
};

const initialState: ActionState = {
  success: false,
  message: '',
};

export async function submitContact(
  prevState: ActionState,
  formData: FormData
): Promise<ActionState> {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  const message = formData.get('message') as string;

  // Validation
  const errors: Record<string, string[]> = {};

  if (!name || name.length < 2) {
    errors.name = ['Name must be at least 2 characters'];
  }
  if (!email || !email.includes('@')) {
    errors.email = ['Please enter a valid email'];
  }
  if (!message || message.length < 10) {
    errors.message = ['Message must be at least 10 characters'];
  }

  if (Object.keys(errors).length > 0) {
    return { success: false, message: 'Validation failed', errors };
  }

  // Process submission
  await sendEmail({ name, email, message });

  return {
    success: true,
    message: 'Thank you for your message! We will get back to you soon.',
  };
}

// app/contact/ContactForm.tsx
'use client';

import { useFormState } from 'react-dom';
import { submitContact, type ActionState } from '@/app/actions';

const initialState: ActionState = {
  success: false,
  message: '',
};

export function ContactForm() {
  const [state, formAction] = useFormState(submitContact, initialState);

  return (
    <form action={formAction} className="space-y-4">
      <div>
        <label htmlFor="name">Name</label>
        <input
          type="text"
          id="name"
          name="name"
          aria-describedby="name-error"
        />
        {state.errors?.name && (
          <p id="name-error" className="text-red-500">
            {state.errors.name.join(', ')}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input
          type="email"
          id="email"
          name="email"
          aria-describedby="email-error"
        />
        {state.errors?.email && (
          <p id="email-error" className="text-red-500">
            {state.errors.email.join(', ')}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="message">Message</label>
        <textarea
          id="message"
          name="message"
          rows={5}
          aria-describedby="message-error"
        />
        {state.errors?.message && (
          <p id="message-error" className="text-red-500">
            {state.errors.message.join(', ')}
          </p>
        )}
      </div>

      <SubmitButton />

      {state.message && (
        <div
          className={state.success ? 'text-green-500' : 'text-red-500'}
          role="alert"
        >
          {state.message}
        </div>
      )}
    </form>
  );
}
```

### useActionState (React 19)

```tsx
'use client';

import { useActionState } from 'react';
import { createTodo } from '@/app/actions';

export function TodoForm() {
  const [state, formAction, isPending] = useActionState(createTodo, {
    success: false,
    message: '',
  });

  return (
    <form action={formAction}>
      <input name="title" disabled={isPending} />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Adding...' : 'Add Todo'}
      </button>
      {state.message && <p>{state.message}</p>}
    </form>
  );
}
```

---

## useFormStatus Hook

### Basic Usage

```tsx
'use client';

import { useFormStatus } from 'react-dom';

export function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

// Usage in a form
export function MyForm() {
  return (
    <form action={serverAction}>
      <input name="field" />
      <SubmitButton />
    </form>
  );
}
```

### All useFormStatus Properties

```tsx
'use client';

import { useFormStatus } from 'react-dom';

export function FormStatusDebug() {
  const { pending, data, method, action } = useFormStatus();

  return (
    <div className="text-sm text-gray-500">
      <p>Pending: {pending ? 'Yes' : 'No'}</p>
      <p>Method: {method || 'N/A'}</p>
      <p>Action: {action?.toString() || 'N/A'}</p>
      {data && (
        <p>Data keys: {Array.from(data.keys()).join(', ')}</p>
      )}
    </div>
  );
}
```

### Advanced Submit Button

```tsx
'use client';

import { useFormStatus } from 'react-dom';
import { Loader2 } from 'lucide-react';

interface SubmitButtonProps {
  children: React.ReactNode;
  loadingText?: string;
  className?: string;
}

export function SubmitButton({
  children,
  loadingText = 'Processing...',
  className = '',
}: SubmitButtonProps) {
  const { pending } = useFormStatus();

  return (
    <button
      type="submit"
      disabled={pending}
      className={`
        inline-flex items-center justify-center gap-2
        px-4 py-2 rounded-md font-medium
        bg-blue-600 text-white
        hover:bg-blue-700
        disabled:opacity-50 disabled:cursor-not-allowed
        transition-colors
        ${className}
      `}
      aria-disabled={pending}
    >
      {pending && <Loader2 className="w-4 h-4 animate-spin" />}
      {pending ? loadingText : children}
    </button>
  );
}

// Variants
export function DeleteButton() {
  const { pending } = useFormStatus();

  return (
    <button
      type="submit"
      disabled={pending}
      className="bg-red-600 hover:bg-red-700 text-white px-4 py-2 rounded"
    >
      {pending ? 'Deleting...' : 'Delete'}
    </button>
  );
}

export function SaveButton() {
  const { pending } = useFormStatus();

  return (
    <button
      type="submit"
      disabled={pending}
      className="bg-green-600 hover:bg-green-700 text-white px-4 py-2 rounded"
    >
      {pending ? 'Saving...' : 'Save Changes'}
    </button>
  );
}
```

### Form with Loading States

```tsx
'use client';

import { useFormStatus } from 'react-dom';
import { useFormState } from 'react-dom';

function FormFields() {
  const { pending } = useFormStatus();

  return (
    <fieldset disabled={pending} className="space-y-4">
      <div>
        <label htmlFor="title">Title</label>
        <input
          type="text"
          id="title"
          name="title"
          className="disabled:bg-gray-100 disabled:cursor-not-allowed"
        />
      </div>
      <div>
        <label htmlFor="content">Content</label>
        <textarea
          id="content"
          name="content"
          className="disabled:bg-gray-100 disabled:cursor-not-allowed"
        />
      </div>
    </fieldset>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Creating...' : 'Create Post'}
    </button>
  );
}

export function CreatePostForm() {
  const [state, formAction] = useFormState(createPost, initialState);

  return (
    <form action={formAction}>
      <FormFields />
      <SubmitButton />
      {state.errors && <ErrorDisplay errors={state.errors} />}
    </form>
  );
}
```

---

## useOptimistic Hook

### Basic Optimistic Updates

```tsx
'use client';

import { useOptimistic } from 'react';
import { addTodo, deleteTodo, toggleTodo } from '@/app/actions';

type Todo = {
  id: string;
  title: string;
  completed: boolean;
  pending?: boolean;
};

export function TodoList({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, updateOptimisticTodos] = useOptimistic(
    todos,
    (state: Todo[], update: { action: string; todo: Partial<Todo> }) => {
      switch (update.action) {
        case 'add':
          return [...state, { ...update.todo, pending: true } as Todo];
        case 'delete':
          return state.filter((t) => t.id !== update.todo.id);
        case 'toggle':
          return state.map((t) =>
            t.id === update.todo.id
              ? { ...t, completed: !t.completed, pending: true }
              : t
          );
        default:
          return state;
      }
    }
  );

  async function handleAdd(formData: FormData) {
    const title = formData.get('title') as string;
    const tempId = `temp-${Date.now()}`;

    updateOptimisticTodos({
      action: 'add',
      todo: { id: tempId, title, completed: false },
    });

    await addTodo(formData);
  }

  async function handleDelete(id: string) {
    updateOptimisticTodos({ action: 'delete', todo: { id } });
    await deleteTodo(id);
  }

  async function handleToggle(id: string) {
    updateOptimisticTodos({ action: 'toggle', todo: { id } });
    await toggleTodo(id);
  }

  return (
    <div>
      <form action={handleAdd}>
        <input name="title" placeholder="New todo" required />
        <button type="submit">Add</button>
      </form>

      <ul>
        {optimisticTodos.map((todo) => (
          <li
            key={todo.id}
            style={{ opacity: todo.pending ? 0.5 : 1 }}
            className="flex items-center gap-2"
          >
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => handleToggle(todo.id)}
            />
            <span className={todo.completed ? 'line-through' : ''}>
              {todo.title}
            </span>
            <button onClick={() => handleDelete(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Optimistic Like Button

```tsx
'use client';

import { useOptimistic, useTransition } from 'react';
import { likePost, unlikePost } from '@/app/actions';
import { Heart, HeartOff } from 'lucide-react';

type LikeState = {
  count: number;
  isLiked: boolean;
};

export function LikeButton({
  postId,
  initialState,
}: {
  postId: string;
  initialState: LikeState;
}) {
  const [isPending, startTransition] = useTransition();

  const [optimisticState, setOptimisticState] = useOptimistic(
    initialState,
    (state: LikeState, optimisticValue: boolean) => ({
      count: optimisticValue ? state.count + 1 : state.count - 1,
      isLiked: optimisticValue,
    })
  );

  function handleClick() {
    startTransition(async () => {
      setOptimisticState(!optimisticState.isLiked);

      if (optimisticState.isLiked) {
        await unlikePost(postId);
      } else {
        await likePost(postId);
      }
    });
  }

  return (
    <button
      onClick={handleClick}
      disabled={isPending}
      className="flex items-center gap-2"
    >
      {optimisticState.isLiked ? (
        <Heart className="fill-red-500 text-red-500" />
      ) : (
        <HeartOff />
      )}
      <span>{optimisticState.count}</span>
    </button>
  );
}
```

### Optimistic Comments

```tsx
'use client';

import { useOptimistic, useRef } from 'react';
import { addComment } from '@/app/actions';

type Comment = {
  id: string;
  content: string;
  author: string;
  createdAt: Date;
  pending?: boolean;
};

export function CommentSection({
  postId,
  comments,
  currentUser,
}: {
  postId: string;
  comments: Comment[];
  currentUser: { name: string };
}) {
  const formRef = useRef<HTMLFormElement>(null);

  const [optimisticComments, addOptimisticComment] = useOptimistic(
    comments,
    (state: Comment[], newComment: Comment) => [newComment, ...state]
  );

  async function handleSubmit(formData: FormData) {
    const content = formData.get('content') as string;

    // Add optimistic comment
    addOptimisticComment({
      id: `temp-${Date.now()}`,
      content,
      author: currentUser.name,
      createdAt: new Date(),
      pending: true,
    });

    // Clear form
    formRef.current?.reset();

    // Submit to server
    await addComment(postId, formData);
  }

  return (
    <div className="space-y-4">
      <form ref={formRef} action={handleSubmit} className="space-y-2">
        <textarea
          name="content"
          placeholder="Write a comment..."
          required
          className="w-full p-2 border rounded"
        />
        <button
          type="submit"
          className="px-4 py-2 bg-blue-600 text-white rounded"
        >
          Post Comment
        </button>
      </form>

      <div className="space-y-3">
        {optimisticComments.map((comment) => (
          <div
            key={comment.id}
            className={`p-3 border rounded ${
              comment.pending ? 'opacity-50' : ''
            }`}
          >
            <div className="font-semibold">{comment.author}</div>
            <p>{comment.content}</p>
            <time className="text-sm text-gray-500">
              {comment.pending
                ? 'Posting...'
                : new Date(comment.createdAt).toLocaleDateString()}
            </time>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## Programmatic Invocation

### Using useTransition

```tsx
'use client';

import { useTransition } from 'react';
import { deletePost, publishPost, archivePost } from '@/app/actions';

export function PostActions({ postId }: { postId: string }) {
  const [isPending, startTransition] = useTransition();

  const handleDelete = () => {
    if (!confirm('Are you sure you want to delete this post?')) return;

    startTransition(async () => {
      await deletePost(postId);
    });
  };

  const handlePublish = () => {
    startTransition(async () => {
      await publishPost(postId);
    });
  };

  const handleArchive = () => {
    startTransition(async () => {
      await archivePost(postId);
    });
  };

  return (
    <div className="flex gap-2">
      <button
        onClick={handlePublish}
        disabled={isPending}
        className="bg-green-600 text-white px-4 py-2 rounded"
      >
        {isPending ? 'Processing...' : 'Publish'}
      </button>
      <button
        onClick={handleArchive}
        disabled={isPending}
        className="bg-yellow-600 text-white px-4 py-2 rounded"
      >
        {isPending ? 'Processing...' : 'Archive'}
      </button>
      <button
        onClick={handleDelete}
        disabled={isPending}
        className="bg-red-600 text-white px-4 py-2 rounded"
      >
        {isPending ? 'Processing...' : 'Delete'}
      </button>
    </div>
  );
}
```

### Calling Server Actions with Arguments

```tsx
'use client';

import { useTransition, useState } from 'react';
import { updateQuantity } from '@/app/actions';

export function QuantitySelector({
  itemId,
  initialQuantity,
}: {
  itemId: string;
  initialQuantity: number;
}) {
  const [quantity, setQuantity] = useState(initialQuantity);
  const [isPending, startTransition] = useTransition();

  const updateQty = (newQty: number) => {
    if (newQty < 1) return;

    setQuantity(newQty); // Optimistic update

    startTransition(async () => {
      const result = await updateQuantity(itemId, newQty);
      if (!result.success) {
        setQuantity(initialQuantity); // Revert on error
      }
    });
  };

  return (
    <div className="flex items-center gap-2">
      <button
        onClick={() => updateQty(quantity - 1)}
        disabled={isPending || quantity <= 1}
        className="px-3 py-1 border rounded"
      >
        -
      </button>
      <span className={isPending ? 'opacity-50' : ''}>{quantity}</span>
      <button
        onClick={() => updateQty(quantity + 1)}
        disabled={isPending}
        className="px-3 py-1 border rounded"
      >
        +
      </button>
    </div>
  );
}
```

### Event Handlers with Server Actions

```tsx
'use client';

import { useTransition } from 'react';
import { trackEvent, updatePreferences } from '@/app/actions';

export function SettingsPanel({ userId }: { userId: string }) {
  const [isPending, startTransition] = useTransition();

  // Track user interactions
  const handleInteraction = (eventName: string) => {
    startTransition(async () => {
      await trackEvent(userId, eventName, { timestamp: Date.now() });
    });
  };

  // Toggle settings
  const handleToggle = (setting: string, value: boolean) => {
    startTransition(async () => {
      await updatePreferences(userId, { [setting]: value });
    });
  };

  return (
    <div onMouseEnter={() => handleInteraction('settings_view')}>
      <h2>Settings</h2>
      <label className="flex items-center gap-2">
        <input
          type="checkbox"
          onChange={(e) => handleToggle('emailNotifications', e.target.checked)}
          disabled={isPending}
        />
        Email Notifications
      </label>
      <label className="flex items-center gap-2">
        <input
          type="checkbox"
          onChange={(e) => handleToggle('darkMode', e.target.checked)}
          disabled={isPending}
        />
        Dark Mode
      </label>
    </div>
  );
}
```

---

## Redirects and Revalidation

### Revalidating Data

```tsx
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';
import { redirect } from 'next/navigation';

// Revalidate a specific path
export async function updatePost(id: string, formData: FormData) {
  await db.post.update({
    where: { id },
    data: {
      title: formData.get('title') as string,
      content: formData.get('content') as string,
    },
  });

  // Revalidate the specific post page
  revalidatePath(`/posts/${id}`);

  // Revalidate the posts list
  revalidatePath('/posts');
}

// Revalidate by tag
export async function createComment(postId: string, formData: FormData) {
  await db.comment.create({
    data: {
      postId,
      content: formData.get('content') as string,
    },
  });

  // Revalidate all data tagged with 'comments'
  revalidateTag('comments');

  // Revalidate specific post's comments
  revalidateTag(`post-${postId}-comments`);
}

// Revalidate layout
export async function updateSiteSettings(formData: FormData) {
  await db.settings.update({
    where: { id: 'site' },
    data: {
      siteName: formData.get('siteName') as string,
    },
  });

  // Revalidate the layout (use 'layout' type)
  revalidatePath('/', 'layout');
}
```

### Using Redirect

```tsx
'use server';

import { redirect } from 'next/navigation';
import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  const post = await db.post.create({
    data: {
      title: formData.get('title') as string,
      content: formData.get('content') as string,
    },
  });

  revalidatePath('/posts');

  // Redirect to the new post - this throws internally
  redirect(`/posts/${post.id}`);
}

// Conditional redirect
export async function processCheckout(formData: FormData) {
  const result = await processPayment(formData);

  if (result.success) {
    revalidatePath('/orders');
    redirect(`/orders/${result.orderId}/confirmation`);
  } else {
    // Return error state instead of redirecting
    return {
      success: false,
      error: result.error,
    };
  }
}

// Redirect with authentication check
export async function updateProfile(formData: FormData) {
  const session = await getSession();

  if (!session) {
    redirect('/login?callbackUrl=/profile');
  }

  await db.user.update({
    where: { id: session.user.id },
    data: {
      name: formData.get('name') as string,
    },
  });

  revalidatePath('/profile');
  redirect('/profile?updated=true');
}
```

### Revalidation Patterns

```tsx
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';

// Pattern 1: Revalidate multiple related paths
export async function deletePost(id: string) {
  const post = await db.post.delete({ where: { id } });

  // Revalidate all affected paths
  revalidatePath('/posts');
  revalidatePath(`/posts/${id}`);
  revalidatePath(`/users/${post.authorId}/posts`);
  revalidatePath('/admin/posts');
}

// Pattern 2: Revalidate by tag for related data
export async function updateCategory(id: string, formData: FormData) {
  await db.category.update({
    where: { id },
    data: { name: formData.get('name') as string },
  });

  // Revalidate all posts in this category
  revalidateTag(`category-${id}`);
  revalidateTag('categories');
}

// Pattern 3: On-demand revalidation
export async function revalidateContent(secret: string, path: string) {
  if (secret !== process.env.REVALIDATION_SECRET) {
    return { success: false, error: 'Invalid secret' };
  }

  revalidatePath(path);
  return { success: true, revalidated: path };
}
```

---

## Error Handling

### Comprehensive Error Handling

```tsx
'use server';

import { z } from 'zod';

// Custom error types
class ValidationError extends Error {
  constructor(public errors: Record<string, string[]>) {
    super('Validation failed');
    this.name = 'ValidationError';
  }
}

class NotFoundError extends Error {
  constructor(resource: string) {
    super(`${resource} not found`);
    this.name = 'NotFoundError';
  }
}

class UnauthorizedError extends Error {
  constructor() {
    super('Unauthorized');
    this.name = 'UnauthorizedError';
  }
}

// Server action with comprehensive error handling
export async function updatePost(
  id: string,
  prevState: ActionState,
  formData: FormData
): Promise<ActionState> {
  try {
    // Auth check
    const session = await getSession();
    if (!session) {
      throw new UnauthorizedError();
    }

    // Validation
    const schema = z.object({
      title: z.string().min(3).max(100),
      content: z.string().min(10),
    });

    const parsed = schema.safeParse({
      title: formData.get('title'),
      content: formData.get('content'),
    });

    if (!parsed.success) {
      throw new ValidationError(parsed.error.flatten().fieldErrors);
    }

    // Check existence
    const post = await db.post.findUnique({ where: { id } });
    if (!post) {
      throw new NotFoundError('Post');
    }

    // Authorization
    if (post.authorId !== session.user.id) {
      throw new UnauthorizedError();
    }

    // Update
    await db.post.update({
      where: { id },
      data: parsed.data,
    });

    revalidatePath(`/posts/${id}`);

    return {
      success: true,
      message: 'Post updated successfully',
    };
  } catch (error) {
    if (error instanceof ValidationError) {
      return {
        success: false,
        message: 'Validation failed',
        errors: error.errors,
      };
    }

    if (error instanceof NotFoundError) {
      return {
        success: false,
        message: error.message,
      };
    }

    if (error instanceof UnauthorizedError) {
      return {
        success: false,
        message: 'You do not have permission to perform this action',
      };
    }

    // Log unexpected errors
    console.error('Unexpected error in updatePost:', error);

    return {
      success: false,
      message: 'An unexpected error occurred. Please try again.',
    };
  }
}
```

### Error Boundaries with Server Actions

```tsx
// app/posts/[id]/edit/error.tsx
'use client';

import { useEffect } from 'react';

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Log error to error reporting service
    console.error('Edit post error:', error);
  }, [error]);

  return (
    <div className="p-4 bg-red-50 border border-red-200 rounded">
      <h2 className="text-lg font-semibold text-red-800">
        Something went wrong!
      </h2>
      <p className="text-red-600">{error.message}</p>
      <button
        onClick={reset}
        className="mt-4 px-4 py-2 bg-red-600 text-white rounded"
      >
        Try again
      </button>
    </div>
  );
}
```

### Try-Catch in Client Components

```tsx
'use client';

import { useState, useTransition } from 'react';
import { riskyAction } from '@/app/actions';

export function RiskyButton() {
  const [error, setError] = useState<string | null>(null);
  const [isPending, startTransition] = useTransition();

  const handleClick = () => {
    setError(null);

    startTransition(async () => {
      try {
        const result = await riskyAction();
        if (!result.success) {
          setError(result.error);
        }
      } catch (e) {
        setError('An unexpected error occurred');
      }
    });
  };

  return (
    <div>
      <button onClick={handleClick} disabled={isPending}>
        {isPending ? 'Processing...' : 'Do Something Risky'}
      </button>
      {error && (
        <p className="text-red-500 mt-2">{error}</p>
      )}
    </div>
  );
}
```

---

## Return Values and State

### Structured Return Types

```tsx
'use server';

// Type definitions for return values
type ActionResult<T = void> =
  | { success: true; data: T; message?: string }
  | { success: false; error: string; errors?: Record<string, string[]> };

// Generic success/failure returns
export async function createUser(
  formData: FormData
): Promise<ActionResult<{ id: string; email: string }>> {
  try {
    const user = await db.user.create({
      data: {
        email: formData.get('email') as string,
        name: formData.get('name') as string,
      },
    });

    return {
      success: true,
      data: { id: user.id, email: user.email },
      message: 'User created successfully',
    };
  } catch (error) {
    return {
      success: false,
      error: 'Failed to create user',
    };
  }
}

// Multiple data types
export async function fetchDashboardData(): Promise<
  ActionResult<{
    stats: { posts: number; users: number; comments: number };
    recentPosts: Post[];
    topUsers: User[];
  }>
> {
  try {
    const [stats, recentPosts, topUsers] = await Promise.all([
      getStats(),
      getRecentPosts(),
      getTopUsers(),
    ]);

    return {
      success: true,
      data: { stats, recentPosts, topUsers },
    };
  } catch (error) {
    return {
      success: false,
      error: 'Failed to fetch dashboard data',
    };
  }
}
```

### State Management Patterns

```tsx
// app/lib/action-state.ts
export type FormState<T = void> = {
  success: boolean;
  message: string;
  errors?: Record<string, string[]>;
  data?: T;
  timestamp?: number;
};

export function createInitialState<T = void>(): FormState<T> {
  return {
    success: false,
    message: '',
  };
}

export function successState<T>(data?: T, message = 'Success'): FormState<T> {
  return {
    success: true,
    message,
    data,
    timestamp: Date.now(),
  };
}

export function errorState(
  message: string,
  errors?: Record<string, string[]>
): FormState {
  return {
    success: false,
    message,
    errors,
    timestamp: Date.now(),
  };
}

// Usage in server action
'use server';

import { successState, errorState, FormState } from '@/app/lib/action-state';

export async function createPost(
  prevState: FormState<{ id: string }>,
  formData: FormData
): Promise<FormState<{ id: string }>> {
  // Validation...

  try {
    const post = await db.post.create({ data: { ... } });
    revalidatePath('/posts');
    return successState({ id: post.id }, 'Post created!');
  } catch (error) {
    return errorState('Failed to create post');
  }
}
```

### Handling Return Values in Components

```tsx
'use client';

import { useFormState } from 'react-dom';
import { useEffect, useRef } from 'react';
import { createPost, FormState } from '@/app/actions';
import { toast } from 'sonner';

export function CreatePostForm() {
  const formRef = useRef<HTMLFormElement>(null);
  const [state, formAction] = useFormState(createPost, {
    success: false,
    message: '',
  });

  // React to state changes
  useEffect(() => {
    if (state.timestamp) {
      if (state.success) {
        toast.success(state.message);
        formRef.current?.reset();
      } else {
        toast.error(state.message);
      }
    }
  }, [state]);

  return (
    <form ref={formRef} action={formAction}>
      {/* Form fields... */}
    </form>
  );
}
```

---

## File Uploads

### Basic File Upload

```tsx
// app/actions.ts
'use server';

import { writeFile } from 'fs/promises';
import path from 'path';

export async function uploadFile(formData: FormData) {
  const file = formData.get('file') as File;

  if (!file || file.size === 0) {
    return { success: false, error: 'No file provided' };
  }

  // Validate file type
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp'];
  if (!allowedTypes.includes(file.type)) {
    return { success: false, error: 'Invalid file type' };
  }

  // Validate file size (5MB max)
  const maxSize = 5 * 1024 * 1024;
  if (file.size > maxSize) {
    return { success: false, error: 'File too large (max 5MB)' };
  }

  // Generate unique filename
  const ext = path.extname(file.name);
  const filename = `${Date.now()}-${Math.random().toString(36).substr(2, 9)}${ext}`;
  const filepath = path.join(process.cwd(), 'public/uploads', filename);

  // Save file
  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);
  await writeFile(filepath, buffer);

  return {
    success: true,
    data: {
      filename,
      url: `/uploads/${filename}`,
      size: file.size,
    },
  };
}

// app/upload/UploadForm.tsx
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { uploadFile } from '@/app/actions';
import { useState } from 'react';

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Uploading...' : 'Upload'}
    </button>
  );
}

export function UploadForm() {
  const [preview, setPreview] = useState<string | null>(null);
  const [state, formAction] = useFormState(uploadFile, {
    success: false,
    message: '',
  });

  const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (file) {
      const reader = new FileReader();
      reader.onloadend = () => setPreview(reader.result as string);
      reader.readAsDataURL(file);
    }
  };

  return (
    <form action={formAction} className="space-y-4">
      <div>
        <input
          type="file"
          name="file"
          accept="image/*"
          onChange={handleFileChange}
          required
        />
      </div>

      {preview && (
        <img src={preview} alt="Preview" className="max-w-xs rounded" />
      )}

      <SubmitButton />

      {state.success && state.data && (
        <div className="text-green-500">
          <p>File uploaded: {state.data.filename}</p>
          <img src={state.data.url} alt="Uploaded" className="max-w-xs" />
        </div>
      )}

      {!state.success && state.error && (
        <p className="text-red-500">{state.error}</p>
      )}
    </form>
  );
}
```

### Multiple File Uploads

```tsx
'use server';

export async function uploadMultiple(formData: FormData) {
  const files = formData.getAll('files') as File[];

  if (files.length === 0) {
    return { success: false, error: 'No files provided' };
  }

  const results = [];
  const errors = [];

  for (const file of files) {
    try {
      // Validate each file
      if (file.size > 5 * 1024 * 1024) {
        errors.push(`${file.name}: File too large`);
        continue;
      }

      const filename = `${Date.now()}-${file.name}`;
      const filepath = path.join(process.cwd(), 'public/uploads', filename);

      const bytes = await file.arrayBuffer();
      await writeFile(filepath, Buffer.from(bytes));

      results.push({
        original: file.name,
        filename,
        url: `/uploads/${filename}`,
      });
    } catch (error) {
      errors.push(`${file.name}: Upload failed`);
    }
  }

  return {
    success: errors.length === 0,
    data: { uploaded: results, errors },
  };
}

// Client component
'use client';

export function MultiUploadForm() {
  return (
    <form action={uploadMultiple}>
      <input
        type="file"
        name="files"
        multiple
        accept="image/*,.pdf,.doc,.docx"
      />
      <button type="submit">Upload All</button>
    </form>
  );
}
```

### Cloud Storage Upload (S3/Cloudinary)

```tsx
'use server';

import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

const s3 = new S3Client({
  region: process.env.AWS_REGION!,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
  },
});

export async function uploadToS3(formData: FormData) {
  const file = formData.get('file') as File;

  if (!file) {
    return { success: false, error: 'No file provided' };
  }

  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);

  const key = `uploads/${Date.now()}-${file.name}`;

  try {
    await s3.send(
      new PutObjectCommand({
        Bucket: process.env.AWS_BUCKET_NAME!,
        Key: key,
        Body: buffer,
        ContentType: file.type,
      })
    );

    return {
      success: true,
      data: {
        url: `https://${process.env.AWS_BUCKET_NAME}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`,
        key,
      },
    };
  } catch (error) {
    console.error('S3 upload error:', error);
    return { success: false, error: 'Upload failed' };
  }
}
```

---

## Authentication in Server Actions

### Session-Based Authentication

```tsx
'use server';

import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import { verifySession } from '@/app/lib/session';

// Helper to get authenticated user
async function getAuthUser() {
  const sessionId = cookies().get('session')?.value;

  if (!sessionId) {
    return null;
  }

  const session = await verifySession(sessionId);
  return session?.user || null;
}

// Protected server action
export async function createPost(formData: FormData) {
  const user = await getAuthUser();

  if (!user) {
    redirect('/login');
  }

  const post = await db.post.create({
    data: {
      title: formData.get('title') as string,
      content: formData.get('content') as string,
      authorId: user.id,
    },
  });

  revalidatePath('/posts');
  redirect(`/posts/${post.id}`);
}

// Action with role-based access
export async function deleteUser(userId: string) {
  const user = await getAuthUser();

  if (!user) {
    return { success: false, error: 'Unauthorized' };
  }

  if (user.role !== 'admin') {
    return { success: false, error: 'Forbidden: Admin access required' };
  }

  await db.user.delete({ where: { id: userId } });
  revalidatePath('/admin/users');

  return { success: true };
}
```

### NextAuth.js Integration

```tsx
'use server';

import { getServerSession } from 'next-auth';
import { authOptions } from '@/app/api/auth/[...nextauth]/route';

export async function protectedAction(formData: FormData) {
  const session = await getServerSession(authOptions);

  if (!session?.user) {
    return { success: false, error: 'Not authenticated' };
  }

  // Use session.user.id, session.user.email, etc.
  const result = await db.item.create({
    data: {
      name: formData.get('name') as string,
      userId: session.user.id,
    },
  });

  return { success: true, data: result };
}

// With custom session data
export async function adminAction(formData: FormData) {
  const session = await getServerSession(authOptions);

  if (!session?.user) {
    return { success: false, error: 'Not authenticated' };
  }

  // Access custom session fields
  if ((session.user as any).role !== 'admin') {
    return { success: false, error: 'Admin access required' };
  }

  // Admin-only logic...
}
```

### Authentication Wrapper

```tsx
// app/lib/auth-action.ts
'use server';

import { getServerSession } from 'next-auth';
import { authOptions } from '@/app/api/auth/[...nextauth]/route';

type AuthenticatedAction<T> = (
  user: { id: string; email: string; role: string },
  ...args: any[]
) => Promise<T>;

export function withAuth<T>(action: AuthenticatedAction<T>) {
  return async (...args: any[]): Promise<T | { success: false; error: string }> => {
    const session = await getServerSession(authOptions);

    if (!session?.user) {
      return { success: false, error: 'Authentication required' };
    }

    return action(session.user as any, ...args);
  };
}

export function withRole<T>(role: string, action: AuthenticatedAction<T>) {
  return async (...args: any[]): Promise<T | { success: false; error: string }> => {
    const session = await getServerSession(authOptions);

    if (!session?.user) {
      return { success: false, error: 'Authentication required' };
    }

    if ((session.user as any).role !== role) {
      return { success: false, error: `${role} access required` };
    }

    return action(session.user as any, ...args);
  };
}

// Usage
export const createPost = withAuth(async (user, formData: FormData) => {
  const post = await db.post.create({
    data: {
      title: formData.get('title') as string,
      authorId: user.id,
    },
  });

  return { success: true, data: post };
});

export const deleteAnyPost = withRole('admin', async (user, postId: string) => {
  await db.post.delete({ where: { id: postId } });
  return { success: true };
});
```

---

## Rate Limiting

### Simple In-Memory Rate Limiting

```tsx
// app/lib/rate-limit.ts
const rateLimitMap = new Map<string, { count: number; lastReset: number }>();

export function rateLimit(
  key: string,
  limit: number,
  windowMs: number
): { success: boolean; remaining: number; resetAt: number } {
  const now = Date.now();
  const windowStart = now - windowMs;

  const current = rateLimitMap.get(key);

  if (!current || current.lastReset < windowStart) {
    rateLimitMap.set(key, { count: 1, lastReset: now });
    return { success: true, remaining: limit - 1, resetAt: now + windowMs };
  }

  if (current.count >= limit) {
    return {
      success: false,
      remaining: 0,
      resetAt: current.lastReset + windowMs,
    };
  }

  current.count++;
  return {
    success: true,
    remaining: limit - current.count,
    resetAt: current.lastReset + windowMs,
  };
}

// app/actions.ts
'use server';

import { headers } from 'next/headers';
import { rateLimit } from '@/app/lib/rate-limit';

export async function submitForm(formData: FormData) {
  const ip = headers().get('x-forwarded-for') || 'anonymous';

  const { success, remaining } = rateLimit(
    `submit:${ip}`,
    10, // 10 requests
    60 * 1000 // per minute
  );

  if (!success) {
    return {
      success: false,
      error: 'Too many requests. Please try again later.',
    };
  }

  // Process form...
  return { success: true, remaining };
}
```

### Redis-Based Rate Limiting

```tsx
// app/lib/rate-limit-redis.ts
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

export async function rateLimitRedis(
  key: string,
  limit: number,
  windowSeconds: number
): Promise<{ success: boolean; remaining: number; resetAt: number }> {
  const now = Math.floor(Date.now() / 1000);
  const windowKey = `ratelimit:${key}:${Math.floor(now / windowSeconds)}`;

  const multi = redis.multi();
  multi.incr(windowKey);
  multi.expire(windowKey, windowSeconds);

  const results = await multi.exec();
  const count = results?.[0]?.[1] as number;

  const remaining = Math.max(0, limit - count);
  const resetAt = (Math.floor(now / windowSeconds) + 1) * windowSeconds * 1000;

  return {
    success: count <= limit,
    remaining,
    resetAt,
  };
}

// app/actions.ts
'use server';

import { headers } from 'next/headers';
import { rateLimitRedis } from '@/app/lib/rate-limit-redis';

export async function apiAction(formData: FormData) {
  const ip = headers().get('x-forwarded-for') || 'anonymous';

  const { success, remaining, resetAt } = await rateLimitRedis(
    `api:${ip}`,
    100, // 100 requests
    3600 // per hour
  );

  if (!success) {
    const retryAfter = Math.ceil((resetAt - Date.now()) / 1000);
    return {
      success: false,
      error: `Rate limit exceeded. Try again in ${retryAfter} seconds.`,
    };
  }

  // Process action...
  return { success: true };
}
```

### User-Specific Rate Limiting

```tsx
'use server';

import { getServerSession } from 'next-auth';
import { rateLimitRedis } from '@/app/lib/rate-limit-redis';

export async function userAction(formData: FormData) {
  const session = await getServerSession(authOptions);

  if (!session?.user) {
    return { success: false, error: 'Not authenticated' };
  }

  // Different limits for different user tiers
  const limits: Record<string, { requests: number; window: number }> = {
    free: { requests: 10, window: 3600 },
    pro: { requests: 100, window: 3600 },
    enterprise: { requests: 1000, window: 3600 },
  };

  const userTier = (session.user as any).tier || 'free';
  const { requests, window } = limits[userTier];

  const { success, remaining } = await rateLimitRedis(
    `user:${session.user.id}`,
    requests,
    window
  );

  if (!success) {
    return {
      success: false,
      error: 'Rate limit exceeded for your plan.',
      upgrade: userTier === 'free',
    };
  }

  // Process action...
}
```

---

## Server Actions with Prisma

### CRUD Operations

```tsx
// app/actions/posts.ts
'use server';

import { prisma } from '@/app/lib/prisma';
import { revalidatePath } from 'next/cache';
import { z } from 'zod';

const PostSchema = z.object({
  title: z.string().min(3).max(100),
  content: z.string().min(10),
  published: z.boolean().default(false),
  categoryId: z.string().optional(),
  tags: z.array(z.string()).optional(),
});

// Create
export async function createPost(prevState: FormState, formData: FormData) {
  const validated = PostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
    published: formData.get('published') === 'true',
    categoryId: formData.get('categoryId') || undefined,
    tags: formData.getAll('tags'),
  });

  if (!validated.success) {
    return { success: false, errors: validated.error.flatten().fieldErrors };
  }

  const { tags, ...postData } = validated.data;

  const post = await prisma.post.create({
    data: {
      ...postData,
      authorId: 'current-user-id', // Get from session
      tags: tags?.length
        ? {
            connectOrCreate: tags.map((tag) => ({
              where: { name: tag },
              create: { name: tag },
            })),
          }
        : undefined,
    },
    include: { tags: true },
  });

  revalidatePath('/posts');
  return { success: true, data: post };
}

// Read
export async function getPosts(page = 1, limit = 10) {
  const [posts, total] = await prisma.$transaction([
    prisma.post.findMany({
      where: { published: true },
      include: {
        author: { select: { name: true, image: true } },
        tags: true,
        _count: { select: { comments: true } },
      },
      orderBy: { createdAt: 'desc' },
      skip: (page - 1) * limit,
      take: limit,
    }),
    prisma.post.count({ where: { published: true } }),
  ]);

  return {
    posts,
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  };
}

// Update
export async function updatePost(
  id: string,
  prevState: FormState,
  formData: FormData
) {
  const validated = PostSchema.partial().safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
    published: formData.get('published') === 'true',
  });

  if (!validated.success) {
    return { success: false, errors: validated.error.flatten().fieldErrors };
  }

  const post = await prisma.post.update({
    where: { id },
    data: validated.data,
  });

  revalidatePath('/posts');
  revalidatePath(`/posts/${id}`);
  return { success: true, data: post };
}

// Delete
export async function deletePost(id: string) {
  await prisma.post.delete({ where: { id } });
  revalidatePath('/posts');
  return { success: true };
}
```

### Complex Queries

```tsx
'use server';

import { prisma } from '@/app/lib/prisma';

// Search with filters
export async function searchPosts(formData: FormData) {
  const query = formData.get('query') as string;
  const category = formData.get('category') as string;
  const tags = formData.getAll('tags') as string[];
  const sortBy = formData.get('sortBy') as string;

  const posts = await prisma.post.findMany({
    where: {
      AND: [
        { published: true },
        query
          ? {
              OR: [
                { title: { contains: query, mode: 'insensitive' } },
                { content: { contains: query, mode: 'insensitive' } },
              ],
            }
          : {},
        category ? { categoryId: category } : {},
        tags.length > 0
          ? { tags: { some: { name: { in: tags } } } }
          : {},
      ],
    },
    include: {
      author: { select: { name: true, image: true } },
      tags: true,
      category: true,
    },
    orderBy: sortBy === 'popular'
      ? { views: 'desc' }
      : sortBy === 'comments'
      ? { comments: { _count: 'desc' } }
      : { createdAt: 'desc' },
  });

  return { success: true, data: posts };
}

// Aggregations
export async function getPostStats(authorId: string) {
  const stats = await prisma.post.aggregate({
    where: { authorId },
    _count: { id: true },
    _sum: { views: true },
    _avg: { views: true },
  });

  const byCategory = await prisma.post.groupBy({
    by: ['categoryId'],
    where: { authorId },
    _count: { id: true },
  });

  return {
    totalPosts: stats._count.id,
    totalViews: stats._sum.views || 0,
    avgViews: Math.round(stats._avg.views || 0),
    byCategory,
  };
}

// Transactions
export async function transferPost(postId: string, newAuthorId: string) {
  return prisma.$transaction(async (tx) => {
    const post = await tx.post.findUnique({
      where: { id: postId },
      include: { author: true },
    });

    if (!post) {
      throw new Error('Post not found');
    }

    // Update post author
    await tx.post.update({
      where: { id: postId },
      data: { authorId: newAuthorId },
    });

    // Create transfer record
    await tx.postTransfer.create({
      data: {
        postId,
        fromUserId: post.authorId,
        toUserId: newAuthorId,
      },
    });

    // Notify both users
    await tx.notification.createMany({
      data: [
        {
          userId: post.authorId,
          type: 'POST_TRANSFERRED_FROM',
          data: { postId, toUserId: newAuthorId },
        },
        {
          userId: newAuthorId,
          type: 'POST_TRANSFERRED_TO',
          data: { postId, fromUserId: post.authorId },
        },
      ],
    });

    return { success: true };
  });
}
```

---

## Testing Server Actions

### Unit Testing with Vitest

```tsx
// __tests__/actions/posts.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { createPost, updatePost, deletePost } from '@/app/actions/posts';
import { prisma } from '@/app/lib/prisma';

// Mock Prisma
vi.mock('@/app/lib/prisma', () => ({
  prisma: {
    post: {
      create: vi.fn(),
      update: vi.fn(),
      delete: vi.fn(),
    },
  },
}));

// Mock Next.js functions
vi.mock('next/cache', () => ({
  revalidatePath: vi.fn(),
}));

describe('Post Actions', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  describe('createPost', () => {
    it('should create a post with valid data', async () => {
      const mockPost = {
        id: '1',
        title: 'Test Post',
        content: 'Test content here',
      };

      (prisma.post.create as any).mockResolvedValue(mockPost);

      const formData = new FormData();
      formData.append('title', 'Test Post');
      formData.append('content', 'Test content here');

      const result = await createPost({ success: false, message: '' }, formData);

      expect(result.success).toBe(true);
      expect(result.data).toEqual(mockPost);
      expect(prisma.post.create).toHaveBeenCalled();
    });

    it('should return validation errors for invalid data', async () => {
      const formData = new FormData();
      formData.append('title', 'AB'); // Too short
      formData.append('content', 'Short'); // Too short

      const result = await createPost({ success: false, message: '' }, formData);

      expect(result.success).toBe(false);
      expect(result.errors).toBeDefined();
      expect(prisma.post.create).not.toHaveBeenCalled();
    });
  });

  describe('deletePost', () => {
    it('should delete a post', async () => {
      (prisma.post.delete as any).mockResolvedValue({ id: '1' });

      const result = await deletePost('1');

      expect(result.success).toBe(true);
      expect(prisma.post.delete).toHaveBeenCalledWith({
        where: { id: '1' },
      });
    });
  });
});
```

### Integration Testing

```tsx
// __tests__/integration/posts.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { prisma } from '@/app/lib/prisma';
import { createPost, getPosts, deletePost } from '@/app/actions/posts';

describe('Post Actions Integration', () => {
  let testUserId: string;

  beforeAll(async () => {
    // Create test user
    const user = await prisma.user.create({
      data: {
        email: 'test@example.com',
        name: 'Test User',
      },
    });
    testUserId = user.id;
  });

  afterAll(async () => {
    // Cleanup
    await prisma.post.deleteMany({ where: { authorId: testUserId } });
    await prisma.user.delete({ where: { id: testUserId } });
    await prisma.$disconnect();
  });

  beforeEach(async () => {
    // Clear posts before each test
    await prisma.post.deleteMany({ where: { authorId: testUserId } });
  });

  it('should create and retrieve a post', async () => {
    const formData = new FormData();
    formData.append('title', 'Integration Test Post');
    formData.append('content', 'This is content for integration testing');
    formData.append('published', 'true');

    // Create post
    const createResult = await createPost(
      { success: false, message: '' },
      formData
    );
    expect(createResult.success).toBe(true);

    // Retrieve posts
    const { posts } = await getPosts(1, 10);
    expect(posts.length).toBeGreaterThan(0);
    expect(posts[0].title).toBe('Integration Test Post');
  });
});
```

### Testing with React Testing Library

```tsx
// __tests__/components/PostForm.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { PostForm } from '@/app/components/PostForm';

// Mock server action
vi.mock('@/app/actions/posts', () => ({
  createPost: vi.fn().mockResolvedValue({ success: true }),
}));

describe('PostForm', () => {
  it('should submit form and show success message', async () => {
    const user = userEvent.setup();
    render(<PostForm />);

    await user.type(screen.getByLabelText(/title/i), 'My New Post');
    await user.type(
      screen.getByLabelText(/content/i),
      'This is the content of my new post'
    );
    await user.click(screen.getByRole('button', { name: /create/i }));

    await waitFor(() => {
      expect(screen.getByText(/success/i)).toBeInTheDocument();
    });
  });

  it('should display validation errors', async () => {
    const { createPost } = await import('@/app/actions/posts');
    (createPost as any).mockResolvedValue({
      success: false,
      errors: {
        title: ['Title is required'],
      },
    });

    const user = userEvent.setup();
    render(<PostForm />);

    await user.click(screen.getByRole('button', { name: /create/i }));

    await waitFor(() => {
      expect(screen.getByText(/title is required/i)).toBeInTheDocument();
    });
  });
});
```

---

## Security Considerations

### Input Sanitization

```tsx
'use server';

import DOMPurify from 'isomorphic-dompurify';
import { z } from 'zod';

// Sanitize HTML content
export async function createContent(formData: FormData) {
  const rawContent = formData.get('content') as string;

  // Sanitize HTML to prevent XSS
  const sanitizedContent = DOMPurify.sanitize(rawContent, {
    ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'a', 'ul', 'ol', 'li'],
    ALLOWED_ATTR: ['href', 'target', 'rel'],
  });

  await db.content.create({
    data: { content: sanitizedContent },
  });
}

// SQL injection prevention (Prisma handles this, but be careful with raw queries)
export async function searchUsers(formData: FormData) {
  const query = formData.get('query') as string;

  // Safe - Prisma parameterizes the query
  const users = await prisma.user.findMany({
    where: {
      name: { contains: query, mode: 'insensitive' },
    },
  });

  // If you must use raw SQL, use parameterized queries
  const rawUsers = await prisma.$queryRaw`
    SELECT * FROM users WHERE name ILIKE ${`%${query}%`}
  `;

  return users;
}
```

### CSRF Protection

Server Actions in Next.js have built-in CSRF protection. The framework:

1. Generates a unique action ID for each server action
2. Validates the action ID on the server
3. Checks the Origin header to prevent cross-origin requests

```tsx
// Additional CSRF token validation if needed
'use server';

import { cookies } from 'next/headers';
import crypto from 'crypto';

export async function sensitiveAction(formData: FormData) {
  const csrfToken = formData.get('csrf_token') as string;
  const storedToken = cookies().get('csrf_token')?.value;

  if (!csrfToken || csrfToken !== storedToken) {
    return { success: false, error: 'Invalid CSRF token' };
  }

  // Proceed with action...
}

// Generate CSRF token
export async function generateCsrfToken() {
  const token = crypto.randomBytes(32).toString('hex');
  cookies().set('csrf_token', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
  });
  return token;
}
```

### Authorization Checks

```tsx
'use server';

import { getServerSession } from 'next-auth';

// Always verify ownership
export async function updatePost(postId: string, formData: FormData) {
  const session = await getServerSession(authOptions);

  if (!session?.user) {
    return { success: false, error: 'Not authenticated' };
  }

  // Verify ownership
  const post = await prisma.post.findUnique({
    where: { id: postId },
    select: { authorId: true },
  });

  if (!post) {
    return { success: false, error: 'Post not found' };
  }

  if (post.authorId !== session.user.id) {
    // Log potential security incident
    console.warn(`Unauthorized access attempt: User ${session.user.id} tried to update post ${postId}`);
    return { success: false, error: 'Not authorized' };
  }

  // Proceed with update...
}

// Role-based access control
export async function adminAction(formData: FormData) {
  const session = await getServerSession(authOptions);

  if (!session?.user) {
    return { success: false, error: 'Not authenticated' };
  }

  // Check permissions from database
  const user = await prisma.user.findUnique({
    where: { id: session.user.id },
    include: {
      roles: {
        include: {
          permissions: true,
        },
      },
    },
  });

  const hasPermission = user?.roles.some((role) =>
    role.permissions.some((p) => p.name === 'admin.write')
  );

  if (!hasPermission) {
    return { success: false, error: 'Insufficient permissions' };
  }

  // Proceed with admin action...
}
```

### Data Exposure Prevention

```tsx
'use server';

// Don't expose sensitive data in return values
export async function getUser(userId: string) {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    select: {
      id: true,
      name: true,
      image: true,
      // Don't include: password, email (if private), tokens, etc.
    },
  });

  return user;
}

// Use DTOs for complex objects
type PublicUserDTO = {
  id: string;
  name: string;
  avatar: string | null;
  postCount: number;
};

export async function getUserProfile(userId: string): Promise<PublicUserDTO | null> {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    include: {
      _count: { select: { posts: true } },
    },
  });

  if (!user) return null;

  // Return only public data
  return {
    id: user.id,
    name: user.name,
    avatar: user.image,
    postCount: user._count.posts,
  };
}
```

---

## Best Practices

### 1. Always Validate Input

```tsx
'use server';

import { z } from 'zod';

// Define schemas outside the action
const CreatePostSchema = z.object({
  title: z.string().min(3).max(100),
  content: z.string().min(10).max(10000),
});

export async function createPost(formData: FormData) {
  // Always validate
  const result = CreatePostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  });

  if (!result.success) {
    return { success: false, errors: result.error.flatten().fieldErrors };
  }

  // Use validated data
  await db.post.create({ data: result.data });
}
```

### 2. Handle Errors Gracefully

```tsx
'use server';

export async function riskyAction(formData: FormData) {
  try {
    // Your logic here
    return { success: true, data: result };
  } catch (error) {
    // Log for debugging
    console.error('Action failed:', error);

    // Return user-friendly error
    return {
      success: false,
      error: 'Something went wrong. Please try again.',
    };
  }
}
```

### 3. Use Revalidation Strategically

```tsx
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';

export async function updatePost(id: string, formData: FormData) {
  await db.post.update({ where: { id }, data: { ... } });

  // Revalidate specific paths
  revalidatePath(`/posts/${id}`);

  // Don't over-revalidate - be specific
  // revalidatePath('/'); // Avoid unless necessary
}
```

### 4. Keep Actions Focused

```tsx
// Good - Single responsibility
export async function createPost(formData: FormData) { /* ... */ }
export async function updatePost(id: string, formData: FormData) { /* ... */ }
export async function deletePost(id: string) { /* ... */ }

// Avoid - Multiple responsibilities
export async function handlePost(action: string, id?: string, formData?: FormData) {
  // Don't do this
}
```

### 5. Type Your Return Values

```tsx
'use server';

type ActionResult<T> =
  | { success: true; data: T }
  | { success: false; error: string; errors?: Record<string, string[]> };

export async function createUser(formData: FormData): Promise<ActionResult<User>> {
  // Explicit return type helps catch errors
}
```

### 6. Use Optimistic Updates for Better UX

```tsx
'use client';

import { useOptimistic, useTransition } from 'react';

export function LikeButton({ postId, likes }) {
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    likes,
    (state, newLike) => state + 1
  );

  return (
    <button onClick={() => {
      addOptimisticLike(1);
      likePost(postId);
    }}>
      {optimisticLikes} Likes
    </button>
  );
}
```

### 7. Implement Proper Loading States

```tsx
'use client';

import { useFormStatus } from 'react-dom';

function SubmitButton({ children }) {
  const { pending } = useFormStatus();

  return (
    <button disabled={pending}>
      {pending ? 'Loading...' : children}
    </button>
  );
}
```

### 8. Organize Actions by Feature

```
app/
  actions/
    posts.ts      # Post-related actions
    users.ts      # User-related actions
    comments.ts   # Comment-related actions
    auth.ts       # Authentication actions
    index.ts      # Re-export all
```

### 9. Use Middleware for Common Concerns

```tsx
// app/lib/action-middleware.ts
export function withValidation<T>(schema: z.ZodSchema<T>) {
  return (action: (data: T) => Promise<any>) => {
    return async (formData: FormData) => {
      const parsed = schema.safeParse(Object.fromEntries(formData));
      if (!parsed.success) {
        return { success: false, errors: parsed.error.flatten().fieldErrors };
      }
      return action(parsed.data);
    };
  };
}
```

### 10. Document Your Actions

```tsx
/**
 * Creates a new blog post
 *
 * @param prevState - Previous form state (for useFormState)
 * @param formData - Form data containing title, content, and category
 * @returns ActionResult with created post or validation errors
 *
 * @example
 * const [state, formAction] = useFormState(createPost, initialState);
 */
export async function createPost(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  // Implementation...
}
```

---

## Additional Resources

- [Next.js Server Actions Documentation](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)
- [React Server Actions RFC](https://github.com/reactjs/rfcs/blob/main/text/0000-server-actions.md)
- [Zod Documentation](https://zod.dev/)
- [React useFormStatus](https://react.dev/reference/react-dom/hooks/useFormStatus)
- [React useOptimistic](https://react.dev/reference/react/useOptimistic)
