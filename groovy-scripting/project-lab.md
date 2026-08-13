# Groovy Scripting Mastery

> **👋 Hey Fresher — Read This First!**

> Groovy is a scripting language that runs on the Java Virtual Machine (JVM). It looks like a simpler, shorter version of Java — most Java code is also valid Groovy, but Groovy lets you do the same thing in far fewer lines. DevOps engineers use Groovy primarily to write **Jenkins Pipeline scripts** — the Jenkinsfile that defines every step of a CI/CD pipeline. This document uses **short code blocks** — each block covers exactly one concept — with a plain-English explanation right beside it. One idea at a time. No walls of code that make freshers give up.

> **Company in this project:** BuildBridge — a Bangalore-based SaaS startup building a project management tool. Their Jenkins pipelines are a mess of shell scripts. You just joined as a Junior DevOps Engineer. Your lead is Meera. You will learn Groovy by writing real automation scripts for BuildBridge's CI/CD system — starting from variables and ending at a full Declarative Pipeline.

#### What You Will Learn and Build in This Project

You will go from zero Groovy knowledge to writing a complete Jenkins Declarative Pipeline — learning every language feature along the way, each explained with a real DevOps use case.

Variables, Strings, Collections, Closures, File I/O, JSON & XML, OOP, Error Handling, Jenkins DSL

> **🔤 Phase 1 — Language Basics**

> Variables, data types, strings, operators, conditionals, and loops — the building blocks every Groovy script starts with.

> Lists, Maps, and Groovy's powerful closures — the patterns that make Groovy so expressive and popular in CI/CD scripts.

> Read build outputs, parse JSON API responses, extract XML test results — real tasks from real pipelines.

> Classes, methods, error handling, and finally — writing a complete Declarative Jenkinsfile from scratch.

**Scene 1 — BuildBridge Office, Bangalore | "Why Do I Need to Learn Groovy?"**

> **Arjun (You)** _Junior DevOps Engineer — Day 1 at BuildBridge_
> 
> Meera, I thought DevOps was about shell scripts and YAML. Why do I need to learn another programming language?

> **Meera** _Senior DevOps Engineer — BuildBridge_
> 
> Because Jenkins Pipelines are written in Groovy. Our entire CI/CD system — build, test, security scan, deploy to staging, smoke test, deploy to production — all of it is Groovy code in a file called Jenkinsfile. If you can't read and write Groovy, you can't maintain or fix our pipelines. And right now, our pipelines break every other day and nobody knows why — because nobody wrote them properly.

> **Vikram** _Backend Engineer — BuildBridge_
> 
> Groovy is also much more readable than shell script for complex logic — conditionals, loops, parsing JSON, error handling. If your pipeline needs to call a REST API, parse the response, and decide whether to continue or abort, doing that in bash is painful. In Groovy, it is ten clean lines.

### 1. Phase 1 — Groovy Language Basics

Groovy runs on the JVM. You do not need to compile it — you run a `.groovy` file directly. Start by installing Groovy, then learn the building blocks.

> **The Big Picture — Groovy's Superpower**

> Groovy is **optionally typed** — you can write `def name = "Arjun"` and let Groovy figure out the type, or write `String name = "Arjun"` like Java. You can skip semicolons. You can skip `return` in methods. Parentheses are often optional. Groovy removes all the Java boilerplate so you can focus on what you actually want to do. In Jenkins, every Jenkinsfile is a Groovy script — knowing Groovy means you can write, debug, and extend any pipeline.

```
Groovy in the DevOps World — Where It Fits
==========================================

  Your Code (Git)
       │
       ▼
  ┌─────────────────────────────────────┐
  │         Jenkinsfile (Groovy)         │
  │  pipeline {                          │
  │    stages {                          │
  │      stage('Build')  { ... }         │   ← you write this in Groovy
  │      stage('Test')   { ... }         │
  │      stage('Deploy') { ... }         │
  │    }                                 │
  │  }                                   │
  └─────────────────────────────────────┘
       │ Jenkins reads this file
       ▼
  Build → Test → Scan → Deploy → Notify
       │                          │
       ▼                          ▼
  Pass/Fail email            Slack alert
  (Groovy logic)           (Groovy HTTP call)
```

#### 1.1 Installing Groovy and Running Your First Script

1. Install Groovy on Linux/Mac
The easiest way is via SDKMAN — a tool for managing JVM languages.

```bash
# Install SDKMAN (one-time setup)
curl -s "https://get.sdkman.io" | bash

# Install Groovy
sdk install groovy

# Check it works
groovy --version
```

> SDKMAN is like `nvm` for Node or `pyenv` for Python — it manages JVM tool versions. After installation, **groovy --version** should print something like `Groovy Version: 4.0.x JVM: 17.x`. You need Java (JDK) installed first since Groovy runs on the JVM. On Windows, download the Groovy installer from groovy-lang.org. In Jenkins, Groovy is already built in — you just write the Jenkinsfile.

2. Run your first Groovy script

```
// Save this as hello.groovy
println "Hello, BuildBridge!"
```

> **println** prints to the terminal with a newline. Notice: no semicolon, no `public static void main`, no class declaration. Groovy scripts just run from top to bottom like Python. Run it with `groovy hello.groovy` — you'll see `Hello, BuildBridge!` in the terminal. This simplicity is exactly why Jenkins uses Groovy for pipelines — scripts, not full Java programs.

```
$ groovy hello.groovy
Hello, BuildBridge!
```

#### 1.2 Variables and Data Types

Groovy has two ways to declare variables: with `def` (dynamic type) or with an explicit type like Java. In pipeline scripts, `def` is most common.

```python
// Dynamic typing with def
def appName    = "buildbridge-api"
def version    = 1
def isPassing  = true
def pi         = 3.14
println appName    // buildbridge-api
println version    // 1
println isPassing  // true
```

**📖 def — The Groovy Way**

**def** means "figure out the type at runtime." Groovy sees `"buildbridge-api"` and knows it's a String. It sees `1` and knows it's an Integer. You don't have to declare the type explicitly.  
  
**When to use def:** Almost always in Groovy scripts and Jenkinsfiles. It keeps code short and readable.  
  
