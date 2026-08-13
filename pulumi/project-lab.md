# 🛠️ Pulumi Project Mastery

> **👋 Hey Fresher — Read This First!**

> Pulumi is Infrastructure-as-Code written in a real programming language — TypeScript, Python, Go, or others — instead of a YAML or HCL templating language. You get actual loops, functions, classes, and your editor's autocomplete and type-checking, and Pulumi translates your code into API calls that create, update, and track real cloud resources, the same way Terraform or CloudFormation does under the hood. This document uses **short, focused code blocks** — each one covers exactly one concept — with a plain-English explanation right beside it.

> **Company in this project:** LearnOrbit — an edtech startup in Delhi running live and recorded classes for competitive exam prep, serving over 200,000 students. Their infrastructure was originally written in raw CloudFormation YAML, and the platform team recently spent two full days trying to add a fifth near-identical regional deployment (for a new UP/Bihar-focused content cluster) because the template had no real way to loop — just four copy-pasted, slightly-diverging blocks of YAML. Leadership approved a rewrite in Pulumi specifically to get real code reuse. You just joined as a Junior Platform Engineer. Your senior mentor is Yash. Let's rebuild LearnOrbit's infrastructure in Pulumi.

#### What You Will Learn and Build in This Project

You will provision LearnOrbit's infrastructure using Pulumi and TypeScript — networking, compute, and storage — learning how real language constructs eliminate the copy-paste duplication that plagued their old templates, and how Pulumi's state, preview, and update model compares to declarative tools.

Pulumi CLI, Stacks, Pulumi Config, TypeScript Resources, Loops for Multi-Region Deployment, Outputs and StackReferences, Preview and Update, State Management, Component Resources, Testing

> **📦 Phase 1 — Project and Stack Setup**
>
> Initialize a Pulumi project in TypeScript and understand stacks as Pulumi's equivalent of separate environments.

> **📦 Phase 2 — Networking with Real Code**
>
> Provision a VPC and subnets, using a loop to avoid repeating the same block per Availability Zone.

> **📦 Phase 3 — Compute, Storage, and Config**
>
> Add EC2 instances and an S3 bucket driven by Pulumi Config, with secrets handled correctly.

> **📦 Phase 4 — Preview, Update, and Diffing**
>
> Use `pulumi preview` to see exactly what will change, and understand in-place updates vs replacements.

> **📦 Phase 5 — Reusable Component Resources**
>
> Package LearnOrbit's regional deployment pattern into a reusable component, solving the "four copy-pasted blocks" problem for good.

> **📦 Phase 6 — Testing and State Management**
>
> Write a unit test for the infrastructure code itself, and understand how Pulumi tracks state versus CloudFormation's built-in tracking.

**Scene 1 — LearnOrbit Engineering Office, Delhi | Two Days for One Copy-Paste**

> **Yash** _Senior Platform Engineer — LearnOrbit_
>
> Divya, our old CloudFormation template has four nearly-identical blocks — one per region we operate in — each one a full VPC, subnet set, and EC2 fleet definition. Adding region five meant copying 80 lines of YAML, changing six values by hand, and hoping I didn't miss one. It took two days including the review cycle to catch a mistyped CIDR block that would have overlapped with region three's range.

> **Divya (You)** _Junior Platform Engineer — Day 1 at LearnOrbit_
>
> I've written scripts before, but never infrastructure code. Isn't that basically the same idea — writing a program that creates servers instead of one that processes data?

> **Yash** _Senior Platform Engineer_
>
> Almost exactly that idea. In Pulumi, "define a VPC for a region" is a function. "Define it for all five regions" is a loop calling that function five times with different arguments. The mistyped CIDR block that took two days to catch becomes a compile-time type error in your editor before you ever run `pulumi up`. That's the entire pitch of Pulumi over YAML-based tools: it's not a new way to describe infrastructure, it's your existing programming skills applied to infrastructure.

