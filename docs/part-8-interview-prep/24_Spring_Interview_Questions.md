# Spring Interview Questions (200+)

> **File:** `24_Spring_Interview_Questions.md`
> **Part:** 8 — Practice & Interview Prep
> **Prerequisites:** All previous files
> **Estimated Study Time:** 20–30 hours (read + practice answering aloud)

---

## Table of Contents

1. [Core Spring — 30 Questions](#1-core-spring--30-questions)
2. [Spring AOP — 15 Questions](#2-spring-aop--15-questions)
3. [Spring MVC — 20 Questions](#3-spring-mvc--20-questions)
4. [Spring Boot — 30 Questions](#4-spring-boot--30-questions)
5. [Spring Data JPA — 25 Questions](#5-spring-data-jpa--25-questions)
6. [Spring Security — 20 Questions](#6-spring-security--20-questions)
7. [Transactions — 15 Questions](#7-transactions--15-questions)
8. [Spring Cloud & Microservices — 20 Questions](#8-spring-cloud--microservices--20-questions)
9. [Spring WebFlux — 10 Questions](#9-spring-webflux--10-questions)
10. [Maven & Gradle — 15 Questions](#10-maven--gradle--15-questions)
11. [Scenario-Based Questions — 20 Questions](#11-scenario-based-questions--20-questions)
12. [Coding Problems — 10 Problems](#12-coding-problems--10-problems)
13. [Quick Reference](#13-quick-reference)

---

## 1. Core Spring — 30 Questions

### Q1. What is Spring Framework?

**Answer:** Spring is a lightweight, open-source Java framework for building enterprise applications. Its core features are **IoC (Inversion of Control)** and **DI (Dependency Injection)**, with modules for web (MVC/WebFlux), data access (JDBC/ORM), security, AOP, messaging, batch, and more. It promotes POJO-based, non-invasive, testable development.

### Q2. What are the core modules of Spring?

**Answer:**
- **Core Container**: Beans, Context, SpEL, IoC/DI
- **AOP**: Aspects, instrumentation
- **Data Access**: JDBC, ORM, TX, JMS
- **Web**: MVC, WebFlux, WebSocket
- **Test**: Mock objects, TestContext framework
- **Integration**: JMS, JMX, messaging

### Q3. What is Inversion of Control (IoC)?

**Answer:** IoC is a design principle where object creation and lifecycle management are delegated to a container (Spring) instead of application code. Instead of `new Service()`, the container creates and injects dependencies. "Don't call us, we'll call you" (Hollywood Principle).

### Q4. What is Dependency Injection (DI)?

**Answer:** DI is the concrete implementation of IoC where dependencies are provided to an object externally rather than the object creating them. Three types: **constructor**, **setter**, and **field** injection. Constructor injection is preferred.

### Q5. Constructor vs setter vs field injection?

**Answer:**
- **Constructor**: Immutable (final), fails fast, testable without Spring. **Recommended.**
- **Setter**: Optional dependencies, mutable, allows reconfiguration.
- **Field**: Concise but untestable without reflection, hidden dependencies. **Avoid.**

### Q6. Why is constructor injection preferred?

**Answer:** 1) Enforces immutability via `final`. 2) Guarantees dependencies at construction (NPE prevention). 3) Fails fast on missing beans at startup. 4) Easy to test — instantiate directly with mocks. 5) No Spring imports needed. 6) Detects circular dependencies at startup.

### Q7. BeanFactory vs ApplicationContext?

**Answer:** `BeanFactory` is the basic container with lazy initialization and minimal features. `ApplicationContext` extends it with eager singleton instantiation, events, i18n, resource loading, AOP, and annotation support. Always prefer `ApplicationContext`.

### Q8. What are bean scopes in Spring?

**Answer:**
- **singleton** (default): one instance per container
- **prototype**: new instance per request
- **request**: one per HTTP request
- **session**: one per HTTP session
- **application**: one per ServletContext
- **websocket**: one per WebSocket session
- Plus custom scopes via `Scope` interface.

### Q9. Explain the bean lifecycle.

**Answer:**
1. Instantiation (constructor)
2. Populate properties (DI)
3. `BeanNameAware.setBeanName()`
4. `BeanFactoryAware.setBeanFactory()`
5. `ApplicationContextAware.setApplicationContext()`
6. `BeanPostProcessor.postProcessBeforeInitialization()`
7. `@PostConstruct`
8. `InitializingBean.afterPropertiesSet()`
9. Custom `init-method`
10. `BeanPostProcessor.postProcessAfterInitialization()` (AOP proxies)
11. Bean ready
12. `@PreDestroy`
13. `DisposableBean.destroy()`
14. Custom `destroy-method`

### Q10. What is the difference between @Component, @Service, @Repository, @Controller?

**Answer:** All are specializations of `@Component` and functionally equivalent for component scanning. They differ semantically: `@Service` (business layer), `@Repository` (data layer + exception translation), `@Controller`/`@RestController` (web layer). Using them communicates intent.

### Q11. What is @Configuration and how does it work?

**Answer:** `@Configuration` marks a class as a bean definition source. Spring subclasses it with **CGLIB** so inter-bean method calls return the same singleton. `proxyBeanMethods = false` disables CGLIB for performance (when no inter-bean calls).

### Q12. Difference between @Configuration and @Component?

**Answer:** `@Configuration` is CGLIB-enhanced by default — `@Bean` method calls return singletons. `@Component` (with `@Bean` methods) is not enhanced — each call creates a new instance. Use `@Configuration` for bean factories.

### Q13. What is @ComponentScan?

**Answer:** Tells Spring where to look for stereotype-annotated classes. Attributes: `basePackages`, `basePackageClasses`, `includeFilters`, `excludeFilters`, `lazyInit`. `@SpringBootApplication` includes `@ComponentScan` for the main class's package and subpackages.

### Q14. How does @Autowired resolve ambiguity?

**Answer:** 1) By type. 2) `@Primary` wins. 3) `@Qualifier` at injection point. 4) Custom qualifier annotations. 5) Bean name matching field/parameter name. Otherwise `NoUniqueBeanDefinitionException`.

### Q15. What is @Primary?

**Answer:** Marks a bean as the default when multiple candidates exist for autowiring. `@Qualifier` at the injection point overrides `@Primary`.

### Q16. What is @Qualifier?

**Answer:** Explicitly selects a bean by name or custom qualifier. Used when multiple beans of the same type exist. `@Qualifier` overrides `@Primary`.

### Q17. What is @Lazy?

**Answer:** Defers bean instantiation until first use. Can be on `@Component`, `@Bean`, or injection points (`@Lazy SomeBean bean`). Useful for expensive beans or breaking circular dependencies. Caveat: errors appear at runtime, not startup.

### Q18. How does Spring resolve circular dependencies?

**Answer:** For setter/field injection, Spring uses a **three-level cache**: `singletonObjects` (fully initialized), `earlySingletonObjects` (early refs), `singletonFactories` (factories). It exposes a partially-initialized bean early. Constructor injection can't be resolved this way (fails with `BeanCurrentlyInCreationException`). Spring Boot 2.6+ disables circular refs by default.

### Q19. What is a BeanPostProcessor?

**Answer:** An interface that intercepts bean creation. `postProcessBeforeInitialization()` runs before init callbacks; `postProcessAfterInitialization()` runs after (where AOP proxies are created). Used for `@Autowired`, `@PostConstruct`, AOP, custom wrapping.

### Q20. What is a BeanFactoryPostProcessor?

**Answer:** Runs **before** any bean is instantiated and can modify **BeanDefinitions**. Example: `PropertySourcesPlaceholderConfigurer` resolves `${...}` placeholders. Order via `Ordered` or `@Order`.

### Q21. Difference between BeanPostProcessor and BeanFactoryPostProcessor?

**Answer:** `BeanFactoryPostProcessor` modifies bean **definitions** (metadata) before instantiation. `BeanPostProcessor` modifies bean **instances** before and after initialization. BPP runs per bean; BFPP runs once.

### Q22. What is SpEL?

**Answer:** Spring Expression Language — runtime expression language (`#{...}`) supporting arithmetic, comparisons, method calls, bean references, static type access (`T()`), safe navigation (`?.`), ternary, Elvis (`?:`), collection selection/projection. Used in `@Value`, `@PreAuthorize`, `@ConditionalOnExpression`. Never evaluate untrusted input.

### Q23. What is @Value?

**Answer:** Injects property values or SpEL expressions into fields, constructor params, or method params. `${property:default}` for properties, `#{expression}` for SpEL. For grouped config, prefer `@ConfigurationProperties`.

### Q24. Difference between @Value and @ConfigurationProperties?

**Answer:** `@Value` injects a single property with SpEL; lacks type-safety and grouping. `@ConfigurationProperties(prefix=...)` binds a group of properties to a POJO with type-safety, validation (`@Validated`), relaxed binding, and IDE completion. Prefer `@ConfigurationProperties` for grouped config.

### Q25. What are Spring Profiles?

**Answer:** Profiles (`@Profile("dev")`) register beans only when a profile is active. Activate via `spring.profiles.active`, `SPRING_PROFILES_ACTIVE` env var, command line, or `@ActiveProfiles` in tests. Support expressions: `!prod`, `dev & cloud`, `dev | staging`.

### Q26. What is the difference between @PropertySource and application.properties?

**Answer:** `@PropertySource` explicitly loads a properties file into the Environment. Spring Boot auto-loads `application.properties`/`application.yml` from classpath, config dirs, and profile-specific variants. `@PropertySource` is used for custom files.

### Q27. What is a FactoryBean?

**Answer:** A bean whose `getObject()` produces another bean. Used for complex creation (e.g., `SqlSessionFactoryBean`, `EntityManagerFactoryBean`). Retrieved by `&beanName` to get the factory itself.

### Q28. What is `ApplicationContextAware`?

**Answer:** Interface with `setApplicationContext(ApplicationContext)`. Spring injects the context automatically. Useful for dynamic bean lookup but couples to Spring. Prefer constructor injection.

### Q29. What is the difference between @Bean and @Component?

**Answer:** `@Component` is on the class — Spring discovers it via scanning. `@Bean` is on a `@Configuration` method — you write the instantiation logic, useful for third-party classes you can't annotate.

### Q30. What are common Spring anti-patterns?

**Answer:**
- Field injection (untestable, hidden deps)
- Overuse of `@Autowired(required=false)`
- Circular dependencies (design smell)
- Prototype bean injected into singleton (frozen instance)
- `@Value` for grouped config (use `@ConfigurationProperties`)
- Untrusted SpEL (RCE risk)
- `@Component` with `@Bean` methods (no CGLIB)

---

## 2. Spring AOP — 15 Questions

### Q31. What is AOP?

**Answer:** Aspect-Oriented Programming separates **cross-cutting concerns** (logging, security, transactions) from business logic. Spring AOP uses **proxy-based** weaving at runtime.

### Q32. What are the core AOP concepts?

**Answer:**
- **Aspect**: modular cross-cutting concern
- **Join Point**: point in execution (method call)
- **Pointcut**: expression matching join points
- **Advice**: action at a join point (before, after, around)
- **Weaving**: linking aspects to targets
- **Target**: object being advised
- **Proxy**: object wrapping the target

### Q33. What are the advice types?

**Answer:**
- `@Before` — before method
- `@After` — after (finally)
- `@AfterReturning` — after successful return
- `@AfterThrowing` — after exception
- `@Around` — wraps method (most powerful)

### Q34. What is a pointcut expression?

**Answer:** An expression matching join points. Common designators: `execution()`, `within()`, `this()`, `target()`, `args()`, `@annotation()`, `@within()`, `bean()`. Combine with `&&`, `||`, `!`.

Example: `execution(* com.example.service.*.*(..))`

### Q35. JDK dynamic proxy vs CGLIB?

**Answer:** JDK proxy: interface-based, uses `java.lang.reflect.Proxy`, requires target to implement interfaces. CGLIB: subclass-based, no interface needed, can't proxy `final` classes/methods. Spring uses JDK by default if interfaces exist, else CGLIB. `proxyTargetClass=true` forces CGLIB.

### Q36. What is @Around and how is it different?

**Answer:** `@Around` wraps the method — you control invocation via `ProceedingJoinPoint.proceed()`. Can modify args, skip invocation, change return, catch exceptions. Most powerful; use for timing, caching, retries.

### Q37. What is @Order in AOP?

**Answer:** Controls aspect execution order. Lower value = higher precedence (outer). `@Order(1)` runs before `@Order(2)`. Also applies to `BeanPostProcessor` and `CommandLineRunner`.

### Q38. What is @DeclareParents?

**Answer:** Introduction (inter-type declaration) — makes a target implement additional interfaces. Example: add `Auditable` behavior to all `Service` beans.

### Q39. What are AOP limitations?

**Answer:**
- Only method-execution join points (not field access, constructor)
- Self-invocation bypasses proxy
- `final` classes/methods can't be proxied by CGLIB
- Private methods can't be advised
- Performance overhead (small)

### Q40. Why does self-invocation bypass AOP?

**Answer:** AOP uses proxies. When a method calls another method on `this`, the call bypasses the proxy. Solutions: inject self via `@Autowired`, use `AopContext.currentProxy()`, or split into separate beans.

### Q41. How does @Transactional relate to AOP?

**Answer:** `@Transactional` is implemented via AOP. Spring creates a proxy that begins/commits/rolls back transactions around the method. Same self-invocation pitfall applies.

### Q42. What is AspectJ vs Spring AOP?

**Answer:** Spring AOP is proxy-based, applies at runtime, only method join points. AspectJ is bytecode-weaving (compile/load-time), supports all join points (field, constructor), more powerful but complex. Spring uses `@AspectJ` syntax with proxy-based weaving.

### Q43. What is a pointcut designator?

**Answer:** Keyword in pointcut expressions:
- `execution(...)` — method execution
- `within(Type)` — within a type
- `this(Type)` — proxy is Type
- `target(Type)` — target is Type
- `args(Type,..)` — argument types
- `@annotation(Ann)` — method has annotation
- `@within(Ann)` — type has annotation
- `bean(name)` — bean name

### Q44. Can you advise private methods?

**Answer:** No — Spring AOP proxies can't intercept private methods. Only public (and protected for CGLIB) methods.

### Q45. What is the difference between @Around and @Before + @After?

**Answer:** `@Before` runs before; `@After` runs after (always). `@Around` gives full control — can skip invocation, modify args/return, catch exceptions, and choose whether to proceed. `@Around` is more powerful but easier to misuse.

---

## 3. Spring MVC — 20 Questions

### Q46. What is DispatcherServlet?

**Answer:** Front controller in Spring MVC. Receives all HTTP requests, delegates to handler mappings, invokes controllers, resolves views, and returns responses. Configured via `web.xml` or auto-configured by Spring Boot.

### Q47. Explain the Spring MVC request flow.

**Answer:**
1. Request → `DispatcherServlet`
2. `HandlerMapping` finds the handler (controller method)
3. `HandlerAdapter` invokes the handler
4. Handler calls service layer
5. Return value → `HandlerMethodReturnValueHandler`
6. `ViewResolver` resolves view (or message converter for `@ResponseBody`)
7. Response returned

### Q48. @Controller vs @RestController?

**Answer:** `@Controller` returns view names (resolved by ViewResolver). `@RestController` = `@Controller` + `@ResponseBody` — returns data serialized to JSON/XML directly.

### Q49. What is @RequestMapping?

**Answer:** Maps HTTP requests to handler methods. Attributes: `value`/`path`, `method`, `params`, `headers`, `consumes`, `produces`. Specializations: `@GetMapping`, `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping`.

### Q50. @RequestParam vs @PathVariable?

**Answer:** `@RequestParam` extracts query parameters (`/users?name=Alice`). `@PathVariable` extracts URI template variables (`/users/{id}`).

### Q51. What is @RequestBody?

**Answer:** Binds the HTTP request body to a method parameter. Uses `HttpMessageConverter` (e.g., `MappingJackson2HttpMessageConverter`) to deserialize JSON/XML into an object.

### Q52. What is @ResponseBody?

**Answer:** Serializes the method return value directly into the HTTP response body (using `HttpMessageConverter`). Used with `@Controller` methods; implicit in `@RestController`.

### Q53. What is ResponseEntity?

**Answer:** Represents the entire HTTP response — status, headers, and body. Gives full control:

```java
return ResponseEntity.status(201)
    .header("Location", "/users/1")
    .body(user);
```

### Q54. How does validation work in Spring MVC?

**Answer:** `@Valid` on a `@RequestBody` triggers Bean Validation (Jakarta). Errors captured in `BindingResult`. `@Validated` at class level enables method validation. Custom validators via `ConstraintValidator`. `@RestControllerAdvice` handles `MethodArgumentNotValidException` globally.

### Q55. What is @ControllerAdvice?

**Answer:** Global interceptor for controllers. Used with `@ExceptionHandler`, `@ModelAttribute`, `@InitBinder` for cross-cutting concerns. `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`.

### Q56. What is @ExceptionHandler?

**Answer:** Method-level annotation to handle exceptions from controller methods. Can be local (in controller) or global (in `@ControllerAdvice`). Maps exception → response with status, headers, body.

### Q57. HandlerInterceptor vs Filter?

**Answer:** Filters (Servlet API) run before/after the servlet, can modify request/response. Interceptors (Spring MVC) run within `DispatcherServlet`, have access to handler metadata, and support `preHandle`, `postHandle`, `afterCompletion`. Filters can't access Spring context; interceptors can.

### Q58. What is a ViewResolver?

**Answer:** Resolves logical view names to actual views. Examples: `InternalResourceViewResolver` (JSP), `ThymeleafViewResolver`, `FreeMarkerViewResolver`. `ContentNegotiatingViewResolver` picks based on `Accept` header.

### Q59. How does content negotiation work?

**Answer:** Client sends `Accept` header; server picks matching `produces` type. Configurable via `ContentNegotiationConfigurer`: URL param (`?format=json`), extension (`.json`), or header. Returns 406 if no match.

### Q60. What is @ModelAttribute?

**Answer:** Binds form data or query params to a model object. On a method parameter — populates the object. On a method — adds data to the model for all requests in the controller.

### Q61. What is @InitBinder?

**Answer:** Customizes data binding — registers `PropertyEditor`, `Converter`, or `Formatter`. Used for date formats, trimming strings, etc.

### Q62. How do you handle file uploads?

**Answer:** `MultipartFile` parameter with `@RequestParam("file")`. Configure `spring.servlet.multipart.max-file-size` and `max-request-size`. Use streaming for large files.

### Q63. What is @ResponseStatus?

**Answer:** Sets the HTTP status code for a handler method or exception class. Example: `@ResponseStatus(HttpStatus.CREATED)` on a POST handler; `@ResponseStatus(HttpStatus.NOT_FOUND)` on a custom exception.

### Q64. What are async request processing options?

**Answer:**
- `Callable<T>` — Spring MVC runs it in a task executor
- `DeferredResult<T>` — complete later from another thread
- `WebAsyncTask<T>` — Callable with timeout/callbacks
- `ResponseBodyEmitter` / `SseEmitter` — streaming
- `StreamingResponseBody` — raw streaming

### Q65. What is REST?

**Answer:** Representational State Transfer — architectural style using HTTP verbs, resource URIs, stateless communication, and standard status codes. Constraints: client-server, stateless, cacheable, uniform interface, layered, optional code-on-demand.

---

## 4. Spring Boot — 30 Questions

### Q66. What is Spring Boot?

**Answer:** Spring Boot is an opinionated extension of Spring Framework that simplifies setup via auto-configuration, starters, and embedded servers. It eliminates boilerplate XML, provides production-ready features (Actuator), and enables standalone JARs.

### Q67. What is @SpringBootApplication?

**Answer:** Meta-annotation combining `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`. Scans the annotated class's package and subpackages.

### Q68. How does auto-configuration work?

**Answer:** `@EnableAutoConfiguration` loads configuration classes listed in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 2.7+) or `spring.factories` (older). Each `@Configuration` class is guarded by `@Conditional` annotations (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc.) so it only applies when relevant.

### Q69. What are Spring Boot starters?

**Answer:** Curated dependency descriptors. E.g., `spring-boot-starter-web` pulls in Spring MVC, Jackson, Tomcat, validation. Starters simplify dependency management and are version-aligned via the Spring Boot BOM.

### Q70. What is the parent POM / BOM?

**Answer:** `spring-boot-starter-parent` provides default plugin config, dependency management (versions), and property defaults. `spring-boot-dependencies` (the BOM) can be imported with `scope=import` if you have your own parent.

### Q71. What embedded servers does Spring Boot support?

**Answer:** Tomcat (default for web), Jetty, Undertow, Netty (WebFlux), Reactor Netty. Exclude the default and add another starter to switch.

### Q72. How do you change the embedded server port?

**Answer:**
```properties
server.port=9090
```
Or programmatically: `server.port=0` (random).

### Q73. application.properties vs application.yml?

**Answer:** Both configure the app. Properties are flat `key=value`; YAML supports hierarchical structure, lists, and is more readable for complex config. Both are auto-loaded; YAML takes precedence if both exist.

### Q74. What is a profile in Spring Boot?

**Answer:** Environment-specific configuration. `application-dev.properties` / `application-prod.yml` load when the corresponding profile is active. Activate via `spring.profiles.active`.

### Q75. How do you externalize configuration?

**Answer:** Property precedence (high→low): command-line args, `SPRING_APPLICATION_JSON`, OS env vars, `application-{profile}.properties` outside JAR, `application-{profile}.properties` inside JAR, `application.properties` outside JAR, `application.properties` inside JAR, `@PropertySource`, defaults.

### Q76. What is @ConfigurationProperties?

**Answer:** Binds a group of properties to a POJO. `@ConfigurationProperties(prefix="myapp")` maps `myapp.*` to fields. Supports relaxed binding, validation (`@Validated`), and nested objects. Requires `@EnableConfigurationProperties` or `@Component`/`@ConfigurationPropertiesScan`.

### Q77. What is @EnableAutoConfiguration?

**Answer:** Enables Spring Boot's auto-configuration mechanism. Usually included via `@SpringBootApplication`. Can exclude specific auto-configs with `exclude` or `spring.autoconfigure.exclude`.

### Q78. What conditional annotations does Spring Boot provide?

**Answer:**
- `@ConditionalOnClass` / `@ConditionalOnMissingClass`
- `@ConditionalOnBean` / `@ConditionalOnMissingBean`
- `@ConditionalOnProperty`
- `@ConditionalOnResource`
- `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication`
- `@ConditionalOnExpression` (SpEL)

### Q79. What is Spring Boot Actuator?

**Answer:** Production-ready features: health, metrics, info, env, beans, mappings, loggers, thread dump, heap dump, shutdown. Endpoints exposed via `management.endpoints.web.exposure.include`. Micrometer for metrics.

### Q80. How do you enable/disable Actuator endpoints?

**Answer:**
```properties
management.endpoints.web.exposure.include=health,info,metrics
management.endpoints.web.exposure.exclude=env,beans
management.endpoint.health.show-details=when-authorized
```

### Q81. What is DevTools?

**Answer:** Development-time tooling: automatic restart on classpath change, LiveReload, sensible defaults (disable caching, verbose logging). Not included in production JARs.

### Q82. How do you create a custom starter?

**Answer:**
1. Create `autoconfigure` module with `@Configuration` classes + `AutoConfiguration.imports`
2. Create `starter` module depending on `autoconfigure`
3. Publish both

### Q83. What is Spring Initializr?

**Answer:** Web tool (`start.spring.io`) and IDE integration to generate Spring Boot projects. Choose build tool, language, Boot version, dependencies, and metadata.

### Q84. How do you exclude auto-configuration?

**Answer:**
```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```
Or in properties:
```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### Q85. What is @SpringBootTest?

**Answer:** Loads the full application context for integration tests. `webEnvironment` modes: `MOCK` (default), `RANDOM_PORT`, `DEFINED_PORT`, `NONE`.

### Q86. What is @WebMvcTest?

**Answer:** Slice test — loads only MVC components (controllers, filters, advice). Mock dependencies with `@MockBean`. Use `MockMvc` for requests.

### Q87. What is @DataJpaTest?

**Answer:** Slice test — configures JPA, embedded DB, `TestEntityManager`. Auto-rolls back transactions.

### Q88. What is @MockBean vs @Mock?

**Answer:** `@Mock` (Mockito) creates a mock — no Spring involvement. `@MockBean` (Spring Boot) creates a mock AND registers it in the Spring context, replacing any existing bean.

### Q89. What is @SpyBean?

**Answer:** Like `@MockBean` but wraps a real bean, allowing partial mocking. Use `doReturn().when()` to stub specific methods.

### Q90. How does Spring Boot handle logging?

**Answer:** Default: Logback + SLF4J. Auto-configures console output. Configure via `application.properties` (`logging.level.*`) or `logback-spring.xml` for advanced config.

### Q91. What is the Spring Boot CLI?

**Answer:** Command-line tool to run Groovy scripts, create projects, and manage dependencies. Rarely used today (Initializr + IDE preferred).

### Q92. How do you run Spring Boot in Docker?

**Answer:** Build a JAR, use `openjdk:17-jre-slim` base image, `COPY app.jar`, `ENTRYPOINT ["java","-jar","/app.jar"]`. Use multi-stage builds and layered JARs (`layertools`) for smaller images.

### Q93. What is the difference between Spring and Spring Boot?

**Answer:** Spring is the framework providing IoC, DI, MVC, etc. Spring Boot is built on Spring, adding auto-configuration, starters, embedded servers, production features, and simplified setup. Boot doesn't replace Spring — it makes it faster to use.

### Q94. How do you test a Spring Boot application?

**Answer:** Unit tests (`@ExtendWith(MockitoExtension.class)`), slice tests (`@WebMvcTest`, `@DataJpaTest`, `@JsonTest`), full integration (`@SpringBootTest`), Testcontainers for real DBs, `@TestRestTemplate` or `WebTestClient` for HTTP.

### Q95. What is @TestConfiguration?

**Answer:** A test-only `@Configuration`. Use on a static nested class in a test or a separate class. Not picked up by component scanning — must be `@Import`ed.

---

## 5. Spring Data JPA — 25 Questions

### Q96. What is JPA? Hibernate? Spring Data JPA?

**Answer:** **JPA** = Java Persistence API (specification). **Hibernate** = JPA implementation. **Spring Data JPA** = Spring abstraction over JPA, providing repository interfaces, derived queries, and pagination.

### Q97. What is the object-relational impedance mismatch?

**Answer:** The conceptual gap between object-oriented code (inheritance, references, identity) and relational DBs (tables, foreign keys, rows). JPA/Hibernate bridges this.

### Q98. What is @Entity?

**Answer:** Marks a class as a JPA entity (mapped to a table). Requires `@Id`. Fields map to columns; use `@Column` for customization.

### Q99. What is @GeneratedValue?

**Answer:** Specifies how the primary key is generated. Strategies: `AUTO` (default), `IDENTITY` (DB auto-increment), `SEQUENCE` (DB sequence), `TABLE` (table-based), `UUID`.

### Q100. Explain JPA relationships.

**Answer:**
- `@OneToOne` — 1:1
- `@OneToMany` / `@ManyToOne` — 1:N / N:1
- `@ManyToMany` — N:M (with `@JoinTable`)
- `@JoinColumn` — FK column
- `@MappedBy` — inverse side (no FK)

### Q101. What are cascade types?

**Answer:** Propagate operations from parent to child:
- `PERSIST`, `MERGE`, `REMOVE`, `REFRESH`, `DETACH`, `ALL`
- `orphanRemoval = true` — remove children detached from parent

### Q102. LAZY vs EAGER fetching?

**Answer:** `LAZY` loads association on first access (proxy). `EAGER` loads immediately. Default: `@ManyToOne`/`@OneToOne` = EAGER, `@OneToMany`/`@ManyToMany` = LAZY. **Prefer LAZY** + fetch joins for specific queries.

### Q103. What is the N+1 problem?

**Answer:** Loading N parents triggers N additional queries for each child collection. Solutions: `JOIN FETCH`, `@EntityGraph`, batch fetching (`@BatchSize`), projections, or `@Fetch(FetchMode.SUBSELECT)`.

### Q104. What is the persistence context?

**Answer:** First-level cache managed by `EntityManager`. Tracks entity state, provides dirty checking (auto-update on flush), identity guarantee (same ID → same instance), and write-behind.

### Q105. Entity lifecycle states?

**Answer:**
- **Transient** — not associated with context
- **Managed** (persistent) — tracked by context
- **Detached** — was managed, no longer tracked
- **Removed** — scheduled for deletion

### Q106. What is dirty checking?

**Answer:** Hibernate compares entity snapshots at flush time and auto-issues UPDATE for changed fields. No explicit `save()` needed for managed entities.

### Q107. What is JpaRepository?

**Answer:** Extends `CrudRepository` and `PagingAndSortingRepository`, adding JPA-specific methods: `flush()`, `saveAndFlush()`, `deleteInBatch()`, `getOne()`.

### Q108. Derived query methods?

**Answer:** Spring Data generates queries from method names. Keywords: `findBy`, `readBy`, `queryBy`, `countBy`, `deleteBy`, `existsBy`; conditions: `And`, `Or`, `Between`, `LessThan`, `Like`, `In`, `IsNull`, `OrderBy`, etc.

Example: `List<User> findByAgeGreaterThanAndNameContaining(int age, String name)`

### Q109. @Query with JPQL vs native SQL?

**Answer:** `@Query("SELECT u FROM User u WHERE ...")` — JPQL (entity-based). `@Query(value="SELECT * FROM users WHERE ...", nativeQuery=true)` — SQL (table-based). JPQL is portable; native for DB-specific features.

### Q110. What is @Param?

**Answer:** Named parameter binding in `@Query`:
```java
@Query("SELECT u FROM User u WHERE u.name = :name")
User findByName(@Param("name") String name);
```

### Q111. What is Pageable?

**Answer:** Interface for pagination + sorting. `PageRequest.of(page, size, sort)`. Repository methods can accept `Pageable` and return `Page<T>` or `Slice<T>`.

### Q112. Page vs Slice vs List?

**Answer:** `Page<T>` — includes total count (extra COUNT query). `Slice<T>` — only has-next info (no COUNT). `List<T>` — all results (no pagination metadata).

### Q113. What are projections?

**Answer:** Partial views of entities:
- **Interface-based**: `interface UserNameOnly { String getName(); }`
- **Class-based (DTO)**: `new com.example.UserDTO(u.id, u.name)`
- **Dynamic**: `@Query` returns `Object[]` or `Tuple`

### Q114. What are Specifications?

**Answer:** Type-safe, composable criteria via `Specification<T>` (Criteria API). Build dynamic queries:

```java
Specification<User> spec = (root, query, cb) ->
    cb.equal(root.get("active"), true);
repo.findAll(spec.and(hasName("Alice")));
```

### Q115. What is @EntityGraph?

**Answer:** Declares which associations to fetch eagerly for a query. Alternative to `JOIN FETCH`:

```java
@EntityGraph(attributePaths = {"orders", "orders.items"})
List<User> findAll();
```

### Q116. What is @Version (optimistic locking)?

**Answer:** Field annotated with `@Version` enables optimistic locking. On update, Hibernate includes `WHERE version = ?`. If 0 rows updated, throws `OptimisticLockException`.

### Q117. Pessimistic locking?

**Answer:** `LockModeType.PESSIMISTIC_WRITE` (`SELECT ... FOR UPDATE`). Blocks concurrent access. Use only when necessary — can cause deadlocks.

### Q118. What is Spring Data auditing?

**Answer:** Auto-populate `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy`. Enable `@EnableJpaAuditing`, implement `AuditorAware<T>`.

### Q119. What is the first-level cache?

**Answer:** The persistence context. Every entity loaded in a transaction is cached. Repeat `find()` with same ID returns the same instance without SQL.

### Q120. What is the second-level cache?

**Answer:** Optional cross-session cache (Ehcache, Hazelcast, Redis). Configure with `@Cacheable` on entity, `hibernate.cache.use_second_level_cache=true`. Requires careful invalidation.

### Q121. How do you prevent N+1?

**Answer:**
- `JOIN FETCH` in JPQL
- `@EntityGraph`
- `@BatchSize(size=N)` on entity
- `hibernate.default_batch_fetch_size`
- Projections when only some fields needed

### Q122. What is `save()` vs `saveAndFlush()`?

**Answer:** `save()` persists (or merges) but doesn't flush immediately. `saveAndFlush()` flushes the persistence context to DB synchronously (triggers SQL, constraints).

### Q123. What is `getOne()` vs `findById()`?

**Answer:** `getOne()` returns a lazy proxy (no SQL until accessed) — throws `EntityNotFoundException` on access if missing. `findById()` returns `Optional` and executes SQL immediately.

### Q124. What is `@MappedSuperclass`?

**Answer:** Base class whose fields are inherited by entities but which is not itself an entity (no table). Common for auditing (`BaseEntity` with `id`, `createdAt`).

### Q125. How do you test JPA repositories?

**Answer:** `@DataJpaTest` with `TestEntityManager`. Uses embedded H2 by default; use `@AutoConfigureTestDatabase(replace = NONE)` + Testcontainers for real DB.

---

## 6. Spring Security — 20 Questions

### Q126. What is Spring Security?

**Answer:** A framework for authentication and authorization in Spring apps. Provides filter chain, `UserDetailsService`, password encoding, CSRF, session management, method-level security, OAuth2, and more.

### Q127. Authentication vs Authorization?

**Answer:** **Authentication** = who you are (login). **Authorization** = what you can do (permissions). Spring Security handles both.

### Q128. What is SecurityFilterChain?

**Answer:** The core of Spring Security 6+ configuration. A chain of servlet filters that process authentication, authorization, CSRF, etc. Configure via `HttpSecurity`:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .anyRequest().authenticated())
        .formLogin(Customizer.withDefaults());
    return http.build();
}
```

### Q129. What is UserDetailsService?

**Answer:** Interface with `loadUserByUsername(String)` returning `UserDetails`. Implement to load users from DB, LDAP, etc. Spring uses it during authentication.

### Q130. What is PasswordEncoder?

**Answer:** Interface for hashing/verifying passwords. `BCryptPasswordEncoder` (default, recommended), `Argon2PasswordEncoder`, `Pbkdf2PasswordEncoder`, `SCryptPasswordEncoder`. Never store plain passwords.

### Q131. Why BCrypt?

**Answer:** Adaptive hashing with salt. Slow by design (resists brute force). Configurable strength (default 10). Salt embedded in hash. Industry standard.

### Q132. What is CSRF and how does Spring handle it?

**Answer:** Cross-Site Request Forgery — malicious site triggers authenticated requests. Spring enables CSRF protection by default for stateful apps: a token must be included in POST/PUT/DELETE. Disable for stateless APIs (`http.csrf(csrf -> csrf.disable())`).

### Q133. What is CORS?

**Answer:** Cross-Origin Resource Sharing — browser mechanism to allow requests to different origins. Configure in Spring Security: `http.cors(Customizer.withDefaults())` + `CorsConfigurationSource` bean.

### Q134. What is session management in Spring Security?

**Answer:** Controls session creation and fixation. Options: `SessionCreationPolicy.IF_REQUIRED`, `STATELESS` (for JWT), `ALWAYS`, `NEVER`. Session fixation protection is enabled by default (change session ID on login).

### Q135. What is HTTP Basic auth?

**Answer:** Sends `Authorization: Basic base64(user:pass)` header. Simple, but credentials sent on every request — use only over HTTPS. Enabled via `http.httpBasic()`.

### Q136. What is form login?

**Answer:** Customizable login page via `http.formLogin()`. Default `/login` page. Configure custom page, success/failure handlers.

### Q137. What is Remember-Me?

**Answer:** Persistent login token (cookie) so users stay logged in. Two approaches: hash-based (token = hash of user+password+key) and persistent (DB table). Enable via `http.rememberMe()`.

### Q138. What is `@EnableMethodSecurity`?

**Answer:** Enables method-level security (`@PreAuthorize`, `@PostAuthorize`, etc.). Replaces `@EnableGlobalMethodSecurity` (deprecated).

### Q139. What is `@PreAuthorize`?

**Answer:** Expression evaluated **before** method execution. Uses SpEL: `@PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")`. Denies with `AccessDeniedException`.

### Q140. What is `@PostAuthorize`?

**Answer:** Evaluated **after** method execution, can access return value (`returnObject`). Use for filtering results.

### Q141. `@PreFilter` / `@PostFilter`?

**Answer:** `@PreFilter` filters input collections; `@PostFilter` filters returned collections. Use SpEL and `filterObject`:

```java
@PostFilter("filterObject.owner == authentication.name")
List<Order> findAll();
```

### Q142. What is JWT?

**Answer:** JSON Web Token — compact, self-contained token with header, payload, signature. Used for stateless auth. Structure: `base64(header).base64(payload).signature`. Payload holds claims (`sub`, `exp`, `roles`).

### Q143. JWT vs opaque token?

**Answer:** JWT is self-contained (server can verify without DB), stateless, but can't be revoked easily. Opaque tokens require server lookup (revocable, but stateful). Choose based on revocation requirements.

### Q144. What is OAuth2?

**Answer:** Authorization framework enabling third-party access without sharing passwords. Roles: Resource Owner, Client, Authorization Server, Resource Server. Grant types: Authorization Code (with PKCE), Client Credentials, Refresh Token, Device Code.

### Q145. What is the Authorization Code flow?

**Answer:**
1. Client redirects user to Authorization Server
2. User authenticates and consents
3. Server redirects back with `code`
4. Client exchanges code for tokens (with PKCE)
5. Client uses access token for API calls

Preferred for web/mobile with user context.

### Q146. What is the Client Credentials flow?

**Answer:** Client authenticates directly with Authorization Server to get a token — no user. Used for machine-to-machine.

### Q147. What is Spring Authorization Server?

**Answer:** A Spring project implementing OAuth2 Authorization Server. Replacement for the deprecated `spring-security-oauth` project. Provides `/oauth2/token`, `/oauth2/authorize`, JWK endpoint, etc.

### Q148. What is a Resource Server?

**Answer:** An API that accepts access tokens and enforces scopes. In Spring: `http.oauth2ResourceServer(oauth2 -> oauth2.jwt())` with `spring.security.oauth2.resourceserver.jwt.issuer-uri`.

### Q149. What security headers should you set?

**Answer:**
- `Strict-Transport-Security` (HSTS)
- `Content-Security-Policy` (CSP)
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy`
- `Permissions-Policy`

Spring Security sets sensible defaults.

### Q150. How do you handle logout?

**Answer:** `http.logout()` — invalidates session, clears `SecurityContext`, deletes cookies (`JSESSIONID`, `remember-me`). For JWT, implement client-side token removal + optional blacklist.

---

## 7. Transactions — 15 Questions

### Q151. What is a transaction?

**Answer:** A unit of work that is atomic, consistent, isolated, and durable (ACID). Either all operations succeed or all roll back.

### Q152. What is @Transactional?

**Answer:** Declarative transaction management. Spring wraps the method in a proxy that begins/commits/rolls back based on outcome. Attributes: `propagation`, `isolation`, `timeout`, `readOnly`, `rollbackFor`, `noRollbackFor`.

### Q153. What are the propagation behaviors?

**Answer:**
- `REQUIRED` (default) — join existing or create new
- `REQUIRES_NEW` — always create new (suspends existing)
- `SUPPORTS` — join if exists, else no TX
- `NOT_SUPPORTED` — suspend existing
- `MANDATORY` — must have existing, else exception
- `NEVER` — must NOT have existing
- `NESTED` — savepoint within existing

### Q154. What are isolation levels?

**Answer:**
- `READ_UNCOMMITTED` — dirty reads allowed
- `READ_COMMITTED` — no dirty reads (default in many DBs)
- `REPEATABLE_READ` — no non-repeatable reads
- `SERIALIZABLE` — full isolation

Higher = safer but slower.

### Q155. What problems do isolation levels solve?

**Answer:**
- **Dirty read** — reading uncommitted data
- **Non-repeatable read** — same query returns different data within TX
- **Phantom read** — new rows appear in repeated range query
- **Lost update** — two TXs overwrite each other

### Q156. What is rollback behavior by default?

**Answer:** Spring rolls back on `RuntimeException` and `Error`, but NOT on checked exceptions. Use `rollbackFor = Exception.class` to include checked. Use `noRollbackFor` to exclude.

### Q157. What is readOnly = true?

**Answer:** Hint to the DB/JPA that the transaction won't modify data. Enables optimizations (no dirty checking in Hibernate, read-only DB connection). Use for read-only service methods.

### Q158. What is self-invocation and why is it a problem?

**Answer:** Calling a `@Transactional` method from another method in the same class bypasses the proxy → no transaction. Fixes: inject self, use `AopContext.currentProxy()`, or split into separate beans.

### Q159. Can @Transactional be on private methods?

**Answer:** No — Spring AOP proxies can't intercept private methods. Use public methods only (or protected for CGLIB).

### Q160. What is the difference between @Transactional on class vs method?

**Answer:** Class-level applies to all public methods. Method-level overrides class-level. Use class-level for defaults, method-level for exceptions.

### Q161. What is a transaction manager?

**Answer:** Interface `PlatformTransactionManager` with implementations: `DataSourceTransactionManager` (JDBC), `JpaTransactionManager` (JPA), `HibernateTransactionManager`, `JtaTransactionManager` (distributed). Spring Boot auto-configures based on classpath.

### Q162. What is `TransactionTemplate`?

**Answer:** Programmatic transaction management:

```java
transactionTemplate.execute(status -> {
    // do work
    return result;
});
```

Use when you need fine-grained control (e.g., dynamic TX boundaries).

### Q163. What is `@Transactional` timeout?

**Answer:** Max duration before rollback. `@Transactional(timeout = 5)` = 5 seconds. Database-specific; not guaranteed.

### Q164. Can you have nested transactions?

**Answer:** With `PROPAGATION_NESTED` (savepoints) — yes, if DB supports savepoints. Inner rollback doesn't affect outer. `REQUIRES_NEW` creates a separate, independent transaction (suspends outer).

### Q165. Common transaction pitfalls?

**Answer:**
- Self-invocation
- Private methods
- Checked exceptions not rolling back
- Long transactions (locks, timeouts)
- Catching exceptions inside `@Transactional` (rollback won't trigger)
- Mixing `REQUIRES_NEW` with outer rollback expectations
- Modifying entities outside a transaction (detached)

---

## 8. Spring Cloud & Microservices — 20 Questions

### Q166. Monolith vs microservices?

**Answer:** Monolith = single deployable, simpler initially, harder to scale teams. Microservices = independent services, independent deploy/scale, complex operations, network failures. Choose based on team size, scale needs, and domain complexity.

### Q167. What is service discovery?

**Answer:** Services register themselves and discover peers dynamically. Netflix Eureka, Consul, Kubernetes DNS. Clients query the registry instead of hardcoding URLs.

### Q168. What is an API Gateway?

**Answer:** Single entry point for clients, routing to backend services. Handles auth, rate limiting, caching, load balancing, circuit breaking. Spring Cloud Gateway (reactive), Zuul (legacy).

### Q169. What is client-side load balancing?

**Answer:** Client picks a service instance from the registry. Spring Cloud LoadBalancer (modern), Ribbon (legacy). Pairs with service discovery.

### Q170. What is OpenFeign?

**Answer:** Declarative REST client. Define an interface with `@FeignClient("service-name")` — Spring generates the implementation, integrating with service discovery and load balancing.

### Q171. What is a circuit breaker?

**Answer:** Prevents cascading failures by short-circuiting calls to failing services. States: CLOSED (normal), OPEN (fail fast), HALF_OPEN (test recovery). Resilience4j is the modern choice; Hystrix is legacy.

### Q172. What is Resilience4j?

**Answer:** Fault-tolerance library: circuit breaker, rate limiter, retry, bulkhead, time limiter. Configured via `@CircuitBreaker`, `@Retry`, `@RateLimiter`, etc.

### Q173. What is Spring Cloud Config?

**Answer:** Centralized configuration server backed by Git, Vault, JDBC, etc. Clients fetch config at startup and can refresh via `/actuator/refresh` (with Spring Cloud Bus for broadcast).

### Q174. What is Spring Cloud Bus?

**Answer:** Connects microservices via a message broker (Kafka/RabbitMQ) for broadcasting events (e.g., config refresh). Uses `@RemoteApplicationEvent`.

### Q175. What is distributed tracing?

**Answer:** Tracking a request across services. Tools: Micrometer Tracing + Zipkin, Jaeger, OpenTelemetry. Each service adds trace/span IDs to logs and reports to the collector.

### Q176. What is Spring Cloud Stream?

**Answer:** Broker-agnostic messaging abstraction. Write `Supplier`/`Function`/`Consumer` beans; bind to Kafka/RabbitMQ via binders. Swap brokers via config.

### Q177. What is Spring Cloud Contract?

**Answer:** Consumer-driven contract testing. Providers publish contracts; consumers verify against stubs generated from contracts. Ensures API compatibility.

### Q178. How do you handle distributed transactions?

**Answer:** Avoid them if possible. Use **Saga** pattern (choreography or orchestration) with compensating transactions. Or **outbox pattern** for DB+broker atomicity. XA is heavy and rarely used.

### Q179. What is the Saga pattern?

**Answer:** Distributed transaction as a sequence of local transactions. On failure, compensating transactions undo prior steps. Two styles: **choreography** (events) and **orchestration** (central coordinator).

### Q180. What is the outbox pattern?

**Answer:** Write business data and an outbox event in the same DB transaction. A poller publishes events to the broker and marks them as published. Solves dual-write inconsistency.

### Q181. What is the circuit breaker state machine?

**Answer:**
- CLOSED → calls pass through, monitor failures
- Failure rate > threshold → OPEN → fail fast
- After wait duration → HALF_OPEN → allow test calls
- Success → CLOSED; failure → OPEN

### Q182. What is API composition?

**Answer:** Aggregating data from multiple services in a single API. Simple alternative to CQRS. Drawback: in-memory joins, latency.

### Q183. What is CQRS?

**Answer:** Command Query Responsibility Segregation — separate models for writes (commands) and reads (queries). Often paired with event sourcing. Improves scalability and read performance.

### Q184. What is eventual consistency?

**Answer:** In distributed systems, replicas converge over time (not immediately). Trade-off for availability/partition tolerance (CAP). Handle with retries, idempotency, and versioning.

### Q185. What is the CAP theorem?

**Answer:** A distributed system can guarantee at most two of: **C**onsistency, **A**vailability, **P**artition tolerance. In practice, P is required, so choose CP (strong consistency) or AP (availability).

---

## 9. Spring WebFlux — 10 Questions

### Q186. What is reactive programming?

**Answer:** Programming with asynchronous data streams and non-blocking backpressure. Values pushed over time; consumers control flow. Foundation: Reactive Streams spec.

### Q187. What is Project Reactor?

**Answer:** Reactive library for JVM. Types: `Mono<T>` (0 or 1 value), `Flux<T>` (0..N values). Operators: `map`, `flatMap`, `filter`, `zip`, `merge`, `concat`. Schedulers control threading.

### Q188. What is backpressure?

**Answer:** Consumer signals producer how many items it can handle. Prevents overwhelming the consumer. Reactor supports: `onBackpressureBuffer`, `onBackpressureDrop`, `onBackpressureLatest`, and request-based flow control.

### Q189. WebFlux vs Spring MVC?

**Answer:** MVC = servlet, blocking, thread-per-request. WebFlux = reactive, non-blocking, event-loop (Netty). WebFlux scales better for I/O-bound workloads; MVC is simpler for CPU-bound or blocking code.

### Q190. What is WebClient?

**Answer:** Reactive HTTP client (replaces blocking `RestTemplate`). Returns `Mono`/`Flux`. Supports streaming, retries, timeouts, filters.

### Q191. What is R2DBC?

**Answer:** Reactive Relational Database Connectivity — non-blocking SQL access. Alternative to JDBC (blocking). Supports PostgreSQL, MySQL, H2, etc. Use with Spring Data R2DBC.

### Q192. Functional vs annotated WebFlux?

**Answer:** Annotated — same `@Controller` style as MVC but returns `Mono`/`Flux`. Functional — `RouterFunction` + `HandlerFunction` for explicit routing.

### Q193. What are Schedulers?

**Answer:** Thread pools for reactive execution. `Schedulers.boundedElastic()` for blocking calls, `Schedulers.parallel()` for CPU work, `Schedulers.single()` for one thread. `subscribeOn` affects source; `publishOn` affects downstream.

### Q194. How do you test WebFlux?

**Answer:** `StepVerifier` for `Mono`/`Flux`, `WebTestClient` for endpoints, `@WebFluxTest` slice. Testcontainers for real DBs.

### Q195. What is RSocket?

**Answer:** Binary, reactive protocol supporting request/response, fire-and-forget, request/stream, and channel. Works over TCP, WebSocket, Aeron. Spring supports it via `spring-boot-starter-rsocket`.

---

## 10. Maven & Gradle — 15 Questions

### Q196. What is Maven?

**Answer:** Build automation and dependency management tool. Uses `pom.xml`, follows convention-over-configuration, has lifecycles/phases/goals.

### Q197. What are Maven coordinates?

**Answer:** `groupId:artifactId:version[:packaging[:classifier]]`. Example: `org.springframework:spring-core:6.1.0`.

### Q198. What are Maven lifecycles?

**Answer:** **default** (compile, test, package, install, deploy), **clean**, **site**. Phases execute in order; goals bind to phases.

### Q199. What are the default lifecycle phases?

**Answer:** `validate → initialize → generate-sources → process-sources → generate-resources → process-resources → compile → process-classes → generate-test-sources → process-test-sources → generate-test-resources → process-test-resources → test-compile → process-test-classes → test → prepare-package → package → pre-integration-test → integration-test → post-integration-test → verify → install → deploy`

### Q200. Goals vs phases?

**Answer:** Phases are lifecycle steps (`compile`, `package`). Goals are plugin tasks (`compiler:compile`, `surefire:test`). Binding a goal to a phase runs it when that phase executes.

### Q201. What is the difference between `install` and `deploy`?

**Answer:** `install` copies the artifact to the local repo (`~/.m2/repository`). `deploy` uploads to a remote repo (Nexus, Artifactory).

### Q202. What are dependency scopes?

**Answer:**
- `compile` (default) — everywhere
- `provided` — compile + test, not packaged (servlet API)
- `runtime` — runtime + test (JDBC drivers)
- `test` — test only
- `system` — local path (avoid)
- `import` — import BOM in `dependencyManagement`

### Q203. What is a BOM?

**Answer:** Bill of Materials — a POM with `dependencyManagement` listing versions. Import with `scope=import` to align versions without a parent.

### Q204. What is the parent POM?

**Answer:** A POM that provides inheritance: dependencies, plugins, properties, dependencyManagement. `spring-boot-starter-parent` is common.

### Q205. What are Maven profiles?

**Answer:** Conditional build configs (dev, prod). Activated by property, JDK, OS, file existence, or `-PprofileName`.

### Q206. What is `mvn dependency:tree`?

**Answer:** Prints the dependency tree. Helps diagnose conflicts and transitive deps. Add `-Dincludes=groupId:artifactId` to filter.

### Q207. How do you resolve dependency conflicts?

**Answer:** Maven uses **nearest-wins**: the version closest to the root wins. Use `<exclusions>` to remove transitive deps, `dependencyManagement` to enforce versions, or `mvn dependency:tree` to diagnose.

### Q208. Multi-module project structure?

**Answer:** Aggregator POM (packaging `pom`) lists `<modules>`. Child modules inherit from parent. Build all with one command; build one with `-pl module -am`.

### Q209. Gradle vs Maven?

**Answer:** Gradle: Groovy/Kotlin DSL, incremental builds, faster for large projects, flexible. Maven: XML, rigid lifecycle, mature, widely used. Spring Boot supports both; Gradle often used for Android and modern JVM projects.

### Q210. What is the Maven wrapper (`mvnw`)?

**Answer:** Script + JAR that downloads and runs a specific Maven version. Ensures reproducible builds without installing Maven. Committed to the repo.

---

## 11. Scenario-Based Questions — 20 Questions

### Q211. Your Spring Boot app starts slowly. How do you diagnose?

**Answer:**
1. Enable `--debug` and check `AutoConfigurationReport`
2. Use `spring-boot-starter-actuator` `/startup` endpoint (Boot 3.1+)
3. `-XX:+FlightRecorder` / JFR for JVM profiling
4. Check for `@Lazy` misuse, heavy `@PostConstruct`, or eager bean chains
5. Profile component scan base packages
6. Reduce classpath (`spring-context-indexer`)

### Q212. A REST endpoint returns 500 with no details. How do you debug?

**Answer:**
1. Check server logs for stack trace
2. Add `@RestControllerAdvice` with `@ExceptionHandler(Exception.class)`
3. Enable `server.error.include-message=always` (dev only)
4. Add correlation ID filter and trace
5. Reproduce in `@WebMvcTest` or with Postman

### Q213. You get `NoUniqueBeanDefinitionException`. What now?

**Answer:**
1. Multiple beans of the same type. Fix with `@Primary` on one, or `@Qualifier("beanName")` at the injection point.
2. Or disable one via `@Profile`.
3. Inspect with `ctx.getBeansOfType(MyType.class)`.

### Q214. A `@Transactional` method isn't rolling back. Why?

**Answer:**
1. Checked exception thrown? Add `rollbackFor = Exception.class`.
2. Self-invocation bypassing proxy? Move to another bean.
3. Exception caught inside the method? Re-throw or use `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.
4. Private method? Make it public.

### Q215. You see `LazyInitializationException`. How to fix?

**Answer:**
1. Access a lazy collection outside the transaction.
2. Fixes: `JOIN FETCH`, `@EntityGraph`, `@Transactional` on the caller, DTO projection, or `OpenSessionInView` (anti-pattern).

### Q216. Your app is slow due to N+1 queries. How do you fix?

**Answer:**
1. Enable SQL logging (`spring.jpa.show-sql=true`, `logging.level.org.hibernate.SQL=DEBUG`)
2. Identify N+1 in logs
3. Apply `JOIN FETCH` or `@EntityGraph`
4. Use projections for partial data
5. Consider `@BatchSize`

### Q217. A microservice call times out. How do you handle it?

**Answer:**
1. Set timeouts on the client (`Feign`, `RestTemplate`, `WebClient`)
2. Add circuit breaker (Resilience4j) — fail fast, fallback
3. Retry with exponential backoff for transient errors
4. Bulkhead to isolate thread pools
5. Monitor and alert

### Q218. You need to process 10M rows nightly. Design.

**Answer:**
1. Spring Batch with chunk processing (500–1000)
2. `JdbcPagingItemReader` for DB; `FlatFileItemWriter` for output
3. Partition by ID range across threads
4. Use `JdbcBatchItemWriter` for batch inserts
5. Persist JobRepository in DB for restart
6. Schedule with Quartz

### Q219. How do you secure a REST API?

**Answer:**
1. HTTPS + HSTS
2. OAuth2/JWT with short-lived access tokens + refresh
3. Validate tokens (signature, issuer, audience, expiry)
4. Method-level `@PreAuthorize`
5. Input validation (`@Valid`)
6. DTOs (no mass assignment)
7. Rate limiting
8. Security headers (CSP, X-Frame-Options)
9. Audit logging
10. Dependency scanning

### Q220. Your Spring app runs out of memory. Diagnose.

**Answer:**
1. Enable heap dump (`-XX:+HeapDumpOnOutOfMemoryError`)
2. Analyze with Eclipse MAT / VisualVM
3. Common causes: unbounded caches, lazy-init leaks, large result sets, ThreadLocal leaks
4. Add `@Cacheable` with size limits; use `Page` for queries; close resources

### Q221. How do you implement rate limiting per API key?

**Answer:**
1. Bucket4j with Redis backend for distributed
2. `HandlerInterceptor` or filter
3. Key = `X-API-Key` header (not IP behind proxy)
4. Return `429` with `Retry-After` and rate headers
5. Different limits per tier (free/paid)

### Q222. You need to update config in running services. How?

**Answer:**
1. Spring Cloud Config Server + Spring Cloud Bus
2. Push change to Git → webhook triggers `/actuator/busrefresh`
3. Beans with `@RefreshScope` re-created
4. Or use Kubernetes ConfigMap + restart

### Q223. How do you handle DB schema migrations?

**Answer:** Flyway or Liquibase. Versioned scripts (`V1__init.sql`, `V2__add_col.sql`) run at startup. Integrated with Spring Boot via `spring-boot-starter-flyway`/`-liquibase`.

### Q224. A deployment breaks. How do you roll back?

**Answer:**
1. Blue-green or canary deployments
2. Keep previous JAR/image tagged
3. Roll back DB migrations carefully (Flyway `undo` or manual)
4. Feature flags to disable broken features without redeploy

### Q225. How do you optimize a slow JPA query?

**Answer:**
1. `EXPLAIN ANALYZE` in DB
2. Add indexes on filter/join columns
3. Avoid `SELECT *` — use projections
4. Use `JOIN FETCH` or `@EntityGraph` to prevent N+1
5. Paginate large results
6. Consider `@Query(nativeQuery=true)` for complex queries
7. Cache immutable reference data

### Q226. Your tests are slow. How do you speed them up?

**Answer:**
1. Use slice tests (`@WebMvcTest`, `@DataJpaTest`) instead of `@SpringBootTest`
2. Reuse context with same configuration
3. Use Testcontainers with singleton pattern
4. Parallel test execution (JUnit 5)
5. Mock external calls (`@MockBean`, WireMock)
6. Avoid `@DirtiesContext`

### Q227. How do you implement idempotent POST?

**Answer:**
1. Client sends `Idempotency-Key` header (UUID)
2. Server stores key + result in DB with unique constraint
3. On duplicate, return stored result (200) instead of creating again (201)
4. TTL for keys (24h)

### Q228. Service A calls B, which calls C. C fails. What happens?

**Answer:**
1. Without protection: A's thread blocks → B's thread blocks → cascading failure
2. With circuit breaker: B fails fast, fallback returns, A gets partial response
3. With timeout: B gives up, A gets timeout response
4. With bulkhead: limited threads, other requests unaffected
5. With retry: transient failures retried with backoff

### Q229. How do you handle secrets in Spring Boot?

**Answer:**
1. Never commit secrets to Git
2. Use environment variables or Vault/AWS Secrets Manager
3. Spring Cloud Vault integration
4. `spring.config.import=vault://`
5. Jasypt for encrypted properties
6. Kubernetes Secrets + `envFrom`

### Q230. How do you trace a request across microservices?

**Answer:**
1. Micrometer Tracing + Zipkin/Jaeger/OTel
2. Auto-propagates `traceId`/`spanId` via headers
3. Logs include `traceId`/`spanId` (MDC)
4. Correlate in Zipkin UI
5. Alert on error spans

---

## 12. Coding Problems — 10 Problems

### Problem 1: Write a REST controller for CRUD.

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {
    
    private final ProductService service;
    
    public ProductController(ProductService service) {
        this.service = service;
    }
    
    @GetMapping
    public Page<ProductDTO> list(@PageableDefault(size = 20) Pageable pageable) {
        return service.findAll(pageable);
    }
    
    @GetMapping("/{id}")
    public ProductDTO get(@PathVariable Long id) {
        return service.findById(id);
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<ProductDTO> create(@Valid @RequestBody CreateProductRequest req,
                                             UriComponentsBuilder uri) {
        ProductDTO dto = service.create(req);
        return ResponseEntity.created(
            uri.path("/api/v1/products/{id}").buildAndExpand(dto.getId()).toUri())
            .body(dto);
    }
    
    @PutMapping("/{id}")
    public ProductDTO update(@PathVariable Long id,
                             @Valid @RequestBody UpdateProductRequest req) {
        return service.update(id, req);
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        service.delete(id);
    }
}
```

### Problem 2: Configure Spring Security with JWT.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtFilter;
    
    public SecurityConfig(JwtAuthenticationFilter jwtFilter) {
        this.jwtFilter = jwtFilter;
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(Customizer.withDefaults())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new BearerTokenAuthenticationEntryPoint())
                .accessDeniedHandler(new BearerTokenAccessDeniedHandler()));
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Problem 3: Implement a JWT filter.

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    
    public JwtAuthenticationFilter(JwtService jwtService,
                                   UserDetailsService uds) {
        this.jwtService = jwtService;
        this.userDetailsService = uds;
    }
    
    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain)
            throws ServletException, IOException {
        String header = req.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(req, res);
            return;
        }
        
        String token = header.substring(7);
        try {
            String username = jwtService.extractUsername(token);
            if (username != null &&
                SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails user = userDetailsService.loadUserByUsername(username);
                if (jwtService.isValid(token, user)) {
                    var auth = new UsernamePasswordAuthenticationToken(
                        user, null, user.getAuthorities());
                    auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(req));
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

### Problem 4: Global exception handler.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Resource Not Found");
        return pd;
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Validation Failed");
        ex.getBindingResult().getFieldErrors().forEach(err ->
            pd.setProperty(err.getField(), err.getDefaultMessage()));
        return pd;
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ProblemDetail handleAll(Exception ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setTitle("Internal Error");
        pd.setDetail("An unexpected error occurred");
        return pd;
    }
}
```

### Problem 5: JPA entity with relationships + auditing.

```java
@Entity
@Table(name = "orders")
@EntityListeners(AuditingEntityListener.class)
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    private BigDecimal total;
    
    @CreatedDate
    @Column(updatable = false)
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant updatedAt;
    
    @Version
    private Long version;
    
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
    
    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
    
    // getters/setters
}
```

### Problem 6: Repository with derived queries + custom JPQL.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    Optional<User> findByEmailIgnoreCase(String email);
    
    List<User> findByAgeGreaterThanEqualAndActiveTrue(int age);
    
    @Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
    Optional<User> findByIdWithOrders(@Param("id") Long id);
    
    @Query("SELECT new com.example.dto.UserSummary(u.id, u.name, COUNT(o)) " +
           "FROM User u LEFT JOIN u.orders o GROUP BY u.id, u.name")
    List<UserSummary> summarize();
    
    @Modifying
    @Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :cutoff")
    int deactivateInactive(@Param("cutoff") Instant cutoff);
    
    @EntityGraph(attributePaths = {"roles"})
    Page<User> findAllByActiveTrue(Pageable pageable);
}
```

### Problem 7: Circuit breaker with Resilience4j.

```java
@Service
public class PaymentService {
    
    private final RestClient restClient;
    
    public PaymentService(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("http://payment-service").build();
    }
    
    @CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
    @Retry(name = "payment")
    @TimeLimiter(name = "payment")
    public CompletableFuture<PaymentResponse> charge(PaymentRequest req) {
        return CompletableFuture.supplyAsync(() ->
            restClient.post().uri("/charge").body(req)
                .retrieve().body(PaymentResponse.class));
    }
    
    public CompletableFuture<PaymentResponse> paymentFallback(
            PaymentRequest req, Throwable t) {
        return CompletableFuture.completedFuture(
            PaymentResponse.failed("Payment service unavailable"));
    }
}
```

**application.yml:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
  retry:
    instances:
      payment:
        max-attempts: 3
        wait-duration: 1s
        enable-exponential-backoff: true
```

### Problem 8: Kafka producer + consumer.

```java
@Service
public class OrderEventProducer {
    
    private final KafkaTemplate<String, OrderEvent> kafka;
    
    public OrderEventProducer(KafkaTemplate<String, OrderEvent> kafka) {
        this.kafka = kafka;
    }
    
    public void publish(OrderEvent event) {
        kafka.send("order-events", event.getOrderId().toString(), event)
             .whenComplete((result, ex) -> {
                 if (ex != null) {
                     log.error("Publish failed: {}", ex.getMessage());
                 }
             });
    }
}

@Component
public class OrderEventConsumer {
    
    @KafkaListener(topics = "order-events", groupId = "order-processors")
    public void onOrderCreated(OrderEvent event,
                               @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                               Acknowledgment ack) {
        try {
            process(event);
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Failed on partition {}: {}", partition, e.getMessage());
            throw e;   // triggers retry/DLT
        }
    }
}
```

### Problem 9: WebFlux reactive endpoint.

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {
    
    private final ProductService service;
    
    public ProductController(ProductService service) {
        this.service = service;
    }
    
    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    public Flux<ProductDTO> list() {
        return service.findAll();
    }
    
    @GetMapping("/{id}")
    public Mono<ProductDTO> get(@PathVariable Long id) {
        return service.findById(id)
            .switchIfEmpty(Mono.error(new ResourceNotFoundException("Product", id)));
    }
    
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ProductDTO> stream() {
        return service.stream()
            .delayElements(Duration.ofMillis(500));
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<ProductDTO> create(@Valid @RequestBody Mono<CreateProductRequest> req) {
        return req.flatMap(service::create);
    }
}
```

### Problem 10: Spring Batch job.

```java
@Configuration
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
    
    @Bean
    public FlatFileItemReader<User> reader() {
        return new FlatFileItemReaderBuilder<User>()
            .name("userReader")
            .resource(new ClassPathResource("users.csv"))
            .delimited()
            .names("id", "name", "email")
            .targetType(User.class)
            .build();
    }
    
    @Bean
    public ItemProcessor<User, User> processor() {
        return user -> user.getEmail().endsWith("@example.com") ? user : null;
    }
    
    @Bean
    public JpaItemWriter<User> writer(EntityManagerFactory emf) {
        JpaItemWriter<User> w = new JpaItemWriter<>();
        w.setEntityManagerFactory(emf);
        return w;
    }
}
```

---

## 13. Quick Reference

### Annotation Frequency (Most Asked)

| Annotation | Topic |
|-----------|-------|
| `@Autowired` | DI |
| `@Component` / `@Service` / `@Repository` / `@Controller` | Stereotypes |
| `@Configuration` / `@Bean` | Java config |
| `@SpringBootApplication` | Boot |
| `@Transactional` | Transactions |
| `@RestController` / `@RequestMapping` | Web |
| `@PreAuthorize` | Security |
| `@Entity` / `@Id` | JPA |
| `@Value` / `@ConfigurationProperties` | Config |
| `@Profile` | Environments |

### Common Interview Themes

1. **IoC/DI** — why constructor injection
2. **Bean lifecycle** — order of callbacks
3. **AOP** — proxy mechanisms, self-invocation
4. **Auto-configuration** — how it works
5. **Transactions** — propagation, self-invocation
6. **N+1 problem** — detection and fix
7. **Security** — filter chain, JWT
8. **Microservices** — circuit breaker, discovery

### Answer Template

For any "how does X work" question:
1. **Definition** — one sentence
2. **Mechanism** — step-by-step
3. **Example** — brief code or scenario
4. **Trade-offs / pitfalls** — when to use/avoid
5. **Related** — cross-reference other concepts

### Red Flags Interviewers Watch For

- Field injection defended as "fine"
- Doesn't know self-invocation breaks `@Transactional`
- Can't explain auto-configuration
- Uses `EAGER` everywhere
- Doesn't understand N+1
- Thinks `@Component` == `@Configuration`
- Can't explain `@Transactional` rollback rules
- No awareness of proxy mechanisms

### Green Flags

- Constructor injection with `final` fields
- Knows bean lifecycle order
- Explains `@Configuration` CGLIB behavior
- Uses `@ConfigurationProperties` for grouped config
- Prefers DTOs, not entities, in APIs
- Understands propagation and self-invocation
- Knows `@PreAuthorize` and method security
- Discusses trade-offs (not just "best practice")
- Mentions observability (metrics, tracing)

---

## Cross-References

- **Previous file:** `23_Spring_Web_Services.md`
- **Next file:** `25_Spring_Project_Architecture.md`
- **All source files:** `01`–`23` for detailed explanations

---

## Practice Plan

**Week 1:** Core Spring (Q1–Q30), AOP (Q31–Q45)
**Week 2:** MVC (Q46–Q65), Boot (Q66–Q95)
**Week 3:** JPA (Q96–Q125), Security (Q126–Q150)
**Week 4:** Transactions (Q151–Q165), Microservices (Q166–Q185)
**Week 5:** WebFlux (Q186–Q195), Maven (Q196–Q210)
**Week 6:** Scenarios (Q211–Q230), Coding (P1–P10)

**Daily:** Answer 10 questions aloud, then check against this file. Code one problem per day.

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [23_Spring_Web_Services.md](./23_Spring_Web_Services.md)
- **Next →:** [25_Spring_Project_Architecture.md](./25_Spring_Project_Architecture.md)
- **Related:** [25_Spring_Project_Architecture.md](./25_Spring_Project_Architecture.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md), [27_Spring_Cheat_Sheet.md](./27_Spring_Cheat_Sheet.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
