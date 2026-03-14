# Contributing to knowledge_base

<!-- SPDX-License-Identifier: MIT -->

Thank you for your interest in contributing to the knowledge base. This repository
provides curated documentation consumed by the
[dev-suite](https://github.com/claude-dev-suite/dev-suite) MCP server.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [How to Contribute](#how-to-contribute)
- [Development Setup](#development-setup)
- [Content Guidelines](#content-guidelines)
- [File Naming Conventions](#file-naming-conventions)
- [Pull Request Process](#pull-request-process)
- [Reporting Issues](#reporting-issues)

## Code of Conduct

This project follows the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).
By participating, you agree to uphold these standards.

## How to Contribute

### Adding New Technology Documentation

1. Fork the repository on GitHub.
2. Create a new branch: `git checkout -b docs/add-<technology>`.
3. Create a directory under `knowledge/<technology>/`.
4. Add a `manifest.json` describing the technology versions and topics
   (see `knowledge/react/manifest.json` as a reference).
5. Add one Markdown file per topic.
6. Submit a pull request.

### Updating Existing Documentation

1. Fork and create a branch: `git checkout -b docs/update-<technology>`.
2. Locate the file under `knowledge/<technology>/`.
3. Apply your changes following the [Content Guidelines](#content-guidelines).
4. Update the `manifest.json` if version information changed.
5. Submit a pull request.

### Fixing Errors or Typos

Open an issue using the **Bug Report** template or submit a small pull request
directly with the fix.

## Development Setup

This repository contains only Markdown and JSON files — no build step is required.

```bash
git clone https://github.com/claude-dev-suite/knowledge_base.git
cd knowledge_base
```

To preview Markdown locally, any standard Markdown viewer or IDE extension works.

## Content Guidelines

- Use clear, concise language suitable for developers.
- Start each file with a `# Topic Title` heading.
- Include a blockquote linking to the official documentation:
  ```markdown
  > **Official Documentation:** [Title](https://example.com/docs)
  ```
- Provide practical, copy-pasteable code examples in fenced code blocks with the
  correct language identifier.
- Keep each file focused on a single topic or closely related set of concepts.
- Do not copy-paste entire official documentation pages; summarise and provide
  working examples instead.
- Run a spell-check before submitting.

## File Naming Conventions

| What | Convention | Example |
|------|-----------|---------|
| Technology directory | lowercase hyphenated | `spring-boot/` |
| Topic file | lowercase hyphenated | `dependency-injection.md` |
| Manifest | always named | `manifest.json` |
| Version delta file | `<topic>_<version>.md` | `hooks_18.md` |

## Pull Request Process

1. Ensure your branch is up to date with `main` before opening a PR.
2. Fill in the pull request template completely.
3. A maintainer will review within **5 business days**.
4. Address any requested changes promptly.
5. Once approved, a maintainer will merge the PR.

For large additions (new technology with many files), open an issue first to
discuss scope and avoid duplicated effort.

## Reporting Issues

Use GitHub Issues with the appropriate template:

- **Bug Report** — incorrect or outdated documentation
- **Feature Request** — request for new technology coverage or structural changes

For security-related concerns, follow the [Security Policy](SECURITY.md).

## License

By contributing, you agree that your contributions will be licensed under the
[MIT License](LICENSE) that covers this project.
