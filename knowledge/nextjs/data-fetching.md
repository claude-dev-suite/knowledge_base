# Next.js Data Fetching

Comprehensive guide to data fetching in Next.js 14+ with the App Router. This documentation covers server-side and client-side data fetching strategies, caching, revalidation, and best practices.

---

## Table of Contents

1. [Server Components Data Fetching](#1-server-components-data-fetching)
2. [Fetch API with Next.js Extensions](#2-fetch-api-with-nextjs-extensions)
3. [Caching Strategies](#3-caching-strategies)
4. [Route Segment Config](#4-route-segment-config)
5. [Parallel vs Sequential Data Fetching](#5-parallel-vs-sequential-data-fetching)
6. [Streaming with Suspense](#6-streaming-with-suspense)
7. [Client-Side Fetching](#7-client-side-fetching)
8. [Revalidation](#8-revalidation)
9. [Request Memoization](#9-request-memoization)
10. [Data Fetching Patterns](#10-data-fetching-patterns)
11. [Error Handling](#11-error-handling)
12. [Loading States](#12-loading-states)
13. [Incremental Static Regeneration](#13-incremental-static-regeneration-isr)
14. [Static vs Dynamic Rendering](#14-static-vs-dynamic-rendering)
15. [generateStaticParams](#15-generatestaticparams)
16. [Best Practices](#16-best-practices)

---

## 1. Server Components Data Fetching

Server Components are the recommended way to fetch data in Next.js. They allow you to use async/await directly in your components.

### Basic Async/Await Pattern

```tsx
// app/posts/page.tsx
async function PostsPage() {
  const response = await fetch('https://api.example.com/posts');
  const posts = await response.json();

  return (
    <main>
      <h1>Blog Posts</h1>
      <ul>
        {posts.map((post: Post) => (
          <li key={post.id}>
            <h2>{post.title}</h2>
            <p>{post.excerpt}</p>
          </li>
        ))}
      </ul>
    </main>
  );
}

export default PostsPage;
```

### Direct Database Access

Server Components can access databases directly without an API layer:

```tsx
// app/users/page.tsx
import { prisma } from '@/lib/prisma';

async function UsersPage() {
  // Direct database query - no API needed!
  const users = await prisma.user.findMany({
    select: {
      id: true,
      name: true,
      email: true,
      createdAt: true,
    },
    orderBy: { createdAt: 'desc' },
  });

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>
          <span>{user.name}</span>
          <span>{user.email}</span>
        </li>
      ))}
    </ul>
  );
}

export default UsersPage;
```

### ORM and Database Libraries

```tsx
// Using Drizzle ORM
import { db } from '@/lib/db';
import { users } from '@/lib/schema';

async function Page() {
  const allUsers = await db.select().from(users);
  return <UserList users={allUsers} />;
}

// Using Mongoose
import connectDB from '@/lib/mongodb';
import User from '@/models/User';

async function Page() {
  await connectDB();
  const users = await User.find({}).lean();
  return <UserList users={users} />;
}

// Using raw SQL with pg
import { sql } from '@vercel/postgres';

async function Page() {
  const { rows } = await sql`SELECT * FROM users ORDER BY created_at DESC`;
  return <UserList users={rows} />;
}
```

### Fetching in Nested Components

Each Server Component can independently fetch its own data:

```tsx
// app/dashboard/page.tsx
import { UserProfile } from './components/user-profile';
import { RecentActivity } from './components/recent-activity';
import { Statistics } from './components/statistics';

export default function DashboardPage() {
  return (
    <div className="dashboard">
      <UserProfile />      {/* Fetches user data */}
      <Statistics />       {/* Fetches stats data */}
      <RecentActivity />   {/* Fetches activity data */}
    </div>
  );
}

// app/dashboard/components/user-profile.tsx
async function UserProfile() {
  const user = await fetch('https://api.example.com/user').then(r => r.json());
  return <div>{user.name}</div>;
}

// app/dashboard/components/statistics.tsx
async function Statistics() {
  const stats = await fetch('https://api.example.com/stats').then(r => r.json());
  return <div>{stats.totalViews} views</div>;
}
```

---

## 2. Fetch API with Next.js Extensions

Next.js extends the native `fetch` API with additional options for caching and revalidation.

### Extended Fetch Options

```tsx
// Default: cached indefinitely (equivalent to force-cache)
const data = await fetch('https://api.example.com/data');

// Revalidate after specified seconds
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 60 }, // Revalidate every 60 seconds
});

// No caching - always fetch fresh data
const data = await fetch('https://api.example.com/data', {
  cache: 'no-store',
});

// Force cache (explicit)
const data = await fetch('https://api.example.com/data', {
  cache: 'force-cache',
});

// Cache with tags for targeted revalidation
const data = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts', 'blog'] },
});

// Combine revalidation with tags
const data = await fetch('https://api.example.com/posts', {
  next: {
    revalidate: 3600,
    tags: ['posts'],
  },
});
```

### Fetch with Request Headers

```tsx
async function Page() {
  const response = await fetch('https://api.example.com/data', {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.API_TOKEN}`,
      'X-Custom-Header': 'custom-value',
    },
    next: { revalidate: 60 },
  });

  if (!response.ok) {
    throw new Error('Failed to fetch data');
  }

  return response.json();
}
```

### POST, PUT, DELETE Requests

```tsx
// POST request
async function createPost(data: PostData) {
  const response = await fetch('https://api.example.com/posts', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(data),
    cache: 'no-store', // Mutations should not be cached
  });

  return response.json();
}

// PUT request with authentication
async function updateUser(id: string, data: UserData) {
  const response = await fetch(`https://api.example.com/users/${id}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.API_TOKEN}`,
    },
    body: JSON.stringify(data),
    cache: 'no-store',
  });

  if (!response.ok) {
    throw new Error('Failed to update user');
  }

  return response.json();
}
```

### Typed Fetch Helper

```tsx
// lib/fetch.ts
async function fetchJson<T>(
  url: string,
  options?: RequestInit & { next?: NextFetchRequestConfig }
): Promise<T> {
  const response = await fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  });

  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }

  return response.json();
}

// Usage
interface Post {
  id: number;
  title: string;
  content: string;
}

const posts = await fetchJson<Post[]>('https://api.example.com/posts', {
  next: { revalidate: 60 },
});
```

---

## 3. Caching Strategies

Next.js provides multiple caching layers for optimal performance.

### Data Cache

The Data Cache persists fetch results across requests and deployments:

```tsx
// Cached indefinitely until manually revalidated
const staticData = await fetch('https://api.example.com/data', {
  cache: 'force-cache', // Default behavior
});

// Opt out of Data Cache
const dynamicData = await fetch('https://api.example.com/data', {
  cache: 'no-store',
});

// Time-based cache with automatic revalidation
const timedData = await fetch('https://api.example.com/data', {
  next: { revalidate: 3600 }, // Cache for 1 hour
});
```

### Tag-Based Caching

Tags allow granular cache invalidation:

```tsx
// lib/data.ts
export async function getPosts() {
  const response = await fetch('https://api.example.com/posts', {
    next: { tags: ['posts'] },
  });
  return response.json();
}

export async function getPost(id: string) {
  const response = await fetch(`https://api.example.com/posts/${id}`, {
    next: { tags: ['posts', `post-${id}`] },
  });
  return response.json();
}

export async function getComments(postId: string) {
  const response = await fetch(`https://api.example.com/posts/${postId}/comments`, {
    next: { tags: [`comments-${postId}`] },
  });
  return response.json();
}
```

### Full Route Cache

The Full Route Cache stores rendered HTML and RSC payloads:

```tsx
// Static route - fully cached at build time
// app/about/page.tsx
export default function AboutPage() {
  return <div>About Us</div>;
}

// Dynamic route - not cached
// app/dashboard/page.tsx
import { cookies } from 'next/headers';

export default async function DashboardPage() {
  const cookieStore = cookies();
  const session = cookieStore.get('session');
  return <div>Dashboard for {session?.value}</div>;
}
```

### Router Cache (Client-Side)

The Router Cache stores RSC payloads on the client:

```tsx
// Prefetching caches route on hover/viewport
import Link from 'next/link';

export default function Navigation() {
  return (
    <nav>
      {/* Prefetched automatically */}
      <Link href="/about">About</Link>

      {/* Disable prefetching */}
      <Link href="/dashboard" prefetch={false}>
        Dashboard
      </Link>
    </nav>
  );
}
```

### Cache Hierarchy

```
Request -> Router Cache (Client) -> Full Route Cache -> Data Cache -> Origin
```

---

## 4. Route Segment Config

Configure caching and rendering behavior at the route level.

### Dynamic Export Options

```tsx
// app/page.tsx

// Force dynamic rendering - re-render on every request
export const dynamic = 'force-dynamic';

// Force static rendering - render at build time only
export const dynamic = 'force-static';

// Auto (default) - Next.js determines based on usage
export const dynamic = 'auto';

// Error if dynamic functions are used
export const dynamic = 'error';
```

### Revalidate Configuration

```tsx
// Revalidate this route every 60 seconds
export const revalidate = 60;

// Never revalidate (static forever)
export const revalidate = false;

// Always revalidate (no caching)
export const revalidate = 0;
```

### Runtime Configuration

```tsx
// Use Edge Runtime
export const runtime = 'edge';

// Use Node.js Runtime (default)
export const runtime = 'nodejs';
```

### Preferred Region

```tsx
// Specify preferred region for Edge functions
export const preferredRegion = 'iad1';

// Multiple regions
export const preferredRegion = ['iad1', 'sfo1'];

// Auto (default)
export const preferredRegion = 'auto';

// Run globally
export const preferredRegion = 'global';

// Run only in home region
export const preferredRegion = 'home';
```

### Maximum Duration

```tsx
// Maximum execution time in seconds (serverless functions)
export const maxDuration = 30;
```

### Fetch Cache Configuration

```tsx
// Default fetch cache behavior for the route
export const fetchCache = 'auto'; // Default
export const fetchCache = 'default-cache';
export const fetchCache = 'only-cache';
export const fetchCache = 'force-cache';
export const fetchCache = 'default-no-store';
export const fetchCache = 'only-no-store';
export const fetchCache = 'force-no-store';
```

### Combined Configuration Example

```tsx
// app/api/webhook/route.ts
export const dynamic = 'force-dynamic';
export const runtime = 'edge';
export const preferredRegion = 'iad1';
export const maxDuration = 30;

export async function POST(request: Request) {
  const body = await request.json();
  // Process webhook...
  return Response.json({ received: true });
}
```

---

## 5. Parallel vs Sequential Data Fetching

### Parallel Data Fetching

Fetch independent data simultaneously for better performance:

```tsx
// app/dashboard/page.tsx
async function DashboardPage() {
  // Start all fetches simultaneously
  const [user, posts, notifications, analytics] = await Promise.all([
    getUser(),
    getPosts(),
    getNotifications(),
    getAnalytics(),
  ]);

  return (
    <div className="dashboard">
      <Header user={user} notifications={notifications} />
      <main>
        <Analytics data={analytics} />
        <PostsList posts={posts} />
      </main>
    </div>
  );
}
```

### Promise.allSettled for Fault Tolerance

```tsx
async function DashboardPage() {
  const results = await Promise.allSettled([
    getUser(),
    getPosts(),
    getNotifications(),
  ]);

  const [userResult, postsResult, notificationsResult] = results;

  return (
    <div>
      {userResult.status === 'fulfilled' ? (
        <UserProfile user={userResult.value} />
      ) : (
        <UserProfileError />
      )}

      {postsResult.status === 'fulfilled' ? (
        <Posts posts={postsResult.value} />
      ) : (
        <PostsError />
      )}

      {notificationsResult.status === 'fulfilled' ? (
        <Notifications items={notificationsResult.value} />
      ) : (
        <NotificationsError />
      )}
    </div>
  );
}
```

### Sequential Data Fetching

Use when data depends on previous fetch results:

```tsx
// app/posts/[id]/page.tsx
async function PostPage({ params }: { params: { id: string } }) {
  // First: Get the post
  const post = await getPost(params.id);

  // Then: Get author (depends on post)
  const author = await getAuthor(post.authorId);

  // Then: Get related posts (depends on post category)
  const relatedPosts = await getRelatedPosts(post.categoryId);

  return (
    <article>
      <h1>{post.title}</h1>
      <AuthorCard author={author} />
      <div>{post.content}</div>
      <RelatedPosts posts={relatedPosts} />
    </article>
  );
}
```

### Hybrid Approach

```tsx
async function PostPage({ params }: { params: { id: string } }) {
  // First fetch - needed for subsequent fetches
  const post = await getPost(params.id);

  // Parallel fetches that depend on post
  const [author, relatedPosts, comments] = await Promise.all([
    getAuthor(post.authorId),
    getRelatedPosts(post.categoryId),
    getComments(params.id),
  ]);

  return (
    <article>
      <h1>{post.title}</h1>
      <AuthorCard author={author} />
      <div>{post.content}</div>
      <Comments comments={comments} />
      <RelatedPosts posts={relatedPosts} />
    </article>
  );
}
```

### Optimizing with Preload Pattern

```tsx
// lib/data.ts
import { cache } from 'react';

export const getPost = cache(async (id: string) => {
  const response = await fetch(`https://api.example.com/posts/${id}`);
  return response.json();
});

// Preload function to start fetch early
export const preloadPost = (id: string) => {
  void getPost(id);
};
```

```tsx
// app/posts/[id]/page.tsx
import { getPost, preloadPost } from '@/lib/data';

export default async function PostPage({ params }: { params: { id: string } }) {
  // Start fetching immediately
  preloadPost(params.id);

  // ... other setup work ...

  // Await the already-started fetch
  const post = await getPost(params.id);

  return <article>{post.content}</article>;
}
```

---

## 6. Streaming with Suspense

Streaming allows you to progressively render UI as data becomes available.

### Basic Suspense Usage

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { PostsSkeleton, RecommendationsSkeleton } from '@/components/skeletons';

export default function DashboardPage() {
  return (
    <div className="dashboard">
      {/* Renders immediately */}
      <Header />

      <div className="content">
        {/* Streams when Posts data is ready */}
        <Suspense fallback={<PostsSkeleton />}>
          <Posts />
        </Suspense>

        {/* Streams independently when Recommendations data is ready */}
        <Suspense fallback={<RecommendationsSkeleton />}>
          <Recommendations />
        </Suspense>
      </div>

      {/* Renders immediately */}
      <Footer />
    </div>
  );
}

// Components with async data fetching
async function Posts() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 },
  }).then(r => r.json());

  return (
    <section>
      {posts.map((post: Post) => (
        <PostCard key={post.id} post={post} />
      ))}
    </section>
  );
}

async function Recommendations() {
  // Simulating slow API
  const recommendations = await fetch('https://api.example.com/recommendations', {
    next: { revalidate: 3600 },
  }).then(r => r.json());

  return (
    <aside>
      {recommendations.map((item: Recommendation) => (
        <RecommendationCard key={item.id} item={item} />
      ))}
    </aside>
  );
}
```

### Nested Suspense Boundaries

```tsx
export default function ProductPage({ params }: { params: { id: string } }) {
  return (
    <div>
      {/* Outer boundary for main product */}
      <Suspense fallback={<ProductSkeleton />}>
        <ProductDetails id={params.id} />

        {/* Nested boundary for reviews */}
        <Suspense fallback={<ReviewsSkeleton />}>
          <ProductReviews productId={params.id} />
        </Suspense>

        {/* Nested boundary for related products */}
        <Suspense fallback={<RelatedSkeleton />}>
          <RelatedProducts productId={params.id} />
        </Suspense>
      </Suspense>
    </div>
  );
}
```

### Streaming with loading.tsx

```tsx
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div className="dashboard-skeleton">
      <div className="skeleton-header" />
      <div className="skeleton-content">
        <div className="skeleton-card" />
        <div className="skeleton-card" />
        <div className="skeleton-card" />
      </div>
    </div>
  );
}

// This loading UI is shown while app/dashboard/page.tsx loads
```

### Sequential Streaming

```tsx
// Stream content in sequence
export default function Page() {
  return (
    <div>
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
        <Suspense fallback={<MainSkeleton />}>
          <MainContent />
          <Suspense fallback={<FooterSkeleton />}>
            <Footer />
          </Suspense>
        </Suspense>
      </Suspense>
    </div>
  );
}
```

### Streaming with React 18 Features

```tsx
'use client';

import { use } from 'react';

// Using the `use` hook to unwrap promises
function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise);

  return (
    <ul>
      {comments.map((comment) => (
        <li key={comment.id}>{comment.text}</li>
      ))}
    </ul>
  );
}

// Parent component
export default function Post({ post }: { post: Post }) {
  // Pass promise to child - it will suspend
  const commentsPromise = fetch(`/api/comments?postId=${post.id}`)
    .then(r => r.json());

  return (
    <article>
      <h1>{post.title}</h1>
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments commentsPromise={commentsPromise} />
      </Suspense>
    </article>
  );
}
```

---

## 7. Client-Side Fetching

Use client-side fetching for real-time updates, user interactions, or when data depends on client state.

### SWR (Stale-While-Revalidate)

```tsx
'use client';

import useSWR from 'swr';

// Fetcher function
const fetcher = (url: string) => fetch(url).then((r) => r.json());

function UserProfile() {
  const { data, error, isLoading, isValidating, mutate } = useSWR(
    '/api/user',
    fetcher,
    {
      revalidateOnFocus: true,
      revalidateOnReconnect: true,
      refreshInterval: 30000, // Refresh every 30 seconds
    }
  );

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Failed to load user</div>;

  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.email}</p>
      {isValidating && <span>Updating...</span>}
      <button onClick={() => mutate()}>Refresh</button>
    </div>
  );
}
```

### SWR with Conditional Fetching

```tsx
'use client';

import useSWR from 'swr';

function UserPosts({ userId }: { userId: string | null }) {
  // Only fetch when userId exists
  const { data, error } = useSWR(
    userId ? `/api/users/${userId}/posts` : null,
    fetcher
  );

  if (!userId) return <div>Select a user</div>;
  if (error) return <div>Error loading posts</div>;
  if (!data) return <div>Loading posts...</div>;

  return (
    <ul>
      {data.map((post: Post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### SWR with Mutations

```tsx
'use client';

import useSWR, { useSWRConfig } from 'swr';

function TodoList() {
  const { data: todos, mutate } = useSWR('/api/todos', fetcher);
  const { mutate: globalMutate } = useSWRConfig();

  async function addTodo(text: string) {
    // Optimistic update
    const optimisticTodos = [...todos, { id: Date.now(), text, completed: false }];

    mutate(
      async () => {
        const response = await fetch('/api/todos', {
          method: 'POST',
          body: JSON.stringify({ text }),
        });
        return response.json();
      },
      {
        optimisticData: optimisticTodos,
        rollbackOnError: true,
        revalidate: true,
      }
    );
  }

  return (
    <div>
      {todos?.map((todo: Todo) => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
      <AddTodoForm onAdd={addTodo} />
    </div>
  );
}
```

### React Query (TanStack Query)

```tsx
'use client';

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function Posts() {
  const queryClient = useQueryClient();

  // Query
  const { data, isPending, isError, error } = useQuery({
    queryKey: ['posts'],
    queryFn: () => fetch('/api/posts').then((r) => r.json()),
    staleTime: 60 * 1000, // 1 minute
    gcTime: 5 * 60 * 1000, // 5 minutes (formerly cacheTime)
  });

  // Mutation
  const createPost = useMutation({
    mutationFn: (newPost: NewPost) =>
      fetch('/api/posts', {
        method: 'POST',
        body: JSON.stringify(newPost),
      }).then((r) => r.json()),
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });

  if (isPending) return <div>Loading...</div>;
  if (isError) return <div>Error: {error.message}</div>;

  return (
    <div>
      {data.map((post: Post) => (
        <PostCard key={post.id} post={post} />
      ))}
      <button
        onClick={() => createPost.mutate({ title: 'New Post', content: '...' })}
        disabled={createPost.isPending}
      >
        {createPost.isPending ? 'Creating...' : 'Create Post'}
      </button>
    </div>
  );
}
```

### React Query Provider Setup

```tsx
// app/providers.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000,
            refetchOnWindowFocus: false,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}

// app/layout.tsx
import { Providers } from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

### Infinite Scroll with React Query

```tsx
'use client';

import { useInfiniteQuery } from '@tanstack/react-query';
import { useInView } from 'react-intersection-observer';
import { useEffect } from 'react';

function InfinitePostsList() {
  const { ref, inView } = useInView();

  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    status,
  } = useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: ({ pageParam = 1 }) =>
      fetch(`/api/posts?page=${pageParam}&limit=10`).then((r) => r.json()),
    getNextPageParam: (lastPage, pages) => {
      return lastPage.hasMore ? pages.length + 1 : undefined;
    },
    initialPageParam: 1,
  });

  useEffect(() => {
    if (inView && hasNextPage) {
      fetchNextPage();
    }
  }, [inView, hasNextPage, fetchNextPage]);

  if (status === 'pending') return <div>Loading...</div>;
  if (status === 'error') return <div>Error loading posts</div>;

  return (
    <div>
      {data.pages.map((page, i) => (
        <div key={i}>
          {page.posts.map((post: Post) => (
            <PostCard key={post.id} post={post} />
          ))}
        </div>
      ))}
      <div ref={ref}>
        {isFetchingNextPage ? 'Loading more...' : hasNextPage ? 'Load more' : 'No more posts'}
      </div>
    </div>
  );
}
```

---

## 8. Revalidation

Revalidation allows you to update cached data without rebuilding the entire application.

### Time-Based Revalidation

```tsx
// Revalidate at the fetch level
const data = await fetch('https://api.example.com/posts', {
  next: { revalidate: 3600 }, // Revalidate every hour
});

// Revalidate at the route level
// app/posts/page.tsx
export const revalidate = 3600; // Revalidate this page every hour

export default async function PostsPage() {
  const posts = await getPosts();
  return <PostsList posts={posts} />;
}
```

### On-Demand Revalidation by Path

```tsx
// app/api/revalidate/route.ts
import { revalidatePath } from 'next/cache';
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const { path, secret } = await request.json();

  // Validate secret
  if (secret !== process.env.REVALIDATION_SECRET) {
    return Response.json({ message: 'Invalid secret' }, { status: 401 });
  }

  try {
    // Revalidate specific path
    revalidatePath(path);

    // Revalidate with type specification
    revalidatePath('/blog', 'page'); // Only the page
    revalidatePath('/blog', 'layout'); // Layout and all nested pages

    return Response.json({ revalidated: true, now: Date.now() });
  } catch (error) {
    return Response.json({ message: 'Error revalidating' }, { status: 500 });
  }
}
```

### On-Demand Revalidation by Tag

```tsx
// app/api/revalidate-tag/route.ts
import { revalidateTag } from 'next/cache';
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const { tag, secret } = await request.json();

  if (secret !== process.env.REVALIDATION_SECRET) {
    return Response.json({ message: 'Invalid secret' }, { status: 401 });
  }

  revalidateTag(tag);

  return Response.json({ revalidated: true, tag, now: Date.now() });
}

// Usage in fetch requests
// lib/data.ts
export async function getPosts() {
  const response = await fetch('https://api.example.com/posts', {
    next: { tags: ['posts'] },
  });
  return response.json();
}

export async function getPost(id: string) {
  const response = await fetch(`https://api.example.com/posts/${id}`, {
    next: { tags: ['posts', `post-${id}`] },
  });
  return response.json();
}

// Revalidate all posts: revalidateTag('posts')
// Revalidate single post: revalidateTag('post-123')
```

### Server Actions with Revalidation

```tsx
// app/posts/actions.ts
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  await db.post.create({
    data: { title, content },
  });

  // Revalidate the posts list
  revalidateTag('posts');
  // Or revalidate by path
  revalidatePath('/posts');
}

export async function updatePost(id: string, formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  await db.post.update({
    where: { id },
    data: { title, content },
  });

  // Revalidate specific post and list
  revalidateTag(`post-${id}`);
  revalidateTag('posts');
}

export async function deletePost(id: string) {
  await db.post.delete({ where: { id } });

  revalidateTag('posts');
  revalidatePath('/posts');
}
```

### Webhook-Based Revalidation

```tsx
// app/api/webhook/cms/route.ts
import { revalidateTag } from 'next/cache';
import { headers } from 'next/headers';
import crypto from 'crypto';

export async function POST(request: Request) {
  const body = await request.text();
  const headersList = headers();
  const signature = headersList.get('x-webhook-signature');

  // Verify webhook signature
  const expectedSignature = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET!)
    .update(body)
    .digest('hex');

  if (signature !== expectedSignature) {
    return Response.json({ message: 'Invalid signature' }, { status: 401 });
  }

  const payload = JSON.parse(body);

  // Revalidate based on webhook event
  switch (payload.event) {
    case 'post.created':
    case 'post.updated':
    case 'post.deleted':
      revalidateTag('posts');
      if (payload.postId) {
        revalidateTag(`post-${payload.postId}`);
      }
      break;
    case 'author.updated':
      revalidateTag(`author-${payload.authorId}`);
      break;
  }

  return Response.json({ revalidated: true });
}
```

---

## 9. Request Memoization

React and Next.js automatically memoize fetch requests with the same URL and options.

### Automatic Memoization

```tsx
// This fetch is automatically memoized across components
// within the same render pass

// app/layout.tsx
async function Layout({ children }: { children: React.ReactNode }) {
  // First call - fetches from origin
  const user = await fetch('https://api.example.com/user').then(r => r.json());

  return (
    <div>
      <Header user={user} />
      {children}
    </div>
  );
}

// app/page.tsx
async function Page() {
  // Second call - returns memoized result (no network request)
  const user = await fetch('https://api.example.com/user').then(r => r.json());

  return <UserProfile user={user} />;
}
```

### Manual Memoization with React Cache

```tsx
// lib/data.ts
import { cache } from 'react';

// Memoize non-fetch functions (database calls, etc.)
export const getUser = cache(async (id: string) => {
  const user = await prisma.user.findUnique({
    where: { id },
    include: { posts: true, profile: true },
  });
  return user;
});

export const getPost = cache(async (id: string) => {
  const post = await prisma.post.findUnique({
    where: { id },
    include: { author: true, comments: true },
  });
  return post;
});

// Both calls will use the same cached result
// Component A
const user = await getUser('123');

// Component B (same render)
const user = await getUser('123'); // Returns cached result
```

### Memoization Scope

```tsx
// Memoization only works within a single server request
// Each new request starts with empty memoization cache

// lib/data.ts
import { cache } from 'react';

export const getExpensiveData = cache(async () => {
  console.log('Fetching expensive data...');
  // This will only log once per request
  return await computeExpensiveData();
});

// Request 1: Logs "Fetching expensive data..."
// - Component A calls getExpensiveData() -> fetches
// - Component B calls getExpensiveData() -> uses cached

// Request 2: Logs "Fetching expensive data..." again
// - Component A calls getExpensiveData() -> fetches fresh
// - Component B calls getExpensiveData() -> uses cached
```

### Opting Out of Memoization

```tsx
// Different options will create separate cache entries
const data1 = await fetch('https://api.example.com/data', {
  cache: 'force-cache',
});

const data2 = await fetch('https://api.example.com/data', {
  cache: 'no-store',
});
// These are NOT deduplicated - different cache options

// Add unique identifier to bypass memoization
const data3 = await fetch(`https://api.example.com/data?_t=${Date.now()}`);
```

---

## 10. Data Fetching Patterns

### Preload Pattern

Start fetching data before it's needed:

```tsx
// lib/data.ts
import { cache } from 'react';

export const getPost = cache(async (id: string) => {
  const response = await fetch(`https://api.example.com/posts/${id}`, {
    next: { tags: [`post-${id}`] },
  });
  return response.json();
});

// Preload function - starts fetch without awaiting
export const preloadPost = (id: string) => {
  void getPost(id);
};
```

```tsx
// app/posts/[id]/page.tsx
import { getPost, preloadPost } from '@/lib/data';
import { Comments } from './comments';

export default async function PostPage({ params }: { params: { id: string } }) {
  // Start fetching immediately
  preloadPost(params.id);

  // Component setup, other work...

  // Await the result (likely already resolved)
  const post = await getPost(params.id);

  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.content}</div>
      <Comments postId={params.id} />
    </article>
  );
}
```

### Data Layer Pattern

Centralize data fetching logic:

```tsx
// lib/data/posts.ts
import { cache } from 'react';
import { notFound } from 'next/navigation';

