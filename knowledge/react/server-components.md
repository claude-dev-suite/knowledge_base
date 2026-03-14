# React Server Components (RSC)

> Official Documentation: https://react.dev/reference/rsc/server-components

## Overview

React Server Components (RSC) represent a fundamental shift in React architecture, enabling components to render exclusively on the server. This paradigm allows direct access to backend resources, reduces client-side JavaScript bundles, and improves initial page load performance while maintaining React's component-based development model.

Server Components are the default in frameworks like Next.js App Router, while Client Components require explicit opt-in via the `'use client'` directive.

---

## 1. Server Components vs Client Components

### Server Components

Server Components render on the server and send their output (React Server Component Payload) to the client. They never re-render on the client.

```tsx
// Server Component (default - no directive needed)
// app/users/page.tsx
import { db } from '@/lib/database';

async function UsersPage() {
  // Direct database access - no API layer needed
  const users = await db.user.findMany({
    select: { id: true, name: true, email: true }
  });

  return (
    <main>
      <h1>Users</h1>
      <ul>
        {users.map(user => (
          <li key={user.id}>
            {user.name} - {user.email}
          </li>
        ))}
      </ul>
    </main>
  );
}

export default UsersPage;
```

**Server Component Capabilities:**
- Direct database queries
- File system access
- Access to environment variables and secrets
- Async/await at the component level
- Zero impact on client bundle size
- Access to server-only APIs and libraries

### Client Components

Client Components render on the server for initial HTML, then hydrate and become interactive on the client.

```tsx
// Client Component - requires directive
'use client';

import { useState, useEffect } from 'react';

interface User {
  id: string;
  name: string;
  isOnline: boolean;
}

function UserStatus({ userId }: { userId: string }) {
  const [status, setStatus] = useState<'online' | 'offline'>('offline');
  const [lastSeen, setLastSeen] = useState<Date | null>(null);

  useEffect(() => {
    const ws = new WebSocket(`wss://api.example.com/status/${userId}`);

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setStatus(data.status);
      setLastSeen(new Date(data.lastSeen));
    };

    return () => ws.close();
  }, [userId]);

  return (
    <div className="flex items-center gap-2">
      <span className={status === 'online' ? 'bg-green-500' : 'bg-gray-400'} />
      <span>{status === 'online' ? 'Online' : `Last seen ${lastSeen?.toLocaleString()}`}</span>
    </div>
  );
}

export default UserStatus;
```

**Client Component Capabilities:**
- useState, useReducer, useEffect, and other hooks
- Event handlers (onClick, onChange, etc.)
- Browser APIs (localStorage, geolocation, etc.)
- Custom hooks with state
- Third-party libraries requiring browser context

---

## 2. Directives: "use client" and "use server"

### The "use client" Directive

Marks a file as a Client Component entry point. All imports become part of the client bundle.

```tsx
'use client';  // Must be at the very top of the file

// Everything in this file and its imports runs on the client
import { useState } from 'react';
import { motion } from 'framer-motion';  // Animation library

export function AnimatedCounter() {
  const [count, setCount] = useState(0);

  return (
    <motion.button
      whileHover={{ scale: 1.1 }}
      whileTap={{ scale: 0.9 }}
      onClick={() => setCount(c => c + 1)}
    >
      Count: {count}
    </motion.button>
  );
}
```

**Important Rules:**
- Must be the first statement (before imports)
- Only needed at the boundary - child components inherit client context
- Cannot import Server Components into Client Components
- Can pass Server Components as children or props

### The "use server" Directive

Marks functions as Server Actions that can be called from Client Components.

```tsx
// app/actions/user-actions.ts
'use server';

import { db } from '@/lib/database';
import { revalidatePath } from 'next/cache';
import { z } from 'zod';

const UserSchema = z.object({
  name: z.string().min(2).max(50),
  email: z.string().email(),
});

export async function createUser(formData: FormData) {
  const rawData = {
    name: formData.get('name'),
    email: formData.get('email'),
  };

  // Validate input
  const validated = UserSchema.safeParse(rawData);
  if (!validated.success) {
    return { error: validated.error.flatten() };
  }

  // Database operation
  const user = await db.user.create({
    data: validated.data,
  });

  revalidatePath('/users');
  return { success: true, user };
}

