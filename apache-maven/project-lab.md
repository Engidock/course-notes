# ☕ Apache Maven Project Mastery

> **Hey Fresher — Read This First!**
>
> Maven is a build automation and dependency management tool for Java projects. Instead of manually downloading JAR files and writing custom scripts to compile, test, and package code, you describe your project — its dependencies, plugins, and build steps — in a single `pom.xml`, and Maven's standardized lifecycle (`validate → compile → test → package → verify → install → deploy`) does the rest, the same way on every developer's machine and every CI server.
>
> **ArenaForge Games** builds real-time multiplayer mobile games out of Bengaluru, and their matchmaking-service — the Java backend that pairs players into ranked matches within milliseconds — is compiled, tested, and packaged dozens of times a day as three different teams push changes. You're joining as a Backend Build Engineer. Your first task: the team's current `pom.xml` is a copy-paste from a five-year-old tutorial, dependency versions conflict silently, there's no separation between a dev build and the optimized JAR that actually ships to production game servers, and nobody's publishing artifacts anywhere teammates can pull them from. You're going to fix all of it, one Maven concept at a time.

#### What You Will Learn and Build in This Project

You will structure a proper multi-module Maven project for the matchmaking-service, manage dependencies and resolve version conflicts using scopes and a BOM, wire up the build lifecycle with unit and integration tests plus code coverage, create environment-specific build profiles, package a fat executable JAR with the Shade plugin, and publish versioned artifacts to a remote repository.

Maven POM Structure, Build Lifecycle and Phases, Dependency Scopes and Transitive Dependencies, Dependency Management with BOMs, Maven Profiles, Surefire and Failsafe Plugins, Code Coverage with JaCoCo, Shade Plugin for Fat JARs, Multi-Module Projects, Deploying Artifacts to a Remote Repository

> **📦 Phase 1 — Project Structure and the POM**
>
> Scaffold the matchmaking-service as a proper Maven project and understand what every core element of `pom.xml` actually controls.

> **📦 Phase 2 — Dependency Management and Scopes**
>
> Add real dependencies, understand transitive dependencies, and resolve a version conflict using `dependencyManagement` and a BOM.

> **📦 Phase 3 — Build Lifecycle, Testing and Coverage**
>
> Wire unit tests through Surefire, integration tests through Failsafe, and enforce a minimum code coverage threshold with JaCoCo.

> **📦 Phase 4 — Profiles for Dev and Production Builds**
>
> Create a `prod` profile that strips test scaffolding and enables JVM-level optimizations, separate from the default `dev` build.

> **📦 Phase 5 — Packaging an Executable Fat JAR**
>
> Use the Shade plugin to bundle the matchmaking-service and all its dependencies into one deployable JAR, and split the project into modules.

> **📦 Phase 6 — Publishing Artifacts to a Remote Repository**
>
> Configure `distributionManagement` and publish versioned JARs so other ArenaForge services can depend on the matchmaking client library.

**Scene 1 — ArenaForge Games, Bengaluru | The Five-Year-Old pom.xml**

> **Tanvi** _Junior Backend Build Engineer_
>
> I opened matchmaking-service's `pom.xml` and it declares Jackson 2.9 directly, but our shared game-events library pulls in Jackson 2.15 transitively. Maven picked 2.9 for the actual build, and that's why deserializing new event fields silently returns null.

> **Karthik** _Principal Backend Architect_
>
> That's a textbook Maven "nearest wins" dependency resolution problem. We need `dependencyManagement` to pin the version explicitly, ideally through a shared BOM, so every ArenaForge service that depends on game-events gets a consistent Jackson version without each team guessing.

> **Neeraj** _Senior Java Engineer_
>
> And matchmaking-service ships as one giant JAR to our game servers — no application server, no classpath assembly at deploy time. If we don't get the Shade plugin's merge strategy right, we'll get duplicate `META-INF/services` entries and Jackson module registration will break in the fat JAR even though it works fine when we run it from the IDE.

> **Tanvi**
>
> So: fix the dependency conflict properly, get real test gates in the lifecycle, separate a dev build from what actually ships, and package it correctly as one runnable JAR.

