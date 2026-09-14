# Spring Data JDBC

> **File:** `11_Spring_Data_JDBC.md`
> **Part:** 3 — Data Access
> **Prerequisites:** `04_Spring_JDBC.md`, `05_Spring_Transaction_Management.md`
> **Estimated Study Time:** 5–6 hours

---

## Table of Contents

1. [What is Spring Data JDBC?](#1-what-is-spring-data-jdbc)
2. [Spring Data JDBC vs JPA](#2-spring-data-jdbc-vs-jpa)
3. [Aggregate Roots & DDD](#3-aggregate-roots--ddd)
4. [Setup & Dependencies](#4-setup--dependencies)
5. [Entity Mapping](#5-entity-mapping)
6. [CrudRepository](#6-crudrepository)
7. [Custom Queries](#7-custom-queries)
8. [Relationships](#8-relationships)
9. [Lifecycle Events](#9-lifecycle-events)
10. [Optimistic Locking](#10-optimistic-locking)
11. [When to Choose Spring Data JDBC](#11-when-to-choose-spring-data-jdbc)
12. [Interview Questions](#12-interview-questions)
13. [Cheat Sheet](#13-cheat-sheet)

---

## 1. What is Spring Data JDBC?

**Spring Data JDBC** is a simpler alternative to Spring Data JPA that maps entities directly to tables without the ORM overhead. It's based on **aggregate-oriented** design and **explicit** SQL semantics.

### Core Principles

- **Aggregates** — the unit of consistency (root + children)
- **No lazy loading** — no proxies, no sessions
- **No caching** — always hits the DB
- **No dirty checking** — you save explicitly
- **Predictable SQL** — you know what executes
- **DDD-friendly** — models follow aggregate boundaries

### What It's Not

- Not a JPA replacement for complex ORMs
- Not for legacy schemas with foreign key spaghetti
- Not for lazy loading or caching-heavy workloads

---

## 2. Spring Data JDBC vs JPA

| Feature | Spring Data JDBC | Spring Data JPA |
|---------|-----------------|-----------------|
| ORM | ❌ | ✅ (Hibernate) |
| Lazy loading | ❌ | ✅ |
| Dirty checking | ❌ | ✅ |
| First-level cache | ❌ | ✅ |
| Second-level cache | ❌ | ✅ |
| Proxies | ❌ | ✅ |
| Aggregate root concept | ✅ Central | Optional |
| Relationship cascade | Aggregate-driven | `CascadeType` |
| N+1 problem | ❌ (by design) | ✅ Common |
| Learning curve | Low | High |
| Predictability | High | Medium |
| Complex relations | Limited | Full |
| DDL generation | Basic | Full |
| Startup time | Fast | Slower |

### When JPA Wins

- Deep object graphs
- Legacy relational schemas
- Fine-grained caching
- Complex bidirectional relationships

### When Spring Data JDBC Wins

- Aggregate-oriented models
- Simple data + explicit SQL
- Avoiding ORM complexity
- Functional domain models
- Microservices with clear boundaries

---

## 3. Aggregate Roots & DDD

### The DDD Aggregate

An **aggregate** is a cluster of domain objects treated as a single unit for data changes. The **aggregate root** is the only entry point.

### Rules

1. Each aggregate has one **root entity**
2. Only the root is loaded directly
3. References to other aggregates are by **ID**, not object
4. Deleting root deletes the whole aggregate
5. Transactional consistency within aggregate; eventual consistency across

### Example

```
Customer (root)
├── Address (embedded)
├── Order (child)
│   └── OrderLine (child of Order)
└── PaymentMethod (child)
```

Loading `Customer` loads the entire aggregate. To load `Order` alone, you'd make it a separate aggregate with a `customerId` reference.

### Spring Data JDBC Enforces This

- Only aggregate roots are repositories
- Children aren't repositories
- Cross-aggregate references are IDs (`Long customerId`)
- Writes save the whole aggregate (replace children)

---

## 4. Setup & Dependencies

### Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

Pulls in:
- `spring-data-jdbc`
- `spring-data-relational`
- `spring-boot-starter-jdbc` (HikariCP + JdbcTemplate)

### Configuration

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
spring.datasource.password=secret
spring.datasource.hikari.maximum-pool-size=20
```

### Enable Repositories

Spring Boot auto-detects. Manual:

```java
@EnableJdbcRepositories(basePackages = "com.example.repo")
@Configuration
public class JdbcConfig {
    @Bean
    public DataSource dataSource() { ... }
    @Bean
    public NamedParameterJdbcOperations operations(DataSource ds) {
        return new NamedParameterJdbcTemplate(ds);
    }
    @Bean
    public PlatformTransactionManager txManager(DataSource ds) {
        return new DataSourceTransactionManager(ds);
    }
}
```

### Schema

Spring Data JDBC does **not** generate DDL. Create schema via:

- Flyway / Liquibase
- `schema.sql` (Boot picks it up automatically)

```sql
CREATE TABLE customer (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    total NUMERIC(10,2)
);

CREATE TABLE order_line (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,
    product VARCHAR(100),
    quantity INT
);
```

---

## 5. Entity Mapping

### Minimal Entity

```java
@Table("customer")
public class Customer {
    @Id
    private Long id;
    private String name;
    private String email;
    // getters/setters or constructor
}
```

### Rules

- `@Table(name)` — optional (default = class name, camelCase → snake_case)
- `@Id` — required (name = `id` by default)
- Fields → columns (camelCase → snake_case by default)
- **No `@Column` needed unless overriding**
- Constructor-based creation supported (preferred)

### Constructor Binding (Recommended)

```java
@Table("customer")
public class Customer {
    @Id private final Long id;
    private final String name;
    private final String email;
    
    public Customer(Long id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }
    
    @PersistenceCreator
    public Customer(String name, String email) {
        this(null, name, email);
    }
    
    // getters only
}
```

### Immutable with Records (Boot 3+)

```java
@Table("customer")
public record Customer(
    @Id Long id, 
    String name, 
    String email
) { }
```

### @Column

```java
@Column("full_name")
private String name;
```

### @Id Generation

Spring Data JDBC doesn't generate IDs. Options:

| Strategy | Configuration |
|----------|--------------|
| Database auto-increment | Column `BIGSERIAL` / `AUTO_INCREMENT` |
| Manual | You set ID before save |
| `@Id` + database sequence | Use DB default |

Unlike JPA, there's no `@GeneratedValue`. The DB generates IDs and Spring reads them back.

### Embedded Value Objects

```java
public class Address {
    private final String street;
    private final String city;
    private final String zip;
    // constructor, getters
}

@Table("customer")
public class Customer {
    @Id private final Long id;
    private final String name;
    private final Address address;   // embedded (columns in same table)
}
```

Columns: `street`, `city`, `zip` in `customer` table.

### Custom Naming Strategy

```java
@Configuration
public class JdbcConfig {
    @Bean
    public NamingStrategy namingStrategy() {
        return NamingStrategy.INSTANCE;   // or custom
    }
}
```

Or via `@Column` per field.

---

## 6. CrudRepository

Spring Data JDBC uses `CrudRepository` (not `JpaRepository`).

### Repository

```java
public interface CustomerRepository extends CrudRepository<Customer, Long> {
}
```

### CRUD Methods

| Method | Effect |
|--------|--------|
| `save(entity)` | Insert or update aggregate |
| `saveAll(Iterable)` | Batch |
| `findById(id)` | Load whole aggregate |
| `findAll()` | Load all aggregates |
| `count()` | Row count |
| `existsById(id)` | Existence |
| `deleteById(id)` | Delete aggregate |
| `delete(entity)` | Delete |
| `deleteAll()` | Delete all |

### Save Semantics

- **New** (id == null) → INSERT
- **Existing** (id != null) → UPDATE (or DELETE + INSERT for children)

Child collections are **replaced** on save — Spring Data JDBC issues DELETE for old children, INSERT for new ones.

### Example

```java
@Service
public class CustomerService {
    private final CustomerRepository repo;
    
    public CustomerService(CustomerRepository repo) { this.repo = repo; }
    
    @Transactional
    public Customer create(String name, String email) {
        Customer c = new Customer(null, name, email);
        return repo.save(c);   // INSERT, returns entity with generated ID
    }
    
    public Optional<Customer> find(Long id) { return repo.findById(id); }
    
    @Transactional
    public Customer update(Long id, String name) {
        Customer c = repo.findById(id).orElseThrow();
        Customer updated = new Customer(c.id(), name, c.email());
        return repo.save(updated);
    }
    
    @Transactional
    public void delete(Long id) { repo.deleteById(id); }
}
```

### PagingAndSortingRepository

Spring Data JDBC supports `PagingAndSortingRepository`:

```java
public interface CustomerRepository extends PagingAndSortingRepository<Customer, Long> {
}
```

Query:

```java
Page<Customer> page = repo.findAll(PageRequest.of(0, 20, Sort.by("name")));
```

Or `ListPagingAndSortingRepository` (Boot 3+) for `List` returns.

---

## 7. Custom Queries

### Derived Query Methods

Same syntax as JPA:

```java
public interface CustomerRepository extends CrudRepository<Customer, Long> {
    Optional<Customer> findByEmail(String email);
    List<Customer> findByNameContaining(String part);
    List<Customer> findByEmailEndingWith(String domain);
    long countByNameStartingWith(String prefix);
    boolean existsByEmail(String email);
    void deleteByEmail(String email);
}
```

### @Query (Native SQL Only)

⚠️ Spring Data JDBC **does not support JPQL**. Use **native SQL**.

```java
@Query("SELECT id, name, email FROM customer WHERE email = :email")
Optional<Customer> findByEmail(@Param("email") String email);

@Query("""
    SELECT c.id, c.name, c.email
    FROM customer c
    JOIN orders o ON o.customer_id = c.id
    WHERE o.total > :minTotal
    """)
List<Customer> findBigSpenders(@Param("minTotal") BigDecimal minTotal);
```

### Modifying Queries

```java
@Modifying
@Query("UPDATE customer SET email = :email WHERE id = :id")
int updateEmail(@Param("id") Long id, @Param("email") String email);

@Modifying
@Query("DELETE FROM customer WHERE email IS NULL")
int deleteWithoutEmail();
```

Must be called within a transaction.

### Query for Projections

```java
public interface CustomerNameOnly {
    Long getId();
    String getName();
}

@Query("SELECT id, name FROM customer WHERE email IS NOT NULL")
List<CustomerNameOnly> findWithEmail();
```

### Query for Aggregates with Children

Spring Data JDBC maps result sets to aggregates. If your query returns multiple joined rows, Spring assembles them:

```java
@Query("""
    SELECT c.id AS customer_id, c.name, 
           o.id AS order_id, o.total
    FROM customer c
    LEFT JOIN orders o ON o.customer_id = c.id
    WHERE c.id = :id
    """)
Optional<Customer> findWithOrders(@Param("id") Long id);
```

Aliases like `customer_id`, `order_id` help Spring disambiguate.

### Named Queries

Not fully supported — use `@Query` or derived methods.

### Query Lookup Strategy

```java
@EnableJdbcRepositories(queryLookupStrategy = QueryLookupStrategy.Key.CREATE_IF_NOT_FOUND)
```

---

## 8. Relationships

### The Aggregate Rule

Spring Data JDBC models aggregates; children are part of the root. Cross-aggregate references are **by ID**.

### One-to-Many (within aggregate)

```java
@Table("customer")
public class Customer {
    @Id private Long id;
    private String name;
    
    @MappedCollection(idColumn = "customer_id")
    private Set<Order> orders = new HashSet<>();
}

@Table("orders")
public class Order {
    @Id private Long id;
    private BigDecimal total;
    // No reference back to Customer!
}
```

Key points:
- `@MappedCollection(idColumn = "customer_id")` — the FK column in `orders`
- Children have no parent reference (no `@ManyToOne`)
- Loading customer loads all orders
- Saving customer saves orders

### One-to-One (within aggregate)

```java
@Table("customer")
public class Customer {
    @Id private Long id;
    
    @MappedCollection(idColumn = "customer_id")
    private Address address;   // columns in address table
}

@Table("address")
public class Address {
    @Id private Long id;
    private String street;
    // Spring Data JDBC treats this as a 1:1
}
```

Or embed the address as a value object (see §5).

### Many-to-Many

**Not directly supported.** Model as separate aggregates:

```java
@Table("customer")
public class Customer {
    @Id private Long id;
    private Set<Long> roleIds = new HashSet<>();   // IDs, not objects
}
```

Or use a join entity within the aggregate:

```java
@Table("customer")
public class Customer {
    @Id private Long id;
    
    @MappedCollection(idColumn = "customer_id")
    private Set<CustomerRole> roles = new HashSet<>();
}

@Table("customer_role")
public class CustomerRole {
    @Id private Long id;
    private Long roleId;
}
```

### Cross-Aggregate Reference (by ID)

```java
@Table("orders")
public class Order {
    @Id private Long id;
    private Long customerId;   // reference, not @ManyToOne
    private BigDecimal total;
}

public interface OrderRepository extends CrudRepository<Order, Long> {
    List<Order> findByCustomerId(Long customerId);
}
```

### @MappedCollection Options

```java
@MappedCollection(
    idColumn = "customer_id",         // FK column name
    keyColumn = "position"            // for Map/List ordering
)
private List<Order> orders;
```

- **Set** — unordered; DB stores rows
- **List** — requires `keyColumn` for ordering
- **Map** — keys stored in `keyColumn`

### Deletion Behavior

When you `deleteById(customer)`, Spring Data JDBC:
1. Deletes all children (cascading within aggregate)
2. Deletes the root

No `CascadeType` needed — it's implicit for the aggregate.

---

## 9. Lifecycle Events

Spring Data JDBC supports `@BeforeSave`, `@AfterSave`, `@BeforeDelete`, and Spring's `ApplicationListener`.

### Entity Callbacks

```java
@Table("customer")
public class Customer {
    @Id private Long id;
    private String name;
    private Instant createdAt;
    private Instant updatedAt;
    
    @BeforeSave
    void beforeSave() {
        if (createdAt == null) createdAt = Instant.now();
        updatedAt = Instant.now();
    }
}
```

Callbacks:

| Annotation | When |
|-----------|------|
| `@BeforeSave` | Before insert/update |
| `@AfterSave` | After insert/update |
| `@BeforeDelete` | Before delete |
| `@AfterDelete` | After delete |
| `@AfterLoad` | After entity loaded |

### Application Events

```java
@Component
public class CustomerListener {
    @EventListener
    public void onAfterSave(AfterSaveEvent<Customer> event) {
        log.info("Saved: {}", event.getEntity());
    }
}
```

Events extend `RelationalEvent` and include the entity, id, and source.

### Auditing

Spring Data JDBC supports auditing similarly to JPA:

```java
@Configuration
@EnableJdbcAuditing
public class AuditConfig {
    @Bean
    public AuditorAware<String> auditor() {
        return () -> Optional.of("system");
    }
}
```

```java
@Table("customer")
public class Customer {
    @Id private Long id;
    
    @CreatedDate @Column(updatable = false)
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant updatedAt;
    
    @CreatedBy
    private String createdBy;
}
```

---

## 10. Optimistic Locking

### @Version

```java
@Table("customer")
public class Customer {
    @Id private Long id;
    private String name;
    
    @Version
    private Long version;
}
```

Spring Data JDBC adds `AND version = ?` to UPDATE. On mismatch → `OptimisticLockingFailureException`.

### Behavior

- On insert: version set to `0`
- On update: `version = version + 1`
- Concurrent updates: one succeeds, one throws

### Handling Failures

```java
@Retryable(retryFor = OptimisticLockingFailureException.class, maxAttempts = 3)
@Transactional
public Customer update(Long id, String name) {
    Customer c = repo.findById(id).orElseThrow();
    return repo.save(new Customer(c.id(), name, c.version()));
}
```

Requires `spring-retry` + `@EnableRetry`.

---

## 11. When to Choose Spring Data JDBC

### Choose Spring Data JDBC If

- ✅ Your domain is aggregate-oriented
- ✅ You want predictable, fast SQL
- ✅ You dislike ORM magic (proxies, lazy loading)
- ✅ Your relationships fit within aggregates
- ✅ You want to avoid the N+1 problem by design
- ✅ Startup time matters
- ✅ You're building microservices with clear boundaries

### Choose Spring Data JPA If

- ✅ Complex bidirectional relationships across aggregates
- ✅ Legacy schemas with intricate FKs
- ✅ Need lazy loading for large graphs
- ✅ Need fine-grained caching
- ✅ Rich domain model with proxies and tracking

### Hybrid Approach

You can use **both** in the same app:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jdbc</artifactId>
</dependency>
```

Use JPA for complex aggregates; Spring Data JDBC (or plain JdbcTemplate) for reports and simple reads. Configure separate repositories with `@EnableJpaRepositories` and `@EnableJdbcRepositories` in different packages.

---

## 12. Interview Questions

### Q1. What is Spring Data JDBC?

**Answer:** A simpler alternative to Spring Data JPA based on aggregate-oriented DDD. It maps entities directly to tables without ORM (no lazy loading, no dirty checking, no caching). Uses `CrudRepository` and native SQL `@Query`.

### Q2. Difference between Spring Data JDBC and Spring Data JPA?

**Answer:** JPA uses an ORM (Hibernate) with lazy loading, dirty checking, caching, and entity lifecycle. Spring Data JDBC has none of these — it saves/loads complete aggregates explicitly. JPA is more powerful but complex; JDBC is simpler and predictable.

### Q3. What is an aggregate root?

**Answer:** A DDD concept: the single entry point to a cluster of related objects treated as a unit. Only the root has a repository. Children are loaded/saved with the root. References to other aggregates are by ID.

### Q4. How does Spring Data JDBC handle relationships?

**Answer:** Children within an aggregate are mapped with `@MappedCollection(idColumn = "...")`. Cross-aggregate references are stored as plain IDs (`Long customerId`). No `@ManyToOne` — that would create a cross-aggregate dependency.

### Q5. Does Spring Data JDBC support JPQL?

**Answer:** No. It only supports **native SQL** in `@Query`. The underlying query language is SQL, not JPQL.

### Q6. How do you create custom queries?

**Answer:** Derived method names (same as JPA) for simple queries. `@Query` with native SQL for complex queries. `@Modifying` for UPDATE/DELETE.

### Q7. How do you generate IDs?

**Answer:** Spring Data JDBC doesn't generate IDs — the database does (`BIGSERIAL`, `AUTO_INCREMENT`). Spring reads them back. There's no `@GeneratedValue` annotation.

### Q8. What is @MappedCollection?

**Answer:** Marks a collection/embedded field within an aggregate. `idColumn` specifies the FK column in the child table. Optional `keyColumn` for List/Map ordering.

### Q9. Does Spring Data JDBC support optimistic locking?

**Answer:** Yes, via `@Version`. Spring adds `AND version = ?` to UPDATEs. On mismatch, `OptimisticLockingFailureException` is thrown.

### Q10. What are the lifecycle callbacks?

**Answer:** `@BeforeSave`, `@AfterSave`, `@BeforeDelete`, `@AfterDelete`, `@AfterLoad`. Plus ApplicationEvents (`AfterSaveEvent`, `BeforeDeleteEvent`, etc.).

### Q11. Does Spring Data JDBC support auditing?

**Answer:** Yes. `@EnableJdbcAuditing` + `AuditorAware<String>` bean, then `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy` on entities.

### Q12. How do you save an aggregate?

**Answer:** `repository.save(rootEntity)` — Spring deletes all children then re-inserts them (or updates if IDs match). This is intentional: aggregate is the unit of persistence.

### Q13. What happens to children on save?

**Answer:** Spring Data JDBC **replaces** children: it deletes children no longer in the collection and inserts new ones. If a child's ID is unchanged, it may issue an UPDATE. This is different from JPA's incremental behavior.

### Q14. Can Spring Data JDBC and JPA coexist?

**Answer:** Yes. Configure `@EnableJpaRepositories` for one package and `@EnableJdbcRepositories` for another. Use JPA for complex aggregates; JDBC for simple reads/reports.

### Q15. When would you choose Spring Data JDBC over JPA?

**Answer:** When your domain is naturally aggregate-oriented, you want predictable SQL, you dislike ORM magic (proxies, sessions, lazy loading), you need fast startup, and cross-aggregate relationships are minimal.

### Q16. Does Spring Data JDBC do DDL generation?

**Answer:** No. You must provide schema via Flyway/Liquibase or `schema.sql`. Spring Data JDBC only maps existing tables to entities.

### Q17. How is pagination supported?

**Answer:** By extending `PagingAndSortingRepository` (or `ListPagingAndSortingRepository`). Use `PageRequest` and `Sort` the same way as JPA. Internally Spring generates `LIMIT/OFFSET` (or DB-specific) SQL.

### Q18. What is @PersistenceCreator?

**Answer:** Marks a constructor Spring uses when reconstructing an entity from a `ResultSet`. Useful when there are multiple constructors — Spring might otherwise pick the wrong one.

### Q19. How does Spring Data JDBC avoid the N+1 problem?

**Answer:** By design: it doesn't lazy-load. Loading an aggregate loads all children in one join. Cross-aggregate references must be explicitly fetched (creating N+1 if you loop). The framework doesn't hide this; you control it.

### Q20. Does Spring Data JDBC support transactions?

**Answer:** Yes — with `DataSourceTransactionManager` (or the same `PlatformTransactionManager` used for JDBC). Annotate service methods with `@Transactional`.

---

## 13. Cheat Sheet

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jdbc</artifactId>
</dependency>
```

### Entity

```java
@Table("customer")
public record Customer(
    @Id Long id,
    String name,
    @MappedCollection(idColumn = "customer_id")
    Set<Order> orders
) { }
```

### Repository

```java
public interface CustomerRepository extends CrudRepository<Customer, Long> {
    Optional<Customer> findByEmail(String email);
    
    @Query("SELECT * FROM customer WHERE name LIKE :pattern")
    List<Customer> searchByName(@Param("pattern") String pattern);
    
    @Modifying
    @Query("UPDATE customer SET email = :email WHERE id = :id")
    int updateEmail(@Param("id") Long id, @Param("email") String email);
}
```

### Annotations

| Annotation | Purpose |
|-----------|---------|
| `@Table` | Maps to table |
| `@Id` | Primary key |
| `@Column` | Override column name |
| `@MappedCollection` | Child collection within aggregate |
| `@Version` | Optimistic locking |
| `@PersistenceCreator` | Constructor for reconstitution |
| `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy` | Auditing |
| `@BeforeSave`, `@AfterSave`, `@BeforeDelete`, `@AfterDelete`, `@AfterLoad` | Lifecycle |

### Aggregates Cheat

```
- Only roots have repositories
- Children loaded/saved with root via @MappedCollection
- Cross-aggregate references = plain ID field
- No lazy loading, no proxies, no sessions
- Save replaces children (delete + insert)
```

### Properties

```properties
spring.datasource.url=jdbc:postgresql://...
spring.datasource.username=admin
spring.datasource.password=secret
```

### Spring Data JDBC vs JPA — Quick Pick

| Need | Choose |
|------|--------|
| Simple aggregates | JDBC |
| Complex bidirectional graphs | JPA |
| Predictable SQL | JDBC |
| Lazy loading | JPA |
| Fast startup | JDBC |
| Caching (L2) | JPA |
| DDD aggregates | JDBC |

### Cross-References

- **Previous:** `10_Spring_Data_JPA.md`
- **Next:** `12_Spring_Caching.md`
- **Related:** `04_Spring_JDBC.md`, `05_Spring_Transaction_Management.md`
- **Interview:** `24_Spring_Interview_Questions.md`

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [10_Spring_Data_JPA.md](./10_Spring_Data_JPA.md)
- **Next →:** [12_Spring_Caching.md](./12_Spring_Caching.md)
- **Related:** [10_Spring_Data_JPA.md](./10_Spring_Data_JPA.md), [04_Spring_JDBC.md](./04_Spring_JDBC.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
