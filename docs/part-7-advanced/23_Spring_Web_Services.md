# SOAP & REST Web Services

> **File:** `23_Spring_Web_Services.md`
> **Part:** 7 — Advanced Topics
> **Prerequisites:** Spring MVC, HTTP fundamentals, XML/JSON
> **Estimated Study Time:** 8–10 hours (read + code + practice)

---

## Table of Contents

1. [SOAP Web Services with Spring WS](#1-soap-web-services-with-spring-ws)
2. [REST Best Practices](#2-rest-best-practices)
3. [HATEOAS with Spring HATEOAS](#3-hateoas-with-spring-hateoas)
4. [Content Negotiation](#4-content-negotiation)
5. [API Versioning Strategies](#5-api-versioning-strategies)
6. [OpenAPI/Swagger with SpringDoc](#6-openapiswagger-with-springdoc)
7. [REST Exception Handling Best Practices](#7-rest-exception-handling-best-practices)
8. [Rate Limiting & Throttling](#8-rate-limiting--throttling)
9. [API Security Best Practices](#9-api-security-best-practices)
10. [Interview Questions](#10-interview-questions)
11. [Summary Cheat Sheet](#11-summary-cheat-sheet)

---

## 1. SOAP Web Services with Spring WS

### What is SOAP?

**SOAP** (Simple Object Access Protocol) is a **protocol** for exchanging structured information via XML over HTTP (or other transports). It's contract-first, strongly typed, and WS-* compliant.

### SOAP vs REST

| Aspect | SOAP | REST |
|--------|------|------|
| Protocol | Strict protocol | Architectural style |
| Message format | XML only | JSON, XML, others |
| Contract | WSDL | OpenAPI (optional) |
| Transport | HTTP, SMTP, JMS | HTTP |
| Standards | WS-Security, WS-Addressing | OAuth2, JWT |
| Performance | Heavier | Lighter |
| Use cases | Enterprise, B2B, banking | Public APIs, web/mobile |

### Spring Web Services (Spring WS)

**Contract-first** SOAP framework.

**Maven dependency:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web-services</artifactId>
</dependency>
<dependency>
    <groupId>wsdl4j</groupId>
    <artifactId>wsdl4j</artifactId>
</dependency>
```

### Contract-First Workflow

```
1. Write XSD (schema)
2. Generate Java classes with JAXB (jaxb2-maven-plugin)
3. Implement Endpoint
4. Configure WSDL + servlet
5. Test with SoapUI
```

### XSD Example

```xml
<!-- order.xsd -->
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           targetNamespace="http://example.com/orders"
           xmlns:tns="http://example.com/orders"
           elementFormDefault="qualified">
    
    <xs:element name="getOrderRequest">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="orderId" type="xs:long"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>
    
    <xs:element name="getOrderResponse">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="order" type="tns:Order"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>
    
    <xs:complexType name="Order">
        <xs:sequence>
            <xs:element name="id" type="xs:long"/>
            <xs:element name="customer" type="xs:string"/>
            <xs:element name="total" type="xs:decimal"/>
        </xs:sequence>
    </xs:complexType>
</xs:schema>
```

### Generating Java Classes

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>jaxb2-maven-plugin</artifactId>
    <version>3.1.0</version>
    <executions>
        <execution>
            <id>xjc</id>
            <goals><goal>xjc</goal></goals>
        </execution>
    </executions>
    <configuration>
        <schemaDirectory>src/main/resources/xsd</schemaDirectory>
        <outputDirectory>target/generated-sources</outputDirectory>
        <packageName>com.example.orders</packageName>
    </configuration>
</plugin>
```

### Endpoint Implementation

```java
@Endpoint
public class OrderEndpoint {
    
    private static final String NAMESPACE = "http://example.com/orders";
    
    private final OrderService orderService;
    
    public OrderEndpoint(OrderService orderService) {
        this.orderService = orderService;
    }
    
    @PayloadRoot(namespace = NAMESPACE, localPart = "getOrderRequest")
    @ResponsePayload
    public GetOrderResponse getOrder(@RequestPayload GetOrderRequest request) {
        Order order = orderService.findById(request.getOrderId());
        GetOrderResponse response = new GetOrderResponse();
        response.setOrder(order);
        return response;
    }
}
```

### Configuration

```java
@Configuration
@EnableWs
public class WebServiceConfig extends WsConfigurerAdapter {
    
    @Bean
    public ServletRegistrationBean<MessageDispatcherServlet> messageDispatcherServlet(
            ApplicationContext ctx) {
        MessageDispatcherServlet servlet = new MessageDispatcherServlet();
        servlet.setApplicationContext(ctx);
        servlet.setTransformWsdlLocations(true);
        return new ServletRegistrationBean<>(servlet, "/ws/*");
    }
    
    @Bean(name = "orders")
    public DefaultWsdl11Definition ordersWsdl(XsdSchema ordersSchema) {
        DefaultWsdl11Definition wsdl = new DefaultWsdl11Definition();
        wsdl.setPortTypeName("OrdersPort");
        wsdl.setLocationUri("/ws");
        wsdl.setTargetNamespace("http://example.com/orders");
        wsdl.setSchema(ordersSchema);
        return wsdl;
    }
    
    @Bean
    public XsdSchema ordersSchema() {
        return new SimpleXsdSchema(new ClassPathResource("xsd/order.xsd"));
    }
}
```

**Access WSDL:** `http://localhost:8080/ws/orders.wsdl`

### Client Side

```java
@Configuration
public class ClientConfig {
    
    @Bean
    public Jaxb2Marshaller marshaller() {
        Jaxb2Marshaller m = new Jaxb2Marshaller();
        m.setContextPath("com.example.orders");
        return m;
    }
    
    @Bean
    public WebServiceTemplate webServiceTemplate(Jaxb2Marshaller marshaller) {
        WebServiceTemplate t = new WebServiceTemplate();
        t.setMarshaller(marshaller);
        t.setUnmarshaller(marshaller);
        t.setDefaultUri("http://localhost:8080/ws");
        return t;
    }
}

// Usage
GetOrderResponse response = (GetOrderResponse) webServiceTemplate
    .marshalSendAndReceive(new GetOrderRequest(1L));
```

### Interceptors (Security, Logging)

```java
@Component
public class LoggingInterceptor implements ClientInterceptor {
    
    @Override
    public boolean handleRequest(MessageContext ctx) {
        log.info("Sending SOAP request");
        return true;
    }
    
    @Override
    public boolean handleResponse(MessageContext ctx) { return true; }
    
    @Override
    public boolean handleFault(MessageContext ctx) { return true; }
    
    @Override
    public void afterCompletion(MessageContext ctx, Exception e) { }
}
```

### WS-Security

```java
@Bean
public Wss4jSecurityInterceptor securityInterceptor() {
    Wss4jSecurityInterceptor i = new Wss4jSecurityInterceptor();
    i.setSecurementActions("UsernameToken Timestamp");
    i.setSecurementUsername("admin");
    i.setSecurementPassword("secret");
    i.setSecurementPasswordType("PasswordText");
    return i;
}
```

---

## 2. REST Best Practices

### REST Maturity Model (Richardson)

| Level | Description |
|-------|-------------|
| 0 | POX (Plain Old XML) — single endpoint |
| 1 | Resources — multiple URIs |
| 2 | HTTP verbs + status codes |
| 3 | HATEOAS — hypermedia controls |

### Resource Naming

✅ **Good:**
```
GET    /users
GET    /users/123
POST   /users
PUT    /users/123
PATCH  /users/123
DELETE /users/123
GET    /users/123/orders
GET    /users/123/orders/456
```

❌ **Bad:**
```
GET  /getUsers
POST /createUser
POST /deleteUser?id=123
GET  /user_list
```

**Rules:**
- Use **nouns**, not verbs
- Plural resource names (`/users`, not `/user`)
- Lowercase, hyphen-separated (`/order-items`)
- Nest for relationships (`/users/123/orders`)
- Avoid deep nesting (>2 levels)
- No file extensions (`/users.json` ❌)

### HTTP Methods (Idempotency)

| Method | Purpose | Idempotent | Safe |
|--------|---------|-----------|------|
| GET | Read | ✅ | ✅ |
| POST | Create | ❌ | ❌ |
| PUT | Replace | ✅ | ❌ |
| PATCH | Partial update | ⚠️ (often) | ❌ |
| DELETE | Remove | ✅ | ❌ |
| HEAD | Headers only | ✅ | ✅ |
| OPTIONS | Capabilities | ✅ | ✅ |

### HTTP Status Codes

| Code | Meaning | When |
|------|---------|------|
| **200** | OK | Successful GET/PUT/PATCH |
| **201** | Created | POST created resource (with `Location` header) |
| **202** | Accepted | Async operation started |
| **204** | No Content | Successful DELETE/PUT with no body |
| **301** | Moved Permanently | Resource moved |
| **304** | Not Modified | Conditional GET |
| **400** | Bad Request | Validation error |
| **401** | Unauthorized | Missing/invalid auth |
| **403** | Forbidden | Authenticated but no permission |
| **404** | Not Found | Resource missing |
| **405** | Method Not Allowed | Wrong HTTP verb |
| **409** | Conflict | Concurrent modification, duplicate |
| **412** | Precondition Failed | ETag mismatch |
| **415** | Unsupported Media Type | Wrong Content-Type |
| **422** | Unprocessable Entity | Semantic validation error |
| **429** | Too Many Requests | Rate limited |
| **500** | Internal Server Error | Unhandled exception |
| **502** | Bad Gateway | Upstream failure |
| **503** | Service Unavailable | Maintenance/overload |
| **504** | Gateway Timeout | Upstream timeout |

### Controller Example

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @GetMapping
    public Page<UserDTO> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String q) {
        return userService.findAll(PageRequest.of(page, size), q);
    }
    
    @GetMapping("/{id}")
    public UserDTO get(@PathVariable Long id) {
        return userService.findById(id);
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<UserDTO> create(
            @Valid @RequestBody CreateUserRequest req,
            UriComponentsBuilder uriBuilder) {
        UserDTO user = userService.create(req);
        URI location = uriBuilder.path("/api/v1/users/{id}")
            .buildAndExpand(user.getId()).toUri();
        return ResponseEntity.created(location).body(user);
    }
    
    @PutMapping("/{id}")
    public UserDTO replace(@PathVariable Long id,
                           @Valid @RequestBody UpdateUserRequest req) {
        return userService.replace(id, req);
    }
    
    @PatchMapping("/{id}")
    public UserDTO update(@PathVariable Long id,
                          @RequestBody Map<String, Object> updates) {
        return userService.partialUpdate(id, updates);
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### Pagination, Sorting, Filtering

```java
@GetMapping
public Page<UserDTO> list(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "id,asc") String[] sort,
        @RequestParam(required = false) String name,
        @RequestParam(required = false) Integer minAge) {
    Sort sorting = Sort.by(Arrays.stream(sort)
        .map(s -> {
            String[] parts = s.split(",");
            return new Sort.Order(
                parts.length > 1 && parts[1].equalsIgnoreCase("desc")
                    ? Sort.Direction.DESC : Sort.Direction.ASC,
                parts[0]);
        }).toList());
    Pageable pageable = PageRequest.of(page, size, sorting);
    return userService.search(name, minAge, pageable);
}
```

**Response:**
```json
{
  "content": [ ... ],
  "page": 0,
  "size": 20,
  "totalElements": 150,
  "totalPages": 8,
  "first": true,
  "last": false
}
```

### ETag & Conditional Requests

```java
@GetMapping("/{id}")
public ResponseEntity<UserDTO> get(@PathVariable Long id,
                                   WebRequest request) {
    UserDTO user = userService.findById(id);
    String etag = "\"" + user.getVersion() + "\"";
    
    if (request.checkNotModified(etag)) {
        return null;   // 304 Not Modified
    }
    
    return ResponseEntity.ok()
        .eTag(etag)
        .body(user);
}
```

### REST Anti-Patterns

| Anti-Pattern | Better |
|--------------|--------|
| Verbs in URLs | Use HTTP methods |
| 200 for everything | Correct status codes |
| Returning entities | Return DTOs |
| Exposing DB IDs | Use UUIDs/public IDs |
| Chatty APIs | Batch endpoints |
| No pagination | Always paginate |
| Breaking changes | Version APIs |
| No rate limiting | Add throttling |
| No caching | Use `Cache-Control`, ETags |

---

## 3. HATEOAS with Spring HATEOAS

### What is HATEOAS?

**H**ypermedia **A**s **T**he **E**ngine **O**f **A**pplication **S**tate — responses include links to related resources and available actions.

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

### RepresentationModel

```java
public class UserDTO extends RepresentationModel<UserDTO> {
    private Long id;
    private String name;
    private String email;
    // getters/setters
}
```

### Adding Links

```java
@GetMapping("/{id}")
public UserDTO get(@PathVariable Long id) {
    UserDTO user = userService.findById(id);
    
    user.add(linkTo(methodOn(UserController.class).get(id)).withSelfRel());
    user.add(linkTo(methodOn(UserController.class).list(0, 20, null))
        .withRel("users"));
    user.add(linkTo(methodOn(OrderController.class).listByUser(id))
        .withRel("orders"));
    
    return user;
}
```

### CollectionModel

```java
@GetMapping
public CollectionModel<UserDTO> list() {
    List<UserDTO> users = userService.findAll();
    
    users.forEach(u -> u.add(
        linkTo(methodOn(UserController.class).get(u.getId())).withSelfRel()));
    
    return CollectionModel.of(users,
        linkTo(methodOn(UserController.class).list()).withSelfRel());
}
```

### EntityModel

```java
@GetMapping("/{id}")
public EntityModel<UserDTO> get(@PathVariable Long id) {
    UserDTO user = userService.findById(id);
    return EntityModel.of(user,
        linkTo(methodOn(UserController.class).get(id)).withSelfRel(),
        linkTo(methodOn(UserController.class).list()).withRel("users"));
}
```

### Response Example

```json
{
  "id": 1,
  "name": "Alice",
  "email": "alice@example.com",
  "_links": {
    "self": { "href": "http://localhost:8080/api/v1/users/1" },
    "users": { "href": "http://localhost:8080/api/v1/users" },
    "orders": { "href": "http://localhost:8080/api/v1/users/1/orders" }
  }
}
```

### HAL Format

Spring HATEOAS uses **HAL** (Hypertext Application Language) by default. `Accept: application/hal+json`.

### MediaType Configuration

```properties
spring.hateoas.use-hal-as-default-json-media-type=true
```

### When to Use HATEOAS

✅ **Good for:**
- Public APIs where discoverability matters
- Evolving APIs (clients follow links)
- Rich hypermedia contracts

❌ **Not for:**
- Simple internal APIs
- High-performance scenarios (overhead)
- Mobile clients that hardcode URLs

---

## 4. Content Negotiation

### How It Works

Client sends `Accept` header; server responds in the requested format.

```http
GET /api/users/1
Accept: application/json
```

**Server:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id":1,"name":"Alice"}
```

### ContentNegotiationConfigurer

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer config) {
        config
            .favorParameter(true)         // ?format=json
            .parameterName("format")
            .ignoreAcceptHeader(false)
            .useRegisteredExtensionsOnly(false)
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML);
    }
}
```

### Producing XML & JSON

```java
@GetMapping(value = "/{id}",
            produces = {MediaType.APPLICATION_JSON_VALUE, 
                        MediaType.APPLICATION_XML_VALUE})
public UserDTO get(@PathVariable Long id) {
    return userService.findById(id);
}
```

**Dependency for XML:**

```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

### Custom Media Types

```java
public static final String APPLICATION_V1_JSON = 
    "application/vnd.example.v1+json";

@GetMapping(produces = APPLICATION_V1_JSON)
public UserV1DTO getV1(@PathVariable Long id) {
    return userService.findV1(id);
}
```

### Consumes

```java
@PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
public UserDTO create(@RequestBody CreateUserRequest req) { ... }
```

Returns **415 Unsupported Media Type** if `Content-Type` mismatched.

### Content Negotiation Precedence

1. `produces` attribute
2. `Accept` header (if not ignored)
3. URL parameter (`?format=json`)
4. Extension (`.json`)
5. Default content type

---

## 5. API Versioning Strategies

### 4 Main Strategies

#### 5.1 URI Versioning (Most Common)

```
/api/v1/users
/api/v2/users
```

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserV1Controller { }

@RestController
@RequestMapping("/api/v2/users")
public class UserV2Controller { }
```

**Pros:** Explicit, easy to route, cache-friendly
**Cons:** URI changes, "not RESTful" (arguable)

#### 5.2 Request Parameter Versioning

```
/api/users?version=1
/api/users?version=2
```

```java
@GetMapping(params = "version=1")
public UserV1DTO getV1(@PathVariable Long id) { ... }

@GetMapping(params = "version=2")
public UserV2DTO getV2(@PathVariable Long id) { ... }
```

**Pros:** No URI change
**Cons:** Ugly URLs, easy to forget

#### 5.3 Custom Header Versioning

```
GET /api/users/1
X-API-Version: 1
```

```java
@GetMapping(headers = "X-API-Version=1")
public UserV1DTO getV1(@PathVariable Long id) { ... }

@GetMapping(headers = "X-API-Version=2")
public UserV2DTO getV2(@PathVariable Long id) { ... }
```

**Pros:** Clean URIs
**Cons:** Hidden from browser, harder to test

#### 5.4 Media Type Versioning (Content Negotiation)

```
Accept: application/vnd.example.v1+json
Accept: application/vnd.example.v2+json
```

```java
@GetMapping(produces = "application/vnd.example.v1+json")
public UserV1DTO getV1(@PathVariable Long id) { ... }

@GetMapping(produces = "application/vnd.example.v2+json")
public UserV2DTO getV2(@PathVariable Long id) { ... }
```

**Pros:** RESTful, uses standard mechanism
**Cons:** Less discoverable, complex

### Comparison

| Strategy | Discoverable | Cacheable | RESTful | Popularity |
|----------|-------------|-----------|---------|-----------|
| URI | ✅ | ✅ | ⚠️ | ✅✅✅ |
| Query param | ✅ | ⚠️ | ⚠️ | ✅ |
| Header | ⚠️ | ✅ | ✅ | ✅ |
| Media type | ⚠️ | ✅ | ✅✅ | ✅ |

**Recommendation:** **URI versioning** for public APIs; **media type** for purists.

### Versioning Best Practices

1. Version from day one (`/v1/`)
2. Never break a version — add new versions
3. Deprecate with `Sunset` header
4. Document version lifecycle
5. Use semantic versioning for breaking changes

```java
@GetMapping("/{id}")
@Deprecated
public ResponseEntity<UserV1DTO> getV1(@PathVariable Long id) {
    return ResponseEntity.ok()
        .header("Deprecation", "true")
        .header("Sunset", "Sat, 31 Dec 2025 23:59:59 GMT")
        .header("Link", "</api/v2/users/" + id + ">; rel=\"successor-version\"")
        .body(userService.findV1(id));
}
```

---

## 6. OpenAPI/Swagger with SpringDoc

### Dependencies

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>
```

**Access:**
- Swagger UI: `http://localhost:8080/swagger-ui.html`
- OpenAPI JSON: `http://localhost:8080/v3/api-docs`

### Configuration

```properties
springdoc.api-docs.path=/v3/api-docs
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.swagger-ui.operationsSorter=method
springdoc.swagger-ui.tagsSorter=alpha
springdoc.packages-to-scan=com.example.api
```

### Annotations

```java
@RestController
@RequestMapping("/api/v1/users")
@Tag(name = "Users", description = "User management APIs")
public class UserController {
    
    @Operation(
        summary = "Get user by ID",
        description = "Returns a single user",
        responses = {
            @ApiResponse(responseCode = "200", description = "Found",
                content = @Content(schema = @Schema(implementation = UserDTO.class))),
            @ApiResponse(responseCode = "404", description = "Not found")
        })
    @GetMapping("/{id}")
    public UserDTO get(
            @Parameter(description = "User ID", example = "1")
            @PathVariable Long id) {
        return userService.findById(id);
    }
    
    @Operation(summary = "Create user")
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserDTO create(
            @io.swagger.v3.oas.annotations.parameters.RequestBody(
                description = "User to create",
                required = true)
            @Valid @RequestBody CreateUserRequest req) {
        return userService.create(req);
    }
}
```

### DTO Documentation

```java
@Schema(description = "User data transfer object")
public class UserDTO {
    
    @Schema(description = "Unique identifier", example = "1", accessMode = READ_ONLY)
    private Long id;
    
    @Schema(description = "Full name", example = "Alice Smith", required = true)
    private String name;
    
    @Schema(description = "Email address", example = "alice@example.com")
    private String email;
    // ...
}
```

### Global OpenAPI Info

```java
@Configuration
public class OpenApiConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("My API")
                .version("1.0.0")
                .description("REST API documentation")
                .contact(new Contact().name("API Team").email("api@example.com"))
                .license(new License().name("Apache 2.0")
                    .url("https://www.apache.org/licenses/LICENSE-2.0")))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
            .components(new Components()
                .addSecuritySchemes("bearerAuth",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")));
    }
}
```

### Security Scheme

```java
@Bean
public OpenAPI openAPI() {
    return new OpenAPI()
        .components(new Components()
            .addSecuritySchemes("bearer-jwt",
                new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT"))
            .addSecuritySchemes("oauth2",
                new SecurityScheme()
                    .type(SecurityScheme.Type.OAUTH2)
                    .flows(new OAuthFlows()
                        .authorizationCode(new OAuthFlow()
                            .authorizationUrl("https://auth.example.com/oauth/authorize")
                            .tokenUrl("https://auth.example.com/oauth/token")
                            .scopes(new Scopes()
                                .addString("read", "Read access")
                                .addString("write", "Write access"))))))
        .addSecurityItem(new SecurityRequirement().addList("bearer-jwt"));
}
```

### Grouping APIs

```java
@Bean
public GroupedOpenApi publicApi() {
    return GroupedOpenApi.builder()
        .group("public")
        .pathsToMatch("/api/public/**")
        .build();
}

@Bean
public GroupedOpenApi adminApi() {
    return GroupedOpenApi.builder()
        .group("admin")
        .pathsToMatch("/api/admin/**")
        .build();
}
```

---

## 7. REST Exception Handling Best Practices

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex,
                                        HttpServletRequest req) {
        return new ErrorResponse(
            Instant.now(),
            HttpStatus.NOT_FOUND.value(),
            "Not Found",
            ex.getMessage(),
            req.getRequestURI());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex,
                                          HttpServletRequest req) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(err ->
            errors.put(err.getField(), err.getDefaultMessage()));
        
        return new ErrorResponse(
            Instant.now(),
            HttpStatus.BAD_REQUEST.value(),
            "Validation Failed",
            "Request validation failed",
            req.getRequestURI(),
            errors);
    }
    
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleConstraint(ConstraintViolationException ex,
                                          HttpServletRequest req) {
        return new ErrorResponse(/* ... */);
    }
    
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ErrorResponse handleAccessDenied(AccessDeniedException ex,
                                            HttpServletRequest req) {
        return new ErrorResponse(/* ... */);
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleAll(Exception ex, HttpServletRequest req) {
        log.error("Unhandled exception", ex);
        return new ErrorResponse(
            Instant.now(),
            500,
            "Internal Server Error",
            "An unexpected error occurred",
            req.getRequestURI());
    }
}
```

### Error Response DTO

```java
public record ErrorResponse(
    Instant timestamp,
    int status,
    String error,
    String message,
    String path,
    Map<String, String> validationErrors
) {
    public ErrorResponse(Instant ts, int s, String e, String m, String p) {
        this(ts, s, e, m, p, null);
    }
}
```

**Response:**
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "status": 400,
  "error": "Validation Failed",
  "message": "Request validation failed",
  "path": "/api/v1/users",
  "validationErrors": {
    "email": "must be a well-formed email address",
    "name": "must not be blank"
  }
}
```

### Custom Exceptions

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, Object id) {
        super(resource + " not found with id: " + id);
    }
}

public class BusinessException extends RuntimeException {
    private final String code;
    public BusinessException(String code, String message) {
        super(message);
        this.code = code;
    }
    public String getCode() { return code; }
}
```

### Problem Details (RFC 7807)

Standard error format:

```java
public class ProblemDetail {
    private URI type;
    private String title;
    private int status;
    private String detail;
    private URI instance;
}
```

**Spring 6+ native support:**

```java
@ExceptionHandler(ResourceNotFoundException.class)
public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
    ProblemDetail pd = ProblemDetail.forStatusAndDetail(
        HttpStatus.NOT_FOUND, ex.getMessage());
    pd.setTitle("Resource Not Found");
    pd.setType(URI.create("https://example.com/errors/not-found"));
    return pd;
}
```

**Response:**
```json
{
  "type": "https://example.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User not found with id: 123",
  "instance": "/api/v1/users/123"
}
```

### Exception Handling Best Practices

1. **Never leak stack traces** to clients
2. **Log server errors** with correlation IDs
3. **Use appropriate status codes**
4. **Consistent error format** across all endpoints
5. **Include correlation/trace ID** for debugging
6. **Don't swallow exceptions** silently
7. **Use RFC 7807** for standard error responses
8. **Validate early** — fail fast with 400
9. **Avoid returning 200 for errors**
10. **Document error responses** in OpenAPI

---

## 8. Rate Limiting & Throttling

### Bucket4j (Token Bucket)

**Dependency:**

```xml
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.10.1</version>
</dependency>
```

### In-Memory Rate Limiter

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    
    private Bucket createBucket() {
        Bandwidth limit = Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1)));
        return Bucket.builder().addLimit(limit).build();
    }
    
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String key = getClientKey(request);
        Bucket bucket = buckets.computeIfAbsent(key, k -> createBucket());
        
        if (bucket.tryConsume(1)) {
            return true;
        }
        
        response.setStatus(429);
        response.setHeader("Retry-After", "60");
        response.getWriter().write("{\"error\":\"Too many requests\"}");
        return false;
    }
    
    private String getClientKey(HttpServletRequest request) {
        String apiKey = request.getHeader("X-API-Key");
        return apiKey != null ? apiKey : request.getRemoteAddr();
    }
}
```

**Register:**

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired private RateLimitInterceptor rateLimitInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(rateLimitInterceptor).addPathPatterns("/api/**");
    }
}
```

### Distributed Rate Limiting (Redis)

```java
@Configuration
public class RedisRateLimitConfig {
    
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://localhost:6379");
        return Redisson.create(config);
    }
    
    @Bean
    public RRateLimiter rateLimiter(RedissonClient client) {
        RRateLimiter limiter = client.getRateLimiter("api:global");
        limiter.trySetRate(RateType.OVERALL, 1000, 1, RateIntervalUnit.MINUTES);
        return limiter;
    }
}
```

### Rate Limit Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1705320000

HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
```

### Algorithms

| Algorithm | Pros | Cons |
|-----------|------|------|
| **Fixed Window** | Simple | Burst at boundary |
| **Sliding Window** | Smooth | Memory |
| **Token Bucket** | Allows bursts | Config complexity |
| **Leaky Bucket** | Steady rate | Delays bursts |

### Best Practices

1. **Different limits per tier** (free, paid, enterprise)
2. **Return 429** with `Retry-After`
3. **Use Redis** for distributed systems
4. **Rate limit by API key** (not IP behind proxy)
5. **Exempt health checks**
6. **Document limits** in API docs
7. **Consider burst allowance**

---

## 9. API Security Best Practices

### Transport Security

- **Always HTTPS** (TLS 1.2+)
- **HSTS header**: `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- **Certificate pinning** for mobile clients
- **Disable weak ciphers**

### Authentication

| Method | Use When |
|--------|----------|
| **API Keys** | Simple, server-to-server |
| **OAuth2** | Third-party access, user delegation |
| **JWT** | Stateless auth, microservices |
| **mTLS** | High-security B2B |
| **Basic Auth** | Only over HTTPS, simple cases |

### Authorization

- **RBAC** — role-based
- **ABAC** — attribute-based
- **Scopes** (OAuth2) — permission grants
- **Ownership checks** — user can access only their data

```java
@PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
public UserDTO getUser(Long userId) { ... }
```

### Security Headers

```java
@Configuration
public class SecurityHeadersConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'"))
            .frameOptions(fo -> fo.deny())
            .xssProtection(xss -> xss.disable())  // deprecated, use CSP
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000))
            .referrerPolicy(rp -> rp
                .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.NO_REFERRER))
            .permissionsPolicy(pp -> pp
                .policy("geolocation=(), camera=()")));
        return http.build();
    }
}
```

### Input Validation

```java
public record CreateUserRequest(
    @NotBlank @Size(min = 2, max = 100) String name,
    @NotBlank @Email String email,
    @Min(18) @Max(120) int age,
    @Pattern(regexp = "^[A-Z]{2}\\d{4}$") String employeeId
) { }
```

### SQL Injection Prevention

```java
// ❌ NEVER: string concatenation
String sql = "SELECT * FROM users WHERE name = '" + name + "'";