> **Karthik**
>
> Let's build it in that order.

### 1. Phase 1 — Project Structure and the POM

**Business Problem:** The current matchmaking-service isn't following Maven's standard directory layout, and the `pom.xml` is missing basic coordinates that other services need to depend on it correctly.

#### 1.1 Standard Maven Directory Layout

```
matchmaking-service/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/arenaforge/matchmaking/
│   │   │       ├── MatchmakingApplication.java
│   │   │       ├── engine/RankedMatchEngine.java
│   │   │       └── queue/PlayerQueueService.java
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       ├── java/
│       │   └── com/arenaforge/matchmaking/
│       │       └── engine/RankedMatchEngineTest.java
│       └── resources/
└── target/
```

> **📖 Why the layout matters**
> Maven follows "convention over configuration" — `src/main/java` for application code, `src/main/resources` for config files bundled into the artifact, `src/test/java` for test code, and `target/` for all build output (compiled classes, the packaged JAR, test reports). Because Maven already knows these paths, the POM doesn't need a single line configuring source directories — every plugin (compiler, Surefire, Shade) just works against the default layout, which is exactly why a Maven project someone else wrote is immediately navigable.

#### 1.2 The Core POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                              http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.arenaforge</groupId>
  <artifactId>matchmaking-service</artifactId>
  <version>1.4.0-SNAPSHOT</version>
  <packaging>jar</packaging>

  <properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

</project>
```

> **📖 What each coordinate means**
> `groupId` (`com.arenaforge`) namespaces every ArenaForge artifact, mirroring reverse-domain convention. `artifactId` (`matchmaking-service`) is this specific project's name. `version` combined with `groupId:artifactId` uniquely identifies this exact build — `1.4.0-SNAPSHOT` marks it as an in-development version that can be rebuilt and republished repeatedly, as opposed to `1.4.0`, an immutable release version Maven will refuse to silently overwrite in a remote repository. `packaging: jar` tells Maven the default lifecycle should produce a JAR file (as opposed to `war`, `pom` for an aggregator module, etc.). `maven.compiler.source/target` fix the Java language level so the whole team and CI compile against Java 17 consistently, regardless of which JDK happens to be on a given machine's PATH.

**Quiz: What is the practical difference between a version like `1.4.0-SNAPSHOT` and `1.4.0` in Maven?**
- SNAPSHOT versions are always smaller JAR files
- SNAPSHOT is a mutable, in-development version that Maven will re-resolve and re-download when changed; a plain version like 1.4.0 is treated as an immutable release
- SNAPSHOT versions cannot have dependencies
- There is no functional difference, only a naming convention

> **Answer/explanation:** The correct answer is the second option. A `-SNAPSHOT` suffix tells Maven this artifact is a moving target — Maven will check the remote repository for a newer snapshot build on each resolution (subject to update policy) rather than assuming a cached copy is final. A release version like `1.4.0` is treated as immutable: once published, Maven repositories are expected to reject attempts to overwrite it, and consumers can trust that `1.4.0` today is identical to `1.4.0` next year. JAR size and dependency capability are unrelated to the SNAPSHOT suffix.

### 2. Phase 2 — Dependency Management and Scopes

**Business Problem:** The exact conflict Tanvi found in the opening scene — matchmaking-service pins Jackson 2.9 while a transitive dependency wants 2.15 — needs a real fix, not a guess, plus a way to make sure it never silently regresses again.

#### 2.1 Diagnosing the Conflict

```bash
mvn dependency:tree -Dincludes=com.fasterxml.jackson.core
```

```
[INFO] com.arenaforge:matchmaking-service:jar:1.4.0-SNAPSHOT
[INFO] +- com.fasterxml.jackson.core:jackson-databind:jar:2.9.10.8:compile
[INFO] \- com.arenaforge:game-events:jar:3.2.0:compile
[INFO]    \- com.fasterxml.jackson.core:jackson-databind:jar:2.15.2:compile (omitted for conflict with 2.9.10.8)
```

> **📖 Reading `dependency:tree` output**
> This confirms exactly what Tanvi suspected: matchmaking-service directly declares `jackson-databind:2.9.10.8`, and the `game-events` library transitively pulls in `2.15.2`. Maven's default conflict resolution is "nearest definition wins" — since the direct dependency (distance 1) is nearer than the transitive one (distance 2 via game-events), Maven picks 2.9.10.8, and the line explicitly says `(omitted for conflict with 2.9.10.8)`. This is silent by default — nothing fails the build, it just quietly uses the older version, which is exactly how this bug went unnoticed.

#### 2.2 Fixing It with `dependencyManagement`

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.15.2</version>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
  <dependency>
    <groupId>com.arenaforge</groupId>
    <artifactId>game-events</artifactId>
    <version>3.2.0</version>
  </dependency>
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

> **📖 dependencyManagement vs. dependencies**
> `<dependencyManagement>` doesn't add anything to the classpath by itself — it declares *which version to use if and when* an artifact is requested, either directly or transitively. Notice the actual `<dependency>` entry for `jackson-databind` under `<dependencies>` has no `<version>` — it inherits `2.15.2` from the management block. Critically, this also overrides the transitive version pulled in from `game-events`, because explicit `dependencyManagement` entries win regardless of dependency distance. This is the correct, durable fix — as opposed to just bumping the direct version number, which would silently drift again the next time someone adds a new transitive dependency with its own Jackson opinion.

#### 2.3 Understanding Dependency Scopes

```xml
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter</artifactId>
  <version>5.10.2</version>
  <scope>test</scope>
