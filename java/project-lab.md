# ☕ Java for DevOps — UrbanNest Project

> **👋 Hey Fresher — Read This First!**

> - Java is the **most widely used backend language** in Indian enterprises — TCS, Infosys, Wipro, banks, and most product companies use Java with Spring Boot for their backend services.
> - This guide is **not a Java textbook**. Every concept is tied directly to where it appears in a DevOps/cloud tool stack — Jenkins, AKS, Azure DevOps, Docker, SonarQube.
> - Every code block is **short and focused** — only the essential snippet. Every explanation is **bullet points only** — no paragraphs.
> - **Company in this project:** UrbanNest — a real-estate platform in Pune. Their backend is built in Java with Spring Boot, deployed on AKS via an Azure DevOps CI/CD pipeline. You just joined as a Junior DevOps Engineer and need to understand the Java codebase to manage the pipeline effectively.

#### What You Will Learn in This Module

You will understand Java's role inside UrbanNest's complete DevOps stack — from writing a class to deploying it on AKS.

OOP, Collections, Streams, Exceptions, Threads, JDBC, Spring Boot, Maven, JUnit, Docker, Logging

> **🏗️ Phase 1 — Java Fundamentals**

> OOP, Collections, Streams, Lambdas, Exceptions, Threads. The building blocks used in every Spring Boot service.

> JDBC, Spring Boot REST APIs, Maven build system. How UrbanNest's backend is structured and built.

> JUnit in CI, Logging to Azure Monitor, Dockerising a Java app, deploying to AKS via Azure DevOps.

### 1. OOP — Classes & Objects

> **Why OOP Matters in UrbanNest's Stack**

> - UrbanNest's backend models real-world entities — **Property, Tenant, Agent, Booking**
> - Java OOP maps these directly into code that Spring Boot, JPA, and the REST API layer can work with
> - Every Spring Boot `@Entity` is a Java class. Every REST response is a Java object serialised to JSON
> - Understanding classes and objects is understanding how the entire Spring Boot application is structured

#### 1.1 Creating a Class and Object

```
class Property {
    String name;
    double price;

    // Constructor — runs when object is created
Property(String name, double price) {
        this.name  = name;
        this.price = price;
    }

    void display() {
        System.out.println(name + " — ₹" + price);
    }
}

// Create object (instance)
Property p = new Property("Hitech City Flat", 8500000);
p.display();
```

**📖 How It Works**

- **class** — Blueprint. Defines fields (data) and methods (behaviour)
- **new** — Creates an actual object in memory from the blueprint
- **Constructor** — Special method that runs automatically when object is created, sets initial values
- **this** — Refers to the current object's own fields, not the parameter
- **void** — Method returns nothing; just performs an action

```
// Output:
Hitech City Flat — ₹8500000.0
```

> **🔧 Where it's used in UrbanNest's Stack**

> - Spring Boot maps HTTP request body → Java object (POJO) using Jackson library automatically
> - JPA/Hibernate uses these classes as `@Entity` models mapped to Azure SQL Database tables
> - Jenkins pipeline utility scripts call Java classes for build notifications and artifact management

### 2. Inheritance & Polymorphism

> **Why Inheritance Matters in UrbanNest's Stack**

> - UrbanNest has multiple listing types — **Residential, Commercial, Plot** — each sharing common fields but with unique behaviour
> - Inheritance avoids duplicating code across all listing types
> - Spring Boot controllers use base controller classes for shared auth and logging logic
> - Custom exception classes extend `RuntimeException` — this is inheritance in daily DevOps work

#### 2.1 extends & @Override

```
class Listing {
    String city;
    double price;

    void info() {
        System.out.println("Listing in " + city);
    }
}

class ResidentialListing extends Listing {
    @Override
void info() {
        System.out.println("Home listing in " + city
            + " at ₹" + price);
    }
}

Listing l = new ResidentialListing();
l.city  = "Hyderabad";
l.price = 9500000;
l.info();
```

**📖 How It Works**

