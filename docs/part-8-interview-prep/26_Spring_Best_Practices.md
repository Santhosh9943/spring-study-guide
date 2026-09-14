# Spring Best Practices & Anti-Patterns

> **File:** `26_Spring_Best_Practices.md`
> **Part:** 8 — Practice & Interview Prep
> **Prerequisites:** All previous files
> **Estimated Study Time:** 10–14 hours (read + apply to code review)

---

## Table of Contents

1. [Configuration Best Practices](#1-configuration-best-practices)
2. [Dependency Injection Best Practices](#2-dependency-injection-best-practices)
3. [Bean Scoping Best Practices](#3-bean-scoping-best-practices)
4. [Transaction Management Best Practices](#4-transaction-management-best-practices)
5. [Exception Handling Best Practices](#5-exception-handling-best-practices)
6. [Security Best Practices](#6-security-best-practices)
7. [JPA/Hibernate Best Practices](#7-jpahibernate-best-practices)
8. [REST API Best Practices](#8-rest-api-best-practices)
9. [Testing Best Practices](#9-testing-best-practices)
10. [Performance Anti-Patterns](#10-performance-anti-patterns)
11. [Logging Best Practices](#11-logging-best-practices)
12. [Common Mistakes & How to Avoid Them](#12-common-mistakes--how-to-avoid-them)
13. [Code Review Checklist](#13-code-review-checklist)
14. [Interview Questions](#14-interview-questions)

---

## 1. Configuration Best Practices

### ✅ DO: Use Java Config Over XML

```java
@Configuration
public class AppConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost:5432/db");
        return ds;
    }
}
```

❌ **AVOID:** XML config in new projects

```xml
<bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource">
    <property name="jdbcUrl" value="jdbc:postgresql://localhost:5432/db"/>
</bean>
```

### ✅ DO: Use @ConfigurationProperties for Grouped Config

```java
@ConfigurationProperties(prefix = "app.mail")
@Validated
public record MailProperties(
    @NotBlank String host,
    @Min(1) @Max(65535) int port,
    @NotBlank String username,
    @NotBlank String password,
    @Email String from,
    boolean tlsEnabled,
    Duration timeout,
    Retry retry
) {
    public record Retry(int maxAttempts, Duration backoff) { }
}
```

**Enable it:**

```java
@SpringBootApplication
@ConfigurationPropertiesScan   // auto-scan @ConfigurationProperties
public class App { }
```

❌ **AVOID: Scattered @Value annotations**

```java
@Value("${app.mail.host}") private String host;
@Value("${app.mail.port}") private int port;
@Value("${app.mail.username}") private String username;
@Value("${app.mail.password}") private String password;
// ... 20 more fields
```

**Why:** No type safety, no grouping, no validation, no IDE completion.

### ✅ DO: Externalize All Environment-Specific Config

```yaml
# application.yml (defaults)
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/dev
    username: dev
    password: dev

---
# application-prod.yml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

❌ **AVOID: Hardcoded values**

```java
String url = "jdbc:postgresql://prod-db-1.internal:5432/app";  // ❌
```

### ✅ DO: Use Environment Variables for Secrets

```properties
spring.datasource.password=${DB_PASSWORD}
spring.security.oauth2.client.registration.google.client-secret=${GOOGLE_SECRET}
```

❌ **AVOID: Secrets in application.properties**

```properties
spring.datasource.password=SuperSecret123  # ❌ committed to Git
```

### ✅ DO: Set `proxyBeanMethods = false` When Possible

```java
@Configuration(proxyBeanMethods = false)
public class MapperConfig {
    // No inter-bean method calls → no CGLIB needed → faster startup
    @Bean
    public UserMapper userMapper() {
        return Mappers.getMapper(UserMapper.class);
    }
}
```

### ✅ DO: Validate Configuration at Startup

```java
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {
    @NotBlank private String name;
    @Min(1) @Max(100) private int maxConnections;
    @DurationMin(seconds = 1) private Duration timeout;
}
```

Fails fast if misconfigured — no runtime surprises.

### ✅ DO: Use Profile Groups (Boot 2.4+)

```properties
spring.profiles.group.production=prod-db,prod-cache,prod-monitoring
```

Then `spring.profiles.active=production` activates all three.

### ✅ DO: Import Config in Modular Way

```java
@Configuration
@Import({DataSourceConfig.class, SecurityConfig.class, CacheConfig.class})
public class AppConfig { }
```

Or with Spring Boot 2.4+:

```properties
spring.config.import=classpath:db.yml,classpath:security.yml
```

### Configuration Anti-Patterns

| Anti-Pattern | Better |
|--------------|--------|
| XML config in new projects | Java config |
| Scattered `@Value` | `@ConfigurationProperties` |
| Secrets in properties files | Vault/env vars |
| Hardcoded URLs/ports | Externalized config |
| No validation | `@Validated` |
| `@Configuration` with `proxyBeanMethods=true` (when no inter-bean calls) | `proxyBeanMethods=false` |
| Multiple profiles mixed | Profile groups |

---

## 2. Dependency Injection Best Practices

### ✅ DO: Constructor Injection (Always for Required Deps)

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final EmailService emailService;
    
    public OrderService(OrderRepository orderRepository,
                        PaymentService paymentService,
                        EmailService emailService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
        this.emailService = emailService;
    }
}
```

**With Lombok:**

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
}
```

❌ **AVOID: Field Injection**

```java
@Service
public class OrderService {
    @Autowired private OrderRepository orderRepository;  // ❌
    @Autowired private PaymentService paymentService;    // ❌
}
```

**Why it's bad:**
- Can't make fields `final`
- Hidden dependencies
- Hard to unit test without Spring
- Encourages too many dependencies
- NPE risk if not wired

### ✅ DO: Depend on Interfaces (Ports)

```java
public interface PaymentGateway {
    PaymentResult charge(BigDecimal amount, PaymentMethod method);
}

@Service
public class StripePaymentGateway implements PaymentGateway { }

@Service
public class OrderService {
    private final PaymentGateway paymentGateway;  // interface!
    
    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

❌ **AVOID: Depending on concrete implementations**

```java
public OrderService(StripePaymentGateway gateway) { }  // ❌ locked to Stripe
```

### ✅ DO: Use @Qualifier When Multiple Beans Exist

```java
@Service
public class NotificationService {
    private final MessageSender emailSender;
    private final MessageSender smsSender;
    
    public NotificationService(
            @Qualifier("emailSender") MessageSender emailSender,
            @Qualifier("smsSender") MessageSender smsSender) {
        this.emailSender = emailSender;
        this.smsSender = smsSender;
    }
}
```

Or custom qualifier annotations:

```java
@Qualifier
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Email { }

@Component
@Email
public class EmailSender implements MessageSender { }

// Usage
public NotificationService(@Email MessageSender sender) { }
```

### ✅ DO: Use ObjectProvider for Optional/Lazy Deps

```java
@Service
public class OrderService {
    private final ObjectProvider<MetricsCollector> metricsProvider;
    
    public OrderService(ObjectProvider<MetricsCollector> metricsProvider) {
        this.metricsProvider = metricsProvider;
    }
    
    public void process() {
        metricsProvider.ifAvailable(m -> m.record("order"));
    }
}
```

❌ **AVOID: `@Autowired(required=false)`**

```java
@Autowired(required = false)  // ❌ silent null, hard to spot
private MetricsCollector metrics;
```

### ✅ DO: Keep Dependencies Minimal

**Rule of thumb:** ≤ 5 constructor dependencies.

If a class has > 5 dependencies, it's likely doing too much — split it.

❌ **AVOID: God classes**

```java
public OrderService(
    OrderRepository orderRepo,
    UserRepository userRepo,
    ProductRepository productRepo,
    PaymentGateway payment,
    EmailService email,
    SmsService sms,
    InventoryService inventory,
    TaxService tax,
    ShippingService shipping,
    AuditService audit) { }  // ❌ 10 dependencies!
```

### ✅ DO: Use `@Configuration` for Third-Party Beans

```java
@Configuration
public class HttpClientConfig {
    
    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        return builder
            .baseUrl("https://api.example.com")
            .requestFactory(new JdkClientHttpRequestFactory())
            .build();
    }
}
```

❌ **AVOID: Wrapping third-party classes just to annotate them**

### ✅ DO: Split Configurations by Concern

```java
@Configuration
public class SecurityConfig { }

@Configuration
public class DataSourceConfig { }

@Configuration
public class CacheConfig { }

@Configuration
public class WebConfig implements WebMvcConfigurer { }
```

❌ **AVOID: One giant `@Configuration` class**

### DI Anti-Patterns

| Anti-Pattern | Better |
|--------------|--------|
| Field injection | Constructor injection |
| `@Autowired(required=false)` | `ObjectProvider<T>` |
| Concrete class injection | Interface injection |
| God classes (>5 deps) | Split into focused services |
| `new` for Spring beans | Inject |
| Self-injection (for AOP) | `AopContext.currentProxy()` or restructure |
| Circular dependencies | Redesign; extract shared logic |

---

## 3. Bean Scoping Best Practices

### ✅ DO: Keep Singletons Stateless

```java
@Service
public class OrderService {
    private final OrderRepository repo;   // ✅ immutable
    // NO mutable state!
    
    public OrderService(OrderRepository repo) {
        this.repo = repo;
    }
}
```

❌ **AVOID: Mutable state in singletons (thread-safety hazard)**

```java
@Service
public class OrderService {
    private int counter;  // ❌ shared across threads!
    private List<String> cache = new ArrayList<>();  // ❌ not thread-safe
}
```

**Fix:** use `AtomicInteger`, `ConcurrentHashMap`, or move state to method scope.

### ✅ DO: Use Prototype for Stateful Objects

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class OrderBuilder {
    private final List<OrderItem> items = new ArrayList<>();
    
    public OrderBuilder addItem(OrderItem item) {
        items.add(item);
        return this;
    }
}
```

### ✅ DO: Use `ObjectProvider` for Prototypes Inside Singletons

```java
@Service
public class OrderService {
    private final ObjectProvider<OrderBuilder> builderProvider;
    
    public OrderService(ObjectProvider<OrderBuilder> builderProvider) {
        this.builderProvider = builderProvider;
    }
    
    public Order create() {
        OrderBuilder builder = builderProvider.getObject();  // fresh each time
        return builder.build();
    }
}
```

❌ **AVOID: Injecting prototype directly into singleton**

```java
@Service
public class OrderService {
    private final OrderBuilder builder;   // ❌ frozen — one instance forever
    
    public OrderService(OrderBuilder builder) {
        this.builder = builder;
    }
}
```

### ✅ DO: Use `@RequestScope` / `@SessionScope` for Web State

```java
@Component
@RequestScope
public class RequestContext {
    private String correlationId;
    private User currentUser;
}
```

**Note:** Web-scoped beans require proxy mode when injected into singletons:

```java
@Component
@RequestScope
public class RequestContext { }

@Service
public class OrderService {
    // Spring injects a proxy that resolves to the current request's bean
    public OrderService(RequestContext ctx) { }
}
```

### ✅ DO: Use `@ApplicationScope` for App-Wide State

```java
@Component
@ApplicationScope
public class ApplicationMetrics {
    private final AtomicLong totalOrders = new AtomicLong();
}
```

### ✅ DO: Prefer `@Lazy` for Heavy Beans

```java
@Bean
@Lazy
public ExpensiveMLModel mlModel() {
    return new ExpensiveMLModel();   // only loaded when first used
}
```

### ✅ DO: Avoid Custom Scopes Unless Necessary

Custom scopes (`ThreadScope`, `TenantScope`) add complexity. Prefer stateless singletons + method params.

### Scoping Anti-Patterns

| Anti-Pattern | Better |
|--------------|--------|
| Mutable fields in singleton | Stateless singleton |
| Prototype injected into singleton | `ObjectProvider<T>` / `@Lookup` |
| Request-scoped in singleton (no proxy) | `proxyMode = TARGET_CLASS` |
| Overusing prototype | Only for stateful objects |
| Eager heavy beans | `@Lazy` |
| Custom scopes for everything | Prefer singletons |

---

## 4. Transaction Management Best Practices

### ✅ DO: @Transactional on Service Layer, Not Controllers/Repositories

```java
@Service
public class OrderService {
    
    @Transactional
    public Order create(CreateOrderRequest req) {
        // multiple repo calls in one transaction
        Order order = orderRepo.save(...);
        paymentRepo.save(...);
        inventoryRepo.decrement(...);
        return order;
    }
}
```

❌ **AVOID: `@Transactional` on controllers**

```java
@RestController
public class OrderController {
    @Transactional  // ❌ too broad — HTTP layer shouldn't know TX
    @PostMapping
    public Order create(...) { }
}
```

### ✅ DO: Mark Read-Only Transactions

```java
@Transactional(readOnly = true)
public List<OrderResponse> findAll() {
    return orderRepo.findAll().stream().map(this::toResponse).toList();
}
```

**Benefits:** Hibernate skips dirty checking, DB may optimize.

### ✅ DO: Be Explicit About Rollback Rules

```java
@Transactional(rollbackFor = Exception.class)  // rolls back on checked too
public void process() throws IOException { }
```

**Default:** Spring rolls back on `RuntimeException` and `Error`, NOT on checked exceptions.

### ✅ DO: Keep Transactions Short

```java
@Transactional
public void process() {
    Order order = orderRepo.findById(id).orElseThrow();
    order.setStatus(PAID);
    // NO HTTP calls, NO sleep, NO heavy computation here
}
```

❌ **AVOID: Long-running operations inside transactions**

```java
@Transactional
public void process() {
    Order order = repo.findById(id).orElseThrow();
    externalPaymentApi.charge(order);   // ❌ HTTP call inside TX
    Thread.sleep(5000);                  // ❌ holds DB connection
    emailService.send(order);            // ❌ could fail → rollback everything
}
```

**Fix:** Do I/O outside transaction, update DB after:

```java
public void process() {
    Order order = transactionTemplate.execute(tx -> repo.findById(id).orElseThrow());
    PaymentResult result = paymentApi.charge(order);   // outside TX
    transactionTemplate.execute(tx -> {
        order.setStatus(PAID);
        return repo.save(order);
    });
    emailService.send(order);
}
```

### ✅ DO: Understand Propagation

```java
@Transactional(propagation = Propagation.REQUIRED)   // default; join or create
@Transactional(propagation = Propagation.REQUIRES_NEW)  // always new TX
@Transactional(propagation = Propagation.SUPPORTS)   // join if present
@Transactional(propagation = Propagation.NOT_SUPPORTED)  // suspend existing
@Transactional(propagation = Propagation.MANDATORY)  // must be inside
@Transactional(propagation = Propagation.NEVER)      // must NOT be inside
@Transactional(propagation = Propagation.NESTED)     // savepoint
```

### ✅ DO: Call @Transactional Methods from Another Bean

```java
@Service
public class OrderService {
    private final OrderProcessor processor;  // separate bean
    
    public void placeOrder() {
        processor.processInTransaction();  // ✅ proxy applied
    }
}

@Service
public class OrderProcessor {
    @Transactional
    public void processInTransaction() { }
}
```

❌ **AVOID: Self-invocation**

```java
@Service
public class OrderService {
    public void placeOrder() {
        processInTransaction();  // ❌ bypasses proxy — no TX!
    }
    
    @Transactional
    public void processInTransaction() { }
}
```

### ✅ DO: Handle Rollback Explicitly

```java
@Transactional
public void process() {
    try {
        riskyOperation();
    } catch (Exception e) {
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        throw new BusinessException("Processing failed", e);
    }
}
```

### ✅ DO: Avoid Nested Transactions When Possible

`REQUIRES_NEW` suspends the outer TX and uses a new connection from the pool — can deadlock under load. Use only when you truly need independence.

### ✅ DO: Prefer `@Transactional` on Public Methods Only

```java
@Transactional
public void process() { }        // ✅ works

@Transactional
private void process() { }       // ❌ proxy can't intercept

@Transactional
protected void process() { }     // ⚠️ works only with CGLIB proxy
```

### ✅ DO: Use `TransactionTemplate` for Fine Control

```java
@Service
public class ReportService {
    private final TransactionTemplate txTemplate;
    
    public ReportService(PlatformTransactionManager txManager) {
        this.txTemplate = new TransactionTemplate(txManager);
        this.txTemplate.setTimeout(30);
        this.txTemplate.setReadOnly(true);
    }
    
    public Report generate() {
        return txTemplate.execute(status -> buildReport());
    }
}
```

### Transaction Anti-Patterns

| Anti-Pattern | Fix |
|--------------|-----|
| `@Transactional` on controller | Move to service |
| Self-invocation | Extract to separate bean |
| Long transactions with I/O | I/O outside TX |
| Checked exceptions not rolling back | `rollbackFor = Exception.class` |
| Catching exceptions inside TX | Re-throw or `setRollbackOnly()` |
| `@Transactional` on private methods | Make public, separate bean |
| Overusing `REQUIRES_NEW` | Only when truly independent |
| Missing `readOnly=true` on queries | Add for read-only methods |
| Transaction spanning HTTP calls | Split into multiple TX |
| Lazy loading outside TX | Fetch in TX, return DTO |

---

## 5. Exception Handling Best Practices

### ✅ DO: Use a Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ProblemDetail> handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Resource Not Found");
        return ResponseEntity.status(404).body(pd);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleAll(Exception ex) {
        log.error("Unhandled exception", ex);
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected error occurred");
        return ResponseEntity.internalServerError().body(pd);
    }
}
```

### ✅ DO: Create a Custom Exception Hierarchy

```java
public abstract class ApplicationException extends RuntimeException {
    private final String code;
    private final HttpStatus status;
    // ...
}

public class ResourceNotFoundException extends ApplicationException {
    public ResourceNotFoundException(String resource, Object id) {
        super("RESOURCE_NOT_FOUND", resource + " not found: " + id, HttpStatus.NOT_FOUND);
    }
}

public class BusinessRuleException extends ApplicationException {
    public BusinessRuleException(String message) {
        super("BUSINESS_RULE_VIOLATION", message, HttpStatus.UNPROCESSABLE_ENTITY);
    }
}
```

### ✅ DO: Return RFC 7807 ProblemDetail

```java
ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed");
pd.setTitle("Validation Error");
pd.setProperty("errors", fieldErrors);
pd.setProperty("timestamp", Instant.now());
pd.setProperty("path", "/api/v1/users");
return ResponseEntity.badRequest().body(pd);
```

### ✅ DO: Log at the Right Level

| Exception | Log Level | Reason |
|-----------|-----------|--------|
| Client error (4xx) | WARN | Not our fault, but worth noticing |
| Server error (5xx) | ERROR | Requires investigation |
| Expected business exception | INFO/WARN | Part of flow |
| Recovery/retry | INFO | Normal operation |

### ✅ DO: Include Context in Exception Messages

```java
throw new ResourceNotFoundException("User", userId);   // ✅ includes ID
throw new ResourceNotFoundException("User not found"); // ❌ no context
```

### ✅ DO: Never Swallow Exceptions

```java
try {
    process();
} catch (Exception e) {
    // ❌ silence — debugging nightmare
}

try {
    process();
} catch (Exception e) {
    log.warn("Processing failed", e);   // ✅ at least log it
}
```

### ✅ DO: Rethrow as Unchecked at Layer Boundaries

```java
@Repository
public class UserRepository {
    public User findById(Long id) {
        try {
            return jdbcTemplate.queryForObject(...);
        } catch (DataAccessException e) {
            throw new RepositoryException("Failed to load user", e);
        }
    }
}
```

### ✅ DO: Use Specific Exception Types

```java
catch (IOException e) { }       // ✅ specific
catch (Exception e) { }         // ❌ too broad
```

### ✅ DO: Don't Use Exceptions for Control Flow

❌ **AVOID:**

```java
try {
    User user = userRepo.findById(id).orElseThrow();
} catch (NoSuchElementException e) {
    user = createDefaultUser();
}
```

✅ **DO:**

```java
User user = userRepo.findById(id).orElseGet(() -> createDefaultUser());
```

### ✅ DO: Fail Fast on Invalid State

```java
public void process(Order order) {
    Objects.requireNonNull(order, "order must not be null");
    if (order.getStatus() != PENDING) {
        throw new IllegalStateException("Order must be PENDING");
    }
    // ... now safe
}
```

### ✅ DO: Include Correlation ID in Errors

```json
{
  "type": "about:blank",
  "title": "Internal Error",
  "status": 500,
  "detail": "An unexpected error occurred",
  "instance": "/api/v1/orders",
  "correlationId": "abc-123-def-456"
}
```

### Exception Handling Anti-Patterns

| Anti-Pattern | Fix |
|--------------|-----|
| Catch and swallow | Log or rethrow |
| `catch (Exception e) { }` | Specific exception types |
| Throwing `RuntimeException("error")` | Custom exception with context |
| Leaking stack traces | Use RFC 7807 |
| Logging and rethrowing (double logging) | Log at boundary only |
| Using exceptions for flow control | `Optional`, conditionals |
| Generic error messages to users | User-friendly + code |
| No correlation ID | Include in response |

---

## 6. Security Best Practices

### ✅ DO: Use Spring Security (Not Custom Filters)

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())  // for stateless APIs
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
    return http.build();
}
```

### ✅ DO: Encode Passwords with BCrypt

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // strength 12
}
```

❌ **AVOID:**

- MD5, SHA-1, SHA-256 (fast → brute-forceable)
- Plain text
- Custom "encryption" schemes

### ✅ DO: Use HTTPS + HSTS

```java
http.headers(headers -> headers
    .httpStrictTransportSecurity(hsts -> hsts
        .includeSubDomains(true)
        .maxAgeInSeconds(31536000)
        .preload(true)));
```

### ✅ DO: Use JWT with Short Expiry + Refresh

```java
String accessToken = Jwts.builder()
    .subject(user.getUsername())
    .claim("roles", user.getRoles())
    .issuedAt(new Date())
    .expiration(Date.from(Instant.now().plus(15, ChronoUnit.MINUTES)))  // short
    .signWith(secretKey)
    .compact();
```

Refresh tokens: longer-lived (7–30 days), stored in HTTP-only cookies or secure storage, revocable.

### ✅ DO: Enforce Authorization on Every Endpoint

```java
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public UserDTO getUser(Long userId) { }
```

### ✅ DO: Validate All Input

```java
public record CreateUserRequest(
    @NotBlank @Size(min = 2, max = 100) String name,
    @NotBlank @Email String email,
    @Min(18) @Max(120) Integer age
) { }
```

### ✅ DO: Use Parameterized Queries (Prevent SQL Injection)

```java
jdbcTemplate.queryForObject(
    "SELECT * FROM users WHERE email = ?",
    userRowMapper, email);   // ✅ parameterized
```

❌ **AVOID:**

```java
String sql = "SELECT * FROM users WHERE email = '" + email + "'";  // ❌
```

### ✅ DO: Prevent Mass Assignment with DTOs

```java
@PostMapping
public UserResponse create(@Valid @RequestBody CreateUserRequest req) {  // ✅ DTO
    return userService.create(req);
}
```

❌ **AVOID:**

```java
@PostMapping
public User create(@RequestBody User user) {   // ❌ client can set role=ADMIN
    return userRepository.save(user);
}
```

### ✅ DO: Restrict CORS

```java
config.setAllowedOrigins(List.of("https://app.example.com"));
config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
config.setAllowCredentials(true);
```

❌ **AVOID: `allowedOrigins("*")` with credentials**

### ✅ DO: Set Security Headers

```java
http.headers(headers -> headers
    .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
    .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
    .contentTypeOptions(Customizer.withDefaults())   // nosniff
    .referrerPolicy(rp -> rp.policy(ReferrerPolicy.NO_REFERRER))
    .permissionsPolicy(pp -> pp.policy("geolocation=(), camera=()")));
```

### ✅ DO: Rate Limit Public APIs

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    // ... token bucket per API key
}
```

### ✅ DO: Store Secrets in Vault/AWS Secrets Manager

```properties
spring.config.import=vault://
spring.cloud.vault.kv.default-context=myapp
```

❌ **AVOID: Secrets in Git, logs, error messages**

### ✅ DO: Disable Unnecessary Features in Production

```properties
spring.devtools.restart.enabled=false
spring.h2.console.enabled=false
spring.jpa.show-sql=false
server.error.include-stacktrace=never
server.error.include-message=never
management.endpoints.web.exposure.include=health,info,metrics
```

### ✅ DO: Log Security Events

```java
log.warn("Authentication failed for user: {} from IP: {}", username, ip);
log.info("User {} logged in from {}", username, ip);
log.warn("Access denied for {} to {}", username, path);
```

**Never log:** passwords, tokens, full credit card numbers, PII.

### ✅ DO: Keep Dependencies Updated

```bash
mvn versions:display-dependency-updates
mvn org.owasp:dependency-check-maven:check
```

### Security Anti-Patterns

| Anti-Pattern | Fix |
|--------------|-----|
| Custom auth filters | Spring Security |
| Plaintext/weak password hashing | BCrypt/Argon2 |
| HTTP in production | HTTPS + HSTS |
| `csrf().disable()` on stateful apps | Keep CSRF for forms |
| `permitAll()` on sensitive endpoints | Explicit auth |
| `@RequestBody User` (entity) | DTOs |
| String concatenation in SQL | Parameterized queries |
| CORS `*` with credentials | Specific origins |
| Long-lived JWTs | 15-min access + refresh |
| Secrets in code | Vault/env vars |
| Verbose errors to clients | RFC 7807, generic messages |
| Logging secrets | Mask/filter |

---

## 7. JPA/Hibernate Best Practices

### ✅ DO: Use `LAZY` Fetching by Default

```java
@ManyToOne(fetch = FetchType.LAZY)   // ✅
@JoinColumn(name = "user_id")
private User user;

@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)   // default, but explicit
private List<OrderItem> items = new ArrayList<>();
```

❌ **AVOID: `EAGER` everywhere**

```java
@ManyToOne(fetch = FetchType.EAGER)   // ❌ loads full graph on every query
```

### ✅ DO: Fetch What You Need with `JOIN FETCH` or `@EntityGraph`

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);

@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findByUserId(Long userId);
```

### ✅ DO: Detect and Fix N+1

**Enable SQL logging:**

```properties
spring.jpa.show-sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE
```

Then watch for repeated queries.

**Fix:**

- `JOIN FETCH`
- `@EntityGraph`
- `@BatchSize(size = N)` on entity
- `hibernate.default_batch_fetch_size=20` globally

### ✅ DO: Use `@Transactional(readOnly = true)` for Reads

```java
@Transactional(readOnly = true)
public OrderResponse findById(Long id) {
    return orderRepo.findById(id).map(this::toResponse).orElseThrow();
}
```

### ✅ DO: Use Projections for Read-Only Views

```java
// Interface projection
public interface OrderSummary {
    Long getId();
    String getStatus();
    BigDecimal getTotal();
}

List<OrderSummary> findByUserId(Long userId);

// Class-based DTO
@Query("SELECT new com.example.OrderDTO(o.id, o.status, o.total) FROM Order o")
List<OrderDTO> findAllAsDTO();
```

### ✅ DO: Always Paginate Large Results

```java
Page<Order> findByStatus(OrderStatus status, Pageable pageable);

// Usage
Page<Order> page = orderRepo.findByStatus(PENDING, PageRequest.of(0, 20, Sort.by("createdAt").descending()));
```

❌ **AVOID: `findAll()` on large tables**

### ✅ DO: Use Optimistic Locking for Concurrent Updates

```java
@Entity
public class Order {
    @Version
    private Long version;
}
```

Handles concurrent updates without locks; throws `OptimisticLockException` on conflict.

### ✅ DO: Prefer `persist` Over `merge` for New Entities

```java
em.persist(order);         // INSERT, efficient
em.merge(order);           // SELECT + UPDATE/INSERT, slower
```

**JPA `save()`:** Calls `persist` if ID is null, `merge` otherwise.

### ✅ DO: Configure Batch Inserts

```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.batch_versioned_data=true
```

### ✅ DO: Use Sequences for ID Generation (When Supported)

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
@SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
private Long id;
```

`IDENTITY` disables batch inserts (must fetch generated ID per row). `SEQUENCE` works with batching.

### ✅ DO: Use `@MappedSuperclass` for Auditing

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {
    
    @CreatedDate
    @Column(updatable = false)
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant updatedAt;
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String lastModifiedBy;
}
```

### ✅ DO: Set `spring.jpa.open-in-view=false`

```properties
spring.jpa.open-in-view=false
```

Prevents lazy loading outside transactions and connection holding.

### ✅ DO: Use DTOs at the Boundary

```java
@GetMapping("/{id}")
public OrderResponse get(@PathVariable Long id) {
    return orderService.findById(id);   // returns DTO, not entity
}
```

❌ **AVOID: Returning entities directly**

Lazy fields → `LazyInitializationException`, sensitive data leaks.

### ✅ DO: Handle `LazyInitializationException` Correctly

**Wrong:** catch and retry outside TX
**Right:** fetch in the transaction, convert to DTO before returning.

### ✅ DO: Use `@Enumerated(EnumType.STRING)`

```java
@Enumerated(EnumType.STRING)
private OrderStatus status;
```

❌ **AVOID: `EnumType.ORDINAL`** — brittle to enum reordering.

### ✅ DO: Index Foreign Keys and Query Columns

```java
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_orders_user", columnList = "user_id"),
    @Index(name = "idx_orders_status_created", columnList = "status, created_at")
})
public class Order { }
```

### ✅ DO: Use `ddl-auto=validate` in Production

```properties
spring.jpa.hibernate.ddl-auto=validate
```

Use Flyway/Liquibase for schema. `update` in prod is dangerous.

### JPA Anti-Patterns

| Anti-Pattern | Fix |
|--------------|-----|
| `EAGER` everywhere | `LAZY` + fetch joins |
| N+1 queries | `JOIN FETCH` / `@EntityGraph` |
| `findAll()` on large tables | Pagination |
| Returning entities | DTOs |
| `EnumType.ORDINAL` | `EnumType.STRING` |
| `ddl-auto=update` in prod | Flyway/Liquibase + `validate` |
| `open-in-view=true` | `false` |
| `IDENTITY` with batch inserts | `SEQUENCE` |
| No `@Version` | Optimistic locking |
| Missing indexes | Index FKs and filters |
| `@Transactional` on controllers | Move to service |
| Lazy loading outside TX | Fetch in TX, return DTO |
| `merge` for new entities | `persist` |

---

## 8. REST API Best Practices

### ✅ DO: Use Nouns in URIs, Verbs via HTTP Methods

```
GET    /users
POST   /users
GET    /users/{id}
PUT    /users/{id}
PATCH  /users/{id}
DELETE /users/{id}
```

❌ **AVOID:**

```
GET  /getUsers
POST /createUser
POST /deleteUser
```

### ✅ DO: Use Plural Resource Names

```
/users          ✅
/user           ❌
/order-items    ✅
/orderItems     ❌
```

### ✅ DO: Return Proper HTTP Status Codes

| Action | Status |
|--------|--------|
| GET found | 200 OK |
| GET not found | 404 Not Found |
| POST created | 201 Created + `Location` |
| PUT updated | 200 OK |
| DELETE | 204 No Content |
| Validation error | 400 Bad Request |
| Unauthenticated | 401 Unauthorized |
| Unauthorized (logged in, no perm) | 403 Forbidden |
| Conflict | 409 Conflict |
| Rate limited | 429 Too Many Requests |

### ✅ DO: Use DTOs, Never Entities

```java
@GetMapping("/{id}")
public UserResponse get(@PathVariable Long id) {
    return userService.findById(id);   // DTO
}
```

### ✅ DO: Paginate Large Collections

```java
@GetMapping
public Page<UserResponse> list(@PageableDefault(size = 20) Pageable pageable) {
    return userService.findAll(pageable);
}
```

### ✅ DO: Support Filtering and Sorting

```java
@GetMapping
public Page<UserResponse> list(
        @RequestParam(required = false) String name,
        @RequestParam(required = false) Integer minAge,
        @PageableDefault(sort = "id") Pageable pageable) {
    return userService.search(name, minAge, pageable);
}
```

### ✅ DO: Use HATEOAS for Public APIs (Optional)

```java
user.add(linkTo(methodOn(UserController.class).get(id)).withSelfRel());
user.add(linkTo(methodOn(OrderController.class).listByUser(id)).withRel("orders"));
```

### ✅ DO: Version from Day One

```
/api/v1/users
/api/v2/users
```

### ✅ DO: Document with OpenAPI

```java
@Operation(summary = "Get user by ID")
@ApiResponse(responseCode = "200", description = "Found")
@ApiResponse(responseCode = "404", description = "Not found")
@GetMapping("/{id}")
public UserResponse get(@Parameter(description = "User ID") @PathVariable Long id) { }
```

### ✅ DO: Use Consistent Error Format

```json
{
  "type": "about:blank",
  "title": "Validation Error",
  "status": 400,
  "detail": "Request validation failed",
  "errors": {
    "email": "must be a valid email",
    "age": "must be at least 18"
  },
  "correlationId": "abc-123"
}
```

### ✅ DO: Return `Location` Header on 201

```java
URI location = uriBuilder.path("/api/v1/users/{id}").buildAndExpand(user.id()).toUri();
return ResponseEntity.created(location).body(user);
```

### ✅ DO: Support Partial Updates with PATCH

```java
@PatchMapping("/{id}")
public UserResponse patch(@PathVariable Long id,
                          @RequestBody JsonPatch patch) {
    // apply JSON Patch (RFC 6902)
}
```

### ✅ DO: Use Content Negotiation

```java
@GetMapping(produces = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE})
```

### ✅ DO: Cache with ETags / Cache-Control

```java
@GetMapping("/{id}")
public ResponseEntity<UserResponse> get(@PathVariable Long id, WebRequest req) {
    UserResponse user = userService.findById(id);
    String etag = "\"" + user.version() + "\"";
    if (req.checkNotModified(etag)) return null;
    return ResponseEntity.ok().eTag(etag).cacheControl(CacheControl.maxAge(Duration.ofMinutes(5))).body(user);
}
```

### ✅ DO: Document and Handle Rate Limiting

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
```

### REST Anti-Patterns

| Anti-Pattern | Fix |
|--------------|-----|
| Verbs in URIs | HTTP methods |
| Singular resource names | Plural |
| 200 for errors | Proper status codes |
| Returning entities | DTOs |
| Exposing DB IDs | UUIDs or opaque IDs |
| No pagination | Always paginate |
| No versioning | Version from day one |
| Breaking changes | New version |
| Chatty APIs | Batch/composite endpoints |
| No rate limiting | Add throttling |
| Verbose errors | RFC 7807 |
| No OpenAPI | Document with SpringDoc |

---

## 9. Testing Best Practices

### ✅ DO: Follow the Test Pyramid

```
        ┌─────────┐
        │   E2E   │  Few
        ├─────────┤
        │Integration│  Some
        ├─────────┤
        │  Unit   │  Many
        └─────────┘
```

### ✅ DO: Write Fast, Deterministic Unit Tests

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock private OrderRepository orderRepo;
    @Mock private PaymentService paymentService;
    @InjectMocks private OrderService orderService;
    
    @Test
    void createOrder_savesAndCharges() {
        // Given
        var req = new CreateOrderRequest(1L, 100.0);
        when(paymentService.charge(any())).thenReturn(new PaymentResult("OK"));
        when(orderRepo.save(any())).thenAnswer(i -> i.getArgument(0));
        
        // When
        Order order = orderService.create(req);
        
        // Then
        assertThat(order.getStatus()).isEqualTo(PAID);
        verify(orderRepo).save(any());
    }
}
```

### ✅ DO: Use Slice Tests for Layers

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean UserService userService;
    // ...
}

@DataJpaTest
class UserRepositoryTest {
    @Autowired UserRepository repo;
    @Autowired TestEntityManager em;
}

@JsonTest
class UserJsonTest { }

@RestClientTest(PaymentClient.class)
class PaymentClientTest { }
```

Faster than full `@SpringBootTest` — only loads the relevant slice.

### ✅ DO: Use Testcontainers for Integration Tests

```java
@SpringBootTest
@Testcontainers
class OrderIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    
    @DynamicPropertySource
    static void config(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
    }
    // ...
}
```

**Why:** H2 differs from production DB (SQL dialect, constraints, isolation). Testcontainers ensures parity.

### ✅ DO: Use `@Transactional` for Rollback (Slice Tests)

```java
@DataJpaTest   // auto @Transactional — rolls back after each test
class UserRepositoryTest { }
```

**Caveat:** Don't use `@Transactional` on `@SpringBootTest` with `RANDOM_PORT` — different thread, no rollback.

### ✅ DO: One Assertion per Test (or One Logical Concept)

```java
@Test
void findById_returnsUser() {
    // ... one logical assertion
}

@Test
void findById_throwsWhenNotFound() {
    // ... one logical assertion
}
```

### ✅ DO: Use Descriptive Test Names

```java
void createUser_throwsWhenEmailExists() { }         // ✅
void createUser_savesAndReturnsResponse() { }       // ✅
void test1() { }                                     // ❌
void testCreateUser() { }                            // ❌ vague
```

Pattern: `methodUnderTest_condition_expectedOutcome`

### ✅ DO: Follow AAA (Arrange, Act, Assert)

```java
@Test
void createOrder_chargesPayment() {
    // Arrange
    var req = new CreateOrderRequest(1L, 100.0);
    when(paymentService.charge(any())).thenReturn(OK);
    
    // Act
    Order result = orderService.create(req);
    
    // Assert
    assertThat(result.getStatus()).isEqualTo(PAID);
}
```

### ✅ DO: Use AssertJ for Readable Assertions

```java
assertThat(users).hasSize(3)
    .extracting(User::getName)
    .containsExactly("Alice", "Bob", "Charlie");

assertThatThrownBy(() -> userService.findById(99L))
    .isInstanceOf(ResourceNotFoundException.class)
    .hasMessageContaining("99");
```

### ✅ DO: Mock External Services (Not Internals)

```java
@MockBean private PaymentGateway paymentGateway;   // ✅ external
@MockBean private UserRepository userRepository;   // ⚠️ prefer Testcontainers
```

### ✅ DO: Use `@Sql` for Test Data

```java
@Test
@Sql("/test-data/users.sql")
void findActiveUsers_returnsOnlyActive() {
    List<User> users = userRepo.findByActiveTrue();
    assertThat(users).hasSize(2);
}
```

### ✅ DO: Test Edge Cases

- Null inputs
- Empty collections
- Boundary values (`Integer.MAX_VALUE`, `0`, negative)
- Concurrent modification
- Missing resources
- Unauthorized access

### ✅ DO: Aim for 70–80% Coverage (Not 100%)

- Critical paths: high coverage
- DTOs, config: minimal
- Framework code: not your concern

Focus on behavior coverage, not line coverage.

### ✅ DO: Use `@MockBean` vs `@Mock` Correctly

| Annotation | When |
|-----------|------|
| `@Mock` (Mockito) | Unit tests, no Spring |
| `@MockBean` (Boot) | Integration/slice tests, replaces Spring bean |
| `@SpyBean` | Partial mock of real Spring bean |

### ✅ DO: Prefer `@Spy` Over `@MockBean` for Partial Mocking

```java
@SpyBean
private OrderService orderService;

doReturn(mockOrder).when(orderService).findById(any());
```

### Testing Anti-Patterns

| Anti-Pattern | Fix |
|--------------|-----|
| Only E2E tests | Test pyramid |
| `@SpringBootTest` for everything | Use slice tests |
| Test interdependence | Independent tests |
| `Thread.sleep()` | Awaitility or virtual time |
| H2 for prod parity | Testcontainers |
| No assertion messages | Descriptive assertions |
| Testing private methods | Test behavior |
| Mocking everything | Mock external only |
| 100% coverage goal | 70–80% behavior |
| Flaky tests | Deterministic, isolated |

---

## 10. Performance Anti-Patterns

### ❌ EAGER Fetching Everywhere

```java
@ManyToOne(fetch = FetchType.EAGER)   // loads full graph on every query
```

**Fix:** `LAZY` + explicit `JOIN FETCH` or `@EntityGraph`.

### ❌ N+1 Queries

```java
List<Order> orders = orderRepo.findAll();       // 1 query
for (Order o : orders) {
    o.getItems().size();                        // N more queries!
}
```

**Fix:** `JOIN FETCH`, `@EntityGraph`, `@BatchSize`, projections.

### ❌ Loading Entire Tables

```java
List<User> allUsers = userRepo.findAll();   // millions of rows!
```

**Fix:** Pagination (`Pageable`), streaming (`Stream<User>`), projections.

### ❌ Long Transactions with I/O

```java
@Transactional
public void process() {
    Order order = repo.findById(id).orElseThrow();
    httpClient.callExternalApi(order);   // ❌ holds DB connection
    Thread.sleep(5000);
}
```

**Fix:** Do I/O outside transaction.

### ❌ Excessive Logging in Hot Paths

```java
for (Order o : orders) {
    log.debug("Processing order {}", o);   // ❌ millions of log calls
}
```

**Fix:** Log aggregately, use appropriate level, guard with `if (log.isDebugEnabled())`.

### ❌ Synchronized on Shared State in Singletons

```java
@Service
public class CounterService {
    private int count;   // ❌
    
    public synchronized void increment() { count++; }
}
```

**Fix:** `AtomicInteger`, `LongAdder`, or stateless design.

### ❌ Creating Threads Manually

```java
new Thread(() -> process()).start();   // ❌ unbounded, no pool
```

**Fix:** `@Async` with configured `TaskExecutor`, or `CompletableFuture` with a pool.

### ❌ Missing Database Indexes

**Fix:** Index foreign keys and query columns; analyze slow queries with `EXPLAIN`.

### ❌ Using `String` Concatenation in Loops

```java
String s = "";
for (int i = 0; i < 10000; i++) {
    s += i;   // ❌ O(n²) — creates 10000 strings
}
```

**Fix:** `StringBuilder` or `String.join`.

### ❌ Boxing in Hot Loops

```java
Long sum = 0L;
for (long i = 0; i < 1_000_000; i++) {
    sum += i;   // ❌ autoboxing each iteration
}
```

**Fix:** `long sum = 0L;`

### ❌ Reflection in Hot Paths

**Fix:** Cache `Method` objects, use codegen (MapStruct), or avoid.

### ❌ Inefficient Collection Choices

```java
List<String> list = new ArrayList<>();
if (list.contains(x)) { }   // ❌ O(n)
```

**Fix:** `HashSet` for membership tests (O(1)).

### ❌ Not Caching Immutable Reference Data

**Fix:** `@Cacheable` for countries, currencies, config.

### ❌ Serializing Huge Object Graphs

**Fix:** DTOs with only needed fields, projections.

### ❌ Synchronous Calls in Parallelizable Work

```java
User user = userService.get(id);
List<Order> orders = orderService.findByUser(id);   // sequential
List<Payment> payments = paymentService.findByUser(id);
```

**Fix:** `CompletableFuture` or `@Async`.

### ❌ Blocking in Reactive Code

```java
Mono<User> user = userRepo.findById(id).map(u -> blockingCall(u));   // ❌
```

**Fix:** `flatMap(u -> Mono.fromCallable(() -> blockingCall(u)).subscribeOn(Schedulers.boundedElastic()))`.

### ❌ Not Using Connection Pooling

**Fix:** HikariCP (default in Spring Boot), tuned pool size.

### ❌ Repeatedly Creating Expensive Objects

```java
for (...) {
    ObjectMapper mapper = new ObjectMapper();   // ❌
}
```

**Fix:** Reuse singletons or inject.

### Performance Checklist

- [ ] LAZY fetching by default
- [ ] N+1 eliminated
- [ ] Pagination on all list endpoints
- [ ] Transactions short, no I/O inside
- [ ] HikariCP pool sized
- [ ] Hibernate batch inserts/updates
- [ ] `open-in-view=false`
- [ ] Caching for hot, immutable data
- [ ] Indexes on FKs and query columns
- [ ] Async for I/O-bound work
- [ ] Virtual threads (Java 21+) for blocking tasks
- [ ] JVM heap sized to container
- [ ] GC tuned (G1GC or ZGC)
- [ ] HTTP compression enabled
- [ ] Structured logging without hot-path spam
- [ ] Profile with JFR/async-profiler before optimizing

---

## 11. Logging Best Practices

### ✅ DO: Use SLF4J (Not Log4j Directly)

```java
private static final Logger log = LoggerFactory.getLogger(MyService.class);
```

### ✅ DO: Use Parameterized Logging

```java
log.info("User created: id={}, email={}", user.getId(), user.getEmail());   // ✅
log.info("User created: id=" + user.getId());   // ❌ string concat
```

**Why:** Parameterized avoids string concatenation when the log level is disabled.

### ✅ DO: Use Guarded Logging for Expensive Operations

```java
if (log.isDebugEnabled()) {
    log.debug("Full order: {}", objectMapper.writeValueAsString(order));
}
```

### ✅ DO: Log Exceptions with Stack Trace

```java
log.error("Failed to process order {}", orderId, ex);   // ✅ includes stack trace
log.error("Failed: " + ex.getMessage());                 // ❌ loses stack trace
```

### ✅ DO: Choose the Right Level

| Level | When |
|-------|------|
| ERROR | Failures requiring immediate attention |
| WARN | Recoverable issues, deprecations |
| INFO | Business events, startup, key actions |
| DEBUG | Development diagnostics |
| TRACE | Very detailed, rarely used |

### ✅ DO: Log at Boundaries

```java
// Controller entry/exit
log.info("POST /orders userId={}", userId);

// External service calls
log.debug("Calling payment service for orderId={}", orderId);

// Exception handling
log.error("Order processing failed", ex);
```

### ✅ DO: Include Context

```java
log.info("Order {} created by user {} with total {}", order.getId(), userId, total);
```

### ✅ DO: Use Correlation IDs (MDC)

```java
MDC.put("correlationId", correlationId);
log.info("Processing order {}", orderId);
```

**Log pattern:**
```
%d{HH:mm:ss} [%thread] %-5level [%X{correlationId}] %logger{36} - %msg%n
```

### ✅ DO: Never Log Sensitive Data

❌ **AVOID:**

```java
log.info("Login attempt: user={}, password={}", username, password);   // ❌
log.debug("Token: {}", jwtToken);                                       // ❌
log.info("Credit card: {}", cardNumber);                                // ❌
```

**Mask:** card numbers, passwords, tokens, SSNs, PII.

### ✅ DO: Use Structured Logging in Production (JSON)

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <includeMdcKeyName>correlationId</includeMdcKeyName>
    <includeMdcKeyName>traceId</includeMdcKeyName>
    <customFields>{"app":"myapp","env":"prod"}</customFields>
</encoder>
```

### ✅ DO: Use `@Slf4j` (Lombok)

```java
@Slf4j
@Service
public class OrderService {
    public void process() {
        log.info("Processing");
    }
}
```

### ✅ DO: Configure Log Rotation

```xml
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/logs/app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>/logs/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>30</maxHistory>
        <totalSizeCap>10GB</totalSizeCap>
    </rollingPolicy>
</appender>
```

### ✅ DO: Don't Log in Tight Loops

```java
// ❌ logs N times
for (User u : users) log.debug("User: {}", u);

// ✅ aggregate
log.debug("Processing {} users", users.size());
```

### ✅ DO: Use Async Appenders for High Throughput

```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>1024</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <appender-ref ref="FILE"/>
</appender>
```

### Logging Anti-Patterns

| Anti-Pattern | Fix |
|--------------|-----|
| `System.out.println` | SLF4J |
| String concat in logs | Parameterized |
| No context in messages | Include IDs |
| Logging passwords/tokens | Mask or exclude |
| `log.error` for expected errors | `warn` or `info` |
| Logging inside loops | Aggregate |
| No correlation ID | MDC filter |
| INFO level for everything | Proper levels |
| No rotation | SizeAndTimeBased |

---

## 12. Common Mistakes & How to Avoid Them

### Mistake 1: Field Injection

```java
@Autowired private UserRepository repo;   // ❌
```

**Fix:** Constructor injection with `final`.

### Mistake 2: Circular Dependencies

```
A → B → A   // ❌ design smell
```

**Fix:** Extract shared logic to C; both depend on C.

### Mistake 3: `@Transactional` Self-Invocation

```java
public void outer() {
    inner();   // ❌ no TX — same class, proxy bypassed
}

@Transactional
public void inner() { }
```

**Fix:** Extract `inner` to a separate bean.

### Mistake 4: Catching and Swallowing Exceptions

```java
try { process(); } catch (Exception e) { }   // ❌
```

**Fix:** Log and rethrow, or handle meaningfully.

### Mistake 5: Returning Entities from Controllers

```java
@GetMapping
public User get() { return userRepo.findById(1L).get(); }   // ❌
```

**Fix:** Return DTOs.

### Mistake 6: N+1 Queries

Watch SQL logs. Fix with `JOIN FETCH` / `@EntityGraph`.

### Mistake 7: `findAll()` on Large Tables

**Fix:** Paginate.

### Mistake 8: `@Configuration` on Classes That Should Be `@Component`

```java
@Configuration
public class EmailService { }   // ❌ CGLIB overhead, wrong semantics

@Component
public class EmailService { }   // ✅
```

### Mistake 9: `@Component` with `@Bean` Methods

```java
@Component
public class MyConfig {
    @Bean
    public Foo foo() { return new Foo(bar()); }
    @Bean
    public Bar bar() { return new Bar(); }   // ❌ new Bar each call
}
```

**Fix:** Use `@Configuration`.

### Mistake 10: `@Autowired(required=false)` for Optional Deps

```java
@Autowired(required = false)
private Metrics metrics;   // ❌ silent null
```

**Fix:** `ObjectProvider<Metrics>`.

### Mistake 11: Not Setting `readOnly=true` on Reads

```java
@Transactional   // ❌ without readOnly
public List<User> findAll() { }
```

**Fix:** `@Transactional(readOnly = true)`.

### Mistake 12: Long Transactions with I/O

**Fix:** Do I/O outside TX, short DB TX.

### Mistake 13: `EAGER` Fetching

**Fix:** `LAZY` + explicit fetch.

### Mistake 14: `open-in-view=true`

```properties
spring.jpa.open-in-view=false   # ✅
```

### Mistake 15: `ddl-auto=update` in Production

**Fix:** Flyway/Liquibase + `validate`.

### Mistake 16: Not Versioning APIs

**Fix:** `/v1/`, `/v2/` from day one.

### Mistake 17: Secrets in Properties/Git

**Fix:** Vault, AWS Secrets Manager, K8s Secrets.

### Mistake 18: Logging Sensitive Data

**Fix:** Mask or exclude PII, tokens, passwords.

### Mistake 19: Not Handling Timeouts on External Calls

```java
restTemplate.getForObject("https://slow-service/api", ...);   // ❌ infinite wait
```

**Fix:** Configure connect + read timeouts, circuit breaker.

### Mistake 20: Creating Threads Manually

**Fix:** `@Async` with `TaskExecutor`, or virtual threads.

### Mistake 21: Using H2 for Production-Parity Tests

**Fix:** Testcontainers with real DB.

### Mistake 22: Testing Everything with `@SpringBootTest`

**Fix:** Slice tests (`@WebMvcTest`, `@DataJpaTest`).

### Mistake 23: Mocking Everything

**Fix:** Mock external dependencies only; use real DB (Testcontainers).

### Mistake 24: No Pagination, No Limits

**Fix:** Always paginate list endpoints; set max page size.

### Mistake 25: No Rate Limiting on Public APIs

**Fix:** Bucket4j, Redis-backed for distributed.

### Mistake 26: No Correlation IDs

**Fix:** `CorrelationIdFilter` + MDC.

### Mistake 27: Ignoring `@Version` for Concurrent Updates

**Fix:** Add `@Version` field for optimistic locking.

### Mistake 28: Manual SQL String Concatenation

**Fix:** Parameterized queries.

### Mistake 29: No Validation on Input

**Fix:** `@Valid` + Bean Validation annotations.

### Mistake 30: Verbose Error Responses

**Fix:** RFC 7807 ProblemDetail; log details server-side.

---

## 13. Code Review Checklist

### Architecture

- [ ] Layered architecture respected (controller → service → repository)
- [ ] No business logic in controllers
- [ ] No HTTP concepts in services
- [ ] DTOs at boundaries; entities never exposed
- [ ] Feature-based packages (or justified layer-based)

### Dependency Injection

- [ ] Constructor injection with `final` fields
- [ ] No field injection
- [ ] No circular dependencies
- [ ] Interfaces for dependencies (ports)
- [ ] ≤ 5 constructor dependencies
- [ ] No `@Autowired(required=false)`

### Configuration

- [ ] `@ConfigurationProperties` for grouped config
- [ ] No hardcoded env values
- [ ] Secrets externalized (Vault/env)
- [ ] `@Validated` on config classes
- [ ] `proxyBeanMethods=false` when possible

### Transactions

- [ ] `@Transactional` on service layer
- [ ] `readOnly=true` for reads
- [ ] No self-invocation
- [ ] No long-running I/O inside TX
- [ ] Explicit `rollbackFor` when needed
- [ ] `open-in-view=false`

### JPA

- [ ] LAZY by default
- [ ] No N+1 (check SQL logs)
- [ ] Pagination on list endpoints
- [ ] Projections for read-only
- [ ] `@Version` for concurrent updates
- [ ] `EnumType.STRING`
- [ ] DTOs at boundary
- [ ] Indexes on FKs and query columns

### Exception Handling

- [ ] Global handler (`@RestControllerAdvice`)
- [ ] Custom exception hierarchy
- [ ] RFC 7807 responses
- [ ] No stack traces to clients
- [ ] No swallow-and-ignore
- [ ] Correlation ID in errors

### Security

- [ ] HTTPS + HSTS
- [ ] Auth on every endpoint
- [ ] Method-level authorization
- [ ] Input validation
- [ ] Parameterized queries
- [ ] DTOs prevent mass assignment
- [ ] Secrets in Vault
- [ ] Security headers set
- [ ] Rate limiting
- [ ] No sensitive data logged

### REST

- [ ] Nouns in URIs, HTTP methods for verbs
- [ ] Proper status codes
- [ ] `Location` header on 201
- [ ] Pagination for collections
- [ ] API versioning
- [ ] OpenAPI documented
- [ ] Consistent error format

### Testing

- [ ] Unit tests for services
- [ ] Slice tests for layers
- [ ] Integration tests with Testcontainers
- [ ] No H2 for prod-parity
- [ ] Descriptive test names
- [ ] Edge cases covered
- [ ] Coverage ≥ 70%

### Performance

- [ ] Caching for hot data
- [ ] Async for I/O-bound
- [ ] HikariCP tuned
- [ ] Hibernate batch configured
- [ ] No logging in hot loops
- [ ] JVM container-aware

### Code Quality

- [ ] Consistent naming
- [ ] No dead code
- [ ] Small methods (≤ 30 lines)
- [ ] Clear intent
- [ ] No commented-out code
- [ ] Javadoc for public APIs (where non-obvious)

### Observability

- [ ] Structured logging with correlation IDs
- [ ] Custom metrics (RED/USE)
- [ ] Health indicators
- [ ] Tracing enabled
- [ ] Alerting rules

---

## 14. Interview Questions

### Q1. Why is constructor injection preferred?

**Answer:** Immutability (final fields), guaranteed dependencies, fail-fast at startup, easy unit testing without Spring, no Spring imports needed, detects circular dependencies. Field injection hides dependencies and prevents immutability.

### Q2. When would you use setter injection?

**Answer:** Optional dependencies (with `@Autowired(required=false)` or `ObjectProvider`), reconfiguration after construction (rare), or legacy code. Not for required dependencies.

### Q3. How do you avoid circular dependencies?

**Answer:** Redesign — extract shared logic into a third class. Or use `@Lazy` on one dependency. Circular deps are a design smell; don't just `spring.main.allow-circular-references=true`.

### Q4. Why not put `@Transactional` on controllers?

**Answer:** Controllers deal with HTTP; transactions belong to the service layer. TX in controllers holds DB connections during HTTP processing and mixes concerns. Also, `@Transactional` on controller methods may span view rendering.

### Q5. Why is self-invocation a problem?

**Answer:** Spring AOP uses proxies. Calling a method on `this` bypasses the proxy, so `@Transactional`, `@Cacheable`, `@Async` etc. are ignored. Fix: extract to another bean, inject self, or use `AopContext.currentProxy()`.

### Q6. How do you handle checked exceptions with `@Transactional`?

**Answer:** By default, Spring rolls back on unchecked only. Use `@Transactional(rollbackFor = Exception.class)` to include checked exceptions. Or catch and call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.

### Q7. Why `spring.jpa.open-in-view=false`?

**Answer:** Open Session In View keeps the Hibernate session open during view rendering, allowing lazy loading outside transactions. This causes N+1 queries, holds connections, and encourages bad patterns. Disabling forces explicit fetching in the service layer.

### Q8. How do you prevent mass assignment?

**Answer:** Never bind entities to request bodies. Use DTOs with only allowed fields. Map DTO → entity in service. Never expose fields like `role`, `admin`, `createdAt` for client control.

### Q9. What's wrong with `EAGER` fetching?

**Answer:** Loads the full graph on every query — even when you don't need the association. Causes over-fetching and performance issues. Use `LAZY` + explicit `JOIN FETCH` or `@EntityGraph` for specific queries.

### Q10. How do you detect N+1 queries?

**Answer:** Enable SQL logging (`spring.jpa.show-sql=true`, `logging.level.org.hibernate.SQL=DEBUG`), watch for repeated SELECTs. In production, use APM (Datadog, New Relic) or P6Spy. Fix with `JOIN FETCH`, `@EntityGraph`, `@BatchSize`, or projections.

### Q11. What is the difference between `save()` and `persist()`?

**Answer:** JPA `EntityManager.persist()` inserts. `merge()` updates (or inserts if no ID). Spring Data `save()` calls `persist` if the entity has no ID, `merge` otherwise. `persist` is more efficient — no SELECT first.

### Q12. Why use `SEQUENCE` over `IDENTITY`?

**Answer:** `IDENTITY` disables JDBC batch inserts (Hibernate must fetch the generated ID after each insert). `SEQUENCE` (or `TABLE`) supports batching, improving bulk insert performance.

### Q13. When to use `@Transactional(readOnly=true)`?

**Answer:** All read-only service methods. Enables Hibernate optimizations (no dirty checking), sets JDBC connection read-only (some DBs route to replicas), and signals intent.

### Q14. How long should a transaction be?

**Answer:** As short as possible. Never include I/O (HTTP, file, email) — those should happen outside the transaction. Long transactions hold DB connections and locks, reduce concurrency, and increase deadlock risk.

### Q15. How do you handle a slow external API in a transaction?

**Answer:** Do the external call outside the transaction. Structure:
1. TX 1: load entity
2. Call external API (no TX)
3. TX 2: save result

Or use async with events and separate transactions.

### Q16. `@Component` vs `@Configuration`?

**Answer:** `@Configuration` is CGLIB-enhanced — inter-bean method calls return singletons. `@Component` is not — each `@Bean` method call creates a new instance. Use `@Configuration` for bean factories; `@Component` for regular stereotypes.

### Q17. What makes a test "good"?

**Answer:** Fast, deterministic, isolated, focused (one concept), descriptive names, covers edge cases, doesn't depend on other tests, uses real dependencies (Testcontainers) where behavior matters.

### Q18. Why not use H2 for tests?

**Answer:** H2 differs from production DBs: SQL dialect, constraints, isolation levels, default behaviors. Tests may pass with H2 and fail with Postgres. Testcontainers gives production parity.

### Q19. When is caching appropriate?

**Answer:** For data that is:
- Read frequently
- Rarely changes
- Expensive to compute/fetch
- Tolerant of staleness (or has explicit invalidation)

Never for user-specific state that changes often, or for data requiring strong consistency.

### Q20. How do you decide between `RestTemplate`, `RestClient`, `WebClient`?

**Answer:**
- `RestTemplate`: legacy, blocking, in maintenance mode
- `RestClient`: modern, blocking, fluent API (Spring 6.1+)
- `WebClient`: reactive, non-blocking, for WebFlux or high concurrency

Use `RestClient` for new MVC apps; `WebClient` for reactive/high-throughput.

### Q21. How do you configure timeouts on HTTP clients?

**Answer:** Always set both connect and read timeouts. For `RestClient`:
```java
RestClient.builder()
    .requestFactory(ClientHttpRequestFactorySettings.defaults()
        .withConnectTimeout(Duration.ofSeconds(5))
        .withReadTimeout(Duration.ofSeconds(10))
        .requestFactory())
    .build();
```

Never leave defaults — can hang forever.

### Q22. Why use circuit breakers?

**Answer:** Prevents cascading failures. If a downstream service is down, fail fast instead of piling up threads. States: CLOSED (normal), OPEN (fail fast), HALF_OPEN (probe recovery). Resilience4j is the modern choice.

### Q23. What's the difference between fail-fast and graceful degradation?

**Answer:** Fail-fast = reject immediately when a dependency is down (circuit breaker open). Graceful degradation = return degraded response (cached data, defaults). Choose based on business impact: payments fail-fast; product recommendations can degrade.

### Q24. How do you ensure API backward compatibility?

**Answer:**
- Never remove/rename fields
- Never tighten validation
- Add new fields (clients ignore unknown)
- Add new endpoints
- Version when breaking changes are needed
- Contract tests (Spring Cloud Contract, Pact)

### Q25. What's the biggest Spring anti-pattern you've seen?

**Answer:** Field injection with `@Autowired` everywhere, followed by misuse of `@Transactional` (self-invocation, checked exceptions not rolling back, long-running transactions). Also common: returning entities from controllers (lazy loading issues, over-exposure) and `EAGER` fetching causing N+1.

---

## Cross-References

- **Previous file:** `25_Spring_Project_Architecture.md`
- **Next file:** `27_Spring_Cheat_Sheet.md`
- **Related:** All files 01–25 for specific topics

---

## Summary Quick Reference

### Top 10 Best Practices

1. **Constructor injection** with `final` fields
2. **DTOs at boundaries** — never entities in controllers
3. **`@Transactional(readOnly=true)`** for reads; short transactions
4. **LAZY fetching** + `JOIN FETCH` when needed
5. **Global exception handler** with RFC 7807
6. **`@ConfigurationProperties`** for grouped config
7. **Testcontainers** for integration tests
8. **Structured logging** with correlation IDs
9. **Parameterized queries** — never concat SQL
10. **Circuit breakers + timeouts** for external calls

### Top 10 Anti-Patterns

1. Field injection
2. `@Transactional` self-invocation
3. N+1 queries
4. `EAGER` everywhere
5. Returning entities from controllers
6. Catching and swallowing exceptions
7. Secrets in Git
8. `findAll()` on large tables
9. Missing pagination
10. Verbose error responses

### Code Review Mantra

> **"Does it fail fast, log clearly, paginate, validate, secure, and test?"**

If yes to all → likely a good PR.

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [25_Spring_Project_Architecture.md](./25_Spring_Project_Architecture.md)
- **Next →:** [27_Spring_Cheat_Sheet.md](./27_Spring_Cheat_Sheet.md)
- **Related:** [27_Spring_Cheat_Sheet.md](./27_Spring_Cheat_Sheet.md), [01_Spring_Framework_Core.md](./01_Spring_Framework_Core.md), [10_Spring_Data_JPA.md](./10_Spring_Data_JPA.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
