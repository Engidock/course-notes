# ⚡ Serverless & Event-Driven Architecture Project Mastery

> **👋 Hey Fresher — Read This First!**

> Serverless means you write a function, upload it, and AWS runs it only when something triggers it — a file upload, a message in a queue, a scheduled time. You never provision a server, patch an OS, or pay for idle capacity. Event-driven means services don't call each other directly and wait; instead, one service publishes "this thing happened" and any interested service reacts, independently, without either one needing to know the other exists. This document uses **short, focused code blocks** — each one covers exactly one concept — with a plain-English explanation right beside it. One idea at a time, always explained simply.

> **Company in this project:** CarePulse Health — a telemedicine startup in Pune connecting patients with doctors for video consultations and e-prescriptions. Doctors currently upload signed prescription PDFs to a shared folder, and an intern manually types the medicine details into a spreadsheet for the pharmacy partner's records — a process that takes up to a day and has already caused two dosage transcription errors. Leadership wants prescriptions processed automatically, in seconds, the moment a doctor uploads one. You just joined as a Junior Backend/Cloud Engineer. Your senior mentor is Rohit. Let's build CarePulse's serverless prescription pipeline.

#### What You Will Learn and Build in This Project

You will build a fully event-driven pipeline for CarePulse — from the moment a doctor uploads a prescription file to the moment it's safely stored, queryable, and other systems have been notified — without provisioning a single server.

AWS Lambda, S3 Event Notifications, DynamoDB, SQS, Dead-Letter Queues, EventBridge, Step Functions, API Gateway, CloudWatch Alarms, IAM Least Privilege

> **📦 Phase 1 — S3-to-Lambda Foundations**
>
> Trigger a Lambda function automatically the instant a doctor uploads a prescription file to S3.

> **📦 Phase 2 — DynamoDB Data Modeling**
>
> Design a single-table DynamoDB schema to store parsed prescription records for fast, cheap lookups.

> **📦 Phase 3 — Reliability with SQS and Dead-Letter Queues**
>
> Make sure a malformed prescription file doesn't get silently dropped or retried forever.

> **📦 Phase 4 — Decoupling with EventBridge**
>
> Let the pharmacy-notification service and the analytics service react to new prescriptions independently, without Lambda calling either directly.

> **📦 Phase 5 — Exposing Data with API Gateway**
>
> Let the doctor's dashboard query prescription status through a clean REST API backed by Lambda.

> **📦 Phase 6 — Observability and Alarms**
>
> Know within seconds if prescriptions stop processing — before a doctor or patient notices.

**Scene 1 — CarePulse Health Office, Pune | The Manual Prescription Problem**

> **Rohit** _Senior Backend/Cloud Engineer — CarePulse Health_
>
> Neha, right now a doctor finishes a video call, uploads a signed PDF prescription to a shared drive, and an intern manually reads it and types the medicine names, dosages, and quantities into a spreadsheet for our pharmacy partner. Last month a "5mg" got typed as "50mg" for a blood pressure medication. Nobody was hurt, but it easily could have gone the other way. We need this pipeline automated end-to-end, and reliable — not "usually works."

> **Neha (You)** _Junior Backend/Cloud Engineer — Day 1 at CarePulse Health_
>
> I've written Lambda functions before, but always as isolated pieces triggered by API Gateway. I've never built a whole pipeline where one event triggers a chain of independent things happening.

> **Rohit** _Senior Backend/Cloud Engineer_
>
> That's exactly the mental shift. In event-driven design, the S3 upload doesn't "call" the pharmacy service and wait for a response. It publishes one fact — "a prescription file arrived" — to Lambda, which turns that into a structured record and publishes a second fact — "a prescription was parsed" — to EventBridge. Anything that cares can subscribe: pharmacy notifications, analytics, audit logging. None of them know about each other. Add a new subscriber next month, and you touch zero existing code.

> **Ananya** _Product Engineering Lead — CarePulse Health_
>
> And build in the assumption that things will fail — a doctor uploads a scanned image instead of a text PDF, a network blip corrupts an upload, a Lambda times out. In healthcare, a silently dropped prescription is not an acceptable failure mode. Every failure needs to land somewhere a human can see it, never just vanish.

### 1. Phase 1 — S3-to-Lambda Foundations

**Business Problem:** CarePulse needs prescription processing to start the instant a file lands in S3 — with zero polling, zero cron jobs checking "is there a new file yet." S3 Event Notifications combined with Lambda give a true push-based trigger.

