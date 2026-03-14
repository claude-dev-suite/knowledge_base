# TanStack Router Comprehensive Guide

TanStack Router is a fully type-safe, modern routing solution for React applications. It provides file-based routing, first-class search parameter support, built-in caching, and seamless TypeScript integration.

---

## Table of Contents

1. [Setup and Installation](#setup-and-installation)
2. [Route Configuration](#route-configuration)
3. [File-based Routing](#file-based-routing)
4. [Route Tree](#route-tree)
5. [Path Parameters](#path-parameters)
6. [Search Parameters (useSearch)](#search-parameters-usesearch)
7. [Route Loaders](#route-loaders)
8. [Route Actions](#route-actions)
9. [Navigation (Link, useNavigate)](#navigation-link-usenavigate)
10. [Nested Routes and Outlets](#nested-routes-and-outlets)
11. [Layout Routes](#layout-routes)
12. [Error Boundaries](#error-boundaries)
13. [Pending States](#pending-states)
14. [Code Splitting and Lazy Loading](#code-splitting-and-lazy-loading)
15. [Route Context](#route-context)
16. [Guards and Authentication](#guards-and-authentication)
17. [TypeScript Integration](#typescript-integration)
18. [Best Practices](#best-practices)

---

## Setup and Installation

### Installing TanStack Router

```bash
# npm
npm install @tanstack/react-router

# pnpm
pnpm add @tanstack/react-router

# yarn
yarn add @tanstack/react-router
```

### Installing the Vite Plugin (Recommended for File-based Routing)

```bash
# npm
npm install -D @tanstack/router-plugin @tanstack/router-devtools

# pnpm
pnpm add -D @tanstack/router-plugin @tanstack/router-devtools

# yarn
yarn add -D @tanstack/router-plugin @tanstack/router-devtools
```

### Vite Configuration

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { TanStackRouterVite } from '@tanstack/router-plugin/vite';

export default defineConfig({
  plugins: [
    TanStackRouterVite(),
    react(),
  ],
});
```

### Basic Router Setup

```tsx
// src/router.ts
import { createRouter } from '@tanstack/react-router';
import { routeTree } from './routeTree.gen';

export const router = createRouter({
  routeTree,
  defaultPreload: 'intent',
  defaultPreloadStaleTime: 0,
});

// Type registration for full type safety
declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router;
  }
}
```

### Main Application Entry Point

```tsx
// src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { RouterProvider } from '@tanstack/react-router';
import { router } from './router';

// Optional: Add devtools in development
import { TanStackRouterDevtools } from '@tanstack/router-devtools';

function App() {
  return (
    <>
      <RouterProvider router={router} />
      {import.meta.env.DEV && (
        <TanStackRouterDevtools router={router} position="bottom-right" />
      )}
    </>
  );
}

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### Router Configuration Options

```ts
// src/router.ts
import { createRouter, ErrorComponent } from '@tanstack/react-router';
import { routeTree } from './routeTree.gen';

export const router = createRouter({
  routeTree,

  // Preloading strategy
  defaultPreload: 'intent', // 'intent' | 'render' | 'viewport' | false
  defaultPreloadStaleTime: 0, // How long preloaded data stays fresh (ms)

  // Error handling
  defaultErrorComponent: ErrorComponent,
  defaultNotFoundComponent: () => <div>Page Not Found</div>,
  defaultPendingComponent: () => <div>Loading...</div>,

  // Scroll restoration
  defaultPendingMinMs: 500, // Minimum time to show pending state
  defaultPendingMs: 1000, // Time before showing pending state

  // Context passed to all routes
  context: {
    auth: undefined!, // Will be set in RouterProvider
  },
});
```

---

## Route Configuration

### Creating a Basic Route (Code-based)

```tsx
import { createRoute, createRootRoute } from '@tanstack/react-router';

// Root route
const rootRoute = createRootRoute({
  component: RootComponent,
});

// Child route
const indexRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/',
  component: HomePage,
});

// About route
const aboutRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/about',
  component: AboutPage,
});

// Build the route tree
const routeTree = rootRoute.addChildren([
  indexRoute,
  aboutRoute,
]);
```

### Route Options

```tsx
export const Route = createFileRoute('/users')({
  // Component to render
  component: UsersPage,

  // Data loading
  loader: async ({ context, params, search }) => {
    return fetchUsers();
  },

  // Validation
  validateSearch: searchSchema,
  parseParams: (params) => ({ id: Number(params.id) }),
  stringifyParams: (params) => ({ id: String(params.id) }),

  // Navigation guards
  beforeLoad: async ({ context, location }) => {
    // Check auth, redirect, etc.
  },

  // Error handling
  errorComponent: ErrorComponent,
  notFoundComponent: NotFoundComponent,
  pendingComponent: LoadingComponent,

  // Metadata
  meta: () => [
    { title: 'Users Page' },
    { name: 'description', content: 'View all users' },
  ],

  // Stale time for loader data
  staleTime: 5000,

  // Preloading
  preload: true,
  preloadStaleTime: 10000,
});
```

---

## File-based Routing

### Directory Structure

```
src/routes/
├── __root.tsx           # Root layout (required)
├── index.tsx            # / (home page)
├── about.tsx            # /about
├── _layout.tsx          # Layout route (pathless)
├── _layout/
│   ├── dashboard.tsx    # /dashboard (wrapped by _layout)
│   └── settings.tsx     # /settings (wrapped by _layout)
├── _authenticated.tsx   # Auth guard layout
├── _authenticated/
│   ├── profile.tsx      # /profile (requires auth)
│   └── admin.tsx        # /admin (requires auth)
├── users/
│   ├── index.tsx        # /users
│   ├── $userId.tsx      # /users/:userId
│   └── $userId/
│       ├── edit.tsx     # /users/:userId/edit
│       └── posts.tsx    # /users/:userId/posts
├── blog/
│   ├── index.tsx        # /blog
│   ├── $slug.tsx        # /blog/:slug
│   └── [...].tsx        # /blog/* (catch-all)
└── (marketing)/
    ├── pricing.tsx      # /pricing (grouped but no URL segment)
    └── features.tsx     # /features
```

### File Naming Conventions

| Pattern | Description | Example URL |
|---------|-------------|-------------|
| `index.tsx` | Index route | `/users` |
| `about.tsx` | Static segment | `/about` |
| `$param.tsx` | Dynamic parameter | `/users/:param` |
| `$param/` | Dynamic with children | `/users/:param/*` |
| `_layout.tsx` | Pathless layout | (no URL change) |
| `_layout/` | Children of layout | Uses layout |
| `(group)/` | Route grouping | (no URL change) |
| `[...].tsx` | Catch-all / splat | `/path/*` |
| `__root.tsx` | Root route | (required) |

### Root Route (Required)

```tsx
// src/routes/__root.tsx
import { createRootRoute, Outlet, ScrollRestoration } from '@tanstack/react-router';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@/lib/queryClient';

export const Route = createRootRoute({
  component: RootComponent,
  notFoundComponent: GlobalNotFound,
  errorComponent: GlobalError,
});

function RootComponent() {
  return (
    <QueryClientProvider client={queryClient}>
      <div className="min-h-screen bg-background">
        <Outlet />
      </div>
      <ScrollRestoration />
    </QueryClientProvider>
  );
}

function GlobalNotFound() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="text-center">
        <h1 className="text-4xl font-bold">404</h1>
        <p className="mt-2 text-muted-foreground">Page not found</p>
      </div>
    </div>
  );
}

function GlobalError({ error }: { error: Error }) {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="text-center text-red-500">
        <h1 className="text-2xl font-bold">Something went wrong</h1>
        <p className="mt-2">{error.message}</p>
      </div>
    </div>
  );
}
```

### Index Route

```tsx
// src/routes/index.tsx
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/')({
  component: HomePage,
});

function HomePage() {
  return (
    <div>
      <h1>Welcome Home</h1>
    </div>
  );
}
```

---

## Route Tree

### Understanding the Route Tree

The route tree is the hierarchical structure of all routes in your application. With file-based routing, it's automatically generated.

```tsx
// src/routeTree.gen.ts (auto-generated)
import { Route as rootRoute } from './routes/__root';
import { Route as IndexRoute } from './routes/index';
import { Route as AboutRoute } from './routes/about';
import { Route as UsersIndexRoute } from './routes/users/index';
import { Route as UsersUserIdRoute } from './routes/users/$userId';

const routeTree = rootRoute.addChildren([
  IndexRoute,
  AboutRoute,
  UsersIndexRoute.addChildren([UsersUserIdRoute]),
]);

export { routeTree };
```

### Manual Route Tree Construction

```tsx
// For code-based routing
import { createRootRoute, createRoute } from '@tanstack/react-router';

const rootRoute = createRootRoute({
  component: Root,
});

const indexRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/',
  component: Home,
});

const usersRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: 'users',
  component: Users,
});

const userRoute = createRoute({
  getParentRoute: () => usersRoute,
  path: '$userId',
  component: User,
});

// Build tree
const routeTree = rootRoute.addChildren([
  indexRoute,
  usersRoute.addChildren([userRoute]),
]);

// Create router
const router = createRouter({ routeTree });
```

### Route Tree Visualization

```
rootRoute (/)
├── indexRoute (/)
├── aboutRoute (/about)
├── usersRoute (/users)
│   ├── usersIndexRoute (/users)
│   └── userRoute (/users/:userId)
│       ├── userEditRoute (/users/:userId/edit)
│       └── userPostsRoute (/users/:userId/posts)
└── _authenticatedRoute (layout only)
    ├── dashboardRoute (/dashboard)
    └── settingsRoute (/settings)
```

---

## Path Parameters

### Single Path Parameter

```tsx
// src/routes/users/$userId.tsx
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/users/$userId')({
  component: UserPage,
  loader: async ({ params }) => {
    // params.userId is automatically typed as string
    return fetchUser(params.userId);
  },
});

function UserPage() {
  // Type-safe params access
  const { userId } = Route.useParams();
  const user = Route.useLoaderData();

  return (
    <div>
      <h1>User: {userId}</h1>
      <pre>{JSON.stringify(user, null, 2)}</pre>
    </div>
  );
}
```

### Multiple Path Parameters

```tsx
// src/routes/orgs/$orgId/projects/$projectId.tsx
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/orgs/$orgId/projects/$projectId')({
  component: ProjectPage,
  loader: async ({ params }) => {
    const { orgId, projectId } = params;
    return fetchProject(orgId, projectId);
  },
});

function ProjectPage() {
  const { orgId, projectId } = Route.useParams();

  return (
    <div>
      <p>Organization: {orgId}</p>
      <p>Project: {projectId}</p>
    </div>
  );
}
```

### Parsing and Validating Parameters

```tsx
// src/routes/posts/$postId.tsx
import { createFileRoute } from '@tanstack/react-router';
import { z } from 'zod';

const paramsSchema = z.object({
  postId: z.coerce.number().int().positive(),
});

export const Route = createFileRoute('/posts/$postId')({
  // Parse string params to typed values
  params: {
    parse: (params) => paramsSchema.parse(params),
    stringify: (params) => ({ postId: String(params.postId) }),
  },

  component: PostPage,

  loader: async ({ params }) => {
    // params.postId is now typed as number
    return fetchPost(params.postId);
  },
});

function PostPage() {
  const { postId } = Route.useParams();
  // postId is number, not string

  return <div>Post ID: {postId}</div>;
}
```

### Catch-all / Splat Routes

```tsx
// src/routes/docs/[...].tsx (matches /docs/*, /docs/a/b/c, etc.)
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/docs/$')({
  component: DocsPage,
});

function DocsPage() {
  const { _splat } = Route.useParams();
  // _splat contains the rest of the path
  // /docs/a/b/c -> _splat = "a/b/c"

  const segments = _splat?.split('/') ?? [];

  return (
    <div>
      <h1>Documentation</h1>
      <p>Path: {_splat}</p>
      <p>Segments: {segments.join(' > ')}</p>
    </div>
  );
}
```

### Using Parameters in Links

```tsx
import { Link } from '@tanstack/react-router';

function UserList({ users }) {
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>
          <Link
            to="/users/$userId"
            params={{ userId: user.id }}
          >
            {user.name}
          </Link>
        </li>
      ))}
    </ul>
  );
}

// Multiple params
<Link
  to="/orgs/$orgId/projects/$projectId"
  params={{ orgId: '123', projectId: '456' }}
>
  View Project
</Link>
```

---

## Search Parameters (useSearch)

### Basic Search Parameters

```tsx
// src/routes/users.tsx
import { createFileRoute } from '@tanstack/react-router';
import { z } from 'zod';

// Define search schema with Zod
const searchSchema = z.object({
  page: z.number().int().positive().default(1),
  pageSize: z.number().int().min(10).max(100).default(20),
  search: z.string().optional(),
  sortBy: z.enum(['name', 'email', 'createdAt']).default('createdAt'),
  sortOrder: z.enum(['asc', 'desc']).default('desc'),
});

type SearchParams = z.infer<typeof searchSchema>;

export const Route = createFileRoute('/users')({
  validateSearch: searchSchema,
  component: UsersPage,
});

function UsersPage() {
  const { page, pageSize, search, sortBy, sortOrder } = Route.useSearch();

  // All params are fully typed based on the schema
  // page: number, search: string | undefined, etc.

  return (
    <div>
      <p>Page: {page}</p>
      <p>Search: {search ?? 'none'}</p>
      <p>Sort: {sortBy} ({sortOrder})</p>
    </div>
  );
}
```

### Updating Search Parameters

```tsx
import { Link, useNavigate } from '@tanstack/react-router';

function UsersPage() {
  const navigate = useNavigate();
  const { page, search, sortBy } = Route.useSearch();

  // Update single param
  const handleNextPage = () => {
    navigate({
      search: (prev) => ({ ...prev, page: prev.page + 1 }),
    });
  };

  // Update multiple params
  const handleSort = (newSort: string) => {
    navigate({
      search: (prev) => ({ ...prev, sortBy: newSort, page: 1 }),
    });
  };

  // Reset all params
  const handleReset = () => {
    navigate({
      search: { page: 1, pageSize: 20, sortBy: 'createdAt', sortOrder: 'desc' },
    });
  };

  return (
    <div>
      {/* Link with search params */}
      <Link
        to="/users"
        search={{ page: page + 1, pageSize: 20, sortBy, sortOrder: 'desc' }}
      >
        Next Page
      </Link>

      {/* Preserve current search while changing one param */}
      <Link
        to="/users"
        search={(prev) => ({ ...prev, sortBy: 'name' })}
      >
        Sort by Name
      </Link>

      <button onClick={handleNextPage}>Next</button>
      <button onClick={() => handleSort('email')}>Sort by Email</button>
      <button onClick={handleReset}>Reset</button>
    </div>
  );
}
```

### Complex Search Parameter Types

```tsx
import { createFileRoute } from '@tanstack/react-router';
import { z } from 'zod';

// Complex search schema with arrays and nested objects
const searchSchema = z.object({
  // Array of values
  tags: z.array(z.string()).default([]),

  // Boolean flags
  showArchived: z.boolean().default(false),

  // Date handling
  startDate: z.string().datetime().optional(),
  endDate: z.string().datetime().optional(),

  // Enum
  status: z.enum(['all', 'active', 'inactive', 'pending']).default('all'),

  // Nullable
  categoryId: z.string().nullable().default(null),

  // With custom parsing
  priceRange: z.object({
    min: z.number().min(0).default(0),
    max: z.number().max(10000).default(10000),
  }).default({ min: 0, max: 10000 }),
});

export const Route = createFileRoute('/products')({
  validateSearch: searchSchema,
  component: ProductsPage,

  // Use search in loader
  loaderDeps: ({ search }) => ({ search }),
  loader: async ({ deps: { search } }) => {
    return fetchProducts(search);
  },
});

function ProductsPage() {
  const search = Route.useSearch();
  const products = Route.useLoaderData();

  return (
    <div>
      <h1>Products</h1>
      <p>Tags: {search.tags.join(', ')}</p>
      <p>Price: ${search.priceRange.min} - ${search.priceRange.max}</p>
      <p>Status: {search.status}</p>
    </div>
  );
}
```

### Search Parameter Inheritance

```tsx
// Parent route with search params
// src/routes/shop.tsx
const shopSearchSchema = z.object({
  currency: z.enum(['USD', 'EUR', 'GBP']).default('USD'),
});

export const Route = createFileRoute('/shop')({
  validateSearch: shopSearchSchema,
  component: ShopLayout,
});

// Child route extending parent search
// src/routes/shop/products.tsx
const productsSearchSchema = z.object({
  // Include parent search params
  currency: z.enum(['USD', 'EUR', 'GBP']).default('USD'),
  // Add child-specific params
  category: z.string().optional(),
  sort: z.string().default('popular'),
});

export const Route = createFileRoute('/shop/products')({
  validateSearch: productsSearchSchema,
  component: ProductsPage,
});
```

---

## Route Loaders

### Basic Loader

```tsx
// src/routes/users/index.tsx
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/users/')({
  loader: async () => {
    const response = await fetch('/api/users');
    return response.json();
  },
  component: UsersPage,
});

function UsersPage() {
  const users = Route.useLoaderData();

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Loader with Parameters

```tsx
// src/routes/users/$userId.tsx
import { createFileRoute, notFound } from '@tanstack/react-router';

export const Route = createFileRoute('/users/$userId')({
  loader: async ({ params, context, abortController }) => {
    const response = await fetch(`/api/users/${params.userId}`, {
      signal: abortController.signal,
    });

    if (!response.ok) {
      if (response.status === 404) {
        throw notFound();
      }
      throw new Error('Failed to fetch user');
    }

    return response.json();
  },
  component: UserPage,
});
```

### Loader Dependencies

```tsx
// Re-run loader when search params change
export const Route = createFileRoute('/users/')({
  // Define which values should trigger loader re-run
  loaderDeps: ({ search }) => ({
    page: search.page,
    search: search.search,
    sort: search.sortBy,
  }),

  loader: async ({ deps }) => {
    const { page, search, sort } = deps;
    return fetchUsers({ page, search, sort });
  },

  component: UsersPage,
});
```

### Loader with TanStack Query Integration

```tsx
// src/routes/users/$userId.tsx
import { createFileRoute } from '@tanstack/react-router';
import { queryOptions } from '@tanstack/react-query';

// Define query options
const userQueryOptions = (userId: string) =>
  queryOptions({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });

export const Route = createFileRoute('/users/$userId')({
  loader: async ({ params, context }) => {
    // context.queryClient is available if passed in router context
    return context.queryClient.ensureQueryData(
      userQueryOptions(params.userId)
    );
  },
  component: UserPage,
});

function UserPage() {
  const { userId } = Route.useParams();

  // Use the same query options for reactive updates
  const { data: user } = useQuery(userQueryOptions(userId));

  return <UserProfile user={user} />;
}
```

### Parallel Data Loading

```tsx
export const Route = createFileRoute('/dashboard')({
  loader: async ({ context }) => {
    // Load multiple resources in parallel
    const [user, stats, notifications, recentActivity] = await Promise.all([
      fetchUser(context.userId),
      fetchStats(context.userId),
      fetchNotifications(context.userId),
      fetchRecentActivity(context.userId),
    ]);

    return { user, stats, notifications, recentActivity };
  },
  component: DashboardPage,
});

function DashboardPage() {
  const { user, stats, notifications, recentActivity } = Route.useLoaderData();

  return (
    <div className="grid grid-cols-2 gap-4">
      <UserCard user={user} />
      <StatsCard stats={stats} />
      <NotificationsCard notifications={notifications} />
      <ActivityFeed activities={recentActivity} />
    </div>
  );
}
```

### Loader Caching and Stale Time

```tsx
export const Route = createFileRoute('/products')({
  // Data is considered fresh for 30 seconds
  staleTime: 30_000,

  // Preload data stays fresh for 60 seconds
  preloadStaleTime: 60_000,

  loader: async () => {
    return fetchProducts();
  },

  component: ProductsPage,
});
```

### Defer and Streaming (Awaited)

```tsx
import { createFileRoute, defer, Await } from '@tanstack/react-router';
import { Suspense } from 'react';

export const Route = createFileRoute('/dashboard')({
  loader: async () => {
    // Critical data - await immediately
    const user = await fetchUser();

    // Non-critical data - defer loading
    const recommendations = fetchRecommendations();
    const analytics = fetchAnalytics();

    return {
      user,
      recommendations: defer(recommendations),
      analytics: defer(analytics),
    };
  },
  component: DashboardPage,
});

function DashboardPage() {
  const { user, recommendations, analytics } = Route.useLoaderData();

  return (
    <div>
      {/* User data is immediately available */}
      <UserProfile user={user} />

      {/* Recommendations stream in */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <Await promise={recommendations}>
          {(data) => <RecommendationsList items={data} />}
        </Await>
      </Suspense>

      {/* Analytics stream in */}
      <Suspense fallback={<AnalyticsSkeleton />}>
        <Await promise={analytics}>
          {(data) => <AnalyticsCard data={data} />}
        </Await>
      </Suspense>
    </div>
  );
}
```

---

## Route Actions

TanStack Router supports route actions for handling form submissions and mutations.

### Basic Action Pattern

```tsx
// src/routes/contact.tsx
import { createFileRoute, useRouter } from '@tanstack/react-router';
import { useState } from 'react';

export const Route = createFileRoute('/contact')({
  component: ContactPage,
});

function ContactPage() {
  const router = useRouter();
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setIsSubmitting(true);
    setError(null);

    const formData = new FormData(e.currentTarget);

    try {
      const response = await fetch('/api/contact', {
        method: 'POST',
        body: JSON.stringify(Object.fromEntries(formData)),
        headers: { 'Content-Type': 'application/json' },
      });

      if (!response.ok) {
        throw new Error('Failed to submit form');
      }

      // Navigate to success page
      router.navigate({ to: '/contact/success' });
    } catch (err) {
      setError(err.message);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" required />
      <input name="email" type="email" required />
      <textarea name="message" required />
      {error && <p className="text-red-500">{error}</p>}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Sending...' : 'Send'}
      </button>
    </form>
  );
}
```

### Actions with TanStack Query Mutations

```tsx
// src/routes/_authenticated/users/$userId/edit.tsx
import { createFileRoute } from '@tanstack/react-router';
import { useMutation, useQueryClient } from '@tanstack/react-query';

export const Route = createFileRoute('/_authenticated/users/$userId/edit')({
  loader: async ({ params }) => fetchUser(params.userId),
  component: EditUserPage,
});

function EditUserPage() {
  const { userId } = Route.useParams();
  const user = Route.useLoaderData();
  const navigate = useNavigate();
  const queryClient = useQueryClient();

  const updateMutation = useMutation({
    mutationFn: (data: UpdateUserInput) => updateUser(userId, data),
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['users', userId] });
      queryClient.invalidateQueries({ queryKey: ['users'] });

      // Navigate back to user profile
      navigate({ to: '/users/$userId', params: { userId } });
    },
  });

  const handleSubmit = (data: UpdateUserInput) => {
    updateMutation.mutate(data);
  };

  return (
    <div>
      <h1>Edit User</h1>
      <UserForm
        defaultValues={user}
        onSubmit={handleSubmit}
        isSubmitting={updateMutation.isPending}
        error={updateMutation.error?.message}
      />
    </div>
  );
}
```

### Delete Action Pattern

```tsx
function DeleteUserButton({ userId }: { userId: string }) {
  const navigate = useNavigate();
  const queryClient = useQueryClient();

  const deleteMutation = useMutation({
    mutationFn: () => deleteUser(userId),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
      navigate({ to: '/users' });
    },
  });

  const handleDelete = async () => {
    if (window.confirm('Are you sure you want to delete this user?')) {
      deleteMutation.mutate();
    }
  };

  return (
    <button
      onClick={handleDelete}
      disabled={deleteMutation.isPending}
      className="text-red-500"
    >
      {deleteMutation.isPending ? 'Deleting...' : 'Delete User'}
    </button>
  );
}
```

---

## Navigation (Link, useNavigate)

### Link Component

```tsx
import { Link } from '@tanstack/react-router';

function Navigation() {
  return (
    <nav>
      {/* Basic link */}
      <Link to="/">Home</Link>

      {/* With type-safe params */}
      <Link to="/users/$userId" params={{ userId: '123' }}>
        User Profile
      </Link>

      {/* With search params */}
      <Link to="/users" search={{ page: 1, sort: 'name' }}>
        Users
      </Link>

      {/* Preserve/modify existing search */}
      <Link to="/users" search={(prev) => ({ ...prev, page: 2 })}>
        Page 2
      </Link>

      {/* With hash */}
      <Link to="/docs" hash="installation">
        Installation
      </Link>

      {/* External link */}
      <a href="https://example.com" target="_blank" rel="noopener noreferrer">
        External
      </a>
    </nav>
  );
}
```

### Link Styling and Active States

```tsx
import { Link } from '@tanstack/react-router';

function NavLink({ to, children, ...props }) {
  return (
    <Link
      to={to}
      {...props}
      className="px-4 py-2 rounded-md"
      activeProps={{
        className: 'bg-blue-500 text-white',
      }}
      inactiveProps={{
        className: 'text-gray-600 hover:bg-gray-100',
      }}
    >
      {children}
    </Link>
  );
}

// With render props for more control
function CustomNavLink({ to, children }) {
  return (
    <Link to={to}>
      {({ isActive, isTransitioning }) => (
        <span
          className={cn(
            'px-4 py-2',
            isActive && 'font-bold text-blue-500',
            isTransitioning && 'opacity-50'
          )}
        >
          {children}
          {isActive && ' (current)'}
        </span>
      )}
    </Link>
  );
}
```

### Link Preloading

```tsx
<Link
  to="/users/$userId"
  params={{ userId: '123' }}
  preload="intent"  // Preload on hover/focus
>
  View User
</Link>

<Link
  to="/dashboard"
  preload="viewport" // Preload when visible
>
  Dashboard
</Link>

<Link
  to="/heavy-page"
  preload={false} // Disable preloading
>
  Heavy Page
</Link>
```

### useNavigate Hook

```tsx
import { useNavigate } from '@tanstack/react-router';

function LoginForm() {
  const navigate = useNavigate();

  const handleLogin = async (credentials) => {
    await login(credentials);

    // Simple navigation
    navigate({ to: '/dashboard' });

    // With params
    navigate({
      to: '/users/$userId',
      params: { userId: '123' },
    });

    // With search params
    navigate({
      to: '/users',
      search: { page: 1, filter: 'active' },
    });

    // Replace history (no back button)
    navigate({
      to: '/dashboard',
      replace: true,
    });

    // Relative navigation
    navigate({
      to: '..',        // Go up one level
    });

    // With state
    navigate({
      to: '/checkout',
      state: { fromCart: true },
    });
  };

  return <form onSubmit={handleLogin}>{/* ... */}</form>;
}
```

### useRouter Hook

```tsx
import { useRouter } from '@tanstack/react-router';

function NavigationButtons() {
  const router = useRouter();

  return (
    <div>
      {/* Go back in history */}
      <button onClick={() => router.history.back()}>
        Back
      </button>

      {/* Go forward in history */}
      <button onClick={() => router.history.forward()}>
        Forward
      </button>

      {/* Invalidate and reload current route */}
      <button onClick={() => router.invalidate()}>
        Refresh Data
      </button>

      {/* Access current location */}
      <p>Current path: {router.state.location.pathname}</p>
    </div>
  );
}
```

### useLocation Hook

```tsx
import { useLocation } from '@tanstack/react-router';

function LocationInfo() {
  const location = useLocation();

  return (
    <div>
      <p>Pathname: {location.pathname}</p>
      <p>Search: {JSON.stringify(location.search)}</p>
      <p>Hash: {location.hash}</p>
      <p>State: {JSON.stringify(location.state)}</p>
    </div>
  );
}
```

### useParams Hook

```tsx
import { useParams } from '@tanstack/react-router';

// Type-safe params from specific route
function UserPage() {
  const { userId } = Route.useParams();
  return <div>User: {userId}</div>;
}

// Generic useParams (less type-safe)
function GenericComponent() {
  const params = useParams({ strict: false });
  return <div>Params: {JSON.stringify(params)}</div>;
}
```

---

## Nested Routes and Outlets

### Understanding Outlets

The `<Outlet />` component renders child routes within parent route components.

```tsx
// src/routes/__root.tsx
import { createRootRoute, Outlet } from '@tanstack/react-router';

export const Route = createRootRoute({
  component: () => (
    <div className="app">
      <header>App Header</header>
      <main>
        <Outlet /> {/* Child routes render here */}
      </main>
      <footer>App Footer</footer>
    </div>
  ),
});
```

### Nested Route Structure

```
src/routes/
├── __root.tsx          # Root layout with <Outlet />
├── users.tsx           # /users layout with <Outlet />
└── users/
    ├── index.tsx       # /users (list)
    └── $userId.tsx     # /users/:userId
```

```tsx
// src/routes/users.tsx
import { createFileRoute, Outlet } from '@tanstack/react-router';

export const Route = createFileRoute('/users')({
  component: UsersLayout,
});

function UsersLayout() {
  return (
    <div className="users-layout">
      <aside>
        <h2>Users Navigation</h2>
        <nav>
          <Link to="/users">All Users</Link>
          <Link to="/users/new">Add User</Link>
        </nav>
      </aside>
      <section className="users-content">
        <Outlet /> {/* /users/*, /users/:userId render here */}
      </section>
    </div>
  );
}
```

### Deep Nesting Example

```tsx
// URL: /organizations/:orgId/projects/:projectId/tasks/:taskId

// src/routes/organizations/$orgId.tsx
function OrgLayout() {
  const { orgId } = Route.useParams();
  const org = Route.useLoaderData();

  return (
    <div>
      <OrgHeader org={org} />
      <Outlet />
    </div>
  );
}

// src/routes/organizations/$orgId/projects/$projectId.tsx
function ProjectLayout() {
  const { projectId } = Route.useParams();
  const project = Route.useLoaderData();

  return (
    <div>
      <ProjectSidebar project={project} />
      <div className="project-content">
        <Outlet />
      </div>
    </div>
  );
}

// src/routes/organizations/$orgId/projects/$projectId/tasks/$taskId.tsx
function TaskPage() {
  const { taskId } = Route.useParams();
  const task = Route.useLoaderData();

  return <TaskDetail task={task} />;
}
```

### Accessing Parent Data in Child Routes

```tsx
// Child route can access parent loader data
// src/routes/users/$userId/posts.tsx
import { createFileRoute, useRouteContext } from '@tanstack/react-router';
import { Route as UserRoute } from '../$userId';

export const Route = createFileRoute('/users/$userId/posts')({
  component: UserPostsPage,
});

function UserPostsPage() {
  // Get parent route's loader data
  const user = UserRoute.useLoaderData();
  const posts = Route.useLoaderData();

  return (
    <div>
      <h1>Posts by {user.name}</h1>
      <PostList posts={posts} />
    </div>
  );
}
```

---

## Layout Routes

Layout routes wrap child routes without adding to the URL path.

### Basic Layout Route

```tsx
// src/routes/_authenticated.tsx (note the underscore prefix)
import { createFileRoute, Outlet, redirect } from '@tanstack/react-router';

export const Route = createFileRoute('/_authenticated')({
  beforeLoad: ({ context }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({ to: '/login' });
    }
  },
  component: AuthenticatedLayout,
});

function AuthenticatedLayout() {
  return (
    <div className="authenticated-layout">
      <Header />
      <div className="flex">
        <Sidebar />
        <main className="flex-1 p-6">
          <Outlet />
        </main>
      </div>
    </div>
  );
}
```

### Children of Layout Routes

```
src/routes/
├── _authenticated.tsx           # Layout (no URL)
├── _authenticated/
│   ├── dashboard.tsx           # /dashboard
│   ├── settings.tsx            # /settings
│   └── profile.tsx             # /profile
```

```tsx
// src/routes/_authenticated/dashboard.tsx
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/_authenticated/dashboard')({
  component: DashboardPage,
});

function DashboardPage() {
  // This renders inside AuthenticatedLayout's <Outlet />
  return (
    <div>
      <h1>Dashboard</h1>
      {/* Dashboard content */}
    </div>
  );
}
```

### Multiple Layout Routes

```
src/routes/
├── _public.tsx                  # Public layout
├── _public/
│   ├── about.tsx               # /about
│   └── pricing.tsx             # /pricing
├── _authenticated.tsx          # Auth layout
├── _authenticated/
│   └── dashboard.tsx           # /dashboard
├── _admin.tsx                  # Admin layout (nested in _authenticated)
└── _admin/
    ├── users.tsx               # /admin/users
    └── settings.tsx            # /admin/settings
```

### Nested Layout Routes

```tsx
// src/routes/_authenticated/_admin.tsx
import { createFileRoute, Outlet, redirect } from '@tanstack/react-router';

export const Route = createFileRoute('/_authenticated/_admin')({
  beforeLoad: ({ context }) => {
    if (!context.auth.user?.isAdmin) {
      throw redirect({ to: '/dashboard' });
    }
  },
  component: AdminLayout,
});

function AdminLayout() {
  return (
    <div className="admin-layout">
      <AdminSidebar />
      <div className="admin-content">
        <Outlet />
      </div>
    </div>
  );
}
```

### Route Groups (Parentheses)

Route groups organize routes without affecting the URL or adding layouts.

```
src/routes/
├── (marketing)/                 # Group only - no URL change, no layout
│   ├── about.tsx               # /about
│   ├── pricing.tsx             # /pricing
│   └── features.tsx            # /features
├── (auth)/
│   ├── login.tsx               # /login
│   └── register.tsx            # /register
```

---

## Error Boundaries

### Route-level Error Handling

```tsx
// src/routes/users/$userId.tsx
import { createFileRoute, ErrorComponent, notFound } from '@tanstack/react-router';

export const Route = createFileRoute('/users/$userId')({
  loader: async ({ params }) => {
    const user = await fetchUser(params.userId);
    if (!user) {
      throw notFound();
    }
    return user;
  },

  component: UserPage,

  // Error component for this route
  errorComponent: UserErrorComponent,

  // Not found component
  notFoundComponent: UserNotFound,

  // Pending/loading component
  pendingComponent: UserLoading,
});

function UserErrorComponent({ error, reset }: ErrorComponentProps) {
  return (
    <div className="p-4 bg-red-50 border border-red-200 rounded-lg">
      <h2 className="text-xl font-bold text-red-700">Something went wrong</h2>
      <p className="text-red-600">{error.message}</p>
      <button
        onClick={reset}
        className="mt-4 px-4 py-2 bg-red-500 text-white rounded"
      >
        Try Again
      </button>
    </div>
  );
}

function UserNotFound() {
  return (
    <div className="p-4 text-center">
      <h2 className="text-xl font-bold">User Not Found</h2>
      <p className="text-gray-600">The user you're looking for doesn't exist.</p>
      <Link to="/users" className="text-blue-500">
        Back to Users
      </Link>
    </div>
  );
}

function UserLoading() {
  return (
    <div className="p-4 animate-pulse">
      <div className="h-8 bg-gray-200 rounded w-1/4 mb-4" />
      <div className="h-4 bg-gray-200 rounded w-1/2 mb-2" />
      <div className="h-4 bg-gray-200 rounded w-1/3" />
    </div>
  );
}
```

### Global Error Component

```tsx
// src/router.ts
import { createRouter, ErrorComponent } from '@tanstack/react-router';

export const router = createRouter({
  routeTree,

  // Default error component for all routes
  defaultErrorComponent: ({ error }) => (
    <div className="min-h-screen flex items-center justify-center">
      <div className="text-center">
        <h1 className="text-2xl font-bold text-red-500">Error</h1>
        <p className="text-gray-600">{error.message}</p>
      </div>
    </div>
  ),

  // Default not found
  defaultNotFoundComponent: () => (
    <div className="min-h-screen flex items-center justify-center">
      <div className="text-center">
        <h1 className="text-4xl font-bold">404</h1>
        <p className="text-gray-600">Page not found</p>
        <Link to="/" className="text-blue-500">Go home</Link>
      </div>
    </div>
  ),
});
```

### Throwing Custom Errors

```tsx
export const Route = createFileRoute('/users/$userId')({
  loader: async ({ params }) => {
    const response = await fetch(`/api/users/${params.userId}`);

    if (response.status === 404) {
      throw notFound();
    }

    if (response.status === 403) {
      throw new Error('You do not have permission to view this user');
    }

    if (!response.ok) {
      throw new Error(`Failed to load user: ${response.statusText}`);
    }

    return response.json();
  },
});
```

### Error Recovery

```tsx
function ErrorComponent({ error, reset }: ErrorComponentProps) {
  const router = useRouter();

  const handleRetry = () => {
    // Reset error boundary state
    reset();
    // Invalidate and reload
    router.invalidate();
  };

  return (
    <div>
      <p>Error: {error.message}</p>
      <button onClick={handleRetry}>Retry</button>
      <Link to="/">Go Home</Link>
    </div>
  );
}
```

---

## Pending States

### Route Pending Component

```tsx
export const Route = createFileRoute('/users')({
  loader: async () => {
    // Slow API call
    return fetchUsers();
  },

  // Show while loading
  pendingComponent: () => (
    <div className="p-4">
      <div className="animate-spin h-8 w-8 border-4 border-blue-500 rounded-full border-t-transparent" />
      <p>Loading users...</p>
    </div>
  ),

  component: UsersPage,
});
```

### Pending Timing Configuration

```tsx
// src/router.ts
const router = createRouter({
  routeTree,

  // Wait 500ms before showing pending UI
  defaultPendingMs: 500,

  // Show pending UI for at least 200ms (prevent flash)
  defaultPendingMinMs: 200,
});

// Per-route configuration
export const Route = createFileRoute('/heavy-page')({
  pendingMs: 100,     // Show loading faster
  pendingMinMs: 500,  // Keep loading visible longer

  loader: () => slowOperation(),
  pendingComponent: LoadingSpinner,
});
```

### useIsPending Hook

```tsx
import { useIsPending, Link } from '@tanstack/react-router';

function NavLink({ to, children }) {
  return (
    <Link to={to}>
      {({ isTransitioning }) => (
        <span className={isTransitioning ? 'opacity-50' : ''}>
          {children}
          {isTransitioning && <Spinner className="ml-2" />}
        </span>
      )}
    </Link>
  );
}

// Global pending indicator
function GlobalPendingIndicator() {
  const isPending = useIsPending();

  if (!isPending) return null;

  return (
    <div className="fixed top-0 left-0 right-0 h-1 bg-blue-500 animate-pulse" />
  );
}
```

### Suspense Integration

```tsx
import { Suspense } from 'react';
import { Await, defer } from '@tanstack/react-router';

export const Route = createFileRoute('/dashboard')({
  loader: () => ({
    // Critical - wait for this
    user: fetchUser(),
    // Non-critical - defer
    stats: defer(fetchStats()),
  }),
});

function DashboardPage() {
  const { user, stats } = Route.useLoaderData();

  return (
    <div>
      <UserHeader user={user} />

      <Suspense fallback={<StatsSkeleton />}>
        <Await promise={stats}>
          {(data) => <StatsCard data={data} />}
        </Await>
      </Suspense>
    </div>
  );
}
```

---

## Code Splitting and Lazy Loading

### Automatic Code Splitting (File-based)

With file-based routing and the Vite plugin, code splitting happens automatically.

```tsx
// Each route file becomes its own chunk
// src/routes/users/$userId.tsx
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/users/$userId')({
  component: UserPage,
});
```

### Lazy Loading Components

```tsx
// src/routes/users/$userId.tsx
import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';

// Lazy load the component
const UserPage = lazy(() => import('@/features/users/UserPage'));

export const Route = createFileRoute('/users/$userId')({
  component: UserPage,
});
```

### Manual Code Splitting (Code-based)

```tsx
import { createRoute, lazyRouteComponent } from '@tanstack/react-router';

const userRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/users/$userId',
  // Lazy load the entire route
  component: lazyRouteComponent(() => import('./pages/UserPage')),
});
```

### Lazy Loading with Route Configuration

```tsx
// src/routes/admin.tsx
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/admin')({
  // Lazy load component
  component: () => import('@/features/admin/AdminDashboard').then(m => m.default),

  // Or use createLazyFileRoute for full lazy loading
});

// Alternative: Split into two files
// src/routes/admin.tsx (configuration)
export const Route = createFileRoute('/admin')({
  validateSearch: searchSchema,
  beforeLoad: authCheck,
});

// src/routes/admin.lazy.tsx (component)
import { createLazyFileRoute } from '@tanstack/react-router';

export const Route = createLazyFileRoute('/admin')({
  component: AdminDashboard,
});
```

### createLazyFileRoute Pattern

```tsx
// src/routes/heavy-feature.tsx (critical path)
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/heavy-feature')({
  // Only critical route configuration here
  beforeLoad: async ({ context }) => {
    // Auth check runs immediately
  },
  validateSearch: searchSchema,
});

// src/routes/heavy-feature.lazy.tsx (lazy loaded)
import { createLazyFileRoute } from '@tanstack/react-router';
import { HeavyFeatureComponent } from '@/features/heavy/HeavyFeature';

export const Route = createLazyFileRoute('/heavy-feature')({
  component: HeavyFeatureComponent,
  pendingComponent: LoadingSkeleton,
  errorComponent: ErrorDisplay,
});
```

### Preloading for Better UX

```tsx
// Router configuration
const router = createRouter({
  routeTree,
  defaultPreload: 'intent', // Preload on hover/focus
});

// Per-link configuration
<Link to="/heavy-page" preload="intent">
  Heavy Page
</Link>

// Programmatic preloading
const router = useRouter();

const handleMouseEnter = () => {
  router.preloadRoute({ to: '/heavy-page' });
};
```

---

## Route Context

Route context allows passing data through the route tree without prop drilling.

### Setting Up Router Context

```tsx
// src/router.ts
import { createRouter } from '@tanstack/react-router';
import { QueryClient } from '@tanstack/react-query';

// Define context type
interface RouterContext {
  queryClient: QueryClient;
  auth: {
    isAuthenticated: boolean;
    user: User | null;
  };
}

export const router = createRouter({
  routeTree,
  context: {
    queryClient: undefined!, // Will be provided in RouterProvider
    auth: undefined!,
  },
});

// Type registration
declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router;
  }
}
```

### Providing Context

```tsx
// src/main.tsx
import { RouterProvider } from '@tanstack/react-router';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { router } from './router';
import { useAuthStore } from './stores/authStore';

