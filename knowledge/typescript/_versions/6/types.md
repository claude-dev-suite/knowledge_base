# TypeScript Types (v6)

## Differences

The type **syntax** in this document is unchanged between TypeScript 6 and 7 — the differences are in the compiler and its configuration:

- TypeScript 6.x runs the **JavaScript-based `tsc`** (7.0 ships the native Go compiler, `tsgo`).
- `strict` is **not** forced on. If your `tsconfig.json` omits `"strict": true`, the strict-family checks (`noImplicitAny`, `strictNullChecks`, ...) stay off — 7.0 enables them by default.
- Legacy compiler options still work: `target: "es5"`, `module: "amd" | "umd" | "system"`, `moduleResolution: "node10"`, and `baseUrl` are all available (removed in 7.0).
- The programmatic compiler API is stable, so tools that embed `tsc` (ts-loader, and Vue/Svelte/Angular template type-checking) run on 6.x — 7.0's API does not land until 7.1.
- The stricter JSDoc-in-JS semantics introduced in 7.0 appear only as **deprecation warnings** in 6.x, not hard errors.
