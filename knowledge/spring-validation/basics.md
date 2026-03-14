# Spring Validation

> Official Documentation: https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html

## Overview

Spring Validation integrates with the Bean Validation specification (JSR-380) to provide declarative validation for Java objects. It allows you to define validation rules using annotations directly on your model classes, ensuring data integrity throughout your application.

### Dependencies

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

```groovy
// Gradle
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

## Bean Validation Annotations

### String Constraints

```java
public class UserRequest {
    @NotNull(message = "Field cannot be null")
    private String requiredField;

    @NotEmpty(message = "Field cannot be null or empty")
    private String nonEmptyField;

    @NotBlank(message = "Field cannot be null, empty, or whitespace only")
    private String nonBlankField;

    @Size(min = 2, max = 100, message = "Must be between {min} and {max} characters")
    private String sizedField;

    @Email(message = "Must be a valid email address")
    private String email;

    @Email(regexp = ".*@company\\.com$", message = "Must be a company email")
    private String companyEmail;

    @Pattern(regexp = "^[A-Z]{2}\\d{6}$", message = "Must match format: XX123456")
    private String referenceCode;
}
```

### Numeric Constraints

```java
public class ProductRequest {
    @Min(value = 0, message = "Must be at least {value}")
    private Integer quantity;

    @Max(value = 1000, message = "Must be at most {value}")
    private Integer maxOrder;

    @Positive(message = "Must be a positive number")
    private BigDecimal price;

    @PositiveOrZero(message = "Must be zero or positive")
    private Integer stock;

    @Negative(message = "Must be a negative number")
    private Integer adjustment;

    @NegativeOrZero(message = "Must be zero or negative")
    private Integer discount;

    @DecimalMin(value = "0.01", inclusive = true, message = "Minimum value is {value}")
    private BigDecimal minPrice;

    @DecimalMax(value = "999999.99", message = "Maximum value is {value}")
    private BigDecimal maxPrice;

    @Digits(integer = 6, fraction = 2, message = "Must have at most {integer} integer digits and {fraction} decimal places")
    private BigDecimal amount;
}
```

### Date and Time Constraints

```java
public class EventRequest {
    @Past(message = "Must be a date in the past")
    private LocalDate birthDate;

    @PastOrPresent(message = "Must be a date in the past or present")
    private LocalDateTime createdAt;

    @Future(message = "Must be a date in the future")
    private LocalDate eventDate;

    @FutureOrPresent(message = "Must be a date in the future or present")
    private LocalDateTime scheduledAt;
}
```

### Collection Constraints

```java
public class OrderRequest {
    @NotEmpty(message = "At least one item is required")
    @Size(min = 1, max = 50, message = "Must have between {min} and {max} items")
    private List<OrderItem> items;
}
```

### Boolean Constraint

```java
public class TermsRequest {
    @AssertTrue(message = "You must accept the terms and conditions")
    private Boolean acceptedTerms;

    @AssertFalse(message = "This option must be disabled")
    private Boolean legacyMode;
}
```

## Validation Groups

Validation groups allow you to apply different validation rules in different scenarios (e.g., create vs. update operations).

### Defining Groups

```java
public interface ValidationGroups {
    interface Create {}
    interface Update {}
    interface Delete {}
}
```

### Applying Groups to Constraints

```java
@Data
public class UserRequest {
    @Null(groups = ValidationGroups.Create.class, message = "ID must be null for creation")
    @NotNull(groups = ValidationGroups.Update.class, message = "ID is required for update")
    private Long id;

    @NotBlank(groups = {ValidationGroups.Create.class, ValidationGroups.Update.class})
    @Size(min = 2, max = 100)
    private String name;

    @NotBlank(groups = ValidationGroups.Create.class)
    @Email
    private String email;

    @NotBlank(groups = ValidationGroups.Create.class)
    @Size(min = 8)
    private String password;
}
```

### Using Groups in Controllers

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<User> create(
            @Validated(ValidationGroups.Create.class) @RequestBody UserRequest request) {
        // Create logic
    }

    @PutMapping("/{id}")
    public ResponseEntity<User> update(
            @PathVariable Long id,
            @Validated(ValidationGroups.Update.class) @RequestBody UserRequest request) {
        // Update logic
    }
}
```

### Group Sequences

```java
@GroupSequence({BasicChecks.class, AdvancedChecks.class, DatabaseChecks.class})
public interface OrderedValidation {}

public class RegistrationRequest {
    @NotBlank(groups = BasicChecks.class)
    @Size(min = 3, max = 50, groups = BasicChecks.class)
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", groups = AdvancedChecks.class)
    @UniqueUsername(groups = DatabaseChecks.class)
    private String username;
}
```

