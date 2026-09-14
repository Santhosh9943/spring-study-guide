# Spring Framework Core: IoC, DI & Bean Lifecycle

> **File:** `01_Spring_Framework_Core.md`
> **Part:** 1 — Foundations
> **Prerequisites:** Basic Java, OOP, interfaces, generics, annotations
> **Estimated Study Time:** 12–16 hours (read + code + practice)

---

## Table of Contents

1. [What is Spring Framework?](#1-what-is-spring-framework)
2. [History & Evolution](#2-history--evolution)
3. [Problems Spring Solves](#3-problems-spring-solves)
4. [Inversion of Control (IoC)](#4-inversion-of-control-ioc)
5. [Dependency Injection (DI)](#5-dependency-injection-di)
6. [The IoC Container: BeanFactory vs ApplicationContext](#6-the-ioc-container-beanfactory-vs-applicationcontext)
7. [Bean Definition & Naming](#7-bean-definition--naming)
8. [Java-Based Configuration](#8-java-based-configuration)
9. [Annotation-Based Configuration](#9-annotation-based-configuration)
10. [Component Scanning](#10-component-scanning)
11. [Bean Scopes](#11-bean-scopes)
12. [Bean Lifecycle](#12-bean-lifecycle)
13. [BeanPostProcessor & BeanFactoryPostProcessor](#13-beanpostprocessor--beanfactorypostprocessor)
14. [Spring Expression Language (SpEL)](#14-spring-expression-language-spel)
15. [Spring Profiles](#15-spring-profiles)
16. [External Properties & Environment](#16-external-properties--environment)
17. [Circular Dependencies](#17-circular-dependencies)
18. [@Lazy, @Primary, @Qualifier](#18-lazy-primary-qualifier)
19. [Interview Questions](#19-interview-questions)
20. [Summary Cheat Sheet](#20-summary-cheat-sheet)

---

## 1. What is Spring Framework?

**Spring Framework** is a lightweight, open-source, comprehensive application framework for Java. Its core purpose is to **simplify enterprise Java development** by providing infrastructure support so developers can focus on business logic.

> **Official definition:** "Spring makes programming Java quicker, easier, and safer for everybody. Spring's focus on speed, simplicity, and productivity has made it the world's most popular Java framework."

### Key Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Lightweight** | No heavyweight containers like EJB; POJO-based |
| **Non-invasive** | Your classes don't need to extend Spring classes or implement Spring interfaces |
| **Modular** | Use only what you need (core, web, data, security, etc.) |
| **POJO-based** | Plain Old Java Objects — no special requirements |
| **Dependency Injection** | Core mechanism to wire objects |
| **AOP support** | Cross-cutting concerns separated cleanly |
| **Portable** | Works with any Java EE/Jakarta EE server or standalone |
| **Testable** | POJOs are easy to unit test; test context framework |

### Core Modules (High-Level)

```
┌─────────────────────────────────────────────────────────┐
│                    SPRING FRAMEWORK                      │
├─────────────────────────────────────────────────────────┤
│  WEB          │  DATA ACCESS  │  INTEGRATION  │  TESTING │
│  (MVC, WebFlux)│ (JDBC, ORM, TX)│ (JMS, JCA)   │          │
├─────────────────────────────────────────────────────────┤
│              AOP + ASPECTS + INSTRUMENTATION             │
├─────────────────────────────────────────────────────────┤
│                     CORE CONTAINER                       │
│           (Beans, Context, SpEL, IoC, DI)                │
└─────────────────────────────────────────────────────────┘
```

### Spring Projects (Ecosystem)

- **Spring Boot** — rapid application development with auto-config
- **Spring Cloud** — microservices & distributed systems
- **Spring Security** — authentication & authorization
- **Spring Data** — unified data access (JPA, MongoDB, Redis…)
- **Spring Batch** — batch processing
- **Spring Integration** — enterprise integration patterns
- **Spring WebFlux** — reactive web
- **Spring Session** — clustered session management

---

## 2. History & Evolution

| Version | Year | Highlights |
|---------|------|-----------|
| Spring 1.0 | 2004 | XML-based IoC, AOP, JdbcTemplate |
| Spring 2.0 | 2006 | XML namespaces, @AspectJ, @Required |
| Spring 2.5 | 2007 | **Annotation-driven config** (@Autowired, @Component) |
| Spring 3.0 | 2009 | **JavaConfig** (@Configuration, @Bean), SpEL, REST support |
| Spring 4.0 | 2013 | Java 8 support, WebSocket, conditional beans |
| Spring 5.0 | 2017 | **Reactive (WebFlux)**, Kotlin support, Java 9+ |
| Spring 5.3 | 2020 | Long-term support, Java 17 support |
| **Spring 6.0** | 2022 | **Jakarta EE 9+**, Java 17 baseline, AOT, native images |
| **Spring 6.1** | 2023 | Virtual threads, RestClient, JDK 21 |
| **Spring 6.2** | 2024 | Improved AOT, observability |
| **Spring 7.0** | 2025 | Jakarta EE 11, Java 17–25, further native improvements |

> **Key shift:** Spring 6.x moved from `javax.*` to `jakarta.*` namespace. If you see `javax.persistence`, it's Spring 5 or older.

---

## 3. Problems Spring Solves

### Problem 1: Tight Coupling

**Without Spring:**

```java
public class OrderService {
    // Hard-coded dependency — tightly coupled
    private PaymentService paymentService = new StripePaymentService();
    //                                        ^^^^^^^^^^^^^^^^^^^^^^^
    // Cannot swap implementations without editing code
    
    public void placeOrder(Order order) {
        paymentService.charge(order.getTotal());
    }
}
```

**With Spring:**

```java
@Service
public class OrderService {
    private final PaymentService paymentService;  // interface!
    
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;     // injected
    }
    
    public void placeOrder(Order order) {
        paymentService.charge(order.getTotal());
    }
}
```

### Problem 2: Boilerplate Code

Raw JDBC (~15 lines), Spring JdbcTemplate (~2 lines):

```java
// Raw JDBC
Connection conn = null;
PreparedStatement ps = null;
ResultSet rs = null;
try {
    conn = dataSource.getConnection();
    ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
    ps.setLong(1, id);
    rs = ps.executeQuery();
    if (rs.next()) { /* map */ }
} catch (SQLException e) {
    // handle
} finally {
    // close all — 6 lines
}

// Spring
User user = jdbcTemplate.queryForObject(
    "SELECT * FROM users WHERE id = ?", 
    userRowMapper, id);
```

### Problem 3: Cross-Cutting Concerns

Logging, security, transactions scatter across code. **AOP** centralizes them.

### Problem 4: Testability

Constructor injection + interfaces = trivial mocking.

### Problem 5: Configuration Chaos

Externalized config, profiles, environment abstraction.

### Problem 6: Integration Complexity

Consistent abstractions over JPA, JMS, Kafka, Redis, etc.

---

## 4. Inversion of Control (IoC)

### Definition

**IoC** is a design principle where the **control of object creation and lifecycle is inverted** — moved from the application code to a container/framework.

> **Hollywood Principle:** "Don't call us, we'll call you."

### Traditional Control Flow

```
Application Code
    │
    ├── new ServiceA()
    ├── new RepositoryA()
    └── serviceA.setRepository(repoA)
```

**You** control creation.

### IoC Flow

```
Application Code
    │
    └── "I need ServiceA"
            │
            ▼
      IoC Container
            │
            ├── creates RepositoryA
            ├── creates ServiceA
            ├── injects RepositoryA into ServiceA
            └── hands back ServiceA
```

**Container** controls creation.

### Types of IoC

| Type | Example |
|------|---------|
| **Dependency Injection** | Spring's primary mechanism |
| **Dependency Lookup** | JNDI, `context.getBean()` |
| **Template Method** | JdbcTemplate, RestTemplate |
| **Strategy Pattern** | Pluggable implementations |
| **Factory Pattern** | BeanFactory |

### Benefits of IoC

- **Loose coupling** — depend on interfaces, not implementations
- **Testability** — inject mocks easily
- **Configurability** — swap implementations via config
- **Lifecycle management** — container handles init/destroy
- **Reusability** — components are portable
- **Separation of concerns** — business logic vs wiring

---

## 5. Dependency Injection (DI)

**DI** is the concrete implementation of IoC: the container **injects dependencies** into an object rather than the object creating them.

### The Three Types of Injection

#### 5.1 Constructor Injection (RECOMMENDED)

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    // Single constructor → @Autowired optional (Spring 4.3+)
    public UserService(UserRepository userRepository, 
                       EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
    
    public User createUser(String name, String email) {
        User user = new User(name, email);
        userRepository.save(user);
        emailService.sendWelcome(email);
        return user;
    }
}
```

**Advantages:**
- ✅ Immutable (fields can be `final`)
- ✅ Guaranteed dependencies at construction
- ✅ Fails fast on missing dependencies
- ✅ Easy to unit test — just pass mocks to constructor
- ✅ No Spring dependency in the class itself
- ✅ Detects circular dependencies at startup

**When to use:** **Always** for required dependencies.

#### 5.2 Setter Injection

```java
@Service
public class UserService {
    private UserRepository userRepository;
    
    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

**Advantages:**
- ✅ Allows optional dependencies
- ✅ Allows re-configuration after construction

**Disadvantages:**
- ❌ Mutable state
- ❌ Cannot use `final`
- ❌ Object can exist in a partially-constructed state

**When to use:** Optional dependencies, legacy code, circular dependencies (last resort).

#### 5.3 Field Injection (AVOID)

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;   // ⚠️ reflection-based
}
```

**Disadvantages:**
- ❌ Cannot be `final`
- ❌ Hidden dependencies (not visible in constructor)
- ❌ Hard to test without Spring/reflection
- ❌ Encourages too many dependencies
- ❌ NullPointerException risk if not wired

**When to use:** Never in production code. Only for quick prototypes.

### Injection Comparison Table

| Aspect | Constructor | Setter | Field |
|--------|-------------|--------|-------|
| Immutability | ✅ | ❌ | ❌ |
| Required deps guaranteed | ✅ | ❌ | ❌ |
| Testability | ✅ Easy | ⚠️ Medium | ❌ Reflection |
| Circular dep detection | ✅ Startup fail | ⚠️ Possible | ⚠️ Possible |
| Spring-independent | ✅ | ✅ | ❌ |
| IDE support | ✅ | ✅ | ✅ |
| Recommended | ✅ **Yes** | Optional only | ❌ No |

### @Autowired Resolution Order

1. **By type** — matches beans assignable to the type
2. If multiple matches → **@Primary** bean wins
3. If still ambiguous → **@Qualifier("name")**
4. If still ambiguous → **by field/parameter name** (matches bean name)
5. Otherwise → `NoUniqueBeanDefinitionException`

### @Autowired Applicable Targets

```java
@Autowired  // 1. Constructor
public UserService(UserRepository repo) { ... }

@Autowired  // 2. Setter
public void setRepo(UserRepository repo) { ... }

@Autowired  // 3. Field
private UserRepository repo;

@Autowired  // 4. Arbitrary method
public void configure(DataSource ds, CacheManager cm) { ... }

@Autowired  // 5. Array/Collection — injects all matching beans
private List<PaymentProcessor> processors;

@Autowired  // 6. Map — key = bean name, value = bean
private Map<String, PaymentProcessor> processorMap;

@Autowired(required = false)  // optional dependency
private MetricsCollector metrics;
```

### JSR-330 Alternative: @Inject

```java
import jakarta.inject.Inject;   // Spring 6+, previously javax.inject

@Service
public class UserService {
    private final UserRepository repo;
    
    @Inject   // equivalent to @Autowired
    public UserService(UserRepository repo) { this.repo = repo; }
}
```

| Feature | @Autowired | @Inject |
|---------|-----------|---------|
| Standard | Spring | JSR-330 |
| `required` attribute | ✅ | ❌ |
| Named qualifier | `@Qualifier` | `@Named` |

---

## 6. The IoC Container: BeanFactory vs ApplicationContext

### What is the IoC Container?

The container is responsible for:
1. **Reading** configuration metadata (XML, annotations, Java config)
2. **Instantiating** beans
3. **Wiring** dependencies
4. **Managing** lifecycle (init, destroy)
5. **Providing** beans on request

### Two Container Interfaces

#### 6.1 BeanFactory (Basic)

```java
Resource res = new ClassPathResource("beans.xml");
BeanFactory factory = new XmlBeanFactory(res);   // deprecated
MyBean bean = factory.getBean("myBean", MyBean.class);
```

- **Lazy** initialization by default
- Minimal features
- Rarely used directly

#### 6.2 ApplicationContext (Advanced — USE THIS)

```java
ApplicationContext ctx = 
    new AnnotationConfigApplicationContext(AppConfig.class);
MyBean bean = ctx.getBean(MyBean.class);
```

**Adds:**
- Eager singleton instantiation
- Internationalization (MessageSource)
- Event publication (ApplicationEventPublisher)
- Resource loading (ResourceLoader)
- AOP integration
- Web application contexts
- Bean post-processing

### ApplicationContext Implementations

| Implementation | Use Case |
|----------------|----------|
| `AnnotationConfigApplicationContext` | Standalone + Java config |
| `ClassPathXmlApplicationContext` | XML config on classpath |
| `FileSystemXmlApplicationContext` | XML config on filesystem |
| `AnnotationConfigWebApplicationContext` | Web + Java config |
| `XmlWebApplicationContext` | Web + XML |
| `GenericApplicationContext` | Programmatic registration |

### Comparison Table

| Feature | BeanFactory | ApplicationContext |
|---------|-------------|--------------------|
| Bean instantiation | Lazy | Eager (singletons) |
| AOP | Manual | Automatic |
| Events | ❌ | ✅ |
| i18n | ❌ | ✅ |
| Resource loading | ❌ | ✅ |
| Annotation support | Limited | Full |
| Memory usage | Lower | Higher |
| Recommended | ❌ | ✅ |

### Hierarchy

```
                BeanFactory
                     ▲
                     │
          ApplicationContext
                     ▲
        ┌────────────┼────────────┐
        │            │            │
WebApplication   Configurable   ...
   Context     ApplicationContext
```

### Obtaining the Context (Multiple Ways)

```java
// 1. Standalone Java
ApplicationContext ctx = 
    new AnnotationConfigApplicationContext(AppConfig.class);

// 2. Spring Boot (injected)
@Autowired
private ApplicationContext context;

// 3. Implement ApplicationContextAware
@Component
public class MyBean implements ApplicationContextAware {
    private ApplicationContext ctx;
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }
}

// 4. In main
public static void main(String[] args) {
    try (ConfigurableApplicationContext ctx = 
            SpringApplication.run(MyApp.class, args)) {
        // ctx is available
    }
}
```

---

## 7. Bean Definition & Naming

### What is a Bean?

A **bean** is an object that is **instantiated, assembled, and managed by the Spring IoC container**.

### Bean Definition Metadata

Every bean definition contains:

| Element | Description |
|---------|-------------|
| **Class** | Fully qualified class name |
| **Name** | Unique identifier (auto-generated if not specified) |
| **Scope** | singleton, prototype, request, session, … |
| **Constructor args** | For dependency injection |
| **Properties** | Setter injection |
| **Autowiring mode** | byType, byName, constructor, no |
| **Lazy init** | Whether to instantiate on first request |
| **Init method** | Custom initialization |
| **Destroy method** | Custom destruction |

### Bean Naming Rules

**XML:**

```xml
<bean id="userService" class="com.example.UserService"/>
<!-- or -->
<bean name="userService,userSvc" class="com.example.UserService"/>
```

**Annotation-based (default name = decapitalized class name):**

```java
@Component
public class UserService { }              // bean name: "userService"

@Component("customName")
public class OrderService { }             // bean name: "customName"
```

**Java config:**

```java
@Configuration
public class AppConfig {
    @Bean                          // bean name = method name
    public UserService userService() { ... }
    
    @Bean("myService")             // explicit name
    public OrderService orderService() { ... }
    
    @Bean(name = {"a", "b"})       // multiple aliases
    public PaymentService payment() { ... }
}
```

### Bean Aliases

```java
// Java
@Bean(name = {"dataSource", "ds", "primaryDs"})

// XML
<alias name="dataSource" alias="ds"/>
```

### Retrieving Beans

```java
// By name (requires cast in old versions)
UserService svc = (UserService) ctx.getBean("userService");

// By type (preferred)
UserService svc = ctx.getBean(UserService.class);

// By name + type
UserService svc = ctx.getBean("userService", UserService.class);

// All beans of a type
Map<String, UserService> all = ctx.getBeansOfType(UserService.class);

// Bean names
String[] names = ctx.getBeanDefinitionNames();

// Check existence
boolean exists = ctx.containsBean("userService");
```

---

## 8. Java-Based Configuration

Introduced in **Spring 3.0**, Java config is now the preferred approach.

### @Configuration + @Bean

```java
@Configuration
public class AppConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        ds.setUsername("admin");
        ds.setPassword("secret");
        ds.setMaximumPoolSize(10);
        return ds;
    }
    
    @Bean
    public UserRepository userRepository(DataSource dataSource) {
        return new JdbcUserRepository(dataSource);
    }
    
    @Bean
    public UserService userService(UserRepository repo) {
        return new UserService(repo);
    }
}
```

### How @Configuration Works — CGLIB Enhancement

`@Configuration` classes are **subclassed via CGLIB** at startup so that **calls between `@Bean` methods return the same singleton**.

```java
@Configuration
public class AppConfig {
    @Bean
    public A a() { return new A(b()); }
    
    @Bean
    public B b() { return new B(); }
}
```

When `a()` calls `b()`, the CGLIB proxy intercepts and returns the cached singleton `B`, **not a new instance**.

**ProxyBeanMethods attribute:**

```java
@Configuration(proxyBeanMethods = false)  // no CGLIB, faster
public class AppConfig {
    // Each @Bean method call creates new instances
    // Use when no inter-bean method calls
}
```

| Setting | Behavior | When to use |
|---------|----------|-------------|
| `proxyBeanMethods = true` (default) | CGLIB subclassing, singleton method calls | Inter-bean dependencies |
| `proxyBeanMethods = false` | No CGLIB, plain factory | No method-to-method calls, faster startup |

### @Configuration vs @Component

| Aspect | @Configuration | @Component |
|--------|---------------|-----------|
| CGLIB enhanced | ✅ (default) | ❌ |
| Inter-bean method calls | Singleton | New instance each time |
| Used for | `@Bean` factory methods | Regular stereotypes |

```java
@Component
public class NotConfig {
    @Bean
    public A a() { return new A(b()); }
    
    @Bean
    public B b() { return new B(); }
}
// a() and any call to b() create NEW instances every time
```

### Importing Other Configs

```java
@Configuration
@Import({DatabaseConfig.class, SecurityConfig.class})
public class AppConfig { }
```

### @ComponentScan in Config

```java
@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig { }
```

---

## 9. Annotation-Based Configuration

### Stereotype Annotations

| Annotation | Layer | Notes |
|------------|-------|-------|
| `@Component` | Generic | Base stereotype |
| `@Service` | Business logic | Semantically meaningful |
| `@Repository` | Data access | Adds exception translation |
| `@Controller` | Web MVC | Handles web requests |
| `@RestController` | REST API | `@Controller` + `@ResponseBody` |

> **Note:** All are functionally equivalent (all are `@Component`). They differ **semantically** for readability and tools. `@Repository` additionally enables `PersistenceExceptionTranslationPostProcessor`.

### Example

```java
@Repository
public class JpaUserRepository implements UserRepository {
    @PersistenceContext
    private EntityManager em;
    
    @Override
    public User findById(Long id) {
        return em.find(User.class, id);
    }
}

@Service
public class UserService {
    private final UserRepository repo;
    
    public UserService(UserRepository repo) { this.repo = repo; }
    
    @Transactional
    public User get(Long id) { return repo.findById(id); }
}

@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService service;
    
    public UserController(UserService service) { this.service = service; }
    
    @GetMapping("/{id}")
    public User get(@PathVariable Long id) { return service.get(id); }
}
```

### @Autowired in Detail

Already covered in §5. Additional points:

```java
// Custom qualifier annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Fast { }

@Component
@Fast
public class FastProcessor implements Processor { }

@Component
public class SlowProcessor implements Processor { }

@Service
public class Service {
    public Service(@Fast Processor p) { }   // injects FastProcessor
}
```

---

## 10. Component Scanning

### How It Works

`@ComponentScan` tells Spring where to look for stereotype-annotated classes. When found, Spring registers them as bean definitions.

```java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    basePackageClasses = UserService.class,
    includeFilters = @Filter(type = FilterType.REGEX, 
                             pattern = ".*ServiceImpl"),
    excludeFilters = @Filter(type = FilterType.ANNOTATION, 
                             classes = Deprecated.class)
)
public class AppConfig { }
```

### @SpringBootApplication = Combined

```java
@SpringBootApplication
// equivalent to:
@Configuration
@EnableAutoConfiguration
@ComponentScan
```

The default `@ComponentScan` scans the package of the main class **and its subpackages**.

```
com.example.app
├── MyApp.java                 ← @SpringBootApplication here
├── service/                   ← scanned
├── repo/                      ← scanned
└── other/
    └── notscanned/            ← under com.example.app → scanned
```

```
com.other.package              ← NOT scanned (sibling)
```

### Filter Types

| FilterType | Description |
|------------|-------------|
| `ANNOTATION` | By annotation presence |
| `ASSIGNABLE_TYPE` | By class/interface |
| `ASPECTJ` | AspectJ expression |
| `REGEX` | Regex on class name |
| `CUSTOM` | Custom `TypeFilter` |

### Custom TypeFilter

```java
public class MyFilter implements TypeFilter {
    @Override
    public boolean match(MetadataReader reader, 
                          MetadataReaderFactory factory) {
        String cls = reader.getClassMetadata().getClassName();
        return cls.startsWith("My");
    }
}

@ComponentScan(
    includeFilters = @Filter(type = FilterType.CUSTOM, 
                             classes = MyFilter.class)
)
```

### When @ComponentScan is Needed

- **Explicitly** in standalone apps (`@Configuration`)
- **Implicitly** via `@SpringBootApplication`

---

## 11. Bean Scopes

### Overview

| Scope | Description | Injects |
|-------|-------------|---------|
| **singleton** (default) | One instance per container | Same instance |
| **prototype** | New instance per request | New each `getBean()` / injection |
| **request** | One per HTTP request | Web only |
| **session** | One per HTTP session | Web only |
| **application** | One per `ServletContext` | Web only |
| **websocket** | One per WebSocket session | Web only |

### Singleton (Default)

```java
@Component
@Scope("singleton")   // or @Scope(ConfigurableBeanFactory.SCOPE_SINGLETON)
public class CacheManager { }
```

**Key points:**
- ONE instance per Spring container (not per JVM)
- Eagerly initialized at startup (unless `@Lazy`)
- Cached in a `singletonObjects` map
- **Stateless recommended**

### Prototype

```java
@Component
@Scope("prototype")   // or ConfigurableBeanFactory.SCOPE_PROTOTYPE
public class ShoppingCart { }
```

**Key points:**
- New instance on every request
- **Spring does NOT manage destruction** — you must clean up
- Injected into a singleton → the singleton gets **one** prototype instance, not fresh ones

**Problem with injecting prototype into singleton:**

```java
@Service  // singleton
public class OrderService {
    private final ShoppingCart cart;  // injected ONCE
    public OrderService(ShoppingCart cart) { this.cart = cart; }
}
```

**Solutions:**

1. **`@Lookup` method:**
```java
@Service
public abstract class OrderService {
    @Lookup
    protected abstract ShoppingCart createCart();
    
    public void place() {
        ShoppingCart cart = createCart();  // new each call
    }
}
```

2. **`ObjectFactory` / `ObjectProvider`:**
```java
@Service
public class OrderService {
    private final ObjectProvider<ShoppingCart> cartProvider;
    public OrderService(ObjectProvider<ShoppingCart> p) { this.cartProvider = p; }
    
    public void place() {
        ShoppingCart cart = cartProvider.getObject();  // new each call
    }
}
```

3. **`ScopedProxyMode` (web scopes only):**
```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestScopedBean { }
```

### Web Scopes

```java
@Component
@RequestScope  // = @Scope("request", proxyMode = TARGET_CLASS)
public class RequestContext { }

@Component
@SessionScope  // = @Scope("session", proxyMode = TARGET_CLASS)
public class UserSession { }

@Component
@ApplicationScope
public class AppState { }
```

**Proxy mode is required** because web-scoped beans are injected into singletons, and the proxy looks up the actual instance per request.

### Custom Scope

```java
public class ThreadScope implements Scope {
    private final ThreadLocal<Map<String, Object>> beans = 
        ThreadLocal.withInitial(HashMap::new);
    
    @Override
    public Object get(String name, ObjectFactory<?> factory) {
        return beans.get().computeIfAbsent(name, k -> factory.getObject());
    }
    // ... other methods
}

// Register
ConfigurableBeanFactory factory = ctx.getBeanFactory();
factory.registerScope("thread", new ThreadScope());
```

### Scope Comparison Table

| Scope | Instance per | Lifecycle managed by Spring |
|-------|-------------|----------------------------|
| singleton | Container | ✅ Fully |
| prototype | Request | ⚠️ Init only |
| request | HTTP request | ✅ |
| session | HTTP session | ✅ |
| application | ServletContext | ✅ |
| websocket | WebSocket | ✅ |
| custom | As defined | Varies |

---

## 12. Bean Lifecycle

### Complete Lifecycle Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                     BEAN LIFECYCLE                              │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Load bean definitions (from XML/annotations/Java config)    │
│                          │                                      │
│                          ▼                                      │
│  2. BeanFactoryPostProcessor.postProcessBeanFactory()           │
│     (modify bean definitions BEFORE any bean is created)        │
│                          │                                      │
│                          ▼                                      │
│  3. Instantiate bean (constructor called)                       │
│                          │                                      │
│                          ▼                                      │
│  4. Populate properties (setter/field injection)                │
│                          │                                      │
│                          ▼                                      │
│  5. BeanNameAware.setBeanName()                                 │
│                          │                                      │
│                          ▼                                      │
│  6. BeanFactoryAware.setBeanFactory()                           │
│                          │                                      │
│                          ▼                                      │
│  7. ApplicationContextAware.setApplicationContext()             │
│                          │                                      │
│                          ▼                                      │
│  8. BeanPostProcessor.postProcessBeforeInitialization()         │
│                          │                                      │
│                          ▼                                      │
│  9. @PostConstruct method                                       │
│                          │                                      │
│                          ▼                                      │
│ 10. InitializingBean.afterPropertiesSet()                       │
│                          │                                      │
│                          ▼                                      │
│ 11. Custom init-method / @Bean(initMethod = "...")              │
│                          │                                      │
│                          ▼                                      │
│ 12. BeanPostProcessor.postProcessAfterInitialization()          │
│     (AOP proxies created here!)                                 │
│                          │                                      │
│                          ▼                                      │
│          ★ BEAN IS READY FOR USE ★                              │
│                          │                                      │
│                          ▼                                      │
│ 13. Container shutdown                                          │
│                          │                                      │
│                          ▼                                      │
│ 14. @PreDestroy method                                          │
│                          │                                      │
│                          ▼                                      │
│ 15. DisposableBean.destroy()                                    │
│                          │                                      │
│                          ▼                                      │
│ 16. Custom destroy-method                                       │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Code Example

```java
@Component
public class LifecycleBean implements BeanNameAware, 
                                        BeanFactoryAware, 
                                        ApplicationContextAware,
                                        InitializingBean, 
                                        DisposableBean {
    
    public LifecycleBean() {
        System.out.println("1. Constructor");
    }
    
    @Autowired
    public void setDependency(SomeDep dep) {
        System.out.println("2. Setter injection");
    }
    
    @Override
    public void setBeanName(String name) {
        System.out.println("3. setBeanName: " + name);
    }
    
    @Override
    public void setBeanFactory(BeanFactory bf) {
        System.out.println("4. setBeanFactory");
    }
    
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("5. setApplicationContext");
    }
    
    @PostConstruct
    public void postConstruct() {
        System.out.println("6. @PostConstruct");
    }
    
    @Override
    public void afterPropertiesSet() {
        System.out.println("7. afterPropertiesSet");
    }
    
    public void customInit() {
        System.out.println("8. customInit");
    }
    
    @PreDestroy
    public void preDestroy() {
        System.out.println("9. @PreDestroy");
    }
    
    @Override
    public void destroy() {
        System.out.println("10. destroy");
    }
    
    public void customDestroy() {
        System.out.println("11. customDestroy");
    }
}
```

**Register with custom init/destroy:**

```java
@Bean(initMethod = "customInit", destroyMethod = "customDestroy")
public LifecycleBean lifecycleBean() { return new LifecycleBean(); }
```

### Order of Initialization Callbacks

Multiple mechanisms may coexist. The order is:

1. `@PostConstruct` (JSR-250) ← **highest priority**
2. `InitializingBean.afterPropertiesSet()`
3. Custom `init-method`

### Order of Destruction Callbacks

1. `@PreDestroy` (JSR-250)
2. `DisposableBean.destroy()`
3. Custom `destroy-method`

### @PostConstruct / @PreDestroy Notes

- Standard: `jakarta.annotation.PostConstruct` (Spring 6+) / `javax.annotation.PostConstruct` (older)
- Called **once** per bean instance
- `@PostConstruct` = after DI complete
- `@PreDestroy` = before container removes bean
- Not invoked for **prototype** beans on destruction

### Prototype Bean Lifecycle

- Init callbacks ARE invoked
- **Destroy callbacks are NOT** invoked by Spring
- You must handle cleanup (e.g., in try-with-resources, or `DisposableBean` manually)

---

## 13. BeanPostProcessor & BeanFactoryPostProcessor

### BeanFactoryPostProcessor

Runs **before** bean instantiation. Can modify bean **definitions**.

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) {
        BeanDefinition def = bf.getBeanDefinition("myService");
        def.setScope("prototype");
        def.getPropertyValues().add("timeout", 5000);
    }
}
```

**Built-in examples:**
- `PropertySourcesPlaceholderConfigurer` — resolves `${...}`
- `ConfigurationClassPostProcessor` — processes `@Configuration`
- `CustomEditorConfigurer`

### BeanPostProcessor

Runs **per bean**, before & after initialization. Can wrap/replace beans.

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String name) {
        System.out.println("Before init: " + name);
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String name) {
        System.out.println("After init: " + name);
        return bean;  // could return a proxy
    }
}
```

**Built-in examples:**
- `AutowiredAnnotationBeanPostProcessor` — handles `@Autowired`
- `CommonAnnotationBeanPostProcessor` — handles `@PostConstruct`, `@PreDestroy`
- `AnnotationAwareAspectJAutoProxyCreator` — creates AOP proxies

### Comparison

| Feature | BeanFactoryPostProcessor | BeanPostProcessor |
|---------|--------------------------|-------------------|
| Runs on | Bean definitions | Bean instances |
| When | Before bean creation | During bean creation |
| Can modify | Metadata | Actual objects |
| Frequency | Once | Per bean |
| Common use | Placeholder resolution | AOP, annotations |

### Ordering

Implement `Ordered` or use `@Order`:

```java
@Component
@Order(1)
public class FirstBPP implements BeanPostProcessor { }

@Component
@Order(2)
public class SecondBPP implements BeanPostProcessor { }
```

---

## 14. Spring Expression Language (SpEL)

### What is SpEL?

A powerful expression language for **querying and manipulating objects at runtime**. Used in `@Value`, `@Conditional`, XML config, Spring Security, Spring Data.

### Syntax Basics

```
#{ expression }
```

### Literals

```java
@Value("#{5}")           int five;
@Value("#{'hello'}")     String hello;
@Value("#{true}")        boolean t;
@Value("#{3.14}")        double pi;
@Value("#{null}")        Object n;
```

### Operators

| Category | Operators |
|----------|-----------|
| Arithmetic | `+ - * / % ^` |
| Comparison | `== != < > <= >=` (also `eq ne lt gt le ge`) |
| Logical | `&& \|\| !` (also `and or not`) |
| String | `+` (concatenation) |
| Regex | `matches` |
| Ternary | `? :` |
| Elvis | `?:` |
| Safe navigation | `?.` |
| Type | `T()` |
| Selection | `.?[]` `.![]` |
| Projection | `.![]` |

### Examples

```java
// Math
@Value("#{2 + 3 * 4}")            int result;      // 14

// String
@Value("#{'Hello ' + 'World'}")   String greet;    // "Hello World"

// Regex
@Value("#{'12345' matches '\\d+'}") boolean isNum; // true

// Ternary
@Value("#{user.age > 18 ? 'Adult' : 'Minor'}")    String cat;

// Elvis (default if null)
@Value("#{systemProperties['user.name'] ?: 'unknown'}") String name;

// Safe navigation (null-safe)
@Value("#{user?.address?.city}")  String city;

// Static method call
@Value("#{T(java.lang.Math).PI}") double pi;

// Bean reference
@Value("#{userService.count()}")  long count;

// List/Map
@Value("#{myList[0]}")             String first;
@Value("#{myMap['key']}")          Object val;
```

### SpEL in @Bean

```java
@Bean
public Cache cache(@Value("#{systemProperties['cache.size'] ?: 100}") 
                    int size) {
    return new Cache(size);
}
```

### SpEL in Spring Security

```java
@PreAuthorize("#order.owner == authentication.name")
public void updateOrder(Order order) { ... }

@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public User getUser(Long userId) { ... }
```

### SpEL in @Conditional

```java
@ConditionalOnExpression("#{environment['feature.x'] == 'true'}")
@Bean
public X x() { return new X(); }
```

### Programmatic SpEL

```java
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'Hello'.concat(' World')");
String msg = exp.getValue(String.class);   // "Hello World"

// With root object
StandardEvaluationContext ctx = new StandardEvaluationContext(user);
Expression nameExp = parser.parseExpression("name.toUpperCase()");
String upper = nameExp.getValue(ctx, String.class);
```

### SpEL Security Warning

⚠️ **Never evaluate untrusted user input as SpEL.** `T(java.lang.Runtime).getRuntime().exec(...)` = RCE. Use `SimpleEvaluationContext` to restrict.

---

## 15. Spring Profiles

### Purpose

**Profiles** allow registering different beans in different environments (dev, test, prod).

### Declaring Profile-Specific Beans

```java
@Component
@Profile("dev")
public class DevDataSource implements DataSource { }

@Component
@Profile("prod")
public class ProdDataSource implements DataSource { }

@Component
@Profile({"dev", "test"})           // OR
public class MockEmailService { }

@Component
@Profile("!prod")                   // NOT prod
public class VerboseLogger { }
```

### Java Config

```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource ds() { return new H2DataSource(); }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource ds() { return new PostgresDataSource(); }
}
```

### Profile Expressions (Spring 5.1+)

```java
@Profile("dev & !cloud")     // AND + NOT
@Profile("dev | staging")    // OR
@Profile("(dev | test) & !ci")
```

### Activating Profiles

**1. application.properties:**
```properties
spring.profiles.active=dev
```

**2. Command line:**
```bash
java -jar app.jar --spring.profiles.active=dev,metrics
```

**3. Environment variable:**
```bash
export SPRING_PROFILES_ACTIVE=dev
```

**4. `@ActiveProfiles` in tests:**
```java
@SpringBootTest
@ActiveProfiles("test")
class MyTest { }
```

**5. Programmatically:**
```java
SpringApplication app = new SpringApplication(MyApp.class);
app.setAdditionalProfiles("dev");
app.run(args);
```

### Profile-Specific Property Files

```
application.properties           ← always loaded
application-dev.properties       ← loaded when "dev" active
application-prod.properties      ← loaded when "prod" active
application-dev,metrics.properties ← multiple profiles
```

**Precedence:** More specific profile properties **override** base.

### Default Profile

If no profile is active, Spring uses `"default"`. Beans annotated `@Profile("default")` are loaded.

### Profile Groups (Spring Boot 2.4+)

```properties
spring.profiles.group.production=prod-db,prod-mq,prod-monitoring
```

Activating `production` activates all three.

### Multiple Profiles

```properties
spring.profiles.active=dev,metrics,debug
```

Beans matching **any** active profile are registered.

### Interview Tip

> If both `dev` and `prod` profiles activate the same bean (e.g., two `DataSource` beans), you'll get a `NoUniqueBeanDefinitionException` — use `@Primary` or `@Profile("!prod")` on the dev one.

---

## 16. External Properties & Environment

### Property Sources

Spring's `Environment` aggregates property sources in a specific order (later overrides earlier in Spring Boot):

1. Default properties (`SpringApplication.setDefaultProperties`)
2. `@PropertySource` on `@Configuration`
3. Config data (application.properties/yml)
4. OS environment variables
5. Java system properties (`-D`)
6. JNDI
7. `ServletConfig` / `ServletContext` init params
8. `SPRING_APPLICATION_JSON`
9. Command-line arguments
10. Test `@TestPropertySource`

### @PropertySource

```java
@Configuration
@PropertySource("classpath:custom.properties")
@PropertySource("file:/etc/myapp/config.properties")
public class AppConfig {
    @Value("${app.name}")
    private String appName;
}
```

**Multiple files:**

```java
@PropertySources({
    @PropertySource("classpath:db.properties"),
    @PropertySource("classpath:cache.properties")
})
```

### @Value

```java
@Component
public class MyBean {
    @Value("${app.name}")                     String name;
    @Value("${app.timeout:5000}")             int timeout;    // default
    @Value("${app.hosts}")                    List<String> hosts;  // comma-split
    @Value("${app.enabled:true}")             boolean enabled;
    @Value("#{systemProperties['user.dir']}") String dir;      // SpEL
}
```

### Environment API

```java
@Autowired
private Environment env;

public void doSomething() {
    String name = env.getProperty("app.name");
    String name2 = env.getProperty("app.name", "defaultName");
    int port = env.getProperty("server.port", Integer.class, 8080);
    
    String[] activeProfiles = env.getActiveProfiles();
    boolean isProd = env.acceptsProfiles(Profiles.of("prod"));
}
```

### Relaxed Binding (Spring Boot)

Property names can be expressed in multiple forms:

| Form | Example |
|------|---------|
| Kebab-case (recommended) | `my-app.max-size` |
| camelCase | `myApp.maxSize` |
| Underscore | `my_app.max_size` |
| Uppercase + underscore (env vars) | `MY_APP_MAX_SIZE` |

### @ConfigurationProperties (Best Practice)

```java
@ConfigurationProperties(prefix = "myapp")
@Component
public class MyAppProperties {
    private String name;
    private int timeout = 5000;
    private List<String> hosts = new ArrayList<>();
    private Database database = new Database();
    
    // getters/setters
    
    public static class Database {
        private String url;
        private String username;
        // getters/setters
    }
}
```

**application.yml:**

```yaml
myapp:
  name: MyApp
  timeout: 3000
  hosts:
    - host1
    - host2
  database:
    url: jdbc:...
    username: admin
```

**Benefits:**
- Type-safe
- IDE auto-completion with `spring-boot-configuration-processor`
- Validation via `@Validated`
- Grouped config

### Validation

```java
@ConfigurationProperties(prefix = "myapp")
@Validated
public class MyAppProperties {
    @NotBlank
    private String name;
    
    @Min(1) @Max(65535)
    private int port;
}
```

---

## 17. Circular Dependencies

### What is a Circular Dependency?

```
Bean A → depends on → Bean B → depends on → Bean A
```

### Example

```java
@Service
public class A {
    public A(B b) { }       // A needs B
}

@Service
public class B {
    public B(A a) { }       // B needs A → CYCLE
}
```

**Result:** `BeanCurrentlyInCreationException` (for constructor injection).

### Why Constructor Injection Fails

A can't be constructed until B exists; B can't be constructed until A exists → deadlock at startup.

### How Spring Handles It (Setter/Field Injection)

Spring uses a **three-level cache** in `DefaultSingletonBeanRegistry`:

| Level | Name | Purpose |
|-------|------|---------|
| 1 | `singletonObjects` | Fully initialized beans |
| 2 | `earlySingletonObjects` | Early references (before init) |
| 3 | `singletonFactories` | Factories to create early references |

**Flow for setter injection:**

1. Create A → put factory in level-3 cache
2. A needs B → create B
3. B needs A → find A's early reference in level-3 → promote to level-2
4. B completes → put in level-1
5. A completes → put in level-1

**Constructor injection can't do this** because the object must exist before being cached.

### Solutions

**1. Use setter injection (if you must):**

```java
@Service
public class A {
    private B b;
    @Autowired public void setB(B b) { this.b = b; }
}
```

**2. Use @Lazy:**

```java
@Service
public class A {
    private final B b;
    public A(@Lazy B b) { this.b = b; }   // proxy injected
}
```

**3. Redesign (BEST):**

Extract shared logic into a third class C; A and B both depend on C.

```java
@Service
public class C { /* shared */ }

@Service
public class A { public A(C c) { } }

@Service
public class B { public B(C c) { } }
```

**4. @PostConstruct to break initialization cycle:**

```java
@Service
public class A {
    @Autowired private B b;
    public A() { }
}

@Service
public class B {
    @Autowired private A a;
    public B() { }
}
```

**5. ApplicationContextAware (last resort):**

```java
@Service
public class A implements ApplicationContextAware {
    private ApplicationContext ctx;
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }
    private B getB() { return ctx.getBean(B.class); }
}
```

### Spring Boot 2.6+ Default

Since **Spring Boot 2.6**, circular references are **disabled by default**. Re-enable (not recommended):

```properties
spring.main.allow-circular-references=true
```

**Best practice:** Treat circular dependencies as a **design smell**. Redesign.

---

## 18. @Lazy, @Primary, @Qualifier

### @Lazy

**Delays bean creation** until first use.

```java
@Component
@Lazy
public class ExpensiveBean { }

// Or per injection point
@Service
public class Service {
    public Service(@Lazy ExpensiveBean bean) { }
}

// Or on @Bean method
@Bean @Lazy
public ExpensiveBean expensive() { ... }
```

**Use cases:**
- Heavy startup beans not always needed
- Break circular dependencies
- Optional features

**Caveat:** Errors in lazy beans appear **at runtime**, not startup.

### @Primary

Marks a bean as the **default** when multiple candidates exist.

```java
@Component
@Primary
public class PostgresDataSource implements DataSource { }

@Component
public class MySQLDataSource implements DataSource { }

@Service
public class Service {
    public Service(DataSource ds) { }   // gets PostgresDataSource
}
```

### @Qualifier

Explicitly select a specific bean.

```java
@Service
public class Service {
    private final DataSource ds;
    
    public Service(@Qualifier("mySQLDataSource") DataSource ds) {
        this.ds = ds;
    }
}
```

### Combining @Primary + @Qualifier

- `@Qualifier` on injection point **overrides** `@Primary`.

### Custom Qualifier Annotations

```java
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Postgres { }

@Component
@Postgres
public class PostgresDataSource implements DataSource { }

@Service
public class Service {
    public Service(@Postgres DataSource ds) { }
}
```

### Priority Order at Injection

1. `@Qualifier` (specific)
2. Custom qualifier
3. `@Primary`
4. Field/parameter name matching
5. `NoUniqueBeanDefinitionException`

### Comparison

| Annotation | Purpose | Placement |
|------------|---------|-----------|
| `@Primary` | Default choice | Bean declaration |
| `@Qualifier` | Specific choice | Injection point |
| `@Lazy` | Defer creation | Either |

---

## 19. Interview Questions

### Q1. What is IoC and how does Spring implement it?

**Answer:** IoC (Inversion of Control) is a design principle where object creation and lifecycle control is transferred from application code to a container. Spring implements IoC via **Dependency Injection** — the `ApplicationContext` container reads bean definitions, instantiates beans, injects their dependencies, and manages their lifecycle. Types: constructor, setter, field injection.

### Q2. BeanFactory vs ApplicationContext?

**Answer:** `BeanFactory` is the basic container with lazy initialization and minimal features. `ApplicationContext` extends it with eager singleton instantiation, event publishing, i18n, resource loading, AOP integration, and annotation support. Always prefer `ApplicationContext` in modern Spring.

### Q3. Difference between @Component, @Service, @Repository, @Controller?

**Answer:** All are specializations of `@Component` and are functionally equivalent for component scanning. Semantically: `@Service` for business layer, `@Repository` for data layer (adds exception translation), `@Controller`/`@RestController` for web layer. Using them communicates intent and enables tooling.

### Q4. Constructor vs setter vs field injection — which to use?

**Answer:** **Constructor injection** is preferred: enforces immutability (final fields), guarantees dependencies at construction, fails fast, easy to test without Spring, and no Spring imports. Setter injection is for optional dependencies. Field injection is discouraged (harder to test, no immutability, hides dependencies).

### Q5. How does Spring resolve circular dependencies?

**Answer:** For setter/field injection, Spring uses a **three-level cache** (`singletonObjects`, `earlySingletonObjects`, `singletonFactories`). It exposes a partially-initialized bean early so the other bean can reference it. Constructor injection cannot be resolved this way (fails with `BeanCurrentlyInCreationException`). From Spring Boot 2.6+, circular references are disabled by default.

### Q6. What are bean scopes?

**Answer:** Singleton (default; one per container), prototype (new per request), request (per HTTP request), session (per HTTP session), application (per ServletContext), websocket (per WebSocket session), plus custom scopes. Web scopes need proxy mode when injected into singletons.

### Q7. Explain bean lifecycle.

**Answer:** Instantiation → populate properties → `BeanNameAware.setBeanName` → `BeanFactoryAware` → `ApplicationContextAware` → `BeanPostProcessor.postProcessBeforeInitialization` → `@PostConstruct` → `InitializingBean.afterPropertiesSet` → custom `init-method` → `BeanPostProcessor.postProcessAfterInitialization` (AOP proxy here) → bean in use → `@PreDestroy` → `DisposableBean.destroy` → custom `destroy-method`.

### Q8. What is @PostConstruct vs afterPropertiesSet vs init-method?

**Answer:** All are initialization callbacks. Order: `@PostConstruct` (JSR-250, preferred) → `InitializingBean.afterPropertiesSet()` → custom init-method. `@PostConstruct` is standard and doesn't couple to Spring.

### Q9. What is @Configuration and how does it work?

**Answer:** `@Configuration` marks a class as a bean definition source. Spring **subclasses** it with CGLIB so that calls between `@Bean` methods return the same singleton. `proxyBeanMethods = false` disables this for performance when no inter-bean calls occur. `@Component` with `@Bean` methods doesn't get this behavior.

### Q10. Difference between @ComponentScan and @SpringBootApplication?

**Answer:** `@SpringBootApplication` is a meta-annotation combining `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`. It scans the package of the annotated class and its subpackages.

### Q11. What is the role of BeanPostProcessor?

**Answer:** It intercepts bean creation, allowing modification of the bean **before** and **after** initialization. Used for annotation processing (`@Autowired`, `@PostConstruct`), AOP proxy creation, and custom logic. `BeanFactoryPostProcessor`, by contrast, modifies bean **definitions** before any bean is created.

### Q12. How does @Autowired resolve ambiguity?

**Answer:** 1) by type, 2) `@Primary` wins, 3) `@Qualifier` on injection point overrides, 4) custom qualifier annotations, 5) bean name matching field/param name. Otherwise `NoUniqueBeanDefinitionException`.

### Q13. What is SpEL?

**Answer:** Spring Expression Language — a runtime expression language (`#{...}`) supporting arithmetic, comparisons, method calls, bean references, static type access (`T()`), safe navigation, collection selection/projection. Used in `@Value`, `@PreAuthorize`, `@ConditionalOnExpression`. ⚠️ Never evaluate untrusted input.

### Q14. How do Spring Profiles work?

**Answer:** Profiles (`@Profile("dev")`) register beans only when the profile is active. Activated via `spring.profiles.active`, env var, or command line. Support expressions (`!prod`, `dev & cloud`). Profile-specific property files (`application-dev.properties`) are also loaded.

### Q15. What is the difference between @Value and @ConfigurationProperties?

**Answer:** `@Value("${key}")` injects a single property, supports SpEL, but lacks type-safety and grouping. `@ConfigurationProperties(prefix=...)` binds a group of properties to a POJO with type safety, validation (`@Validated`), relaxed binding, and IDE completion. Prefer `@ConfigurationProperties` for grouped config.

### Q16. What happens if a prototype bean is injected into a singleton?

**Answer:** The singleton receives **one** prototype instance at creation and reuses it forever. To get fresh prototypes, use `@Lookup`, `ObjectProvider<T>`, `ObjectFactory<T>`, or `ApplicationContext.getBean()`.

### Q17. What is the difference between @Lazy and @Scope("prototype")?

**Answer:** `@Lazy` defers creation until first use but the bean is still a **singleton** (one instance). `@Scope("prototype")` creates a **new instance** on every request.

### Q18. How to enable circular references in Spring Boot 2.6+?

**Answer:** `spring.main.allow-circular-references=true`. However, this is discouraged — refactor instead.

### Q19. What is a BeanDefinition?

**Answer:** A `BeanDefinition` is Spring's metadata object describing a bean: class name, scope, constructor args, properties, autowiring mode, lazy flag, init/destroy methods, factory method/bean, etc. `BeanFactoryPostProcessor` can modify `BeanDefinition`s before instantiation.

### Q20. What's the difference between an ApplicationContext and a WebApplicationContext?

**Answer:** `WebApplicationContext` extends `ApplicationContext` with web-specific capabilities: access to `ServletContext`, request/session scopes, `ThemeSource`, and integration with `DispatcherServlet`. Used in Spring MVC.

### Q21. Explain @Required, @Autowired(required=false), Optional<T>, @Nullable.

**Answer:**
- `@Required` (deprecated) — setter must be configured in XML
- `@Autowired(required=false)` — inject null if no bean; risky
- `Optional<T>` — explicit optional dependency, constructor injection works
- `@Nullable` — allows null parameter/field

**Prefer `Optional<T>` or `@Nullable` over `required=false`.**

### Q22. What is a FactoryBean?

**Answer:** A `FactoryBean<T>` is a Spring bean whose `getObject()` produces another bean. Used for complex bean creation (e.g., `SqlSessionFactoryBean`, `LocalContainerEntityManagerFactoryBean`). Retrieved by prefixing `&` to get the factory itself.

```java
public class MyFactoryBean implements FactoryBean<MyBean> {
    public MyBean getObject() { return new MyBean(); }
    public Class<?> getObjectType() { return MyBean.class; }
    public boolean isSingleton() { return true; }
}
```

### Q23. What is the difference between @Bean and @Component?

**Answer:** `@Component` is on the class — Spring discovers it via scanning and creates an instance. `@Bean` is on a `@Configuration` method — you write the instantiation code, useful for third-party classes you can't annotate.

### Q24. How does Spring manage the bean lifecycle for prototype-scoped beans?

**Answer:** Spring instantiates, injects dependencies, runs `BeanPostProcessor`s and init callbacks. But **it does not manage destruction** — no `@PreDestroy`, `DisposableBean`, or destroy-method is invoked. You must clean up manually.

### Q25. What is ApplicationContextAware? When to use?

**Answer:** Interface with `setApplicationContext(ApplicationContext)`. Spring injects the context automatically. Useful for dynamic bean lookup, but **use sparingly** — it couples your class to Spring. Prefer constructor injection.

---

## 20. Summary Cheat Sheet

### Annotations Quick Reference

| Annotation | Purpose | Layer |
|------------|---------|-------|
| `@Configuration` | Java config class (CGLIB-enhanced) | Config |
| `@Bean` | Factory method producing a bean | Config |
| `@Component` | Generic stereotype | Any |
| `@Service` | Business logic | Service |
| `@Repository` | Data access (exception translation) | Persistence |
| `@Controller` / `@RestController` | Web | Web |
| `@ComponentScan` | Discover components | Config |
| `@Autowired` | Inject dependency | Field/ctor/setter |
| `@Qualifier` | Specific bean selection | Injection point |
| `@Primary` | Default bean among candidates | Bean decl |
| `@Lazy` | Defer instantiation | Either |
| `@Scope` | Bean scope | Bean decl |
| `@Profile` | Conditional on profile | Bean decl |
| `@Value` | Inject property/SpEL | Field/param |
| `@ConfigurationProperties` | Bind group of props | Class |
| `@PostConstruct` | Init callback | Method |
| `@PreDestroy` | Destroy callback | Method |
| `@PropertySource` | Load properties file | Config |
| `@Import` | Include other configs | Config |
| `@Order` | Ordering for BPP, aspects | Class |

### Bean Scope Summary

```
singleton  → 1 per container (default)
prototype  → new each request
request    → 1 per HTTP request
session    → 1 per HTTP session
application→ 1 per ServletContext
websocket  → 1 per WebSocket
```

### Lifecycle Order (Memorize!)

```
Create → Inject → *Aware setters → BPP.before → @PostConstruct 
→ afterPropertiesSet → init-method → BPP.after → READY
...
@PreDestroy → destroy → destroy-method
```

### DI Preference

```
✅ Constructor injection (final fields)
🟡 Setter injection (optional deps only)
❌ Field injection (avoid)
```

### Getting Beans

```java
ctx.getBean(UserService.class)              // by type
ctx.getBean("userService", UserService.class) // by name
ctx.getBeansOfType(Service.class)           // all
```

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Field injection | Constructor injection |
| Circular dependency | Redesign, or `@Lazy` |
| Prototype in singleton | `@Lookup` / `ObjectProvider` |
| `@Value` for grouped config | `@ConfigurationProperties` |
| Untrusted SpEL | `SimpleEvaluationContext` |
| Two beans same type | `@Primary` or `@Qualifier` |
| Config in `@Component` (not `@Configuration`) | Use `@Configuration` |

### Key Constants

```java
ConfigurableBeanFactory.SCOPE_SINGLETON   // "singleton"
ConfigurableBeanFactory.SCOPE_PROTOTYPE   // "prototype"
WebApplicationContext.SCOPE_REQUEST       // "request"
WebApplicationContext.SCOPE_SESSION       // "session"
WebApplicationContext.SCOPE_APPLICATION   // "application"
```

---

## Cross-References

- **Next file:** `02_Spring_AOP.md` — Aspect-Oriented Programming
- **Related topics:** `05_Spring_Transaction_Management.md` (uses AOP proxies), `06_Spring_Boot_Fundamentals.md` (auto-config uses conditions)
- **Interview file:** `24_Spring_Interview_Questions.md` (§Core Spring)

---

## Practice Exercises

1. **Build a standalone Spring app** (no Spring Boot) with `AnnotationConfigApplicationContext`, define 3 beans, wire them with constructor injection, and print the lifecycle order.
2. **Implement a custom `BeanPostProcessor`** that logs every bean name and its class.
3. **Write a `@Configuration` class** with 3 `@Bean` methods where one depends on another. Verify singleton behavior with `proxyBeanMethods=true` vs `false`.
4. **Create two `DataSource` beans** and resolve with `@Primary` and `@Qualifier`.
5. **Demonstrate a circular dependency** with setter injection (works) and constructor injection (fails). Fix it with `@Lazy`.
6. **Use SpEL** in `@Value` to inject: a system property, an arithmetic result, a regex match, and a bean method call.
7. **Implement `InitializingBean`, `DisposableBean`, `@PostConstruct`, `@PreDestroy`** in one class and observe the order.
8. **Write a `@ConfigurationProperties` class** with nested objects, list, and validation annotations. Bind from `application.yml`.

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** —
- **Next →:** [02_Spring_AOP.md](./02_Spring_AOP.md)
- **Related:** [02_Spring_AOP.md](./02_Spring_AOP.md), [06_Spring_Boot_Fundamentals.md](./06_Spring_Boot_Fundamentals.md), [13_Spring_Security_Core.md](./13_Spring_Security_Core.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
