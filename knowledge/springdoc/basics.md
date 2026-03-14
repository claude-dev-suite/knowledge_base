# Springdoc OpenAPI

> Official Documentation: https://springdoc.org/

## Overview

Springdoc OpenAPI is a library that automatically generates OpenAPI 3.0 documentation for Spring Boot applications. It integrates seamlessly with Spring MVC, Spring WebFlux, and provides an embedded Swagger UI for interactive API exploration.

Key features:
- Automatic API documentation from Spring annotations
- Support for OpenAPI 3.0/3.1 specification
- Integrated Swagger UI
- Support for Spring Security
- Customizable through annotations and configuration

---

## Maven/Gradle Setup

### Maven

```xml
<!-- For Spring MVC (Spring Boot 3.x) -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>

<!-- For Spring WebFlux (reactive) -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
    <version>2.3.0</version>
</dependency>

<!-- For Spring Boot 2.x (legacy) -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.7.0</version>
</dependency>
```

### Gradle

```groovy
// For Spring MVC (Spring Boot 3.x)
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0'

// For Spring WebFlux (reactive)
implementation 'org.springdoc:springdoc-openapi-starter-webflux-ui:2.3.0'

// For Spring Boot 2.x (legacy)
implementation 'org.springdoc:springdoc-openapi-ui:1.7.0'
```

---

## Basic Configuration (application.yml)

```yaml
springdoc:
  # API docs endpoint configuration
  api-docs:
    enabled: true
    path: /v3/api-docs
    resolve-schema-properties: true

  # Swagger UI configuration
  swagger-ui:
    enabled: true
    path: /swagger-ui.html
    operationsSorter: method          # Sort by HTTP method
    tagsSorter: alpha                  # Sort tags alphabetically
    displayRequestDuration: true       # Show request duration
    filter: true                       # Enable search filter
    tryItOutEnabled: true             # Enable "Try it out" by default
    persistAuthorization: true        # Persist auth between page refreshes

  # Package scanning
  packages-to-scan: com.example.api.controller
  paths-to-match: /api/**

  # Exclude paths from documentation
  paths-to-exclude: /actuator/**, /error

  # Default produces/consumes media types
  default-produces-media-type: application/json
  default-consumes-media-type: application/json

  # Show actuator endpoints
  show-actuator: false

  # Enable/disable by profile
  # api-docs.enabled: ${ENABLE_SWAGGER:true}
```

### Access Points

After configuration, documentation is available at:
- **Swagger UI**: `http://localhost:8080/swagger-ui.html`
- **OpenAPI JSON**: `http://localhost:8080/v3/api-docs`
- **OpenAPI YAML**: `http://localhost:8080/v3/api-docs.yaml`

---

## @Operation and @ApiResponse Annotations

The `@Operation` annotation documents individual API operations, while `@ApiResponse` describes possible responses.

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.media.ExampleObject;

@RestController
@RequestMapping("/api/v1/users")
@Tag(name = "Users", description = "User management operations")
public class UserController {

    @Operation(
        summary = "Create a new user",
        description = "Creates a new user account with the provided details. " +
                      "Returns the created user with generated ID.",
        operationId = "createUser",
        tags = {"Users", "Admin"}
    )
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "201",
            description = "User created successfully",
            content = @Content(
                mediaType = "application/json",
                schema = @Schema(implementation = UserResponse.class),
                examples = @ExampleObject(
                    name = "Success",
                    value = """
                        {
                            "id": 1,
                            "name": "John Doe",
                            "email": "john@example.com"
                        }
                        """
                )
            )
        ),
        @ApiResponse(
            responseCode = "400",
            description = "Invalid input data",
            content = @Content(
                mediaType = "application/json",
                schema = @Schema(implementation = ErrorResponse.class)
            )
        ),
        @ApiResponse(
            responseCode = "409",
            description = "User with email already exists",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        )
    })
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        // implementation
    }

    @Operation(
        summary = "Get user by ID",
        description = "Retrieves a user by their unique identifier",
        deprecated = false  // Mark as deprecated if needed
    )
    @ApiResponse(responseCode = "200", description = "User found")
    @ApiResponse(responseCode = "404", description = "User not found")
    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        // implementation
    }
}
```

---

## @Parameter Annotation

The `@Parameter` annotation documents request parameters, path variables, and headers.

```java
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.enums.ParameterIn;

