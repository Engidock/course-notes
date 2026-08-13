# JavaScript — Hands-On Project Lab

> Build an interactive to-do app with vanilla JavaScript.

## Objective

Practice DOM manipulation, events, and async fetch without a framework doing it for you.

## Prerequisites

- A code editor and a browser

## Steps

1. Build the HTML skeleton: input, add button, and an empty list.
2. Add items to the DOM on button click and on Enter keypress.
3. Add a delete button per item that removes just that item from the DOM.
4. Persist the list by fetching/saving to a mock API (e.g. json-server or localStorage-free in-memory server).
5. Add a 'mark complete' toggle that updates styling without re-rendering the whole list.
6. Debounce a search/filter input so it doesn't re-filter on every single keystroke.

## Deliverable

A working to-do app with add/complete/delete/filter, with no page reloads.

## Stretch goals

- Refactor the DOM logic into small pure functions and add a couple of unit tests with Vitest/Jest.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
