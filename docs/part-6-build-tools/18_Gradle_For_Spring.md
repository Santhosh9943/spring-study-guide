# Gradle for Spring Developers

> **File:** `18_Gradle_For_Spring.md`
> **Part:** 6 — Build Tools
> **Prerequisites:** Basic Java, Maven helpful (comparison)
> **Estimated Study Time:** 5–6 hours

---

## Table of Contents

1. [What is Gradle?](#1-what-is-gradle)
2. [Gradle vs Maven](#2-gradle-vs-maven)
3. [Installation & Wrapper](#3-installation--wrapper)
4. [Project Structure](#4-project-structure)
5. [build.gradle (Groovy)](#5-buildgradle-groovy)
6. [build.gradle.kts (Kotlin DSL)](#6-buildgradlekts-kotlin-dsl)
7. [settings.gradle](#7-settingsgradle)
8. [Dependency Configurations](#8-dependency-configurations)
9. [Spring Boot Gradle Plugin](#9-spring-boot-gradle-plugin)
10. [Dependency Management Plugin](#10-dependency-management-plugin)
11. [Gradle Tasks](#11-gradle-tasks)
12. [Multi-Project Builds](#12-multi-project-builds)
13. [Properties & Profiles](#13-properties--profiles)
14. [Common Commands](#14-common-commands)
15. [Interview Questions](#15-interview-questions)
16. [Cheat Sheet](#16-cheat-sheet)

---

## 1. What is Gradle?

**Gradle** is a modern build automation tool using Groovy or Kotlin DSL instead of XML. It builds on Maven's conventions but adds:

- **Programmable build logic** — write code, not just config
- **Faster builds** — incremental, build cache, daemon
- **Flexible** — tasks are first-class
- **Multi-language** — Java, Kotlin, Groovy, Scala, Android

### Core Concepts

| Concept | Meaning |
|---------|---------|
| **Project** | A build unit (root or subproject) |
| **Task** | A unit of work (compileJava, test, jar) |
| **Plugin** | Adds tasks/conventions (java, boot) |
| **Configuration** | Dependency bucket (implementation, testImplementation) |
| **Build script** | `build.gradle` or `build.gradle.kts` |

---

## 2. Gradle vs Maven

| Aspect | Maven | Gradle |
|--------|-------|--------|
| Config file | `pom.xml` (XML) | `build.gradle` / `.kts` |
| Language | XML | Groovy / Kotlin |
| Build logic | Limited | Full programming |
| Speed | Slower | Faster (daemon, incremental, cache) |
| Convention | Strong | Convention + flexibility |
| Learning curve | Easier | Steeper |
| Android | ❌ | ✅ (default) |
| Spring Boot | ✅ | ✅ |
| Multi-module | Verbose | Concise |
| Task graph | Fixed lifecycle | Flexible DAG |

### Which to Choose?

**Maven:**
- Simpler builds
- Team familiar with XML
- Strict conventions desired
- Large ecosystem alignment

**Gradle:**
- Complex custom logic
- Performance matters
- Multi-language / Android
- Kotlin/Groovy-friendly team

**Spring Boot supports both equally.** Spring Initializr offers both.

---

## 3. Installation & Wrapper

### Install

**SDKMAN:**
```bash
sdk install gradle
```

**Homebrew:**
```bash
brew install gradle
```

### Wrapper (Recommended)

```bash
gradle wrapper --gradle-version 8.5
```

Creates:

```
gradlew
gradlew.bat
gradle/wrapper/gradle-wrapper.jar
gradle/wrapper/gradle-wrapper.properties
```

Use wrapper always:

```bash
./gradlew build
./gradlew.bat build     # Windows
```

### gradle-wrapper.properties

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

### Caches

| Location | Purpose |
|----------|---------|
| `~/.gradle/caches/` | Dependency & build caches |
| `~/.gradle/wrapper/` | Wrapper distributions |
| `<project>/.gradle/` | Project-specific state |

---

## 4. Project Structure

### Standard Layout

```
my-project/
├── build.gradle                (or build.gradle.kts)
├── settings.gradle             (or settings.gradle.kts)
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── src/
│   ├── main/
│   │   ├── java/
│   │   ├── resources/
│   │   └── webapp/
│   └── test/
│       ├── java/
│       └── resources/
└── build/                      (output, gitignored)
    ├── classes/
    ├── libs/
    └── reports/
```

Same layout as Maven (this is the **default**).

---

## 5. build.gradle (Groovy)

### Minimal

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

group = 'com.example'
version = '1.0.0'
sourceCompatibility = '17'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### Anatomy

| Block | Purpose |
|-------|---------|
| `plugins` | Add plugins |
| `group`, `version` | Project coordinates |
| `repositories` | Where to fetch deps |
| `dependencies` | Declare deps |
| `tasks` | Configure tasks |

### Extending

```groovy
plugins {
    id 'java'
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

tasks.withType(JavaCompile).configureEach {
    options.encoding = 'UTF-8'
}
```

### Custom Task

```groovy
tasks.register('hello') {
    doLast {
        println 'Hello, Gradle!'
    }
}
```

Run: `./gradlew hello`

---

## 6. build.gradle.kts (Kotlin DSL)

Modern, type-safe, IDE-friendly. **Recommended for new projects.**

### Minimal Spring Boot

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
}

group = "com.example"
version = "1.0.0"

java {
    sourceCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

### Kotlin Project

```kotlin
plugins {
    kotlin("jvm") version "1.9.20"
    kotlin("plugin.spring") version "1.9.20"
    kotlin("plugin.jpa") version "1.9.20"
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.jetbrains.kotlin:kotlin-test-junit5")
}
```

### Custom Task

```kotlin
tasks.register("hello") {
    doLast {
        println("Hello, Gradle!")
    }
}
```

### Why Kotlin DSL?

- Type-safe (IDE completion, refactoring)
- Better error messages
- Kotlin-first teams

### Why Groovy DSL?

- Terser for simple builds
- Older, more examples
- Dynamic flexibility

**Choose one; don't mix.**

---

## 7. settings.gradle

Defines the root project name and included subprojects.

### Groovy

```groovy
rootProject.name = 'my-app'
include 'module-a', 'module-b'
```

### Kotlin

```kotlin
rootProject.name = "my-app"
include("module-a", "module-b")
```

### Plugin Management

```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}
```

### Dependency Resolution Management (Gradle 7+)

```kotlin
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        mavenCentral()
    }
}
```

Forces repositories to be defined centrally.

---

## 8. Dependency Configurations

### Common Configurations

| Configuration | Extends | Purpose |
|---------------|---------|---------|
| `implementation` | — | Runtime + compile (not exported to consumers) |
| `api` | — | Exposed to consumers (for library projects) |
| `compileOnly` | — | Compile only (Lombok, annotations) |
| `runtimeOnly` | — | Runtime only (JDBC drivers) |
| `testImplementation` | implementation | Test compile + runtime |
| `testCompileOnly` | compileOnly | Test compile only |
| `testRuntimeOnly` | runtimeOnly | Test runtime only |
| `annotationProcessor` | — | Annotation processors |
| `testAnnotationProcessor` | — | Test annotation processors |
| `developmentOnly` | — | Dev-only (Spring Boot DevTools) |

### Examples

```kotlin
dependencies {
    // Main code
    implementation("org.springframework.boot:spring-boot-starter-web")
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
    runtimeOnly("org.postgresql:postgresql")
    developmentOnly("org.springframework.boot:spring-boot-devtools")
    
    // Tests
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.testcontainers:postgresql")
    testCompileOnly("org.projectlombok:lombok")
    testAnnotationProcessor("org.projectlombok:lombok")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

### implementation vs api

- `implementation` — deps not exposed to consumers; internal
- `api` — deps exposed to consumers (they can use them transitively)

For **applications**, use `implementation`. For **libraries**, use `api` only for public API types.

### Excluding Transitive

```kotlin
implementation("org.springframework.boot:spring-boot-starter-web") {
    exclude(group = "org.springframework.boot", module = "spring-boot-starter-tomcat")
}
```

Global exclusion:

```kotlin
configurations.all {
    exclude(group = "commons-logging", module = "commons-logging")
}
```

### Force Version

```kotlin
configurations.all {
    resolutionStrategy {
        force("com.google.guava:guava:32.1.3-jre")
    }
}
```

### Fail on Version Conflict

```kotlin
configurations.all {
    resolutionStrategy {
        failOnVersionConflict()
    }
}
```

### Dynamic Versions

```kotlin
implementation("com.example:lib:1.+")           // ⚠️ avoid
implementation("com.example:lib:[1.0,2.0)")     // ⚠️ avoid
```

Use exact versions or BOMs.

### Dependency Insight

```bash
./gradlew dependencyInsight --dependency guava --configuration runtimeClasspath
```

Shows why a version was chosen.

### Dependency Tree

```bash
./gradlew dependencies
./gradlew dependencies --configuration runtimeClasspath
```

---

## 9. Spring Boot Gradle Plugin

### Apply

```kotlin
plugins {
    id("org.springframework.boot") version "3.2.0"
}
```

Adds:
- `bootJar` task — creates executable fat JAR
- `bootRun` task — runs the app
- `bootBuildInfo` — generates `build-info.properties`
- Dependency management for `spring-boot-dependencies`
- Replaces the standard `jar` with `bootJar` by default

### Tasks

| Task | Purpose |
|------|---------|
| `bootJar` | Build executable JAR |
| `bootWar` | Build executable WAR |
| `bootRun` | Run the app |
| `bootBuildImage` | Build container image |
| `bootBuildInfo` | Generate build info |

### Customize

```kotlin
tasks.named<org.springframework.boot.gradle.tasks.bundling.BootJar>("bootJar") {
    archiveFileName.set("my-app.jar")
    mainClass.set("com.example.App")
    layered {
        enabled.set(true)
    }
}
```

### Run

```bash
./gradlew bootRun
./gradlew bootRun --args='--server.port=9090'
```

### Build Image (Cloud Native Buildpacks)

```bash
./gradlew bootBuildImage
```

Requires Docker; produces an optimized OCI image.

### Exclude DevTools in Production

```kotlin
developmentOnly("org.springframework.boot:spring-boot-devtools")
```

`developmentOnly` is automatically excluded from the bootJar.

---

## 10. Dependency Management Plugin

`io.spring.dependency-management` mimics Maven's BOM behavior.

### Apply

```kotlin
plugins {
    id("io.spring.dependency-management") version "1.1.4"
}
```

### Versionless Dependencies

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
}

dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.0")
    }
}
```

### Per-Dependency Override

```kotlin
dependencyManagement {
    dependencies {
        dependency("com.google.guava:guava:32.1.3-jre")
    }
}
```

### Alternative: Gradle Platform

Modern alternative without the plugin:

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.2.0"))
    implementation(platform("org.springframework.cloud:spring-cloud-dependencies:2023.0.0"))
    
    implementation("org.springframework.boot:spring-boot-starter-web")
    // version omitted — from BOM
}
```

**Native Gradle**, no extra plugin. Recommended for new projects.

---

## 11. Gradle Tasks

### Core Java Tasks

| Task | Purpose |
|------|---------|
| `compileJava` | Compile main sources |
| `compileTestJava` | Compile test sources |
| `processResources` | Copy main resources |
| `processTestResources` | Copy test resources |
| `classes` | Compile main + process resources |
| `testClasses` | Compile tests |
| `test` | Run tests |
| `jar` | Build JAR |
| `assemble` | Build everything |
| `check` | Run verification (tests, lint) |
| `build` | assemble + check |
| `clean` | Delete `build/` |

### Lifecycle Tasks

```
build = assemble + check
assemble = jar, bootJar, ...
check = test, integrationTest, ...
```

### Custom Task

```kotlin
tasks.register("greet") {
    doLast {
        println("Hello, Gradle!")
    }
}
```

### Task with Inputs/Outputs

```kotlin
tasks.register<Copy>("copyDocs") {
    from("docs")
    into(layout.buildDirectory.dir("docs"))
}
```

Enables up-to-date checks and caching.

### Task Dependencies

```kotlin
tasks.named("test") {
    dependsOn("integrationTest")
}
```

Or:

```kotlin
tasks.register("doWork") {
    dependsOn("compileJava")
    doLast { /* ... */ }
}
```

### Task Types

```kotlin
tasks.register<Test>("integrationTest") {
    description = "Runs integration tests"
    group = "verification"
    testClassesDirs = sourceSets["test"].output.classesDirs
    classpath = sourceSets["test"].runtimeClasspath
    useJUnitPlatform {
        includeTags("integration")
    }
    shouldRunAfter(tasks.test)
}

tasks.named("check") {
    dependsOn("integrationTest")
}
```

### Skipping Tasks

```bash
./gradlew build -x test             # exclude test
./gradlew test --tests "*ServiceTest"
./gradlew test --tests "com.example.UserServiceTest.findById"
```

### Parallel Execution

```bash
./gradlew build --parallel
```

Or in `gradle.properties`:

```properties
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.daemon=true
```

---

## 12. Multi-Project Builds

### Structure

```
root/
├── settings.gradle.kts
├── build.gradle.kts          (root: shared config)
├── module-a/
│   └── build.gradle.kts
├── module-b/
│   └── build.gradle.kts
└── module-c/
    └── build.gradle.kts
```

### settings.gradle.kts

```kotlin
rootProject.name = "my-platform"
include("module-a", "module-b", "module-c")
```

### Root build.gradle.kts

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.2.0" apply false
    id("io.spring.dependency-management") version "1.1.4" apply false
}

allprojects {
    group = "com.example"
    version = "1.0.0"
    
    repositories {
        mavenCentral()
    }
}

subprojects {
    apply(plugin = "java")
    apply(plugin = "io.spring.dependency-management")
    
    java {
        sourceCompatibility = JavaVersion.VERSION_17
    }
    
    dependencies {
        testImplementation("org.springframework.boot:spring-boot-starter-test")
    }
    
    tasks.withType<Test> {
        useJUnitPlatform()
    }
}
```

### Subproject build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot")
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation(project(":module-b"))   // inter-module dependency
}
```

### Building

```bash
./gradlew build                             # everything
./gradlew :module-a:build                   # single module
./gradlew :module-a:build --parallel
```

### Shared Config Approaches

| Approach | Description |
|----------|-------------|
| `allprojects` | Apply to root + all subprojects |
| `subprojects` | Apply to subprojects only |
| Convention plugins | Reusable plugin (buildSrc) — recommended |
| `buildSrc` | Shared build logic as code |

### Convention Plugin (Advanced)

```
buildSrc/
├── build.gradle.kts
└── src/main/kotlin/
    └── my-java-conventions.gradle.kts
```

Then in subprojects:

```kotlin
plugins {
    id("my-java-conventions")
}
```

---

## 13. Properties & Profiles

### gradle.properties

```properties
org.gradle.jvmargs=-Xmx2g -XX:MaxMetaspaceSize=512m
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.daemon=true

myApp.version=1.0.0
springBootVersion=3.2.0
```

### Using in build.gradle.kts

```kotlin
version = property("myApp.version") as String
```

### Command-Line Properties

```bash
./gradlew build -Penv=prod
./gradlew build -Denv=prod         # system property
```

```kotlin
val env = project.findProperty("env") ?: "dev"
```

### Environment Variables

```kotlin
val ci = System.getenv("CI") == "true"
```

### Conditional Configuration

```kotlin
if (project.hasProperty("prod")) {
    dependencies {
        implementation("com.example:prod-lib:1.0")
    }
}
```

### Build Types (Android-style, not standard for Spring)

Gradle doesn't have Maven profiles. Use:
- Property-based conditionals
- Separate source sets
- Different build scripts
- `-P` flags

---

## 14. Common Commands

### Build

```bash
./gradlew build                 # compile + test + assemble
./gradlew assemble              # create artifacts
./gradlew check                 # run verification
./gradlew clean                 # delete build/
./gradlew clean build
./gradlew bootJar               # build fat JAR
./gradlew jar                   # build thin JAR
./gradlew bootRun               # run app
./gradlew bootBuildImage        # build container image
```

### Test

```bash
./gradlew test
./gradlew test --tests "*UserServiceTest"
./gradlew test --tests "com.example.UserServiceTest.findById"
./gradlew test -x testIntegration
```

### Dependencies

```bash
./gradlew dependencies
./gradlew dependencies --configuration runtimeClasspath
./gradlew dependencyInsight --dependency guava
./gradlew dependencyUpdates
```

### Tasks

```bash
./gradlew tasks                 # list tasks
./gradlew tasks --all
./gradlew help
./gradlew help --task bootRun
```

### Build Info

```bash
./gradlew buildEnvironment
./gradlew properties
./gradlew projects
```

### Performance

```bash
./gradlew build --parallel
./gradlew build --build-cache
./gradlew build --scan          # build scan (develocity)
./gradlew build --refresh-dependencies
```

### Wrapper Upgrade

```bash
./gradlew wrapper --gradle-version 8.6
```

### Debugging

```bash
./gradlew build --info
./gradlew build --debug
./gradlew build --stacktrace
./gradlew build --warning-mode all
```

### Clean Caches

```bash
./gradlew --stop                 # stop daemon
rm -rf ~/.gradle/caches
./gradlew clean build --refresh-dependencies
```

---

## 15. Interview Questions

### Q1. What is Gradle?

**Answer:** A modern build automation tool using Groovy or Kotlin DSL. Features: incremental builds, build cache, daemon, flexible task graph, multi-language support. Default for Android; fully supported by Spring Boot.

### Q2. Difference between Gradle and Maven?

**Answer:** Maven uses XML POM with fixed lifecycles. Gradle uses Groovy/Kotlin DSL with programmable build logic and a DAG task graph. Gradle is often faster due to daemon and incremental builds. Maven is simpler and more standardized.

### Q3. What is a Gradle task?

**Answer:** A unit of work in the build (compileJava, test, jar, bootJar). Tasks have inputs/outputs, dependencies, and actions. Task graph is dynamically built.

### Q4. What is the Gradle wrapper?

**Answer:** Scripts (`gradlew`, `gradlew.bat`) + JAR that download and run a pinned Gradle version. Ensures reproducible builds regardless of developer's global Gradle.

### Q5. Difference between implementation and api?

**Answer:** `implementation` hides the dependency from consumers (better encapsulation). `api` exposes it (consumers can use it transitively). Use `implementation` unless the type is part of your public API.

### Q6. What are dependency configurations?

**Answer:** Buckets grouping dependencies by usage: `implementation`, `api`, `compileOnly`, `runtimeOnly`, `testImplementation`, `annotationProcessor`, `developmentOnly`. Analogous to Maven scopes.

### Q7. What is the Spring Boot Gradle plugin?

**Answer:** Adds `bootJar` (executable fat JAR), `bootRun` (run app), `bootBuildImage` (container image), `bootBuildInfo` (build metadata). Also imports `spring-boot-dependencies` version management when combined with `io.spring.dependency-management`.

### Q8. How do you manage dependency versions?

**Answer:** Use `io.spring.dependency-management` plugin with `dependencyManagement { imports { mavenBom("...") } }`, or native Gradle `platform("group:artifact:version")`.

### Q9. How do you exclude a transitive dependency?

**Answer:**

```kotlin
implementation("org.springframework.boot:spring-boot-starter-web") {
    exclude(group = "org.springframework.boot", module = "spring-boot-starter-tomcat")
}
```

### Q10. How do you force a specific version?

**Answer:**

```kotlin
configurations.all {
    resolutionStrategy {
        force("com.google.guava:guava:32.1.3-jre")
    }
}
```

### Q11. Difference between build, assemble, and check?

**Answer:** `build` = `assemble` + `check`. `assemble` creates artifacts (JARs, WARs). `check` runs verification (tests, lint). So `build` produces artifacts and verifies them.

### Q12. How do you run a Spring Boot app?

**Answer:** `./gradlew bootRun`. Add `--args='--server.port=9090'` for args.

### Q13. How do you create a fat JAR?

**Answer:** `./gradlew bootJar` uses the Spring Boot plugin's `bootJar` task. Output: `build/libs/<name>-<version>.jar`.

### Q14. How do you run integration tests separately?

**Answer:** Create a custom `integrationTest` task of type `Test`, point at test class dirs, use JUnit tags to select. Hook into `check` via `dependsOn`.

### Q15. How do you enable parallel execution?

**Answer:** `./gradlew build --parallel` or `org.gradle.parallel=true` in `gradle.properties`.

### Q16. What is the Gradle daemon?

**Answer:** A long-lived background process that speeds up builds by caching JVM startup and in-memory state. Enabled by default. Stop with `./gradlew --stop`.

### Q17. What is the build cache?

**Answer:** A local/remote cache of task outputs keyed by inputs. If inputs haven't changed, tasks are skipped. Enable with `org.gradle.caching=true` or `--build-cache`.

### Q18. How do you customize the JAR name?

**Answer:**

```kotlin
tasks.named<BootJar>("bootJar") {
    archiveFileName.set("my-app.jar")
}
```

Or globally:

```kotlin
tasks.withType<Jar> {
    archiveFileName.set("${project.name}.jar")
}
```

### Q19. Groovy DSL vs Kotlin DSL?

**Answer:** Groovy DSL is terser, older, more examples. Kotlin DSL is type-safe, IDE-friendly, better errors — recommended for new projects. Choose one; don't mix within a project.

### Q20. How do you see why a dependency version was chosen?

**Answer:** `./gradlew dependencyInsight --dependency <artifact> --configuration <config>` shows the resolution reasoning.

### Q21. How do you build a container image with Gradle?

**Answer:** `./gradlew bootBuildImage` — uses Cloud Native Buildpacks to produce an optimized OCI image. Requires Docker.

### Q22. How do you share configuration across subprojects?

**Answer:** Use `allprojects`/`subprojects` in root build, or better, **convention plugins** in `buildSrc`. `allprojects` is discouraged due to configuration-time coupling.

### Q23. What is `gradle.properties`?

**Answer:** A file for Gradle properties — JVM args, daemon/parallel/cache toggles, and custom `key=value` properties. Found at project root or `~/.gradle/gradle.properties` (user-level).

### Q24. How do you migrate from Maven to Gradle?

**Answer:** Use `gradle init` on an existing Maven project. It reads `pom.xml` and generates equivalent `build.gradle(.kts)` and `settings.gradle(.kts)`. Verify tasks, dependencies, and plugins after conversion.

### Q25. What's the equivalent of Maven's `<dependencyManagement>`?

**Answer:** Either the `io.spring.dependency-management` plugin (recommended by Spring), or Gradle's native `platform(...)` / `enforcedPlatform(...)` in `dependencies` — the latter is the modern approach.

---

## 16. Cheat Sheet

### Minimal Spring Boot build.gradle.kts

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
}

group = "com.example"
version = "1.0.0"

java {
    sourceCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

### settings.gradle.kts

```kotlin
rootProject.name = "my-app"
```

### Configurations

```
implementation       runtime + compile (hidden)
api                  runtime + compile (exposed)
compileOnly          compile only
runtimeOnly          runtime only
testImplementation   test compile + runtime
testCompileOnly      test compile only
testRuntimeOnly      test runtime only
annotationProcessor  annotation processing
developmentOnly      dev-only (Spring Boot)
```

### Common Commands

```bash
./gradlew build               # full build
./gradlew bootRun             # run app
./gradlew bootJar             # fat JAR
./gradlew bootBuildImage      # container image
./gradlew test                # tests
./gradlew dependencies        # dep tree
./gradlew dependencyInsight --dependency guava
./gradlew tasks               # list tasks
./gradlew clean build
./gradlew build --parallel
./gradlew build --refresh-dependencies
```

### BOM (Native Gradle)

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.2.0"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    // version omitted — from BOM
}
```

### BOM (with plugin)

```kotlin
plugins {
    id("io.spring.dependency-management") version "1.1.4"
}

dependencyManagement {
    imports {
        mavenBom("org.springframework.boot:spring-boot-dependencies:3.2.0")
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.0")
    }
}
```

### Exclusion

```kotlin
implementation("org.springframework.boot:spring-boot-starter-web") {
    exclude(group = "org.springframework.boot", module = "spring-boot-starter-tomcat")
}
```

### Force Version

```kotlin
configurations.all {
    resolutionStrategy {
        force("com.google.guava:guava:32.1.3-jre")
    }
}
```

### Custom Test Task

```kotlin
tasks.register<Test>("integrationTest") {
    testClassesDirs = sourceSets["test"].output.classesDirs
    classpath = sourceSets["test"].runtimeClasspath
    useJUnitPlatform { includeTags("integration") }
}
tasks.named("check") { dependsOn("integrationTest") }
```

### gradle.properties

```properties
org.gradle.jvmargs=-Xmx2g
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.daemon=true
```

### Multi-Module

```kotlin
// settings.gradle.kts
include("module-a", "module-b")

// module-a/build.gradle.kts
plugins { id("org.springframework.boot") }
dependencies {
    implementation(project(":module-b"))
}
```

```bash
./gradlew :module-a:build
./gradlew build --parallel
```

### Maven vs Gradle Equivalents

| Maven | Gradle |
|-------|--------|
| `pom.xml` | `build.gradle.kts` |
| `<dependency>` | `implementation(...)` |
| `<scope>test</scope>` | `testImplementation(...)` |
| `<dependencyManagement>` | `platform(...)` / `dependencyManagement{}` |
| `<parent>` | convention plugin / root build |
| `<modules>` | `include(...)` |
| `<plugin>` | `plugins { id(...) }` |
| `<profiles>` | `-P` properties / conditionals |
| `mvn package` | `./gradlew build` |
| `mvn install` | `./gradlew publishToMavenLocal` |
| `mvn deploy` | `./gradlew publish` |

### Cross-References

- **Previous:** `17_Maven_Build_Tool.md`
- **Next:** `19_Spring_Microservices_Cloud.md`
- **Related:** `06_Spring_Boot_Fundamentals.md`
- **Interview:** `24_Spring_Interview_Questions.md` (§Maven)

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [17_Maven_Build_Tool.md](./17_Maven_Build_Tool.md)
- **Next →:** [19_Spring_Microservices_Cloud.md](./19_Spring_Microservices_Cloud.md)
- **Related:** [17_Maven_Build_Tool.md](./17_Maven_Build_Tool.md), [08_Spring_Boot_Configuration.md](./08_Spring_Boot_Configuration.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
