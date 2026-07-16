# Spring Boot Security (v3)

## Differences

Spring Boot 3.x pairs with **Spring Security 6** (on Spring Framework 6), whereas Spring Boot 4.x moves to the Spring Security 7 / Spring Framework 7 generation:

- Security is configured with the Spring Security 6 lambda DSL — a `SecurityFilterChain` bean using `http.authorizeHttpRequests(...)`. The legacy `WebSecurityConfigurerAdapter` is already removed, so component-based configuration is required.
- Servlet and security APIs are `jakarta.*` under **Jakarta EE 10** (Servlet 6.0). Spring Boot 4 upgrades to the Jakarta EE 11-era servlet stack (Servlet 6.1).
- Method security, CSRF, and CORS configuration follow the Spring Security 6 APIs; review the Spring Security 7 migration notes before moving to Spring Boot 4.