## Custom Validators with @Constraint

### Creating a Custom Annotation

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
@Documented
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

### Implementing the Validator

```java
@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    private final UserRepository userRepository;

    public UniqueEmailValidator(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public void initialize(UniqueEmail constraintAnnotation) {
        // Optional initialization logic
    }

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null || email.isBlank()) {
            return true; // Let @NotBlank handle null/blank validation
        }
        return !userRepository.existsByEmail(email);
    }
}
```

### Custom Validator with Parameters

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = AllowedValuesValidator.class)
public @interface AllowedValues {
    String message() default "Value must be one of: {values}";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String[] values();
}

public class AllowedValuesValidator implements ConstraintValidator<AllowedValues, String> {
    private Set<String> allowedValues;

    @Override
    public void initialize(AllowedValues annotation) {
        this.allowedValues = Set.of(annotation.values());
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value == null || allowedValues.contains(value);
    }
}

// Usage
@AllowedValues(values = {"PENDING", "ACTIVE", "INACTIVE"})
private String status;
```

## Cross-Field Validation

### Class-Level Constraint

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchValidator.class)
public @interface PasswordMatch {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String passwordField() default "password";
    String confirmPasswordField() default "confirmPassword";
}
```

### Cross-Field Validator Implementation

```java
public class PasswordMatchValidator implements ConstraintValidator<PasswordMatch, Object> {
    private String passwordField;
    private String confirmPasswordField;

    @Override
    public void initialize(PasswordMatch annotation) {
        this.passwordField = annotation.passwordField();
        this.confirmPasswordField = annotation.confirmPasswordField();
    }

    @Override
    public boolean isValid(Object obj, ConstraintValidatorContext context) {
        try {
            BeanWrapper wrapper = new BeanWrapperImpl(obj);
            Object password = wrapper.getPropertyValue(passwordField);
            Object confirmPassword = wrapper.getPropertyValue(confirmPasswordField);

            boolean isValid = Objects.equals(password, confirmPassword);

            if (!isValid) {
                context.disableDefaultConstraintViolation();
                context.buildConstraintViolationWithTemplate(context.getDefaultConstraintMessageTemplate())
                       .addPropertyNode(confirmPasswordField)
                       .addConstraintViolation();
            }

            return isValid;
        } catch (Exception e) {
            return false;
        }
    }
}

// Usage
@PasswordMatch
@Data
public class RegistrationRequest {
    @NotBlank
    @Size(min = 8)
    private String password;

    @NotBlank
    private String confirmPassword;
}
```

### Date Range Validation

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = DateRangeValidator.class)
public @interface ValidDateRange {
    String message() default "End date must be after start date";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String startDate();
    String endDate();
}

@ValidDateRange(startDate = "startDate", endDate = "endDate")
public class EventRequest {
    @NotNull
    private LocalDate startDate;

    @NotNull
    private LocalDate endDate;
}
```

## Validating Nested Objects with @Valid

### Cascading Validation

```java
@Data
public class OrderRequest {
    @NotBlank
    private String orderNumber;

    @Valid  // Triggers validation on nested object
    @NotNull
    private CustomerInfo customer;

    @Valid  // Triggers validation on each item in the list
    @NotEmpty
    @Size(min = 1, max = 100)
    private List<OrderItem> items;
}

@Data
public class CustomerInfo {
    @NotBlank
    private String name;

    @NotBlank
    @Email
    private String email;

    @Valid
    @NotNull
    private Address shippingAddress;
}

@Data
public class Address {
    @NotBlank
    private String street;

    @NotBlank
    private String city;

    @NotBlank
    @Pattern(regexp = "^\\d{5}(-\\d{4})?$")
    private String zipCode;
}

@Data
public class OrderItem {
    @NotBlank
    private String productId;

    @Positive
    private Integer quantity;

    @PositiveOrZero
    private BigDecimal price;
}
```

## Controller Validation with @Valid and @Validated

### @Valid vs @Validated

```java
@RestController
@RequestMapping("/api/products")
@Validated  // Required for method-level validation (path variables, request params)
public class ProductController {

    // @Valid - Standard JSR-380 annotation, works for @RequestBody
    @PostMapping
    public ResponseEntity<Product> create(@Valid @RequestBody ProductRequest request) {
        // ...
    }

    // @Validated - Spring-specific, supports validation groups
    @PostMapping("/v2")
    public ResponseEntity<Product> createV2(
            @Validated(ValidationGroups.Create.class) @RequestBody ProductRequest request) {
        // ...
    }

