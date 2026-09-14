# Spring Ecosystem Study Guide — Master Plan & Table of Contents

## How to Use This Guide

This master file contains the **complete table of contents** for the entire Spring study guide. Each section corresponds to a separate `.md` file that will be created when you trigger it. Place all files in the **same folder** and trigger them in order.

**Workflow:**
1. Read this master file to understand the full scope.
2. Trigger each file individually (e.g., "Now write 01_Spring_Framework_Core.md").
3. Each file will be generated with **maximum detail** — full code examples, explanations, diagrams, and interview notes.
4. After all files are generated, you'll have a complete, interconnected study guide.

---

## File Directory & Table of Contents

### PART 1: FOUNDATIONS

| File | Title | Topics Covered |
|------|-------|----------------|
| `01_Spring_Framework_Core.md` | Spring Framework Core: IoC, DI & Bean Lifecycle | What is Spring, IoC container, Dependency Injection types, BeanFactory vs ApplicationContext, Bean scopes, Bean lifecycle callbacks, Java config, component scanning, stereotypes, SpEL, profiles |
| `02_Spring_AOP.md` | Aspect-Oriented Programming | AOP concepts (aspect, join point, pointcut, advice), @AspectJ annotations, advice types, pointcut expressions, proxy mechanisms (JDK dynamic vs CGLIB), AOP use cases |
| `03_Spring_MVC.md` | Spring MVC & Web Layer | DispatcherServlet, @Controller, @RestController, @RequestMapping, data binding, validation, interceptors, exception handling, view resolvers, REST principles |
| `04_Spring_JDBC.md` | Spring JDBC & JdbcTemplate | Problems with raw JDBC, JdbcTemplate usage, RowMapper, ResultSetExtractor, exception translation, NamedParameterJdbcTemplate |
| `05_Spring_Transaction_Management.md` | Transaction Management | ACID, programmatic vs declarative transactions, @Transactional, propagation behaviors, isolation levels, rollback rules, transaction managers |

---

### PART 2: SPRING BOOT

| File | Title | Topics Covered |
|------|-------|----------------|
| `06_Spring_Boot_Fundamentals.md` | Spring Boot Fundamentals | What is Spring Boot, auto-configuration, @SpringBootApplication, starters, embedded servers, Spring Initializr, application.properties/yml, profiles, Actuator |
| `07_Spring_Boot_Starters.md` | Spring Boot Starters Deep Dive | spring-boot-starter-web, -data-jpa, -security, -test, -actuator, -validation, -aop, custom starters, dependency management |
| `08_Spring_Boot_Configuration.md` | Configuration & Profiles | application.properties vs yml, @Value, @ConfigurationProperties, profile-specific configs, externalized configuration, environment abstraction, property precedence |
| `09_Spring_Boot_Actuator.md` | Spring Boot Actuator | Endpoints (health, metrics, info, env, beans), custom endpoints, Micrometer integration, Prometheus, Grafana, security for Actuator |

---

### PART 3: DATA ACCESS

| File | Title | Topics Covered |
|------|-------|----------------|
| `10_Spring_Data_JPA.md` | Spring Data JPA & Hibernate | JPA entities, @Entity, relationships, EntityManager, JpaRepository, derived queries, @Query (JPQL/Native), pagination, sorting, projections, Specifications, auditing |
| `11_Spring_Data_JDBC.md` | Spring Data JDBC | Spring Data JDBC vs JPA, aggregate roots, CrudRepository, custom queries, when to use JDBC over JPA |
| `12_Spring_Caching.md` | Spring Caching | @Cacheable, @CachePut, @CacheEvict, CacheManager, Redis, Ehcache, Caffeine, cache providers, cache configuration |

---

### PART 4: SECURITY

| File | Title | Topics Covered |
|------|-------|----------------|
| `13_Spring_Security_Core.md` | Spring Security Core | Authentication vs Authorization, SecurityFilterChain, PasswordEncoder, UserDetailsService, form login, HTTP Basic, session management, CSRF, CORS |
| `14_Spring_Security_JWT_OAuth2.md` | JWT & OAuth2 | JWT structure, token generation/validation, OAuth2 flows (Authorization Code, Client Credentials), Resource Server, Authorization Server, Keycloak integration |
| `15_Spring_Security_Method_Level.md` | Method-Level Security | @PreAuthorize, @PostAuthorize, @Secured, @RolesAllowed, SpEL in security expressions, ACL |

