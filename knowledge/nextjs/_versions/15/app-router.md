# Next.js App Router (v15)

## Differences

In Next.js 15, the App Router differs from the Next.js 16 base document:

- Turbopack is opt-in (`next dev --turbopack`); **webpack is the default bundler** (16 makes Turbopack the default for dev and build).
- Caching is configured with `fetch` options, `unstable_cache`, and route segment config — there is no `use cache` / `cacheComponents` model.
- Request interception uses `middleware.ts` (16 introduces `proxy.ts` on the Node.js runtime).
- Parallel routes do **not** require an explicit `default.js` for every slot (16 requires it and fails the build otherwise).
- Uses React 19.0; Next.js 16 uses React 19.2 (View Transitions, `useEffectEvent`, `Activity`).