    // Path variable and request param validation requires @Validated on class
    @GetMapping("/{id}")
    public ResponseEntity<Product> findById(
            @PathVariable @Min(1) Long id) {
        // ...
    }

    @GetMapping
    public ResponseEntity<Page<Product>> search(
            @RequestParam @NotBlank String query,
            @RequestParam @Min(0) int page,
            @RequestParam @Min(1) @Max(100) int size) {
        // ...
    }
}
```

## Error Handling with BindingResult

### Manual Error Handling

```java
@PostMapping
public ResponseEntity<?> create(
        @Valid @RequestBody UserRequest request,
        BindingResult bindingResult) {

    if (bindingResult.hasErrors()) {
        Map<String, String> errors = bindingResult.getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                error -> error.getDefaultMessage() != null ? error.getDefaultMessage() : "Invalid value",
                (existing, replacement) -> existing  // Handle duplicate keys
            ));
        return ResponseEntity.badRequest().body(errors);
    }

    // Process valid request
    User user = userService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(user);
}
```

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidationErrors(MethodArgumentNotValidException ex) {
        List<FieldError> fieldErrors = ex.getBindingResult().getFieldErrors();

        Map<String, List<String>> errors = fieldErrors.stream()
            .collect(Collectors.groupingBy(
                FieldError::getField,
                Collectors.mapping(
                    error -> error.getDefaultMessage() != null ? error.getDefaultMessage() : "Invalid",
                    Collectors.toList()
                )
            ));

        ApiError apiError = new ApiError(
            HttpStatus.BAD_REQUEST.value(),
            "Validation failed",
            errors
        );

        return ResponseEntity.badRequest().body(apiError);
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ApiError> handleConstraintViolation(ConstraintViolationException ex) {
        Map<String, List<String>> errors = ex.getConstraintViolations().stream()
            .collect(Collectors.groupingBy(
                violation -> violation.getPropertyPath().toString(),
                Collectors.mapping(ConstraintViolation::getMessage, Collectors.toList())
            ));

        ApiError apiError = new ApiError(
            HttpStatus.BAD_REQUEST.value(),
            "Constraint violation",
            errors
        );

        return ResponseEntity.badRequest().body(apiError);
    }
}

@Data
@AllArgsConstructor
public class ApiError {
    private int status;
    private String message;
    private Map<String, List<String>> errors;
}
```

## Message Customization

### Using ValidationMessages.properties

```properties
# src/main/resources/ValidationMessages.properties
javax.validation.constraints.NotBlank.message=This field is required
javax.validation.constraints.Size.message=Must be between {min} and {max} characters
javax.validation.constraints.Email.message=Please provide a valid email address

# Custom messages
user.name.required=User name is required
user.email.invalid=Please enter a valid email address
user.password.weak=Password must contain at least one uppercase letter, one number, and be at least 8 characters
```

### Using Message Keys in Annotations

```java
public class UserRequest {
    @NotBlank(message = "{user.name.required}")
    private String name;

    @Email(message = "{user.email.invalid}")
    private String email;

    @Pattern(regexp = "^(?=.*[A-Z])(?=.*\\d).{8,}$", message = "{user.password.weak}")
    private String password;
}
```

### Interpolating Values in Messages

```java
public class ProductRequest {
    @Size(min = 2, max = 100, message = "Name must be between {min} and {max} characters")
    private String name;

    @DecimalMin(value = "0.01", message = "Price must be at least ${value}")
    private BigDecimal price;
}
```

### Custom Message Interpolator

```java
@Configuration
public class ValidationConfig {

    @Bean
    public LocalValidatorFactoryBean validator(MessageSource messageSource) {
        LocalValidatorFactoryBean bean = new LocalValidatorFactoryBean();
        bean.setValidationMessageSource(messageSource);
        return bean;
    }
}
```

## Method-Level Validation

### Service Layer Validation

```java
@Service
@Validated  // Enable method-level validation
public class UserService {

    public User createUser(@Valid UserRequest request) {
        // Validation happens before method execution
        return userRepository.save(mapToEntity(request));
    }

    @NotNull
    public User findById(@NotNull @Min(1) Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
    }

    public void updateEmail(
            @NotNull @Min(1) Long userId,
            @NotBlank @Email String newEmail) {
        // Both parameters are validated
        User user = findById(userId);
        user.setEmail(newEmail);
        userRepository.save(user);
    }

    @NotEmpty
    public List<User> findByStatus(@NotNull UserStatus status) {
        // Return value is validated to not be empty
        return userRepository.findByStatus(status);
    }
}
```