- **extends** — Child class inherits all fields and methods from parent
- **@Override** — Child redefines a parent method with its own logic; annotation makes intent clear
- **Polymorphism** — Variable type is `Listing` but object is `ResidentialListing`; Java calls the child's version of `info()`
- Parent fields (`city`, `price`) are accessible in child automatically
- Enables **open/closed principle** — extend without modifying base code

```
// Output:
Home listing in Hyderabad at ₹9500000.0
```

> **🔧 Where it's used in UrbanNest's Stack**

> - Spring Boot controllers extend a `BaseController` for shared request logging and auth token validation
> - Custom exception `PropertyNotFoundException extends RuntimeException` — used daily in service layers
> - JUnit test classes override `@BeforeEach` setup from base test classes shared across all test suites

### 3. Interfaces & Abstraction

> **Why Interfaces Matter in UrbanNest's Stack**

> - UrbanNest sends alerts via **Email, SMS, and WhatsApp** — same contract, different implementation
> - Interfaces enforce the contract: every notifier *must* have a `send(message)` method
> - Spring DI injects an interface; Spring picks the right implementation at runtime — this is how all Spring beans work
> - Spring Data JPA repositories are *interfaces* — you never write the implementation, Spring generates it

#### 3.1 Defining and Implementing an Interface

```
interface Notifier {
    void send(String message);
}

class EmailNotifier implements Notifier {
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}

class SMSNotifier implements Notifier {
    public void send(String message) {
        System.out.println("SMS: " + message);
    }
}

// Swap implementation without changing caller
Notifier n = new EmailNotifier();
n.send("New property in Banjara Hills!");
```

**📖 How It Works**

- **interface** — Defines method signatures only; no implementation code
- **implements** — Class agrees to provide all methods declared in the interface
- If a class implements the interface but misses a method, Java *won't compile* — enforced contract
- You can swap `EmailNotifier` → `SMSNotifier` without changing any calling code
- Spring uses this exact pattern — you `@Autowired` an interface, Spring injects the right bean

> **🔧 Where it's used in UrbanNest's Stack**

> - Spring Data JPA: `interface PropertyRepository extends JpaRepository<Property, Long>` — zero implementation code needed
> - Spring `@Service` beans implement interfaces; `@Autowired` injects the interface type, enabling easy mocking in JUnit
> - SonarQube quality rules check interface compliance and flag classes missing required contract methods

### 4. Collections — List, Map, Set

> **Why Collections Matter in UrbanNest's Stack**

> - UrbanNest's API returns **lists of properties**, maps **agent IDs to their listings**, and ensures **no duplicate booking IDs**
> - Collections are used in every Spring controller response, every service method, and every Jenkins pipeline Groovy script
> - When Spring Boot serialises a `List<Property>` to JSON — that's a Collection becoming your API response

#### 4.1 ArrayList, HashMap, HashSet

```
// ArrayList — ordered, allows duplicates
List<String> cities = new ArrayList<>();
cities.add("Hyderabad");
cities.add("Bengaluru");
cities.add("Hyderabad"); // duplicate allowed
// HashMap — key-value store
Map<Integer, String> agents = new HashMap<>();
agents.put(101, "Ravi Kumar");
agents.put(102, "Priya Sharma");
System.out.println(agents.get(101)); // Ravi Kumar
// HashSet — no duplicates
Set<String> bookingIds = new HashSet<>();
bookingIds.add("BK-001");
bookingIds.add("BK-001"); // silently ignored
System.out.println(bookingIds.size()); // 1
```

**📖 How It Works**

- **ArrayList** — ordered list; fast for getting by index; allows duplicates; use for property result lists
- **HashMap** — key → value lookup; O(1) average access; use to map IDs to objects
- **HashSet** — unique values only; automatically ignores duplicates on `add()`
- Always declare using the **interface type** (`List`, `Map`, `Set`) — not the implementation — for flexibility
- Use **diamond operator** `<>` — Java infers the generic type automatically

> **🔧 Where it's used in UrbanNest's Stack**

> - Spring Boot REST controllers return `List<Property>` — Jackson serialises it to a JSON array automatically
> - Jenkins Groovy pipeline scripts use Maps extensively: `def params = [env: 'prod', region: 'centralindia']`
> - Azure Functions queue triggers receive messages as a List; Java processes each item using Collections iteration

