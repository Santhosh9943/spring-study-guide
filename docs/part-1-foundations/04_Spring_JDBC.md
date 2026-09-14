# Spring JDBC & JdbcTemplate

> **File:** `04_Spring_JDBC.md`
> **Part:** 1 — Foundations
> **Prerequisites:** `01_Spring_Framework_Core.md`, `05_Spring_Transaction_Management.md` (helpful)
> **Estimated Study Time:** 6–8 hours

---

## Table of Contents

1. [Problems with Raw JDBC](#1-problems-with-raw-jdbc)
2. [Spring JDBC Architecture](#2-spring-jdbc-architecture)
3. [DataSource Configuration](#3-datasource-configuration)
4. [JdbcTemplate — CRUD](#4-jdbctemplate--crud)
5. [RowMapper & ResultSetExtractor](#5-rowmapper--resultsetextractor)
6. [Batch Operations](#6-batch-operations)
7. [NamedParameterJdbcTemplate](#7-namedparameterjdbctemplate)
8. [SimpleJdbcInsert & SimpleJdbcCall](#8-simplejdbcinsert--simplejdbccall)
9. [Exception Translation](#9-exception-translation)
10. [Transaction Integration](#10-transaction-integration)
11. [JDBC vs JPA](#11-jdbc-vs-jpa)
12. [Interview Questions](#12-interview-questions)
13. [Cheat Sheet](#13-cheat-sheet)

---

## 1. Problems with Raw JDBC

### The 7 Pain Points

```java
public User findById(Long id) {
    Connection conn = null;
    PreparedStatement ps = null;
    ResultSet rs = null;
    try {
        conn = dataSource.getConnection();                       // 1. Boilerplate
        ps = conn.prepareStatement(                              // 2. SQL strings
            "SELECT id, name, email FROM users WHERE id = ?");
        ps.setLong(1, id);                                       // 3. Positional params
        rs = ps.executeQuery();
        if (rs.next()) {                                         // 4. Manual mapping
            User u = new User();
            u.setId(rs.getLong("id"));
            u.setName(rs.getString("name"));
            u.setEmail(rs.getString("email"));
            return u;
        }
        return null;
    } catch (SQLException e) {                                   // 5. Verbose exceptions
        throw new RuntimeException(e);
    } finally {                                                  // 6. Resource cleanup
        if (rs != null) try { rs.close(); } catch (SQLException ignored) {}
        if (ps != null) try { ps.close(); } catch (SQLException ignored) {}
        if (conn != null) try { conn.close(); } catch (SQLException ignored) {}
    }                                                                // 7. Vendor exceptions
}
```

### Problems Summary

| # | Problem | Impact |
|---|---------|--------|
| 1 | Boilerplate getConnection/prepareStatement | Code bloat |
| 2 | SQL embedded in Java | Poor readability |
| 3 | Positional parameters | Fragile when order changes |
| 4 | Manual ResultSet → object mapping | Repetitive, error-prone |
| 5 | SQLException (checked) | Forces try/catch everywhere |
| 6 | Explicit resource cleanup | Leak risk |
| 7 | Vendor-specific error codes | Non-portable |

### What Spring JDBC Fixes

- **Template pattern** — boilerplate eliminated
- **RowMapper** — clean mapping
- **DataAccessException** — unchecked, portable
- **Automatic resource management** — no leaks
- **NamedParameterJdbcTemplate** — named params
- **Batch support** — first-class

---

## 2. Spring JDBC Architecture

### Module Stack

```
┌──────────────────────────────────────────────────┐
│              Application Code                    │
├──────────────────────────────────────────────────┤
│  JdbcTemplate / NamedParameterJdbcTemplate      │
│  SimpleJdbcInsert / SimpleJdbcCall / JdbcDaoSupport │
├──────────────────────────────────────────────────┤
│      Spring JDBC Core (org.springframework.jdbc) │
│      Exception translation, RowMapper, etc.      │
├──────────────────────────────────────────────────┤
│   DataSource Abstraction                         │
│   (HikariCP, Tomcat, DBCP, ...)                  │
├──────────────────────────────────────────────────┤
│         JDBC Driver (PostgreSQL, MySQL, ...)     │
├──────────────────────────────────────────────────┤
│              Database                            │
└──────────────────────────────────────────────────┘
```

### Key Classes

| Class | Purpose |
|-------|---------|
| `JdbcTemplate` | Main workhorse |
| `NamedParameterJdbcTemplate` | Named parameters |
| `SimpleJdbcInsert` | Inserts without SQL |
| `SimpleJdbcCall` | Stored procedures |
| `JdbcDaoSupport` | Base class for DAOs (legacy) |
| `DataSource` | Connection source |
| `RowMapper` | Row → object (1 per row) |
| `ResultSetExtractor` | ResultSet → object (whole set) |
| `RowCallbackHandler` | Row → side-effect (no return) |
| `PreparedStatementSetter` | Bind params |
| `GeneratedKeyHolder` | Retrieve auto-generated keys |

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

Includes HikariCP by default.

---

## 3. DataSource Configuration

### Spring Boot Auto-Configuration

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
spring.datasource.password=secret
spring.datasource.driver-class-name=org.postgresql.Driver
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.pool-name=MyHikariPool
```

### Manual Config

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @ConfigurationProperties("spring.datasource.hikari")
    public HikariDataSource dataSource() {
        return new HikariDataSource();
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource ds) {
        return new JdbcTemplate(ds);
    }
    
    @Bean
    public NamedParameterJdbcTemplate namedJdbc(DataSource ds) {
        return new NamedParameterJdbcTemplate(ds);
    }
}
```

### Connection Pool Comparison

| Pool | Notes |
|------|-------|
| **HikariCP** | Default in Spring Boot, fastest |
| Tomcat JDBC | Bundled with Tomcat, feature-rich |
| Apache DBCP2 | Legacy, widely known |
| C3P0 | Old, avoid |
| Vibur | Low-latency |

**Always use Hikari for new apps.**

### Multiple DataSources

```java
@Configuration
public class MultiDsConfig {
    
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDs() { return DataSourceBuilder.create().build(); }
    
    @Bean
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDs() { return DataSourceBuilder.create().build(); }
    
    @Bean @Primary
    public JdbcTemplate primaryJdbc(@Qualifier("primaryDs") DataSource ds) {
        return new JdbcTemplate(ds);
    }
    
    @Bean
    public JdbcTemplate secondaryJdbc(@Qualifier("secondaryDs") DataSource ds) {
        return new JdbcTemplate(ds);
    }
}
```

---

## 4. JdbcTemplate — CRUD

### Setup

```java
@Repository
public class UserRepository {
    private final JdbcTemplate jdbc;
    
    public UserRepository(JdbcTemplate jdbc) { this.jdbc = jdbc; }
}
```

### CREATE — Insert

```java
public int insert(User user) {
    return jdbc.update(
        "INSERT INTO users (name, email) VALUES (?, ?)",
        user.getName(), user.getEmail());
}
```

### Insert with Generated Key

```java
public Long insertAndReturnId(User user) {
    KeyHolder keyHolder = new GeneratedKeyHolder();
    jdbc.update(con -> {
        PreparedStatement ps = con.prepareStatement(
            "INSERT INTO users (name, email) VALUES (?, ?)",
            Statement.RETURN_GENERATED_KEYS);
        ps.setString(1, user.getName());
        ps.setString(2, user.getEmail());
        return ps;
    }, keyHolder);
    return keyHolder.getKey().longValue();
}
```

### READ — Query

```java
// Count
public int count() {
    return jdbc.queryForObject("SELECT COUNT(*) FROM users", Integer.class);
}

// Single value
public String findEmailById(Long id) {
    return jdbc.queryForObject(
        "SELECT email FROM users WHERE id = ?", String.class, id);
}

// Single row → object
public User findById(Long id) {
    return jdbc.queryForObject(
        "SELECT id, name, email FROM users WHERE id = ?",
        (rs, rowNum) -> new User(
            rs.getLong("id"),
            rs.getString("name"),
            rs.getString("email")),
        id);
}

// List
public List<User> findAll() {
    return jdbc.query(
        "SELECT id, name, email FROM users ORDER BY id",
        (rs, rowNum) -> new User(
            rs.getLong("id"),
            rs.getString("name"),
            rs.getString("email")));
}

// Map
public List<Map<String, Object>> findRaw() {
    return jdbc.queryForList("SELECT * FROM users");
}
```

### UPDATE

```java
public int update(User user) {
    return jdbc.update(
        "UPDATE users SET name = ?, email = ? WHERE id = ?",
        user.getName(), user.getEmail(), user.getId());
}
```

### DELETE

```java
public int delete(Long id) {
    return jdbc.update("DELETE FROM users WHERE id = ?", id);
}
```

### Handling No Result

`queryForObject` throws `EmptyResultDataAccessException` if no rows. Handle:

```java
public Optional<User> findById(Long id) {
    try {
        return Optional.ofNullable(jdbc.queryForObject(
            "SELECT id, name, email FROM users WHERE id = ?",
            userRowMapper, id));
    } catch (EmptyResultDataAccessException e) {
        return Optional.empty();
    }
}
```

### Available Methods

| Method | Purpose |
|--------|---------|
| `update(sql, args...)` | INSERT/UPDATE/DELETE |
| `queryForObject(sql, Class, args...)` | Single value |
| `queryForObject(sql, RowMapper, args...)` | Single row |
| `query(sql, RowMapper, args...)` | List of rows |
| `queryForList(sql, Class, args...)` | List of values |
| `queryForList(sql, args...)` | List of maps |
| `queryForMap(sql, args...)` | Single map |
| `execute(sql)` | Any SQL (DDL) |
| `batchUpdate(sql, BatchPreparedStatementSetter)` | Batch |
| `batchUpdate(sql, List<Object[]>)` | Batch with args |

### RowMapper as Bean or Class

```java
public class UserRowMapper implements RowMapper<User> {
    @Override
    public User mapRow(ResultSet rs, int rowNum) throws SQLException {
        return new User(
            rs.getLong("id"),
            rs.getString("name"),
            rs.getString("email"));
    }
}

// Usage
jdbc.query("SELECT * FROM users", new UserRowMapper());
```

---

## 5. RowMapper & ResultSetExtractor

### RowMapper<T>

Maps **one row** at a time. Spring iterates.

```java
public interface RowMapper<T> {
    T mapRow(ResultSet rs, int rowNum) throws SQLException;
}
```

**Pros:** Simple, memory-efficient for large results.

### ResultSetExtractor<T>

Receives the **entire** ResultSet. You iterate manually.

```java
public interface ResultSetExtractor<T> {
    T extractData(ResultSet rs) throws SQLException, DataAccessException;
}
```

**Use case:** Custom shapes (e.g., customer + orders in one query).

```java
public List<Customer> findCustomersWithOrders() {
    return jdbc.query(
        "SELECT c.id cid, c.name cname, o.id oid, o.total ototal " +
        "FROM customers c LEFT JOIN orders o ON c.id = o.customer_id " +
        "ORDER BY c.id",
        rs -> {
            Map<Long, Customer> map = new LinkedHashMap<>();
            while (rs.next()) {
                Long cid = rs.getLong("cid");
                Customer c = map.computeIfAbsent(cid, id -> {
                    try {
                        return new Customer(id, rs.getString("cname"));
                    } catch (SQLException e) { throw new RuntimeException(e); }
                });
                long oid = rs.getLong("oid");
                if (!rs.wasNull()) {
                    c.getOrders().add(new Order(oid, rs.getBigDecimal("ototal")));
                }
            }
            return new ArrayList<>(map.values());
        });
}
```

### RowCallbackHandler

For side effects (no return value).

```java
public void processAllUsers(Consumer<User> consumer) {
    jdbc.query("SELECT * FROM users",
        (RowCallbackHandler) rs -> consumer.accept(new User(
            rs.getLong("id"),
            rs.getString("name"),
            rs.getString("email"))));
}
```

### Comparison

| Interface | Scope | Returns | Use case |
|-----------|-------|---------|----------|
| `RowMapper` | 1 row | T | Normal row → object |
| `ResultSetExtractor` | Whole set | T | Complex object graph |
| `RowCallbackHandler` | 1 row | void | Streaming side effects |

### BeanPropertyRowMapper

Auto-maps columns to bean fields (by name).

```java
RowMapper<User> mapper = new BeanPropertyRowMapper<>(User.class);
List<User> users = jdbc.query("SELECT id, name, email FROM users", mapper);
```

**Rules:**
- Column `user_name` → field `userName` (via `underscoreToCamelCase`)
- Types auto-converted by Spring
- Works with setters or fields

**Caveat:** Slightly slower (reflection), but very convenient.

---

## 6. Batch Operations

### Batch Insert — List of Arrays

```java
public int[] batchInsert(List<User> users) {
    List<Object[]> args = users.stream()
        .map(u -> new Object[]{u.getName(), u.getEmail()})
        .toList();
    return jdbc.batchUpdate(
        "INSERT INTO users (name, email) VALUES (?, ?)", args);
}
```

### BatchPreparedStatementSetter

```java
public int[] batchInsertWithSetter(List<User> users) {
    return jdbc.batchUpdate(
        "INSERT INTO users (name, email) VALUES (?, ?)",
        new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                User u = users.get(i);
                ps.setString(1, u.getName());
                ps.setString(2, u.getEmail());
            }
            @Override
            public int getBatchSize() { return users.size(); }
        });
}
```

### ParameterizedBatchUpdate — Batch with Named Params

```java
namedJdbc.batchUpdate(
    "INSERT INTO users (name, email) VALUES (:name, :email)",
    new SqlParameterSource[]{
        new MapSqlParameterSource().addValue("name", "A").addValue("email", "a@x.com"),
        new MapSqlParameterSource().addValue("name", "B").addValue("email", "b@x.com")
    });
```

### BeanPropertySqlParameterSource

```java
SqlParameterSource[] sources = users.stream()
    .map(BeanPropertySqlParameterSource::new)
    .toArray(SqlParameterSource[]::new);
namedJdbc.batchUpdate("INSERT INTO users (name, email) VALUES (:name, :email)", sources);
```

### Batch Performance Tips

- Batch size **500–5000** typically optimal
- Use `rewriteBatchedStatements=true` for MySQL
- Batch `INSERT`, `UPDATE`, `DELETE` — not `SELECT`
- Use transactions to avoid per-statement commit

```java
@Transactional
public void insertMany(List<User> users) { batchInsert(users); }
```

---

## 7. NamedParameterJdbcTemplate

Cleaner than positional `?`.

### Setup

```java
@Repository
public class UserRepository {
    private final NamedParameterJdbcTemplate namedJdbc;
    public UserRepository(NamedParameterJdbcTemplate namedJdbc) {
        this.namedJdbc = namedJdbc;
    }
}
```

### Named Params

```java
public User findByEmail(String email) {
    MapSqlParameterSource params = new MapSqlParameterSource()
        .addValue("email", email);
    return namedJdbc.queryForObject(
        "SELECT * FROM users WHERE email = :email",
        params,
        userRowMapper);
}
```

### Map-based

```java
Map<String, Object> params = Map.of(
    "minAge", 18,
    "maxAge", 65
);
List<User> users = namedJdbc.query(
    "SELECT * FROM users WHERE age BETWEEN :minAge AND :maxAge",
    params,
    userRowMapper);
```

### IN Clause

```java
MapSqlParameterSource params = new MapSqlParameterSource()
    .addValue("ids", List.of(1L, 2L, 3L));
List<User> users = namedJdbc.query(
    "SELECT * FROM users WHERE id IN (:ids)",
    params,
    userRowMapper);
```

### Updates

```java
namedJdbc.update("UPDATE users SET name = :name WHERE id = :id",
    Map.of("name", "Alice", "id", 1L));
```

### SqlParameterSource Options

| Class | Use case |
|-------|----------|
| `MapSqlParameterSource` | Explicit map of params |
| `BeanPropertySqlParameterSource` | Java bean props = params |
| `EmptySqlParameterSource` | No params |

---

## 8. SimpleJdbcInsert & SimpleJdbcCall

### SimpleJdbcInsert

Insert without writing SQL.

```java
@Repository
public class UserRepository {
    private final SimpleJdbcInsert insert;
    
    public UserRepository(JdbcTemplate jdbc) {
        this.insert = new SimpleJdbcInsert(jdbc)
            .withTableName("users")
            .usingGeneratedKeyColumns("id")
            .usingColumns("name", "email");
    }
    
    public Long insert(User user) {
        Map<String, Object> params = Map.of(
            "name", user.getName(),
            "email", user.getEmail());
        return insert.executeAndReturnKey(params).longValue();
    }
}
```

### SimpleJdbcCall (Stored Procedures)

```java
SimpleJdbcCall call = new SimpleJdbcCall(jdbc)
    .withProcedureName("sp_find_users")
    .declareParameters(
        new SqlParameter("in_status", Types.VARCHAR),
        new SqlOutParameter("out_count", Types.INTEGER));

Map<String, Object> result = call.execute(Map.of("in_status", "ACTIVE"));
int count = (int) result.get("out_count");
```

### With RowMapper

```java
SimpleJdbcCall call = new SimpleJdbcCall(jdbc)
    .withProcedureName("sp_get_user")
    .returningResultSet("user", userRowMapper)
    .declareParameters(new SqlParameter("id", Types.BIGINT));

Map<String, Object> result = call.execute(Map.of("id", 1L));
List<User> users = (List<User>) result.get("user");
```

---

## 9. Exception Translation

### Without Spring

```java
try {
    // SQL
} catch (SQLException e) {
    // Check vendor code: "23505" (PostgreSQL unique), "1062" (MySQL)
}
```

### With Spring

Spring translates `SQLException` → `DataAccessException` (unchecked) via `SQLExceptionTranslator`.

### Exception Hierarchy

```
DataAccessException (unchecked, root)
├── NonTransientDataAccessException (retry won't help)
│   ├── BadSqlGrammarException
│   ├── DataIntegrityViolationException
│   ├── DuplicateKeyException
│   ├── DataAccessResourceFailureException
│   ├── InvalidDataAccessApiUsageException
│   └── ...
├── TransientDataAccessException (retry may help)
│   ├── TransientDataAccessResourceException
│   ├── ConcurrencyFailureException
│   │   ├── OptimisticLockingFailureException
│   │   └── PessimisticLockingFailureException
│   └── QueryTimeoutException
├── RecoverableDataAccessException
└── ...
```

### Usage

```java
try {
    jdbc.update("INSERT INTO users (email) VALUES (?)", email);
} catch (DuplicateKeyException e) {
    throw new EmailAlreadyExistsException(email, e);
} catch (DataIntegrityViolationException e) {
    throw new ValidationException("Data integrity violation", e);
}
```

### How Translation Works

- Spring uses `SQLExceptionTranslator` (default: `SQLExceptionSubclassTranslator` + `SQLStateSQLExceptionTranslator`)
- It examines `SQLState`, error code, and vendor codes
- For `@Repository` beans, translation is applied via `PersistenceExceptionTranslationPostProcessor`

### Custom Translator

```java
@Component
public class MyTranslator extends SQLErrorCodeSQLExceptionTranslator {
    @Override
    protected DataAccessException customTranslate(String task, String sql, SQLException ex) {
        if (ex.getErrorCode() == 12345) {
            return new MyCustomException("Oops", ex);
        }
        return super.customTranslate(task, sql, ex);
    }
}
```

---

## 10. Transaction Integration

### @Transactional on Repository/Service

```java
@Service
public class UserService {
    private final UserRepository repo;
    
    public UserService(UserRepository repo) { this.repo = repo; }
    
    @Transactional
    public User createUserWithProfile(CreateUserRequest req) {
        Long id = repo.insert(new User(req.name(), req.email()));
        repo.insertProfile(id, req.profile());
        return repo.findById(id).orElseThrow();
    }
}
```

### Programmatic Transaction (TransactionTemplate)

```java
@Service
public class UserService {
    private final TransactionTemplate txTemplate;
    private final JdbcTemplate jdbc;
    
    public UserService(PlatformTransactionManager tm, JdbcTemplate jdbc) {
        this.txTemplate = new TransactionTemplate(tm);
        this.jdbc = jdbc;
    }
    
    public User create(...) {
        return txTemplate.execute(status -> {
            // ... operations
            return user;
        });
    }
}
```

### DataSourceTransactionManager

Auto-configured by Spring Boot when `DataSource` is present. Uses `Connection.setAutoCommit(false)` + `commit()/rollback()`.

### Best Practice

- Put `@Transactional` on **service** layer, not repository
- Batch inserts inside a single transaction
- Set `readOnly=true` for queries

---

## 11. JDBC vs JPA

| Aspect | Spring JDBC | Spring Data JPA |
|--------|------------|-----------------|
| SQL control | Full | Generated (JPQL/Criteria) |
| Learning curve | Low | Medium |
| Boilerplate | Medium (SQL + mapping) | Low |
| Performance | Higher (direct) | Overhead of ORM |
| Complex queries | Easy | Harder (native queries) |
| Batch operations | Native support | Requires config |
| Relationships | Manual | Auto (ORM) |
| Caching | Manual | 1st/2nd level |
| Schema changes | Manual | DDL auto |
| Best for | Reports, batch, high perf | CRUD, domain-rich apps |

**Rule of thumb:**
- **Simple CRUD with objects** → JPA
- **Complex reporting/batch** → JDBC
- **Hybrid** — both in same app is common

---

## 12. Interview Questions

### Q1. Why use JdbcTemplate over plain JDBC?

**Answer:** Eliminates boilerplate (connection, statement, resultset, close), provides automatic resource management, translates `SQLException` to unchecked `DataAccessException`, supports RowMapper for clean mapping, offers batch operations, and integrates with Spring transactions. Focus remains on SQL, not plumbing.

### Q2. Difference between RowMapper and ResultSetExtractor?

**Answer:** `RowMapper` maps **one row** to an object; Spring iterates. `ResultSetExtractor` receives the **entire ResultSet** and you control iteration — needed when the output requires combining multiple rows (e.g., customer with orders in one query).

### Q3. What is NamedParameterJdbcTemplate?

**Answer:** A wrapper around `JdbcTemplate` that supports **named parameters** (`:name`) instead of positional `?`. Cleaner, safer, especially when the same parameter is used multiple times or when the SQL is long.

### Q4. How does Spring handle SQLException?

**Answer:** Spring wraps `SQLException` (checked, vendor-specific) into `DataAccessException` (unchecked, portable) via `SQLExceptionTranslator`. Subclasses communicate the nature of the failure (`DuplicateKeyException`, `BadSqlGrammarException`, etc.).

### Q5. How to insert and get generated ID with JdbcTemplate?

**Answer:** Use `KeyHolder` + `PreparedStatementCreator`:

```java
KeyHolder keyHolder = new GeneratedKeyHolder();
jdbc.update(con -> {
    PreparedStatement ps = con.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
    // set params
    return ps;
}, keyHolder);
Long id = keyHolder.getKey().longValue();
```

### Q6. What is SimpleJdbcInsert?

**Answer:** A helper that builds `INSERT` statements from metadata — you don't write SQL. Configure with table name and generated key column. Provides `execute()`, `executeAndReturnKey()`, `executeAndReturnKeyHolder()`.

### Q7. How to perform batch insert?

**Answer:** `jdbc.batchUpdate(sql, List<Object[]>)` or `jdbc.batchUpdate(sql, BatchPreparedStatementSetter)`. For named params, use `NamedParameterJdbcTemplate.batchUpdate(sql, SqlParameterSource[])`. Wrap in a transaction for efficiency.

### Q8. How to enable exception translation?

**Answer:** Automatically applies if the class is a Spring bean. For extra safety, annotate DAO with `@Repository` — Spring applies `PersistenceExceptionTranslationPostProcessor` which wraps and translates. With `JdbcTemplate` directly, translation already happens.

### Q9. What is BeanPropertyRowMapper?

**Answer:** A `RowMapper` that auto-maps `ResultSet` columns to a bean's fields/setters by name. Supports underscore-to-camelCase mapping. Convenient but slower than a hand-written `RowMapper` due to reflection.

### Q10. How to query a list of values?

**Answer:** `jdbc.queryForList("SELECT name FROM users", String.class)` returns `List<String>`. For a single value: `queryForObject`.

### Q11. How to handle `EmptyResultDataAccessException`?

**Answer:** Wrap the `queryForObject` call in try/catch and return `Optional.empty()`, or use `query` and check size. There is no built-in "Optional-returning" variant.

### Q12. JdbcTemplate vs JPA?

**Answer:** `JdbcTemplate` gives full SQL control with less abstraction — great for reports/batch. JPA handles ORM, relationships, caching — great for domain modeling. Many production systems mix both: JPA for CRUD, JDBC for complex queries/batch.

### Q13. How to configure multiple DataSources?

**Answer:** Define multiple `@Bean DataSource`s, mark one `@Primary`, create separate `JdbcTemplate` beans qualified by name, and use `@Qualifier` at injection. Use `DataSourceBuilder.create().build()` bound to distinct property prefixes.

### Q14. What is `SimpleJdbcCall`?

**Answer:** A helper for calling stored procedures/functions. Configure procedure name, IN/OUT parameters, and RowMapper for returning result sets. Handles `CallableStatement` boilerplate.

### Q15. How do you get a `Connection` manually?

**Answer:** Use `jdbc.execute(ConnectionCallback<T>)`:

```java
jdbc.execute((ConnectionCallback<String>) con -> con.getMetaData().getDatabaseProductName());
```

Forgetting to close isn't an issue — Spring does it.

### Q16. What is the difference between `execute` and `update`?

**Answer:** `execute(String sql)` runs DDL/any SQL and returns void. `update(...)` runs DML (INSERT/UPDATE/DELETE) and returns the number of affected rows.

### Q17. How to bind a `List` to an `IN` clause in NamedParameterJdbcTemplate?

**Answer:** Pass it as a named parameter value:

```java
MapSqlParameterSource p = new MapSqlParameterSource("ids", ids);
namedJdbc.query("SELECT * FROM users WHERE id IN (:ids)", p, mapper);
```

Spring expands the list automatically.

### Q18. How to log executed SQL?

**Answer:** Set logger:

```properties
logging.level.org.springframework.jdbc.core=DEBUG
logging.level.org.springframework.jdbc.core.JdbcTemplate=DEBUG
logging.level.org.springframework.jdbc.core.StatementCreatorUtils=TRACE
```

### Q19. When would you choose JDBC over JPA?

**Answer:**
- Complex reports/analytics with custom SQL
- Bulk data operations
- Stored procedure usage
- Full control over SQL and performance
- Simple, mostly-tabular data
- Avoiding ORM overhead (memory/perf)

### Q20. How do you test JdbcTemplate code?

**Answer:** Options:
- **`@JdbcTest`** (Spring Boot) — auto-configures an in-memory DB + `JdbcTemplate`
- **Testcontainers** — real DB in Docker
- **H2 in-memory** — fast, some dialect differences
- **Mockito** — mock `JdbcTemplate` for unit tests

```java
@JdbcTest
class UserRepositoryTest {
    @Autowired JdbcTemplate jdbc;
    
    @Test
    void insertAndFind() {
        jdbc.update("INSERT INTO users (id, name) VALUES (?, ?)", 1L, "Alice");
        String name = jdbc.queryForObject(
            "SELECT name FROM users WHERE id = ?", String.class, 1L);
        assertThat(name).isEqualTo("Alice");
    }
}
```

---

## 13. Cheat Sheet

### Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

```properties
spring.datasource.url=jdbc:...
spring.datasource.username=...
spring.datasource.password=...
spring.datasource.hikari.maximum-pool-size=20
```

### Common Operations

```java
// Insert
jdbc.update("INSERT INTO t (a,b) VALUES (?,?)", a, b);

// Insert returning id
KeyHolder kh = new GeneratedKeyHolder();
jdbc.update(con -> {
    PreparedStatement ps = con.prepareStatement(sql, RETURN_GENERATED_KEYS);
    // set params
    return ps;
}, kh);
Long id = kh.getKey().longValue();

// Query single value
jdbc.queryForObject("SELECT COUNT(*) FROM t", Integer.class);

// Query single row
jdbc.queryForObject("SELECT * FROM t WHERE id=?", rowMapper, id);

// Query list
jdbc.query("SELECT * FROM t", rowMapper);

// Update
jdbc.update("UPDATE t SET a=? WHERE id=?", a, id);

// Delete
jdbc.update("DELETE FROM t WHERE id=?", id);

// Batch
jdbc.batchUpdate("INSERT INTO t (a,b) VALUES (?,?)", listOfArrays);

// Named params
namedJdbc.query("SELECT * FROM t WHERE a=:a", Map.of("a", 1), rowMapper);
```

### RowMapper Example

```java
RowMapper<User> userRowMapper = (rs, rowNum) -> new User(
    rs.getLong("id"),
    rs.getString("name"),
    rs.getString("email"));
```

### Exception Hierarchy (Memory Aid)

```
DataAccessException
├── NonTransient (bad SQL, integrity)
│   ├── DuplicateKey
│   ├── DataIntegrityViolation
│   └── BadSqlGrammar
├── Transient (retry-able)
│   ├── ConcurrencyFailure
│   ├── QueryTimeout
│   └── TransientDataAccessResource
└── Recoverable
```

### Debug Logging

```properties
logging.level.org.springframework.jdbc.core=DEBUG
logging.level.org.springframework.jdbc.core.JdbcTemplate=TRACE
logging.level.org.springframework.jdbc.datasource.DataSourceTransactionManager=DEBUG
```

### Cross-References

- **Previous:** `03_Spring_MVC.md`
- **Next:** `05_Spring_Transaction_Management.md`
- **Related:** `10_Spring_Data_JPA.md`, `11_Spring_Data_JDBC.md`
- **Interview:** `24_Spring_Interview_Questions.md`

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [03_Spring_MVC.md](./03_Spring_MVC.md)
- **Next →:** [05_Spring_Transaction_Management.md](./05_Spring_Transaction_Management.md)
- **Related:** [10_Spring_Data_JPA.md](./10_Spring_Data_JPA.md), [11_Spring_Data_JDBC.md](./11_Spring_Data_JDBC.md), [05_Spring_Transaction_Management.md](./05_Spring_Transaction_Management.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