---

### PART 5: TESTING

| File | Title | Topics Covered |
|------|-------|----------------|
| `16_Spring_Testing.md` | Spring Testing | JUnit 5, @SpringBootTest, @WebMvcTest, @DataJpaTest, @MockBean, @SpyBean, TestRestTemplate, MockMvc, Testcontainers, integration vs unit testing |

---

### PART 6: BUILD TOOLS

| File | Title | Topics Covered |
|------|-------|----------------|
| `17_Maven_Build_Tool.md` | Apache Maven Complete Guide | POM structure, build lifecycles (default, clean, site), phases, goals, plugins, dependency management, scopes, BOM, parent POM, profiles, multi-module projects, repositories |
| `18_Gradle_For_Spring.md` | Gradle for Spring Developers | Gradle build scripts, dependencies, tasks, plugins, Gradle vs Maven, Spring Boot Gradle plugin |

---

### PART 7: ADVANCED TOPICS

| File | Title | Topics Covered |
|------|-------|----------------|
| `19_Spring_Microservices_Cloud.md` | Spring Cloud & Microservices | Service discovery (Eureka), API Gateway, Config Server, Load balancing, Circuit breakers (Resilience4j), Distributed tracing (Zipkin), Feign, Spring Cloud Bus |
| `20_Spring_WebFlux_Reactive.md` | Spring WebFlux & Reactive Programming | Reactive Streams, Project Reactor (Mono, Flux), backpressure, WebClient, R2DBC, functional endpoints, RSocket |
| `21_Spring_Batch.md` | Spring Batch | Jobs, Steps, ItemReader, ItemProcessor, ItemWriter, chunk processing, job repository, scheduling, error handling, restartability |
| `22_Spring_Messaging.md` | Spring Messaging & Integration | JMS, AMQP (RabbitMQ), Kafka, Spring Cloud Stream, @JmsListener, @KafkaListener, message converters, integration patterns |
| `23_Spring_Web_Services.md` | SOAP & REST Web Services | Spring Web Services (SOAP), REST best practices, HATEOAS, content negotiation, API versioning, OpenAPI/Swagger |

---

### PART 8: PRACTICE & INTERVIEW PREP

| File | Title | Topics Covered |
|------|-------|----------------|
| `24_Spring_Interview_Questions.md` | Spring Interview Questions (200+) | Core Spring, Spring Boot, JPA, Security, Microservices, AOP, transactions, Maven, scenario-based questions |
| `25_Spring_Project_Architecture.md` | Real-World Project Architecture | Layered architecture, DTOs, exception handling, logging, validation, testing strategy, CI/CD, Docker, deployment |
| `26_Spring_Best_Practices.md` | Spring Best Practices & Anti-Patterns | Configuration best practices, dependency injection guidelines, transaction pitfalls, security hardening, performance tuning |
| `27_Spring_Cheat_Sheet.md` | Spring Ecosystem Cheat Sheet | All key annotations, configuration snippets, Maven commands, common patterns — quick reference |

---

## Detailed Topic Breakdown per File

Below is the **full topic list** for each file. When you trigger a file, I will generate the complete content with explanations, code examples, diagrams, and interview tips.

---

### 01_Spring_Framework_Core.md — Spring Core: IoC, DI & Bean Lifecycle

**Topics:**
1. What is Spring Framework? History & evolution (Spring 1.x → 6.x/7.x)
2. Problems Spring solves (tight coupling, boilerplate, testability)
3. Inversion of Control (IoC) — Hollywood Principle
4. Dependency Injection (DI) — constructor, setter, field injection
5. The IoC Container: BeanFactory vs ApplicationContext
6. Bean definition, bean naming, bean aliases
7. Java-based configuration (@Configuration, @Bean)
8. Annotation-based configuration (@Component, @Service, @Repository, @Controller)
9. Component scanning (@ComponentScan, basePackages, filters)
10. Bean scopes: singleton, prototype, request, session, application, websocket
11. Bean lifecycle: initialization callbacks (@PostConstruct, InitializingBean, init-method)
12. Bean lifecycle: destruction callbacks (@PreDestroy, DisposableBean, destroy-method)
13. BeanPostProcessor and BeanFactoryPostProcessor
14. Spring Expression Language (SpEL) — syntax, operators, usage
15. Spring Profiles — @Profile, profile activation
16. External properties — @PropertySource, Environment abstraction
17. Circular dependencies and how Spring resolves them
18. Lazy initialization (@Lazy)
19. @Primary and @Qualifier
20. Interview questions + code examples for each concept

