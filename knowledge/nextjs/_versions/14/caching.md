# Next.js Caching (v14)

## Not available in 14
- What's New in Next.js 16

## Differences

Next.js 14 caches aggressively by default — the opposite of the 15/16 behavior described in the base document:

- `fetch` is **cached indefinitely by default** (`force-cache` is implicit). Opt out per request with `cache: 'no-store'` or `next: { revalidate: 0 }`.
- `revalidateTag(tag)` and `revalidatePath(path)` take a **single argument**; `updateTag()`, `refresh()`, and the `use cache` directive do not exist.
- Async request APIs are **not** required: `cookies()`, `headers()`, `params`, and `searchParams` are synchronous in 14 (they became async in 15).
- Partial Prerendering is experimental (`experimental.ppr`), and Turbopack is dev-only and not the default bundler.
