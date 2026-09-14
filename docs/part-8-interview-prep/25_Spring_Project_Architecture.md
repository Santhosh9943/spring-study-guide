# Real-World Project Architecture

> **File:** `25_Spring_Project_Architecture.md`
> **Part:** 8 — Practice & Interview Prep
> **Prerequisites:** All previous files (especially 01–16)
> **Estimated Study Time:** 12–16 hours (read + code + apply to a real project)

---

## Table of Contents

1. [Layered Architecture](#1-layered-architecture)
2. [Package Structure](#2-package-structure)
3. [DTO Pattern & MapStruct](#3-dto-pattern--mapstruct)
4. [Exception Handling Strategy](#4-exception-handling-strategy)
5. [Logging Strategy](#5-logging-strategy)
6. [Validation Strategy](#6-validation-strategy)
7. [Testing Strategy](#7-testing-strategy)
8. [CI/CD with GitHub Actions & Jenkins](#8-cicd-with-github-actions--jenkins)
9. [Dockerizing Spring Boot](#9-dockerizing-spring-boot)
10. [Kubernetes Deployment](#10-kubernetes-deployment)
11. [Database Migration with Flyway/Liquibase](#11-database-migration-with-flywayliquibase)
12. [Monitoring & Observability](#12-monitoring--observability)
13. [Security Hardening for Production](#13-security-hardening-for-production)
14. [Performance Tuning](#14-performance-tuning)
15. [Complete Project Example](#15-complete-project-example)
16. [Interview Questions](#16-interview-questions)

---

## 1. Layered Architecture

### The Classic 4-Layer Architecture

```
┌─────────────────────────────────────────────────────┐
│                  PRESENTATION                        │
│         (Controllers, DTOs, Exception Handlers)      │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                    SERVICE                           │
│      (Business Logic, Transactions, Validation)      │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                   REPOSITORY                         │
│         (Data Access, JPA, Queries)                  │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                    DOMAIN                            │
│              (Entities, Value Objects)               │
└─────────────────────────────────────────────────────┘
```

### Responsibilities

| Layer | Responsibility | Should NOT |
|-------|---------------|------------|
| **Presentation** | HTTP handling, DTO mapping, validation | Contain business logic |
| **Service** | Business rules, transactions, orchestration | Know HTTP |
| **Repository** | Data access, queries | Contain business logic |
| **Domain** | Entities, value objects, domain logic | Know about persistence |

### Anti-Patterns to Avoid

❌ **Fat Controller** — business logic in controller
❌ **Anemic Domain Model** — entities with only getters/setters
❌ **Service calling controller** — inverted dependency
❌ **Repository with business logic** — wrong layer
❌ **Entity leaking to controller** — tight coupling

### Hexagonal Architecture (Ports & Adapters)

```
                  ┌─────────────────────┐
   REST ─────────►│                     │
                  │      DOMAIN         │
   gRPC ─────────►│   (Business Core)   │◄───────── Kafka
                  │                     │
   CLI  ─────────►│                     │◄───────── DB
                  └─────────────────────┘
                    Ports (interfaces)
                    Adapters (implementations)
```

**Benefits:**
- Domain independent of frameworks
- Testable core
- Swap adapters (REST → gRPC, JPA → JDBC)

### When to Use Which

| Architecture | When |
|-------------|------|
| **Layered** | CRUD apps, small/medium projects |
| **Hexagonal** | Complex domains, multiple I/O |
| **Clean/Onion** | Large enterprise, long-lived |
| **CQRS** | Read/write asymmetry |
| **Event-Driven** | Async workflows, microservices |

---

## 2. Package Structure

### Feature-Based (Recommended)

```
com.example.app
├── Application.java
├── config/
│   ├── SecurityConfig.java
│   ├── OpenApiConfig.java
│   └── JacksonConfig.java
├── common/
│   ├── exception/
│   │   ├── GlobalExceptionHandler.java
│   │   ├── ResourceNotFoundException.java
│   │   └── ErrorResponse.java
│   ├── audit/
│   │   └── AuditableEntity.java
│   └── util/
├── user/
│   ├── UserController.java
│   ├── UserService.java
│   ├── UserRepository.java
│   ├── User.java
│   ├── dto/
│   │   ├── CreateUserRequest.java
│   │   ├── UserResponse.java
│   │   └── UserMapper.java
│   └── UserServiceTest.java
├── order/
│   ├── OrderController.java
│   ├── OrderService.java
│   ├── OrderRepository.java
│   ├── Order.java
│   ├── OrderItem.java
│   └── dto/
└── payment/
    ├── PaymentController.java
    ├── PaymentService.java
    ├── PaymentGateway.java          (port)
    └── gateway/
        └── StripePaymentGateway.java (adapter)
```

### Layer-Based (Traditional)

```
com.example.app
├── controller/
│   ├── UserController.java
│   └── OrderController.java
├── service/
│   ├── UserService.java
│   └── OrderService.java
├── repository/
│   ├── UserRepository.java
│   └── OrderRepository.java
├── entity/
│   ├── User.java
│   └── Order.java
├── dto/
│   ├── UserDTO.java
│   └── OrderDTO.java
├── exception/
├── config/
└── util/
```

### Comparison

| Aspect | Feature-Based | Layer-Based |
|--------|--------------|-------------|
| Cohesion | High (related files together) | Low |
| Navigation | Easy for new features | Easy for layer changes |
| Scalability | ✅ Modular | ❌ Cross-cutting changes |
| Microservices migration | ✅ Extract module | ❌ Hard |
| Small projects | Overkill | ✅ Simple |

**Recommendation:** **Feature-based** for anything non-trivial.

### Maven Multi-Module Structure

```
parent/
├── pom.xml                    (packaging: pom)
├── common/                    (shared utilities)
│   └── pom.xml
├── domain/                    (entities, value objects)
│   └── pom.xml
├── application/               (services, use cases)
│   └── pom.xml
├── infrastructure/            (adapters: JPA, Kafka, REST clients)
│   └── pom.xml
├── api/                       (REST controllers)
│   └── pom.xml
└── boot/                      (Spring Boot app)
    └── pom.xml
```

**Dependency direction:** `boot → api → application → domain` and `infrastructure → application → domain`. Domain depends on nothing.

---

## 3. DTO Pattern & MapStruct

### Why DTOs?

- **Decouple** API contract from DB schema
- **Prevent over-posting** (mass assignment)
- **Control serialization** (hide sensitive fields)
- **Shape data** for the client
- **Version APIs** independently

### DTO Types

| DTO | Purpose |
|-----|---------|
| `CreateUserRequest` | Input for POST |
| `UpdateUserRequest` | Input for PUT/PATCH |
| `UserResponse` | Output |
| `UserSummary` | Lightweight list view |
| `UserDetail` | Full detail view |
| `UserFilter` | Query parameters |

### Example DTOs (Java Records)

```java
public record CreateUserRequest(
    @NotBlank @Size(min = 2, max = 100) String name,
    @NotBlank @Email String email,
    @NotNull @Min(18) Integer age
) { }

public record UserResponse(
    Long id,
    String name,
    String email,
    Integer age,
    Instant createdAt,
    List<String> roles
) { }

public record UserSummary(Long id, String name) { }
```

### MapStruct — Type-Safe Mapping

**Dependency:**

```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.6.3</version>
</dependency>
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>1.6.3</version>
    <scope>provided</scope>
</dependency>
```

**Mapper interface:**

```java
@Mapper(componentModel = "spring",
        unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface UserMapper {
    
    UserResponse toResponse(User user);
    
    UserSummary toSummary(User user);
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "roles", ignore = true)
    User toEntity(CreateUserRequest req);
    
    List<UserResponse> toResponseList(List<User> users);
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "email", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    void updateEntity(UpdateUserRequest req, @MappingTarget User user);
    
    // Nested mapping
    @Mapping(source = "address.city", target = "city")
    @Mapping(source = "address.country", target = "country")
    UserDetail toDetail(User user);
    
    // Custom method
    default String formatName(User user) {
        return user.getFirstName() + " " + user.getLastName();
    }
}
```

**Usage:**

```java
@Service
public class UserService {
    private final UserMapper mapper;
    
    public UserService(UserMapper mapper) {
        this.mapper = mapper;
    }
    
    public UserResponse findById(Long id) {
        User user = userRepo.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
        return mapper.toResponse(user);
    }
}
```

### Alternative: ModelMapper

```java
@Bean
public ModelMapper modelMapper() {
    ModelMapper m = new ModelMapper();
    m.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT);
    return m;
}

// Usage
UserResponse response = modelMapper.map(user, UserResponse.class);
```

### Manual Mapping

```java
public UserResponse toResponse(User user) {
    return new UserResponse(
        user.getId(),
        user.getName(),
        user.getEmail(),
        user.getAge(),
        user.getCreatedAt(),
        user.getRoles().stream().map(Role::getName).toList()
    );
}
```

### Comparison

| Tool | Pros | Cons |
|------|------|------|
| **MapStruct** | Compile-time, fast, type-safe | Code generation, IDE setup |
| **ModelMapper** | Runtime, flexible | Slow, reflection, silent failures |
| **Manual** | Zero deps, explicit | Verbose |
| **Records + constructor** | Concise, immutable | Still manual |

**Recommendation:** **MapStruct** for large projects; manual for small.

### DTO Best Practices

1. **Never expose entities** in controllers
2. Use **records** for immutability
3. **Validate** on input DTOs
4. **Map** in the service layer (not controller)
5. **Separate** read/write DTOs
6. **Project** only needed fields
7. **Ignore** sensitive fields explicitly

---

## 4. Exception Handling Strategy

### Exception Hierarchy

```
Throwable
├── Error                         (don't catch)
└── Exception
    ├── IOException               (checked)
    ├── SQLException              (checked)
    └── RuntimeException          (unchecked)
        ├── BusinessException     (custom)
        │   ├── ResourceNotFoundException
        │   ├── DuplicateResourceException
        │   ├── BusinessRuleException
        │   └── ValidationException
        └── TechnicalException    (custom)
            ├── IntegrationException
            └── ConfigurationException
```

### Custom Exceptions

```java
// Base
public abstract class ApplicationException extends RuntimeException {
    private final String code;
    private final HttpStatus status;
    
    protected ApplicationException(String code, String message, HttpStatus status) {
        super(message);
        this.code = code;
        this.status = status;
    }
    
    public String getCode() { return code; }
    public HttpStatus getStatus() { return status; }
}

// Specific
public class ResourceNotFoundException extends ApplicationException {
    public ResourceNotFoundException(String resource, Object id) {
        super("RESOURCE_NOT_FOUND",
              resource + " not found with id: " + id,
              HttpStatus.NOT_FOUND);
    }
}

public class DuplicateResourceException extends ApplicationException {
    public DuplicateResourceException(String resource, String field, Object value) {
        super("DUPLICATE_RESOURCE",
              resource + " already exists with " + field + ": " + value,
              HttpStatus.CONFLICT);
    }
}

public class BusinessRuleException extends ApplicationException {
    public BusinessRuleException(String message) {
        super("BUSINESS_RULE_VIOLATION", message, HttpStatus.UNPROCESSABLE_ENTITY);
    }
}

public class IntegrationException extends ApplicationException {
    public IntegrationException(String service, String message, Throwable cause) {
        super("INTEGRATION_ERROR",
              "Failed to communicate with " + service + ": " + message,
              HttpStatus.BAD_GATEWAY);
        initCause(cause);
    }
}
```

### Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ApplicationException.class)
    public ResponseEntity<ProblemDetail> handleApplication(ApplicationException ex,
                                                            HttpServletRequest req) {
        log.warn("Application exception: code={}, message={}, path={}",
                 ex.getCode(), ex.getMessage(), req.getRequestURI());
        
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(ex.getStatus(), ex.getMessage());
        pd.setTitle(ex.getCode());
        pd.setProperty("code", ex.getCode());
        pd.setProperty("timestamp", Instant.now());
        pd.setProperty("path", req.getRequestURI());
        return ResponseEntity.status(ex.getStatus()).body(pd);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ProblemDetail> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest req) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(err ->
            errors.put(err.getField(), err.getDefaultMessage()));
        ex.getBindingResult().getGlobalErrors().forEach(err ->
            errors.put("_global", err.getDefaultMessage()));
        
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, "Validation failed");
        pd.setTitle("VALIDATION_ERROR");
        pd.setProperty("code", "VALIDATION_ERROR");
        pd.setProperty("errors", errors);
        pd.setProperty("timestamp", Instant.now());
        pd.setProperty("path", req.getRequestURI());
        return ResponseEntity.badRequest().body(pd);
    }
    
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ProblemDetail> handleConstraint(
            ConstraintViolationException ex, HttpServletRequest req) {
        Map<String, String> errors = new HashMap<>();
        ex.getConstraintViolations().forEach(cv ->
            errors.put(cv.getPropertyPath().toString(), cv.getMessage()));
        
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, "Constraint violation");
        pd.setTitle("VALIDATION_ERROR");
        pd.setProperty("errors", errors);
        return ResponseEntity.badRequest().body(pd);
    }
    
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ProblemDetail> handleAccessDenied(
            AccessDeniedException ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.FORBIDDEN, "Access denied");
        pd.setTitle("FORBIDDEN");
        pd.setProperty("path", req.getRequestURI());
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(pd);
    }
    
    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ProblemDetail> handleDataIntegrity(
            DataIntegrityViolationException ex, HttpServletRequest req) {
        log.warn("Data integrity violation", ex);
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT, "Data integrity violation");
        pd.setTitle("CONFLICT");
        return ResponseEntity.status(HttpStatus.CONFLICT).body(pd);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleAll(Exception ex,
                                                    HttpServletRequest req) {
        log.error("Unhandled exception at {}", req.getRequestURI(), ex);
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected error occurred");
        pd.setTitle("INTERNAL_ERROR");
        pd.setProperty("timestamp", Instant.now());
        pd.setProperty("path", req.getRequestURI());
        return ResponseEntity.internalServerError().body(pd);
    }
}
```

### Error Response Format (RFC 7807)

```json
{
  "type": "about:blank",
  "title": "RESOURCE_NOT_FOUND",
  "status": 404,
  "detail": "User not found with id: 123",
  "instance": "/api/v1/users/123",
  "code": "RESOURCE_NOT_FOUND",
  "timestamp": "2024-01-15T10:30:00Z",
  "path": "/api/v1/users/123"
}
```

### Exception Handling Rules

1. **Don't catch and swallow** — log or rethrow
2. **Don't catch `Exception`** unless at boundary
3. **Don't use exceptions for control flow**
4. **Include context** in messages (IDs, params)
5. **Never leak stack traces** to clients
6. **Log server errors (5xx) at ERROR, client (4xx) at WARN**
7. **Use correlation IDs** for tracing
8. **Map to correct status codes**

---

## 5. Logging Strategy

### SLF4J + Logback (Default)

**Dependency** (included in `spring-boot-starter`):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
</dependency>
```

### Log Levels

| Level | Use For |
|-------|---------|
| **TRACE** | Very detailed (never in prod) |
| **DEBUG** | Development diagnostics |
| **INFO** | Business events, startup, key actions |
| **WARN** | Recoverable issues, deprecations |
| **ERROR** | Failures requiring attention |

### Structured Logging (JSON)

```xml
<!-- logback-spring.xml -->
<configuration>
    <springProfile name="!prod">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
    </springProfile>
    
    <springProfile name="prod">
        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeMdcKeyName>traceId</includeMdcKeyName>
                <includeMdcKeyName>spanId</includeMdcKeyName>
                <includeMdcKeyName>userId</includeMdcKeyName>
                <includeMdcKeyName>correlationId</includeMdcKeyName>
                <customFields>{"app":"${spring.application.name}"}</customFields>
            </encoder>
        </appender>
    </springProfile>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="JSON"/>
    </root>
    
    <logger name="com.example.app" level="DEBUG"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
</configuration>
```

**Dependency:**

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>8.0</version>
</dependency>
```

### MDC for Correlation IDs

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {
    
    private static final String CORRELATION_ID_HEADER = "X-Correlation-Id";
    private static final String CORRELATION_ID_MDC = "correlationId";
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {
        String correlationId = request.getHeader(CORRELATION_ID_HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }
        
        MDC.put(CORRELATION_ID_MDC, correlationId);
        response.setHeader(CORRELATION_ID_HEADER, correlationId);
        
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

### Structured Logging with Key-Value

```java
import net.logstash.logback.argument.StructuredArguments;

log.info("Order created",
    kv("orderId", order.getId()),
    kv("userId", order.getUserId()),
    kv("amount", order.getTotal()));
```

**JSON output:**
```json
{
  "message": "Order created",
  "orderId": 123,
  "userId": 456,
  "amount": 99.99,
  "level": "INFO",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Logging Best Practices

1. **Log at boundaries** — controller entry/exit, external calls
2. **Never log** passwords, tokens, PII, full credit cards
3. **Use parameterized logging** — `log.info("User {}", id)` not string concat
4. **Log exceptions with stack trace** — `log.error("...", ex)`
5. **Correlation IDs** in every log
6. **Consistent format** across services
7. **Avoid logging in loops** — aggregate
8. **Mask sensitive data** with custom converters

### Sensitive Data Masking

```java
public class MaskingConverter extends ClassicConverter {
    @Override
    public String convert(ILoggingEvent event) {
        String msg = event.getFormattedMessage();
        return msg.replaceAll("\\b\\d{16}\\b", "****-****-****-****")
                  .replaceAll("(?i)password=\\S+", "password=***");
    }
}
```

---

## 6. Validation Strategy

### Bean Validation (Jakarta)

```java
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be 2-100 chars")
    String name,
    
    @NotBlank @Email(message = "Valid email required")
    String email,
    
    @NotNull @Min(18) @Max(120)
    Integer age,
    
    @Pattern(regexp = "^\\+?[1-9]\\d{1,14}$", message = "Invalid phone")
    String phone,
    
    @Valid @NotNull
    AddressRequest address,
    
    @NotEmpty @Size(max = 5)
    List<@NotBlank String> tags
) { }

public record AddressRequest(
    @NotBlank String street,
    @NotBlank String city,
    @Pattern(regexp = "^\\d{5}(-\\d{4})?$") String zip
) { }
```

### Triggering Validation

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public UserResponse create(@Valid @RequestBody CreateUserRequest req) {
    return userService.create(req);
}

// For @PathVariable / @RequestParam validation
@RestController
@Validated   // ← required for method param validation
public class UserController {
    
    @GetMapping("/{id}")
    public UserResponse get(
            @PathVariable @Positive Long id,
            @RequestParam @Min(0) @Max(100) int size) {
        return userService.findById(id);
    }
}
```

### Custom Validators

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    
    private final UserRepository userRepository;
    
    public UniqueEmailValidator(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Override
    public boolean isValid(String email, ConstraintValidatorContext ctx) {
        if (email == null) return true;
        return !userRepository.existsByEmailIgnoreCase(email);
    }
}
```

### Cross-Field Validation

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchesValidator.class)
public @interface PasswordMatches {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PasswordMatchesValidator
        implements ConstraintValidator<PasswordMatches, Object> {
    
    @Override
    public boolean isValid(Object obj, ConstraintValidatorContext ctx) {
        var req = (RegisterRequest) obj;
        return Objects.equals(req.password(), req.confirmPassword());
    }
}

@PasswordMatches
public record RegisterRequest(String password, String confirmPassword) { }
```

### Validation Groups

```java
public interface Create {}
public interface Update {}

public record UserRequest(
    @Null(groups = Create.class)
    @NotNull(groups = Update.class)
    Long id,
    
    @NotBlank(groups = {Create.class, Update.class})
    String name
) { }

// Controller
@PostMapping
public UserResponse create(@Validated(Create.class) @RequestBody UserRequest req) { }

@PutMapping("/{id}")
public UserResponse update(@Validated(Update.class) @RequestBody UserRequest req) { }
```

### Validation Best Practices

1. **Validate at the boundary** (controller)
2. **Use DTOs** — never entities
3. **Custom messages** — user-friendly
4. **Group validations** for create/update
5. **Cross-field** via class-level constraints
6. **Database constraints** as last line of defense
7. **Fail fast** — return all errors at once

---

## 7. Testing Strategy

### Test Pyramid

```
        ┌─────────┐
        │   E2E   │  Few, slow, expensive
        ├─────────┤
        │Integration│  Some, medium
        ├─────────┤
        │  Unit   │  Many, fast, cheap
        └─────────┘
```

### Test Types

| Type | Scope | Speed | Tools |
|------|-------|-------|-------|
| **Unit** | Single class | ms | JUnit, Mockito, AssertJ |
| **Slice** | One layer | 100ms | `@WebMvcTest`, `@DataJpaTest` |
| **Integration** | Full context | seconds | `@SpringBootTest`, Testcontainers |
| **Contract** | API compatibility | seconds | Spring Cloud Contract, Pact |
| **E2E** | Full system | minutes | RestAssured, Selenium |

### Unit Test

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock private UserRepository userRepository;
    @Mock private UserMapper userMapper;
    @Mock private EmailService emailService;
    
    @InjectMocks private UserService userService;
    
    @Test
    void createUser_savesAndSendsWelcome() {
        // Given
        var req = new CreateUserRequest("Alice", "alice@example.com", 30);
        var entity = new User();
        entity.setId(1L);
        var response = new UserResponse(1L, "Alice", "alice@example.com", 30, null, List.of());
        
        when(userRepository.existsByEmailIgnoreCase("alice@example.com")).thenReturn(false);
        when(userMapper.toEntity(req)).thenReturn(entity);
        when(userRepository.save(entity)).thenReturn(entity);
        when(userMapper.toResponse(entity)).thenReturn(response);
        
        // When
        UserResponse result = userService.create(req);
        
        // Then
        assertThat(result).isEqualTo(response);
        verify(emailService).sendWelcome("alice@example.com");
        verify(userRepository).save(entity);
    }
    
    @Test
    void createUser_throwsWhenEmailExists() {
        var req = new CreateUserRequest("Alice", "alice@example.com", 30);
        when(userRepository.existsByEmailIgnoreCase("alice@example.com")).thenReturn(true);
        
        assertThatThrownBy(() -> userService.create(req))
            .isInstanceOf(DuplicateResourceException.class)
            .hasMessageContaining("alice@example.com");
        
        verify(userRepository, never()).save(any());
    }
}
```

### Web Layer Slice Test

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;
    
    @MockBean private UserService userService;
    @MockBean private UserMapper userMapper;
    
    @Test
    void getById_returnsUser() throws Exception {
        var response = new UserResponse(1L, "Alice", "alice@example.com", 30, null, List.of());
        when(userService.findById(1L)).thenReturn(response);
        
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("Alice"))
            .andExpect(jsonPath("$.email").value("alice@example.com"));
    }
    
    @Test
    void createUser_validatesInput() throws Exception {
        var invalid = new CreateUserRequest("", "not-an-email", 15);
        
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(invalid)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors.name").exists())
            .andExpect(jsonPath("$.errors.email").exists())
            .andExpect(jsonPath("$.errors.age").exists());
        
        verify(userService, never()).create(any());
    }
    
    @Test
    void getById_notFound_returns404() throws Exception {
        when(userService.findById(99L))
            .thenThrow(new ResourceNotFoundException("User", 99L));
        
        mockMvc.perform(get("/api/v1/users/99"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.code").value("RESOURCE_NOT_FOUND"));
    }
}
```

### JPA Slice Test

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired private UserRepository userRepository;
    @Autowired private TestEntityManager em;
    
    @Test
    void findByEmailIgnoreCase_returnsUser() {
        User user = new User("Alice", "Alice@Example.COM", 30);
        em.persistAndFlush(user);
        
        Optional<User> found = userRepository.findByEmailIgnoreCase("alice@example.com");
        
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Alice");
    }
}
```

### Integration Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    
    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired private TestRestTemplate restTemplate;
    
    @Test
    void createAndFetchUser() {
        var req = new CreateUserRequest("Alice", "alice@example.com", 30);
        
        ResponseEntity<UserResponse> created = restTemplate.postForEntity(
            "/api/v1/users", req, UserResponse.class);
        
        assertThat(created.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(created.getHeaders().getLocation()).isNotNull();
        
        Long id = created.getBody().id();
        ResponseEntity<UserResponse> fetched = restTemplate.getForEntity(
            "/api/v1/users/" + id, UserResponse.class);
        
        assertThat(fetched.getBody().name()).isEqualTo("Alice");
    }
}
```

### Testcontainers

```java
@Testcontainers
@SpringBootTest
class FullIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withReuse(true);
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
        r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
        r.add("spring.data.redis.host", redis::getHost);
        r.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }
}
```

### JaCoCo Coverage

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.70</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Testing Best Practices

1. **Follow AAA** — Arrange, Act, Assert
2. **One assertion per test** (or one logical concept)
3. **Descriptive test names** — `methodName_condition_expectedResult`
4. **Don't test frameworks** — test your code
5. **Use Testcontainers** for real DBs (not H2 for prod parity)
6. **Fast feedback** — slice tests first
7. **Deterministic** — no flaky tests
8. **Coverage target** — 70–80% (not 100%)
9. **Test behavior, not implementation**
10. **Contract tests** for microservices

---

## 8. CI/CD with GitHub Actions & Jenkins

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: [5432:5432]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven
      
      - name: Build and test
        run: mvn -B clean verify
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: target/site/jacoco/jacoco.xml
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar
      
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .
      
      - name: Push to registry
        if: github.ref == 'refs/heads/main'
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u "${{ secrets.REGISTRY_USERNAME }}" --password-stdin
          docker push myapp:${{ github.sha }}
      
      - name: Deploy to staging
        if: github.ref == 'refs/heads/main'
        run: |
          kubectl set image deployment/myapp myapp=myapp:${{ github.sha }}
```

### Jenkins Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    tools {
        maven 'Maven-3.9'
        jdk 'JDK-21'
    }
    
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE = "myapp"
        TAG = "${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn -B clean compile'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn -B test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                    jacoco(
                        execPattern: 'target/jacoco.exec',
                        classPattern: 'target/classes',
                        sourcePattern: 'src/main/java'
                    )
                }
            }
        }
        
        stage('SonarQube') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn -B sonar:sonar'
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn -B package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar'
            }
        }
        
        stage('Docker Build & Push') {
            when { branch 'main' }
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'registry-credentials') {
                        def image = docker.build("${IMAGE}:${TAG}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh "kubectl set image deployment/myapp myapp=${REGISTRY}/${IMAGE}:${TAG}"
                sh "kubectl rollout status deployment/myapp --timeout=5m"
            }
        }
    }
    
    post {
        failure {
            slackSend(color: 'danger', message: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
        success {
            slackSend(color: 'good', message: "Build succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
    }
}
```

### CI/CD Best Practices

1. **Fail fast** — fast checks first
2. **Cache dependencies** — Maven/Gradle caches
3. **Parallelize** — independent stages
4. **Artifact versioning** — commit SHA or build number
5. **Quality gates** — SonarQube, coverage thresholds
6. **Secrets in vault** — never in repo
7. **Reproducible builds** — pinned versions
8. **Blue-green/canary** deployments
9. **Rollback plan** — keep previous version
10. **Notifications** — Slack/email on failure

---

## 9. Dockerizing Spring Boot

### Basic Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app
COPY target/app.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### Multi-Stage Build (Smaller Image)

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=build /build/target/*.jar app.jar
USER app
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### Layered JAR (Fast Docker Builds)

**pom.xml:**
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers>
            <enabled>true</enabled>
        </layers>
    </configuration>
</plugin>
```

**Dockerfile:**
```dockerfile
FROM eclipse-temurin:21-jre-alpine AS builder
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

Only the `application` layer changes on code updates — Docker caches the rest.

### Docker Compose

```yaml
version: '3.9'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/appdb
      SPRING_DATASOURCE_USERNAME: app
      SPRING_DATASOURCE_PASSWORD: secret
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 5
  
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
    ports:
      - "9092:9092"
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

### JVM Container Tuning

```dockerfile
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:InitialRAMPercentage=50.0", \
  "-XX:+UseG1GC", \
  "-XX:+ExitOnOutOfMemoryError", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "/app.jar"]
```

### Docker Best Practices

1. **Multi-stage builds** — smaller final image
2. **Non-root user** — security
3. **`.dockerignore`** — exclude `target/`, `.git/`, `node_modules/`
4. **Pin base image versions** — reproducibility
5. **Health checks** — orchestration readiness
6. **Layered JARs** — fast rebuilds
7. **Read-only filesystem** where possible
8. **Don't bake secrets** into images

---

## 10. Kubernetes Deployment

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: registry.example.com/myapp:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: SPRING_DATASOURCE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            failureThreshold: 30
            periodSeconds: 5
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: myapp-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  application.yml: |
    server:
      port: 8080
    spring:
      jpa:
        hibernate:
          ddl-auto: validate
    management:
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  url: jdbc:postgresql://postgres:5432/appdb
  username: app
  password: secret
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### Spring Boot Kubernetes Integration

**Dependency:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
</dependency>
```

**Configuration:**

```properties
# Graceful shutdown
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s

# Liveness/Readiness probes
management.endpoint.health.probes.enabled=true
management.health.livenessState.enabled=true
management.health.readinessState.enabled=true

# Config from ConfigMap/Secret
spring.config.import=kubernetes:
spring.cloud.kubernetes.config.name=myapp-config
spring.cloud.kubernetes.secrets.name=db-secret
```

### Kubernetes Best Practices

1. **Resource requests/limits** — avoid noisy neighbors
2. **Liveness/readiness probes** — correct endpoints
3. **Graceful shutdown** — `server.shutdown=graceful`
4. **Horizontal Pod Autoscaler** — CPU/memory based
5. **PodDisruptionBudget** — for availability
6. **Secrets** — not in ConfigMaps
7. **Network policies** — restrict traffic
8. **Rolling updates** — `maxUnavailable: 0`

---

## 11. Database Migration with Flyway/Liquibase

### Flyway

**Dependency:**

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

**Structure:**

```
src/main/resources/db/migration/
├── V1__create_users_table.sql
├── V2__add_email_index.sql
├── V3__create_orders_table.sql
├── V4__add_orders_status_column.sql
└── V5__insert_default_roles.sql
```

**Migration example:**

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    age INTEGER,
    active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

**Configuration:**

```properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true
spring.flyway.validate-on-migrate=true
spring.flyway.clean-disabled=true
```

**Java-based migration:**

```java
@Component
public class V6__AddUserRoles implements JavaMigration {
    @Override
    public void migrate(Context context) throws Exception {
        try (Statement stmt = context.getConnection().createStatement()) {
            stmt.execute("INSERT INTO roles (name) VALUES ('USER'), ('ADMIN')");
        }
    }
    @Override
    public MigrationVersion getVersion() {
        return MigrationVersion.fromVersion("6");
    }
}
```

### Liquibase

**Dependency:**

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

**Master changelog:**

```xml
<!-- db/changelog/db.changelog-master.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="...">
    <include file="db/changelog/changes/001-create-users.xml"/>
    <include file="db/changelog/changes/002-create-orders.xml"/>
</databaseChangeLog>
```

**Changeset:**

```xml
<changeSet id="001-create-users" author="alice">
    <createTable tableName="users">
        <column name="id" type="bigint" autoIncrement="true">
            <constraints primaryKey="true" nullable="false"/>
        </column>
        <column name="name" type="varchar(100)">
            <constraints nullable="false"/>
        </column>
        <column name="email" type="varchar(255)">
            <constraints nullable="false" unique="true"/>
        </column>
        <column name="created_at" type="timestamp with time zone" defaultValueComputed="NOW()"/>
    </createTable>
    <createIndex tableName="users" indexName="idx_users_email">
        <column name="email"/>
    </createIndex>
    <rollback>
        <dropTable tableName="users"/>
    </rollback>
</changeSet>
```

**YAML format:**

```yaml
databaseChangeLog:
  - changeSet:
      id: 001-create-users
      author: alice
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: bigint
                  autoIncrement: true
                  constraints:
                    primaryKey: true
  - changeSet:
      id: 002-add-roles
      author: alice
      changes:
        - addColumn:
            tableName: users
            columns:
              - column:
                  name: role
                  type: varchar(20)
                  defaultValue: USER
```

### Flyway vs Liquibase

| Feature | Flyway | Liquibase |
|---------|--------|-----------|
| Format | SQL (mostly) | XML, YAML, JSON, SQL |
| Learning curve | Low | Medium |
| Rollback | Manual (undo scripts) | Built-in |
| DB-agnostic | Less | More |
| Conditional logic | Limited | Rich |
| Popularity | ✅ High | ✅ High |

**Recommendation:** **Flyway** for SQL-first teams; **Liquibase** for multi-DB or complex logic.

### Migration Best Practices

1. **Never modify applied migrations** — add new ones
2. **Test migrations** on production-like data
3. **Backward-compatible** — deploy app + migration separately
4. **Expand-contract** pattern for column renames
5. **Version control** all migration scripts
6. **CI check** — validate migrations in pipeline
7. **Baseline existing DBs** — `baseline-on-migrate`
8. **Never `clean` in production**

### Expand-Contract Pattern

```
Phase 1 (Expand):
  - Add new column (nullable)
  - Deploy app writing to both old & new
  - Migrate data

Phase 2 (Migrate):
  - Backfill data
  - Update reads to use new column

Phase 3 (Contract):
  - Stop writing old column
  - Drop old column
```

---

## 12. Monitoring & Observability

### The Three Pillars

| Pillar | Tool | Purpose |
|--------|------|---------|
| **Metrics** | Micrometer + Prometheus + Grafana | Numeric time-series |
| **Logs** | Logback + ELK/Loki | Event records |
| **Traces** | Micrometer Tracing + Zipkin/Jaeger | Request flow |

### Spring Boot Actuator + Micrometer

**Dependencies:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

**Configuration:**

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus,loggers,env
management.endpoint.health.show-details=when-authorized
management.metrics.export.prometheus.enabled=true
management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://zipkin:9411/api/v2/spans
```

### Custom Metrics

```java
@Service
public class OrderService {
    
    private final Counter ordersCreated;
    private final Timer orderProcessingTimer;
    private final AtomicInteger activeOrders;
    private final DistributionSummary orderValue;
    
    public OrderService(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("orders.created")
            .description("Number of orders created")
            .tag("service", "order")
            .register(registry);
        
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Order processing duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.activeOrders = registry.gauge("orders.active", new AtomicInteger(0));
        
        this.orderValue = DistributionSummary.builder("orders.value")
            .baseUnit("USD")
            .register(registry);
    }
    
    public Order create(CreateOrderRequest req) {
        return orderProcessingTimer.record(() -> {
            ordersCreated.increment();
            activeOrders.incrementAndGet();
            try {
                Order order = doCreate(req);
                orderValue.record(order.getTotal().doubleValue());
                return order;
            } finally {
                activeOrders.decrementAndGet();
            }
        });
    }
}
```

### Health Indicators

```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {
    
    private final PaymentClient client;
    
    public PaymentGatewayHealthIndicator(PaymentClient client) {
        this.client = client;
    }
    
    @Override
    public Health health() {
        try {
            var status = client.ping();
            return Health.up()
                .withDetail("status", status)
                .withDetail("latency", status.getLatency())
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-boot'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['myapp:8080']
```

### Grafana Dashboard Queries

```promql
# Request rate
rate(http_server_requests_seconds_count[5m])

# Error rate
rate(http_server_requests_seconds_count{status=~"5.."}[5m])

# P95 latency
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))

# JVM memory
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}

# DB pool
hikaricp_connections_active / hikaricp_connections_max
```

### Distributed Tracing

```java
@RestController
public class OrderController {
    
    private static final Logger log = LoggerFactory.getLogger(OrderController.class);
    
    @GetMapping("/orders/{id}")
    public Order get(@PathVariable Long id) {
        // traceId & spanId automatically in MDC
        log.info("Fetching order {}", id);
        return orderService.findById(id);
    }
}
```

**Log output:**
```
[orders-service,trace-id-abc123,span-id-def456] Fetching order 1
```

### Alerting Rules (Prometheus)

```yaml
groups:
  - name: spring-boot-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.05
        for: 5m
        annotations:
          summary: "High 5xx error rate"
      
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 1
        for: 5m
        annotations:
          summary: "P95 latency > 1s"
      
      - alert: JvmMemoryHigh
        expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.9
        for: 10m
        annotations:
          summary: "Heap usage > 90%"
```

### Observability Best Practices

1. **RED metrics** — Rate, Errors, Duration per endpoint
2. **USE metrics** — Utilization, Saturation, Errors for resources
3. **Structured logs** — JSON for machine parsing
4. **Correlation IDs** — trace across services
5. **Sample traces** — 1–10% in prod
6. **Alert on symptoms** — not causes
7. **Dashboards per service** — golden signals
8. **SLOs** — define and monitor

---

## 13. Security Hardening for Production

### Checklist

- [ ] HTTPS enforced (HSTS)
- [ ] Strong authentication (OAuth2/JWT)
- [ ] Authorization on every endpoint
- [ ] Input validation
- [ ] Output encoding
- [ ] Parameterized queries
- [ ] Rate limiting
- [ ] Request size limits
- [ ] CORS restricted
- [ ] Security headers
- [ ] Secrets in vault
- [ ] Dependency scanning
- [ ] Container scanning
- [ ] Audit logging
- [ ] Fail secure (deny by default)

### Security Headers

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; script-src 'self'; object-src 'none'"))
            .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000)
                .preload(true))
            .referrerPolicy(rp -> rp
                .policy(ReferrerPolicy.NO_REFERRER))
            .permissionsPolicy(pp -> pp
                .policy("geolocation=(), camera=(), microphone=()"))
            .crossOriginOpenerPolicy(coop -> coop
                .policy(CrossOriginOpenerPolicyHeaderWriter.CrossOriginOpenerPolicy.SAME_ORIGIN))
            .crossOriginEmbedderPolicy(coep -> coep
                .policy(CrossOriginEmbedderPolicyHeaderWriter.CrossOriginEmbedderPolicy.REQUIRE_CORP))
            .crossOriginResourcePolicy(corp -> corp
                .policy(CrossOriginResourcePolicyHeaderWriter.CrossOriginResourcePolicy.SAME_ORIGIN)));
    return http.build();
}
```

### Secrets Management

**Spring Cloud Vault:**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```properties
spring.cloud.vault.host=vault.example.com
spring.cloud.vault.port=8200
spring.cloud.vault.scheme=https
spring.cloud.vault.authentication=KUBERNETES
spring.cloud.vault.kv.backend=secret
spring.cloud.vault.kv.default-context=myapp
spring.config.import=vault://
```

**Kubernetes Secrets:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  db-password: ${DB_PASSWORD}
  jwt-secret: ${JWT_SECRET}
```

**AWS Secrets Manager:**

```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-secrets-manager</artifactId>
</dependency>
```

```properties
spring.config.import=aws-secretsmanager:myapp/prod
```

### CORS Configuration

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Correlation-Id"));
    config.setExposedHeaders(List.of("X-Total-Count", "Location"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

### Rate Limiting

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {
    
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    
    private Bucket newBucket() {
        return Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
            .build();
    }
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {
        String key = resolveKey(request);
        Bucket bucket = buckets.computeIfAbsent(key, k -> newBucket());
        
        if (bucket.tryConsume(1)) {
            chain.doFilter(request, response);
            return;
        }
        
        response.setStatus(429);
        response.setHeader("Retry-After", "60");
        response.setHeader("X-RateLimit-Limit", "100");
        response.setContentType("application/json");
        response.getWriter().write("{\"error\":\"Too many requests\"}");
    }
    
    private String resolveKey(HttpServletRequest req) {
        String apiKey = req.getHeader("X-API-Key");
        return apiKey != null ? apiKey : req.getRemoteAddr();
    }
}
```

### Dependency Scanning

**OWASP Dependency Check:**

```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>10.0.4</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <suppressionFile>dependency-check-suppressions.xml</suppressionFile>
    </configuration>
</plugin>
```

**Snyk:**

```yaml
# .github/workflows/security.yml
- name: Run Snyk
  uses: snyk/actions/maven@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

### Production Configuration

```properties
# Disable dev features
spring.devtools.restart.enabled=false
spring.h2.console.enabled=false
spring.jpa.show-sql=false

# Disable stack traces in responses
server.error.include-stacktrace=never
server.error.include-message=never
server.error.include-binding-errors=never

# Actuator security
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=when-authorized

# Session
server.servlet.session.cookie.secure=true
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.same-site=strict
server.servlet.session.timeout=15m

# TLS
server.ssl.enabled=true
server.ssl.key-store=classpath:keystore.p12
server.ssl.key-store-password=${KEYSTORE_PASSWORD}
server.ssl.key-store-type=PKCS12
```

---

## 14. Performance Tuning

### JVM Tuning

```bash
java \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+ParallelRefProcEnabled \
  -XX:+UseStringDeduplication \
  -XX:+ExitOnOutOfMemoryError \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/dumps \
  -Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=10M \
  -jar app.jar
```

### ZGC (Low Latency)

```bash
java -XX:+UseZGC -XX:MaxGCPauseMillis=10 -jar app.jar
```

### Virtual Threads (Java 21+)

```properties
spring.threads.virtual.enabled=true
```

### HikariCP Tuning

```properties
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.leak-detection-threshold=60000
spring.datasource.hikari.pool-name=MyAppPool
```

**Sizing formula:** `pool_size = (core_count * 2) + effective_spindle_count`

### Tomcat Tuning

```properties
server.tomcat.threads.max=200
server.tomcat.threads.min-spare=20
server.tomcat.accept-count=100
server.tomcat.max-connections=8192
server.tomcat.connection-timeout=20s
server.tomcat.keep-alive-timeout=60s
server.tomcat.max-keep-alive-requests=100
```

### JPA/Hibernate Tuning

```properties
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.batch_versioned_data=true
spring.jpa.properties.hibernate.jdbc.fetch_size=100
spring.jpa.properties.hibernate.default_batch_fetch_size=20
spring.jpa.properties.hibernate.query.in_clause_parameter_padding=true
spring.jpa.open-in-view=false
```

### Caching

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .recordStats());
        return manager;
    }
}
```

**Redis cache:**

```properties
spring.cache.type=redis
spring.cache.redis.time-to-live=600000
spring.cache.redis.cache-null-values=false
spring.data.redis.host=redis
spring.data.redis.port=6379
```

### HTTP Compression

```properties
server.compression.enabled=true
server.compression.mime-types=application/json,application/xml,text/html,text/plain
server.compression.min-response-size=1024
```

### Async Processing

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(10);
        exec.setMaxPoolSize(50);
        exec.setQueueCapacity(200);
        exec.setThreadNamePrefix("async-");
        exec.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        exec.initialize();
        return exec;
    }
}
```

