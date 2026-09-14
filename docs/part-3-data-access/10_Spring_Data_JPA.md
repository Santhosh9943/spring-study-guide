# Spring Data JPA & Hibernate

> **File:** `10_Spring_Data_JPA.md`
> **Part:** 3 — Data Access
> **Prerequisites:** `04_Spring_JDBC.md`, `05_Spring_Transaction_Management.md`
> **Estimated Study Time:** 16–20 hours (largest file in this guide)

---

## Table of Contents

1. [ORM & Impedance Mismatch](#1-orm--impedance-mismatch)
2. [JPA vs Hibernate vs Spring Data JPA](#2-jpa-vs-hibernate-vs-spring-data-jpa)
3. [Entity Mapping Basics](#3-entity-mapping-basics)
4. [Relationships](#4-relationships)
5. [Cascade Types & Orphan Removal](#5-cascade-types--orphan-removal)
6. [Fetch Types & N+1 Problem](#6-fetch-types--n1-problem)
7. [Entity Lifecycle](#7-entity-lifecycle)
8. [Persistence Context & Dirty Checking](#8-persistence-context--dirty-checking)
9. [Repository Interfaces](#9-repository-interfaces)
10. [Derived Query Methods](#10-derived-query-methods)
11. [@Query (JPQL)](#11-query-jpql)
12. [@Query (Native SQL)](#12-query-native-sql)
13. [Pagination & Sorting](#13-pagination--sorting)
14. [Projections](#14-projections)
15. [Specifications & Criteria API](#15-specifications--criteria-api)
16. [Entity Graphs](#16-entity-graphs)
17. [Auditing](#17-auditing)
18. [Locking](#18-locking)
19. [Caching (L1, L2, Query)](#19-caching-l1-l2-query)
20. [Interview Questions](#20-interview-questions)
21. [Cheat Sheet](#21-cheat-sheet)

---

## 1. ORM & Impedance Mismatch

**ORM** (Object-Relational Mapping) bridges the gap between object-oriented Java and relational databases.

### The Impedance Mismatch

| OOP | Relational |
|-----|-----------|
| Classes | Tables |
| Objects | Rows |
| Fields | Columns |
| References | Foreign keys |
| Inheritance | No direct mapping |
| Polymorphism | No direct mapping |
| Encapsulation | Public columns |
| Identity (==) | Primary key |

### What ORM Solves

- Automatic SQL generation
- Object navigation (no manual joins)
- Identity map (same object for same row)
- Change tracking (dirty checking)
- Lazy loading
- Caching
- Transaction integration

### What ORM Doesn't Solve

- N+1 problem (must handle)
- Bulk operations (often use JDBC)
- Complex reports (native SQL)
- Performance tuning (still needed)

---

## 2. JPA vs Hibernate vs Spring Data JPA

| Layer | What | Who |
|-------|------|-----|
| **JPA** | Specification (API) | Jakarta EE / JCP |
| **Hibernate** | Implementation of JPA | Red Hat |
| **Spring Data JPA** | Repository abstraction on top of JPA | Spring |

### JPA

- Specification defining annotations and interfaces
- `@Entity`, `@Id`, `EntityManager`, `EntityManagerFactory`
- No implementation

### Hibernate

- Most popular JPA provider
- Implements `EntityManager`
- Adds extensions (`@Formula`, `@BatchSize`, `Session` API)
- Manages SQL generation, caching, dirty checking

### Spring Data JPA

- Simplifies repository creation
- `JpaRepository`, method-name query derivation
- `@Query`, `Specification`, `Sort`, `Pageable`
- Uses Hibernate (or other JPA provider) internally

### Stack Diagram

```
┌──────────────────────────────────────┐
│     Your Service / Repository        │
├──────────────────────────────────────┤
│         Spring Data JPA              │  ← Repository interfaces
├──────────────────────────────────────┤
│              JPA API                 │  ← @Entity, EntityManager
├──────────────────────────────────────┤
│             Hibernate                │  ← Actual ORM
├──────────────────────────────────────┤
│           JDBC Driver                │
└──────────────────────────────────────┘
```

---

## 3. Entity Mapping Basics

### Minimal Entity

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "full_name", nullable = false, length = 100)
    private String name;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    // getters/setters, equals/hashCode
}
```

### Required Annotations

| Annotation | Purpose |
|-----------|---------|
| `@Entity` | Marks class as persistent |
| `@Table(name = "...")` | Maps to specific table |
| `@Id` | Primary key |
| `@GeneratedValue` | ID generation strategy |

### ID Generation Strategies

| Strategy | Use Case |
|----------|----------|
| `IDENTITY` | Auto-increment column (MySQL, PostgreSQL) |
| `SEQUENCE` | DB sequence (Oracle, PostgreSQL) |
| `TABLE` | Table-based (portable, slow) |
| `AUTO` | Provider chooses (Hibernate → SEQUENCE or TABLE) |
| `UUID` | Hibernate 6 supports UUID natively |

```java
// IDENTITY
@Id @GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// SEQUENCE
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(name = "user_seq", sequenceName = "users_seq", allocationSize = 50)
private Long id;

// UUID
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;
```

> **Important:** `IDENTITY` disables JDBC batch inserts (Hibernate can't know ID before insert). Use `SEQUENCE` for batch inserts.

### @Column Options

```java
@Column(
    name = "email",
    nullable = false,
    unique = true,
    length = 100,
    precision = 10, scale = 2,             // for BigDecimal
    columnDefinition = "VARCHAR(100) NOT NULL",
    insertable = true, updatable = true,
    table = "users"
)
private String email;
```

### @Transient

Not persisted:

```java
@Transient
private String computedField;
```

### @Enumerated

```java
@Enumerated(EnumType.STRING)   // stores name (recommended)
private Status status;

@Enumerated(EnumType.ORDINAL)  // stores index (fragile!)
private Status status;
```

### @Temporal (legacy)

For `java.util.Date`:

```java
@Temporal(TemporalType.TIMESTAMP)
private Date created;
```

Modern: use `java.time.*` (LocalDate, LocalDateTime, Instant) — no `@Temporal` needed.

### @Lob

For large objects (CLOB / BLOB):

```java
@Lob private String longText;
@Lob private byte[] image;
```

### @Embedded / @Embeddable

Compose value objects:

```java
@Embeddable
public class Address {
    private String street;
    private String city;
    private String zip;
    // getters/setters
}

@Entity
public class Customer {
    @Id @GeneratedValue private Long id;
    
    @Embedded
    private Address address;
}
```

Columns appear in `customer` table.

### @AttributeOverride

Override column names for embedded:

```java
@Embedded
@AttributeOverrides({
    @AttributeOverride(name = "street", column = @Column(name = "ship_street")),
    @AttributeOverride(name = "city", column = @Column(name = "ship_city"))
})
private Address shippingAddress;
```

### @ElementCollection

Collections of simple types or embeddables:

```java
@ElementCollection
@CollectionTable(name = "user_phones", joinColumns = @JoinColumn(name = "user_id"))
@Column(name = "phone")
private List<String> phones = new ArrayList<>();
```

Creates `user_phones` table.

### equals() / hashCode()

For entities, use the **business key** or **ID if available**:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof User)) return false;
    User other = (User) o;
    return id != null && id.equals(other.id);
}

@Override
public int hashCode() {
    return getClass().hashCode();   // stable across persistence
}
```

**Why `getClass().hashCode()`?** ID is null before persist — using ID in `hashCode()` breaks `Set` semantics.

### Lombok Integration

```java
@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)   // JPA requires no-arg ctor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class User {
    @Id @GeneratedValue
    @EqualsAndHashCode.Include
    private Long id;
    private String name;
}
```

⚠️ Avoid Lombok `@Data` on entities — generated `equals/hashCode` includes all fields, which breaks with lazy proxies and mutations.

---

## 4. Relationships

### @ManyToOne (the most common)

```java
@Entity
public class Order {
    @Id @GeneratedValue private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)   // always LAZY
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;
}
```

### @OneToMany (inverse side)

```java
@Entity
public class Customer {
    @Id @GeneratedValue private Long id;
    
    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
    
    public void addOrder(Order o) {
        orders.add(o);
        o.setCustomer(this);
    }
}
```

### @OneToOne

```java
@Entity
public class User {
    @Id @GeneratedValue private Long id;
    
    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id", unique = true)
    private Profile profile;
}
```

### @ManyToMany

```java
@Entity
public class Student {
    @Id @GeneratedValue private Long id;
    
    @ManyToMany
    @JoinTable(
        name = "student_courses",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}
```

⚠️ `@ManyToMany` is often a design smell — use a join entity with extra columns if relationship has attributes.

### Owning vs Inverse Side

- **Owning side** — has `@JoinColumn` or `@JoinTable`; changes to the relationship are persisted
- **Inverse side** — has `mappedBy`; changes are ignored for FK updates

### Relationship Comparison

| Annotation | FK Location | Default Fetch | Multiplicity |
|-----------|-------------|---------------|--------------|
| `@ManyToOne` | Owning table | EAGER | N → 1 |
| `@OneToMany` | Other table | LAZY | 1 → N |
| `@OneToOne` | Owning (or other with `mappedBy`) | EAGER | 1 → 1 |
| `@ManyToMany` | Join table | LAZY | N → N |

### Always LAZY

Default `EAGER` on `@ManyToOne` and `@OneToOne` causes N+1. Always use `LAZY` and fetch explicitly when needed.

### Bidirectional Best Practice

```java
public void addOrder(Order o) {
    orders.add(o);
    o.setCustomer(this);
}

public void removeOrder(Order o) {
    orders.remove(o);
    o.setCustomer(null);
}
```

Keep both sides in sync.

---

## 5. Cascade Types & Orphan Removal

### Cascade Types

| Cascade | Effect |
|---------|--------|
| `PERSIST` | Persist parent → persist children |
| `MERGE` | Merge parent → merge children |
| `REMOVE` | Remove parent → remove children |
| `REFRESH` | Refresh parent → refresh children |
| `DETACH` | Detach parent → detach children |
| `ALL` | All of the above |

```java
@OneToMany(mappedBy = "customer", cascade = CascadeType.ALL)
private List<Order> orders;
```

### orphanRemoval

When a child is removed from the collection, it's deleted from DB:

```java
@OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Order> orders;

// Removing from collection deletes the row
customer.getOrders().remove(order);
```

### Cascade vs OrphanRemoval

| Aspect | Cascade REMOVE | orphanRemoval |
|--------|---------------|--------------|
| Triggered by | Parent deletion | Child removed from collection |
| Scope | Deleting parent | Deleting child directly |
| Common use | Aggregate root deletion | Composition |

### Warning

- **Never** use `CascadeType.REMOVE` or `ALL` on `@ManyToMany` — deleting one parent deletes shared children
- **Never** cascade `REMOVE` from `@ManyToOne` (would delete the referenced parent)

---

## 6. Fetch Types & N+1 Problem

### LAZY vs EAGER

| Type | Behavior |
|------|----------|
| `LAZY` | Load on first access (via proxy) |
| `EAGER` | Load immediately with parent |

### Defaults

| Annotation | Default |
|-----------|---------|
| `@ManyToOne` | EAGER |
| `@OneToOne` | EAGER |
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |

### The N+1 Problem

Loading N parents triggers N additional queries for children:

```java
List<Customer> customers = customerRepo.findAll();   // 1 query
for (Customer c : customers) {
    c.getOrders().size();                             // N queries!
}
```

Total: **N+1 queries**.

### LazyInitializationException

Accessing a lazy association outside the persistence context (e.g., after transaction ends, in controller):

```java
// In service (transactional)
User user = userService.find(id);   // transaction ends

// In controller (no transaction)
user.getOrders().size();            // LazyInitializationException!
```

### Fixes for N+1

#### 1. JOIN FETCH

```java
@Query("SELECT DISTINCT c FROM Customer c JOIN FETCH c.orders WHERE c.id = :id")
Customer findWithOrders(@Param("id") Long id);
```

#### 2. @EntityGraph

```java
@EntityGraph(attributePaths = {"orders", "address"})
List<Customer> findAll();
```

#### 3. @BatchSize

Fetches children in batches:

```java
@BatchSize(size = 20)
@OneToMany(mappedBy = "customer")
private List<Order> orders;
```

For 100 customers → 5 queries (each fetching 20 customers' orders).

#### 4. @Fetch(FetchMode.SUBSELECT)

Hibernate-specific:

```java
@Fetch(FetchMode.SUBSELECT)
@OneToMany(mappedBy = "customer")
private List<Order> orders;
```

Issues a single subquery for all parents' children.

#### 5. DTO Projection

Return only needed columns:

```java
@Query("SELECT new com.example.CustomerDto(c.id, c.name, COUNT(o)) " +
       "FROM Customer c LEFT JOIN c.orders o GROUP BY c.id, c.name")
List<CustomerDto> findCustomerSummaries();
```

### Detecting N+1

- Enable SQL logging: `spring.jpa.show-sql=true`, `spring.jpa.properties.hibernate.format_sql=true`
- Set `logging.level.org.hibernate.SQL=DEBUG`
- Use **datasource-proxy** or **p6spy** to count queries per request
- Hypothesis: any loop over entities accessing lazy collections

### Open Session in View (OSIV)

Spring Boot's `spring.jpa.open-in-view=true` (default true) keeps the session open during view rendering — **hides** N+1 problems but causes long-held connections.

**Best practice:** `spring.jpa.open-in-view=false`. Handle fetching in service layer.

---

## 7. Entity Lifecycle

### States

| State | Meaning |
|-------|---------|
| **Transient (New)** | Not associated with persistence context, no ID |
| **Managed (Persistent)** | In persistence context, changes tracked |
| **Detached** | Was managed, no longer associated |
| **Removed** | Marked for deletion |

### Transitions

```
                  persist()
  Transient ─────────────────► Managed
                                   │
                                   │ merge()
                    ◄──────────────┤
                (returns copy)     │
                                   │
                                   │ detach() / clear() / close()
                                   ▼
                                Detached
                                   │
                                   │ merge()
                                   ▼
                                Managed
                                   
  Managed ─────remove()─────► Removed ──flush()──► Deleted
```

### EntityManager Methods

| Method | Effect |
|--------|--------|
| `persist(entity)` | New → Managed |
| `merge(entity)` | Detached → Managed (returns copy) |
| `remove(entity)` | Managed → Removed |
| `detach(entity)` | Managed → Detached |
| `refresh(entity)` | Reload from DB |
| `contains(entity)` | Is it managed? |
| `flush()` | Sync to DB |
| `clear()` | Detach all |

### Repository Operations Mapping

| Repo Method | EM Operation |
|-------------|-------------|
| `save(newEntity)` | `persist` |
| `save(existingEntity)` | `merge` |
| `delete(entity)` | `remove` |
| `findById(id)` | `find` |
| `getById(id)` | `getReference` (proxy) |
| `flush()` | `flush` |

### `save()` Behavior

```java
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        em.persist(entity);
        return entity;
    } else {
        return em.merge(entity);
    }
}
```

Determines "new" by checking if `@Id` is null (or `Persistable.isNew()`).

### Custom `isNew()`

```java
@Entity
public class User implements Persistable<Long> {
    @Id private Long id;
    @Transient private boolean isNew;
    
    @PostLoad @PostPersist
    void markNotNew() { this.isNew = false; }
    
    @Override public boolean isNew() { return isNew; }
}
```

Useful when IDs are assigned externally.

---

## 8. Persistence Context & Dirty Checking

### Persistence Context

A first-level cache holding managed entities per transaction. Same ID → same object instance.

```java
User u1 = repo.findById(1L).get();
User u2 = repo.findById(1L).get();
System.out.println(u1 == u2);   // true (same instance)
```

### Dirty Checking

Managed entities are tracked. On flush, Hibernate diffs the state and issues `UPDATE` for changes.

```java
@Transactional
public void update(Long id) {
    User u = repo.findById(id).orElseThrow();
    u.setName("New Name");
    // No save() needed — dirty checking triggers UPDATE on commit
}
```

### Flush Modes

| Mode | Behavior |
|------|----------|
| `AUTO` (default) | Flush before query and at commit |
| `COMMIT` | Flush only at commit |
| `MANUAL` | Explicit `flush()` required |
| `ALWAYS` | Flush before every query |

```properties
spring.jpa.properties.org.hibernate.flushMode=COMMIT
```

### When Hibernate Flushes

1. Before a query that could be affected by pending changes
2. On `em.flush()`
3. On transaction commit

### Force Flush

```java
repo.save(user);
repo.flush();   // force INSERT/UPDATE now
```

Useful before a native query or when you need DB-generated values.

### Clearing the Context

```java
repo.save(user);
em.clear();     // detach everything → forced reload
```

For batch inserts, flush + clear prevents OutOfMemoryError:

```java
for (int i = 0; i < 10000; i++) {
    repo.save(new Entity(...));
    if (i % 100 == 0) {
        em.flush();
        em.clear();
    }
}
```

---

## 9. Repository Interfaces

### Hierarchy

```
Repository<T, ID>
    │
    ├── CrudRepository<T, ID>
    │       │
    │       ├── PagingAndSortingRepository<T, ID>
    │       │       │
    │       │       └── JpaRepository<T, ID>
    │       │
    │       └── ListCrudRepository (Boot 3+, returns List)
    │
    └── ...
```

### Core Methods

| Interface | Methods |
|-----------|---------|
| `CrudRepository` | save, saveAll, findById, existsById, findAll, count, deleteById, delete, deleteAll |
| `ListCrudRepository` | Same but returns `List<T>` |
| `PagingAndSortingRepository` | findAll(Sort), findAll(Pageable) |
| `JpaRepository` | All above + flush, saveAndFlush, deleteInBatch, getReferenceById |

### JpaRepository Extras

```java
List<T> findAll();
List<T> findAll(Sort sort);
List<T> findAllById(Iterable<ID> ids);
<S extends T> List<S> saveAll(Iterable<S> entities);
void flush();
<S extends T> S saveAndFlush(S entity);
void deleteAllInBatch(Iterable<T> entities);
void deleteAllByIdInBatch(Iterable<ID> ids);
T getReferenceById(ID id);
```

### Custom Repository

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Derived query methods
    List<User> findByEmailEndingWith(String domain);
    
    // @Query
    @Query("SELECT u FROM User u WHERE u.active = true")
    List<User> findActive();
    
    // Modify
    @Modifying
    @Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :date")
    int deactivateInactive(@Param("date") LocalDateTime date);
}
```

### Repository Fragments

Extract common methods across repositories:

```java
public interface CustomRepository<T, ID> {
    void refresh(T entity);
}

public class CustomRepositoryImpl<T, ID> implements CustomRepository<T, ID> {
    @PersistenceContext private EntityManager em;
    @Override
    public void refresh(T entity) { em.refresh(entity); }
}

public interface UserRepository extends JpaRepository<User, Long>, CustomRepository<User, Long> { }
```

Spring finds `CustomRepositoryImpl` (naming convention).

### @NoRepositoryBean

For base interfaces that shouldn't be instantiated:

```java
@NoRepositoryBean
public interface BaseRepository<T, ID> extends JpaRepository<T, ID> {
    List<T> findAllActive();
}
```

### Enabling Repositories

Spring Boot auto-detects. For manual config:

```java
@EnableJpaRepositories(basePackages = "com.example.repo")
```

### @RepositoryDefinition

For minimal interfaces:

```java
@RepositoryDefinition(domainClass = User.class, idClass = Long.class)
public interface UserRepo {
    User findById(Long id);
}
```

---

## 10. Derived Query Methods

Spring Data parses method names into queries.

### Structure

```
find / read / get / query / stream → By → Property → Conditions
```

### Keywords

| Keyword | Example | JPQL |
|---------|---------|------|
| `findBy` | `findByName(String)` | `WHERE name = ?` |
| `And` | `findByNameAndEmail` | `WHERE name = ? AND email = ?` |
| `Or` | `findByNameOrEmail` | `WHERE name = ? OR email = ?` |
| `Is`, `Equals` | `findByName` | `=` |
| `Not` | `findByNameNot` | `<>` |
| `StartingWith` | `findByNameStartingWith` | `LIKE 'x%'` |
| `EndingWith` | `findByNameEndingWith` | `LIKE '%x'` |
| `Containing` | `findByNameContaining` | `LIKE '%x%'` |
| `Like` | `findByNameLike` | `LIKE ?` |
| `LessThan` | `findByAgeLessThan` | `<` |
| `LessThanEqual` | `findByAgeLessThanEqual` | `<=` |
| `GreaterThan` | `findByAgeGreaterThan` | `>` |
| `Between` | `findByAgeBetween` | `BETWEEN ? AND ?` |
| `In` | `findByIdIn(List)` | `IN (...)` |
| `NotIn` | `findByIdNotIn` | `NOT IN` |
| `IsNull` | `findByEmailIsNull` | `IS NULL` |
| `IsNotNull` | `findByEmailIsNotNull` | `IS NOT NULL` |
| `True` / `False` | `findByActiveTrue` | `= true` |
| `IgnoreCase` | `findByNameIgnoreCase` | `UPPER = UPPER` |
| `OrderBy` | `findByNameOrderByAgeAsc` | `ORDER BY age ASC` |
| `First` / `Top` | `findFirst5ByName` | `LIMIT 5` |
| `Distinct` | `findDistinctByName` | `DISTINCT` |

### Examples

```java
List<User> findByName(String name);
Optional<User> findByEmail(String email);
List<User> findByNameAndEmail(String name, String email);
List<User> findByNameOrEmail(String name, String email);
List<User> findByAgeBetween(int min, int max);
List<User> findByAgeGreaterThanAndActiveTrue(int age);
List<User> findByNameContainingIgnoreCase(String part);
List<User> findTop10ByOrderByCreatedAtDesc();
List<User> findByStatusIn(Collection<Status> statuses);
long countByActiveTrue();
boolean existsByEmail(String email);
void deleteByEmail(String email);
```

### Nested Properties

```java
List<Order> findByCustomer_Email(String email);
// or with underscore recommended for clarity
List<Order> findByCustomerEmail(String email);   // works too
```

### @Query with Sort/Pageable

```java
Page<User> findByActiveTrue(Pageable pageable);
List<User> findByActiveTrue(Sort sort);
```

### Projection in Derived Queries

```java
List<UserNameOnly> findByActiveTrue();       // interface projection
```

### @Modifying for Bulk

```java
@Modifying
@Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :date")
int deactivateInactive(@Param("date") LocalDateTime date);
```

Requires `@Transactional` at service level; clears persistence context (`clearAutomatically = true`).

### Limitations

- Complex joins → use `@Query`
- Method name parser is limited
- Camel-case property matching can be ambiguous

---

## 11. @Query (JPQL)

**JPQL** — Java Persistence Query Language. Object-oriented SQL. Uses entity names and fields, not table/column names.

### Basic

```java
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);
```

### Positional Parameters

```java
@Query("SELECT u FROM User u WHERE u.email = ?1")
Optional<User> findByEmail(String email);
```

Prefer named (`:email`) over positional (`?1`).

### LIKE

```java
@Query("SELECT u FROM User u WHERE u.name LIKE %:keyword%")
List<User> search(@Param("keyword") String keyword);
```

### JOIN

```java
@Query("SELECT o FROM Order o JOIN o.customer c WHERE c.email = :email")
List<Order> findByCustomerEmail(@Param("email") String email);
```

### JOIN FETCH

```java
@Query("SELECT DISTINCT c FROM Customer c JOIN FETCH c.orders WHERE c.id = :id")
Optional<Customer> findWithOrders(@Param("id") Long id);
```

`DISTINCT` avoids duplicates when parent has many children.

### LEFT JOIN FETCH

```java
@Query("SELECT c FROM Customer c LEFT JOIN FETCH c.orders")
List<Customer> findAllWithOrders();
```

### Aggregation

```java
@Query("SELECT COUNT(u) FROM User u WHERE u.active = true")
long countActive();

@Query("SELECT AVG(o.total) FROM Order o WHERE o.customer.id = :id")
Double averageOrderTotal(@Param("id") Long id);

@Query("SELECT u.status, COUNT(u) FROM User u GROUP BY u.status")
List<Object[]> countByStatus();
```

### DTO Projection

```java
@Query("SELECT new com.example.dto.UserSummary(u.id, u.name, COUNT(o)) " +
       "FROM User u LEFT JOIN u.orders o GROUP BY u.id, u.name")
List<UserSummary> userSummaries();
```

### Update/Delete

```java
@Modifying
@Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :date")
int deactivateInactive(@Param("date") LocalDateTime date);

@Modifying
@Query("DELETE FROM User u WHERE u.active = false")
int deleteInactive();
```

### @Modifying Options

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
```

- `clearAutomatically` — clears persistence context after execute (prevents stale entities)
- `flushAutomatically` — flushes before executing (avoids lost updates)

### Native Query Flag

```java
@Query(value = "SELECT * FROM users WHERE email = :email", nativeQuery = true)
```

### Named Queries

Declared on entity:

```java
@Entity
@NamedQuery(name = "User.findActive", query = "SELECT u FROM User u WHERE u.active = true")
@NamedQueries({
    @NamedQuery(name = "User.findByEmail", query = "..."),
    @NamedQuery(name = "User.countActive", query = "...")
})
public class User { ... }
```

Referenced:

```java
@Query(name = "User.findActive")   // or method name matches
List<User> findActive();
```

Modern: prefer `@Query` for locality.

### SpEL in @Query (Spring Data Extensions)

```java
@Query("SELECT u FROM #{#entityName} u WHERE u.status = :status")
List<User> findByStatus(@Param("status") String status);
```

Useful in generic repositories.

---

## 12. @Query (Native SQL)

Write actual SQL, use table/column names.

```java
@Query(value = "SELECT * FROM users WHERE email = :email", nativeQuery = true)
Optional<User> findByEmailNative(@Param("email") String email);
```

### Native with Projection

```java
@Query(value = """
    SELECT u.id AS id, u.full_name AS name, COUNT(o.id) AS orderCount
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    GROUP BY u.id, u.full_name
    """, nativeQuery = true)
List<UserSummaryProjection> userSummaries();
```

### DTO with Native

```java
@Query(value = "SELECT new com.example.UserDto(u.id, u.name) FROM User u")
```

**Problem:** `new` doesn't work with `nativeQuery=true`. Use interface projection or `@SqlResultSetMapping`.

### Positional Parameters

Native queries use `?1` or `:name`. Prefer named.

### When to Use Native

- DB-specific features (window functions, CTEs, hints)
- Complex reports
- Performance-tuned queries
- Procedures

### Downsides

- Not portable across DBs
- Bypasses Hibernate's type conversions
- No entity graph support
- Returns `Object[]` unless projected

---

## 13. Pagination & Sorting

### Pageable

```java
Page<User> findByStatus(Status status, Pageable pageable);
```

Call:

```java
Pageable pageable = PageRequest.of(0, 20, Sort.by("name").ascending());
Page<User> page = repo.findByStatus(Status.ACTIVE, pageable);

page.getContent();          // List<User>
page.getTotalElements();    // long
page.getTotalPages();       // int
page.getNumber();           // current page (0-based)
page.getSize();             // page size
page.hasNext();             // bool
page.hasPrevious();         // bool
```

### Sort

```java
Sort sort = Sort.by("name").ascending().and(Sort.by("age").descending());
List<User> users = repo.findAll(sort);

// Multiple orders
Sort sort = Sort.by(
    Sort.Order.asc("name"),
    Sort.Order.desc("age")
);
```

### Slice

For "next page" navigation without total count (faster):

```java
Slice<User> findByStatus(Status status, Pageable pageable);
```

### Slice vs Page

| Feature | Slice | Page |
|---------|-------|------|
| Content | ✅ | ✅ |
| Has next | ✅ | ✅ |
| Total count | ❌ | ✅ |
| Count query | None | Extra `SELECT COUNT(*)` |

**Use `Slice` for infinite scroll; `Page` for numbered pagination.**

### Web Integration (Pageable in Controller)

```java
@GetMapping("/users")
public Page<User> list(Pageable pageable) {
    return userService.findAll(pageable);
}
```

Query: `/users?page=0&size=20&sort=name,asc`

Configure defaults:

```properties
spring.data.web.pageable.default-page-size=20
spring.data.web.pageable.max-page-size=2000
spring.data.web.pageable.one-indexed-parameters=false
```

`PageableHandlerMethodArgumentResolver` handles it automatically.

### Custom Sorting Field

```java
@GetMapping("/users")
public List<User> list(@PageableDefault(size = 20, sort = "createdAt") Pageable pageable) {
    return repo.findAll(pageable).getContent();
}
```

### @Query with Pageable

```java
@Query("SELECT u FROM User u WHERE u.active = true")
Page<User> findActive(Pageable pageable);

@Query(value = "SELECT * FROM users WHERE active = true",
       countQuery = "SELECT COUNT(*) FROM users WHERE active = true",
       nativeQuery = true)
Page<User> findActiveNative(Pageable pageable);
```

Provide a separate `countQuery` for optimized count.

---

## 14. Projections

Return only needed columns (avoids loading full entities).

### Interface-Based

```java
public interface UserNameOnly {
    Long getId();
    String getName();
}

public interface UserRepository extends JpaRepository<User, Long> {
    List<UserNameOnly> findByActiveTrue();
}
```

Spring generates proxy implementing the interface.

### Nested Projections

```java
public interface UserWithAddress {
    Long getId();
    String getName();
    AddressProjection getAddress();       // nested
}

public interface AddressProjection {
    String getCity();
}
```

Use `@Value` for computed fields:

```java
public interface UserSummary {
    Long getId();
    String getName();
    
    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
}
```

### Class-Based (DTO)

```java
public class UserDto {
    private final Long id;
    private final String name;
    
    public UserDto(Long id, String name) {
        this.id = id;
        this.name = name;
    }
    // getters
}

@Query("SELECT new com.example.UserDto(u.id, u.name) FROM User u")
List<UserDto> findUserDtos();
```

**Faster than interface projection** (no proxy) but requires constructor.

### Dynamic Projections

```java
<T> List<T> findByActiveTrue(Class<T> type);
```

Call:

```java
List<UserNameOnly> names = repo.findByActiveTrue(UserNameOnly.class);
List<UserDto> dtos = repo.findByActiveTrue(UserDto.class);
```

### When to Use Projections

- Read-only views
- Avoiding N+1 (only select needed fields)
- DTOs for controllers
- Large entity has many columns

### Cost

- Interface projections: dynamic proxy → slight overhead
- Class-based: requires constructor → works with records

### Records (Boot 3 / Java 16+)

```java
public record UserDto(Long id, String name) { }

@Query("SELECT new com.example.UserDto(u.id, u.name) FROM User u")
List<UserDto> findDtos();
```

Clean, immutable, no boilerplate.

---

## 15. Specifications & Criteria API

**Specifications** — type-safe, dynamic, composable query predicates. Based on JPA Criteria API.

### Enabling

```java
public interface UserRepository extends JpaRepository<User, Long>, 
                                         JpaSpecificationExecutor<User> { }
```

### Basic Specification

```java
public class UserSpecs {
    public static Specification<User> hasName(String name) {
        return (root, query, cb) -> cb.equal(root.get("name"), name);
    }
    
    public static Specification<User> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }
    
    public static Specification<User> ageGreaterThan(int age) {
        return (root, query, cb) -> cb.greaterThan(root.get("age"), age);
    }
}
```

### Combining

```java
Specification<User> spec = Specification
    .where(hasName("Alice"))
    .and(isActive())
    .and(ageGreaterThan(18));

List<User> users = repo.findAll(spec);
```

### Null-Safe Composition

```java
public static Specification<User> fromFilter(UserFilter f) {
    Specification<User> spec = Specification.where(null);
    if (f.getName() != null) spec = spec.and(hasName(f.getName()));
    if (f.getActive() != null) spec = spec.and(isActive());
    if (f.getMinAge() != null) spec = spec.and(ageGreaterThan(f.getMinAge()));
    return spec;
}
```

### Joins

```java
public static Specification<Order> hasCustomerEmail(String email) {
    return (root, query, cb) -> {
        Join<Order, Customer> customer = root.join("customer");
        return cb.equal(customer.get("email"), email);
    };
}
```

Avoid duplicate joins with `root.fetch` or `@EntityGraph`.

### Paging with Specs

```java
Page<User> page = repo.findAll(spec, PageRequest.of(0, 20, Sort.by("name")));
```

### Or Conditions

```java
Specification<User> spec = Specification
    .where(hasName("Alice"))
    .or(hasName("Bob"));
```

### Not

```java
Specification<User> notActive = Specification.not(isActive());
```

### Subqueries

```java
public static Specification<Customer> hasOrders() {
    return (root, query, cb) -> {
        Subquery<Long> sub = query.subquery(Long.class);
        Root<Order> orderRoot = sub.from(Order.class);
        sub.select(cb.count(orderRoot))
           .where(cb.equal(orderRoot.get("customer"), root));
        return cb.greaterThan(sub, 0L);
    };
}
```

### CriteriaBuilder Methods

| Method | SQL |
|--------|-----|
| `equal` | `=` |
| `notEqual` | `<>` |
| `greaterThan` | `>` |
| `lessThan` | `<` |
| `between` | `BETWEEN` |
| `like` | `LIKE` |
| `in` | `IN` |
| `isNull` / `isNotNull` | `IS NULL` |
| `and` / `or` / `not` | logical |
| `count`, `sum`, `avg`, `max`, `min` | aggregates |
| `asc`, `desc` | ordering |

### When to Use Specifications

- Dynamic filters (search APIs)
- Composable query parts
- When criteria vary per request

### Alternatives

- QueryDSL (better than Criteria API)
- JPA Criteria API directly
- Native SQL for complex reports

---

## 16. Entity Graphs

Define fetch plans **outside** the query, overriding defaults.

### @NamedEntityGraph on Entity

```java
@Entity
@NamedEntityGraph(
    name = "User.withOrders",
    attributeNodes = @NamedAttributeNode("orders")
)
@NamedEntityGraph(
    name = "User.withOrdersAndProfile",
    attributeNodes = {
        @NamedAttributeNode("orders"),
        @NamedAttributeNode(value = "profile", subgraph = "profileDetails")
    },
    subgraphs = @NamedSubgraph(
        name = "profileDetails",
        attributeNodes = @NamedAttributeNode("address")
    )
)
public class User {
    @Id @GeneratedValue private Long id;
    
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
    
    @OneToOne
    private Profile profile;
}
```

### Repository Usage

```java
@EntityGraph(value = "User.withOrders", type = EntityGraphType.FETCH)
Optional<User> findById(Long id);

@EntityGraph(attributePaths = {"orders", "profile"})
List<User> findAll();
```

### @EntityGraph Types

| Type | Behavior |
|------|----------|
| `FETCH` | Eager load listed attributes |
| `LOAD` | Load listed attributes as eager; don't override others |

### Ad-hoc Graph

```java
EntityGraph<User> graph = em.createEntityGraph(User.class);
graph.addAttributeNodes("orders");
Map<String, Object> hints = Map.of("jakarta.persistence.fetchgraph", graph);
em.find(User.class, 1L, hints);
```

### When to Use

- Repository methods need eager fetch for specific attributes
- Avoid N+1 for read-heavy endpoints
- No annotation pollution on queries

### @EntityGraph with Pagination

Pagination + `JOIN FETCH` collection → **HHH000104: firstResult/maxResults specified with collection fetch; applying in memory**. `@EntityGraph` has the same issue.

**Fix:** Use pagination on entity IDs, then fetch children separately. Or use `Slice`/two queries.

---

## 17. Auditing

Automatically populate `createdAt`, `createdBy`, `lastModifiedAt`, `lastModifiedBy`.

### Enable

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .map(Authentication::getName);
    }
}
```

### Base Entity

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {
    @CreatedDate
    @Column(updatable = false)
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant lastModifiedAt;
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String lastModifiedBy;
    // getters
}
```

### Entity Inherits

```java
@Entity
public class User extends Auditable {
    @Id @GeneratedValue private Long id;
    private String name;
}
```

### Audit Annotations

| Annotation | Filled By |
|-----------|-----------|
| `@CreatedDate` | `DateTimeProvider` |
| `@LastModifiedDate` | `DateTimeProvider` |
| `@CreatedBy` | `AuditorAware` |
| `@LastModifiedBy` | `AuditorAware` |

### Custom DateTimeProvider

```java
@Bean
public DateTimeProvider dateTimeProvider() {
    return () -> Optional.of(Instant.now());
}
```

### Hibernate Envers

For **full audit history** (every change to every field), use Hibernate Envers:

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-envers</artifactId>
</dependency>
```

```java
@Entity
@Audited
public class User { ... }
```

Query history via `AuditReader`. Creates `users_AUD` tables.

---

## 18. Locking

Prevent concurrent modifications.

### Optimistic Locking (@Version)

Assumes no conflict; checks at update.

```java
@Entity
public class Product {
    @Id @GeneratedValue private Long id;
    
    @Version
    private Long version;
    
    private BigDecimal price;
}
```

**How it works:**
- `UPDATE products SET price = ?, version = version + 1 WHERE id = ? AND version = ?`
- If rows affected = 0 → `OptimisticLockingFailureException`

### Handling Optimistic Failures

```java
@Retryable(
    retryFor = OptimisticLockingFailureException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 100)
)
@Transactional
public void updatePrice(Long id, BigDecimal newPrice) {
    Product p = repo.findById(id).orElseThrow();
    p.setPrice(newPrice);
}
```

Requires `spring-retry`.

### Pessimistic Locking

Locks row at read time.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Product findForUpdate(@Param("id") Long id);
```

Or with `EntityManager`:

```java
em.find(Product.class, id, LockModeType.PESSIMISTIC_WRITE);
```

### Lock Modes

| Mode | Effect |
|------|--------|
| `PESSIMISTIC_READ` | Shared lock (SELECT FOR SHARE) |
| `PESSIMISTIC_WRITE` | Exclusive lock (SELECT FOR UPDATE) |
| `PESSIMISTIC_FORCE_INCREMENT` | Write + version increment |
| `OPTIMISTIC` | Read with version check at commit |
| `OPTIMISTIC_FORCE_INCREMENT` | Read + version increment at commit |
| `NONE` | No lock |

### Timeouts

```java
Map<String, Object> hints = Map.of(
    "jakarta.persistence.lock.timeout", 5000);
em.find(Product.class, id, LockModeType.PESSIMISTIC_WRITE, hints);
```

### Optimistic vs Pessimistic

| Aspect | Optimistic | Pessimistic |
|--------|-----------|-------------|
| Lock timing | At commit | At read |
| Blocking | No | Yes |
| Deadlock risk | Low | Higher |
| Scalability | Better | Worse |
| Use case | Rare conflicts | Frequent conflicts |

**Rule:** Prefer optimistic unless conflicts are frequent.

---

## 19. Caching (L1, L2, Query)

### First-Level Cache (Persistence Context)

- Always on
- Per transaction
- Same ID → same object
- Cleared on `em.clear()` or transaction end

### Second-Level Cache (Shared)

Shared across transactions for a session factory.

### Enable L2 Cache

```properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory
spring.jpa.properties.jakarta.persistence.sharedCache.mode=ENABLE_SELECTIVE
```

### Add Cache Provider

```xml
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

### Enable Caching on Entities

```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class User { ... }
```

### Cache Concurrency Strategies

| Strategy | Use Case |
|----------|----------|
| `READ_ONLY` | Immutable data |
| `READ_WRITE` | Frequent reads, occasional writes |
| `NONSTRICT_READ_WRITE` | Rare writes, tolerable staleness |
| `TRANSACTIONAL` | Full XA transaction support |

### Query Cache

Caches query results (IDs).

```properties
spring.jpa.properties.hibernate.cache.use_query_cache=true
```

```java
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<User> findByStatus(Status status);
```

Or per query:

```java
em.createQuery("...").setHint("org.hibernate.cacheable", true).getResultList();
```

### Invalidation

Automatic on entity changes. For query cache, must invalidate on any write to the affected tables.

### When to Use

- Read-heavy, rarely-modified data (reference tables)
- Expensive queries with stable inputs
- Multi-node clusters sharing cache (Redis-backed)

### When NOT to Use

- Frequently-updated data
- Small data (first-level cache enough)
- Consistency-critical

### Spring Cache vs Hibernate L2

Spring's `@Cacheable` caches **method return values**. Hibernate L2 caches **entities and queries**. They're complementary — use Spring Cache for service-level caching, Hibernate L2 for entity caching.

---

## 20. Interview Questions

### Q1. What is the difference between JPA, Hibernate, and Spring Data JPA?

**Answer:** JPA is the specification (API). Hibernate is an implementation. Spring Data JPA is a repository abstraction that simplifies JPA usage. Spring Data JPA uses Hibernate (or another provider) under the hood.

### Q2. What is the persistence context?

**Answer:** A first-level cache in Hibernate where managed entities live. Guarantees that the same row returns the same object instance within a transaction. Tracks changes for dirty checking.

### Q3. What is dirty checking?

**Answer:** Hibernate automatically detects changes to managed entities and issues `UPDATE` on flush. You don't need to call `save()` for updates to managed entities — just modify fields.

### Q4. Explain entity lifecycle states.

**Answer:** Transient (new, no ID, not managed), Managed (in persistence context, tracked), Detached (was managed, no longer), Removed (scheduled for delete).

### Q5. Difference between `save()` and `saveAndFlush()`?

**Answer:** `save()` persists/merges but may defer the actual SQL until flush. `saveAndFlush()` forces immediate flush so DB constraints are checked immediately.

### Q6. What is the N+1 problem?

**Answer:** Loading N parents triggers N additional queries for their lazy children → N+1 total. Fix with `JOIN FETCH`, `@EntityGraph`, `@BatchSize`, or projections.

### Q7. Explain LAZY vs EAGER fetch.

**Answer:** LAZY loads on first access (via proxy). EAGER loads immediately with parent. Defaults: `@ManyToOne`/`@OneToOne` → EAGER; `@OneToMany`/`@ManyToMany` → LAZY. Prefer LAZY to avoid unnecessary data loading.

### Q8. What is `LazyInitializationException`?

**Answer:** Thrown when a lazy association is accessed outside the persistence context (after transaction ends). Fix by fetching eagerly within the transaction, using `@EntityGraph`, or keeping session open (`open-in-view=true`, not recommended).

### Q9. What is Open Session in View (OSIV)?

**Answer:** A Spring Boot default that keeps the Hibernate session open during view rendering, allowing lazy loading in controllers/views. Downside: holds DB connections longer, hides N+1. Best practice: `spring.jpa.open-in-view=false`.

### Q10. Difference between `CrudRepository`, `PagingAndSortingRepository`, and `JpaRepository`?

**Answer:**
- `CrudRepository`: save, findById, delete, etc.
- `PagingAndSortingRepository`: adds pagination and sorting
- `JpaRepository`: adds JPA-specific (flush, saveAndFlush, deleteInBatch, getReferenceById) and extends the above

### Q11. How does Spring Data derive queries?

**Answer:** By parsing method names. `findByEmailAndStatus(String, Status)` → `SELECT u FROM User u WHERE u.email = ? AND u.status = ?`. Keywords like `And`, `Or`, `OrderBy`, `Containing`, `Between` map to JPQL.

### Q12. When to use @Query?

**Answer:** For complex queries, JPQL features (JOIN FETCH, subqueries, DTO projection), or when method name would be too long. Use JPQL for portability; native SQL for DB-specific features.

### Q13. What is `@Modifying`?

**Answer:** Marks a `@Query` as UPDATE/DELETE (not SELECT). Required for bulk operations. Combine with `@Transactional` at service layer and `clearAutomatically = true` to avoid stale entities.

### Q14. Explain `Page`, `Slice`, `Pageable`, `Sort`.

**Answer:** `Pageable` — request for a page (number, size, sort). `Sort` — order specification. `Page` — result with content + total count (issues count query). `Slice` — result with content + has-next (no count, faster).

### Q15. What are projections?

**Answer:** Ways to return partial entities. **Interface-based** (dynamic proxy) and **class-based** (constructor with `new`). Useful for read-only views, avoiding full entity loads and N+1.

### Q16. What are Specifications?

**Answer:** Type-safe, composable query predicates based on JPA Criteria API. Enable dynamic filters: `Specification.where(hasName("A")).and(isActive())`. Repository must extend `JpaSpecificationExecutor`.

### Q17. What is an Entity Graph?

**Answer:** A fetch plan defined outside queries (`@NamedEntityGraph` or `@EntityGraph(attributePaths=...)`) that overrides default fetch types for a specific repository method. Use it to fetch associations eagerly without `JOIN FETCH`.

### Q18. What is optimistic locking?

**Answer:** A `@Version` field that Hibernate checks on UPDATE. If the version changed, `OptimisticLockingFailureException` is thrown. No DB locks; good for rare conflicts. Retry the operation.

### Q19. What is pessimistic locking?

**Answer:** Acquires a DB lock at read time (`SELECT FOR UPDATE`). Blocks other writers until the transaction ends. Use for frequent conflicts or critical sections. Risk of deadlocks and reduced concurrency.

### Q20. What is the difference between first-level and second-level cache?

**Answer:** First-level is per-transaction (persistence context) — always on. Second-level is shared across transactions for a session factory — opt-in per entity, needs a provider (Ehcache, Redis). L2 reduces DB hits for reference data.

### Q21. How does auditing work?

**Answer:** Annotate a base class with `@EntityListeners(AuditingEntityListener.class)` and fields with `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy`. Enable via `@EnableJpaAuditing`. Provide `AuditorAware` for current user.

### Q22. How to avoid N+1 in pagination?

**Answer:** Never `JOIN FETCH` a collection with pagination — Hibernate warns `HHH000104` and paginates in memory. Instead:
1. Fetch IDs of the page, then fetch children in a second query
2. Use `@EntityGraph` with `@BatchSize`
3. Use `Slice` with `@Fetch(SUBSELECT)`

### Q23. What is `@MappedSuperclass`?

**Answer:** A class whose fields are inherited by child entities, but has no table of its own. Used for common fields (id, timestamps, audit). Not an entity.

### Q24. What is `@Embeddable` vs `@Embedded`?

**Answer:** `@Embeddable` marks a value object whose fields are stored in the parent's table. `@Embedded` uses it inside an entity. Reusable across entities.

### Q25. Difference between `getById` and `findById`?

**Answer:** `findById` returns `Optional<T>` and issues a SELECT immediately. `getById` (formerly `getOne`) returns a **lazy proxy** without hitting the DB until a field is accessed. Throws `EntityNotFoundException` on access if missing.

---

## 21. Cheat Sheet

### Entity Annotations

```java
@Entity
@Table(name = "users", indexes = @Index(columnList = "email"))
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
    @SequenceGenerator(name = "user_seq", sequenceName = "users_seq", allocationSize = 50)
    private Long id;
    
    @Column(nullable = false, length = 100)
    private String name;
    
    @Enumerated(EnumType.STRING)
    private Status status;
    
    @Version
    private Long version;
    
    @CreatedDate @Column(updatable = false)
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant updatedAt;
}
```

### Relationship Annotations

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "customer_id")
private Customer customer;

@OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Order> orders = new ArrayList<>();

@OneToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "profile_id")
private Profile profile;

@ManyToMany
@JoinTable(name = "user_roles",
    joinColumns = @JoinColumn(name = "user_id"),
    inverseJoinColumns = @JoinColumn(name = "role_id"))
private Set<Role> roles = new HashSet<>();
```

### Repository

```java
public interface UserRepository extends JpaRepository<User, Long>,
                                          JpaSpecificationExecutor<User> {
    
    Optional<User> findByEmail(String email);
    
    List<User> findByStatusAndCreatedAtAfter(Status s, Instant t);
    
    @Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
    Optional<User> findWithOrders(@Param("id") Long id);
    
    @Modifying(clearAutomatically = true)
    @Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :date")
    int deactivateInactive(@Param("date") LocalDateTime date);
    
    Page<User> findByStatus(Status status, Pageable pageable);
    
    @EntityGraph(attributePaths = {"orders", "profile"})
    List<User> findByStatus(Status status);
    
    <T> List<T> findByStatus(Status status, Class<T> type);   // projection
}
```

### Derived Query Keywords

```
findBy...And, Or, Is, Not, StartingWith, EndingWith,
Containing, Like, LessThan, GreaterThan, Between, In,
NotNull, True/False, IgnoreCase, OrderBy, First/Top, Distinct
```

### Pageable

```java
Pageable pageable = PageRequest.of(0, 20, Sort.by("name").ascending());
Page<User> page = repo.findByStatus(Status.ACTIVE, pageable);
```

### Specifications

```java
Specification<User> spec = Specification.where(hasName("Alice"))
    .and(isActive())
    .and(ageGreaterThan(18));

Page<User> page = repo.findAll(spec, pageable);
```

### Locking

```java
@Version
private Long version;   // optimistic

@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Product findForUpdate(@Param("id") Long id);
```

### Properties

```properties
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.open-in-view=false
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.use_query_cache=true
spring.jpa.properties.hibernate.jdbc.time_zone=UTC
```

### Fetch Types Cheat

```
@ManyToOne    → default EAGER, use LAZY
@OneToOne     → default EAGER, use LAZY
@OneToMany    → default LAZY
@ManyToMany   → default LAZY
```

### N+1 Fixes

```
1. JOIN FETCH
2. @EntityGraph(attributePaths = {...})
3. @BatchSize(size = N)
4. @Fetch(FetchMode.SUBSELECT)
5. DTO projection
```

### When Things Go Wrong

| Symptom | Cause | Fix |
|---------|-------|-----|
| `LazyInitializationException` | Lazy outside session | Fetch in service; `@EntityGraph` |
| `HHH000104` | Fetch join + pagination | Two-query approach |
| `OptimisticLockingFailureException` | Concurrent update | Retry |
| `MultipleBagFetchException` | Two `List` fetch joins | Use `Set` or `@OrderColumn` |
| `StackOverflowError` on toString | Bidirectional `toString` | Exclude from `toString` |

### Cross-References

- **Previous:** `09_Spring_Boot_Actuator.md`
- **Next:** `11_Spring_Data_JDBC.md`
- **Related:** `04_Spring_JDBC.md`, `05_Spring_Transaction_Management.md`, `12_Spring_Caching.md`
- **Interview:** `24_Spring_Interview_Questions.md` (§Data JPA — 25 questions)

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [09_Spring_Boot_Actuator.md](./09_Spring_Boot_Actuator.md)
- **Next →:** [11_Spring_Data_JDBC.md](./11_Spring_Data_JDBC.md)
- **Related:** [04_Spring_JDBC.md](./04_Spring_JDBC.md), [11_Spring_Data_JDBC.md](./11_Spring_Data_JDBC.md), [12_Spring_Caching.md](./12_Spring_Caching.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
