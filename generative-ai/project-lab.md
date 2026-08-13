# Generative AI — Hands-On Project Lab

> Build a small app that uses an LLM API to summarize or generate content.

## Objective

Go from 'chatting with an AI in a browser' to calling a model programmatically inside a real app.

## Prerequisites

- An LLM API key
- Basic Python or JavaScript

## Steps

1. Call the LLM API from a script with a single hardcoded prompt and print the response.
2. Parameterize the prompt so it accepts user input (e.g. paste an article, get a summary).
3. Add basic error handling for rate limits and empty/invalid responses.
4. Add a system prompt that constrains the output format (e.g. always return 3 bullet points).
5. Cache identical requests so you don't re-call the API for the same input.
6. Add a simple CLI or web front-end so a non-technical user could use it.

## Deliverable

A working app that reliably returns the constrained output format for 5 different inputs.

## Stretch goals

- Add streaming responses so output appears token-by-token instead of all at once.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
