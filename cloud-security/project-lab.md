# 🔐 Cloud Security Project Mastery

> **👋 Hey Fresher — Read This First!**

> Cloud security in AWS isn't one product — it's a set of layers working together: IAM controls *who* can do *what*, CloudTrail records *every single API call* anyone makes, GuardDuty watches for *suspicious behavior* in real time, and AWS Config continuously checks that your resources stay *configured the way they should be*. Individually each is useful; together they turn "we hope nothing bad is happening" into "we would know within minutes if it were." This document uses **short, focused code blocks** — each one covers exactly one concept — with a plain-English explanation right beside it.

> **Company in this project:** TransWay Logistics — a fleet and freight management company in Chennai whose AWS account was set up in a hurry three years ago by a founding engineer who has since left. Every developer has `AdministratorAccess`, MFA is optional, nobody can say for certain what changed in production last week, and a routine customer security questionnaire ahead of a large enterprise contract just came back with 14 unanswered questions about access control and monitoring. Leadership needs the account audit-ready in three weeks. You just joined as a Junior Cloud Security Engineer. Your senior mentor is Karthik. Let's lock down TransWay's AWS account properly.

#### What You Will Learn and Build in This Project

You will take a loosely configured AWS account and systematically harden it — least-privilege IAM, mandatory MFA, full audit logging, automated threat detection, and continuous compliance checking — learning why each control exists and what specific incident it prevents.

IAM Policies, Least Privilege, MFA Enforcement, CloudTrail, GuardDuty, AWS Config, Config Rules, Security Hub, S3 Block Public Access, IAM Access Analyzer

> **📦 Phase 1 — IAM Foundations & Least Privilege**
>
> Replace blanket `AdministratorAccess` grants with scoped, least-privilege IAM policies tied to actual job functions.

> **📦 Phase 2 — Enforcing MFA Everywhere**
>
> Require multi-factor authentication on the root account and every IAM user before they can do anything else.

> **📦 Phase 3 — Full Audit Trail with CloudTrail**
>
> Record every API call made in the account, across every region, tamper-evident and durable.

> **📦 Phase 4 — Real-Time Threat Detection with GuardDuty**
>
> Automatically detect compromised credentials, port scanning, and unusual API activity.

> **📦 Phase 5 — Continuous Compliance with AWS Config**
>
> Catch misconfigurations — like a public S3 bucket — automatically, the moment they happen, not during the next audit.

> **📦 Phase 6 — Aggregating Everything in Security Hub**
>
> Bring IAM findings, GuardDuty alerts, and Config compliance into one dashboard the security team actually watches.

**Scene 1 — TransWay Logistics Office, Chennai | The Security Questionnaire**

> **Karthik** _Senior Cloud Security Architect — TransWay Logistics_
>
> Ishita, we're three weeks from potentially signing our biggest enterprise contract yet, and their security team sent back a questionnaire we can't honestly answer. "Do all users have MFA enabled?" No. "Is API activity logged and retained?" Not reliably. "How do you detect compromised credentials?" We don't. Every one of these is fixable, but we need to actually do the work, not just answer "yes" and hope nobody checks.

> **Ishita (You)** _Junior Cloud Security Engineer — Day 1 at TransWay Logistics_
>
> I've read about IAM and GuardDuty in theory, but I've never had to actually clean up an account that's been running loose for years. Where's the highest-risk thing to fix first?

> **Karthik** _Senior Cloud Security Architect_
>
> Access first, always. If everyone has `AdministratorAccess`, nothing else we do matters — a single compromised laptop is a compromised entire AWS account. We fix who-can-do-what before we worry about detecting bad behavior, because right now almost any behavior is "allowed" by policy. Least privilege first, then MFA, then we start watching.

> **Devansh** _Compliance & Risk Lead — TransWay Logistics_
>
> And document as you go. For an enterprise customer, "we fixed it" isn't enough — we need to show *evidence*: CloudTrail logs proving MFA is enforced, GuardDuty findings showing zero unresolved highs, a Config dashboard showing our S3 buckets are compliant. Security work that can't be evidenced might as well not have happened, from an auditor's point of view.

### 1. Phase 1 — IAM Foundations & Least Privilege