@GetMapping
@Operation(summary = "Search users with pagination and filtering")
public Page<UserResponse> searchUsers(

    @Parameter(
        description = "Search query for user name or email",
        example = "john",
        required = false
    )
    @RequestParam(required = false) String search,

    @Parameter(
        description = "Filter by user status",
        schema = @Schema(
            allowableValues = {"ACTIVE", "INACTIVE", "PENDING"}
        )
    )
    @RequestParam(defaultValue = "ACTIVE") String status,

    @Parameter(
        description = "Page number (0-indexed)",
        example = "0",
        schema = @Schema(minimum = "0")
    )
    @RequestParam(defaultValue = "0") int page,

    @Parameter(
        description = "Page size",
        example = "20",
        schema = @Schema(minimum = "1", maximum = "100")
    )
    @RequestParam(defaultValue = "20") int size,

    @Parameter(
        description = "Sort field and direction",
        example = "createdAt,desc",
        array = @ArraySchema(schema = @Schema(type = "string"))
    )
    @RequestParam(defaultValue = "createdAt,desc") String[] sort
) {
    // implementation
}

@GetMapping("/{userId}/orders/{orderId}")
public OrderResponse getOrder(

    @Parameter(
        description = "User ID",
        required = true,
        in = ParameterIn.PATH
    )
    @PathVariable Long userId,

    @Parameter(
        description = "Order ID",
        required = true,
        in = ParameterIn.PATH
    )
    @PathVariable Long orderId,

    @Parameter(
        description = "Correlation ID for tracing",
        in = ParameterIn.HEADER,
        required = false
    )
    @RequestHeader(value = "X-Correlation-ID", required = false) String correlationId
) {
    // implementation
}
```

---

## @Schema for DTOs

Use `@Schema` to document request/response models with validation rules and examples.

```java
import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.*;

@Schema(description = "Request payload for creating a new user")
public class CreateUserRequest {

    @Schema(
        description = "User's full name",
        example = "John Doe",
        minLength = 2,
        maxLength = 100,
        requiredMode = Schema.RequiredMode.REQUIRED
    )
    @NotBlank
    @Size(min = 2, max = 100)
    private String name;

    @Schema(
        description = "User's email address",
        example = "john.doe@example.com",
        format = "email",
        requiredMode = Schema.RequiredMode.REQUIRED
    )
    @NotBlank
    @Email
    private String email;

    @Schema(
        description = "User's password",
        example = "SecureP@ss123",
        minLength = 8,
        maxLength = 50,
        accessMode = Schema.AccessMode.WRITE_ONLY  // Never shown in responses
    )
    @NotBlank
    @Size(min = 8, max = 50)
    private String password;

    @Schema(
        description = "User's age",
        example = "25",
        minimum = "18",
        maximum = "120"
    )
    @Min(18)
    @Max(120)
    private Integer age;

    @Schema(
        description = "Account type",
        allowableValues = {"STANDARD", "PREMIUM", "ENTERPRISE"},
        defaultValue = "STANDARD"
    )
    private String accountType = "STANDARD";

    @Schema(
        description = "User preferences",
        implementation = UserPreferences.class
    )
    private UserPreferences preferences;

    // getters and setters
}

@Schema(description = "User response with all public information")
public class UserResponse {

    @Schema(description = "Unique user identifier", example = "12345")
    private Long id;

    @Schema(description = "User's full name", example = "John Doe")
    private String name;

    @Schema(description = "User's email address", example = "john.doe@example.com")
    private String email;

    @Schema(
        description = "Account creation timestamp",
        example = "2024-01-15T10:30:00Z",
        format = "date-time",
        accessMode = Schema.AccessMode.READ_ONLY
    )
    private Instant createdAt;

    @Schema(description = "User roles", example = "[\"USER\", \"ADMIN\"]")
    private List<String> roles;

    // getters and setters
}

// Enum documentation
@Schema(description = "User account status")
public enum UserStatus {
    @Schema(description = "User is active and can log in")
    ACTIVE,