const BASE_URL = process.env.API_URL;

export const getPosts = cache(async (options?: {
  page?: number;
  limit?: number;
  category?: string;
}) => {
  const params = new URLSearchParams();
  if (options?.page) params.set('page', String(options.page));
  if (options?.limit) params.set('limit', String(options.limit));
  if (options?.category) params.set('category', options.category);

  const response = await fetch(`${BASE_URL}/posts?${params}`, {
    next: { tags: ['posts'], revalidate: 60 },
  });

  if (!response.ok) {
    throw new Error('Failed to fetch posts');
  }

  return response.json();
});

export const getPost = cache(async (id: string) => {
  const response = await fetch(`${BASE_URL}/posts/${id}`, {
    next: { tags: ['posts', `post-${id}`] },
  });

  if (response.status === 404) {
    notFound();
  }

  if (!response.ok) {
    throw new Error('Failed to fetch post');
  }

  return response.json();
});

export const preloadPost = (id: string) => void getPost(id);
export const preloadPosts = () => void getPosts();
```

### Server Actions for Mutations

```tsx
// app/posts/actions.ts
'use server';

import { revalidateTag } from 'next/cache';
import { redirect } from 'next/navigation';
import { z } from 'zod';

const PostSchema = z.object({
  title: z.string().min(1).max(100),
  content: z.string().min(1),
  published: z.boolean().default(false),
});

