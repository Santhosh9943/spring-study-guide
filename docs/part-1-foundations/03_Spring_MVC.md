# 03_Spring_MVC.md

# Spring MVC & Web Layer

> **File:** `03_Spring_MVC.md`
> **Part:** 1 — Foundations
> **Prerequisites:** `01_Spring_Framework_Core.md`, `02_Spring_AOP.md`
> **Estimated Study Time:** 10–14 hours

---

## Table of Contents

1. [MVC Pattern Overview](#1-mvc-pattern-overview)
2. [DispatcherServlet — Front Controller](#2-dispatcherservlet--front-controller)
3. [Spring MVC Request Flow](#3-spring-mvc-request-flow)
4. [@Controller vs @RestController](#4-controller-vs-restcontroller)
5. [Request Mapping Annotations](#5-request-mapping-annotations)
6. [Request Data Binding](#6-request-data-binding)
7. [Response Handling](#7-response-handling)
8. [Data Binding & Type Conversion](#8-data-binding--type-conversion)
9. [Validation](#9-validation)
10. [Exception Handling](#10-exception-handling)
11. [Interceptors](#11-interceptors)
12. [Filters vs Interceptors](#12-filters-vs-interceptors)
13. [View Resolvers](#13-view-resolvers)
14. [Content Negotiation](#14-content-negotiation)
15. [REST Principles & HATEOAS](#15-rest-principles--hateoas)
16. [File Upload/Download](#16-file-uploaddownload)
17. [Async Request Processing](#17-async-request-processing)
18. [Interview Questions](#18-interview-questions)
19. [Cheat Sheet](#19-cheat-sheet)

---

## 1. MVC Pattern Overview

**Model-View-Controller** is a design pattern that separates an application into three roles:

| Component | Responsibility |
|-----------|---------------|
| **Model** | Business data and logic |
| **View** | Presentation / rendering |
| **Controller** | Handles requests, orchestrates Model & View |

### Flow

```
User → Controller → Model → View → User
         ↑            ↓
      (request)   (data/logic)
```

### Benefits

- **Separation of concerns** — UI decoupled from logic
- **Testability** — controller/model testable without UI
- **Reusability** — same model for multiple views
- **Parallel development** — UI team + backend team

### Spring MVC Position

Spring MVC is a **request-driven, front-controller** framework. It implements MVC for web apps and REST APIs.

---

## 2. DispatcherServlet — Front Controller

**DispatcherServlet** is the central servlet of Spring MVC — the **Front Controller** that routes every HTTP request.

### Key Responsibilities

1. Receives all HTTP requests
2. Consults `HandlerMapping` to find a handler
3. Invokes `HandlerAdapter` to call the handler
4. Resolves the view (for HTML) or serializes the body (for REST)
5. Handles exceptions via `HandlerExceptionResolver`

### Registration

**Spring Boot (auto):**

```properties
spring.mvc.servlet.path=/api     # optional
```

**Traditional web.xml:**

```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>
        org.springframework.web.servlet.DispatcherServlet
    </servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

**Java config:**

```java
public class MyWebAppInitializer 
        extends AbstractAnnotationConfigDispatcherServletInitializer {
    @Override
    protected Class<?>[] getRootConfigClasses() { return new Class[]{RootConfig.class}; }
    @Override
    protected Class<?>[] getServletConfigClasses() { return new Class[]{WebConfig.class}; }
    @Override
    protected String[] getServletMappings() { return new String[]{"/"}; }
}
```

### WebApplicationContext Hierarchy

```
RootContext (parent)           WebContext (child, DispatcherServlet)
───────────────────            ───────────────────────────────────
@Service                       @Controller
@Repository                    @ControllerAdvice
@Configuration                 ViewResolver
                               HandlerMapping
                               (can access parent's beans)
```

- **Root context** — services, repositories, data sources
- **Child context** — controllers, view resolvers, MVC infra
- Children can see parent beans; **not vice versa**

---

## 3. Spring MVC Request Flow

### Complete Flow Diagram

```
         ┌──────────────────────────────────────────────────────┐
         │                     HTTP Request                     │
         └────────────────────────┬─────────────────────────────┘
                                  ▼
                    ┌────────────────────────────┐
                    │      DispatcherServlet     │
                    └────────────┬───────────────┘
                                 ▼
                    ┌────────────────────────────┐
                    │      HandlerMapping        │◄─── UrlHandlerMapping,
                    │   (find controller)        │     RequestMappingHandlerMapping
                    └────────────┬───────────────┘
                                 ▼  HandlerExecutionChain
                    ┌────────────────────────────┐
                    │  PreHandle Interceptors    │
                    └────────────┬───────────────┘
                                 ▼
                    ┌────────────────────────────┐
                    │      HandlerAdapter        │◄─── RequestMappingHandlerAdapter
                    │  (invoke handler method)   │
                    └────────────┬───────────────┘
                                 ▼
                    ┌────────────────────────────┐
                    │  Argument Resolvers        │◄─── @RequestParam, @RequestBody,
                    │  (bind parameters)         │     @PathVariable, etc.
                    └────────────┬───────────────┘
                                 ▼
                    ┌────────────────────────────┐
                    │       Controller           │
                    │       method(...)          │
                    └────────────┬───────────────┘
                                 ▼
                    ┌────────────────────────────┐
                    │  ReturnValueHandlers       │◄─── @ResponseBody,
                    │  (process return value)    │     ResponseEntity, String(view)
                    └────────────┬───────────────┘
                                 ▼
                    ┌────────────────────────────┐
                    │  PostHandle Interceptors   │
                    └────────────┬───────────────┘
                                 ▼
              ┌──────────────────┴─────────────────┐
              ▼                                    ▼
    ┌──────────────────┐                ┌───────────────────┐
    │  ViewResolver    │                │  HttpMessageConv. │
    │  (render HTML)   │                │  (JSON/XML body)  │
    └────────┬─────────┘                └─────────┬─────────┘
             ▼                                    ▼
      HTML Response                       JSON/XML Response
                      \                  /
                       \                /
                        ▼              ▼
                    ┌──────────────────────┐
                    │   AfterComplete      │
                    │   Interceptors       │
                    └──────────┬───────────┘
                               ▼
                       ┌───────────────┐
                       │ HTTP Response │
                       └───────────────┘
```

### Step-by-Step

1. **Request** hits `DispatcherServlet` (mapped to `/`)
2. **HandlerMapping** identifies the handler and its interceptors
3. Interceptors' `preHandle()` runs (reverse order after match)
4. **HandlerAdapter** invokes the controller method
5. **Argument resolvers** bind params (`@RequestParam`, `@RequestBody`, ...)
6. **Controller** executes logic
7. **ReturnValueHandlers** process return value (`@ResponseBody` → `HttpMessageConverter`; `String` → view name)
8. Interceptors' `postHandle()`
9. **ViewResolver** resolves logical view → actual view (for HTML)
10. **View** renders into the response
11. Interceptors' `afterCompletion()`

### Key Components (Framework Internals)

| Component | Role |
|-----------|------|
| `HandlerMapping` | Maps request → handler + interceptors |
| `HandlerAdapter` | Invokes the handler |
| `HandlerExceptionResolver` | Resolves exceptions |
| `HandlerMethodReturnValueHandler` | Processes return values |
| `HandlerMethodArgumentResolver` | Binds method args |
| `HttpMessageConverter` | Converts `@RequestBody`/`@ResponseBody` |
| `ViewResolver` | Maps view name → `View` |
| `MultipartResolver` | Handles file uploads |
| `LocaleResolver` | Resolves locale for i18n |
| `ThemeResolver` | Theme resolution |

---

## 4. @Controller vs @RestController

| Aspect | `@Controller` | `@RestController` |
|--------|--------------|------------------|
| Purpose | Web MVC with views | REST API |
| Return value | View name (String) | Body (serialized) |
| `@ResponseBody` | Must add per method | Implicit |
| Introduced | Spring 2.5 | Spring 4.0 |
| Composed of | `@Component` | `@Controller` + `@ResponseBody` |

```java
// @Controller — returns view name "users/list"
@Controller
public class UserViewController {
    @GetMapping("/users")
    public String list(Model model) {
        model.addAttribute("users", userService.findAll());
        return "users/list";           // → resolved by ViewResolver
    }
}

// @RestController — returns JSON body
@RestController
@RequestMapping("/api/users")
public class UserApiController {
    @GetMapping
    public List<User> list() {          // → serialized as JSON
        return userService.findAll();
    }
}
```

**To mix both in one `@Controller`:**

```java
@Controller
public class MixedController {
    @GetMapping("/page")
    public String page() { return "view"; }
    
    @GetMapping("/api/data")
    @ResponseBody
    public Data data() { return new Data(); }
}
```

---

## 5. Request Mapping Annotations

### HTTP Method Shortcuts

| Annotation | HTTP Method | Semantics |
|-----------|-------------|-----------|
| `@GetMapping` | GET | Retrieve |
| `@PostMapping` | POST | Create |
| `@PutMapping` | PUT | Replace |
| `@PatchMapping` | PATCH | Partial update |
| `@DeleteMapping` | DELETE | Delete |
| `@RequestMapping` | Any | General |

### @RequestMapping Attributes

```java
@RequestMapping(
    value = "/users/{id}",            // URL path
    method = RequestMethod.GET,       // HTTP method
    params = "format=json",           // required params
    headers = "X-API-Version=1",      // required headers
    consumes = MediaType.APPLICATION_JSON_VALUE,  // request content-type
    produces = MediaType.APPLICATION_JSON_VALUE,  // response content-type
    name = "getUser"
)
```

### Class-Level Base Path

```java
@RestController
@RequestMapping("/api/v1/users")     // base path applies to all methods
public class UserController {
    @GetMapping                      // GET /api/v1/users
    public List<User> list() { ... }
    
    @GetMapping("/{id}")             // GET /api/v1/users/{id}
    public User get(@PathVariable Long id) { ... }
    
    @PostMapping                     // POST /api/v1/users
    public User create(@RequestBody User u) { ... }
}
```

### Path Patterns

| Pattern | Matches |
|---------|---------|
| `/users/*` | One segment after users |
| `/users/**` | Any depth (Spring 5.3+) |
| `/users/{id}` | Path variable |
| `/users/{id:[0-9]+}` | Path variable with regex |
| `/files/**/*.jpg` | Ant-style (limited support in 5.3+) |

### Multiple Paths

```java
@GetMapping({"/users", "/people"})
```

---

## 6. Request Data Binding

### @PathVariable

Extract from URI template.

```java
@GetMapping("/users/{id}/orders/{orderId}")
public Order get(@PathVariable Long id, @PathVariable Long orderId) { ... }

// Rename:
@GetMapping("/users/{userId}")
public User get(@PathVariable("userId") Long id) { ... }

// Optional (Spring 4.3+):
@GetMapping({"/users", "/users/{id}"})
public User get(@PathVariable(required = false) Long id) { ... }
```

### @RequestParam

Extract from query string or form data.

```java
@GetMapping("/search")
public List<User> search(
    @RequestParam String q,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(required = false) String sort
) { ... }
```

### @RequestBody

Deserialize HTTP body (JSON/XML) into a Java object via `HttpMessageConverter`.

```java
@PostMapping("/users")
@ResponseStatus(HttpStatus.CREATED)
public User create(@RequestBody @Valid CreateUserRequest req) { ... }
```

### @RequestHeader

```java
@GetMapping("/data")
public Data data(
    @RequestHeader("User-Agent") String ua,
    @RequestHeader(value = "X-Trace", required = false) String trace,
    @RequestHeader HttpHeaders headers    // all headers
) { ... }
```

### @CookieValue

```java
@GetMapping("/profile")
public Profile profile(@CookieValue("sessionId") String sessionId) { ... }
```

### @MatrixVariable

Rare. Extracts semicolon-separated values.

```java
// /cars;color=red;year=2020
@GetMapping("/cars/{make}")
public Car car(@PathVariable String make,
               @MatrixVariable String color,
               @MatrixVariable int year) { ... }
```

### @ModelAttribute

Bind form data to a model object (form-backing bean).

```java
@PostMapping("/users")
public String create(@ModelAttribute UserForm form) { ... }
// All request params matched to form's setters
```

### @SessionAttribute

Access session attributes.

```java
@GetMapping("/cart")
public Cart cart(@SessionAttribute("cart") Cart cart) { ... }
```

### @RequestAttribute

Access request-scoped attributes (set by filters/interceptors).

```java
@GetMapping("/me")
public User me(@RequestAttribute("currentUser") User user) { ... }
```

### Binding Summary

| Annotation | Source |
|-----------|--------|
| `@PathVariable` | URI template |
| `@RequestParam` | Query string / form |
| `@RequestBody` | Request body |
| `@RequestHeader` | HTTP header |
| `@CookieValue` | Cookie |
| `@ModelAttribute` | Query/form → bean |
| `@SessionAttribute` | Session |
| `@RequestAttribute` | Request attribute |
| `@MatrixVariable` | Matrix path params |

---

## 7. Response Handling

### @ResponseBody

Serializes return value to body.

```java
@GetMapping("/ping")
@ResponseBody
public String ping() { return "pong"; }
```

### ResponseEntity<T>

Full control over status, headers, body.

```java
@GetMapping("/users/{id}")
public ResponseEntity<User> get(@PathVariable Long id) {
    return userService.findById(id)
        .map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
}

@PostMapping("/users")
public ResponseEntity<User> create(@RequestBody User user) {
    User saved = userService.save(user);
    URI location = URI.create("/api/users/" + saved.getId());
    return ResponseEntity.created(location).body(saved);
}
```

### @ResponseStatus

Sets status code declaratively.

```java
@PostMapping("/users")
@ResponseStatus(HttpStatus.CREATED)
public User create(@RequestBody User u) { return userService.save(u); }

@ResponseStatus(value = HttpStatus.NOT_FOUND, reason = "User not found")
public class UserNotFoundException extends RuntimeException { }
```

### Return Types Supported

| Type | Behaviour |
|------|-----------|
| `String` | View name (in `@Controller`); body (in `@RestController`) |
| `void` | No body, uses view from request or method name |
| `ModelAndView` | View + model |
| `ResponseEntity<T>` | Status + headers + body |
| `T` (any POJO) | Serialized to body in `@RestController` |
| `Optional<T>` | Body if present, else 404 (via `ResponseEntity`) |
| `Callable<T>` / `DeferredResult<T>` | Async |
| `Mono<T>` / `Flux<T>` | WebFlux reactive |

### Setting Headers

```java
@GetMapping("/export")
public ResponseEntity<byte[]> export() {
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, 
                "attachment; filename=export.csv")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(csvBytes);
}
```

---

## 8. Data Binding & Type Conversion

### Default Converters

Spring converts strings to primitives, wrappers, enums, dates, UUIDs, etc.

```java
@GetMapping("/users")
public List<User> list(
    @RequestParam int page,                      // "1" → 1
    @RequestParam Status status,                 // "ACTIVE" → Status.ACTIVE
    @RequestParam @DateTimeFormat(iso = ISO.DATE) LocalDate from,
    @RequestParam UUID traceId
) { ... }
```

### @DateTimeFormat

```java
public class UserForm {
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private LocalDate birthDate;
    
    @DateTimeFormat(iso = ISO.DATE_TIME)
    private LocalDateTime createdAt;
}
```

### @NumberFormat

```java
public class Product {
    @NumberFormat(pattern = "#,###.##")
    private BigDecimal price;
}
```

### Custom Converter

```java
@Component
public class StringToMoneyConverter implements Converter<String, Money> {
    @Override
    public Money convert(String source) {
        return Money.parse(source);   // "USD 10.00"
    }
}
```

### Custom Formatter (locale-aware)

```java
@Component
public class MoneyFormatter implements Formatter<Money> {
    @Override
    public Money parse(String text, Locale locale) { return Money.parse(text); }
    @Override
    public String print(Money object, Locale locale) { return object.toString(); }
}
```

### Custom Editor (legacy)

```java
@InitBinder
public void initBinder(WebDataBinder binder) {
    binder.registerCustomEditor(Money.class, new MoneyEditor());
}
```

### JSON with Jackson

```java
public class User {
    private Long id;
    
    @JsonProperty("full_name")           // rename field in JSON
    private String fullName;
    
    @JsonIgnore                          // omit from JSON
    private String password;
    
    @JsonFormat(pattern = "yyyy-MM-dd")  // date format
    private LocalDate birthDate;
    
    @JsonInclude(JsonInclude.Include.NON_NULL)
    private String nickname;
}
```

### Configure ObjectMapper

```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer jsonCustomizer() {
    return builder -> builder
        .serializationInclusion(JsonInclude.Include.NON_NULL)
        .failOnUnknownProperties(false);
}
```

Or via properties:

```properties
spring.jackson.serialization.indent-output=true
spring.jackson.default-property-inclusion=non_null
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
```

---

## 9. Validation

### Add Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### Constraints (Jakarta Bean Validation)

| Annotation | Purpose |
|-----------|---------|
| `@NotNull` | Not null |
| `@NotEmpty` | Not null and not empty (String, Collection) |
| `@NotBlank` | Not null, not empty, not whitespace |
| `@Size(min, max)` | String/collection size |
| `@Min` / `@Max` | Numeric range |
| `@Positive` / `@Negative` | Sign |
| `@Email` | Email format |
| `@Pattern(regexp)` | Regex |
| `@Past` / `@Future` | Date/time |
| `@PastOrPresent` / `@FutureOrPresent` | Inclusive |
| `@DecimalMin` / `@DecimalMax` | BigDecimal range |
| `@Digits` | Precision/scale |
| `@Null` | Must be null |
| `@AssertTrue` / `@AssertFalse` | Boolean |

### DTO Example

```java
public class CreateUserRequest {
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 50)
    private String name;
    
    @NotBlank @Email
    private String email;
    
    @NotNull @Min(18) @Max(120)
    private Integer age;
    
    @Pattern(regexp = "^\\+?[0-9]{7,15}$")
    private String phone;
    
    // getters/setters
}
```

### Controller Validation

```java
@PostMapping("/users")
public ResponseEntity<User> create(@Valid @RequestBody CreateUserRequest req) {
    return ResponseEntity.ok(userService.create(req));
}
```

**On failure** → `MethodArgumentNotValidException` → 400 by default.

### Handling Validation Errors

```java
@PostMapping("/users")
public ResponseEntity<?> create(
        @Valid @RequestBody CreateUserRequest req,
        BindingResult result) {
    if (result.hasErrors()) {
        Map<String, String> errors = new HashMap<>();
        result.getFieldErrors().forEach(e -> 
            errors.put(e.getField(), e.getDefaultMessage()));
        return ResponseEntity.badRequest().body(errors);
    }
    return ResponseEntity.ok(userService.create(req));
}
```

### Validation Groups

```java
public interface OnCreate { }
public interface OnUpdate { }

public class UserDto {
    @NotNull(groups = OnUpdate.class)
    private Long id;
    
    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    private String name;
}

// Controller:
public User create(@Validated(OnCreate.class) @RequestBody UserDto dto) { ... }
public User update(@Validated(OnUpdate.class) @RequestBody UserDto dto) { ... }
```

### Custom Validator

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = StrongPasswordValidator.class)
public @interface StrongPassword {
    String message() default "Password too weak";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class StrongPasswordValidator 
        implements ConstraintValidator<StrongPassword, String> {
    @Override
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        if (value == null) return true;
        return value.length() >= 8
            && value.matches(".*[A-Z].*")
            && value.matches(".*[0-9].*");
    }
}
```

### Method-Level Validation

```java
@Service
@Validated
public class UserService {
    public User create(@Valid @NotNull CreateUserRequest req) { ... }
}
```

---

## 10. Exception Handling

### Default Behavior

- Spring returns 500 for unhandled exceptions
- 404 for `NoHandlerFoundException`
- 400 for validation errors (`MethodArgumentNotValidException`)
- 405 for method not allowed

### @ExceptionHandler (Local)

Handles exceptions for a single controller.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(UserNotFoundException ex) {
        return new ErrorResponse("USER_NOT_FOUND", ex.getMessage());
    }
    
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleBadRequest(IllegalArgumentException ex) {
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("BAD_REQUEST", ex.getMessage()));
    }
}
```

### @ControllerAdvice / @RestControllerAdvice (Global)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse notFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ValidationErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> fields = ex.getBindingResult().getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                FieldError::getDefaultMessage,
                (a, b) -> a + "; " + b));
        return new ValidationErrorResponse("VALIDATION_FAILED", fields);
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse generic(Exception ex) {
        return new ErrorResponse("INTERNAL_ERROR", "Please try later");
    }
}
```

### ResponseStatusException (Spring 5+)

```java
@GetMapping("/users/{id}")
public User get(@PathVariable Long id) {
    return userService.findById(id)
        .orElseThrow(() -> new ResponseStatusException(
            HttpStatus.NOT_FOUND, "User not found"));
}
```

### Problem Details (RFC 7807) — Spring 6

```properties
spring.mvc.problemdetails.enabled=true
```

Auto-formats errors as `application/problem+json`.

### Handler Resolution Order

1. `@ExceptionHandler` in the **same controller**
2. `@ExceptionHandler` in `@ControllerAdvice` (ordered by `@Order`)
3. `ResponseStatusExceptionResolver` (`@ResponseStatus`)
4. `DefaultHandlerExceptionResolver` (Spring's own)
5. Container's error handling

### @ControllerAdvice Selectors

```java
@ControllerAdvice(assignableTypes = UserController.class)
@ControllerAdvice(basePackages = "com.example.api")
@ControllerAdvice(annotations = RestController.class)
```

---

## 11. Interceptors

Implement `HandlerInterceptor` to run logic around controller invocation.

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {
    private static final Logger log = LoggerFactory.getLogger(LoggingInterceptor.class);
    
    @Override
    public boolean preHandle(HttpServletRequest req,
                             HttpServletResponse res,
                             Object handler) throws Exception {
        log.info("→ {} {}", req.getMethod(), req.getRequestURI());
        req.setAttribute("startTime", System.currentTimeMillis());
        return true;   // return false to stop the chain
    }
    
    @Override
    public void postHandle(HttpServletRequest req,
                           HttpServletResponse res,
                           Object handler,
                           ModelAndView mv) throws Exception {
        log.info("← post handle");
    }
    
    @Override
    public void afterCompletion(HttpServletRequest req,
                                HttpServletResponse res,
                                Object handler,
                                Exception ex) throws Exception {
        long start = (Long) req.getAttribute("startTime");
        log.info("← done in {} ms", System.currentTimeMillis() - start);
    }
}
```

### Registration

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private LoggingInterceptor loggingInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loggingInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/health", "/api/public/**")
                .order(1);
    }
}
```

### Method Summary

| Method | When |
|--------|------|
| `preHandle` | Before controller method. Return `false` to abort. |
| `postHandle` | After controller, before view render. |
| `afterCompletion` | After view render, in all cases. |

---

## 12. Filters vs Interceptors

| Aspect | Filter | Interceptor |
|--------|--------|------------|
| Spec | Servlet API | Spring MVC |
| Scope | All requests (any servlet) | Only DispatcherServlet-handled |
| Access to Spring beans | Limited (needs DI hack) | Full |
| Access to handler | ❌ | ✅ |
| Access to ModelAndView | ❌ | ✅ |
| Access to request/response | ✅ | ✅ |
| Ordering | `@Order` or XML | `order()` in registry |
| Typical use | Encoding, CORS, security | Auth, logging, MDC |

### Filter Example

```java
@Component
@Order(1)
public class RequestIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws IOException, ServletException {
        String id = req.getHeader("X-Request-Id");
        if (id == null) id = UUID.randomUUID().toString();
        MDC.put("requestId", id);
        res.setHeader("X-Request-Id", id);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("requestId");
        }
    }
}
```

### Registration via FilterRegistrationBean

```java
@Bean
public FilterRegistrationBean<RequestIdFilter> requestIdFilter() {
    FilterRegistrationBean<RequestIdFilter> reg = new FilterRegistrationBean<>();
    reg.setFilter(new RequestIdFilter());
    reg.addUrlPatterns("/*");
    reg.setOrder(Ordered.HIGHEST_PRECEDENCE);
    return reg;
}
```

### Which to Choose?

- **Filter** — needs to run before/after DispatcherServlet, wrap request/response, work at servlet level
- **Interceptor** — needs access to handler method, Spring beans, controller metadata

---

## 13. View Resolvers

For server-side rendering (JSP, Thymeleaf, FreeMarker), the logical view name must be resolved to an actual `View`.

### Common View Resolvers

| Resolver | Purpose |
|----------|---------|
| `InternalResourceViewResolver` | JSP / HTML |
| `ThymeleafViewResolver` | Thymeleaf |
| `FreeMarkerViewResolver` | FreeMarker |
| `ContentNegotiatingViewResolver` | Chooses by content type |
| `BeanNameViewResolver` | View bean by name |
| `XmlViewResolver` | Views from XML |

### InternalResourceViewResolver (JSP)

```java
@Bean
public ViewResolver viewResolver() {
    InternalResourceViewResolver r = new InternalResourceViewResolver();
    r.setPrefix("/WEB-INF/views/");
    r.setSuffix(".jsp");
    return r;
}
```

**Controller:**

```java
@GetMapping("/users")
public String list(Model model) {
    model.addAttribute("users", userService.findAll());
    return "users/list";  // → /WEB-INF/views/users/list.jsp
}
```

### Thymeleaf (Spring Boot default)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

```properties
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html
spring.thymeleaf.mode=HTML
spring.thymeleaf.cache=false
```

**Template:**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><title>Users</title></head>
<body>
<ul>
    <li th:each="u : ${users}" th:text="${u.name}"></li>
</ul>
</body>
</html>
```

---

## 14. Content Negotiation

Spring selects the response format based on:

1. `Accept` header
2. URL extension (`.json`, `.xml`) — deprecated in Spring 5.3
3. Query param (`?format=json`)
4. Default (`application/json`)

### Configure Negotiation

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer config) {
        config
            .favorParameter(true)
            .parameterName("format")
            .ignoreAcceptHeader(false)
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML);
    }
}
```

### Produces Attribute

```java
@GetMapping(value = "/users", produces = MediaType.APPLICATION_JSON_VALUE)
public List<User> usersJson() { ... }

@GetMapping(value = "/users", produces = MediaType.APPLICATION_XML_VALUE)
public List<User> usersXml() { ... }
```

### Consumes Attribute

```java
@PostMapping(value = "/users", consumes = MediaType.APPLICATION_JSON_VALUE)
public User createJson(@RequestBody User u) { ... }

@PostMapping(value = "/users", consumes = MediaType.APPLICATION_XML_VALUE)
public User createXml(@RequestBody User u) { ... }
```

---

## 15. REST Principles & HATEOAS

### Richardson Maturity Model

| Level | Characteristics |
|-------|-----------------|
| 0 | POX (plain old XML) — single endpoint |
| 1 | Resources — multiple URIs |
| 2 | HTTP verbs + status codes |
| 3 | Hypermedia (HATEOAS) |

### REST Best Practices

- **Nouns, not verbs** in URIs: `/users/123`, not `/getUser?id=123`
- **Plural resource names**: `/orders`, not `/order`
- **HTTP methods** for operations: GET (read), POST (create), PUT (replace), PATCH (partial), DELETE
- **Status codes**: 200, 201, 204, 400, 401, 403, 404, 409, 422, 500
- **Versioning**: `/api/v1/users` or `Accept: application/vnd.api.v1+json`
- **Pagination**: `?page=0&size=20`
- **Filtering/Sorting**: `?status=ACTIVE&sort=name,asc`
- **Idempotency** for PUT/DELETE
- **Consistent error format** (RFC 7807)

### HATEOAS

Add links to related resources in the response.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

```java
@GetMapping("/users/{id}")
public EntityModel<User> get(@PathVariable Long id) {
    User user = userService.findById(id).orElseThrow();
    return EntityModel.of(user,
        linkTo(methodOn(UserController.class).get(id)).withSelfRel(),
        linkTo(methodOn(UserController.class).all()).withRel("users"));
}
```

**Response:**

```json
{
  "id": 1,
  "name": "Alice",
  "_links": {
    "self": { "href": "http://localhost/users/1" },
    "users": { "href": "http://localhost/users" }
  }
}
```

### CollectionModel

```java
@GetMapping("/users")
public CollectionModel<EntityModel<User>> all() {
    List<EntityModel<User>> users = userService.findAll().stream()
        .map(u -> EntityModel.of(u,
            linkTo(methodOn(UserController.class).get(u.getId())).withSelfRel()))
        .toList();
    return CollectionModel.of(users,
        linkTo(methodOn(UserController.class).all()).withSelfRel());
}
```

---

## 16. File Upload/Download

### Upload — MultipartFile

```java
@PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file,
                                     @RequestParam(value = "desc", required = false) String desc)
        throws IOException {
    if (file.isEmpty()) {
        return ResponseEntity.badRequest().body("Empty file");
    }
    Path dest = Paths.get("/uploads", file.getOriginalFilename());
    Files.copy(file.getInputStream(), dest, StandardCopyOption.REPLACE_EXISTING);
    return ResponseEntity.ok("Uploaded: " + dest);
}
```

### Multiple Files

```java
@PostMapping("/upload-multi")
public String uploadMulti(@RequestParam("files") MultipartFile[] files) { ... }
```

### Binding Multipart to Object

```java
public class UploadForm {
    private MultipartFile file;
    private String description;
}
```

### Configuration

```properties
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=50MB
spring.servlet.multipart.enabled=true
```

### Download

```java
@GetMapping("/download/{name}")
public ResponseEntity<Resource> download(@PathVariable String name) throws IOException {
    Path path = Paths.get("/uploads").resolve(name);
    Resource resource = new UrlResource(path.toUri());
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"" + resource.getFilename() + "\"")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(resource);
}
```

### Streaming Download (large files)

```java
@GetMapping("/stream/{name}")
public StreamingResponseBody stream(@PathVariable String name) {
    return outputStream -> {
        try (InputStream in = Files.newInputStream(Paths.get("/uploads", name))) {
            in.transferTo(outputStream);
        }
    };
}
```

---

## 17. Async Request Processing

### Callable<T>

```java
@GetMapping("/async")
public Callable<String> async() {
    return () -> {
        Thread.sleep(2000);
        return "Done";
    };
}
```

Spring runs this in a `TaskExecutor` and re-dispatches to `DispatcherServlet` on completion.

### DeferredResult<T>

```java
@GetMapping("/deferred")
public DeferredResult<String> deferred() {
    DeferredResult<String> result = new DeferredResult<>(5000L);
    executor.submit(() -> result.setResult("Ready"));
    return result;
}
```

### CompletableFuture<T> (Spring 4.2+)

```java
@GetMapping("/future")
public CompletableFuture<String> future() {
    return CompletableFuture.supplyAsync(() -> "Done", executor);
}
```

### Server-Sent Events (SSE)

```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter events() {
    SseEmitter emitter = new SseEmitter();
    executor.execute(() -> {
        try {
            for (int i = 0; i < 10; i++) {
                emitter.send(SseEmitter.event().name("tick").data("tick " + i));
                Thread.sleep(1000);
            }
            emitter.complete();
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    });
    return emitter;
}
```

### StreamingResponseBody

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_PLAIN_VALUE)
public StreamingResponseBody stream() {
    return outputStream -> {
        for (int i = 0; i < 100; i++) {
            outputStream.write(("line " + i + "\n").getBytes());
            outputStream.flush();
        }
    };
}
```

### Async Configuration

```java
@Configuration
@EnableAsync
public class AsyncConfig implements WebMvcConfigurer {
    
    @Override
    public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
        configurer.setDefaultTimeout(30_000);
        configurer.setTaskExecutor(taskExecutor());
    }
    
    @Bean
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(10);
        ex.setMaxPoolSize(50);
        ex.setQueueCapacity(200);
        ex.setThreadNamePrefix("async-");
        ex.initialize();
        return ex;
    }
}
```

---

## 18. Interview Questions

### Q1. What is DispatcherServlet?

**Answer:** The **Front Controller** of Spring MVC. It receives every HTTP request, finds the matching handler via `HandlerMapping`, invokes it via `HandlerAdapter`, resolves views or bodies, and handles exceptions. Configured at `/` by default in Spring Boot.

### Q2. Explain the Spring MVC request flow.

**Answer:** Request → `DispatcherServlet` → `HandlerMapping` (find controller + interceptors) → interceptors' `preHandle` → `HandlerAdapter` invokes controller (argument resolvers bind params) → controller returns value → `ReturnValueHandler` processes it → view resolved by `ViewResolver` OR body serialized by `HttpMessageConverter` → interceptors' `postHandle` → response rendered → interceptors' `afterCompletion`.

### Q3. Difference between @Controller and @RestController?

**Answer:** `@RestController` = `@Controller` + `@ResponseBody`. In `@RestController`, every method's return value is serialized to the response body. In `@Controller`, `String` return values are treated as **view names**, and you must add `@ResponseBody` per method to serialize.

### Q4. What is the difference between @RequestParam and @PathVariable?

**Answer:** `@RequestParam` binds **query string / form parameters** (`?key=value`). `@PathVariable` binds **URI template variables** (`/users/{id}`).

### Q5. What does @RequestBody do?

**Answer:** Deserializes the HTTP request body into a Java object using an `HttpMessageConverter` (Jackson for JSON, JAXB for XML). Content type determines converter.

### Q6. How do you handle exceptions globally?

**Answer:** Use `@RestControllerAdvice` (or `@ControllerAdvice`) with `@ExceptionHandler` methods. Return `ResponseEntity` or annotate with `@ResponseStatus`. Spring 6 supports RFC 7807 Problem Details via `spring.mvc.problemdetails.enabled=true`.

### Q7. What is @ControllerAdvice?

**Answer:** A global component that applies `@ExceptionHandler`, `@InitBinder`, and `@ModelAttribute` methods to **all** controllers (or filtered by `basePackages`, `annotations`, `assignableTypes`).

### Q8. How do interceptors differ from filters?

**Answer:** Filters are part of the Servlet spec, apply to all requests, and lack access to Spring context/handler. Interceptors are Spring MVC components, run only for DispatcherServlet-handled requests, and can access the handler method, model, and Spring beans. Interceptors have `preHandle/postHandle/afterCompletion`.

### Q9. What is content negotiation?

**Answer:** Selecting a response format based on `Accept` header, URL extension, or query param. Spring chooses an `HttpMessageConverter` matching the resolved media type (JSON, XML, etc.).

### Q10. How do you validate request bodies?

**Answer:** Add `spring-boot-starter-validation`, annotate DTO fields with `@NotNull`, `@Size`, `@Email`, etc., and use `@Valid @RequestBody DTO` in the controller. Invalid requests raise `MethodArgumentNotValidException` (400). Handle in `@ControllerAdvice`.

### Q11. What is @ModelAttribute?

**Answer:** Binds request parameters to a Java object (form-backing bean). Can be used on method params to receive bound data, or on a method to pre-populate model attributes before handler execution.

### Q12. Explain ResponseEntity.

**Answer:** Wrapper for HTTP response with full control: status, headers, and body. Use `ResponseEntity.ok(body)`, `.status(...).body(...)`, `.created(uri)`, `.noContent()`, etc.

### Q13. What is the role of HttpMessageConverter?

**Answer:** Converts between Java objects and HTTP request/response bodies. `MappingJackson2HttpMessageConverter` for JSON, `Jaxb2RootElementHttpMessageConverter` for XML, `StringHttpMessageConverter`, etc. Registered automatically by Spring Boot.

### Q14. How does file upload work?

**Answer:** Add multipart config (Spring Boot auto-configures), inject `MultipartFile` in the controller. Configure size limits via `spring.servlet.multipart.max-file-size` and `max-request-size`.

### Q15. Difference between @RequestMapping(method=...) and @GetMapping?

**Answer:** `@GetMapping` is a shortcut for `@RequestMapping(method=GET)`. Same for `@PostMapping`, `@PutMapping`, etc. Prefer the shortcut — more readable and self-documenting.

### Q16. How do you serve JSON vs XML for the same endpoint?

**Answer:** Two `@GetMapping` methods with different `produces`, or use content negotiation (`Accept` header). With Jackson XML (`jackson-dataformat-xml`), Spring auto-converts to XML when requested.

### Q17. What are async request processing options?

**Answer:** `Callable<T>` (returns value asynchronously), `DeferredResult<T>` (complete later), `CompletableFuture<T>` (Java 8+), `SseEmitter` (SSE), `StreamingResponseBody` (streaming). Configure via `WebMvcConfigurer.configureAsyncSupport`.

### Q18. What is a ViewResolver?

**Answer:** Maps a logical view name (returned by controller) to an actual `View`. Common: `InternalResourceViewResolver` (JSP), `ThymeleafViewResolver`. In Spring Boot, Thymeleaf is the default for HTML.

### Q19. What is @ResponseStatus?

**Answer:** Sets the HTTP status code for a controller method's response or for a custom exception. Example: `@ResponseStatus(HttpStatus.CREATED)` on a POST method.

### Q20. What is the difference between @RequestParam(required=false) and defaultValue?

**Answer:** `required=false` makes the param optional but passes `null` if absent. `defaultValue="x"` provides a default when absent. Use `defaultValue` for primitives (avoids NPE on autoboxing).

### Q21. How to add CORS to a Spring Boot app?

**Answer:** Three ways:
1. `@CrossOrigin` on controller/method
2. `WebMvcConfigurer.addCorsMappings(CorsRegistry)` globally
3. `CorsFilter` bean

**Example:**

```java
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/api/**")
        .allowedOrigins("https://example.com")
        .allowedMethods("GET","POST","PUT","DELETE")
        .allowedHeaders("*")
        .allowCredentials(true)
        .maxAge(3600);
}
```

### Q22. What is HATEOAS?

**Answer:** Hypermedia as the Engine of Application State. REST responses include links to related actions/resources, allowing clients to navigate the API dynamically. Spring provides `spring-boot-starter-hateoas` with `EntityModel`, `CollectionModel`, `linkTo(methodOn(...))`.

### Q23. How to version a REST API in Spring?

**Answer:** Options:
1. **URI versioning** — `/api/v1/users` (common, easy)
2. **Header versioning** — `Accept: application/vnd.api.v1+json`
3. **Query param** — `?version=1`
4. **Custom header** — `X-API-Version: 1`

Use `produces` with media types for header versioning.

### Q24. What is @InitBinder?

**Answer:** Method-level annotation in a controller (or `@ControllerAdvice`) that customizes `WebDataBinder` before binding — registers custom editors, formatters, or validators. Example: register a `CustomDateEditor` for date strings.

### Q25. What happens if no handler matches a URL?

**Answer:** Spring returns 404. With `spring.mvc.throw-exception-if-no-handler-found=true` (and `spring.web.resources.add-mappings=false`), a `NoHandlerFoundException` is thrown and can be handled in `@ControllerAdvice`.

---

## 19. Cheat Sheet

### Request Annotations

| Annotation | Binds |
|-----------|-------|
| `@PathVariable` | URI template |
| `@RequestParam` | Query/form |
| `@RequestBody` | Body |
| `@RequestHeader` | Header |
| `@CookieValue` | Cookie |
| `@ModelAttribute` | Form → bean |
| `@SessionAttribute` | Session |
| `@RequestAttribute` | Request attr |
| `@MatrixVariable` | Matrix param |

### HTTP Method Mappings

```java
@GetMapping    @PostMapping    @PutMapping
@PatchMapping  @DeleteMapping  @RequestMapping
```

### ResponseEntity Factories

```java
ResponseEntity.ok(body)
ResponseEntity.created(uri).body(body)
ResponseEntity.accepted().build()
ResponseEntity.noContent().build()
ResponseEntity.badRequest().body(err)
ResponseEntity.notFound().build()
ResponseEntity.status(HttpStatus.CONFLICT).body(err)
```

### Validation Cheat

```java
@NotNull @NotEmpty @NotBlank
@Size(min,max) @Min @Max @Positive @Negative
@Email @Pattern(regexp) @Past @Future
@Valid (cascade) @Validated(groups)
```

### Common Status Codes

| Code | Meaning |
|------|---------|
| 200 | OK |
| 201 | Created |
| 204 | No Content |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 405 | Method Not Allowed |
| 409 | Conflict |
| 415 | Unsupported Media Type |
| 422 | Unprocessable Entity |
| 500 | Internal Server Error |

### Spring Boot MVC Properties

```properties
server.port=8080
spring.mvc.servlet.path=/
spring.mvc.throw-exception-if-no-handler-found=true
spring.mvc.problemdetails.enabled=true
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=50MB
spring.jackson.serialization.indent-output=true
spring.jackson.default-property-inclusion=non_null
```

### Cross-References

- **Previous:** `02_Spring_AOP.md`
- **Next:** `04_Spring_JDBC.md`
- **Related:** `10_Spring_Data_JPA.md`, `23_Spring_Web_Services.md`
- **Interview:** `24_Spring_Interview_Questions.md` (§Spring MVC)

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [02_Spring_AOP.md](./02_Spring_AOP.md)
- **Next →:** [04_Spring_JDBC.md](./04_Spring_JDBC.md)
- **Related:** [06_Spring_Boot_Fundamentals.md](./06_Spring_Boot_Fundamentals.md), [20_Spring_WebFlux_Reactive.md](./20_Spring_WebFlux_Reactive.md), [23_Spring_Web_Services.md](./23_Spring_Web_Services.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