**When to use explicit types:** When writing shared Groovy library code or when you want the compiler to catch type mistakes early.

```
// Explicit types (Java style — also valid)
String  buildEnv  = "staging"
Integer buildNum  = 42
Boolean deployed  = false
Double  coverage  = 87.5
println "Build #${buildNum} in ${buildEnv}"
// Build #42 in staging
```

**📖 Explicit Types — Java Style**

**String, Integer, Boolean, Double** — these are the same Java types, all valid in Groovy.  
  
**When you see `${variable}` inside a String** — that's Groovy String interpolation. Groovy replaces `${buildNum}` with the actual value. Use double quotes `" "` for strings with interpolation. Use single quotes `' '` for strings where you want no interpolation (the `$` is treated as a literal character).

> **💡 Fresher Tip — Single Quotes vs Double Quotes**

> This trips up every Groovy beginner. Use **double quotes** `"Hello ${name}"` when you want to embed variable values. Use **single quotes** `'Hello ${name}'` when you want the text literally (no substitution). In Jenkins pipelines, shell commands use single quotes to avoid Groovy interpreting `$` before bash gets it: `sh 'echo $HOME'` — here `$HOME` should be interpreted by bash, not Groovy.

#### 1.3 Strings — The Most Used Type in DevOps Scripts

```python
def branch  = "feature/login"
def tag     = "v2.3.1"
// Interpolation
println "Deploying ${tag} from ${branch}"
// String methods
println branch.toUpperCase()
println tag.startsWith("v")
println branch.contains("feature")
println tag.replace("v", "")
```

**📖 Useful String Methods**

**toUpperCase() / toLowerCase()** — change case. Common for normalising environment names.  
  
**startsWith("v")** — returns true/false. Use to check if a Git tag is a version tag before deploying.  
  
**contains("feature")** — returns true if the substring exists. Use in pipelines to check if the branch name matches a pattern.  
  
**replace("v", "")** — removes or swaps text. Use to strip the `v` prefix from a version tag before passing it to Docker.

```python
// Multi-line strings (triple quotes)
def slackMsg = """
Build #42 completed!
Branch : feature/login
Status : SUCCESS
"""
println slackMsg

// Split a string into a list
def services = "api,frontend,worker".split(",")
println services[0]  // api
```

> **Triple-quoted strings** (three double quotes) span multiple lines — perfect for composing Slack notification messages or writing multi-line shell commands. **split(",")** breaks a comma-separated string into an array — you'll use this constantly in pipelines where environment variables store comma-delimited lists of services or environments to deploy to.

#### 1.4 Conditionals — Making Decisions in Your Pipeline

```python
def branch = "main"
if (branch == "main") {
    println "Deploy to production"
} else if (branch == "staging") {
    println "Deploy to staging"
} else {
    println "Run tests only"
}
```

**📖 if / else if / else**

Identical to Java and JavaScript. In Jenkins pipelines, this pattern appears constantly — you check the branch name to decide where to deploy.  
  
**Common pipeline logic:**  
• `main` → deploy to production  
• `develop` → deploy to staging  
• any other branch → just run tests, don't deploy  
  
This single `if` block controls your entire deployment strategy.

```python
def testsPassed = true
def coverage    = 72
// && means AND, || means OR
if (testsPassed && coverage >= 80) {
    println "Gate passed — deploying"
} else {
    println "Gate failed — blocking deploy"
    error("Coverage below 80%")
}
```

**📖 Quality Gate Pattern**

This is one of the most common Groovy patterns in pipelines — a **quality gate** that blocks deployment if tests fail or coverage is too low.  
  
**&&** = AND. Both conditions must be true.  
**||** = OR. Either condition being true is enough.  
  
**error("message")** is a Jenkins Pipeline function that immediately fails the build and displays the message. Use it to enforce standards — never let a broken build reach production.

#### 1.5 Loops — Running the Same Logic for Multiple Targets

```python
// for loop — classic style
for (def i = 0; i < 3; i++) {
    println "Attempt ${i + 1}"
}

// for-in loop — cleaner for lists
def envs = ["dev", "staging", "prod"]
for (env in envs) {
    println "Deploying to ${env}"
}
```

**📖 Two Kinds of for Loop**

**Classic for loop** — use when you need an index (attempt number, retry count).  
  
**for-in loop** — use when you have a list of things to iterate over. Much more readable. In pipelines: loop over a list of microservices to deploy each one, or loop over environments to run health checks against all of them.  
  
In Jenkins Scripted Pipelines, the for-in style is preferred. In Declarative Pipelines, you often use the closure style shown in Phase 2.

```python
// while loop — for retries with a condition
def attempts = 0
def success  = false
while (!success && attempts < 3) {
    attempts++
    println "Health check attempt ${attempts}"
    success = true // replace with real check
}
println success ? "Healthy!" : "Failed after 3 tries"
```

> **while loop** — keeps running until the condition is false. The retry pattern above is extremely common in deployment pipelines: try calling a health check endpoint, if it returns 200 set `success = true`, otherwise increment attempts and try again. Stop after 3 attempts. The last line uses a **ternary operator** — `condition ? valueIfTrue : valueIfFalse` — a compact one-line if/else. In pipelines, you'll use this for build status messages.

**Quiz: ❓ Quick Check — Variables & Conditionals**

- A) prod
- B) PROD ✅
- C) Prod
- D) Error — def can't call methods

> **Answer/explanation:** B is correct. `toUpperCase()` converts every character to uppercase. Even though `env` was declared with `def`, Groovy knows at runtime it's a String and allows all String methods. This is the power of dynamic typing — you get the concise `def` declaration AND full access to all the type's methods.

### 2. Phase 2 — Collections and Closures

**Business Problem:** BuildBridge deploys 6 microservices to 3 environments. Writing separate deploy code for each combination is 18 blocks of near-identical code. Collections and closures let you express that in 5 clean lines.

**Scene 2 — BuildBridge Jenkins Server | "This Pipeline Is 400 Lines of Copy-Paste"**