**Business Problem:** Every one of TransWay's 18 developers currently has `AdministratorAccess` — full control over the entire AWS account, including billing, IAM itself, and every customer's shipment data. A single phished laptop is a full account compromise. Least privilege means each person and service can do exactly what their job requires, nothing more.

#### 1.1 Audit Current Access with IAM Access Analyzer

```bash
aws accessanalyzer create-analyzer \
  --analyzer-name transway-account-analyzer \
  --type ACCOUNT

aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-south-1:333333333333:analyzer/transway-account-analyzer
```

> **📖 IAM Access Analyzer — Finding What's Actually Exposed**
>
> Access Analyzer uses automated reasoning to examine resource policies (S3 bucket policies, IAM role trust policies, KMS key policies) and flags anything accessible from **outside** the account — a genuinely different job from checking whether users have too much access internally. `list-findings` surfaces things like an S3 bucket policy that unintentionally grants read access to any AWS account, or an IAM role whose trust policy allows any external identity to assume it. This is the first thing to run — it tells you what's exposed *right now*, before you start rewriting any policies.

#### 1.2 Write a Least-Privilege Policy for the Fleet-Tracking Developer Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadWriteFleetTable",
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:Query"],
      "Resource": "arn:aws:dynamodb:ap-south-1:333333333333:table/TransWay-FleetTracking"
    },
    {
      "Sid": "ReadOnlyDeployLogs",
      "Effect": "Allow",
      "Action": ["logs:GetLogEvents", "logs:FilterLogEvents"],
      "Resource": "arn:aws:logs:ap-south-1:333333333333:log-group:/transway/fleet-api:*"
    },
    {
      "Sid": "DenyIAMChanges",
      "Effect": "Deny",
      "Action": ["iam:*", "organizations:*"],
      "Resource": "*"
    }
  ]
}
```

> **📖 Scoped Actions, Scoped Resources, Explicit Denies**
>
> Every `Resource` is a specific ARN, never `"*"` — this developer's policy can touch exactly one DynamoDB table and one log group, nothing else in the account. The explicit `Deny` on `iam:*` is a deliberate guardrail: even if this role is later accidentally attached to a broader policy through some other path, an explicit Deny always wins over any Allow in IAM's evaluation logic, so this developer can never modify IAM permissions or the AWS Organization no matter what else changes. This is the direct replacement for the blanket `AdministratorAccess` TransWay's developers had before.

#### 1.3 Attach Policies via Groups, Never Directly to Users

```bash
aws iam create-group --group-name transway-fleet-developers

aws iam put-group-policy \
  --group-name transway-fleet-developers \
  --policy-name FleetTrackingLeastPrivilege \
  --policy-document file://fleet-developer-policy.json

aws iam add-user-to-group --user-name priya.dev --group-name transway-fleet-developers
```

> **📖 Groups Over Direct User Policies**
>
> Attaching policies to a **group** instead of individual users means every developer on the fleet-tracking team automatically has identical, reviewable permissions, and onboarding a new hire is `add-user-to-group` — one command, guaranteed to match everyone else's access. Attaching policies directly to individual users, by contrast, tends to drift over time as one-off permissions get bolted on for "just this one task" and never removed — this is exactly how TransWay ended up with 18 different, undocumented, effectively-admin permission sets.

**IAM Policies vs SCPs (Service Control Policies) — Two Different Layers**

- **IAM Policies** — attached to users, groups, or roles; define the maximum a specific identity can do *within* the account it's attached in.
- **Service Control Policies** — attached at the AWS Organizations level, to an entire account or organizational unit; define the maximum *any* identity in that account can ever do, regardless of their IAM policy. Use SCPs for org-wide guardrails (e.g. "no one, including admins, can disable CloudTrail in any account") that individual account-level IAM policies can't override.

### 2. Phase 2 — Enforcing MFA Everywhere

**Business Problem:** TransWay's root account and IAM users can currently log in with just a password. A leaked or phished password is a full account takeover. Multi-factor authentication means a stolen password alone is useless without the second factor.

**Scene 2 — TransWay Security Review | "The Root Account Has No MFA"**

> **Karthik** _Senior Cloud Security Architect_
>
> The root account — the one with unrestricted access to everything, including the ability to close the entire AWS account — currently has no MFA. If that password ever leaks, there is no second gate at all. This gets fixed today, before anything else.

#### 2.1 Enable MFA on the Root Account and IAM Users

```bash
# Root MFA must be enabled through the AWS Console (no API for root MFA setup)
# For IAM users, enable a virtual MFA device via CLI:
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name ishita-mfa \
  --outfile ishita-mfa-qrcode.png \
  --bootstrap-method QRCodePNG