    @Schema(description = "User is temporarily suspended")
    INACTIVE,

    @Schema(description = "User registration pending email verification")
    PENDING
}
```

---

## Security Schemes (JWT Bearer, OAuth2)

### OpenAPI Configuration Class

```java
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.License;
import io.swagger.v3.oas.models.security.*;
import io.swagger.v3.oas.models.Components;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(apiInfo())
            .components(securityComponents())
            // Apply JWT security globally to all endpoints
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"));
    }

    private Info apiInfo() {
        return new Info()
            .title("User Management API")
            .description("RESTful API for managing users and accounts")
            .version("1.0.0")
            .contact(new Contact()
                .name("API Support")
                .email("support@example.com")
                .url("https://example.com/support"))
            .license(new License()
                .name("Apache 2.0")
                .url("https://www.apache.org/licenses/LICENSE-2.0"));
    }

    private Components securityComponents() {
        return new Components()
            // JWT Bearer Token
            .addSecuritySchemes("bearerAuth", new SecurityScheme()
                .type(SecurityScheme.Type.HTTP)
                .scheme("bearer")
                .bearerFormat("JWT")
                .description("Enter JWT token"))

            // API Key in header
            .addSecuritySchemes("apiKey", new SecurityScheme()
                .type(SecurityScheme.Type.APIKEY)
                .in(SecurityScheme.In.HEADER)
                .name("X-API-Key")
                .description("API key for service-to-service calls"))

            // OAuth2 with multiple flows
            .addSecuritySchemes("oauth2", new SecurityScheme()
                .type(SecurityScheme.Type.OAUTH2)
                .description("OAuth2 authentication")
                .flows(new OAuthFlows()
                    // Authorization Code flow (for web apps)
                    .authorizationCode(new OAuthFlow()
                        .authorizationUrl("https://auth.example.com/oauth/authorize")
                        .tokenUrl("https://auth.example.com/oauth/token")
                        .refreshUrl("https://auth.example.com/oauth/refresh")
                        .scopes(new Scopes()
                            .addString("read:users", "Read user information")
                            .addString("write:users", "Create and update users")
                            .addString("admin", "Full administrative access")))
                    // Client Credentials flow (for machine-to-machine)
                    .clientCredentials(new OAuthFlow()
                        .tokenUrl("https://auth.example.com/oauth/token")
                        .scopes(new Scopes()
                            .addString("read:users", "Read user information")))));
    }
}
```

### Applying Security at Controller/Method Level

```java
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.security.SecurityRequirements;

@RestController
@RequestMapping("/api/v1/admin")
@Tag(name = "Admin", description = "Administrative operations")
// Apply security to all methods in this controller
@SecurityRequirement(name = "bearerAuth")
public class AdminController {

    @GetMapping("/users")
    @Operation(summary = "List all users (admin only)")
    public List<UserResponse> listAllUsers() {
        // implementation
    }

    // Override with different security
    @GetMapping("/stats")
    @Operation(summary = "Get system statistics")
    @SecurityRequirements({
        @SecurityRequirement(name = "bearerAuth"),
        @SecurityRequirement(name = "apiKey")  // Either auth method works
    })
    public StatsResponse getStats() {
        // implementation
    }