const queryClient = new QueryClient();

function App() {
  const auth = useAuthStore();

  return (
    <QueryClientProvider client={queryClient}>
      <RouterProvider
        router={router}
        context={{
          queryClient,
          auth: {
            isAuthenticated: auth.isAuthenticated,
            user: auth.user,
          },
        }}
      />
    </QueryClientProvider>
  );
}
```

### Using Context in Routes

```tsx
// src/routes/__root.tsx
import { createRootRouteWithContext, Outlet } from '@tanstack/react-router';
import type { QueryClient } from '@tanstack/react-query';

interface RouterContext {
  queryClient: QueryClient;
  auth: {
    isAuthenticated: boolean;
    user: User | null;
  };
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: RootComponent,
});

// Access context in beforeLoad
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: ({ context }) => {
    // context is fully typed
    if (!context.auth.isAuthenticated) {
      throw redirect({ to: '/login' });
    }
  },
});

// Access context in loader
export const Route = createFileRoute('/users')({
  loader: async ({ context }) => {
    return context.queryClient.ensureQueryData({
      queryKey: ['users'],
      queryFn: fetchUsers,
    });
  },
});
```

### Extending Context in Child Routes

```tsx
// Parent route extends context
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: async ({ context }) => {
    // Add data to context for child routes
    const permissions = await fetchPermissions(context.auth.user.id);

    return {
      permissions,
    };
  },
});

