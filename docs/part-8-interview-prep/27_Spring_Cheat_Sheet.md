# Spring Ecosystem Cheat Sheet

> **File:** `27_Spring_Cheat_Sheet.md`
> **Part:** 8 — Practice & Interview Prep
> **Purpose:** Quick reference for all key annotations, configurations, commands, and patterns
> **Use:** Print it, bookmark it, keep it open while coding

---

## Table of Contents

1. [Core Spring Annotations](#1-core-spring-annotations)
2. [Spring Boot Annotations](#2-spring-boot-annotations)
3. [Spring MVC / REST Annotations](#3-spring-mvc--rest-annotations)
4. [Spring Data JPA](#4-spring-data-jpa)
5. [Spring Security](#5-spring-security)
6. [Spring AOP](#6-spring-aop)
7. [Spring Transaction](#7-spring-transaction)
8. [Spring Caching](#8-spring-caching)
9. [Spring Batch](#9-spring-batch)
10. [Spring Messaging](#10-spring-messaging)
11. [Spring Cloud](#11-spring-cloud)
12. [Spring WebFlux](#12-spring-webflux)
13. [Testing Annotations](#13-testing-annotations)
14. [Configuration Properties](#14-configuration-properties)
15. [Maven Cheat Sheet](#15-maven-cheat-sheet)
16. [Gradle Cheat Sheet](#16-gradle-cheat-sheet)
17. [Spring Boot Actuator Endpoints](#17-spring-boot-actuator-endpoints)
18. [Common Patterns & Snippets](#18-common-patterns--snippets)
19. [Docker & Kubernetes Snippets](#19-docker--kubernetes-snippets)
20. [Interview Rapid-Fire Answers](#20-interview-rapid-fire-answers)

---

## 1. Core Spring Annotations

### Stereotypes

| Annotation | Purpose | Layer |
|-----------|---------|-------|
| `@Component` | Generic Spring bean | Any |
| `@Service` | Business logic | Service |
| `@Repository` | Data access + exception translation | Persistence |
| `@Controller` | Web MVC controller | Web |
| `@RestController` | `@Controller` + `@ResponseBody` | Web |

### Configuration

| Annotation | Purpose |
|-----------|---------|
| `@Configuration` | Java config class (CGLIB-enhanced) |
| `@Configuration(proxyBeanMethods = false)` | No CGLIB; faster startup |
| `@Bean` | Factory method producing a bean |
| `@Import` | Include other config classes |
| `@ComponentScan` | Discover components |
| `@PropertySource` | Load properties file |
| `@PropertySources` | Multiple property files |
| `@EnableAutoConfiguration` | Boot auto-config |
| `@SpringBootApplication` | `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |

### Dependency Injection

| Annotation | Purpose |
|-----------|---------|
| `@Autowired` | Inject dependency |
| `@Autowired(required = false)` | Optional (avoid; use `ObjectProvider`) |
| `@Qualifier("name")` | Specific bean by name |
| `@Primary` | Default among candidates |
| `@Lazy` | Defer instantiation |
| `@Value("${prop}")` | Inject property |
| `@Value("#{spel}")` | Inject SpEL expression |
| `@RequiredArgsConstructor` | Lombok: constructor for final fields |

### Bean Lifecycle

| Annotation / Interface | Phase |
|------------------------|-------|
| `@PostConstruct` | After DI, before use |
| `@PreDestroy` | Before container shutdown |
| `InitializingBean.afterPropertiesSet()` | Init callback |
| `DisposableBean.destroy()` | Destroy callback |
| `BeanNameAware` | Receive bean name |
| `BeanFactoryAware` | Receive BeanFactory |
| `ApplicationContextAware` | Receive ApplicationContext |
| `BeanPostProcessor` | Wrap/modify beans |
| `BeanFactoryPostProcessor` | Modify bean definitions |

### Scope

| Annotation | Scope |
|-----------|-------|
| `@Scope("singleton")` | One per container (default) |
| `@Scope("prototype")` | New per request |
| `@RequestScope` | Per HTTP request |
| `@SessionScope` | Per HTTP session |
| `@ApplicationScope` | Per ServletContext |
| `@Scope(value = "request", proxyMode = TARGET_CLASS)` | Proxy for injection |

### Profiles & Conditions

| Annotation | Purpose |
|-----------|---------|
| `@Profile("dev")` | Only in profile |
| `@Profile("!prod")` | NOT in profile |
| `@Profile({"dev", "test"})` | In any of |
| `@ConditionalOnClass` | If class present |
| `@ConditionalOnMissingBean` | If bean absent |
| `@ConditionalOnProperty` | If property set |
| `@ConditionalOnExpression` | SpEL condition |

### Bean Lifecycle Order (Memorize)

```
1. Instantiation (constructor)
2. Populate properties (DI)
3. BeanNameAware.setBeanName()
4. BeanFactoryAware.setBeanFactory()
5. ApplicationContextAware.setApplicationContext()
6. BeanPostProcessor.postProcessBeforeInitialization()
7. @PostConstruct
8. InitializingBean.afterPropertiesSet()
9. Custom init-method
10. BeanPostProcessor.postProcessAfterInitialization()   ← AOP proxies
11. ★ READY ★
12. @PreDestroy
13. DisposableBean.destroy()
14. Custom destroy-method
```

---

## 2. Spring Boot Annotations

| Annotation | Purpose |
|-----------|---------|
| `@SpringBootApplication` | Main application class |
| `@EnableAutoConfiguration` | Auto-config (included) |
| `@SpringBootConfiguration` | `@Configuration` alias |
| `@ConfigurationProperties(prefix = "app")` | Bind grouped properties |
| `@EnableConfigurationProperties` | Register `@ConfigurationProperties` beans |
| `@ConfigurationPropertiesScan` | Auto-scan `@ConfigurationProperties` |
| `@ConditionalOnClass` | Class-conditioned config |
| `@ConditionalOnMissingBean` | Missing-bean condition |
| `@ConditionalOnProperty` | Property condition |
| `@ConditionalOnWebApplication` | Web app condition |
| `@ConditionalOnResource` | Resource present |
| `@AutoConfigureBefore` / `@AutoConfigureAfter` | Order auto-config |
| `@SpringBootTest` | Full integration test |
| `@TestConfiguration` | Test-only config |
| `@MockBean` | Replace bean with mock |
| `@SpyBean` | Partial mock of bean |
| `@ActiveProfiles("test")` | Activate profiles in test |
| `@DynamicPropertySource` | Dynamic test properties |
| `@EnableAsync` | Enable `@Async` |
| `@EnableScheduling` | Enable `@Scheduled` |
| `@EnableCaching` | Enable caching |
| `@EnableTransactionManagement` | Enable `@Transactional` (auto in Boot) |
| `@EnableJpaAuditing` | Enable auditing |
| `@EnableJpaRepositories` | Enable Spring Data JPA |
| `@EnableWebSecurity` | Enable Spring Security |
| `@EnableMethodSecurity` | Enable `@PreAuthorize` etc. |
| `@EnableJms` | Enable JMS listeners |
| `@EnableRabbit` | Enable RabbitMQ listeners |
| `@EnableKafka` | Enable Kafka listeners |
| `@EnableBatchProcessing` | Enable Spring Batch (Boot 2; auto in Boot 3) |
| `@EnableRetry` | Enable `@Retryable` |
| `@EnableFeignClients` | Enable Feign clients |
| `@EnableDiscoveryClient` | Service discovery |

---

## 3. Spring MVC / REST Annotations

### Request Mapping

| Annotation | HTTP Method |
|-----------|-------------|
| `@RequestMapping` | Any |
| `@GetMapping` | GET |
| `@PostMapping` | POST |
| `@PutMapping` | PUT |
| `@PatchMapping` | PATCH |
| `@DeleteMapping` | DELETE |

### Request Binding

| Annotation | Binds |
|-----------|-------|
| `@PathVariable` | URI template variable |
| `@RequestParam` | Query parameter |
| `@RequestBody` | Request body (JSON/XML) |
| `@RequestHeader` | HTTP header |
| `@CookieValue` | Cookie |
| `@RequestPart` | Multipart part |
| `@ModelAttribute` | Form/query to object |
| `@Valid` | Trigger validation |
| `@Validated` | Method-level validation |
| `@NotNull` / `@NotBlank` / `@NotEmpty` | Validation |
| `@Size` / `@Min` / `@Max` / `@Pattern` / `@Email` | Validation |

### Response Handling

| Annotation / Type | Purpose |
|-------------------|---------|
| `@ResponseBody` | Serialize return value |
| `@ResponseStatus(HttpStatus.CREATED)` | Set status |
| `ResponseEntity<T>` | Full response control |
| `ProblemDetail` | RFC 7807 error |
| `@RestControllerAdvice` | Global exception handler |
| `@ControllerAdvice` | Global advice |
| `@ExceptionHandler` | Handle exception |
| `@InitBinder` | Custom data binding |
| `@CrossOrigin` | CORS for endpoint |

### MVC Config

| Interface | Purpose |
|-----------|---------|
| `WebMvcConfigurer` | Customize MVC |
| `HandlerInterceptor` | Intercept requests |
| `Filter` | Servlet filter |
| `ViewResolver` | Resolve views |

### Common Controller Snippet

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping
    public Page<UserResponse> list(@PageableDefault(size = 20) Pageable pageable) {
        return userService.findAll(pageable);
    }

    @GetMapping("/{id}")
    public UserResponse get(@PathVariable @Positive Long id) {
        return userService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<UserResponse> create(
            @Valid @RequestBody CreateUserRequest req,
            UriComponentsBuilder uri) {
        UserResponse user = userService.create(req);
        return ResponseEntity
            .created(uri.path("/api/v1/users/{id}").buildAndExpand(user.id()).toUri())
            .body(user);
    }

    @PutMapping("/{id}")
    public UserResponse update(@PathVariable Long id,
                               @Valid @RequestBody UpdateUserRequest req) {
        return userService.update(id, req);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @PreAuthorize("hasRole('ADMIN')")
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

---

## 4. Spring Data JPA

### Entity Annotations

| Annotation | Purpose |
|-----------|---------|
| `@Entity` | JPA entity |
| `@Table(name = "users", indexes = ...)` | Table mapping |
| `@Id` | Primary key |
| `@GeneratedValue(strategy = ...)` | ID generation |
| `@SequenceGenerator` | Sequence config |
| `@Column(name = "x", nullable = false, length = 100)` | Column mapping |
| `@Enumerated(EnumType.STRING)` | Enum as string |
| `@Temporal(TemporalType.TIMESTAMP)` | Date/time |
| `@Lob` | Large object (BLOB/CLOB) |
| `@Version` | Optimistic locking |
| `@Transient` | Not persisted |
| `@MappedSuperclass` | Inherit fields |
| `@Embeddable` / `@Embedded` | Value object |
| `@EntityListeners(AuditingEntityListener.class)` | Auditing |
| `@CreatedDate` / `@LastModifiedDate` | Auditing timestamps |
| `@CreatedBy` / `@LastModifiedBy` | Auditing users |

### Relationships

| Annotation | Relationship |
|-----------|-------------|
| `@OneToOne` | 1:1 |
| `@OneToMany(mappedBy = "x")` | 1:N (inverse) |
| `@ManyToOne(fetch = LAZY)` | N:1 (owning) |
| `@ManyToMany` | N:M |
| `@JoinColumn(name = "user_id")` | FK column |
| `@JoinTable(name = "...")` | Join table |
| `@OrderBy("name ASC")` | Collection order |
| `@MapKey` | Map key |
| `@Cascade(CascadeType.ALL)` or `cascade = ...` | Cascade |
| `orphanRemoval = true` | Remove orphans |

### Fetch Strategies

| Strategy | When |
|----------|------|
| `FetchType.LAZY` | Default preference |
| `FetchType.EAGER` | Rarely; explicit only |
| `@EntityGraph(attributePaths = {"items"})` | Fetch specific graph |
| `JOIN FETCH` (in JPQL) | Fetch in query |

### Repository Interfaces

| Interface | Provides |
|-----------|----------|
| `Repository<T, ID>` | Marker |
| `CrudRepository<T, ID>` | CRUD |
| `PagingAndSortingRepository<T, ID>` | + Pagination/Sort |
| `JpaRepository<T, ID>` | + JPA extras (`flush`, `saveAndFlush`) |

### Query Methods

```java
// Derived queries
List<User> findByName(String name);
List<User> findByAgeGreaterThan(int age);
List<User> findByNameContainingIgnoreCase(String name);
Optional<User> findByEmail(String email);
boolean existsByEmail(String email);
long countByActiveTrue();
void deleteByEmail(String email);

// With pagination
Page<User> findByActiveTrue(Pageable pageable);
Slice<User> findByAgeGreaterThan(int age, Pageable pageable);
List<User> findTop10ByOrderByCreatedAtDesc();

// JPQL
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmailJpql(@Param("email") String email);

@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findByIdWithOrders(@Param("id") Long id);

@Query("SELECT new com.example.UserDTO(u.id, u.name) FROM User u")
List<UserDTO> findAllAsDto();

// Native
@Query(value = "SELECT * FROM users WHERE email = ?1", nativeQuery = true)
Optional<User> findByEmailNative(String email);

// Modifying
@Modifying
@Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :cutoff")
int deactivateInactive(@Param("cutoff") Instant cutoff);

// Projections
interface UserSummary { Long getId(); String getName(); }
List<UserSummary> findByActiveTrue();

// Specifications
Specification<User> spec = (root, query, cb) -> cb.equal(root.get("active"), true);
List<User> users = userRepository.findAll(spec);
```

### Derived Query Keywords

| Keyword | Example |
|---------|---------|
| `And` / `Or` | `findByNameAndAge` |
| `Is` / `Equals` | `findByName` / `findByNameIs` |
| `Between` | `findByAgeBetween` |
| `LessThan` / `LessThanEqual` | `findByAgeLessThan` |
| `GreaterThan` / `GreaterThanEqual` | `findByAgeGreaterThan` |
| `Like` / `NotLike` | `findByNameLike` |
| `StartingWith` / `EndingWith` / `Containing` | `findByNameContaining` |
| `In` / `NotIn` | `findByStatusIn` |
| `IsNull` / `IsNotNull` | `findByEmailIsNull` |
| `True` / `False` | `findByActiveTrue` |
| `IgnoreCase` | `findByNameIgnoreCase` |
| `OrderBy...Asc` / `Desc` | `findByActiveOrderByCreatedAtDesc` |
| `First` / `Top` | `findTop5ByOrderByCreatedAtDesc` |
| `Distinct` | `findDistinctByStatus` |

---

## 5. Spring Security

### Core Annotations

| Annotation | Purpose |
|-----------|---------|
| `@EnableWebSecurity` | Enable web security |
| `@EnableMethodSecurity` | Enable `@PreAuthorize` etc. |
| `@PreAuthorize("...")` | Authorize before method |
| `@PostAuthorize("...")` | Authorize after method |
| `@PreFilter("...")` | Filter input collection |
| `@PostFilter("...")` | Filter output collection |
| `@Secured("ROLE_ADMIN")` | Legacy role check |
| `@RolesAllowed("ADMIN")` | JSR-250 |
| `@AuthenticationPrincipal` | Inject current user |
| `@CurrentSecurityContext` | Inject security context |

### SpEL Security Expressions

| Expression | Meaning |
|-----------|---------|
| `hasRole('ADMIN')` | Has ROLE_ADMIN |
| `hasAnyRole('ADMIN', 'USER')` | Any of |
| `hasAuthority('read:users')` | Has authority |
| `hasAnyAuthority('read', 'write')` | Any authority |
| `isAuthenticated()` | Logged in |
| `isAnonymous()` | Anonymous |
| `isRememberMe()` | Remember-me |
| `isFullyAuthenticated()` | Not remember-me |
| `permitAll()` | Allow all |
| `denyAll()` | Deny all |
| `#userId == authentication.principal.id` | SpEL with params |
| `#order.owner == authentication.name` | Object property |
| `returnObject.owner == authentication.name` | Return value |

### SecurityFilterChain Config

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())   // for stateless APIs
            .cors(Customizer.withDefaults())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/**").hasAuthority("read")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new BearerTokenAuthenticationEntryPoint())
                .accessDeniedHandler(new BearerTokenAccessDeniedHandler()))
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
                .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)));
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public UserDetailsService userDetailsService(UserRepository repo) {
        return username -> repo.findByUsername(username)
            .map(UserPrincipal::new)
            .orElseThrow(() -> new UsernameNotFoundException(username));
    }
}
```

### JWT Snippet

```java
// Generate
String token = Jwts.builder()
    .subject(user.getUsername())
    .claim("roles", user.getRoles())
    .issuedAt(new Date())
    .expiration(Date.from(Instant.now().plus(15, ChronoUnit.MINUTES)))
    .signWith(secretKey)
    .compact();

// Validate
Claims claims = Jwts.parser()
    .verifyWith(secretKey)
    .build()
    .parseSignedClaims(token)
    .getPayload();
```

---

## 6. Spring AOP

### Annotations

| Annotation | Purpose |
|-----------|---------|
| `@Aspect` | Declare aspect |
| `@Pointcut("execution(...)")` | Pointcut expression |
| `@Before("pointcut()")` | Before advice |
| `@After("pointcut()")` | After (finally) |
| `@AfterReturning(pointcut = "...", returning = "result")` | After return |
| `@AfterThrowing(pointcut = "...", throwing = "ex")` | After throw |
| `@Around("pointcut()")` | Around advice |
| `@Order(n)` | Aspect order |
| `@DeclareParents` | Introduction |
| `@EnableAspectJAutoProxy` | Enable AOP (auto in Boot) |

### Pointcut Designators

| Designator | Matches |
|-----------|---------|
| `execution(* com.example.service.*.*(..))` | Method execution |
| `within(com.example.service.*)` | Types in package |
| `this(com.example.Service)` | Proxy is type |
| `target(com.example.Service)` | Target is type |
| `args(String, ..)` | Argument types |
| `@annotation(org.springframework.transaction.annotation.Transactional)` | Method annotation |
| `@within(org.springframework.stereotype.Service)` | Type annotation |
| `@args(...)` | Argument annotation |
| `bean(userService)` | Bean name |

### Aspect Snippet

```java
@Aspect
@Component
@Order(1)
@Slf4j
public class LoggingAspect {

    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceMethods() { }

    @Around("serviceMethods()")
    public Object logExecutionTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long duration = System.nanoTime() - start;
            log.info("{} took {} ms", pjp.getSignature().toShortString(), duration / 1_000_000);
        }
    }
}
```

### Proxy Rules

| Target | Proxy Type |
|--------|-----------|
| Implements interface | JDK dynamic proxy |
| No interface | CGLIB (subclass) |
| `proxyTargetClass = true` | Force CGLIB |
| `final` class | ❌ Can't proxy |
| `final` method | ❌ Can't proxy |
| Private method | ❌ Can't proxy |
| Self-invocation | ❌ Bypasses proxy |

---

## 7. Spring Transaction

### Annotation

```java
@Transactional(
    propagation = Propagation.REQUIRED,      // default
    isolation = Isolation.DEFAULT,            // DB default
    timeout = 30,                             // seconds
    readOnly = false,
    rollbackFor = Exception.class,
    noRollbackFor = BusinessException.class
)
```

### Propagation

| Behavior | Meaning |
|----------|---------|
| `REQUIRED` (default) | Join existing or create new |
| `REQUIRES_NEW` | Always new (suspends outer) |
| `SUPPORTS` | Join if exists, else none |
| `NOT_SUPPORTED` | Suspend existing |
| `MANDATORY` | Must have existing |
| `NEVER` | Must NOT have existing |
| `NESTED` | Savepoint within existing |

### Isolation

| Level | Prevents |
|-------|----------|
| `READ_UNCOMMITTED` | Nothing |
| `READ_COMMITTED` | Dirty reads |
| `REPEATABLE_READ` | Dirty + non-repeatable |
| `SERIALIZABLE` | All (dirty, non-repeatable, phantom) |
| `DEFAULT` | DB default |

### Rollback Rules

| Exception Type | Rolls Back? |
|----------------|-------------|
| `RuntimeException` | ✅ Yes |
| `Error` | ✅ Yes |
| Checked `Exception` | ❌ No (default) |
| Checked + `rollbackFor` | ✅ Yes |
| RuntimeException + `noRollbackFor` | ❌ No |

### Common Snippet

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    @Transactional(readOnly = true)
    public OrderResponse findById(Long id) {
        return orderRepo.findById(id).map(this::toDto).orElseThrow();
    }

    @Transactional(rollbackFor = Exception.class)
    public OrderResponse create(CreateOrderRequest req) {
        Order order = orderRepo.save(new Order(req));
        paymentService.charge(order);   // in same TX
        return toDto(order);
    }
}
```

### Pitfalls

| Pitfall | Fix |
|---------|-----|
| Self-invocation | Extract to another bean |
| Private methods | Make public |
| Checked exceptions not rolling back | `rollbackFor = Exception.class` |
| Catching and swallowing | Rethrow or `setRollbackOnly()` |
| Long TX with I/O | Move I/O outside TX |
| `@Transactional` on controller | Move to service |

---

## 8. Spring Caching

### Annotations

| Annotation | Purpose |
|-----------|---------|
| `@EnableCaching` | Enable caching |
| `@Cacheable("cacheName")` | Cache method result |
| `@Cacheable(value = "users", key = "#id")` | Custom key |
| `@Cacheable(condition = "#id > 0")` | Conditional caching |
| `@Cacheable(unless = "#result == null")` | Skip caching null |
| `@CachePut("users")` | Always execute + update cache |
| `@CacheEvict("users")` | Remove from cache |
| `@CacheEvict(value = "users", allEntries = true)` | Clear cache |
| `@CacheEvict(beforeInvocation = true)` | Evict before method |
| `@Caching(...)` | Combine multiple |
| `@CacheConfig(cacheNames = "users")` | Class-level default |

### Snippet

```java
@Service
@RequiredArgsConstructor
public class UserService {

    @Cacheable(value = "users", key = "#id", unless = "#result == null")
    public UserResponse findById(Long id) {
        return userRepo.findById(id).map(this::toDto).orElse(null);
    }

    @CachePut(value = "users", key = "#result.id()")
    @Transactional
    public UserResponse create(CreateUserRequest req) {
        return toDto(userRepo.save(new User(req)));
    }

    @CacheEvict(value = "users", key = "#id")
    @Transactional
    public void delete(Long id) {
        userRepo.deleteById(id);
    }

    @CacheEvict(value = "users", allEntries = true)
    public void clearCache() { }
}
```

### Cache Providers

| Provider | When |
|----------|------|
| `ConcurrentMapCache` | Simple, in-memory, default |
| **Caffeine** | High-performance in-memory |
| **Redis** | Distributed cache |
| **Ehcache** | JVM + disk |
| **Hazelcast** | Distributed |

### Configuration

```properties
spring.cache.type=caffeine
spring.cache.caffeine.spec=maximumSize=10000,expireAfterWrite=10m

# or Redis
spring.cache.type=redis
spring.cache.redis.time-to-live=600000
spring.cache.redis.cache-null-values=false
```

---

## 9. Spring Batch

### Annotations

| Annotation | Purpose |
|-----------|---------|
| `@EnableBatchProcessing` | Enable (Boot 2; auto in Boot 3) |
| `@StepScope` | Per-step bean scope |
| `@JobScope` | Per-job bean scope |
| `@BeforeJob` / `@AfterJob` | Job listener |
| `@BeforeStep` / `@AfterStep` | Step listener |
| `@BeforeChunk` / `@AfterChunk` | Chunk listener |
| `@OnReadError` / `@OnProcessError` / `@OnWriteError` | Error listeners |
| `@OnSkipInRead` / `@OnSkipInProcess` / `@OnSkipInWrite` | Skip listeners |

### Job/Step Snippet

```java
@Configuration
@RequiredArgsConstructor
public class BatchConfig {

    @Bean
    public Job importUserJob(JobRepository repo,
                             Step importStep,
                             JobCompletionNotificationListener listener) {
        return new JobBuilder("importUserJob", repo)
            .listener(listener)
            .start(importStep)
            .build();
    }

    @Bean
    public Step importStep(JobRepository repo,
                           PlatformTransactionManager tx,
                           ItemReader<User> reader,
                           ItemProcessor<User, User> processor,
                           ItemWriter<User> writer) {
        return new StepBuilder("importStep", repo)
            .<User, User>chunk(100, tx)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .skip(FlatFileParseException.class)
            .skipLimit(50)
            .retry(TransientDataAccessException.class)
            .retryLimit(3)
            .build();
    }
}
```

### Launch

```java
JobParameters params = new JobParametersBuilder()
    .addString("inputFile", "input.csv")
    .addLong("timestamp", System.currentTimeMillis())
    .toJobParameters();

jobLauncher.run(myJob, params);
```

### Batch Tables

```
BATCH_JOB_INSTANCE
BATCH_JOB_EXECUTION
BATCH_JOB_EXECUTION_PARAMS
BATCH_JOB_EXECUTION_CONTEXT
BATCH_STEP_EXECUTION
BATCH_STEP_EXECUTION_CONTEXT
```

---

## 10. Spring Messaging

### JMS

| Annotation | Purpose |
|-----------|---------|
| `@EnableJms` | Enable JMS |
| `@JmsListener(destination = "queue")` | Consume from queue |
| `@JmsListener(destination = "topic", subscription = "sub1")` | Durable subscription |

```java
jmsTemplate.convertAndSend("queue", order);

@JmsListener(destination = "queue")
@SendTo("reply.queue")
public Receipt handle(Order order) { ... }
```

### RabbitMQ (AMQP)

| Annotation | Purpose |
|-----------|---------|
| `@EnableRabbit` | Enable Rabbit |
| `@RabbitListener(queues = "queue")` | Consume |
| `@RabbitListener(queues = "q", ackMode = "MANUAL")` | Manual ack |

```java
rabbitTemplate.convertAndSend("exchange", "routing.key", order);

@RabbitListener(queues = "queue")
public void handle(Order order) { ... }
```

**Exchange types:** `DirectExchange`, `TopicExchange`, `FanoutExchange`, `HeadersExchange`

### Kafka

| Annotation | Purpose |
|-----------|---------|
| `@EnableKafka` | Enable Kafka |
| `@KafkaListener(topics = "orders", groupId = "g1")` | Consume |
| `@KafkaListener(topics = "orders", groupId = "g1", concurrency = "3")` | Multi-thread |

```java
kafkaTemplate.send("orders", order.getId().toString(), order);

@KafkaListener(topics = "orders", groupId = "order-processors")
public void handle(Order order,
                   @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                   Acknowledgment ack) {
    process(order);
    ack.acknowledge();
}
```

### Spring Cloud Stream

```java
@Bean
public Supplier<Order> orderSupplier() {
    return () -> new Order(...);
}

@Bean
public Consumer<Order> processOrder() {
    return order -> System.out.println("Processing: " + order);
}

@Bean
public Function<Order, Receipt> generateReceipt() {
    return order -> new Receipt(order.getId());
}
```

```properties
spring.cloud.stream.bindings.processOrder-in-0.destination=orders
spring.cloud.stream.bindings.processOrder-in-0.group=order-processors
```

### DLQ Snippet

**RabbitMQ:**

```java
@Bean
public Queue orderQueue() {
    return QueueBuilder.durable("order.queue")
        .withArgument("x-dead-letter-exchange", "dlx")
        .withArgument("x-dead-letter-routing-key", "order.failed")
        .build();
}
```

**Kafka:**

```java
@Bean
public DefaultErrorHandler errorHandler(DeadLetterPublishingRecoverer recoverer) {
    return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3L));
}
```

---

## 11. Spring Cloud

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

### Annotations

| Annotation | Purpose |
|-----------|---------|
| `@EnableDiscoveryClient` | Register with registry |
| `@EnableEurekaClient` | Eureka client |
| `@EnableFeignClients` | Enable Feign |
| `@FeignClient("service-name")` | Declarative client |
| `@LoadBalanced` | Client-side load balancing |
| `@CircuitBreaker` | Circuit breaker |
| `@Retry` | Retry |
| `@Bulkhead` | Bulkhead |
| `@RateLimiter` | Rate limiting |
| `@RefreshScope` | Refreshable bean |

### Feign Client

```java
@FeignClient(name = "payment-service", fallback = PaymentFallback.class)
public interface PaymentClient {

    @PostMapping("/charge")
    PaymentResponse charge(@RequestBody PaymentRequest req);
}
```

### Circuit Breaker

```java
@CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
@Retry(name = "payment")
@TimeLimiter(name = "payment")
public CompletableFuture<PaymentResponse> charge(PaymentRequest req) { ... }
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
```

### Config Server

```properties
spring.config.import=configserver:http://config-server:8888
spring.cloud.config.name=myapp
spring.cloud.config.profile=prod
```

---

## 12. Spring WebFlux

### Types

| Type | Description |
|------|-------------|
| `Mono<T>` | 0 or 1 value |
| `Flux<T>` | 0..N values |

### Creating

| Method | Emits |
|--------|-------|
| `Mono.just(v)` | Single value |
| `Mono.empty()` | Empty |
| `Mono.error(e)` | Error |
| `Flux.just(a, b)` | Multiple |
| `Flux.fromIterable(list)` | From collection |
| `Flux.range(1, 10)` | Range |
| `Flux.interval(Duration)` | Periodic |

### Operators

| Operator | Purpose |
|----------|---------|
| `map` | Transform |
| `flatMap` | Async transform |
| `filter` | Filter |
| `concatMap` | Ordered flatMap |
| `zip` | Combine |
| `merge` | Interleave |
| `concat` | Sequential |
| `reduce` | Aggregate |
| `collect` | Collect to container |
| `delayElements` | Delay |
| `timeout` | Timeout |
| `retry` | Retry |
| `onErrorReturn` | Fallback value |
| `onErrorResume` | Fallback publisher |
| `doOnNext` | Side effect |
| `subscribeOn` | Subscription scheduler |
| `publishOn` | Downstream scheduler |

### Controller Snippet

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    @GetMapping
    public Flux<ProductDTO> list() {
        return productService.findAll();
    }

    @GetMapping("/{id}")
    public Mono<ProductDTO> get(@PathVariable Long id) {
        return productService.findById(id);
    }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ProductDTO> stream() {
        return productService.stream();
    }
}
```

### WebClient

```java
WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader("Authorization", "Bearer " + token)
    .build();

Mono<User> user = client.get()
    .uri("/users/{id}", id)
    .retrieve()
    .bodyToMono(User.class);
```

### Test

```java
StepVerifier.create(service.findById(1L))
    .expectNextMatches(dto -> dto.id().equals(1L))
    .verifyComplete();
```

---

## 13. Testing Annotations

### JUnit 5

| Annotation | Purpose |
|-----------|---------|
| `@Test` | Test method |
| `@BeforeEach` / `@AfterEach` | Per-test setup/teardown |
| `@BeforeAll` / `@AfterAll` | Once per class (static) |
| `@DisplayName("...")` | Human-readable name |
| `@Disabled` | Skip |
| `@Nested` | Nested test class |
| `@ParameterizedTest` | Parameterized |
| `@ValueSource` / `@CsvSource` / `@MethodSource` | Parameter sources |
| `@Tag("integration")` | Tag for filtering |
| `@Timeout(5)` | Timeout |

### Spring Boot Test

| Annotation | Purpose |
|-----------|---------|
| `@SpringBootTest` | Full context |
| `@SpringBootTest(webEnvironment = RANDOM_PORT)` | Real HTTP |
| `@WebMvcTest(Controller.class)` | MVC slice |
| `@DataJpaTest` | JPA slice |
| `@JdbcTest` | JDBC slice |
| `@JsonTest` | JSON serialization |
| `@RestClientTest` | REST client |
| `@Testcontainers` | Testcontainers |
| `@Container` | Container |
| `@DynamicPropertySource` | Dynamic props |
| `@ActiveProfiles("test")` | Profile |
| `@AutoConfigureMockMvc` | MockMvc |
| `@AutoConfigureTestDatabase(replace = NONE)` | Real DB |
| `@Transactional` | Rollback after test |
| `@Rollback` / `@Commit` | Control rollback |
| `@Sql("/test-data.sql")` | Execute SQL |
| `@MockBean` / `@SpyBean` | Replace bean |
| `@MockitoBean` / `@MockitoSpyBean` (Boot 3.4+) | New |
| `@TestConfiguration` | Test-only config |
| `@Import` | Import config |
| `@DirtiesContext` | Recreate context (avoid) |

### Mockito

| Annotation | Purpose |
|-----------|---------|
| `@Mock` | Mock object |
| `@Spy` | Partial mock |
| `@InjectMocks` | Inject mocks |
| `@Captor` | Argument captor |
| `@ExtendWith(MockitoExtension.class)` | Enable |

```java
when(repo.findById(1L)).thenReturn(Optional.of(user));
when(repo.save(any())).thenAnswer(i -> i.getArgument(0));
doThrow(new RuntimeException()).when(service).process();
verify(repo).save(any());
```

### Testcontainers

```java
@Testcontainers
@SpringBootTest
class IntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }
}
```

### AssertJ

```java
assertThat(users).hasSize(3);
assertThat(users).extracting(User::getName)
    .containsExactly("Alice", "Bob", "Charlie");
assertThatThrownBy(() -> service.findById(99L))
    .isInstanceOf(ResourceNotFoundException.class)
    .hasMessageContaining("99");
assertThat(actual).isEqualTo(expected);
assertThat(list).isNotEmpty().contains(item);
assertThat(number).isGreaterThan(0).isLessThan(100);
```

---

## 14. Configuration Properties

### application.properties

```properties
spring.application.name=myapp
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/db
spring.datasource.username=user
spring.datasource.password=secret
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.open-in-view=false
```

### application.yml

```yaml
spring:
  application:
    name: myapp
  datasource:
    url: jdbc:postgresql://localhost:5432/db
    username: user
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true

server:
  port: 8080
  shutdown: graceful
```

### Key Properties

| Property | Purpose |
|----------|---------|
| `spring.profiles.active` | Active profiles |
| `spring.config.import` | Import config |
| `spring.datasource.*` | DataSource |
| `spring.jpa.*` | JPA/Hibernate |
| `spring.flyway.*` | Flyway |
| `spring.liquibase.*` | Liquibase |
| `spring.redis.*` / `spring.data.redis.*` | Redis |
| `spring.kafka.*` | Kafka |
| `spring.rabbitmq.*` | RabbitMQ |
| `spring.security.oauth2.*` | OAuth2 |
| `management.endpoints.web.exposure.include` | Actuator endpoints |
| `server.port` | HTTP port |
| `server.compression.enabled` | Gzip |
| `server.tomcat.threads.max` | Tomcat threads |
| `logging.level.*` | Log levels |
| `spring.threads.virtual.enabled` | Virtual threads |
| `spring.jackson.*` | Jackson config |

### Profiles

```properties
# application-dev.properties
spring.datasource.url=jdbc:h2:mem:devdb

# application-prod.properties
spring.datasource.url=jdbc:postgresql://prod-db:5432/app
```

### Property Precedence (High → Low)

```
1. Devtools home properties
2. @TestPropertySource
3. Command-line args
4. SPRING_APPLICATION_JSON
5. ServletConfig init params
6. ServletContext init params
7. JNDI (java:comp/env)
8. Java system properties (-D)
9. OS environment variables
10. application-{profile}.properties outside JAR
11. application-{profile}.properties inside JAR
12. application.properties outside JAR
13. application.properties inside JAR
14. @PropertySource
15. Defaults
```

### Relaxed Binding Examples

```
myapp.max-size       → maxSize
myapp.maxSize        → maxSize
myapp.max_size       → maxSize
MYAPP_MAX_SIZE       → maxSize (env var)
```

### @ConfigurationProperties

```java
@ConfigurationProperties(prefix = "app.mail")
@Validated
public record MailProperties(
    @NotBlank String host,
    @Min(1) @Max(65535) int port,
    @Email String from,
    Duration timeout,
    Retry retry
) {
    public record Retry(int maxAttempts, Duration backoff) { }
}
```

---

## 15. Maven Cheat Sheet

### Common Commands

| Command | Purpose |
|---------|---------|
| `mvn clean` | Delete `target/` |
| `mvn compile` | Compile sources |
| `mvn test` | Run tests |
| `mvn package` | Build JAR/WAR |
| `mvn install` | Install to local repo |
| `mvn deploy` | Deploy to remote |
| `mvn verify` | Run integration tests |
| `mvn clean install` | Clean + install |
| `mvn clean package -DskipTests` | Skip tests |
| `mvn -pl module -am install` | Build module + deps |
| `mvn dependency:tree` | Show dependency tree |
| `mvn dependency:tree -Dincludes=groupId:artifactId` | Filter |
| `mvn versions:display-dependency-updates` | Check updates |
| `mvn help:effective-pom` | Show effective POM |
| `mvn spring-boot:run` | Run Spring Boot |
| `mvn spring-boot:build-image` | Build OCI image |
| `mvn -o` | Offline |
| `mvn -X` | Debug output |
| `mvn -T 4` | Parallel (4 threads) |
| `mvn -PprofileName` | Activate profile |

### POM Structure

```xml
<project>
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>myapp</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### Lifecycle Phases (Default)

```
validate
initialize
generate-sources
process-sources
generate-resources
process-resources
compile
process-classes
generate-test-sources
process-test-sources
generate-test-resources
process-test-resources
test-compile
process-test-classes
test
prepare-package
package
pre-integration-test
integration-test
post-integration-test
verify
install
deploy
```

### Dependency Scopes

| Scope | Compile | Test | Runtime | Packaged |
|-------|---------|------|---------|----------|
| `compile` (default) | ✅ | ✅ | ✅ | ✅ |
| `provided` | ✅ | ✅ | ❌ | ❌ |
| `runtime` | ❌ | ✅ | ✅ | ✅ |
| `test` | ❌ | ✅ | ❌ | ❌ |
| `system` | ✅ | ✅ | ❌ | ❌ |
| `import` | (BOM only) | | | |

### BOM Import

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.3.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Profiles

```xml
<profiles>
    <profile>
        <id>prod</id>
        <activation>
            <property><name>env</name><value>prod</value></property>
        </activation>
        <properties>
            <spring.profiles.active>prod</spring.profiles.active>
        </properties>
    </profile>
</profiles>
```

---

## 16. Gradle Cheat Sheet

### Common Commands

| Command | Purpose |
|---------|---------|
| `./gradlew build` | Build |
| `./gradlew test` | Run tests |
| `./gradlew bootRun` | Run Spring Boot |
| `./gradlew bootJar` | Build executable JAR |
| `./gradlew clean` | Clean |
| `./gradlew dependencies` | Show dependencies |
| `./gradlew tasks` | List tasks |
| `./gradlew --refresh-dependencies` | Refresh |
| `./gradlew -Pprofile=prod build` | With property |

### build.gradle

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.5'
}

group = 'com.example'
version = '1.0.0'
java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }

repositories { mavenCentral() }

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
}

tasks.named('test') { useJUnitPlatform() }
```

### Dependency Configurations

| Configuration | Purpose |
|--------------|---------|
| `implementation` | Internal deps (not exposed) |
| `api` | Exposed deps (for libraries) |
| `compileOnly` | Compile-only (Lombok) |
| `runtimeOnly` | Runtime-only (JDBC drivers) |
| `testImplementation` | Test deps |
| `testRuntimeOnly` | Test runtime |
| `annotationProcessor` | Annotation processors |

### Kotlin DSL

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.5"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

## 17. Spring Boot Actuator Endpoints

### Default Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/actuator` | Discovery |
| `/actuator/health` | Health check |
| `/actuator/health/liveness` | Liveness probe |
| `/actuator/health/readiness` | Readiness probe |
| `/actuator/info` | App info |
| `/actuator/metrics` | Micrometer metrics |
| `/actuator/metrics/{name}` | Specific metric |
| `/actuator/prometheus` | Prometheus format |
| `/actuator/env` | Environment |
| `/actuator/beans` | Spring beans |
| `/actuator/mappings` | Request mappings |
| `/actuator/loggers` | Log levels |
| `/actuator/threaddump` | Thread dump |
| `/actuator/heapdump` | Heap dump |
| `/actuator/httpexchanges` | HTTP traces |
| `/actuator/startup` | Startup steps (Boot 3.1+) |
| `/actuator/scheduledtasks` | Scheduled tasks |
| `/actuator/refresh` | Refresh config (Cloud) |
| `/actuator/busrefresh` | Bus refresh |
| `/actuator/shutdown` | Shutdown (POST) |

### Configuration

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoints.web.exposure.exclude=env,beans
management.endpoint.health.show-details=when-authorized
management.endpoint.health.probes.enabled=true
management.metrics.export.prometheus.enabled=true
management.tracing.sampling.probability=1.0
```

### Security

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/actuator/health").permitAll()
    .requestMatchers("/actuator/**").hasRole("ADMIN")
    .anyRequest().authenticated());
```

### Custom Health Indicator

```java
@Component
public class PaymentHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        try {
            paymentClient.ping();
            return Health.up().build();
        } catch (Exception e) {
            return Health.down().withDetail("error", e.getMessage()).build();
        }
    }
}
```

### Custom Endpoint

```java
@Component
@Endpoint(id = "features")
public class FeaturesEndpoint {

    @ReadOperation
    public Map<String, Boolean> features() {
        return Map.of("experimental", true);
    }

    @WriteOperation
    public void toggle(@Selector String name, boolean enabled) { }
}
```

---

## 18. Common Patterns & Snippets

### Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ProblemDetail> handle(ResourceNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Not Found");
        return ResponseEntity.status(404).body(pd);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ProblemDetail> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed");
        pd.setProperty("errors", errors);
        return ResponseEntity.badRequest().body(pd);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleAll(Exception ex) {
        log.error("Unhandled", ex);
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR, "Unexpected error");
        return ResponseEntity.internalServerError().body(pd);
    }
}
```

### Correlation ID Filter

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String HEADER = "X-Correlation-Id";

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String id = req.getHeader(HEADER);
        if (id == null) id = UUID.randomUUID().toString();
        MDC.put("correlationId", id);
        res.setHeader(HEADER, id);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.clear();
        }
    }
}
```

### JWT Authentication Filter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String header = req.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(req, res);
            return;
        }
        try {
            String token = header.substring(7);
            String username = jwtService.extractUsername(token);
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails user = userDetailsService.loadUserByUsername(username);
                if (jwtService.isValid(token, user)) {
                    var auth = new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        } catch (JwtException e) {
            // invalid token — proceed unauthenticated
        }
        chain.doFilter(req, res);
    }
}
```

### Auditable Entity

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter
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

### DTO + MapStruct

```java
public record UserResponse(Long id, String name, String email) { }

@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface UserMapper {

    UserResponse toResponse(User user);

    @Mapping(target = "id", ignore = true)
    User toEntity(CreateUserRequest req);
}
```

### Validation

```java
public record CreateUserRequest(
    @NotBlank @Size(min = 2, max = 100) String name,
    @NotBlank @Email String email,
    @Min(18) @Max(120) Integer age
) { }
```

### Rate Limiting (Bucket4j)

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
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String key = req.getHeader("X-API-Key");
        if (key == null) key = req.getRemoteAddr();
        Bucket bucket = buckets.computeIfAbsent(key, k -> newBucket());
        if (bucket.tryConsume(1)) {
            chain.doFilter(req, res);
            return;
        }
        res.setStatus(429);
        res.setHeader("Retry-After", "60");
    }
}
```

### Async Method

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
        exec.initialize();
        return exec;
    }
}

@Service
public class NotificationService {

    @Async
    public CompletableFuture<Void> sendEmail(String to, String body) { ... }
}
```

### Scheduled Task

```java
@Component
@EnableScheduling
public class ScheduledTasks {

    @Scheduled(cron = "0 0 2 * * ?")     // 2 AM daily
    public void cleanup() { }

    @Scheduled(fixedDelay = 60_000)       // 60s after completion
    public void poll() { }

    @Scheduled(fixedRate = 30_000)        // every 30s
    public void heartbeat() { }
}
```

### Caching

```java
@Service
public class UserService {

    @Cacheable(value = "users", key = "#id", unless = "#result == null")
    public UserResponse findById(Long id) { ... }

    @CachePut(value = "users", key = "#result.id()")
    public UserResponse create(CreateUserRequest req) { ... }

    @CacheEvict(value = "users", key = "#id")
    public void delete(Long id) { }
}
```

---

## 19. Docker & Kubernetes Snippets

### Multi-Stage Dockerfile

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Runtime stage
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=build /build/target/*.jar app.jar
USER app
EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "/app.jar"]
```

### Layered JAR Dockerfile

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

### docker-compose.yml

```yaml
version: '3.9'
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/appdb
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
        - name: myapp
          image: registry.example.com/myapp:1.0.0
          ports: [{ containerPort: 8080 }]
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
          resources:
            requests: { memory: "512Mi", cpu: "250m" }
            limits: { memory: "1Gi", cpu: "1000m" }
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 60
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 30
```

### Kubernetes Service + Ingress

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector: { app: myapp }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
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
              service: { name: myapp, port: { number: 80 } }
```

---

## 20. Interview Rapid-Fire Answers

### Core Spring

| Question | Answer |
|----------|--------|
| What is IoC? | Container manages object creation/lifecycle; you declare dependencies |
| What is DI? | Container injects dependencies; constructor/setter/field |
| Preferred injection? | Constructor (immutable, testable, fail-fast) |
| BeanFactory vs ApplicationContext? | Basic lazy container vs advanced eager container with events, i18n, AOP |
| Default scope? | Singleton |
| Prototype lifecycle? | Spring doesn't call destroy callbacks |
| @Component vs @Service? | Semantically different; both are @Component |
| @Configuration CGLIB? | Inter-bean method calls return singletons |
| Circular dependency? | Three-level cache for setter; constructor fails |
| @PostConstruct order? | After DI, before afterPropertiesSet |

### AOP

| Question | Answer |
|----------|--------|
| What is AOP? | Cross-cutting concerns via proxies |
| Proxy types? | JDK (interface) vs CGLIB (subclass) |
| Around advice? | Wraps method; controls proceed() |
| Self-invocation? | Bypasses proxy — no advice applied |
| @Order? | Lower value = higher precedence |

### MVC / REST

| Question | Answer |
|----------|--------|
| DispatcherServlet? | Front controller |
| @Controller vs @RestController? | `@ResponseBody` on every method |
| @RequestParam vs @PathVariable? | Query vs URI template |
| @ControllerAdvice? | Global exception/advice |
| Idempotent methods? | GET, PUT, DELETE, HEAD |
| 401 vs 403? | Unauthenticated vs unauthorized |
| 201 with Location? | Created resource |

### Boot

| Question | Answer |
|----------|--------|
| @SpringBootApplication? | @Configuration + @EnableAutoConfiguration + @ComponentScan |
| Auto-config? | Conditional configs from `AutoConfiguration.imports` |
| @ConditionalOnMissingBean? | Only register if bean absent |
| Starters? | Curated dependency sets |
| Actuator? | Production endpoints (health, metrics) |
| @ConfigurationProperties? | Grouped, type-safe config binding |
| Property precedence? | CLI > env > profile props > base props |

### JPA

| Question | Answer |
|----------|--------|
| N+1? | 1 query for parents + N for children; fix with JOIN FETCH |
| LAZY vs EAGER? | Load on access vs immediately |
| Persistence context? | First-level cache + dirty checking |
| @Version? | Optimistic locking |
| save() vs persist()? | save uses persist (no ID) or merge |
| open-in-view? | Should be false in production |
| Projections? | Interface-based or DTO-class |
| @EntityGraph? | Declare fetch plan per query |
| Entity lifecycle? | Transient, Managed, Detached, Removed |

### Security

| Question | Answer |
|----------|--------|
| Authentication vs authorization? | Who vs what |
| BCrypt? | Adaptive hash; slow by design |
| CSRF? | Token for stateful; disable for stateless |
| JWT structure? | header.payload.signature |
| Access vs refresh? | Short-lived access + long-lived refresh |
| @PreAuthorize? | SpEL authorization before method |
| CORS? | Cross-origin request policy |
| HSTS? | Force HTTPS |

### Transactions

| Question | Answer |
|----------|--------|
| Default propagation? | REQUIRED |
| REQUIRES_NEW? | Always new, suspends outer |
| Rollback by default? | RuntimeException and Error only |
| Self-invocation? | Bypasses proxy — no TX |
| readOnly? | Optimization hint; use for reads |
| Isolation default? | DB default (usually READ_COMMITTED) |

### Microservices

| Question | Answer |
|----------|--------|
| Service discovery? | Eureka, Consul, K8s DNS |
| Circuit breaker? | Fail fast when downstream down |
| API Gateway? | Single entry, routes to services |
| Distributed tracing? | Micrometer + Zipkin/Jaeger |
| Saga? | Distributed TX with compensations |
| Outbox? | DB + broker atomicity pattern |

### Testing

| Question | Answer |
|----------|--------|
| @SpringBootTest? | Full context |
| @WebMvcTest? | MVC slice |
| @DataJpaTest? | JPA slice with rollback |
| Testcontainers? | Real DB in Docker for tests |
| @MockBean vs @Mock? | Spring context replacement vs plain Mockito |
| Coverage target? | 70–80% (behavior, not lines) |

### Maven

| Question | Answer |
|----------|--------|
| Lifecycle phases? | validate → compile → test → package → install → deploy |
| Scopes? | compile, provided, runtime, test, system, import |
| BOM? | Import versions without parent |
| `mvn install` vs `deploy`? | Local repo vs remote |
| Dependency conflict? | Nearest wins |
| Multi-module? | Aggregator POM + modules |

### Performance

| Question | Answer |
|----------|--------|
| N+1? | JOIN FETCH / @EntityGraph |
| Pagination? | Pageable + Page<T> |
| HikariCP sizing? | (cores * 2) + spindles; 10–20 typical |
| JVM container? | -XX:MaxRAMPercentage=75 |
| Virtual threads? | spring.threads.virtual.enabled=true |
| Caching? | @Cacheable for hot, immutable data |
| Async? | @Async with TaskExecutor |
| Profiling? | JFR, async-profiler |

### Security Headers

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
Permissions-Policy: geolocation=(), camera=()
```

### HTTP Status Codes

```
200 OK          | 201 Created      | 202 Accepted    | 204 No Content
301 Moved       | 304 Not Modified
400 Bad Request | 401 Unauthorized | 403 Forbidden   | 404 Not Found
405 Not Allowed | 409 Conflict     | 415 Unsupported | 422 Unprocessable
429 Too Many
500 Internal    | 502 Bad Gateway  | 503 Unavailable | 504 Timeout
```

### Maven Quick Commands

```bash
mvn clean install                    # full build
mvn clean package -DskipTests        # skip tests
mvn test                             # run tests
mvn spring-boot:run                  # run app
mvn dependency:tree                  # inspect deps
mvn versions:display-dependency-updates
mvn -pl module -am install           # module + deps
```

### Key Properties

```properties
# Server
server.port=8080
server.shutdown=graceful

# JPA
spring.jpa.open-in-view=false
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.properties.hibernate.jdbc.batch_size=50

# Actuator
management.endpoints.web.exposure.include=health,info,metrics,prometheus

# Virtual threads
spring.threads.virtual.enabled=true

# Logging
logging.level.com.example=DEBUG
```

### Annotation Frequency (Top 20)

1. `@Autowired`
2. `@Component` / `@Service` / `@Repository` / `@Controller`
3. `@Configuration` / `@Bean`
4. `@SpringBootApplication`
5. `@Transactional`
6. `@RestController` / `@RequestMapping` / `@GetMapping`
7. `@RequestBody` / `@PathVariable` / `@RequestParam`
8. `@Valid`
9. `@Entity` / `@Id` / `@GeneratedValue`
10. `@PreAuthorize`
11. `@Value` / `@ConfigurationProperties`
12. `@Profile`
13. `@PostConstruct` / `@PreDestroy`
14. `@Qualifier` / `@Primary`
15. `@Cacheable` / `@CacheEvict`
16. `@Async` / `@Scheduled`
17. `@ControllerAdvice` / `@ExceptionHandler`
18. `@Test` / `@SpringBootTest` / `@MockBean`
19. `@Query`
20. `@Slf4j`

---

## Final Words

You now have the **complete Spring Ecosystem Study Guide** — 27 files covering:

- **Foundations** (01–05): Core, AOP, MVC, JDBC, Transactions
- **Spring Boot** (06–09): Fundamentals, Starters, Config, Actuator
- **Data** (10–12): JPA, Spring Data JDBC, Caching
- **Security** (13–15): Core, JWT/OAuth2, Method-Level
- **Testing** (16): Slices, Mockito, Testcontainers
- **Build** (17–18): Maven, Gradle
- **Advanced** (19–23): Microservices, WebFlux, Batch, Messaging, Web Services
- **Interview Prep** (24–27): Questions, Architecture, Best Practices, Cheat Sheet

### Suggested Study Path

**Weeks 1–4:** Foundations (01–05)
**Weeks 5–7:** Spring Boot (06–09)
**Weeks 8–11:** Data & Security (10–15)
**Weeks 12–13:** Testing & Build (16–18)
**Weeks 14–18:** Advanced (19–23)
**Weeks 19–20:** Interview Prep (24–27)

### Practice Advice

1. **Code every day** — build a small project using each concept
2. **Answer aloud** — simulate interview conditions
3. **Read source code** — Spring is open source
4. **Write tests** — they cement understanding
5. **Deploy something** — Docker + K8s + CI/CD
6. **Review your code** against the Best Practices file

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md)
- **Next →:** —
- **Related:** [24_Spring_Interview_Questions.md](./24_Spring_Interview_Questions.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md), [01_Spring_Framework_Core.md](./01_Spring_Framework_Core.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