---

### 02_Spring_AOP.md — Aspect-Oriented Programming

**Topics:**
1. What is AOP? Why AOP? Cross-cutting concerns
2. AOP core concepts: Aspect, Join Point, Pointcut, Advice, Weaving, Target, Proxy
3. Types of advice: @Before, @After, @AfterReturning, @AfterThrowing, @Around
4. Pointcut expressions: execution, within, this, target, args, @annotation, bean
5. Combining pointcuts (&&, ||, !)
6. @AspectJ annotation style
7. XML-based AOP configuration (legacy)
8. Proxy mechanisms: JDK dynamic proxy vs CGLIB
9. When Spring uses JDK proxy vs CGLIB
10. AOP use cases: logging, security, transactions, caching, performance monitoring
11. Ordering aspects (@Order)
12. Introduction (inter-type declarations) with @DeclareParents
13. AOP limitations and pitfalls
14. Interview questions + practical examples

---

### 03_Spring_MVC.md — Spring MVC & Web Layer

**Topics:**
1. MVC pattern overview
2. DispatcherServlet — front controller
3. Spring MVC request flow (step-by-step)
4. @Controller vs @RestController
5. @RequestMapping, @GetMapping, @PostMapping, @PutMapping, @DeleteMapping, @PatchMapping
6. Request parameters: @RequestParam, @PathVariable, @RequestBody, @RequestHeader, @CookieValue
7. Response handling: @ResponseBody, ResponseEntity, @ResponseStatus
8. Data binding and type conversion
9. Validation: @Valid, @Validated, BindingResult, custom validators
10. Exception handling: @ExceptionHandler, @ControllerAdvice, @RestControllerAdvice
11. Interceptors (HandlerInterceptor)
12. Filters vs Interceptors
13. View resolvers: InternalResourceViewResolver, Thymeleaf, JSP
14. Content negotiation (JSON, XML)
15. REST principles and HATEOAS
16. File upload/download
17. Async request processing (Callable, DeferredResult)
18. Interview questions + code examples

---

### 04_Spring_JDBC.md — Spring JDBC & JdbcTemplate

**Topics:**
1. Problems with raw JDBC (boilerplate, resource leaks, exception handling)
2. Spring JDBC architecture
3. DataSource configuration (HikariCP, Tomcat JDBC)
4. JdbcTemplate — CRUD operations
5. RowMapper and ResultSetExtractor
6. Batch operations with JdbcTemplate
7. NamedParameterJdbcTemplate
8. SimpleJdbcInsert and SimpleJdbcCall
9. Exception translation (DataAccessException hierarchy)
10. Transaction integration with JdbcTemplate
11. When to use JDBC vs JPA
12. Interview questions + code examples

---

### 05_Spring_Transaction_Management.md — Transaction Management

**Topics:**
1. ACID properties
2. Programmatic transactions (TransactionTemplate, PlatformTransactionManager)
3. Declarative transactions (@Transactional)
4. Transaction propagation behaviors (REQUIRED, REQUIRES_NEW, SUPPORTS, NOT_SUPPORTED, MANDATORY, NEVER, NESTED)
5. Isolation levels (READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE)
6. Rollback rules (rollbackFor, noRollbackFor)
7. readOnly and timeout attributes
8. Transaction managers: DataSourceTransactionManager, JpaTransactionManager, ChainedTransactionManager
9. @Transactional on class vs method level
10. Common pitfalls (self-invocation, private methods, proxy mode)
11. Transaction synchronization and event handling
12. Interview questions + code examples

---

### 06_Spring_Boot_Fundamentals.md — Spring Boot Fundamentals

