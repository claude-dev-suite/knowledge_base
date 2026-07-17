# Next.js Caching

> Official Documentation: https://nextjs.org/docs/app/building-your-application/caching

## Overview

Next.js implements a multi-layered caching architecture to optimize performance and reduce redundant computations. Understanding these layers is crucial for building performant applications and knowing when to opt out of caching.

**The Four Caching Mechanisms:**

| Mechanism | What | Where | Purpose | Duration |
|-----------|------|-------|---------|----------|
| Request Memoization | Return values of functions | Server | Re-use data in a React Component tree | Per-request lifecycle |
| Data Cache | Data | Server | Store data across user requests and deployments | Persistent (can be revalidated) |
| Full Route Cache | HTML and RSC payload | Server | Reduce rendering cost and improve performance | Persistent (can be revalidated) |
| Router Cache | RSC Payload | Client | Reduce server requests on navigation | User session or time-based |

---

## What's New in Next.js 16

Next.js 16 makes caching **explicit and opt-in** through **Cache Components**.

- **`use cache` directive** — mark a component, function, or route as cached. Cache keys are generated automatically by the compiler. Enable it in config:

  ```ts
  // next.config.ts
  const nextConfig = { cacheComponents: true };
  export default nextConfig;
  ```

  ```tsx
  async function Posts() {
    'use cache';
    const posts = await db.post.findMany();
    return <PostList posts={posts} />;
  }
  ```

  This replaces the experimental `dynamicIO` flag and completes the Partial Prerendering (PPR) model; the `experimental.ppr` flag and `export const experimental_ppr` are removed.

- **`revalidateTag(tag, profile)`** — now takes a `cacheLife` profile as the second argument for stale-while-revalidate behavior (`'max'`, `'hours'`, `'days'`, or `{ expire: seconds }`). The single-argument form is deprecated.

- **`updateTag(tag)`** — new Server Actions-only API with read-your-writes semantics (expires and immediately re-reads within the same request).

- **`refresh()`** — new Server Actions-only API that refreshes uncached data only, without touching the cache.

```ts
'use server';
import { revalidateTag, updateTag, refresh } from 'next/cache';

// SWR invalidation for tagged content
revalidateTag('posts', 'max');

// Immediate read-your-writes after a mutation
updateTag(`user-${id}`);

// Refresh uncached data (e.g. a live counter)
refresh();
```

> Middleware note: `middleware.ts` is deprecated in favour of `proxy.ts` (Node.js runtime). Turbopack is now the default bundler.

---

## Request Memoization

React extends the `fetch` API to automatically memoize requests with the same URL and options during a single render pass. This means you can call the same fetch in multiple places without worrying about duplicate network requests.

### How It Works

```tsx
// layout.tsx
async function Layout({ children }: { children: React.ReactNode }) {
  // First call - makes network request
  const user = await fetch('/api/user').then(res => res.json());

  return (
    <div>
      <Header user={user} />
      {children}
    </div>
  );
}

// page.tsx (in same render)
async function Page() {
  // Second call - returns memoized result, no network request
  const user = await fetch('/api/user').then(res => res.json());

  return <Profile user={user} />;
}
```

### Using React cache() for Non-Fetch Functions

For functions that don't use `fetch`, use React's `cache()` function:

```tsx
import { cache } from 'react';

// Memoize database queries or other expensive operations
export const getUser = cache(async (id: string) => {
  const user = await db.user.findUnique({ where: { id } });
  return user;
});

// Both calls use the same cached result
async function Layout() {
  const user = await getUser('123');
  return <div>{/* ... */}</div>;
}

async function Page() {
  const user = await getUser('123'); // Memoized!
  return <Profile user={user} />;
}
```

### Key Characteristics

- Only works during a single server render pass
- Only applies to GET method in fetch requests
- Only works in React Server Components
- Automatically cleared after the render completes
- Cannot be shared across different requests

---

## Data Cache

The Data Cache persists fetch results across incoming server requests and deployments. Unlike Request Memoization, this cache survives beyond a single render.

### Default Behavior

Since Next.js 15, `fetch` is **not cached by default** — each call hits the network at request time unless you opt in. (Next.js 14 and earlier cached `fetch` indefinitely by default.)

```tsx
// NOT cached by default in Next.js 15/16 — fetched at request time
const data = await fetch('https://api.example.com/posts');

// Opt into caching explicitly
const data = await fetch('https://api.example.com/posts', {
  cache: 'force-cache', // Cache until revalidated
});
```