aws iam enable-mfa-device \
  --user-name ishita.security \
  --serial-number arn:aws:iam::333333333333:mfa/ishita-mfa \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

> **📖 Two Consecutive Codes to Prove Possession**
>
> `enable-mfa-device` requires two consecutive codes from the authenticator app (30 seconds apart) rather than one — this proves the person setting it up actually has the physical device generating a continuously rotating code, not just a lucky single guess. Root account MFA specifically must be configured by logging in as root through the AWS Console — there is no API for enabling root MFA, which is a deliberate AWS design choice to keep root account changes tied to an authenticated console session.

#### 2.2 IAM Policy That Denies Everything Unless MFA Is Present

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllExceptMFASetupUnlessMFAPresent",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:ListMFADevices",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" }
      }
    }
  ]
}
```

> **📖 The MFA Enforcement Pattern**
>
> `NotAction` inverts the statement: this Deny applies to *every action except* the ones listed — meaning a user without MFA can still log in and set up their own MFA device, but can do absolutely nothing else. `aws:MultiFactorAuthPresent: false` checks whether the current session was authenticated with MFA at all. `BoolIfExists` handles the edge case where this condition key might not exist on certain request types, avoiding an unintended blanket deny. Attach this to every IAM group — it's the standard pattern for making MFA effectively mandatory, not just recommended.

> **Quiz: A developer's IAM policy grants `s3:*` on a specific bucket, but a separately attached policy has an explicit `Deny` on `s3:DeleteObject` for that same bucket. What happens when they try to delete an object?**
> - The delete succeeds because the Allow was more specific
> - The delete is denied — an explicit Deny always overrides any Allow, regardless of which policy it's in
> - AWS asks the user which policy should take precedence
>
> > **Answer/explanation:** The second option. IAM's evaluation logic is: start with an implicit deny, check all applicable policies for an explicit Deny (if found, deny wins immediately, full stop), otherwise check for an Allow. Explicit Deny statements always take precedence over Allow statements, no matter which policy, group, or role they come from, and no matter how "specific" the Allow seems. This is exactly why the `DenyIAMChanges` statement in Phase 1 is safe to rely on as a guardrail — no other policy anyone attaches later can override it.

### 3. Phase 3 — Full Audit Trail with CloudTrail

**Business Problem:** Devansh needs to answer "what changed in production last week and who did it" for the enterprise questionnaire — and right now, TransWay has no reliable way to answer that question at all.

#### 3.1 Create a Multi-Region Trail with Log File Validation

```bash
aws cloudtrail create-trail \
  --name transway-org-trail \
  --s3-bucket-name transway-cloudtrail-logs \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name transway-org-trail
```

> **📖 Multi-Region and Log File Validation**
>
> `--is-multi-region-trail` captures API activity in **every** AWS region, not just `ap-south-1` — without this, an attacker (or a careless admin) could operate quietly in an unmonitored region like `us-east-2` and leave no trace in TransWay's logs at all. `--enable-log-file-validation` has CloudTrail generate a cryptographic digest for each delivered log file, so if anyone tampers with or deletes a log file after the fact, the digest mismatch proves it — critical for the log integrity guarantee an auditor or a court would actually trust.

#### 3.2 Lock Down the CloudTrail Log Bucket

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDeleteOfCloudTrailLogs",
      "Effect": "Deny",
      "Principal": "*",
      "Action": ["s3:DeleteObject", "s3:DeleteBucket"],
      "Resource": [
        "arn:aws:s3:::transway-cloudtrail-logs",
        "arn:aws:s3:::transway-cloudtrail-logs/*"
      ],
      "Condition": {
        "StringNotEquals": { "aws:PrincipalArn": "arn:aws:iam::333333333333:role/TransWaySecurityAdmin" }
      }
    }
  ]
}
```