**Topics:**
1. What is Spring Boot? Why Spring Boot?
2. Spring Boot vs Spring Framework
3. @SpringBootApplication (combination of @Configuration, @EnableAutoConfiguration, @ComponentScan)
4. Auto-configuration: how it works (@EnableAutoConfiguration, spring.factories, AutoConfiguration.imports)
5. Conditional annotations (@ConditionalOnClass, @ConditionalOnMissingBean, @ConditionalOnProperty)
6. Spring Boot Starters — list and purpose
7. Embedded servers (Tomcat, Jetty, Undertow)
8. Spring Initializr (start.spring.io)
9. Spring Boot CLI
10. application.properties vs application.yml
11. Spring Boot DevTools
12. Spring Boot versioning and release cadence
13. Migrating from Spring to Spring Boot
14. Interview questions + code examples

---

### 07_Spring_Boot_Starters.md — Spring Boot Starters Deep Dive

**Topics:**
1. What is a starter? How starters work
2. spring-boot-starter (core)
3. spring-boot-starter-web (MVC, REST, embedded Tomcat)
4. spring-boot-starter-data-jpa (Hibernate, Spring Data JPA)
5. spring-boot-starter-data-jdbc
6. spring-boot-starter-data-mongodb
7. spring-boot-starter-data-redis
8. spring-boot-starter-security
9. spring-boot-starter-test (JUnit 5, Mockito, AssertJ, MockMvc)
10. spring-boot-starter-actuator
11. spring-boot-starter-validation
12. spring-boot-starter-aop
13. spring-boot-starter-thymeleaf
14. spring-boot-starter-webflux
15. spring-boot-starter-batch
16. spring-boot-starter-amqp (RabbitMQ)
17. spring-boot-starter-kafka
18. Creating custom starters
19. Excluding auto-configuration
20. Interview questions

---

### 08_Spring_Boot_Configuration.md — Configuration & Profiles

**Topics:**
1. application.properties syntax
2. application.yml syntax
3. Property placeholders and random values
4. @Value annotation
5. @ConfigurationProperties (relaxed binding, validation)
6. Profile-specific properties (application-dev.properties, application-prod.yml)
7. Activating profiles (spring.profiles.active, @Profile, programmatically)
8. Profile groups (Spring Boot 2.4+)
9. Externalized configuration (environment variables, command-line args)
10. Property precedence order
11. @PropertySource and @PropertySources
12. Config import (spring.config.import)
13. Encrypting properties (Jasypt)
14. Interview questions + code examples

---

### 09_Spring_Boot_Actuator.md — Spring Boot Actuator

**Topics:**
1. What is Actuator? Why use it?
2. Enabling Actuator (dependency + endpoints)
3. Built-in endpoints: /health, /info, /metrics, /env, /beans, /mappings, /loggers, /threaddump, /heapdump, /shutdown
4. Health indicators (custom health checks)
5. Metrics with Micrometer
6. Prometheus integration
7. Grafana dashboards
8. Custom endpoints (@Endpoint, @ReadOperation, @WriteOperation)
9. Securing Actuator endpoints
10. Actuator in production (best practices)
11. Interview questions + configuration examples

---

### 10_Spring_Data_JPA.md — Spring Data JPA & Hibernate

**Topics:**
1. ORM concepts and the object-relational impedance mismatch
2. JPA vs Hibernate vs Spring Data JPA
3. Entity mapping: @Entity, @Table, @Id, @GeneratedValue, @Column
4. Relationships: @OneToOne, @OneToMany, @ManyToOne, @ManyToMany, @JoinColumn, @JoinTable
5. Cascade types and orphan removal
6. Fetch types (LAZY vs EAGER) and N+1 problem
7. Entity lifecycle (transient, managed, detached, removed)
8. Persistence context and dirty checking
9. JpaRepository, CrudRepository, PagingAndSortingRepository
10. Derived query methods (findBy, countBy, deleteBy, existsBy)
11. @Query with JPQL
12. @Query with native SQL
13. Named parameters and SpEL in queries
14. Pagination and sorting (Pageable, Sort)
15. Projections (interface-based, class-based)
16. Specifications and Criteria API
17. Entity graphs
18. Auditing (@CreatedDate, @LastModifiedDate, @CreatedBy, @LastModifiedBy)
19. Locking (optimistic @Version, pessimistic)
20. Caching (first-level, second-level, query cache)
21. Interview questions + code examples

---

### 11_Spring_Data_JDBC.md — Spring Data JDBC