// Child route accesses extended context
export const Route = createFileRoute('/_authenticated/admin')({
  beforeLoad: ({ context }) => {
    // context now includes permissions from parent
    if (!context.permissions.includes('admin')) {
      throw redirect({ to: '/dashboard' });
    }
  },
});
```

### useRouteContext Hook

```tsx
import { useRouteContext } from '@tanstack/react-router';

function SomeComponent() {
  // Access context anywhere in the app
  const context = useRouteContext({ from: '/_authenticated' });

  return (
    <div>
      <p>User: {context.auth.user?.name}</p>
      <p>Permissions: {context.permissions?.join(', ')}</p>
    </div>
  );
}
```

---

## Guards and Authentication

### Authentication Guard Pattern

```tsx
// src/routes/_authenticated.tsx
import { createFileRoute, Outlet, redirect } from '@tanstack/react-router';

export const Route = createFileRoute('/_authenticated')({
  beforeLoad: async ({ context, location }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({
        to: '/login',
        search: {
          redirect: location.href,
        },
      });
    }
  },
  component: () => <Outlet />,
});
```

### Login Route with Redirect

```tsx
// src/routes/login.tsx
import { createFileRoute, redirect, useNavigate } from '@tanstack/react-router';
import { z } from 'zod';

const searchSchema = z.object({
  redirect: z.string().optional(),
});