### Performance Checklist

- [ ] JVM heap sized correctly (container-aware)
- [ ] G1GC or ZGC configured
- [ ] HikariCP pool sized
- [ ] Tomcat threads tuned
- [ ] Hibernate batch + fetch size
- [ ] `open-in-view=false`
- [ ] Caching for hot data
- [ ] HTTP compression
- [ ] DB indexes on filter/join columns
- [ ] N+1 queries eliminated
- [ ] Pagination on list endpoints
- [ ] Async for I/O-bound work
- [ ] Profiling under load (JFR, async-profiler)

### Profiling Tools

| Tool | Purpose |
|------|---------|
| **JFR** | Built-in, low overhead |
| **async-profiler** | Sampling, flamegraphs |
| **VisualVM** | General profiling |
| **YourKit** | Commercial, feature-rich |
| **JProfiler** | Commercial |
| **Arthas** | Production diagnostics |

---

## 15. Complete Project Example

### Project Structure

```
myapp/
├── pom.xml
├── Dockerfile
├── docker-compose.yml
├── .github/workflows/ci.yml
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── src/
    ├── main/
    │   ├── java/com/example/myapp/
    │   │   ├── MyAppApplication.java
    │   │   ├── config/
    │   │   │   ├── SecurityConfig.java
    │   │   │   ├── OpenApiConfig.java
    │   │   │   ├── CacheConfig.java
    │   │   │   └── AsyncConfig.java
    │   │   ├── common/
    │   │   │   ├── exception/
    │   │   │   │   ├── ApplicationException.java
    │   │   │   │   ├── ResourceNotFoundException.java
    │   │   │   │   ├── DuplicateResourceException.java
    │   │   │   │   └── GlobalExceptionHandler.java
    │   │   │   ├── audit/
    │   │   │   │   └── AuditableEntity.java
    │   │   │   └── web/
    │   │   │       ├── CorrelationIdFilter.java
    │   │   │       └── RateLimitFilter.java
    │   │   ├── user/
    │   │   │   ├── UserController.java
    │   │   │   ├── UserService.java
    │   │   │   ├── UserRepository.java
    │   │   │   ├── User.java
    │   │   │   └── dto/
    │   │   │       ├── CreateUserRequest.java
    │   │   │       ├── UpdateUserRequest.java
    │   │   │       ├── UserResponse.java
    │   │   │       └── UserMapper.java
    │   │   └── order/
    │   │       ├── OrderController.java
    │   │       ├── OrderService.java
    │   │       ├── OrderRepository.java
    │   │       ├── Order.java
    │   │       └── dto/
    │   └── resources/
    │       ├── application.yml
    │       ├── application-dev.yml
    │       ├── application-prod.yml
    │       ├── logback-spring.xml
    │       └── db/migration/
    │           └── V1__init.sql
    └── test/
        └── java/com/example/myapp/
            ├── user/
            │   ├── UserControllerTest.java
            │   ├── UserServiceTest.java
            │   └── UserRepositoryTest.java
            └── integration/
                └── UserIntegrationTest.java
```

### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>myapp</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <name>myapp</name>
    <description>Spring Boot application</description>
    
    <properties>
        <java.version>21</java.version>
        <mapstruct.version>1.6.3</mapstruct.version>
        <springdoc.version>2.6.0</springdoc.version>
        <bucket4j.version>8.10.1</bucket4j.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        
        <!-- Database -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-database-postgresql</artifactId>
        </dependency>
        
        <!-- Caching -->
        <dependency>
            <groupId>com.github.ben-manes.caffeine</groupId>
            <artifactId>caffeine</artifactId>
        </dependency>
        
        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.12.6</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.12.6</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.12.6</version>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Mapping -->
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${mapstruct.version}</version>
        </dependency>
        
        <!-- OpenAPI -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>${springdoc.version}</version>
        </dependency>
        
        <!-- Rate Limiting -->
        <dependency>
            <groupId>com.bucket4j</groupId>
            <artifactId>bucket4j-core</artifactId>
            <version>${bucket4j.version}</version>
        </dependency>
        
        <!-- Metrics -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
        
        <!-- JSON Logging -->
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>8.0</version>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <layers><enabled>true</enabled></lays>
                </configuration>
            </plugin>
            
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${mapstruct.version}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
            
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.12</version>
                <executions>
                    <execution>
                        <goals><goal>prepare-agent</goal></goals>
                    </execution>
                    <execution>
                        <id>report</id>
                        <phase>test</phase>
                        <goals><goal>report</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### application.yml

