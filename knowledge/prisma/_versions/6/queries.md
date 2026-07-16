# Prisma Queries (Prisma 6)

## Differences

Prisma 6 still supports **client middleware** via `prisma.$use(...)`, which Prisma 7 removes in favour of Client Extensions (`$extends`):

```ts
prisma.$use(async (params, next) => {
  const before = Date.now();
  const result = await next(params);
  console.log(`${params.model}.${params.action} took ${Date.now() - before}ms`);
  return result;
});
```

Other Prisma 6 differences:

- The client is imported from **`@prisma/client`** (Prisma 7 imports from your generated `output` path).
- `prisma generate` still runs **automatically** after `prisma migrate`, and the `--skip-generate` / `--skip-seed` CLI flags are available (both removed in Prisma 7).
- Seeding runs automatically as part of the migrate workflow (Prisma 7 requires an explicit `prisma db seed`).
- Environment variables are loaded automatically without `dotenv`.