export const Route = createFileRoute('/login')({
  validateSearch: searchSchema,

  // Redirect if already logged in
  beforeLoad: ({ context, search }) => {
    if (context.auth.isAuthenticated) {
      throw redirect({
        to: search.redirect ?? '/dashboard',
      });
    }
  },

  component: LoginPage,
});

function LoginPage() {
  const search = Route.useSearch();
  const navigate = useNavigate();

  const handleLogin = async (credentials: Credentials) => {
    await login(credentials);

    // Redirect to original destination or dashboard
    navigate({
      to: search.redirect ?? '/dashboard',
      replace: true,
    });
  };

  return <LoginForm onSubmit={handleLogin} />;
}
```

### Role-based Authorization

```tsx
// src/routes/_authenticated/_admin.tsx
export const Route = createFileRoute('/_authenticated/_admin')({
  beforeLoad: ({ context }) => {
    if (context.auth.user?.role !== 'admin') {
      throw redirect({
        to: '/dashboard',
      });
    }
  },
  component: () => <Outlet />,
});

// More granular permission check
export const Route = createFileRoute('/_authenticated/users')({
  beforeLoad: ({ context }) => {
    const requiredPermissions = ['users:read'];
    const hasPermission = requiredPermissions.every(
      (p) => context.auth.user?.permissions.includes(p)
    );

    if (!hasPermission) {
      throw redirect({ to: '/unauthorized' });
    }
  },
});
```

### Async Authentication Check

```tsx
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: async ({ context }) => {
    // Verify token is still valid
    try {
      const user = await verifySession();

      if (!user) {
        throw redirect({ to: '/login' });
      }

      // Return updated user data for child routes
      return { user };
    } catch (error) {
      // Token expired or invalid
      await logout();
      throw redirect({ to: '/login' });
    }
  },
});
```

### Protecting Individual Routes

```tsx
// src/routes/users/$userId/edit.tsx
export const Route = createFileRoute('/users/$userId/edit')({
  beforeLoad: async ({ context, params }) => {
    // Check if user can edit this specific user
    const canEdit =
      context.auth.user?.id === params.userId ||
      context.auth.user?.role === 'admin';

    if (!canEdit) {
      throw redirect({
        to: '/users/$userId',
        params: { userId: params.userId },
      });
    }
  },
});
```

### Custom Not Authorized Component

```tsx
// src/routes/unauthorized.tsx
import { createFileRoute, Link } from '@tanstack/react-router';