### Cache Options

```tsx
// Option 1: No caching - always fetch fresh data
const data = await fetch('https://api.example.com/posts', {
  cache: 'no-store',
});

// Option 2: Time-based revalidation
const data = await fetch('https://api.example.com/posts', {
  next: { revalidate: 3600 }, // Revalidate every hour
});

// Option 3: Tag-based revalidation
const data = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts', 'homepage'] },
});

// Option 4: Combine revalidation with tags
const data = await fetch('https://api.example.com/posts', {
  next: {
    revalidate: 3600,
    tags: ['posts'],
  },
});
```

### Fetch Cache Behavior Summary

| Option | Data Cache | Revalidation |
|--------|------------|--------------|
| (no option) — default in 15/16 | Not cached | Every request |
| `cache: 'force-cache'` | Cached | None until manual revalidation |
| `cache: 'no-store'` | Not cached | Every request |
| `next: { revalidate: N }` | Cached | After N seconds |
| `next: { tags: [...] }` | Cached | On-demand via `revalidateTag()` |

---

## Full Route Cache

Next.js automatically renders and caches routes at build time. This is the Full Route Cache, which stores both the HTML and React Server Component Payload.

### Static vs Dynamic Rendering

```tsx
// STATIC: Cached at build time
// This page will be statically generated
export default async function StaticPage() {
  const posts = await fetch('https://api.example.com/posts');
  return <PostList posts={posts} />;
}

// DYNAMIC: Rendered on every request
// Using dynamic functions opts out of static rendering
export default async function DynamicPage() {
  const session = await cookies(); // Dynamic function
  return <Dashboard user={session.user} />;
}
```

### What Makes a Route Dynamic

Routes become dynamic when they use:

```tsx
// 1. Dynamic functions
import { cookies, headers } from 'next/headers';

const cookieStore = await cookies();
const headersList = await headers();

// 2. Uncached data requests
fetch(url, { cache: 'no-store' });

// 3. Dynamic route segment config
export const dynamic = 'force-dynamic';

// 4. searchParams in page components
export default async function Page({ searchParams }: {
  searchParams: Promise<{ query: string }>
}) {
  const { query } = await searchParams;
  // Using searchParams makes the page dynamic
}
```

### Route Segment Config Options

```tsx
// Force static generation (error if dynamic functions used)
export const dynamic = 'force-static';

// Force dynamic rendering
export const dynamic = 'force-dynamic';

// Default: auto-detect based on usage
export const dynamic = 'auto';

// Set default revalidation for all fetches in segment
export const revalidate = 3600; // seconds
```

---

## Router Cache

The Router Cache is a client-side in-memory cache that stores the React Server Component Payload of visited route segments.

### How Prefetching Works

```tsx
import Link from 'next/link';

// Static routes: Fully prefetched and cached for 5 minutes
<Link href="/about">About</Link>

// Dynamic routes: Only the shared layout is prefetched; the page itself is
// not cached by default (staleTimes.dynamic is 0 since Next.js 15)
<Link href="/dashboard">Dashboard</Link>

// Disable prefetching
<Link href="/heavy-page" prefetch={false}>Heavy Page</Link>

// Force full prefetch for dynamic routes
<Link href="/dashboard" prefetch={true}>Dashboard</Link>
```

### Router Cache Duration

| Route Type | Default Duration | After Invalidation |
|------------|------------------|-------------------|
| Static | 5 minutes | Immediate refresh |
| Dynamic | 0 (not cached by default since Next.js 15) | Immediate refresh |

### Invalidating Router Cache

```tsx
'use client';

import { useRouter } from 'next/navigation';

function RefreshButton() {
  const router = useRouter();

  return (
    <button onClick={() => router.refresh()}>
      Refresh
    </button>
  );
}
```

---

## cache() Function Usage

React's `cache()` function memoizes the return value of a function for the duration of a request.

### Basic Usage

```tsx
import { cache } from 'react';
import { db } from '@/lib/db';

// Define the cached function
export const getUser = cache(async (id: string) => {
  const user = await db.user.findUnique({
    where: { id },
    include: { posts: true },
  });
  return user;
});

// Use in multiple components - only one DB query
async function UserProfile({ userId }: { userId: string }) {
  const user = await getUser(userId);
  return <div>{user.name}</div>;
}

async function UserPosts({ userId }: { userId: string }) {
  const user = await getUser(userId); // Same user, no extra query
  return <PostList posts={user.posts} />;
}
```

