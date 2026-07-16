# Next.js Caching (v15)

## Not available in 15
- What's New in Next.js 16

## Differences

In Next.js 15, `fetch` is already opt-in for caching (not cached by default), but the Next.js 16 **Cache Components** model does not exist:

- No `use cache` directive and no `cacheComponents` config option. Use `unstable_cache`, `fetch` cache options, and route segment config instead.
- `revalidateTag(tag)` takes a **single argument** — there is no `cacheLife` profile parameter, and `updateTag()` / `refresh()` do not exist. Use `revalidatePath()`, `revalidateTag()`, and client-side `router.refresh()`.
- Partial Prerendering is opt-in via `export const experimental_ppr = true` plus `experimental.ppr` in `next.config`.
- Request interception uses `middleware.ts` (renamed to `proxy.ts` in 16), and the default bundler is webpack unless Turbopack is explicitly enabled.