**Topics:**
1. Spring Data JDBC vs Spring Data JPA
2. Aggregate roots and aggregate design
3. CrudRepository for Spring Data JDBC
4. @Id mapping and ID generation
5. Custom queries (@Query)
6. Relationships in Spring Data JDBC (@MappedCollection)
7. When to choose Spring Data JDBC over JPA
8. Interview questions + code examples

---

### 12_Spring_Caching.md — Spring Caching

**Topics:**
1. Caching concepts (cache hit, miss, eviction)
2. @EnableCaching
3. @Cacheable (key generation, condition, unless)
4. @CachePut
5. @CacheEvict (beforeInvocation, allEntries)
6. @Caching (combining multiple annotations)
7. CacheManager abstraction
8. Cache providers: ConcurrentMapCache, Ehcache, Caffeine, Redis, Hazelcast
9. Configuring Redis cache
10. Custom key generators
11. Cache metrics and monitoring
12. Interview questions + code examples

---

### 13_Spring_Security_Core.md — Spring Security Core

**Topics:**
1. Security concepts: authentication, authorization, principal, authority
2. SecurityFilterChain (Spring Security 6+)
3. PasswordEncoder (BCrypt, Argon2, PBKDF2)
4. UserDetails and UserDetailsService
5. In-memory authentication
6. JDBC authentication
7. Form login (custom login page)
8. HTTP Basic authentication
9. Session management (session fixation, concurrency control)
10. CSRF protection (how it works, when to disable)
11. CORS configuration
12. Logout handling
13. Remember-me authentication
14. Security headers (HSTS, X-Frame-Options, CSP)
15. Interview questions + code examples

---

### 14_Spring_Security_JWT_OAuth2.md — JWT & OAuth2

**Topics:**
1. JWT structure (header, payload, signature)
2. JWT vs opaque tokens
3. Generating and validating JWT (jjwt library)
4. JWT authentication filter
5. Refresh tokens
6. OAuth2 roles: Resource Owner, Client, Authorization Server, Resource Server
7. OAuth2 grant types (Authorization Code, Client Credentials, Password, Refresh Token)
8. Configuring Spring Security as OAuth2 Resource Server
9. Configuring Spring Authorization Server
10. Keycloak integration
11. Social login (Google, GitHub)
12. Interview questions + code examples

---

### 15_Spring_Security_Method_Level.md — Method-Level Security

**Topics:**
1. @EnableMethodSecurity
2. @PreAuthorize and @PostAuthorize
3. @PreFilter and @PostFilter
4. @Secured (legacy)
5. @RolesAllowed (JSR-250)
6. SpEL expressions in security annotations
7. @AuthenticationPrincipal
8. ACL (Access Control List) with Spring Security ACL
9. Interview questions + code examples

---

### 16_Spring_Testing.md — Spring Testing

**Topics:**
1. JUnit 5 fundamentals (@Test, @BeforeEach, @AfterEach, @ParameterizedTest)
2. @SpringBootTest (webEnvironment modes)
3. @WebMvcTest (MockMvc, @MockBean)
4. @DataJpaTest (TestEntityManager, @AutoConfigureTestDatabase)
5. @JdbcTest, @JsonTest, @RestClientTest
6. Mockito: @Mock, @Spy, @InjectMocks, @MockBean, @SpyBean
7. MockMvc — request builders, matchers
8. TestRestTemplate vs WebTestClient
9. Testcontainers (PostgreSQL, MySQL, Redis)
10. @Sql for test data
11. Transactional tests (@Transactional, @Rollback)
12. Test slices vs full integration tests
13. Test coverage (JaCoCo)
14. Interview questions + code examples

---

### 17_Maven_Build_Tool.md — Apache Maven Complete Guide

**Topics:**
1. What is Maven? Why Maven?
2. Installing Maven and directory structure
3. POM (Project Object Model) — structure and elements
4. Maven coordinates (groupId, artifactId, version, packaging, classifier)
5. Maven build lifecycles: default, clean, site
6. Default lifecycle phases (validate → compile → test → package → verify → install → deploy)
7. Clean lifecycle phases
8. Site lifecycle phases
9. Goals vs phases
10. Plugins (maven-compiler-plugin, maven-surefire-plugin, maven-jar-plugin, spring-boot-maven-plugin)
11. Dependency management (dependencies, transitive dependencies, exclusions)
12. Dependency scopes (compile, provided, runtime, test, system, import)
13. Parent POM and inheritance
14. BOM (Bill of Materials) — dependencyManagement
15. Maven profiles
16. Properties and filtering
17. Repositories (local, central, remote)
18. Multi-module projects (aggregator POM, modules)
19. Maven wrapper (mvnw)
20. Common Maven commands (mvn clean install, mvn package, mvn test, mvn dependency:tree)
21. Maven vs Gradle comparison
22. Interview questions + code examples