</dependency>

<dependency>
  <groupId>jakarta.servlet</groupId>
  <artifactId>jakarta.servlet-api</artifactId>
  <version>6.0.0</version>
  <scope>provided</scope>
</dependency>

<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <version>42.7.3</version>
  <scope>runtime</scope>
</dependency>
```

**Dependency Scope Comparison**

- **compile (default)** — available at compile time and packaged into the final artifact; used for libraries the code directly imports and needs at runtime, like `game-events`.
- **test** — only available when compiling and running tests, never packaged into the shipped JAR; JUnit belongs here so it never ends up in production.
- **provided** — available at compile time but expected to be supplied by the runtime environment, so it is not packaged; used when a container or platform already provides the dependency.
- **runtime** — not needed to compile the code, but required when running it; the PostgreSQL JDBC driver is a classic example — matchmaking-service's code talks to `java.sql.Connection` interfaces at compile time, and only needs the concrete driver JAR present at runtime.

> **Key takeaways**
> - `mvn dependency:tree` reveals transitive dependencies and shows exactly which version "won" and why, including omitted conflicts.
> - Maven's default conflict resolution is nearest-definition-wins, which can silently pick an outdated version — `dependencyManagement` pins the version explicitly regardless of dependency distance.
> - Scopes (`compile`, `test`, `provided`, `runtime`) control both what's available at which build phase and, crucially, what actually ships inside the packaged JAR.

### 3. Phase 3 — Build Lifecycle, Testing and Coverage

**Business Problem:** `mvn package` currently succeeds even when unit tests fail to compile properly in edge cases, and there's no integration test step at all for `PlayerQueueService`, which talks to Redis and needs a real connection to test meaningfully — plus no visibility into how much of the matchmaking logic is actually covered by tests.

#### 3.1 The Default Lifecycle and Where Plugins Bind

```bash
mvn clean test
mvn clean verify
```

> **📖 The lifecycle phases that matter here**
> Maven's default lifecycle runs phases in a fixed order: `validate → compile → test → package → verify → install → deploy`. Running `mvn clean verify` executes every phase up through `verify`, which is why it's the standard command in CI — it compiles, unit tests, packages, then runs any checks bound to `verify` (like integration tests and coverage enforcement), without publishing anything anywhere yet. Each plugin goal binds to a specific phase; Surefire's `test` goal binds to the `test` phase by default, which is what runs whenever you call `mvn test` or any later phase.

#### 3.2 Unit Tests with Surefire, Integration Tests with Failsafe

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.2.5</version>
      <configuration>
        <includes>
          <include>**/*Test.java</include>
        </includes>
      </configuration>
    </plugin>

    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-failsafe-plugin</artifactId>
      <version>3.2.5</version>
      <executions>
        <execution>
          <goals>
            <goal>integration-test</goal>
            <goal>verify</goal>
          </goals>
        </execution>
      </executions>
      <configuration>
        <includes>
          <include>**/*IT.java</include>
        </includes>
      </configuration>
    </plugin>
  </plugins>
</build>
```