> **Meenal** _Staff Engineer — LearnOrbit_
>
> Just remember Pulumi still ends up calling the same AWS APIs as CloudFormation or Terraform — a VPC is a VPC. What's different is entirely the *authoring* experience and what your language gives you for free: loops, functions, real unit tests, and your IDE catching mistakes before deployment instead of during it.

### 1. Phase 1 — Project and Stack Setup

**Business Problem:** LearnOrbit needs a Pulumi project structured cleanly from day one, with a real separation between dev, staging, and each production region — mirroring what "environments" meant in their old CloudFormation setup, but managed through Pulumi's native stack concept.

#### 1.1 Initialize the Project

```bash
mkdir learnorbit-infra && cd learnorbit-infra

pulumi new aws-typescript --name learnorbit-infra

# Creates: Pulumi.yaml, index.ts, package.json, tsconfig.json
```

> **📖 pulumi new — Scaffolding a Real TypeScript Project**
>
> `pulumi new aws-typescript` scaffolds a genuine Node.js/TypeScript project — `package.json` and `tsconfig.json` are standard files any TypeScript developer already recognizes, not something Pulumi-specific to learn from scratch. `Pulumi.yaml` records the project name and the runtime (`nodejs`) Pulumi uses to execute the code. Unlike a CloudFormation template, `index.ts` is executable code — Pulumi actually *runs* it (via Node.js) to determine what resources should exist, rather than parsing it as static declarative data.

#### 1.2 Create Stacks for Each Environment

```bash
pulumi stack init dev
pulumi stack init staging
pulumi stack init production-delhi
pulumi stack init production-mumbai

pulumi stack select production-delhi
pulumi stack ls
```

> **📖 Stacks — Pulumi's Isolated Environments**
>
> A **stack** is an independently configured, independently deployed instance of the *same* program — `production-delhi` and `production-mumbai` run identical `index.ts` code but hold entirely separate state and configuration values. This maps directly onto what LearnOrbit's four copy-pasted CloudFormation blocks were trying to achieve, except the underlying code is written once. `pulumi stack select` switches which stack subsequent commands (`preview`, `up`, `destroy`) operate against — a safety-critical command, since running `pulumi destroy` against the wrong stack destroys real infrastructure.

#### 1.3 Set Region-Specific Configuration Per Stack

```bash
pulumi config set aws:region ap-south-1 --stack production-delhi
pulumi config set aws:region ap-south-1 --stack production-mumbai
pulumi config set instanceCount 4 --stack production-delhi
pulumi config set instanceCount 2 --stack staging
```

> **📖 Pulumi Config — Per-Stack Values**
>
> `pulumi config set` writes values into a `Pulumi.<stack-name>.yaml` file — one per stack, checked into Git alongside the code (unless the value is a secret, covered in Phase 3). `aws:region` is a well-known configuration key the AWS provider reads automatically. `instanceCount` is a custom key LearnOrbit defines and reads inside `index.ts` — this is the direct equivalent of a CloudFormation Parameter, except values live in per-stack files rather than being passed as `--parameters` flags on every deploy command.

### 2. Phase 2 — Networking with Real Code

**Business Problem:** LearnOrbit's VPC needs public and private subnets across multiple Availability Zones — the exact kind of repetitive block that caused the two-day copy-paste incident. A loop, not a copy-pasted block, is the fix.

**Scene 2 — LearnOrbit Code Review | "This Is Just a Loop"**

> **Divya (You)** _Junior Platform Engineer_
>
> In the old CloudFormation template, each subnet was its own resource block, copy-pasted per AZ with the CIDR block and AZ name changed by hand. How do I avoid that here?

> **Yash** _Senior Platform Engineer_
>
> You loop over a list of Availability Zones and call a function that creates one subnet, passing in the AZ and a calculated CIDR block each time. It's genuinely just a `for` loop — no special Pulumi syntax, no macro system to learn. That's the whole point.

