# Cybersecurity — Hands-On Project Lab

> Perform a basic OWASP Top 10 assessment of a sample web app.

## Objective

Learn to think like an attacker against a real (deliberately vulnerable) app, then fix what you find.

## Prerequisites

- A deliberately vulnerable app (e.g. OWASP Juice Shop or DVWA) run locally
- Browser dev tools

## Steps

1. Attempt a SQL injection against a login form and document whether it succeeds.
2. Attempt a reflected XSS via a search or comment field.
3. Check for broken access control by editing an object ID in a URL/API call to access another user's data.
4. Check HTTP response headers for missing security headers (CSP, X-Frame-Options, HSTS).
5. Check for a sensitive-data-exposure issue (e.g. passwords or tokens visible in network responses).
6. For each finding, write the fix (e.g. parameterized queries, output encoding, header config).

## Deliverable

A findings report (OWASP Top 10 style) listing each vulnerability found, its risk, and the specific fix.

## Stretch goals

- Re-test the app after applying at least one fix and confirm the exploit no longer works.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
