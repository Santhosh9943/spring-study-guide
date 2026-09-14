# Apache Maven Complete Guide

> **File:** `17_Maven_Build_Tool.md`
> **Part:** 6 — Build Tools
> **Prerequisites:** Basic Java, understanding of dependency management
> **Estimated Study Time:** 8–12 hours

---

## Table of Contents

1. [What is Maven?](#1-what-is-maven)
2. [Installation & Setup](#2-installation--setup)
3. [Project Structure](#3-project-structure)
4. [POM — Project Object Model](#4-pom--project-object-model)
5. [Maven Coordinates](#5-maven-coordinates)
6. [Build Lifecycles](#6-build-lifecycles)
7. [Phases & Goals](#7-phases--goals)
8. [Plugins](#8-plugins)
9. [Dependency Management](#9-dependency-management)
10. [Dependency Scopes](#10-dependency-scopes)
11. [Transitive Dependencies & Exclusions](#11-transitive-dependencies--exclusions)
12. [Parent POM & Inheritance](#12-parent-pom--inheritance)
13. [BOM (Bill of Materials)](#13-bom-bill-of-materials)
14. [Maven Profiles](#14-maven-profiles)
15. [Properties & Filtering](#15-properties--filtering)
16. [Repositories](#16-repositories)
17. [Multi-Module Projects](#17-multi-module-projects)
18. [Maven Wrapper (mvnw)](#18-maven-wrapper-mvnw)
19. [Common Commands](#19-common-commands)
20. [Troubleshooting](#20-troubleshooting)
21. [Interview Questions](#21-interview-questions)
22. [Cheat Sheet](#22-cheat-sheet)

---

## 1. What is Maven?

**Apache Maven** is a build automation and dependency management tool for Java. It uses a declarative XML file (`pom.xml`) to describe a project's structure, dependencies, and build process.

### Core Features

- **Convention over configuration** — standard directory layout
- **Declarative build** — describe *what*, not *how*
- **Dependency management** — transitive resolution, conflict resolution
- **Lifecycle model** — phases like compile, test, package, install
- **Plugin architecture** — extensible with plugins
- **Reproducible builds** — same POM → same output (mostly)

### Why Maven?

| Without Maven | With Maven |
|---------------|-----------|
| Manual jar downloads | Declare dependency; Maven downloads |
| Hand-managed classpath | Transitive resolution |
| Custom build scripts per project | Standard lifecycle |
| IDE-specific setup | IDE-agnostic |
| Manual version bumping | Centralized version management |

---

## 2. Installation & Setup

### Install

**SDKMAN (Linux/macOS):**
```bash
sdk install maven
```

**macOS (Homebrew):**
```bash
brew install maven
```

**Windows:** Download from https://maven.apache.org/download.cgi, extract, add `bin` to `PATH`.

**Docker:**
```bash
docker run -it --rm -v $(pwd):/app -w /app maven:3.9-eclipse-temurin-17 mvn package
```

### Verify

```bash
mvn -version
# Apache Maven 3.9.6
# Java version: 17.0.x
# OS: ...
```

### Directories

| Location | Purpose |
|----------|---------|
| `~/.m2/repository` | Local repository (downloaded artifacts) |
| `~/.m2/settings.xml` | User-level settings |
| `<project>/pom.xml` | Project config |
| `~/.m2/wrapper` | Maven wrapper distributions |

### settings.xml

```xml
<settings>
    <localRepository>${user.home}/.m2/repository</localRepository>
    <mirrors>
        <mirror>
            <id>central-mirror</id>
            <mirrorOf>central</mirrorOf>
            <url>https://repo1.maven.org/maven2</url>
        </mirror>
    </mirrors>
    <servers>
        <server>
            <id>internal-repo</id>
            <username>admin</username>
            <password>secret</password>
        </server>
    </servers>
    <profiles>
        <profile>
            <id>jdk17</id>
            <activation><activeByDefault>true</activeByDefault></activation>
            <properties>
                <maven.compiler.source>17</maven.compiler.source>
                <maven.compiler.target>17</maven.compiler.target>
            </properties>
        </profile>
    </profiles>
</settings>
```

---

## 3. Project Structure

### Standard Maven Layout

```
my-project/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/App.java
│   │   ├── resources/
│   │   │   ├── application.properties
│   │   │   └── logback.xml
│   │   └── webapp/                 (only for WAR)
│   │       └── WEB-INF/web.xml
│   └── test/
│       ├── java/
│       │   └── com/example/AppTest.java
│       └── resources/
│           └── test-data.sql
└── target/                          (build output, gitignored)
    ├── classes/
    ├── test-classes/
    └── my-app-1.0.0.jar
```

### Conventions

| Directory | Contents |
|-----------|----------|
| `src/main/java` | Production source |
| `src/main/resources` | Prod resources (properties, XML) |
| `src/main/webapp` | Web app files (WAR only) |
| `src/test/java` | Test source |
| `src/test/resources` | Test resources |
| `target/` | Build output (created) |

Deviate only if absolutely needed (via `<build><sourceDirectory>`).

---

## 4. POM — Project Object Model

The `pom.xml` is the heart of a Maven project.

### Minimal POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <name>My App</name>
    <description>Demo application</description>
</project>
```

### POM Sections

| Element | Purpose |
|---------|---------|
| `modelVersion` | Always `4.0.0` |
| `groupId` | Organization (reverse DNS) |
| `artifactId` | Project name |
| `version` | Version |
| `packaging` | `jar` (default), `war`, `pom`, `ear`, `maven-plugin` |
| `parent` | Parent POM for inheritance |
| `properties` | Key-value pairs |
| `dependencies` | Runtime/build dependencies |
| `dependencyManagement` | Versions for dependencies (not actual deps) |
| `build` | Plugins, resources, output dirs |
| `profiles` | Environment-specific configs |
| `modules` | Sub-modules (aggregator) |
| `distributionManagement` | Where to deploy |
| `repositories` | Additional repos |
| `licenses`, `developers`, `scm` | Metadata |

### Effective POM

The POM after inheritance and interpolation is called the **effective POM**.

```bash
mvn help:effective-pom
```

Prints the full merged POM.

---

## 5. Maven Coordinates

A dependency is uniquely identified by **GAV**:

```
groupId:artifactId:version[:packaging[:classifier]]
```

Example:

```
org.springframework.boot:spring-boot-starter-web:3.2.0
```

### Coordinates

| Part | Meaning | Example |
|------|---------|---------|
| `groupId` | Organization / namespace | `com.google.guava` |
| `artifactId` | Project name | `guava` |
| `version` | Version | `32.1.3-jre` |
| `packaging` | Type (default jar) | `jar`, `war`, `pom` |
| `classifier` | Variant | `sources`, `javadoc`, `tests` |

### Version Ranges (avoid!)

```xml
<version>[1.0,2.0)</version>   <!-- ⚠️ non-deterministic -->
```

Use exact versions or BOMs instead.

### Snapshots

`1.0.0-SNAPSHOT` — development version. Maven checks for updates frequently.

---

## 6. Build Lifecycles

Maven has **three** built-in lifecycles.

### 6.1 Default (Build) Lifecycle

The main lifecycle — compiles, tests, packages, deploys.

| Phase | Purpose |
|-------|---------|
| `validate` | Validate project structure |
| `initialize` | Initialize build state |
| `generate-sources` | Generate sources |
| `process-sources` | Process (filter) sources |
| `generate-resources` | Generate resources |
| `process-resources` | Copy resources to `target/classes` |
| `compile` | Compile source |
| `process-classes` | Post-process compiled classes |
| `generate-test-sources` | Generate test sources |
| `process-test-sources` | Process test sources |
| `generate-test-resources` | Generate test resources |
| `process-test-resources` | Copy test resources |
| `test-compile` | Compile test source |
| `process-test-classes` | Post-process test classes |
| `test` | Run tests |
| `prepare-package` | Prepare for packaging |
| `package` | Package into JAR/WAR |
| `pre-integration-test` | Setup IT |
| `integration-test` | Run IT |
| `post-integration-test` | Teardown IT |
| `verify` | Verify package is valid |
| `install` | Install to local repo (`~/.m2`) |
| `deploy` | Deploy to remote repo |

Running a phase executes all prior phases.

`mvn package` → validate → compile → test → package.

### 6.2 Clean Lifecycle

| Phase | Purpose |
|-------|---------|
| `pre-clean` | Pre-clean tasks |
| `clean` | Delete `target/` |
| `post-clean` | Post-clean tasks |

### 6.3 Site Lifecycle

| Phase | Purpose |
|-------|---------|
| `pre-site` | Before site |
| `site` | Generate site docs |
| `post-site` | After site |
| `site-deploy` | Deploy site |

### Lifecycle Relationships

```
clean   → clean lifecycle
default → build lifecycle
site    → site lifecycle
```

You can mix: `mvn clean install site`.

---

## 7. Phases & Goals

### Phase vs Goal

| Term | Meaning |
|------|---------|
| **Phase** | A step in a lifecycle (`compile`, `test`) |
| **Goal** | A specific task in a plugin (`compiler:compile`) |

A phase is bound to plugin goals.

### Running Goals Directly

```bash
mvn compiler:compile       # specific goal, bypasses lifecycle
mvn dependency:tree
mvn help:effective-pom
mvn surefire:test
```

### Plugin Goal Format

```
plugin-prefix:goal
```

Example: `surefire:test` runs the Surefire plugin's `test` goal.

Fully qualified:

```
org.apache.maven.plugins:maven-surefire-plugin:3.2.2:test
```

### Default Goal Bindings

| Phase | Bound Goal |
|-------|-----------|
| `compile` | `compiler:compile` |
| `test` | `surefire:test` |
| `package` | `jar:jar` or `war:war` |
| `install` | `install:install` |
| `deploy` | `deploy:deploy` |
| `clean` | `clean:clean` |
| `site` | `site:site` |

### Chaining Phases

```bash
mvn clean compile test package install
```

Same as `mvn clean install` (later phases subsume earlier).

---

## 8. Plugins

Plugins extend Maven. Nearly every action is a plugin goal.

### Common Plugins

| Plugin | Purpose |
|--------|---------|
| `maven-compiler-plugin` | Compile Java |
| `maven-surefire-plugin` | Run unit tests |
| `maven-failsafe-plugin` | Run integration tests |
| `maven-jar-plugin` | Build JAR |
| `maven-war-plugin` | Build WAR |
| `maven-install-plugin` | Install to local repo |
| `maven-deploy-plugin` | Deploy to remote |
| `maven-clean-plugin` | Clean target/ |
| `maven-resources-plugin` | Copy/filter resources |
| `maven-scm-plugin` | SCM integration |
| `maven-release-plugin` | Release management |
| `maven-enforcer-plugin` | Enforce rules (Java version) |
| `spring-boot-maven-plugin` | Fat JAR / run |
| `jacoco-maven-plugin` | Code coverage |
| `maven-shade-plugin` | Uber JAR with relocation |
| `maven-assembly-plugin` | Custom distributions |
| `maven-source-plugin` | Attach sources |
| `maven-javadoc-plugin` | Attach javadocs |

### Plugin Configuration

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <source>17</source>
                <target>17</target>
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Default Plugin Versions

Maven defines default plugin versions. Override only when needed.

### Spring Boot Plugin

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>   <!-- create fat jar -->
                <goal>build-info</goal>  <!-- META-INF/build-info.properties -->
            </goals>
        </execution>
    </executions>
</plugin>
```

### Multiple Executions

```xml
<plugin>
    <artifactId>maven-failsafe-plugin</artifactId>
    <executions>
        <execution>
            <id>integration-tests</id>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### Skipping Plugins

```bash
mvn package -DskipTests=true          # skip test execution
mvn package -Dmaven.test.skip=true    # skip compile + execution
mvn install -Dcheckstyle.skip=true
```

---

## 9. Dependency Management

### Declaring Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>3.2.0</version>
    </dependency>
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>32.1.3-jre</version>
    </dependency>
</dependencies>
```

### Dependency Scope

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

### Optional

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>optional-lib</artifactId>
    <optional>true</optional>   <!-- not transitive -->
</dependency>
```

### Dependency Tree

```bash
mvn dependency:tree
mvn dependency:tree -Dincludes=org.springframework.boot
mvn dependency:tree -Dverbose
```

### Duplicate Dependency (Same GAV, different declared version)

Maven uses **nearest-wins** strategy by default:

```
A → B → C:1.0   (depth 2)
A → C:2.0       (depth 1)   ← wins
```

Also applies `<dependencyManagement>` and BOM versions.

### Force Version

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>32.1.3-jre</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

`dependencyManagement` sets **default versions** — used when a dependency is declared without a version.

### Analyze Dependencies

```bash
mvn dependency:analyze          # unused / undeclared
mvn dependency:analyze-duplicate
mvn dependency:list
mvn dependency:purge-local-repository
```

---

## 10. Dependency Scopes

| Scope | Compile classpath | Test classpath | Runtime classpath | Transitive |
|-------|-------------------|----------------|-------------------|-----------|
| `compile` (default) | ✅ | ✅ | ✅ | ✅ |
| `provided` | ✅ | ✅ | ❌ | ❌ |
| `runtime` | ❌ | ✅ | ✅ | ✅ |
| `test` | ❌ | ✅ | ❌ | ❌ |
| `system` | ✅ | ✅ | ✅ | ❌ (avoid) |
| `import` | (in `dependencyManagement` only) | — | — | — |

### Examples

```xml
<!-- compile: available everywhere, propagated -->
<dependency>
    <artifactId>guava</artifactId>
    <scope>compile</scope>
</dependency>

<!-- provided: JDK or container supplies at runtime (e.g., servlet-api, lombok) -->
<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <scope>provided</scope>
</dependency>

<!-- runtime: not needed to compile, but needed to run (e.g., JDBC driver) -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- test: only in test -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>

<!-- system: local file (avoid; not portable) -->
<dependency>
    <groupId>com.example</groupId>
    <artifactId>local-lib</artifactId>
    <version>1.0</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/lib/local-lib.jar</systemPath>
</dependency>
```

### Scope Summary

- **compile** — the default. Everything uses it.
- **provided** — the runtime environment provides it (Servlet API, Lombok).
- **runtime** — needed at runtime, not compile (JDBC drivers).
- **test** — testing frameworks.
- **system** — local path (avoid).
- **import** — imports a BOM (see §13).

---

## 11. Transitive Dependencies & Exclusions

### Transitive by Default

`A depends on B; B depends on C` → A's classpath includes B and C (if scope permits).

### Exclusion

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
        <exclusion>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### Wildcard Exclusion

```xml
<exclusion>
    <groupId>com.example.*</groupId>
    <artifactId>*</artifactId>
</exclusion>
```

(Spring Boot only supports this via shade/assembly plugin — plain Maven requires explicit.)

### Version Conflict Example

```
A → B:1.0
A → C:1.0
C → B:2.0
```

Nearest wins: B:1.0 (depth 1) over B:2.0 (depth 2). Force with `dependencyManagement`.

### Managing Transitives

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.16.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Overrides any transitive version of `jackson-databind`.

### Debugging Conflicts

```bash
mvn dependency:tree -Dverbose -Dincludes=com.fasterxml.jackson.core
```

---

## 12. Parent POM & Inheritance

### Parent

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
    <relativePath/>   <!-- look in repo, not filesystem -->
</parent>
```

Inherits:
- Dependencies (via `dependencyManagement`)
- Plugin versions and configurations
- Properties (`java.version`, `spring-boot.version`, etc.)
- Resource filtering config
- Encoding defaults

### Custom Parent POM

```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>my-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.2.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

Child:

```xml
<parent>
    <groupId>com.example</groupId>
    <artifactId>my-parent</artifactId>
    <version>1.0.0</version>
</parent>
```

### Inheritance vs Import

| Aspect | Inheritance (`<parent>`) | Import (`<scope>import</scope>`) |
|--------|--------------------------|----------------------------------|
| Only one | ✅ | ❌ (multiple) |
| Inherits properties | ✅ | ❌ |
| Inherits plugins | ✅ | ❌ (only `dependencyManagement`) |
| Purpose | Full parent-child | Shared versions |

---

## 13. BOM (Bill of Materials)

A **BOM** is a POM that only declares versions in `dependencyManagement`. Import it to align versions across many artifacts.

### Spring Boot BOM

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <!-- no version — BOM handles it -->
    </dependency>
</dependencies>
```

### Why BOM?

- Aligns versions across a family of libraries
- Avoids version drift
- Enables a single version bump

### Common BOMs

| BOM | Purpose |
|-----|---------|
| `spring-boot-dependencies` | All Spring Boot deps |
| `spring-cloud-dependencies` | Spring Cloud |
| `jackson-bom` | Jackson |
| `netty-bom` | Netty |
| `spring-framework-bom` | Spring Framework |
| `grpc-bom` | gRPC |

### Custom BOM

```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>my-bom</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>lib-a</artifactId>
                <version>1.2.0</version>
            </dependency>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>lib-b</artifactId>
                <version>2.0.1</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

Consumers `import` it.

---

## 14. Maven Profiles

Profiles activate config conditionally (dev, ci, prod, jdk version, OS, etc.).

### Defining

```xml
<profiles>
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <env>dev</env>
        </properties>
        <dependencies>
            <dependency>
                <groupId>com.h2database</groupId>
                <artifactId>h2</artifactId>
                <scope>runtime</scope>
            </dependency>
        </dependencies>
    </profile>
    
    <profile>
        <id>prod</id>
        <properties>
            <env>prod</env>
        </properties>
    </profile>
</profiles>
```

### Activation

| Trigger | Example |
|---------|---------|
| Default | `<activeByDefault>true</activeByDefault>` |
| Property | `<property><name>env</name><value>prod</value></property>` |
| JDK | `<jdk>17</jdk>` |
| OS | `<os><family>unix</family></os>` |
| File | `<file><exists>foo.txt</exists></file>` |
| By id (CLI) | `-Pprod` |

### Activation via Property

```bash
mvn package -Denv=prod
```

### Activation via Profile ID

```bash
mvn package -Pprod
mvn package -Pdev,ci   # multiple
mvn package -P !prod   # disable profile
```

### Profile Content

Profiles can override or add: `dependencies`, `dependencyManagement`, `properties`, `build` (plugins, resources), `repositories`, `distributionManagement`.

### Inspecting Active Profiles

```bash
mvn help:active-profiles
```

### Best Practices

- Avoid `activeByDefault` — hidden
- Prefer **Spring Profiles** at runtime for app config; use **Maven profiles** for build-time concerns
- Keep profile count small

---

## 15. Properties & Filtering

### Defining Properties

```xml
<properties>
    <java.version>17</java.version>
    <spring-boot.version>3.2.0</spring-boot.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

### Using Properties

```xml
<version>${spring-boot.version}</version>
<source>${java.version}</source>
```

### Built-in Properties

| Property | Value |
|----------|-------|
| `${project.basedir}` | Project directory |
| `${project.build.directory}` | `target/` |
| `${project.version}` | Project version |
| `${project.artifactId}` | Artifact ID |
| `${project.groupId}` | Group ID |
| `${project.name}` | Project name |
| `${basedir}` | Alias for `project.basedir` |

### Environment Variables

```xml
<property>
    <name>env</name>
    <value>${env.HOME}</value>
</property>
```

Format: `${env.VAR}`.

### System Properties

```bash
mvn package -DskipTests -Denv=prod
```

```xml
<property>${env}</property>
```

### Resource Filtering

Replace `${...}` in resource files at build time.

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
            <includes>
                <include>application-*.properties</include>
            </includes>
        </resource>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>false</filtering>
            <excludes>
                <exclude>application-*.properties</exclude>
            </excludes>
        </resource>
    </resources>
</build>
```

**Example:** `application.properties`:

```properties
app.version=${project.version}
app.name=${project.name}
```

Becomes:

```properties
app.version=1.0.0
app.name=My App
```

⚠️ **Don't filter binaries** — corrupts them. Use `<nonFilteredFileExtensions>`.

```xml
<plugin>
    <artifactId>maven-resources-plugin</artifactId>
    <configuration>
        <nonFilteredFileExtensions>
            <nonFilteredFileExtension>woff2</nonFilteredFileExtension>
            <nonFilteredFileExtension>ttf</nonFilteredFileExtension>
        </nonFilteredFileExtensions>
    </configuration>
</plugin>
```

---

## 16. Repositories

### Local Repository

`~/.m2/repository` — downloaded artifacts + installed local projects.

### Central Repository

https://repo.maven.apache.org/maven2 — default for all Maven builds.

### Remote Repository

```xml
<repositories>
    <repository>
        <id>spring-milestones</id>
        <url>https://repo.spring.io/milestone</url>
    </repository>
</repositories>
```

### Plugin Repository

```xml
<pluginRepositories>
    <pluginRepository>
        <id>spring-milestones</id>
        <url>https://repo.spring.io/milestone</url>
    </pluginRepository>
</pluginRepositories>
```

### Repository Order

1. Local
2. Each declared `<repository>` in order
3. Central (unless `<mirrorOf>*</mirrorOf>` overrides)

### Snapshot vs Release

- **Release** — immutable, permanent
- **Snapshot** — dev, overwritten; Maven checks daily or with `-U`

```bash
mvn clean install -U        # force update snapshots
```

### Deploying

```xml
<distributionManagement>
    <repository>
        <id>my-releases</id>
        <url>https://repo.example.com/releases</url>
    </repository>
    <snapshotRepository>
        <id>my-snapshots</id>
        <url>https://repo.example.com/snapshots</url>
    </snapshotRepository>
</distributionManagement>
```

Credentials in `settings.xml`.

### Mirrors

Route all repository requests through a mirror:

```xml
<mirrors>
    <mirror>
        <id>company-proxy</id>
        <url>https://nexus.example.com/repository/maven-public/</url>
        <mirrorOf>*</mirrorOf>
    </mirror>
</mirrors>
```

Common `<mirrorOf>` values:

| Value | Meaning |
|-------|---------|
| `central` | Central only |
| `*` | Everything |
| `external:*` | Everything except localhost |
| `*,!internal` | All but `internal` |

---

## 17. Multi-Module Projects

### Structure

```
parent/
├── pom.xml                 (aggregator + parent)
├── module-a/
│   └── pom.xml
├── module-b/
│   └── pom.xml
└── module-c/
    └── pom.xml
```

### Aggregator POM

```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <modules>
        <module>module-a</module>
        <module>module-b</module>
        <module>module-c</module>
    </modules>
    
    <properties>
        <java.version>17</java.version>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>module-a</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

### Module POM

```xml
<parent>
    <groupId>com.example</groupId>
    <artifactId>parent</artifactId>
    <version>1.0.0</version>
</parent>

<artifactId>module-a</artifactId>
<packaging>jar</packaging>

<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>module-b</artifactId>
        <!-- version from parent's depMgmt -->
    </dependency>
</dependencies>
```

### Building

```bash
mvn install                # build all modules (from root)
mvn install -pl module-a   # single module
mvn install -pl module-a -am   # also build dependencies
mvn install -pl module-a -amd  # also build dependents
```

### Aggregator vs Parent

| Aspect | Aggregator | Parent |
|--------|-----------|--------|
| Role | Groups modules | Inherits config |
| Packaging | `pom` | `pom` |
| Contains `<modules>` | ✅ | ❌ |
| Referenced via `<parent>` | ❌ | ✅ |
| Same POM can be both | ✅ | ✅ |

Often the same POM is both.

---

## 18. Maven Wrapper (mvnw)

Bundles Maven so builds use a specific version without installation.

### Generate

```bash
mvn wrapper:wrapper              # default
mvn wrapper:wrapper -Dmaven=3.9.6
```

Creates:

```
.mvn/
├── wrapper/
│   └── maven-wrapper.properties
└── jvm.config (optional)
mvnw
mvnw.cmd
```

### Usage

```bash
./mvnw clean install
./mvnw.cmd clean install    # Windows
```

### maven-wrapper.properties

```properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.6/apache-maven-3.9.6-bin.zip
wrapperUrl=https://repo.maven.apache.org/maven2/org/apache/maven/wrapper/maven-wrapper/3.2.0/maven-wrapper-3.2.0.jar
```

### Why Use It

- **Reproducible builds** across dev/CI
- **No Maven installation** required
- **Version pinned** in repo
- Same for Gradle (gradlew)

**Recommended for all new projects.** Spring Initializr includes it.

---

## 19. Common Commands

### Build

```bash
mvn compile               # compile
mvn test                  # run tests
mvn package               # create JAR/WAR (in target/)
mvn install               # install to ~/.m2
mvn deploy                # deploy to remote
mvn clean                 # delete target/
mvn clean install         # clean + install
mvn clean package -DskipTests
```

### Run

```bash
mvn spring-boot:run
mvn exec:java -Dexec.mainClass="com.example.App"
```

### Dependency

```bash
mvn dependency:tree
mvn dependency:tree -Dincludes=com.fasterxml.jackson
mvn dependency:tree -Dverbose
mvn dependency:analyze
mvn dependency:list
mvn dependency:resolve
mvn dependency:purge-local-repository
```

### Information

```bash
mvn help:effective-pom
mvn help:describe -Dplugin=compiler
mvn help:active-profiles
mvn help:evaluate -Dexpression=project.version -q -DforceStdout
```

### Plugin Goal

```bash
mvn compiler:compile
mvn surefire:test -Dtest=MyTest
mvn surefire:test -Dtest=MyTest#myMethod
mvn failsafe:integration-test
mvn jacoco:report
mvn versions:display-dependency-updates
mvn versions:display-plugin-updates
mvn versions:use-latest-releases
```

### Release

```bash
mvn versions:set -DnewVersion=2.0.0
mvn release:prepare
mvn release:perform
```

### Debug

```bash
mvn -X package             # debug output
mvn -e package             # show errors
mvn -U package             # force update snapshots
mvn -o package             # offline
mvn -q package             # quiet
```

---

## 20. Troubleshooting

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Could not resolve dependencies` | Missing repo / offline | Check network, add repository |
| `Dependency convergence error` | Version conflict (Enforcer) | Use `dependencyManagement` |
| `Unknown host` | Offline / proxy | Set `settings.xml` proxy |
| `Invalid POM` | XML error | Validate XML |
| `ClassNotFoundException` | Wrong scope or missing dep | `mvn dependency:tree` |
| `No compiler provided` | Java version mismatch | Set `maven.compiler.source/target` |
| `OutOfMemoryError` | Maven heap too small | `MAVEN_OPTS=-Xmx2g` |
| `HTTPS required` | Repo requires HTTPS | Update to HTTPS URLs |

### Proxy

```xml
<proxies>
    <proxy>
        <id>http-proxy</id>
        <active>true</active>
        <protocol>http</protocol>
        <host>proxy.example.com</host>
        <port>8080</port>
        <username>user</username>
        <password>pwd</password>
        <nonProxyHosts>localhost|*.internal</nonProxyHosts>
    </proxy>
</proxies>
```

### MAVEN_OPTS

```bash
export MAVEN_OPTS="-Xmx2g -XX:+UseG1GC"
```

### Force Update

```bash
mvn clean install -U
```

Or delete: `rm -rf ~/.m2/repository/org/springframework`

### Logging

```bash
mvn -X            # debug
mvn -e            # error stack traces
mvn -Dorg.slf4j.simpleLogger.defaultLogLevel=debug
```

---

## 21. Interview Questions

### Q1. What is Maven?

**Answer:** A build automation and dependency management tool for Java. Uses a declarative `pom.xml` to describe project structure, dependencies, plugins, and lifecycle. Follows convention over configuration.

### Q2. Explain Maven's build lifecycles.

**Answer:** Three: **default** (validate → compile → test → package → verify → install → deploy), **clean** (pre-clean → clean → post-clean), **site** (pre-site → site → post-site → site-deploy). Running a phase executes all prior phases in that lifecycle.

### Q3. Difference between phase and goal?

**Answer:** **Phase** is a step in a lifecycle (`compile`, `package`). **Goal** is a specific plugin task (`compiler:compile`, `surefire:test`). Phases bind to goals — running a phase triggers its goals.

### Q4. What are Maven coordinates?

**Answer:** GAV — `groupId`, `artifactId`, `version` (plus optional packaging and classifier). Uniquely identify artifacts in repositories.

### Q5. What are dependency scopes?

**Answer:** `compile` (default; available everywhere), `provided` (JDK/container supplies), `runtime` (needed at runtime, not compile), `test` (only testing), `system` (local path, avoid), `import` (imports a BOM in depMgmt).

### Q6. What is transitive dependency?

**Answer:** A dependency pulled in automatically because another dependency needs it. Maven resolves them recursively. Version conflicts resolved by nearest-wins.

### Q7. What is the nearest-wins strategy?

**Answer:** If multiple paths lead to different versions of the same artifact, Maven picks the one closest to the root. Depth 1 beats depth 2, even if depth 2 has a newer version.

### Q8. What is dependencyManagement?

**Answer:** A section that declares default versions for dependencies. Children declare dependencies without a version; the version comes from depMgmt. Ensures consistent versions across modules.

### Q9. Difference between dependencies and dependencyManagement?

**Answer:** `dependencies` = actual dependencies (included in build). `dependencyManagement` = defaults (versions/scopes) for children's dependencies; doesn't add dependencies itself.

### Q10. What is a BOM?

**Answer:** Bill of Materials — a POM with only `dependencyManagement` declaring versions. Import it with `<scope>import</scope>` to align versions across a family of libraries. Spring Boot's `spring-boot-dependencies` is a common BOM.

### Q11. What is a parent POM?

**Answer:** A POM referenced with `<parent>`. Inherits dependencies, depMgmt, plugins, and properties. Only one parent per project. Spring Boot uses `spring-boot-starter-parent`.

### Q12. Parent vs BOM — which is better?

**Answer:** They solve different problems. Parent provides full inheritance (properties, plugins). BOM provides shared versions for dependencies. You can have one parent and import multiple BOMs. Spring Boot's parent already imports `spring-boot-dependencies`.

### Q13. What are Maven profiles?

**Answer:** Conditional configurations for different environments (dev, prod, ci, jdk, os). Activated via property, file, JDK, OS, or `-P<id>`. Used for build-time variations, not runtime config.

### Q14. How do you exclude a transitive dependency?

**Answer:**

```xml
<dependency>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### Q15. What is the Maven Wrapper?

**Answer:** A self-contained Maven for the project. `mvnw`/`mvnw.cmd` scripts download and use a pinned Maven version. No global install needed. Ensures reproducible builds.

### Q16. How do you run a Spring Boot app with Maven?

**Answer:**

```bash
mvn spring-boot:run
```

Uses the `spring-boot-maven-plugin`.

### Q17. How do you build a fat JAR?

**Answer:** Spring Boot's `spring-boot-maven-plugin` `repackage` goal creates an executable JAR with all dependencies. Alternatively, `maven-shade-plugin` or `maven-assembly-plugin`.

### Q18. How do you skip tests?

**Answer:** `-DskipTests=true` (compile but don't run). `-Dmaven.test.skip=true` (skip compile + run). `-Dtest=MyTest` to run specific tests.

### Q19. What is the difference between install and deploy?

**Answer:** `install` copies the built artifact to the local repo (`~/.m2`). `deploy` uploads it to a remote repo (`<distributionManagement>`) for other developers/CI.

### Q20. How do you analyze dependencies?

**Answer:** `mvn dependency:tree` (show tree), `mvn dependency:analyze` (find unused/undeclared), `mvn dependency:list`, `mvn versions:display-dependency-updates`.

### Q21. What is resource filtering?

**Answer:** Substitution of `${...}` placeholders in resource files during build. Configure with `<resources><resource><filtering>true</filtering>`. Useful for injecting versions, env-specific config. Exclude binaries.

### Q22. How do you resolve dependency conflicts?

**Answer:** `dependencyManagement` (pin version), `<exclusions>` (remove transitive), `maven-enforcer-plugin` (fail on convergence issues), or upgrade the transitive dependency.

### Q23. What is a multi-module project?

**Answer:** A parent POM with `<modules>` listing child modules. Parent can be a shared parent (`<parent>` in children) and/or aggregator. Build with `mvn install` from root. Options: `-pl` (list), `-am` (also make deps), `-amd` (also make dependents).

### Q24. What is dependency mediation?

**Answer:** The process of choosing a single version when transitive dependencies conflict. Maven 3 uses nearest-wins. Override with `dependencyManagement`.

### Q25. How do you force Maven to re-download dependencies?

**Answer:** `mvn -U clean install` (update snapshots). Delete `~/.m2/repository/org/springframework` for a full refresh.

---

## 22. Cheat Sheet

### POM Structure

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <properties>
        <java.version>17</java.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### Lifecycle (Default)

```
validate → compile → test → package → verify → install → deploy
```

### Scopes

```
compile   ✅ everywhere
provided  runtime env supplies
runtime   runtime only
test      tests only
system    local path (avoid)
import    for BOMs
```

### Common Commands

```bash
mvn clean install                 # full build
mvn package                       # build JAR
mvn test                          # run tests
mvn spring-boot:run               # run app
mvn dependency:tree               # dep tree
mvn dependency:analyze            # unused/undeclared
mvn help:effective-pom            # merged POM
mvn versions:display-dependency-updates
mvn clean install -DskipTests     # skip tests
mvn clean install -U              # force update
mvn -X package                    # debug
```

### Multi-Module

```bash
mvn install                # all modules
mvn install -pl module-a   # single
mvn install -pl module-a -am   # + dependencies
mvn install -pl module-a -amd  # + dependents
```

### Wrapper

```bash
./mvnw clean install
```

### BOM Import

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Exclusion

```xml
<exclusions>
    <exclusion>
        <groupId>commons-logging</groupId>
        <artifactId>commons-logging</artifactId>
    </exclusion>
</exclusions>
```

### Profiles

```xml
<profiles>
    <profile>
        <id>prod</id>
        <properties><env>prod</env></properties>
    </profile>
</profiles>
```

```bash
mvn package -Pprod
```

### Cross-References

- **Previous:** `16_Spring_Testing.md`
- **Next:** `18_Gradle_For_Spring.md`
- **Related:** `06_Spring_Boot_Fundamentals.md`
- **Interview:** `24_Spring_Interview_Questions.md` (§Maven)

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [16_Spring_Testing.md](./16_Spring_Testing.md)
- **Next →:** [18_Gradle_For_Spring.md](./18_Gradle_For_Spring.md)
- **Related:** [18_Gradle_For_Spring.md](./18_Gradle_For_Spring.md), [06_Spring_Boot_Fundamentals.md](./06_Spring_Boot_Fundamentals.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
