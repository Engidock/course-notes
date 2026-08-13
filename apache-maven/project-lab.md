# Apache Maven — Hands-On Project Lab

> Build and package a Java project with Maven, including a custom profile.

## Objective

Get comfortable with Maven's dependency management and build lifecycle beyond the defaults.

## Prerequisites

- JDK installed
- Basic Java

## Steps

1. Scaffold a Maven project with `mvn archetype:generate`.
2. Add two dependencies (e.g. JUnit and a logging library) and resolve version conflicts if any appear.
3. Write and run a unit test with `mvn test`.
4. Add a `<profile>` for a 'prod' build that excludes test dependencies and enables optimizations.
5. Package the app as an executable JAR with the Shade or Assembly plugin.
6. Run `mvn dependency:tree` and explain (in the README) why one transitive dependency is pulled in.

## Deliverable

A runnable JAR built via `mvn -P prod package`, plus the dependency tree output with your explanation.

## Stretch goals

- Publish the JAR to a local Nexus repository (ties into the Nexus Repository lab).

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