export async function deleteUser(userId: string) {
  await db.user.delete({ where: { id: userId } });
  revalidatePath('/users');
}
```

**Inline Server Actions:**

```tsx
// Can also be defined inline within Server Components
async function UserForm() {
  async function handleSubmit(formData: FormData) {
    'use server';  // Inline directive

    const name = formData.get('name') as string;
    await db.user.create({ data: { name } });
    revalidatePath('/users');
  }

  return (
    <form action={handleSubmit}>
      <input name="name" required />
      <button type="submit">Create User</button>
    </form>
  );
}
```

---

## 3. When to Use Server vs Client Components

### Decision Tree

```
START: Do you need this functionality?
|
+-- useState, useReducer, useEffect?
|   YES -> Client Component
|
+-- Event listeners (onClick, onChange)?
|   YES -> Client Component
|
+-- Browser APIs (window, localStorage, navigator)?
|   YES -> Client Component
|
+-- Custom hooks with state/effects?
|   YES -> Client Component
|
+-- Third-party library requiring browser?
|   YES -> Client Component
|
+-- Direct data fetching from database?
|   YES -> Server Component
|
+-- Access to sensitive environment variables?
|   YES -> Server Component
|
+-- Large dependencies that shouldn't ship to client?
|   YES -> Server Component
|
+-- Static content with no interactivity?
|   YES -> Server Component
|
DEFAULT -> Server Component (prefer by default)
```

### Comprehensive Comparison Table

| Feature/Need | Server Component | Client Component |
|--------------|------------------|------------------|
| Direct database access | Yes | No |
| File system access | Yes | No |
| Environment secrets | Yes | No (NEXT_PUBLIC_ only) |
| useState/useReducer | No | Yes |
| useEffect/useLayoutEffect | No | Yes |
| Event handlers | No | Yes |
| Browser APIs | No | Yes |
| Async component | Yes | No |
| Bundle size impact | None | Adds to bundle |
| SEO for dynamic content | Excellent | Requires SSR |
| Real-time updates | No (use polling) | Yes (WebSocket, etc.) |
| Form validation (live) | No | Yes |
| Animations | Limited CSS | Full (Framer, etc.) |

### Practical Examples

```tsx
// GOOD: Server Component for data display
async function ProductList() {
  const products = await db.product.findMany();
  return (
    <ul>
      {products.map(p => <ProductCard key={p.id} product={p} />)}
    </ul>
  );
}

// GOOD: Client Component for interactivity
'use client';
function AddToCartButton({ productId }: { productId: string }) {
  const [isPending, setIsPending] = useState(false);

  async function handleClick() {
    setIsPending(true);
    await addToCart(productId);
    setIsPending(false);
  }

  return (
    <button onClick={handleClick} disabled={isPending}>
      {isPending ? 'Adding...' : 'Add to Cart'}
    </button>
  );
}
```

---

## 4. Data Fetching in Server Components

### Basic Data Fetching

```tsx
// Direct async/await in component
async function BlogPost({ slug }: { slug: string }) {
  const post = await db.post.findUnique({
    where: { slug },
    include: { author: true, comments: true },
  });

  if (!post) {
    notFound();
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <p>By {post.author.name}</p>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}
```

### Parallel Data Fetching

```tsx
async function Dashboard() {
  // Initiate all fetches simultaneously
  const userPromise = getUser();
  const statsPromise = getStats();
  const notificationsPromise = getNotifications();
  const activityPromise = getRecentActivity();

  // Wait for all to complete
  const [user, stats, notifications, activity] = await Promise.all([
    userPromise,
    statsPromise,
    notificationsPromise,
    activityPromise,
  ]);

  return (
    <div className="grid grid-cols-2 gap-4">
      <UserProfile user={user} />
      <StatsPanel stats={stats} />
      <NotificationList notifications={notifications} />
      <ActivityFeed activity={activity} />
    </div>
  );
}
```

### Sequential Data Fetching (Dependent Queries)

```tsx
async function UserOrders({ userId }: { userId: string }) {
  // First fetch: get user
  const user = await db.user.findUnique({
    where: { id: userId },
    select: { id: true, membershipTier: true },
  });

  if (!user) {
    notFound();
  }

  // Second fetch: depends on user's membership tier
  const orders = await db.order.findMany({
    where: { userId: user.id },
    take: user.membershipTier === 'premium' ? 100 : 10,
    orderBy: { createdAt: 'desc' },
  });

  return <OrderList orders={orders} />;
}
```

### Fetch with Caching (Next.js)

```tsx
async function CachedData() {
  // Default: cached indefinitely
  const data = await fetch('https://api.example.com/data');

  // Force fresh data
  const freshData = await fetch('https://api.example.com/data', {
    cache: 'no-store',
  });

  // Revalidate after 1 hour
  const timedData = await fetch('https://api.example.com/data', {
    next: { revalidate: 3600 },
  });

  // Tag-based revalidation
  const taggedData = await fetch('https://api.example.com/data', {
    next: { tags: ['products'] },
  });

  return <div>{/* render data */}</div>;
}
```

---

## 5. Passing Data from Server to Client

### Basic Props Pattern

```tsx
// Server Component
async function ProductPage({ productId }: { productId: string }) {
  const product = await db.product.findUnique({
    where: { id: productId },
    include: { reviews: true },
  });

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>

      {/* Pass serializable data to Client Component */}
      <ProductInteractions
        productId={product.id}
        initialStock={product.stock}
        initialPrice={product.price}
      />

      <ReviewSection
        productId={product.id}
        initialReviews={product.reviews}
      />
    </div>
  );
}
```

```tsx
// Client Component
'use client';