#### 1.1 S3 Bucket with Event Notification Configured

```bash
aws s3api create-bucket \
  --bucket carepulse-prescriptions-raw \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

aws s3api put-bucket-notification-configuration \
  --bucket carepulse-prescriptions-raw \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [{
      "LambdaFunctionArn": "arn:aws:lambda:ap-south-1:111111111111:function:carepulse-parse-prescription",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [{ "Name": "suffix", "Value": ".pdf" }]
        }
      }
    }]
  }'
```

> **📖 Event Notifications, Not Polling**
>
> `s3:ObjectCreated:*` matches any way an object can be created — PUT, POST, or multipart upload completion — so it doesn't matter whether the doctor's browser uploads directly or through a backend service. The `suffix: ".pdf"` filter means the Lambda only fires for actual prescription PDFs, ignoring any stray `.tmp` or `.DS_Store` files that might land in the same bucket. This entire mechanism is push-based — S3 invokes Lambda directly the moment the object finishes uploading, with no polling Lambda that costs money while waiting.

#### 1.2 Grant S3 Permission to Invoke the Lambda

```bash
aws lambda add-permission \
  --function-name carepulse-parse-prescription \
  --statement-id AllowS3Invoke \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::carepulse-prescriptions-raw \
  --source-account 111111111111
```