> **📖 Protecting the Evidence Itself**
>
> This bucket policy denies deletion of any object (or the bucket itself) to everyone except one dedicated security-admin role — even a compromised developer role with broad S3 permissions elsewhere cannot delete these logs. This directly addresses a real attack pattern: an attacker who gains access often tries to delete CloudTrail logs to cover their tracks. Combined with S3 versioning and MFA Delete on the bucket (an even stronger protection requiring MFA for any delete or version-suspend action), this makes the audit trail genuinely tamper-resistant, not just tamper-evident.

#### 3.3 Query Recent Activity with CloudTrail Lake or Athena

```sql
-- Athena query against CloudTrail logs (table pre-registered via Glue Crawler)
SELECT
  eventtime, eventname, useridentity.arn, sourceipaddress
FROM transway_cloudtrail_logs
WHERE eventname IN ('DeleteBucket', 'PutBucketPolicy', 'AuthorizeSecurityGroupIngress')
  AND eventtime > '2026-08-06T00:00:00Z'
ORDER BY eventtime DESC;
```

> **📖 Turning Raw Logs Into an Answerable Question**
>
> This is exactly the query that answers Devansh's "what changed and who did it" question — filtering for high-impact event names like opening a security group or changing a bucket policy, over the past week. Raw CloudTrail JSON files in S3 aren't practically searchable at scale; registering them as an Athena table (via a Glue Crawler) turns months of logs into something queryable with standard SQL in seconds, which is what actually makes the audit trail useful during an incident rather than just theoretically complete.

### 4. Phase 4 — Real-Time Threat Detection with GuardDuty

**Business Problem:** CloudTrail tells TransWay what happened *after the fact*. GuardDuty is the layer that actively looks for suspicious patterns — a credential being used from an unusual location, an EC2 instance suddenly port-scanning internally, API calls consistent with reconnaissance — and alerts in near real time.

**Scene 3 — TransWay Security Slack | "Is Someone in Our Account Right Now?"**

> **Ishita (You)** _Junior Cloud Security Engineer_
>
> CloudTrail shows us the history, but if someone's credentials are compromised right now, how would we actually know today, not after reviewing logs next week?

> **Karthik** _Senior Cloud Security Architect_
>
> That's exactly what GuardDuty is for. It doesn't wait for you to query logs — it continuously analyzes CloudTrail, VPC Flow Logs, and DNS logs using threat intelligence and anomaly detection, and pushes a finding the moment it sees something like an API call from an unusual location for that user, or an EC2 instance suddenly talking to a known cryptomining pool.

#### 4.1 Enable GuardDuty

```bash
aws guardduty create-detector --enable

aws guardduty list-findings \
  --detector-id 8b0a1c2d3e4f5a6b7c8d9e0f1a2b3c4d \
  --finding-criteria '{"Criterion":{"severity":{"Gte":7}}}'
```

> **📖 Zero Infrastructure, Immediate Coverage**
>
> `create-detector --enable` is the entire setup — GuardDuty requires no agents to install and no sensors to deploy; it works directly off logs and network metadata AWS already has access to. `finding-criteria` with `severity: Gte 7` filters for only High and Critical severity findings (GuardDuty scores 0.1–8.9, with 7+ generally treated as High/Critical) — this is the exact list Devansh wants to be able to show as "zero unresolved highs" for the enterprise questionnaire.

#### 4.2 Route Findings to Slack and PagerDuty via EventBridge

```json
{
  "Name": "guardduty-high-severity-to-oncall",
  "EventPattern": {
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": { "severity": [{ "numeric": [">=", 7] }] }
  },
  "Targets": [
    { "Id": "OnCallSNS", "Arn": "arn:aws:sns:ap-south-1:333333333333:transway-security-oncall" }
  ]
}
```

> **📖 From Finding to Human, in Seconds**
>
> GuardDuty findings are automatically published as EventBridge events — no polling required. This rule filters for `severity >= 7` and routes only those to the on-call SNS topic (fanning out to Slack and PagerDuty), so the security team isn't paged for every low-severity informational finding, only the ones that genuinely need immediate human attention. Lower-severity findings still land in the GuardDuty console for weekly review, just without waking anyone up at 2 AM.

> **GuardDuty vs AWS Config — Different Questions, Both Needed**
>
> - **GuardDuty** — answers "is something malicious or anomalous happening right now?" using threat intelligence and behavioral analysis. Detects active threats: compromised credentials, malware communication, reconnaissance.
> - **AWS Config** — answers "is this resource configured the way our policy says it should be?" using rule-based configuration checks. Detects drift and misconfiguration: a public S3 bucket, an unencrypted volume, a security group open to `0.0.0.0/0`.