#### 2.1 VPC and a Loop Over Availability Zones

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const config = new pulumi.Config();
const azCount = config.getNumber("azCount") || 2;

const vpc = new aws.ec2.Vpc("learnorbit-vpc", {
  cidrBlock: "10.50.0.0/16",
  enableDnsSupport: true,
  enableDnsHostnames: true,
  tags: { Name: "learnorbit-vpc" },
});

const azs = pulumi.output(aws.getAvailabilityZones({ state: "available" }));

const publicSubnets: aws.ec2.Subnet[] = [];

for (let i = 0; i < azCount; i++) {
  const subnet = new aws.ec2.Subnet(`learnorbit-public-${i}`, {
    vpcId: vpc.id,
    cidrBlock: `10.50.${i}.0/24`,
    availabilityZone: azs.names[i],
    mapPublicIpOnLaunch: true,
    tags: { Name: `learnorbit-public-${i}` },
  });
  publicSubnets.push(subnet);
}
```

> **📖 A Real for Loop, Not a Templating Trick**
>
> `azCount` comes from Pulumi Config, so `production-delhi` can request 3 AZs while `staging` requests 2, by changing one config value — no code duplication either way. The `for` loop creates exactly `azCount` subnets, each with a CIDR block computed by string interpolation (`10.50.${i}.0/24`) rather than typed out by hand — this is precisely the class of mistake (a mistyped CIDR) that cost LearnOrbit two days in CloudFormation, now structurally impossible since the value is computed, not manually written per block. `vpc.id` and `azs.names[i]` are Pulumi **Outputs** — values that don't exist yet at the time this code runs (the VPC hasn't been created), but Pulumi resolves the dependency graph and fills them in during deployment.

#### 2.2 Internet Gateway and Route Table, Also Looped for Route Associations

```typescript
const igw = new aws.ec2.InternetGateway("learnorbit-igw", { vpcId: vpc.id });

const publicRouteTable = new aws.ec2.RouteTable("learnorbit-public-rt", {
  vpcId: vpc.id,
  routes: [{ cidrBlock: "0.0.0.0/0", gatewayId: igw.id }],
});

publicSubnets.forEach((subnet, i) => {
  new aws.ec2.RouteTableAssociation(`learnorbit-rta-${i}`, {
    subnetId: subnet.id,
    routeTableId: publicRouteTable.id,
  });
});
```

> **📖 forEach Over the Array Built in Phase 2.1**
>
> Because `publicSubnets` is a real TypeScript array populated by the earlier loop, associating a route table with every subnet is just `forEach` — no special multi-resource CloudFormation syntax needed. Each `RouteTableAssociation` gets a unique logical name (`learnorbit-rta-0`, `learnorbit-rta-1`, ...) derived from the loop index — Pulumi requires every resource to have a unique name within its type, and building that name programmatically means it always stays correct even if `azCount` changes.

**Pulumi Loops vs CloudFormation's Approach to Repetition**

- **Pulumi (real loops)** — a genuine `for` loop or `.forEach()`/`.map()` over an array, with full IDE autocomplete and type-checking on every iteration. Scales cleanly to any count driven by config.
- **CloudFormation (no native loops)** — repetition means either copy-pasting near-identical resource blocks (what caused LearnOrbit's incident), or reaching for CloudFormation Macros / the newer `Fn::ForEach` intrinsic (added in 2022), which works but is less flexible and less familiar than a language's native loop construct.

### 3. Phase 3 — Compute, Storage, and Config

**Business Problem:** LearnOrbit needs EC2 instances for its video-streaming API and an S3 bucket for recorded class uploads — sized differently per stack, and referencing a database password that must never be committed to Git in plaintext.

#### 3.1 EC2 Instances Driven by Config

```typescript
const instanceCount = config.getNumber("instanceCount") || 2;
const instanceType = config.get("instanceType") || "t3.micro";

