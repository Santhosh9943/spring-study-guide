# Spring Boot Fundamentals

> **File:** `06_Spring_Boot_Fundamentals.md`
> **Part:** 2 — Spring Boot
> **Prerequisites:** `01_Spring_Framework_Core.md`
> **Estimated Study Time:** 8–10 hours

---

## Table of Contents

1. [What is Spring Boot?](#1-what-is-spring-boot)
2. [Spring Boot vs Spring Framework](#2-spring-boot-vs-spring-framework)
3. [@SpringBootApplication](#3-springbootapplication)
4. [Auto-Configuration](#4-auto-configuration)
5. [Conditional Annotations](#5-conditional-annotations)
6. [Starters](#6-starters)
7. [Embedded Servers](#7-embedded-servers)
8. [Spring Initializr](#8-spring-initializr)
9. [Spring Boot CLI](#9-spring-boot-cli)
10. [application.properties vs yml](#10-applicationproperties-vs-yml)
11. [DevTools](#11-devtools)
12. [Versioning & Release Cadence](#12-versioning--release-cadence)
13. [Migrating from Spring to Spring Boot](#13-migrating-from-spring-to-spring-boot)
14. [Interview Questions](#14-interview-questions)
15. [Cheat Sheet](#15-cheat-sheet)

---

## 1. What is Spring Boot?

**Spring Boot** is an opinionated, convention-over-configuration framework built on top of Spring Framework. It eliminates boilerplate configuration so you can build production-ready Spring applications quickly.

### Core Goals

1. **Faster development** — start coding immediately
2. **Zero XML** — Java/annotation config only
3. **Sensible defaults** — works out of the box
4. **Standalone** — embeds a server (`java -jar app.jar`)
5. **Production-ready** — metrics, health, externalized config
6. **No code generation** — no XML/codegen required

### Key Features

| Feature | Description |
|---------|-------------|
| **Auto-Configuration** | Configures beans based on classpath |
| **Starters** | Curated dependency bundles |
| **Embedded Server** | Tomcat/Jetty/Undertow built-in |
| **Actuator** | Production endpoints (health, metrics) |
| **Externalized Config** | Properties, YAML, env vars |
| **DevTools** | Hot reload, live reload |
| **Fat JAR** | Single executable JAR |
| **No `web.xml`** | Servlet 3+ programmatic setup |

### Minimal App

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}

@RestController
class HelloController {
    @GetMapping("/hello")
    public String hello() { return "Hello, Spring Boot!"; }
}
```

Run: `java -jar demo.jar` → serves on port 8080.

---

## 2. Spring Boot vs Spring Framework

| Aspect | Spring Framework | Spring Boot |
|--------|------------------|-------------|
| Purpose | DI, AOP, MVC, data | Simplify Spring setup |
| Config | Explicit (XML/Java) | Auto + overridable |
| Dependencies | Manual | Starters |
| Server | External WAR | Embedded |
| Build | WAR | Fat JAR |
| Boilerplate | High | Minimal |
| Actuator | ❌ | ✅ |
| Opinionated | ❌ | ✅ |
| Versioning | Own | Manages all versions |

**Not a replacement** — Spring Boot **uses** Spring Framework.

---

## 3. @SpringBootApplication

### Meta-Annotation Breakdown

```java
@SpringBootApplication
// is equivalent to:

@SpringBootConfiguration   // = @Configuration
@EnableAutoConfiguration   // enable auto-config
@ComponentScan             // scan base package + subpackages
```

### Attributes

```java
@SpringBootApplication(
    scanBasePackages = {"com.example", "com.shared"},
    scanBasePackageClasses = {DemoApplication.class, SharedConfig.class},
    exclude = {DataSourceAutoConfiguration.class},
    excludeName = {"org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration"}
)
```

### Main Method

```java
public static void main(String[] args) {
    SpringApplication.run(MyApp.class, args);
}
```

### Customizing Startup

```java
public static void main(String[] args) {
    SpringApplication app = new SpringApplication(MyApp.class);
    app.setBannerMode(Banner.Mode.OFF);
    app.setAdditionalProfiles("dev");
    app.setDefaultProperties(Map.of("server.port", "9999"));
    app.setLazyInitialization(true);
    app.setLogStartupInfo(true);
    app.run(args);
}
```

### @SpringBootConfiguration

- A specialization of `@Configuration`
- Only **one** per app recommended
- Detected by tests and DevTools

### Application Events

Order:

1. `ApplicationStartingEvent`
2. `ApplicationEnvironmentPreparedEvent`
3. `ApplicationContextInitializedEvent`
4. `ApplicationPreparedEvent`
5. `ContextRefreshedEvent`
6. `ApplicationStartedEvent`
7. `AvailabilityChangeEvent` (LivenessState.CORRECT)
8. `ApplicationReadyEvent`
9. `AvailabilityChangeEvent` (ReadinessState.ACCEPTING_TRAFFIC)
10. On shutdown: `ContextClosedEvent` → `ApplicationStoppingEvent` → `AvailabilityChangeEvent` (LivenessState.BROKEN) → `ApplicationFailedEvent`

Subscribe:

```java
@Component
public class StartupListener {
    @EventListener
    public void onReady(ApplicationReadyEvent e) {
        log.info("App is ready!");
    }
}
```

---

## 4. Auto-Configuration

### What It Does

Spring Boot inspects the classpath, existing beans, and properties, then **automatically configures beans** you'd otherwise configure manually.

Example: With `spring-boot-starter-web` and a `DataSource` on the classpath:
- Embedded Tomcat starts
- `DispatcherServlet` registered
- `DataSource` configured
- `JdbcTemplate` bean created
- `PlatformTransactionManager` created
- Jackson `ObjectMapper` registered
- Spring MVC infrastructure configured

### How It Works

Since **Spring Boot 3.0**, auto-configurations are listed in:

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

(Spring Boot 2.x used `META-INF/spring.factories` with `org.springframework.boot.autoconfigure.EnableAutoConfiguration` key.)

Each line = fully qualified class of an auto-configuration:

```
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
...
```

### Auto-Config Class Example

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}
```

**Reading:**
- `@AutoConfiguration` — meta-annotated with `@Configuration`
- `@ConditionalOnClass` — only if this class is on classpath
- `@ConditionalOnMissingBean` — only if user hasn't defined one
- `@EnableConfigurationProperties` — bind `spring.datasource.*`

### Ordering

```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@AutoConfiguration(before = JpaAutoConfiguration.class)
```

Or via `@AutoConfigureAfter`, `@AutoConfigureBefore`, `@AutoConfigureOrder`.

### Overriding Auto-Config

**1. Define your own bean** — `@ConditionalOnMissingBean` yields:

```java
@Bean
public DataSource dataSource() { return myCustomDataSource(); }
```

**2. Exclude specific auto-config:**

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
```

Or properties:

```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

**3. Override via properties:**

```properties
server.port=9090
spring.datasource.url=...
```

### Debugging Auto-Config

Run with `--debug`:

```bash
java -jar app.jar --debug
```

Or in properties:

```properties
debug=true
```

Prints:
- **Positive matches** — which configs applied and why
- **Negative matches** — which didn't and why
- **Exclusions**

Also:

```properties
logging.level.org.springframework.boot.autoconfigure=DEBUG
```

### @EnableAutoConfiguration

Rarely used directly. Included via `@SpringBootApplication`. If used separately, must be on a `@Configuration` class.

### Custom Auto-Configuration

1. Create a `@AutoConfiguration` class
2. Add conditions (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc.)
3. Register in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
4. Package as a library

```java
@AutoConfiguration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties props) {
        return new MyService(props.getEndpoint());
    }
}
```

---

## 5. Conditional Annotations

### Common Conditions

| Annotation | Condition |
|------------|-----------|
| `@ConditionalOnClass` | Class present on classpath |
| `@ConditionalOnMissingClass` | Class absent |
| `@ConditionalOnBean` | Bean present in context |
| `@ConditionalOnMissingBean` | Bean absent |
| `@ConditionalOnProperty` | Property matches |
| `@ConditionalOnResource` | Resource exists |
| `@ConditionalOnWebApplication` | Web app |
| `@ConditionalOnNotWebApplication` | Non-web |
| `@ConditionalOnExpression` | SpEL true |
| `@ConditionalOnJava` | Java version matches |
| `@ConditionalOnSingleCandidate` | Exactly one bean |

### Examples

```java
@Bean
@ConditionalOnClass(name = "com.example.Feature")
public Feature feature() { return new Feature(); }

@Bean
@ConditionalOnMissingBean(EmailService.class)
public EmailService defaultEmailService() { return new NoopEmailService(); }

@Bean
@ConditionalOnProperty(
    name = "feature.x.enabled",
    havingValue = "true",
    matchIfMissing = false
)
public XFeature xFeature() { return new XFeature(); }

@Bean
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
public ServletFilter servletFilter() { ... }
```

### @Conditional (custom)

```java
public class OnProdEnvironment implements Condition {
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata md) {
        return Arrays.asList(ctx.getEnvironment().getActiveProfiles()).contains("prod");
    }
}

@Bean
@Conditional(OnProdEnvironment.class)
public Cache distributedCache() { ... }
```

### Condition Ordering

Multiple conditions on a bean are **AND**-ed.

### Common Patterns in Auto-Config

```java
@AutoConfiguration
@ConditionalOnWebApplication(type = SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
@ConditionalOnMissingBean(DispatcherServlet.class)
public class WebMvcAutoConfiguration {
    // ...
}
```

---

## 6. Starters

### What is a Starter?

A **starter** is a POM that aggregates dependencies needed for a feature. You add one starter → all transitive deps arrive → auto-config kicks in.

### Core Starters

| Starter | Provides |
|---------|----------|
| `spring-boot-starter` | Core (Spring, logging, autoconfig) |
| `spring-boot-starter-web` | MVC + Tomcat + Jackson + validation |
| `spring-boot-starter-webflux` | Reactive web |
| `spring-boot-starter-data-jpa` | Hibernate + Spring Data JPA |
| `spring-boot-starter-data-jdbc` | Spring Data JDBC |
| `spring-boot-starter-data-mongodb` | MongoDB |
| `spring-boot-starter-data-redis` | Redis + Lettuce |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-validation` | Hibernate Validator |
| `spring-boot-starter-aop` | Spring AOP + AspectJ |
| `spring-boot-starter-actuator` | Actuator endpoints |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, MockMvc |
| `spring-boot-starter-thymeleaf` | Thymeleaf view |
| `spring-boot-starter-batch` | Spring Batch |
| `spring-boot-starter-amqp` | RabbitMQ |
| `spring-boot-starter-kafka` | Kafka |
| `spring-boot-starter-mail` | JavaMail |
| `spring-boot-starter-quartz` | Quartz Scheduler |
| `spring-boot-starter-cache` | Spring Cache abstraction |
| `spring-boot-starter-logging` | Logback |
| `spring-boot-starter-json` | Jackson |
| `spring-boot-starter-tomcat` | Embedded Tomcat |
| `spring-boot-starter-jetty` | Embedded Jetty |
| `spring-boot-starter-undertow` | Embedded Undertow |
| `spring-boot-starter-parent` | Parent POM (plugin mgmt) |

### Dependency Tree (web starter)

```
spring-boot-starter-web
├── spring-boot-starter
├── spring-boot-starter-json (Jackson)
├── spring-boot-starter-tomcat
├── spring-web
└── spring-webmvc
```

### Creating a Custom Starter

**Convention:**
- `<name>-spring-boot-autoconfigure` — code + auto-config
- `<name>-spring-boot-starter` — empty POM pulling in the above

**Auto-config module POM:**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
    </dependency>
    <!-- library deps -->
</dependencies>
```

**Auto-config class:**

```java
@AutoConfiguration
@ConditionalOnClass(MyClient.class)
@EnableConfigurationProperties(MyClientProperties.class)
public class MyClientAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public MyClient myClient(MyClientProperties props) {
        return new MyClient(props.getEndpoint());
    }
}
```

**Register in imports file:**

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.MyClientAutoConfiguration
```

**Starter POM:**

```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>myclient-spring-boot-autoconfigure</artifactId>
        <version>1.0.0</version>
    </dependency>
</dependencies>
```

Consumers add only the starter.

### Excluding Transitive Deps

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!-- Add Jetty instead -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

---

## 7. Embedded Servers

### Default: Tomcat

Included via `spring-boot-starter-web`. No external installation needed.

### Configuration

```properties
server.port=8080
server.address=0.0.0.0
server.servlet.context-path=/api
server.servlet.session.timeout=30m
server.compression.enabled=true
server.compression.min-response-size=1024
server.http2.enabled=true
server.ssl.enabled=true
server.ssl.key-store=classpath:keystore.p12
server.ssl.key-store-password=secret
server.ssl.key-store-type=PKCS12
server.tomcat.max-threads=200
server.tomcat.min-spare-threads=10
server.tomcat.accept-count=100
server.tomcat.max-connections=8192
server.tomcat.basedir=/tmp/tomcat
server.tomcat.accesslog.enabled=true
```

### Switching Servers

**Remove Tomcat, add Jetty:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

Same for `spring-boot-starter-undertow`.

### Server Selection Table

| Server | Pros | Cons |
|--------|------|------|
| Tomcat | Default, mature, well-known | Slightly slower |
| Jetty | Lightweight, async-friendly | Less default tooling |
| Undertow | Fast, low memory, JBoss | Smaller community |
| Netty (WebFlux) | Reactive, non-blocking | Not for Servlet |

### WAR Deployment (Traditional)

If you must deploy to an external servlet container:

```xml
<packaging>war</packaging>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```

```java
public class ServletInitializer extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(MyApp.class);
    }
}
```

---

## 8. Spring Initializr

### Web UI

**https://start.spring.io**

Options:
- Project: Maven / Gradle
- Language: Java / Kotlin / Groovy
- Spring Boot version
- Group, Artifact, Name, Package
- Packaging: Jar / War
- Java version
- Dependencies (Starters)

Generates a ready-to-import ZIP.

### CLI

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,security,actuator \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.2.0 \
  -d javaVersion=17 \
  -d groupId=com.example \
  -d artifactId=demo \
  -d name=demo \
  -d packageName=com.example.demo \
  -o demo.zip
```

### Via IDE

- IntelliJ IDEA → New Project → Spring Initializr
- STS → New → Spring Starter Project
- VS Code → Spring Initializr extension

### Generated Structure (Maven)

```
demo/
├── mvnw, mvnw.cmd
├── pom.xml
├── .gitignore
└── src/
    ├── main/
    │   ├── java/com/example/demo/
    │   │   └── DemoApplication.java
    │   └── resources/
    │       ├── application.properties
    │       ├── static/
    │       └── templates/
    └── test/
        └── java/com/example/demo/
            └── DemoApplicationTests.java
```

---

## 9. Spring Boot CLI

A command-line tool for rapid prototyping with Groovy.

### Install (SDKMAN)

```bash
sdk install springboot
```

### Run a Script

```groovy
// app.groovy
@RestController
class Hello {
    @GetMapping("/")
    String hi() { "Hello, Boot CLI" }
}
```

```bash
spring run app.groovy
```

### Commands

```bash
spring run app.groovy          # run a script
spring test app.groovy         # run tests
spring shell                   # interactive shell
spring init -d=web demo        # generate project
spring encodepassword secret   # BCrypt encode
```

**Use cases:** Demos, quick POCs, learning. Rare in production.

---

## 10. application.properties vs yml

### Properties

```properties
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
spring.jpa.show-sql=true
```

### YAML

```yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
  jpa:
    show-sql: true
```

### Comparison

| Aspect | Properties | YAML |
|--------|-----------|------|
| Structure | Flat | Hierarchical |
| Readability | OK for few | Better for nested |
| Duplicate keys | Allowed (last wins) | Error |
| Multi-doc | ❌ | ✅ (`---` separator) |
| Profiles | Separate files | Multi-doc supported |
| Lists | Awkward | Natural |
| Tooling | Universal | Requires SnakeYAML |

### Multi-Document YAML

```yaml
# application.yml
spring:
  profiles:
    active: dev

---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8080

---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```

### Lists

```yaml
# YAML
myapp:
  hosts:
    - host1
    - host2
```

```properties
# properties
myapp.hosts[0]=host1
myapp.hosts[1]=host2
```

### Profile-Specific Files

```
application.properties           # base
application-dev.properties       # dev overrides
application-prod.properties      # prod overrides
```

Same for `.yml`.

### Which to Choose?

- **YAML** for complex/nested config
- **Properties** for simple, flat configs
- Don't mix both for the same app (confusing)
- Pick based on team preference

### Special Property Files

| File | Purpose |
|------|---------|
| `application.properties/yml` | Main |
| `application-{profile}.properties/yml` | Per profile |
| `bootstrap.properties/yml` | Spring Cloud Config (pre-Spring Boot 2.4) |
| `spring.factories` | Auto-config (2.x) |
| `AutoConfiguration.imports` | Auto-config (3.x+) |

### Property Placeholders

```properties
app.name=Demo
app.description=${app.name} is awesome
app.random=${random.uuid}
app.int=${random.int(1,100)}
```

### Command-Line Overrides

```bash
java -jar app.jar --server.port=9090 --spring.profiles.active=prod
```

---

## 11. DevTools

### Add Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

### Features

| Feature | Description |
|---------|-------------|
| **Automatic restart** | Restart on classpath change |
| **LiveReload** | Browser auto-refresh (needs extension) |
| **Property defaults** | Disables caching for templates |
| **H2 console auto-config** | When H2 present |
| **Global settings** | `~/.spring-boot-devtools.properties` |
| **Remote debug tunnel** | For remote apps |

### How Restart Works

- Two classloaders: base (deps) + restart (your classes)
- Only your classes reload — fast
- Trigger: classpath change (IDE compile or file save)

### Configuration

```properties
# Disable restart
spring.devtools.restart.enabled=false

# Exclude paths from restart trigger
spring.devtools.restart.exclude=static/**,public/**

# Additional watch paths
spring.devtools.restart.additional-paths=../shared

# Trigger file
spring.devtools.restart.trigger-file=.trigger
```

### Production Safety

DevTools is **disabled** when running a packaged JAR (`java -jar`). Property defaults don't apply. LiveReload server doesn't start.

### Caveats

- Not for production
- IDE must compile automatically
- Some libraries (Spring Security) need re-login on restart
- Doesn't reload `application.properties` in all cases (restart needed)

---

## 12. Versioning & Release Cadence

### Versioning Scheme

- **Major** — Spring Boot 3.x (Spring Framework 6, Jakarta EE 9+)
- **Minor** — 3.1, 3.2 (new features)
- **Patch** — 3.2.1 (bug/security fixes)

### Support Windows

- Each minor version: ~13 months of OSS support
- Extended commercial support available

### Major Milestones

| Version | Released | Highlights |
|---------|----------|-----------|
| 1.0 | 2014 | Initial |
| 1.5 | 2017 | LTS-ish |
| 2.0 | 2018 | WebFlux, Kotlin, Micrometer |
| 2.1–2.3 | 2018–2020 | Refinements |
| 2.4 | 2020 | Config data API, profile groups |
| 2.5 | 2021 | |
| 2.6 | 2021 | Circular refs disabled |
| 2.7 | 2022 | Last 2.x |
| **3.0** | Nov 2022 | Jakarta EE 9, Java 17, AOT, native |
| **3.1** | 2023 | Docker Compose, SSL bundle |
| **3.2** | 2023 | Virtual threads, RestClient |
| **3.3** | 2024 | CDS, observability |
| **3.4** | 2024 | Structured logging |
| **3.5** | 2025 | Extended support |

### Choosing a Version

- **Latest stable** for new projects
- **LTS-like** for conservative teams (e.g., 3.2.x)
- Match Spring Cloud, Spring Security versions
- Always check [start.spring.io](https://start.spring.io) for supported combos

### Dependency Management

Spring Boot's parent POM pins versions of ~300 libraries. Your app declares versions only for those not managed.

```xml
<!-- No version needed -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>

<!-- Override managed version (rare) -->
<properties>
    <jackson.version>2.17.0</jackson.version>
</properties>
```

---

## 13. Migrating from Spring to Spring Boot

### Steps

1. **Assess** — inventory XML configs, dependencies, servlet container
2. **Add Spring Boot Parent POM** (or BOM)
3. **Convert to Starters** — replace explicit deps
4. **Consolidate config** — into `application.properties/yml`
5. **Replace XML with auto-config** — remove boilerplate
6. **Add `@SpringBootApplication`**
7. **Migrate web.xml** — use embedded server or `SpringBootServletInitializer`
8. **Test** — full integration

### Example Migration

**Before (Spring):**

```java
@Configuration
@EnableWebMvc
@ComponentScan("com.example")
public class WebConfig { /* ... */ }

@Configuration
@EnableTransactionManagement
public class DataConfig {
    @Bean public DataSource ds() { /* ... */ }
    @Bean public JdbcTemplate jdbc() { /* ... */ }
    @Bean public PlatformTransactionManager tx() { /* ... */ }
}

public class WebAppInitializer 
      extends AbstractAnnotationConfigDispatcherServletInitializer {
    // ...
}
```

**After (Spring Boot):**

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) { SpringApplication.run(MyApp.class, args); }
}
```

`application.properties`:

```properties
spring.datasource.url=jdbc:...
spring.datasource.username=admin
```

Auto-config provides `DataSource`, `JdbcTemplate`, `PlatformTransactionManager`, `DispatcherServlet`, etc.

### Common Migration Issues

- `javax.*` → `jakarta.*` (Spring Boot 3)
- Servlet container features not available
- Custom `web.xml` mappings
- Legacy XML security config → Java config
- Filter/servlet registrations

---

## 14. Interview Questions

### Q1. What is Spring Boot?

**Answer:** Spring Boot is an opinionated framework built on Spring that simplifies setup via auto-configuration, starter dependencies, embedded servers, and production-ready features. It eliminates XML and boilerplate, letting you focus on business logic.

### Q2. What does @SpringBootApplication do?

**Answer:** It combines `@SpringBootConfiguration` (= `@Configuration`), `@EnableAutoConfiguration`, and `@ComponentScan`. It marks the main class and enables auto-config + component scanning of the base package and subpackages.

### Q3. How does auto-configuration work?

**Answer:** Spring Boot inspects the classpath, existing beans, and properties. Auto-configuration classes (listed in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` in Spring Boot 3) are conditionally applied via `@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc. This produces beans matching the environment. You override any bean by defining your own.

### Q4. How do you disable auto-configuration?

**Answer:** Use `@SpringBootApplication(exclude = XyzAutoConfiguration.class)` or property `spring.autoconfigure.exclude=...`. Or define a bean of the same type (`@ConditionalOnMissingBean` yields).

### Q5. What are starters?

**Answer:** Starters are dependency descriptors (POMs) bundling libraries for a feature. E.g., `spring-boot-starter-web` pulls in Tomcat, Spring MVC, Jackson, validation. One dependency → all needed libs → auto-config triggers.

### Q6. Difference between spring-boot-starter-web and spring-boot-starter-webflux?

**Answer:** `web` is Servlet-based, blocking, uses Tomcat. `webflux` is reactive, non-blocking, uses Netty (or other reactive server). Choose `web` for traditional APIs; `webflux` for high-concurrency reactive apps.

### Q7. Embedded server vs external?

**Answer:** Spring Boot embeds Tomcat/Jetty/Undertow, so the app runs as a standalone JAR (`java -jar app.jar`). This simplifies deployment (no external container). External deployment is still possible with `packaging=war` + `SpringBootServletInitializer`.

### Q8. How to change the server port?

**Answer:** `server.port=9090` in `application.properties`, or `--server.port=9090` on command line, or via env var `SERVER_PORT=9090`.

### Q9. Difference between application.properties and application.yml?

**Answer:** `.properties` is flat `key=value`; `.yml` is hierarchical. YAML handles nested data and lists more naturally and supports multi-document (multi-profile) files. Both load identically; choose one style.

### Q10. How do profiles work in Spring Boot?

**Answer:** `spring.profiles.active=dev` activates a profile. Profile-specific files (`application-dev.properties`) are loaded and override base. Beans can be conditional via `@Profile("dev")` or `@ConditionalOnProperty`. Spring Boot 2.4+ supports profile groups.

### Q11. How do you override a property?

**Answer:** Precedence (high to low, mostly):
1. DevTools global settings
2. `@TestPropertySource`
3. Command-line args
4. `SPRING_APPLICATION_JSON`
5. Servlet config/context params
6. JNDI
7. Java system properties
8. OS environment variables
9. `application-{profile}.properties`
10. `application.properties`
11. `@PropertySource`
12. Default properties

### Q12. What is DevTools?

**Answer:** Development-only helper providing automatic restart on classpath changes, LiveReload, sensible development-time defaults (disabled template caching), and H2 console auto-config. Disabled in production (`java -jar`).

### Q13. How to create a fat/executable JAR?

**Answer:** Use `spring-boot-maven-plugin` (`repackage` goal). The plugin bundles your classes + dependencies into a single JAR with a launcher that creates a classloader hierarchy. `java -jar app.jar` runs it.

### Q14. What is the Spring Initializr?

**Answer:** A project generator (start.spring.io) that produces a skeleton Spring Boot project with selected dependencies. Accessible via web, IDE, or command line.

### Q15. How to view what auto-config applied?

**Answer:** Run with `--debug` or set `debug=true`; Spring Boot logs a **Condition Evaluation Report** listing positive/negative matches and reasons. Actuator's `/actuator/conditions` endpoint also exposes it.

### Q16. How to define custom auto-config?

**Answer:**
1. Create `@AutoConfiguration` class with conditions
2. Add to `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 3) or `spring.factories` (2.x)
3. Package as library/starter

### Q17. Difference between @Configuration and @SpringBootConfiguration?

**Answer:** `@SpringBootConfiguration` is a specialization of `@Configuration`, used to mark the main application class. There should be only one per application; tests find it for `@SpringBootTest`.

### Q18. What happens if two auto-configs define the same bean?

**Answer:** Spring Boot applies ordering (`@AutoConfigureBefore/After`) and `@ConditionalOnMissingBean`. If user's bean exists, auto-config backs off. If both are auto-configs and both apply, conflicts cause startup failure — resolved by ordering or explicit exclusion.

### Q19. How to change logging level?

**Answer:** `logging.level.com.example=DEBUG` in `application.properties`, or `--logging.level.com.example=DEBUG`, or Actuator's `/actuator/loggers/com.example` POST endpoint.

### Q20. How does Spring Boot work without web.xml?

**Answer:** Spring Boot uses Servlet 3+ programmatic registration (or embedded server). `DispatcherServlet` and filters are registered via `ServletContextInitializer` beans by auto-config. No `web.xml` needed.

### Q21. Can you use Spring Boot with WAR packaging?

**Answer:** Yes — set `<packaging>war</packaging>`, mark `spring-boot-starter-tomcat` as `provided`, and extend `SpringBootServletInitializer` to configure the app for the container. Rarely needed today.

### Q22. Difference between a starter and a library?

**Answer:** A starter is a **dependency-only POM** that pulls in libraries + auto-config. It contains no code. A library is actual code. Starters make setup easy; libraries do the work.

### Q23. What is `@EnableAutoConfiguration`?

**Answer:** A meta-annotation that enables Spring Boot's auto-configuration mechanism. Included by `@SpringBootApplication`. It imports `AutoConfigurationImportSelector` which reads auto-configuration candidates.

### Q24. How do you exclude a starter's transitive dependency?

**Answer:** Use `<exclusions>` in Maven or `exclude` in Gradle. Example: exclude `spring-boot-starter-tomcat` from `spring-boot-starter-web`, then add `spring-boot-starter-jetty`.

### Q25. How does Spring Boot know which auto-config to apply?

**Answer:** It reads a curated list from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 3) or `META-INF/spring.factories` (2.x). Each candidate is filtered by conditions (classpath, beans, properties, etc.).

---

## 15. Cheat Sheet

### Main Class

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

### Common Properties

```properties
server.port=8080
server.servlet.context-path=/
spring.application.name=my-app
spring.profiles.active=dev
logging.level.root=INFO
logging.level.com.example=DEBUG
debug=true
```

### Starters (Most Used)

```
web, webflux, data-jpa, data-jdbc, security, validation,
aop, actuator, test, thymeleaf, batch, amqp, kafka,
redis, mongodb, quartz, cache, mail
```

### Conditional Annotations

```
@ConditionalOnClass
@ConditionalOnMissingClass
@ConditionalOnBean
@ConditionalOnMissingBean
@ConditionalOnProperty
@ConditionalOnResource
@ConditionalOnWebApplication
@ConditionalOnNotWebApplication
@ConditionalOnExpression
```

### Auto-Config Files

```
Spring Boot 2.x: META-INF/spring.factories
Spring Boot 3.x: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

### Building Fat JAR

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

```bash
mvn clean package
java -jar target/app.jar
```

### Override Auto-Config

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
```

```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### Debug Auto-Config

```properties
debug=true
```

```properties
logging.level.org.springframework.boot.autoconfigure=DEBUG
```

### Actuator Endpoints (with actuator starter)

```
/actuator/health
/actuator/info
/actuator/metrics
/actuator/env
/actuator/beans
/actuator/mappings
/actuator/conditions
/actuator/loggers
```

### Cross-References

- **Previous:** `05_Spring_Transaction_Management.md`
- **Next:** `07_Spring_Boot_Starters.md`
- **Related:** `08_Spring_Boot_Configuration.md`, `09_Spring_Boot_Actuator.md`
- **Interview:** `24_Spring_Interview_Questions.md` (§Spring Boot)

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [05_Spring_Transaction_Management.md](./05_Spring_Transaction_Management.md)
- **Next →:** [07_Spring_Boot_Starters.md](./07_Spring_Boot_Starters.md)
- **Related:** [01_Spring_Framework_Core.md](./01_Spring_Framework_Core.md), [08_Spring_Boot_Configuration.md](./08_Spring_Boot_Configuration.md), [16_Spring_Testing.md](./16_Spring_Testing.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