> **Meera** _Senior DevOps Engineer — BuildBridge_
> 
> Arjun, look at our current Jenkinsfile. 400 lines. Each service has its own copy-pasted deploy block. When we added the worker service last month, someone forgot to update one of the copies and we deployed the wrong version to production. Lists and closures will reduce this to 40 lines and make it impossible to forget a service.

#### 2.1 Lists — Ordered Collections

```python
// Create a list
def services = ["api", "frontend", "worker"]

// Access by index (starts at 0)
println services[0]      // api
println services[-1]     // worker (last item)
// Add / remove items
services.add("scheduler")
services.remove("worker")
println services.size()   // 3
```

**📖 List Basics**

**Square brackets []** create a list. Items are separated by commas.  
  
**Index from 0** — the first item is `[0]`, the second is `[1]`.  
  
**Negative index [-1]** — Groovy lets you count from the end. `[-1]` = last item. This is much cleaner than `services[services.size()-1]`.  
  
**.add() / .remove()** — modify the list in place. **.size()** returns the count. Use lists in pipelines to define which services, environments, or regions to process.

```python
// Useful list operations for pipelines
def tags = ["v1.0", "v1.1", "v2.0"]

println tags.contains("v2.0")     // true
println tags.first()               // v1.0
println tags.last()                // v2.0
println tags.sort()                // [v1.0, v1.1, v2.0]
println tags.join(", ")           // v1.0, v1.1, v2.0
```

> **contains()** — checks if an item exists. Use to verify a Docker tag was built before trying to deploy it. **sort()** — sorts alphabetically. **join(", ")** — converts the list to a string with a separator between items — use this to build a comma-separated list of artifacts for a Slack notification message or an email report.

#### 2.2 Maps — Key-Value Collections

```python
// Create a map (like a dictionary)
def config = [
    appName   : "buildbridge-api",
    version   : "v2.3.1",
    port      : 8080,
    replicas  : 3
]

// Access values
println config.appName      // buildbridge-api
println config["version"]  // v2.3.1
// Add a new key
config.region = "ap-south-1"
```

**📖 Maps — The Config Object of Groovy**

**Maps** store key-value pairs. Two ways to access values:  
• Dot notation: `config.appName` — clean, readable  
• Bracket notation: `config["version"]` — needed when the key name has special characters or is stored in a variable  
  
In Jenkins pipelines, maps are used constantly for environment-specific configuration — define a map per environment (dev/staging/prod) with different URLs, replica counts, and AWS regions. Pass the right map to a shared deploy function.

```python
// Environment config map — common pipeline pattern
def envConfig = [
    dev     : [url: "dev.buildbridge.in",  replicas: 1],
    staging : [url: "stg.buildbridge.in",  replicas: 2],
    prod    : [url: "buildbridge.in",      replicas: 5]
]

def env = "staging"
println envConfig[env].url       // stg.buildbridge.in
println envConfig[env].replicas  // 2
```

> This **nested map pattern** is one of the most powerful patterns in pipeline scripting. You define all environment differences in one place, then look up the right config with `envConfig[env]`. Change the staging URL once in the map — every stage of the pipeline that uses it updates automatically. No more searching through 400 lines looking for hardcoded values.

#### 2.3 Closures — Groovy's Most Powerful Feature

> **💡 Fresher Tip — What is a Closure?**

> A closure is a **block of code you can store in a variable and run later**. Think of it as a mini-function without a name. In Jenkins Declarative Pipelines, every stage block (`steps { ... }`) is actually a closure being passed to the pipeline framework. Understanding closures explains why Jenkinsfiles look the way they do.

```python
// A closure stored in a variable
def greet = { name ->
    println "Hello, ${name}!"
}

// Call it like a function
greet("Arjun")    // Hello, Arjun!
greet("Meera")    // Hello, Meera!
```

**📖 Closure Syntax Explained**

**{ name -> ... }** — the curly braces define the closure. The arrow `->` separates the parameter list from the body.  
  
**Storing in a variable:** `def greet = { ... }` — now `greet` holds the closure. Call it with parentheses just like a function.  
  
If a closure has no parameters, omit the arrow: `def sayHi = { println "Hi" }`. If it has exactly one parameter, Groovy provides a default name: `it`.

```python
// each{} — loop using a closure
def services = ["api", "frontend", "worker"]

services.each { svc ->
    println "Deploying: ${svc}"
}

// Using the implicit 'it' variable
services.each {
    println "Service: ${it}"
}
```

**📖 each{} — The Groovy Way to Loop**

**.each{ item -> }** is Groovy's preferred loop for lists. It's a method that takes a closure as its argument and calls it once for each item.  
  
**it** — when a closure has exactly one parameter and you don't name it, Groovy automatically names it `it`. So `services.each { println it }` is the same as `services.each { svc -> println svc }`.  
  
In Jenkins, you'll write: `services.each { deploy(it) }` to trigger a deploy function for every service.

```python
// collect{} — transform every item in a list
def services = ["api", "frontend", "worker"]

def dockerImages = services.collect { svc ->
    "buildbridge/${svc}:v2.3.1"
}

println dockerImages
// [buildbridge/api:v2.3.1, buildbridge/frontend:v2.3.1, ...]
// findAll{} — filter a list
def sizes = [45, 120, 30, 95]
def large = sizes.findAll { it > 80 }
println large  // [120, 95]
```

> **collect{}** transforms every item — the closure receives each item and returns a new value. The result is a new list of the transformed values. Use it to build Docker image names, construct file paths, or format values before printing. **findAll{}** filters a list — the closure returns true/false for each item; only items where the closure returns true are kept. Use it to filter failed test results, find services above a resource threshold, or select only production environments.

**Quiz: ❓ Quick Check — Collections & Closures**

- A) "ABC"
- B) ["A", "B", "C"] ✅
- C) ["a", "b", "c"]
- D) null

> **Answer/explanation:** B is correct. `collect{}` transforms each item and returns a new list. `it.toUpperCase()` converts each string to uppercase. The result is a new list — the original is unchanged. This pattern is used constantly in pipelines: take a list of service names, transform them into Docker image tags, deploy them all.

### 3. Phase 3 — Methods, File I/O, and Parsing

**Business Problem:** BuildBridge's pipeline needs to read a VERSION file to get the current version, parse the test result XML to check if tests passed, and call a REST API to notify the monitoring system. All of this requires file reading, JSON/XML parsing, and HTTP calls.

