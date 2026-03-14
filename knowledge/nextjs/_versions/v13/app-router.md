# Next.js 13 → App Router Delta

## Status in Next.js 13

- App Router introduced as **beta** in 13.0
- Became stable in 13.4
- Server Actions introduced as **alpha**

## Differences from Next.js 14/15

### Server Actions

```tsx
// Next.js 14/15 - Server Actions stable
async function submitForm(formData: FormData) {
  'use server'
  // ...
}

// Next.js 13 - Server Actions experimental
// next.config.js required:
module.exports = {
  experimental: {
    serverActions: true
  }
}
```

### Metadata API

```tsx
// Next.js 14/15 - generateMetadata async stable
export async function generateMetadata({ params }): Promise<Metadata> {
  const product = await getProduct(params.id)
  return { title: product.name }
}

// Next.js 13 - Same API, but some features in beta
```

### Caching Behavior

```tsx
// Next.js 13/14 - fetch() cached by default
const data = await fetch(url)  // Cached

// Next.js 15 - fetch() NOT cached by default
const data = await fetch(url)  // NOT cached
const cached = await fetch(url, { cache: 'force-cache' })  // Explicitly cached
```

### Turbopack

```bash
# Next.js 13 - Turbopack alpha
next dev --turbo  # Alpha, many features missing

# Next.js 14 - Turbopack beta
next dev --turbo  # Most features work

# Next.js 15 - Turbopack stable
next dev --turbo  # Production ready
```

## Still Current in Next.js 13

- File-based routing in `/app`
- Layouts and nested layouts
- Loading and error boundaries
- `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`
- Route groups `(folder)`
- Parallel routes `@folder`
- Intercepting routes `(..)`
- Static and dynamic rendering

## Recommendations for Next.js 13 Users

1. **Upgrade to 14+** for stable Server Actions
2. **Enable experimental flags** for Server Actions in 13
3. **Use Pages Router** for features not yet stable in App Router
4. **Consider 15** for React 19 features

## Migration Notes

Next.js 13 → 14:
- Server Actions become stable (remove experimental flag)
- Turbopack more reliable

Next.js 14 → 15:
- Update fetch() calls with explicit cache options
- headers(), cookies(), params now async
- Consider next/form for enhanced forms

