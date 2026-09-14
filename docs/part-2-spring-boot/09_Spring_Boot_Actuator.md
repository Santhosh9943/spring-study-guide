# Spring Boot Actuator

> **File:** `09_Spring_Boot_Actuator.md`
> **Part:** 2 — Spring Boot
> **Prerequisites:** `06_Spring_Boot_Fundamentals.md`, `08_Spring_Boot_Configuration.md`
> **Estimated Study Time:** 6–8 hours

---

## Table of Contents

1. [What is Actuator?](#1-what-is-actuator)
2. [Enabling Actuator](#2-enabling-actuator)
3. [Built-In Endpoints](#3-built-in-endpoints)
4. [Health Indicators](#4-health-indicators)
5. [Info Endpoint](#5-info-endpoint)
6. [Metrics with Micrometer](#6-metrics-with-micrometer)
7. [Prometheus & Grafana](#7-prometheus--grafana)
8. [Custom Metrics](#8-custom-metrics)
9. [Custom Endpoints](#9-custom-endpoints)
10. [Securing Actuator](#10-securing-actuator)
11. [Production Best Practices](#11-production-best-practices)
12. [Interview Questions](#12-interview-questions)
13. [Cheat Sheet](#13-cheat-sheet)

---

## 1. What is Actuator?

**Spring Boot Actuator** provides production-ready features for monitoring, managing, and troubleshooting applications. It exposes **HTTP endpoints** (or JMX) that report:

- **Health** — Is the app up? Are dependencies reachable?
- **Metrics** — CPU, memory, HTTP requests, custom KPIs
- **Info** — Build metadata, git commit
- **Environment** — Properties, active profiles
- **Beans** — All Spring beans, their dependencies
- **Mappings** — All HTTP endpoints
- **Loggers** — View/change log levels at runtime
- **Thread dump**, **heap dump**
- **Conditions** — Why auto-config applied or didn't
- **Scheduled tasks**, **HTTP traces**, **caches**

### Why It Matters

- **Observability** — real-time insight into running apps
- **Health checks** — Kubernetes liveness/readiness probes
- **Metrics** — feed Prometheus, Datadog, New Relic
- **Debugging** — without restarting the app
- **Standard interface** — same endpoints across all Spring Boot apps

---

## 2. Enabling Actuator

### Add Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Pulls in:
- `spring-boot-actuator`
- `spring-boot-actuator-autoconfigure`
- `micrometer-core` (metrics façade)

### Default Behavior

- `/actuator/health` and `/actuator/info` exposed over HTTP by default
- All others are available but **not exposed**
- Base path: `/actuator`
- Expose endpoints via properties

### Expose Endpoints

```properties
# Expose specific endpoints
management.endpoints.web.exposure.include=health,info,metrics,env,beans

# Expose everything (dev only!)
management.endpoints.web.exposure.include=*

# Exclude specific
management.endpoints.web.exposure.exclude=env,beans,heapdump
```

### Custom Base Path

```properties
management.endpoints.web.base-path=/manage
# → /manage/health, /manage/metrics
```

### Separate Port

```properties
management.server.port=9001
management.server.address=127.0.0.1
```

Runs Actuator on a different port — useful for isolating from public traffic.

### Custom Context Path

```properties
management.server.base-path=/actuator
```

### JMX Instead of HTTP

```properties
management.endpoints.jmx.exposure.include=*
spring.jmx.enabled=true
```

Access via JConsole, VisualVM, or JMX clients.

---

## 3. Built-In Endpoints

### Endpoint Reference

| ID | Path | Purpose |
|----|------|---------|
| `auditevents` | `/actuator/auditevents` | Audit events |
| `beans` | `/actuator/beans` | All beans |
| `caches` | `/actuator/caches` | Cache managers |
| `conditions` | `/actuator/conditions` | Auto-config conditions |
| `configprops` | `/actuator/configprops` | `@ConfigurationProperties` |
| `env` | `/actuator/env` | Environment properties |
| `flyway` | `/actuator/flyway` | Flyway migrations |
| `health` | `/actuator/health` | Aggregate health |
| `heapdump` | `/actuator/heapdump` | Heap dump file |
| `httpexchanges` | `/actuator/httpexchanges` | HTTP request traces |
| `info` | `/actuator/info` | App info |
| `integrationgraph` | `/actuator/integrationgraph` | Spring Integration graph |
| `jolokia` | `/actuator/jolokia` | JMX over HTTP |
| `liquibase` | `/actuator/liquibase` | Liquibase migrations |
| `logfile` | `/actuator/logfile` | Log file content |
| `loggers` | `/actuator/loggers` | Log levels |
| `mappings` | `/actuator/mappings` | Request mappings |
| `metrics` | `/actuator/metrics` | Metrics list |
| `prometheus` | `/actuator/prometheus` | Prometheus format |
| `quartz` | `/actuator/quartz` | Quartz jobs |
| `scheduledtasks` | `/actuator/scheduledtasks` | Scheduled tasks |
| `sessions` | `/actuator/sessions` | HTTP sessions |
| `shutdown` | `/actuator/shutdown` | Graceful shutdown (disabled) |
| `startup` | `/actuator/startup` | Startup steps |
| `threaddump` | `/actuator/threaddump` | Thread dump |

### Health

```bash
curl http://localhost:8080/actuator/health
```

```json
{"status":"UP"}
```

With `show-details=always`:

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": { "database": "PostgreSQL", "validationQuery": "isValid()" }
    },
    "diskSpace": {
      "status": "UP",
      "details": { "total": 500000000000, "free": 300000000000 }
    },
    "ping": { "status": "UP" }
  }
}
```

### Beans

```bash
curl http://localhost:8080/actuator/beans
```

Lists all beans, their dependencies, scope, type.

### Env

```bash
curl http://localhost:8080/actuator/env
```

Lists property sources and their values (secrets redacted by default).

### Mappings

```bash
curl http://localhost:8080/actuator/mappings
```

All `@RequestMapping` endpoints, filters, servlets.

### Loggers

```bash
# List all loggers
curl http://localhost:8080/actuator/loggers

# View specific
curl http://localhost:8080/actuator/loggers/com.example

# Change log level at runtime
curl -X POST http://localhost:8080/actuator/loggers/com.example \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel":"DEBUG"}'
```

### Thread Dump

```bash
curl http://localhost:8080/actuator/threaddump
```

JSON thread dump — same as `jstack`.

### Heap Dump

```bash
curl http://localhost:8080/actuator/heapdump -o heap.hprof
```

Downloads an HPROF file for analysis in Eclipse MAT or VisualVM.

### Conditions

```bash
curl http://localhost:8080/actuator/conditions
```

Shows which auto-configs applied (and why), which didn't, and the reasons.

### Scheduled Tasks

```bash
curl http://localhost:8080/actuator/scheduledtasks
```

All `@Scheduled` methods with cron/fixedRate/fixedDelay.

### Caches

```bash
curl http://localhost:8080/actuator/caches
```

Lists caches (name, target). Supports `DELETE /actuator/caches/{name}` to clear.

### Shutdown (Disabled by Default)

```properties
management.endpoint.shutdown.enabled=true
```

```bash
curl -X POST http://localhost:8080/actuator/shutdown
```

⚠️ **Never expose in production.**

---

## 4. Health Indicators

### Aggregate Health

Spring Boot aggregates all `HealthIndicator` beans. If any is DOWN → aggregate DOWN.

### Built-In Health Indicators

| Indicator | Requires | Checks |
|-----------|----------|--------|
| `DataSourceHealthIndicator` | `DataSource` | DB connectivity |
| `DiskSpaceHealthIndicator` | Always | Free disk space |
| `MongoHealthIndicator` | MongoDB | MongoDB ping |
| `RedisHealthIndicator` | Redis | Redis ping |
| `CassandraHealthIndicator` | Cassandra | Cassandra |
| `ElasticsearchRestHealthIndicator` | Elasticsearch | ES ping |
| `RabbitHealthIndicator` | RabbitMQ | Rabbit connection |
| `KafkaHealthIndicator` | Kafka | Broker metadata |
| `MailHealthIndicator` | JavaMail | SMTP connection |
| `LdapHealthIndicator` | LDAP | LDAP |
| `PingHealthIndicator` | Always | Always UP (unless down) |
| `JmsHealthIndicator` | JMS | JMS broker |
| `Neo4jHealthIndicator` | Neo4j | Neo4j |
| `SolrHealthIndicator` | Solr | Solr |
| `Db2HealthIndicator` | DB2 | DB2 |
| `CassandraDriverHealthIndicator` | Driver | Cluster metadata |

### Health Configuration

```properties
# Show details always, when-authorized, or never
management.endpoint.health.show-details=always

# Show components
management.endpoint.health.show-components=always

# Roles for authorized access
management.endpoint.health.roles=ADMIN

# Status mapping (custom HTTP status codes)
management.endpoint.health.status.http-mapping.DOWN=503
management.endpoint.health.status.http-mapping.OUT_OF_SERVICE=503
management.endpoint.health.status.http-mapping.UP=200

# Add custom status ordering
management.endpoint.health.status.order=DOWN,OUT_OF_SERVICE,UP,UNKNOWN
```

### Custom Health Indicator

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {
    
    private final RestClient restClient;
    
    public ExternalApiHealthIndicator(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("https://api.example.com").build();
    }
    
    @Override
    public Health health() {
        try {
            long start = System.currentTimeMillis();
            restClient.get().uri("/health").retrieve().toBodilessEntity();
            long latency = System.currentTimeMillis() - start;
            return Health.up()
                .withDetail("latencyMs", latency)
                .withDetail("url", "https://api.example.com")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("reason", e.getMessage())
                .withException(e)
                .build();
        }
    }
}
```

### Reactive Health Indicator (WebFlux)

```java
@Component
public class ReactiveApiHealth implements ReactiveHealthIndicator {
    @Override
    public Mono<Health> health() {
        return webClient.get().uri("/health").retrieve().toBodilessEntity()
            .map(r -> Health.up().build())
            .onErrorResume(e -> Mono.just(Health.down(e).build()));
    }
}
```

### Health Groups

Group health indicators for different use cases (e.g., readiness vs liveness).

```properties
management.endpoint.health.group.liveness.include=ping,diskSpace
management.endpoint.health.group.readiness.include=db,redis,externalApi
management.endpoint.health.group.readiness.show-details=always
```

Access:

```bash
curl /actuator/health/liveness
curl /actuator/health/readiness
```

### Kubernetes Probes

```properties
management.endpoint.health.probes.enabled=true
management.health.livenessstate.enabled=true
management.health.readinessstate.enabled=true
```

Exposes:
- `/actuator/health/liveness` — is the app alive?
- `/actuator/health/readiness` — ready to serve traffic?

**Kubernetes deployment:**

```yaml
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
```

### Custom Status

```java
public final class CustomStatus implements Health.Status {
    public static final Health.Status DEGRADED = new CustomStatus("DEGRADED");
    private final String code;
    // ...
}
```

Or use `Status` builder:

```java
return Health.status("DEGRADED").withDetail("msg", "partial").build();
```

Register ordering:

```properties
management.endpoint.health.status.order=DEGRADED,DOWN,OUT_OF_SERVICE,UP,UNKNOWN
```

---

## 5. Info Endpoint

### Default Content

Empty by default. Populate via:

- `info.*` properties in `application.properties`
- `InfoContributor` beans
- Build info (Maven/Gradle plugin)
- Git info (git-commit-id plugin)
- Environment info

### Property-Based

```properties
info.app.name=My App
info.app.version=1.0.0
info.app.description=Demo application
```

```bash
curl /actuator/info
```

```json
{
  "app": {
    "name": "My App",
    "version": "1.0.0",
    "description": "Demo application"
  }
}
```

### Build Info

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>build-info</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Creates `META-INF/build-info.properties`. Accessed at `/actuator/info`.

### Git Info

```xml
<plugin>
    <groupId>io.github.git-commit-id</groupId>
    <artifactId>git-commit-id-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals><goal>revision</goal></goals>
        </execution>
    </executions>
    <configuration>
        <generateGitPropertiesFile>true</generateGitPropertiesFile>
    </configuration>
</plugin>
```

```properties
management.info.git.mode=full
```

Output includes commit ID, branch, author, timestamp.

### Custom InfoContributor

```java
@Component
public class CustomInfoContributor implements InfoContributor {
    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("custom", Map.of(
            "team", "Platform",
            "region", "us-east-1",
            "featureFlags", featureFlagService.getActive()
        ));
    }
}
```

### Environment Info (Boot 3.0+)

```properties
management.info.env.enabled=true
```

Enables `info.*` from properties (disabled by default for security in newer versions).

### Java Info

```properties
management.info.java.enabled=true
```

Includes JVM version, vendor, runtime.

### OS Info

```properties
management.info.os.enabled=true
```

Includes OS name, version, arch.

---

## 6. Metrics with Micrometer

**Micrometer** is a vendor-neutral metrics façade. Spring Boot Actuator integrates it. Add a registry (Prometheus, Datadog, etc.) and metrics flow there.

### Registries

| Registry | Dependency |
|----------|-----------|
| Prometheus | `micrometer-registry-prometheus` |
| Datadog | `micrometer-registry-datadog` |
| New Relic | `micrometer-registry-new-relic` |
| Graphite | `micrometer-registry-graphite` |
| Influx | `micrometer-registry-influx` |
| StatsD | `micrometer-registry-statsd` |
| CloudWatch | `micrometer-registry-cloudwatch2` |
| JMX | `micrometer-registry-jmx` |
| Simple (in-memory) | `micrometer-registry-simple` |

### Default Metrics

Spring Boot auto-registers:

- **JVM**: memory, GC, threads, classes
- **System**: CPU, load average, uptime
- **HTTP**: request count, latency (via `WebMvcTagsProvider`)
- **Tomcat**: sessions, threads
- **DataSource**: Hikari connections
- **Logback**: log events
- **Kafka**, **RabbitMQ**, **MongoDB** (if present)
- **Spring Integration**
- **RestTemplate** / **WebClient**

### Enable Metrics

Automatically enabled when `micrometer-registry-*` is on classpath.

```properties
management.metrics.enable.all=true
management.metrics.enable.jvm=true
management.metrics.enable.http.server.requests=true
```

### Metric Types

| Type | Use Case | Example |
|------|----------|---------|
| `Counter` | Monotonic increase | requests total |
| `Gauge` | Instant value | memory usage |
| `Timer` | Latency + count | HTTP request duration |
| `DistributionSummary` | Non-time distribution | payload size |

### Query Metrics

```bash
# List metric names
curl /actuator/metrics

# Query specific
curl /actuator/metrics/jvm.memory.used

# Filter by tag
curl /actuator/metrics/http.server.requests?tag=uri:/api/users&tag=method:GET
```

Response:

```json
{
  "name": "jvm.memory.used",
  "measurements": [{ "statistic": "VALUE", "value": 1.23e8 }],
  "availableTags": [
    { "tag": "area", "values": ["heap", "nonheap"] },
    { "tag": "id", "values": ["G1 Eden Space", "G1 Old Gen"] }
  ]
}
```

### MeterRegistry Injection

```java
@Service
public class OrderService {
    private final Counter ordersCounter;
    private final Timer orderTimer;
    
    public OrderService(MeterRegistry registry) {
        this.ordersCounter = Counter.builder("orders.created")
            .description("Total orders created")
            .tag("service", "order")
            .register(registry);
        this.orderTimer = Timer.builder("orders.duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
    
    public Order create(CreateOrderRequest req) {
        return orderTimer.record(() -> {
            Order o = doCreate(req);
            ordersCounter.increment();
            return o;
        });
    }
}
```

### Common Custom Metrics

```java
// Gauge — tracks a value over time
Gauge.builder("cache.size", cache, c -> c.size())
    .description("Current cache size")
    .register(registry);

// Counter — increment on events
Counter errors = registry.counter("errors", "type", "validation");

// Timer — measure latency
Timer.Sample sample = Timer.start(registry);
// ... operation
sample.stop(registry.timer("op.duration"));

// DistributionSummary — non-time values
registry.summary("payload.bytes").record(bytes);
```

### @Timed / @Counted (AOP-based)

```java
@Timed(value = "api.requests", percentiles = {0.5, 0.95, 0.99})
@Counted(value = "api.calls")
public User findUser(Long id) { ... }
```

Requires:

```java
@Bean
public TimedAspect timedAspect(MeterRegistry registry) {
    return new TimedAspect(registry);
}

@Bean
public CountedAspect countedAspect(MeterRegistry registry) {
    return new CountedAspect(registry);
}
```

Or with `@EnableAspectJAutoProxy`.

### Common Tags

```properties
management.metrics.tags.application=my-app
management.metrics.tags.region=us-east-1
management.metrics.tags.environment=prod
```

Every metric carries these tags — useful for grouping in Prometheus/Grafana.

### Meter Filters

```properties
# Disable specific metrics
management.metrics.enable.jvm.gc.pause=false

# Deny by name pattern
management.metrics.enable.process.cpu=false

# Prefix filter
management.metrics.enable.http=false
```

Programmatic filter:

```java
@Bean
public MeterRegistryCustomizer<MeterRegistry> metricsCommon() {
    return registry -> registry.config()
        .commonTags("app", "my-app")
        .meterFilter(MeterFilter.denyNameStartsWith("jvm.gc"));
}
```

### Percentiles & Histograms

```properties
management.metrics.distribution.percentiles-histogram.http.server.requests=true
management.metrics.distribution.percentiles.http.server.requests=0.5,0.95,0.99
management.metrics.distribution.sla.http.server.requests=10ms,50ms,100ms,500ms
management.metrics.distribution.minimum-expected-value.http.server.requests=10ms
management.metrics.distribution.maximum-expected-value.http.server.requests=5s
```

Enables histograms and percentiles in Prometheus output.

---

## 7. Prometheus & Grafana

### Add Prometheus Registry

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### Expose Endpoint

```properties
management.endpoints.web.exposure.include=health,info,prometheus
management.metrics.export.prometheus.enabled=true
```

Endpoint: `/actuator/prometheus`

### Sample Output

```
# HELP http_server_requests_seconds
# TYPE http_server_requests_seconds summary
http_server_requests_seconds_count{method="GET",status="200",uri="/api/users",} 42.0
http_server_requests_seconds_sum{method="GET",status="200",uri="/api/users",} 1.234

# HELP jvm_memory_used_bytes
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{area="heap",id="G1 Eden Space",} 3.2e8
```

### Prometheus Configuration

**prometheus.yml:**

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-boot'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['my-app:8080']
```

With authentication:

```yaml
scrape_configs:
  - job_name: 'spring-boot'
    basic_auth:
      username: prometheus
      password: secret
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['my-app:8080']
```

### Docker Compose Example

```yaml
version: '3.8'
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: health,prometheus

  prometheus:
    image: prom/prometheus
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports: ["3000:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
```

### Grafana Dashboard

Popular dashboards:

- **JVM (Micrometer)** — ID `4701`
- **Spring Boot 2.1 Statistics** — ID `11378`
- **Spring Boot Observability** — ID `12687`

Import by ID in Grafana.

### Common PromQL Queries

```promql
# Request rate
rate(http_server_requests_seconds_count[1m])

# P95 latency
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))

# Error rate
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count[5m]))

# Heap usage
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}
```

---

## 8. Custom Metrics

### Counter

```java
@Component
public class MetricsService {
    private final Counter ordersPlaced;
    
    public MetricsService(MeterRegistry registry) {
        this.ordersPlaced = Counter.builder("orders.placed")
            .description("Total orders placed")
            .tag("currency", "USD")
            .register(registry);
    }
    
    public void orderPlaced() { ordersPlaced.increment(); }
    public void bulkOrderPlaced(int n) { ordersPlaced.increment(n); }
}
```

### Timer

```java
@Component
public class LatencyTracker {
    private final Timer dbTimer;
    
    public LatencyTracker(MeterRegistry registry) {
        this.dbTimer = Timer.builder("db.query.duration")
            .description("DB query duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
    
    public <T> T track(Supplier<T> operation) {
        return dbTimer.record(operation);
    }
}
```

### Gauge

```java
@Component
public class QueueMetrics {
    private final Queue<Job> queue;
    
    public QueueMetrics(MeterRegistry registry, Queue<Job> queue) {
        this.queue = queue;
        Gauge.builder("jobs.queued", queue, Queue::size)
            .description("Jobs in queue")
            .register(registry);
    }
}
```

### DistributionSummary

```java
registry.summary("response.size", "endpoint", "/api/data")
    .record(bytes);
```

### @Timed with Aspects

```java
@Service
public class UserService {
    
    @Timed(value = "user.find", 
           percentiles = {0.5, 0.95, 0.99},
           extraTags = {"layer", "service"})
    public User find(Long id) { ... }
}
```

Requires `TimedAspect` bean (see §6).

---

## 9. Custom Endpoints

### @Endpoint (Generic — HTTP + JMX)

```java
@Component
@Endpoint(id = "features")
public class FeaturesEndpoint {
    
    private final Map<String, Boolean> features = new ConcurrentHashMap<>();
    
    @ReadOperation
    public Map<String, Boolean> features() {
        return features;
    }
    
    @ReadOperation
    public Feature feature(@Selector String name) {
        return new Feature(name, features.getOrDefault(name, false));
    }
    
    @WriteOperation
    public void configure(@Selector String name, boolean enabled) {
        features.put(name, enabled);
    }
    
    @DeleteOperation
    public void deleteFeature(@Selector String name) {
        features.remove(name);
    }
    
    public record Feature(String name, boolean enabled) { }
}
```

Endpoints:

- `GET /actuator/features` → all features
- `GET /actuator/features/{name}` → one feature
- `POST /actuator/features/{name}` with body `{"enabled":true}` → set
- `DELETE /actuator/features/{name}` → remove

Must be **exposed**:

```properties
management.endpoints.web.exposure.include=health,features
```

### @WebEndpoint (HTTP only)

```java
@Component
@WebEndpoint(id = "cache-stats")
public class CacheStatsEndpoint {
    
    @ReadOperation
    public CacheStats stats() {
        return new CacheStats(...);
    }
    
    public record CacheStats(long hits, long misses, double hitRatio) { }
}
```

### @JmxEndpoint (JMX only)

```java
@Component
@JmxEndpoint(id = "custom")
public class CustomJmxEndpoint {
    @ReadOperation
    public String status() { return "OK"; }
}
```

### Endpoint Attributes

```java
@Component
@Endpoint(id = "secure-feature")
public class SecureFeatureEndpoint {
    
    @ReadOperation
    public String status() { return "running"; }
}
```

Enable/disable:

```properties
management.endpoint.secure-feature.enabled=true
```

### Reactive Read Operation

```java
@ReadOperation
public Mono<String> status() { return Mono.just("running"); }
```

Supported on WebFlux.

### @Selector for Path Variables

```java
@ReadOperation
public User user(@Selector Long id) { ... }
```

Path `/actuator/users/{id}`.

### Endpoint Health Integration

Custom health can be queried via `/actuator/health`. Custom endpoints are independent — they don't contribute to aggregate health.

---

## 10. Securing Actuator

### Default Security

With `spring-boot-starter-security`:

- All Actuator endpoints require authentication (except `/actuator/health` unless configured)
- CSRF applies to POST/PUT/DELETE

### Custom SecurityFilterChain

```java
@Configuration
@EnableWebSecurity
public class ActuatorSecurityConfig {
    
    @Bean
    @Order(1)
    public SecurityFilterChain actuatorChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher(EndpointRequest.toAnyEndpoint()
                .excluding("health", "info", "prometheus"))
            .authorizeHttpRequests(auth -> auth
                .anyRequest().hasRole("ACTUATOR"))
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
    
    @Bean
    @Order(2)
    public SecurityFilterChain appChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/info", 
                                 "/actuator/prometheus").permitAll()
                .anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());
        return http.build();
    }
}
```

### EndpointRequest Matcher

```java
EndpointRequest.to("health", "info")           // specific endpoints
EndpointRequest.toAnyEndpoint()                // all endpoints
EndpointRequest.toLinks()                      // root links endpoint
```

### Sanitize Sensitive Data

```properties
# Keys always sanitized
management.endpoint.env.keys-to-sanitize=password,secret,token,key
```

Defaults include common patterns (`password`, `secret`, `key`, `token`, `*credentials*`).

### Restrict Exposure

```properties
# Only these endpoints over HTTP
management.endpoints.web.exposure.include=health,info,prometheus,metrics

# Exclude specific
management.endpoints.web.exposure.exclude=env,beans,heapdump,threaddump,shutdown

# Never enable shutdown in production
management.endpoint.shutdown.enabled=false
```

### Roles for Health Details

```properties
management.endpoint.health.show-details=when-authorized
management.endpoint.health.roles=ADMIN
```

### Separate Port + Firewall

```properties
management.server.port=9001
management.server.address=127.0.0.1
```

Only accessible locally (or via VPN).

### TLS for Actuator

Use the same HTTPS config as the main server, or configure separately:

```properties
management.server.ssl.enabled=true
management.server.ssl.key-store=classpath:keystore.p12
management.server.ssl.key-store-password=secret
```

---

## 11. Production Best Practices

### Exposure Checklist

- ✅ Expose only `health`, `info`, `prometheus`, `metrics` (or a subset)
- ✅ Never expose `env`, `beans`, `heapdump`, `threaddump`, `shutdown` publicly
- ✅ Disable `shutdown` endpoint entirely
- ✅ Use separate port + address binding for Actuator
- ✅ Secure with Spring Security
- ✅ Enable authentication for `/actuator/prometheus` if internet-reachable
- ✅ Sanitize sensitive env keys
- ✅ Restrict heap/thread dump access

### Health Checks

- ✅ Use `/actuator/health/liveness` for K8s livenessProbe
- ✅ Use `/actuator/health/readiness` for K8s readinessProbe
- ✅ Include DB, cache, and external API in readiness (not liveness)
- ✅ Liveness should NOT check dependencies (else pod restarts on dependency failure)
- ✅ Set `initialDelaySeconds` to allow startup

### Metrics

- ✅ Add `micrometer-registry-prometheus`
- ✅ Add common tags (app, env, region)
- ✅ Publish percentile histograms for HTTP
- ✅ Exclude high-cardinality tags (user IDs)
- ✅ Set retention in Prometheus
- ✅ Alert on error rate, latency, saturation

### Startup

```properties
management.endpoint.startup.enabled=true
```

Expose startup steps for slow-start analysis.

### Memory

```properties
# Enable heap dump endpoint only in dev/staging
management.endpoint.heapdump.enabled=false
```

### Logging

```properties
# Don't log Actuator requests
logging.level.org.springframework.boot.actuate=WARN
```

---

## 12. Interview Questions

### Q1. What is Spring Boot Actuator?

**Answer:** A sub-project providing production-ready features: monitoring endpoints (`/health`, `/metrics`, `/env`, `/beans`, `/loggers`, `/threaddump`), health indicators, metrics integration (Micrometer), and JMX support. Endpoints are exposed over HTTP or JMX and can be secured.

### Q2. Which endpoints are exposed by default?

**Answer:** Only `/actuator/health` and `/actuator/info` are exposed over HTTP by default. Others exist but must be exposed explicitly via `management.endpoints.web.exposure.include`.

### Q3. How do you expose more endpoints?

**Answer:** Set `management.endpoints.web.exposure.include=health,metrics,env,beans` (or `*` for all in dev). Exclude with `.exclude`.

### Q4. What is the difference between `/actuator/health` and `/actuator/health/liveness`?

**Answer:** `/health` is the aggregate status of all `HealthIndicator` beans. `/health/liveness` (Kubernetes probe) checks if the app is alive (JVM not stuck) — usually excludes dependencies. `/health/readiness` checks if the app is ready to serve traffic — includes dependencies.

### Q5. What is Micrometer?

**Answer:** A vendor-neutral metrics façade. Applications use Micrometer's API (`MeterRegistry`, `Counter`, `Timer`, `Gauge`), and Micrometer publishes to any registry (Prometheus, Datadog, New Relic, etc.) via a `MeterRegistry` implementation. Spring Boot auto-configures Micrometer and common registries.

### Q6. How do you enable Prometheus?

**Answer:** Add `micrometer-registry-prometheus` dependency and expose the `prometheus` endpoint via `management.endpoints.web.exposure.include=health,info,prometheus`. Scrape `/actuator/prometheus`.

### Q7. What is a HealthIndicator?

**Answer:** A bean implementing `HealthIndicator` (or `ReactiveHealthIndicator`) that reports `Health.up()` / `Health.down()` with optional details. Spring Boot aggregates all indicators into `/actuator/health`. Built-in indicators cover DB, Redis, disk, etc.

### Q8. How do you create a custom endpoint?

**Answer:** Annotate a bean with `@Endpoint(id = "...")` and methods with `@ReadOperation`, `@WriteOperation`, `@DeleteOperation`. Use `@Selector` for path parameters. Register by exposing the ID via `management.endpoints.web.exposure.include`.

### Q9. How do you change log levels at runtime?

**Answer:** POST to `/actuator/loggers/{name}` with `{"configuredLevel":"DEBUG"}`. The `loggers` endpoint must be exposed. Reset by sending `null`.

### Q10. How do you secure Actuator endpoints?

**Answer:** Use Spring Security with `EndpointRequest.toAnyEndpoint()` matcher. Configure role-based access. Use a separate port (`management.server.port=9001`) and bind to `127.0.0.1`. Exclude non-sensitive endpoints (`health`, `info`) from auth. Sanitize env keys via `management.endpoint.env.keys-to-sanitize`.

### Q11. What does `/actuator/env` do?

**Answer:** Lists all property sources and their values, showing where each property comes from. Sensitive keys (`password`, `secret`, `key`) are sanitized (`******`) by default.

### Q12. Difference between `health.show-details=always` and `when-authorized`?

**Answer:** `always` shows component details to anyone. `when-authorized` shows details only to users with roles specified in `management.endpoint.health.roles`.

### Q13. How do you expose heap/thread dumps?

**Answer:** `management.endpoints.web.exposure.include=heapdump,threaddump`. `GET /actuator/heapdump` downloads an HPROF. `GET /actuator/threaddump` returns JSON thread dump. **Never expose publicly**.

### Q14. What is a MeterRegistry?

**Answer:** Micrometer's central interface for creating and querying meters (counter, gauge, timer, distribution summary). Spring Boot registers a `MeterRegistry` bean per enabled registry (Prometheus, JMX, etc.). Inject it to create custom metrics.

### Q15. What are common tags?

**Answer:** Key-value pairs added to every metric. Set via `management.metrics.tags.*` or programmatically with `MeterRegistryCustomizer`. Useful for grouping metrics by app name, environment, region.

### Q16. What is @Timed?

**Answer:** A Micrometer annotation that times a method and records to a `Timer`. Requires a `TimedAspect` bean registered. Records count, total time, percentiles.

### Q17. How do you disable specific metrics?

**Answer:** `management.metrics.enable.jvm.gc.pause=false` or programmatically with a `MeterFilter` (e.g., `MeterFilter.denyNameStartsWith("jvm.gc")`).

### Q18. What are Kubernetes probes and how does Actuator help?

**Answer:** Liveness (is the app alive?) and readiness (can it serve traffic?). Spring Boot exposes `/actuator/health/liveness` and `/actuator/health/readiness` when `management.endpoint.health.probes.enabled=true`. K8s `livenessProbe` and `readinessProbe` point at these paths.

### Q19. How do you customize info endpoint?

**Answer:** Add `info.*` properties, `InfoContributor` beans, or plugins (`spring-boot-maven-plugin:build-info`, `git-commit-id-maven-plugin`). Set `management.info.env.enabled=true` to include properties, `management.info.git.mode=full` for full git info.

### Q20. What is the `/actuator/conditions` endpoint for?

**Answer:** Shows the auto-configuration report: which configs applied (positive matches), which didn't (negative matches), with reasons. Equivalent to `debug=true` output. Invaluable for debugging why a bean wasn't created.

---

## 13. Cheat Sheet

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### Key Properties

```properties
management.endpoints.web.exposure.include=health,info,prometheus,metrics
management.endpoints.web.exposure.exclude=env,beans,shutdown
management.endpoints.web.base-path=/actuator
management.server.port=9001
management.endpoint.health.show-details=when-authorized
management.endpoint.health.roles=ADMIN
management.endpoint.shutdown.enabled=false
management.metrics.tags.application=my-app
management.metrics.distribution.percentiles-histogram.http.server.requests=true
```

### Common Endpoints

```
/actuator
/actuator/health
/actuator/health/liveness
/actuator/health/readiness
/actuator/info
/actuator/metrics
/actuator/metrics/{name}
/actuator/prometheus
/actuator/env
/actuator/beans
/actuator/mappings
/actuator/configprops
/actuator/loggers
/actuator/loggers/{name}   (POST)
/actuator/threaddump
/actuator/heapdump
/actuator/conditions
/actuator/scheduledtasks
/actuator/caches
```

### Custom Health

```java
@Component
public class ApiHealth implements HealthIndicator {
    public Health health() {
        return apiUp() ? Health.up().withDetail("latency", 12).build()
                       : Health.down().withDetail("reason", "timeout").build();
    }
}
```

### Custom Counter

```java
Counter c = Counter.builder("orders.placed")
    .tag("service", "order").register(registry);
c.increment();
```

### Custom Endpoint

```java
@Component
@Endpoint(id = "features")
public class FeaturesEndpoint {
    @ReadOperation public Map<String, Boolean> features() { ... }
    @WriteOperation public void set(@Selector String name, boolean enabled) { ... }
}
```

### Secure Actuator

```java
@Bean
@Order(1)
public SecurityFilterChain actuator(HttpSecurity http) throws Exception {
    http.securityMatcher(EndpointRequest.toAnyEndpoint()
            .excluding("health", "info", "prometheus"))
        .authorizeHttpRequests(a -> a.anyRequest().hasRole("ACTUATOR"))
        .httpBasic(Customizer.withDefaults());
    return http.build();
}
```

### PromQL Examples

```promql
rate(http_server_requests_seconds_count[1m])
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
jvm_memory_used_bytes{area="heap"}
```

### Cross-References

- **Previous:** `08_Spring_Boot_Configuration.md`
- **Next:** `10_Spring_Data_JPA.md`
- **Related:** `19_Spring_Microservices_Cloud.md` (distributed tracing)
- **Interview:** `24_Spring_Interview_Questions.md`

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [08_Spring_Boot_Configuration.md](./08_Spring_Boot_Configuration.md)
- **Next →:** [10_Spring_Data_JPA.md](./10_Spring_Data_JPA.md)
- **Related:** [06_Spring_Boot_Fundamentals.md](./06_Spring_Boot_Fundamentals.md), [16_Spring_Testing.md](./16_Spring_Testing.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