### 5. Streams & Lambda Expressions

> **Why Streams Matter in UrbanNest's Stack**

> - UrbanNest's search API must **filter** properties by price, **sort** by area, and **return top 5**
> - Streams do this in one readable pipeline — no for-loops, no temporary lists
> - Spring Boot service layers use Streams to process database results before returning API responses
> - SonarQube code quality rules *prefer* Streams over traditional loops — impacts your quality gate score

#### 5.1 filter, map, sorted, collect

```
List<Integer> prices = List.of(
    5000000, 8500000, 3200000, 12000000, 7800000
);

List<Integer> result = prices.stream()
    .filter(p -> p < 9000000)   // keep under 90L
    .sorted()                      // lowest first
    .limit(3)                     // top 3 only
    .collect(Collectors.toList()); // back to List

System.out.println(result);
```

**📖 How It Works**

- **stream()** — Opens a pipeline on the collection; doesn't modify original
- **filter(p → p < 9000000)** — Lambda: keeps only items where the condition is true
- **sorted()** — Natural order; use `sorted(Comparator.reverseOrder())` for descending
- **limit(3)** — Takes only the first N items from the stream
- **collect()** — Terminal operation; ends the pipeline and produces a result

```
// Output:
[3200000, 5000000, 7800000]
```

> **🔧 Where it's used in UrbanNest's Stack**

> - Spring Boot service: `properties.stream().filter(p -> p.getCity().equals(city)).collect(...)` — used in every search endpoint
> - Azure Functions process blob event payloads using Streams to transform and filter document metadata
> - SonarQube Java rules flag traditional for-loops that can be replaced with Streams — affects quality gate pass/fail

### 6. Exception Handling

> **Why Exception Handling Matters in UrbanNest's Stack**

> - UrbanNest's API must return a **clean 404 JSON response** when a property ID doesn't exist — not a 500 stack trace
> - Custom exceptions + Spring's `@ControllerAdvice` map Java exceptions → HTTP status codes automatically
> - Jenkins pipeline Groovy scripts use `try-catch` to handle deployment failures and send Slack alerts
> - Azure Functions wrap blob processing in `try-finally` to always close streams even on error

#### 6.1 Custom Exception + try-catch

```
// 1. Define custom exception
class PropertyNotFoundException
extends RuntimeException {
    public PropertyNotFoundException(int id) {
        super("Property not found: ID " + id);
    }
}

// 2. Throw in service layer
Property find(int id) {
    return repo.findById(id)
        .orElseThrow(() ->
            new PropertyNotFoundException(id));
}

// 3. Catch anywhere above
try {
    find(999);
} catch (PropertyNotFoundException e) {
    System.out.println(e.getMessage());
}
```

**📖 How It Works**

- **RuntimeException** — Unchecked; no need to declare it with `throws` — simpler for service code
- **super(message)** — Passes error message to parent Exception class
- **orElseThrow()** — Stream-style: if Optional is empty, throw the given exception
- **try-catch** — Wraps risky code; catches only the specific exception type declared
- **finally** — Runs whether or not exception occurred; use for closing DB connections or file streams

> **🔧 Where it's used in UrbanNest's Stack**

> - Spring Boot `@ControllerAdvice` catches `PropertyNotFoundException` and returns `HTTP 404` JSON — no stack traces exposed
> - Jenkins Groovy pipeline: `try { sh 'kubectl apply -f deploy.yaml' } catch(e) { slackSend "Deploy failed: ${e}" }`
> - Azure Functions: `try-finally` ensures blob InputStream is always closed after processing, even on error

### 7. Multithreading & Concurrency

> **Why Threads Matter in UrbanNest's Stack**

> - UrbanNest processes 200 PDF uploads simultaneously — sequential processing would take 10x longer
> - Spring Boot's embedded **Tomcat handles each HTTP request in its own thread** — understanding threads helps tune the thread pool
> - Azure Functions scale out parallel instances; Java code inside must be **thread-safe** (no shared mutable state)
> - Jenkins parallel stages run Java-based build steps concurrently across multiple agents