import { useState, useOptimistic } from 'react';
import { addReview } from '@/app/actions';

interface Review {
  id: string;
  content: string;
  rating: number;
  createdAt: string;
}

function ReviewSection({
  productId,
  initialReviews,
}: {
  productId: string;
  initialReviews: Review[];
}) {
  const [reviews, setReviews] = useState(initialReviews);
  const [optimisticReviews, addOptimisticReview] = useOptimistic(
    reviews,
    (state, newReview: Review) => [...state, newReview]
  );

  async function handleSubmit(formData: FormData) {
    const tempReview: Review = {
      id: 'temp-' + Date.now(),
      content: formData.get('content') as string,
      rating: Number(formData.get('rating')),
      createdAt: new Date().toISOString(),
    };

    addOptimisticReview(tempReview);
    const result = await addReview(productId, formData);

    if (result.success) {
      setReviews(prev => [...prev, result.review]);
    }
  }

  return (
    <div>
      <ul>
        {optimisticReviews.map(review => (
          <li key={review.id}>{review.content}</li>
        ))}
      </ul>
      <form action={handleSubmit}>
        {/* form fields */}
      </form>
    </div>
  );
}
```

### Serialization Constraints

```tsx
// WRONG: Cannot pass non-serializable data
async function BadExample() {
  const getData = () => fetch('/api/data'); // Function
  const date = new Date(); // Date object
  const map = new Map(); // Map

  return (
    <ClientComponent
      getData={getData}  // ERROR: Functions not serializable
      date={date}        // ERROR: Date not serializable
      map={map}          // ERROR: Map not serializable
    />
  );
}

// CORRECT: Pass serializable equivalents
async function GoodExample() {
  const data = await fetch('/api/data').then(r => r.json());
  const dateString = new Date().toISOString();
  const mapAsObject = Object.fromEntries(someMap);

  return (
    <ClientComponent
      data={data}              // Plain object - OK
      dateString={dateString}  // String - OK
      items={mapAsObject}      // Plain object - OK
    />
  );
}
```

---

## 6. Composition Patterns

### Server Component Wrapping Client Component

```tsx
// Server Component (Layout)
async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const user = await getCurrentUser();
  const permissions = await getPermissions(user.id);

  return (
    <div className="flex">
      {/* Server-rendered sidebar with user data */}
      <Sidebar user={user} permissions={permissions} />

      {/* Client Component for interactive navigation */}
      <NavigationProvider>
        <main className="flex-1">
          {children}
        </main>
      </NavigationProvider>
    </div>
  );
}
```

### Passing Server Components as Children

```tsx
// Client Component that accepts Server Component children
'use client';

import { useState } from 'react';

function Tabs({ children }: { children: React.ReactNode }) {
  const [activeTab, setActiveTab] = useState(0);

  return (
    <div>
      <div className="tab-buttons">
        <button onClick={() => setActiveTab(0)}>Tab 1</button>
        <button onClick={() => setActiveTab(1)}>Tab 2</button>
      </div>
      <div className="tab-content">
        {children}  {/* Server Components can be passed here */}
      </div>
    </div>
  );
}

