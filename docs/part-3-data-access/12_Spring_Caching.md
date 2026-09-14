# Spring Caching

> **File:** `12_Spring_Caching.md`
> **Part:** 3 — Data Access
> **Prerequisites:** `01_Spring_Framework_Core.md`, `02_Spring_AOP.md`
> **Estimated Study Time:** 5–7 hours

---

## Table of Contents

1. [Caching Concepts](#1-caching-concepts)
2. [Spring Cache Abstraction](#2-spring-cache-abstraction)
3. [Enabling Caching](#3-enabling-caching)
4. [@Cacheable](#4-cacheable)
5. [@CachePut](#5-cacheput)
6. [@CacheEvict](#6-cacheevict)
7. [@Caching (Combining)](#7-caching-combining)
8. [Cache Configuration](#8-cache-configuration)
9. [Key Generation](#9-key-generation)
10. [Conditional Caching](#10-conditional-caching)
11. [Cache Providers](#11-cache-providers)
12. [Redis Cache](#12-redis-cache)
13. [Caffeine Cache](#13-caffeine-cache)
14. [Cache Metrics & Monitoring](#14-cache-metrics--monitoring)
15. [Common Pitfalls](#15-common-pitfalls)
16. [Interview Questions](#16-interview-questions)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. Caching Concepts

**Caching** stores the result of an expensive operation so subsequent calls return faster.

### Core Terms

| Term | Meaning |
|------|---------|
| **Cache** | Storage for cached values |
| **Cache hit** | Value found in cache |
| **Cache miss** | Value not found; compute and store |
| **Eviction** | Removing entries |
| **TTL** | Time-to-live for entries |
| **Invalidation** | Marking entries stale on data change |

### Cache Hit Ratio

```
hitRatio = hits / (hits + misses)
```

Higher = better. Aim for >80% for effective caching.

### What to Cache

- ✅ Expensive computations (aggregations, ML inference)
- ✅ Frequent reads of rarely-changing data (config, reference data)
- ✅ External API responses
- ✅ Session data
- ❌ Rapidly-changing data
- ❌ Large objects with low hit ratio
- ❌ User-specific data with strict consistency needs

### Cache Strategies

| Strategy | Behavior |
|----------|----------|
| **Cache-Aside** | App reads cache; on miss, reads DB and populates |
| **Read-Through** | Cache reads DB on miss |
| **Write-Through** | Write to cache and DB together |
| **Write-Behind** | Write to cache; async to DB |

Spring's `@Cacheable` implements **cache-aside**.

### Cache Invalidation

> "There are only two hard things in computer science: cache invalidation and naming things." — Phil Karlton

Strategies:
- **TTL** — expire after time
- **Explicit evict** — on write, remove key
- **Versioned keys** — include a version in the key
- **Write-through** — update on write

Spring provides `@CacheEvict` and `@CachePut` for explicit invalidation.

---

## 2. Spring Cache Abstraction

Spring provides a **cache abstraction** — a consistent API over multiple cache providers.

### The SPI

```
┌──────────────────────────────────┐
│     Application Code             │
│     @Cacheable, @CachePut, ...   │
├──────────────────────────────────┤
│     Spring Cache Abstraction     │
│     CacheManager, Cache          │
├──────────────────────────────────┤
│     Providers                    │
│  ┌────────┬─────────┬──────────┐ │
│  │Concur  │ Redis   │ Caffeine │ │
│  │rentMap │         │          │ │
│  └────────┴─────────┴──────────┘ │
│         ... (Ehcache, Hazelcast) │
└──────────────────────────────────┘
```

### Key Interfaces

| Interface | Purpose |
|-----------|---------|
| `CacheManager` | Creates/manages caches |
| `Cache` | A single cache (get, put, evict) |
| `CacheResolver` | Resolves which cache to use |
| `KeyGenerator` | Generates cache keys |
| `CacheErrorHandler` | Handles cache errors |

### Enabling the Abstraction

Add `@EnableCaching` and a `CacheManager` (or let Spring Boot auto-configure).

```java
@Configuration
@EnableCaching
public class CacheConfig { }
```

---

## 3. Enabling Caching

### Add Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

Pulls in `spring-context-support` (for Caffeine, Ehcache integration).

### Enable

```java
@SpringBootApplication
@EnableCaching
public class App { ... }
```

Or on a config class:

```java
@Configuration
@EnableCaching
public class CacheConfig { }
```

### Auto-Configuration

Spring Boot auto-configures a `CacheManager` when:

- `@EnableCaching` present
- One of these on classpath: Caffeine, Ehcache, Redis, Hazelcast, Couchbase, Infinispan, or none

If none → `ConcurrentMapCacheManager`.

### Specifying Provider

```properties
spring.cache.type=redis         # or caffeine, ehcache, none
```

Values: `caffeine`, `couchbase`, `ehcache`, `hazelcast`, `infinispan`, `jcache`, `redis`, `simple`, `none`.

---

## 4. @Cacheable

Caches the **return value** of a method.

```java
@Cacheable("users")
public User findById(Long id) {
    // expensive lookup
    return userRepository.findById(id).orElseThrow();
}
```

### Behavior

1. Method called with args → cache key computed
2. Key present → return cached value (**hit**)
3. Key absent → invoke method, store result (**miss**)

### Value/CacheName

```java
@Cacheable("users")                       // shorthand
@Cacheable(cacheNames = "users")
@Cacheable(value = "users")
@Cacheable({"users", "allUsers"})         // multiple caches — value stored in all
```

Multiple caches: the value is stored in all; lookup checks each in order.

### Key

```java
@Cacheable(value = "users", key = "#id")
public User findById(Long id) { ... }

@Cacheable(value = "users", key = "#user.email")
public User save(User user) { ... }

@Cacheable(value = "users", key = "#id + ':' + #tenant")
public User findById(Long id, String tenant) { ... }
```

Default key generation uses all parameters.

### Condition (evaluated before invocation)

```java
@Cacheable(value = "users", condition = "#id > 0")
public User findById(Long id) { ... }
```

If condition is false → **not cached, not looked up**.

### Unless (evaluated after invocation, on result)

```java
@Cacheable(value = "users", unless = "#result == null")
public User findById(Long id) { ... }

@Cacheable(value = "users", unless = "#result.active == false")
public User findById(Long id) { ... }
```

If unless is true → **not cached** (but was computed).

### Condition vs Unless

| Attribute | Evaluated | Purpose |
|-----------|-----------|---------|
| `condition` | Before | Skip cache entirely |
| `unless` | After | Don't cache certain results |

### Sync

```java
@Cacheable(value = "users", sync = true)
public User findById(Long id) { ... }
```

Blocks concurrent calls for the same key — avoids duplicate computation (**cache stampede**). Only supported by some providers (ConcurrentMap, Redis via `@Cacheable` since 4.3).

### @Cacheable on Class

```java
@Service
@Cacheable("users")
public class UserService {
    // all public methods cached
}
```

Rarely used — usually method-level.

### Return Types

- Works with any return type
- `Optional<T>` is cached as-is (not unwrapped) — check semantics

---

## 5. @CachePut

**Always invokes the method** and **updates the cache** with the result.

```java
@CachePut(value = "users", key = "#user.id")
public User update(User user) {
    return userRepository.save(user);
}
```

### Behavior

1. Method always invoked
2. Result stored in cache under the key

**Use case:** Update methods that must update both DB and cache.

### @Cacheable vs @CachePut

| Aspect | @Cacheable | @CachePut |
|--------|-----------|-----------|
| Method invoked | On miss | Always |
| Cache read | ✅ | ❌ |
| Cache write | ✅ | ✅ |
| Use case | Read methods | Write methods |

### Example

```java
@Service
public class UserService {
    
    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
    
    @CachePut(value = "users", key = "#result.id")
    public User update(Long id, UserRequest req) {
        User u = userRepository.findById(id).orElseThrow();
        u.setName(req.name());
        return userRepository.save(u);
    }
}
```

`@CachePut(value="users", key="#result.id")` — uses the returned entity's ID as the key, avoiding stale args.

---

## 6. @CacheEvict

**Removes** entries from cache.

```java
@CacheEvict("users")
public void clearAll() { }

@CacheEvict(value = "users", key = "#id")
public void delete(Long id) { }

@CacheEvict(value = "users", allEntries = true)
public void refreshAll() { }
```

### Attributes

| Attribute | Default | Effect |
|-----------|---------|--------|
| `value`/`cacheNames` | Required | Cache(s) to evict |
| `key` | All params | Key to evict |
| `condition` | — | Only evict if true |
| `allEntries` | `false` | Evict all entries |
| `beforeInvocation` | `false` | Evict before method runs |

### beforeInvocation

```java
@CacheEvict(value = "users", key = "#id", beforeInvocation = true)
public void delete(Long id) { ... }
```

- `false` (default) — evict **after** method succeeds (method can throw → no evict)
- `true` — evict **before** method (even if method throws)

Use `beforeInvocation = true` for safety when data may be removed regardless of the method outcome.

### Example

```java
@Service
public class UserService {
    
    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) { ... }
    
    @CacheEvict(value = "users", key = "#id")
    public void delete(Long id) {
        userRepository.deleteById(id);
    }
    
    @CacheEvict(value = "users", allEntries = true)
    public void importAll(List<User> users) {
        userRepository.saveAll(users);
    }
}
```

---

## 7. @Caching (Combining)

Combine multiple cache operations on one method.

```java
@Caching(
    put = {
        @CachePut(value = "users", key = "#user.id"),
        @CachePut(value = "usersByEmail", key = "#user.email")
    },
    evict = {
        @CacheEvict(value = "userList", allEntries = true)
    }
)
public User update(User user) {
    return userRepository.save(user);
}
```

### Attributes

- `cacheable = {...}`
- `put = {...}`
- `evict = {...}`

### Use Case

Method touches multiple caches simultaneously:

- Update user → update `users` cache by ID
- Also update `usersByEmail` cache (lookup by email)
- Evict `userList` (list is stale)

### Avoid Over-Combining

If a class has multiple `@Caching` methods, consider whether the caching strategy is too complex. Often it's better to have a single primary cache and let other data be evicted.

---

## 8. Cache Configuration

### Default CacheManager

Without a provider, Spring Boot uses `ConcurrentMapCacheManager` — in-memory, no eviction, suitable for dev.

### Custom CacheManager

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cm = new CaffeineCacheManager();
        cm.setCaffeine(Caffeine.newBuilder()
            .expireAfterWrite(Duration.ofMinutes(10))
            .maximumSize(1000));
        return cm;
    }
}
```

### Per-Cache Configuration

```java
@Bean
public CacheManager cacheManager() {
    SimpleCacheManager cm = new SimpleCacheManager();
    cm.setCaches(List.of(
        new ConcurrentMapCache("users"),
        new ConcurrentMapCache("products")
    ));
    return cm;
}
```

### Disable Caching in Tests

```properties
spring.cache.type=none
```

Or:

```java
@TestPropertySource(properties = "spring.cache.type=none")
class MyTest { }
```

### Multiple Caches

With `@Cacheable({"users", "allUsers"})` — first is primary; the value is stored in all caches.

### Cache Names Constants

```java
public final class CacheNames {
    public static final String USERS = "users";
    public static final String PRODUCTS = "products";
    private CacheNames() { }
}

@Cacheable(CacheNames.USERS)
public User findById(Long id) { ... }
```

### Boot Properties

```properties
spring.cache.type=caffeine
spring.cache.cache-names=users,products,orders
spring.cache.caffeine.spec=maximumSize=500,expireAfterWrite=10m
spring.cache.redis.time-to-live=600000
spring.cache.redis.cache-null-values=false
spring.cache.redis.key-prefix=myapp:cache:
```

### Error Handling

By default, cache errors propagate. Customize:

```java
@Configuration
@EnableCaching
public class CacheConfig extends CachingConfigurerSupport {
    @Override
    public CacheErrorHandler errorHandler() {
        return new SimpleCacheErrorHandler() {
            @Override
            public void handleCacheGetError(RuntimeException ex, Cache cache, Object key) {
                log.warn("Cache GET failed: {}", cache.getName(), ex);
                // Swallow — fall back to method invocation
            }
            // similar for put/evict/clear
        };
    }
}
```

**Production tip:** Cache errors (Redis down) shouldn't break the app. Swallow + log.

---

## 9. Key Generation

### Default KeyGenerator

Uses all method args:
- No args → `SimpleKey.EMPTY`
- One arg → the arg itself
- Multiple args → `SimpleKey(args...)`

`SimpleKey` has value semantics (`equals`/`hashCode`).

### SpEL Keys

```java
@Cacheable(value = "users", key = "#id")
public User findById(Long id) { ... }

@Cacheable(value = "users", key = "#user.id")
public User save(User user) { ... }

@Cacheable(value = "userPages", key = "#page + '-' + #size")
public Page<User> findPage(int page, int size) { ... }

@Cacheable(value = "users", key = "#root.methodName + ':' + #id")
public User findById(Long id) { ... }
```

### Useful SpEL Variables

| Variable | Value |
|----------|-------|
| `#root.method` | Method object |
| `#root.methodName` | Method name |
| `#root.target` | Target object |
| `#root.targetClass` | Target class |
| `#root.args` | Args array |
| `#root.caches` | Caches array |
| `#result` | Return value (for `unless`) |
| `#a0`, `#p0` | First arg |
| `#paramName` | Named param (if compiled with `-parameters`) |

### Custom KeyGenerator

```java
@Component("myKeyGen")
public class MyKeyGenerator implements KeyGenerator {
    @Override
    public Object generate(Object target, Method method, Object... params) {
        return method.getName() + ":" + Arrays.toString(params);
    }
}

@Cacheable(value = "users", keyGenerator = "myKeyGen")
public User findById(Long id) { ... }
```

### Configure Default KeyGenerator

```java
@Configuration
@EnableCaching
public class CacheConfig implements CachingConfigurer {
    @Override
    public KeyGenerator keyGenerator() {
        return new MyKeyGenerator();
    }
}
```

### Null-Safe Keys

```java
@Cacheable(value = "users", key = "#id ?: 'unknown'")
```

### @Cacheable with Composite Key Object

```java
public record UserCacheKey(Long id, String tenant) { }

@Cacheable(value = "users", key = "new com.example.UserCacheKey(#id, #tenant)")
```

Simpler: use string concatenation.

```java
@Cacheable(value = "users", key = "#id + '-' + #tenant")
```

---

## 10. Conditional Caching

### condition — before invocation

```java
@Cacheable(value = "users", condition = "#id > 0")
public User findById(Long id) { ... }
```

False → no cache interaction, method runs.

### unless — after invocation

```java
@Cacheable(value = "users", unless = "#result == null")
public User findById(Long id) { ... }

@Cacheable(value = "users", unless = "#result.age < 18")
public User findById(Long id) { ... }
```

True → do not cache the result.

### Combined

```java
@Cacheable(
    value = "users", 
    condition = "#id > 0",
    unless = "#result == null || #result.disabled"
)
public User findById(Long id) { ... }
```

### @CachePut with condition/unless

```java
@CachePut(value = "users", key = "#user.id", condition = "#user.active")
public User save(User user) { ... }
```

### @CacheEvict with condition

```java
@CacheEvict(value = "users", key = "#id", condition = "#id != null")
public void delete(Long id) { ... }
```

### Runtime Args & Result

- `condition` sees only args (not result)
- `unless` sees args **and** `#result`
- `#root.target`, `#root.method` available in both

---

## 11. Cache Providers

### Comparison

| Provider | Type | Distributed | TTL | Best For |
|----------|------|-------------|-----|----------|
| **ConcurrentMap** | In-memory | ❌ | ❌ | Dev, tests |
| **Caffeine** | In-memory | ❌ | ✅ | Single-node prod |
| **Ehcache** | In-memory + disk | Cluster option | ✅ | Heavy local caching |
| **Redis** | External | ✅ | ✅ | Distributed, sessions |
| **Hazelcast** | Distributed | ✅ | ✅ | In-memory data grid |
| **Infinispan** | Distributed | ✅ | ✅ | Enterprise, JCache |
| **Couchbase** | External | ✅ | ✅ | Document + cache |
| **JCache (JSR-107)** | Spec | Depends | ✅ | Standard API |

### Auto-Selection Order (Boot)

Spring Boot auto-configures in this order (first available):

1. Generic (custom `CacheManager` bean)
2. JCache
3. Hazelcast
4. Infinispan
5. Couchbase
6. Redis
7. Caffeine
8. Simple (ConcurrentMap)

Override with `spring.cache.type`.

### ConcurrentMapCache (Default)

- In-memory
- No eviction
- Per JVM
- Fine for dev, not prod

### Ehcache 3

```xml
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

`ehcache.xml`:

```xml
<config xmlns="http://www.ehcache.org/v3">
    <cache alias="users">
        <key-type>java.lang.Long</key-type>
        <value-type>com.example.User</value-type>
        <expiry><ttl unit="minutes">10</ttl></expiry>
        <resources>
            <heap unit="entries">1000</heap>
            <offheap unit="MB">50</offheap>
        </resources>
    </cache>
</config>
```

Boot auto-detects `ehcache.xml`.

### JCache (JSR-107)

```xml
<dependency>
    <groupId>javax.cache</groupId>
    <artifactId>cache-api</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <classifier>jakarta</classifier>
</dependency>
```

```properties
spring.cache.type=jcache
spring.cache.jcache.config=classpath:ehcache.xml
```

Uses standard `@CacheResult`, `@CachePut`, `@CacheRemove` annotations — but Spring's `@Cacheable` also works via the `JCacheCacheManager`.

---

## 12. Redis Cache

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

### Configuration

```properties
spring.cache.type=redis
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.data.redis.password=secret
spring.cache.redis.time-to-live=600000
spring.cache.redis.cache-null-values=false
spring.cache.redis.key-prefix=myapp:cache:
spring.cache.redis.use-key-prefix=true
```

### Serialization

Default uses JDK serialization — verbose. Override:

```java
@Bean
public RedisCacheConfiguration cacheConfiguration() {
    return RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(10))
        .disableCachingNullValues()
        .serializeKeysWith(RedisSerializationContext.SerializationPair
            .fromSerializer(new StringRedisSerializer()))
        .serializeValuesWith(RedisSerializationContext.SerializationPair
            .fromSerializer(new GenericJackson2JsonRedisSerializer()));
}

@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory cf) {
    return RedisCacheManager.builder(cf)
        .cacheDefaults(cacheConfiguration())
        .withCacheConfiguration("users", 
            cacheConfiguration().entryTtl(Duration.ofMinutes(5)))
        .withCacheConfiguration("products", 
            cacheConfiguration().entryTtl(Duration.ofHours(1)))
        .build();
}
```

### Key Prefix

`myapp:cache:users::1`

Prefix isolates app caches from other Redis keys.

### Distributed Caching

Redis works across multiple app instances — critical for microservices and K8s deployments.

### Session Storage (bonus)

Redis also backs Spring Session:

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

### Pitfalls

- **Network latency** — every cache call is a round-trip. Cache small data.
- **Serialization cost** — JSON or binary. Choose the serializer wisely.
- **Cache stampede** — many concurrent misses on same key. Use `sync=true`.
- **Eviction policy** — Redis default is `noeviction` (fails writes when full). Set `allkeys-lru` or `volatile-lru`.

### Redis Eviction Policies

| Policy | Behavior |
|--------|----------|
| `noeviction` | Fail writes when memory full |
| `allkeys-lru` | Evict any key (LRU) |
| `allkeys-lfu` | Evict any key (LFU) |
| `volatile-lru` | Evict keys with TTL (LRU) |
| `volatile-ttl` | Evict keys with shortest TTL |

Set in `redis.conf`:

```
maxmemory 2gb
maxmemory-policy allkeys-lru
```

---

## 13. Caffeine Cache

**Caffeine** is a high-performance in-memory cache, the modern replacement for Guava Cache.

### Dependency

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

### Auto-Config

```properties
spring.cache.type=caffeine
spring.cache.cache-names=users,products
spring.cache.caffeine.spec=maximumSize=1000,expireAfterWrite=10m
```

Spring Boot auto-configures `CaffeineCacheManager`.

### Custom Configuration

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cm = new CaffeineCacheManager();
        cm.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .expireAfterAccess(Duration.ofMinutes(5))
            .recordStats());
        return cm;
    }
}
```

### Per-Cache Builder

```java
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager cm = new CaffeineCacheManager();
    cm.setCacheSpecification("maximumSize=1000,expireAfterWrite=10m");
    cm.registerCustomCache("users", 
        Caffeine.newBuilder()
            .maximumSize(500)
            .expireAfterWrite(Duration.ofMinutes(30))
            .build());
    return cm;
}
```

### Features

- Size-based eviction (LRU-ish, Window TinyLFU)
- Time-based expiration (write, access, custom)
- Refresh (async reload)
- Stats (hit rate, load time)
- Async loading

### When to Use

- Single-node apps needing fast local cache
- Replace default ConcurrentMap
- When Redis is overkill

### When Not to Use

- Multiple app instances needing shared cache → use Redis/Hazelcast

### Stats

```properties
spring.cache.caffeine.spec=recordStats
```

```java
CaffeineCache cache = (CaffeineCache) cacheManager.getCache("users");
CacheStats stats = cache.getNativeCache().stats();
log.info("Hit rate: {}", stats.hitRate());
```

Exposed via Actuator `/actuator/metrics/cache.*` if Micrometer enabled.

---

## 14. Cache Metrics & Monitoring

### Auto-Registered Metrics

With Actuator + Micrometer, cache metrics are auto-published:

```
cache.gets{name="users", result="hit"}
cache.gets{name="users", result="miss"}
cache.puts{name="users"}
cache.evictions{name="users"}
cache.size{name="users"}
```

### Enable Cache Metrics

```java
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager cm = new CaffeineCacheManager();
    cm.setCaffeine(Caffeine.newBuilder().recordStats());
    return cm;
}
```

For Redis, use `RedisCacheManager` and Micrometer auto-instruments.

### Query Metrics

```bash
curl /actuator/metrics/cache.gets
curl /actuator/metrics/cache.gets?tag=name:users&tag=result:hit
```

### Custom Metrics

```java
@Service
public class CacheMetrics {
    private final MeterRegistry registry;
    
    public CacheMetrics(MeterRegistry registry, CacheManager cm) {
        this.registry = registry;
        Gauge.builder("cache.size", cm, m -> {
            Cache c = m.getCache("users");
            return c != null ? ((CaffeineCache) c).getNativeCache().estimatedSize() : 0;
        }).register(registry);
    }
}
```

### Alerting

Alert on:
- **Low hit ratio** (<50%) → cache is ineffective
- **High eviction rate** → cache too small
- **Redis connection errors** → cache layer down

### Logging

Enable cache debug logs:

```properties
logging.level.org.springframework.cache=DEBUG
```

Logs cache hits/misses with method and key.

---

## 15. Common Pitfalls

### 15.1 Self-Invocation

Same as `@Transactional`:

```java
@Service
public class UserService {
    
    public User get(Long id) {
        return this.load(id);   // ❌ bypasses @Cacheable
    }
    
    @Cacheable("users")
    public User load(Long id) { ... }
}
```

Fix: split into another bean, or inject self, or use `AopContext.currentProxy()`.

### 15.2 Caching Nulls

By default, null results are cached. For optional values, may be desirable; for missing data, often not.

```java
@Cacheable(value = "users", unless = "#result == null")
public User findById(Long id) { ... }
```

Or set `spring.cache.redis.cache-null-values=false`.

### 15.3 Key Collisions

Two different methods returning different types but same key value → same cache entry. Use distinct caches per method or prefix keys.

### 15.4 Mutable Objects

Cached objects are shared. Mutating one affects all callers.

```java
@Cacheable("users")
public User find(Long id) { ... }

User u = find(1L);
u.setName("Hacked");    // ⚠️ cached object mutated!
```

Fix: cache immutable DTOs, or deep-copy on cache/retrieve.

### 15.5 Cache Stampede

Many concurrent misses on the same key → all compute simultaneously.

Fix: `@Cacheable(sync = true)`.

### 15.6 Forgetting Eviction

Cache grows stale after writes.

Fix: `@CacheEvict` or `@CachePut` on write methods.

### 15.7 Using Cache for Consistency-Critical Data

Cached data is eventually consistent. Don't cache data where stale reads cause bugs.

### 15.8 Cache Inherited Methods

`@Cacheable` on a class caches inherited public methods, but annotations on overridden methods may not inherit.

### 15.9 Cache + Transaction Ordering

If `@Cacheable` method is also `@Transactional`, careful:

```java
@Cacheable("users")
@Transactional(readOnly = true)
public User find(Long id) { ... }
```

Order: Cache advice runs **outside** transaction advice by default. On miss, opens transaction, returns result, caches after commit. Usually fine.

For write methods with `@CachePut` + `@Transactional`, the cache is updated **before** commit — if transaction rolls back, cache has stale value.

Fix: use `@TransactionalEventListener(AFTER_COMMIT)` to populate cache.

### 15.10 Big Objects

Caching large objects (images, big lists) can exhaust memory.

Fix: bound size (`maximumSize`), use off-heap (Ehcache), or cache references/IDs.

### Pitfalls Checklist

- [ ] No self-invocation
- [ ] `unless` set for nulls if needed
- [ ] Keys unique per cache
- [ ] Cached objects immutable
- [ ] `sync=true` for hot keys
- [ ] Eviction on writes
- [ ] TTL set appropriately
- [ ] Cache manager configured for prod
- [ ] Metrics monitored

---

## 16. Interview Questions

### Q1. What is caching and why use Spring's cache abstraction?

**Answer:** Caching stores expensive operation results for faster retrieval. Spring's abstraction provides consistent annotations (`@Cacheable`, `@CachePut`, `@CacheEvict`) over multiple providers (Caffeine, Redis, Ehcache), so you can swap providers without changing business code.

### Q2. Difference between @Cacheable, @CachePut, and @CacheEvict?

**Answer:**
- `@Cacheable`: check cache; on miss, invoke method and cache result
- `@CachePut`: always invoke method and update cache
- `@CacheEvict`: remove entries from cache

### Q3. What is the difference between `condition` and `unless`?

**Answer:** `condition` is evaluated **before** the method — if false, cache is skipped entirely. `unless` is evaluated **after** the method using `#result` — if true, the result is not cached.

### Q4. How does Spring generate cache keys?

**Answer:** By default, all method arguments — single arg is used directly; multiple args → `SimpleKey`. Override with `key = "#id"` (SpEL) or a custom `KeyGenerator`.

### Q5. What is `allEntries = true` in @CacheEvict?

**Answer:** Clears **all entries** in the specified cache(s), ignoring the key. Use for bulk invalidation ("refresh entire cache").

### Q6. What is `beforeInvocation = true` in @CacheEvict?

**Answer:** Evicts **before** the method runs, even if it throws. Default is false — eviction happens only on successful method return.

### Q7. How do you cache multiple keys for one method?

**Answer:** Use `@Caching` with multiple `@CachePut` / `@CacheEvict`:

```java
@Caching(put = {
    @CachePut(value = "users", key = "#user.id"),
    @CachePut(value = "usersByEmail", key = "#user.email")
})
```

### Q8. What is cache stampede and how do you prevent it?

**Answer:** Many concurrent requests miss the same cache key and all invoke the method. Prevent with `@Cacheable(sync = true)` (single execution per key) or by pre-warming caches.

### Q9. How do you disable caching in tests?

**Answer:** `spring.cache.type=none` property, or `@TestPropertySource(properties = "spring.cache.type=none")`, or don't add `@EnableCaching`.

### Q10. How do you customize cache error handling?

**Answer:** Implement `CachingConfigurer.errorHandler()` returning a `CacheErrorHandler`. Often used to log and swallow cache errors (cache should never break the app).

### Q11. What are the differences between Caffeine and Redis caches?

**Answer:** Caffeine is in-memory, per-JVM, fastest — good for single-node. Redis is external, distributed, shared across instances — required for multi-node. Caffeine can't share state; Redis adds network latency.

### Q12. How do you configure different TTLs for different caches?

**Answer:** With `RedisCacheManager` or `CaffeineCacheManager` and `withCacheConfiguration`:

```java
RedisCacheManager.builder(cf)
    .withCacheConfiguration("users", config().entryTtl(Duration.ofMinutes(5)))
    .withCacheConfiguration("products", config().entryTtl(Duration.ofHours(1)))
    .build();
```

### Q13. Why is caching nulls sometimes bad?

**Answer:** Caching nulls consumes memory and can hide data that later appears (e.g., a user created later). Use `unless = "#result == null"` to skip caching nulls.

### Q14. How does @Cacheable interact with @Transactional?

**Answer:** Cache advice runs **outside** the transaction. On miss, transaction opens, method runs, transaction commits, then result is cached. For write methods, `@CachePut` may update cache before commit — consider `@TransactionalEventListener(AFTER_COMMIT)` for correctness.

### Q15. What is the "self-invocation" problem in caching?

**Answer:** Calling a `@Cacheable` method from within the same bean bypasses the proxy, so caching is skipped. Same as `@Transactional`. Fix by splitting into another bean or injecting self.

### Q16. What is the default CacheManager in Spring Boot?

**Answer:** `ConcurrentMapCacheManager` (in-memory, no TTL, no eviction) when no provider is present. Fine for dev, not production.

### Q17. How do you cache collection results?

**Answer:** Same as any method — `@Cacheable` on a method returning `List<T>`. Be careful: the entire list is cached as one object. If any item changes, the whole list is stale. Better: cache individual items by key.

### Q18. What is a cache hit ratio and how do you monitor it?

**Answer:** `hits / (hits + misses)`. Monitor via Micrometer metrics (`cache.gets{result=hit|miss}`) exposed on `/actuator/metrics/cache.gets`. Aim >80% for effective caching.

### Q19. Can you cache Optional<User>?

**Answer:** Yes, but it caches the `Optional` wrapper. Note that Java `Optional` isn't `Serializable` by default — for Redis, disable `Optional` caching or provide a custom serializer.

### Q20. How do you invalidate the cache when data changes externally?

**Answer:** Options:
1. **TTL** — expire after N minutes
2. **Explicit eviction** via `@CacheEvict` in the write path
3. **Versioned keys** — include a version in the key
4. **Pub/sub** — broadcast invalidation (Redis pub/sub, Spring Cloud Bus)

---

## 17. Cheat Sheet

### Enable

```java
@SpringBootApplication
@EnableCaching
public class App { }
```

```properties
spring.cache.type=redis
spring.cache.redis.time-to-live=600000
```

### Annotations

```java
@Cacheable(value = "users", key = "#id", unless = "#result == null")
public User findById(Long id) { ... }

@CachePut(value = "users", key = "#user.id")
public User update(User user) { ... }

@CacheEvict(value = "users", key = "#id")
public void delete(Long id) { ... }

@CacheEvict(value = "users", allEntries = true)
public void clearUsers() { ... }

@Caching(
    put = { @CachePut(value = "users", key = "#user.id"),
            @CachePut(value = "usersByEmail", key = "#user.email") },
    evict = { @CacheEvict(value = "userList", allEntries = true) }
)
public User save(User user) { ... }
```

### SpEL Variables

```
#id, #a0, #p0, #user.email       → args
#result                          → return value (unless)
#root.method, #root.methodName
#root.target, #root.targetClass
#root.args, #root.caches
```

### Conditional

```java
condition = "#id > 0"             // before invocation
unless = "#result == null"        // after invocation
```

### Providers

| Type | Dependency | Distributed |
|------|-----------|-------------|
| Simple | (none) | ❌ |
| Caffeine | `caffeine` | ❌ |
| Ehcache | `ehcache` | ⚠️ |
| Redis | `spring-boot-starter-data-redis` | ✅ |
| Hazelcast | `spring-boot-starter-data-hazelcast` | ✅ |

### Redis Config

```properties
spring.cache.type=redis
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.cache.redis.time-to-live=600000
spring.cache.redis.cache-null-values=false
spring.cache.redis.key-prefix=myapp:
spring.cache.redis.use-key-prefix=true
```

### Custom Redis CacheManager

```java
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory cf) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(10))
        .disableCachingNullValues()
        .serializeKeysWith(fromSerializer(new StringRedisSerializer()))
        .serializeValuesWith(fromSerializer(new GenericJackson2JsonRedisSerializer()));
    return RedisCacheManager.builder(cf).cacheDefaults(config).build();
}
```

### Caffeine Config

```properties
spring.cache.type=caffeine
spring.cache.caffeine.spec=maximumSize=1000,expireAfterWrite=10m,recordStats
```

### Metrics

```bash
curl /actuator/metrics/cache.gets
curl /actuator/metrics/cache.gets?tag=name:users&tag=result:hit
curl /actuator/metrics/cache.evictions?tag=name:users
```

### Pitfalls Checklist

- [ ] No self-invocation
- [ ] `unless` for nulls
- [ ] Immutable cached objects
- [ ] Distinct caches/keys
- [ ] `sync=true` for hot keys
- [ ] Evict on writes
- [ ] TTL set
- [ ] `CacheErrorHandler` swallows errors
- [ ] Metrics monitored
- [ ] `spring.cache.type=none` in tests

### Cross-References

- **Previous:** `11_Spring_Data_JDBC.md`
- **Next:** `13_Spring_Security_Core.md`
- **Related:** `02_Spring_AOP.md`, `09_Spring_Boot_Actuator.md`, `10_Spring_Data_JPA.md` (L2 cache)
- **Interview:** `24_Spring_Interview_Questions.md`

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [11_Spring_Data_JDBC.md](./11_Spring_Data_JDBC.md)
- **Next →:** [13_Spring_Security_Core.md](./13_Spring_Security_Core.md)
- **Related:** [10_Spring_Data_JPA.md](./10_Spring_Data_JPA.md), [09_Spring_Boot_Actuator.md](./09_Spring_Boot_Actuator.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