```yaml
spring:
  application:
    name: myapp
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
        default_batch_fetch_size: 20
  flyway:
    enabled: true
    baseline-on-migrate: true
    clean-disabled: true
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      max-lifetime: 1800000
  threads:
    virtual:
      enabled: true

server:
  port: 8080
  shutdown: graceful
  compression:
    enabled: true
  error:
    include-stacktrace: never
    include-message: never

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  metrics:
    tags:
      application: ${spring.application.name}

logging:
  level:
    root: INFO
    com.example.myapp: DEBUG
    org.hibernate.SQL: DEBUG

springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
```

### UserController.java

```java
@RestController
@RequestMapping("/api/v1/users")
@Tag(name = "Users", description = "User management")
@RequiredArgsConstructor
public class UserController {
    
    private final UserService userService;
    
    @GetMapping
    @Operation(summary = "List users")
    public Page<UserResponse> list(
            @PageableDefault(size = 20, sort = "id") Pageable pageable) {
        return userService.findAll(pageable);
    }
    
    @GetMapping("/{id}")
    @Operation(summary = "Get user by ID")
    public UserResponse get(@PathVariable @Positive Long id) {
        return userService.findById(id);
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(summary = "Create user")
    public ResponseEntity<UserResponse> create(
            @Valid @RequestBody CreateUserRequest req,
            UriComponentsBuilder uri) {
        UserResponse user = userService.create(req);
        return ResponseEntity
            .created(uri.path("/api/v1/users/{id}").buildAndExpand(user.id()).toUri())
            .body(user);
    }
    
    @PutMapping("/{id}")
    @Operation(summary = "Update user")
    public UserResponse update(@PathVariable Long id,
                               @Valid @RequestBody UpdateUserRequest req) {
        return userService.update(id, req);
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Delete user")
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### UserService.java

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {
    
    private final UserRepository userRepository;
    private final UserMapper userMapper;
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional(readOnly = true)
    public Page<UserResponse> findAll(Pageable pageable) {
        return userRepository.findAll(pageable).map(userMapper::toResponse);
    }
    
    @Transactional(readOnly = true)
    public UserResponse findById(Long id) {
        return userRepository.findById(id)
            .map(userMapper::toResponse)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }
    
    @Transactional
    public UserResponse create(CreateUserRequest req) {
        if (userRepository.existsByEmailIgnoreCase(req.email())) {
            throw new DuplicateResourceException("User", "email", req.email());
        }
        
        User user = userMapper.toEntity(req);
        User saved = userRepository.save(user);
        
        log.info("User created: id={}, email={}", saved.getId(), saved.getEmail());
        eventPublisher.publishEvent(new UserCreatedEvent(saved.getId(), saved.getEmail()));
        
        return userMapper.toResponse(saved);
    }
    
    @Transactional
    public UserResponse update(Long id, UpdateUserRequest req) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
        
        userMapper.updateEntity(req, user);
        return userMapper.toResponse(user);
    }
    
    @Transactional
    public void delete(Long id) {
        if (!userRepository.existsById(id)) {
            throw new ResourceNotFoundException("User", id);
        }
        userRepository.deleteById(id);
        log.info("User deleted: id={}", id);
    }
}
```

