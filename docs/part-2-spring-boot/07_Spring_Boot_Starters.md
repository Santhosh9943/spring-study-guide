# Spring Boot Starters Deep Dive

> **File:** `07_Spring_Boot_Starters.md`
> **Part:** 2 — Spring Boot
> **Prerequisites:** `06_Spring_Boot_Fundamentals.md`
> **Estimated Study Time:** 6–8 hours

---

## Table of Contents

1. [What is a Starter?](#1-what-is-a-starter)
2. [How Starters Work](#2-how-starters-work)
3. [spring-boot-starter (Core)](#3-spring-boot-starter-core)
4. [spring-boot-starter-web](#4-spring-boot-starter-web)
5. [spring-boot-starter-data-jpa](#5-spring-boot-starter-data-jpa)
6. [spring-boot-starter-data-jdbc](#6-spring-boot-starter-data-jdbc)
7. [spring-boot-starter-data-mongodb](#7-spring-boot-starter-data-mongodb)
8. [spring-boot-starter-data-redis](#8-spring-boot-starter-data-redis)
9. [spring-boot-starter-security](#9-spring-boot-starter-security)
10. [spring-boot-starter-test](#10-spring-boot-starter-test)
11. [spring-boot-starter-actuator](#11-spring-boot-starter-actuator)
12. [spring-boot-starter-validation](#12-spring-boot-starter-validation)
13. [spring-boot-starter-aop](#13-spring-boot-starter-aop)
14. [spring-boot-starter-thymeleaf](#14-spring-boot-starter-thymeleaf)
15. [spring-boot-starter-webflux](#15-spring-boot-starter-webflux)
16. [spring-boot-starter-batch](#16-spring-boot-starter-batch)
17. [spring-boot-starter-amqp (RabbitMQ)](#17-spring-boot-starter-amqp-rabbitmq)
18. [spring-boot-starter-kafka](#18-spring-boot-starter-kafka)
19. [Creating Custom Starters](#19-creating-custom-starters)
20. [Excluding Auto-Configuration](#20-excluding-auto-configuration)
21. [Interview Questions](#21-interview-questions)
22. [Cheat Sheet](#22-cheat-sheet)

---

## 1. What is a Starter?

A **starter** is a Maven/Gradle dependency descriptor (a POM that contains **no code**) that **aggregates** the libraries needed for a feature and triggers the matching auto-configuration.

> **Spring docs:** "Starters are a set of convenient dependency descriptors that you can include in your application. You get a one-stop shop for all the Spring and related technologies that you need, without having to hunt through sample code and copy-paste loads of dependency descriptors."

### Example

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

This single dependency pulls in:
- `spring-web`, `spring-webmvc`
- `spring-boot-starter` (core)
- `spring-boot-starter-json` (Jackson)
- `spring-boot-starter-tomcat` (embedded server)
- `spring-boot-starter-validation` (from Boot 3.4+)
- Logging (`spring-boot-starter-logging` → Logback + SLF4J)

### Naming Convention

| Prefix | Meaning |
|--------|---------|
| `spring-boot-starter-*` | Official Spring Boot starters |
| `*-spring-boot-starter` | Third-party starters (e.g., `mybatis-spring-boot-starter`) |
| `spring-boot-starter` | Core starter (no suffix) |

**Rule:** Official starts with `spring-boot-starter-`; third-party ends with `-spring-boot-starter`.

---

## 2. How Starters Work

Starters combine **three mechanisms**:

```
┌──────────────────────────────────────────────┐
│  1. Dependency Aggregation (POM)             │
│     — you add ONE starter → all libs arrive  │
├──────────────────────────────────────────────┤
│  2. Auto-Configuration (conditional beans)   │
│     — beans defined based on classpath       │
├──────────────────────────────────────────────┤
│  3. Property Externalization                 │
│     — @ConfigurationProperties for tunables  │
└──────────────────────────────────────────────┘
```

### Flow Diagram

```
Add starter to pom.xml
        │
        ▼
Transitive deps downloaded
        │
        ▼
Spring Boot reads AutoConfiguration.imports
        │
        ▼
Evaluates @ConditionalOnClass / @ConditionalOnMissingBean
        │
        ▼
Creates beans using @ConfigurationProperties
        │
        ▼
Your app has fully working feature with sensible defaults
```

### Overriding Defaults

Every auto-config bean is `@ConditionalOnMissingBean`. Define your own bean → auto-config backs off.

```java
// Starter auto-configures a DataSource
// You override:
@Bean
public DataSource dataSource() {
    return myCustomDataSource();
}
```

Or via properties:

```properties
spring.datasource.url=jdbc:postgresql://...
```

---

## 3. spring-boot-starter (Core)

**The base starter** every other starter depends on (transitively).

### Dependencies Pulled In

| Dependency | Purpose |
|-----------|---------|
| `spring-boot` | Core Boot runtime |
| `spring-boot-autoconfigure` | Auto-config engine |
| `spring-core` | Spring core |
| `spring-context` | IoC container |
| `spring-aop` | AOP support |
| `spring-beans` | Bean factory |
| `spring-expression` | SpEL |
| `spring-boot-starter-logging` | Logback + SLF4J + Log4j-to-SLF4J |
| `jakarta.annotation-api` | `@PostConstruct`, `@PreDestroy` |
| `snakeyaml` | YAML parser |
| `spring-boot-starter-logging` | Logging |

### Auto-Configuration

- `ApplicationContext` setup
- `Environment` (properties, profiles)
- Logging system
- Banner
- Task execution/scheduling
- Validation (Jakarta Bean Validation)
- `MessageSource` (i18n)
- `ApplicationArguments`

### When to Use

**Rarely** added directly. Included transitively by other starters. Only add if you want the absolute minimum Spring Boot runtime (no web, no data, no security).

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
```

### Typical Use Case

- CLI tools
- Background workers with no web
- Library projects

---

## 4. spring-boot-starter-web

The most-used starter. Traditional Servlet-based web apps and REST APIs.

### Dependencies Pulled In

| Dependency | Purpose |
|-----------|---------|
| `spring-boot-starter` | Core Boot |
| `spring-boot-starter-json` | Jackson JSON |
| `spring-boot-starter-tomcat` | Embedded Tomcat |
| `spring-boot-starter-validation` | Bean Validation (Boot 3.4+) |
| `spring-web` | HTTP abstractions |
| `spring-webmvc` | MVC framework |

### Auto-Configuration Classes

- `DispatcherServletAutoConfiguration`
- `ServletWebServerFactoryAutoConfiguration` (Tomcat)
- `WebMvcAutoConfiguration`
- `JacksonAutoConfiguration`
- `HttpMessageConvertersAutoConfiguration`
- `ErrorMvcAutoConfiguration` (default error page)
- `MultipartAutoConfiguration` (file upload)
- `ValidationAutoConfiguration`
- `ServletWebServerFactoryAutoConfiguration`

### Minimal Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping
    public List<String> list() { return List.of("Alice", "Bob"); }
}
```

### Common Properties

```properties
server.port=8080
server.servlet.context-path=/
server.tomcat.max-threads=200
spring.mvc.servlet.path=/
spring.mvc.throw-exception-if-no-handler-found=true
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=50MB
spring.jackson.default-property-inclusion=non_null
spring.jackson.serialization.indent-output=true
```

### When NOT to Use

- **Reactive apps** → use `spring-boot-starter-webflux`
- **Only need HTTP client** → use `spring-web` directly
- **Serverless** → consider `spring-cloud-function-web`

### Switching to Jetty

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

---

## 5. spring-boot-starter-data-jpa

JPA + Hibernate + Spring Data JPA + HikariCP.

### Dependencies Pulled In

| Dependency | Purpose |
|-----------|---------|
| `spring-boot-starter-aop` | AOP (for @Transactional) |
| `spring-boot-starter-jdbc` | JDBC + HikariCP + tx |
| `hibernate-core` | JPA provider |
| `spring-data-jpa` | Repository abstraction |
| `jakarta.persistence-api` | JPA API |
| `jakarta.transaction-api` | JTA API |
| `spring-aspects` | AspectJ load-time weaving |

### Auto-Configuration

- `DataSourceAutoConfiguration`
- `HibernateJpaAutoConfiguration`
- `JpaRepositoriesAutoConfiguration`
- `DataSourceTransactionManagerAutoConfiguration`
- `TransactionAutoConfiguration`

### Minimal Entity + Repository

```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
    // getters/setters
}

public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmailEndingWith(String domain);
}
```

### Common Properties

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
spring.datasource.password=secret
spring.datasource.hikari.maximum-pool-size=20

spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.open-in-view=false

spring.data.jpa.repositories.bootstrap-mode=default
```

### DDL Auto Modes

| Mode | Behavior |
|------|----------|
| `none` | No action |
| `validate` | Verify schema matches (recommended for prod) |
| `update` | Update schema (dev only) |
| `create` | Drop + create on startup |
| `create-drop` | Create, drop on shutdown |

⚠️ Never use `update`/`create` in production. Use Flyway or Liquibase.

### When to Use

- Domain-model-heavy apps
- Standard CRUD with relationships
- Rich entity graphs

### When NOT to Use

- Purely tabular data → `spring-boot-starter-data-jdbc`
- Complex SQL reports → `spring-boot-starter-jdbc` (JdbcTemplate)
- NoSQL → MongoDB/Cassandra starters

---

## 6. spring-boot-starter-data-jdbc

Spring Data JDBC — simpler, no lazy loading, aggregates.

### Dependencies Pulled In

- `spring-boot-starter-jdbc`
- `spring-data-jdbc`
- `spring-data-relational`

### Features

- Aggregate-oriented
- No lazy loading (avoids N+1)
- Explicit SQL for complex queries
- Simpler mental model than JPA

### Example

```java
@Table("users")
public class User {
    @Id
    private Long id;
    private String name;
    @MappedCollection(idColumn = "user_id")
    private Set<Order> orders = new HashSet<>();
}

public interface UserRepository extends CrudRepository<User, Long> { }
```

### Configuration

```properties
spring.datasource.url=jdbc:postgresql://...
```

### When to Use

- Simple aggregates
- You want explicit SQL
- Avoiding JPA complexity
- DDD-style aggregates

### When NOT to Use

- Complex relations with lazy loading needed → JPA
- Reporting → plain JdbcTemplate

---

## 7. spring-boot-starter-data-mongodb

MongoDB integration.

### Dependencies Pulled In

- `spring-data-mongodb`
- `mongodb-driver-sync`

### Example

```java
@Document(collection = "users")
public class User {
    @Id private String id;
    private String name;
    private List<String> tags;
}

public interface UserRepository extends MongoRepository<User, String> {
    List<User> findByTagsContaining(String tag);
}
```

### Configuration

```properties
spring.data.mongodb.uri=mongodb://localhost:27017/mydb
# or
spring.data.mongodb.host=localhost
spring.data.mongodb.port=27017
spring.data.mongodb.database=mydb
spring.data.mongodb.username=admin
spring.data.mongodb.password=secret
```

### Reactive Variant

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb-reactive</artifactId>
</dependency>
```

Use `ReactiveMongoRepository` + `ReactiveMongoTemplate` + `Mono`/`Flux`.

### When to Use

- Document-oriented data
- Schemaless JSON
- Hierarchical data
- Horizontal scaling

---

## 8. spring-boot-starter-data-redis

Redis integration with Spring Data Redis + Lettuce.

### Dependencies Pulled In

- `spring-data-redis`
- `lettuce-core` (default client)

### RedisTemplate / StringRedisTemplate

```java
@Service
public class SessionService {
    private final StringRedisTemplate redis;
    
    public SessionService(StringRedisTemplate redis) { this.redis = redis; }
    
    public void put(String key, String value) { redis.opsForValue().set(key, value); }
    public String get(String key) { return redis.opsForValue().get(key); }
    public void incr(String key) { redis.opsForValue().increment(key); }
    public void push(String key, String v) { redis.opsForList().rightPush(key, v); }
}
```

### Configuration

```properties
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.data.redis.password=secret
spring.data.redis.database=0
spring.data.redis.timeout=2s
spring.data.redis.lettuce.pool.max-active=8
spring.data.redis.lettuce.pool.max-idle=8
```

### Using Redis as Cache Provider

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```properties
spring.cache.type=redis
spring.cache.redis.time-to-live=600000
```

```java
@Cacheable("users")
public User findById(Long id) { ... }
```

### Also Redis for Spring Session

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

### When to Use

- Caching
- Session storage (distributed)
- Rate limiting
- Pub/Sub
- Distributed locks (Redisson)

---

## 9. spring-boot-starter-security

Spring Security integration.

### Dependencies Pulled In

- `spring-security-web`
- `spring-security-config`
- `spring-security-core`

### Effects of Adding It

- All endpoints secured (HTTP Basic + form login) by default
- `SecurityFilterChain` auto-configured
- Default user generated (password printed on startup)
- CSRF protection for state-changing requests
- Session management enabled
- `PasswordEncoder` (BCrypt via `DelegatingPasswordEncoder`)

### Security 6+ Config

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .formLogin(Customizer.withDefaults())
            .httpBasic(Customizer.withDefaults())
            .csrf(csrf -> csrf.ignoringRequestMatchers("/api/**"));
        return http.build();
    }
    
    @Bean
    public UserDetailsService users(PasswordEncoder encoder) {
        UserDetails alice = User.builder()
            .username("alice").password(encoder.encode("pwd"))
            .roles("USER").build();
        return new InMemoryUserDetailsManager(alice);
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Common Properties

```properties
spring.security.user.name=admin
spring.security.user.password=secret
spring.security.user.roles=ADMIN
```

### When to Use

Any app with authentication/authorization.

### Related

- JWT + OAuth2 → `spring-boot-starter-oauth2-resource-server`
- Client → `spring-boot-starter-oauth2-client`
- Method security — `@EnableMethodSecurity`

---

## 10. spring-boot-starter-test

Testing toolkit — pulled in with `<scope>test</scope>`.

### Dependencies Pulled In

| Library | Purpose |
|---------|---------|
| JUnit 5 (Jupiter) | Test framework |
| Mockito | Mocking |
| AssertJ | Fluent assertions |
| Hamcrest | Matchers |
| JSONassert | JSON assertions |
| JsonPath | JSON path queries |
| Spring Test | Test context framework |
| XMLUnit | XML assertions |
| Awaitility | Async assertions (Boot 2.2+) |
| Mockito JUnit Jupiter | Mockito extension |

### Common Annotations

```java
@SpringBootTest                 // full context
@WebMvcTest(UserController.class)  // MVC slice
@DataJpaTest                    // JPA slice
@JdbcTest                       // JDBC slice
@JsonTest                       // JSON slice
@RestClientTest                 // REST client slice
@Testcontainers                 // Testcontainers
@MockBean                       // replace bean with mock
@SpyBean                        // wrap bean with spy
@ActiveProfiles("test")         // activate profile
@AutoConfigureMockMvc           // inject MockMvc
```

### Example

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {
    
    @Autowired MockMvc mockMvc;
    @MockBean UserService userService;
    
    @Test
    void getReturnsUser() throws Exception {
        given(userService.findById(1L)).willReturn(new User(1L, "Alice"));
        
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"));
    }
}
```

### When to Use

Every Spring Boot project. Add with `<scope>test</scope>`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

---

## 11. spring-boot-starter-actuator

Production-ready endpoints.

### Dependencies Pulled In

- `spring-boot-actuator`
- `spring-boot-actuator-autoconfigure`
- `micrometer-core`

### Built-In Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/actuator/health` | Health (aggregate) |
| `/actuator/info` | App info |
| `/actuator/metrics` | Micrometer metrics |
| `/actuator/env` | Environment properties |
| `/actuator/beans` | All beans |
| `/actuator/mappings` | Request mappings |
| `/actuator/configprops` | @ConfigurationProperties |
| `/actuator/loggers` | View/change log levels |
| `/actuator/threaddump` | Thread dump |
| `/actuator/heapdump` | Heap dump (download) |
| `/actuator/conditions` | Auto-config conditions |
| `/actuator/shutdown` | Graceful shutdown (disabled by default) |

### Configuration

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoints.web.exposure.exclude=env,beans
management.endpoint.health.show-details=when-authorized
management.server.port=9001
management.endpoints.web.base-path=/manage
```

### Health Details

```properties
management.endpoint.health.show-details=always
management.endpoint.health.show-components=always
```

### Custom Health Indicator

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean up = pingExternalApi();
        return up ? Health.up().withDetail("latency", "12ms").build()
                  : Health.down().withDetail("reason", "timeout").build();
    }
}
```

### Prometheus Integration

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```properties
management.endpoints.web.exposure.include=health,info,prometheus
management.metrics.export.prometheus.enabled=true
```

Scraped at `/actuator/prometheus`.

### When to Use

Production applications for monitoring, health checks, and diagnostics.

---

## 12. spring-boot-starter-validation

Jakarta Bean Validation + Hibernate Validator.

### Dependencies Pulled In

- `hibernate-validator`
- `jakarta.validation-api`
- `jakarta.el` (for message interpolation)

### Usage

```java
public class CreateUserRequest {
    @NotBlank @Size(min = 2, max = 50)
    private String name;
    
    @Email @NotBlank
    private String email;
    
    @Min(18) @Max(120)
    private int age;
}

@PostMapping("/users")
public User create(@Valid @RequestBody CreateUserRequest req) { ... }
```

### Auto-Configuration

- `ValidationAutoConfiguration` registers a `LocalValidatorFactoryBean` as `Validator`
- Auto-detects message bundles (`ValidationMessages.properties`, `messages.properties`)

### Custom Messages

```properties
# messages.properties
NotBlank.name=Name must not be blank
Email.email=Invalid email format
```

Or inline: `@NotBlank(message = "{NotBlank.name}")`

### When to Use

Whenever you need input validation. Included automatically by `spring-boot-starter-web` in Boot 3.4+ (before that you had to add it explicitly).

---

## 13. spring-boot-starter-aop

Spring AOP + AspectJ.

### Dependencies Pulled In

- `spring-aop`
- `aspectjweaver`
- `spring-boot-starter` (transitively)

### Auto-Configuration

- `AopAutoConfiguration` — enables `@EnableAspectJAutoProxy` if `@Aspect` classes present
- Proxy mode: CGLIB (`proxyTargetClass=true`) by default in Spring Boot 2.0+

### Usage

```java
@Aspect
@Component
public class LoggingAspect {
    @Around("@annotation(Loggable)")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        log.info("→ {}", pjp.getSignature());
        try { return pjp.proceed(); }
        finally { log.info("← {}", pjp.getSignature()); }
    }
}
```

### Configuration

```properties
spring.aop.auto=true
spring.aop.proxy-target-class=true
```

### When to Use

- Custom AOP aspects (audit, metrics, logging)
- Doesn't need to be added explicitly if another starter already includes it
- `spring-boot-starter-data-jpa` pulls it in transitively

---

## 14. spring-boot-starter-thymeleaf

Thymeleaf server-side template engine.

### Dependencies Pulled In

- `thymeleaf-spring6` (Boot 3.x)
- `thymeleaf-extras-java8time`

### Configuration

```properties
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html
spring.thymeleaf.mode=HTML
spring.thymeleaf.encoding=UTF-8
spring.thymeleaf.cache=true
```

Set `cache=false` for development.

### Template

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><title>Users</title></head>
<body>
<h1>Users</h1>
<ul>
    <li th:each="u : ${users}" th:text="${u.name}"></li>
</ul>
</body>
</html>
```

### Controller

```java
@Controller
public class UserController {
    @GetMapping("/users")
    public String list(Model model) {
        model.addAttribute("users", userService.findAll());
        return "users";   // → templates/users.html
    }
}
```

### Extras

- Layout dialect (`nz.net.ultraq.thymeleaf:thymeleaf-layout-dialect`) for layouts
- Security extras (`thymeleaf-extras-springsecurity6`) for `sec:authorize`

### When to Use

Server-rendered HTML (traditional web apps). For SPA APIs, use REST + frontend framework.

---

## 15. spring-boot-starter-webflux

Reactive web stack.

### Dependencies Pulled In

- `spring-webflux`
- `spring-boot-starter-reactor-netty` (default server)
- `reactor-core`
- `reactor-netty-http`
- `spring-boot-starter-json` (Jackson)

### Auto-Configuration

- `WebFluxAutoConfiguration`
- `ReactiveWebServerFactoryAutoConfiguration` (Netty)
- `CodecsAutoConfiguration`
- `WebClientAutoConfiguration`
- `ReactiveOAuth2AutoConfiguration` (if OAuth added)

### Minimal Controller

```java
@RestController
public class UserController {
    @GetMapping("/users")
    public Flux<User> list() { return userService.findAll(); }
    
    @GetMapping("/users/{id}")
    public Mono<User> get(@PathVariable Long id) { return userService.findById(id); }
}
```

### Functional Style

```java
@Configuration
public class Routes {
    @Bean
    public RouterFunction<ServerResponse> routes(UserHandler handler) {
        return RouterFunctions.route()
            .GET("/users", handler::list)
            .GET("/users/{id}", handler::get)
            .POST("/users", handler::create)
            .build();
    }
}
```

### WebClient

```java
WebClient client = WebClient.create("http://api.example.com");
Mono<User> user = client.get().uri("/users/{id}", 1)
    .retrieve().bodyToMono(User.class);
```

### Common Properties

```properties
server.port=8080
spring.webflux.base-path=/
spring.codec.max-in-memory-size=2MB
```

### When to Use

- High-concurrency, non-blocking apps
- Streaming (SSE, WebSocket)
- Reactive end-to-end (R2DBC, Reactive Redis, etc.)
- Microservices with many parallel calls

### When NOT to Use

- Blocking DB drivers (JDBC, JPA)
- Traditional teams not familiar with reactive
- Simple CRUD — WebMVC is simpler

---

## 16. spring-boot-starter-batch

Spring Batch.

### Dependencies Pulled In

- `spring-boot-starter-jdbc` (Batch needs a `DataSource`)
- `spring-batch-core`
- `spring-batch-infrastructure`

### Configuration

```properties
spring.batch.job.enabled=true
spring.batch.jdbc.initialize-schema=always
spring.batch.job.name=myJob
```

### Job Example

```java
@Configuration
public class JobConfig {
    
    @Bean
    public Job importUsersJob(JobRepository repo, Step step) {
        return new JobBuilder("importUsersJob", repo)
            .start(step)
            .build();
    }
    
    @Bean
    public Step step(JobRepository repo, PlatformTransactionManager tx,
                     ItemReader<User> reader, ItemProcessor<User, User> proc,
                     ItemWriter<User> writer) {
        return new StepBuilder("step", repo)
            .<User, User>chunk(100, tx)
            .reader(reader)
            .processor(proc)
            .writer(writer)
            .build();
    }
    
    @Bean
    public FlatFileItemReader<User> reader() {
        return new FlatFileItemReaderBuilder<User>()
            .name("reader")
            .resource(new ClassPathResource("users.csv"))
            .delimited().names("name", "email").targetType(User.class)
            .build();
    }
    
    @Bean
    public JdbcBatchItemWriter<User> writer(DataSource ds) {
        return new JdbcBatchItemWriterBuilder<User>()
            .dataSource(ds)
            .sql("INSERT INTO users (name, email) VALUES (:name, :email)")
            .beanMapped()
            .build();
    }
}
```

### When to Use

- ETL processes
- Scheduled bulk data imports
- Large-scale processing with restart/skip/retry

---

## 17. spring-boot-starter-amqp (RabbitMQ)

RabbitMQ integration.

### Dependencies Pulled In

- `spring-rabbit`
- `spring-amqp`
- `amqp-client` (Java client)

### Configuration

```properties
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.virtual-host=/
spring.rabbitmq.listener.simple.concurrency=5
spring.rabbitmq.listener.simple.max-concurrency=10
```

### Producer

```java
@Service
public class OrderPublisher {
    private final RabbitTemplate rabbit;
    
    public void publish(Order order) {
        rabbit.convertAndSend("orders-exchange", "order.created", order);
    }
}
```

### Consumer

```java
@Component
public class OrderListener {
    @RabbitListener(queues = "orders-queue")
    public void onOrder(Order order) {
        log.info("Received {}", order);
    }
}
```

### Queue/Exchange Configuration

```java
@Configuration
public class RabbitConfig {
    @Bean public Queue queue() { return new Queue("orders-queue", true); }
    @Bean public TopicExchange exchange() { return new TopicExchange("orders-exchange"); }
    @Bean public Binding binding(Queue q, TopicExchange ex) {
        return BindingBuilder.bind(q).to(ex).with("order.*");
    }
    @Bean public Jackson2JsonMessageConverter converter() { return new Jackson2JsonMessageConverter(); }
}
```

### When to Use

- Task queues
- Pub/sub with routing
- Complex routing (topic, headers exchanges)

---

## 18. spring-boot-starter-kafka

Apache Kafka integration.

### Dependencies Pulled In

- `spring-kafka`
- `kafka-clients`

### Configuration

```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=my-group
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
```

### Producer

```java
@Service
public class OrderProducer {
    private final KafkaTemplate<String, Order> kafka;
    
    public void publish(Order o) {
        kafka.send("orders", o.getId().toString(), o);
    }
}
```

### Consumer

```java
@Component
public class OrderConsumer {
    @KafkaListener(topics = "orders", groupId = "my-group")
    public void listen(Order o) {
        log.info("Received {}", o);
    }
}
```

### When to Use

- Event streaming
- High-throughput pipelines
- Log aggregation
- Event sourcing

---

## 19. Creating Custom Starters

### Structure

```
my-starter-parent/
├── my-starter-autoconfigure/        ← Auto-config + @ConfigurationProperties
│   └── pom.xml
└── my-starter/                      ← Aggregator (empty POM)
    └── pom.xml
```

### Autoconfigure Module

**pom.xml:**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>my-library</artifactId>
        <version>1.0.0</version>
    </dependency>
</dependencies>
```

**Properties class:**

```java
@ConfigurationProperties(prefix = "my.starter")
public class MyStarterProperties {
    private String endpoint = "http://default";
    private Duration timeout = Duration.ofSeconds(5);
    private boolean enabled = true;
    // getters/setters
}
```

**Auto-config:**

```java
@AutoConfiguration
@ConditionalOnClass(MyClient.class)
@EnableConfigurationProperties(MyStarterProperties.class)
public class MyStarterAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "my.starter", name = "enabled", 
                           havingValue = "true", matchIfMissing = true)
    public MyClient myClient(MyStarterProperties props) {
        return new MyClient(props.getEndpoint(), props.getTimeout());
    }
}
```

**Register in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:**

```
com.example.mystarter.MyStarterAutoConfiguration
```

### Starter Module

**pom.xml:**

```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>my-starter-autoconfigure</artifactId>
        <version>1.0.0</version>
    </dependency>
</dependencies>
```

**No code** in this module.

### Consumer Usage

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>my-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

```properties
my.starter.endpoint=https://api.example.com
my.starter.timeout=10s
```

### Additional Files

- `additional-spring-configuration-metadata.json` — for IDE hints
- `spring-configuration-metadata.json` — auto-generated by `spring-boot-configuration-processor`

### Best Practices

- Prefix properties (`my.starter.*`) to avoid collisions
- Use `@ConditionalOnMissingBean` on every bean
- Use `@ConditionalOnClass` on the entire auto-config class
- Support `matchIfMissing` for boolean toggles
- Don't depend on users' beans by type unless you define a dependency

---

## 20. Excluding Auto-Configuration

### Three Ways

**1. @SpringBootApplication:**

```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```

**2. Properties:**

```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

**3. Override beans (most common):**

```java
@Bean
public DataSource dataSource() { return myCustomDataSource(); }
```

Auto-config backs off due to `@ConditionalOnMissingBean`.

### Detecting Active Configs

```bash
java -jar app.jar --debug
```

Output includes:
```
Positive matches:
-----------------
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required class 'javax.sql.DataSource' (OnClassCondition)

Negative matches:
-----------------
   MongoAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'com.mongodb.MongoClient' (OnClassCondition)
```

Actuator: `/actuator/conditions`.

---

## 21. Interview Questions

### Q1. What is a Spring Boot starter?

**Answer:** A starter is a dependency descriptor (POM with no code) that aggregates the libraries needed for a feature and enables auto-configuration. Add one starter → all transitive deps arrive with sensible defaults.

### Q2. How does spring-boot-starter-web work?

**Answer:** It pulls in `spring-web`, `spring-webmvc`, `spring-boot-starter-tomcat`, `spring-boot-starter-json`, and validation. Auto-config registers a `DispatcherServlet`, embedded Tomcat, Jackson converters, error handling, and web MVC infrastructure.

### Q3. Difference between spring-boot-starter-web and spring-boot-starter-webflux?

**Answer:** `web` is Servlet-based (blocking, Tomcat). `webflux` is reactive (non-blocking, Netty). Use `web` for traditional REST, `webflux` for high-concurrency reactive.

### Q4. What does spring-boot-starter-data-jpa include?

**Answer:** Spring Data JPA, Hibernate, JDBC, HikariCP, transaction management, AOP support, and JPA API. Auto-configures `EntityManagerFactory`, `PlatformTransactionManager`, and repositories.

### Q5. Difference between data-jpa and data-jdbc starters?

**Answer:** `data-jpa` uses ORM (Hibernate) with lazy loading, dirty checking, entity graphs. `data-jdbc` is simpler — no lazy loading, aggregate-oriented, explicit SQL. Choose JPA for domain modeling; JDBC for simpler data.

### Q6. How to create a custom starter?

**Answer:**
1. Create autoconfigure module with `@AutoConfiguration` + `@ConfigurationProperties`
2. Register auto-config in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
3. Create a starter module (empty POM) depending on autoconfigure
4. Consumer adds the starter

### Q7. Why prefix auto-config properties?

**Answer:** To avoid collisions with other libraries' properties. Use `my.starter.*` namespace. `@ConfigurationProperties(prefix = "my.starter")` groups them.

### Q8. What happens if you add spring-boot-starter-security?

**Answer:** All endpoints become secured by default. A `SecurityFilterChain` is auto-configured with HTTP Basic + form login, a generated default user + password (printed at startup), CSRF protection, and session management. You customize by defining your own `SecurityFilterChain` bean.

### Q9. What is spring-boot-starter-actuator for?

**Answer:** Provides production-ready endpoints (health, metrics, env, beans, loggers, threaddump, heapdump, etc.) for monitoring and diagnostics. Expose selectively via `management.endpoints.web.exposure.include`.

### Q10. When would you use data-mongodb vs data-jpa?

**Answer:** `data-mongodb` for document-oriented data with flexible schemas. `data-jpa` for relational data with strong schemas and relationships.

### Q11. Difference between data-redis and cache starter?

**Answer:** `data-redis` provides `RedisTemplate` for direct Redis operations. To use Redis as the cache provider, add both `spring-boot-starter-cache` and `spring-boot-starter-data-redis`, then set `spring.cache.type=redis`.

### Q12. Can you have both web and webflux starters?

**Answer:** Technically yes, but Spring Boot can't run both stacks simultaneously at the same path. One will be active; behavior is undefined for mixing. Choose one.

### Q13. What does spring-boot-starter-test include?

**Answer:** JUnit 5, Mockito, AssertJ, Hamcrest, JSONassert, JsonPath, Spring Test framework, XMLUnit, Awaitility, and Mockito JUnit extension.

### Q14. How to exclude auto-config?

**Answer:** `@SpringBootApplication(exclude = X.class)`, `spring.autoconfigure.exclude=...` in properties, or by defining your own bean of the same type (auto-config backs off via `@ConditionalOnMissingBean`).

### Q15. What is the difference between starter and library?

**Answer:** A starter contains **no code** — it's a dependency aggregate. A library has actual classes. Starters make setup easy; libraries do the work.

### Q16. How does spring-boot-starter-batch work?

**Answer:** Pulls Spring Batch + Spring JDBC starter (for job repository). Auto-configures a `JobRepository`, `JobLauncher`, `PlatformTransactionManager`, and runs jobs on startup (controlled by `spring.batch.job.enabled`).

### Q17. Why does Spring Boot require HikariCP?

**Answer:** HikariCP is the default connection pool for JDBC/JPA starters because it's fast, small, and battle-tested. Configured via `spring.datasource.hikari.*`.

### Q18. When to use spring-boot-starter-amqp vs kafka?

**Answer:** AMQP (RabbitMQ) for task queues, routing (topic/headers exchanges), and complex routing. Kafka for high-throughput event streaming, log aggregation, and event sourcing. Both can coexist.

### Q19. What is spring-boot-starter-aop for?

**Answer:** Enables Spring AOP + AspectJ. Auto-configures `@EnableAspectJAutoProxy`. Used for custom aspects (audit, tracing). Pulled in transitively by JPA and Security.

### Q20. Is spring-boot-starter-validation required in Boot 3.4+?

**Answer:** No — `spring-boot-starter-web` includes it since Spring Boot 3.4. In earlier versions you had to add it explicitly.

---

## 22. Cheat Sheet

### Common Starters

```
spring-boot-starter
spring-boot-starter-web
spring-boot-starter-webflux
spring-boot-starter-data-jpa
spring-boot-starter-data-jdbc
spring-boot-starter-data-mongodb
spring-boot-starter-data-redis
spring-boot-starter-security
spring-boot-starter-validation
spring-boot-starter-aop
spring-boot-starter-actuator
spring-boot-starter-test
spring-boot-starter-thymeleaf
spring-boot-starter-batch
spring-boot-starter-amqp
spring-boot-starter-kafka
spring-boot-starter-cache
spring-boot-starter-quartz
spring-boot-starter-mail
spring-boot-starter-logging
```

### Starter vs Auto-Config

- **Starter** = dependency aggregate (POM)
- **Auto-config** = `@AutoConfiguration` classes with conditions
- Consumer adds **starter**, Boot applies **auto-config**

### Custom Starter Files

```
my-starter/
├── my-starter-autoconfigure/
│   └── src/main/java/.../MyStarterAutoConfiguration.java
│   └── src/main/resources/META-INF/spring/
│         org.springframework.boot.autoconfigure.AutoConfiguration.imports
└── my-starter/
    └── pom.xml (depends on autoconfigure)
```

### Excluding Auto-Config

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
```

```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### Debug

```properties
debug=true
```

### Cross-References

- **Previous:** `06_Spring_Boot_Fundamentals.md`
- **Next:** `08_Spring_Boot_Configuration.md`
- **Related:** `09_Spring_Boot_Actuator.md`, `10_Spring_Data_JPA.md`
- **Interview:** `24_Spring_Interview_Questions.md`

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [06_Spring_Boot_Fundamentals.md](./06_Spring_Boot_Fundamentals.md)
- **Next →:** [08_Spring_Boot_Configuration.md](./08_Spring_Boot_Configuration.md)
- **Related:** [06_Spring_Boot_Fundamentals.md](./06_Spring_Boot_Fundamentals.md), [10_Spring_Data_JPA.md](./10_Spring_Data_JPA.md), [13_Spring_Security_Core.md](./13_Spring_Security_Core.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