**Scene 3 — BuildBridge Post-Incident Review | "How Did a Broken Build Reach Production?"**

> **Meera** _Senior DevOps Engineer — BuildBridge_
> 
> Our pipeline was not parsing the test result file. It just checked if the test command returned exit code 0 — but JUnit writes failures to an XML file. The command exited 0 even though 8 tests failed. Groovy can parse that XML in three lines and abort the build if any test failed. We just weren't doing it.

#### 3.1 Defining and Calling Methods

```python
// Method definition
def getDockerTag(appName, version) {
    return "buildbridge/${appName}:${version}"
}

// Call the method
def tag = getDockerTag("api", "v2.3.1")
println tag  // buildbridge/api:v2.3.1
```

**📖 Methods in Groovy**

**def methodName(params) { }** — the basic method syntax. The `return` keyword is optional — Groovy returns the last evaluated expression automatically.  
  
So `def add(a, b) { a + b }` works without `return`. This is called an **implicit return**.  
  
In Jenkins Shared Libraries (reusable code across pipelines), you write Groovy methods like this that any Jenkinsfile can call. `getDockerTag()` is a great shared library candidate — every pipeline needs it.

```python
// Method with a default parameter value
def deployService(serviceName, env = "staging") {
    println "Deploying ${serviceName} to ${env}"
}

deployService("api")             // uses default: staging
deployService("api", "prod")    // overrides to: prod
```

> **Default parameter values** — if the caller doesn't provide a value for `env`, it defaults to `"staging"`. This is very useful in Jenkins pipelines where most deployments target staging, but you occasionally override to prod. One method handles both cases — no duplication. Groovy supports default parameters natively; Java does not (Java requires overloading).

#### 3.2 Reading and Writing Files

```python
// Read entire file as string
def version = new File("VERSION").text.trim()
println "Building version: ${version}"
// Read file line by line
new File("services.txt").eachLine { line ->
    println "Found service: ${line}"
}
```

**📖 Reading Files in Groovy**

**new File("path").text** — reads the entire file content as a String. **.trim()** removes leading/trailing whitespace and newlines (always do this — files often have a trailing newline that breaks comparisons).  
  
**.eachLine{ line -> }** — reads the file line by line and calls the closure for each one. Use this when the file has many entries and you don't want to load all of them into memory at once.  
  
In Jenkins pipelines, use `readFile('VERSION')` (the pipeline step) instead of `new File()` — it works on agent nodes correctly.

```python
// Write to a file
def report = "Build: v2.3.1\nStatus: SUCCESS\nTests: 142 passed"
new File("build-report.txt").text = report

// Append to a file
new File("deploy.log") << "\nDeployed api at 14:32"
```

> **.text = value** — writes (overwrites) the file with the value. **<< "text"** — appends text to the file without erasing existing content. The `<<` operator is called the "left shift" operator — in Groovy it's been repurposed for stream/file appending. Use the append operator when you're writing a rolling deploy log across multiple pipeline stages.

#### 3.3 Parsing JSON — Reading API Responses

```python
import groovy.json.JsonSlurper

def jsonText = '''
{
  "service": "api",
  "version": "v2.3.1",
  "healthy": true,
  "pods": 3
}
'''
def data = new JsonSlurper().parseText(jsonText)

println data.service   // api
println data.healthy   // true
println data.pods      // 3
```

**📖 JsonSlurper — Parse JSON in 2 Lines**

**import groovy.json.JsonSlurper** — bring in the built-in JSON parser (no external library needed).  
  
**new JsonSlurper().parseText()** — converts a JSON string into a Groovy Map. Access fields with dot notation exactly like a Groovy map.  
  
In pipelines, call a health check API endpoint, get the JSON response, parse it, and check `data.healthy == true` before proceeding. Three lines, zero dependencies.

```python
import groovy.json.JsonOutput

// Convert a Groovy map to JSON string
def payload = [
    build  : "#42",
    status : "SUCCESS",
    branch : "main"
]

def jsonStr = JsonOutput.toJson(payload)
println JsonOutput.prettyPrint(jsonStr)
```

> **JsonOutput.toJson()** converts a Groovy map into a JSON string — use this to build the body of a POST request to a Slack webhook, PagerDuty, or any REST API. **JsonOutput.prettyPrint()** formats it with indentation for human-readable logs. In Jenkins pipelines, you'll use this pattern: build a map of build metadata, convert to JSON, POST to your notification system.

#### 3.4 Parsing XML — Reading Test Results

```python
// Parse JUnit XML test results
def xml = new XmlSlurper().parseText("""
<testsuite tests="10" failures="2">
  <testcase name="login" />
  <testcase name="payment">
    <failure message="Timeout" />
  </testcase>
</testsuite>
""")

println xml.@tests     // 10
println xml.@failures  // 2
```

**📖 XmlSlurper — Parse XML Test Reports**

**new XmlSlurper().parseText()** — parses XML into a navigable Groovy object.  
  
**xml.@tests** — the `@` symbol accesses XML attributes. `xml.@tests` reads the `tests="10"` attribute.  
  
JUnit, TestNG, and most test frameworks produce XML result files. Your pipeline should parse them and call `error("2 tests failed!")` if failures > 0. This catches failures that the test runner exit code might miss.

#### 3.5 Making HTTP Calls

```python
import groovy.json.JsonOutput

// POST to a Slack webhook (notify team)
def notifySlack(message) {
    def payload = JsonOutput.toJson([text: message])
    def url     = "https://hooks.slack.com/services/YOUR/WEBHOOK"

    ["curl", "-X", "POST",
     "-H", "Content-Type: application/json",
     "-d", payload, url
    ].execute().waitFor()
}

notifySlack("✅ BuildBridge v2.3.1 deployed to prod!")
```

> **.execute()** — Groovy allows you to execute any command as a list of strings. This runs `curl` with the arguments provided. **.waitFor()** blocks until the command finishes. This pattern works in standalone Groovy scripts. In Jenkins Pipelines, use the built-in `httpRequest` step or `sh 'curl ...'` step instead — they handle agent connectivity and credentials more reliably than `.execute()`.

