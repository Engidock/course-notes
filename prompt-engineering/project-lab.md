# Prompt Engineering — Hands-On Project Lab

> Iterate on prompts for a specific task and document what actually improved output.

## Objective

Treat prompting as an engineering discipline: hypothesis, test, measure, not just vibes.

## Prerequisites

- Access to any LLM
- A specific task, e.g. classifying support tickets by urgency

## Steps

1. Write a baseline zero-shot prompt for the task and run it against 10 sample inputs.
2. Record the accuracy/quality of the baseline against a hand-labeled answer key.
3. Write a few-shot version with 3 examples and re-run against the same 10 inputs.
4. Write a chain-of-thought version that asks the model to reason before answering, and re-run.
5. Compare all three versions on accuracy, consistency, and output-format compliance.
6. Write up which technique won and your best guess why.

## Deliverable

A comparison table (technique vs accuracy vs format compliance) across all three prompt versions, with your written analysis.

## Stretch goals

- Add a 4th version using structured output (JSON schema / function calling) and compare parse-failure rate.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
