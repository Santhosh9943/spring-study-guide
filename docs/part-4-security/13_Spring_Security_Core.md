# Spring Security Core

> **File:** `13_Spring_Security_Core.md`
> **Part:** 4 — Security
> **Prerequisites:** `01_Spring_Framework_Core.md`, `03_Spring_MVC.md`, `06_Spring_Boot_Fundamentals.md`
> **Estimated Study Time:** 10–14 hours

---

## Table of Contents

1. [Security Concepts](#1-security-concepts)
2. [Spring Security Architecture](#2-spring-security-architecture)
3. [SecurityFilterChain (Spring Security 6+)](#3-securityfilterchain-spring-security-6)
4. [AuthenticationManager & ProviderManager](#4-authenticationmanager--providermanager)
5. [UserDetails & UserDetailsService](#5-userdetails--userdetailsservice)
6. [PasswordEncoder](#6-passwordencoder)
7. [In-Memory Authentication](#7-in-memory-authentication)
8. [JDBC Authentication](#8-jdbc-authentication)
9. [Form Login](#9-form-login)
10. [HTTP Basic](#10-http-basic)
11. [Session Management](#11-session-management)
12. [CSRF Protection](#12-csrf-protection)
13. [CORS](#13-cors)
14. [Logout](#14-logout)
15. [Remember-Me Authentication](#15-remember-me-authentication)
16. [Security Headers](#16-security-headers)
17. [Interview Questions](#17-interview-questions)
18. [Cheat Sheet](#18-cheat-sheet)

---

## 1. Security Concepts

### Core Terms

| Term | Meaning |
|------|---------|
| **Authentication** | Who are you? (identity verification) |
| **Authorization** | What can you do? (access control) |
| **Principal** | The authenticated user |
| **Credential** | Proof of identity (password, token) |
| **Authority** | A permission (`ROLE_ADMIN`, `READ_USERS`) |
| **Role** | An authority with `ROLE_` prefix (`ROLE_ADMIN`) |
| **Subject** | Security context holder (Java Security) |

### Authentication vs Authorization

```
Authentication         Authorization
─────────────         ─────────────
"Who are you?"        "Can you do this?"
Login → Principal    Check permissions
Success / Failure     Allow / Deny
```

### Authentication Flow

```
1. User provides credentials (username/password, token)
2. Security filter extracts them
3. AuthenticationManager authenticates
4. ProviderManager delegates to providers
5. Provider validates (DB, LDAP, etc.)
6. SecurityContext set with Authentication
7. AccessDecisionManager authorizes
8. Request proceeds or 401/403
```

### Authorization Levels

| Level | Where |
|-------|-------|
| **Request-level** | URL patterns (`/admin/**`) |
| **Method-level** | `@PreAuthorize`, `@Secured` |
| **Domain-level** | ACL, custom checks |

### Common Status Codes

- **401 Unauthorized** — not authenticated
- **403 Forbidden** — authenticated but no permission
- **419 / 440** — session expired (framework-specific)
- **302 / 200** — redirect to login page

---

## 2. Spring Security Architecture

### Filter Chain (Servlet)

Every request passes through a chain of filters before reaching your controllers.

```
Client Request
    │
    ▼
┌─────────────────────────────────────────────────┐
│         SecurityFilterChain                     │
│  ┌─────────────────────────────────────────┐   │
│  │ 1. SecurityContextHolderFilter          │   │  ← loads SecurityContext
│  ├─────────────────────────────────────────┤   │
│  │ 2. HeaderWriterFilter                   │   │
│  ├─────────────────────────────────────────┤   │
│  │ 3. CorsFilter                           │   │
│  ├─────────────────────────────────────────┤   │
│  │ 4. CsrfFilter                           │   │
│  ├─────────────────────────────────────────┤   │
│  │ 5. LogoutFilter                         │   │
│  ├─────────────────────────────────────────┤   │
│  │ 6. UsernamePasswordAuthenticationFilter │   │  ← form login
│  ├─────────────────────────────────────────┤   │
│  │ 7. DefaultLoginPageGeneratingFilter    │   │
│  ├─────────────────────────────────────────┤   │
│  │ 8. BasicAuthenticationFilter            │   │  ← HTTP Basic
│  ├─────────────────────────────────────────┤   │
│  │ 9. BearerTokenAuthenticationFilter      │   │  ← JWT/OAuth2
│  ├─────────────────────────────────────────┤   │
│  │ 10. RequestCacheAwareFilter             │   │
│  ├─────────────────────────────────────────┤   │
│  │ 11. SecurityContextHolderAwareRequestFilter │
│  ├─────────────────────────────────────────┤   │
│  │ 12. AnonymousAuthenticationFilter       │   │
│  ├─────────────────────────────────────────┤   │
│  │ 13. SessionManagementFilter             │   │
│  ├─────────────────────────────────────────┤   │
│  │ 14. ExceptionTranslationFilter          │   │  ← 401/403 handling
│  ├─────────────────────────────────────────┤   │
│  │ 15. AuthorizationFilter                 │   │  ← access decisions
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
    │
    ▼
DispatcherServlet → Controllers
```

### Key Components

| Component | Purpose |
|-----------|---------|
| `SecurityFilterChain` | Ordered chain of filters |
| `FilterChainProxy` | Dispatches to `SecurityFilterChain`s |
| `DelegatingFilterProxy` | Bridges Servlet filter to Spring bean |
| `AuthenticationManager` | Authenticates credentials |
| `ProviderManager` | Manages `AuthenticationProvider`s |
| `AuthenticationProvider` | Verifies credentials |
| `UserDetailsService` | Loads user details |
| `SecurityContextHolder` | Holds the authenticated principal |
| `AccessDecisionManager` | (Legacy) authorizes |
| `AuthorizationManager` | (Modern) authorizes |
| `AuthenticationEntryPoint` | Handles 401 (unauthenticated) |
| `AccessDeniedHandler` | Handles 403 (authenticated but denied) |

### SecurityContextHolder

Where the current user is stored.

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String name = auth.getName();
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
boolean isAdmin = auth.getAuthorities().stream()
    .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
```

### Storage Strategies

| Strategy | Description |
|----------|-------------|
| `MODE_THREADLOCAL` (default) | Per thread |
| `MODE_INHERITABLETHREADLOCAL` | Inheritable by child threads |
| `MODE_GLOBAL` | JVM-wide (rare) |

For reactive: **Reactor Context**.

---

## 3. SecurityFilterChain (Spring Security 6+)

Modern config uses a `SecurityFilterChain` bean (replaced `WebSecurityConfigurerAdapter` in 5.7, removed in 6.0).

### Minimal Config

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/login").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").authenticated()
                .anyRequest().denyAll()
            )
            .formLogin(Customizer.withDefaults())
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

### RequestMatchers Options

```java
.requestMatchers("/api/**")                    // Ant-style path
.requestMatchers(HttpMethod.GET, "/api/**")    // by method
.requestMatchers(new AntPathRequestMatcher("/**"))  // explicit
.requestMatchers("/api/{id}")                  // MVC pattern (6.0+)
```

### Authorization Rules

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/public/**").permitAll()
    .requestMatchers("/api/**").authenticated()
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/report/**").hasAuthority("REPORT_VIEW")
    .requestMatchers("/org/**").hasAnyRole("ADMIN", "OWNER")
    .requestMatchers("/user/**").access(new CustomAuthorizationManager())
    .anyRequest().denyAll()
)
```

**Order matters:** Rules evaluated top-down; first match wins.

### Multiple SecurityFilterChains

Split config for different URL patterns:

```java
@Bean
@Order(1)
public SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults())
        .sessionManagement(s -> s.sessionCreationPolicy(STATELESS));
    return http.build();
}

@Bean
@Order(2)
public SecurityFilterChain webChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/login").permitAll()
            .anyRequest().authenticated())
        .formLogin(Customizer.withDefaults());
    return http.build();
}
```

Lower `@Order` = evaluated first.

### Customizer Pattern

Spring Security 6 uses `Customizer<HttpSecurity>` and lambda DSL:

```java
http
    .csrf(csrf -> csrf.disable())
    .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
    .formLogin(form -> form
        .loginPage("/login")
        .loginProcessingUrl("/perform_login")
        .defaultSuccessUrl("/home", true)
        .failureUrl("/login?error")
        .permitAll()
    )
    .logout(logout -> logout
        .logoutUrl("/logout")
        .logoutSuccessUrl("/login?logout")
        .deleteCookies("JSESSIONID")
    );
```

---

## 4. AuthenticationManager & ProviderManager

### AuthenticationManager

Central interface:

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication)
        throws AuthenticationException;
}
```

### ProviderManager

Default implementation — delegates to a list of `AuthenticationProvider`s.

```java
@Bean
public AuthenticationManager authenticationManager(
        AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
}
```

### AuthenticationProvider

Handles a specific type of `Authentication`.

```java
@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {
    
    private final UserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;
    
    public CustomAuthenticationProvider(UserDetailsService uds, PasswordEncoder encoder) {
        this.userDetailsService = uds;
        this.passwordEncoder = encoder;
    }
    
    @Override
    public Authentication authenticate(Authentication auth) 
            throws AuthenticationException {
        String username = auth.getName();
        String password = auth.getCredentials().toString();
        
        UserDetails user = userDetailsService.loadUserByUsername(username);
        
        if (!passwordEncoder.matches(password, user.getPassword())) {
            throw new BadCredentialsException("Invalid credentials");
        }
        
        return new UsernamePasswordAuthenticationToken(
            user, password, user.getAuthorities());
    }
    
    @Override
    public boolean supports(Class<?> auth) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(auth);
    }
}
```

### Provider Order

Multiple providers → first that returns non-null wins.

### AuthenticationException Hierarchy

```
AuthenticationException
├── BadCredentialsException
├── UsernameNotFoundException
├── DisabledException
├── LockedException
├── AccountExpiredException
├── CredentialsExpiredException
└── InsufficientAuthenticationException
```

### Programmatic Authentication

```java
@Autowired
private AuthenticationManager authManager;

public String login(String username, String password) {
    Authentication auth = authManager.authenticate(
        new UsernamePasswordAuthenticationToken(username, password));
    SecurityContextHolder.getContext().setAuthentication(auth);
    return "Logged in";
}
```

### Parent AuthenticationManager

For multiple `HttpSecurity` configs:

```java
@Bean
public AuthenticationManager authManager(
        HttpSecurity http, 
        AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
}
```

---

## 5. UserDetails & UserDetailsService

### UserDetails

Interface representing a user:

```java
public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();
    String getUsername();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

### Built-In Implementations

- `User` (builder-based)
- `org.springframework.security.core.userdetails.User`

```java
UserDetails user = User.builder()
    .username("alice")
    .password(passwordEncoder.encode("pwd"))
    .roles("USER", "ADMIN")
    .accountExpired(false)
    .accountLocked(false)
    .credentialsExpired(false)
    .disabled(false)
    .build();
```

### Custom UserDetails

```java
public class AppUser implements UserDetails {
    
    private final Long id;
    private final String username;
    private final String password;
    private final Set<GrantedAuthority> authorities;
    private final boolean enabled;
    
    public AppUser(User entity) {
        this.id = entity.getId();
        this.username = entity.getUsername();
        this.password = entity.getPassword();
        this.authorities = entity.getRoles().stream()
            .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
            .collect(Collectors.toSet());
        this.enabled = entity.isEnabled();
    }
    
    @Override public Collection<? extends GrantedAuthority> getAuthorities() { return authorities; }
    @Override public String getPassword() { return password; }
    @Override public String getUsername() { return username; }
    @Override public boolean isAccountNonExpired() { return true; }
    @Override public boolean isAccountNonLocked() { return true; }
    @Override public boolean isCredentialsNonExpired() { return true; }
    @Override public boolean isEnabled() { return enabled; }
    
    public Long getId() { return id; }
}
```

### UserDetailsService

Loads user by username:

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username)
        throws UsernameNotFoundException;
}
```

### Custom UserDetailsService

```java
@Service
public class JpaUserDetailsService implements UserDetailsService {
    
    private final UserRepository userRepo;
    
    public JpaUserDetailsService(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
    
    @Override
    public UserDetails loadUserByUsername(String username) {
        return userRepo.findByUsername(username)
            .map(AppUser::new)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found: " + username));
    }
}
```

Spring Security auto-detects a single `UserDetailsService` bean and uses it as the default.

### Using the Principal in Controllers

```java
@GetMapping("/me")
public UserDto me(@AuthenticationPrincipal AppUser user) {
    return new UserDto(user.getId(), user.getUsername());
}

// Or
@GetMapping("/me")
public UserDto me(Principal principal) {
    return userService.findByUsername(principal.getName());
}

// Or
@GetMapping("/me")
public UserDto me(Authentication auth) {
    AppUser u = (AppUser) auth.getPrincipal();
    return new UserDto(u.getId(), u.getUsername());
}
```

### Roles vs Authorities

- **Authority** — a permission: `READ_USERS`, `WRITE_ORDERS`
- **Role** — an authority with `ROLE_` prefix: `ROLE_ADMIN`

```java
.roles("ADMIN", "USER")           // → ROLE_ADMIN, ROLE_USER
.authorities("READ_USERS", "ROLE_ADMIN")  // raw authorities
```

Check:

```java
.hasRole("ADMIN")           // matches ROLE_ADMIN
.hasAuthority("ROLE_ADMIN") // matches ROLE_ADMIN
.hasAuthority("READ_USERS") // matches READ_USERS
```

⚠️ Don't mix — if you store `ADMIN`, use `hasAuthority("ADMIN")`, not `hasRole("ADMIN")`.

---

## 6. PasswordEncoder

### Why Encode?

- Never store plaintext passwords
- Hash is one-way, salted
- Resistant to rainbow table attacks

### Recommended: BCrypt

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

### PasswordEncoder Interface

```java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword);
    boolean matches(CharSequence rawPassword, String encodedPassword);
}
```

### Implementations

| Encoder | Notes |
|---------|-------|
| `BCryptPasswordEncoder` | Recommended, adaptive |
| `Argon2PasswordEncoder` | Winner PHC, best security |
| `Pbkdf2PasswordEncoder` | Standards-based |
| `SCryptPasswordEncoder` | Memory-hard |
| `NoOpPasswordEncoder` | ⚠️ **Never use** |
| `DelegatingPasswordEncoder` | Prefix-based, supports migration |

### DelegatingPasswordEncoder (Spring Default)

Prefix format: `{bcrypt}$2a$10$...`

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

Supports multiple encoders, prefixed in the encoded value. Old passwords keep working when migrating algorithms.

### Migration

1. Add new encoder to `DelegatingPasswordEncoder`
2. Existing hashes prefixed with old scheme
3. On next login, re-hash with new scheme

### Salt & Strength

BCrypt uses random salts automatically. Strength default 10; higher = slower = more secure.

```java
new BCryptPasswordEncoder(12);   // strength 12 (slower)
```

### Encoding

```java
String hash = encoder.encode("myPassword");
// $2a$10$...
```

### Matching

```java
boolean ok = encoder.matches("myPassword", storedHash);
```

### Upgrading Encoded Passwords

```java
if (encoder.upgradeEncoding(storedHash)) {
    user.setPassword(encoder.encode(rawPassword));
}
```

---

## 7. In-Memory Authentication

For dev/testing.

### Via Properties

```properties
spring.security.user.name=admin
spring.security.user.password=secret
spring.security.user.roles=ADMIN,USER
```

### Via InMemoryUserDetailsManager

```java
@Bean
public UserDetailsService userDetailsService(PasswordEncoder encoder) {
    UserDetails admin = User.builder()
        .username("admin")
        .password(encoder.encode("admin123"))
        .roles("ADMIN", "USER")
        .build();
    
    UserDetails user = User.builder()
        .username("user")
        .password(encoder.encode("user123"))
        .roles("USER")
        .build();
    
    return new InMemoryUserDetailsManager(admin, user);
}
```

### Via AuthenticationManagerBuilder (legacy)

```java
@Autowired
public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
    auth.inMemoryAuthentication()
        .withUser("admin").password(encoder.encode("pwd")).roles("ADMIN");
}
```

Modern: define `UserDetailsService` bean.

---

## 8. JDBC Authentication

Store users in DB.

### Schema

```sql
CREATE TABLE users (
    username VARCHAR(50) PRIMARY KEY,
    password VARCHAR(500) NOT NULL,
    enabled BOOLEAN NOT NULL
);

CREATE TABLE authorities (
    username VARCHAR(50) NOT NULL,
    authority VARCHAR(50) NOT NULL,
    FOREIGN KEY (username) REFERENCES users(username)
);
```

### Configuration

```java
@Bean
public UserDetailsService userDetailsService(DataSource dataSource) {
    JdbcUserDetailsManager manager = new JdbcUserDetailsManager(dataSource);
    manager.setUsersByUsernameQuery(
        "SELECT username, password, enabled FROM users WHERE username = ?");
    manager.setAuthoritiesByUsernameQuery(
        "SELECT username, authority FROM authorities WHERE username = ?");
    return manager;
}
```

Custom queries override default schema names.

### Programmatic User Creation

```java
@Service
public class UserService {
    private final UserDetailsManager userDetailsManager;
    private final PasswordEncoder encoder;
    
    public void register(String username, String rawPassword) {
        UserDetails user = User.builder()
            .username(username)
            .password(encoder.encode(rawPassword))
            .roles("USER")
            .build();
        userDetailsManager.createUser(user);
    }
}
```

### Custom Schema (Own Tables)

Provide queries to match your schema, or implement a custom `UserDetailsService`.

---

## 9. Form Login

### Default

With form login enabled, Spring Security:

- Serves `/login` page automatically (default)
- POST `/login` processes authentication
- Redirects to `/` after success
- Redirects to `/login?error` on failure

### Custom Form Login

```java
http.formLogin(form -> form
    .loginPage("/login")                    // GET endpoint
    .loginProcessingUrl("/perform_login")   // POST endpoint (Spring handles)
    .defaultSuccessUrl("/dashboard", true)  // always redirect here
    .failureUrl("/login?error=true")
    .usernameParameter("username")
    .passwordParameter("password")
    .permitAll()
);
```

### Thymeleaf Login Page

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><title>Login</title></head>
<body>
<form th:action="@{/perform_login}" method="post">
    <input type="text" name="username" placeholder="Username" required/>
    <input type="password" name="password" placeholder="Password" required/>
    <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}"/>
    <button type="submit">Login</button>
</form>
<p th:if="${param.error}">Invalid credentials</p>
<p th:if="${param.logout}">Logged out</p>
</body>
</html>
```

**CSRF token** must be included for form POST.

### Success Handler

```java
http.formLogin(form -> form
    .successHandler((req, res, auth) -> {
        log.info("Login: {}", auth.getName());
        res.sendRedirect("/dashboard");
    })
    .failureHandler((req, res, ex) -> {
        log.warn("Failed login: {}", ex.getMessage());
        res.sendRedirect("/login?error");
    })
);
```

### AuthenticationSuccessHandler (interface)

```java
@Component
public class MySuccessHandler implements AuthenticationSuccessHandler {
    @Override
    public void onAuthenticationSuccess(HttpServletRequest req, 
                                        HttpServletResponse res,
                                        Authentication auth) throws IOException {
        // Custom logic (e.g., set cookies, log audit)
        res.sendRedirect("/dashboard");
    }
}
```

---

## 10. HTTP Basic

Client sends `Authorization: Basic base64(username:password)`.

### Enable

```java
http.httpBasic(Customizer.withDefaults());
```

### Custom Entry Point

```java
http.httpBasic(basic -> basic
    .realmName("MyApp")
    .authenticationEntryPoint(new CustomBasicEntryPoint())
);
```

### Client Usage

```bash
curl -u admin:secret http://localhost:8080/api/users
```

Or header:

```
Authorization: Basic YWRtaW46c2VjcmV0
```

### When to Use

- Simple REST APIs
- Internal services
- Testing

### Downsides

- Sends credentials every request
- Must use HTTPS
- No logout concept (client discards credentials)
- No session management benefit

---

## 11. Session Management

### Session-Based Authentication

Form login stores the authenticated user in `HttpSession`. Subsequent requests send `JSESSIONID` cookie.

### Session Configuration

```java
http.sessionManagement(session -> session
    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
    .maximumSessions(1)                    // max concurrent sessions per user
    .maxSessionsPreventsLogin(false)       // false = evict oldest; true = block new
    .expiredUrl("/login?expired")
    .sessionFixation().migrateSession()    // default
    .sessionConcurrency(c -> c
        .maximumSessions(1)
        .expiredUrl("/login?expired"))
);
```

### SessionCreationPolicy

| Policy | Description |
|--------|-------------|
| `ALWAYS` | Always create session |
| `IF_REQUIRED` (default) | Create on demand |
| `NEVER` | Don't create but use if present |
| `STATELESS` | Never create/use — for JWT |

### STATELESS for JWT APIs

```java
http.sessionManagement(s -> s.sessionCreationPolicy(STATELESS));
```

### Session Fixation Protection

Attack: attacker sets a known session ID before victim logs in. Fix: generate new session ID on login.

| Strategy | Behavior |
|----------|----------|
| `none()` | No change (insecure) |
| `newSession()` | New session, no attributes copied |
| `migrateSession()` (default) | New session, attributes copied |
| `changeSessionId()` | Servlet 3.1 — same session, new ID |

### Session Timeout

```properties
server.servlet.session.timeout=30m
```

### Concurrent Sessions

```java
@Bean
public HttpSessionEventPublisher httpSessionEventPublisher() {
    return new HttpSessionEventPublisher();
}
```

Required for concurrent session control to detect session destruction.

### Distributed Sessions (Redis)

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

```properties
spring.session.store-type=redis
spring.session.timeout=30m
spring.data.redis.host=localhost
```

Sessions are shared across app instances.

### Stateless JWT Alternative

For stateless APIs (JWT), disable sessions entirely:

```java
http
    .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
    .csrf(csrf -> csrf.disable());   // no session = no CSRF token
```

### Session in Controllers

```java
@GetMapping("/cart")
public Cart cart(HttpSession session) {
    Cart cart = (Cart) session.getAttribute("cart");
    if (cart == null) {
        cart = new Cart();
        session.setAttribute("cart", cart);
    }
    return cart;
}
```

---

## 12. CSRF Protection

### What is CSRF?

**Cross-Site Request Forgery** — attacker tricks a logged-in user's browser into making an unintended request to your site (e.g., transferring money).

### Why It Works

Browsers automatically send cookies (including `JSESSIONID`) with cross-site requests. If server authenticates by cookie, the attacker can trigger authenticated actions.

### How Spring Protects

Spring generates a **CSRF token** per session. Every state-changing request (POST, PUT, PATCH, DELETE) must include a matching token.

### Enabled by Default

For **stateful** (session-based) apps: CSRF protection is **on**.

For **stateless** (JWT) APIs: disable CSRF since no cookie-based auth.

### Form Usage

```html
<form method="post" action="/users">
    <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}"/>
</form>
```

Or via meta tags + JS.

### AJAX Usage

```javascript
fetch('/api/users', {
    method: 'POST',
    headers: {
        'X-CSRF-TOKEN': document.querySelector('meta[name="_csrf"]').content,
        'Content-Type': 'application/json'
    },
    body: JSON.stringify(data)
});
```

### Cookie-Based (for SPAs)

```java
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
);
```

Sends `XSRF-TOKEN` cookie that JS reads and sends back as `X-XSRF-TOKEN` header.

### Disabling CSRF

```java
http.csrf(csrf -> csrf.disable());
```

**When to disable:**
- Stateless REST API using JWT/Bearer tokens
- No cookies for authentication

**When NOT to disable:**
- Form-login web apps
- Cookie-based session auth

### Selective Disabling

```java
http.csrf(csrf -> csrf
    .ignoringRequestMatchers("/api/webhooks/**", "/actuator/**")
);
```

### CSRF Token Repository

| Repository | Storage |
|-----------|---------|
| `HttpSessionCsrfTokenRepository` (default) | Session |
| `CookieCsrfTokenRepository` | Cookie |
| Custom | Anywhere |

### When It Fails

- Missing token → 403
- Token mismatch → 403
- Token expired → 403

---

## 13. CORS

### What is CORS?

**Cross-Origin Resource Sharing** — browser security policy that blocks JS from making requests to a different origin unless the server explicitly allows it.

Origin = scheme + host + port. `http://localhost:3000` ≠ `http://localhost:8080`.

### CORS Flow

```
1. Browser sends OPTIONS preflight (for non-simple requests)
2. Server responds with Access-Control-Allow-* headers
3. If allowed → actual request
4. If not → browser blocks response
```

### Global CORS Config

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .exposedHeaders("X-Total-Count")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

### CORS in Spring Security

Spring Security has its own CORS handling — enable it:

```java
http.cors(Customizer.withDefaults());
```

Without this, preflight requests fail at the security layer.

### CorsConfigurationSource Bean

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

Spring Security auto-uses this bean.

### @CrossOrigin (Per Controller)

```java
@RestController
@CrossOrigin(origins = "https://app.example.com", maxAge = 3600)
public class UserController {
    
    @GetMapping("/users")
    @CrossOrigin(origins = "*")
    public List<User> list() { ... }
}
```

### allowedOrigins vs allowedOriginPatterns

- `allowedOrigins("https://a.com")` — exact match
- `allowedOriginPatterns("https://*.example.com")` — pattern, required with `allowCredentials=true`

With `allowCredentials(true)`:
- Cannot use `allowedOrigins("*")`
- Use `allowedOriginPatterns("*")` or explicit origins

### Preflight Cache

```properties
spring.mvc.dispatch-options-request=true
```

Set `maxAge` to cache preflight responses.

### Common Issues

| Symptom | Cause |
|---------|-------|
| CORS preflight fails (403) | Spring Security not configured for CORS |
| Missing `Access-Control-Allow-Origin` | No `cors()` config |
| Credentials blocked | `allowCredentials(true)` with `*` origin |
| Custom header blocked | Not in `allowedHeaders` or `exposedHeaders` |

---

## 14. Logout

### Default Logout

`POST /logout` (with CSRF token) invalidates session and clears cookies.

### Custom Logout

```java
http.logout(logout -> logout
    .logoutUrl("/signout")                     // POST endpoint
    .logoutSuccessUrl("/login?logout")         // redirect
    .logoutSuccessHandler(customHandler)       // or handler
    .invalidateHttpSession(true)               // default true
    .clearAuthentication(true)                 // default true
    .deleteCookies("JSESSIONID", "remember-me")
    .permitAll()
);
```

### GET Logout (not recommended)

```java
.logoutRequestMatcher(new AntPathRequestMatcher("/logout", "GET"))
```

⚠️ GET logout is CSRF-vulnerable.

### LogoutSuccessHandler

```java
@Component
public class CustomLogoutHandler implements LogoutSuccessHandler {
    @Override
    public void onLogoutSuccess(HttpServletRequest req, 
                                 HttpServletResponse res,
                                 Authentication auth) throws IOException {
        if (auth != null) {
            audit.logout(auth.getName());
        }
        res.sendRedirect("/login?logout");
    }
}
```

### Per-User Logout

For JWT, "logout" typically means invalidating the token (blacklist) or short expiry + refresh tokens.

---

## 15. Remember-Me Authentication

### What It Does

Persists authentication across browser sessions using a cookie. User stays logged in after closing browser.

### Token-Based (Simple)

```java
http.rememberMe(remember -> remember
    .key("uniqueAndSecret")
    .tokenValiditySeconds(60 * 60 * 24 * 14)   // 2 weeks
    .rememberMeParameter("remember-me")
);
```

Login form:

```html
<input type="checkbox" name="remember-me"/>
```

### Persistent Token (Database)

More secure — tokens stored in DB, rotated.

```java
@Bean
public PersistentTokenRepository tokenRepository(DataSource ds) {
    JdbcTokenRepositoryImpl repo = new JdbcTokenRepositoryImpl();
    repo.setDataSource(ds);
    return repo;
}

http.rememberMe(remember -> remember
    .tokenRepository(tokenRepository)
    .tokenValiditySeconds(60 * 60 * 24 * 14));
```

Schema required:

```sql
CREATE TABLE persistent_logins (
    username VARCHAR(64) NOT NULL,
    series VARCHAR(64) PRIMARY KEY,
    token VARCHAR(64) NOT NULL,
    last_used TIMESTAMP NOT NULL
);
```

### How It Works

1. First login with "remember me" → server sends cookie with series + token
2. Cookie is base64(username:series:token)
3. Next visit → server validates series, checks token
4. New token issued; old invalidated

### Security Considerations

- Cookie must be HTTPS-only
- Compromised cookie = full account access
- Persistent token repository adds DB validation

### Combining with JWT

For modern APIs, use refresh tokens instead of remember-me. Remember-me is a session-based concept.

---

## 16. Security Headers

Spring Security sets several headers by default.

### Default Headers

| Header | Value | Purpose |
|--------|-------|---------|
| `X-Content-Type-Options` | `nosniff` | Prevent MIME sniffing |
| `X-Frame-Options` | `DENY` | Prevent clickjacking |
| `X-XSS-Protection` | `0` (modern browsers ignore) | Legacy |
| `Cache-Control` | `no-cache, no-store, ...` | Prevent caching sensitive data |
| `Strict-Transport-Security` | (HTTPS only) | Force HTTPS |
| `Content-Security-Policy` | (off by default) | XSS mitigation |

### Customize Headers

```java
http.headers(headers -> headers
    .frameOptions(frame -> frame.sameOrigin())    // allow same-origin frames
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; script-src 'self'"))
    .httpStrictTransportSecurity(hsts -> hsts
        .includeSubDomains(true)
        .maxAgeInSeconds(31536000))
    .referrerPolicy(ref -> ref
        .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
    .permissionsPolicy(pp -> pp
        .policy("geolocation=(), microphone=()"))
);
```

### CSP

```java
.contentSecurityPolicy(csp -> csp
    .policyDirectives(
        "default-src 'self'; " +
        "script-src 'self' https://cdn.example.com; " +
        "style-src 'self' 'unsafe-inline'; " +
        "img-src 'self' data:; " +
        "connect-src 'self' https://api.example.com"))
```

### HSTS

```properties
server.ssl.enabled=true
```

Then:

```java
http.headers(h -> h.httpStrictTransportSecurity(hsts -> hsts
    .includeSubDomains(true)
    .maxAgeInSeconds(31536000)
    .preload(true)));
```

### Disabling Headers (rare)

```java
http.headers(h -> h
    .frameOptions(HeadersConfigurer.FrameOptionsConfig::disable)
    .contentSecurityPolicy(Customizer.withDefaults())   // keep default
);
```

---

## 17. Interview Questions

### Q1. What is Spring Security?

**Answer:** A framework for authentication and authorization in Java apps. Provides a filter chain, `AuthenticationManager`, `UserDetailsService`, access control (URL and method level), CSRF/CORS protection, and integration with OAuth2, SAML, LDAP, etc.

### Q2. Difference between authentication and authorization?

**Answer:** Authentication = "who are you?" (verify identity). Authorization = "what can you do?" (check permissions). Authentication happens first; authorization follows.

### Q3. Explain Spring Security architecture.

**Answer:** A `DelegatingFilterProxy` bridges Servlet filters to Spring beans. `FilterChainProxy` dispatches to one or more `SecurityFilterChain`s. Each chain is an ordered list of filters (`SecurityContextHolderFilter`, `UsernamePasswordAuthenticationFilter`, `AuthorizationFilter`, etc.). `AuthenticationManager` (via `ProviderManager`) authenticates using `AuthenticationProvider`s. `SecurityContextHolder` stores the authenticated principal.

### Q4. What is SecurityFilterChain?

**Answer:** A chain of security filters applied to matching requests. Configured as a `SecurityFilterChain` bean. Multiple chains can be registered with different matchers and `@Order`.

### Q5. What is the SecurityContextHolder?

**Answer:** Holds the authenticated user (`Authentication` object) for the current request/thread. Strategies: `MODE_THREADLOCAL` (default), `MODE_INHERITABLETHREADLOCAL`, `MODE_GLOBAL`. Use `SecurityContextHolder.getContext().getAuthentication()`.

### Q6. What is UserDetailsService?

**Answer:** Interface with `loadUserByUsername(String)` returning `UserDetails`. Spring Security uses it to load users during authentication. Implement to connect to a DB, LDAP, etc.

### Q7. What is UserDetails?

**Answer:** Interface representing a user: `getAuthorities()`, `getPassword()`, `getUsername()`, and account status flags (`isAccountNonExpired()`, etc.). Implementations: `User`, custom.

### Q8. What PasswordEncoder should I use?

**Answer:** BCrypt (default in Spring Boot), Argon2, or SCrypt. Never `NoOpPasswordEncoder` in production. Use `DelegatingPasswordEncoder` for algorithm migration.

### Q9. What is CSRF?

**Answer:** Cross-Site Request Forgery — attacker tricks a logged-in user's browser into a state-changing request. Spring protects by requiring a per-session CSRF token for POST/PUT/DELETE. Disabled for stateless JWT APIs.

### Q10. What is CORS and how is it configured?

**Answer:** Cross-Origin Resource Sharing — browser policy blocking cross-origin requests. Configure with `WebMvcConfigurer.addCorsMappings` or `CorsConfigurationSource` bean. In Spring Security, call `.cors()` to enable.

### Q11. Session vs stateless?

**Answer:** Session-based stores auth in `HttpSession` (JSESSIONID cookie). Stateless (JWT) sends a token with every request, no server session. Stateless scales better, no session affinity. Session easier for form login.

### Q12. What is session fixation?

**Answer:** Attacker sets a known session ID before victim logs in, then hijacks the session. Spring fixes by generating a new session ID on login (`migrateSession()` default).

### Q13. How do you configure multiple security filter chains?

**Answer:** Define multiple `SecurityFilterChain` beans with different `securityMatcher()` and `@Order`. First matching chain applies.

### Q14. What are the account status flags?

**Answer:** `isAccountNonExpired`, `isAccountNonLocked`, `isCredentialsNonExpired`, `isEnabled`. If any is false, authentication fails with `AccountExpiredException`, `LockedException`, `CredentialsExpiredException`, or `DisabledException`.

### Q15. Difference between `hasRole` and `hasAuthority`?

**Answer:** `hasRole("ADMIN")` checks for `ROLE_ADMIN`. `hasAuthority("ROLE_ADMIN")` checks for the exact string. Roles are authorities with a prefix.

### Q16. How do you get the current user?

**Answer:** In controllers: `@AuthenticationPrincipal`, `Principal`, or `Authentication` param. Programmatically: `SecurityContextHolder.getContext().getAuthentication()`.

### Q17. What is the difference between `AuthenticationEntryPoint` and `AccessDeniedHandler`?

**Answer:** `AuthenticationEntryPoint` handles **unauthenticated** requests (returns 401). `AccessDeniedHandler` handles **forbidden** requests (returns 403).

### Q18. What is a `WebSecurityConfigurerAdapter`?

**Answer:** A legacy base class for security config (deprecated in Spring Security 5.7, removed in 6.0). Modern: define `SecurityFilterChain` bean.

### Q19. How do you test Spring Security endpoints?

**Answer:** Use `@WebMvcTest` + `@WithMockUser` / `@WithUserDetails` for unit tests. `@SpringBootTest` with `MockMvc` for integration. Add `spring-security-test` dependency.

### Q20. What security headers are set by default?

**Answer:** `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Cache-Control: no-cache, no-store`, `Strict-Transport-Security` (HTTPS), and `X-XSS-Protection: 0`.

### Q21. How do you handle 401 vs 403?

**Answer:** 401 Unauthorized → user not authenticated (redirect to login or send `WWW-Authenticate`). 403 Forbidden → authenticated but denied. Customize via `AuthenticationEntryPoint` and `AccessDeniedHandler`.

### Q22. What is anonymous authentication?

**Answer:** Spring assigns an `AnonymousAuthenticationToken` to unauthenticated requests so `Authentication` is never null. `isAnonymous()` checks it. `permitAll()` allows anonymous access.

### Q23. How do you change session timeout?

**Answer:** `server.servlet.session.timeout=30m` in `application.properties`. Or programmatically via `HttpSessionListener`/`SessionRepository`.

### Q24. What is `@EnableWebSecurity`?

**Answer:** Enables Spring Security's web support. Auto-configured in Spring Boot when `spring-boot-starter-security` is present. Only needed explicitly in non-Boot apps.

### Q25. How do you secure Actuator endpoints?

**Answer:** Use `EndpointRequest.toAnyEndpoint()` in a separate `SecurityFilterChain` with role restrictions. Or filter by URL paths (`/actuator/**`) in the main chain. Consider running Actuator on a separate port.

---

## 18. Cheat Sheet

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

### Minimal Config

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated())
            .formLogin(Customizer.withDefaults())
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

### PasswordEncoder

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

### UserDetailsService

```java
@Service
public class JpaUserService implements UserDetailsService {
    public UserDetails loadUserByUsername(String u) {
        return userRepo.findByUsername(u).map(AppUser::new)
            .orElseThrow(() -> new UsernameNotFoundException(u));
    }
}
```

### Authorization Rules

```java
.requestMatchers("/public/**").permitAll()
.requestMatchers("/admin/**").hasRole("ADMIN")
.requestMatchers("/api/**").hasAnyRole("USER", "ADMIN")
.requestMatchers(HttpMethod.GET, "/reports/**").hasAuthority("REPORT_VIEW")
.anyRequest().authenticated()
```

### Access Control (both)

```java
http.authorizeHttpRequests(auth -> auth...)

// Method-level
@EnableMethodSecurity
@PreAuthorize("hasRole('ADMIN')")
```

### CSRF

```java
// Stateful (default)
http.csrf(Customizer.withDefaults());

// Stateless (JWT)
http.csrf(csrf -> csrf.disable());

// Cookie-based (SPA)
http.csrf(c -> c.csrfTokenRepository(
    CookieCsrfTokenRepository.withHttpOnlyFalse()));
```

### CORS

```java
http.cors(Customizer.withDefaults());

@Bean
public CorsConfigurationSource source() {
    CorsConfiguration c = new CorsConfiguration();
    c.setAllowedOrigins(List.of("https://app.example.com"));
    c.setAllowedMethods(List.of("GET","POST","PUT","DELETE"));
    c.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource s = new UrlBasedCorsConfigurationSource();
    s.registerCorsConfiguration("/api/**", c);
    return s;
}
```

### Session Management

```java
http.sessionManagement(s -> s
    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
    .maximumSessions(1)
    .maxSessionsPreventsLogin(false));

// Stateless for JWT:
.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
```

### Get Current User

```java
@AuthenticationPrincipal AppUser user
Principal principal
Authentication auth
SecurityContextHolder.getContext().getAuthentication()
```

### Logout

```java
http.logout(logout -> logout
    .logoutUrl("/signout")
    .logoutSuccessUrl("/login?logout")
    .deleteCookies("JSESSIONID")
    .invalidateHttpSession(true));
```

### Status Codes

| Code | Meaning | Trigger |
|------|---------|---------|
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | Authenticated, no access |
| 302 | Redirect | To login page |

### Common Properties

```properties
spring.security.user.name=admin
spring.security.user.password=secret
spring.security.user.roles=ADMIN

server.servlet.session.timeout=30m
spring.session.store-type=redis
```

### Test

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mvc;
    
    @Test
    @WithMockUser(roles = "USER")
    void allowedForUser() throws Exception {
        mvc.perform(get("/api/users")).andExpect(status().isOk());
    }
    
    @Test
    @WithAnonymousUser
    void unauthorized() throws Exception {
        mvc.perform(get("/api/users")).andExpect(status().isUnauthorized());
    }
}
```

### Cross-References

- **Previous:** `12_Spring_Caching.md`
- **Next:** `14_Spring_Security_JWT_OAuth2.md`
- **Related:** `03_Spring_MVC.md`, `15_Spring_Security_Method_Level.md`
- **Interview:** `24_Spring_Interview_Questions.md` (§Security)

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [12_Spring_Caching.md](./12_Spring_Caching.md)
- **Next →:** [14_Spring_Security_JWT_OAuth2.md](./14_Spring_Security_JWT_OAuth2.md)
- **Related:** [01_Spring_Framework_Core.md](./01_Spring_Framework_Core.md), [14_Spring_Security_JWT_OAuth2.md](./14_Spring_Security_JWT_OAuth2.md), [16_Spring_Testing.md](./16_Spring_Testing.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