export default Tabs;
```

```tsx
// Server Component using the Client tabs
async function TabsPage() {
  return (
    <Tabs>
      {/* These Server Components are passed as children */}
      <ServerRenderedContent />
      <AnotherServerComponent />
    </Tabs>
  );
}

async function ServerRenderedContent() {
  const data = await db.content.findMany();
  return <div>{/* render data */}</div>;
}
```

### Slot Pattern for Complex Layouts

```tsx
// Client Component with multiple slots
'use client';

function Modal({
  trigger,
  header,
  content,
  footer,
}: {
  trigger: React.ReactNode;
  header: React.ReactNode;
  content: React.ReactNode;
  footer: React.ReactNode;
}) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <>
      <div onClick={() => setIsOpen(true)}>{trigger}</div>
      {isOpen && (
        <div className="modal-overlay">
          <div className="modal">
            <div className="modal-header">{header}</div>
            <div className="modal-content">{content}</div>
            <div className="modal-footer">{footer}</div>
          </div>
        </div>
      )}
    </>
  );
}
```

```tsx
// Server Component composing with slots
async function ProductModal({ productId }: { productId: string }) {
  const product = await getProduct(productId);

  return (
    <Modal
      trigger={<button>View Details</button>}
      header={<ProductHeader product={product} />}  // Server Component
      content={<ProductDetails product={product} />}  // Server Component
      footer={<AddToCartButton productId={productId} />}  // Client Component
    />
  );
}
```

---

## 7. Server Actions

### Form Handling with Server Actions

```tsx
// app/actions/form-actions.ts
'use server';

import { z } from 'zod';
import { db } from '@/lib/database';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

const ContactSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
  message: z.string().min(10, 'Message must be at least 10 characters'),
});

export type FormState = {
  errors?: {
    name?: string[];
    email?: string[];
    message?: string[];
  };
  message?: string;
  success?: boolean;
};

export async function submitContact(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  const rawData = {
    name: formData.get('name'),
    email: formData.get('email'),
    message: formData.get('message'),
  };

  const validated = ContactSchema.safeParse(rawData);

  if (!validated.success) {
    return {
      errors: validated.error.flatten().fieldErrors,
      message: 'Please fix the errors above.',
    };
  }

  try {
    await db.contact.create({
      data: validated.data,
    });

    revalidatePath('/contact');
    return { success: true, message: 'Message sent successfully!' };
  } catch (error) {
    return { message: 'Failed to send message. Please try again.' };
  }
}
```

### Using Server Actions in Client Components

```tsx
'use client';

import { useActionState } from 'react';
import { submitContact, type FormState } from '@/app/actions/form-actions';

const initialState: FormState = {};