> **📖 Why two test plugins, not one**
> Surefire runs classes named `*Test.java` during the `test` phase — fast, isolated unit tests like `RankedMatchEngineTest`, which don't touch a network or database. Failsafe runs classes named `*IT.java` (integration test) during the separate `integration-test` and `verify` phases — this is where `PlayerQueueServiceIT` lives, exercising a real Redis connection (typically via Testcontainers). The split matters because Failsafe's `verify` goal runs *after* integration tests execute, checking their results and failing the build only then — unlike Surefire, which would fail immediately mid-phase, Failsafe lets you run cleanup (tearing down a test container) between the test run and the pass/fail decision.

#### 3.3 Enforcing Coverage with JaCoCo

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.12</version>
  <executions>
    <execution>
      <goals>
        <goal>prepare-agent</goal>
      </goals>
    </execution>
    <execution>
      <id>check-coverage</id>
      <phase>verify</phase>
      <goals>
        <goal>check</goal>
      </goals>
      <configuration>
        <rules>
          <rule>
            <element>PACKAGE</element>
            <limits>
              <limit>
                <counter>LINE</counter>
                <minimum>0.75</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

> **📖 What the coverage gate does**
> `prepare-agent` attaches a Java agent to the JVM running Surefire's tests, so JaCoCo can instrument bytecode and record which lines actually execute. The `check` goal, bound to `verify`, then fails the build if the `engine` package's line coverage falls under `0.75` (75%) — meaning `mvn clean verify` for matchmaking-service now genuinely fails, not just reports a low number, if a change to `RankedMatchEngine` isn't adequately tested. This turns "we should have good test coverage" from a wish into an enforced build gate every engineer hits locally, before code review.

**Quiz: Why does matchmaking-service split tests between `*Test.java` (Surefire) and `*IT.java` (Failsafe) instead of running everything through Surefire?**
- Failsafe runs tests faster than Surefire
- Integration tests need a real dependency like Redis to be up, and Failsafe's separate verify goal lets teardown happen between running the tests and deciding pass/fail, unlike Surefire which fails immediately in the test phase
- Surefire cannot run any test that uses assertions
- JUnit only works with Failsafe, not Surefire

> **Answer/explanation:** The correct answer is the second option. Integration tests like `PlayerQueueServiceIT` depend on external state (a running Redis instance, often started via Testcontainers) that needs setup before and teardown after the test run — Failsafe's `integration-test`/`verify` split allows that teardown to run regardless of test outcome, before the build decides to fail. Surefire, by contrast, fails the build the moment a unit test fails, with no room for a cleanup phase in between. Failsafe isn't inherently faster, Surefire runs assertion-based tests just fine, and both plugins work with JUnit.

### 4. Phase 4 — Profiles for Dev and Production Builds

**Business Problem:** Right now, every build — whether Tanvi is iterating locally or the CI system is producing the JAR that ships to game servers — runs identically. Karthik wants a production build that skips slow integration tests and turns on JVM-level optimizations, without maintaining two separate POMs.

#### 4.1 Defining the `prod` Profile

```xml
<profiles>
  <profile>
    <id>dev</id>
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
    <properties>
      <skipITs>true</skipITs>
      <maven.compiler.debug>true</maven.compiler.debug>
    </properties>
  </profile>

  <profile>
    <id>prod</id>
    <properties>
      <skipITs>false</skipITs>
      <maven.compiler.debug>false</maven.compiler.debug>
      <maven.compiler.optimize>true</maven.compiler.optimize>
    </properties>
    <build>
      <plugins>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-failsafe-plugin</artifactId>
          <configuration>
            <skipITs>${skipITs}</skipITs>
          </configuration>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```

