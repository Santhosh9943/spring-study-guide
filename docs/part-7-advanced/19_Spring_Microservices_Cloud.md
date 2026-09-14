# Spring Cloud & Microservices

> **File:** `19_Spring_Microservices_Cloud.md`
> **Part:** 7 — Advanced Topics
> **Prerequisites:** `06_Spring_Boot_Fundamentals.md`, `07_Spring_Boot_Starters.md`, `08_Spring_Boot_Configuration.md`, `09_Spring_Boot_Actuator.md`
> **Estimated Study Time:** 14–18 hours

---

## Table of Contents

1. [Monolith vs Microservices](#1-monolith-vs-microservices)
2. [Microservices Design Principles](#2-microservices-design-principles)
3. [Service Discovery (Eureka)](#3-service-discovery-eureka)
4. [API Gateway](#4-api-gateway)
5. [Client-Side Load Balancing](#5-client-side-load-balancing)
6. [Declarative REST Clients (OpenFeign)](#6-declarative-rest-clients-openfeign)
7. [Circuit Breakers (Resilience4j)](#7-circuit-breakers-resilience4j)
8. [Distributed Configuration (Config Server)](#8-distributed-configuration-config-server)
9. [Distributed Tracing (Micrometer Tracing & Zipkin)](#9-distributed-tracing-micrometer-tracing--zipkin)
10. [Spring Cloud Bus](#10-spring-cloud-bus)
11. [Spring Cloud Stream](#11-spring-cloud-stream)
12. [Spring Cloud Contract](#12-spring-cloud-contract)
13. [Spring Cloud Security](#13-spring-cloud-security)
14. [Docker & Kubernetes for Spring Boot](#14-docker--kubernetes-for-spring-boot)
15. [Microservices Patterns](#15-microservices-patterns)
16. [Interview Questions](#16-interview-questions)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. Monolith vs Microservices

### Monolith

A single deployable unit containing all functionality.

```
┌─────────────────────────────────────┐
│        Monolithic Application        │
│  ┌────────┬────────┬─────────────┐  │
│  │ Users  │ Orders │  Payments   │  │
│  ├────────┼────────┼─────────────┤  │
│  │ Catalog│ Cart   │  Shipping   │  │
│  └────────┴────────┴─────────────┘  │
│           One Database               │
└─────────────────────────────────────┘
```

**Pros:** Simple development, testing, deployment; ACID transactions; no network overhead.
**Cons:** Single point of failure; scaling = scale everything; tech stack lock-in; slow releases.

### Microservices

Independent services communicating over network.

```
       ┌──────────┐
       │ Gateway  │
       └────┬─────┘
            │
    ┌───────┼───────┬───────┐
    ▼       ▼       ▼       ▼
┌──────┐┌──────┐┌──────┐┌──────┐
│Users ││Orders││Pay   ││Cart  │
│  DB  ││  DB  ││  DB  ││  DB  │
└──────┘└──────┘└──────┘└──────┘
```

**Pros:** Independent deploy/scale; technology freedom; fault isolation; small teams.
**Cons:** Distributed complexity; eventual consistency; network latency; harder debugging; ops overhead.

### Comparison

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| Deploy | One unit | Many units |
| Scale | Whole app | Per service |
| Data | Shared DB | DB per service |
| Transactions | ACID | Sagas / eventual |
| Latency | In-process | Network |
| Testing | Simpler | Complex |
| Team | One | Many |
| Failure | All or nothing | Isolated |
| Complexity | In code | In ops |

**Rule:** Start with a **modular monolith**; split into services only when you must.

---

## 2. Microservices Design Principles

### Bounded Context

Each service owns a **domain** (bounded context from DDD). No shared domain model across services.

### Database per Service

Each service has its **own** database. No cross-service JOINs. Communication via APIs or events.

### Loose Coupling, High Cohesion

Services should be independently deployable and changeable.

### API Contracts

Communication via stable, versioned APIs (REST, gRPC, messaging).

### Design for Failure

- Timeouts everywhere
- Retries with backoff + jitter
- Circuit breakers
- Bulkheads (isolated thread/connection pools per dependency)
- Fallbacks

### Externalize Configuration

Config lives outside the artifact; injected at runtime (env vars, Config Server).

### Stateless Services

No local session; scale horizontally freely. State goes to DB/cache/message broker.

### Observability

- Centralized logging
- Distributed tracing (correlation IDs)
- Metrics & alerts (RED/USE)
- Health checks & readiness

### Automation

- CI/CD pipelines
- Containerized deployments
- Infrastructure as Code
- Blue/green or canary releases

### 12-Factor App

1. Codebase (one per service)
2. Dependencies (explicitly declared)
3. Config (externalized)
4. Backing services (attached)
5. Build, release, run (separated)
6. Processes (stateless)
7. Port binding (self-contained)
8. Concurrency (scale out)
9. Disposability (fast startup, graceful shutdown)
10. Dev/prod parity
11. Logs (as streams)
12. Admin processes (one-off tasks)

---

## 3. Service Discovery (Eureka)

### Why Discovery?

Services scale dynamically — IPs change. Hard-coded URLs don't work. A **registry** tracks live instances.

### Two Patterns

- **Client-side discovery** — client queries registry, picks instance (Eureka, Consul, Zookeeper)
- **Server-side discovery** — client calls LB, LB routes (Kubernetes Service, AWS ELB)

### Eureka Architecture

```
      ┌─────────────────────────┐
      │      Eureka Server      │
      │      (registry)         │
      └────────▲────────────────┘
               │ register / heartbeat
   ┌───────────┼───────────┐
   │           │           │
┌──┴───┐   ┌──┴───┐   ┌──┴───┐
│Users │   │Orders│   │Pay   │
└──────┘   └──────┘   └──────┘
   │           │           │
   └───── query: "orders?" ┘
          get instances
```

### Eureka Server

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApp {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApp.class, args);
    }
}
```

```properties
server.port=8761
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

Visit `http://localhost:8761` for the dashboard.

### Eureka Client

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```properties
spring.application.name=order-service
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
eureka.instance.prefer-ip-address=true
```

```java
@SpringBootApplication
@EnableDiscoveryClient   // optional in recent versions
public class OrderApp { }
```

### DiscoveryClient API

```java
@Autowired
private DiscoveryClient discoveryClient;

public List<ServiceInstance> instancesOf(String serviceId) {
    return discoveryClient.getInstances(serviceId);
}

public URI uriOf(String serviceId) {
    ServiceInstance instance = discoveryClient.getInstances(serviceId).get(0);
    return instance.getUri();
}
```

### Eureka Self-Preservation

When Eureka doesn't receive heartbeats (e.g., network partition), it stops evicting instances to avoid mass deregistration.

- Dev: disable (`eureka.server.enable-self-preservation=false`)
- Prod: keep enabled

### Health & Heartbeat

- Client sends heartbeats every 30 s (default)
- Instance considered up if healthy (Actuator `/health`)
- Eviction after 90 s of missed heartbeats

### Region & Zone

For multi-region deployments, clients prefer instances in their own zone.

### Alternatives

- **Consul** — HashiCorp, KV store + discovery
- **Zookeeper** — old, CP
- **Kubernetes Service** — built-in; often replaces Eureka

### When NOT to Use Eureka

- Kubernetes: use native Services
- Static infra: DNS is enough
- AWS: use ELB + Route53

---

## 4. API Gateway

### Purpose

Single entry point for clients. Handles:
- Routing
- Authentication / JWT validation
- Rate limiting
- CORS
- Request/response transformation
- Load balancing
- Circuit breaking

### Spring Cloud Gateway (Reactive)

Replaces Netflix Zuul (deprecated).

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### Properties-Based Routes

```properties
spring.application.name=api-gateway
server.port=8080

spring.cloud.gateway.routes[0].id=user-service
spring.cloud.gateway.routes[0].uri=lb://user-service
spring.cloud.gateway.routes[0].predicates[0]=Path=/users/**
spring.cloud.gateway.routes[0].filters[0]=StripPrefix=1

spring.cloud.gateway.routes[1].id=order-service
spring.cloud.gateway.routes[1].uri=lb://order-service
spring.cloud.gateway.routes[1].predicates[0]=Path=/orders/**
```

`lb://` uses client-side load balancing via Eureka.

### Java DSL Routes

```java
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r
                .path("/users/**")
                .filters(f -> f.stripPrefix(1))
                .uri("lb://user-service"))
            .route("order-service", r -> r
                .path("/orders/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Gateway", "true")
                    .retry(config -> config.setRetries(3)))
                .uri("lb://order-service"))
            .build();
    }
}
```

### Predicates

| Predicate | Example |
|-----------|---------|
| `Path` | `/users/**` |
| `Method` | `GET`, `POST` |
| `Header` | `Header=X-Request-Id, \d+` |
| `Query` | `Query=type,admin` |
| `Host` | `Host=**.example.com` |
| `Cookie` | `Cookie=session,abc` |
| `After` / `Before` / `Between` | Date/time windows |
| `RemoteAddr` | IP |
| `Weight` | Canary traffic split |

### Filters

**Built-in:**
- `AddRequestHeader`, `AddResponseHeader`
- `StripPrefix`, `PrefixPath`, `RewritePath`
- `Retry`, `RequestRateLimiter`
- `CircuitBreaker`
- `SetStatus`
- `RedirectTo`

**Custom:**

```java
@Component
public class RequestIdGatewayFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String id = UUID.randomUUID().toString();
        exchange.getRequest().mutate()
            .header("X-Request-Id", id)
            .build();
        exchange.getResponse().getHeaders().add("X-Request-Id", id);
        return chain.filter(exchange);
    }
    
    @Override
    public int getOrder() { return -1; }
}
```

### Rate Limiting (Redis)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getQueryParams().getFirst("userId"));
}
```

```properties
spring.cloud.gateway.routes[0].filters[0].name=RequestRateLimiter
spring.cloud.gateway.routes[0].filters[0].args.redis-rate-limiter.replenishRate=10
spring.cloud.gateway.routes[0].filters[0].args.redis-rate-limiter.burstCapacity=20
spring.cloud.gateway.routes[0].filters[0].args.key-resolver=#{@userKeyResolver}
```

### Gateway + Security (JWT Validation)

```java
@Bean
public SecurityWebFilterChain chain(ServerHttpSecurity http) {
    return http
        .csrf(ServerHttpSecurity.CsrfSpec::disable)
        .authorizeExchange(auth -> auth
            .pathMatchers("/auth/**").permitAll()
            .anyExchange().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
        .build();
}
```

Gateway validates the JWT, then forwards with user info as headers.

### Zuul (Legacy)

`spring-cloud-starter-netflix-zuul` — Servlet-based. Deprecated. Do not use for new projects.

---

## 5. Client-Side Load Balancing

### Spring Cloud LoadBalancer

Replaces Ribbon (deprecated).

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

Used automatically with `RestTemplate`, `WebClient`, `Feign`, `Gateway` when `lb://` is used.

### RestTemplate

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// Usage — service name resolved via Eureka
restTemplate.getForObject("http://user-service/users/1", User.class);
```

### WebClient

```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}

webClientBuilder.build()
    .get().uri("http://user-service/users/1")
    .retrieve().bodyToMono(User.class);
```

### Algorithms

- Round-robin (default)
- Random
- Custom

```java
@Bean
public ReactorLoadBalancer<ServiceInstance> loadBalancer(
        Environment env, LoadBalancerClientFactory factory) {
    String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
    return new RandomLoadBalancer(
        factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
}
```

### Health Checks

LoadBalancer filters out unhealthy instances via Actuator health.

```properties
spring.cloud.loadbalancer.health-check.path.default=/actuator/health
```

### Sticky Sessions

For stateful backends, use `SameInstancePreference` (via custom config). Not recommended for microservices.

---

## 6. Declarative REST Clients (OpenFeign)

### Why Feign?

Writing HTTP clients manually is verbose. Feign generates the implementation from an interface.

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

### Enable

```java
@SpringBootApplication
@EnableFeignClients
public class OrderApp { }
```

### Define Client

```java
@FeignClient(name = "user-service")
public interface UserClient {
    
    @GetMapping("/users/{id}")
    User getUser(@PathVariable Long id);
    
    @PostMapping("/users")
    User create(@RequestBody CreateUserRequest req);
    
    @GetMapping("/users")
    List<User> search(@RequestParam("q") String q,
                      @RequestParam(defaultValue = "0") int page);
}
```

### Inject & Use

```java
@Service
public class OrderService {
    private final UserClient userClient;
    
    public OrderService(UserClient userClient) { this.userClient = userClient; }
    
    public Order place(PlaceOrderRequest req) {
        User user = userClient.getUser(req.userId());
        // ...
    }
}
```

Feign uses Spring Cloud LoadBalancer + Eureka automatically.

### Configuration

```properties
# Timeouts
feign.client.config.default.connect-timeout=5000
feign.client.config.default.read-timeout=10000

# Per-client
feign.client.config.user-service.connect-timeout=2000

# Logging
feign.client.config.default.logger-level=full
logging.level.com.example.clients.UserClient=DEBUG
```

### Interceptors

Add headers, auth tokens:

```java
@Bean
public RequestInterceptor authInterceptor() {
    return template -> {
        String token = getToken();
        if (token != null) template.header("Authorization", "Bearer " + token);
    };
}
```

Or propagate correlation ID (Micrometer Tracing):

```java
@Bean
public RequestInterceptor traceInterceptor() {
    return template -> {
        String traceId = MDC.get("traceId");
        if (traceId != null) template.header("X-Trace-Id", traceId);
    };
}
```

### Error Handling

```java
@Configuration
public class FeignConfig {
    @Bean
    public ErrorDecoder errorDecoder() {
        return (methodKey, response) -> {
            if (response.status() == 404) {
                return new NotFoundException("Resource not found");
            }
            return new FeignException.FeignClientException(
                response.status(), "Error", response.request(), null, null);
        };
    }
}
```

### Fallback (with Resilience4j)

```java
@FeignClient(name = "user-service", fallback = UserClientFallback.class)
public interface UserClient { ... }

@Component
public class UserClientFallback implements UserClient {
    @Override
    public User getUser(Long id) {
        return new User(id, "Unknown", "unknown@example.com");
    }
    // ...
}
```

```properties
feign.circuitbreaker.enabled=true
```

### Feign vs WebClient vs RestTemplate

| Feature | Feign | WebClient | RestTemplate |
|---------|-------|-----------|--------------|
| Declarative | ✅ | ❌ | ❌ |
| Reactive | ❌ (in classic Feign) | ✅ | ❌ |
| Blocking | ✅ | ⚠️ (block) | ✅ |
| Load Balanced | ✅ | ✅ | ✅ (with @LoadBalanced) |
| Spring Cloud integration | ✅ | ✅ | ✅ |

**Modern choice:**
- Blocking microservices → OpenFeign
- Reactive → WebClient
- Legacy → RestTemplate (deprecated)

---

## 7. Circuit Breakers (Resilience4j)

### Why Circuit Breakers?

Prevent cascading failures. If a downstream service is failing, stop calling it — fail fast, fall back, or degrade.

### States

```
        CLOSED  ◄──── success ─────┐
           │                       │
           │ failures ≥ threshold  │
           ▼                       │
         OPEN ────── timeout ───► HALF_OPEN
           │                       │
           │                       │ success
           │                       ▼
           └─────────── ──────► CLOSED
```

- **CLOSED** — normal; calls pass through
- **OPEN** — failing; calls rejected immediately
- **HALF_OPEN** — testing; some calls pass to check recovery

### Resilience4j Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

### Configuration

```properties
resilience4j.circuitbreaker.instances.userService.slidingWindowSize=10
resilience4j.circuitbreaker.instances.userService.minimumNumberOfCalls=5
resilience4j.circuitbreaker.instances.userService.failureRateThreshold=50
resilience4j.circuitbreaker.instances.userService.waitDurationInOpenState=10s
resilience4j.circuitbreaker.instances.userService.permittedNumberOfCallsInHalfOpenState=3

resilience4j.timelimiter.instances.userService.timeoutDuration=2s
resilience4j.retry.instances.userService.maxAttempts=3
resilience4j.retry.instances.userService.waitDuration=500ms
```

### Programmatic Use

```java
@Service
public class UserService {
    private final CircuitBreakerFactory<?, ?> factory;
    private final RestTemplate rest;
    
    public User getUser(Long id) {
        CircuitBreaker cb = factory.create("userService");
        return cb.run(
            () -> rest.getForObject("http://user-service/users/" + id, User.class),
            throwable -> fallbackUser(id)
        );
    }
    
    private User fallbackUser(Long id) {
        return new User(id, "Fallback", "n/a");
    }
}
```

### Annotations (Aspect-Oriented)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

```java
@Service
public class UserService {
    
    @CircuitBreaker(name = "userService", fallbackMethod = "fallbackUser")
    @Retry(name = "userService")
    @TimeLimiter(name = "userService")
    public CompletableFuture<User> getUser(Long id) {
        return CompletableFuture.supplyAsync(() -> 
            rest.getForObject("http://user-service/users/" + id, User.class));
    }
    
    public CompletableFuture<User> fallbackUser(Long id, Throwable t) {
        return CompletableFuture.completedFuture(
            new User(id, "Fallback", "n/a"));
    }
}
```

Note: `@TimeLimiter` requires a `CompletableFuture` return type.

### Bulkhead

Isolate calls per dependency (thread pool or semaphore).

```java
@Bulkhead(name = "userService", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<User> getUser(Long id) { ... }
```

```properties
resilience4j.thread-pool-bulkhead.instances.userService.maxThreadPoolSize=10
resilience4j.thread-pool-bulkhead.instances.userService.coreThreadPoolSize=5
resilience4j.thread-pool-bulkhead.instances.userService.queueCapacity=20
```

### Rate Limiter

```java
@RateLimiter(name = "userService")
public User getUser(Long id) { ... }
```

```properties
resilience4j.ratelimiter.instances.userService.limitForPeriod=100
resilience4j.ratelimiter.instances.userService.limitRefreshPeriod=1s
```

### Retry

```java
@Retry(name = "userService")
public User getUser(Long id) { ... }
```

Only retries on configured exceptions.

```properties
resilience4j.retry.instances.userService.retryExceptions=java.io.IOException
resilience4j.retry.instances.userService.ignoreExceptions=com.example.BadRequestException
```

### Metrics

Resilience4j auto-publishes Micrometer metrics:

```
resilience4j_circuitbreaker_state{name="userService"}
resilience4j_circuitbreaker_calls_total{name="userService", kind="successful"}
resilience4j_retry_calls_total{name="userService", kind="failed_with_retry"}
```

Visible at `/actuator/metrics` or Prometheus.

### Fallback Strategy Examples

- Return cached value
- Return default/safe value
- Return empty response
- Queue for later (async)
- Return structured error

### Circuit Breaker in Gateway

```yaml
filters:
  - name: CircuitBreaker
    args:
      name: userServiceCB
      fallbackUri: forward:/fallback/users
```

---

## 8. Distributed Configuration (Config Server)

### Why Config Server?

Central source of truth for configuration across all services. Enables runtime updates and audit.

### Config Server

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApp { }
```

```properties
server.port=8888
spring.cloud.config.server.git.uri=https://github.com/org/config-repo
spring.cloud.config.server.git.default-label=main
spring.cloud.config.server.git.username=${GIT_USER}
spring.cloud.config.server.git.password=${GIT_TOKEN}
```

### Config Repository Layout

```
config-repo/
├── application.yml               # global
├── application-dev.yml           # global dev
├── user-service.yml
├── user-service-dev.yml
├── user-service-prod.yml
├── order-service.yml
└── order-service-prod.yml
```

### Config Client

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

```properties
spring.application.name=user-service
spring.config.import=configserver:http://config-server:8888
spring.profiles.active=prod
```

Since Spring Boot 2.4+, `spring.config.import=configserver:` is the modern approach (replaces `bootstrap.properties`).

### Config Server Endpoints

```
GET /{application}/{profile}
GET /{application}/{profile}/{label}
GET /{application}-{profile}.yml
GET /{label}/{application}-{profile}.yml
```

Example:

```bash
curl http://localhost:8888/user-service/prod
```

### Refresh Without Restart

Add Actuator to client:

```properties
management.endpoints.web.exposure.include=refresh
```

Trigger refresh:

```bash
curl -X POST http://user-service:8080/actuator/refresh
```

Reloads `@RefreshScope` beans.

### @RefreshScope

```java
@Component
@RefreshScope
public class FeatureFlags {
    @Value("${feature.new-ui:false}")
    private boolean newUi;
    // getter
}
```

`@RefreshScope` recreates the bean on refresh so values are reloaded.

### Multiple Config Sources

```properties
spring.cloud.config.server.git.uri=...
spring.cloud.config.server.composite[0].type=git
spring.cloud.config.server.composite[0].uri=...
spring.cloud.config.server.composite[1].type=vault
spring.cloud.config.server.composite[1].vault.host=...
```

### Vault Backend

```properties
spring.profiles.active=vault
spring.cloud.config.server.vault.host=vault.example.com
spring.cloud.config.server.vault.port=8200
spring.cloud.config.server.vault.token=${VAULT_TOKEN}
```

Great for secrets.

### Security

Protect Config Server with basic auth or OAuth2:

```java
@Configuration
@EnableWebSecurity
public class ConfigSecurityConfig {
    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

Client:

```properties
spring.cloud.config.username=config-user
spring.cloud.config.password=${CONFIG_PASSWORD}
```

---

## 9. Distributed Tracing (Micrometer Tracing & Zipkin)

### The Problem

A user request spans multiple services. Logs alone don't show the full path.

### Concepts

- **Trace** — one end-to-end request (has a `traceId`)
- **Span** — an operation within a trace (has a `spanId`, `parentId`)
- **Context propagation** — passing trace/span IDs across service calls

### W3C Trace Context

Standards-based headers:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: vendor=value
```

### Micrometer Tracing (Boot 3+)

Replaces Spring Cloud Sleuth.

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Configuration

```properties
management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://zipkin:9411/api/v2/spans
logging.pattern.level=%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]
```

### Trace IDs in Logs

```
INFO [user-service,a3c5f1b8e9d2,00112233445566] --- [nio-8080-exec-1] c.e.UserController : Fetching user
```

`traceId`, `spanId` in every log line — critical for grep.

### Zipkin Server

```bash
docker run -d -p 9411:9411 openzipkin/zipkin
```

Visit `http://localhost:9411` to see traces.

### Automatic Instrumentation

- HTTP client/server
- Kafka, RabbitMQ
- JDBC (query tracing)
- RestTemplate, WebClient, Feign

### Custom Spans

```java
@Autowired
private Tracer tracer;

public void process() {
    Span span = tracer.nextSpan().name("business-process").start();
    try (Tracer.SpanInScope scope = tracer.withSpan(span)) {
        // do work
    } finally {
        span.end();
    }
}
```

Or annotations:

```java
@Observed(name = "process", contextualName = "process-order")
public Order process(Long orderId) { ... }
```

### Exporters

| Exporter | Library |
|----------|---------|
| Zipkin | `zipkin-reporter-brave` |
| OTLP (OpenTelemetry) | `micrometer-tracing-bridge-otel` + `opentelemetry-exporter-otlp` |
| Wavefront | `micrometer-tracing-reporter-wavefront` |

### OTLP to Jaeger/Tempo

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```properties
management.otlp.tracing.endpoint=http://tempo:4318/v1/traces
```

---

## 10. Spring Cloud Bus

### Purpose

Broadcast events (e.g., config refresh) to all services via a message broker.

### Setup

Uses RabbitMQ or Kafka.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

### Refresh All Services

```bash
curl -X POST http://config-server/actuator/busrefresh
```

All subscribed services refresh their `@RefreshScope` beans. No per-service curl needed.

### Destination

```bash
curl -X POST http://config-server/actuator/busrefresh/user-service:8080
```

Target a specific instance.

### Custom Events

```java
@Autowired
private ApplicationEventPublisher publisher;

public void notifyAll() {
    publisher.publishEvent(new CustomEvent("data"));
}
```

Cloud Bus distributes to all instances.

### Refresh on Config Change (Webhook)

GitHub webhook → Config Server `/monitor` → Bus broadcasts refresh.

```properties
spring.cloud.config.server.monitor.github.enabled=true
```

---

## 11. Spring Cloud Stream

### Purpose

Messaging abstraction over Kafka/RabbitMQ. Write once, run on either.

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-kafka</artifactId>
</dependency>
```

### Functional Model (Modern)

```java
@Configuration
public class StreamConfig {
    
    @Bean
    public Supplier<Order> orderSupplier() {
        return () -> new Order(UUID.randomUUID().toString());
    }
    
    @Bean
    public Function<Order, ProcessedOrder> processOrder() {
        return order -> new ProcessedOrder(order.id(), "PROCESSED");
    }
    
    @Bean
    public Consumer<ProcessedOrder> consumeProcessed() {
        return processed -> log.info("Received: {}", processed);
    }
}
```

### Binding

```properties
spring.cloud.function.definition=orderSupplier;processOrder;consumeProcessed

spring.cloud.stream.bindings.orderSupplier-out-0.destination=orders
spring.cloud.stream.bindings.processOrder-in-0.destination=orders
spring.cloud.stream.bindings.processOrder-out-0.destination=processed-orders
spring.cloud.stream.bindings.consumeProcessed-in-0.destination=processed-orders
```

### Consumer Groups

```properties
spring.cloud.stream.bindings.consumeProcessed-in-0.group=order-processors
```

Enables competing consumers (load balancing).

### Partitions

```properties
spring.cloud.stream.bindings.processOrder-out-0.producer.partitionKeyExpression=payload.id
spring.cloud.stream.bindings.processOrder-out-0.producer.partitionCount=3
spring.cloud.stream.bindings.processOrder-in-0.consumer.partitioned=true
```

### Error Handling

```properties
spring.cloud.stream.bindings.consumeProcessed-in-0.consumer.max-attempts=3
spring.cloud.stream.bindings.consumeProcessed-in-0.consumer.back-off-initial-interval=1000
spring.cloud.stream.bindings.consumeProcessed-in-0.consumer.back-off-max-interval=10000
spring.cloud.stream.bindings.consumeProcessed-in-0.consumer.back-off-multiplier=2
```

DLQ:

```properties
spring.cloud.stream.bindings.consumeProcessed-in-0.consumer.enable-dlq=true
spring.cloud.stream.bindings.consumeProcessed-in-0.consumer.dlq-name=processed-orders-dlq
```

### Binder Switching

Change binder without code changes:

```properties
# Kafka
spring.cloud.stream.default-binder=kafka

# RabbitMQ
spring.cloud.stream.default-binder=rabbit
```

### When to Use

- Cross-broker portability
- Functional programming style
- ETL pipelines
- Event-driven microservices

### When NOT to Use

- Broker-specific features needed → use `spring-kafka` or `spring-amqp` directly
- Fine control over partitioning/offsets
- Very simple producer/consumer (native is simpler)

---

## 12. Spring Cloud Contract

### Purpose

Consumer-driven contracts. Provider publishes contracts; consumer uses them to generate stubs for testing.

### Producer Side

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-verifier</artifactId>
    <scope>test</scope>
</dependency>
```

Contract (Groovy DSL):

```groovy
Contract.make {
    description "Get user by id"
    request {
        method GET()
        url "/users/1"
    }
    response {
        status OK()
        headers { contentType applicationJson() }
        body([
            id: 1,
            name: "Alice"
        ])
    }
}
```

Plugin generates tests. Passing tests → `spring-cloud-contract-verifier` produces a stub JAR.

### Consumer Side

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureStubRunner(
    ids = "com.example:user-service:+:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class UserClientTest {
    
    @Test
    void testGetUser() {
        // client hits localhost:8090 — served by generated stubs
    }
}
```

### Benefits

- Consumer defines expectations
- Provider verifies them
- Both sides test against the same contract
- Breaks surface at build time, not runtime

---

## 13. Spring Cloud Security

### Propagating Tokens

Feign + OAuth2: propagate the JWT from the incoming request to downstream calls.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

```java
@Bean
public RequestInterceptor oauth2FeignRequestInterceptor() {
    return template -> {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.getCredentials() instanceof AbstractOAuth2Token token) {
            template.header("Authorization", "Bearer " + token.getTokenValue());
        }
    };
}
```

### Gateway + Resource Server

Gateway validates JWT, then forwards claims as headers:

```java
@Component
public class UserHeaderGatewayFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return exchange.getPrincipal()
            .cast(JwtAuthenticationToken.class)
            .flatMap(auth -> {
                ServerHttpRequest req = exchange.getRequest().mutate()
                    .header("X-User-Id", (String) auth.getToken().getClaim("sub"))
                    .build();
                return chain.filter(exchange.mutate().request(req).build());
            });
    }
}
```

Downstream services trust the header (only gateway is exposed).

### Service-to-Service with Client Credentials

See §10 in `14_Spring_Security_JWT_OAuth2.md`.

### mTLS

For zero-trust internal networks, use mutual TLS (Istio, Linkerd provide it transparently).

---

## 14. Docker & Kubernetes for Spring Boot

### Dockerfile (Layered)

```dockerfile
FROM eclipse-temurin:17-jre AS builder
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

Layered JAR improves Docker caching (dependencies change less often than code).

### Build Image with Spring Boot Plugin

```bash
./mvnw spring-boot:build-image
# or
./gradlew bootBuildImage
```

Uses Cloud Native Buildpacks; no Dockerfile needed.

```properties
spring-boot.build-image.imageName=my-app:1.0.0
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: my-registry/user-service:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1"
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 8080
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
data:
  application.yaml: |
    server:
      port: 8080
    spring:
      datasource:
        url: jdbc:postgresql://db:5432/users
```

Mount:

```yaml
volumeMounts:
- name: config
  mountPath: /config
volumes:
- name: config
  configMap:
    name: user-service-config
```

Or import:

```properties
spring.config.import=configtree:/etc/config/
```

### Spring Cloud Kubernetes

Reads ConfigMaps and Secrets as property sources; native service discovery via K8s.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
</dependency>
```

```properties
spring.cloud.kubernetes.config.enabled=true
spring.cloud.kubernetes.secrets.enabled=true
```

### Graceful Shutdown

```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

Ensure K8s `terminationGracePeriodSeconds` > shutdown timeout.

### Health Probes

```properties
management.endpoint.health.probes.enabled=true
```

K8s livenessProbe → `/actuator/health/liveness`; readinessProbe → `/actuator/health/readiness`.

---

## 15. Microservices Patterns

### Decomposition

- **By business capability** (Orders, Users, Payments)
- **By subdomain** (DDD bounded contexts)

### Data Management

- **Database per service**
- **Saga** (choreography or orchestration) for distributed transactions
- **Outbox** — atomic write to DB + event table
- **Event sourcing** — store events, derive state
- **CQRS** — separate read/write models

### Communication

- **Sync** — REST, gRPC
- **Async** — Kafka, RabbitMQ, Spring Cloud Stream
- **Service mesh** — Istio, Linkerd (sidecar proxy)

### Resilience

- **Timeout** everywhere
- **Retry** with backoff + jitter
- **Circuit breaker** (Resilience4j)
- **Bulkhead** — isolate thread/connection pools
- **Rate limiter**
- **Fallback** — safe default, cache, defer

### Observability

- **Logs** — structured, centralized (ELK, Loki)
- **Metrics** — Micrometer + Prometheus + Grafana
- **Traces** — Micrometer Tracing + Zipkin/Jaeger/Tempo
- **Correlation IDs** — propagate headers
- **Health checks** — Actuator

### Deployment

- **Containers** — Docker
- **Orchestration** — Kubernetes
- **Service mesh** — Istio, Linkerd
- **GitOps** — ArgoCD, Flux
- **Blue/green**, **canary**, **rolling**

### Testing

- **Contract tests** — Spring Cloud Contract, Pact
- **Component tests** — `@SpringBootTest` + Testcontainers
- **Consumer-driven contracts**
- **Chaos engineering** — Chaos Monkey, Litmus

### Saga Pattern (Choreography)

```
Order Service  ──OrderCreated──▶  Payment Service
                                      │
                                 PaymentDone
                                      │
                                      ▼
                                 Inventory Service
                                      │
                                 InventoryReserved
                                      │
                                      ▼
                                 Shipping Service
```

Compensating transactions on failure.

### Outbox Pattern

```java
@Transactional
public void placeOrder(Order order) {
    orderRepo.save(order);
    outboxRepo.save(new OutboxEvent("OrderCreated", toJson(order)));
}

// Separate publisher reads outbox and sends to Kafka
@Scheduled(fixedDelay = 1000)
public void publishOutbox() {
    outboxRepo.findUnpublished().forEach(event -> {
        kafka.send(event.type(), event.payload());
        event.markPublished();
    });
}
```

Ensures atomicity: DB write + event write in same transaction; async publisher picks up.

### Saga Orchestration

A coordinator service drives the saga steps:

```
OrderSaga
├── 1. reservePayment()  → success → 2
├── 2. reserveInventory() → success → 3
├── 3. ship()            → success → DONE
└── on failure at step N:
    compensate steps N-1...1 (release reservation, refund)
```

Tools: Axon, Camunda, Temporal, or custom state machine.

### Strangler Fig (Migration)

Incrementally replace parts of a monolith with microservices behind a facade.

```
Old monolith ─┐
              ├── Facade/Gateway ── Clients
New services ─┘
```

As services are built, facade routes traffic to them.

---

## 16. Interview Questions

### Q1. What is a microservice?

**Answer:** An independently deployable, loosely coupled service that owns a specific business capability and its data. Communicates with others via lightweight protocols (HTTP, messaging). Enables independent scaling, technology choice, and team autonomy.

### Q2. Monolith vs microservices — which to choose?

**Answer:** Start with a **modular monolith** for simplicity, ACID transactions, and speed. Split into microservices only when you need independent deploy/scale, have multiple teams, and can absorb operational complexity. Premature microservices = distributed monolith pain.

### Q3. What is service discovery?

**Answer:** Dynamic registry of live service instances. Clients query the registry to find instances (client-side) or a load balancer routes (server-side). In Spring: **Netflix Eureka**. In K8s: **native Services**.

### Q4. How does Eureka work?

**Answer:** Eureka Server maintains a registry. Clients register on startup, send heartbeats every 30s, and are evicted after 90s of missed heartbeats. Clients fetch the registry and pick instances via LoadBalancer. Supports self-preservation to avoid mass evictions on network partitions.

### Q5. What is Spring Cloud Gateway?

**Answer:** A reactive API gateway. Routes requests to backend services via predicates (Path, Method, Header) and filters (StripPrefix, Retry, CircuitBreaker). Integrates with Eureka (`lb://`), JWT validation, and rate limiting.

### Q6. Zuul vs Spring Cloud Gateway?

**Answer:** Zuul (Netflix) is Servlet-based, blocking — now deprecated. Spring Cloud Gateway is reactive (Project Reactor + Netty), supports modern filters, and is the recommended choice for Spring Boot 2.x/3.x.

### Q7. What is OpenFeign?

**Answer:** A declarative HTTP client. Define an interface with `@FeignClient` and Spring generates the implementation. Integrates with Eureka, LoadBalancer, OAuth2, Resilience4j. Simplifies service-to-service calls.

### Q8. Explain circuit breaker pattern.

**Answer:** Prevents cascading failures by tripping a "circuit" (OPEN) when a downstream service fails repeatedly. Calls fail fast (or use fallback) instead of waiting. After a timeout, the breaker moves to HALF_OPEN to test recovery. **Resilience4j** is the modern Spring implementation.

### Q9. What is Resilience4j?

**Answer:** A lightweight fault-tolerance library replacing Hystrix. Provides CircuitBreaker, RateLimiter, Bulkhead, Retry, TimeLimiter. Integrates via `@CircuitBreaker`, `@Retry`, `@Bulkhead`, `@RateLimiter`, `@TimeLimiter` annotations. Publishes Micrometer metrics.

### Q10. What is Spring Cloud Config?

**Answer:** A centralized configuration server. Stores configs in Git, Vault, or file system. Clients fetch config on startup via `spring.config.import=configserver:...`. Supports refresh without restart via Actuator + `@RefreshScope`.

### Q11. How do you refresh config without restarting?

**Answer:** Add Actuator `refresh` endpoint to client, use `@RefreshScope` on beans. POST to `/actuator/refresh` (per service) or `/actuator/busrefresh` (all services via Spring Cloud Bus).

### Q12. What is Spring Cloud Bus?

**Answer:** Broadcasts events (like config refresh) to all service instances via a message broker (RabbitMQ, Kafka). One POST to `/actuator/busrefresh` refreshes every subscribed service.

### Q13. What is distributed tracing?

**Answer:** Tracking a request across multiple services using a shared `traceId` and per-operation `spanId`s. Spring Boot 3 uses **Micrometer Tracing** (replacing Sleuth) with exporters to Zipkin, Jaeger, or OTLP. Auto-instruments HTTP, DB, messaging.

### Q14. What is OpenTelemetry and how does it relate?

**Answer:** An open standard for traces, metrics, and logs. Micrometer Tracing can bridge to OpenTelemetry (`micrometer-tracing-bridge-otel`) and export via OTLP to any backend (Jaeger, Tempo, Datadog).

### Q15. What is the Saga pattern?

**Answer:** Manages distributed transactions via a sequence of local transactions with compensating actions on failure. Two styles: **choreography** (services react to events) and **orchestration** (central coordinator drives steps).

### Q16. What is the Outbox pattern?

**Answer:** Atomically writes a business change and an outbox event in the same DB transaction. A separate publisher polls the outbox and publishes to the message broker. Solves the "DB commit + message publish" atomicity problem.

### Q17. How do you test microservices?

**Answer:** Unit tests per service; **contract tests** (Spring Cloud Contract, Pact); **integration tests** with Testcontainers (real DB/broker); **component tests** with `@SpringBootTest`; **E2E** on staging. Contract tests catch cross-service breakage early.

### Q18. How do you secure microservices?

**Answer:** External auth at the Gateway (OAuth2/OIDC). Services act as Resource Servers (validate JWTs). Service-to-service uses client credentials or mTLS (via service mesh). Propagate tokens via Feign interceptors.

### Q19. What is service mesh and do you need it?

**Answer:** An infrastructure layer (Istio, Linkerd) with sidecar proxies providing mTLS, tracing, retries, circuit breaking, and traffic shifting at the platform level. Removes this logic from app code. Needed for large clusters; overkill for small ones.

### Q20. What is the strangler fig pattern?

**Answer:** Incrementally migrate from monolith to microservices by placing a facade in front. Route new functionality to new services while keeping legacy flows through the monolith. Reduces big-bang migration risk.

### Q21. How do you handle eventual consistency?

**Answer:** Use sagas with compensations, idempotent operations, and event-driven updates. Design UI for eventual consistency (optimistic updates, polling). Add reconciliation jobs for drift.

### Q22. What is CQRS?

**Answer:** Command Query Responsibility Segregation — separate read and write models. Writes go to a normalized store; reads served from denormalized projections. Enables independent scaling and optimized read models.

### Q23. What is the difference between Spring Cloud Stream and using Kafka directly?

**Answer:** Spring Cloud Stream is an abstraction over brokers (Kafka, RabbitMQ). Same code works on either by swapping binders. Use it for portability and functional style. Use native Spring Kafka for broker-specific features.

### Q24. How do you deploy a Spring Boot microservice to Kubernetes?

**Answer:** Build a container image (`bootBuildImage` or Dockerfile). Create a Deployment, Service, ConfigMap, Secret. Configure liveness/readiness probes using Actuator. Set env vars for profile and secrets. Use HPA for autoscaling. Optionally use Spring Cloud Kubernetes for config/discovery.

### Q25. What are health checks and why are they important?

**Answer:** Endpoints that report service status. K8s uses **liveness** (restart if dead) and **readiness** (route traffic only when ready). Spring Boot Actuator exposes them at `/actuator/health/liveness` and `/actuator/health/readiness` with proper semantics (liveness should not depend on downstreams).

---

## 17. Cheat Sheet

### Spring Cloud Modules

```
Eureka              → Service discovery
Gateway             → API Gateway
LoadBalancer        → Client-side LB
OpenFeign           → Declarative clients
Circuit Breaker     → Resilience4j
Config              → Centralized config
Bus                 → Broadcast config refresh
Stream              → Kafka/RabbitMQ abstraction
Sleuth/Tracing      → Distributed tracing
Contract            → Consumer-driven contracts
Kubernetes          → K8s integration
Vault               → Secrets (via Config Server)
```

### Common Dependencies

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Eureka Server

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApp { }
```

```properties
server.port=8761
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

### Eureka Client

```properties
spring.application.name=user-service
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

### Gateway Routes (YAML)

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: user-service
        uri: lb://user-service
        predicates:
        - Path=/users/**
        filters:
        - StripPrefix=1
```

### Feign Client

```java
@FeignClient(name = "user-service", fallback = UserClientFallback.class)
public interface UserClient {
    @GetMapping("/users/{id}")
    User getUser(@PathVariable Long id);
}
```

```java
@SpringBootApplication
@EnableFeignClients
public class OrderApp { }
```

### Circuit Breaker

```java
@CircuitBreaker(name = "userService", fallbackMethod = "fallbackUser")
@Retry(name = "userService")
public User getUser(Long id) { ... }

public User fallbackUser(Long id, Throwable t) { return new User(id, "N/A"); }
```

```properties
resilience4j.circuitbreaker.instances.userService.failureRateThreshold=50
resilience4j.circuitbreaker.instances.userService.slidingWindowSize=10
resilience4j.circuitbreaker.instances.userService.waitDurationInOpenState=10s
```

### Config Client

```properties
spring.application.name=user-service
spring.config.import=configserver:http://config-server:8888
spring.profiles.active=prod
```

### Tracing

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```properties
management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://zipkin:9411/api/v2/spans
logging.pattern.level=%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]
```

### Kubernetes Probes

```properties
management.endpoint.health.probes.enabled=true
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

```yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8080 }
```

### Docker Build

```bash
./mvnw spring-boot:build-image
./gradlew bootBuildImage
```

### Microservices Checklist

- [ ] Bounded context per service
- [ ] Database per service
- [ ] Timeouts everywhere
- [ ] Retries with backoff + jitter
- [ ] Circuit breaker per dependency
- [ ] Idempotent operations
- [ ] Correlation ID propagation
- [ ] Metrics + logs + traces
- [ ] Health checks (liveness + readiness)
- [ ] Graceful shutdown
- [ ] Externalized config
- [ ] Contract tests
- [ ] Secrets via Vault/K8s Secrets
- [ ] API versioning strategy

### Cross-References

- **Previous:** `18_Gradle_For_Spring.md`
- **Next:** `20_Spring_WebFlux_Reactive.md`
- **Related:** `09_Spring_Boot_Actuator.md`, `13_Spring_Security_Core.md`, `14_Spring_Security_JWT_OAuth2.md`, `16_Spring_Testing.md`
- **Interview:** `24_Spring_Interview_Questions.md` (§Microservices)

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [18_Gradle_For_Spring.md](./18_Gradle_For_Spring.md)
- **Next →:** [20_Spring_WebFlux_Reactive.md](./20_Spring_WebFlux_Reactive.md)
- **Related:** [14_Spring_Security_JWT_OAuth2.md](./14_Spring_Security_JWT_OAuth2.md), [20_Spring_WebFlux_Reactive.md](./20_Spring_WebFlux_Reactive.md), [22_Spring_Messaging.md](./22_Spring_Messaging.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