export async function createPost(formData: FormData) {
  const validatedFields = PostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
    published: formData.get('published') === 'true',
  });

  if (!validatedFields.success) {
    return { error: 'Invalid fields', issues: validatedFields.error.issues };
  }

  const response = await fetch(`${process.env.API_URL}/posts`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(validatedFields.data),
  });

  if (!response.ok) {
    return { error: 'Failed to create post' };
  }

  const post = await response.json();

  revalidateTag('posts');
  redirect(`/posts/${post.id}`);
}
```

### Render-As-You-Fetch Pattern

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

// Start fetching immediately when component renders
async function UserStats() {
  const stats = await fetchUserStats(); // Fetch starts here
  return <StatsDisplay stats={stats} />;
}

async function RecentActivity() {
  const activity = await fetchRecentActivity();
  return <ActivityList items={activity} />;
}

async function Notifications() {
  const notifications = await fetchNotifications();
  return <NotificationList items={notifications} />;
}

export default function DashboardPage() {
  // All three components start fetching in parallel
  // Each renders independently when its data is ready
  return (
    <div className="dashboard">
      <ErrorBoundary fallback={<StatsError />}>
        <Suspense fallback={<StatsSkeleton />}>
          <UserStats />
        </Suspense>
      </ErrorBoundary>

      <ErrorBoundary fallback={<ActivityError />}>
        <Suspense fallback={<ActivitySkeleton />}>
          <RecentActivity />
        </Suspense>
      </ErrorBoundary>

      <ErrorBoundary fallback={<NotificationsError />}>
        <Suspense fallback={<NotificationsSkeleton />}>
          <Notifications />
        </Suspense>
      </ErrorBoundary>
    </div>
  );
}
```