---

### 18_Gradle_For_Spring.md — Gradle for Spring Developers

**Topics:**
1. Gradle vs Maven — pros and cons
2. build.gradle and settings.gradle
3. Gradle plugins (java, org.springframework.boot, io.spring.dependency-management)
4. Dependency configurations (implementation, api, compileOnly, runtimeOnly, testImplementation)
5. Gradle tasks and lifecycle
6. Gradle wrapper (gradlew)
7. Multi-project builds
8. Gradle Kotlin DSL
9. Common Gradle commands
10. Interview questions

---

### 19_Spring_Microservices_Cloud.md — Spring Cloud & Microservices

**Topics:**
1. Monolith vs Microservices — pros and cons
2. Microservices design principles (bounded context, database per service, API gateway)
3. Service discovery: Netflix Eureka, Consul
4. API Gateway: Spring Cloud Gateway, Netflix Zuul (legacy)
5. Client-side load balancing: Spring Cloud LoadBalancer, Ribbon (legacy)
6. Declarative REST clients: OpenFeign
7. Circuit breakers: Resilience4j, Hystrix (legacy)
8. Distributed configuration: Spring Cloud Config Server
9. Distributed tracing: Sleuth, Zipkin, Micrometer Tracing
10. Spring Cloud Bus (event-driven configuration refresh)
11. Spring Cloud Stream (Kafka, RabbitMQ)
12. Spring Cloud Contract (consumer-driven contracts)
13. Spring Cloud Security (OAuth2, JWT propagation)
14. Docker and Kubernetes for Spring Boot
15. Interview questions + architecture examples

---

### 20_Spring_WebFlux_Reactive.md — Spring WebFlux & Reactive Programming

**Topics:**
1. Reactive programming paradigm
2. Reactive Streams specification
3. Project Reactor: Mono and Flux
4. Creating Mono/Flux (just, fromIterable, fromStream, empty, error)
5. Operators: map, flatMap, filter, zip, merge, concat, reduce, collect
6. Backpressure handling
7. Error handling in Reactor (onErrorReturn, onErrorResume, retry)
8. Schedulers and threading (subscribeOn, publishOn)
9. Spring WebFlux: functional endpoints (RouterFunction, HandlerFunction)
10. Spring WebFlux: annotated controllers
11. WebClient (reactive HTTP client)
12. R2DBC (reactive relational database access)
13. Reactive MongoDB
14. Server-Sent Events (SSE)
15. RSocket
16. Testing reactive streams (StepVerifier, WebTestClient)
17. Interview questions + code examples

---

### 21_Spring_Batch.md — Spring Batch

**Topics:**
1. Batch processing concepts
2. Spring Batch architecture (Job, Step, JobLauncher, JobRepository)
3. ItemReader, ItemProcessor, ItemWriter
4. Chunk-oriented processing
5. Tasklet processing
6. Job parameters and JobExecutionContext
7. Job listeners and Step listeners
8. Skip, retry, and restart logic
9. Job scheduling (Spring Scheduler, Quartz)
10. Partitioning and parallel processing
11. Spring Batch Admin and monitoring
12. Interview questions + code examples

---

### 22_Spring_Messaging.md — Spring Messaging & Integration

**Topics:**
1. Messaging concepts (producer, consumer, queue, topic, broker)
2. JMS with Spring (JmsTemplate, @JmsListener, ActiveMQ, Artemis)
3. AMQP with Spring (RabbitTemplate, @RabbitListener, RabbitMQ)
4. Apache Kafka with Spring (@KafkaListener, KafkaTemplate)
5. Spring Cloud Stream (binders, @StreamListener, functional model)
6. Spring Integration (channels, endpoints, adapters, gateways)
7. Message converters and serialization (JSON, Avro)
8. Error handling and dead-letter queues
9. Transactional messaging
10. Interview questions + code examples