    // Public endpoint - no security
    @GetMapping("/health")
    @Operation(summary = "Health check", security = {})  // Empty = no security
    public String health() {
        return "OK";
    }
}
```

---

## GroupedOpenApi for Multiple API Groups

Organize APIs into logical groups with separate documentation.

```java
import org.springdoc.core.models.GroupedOpenApi;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiGroupConfig {

    @Bean
    public GroupedOpenApi publicApi() {
        return GroupedOpenApi.builder()
            .group("public")
            .displayName("Public API")
            .pathsToMatch("/api/v1/public/**")
            .packagesToScan("com.example.api.controller.public")
            .build();
    }

    @Bean
    public GroupedOpenApi adminApi() {
        return GroupedOpenApi.builder()
            .group("admin")
            .displayName("Admin API")
            .pathsToMatch("/api/v1/admin/**")
            .packagesToScan("com.example.api.controller.admin")
            .addOpenApiCustomizer(openApi -> {
                openApi.info(new Info()
                    .title("Admin API")
                    .description("Administrative endpoints - requires elevated permissions")
                    .version("1.0.0"));
            })
            .build();
    }

    @Bean
    public GroupedOpenApi internalApi() {
        return GroupedOpenApi.builder()
            .group("internal")
            .displayName("Internal API")
            .pathsToMatch("/internal/**")
            .pathsToExclude("/internal/health", "/internal/metrics")
            .addOperationCustomizer((operation, handlerMethod) -> {
                // Add custom header to all operations in this group
                operation.addParametersItem(
                    new Parameter()
                        .in("header")
                        .name("X-Internal-Token")
                        .required(true)
                        .schema(new StringSchema())
                );
                return operation;
            })
            .build();
    }

    @Bean
    public GroupedOpenApi allApis() {
        return GroupedOpenApi.builder()
            .group("all")
            .displayName("All APIs")
            .pathsToMatch("/api/**", "/internal/**")
            .build();
    }
}
```

Access grouped documentation:
- `http://localhost:8080/swagger-ui.html?group=public`
- `http://localhost:8080/v3/api-docs/public`

---

## Customizing Swagger UI

### Application Configuration

```yaml
springdoc:
  swagger-ui:
    path: /swagger-ui.html
    configUrl: /v3/api-docs/swagger-config

    # Display settings
    displayOperationId: false
    displayRequestDuration: true
    defaultModelsExpandDepth: 1
    defaultModelExpandDepth: 3
    defaultModelRendering: model
    docExpansion: list              # none, list, full
    filter: true
    maxDisplayedTags: 50
    operationsSorter: alpha         # alpha, method
    showExtensions: true
    showCommonExtensions: true
    tagsSorter: alpha

    # Behavior settings
    tryItOutEnabled: true
    persistAuthorization: true
    queryConfigEnabled: false

    # Security settings
    oauth:
      clientId: swagger-ui
      clientSecret: swagger-ui-secret
      realm: swagger
      appName: Swagger UI
      scopeSeparator: ","
      useBasicAuthenticationWithAccessCodeGrant: true

    # Customization
    syntaxHighlight:
      activated: true
      theme: monokai              # agate, arta, monokai, nord, obsidian

    # Disable specific features
    supportedSubmitMethods:
      - get
      - post
      - put
      - delete
      - patch
```

### Programmatic Customization

```java
import org.springdoc.core.customizers.OpenApiCustomizer;
import org.springdoc.core.customizers.OperationCustomizer;

@Configuration
public class SwaggerUiCustomization {

    @Bean
    public OpenApiCustomizer globalHeaderCustomizer() {
        return openApi -> {
            openApi.getPaths().values().forEach(pathItem ->
                pathItem.readOperations().forEach(operation -> {
                    // Add global response for all operations
                    operation.getResponses().addApiResponse("500",
                        new ApiResponse()
                            .description("Internal server error")
                            .content(new Content()
                                .addMediaType("application/json",
                                    new MediaType()
                                        .schema(new Schema<>().$ref("#/components/schemas/ErrorResponse")))));
                })
            );
        };
    }

    @Bean
    public OperationCustomizer addCorrelationIdHeader() {
        return (operation, handlerMethod) -> {
            if (operation.getParameters() == null) {
                operation.setParameters(new ArrayList<>());
            }
            operation.getParameters().add(
                new Parameter()
                    .in("header")
                    .name("X-Correlation-ID")
                    .description("Request correlation ID for tracing")
                    .required(false)
                    .schema(new StringSchema().format("uuid"))
            );
            return operation;
        };
    }
}
```

---

## Hiding Endpoints

### Using @Hidden Annotation

```java
import io.swagger.v3.oas.annotations.Hidden;

// Hide entire controller
@Hidden
@RestController
@RequestMapping("/internal/maintenance")
public class MaintenanceController {
    // All endpoints hidden
}

// Hide specific methods
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @GetMapping
    @Operation(summary = "List users")
    public List<UserResponse> listUsers() { }

    @Hidden  // This endpoint won't appear in docs
    @DeleteMapping("/cache")
    public void clearCache() { }

    @Operation(hidden = true)  // Alternative way
    @PostMapping("/internal-sync")
    public void syncUsers() { }
}

// Hide model from schemas
@Hidden
@Schema
public class InternalMetrics {
    // Won't appear in component schemas
}
```