---

## 11. Error Handling

### Error Boundaries with error.tsx

```tsx
// app/posts/error.tsx
'use client';

import { useEffect } from 'react';

export default function PostsError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Log error to error reporting service
    console.error('Posts page error:', error);
  }, [error]);

  return (
    <div className="error-container">
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      {error.digest && <p>Error ID: {error.digest}</p>}
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

### Global Error Handling

```tsx
// app/global-error.tsx
'use client';

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <html>
      <body>
        <div className="global-error">
          <h1>Something went wrong!</h1>
          <p>We encountered an unexpected error.</p>
          <button onClick={() => reset()}>Try again</button>
        </div>
      </body>
    </html>
  );
}
```

### Try-Catch in Server Components

```tsx
// app/posts/[id]/page.tsx
import { notFound } from 'next/navigation';

async function getPost(id: string) {
  try {
    const response = await fetch(`https://api.example.com/posts/${id}`, {
      next: { tags: [`post-${id}`] },
    });

    if (response.status === 404) {
      return null;
    }

    if (!response.ok) {
      throw new Error(`Failed to fetch post: ${response.statusText}`);
    }

    return response.json();
  } catch (error) {
    console.error('Error fetching post:', error);
    throw error; // Re-throw to trigger error boundary
  }
}

export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await getPost(params.id);

  if (!post) {
    notFound(); // Triggers not-found.tsx
  }

  return <PostContent post={post} />;
}
```

### Not Found Handling

```tsx
// app/posts/[id]/not-found.tsx
import Link from 'next/link';