---

### 23_Spring_Web_Services.md — SOAP & REST Web Services

**Topics:**
1. SOAP web services with Spring Web Services (Contract-First, WSDL)
2. REST best practices (resource naming, HTTP status codes, idempotency)
3. HATEOAS with Spring HATEOAS
4. Content negotiation (JSON, XML, custom media types)
5. API versioning strategies (URI, header, media type)
6. OpenAPI/Swagger with SpringDoc
7. REST exception handling best practices
8. Rate limiting and throttling
9. API security best practices
10. Interview questions + code examples

---

### 24_Spring_Interview_Questions.md — Spring Interview Questions (200+)

**Topics:**
1. Core Spring (IoC, DI, Bean lifecycle, scopes) — 30 questions
2. Spring AOP — 15 questions
3. Spring MVC — 20 questions
4. Spring Boot — 30 questions
5. Spring Data JPA — 25 questions
6. Spring Security — 20 questions
7. Transactions — 15 questions
8. Spring Cloud & Microservices — 20 questions
9. Spring WebFlux — 10 questions
10. Maven — 15 questions
11. Scenario-based questions — 20 questions
12. Coding problems (write a REST controller, configure security, etc.)

---

### 25_Spring_Project_Architecture.md — Real-World Project Architecture

**Topics:**
1. Layered architecture (controller → service → repository → entity)
2. DTO pattern and MapStruct
3. Exception handling strategy (global exception handler)
4. Logging strategy (SLF4J, Logback, structured logging)
5. Validation strategy (Jakarta Validation, custom validators)
6. Testing strategy (unit, integration, contract, e2e)
7. CI/CD with GitHub Actions, Jenkins
8. Dockerizing Spring Boot applications
9. Kubernetes deployment (Deployment, Service, ConfigMap, Secret)
10. Database migration (Flyway, Liquibase)
11. Monitoring and observability (Actuator, Micrometer, Prometheus, Grafana)
12. Security hardening for production
13. Performance tuning (JVM, connection pooling, caching)
14. Code examples and project structure

---

### 26_Spring_Best_Practices.md — Spring Best Practices & Anti-Patterns

**Topics:**
1. Configuration best practices (Java config over XML, @ConfigurationProperties)
2. Dependency injection best practices (constructor injection over field injection)
3. Bean scoping best practices
4. Transaction management best practices
5. Exception handling best practices
6. Security best practices
7. JPA/Hibernate best practices (avoid N+1, use projections, pagination)
8. REST API best practices
9. Testing best practices
10. Performance anti-patterns (eager fetching, excessive logging, large transactions)
11. Common mistakes and how to avoid them
12. Code review checklist

---

### 27_Spring_Cheat_Sheet.md — Spring Ecosystem Cheat Sheet

**Topics:**
1. All Spring annotations (organized by category)
2. All Spring Boot annotations
3. Spring Data JPA query keywords
4. Spring Security configuration snippets
5. Maven commands and POM snippets
6. Common application.properties/yml properties
7. Spring Boot Actuator endpoints
8. Testing annotations
9. Microservices annotations
10. WebFlux annotations
11. Quick reference tables and diagrams

---

## Study Path Recommendation

**Phase 1: Foundations (Weeks 1–4)**
- 01 Spring Core → 02 AOP → 03 Spring MVC → 04 Spring JDBC → 05 Transactions

**Phase 2: Spring Boot (Weeks 5–7)**
- 06 Spring Boot Fundamentals → 07 Starters → 08 Configuration → 09 Actuator

**Phase 3: Data & Security (Weeks 8–11)**
- 10 Spring Data JPA → 11 Spring Data JDBC → 12 Caching → 13 Security Core → 14 JWT/OAuth2 → 15 Method Security

**Phase 4: Testing & Build (Weeks 12–13)**
- 16 Spring Testing → 17 Maven → 18 Gradle

**Phase 5: Advanced (Weeks 14–18)**
- 19 Microservices → 20 WebFlux → 21 Batch → 22 Messaging → 23 Web Services

**Phase 6: Interview Prep (Weeks 19–20)**
- 24 Interview Questions → 25 Project Architecture → 26 Best Practices → 27 Cheat Sheet

---