// ✅ Parameterized query
jdbcTemplate.queryForObject(
    "SELECT * FROM users WHERE name = ?",
    userRowMapper, name);
```

### Mass Assignment Prevention

```java
// ❌ NEVER: bind entity directly
@PostMapping
public User create(@RequestBody User user) { ... }   // user can set role=ADMIN!

// ✅ Use DTO
@PostMapping
public UserDTO create(@RequestBody CreateUserRequest req) {
    User user = new User();
    user.setName(req.name());
    user.setEmail(req.email());
    // role is NOT settable
}
```

### Rate Limiting & DDoS

- Rate limit per API key / IP
- WAF (Web Application Firewall)
- CDN with DDoS protection
- Request size limits

```properties
server.tomcat.max-http-form-post-size=2MB
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
```

### CORS Configuration

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH")
            .allowedHeaders("*")
            .exposedHeaders("X-Total-Count", "Location")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

### Logging & Monitoring

- **Never log** secrets, passwords, tokens, PII
- **Log** auth failures, 4xx, 5xx, rate limits
- **Correlation IDs** across requests
- **Audit logs** for sensitive operations
- **Alert** on anomalies

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    
    private static final String HEADER = "X-Correlation-Id";
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String id = request.getHeader(HEADER);
        if (id == null) id = UUID.randomUUID().toString();
        
        MDC.put("correlationId", id);
        response.setHeader(HEADER, id);
        
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

### API Security Checklist

- [ ] HTTPS enforced (HSTS)
- [ ] Strong authentication (OAuth2/JWT)
- [ ] Authorization on every endpoint
- [ ] Input validation
- [ ] Output encoding (prevent XSS)
- [ ] Parameterized queries (prevent SQLi)
- [ ] Rate limiting
- [ ] Request size limits
- [ ] CORS restricted
- [ ] Security headers
- [ ] No sensitive data in URLs/logs
- [ ] Audit logging
- [ ] Dependency scanning (OWASP)
- [ ] Regular pentesting
- [ ] Secret management (Vault, AWS Secrets Manager)

---

## 10. Interview Questions

### Q1. SOAP vs REST — when to use which?

**Answer:** SOAP for enterprise, B2B, banking — strict contracts (WSDL), WS-Security, ACID transactions, WS-* standards. REST for public APIs, mobile/web, lightweight JSON, caching, scalability. SOAP = protocol; REST = architectural style.

### Q2. What is contract-first in Spring WS?

**Answer:** Define XSD first, generate Java classes with JAXB, then implement endpoints. Ensures the contract is the source of truth, unlike contract-last (code → WSDL). Best practice for SOAP.

### Q3. What are the REST maturity levels?

**Answer:** Level 0 — POX (single endpoint). Level 1 — Resources (multiple URIs). Level 2 — HTTP verbs + status codes (most APIs). Level 3 — HATEOAS (hypermedia). Most production APIs are level 2.

### Q4. Explain idempotency in REST.

**Answer:** An operation is idempotent if performing it multiple times yields the same result. GET, PUT, DELETE, HEAD are idempotent. POST is not (creates new resource each time). Important for retries.

### Q5. PUT vs PATCH?

**Answer:** PUT replaces the entire resource; PATCH applies partial modifications. PUT is idempotent; PATCH often is but not required. PATCH uses `application/merge-patch+json` or JSON Patch (RFC 6902).

### Q6. How do you version a REST API?

**Answer:** 1) **URI** (`/v1/users`) — most common. 2) **Query param** (`?version=1`). 3) **Header** (`X-API-Version: 1`). 4) **Media type** (`Accept: application/vnd.example.v1+json`). URI versioning is easiest; media type is most RESTful.

### Q7. What is HATEOAS?

**Answer:** Hypermedia As The Engine Of Application State — responses include links to related resources/actions. Clients navigate via links, not hardcoded URLs. Improves discoverability and evolution. Spring HATEOAS uses HAL format.

### Q8. How does content negotiation work?

**Answer:** Client sends `Accept` header (e.g., `application/json`). Server picks the best matching `produces` type. Fallback strategies: URL parameter (`?format=json`), extension (`.json`). Returns 406 if no match.

### Q9. How do you handle exceptions globally in REST?

**Answer:** Use `@RestControllerAdvice` with `@ExceptionHandler` methods. Return consistent error DTOs (or ProblemDetail/RFC 7807). Map exceptions to appropriate status codes (404, 400, 403, 500). Log server errors; never leak stack traces.

### Q10. How do you document a REST API?

**Answer:** **SpringDoc OpenAPI** (Swagger UI at `/swagger-ui.html`, JSON at `/v3/api-docs`). Use `@Operation`, `@Parameter`, `@Schema`, `@ApiResponse`. Group with `GroupedOpenApi`. Add security schemes for auth.

### Q11. What is rate limiting and how to implement it?

**Answer:** Controls request frequency per client. Algorithms: token bucket (Bucket4j), sliding window, leaky bucket. Implement via HandlerInterceptor or filter. Return 429 with `Retry-After`. Use Redis for distributed systems.

### Q12. REST API security best practices?

**Answer:** HTTPS + HSTS; OAuth2/JWT auth; authorization on every endpoint; input validation; parameterized queries; avoid mass assignment (use DTOs); rate limiting; CORS restrictions; security headers (CSP, X-Frame-Options); no secrets in logs; audit logging.

### Q13. How do you prevent mass assignment vulnerabilities?

**Answer:** Never bind entities directly from request bodies. Use **DTOs** with only allowed fields. Map DTO → entity manually or with MapStruct. Never expose `role`, `admin`, `createdAt` fields for client setting.

### Q14. What is RFC 7807?

**Answer:** "Problem Details for HTTP APIs" — standard error response format: `type`, `title`, `status`, `detail`, `instance`. Spring 6+ has `ProblemDetail` support. Improves API consistency.

### Q15. How do you test REST APIs?

**Answer:** 
- **Unit**: `@WebMvcTest` + `MockMvc` (controller slice)
- **Integration**: `@SpringBootTest` + `TestRestTemplate` / `WebTestClient`
- **Contract**: Spring Cloud Contract
- **E2E**: RestAssured, Postman/Newman
- **Load**: JMeter, Gatling, k6

### Q16. What is the difference between 401 and 403?

**Answer:** **401 Unauthorized** — authentication required or failed (missing/invalid credentials). **403 Forbidden** — authenticated but not authorized (no permission). 401 should include `WWW-Authenticate` header.

### Q17. How do you implement ETags?

**Answer:** Server computes hash/version of resource, returns it as `ETag` header. Client sends `If-None-Match` on subsequent GETs. Server returns **304 Not Modified** if unchanged. For updates, use `If-Match` to prevent lost updates.

### Q18. What are the differences between Spring WS and JAX-WS?

**Answer:** Spring WS is contract-first, Spring-native, uses `MessageDispatcherServlet`. JAX-WS is Java EE standard, code-first capable, uses servlet container. Spring WS has better Spring integration and is more testable.

### Q19. How do you handle large file uploads/downloads?

**Answer:** 
- **Upload**: `MultipartFile`, streaming, `spring.servlet.multipart.max-file-size`.
- **Download**: `StreamingResponseBody`, `ResourceRegion` for range requests.
- Avoid loading entire file into memory.
- Use chunked transfer encoding.

### Q20. What is API gateway and why use it?

**Answer:** A reverse proxy in front of microservices that handles cross-cutting concerns: routing, auth, rate limiting, caching, logging, circuit breaking. Spring Cloud Gateway is the Spring-native option. Centralizes concerns, decouples clients from service topology.

---

## 11. Summary Cheat Sheet

### SOAP with Spring WS

```java
@EnableWs
@Configuration
public class WsConfig {
    @Bean
    public ServletRegistrationBean<MessageDispatcherServlet> servlet(ApplicationContext ctx) { ... }
    