### User.java

```java
@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_users_email", columnList = "email")
})
@EntityListeners(AuditingEntityListener.class)
@Getter
@Setter
@NoArgsConstructor
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 100)
    private String name;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    private Integer age;
    
    @Column(nullable = false)
    private boolean active = true;
    
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private Instant updatedAt;
    
    @Version
    private Long version;
    
    public User(String name, String email, Integer age) {
        this.name = name;
        this.email = email;
        this.age = age;
    }
}
```

### UserMapper.java

```java
@Mapper(componentModel = "spring",
        unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface UserMapper {
    
    UserResponse toResponse(User user);
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "active", constant = "true")
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "version", ignore = true)
    User toEntity(CreateUserRequest req);
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "email", ignore = true)
    @Mapping(target = "active", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "version", ignore = true)
    void updateEntity(UpdateUserRequest req, @MappingTarget User user);
}
```

---

## 16. Interview Questions

### Q1. Why layered architecture?

**Answer:** Separates concerns: presentation (HTTP), service (business), repository (data), domain (entities). Improves testability, maintainability, and allows swapping implementations. Each layer depends only on the one below (or via interfaces in hexagonal).

### Q2. Feature-based vs layer-based packages?

**Answer:** Feature-based groups related files by domain (user/, order/), improving cohesion and modularity. Layer-based groups by technical layer (controller/, service/). Feature-based is preferred for medium/large apps and eases microservices extraction.

