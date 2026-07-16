# Next.js Data Fetching (v15)

## Differences

In Next.js 15, data fetching differs from the Next.js 16 base document as follows:

- `fetch` is not cached by default (same as 16), but there is no `use cache` directive or Cache Components model — cache explicitly with `fetch` options, `unstable_cache`, or route segment config.
- `revalidateTag()` takes a **single argument** (no `cacheLife` profile); `updateTag()` and `refresh()` are not available.
- Async request APIs (`await cookies()`, `await headers()`, `await params`, `await searchParams`) are required as of 15.
- Requests are intercepted with `middleware.ts` (renamed to `proxy.ts` in 16).
