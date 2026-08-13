# Groovy Scripting — Hands-On Project Lab

> Write a Groovy shared-library function for a Jenkins pipeline.

## Objective

Move reusable pipeline logic out of individual Jenkinsfiles and into a shared, testable library.

## Prerequisites

- A Jenkins instance
- Basic Groovy/Java syntax

## Steps

1. Set up a Jenkins Shared Library repo with the standard `vars/` directory structure.
2. Write a `vars/buildAndTest.groovy` function that runs a build and test step.
3. Reference the shared library from a Jenkinsfile and call your function.
4. Add a parameter to the function (e.g. which build tool to use) and branch logic on it.
5. Add basic error handling (`try/catch`) that fails the stage with a clear message.
6. Write a Groovy unit test for the function's logic outside of Jenkins, using Spock or JUnit.

## Deliverable

A Jenkinsfile that's under 20 lines because the real logic lives in the tested shared library function.

## Stretch goals

- Add a second shared library function for deployment and chain them together.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