export const Route = createFileRoute('/unauthorized')({
  component: UnauthorizedPage,
});

function UnauthorizedPage() {
  return (
    <div className="min-h-screen flex items-center justify-center">
      <div className="text-center">
        <h1 className="text-4xl font-bold text-red-500">403</h1>
        <h2 className="text-xl mt-2">Access Denied</h2>
        <p className="text-gray-600 mt-2">
          You don't have permission to access this page.
        </p>
        <Link to="/dashboard" className="text-blue-500 mt-4 inline-block">
          Return to Dashboard
        </Link>
      </div>
    </div>
  );
}
```

---

## TypeScript Integration

TanStack Router provides first-class TypeScript support with fully typed routes.

### Type Registration

```tsx
// src/router.ts
import { createRouter } from '@tanstack/react-router';
import { routeTree } from './routeTree.gen';

export const router = createRouter({
  routeTree,
});

// Register router type globally
declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router;
  }
}
```

### Typed Route Parameters

```tsx
// src/routes/users/$userId.tsx
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/users/$userId')({
  component: UserPage,
});

function UserPage() {
  // userId is typed as string
  const { userId } = Route.useParams();

  return <div>User: {userId}</div>;
}

// With parameter parsing
import { z } from 'zod';

const paramsSchema = z.object({
  userId: z.coerce.number(),
});