function ContactForm() {
  const [state, formAction, isPending] = useActionState(
    submitContact,
    initialState
  );

  return (
    <form action={formAction}>
      <div>
        <label htmlFor="name">Name</label>
        <input
          id="name"
          name="name"
          required
          aria-describedby="name-error"
        />
        {state.errors?.name && (
          <p id="name-error" className="text-red-500">
            {state.errors.name[0]}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          name="email"
          type="email"
          required
          aria-describedby="email-error"
        />
        {state.errors?.email && (
          <p id="email-error" className="text-red-500">
            {state.errors.email[0]}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="message">Message</label>
        <textarea
          id="message"
          name="message"
          required
          aria-describedby="message-error"
        />
        {state.errors?.message && (
          <p id="message-error" className="text-red-500">
            {state.errors.message[0]}
          </p>
        )}
      </div>

      <button type="submit" disabled={isPending}>
        {isPending ? 'Sending...' : 'Send Message'}
      </button>

      {state.success && (
        <p className="text-green-500">{state.message}</p>
      )}
    </form>
  );
}
```

### Optimistic Updates with Server Actions

```tsx
'use client';

import { useOptimistic, useTransition } from 'react';
import { toggleLike } from '@/app/actions';

interface Post {
  id: string;
  likes: number;
  isLiked: boolean;
}

function LikeButton({ post }: { post: Post }) {
  const [isPending, startTransition] = useTransition();
  const [optimisticPost, setOptimisticPost] = useOptimistic(
    post,
    (currentPost, newLikedState: boolean) => ({
      ...currentPost,
      isLiked: newLikedState,
      likes: newLikedState ? currentPost.likes + 1 : currentPost.likes - 1,
    })
  );

  function handleClick() {
    startTransition(async () => {
      setOptimisticPost(!optimisticPost.isLiked);
      await toggleLike(post.id);
    });
  }

  return (
    <button onClick={handleClick} disabled={isPending}>
      {optimisticPost.isLiked ? 'Unlike' : 'Like'} ({optimisticPost.likes})
    </button>
  );
}
```

---

## 8. Streaming and Suspense

### Basic Streaming Pattern

```tsx
import { Suspense } from 'react';

async function Page() {
  return (
    <div>
      {/* Renders immediately */}
      <Header />

      {/* Streams when ready - independent boundaries */}
      <Suspense fallback={<HeroSkeleton />}>
        <HeroSection />
      </Suspense>

      <div className="grid grid-cols-3 gap-4">
        <Suspense fallback={<ProductsSkeleton />}>
          <FeaturedProducts />
        </Suspense>

        <Suspense fallback={<CategoriesSkeleton />}>
          <Categories />
        </Suspense>

        <Suspense fallback={<RecommendationsSkeleton />}>
          <Recommendations />
        </Suspense>
      </div>

      <Suspense fallback={<ReviewsSkeleton />}>
        <RecentReviews />
      </Suspense>

      {/* Renders immediately */}
      <Footer />
    </div>
  );
}
```

### Nested Suspense Boundaries

```tsx
async function ProductPage({ productId }: { productId: string }) {
  return (
    <div>
      {/* Outer boundary for main content */}
      <Suspense fallback={<ProductPageSkeleton />}>
        <ProductContent productId={productId} />
      </Suspense>
    </div>
  );
}

async function ProductContent({ productId }: { productId: string }) {
  const product = await getProduct(productId);

  return (
    <div>
      <ProductHeader product={product} />
      <ProductGallery images={product.images} />

      {/* Nested boundary - streams independently after parent */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews productId={productId} />
      </Suspense>

      <Suspense fallback={<RelatedSkeleton />}>
        <RelatedProducts categoryId={product.categoryId} />
      </Suspense>
    </div>
  );
}
```

### Loading UI with loading.tsx (Next.js)

```tsx
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div className="animate-pulse">
      <div className="h-8 w-48 bg-gray-200 rounded mb-4" />
      <div className="grid grid-cols-3 gap-4">
        {[...Array(6)].map((_, i) => (
          <div key={i} className="h-32 bg-gray-200 rounded" />
        ))}
      </div>
    </div>
  );
}

// app/dashboard/page.tsx
async function DashboardPage() {
  const data = await fetchDashboardData(); // Slow fetch
  return <Dashboard data={data} />;
}
```

---

## 9. Shared Components

### Creating Isomorphic Components

```tsx
// components/Button.tsx
// Works in both Server and Client Components

interface ButtonProps {
  children: React.ReactNode;
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  className?: string;
  disabled?: boolean;
  type?: 'button' | 'submit' | 'reset';
}

const variantStyles = {
  primary: 'bg-blue-600 text-white hover:bg-blue-700',
  secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
  danger: 'bg-red-600 text-white hover:bg-red-700',
};

const sizeStyles = {
  sm: 'px-2 py-1 text-sm',
  md: 'px-4 py-2 text-base',
  lg: 'px-6 py-3 text-lg',
};

export function Button({
  children,
  variant = 'primary',
  size = 'md',
  className = '',
  disabled = false,
  type = 'button',
}: ButtonProps) {
  return (
    <button
      type={type}
      disabled={disabled}
      className={`
        rounded font-medium transition-colors
        ${variantStyles[variant]}
        ${sizeStyles[size]}
        ${disabled ? 'opacity-50 cursor-not-allowed' : ''}
        ${className}
      `}
    >
      {children}
    </button>
  );
}
```

### Client-Only Wrapper Pattern

```tsx
// components/ClientOnly.tsx
'use client';

import { useEffect, useState, type ReactNode } from 'react';

interface ClientOnlyProps {
  children: ReactNode;
  fallback?: ReactNode;
}

export function ClientOnly({ children, fallback = null }: ClientOnlyProps) {
  const [hasMounted, setHasMounted] = useState(false);

  useEffect(() => {
    setHasMounted(true);
  }, []);

  if (!hasMounted) {
    return fallback;
  }

  return children;
}
```

```tsx
// Usage in Server Component
import { ClientOnly } from '@/components/ClientOnly';
import { BrowserOnlyChart } from '@/components/BrowserOnlyChart';

function AnalyticsPage() {
  return (
    <div>
      <h1>Analytics</h1>
      <ClientOnly fallback={<ChartSkeleton />}>
        <BrowserOnlyChart />
      </ClientOnly>
    </div>
  );
}
```

---

## 10. Third-Party Libraries Handling

### Wrapping Client-Only Libraries

```tsx
// components/Chart.tsx
'use client';

import dynamic from 'next/dynamic';
import { Suspense } from 'react';

// Dynamic import for heavy chart library
const Chart = dynamic(
  () => import('react-chartjs-2').then(mod => mod.Line),
  {
    ssr: false,
    loading: () => <ChartSkeleton />
  }
);

interface ChartWrapperProps {
  data: {
    labels: string[];
    values: number[];
  };
}

export function ChartWrapper({ data }: ChartWrapperProps) {
  const chartData = {
    labels: data.labels,
    datasets: [
      {
        label: 'Values',
        data: data.values,
        borderColor: 'rgb(75, 192, 192)',
        tension: 0.1,
      },
    ],
  };

  return (
    <div className="w-full h-64">
      <Chart data={chartData} />
    </div>
  );
}
```

### Context Providers Pattern

```tsx
// providers/Providers.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ThemeProvider } from 'next-themes';
import { SessionProvider } from 'next-auth/react';
import { useState, type ReactNode } from 'react';

export function Providers({ children }: { children: ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000,
          },
        },
      })
  );

  return (
    <SessionProvider>
      <QueryClientProvider client={queryClient}>
        <ThemeProvider attribute="class" defaultTheme="system">
          {children}
        </ThemeProvider>
      </QueryClientProvider>
    </SessionProvider>
  );
}
```

```tsx
// app/layout.tsx (Server Component)
import { Providers } from '@/providers/Providers';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