### Validating Return Values

```java
@Service
@Validated
public class OrderService {

    @Valid  // Validates the returned object
    @NotNull
    public Order processOrder(@Valid OrderRequest request) {
        Order order = createOrder(request);
        return order;  // Returned Order object will be validated
    }
}
```

## Best Practices

### 1. Fail Fast with Validation Groups

```java
// Define a sequence to validate in order
@GroupSequence({BasicValidation.class, BusinessValidation.class, Default.class})
public interface OrderedValidation {}

// Stop at first group that fails
@Validated(OrderedValidation.class)
```

### 2. Separate DTOs from Entities

```java
// Good: Dedicated request/response DTOs
public class CreateUserRequest { ... }
public class UpdateUserRequest { ... }
public class UserResponse { ... }

// Bad: Using entities directly in controllers
```

### 3. Use Meaningful Error Messages

```java
// Good
@Size(min = 8, message = "Password must be at least 8 characters for security")

// Bad
@Size(min = 8)  // Uses generic message
```

### 4. Validate at the Right Layer

```java
// Controller layer: Input validation
@PostMapping
public ResponseEntity<User> create(@Valid @RequestBody UserRequest request) { }

// Service layer: Business rule validation
@Service
@Validated
public class UserService {
    public User create(@Valid UserRequest request) {
        // Additional business validation
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new BusinessException("Email already registered");
        }
    }
}
```

### 5. Compose Validators for Reusability

```java
@NotBlank
@Size(min = 8, max = 100)
@Pattern(regexp = "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d).*$")
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = {})
public @interface StrongPassword {
    String message() default "Password does not meet security requirements";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Usage
@StrongPassword
private String password;
```

## Common Pitfalls

### 1. Forgetting @Valid for Nested Objects

```java
// Wrong: Nested objects won't be validated
public class OrderRequest {
    @NotNull
    private CustomerInfo customer;  // Fields inside CustomerInfo are NOT validated
}

// Correct: Add @Valid
public class OrderRequest {
    @Valid
    @NotNull
    private CustomerInfo customer;  // Now CustomerInfo fields ARE validated
}
```

### 2. Missing @Validated on Class for Parameter Validation

```java
// Wrong: @Min won't work on path variable
@RestController
public class UserController {
    @GetMapping("/{id}")
    public User findById(@PathVariable @Min(1) Long id) { }  // Validation ignored!
}

// Correct: Add @Validated to class
@RestController
@Validated
public class UserController {
    @GetMapping("/{id}")
    public User findById(@PathVariable @Min(1) Long id) { }  // Now works
}
```

### 3. Not Handling Null in Custom Validators

```java
// Wrong: May throw NullPointerException
public boolean isValid(String value, ConstraintValidatorContext context) {
    return value.length() > 5;  // NPE if value is null
}

// Correct: Handle null explicitly
public boolean isValid(String value, ConstraintValidatorContext context) {
    if (value == null) {
        return true;  // Let @NotNull handle null validation
    }
    return value.length() > 5;
}
```

### 4. Validation Order Dependency

```java
// Wrong: Assumes validations run in order
public class DateRangeRequest {
    @NotNull
    private LocalDate startDate;

    @NotNull
    @Future  // This might run before @NotNull on startDate
    private LocalDate endDate;
}

// Correct: Use validation groups for ordering
@GroupSequence({NotNullChecks.class, DateChecks.class})
public interface OrderedDateValidation {}
```

### 5. BindingResult Not Immediately After @Valid

```java
// Wrong: BindingResult must immediately follow @Valid parameter
@PostMapping
public ResponseEntity<?> create(
        @Valid @RequestBody UserRequest request,
        @RequestHeader String auth,  // This breaks the chain
        BindingResult result) { }

// Correct: BindingResult immediately after validated parameter
@PostMapping
public ResponseEntity<?> create(
        @Valid @RequestBody UserRequest request,
        BindingResult result,
        @RequestHeader String auth) { }
```

### 6. Using Wrong Annotation for Strings

```java
// Wrong: @NotNull allows empty strings
@NotNull
private String name;  // "" passes validation

// Correct: Use @NotBlank for required strings
@NotBlank
private String name;  // "" fails validation
```

### 7. Circular Validation with @Valid

```java
// Be careful with bidirectional relationships
public class Parent {
    @Valid
    private List<Child> children;
}

public class Child {
    @Valid  // Could cause infinite loop
    private Parent parent;
}

// Solution: Only validate in one direction or use DTOs
```
