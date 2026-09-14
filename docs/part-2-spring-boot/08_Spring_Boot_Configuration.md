# Spring Boot Configuration & Profiles

> **File:** `08_Spring_Boot_Configuration.md`
> **Part:** 2 — Spring Boot
> **Prerequisites:** `06_Spring_Boot_Fundamentals.md`
> **Estimated Study Time:** 6–8 hours

---

## Table of Contents

1. [Configuration Overview](#1-configuration-overview)
2. [application.properties Syntax](#2-applicationproperties-syntax)
3. [application.yml Syntax](#3-applicationyml-syntax)
4. [Property Placeholders & Random Values](#4-property-placeholders--random-values)
5. [@Value Annotation](#5-value-annotation)
6. [@ConfigurationProperties](#6-configurationproperties)
7. [Validation of Configuration Properties](#7-validation-of-configuration-properties)
8. [Profile-Specific Configuration](#8-profile-specific-configuration)
9. [Activating Profiles](#9-activating-profiles)
10. [Profile Groups](#10-profile-groups)
11. [Externalized Configuration](#11-externalized-configuration)
12. [Property Precedence Order](#12-property-precedence-order)
13. [@PropertySource](#13-propertysource)
14. [Config Import](#14-config-import)
15. [Encrypting Properties (Jasypt)](#15-encrypting-properties-jasypt)
16. [Interview Questions](#16-interview-questions)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. Configuration Overview

Spring Boot supports multiple configuration sources, unified via the `Environment` abstraction:

- `application.properties` / `application.yml`
- Profile-specific variants
- Environment variables
- System properties
- Command-line arguments
- `@PropertySource`
- `spring.config.import`

### Design Principles

1. **Convention over configuration** — sensible defaults
2. **Externalize everything** — 12-factor app
3. **Override easily** — layered precedence
4. **Type-safe binding** — `@ConfigurationProperties`

### Where Files Live

| Location | Precedence |
|----------|-----------|
| `classpath:/` (jar root) | Lowest |
| `classpath:/config/` | Higher |
| `file:./` (current dir) | Higher |
| `file:./config/` | Higher |
| `file:./config/*/` | Higher |
| Command-line / env | Highest |

---

## 2. application.properties Syntax

### Basic Format

```properties
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
logging.level.com.example=DEBUG
```

### Comments

```properties
# This is a comment
! This is also a comment
```

### Line Continuation

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb?\
useSSL=false&serverTimezone=UTC
```

### Lists

```properties
myapp.hosts[0]=host1
myapp.hosts[1]=host2
myapp.hosts[2]=host3

# Or comma-separated
myapp.hosts=host1,host2,host3
```

### Maps

```properties
myapp.endpoints.user=http://user.api
myapp.endpoints.order=http://order.api
```

Bound to `Map<String, String> endpoints` or `Map<String, URL>`.

### Special Characters

Escape `:` `=` `\` with backslash:

```properties
my.key=value\:with\:colons
```

### Multi-line / Complex Values

```properties
app.description=Line one\n\
Line two\n\
Line three
```

---

## 3. application.yml Syntax

### Basic Format

```yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
logging:
  level:
    com.example: DEBUG
```

### Lists

```yaml
myapp:
  hosts:
    - host1
    - host2
    - host3

# or inline
myapp:
  hosts: [host1, host2, host3]
```

### Maps

```yaml
myapp:
  endpoints:
    user: http://user.api
    order: http://order.api
```

### Multi-Document (Profiles)

```yaml
# Base
spring:
  application:
    name: myapp

---
# Dev profile
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8080

---
# Prod profile
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```

### Anchors & Aliases (DRY)

```yaml
defaults: &defaults
  timeout: 5000
  retries: 3

service-a:
  <<: *defaults
  url: http://a.api

service-b:
  <<: *defaults
  url: http://b.api
```

### Data Types

```yaml
app:
  enabled: true
  count: 100
  ratio: 0.85
  name: "Hello"
  date: 2025-01-01
  duration: 30s          # → Duration
  size: 10MB             # → DataSize
  list: [a, b, c]
  map: { k1: v1, k2: v2 }
```

Spring Boot converts `30s`, `5m`, `10MB`, `1GB` automatically.

### YAML vs Properties Decision

| Feature | Properties | YAML |
|---------|-----------|------|
| Hierarchical | ❌ | ✅ |
| Multi-doc | ❌ | ✅ |
| Lists | Awkward | Natural |
| Dup keys | Last wins | Error |
| Tooling | Universal | SnakeYAML needed |
| Profiles inline | ❌ | ✅ |

---

## 4. Property Placeholders & Random Values

### Placeholders

```properties
app.name=MyApp
app.description=${app.name} is awesome
app.home=${user.home}/.myapp
```

### Random Values

```properties
app.uuid=${random.uuid}
app.int=${random.int}
app.intRange=${random.int(1,100)}
app.long=${random.long}
app.longRange=${random.long(1000,9999)}
app.bytes=${random.bytes(16)}
```

### Default Values

```properties
app.port=${SERVER_PORT:8080}
app.name=${APP_NAME:MyApp}
```

If property missing → use default after `:`.

### Nested Defaults

```properties
app.url=${APP_URL:http://${APP_HOST:localhost}:${APP_PORT:8080}}
```

### Referencing Other Configs

```properties
spring.datasource.url=jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
```

---

## 5. @Value Annotation

### Basic Usage

```java
@Component
public class MyBean {
    @Value("${app.name}")
    private String name;
    
    @Value("${app.timeout:5000}")   // default
    private int timeout;
    
    @Value("${app.enabled:true}")
    private boolean enabled;
}
```

### SpEL in @Value

```java
@Value("#{systemProperties['user.dir']}")
private String workingDir;

@Value("#{2 + 3}")
private int sum;

@Value("#{myProperties.list}")
private List<String> list;
```

### List Injection

```properties
app.hosts=host1,host2,host3
```

```java
@Value("${app.hosts}")
private List<String> hosts;   // auto-splits by comma

@Value("${app.hosts}")
private String[] hostArray;   // comma-split
```

### Map Injection

```properties
app.endpoints.user=http://user.api
app.endpoints.order=http://order.api
```

```java
@Value("#{${app.endpoints}}")
private Map<String, String> endpoints;
```

### Constructor Injection

```java
public MyBean(@Value("${app.name}") String name) {
    this.name = name;
}
```

### Drawbacks of @Value

- No type-safe binding
- No relaxed binding (kebab-case only)
- No validation
- Not grouped
- Harder to refactor
- Scattered across classes

### When to Prefer @ConfigurationProperties

Use `@Value` for:
- Simple, single values
- SpEL expressions
- Ad-hoc overrides

Use `@ConfigurationProperties` for:
- Grouped config
- Type-safe
- Validation
- IDE support

---

## 6. @ConfigurationProperties

Type-safe binding of a group of properties to a POJO.

### Basic

```java
@ConfigurationProperties(prefix = "app.mail")
@Component
public class MailProperties {
    private String host;
    private int port = 25;
    private String username;
    private String password;
    private boolean ssl = false;
    // getters/setters
}
```

`application.yml`:

```yaml
app:
  mail:
    host: smtp.example.com
    port: 587
    username: noreply@example.com
    password: secret
    ssl: true
```

### Enabling via @EnableConfigurationProperties

```java
@Configuration
@EnableConfigurationProperties(MailProperties.class)
public class MailConfig { }
```

### Or @ConfigurationPropertiesScan (Boot 2.2+)

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class App { }
```

Scans for `@ConfigurationProperties` classes without `@Component`.

### Nested Properties

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private Mail mail = new Mail();
    private Database database = new Database();
    private List<String> features = new ArrayList<>();
    private Map<String, Duration> timeouts = new HashMap<>();
    
    public static class Mail {
        private String host;
        private int port = 25;
        // getters/setters
    }
    
    public static class Database {
        private String url;
        private String username;
        // getters/setters
    }
    // getters/setters
}
```

```yaml
app:
  name: MyApp
  mail:
    host: smtp.example.com
    port: 587
  database:
    url: jdbc:...
    username: admin
  features:
    - auth
    - metrics
  timeouts:
    cache: 30s
    api: 5s
```

### Constructor Binding (Immutable)

```java
@ConfigurationProperties(prefix = "app.mail")
@ConstructorBinding
public class MailProperties {
    private final String host;
    private final int port;
    
    public MailProperties(String host, @DefaultValue("25") int port) {
        this.host = host;
        this.port = port;
    }
    // getters only
}
```

Since Boot 3.0, constructor binding is enabled automatically for records and single-constructor classes (no `@ConstructorBinding` needed unless multiple constructors).

### Records (Boot 2.6+)

```java
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(String host, @DefaultValue("25") int port, boolean ssl) { }
```

Immutable, concise, ideal for config.

### Relaxed Binding

Multiple property name styles bind to the same field:

| Form | Example |
|------|---------|
| Kebab | `app.mail.host-name` |
| camelCase | `app.mail.hostName` |
| Underscore | `app.mail.host_name` |
| Uppercase env | `APP_MAIL_HOSTNAME` |

Bind to field `hostName` in all cases.

### Using Properties

```java
@Service
public class MailService {
    private final MailProperties props;
    
    public MailService(MailProperties props) { this.props = props; }
    
    public void send() {
        // use props.getHost(), props.getPort()
    }
}
```

### IDE Support

Add `spring-boot-configuration-processor`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

Generates `spring-configuration-metadata.json` → IDE autocomplete for properties.

### @ConfigurationProperties vs @Value

| Feature | @ConfigurationProperties | @Value |
|---------|-------------------------|--------|
| Type-safe binding | ✅ | Partial |
| Grouped | ✅ | ❌ |
| Relaxed binding | ✅ | ❌ |
| SpEL | ❌ (needs workaround) | ✅ |
| Validation | ✅ | ❌ |
| Constructor binding | ✅ | Partial |
| Immutability | ✅ | ❌ |
| IDE support | ✅ | Partial |
| Default values | Field default | `${x:default}` |

**Rule:** Grouped/structured config → `@ConfigurationProperties`. Simple single values → `@Value`.

---

## 7. Validation of Configuration Properties

### Add JSR-303 Annotations

```java
@ConfigurationProperties(prefix = "app.mail")
@Validated
public class MailProperties {
    @NotBlank
    private String host;
    
    @Min(1) @Max(65535)
    private int port = 25;
    
    @Email
    private String username;
    
    @Size(min = 8)
    private String password;
    // getters/setters
}
```

Add dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### Failure Behavior

Invalid properties → startup fails with `BindValidationException`.

### Nested Validation

```java
@Valid
private Database database;
```

Nested object must have `@Valid` to trigger its validators.

### Custom Validators

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UrlValidator.class)
public @interface ValidUrl {
    String message() default "Invalid URL";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class UrlValidator implements ConstraintValidator<ValidUrl, String> {
    @Override
    public boolean isValid(String url, ConstraintValidatorContext ctx) {
        if (url == null) return true;
        try { new URL(url); return true; }
        catch (MalformedURLException e) { return false; }
    }
}
```

---

## 8. Profile-Specific Configuration

### File-Based

```
src/main/resources/
├── application.properties               # base
├── application-dev.properties           # dev
├── application-prod.properties          # prod
├── application-test.properties          # test
```

### Base

```properties
app.name=MyApp
spring.datasource.url=jdbc:h2:mem:test
logging.level.root=INFO
```

### Dev Override

```properties
logging.level.root=DEBUG
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb_dev
spring.jpa.show-sql=true
```

### Prod Override

```properties
logging.level.root=WARN
spring.datasource.url=jdbc:postgresql://prod-host:5432/mydb
spring.jpa.hibernate.ddl-auto=validate
spring.datasource.hikari.maximum-pool-size=50
```

When `dev` active: base + dev overrides. Same for prod.

### Multi-Profile Files

```
application-dev,metrics.properties
```

Both must be active for this file to load.

### YAML Multi-Doc

```yaml
# application.yml
app:
  name: MyApp

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

### Bean-Level Profiles

```java
@Component
@Profile("dev")
public class DevEmailService implements EmailService { }

@Component
@Profile("prod")
public class ProdEmailService implements EmailService { }
```

### Profile Expressions (Boot 2.4+)

```java
@Profile("dev & !cloud")
@Profile("dev | staging")
@Profile("(dev | test) & !ci")
```

### Profile-Specific Beans via @Profile on @Bean

```java
@Configuration
public class DataConfig {
    
    @Bean
    @Profile("dev")
    public DataSource devDs() { return new H2DataSource(); }
    
    @Bean
    @Profile("prod")
    public DataSource prodDs() { return new PostgresDataSource(); }
}
```

### Default Profile

When no profile is active, `default` is used.

```properties
spring.profiles.default=local
```

Beans with `@Profile("default")` load.

---

## 9. Activating Profiles

### 1. Property File

```properties
spring.profiles.active=dev
```

### 2. Command Line

```bash
java -jar app.jar --spring.profiles.active=dev
```

### 3. Environment Variable

```bash
export SPRING_PROFILES_ACTIVE=dev
```

Or:

```bash
SPRING_PROFILES_ACTIVE=dev,metrics java -jar app.jar
```

### 4. JVM System Property

```bash
java -Dspring.profiles.active=dev -jar app.jar
```

### 5. Programmatic

```java
SpringApplication app = new SpringApplication(MyApp.class);
app.setAdditionalProfiles("dev");
app.run(args);
```

Or:

```java
ConfigurableEnvironment env = app.run().getEnvironment();
env.setActiveProfiles("dev", "metrics");
```

### 6. In Tests

```java
@SpringBootTest
@ActiveProfiles("test")
class MyTest { }
```

### Multiple Profiles

```properties
spring.profiles.active=dev,metrics
```

Beans matching **any** active profile load.

### Include Profiles

```properties
spring.profiles.include=common
```

Adds `common` even if not listed in `active`. Only applies to non-profile-specific configs (Boot 2.4+).

### Overriding Active Profiles

If set on command line, it **replaces** `spring.profiles.active` from properties.

### Precedence

1. Command line `--spring.profiles.active`
2. Environment variable `SPRING_PROFILES_ACTIVE`
3. JVM system property `-Dspring.profiles.active`
4. `application.properties` `spring.profiles.active`
5. Programmatic `setAdditionalProfiles`

---

## 10. Profile Groups

Boot 2.4+ feature to activate multiple profiles via a single alias.

### Definition

```properties
spring.profiles.group.production=prod-db,prod-mq,prod-monitoring
spring.profiles.group.dev=dev-db,dev-mail
```

Now:

```properties
spring.profiles.active=production
```

Activates all three: `prod-db`, `prod-mq`, `prod-monitoring`.

### YAML

```yaml
spring:
  profiles:
    group:
      production:
        - prod-db
        - prod-mq
        - prod-monitoring
      dev:
        - dev-db
        - dev-mail
```

### Use Cases

- Environment aliases (`prod` → `db,mq,logging,metrics`)
- Feature toggles across profiles
- Cleaner activation on deploy

### Nested Groups (avoid)

Groups can reference other groups but keep it shallow to avoid confusion.

---

## 11. Externalized Configuration

### Sources (in addition to classpath files)

- Command-line args (`--key=value`)
- OS environment variables (`MYAPP_KEY=value`)
- JVM system properties (`-Dmyapp.key=value`)
- External files (`--spring.config.location=/etc/myapp/`)
- Config server (Spring Cloud Config)
- `SPRING_APPLICATION_JSON` env var
- Kubernetes ConfigMaps / Secrets (mounted or via Spring Cloud K8s)

### External Files

```bash
# Point to a directory
java -jar app.jar --spring.config.location=file:/etc/myapp/

# Or additional locations (higher precedence)
java -jar app.jar --spring.config.additional-location=/opt/myapp/

# Explicit file
java -jar app.jar --spring.config.location=file:/etc/myapp/application-prod.properties
```

Multiple locations:

```bash
--spring.config.location=classpath:/default/,file:/etc/myapp/
```

### Optional Locations

```bash
--spring.config.location=optional:file:/etc/myapp/
```

Missing files don't cause failure.

### Environment Variables

```bash
export SPRING_DATASOURCE_URL=jdbc:postgresql://prod:5432/mydb
export SPRING_PROFILES_ACTIVE=prod
```

Naming: uppercase + underscores, matches `spring.datasource.url`.

### SPRING_APPLICATION_JSON

```bash
export SPRING_APPLICATION_JSON='{"spring":{"datasource":{"url":"jdbc:..."}}}'
```

Great for CI/CD pipelines.

### Command-Line Args

```bash
java -jar app.jar --server.port=9090 --spring.profiles.active=prod
```

These are highest precedence (after DevTools/TestPropertySource).

---

## 12. Property Precedence Order

Spring Boot 2.4+ uses a **layered** approach. Later wins:

| Priority | Source |
|----------|--------|
| **Lowest** | Default properties (`SpringApplication.setDefaultProperties`) |
| | `@PropertySource` on `@Configuration` |
| | Config data (application.properties/yml from classpath) |
| | Config data (from `file:./config/`) |
| | Config data (from `file:./`) |
| | Profile-specific config data |
| | Random values (`${random.*}`) |
| | OS environment variables |
| | Java system properties (`-D`) |
| | JNDI (`java:comp/env`) |
| | `ServletConfig` init params |
| | `ServletContext` init params |
| | `SPRING_APPLICATION_JSON` |
| | Command-line args |
| | `@TestPropertySource` |
| **Highest** | `@DynamicPropertySource` (tests) |
| | DevTools global settings (`~/.config/spring-boot/spring-boot-devtools.properties`) |

### Within a Level

- Profile-specific files beat generic files
- Last-loaded wins within same source

### Precedence Mnemonic

> "Command line beats env beats system beats file beats default."

### Practical Example

If `server.port` is set in:
- `application.properties`: 8080
- `application-prod.properties`: 80
- Env var `SERVER_PORT=9090`
- Command-line `--server.port=7070`

With prod active → **7070** wins.

### Overriding Managed Properties

To override a Boot-managed dependency version:

```xml
<properties>
    <jackson.version>2.17.0</jackson.version>
</properties>
```

Check the Boot BOM for the property name.

---

## 13. @PropertySource

Load properties from a custom file.

### Basic

```java
@Configuration
@PropertySource("classpath:custom.properties")
public class Config {
    @Value("${custom.prop}")
    private String prop;
}
```

### Multiple Files

```java
@PropertySources({
    @PropertySource("classpath:db.properties"),
    @PropertySource("classpath:cache.properties")
})
```

### File System

```java
@PropertySource("file:/etc/myapp/config.properties")
```

### With Profile

```java
@Configuration
@PropertySource("classpath:dev.properties")
@Profile("dev")
public class DevConfig { }
```

### Ignoring Missing Files (Boot 2.4+)

```java
@PropertySource(value = "classpath:missing.properties", ignoreResourceNotFound = true)
```

⚠️ **Note:** `@PropertySource` does **not** support YAML. Use `application.yml` or write a custom `PropertySourceFactory`.

### Custom YAML PropertySource

```java
public class YamlPropertySourceFactory implements PropertySourceFactory {
    @Override
    public PropertySource<?> createPropertySource(String name, 
            EncodedResource resource) throws IOException {
        YamlPropertySourceLoader loader = new YamlPropertySourceLoader();
        List<PropertySource<?>> sources = 
            loader.load(name, resource.getResource());
        return sources.get(0);
    }
}

@PropertySource(value = "classpath:custom.yml", 
                factory = YamlPropertySourceFactory.class)
```

---

## 14. Config Import

Boot 2.4+ feature to import configs from anywhere.

### Import in application.properties

```properties
spring.config.import=classpath:extra.properties
spring.config.import=file:/etc/myapp/secrets.properties
spring.config.import=optional:file:/etc/missing.properties
spring.config.import=configtree:/etc/secrets/
```

### Config Tree

For Kubernetes/Docker secrets mounted as files:

```
/etc/secrets/
├── db-password
└── api-key
```

```properties
spring.config.import=configtree:/etc/secrets/
```

Each file becomes a property: `db.password`, `api.key`.

### Config Server (Spring Cloud)

```properties
spring.config.import=configserver:http://config-server:8888
```

### Profiles + Import

```properties
spring.config.import=classpath:extra-${spring.profiles.active}.properties
```

### Precedence

Imports are **lower precedence** than the importing file. So `application.properties` overrides imported values if both define the same key.

---

## 15. Encrypting Properties (Jasypt)

### Add Dependency

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
```

### Encrypt a Value

```bash
java -cp jasypt-1.9.3.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI \
  input="mySecretPassword" \
  password="encryptionKey" \
  algorithm="PBEWithMD5AndDES"
```

Or programmatically:

```java
StandardPBEStringEncryptor enc = new StandardPBEStringEncryptor();
enc.setPassword("encryptionKey");
String cipher = enc.encrypt("mySecretPassword");
```

### Use in Properties

```properties
spring.datasource.password=ENC(AbCdEf123...)
```

### Provide Encryption Key

Via env var:

```bash
export JASYPT_ENCRYPTOR_PASSWORD=encryptionKey
java -jar app.jar
```

Via command line:

```bash
java -jar app.jar --jasypt.encryptor.password=encryptionKey
```

### Configuration

```properties
jasypt.encryptor.algorithm=PBEWithMD5AndDES
jasypt.encryptor.iv-generator-classname=org.jasypt.iv.NoIvGenerator
jasypt.encryptor.property.prefix=ENC(
jasypt.encryptor.property.suffix=)
```

### Best Practices

- **Never** store the encryption key in the same file as encrypted values
- Use OS environment variable, Vault, or AWS Secrets Manager
- Rotate keys periodically
- Consider Spring Cloud Vault instead for production

---

## 16. Interview Questions

### Q1. How does Spring Boot externalize configuration?

**Answer:** Via the `Environment` abstraction, which aggregates multiple property sources in a layered order: defaults → `application.properties` → profile-specific files → env vars → system properties → command-line args. Later sources override earlier ones.

### Q2. application.properties vs application.yml?

**Answer:** `.properties` is flat key=value. `.yml` is hierarchical YAML with multi-doc support. Both load identically; choose based on team preference and structure complexity.

### Q3. Difference between @Value and @ConfigurationProperties?

**Answer:** `@Value` injects a single property with SpEL support but no type-safe binding, relaxed binding, or validation. `@ConfigurationProperties` binds a group of properties to a POJO with type safety, validation, and IDE support. Use `@ConfigurationProperties` for grouped config.

### Q4. How do you validate configuration properties?

**Answer:** Annotate the `@ConfigurationProperties` class with `@Validated` and add JSR-303 constraints (`@NotBlank`, `@Min`, `@Email`, etc.) to fields. Invalid properties fail application startup with `BindValidationException`.

### Q5. What is relaxed binding?

**Answer:** A feature of `@ConfigurationProperties` that binds property names in multiple forms (kebab-case, camelCase, underscore, uppercase env vars) to the same Java field. E.g., `my-app.max-size`, `myApp.maxSize`, `MY_APP_MAX_SIZE` all map to `maxSize`.

### Q6. How do you activate a profile?

**Answer:** Via `spring.profiles.active=dev` in properties, `--spring.profiles.active=dev` on command line, `SPRING_PROFILES_ACTIVE=dev` env var, `-Dspring.profiles.active=dev` JVM arg, or `@ActiveProfiles` in tests.

### Q7. What are profile groups?

**Answer:** A Boot 2.4+ feature to activate multiple profiles via a single alias. Define `spring.profiles.group.production=prod-db,prod-mq`; activating `production` activates all listed profiles.

### Q8. Explain the property precedence order.

**Answer (high to low):** DevTools global > `@TestPropertySource` > `@DynamicPropertySource` > command-line args > `SPRING_APPLICATION_JSON` > servlet params > JNDI > system properties > OS env vars > profile-specific files > base `application.properties` > `@PropertySource` > defaults.

### Q9. What is @ConfigurationPropertiesScan?

**Answer:** A Boot 2.2+ annotation that scans for `@ConfigurationProperties` classes without `@Component`, avoiding need to list them in `@EnableConfigurationProperties`.

### Q10. Can you bind constructor parameters in @ConfigurationProperties?

**Answer:** Yes, with constructor binding. Since Boot 2.6, records work natively; since Boot 3.0, single-constructor classes bind automatically (no `@ConstructorBinding` needed). Records are the modern idiomatic choice.

### Q11. How do you encrypt sensitive properties?

**Answer:** Use Jasypt (`jasypt-spring-boot-starter`), Spring Cloud Vault, AWS Secrets Manager, or Kubernetes Secrets. Encrypt values with `ENC(...)`, provide the key via env var. Never commit the encryption key.

### Q12. What is `spring.config.import`?

**Answer:** A Boot 2.4+ mechanism to import config from additional locations (files, config tree, Config Server). Supports `optional:` prefix and `configtree:` scheme for directory-mounted files.

### Q13. How do you override a Boot-managed dependency version?

**Answer:** Add a property to your POM using the Boot BOM's property name (e.g., `<jackson.version>2.17.0</jackson.version>`). Check Spring Boot's parent POM for names. Rarely needed.

### Q14. What is `@PropertySource`?

**Answer:** Loads properties from a specific file into the environment. Does **not** support YAML by default — requires a custom `PropertySourceFactory`.

### Q15. Difference between `spring.profiles.active` and `spring.profiles.include`?

**Answer:** `active` **sets** the active profiles (replaces any programmatic profiles). `include` **adds** profiles, merged with those set by active. Since Boot 2.4, `include` should be used in non-profile-specific files only.

### Q16. How do you pass configuration to a Spring Boot app in Kubernetes?

**Answer:** Via ConfigMaps (mounted as files or env vars), Secrets, or Spring Cloud Kubernetes. Common pattern: `spring.config.import=configtree:/etc/config/` for mounted config.

### Q17. What is the default value syntax in @Value?

**Answer:** `${property.name:defaultValue}` — the value after `:` is used if the property is missing.

### Q18. How do you bind a Map in @ConfigurationProperties?

**Answer:** Declare a `Map<String, T>` field. Properties like `app.endpoints.user=...`, `app.endpoints.order=...` populate keys `user`, `order`. Keys can be quoted for special chars.

### Q19. What is `spring.config.location` vs `spring.config.additional-location`?

**Answer:** `location` **replaces** default locations. `additional-location` **adds** to them. Use `location` when you know exactly where config lives; `additional-location` to extend defaults.

### Q20. How do you know which properties are available?

**Answer:** Add `spring-boot-configuration-processor` for IDE autocomplete. Or check `spring-configuration-metadata.json` in the Boot jar. Or use `application.yml` samples in official docs.

---

## 17. Cheat Sheet

### Property Sources (Highest Wins)

```
1. DevTools global settings
2. @TestPropertySource / @DynamicPropertySource
3. Command-line args (--key=value)
4. SPRING_APPLICATION_JSON
5. ServletConfig/ServletContext init params
6. JNDI
7. Java system properties (-Dkey=value)
8. OS environment variables
9. Profile-specific application-{profile}.properties/yml
10. application.properties/yml
11. @PropertySource
12. Default properties
```

### Common Property Files

```
application.properties
application.yml
application-dev.properties
application-prod.properties
application-{profile}.yml
```

### Environment Variable Naming

```
spring.datasource.url  →  SPRING_DATASOURCE_URL
app.mail.host          →  APP_MAIL_HOST
```

### @ConfigurationProperties

```java
@ConfigurationProperties(prefix = "app.mail")
@Component  // or @EnableConfigurationProperties / @ConfigurationPropertiesScan
@Validated
public class MailProperties {
    @NotBlank private String host;
    @Min(1) private int port = 25;
    private boolean ssl = false;
    // getters/setters
}
```

### Record Style

```java
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(String host, @DefaultValue("25") int port) { }
```

### Activate Profiles

```bash
# Properties
spring.profiles.active=dev,metrics

# Env
export SPRING_PROFILES_ACTIVE=dev

# Command line
java -jar app.jar --spring.profiles.active=dev

# Test
@ActiveProfiles("test")
```

### Profile Groups

```properties
spring.profiles.group.production=prod-db,prod-mq
```

### Import Config

```properties
spring.config.import=classpath:extra.properties
spring.config.import=optional:file:/etc/secret.properties
spring.config.import=configtree:/etc/secrets/
```

### Externalized Config

```bash
java -jar app.jar \
  --spring.config.location=file:/etc/myapp/ \
  --spring.config.additional-location=/opt/override/ \
  --spring.profiles.active=prod \
  --server.port=9090
```

### Enable Metadata

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

### Jasypt

```
spring.datasource.password=ENC(...)
JASYPT_ENCRYPTOR_PASSWORD=secretKey java -jar app.jar
```

### Debug

```properties
debug=true
logging.level.org.springframework.boot.context.config=TRACE
```

### Cross-References

- **Previous:** `07_Spring_Boot_Starters.md`
- **Next:** `09_Spring_Boot_Actuator.md`
- **Related:** `06_Spring_Boot_Fundamentals.md`, `19_Spring_Microservices_Cloud.md` (Config Server)
- **Interview:** `24_Spring_Interview_Questions.md`

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [07_Spring_Boot_Starters.md](./07_Spring_Boot_Starters.md)
- **Next →:** [09_Spring_Boot_Actuator.md](./09_Spring_Boot_Actuator.md)
- **Related:** [06_Spring_Boot_Fundamentals.md](./06_Spring_Boot_Fundamentals.md), [17_Maven_Build_Tool.md](./17_Maven_Build_Tool.md), [18_Gradle_For_Spring.md](./18_Gradle_For_Spring.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