> **📖 Resource-Based Policies — The Missing Step Freshers Forget**
>
> Configuring the S3 notification alone is not enough — S3 also needs *permission* to invoke the Lambda, granted via a resource-based policy attached directly to the Lambda function (separate from the Lambda's own execution role, which governs what the Lambda can do, not who can invoke it). `--source-arn` and `--source-account` scope this permission tightly: only this specific bucket, owned by this specific AWS account, can trigger this function — preventing a similarly-named bucket in another account from invoking it.

#### 1.3 The Lambda Function — Parse the Uploaded Prescription

```python
import json
import boto3
import urllib.parse

s3 = boto3.client("s3")

def handler(event, context):
    record = event["Records"][0]
    bucket = record["s3"]["bucket"]["name"]
    key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])

    print(f"New prescription uploaded: s3://{bucket}/{key}")

    response = s3.get_object(Bucket=bucket, Key=key)
    pdf_bytes = response["Body"].read()

    prescription = extract_prescription_fields(pdf_bytes)
    return {"statusCode": 200, "body": json.dumps(prescription)}
```

> **📖 Reading the S3 Event Payload**
>
> `event["Records"][0]` — S3 always wraps the notification in a `Records` array, even for a single upload, because the same Lambda invocation format is shared across many AWS event sources. `urllib.parse.unquote_plus` is essential: S3 URL-encodes object keys with spaces or special characters (a filename like `Dr Mehta Rx.pdf` arrives as `Dr+Mehta+Rx.pdf`), and forgetting to decode it causes a "key does not exist" error on the very next line when you try to fetch the object. The Lambda re-fetches the object from S3 using `get_object` — the event only tells you *that* something changed and *where*, never the file contents themselves.

**Lambda vs Fargate for This Workload**

- **Lambda** — pay only per invocation and execution time (billed to the nearest millisecond), scales to zero when idle, cold starts add ~100-500ms for Python. Use for: CarePulse's exact case — short, event-triggered, bursty prescription processing.
- **Fargate** — a container that runs continuously (or on a schedule), better for long-running or CPU/memory-heavy processing (e.g. OCR on scanned handwriting that takes minutes), no 15-minute execution limit. Use for: workloads that exceed Lambda's constraints or need a persistent process.

### 2. Phase 2 — DynamoDB Data Modeling

**Business Problem:** Once a prescription is parsed, CarePulse needs to store it somewhere queryable by both patient ID (for the patient's medical history) and doctor ID (for the doctor's dashboard) — fast, cheap, and without managing a database server.

**Scene 2 — CarePulse Data Modeling Session | "One Table, Many Access Patterns"**

> **Rohit** _Senior Backend/Cloud Engineer_
>
> With DynamoDB, you design the table around how you'll *query* it, not around "entities" the way you would in a relational database. We need to fetch "all prescriptions for patient X" and "all prescriptions written by doctor Y, this month" — both need to be fast without a full table scan.

#### 2.1 Create the Table with a Global Secondary Index

```bash
aws dynamodb create-table \
  --table-name CarePulsePrescriptions \
  --attribute-definitions \
      AttributeName=PatientId,AttributeType=S \
      AttributeName=PrescriptionId,AttributeType=S \
      AttributeName=DoctorId,AttributeType=S \
      AttributeName=CreatedAt,AttributeType=S \
  --key-schema \
      AttributeName=PatientId,KeyType=HASH \
      AttributeName=PrescriptionId,KeyType=RANGE \
  --global-secondary-indexes '[{
      "IndexName": "DoctorId-CreatedAt-index",
      "KeySchema": [
        {"AttributeName": "DoctorId", "KeyType": "HASH"},
        {"AttributeName": "CreatedAt", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
  }]' \
  --billing-mode PAY_PER_REQUEST
```

> **📖 Partition Key, Sort Key, and the GSI**
>
> **PatientId** as the partition key means all of one patient's prescriptions are stored together, so "get all prescriptions for patient X" is a single fast query. **PrescriptionId** as the sort key lets one patient have many prescriptions while keeping each uniquely addressable. The **Global Secondary Index** on `DoctorId` + `CreatedAt` gives a *second* efficient access pattern — "all prescriptions by doctor Y, most recent first" — without scanning the entire table, which DynamoDB's primary key alone can't serve. `PAY_PER_REQUEST` billing means CarePulse pays only for actual reads/writes, ideal for a workload with unpredictable, bursty traffic like ad-hoc doctor consultations.

#### 2.2 Write the Parsed Record from Lambda

```python
import boto3
import uuid
from datetime import datetime, timezone

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("CarePulsePrescriptions")

def save_prescription(patient_id, doctor_id, medicines, s3_key):
    table.put_item(Item={
        "PatientId":       patient_id,
        "PrescriptionId":  str(uuid.uuid4()),
        "DoctorId":        doctor_id,
        "CreatedAt":       datetime.now(timezone.utc).isoformat(),
        "Medicines":       medicines,
        "SourceFile":      s3_key,
        "Status":          "PROCESSED",
    })
```

> **📖 put_item — Schemaless by Design**
>
> DynamoDB only enforces types for the partition key and sort key (both `S` for string here) — every other attribute, like `Medicines` (a list of dicts with drug name, dosage, and quantity), can vary freely between items without a schema migration. `str(uuid.uuid4())` generates a globally unique prescription ID so two prescriptions uploaded in the same millisecond never collide. Storing `SourceFile` (the original S3 key) preserves a direct link back to the signed PDF for audit purposes — a legal requirement in healthcare record-keeping.

### 3. Phase 3 — Reliability with SQS and Dead-Letter Queues

**Business Problem:** A doctor occasionally uploads a scanned image instead of a text-based PDF, or a corrupted file. The parsing Lambda throws an exception. Without a safety net, that prescription silently vanishes — nobody at CarePulse knows a patient's prescription never made it to the pharmacy.

**Scene 3 — CarePulse Incident Review | "Where Did This Prescription Go?"**

> **Ananya** _Product Engineering Lead_
>
> A patient called the pharmacy asking where their prescription was. We checked DynamoDB — nothing. We checked S3 — the file was there. The Lambda had failed silently on a corrupted upload and there was no record of the failure anywhere. That's not acceptable for a healthcare product.

#### 3.1 Create a Dead-Letter Queue and Attach It to the Lambda

```bash
aws sqs create-queue --queue-name carepulse-prescription-dlq

aws lambda update-function-configuration \
  --function-name carepulse-parse-prescription \
  --dead-letter-config TargetArn=arn:aws:sqs:ap-south-1:111111111111:carepulse-prescription-dlq
```

> **📖 Dead-Letter Queue — Where Failures Go to Be Seen**
>
> S3 invokes Lambda **asynchronously** — S3 doesn't wait for a response. For asynchronous invocations, Lambda automatically retries a failed execution up to 2 more times (3 attempts total) with a delay between retries. If all attempts fail, Lambda sends the original event payload to the configured Dead-Letter Queue instead of discarding it. This turns a silent failure into a message sitting in a queue, exactly where an engineer or an automated alert can find it — the corrupted-PDF prescription from Ananya's story would now be sitting in `carepulse-prescription-dlq`, not gone.

#### 3.2 Give the Lambda Permission to Send to the DLQ

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:ap-south-1:111111111111:carepulse-prescription-dlq"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::carepulse-prescriptions-raw/*"
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem"],
      "Resource": "arn:aws:dynamodb:ap-south-1:111111111111:table/CarePulsePrescriptions"
    }
  ]
}
```

> **📖 Least-Privilege IAM for the Lambda's Execution Role**
>
> This policy is attached to the Lambda's **execution role** — it governs what the function's code is allowed to do once it's running, which is a completely separate concern from the resource-based policy from Phase 1 that governs who can *invoke* it. Notice each statement is scoped to one exact resource ARN — `s3:GetObject` only on the specific bucket, `dynamodb:PutItem` only on the specific table — never a wildcard `Resource: "*"`. If this Lambda were ever compromised, the blast radius is exactly these three actions on these three resources, nothing else in the AWS account.

#### 3.3 Redrive Policy on an SQS-Based Pipeline Stage

```bash
aws sqs set-queue-attributes \
  --queue-url https://sqs.ap-south-1.amazonaws.com/111111111111/carepulse-pharmacy-notify \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:ap-south-1:111111111111:carepulse-prescription-dlq\",\"maxReceiveCount\":\"3\"}"
  }'