    @Bean
    public DefaultWsdl11Definition wsdl(XsdSchema schema) { ... }
    
    @Bean
    public XsdSchema schema() { return new SimpleXsdSchema(new ClassPathResource("xsd/order.xsd")); }
}

@Endpoint
public class OrderEndpoint {
    @PayloadRoot(namespace = "http://...", localPart = "getOrderRequest")
    @ResponsePayload
    public GetOrderResponse get(@RequestPayload GetOrderRequest req) { ... }
}
```

### REST Annotations

| Annotation | Purpose |
|-----------|---------|
| `@RestController` | `@Controller` + `@ResponseBody` |
| `@RequestMapping` | Base path |
| `@GetMapping` / `@PostMapping` / `@PutMapping` / `@PatchMapping` / `@DeleteMapping` | HTTP methods |
| `@PathVariable` | URI variable |
| `@RequestParam` | Query parameter |
| `@RequestBody` | Request body |
| `@RequestHeader` | HTTP header |
| `@ResponseStatus` | Response status |
| `@Valid` | Trigger validation |
| `@RestControllerAdvice` | Global exception handler |
| `@ExceptionHandler` | Handle specific exception |
| `@CrossOrigin` | CORS for endpoint |

### HTTP Status Codes

```
200 OK          | 201 Created      | 202 Accepted   | 204 No Content
301 Moved       | 304 Not Modified
400 Bad Request | 401 Unauthorized | 403 Forbidden  | 404 Not Found
405 Not Allowed | 409 Conflict     | 415 Unsupported| 422 Unprocessable
429 Too Many
500 Internal    | 502 Bad Gateway  | 503 Unavailable| 504 Timeout
```

### HATEOAS

```java
user.add(linkTo(methodOn(UserController.class).get(id)).withSelfRel());
return EntityModel.of(user, links);
return CollectionModel.of(users, links);
```

### Content Negotiation

```java
@GetMapping(produces = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE})
@PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
```

### Versioning

| Strategy | Example |
|----------|---------|
| URI | `/api/v1/users` |
| Query | `/api/users?version=1` |
| Header | `X-API-Version: 1` |
| Media type | `Accept: application/vnd.example.v1+json` |

### OpenAPI

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
</dependency>
```