### Using Configuration

```yaml
springdoc:
  paths-to-exclude:
    - /actuator/**
    - /internal/**
    - /error
    - /api/v1/debug/**

  packages-to-exclude:
    - com.example.internal
    - com.example.debug
```

### Using GroupedOpenApi

```java
@Bean
public GroupedOpenApi publicApi() {
    return GroupedOpenApi.builder()
        .group("public")
        .pathsToMatch("/api/**")
        .pathsToExclude("/api/**/internal/**", "/api/**/debug/**")
        .build();
}
```

---

## Global Parameters and Headers

### Method 1: OperationCustomizer

```java
@Bean
public OperationCustomizer globalHeadersCustomizer() {
    return (operation, handlerMethod) -> {
        // Add tenant header
        operation.addParametersItem(
            new Parameter()
                .in("header")
                .name("X-Tenant-ID")
                .description("Tenant identifier for multi-tenant operations")
                .required(true)
                .schema(new StringSchema()));

        // Add accept-language header
        operation.addParametersItem(
            new Parameter()
                .in("header")
                .name("Accept-Language")
                .description("Preferred language for responses")
                .required(false)
                .schema(new StringSchema()
                    .addEnumItem("en")
                    .addEnumItem("es")
                    .addEnumItem("fr")
                    ._default("en")));

        return operation;
    };
}
```

### Method 2: OpenApiCustomizer

```java
@Bean
public OpenApiCustomizer globalParametersCustomizer() {
    return openApi -> {
        // Add reusable parameters to components
        if (openApi.getComponents().getParameters() == null) {
            openApi.getComponents().setParameters(new HashMap<>());
        }

        openApi.getComponents().getParameters().put("tenantId",
            new Parameter()
                .in("header")
                .name("X-Tenant-ID")
                .required(true)
                .schema(new StringSchema()));

        openApi.getComponents().getParameters().put("correlationId",
            new Parameter()
                .in("header")
                .name("X-Correlation-ID")
                .required(false)
                .schema(new StringSchema().format("uuid")));

        // Reference parameters in all operations
        openApi.getPaths().values().forEach(pathItem ->
            pathItem.readOperations().forEach(operation -> {
                operation.addParametersItem(
                    new Parameter().$ref("#/components/parameters/tenantId"));
                operation.addParametersItem(
                    new Parameter().$ref("#/components/parameters/correlationId"));
            }));
    };
}
```

### Method 3: Controller-Level with @Parameters

```java
@RestController
@RequestMapping("/api/v1/products")
@Tag(name = "Products")
public class ProductController {

    @Operation(summary = "List products")
    @Parameters({
        @Parameter(
            name = "X-Tenant-ID",
            in = ParameterIn.HEADER,
            required = true,
            description = "Tenant identifier"
        ),
        @Parameter(
            name = "X-Request-ID",
            in = ParameterIn.HEADER,
            required = false,
            description = "Request tracking ID"
        )
    })
    @GetMapping
    public List<ProductResponse> listProducts(
        @RequestHeader("X-Tenant-ID") String tenantId,
        @RequestHeader(value = "X-Request-ID", required = false) String requestId
    ) {
        // implementation
    }
}
```

---

## Best Practices

### 1. Use Meaningful Descriptions

```java
// Good
@Operation(
    summary = "Create a new order",
    description = "Creates a new order for the authenticated user. " +
                  "The order will be in PENDING status until payment is confirmed. " +
                  "Stock will be reserved for 15 minutes."
)

// Bad
@Operation(summary = "Create order")
```

### 2. Document All Response Codes

```java
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Order retrieved successfully"),
    @ApiResponse(responseCode = "400", description = "Invalid order ID format"),
    @ApiResponse(responseCode = "401", description = "Authentication required"),
    @ApiResponse(responseCode = "403", description = "Not authorized to view this order"),
    @ApiResponse(responseCode = "404", description = "Order not found")
})
```

### 3. Use Schema References for Common Models

