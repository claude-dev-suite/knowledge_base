# Next.js Routing

Next.js uses a file-system based router where folders and files define routes. This comprehensive guide covers all routing concepts in the Next.js App Router.

---

## Table of Contents

1. [File-System Based Routing](#file-system-based-routing)
2. [Pages and Layouts](#pages-and-layouts)
3. [Dynamic Routes](#dynamic-routes)
4. [Route Parameters](#route-parameters)
5. [Search Params](#search-params)
6. [Navigation](#navigation)
7. [Active Links](#active-links)
8. [Route Groups](#route-groups)
9. [Parallel Routes](#parallel-routes)
10. [Intercepting Routes](#intercepting-routes)
11. [API Routes](#api-routes)
12. [Middleware](#middleware)
13. [Redirects and Rewrites](#redirects-and-rewrites)
14. [Not Found Pages](#not-found-pages)
15. [Error Handling Pages](#error-handling-pages)
16. [Loading UI](#loading-ui)
17. [Best Practices](#best-practices)

---

## File-System Based Routing

Next.js App Router uses a file-system based routing mechanism where the folder structure inside the `app` directory defines your application's URL routes.

### Basic Route Structure

```
app/
├── page.tsx              → /
├── about/page.tsx        → /about
├── blog/page.tsx         → /blog
├── contact/page.tsx      → /contact
└── products/
    ├── page.tsx          → /products
    └── details/page.tsx  → /products/details
```

### Special Files

Next.js provides special files to create UI for each route segment:

| File | Purpose |
|------|---------|
| `page.tsx` | Unique UI for a route (makes route publicly accessible) |
| `layout.tsx` | Shared UI for a segment and its children |
| `loading.tsx` | Loading UI for a segment and its children |
| `error.tsx` | Error UI for a segment and its children |
| `not-found.tsx` | Not found UI for a segment |
| `template.tsx` | Re-rendered layout (new instance on navigation) |
| `default.tsx` | Fallback UI for parallel routes |
| `route.tsx` | Server-side API endpoint |

### Component Hierarchy

When multiple special files exist in a route segment, Next.js renders them in a specific hierarchy:

```tsx
<Layout>
  <Template>
    <ErrorBoundary fallback={<Error />}>
      <Suspense fallback={<Loading />}>
        <ErrorBoundary fallback={<NotFound />}>
          <Page />
        </ErrorBoundary>
      </Suspense>
    </ErrorBoundary>
  </Template>
</Layout>
```

### Colocation

You can safely colocate your own files (components, styles, tests) inside route segments:

```
app/
├── dashboard/
│   ├── page.tsx           # Route file
│   ├── components/        # Colocated components
│   │   ├── Chart.tsx
│   │   └── Stats.tsx
│   ├── lib/               # Colocated utilities
│   │   └── data.ts
│   └── styles/            # Colocated styles
│       └── dashboard.css
```

### Private Folders

Prefix a folder with underscore `_` to opt out of routing:

```
app/
├── _components/           # Not a route
│   ├── Button.tsx
│   └── Modal.tsx
├── _lib/                  # Not a route
│   └── utils.ts
└── dashboard/
    └── page.tsx           # /dashboard
```

---

## Pages and Layouts

### Pages

A page is the unique UI for a route. Pages are Server Components by default:

```tsx
// app/page.tsx - Home page (/)
export default function HomePage() {
  return (
    <main>
      <h1>Welcome to My App</h1>
      <p>This is the home page.</p>
    </main>
  );
}
```

```tsx
// app/blog/page.tsx - Blog listing page (/blog)
import { getPosts } from '@/lib/posts';

export default async function BlogPage() {
  const posts = await getPosts();

  return (
    <section>
      <h1>Blog</h1>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>
            <Link href={`/blog/${post.slug}`}>{post.title}</Link>
          </li>
        ))}
      </ul>
    </section>
  );
}
```

### Root Layout

The root layout is required and wraps all pages:

```tsx
// app/layout.tsx
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';

const inter = Inter({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: 'My Application',
  description: 'Built with Next.js',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <header>
          <nav>{/* Navigation */}</nav>
        </header>
        <main>{children}</main>
        <footer>{/* Footer */}</footer>
      </body>
    </html>
  );
}
```

### Nested Layouts

Create layouts for specific route segments:

```tsx
// app/dashboard/layout.tsx
import { Sidebar } from './components/Sidebar';

export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="dashboard-container">
      <Sidebar />
      <div className="dashboard-content">{children}</div>
    </div>
  );
}
```

### Templates

Templates are similar to layouts but create a new instance on navigation:

```tsx
// app/dashboard/template.tsx
'use client';

import { useEffect } from 'react';

export default function DashboardTemplate({
  children,
}: {
  children: React.ReactNode;
}) {
  useEffect(() => {
    // Runs on every navigation
    console.log('Dashboard mounted');
    return () => console.log('Dashboard unmounted');
  }, []);

  return <div className="template-wrapper">{children}</div>;
}
```

### Layout vs Template

| Feature | Layout | Template |
|---------|--------|----------|
| State preservation | Yes | No |
| Re-renders on navigation | No | Yes |
| Effects run on navigation | No | Yes |
| Use case | Persistent UI | Animations, logging |

---

## Dynamic Routes

Dynamic routes allow you to create routes from dynamic data using brackets notation.

### Single Dynamic Segment [slug]

```tsx
// app/blog/[slug]/page.tsx
// Matches: /blog/hello-world, /blog/my-first-post

interface BlogPostPageProps {
  params: { slug: string };
}

export default function BlogPostPage({ params }: BlogPostPageProps) {
  return (
    <article>
      <h1>Post: {params.slug}</h1>
    </article>
  );
}

// Generate static params for SSG
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then((res) =>
    res.json()
  );

  return posts.map((post: { slug: string }) => ({
    slug: post.slug,
  }));
}
```

### Multiple Dynamic Segments

```tsx
// app/products/[category]/[id]/page.tsx
// Matches: /products/electronics/123, /products/clothing/456

interface ProductPageProps {
  params: {
    category: string;
    id: string;
  };
}

export default async function ProductPage({ params }: ProductPageProps) {
  const product = await getProduct(params.category, params.id);

  return (
    <div>
      <h1>{product.name}</h1>
      <p>Category: {params.category}</p>
      <p>ID: {params.id}</p>
    </div>
  );
}

export async function generateStaticParams() {
  const products = await getProducts();

  return products.map((product) => ({
    category: product.category,
    id: product.id,
  }));
}
```

### Catch-All Segments [...slug]

Catch-all routes match any number of segments:

```tsx
// app/docs/[...slug]/page.tsx
// Matches: /docs/a, /docs/a/b, /docs/a/b/c
// Does NOT match: /docs (requires at least one segment)

interface DocsPageProps {
  params: { slug: string[] };
}

export default function DocsPage({ params }: DocsPageProps) {
  // /docs/getting-started/installation
  // slug = ['getting-started', 'installation']

  const path = params.slug.join('/');

  return (
    <article>
      <h1>Documentation: {path}</h1>
      <nav>
        <ol>
          {params.slug.map((segment, index) => (
            <li key={index}>{segment}</li>
          ))}
        </ol>
      </nav>
    </article>
  );
}

export async function generateStaticParams() {
  return [
    { slug: ['getting-started'] },
    { slug: ['getting-started', 'installation'] },
    { slug: ['api', 'reference'] },
    { slug: ['api', 'reference', 'hooks'] },
  ];
}
```

### Optional Catch-All Segments [[...slug]]

Optional catch-all routes also match the parent route:

```tsx
// app/shop/[[...slug]]/page.tsx
// Matches: /shop, /shop/a, /shop/a/b, /shop/a/b/c

interface ShopPageProps {
  params: { slug?: string[] };
}

export default function ShopPage({ params }: ShopPageProps) {
  // /shop → slug = undefined
  // /shop/electronics → slug = ['electronics']
  // /shop/electronics/phones → slug = ['electronics', 'phones']

  if (!params.slug) {
    return <h1>All Products</h1>;
  }

  return (
    <div>
      <h1>Shop: {params.slug.join(' > ')}</h1>
      <p>Browsing {params.slug.length} levels deep</p>
    </div>
  );
}
```

### Dynamic Segment Comparison

| Pattern | Example Path | params Value |
|---------|--------------|--------------|
| `[slug]` | `/blog/hello` | `{ slug: 'hello' }` |
| `[...slug]` | `/blog/a/b` | `{ slug: ['a', 'b'] }` |
| `[[...slug]]` | `/blog` | `{ slug: undefined }` |
| `[[...slug]]` | `/blog/a/b` | `{ slug: ['a', 'b'] }` |

---

## Route Parameters

Route parameters are passed to page components via the `params` prop.

### Accessing Params in Server Components

```tsx
// app/users/[userId]/posts/[postId]/page.tsx

interface PageProps {
  params: {
    userId: string;
    postId: string;
  };
}

export default async function UserPostPage({ params }: PageProps) {
  const { userId, postId } = params;

  const [user, post] = await Promise.all([
    getUser(userId),
    getPost(postId),
  ]);

  return (
    <article>
      <p>Author: {user.name}</p>
      <h1>{post.title}</h1>
      <div>{post.content}</div>
    </article>
  );
}
```

### Accessing Params in Client Components

```tsx
'use client';

import { useParams } from 'next/navigation';

export function PostActions() {
  const params = useParams<{ userId: string; postId: string }>();

  const handleDelete = async () => {
    await deletePost(params.userId, params.postId);
  };

  return (
    <button onClick={handleDelete}>
      Delete Post {params.postId}
    </button>
  );
}
```

### Accessing Params in Layouts

Layouts receive params for their segment and child segments:

```tsx
// app/[locale]/layout.tsx

interface LocaleLayoutProps {
  children: React.ReactNode;
  params: { locale: string };
}

export default function LocaleLayout({ children, params }: LocaleLayoutProps) {
  return (
    <html lang={params.locale}>
      <body>{children}</body>
    </html>
  );
}
```

### Type-Safe Params with generateStaticParams

```tsx
// app/blog/[slug]/page.tsx

type Params = {
  slug: string;
};

export async function generateStaticParams(): Promise<Params[]> {
  const posts = await getPosts();
  return posts.map((post) => ({ slug: post.slug }));
}

interface PageProps {
  params: Params;
}

export default function BlogPost({ params }: PageProps) {
  return <h1>{params.slug}</h1>;
}
```

---

## Search Params

Search params (query strings) are available in pages and can be accessed differently in Server and Client Components.

### Server Component Access

```tsx
// app/search/page.tsx
// URL: /search?query=nextjs&page=2&sort=date

interface SearchPageProps {
  searchParams: {
    query?: string;
    page?: string;
    sort?: string;
    filters?: string | string[];
  };
}

export default async function SearchPage({ searchParams }: SearchPageProps) {
  const query = searchParams.query || '';
  const page = Number(searchParams.page) || 1;
  const sort = searchParams.sort || 'relevance';

  // Handle array params: ?filters=new&filters=sale
  const filters = Array.isArray(searchParams.filters)
    ? searchParams.filters
    : searchParams.filters
    ? [searchParams.filters]
    : [];

  const results = await search({ query, page, sort, filters });

  return (
    <div>
      <h1>Search Results for "{query}"</h1>
      <p>Page {page} | Sort by {sort}</p>
      <p>Filters: {filters.join(', ')}</p>
      <SearchResults results={results} />
    </div>
  );
}
```

### Client Component Access with useSearchParams

```tsx
'use client';

import { useSearchParams, useRouter, usePathname } from 'next/navigation';
import { useCallback } from 'react';

export function SearchFilters() {
  const searchParams = useSearchParams();
  const pathname = usePathname();
  const router = useRouter();

  // Get current values
  const currentSort = searchParams.get('sort') || 'relevance';
  const currentPage = searchParams.get('page') || '1';

  // Create query string helper
  const createQueryString = useCallback(
    (name: string, value: string) => {
      const params = new URLSearchParams(searchParams.toString());
      params.set(name, value);
      return params.toString();
    },
    [searchParams]
  );

  // Update single param
  const updateParam = (name: string, value: string) => {
    router.push(`${pathname}?${createQueryString(name, value)}`);
  };

  // Update multiple params
  const updateParams = (updates: Record<string, string>) => {
    const params = new URLSearchParams(searchParams.toString());
    Object.entries(updates).forEach(([key, value]) => {
      if (value) {
        params.set(key, value);
      } else {
        params.delete(key);
      }
    });
    router.push(`${pathname}?${params.toString()}`);
  };

  // Remove param
  const removeParam = (name: string) => {
    const params = new URLSearchParams(searchParams.toString());
    params.delete(name);
    router.push(`${pathname}?${params.toString()}`);
  };

  return (
    <div className="filters">
      <select
        value={currentSort}
        onChange={(e) => updateParam('sort', e.target.value)}
      >
        <option value="relevance">Relevance</option>
        <option value="date">Date</option>
        <option value="price-asc">Price: Low to High</option>
        <option value="price-desc">Price: High to Low</option>
      </select>

      <button onClick={() => updateParams({ page: '1', sort: 'relevance' })}>
        Reset Filters
      </button>
    </div>
  );
}
```

### Pagination with Search Params

```tsx
'use client';

import Link from 'next/link';
import { useSearchParams, usePathname } from 'next/navigation';

interface PaginationProps {
  totalPages: number;
}

export function Pagination({ totalPages }: PaginationProps) {
  const searchParams = useSearchParams();
  const pathname = usePathname();
  const currentPage = Number(searchParams.get('page')) || 1;

  const createPageURL = (pageNumber: number) => {
    const params = new URLSearchParams(searchParams.toString());
    params.set('page', pageNumber.toString());
    return `${pathname}?${params.toString()}`;
  };

  return (
    <nav className="pagination">
      <Link
        href={createPageURL(currentPage - 1)}
        className={currentPage <= 1 ? 'disabled' : ''}
        aria-disabled={currentPage <= 1}
      >
        Previous
      </Link>

      {Array.from({ length: totalPages }, (_, i) => i + 1).map((page) => (
        <Link
          key={page}
          href={createPageURL(page)}
          className={currentPage === page ? 'active' : ''}
        >
          {page}
        </Link>
      ))}

      <Link
        href={createPageURL(currentPage + 1)}
        className={currentPage >= totalPages ? 'disabled' : ''}
        aria-disabled={currentPage >= totalPages}
      >
        Next
      </Link>
    </nav>
  );
}
```

### Suspense Boundary for Search Params

When using `useSearchParams`, wrap components in Suspense to prevent the entire page from becoming client-rendered:

```tsx
// app/search/page.tsx
import { Suspense } from 'react';
import { SearchFilters } from './SearchFilters';
import { SearchResults } from './SearchResults';

export default function SearchPage() {
  return (
    <div>
      <h1>Search</h1>
      <Suspense fallback={<div>Loading filters...</div>}>
        <SearchFilters />
      </Suspense>
      <Suspense fallback={<div>Loading results...</div>}>
        <SearchResults />
      </Suspense>
    </div>
  );
}
```

---

## Navigation

Next.js provides multiple ways to navigate between routes.

### Link Component (Recommended)

The `<Link>` component is the primary way to navigate:

```tsx
import Link from 'next/link';

export function Navigation() {
  return (
    <nav>
      {/* Basic link */}
      <Link href="/">Home</Link>

      {/* Dynamic route */}
      <Link href={`/blog/${post.slug}`}>Read More</Link>

      {/* With query params */}
      <Link href="/search?query=nextjs">Search Next.js</Link>

      {/* Object syntax */}
      <Link
        href={{
          pathname: '/blog/[slug]',
          query: { slug: 'my-post' },
        }}
      >
        My Post
      </Link>

      {/* Disable prefetching */}
      <Link href="/heavy-page" prefetch={false}>
        Heavy Page
      </Link>

      {/* Replace history (no back button) */}
      <Link href="/login" replace>
        Login
      </Link>

      {/* Scroll to top disabled */}
      <Link href="/same-page#section" scroll={false}>
        Jump to Section
      </Link>
    </nav>
  );
}
```

### Link with Children Components

```tsx
import Link from 'next/link';
import Image from 'next/image';

export function ProductCard({ product }) {
  return (
    <Link href={`/products/${product.id}`} className="product-card">
      <Image
        src={product.image}
        alt={product.name}
        width={200}
        height={200}
      />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
    </Link>
  );
}
```

### Programmatic Navigation with useRouter

```tsx
'use client';

import { useRouter } from 'next/navigation';

export function NavigationButtons() {
  const router = useRouter();

  return (
    <div>
      {/* Push to history stack */}
      <button onClick={() => router.push('/dashboard')}>
        Go to Dashboard
      </button>

      {/* Replace current history entry */}
      <button onClick={() => router.replace('/login')}>
        Replace with Login
      </button>

      {/* Navigate back */}
      <button onClick={() => router.back()}>
        Go Back
      </button>

      {/* Navigate forward */}
      <button onClick={() => router.forward()}>
        Go Forward
      </button>

      {/* Refresh current route (re-fetch data) */}
      <button onClick={() => router.refresh()}>
        Refresh Data
      </button>

      {/* Prefetch a route */}
      <button onMouseEnter={() => router.prefetch('/about')}>
        Hover to Prefetch About
      </button>
    </div>
  );
}
```

### Navigation with Form Submission

```tsx
'use client';

import { useRouter } from 'next/navigation';
import { FormEvent } from 'react';

export function SearchForm() {
  const router = useRouter();

  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const query = formData.get('query') as string;
    router.push(`/search?query=${encodeURIComponent(query)}`);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" name="query" placeholder="Search..." />
      <button type="submit">Search</button>
    </form>
  );
}
```

### Server-Side Navigation with redirect

```tsx
// In Server Components or Server Actions
import { redirect } from 'next/navigation';

// In a Server Component
export default async function ProtectedPage() {
  const session = await getSession();

  if (!session) {
    redirect('/login');
  }

  return <div>Protected Content</div>;
}

// In a Server Action
async function createPost(formData: FormData) {
  'use server';

  const post = await db.post.create({
    data: {
      title: formData.get('title') as string,
      content: formData.get('content') as string,
    },
  });

  redirect(`/posts/${post.id}`);
}
```

---

## Active Links

Create navigation links that indicate the current active route.

### Basic Active Link

```tsx
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';

interface NavLinkProps {
  href: string;
  children: React.ReactNode;
  exact?: boolean;
}

export function NavLink({ href, children, exact = false }: NavLinkProps) {
  const pathname = usePathname();

  const isActive = exact
    ? pathname === href
    : pathname.startsWith(href);

  return (
    <Link
      href={href}
      className={`nav-link ${isActive ? 'active' : ''}`}
      aria-current={isActive ? 'page' : undefined}
    >
      {children}
    </Link>
  );
}
```

### Navigation with Active States

```tsx
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';

const navItems = [
  { href: '/', label: 'Home', exact: true },
  { href: '/dashboard', label: 'Dashboard' },
  { href: '/blog', label: 'Blog' },
  { href: '/settings', label: 'Settings' },
];

export function MainNav() {
  const pathname = usePathname();

  return (
    <nav className="main-nav">
      <ul>
        {navItems.map((item) => {
          const isActive = item.exact
            ? pathname === item.href
            : pathname.startsWith(item.href);

          return (
            <li key={item.href}>
              <Link
                href={item.href}
                className={isActive ? 'active' : ''}
                aria-current={isActive ? 'page' : undefined}
              >
                {item.label}
              </Link>
            </li>
          );
        })}
      </ul>
    </nav>
  );
}
```

### Sidebar with Nested Active States

```tsx
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';

interface SidebarItem {
  href: string;
  label: string;
  icon: React.ReactNode;
  children?: SidebarItem[];
}

const sidebarItems: SidebarItem[] = [
  { href: '/dashboard', label: 'Overview', icon: <HomeIcon /> },
  {
    href: '/dashboard/analytics',
    label: 'Analytics',
    icon: <ChartIcon />,
    children: [
      { href: '/dashboard/analytics/traffic', label: 'Traffic', icon: null },
      { href: '/dashboard/analytics/revenue', label: 'Revenue', icon: null },
    ],
  },
  { href: '/dashboard/settings', label: 'Settings', icon: <GearIcon /> },
];

export function Sidebar() {
  const pathname = usePathname();

  const renderItem = (item: SidebarItem) => {
    const isActive = pathname === item.href;
    const isParentActive = item.children?.some((child) =>
      pathname.startsWith(child.href)
    );

    return (
      <li key={item.href}>
        <Link
          href={item.href}
          className={`sidebar-link ${isActive ? 'active' : ''} ${
            isParentActive ? 'parent-active' : ''
          }`}
        >
          {item.icon}
          <span>{item.label}</span>
        </Link>
        {item.children && (
          <ul className="sidebar-submenu">
            {item.children.map(renderItem)}
          </ul>
        )}
      </li>
    );
  };

  return (
    <aside className="sidebar">
      <ul>{sidebarItems.map(renderItem)}</ul>
    </aside>
  );
}
```

---

## Route Groups

Route groups allow you to organize routes without affecting the URL structure.

### Basic Route Groups

```
app/
├── (marketing)/              # Group: no URL segment
│   ├── layout.tsx            # Marketing-specific layout
│   ├── page.tsx              # / (home)
│   ├── about/page.tsx        # /about
│   └── blog/page.tsx         # /blog
├── (shop)/
│   ├── layout.tsx            # Shop-specific layout
│   ├── products/page.tsx     # /products
│   └── cart/page.tsx         # /cart
└── (auth)/
    ├── layout.tsx            # Auth-specific layout
    ├── login/page.tsx        # /login
    └── register/page.tsx     # /register
```

### Marketing Layout

```tsx
// app/(marketing)/layout.tsx
import { MarketingNav } from '@/components/MarketingNav';
import { Footer } from '@/components/Footer';

export default function MarketingLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <>
      <MarketingNav />
      <main className="marketing-main">{children}</main>
      <Footer />
    </>
  );
}
```

### Shop Layout with Different Navigation

```tsx
// app/(shop)/layout.tsx
import { ShopNav } from '@/components/ShopNav';
import { CartProvider } from '@/context/CartContext';

export default function ShopLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <CartProvider>
      <ShopNav />
      <main className="shop-main">{children}</main>
    </CartProvider>
  );
}
```

### Auth Layout (Minimal)

```tsx
// app/(auth)/layout.tsx
export default function AuthLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="auth-container">
      <div className="auth-card">{children}</div>
    </div>
  );
}
```

### Multiple Root Layouts

You can have multiple root layouts by creating route groups at the top level:

```
app/
├── (main)/
│   ├── layout.tsx            # Main app layout with html/body
│   └── page.tsx
└── (admin)/
    ├── layout.tsx            # Admin layout with html/body
    └── admin/page.tsx
```

```tsx
// app/(main)/layout.tsx
export default function MainLayout({ children }) {
  return (
    <html lang="en">
      <body className="main-theme">{children}</body>
    </html>
  );
}

// app/(admin)/layout.tsx
export default function AdminLayout({ children }) {
  return (
    <html lang="en">
      <body className="admin-theme">{children}</body>
    </html>
  );
}
```

---

## Parallel Routes

Parallel routes allow you to render multiple pages simultaneously in the same layout.

### Defining Parallel Routes

Use the `@folder` convention to define parallel routes (slots):

```
app/
├── layout.tsx
├── page.tsx
├── @dashboard/
│   ├── page.tsx
│   └── loading.tsx
├── @analytics/
│   ├── page.tsx
│   └── loading.tsx
└── @notifications/
    └── page.tsx
```

### Layout with Parallel Routes

```tsx
// app/layout.tsx
interface LayoutProps {
  children: React.ReactNode;
  dashboard: React.ReactNode;
  analytics: React.ReactNode;
  notifications: React.ReactNode;
}

export default function Layout({
  children,
  dashboard,
  analytics,
  notifications,
}: LayoutProps) {
  return (
    <div className="app-layout">
      <main>{children}</main>
      <aside className="sidebar">
        <section className="dashboard-slot">{dashboard}</section>
        <section className="analytics-slot">{analytics}</section>
        <section className="notifications-slot">{notifications}</section>
      </aside>
    </div>
  );
}
```

### Default Files for Parallel Routes

Use `default.tsx` as a fallback when a slot doesn't match the current route:

```tsx
// app/@analytics/default.tsx
export default function AnalyticsDefault() {
  return <div>No analytics data for this view</div>;
}
```

### Conditional Rendering with Parallel Routes

```tsx
// app/layout.tsx
import { getUser } from '@/lib/auth';

interface LayoutProps {
  children: React.ReactNode;
  admin: React.ReactNode;
  user: React.ReactNode;
}

export default async function Layout({ children, admin, user }: LayoutProps) {
  const currentUser = await getUser();

  return (
    <div>
      {children}
      {currentUser?.role === 'admin' ? admin : user}
    </div>
  );
}
```

### Modal with Parallel Routes

```
app/
├── layout.tsx
├── page.tsx
├── @modal/
│   ├── default.tsx           # Empty when no modal
│   └── (.)photo/[id]/
│       └── page.tsx          # Modal content
└── photo/[id]/
    └── page.tsx              # Full page (direct access)
```

```tsx
// app/layout.tsx
interface LayoutProps {
  children: React.ReactNode;
  modal: React.ReactNode;
}

export default function Layout({ children, modal }: LayoutProps) {
  return (
    <>
      {children}
      {modal}
    </>
  );
}

// app/@modal/default.tsx
export default function Default() {
  return null;
}

// app/@modal/(.)photo/[id]/page.tsx
import { Modal } from '@/components/Modal';

export default function PhotoModal({ params }: { params: { id: string } }) {
  return (
    <Modal>
      <Photo id={params.id} />
    </Modal>
  );
}
```

---

## Intercepting Routes

Intercepting routes allow you to load a route from another part of your application while keeping the current layout context.

### Convention

| Convention | Description |
|------------|-------------|
| `(.)` | Match segments on the same level |
| `(..)` | Match segments one level above |
| `(..)(..)` | Match segments two levels above |
| `(...)` | Match segments from the root |

### Photo Gallery Modal Example

```
app/
├── feed/
│   ├── page.tsx
│   └── @modal/
│       ├── default.tsx
│       └── (.)photo/[id]/
│           └── page.tsx      # Intercepts /photo/[id] when navigating from /feed
└── photo/[id]/
    └── page.tsx              # Full page when accessed directly
```

```tsx
// app/feed/page.tsx
import Link from 'next/link';

export default function FeedPage() {
  const photos = getPhotos();

  return (
    <div className="photo-grid">
      {photos.map((photo) => (
        <Link key={photo.id} href={`/photo/${photo.id}`}>
          <img src={photo.thumbnail} alt={photo.title} />
        </Link>
      ))}
    </div>
  );
}

// app/feed/@modal/(.)photo/[id]/page.tsx
'use client';

import { useRouter } from 'next/navigation';

export default function PhotoModal({ params }: { params: { id: string } }) {
  const router = useRouter();

  return (
    <div className="modal-overlay" onClick={() => router.back()}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <img src={`/photos/${params.id}.jpg`} alt="Photo" />
        <button onClick={() => router.back()}>Close</button>
      </div>
    </div>
  );
}

// app/photo/[id]/page.tsx (full page version)
export default function PhotoPage({ params }: { params: { id: string } }) {
  return (
    <div className="photo-page">
      <img src={`/photos/${params.id}.jpg`} alt="Photo" />
      <div className="photo-details">
        {/* Full photo details */}
      </div>
    </div>
  );
}
```

### Login Modal Interception

```
app/
├── layout.tsx
├── @auth/
│   ├── default.tsx
│   └── (.)login/
│       └── page.tsx          # Shows as modal when navigating
└── login/
    └── page.tsx              # Full page when accessed directly
```

```tsx
// app/@auth/(.)login/page.tsx
'use client';

import { useRouter } from 'next/navigation';
import { LoginForm } from '@/components/LoginForm';

export default function LoginModal() {
  const router = useRouter();

  return (
    <div className="modal-backdrop">
      <div className="modal">
        <button onClick={() => router.back()} className="close-btn">
          X
        </button>
        <LoginForm />
      </div>
    </div>
  );
}
```

---

## API Routes

API routes in the App Router use Route Handlers defined in `route.ts` files.

### Basic Route Handler

```tsx
// app/api/hello/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  return NextResponse.json({ message: 'Hello, World!' });
}
```

### HTTP Methods

```tsx
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server';

// GET /api/posts
export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const page = searchParams.get('page') || '1';
  const limit = searchParams.get('limit') || '10';

  const posts = await db.post.findMany({
    skip: (Number(page) - 1) * Number(limit),
    take: Number(limit),
  });

  return NextResponse.json(posts);
}

// POST /api/posts
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();

    const post = await db.post.create({
      data: {
        title: body.title,
        content: body.content,
        authorId: body.authorId,
      },
    });

    return NextResponse.json(post, { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to create post' },
      { status: 500 }
    );
  }
}
```

### Dynamic Route Handlers

```tsx
// app/api/posts/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

interface RouteParams {
  params: { id: string };
}

// GET /api/posts/:id
export async function GET(request: NextRequest, { params }: RouteParams) {
  const post = await db.post.findUnique({
    where: { id: params.id },
  });

  if (!post) {
    return NextResponse.json(
      { error: 'Post not found' },
      { status: 404 }
    );
  }

  return NextResponse.json(post);
}

// PUT /api/posts/:id
export async function PUT(request: NextRequest, { params }: RouteParams) {
  const body = await request.json();

  const post = await db.post.update({
    where: { id: params.id },
    data: body,
  });

  return NextResponse.json(post);
}

// PATCH /api/posts/:id
export async function PATCH(request: NextRequest, { params }: RouteParams) {
  const body = await request.json();

  const post = await db.post.update({
    where: { id: params.id },
    data: body,
  });

  return NextResponse.json(post);
}

// DELETE /api/posts/:id
export async function DELETE(request: NextRequest, { params }: RouteParams) {
  await db.post.delete({
    where: { id: params.id },
  });

  return new NextResponse(null, { status: 204 });
}
```

### Request Helpers

```tsx
// app/api/upload/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  // Get headers
  const contentType = request.headers.get('content-type');
  const authorization = request.headers.get('authorization');

  // Get cookies
  const token = request.cookies.get('token');

  // Handle FormData
  if (contentType?.includes('multipart/form-data')) {
    const formData = await request.formData();
    const file = formData.get('file') as File;
    const name = formData.get('name') as string;

    // Process file...
    const bytes = await file.arrayBuffer();
    const buffer = Buffer.from(bytes);

    return NextResponse.json({
      filename: file.name,
      size: file.size,
    });
  }

  // Handle JSON
  const body = await request.json();
  return NextResponse.json(body);
}
```

### Response Helpers

```tsx
// app/api/examples/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  // JSON response
  return NextResponse.json({ data: 'value' });

  // With status code
  return NextResponse.json({ error: 'Not found' }, { status: 404 });

  // With headers
  return NextResponse.json(
    { data: 'value' },
    {
      headers: {
        'Cache-Control': 'public, max-age=3600',
        'X-Custom-Header': 'value',
      },
    }
  );

  // Redirect
  return NextResponse.redirect(new URL('/new-path', request.url));

  // Rewrite
  return NextResponse.rewrite(new URL('/proxy-path', request.url));

  // Stream response
  const stream = new ReadableStream({
    start(controller) {
      controller.enqueue('Hello');
      controller.enqueue(' World');
      controller.close();
    },
  });

  return new NextResponse(stream, {
    headers: { 'Content-Type': 'text/plain' },
  });
}
```

### CORS Configuration

```tsx
// app/api/cors/route.ts
import { NextRequest, NextResponse } from 'next/server';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization',
};

export async function OPTIONS() {
  return NextResponse.json({}, { headers: corsHeaders });
}

export async function GET() {
  return NextResponse.json(
    { data: 'value' },
    { headers: corsHeaders }
  );
}
```

### Route Segment Config

```tsx
// app/api/revalidate/route.ts

// Force dynamic rendering
export const dynamic = 'force-dynamic';

// Or force static
export const dynamic = 'force-static';

// Revalidation time
export const revalidate = 60; // seconds

// Runtime
export const runtime = 'edge'; // or 'nodejs'

// Max duration (for edge: 30s, serverless: varies by plan)
export const maxDuration = 30;
```

---

## Middleware

Middleware runs before a request is completed, allowing you to modify the response.

### Basic Middleware

```tsx
// middleware.ts (root of project)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Clone the response
  const response = NextResponse.next();

  // Add custom header
  response.headers.set('x-middleware-timestamp', Date.now().toString());

  return response;
}

// Configure which paths middleware runs on
export const config = {
  matcher: '/((?!api|_next/static|_next/image|favicon.ico).*)',
};
```

### Authentication Middleware

```tsx
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { verifyToken } from '@/lib/auth';

const protectedRoutes = ['/dashboard', '/settings', '/profile'];
const authRoutes = ['/login', '/register'];

export async function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token')?.value;
  const pathname = request.nextUrl.pathname;

  // Check if route is protected
  const isProtectedRoute = protectedRoutes.some((route) =>
    pathname.startsWith(route)
  );

  // Check if route is auth route
  const isAuthRoute = authRoutes.some((route) =>
    pathname.startsWith(route)
  );

  // Verify token
  const isAuthenticated = token ? await verifyToken(token) : false;

  // Redirect unauthenticated users from protected routes
  if (isProtectedRoute && !isAuthenticated) {
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('redirect', pathname);
    return NextResponse.redirect(loginUrl);
  }

  // Redirect authenticated users from auth routes
  if (isAuthRoute && isAuthenticated) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
};
```

### Internationalization Middleware

```tsx
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

const locales = ['en', 'de', 'fr', 'es'];
const defaultLocale = 'en';

function getLocale(request: NextRequest): string {
  // Check cookie
  const cookieLocale = request.cookies.get('NEXT_LOCALE')?.value;
  if (cookieLocale && locales.includes(cookieLocale)) {
    return cookieLocale;
  }

  // Check Accept-Language header
  const acceptLanguage = request.headers.get('accept-language');
  if (acceptLanguage) {
    const preferredLocale = acceptLanguage
      .split(',')[0]
      .split('-')[0]
      .toLowerCase();
    if (locales.includes(preferredLocale)) {
      return preferredLocale;
    }
  }

  return defaultLocale;
}

export function middleware(request: NextRequest) {
  const pathname = request.nextUrl.pathname;

  // Check if pathname already has locale
  const pathnameHasLocale = locales.some(
    (locale) =>
      pathname.startsWith(`/${locale}/`) || pathname === `/${locale}`
  );

  if (pathnameHasLocale) return NextResponse.next();

  // Redirect to localized path
  const locale = getLocale(request);
  const newUrl = new URL(`/${locale}${pathname}`, request.url);

  return NextResponse.redirect(newUrl);
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
};
```

### Rate Limiting Middleware

```tsx
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

const rateLimit = new Map<string, { count: number; timestamp: number }>();
const WINDOW_MS = 60 * 1000; // 1 minute
const MAX_REQUESTS = 100;

function getRateLimitKey(request: NextRequest): string {
  const ip = request.ip || request.headers.get('x-forwarded-for') || 'unknown';
  return ip;
}

export function middleware(request: NextRequest) {
  // Only rate limit API routes
  if (!request.nextUrl.pathname.startsWith('/api')) {
    return NextResponse.next();
  }

  const key = getRateLimitKey(request);
  const now = Date.now();
  const windowStart = now - WINDOW_MS;

  // Clean up old entries
  const entry = rateLimit.get(key);
  if (entry && entry.timestamp < windowStart) {
    rateLimit.delete(key);
  }

  // Check rate limit
  const current = rateLimit.get(key);
  if (current) {
    if (current.count >= MAX_REQUESTS) {
      return NextResponse.json(
        { error: 'Too many requests' },
        { status: 429 }
      );
    }
    current.count++;
  } else {
    rateLimit.set(key, { count: 1, timestamp: now });
  }

  return NextResponse.next();
}
```

### Matcher Patterns

```tsx
export const config = {
  matcher: [
    // Match all paths except static files
    '/((?!api|_next/static|_next/image|favicon.ico).*)',

    // Match specific paths
    '/dashboard/:path*',
    '/api/:path*',

    // Match with regex
    '/(about|contact)',

    // Exclude specific file extensions
    '/((?!.*\\..*|_next).*)',
  ],
};
```

---

## Redirects and Rewrites

### Static Redirects in next.config.js

```tsx
// next.config.js
module.exports = {
  async redirects() {
    return [
      // Simple redirect
      {
        source: '/old-page',
        destination: '/new-page',
        permanent: true, // 308 status code
      },
      // Temporary redirect
      {
        source: '/temp-redirect',
        destination: '/destination',
        permanent: false, // 307 status code
      },
      // With path parameters
      {
        source: '/old-blog/:slug',
        destination: '/blog/:slug',
        permanent: true,
      },
      // Wildcard
      {
        source: '/docs/:path*',
        destination: 'https://docs.example.com/:path*',
        permanent: true,
      },
      // With query params
      {
        source: '/search',
        has: [{ type: 'query', key: 'q' }],
        destination: '/search/:q',
        permanent: false,
      },
      // Based on header
      {
        source: '/mobile-page',
        has: [
          {
            type: 'header',
            key: 'x-mobile-device',
            value: 'true',
          },
        ],
        destination: '/mobile',
        permanent: false,
      },
    ];
  },
};
```

### Static Rewrites in next.config.js

```tsx
// next.config.js
module.exports = {
  async rewrites() {
    return {
      // Run before filesystem check
      beforeFiles: [
        {
          source: '/api/:path*',
          destination: 'https://api.example.com/:path*',
        },
      ],
      // Run after filesystem, before dynamic routes
      afterFiles: [
        {
          source: '/non-existent',
          destination: '/somewhere-else',
        },
      ],
      // Run after all checks
      fallback: [
        {
          source: '/:path*',
          destination: 'https://fallback.example.com/:path*',
        },
      ],
    };
  },
};
```

### Programmatic Redirects

```tsx
// Server Component
import { redirect, permanentRedirect } from 'next/navigation';

export default async function Page() {
  const user = await getUser();

  if (!user) {
    redirect('/login'); // 307 temporary redirect
  }

  if (user.needsOnboarding) {
    permanentRedirect('/onboarding'); // 308 permanent redirect
  }

  return <Dashboard user={user} />;
}
```

### Redirects in Server Actions

```tsx
'use server';

import { redirect } from 'next/navigation';

export async function createPost(formData: FormData) {
  const post = await db.post.create({
    data: {
      title: formData.get('title') as string,
      content: formData.get('content') as string,
    },
  });

  redirect(`/posts/${post.id}`);
}
```

### Redirects in Route Handlers

```tsx
// app/api/redirect/route.ts
import { redirect } from 'next/navigation';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  // Using redirect function
  redirect('/destination');

  // Or using NextResponse
  return NextResponse.redirect(new URL('/destination', request.url));
}
```

---

## Not Found Pages

### Global Not Found

```tsx
// app/not-found.tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <div className="not-found">
      <h1>404 - Page Not Found</h1>
      <p>The page you're looking for doesn't exist.</p>
      <Link href="/">Go back home</Link>
    </div>
  );
}
```

### Segment-Level Not Found

```tsx
// app/blog/not-found.tsx
import Link from 'next/link';

export default function BlogNotFound() {
  return (
    <div className="not-found">
      <h1>Blog Post Not Found</h1>
      <p>We couldn't find the blog post you're looking for.</p>
      <Link href="/blog">Browse all posts</Link>
    </div>
  );
}
```

### Triggering Not Found

```tsx
// app/blog/[slug]/page.tsx
import { notFound } from 'next/navigation';

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug);

  if (!post) {
    notFound(); // Renders the closest not-found.tsx
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.content}</div>
    </article>
  );
}
```

### Not Found with Metadata

```tsx
// app/not-found.tsx
import Link from 'next/link';
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: '404 - Page Not Found',
  description: 'The page you requested could not be found.',
};

export default function NotFound() {
  return (
    <div className="not-found-container">
      <div className="not-found-content">
        <h1>404</h1>
        <h2>Page Not Found</h2>
        <p>Sorry, we couldn't find the page you're looking for.</p>
        <div className="not-found-actions">
          <Link href="/" className="btn-primary">
            Go Home
          </Link>
          <Link href="/contact" className="btn-secondary">
            Contact Support
          </Link>
        </div>
      </div>
    </div>
  );
}
```

---

## Error Handling Pages

### Global Error Boundary

```tsx
// app/error.tsx
'use client';

import { useEffect } from 'react';

interface ErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

export default function Error({ error, reset }: ErrorProps) {
  useEffect(() => {
    // Log error to reporting service
    console.error('Application error:', error);
  }, [error]);

  return (
    <div className="error-container">
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      {error.digest && (
        <p className="error-digest">Error ID: {error.digest}</p>
      )}
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

### Segment-Level Error Handling

```tsx
// app/dashboard/error.tsx
'use client';

import { useEffect } from 'react';
import Link from 'next/link';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Log to analytics
    logError('dashboard', error);
  }, [error]);

  return (
    <div className="dashboard-error">
      <h2>Dashboard Error</h2>
      <p>Failed to load dashboard data.</p>
      <div className="error-actions">
        <button onClick={reset}>Retry</button>
        <Link href="/">Go to homepage</Link>
      </div>
    </div>
  );
}
```

### Global Error (Including Root Layout)

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
          <h2>Critical Error</h2>
          <p>A critical error occurred. Please try again.</p>
          <button onClick={() => reset()}>Try again</button>
        </div>
      </body>
    </html>
  );
}
```

### Error Recovery Patterns

```tsx
// app/posts/error.tsx
'use client';

import { useRouter } from 'next/navigation';

export default function PostsError({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  const router = useRouter();

  const handleRetry = () => {
    // Clear any cached data
    router.refresh();
    // Then reset the error boundary
    reset();
  };

  return (
    <div className="error-page">
      <h2>Failed to load posts</h2>
      <p>{error.message}</p>
      <div className="error-actions">
        <button onClick={handleRetry}>Retry</button>
        <button onClick={() => router.back()}>Go Back</button>
        <button onClick={() => router.push('/')}>Go Home</button>
      </div>
    </div>
  );
}
```

---

## Loading UI

### Basic Loading State

```tsx
// app/loading.tsx
export default function Loading() {
  return (
    <div className="loading-spinner">
      <div className="spinner"></div>
      <p>Loading...</p>
    </div>
  );
}
```

### Skeleton Loading

```tsx
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div className="dashboard-skeleton">
      <div className="skeleton-header" />
      <div className="skeleton-grid">
        {Array.from({ length: 4 }).map((_, i) => (
          <div key={i} className="skeleton-card">
            <div className="skeleton-image" />
            <div className="skeleton-title" />
            <div className="skeleton-text" />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Streaming with Suspense

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { DashboardHeader } from './components/DashboardHeader';
import { RecentActivity } from './components/RecentActivity';
import { Statistics } from './components/Statistics';
import { LoadingSkeleton } from './components/LoadingSkeleton';

export default function DashboardPage() {
  return (
    <div className="dashboard">
      <DashboardHeader />

      <div className="dashboard-grid">
        <Suspense fallback={<LoadingSkeleton type="stats" />}>
          <Statistics />
        </Suspense>

        <Suspense fallback={<LoadingSkeleton type="activity" />}>
          <RecentActivity />
        </Suspense>
      </div>
    </div>
  );
}
```

### Loading Component with Animation

```tsx
// app/blog/loading.tsx
export default function BlogLoading() {
  return (
    <div className="blog-loading">
      <div className="loading-header">
        <div className="shimmer" style={{ width: '60%', height: '2rem' }} />
        <div className="shimmer" style={{ width: '40%', height: '1rem' }} />
      </div>
      <div className="loading-posts">
        {[1, 2, 3].map((i) => (
          <div key={i} className="loading-post-card">
            <div className="shimmer loading-image" />
            <div className="loading-content">
              <div className="shimmer" style={{ width: '80%', height: '1.5rem' }} />
              <div className="shimmer" style={{ width: '100%', height: '1rem' }} />
              <div className="shimmer" style={{ width: '60%', height: '1rem' }} />
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Instant Loading States

```tsx
// app/products/page.tsx
import { Suspense } from 'react';
import { ProductGrid } from './ProductGrid';
import { ProductFilters } from './ProductFilters';

// This component loads instantly
function InstantHeader() {
  return (
    <header className="products-header">
      <h1>Products</h1>
      <p>Browse our collection</p>
    </header>
  );
}

export default function ProductsPage() {
  return (
    <div className="products-page">
      {/* Shows immediately */}
      <InstantHeader />

      {/* Streams in when ready */}
      <Suspense fallback={<FiltersSkeleton />}>
        <ProductFilters />
      </Suspense>

      <Suspense fallback={<GridSkeleton />}>
        <ProductGrid />
      </Suspense>
    </div>
  );
}
```

---

## Best Practices

### 1. Route Organization

```
app/
├── (marketing)/              # Public pages
│   ├── page.tsx
│   ├── about/
│   └── blog/
├── (app)/                    # Authenticated app
│   ├── dashboard/
│   ├── settings/
│   └── profile/
├── (auth)/                   # Auth flows
│   ├── login/
│   └── register/
└── api/                      # API routes
    ├── auth/
    └── webhooks/
```

### 2. Component Organization

Keep route-specific components colocated:

```
app/dashboard/
├── page.tsx
├── layout.tsx
├── loading.tsx
├── error.tsx
├── components/              # Route-specific components
│   ├── DashboardChart.tsx
│   └── DashboardStats.tsx
└── lib/                     # Route-specific utilities
    └── dashboard-utils.ts
```

### 3. Type Safety for Params

```tsx
// Define types for your routes
type BlogParams = {
  slug: string;
};

type ProductParams = {
  category: string;
  id: string;
};

// Use in page components
interface PageProps {
  params: BlogParams;
  searchParams: { [key: string]: string | string[] | undefined };
}
```

### 4. Prefetching Strategy

```tsx
// Disable prefetch for rarely visited pages
<Link href="/terms" prefetch={false}>Terms</Link>

// Manual prefetch on hover
const router = useRouter();

<button
  onMouseEnter={() => router.prefetch('/dashboard')}
  onClick={() => router.push('/dashboard')}
>
  Go to Dashboard
</button>
```

### 5. Search Params Patterns

```tsx
// Create a URL builder utility
function createUrl(pathname: string, params: Record<string, string | number | null>) {
  const searchParams = new URLSearchParams();

  Object.entries(params).forEach(([key, value]) => {
    if (value !== null && value !== undefined && value !== '') {
      searchParams.set(key, String(value));
    }
  });

  const query = searchParams.toString();
  return query ? `${pathname}?${query}` : pathname;
}

// Usage
const url = createUrl('/search', {
  query: 'nextjs',
  page: 1,
  sort: 'date',
});
```

### 6. Error Boundary Placement

Place error boundaries at appropriate levels:

```
app/
├── error.tsx               # Catches all app errors
├── dashboard/
│   ├── error.tsx          # Catches dashboard errors
│   └── analytics/
│       └── error.tsx      # Catches analytics-specific errors
```

### 7. Loading State Strategy

Use granular loading states for better UX:

```tsx
// Instead of one large loading state
// app/dashboard/loading.tsx (shows for entire page)

// Use Suspense for granular loading
// app/dashboard/page.tsx
<div>
  <Header />  {/* Shows immediately */}
  <Suspense fallback={<StatsSkeleton />}>
    <Stats />  {/* Streams when ready */}
  </Suspense>
  <Suspense fallback={<ChartSkeleton />}>
    <Chart />  {/* Streams when ready */}
  </Suspense>
</div>
```

### 8. API Route Best Practices

```tsx
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';

// Validation schema
const createPostSchema = z.object({
  title: z.string().min(1).max(100),
  content: z.string().min(1),
});

export async function POST(request: NextRequest) {
  try {
    // Parse and validate
    const body = await request.json();
    const validated = createPostSchema.parse(body);

    // Create resource
    const post = await db.post.create({ data: validated });

    // Return with proper status
    return NextResponse.json(post, { status: 201 });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Validation failed', details: error.errors },
        { status: 400 }
      );
    }

    console.error('Failed to create post:', error);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

### 9. Middleware Best Practices

```tsx
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Early return for static files
  if (
    request.nextUrl.pathname.startsWith('/_next') ||
    request.nextUrl.pathname.includes('.')
  ) {
    return NextResponse.next();
  }

  // Add request ID for tracing
  const requestId = crypto.randomUUID();
  const response = NextResponse.next();
  response.headers.set('x-request-id', requestId);

  return response;
}

// Be specific with matcher
export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico|public).*)',
  ],
};
```

### 10. Dynamic Route Generation

```tsx
// app/blog/[slug]/page.tsx

// Generate static params at build time
export async function generateStaticParams() {
  const posts = await getAllPosts();

  return posts.map((post) => ({
    slug: post.slug,
  }));
}

// Enable ISR for new posts
export const dynamicParams = true; // default
export const revalidate = 3600; // Revalidate every hour

// Or disable for 404 on unknown slugs
// export const dynamicParams = false;
```

### 11. Handling External Redirects

```tsx
// For external URLs, use redirect in middleware or API routes
// middleware.ts
export function middleware(request: NextRequest) {
  if (request.nextUrl.pathname === '/github') {
    return NextResponse.redirect('https://github.com/myorg');
  }
}

// Or in next.config.js
module.exports = {
  async redirects() {
    return [
      {
        source: '/github',
        destination: 'https://github.com/myorg',
        permanent: false,
      },
    ];
  },
};
```

### 12. Route Handler Caching

```tsx
// app/api/posts/route.ts

// Force dynamic (no caching)
export const dynamic = 'force-dynamic';

// Or with revalidation
export const revalidate = 60;

// With tags for on-demand revalidation
import { revalidateTag } from 'next/cache';

export async function GET() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { tags: ['posts'] },
  });

  return NextResponse.json(await posts.json());
}

// Revalidate from another route
export async function POST() {
  revalidateTag('posts');
  return NextResponse.json({ revalidated: true });
}
```

---

## Summary

Next.js App Router provides a powerful, file-system based routing solution with:

- **File-System Routing**: Intuitive folder-based route definitions
- **Nested Layouts**: Shared UI that persists across navigations
- **Dynamic Routes**: Flexible patterns for dynamic content
- **Parallel & Intercepting Routes**: Advanced UI patterns like modals
- **API Routes**: Full-featured backend endpoints
- **Middleware**: Request/response transformation
- **Error & Loading States**: Built-in UI for edge cases

For the most up-to-date information, always refer to the [official Next.js documentation](https://nextjs.org/docs).