### Q3. Why use DTOs?

**Answer:** Decouple API contract from DB schema, prevent mass assignment, control serialization (hide sensitive fields), shape data for clients, and version APIs independently. Never expose entities in controllers.

### Q4. MapStruct vs ModelMapper?

**Answer:** MapStruct generates mapping code at compile time (fast, type-safe, fails on mismatch). ModelMapper uses runtime reflection (slower, silent failures). MapStruct is preferred for production.

### Q5. How do you handle exceptions globally?

**Answer:** `@RestControllerAdvice` with `@ExceptionHandler` methods mapping exceptions to `ProblemDetail` (RFC 7807). Custom exception hierarchy: `ApplicationException` base with code/status. Log 5xx at ERROR, 4xx at WARN. Never leak stack traces.

### Q6. What is RFC 7807?

**Answer:** "Problem Details for HTTP APIs" — standard error format with `type`, `title`, `status`, `detail`, `instance`. Spring 6+ has native `ProblemDetail` support.

### Q7. Structured logging vs plain?

**Answer:** Structured (JSON) logs are machine-parseable, enabling indexing/searching in ELK/Loki. Plain text is human-readable but harder to query at scale. Production should use JSON with correlation IDs.

### Q8. What is MDC?

**Answer:** Mapped Diagnostic Context — thread-local key-value store for log context (correlation ID, user ID, trace ID). Populated by filters, cleared after request. Included in log patterns automatically.