const ami = aws.ec2.getAmiOutput({
  mostRecent: true,
  owners: ["amazon"],
  filters: [{ name: "name", values: ["amzn2-ami-hvm-*-x86_64-gp2"] }],
});

const apiInstances: aws.ec2.Instance[] = [];

for (let i = 0; i < instanceCount; i++) {
  const instance = new aws.ec2.Instance(`learnorbit-api-${i}`, {
    instanceType: instanceType,
    ami: ami.id,
    subnetId: publicSubnets[i % publicSubnets.length].id,
    tags: { Name: `learnorbit-streaming-api-${i}` },
  });
  apiInstances.push(instance);
}
```

> **📖 getAmiOutput and the Modulo Trick for Spreading Across AZs**
>
> `aws.ec2.getAmiOutput` is a **data source lookup**, not a resource — it queries AWS for the latest matching Amazon Linux 2 AMI at deployment time rather than hardcoding an AMI ID that goes stale as AWS releases updates. `publicSubnets[i % publicSubnets.length]` distributes instances round-robin across however many subnets exist — with `instanceCount: 4` and 2 subnets, instances land in subnet 0, 1, 0, 1 — spreading load across AZs automatically regardless of how `instanceCount` and `azCount` are separately configured per stack.

#### 3.2 S3 Bucket for Recorded Class Uploads

```typescript
const recordingsBucket = new aws.s3.BucketV2("learnorbit-recordings", {
  bucket: pulumi.interpolate`learnorbit-recordings-${pulumi.getStack()}`,
});

new aws.s3.BucketVersioningV2("learnorbit-recordings-versioning", {
  bucket: recordingsBucket.id,
  versioningConfiguration: { status: "Enabled" },
});

new aws.s3.BucketPublicAccessBlock("learnorbit-recordings-block-public", {
  bucket: recordingsBucket.id,
  blockPublicAcls: true,
  blockPublicPolicy: true,
  ignorePublicAcls: true,
  restrictPublicBuckets: true,
});
```

> **📖 pulumi.getStack() and Composed Resource Configuration**
>
> `pulumi.getStack()` returns the current stack's name (`production-delhi`, `staging`, etc.) at runtime, letting one bucket-naming expression produce `learnorbit-recordings-production-delhi` and `learnorbit-recordings-staging` from the same line of code across different stacks — no per-environment naming logic duplicated anywhere. Notice versioning and public-access-block are separate resources (`BucketVersioningV2`, `BucketPublicAccessBlock`) rather than properties on the bucket itself — this reflects the current AWS provider's design, which favors small composable resources over one large bucket resource with dozens of optional properties.

#### 3.3 Secrets — Never in Plaintext Config

```bash
pulumi config set --secret dbPassword "S3cur3-P@ssw0rd-2026" --stack production-delhi
```

```typescript
const dbPassword = config.requireSecret("dbPassword");