### With Preload Pattern

```tsx
import { cache } from 'react';

export const getItem = cache(async (id: string) => {
  const item = await db.item.findUnique({ where: { id } });
  return item;
});

// Preload function to start fetching early
export const preloadItem = (id: string) => {
  void getItem(id);
};

// In a parent component
import { preloadItem } from './data';

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  // Start fetching immediately
  preloadItem(id);

  return (
    <Suspense fallback={<Loading />}>
      <ItemDetails id={id} />
    </Suspense>
  );
}
```

### Important Notes

- `cache()` is for request-level memoization only
- Arguments must be serializable
- Each argument combination creates a separate cache entry
- Only works in Server Components

---

## unstable_cache for Database Queries

`unstable_cache` extends caching to non-fetch operations like database queries, persisting results across requests.

### Basic Usage

```tsx
import { unstable_cache } from 'next/cache';
import { db } from '@/lib/db';

const getCachedUser = unstable_cache(
  async (id: string) => {
    return db.user.findUnique({ where: { id } });
  },
  ['user-cache'], // Cache key prefix
  {
    revalidate: 3600, // Revalidate every hour
    tags: ['users'],  // For on-demand revalidation
  }
);

// Usage
const user = await getCachedUser('123');
```

### With Multiple Tags

```tsx
const getCachedPosts = unstable_cache(
  async (userId: string) => {
    return db.post.findMany({
      where: { authorId: userId },
      orderBy: { createdAt: 'desc' },
    });
  },
  ['posts-by-user'],
  {
    revalidate: 1800,
    tags: ['posts', 'user-posts'],
  }
);
```

### Dynamic Cache Keys

```tsx
const getCachedData = unstable_cache(
  async (category: string, page: number) => {
    return db.item.findMany({
      where: { category },
      skip: page * 10,
      take: 10,
    });
  },
  // Key parts can be dynamic
  ['items'],
  { tags: ['items'] }
);
```

---

## revalidatePath and revalidateTag

These functions enable on-demand cache invalidation from Server Actions and Route Handlers.

### revalidatePath

```tsx
'use server';

import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  await db.post.create({ data: { /* ... */ } });

  // Revalidate a specific path
  revalidatePath('/posts');

  // Revalidate a dynamic route
  revalidatePath('/posts/[slug]', 'page');

  // Revalidate all data for a layout
  revalidatePath('/dashboard', 'layout');

  // Revalidate everything
  revalidatePath('/', 'layout');
}
```

### revalidateTag

```tsx
'use server';

import { revalidateTag } from 'next/cache';

export async function updateUser(userId: string, data: UserData) {
  await db.user.update({ where: { id: userId }, data });

  // Revalidate all fetches tagged with 'users'
  revalidateTag('users');

  // Revalidate specific user
  revalidateTag(`user-${userId}`);
}

// Setup tags in fetch
const user = await fetch(`/api/users/${id}`, {
  next: { tags: ['users', `user-${id}`] },
});
```

### In Route Handlers

```tsx
// app/api/revalidate/route.ts
import { revalidateTag, revalidatePath } from 'next/cache';
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const { tag, path, secret } = await request.json();

  // Verify secret token
  if (secret !== process.env.REVALIDATION_SECRET) {
    return Response.json({ error: 'Invalid secret' }, { status: 401 });
  }

  if (tag) {
    revalidateTag(tag);
  }

  if (path) {
    revalidatePath(path);
  }

  return Response.json({ revalidated: true, now: Date.now() });
}
```

---

## Time-Based Revalidation

Configure automatic cache revalidation after a specified time interval.

### Fetch-Level Revalidation

```tsx
// Revalidate every 60 seconds
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 60 },
});

// Revalidate every hour
const posts = await fetch('https://api.example.com/posts', {
  next: { revalidate: 3600 },
});

// Revalidate daily
const config = await fetch('https://api.example.com/config', {
  next: { revalidate: 86400 },
});
```

### Segment-Level Revalidation

```tsx
// app/posts/layout.tsx
// All fetches in this segment will revalidate every hour by default
export const revalidate = 3600;

export default function PostsLayout({ children }) {
  return <div>{children}</div>;
}
```

### Stale-While-Revalidate Behavior