---

## 11. Security Considerations

### Server-Only Code Protection

```tsx
// lib/server-only.ts
import 'server-only';  // Throws error if imported in Client Component

import { db } from './database';

export async function getSecretData() {
  // This can never accidentally run on the client
  const secret = process.env.SECRET_KEY;
  return db.secrets.findMany();
}
```

### Input Validation in Server Actions

```tsx
'use server';

import { z } from 'zod';
import { auth } from '@/lib/auth';

const UpdateProfileSchema = z.object({
  name: z.string().min(1).max(100),
  bio: z.string().max(500).optional(),
});

export async function updateProfile(formData: FormData) {
  // 1. Authentication check
  const session = await auth();
  if (!session?.user) {
    throw new Error('Unauthorized');
  }

  // 2. Input validation
  const rawData = {
    name: formData.get('name'),
    bio: formData.get('bio'),
  };

  const validated = UpdateProfileSchema.safeParse(rawData);
  if (!validated.success) {
    return { error: 'Invalid input' };
  }

  // 3. Authorization check
  const userId = session.user.id;

  // 4. Sanitize output if needed
  const sanitizedBio = validated.data.bio?.replace(/<[^>]*>/g, '');

  // 5. Perform operation
  await db.user.update({
    where: { id: userId },
    data: {
      name: validated.data.name,
      bio: sanitizedBio,
    },
  });

  return { success: true };
}
```

### Environment Variable Safety

```tsx
// WRONG: Exposes secret to client
'use client';
function BadComponent() {
  // This would be undefined (good) but shows bad practice
  const secret = process.env.DATABASE_URL;
}

// CORRECT: Public variables only in client
'use client';
function GoodComponent() {
  // Only NEXT_PUBLIC_ variables are available
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;
}

// Server Component - full access
async function ServerComponent() {
  const dbUrl = process.env.DATABASE_URL;  // Safe
  const apiKey = process.env.API_SECRET;   // Safe
}
```

---

## 12. Best Practices and Common Pitfalls

### Best Practices