export default function PostNotFound() {
  return (
    <div className="not-found">
      <h1>Post Not Found</h1>
      <p>The post you're looking for doesn't exist or has been removed.</p>
      <Link href="/posts">Back to Posts</Link>
    </div>
  );
}
```

### Error Handling with fetch

```tsx
// lib/fetch-utils.ts
export class FetchError extends Error {
  constructor(
    message: string,
    public status: number,
    public statusText: string
  ) {
    super(message);
    this.name = 'FetchError';
  }
}

export async function safeFetch<T>(
  url: string,
  options?: RequestInit & { next?: NextFetchRequestConfig }
): Promise<T> {
  const response = await fetch(url, options);

  if (!response.ok) {
    throw new FetchError(
      `HTTP error ${response.status}`,
      response.status,
      response.statusText
    );
  }

  return response.json();
}

// Usage
try {
  const data = await safeFetch<Post[]>('/api/posts');
} catch (error) {
  if (error instanceof FetchError) {
    if (error.status === 404) {
      notFound();
    }
    if (error.status === 401) {
      redirect('/login');
    }
  }
  throw error;
}
```

---

## 12. Loading States

### Route-Level Loading UI

```tsx
// app/posts/loading.tsx
export default function PostsLoading() {
  return (
    <div className="posts-loading">
      <div className="header-skeleton">
        <div className="skeleton-line w-48 h-8" />
      </div>
      <div className="posts-grid">
        {[...Array(6)].map((_, i) => (
          <div key={i} className="post-skeleton">
            <div className="skeleton-image" />
            <div className="skeleton-line w-full h-6" />
            <div className="skeleton-line w-3/4 h-4" />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Skeleton Components

```tsx
// components/skeletons.tsx
export function PostCardSkeleton() {
  return (
    <div className="post-card-skeleton animate-pulse">
      <div className="bg-gray-200 h-48 rounded-t-lg" />
      <div className="p-4 space-y-3">
        <div className="bg-gray-200 h-6 rounded w-3/4" />
        <div className="bg-gray-200 h-4 rounded w-full" />
        <div className="bg-gray-200 h-4 rounded w-2/3" />
      </div>
    </div>
  );
}

export function PostsSkeleton() {
  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
      {[...Array(6)].map((_, i) => (
        <PostCardSkeleton key={i} />
      ))}
    </div>
  );
}

export function UserProfileSkeleton() {
  return (
    <div className="flex items-center space-x-4 animate-pulse">
      <div className="bg-gray-200 rounded-full h-12 w-12" />
      <div className="space-y-2">
        <div className="bg-gray-200 h-4 rounded w-24" />
        <div className="bg-gray-200 h-3 rounded w-32" />
      </div>
    </div>
  );
}
```

### Loading States with Suspense

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';
import {
  StatsSkeleton,
  ChartSkeleton,
  TableSkeleton,
} from '@/components/skeletons';

export default function DashboardPage() {
  return (
    <div className="dashboard">
      <h1>Dashboard</h1>

      <section className="stats-section">
        <Suspense fallback={<StatsSkeleton />}>
          <Stats />
        </Suspense>
      </section>

      <section className="charts-section">
        <Suspense fallback={<ChartSkeleton />}>
          <RevenueChart />
        </Suspense>
        <Suspense fallback={<ChartSkeleton />}>
          <UserChart />
        </Suspense>
      </section>

      <section className="table-section">
        <Suspense fallback={<TableSkeleton rows={10} />}>
          <RecentOrders />
        </Suspense>
      </section>
    </div>
  );
}
```

### Progressive Loading

```tsx
// Show critical content first, then progressively load more
export default function ProductPage({ params }: { params: { id: string } }) {
  return (
    <div>
      {/* Critical - load first */}
      <Suspense fallback={<ProductHeaderSkeleton />}>
        <ProductHeader id={params.id} />
      </Suspense>

      {/* Important - load second */}
      <Suspense fallback={<ProductDetailsSkeleton />}>
        <ProductDetails id={params.id} />
      </Suspense>

      {/* Less critical - can load later */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews id={params.id} />
      </Suspense>

      {/* Lowest priority */}
      <Suspense fallback={<RelatedSkeleton />}>
        <RelatedProducts id={params.id} />
      </Suspense>
    </div>
  );
}
```

---

## 13. Incremental Static Regeneration (ISR)

ISR allows you to update static pages after deployment without rebuilding the entire site.

### Basic ISR Setup

```tsx
// app/posts/page.tsx
export const revalidate = 60; // Revalidate every 60 seconds

export default async function PostsPage() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 },
  }).then(r => r.json());

  return (
    <div>
      <h1>Blog Posts</h1>
      <p>Last updated: {new Date().toISOString()}</p>
      <PostsList posts={posts} />
    </div>
  );
}
```

### ISR with Dynamic Routes

```tsx
// app/posts/[slug]/page.tsx
import { notFound } from 'next/navigation';

// Generate static pages at build time
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());

  return posts.map((post: Post) => ({
    slug: post.slug,
  }));
}

