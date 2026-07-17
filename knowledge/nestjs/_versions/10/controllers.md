# Controllers (NestJS 10)

## Differences

NestJS 10 runs on **Express 4** by default (or Fastify 4). Routing uses the Express 4 / path-to-regexp v0 syntax, which differs from the Express 5 defaults documented for NestJS 11:

- **Bare wildcards are valid.** `@Get('*')` matches any path, and inline patterns such as `@Get('ab*cd')` work directly:

  ```typescript
  @Get('*')
  findAll() {
    return 'Wildcard route';
  }
  ```

- **Inline regex in route paths is supported.** `@Get('users/:id(\\d+)')` matches numeric IDs only (removed in Express 5 / NestJS 11).

Other NestJS 10 differences:

- **Node.js 16+** is supported (NestJS 11 requires Node.js 20+).
- `CacheModule` is backed by **cache-manager v5** (NestJS 11 moves to cache-manager v6 / Keyv).
- Termination lifecycle hooks (`OnModuleDestroy`, `OnApplicationShutdown`, ...) run in **registration order** (reversed in NestJS 11).
- The default `ConsoleLogger` has no built-in JSON logging (added in NestJS 11).