#### 7.1 ExecutorService Thread Pool

```
// Fixed thread pool — 5 threads max
ExecutorService pool =
    Executors.newFixedThreadPool(5);

Runnable processDoc = () -> {
    System.out.println("Processing PDF in: "
        + Thread.currentThread().getName());
    // PDF extraction logic here
};

// Submit 10 tasks — pool queues the rest
for (int i = 0; i < 10; i++) {
    pool.submit(processDoc);
}
pool.shutdown(); // no new tasks; finish existing
```

**📖 How It Works**

- **newFixedThreadPool(5)** — Creates exactly 5 threads; extra tasks wait in a queue
- **Runnable** — A task with no return value; written as a lambda `() → { }`
- **submit()** — Adds task to pool; a free thread picks it up immediately
- **shutdown()** — Stops accepting new tasks; waits for running tasks to finish
- Use **Callable + Future** when you need the result back from the background task

> **🔧 Where it's used in UrbanNest's Stack**

> - Spring Boot Tomcat thread pool config: `server.tomcat.threads.max=200` in `application.yml` — each HTTP request uses one thread
> - Azure Functions: multiple function instances run in parallel; Java code must not use static mutable variables
> - Jenkins parallel pipeline: `parallel { stage('Test') {...} stage('SonarQube') {...} }` runs Java build steps concurrently

### 8. Environment Variables & File I/O

> **Why Env Vars Matter in UrbanNest's Stack**

> - UrbanNest's app reads **database URL, API keys, and Azure credentials** from environment variables — never hardcoded
> - In Docker/Kubernetes, secrets are injected as env vars at container startup — Java reads them with `System.getenv()`
> - Spring Boot `application.yml` uses `${DB_URL}` placeholders — same value injected by Kubernetes Secret
> - Azure Key Vault secrets are exposed as env vars to AKS pods — zero code changes needed

#### 8.1 Reading Env Vars and Files

```
// Read env var — injected by K8s/Docker
String dbUrl = System.getenv("DB_URL");
System.out.println("Connecting to: " + dbUrl);

// Read a config file (Java 11+)
Path path = Path.of("/config/app.properties");
String config = Files.readString(path);
System.out.println(config);

// Write output to a file
Files.writeString(
    Path.of("/tmp/report.txt"),
    "Processed 200 PDFs successfully"
);
```

**📖 How It Works**

- **System.getenv("KEY")** — Reads an OS environment variable; returns `null` if not set
- **Files.readString(path)** — Reads entire file as a String in one line (Java 11+)
- **Path.of()** — Modern way to represent file paths; works on all OS
- **Files.writeString()** — Writes a String to file; creates file if it doesn't exist
- Always check `System.getenv() != null` before using — prevents NullPointerException on missing config

> **🔧 Where it's used in UrbanNest's Stack**

> - Kubernetes Secret → env var → Spring Boot reads `${DB_URL}` in `application.yml` → same `System.getenv()` under the hood
> - Azure Key Vault secret exposed to AKS pod as env var — Java reads it with `System.getenv("SQL_PASSWORD")`, zero code change
> - Jenkins pipeline injects credentials as env vars into build steps — Java utility scripts consume them securely without logging

### 9. JDBC — Database Connectivity

> **Why JDBC Matters in UrbanNest's Stack**

> - UrbanNest's backend connects to **Azure SQL Database** — JDBC is Java's native protocol for SQL database connections
> - Spring Data JPA abstracts JDBC — but the **connection pool (HikariCP) and connection string are pure JDBC config**
> - Knowing JDBC helps debug connection pool exhaustion, timeouts, and SQL errors in production AKS logs
> - SonarQube flags raw string concatenation in SQL queries — use `PreparedStatement` to pass the security gate

#### 9.1 Connect, Query, Close