export const Route = createFileRoute('/users/$userId')({
  params: {
    parse: (params) => paramsSchema.parse(params),
    stringify: (params) => ({ userId: String(params.userId) }),
  },
});

function UserPage() {
  // userId is now typed as number
  const { userId } = Route.useParams();
}
```

### Typed Search Parameters

```tsx
import { createFileRoute } from '@tanstack/react-router';
import { z } from 'zod';

const searchSchema = z.object({
  page: z.number().default(1),
  search: z.string().optional(),
  sort: z.enum(['name', 'date', 'status']).default('date'),
  filters: z.array(z.string()).default([]),
});

type SearchParams = z.infer<typeof searchSchema>;

export const Route = createFileRoute('/users')({
  validateSearch: searchSchema,
  component: UsersPage,
});

function UsersPage() {
  // Fully typed search params
  const { page, search, sort, filters } = Route.useSearch();
  // page: number
  // search: string | undefined
  // sort: 'name' | 'date' | 'status'
  // filters: string[]
}
```

### Typed Loader Data

```tsx
interface User {
  id: string;
  name: string;
  email: string;
}

export const Route = createFileRoute('/users/$userId')({
  loader: async ({ params }): Promise<User> => {
    const response = await fetch(`/api/users/${params.userId}`);
    return response.json();
  },
  component: UserPage,
});