```

> **📖 maxReceiveCount — When SQS Itself Uses a DLQ**
>
> This is a second, distinct pattern: for a standard SQS queue (like the pharmacy notification queue used later in Phase 4), `maxReceiveCount: 3` means if a consumer receives a message and fails to delete it (because processing threw an error) three times, SQS automatically moves that message to the DLQ instead of retrying forever. Without this, a permanently broken message would be redelivered infinitely, wasting compute and hiding the fact that something is permanently stuck.

> **Quiz: A prescription-parsing Lambda throws an unhandled exception on every retry for a corrupted file. Where does the event end up, and why does that matter for CarePulse?**
> - It's silently discarded after the retries are exhausted
> - It lands in the configured Dead-Letter Queue, preserving the original event for investigation
> - Lambda pauses all future invocations until someone manually intervenes
>
> > **Answer/explanation:** The second option. For asynchronous invocations (like S3 event notifications), Lambda retries automatically and, if a Dead-Letter Queue is configured, sends the failed event there after retries are exhausted rather than discarding it. This is exactly the fix for Ananya's incident — the corrupted prescription file's event now sits visibly in the DLQ instead of vanishing, and a CloudWatch alarm on the DLQ's message count can page an engineer immediately. Without a DLQ configured, the event genuinely is discarded — which is why setting one up is considered a production requirement, not an optional extra. Lambda does not pause other invocations on unrelated events; each invocation is independent.

### 4. Phase 4 — Decoupling with EventBridge

**Business Problem:** Once a prescription is successfully saved, three separate things need to happen: notify the pharmacy partner, log an audit event for compliance, and update a real-time analytics dashboard. If the parsing Lambda called all three directly, it would need to know about all three systems, and a slow or failing pharmacy API would delay or break the audit log too.

#### 4.1 Publish a Custom Event to EventBridge

```python
import boto3
import json

events = boto3.client("events")

def publish_prescription_processed(prescription):
    events.put_events(Entries=[{
        "Source":       "carepulse.prescriptions",
        "DetailType":   "PrescriptionProcessed",
        "Detail":       json.dumps(prescription),
        "EventBusName": "carepulse-event-bus",
    }])