### 4. Phase 4 — Object-Oriented Groovy

Groovy is fully object-oriented. For large Jenkins Shared Libraries, you organise your code into classes. Even if you only ever write Jenkinsfiles, understanding classes helps you read library code and understand how Jenkins objects (like `currentBuild`) work.

#### 4.1 Defining a Class

```python
class Service {
    String name
    String version
    Integer replicas = 1 // default
def getDockerTag() {
        return "buildbridge/${name}:${version}"
    }

    def describe() {
        println "${name} v${version} — ${replicas} replica(s)"
    }
}
```

**📖 Class Anatomy**

**class ClassName { }** — defines a class. By convention, class names start with an uppercase letter.  
  
**Fields** (name, version, replicas) — the data the class holds. Give a field a default value with `= value`.  
  
**Methods** (getDockerTag, describe) — the actions the class can do.  
  
In Jenkins Shared Libraries, you create a class like `Service` to represent a microservice with all its config. Then every Jenkinsfile creates `Service` objects and calls methods on them — no more copy-pasted config.

```python
// Create objects from the class
def api = new Service(name: "api", version: "v2.3.1", replicas: 3)
def frontend = new Service(name: "frontend", version: "v2.3.1")

api.describe()           // api v2.3.1 — 3 replica(s)
frontend.describe()      // frontend v2.3.1 — 1 replica(s)
println api.getDockerTag() // buildbridge/api:v2.3.1
```

> **new Service(name: "api", ...)** — creates an object using Groovy's named-parameter constructor. You don't need to write a constructor method — Groovy generates one automatically for classes that have fields. Pass only the fields you want to set; the rest use their default values. `frontend` didn't specify `replicas`, so it defaults to `1` as defined in the class.

### 5. Phase 5 — Error Handling

**Business Problem:** BuildBridge's deploy script crashes without explanation when a service is unavailable. Error handling lets you catch failures, log useful messages, and decide whether to retry, skip, or abort.

#### 5.1 try / catch / finally

```python
try {
    println "Deploying api service..."
def result = deployApi()  // might throw
println "Deploy succeeded"

} catch (Exception e) {
    println "Deploy failed: ${e.message}"

} finally {
    println "Cleaning up temp files..."
}
```

**📖 try / catch / finally**

**try { }** — the code that might fail goes here.  
  
**catch (Exception e) { }** — if anything in the try block throws an exception, execution jumps here. `e.message` gives you the error message to log.  
  
**finally { }** — runs whether or not an exception occurred. Use for cleanup: remove temp files, send a "pipeline finished" notification, or reset a flag. In Jenkins pipelines, `post { always { } }` is the Declarative equivalent of `finally`.

```python
// Retry pattern — try up to 3 times before giving up
def withRetry(attempts, closure) {
    for (def i = 1; i <= attempts; i++) {
        try {
            closure()
            return // success — exit method
        } catch (Exception e) {
            println "Attempt ${i} failed: ${e.message}"
if (i == attempts) throw e
        }
    }
}

withRetry(3) {
    println "Calling health check endpoint..."
}
```

> This **retry utility method** takes a number of attempts and a closure (the code to retry). It tries the closure, catches any exception, logs it, and tries again. If all attempts fail, it rethrows the final exception. Notice: the method accepts a closure as its second parameter — this is a core Groovy pattern. In Jenkins, the built-in `retry(3) { }` step does the same thing, but understanding how to build it yourself makes you a much better pipeline author.

### 6. Phase 6 — Jenkins Pipeline DSL

Everything you've learned — variables, conditionals, loops, closures, maps, error handling — comes together in the Jenkinsfile. Declarative Pipeline is the recommended syntax for Jenkins and it's written entirely in Groovy.

**Scene 6 — BuildBridge CI/CD War Room | "Finally, a Real Pipeline"**

> **Meera** _Senior DevOps Engineer — BuildBridge_
> 
> Every concept you've learned for the past week — closures, maps, error handling — exists in the Jenkinsfile. The `pipeline { }` block is a closure. Each `stage('name') { }` is a closure. The `when { }` block is a conditional. `post { always { } }` is the finally block. It all maps directly. Let's write BuildBridge's full pipeline.

#### 6.1 Declarative Pipeline — The Basic Structure

```
pipeline {
    agent any

    environment {
        APP_NAME = 'buildbridge-api'
        VERSION  = 'v2.3.1'
    }

    stages {
        stage('Hello') {
            steps {
                echo "Building ${APP_NAME}"
            }
        }
    }
}
```

**📖 Declarative Pipeline Skeleton**

**pipeline { }** — the root closure. Everything goes inside this.  
  
**agent any** — run on any available Jenkins worker node. You can also specify a Docker image or a specific labeled node.  
  
**environment { }** — define environment variables available to all stages. These become actual shell env vars — you can reference them in `sh` steps as `$APP_NAME`.  
  
**stages { }** — the list of stages to run in order.  
  
**stage('name') { steps { } }** — one step in the pipeline. The name appears in the Jenkins UI.

#### 6.2 Build Stage — Compile and Package

```
 stage('Build') {
            steps {
                sh '''
                    echo "Building..."
                    mvn clean package -q
                '''
archiveArtifacts(
                    artifacts: 'target/*.jar',
                    fingerprint: true
                )
            }
        }
```

**📖 Build Stage Explained**

**sh '...'** — runs a shell command on the Jenkins agent. Triple single quotes allow multi-line shell scripts. Single quotes prevent Groovy from interpolating `$` signs before the shell sees them.  
  
**archiveArtifacts** — saves the built JAR file in Jenkins so you can download it later or pass it to other stages. `fingerprint: true` tracks which builds produced which artifacts.  
  
Replace `mvn` with `npm run build`, `gradle build`, or any build command.

#### 6.3 Test Stage — Run Tests and Publish Results

```
 stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/**/*.xml'
                }
                failure {
                    echo "Tests failed! Check test report."
                }
            }
        }
```

**📖 Test Stage with post { }**

**post { }** inside a stage — runs after the stage, regardless of pass/fail.  
  
**always { }** — always runs. Use for publishing test reports, archiving logs.  
  