> **📖 How activation and overrides work**
> The `dev` profile has `activeByDefault: true`, so a plain `mvn package` on a developer's laptop uses it automatically — fast local builds, debug symbols included, integration tests skipped. The `prod` profile is only applied when explicitly requested with `-P prod`, and the moment it activates, Maven's rule is that any explicitly-activated profile disables profiles marked `activeByDefault`, so `dev`'s settings don't leak in. `prod` flips `skipITs` to `false` (integration tests run, because this is the build that actually ships) and turns off debug symbols while enabling compiler optimization flags, producing a smaller, faster runtime artifact for the game servers.

#### 4.2 Running Each Profile

```bash
# Local dev build — fast, skips integration tests
mvn clean package

# Production build — runs everything, ships-optimized
mvn clean verify -P prod
```

**Dev Profile vs. Prod Profile**

- **dev (default)** — integration tests skipped, debug symbols on; optimized for fast local iteration, what Tanvi runs dozens of times a day.
- **prod** — integration tests run, debug symbols off, compiler optimization on; this is the only build configuration Karthik allows to be promoted to a release artifact that reaches production game servers.

> **Key takeaways**
> - `mvn clean verify` runs the full lifecycle through `verify`, the standard command for CI because it includes packaging and integration testing.
> - Surefire owns unit tests bound to the `test` phase; Failsafe owns integration tests bound to `integration-test`/`verify`, allowing teardown between test execution and pass/fail evaluation.
> - Profiles let one POM produce meaningfully different builds (`dev` vs `prod`) via `-P`, without maintaining separate POM files.

### 5. Phase 5 — Packaging an Executable Fat JAR

**Business Problem:** Matchmaking-service runs directly on ArenaForge's game servers with `java -jar matchmaking-service.jar` — there's no application server to provide the classpath, so every dependency needs to be bundled into one self-contained JAR, correctly, including merged service-provider files that Jackson relies on for module registration.

