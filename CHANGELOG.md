# Changelog

<!-- SPDX-License-Identifier: MIT -->

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Added
- Open-source community health files (LICENSE, CONTRIBUTING, CODE_OF_CONDUCT, SECURITY)
- GitHub issue and pull request templates
- Version delta files under `knowledge/{tech}/_versions/{version}/{topic}.md` for the
  previously-current majors of nestjs (10), prisma (6), nextjs (14, 15), spring-boot (3),
  and typescript (6), so the documentation server can reconstruct non-latest versions

### Fixed
- `bulk-engineering/namur-ne148.md` documented the wrong NAMUR recommendation and is now
  `namur-ne150.md`. Its subject — the standardised interface for exchanging engineering data
  between CAE systems and PCS engineering tools — is NE 150 (version 2014-10-13). NE 148 is
  "Automation Requirements relating to Modularisation of Process Plants", an unrelated
  recommendation about modular plants. Both titles verified against namur.net. The file's two
  cited sources were dead (404): the NAMUR per-recommendation detail page it linked does not
  exist (NAMUR publishes no free per-NE pages; full texts are paid via DIN Media), and the
  automation.com article is gone. Replaced with the NE 150 announcement page and the
  recommendations index. Nothing referenced the old path, so no `local` path breaks.

### Changed
- Updated stale version manifests to the current stable majors (verified against upstream
  release channels, 2026-07): nestjs 10→11, prisma 5→7, nextjs 15→16, spring-boot 3→4,
  typescript 5→7. Refreshed `latest`/`supported`/`eol`, added `breaking_changes` for each
  new transition, and updated the affected base topic docs to describe the new latest.
- Audited and corrected code examples in the base docs of the five re-versioned techs
  against official docs: Next.js async `params`/`searchParams`/`cookies`/`headers`, Cache
  Components, `proxy.ts`, `useActionState`; Prisma 7 `prisma-client` generator, generated-path
  imports, mandatory driver adapters, `$extends` (no `$use`); Spring Boot 4 `@MockitoBean`,
  Jackson 3, Spring Cloud 2025.1; plus smaller NestJS 11 and TypeScript 7 fixes.
- Refreshed stale third-party API usage in the same base docs: RxJS `retryWhen` → `retry({ delay })`
  and removed a dead `class-transformer` import (NestJS interceptors); jjwt `0.11.5` → `0.13.0` with
  the 0.12+ `Jwts.parser().verifyWith(...).parseSignedClaims(...)` / builder API (Spring Boot security).
- Documented that Next.js 16 requires an explicit `default.js` in every parallel-route slot.

---

## [0.5.0] - 2026-03-14

### Added
- Angular, .NET, and C# documentation
- Comprehensive Python documentation
- Spring ecosystem documentation
- Node.js, ESLint, WCAG, JSDoc, TSDoc, SPDX documentation

## [0.1.0] - 2026-01-01

### Added
- Initial knowledge base structure with 66+ technologies documented

[Unreleased]: https://github.com/claude-dev-suite/knowledge_base/compare/HEAD...HEAD
[0.5.0]: https://github.com/claude-dev-suite/knowledge_base/releases/tag/v0.5.0
[0.1.0]: https://github.com/claude-dev-suite/knowledge_base/releases/tag/v0.1.0