- Swagger UI: `/swagger-ui.html`
- OpenAPI JSON: `/v3/api-docs`

```java
@Operation(summary = "...", description = "...")
@Parameter(description = "...")
@Schema(description = "...")
@Tag(name = "...", description = "...")
```

### Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(NotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handle(NotFoundException ex, HttpServletRequest req) { ... }
}
```

### Security Headers

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
Permissions-Policy: geolocation=(), camera=()
```

### REST Best Practices Checklist

- [ ] Nouns in URIs, verbs via HTTP methods
- [ ] Plural resource names
- [ ] Correct HTTP status codes
- [ ] Idempotent PUT/DELETE
- [ ] Pagination for collections
- [ ] Versioning from day one
- [ ] DTOs, never entities
- [ ] Validation with `@Valid`
- [ ] Global exception handling
- [ ] HATEOAS for public APIs
- [ ] OpenAPI documentation
- [ ] HTTPS + security headers
- [ ] Rate limiting
- [ ] Meaningful error messages
- [ ] Correlation IDs for tracing

---

## Cross-References

- **Previous file:** `22_Spring_Messaging.md` — Messaging
- **Next file:** `24_Spring_Interview_Questions.md` — Interview Questions
- **Related:** `03_Spring_MVC.md`, `13_Spring_Security_Core.md`, `14_Spring_Security_JWT_OAuth2.md`, `19_Spring_Microservices_Cloud.md`

---

## Practice Exercises

1. Build a SOAP endpoint with Spring WS (XSD → JAXB → Endpoint).
2. Implement a REST controller with proper status codes, pagination, and versioning.
3. Add HATEOAS links to a REST response.
4. Configure content negotiation for JSON + XML.
5. Document the API with SpringDoc + security scheme.
6. Implement global exception handling with RFC 7807.
7. Add rate limiting with Bucket4j.
8. Secure the API with JWT + security headers.

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [22_Spring_Messaging.md](./22_Spring_Messaging.md)
- **Next →:** [24_Spring_Interview_Questions.md](./24_Spring_Interview_Questions.md)
- **Related:** [03_Spring_MVC.md](./03_Spring_MVC.md), [20_Spring_WebFlux_Reactive.md](./20_Spring_WebFlux_Reactive.md), [22_Spring_Messaging.md](./22_Spring_Messaging.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