// Revalidate every 60 seconds
export const revalidate = 60;

export default async function PostPage({ params }: { params: { slug: string } }) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`, {
    next: { revalidate: 60 },
  }).then(r => r.json());

  if (!post) {
    notFound();
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.content}</div>
    </article>
  );
}
```

### On-Demand ISR

```tsx
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const secret = searchParams.get('secret');
  const path = searchParams.get('path');
  const tag = searchParams.get('tag');

  // Validate secret token
  if (secret !== process.env.REVALIDATION_SECRET) {
    return Response.json({ message: 'Invalid token' }, { status: 401 });
  }

  try {
    if (tag) {
      revalidateTag(tag);
      return Response.json({ revalidated: true, tag });
    }

    if (path) {
      revalidatePath(path);
      return Response.json({ revalidated: true, path });
    }

    return Response.json({ message: 'Missing path or tag' }, { status: 400 });
  } catch (error) {
    return Response.json({ message: 'Error revalidating' }, { status: 500 });
  }
}
```

### ISR with fallback Behavior

```tsx
// app/products/[id]/page.tsx
export async function generateStaticParams() {
  // Only pre-render top 100 products
  const products = await fetch('https://api.example.com/products?limit=100')
    .then(r => r.json());

  return products.map((product: Product) => ({
    id: product.id.toString(),
  }));
}

// Enable dynamic params for products not pre-rendered
export const dynamicParams = true; // Default is true

export default async function ProductPage({ params }: { params: { id: string } }) {
  // Products not in generateStaticParams will be:
  // 1. Generated on first request
  // 2. Cached for subsequent requests
  const product = await fetch(`https://api.example.com/products/${params.id}`, {
    next: { revalidate: 3600 },
  }).then(r => r.json());

  return <ProductDetails product={product} />;
}
```

---

## 14. Static vs Dynamic Rendering

### Static Rendering (Default)

Static rendering happens at build time or during revalidation:

```tsx
// app/about/page.tsx
// Fully static - rendered at build time
export default function AboutPage() {
  return (
    <div>
      <h1>About Us</h1>
      <p>This page is statically rendered.</p>
    </div>
  );
}