**failure { }** — only runs if the stage failed. Use to send a Slack alert or log extra debug info.  
  
**junit 'path'** — parses JUnit XML reports and shows test pass/fail data in the Jenkins UI with a graph over time. Without this step, you can't see which tests failed in the Jenkins interface.

#### 6.4 Docker Build Stage — Build and Push Image

```python
 stage('Docker Build') {
            steps {
                script {
                    def tag = "${APP_NAME}:${VERSION}"
sh "docker build -t ${tag} ."
sh "docker push ${tag}"
                }
            }
        }
```

**📖 script { } — Groovy Inside Declarative**

**script { }** — inside Declarative Pipeline, you can only use predefined Pipeline steps. But inside a `script { }` block, you can write any Groovy code: variables, loops, conditionals, method calls.  
  
This is the bridge between Declarative's structure and Scripted Pipeline's flexibility.  
  
Use `script { }` when you need a loop (deploy to multiple environments), a conditional (different commands for different branches), or Groovy logic that doesn't fit in a single step.

#### 6.5 Deploy Stage — Only Deploy from main Branch

```
 stage('Deploy to Prod') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?',
                      ok: 'Yes, deploy!'
sh './scripts/deploy-prod.sh'
            }
        }
```

**📖 when { } and input — Gate the Deploy**

**when { branch 'main' }** — this stage only runs when the pipeline is triggered from the `main` branch. For feature branches, this stage is automatically skipped.  
  
**input** — pauses the pipeline and shows a button in the Jenkins UI. A human must click "Yes, deploy!" before the deploy step runs. This is a **manual approval gate** — every production deployment requires explicit human sign-off. Nobody deploys to production by accident.

#### 6.6 The Complete Jenkinsfile — BuildBridge Production Pipeline

Putting every stage together with global post actions for notifications:

```python
pipeline {
    agent any

    environment {
        APP_NAME   = 'buildbridge-api'
        VERSION    = "v${BUILD_NUMBER}" // Jenkins built-in var
        DOCKER_REG = 'registry.buildbridge.in'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm   // pull code from Git
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
archiveArtifacts 'target/*.jar'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always { junit 'target/surefire-reports/**/*.xml' }
            }
        }

        stage('Docker') {
            steps {
                script {
                    def img = "${DOCKER_REG}/${APP_NAME}:${VERSION}"
sh "docker build -t ${img} ."
sh "docker push ${img}"
                }
            }
        }

        stage('Deploy Staging') {
            steps {
                sh "./deploy.sh staging ${VERSION}"
            }
        }

        stage('Deploy Prod') {
            when { branch 'main' }
            steps {
                input 'Approve production deploy?'
sh "./deploy.sh prod ${VERSION}"
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline SUCCESS — ${APP_NAME} ${VERSION}"
        }
        failure {
            echo "❌ Pipeline FAILED — check logs"
        }
        always {
            cleanWs()   // clean workspace after every run
        }
    }
}
```

> This is a **complete production Jenkinsfile** for BuildBridge. Walk through it stage by stage: **Checkout** pulls code from Git. **Build** compiles the app and archives the JAR. **Test** runs unit tests and publishes XML results to Jenkins UI. **Docker** builds and pushes the image to the registry — note the `script { }` block for the Groovy variable. **Deploy Staging** always runs. **Deploy Prod** only runs on the `main` branch and requires manual approval. **post { }** at pipeline level runs after all stages — success/failure notifications and workspace cleanup. **cleanWs()** deletes the workspace files so the Jenkins agent doesn't fill its disk over hundreds of builds.

### 7. Phase 7 — Jenkins Shared Libraries

BuildBridge has 6 microservices. Each has a nearly identical Jenkinsfile. Shared Libraries let you write the common pipeline logic once in a Groovy file and import it into every Jenkinsfile.

#### 7.1 Writing a Shared Library Step

```bash
// File: vars/deployService.groovy
def call(String serviceName, String env) {
    echo "Deploying ${serviceName} to ${env}"
sh """
        kubectl set image deployment/${serviceName} \
          ${serviceName}=registry.buildbridge.in/${serviceName}:latest \
          -n ${env}
    """
}
```

**📖 Shared Library Step**

Create a file in the `vars/` folder of a Git repo — Jenkins calls this the "Shared Library" repo.  
  
**The file name becomes the step name.** A file called `deployService.groovy` with a `call()` method creates a new Jenkins step called `deployService()`.  
  
Every Jenkinsfile in every project can then call `deployService("api", "prod")` — they all share the same deploy logic. Fix a bug in the shared library once and it's fixed everywhere.

```python
// In any project's Jenkinsfile — import the library
@Library('buildbridge-shared-lib') _

pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                deployService('api', 'prod')
            }
        }
    }
}
```