### 5. Phase 5 — Continuous Compliance with AWS Config

**Business Problem:** During the account audit, TransWay found a publicly-readable S3 bucket containing customer shipment manifests — nobody remembers who made it public or when. Fixing it once isn't enough; TransWay needs to know automatically if it (or any bucket) ever becomes public again.

#### 5.1 Enable AWS Config and the Managed Rule for Public Buckets

```bash
aws configservice put-configuration-recorder \
  --configuration-recorder name=transway-recorder,roleARN=arn:aws:iam::333333333333:role/AWSConfigServiceRole \
  --recording-group allSupported=true,includeGlobalResourceTypes=true

aws configservice start-configuration-recorder --configuration-recorder-name transway-recorder

aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    }
  }'
```

> **📖 Continuous Recording Plus a Managed Rule**
>
> The configuration recorder continuously tracks the configuration state of every supported resource in the account and stores a full history — this alone lets you answer "what did this bucket's policy look like on Tuesday." Layered on top, `S3_BUCKET_PUBLIC_READ_PROHIBITED` is an AWS-managed rule (no custom code needed) that evaluates every S3 bucket's policy and ACLs against the rule's definition, marking it `NON_COMPLIANT` the moment any bucket becomes publicly readable — including buckets created after this rule was set up.

#### 5.2 Auto-Remediate a Public Bucket

```bash
aws configservice put-remediation-configurations \
  --remediation-configurations '[{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "TargetType": "SSM_DOCUMENT",
    "TargetId": "AWSConfigRemediation-RemoveVPCDefaultSecurityGroupRules",
    "Automatic": true,
    "MaximumAutomaticAttempts": 3
  }]'
```

> **📖 From Detection to Automatic Fix**
>
> Attaching a remediation configuration means AWS Config doesn't just flag the non-compliant resource — it triggers an SSM Automation document to fix it, automatically, up to `MaximumAutomaticAttempts` times. For TransWay's actual public-bucket incident, the equivalent remediation is `AWSConfigRemediation-EnableS3BucketPublicAccessBlock`, which re-enables Block Public Access on the offending bucket the moment Config detects it went public — turning a days-long manual incident response into a fix applied within minutes of the misconfiguration occurring.

#### 5.3 Enforce S3 Block Public Access at the Account Level

```bash
aws s3control put-public-access-block \
  --account-id 333333333333 \
  --public-access-block-configuration \
      BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

> **📖 An Account-Wide Backstop**
>
> This setting applies to **every** S3 bucket in the account, present and future, overriding any individual bucket policy or ACL that would otherwise make an object public. `BlockPublicAcls` and `IgnorePublicAcls` stop public ACLs from being set (or honored, even if one already exists) on any object or bucket; `BlockPublicPolicy` and `RestrictPublicBuckets` do the equivalent for bucket policies. Setting this at the account level means TransWay's original incident — someone accidentally set a bucket policy allowing public reads — becomes structurally close to impossible, rather than something that depends on every engineer remembering to check.

> **Key takeaways**
> - Fix access (IAM least privilege) before worrying about detection — if everyone is effectively an admin, monitoring tools are just watching a wide-open door.
> - MFA should be enforced with policy, not just requested — a Deny statement keyed on `aws:MultiFactorAuthPresent` makes it structurally required, not optional.
> - CloudTrail is only trustworthy evidence if the log bucket itself is protected from deletion — logging without protecting the logs is not real auditability.
> - GuardDuty answers "is something bad happening right now"; AWS Config answers "is this resource configured correctly right now" — a mature security posture needs both, they are not substitutes for each other.
> - Prefer account-wide backstops (like S3 Block Public Access at the account level) over per-resource discipline wherever AWS offers one — it removes the possibility of human error entirely for that specific risk.

### 6. Phase 6 — Aggregating Everything in Security Hub

**Business Problem:** Devansh now has IAM policies, CloudTrail logs, GuardDuty findings, and Config compliance results — four different places to check. For the security team to actually watch this daily, it needs to be one dashboard, not four browser tabs.

#### 6.1 Enable Security Hub with the CIS AWS Foundations Standard

```bash
aws securityhub enable-security-hub \
  --enabled-standards '[{"StandardsArn":"arn:aws:securityhub:ap-south-1::standards/cis-aws-foundations-benchmark/v/1.4.0"}]'