```
Request 1 (0s)    -> Cache MISS -> Fetch -> Store -> Return
Request 2 (30s)   -> Cache HIT -> Return cached
Request 3 (61s)   -> Cache STALE -> Return cached -> Background revalidate
Request 4 (62s)   -> Cache HIT (fresh) -> Return new data
```

---

## On-Demand Revalidation

Trigger cache revalidation programmatically in response to events.

### Server Action Example

```tsx
// actions.ts
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';

export async function publishPost(postId: string) {
  // Update database
  await db.post.update({
    where: { id: postId },
    data: { published: true },
  });

  // Revalidate relevant caches
  revalidateTag('posts');
  revalidatePath('/posts');
  revalidatePath(`/posts/${postId}`);
}

export async function deleteComment(commentId: string, postId: string) {
  await db.comment.delete({ where: { id: commentId } });

  // Only revalidate the affected post
  revalidateTag(`post-${postId}`);
  revalidatePath(`/posts/${postId}`);
}
```

### Webhook Handler

```tsx
// app/api/webhook/route.ts
import { revalidateTag } from 'next/cache';
import { headers } from 'next/headers';
import crypto from 'crypto';

export async function POST(request: Request) {
  const body = await request.text();
  const signature = (await headers()).get('x-webhook-signature');

  // Verify webhook signature
  const expectedSignature = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET!)
    .update(body)
    .digest('hex');

  if (signature !== expectedSignature) {
    return Response.json({ error: 'Invalid signature' }, { status: 401 });
  }

  const payload = JSON.parse(body);

  // Revalidate based on event type
  switch (payload.event) {
    case 'product.updated':
      revalidateTag(`product-${payload.productId}`);
      revalidateTag('products');
      break;
    case 'inventory.changed':
      revalidateTag('inventory');
      break;
  }

  return Response.json({ success: true });
}
```

---

## Opting Out of Caching

Sometimes you need fresh data on every request.

### Fetch-Level Opt-Out

```tsx
// No caching for this specific fetch
const data = await fetch('https://api.example.com/realtime', {
  cache: 'no-store',
});
```

### Segment-Level Opt-Out

```tsx
// app/dashboard/page.tsx
export const dynamic = 'force-dynamic';
export const revalidate = 0;

export default async function Dashboard() {
  // This page will never be cached
  const stats = await getRealtimeStats();
  return <StatsDisplay stats={stats} />;
}
```

### Using Dynamic Functions

```tsx
import { cookies, headers } from 'next/headers';

export default async function Page() {
  // Using these automatically opts out of static caching
  const cookieStore = await cookies();
  const headersList = await headers();

  return <div>Dynamic content</div>;
}
```

### noStore() Function

```tsx
import { unstable_noStore as noStore } from 'next/cache';

export default async function Page() {
  // Explicitly opt out of caching for this component
  noStore();

  const data = await fetchSensitiveData();
  return <div>{data}</div>;
}
```

---

## Cache Interactions and Invalidation

Understanding how caches interact is crucial for proper cache management.

### Cache Flow Diagram

```
Client Request
     |
     v
[Router Cache] -- HIT --> Return cached RSC Payload
     |
     | MISS
     v
[Full Route Cache] -- HIT --> Return cached HTML/RSC
     |
     | MISS
     v
[Data Cache] -- HIT --> Return cached data
     |
     | MISS
     v
[Request Memoization] -- HIT --> Return memoized
     |
     | MISS
     v
Origin (API/Database)
```

### Invalidation Cascade

```tsx
// When you call revalidateTag('posts'):
// 1. Data Cache entries with 'posts' tag are invalidated
// 2. Full Route Cache for affected routes is invalidated
// 3. Router Cache on clients will fetch fresh data on next navigation

'use server';

import { revalidateTag } from 'next/cache';

export async function updatePost(id: string, data: PostData) {
  await db.post.update({ where: { id }, data });

  // This triggers invalidation cascade
  revalidateTag('posts');
  revalidateTag(`post-${id}`);
}
```

### Handling Cache Dependencies

```tsx
// Tag your fetches to create logical groups
const posts = await fetch('/api/posts', {
  next: { tags: ['posts', 'homepage-data'] },
});

const featuredPosts = await fetch('/api/posts/featured', {
  next: { tags: ['posts', 'featured', 'homepage-data'] },
});

// Invalidate all homepage data at once
revalidateTag('homepage-data');

// Or just featured posts
revalidateTag('featured');
```