```
String url  = System.getenv("DB_URL");
String user = System.getenv("DB_USER");
String pass = System.getenv("DB_PASS");

// try-with-resources auto-closes connection
try (Connection conn =
         DriverManager.getConnection(url, user, pass);
     PreparedStatement ps = conn.prepareStatement(
         "SELECT name FROM properties WHERE city = ?")) {

    ps.setString(1, "Hyderabad"); // safe — no SQL injection
ResultSet rs = ps.executeQuery();
    while (rs.next()) {
        System.out.println(rs.getString("name"));
    }
}
```

**📖 How It Works**

- **DriverManager.getConnection()** — Opens a physical connection to Azure SQL using URL, user, password
- **PreparedStatement** — Parameterised query; `?` is filled safely by `setString()` — prevents SQL injection
- **ResultSet** — A cursor; call `rs.next()` to move to next row; read columns by name
- **try-with-resources** — Auto-closes Connection and PreparedStatement even if exception occurs
- In Spring Boot, HikariCP manages a pool of these connections — you never call `DriverManager` directly

> **🔧 Where it's used in UrbanNest's Stack**

> - Spring Data JPA wraps JDBC; `application.yml` sets `spring.datasource.url` — same Azure SQL JDBC URL format
> - Azure SQL private endpoint: connection string uses `urbannest-sqlserver.database.windows.net` — resolves to private IP inside AKS VNet
> - SonarQube security rules flag `"SELECT * FROM t WHERE id = " + id` — PreparedStatement required to pass the quality gate

### 10. Spring Boot — REST API

> **Why Spring Boot Matters in UrbanNest's Stack**

> - UrbanNest's frontend React app calls backend **REST APIs** to fetch listings, submit bookings, and upload images
> - Spring Boot creates production-ready REST endpoints in minutes with embedded Tomcat — no server setup needed
> - The Spring Boot JAR is what gets **Dockerised, pushed to ACR, and deployed to AKS** via Azure DevOps pipeline
> - Azure Load Balancer distributes traffic across multiple Spring Boot pods running in AKS

#### 10.1 Controller + Service + Repository

```
// Controller — handles HTTP requests
@RestController
@RequestMapping("/api/properties")
public class PropertyController {

    @Autowired
private PropertyService service;

    @GetMapping("/{id}")
    public Property get(@PathVariable int id) {
        return service.findById(id);
    }

    @PostMapping
public Property create(
            @RequestBody Property p) {
        return service.save(p);
    }
}
```

**📖 How It Works**

- **@RestController** — Marks class as HTTP handler; all methods return JSON automatically
- **@RequestMapping** — Base URL path for all methods in this controller
- **@Autowired** — Spring injects the `PropertyService` bean automatically — no `new` keyword
- **@GetMapping("/{id}")** — Maps GET /api/properties/42 to this method
- **@PathVariable** — Extracts `42` from the URL and passes it as `id`
- **@RequestBody** — Deserialises JSON request body → Java `Property` object via Jackson

> **🔧 Where it's used in UrbanNest's Stack**

> - Spring Boot app is packaged as a JAR by Maven, Dockerised, pushed to ACR, deployed to AKS — this controller handles live traffic
> - Postman/curl tests hit these endpoints; JUnit tests mock the service layer for CI pipeline gate in Jenkins and Azure DevOps
> - Azure Load Balancer distributes HTTP traffic across all Spring Boot pods; Spring Boot's embedded Tomcat handles each request in its own thread

### 11. Maven — Build & Dependency Management

> **Why Maven Matters in UrbanNest's Stack**

> - UrbanNest's Java app depends on 30+ libraries — Spring Boot, Jackson, Hibernate, Azure SDK — Maven manages all of them
> - The **Jenkins pipeline and Azure DevOps YAML both run `mvn package`** as their core build step
> - Maven produces the JAR artifact that the Dockerfile copies into the container image
> - SonarQube Maven plugin sends code coverage reports to the quality gate during `mvn verify`

#### 11.1 pom.xml Dependency + Key Commands

```
<!-- pom.xml — add Spring Boot web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.2.0</version>
</dependency>

# Compile + test + package into JAR
mvn clean package
# Run only tests
mvn test
# Skip tests (faster build for debugging)
mvn clean package -DskipTests
# Show full dependency tree
mvn dependency:tree
```