> **@Library('name') _** — imports the shared library at the top of the Jenkinsfile. The underscore `_` at the end is required syntax — it tells Groovy not to import anything specific (you're importing the whole library). After this line, all `vars/*.groovy` files from the library become available as steps. The Jenkinsfile now has a clean one-liner deploy call instead of 15 lines of kubectl commands. Every team benefits from one fix.

### 8. Phase 8 — Groovy Patterns in the Real World

These are the patterns you will see (and write) most often once you start working with real Jenkins pipelines at companies like BuildBridge.

#### 8.1 Reading Build Parameters

```
pipeline {
    agent any
    parameters {
        string(name: 'TARGET_ENV',
               defaultValue: 'staging',
               description: 'Deploy target')
        booleanParam(name: 'RUN_TESTS',
                      defaultValue: true)
    }
    stages {
        stage('Deploy') {
            steps {
                echo "Env: ${params.TARGET_ENV}"
            }
        }
    }
}
```

**📖 Pipeline Parameters**

**parameters { }** — defines inputs a user can change when triggering the pipeline manually in Jenkins UI.  
  
**string()** — a text input. **booleanParam()** — a checkbox.  
  
Access parameter values with `params.PARAM_NAME` anywhere in the pipeline.  
  
Common parameters: `TARGET_ENV` (which environment to deploy to), `IMAGE_TAG` (which version to deploy), `RUN_TESTS` (skip tests for hotfixes). Parameters make your pipeline reusable without editing the code.

#### 8.2 Using Credentials Safely

```
 stage('Push to Registry') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'docker-registry-creds',
                usernameVariable: 'REG_USER',
                passwordVariable: 'REG_PASS'
            )
        ]) {
            sh 'docker login -u $REG_USER -p $REG_PASS'
        }
    }
}
```

**📖 withCredentials — Never Hardcode Secrets**

**withCredentials([...])** — fetches secrets stored in Jenkins Credentials (not in your code). Inside this block, the secret is available as an environment variable.  
  
The secret is **masked in logs** — Jenkins replaces the actual value with `****` so it never appears in build output.  
  
Store Docker registry passwords, AWS keys, Slack tokens, and SSH keys in Jenkins Credentials. Never write a password directly in a Jenkinsfile. If the file is in Git, the password is in Git history forever.

#### 8.3 Parallel Stages — Speed Up Your Pipeline

```
 stage('Test & Lint') {
    parallel {
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Code Lint') {
            steps {
                sh 'mvn checkstyle:check'
            }
        }
    }
}
```

**📖 parallel { } — Run Stages Simultaneously**

**parallel { }** — runs multiple stages at the same time on different agents. If tests take 5 minutes and linting takes 3 minutes, running them sequentially takes 8 minutes. Running them in parallel takes 5 minutes (the longer one).  
  
Common parallel uses: unit tests + integration tests + linting at the same time; building for multiple platforms simultaneously; deploying to multiple regions at once.  
  
The pipeline waits for all parallel branches to complete before moving to the next stage.

**Quiz: ❓ Quick Check — Jenkins Pipeline**

- A) `if (branch == 'main') { }`
- B) `when { branch 'main' }` ✅
- C) `condition { main }`
- D) `filter { branch: 'main' }`

> **Answer/explanation:** B is correct. In Declarative Pipeline, `when { branch 'main' }` is the proper DSL syntax to conditionally run a stage. While option A (a Groovy if statement) would work inside a `script { }` block, the `when { }` directive is the Declarative way — it's evaluated before the stage runs, appears in the Jenkins UI, and integrates properly with stage skipping and reporting.

### 9. Phase 9 — Debugging and Best Practices

#### 9.1 Debugging Groovy Scripts

```python
// Print variable types and values during debugging
def data = [name: "api", version: 42]

println data.getClass()           // class java.util.LinkedHashMap
println data.name.getClass()     // class java.lang.String
println data.inspect()           // ['name':'api', 'version':42]
```

> **.getClass()** — tells you the actual Java type of any Groovy variable at runtime. When your script behaves unexpectedly, print the class to confirm the variable is what you think it is — a `def` might hold a String where you expected an Integer. **.inspect()** — prints a Groovy-readable representation of the object. Unlike `println data` (which just calls `toString()`), `inspect()` shows the structure clearly, including quotes around strings.

```python
// In Jenkins — the Groovy Console (for live debugging)
// Go to: Jenkins → Manage Jenkins → Script Console
// Type any Groovy code and run it immediately
def jobs = Jenkins.instance.items.collect { it.name }
println jobs
```

> Jenkins has a built-in **Groovy Script Console** at `/script`. You can run any Groovy code directly against the live Jenkins instance — list all jobs, check plugin versions, fix a stuck build, inspect a credential. It's the most powerful debugging tool Jenkins offers. Access it via Manage Jenkins → Script Console. Only admins should have access — it can do anything Groovy can do on the JVM.

> **✅ Key Takeaways — What Every Groovy DevOps Engineer Knows**

> - Use `def` for variables in scripts. Use explicit types in shared library classes where you want the compiler to catch mistakes.
> - Double quotes for strings with `${variable}` interpolation. Single quotes for strings where `$` should be treated literally — especially in Jenkins `sh` steps where `$` belongs to bash.
> - Closures are everywhere in Jenkins pipelines — every `stage { }`, `steps { }`, `post { }`, and `when { }` block is a closure being passed to the pipeline framework.
> - Use `script { }` inside Declarative Pipeline when you need Groovy logic — loops, conditionals, variable assignments — that doesn't fit in a single Pipeline step.
> - Never hardcode secrets in a Jenkinsfile. Use `withCredentials()` to retrieve them from Jenkins Credentials at runtime. The credential is masked in logs automatically.
> - Shared Libraries eliminate copy-paste pipelines. Write common logic once in `vars/`, import with `@Library`, and every project benefits from every improvement.

##### Groovy Standards — BuildBridge Pipeline Engineering Rules

- Keep your Jenkinsfile under 60 lines by moving all complex logic into Shared Library methods. The Jenkinsfile should read like a high-level summary of what the pipeline does, not a wall of implementation details.
- Always use `cleanWs()` in the `post { always { } }` block. Without it, Jenkins agents accumulate hundreds of gigabytes of workspace files over time and eventually run out of disk.
- Use `timeout(time: 30, unit: 'MINUTES') { }` around long-running stages. Without a timeout, a stuck process will hold a Jenkins agent forever — blocking every other build that needs that agent.
- Validate your Jenkinsfile syntax before pushing it. Use the Jenkins Pipeline Linter (available at `JENKINS_URL/pipeline-model-converter/validate`) or the Jenkins CLI `declarative-linter` command.
- Version your Shared Library — use Git tags. Reference a specific version in Jenkinsfiles: `@Library('buildbridge-lib@v1.2')`. Referencing `main` means a library update can break every pipeline at once.
- Use `parallel { }` for independent stages. Build time is a developer experience metric — every minute saved in the pipeline is a minute developers aren't waiting for feedback. Aim for pipelines under 10 minutes.

##### 🏋️ Hands-On Exercises — Build It Yourself