---

## Best Practices

### 1. Use Appropriate Cache Strategies

```tsx
// Frequently changing data - short revalidation
const stockPrices = await fetch('/api/stocks', {
  next: { revalidate: 60 },
});

// Rarely changing data - longer revalidation
const categories = await fetch('/api/categories', {
  next: { revalidate: 86400 },
});

// User-specific data - no cache
const userSession = await fetch('/api/session', {
  cache: 'no-store',
});
```

### 2. Implement Granular Tags

```tsx
// Good: Specific tags allow targeted invalidation
const post = await fetch(`/api/posts/${id}`, {
  next: { tags: ['posts', `post-${id}`, `author-${authorId}`] },
});

// Bad: Too broad, invalidates too much
const post = await fetch(`/api/posts/${id}`, {
  next: { tags: ['data'] },
});
```

### 3. Combine cache() with unstable_cache

```tsx
import { cache } from 'react';
import { unstable_cache } from 'next/cache';

// Outer: Request memoization
// Inner: Cross-request caching
export const getUser = cache(
  unstable_cache(
    async (id: string) => db.user.findUnique({ where: { id } }),
    ['users'],
    { revalidate: 3600, tags: ['users'] }
  )
);
```

### 4. Preload Critical Data

```tsx
import { getPost, preloadPost } from './data';

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  // Start fetching immediately, don't await
  preloadPost(id);

  return (
    <Suspense fallback={<PostSkeleton />}>
      <PostContent id={id} />
    </Suspense>
  );
}
```

### 5. Use generateStaticParams for Known Routes

```tsx
// Generate static pages for known posts
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());

  return posts.map((post) => ({
    slug: post.slug,
  }));
}
```

---

## Common Pitfalls

### 1. Forgetting Cache Persists Across Deployments

```tsx
// Problem: Stale data after deployment
const config = await fetch('/api/config');

// Solution: Use revalidation or invalidate on deploy
const config = await fetch('/api/config', {
  next: { revalidate: 300, tags: ['config'] },
});
```

### 2. Not Handling Cache in Development

```tsx
// Cache behaves differently in dev vs production
// In development, caches are cleared on every refresh
// Test caching behavior with: next build && next start
```

### 3. Over-Caching Dynamic Data

```tsx
// Problem: User sees stale personalized content
const recommendations = await fetch(`/api/recommendations?user=${userId}`);

// Solution: Opt out for personalized data
const recommendations = await fetch(`/api/recommendations?user=${userId}`, {
  cache: 'no-store',
});
```

### 4. Incorrect Tag Revalidation

```tsx
// Problem: Tags must match exactly
fetch('/api/data', { next: { tags: ['my-tag'] } });
revalidateTag('mytag'); // Won't work! Different tag

// Solution: Use constants for tag names
const TAGS = {
  POSTS: 'posts',
  USERS: 'users',
} as const;

fetch('/api/posts', { next: { tags: [TAGS.POSTS] } });
revalidateTag(TAGS.POSTS); // Works correctly
```

### 5. Missing Error Handling

```tsx
// Problem: Cached errors
const data = await fetch('/api/data', { next: { revalidate: 3600 } });

// Solution: Don't cache errors
async function getData() {
  const res = await fetch('/api/data', { next: { revalidate: 3600 } });

  if (!res.ok) {
    // Throw to prevent caching the error
    throw new Error('Failed to fetch data');
  }

  return res.json();
}
```

### 6. Router Cache Confusion

```tsx
// Problem: User sees stale data after server update
// Router Cache on client still has old data

// Solution 1: Call router.refresh() after mutations
'use client';
import { useRouter } from 'next/navigation';

function UpdateButton() {
  const router = useRouter();

  async function handleUpdate() {
    await updateData();
    router.refresh(); // Clear router cache
  }

  return <button onClick={handleUpdate}>Update</button>;
}

// Solution 2: Use Server Actions (automatically refresh)
'use server';
async function updateAction() {
  await updateData();
  revalidatePath('/dashboard');
  // Router cache automatically cleared for this path
}
```

### 7. Caching with Authentication

```tsx
// Problem: Cached data leaks between users
const userData = await fetch('/api/user/profile');

// Solution: Always use no-store for user-specific data
const userData = await fetch('/api/user/profile', {
  cache: 'no-store',
  headers: {
    Authorization: `Bearer ${token}`,
  },
});
```
