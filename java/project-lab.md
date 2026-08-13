# Java — Hands-On Project Lab

> Build a CLI inventory management app using core OOP principles.

## Objective

Apply classes, interfaces, and collections to a small but real problem instead of toy examples.

## Prerequisites

- JDK installed
- Basic programming knowledge

## Steps

1. Define an `Item` class and an `Inventory` class backed by a `Map<String, Item>`.
2. Define an interface `Sellable` with a `sell(int qty)` method and implement it on `Item`.
3. Build a CLI menu (add/remove/list/sell item) using a loop and `Scanner`.
4. Add input validation that rejects negative quantities without crashing the app.
5. Add a custom exception (`OutOfStockException`) thrown and caught explicitly.
6. Write JUnit tests for the `sell` method covering the out-of-stock case.

## Deliverable

A runnable CLI app with passing JUnit tests, including the out-of-stock exception path.

## Stretch goals

- Persist the inventory to a JSON file on exit and reload it on startup.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
