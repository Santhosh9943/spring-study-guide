# Spring Transaction Management

> **File:** `05_Spring_Transaction_Management.md`
> **Part:** 1 — Foundations
> **Prerequisites:** `01_Spring_Framework_Core.md`, `02_Spring_AOP.md`, `04_Spring_JDBC.md`
> **Estimated Study Time:** 8–12 hours

---

## Table of Contents

1. [ACID Properties](#1-acid-properties)
2. [Why Transaction Management?](#2-why-transaction-management)
3. [Programmatic Transactions](#3-programmatic-transactions)
4. [Declarative Transactions: @Transactional](#4-declarative-transactions-transactional)
5. [Propagation Behaviors](#5-propagation-behaviors)
6. [Isolation Levels](#6-isolation-levels)
7. [Rollback Rules](#7-rollback-rules)
8. [readOnly, timeout, and Other Attributes](#8-readonly-timeout-and-other-attributes)
9. [Transaction Managers](#9-transaction-managers)
10. [Class vs Method Level](#10-class-vs-method-level)
11. [Common Pitfalls](#11-common-pitfalls)
12. [Transaction Synchronization & Events](#12-transaction-synchronization--events)
13. [Nested Transactions & Savepoints](#13-nested-transactions--savepoints)
14. [Reactive Transactions](#14-reactive-transactions)
15. [Interview Questions](#15-interview-questions)
16. [Cheat Sheet](#16-cheat-sheet)

---

## 1. ACID Properties

Every reliable transaction satisfies ACID:

| Property | Meaning | Example |
|----------|---------|---------|
| **Atomicity** | All or nothing | Transfer: debit AND credit, or neither |
| **Consistency** | Valid state → valid state | Constraints hold before/after |
| **Isolation** | Concurrent txns don't interfere | Two transfers don't corrupt balances |
| **Durability** | Committed data survives crash | Committed transfer persists after power loss |

### Atomicity Example

```
BEGIN
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT
```

If either UPDATE fails → rollback both.

### Isolation Anomalies

| Anomaly | Description |
|---------|-------------|
| **Dirty Read** | Read uncommitted data from another txn |
| **Non-Repeatable Read** | Same row returns different values in same txn |
| **Phantom Read** | Same query returns different rows in same txn |
| **Lost Update** | Two txns overwrite each other |
| **Write Skew** | Each txn individually valid, combined invalid |

Isolation levels trade off performance vs anomaly prevention (§6).

---

## 2. Why Transaction Management?

### Without Transactions

```java
public void transfer(Long from, Long to, BigDecimal amt) {
    accountRepo.debit(from, amt);   // succeeds
    // ⚠️ app crashes here
    accountRepo.credit(to, amt);    // never runs → money lost
}
```

### With Transactions

```java
@Transactional
public void transfer(Long from, Long to, BigDecimal amt) {
    accountRepo.debit(from, amt);
    accountRepo.credit(to, amt);    // both commit, or both rollback
}
```

### Spring's Value

- **Declarative** — `@Transactional` annotation; no manual BEGIN/COMMIT
- **Portable** — same code across JDBC, JPA, JMS, JTA
- **AOP-based** — proxies wrap methods
- **Propagation & isolation** — configurable per method
- **Integration** — works with all Spring data access

---

## 3. Programmatic Transactions

### 3.1 PlatformTransactionManager

Core SPI:

```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition);
    void commit(TransactionStatus status);
    void rollback(TransactionStatus status);
}
```

### 3.2 Manual — Low-level

```java
@Autowired
private PlatformTransactionManager txManager;

public void doWork() {
    DefaultTransactionDefinition def = new DefaultTransactionDefinition();
    def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    
    TransactionStatus status = txManager.getTransaction(def);
    try {
        // ... business logic
        txManager.commit(status);
    } catch (Exception e) {
        txManager.rollback(status);
        throw e;
    }
}
```

### 3.3 TransactionTemplate (RECOMMENDED programmatic style)

```java
@Service
public class TransferService {
    private final TransactionTemplate txTemplate;
    
    public TransferService(PlatformTransactionManager txManager) {
        this.txTemplate = new TransactionTemplate(txManager);
        this.txTemplate.setTimeout(30);
        this.txTemplate.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    }
    
    public void transfer(Long from, Long to, BigDecimal amt) {
        txTemplate.execute(status -> {
            accountRepo.debit(from, amt);
            accountRepo.credit(to, amt);
            return null;
        });
    }
    
    // With result
    public BigDecimal getBalance() {
        return txTemplate.execute(status -> {
            return accountRepo.getTotalBalance();
        });
    }
    
    // Rollback programmatically
    public void conditional() {
        txTemplate.execute(status -> {
            if (somethingWrong) {
                status.setRollbackOnly();
            }
            return null;
        });
    }
}
```

**Pros:**
- Explicit, easy to read
- No AOP surprises
- Works with self-invocation

**Cons:**
- More verbose than `@Transactional`
- Requires injecting the template

### 3.4 TransactionCallbackWithoutResult

```java
txTemplate.execute(new TransactionCallbackWithoutResult() {
    @Override
    protected void doInTransactionWithoutResult(TransactionStatus status) {
        // no return value
    }
});
```

### When to Use Programmatic

- Complex transaction logic (conditional rollback, loops)
- Self-invocation contexts
- Fine-grained control
- Non-Spring-managed objects

---

## 4. Declarative Transactions: @Transactional

### Enabling

**Spring Boot:** auto-enabled via `TransactionAutoConfiguration`.

**Standalone Spring:**

```java
@Configuration
@EnableTransactionManagement
public class TxConfig {
    @Bean
    public PlatformTransactionManager txManager(DataSource ds) {
        return new DataSourceTransactionManager(ds);
    }
}
```

### Basic Usage

```java
@Service
public class UserService {
    
    @Transactional
    public User createUser(CreateUserRequest req) {
        User user = userRepo.save(new User(req.name()));
        profileRepo.save(new Profile(user.getId(), req.bio()));
        return user;
    }
}
```

- Any `RuntimeException` → rollback
- Any checked exception → **commit** (by default!)

### Attributes of @Transactional

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.DEFAULT,
    timeout = 30,
    readOnly = false,
    rollbackFor = {SQLException.class, MyException.class},
    noRollbackFor = {BusinessWarning.class},
    transactionManager = "txManager"
)
public void doWork() { ... }
```

### How It Works (AOP)

1. Spring creates a **proxy** around the bean
2. Calls to `@Transactional` methods go through `TransactionInterceptor` (`@Around` advice)
3. Interceptor:
   - Gets/creates a `TransactionStatus` from the manager
   - Invokes `proceed()` (target method)
   - On success → `commit`
   - On exception (per rollback rules) → `rollback`

### Full Interceptor Flow

```
Caller → Proxy → TransactionInterceptor.invoke()
    ├─ Determine tx attribute from @Transactional
    ├─ getTransaction(def) → TransactionStatus
    ├─ try:
    │    ├─ Invoke target method
    │    ├─ commit(status)
    │    └─ return result
    └─ catch (Throwable t):
         ├─ if rollbackOn(t): rollback(status)
         └─ else: commit(status)
         throw t
```

### Proxy Types

Same as AOP (§09 in `02_Spring_AOP.md`):
- **JDK proxy** by default (interface-based)
- **CGLIB** for classes / if `proxyTargetClass=true`
- Spring Boot 2.0+ → CGLIB default

---

## 5. Propagation Behaviors

Propagation defines **how a transactional method behaves when called within an existing transaction**.

### The 7 Propagations

| Propagation | Behavior |
|-------------|----------|
| **REQUIRED** (default) | Join existing txn, or create new if none |
| **REQUIRES_NEW** | Always create new txn, suspend existing |
| **SUPPORTS** | Join if exists, else run non-transactionally |
| **NOT_SUPPORTED** | Suspend existing, run non-transactionally |
| **MANDATORY** | Must have existing txn, else throw |
| **NEVER** | Must NOT have existing txn, else throw |
| **NESTED** | Savepoint within existing txn (create new if none) |

### Detailed Behavior

#### REQUIRED (default)

```
Caller (no txn) → method() → NEW txn
Caller (txn A) → method() → joins A
```

```java
@Transactional  // default = REQUIRED
public void update() { ... }
```

**Use case:** 95% of cases.

#### REQUIRES_NEW

```
Caller (txn A) → method() → suspend A, create B, commit/rollback B, resume A
```

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logAction(String action) {
    auditRepo.save(new AuditRecord(action));  // always commits separately
}
```

**Use case:** Audit logging that must persist even if the outer txn rolls back.

⚠️ **Requires a second DB connection** — can deadlock if pool is exhausted.

#### SUPPORTS

```
Caller (txn A) → method() → joins A
Caller (no txn) → method() → runs without txn
```

```java
@Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
public User find(Long id) { ... }
```

**Use case:** Read methods that work either way.

#### NOT_SUPPORTED

```
Caller (txn A) → method() → suspend A, run without txn, resume A
```

**Use case:** Long operations that shouldn't hold a connection.

#### MANDATORY

```
Caller (txn A) → method() → joins A
Caller (no txn) → method() → throws IllegalTransactionStateException
```

**Use case:** Enforcing that callers always manage transactions.

#### NEVER

```
Caller (txn A) → method() → throws IllegalTransactionStateException
Caller (no txn) → method() → runs normally
```

**Use case:** Guarantee method is never called in a transaction.

#### NESTED

```
Caller (txn A) → method() → savepoint SP inside A
    Inner fails → rollback to SP (A continues)
    Inner succeeds → commit SP (still inside A)
Caller (no txn) → method() → NEW txn (like REQUIRED)
```

```java
@Transactional(propagation = Propagation.NESTED)
public void step1() { ... }
```

**Use case:** Partial rollback — "try this step; if it fails, skip it, keep the rest".

**Requirements:** JDBC (`DataSourceTransactionManager`) supports savepoints. JPA/Hibernate supports via `JpaTransactionManager` with savepoints.

### Propagation Visual

```
REQUIRED:
  Outer [────── A ──────]
  Inner   [── A ──]

REQUIRES_NEW:
  Outer [── A ──]  [B]  [── A ──]
  Inner            [B]

NESTED:
  Outer [── A ──────────────]
  Inner     [savepoint B ]
            (rollback B doesn't kill A)
```

### Comparison

| Propagation | Joins outer? | New txn? | Suspend outer? | Savepoint? |
|-------------|--------------|----------|---------------|-----------|
| REQUIRED | ✅ (if exists) | ✅ (if none) | ❌ | ❌ |
| REQUIRES_NEW | ❌ | ✅ (always) | ✅ | ❌ |
| SUPPORTS | ✅ (if exists) | ❌ | ❌ | ❌ |
| NOT_SUPPORTED | ❌ | ❌ | ✅ | ❌ |
| MANDATORY | ✅ (required) | ❌ | ❌ | ❌ |
| NEVER | ❌ | ❌ | ❌ | ❌ |
| NESTED | ✅ (as savepoint) | ✅ (if none) | ❌ | ✅ |

### Real-World Example

```java
@Service
public class OrderService {
    
    @Autowired private AuditService audit;
    
    @Transactional  // REQUIRED
    public Order placeOrder(Order o) {
        Order saved = orderRepo.save(o);
        
        try {
            paymentService.charge(saved);          // REQUIRED (joins)
        } catch (PaymentException e) {
            audit.logFailedPayment(saved, e);      // REQUIRES_NEW
            throw e;                                // outer rolls back
        }
        
        audit.logSuccess(saved);                    // REQUIRES_NEW (persists)
        return saved;
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logSuccess(Order o) { auditRepo.save(...); }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logFailedPayment(Order o, Exception e) { auditRepo.save(...); }
}
```

Result: failed payments leave an audit trail even though the order rolls back.

---

## 6. Isolation Levels

Isolation controls **how much concurrent transactions see of each other**.

### The Four Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| **READ_UNCOMMITTED** | ✅ Possible | ✅ Possible | ✅ Possible |
| **READ_COMMITTED** | ❌ Prevented | ✅ Possible | ✅ Possible |
| **REPEATABLE_READ** | ❌ | ❌ | ✅ Possible |
| **SERIALIZABLE** | ❌ | ❌ | ❌ |

### `Isolation.DEFAULT`

Uses DB default (PostgreSQL/MySQL InnoDB → READ_COMMITTED; Oracle → READ_COMMITTED; SQL Server → READ_COMMITTED).

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void transfer(...) { ... }
```

### Consistency vs Concurrency

- Higher isolation = safer, but **slower** (more locking, less concurrency)
- Lower isolation = faster, but risk of anomalies

### Real-World Usage

- **READ_COMMITTED** — most common default; balances safety & speed
- **REPEATABLE_READ** — for reports requiring stable reads
- **SERIALIZABLE** — financial transactions, but beware of deadlocks and retries

### Optimistic vs Pessimistic Locking

**Optimistic** — assume no conflict, check at commit:

```java
@Entity
public class Product {
    @Version
    private Long version;
}

@Transactional
public void update(Long id) {
    Product p = repo.findById(id).orElseThrow();
    p.setPrice(newPrice);
    // flush → if version changed, OptimisticLockingFailureException
}
```

**Pessimistic** — lock at read:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select p from Product p where p.id = :id")
Product findForUpdate(@Param("id") Long id);
```

### Interview Tip

> Do **not** change isolation casually — it affects DB-level behavior and can cause deadlocks or performance issues. Default (DB default) is usually correct.

---

## 7. Rollback Rules

### Default Behavior

- **RuntimeException** → rollback
- **Error** → rollback
- **Checked Exception** → **commit** (surprising!)

```java
@Transactional
public void doWork() throws IOException {
    // throws IOException → transaction COMMITS (unintended!)
}
```

### rollbackFor / noRollbackFor

```java
@Transactional(rollbackFor = {IOException.class, MyChecked.class})
public void doWork() throws IOException { ... }

@Transactional(noRollbackFor = {BusinessWarning.class})
public void doWork() { ... }
```

### Best Practice

Always specify `rollbackFor = Exception.class` if you want all exceptions to roll back:

```java
@Transactional(rollbackFor = Exception.class)
public void doWork() throws Exception { ... }
```

### Programmatic Rollback

```java
@Transactional
public void doWork() {
    // ...
    TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
}
```

### Catch-and-Rethrow Trap

```java
@Transactional
public void outer() {
    try {
        inner();                // also @Transactional
    } catch (Exception e) {
        // swallowed → transaction still commits!
        log.error("Ignored", e);
    }
}
```

If you want rollback, **rethrow** or `setRollbackOnly()`.

### Inner Call Caught by Outer

```java
@Transactional
public void outer() {
    try {
        inner();      // @Transactional(REQUIRED) → same txn
    } catch (RuntimeException e) {
        // outer catches; but inner already marked rollback-only!
        // → UnexpectedRollbackException at commit
    }
}
```

Fix: use `REQUIRES_NEW` on inner, or ensure outer rolls back too.

---

## 8. readOnly, timeout, and Other Attributes

### readOnly

```java
@Transactional(readOnly = true)
public List<User> search(String q) { ... }
```

**Effects:**
- JDBC: sets `Connection.setReadOnly(true)` (hint)
- Hibernate: `Session.setDefaultReadOnly(true)` — skips dirty checking
- **Optimization**, not enforcement — DB must enforce

**Use case:** All read methods. Ensures no accidental writes.

### timeout

```java
@Transactional(timeout = 30)   // seconds
public void longRunning() { ... }
```

- Causes `TransactionTimedOutException` if exceeded
- Requires DB/driver support
- **Not a substitute for query timeout** — a long query may still run

### transactionManager

```java
@Transactional(transactionManager = "secondaryTxManager")
public void inSecondary() { ... }
```

For multiple `PlatformTransactionManager`s.

### value / propagation aliases

```java
@Transactional("secondaryTxManager")   // value = txManager bean name
@Transactional(propagation = Propagation.REQUIRES_NEW)
```

### Full Attribute List

| Attribute | Type | Default |
|-----------|------|---------|
| `propagation` | `Propagation` | `REQUIRED` |
| `isolation` | `Isolation` | `DEFAULT` |
| `timeout` | `int` (sec) | `-1` (no timeout) |
| `readOnly` | `boolean` | `false` |
| `rollbackFor` | `Class[]` | `{}` |
| `rollbackForClassName` | `String[]` | `{}` |
| `noRollbackFor` | `Class[]` | `{}` |
| `noRollbackForClassName` | `String[]` | `{}` |
| `transactionManager` | `String` | primary |

---

## 9. Transaction Managers

### Common Implementations

| Manager | Use Case |
|---------|----------|
| `DataSourceTransactionManager` | Plain JDBC |
| `JpaTransactionManager` | JPA / Hibernate |
| `HibernateTransactionManager` | Hibernate (native) |
| `JtaTransactionManager` | Distributed (XA) |
| `ChainedTransactionManager` | Multiple resources (deprecated in Spring 6) |
| `ReactiveTransactionManager` | WebFlux / R2DBC |

### Configuration — JDBC

```java
@Bean
public PlatformTransactionManager txManager(DataSource ds) {
    return new DataSourceTransactionManager(ds);
}
```

### Configuration — JPA

```java
@Bean
public PlatformTransactionManager txManager(EntityManagerFactory emf) {
    return new JpaTransactionManager(emf);
}
```

Spring Boot auto-configures both when dependencies are present.

### Multiple Managers

```java
@Bean
public PlatformTransactionManager primaryTx(DataSource ds) {
    return new DataSourceTransactionManager(ds);
}

@Bean
public PlatformTransactionManager secondaryTx(
        @Qualifier("secondaryDs") DataSource ds) {
    return new DataSourceTransactionManager(ds);
}
```

```java
@Transactional("secondaryTx")
public void inSecondary() { ... }
```

### JTA — Distributed Transactions

For multi-DB, DB + MQ, etc. Requires an XA-capable transaction manager (Narayana, Atomikos, Bitronix).

```java
@Bean
public PlatformTransactionManager txManager() {
    return new JtaTransactionManager();
}
```

⚠️ 2-phase commit is slow — often avoided via **Sagas** or **Outbox pattern** in microservices.

---

## 10. Class vs Method Level

### Class-Level

```java
@Service
@Transactional
public class UserService {
    public User create(...) { ... }   // inherits @Transactional
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)   // overrides
    public void log(...) { ... }
    
    @Transactional(readOnly = true)   // overrides
    public User find(Long id) { ... }
}
```

- **All public methods** become transactional
- **Method-level** overrides class-level
- Private/protected methods are **not advised**

### Recommended Pattern

```java
@Service
@Transactional(readOnly = true)   // default for reads
public class UserService {
    
    public User find(Long id) { ... }   // readOnly
    
    @Transactional                   // read-write for writes
    public User create(...) { ... }
}
```

### Interface-Level (avoid)

Spring supports `@Transactional` on interfaces, but:
- Not recommended by Spring docs
- Confusing with JDK proxies
- **Always annotate the concrete class/method**

### Rollback Exception During Commit

If commit fails, `TransactionSystemException` is thrown **after** the method returns. In that case, rollback already happened — you can't recover.

---

## 11. Common Pitfalls

### 11.1 Self-Invocation (The #1 Pitfall)

```java
@Service
public class UserService {
    
    public void outer() {
        this.inner();      // ❌ proxy bypassed
    }
    
    @Transactional
    public void inner() { ... }
}
```

**Why:** AOP proxy wraps the bean, but `this.inner()` is a plain Java call — no proxy.

**Fixes:**

```java
// Option 1: Split into another bean
@Service public class UserService {
    @Autowired private UserTxService tx;
    public void outer() { tx.inner(); }
}

// Option 2: Inject self
@Service public class UserService {
    @Autowired @Lazy private UserService self;
    public void outer() { self.inner(); }
}

// Option 3: AopContext
@EnableAspectJAutoProxy(exposeProxy = true)
...
((UserService) AopContext.currentProxy()).inner();
```

### 11.2 Private / Protected Methods

`@Transactional` on private → **silently ignored**.

```java
@Transactional
private void doWork() { ... }   // ⚠️ never transactional!
```

Fix: make it public, or move to another bean.

### 11.3 Final Methods / Classes

CGLIB cannot override final → advice skipped silently.

### 11.4 Checked Exception Rollback

By default, `IOException` won't roll back. Add `rollbackFor = Exception.class`.

### 11.5 Catch and Swallow

```java
@Transactional
public void outer() {
    try { inner(); } catch (Exception e) { /* ignored */ }
    // still commits
}
```

Rethrow or `setRollbackOnly()`.

### 11.6 Wrong Propagation

- `REQUIRES_NEW` with a small connection pool → pool exhaustion
- `NESTED` on JPA without savepoint support → falls back to REQUIRED silently

### 11.7 Long Transactions

- Hold DB locks
- Block other txns
- Cause deadlocks
- Cause connection pool starvation

**Keep transactions short. No HTTP calls inside transactions.**

### 11.8 readOnly Not Enforced

`readOnly=true` is a **hint**. Some databases ignore it. Accidental writes still succeed in some dialects.

### 11.9 Testing With @Transactional

`@Transactional` on a test method **rolls back automatically** — but this hides flushing issues (JPA). Prefer explicit cleanup with Testcontainers.

### 11.10 Multiple Transaction Managers

Without specifying `transactionManager`, Spring uses the primary. Mixing managers can silently misconfigure.

### 11.11 Async + @Transactional

`@Async` + `@Transactional` on the same method → the async thread runs **without** the caller's transaction. Ensure the async method itself is `@Transactional`.

### 11.12 Proxy vs Target Casting

```java
UserServiceImpl impl = ctx.getBean(UserServiceImpl.class);  // ❌ NoSuchBeanDefinition
UserService svc = ctx.getBean(UserService.class);           // ✅
```

### Checklist

- [ ] Method is **public**
- [ ] No **self-invocation**
- [ ] Class/method **not final**
- [ ] Bean is **Spring-managed**
- [ ] `rollbackFor` set if checked exceptions expected
- [ ] No **catch-and-swallow**
- [ ] Propagation appropriate
- [ ] readOnly hint used for reads
- [ ] No remote calls inside txn
- [ ] No `this.` calls to transactional methods

---

## 12. Transaction Synchronization & Events

### TransactionSynchronizationManager

```java
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            // runs only if commit succeeded
        }
        @Override
        public void afterCompletion(int status) {
            // status: COMMITTED / ROLLED_BACK
        }
    });
```

### @TransactionalEventListener

Fire events **after commit** or **after completion**:

```java
@Component
public class OrderListener {
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlaced(OrderPlacedEvent event) {
        emailService.sendConfirmation(event.orderId());
    }
}
```

| Phase | When |
|-------|------|
| `BEFORE_COMMIT` | Before commit |
| `AFTER_COMMIT` (default) | After successful commit |
| `AFTER_ROLLBACK` | After rollback |
| `AFTER_COMPLETION` | After either |

`fallbackExecution = true` runs even without a transaction.

### Use Case: Transactional Outbox

```java
@Transactional
public void placeOrder(Order o) {
    orderRepo.save(o);
    outboxRepo.save(new OutboxMessage("OrderPlaced", o.getId()));
    // event listener publishes to Kafka AFTER_COMMIT
}
```

---

## 13. Nested Transactions & Savepoints

### NESTED via Savepoints

```java
@Transactional
public void outer() {
    doA();
    try {
        inner();          // NESTED
    } catch (Exception e) {
        // rollback to savepoint — outer continues
    }
    doB();
}

@Transactional(propagation = Propagation.NESTED)
public void inner() { ... }
```

**JDBC:**

```java
Savepoint sp = connection.setSavepoint("inner");
try {
    // ...
} catch (Exception e) {
    connection.rollback(sp);
}
```

### NESTED vs REQUIRES_NEW

| Aspect | NESTED | REQUIRES_NEW |
|--------|--------|--------------|
| New connection | ❌ | ✅ |
| Independent commit | ❌ | ✅ |
| Savepoint-based | ✅ | ❌ |
| Outer rollback kills inner | ✅ | ❌ |
| Pool exhaustion risk | Low | High |

**Rule:** NESTED for partial rollback; REQUIRES_NEW for truly independent work.

---

## 14. Reactive Transactions

### ReactiveTransactionManager (WebFlux + R2DBC)

```java
@Bean
public ReactiveTransactionManager reactiveTxManager(ConnectionFactory cf) {
    return new R2dbcTransactionManager(cf);
}
```

### @Transactional Works

```java
@Service
public class UserService {
    @Transactional
    public Mono<User> create(Mono<CreateUserRequest> req) {
        return req.flatMap(r -> userRepo.save(new User(r.name())));
    }
}
```

Rules:
- Must return `Mono` / `Flux`
- Uses `ReactiveTransactionManager`
- No ThreadLocal — uses Reactor Context
- Not compatible with blocking `PlatformTransactionManager`

### Programmatic Reactive

```java
TransactionalOperator operator = TransactionalOperator.create(txManager);
return operator.transactional(userRepo.save(user));
```

---

## 15. Interview Questions

### Q1. What is a transaction? ACID?

**Answer:** A transaction is a unit of work executed atomically. ACID: **A**tomicity (all/nothing), **C**onsistency (valid state → valid state), **I**solation (concurrent txns don't interfere), **D**urability (committed data persists).

### Q2. Programmatic vs declarative transactions?

**Answer:** **Programmatic** (`TransactionTemplate`) is explicit in code — more verbose but flexible. **Declarative** (`@Transactional`) uses AOP — cleaner, less code, but with pitfalls (self-invocation). Use declarative by default, programmatic for complex logic.

### Q3. How does @Transactional work internally?

**Answer:** Spring creates a **proxy** (JDK or CGLIB) around the bean. When a `@Transactional` method is called, `TransactionInterceptor` (an `@Around` advice) intercepts, obtains a `TransactionStatus` from `PlatformTransactionManager`, invokes the method, then commits or rolls back based on the outcome.

### Q4. Explain propagation behaviors.

**Answer:**
- `REQUIRED` (default) — join or create
- `REQUIRES_NEW` — always new, suspend outer
- `SUPPORTS` — join if present, else none
- `NOT_SUPPORTED` — suspend, run without
- `MANDATORY` — must have outer
- `NEVER` — must not have outer
- `NESTED` — savepoint within outer

### Q5. What's the difference between REQUIRES_NEW and NESTED?

**Answer:** `REQUIRES_NEW` opens a **new independent transaction** (new connection, own commit). `NESTED` uses a **savepoint within the same transaction** — inner rollback only reverts to the savepoint; outer can continue. `REQUIRES_NEW` consumes an extra connection.

### Q6. What are the isolation levels?

**Answer:** `READ_UNCOMMITTED` (dirty reads allowed), `READ_COMMITTED` (no dirty reads), `REPEATABLE_READ` (no non-repeatable), `SERIALIZABLE` (fully isolated). Higher isolation = more locking = lower concurrency.

### Q7. Which exceptions trigger rollback by default?

**Answer:** `RuntimeException` and `Error`. **Checked exceptions do NOT roll back** by default. Use `rollbackFor = Exception.class` to include checked.

### Q8. What is self-invocation problem?

**Answer:** Calling a `@Transactional` method from within the same bean via `this.method()` bypasses the proxy — no transaction advice applied. Fix: extract to another bean, inject self, or use `AopContext.currentProxy()`.

### Q9. Can you use @Transactional on private methods?

**Answer:** No — proxy-based AOP only advises public methods. Annotation on private is silently ignored.

### Q10. How do you configure transactions in Spring Boot?

**Answer:** Spring Boot auto-configures `PlatformTransactionManager` (JPA → `JpaTransactionManager`, JDBC → `DataSourceTransactionManager`) and enables `@EnableTransactionManagement`. Just annotate methods with `@Transactional`.

### Q11. What is `TransactionSynchronizationManager`?

**Answer:** Manages transaction-scoped resources (connections, sessions) via ThreadLocals. Supports registering `TransactionSynchronization` callbacks (`beforeCommit`, `afterCommit`, `afterCompletion`).

### Q12. What is `@TransactionalEventListener`?

**Answer:** A listener for application events, invoked at a specific transaction phase (default `AFTER_COMMIT`). Ensures event handling happens only after the transaction commits — avoiding side effects on rolled-back work.

### Q13. Can you have multiple transaction managers?

**Answer:** Yes. Define multiple `PlatformTransactionManager` beans, mark one `@Primary`, and select via `@Transactional("beanName")` or `@Transactional(transactionManager = "..."`.

### Q14. What is a distributed transaction? Do you use JTA?

**Answer:** Multi-resource transaction requiring 2-phase commit (2PC). JTA + XA managers (Atomikos, Narayana) provide it. It's slow and fragile at scale. Modern microservices use **Sagas** or the **Outbox pattern** instead.

### Q15. What is savepoint?

**Answer:** A marker inside a transaction to which you can roll back **without** ending the whole transaction. Used by `Propagation.NESTED` via `Connection.setSavepoint()`.

### Q16. Can @Transactional be used on interfaces?

**Answer:** Yes, but not recommended. Spring docs advise annotating concrete classes/methods. On interfaces with JDK proxies, it works, but it's fragile with CGLIB and inheritance.

### Q17. How does Spring handle timeout?

**Answer:** `@Transactional(timeout=30)` sets a transaction timeout. Spring calls `setTransactionTimeout` on the manager, which delegates to the JDBC driver/JTA. Exceeding → `TransactionTimedOutException`.

### Q18. What is `readOnly=true` and how does it help?

**Answer:** Sets `Connection.setReadOnly(true)` and disables Hibernate dirty checking. It's a hint — improves performance and discourages accidental writes, but DB must enforce.

### Q19. How to test @Transactional code?

**Answer:** Use `@DataJpaTest` (auto-rollback), `@SpringBootTest` with `@Transactional`, or Testcontainers for real DB. Note that Spring's `@Transactional` on tests rolls back automatically — sometimes hiding flush issues.

### Q20. Why might `UnexpectedRollbackException` occur?

**Answer:** An inner method marked the transaction as **rollback-only** (e.g., an exception propagated from a REQUIRED inner method was caught by the outer), and the outer tried to commit. Spring throws `UnexpectedRollbackException` at commit time.

---

## 16. Cheat Sheet

### @Transactional

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.READ_COMMITTED,
    timeout = 30,
    readOnly = false,
    rollbackFor = Exception.class,
    noRollbackFor = BusinessWarning.class,
    transactionManager = "txManager"
)
```

### Propagation (Memory Aid)

```
REQUIRED      = join or create (default)
REQUIRES_NEW  = always new (suspend outer)
SUPPORTS      = join if exists
NOT_SUPPORTED = suspend, no txn
MANDATORY     = must have outer
NEVER         = must not have outer
NESTED        = savepoint inside outer
```

### Isolation

```
READ_UNCOMMITTED → dirty reads allowed
READ_COMMITTED   → no dirty reads
REPEATABLE_READ  → no dirty + no non-repeatable
SERIALIZABLE     → full isolation
```

### Default Rollback

```
RuntimeException → ROLLBACK
Error            → ROLLBACK
Checked          → COMMIT (surprising!)
```

### Programmatic

```java
@Autowired TransactionTemplate txTemplate;

txTemplate.execute(status -> {
    // work
    if (fail) status.setRollbackOnly();
    return result;
});
```

### Debug Logging

```properties
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.transaction.interceptor=TRACE
logging.level.org.springframework.jdbc.datasource.DataSourceTransactionManager=DEBUG
```

### Pitfalls Quick List

- Self-invocation
- Private methods
- Final methods/classes
- Checked exception rollback missing
- Catch-and-swallow
- Wrong propagation
- Long transactions
- readOnly assumed enforced
- Mixing tx managers
- Async + @Transactional mismatch

### Cross-References

- **Previous:** `04_Spring_JDBC.md`
- **Next:** `06_Spring_Boot_Fundamentals.md`
- **Related:** `10_Spring_Data_JPA.md`, `19_Spring_Microservices_Cloud.md` (Sagas)
- **Interview:** `24_Spring_Interview_Questions.md`

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [04_Spring_JDBC.md](./04_Spring_JDBC.md)
- **Next →:** [06_Spring_Boot_Fundamentals.md](./06_Spring_Boot_Fundamentals.md)
- **Related:** [04_Spring_JDBC.md](./04_Spring_JDBC.md), [10_Spring_Data_JPA.md](./10_Spring_Data_JPA.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