// Static with data
// app/posts/page.tsx
export default async function PostsPage() {
  // Cached fetch - page is static
  const posts = await fetch('https://api.example.com/posts', {
    cache: 'force-cache',
  }).then(r => r.json());

  return <PostsList posts={posts} />;
}
```

### Dynamic Rendering

Dynamic rendering happens at request time when using:

```tsx
// 1. Dynamic functions
import { cookies, headers } from 'next/headers';

export default async function Page() {
  // Using cookies() makes the page dynamic
  const cookieStore = cookies();
  const theme = cookieStore.get('theme');

  // Using headers() makes the page dynamic
  const headersList = headers();
  const userAgent = headersList.get('user-agent');

  return <div>Theme: {theme?.value}</div>;
}

// 2. searchParams prop
export default function Page({
  searchParams,
}: {
  searchParams: { q: string };
}) {
  // Accessing searchParams makes the page dynamic
  return <SearchResults query={searchParams.q} />;
}

// 3. Uncached fetch
export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'no-store', // Opts into dynamic rendering
  }).then(r => r.json());

  return <DataDisplay data={data} />;
}
```

### Force Static or Dynamic

```tsx
// Force static rendering
export const dynamic = 'force-static';

export default async function Page() {
  // Will be statically rendered even with dynamic features
  return <div>Static content</div>;
}

// Force dynamic rendering
export const dynamic = 'force-dynamic';

export default async function Page() {
  // Will always be dynamically rendered
  const data = await fetch('https://api.example.com/data').then(r => r.json());
  return <div>{data.value}</div>;
}

// Error on dynamic usage
export const dynamic = 'error';

export default async function Page() {
  // Will throw error if any dynamic features are used
  return <div>Must be fully static</div>;
}
```

### Partial Prerendering (Experimental)

```tsx
// next.config.js
module.exports = {
  experimental: {
    ppr: true,
  },
};

// app/page.tsx
import { Suspense } from 'react';

export default function Page() {
  return (
    <div>
      {/* Static shell - rendered at build time */}
      <Header />
      <StaticContent />

      {/* Dynamic content - streamed at request time */}
      <Suspense fallback={<Loading />}>
        <DynamicUserContent />
      </Suspense>

      <Footer />
    </div>
  );
}
```

---

## 15. generateStaticParams

`generateStaticParams` allows you to statically generate routes at build time.

### Basic Usage

```tsx
// app/posts/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());

  return posts.map((post: Post) => ({
    slug: post.slug,
  }));
}

export default async function PostPage({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug);
  return <PostContent post={post} />;
}
```

### Multiple Dynamic Segments

```tsx
// app/[locale]/posts/[slug]/page.tsx
export async function generateStaticParams() {
  const locales = ['en', 'es', 'fr'];
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());

  return locales.flatMap((locale) =>
    posts.map((post: Post) => ({
      locale,
      slug: post.slug,
    }))
  );
}

export default function Page({
  params,
}: {
  params: { locale: string; slug: string };
}) {
  return <div>Locale: {params.locale}, Slug: {params.slug}</div>;
}
```

### With Parent Params

```tsx
// app/products/[category]/[id]/page.tsx
export async function generateStaticParams({
  params: { category },
}: {
  params: { category: string };
}) {
  // Receive parent segment params
  const products = await fetch(
    `https://api.example.com/products?category=${category}`
  ).then(r => r.json());

  return products.map((product: Product) => ({
    id: product.id.toString(),
  }));
}

// Parent segment
// app/products/[category]/layout.tsx
export async function generateStaticParams() {
  return [
    { category: 'electronics' },
    { category: 'clothing' },
    { category: 'books' },
  ];
}
```

### Controlling Dynamic Params

```tsx
// app/posts/[slug]/page.tsx

export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map((post) => ({ slug: post.slug }));
}

// If true (default): URLs not in generateStaticParams are generated on-demand
// If false: URLs not in generateStaticParams return 404
export const dynamicParams = false;

export default async function Page({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug);
  return <PostContent post={post} />;
}
```

### Catch-All Segments

```tsx
// app/docs/[...slug]/page.tsx
export async function generateStaticParams() {
  const docs = await getAllDocs();

  return docs.map((doc) => ({
    slug: doc.path.split('/'), // ['getting-started', 'installation']
  }));
}

export default function DocPage({ params }: { params: { slug: string[] } }) {
  const path = params.slug.join('/');
  return <DocContent path={path} />;
}
```

### Optional Catch-All

```tsx
// app/shop/[[...slug]]/page.tsx
export async function generateStaticParams() {
  return [
    { slug: [] }, // /shop
    { slug: ['category'] }, // /shop/category
    { slug: ['category', 'subcategory'] }, // /shop/category/subcategory
  ];
}

export default function ShopPage({
  params,
}: {
  params: { slug?: string[] };
}) {
  const segments = params.slug ?? [];
  // Handle different path depths
  return <Shop segments={segments} />;
}
```

---

## 16. Best Practices

### 1. Prefer Server Components for Data Fetching

```tsx
// GOOD: Server Component with direct data access
async function UserProfile({ userId }: { userId: string }) {
  const user = await prisma.user.findUnique({
    where: { id: userId },
  });
  return <Profile user={user} />;
}

