# Aspect-Oriented Programming (AOP) in Spring

> **File:** `02_Spring_AOP.md`
> **Part:** 1 — Foundations
> **Prerequisites:** `01_Spring_Framework_Core.md` (IoC, DI, BeanPostProcessor)
> **Estimated Study Time:** 8–12 hours (read + code + practice)

---

## Table of Contents

1. [What is AOP?](#1-what-is-aop)
2. [Why AOP? Cross-Cutting Concerns](#2-why-aop-cross-cutting-concerns)
3. [AOP Core Concepts](#3-aop-core-concepts)
4. [Types of Advice](#4-types-of-advice)
5. [Pointcut Expressions](#5-pointcut-expressions)
6. [Combining Pointcuts](#6-combining-pointcuts)
7. [@AspectJ Annotation Style](#7-aspectj-annotation-style)
8. [XML-Based AOP Configuration](#8-xml-based-aop-configuration)
9. [Proxy Mechanisms: JDK vs CGLIB](#9-proxy-mechanisms-jdk-vs-cglib)
10. [When Spring Uses JDK vs CGLIB](#10-when-spring-uses-jdk-vs-cglib)
11. [AOP Use Cases](#11-aop-use-cases)
12. [Ordering Aspects](#12-ordering-aspects)
13. [Introductions (Inter-Type Declarations)](#13-introductions-inter-type-declarations)
14. [AOP Limitations & Pitfalls](#14-aop-limitations--pitfalls)
15. [AspectJ vs Spring AOP](#15-aspectj-vs-spring-aop)
16. [Interview Questions](#16-interview-questions)
17. [Summary Cheat Sheet](#17-summary-cheat-sheet)

---

## 1. What is AOP?

**Aspect-Oriented Programming (AOP)** is a programming paradigm that aims to increase **modularity** by allowing the **separation of cross-cutting concerns**. It complements Object-Oriented Programming (OOP) by providing another way of thinking about program structure.

> **In OOP**, the unit of modularity is the **class**.
> **In AOP**, the unit of modularity is the **aspect**.

### The Problem AOP Solves

In OOP, some concerns cannot be cleanly decomposed into a single class — they naturally cut across multiple classes:

- **Logging**
- **Security / Authentication / Authorization**
- **Transaction management**
- **Caching**
- **Performance monitoring / metrics**
- **Auditing**
- **Error handling**
- **Retry logic**
- **Rate limiting**

These are called **cross-cutting concerns** because they "cut across" the typical class hierarchy.

### Simple Illustration

**Without AOP** — the same logging code is scattered:

```java
public class OrderService {
    public void placeOrder(Order order) {
        long start = System.currentTimeMillis();
        log.info("Entering placeOrder with {}", order);
        try {
            // actual business logic
            validate(order);
            process(order);
        } catch (Exception e) {
            log.error("Error in placeOrder", e);
            throw e;
        } finally {
            log.info("Exiting placeOrder in {} ms", System.currentTimeMillis() - start);
        }
    }
}

public class UserService {
    public void createUser(User user) {
        long start = System.currentTimeMillis();
        log.info("Entering createUser with {}", user);
        try {
            // actual business logic
            validate(user);
            save(user);
        } catch (Exception e) {
            log.error("Error in createUser", e);
            throw e;
        } finally {
            log.info("Exiting createUser in {} ms", System.currentTimeMillis() - start);
        }
    }
}
```

**With AOP** — logic stays clean, cross-cutting concern centralized:

```java
public class OrderService {
    public void placeOrder(Order order) {
        validate(order);
        process(order);
    }
}

public class UserService {
    public void createUser(User user) {
        validate(user);
        save(user);
    }
}

@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        log.info("Entering {}", pjp.getSignature());
        try {
            return pjp.proceed();
        } finally {
            log.info("Exiting {} in {} ms", 
                     pjp.getSignature(), System.currentTimeMillis() - start);
        }
    }
}
```

---

## 2. Why AOP? Cross-Cutting Concerns

### Definition

A **cross-cutting concern** is a concern that affects multiple classes/modules and cannot be cleanly modularized into a single class in OOP.

### Examples

| Concern | Affects | Example behavior |
|---------|---------|-----------------|
| **Logging** | Every method | Log entry/exit/args |
| **Security** | Sensitive methods | Check `@PreAuthorize` |
| **Transactions** | Service methods | Begin/commit/rollback |
| **Caching** | Read methods | Return cached value |
| **Metrics** | API endpoints | Count calls, measure latency |
| **Auditing** | Persistence methods | Record who/what/when |
| **Retry** | External calls | Retry on failure |
| **Rate limiting** | Public APIs | Throttle calls |

### AOP Benefits

| Benefit | Description |
|---------|-------------|
| **Modularity** | Centralize concern in one place |
| **Reusability** | Apply concern to many methods via config |
| **Maintainability** | Change once, applies everywhere |
| **Readability** | Business methods stay focused |
| **DRY** | No repeated boilerplate |
| **Separation of concerns** | Business logic vs infrastructure |
| **Non-invasive** | No code change to business methods |

### AOP is Everywhere in Spring

- `@Transactional` — implemented via AOP
- `@Cacheable` — implemented via AOP
- `@PreAuthorize` — implemented via AOP
- `@Async` — implemented via AOP
- `@Retryable` (Spring Retry) — implemented via AOP
- `@Validated` — implemented via AOP
- `@Scheduled` — *not* AOP (uses ScheduledTaskRegistrar)

Understanding AOP is understanding **how half of Spring works**.

---

## 3. AOP Core Concepts

### The Six Key Terms

```
┌─────────────────────────────────────────────────────────────┐
│  1. Aspect        — What (cross-cutting concern module)     │
│  2. Join Point    — Where possible (execution candidates)   │
│  3. Pointcut      — Where exactly (predicate/expression)    │
│  4. Advice        — When + What (action at join point)      │
│  5. Target        — Whom (the business object)              │
│  6. Weaving       — How (integration with target)           │
└─────────────────────────────────────────────────────────────┘
```

### 3.1 Aspect

A **module** that encapsulates a cross-cutting concern. In Spring, it's a class annotated with `@Aspect`.

```java
@Aspect
@Component
public class LoggingAspect {
    // advices live here
}
```

### 3.2 Join Point

A **candidate point** in program execution where advice can be applied. In Spring AOP, **only method execution** is a join point.

> ⚠️ **Spring AOP only supports method execution join points**. Full AspectJ also supports constructor calls, field access, static initialization, etc.

### 3.3 Pointcut

A **predicate** that matches join points. Defines **where** advice applies.

```java
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceMethods() { }
```

### 3.4 Advice

The **action** taken at a matched join point. Defines **what** and **when**.

Types: `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`.

### 3.5 Target

The **object** being advised (the original business object).

### 3.6 Weaving

The process of **linking aspects with target objects** to create advised (proxied) objects.

**Weaving times:**

| Time | Description | Used by |
|------|-------------|---------|
| **Compile time** | During compilation | AspectJ (ajc compiler) |
| **Load time** | During class loading | AspectJ LTW |
| **Runtime** | When beans are created | **Spring AOP** |

### Diagram

```
                     ┌──────────────────────┐
                     │       ASPECT         │
                     │  (logging, security) │
                     └──────────┬───────────┘
                                │
                           POINTCUT
                          (where exactly)
                                │
                                ▼
    ┌─────────────┐       ┌──────────────┐
    │ Join Points │◄──────│   WEAVING    │
    │ (candidates)│       │  (runtime)   │
    └──────┬──────┘       └──────┬───────┘
           │                     │
           ▼                     ▼
    ┌──────────────────────────────────────┐
    │            TARGET OBJECT              │
    │   (OrderService, UserService, ...)    │
    └────────────────┬─────────────────────┘
                     │
                     ▼
                ADVICE called
             (@Before, @Around, ...)
```

### Complete Picture

```java
// 1. ASPECT
@Aspect
@Component
public class AuditAspect {
    
    // 2. POINTCUT
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() { }
    
    // 3. ADVICE (which is applied to a JOIN POINT on TARGET)
    @Before("serviceLayer()")
    public void before(JoinPoint jp) {
        log.info("Calling {}", jp.getSignature());
    }
}
```

---

## 4. Types of Advice

Spring AOP supports five advice types.

### 4.1 @Before

Runs **before** the join point. Cannot prevent execution (unless it throws).

```java
@Before("execution(* com.example.service.*.*(..))")
public void before(JoinPoint jp) {
    log.info("Before: {}", jp.getSignature().toShortString());
}
```

### 4.2 @AfterReturning

Runs **after** successful return. Can access the return value.

```java
@AfterReturning(
    pointcut = "execution(* com.example.service.*.*(..))",
    returning = "result"
)
public void afterReturning(JoinPoint jp, Object result) {
    log.info("Returned: {} from {}", result, jp.getSignature());
}
```

### 4.3 @AfterThrowing

Runs **after** an exception is thrown. Can access the exception.

```java
@AfterThrowing(
    pointcut = "execution(* com.example.service.*.*(..))",
    throwing = "ex"
)
public void afterThrowing(JoinPoint jp, Throwable ex) {
    log.error("Exception in {}: {}", jp.getSignature(), ex.getMessage());
}
```

### 4.4 @After (finally)

Runs **after** the method, regardless of outcome (like `finally`).

```java
@After("execution(* com.example.service.*.*(..))")
public void after(JoinPoint jp) {
    log.info("After: {}", jp.getSignature());
}
```

### 4.5 @Around (Most Powerful)

**Wraps** the join point. Must call `proceed()` to continue, or you can skip/alter it.

```java
@Around("execution(* com.example.service.*.*(..))")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    try {
        Object result = pjp.proceed();   // MUST call to invoke target
        return result;
    } finally {
        long elapsed = System.currentTimeMillis() - start;
        log.info("{} took {} ms", pjp.getSignature(), elapsed);
    }
}
```

**Can:**
- Skip execution (don't call `proceed()`)
- Change arguments (`pjp.proceed(newArgs)`)
- Change return value
- Swallow exceptions
- Retry
- Add caching

### Advice Ordering (Single Aspect)

```
       ┌────────────────────────────────────┐
       │           @Around (before)          │
       ├────────────────────────────────────┤
       │             @Before                 │
       ├────────────────────────────────────┤
       │                                     │
       │     ═══ TARGET METHOD ═══           │
       │                                     │
       ├────────────────────────────────────┤
       │           @AfterReturning           │
       │           OR @AfterThrowing         │
       ├────────────────────────────────────┤
       │             @After                  │
       ├────────────────────────────────────┤
       │           @Around (after)           │
       └────────────────────────────────────┘
```

### Advice Comparison Table

| Advice | Runs | Can modify return | Can prevent exec | Access exception | Access return val |
|--------|------|-------------------|------------------|------------------|-------------------|
| `@Before` | Before | ❌ | Only by throwing | ❌ | ❌ |
| `@After` | After (finally) | ❌ | ❌ | ❌ | ❌ |
| `@AfterReturning` | After success | ❌ | ❌ | ❌ | ✅ |
| `@AfterThrowing` | After exception | ❌ | ❌ | ✅ | ❌ |
| `@Around` | Around | ✅ | ✅ (skip proceed) | ✅ | ✅ |

### ProceedingJoinPoint vs JoinPoint

| Feature | JoinPoint | ProceedingJoinPoint |
|---------|-----------|---------------------|
| Used by | `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing` | `@Around` only |
| `proceed()` | ❌ | ✅ |
| `getArgs()` | ✅ | ✅ |
| `getSignature()` | ✅ | ✅ |
| `getTarget()` | ✅ | ✅ |
| `getThis()` | ✅ | ✅ |
| `getKind()` | ✅ | ✅ |

### Accessing Method Arguments

```java
@Before("execution(* com.example.service.*.*(..))")
public void before(JoinPoint jp) {
    Object[] args = jp.getArgs();
    MethodSignature sig = (MethodSignature) jp.getSignature();
    String method = sig.getName();
    Class<?>[] types = sig.getParameterTypes();
}
```

### Binding Arguments by Name

```java
@Before("execution(* com.example.service.*.*(..)) && args(userId, ..)")
public void before(Long userId) {
    log.info("userId = {}", userId);
}

// Or with annotation binding
@Before("@annotation(com.example.Auditable) && args(order, ..)")
public void before(Order order) { ... }
```

---

## 5. Pointcut Expressions

### Designators

Spring AOP supports the following pointcut designators (PCDs):

| PCD | Description |
|-----|-------------|
| `execution` | Match method execution (most common) |
| `within` | Match types (classes/packages) |
| `this` | Match proxy is instance of type |
| `target` | Match target is instance of type |
| `args` | Match method args by type |
| `@target` | Target class has annotation |
| `@within` | Class (declaring type) has annotation |
| `@args` | Runtime arg types have annotation |
| `@annotation` | Method has annotation |
| `bean` | Match by bean name (Spring-specific) |

> ⚠️ **Not supported in Spring AOP** (AspectJ only): `call`, `get`, `set`, `initialization`, `preinitialization`, `staticinitialization`, `handler`, `withincode`, `cflow`, `if`.

### execution — Full Syntax

```
execution(modifiers? return-type declaring-type?.method-name(param-types) throws?)
```

Only `return-type`, `method-name`, and `param-types` are **required**.

### execution Examples

```java
// Any public method in com.example.service
execution(public * com.example.service.*.*(..))

// Any method (any visibility) in com.example.service and subpackages
execution(* com.example.service..*.*(..))

// Any method named "find*" with any args
execution(* find*(..))

// Any method returning String
execution(String *(..))

// Methods taking exactly one Long arg
execution(* *(Long))

// Methods taking Long as FIRST arg (any others)
execution(* *(Long, ..))

// Method named save on any class
execution(* save(..))

// Method save on OrderService only
execution(* com.example.OrderService.save(..))

// Any method throwing Exception
execution(* *(..) throws Exception)
```

### Wildcards

| Symbol | Meaning |
|--------|---------|
| `*` | Any number of chars (within a segment) |
| `..` | Any number of segments (packages/args) |
| `+` | Subtype (after type name) |

**Examples:**

```java
execution(* com.example.*.*(..))         // one package level
execution(* com.example..*.*(..))        // package and subpackages
execution(* com.example..*(..))          // same as above
execution(* *(..))                       // any method any package
execution(* Service+.*(..))              // Service or its subtypes
```

### within

Match by **type** (class/package). No method-level matching.

```java
within(com.example.service.*)          // classes directly in service
within(com.example.service..*)         // service + subpackages
within(OrderService)                   // specific class
within(OrderService+)                  // OrderService and subtypes
within(@org.springframework.stereotype.Service *)  // annotated with @Service
```

### this vs target

- **`this`** — matches when the **proxy** is an instance of the type.
- **`target`** — matches when the **target object** is an instance of the type.

```java
this(com.example.MyInterface)          // proxy implements MyInterface
target(com.example.MyClass)            // target is MyClass
```

**Why they differ:** With JDK proxies, the proxy implements only interfaces, so `target(ConcreteClass)` matches but `this(ConcreteClass)` doesn't. With CGLIB, both match.

### args

Match by **argument types at runtime**.

```java
args(java.lang.String)                    // exactly one String arg
args(java.lang.String, ..)                // first arg String
args(.., java.lang.String)                // last arg String
args(java.lang.String, java.lang.Long)    // exact two args
```

**Difference from execution param matching:** `args` uses runtime types (subtypes match); `execution` uses declared types.

### @annotation

Match methods annotated with a given annotation.

```java
@annotation(org.springframework.transaction.annotation.Transactional)
@annotation(com.example.Loggable)
@annotation(com.example.Auditable)
```

### @within / @target

- **`@within`** — declaring class has annotation.
- **`@target`** — target object's runtime class has annotation.

```java
@within(org.springframework.stereotype.Service)
@target(com.example.Marker)
```

### bean (Spring-specific)

Match by **bean name**.

```java
bean(*Service)                // beans ending in Service
bean(orderService)            // specific bean name
bean(userService || orderService)
```

### Real-World Pointcut Examples

```java
// All public methods in service layer
@Pointcut("execution(public * com.example..service..*(..))")

// All @Transactional methods (whether annotated on method or class)
@Pointcut("@annotation(org.springframework.transaction.annotation.Transactional) || "
        + "@within(org.springframework.transaction.annotation.Transactional)")

// Any Spring bean with @Service
@Pointcut("within(@org.springframework.stereotype.Service *)")

// Any method in classes whose name ends with "Controller"
@Pointcut("within(*..*Controller)")

// Any method that takes exactly one @RequestBody-annotated param
@Pointcut("execution(* *(..)) && @args(org.springframework.web.bind.annotation.RequestBody)")
```

---

## 6. Combining Pointcuts

Pointcuts can be **combined** with logical operators.

| Operator | Meaning | Symbol | Keyword |
|----------|---------|--------|---------|
| AND | Both match | `&&` | `and` |
| OR | Either matches | `\|\|` | `or` |
| NOT | Negation | `!` | `not` |

### Examples

```java
// Service methods that aren't private
execution(* com.example.service.*.*(..)) && !execution(private * *(..))

// @Transactional OR @Cacheable
@annotation(org.springframework.transaction.annotation.Transactional) ||
@annotation(org.springframework.cache.annotation.Cacheable)

// Service layer AND public
within(@org.springframework.stereotype.Service *) && execution(public * *(..))
```

### Reusable Pointcuts

Named pointcuts can be **referenced by other aspects** if they're in a public class.

```java
@Aspect
@Component
public class Pointcuts {
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() { }
    
    @Pointcut("within(@org.springframework.stereotype.Repository *)")
    public void repositoryLayer() { }
}
```

**Reference from another aspect:**

```java
@Aspect
@Component
public class AuditAspect {
    @Before("com.example.aspects.Pointcuts.serviceLayer()")
    public void audit(JoinPoint jp) { }
}
```

---

## 7. @AspectJ Annotation Style

### Enabling

**Option A: `@EnableAspectJAutoProxy`** (standalone Spring)

```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig { }
```

**Option B: Spring Boot** — enabled automatically by `AopAutoConfiguration`.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### Full Working Example

```java
@Aspect
@Component
@Order(1)
public class LoggingAspect {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);
    
    // Reusable pointcut
    @Pointcut("execution(public * com.example.service..*(..))")
    public void serviceLayer() { }
    
    @Pointcut("@annotation(com.example.Loggable)")
    public void loggableMethods() { }
    
    // ========== ADVICES ==========
    
    @Before("serviceLayer()")
    public void beforeAdvice(JoinPoint jp) {
        log.info("[BEFORE] {}", jp.getSignature().toShortString());
    }
    
    @AfterReturning(
        pointcut = "serviceLayer()",
        returning = "result"
    )
    public void afterReturning(JoinPoint jp, Object result) {
        log.info("[RETURN] {} → {}", jp.getSignature(), result);
    }
    
    @AfterThrowing(
        pointcut = "serviceLayer()",
        throwing = "ex"
    )
    public void afterThrowing(JoinPoint jp, Throwable ex) {
        log.error("[THROW] {} → {}", jp.getSignature(), ex.getMessage());
    }
    
    @After("serviceLayer()")
    public void afterAdvice(JoinPoint jp) {
        log.info("[AFTER] {}", jp.getSignature());
    }
    
    @Around("loggableMethods()")
    public Object aroundAdvice(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            log.info("[PERF] {} took {} ms",
                     pjp.getSignature(), System.currentTimeMillis() - start);
        }
    }
}
```

### Custom Marker Annotation

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable { }
```

**Usage:**

```java
@Service
public class UserService {
    @Loggable
    public User findById(Long id) { ... }   // perf-timed
}
```

### Accessing Annotation Value

```java
@Around("@annotation(auditable)")
public Object audit(ProceedingJoinPoint pjp, Auditable auditable) throws Throwable {
    log.info("Action={}, Auditable.value={}", 
             pjp.getSignature().getName(), auditable.value());
    return pjp.proceed();
}
```

---

## 8. XML-Based AOP Configuration

Legacy but still valid. Useful when aspects must be configured declaratively.

```xml
<aop:config>
    <aop:pointcut id="serviceLayer"
                  expression="execution(* com.example.service.*.*(..))"/>
    
    <aop:aspect id="loggingAspect" ref="loggingAspectBean" order="1">
        <aop:before pointcut-ref="serviceLayer" method="beforeAdvice"/>
        <aop:after-returning pointcut-ref="serviceLayer"
                             method="afterReturning"
                             returning="result"/>
        <aop:after-throwing pointcut-ref="serviceLayer"
                            method="afterThrowing"
                            throwing="ex"/>
        <aop:after pointcut-ref="serviceLayer" method="afterAdvice"/>
        <aop:around pointcut-ref="serviceLayer" method="aroundAdvice"/>
    </aop:aspect>
</aop:config>

<bean id="loggingAspectBean" class="com.example.LoggingAspect"/>
```

**Mixed mode (schema-based + @AspectJ):** Don't mix in the same context — choose one.

### @EnableAspectJAutoProxy Attributes

```java
@Configuration
@EnableAspectJAutoProxy(
    proxyTargetClass = true,        // force CGLIB
    exposeProxy = true              // allow AopContext.currentProxy()
)
public class AppConfig { }
```

| Attribute | Default | Purpose |
|-----------|---------|---------|
| `proxyTargetClass` | `false` | If true, always CGLIB |
| `exposeProxy` | `false` | Expose current proxy via `AopContext` |

### Spring Boot Auto-Configuration

Spring Boot auto-enables AOP if `aspectjweaver` is on the classpath:

```properties
spring.aop.auto=true                # default
spring.aop.proxy-target-class=true  # default since Spring Boot 2.0
```

---

## 9. Proxy Mechanisms: JDK vs CGLIB

Spring AOP is **proxy-based**. It never modifies bytecode (unlike AspectJ's compile/load-time weaving).

### JDK Dynamic Proxy

- Built into JDK (`java.lang.reflect.Proxy`)
- Works only for **interfaces**
- The proxy **implements the same interfaces** as the target
- Invocation routed to `InvocationHandler`

```java
public interface UserService {
    User findById(Long id);
}

public class UserServiceImpl implements UserService {
    public User findById(Long id) { return new User(id); }
}

// Spring creates:
UserService proxy = (UserService) Proxy.newProxyInstance(
    classLoader,
    new Class[]{UserService.class},   // interface only
    (p, method, args) -> { /* invoke aspect + target */ }
);
```

### CGLIB Proxy

- **Subclasses** the target class at runtime
- Works for concrete classes
- Cannot proxy `final` classes or `final` methods
- Requires a no-arg constructor (unless using Objenesis)
- Bundled with Spring (repackage `org.springframework.cglib`)

```java
public class UserServiceImpl {
    public User findById(Long id) { return new User(id); }
}

// Spring creates:
UserServiceImpl proxy = /* CGLIB subclass */;
```

### Comparison Table

| Aspect | JDK Proxy | CGLIB |
|--------|-----------|-------|
| Requires interface | ✅ Yes | ❌ No |
| Works with classes | ❌ | ✅ |
| Final classes | N/A | ❌ Fails |
| Final methods | N/A | ❌ Not advised |
| Private methods | N/A | ❌ Not advised |
| Performance (create) | Fast | Slower |
| Performance (call) | Fast | Fast |
| Bundled in JDK | ✅ | ❌ (Spring repackages) |
| Requires no-arg ctor | ❌ | ✅ (or Objenesis) |

### Why Spring Boot Defaults to CGLIB

Since **Spring Boot 2.0**, `spring.aop.proxy-target-class=true` by default:
- Simpler mental model — no need for interfaces
- Avoids surprising "interface not injected" bugs
- Slight perf/memory tradeoff, but negligible

---

## 10. When Spring Uses JDK vs CGLIB

Spring decides as follows:

```
1. If proxyTargetClass = true → CGLIB (always)
2. If proxyTargetClass = false (Spring default pre-Boot 2.0):
     If target implements ≥1 interface → JDK
     Else → CGLIB
3. Spring Boot 2.0+ → proxyTargetClass = true → CGLIB
```

### Forcing a Specific Proxy

```java
// Force CGLIB globally
@EnableAspectJAutoProxy(proxyTargetClass = true)

// Force CGLIB for specific bean
@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS)

// Force JDK for specific bean
@Scope(proxyMode = ScopedProxyMode.INTERFACES)
```

### Detection: Is Your Bean Proxied?

```java
@Autowired
private UserService userService;

public void check() {
    System.out.println(userService.getClass().getName());
    // com.example.UserServiceImpl$$EnhancerBySpringCGLIB$$abc123
    //  OR
    // jdk.proxy2.$Proxy56
    
    System.out.println(AopUtils.isAopProxy(userService));       // true
    System.out.println(AopUtils.isJdkDynamicProxy(userService)); // false
    System.out.println(AopUtils.isCglibProxy(userService));      // true
}
```

### Critical Implications

**1. Only public methods can be advised (for CGLIB) and none can be final.**

**2. Self-invocation bypasses the proxy!**

```java
@Service
public class OrderService {
    public void placeOrder() {
        validate();     // ⚠️ NOT proxied — direct call, not through proxy
    }
    
    @Transactional
    public void validate() { ... }
}
```

The internal call to `validate()` is a plain Java call — the proxy is **not** involved. **Transaction advice is skipped.**

**Fix options:**
- Move `validate()` to another bean
- Use `AopContext.currentProxy()` (requires `exposeProxy=true`)
- Inject self
- Use `@Transactional` on the outer method

**3. `final` methods cannot be advised by CGLIB.**

**4. `private` methods cannot be advised (by either proxy).**

**5. `final` classes cannot be CGLIB-proxied.**

---

## 11. AOP Use Cases

### 11.1 Logging

```java
@Aspect
@Component
public class LoggingAspect {
    @Around("@within(org.springframework.stereotype.Service)")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        log.info("→ {} args={}", pjp.getSignature().toShortString(), pjp.getArgs());
        try {
            Object result = pjp.proceed();
            log.info("← {} = {}", pjp.getSignature().getName(), result);
            return result;
        } catch (Exception e) {
            log.error("✗ {} : {}", pjp.getSignature().getName(), e.getMessage());
            throw e;
        }
    }
}
```

### 11.2 Performance Monitoring

```java
@Aspect
@Component
public class PerformanceAspect {
    @Around("execution(* com.example..*(..))")
    public Object measure(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long ms = (System.nanoTime() - start) / 1_000_000;
            if (ms > 500) {
                log.warn("SLOW: {} took {} ms", pjp.getSignature(), ms);
            }
        }
    }
}
```

### 11.3 Security

```java
@Aspect
@Component
public class SecurityAspect {
    
    @Before("@annotation(requiresAdmin)")
    public void checkAdmin(RequiresAdmin requiresAdmin) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (!auth.getAuthorities().contains(new SimpleGrantedAuthority("ROLE_ADMIN"))) {
            throw new AccessDeniedException("Admin only");
        }
    }
}
```

### 11.4 Auditing

```java
@Aspect
@Component
public class AuditAspect {
    
    @AfterReturning(
        pointcut = "@annotation(auditable)",
        returning = "result"
    )
    public void audit(JoinPoint jp, Auditable auditable, Object result) {
        String user = SecurityContextHolder.getContext()
                          .getAuthentication().getName();
        auditRepository.save(new AuditRecord(
            user, auditable.value(), jp.getSignature().toString(),
            Instant.now(), result != null ? result.toString() : null
        ));
    }
}
```

### 11.5 Retry Logic

```java
@Aspect
@Component
public class RetryAspect {
    
    @Around("@annotation(retry)")
    public Object retry(ProceedingJoinPoint pjp, Retry retry) throws Throwable {
        int attempts = 0;
        Throwable lastEx = null;
        while (attempts < retry.maxAttempts()) {
            try {
                return pjp.proceed();
            } catch (Throwable t) {
                lastEx = t;
                attempts++;
                Thread.sleep(retry.delayMs());
            }
        }
        throw lastEx;
    }
}
```

### 11.6 Rate Limiting

```java
@Aspect
@Component
public class RateLimitAspect {
    private final Map<String, AtomicInteger> counters = new ConcurrentHashMap<>();
    
    @Around("@annotation(rateLimit)")
    public Object limit(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable {
        String key = pjp.getSignature().toShortString();
        AtomicInteger count = counters.computeIfAbsent(key, k -> new AtomicInteger());
        if (count.incrementAndGet() > rateLimit.value()) {
            throw new TooManyRequestsException();
        }
        return pjp.proceed();
    }
}
```

### 11.7 Caching (simplified)

```java
@Aspect
@Component
public class SimpleCacheAspect {
    private final Map<String, Object> cache = new ConcurrentHashMap<>();
    
    @Around("@annotation(Cacheable)")
    public Object cache(ProceedingJoinPoint pjp, Cacheable cacheable) throws Throwable {
        String key = pjp.getSignature() + Arrays.toString(pjp.getArgs());
        return cache.computeIfAbsent(key, k -> {
            try { return pjp.proceed(); }
            catch (Throwable t) { throw new RuntimeException(t); }
        });
    }
}
```

### 11.8 Validation

```java
@Aspect
@Component
public class ValidationAspect {
    
    @Before("execution(* com.example.service.*.*(..)) && args(entity, ..)")
    public void validate(Object entity) {
        Set<ConstraintViolation<Object>> violations = 
            ValidatorFactoryHolder.getValidator().validate(entity);
        if (!violations.isEmpty()) {
            throw new ConstraintViolationException(violations);
        }
    }
}
```

### 11.9 Distributed Tracing (correlation IDs)

```java
@Aspect
@Component
public class TraceAspect {
    
    @Around("execution(* com.example..*(..))")
    public Object trace(ProceedingJoinPoint pjp) throws Throwable {
        String correlationId = MDC.get("correlationId");
        if (correlationId == null) {
            MDC.put("correlationId", UUID.randomUUID().toString());
        }
        try { return pjp.proceed(); }
        finally { MDC.remove("correlationId"); }
    }
}
```

---

## 12. Ordering Aspects

When multiple aspects match the same join point, order matters.

### @Order Annotation

```java
@Aspect
@Component
@Order(1)         // lower = higher priority = runs "more outside"
public class SecurityAspect { }

@Aspect
@Component
@Order(2)
public class LoggingAspect { }

@Aspect
@Component
@Order(3)
public class TransactionAspect { }
```

**Order semantics:**
- **Lower number = higher priority = runs FIRST on entry, LAST on exit** (like an onion).
- If two aspects have the same order, order is undefined.

### Ordered Interface

```java
@Aspect
@Component
public class MyAspect implements Ordered {
    @Override
    public int getOrder() { return 10; }
}
```

### Combined Ordering Example

```java
@Order(1) SecurityAspect {
    @Around:
        // BEFORE target
        pjp.proceed();
        // AFTER target
}

@Order(2) LoggingAspect {
    @Around:
        // BEFORE target
        pjp.proceed();
        // AFTER target
}
```

**Execution flow:**

```
SecurityAspect.before
  → LoggingAspect.before
      → Target.method()
      ← LoggingAspect.after
  ← SecurityAspect.after
```

Like an onion: **higher-priority aspect surrounds lower-priority**.

### Special Order Constants

Spring's own aspects use specific orders:

| Aspect | Order |
|--------|-------|
| `ExposeInvocationInterceptor` | `Ordered.HIGHEST_PRECEDENCE + 1` |
| `AspectJAfterThrowingAdvice` | `HIGHEST` (internal) |
| `@Transactional` | `Ordered.LOWEST_PRECEDENCE` (default) |
| `@Async` | `Ordered.LOWEST_PRECEDENCE` |
| `@Cacheable` | default order |
| `@Validated` | `HIGHEST_PRECEDENCE` (roughly) |

### Best-Practice Ordering

| Layer | Order |
|-------|-------|
| Tracing / Correlation ID | `Integer.MIN_VALUE + 100` |
| Security | `0` |
| Logging | `100` |
| Transactions | `200` |
| Business aspects | `500` |
| Caching | `1000` (often inside tx) |

---

## 13. Introductions (Inter-Type Declarations)

An **introduction** allows an aspect to declare that a target object **implements a new interface**, and provides a default implementation.

### @DeclareParents

```java
public interface Auditable { 
    void audit(); 
}

@Aspect
@Component
public class AuditIntroduction {
    
    @DeclareParents(
        value = "com.example.service.*+",         // target types
        defaultImpl = AuditableImpl.class         // implementation
    )
    public static Auditable auditable;
}

public class AuditableImpl implements Auditable {
    @Override
    public void audit() {
        System.out.println("Audited!");
    }
}
```

Now any `com.example.service.*` bean can be **cast to `Auditable`**:

```java
@Autowired
private UserService userService;

public void demo() {
    ((Auditable) userService).audit();     // works!
}
```

### When to Use

Rarely. Useful for:
- Adding a cross-cutting capability (e.g., `Auditable`, `Versionable`)
- Framework-level concerns
- Legacy code where you can't change class definitions

### Pitfalls

- Requires proxy (JDK or CGLIB)
- Only works on Spring beans
- Can surprise developers who don't expect the cast to work
- Doesn't show up in IDEs

---

## 14. AOP Limitations & Pitfalls

### 14.1 Self-Invocation (The #1 Pitfall)

```java
@Service
public class OrderService {
    
    public void placeOrder() {
        validate();     // ❌ Proxy bypassed → no advice
    }
    
    @Transactional
    public void validate() { ... }
}
```

**Why:** The proxy is external. Internal `this.validate()` calls don't go through the proxy.

**Fixes:**
1. Move `validate()` to a separate bean
2. Use `AopContext.currentProxy().validate()` (with `exposeProxy=true`)
3. Inject self:
```java
@Service
public class OrderService {
    @Autowired
    private OrderService self;   // self-injected proxy
    
    public void placeOrder() {
        self.validate();     // ✅ proxy call
    }
}
```

### 14.2 Only Public Methods

Both JDK proxies and CGLIB proxies only advise **public** methods.

- `private`, `protected`, package-private → **not advised**
- CGLIB technically can override protected, but Spring doesn't advise them

### 14.3 Final Classes / Methods

- **`final` class** → cannot be CGLIB-proxied → falls back to JDK (needs interface)
- **`final` method** → cannot be overridden → cannot be advised

### 14.4 Proxy vs Target Type Mismatch

With JDK proxy:

```java
@Autowired
private UserServiceImpl userService;  // ❌ NoSuchBeanDefinitionException

// Correct:
@Autowired
private UserService userService;      // ✅ interface
```

Fix: use interfaces, or set `proxyTargetClass=true`.

### 14.5 Constructor Injection Not Advised

The constructor runs **before** the object is proxied. Advice never wraps the constructor.

### 14.6 Advising Internal Infrastructure

Avoid advising `@Configuration` classes or Spring's own beans. Prefer narrower pointcuts.

### 14.7 Performance Overhead

Each proxied call incurs overhead (reflection, chain traversal). Keep aspects **small and fast**.

### 14.8 Aspect Nesting & Unexpected Order

Multiple aspects + `@Order` conflicts → hard-to-debug behavior. Document orders.

### 14.9 Testing Complications

Testing advised beans requires `@SpringBootTest` or manual proxy setup. `new MyService()` bypasses advice.

### 14.10 Stack Trace Unreadable

Proxies add frames (`$Proxy`, `EnhancerByCGLIB`). Debugging requires care.

### 14.11 Aspect Bean Requires No Circular Ref

An aspect that depends on a service that is itself advised by that aspect → circular. Keep aspects dependency-light.

### 14.12 Pointcut Typos Fail Silently

Misspelled pointcuts match **nothing** — no error, no advice. Always test with `AopUtils` / `Debug`.

Enable debug logging:

```properties
logging.level.org.springframework.aop=DEBUG
```

---

## 15. AspectJ vs Spring AOP

| Feature | Spring AOP | AspectJ |
|---------|-----------|---------|
| Weaving time | Runtime | Compile / Load / Runtime |
| Join points | Method execution only | All (method, constructor, field, static) |
| Requires `ajc` compiler | ❌ | ✅ (for compile-time) |
| Requires proxy | ✅ | ❌ |
| Self-invocation | ❌ Not advised | ✅ Advised |
| `final` classes | ❌ (CGLIB fails) | ✅ |
| `private` methods | ❌ | ✅ |
| Performance | Slower (proxy) | Faster |
| Complexity | Low | High |
| Spring integration | Native | Supported |
| Pointcut designators | Limited | Full |

### When to Use AspectJ over Spring AOP

- You need to advise `private`, `final`, or `static` methods
- You need constructor/field join points
- You need compile-time weaving (e.g., Android, GraalVM)
- You want to avoid proxying overhead
- Self-invocation must be advised

### Enabling AspectJ in Spring

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <version>1.14.0</version>
    <configuration>
        <source>17</source>
        <target>17</target>
    </configuration>
</plugin>
```

Or **load-time weaving (LTW)** with `-javaagent:aspectjweaver.jar`:

```java
@EnableLoadTimeWeaving
@Configuration
public class AppConfig { }
```

---

## 16. Interview Questions

### Q1. What is AOP and why is it used?

**Answer:** AOP (Aspect-Oriented Programming) is a paradigm for modularizing cross-cutting concerns (logging, security, transactions, caching) that can't be cleanly isolated in OOP. It separates these concerns into **aspects** and applies them to target methods via **pointcuts**. Spring AOP is used to implement `@Transactional`, `@Cacheable`, `@PreAuthorize`, `@Async`, etc.

### Q2. Explain the core AOP concepts.

**Answer:**
- **Aspect** — module containing cross-cutting logic (`@Aspect` class)
- **Join Point** — candidate execution point (Spring: only method execution)
- **Pointcut** — predicate selecting join points (e.g., `execution(* service.*.*(..))`)
- **Advice** — action taken (`@Before`, `@Around`, etc.)
- **Target** — object being advised
- **Weaving** — linking aspect to target (Spring: runtime via proxy)
- **Proxy** — object that intercepts calls and applies advice

### Q3. What advice types does Spring support?

**Answer:** Five:
- `@Before` — before method
- `@AfterReturning` — after successful return
- `@AfterThrowing` — after exception
- `@After` — after method (finally)
- `@Around` — wraps method; most powerful (can skip/modify)

### Q4. Difference between @Before and @Around?

**Answer:** `@Before` runs before the method and cannot modify its behavior (except by throwing). `@Around` wraps the method — it receives `ProceedingJoinPoint`, can call `proceed()` zero or more times, can change arguments and return value, and can skip execution entirely.

### Q5. JDK dynamic proxy vs CGLIB?

**Answer:** JDK proxy is JDK-built, works only for interface-implementing targets, and produces a proxy that implements the same interfaces. CGLIB subclasses the target class at runtime, works for concrete classes, but cannot proxy `final` classes/methods. Spring Boot 2.0+ defaults to CGLIB.

### Q6. When does Spring use each proxy type?

**Answer:**
- `proxyTargetClass=true` → CGLIB
- `proxyTargetClass=false` (Spring default pre-Boot 2.0): JDK if target implements interfaces, else CGLIB
- Spring Boot 2.0+ → CGLIB by default

### Q7. What is a pointcut expression? Give examples.

**Answer:** A predicate matching join points. Examples:
- `execution(* com.example.service.*.*(..))` — any method in `service` package
- `execution(public * *(..))` — all public methods
- `@annotation(org.springframework.transaction.annotation.Transactional)` — methods with `@Transactional`
- `within(com.example..*)` — any type in package and subpackages

### Q8. What is a self-invocation problem?

**Answer:** When a method within a bean calls another method of the same bean (`this.other()`), the call **bypasses the proxy**, so advice (e.g., `@Transactional`) does not apply. Fixes: extract to another bean, use `AopContext.currentProxy()`, or inject self.

### Q9. Can you advise private methods with Spring AOP?

**Answer:** No. Spring AOP only advises public methods on Spring-managed beans. Private/protected/package-private methods can't be advised via proxy. AspectJ (weaving) can advise them.

### Q10. Can you advise final methods or final classes?

**Answer:** No for CGLIB (can't subclass/override). No for JDK (no interface-based override possible). AspectJ can.

### Q11. What is the difference between `this` and `target` pointcut designators?

**Answer:** `this(Type)` matches when the **proxy** is an instance of Type. `target(Type)` matches when the **target object** is. With JDK proxies, `target(ConcreteClass)` matches but `this(ConcreteClass)` doesn't (proxy is a different class).

### Q12. How to order multiple aspects?

**Answer:** Use `@Order(n)` or implement `Ordered`. Lower number = higher precedence = runs first on entry, last on exit (like an onion). Spring's own aspects have fixed orders (e.g., `@Transactional` = `LOWEST_PRECEDENCE`).

### Q13. What is `@DeclareParents`?

**Answer:** An AspectJ annotation that introduces a **new interface** to target types at runtime, with a `defaultImpl` providing the implementation. Useful for adding cross-cutting capabilities (e.g., `Auditable`).

### Q14. What are the limitations of Spring AOP?

**Answer:**
- Only method execution join points
- Only public methods
- No `final` classes/methods (CGLIB)
- No self-invocation advice
- No constructor advice
- Runtime weaving only
- Proxy overhead
- Silent failure on typos in pointcuts

### Q15. Spring AOP vs AspectJ?

**Answer:** Spring AOP is proxy-based, runtime weaving, method-only. AspectJ is a full AOP language with compile/load-time weaving, supporting all join points (constructor, field, static). AspectJ is more powerful but complex. Spring AOP is easier and covers 95% of use cases.

### Q16. How is @Transactional implemented in Spring?

**Answer:** Via AOP. Spring creates a proxy around the bean. When a `@Transactional` method is called, the proxy invokes `TransactionInterceptor` (`@Around` advice) which begins/commits/rolls back the transaction around `proceed()`.

### Q17. Why do you get `NoSuchBeanDefinitionException` for a service class after adding AOP?

**Answer:** Because the bean is now a **proxy** implementing only interfaces (JDK proxy), and you tried to autowire by concrete class. Fix: inject by interface, or set `proxyTargetClass=true`.

### Q18. How to debug whether a bean is proxied?

**Answer:** `AopUtils.isAopProxy(bean)`, `AopUtils.isJdkDynamicProxy(bean)`, `AopUtils.isCglibProxy(bean)`. Or print `bean.getClass().getName()` — you'll see `$Proxy` (JDK) or `$$EnhancerBySpringCGLIB$$` (CGLIB).

### Q19. What does `@EnableAspectJAutoProxy(exposeProxy = true)` do?

**Answer:** It exposes the current proxy through `AopContext.currentProxy()`. Useful to work around self-invocation: `((MyService) AopContext.currentProxy()).otherMethod()`. Adds a ThreadLocal lookup.

### Q20. Can you combine multiple pointcuts?

**Answer:** Yes, with `&&`, `||`, `!` (or `and`, `or`, `not`). Also can define named pointcuts (`@Pointcut`) and reference them from other aspects.

### Q21. What is the difference between `execution` and `within`?

**Answer:** `execution` matches **methods** (name, args, return type). `within` matches **types** (classes/packages). `within(com.foo.*)` matches all methods of any class in `com.foo`.

### Q22. What does `@args` do?

**Answer:** Matches methods where the **runtime argument types** have a given annotation. Example: `@args(RequestBody)` matches methods with an argument whose runtime class is `@RequestBody`-annotated (rare but useful for framework-style logic).

### Q23. What is a `ProceedingJoinPoint`?

**Answer:** The `JoinPoint` subtype passed to `@Around`. Adds `proceed()` (to invoke the target) and `proceed(Object[])` (to invoke with modified args). Only `@Around` receives it.

### Q24. Does AOP work for beans created with `new`?

**Answer:** No. Spring AOP only works on **Spring-managed beans** obtained from the container. Objects created with `new` (outside the container) are not proxied.

### Q25. What happens if two aspects match with the same @Order?

**Answer:** Order is **undefined**; behavior may vary across runs or Spring versions. Always assign unique orders.

---

## 17. Summary Cheat Sheet

### Annotations

| Annotation | Purpose |
|------------|---------|
| `@Aspect` | Marks a class as an aspect |
| `@Component` | Register aspect as bean |
| `@Pointcut` | Named reusable pointcut |
| `@Before` | Advice: before method |
| `@After` | Advice: after (finally) |
| `@AfterReturning` | Advice: after success |
| `@AfterThrowing` | Advice: after exception |
| `@Around` | Advice: wraps method |
| `@Order` | Aspect ordering |
| `@DeclareParents` | Introduction (add interface) |
| `@EnableAspectJAutoProxy` | Enable AOP (standalone) |

### Pointcut Designators

| PCD | Matches |
|-----|---------|
| `execution(...)` | Method execution |
| `within(...)` | Type/package |
| `this(...)` | Proxy type |
| `target(...)` | Target type |
| `args(...)` | Runtime arg types |
| `@target(...)` | Target class annotation |
| `@within(...)` | Declaring class annotation |
| `@args(...)` | Arg type annotations |
| `@annotation(...)` | Method annotation |
| `bean(...)` | Spring bean name |

### Pointcut Wildcards

| Symbol | Meaning |
|--------|---------|
| `*` | Any chars in segment |
| `..` | Any segments |
| `+` | Subtype |

### Execution Syntax

```
execution(modifiers? return-type declaring-type?.method-name(params) throws?)
```

### Advice Cheat

```java
@Before("pc()")                          void before(JoinPoint jp)
@After("pc()")                           void after(JoinPoint jp)
@AfterReturning(pointcut="pc()", 
                returning="r")            void after(JoinPoint jp, Object r)
@AfterThrowing(pointcut="pc()", 
               throwing="e")              void after(JoinPoint jp, Throwable e)
@Around("pc()")                          Object around(ProceedingJoinPoint pjp)
```

### Proxy Decision

```
proxyTargetClass=true          → CGLIB
proxyTargetClass=false:
   target implements interface → JDK
   else                        → CGLIB
Spring Boot 2.0+               → CGLIB
```

### Common Pitfalls Checklist

- [ ] Method is **public**?
- [ ] No **self-invocation** (`this.method()`)?
- [ ] Class is **not final**?
- [ ] Method is **not final**?
- [ ] Bean is **Spring-managed** (not `new`)?
- [ ] Autowiring by **interface** (JDK proxy) or `proxyTargetClass=true`?
- [ ] Pointcut **expression correct** (no silent typos)?
- [ ] Aspect has proper **@Order**?
- [ ] Aspect beans are not caught by their own pointcuts?

### Debug Properties

```properties
logging.level.org.springframework.aop=DEBUG
logging.level.org.springframework.aop.framework=TRACE
```

### Useful APIs

```java
AopUtils.isAopProxy(bean);
AopUtils.isJdkDynamicProxy(bean);
AopUtils.isCglibProxy(bean);
AopContext.currentProxy();               // needs exposeProxy=true
ProxyMethodInvocation.getMethod();
```

### Multi-Aspect Execution Order (Onion Model)

```
Order(1) Security
  Order(2) Logging
    Order(3) Transaction
      Target.method()
    Order(3) Transaction after
  Order(2) Logging after
Order(1) Security after
```

---

## Cross-References

- **Previous:** `01_Spring_Framework_Core.md` — IoC, DI, BeanPostProcessor (AOP uses BPP)
- **Next:** `03_Spring_MVC.md` — Spring MVC & Web Layer
- **Related:** `05_Spring_Transaction_Management.md` — `@Transactional` is AOP-based
- **Related:** `12_Spring_Caching.md` — `@Cacheable` is AOP-based
- **Related:** `15_Spring_Security_Method_Level.md` — `@PreAuthorize` is AOP-based
- **Interview:** `24_Spring_Interview_Questions.md` (§AOP — 15 questions)

---

## Practice Exercises

1. **Build a standalone Spring AOP example** with `@EnableAspectJAutoProxy`, one aspect, one service, and log method entry/exit using `@Around`.
2. **Implement all five advice types** in a single aspect and observe their order using a service method that prints "business logic".
3. **Write a `@Loggable` marker annotation** and an `@Around` advice that times only annotated methods.
4. **Reproduce self-invocation failure** — a `@Transactional`-like aspect on an internal call. Fix using `AopContext.currentProxy()`.
5. **Force JDK proxy** with `proxyTargetClass=false` and observe `NoSuchBeanDefinitionException` when autowiring by concrete class. Fix by injecting interface.
6. **Order three aspects** using `@Order` and verify the onion execution via print statements.
7. **Use `@DeclareParents`** to add an `Auditable` interface to all service beans; call it via cast.
8. **Convert an `@Before` advice to `@Around`** that skips execution when an argument is invalid, returning a default value.
9. **Measure performance impact** of AOP — call a simple method 1M times with and without an advice.
10. **Write a custom `Pointcut`** using `bean(*Service)` and log only service beans.

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [01_Spring_Framework_Core.md](./01_Spring_Framework_Core.md)
- **Next →:** [03_Spring_MVC.md](./03_Spring_MVC.md)
- **Related:** [01_Spring_Framework_Core.md](./01_Spring_Framework_Core.md), [15_Spring_Security_Method_Level.md](./15_Spring_Security_Method_Level.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
