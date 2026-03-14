# Next.js 13 → Data Fetching Delta

## Differences from Next.js 14/15

### Caching Defaults

```tsx
// Next.js 13/14 - Cached by default
const data = await fetch('https://api.example.com/data')
// Equivalent to: fetch(url, { cache: 'force-cache' })

// Next.js 15 - NOT cached by default
const data = await fetch('https://api.example.com/data')
// Equivalent to: fetch(url, { cache: 'no-store' })

// To cache in Next.js 15:
const data = await fetch(url, { cache: 'force-cache' })
```

### Async Request APIs (Next.js 15+)

```tsx
// Next.js 13/14 - Synchronous
import { headers, cookies } from 'next/headers'

export default function Page() {
  const headerList = headers()
  const cookieStore = cookies()
  const theme = cookieStore.get('theme')

  return <div>{theme?.value}</div>
}

// Next.js 15 - Async (breaking change)
export default async function Page() {
  const headerList = await headers()
  const cookieStore = await cookies()
  const theme = cookieStore.get('theme')

  return <div>{theme?.value}</div>
}
```

### Dynamic Params

```tsx
// Next.js 13/14 - Synchronous params
export default function Page({ params }: { params: { id: string } }) {
  const { id } = params
  return <div>ID: {id}</div>
}

// Next.js 15 - Async params
export default async function Page({
  params
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  return <div>ID: {id}</div>
}
```

### SearchParams

```tsx
// Next.js 13/14 - Synchronous searchParams
export default function Page({
  searchParams
}: {
  searchParams: { q?: string }
}) {
  return <div>Search: {searchParams.q}</div>
}

// Next.js 15 - Async searchParams
export default async function Page({
  searchParams
}: {
  searchParams: Promise<{ q?: string }>
}) {
  const { q } = await searchParams
  return <div>Search: {q}</div>
}
```

## Still Current in Next.js 13

- Server Components (default)
- Client Components with 'use client'
- fetch() in Server Components
- generateStaticParams
- revalidatePath and revalidateTag
- Route segment config (dynamic, revalidate, etc.)

## Recommendations for Next.js 13 Users

1. **Be aware of caching** - fetch() is cached by default
2. **Use revalidate options** for ISR behavior
3. **Plan for async APIs** when upgrading to 15

