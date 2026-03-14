# Next.js App Router

## Directory Structure

```
app/
├── layout.tsx          # Root layout (required)
├── page.tsx            # Home page (/)
├── loading.tsx         # Loading UI
├── error.tsx           # Error UI
├── not-found.tsx       # 404 page
├── global-error.tsx    # Global error boundary
├── dashboard/
│   ├── layout.tsx      # Nested layout
│   ├── page.tsx        # /dashboard
│   └── settings/
│       └── page.tsx    # /dashboard/settings
├── blog/
│   ├── page.tsx        # /blog
│   └── [slug]/
│       └── page.tsx    # /blog/:slug (dynamic)
└── api/
    └── users/
        └── route.ts    # API route
```

## File Conventions

| File | Purpose |
|------|---------|
| `page.tsx` | Page component (makes route accessible) |
| `layout.tsx` | Shared UI that wraps pages |
| `loading.tsx` | Loading UI (Suspense boundary) |
| `error.tsx` | Error boundary |
| `not-found.tsx` | 404 UI |
| `route.ts` | API endpoint |
| `template.tsx` | Re-rendered layout on navigation |

## Layouts

```tsx
// app/layout.tsx - Root layout (required)
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <nav>...</nav>
        <main>{children}</main>
        <footer>...</footer>
      </body>
    </html>
  );
}

// app/dashboard/layout.tsx - Nested layout
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="dashboard">
      <Sidebar />
      <div className="content">{children}</div>
    </div>
  );
}
```

## Pages

```tsx
// app/page.tsx
export default function Home() {
  return <h1>Welcome</h1>;
}

// app/blog/[slug]/page.tsx - Dynamic page
export default function BlogPost({
  params,
}: {
  params: { slug: string };
}) {
  return <article>Post: {params.slug}</article>;
}

// Generate static params for dynamic routes
export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map((post) => ({ slug: post.slug }));
}
```

## Loading & Error States

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <div className="skeleton">Loading...</div>;
}

// app/dashboard/error.tsx
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

## Metadata

```tsx
// Static metadata
export const metadata = {
  title: 'My App',
  description: 'App description',
};

// Dynamic metadata
export async function generateMetadata({
  params,
}: {
  params: { slug: string };
}) {
  const post = await getPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      images: [post.image],
    },
  };
}
```

## Route Groups

```
app/
├── (marketing)/       # Route group (not in URL)
│   ├── layout.tsx     # Shared marketing layout
│   ├── about/page.tsx # /about
│   └── blog/page.tsx  # /blog
├── (shop)/
│   ├── layout.tsx     # Shared shop layout
│   ├── products/page.tsx  # /products
│   └── cart/page.tsx      # /cart
└── layout.tsx         # Root layout
```

## Parallel Routes

```
app/
├── @modal/           # Parallel route slot
│   └── login/page.tsx
├── @sidebar/
│   └── default.tsx
├── layout.tsx
└── page.tsx

// layout.tsx
export default function Layout({
  children,
  modal,
  sidebar,
}: {
  children: React.ReactNode;
  modal: React.ReactNode;
  sidebar: React.ReactNode;
}) {
  return (
    <>
      {sidebar}
      {children}
      {modal}
    </>
  );
}
```

## Intercepting Routes

```
app/
├── feed/
│   └── page.tsx
├── photo/[id]/
│   └── page.tsx
└── @modal/
    └── (.)photo/[id]/     # Intercepts /photo/[id]
        └── page.tsx       # Shows in modal
```