```tsx
// 1. Default to Server Components
// Only add 'use client' when necessary

// 2. Keep Client Components at the leaves
// BAD: Making entire page a Client Component
'use client';
function BadPage() { /* everything is client */ }

// GOOD: Only interactive parts are Client Components
async function GoodPage() {
  const data = await getData();
  return (
    <div>
      <ServerContent data={data} />
      <InteractiveWidget />  {/* Only this is 'use client' */}
    </div>
  );
}

// 3. Fetch data in Server Components, pass to Client
async function ProductPage({ id }: { id: string }) {
  const product = await getProduct(id);
  return <ProductEditor initialProduct={product} />;
}

// 4. Use Suspense for granular loading states
async function Page() {
  return (
    <>
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>
      <Suspense fallback={<ContentSkeleton />}>
        <Content />
      </Suspense>
    </>
  );
}

// 5. Colocate Server Actions with forms
async function CommentForm({ postId }: { postId: string }) {
  async function addComment(formData: FormData) {
    'use server';
    await db.comment.create({
      data: {
        postId,
        content: formData.get('content') as string,
      },
    });
    revalidatePath(`/posts/${postId}`);
  }

  return (
    <form action={addComment}>
      <textarea name="content" required />
      <button type="submit">Add Comment</button>
    </form>
  );
}
```

### Common Pitfalls to Avoid

```tsx
// PITFALL 1: Importing Server Component into Client Component
// This will NOT work as expected
'use client';
import { ServerComponent } from './ServerComponent';  // Wrong!

// SOLUTION: Pass as children or use composition
function ClientWrapper({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
}

// PITFALL 2: Using hooks in Server Components
async function ServerComponent() {
  const [state, setState] = useState();  // ERROR!
  useEffect(() => {});  // ERROR!
}

// PITFALL 3: Passing non-serializable props
async function BadExample() {
  return (
    <ClientComponent
      onClick={() => {}}  // ERROR: Functions not serializable
      date={new Date()}   // ERROR: Date not serializable
    />
  );
}

// PITFALL 4: Accessing browser APIs in Server Components
async function ServerComponent() {
  const width = window.innerWidth;  // ERROR!
  localStorage.getItem('key');      // ERROR!
}

// PITFALL 5: Not handling async properly
// BAD: Forgetting to await
async function BadAsync() {
  const data = fetchData();  // Missing await!
  return <div>{data}</div>;  // Will show [object Promise]
}

// PITFALL 6: Over-fetching by making everything client-side
'use client';
function BadPattern() {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetch('/api/data').then(r => r.json()).then(setData);
  }, []);
  // This adds unnecessary client JS and API route
}

// BETTER: Server Component
async function GoodPattern() {
  const data = await db.data.findMany();
  return <DataDisplay data={data} />;
}
```

### Performance Optimization Tips

```tsx
// 1. Parallel data fetching
async function OptimizedPage() {
  const [a, b, c] = await Promise.all([
    fetchA(),
    fetchB(),
    fetchC(),
  ]);
  return <Content a={a} b={b} c={c} />;
}

// 2. Streaming with multiple Suspense boundaries
async function StreamedPage() {
  return (
    <>
      <Suspense fallback={<FastSkeleton />}>
        <FastContent />
      </Suspense>
      <Suspense fallback={<SlowSkeleton />}>
        <SlowContent />
      </Suspense>
    </>
  );
}

// 3. Preload data pattern
import { preload } from 'react-dom';

async function Page({ id }: { id: string }) {
  // Start fetching immediately
  const dataPromise = getData(id);

  return (
    <Suspense fallback={<Loading />}>
      <Content dataPromise={dataPromise} />
    </Suspense>
  );
}
```

---

## Quick Reference

| Task | Approach |
|------|----------|
| Database query | Server Component with async/await |
| Form with validation | Server Action + useActionState |
| Interactive UI | Client Component with useState |
| Real-time updates | Client Component with WebSocket |
| Authentication check | Server Component or Server Action |
| File upload | Server Action |
| Third-party animations | Client Component wrapper |
| SEO-critical content | Server Component |
| User preferences | Client Component with localStorage |
| API route replacement | Server Action |

---

## Related Documentation

- [React Server Components RFC](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)
- [Next.js App Router](https://nextjs.org/docs/app)
- [Server Actions](https://react.dev/reference/rsc/server-actions)
- [Suspense for Data Fetching](https://react.dev/reference/react/Suspense)