const database = new aws.rds.Instance("learnorbit-db", {
  engine: "postgres",
  instanceClass: "db.t3.medium",
  allocatedStorage: 20,
  username: "learnorbit_admin",
  password: dbPassword,
  skipFinalSnapshot: true,
});
```

> **📖 --secret Encrypts at Rest, requireSecret Tracks It Through the Program**
>
> `pulumi config set --secret` encrypts the value before writing it to `Pulumi.production-delhi.yaml` — the file safely contains an encrypted ciphertext blob, not the plaintext password, and is safe to commit to Git. `config.requireSecret("dbPassword")` returns a Pulumi **Secret**-wrapped Output — Pulumi tracks this taint through the entire program, so if this value is ever passed into another resource's property or accidentally logged, Pulumi automatically redacts it in CLI output and in the state file, printing `[secret]` instead of the real value. This is meaningfully stronger than a plain environment variable, which has no such tracking once it leaves its original context.

> **Quiz: A LearnOrbit engineer runs `pulumi config set dbPassword "hunter2"` without the `--secret` flag. What's the actual risk?**
> - Nothing — Pulumi encrypts all config values automatically regardless of flags
> - The password is stored in plaintext in the per-stack YAML file, visible to anyone with repo access, and won't be redacted in CLI output
> - The deployment will fail because passwords require the --secret flag
>
> > **Answer/explanation:** The second option. Without `--secret`, Pulumi treats the value as ordinary configuration — stored as plaintext in `Pulumi.<stack>.yaml`, visible in plain text to anyone who can read the file (including in Git history, permanently, even if fixed later), and displayed unredacted in `pulumi preview`/`pulumi up` output and in the state file. Pulumi does not fail the deployment or automatically detect that a key named "dbPassword" should be secret — the `--secret` flag (or `config.requireSecret` on the read side working together with it) is an opt-in behavior the engineer must explicitly choose. This is why LearnOrbit's platform team enforces a code-review checklist item specifically checking for `--secret` on any credential-like config key.

### 4. Phase 4 — Preview, Update, and Diffing

**Business Problem:** Meenal wants to resize the production-delhi API fleet from `t3.micro` to `t3.small` ahead of an exam-results-day traffic spike — but needs certainty about whether this happens in place or forces new instances (and new public IPs, which matter because the mobile app has some of them cached).

#### 4.1 Preview Before Touching Anything

```bash
pulumi config set instanceType t3.small --stack production-delhi

pulumi preview --stack production-delhi
```

```
     Type                    Name                      Plan
     pulumi:pulumi:Stack     learnorbit-infra-prod...
 ~   ├─ aws:ec2:Instance     learnorbit-api-0          update
 ~   ├─ aws:ec2:Instance     learnorbit-api-1          update
     └─ ...

Resources:
    ~ 2 to update
    6 unchanged
```

> **📖 Reading the Preview Diff**
>
> `pulumi preview` computes the difference between the current state and what the program now describes, without changing anything in AWS — this is Pulumi's direct equivalent of a CloudFormation change set or `terraform plan`. The `~` symbol means "update in place" (as opposed to `+` for create or `-` for delete, or `+-` for replacement). Here, changing `instanceType` on an EC2 instance is confirmed as an in-place update — the instance keeps its ID and, typically, its private IP, though a stop/start cycle for the resize does briefly interrupt the instance.

#### 4.2 Apply the Update

```bash
pulumi up --stack production-delhi
```

> **📖 pulumi up — Preview Plus Confirmation, In One Command**
>
> `pulumi up` runs the same preview internally, shows it interactively, and then prompts for confirmation before applying — in an automated CI/CD pipeline, `pulumi up --yes` skips the interactive prompt for non-interactive execution, but LearnOrbit's platform team requires the interactive confirmation step for any manual production deploy specifically so a human reads the diff before approving it, matching the discipline of reviewing a CloudFormation change set before executing it.

#### 4.3 A Change That Forces Replacement

```typescript
// Changing keyName forces a full replace, not an in-place update
const instance = new aws.ec2.Instance(`learnorbit-api-0`, {
  instanceType: instanceType,
  ami: ami.id,
  keyName: "learnorbit-prod-keypair-v2", // was: learnorbit-prod-keypair
  subnetId: publicSubnets[0].id,
});
```

```
 +-  aws:ec2:Instance   learnorbit-api-0    replace