**📖 How It Works**

- **pom.xml** — Project Object Model; declares all dependencies, plugins, and build config
- **mvn clean** — Deletes the `target/` folder (old build artifacts)
- **mvn package** — Compiles code → runs tests → produces `target/app.jar`
- **groupId + artifactId + version** — Unique coordinates to locate any library on Maven Central
- **dependency:tree** — Shows all transitive dependencies; use to debug version conflicts

> **🔧 Where it's used in UrbanNest's Stack**

> - Jenkins pipeline: `sh 'mvn clean package -DskipTests=false'` — produces JAR; next stage copies it into Docker image
> - Azure DevOps YAML uses Maven task: `- task: Maven@3 inputs: goals: 'clean package'` on every Git push to main
> - SonarQube: `mvn verify sonar:sonar` — runs tests, collects coverage, sends report to SonarQube server for quality gate evaluation

### 12. JUnit — Unit Testing

> **Why JUnit Matters in UrbanNest's Stack**

> - Before deploying to AKS, the CI pipeline must verify that **pricing logic, booking validation, and API endpoints** work correctly
> - JUnit 5 tests run automatically in Jenkins and Azure DevOps — if any test fails, the pipeline **stops the deployment**
> - SonarQube reads JUnit XML reports to calculate code coverage; minimum 80% required to pass the quality gate
> - JUnit tests run inside the Docker build layer — broken code never reaches the container registry

#### 12.1 @Test + Assertions + Mockito

```
// JUnit 5 test class
class PropertyServiceTest {

    @Mock
PropertyRepository repo; // mocked — no real DB
@InjectMocks
PropertyService service;

    @BeforeEach
void setup() {
        MockitoAnnotations.openMocks(this);
    }

    @Test
void discountShouldReducePriceByPercent() {
        double result = service.applyDiscount(10000000, 10);
        Assertions.assertEquals(9000000, result);
    }
}
```

**📖 How It Works**

- **@Test** — Marks a method as a test case; JUnit runs it and reports pass/fail
- **@Mock** — Mockito creates a fake repository with no real DB connection
- **@InjectMocks** — Mockito creates the service and injects mocked dependencies automatically
- **@BeforeEach** — Runs setup before every individual test method
- **Assertions.assertEquals(expected, actual)** — Fails test if values don't match; prints clear message

> **🔧 Where it's used in UrbanNest's Stack**

> - Jenkins pipeline runs `mvn test`; if any JUnit test fails, the stage is marked red and deployment is blocked
> - SonarQube reads `target/surefire-reports/*.xml` (JUnit output) to calculate line coverage and mark quality gate pass/fail
> - Azure DevOps publishes JUnit test results as a tab in the pipeline run UI — engineers can see exactly which tests failed

### 13. Logging with SLF4J & Logback

> **Why Logging Matters in UrbanNest's Stack**

> - When UrbanNest's API fails at 2 AM, the on-call engineer needs logs — **`System.out.println` doesn't go to Azure Monitor**
> - SLF4J structured logs from AKS pods stream automatically to **Azure Monitor and Application Insights**
> - Azure Monitor alert rules trigger on `log.error()` patterns — pages the on-call engineer via PagerDuty before the customer notices
> - Log levels controlled via Kubernetes ConfigMap env var — no redeployment needed to change verbosity

#### 13.1 Logger Setup + Log Levels

```python
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class BookingService {

    private static final Logger log =
        LoggerFactory.getLogger(BookingService.class);

    public void createBooking(int propertyId) {
        log.info("Creating booking: propertyId={}", propertyId);
        try {
            // booking logic
        } catch (Exception e) {
            log.error("Booking failed: id={}, error={}",
                propertyId, e.getMessage());
        }
    }
}
```

**📖 How It Works**

- **LoggerFactory.getLogger(ClassName.class)** — Creates a logger tagged with the class name for easy filtering
- **log.info()** — Normal operational events; always visible in production logs
- **log.error()** — Errors; Azure Monitor alert rules watch for these patterns
- **{} placeholders** — Lazily evaluated; `log.info("id={}", id)` is faster than string concatenation
- Log levels: TRACE < DEBUG < INFO < WARN < ERROR — set via env var `LOGGING_LEVEL_ROOT=INFO`

