# Serverless & Event-Driven Architecture — Hands-On Project Lab

> Build an S3-to-Lambda-to-DynamoDB event pipeline.

## Objective

Wire together a fully event-driven pipeline with no servers to manage.

## Prerequisites

- AWS account
- Basic Python or Node.js

## Steps

1. Create an S3 bucket for file uploads.
2. Write a Lambda function triggered on S3 `ObjectCreated` events.
3. Parse the uploaded file (e.g. CSV) inside the Lambda and extract records.
4. Write the parsed records to a DynamoDB table.
5. Add a dead-letter queue (SQS) for failed invocations.
6. Add CloudWatch alarms on Lambda errors.

## Deliverable

An uploaded sample file that ends up as queryable rows in DynamoDB, plus a screenshot of a triggered DLQ message from a deliberately malformed file.

## Stretch goals

- Add an API Gateway endpoint that queries DynamoDB and returns JSON.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