```

> **📖 The +- Symbol Means Delete-and-Recreate**
>
> Changing `keyName` on an existing EC2 instance is a property AWS doesn't support updating in place, so Pulumi's provider marks it as forcing replacement — shown as `+-` in the preview. This creates a *new* instance (new ID, likely a new public IP) and deletes the old one, by default creating the new one first for resources that support it (`create before delete`), to minimize downtime. This is exactly the kind of surprise `pulumi preview` exists to catch before a production apply, the same discipline as reviewing a CloudFormation change set's `Replacement` field.

> **Key takeaways**
> - `pulumi preview` (or the preview inside `pulumi up`) must be reviewed before any production apply — it's the only reliable way to know whether a change updates in place or replaces a resource.
> - `~` means in-place update, `+-` means replace (delete and recreate), `+` means create, `-` means delete — learn these four symbols and every Pulumi diff becomes readable at a glance.
> - Always use `--secret` for credential-like config values — Pulumi does not infer secrecy from a key's name, and an unmarked secret is stored in plaintext and shown unredacted in CLI output.
> - Stacks are Pulumi's environment boundary — the same program, deployed with different config, produces independently tracked state per stack (dev, staging, production-delhi, production-mumbai).

### 5. Phase 5 — Reusable Component Resources

**Business Problem:** Even with loops, `index.ts` is growing into one large file repeating the same "networking + compute + storage for one region" pattern for both `production-delhi` and `production-mumbai`. LearnOrbit needs this packaged as a genuinely reusable unit — the direct fix for the original two-day copy-paste problem.

**Scene 3 — LearnOrbit Architecture Sync | "Package the Whole Region as One Thing"**

> **Meenal** _Staff Engineer_
>
> We're still repeating "VPC + subnets + instances + bucket" as a block of code per region, even if it's shorter now with loops. What we actually want is a `RegionalDeployment` component — call it once per region with a config object, get back a fully formed environment. That's a class, the same as you'd write in any other TypeScript codebase.

#### 5.1 Define a ComponentResource

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

interface RegionalDeploymentArgs {
  cidrBlock: string;
  instanceCount: number;
  instanceType: string;
}

export class RegionalDeployment extends pulumi.ComponentResource {
  public readonly vpcId: pulumi.Output<string>;
  public readonly instanceIds: pulumi.Output<string>[];

  constructor(name: string, args: RegionalDeploymentArgs, opts?: pulumi.ComponentResourceOptions) {
    super("learnorbit:infra:RegionalDeployment", name, {}, opts);

    const vpc = new aws.ec2.Vpc(`${name}-vpc`, {
      cidrBlock: args.cidrBlock,
      enableDnsSupport: true,
    }, { parent: this });

    const instances: aws.ec2.Instance[] = [];
    for (let i = 0; i < args.instanceCount; i++) {
      instances.push(new aws.ec2.Instance(`${name}-api-${i}`, {
        instanceType: args.instanceType,
        ami: "ami-0c2af51e265bd5e0e",
      }, { parent: this }));
    }

    this.vpcId = vpc.id;
    this.instanceIds = instances.map(i => i.id);
    this.registerOutputs({ vpcId: this.vpcId });
  }
}
```

> **📖 ComponentResource — A Real Class Wrapping Many Resources**
>
> `extends pulumi.ComponentResource` turns this into a genuine reusable unit — a class, exactly like any other TypeScript class, that happens to create cloud resources in its constructor. `{ parent: this }` on every child resource groups them logically under this component in Pulumi's resource tree — visible in `pulumi stack graph` and the Pulumi Cloud console as a clean hierarchy, instead of a flat list of dozens of unrelated-looking resources. `registerOutputs` finalizes which properties are exposed as this component's own Outputs.

#### 5.2 Instantiate the Component Per Region

```typescript
const delhiDeployment = new RegionalDeployment("learnorbit-delhi", {
  cidrBlock: "10.50.0.0/16",
  instanceCount: 4,
  instanceType: "t3.small",
});

const mumbaiDeployment = new RegionalDeployment("learnorbit-mumbai", {
  cidrBlock: "10.51.0.0/16",
  instanceCount: 2,
  instanceType: "t3.micro",
});

export const delhiVpcId = delhiDeployment.vpcId;
export const mumbaiVpcId = mumbaiDeployment.vpcId;
```

> **📖 This Is the Actual Fix for the Two-Day Incident**
>
> Adding LearnOrbit's fifth region is now three lines — one `new RegionalDeployment(...)` call with a new name, CIDR block, and instance count. The CIDR block is still a value a human types, but it's the *only* thing that changes, isolated to a single line instead of scattered across an 80-line copy-pasted YAML block — dramatically reducing the surface area for the kind of typo that caused the original incident. `export const delhiVpcId` at the top level of `index.ts` becomes this stack's Output, retrievable via `pulumi stack output delhiVpcId` or referenced by another stack via a StackReference (covered next).