1. **Environment Config Script:** Write a standalone Groovy script that defines a nested map of environments (dev, staging, prod) with different URLs, replica counts, and database names. Write a method `printEnvConfig(envName)` that prints all details for the given environment. Call it for all three environments using `.each{}`.
2. **Test Result Parser:** Write a Groovy script that reads a JUnit XML file (create a sample one with 2 failures), parses it with `XmlSlurper`, and prints how many tests passed and how many failed. If there are any failures, print each failed test's name.
3. **Slack Notifier Method:** Write a `notifySlack(String status, String buildUrl)` method that builds a JSON payload (using `JsonOutput`) and sends it to a Slack webhook using `curl`. The message should include the build status (SUCCESS/FAILURE), build number, and a link to the Jenkins build URL.
4. **Shared Library Step:** Create a Groovy Shared Library step file at `vars/dockerBuildPush.groovy` with a `call(String imageName, String tag)` method. It should run `docker build`, `docker tag`, and `docker push`. Write a sample Jenkinsfile that imports it and uses it in a Docker stage.
5. **Full Pipeline:** Write a complete Declarative Jenkinsfile for a Node.js app. Stages: Checkout → Install deps (`npm install`) → Test (`npm test`, publish JUnit results) → Docker Build/Push → Deploy to staging (always) → Deploy to prod (only on `main` branch, with manual approval). Add a global `post` block with success/failure echo and `cleanWs()`.

##### ❓ Fresher Q&A — Questions Every Beginner Asks

**Q: Q: What is the difference between Scripted Pipeline and Declarative Pipeline in Jenkins?**

A: Declarative Pipeline uses the `pipeline { }` block with a fixed structure — agent, environment, stages, post. It's stricter but gives you a nicer UI, better error messages, and stage-level restarts. Scripted Pipeline is pure Groovy in a `node { }` block — more flexible, but harder to read and maintain. Almost all new pipelines use Declarative. You use `script { }` inside Declarative when you need Scripted-style Groovy flexibility in one section.

**Q: Q: Can I use Python or Bash instead of Groovy for Jenkins Pipelines?**

A: The Jenkinsfile itself must be Groovy — that's how Jenkins works. But inside a stage, you call `sh 'python myscript.py'` or `sh 'bash myscript.sh'` and the heavy lifting is done by your script. In practice, Groovy handles the pipeline orchestration (decisions, loops, API calls, credential management) and shell scripts handle the actual build/deploy commands. You need both.

**Q: Q: Is Groovy used anywhere besides Jenkins?**

A: Yes — Groovy is used in Gradle build scripts (Android builds use Groovy/Kotlin DSL), Apache Camel integration pipelines, SoapUI test automation, and Spock testing framework. But in DevOps specifically, Jenkins is by far the biggest consumer. Learning Groovy for Jenkins is immediately practical — you'll use it on day one in any company that runs Jenkins (which is most of them).

**Q: Q: How do I stop a Jenkins build from inside Groovy code?**

A: Use `error("your message here")` — it throws a `FlowInterruptedException` that marks the build as FAILED and stops execution. Use `currentBuild.result = 'UNSTABLE'` to mark as unstable (yellow) without failing. Use `currentBuild.result = 'ABORTED'; error('Aborted')` to mark as aborted. The most common pattern is `error("Tests failed — blocking deploy")` after checking test results.

Groovy Pattern

What It Does

When to Use

def x = value

Declare variable (dynamic type)

Always — for scripts and Jenkinsfiles

"Hello ${name}"

String with variable interpolation

Building messages, paths, Docker tags

list.each { }

Loop over every item

Deploy to multiple services/environments

list.collect { }

Transform every item, return new list

Build Docker image tag list from service names

list.findAll { }

Filter items matching condition

Find failed tests, filter environments

map.key or map["key"]

Access a map value

Env config lookup by environment name

try { } catch { }

Handle errors gracefully

Any step that might fail (deploy, HTTP call)

new JsonSlurper().parseText()

Parse JSON string to map

Read API responses, health check results

JsonOutput.toJson(map)

Convert map to JSON string

Build Slack/webhook notification payloads

new XmlSlurper().parseText()

Parse XML string

Read JUnit test result files

new File("path").text

Read entire file as string

Read VERSION, config files in scripts

script { }

Groovy code inside Declarative Pipeline

Loops, conditionals, variables in stages

when { branch 'main' }

Run stage only on specific branch

Production deploy gates

parallel { stage(){} }

Run stages simultaneously

Speed up test + lint + scan stages

withCredentials([...])

Inject secrets from Jenkins Credentials

Docker login, AWS keys, Slack tokens

input 'message'

Pause for human approval

Production deploy sign-off

@Library('name') _

Import Jenkins Shared Library

Reuse common pipeline logic across projects

### Groovy Scripting Complete 🎉

You have gone from zero Groovy knowledge to writing a complete Jenkins Declarative Pipeline — variables, strings, collections, closures, file I/O, JSON/XML parsing, OOP, error handling, Shared Libraries, and a full Jenkinsfile for BuildBridge's CI/CD system. You can now read, write, and debug any Groovy pipeline script you encounter in a real DevOps role.

> **Meera**
> 
> "Arjun, last month our pipelines were 400 lines of copy-pasted bash. This week you wrote a 55-line Declarative Jenkinsfile backed by a Shared Library. Every team's pipeline now calls the same deploy logic. We fixed a missing health check in one file and all 6 pipelines got safer overnight. That is what Groovy in Jenkins is for — and you built it."

> **Vikram**
> 
> "The mental model shift you made — from 'shell script with Jenkins glue' to 'Groovy programs that orchestrate the entire delivery process' — that is the DevOps engineer mindset. Once that clicks, you can read any Jenkinsfile in the world and understand what it does in minutes. And you can write better ones."

> **Next: Advanced Groovy — Metaprogramming, DSL Creation, and Jenkins Pipeline Testing**

> - Metaprogramming — `methodMissing`, `propertyMissing`, and ExpandoMetaClass to add methods to existing classes at runtime
> - Building your own DSL — how Jenkins, Gradle, and Spock use Groovy's syntax to create readable domain-specific languages
> - Jenkins Pipeline Unit — a testing framework that lets you unit test your Jenkinsfile logic without running a real Jenkins server
> - Groovy traits — like Java interfaces but with default method implementations; used in Shared Libraries for composable pipeline behaviours
> - Category classes and extensions — add methods to any existing class (even Java's String) across your codebase without modifying the class itself
> - GStringTemplateEngine — generate dynamic shell scripts, Kubernetes YAML, and Helm values files from Groovy templates at pipeline runtime
