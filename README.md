# Knowledge Base

Documentation knowledge base for [dev-suite](https://github.com/claude-dev-suite/dev-suite) MCP server.

## Structure

```
knowledge/
├── react/          # React documentation
├── spring-boot/    # Spring Boot documentation
├── nextjs/         # Next.js documentation
├── postgresql/     # PostgreSQL documentation
└── [66+ technologies]
```

## Technologies Covered (66+)

### Frontend Frameworks
- React, Vue, Angular, Svelte, Solid
- Next.js, Nuxt, Remix, SvelteKit, Astro
- Electron

### Backend Frameworks
- Node.js: NestJS, Express, Fastify, Hono
- Python: FastAPI, Django, Flask
- Java: Spring Boot, Spring Data JPA, Spring Security

### Databases & ORM
- PostgreSQL, MySQL, MongoDB, Redis
- Prisma, Drizzle, TypeORM, SQLAlchemy

### Testing
- Vitest, Jest, Playwright, Cypress, pytest, JUnit
- Testing Library

### State Management
- Zustand, Redux Toolkit, Pinia, TanStack Query

### Authentication
- JWT, OAuth2, NextAuth

### Infrastructure
- Docker, Docker Compose, Kubernetes
- GitHub Actions

### API Design
- REST API, GraphQL, tRPC, OpenAPI

### Tooling
- TailwindCSS, Biome
- Flyway, Lombok, MapStruct, Springdoc OpenAPI
- shadcn/ui, react-hook-form

### Messaging
- Kafka, RabbitMQ, ActiveMQ, SQS, Redis Pub/Sub
- NATS, Pulsar, Azure Service Bus, Google Pub/Sub

## Usage

This repository is consumed by the dev-suite documentation MCP server using **Git sparse checkout**:

1. User requests documentation (e.g., `react/hooks`)
2. MCP server clones only `knowledge/react/` (sparse checkout)
3. Files are cached locally for 2 hours
4. Automatic refresh after cache expiration

See [dev-suite documentation](https://github.com/claude-dev-suite/dev-suite/blob/develop/mcp-servers/documentation/README.md) for details.

## File Format

All documentation files are Markdown (`.md`) with the following structure:

```markdown
# Topic Title

> **Official Documentation:** [Link](https://example.com)

## Section 1

Content...

## Section 2

Content...

## Examples

\`\`\`language
code example
\`\`\`
```

## Contributing

To add or update documentation:

1. Fork this repository
2. Add/update files in `knowledge/{technology}/`
3. Follow existing file structure and naming conventions
4. Submit a pull request

### Guidelines

- Use clear, concise language
- Include official documentation links
- Provide practical code examples
- Keep files focused on specific topics
- Use consistent Markdown formatting

## Maintenance

This knowledge base is maintained separately from dev-suite to allow:
- Independent updates without MCP server releases
- Community contributions
- Version tracking for documentation
- Rapid bug fixes and improvements

## License

MIT

## Statistics

- **Technologies**: 66+
- **Documentation Files**: 136+
- **Total Size**: ~2MB