#### 5.3 Referencing Outputs From Another Stack

```typescript
const networkingStack = new pulumi.StackReference("learnorbit/networking/production-delhi");
const sharedVpcId = networkingStack.getOutput("delhiVpcId");

const apiSecurityGroup = new aws.ec2.SecurityGroup("learnorbit-api-sg", {
  vpcId: sharedVpcId,
  ingress: [{ protocol: "tcp", fromPort: 443, toPort: 443, cidrBlocks: ["0.0.0.0/0"] }],
});
```

> **📖 StackReference — Pulumi's Cross-Stack Sharing**
>
> `StackReference` reads another stack's exported Outputs by its fully qualified name (`org/project/stack`) — this is Pulumi's equivalent of CloudFormation's cross-stack `Fn::ImportValue`/`Export` pattern. LearnOrbit uses this to keep a separate `networking` project (owned by the platform team) fully independent from an `application` project (owned by feature teams), with the application stacks referencing the shared VPC ID without needing permission to modify the networking stack itself.

### 6. Phase 6 — Testing and State Management

**Business Problem:** Meenal wants confidence that a change to the `RegionalDeployment` component (like accidentally hardcoding `instanceCount: 4` regardless of the argument) is caught in code review, before it ever reaches `pulumi preview` against a real stack.

#### 6.1 Unit Test the Component with Pulumi's Mocking Framework

```typescript
import * as pulumi from "@pulumi/pulumi";

pulumi.runtime.setMocks({
  newResource: (args: pulumi.runtime.MockResourceArgs): { id: string; state: any } => {
    return { id: `${args.name}-mock-id`, state: args.inputs };
  },
  call: (args: pulumi.runtime.MockCallArgs) => args.inputs,
});

import { RegionalDeployment } from "../regionalDeployment";

describe("RegionalDeployment", () => {
  it("creates the exact number of instances requested", async () => {
    const deployment = new RegionalDeployment("test-region", {
      cidrBlock: "10.99.0.0/16",
      instanceCount: 3,
      instanceType: "t3.micro",
    });

    const ids = await new Promise<string[]>((resolve) =>
      pulumi.all(deployment.instanceIds).apply(resolve)
    );

    expect(ids.length).toBe(3);
  });
});
```

> **📖 Mocks — Testing Infrastructure Code Without Deploying Anything**
>
> `pulumi.runtime.setMocks` replaces the real AWS provider with a fake one that returns synthetic IDs instantly, with zero real API calls and zero cost — this is what makes it possible to unit test infrastructure code the same way you'd test any other function, running in milliseconds as part of a normal CI pipeline. The test asserts `instanceCount: 3` in the arguments actually results in 3 entries in `deployment.instanceIds` — exactly the kind of "silently hardcoded instead of using the parameter" bug that's easy to introduce and easy to miss in review without an automated check like this.

#### 6.2 Where Pulumi State Actually Lives

```bash
# Default: Pulumi's own managed backend (Pulumi Cloud)
pulumi login

# Self-managed alternative: store state in an S3 bucket instead
pulumi login s3://learnorbit-pulumi-state
```

> **📖 State Backend — A Genuine Design Choice CloudFormation Doesn't Require**
>
> Every resource Pulumi creates is recorded in a **state file**, tracking the mapping between your code's logical resource names and real cloud resource IDs — this is what `pulumi preview` diffs against. By default this lives in Pulumi's managed Pulumi Cloud backend (free for individuals, handles encryption and locking automatically). `pulumi login s3://...` switches to a self-managed backend, storing the same state as a file in an S3 bucket LearnOrbit owns and controls directly. CloudFormation has no equivalent decision to make — AWS tracks stack state internally, invisible and non-configurable — which is simpler operationally but means you can't inspect or move that state the way Pulumi allows.