function UserPage() {
  // user is typed as User
  const user = Route.useLoaderData();
}
```

### Typed Context

```tsx
// Define context type
interface RouterContext {
  queryClient: QueryClient;
  auth: {
    isAuthenticated: boolean;
    user: User | null;
    permissions: string[];
  };
}

// Root route with context type
import { createRootRouteWithContext } from '@tanstack/react-router';

export const Route = createRootRouteWithContext<RouterContext>()({
  component: RootComponent,
});

// Context is typed in all routes
export const Route = createFileRoute('/dashboard')({
  beforeLoad: ({ context }) => {
    // context.auth is fully typed
    console.log(context.auth.user?.name);
  },
});
```

### Type-safe Navigation

```tsx
import { Link, useNavigate } from '@tanstack/react-router';

// Link with type checking
<Link
  to="/users/$userId"
  params={{ userId: '123' }} // TypeScript checks this
>
  View User
</Link>

// Error: Property 'invalidParam' does not exist
<Link
  to="/users/$userId"
  params={{ invalidParam: '123' }} // TypeScript error!
>
  View User
</Link>

// useNavigate with types
const navigate = useNavigate();

navigate({
  to: '/users/$userId',
  params: { userId: '123' },
  search: { tab: 'profile' }, // Only if search params are defined for route
});
```

### Generic Components with Route Types

```tsx
import { LinkProps } from '@tanstack/react-router';

// Type-safe link wrapper
function AppLink<TTo extends string>(props: LinkProps<TTo>) {
  return (
    <Link
      {...props}
      className={cn('text-blue-500 hover:underline', props.className)}
    />
  );
}

// Usage - fully typed
<AppLink to="/users/$userId" params={{ userId: '123' }}>
  View User
</AppLink>
```

### Route Type Exports

```tsx
// Export types from route files
// src/routes/users/$userId.tsx
import { createFileRoute } from '@tanstack/react-router';

export const Route = createFileRoute('/users/$userId')({
  loader: fetchUser,
  component: UserPage,
});

// Export loader data type
export type UserLoaderData = Awaited<ReturnType<typeof Route.options.loader>>;

// Export params type
export type UserParams = typeof Route.types.params;

// Export search type (if defined)
export type UserSearch = typeof Route.types.search;
```

---

## Best Practices

### File Organization

```
src/
├── routes/                    # All route files
│   ├── __root.tsx
│   ├── _authenticated.tsx
│   └── ...
├── features/                  # Feature-based organization
│   ├── users/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api.ts
│   │   └── types.ts
│   └── dashboard/
├── components/               # Shared components
├── hooks/                    # Shared hooks
├── lib/                      # Utilities
├── stores/                   # State management
└── router.ts                 # Router configuration
```

### Route File Best Practices

```tsx
// src/routes/users/$userId.tsx
import { createFileRoute, notFound } from '@tanstack/react-router';
import { z } from 'zod';