> **🔧 Where it's used in UrbanNest's Stack**

> - AKS pod stdout/stderr streams to Azure Monitor; Application Insights captures structured `log.info()` fields as searchable properties
> - Azure Monitor alert: when `log.error` count > 10 in 5 minutes → trigger PagerDuty alert → on-call engineer paged
> - Log level set via Kubernetes ConfigMap: `LOGGING_LEVEL_ROOT: "DEBUG"` for prod debugging — no Docker rebuild needed

### 14. Dockerising a Java App

> **Why Docker Matters in UrbanNest's Stack**

> - UrbanNest's Spring Boot app runs on the developer's laptop but crashes on the production VM — Docker fixes "works on my machine"
> - Docker packages the app + JRE + config into one image that runs identically on **any machine, any cloud, any AKS node**
> - Multi-stage Dockerfile keeps the production image small — no Maven, no JDK in the final image
> - Azure DevOps pipeline builds this Dockerfile and pushes the image to ACR on every merge to main

#### 14.1 Multi-Stage Dockerfile

```dockerfile
# Stage 1 — Build: Maven + JDK
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests
# Stage 2 — Run: JRE only (smaller)
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /app/target/urbannest.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**📖 How It Works**

- **Stage 1 (builder)** — Has full Maven + JDK to compile code and produce JAR
- **Stage 2 (run)** — Has only JRE (100MB vs 500MB); copies only the JAR from Stage 1
- **COPY --from=builder** — Pulls artifact from Stage 1 into Stage 2; Stage 1 is discarded from final image
- **EXPOSE 8080** — Documents which port the app listens on; Kubernetes Service maps to this
- **ENTRYPOINT** — Command that runs when container starts; runs the Spring Boot JAR

> **🔧 Where it's used in UrbanNest's Stack**

> - Azure DevOps pipeline: `docker build -t urbannest-acr.azurecr.io/api:$(Build.BuildId) .` then `docker push` to ACR
> - AKS pulls the image from ACR using Managed Identity — no Docker credentials stored anywhere
> - Jenkins: `sh 'docker build -t urbannest-api . && docker push acr.io/urbannest-api:${BUILD_NUMBER}'`

### 15. Java in the Full UrbanNest DevOps Stack

Stack Layer

Tool

Java Concept Used

Key Detail

Source Code

Git + IntelliJ

OOP, Collections, Streams

Spring Boot app + Maven project structure

Build

Jenkins / Azure DevOps

Maven lifecycle

mvn clean package produces JAR artifact

Test Gate

JUnit + Mockito

@Test, @Mock, Assertions

Fail build if any test fails; blocks deployment

Code Quality

SonarQube

Streams, PreparedStatement

Coverage > 80% required; SQL injection rules

Containerise

Docker

Multi-stage Dockerfile

Stage 1: Maven build; Stage 2: JRE-only image

Registry

Azure Container Registry

Docker image tag

Tagged with build ID; pushed by pipeline

Deploy

AKS (Kubernetes)

Env vars via System.getenv()

Deployment YAML injects secrets as env vars

Secrets

Azure Key Vault

System.getenv()

Key Vault secrets exposed as pod env vars

Database

Azure SQL

JDBC / Spring Data JPA

HikariCP pool; private endpoint inside VNet

Serverless

Azure Functions

Threads, File I/O

Java Function triggered by Blob Storage events

Observability

Azure Monitor + App Insights

SLF4J logging

Pod stdout streams to Azure Monitor; error alerts

Traffic

Azure Load Balancer

Spring Boot Tomcat threads

LB distributes to all healthy pods; 200 threads/pod

#### 🎉 You Now Know Java's Role in the UrbanNest DevOps Stack

OOP → Collections → Streams → Exceptions → Threads → JDBC → Spring Boot → Maven → JUnit → Logging → Docker → AKS. Every concept mapped to a real tool in the pipeline.
