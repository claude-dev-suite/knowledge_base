# Spring Boot Basics (v3)

## Differences

Spring Boot 3.x differs from the 4.x base document:

- Built on **Spring Framework 6** (4.x uses Spring Framework 7).
- **Jakarta EE 10** baseline (Servlet 6.0, Persistence 3.1, Validation 3.0); 4.x moves to the Jakarta EE 11-era modules (Servlet 6.1, Persistence 3.2, Validation 3.1).
- **Jackson 2** is the default JSON mapper (4.x defaults to Jackson 3, with Jackson 2 support shipped deprecated).
- Environment post-processors implement `org.springframework.boot.env.EnvironmentPostProcessor` (moved to `org.springframework.boot.EnvironmentPostProcessor` in 4.x).
- Tracing uses `@ConditionalOnEnabledTracing` and the `management.tracing.enabled` property (renamed to `@ConditionalOnEnabledTracingExport` and `management.tracing.export.enabled` in 4.x).
- `spring.dao.exceptiontranslation.enabled` is used (renamed to `spring.persistence.exceptiontranslation.enabled` in 4.x).
- Java 17 is the baseline (unchanged in 4.x, which still requires Java 17+).