// 1. Define schemas at the top
const searchSchema = z.object({
  tab: z.enum(['profile', 'posts', 'settings']).default('profile'),
});

// 2. Export the Route object
export const Route = createFileRoute('/users/$userId')({
  // 3. Validation first
  validateSearch: searchSchema,

  // 4. Guards/beforeLoad
  beforeLoad: ({ context }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({ to: '/login' });
    }
  },

  // 5. Loader
  loader: async ({ params }) => {
    const user = await fetchUser(params.userId);
    if (!user) throw notFound();
    return user;
  },

  // 6. Components
  component: UserPage,
  pendingComponent: UserSkeleton,
  errorComponent: UserError,
  notFoundComponent: UserNotFound,
});

// 7. Component implementations below
function UserPage() {
  const user = Route.useLoaderData();
  const { tab } = Route.useSearch();

  return (
    <div>
      <UserHeader user={user} />
      <UserTabs activeTab={tab} />
    </div>
  );
}
```

### Error Handling Best Practices

```tsx
// 1. Use notFound() for missing resources
loader: async ({ params }) => {
  const item = await fetchItem(params.id);
  if (!item) throw notFound();
  return item;
},

// 2. Provide meaningful error components
errorComponent: ({ error, reset }) => (
  <ErrorCard
    title="Failed to load user"
    message={error.message}
    onRetry={reset}
  />
),

// 3. Use type-safe error handling
try {
  await riskyOperation();
} catch (error) {
  if (error instanceof NotFoundError) {
    throw notFound();
  }
  if (error instanceof UnauthorizedError) {
    throw redirect({ to: '/login' });
  }
  throw error; // Re-throw unknown errors
}
```

### Data Loading Best Practices

```tsx
// 1. Use loaderDeps for reactive loading
export const Route = createFileRoute('/users')({
  loaderDeps: ({ search }) => ({
    page: search.page,
    filters: search.filters,
  }),
  loader: ({ deps }) => fetchUsers(deps),
});

// 2. Integrate with TanStack Query for caching
export const Route = createFileRoute('/users/$userId')({
  loader: ({ params, context }) => {
    return context.queryClient.ensureQueryData(
      userQueryOptions(params.userId)
    );
  },
});

// 3. Use defer for non-critical data
export const Route = createFileRoute('/dashboard')({
  loader: async () => ({
    user: await fetchUser(), // Critical - await
    stats: defer(fetchStats()), // Non-critical - defer
    notifications: defer(fetchNotifications()),
  }),
});

// 4. Set appropriate stale times
export const Route = createFileRoute('/users')({
  staleTime: 30_000, // 30 seconds
  preloadStaleTime: 60_000, // 1 minute for preloads
});
```

### Navigation Best Practices

```tsx
// 1. Use Link for declarative navigation
<Link to="/users/$userId" params={{ userId }}>
  View User
</Link>

// 2. Use useNavigate for programmatic navigation
const navigate = useNavigate();

const handleSubmit = async () => {
  await saveData();
  navigate({ to: '/success', replace: true });
};

// 3. Preserve search params when needed
<Link to="/users" search={(prev) => ({ ...prev, page: 2 })}>
  Next Page
</Link>

// 4. Use preloading for better UX
<Link to="/heavy-page" preload="intent">
  Heavy Page
</Link>
```

### Authentication Best Practices

```tsx
// 1. Use layout routes for auth guards
// routes/_authenticated.tsx
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: ({ context, location }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({
        to: '/login',
        search: { redirect: location.href },
      });
    }
  },
});

// 2. Verify tokens on sensitive operations
beforeLoad: async ({ context }) => {
  const isValid = await verifyToken(context.auth.token);
  if (!isValid) {
    await context.auth.logout();
    throw redirect({ to: '/login' });
  }
},

// 3. Check permissions at the route level
beforeLoad: ({ context }) => {
  if (!context.auth.permissions.includes('admin')) {
    throw redirect({ to: '/unauthorized' });
  }
},
```

### Performance Best Practices

```tsx
// 1. Enable preloading
const router = createRouter({
  routeTree,
  defaultPreload: 'intent',
  defaultPreloadStaleTime: 30_000,
});

// 2. Use code splitting with lazy routes
// routes/heavy.tsx (config only)
export const Route = createFileRoute('/heavy')({
  beforeLoad: authCheck,
});

// routes/heavy.lazy.tsx (component)
export const Route = createLazyFileRoute('/heavy')({
  component: HeavyComponent,
});

// 3. Avoid loading unnecessary data
loaderDeps: ({ search }) => ({
  // Only include params that affect the data
  page: search.page,
  // Don't include UI-only params like 'tab'
}),

// 4. Use proper pending states
export const Route = createFileRoute('/users')({
  pendingMs: 500, // Don't show loading for fast responses
  pendingMinMs: 200, // Prevent loading flash
});
```

### TypeScript Best Practices

```tsx
// 1. Always register your router type
declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router;
  }
}

// 2. Use Zod for schema validation
const searchSchema = z.object({
  page: z.number().default(1),
});

export const Route = createFileRoute('/users')({
  validateSearch: searchSchema,
});

// 3. Type your context properly
interface RouterContext {
  queryClient: QueryClient;
  auth: AuthState;
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: Root,
});

// 4. Export types from routes for reuse
export type UserLoaderData = Awaited<ReturnType<typeof Route.options.loader>>;
```

---

## Quick Reference

### File Naming Conventions

| File | URL | Purpose |
|------|-----|---------|
| `__root.tsx` | - | Root layout |
| `index.tsx` | `/` | Index route |
| `about.tsx` | `/about` | Static route |
| `$userId.tsx` | `/:userId` | Dynamic param |
| `_layout.tsx` | - | Pathless layout |
| `(group)/` | - | Route grouping |
| `[...].tsx` | `/*` | Catch-all |

### Essential Hooks

| Hook | Purpose |
|------|---------|
| `Route.useLoaderData()` | Get loader data |
| `Route.useParams()` | Get path params |
| `Route.useSearch()` | Get search params |
| `useNavigate()` | Programmatic navigation |
| `useRouter()` | Router instance |
| `useLocation()` | Current location |
| `useRouteContext()` | Route context |

### Route Options

```tsx
createFileRoute('/path')({
  component: Component,
  errorComponent: ErrorComponent,
  pendingComponent: LoadingComponent,
  notFoundComponent: NotFoundComponent,

  loader: async ({ params, context }) => data,
  loaderDeps: ({ search }) => deps,

  beforeLoad: async ({ context, location }) => void,

  validateSearch: zodSchema,

  staleTime: 30000,
  pendingMs: 500,
  pendingMinMs: 200,
});
```

### Navigation Methods

```tsx
// Link component
<Link to="/users/$userId" params={{ userId: '123' }} search={{ tab: 'profile' }}>
  User
</Link>

// Programmatic
navigate({ to: '/path', params: {}, search: {}, replace: true });

// Router methods
router.navigate({ to: '/path' });
router.invalidate();
router.history.back();
```

---

## Additional Resources

- [TanStack Router Documentation](https://tanstack.com/router/latest)
- [TanStack Router GitHub](https://github.com/tanstack/router)
- [TanStack Query Integration](https://tanstack.com/query/latest)
- [Vite Plugin Documentation](https://tanstack.com/router/latest/docs/framework/react/guide/file-based-routing)