### Q9. How do you validate input?

**Answer:** Jakarta Bean Validation with `@Valid` on `@RequestBody` DTOs. Annotations: `@NotBlank`, `@Email`, `@Size`, `@Min`, `@Pattern`. Custom constraints via `ConstraintValidator`. Cross-field via class-level constraints. Groups for create/update.

### Q10. Testing strategy?

**Answer:** Test pyramid: many unit tests (fast), some slice tests (`@WebMvcTest`, `@DataJpaTest`), few integration tests (`@SpringBootTest` + Testcontainers), contract tests for microservices. Target 70–80% coverage, not 100%.

### Q11. Why Testcontainers?

**Answer:** Real databases/brokers (PostgreSQL, Kafka) in Docker for tests — production parity. H2 often behaves differently (SQL dialect, constraints). Testcontainers ensures tests match production behavior.

### Q12. How do you structure a CI/CD pipeline?

**Answer:** Stages: checkout → build → unit tests → integration tests → quality gates (SonarQube) → package → Docker build → push → deploy (staging → prod). Fail fast, cache dependencies, version artifacts, rollback plan.

### Q13. Why multi-stage Docker builds?

**Answer:** Build stage uses full JDK + Maven; runtime stage uses slim JRE only. Result: smaller image, smaller attack surface, faster pulls. Layered JARs further optimize rebuild caching.