aws securityhub get-findings \
  --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}],"RecordState":[{"Value":"ACTIVE","Comparison":"EQUALS"}]}'
```

> **📖 One Dashboard, Many Sources**
>
> Security Hub automatically ingests findings from GuardDuty, Config, IAM Access Analyzer, and Inspector into a single normalized format (the AWS Security Finding Format), so the security team checks one place instead of four. Enabling the **CIS AWS Foundations Benchmark** standard runs a broad, industry-recognized set of automated checks — root account usage, unused credentials, password policy strength — and scores TransWay's account against it, which doubles as ready-made evidence for exactly the kind of enterprise security questionnaire Devansh needs to answer.

> **Quiz: TransWay's security team wants a single view combining GuardDuty threat findings, Config compliance results, and IAM Access Analyzer findings — without building custom integration code. What's the right AWS service for this?**
> - CloudWatch Dashboards
> - AWS Security Hub
> - AWS Trusted Advisor
>
> > **Answer/explanation:** AWS Security Hub. It's purpose-built to aggregate findings from GuardDuty, Config, IAM Access Analyzer, Inspector, and other AWS security services (plus many third-party tools) into one normalized, filterable view, with built-in compliance standards like CIS AWS Foundations. CloudWatch Dashboards can visualize metrics but doesn't natively understand or aggregate security findings across these specific services. Trusted Advisor gives general account best-practice checks (cost, performance, some security) but isn't the aggregation layer for GuardDuty/Config/Access Analyzer findings specifically — that's Security Hub's core purpose.

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Write a custom Config rule with a Lambda:** Build a custom AWS Config rule (backed by a Lambda function) that flags any EC2 security group allowing inbound traffic on port 22 from `0.0.0.0/0`, and test it by intentionally opening a security group to trigger the finding.
2. **Set up cross-account CloudTrail aggregation:** If TransWay creates a second AWS account for a new business unit, configure an AWS Organizations-level CloudTrail trail so both accounts' logs land in one central, security-account-owned S3 bucket.
3. **Create a break-glass emergency access role:** Design an IAM role with broad permissions but a very short session duration and mandatory MFA, intended only for genuine emergencies, with CloudTrail alerting anyone whenever it's assumed.
4. **Simulate a GuardDuty finding:** Use GuardDuty's built-in sample findings generator (`aws guardduty create-sample-findings`) to populate example findings of every type, then practice triaging each one and writing a one-paragraph response plan for the highest severity example.
5. **Build an IAM permissions boundary:** Create a permissions boundary policy that caps what any role created by TransWay's Terraform pipeline can ever be granted, even if a future IAM policy attached to that role is overly broad by mistake.

### Cloud Security Project Complete 🎉

You have taken TransWay Logistics' loosely configured AWS account and hardened it end-to-end — least-privilege IAM policies replacing blanket admin access, enforced MFA, a tamper-evident multi-region CloudTrail, real-time GuardDuty threat detection, automatically remediated Config compliance rules, and a Security Hub dashboard aggregating all of it. This is the exact set of controls an enterprise security questionnaire is checking for.

> **Karthik**
>
> "Ishita, three weeks ago any one of 18 laptops was effectively a master key to our entire AWS account. Today, every identity has exactly the access its job requires, MFA is structurally enforced, and if anything suspicious happens, GuardDuty pages someone within minutes instead of us finding out from a customer."

> **Devansh**
>
> "I answered all 14 questions on that security questionnaire this morning — not from memory, from evidence. CloudTrail logs showing MFA enforcement, a Config dashboard at 100% compliance on the public-bucket rule, GuardDuty at zero unresolved highs. That's the difference between claiming we're secure and being able to prove it."

> **Next: AWS CloudFormation — Codifying These Security Controls So They Never Drift**

> - Turn TransWay's IAM policies, CloudTrail trail, GuardDuty detector, and Config rules into a CloudFormation stack, deployed identically to every new AWS account TransWay creates
> - Use a StackSet to roll out the same security baseline across an entire AWS Organization automatically
> - Detect configuration drift on the security stack itself — if someone manually disables a Config rule, know immediately