```

> **📖 One Publish, Many Independent Subscribers**
>
> `Source` and `DetailType` are how EventBridge rules later filter which events they care about — think of them as the event's category and specific name. The parsing Lambda calls `put_events` exactly once and has zero knowledge of who's listening — it doesn't know the pharmacy service exists, doesn't know analytics exists. This is the core of event-driven decoupling: publishers and subscribers are never directly wired together, so adding a fourth subscriber next quarter requires zero changes to this function.

#### 4.2 EventBridge Rule Routing to the Pharmacy Notification Queue

```json
{
  "Name": "route-to-pharmacy-notify",
  "EventBusName": "carepulse-event-bus",
  "EventPattern": {
    "source": ["carepulse.prescriptions"],
    "detail-type": ["PrescriptionProcessed"]
  },
  "Targets": [
    {
      "Id": "PharmacyNotifyQueue",
      "Arn": "arn:aws:sqs:ap-south-1:111111111111:carepulse-pharmacy-notify"
    }
  ]
}
```

> **📖 Event Patterns — Content-Based Routing**
>
> The `EventPattern` acts as a filter: only events matching `source: carepulse.prescriptions` AND `detail-type: PrescriptionProcessed` get routed to this target. CarePulse can add a second rule on the *same* event bus, matching the *same* event, routing to a completely different target (like a Kinesis stream feeding the analytics dashboard) — one published event, multiple independent consumers, each with their own rule and their own failure isolation. If the pharmacy queue is down, the analytics rule is entirely unaffected.

**EventBridge vs Direct Lambda-to-Lambda Calls**

- **EventBridge** — publisher and subscribers are fully decoupled; add/remove subscribers without touching the publisher; built-in content-based filtering; slight added latency (typically under a second). Use for: CarePulse's fan-out to pharmacy notification, audit logging, and analytics.
- **Direct invocation (Lambda calling Lambda, or via SQS point-to-point)** — simpler for a true 1:1 relationship where the caller genuinely needs a synchronous response. Use for: a strict request/response need, like a Lambda behind API Gateway that must return a result to the caller immediately.

### 5. Phase 5 — Exposing Data with API Gateway

**Business Problem:** The doctor's dashboard needs to show prescription status in real time — "processing," "sent to pharmacy," "delivered" — via a clean HTTPS API, not by querying DynamoDB directly from the frontend.

#### 5.1 REST API Route Backed by Lambda

```yaml
# Simplified OpenAPI-style definition for API Gateway
paths:
  /prescriptions/{patientId}:
    get:
      summary: List a patient's prescriptions
      x-amazon-apigateway-integration:
        type: aws_proxy
        httpMethod: POST
        uri: arn:aws:apigateway:ap-south-1:lambda:path/2015-03-31/functions/arn:aws:lambda:ap-south-1:111111111111:function:carepulse-get-prescriptions/invocations
```

> **📖 Lambda Proxy Integration**
>
> `type: aws_proxy` means API Gateway forwards the entire HTTP request (path parameters, query string, headers, body) as-is into the Lambda's `event` object, and expects the Lambda to return a specific JSON shape (`statusCode`, `headers`, `body`) back — this is called Lambda proxy integration, and it's the standard pattern because it keeps all routing logic in code rather than split across API Gateway's mapping templates.

```python
import json
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("CarePulsePrescriptions")

def handler(event, context):
    patient_id = event["pathParameters"]["patientId"]
    response = table.query(
        KeyConditionExpression=Key("PatientId").eq(patient_id)
    )
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(response["Items"]),
    }
```

> **📖 Query, Not Scan**
>
> `table.query` with `KeyConditionExpression` uses the partition key directly, reading only the items belonging to this one patient — fast and cheap regardless of table size. A `table.scan()` instead would read every item in the entire table checking each one, which gets slower and more expensive as CarePulse's prescription history grows into the millions. This is the direct payoff of the data model designed back in Phase 2.

### 6. Phase 6 — Observability and Alarms

**Business Problem:** If the parsing Lambda starts failing at scale — a bad deploy, an AWS regional issue — CarePulse needs to know within minutes, not when a pharmacy calls asking where prescriptions went.

#### 6.1 CloudWatch Alarm on Lambda Errors

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name carepulse-prescription-lambda-errors \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=carepulse-parse-prescription \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 3 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:111111111111:carepulse-oncall-alerts
```

> **📖 Errors Metric and the Threshold**
>
> Lambda automatically emits an `Errors` metric — one data point per invocation that threw an unhandled exception. `period: 300` with `evaluation-periods: 1` means: sum up errors in any single 5-minute window; if that sum reaches 3 or more, fire the alarm. `alarm-actions` points at an SNS topic that fans out to Slack and PagerDuty for the on-call engineer. Three errors in five minutes is deliberately sensitive for a healthcare workload — CarePulse would rather get a false-positive page than miss a real pattern of failing prescriptions.

#### 6.2 Alarm on the Dead-Letter Queue Depth

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name carepulse-dlq-not-empty \
  --namespace AWS/SQS \
  --metric-name ApproximateNumberOfMessagesVisible \
  --dimensions Name=QueueName,Value=carepulse-prescription-dlq \
  --statistic Maximum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:111111111111:carepulse-oncall-alerts
