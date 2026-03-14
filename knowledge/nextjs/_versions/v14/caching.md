# Next.js 14 → Caching Delta

## Caching Defaults Changed in Next.js 15

### fetch() Behavior

```tsx
// Next.js 14 - Cached by default
const data = await fetch('https://api.example.com/data')
// This is cached! Equivalent to { cache: 'force-cache' }

// Next.js 15 - NOT cached by default
const data = await fetch('https://api.example.com/data')
// This is NOT cached! Equivalent to { cache: 'no-store' }
```

### Explicit Caching in Next.js 15

```tsx
// Cache the request
const data = await fetch(url, { cache: 'force-cache' })

// Time-based revalidation
const data = await fetch(url, { next: { revalidate: 3600 } })

// Tag-based revalidation
const data = await fetch(url, { next: { tags: ['products'] } })
```

### GET Route Handlers

```tsx
// Next.js 14 - GET handlers cached by default
export async function GET() {
  const data = await db.query()
  return Response.json(data)  // Cached!
}

// Next.js 15 - GET handlers NOT cached by default
export async function GET() {
  const data = await db.query()
  return Response.json(data)  // NOT cached
}

// Opt-in to caching in Next.js 15
export const dynamic = 'force-static'
export async function GET() {
  const data = await db.query()
  return Response.json(data)  // Now cached
}
```

### Client-side Router Cache

```tsx
// Next.js 14 - Pages cached for 30 seconds
// Navigating to a previously visited page uses cache

// Next.js 15 - Default staleTime is 0
// Pages are not cached by default

// Opt-in to caching in Next.js 15
// next.config.js
module.exports = {
  experimental: {
    staleTimes: {
      dynamic: 30,  // Cache dynamic pages for 30s
      static: 180   // Cache static pages for 180s
    }
  }
}
```

## Why the Change?

Next.js 15 prioritizes **freshness over performance** by default:
- Prevents stale data issues
- More predictable behavior
- Explicit caching = fewer surprises

## Migration from Next.js 14 to 15

1. **Audit fetch calls** - Add `cache: 'force-cache'` where needed
2. **Check Route Handlers** - Add `dynamic = 'force-static'` for cached routes
3. **Configure staleTimes** - If relying on client router cache
4. **Test thoroughly** - Caching changes can affect performance

## Still Current in Next.js 14

- revalidatePath() and revalidateTag()
- generateStaticParams
- Time-based revalidation
- Tag-based revalidation
- unstable_cache (for non-fetch)