// AVOID: Client-side fetching for initial data
'use client';
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser);
  }, [userId]);
  return user ? <Profile user={user} /> : <Loading />;
}
```

### 2. Use Parallel Data Fetching

```tsx
// GOOD: Parallel fetching
async function Dashboard() {
  const [user, posts, notifications] = await Promise.all([
    getUser(),
    getPosts(),
    getNotifications(),
  ]);
  return <DashboardContent user={user} posts={posts} notifications={notifications} />;
}

// AVOID: Sequential fetching when not needed
async function Dashboard() {
  const user = await getUser();
  const posts = await getPosts(); // Waits for user
  const notifications = await getNotifications(); // Waits for posts
  return <DashboardContent user={user} posts={posts} notifications={notifications} />;
}
```

### 3. Implement Proper Error Boundaries

```tsx
// Create error.tsx files at appropriate levels
// app/error.tsx - Global errors
// app/posts/error.tsx - Posts section errors
// app/posts/[id]/error.tsx - Individual post errors

// Use ErrorBoundary for client components
import { ErrorBoundary } from 'react-error-boundary';

function Dashboard() {
  return (
    <ErrorBoundary fallback={<ErrorFallback />}>
      <DashboardContent />
    </ErrorBoundary>
  );
}
```

### 4. Use Streaming for Better UX

```tsx
// GOOD: Stream independent sections
export default function Page() {
  return (
    <div>
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>
      <Suspense fallback={<ContentSkeleton />}>
        <Content />
      </Suspense>
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
    </div>
  );
}
```

### 5. Cache Appropriately

```tsx
// Static data - cache indefinitely
const config = await fetch('/api/config', {
  cache: 'force-cache',
});

// Frequently updated - short revalidation
const posts = await fetch('/api/posts', {
  next: { revalidate: 60 },
});

// User-specific or real-time - no cache
const session = await fetch('/api/session', {
  cache: 'no-store',
});

// Use tags for granular invalidation
const post = await fetch(`/api/posts/${id}`, {
  next: { tags: ['posts', `post-${id}`] },
});
```

### 6. Centralize Data Fetching Logic

```tsx
// lib/data/index.ts
export * from './posts';
export * from './users';
export * from './comments';

// lib/data/posts.ts
import { cache } from 'react';

export const getPosts = cache(async (options?: GetPostsOptions) => {
  // Centralized fetching logic
  // Error handling
  // Type safety
  // Caching configuration
});

export const getPost = cache(async (id: string) => {
  // ...
});

export const preloadPost = (id: string) => void getPost(id);
```

### 7. Type Your Data

```tsx
// types/index.ts
export interface Post {
  id: string;
  title: string;
  content: string;
  authorId: string;
  createdAt: string;
  updatedAt: string;
}

export interface User {
  id: string;
  name: string;
  email: string;
}

// lib/data/posts.ts
export async function getPost(id: string): Promise<Post | null> {
  const response = await fetch(`/api/posts/${id}`);
  if (!response.ok) return null;
  return response.json();
}
```

### 8. Use Request Memoization

```tsx
// lib/data.ts
import { cache } from 'react';

// Automatically deduplicated across components
export const getUser = cache(async (id: string) => {
  return prisma.user.findUnique({ where: { id } });
});

// Both components get same cached result
// Layout.tsx
const user = await getUser(params.id);

// Page.tsx
const user = await getUser(params.id); // No extra DB query
```

### 9. Handle Loading States Gracefully

```tsx
// Create meaningful loading UI
// app/posts/loading.tsx
export default function Loading() {
  return (
    <div className="posts-loading" role="status" aria-label="Loading posts">
      <PostsGridSkeleton count={6} />
    </div>
  );
}

// Use skeleton components that match content layout
function PostsGridSkeleton({ count }: { count: number }) {
  return (
    <div className="grid grid-cols-3 gap-4">
      {Array.from({ length: count }).map((_, i) => (
        <PostCardSkeleton key={i} />
      ))}
    </div>
  );
}
```

### 10. Implement Revalidation Strategy

```tsx
// Choose appropriate revalidation strategy:

// 1. Time-based for predictable content
export const revalidate = 3600; // 1 hour for blog posts

// 2. On-demand for user-triggered changes
// app/posts/actions.ts
'use server';
import { revalidateTag } from 'next/cache';

export async function updatePost(id: string, data: PostData) {
  await db.post.update({ where: { id }, data });
  revalidateTag(`post-${id}`);
  revalidateTag('posts');
}

// 3. Webhook-based for CMS changes
// app/api/webhook/route.ts
export async function POST(request: Request) {
  const { type, id } = await request.json();
  revalidateTag(type);
  return Response.json({ revalidated: true });
}
```

### 11. Optimize for Performance

```tsx
// Use generateStaticParams for known routes
export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map((post) => ({ slug: post.slug }));
}

// Preload critical data
import { preloadPost } from '@/lib/data';

export default async function Page({ params }: { params: { id: string } }) {
  preloadPost(params.id); // Start fetch early
  // ... other setup
  const post = await getPost(params.id);
  return <Post post={post} />;
}

// Use dynamic imports for heavy components
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('@/components/chart'), {
  loading: () => <ChartSkeleton />,
});
```

### 12. Security Considerations

```tsx
// Validate and sanitize inputs
export async function getPost(id: string) {
  // Validate ID format
  if (!/^[a-zA-Z0-9-]+$/.test(id)) {
    return null;
  }
  return fetch(`/api/posts/${encodeURIComponent(id)}`);
}

// Never expose sensitive data
async function UserProfile({ userId }: { userId: string }) {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    select: {
      id: true,
      name: true,
      // Exclude: password, email, etc.
    },
  });
  return <Profile user={user} />;
}

// Protect revalidation endpoints
export async function POST(request: Request) {
  const secret = request.headers.get('x-revalidation-secret');
  if (secret !== process.env.REVALIDATION_SECRET) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }
  // ... revalidate
}
```

---

## Summary

Next.js provides a comprehensive data fetching system with:

- **Server Components**: Direct async/await data fetching without API routes
- **Extended Fetch API**: Built-in caching and revalidation options
- **Multiple Caching Layers**: Data Cache, Full Route Cache, and Router Cache
- **Flexible Rendering**: Static, Dynamic, and ISR strategies
- **Streaming**: Progressive UI rendering with Suspense
- **Request Memoization**: Automatic deduplication of fetch requests
- **On-Demand Revalidation**: Update content without full rebuilds

Choose the right strategy based on your data freshness requirements, performance needs, and user experience goals.
