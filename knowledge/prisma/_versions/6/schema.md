# Prisma Schema (Prisma 6)

## Differences

Prisma 6 uses the classic generator and CommonJS-friendly defaults, which differ from the Prisma 7 base documentation:

- The generator provider is **`prisma-client-js`** and `output` is **optional** — the client is generated into `node_modules/@prisma/client` by default:

  ```prisma
  generator client {
    provider = "prisma-client-js"
  }
  ```

- **No `prisma.config.ts`** is required. The datasource `url` is read from `.env` automatically (environment variables are auto-loaded).
- **ESM is not required** — CommonJS projects work without `"type": "module"` in `package.json`.
- **Driver adapters are optional** (an opt-in preview), not required per datasource.
- The client is imported from `@prisma/client`, not from a generated `output` path.
