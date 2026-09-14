# Spring WebFlux & Reactive Programming

> **File:** `20_Spring_WebFlux_Reactive.md`
> **Part:** 7 — Advanced Topics
> **Prerequisites:** `03_Spring_MVC.md`, `06_Spring_Boot_Fundamentals.md`, Java 8+ lambdas/streams
> **Estimated Study Time:** 10–14 hours

---

## Table of Contents

1. [Reactive Programming](#1-reactive-programming)
2. [Reactive Streams Specification](#2-reactive-streams-specification)
3. [Project Reactor: Mono and Flux](#3-project-reactor-mono-and-flux)
4. [Creating Publishers](#4-creating-publishers)
5. [Operators](#5-operators)
6. [Backpressure](#6-backpressure)
7. [Error Handling in Reactor](#7-error-handling-in-reactor)
8. [Schedulers & Threading](#8-schedulers--threading)
9. [Spring WebFlux — Annotated Controllers](#9-spring-webflux--annotated-controllers)
10. [Spring WebFlux — Functional Endpoints](#10-spring-webflux--functional-endpoints)
11. [WebClient](#11-webclient)
12. [R2DBC (Reactive SQL)](#12-r2dbc-reactive-sql)
13. [Reactive MongoDB & Redis](#13-reactive-mongodb--redis)
14. [Server-Sent Events (SSE)](#14-server-sent-events-sse)
15. [WebSocket](#15-websocket)
16. [Testing Reactive Streams](#16-testing-reactive-streams)
17. [WebFlux vs WebMVC](#17-webflux-vs-webmvc)
18. [Interview Questions](#18-interview-questions)
19. [Cheat Sheet](#19-cheat-sheet)

---

## 1. Reactive Programming

**Reactive programming** is a paradigm focused on **asynchronous data streams** and **propagation of change**.

### Core Principles

- **Asynchronous** — non-blocking
- **Event-driven** — react to events
- **Non-blocking** — no thread waiting
- **Composable** — chain operators
- **Backpressure-aware** — consumer controls flow

### Imperative vs Reactive

**Imperative (blocking):**
```java
User user = userRepo.findById(1L);      // thread blocks
Order order = orderClient.getOrder(1L); // thread blocks
return combine(user, order);            // done
```

**Reactive (non-blocking):**
```java
Mono<User> user = userRepo.findById(1L);
Mono<Order> order = orderClient.getOrder(1L);
return Mono.zip(user, order).map(this::combine);
```

The reactive code returns immediately. The work happens when subscribed.

### Why Reactive?

- **Efficient** — handle more concurrent requests with fewer threads
- **Scalable** — no thread per request
- **Composable** — no callback hell
- **Backpressure** — no overwhelming consumers
- **Streaming** — natural fit for SSE, WebSocket

### When to Use Reactive

- High concurrency, I/O-bound workloads
- Streaming data (SSE, WebSocket)
- Microservice gateways, aggregators
- Reactive data sources (Mongo, Cassandra, Redis, R2DBC)

### When NOT to Use

- Blocking DB drivers (JDBC, JPA) — kills the benefit
- Heavy CPU work — thread pool better
- Traditional teams unfamiliar with reactive
- Simple CRUD — WebMVC is simpler and just as fast

> ⚠️ **Rule:** If your DB driver is blocking (JDBC/JPA), don't adopt WebFlux. You gain nothing.

---

## 2. Reactive Streams Specification

Standard defining async stream processing with backpressure.

### Four Interfaces

```java
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> s);
}

public interface Subscriber<T> {
    void onSubscribe(Subscription s);
    void onNext(T t);
    void onError(Throwable t);
    void onComplete();
}

public interface Subscription {
    void request(long n);
    void cancel();
}

public interface Processor<T, R> extends Subscriber<T>, Publisher<R> { }
```

### Flow

```
Publisher ──onSubscribe──▶ Subscriber
          ◀──request(n)──
          ──onNext(item)─▶
          ──onNext(item)─▶
          ... (up to n)
          ──onComplete───▶
          (or onError)
```

### Rules

1. Subscriber controls flow via `request(n)`
2. Publisher can't emit more than requested
3. If subscriber requests `Long.MAX_VALUE` → unlimited
4. Errors terminate the stream

### Implementations

- **Project Reactor** — Spring's default
- **RxJava** — Netflix
- **Akka Streams**
- **JDK `Flow` API** (Java 9+)

---

## 3. Project Reactor: Mono and Flux

### Mono<T>

**0 or 1 element.** Represents a single async result.

```java
Mono<String> mono = Mono.just("Hello");
Mono<User> user = Mono.empty();
Mono<User> error = Mono.error(new RuntimeException());
```

### Flux<T>

**0..N elements.** Represents a stream.

```java
Flux<String> flux = Flux.just("A", "B", "C");
Flux<Integer> range = Flux.range(1, 10);
Flux<Long> interval = Flux.interval(Duration.ofSeconds(1));
Flux<User> users = userRepo.findAll();
```

### Lazy vs Eager

Nothing happens until **subscription**:

```java
Mono<String> mono = Mono.fromCallable(() -> {
    System.out.println("Computed");
    return "Hi";
});
// Nothing printed yet!

mono.subscribe(v -> System.out.println("Got: " + v));
// Now: "Computed", "Got: Hi"
```

### Subscribing

```java
mono.subscribe(
    value -> System.out.println("Value: " + value),
    error -> System.err.println("Error: " + error),
    () -> System.out.println("Done"),
    subscription -> subscription.request(1)
);
```

⚠️ **Rarely subscribe manually in WebFlux** — framework subscribes on HTTP layer.

### Blocking vs Non-Blocking

```java
// Blocking (NEVER in reactive code)
String value = mono.block();
List<String> all = flux.collectList().block();

// Compose instead
mono.map(String::toUpperCase).subscribe();
```

Use `block()` only in tests or non-reactive entry points.

### Transformation vs Subscription

```java
Mono<User> userMono = Mono.just(new User("Alice"));

Mono<String> nameMono = userMono.map(User::getName);       // not executed yet
nameMono.subscribe(System.out::println);                   // executed now
```

---

## 4. Creating Publishers

### From Values

```java
Mono.just("value")
Mono.empty()
Mono.never()
Mono.error(new RuntimeException("Oops"))
Flux.just(1, 2, 3)
Flux.range(1, 10)
Flux.empty()
Flux.never()
Flux.error(new RuntimeException())
```

### From Collections

```java
Flux.fromIterable(List.of(1, 2, 3))
Flux.fromArray(new String[]{"a", "b"})
Flux.fromStream(Stream.of(1, 2, 3))
```

### From Optional/Nullable

```java
Mono.justOrEmpty(Optional.of("x"))
Mono.justOrEmpty((String) null)   // empty Mono
```

### From Callable/Supplier (lazy)

```java
Mono.fromCallable(() -> computeExpensiveValue())
Mono.fromSupplier(() -> fetchValue())
Mono.fromRunnable(() -> doWork())
```

### From CompletableFuture

```java
Mono.fromFuture(CompletableFuture.supplyAsync(() -> "Hi"))
Mono.fromCompletionStage(cs)
```

### From Publisher

```java
Flux.from(otherFlux)
Mono.from(otherFlux)
```

### Interval / Timer

```java
Flux.interval(Duration.ofSeconds(1))            // 0, 1, 2, ...
Flux.interval(Duration.ofSeconds(1), Duration.ofMillis(100))   // delay start
Mono.delay(Duration.ofSeconds(2))                // emits 0L after 2s
```

### Generating

```java
Flux.generate(sink -> sink.next(42));

Flux.generate(
    () -> 0,                                 // state supplier
    (state, sink) -> {
        sink.next(state);
        if (state == 10) sink.complete();
        return state + 1;
    }
);
```

### Programmatic (Sink / Processor)

```java
Sinks.Many<String> sink = Sinks.many().multicast().onBackpressureBuffer();
Flux<String> flux = sink.asFlux();
sink.tryEmitNext("Hello");
sink.tryEmitComplete();

// Replay
Sinks.Many<String> replay = Sinks.many().replay().all();

// Unicast (single subscriber)
Sinks.Many<String> unicast = Sinks.many().unicast().onBackpressureBuffer();
```

Older API: `FluxProcessor`, `UnicastProcessor`, `EmitterProcessor` — replaced by `Sinks` in Reactor 3.4+.

### Hot vs Cold

- **Cold** — each subscriber triggers a fresh start (`Flux.range`, `Mono.just`)
- **Hot** — emits regardless of subscribers (`ConnectableFlux`, `Sinks`)

```java
ConnectableFlux<Long> hot = Flux.interval(Duration.ofSeconds(1)).publish();
hot.connect();   // starts emitting
hot.subscribe(System.out::println);
Thread.sleep(5000);
hot.subscribe(System.out::println);   // misses earlier emissions
```

---

## 5. Operators

### Transformation

```java
map(x -> x * 2)                        // 1:1 sync transform
flatMap(x -> Mono.just(x + 1))         // async, 1:N
flatMapMany(x -> Flux.just(x, x))      // Mono → Flux
concatMap(x -> Mono.just(x + 1))       // preserves order, sequential
cast(User.class)
defaultIfEmpty(value)
switchIfEmpty(Mono.just(other))
```

### Filtering

```java
filter(x -> x > 5)
distinct()
distinctUntilChanged()
take(3)               // first 3
takeLast(3)           // last 3
takeWhile(x -> x < 5)
skip(2)               // skip first 2
skipLast(2)
skipWhile(x -> x < 5)
sample(Duration.ofSeconds(1))
elementAt(2)
single()
```

### Combining

```java
Flux.merge(a, b)         // interleave, no order
Flux.concat(a, b)        // sequential, preserves order
Flux.zip(a, b)           // pair elements
Mono.zip(m1, m2)         // combine Monos
Flux.combineLatest(a, b) // emit when either emits
a.startWith(0)           // prepend
a.concatWith(b)
```

### Aggregation

```java
count()
reduce(0, Integer::sum)
collectList()                        // Flux<T> → Mono<List<T>>
collectMap(User::getId, u -> u)
collectMultimap(User::getName)
```

### Side Effects

```java
doOnNext(v -> log.info("Emitted: {}", v))
doOnSubscribe(s -> log.info("Subscribed"))
doOnComplete(() -> log.info("Done"))
doOnError(e -> log.error("Failed", e))
doFinally(signal -> log.info("Finally: {}", signal))
doOnCancel(() -> log.info("Cancelled"))
```

### Utility

```java
log()                        // logs all signals
timeout(Duration.ofSeconds(5))
retry(3)                     // retry up to 3 times
retryWhen(Retry.backoff(3, Duration.ofMillis(100)))
repeat(3)
cache()                      // cache and replay
delayElements(Duration.ofMillis(100))
delaySubscription(Duration.ofSeconds(1))
subscribeOn(scheduler)
publishOn(scheduler)
```

### Operator Chain Example

```java
Flux.just("apple", "banana", "cherry")
    .filter(f -> f.length() > 5)
    .map(String::toUpperCase)
    .map(s -> s + "!")
    .doOnNext(s -> log.info("After map: {}", s))
    .subscribe(result -> log.info("Result: {}", result));
```

Output:
```
After map: BANANA!
Result: BANANA!
After map: CHERRY!
Result: CHERRY!
```

### Mono.zip Example

```java
Mono<User> user = userRepo.findById(1L);
Mono<Order> order = orderClient.getOrder(1L);
Mono<Invoice> invoice = invoiceClient.getInvoice(1L);

Mono<Dashboard> dashboard = Mono.zip(user, order, invoice)
    .map(t -> new Dashboard(t.getT1(), t.getT2(), t.getT3()));
```

### flatMap vs concatMap vs flatMapSequential

| Operator | Order | Concurrency |
|----------|-------|-------------|
| `flatMap` | Unordered | Parallel |
| `concatMap` | Preserved | Sequential |
| `flatMapSequential` | Preserved | Parallel, buffered |

```java
Flux.just(1, 2, 3)
    .flatMap(i -> Mono.just(i * 10)
        .delayElement(Duration.ofMillis(100 - i * 30)))
    .subscribe(System.out::println);
// May print out of order: 30, 20, 10

Flux.just(1, 2, 3)
    .concatMap(i -> Mono.just(i * 10)
        .delayElement(Duration.ofMillis(100 - i * 30)))
    .subscribe(System.out::println);
// Always: 10, 20, 30
```

---

## 6. Backpressure

Producer emits faster than consumer handles → overwhelmed. Backpressure is the mechanism to slow the producer.

### Strategies

| Strategy | Behavior |
|----------|----------|
| `BUFFER` (default) | Buffer all (unbounded) |
| `DROP` | Drop new items when full |
| `LATEST` | Keep only latest |
| `ERROR` | Signal `OverflowException` |

```java
Flux.range(1, 1_000_000)
    .onBackpressureBuffer(1000)         // bounded buffer
    .subscribe(...);

Flux.range(1, 1_000_000)
    .onBackpressureDrop(System.out::println)
    .subscribe(...);

Flux.range(1, 1_000_000)
    .onBackpressureLatest()
    .subscribe(...);

Flux.range(1, 1_000_000)
    .onBackpressureError()
    .subscribe(...);
```

### Custom Subscriber

```java
flux.subscribe(new BaseSubscriber<Integer>() {
    @Override
    protected void hookOnSubscribe(Subscription subscription) {
        request(1);   // demand 1
    }
    
    @Override
    protected void hookOnNext(Integer value) {
        process(value);
        request(1);   // demand next
    }
});
```

### Operators That Handle Backpressure

- `limitRate(n)` — request n at a time
- `buffer(n)` — collect n items
- `window(n)` — split into windows
- `sample(duration)` — take latest per interval
- `throttle` — rate limit

### Real-World

Most HTTP responses don't need explicit backpressure — TCP flow control + `request(Long.MAX_VALUE)` from framework handle it. Backpressure matters for:
- Reactive DB streams
- Message consumers
- File/stream processing
- Slow downstream

---

## 7. Error Handling in Reactor

### onErrorReturn

Replace error with a fallback value.

```java
userService.find(id)
    .onErrorReturn(new User("Default"))
```

### onErrorResume

Replace error with another publisher.

```java
userService.find(id)
    .onErrorResume(e -> cacheService.get(id))
    .onErrorResume(TimeoutException.class, e -> Mono.empty())
```

### onErrorMap

Transform exception type.

```java
userService.find(id)
    .onErrorMap(IOException.class, e -> new ServiceException("Fetch failed", e))
```

### doOnError

Side effect on error.

```java
userService.find(id)
    .doOnError(e -> log.error("Failed to find user", e))
```

### Retry

```java
userService.find(id)
    .retry(3)                                     // immediate, 3 attempts

userService.find(id)
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
        .maxBackoff(Duration.ofSeconds(2))
        .filter(e -> e instanceof IOException)
        .onRetryExhaustedThrow((spec, signal) -> new ServiceException("All retries failed")));
```

### Timeout

```java
userService.find(id)
    .timeout(Duration.ofSeconds(2))
    .onErrorResume(TimeoutException.class, e -> Mono.empty());
```

### Combining

```java
userService.find(id)
    .timeout(Duration.ofSeconds(2))
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100)))
    .onErrorResume(e -> Mono.just(fallbackUser(id)))
    .doOnError(e -> log.error("All failed", e))
    .onErrorReturn(defaultUser);
```

### try-catch in Reactive Code

**Wrong:**
```java
Mono<User> user = Mono.fromCallable(() -> userRepo.find(id))   // exception thrown here?
```
`Mono.fromCallable` catches thrown exceptions automatically.

**Wrong inside operators:**
```java
mono.map(u -> {
    if (u == null) throw new IllegalStateException();   // ⚠️ might escape
    return u;
})
```

**Right:**
```java
mono.flatMap(u -> u == null
    ? Mono.error(new IllegalStateException())
    : Mono.just(u))
```

Or `handle`:

```java
mono.handle((u, sink) -> {
    if (u == null) sink.error(new IllegalStateException());
    else sink.next(u);
});
```

### Global Error Handling (WebFlux)

```java
@Component
@Order(-2)
public class GlobalErrorWebExceptionHandler implements ErrorWebExceptionHandler {
    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);
        byte[] body = ("{\"error\":\"" + ex.getMessage() + "\"}").getBytes();
        return response.writeWith(Mono.just(response.bufferFactory().wrap(body)));
    }
}
```

---

## 8. Schedulers & Threading

### Schedulers

| Scheduler | Use |
|-----------|-----|
| `Schedulers.immediate()` | Current thread |
| `Schedulers.single()` | Single reusable thread |
| `Schedulers.parallel()` | Fixed pool (CPU cores) — CPU-bound |
| `Schedulers.boundedElastic()` | Elastic pool — blocking I/O |
| `Schedulers.fromExecutor(exec)` | Custom |
| `Schedulers.newParallel(name, n)` | Fresh pool |

### subscribeOn vs publishOn

- **subscribeOn** — affects where subscription (and upstream work) happens
- **publishOn** — affects where downstream operators execute

```java
Flux.range(1, 10)
    .subscribeOn(Schedulers.boundedElastic())    // source runs on this scheduler
    .publishOn(Schedulers.parallel())            // everything after runs on parallel
    .map(this::cpuHeavyWork)
    .publishOn(Schedulers.single())              // subsequent ops on single
    .subscribe(this::saveResult);
```

### Blocking Calls

Never block on a non-blocking scheduler. Wrap:

```java
Mono.fromCallable(() -> jdbcTemplate.query(...))   // blocking
    .subscribeOn(Schedulers.boundedElastic())       // offload to elastic
```

### Reactor Context

Thread-local replacement for reactive pipelines.

```java
String userId = "abc";

Mono.just("data")
    .flatMap(v -> Mono.deferContextual(ctx -> {
        String user = ctx.get("userId");
        return Mono.just(user + ": " + v);
    }))
    .contextWrite(Context.of("userId", userId))
    .subscribe(System.out::println);
```

### MDC Logging (Correlation ID)

MDC is ThreadLocal — doesn't work across reactive threads. Use Reactor Context + a custom operator or Micrometer Tracing's reactive support.

```java
// Micrometer Tracing auto-populates traceId/spanId in Reactor context
```

Or manually:

```java
mono.contextWrite(ctx -> ctx.put("traceId", UUID.randomUUID().toString()))
    .doOnEach(signal -> {
        String traceId = signal.getContextView().get("traceId");
        // include in log
    });
```

---

## 9. Spring WebFlux — Annotated Controllers

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

Pulls in: `spring-webflux`, `reactor-core`, `reactor-netty-http`, Jackson.

### Minimal Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @GetMapping
    public Flux<User> list() {
        return userService.findAll();
    }
    
    @GetMapping("/{id}")
    public Mono<ResponseEntity<User>> get(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<User> create(@RequestBody @Valid Mono<CreateUserRequest> req) {
        return req.flatMap(userService::create);
    }
}
```

### Return Types

| Return Type | Meaning |
|-------------|---------|
| `Mono<T>` | 0..1 body |
| `Flux<T>` | 0..N body (stream) |
| `Mono<ResponseEntity<T>>` | Full response control |
| `Mono<Void>` | No body |
| `Flux<ServerSentEvent<T>>` | SSE |

### Annotations (Same as WebMVC)

```java
@GetMapping, @PostMapping, @PutMapping, @PatchMapping, @DeleteMapping
@RequestMapping
@PathVariable, @RequestParam, @RequestBody, @RequestHeader, @CookieValue
@ResponseStatus
@Valid
```

### Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse notFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }
    
    @ExceptionHandler(WebExchangeBindException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse validation(WebExchangeBindException ex) {
        return new ErrorResponse("VALIDATION", ex.getMessage());
    }
}
```

Same `@ControllerAdvice` pattern as MVC.

### Request Body as Mono

```java
@PostMapping
public Mono<User> create(@RequestBody Mono<CreateUserRequest> req) {
    return req.flatMap(userService::create);
}
```

This avoids reading the body eagerly if you plan to do more async work.

### Validation

```java
public class CreateUserRequest {
    @NotBlank private String name;
    @Email private String email;
}

@PostMapping
public Mono<User> create(@Valid @RequestBody Mono<CreateUserRequest> req) { ... }
```

Spring validates when the Mono is resolved.

---

## 10. Spring WebFlux — Functional Endpoints

Alternative to annotations. Uses `RouterFunction` + `HandlerFunction`.

### Router

```java
@Configuration
public class UserRouter {
    
    @Bean
    public RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
        return RouterFunctions.route()
            .GET("/api/users", handler::list)
            .GET("/api/users/{id}", handler::get)
            .POST("/api/users", handler::create)
            .DELETE("/api/users/{id}", handler::delete)
            .build();
    }
}
```

### Handler

```java
@Component
public class UserHandler {
    
    private final UserService userService;
    
    public UserHandler(UserService userService) { this.userService = userService; }
    
    public Mono<ServerResponse> list(ServerRequest req) {
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(userService.findAll(), User.class);
    }
    
    public Mono<ServerResponse> get(ServerRequest req) {
        Long id = Long.valueOf(req.pathVariable("id"));
        return userService.findById(id)
            .flatMap(u -> ServerResponse.ok().bodyValue(u))
            .switchIfEmpty(ServerResponse.notFound().build());
    }
    
    public Mono<ServerResponse> create(ServerRequest req) {
        return req.bodyToMono(CreateUserRequest.class)
            .flatMap(userService::create)
            .flatMap(u -> ServerResponse.created(
                URI.create("/api/users/" + u.getId())).bodyValue(u));
    }
    
    public Mono<ServerResponse> delete(ServerRequest req) {
        Long id = Long.valueOf(req.pathVariable("id"));
        return userService.delete(id)
            .then(ServerResponse.noContent().build());
    }
}
```

### Nested Routes

```java
RouterFunctions.route()
    .path("/api", builder -> builder
        .path("/users", b -> b
            .GET("", handler::list)
            .GET("/{id}", handler::get)
            .POST("", handler::create)
            .DELETE("/{id}", handler::delete))
        .path("/orders", b -> b
            .GET("", orderHandler::list)
            .GET("/{id}", orderHandler::get)))
    .build();
```

### Predicates

```java
GET("/users").and(accept(MediaType.APPLICATION_JSON))
GET("/users").and(header("X-Version", "v1"))
```

### Filters

```java
@Bean
public RouterFunction<ServerResponse> routes(UserHandler handler) {
    return RouterFunctions.route()
        .GET("/api/users", handler::list)
        .filter((req, next) -> {
            log.info("Request: {}", req.path());
            return next.handle(req);
        })
        .build();
}
```

### When to Use Functional

- Programmatic routing (dynamic)
- Cleaner composition
- Kotlin-friendly (DSL)
- No annotation processing

### When to Use Annotated

- Team familiar with MVC
- REST conventions
- Reuse existing MVC knowledge

---

## 11. WebClient

Reactive HTTP client. Replaces `RestTemplate`.

### Creating

```java
// From scratch
WebClient client = WebClient.create("https://api.example.com");

// Via builder (recommended)
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder
        .baseUrl("https://api.example.com")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .defaultHeader(HttpHeaders.USER_AGENT, "my-app")
        .filter(ExchangeFilterFunction.ofRequestProcessor(req -> {
            log.info("→ {} {}", req.method(), req.url());
            return Mono.just(req);
        }))
        .build();
}
```

### GET

```java
Mono<User> user = client.get()
    .uri("/users/{id}", id)
    .retrieve()
    .bodyToMono(User.class);

Flux<User> users = client.get()
    .uri("/users")
    .retrieve()
    .bodyToFlux(User.class);
```

### POST

```java
Mono<User> created = client.post()
    .uri("/users")
    .contentType(MediaType.APPLICATION_JSON)
    .bodyValue(new CreateUserRequest("Alice"))
    .retrieve()
    .bodyToMono(User.class);
```

### With Response Status

```java
Mono<ResponseEntity<User>> res = client.get()
    .uri("/users/{id}", id)
    .retrieve()
    .toEntity(User.class);

res.subscribe(r -> log.info("Status: {}", r.getStatusCode()));
```

### Error Handling

```java
client.get()
    .uri("/users/{id}", id)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, response ->
        response.bodyToMono(ErrorResponse.class)
            .flatMap(err -> Mono.error(new ClientException(err))))
    .onStatus(HttpStatusCode::is5xxServerError, response ->
        Mono.error(new ServerException()))
    .bodyToMono(User.class);
```

### Exchange (Full Control)

```java
Mono<ClientResponse> res = client.get()
    .uri("/users/{id}", id)
    .exchangeToMono(response -> {
        if (response.statusCode().is2xxSuccessful()) {
            return response.bodyToMono(User.class);
        } else {
            return response.createException().flatMap(Mono::error);
        }
    });
```

Use `exchangeToMono` (recommended) or `exchange` (older, beware memory leaks).

### Timeouts

```java
Mono<User> user = client.get().uri("/users/1")
    .retrieve().bodyToMono(User.class)
    .timeout(Duration.ofSeconds(5));
```

Or via HttpClient:

```java
HttpClient httpClient = HttpClient.create()
    .responseTimeout(Duration.ofSeconds(5))
    .doOnConnected(conn -> conn
        .addHandlerLast(new ReadTimeoutHandler(5))
        .addHandlerLast(new WriteTimeoutHandler(5)));

WebClient client = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .build();
```

### Retry

```java
client.get().uri("/users/1")
    .retrieve().bodyToMono(User.class)
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100)));
```

### Filter (Global)

```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder
        .filter((req, next) -> {
            log.info("Request: {} {}", req.method(), req.url());
            return next.exchange(req);
        })
        .filter(ExchangeFilterFunctions.basicAuthentication("user", "pwd"))
        .build();
}
```

### OAuth2 Client Credentials

```java
@Bean
public WebClient webClient(OAuth2AuthorizedClientManager mgr) {
    ServletOAuth2AuthorizedClientExchangeFilterFunction f =
        new ServletOAuth2AuthorizedClientExchangeFilterFunction(mgr);
    f.setDefaultClientRegistrationId("service");
    return WebClient.builder().filter(f).build();
}
```

### WebClient in MVC

You can use WebClient in a WebMVC app — but `.block()` on every call defeats the purpose. Use RestTemplate or RestClient (Spring 6.1+) in blocking apps.

### WebClient vs RestTemplate

| Feature | WebClient | RestTemplate |
|---------|-----------|--------------|
| Reactive | ✅ | ❌ |
| Blocking | ✅ (block) | ✅ |
| Streaming | ✅ | ❌ |
| Composition | ✅ | ❌ |
| Status | Current | Maintenance (use RestClient in 6.1+) |

### RestClient (Spring 6.1+)

Modern blocking client replacing `RestTemplate`:

```java
RestClient client = RestClient.create("https://api.example.com");
User user = client.get().uri("/users/1")
    .retrieve().body(User.class);
```

Fluent, synchronous, with better defaults than RestTemplate.

---

## 12. R2DBC (Reactive SQL)

**Reactive Relational Database Connectivity** — non-blocking SQL.

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

### Configuration

```properties
spring.r2dbc.url=r2dbc:postgresql://localhost:5432/mydb
spring.r2dbc.username=admin
spring.r2dbc.password=secret
spring.r2dbc.pool.enabled=true
spring.r2dbc.pool.initial-size=5
spring.r2dbc.pool.max-size=20
```

### Entity

```java
@Table("users")
public class User {
    @Id private Long id;
    private String name;
    private String email;
    // getters/setters
}
```

### Repository

```java
public interface UserRepository extends ReactiveCrudRepository<User, Long> {
    Flux<User> findByNameContaining(String part);
    Mono<User> findByEmail(String email);
    
    @Query("SELECT * FROM users WHERE age > :age")
    Flux<User> findOlderThan(int age);
}
```

### Controller

```java
@RestController
public class UserController {
    @Autowired private UserRepository repo;
    
    @GetMapping("/users")
    public Flux<User> list() { return repo.findAll(); }
    
    @GetMapping("/users/{id}")
    public Mono<User> get(@PathVariable Long id) { return repo.findById(id); }
    
    @PostMapping("/users")
    public Mono<User> create(@RequestBody User user) { return repo.save(user); }
}
```

### Reactive Transactions

```java
@Transactional
public Mono<User> createWithProfile(CreateUserRequest req) {
    return userRepo.save(new User(req.name()))
        .flatMap(u -> profileRepo.save(new Profile(u.getId()))
            .thenReturn(u));
}
```

Uses `ReactiveTransactionManager` + Reactor context.

### R2DBC vs JDBC/JPA

| Feature | R2DBC | JDBC/JPA |
|---------|-------|----------|
| Non-blocking | ✅ | ❌ |
| Reactive | ✅ | ❌ |
| ORM | ❌ | ✅ |
| Relationships | Manual | ORM |
| Maturity | Younger | Very mature |
| Driver support | Postgres, MySQL, MSSQL, H2 | All |

### When to Use

- Fully reactive stacks (WebFlux + R2DBC + Redis)
- High-concurrency microservices
- Streaming large result sets

### When NOT to Use

- Complex ORM features needed → JPA
- Team unfamiliar with reactive
- Simple CRUD — JDBC/JPA is simpler

---

## 13. Reactive MongoDB & Redis

### MongoDB Reactive

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb-reactive</artifactId>
</dependency>
```

```properties
spring.data.mongodb.uri=mongodb://localhost:27017/mydb
```

```java
@Document
public class User {
    @Id private String id;
    private String name;
}

public interface UserRepository extends ReactiveMongoRepository<User, String> {
    Flux<User> findByNameContaining(String part);
}

// Use
Flux<User> users = repo.findByNameContaining("Ali");
```

### Redis Reactive

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

```java
@Autowired private ReactiveRedisTemplate<String, String> redis;

public Mono<Void> set(String key, String value) {
    return redis.opsForValue().set(key, value).then();
}

public Mono<String> get(String key) {
    return redis.opsForValue().get(key);
}
```

### Reactive Caching

```java
@Bean
public ReactiveCacheManager cacheManager(RedisConnectionFactory cf) {
    return RedisCacheManager.create(cf);  // reactive variant
}
```

```java
@Cacheable("users")
public Mono<User> find(Long id) { ... }
```

Cache annotations work with Mono/Flux returns.

### Reactive Cassandra

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-cassandra-reactive</artifactId>
</dependency>
```

Same pattern: `ReactiveCassandraRepository`.

### Reactive Streams Adapters

Spring provides `ReactiveAdapterRegistry` for converting between `Mono/Flux`, `CompletableFuture`, JDK `Flow`, RxJava, Kotlin coroutines.

---

## 14. Server-Sent Events (SSE)

### What is SSE?

Unidirectional server→client streaming over HTTP. Client uses `EventSource`.

### WebFlux SSE Endpoint

```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> events() {
    return Flux.interval(Duration.ofSeconds(1))
        .map(i -> ServerSentEvent.<String>builder()
            .id(String.valueOf(i))
            .event("tick")
            .data("Tick " + i)
            .build());
}
```

### Client

```javascript
const es = new EventSource('/events');
es.addEventListener('tick', e => console.log(e.data));
es.onerror = err => console.error(err);
```

### Stream From DB

```java
@GetMapping(value = "/users/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<User> streamUsers() {
    return userRepo.findAll();   // reactive driver streams results
}
```

### With Keep-Alive

```java
return Flux.interval(Duration.ofSeconds(1))
    .map(i -> ServerSentEvent.builder("data " + i).build())
    .mergeWith(Flux.never());   // prevents completion
```

### Backpressure & Slow Clients

Spring handles backpressure via TCP. Configure:

```properties
spring.webflux.base-path=/
```

### SSE vs WebSocket

| Feature | SSE | WebSocket |
|---------|-----|-----------|
| Direction | Server → client | Bidirectional |
| Protocol | HTTP | WebSocket |
| Reconnect | Built-in | Manual |
| Binary | ❌ | ✅ |
| Complexity | Simple | More complex |

### When to Use SSE

- Live feeds (notifications, prices)
- Progress updates
- Log tails
- One-way streaming

---

## 15. WebSocket

### Config

```java
@Configuration
public class WebSocketConfig {
    
    @Bean
    public HandlerMapping webSocketMapping(MyWebSocketHandler handler) {
        Map<String, WebSocketHandler> map = Map.of("/ws", handler);
        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setUrlMap(map);
        mapping.setOrder(-1);
        return mapping;
    }
    
    @Bean
    public WebSocketHandlerAdapter handlerAdapter() {
        return new WebSocketHandlerAdapter();
    }
}
```

### Handler

```java
@Component
public class MyWebSocketHandler implements WebSocketHandler {
    
    private final Sinks.Many<String> sink = Sinks.many().multicast().onBackpressureBuffer();
    
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        Mono<Void> input = session.receive()
            .map(WebSocketMessage::getPayloadAsText)
            .doOnNext(msg -> log.info("Received: {}", msg))
            .then();
        
        Mono<Void> output = session.send(
            sink.asFlux().map(session::textMessage));
        
        return Mono.zip(input, output).then();
    }
    
    public void broadcast(String msg) {
        sink.tryEmitNext(msg);
    }
}
```

### Reactive STOMP

For pub/sub patterns:

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketStompConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").withSockJS();
    }
}

@Controller
public class ChatController {
    @MessageMapping("/chat")
    @SendTo("/topic/messages")
    public Message send(Message message) { return message; }
}
```

### When to Use WebSocket

- Real-time bidirectional (chat, games)
- High-frequency updates
- Low-latency requirements

For one-way, SSE is simpler.

---

## 16. Testing Reactive Streams

### StepVerifier (Reactor Test)

```xml
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <scope>test</scope>
</dependency>
```

### Basic

```java
@Test
void testFlux() {
    Flux<String> flux = Flux.just("A", "B", "C");
    
    StepVerifier.create(flux)
        .expectNext("A")
        .expectNext("B")
        .expectNext("C")
        .verifyComplete();
}
```

### With Errors

```java
StepVerifier.create(Mono.error(new RuntimeException("Oops")))
    .expectErrorMatches(e -> e.getMessage().equals("Oops"))
    .verify();
```

### With Time

```java
StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofSeconds(1)).take(3))
    .thenAwait(Duration.ofSeconds(3))
    .expectNext(0L, 1L, 2L)
    .verifyComplete();
```

### Expect No Events

```java
StepVerifier.create(Flux.empty())
    .expectComplete()
    .verify();

StepVerifier.create(Mono.never())
    .expectSubscription()
    .expectNoEvent(Duration.ofMillis(100))
    .thenCancel()
    .verify();
```

### Assertions

```java
StepVerifier.create(flux)
    .expectNextCount(5)
    .verifyComplete();

StepVerifier.create(flux.collectList())
    .assertNext(list -> assertThat(list).hasSize(3))
    .verifyComplete();
```

### Mocking Reactive Services

```java
@MockBean
private UserService userService;

@Test
void controllerTest() {
    when(userService.findById(1L)).thenReturn(Mono.just(new User(1L, "Alice")));
    
    webTestClient.get().uri("/api/users/1")
        .exchange()
        .expectStatus().isOk()
        .expectBody().jsonPath("$.name").isEqualTo("Alice");
}
```

### WebTestClient for WebFlux

```java
@WebFluxTest(UserController.class)
class UserControllerTest {
    
    @Autowired WebTestClient client;
    @MockBean UserService userService;
    
    @Test
    void get() {
        when(userService.findById(1L)).thenReturn(Mono.just(new User(1L, "Alice")));
        
        client.get().uri("/api/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody()
            .jsonPath("$.name").isEqualTo("Alice");
    }
    
    @Test
    void stream() {
        when(userService.findAll()).thenReturn(Flux.just(
            new User(1L, "Alice"), new User(2L, "Bob")));
        
        client.get().uri("/api/users")
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(User.class).hasSize(2);
    }
}
```

### Test Schedulers

```java
@Test
void withVirtualTime() {
    StepVerifier.withVirtualTime(() -> 
            Mono.delay(Duration.ofHours(1)).map(t -> "done"))
        .thenAwait(Duration.ofHours(1))
        .expectNext("done")
        .verifyComplete();
}
```

`VirtualTimeScheduler` fast-forwards time — no real waiting.

### Testing Backpressure

```java
StepVerifier.create(flux, 0)      // request 0 initially
    .thenRequest(1)
    .expectNext("A")
    .thenRequest(2)
    .expectNext("B", "C")
    .verifyComplete();
```

---

## 17. WebFlux vs WebMVC

| Aspect | WebMVC | WebFlux |
|--------|--------|---------|
| Model | Blocking (Servlet) | Non-blocking (Reactive) |
| Threads | Thread per request | Event loop |
| API | `HttpServletRequest` | `ServerWebExchange` |
| Return | Objects, `ResponseEntity` | `Mono`, `Flux` |
| Server | Tomcat/Jetty/Undertow | Netty (default) / Tomcat |
| Data | JDBC, JPA | R2DBC, Reactive Mongo |
| Backpressure | N/A | Native |
| Debug | Easy stack traces | Harder |
| Learning curve | Low | Medium |
| Performance (I/O bound) | Good | Excellent |
| Performance (CPU bound) | Good | Same |
| Ecosystem | Mature | Growing |
| Threads scale | ~200/CPU | ~Few per core |

### Both Can Coexist

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

Spring Boot runs **both** — WebMVC on the Servlet stack, WebFlux on the reactive stack if no Servlet container is present or if `spring.main.web-application-type=reactive`.

Actually if both are present, Spring Boot prefers **WebMVC** by default. To force reactive:

```properties
spring.main.web-application-type=reactive
```

### Decision Guide

| Situation | Choose |
|-----------|--------|
| Traditional REST | WebMVC |
| JDBC/JPA backend | WebMVC |
| High-concurrency I/O | WebFlux |
| Streaming (SSE, WebSocket) | WebFlux |
| Reactive DB (Mongo, R2DBC) | WebFlux |
| Team new to reactive | WebMVC |
| Simple CRUD | WebMVC |
| Gateway | WebFlux |

> **"Choose WebMVC unless you have a reason to choose WebFlux."** — Spring Team.

---

## 18. Interview Questions

### Q1. What is reactive programming?

**Answer:** A paradigm for asynchronous, non-blocking data streams with backpressure. You compose operations on streams (Publisher/Subscriber) rather than executing steps imperatively. Nothing happens until subscription.

### Q2. What is Project Reactor?

**Answer:** A Reactive Streams implementation — the foundation of Spring WebFlux. Provides `Mono` (0..1) and `Flux` (0..N) plus hundreds of operators.

### Q3. Difference between Mono and Flux?

**Answer:** `Mono<T>` — emits 0 or 1 element (or error). `Flux<T>` — emits 0..N elements (or error). Use Mono for single async results; Flux for streams.

### Q4. What is backpressure?

**Answer:** A mechanism where the consumer controls the rate of emission via `request(n)`. Prevents overwhelming slow consumers. Strategies: buffer, drop, latest, error.

### Q5. Why is nothing executed until subscribe?

**Answer:** Reactor is lazy — publishers are declarative descriptions. Subscription triggers the pipeline. This enables composition, retry, and resource reuse without side effects.

### Q6. subscribeOn vs publishOn?

**Answer:** `subscribeOn` — where the subscription (and upstream work) runs. `publishOn` — where downstream operators run. You can chain multiple `publishOn` to switch threads.

### Q7. What are Schedulers?

**Answer:** Thread pools for reactive pipelines. `parallel()` (CPU), `boundedElastic()` (blocking I/O), `single()`, `immediate()`, `fromExecutor()`. Choose `boundedElastic` for blocking calls to avoid blocking the event loop.

### Q8. How do you handle errors in Reactor?

**Answer:** `onErrorReturn` (fallback value), `onErrorResume` (fallback publisher), `onErrorMap` (transform), `doOnError` (side effect), `retry`, `retryWhen`, `timeout`. Exceptions inside operators must be converted to `Mono.error(...)`.

### Q9. What is WebFlux?

**Answer:** Spring's reactive web framework. Non-blocking, event-loop based (Netty). Supports annotated (`@RestController`) and functional endpoints. Returns `Mono`/`Flux`. Uses reactive data sources (R2DBC, reactive Mongo/Redis).

### Q10. WebFlux vs WebMVC?

**Answer:** WebMVC = blocking Servlet stack, thread per request, JDBC/JPA. WebFlux = non-blocking reactive stack, event loop, R2DBC/reactive drivers. Use WebFlux for high-concurrency I/O-bound or streaming apps; WebMVC otherwise.

### Q11. What is WebClient?

**Answer:** Reactive HTTP client for WebFlux (and usable in WebMVC). Non-blocking, composable, returns Mono/Flux. Supports retries, timeouts, filters, OAuth2. Replaces `RestTemplate` (which is in maintenance; `RestClient` is the modern blocking replacement in Spring 6.1+).

### Q12. What is R2DBC?

**Answer:** Reactive Relational Database Connectivity — non-blocking SQL. Alternative to JDBC for reactive stacks. Used with Spring Data R2DBC. Doesn't support JPA (no lazy loading, no entities).

### Q13. When should you NOT use WebFlux?

**Answer:** When using blocking drivers (JDBC, JPA), CPU-heavy work, simple CRUD, or when the team lacks reactive experience. Reactive adds complexity without benefit if you're blocking anyway.

### Q14. What is the event loop?

**Answer:** A single-threaded (or few-threaded) loop that handles many concurrent connections. Non-blocking I/O through OS mechanisms (epoll, kqueue). Netty uses it. You must not block the event loop.

### Q15. Can you block in WebFlux?

**Answer:** Not on the event loop. If you must call a blocking API, wrap with `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`. Blocking the event loop kills throughput.

### Q16. How do you test reactive code?

**Answer:** `StepVerifier` (Reactor Test) for stream assertions. `WebTestClient` for WebFlux endpoints. `withVirtualTime` for time-based tests. `@WebFluxTest` for slices.

### Q17. What is Reactor Context?

**Answer:** A per-subscription key-value store replacing ThreadLocal for reactive pipelines. Use `contextWrite(Context.of(...))` and `Mono.deferContextual(...)`. Used for tracing, MDC, security principals.

### Q18. flatMap vs concatMap vs flatMapSequential?

**Answer:** `flatMap` — parallel, unordered. `concatMap` — sequential, preserves order. `flatMapSequential` — parallel, preserves order, buffers. Choose based on concurrency needs.

### Q19. What is the difference between hot and cold publishers?

**Answer:** Cold — each subscriber gets its own execution (e.g., `Flux.range`, HTTP requests). Hot — emits regardless of subscribers (e.g., `Sinks`, `ConnectableFlux`). Cache/`replay` converts cold to hot.

### Q20. How do you stream data from DB in WebFlux?

**Answer:** `Flux<User> users = repo.findAll()` — the reactive driver streams results with backpressure. Spring writes them to the HTTP response as a stream (chunked). No memory explosion for large datasets.

### Q21. What is Sinks?

**Answer:** Reactor 3.4+ API for programmatically emitting into a stream. Replaces `Processor`. Variants: `Sinks.many().multicast()`, `unicast()`, `replay()`. Used for bridging imperative code to reactive.

### Q22. Can WebFlux call WebMVC services?

**Answer:** Yes — over HTTP. But if the downstream is blocking, wrap calls in `boundedElastic` (or use WebClient which is non-blocking on the client side even if the server blocks; server-side blocking is a separate concern).

### Q23. What is Server-Sent Events?

**Answer:** One-way server→client streaming over HTTP (`text/event-stream`). Simpler than WebSocket, auto-reconnect. Spring WebFlux returns `Flux<ServerSentEvent<T>>`.

### Q24. How does WebFlux handle thread-safety?

**Answer:** Code must be non-blocking and safe — no mutable shared state, no ThreadLocals. Use Reactor Context for per-request data. Reactor operators are safe; your lambda code must not mutate shared state.

### Q25. What is Spring's recommendation on WebFlux adoption?

**Answer:** Choose WebFlux when you have a compelling reason: existing reactive stack, high-concurrency I/O, streaming, or a gateway. For most applications, WebMVC is simpler, well-understood, and equally performant.

---

## 19. Cheat Sheet

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
</dependency>
```

### Create Publishers

```java
Mono.just("x")
Mono.empty()
Mono.error(ex)
Mono.fromCallable(() -> ...)
Mono.fromFuture(cf)
Mono.fromSupplier(() -> ...)

Flux.just(1, 2, 3)
Flux.range(1, 10)
Flux.interval(Duration.ofSeconds(1))
Flux.fromIterable(list)
Flux.generate(sink -> ...)
```

### Core Operators

```
map            flatMap       concatMap     flatMapSequential
filter         distinct      take          skip
merge          concat        zip           combineLatest
reduce         collectList   count         collectMap
doOnNext       doOnError     doOnComplete  doFinally
retry          retryWhen     timeout       defaultIfEmpty
onErrorReturn  onErrorResume onErrorMap    cache
subscribeOn    publishOn     delayElement  buffer
```

### Error Handling

```java
mono.onErrorReturn(fallback)
    .onErrorResume(e -> alternative())
    .onErrorMap(IOException.class, e -> new ServiceException(e))
    .doOnError(e -> log.error("Failed", e))
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100)))
    .timeout(Duration.ofSeconds(5));
```

### Schedulers

```java
Schedulers.immediate()
Schedulers.single()
Schedulers.parallel()          // CPU
Schedulers.boundedElastic()    // blocking I/O
Schedulers.fromExecutor(exec)

mono.subscribeOn(Schedulers.boundedElastic())
    .publishOn(Schedulers.parallel())
```

### WebFlux Controller

```java
@RestController
public class UserController {
    @GetMapping("/users")
    public Flux<User> list() { return repo.findAll(); }
    
    @GetMapping("/users/{id}")
    public Mono<User> get(@PathVariable Long id) { return repo.findById(id); }
    
    @PostMapping("/users")
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<User> create(@Valid @RequestBody Mono<CreateUserRequest> req) {
        return req.flatMap(service::create);
    }
}
```

### Functional Router

```java
@Bean
public RouterFunction<ServerResponse> routes(UserHandler h) {
    return RouterFunctions.route()
        .GET("/users", h::list)
        .GET("/users/{id}", h::get)
        .POST("/users", h::create)
        .build();
}
```

### WebClient

```java
Mono<User> user = webClient.get()
    .uri("/users/{id}", id)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, r ->
        r.bodyToMono(ErrorResponse.class).flatMap(e -> Mono.error(new ClientError(e))))
    .bodyToMono(User.class)
    .timeout(Duration.ofSeconds(5))
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100)));
```

### SSE Endpoint

```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> events() {
    return Flux.interval(Duration.ofSeconds(1))
        .map(i -> ServerSentEvent.builder("tick " + i).event("tick").build());
}
```

### R2DBC

```properties
spring.r2dbc.url=r2dbc:postgresql://localhost:5432/mydb
spring.r2dbc.username=admin
spring.r2dbc.password=secret
spring.r2dbc.pool.enabled=true
```

```java
public interface UserRepository extends ReactiveCrudRepository<User, Long> {
    Flux<User> findByName(String name);
}
```

### Testing

```java
StepVerifier.create(flux)
    .expectNext("A", "B")
    .verifyComplete();

StepVerifier.create(Mono.error(new RuntimeException()))
    .expectError(RuntimeException.class)
    .verify();

StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofHours(1)).take(2))
    .thenAwait(Duration.ofHours(2))
    .expectNext(0L, 1L)
    .verifyComplete();
```

```java
@WebFluxTest(UserController.class)
class UserControllerTest {
    @Autowired WebTestClient client;
    @MockBean UserService service;
    
    @Test void test() {
        when(service.findById(1L)).thenReturn(Mono.just(new User(1L, "A")));
        client.get().uri("/api/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody().jsonPath("$.name").isEqualTo("A");
    }
}
```

### Reactor Context

```java
Mono.deferContextual(ctx -> Mono.just(ctx.get("userId")))
    .contextWrite(Context.of("userId", "abc"))
    .subscribe(System.out::println);
```

### Hot vs Cold

```
Cold: Mono.just, Flux.range, HTTP calls (per subscriber)
Hot:  Sinks, ConnectableFlux, .publish() (shared)

conversion:
  cold.publish() → ConnectableFlux (hot)
  cold.cache() → hot cache
  cold.share() → hot multicast
```

### WebFlux vs WebMVC Decision

```
Choose WebFlux when:
  ✅ Reactive DB (R2DBC, Mongo, Cassandra)
  ✅ High-concurrency I/O
  ✅ Streaming (SSE, WebSocket)
  ✅ Gateway
  ✅ Team comfortable with reactive

Choose WebMVC when:
  ✅ JDBC/JPA
  ✅ Simple CRUD
  ✅ CPU-bound
  ✅ Team unfamiliar with reactive
```

### Pitfalls

- ❌ Blocking on the event loop (`.block()`, JDBC, `Thread.sleep`, `synchronized` on event-loop thread)
- ❌ Mutable shared state
- ❌ ThreadLocals (use Reactor Context)
- ❌ Ignoring exceptions in operators (they may not propagate)
- ❌ Subscribing manually inside controllers (framework subscribes)
- ❌ Using WebFlux with JDBC (defeats purpose)
- ❌ Forgetting `Schedulers.boundedElastic()` around blocking calls
- ❌ Not handling `null` (Reactor disallows null values)
- ❌ Infinite `Flux.interval` without termination

### Cross-References

- **Previous:** `19_Spring_Microservices_Cloud.md`
- **Next:** `21_Spring_Batch.md`
- **Related:** `03_Spring_MVC.md`, `11_Spring_Data_JDBC.md`, `12_Spring_Caching.md`
- **Interview:** `24_Spring_Interview_Questions.md` (§WebFlux)

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [19_Spring_Microservices_Cloud.md](./19_Spring_Microservices_Cloud.md)
- **Next →:** [21_Spring_Batch.md](./21_Spring_Batch.md)
- **Related:** [03_Spring_MVC.md](./03_Spring_MVC.md), [19_Spring_Microservices_Cloud.md](./19_Spring_Microservices_Cloud.md), [11_Spring_Data_JDBC.md](./11_Spring_Data_JDBC.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