> **Quiz: Two engineers run `pulumi up` against the same stack at nearly the same time. What prevents them from corrupting the stack's state?**
> - Nothing — Pulumi assumes only one engineer ever deploys at a time
> - Pulumi's state backend applies a lock during an update; the second `pulumi up` is blocked until the first completes
> - AWS itself rejects the second set of API calls automatically
>
> > **Answer/explanation:** The second option. Both Pulumi Cloud and a self-managed S3 backend implement state locking — when `pulumi up` starts, it acquires a lock on the stack's state file, and a second concurrent `pulumi up` against the *same* stack is blocked (or fails with a clear "another update is in progress" error) until the first finishes and releases the lock. This is essential for LearnOrbit now that multiple engineers can deploy to the same stack — without locking, two simultaneous updates could both read the same "before" state, compute conflicting plans, and corrupt the recorded state so it no longer matches reality. AWS itself has no awareness of Pulumi's state file at all — this protection exists entirely at the Pulumi backend layer.

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Add a third region using only config:** Extend `index.ts` to read a list of regions from Pulumi Config (as a JSON array) and loop over `RegionalDeployment` instantiations, rather than hardcoding Delhi and Mumbai as two separate named variables.
2. **Add auto-tagging via transformations:** Use a Pulumi stack transformation (`pulumi.runtime.registerStackTransformation`) to automatically apply a `ManagedBy: pulumi` and `CostCenter: platform-eng` tag to every taggable resource in the program, without adding tags to each resource individually.
3. **Convert the RegionalDeployment component to also accept a VPC CIDR range and validate no overlap:** Add a TypeScript function that checks a new region's CIDR block against all previously deployed regions' CIDR blocks and throws a clear error at `pulumi preview` time if they overlap — turning LearnOrbit's original incident into something the code itself prevents.
4. **Write a policy pack with CrossGuard:** Use `@pulumi/policy` to write a Pulumi CrossGuard policy that fails any `pulumi up` attempting to create an S3 bucket without `BlockPublicAccess` fully enabled, enforced automatically before any resource is created.
5. **Import an existing hand-created resource:** Use `pulumi import` to bring an S3 bucket that was created manually in the AWS Console (outside of Pulumi) under Pulumi's management, without recreating or losing the bucket's existing data.

### Pulumi Project Complete 🎉

You have rebuilt LearnOrbit's infrastructure in Pulumi and TypeScript — networking and compute driven by real loops instead of copy-pasted blocks, secrets handled with proper encryption and redaction, `pulumi preview` catching in-place updates versus replacements before they touch production, a reusable `RegionalDeployment` component that turns adding a new region into a three-line change, and a unit test that verifies the component's behavior without deploying anything real.

> **Yash**
>
> "Divya, the two-day incident that started this whole rewrite — a mistyped CIDR block copy-pasted into a new region's block — literally cannot happen anymore. The CIDR is a config value passed into a component once. There's no second copy of that logic anywhere to drift out of sync."

> **Meenal**
>
> "The thing I didn't expect to value this much is the unit test. Infrastructure code almost never gets tested the way application code does, and it should — catching 'this component silently ignores its instanceCount argument' in a five-second test run in CI, instead of during a live `pulumi preview` against production, is a real quality-of-life change for how confidently we ship infra changes now."

> **Next: Comparing IaC Tools — When to Reach for CloudFormation, Terraform, or Pulumi**

> - Revisit LearnOrbit's original CloudFormation templates side-by-side with this Pulumi rewrite and catalogue exactly which problems each tool structurally prevents versus merely discourages
> - Explore Pulumi's multi-cloud providers to see how the same component pattern would extend to a hypothetical GCP or Azure region
> - Set up a CI/CD pipeline that runs `pulumi preview` on every pull request and posts the diff as a PR comment before merge
