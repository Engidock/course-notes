# 📦 Nexus Repository Project Mastery

> **Hey Fresher — Read This First!**
>
> Nexus Repository is a private, self-hosted artifact repository manager — it stores the things your build produces (JARs, npm packages, Docker images) and caches the things your build downloads from the public internet (Maven Central, npmjs, Docker Hub), all behind one URL your build tools talk to. Instead of every developer and CI runner hitting the public internet directly for every dependency, and instead of having nowhere to publish your own internal packages, Nexus becomes the single source of truth both directions.
>
> **TuneWave Streaming** runs a music and podcast streaming service used across South Asia, with a Java-based recommendation engine, a Node.js transcoding-worker fleet, and a growing set of internal shared libraries neither team can currently publish anywhere except a shared Google Drive folder. You're joining as a Build Infrastructure Engineer. Your first task: stand up Nexus so TuneWave's own artifacts get published somewhere real, public dependency downloads get cached (so a Maven Central outage doesn't stop every build in the company), and Docker images for the transcoding workers have a private, access-controlled home instead of sitting in a public Docker Hub repo with sensitive audio-codec licensing code baked in.

#### What You Will Learn and Build in This Project

You will install Nexus Repository and understand blob stores, create hosted repositories for TuneWave's own Maven and npm artifacts, set up proxy repositories that cache Maven Central and the npm registry, combine both into group repositories that build tools point at with a single URL, add a private Docker registry for transcoding-worker images, and configure security roles and cleanup policies to keep storage and access under control.

Nexus Repository Manager, Blob Stores, Hosted Repositories, Proxy Repositories, Group Repositories, Maven Repository Format, npm Repository Format, Docker Registry in Nexus, Repository Roles and Privileges, Cleanup Policies, Content Selectors

> **📦 Phase 1 — Installing Nexus and Understanding Blob Stores**
>
> Run Nexus Repository via Docker, complete initial setup, and understand where and how artifact data is actually stored.

> **📦 Phase 2 — Hosted Repositories for TuneWave's Own Artifacts**
>
> Create hosted Maven and npm repositories and publish TuneWave's shared libraries into them from real build tooling.

> **📦 Phase 3 — Proxy Repositories for Public Dependencies**
>
> Cache Maven Central and the public npm registry through Nexus so builds don't depend on direct internet access, and so a public outage doesn't halt TuneWave's CI.

> **📦 Phase 4 — Group Repositories: One URL for Everything**
>
> Combine hosted and proxy repositories into a single group repository each build tool points at, resolving both internal and public dependencies transparently.

> **📦 Phase 5 — A Private Docker Registry for Transcoding Workers**
>
> Stand up hosted and proxy Docker repositories in Nexus so transcoding-worker images stay private and access-controlled.

> **📦 Phase 6 — Security Roles and Cleanup Policies**
>
> Lock down who can publish to which repository, and configure cleanup policies so old snapshot and dev-tag artifacts don't quietly fill the disk.

**Scene 1 — TuneWave Streaming, Hyderabad | The Google Drive Dependency Problem**

> **Roshan** _Junior Build Infrastructure Engineer_
>
> I asked how the recommendation-engine team shares their `tunewave-audio-fingerprint` library with the transcoding team, and the answer was "download the JAR from this Drive folder, there might be a newer version, ask in Slack." That's not a dependency management strategy.

> **Priya** _Senior DevOps Engineer_
>
> It's the classic symptom of not having a private artifact repository. Every internal library ends up shared ad hoc, with no versioning discipline and no way for a build to just declare a dependency and get it reliably.

> **Amit** _Platform Architect_
>
> And it's not only internal artifacts. Last month Maven Central had a regional slowdown and every single one of our Java builds queued for twenty minutes. If we're caching dependencies through Nexus instead of hitting the public internet directly on every build, that entire class of problem goes away — plus our Docker images for the transcoding workers, which embed licensed audio codec logic, have no business sitting in a public registry at all.

> **Roshan**
>
> So we need hosted repos for our own stuff, proxy repos caching the public registries, one URL combining both per build tool, and a private Docker registry. Four repository types, one Nexus install.

> **Priya**
>
> Exactly — and access control and cleanup policies so it doesn't become its own mess in six months.

### 1. Phase 1 — Installing Nexus and Understanding Blob Stores

**Business Problem:** TuneWave has no artifact repository manager running anywhere yet. Before creating a single repository, Nexus itself needs to be running with a properly configured storage backend that can grow with the company's artifact volume.

#### 1.1 Running Nexus via Docker

```bash
docker volume create nexus-data

docker run -d \
  --name tunewave-nexus \
  -p 8081:8081 \
  -v nexus-data:/nexus-data \
  sonatype/nexus3:3.70.1

# Wait for startup, then retrieve the initial admin password
docker exec tunewave-nexus \
  cat /nexus-data/admin.password
```

> **📖 What each piece is for**
> `docker volume create nexus-data` creates a named, persistent Docker volume — without it, Nexus's data (repositories, blob storage, configuration) would live inside the container's writable layer and vanish the moment the container is removed. `-p 8081:8081` exposes Nexus's default web UI and API port. The `admin.password` file is auto-generated on first boot and required to complete the setup wizard at `http://localhost:8081`, after which Nexus prompts you to set a real admin password — this initial file is deleted once setup completes.

#### 1.2 Understanding Blob Stores

```
Nexus Repository storage model:

  Repository (e.g. tunewave-maven-hosted)
        │
        │ writes artifact content to
        ▼
  Blob Store (e.g. "default" or "tunewave-releases-blobs")
        │
        │ physically stored as
        ▼
  Files on disk at /nexus-data/blobs/<blob-store-name>/
```

> **📖 Why blob stores are a separate concept from repositories**
> A **repository** is the logical, named thing your build tool talks to (an npm registry URL, a Maven repository ID) — it defines format (Maven, npm, Docker), type (hosted, proxy, group), and policy. A **blob store** is where the actual bytes physically live on disk (or in S3-compatible storage). Multiple repositories can share one blob store, or each can have its own — TuneWave gives the Docker registry its own dedicated blob store (`tunewave-docker-blobs`) specifically so its typically much larger image layers don't compete for the same disk quota as Maven JARs, and so a cleanup or capacity issue on one doesn't affect the other.

```bash
# Creating a dedicated blob store for Docker images via the REST API
curl -u admin:${NEXUS_ADMIN_PW} -X POST \
  "http://localhost:8081/service/rest/v1/blobstores/file" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "tunewave-docker-blobs",
        "path": "/nexus-data/blobs/tunewave-docker-blobs",
        "softQuota": {
          "type": "spaceRemainingQuota",
          "limit": 10737418240
        }
      }'
```

> **`softQuota`** — configures Nexus to stop accepting new writes into this blob store once free space drops below the configured limit (here, 10 GiB remaining, expressed in bytes), rather than filling the disk completely and taking the whole Nexus instance down. This is exactly the kind of guardrail Amit wants around the Docker blob store, since container image layers can be large and numerous.

> **Key takeaways**
> - A repository is the logical, named endpoint your tooling talks to; a blob store is the physical storage backing it, and the two are configured independently.
> - Multiple repositories can share a blob store, or each can have a dedicated one — dedicating one per repository format isolates capacity issues.
> - `softQuota` prevents a single repository's growth from filling disk and taking down the entire Nexus instance.

### 2. Phase 2 — Hosted Repositories for TuneWave's Own Artifacts

**Business Problem:** `tunewave-audio-fingerprint` and TuneWave's other internal libraries need an actual home other teams' builds can declare as a normal dependency, replacing the Drive-folder workflow entirely.

#### 2.1 Creating a Hosted Maven Repository

```bash
# Via UI: Repositories → Create repository → maven2 (hosted)
# Settings used:
#   Name: tunewave-maven-releases
#   Version policy: Release
#   Deployment policy: Disable redeploy
```

```xml
<!-- tunewave-audio-fingerprint/pom.xml -->
<distributionManagement>
  <repository>
    <id>tunewave-maven-releases</id>
    <url>http://nexus.tunewave.internal:8081/repository/tunewave-maven-releases/</url>
  </repository>
</distributionManagement>
```

> **📖 Why "Disable redeploy" matters**
> A **hosted** repository is where Nexus itself is the source of truth — nothing is proxied, artifacts only exist because someone published them here directly. Setting **Deployment policy: Disable redeploy** on the releases repository means once `tunewave-audio-fingerprint:2.3.0` is published, Nexus will reject any attempt to publish a different JAR under that same coordinate — protecting every downstream consumer from a silent, un-versioned change to an artifact they've already built against. **Version policy: Release** further restricts this repository to non-SNAPSHOT versions only; a separate `tunewave-maven-snapshots` hosted repository, configured with policy **Snapshot**, is where in-development builds go and where overwriting is expected and allowed.

#### 2.2 Publishing from Maven

```xml
<!-- ~/.m2/settings.xml -->
<servers>
  <server>
    <id>tunewave-maven-releases</id>
    <username>${env.NEXUS_USER}</username>
    <password>${env.NEXUS_PASSWORD}</password>
  </server>
</servers>
```

```bash
mvn clean deploy
```

```
[INFO] Uploading to tunewave-maven-releases: http://nexus.tunewave.internal:8081/repository/tunewave-maven-releases/com/tunewave/tunewave-audio-fingerprint/2.3.0/tunewave-audio-fingerprint-2.3.0.jar
[INFO] BUILD SUCCESS
```

#### 2.3 Creating a Hosted npm Repository

```bash
# Via UI: Repositories → Create repository → npm (hosted)
# Name: tunewave-npm-internal
```

```bash
# .npmrc in the transcoding-worker repo
registry=http://nexus.tunewave.internal:8081/repository/tunewave-npm-group/
//nexus.tunewave.internal:8081/repository/tunewave-npm-internal/:_authToken=${NEXUS_NPM_TOKEN}
```

```json
// package.json for the shared internal package
{
  "name": "@tunewave/transcode-utils",
  "version": "1.2.0",
  "publishConfig": {
    "registry": "http://nexus.tunewave.internal:8081/repository/tunewave-npm-internal/"
  }
}
```

```bash
npm publish
```

> **📖 Scoped packages and publishConfig**
> The `@tunewave/` scope on `transcode-utils` isn't just naming convention — npm uses the scope to help route publish and install requests, and it visually distinguishes TuneWave's internal packages from public ones in every `package.json`. `publishConfig.registry` overrides the default registry specifically for the `npm publish` command, pointing it at the hosted repository, while the `.npmrc` registry line (pointed at a *group* repository, built in Phase 4) is what regular `npm install` uses to resolve dependencies — publish and install intentionally go through different repository types.

> **Key takeaways**
> - Hosted repositories are where Nexus is the authoritative source — nothing is proxied or cached, only explicitly published content exists there.
> - "Disable redeploy" on a releases repository enforces artifact immutability, protecting downstream consumers from silent version drift.
> - Separate hosted repositories for releases vs. snapshots (Maven) let you apply different overwrite policies to each.

**Quiz: Why does TuneWave configure the Maven hosted-releases repository with "Disable redeploy" instead of allowing artifacts to be overwritten?**
- Disabling redeploy makes uploads faster
- It guarantees that once a version like 2.3.0 is published, every consumer building against that exact coordinate always gets the same bytes, preventing silent, unversioned changes
- Nexus requires redeploy to be disabled for Maven repositories by default
- It reduces the number of blob stores needed

> **Answer/explanation:** The correct answer is the second option. Once other teams' builds depend on `tunewave-audio-fingerprint:2.3.0`, they're trusting that coordinate to always resolve to the same artifact. If redeploy were allowed, someone could silently republish different bytes under the same version, breaking reproducible builds and potentially introducing untested code without any version bump signaling the change. It isn't about upload speed, isn't Nexus's universal default (snapshot repositories intentionally allow overwriting), and has no relationship to blob store count.

### 3. Phase 3 — Proxy Repositories for Public Dependencies

**Business Problem:** Every TuneWave build currently hits Maven Central and the public npm registry directly. When Maven Central had a slow region last month, every Java build in the company queued for twenty minutes simultaneously — and there's no caching, so the same popular dependency gets re-downloaded repeatedly across dozens of CI runs a day.

#### 3.1 Creating a Maven Central Proxy

```bash
# Via UI: Repositories → Create repository → maven2 (proxy)
# Name: tunewave-maven-central-proxy
# Remote storage: https://repo1.maven.org/maven2/
```

> **📖 How a proxy repository behaves**
> The first time any TuneWave build requests, say, `com.fasterxml.jackson.core:jackson-databind:2.15.2` through this proxy, Nexus fetches it from `repo1.maven.org`, stores a copy in its own blob store, and serves it back. Every subsequent request for that exact artifact — from any developer or CI runner in the company — is served straight from Nexus's local cache, never touching the public internet again. If Maven Central is slow or briefly unreachable, cached artifacts are unaffected; only genuinely new, never-before-requested artifacts are impacted.

#### 3.2 Creating an npm Registry Proxy

```bash
# Via UI: Repositories → Create repository → npm (proxy)
# Name: tunewave-npm-proxy
# Remote storage: https://registry.npmjs.org
```

#### 3.3 Configuring Cache Behavior

```bash
# Metadata max age (how long before Nexus re-checks upstream for a newer version
# of mutable metadata like package.json's "latest" tag)
curl -u admin:${NEXUS_ADMIN_PW} -X PUT \
  "http://localhost:8081/service/rest/v1/repositories/npm/proxy/tunewave-npm-proxy" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "tunewave-npm-proxy",
        "proxy": {
          "remoteUrl": "https://registry.npmjs.org",
          "contentMaxAge": 1440,
          "metadataMaxAge": 1440
        },
        "negativeCache": { "enabled": true, "timeToLive": 1440 },
        "httpClient": { "blocked": false, "autoBlock": true }
      }'
```

> **📖 The caching knobs that matter**
> `contentMaxAge` and `metadataMaxAge` (in minutes) control how long Nexus trusts its cached copy of an immutable artifact vs. mutable metadata before re-validating against the upstream registry — a specific version's tarball never changes, but what "latest" points to does, so metadata needs to be revalidated more readily than content in principle, even though both are set to 1440 (24 hours) here for simplicity. `negativeCache` remembers "this package/version doesn't exist upstream" responses for the configured TTL, so a typo'd package name doesn't hammer the public registry with repeated 404 lookups. `autoBlock: true` means if the upstream registry starts returning consistent errors, Nexus temporarily stops trying to reach it and serves only what's cached, rather than letting every build hang waiting on a struggling upstream.

**Proxy Repository vs. Hosted Repository**

- **Proxy** — content originates from an external source (Maven Central, npmjs, Docker Hub); Nexus caches what's requested, and the repository has no content of its own until something is fetched through it.
- **Hosted** — content originates from explicit publishes into Nexus itself; nothing is fetched from elsewhere, and Nexus is the authoritative, permanent source for whatever's published there.

> **Key takeaways**
> - Proxy repositories cache-on-first-request; subsequent requests for the same artifact are served locally, insulating builds from upstream slowness or outages.
> - `negativeCache` and `autoBlock` protect both Nexus and the upstream registry from repeated failed lookups.
> - Immutable artifact content and mutable metadata (like npm's `latest` tag) can be configured with different cache lifetimes.

### 4. Phase 4 — Group Repositories: One URL for Everything

**Business Problem:** Right now, developers would need to configure two separate repository URLs per build tool — one for TuneWave's own packages, one for the public proxy — and remember which is which. That's exactly the kind of manual, error-prone config Priya wants to eliminate.

#### 4.1 Creating a Maven Group Repository

```bash
# Via UI: Repositories → Create repository → maven2 (group)
# Name: tunewave-maven-group
# Member repositories, in order:
#   1. tunewave-maven-releases
#   2. tunewave-maven-snapshots
#   3. tunewave-maven-central-proxy
```

> **📖 Why member order matters**
> A **group** repository presents multiple repositories through a single URL, and Nexus checks member repositories in the listed order, returning the first match found. Listing `tunewave-maven-releases` before the Maven Central proxy means if TuneWave ever published an artifact under a `groupId:artifactId:version` that happens to collide with something on Central (unlikely given `com.tunewave` namespacing, but a real safety property), the internal one wins. It also means internal lookups don't pay the (tiny) overhead of checking the proxy first.

```xml
<!-- ~/.m2/settings.xml, used company-wide -->
<mirrors>
  <mirror>
    <id>tunewave-maven-group</id>
    <mirrorOf>*</mirrorOf>
    <url>http://nexus.tunewave.internal:8081/repository/tunewave-maven-group/</url>
  </mirror>
</mirrors>
```

> **`<mirrorOf>*</mirrorOf>`** — this tells Maven to route absolutely every repository request, regardless of what's declared in any project's `pom.xml`, through the single group URL. This is the standard company-wide Nexus setup: individual `pom.xml` files never need to know Nexus exists at all, they declare dependencies normally, and the mirror config transparently redirects every resolution through the group.

#### 4.2 Creating an npm Group Repository

```bash
# Via UI: Repositories → Create repository → npm (group)
# Name: tunewave-npm-group
# Member repositories: tunewave-npm-internal, tunewave-npm-proxy
```

```bash
# .npmrc — the registry line used earlier in Phase 2 now makes full sense
registry=http://nexus.tunewave.internal:8081/repository/tunewave-npm-group/
```

#### 4.3 Verifying Resolution Through the Group

```bash
# Resolves the internal @tunewave/transcode-utils AND public express in one install
npm install

added 143 packages in 8s
```

```bash
mvn dependency:tree | grep -E "tunewave|jackson"
[INFO] +- com.tunewave:tunewave-audio-fingerprint:jar:2.3.0:compile
[INFO] +- com.fasterxml.jackson.core:jackson-databind:jar:2.15.2:compile
```

> Both lines resolved through the exact same `tunewave-maven-group` URL — one artifact came from the hosted releases repository, the other from the Maven Central proxy cache, and neither Maven's output nor the developer running the build needs to know or care which.

> **Key takeaways**
> - Group repositories expose multiple repositories (hosted and proxy, mixed) through one URL, checked in a configured member order.
> - Company-wide build tool configuration (`mirrorOf: *` in Maven, a single `.npmrc` registry) can route every request through the group, so individual projects need zero Nexus-specific configuration.
> - A single `npm install` or `mvn dependency:tree` can transparently pull some artifacts from the internal hosted repo and others from the public proxy cache in the same run.

### 5. Phase 5 — A Private Docker Registry for Transcoding Workers

**Business Problem:** TuneWave's transcoding-worker images embed licensed audio codec integration code that cannot legally sit in a public Docker Hub repository, and the base images the team builds on top of (`node:20-alpine`, `ffmpeg`-bundled images) should also be cached rather than pulled fresh from Docker Hub on every CI run.

**Scene 2 — Security Review**

> **Amit** _Platform Architect_
>
> Before we create the Docker hosted repository — what port strategy are we using? Docker's registry API needs its own port per repository type in Nexus, it can't share the 8081 web UI port the way Maven and npm repositories do.

> **Roshan** _Junior Build Infrastructure Engineer_
>
> Right, so hosted goes on 8082, proxy on 8083, and then Docker clients need `insecure-registries` configured for local testing since we don't have TLS in front of Nexus yet — that has to change before this goes anywhere near production, though.

#### 5.1 Creating a Hosted Docker Repository

```bash
# Via UI: Repositories → Create repository → docker (hosted)
# Name: tunewave-docker-hosted
# HTTP port: 8082
# Blob store: tunewave-docker-blobs (created in Phase 1)
```

```bash
# Docker daemon config for local testing (production uses real TLS)
# /etc/docker/daemon.json
{
  "insecure-registries": ["nexus.tunewave.internal:8082"]
}
```

```bash
docker tag tunewave-transcode-worker:1.7.0 \
  nexus.tunewave.internal:8082/tunewave-transcode-worker:1.7.0

docker login nexus.tunewave.internal:8082

docker push nexus.tunewave.internal:8082/tunewave-transcode-worker:1.7.0
```

> **📖 Why Docker gets dedicated ports**
> Unlike Maven and npm, which are HTTP registries that Nexus can multiplex under one port via URL path (`/repository/<name>/`), the Docker Registry API v2 protocol expects to own the connecting port directly. Each Docker repository (hosted, proxy, group) in Nexus is assigned its own HTTP or HTTPS port at creation time, which is why `tunewave-docker-hosted` binds 8082 while the proxy in the next step will bind a different port entirely.

#### 5.2 Creating a Docker Hub Proxy

```bash
# Via UI: Repositories → Create repository → docker (proxy)
# Name: tunewave-dockerhub-proxy
# HTTP port: 8083
# Remote storage: https://registry-1.docker.io
```

```dockerfile
# transcoding-worker/Dockerfile — base image pulled through the proxy
FROM nexus.tunewave.internal:8083/library/node:20-alpine
```

> **`library/node`** — Docker Hub's official images live under the implicit `library/` namespace; when proxying Docker Hub through Nexus, that namespace has to be referenced explicitly in the image path. Every subsequent CI build pulling `node:20-alpine` through this proxy hits Nexus's cache after the first pull, meaningfully speeding up build times and removing a dependency on Docker Hub's public rate limits, which anonymous or low-tier pulls are subject to.

#### 5.3 A Docker Group Repository

```bash
# Via UI: Repositories → Create repository → docker (group)
# Name: tunewave-docker-group
# HTTP port: 8084
# Member repositories: tunewave-docker-hosted, tunewave-dockerhub-proxy
```

```bash
docker pull nexus.tunewave.internal:8084/tunewave-transcode-worker:1.7.0
docker pull nexus.tunewave.internal:8084/library/ffmpeg:6.1
```

**Quiz: Why does Nexus require a separate HTTP port for each Docker repository, while Maven and npm repositories can all share port 8081?**
- Docker images are always larger, so they need more bandwidth
- The Docker Registry API v2 protocol expects to own the port it's served on directly, unlike Maven/npm which are addressed by URL path under one shared HTTP port
- It's a licensing restriction from Docker Inc.
- Nexus only supports one Docker repository per installation

> **Answer/explanation:** The correct answer is the second option. Maven and npm clients talk plain HTTP and are routed by URL path (`/repository/tunewave-maven-group/...`), so Nexus can serve many such repositories from one port. The Docker Registry API v2 protocol, by contrast, expects the registry to be addressable directly at a host:port with no differentiating path prefix for repository identity in the way Maven/npm use, so Nexus assigns each Docker repository (hosted, proxy, or group) its own dedicated port. This has nothing to do with image size, isn't a Docker Inc. licensing rule, and Nexus supports many Docker repositories per installation, each just needing its own port.

### 6. Phase 6 — Security Roles and Cleanup Policies

**Business Problem:** Right now every developer with Nexus credentials can publish to every repository, including production release repositories — an intern could accidentally overwrite a shared library used across the company. And nobody has looked at disk usage since setup; old SNAPSHOT builds and untagged Docker layers are almost certainly accumulating.

#### 6.1 Creating Scoped Roles

```bash
# Via UI: Security → Roles → Create role
# Role: tunewave-maven-publisher
# Privileges: nx-repository-view-maven2-tunewave-maven-releases-add
#             nx-repository-view-maven2-tunewave-maven-releases-edit
#             nx-repository-view-maven2-tunewave-maven-snapshots-*

# Role: tunewave-docker-publisher
# Privileges: nx-repository-view-docker-tunewave-docker-hosted-*
```

> **📖 Privilege naming and scoping**
> Nexus privileges are generated per repository, per action (`browse`, `read`, `add`, `edit`, `delete`), following a `nx-repository-view-<format>-<repository-name>-<action>` pattern. Building a role from only the specific privileges needed — publish rights to `tunewave-maven-releases` and `tunewave-maven-snapshots`, nothing else — means a user assigned `tunewave-maven-publisher` can push a new library version but cannot touch the Docker registry, delete another team's artifacts, or modify Nexus's own security configuration. CI service accounts get their own narrowly scoped roles, separate from human developer accounts, following the same principle.

#### 6.2 Assigning Roles to Users and CI Service Accounts

```bash
curl -u admin:${NEXUS_ADMIN_PW} -X POST \
  "http://localhost:8081/service/rest/v1/security/users" \
  -H "Content-Type: application/json" \
  -d '{
        "userId": "ci-recommendation-engine",
        "firstName": "CI",
        "lastName": "RecommendationEngine",
        "emailAddress": "ci-bot@tunewave.internal",
        "password": "'"${CI_SERVICE_PASSWORD}"'",
        "status": "active",
        "roles": ["tunewave-maven-publisher"]
      }'
```

> A dedicated `ci-recommendation-engine` service account, scoped to only `tunewave-maven-publisher`, is what actually runs `mvn deploy` from the CI pipeline — never a personal developer credential. If this account's credentials ever leak, the blast radius is "can publish Maven artifacts," not "can do anything any admin can do."

#### 6.3 Cleanup Policies

```bash
# Via UI: Repository → Cleanup Policies → Create policy
# Name: tunewave-snapshot-cleanup
# Format: maven2
# Criteria: Component age → 30 days (last updated)
#           Component published for → snapshot components only
```

```bash
# Applied to the repository, and run on a schedule via a Nexus Task
# Admin → System → Tasks → Create task → "Cleanup service"
# Task name: nightly-snapshot-cleanup
# Repositories: tunewave-maven-snapshots
# Cleanup policies: tunewave-snapshot-cleanup
# Schedule: Daily at 02:00
```

> **📖 How cleanup actually runs**
> A **cleanup policy** just defines the *criteria* (older than 30 days, snapshot components, etc.) — it does nothing by itself until attached to a repository and executed by a scheduled **Task**. TuneWave's `nightly-snapshot-cleanup` task runs the policy against `tunewave-maven-snapshots` every night at 2 a.m., permanently deleting snapshot builds older than 30 days. A near-identical policy and task pair, scoped to `tunewave-docker-hosted` with criteria on untagged manifests, keeps orphaned Docker layers (left behind when a worker image gets rebuilt and retagged without the old tag ever being pulled again) from silently consuming the dedicated Docker blob store's quota.

**Hosted-Only Cleanup vs. Proxy Cache Cleanup**

- **Hosted repository cleanup policies** — permanently delete components that will never come back, since hosted repos are the only source; TuneWave applies age-based deletion here deliberately and conservatively (30 days for snapshots only, never for releases).
- **Proxy repository cache cleanup** — safe to be more aggressive, since anything deleted from a proxy's cache is simply re-fetched from upstream on the next request; Nexus's built-in blob store compaction and the proxy's own cache eviction handle most of this automatically.

> **Key takeaways**
> - Nexus privileges are generated per repository and action; scoped roles built from only necessary privileges limit blast radius if credentials leak.
> - CI/service accounts should get their own narrowly-scoped roles, never reuse a personal or admin account.
> - A cleanup policy defines criteria; a scheduled Task actually executes it against a repository — both pieces are required for cleanup to happen.
> - Hosted repository cleanup should be conservative (it's permanent, irreversible loss); proxy cache cleanup can be aggressive since the source of truth is still upstream.

##### 🏋️ Hands-On Exercises — Extend the Project

1. Create a `tunewave-npm-scoped-cleanup` policy that deletes any pre-release `@tunewave/*` package version (matching a `-beta` or `-rc` suffix) older than 14 days, and schedule a task to run it weekly.
2. Add a content selector restricting the `tunewave-maven-publisher` role so it can only publish artifacts under the `com.tunewave.*` groupId, and verify a publish attempt with a different groupId is rejected.
3. Set up a second Docker hosted repository, `tunewave-docker-hosted-internal-tools`, isolated from `tunewave-docker-hosted`, specifically for internal tooling images that should never be pulled by production transcoding workers, and add it to a separate group repository.
4. Configure Nexus's built-in email notifications (Administration → System → Email) so the platform team gets alerted when any blob store crosses 80% of its configured soft quota.
5. Write a small script using the Nexus REST API (`/service/rest/v1/search`) that lists every Maven artifact published by a non-CI user account in the last 7 days, as a lightweight audit of who is publishing manually versus through automated pipelines.

### Nexus Repository Project Complete 🎉

You have stood up a complete private artifact management layer for TuneWave: hosted Maven and npm repositories replacing the Google Drive workflow, proxy repositories caching Maven Central, the public npm registry, and Docker Hub so public outages stop affecting local builds, group repositories giving every build tool one URL to resolve both internal and public dependencies, a dedicated private Docker registry protecting licensed transcoding code, and scoped security roles plus scheduled cleanup policies keeping both access and disk usage under control.

> **Roshan** _Junior Build Infrastructure Engineer_
>
> `tunewave-audio-fingerprint` has a real home now. The transcoding team adds one line to their `pom.xml`, and they're on whatever version we've released — no more "check Slack for the latest JAR."

> **Priya** _Senior DevOps Engineer_
>
> And when Maven Central had that regional issue again last week, nobody on the team even noticed — every dependency we'd already resolved once was sitting in the proxy cache.

> **Amit** _Platform Architect_
>
> The Docker registry closed a real compliance gap. Our codec-licensing code was never supposed to be reachable outside TuneWave's network, and now it structurally isn't — it lives in a hosted repository with role-scoped publish access, full stop.

> **Next: Wiring Nexus into the delivery pipeline**
>
> - CI pipeline integration (Jenkins or GitHub Actions) that authenticates as scoped service accounts and runs `mvn deploy` / `docker push` automatically on every merge to main.
> - Nexus Repository Firewall or IQ Server for automated vulnerability and license scanning of every proxied dependency before it reaches a build.
> - Migrating the blob store to S3-compatible object storage for durability and horizontal scaling as TuneWave's artifact volume grows.