```java
// Define common schemas
@Bean
public OpenAPI openAPI() {
    return new OpenAPI()
        .components(new Components()
            .addSchemas("Error", new Schema<ErrorResponse>()
                .type("object")
                .addProperty("code", new StringSchema())
                .addProperty("message", new StringSchema())
                .addProperty("timestamp", new StringSchema().format("date-time")))
            .addSchemas("PageInfo", new Schema<>()
                .type("object")
                .addProperty("page", new IntegerSchema())
                .addProperty("size", new IntegerSchema())
                .addProperty("totalElements", new IntegerSchema())
                .addProperty("totalPages", new IntegerSchema())));
}
```

### 4. Version Your APIs

```java
@Bean
public GroupedOpenApi v1Api() {
    return GroupedOpenApi.builder()
        .group("v1")
        .pathsToMatch("/api/v1/**")
        .addOpenApiCustomizer(api -> api.info(
            new Info().title("API v1").version("1.0.0")))
        .build();
}

@Bean
public GroupedOpenApi v2Api() {
    return GroupedOpenApi.builder()
        .group("v2")
        .pathsToMatch("/api/v2/**")
        .addOpenApiCustomizer(api -> api.info(
            new Info().title("API v2").version("2.0.0")))
        .build();
}
```

### 5. Disable in Production (Optional)

```yaml
# application-prod.yml
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
```

### 6. Use Examples Effectively

```java
@Schema(description = "User creation request")
public class CreateUserRequest {
    @Schema(
        description = "User email",
        example = "user@example.com",
        pattern = "^[\\w-\\.]+@([\\w-]+\\.)+[\\w-]{2,4}$"
    )
    private String email;
}
```

---

## Common Pitfalls

### 1. Missing Dependencies

Ensure you have the correct dependency for your Spring Boot version:
- Spring Boot 3.x: `springdoc-openapi-starter-webmvc-ui`
- Spring Boot 2.x: `springdoc-openapi-ui`

### 2. Security Not Appearing in Swagger UI

Ensure security schemes are properly referenced:

```java
// Must add SecurityRequirement at OpenAPI level or controller/method level
.addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
```

### 3. Models Not Showing in Schemas

Ensure DTOs are:
- Public classes with proper getters/setters
- Not annotated with `@Hidden`
- Actually referenced in controller methods

### 4. Path Variables Not Documented

Always use `@PathVariable` with matching names:

```java
// Correct
@GetMapping("/{userId}")
public User getUser(@PathVariable Long userId) { }

// Incorrect - parameter name mismatch
@GetMapping("/{userId}")
public User getUser(@PathVariable Long id) { }
```

### 5. Generics Not Resolved

Use `@Schema(implementation = ...)` for complex generics:

```java
@Operation(summary = "Get paginated users")
@ApiResponse(
    responseCode = "200",
    content = @Content(schema = @Schema(implementation = UserPage.class))
)
public Page<UserResponse> getUsers() { }

// Create a concrete type for the schema
@Schema(description = "Paginated user response")
class UserPage extends PageImpl<UserResponse> { }
```

### 6. CORS Issues with Swagger UI

Configure CORS for Swagger endpoints:

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/v3/api-docs/**").allowedOrigins("*");
        registry.addMapping("/swagger-ui/**").allowedOrigins("*");
    }
}
```

### 7. Actuator Endpoints Cluttering Documentation

Exclude actuator endpoints:

```yaml
springdoc:
  paths-to-exclude: /actuator/**
  show-actuator: false
```

---

## Quick Reference

| Annotation | Purpose |
|------------|---------|
| `@Tag` | Group related endpoints |
| `@Operation` | Document single endpoint |
| `@ApiResponse` | Document response code |
| `@Parameter` | Document parameter |
| `@Schema` | Document model/field |
| `@Hidden` | Hide from documentation |
| `@SecurityRequirement` | Apply security scheme |

| URL | Description |
|-----|-------------|
| `/swagger-ui.html` | Interactive UI |
| `/v3/api-docs` | OpenAPI JSON |
| `/v3/api-docs.yaml` | OpenAPI YAML |
| `/v3/api-docs/{group}` | Grouped API docs |

---

**Full Documentation:** https://springdoc.org/
