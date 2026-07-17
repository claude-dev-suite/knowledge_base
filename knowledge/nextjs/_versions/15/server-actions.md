# Next.js Server Actions (v15)

## Differences

In Next.js 15, Server Actions differ from the Next.js 16 base document:

- Cache invalidation uses `revalidatePath()` and `revalidateTag(tag)` with a **single argument**. The read-your-writes `updateTag()` and the uncached-data `refresh()` APIs are new in 16 and not available in 15.
- There is no `use cache` directive / Cache Components integration.
- The enhanced Server Action security and dead-code elimination introduced in 15 apply, but there is no `proxy.ts` — request interception uses `middleware.ts`.