### Q14. What is a layered JAR?

**Answer:** Spring Boot 2.3+ feature — JAR split into `dependencies`, `spring-boot-loader`, `snapshot-dependencies`, `application`. Only `application` changes on code updates, so Docker caches the rest → fast rebuilds.

### Q15. Kubernetes probes?

**Answer:** **Liveness** — is the app alive? Restart if failed. **Readiness** — can it serve traffic? Remove from service if failed. **Startup** — has it started? Delays liveness checks. Spring Boot Actuator exposes `/actuator/health/liveness` and `/readiness`.

### Q16. Flyway vs Liquibase?

**Answer:** Flyway: SQL-first, simple versioned migrations, popular with Java devs. Liquibase: DB-agnostic, XML/YAML/JSON, rollback support, better for multi-DB. Flyway for SQL-centric teams; Liquibase for complex logic.

### Q17. What is expand-contract migration?

**Answer:** Safe schema change pattern: (1) **Expand** — add new column/table, deploy app writing to both. (2) **Migrate** — backfill data. (3) **Contract** — stop writing old, drop it. Avoids downtime and rollback issues.

### Q18. Observability pillars?

**Answer:** Metrics (numeric, Prometheus), logs (events, ELK/Loki), traces (request flow, Zipkin/Jaeger). All correlated by trace/correlation IDs. Micrometer unifies metrics + tracing for Spring Boot.

### Q19. Custom metrics?

**Answer:** Inject `MeterRegistry`; create `Counter`, `Timer`, `Gauge`, `DistributionSummary` with tags. Use RED (Rate, Errors, Duration) per endpoint, USE (Utilization, Saturation, Errors) per resource.

### Q20. JVM tuning for containers?

**Answer:** `-XX:+UseContainerSupport` (default in JDK 11+), `-XX:MaxRAMPercentage=75.0` (heap % of container limit), `-XX:+UseG1GC` or ZGC, `-XX:+ExitOnOutOfMemoryError`, GC logging. Spring Boot 3.2+ supports virtual threads.

### Q21. HikariCP pool sizing?

**Answer:** Formula: `connections = (core_count * 2) + effective_spindle_count`. Typically 10–20 for web apps. Set `max-lifetime` < DB wait_timeout. Enable `leak-detection-threshold` to catch leaks.

### Q22. Why `spring.jpa.open-in-view=false`?

**Answer:** Prevents lazy loading in views/controllers (outside transaction), which causes N+1 and connection holding. Forces explicit fetching in service layer. Spring Boot warns if enabled.

### Q23. How do you secure secrets?

**Answer:** Never commit to Git. Use HashiCorp Vault, AWS Secrets Manager, Kubernetes Secrets, or Spring Cloud Vault. Inject via environment variables. Rotate regularly. Audit access.

### Q24. Security headers to set?

**Answer:** HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, COOP/COEP/CORP. Spring Security sets sensible defaults; customize as needed.

### Q25. How do you prevent N+1?

**Answer:** Enable SQL logging to detect. Fix with `JOIN FETCH`, `@EntityGraph`, `@BatchSize`, projections, or `default_batch_fetch_size`. Use DTOs when only some fields are needed.

### Q26. Rate limiting implementation?

**Answer:** Bucket4j (token bucket) with in-memory or Redis backend. Filter or `HandlerInterceptor`. Return 429 with `Retry-After` and rate limit headers. Different limits per tier/API key.

### Q27. What is correlation ID?

**Answer:** Unique ID per request, generated at entry, propagated via header (`X-Correlation-Id`) and MDC. Enables tracing across services and correlating logs. Essential for debugging distributed systems.

### Q28. Blue-green vs canary deployment?

**Answer:** **Blue-green** — two identical environments, switch traffic at once (fast rollback). **Canary** — gradually shift traffic to new version (safer, slower). Both enable zero-downtime deploys.

### Q29. How do you handle DB schema changes without downtime?

**Answer:** Expand-contract pattern: add new column/table → deploy app writing both → backfill → migrate reads → remove old. Never drop columns in the same deploy. Test on production-like data.

### Q30. Performance tuning checklist?

**Answer:** JVM (heap, GC), HikariCP pool, Tomcat threads, Hibernate batch/fetch, `open-in-view=false`, caching, compression, DB indexes, N+1 elimination, pagination, async for I/O. Profile with JFR/async-profiler before optimizing.

---

## Cross-References

- **Previous file:** `24_Spring_Interview_Questions.md`
- **Next file:** `26_Spring_Best_Practices.md`
- **Related:** `01`–`16` for individual topics, `17`–`18` for build tools

---

## Summary Checklist

Use this as a template for any new Spring Boot project:

- [ ] Feature-based package structure
- [ ] DTOs + MapStruct for mapping
- [ ] Custom exception hierarchy + global handler
- [ ] Structured logging with correlation IDs
- [ ] Bean Validation on DTOs
- [ ] Test pyramid (unit, slice, integration)
- [ ] CI/CD pipeline with quality gates
- [ ] Multi-stage Docker build + layered JAR
- [ ] Kubernetes deployment with probes
- [ ] Flyway/Liquibase migrations
- [ ] Actuator + Micrometer + Prometheus
- [ ] Security headers, CORS, rate limiting
- [ ] Secrets in Vault
- [ ] JVM/HikariCP/Hibernate tuning
- [ ] Caching for hot data
- [ ] OpenAPI documentation

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [24_Spring_Interview_Questions.md](./24_Spring_Interview_Questions.md)
- **Next →:** [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md)
- **Related:** [24_Spring_Interview_Questions.md](./24_Spring_Interview_Questions.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md), [01_Spring_Framework_Core.md](./01_Spring_Framework_Core.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