```

> **📖 Any Message in the DLQ Is Worth Paging On**
>
> `threshold: 0` with `GreaterThanThreshold` means the alarm fires the instant even **one** message appears in the Dead-Letter Queue. For most systems that would be too noisy, but for CarePulse a single stuck prescription is a patient-safety issue, not background noise — the team would rather investigate one message immediately than let several accumulate.

> **Key takeaways**
> - S3 event notifications push Lambda invocations directly — no polling — but the Lambda needs both a resource-based policy (who can invoke it) and an execution role (what it can do) configured correctly.
> - DynamoDB tables are designed around access patterns, not entities — a Global Secondary Index adds a second efficient query path without duplicating a full second table.
> - Every asynchronous Lambda invocation needs a Dead-Letter Queue; a failure with no DLQ is a silent data-loss bug waiting to be discovered by a customer.
> - EventBridge decouples publishers from subscribers entirely — the publisher never needs to know who (or how many services) are listening, which makes adding new reactions to an event a zero-risk change.
> - CloudWatch alarms on Errors and DLQ depth turn a serverless pipeline from "hope it's working" into "know immediately when it's not."

> **Quiz: CarePulse wants to add a fourth reaction to "PrescriptionProcessed" — sending an SMS to the patient — without touching the existing parsing Lambda or any existing subscriber. What's the correct approach?**
> - Modify the parsing Lambda to also call the SMS service directly
> - Add a new EventBridge rule matching the same event pattern, targeting a new SMS-sending Lambda
> - Have the pharmacy notification service call the SMS service after it finishes
>
> > **Answer/explanation:** The second option. Because the parsing Lambda already publishes a `PrescriptionProcessed` event to EventBridge, adding a new subscriber is just a new rule with a matching event pattern and a new target — zero changes to the publisher or any existing subscriber, and no new coupling introduced. Modifying the parsing Lambda directly (the first option) re-introduces tight coupling and risks breaking the core pipeline for an unrelated feature. Chaining it off the pharmacy service (the third option) creates a hidden dependency — if pharmacy notification is ever slow or down, SMS sending would be affected for no logical reason, which defeats the entire purpose of event-driven design.

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Add a Step Functions state machine:** Replace the linear Lambda logic with an AWS Step Functions workflow that has explicit states for `ParsePrescription`, `ValidateDosage`, and `SaveToDynamoDB`, with a `Catch` block routing validation failures to a human-review SQS queue instead of the general DLQ.
2. **Add S3 object versioning and a lifecycle rule:** Enable versioning on the `carepulse-prescriptions-raw` bucket for audit purposes, and add a lifecycle rule that transitions prescriptions older than 90 days to S3 Glacier Instant Retrieval to cut storage cost.
3. **Add a DynamoDB Stream and a second Lambda:** Enable a DynamoDB Stream on `CarePulsePrescriptions` and write a second Lambda that reacts to new items by checking for known drug interaction pairs, publishing a `DrugInteractionWarning` event to EventBridge if found.
4. **Build a batch reprocessing script:** Write a script using `boto3` that lists every message currently sitting in the Dead-Letter Queue, attempts to redrive them into the original processing flow, and reports which ones still fail after a fix has been deployed.
5. **Add request throttling to API Gateway:** Configure a usage plan with a rate limit (e.g. 10 requests/second, burst of 20) on the `/prescriptions/{patientId}` route so a misbehaving dashboard client can't overwhelm the DynamoDB table.

### Serverless & Event-Driven Project Complete 🎉

You have built CarePulse Health's complete serverless prescription pipeline — S3-triggered Lambda parsing, a purpose-modeled DynamoDB table with a GSI, dead-letter queues protecting against silent failure, EventBridge decoupling three independent downstream reactions, an API Gateway endpoint for the doctor dashboard, and CloudWatch alarms watching the whole system. No servers were provisioned anywhere in this build.

> **Rohit**
>
> "Neha, before this, a prescription took up to a day to reach the pharmacy and depended on an intern typing it correctly. Now it's seconds, automated, and if anything fails, it's sitting visibly in a queue instead of just gone. That dosage transcription error we had last month — structurally, it can't happen the same way again."

> **Ananya**
>
> "What I like most is that adding SMS reminders next quarter, or a new analytics dashboard, doesn't mean touching this pipeline at all. We just add a new EventBridge rule. That's the real value of event-driven design — the system gets easier to extend over time, not harder."

> **Next: AWS CloudFormation — Turning This Pipeline Into a Repeatable Template**

> - Define the S3 bucket, Lambda functions, DynamoDB table, SQS queues, and EventBridge rules from this project as a single CloudFormation stack
> - Use CloudFormation's intrinsic functions to wire resource ARNs together instead of hardcoding them
> - Deploy identical staging and production environments from the same template with different Parameters
> - Preview every change with a changeset before it ever touches production data
