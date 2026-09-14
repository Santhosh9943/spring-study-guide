# Spring Testing

> **File:** `16_Spring_Testing.md`
> **Part:** 5 — Testing
> **Prerequisites:** `01_Spring_Framework_Core.md`, `06_Spring_Boot_Fundamentals.md`
> **Estimated Study Time:** 10–14 hours

---

## Table of Contents

1. [Testing Philosophy](#1-testing-philosophy)
2. [JUnit 5 Fundamentals](#2-junit-5-fundamentals)
3. [Spring Boot Test Annotations Overview](#3-spring-boot-test-annotations-overview)
4. [@SpringBootTest](#4-springboottest)
5. [@WebMvcTest](#5-webmvctest)
6. [@DataJpaTest](#6-datajpatest)
7. [Other Test Slices](#7-other-test-slices)
8. [Mockito & @MockBean / @SpyBean](#8-mockito--mockbean--spybean)
9. [MockMvc](#9-mockmvc)
10. [TestRestTemplate & WebTestClient](#10-testresttemplate--webtestclient)
11. [Testcontainers](#11-testcontainers)
12. [@Sql & Test Data](#12-sql--test-data)
13. [@Transactional Testing](#13-transactional-testing)
14. [@TestPropertySource & @DynamicPropertySource](#14-testpropertysource--dynamicpropertysource)
15. [Test Coverage (JaCoCo)](#15-test-coverage-jacoco)
16. [Interview Questions](#16-interview-questions)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. Testing Philosophy

### Test Pyramid

```
       ▲
      / \
     /   \        E2E / UI (few, slow, brittle)
    /-----\
   /       \      Integration (medium)
  /---------\
 /           \    Unit (many, fast, isolated)
/_____________\
```

- **Unit** — one class, all deps mocked
- **Integration** — multiple layers, real DB / broker
- **E2E** — full app, real HTTP

### Testing in Spring

Spring provides:
- Test context framework (DI in tests)
- Slices (`@WebMvcTest`, `@DataJpaTest`) — focused contexts
- Mock infrastructure (`@MockBean`, `MockMvc`)
- Testcontainers integration

### Speed vs Confidence Trade-off

| Type | Speed | Realism | Fragility |
|------|-------|---------|-----------|
| Unit | Fast | Low | Low |
| Slice | Medium | Medium | Medium |
| Full `@SpringBootTest` | Slow | High | High |
| E2E | Slowest | Highest | Highest |

**Recommendation:** Many unit tests, focused slices for critical paths, few full integrations.

---

## 2. JUnit 5 Fundamentals

### Basic Test

```java
class CalculatorTest {
    
    @Test
    void addReturnsSum() {
        Calculator c = new Calculator();
        assertEquals(5, c.add(2, 3));
    }
}
```

### Lifecycle

```java
class LifecycleTest {
    
    @BeforeAll
    static void beforeAll() { /* once */ }
    
    @BeforeEach
    void setUp() { /* per test */ }
    
    @AfterEach
    void tearDown() { /* per test */ }
    
    @AfterAll
    static void afterAll() { /* once */ }
    
    @Test
    void test1() { }
}
```

### Assertions

```java
assertEquals(expected, actual)
assertNotEquals(1, 2)
assertTrue(condition)
assertFalse(condition)
assertNull(obj)
assertNotNull(obj)
assertThrows(IllegalArgumentException.class, () -> service.call())
assertDoesNotThrow(() -> service.call())
assertIterableEquals(list1, list2)
assertAll(
    () -> assertEquals(1, x),
    () -> assertEquals(2, y)
)
```

### Assumptions

```java
assumeTrue(condition);   // skip test if not true
assumeFalse(condition);
```

### Parameterized Tests

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3, 4, 5})
void isPositive(int n) {
    assertTrue(n > 0);
}

@ParameterizedTest
@CsvSource({
    "1, 1, 2",
    "2, 3, 5",
    "10, 20, 30"
})
void add(int a, int b, int expected) {
    assertEquals(expected, calculator.add(a, b));
}

@ParameterizedTest
@MethodSource("provideUsers")
void testUser(User u) { ... }

static Stream<User> provideUsers() {
    return Stream.of(new User("a"), new User("b"));
}

@ParameterizedTest
@EnumSource(Status.class)
void testStatus(Status s) { ... }
```

### Nested Tests

```java
class UserServiceTest {
    
    @Nested
    class WhenActive {
        @Test void canLogin() { ... }
    }
    
    @Nested
    class WhenInactive {
        @Test void cannotLogin() { ... }
    }
}
```

### Display Names

```java
@DisplayName("User service")
class UserServiceTest {
    @Test
    @DisplayName("should find user by id")
    void findById() { ... }
}
```

### Tags

```java
@Tag("integration")
class SlowTest { }
```

Run specific tags:

```
mvn test -Dgroups=integration
```

### AssertJ (preferred in Spring Boot)

```java
import static org.assertj.core.api.Assertions.assertThat;

assertThat(user.getName()).isEqualTo("Alice");
assertThat(list).hasSize(3).contains("a");
assertThat(optional).isPresent();
assertThatThrownBy(() -> service.call())
    .isInstanceOf(IllegalStateException.class)
    .hasMessage("Bad");
```

### Given-When-Then Pattern

```java
@Test
void placeOrderDeductsStock() {
    // given
    Product product = new Product("X", 10);
    when(repo.findById(1L)).thenReturn(Optional.of(product));
    
    // when
    service.placeOrder(1L, 3);
    
    // then
    assertThat(product.getStock()).isEqualTo(7);
    verify(repo).save(product);
}
```

---

## 3. Spring Boot Test Annotations Overview

| Annotation | Context | Use |
|------------|---------|-----|
| `@SpringBootTest` | Full | Integration |
| `@WebMvcTest` | Controllers | MVC slice |
| `@DataJpaTest` | JPA | Repository slice |
| `@JdbcTest` | JDBC | JdbcTemplate slice |
| `@DataJdbcTest` | Spring Data JDBC | JDBC repositories |
| `@DataMongoTest` | MongoDB | Mongo slice |
| `@DataRedisTest` | Redis | Redis slice |
| `@JsonTest` | JSON | Jackson slice |
| `@RestClientTest` | REST client | RestTemplate/WebClient |
| `@WebFluxTest` | WebFlux controllers | Reactive MVC slice |
| `@JooqTest` | jOOQ | jOOQ slice |

### Slice Tests

Slice tests load a **subset** of the application context — faster and focused.

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

Includes: JUnit 5, Mockito, AssertJ, Hamcrest, JSONassert, JsonPath, Spring Test, XMLUnit, Awaitility.

---

## 4. @SpringBootTest

Loads the **full application context**.

### Basic

```java
@SpringBootTest
class ApplicationTest {
    
    @Autowired
    private UserService userService;
    
    @Test
    void contextLoads() {
        assertThat(userService).isNotNull();
    }
}
```

### Web Environment Modes

```java
@SpringBootTest(webEnvironment = WebEnvironment.MOCK)          // default
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)   // real server
@SpringBootTest(webEnvironment = WebEnvironment.DEFINED_PORT)  // port 8080
@SpringBootTest(webEnvironment = WebEnvironment.NONE)          // no web
```

| Mode | Provides |
|------|----------|
| `MOCK` | Mock servlet environment (`MockMvc` auto-configured if `@AutoConfigureMockMvc`) |
| `RANDOM_PORT` | Real embedded server on random port |
| `DEFINED_PORT` | Real embedded server on `server.port` |
| `NONE` | Non-web context |

### MOCK + MockMvc

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerIT {
    
    @Autowired MockMvc mvc;
    
    @Test
    void getUsers() throws Exception {
        mvc.perform(get("/api/users"))
            .andExpect(status().isOk());
    }
}
```

### RANDOM_PORT + TestRestTemplate

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class UserApiIT {
    
    @Autowired TestRestTemplate rest;
    
    @Test
    void getUsers() {
        ResponseEntity<User[]> res = rest.getForEntity("/api/users", User[].class);
        assertThat(res.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

### Properties

```java
@SpringBootTest(properties = {
    "app.feature.enabled=true",
    "logging.level.com.example=DEBUG"
})
```

### Custom Configuration

```java
@SpringBootTest(classes = {MyApp.class, TestConfig.class})
class IntegrationTest { }
```

### WebClient (WebFlux)

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class ReactiveTest {
    @Autowired WebTestClient webTestClient;
    
    @Test
    void getUsers() {
        webTestClient.get().uri("/api/users")
            .exchange()
            .expectStatus().isOk();
    }
}
```

---

## 5. @WebMvcTest

Loads **only the web layer**: controllers, `@ControllerAdvice`, `@JsonComponent`, filters, `WebMvcConfigurer`. **Does not** load `@Service`, `@Repository`, `@Component`.

### Basic

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired MockMvc mvc;
    
    @MockBean UserService userService;
    
    @Test
    void getReturnsUser() throws Exception {
        when(userService.findById(1L)).thenReturn(new User(1L, "Alice"));
        
        mvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"));
    }
}
```

### Load All Controllers

```java
@WebMvcTest   // no controller class → all controllers loaded
```

### Includes

- `@Controller`, `@RestController`, `@ControllerAdvice`
- `Filter`, `WebMvcConfigurer`, `HandlerMethodArgumentResolver`
- `MockMvc`

### Excludes

- `@Component`, `@Service`, `@Repository`
- Full datasource
- Security filters (unless `@Import`ed)

### With Security

```java
@WebMvcTest(UserController.class)
@Import(SecurityConfig.class)
class SecuredControllerTest {
    
    @Autowired MockMvc mvc;
    
    @MockBean UserDetailsService uds;
    
    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanAccess() throws Exception {
        mvc.perform(get("/admin")).andExpect(status().isOk());
    }
}
```

Spring Security auto-config applies to `@WebMvcTest` if on classpath.

### Mocking Dependencies

```java
@MockBean UserService userService;
@MockBean OrderService orderService;
```

Every dependency the controller needs must be `@MockBean`ed.

### Advice & Filters

`@ControllerAdvice` and filters are auto-detected. Mock them or use real ones.

### Adding Custom Beans

```java
@WebMvcTest(UserController.class)
@Import(TestConfig.class)
class Test { }

@TestConfiguration
static class TestConfig {
    @Bean
    CustomMapper mapper() { return new CustomMapper(); }
}
```

---

## 6. @DataJpaTest

Loads only JPA components: entities, repositories, `TestEntityManager`.

### Basic

```java
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired TestEntityManager em;
    @Autowired UserRepository repo;
    
    @Test
    void findByEmail() {
        User u = em.persistAndFlush(new User("Alice", "alice@x.com"));
        
        Optional<User> found = repo.findByEmail("alice@x.com");
        
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Alice");
    }
}
```

### Defaults

- Auto-configures in-memory DB (H2) if available
- Each test is `@Transactional` → auto-rollback
- Replaces `DataSource` with embedded (can be disabled)
- `TestEntityManager` for test-friendly operations

### Real DB

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
class RealDbTest { ... }
```

Or with Testcontainers:

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = Replace.NONE)
class PostgresTest {
    
    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");
    
    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }
}
```

### TestEntityManager

```java
User persisted = em.persistAndFlush(new User("Alice"));
User found = em.find(User.class, persisted.getId());
em.remove(found);
em.flush();
```

Methods: `persist`, `persistAndFlush`, `persistAndGetId`, `find`, `merge`, `remove`, `flush`, `clear`, `refresh`.

### Transactional by Default

Every test is transactional → rolls back after. To disable:

```java
@DataJpaTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)
```

Or use `@Commit`.

---

## 7. Other Test Slices

### @JdbcTest

```java
@JdbcTest
class UserDaoTest {
    @Autowired JdbcTemplate jdbc;
    
    @Test
    void insert() {
        jdbc.update("INSERT INTO users (id, name) VALUES (?, ?)", 1L, "Alice");
        String name = jdbc.queryForObject(
            "SELECT name FROM users WHERE id = ?", String.class, 1L);
        assertThat(name).isEqualTo("Alice");
    }
}
```

Loads `DataSource` + `JdbcTemplate` (no JPA).

### @JsonTest

```java
@JsonTest
class UserJsonTest {
    @Autowired JacksonTester<User> json;
    
    @Test
    void serialize() throws Exception {
        User u = new User(1L, "Alice");
        assertThat(json.write(u)).hasJsonPathStringValue("@.name");
    }
    
    @Test
    void deserialize() throws Exception {
        String content = "{\"id\":1,\"name\":\"Alice\"}";
        assertThat(json.parseObject(content).getName()).isEqualTo("Alice");
    }
}
```

### @RestClientTest

```java
@RestClientTest(UserClient.class)
class UserClientTest {
    @Autowired MockRestServiceServer server;
    @Autowired UserClient client;
    
    @Test
    void get() {
        server.expect(requestTo("/users/1"))
            .andRespond(withSuccess("{\"id\":1,\"name\":\"Alice\"}", 
                                    MediaType.APPLICATION_JSON));
        
        User u = client.get(1L);
        assertThat(u.getName()).isEqualTo("Alice");
    }
}
```

### @DataMongoTest

```java
@DataMongoTest
class UserMongoTest {
    @Autowired MongoTemplate mongo;
    @Autowired UserRepository repo;
    
    @Test
    void saveAndFind() {
        User u = repo.save(new User(null, "Alice"));
        assertThat(repo.findById(u.getId())).isPresent();
    }
}
```

Requires Testcontainers for real Mongo.

### @WebFluxTest

```java
@WebFluxTest(UserController.class)
class ReactiveControllerTest {
    @Autowired WebTestClient client;
    @MockBean UserService userService;
    
    @Test
    void get() {
        when(userService.findById(1L)).thenReturn(Mono.just(new User(1L, "Alice")));
        
        client.get().uri("/api/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody().jsonPath("$.name").isEqualTo("Alice");
    }
}
```

### @DataJdbcTest

```java
@DataJdbcTest
class UserJdbcRepoTest {
    @Autowired UserRepository repo;
    
    @Test
    void saveAndFind() {
        User u = repo.save(new User(null, "Alice"));
        assertThat(repo.findById(u.id())).isPresent();
    }
}
```

---

## 8. Mockito & @MockBean / @SpyBean

### @MockBean

Replaces a bean in the context with a Mockito mock.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @MockBean UserService userService;
    
    @Test
    void test() {
        when(userService.find(1L)).thenReturn(new User());
        // ...
    }
}
```

- **Replaces** existing bean (or adds if absent)
- **Triggers context restart** if mock differs between tests (Spring caches contexts by config)

### @SpyBean

Wraps the real bean — real methods called unless stubbed.

```java
@SpringBootTest
class SpyTest {
    @SpyBean EmailService emailService;
    
    @Test
    void send() {
        doNothing().when(emailService).send(any());
        
        service.register("alice@x.com");
        
        verify(emailService).send("alice@x.com");
    }
}
```

### @Mock vs @MockBean

| Aspect | @Mock | @MockBean |
|--------|-------|-----------|
| Type | Mockito annotation | Spring Boot |
| Where | Anywhere | Test field of Spring context |
| Injects into | `@InjectMocks` | Spring context |
| Context effect | ❌ | Restarts context if mock differs |

### Mockito Basics

```java
// Stubbing
when(service.find(1L)).thenReturn(user);
when(service.find(anyLong())).thenThrow(new RuntimeException());
when(service.find(1L)).thenReturn(u1).thenReturn(u2);   // consecutive

// Void methods
doNothing().when(service).delete(anyLong());
doThrow(new RuntimeException()).when(service).delete(1L);

// Argument matchers
when(service.find(anyLong())).thenReturn(user);
when(service.create(any(), eq("Alice"))).thenReturn(user);

// Verification
verify(service).find(1L);
verify(service, times(2)).find(anyLong());
verify(service, never()).delete(anyLong());
verifyNoMoreInteractions(service);

// ArgumentCaptor
ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
verify(service).save(captor.capture());
assertThat(captor.getValue().getName()).isEqualTo("Alice");

// @Captor (field-based)
@Captor ArgumentCaptor<User> userCaptor;

// Reset (rare)
reset(service);
```

### MockitoExtension

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock UserRepository userRepo;
    @InjectMocks UserService userService;
    
    @Test
    void find() {
        when(userRepo.findById(1L)).thenReturn(Optional.of(new User()));
        assertThat(userService.find(1L)).isNotNull();
    }
}
```

- Pure unit test — no Spring context
- Fast

### Strict Stubs

Mockito JUnit 5 extension uses strict stubs by default. Unused stubs fail tests.

```java
@ExtendWith(MockitoExtension.class)
@MockitoSettings(strictness = Strictness.LENIENT)
```

Or lenient per stub:

```java
lenient().when(service.find(anyLong())).thenReturn(user);
```

### Stubbing Best Practices

- Stub the minimum needed
- Prefer `when(...).thenReturn(...)` (server-side)
- Use `doX().when(mock).method()` for void methods
- Prefer `verify` only when behavior matters

---

## 9. MockMvc

Server-side test framework for Spring MVC without starting a real server.

### Setup

```java
@WebMvcTest(UserController.class)
@AutoConfigureMockMvc   // only needed with @SpringBootTest
class UserControllerTest {
    @Autowired MockMvc mvc;
    @MockBean UserService userService;
}
```

### Request Builders

```java
mvc.perform(get("/api/users"))
mvc.perform(post("/api/users").contentType(MediaType.APPLICATION_JSON).content(json))
mvc.perform(put("/api/users/1").content(json).contentType(APPLICATION_JSON))
mvc.perform(delete("/api/users/1"))
mvc.perform(patch("/api/users/1").content(json))

// With params, headers, cookies
get("/api/users")
    .param("page", "0")
    .param("size", "20")
    .header("X-Trace", "abc")
    .cookie(new Cookie("session", "..."))
    .accept(MediaType.APPLICATION_JSON)
```

### Expectations

```java
.andExpect(status().isOk())
.andExpect(status().isCreated())
.andExpect(status().isBadRequest())
.andExpect(status().isNotFound())
.andExpect(status().isUnauthorized())
.andExpect(status().isForbidden())

.andExpect(content().contentType(MediaType.APPLICATION_JSON))
.andExpect(content().string("hello"))
.andExpect(content().json("{...}"))
.andExpect(jsonPath("$.name").value("Alice"))
.andExpect(jsonPath("$.users", hasSize(3)))
.andExpect(jsonPath("$.users[0].id").value(1))
.andExpect(jsonPath("$.users[*].name", contains("A", "B")))
.andExpect(jsonPath("$.id").exists())
.andExpect(jsonPath("$.id").doesNotExist())
.andExpect(jsonPath("$.id").isNumber())

.andExpect(header().exists("Location"))
.andExpect(header().string("Location", "/users/1"))
.andExpect(cookie().value("XSRF-TOKEN", "abc"))

.andExpect(model().attribute("key", "value"))
.andExpect(view().name("users/list"))
.andExpect(forwardedUrl("/error"))
.andExpect(redirectedUrl("/login"))
```

### JSON Content Comparison

```java
.andExpect(content().json("""
    {"id":1,"name":"Alice"}
"""))
```

Or with lenient mode:

```java
.andExpect(content().json(expectedJson, true));   // ignore extra fields
```

### Perform + Do

```java
MvcResult result = mvc.perform(get("/api/users"))
    .andDo(print())   // log request/response
    .andExpect(status().isOk())
    .andReturn();

String body = result.getResponse().getContentAsString();
```

### JSON Path Examples

```java
jsonPath("$").isArray()
jsonPath("$", hasSize(3))
jsonPath("$.name").value("Alice")
jsonPath("$.age").value(greaterThan(18))
jsonPath("$[?(@.active == true)]").exists()
jsonPath("$.items[*].name", contains("A", "B"))
```

### Security Testing

```java
@Test
@WithMockUser(roles = "ADMIN")
void adminCanAccess() throws Exception {
    mvc.perform(get("/api/admin")).andExpect(status().isOk());
}

@Test
@WithMockUser(roles = "USER")
void userForbidden() throws Exception {
    mvc.perform(get("/api/admin")).andExpect(status().isForbidden());
}

@Test
void anonymousUnauthorized() throws Exception {
    mvc.perform(get("/api/admin")).andExpect(status().isUnauthorized());
}
```

### Custom Request Post-Processor

```java
@Test
void withCsrf() throws Exception {
    mvc.perform(post("/api/users")
            .with(csrf())   // from SecurityMockMvcRequestPostProcessors
            .contentType(APPLICATION_JSON)
            .content("{}"))
        .andExpect(status().isCreated());
}
```

### File Upload

```java
MockMultipartFile file = new MockMultipartFile(
    "file", "test.txt", "text/plain", "content".getBytes());

mvc.perform(multipart("/api/upload").file(file))
    .andExpect(status().isOk());
```

### Async

```java
MvcResult result = mvc.perform(get("/async"))
    .andExpect(request().asyncStarted())
    .andReturn();

mvc.perform(asyncDispatch(result))
    .andExpect(status().isOk())
    .andExpect(content().string("Done"));
```

---

## 10. TestRestTemplate & WebTestClient

### TestRestTemplate (MVC)

Real HTTP client for `@SpringBootTest(webEnvironment = RANDOM_PORT)`.

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class UserApiIT {
    
    @Autowired TestRestTemplate rest;
    
    @Test
    void createAndGet() {
        User created = rest.postForObject("/api/users",
            new CreateUserRequest("Alice"), User.class);
        
        ResponseEntity<User> res = rest.getForEntity(
            "/api/users/" + created.getId(), User.class);
        
        assertThat(res.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(res.getBody().getName()).isEqualTo("Alice");
    }
    
    @Test
    void withAuth() {
        rest.withBasicAuth("user", "pwd").getForEntity("/api/me", User.class);
    }
}
```

### Methods

| Method | HTTP |
|--------|------|
| `getForObject` / `getForEntity` | GET |
| `postForObject` / `postForEntity` | POST |
| `put` / `delete` | PUT / DELETE |
| `exchange(url, method, entity, responseType)` | Any |

### WebTestClient (WebFlux / MVC)

Fluid API for reactive clients.

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class UserApiIT {
    
    @Autowired WebTestClient client;
    
    @Test
    void get() {
        client.get().uri("/api/users")
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(User.class).hasSize(3);
    }
    
    @Test
    void create() {
        client.post().uri("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(new CreateUserRequest("Alice"))
            .exchange()
            .expectStatus().isCreated()
            .expectBody().jsonPath("$.name").isEqualTo("Alice");
    }
}
```

### WebTestClient Bound to MockMvc

```java
@SpringBootTest
@AutoConfigureMockMvc
class Test {
    @Autowired MockMvc mvc;
    
    WebTestClient client;
    
    @BeforeEach
    void setUp() {
        client = WebTestClient.bindToController(new UserController()).build();
        // or:
        client = WebTestClient.bindTo(mvc).build();
    }
}
```

### TestRestTemplate vs WebTestClient

| Feature | TestRestTemplate | WebTestClient |
|---------|------------------|---------------|
| Blocking | ✅ | ✅ |
| Reactive | ❌ | ✅ |
| Works with WebFlux | ✅ (client side) | ✅ |
| Auto-configured | `@SpringBootTest(RANDOM_PORT)` | Same |
| Assertions | Manual | Fluent |

---

## 11. Testcontainers

Run real databases, brokers, etc. in Docker containers for tests.

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

### Basic

```java
@SpringBootTest
@Testcontainers
class UserIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("mydb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }
    
    @Test
    void saveAndFind() { ... }
}
```

### Spring Boot 3.1+ — @ServiceConnection

```java
@TestConfiguration(proxyBeanMethods = false)
class TestcontainersConfig {
    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgres() {
        return new PostgreSQLContainer<>("postgres:16");
    }
}
```

Spring auto-wires `spring.datasource.url` etc. from the container. No `@DynamicPropertySource` needed.

### Common Containers

```java
PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");
MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8");
MongoDBContainer mongo = new MongoDBContainer("mongo:7");
KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
RabbitMQContainer rabbit = new RabbitMQContainer("rabbitmq:3");
GenericContainer<?> redis = new GenericContainer<>("redis:7")
    .withExposedPorts(6379);
```

### Multiple Containers

```java
@Testcontainers
class FullStackIT {
    
    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    
    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
}
```

### Reusable Containers

Faster tests by not restarting containers between test classes:

```java
@Container
static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16")
    .withReuse(true);
```

Requires `~/.testcontainers.properties`:

```
testcontainers.reuse.enable=true
```

### Container Lifecycle

- `static` + `@Container` → shared across tests in class
- `static` + `@Testcontainers(parallel = true)` → parallel
- Non-static + `@Container` → per test (slower)

### When to Use Testcontainers

- Integration tests with real DB/broker
- Verify SQL dialect-specific behavior
- Test data migrations
- Avoid H2-vs-prod differences

### When NOT to Use

- Unit tests (overkill)
- Simple slices (`@DataJpaTest` with H2 sufficient)
- CI without Docker

---

## 12. @Sql & Test Data

### @Sql

Run SQL scripts before/after test methods.

```java
@DataJpaTest
@Sql("/test-data.sql")
class UserRepositoryTest { }
```

### Per-Method

```java
@Test
@Sql(scripts = "/users.sql", executionPhase = BEFORE_TEST_METHOD)
void test() { }

@Test
@Sql(scripts = "/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
```

### Inline

```java
@Test
@Sql(statements = {
    "INSERT INTO users (id, name) VALUES (1, 'Alice')",
    "INSERT INTO users (id, name) VALUES (2, 'Bob')"
})
void test() { }
```

### Class-Level

```java
@DataJpaTest
@Sql(scripts = "/schema.sql", executionPhase = BEFORE_TEST_CLASS)
@Sql(scripts = "/cleanup.sql", executionPhase = AFTER_TEST_CLASS)
class UserRepositoryTest { }
```

### @SqlConfig

```java
@Sql(scripts = "/x.sql",
     config = @SqlConfig(encoding = "UTF-8", separator = ";"))
```

### Default Test Data Scripts

Place `data.sql` in `src/test/resources/` — runs automatically for `@DataJpaTest` if configured.

### Boot's data.sql

```properties
spring.sql.init.mode=always
spring.sql.init.data-locations=classpath:data.sql
```

Runs on startup — useful for demos/tests.

### Best Practices

- Prefer Java fixtures over SQL for complex objects
- Use SQL for `@DataJpaTest` (faster than builders)
- Keep scripts small and focused
- One concern per file

### Alternative: Builders/Fixtures

```java
public class TestData {
    public static User alice() { return new User("Alice", "alice@x.com"); }
    public static User bob() { return new User("Bob", "bob@x.com"); }
}

@Test
void test() {
    User u = repo.save(TestData.alice());
}
```

Fluent with **Fixtures**:

```java
User u = UserFixtures.user()
    .withName("Alice")
    .withEmail("alice@x.com")
    .build();
```

---

## 13. @Transactional Testing

### Auto-Rollback

Tests annotated `@Transactional` (or within `@DataJpaTest`) roll back automatically.

```java
@SpringBootTest
@Transactional
class UserServiceTest {
    
    @Test
    void createUser() {
        userService.create("Alice");
        // auto-rollback after test
    }
}
```

### Why It's Both Good and Bad

**Good:**
- Fast isolation
- No cleanup needed

**Bad:**
- Hides flush issues (Hibernate may not flush before assertions)
- Hides LazyInitializationException
- Doesn't test transaction boundaries
- Doesn't test propagation

### Best Practice

For **unit-ish integration tests** → `@Transactional` is fine.
For **true integration tests** → run without `@Transactional`, clean DB explicitly.

### Disabling

```java
@SpringBootTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)
```

Or use `@Commit`:

```java
@Test
@Commit
void persistsData() { }
```

### Flush Issues

```java
@Transactional
void test() {
    service.create("Alice");
    // Not flushed! DB may not have the row yet
    
    // Force flush:
    entityManager.flush();
    // or
    userRepo.flush();
    
    assertThat(userRepo.count()).isEqualTo(1);
}
```

### TransactionTemplate in Tests

```java
@Autowired TransactionTemplate txTemplate;

@Test
void test() {
    txTemplate.execute(status -> {
        service.doWork();
        status.setRollbackOnly();   // explicit rollback
        return null;
    });
}
```

### Cleanup Strategies (No @Transactional)

1. `@Sql` cleanup script
2. `@DirtiesContext` (recreate context — slow)
3. `@BeforeEach` delete all
4. Testcontainers with fresh DB per test (slow)
5. Truncate in `@AfterEach`

---

## 14. @TestPropertySource & @DynamicPropertySource

### @TestPropertySource

Static properties for tests.

```java
@SpringBootTest
@TestPropertySource(properties = {
    "app.feature.enabled=true",
    "logging.level.root=WARN"
})
class Test { }
```

Or from file:

```java
@TestPropertySource(locations = "classpath:test-application.properties")
```

Or inline:

```java
@TestPropertySource(locations = {
    "classpath:base.properties",
    "classpath:test-overrides.properties"
})
```

### @ActiveProfiles

```java
@SpringBootTest
@ActiveProfiles("test")
class Test { }
```

Activates profile → loads `application-test.properties`.

### @DynamicPropertySource

Register properties from runtime sources (e.g., containers).

```java
@DynamicPropertySource
static void props(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", () -> pg.getJdbcUrl());
    registry.add("spring.datasource.username", pg::getUsername);
    registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
}
```

Providers are `Supplier<T>` — evaluated lazily.

### Precedence in Tests

High to low:
1. `@DynamicPropertySource`
2. `@TestPropertySource(properties=...)`
3. `@TestPropertySource(locations=...)`
4. `@ActiveProfiles` → profile files
5. `application.properties`

### Test Properties Files

`src/test/resources/application-test.properties`:

```properties
spring.datasource.url=jdbc:h2:mem:test
spring.jpa.hibernate.ddl-auto=create-drop
logging.level.root=WARN
```

Activate via `@ActiveProfiles("test")`.

---

## 15. Test Coverage (JaCoCo)

### Maven Plugin

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

Report: `target/site/jacoco/index.html`.

### Gradle Plugin

```groovy
plugins {
    id 'jacoco'
}

jacocoTestReport {
    reports {
        xml.required = true
        html.required = true
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.80
            }
        }
    }
}
```

### Coverage Metrics

- **Line** — covered lines / total
- **Branch** — covered branches / total
- **Instruction** — bytecode
- **Method** / **Class**

### Target Coverage

| Layer | Target |
|-------|--------|
| Service | 80–90% |
| Controller | 60–70% |
| Repository | 50–70% (integration-tested) |
| Config | Low (accept) |
| DTO | Low |

**Don't chase 100%** — target meaningful tests.

### Excluding Classes

```xml
<configuration>
    <excludes>
        <exclude>**/dto/**</exclude>
        <exclude>**/config/**</exclude>
        <exclude>**/*Application.class</exclude>
    </excludes>
</configuration>
```

### SonarQube Integration

JaCoCo XML report feeds SonarQube.

```xml
<sonar.coverage.jacoco.xmlReportPaths>
    target/site/jacoco/jacoco.xml
</sonar.coverage.jacoco.xmlReportPaths>
```

---

## 16. Interview Questions

### Q1. What is @SpringBootTest?

**Answer:** Loads the full application context — all beans, configurations, and auto-configurations. Use for integration tests. Supports web environment modes: `MOCK` (default, MockMvc), `RANDOM_PORT` (real server), `DEFINED_PORT`, `NONE`.

### Q2. What are slice tests?

**Answer:** Focused tests that load a subset of the context. Examples: `@WebMvcTest` (controllers), `@DataJpaTest` (repositories), `@JsonTest` (Jackson). Faster than `@SpringBootTest` and isolate the layer under test.

### Q3. @WebMvcTest vs @SpringBootTest?

**Answer:** `@WebMvcTest` loads only the web layer (controllers, `@ControllerAdvice`, filters) with a mocked service layer. `@SpringBootTest` loads everything. Use `@WebMvcTest` for controller tests with `@MockBean` deps.

### Q4. What does @DataJpaTest do?

**Answer:** Configures an in-memory DB (H2 if present), replaces the `DataSource`, enables JPA repositories and `TestEntityManager`, and marks tests `@Transactional` (auto-rollback). Only JPA-related beans are loaded.

### Q5. Difference between @Mock and @MockBean?

**Answer:** `@Mock` is pure Mockito — no Spring integration. `@MockBean` replaces a bean in the Spring context (or adds one). `@MockBean` can cause context restarts if mocks differ between tests, slowing the suite.

### Q6. What is MockMvc?

**Answer:** A server-side test framework for Spring MVC — sends mock requests to controllers without a real server. Fluent API for building requests (`get`, `post`), asserting responses (`status`, `jsonPath`, `content`).

### Q7. How do you test a secured endpoint?

**Answer:** Use `@WithMockUser`, `@WithUserDetails`, `@WithAnonymousUser`, or `.with(jwt()...)`. Or `@SpringBootTest` + `TestRestTemplate` with basic auth.

### Q8. What is Testcontainers?

**Answer:** A library that runs real databases/brokers in Docker containers for tests. Ensures tests run against the production-like infrastructure. Integrates via `@Container` and `@DynamicPropertySource` (or `@ServiceConnection` in Boot 3.1+).

### Q9. What is @ServiceConnection?

**Answer:** Spring Boot 3.1+ feature — auto-wires connection properties from a Testcontainer bean. Replaces `@DynamicPropertySource`. Just annotate a container `@Bean` with `@ServiceConnection`.

### Q10. What is @Sql used for?

**Answer:** Runs SQL scripts before/after test methods or classes. Common in `@DataJpaTest` for setup/cleanup. Supports `statements`, `scripts`, `executionPhase`.

### Q11. What does @Transactional on a test do?

**Answer:** Wraps each test in a transaction that rolls back afterward. Fast isolation. Downside: may hide flush issues and lazy loading exceptions since Hibernate defers/flushes differently.

### Q12. Why might a @Transactional test pass but production fail?

**Answer:** Because the DB isn't actually hit — dirty-checking delays flushes, lazy loading works (session open), constraints may not fire. Real integration tests (without `@Transactional`) or Testcontainers catch these.

### Q13. What is @DynamicPropertySource?

**Answer:** A static test method that dynamically adds properties to the `Environment`. Common with Testcontainers — you register `spring.datasource.url` from the running container. Uses `Supplier` for lazy evaluation.

### Q14. What is the difference between @TestPropertySource and @ActiveProfiles?

**Answer:** `@TestPropertySource` adds specific properties/files. `@ActiveProfiles` activates a profile so profile-specific files (`application-test.properties`) load. Often used together.

### Q15. How do you test async controllers?

**Answer:** Use MockMvc's async support: `andExpect(request().asyncStarted())` then `mvc.perform(asyncDispatch(result))`. Or use `Awaitility` for `@Async` beans.

### Q16. How do you test a REST client (RestTemplate/WebClient)?

**Answer:** `@RestClientTest(Client.class)` + `MockRestServiceServer` for RestTemplate, or `MockWebServer` (OkHttp) / `WireMock` for WebClient. Assert requests made and stub responses.

### Q17. What is JaCoCo?

**Answer:** A code coverage library for Java. Integrates with Maven/Gradle to generate reports (`target/site/jacoco/`). Provides line/branch coverage metrics. Used in CI to enforce thresholds.

### Q18. How do you test WebFlux?

**Answer:** `@WebFluxTest(Controller.class)` + `WebTestClient`. Mock `@MockBean` services returning `Mono`/`Flux`. `WebTestClient` gives a fluent reactive API.

### Q19. What is @AutoConfigureMockMvc?

**Answer:** Auto-configures `MockMvc` in a `@SpringBootTest`. Needed for `MOCK` web environment because MockMvc isn't part of the slice by default. `@WebMvcTest` enables it automatically.

### Q20. What's the difference between @WebMvcTest and @AutoConfigureMockMvc?

**Answer:** `@WebMvcTest` is a **slice** — loads only web layer. `@AutoConfigureMockMvc` is an **auto-config annotation** — adds MockMvc to any test (often used with `@SpringBootTest` to get MockMvc in a full context).

### Q21. How do you test a JPA query?

**Answer:** `@DataJpaTest` + `TestEntityManager` to persist fixtures + `@Autowired` repository. Assert results. For native/JPQL queries, Testcontainers with real DB is best to verify dialect behavior.

### Q22. What is @DirtiesContext?

**Answer:** Marks the test context dirty so it's discarded and rebuilt for the next test. Slow. Avoid except for tests that mutate global state (e.g., dynamic property sources that can't be reset).

### Q23. How do you test with security context?

**Answer:** `@WithMockUser` (any username/roles), `@WithUserDetails("alice")` (loads actual user), `@WithAnonymousUser`. For JWT, `.with(jwt().jwt(j -> ...))`. For custom principals, write a `@WithSecurityContext` annotation.

### Q24. What is the difference between integration and slice tests?

**Answer:** Integration tests load the full application context (`@SpringBootTest`) — highest fidelity, slowest. Slice tests load a subset — faster, focused, but may miss cross-layer issues.

### Q25. How do you test repository layer without a real DB?

**Answer:** Use `@DataJpaTest` with H2 (default). H2 mimics SQL but has dialect differences. For critical SQL, use Testcontainers with a real PostgreSQL/MySQL container.

---

## 17. Cheat Sheet

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

### Test Annotations

```java
@SpringBootTest                          // full context
@SpringBootTest(webEnvironment = RANDOM_PORT)
@WebMvcTest(UserController.class)        // MVC slice
@DataJpaTest                             // JPA slice
@JdbcTest                                // JDBC slice
@DataJdbcTest                            // Spring Data JDBC slice
@JsonTest                                // Jackson slice
@RestClientTest(UserClient.class)        // REST client slice
@WebFluxTest(UserController.class)       // WebFlux slice
@Testcontainers                          // Testcontainers
@ActiveProfiles("test")
@TestPropertySource(properties = "...")
@DynamicPropertySource                   // For containers
@AutoConfigureMockMvc                    // with @SpringBootTest
@MockBean / @SpyBean                     // replace/wrap beans
@WithMockUser(roles = "ADMIN")           // security testing
@WithUserDetails("alice")
@WithAnonymousUser
@Transactional                           // auto-rollback
@Sql("/test-data.sql")
```

### MockMvc Essentials

```java
mvc.perform(post("/api/users")
        .contentType(APPLICATION_JSON)
        .content(json))
    .andExpect(status().isCreated())
    .andExpect(header().exists("Location"))
    .andExpect(jsonPath("$.name").value("Alice"))
    .andDo(print())
    .andReturn();
```

### JUnit 5 Essentials

```java
@Test
@BeforeEach / @AfterEach
@BeforeAll / @AfterAll
@ParameterizedTest @ValueSource / @CsvSource / @MethodSource
@Nested
@DisplayName
@Tag("integration")
@Disabled("reason")
assertThrows / assertDoesNotThrow / assertAll
assumeTrue / assumeFalse
```

### AssertJ

```java
assertThat(x).isEqualTo(y);
assertThat(list).hasSize(3).contains("a");
assertThat(optional).isPresent();
assertThatThrownBy(() -> ...).isInstanceOf(...).hasMessage("...");
```

### Mockito

```java
when(repo.findById(1L)).thenReturn(Optional.of(user));
when(repo.save(any())).thenAnswer(i -> i.getArgument(0));
doThrow(new RuntimeException()).when(svc).delete(anyLong());

verify(repo, times(2)).save(any());
verify(repo, never()).delete(anyLong());
ArgumentCaptor<User> cap = ArgumentCaptor.forClass(User.class);
verify(repo).save(cap.capture());
```

### Testcontainers

```java
@Testcontainers
class IT {
    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");
    
    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }
}
```

**Boot 3.1+ shortcut:**

```java
@TestConfiguration
class TcConfig {
    @Bean @ServiceConnection
    PostgreSQLContainer<?> pg() { return new PostgreSQLContainer<>("postgres:16"); }
}
```

### Test Properties (src/test/resources/application-test.properties)

```properties
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=create-drop
logging.level.root=WARN
```

### JaCoCo Coverage

```bash
mvn test jacoco:report
# Report at target/site/jacoco/index.html
```

### Test Pyramid Reminder

```
        /\
       /E2\       Few — slow, real, high confidence
      /────\
     /Inte \     Medium — slices / full context
    /──────\
   /  Unit  \    Many — fast, mocked
  /──────────\
```

### Common Pitfalls

| Symptom | Cause |
|---------|-------|
| Context restart per test | Different `@MockBean` sets per test class |
| Slow suite | `@SpringBootTest` for everything |
| Passes but prod fails | `@Transactional` hiding flush/lazy issues |
| `LazyInitializationException` | Real integration issue, not test bug |
| Testcontainers fail in CI | Docker not available |
| Wrong schema in test | H2 vs Postgres dialect |
| Security tests fail | Missing `spring-security-test` |
| Async test flakes | Use Awaitility or `asyncDispatch` |

### Cross-References

- **Previous:** `15_Spring_Security_Method_Level.md`
- **Next:** `17_Maven_Build_Tool.md`
- **Related:** `10_Spring_Data_JPA.md`, `13_Spring_Security_Core.md`
- **Interview:** `24_Spring_Interview_Questions.md` (§Testing)

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [15_Spring_Security_Method_Level.md](./15_Spring_Security_Method_Level.md)
- **Next →:** [17_Maven_Build_Tool.md](./17_Maven_Build_Tool.md)
- **Related:** [06_Spring_Boot_Fundamentals.md](./06_Spring_Boot_Fundamentals.md), [13_Spring_Security_Core.md](./13_Spring_Security_Core.md), [17_Maven_Build_Tool.md](./17_Maven_Build_Tool.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
