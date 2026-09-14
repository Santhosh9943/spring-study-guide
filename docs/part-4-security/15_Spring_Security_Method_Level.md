# Method-Level Security

> **File:** `15_Spring_Security_Method_Level.md`
> **Part:** 4 — Security
> **Prerequisites:** `13_Spring_Security_Core.md`, `14_Spring_Security_JWT_OAuth2.md`, `02_Spring_AOP.md`
> **Estimated Study Time:** 5–7 hours

---

## Table of Contents

1. [Why Method-Level Security?](#1-why-method-level-security)
2. [Enabling Method Security](#2-enabling-method-security)
3. [@PreAuthorize](#3-preauthorize)
4. [@PostAuthorize](#4-postauthorize)
5. [@PreFilter & @PostFilter](#5-prefilter--postfilter)
6. [@Secured (Legacy)](#6-secured-legacy)
7. [@RolesAllowed (JSR-250)](#7-rolesallowed-jsr-250)
8. [SpEL in Security Expressions](#8-spel-in-security-expressions)
9. [@AuthenticationPrincipal](#9-authenticationprincipal)
10. [Custom Permission Evaluator](#10-custom-permission-evaluator)
11. [Domain Object Security (ACL)](#11-domain-object-security-acl)
12. [How It Works (AOP)](#12-how-it-works-aop)
13. [Testing Method Security](#13-testing-method-security)
14. [Common Pitfalls](#14-common-pitfalls)
15. [Interview Questions](#15-interview-questions)
16. [Cheat Sheet](#16-cheat-sheet)

---

## 1. Why Method-Level Security?

### URL-Level vs Method-Level

| Aspect | URL-Level | Method-Level |
|--------|-----------|--------------|
| Where | SecurityFilterChain | Annotated methods |
| Granularity | URL patterns | Per method |
| Cross-service | ❌ | ✅ |
| Domain-aware | ❌ | ✅ |
| SpEL | Limited | Full |
| Config location | Centralized | On methods |

### When to Use Method-Level

- Service-layer authorization (URL patterns can't distinguish business rules)
- Cross-cutting concerns on services (not just controllers)
- Fine-grained rules: "owner can update their own order"
- Reusing the same service from multiple controllers
- Domain-driven security

### Example Scenario

```java
// URL-level can't express this:
@PreAuthorize("#order.ownerId == authentication.principal.id")
public Order updateOrder(Long orderId, Order order) { ... }

// URL-level would need custom code in controller — messy
```

---

## 2. Enabling Method Security

### @EnableMethodSecurity (Recommended — Spring Security 6)

```java
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,          // default true — @PreAuthorize, @PostAuthorize
    securedEnabled = false,         // enable @Secured
    jsr250Enabled = false           // enable @RolesAllowed
)
public class MethodSecurityConfig { }
```

Spring Boot auto-configures when `spring-boot-starter-security` is present, but `@EnableMethodSecurity` must be **explicitly** added to enable annotations.

### Legacy: @EnableGlobalMethodSecurity

```java
@Configuration
@EnableGlobalMethodSecurity(
    prePostEnabled = true,
    securedEnabled = true,
    jsr250Enabled = true
)
public class MethodSecurityConfig { }
```

Deprecated since Spring Security 5.6; replaced by `@EnableMethodSecurity`.

### How It Works — AOP Proxy

Method security uses **Spring AOP** — a proxy wraps beans and intercepts method calls. Same proxy mechanisms as `@Transactional` (JDK or CGLIB).

### Configuration Attributes

| Attribute | Default | Effect |
|-----------|---------|--------|
| `prePostEnabled` | `true` | Enables `@PreAuthorize`, `@PostAuthorize`, `@PreFilter`, `@PostFilter` |
| `securedEnabled` | `false` | Enables `@Secured` |
| `jsr250Enabled` | `false` | Enables `@RolesAllowed`, `@PermitAll`, `@DenyAll` |
| `mode` | `PROXY` | `PROXY` (AOP) or `ASPECTJ` (weaving) |
| `proxyTargetClass` | `false` | Use CGLIB for all proxies |

---

## 3. @PreAuthorize

Evaluated **before** the method executes. Can use SpEL to access method arguments, `authentication`, `principal`, `#root`, etc.

### Basic Usage

```java
@Service
public class UserService {
    
    @PreAuthorize("hasRole('ADMIN')")
    public List<User> findAll() { ... }
    
    @PreAuthorize("hasAuthority('USER_READ')")
    public User findById(Long id) { ... }
    
    @PreAuthorize("isAuthenticated()")
    public User getCurrentUser() { ... }
}
```

### Accessing Method Arguments

```java
@PreAuthorize("#id == authentication.principal.id or hasRole('ADMIN')")
public User findById(Long id) { ... }

@PreAuthorize("#user.id == authentication.principal.id")
public void updateUser(User user) { ... }

@PreAuthorize("#username == authentication.name")
public Profile getProfile(String username) { ... }
```

### Return-Based Rules (on method)

```java
@PreAuthorize("#order.ownerId == authentication.principal.id")
public void updateOrder(Order order) { ... }
```

### Common Expressions

| Expression | Meaning |
|------------|---------|
| `isAuthenticated()` | Any authenticated user |
| `isAnonymous()` | Anonymous user |
| `isRememberMe()` | Remember-me authenticated |
| `isFullyAuthenticated()` | Not remember-me |
| `hasRole('ADMIN')` | Has `ROLE_ADMIN` |
| `hasAnyRole('ADMIN', 'USER')` | Has any |
| `hasAuthority('READ_USERS')` | Has exact authority |
| `hasAnyAuthority('A', 'B')` | Has any authority |
| `permitAll` | Always true |
| `denyAll` | Always false |
| `principal.username == #username` | Custom check |

### Class-Level @PreAuthorize

```java
@Service
@PreAuthorize("hasRole('ADMIN')")   // applies to all methods
public class AdminService {
    
    public List<User> findAll() { ... }    // needs ADMIN
    
    @PreAuthorize("hasRole('SUPER_ADMIN')")   // overrides class-level
    public void deleteAll() { ... }
}
```

Method-level overrides class-level.

### Accessing Beans

```java
@PreAuthorize("@permissionService.canAccess(#userId)")
public User getUser(Long userId) { ... }
```

`@permissionService` references the `permissionService` bean.

### Combining Conditions

```java
@PreAuthorize("hasRole('ADMIN') or (#userId == authentication.principal.id)")
public User getUser(Long userId) { ... }

@PreAuthorize("isAuthenticated() and !isAnonymous()")
public List<Order> myOrders() { ... }
```

### PathVariable Example

```java
@RestController
public class OrderController {
    
    @PreAuthorize("#id == authentication.principal.id or hasRole('ADMIN')")
    @GetMapping("/users/{id}/orders")
    public List<Order> orders(@PathVariable Long id) { ... }
}
```

---

## 4. @PostAuthorize

Evaluated **after** the method returns. Can access the return value via `returnObject`.

### Basic

```java
@PostAuthorize("returnObject.owner == authentication.name")
public Document getDocument(Long id) { ... }
```

Method runs, then result is checked. If denied → 403 (but expensive work already done).

### Why @PostAuthorize?

When the rule depends on the **returned data**, not just args:

```java
@PostAuthorize("returnObject.ownerId == authentication.principal.id")
public Order findOrder(Long orderId) {
    // fetch and return — security check after
    return orderRepo.findById(orderId).orElseThrow();
}
```

**Downside:** method body executes regardless. Use for cheap operations, or when return value matters.

### @PostAuthorize with Collections

```java
@PostAuthorize("returnObject.ownerId == authentication.principal.id")
public List<Order> findMyOrders() { ... }
```

For collections, use `@PostFilter` (see next section).

### Combining @PreAuthorize and @PostAuthorize

Not usually needed. Use one.

### Performance Note

`@PostAuthorize` doesn't short-circuit — the method runs even for unauthorized users. Use `@PreAuthorize` when possible.

---

## 5. @PreFilter & @PostFilter

Filter collections based on security rules.

### @PreFilter

Filters method **arguments** before invocation. Only the user's own items are passed.

```java
@PreFilter("filterObject.ownerId == authentication.principal.id")
public void saveAll(List<Order> orders) { ... }
```

`filterObject` = the current element.

### @PostFilter

Filters the **returned collection** — only elements the user can access are kept.

```java
@PostFilter("filterObject.ownerId == authentication.principal.id")
public List<Order> findAll() {
    return orderRepo.findAll();   // returns only user's orders
}
```

### List, Set, Map Support

```java
@PostFilter("filterObject.value.ownerId == authentication.name")
public Map<Long, Order> getAll() { ... }
```

For Map, `filterObject.value` is the map entry value.

### Common with @PostAuthorize

```java
@PostAuthorize("hasRole('ADMIN') or returnObject.ownerId == authentication.name")
public Order find(Long id) { ... }

@PostFilter("hasRole('ADMIN') or filterObject.ownerId == authentication.name")
public List<Order> findAll() { ... }
```

### Performance Warning

`@PreFilter` and `@PostFilter` operate on the **whole collection in memory**. For large datasets, prefer a **custom query** that filters at the DB level, or use a `PermissionEvaluator`.

### Example

```java
@Service
public class DocumentService {
    
    @PostFilter("hasPermission(filterObject, 'READ')")
    public List<Document> search(String keyword) {
        return documentRepo.findByTitleContaining(keyword);
    }
}
```

Uses custom `PermissionEvaluator` for `hasPermission(...)`.

---

## 6. @Secured (Legacy)

Simple role check without SpEL. Enable via `securedEnabled = true`.

```java
@Configuration
@EnableMethodSecurity(securedEnabled = true)
public class SecurityConfig { }
```

### Usage

```java
@Secured("ROLE_ADMIN")
public void deleteAll() { ... }

@Secured({"ROLE_ADMIN", "ROLE_MANAGER"})
public void deleteById(Long id) { ... }
```

### Rules

- Requires `ROLE_` prefix explicitly
- No SpEL (can't access args or return value)
- Deprecated in favor of `@PreAuthorize`

### @Secured vs @PreAuthorize

| Feature | @Secured | @PreAuthorize |
|---------|---------|---------------|
| Roles only | ✅ | ✅ |
| Authorities | ❌ (needs ROLE_) | ✅ |
| SpEL | ❌ | ✅ |
| Arg access | ❌ | ✅ |
| Return value | ❌ | ✅ (post) |
| Recommended | ❌ | ✅ |

**Rule:** Use `@PreAuthorize` instead of `@Secured` in new code.

---

## 7. @RolesAllowed (JSR-250)

JSR-250 standard annotation. Enable via `jsr250Enabled = true`.

```java
@Configuration
@EnableMethodSecurity(jsr250Enabled = true)
public class SecurityConfig { }
```

### Usage

```java
@RolesAllowed("ADMIN")
public void deleteAll() { ... }

@RolesAllowed({"ADMIN", "MANAGER"})
public void update(Long id) { ... }

@PermitAll
public User getPublicInfo() { ... }

@DenyAll
public void dangerousOperation() { ... }
```

### @RolesAllowed vs @Secured

- Both role-based, no SpEL
- `@RolesAllowed` is standard (JSR-250)
- `@Secured` is Spring-specific
- Both add `ROLE_` prefix automatically (unlike `@Secured` — which requires it explicitly)

**Rule:** Prefer `@PreAuthorize`. Use JSR-250 only for standards compliance (e.g., shared code across frameworks).

### Annotations Summary

| Annotation | Type | Roles prefix auto |
|------------|------|-------------------|
| `@Secured("ROLE_ADMIN")` | Spring | ❌ (need `ROLE_`) |
| `@RolesAllowed("ADMIN")` | JSR-250 | ✅ |
| `@PreAuthorize("hasRole('ADMIN')")` | Spring | ✅ |

---

## 8. SpEL in Security Expressions

### Available Objects

| Variable | Type |
|----------|------|
| `authentication` | `Authentication` |
| `principal` | `authentication.getPrincipal()` |
| `#argName` / `#a0` / `#p0` | Method args |
| `#root` | `MethodSecurityExpressionRoot` |
| `returnObject` | (post) Return value |
| `filterObject` | (filter) Current element |
| `@beanName` | Any Spring bean |

### Common Methods

```
hasRole('X')                    hasAuthority('Y')
hasAnyRole('X','Y')             hasAnyAuthority('X','Y')
isAuthenticated()               isAnonymous()
isRememberMe()                  isFullyAuthenticated()
permitAll                       denyAll
hasPermission(target, perm)     hasPermission(targetId, type, perm)
```

### Role vs Authority

- `hasRole("ADMIN")` → checks for `ROLE_ADMIN`
- `hasAuthority("ROLE_ADMIN")` → checks for exact string

⚠️ Do not mix — either store `ROLE_ADMIN` and use `hasRole("ADMIN")`, or store `ADMIN` and use `hasAuthority("ADMIN")`.

### Accessing the Principal

```java
@PreAuthorize("#principal.username == #username")
public Profile getProfile(String username) { ... }

// With custom UserDetails
@PreAuthorize("#id == principal.id")
public User findById(Long id) { ... }
```

### Bean Reference

```java
@PreAuthorize("@securityService.isOwner(#orderId, authentication)")
public void cancel(Long orderId) { ... }
```

### Nested Properties

```java
@PreAuthorize("#request.owner == authentication.name")
public void process(Request request) { ... }
```

### Method Calls

```java
@PreAuthorize("#userService.isActive(#id)")
public User getActiveUser(Long id) { ... }
```

### Complex Expressions

```java
@PreAuthorize("hasRole('ADMIN') or " +
              "(hasRole('MANAGER') and #amount < 1000) or " +
              "authentication.name == #customerName")
public void approve(String customerName, BigDecimal amount) { ... }
```

### Type References

```java
@PreAuthorize("@permissionService.hasRole(authentication, T(com.example.Role).ADMIN)")
```

### Boolean Operators

```
and / or / not    → and && || !
```

Both forms work; `and/or/not` preferred in annotations for readability.

---

## 9. @AuthenticationPrincipal

Injects the authenticated principal (or a custom `UserDetails`) into controller/service method parameters.

### Basic

```java
@GetMapping("/me")
public UserDto me(@AuthenticationPrincipal UserDetails user) {
    return new UserDto(user.getUsername());
}
```

### Custom UserDetails

```java
public class AppUser implements UserDetails {
    private final Long id;
    private final String username;
    // ...
    public Long getId() { return id; }
}

@GetMapping("/me")
public UserDto me(@AuthenticationPrincipal AppUser user) {
    return new UserDto(user.getId(), user.getUsername());
}
```

### Handling Anonymous Users

`@AuthenticationPrincipal` may be null for anonymous. Use `errorOnInvalidType` or check:

```java
@GetMapping("/profile")
public Profile profile(@AuthenticationPrincipal AppUser user) {
    if (user == null) throw new UnauthorizedException();
    return profileService.find(user.getId());
}
```

Or provide a fallback:

```java
@GetMapping("/profile")
public Profile profile(
    @AuthenticationPrincipal(errorOnInvalidType = false) AppUser user) { ... }
```

### Expression Support (Spring Security 5.2+)

```java
@GetMapping("/id")
public Long myId(@AuthenticationPrincipal(expression = "id") Long id) { ... }
```

SpEL on the principal.

### Common with JWT

```java
@GetMapping("/claims")
public Map<String, Object> claims(@AuthenticationPrincipal Jwt jwt) {
    return jwt.getClaims();
}
```

With OAuth2 Resource Server, the principal is `Jwt` (not `UserDetails`).

### Injection Alternatives

| Mechanism | Type |
|-----------|------|
| `@AuthenticationPrincipal` | Custom principal/UserDetails |
| `Principal` | `java.security.Principal` (just name) |
| `Authentication` | Spring Security `Authentication` |
| `@CurrentSecurityContext` | Full security context |

---

## 10. Custom Permission Evaluator

For domain-object rules like `hasPermission(order, 'WRITE')`.

### PermissionEvaluator Interface

```java
public interface PermissionEvaluator {
    boolean hasPermission(Authentication auth, Object target, Object permission);
    boolean hasPermission(Authentication auth, Serializable targetId, 
                          String targetType, Object permission);
}
```

### Implementation

```java
@Component
public class DomainPermissionEvaluator implements PermissionEvaluator {
    
    private final OrderRepository orderRepo;
    
    public DomainPermissionEvaluator(OrderRepository orderRepo) {
        this.orderRepo = orderRepo;
    }
    
    @Override
    public boolean hasPermission(Authentication auth, Object target, Object permission) {
        if (target instanceof Order order) {
            return switch (permission.toString()) {
                case "READ" -> order.getOwnerId().equals(principalId(auth))
                    || hasRole(auth, "ADMIN");
                case "WRITE", "DELETE" -> order.getOwnerId().equals(principalId(auth));
                default -> false;
            };
        }
        return false;
    }
    
    @Override
    public boolean hasPermission(Authentication auth, Serializable targetId,
                                 String targetType, Object permission) {
        if ("Order".equals(targetType)) {
            return orderRepo.findById((Long) targetId)
                .map(o -> hasPermission(auth, o, permission))
                .orElse(false);
        }
        return false;
    }
    
    private Long principalId(Authentication auth) {
        return ((AppUser) auth.getPrincipal()).getId();
    }
    
    private boolean hasRole(Authentication auth, String role) {
        return auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_" + role));
    }
}
```

### Register with MethodSecurityExpressionHandler

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
    
    @Bean
    public MethodSecurityExpressionHandler expressionHandler(
            PermissionEvaluator permissionEvaluator) {
        DefaultMethodSecurityExpressionHandler handler = 
            new DefaultMethodSecurityExpressionHandler();
        handler.setPermissionEvaluator(permissionEvaluator);
        return handler;
    }
}
```

### Usage

```java
@PreAuthorize("hasPermission(#order, 'WRITE')")
public Order update(Order order) { ... }

@PreAuthorize("hasPermission(#orderId, 'Order', 'DELETE')")
public void delete(Long orderId) { ... }

@PostFilter("hasPermission(filterObject, 'READ')")
public List<Order> findAll() { ... }
```

### Testing

```java
@Test
@WithMockUser
void userCanReadOwnOrder() {
    // mock evaluator returns true
}
```

---

## 11. Domain Object Security (ACL)

Spring Security provides a full **ACL** system for per-instance permissions.

### Concepts

- **Object Identity** (`ObjectIdentity`) — a domain object (class + id)
- **SID** (Security Identity) — user or authority
- **ACL** — list of `AccessControlEntry` for an object
- **Permission** — bit mask (READ, WRITE, DELETE, ADMINISTRATION, ...)

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-acl</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
</dependency>
```

### ACL Schema

Spring provides SQL scripts to create ACL tables:

```
acl_sid
acl_class
acl_object_identity
acl_entry
```

### Configuration

```java
@Configuration
@EnableMethodSecurity
public class AclConfig {
    
    @Bean
    public AclService aclService(DataSource ds) {
        return new JdbcMutableAclService(
            ds, new LookupStrategyImpl(...), new BasicAclEntryCache());
    }
    
    @Bean
    public PermissionEvaluator aclPermissionEvaluator(AclService aclService) {
        return new AclPermissionEvaluator(aclService);
    }
    
    @Bean
    public MethodSecurityExpressionHandler expressionHandler(
            PermissionEvaluator aclPermissionEvaluator) {
        DefaultMethodSecurityExpressionHandler h = 
            new DefaultMethodSecurityExpressionHandler();
        h.setPermissionEvaluator(aclPermissionEvaluator);
        return h;
    }
}
```

### Adding Permissions

```java
public void grantAccess(Long orderId, String username, Permission permission) {
    ObjectIdentity oid = new ObjectIdentityImpl(Order.class, orderId);
    Sid sid = new PrincipalSid(username);
    MutableAcl acl = aclService.createAcl(oid);
    acl.insertAce(acl.getEntries().size(), permission, sid, true);
    aclService.updateAcl(acl);
}
```

### Usage

```java
@PreAuthorize("hasPermission(#orderId, 'com.example.Order', 'READ')")
public Order getOrder(Long orderId) { ... }
```

### When to Use ACL

- Fine-grained per-instance permissions (documents, projects)
- Many users × many objects
- Permissions change frequently

### When NOT to Use

- Simple "owner or admin" rules — use SpEL
- Few users or objects — use simpler checks
- ACL adds complexity and DB tables

---

## 12. How It Works (AOP)

Method security uses Spring AOP:

```
Caller → Proxy → MethodSecurityInterceptor
                    ├── PreAuthorizeAdvice (before)
                    ├── Invoke target method
                    ├── PostAuthorizeAdvice (after)
                    └── Return / throw AccessDeniedException
```

### Key Classes

| Class | Role |
|-------|------|
| `AuthorizationManagerBeforeMethodInterceptor` | Handles `@PreAuthorize`, `@Secured`, `@RolesAllowed` |
| `AuthorizationManagerAfterMethodInterceptor` | Handles `@PostAuthorize` |
| `PreFilterAuthorizationMethodInterceptor` | Handles `@PreFilter` |
| `PostFilterAuthorizationMethodInterceptor` | Handles `@PostFilter` |
| `MethodSecurityExpressionHandler` | Evaluates SpEL |

### Proxy Mode vs AspectJ Mode

| Mode | Proxying | Supports self-invocation | Enables |
|------|----------|-------------------------|---------|
| `PROXY` (default) | JDK/CGLIB | ❌ | Normal beans |
| `ASPECTJ` | AspectJ weaving | ✅ | Any code |

Enable AspectJ mode:

```java
@EnableMethodSecurity(mode = AdviceMode.ASPECTJ)
```

Requires AspectJ load-time weaving.

### Self-Invocation Problem

Same as `@Transactional` and `@Cacheable`:

```java
@Service
public class OrderService {
    
    public void outer() {
        this.inner();    // ❌ no security applied
    }
    
    @PreAuthorize("hasRole('ADMIN')")
    public void inner() { ... }
}
```

**Fixes:** extract to another bean, inject self, use AspectJ mode.

---

## 13. Testing Method Security

### @WithMockUser

```java
@Test
@WithMockUser(roles = "ADMIN")
void adminCanDelete() {
    service.deleteAll();   // succeeds
}

@Test
@WithMockUser(roles = "USER")
void userCannotDelete() {
    assertThatThrownBy(() -> service.deleteAll())
        .isInstanceOf(AccessDeniedException.class);
}
```

### @WithUserDetails

Loads the actual `UserDetails` from `UserDetailsService`.

```java
@Test
@WithUserDetails("alice")
void aliceCanDoX() { ... }
```

### @WithAnonymousUser

```java
@Test
@WithAnonymousUser
void anonymousDenied() { ... }
```

### @WithSecurityContext (Custom)

```java
@Retention(RUNTIME)
@WithSecurityContext(factory = WithMockAppUserFactory.class)
public @interface WithMockAppUser {
    long id() default 1L;
    String username() default "test";
    String[] roles() default {"USER"};
}

public class WithMockAppUserFactory 
        implements WithSecurityContextFactory<WithMockAppUser> {
    @Override
    public SecurityContext createSecurityContext(WithMockAppUser ann) {
        AppUser user = new AppUser(ann.id(), ann.username(), ann.roles());
        Authentication auth = new UsernamePasswordAuthenticationToken(
            user, "password", user.getAuthorities());
        SecurityContext ctx = SecurityContextHolder.createEmptyContext();
        ctx.setAuthentication(auth);
        return ctx;
    }
}
```

### @SpringBootTest Integration

```java
@SpringBootTest
class OrderServiceSecuredTest {
    
    @Autowired OrderService service;
    
    @Test
    @WithMockUser(username = "alice", roles = "USER")
    void userCannotAccessOtherUserOrder() {
        // assume order 42 belongs to bob
        assertThatThrownBy(() -> service.getOrder(42L))
            .isInstanceOf(AccessDeniedException.class);
    }
    
    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanAccessAnyOrder() {
        Order o = service.getOrder(42L);
        assertThat(o).isNotNull();
    }
}
```

### With JWT (Resource Server)

```java
@Test
void jwtAccess() throws Exception {
    mvc.perform(get("/api/admin")
            .with(jwt().jwt(j -> j.subject("alice")
                .claim("roles", List.of("ADMIN")))))
        .andExpect(status().isOk());
}
```

---

## 14. Common Pitfalls

### 14.1 Self-Invocation

Same as AOP — internal calls bypass the proxy. Fix with self-injection, extract bean, or AspectJ mode.

### 14.2 Private / Protected Methods

Only public methods are advised. Annotations on private methods are **silently ignored**.

### 14.3 Final Methods / Classes

CGLIB can't override final → advice skipped silently.

### 14.4 Missing @EnableMethodSecurity

Annotation present but `@EnableMethodSecurity` not added → security not enforced. **Silent failure**.

### 14.5 Roles vs Authorities Mismatch

`hasRole("ADMIN")` requires authority `ROLE_ADMIN`, not `ADMIN`. Mixing fails silently.

### 14.6 Class-Level vs Method-Level

Method-level **overrides** class-level. If you want both, use `and`:

```java
@Service
@PreAuthorize("isAuthenticated()")   // class
public class Service {
    @PreAuthorize("hasRole('ADMIN')")    // method — replaces class!
    public void delete() { ... }
}
```

### 14.7 @PostAuthorize Runs Regardless

Method body executes even if access denied. Prefer `@PreAuthorize` for expensive operations.

### 14.8 @PostFilter on Large Collections

Filters in-memory. For huge result sets, use DB filtering.

### 14.9 @PreAuthorize on Interfaces

Same as `@Transactional` — Spring recommends annotating concrete classes.

### 14.10 Expression Errors

A typo in SpEL throws at runtime (not compile). Test thoroughly.

### 14.11 Nested Method Security

Nested service calls each evaluate their own `@PreAuthorize`. No context propagation needed — the `Authentication` is in `SecurityContextHolder`.

### Pitfalls Checklist

- [ ] `@EnableMethodSecurity` present
- [ ] Public methods
- [ ] No self-invocation
- [ ] Not final class/method
- [ ] Roles prefixed correctly
- [ ] `@PreAuthorize` used over `@Secured`
- [ ] Class vs method precedence understood
- [ ] Tested with different roles

---

## 15. Interview Questions

### Q1. What is method-level security?

**Answer:** Security enforced on service/component methods via annotations (`@PreAuthorize`, `@PostAuthorize`, `@Secured`, `@RolesAllowed`). Uses AOP proxies. Provides fine-grained control that URL-level security can't express.

### Q2. How do you enable method security?

**Answer:** Add `@EnableMethodSecurity` to a `@Configuration` class. Options: `prePostEnabled` (default true), `securedEnabled`, `jsr250Enabled`. Spring Boot does not enable it by default — you must add the annotation.

### Q3. Difference between @PreAuthorize and @PostAuthorize?

**Answer:** `@PreAuthorize` runs **before** the method — checks args/auth. `@PostAuthorize` runs **after** — checks `returnObject`. Use pre for args; post for return-value rules. Pre short-circuits; post doesn't.

### Q4. What is the difference between @PreFilter and @PostFilter?

**Answer:** `@PreFilter` filters method **arguments** (collections) before invocation — only accessible items passed. `@PostFilter` filters the **returned** collection — only accessible items returned.

### Q5. When to use @Secured vs @PreAuthorize?

**Answer:** `@PreAuthorize` supports SpEL, arg/return access, and full expressions. `@Secured` is a simple role check with `ROLE_` prefix and no SpEL. Prefer `@PreAuthorize`.

### Q6. What are JSR-250 annotations?

**Answer:** `@RolesAllowed`, `@PermitAll`, `@DenyAll`. Standard Java security annotations. Enable with `jsr250Enabled = true`. Role names auto-prefixed with `ROLE_`.

### Q7. How do you access method arguments in @PreAuthorize?

**Answer:** Via SpEL: `#argName` (with `-parameters` compile flag), `#a0`/`#p0` for positional. Example: `@PreAuthorize("#id == principal.id")`.

### Q8. How do you access the authenticated user?

**Answer:** `authentication`, `principal`, or `@AuthenticationPrincipal` in controller methods. `authentication.name`, `authentication.principal.id`, etc.

### Q9. What is @AuthenticationPrincipal?

**Answer:** Injects the principal from `SecurityContext` into a method parameter. Custom `UserDetails` or `Jwt` (for OAuth2 resource server). Supports SpEL via `expression` attribute.

### Q10. How do you implement custom permission checks?

**Answer:** Implement `PermissionEvaluator` with `hasPermission(auth, target, perm)`. Register it with the `MethodSecurityExpressionHandler`. Use in expressions: `@PreAuthorize("hasPermission(#order, 'WRITE')")`.

### Q11. What is Spring ACL?

**Answer:** A domain-object security system for per-instance permissions. Stores ACLs in DB tables (`acl_sid`, `acl_class`, `acl_object_identity`, `acl_entry`). Uses `AclPermissionEvaluator` with `hasPermission(...)`. Powerful but complex.

### Q12. How does method security work internally?

**Answer:** Via Spring AOP — proxy wraps the bean. `AuthorizationManagerBeforeMethodInterceptor` evaluates `@PreAuthorize` before invocation. `AfterMethodInterceptor` handles `@PostAuthorize`. `Filter` interceptors handle `@PreFilter`/`@PostFilter`.

### Q13. What is self-invocation problem?

**Answer:** Calling a secured method from within the same bean (`this.method()`) bypasses the proxy → security not applied. Fix: extract to another bean, inject self, or use `mode = ASPECTJ` (weaving).

### Q14. Can you annotate interfaces?

**Answer:** Yes, but not recommended. Spring docs advise annotating concrete classes. With JDK proxies, interface annotations work; with CGLIB, they may not propagate predictably.

### Q15. How do you test method security?

**Answer:** Use `@WithMockUser`, `@WithUserDetails`, `@WithAnonymousUser`, or `.with(jwt()...)` for JWT. Assert `AccessDeniedException` for denied cases.

### Q16. What happens if @EnableMethodSecurity is missing?

**Answer:** Annotations are **silently ignored**. No security is enforced. This is a common and dangerous mistake — always verify with tests.

### Q17. What exceptions are thrown on failure?

**Answer:** `AccessDeniedException` (`org.springframework.security.access.AccessDeniedException`) for authenticated-but-denied. `AuthenticationCredentialsNotFoundException` for anonymous when required.

### Q18. Can you combine URL and method security?

**Answer:** Yes — URL rules run first (at the filter level), then method rules (at the AOP level). Both must pass.

### Q19. What is the difference between hasRole and hasAuthority?

**Answer:** `hasRole("ADMIN")` checks for `ROLE_ADMIN`. `hasAuthority("ROLE_ADMIN")` checks exact. Roles are authorities with `ROLE_` prefix.

### Q20. What is @PreAuthorize's `#root`?

**Answer:** `MethodSecurityExpressionRoot` — access method metadata: `#root.method`, `#root.target`, `#root.args`, `#root.methodName`. Useful for generic rules.

### Q21. How do you conditionally apply method security?

**Answer:** Use `@PreAuthorize` with `@ConditionalOnExpression` on beans, or guard the service method body with `if (permission) { ... }`. There's no built-in way to skip the annotation per-request.

### Q22. What's the performance impact of method security?

**Answer:** A proxy call + SpEL evaluation per invocation. For most apps, negligible. For hot paths, use simple checks (`@Secured`) or cache decisions.

### Q23. What is @PostFilter's `filterObject`?

**Answer:** The current element of the returned collection. In `@PostFilter("filterObject.owner == authentication.name")`, `filterObject` is each item.

### Q24. Can @PreAuthorize call other beans?

**Answer:** Yes — via `@beanName` syntax: `@PreAuthorize("@permissionService.check(#id)")`. Useful for complex rule logic.

### Q25. What modes does @EnableMethodSecurity support?

**Answer:** `PROXY` (default, uses AOP) and `ASPECTJ` (compile/load-time weaving, supports self-invocation). `ASPECTJ` requires AspectJ setup.

---

## 16. Cheat Sheet

### Enable

```java
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,     // @PreAuthorize, @PostAuthorize, @Pre/PostFilter
    securedEnabled = true,     // @Secured
    jsr250Enabled = true       // @RolesAllowed, @PermitAll, @DenyAll
)
public class SecurityConfig { }
```

### Annotations

```java
@PreAuthorize("hasRole('ADMIN')")
@PostAuthorize("returnObject.owner == authentication.name")
@PreFilter("filterObject.owner == authentication.name")
@PostFilter("hasRole('ADMIN') or filterObject.owner == authentication.name")
@Secured({"ROLE_ADMIN", "ROLE_USER"})
@RolesAllowed({"ADMIN", "USER"})
@PermitAll
@DenyAll
@AuthenticationPrincipal AppUser user
```

### Common Expressions

```
isAuthenticated()               isAnonymous()
hasRole('X')                    hasAnyRole('X','Y')
hasAuthority('Y')               hasAnyAuthority('X','Y')
#argName == principal.name       returnObject.owner == #user
hasPermission(#obj, 'WRITE')    @bean.check(#id)
```

### Custom PermissionEvaluator

```java
@Component
public class DomainPermissionEvaluator implements PermissionEvaluator {
    public boolean hasPermission(Authentication auth, Object target, Object perm) { ... }
    public boolean hasPermission(Authentication auth, Serializable id, 
                                 String type, Object perm) { ... }
}

@Bean
public MethodSecurityExpressionHandler handler(PermissionEvaluator pe) {
    DefaultMethodSecurityExpressionHandler h = 
        new DefaultMethodSecurityExpressionHandler();
    h.setPermissionEvaluator(pe);
    return h;
}
```

### Testing

```java
@WithMockUser(roles = "ADMIN")
@WithMockUser(username = "alice", authorities = {"ROLE_USER", "READ"})
@WithUserDetails("alice")
@WithAnonymousUser
.with(jwt().jwt(j -> j.subject("alice").claim("roles", List.of("USER"))))
```

### Pitfalls

- Missing `@EnableMethodSecurity` → silent no-op
- Self-invocation → no proxy
- Private methods → not advised
- Roles vs authorities mismatch
- `@Secured` needs `ROLE_` prefix explicitly
- Method-level overrides class-level

### AOP Classes

```
AuthorizationManagerBeforeMethodInterceptor
AuthorizationManagerAfterMethodInterceptor
PreFilterAuthorizationMethodInterceptor
PostFilterAuthorizationMethodInterceptor
```

### Cross-References

- **Previous:** `14_Spring_Security_JWT_OAuth2.md`
- **Next:** `16_Spring_Testing.md`
- **Related:** `02_Spring_AOP.md`, `13_Spring_Security_Core.md`
- **Interview:** `24_Spring_Interview_Questions.md` (§Security)

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [14_Spring_Security_JWT_OAuth2.md](./14_Spring_Security_JWT_OAuth2.md)
- **Next →:** [16_Spring_Testing.md](./16_Spring_Testing.md)
- **Related:** [13_Spring_Security_Core.md](./13_Spring_Security_Core.md), [02_Spring_AOP.md](./02_Spring_AOP.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