#### 5.1 Configuring the Shade Plugin

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-shade-plugin</artifactId>
  <version>3.5.3</version>
  <executions>
    <execution>
      <phase>package</phase>
      <goals>
        <goal>shade</goal>
      </goals>
      <configuration>
        <transformers>
          <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
            <mainClass>com.arenaforge.matchmaking.MatchmakingApplication</mainClass>
          </transformer>
          <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer" />
        </transformers>
        <filters>
          <filter>
            <artifact>*:*</artifact>
            <excludes>
              <exclude>META-INF/*.SF</exclude>
              <exclude>META-INF/*.DSA</exclude>
              <exclude>META-INF/*.RSA</exclude>
            </excludes>
          </filter>
        </filters>
      </configuration>
    </execution>
  </executions>
</plugin>
```

> **📖 The two transformers that actually matter here**
> `ManifestResourceTransformer` writes a `Main-Class` entry into the fat JAR's manifest, which is what lets `java -jar matchmaking-service.jar` know to start `MatchmakingApplication.main()` without needing `-cp` and a fully qualified class name on the command line. `ServicesResourceTransformer` is the fix for exactly the bug Neeraj flagged in the opening scene: Jackson (and many other libraries) use Java's `META-INF/services/` service-loader mechanism to register modules, and when multiple dependency JARs each ship their own `META-INF/services/com.fasterxml.jackson.databind.Module` file, a naive merge would let one silently overwrite the others. `ServicesResourceTransformer` concatenates them correctly into one merged file, so every module from every dependency registers properly in the shaded JAR. The `filters` block strips signature files (`.SF`, `.DSA`, `.RSA`) from signed dependency JARs — without this, the shaded JAR fails at runtime because its merged contents no longer match the original signatures.

#### 5.2 Building and Running the Fat JAR

```bash
mvn clean package -P prod

ls -lh target/matchmaking-service-1.4.0-SNAPSHOT.jar
# -rw-r--r-- 1 tanvi staff 48M matchmaking-service-1.4.0-SNAPSHOT.jar

java -jar target/matchmaking-service-1.4.0-SNAPSHOT.jar
# Matchmaking service started on port 8081
```

**Shade Plugin vs. Assembly Plugin**

- **maven-shade-plugin** — merges dependency classes directly into the output JAR and understands content-aware merging (like the Services transformer above); ArenaForge uses this because matchmaking-service depends on multiple libraries that ship `META-INF/services` files that need correct merging, not just concatenation-by-luck.
- **maven-assembly-plugin** — more general-purpose packaging (zip, tar.gz, or fat JAR via a descriptor), simpler for basic fat-JAR needs, but its default `jar-with-dependencies` merge is naive file-overwrite rather than content-aware — it would silently break Jackson module registration the way ArenaForge's team hit before switching to Shade.

#### 5.3 Splitting into a Multi-Module Project

```xml
<!-- parent pom.xml -->
<groupId>com.arenaforge</groupId>
<artifactId>matchmaking-parent</artifactId>
<version>1.4.0-SNAPSHOT</version>
<packaging>pom</packaging>

<modules>
  <module>matchmaking-core</module>
  <module>matchmaking-service</module>
</modules>
```

> **Why split at all** — `matchmaking-core` holds the matchmaking algorithm and domain model with no networking or Redis dependency, so it can be unit tested in isolation and reused by an offline simulation tool the game-design team runs to tune ranking parameters. `matchmaking-service` depends on `matchmaking-core` and adds the Redis queue, REST API, and everything that gets shaded into the deployable JAR. The parent POM's `packaging: pom` marks it as an aggregator with no code of its own — running `mvn install` from the parent directory builds both modules in dependency order automatically.

### 6. Phase 6 — Publishing Artifacts to a Remote Repository

**Business Problem:** `matchmaking-core` is genuinely useful to the game-design team's offline simulation tool, but right now the only way to get it is copying a JAR over Slack. ArenaForge needs it published somewhere other projects can declare as a normal Maven dependency.

#### 6.1 Configuring `distributionManagement`

```xml
<distributionManagement>
  <repository>
    <id>arenaforge-releases</id>
    <url>https://nexus.arenaforge.internal/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>arenaforge-snapshots</id>
    <url>https://nexus.arenaforge.internal/repository/maven-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```

```xml
<!-- ~/.m2/settings.xml -->
<settings>
  <servers>
    <server>
      <id>arenaforge-releases</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>
    <server>
      <id>arenaforge-snapshots</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>
  </servers>
</settings>
```

> **📖 Why credentials live in settings.xml, not the POM**
> `distributionManagement` in `pom.xml` says *where* to publish and is safe to commit — it contains no secrets. The matching `<id>` values in `~/.m2/settings.xml` (kept outside the repo, per-machine or injected by CI as a secret file) supply the actual credentials. Maven matches repository `id` to server `id` to decide which credentials to use for which URL. This split is what lets the exact same `pom.xml` be published from a developer's laptop and from a CI runner, each authenticating with its own credentials, without any secret ever touching version control.

#### 6.2 Publishing with the Deploy Plugin

```bash
mvn clean deploy -P prod
```

```
[INFO] Uploading to arenaforge-releases: https://nexus.arenaforge.internal/repository/maven-releases/com/arenaforge/matchmaking-core/1.4.0/matchmaking-core-1.4.0.jar
[INFO] Uploaded to arenaforge-releases: matchmaking-core-1.4.0.jar (112 kB)
[INFO] BUILD SUCCESS
```

> **`mvn deploy`** is the final phase of Maven's default lifecycle — it runs everything before it (compile, test, package, verify, install) and then uploads the resulting artifact to the repository configured in `distributionManagement`. Because `matchmaking-core`'s version is `1.4.0` (no `-SNAPSHOT`), it goes to `maven-releases`, and Nexus will reject a second deploy attempt at the same version — release coordinates are immutable by policy. A `-SNAPSHOT` version instead goes to `maven-snapshots`, which does allow repeated overwriting.

#### 6.3 Consuming the Published Artifact from Another Project

```xml
<!-- game-design-sim/pom.xml -->
<dependency>
  <groupId>com.arenaforge</groupId>
  <artifactId>matchmaking-core</artifactId>
  <version>1.4.0</version>
</dependency>
```

```xml
<repositories>
  <repository>
    <id>arenaforge-releases</id>
    <url>https://nexus.arenaforge.internal/repository/maven-releases/</url>
  </repository>
</repositories>
```

> **From Slack JAR to real dependency** — the game-design simulation tool now declares `matchmaking-core` exactly like any third-party library. `mvn install` on that project resolves it from ArenaForge's Nexus, downloads it once into the local `~/.m2/repository` cache, and every subsequent build reuses the cached copy. When Tanvi ships a matchmaking-core bug fix as `1.4.1`, the game-design team bumps one version number to pick it up — no more manually re-copying JARs.

> **Key takeaways**
> - `distributionManagement` declares publish targets in the POM; actual credentials live in `~/.m2/settings.xml`, matched by repository `id`, keeping secrets out of version control.
> - Release versions (no `-SNAPSHOT`) are immutable once published; snapshot versions can be overwritten repeatedly.
> - `mvn deploy` runs the entire lifecycle up through `install` first, then uploads — a broken build never gets published.
> - Publishing internal libraries to a shared repository turns "copy the JAR over Slack" into a normal, versioned dependency other teams can consume.

##### 🏋️ Hands-On Exercises — Extend the Project

1. Add a `staging` profile alongside `dev` and `prod` that runs integration tests but keeps debug symbols on, and confirm with `mvn help:active-profiles -P staging` that only `staging` is active.
2. Introduce a genuine version conflict on purpose (add a dependency that transitively pulls a different `slf4j-api` version), observe it with `mvn dependency:tree -Dverbose`, then fix it with `dependencyManagement` and confirm the conflict warning disappears.
3. Add a `maven-enforcer-plugin` rule that fails the build if `mvn` is run with a JDK older than 17, protecting against a developer accidentally building with an outdated local JDK.
4. Split `matchmaking-service` further by extracting `matchmaking-queue` (the Redis-facing code) into its own module under the parent POM, and update `matchmaking-service` to depend on both `matchmaking-core` and `matchmaking-queue`.
5. Configure the JaCoCo coverage rule from Phase 3 to apply per-class instead of per-package, with a stricter 90% threshold specifically for the `RankedMatchEngine` class, and verify the build fails when you temporarily comment out one of its tests.

### Apache Maven Project Complete 🎉

You have taken matchmaking-service from a copy-pasted, dependency-conflicted `pom.xml` to a properly structured, multi-module Maven project: dependency versions pinned deliberately through `dependencyManagement`, unit and integration tests gated through Surefire and Failsafe with an enforced JaCoCo coverage threshold, separate `dev` and `prod` build profiles, a correctly shaded fat JAR with merged service-provider files, and a reusable `matchmaking-core` library published to a real remote repository other ArenaForge teams can depend on.

> **Tanvi** _Junior Backend Build Engineer_
>
> The Jackson bug that took me two days to find in the first place would never happen now — `dependencyManagement` pins the version, and `dependency:tree` shows exactly what resolves where, in seconds.

> **Neeraj** _Senior Java Engineer_
>
> The Shade plugin's Services transformer fixed the exact class of bug I was worried about. Every dependency's Jackson module registers correctly in the shaded JAR now, because the merge is content-aware instead of last-file-wins.

> **Karthik** _Principal Backend Architect_
>
> And matchmaking-core being a real published artifact means the game-design team isn't three Slack messages behind our latest ranking algorithm anymore. They bump a version number and get it.

> **Next: Wiring this build into a real CI/CD pipeline**
>
> - A CI system (Jenkins or GitHub Actions) running `mvn clean verify -P prod` on every pull request, with JaCoCo coverage reported back on the PR.
> - Publishing every `main`-branch build automatically to Nexus with `mvn deploy`, keyed to a CI-managed version bump.
> - Static analysis integration (SonarQube) bound into the `verify` phase alongside JaCoCo, so code smells and vulnerabilities are gated the same way test coverage is.
