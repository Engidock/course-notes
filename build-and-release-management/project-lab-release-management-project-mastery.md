# 📦 Release Management Project Mastery

### The ReleaseFlow Project — What We're Building

**ReleaseFlow** is a SaaS project management platform. Teams collaborate, track issues, manage sprints. Every 2 weeks: a new release with bug fixes, features, performance improvements.

**The Challenge:** Without release management, chaos emerges. Developers push code whenever they want. What gets deployed? Which version is live? What changed between v1.3.2 and v1.3.3? A bug appears in production — was it in v1.3.3? Can we rollback? No process = no control.

**The Solution:** Release management. Version code with semantic versioning (1.2.3). Automate builds with CI/CD (GitHub Actions). Tag releases in Git. Create release notes documenting changes. Deploy with rollback capability. Monitor each release. If a release breaks production, rollback in seconds to the previous version.

**📍 Scene: ReleaseFlow's Deployment Nightmare**

> **Amit (Release Lead)**
> 
> "Last month, we pushed a deployment at 3 PM on Friday. Someone uploaded a file to production, but we don't know who, when, or what changed. By 6 PM, users reported a critical bug. We tried rolling back, but which version? Git has 200 commits since the last release. We don't have a clean release boundary. We ended up manually reverting code changes while the CEO watches the user count drop."

> **Priya (DevOps Lead)**
> 
> "That's why we implement release management. We'll use semantic versioning — 1.2.3 means major.minor.patch. A minor release gets new features. A patch release gets only bug fixes. We tag every release in Git. Automated builds create artifacts. Every release is documented with a changelog. And we can rollback any release in 30 seconds."

> **Amit**
> 
> "How do we automate all this? Deploy to 5 servers? Monitor for issues? Notify the team?"

> **Priya**
> 
> "CI/CD pipelines. GitHub Actions. Every commit to main branch triggers an automated build, tests, and deployment. Blue-green deployments for zero downtime. Automated rollback if health checks fail. This is enterprise release management."

### 1. Semantic Versioning — The Foundation of Release Management

#### Understanding Semantic Versioning

#### Versioning Your Application

```
# package.json (Node.js / Frontend)
{
  "name": "releaseflow",
  "version": "1.2.3",
  "description": "Project management SaaS"
}

# build.gradle (Java)
version = '1.2.3'
group = 'com.releaseflow'
# setup.py (Python)
setup(
  name='releaseflow',
  version='1.2.3'
)
```

> **Single source of truth:** Define version in one place (package.json, build.gradle, setup.py). CI/CD reads from here.
**Version propagates:** When you bump version in package.json to 1.2.4, the build automatically uses 1.2.4 for the Docker image, release tag, artifact name.
**Consistency:** Application, Docker image, Git tag, release notes — all show version 1.2.4. No confusion.

### 2. Git Tags & Branching Strategy — Release Boundaries

#### Git Flow Branching Model

```
    Git Flow Release Model

        main (production)
        │
        ├─ v1.0.0 ── v1.1.0 ── v1.2.0 ── v1.2.1 (release tags)
        │
        release/1.2.0 (hotfix branch)
        │
        develop (staging)
        │
        ├─ feature/user-auth (feature branch)
        ├─ feature/payment-integration
        └─ feature/analytics
        
        Flow:
        1. Create feature branch from develop
        2. Developers commit to feature branches
        3. PR review, merge to develop
        4. Develop is tested on staging
        5. Create release/1.2.0 branch from develop
        6. Final testing, version bump, merge to main
        7. Tag main as v1.2.0
        8. If hotfix needed: create hotfix/1.2.1, merge to main, tag v1.2.1
        
```

#### Creating Release Tags in Git

```
$ git checkout develop
$ git pull origin develop

# Bump version in package.json from 1.2.2 to 1.2.3
$ nano package.json
$ git add package.json
$ git commit -m "Bump version to 1.2.3"
# Create release branch
$ git checkout -b release/1.2.3

# Run final tests, update CHANGELOG
$ npm test

# Merge release to main
$ git checkout main
$ git merge --no-ff release/1.2.3
$ git tag -a v1.2.3 -m "Release version 1.2.3"
$ git push origin main --tags

# Merge back to develop
$ git checkout develop
$ git merge --no-ff release/1.2.3
$ git push origin develop
```

> **release/1.2.3 branch:** Isolated space for final release prep. No new features. Only bug fixes and version bumps.
**git merge --no-ff:** Creates a merge commit even if fast-forward possible. Preserves release history in Git log.
**git tag -a v1.2.3:** Annotated tag (not lightweight). Contains tagger info, date, message. Marks exact commit for this release.
**git push origin main --tags:** Pushes both commits and tags to remote. CI/CD watches for tags and triggers deployment.

#### Querying Release History

```
$ git tag -l "v*"
v1.0.0
v1.1.0
v1.2.0
v1.2.1
v1.2.2
v1.2.3
$ git log --oneline --decorate | head -10
abc1234 (HEAD -> main, tag: v1.2.3) Bump version to 1.2.3
def5678 Merge release/1.2.3
ghi9012 Fix: database connection timeout
jkl3456 (tag: v1.2.2) Bump version to 1.2.2
$ git show v1.2.3
tag v1.2.3
Tagger: Priya Patel
Date: Mon Jan 15 14:32:00 2024
Message: Release version 1.2.3
commit abc1234de5f6g7h8i9j0
```

> **git tag -l:** Lists all tags. Easy to see release history. Last tag is always the latest release.
**git log --decorate:** Shows which commit is tagged with which version. Provides audit trail.
**git show v1.2.3:** Shows who created the tag, when, and commit details. Audit-friendly for compliance.

### 3. Changelog & Release Notes — Documenting Changes

#### CHANGELOG.md Format

```
# Changelog
## [1.2.3] - 2024-01-15
### Added
- User profile customization (name, avatar, timezone)
- API endpoint /api/users/{id}/profile for frontend to fetch profile
- Email verification on signup

### Fixed
- Database connection timeout when load > 1000 req/sec
- Incorrect calculation of project completion percentage
- Memory leak in WebSocket connection handler

### Changed
- API response format for /projects endpoint (BREAKING)
- Authentication now requires X-API-Key header

### Deprecated
- Old /api/user endpoint replaced by /api/users/{id}/profile

### Security
- Fixed SQL injection vulnerability in search (CVE-2024-1234)
- Upgraded dependencies: express 4.18 → 4.19, lodash 4.17 → 4.18

## [1.2.2] - 2024-01-08
### Fixed
- Dashboard graphs not loading for large datasets
- Export to CSV failing for files > 50 MB
```

> **Keep It Organized:** Added, Fixed, Changed, Deprecated, Security sections. Users know exactly what changed.
**Include Dates:** YYYY-MM-DD format. Teams know when a fix was released.
**Note Breaking Changes:** (BREAKING) tag warns users they must update config or code.
**Security Fixes:** Separate section. Critical for compliance teams monitoring CVEs.
**Reference Issues:** "Fix #1234" links to GitHub issue. Traceability.

### 4. CI/CD Pipelines — Automated Build, Test, Deploy

#### .github/workflows/release.yml — GitHub Actions

```bash
name: Release & Deploy
on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
run: npm ci
      
      - name: Run tests
run: npm test
      
      - name: Build artifacts
run: npm run build
      
      - name: Get version from package.json
id: version
        run: echo "version=$(cat package.json | grep version | head -1 | awk -F: '{ print $2 }' | sed 's/[",]//g' | tr -d ' ')" >> $GITHUB_OUTPUT
      
      - name: Build Docker image
run: docker build -t releaseflow-api:${{ steps.version.outputs.version }} .
      
      - name: Push to ECR
run: |
          aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com
          docker tag releaseflow-api:${{ steps.version.outputs.version }} 123456789.dkr.ecr.us-east-1.amazonaws.com/releaseflow-api:${{ steps.version.outputs.version }}
          docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/releaseflow-api:${{ steps.version.outputs.version }}

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
steps:
      - name: Deploy to staging
run: |
          ssh -i ${{ secrets.SSH_KEY }} ubuntu@staging.releaseflow.com << EOF
          docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/releaseflow-api:${{ needs.build.outputs.version }}
          docker stop releaseflow-api || true
          docker run -d --name releaseflow-api -p 3000:3000 123456789.dkr.ecr.us-east-1.amazonaws.com/releaseflow-api:${{ needs.build.outputs.version }}
          EOF

  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
      - name: Blue-Green Deployment
run: |
          ssh -i ${{ secrets.SSH_KEY }} ubuntu@prod.releaseflow.com << EOF
          ./deploy.sh ${{ needs.build.outputs.version }}
          EOF
```

> **on.push.tags: ['v*']** — workflow triggers when you push a tag starting with 'v'. This is your release trigger.
**on.push.branches: [main]** — also triggers on main branch pushes (non-tag). Use for staging deployments.
**npm ci** — clean install, respects lock file. More reliable than npm install.
**npm test** — unit tests run automatically. If tests fail, pipeline stops. Prevents broken code reaching production.
**Get version from package.json:** Reads version string, makes it available as ${{ steps.version.outputs.version }}.
**docker build -t releaseflow-api:1.2.3** — image tagged with version. Later deployed to production as this exact version.
**needs: build:** deploy jobs wait for build job to finish. If build fails, deployment doesn't happen.
**if: startsWith(github.ref, 'refs/tags/v'):** production deployment only on version tags, not on every main branch push.

### 5. Blue-Green Deployment — Zero-Downtime Releases

#### Understanding Blue-Green Deployment

```
    Blue-Green Deployment Strategy

        BEFORE (v1.2.2):
        Load Balancer → Blue Server (v1.2.2 active)
                     → Green Server (v1.2.1 idle)

        DEPLOY v1.2.3:
        Load Balancer → Blue Server (v1.2.2 still active)
                     → Green Server (v1.2.3 deployed, warming up)
        
        TEST:
        - Health checks on Green ✓
        - Smoke tests on Green ✓
        - Load tests on Green ✓
        
        SWITCH:
        Load Balancer → Blue Server (v1.2.2 now idle)
                     → Green Server (v1.2.3 now active)
        
        Traffic flows to v1.2.3. 0 downtime.
        
        ROLLBACK (if bug found):
        Load Balancer → Blue Server (v1.2.2 re-activated)
                     → Green Server (v1.2.3 idle)
        
        Instant rollback. All users see v1.2.2 again.
        
```

#### Blue-Green Deployment Script

```bash
#!/bin/bash
# deploy.sh — Blue-Green deployment

VERSION=$1
REGION="us-east-1"
PROD_LOAD_BALANCER="prod-lb.releaseflow.com"

echo "Deploying v$VERSION to Green environment..."
# Step 1: Pull latest image from ECR
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.$REGION.amazonaws.com

docker pull 123456789.dkr.ecr.$REGION.amazonaws.com/releaseflow-api:$VERSION

# Step 2: Start Green container
docker run -d --name releaseflow-api-green \
  -p 3001:3000 \
  -e APP_VERSION=$VERSION \
  123456789.dkr.ecr.$REGION.amazonaws.com/releaseflow-api:$VERSION

# Step 3: Wait for health checks
echo "Waiting for Green to be healthy..."
for i in {1..30}; do
  if curl -f http://localhost:3001/health > /dev/null 2>&1; then
    echo "✓ Green is healthy"
    break
  fi
  echo "Waiting... ($i/30)"
  sleep 2
done

# Step 4: Run smoke tests on Green
echo "Running smoke tests on Green..."
npm run test:smoke -- --base-url=http://localhost:3001
if [ $? -ne 0 ]; then
  echo "❌ Smoke tests failed! Rolling back..."
  docker stop releaseflow-api-green
  docker rm releaseflow-api-green
  exit 1
fi

# Step 5: Switch load balancer to Green
echo "Switching load balancer to Green..."
aws elb set-instance-health \
  --load-balancer-name prod-lb \
  --instances i-blue-instance \
  --state OutOfService

aws elb set-instance-health \
  --load-balancer-name prod-lb \
  --instances i-green-instance \
  --state InService

# Step 6: Monitor Green for 5 minutes
echo "Monitoring v$VERSION in production (5 minutes)..."
for i in {1..30}; do
  ERROR_RATE=$(curl -s http://localhost:3001/metrics | grep error_rate | awk '{print $2}')
  if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
    echo "❌ Error rate too high! Rolling back..."
    ./rollback.sh
    exit 1
  fi
  echo "Monitoring... error_rate: $ERROR_RATE (10s intervals)"
  sleep 10
done

echo "✓ v$VERSION deployed successfully!"
```

> **Step 1: Pull image** — fetches latest Docker image for this version from ECR.
**Step 2: Start Green** — starts new container on port 3001 (different from Blue's 3000). No traffic yet.
**Step 3: Health checks** — waits for /health endpoint to return 200. If fails after 30 attempts (60s), rollback immediately.
**Step 4: Smoke tests** — automated tests verify critical paths. Payment, login, search. If any fail, rollback. No humans required.
**Step 5: Switch LB** — AWS ELB changes health status. Old Blue goes OutOfService, Green goes InService. Traffic now flows to Green.
**Step 6: Monitor** — watch error rate for 5 minutes. If spikes above 5%, automatic rollback. Old Blue re-activated.

### 6. Automated Rollback — Quick Recovery from Failures

#### Rollback Script

```bash
#!/bin/bash
# rollback.sh — Instant rollback to previous version

CURRENT_VERSION=$(docker inspect releaseflow-api-blue | \
  grep APP_VERSION | \
  awk -F= '{print $2}' | tr -d '"')

PREVIOUS_VERSION=$(git describe --tags --abbrev=0 $CURRENT_VERSION^)

echo "Rolling back from $CURRENT_VERSION to $PREVIOUS_VERSION..."
# Step 1: Get current Blue instance health status
BLUE_INSTANCE=$(aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=prod" "Name=tag:Role,Values=blue" \
  --query 'Instances[0].InstanceId' \
  --output text)

# Step 2: Switch LB back to Blue
aws elb set-instance-health \
  --load-balancer-name prod-lb \
  --instances $BLUE_INSTANCE \
  --state InService

aws elb set-instance-health \
  --load-balancer-name prod-lb \
  --instances i-green-instance \
  --state OutOfService

# Step 3: Notify team
curl -X POST https://slack.com/api/chat.postMessage \
  -H 'Content-Type: application/json' \
  -d '{
    "channel": "#deployments",
    "text": "⚠️ ROLLBACK: v'$CURRENT_VERSION' → v'$PREVIOUS_VERSION' — Error rate threshold exceeded"
  }' \
  -H "Authorization: Bearer $SLACK_TOKEN"

echo "✓ Rolled back to v$PREVIOUS_VERSION"
```

> **CURRENT_VERSION:** reads version from running Blue container. Knows exactly what's running.
**PREVIOUS_VERSION:** queries Git for the tag before current. Automatic rollback target.
**Switch LB:** Green goes OutOfService, Blue re-activated. Traffic reroutes in seconds.
**Slack notification:** team is informed immediately. No surprises.
**Rollback is instant.** No manual intervention. No waiting. Users see the previous stable version within 30 seconds.

### 7. Complete Release Process — End-to-End Workflow

#### Step-by-Step Release Checklist

```
# 1. Prepare Release Branch
$ git checkout develop
$ git pull origin develop
$ git checkout -b release/1.3.0

# 2. Bump Version
$ npm version minor              # Auto-bumps, commits, tags
# OR manually:
$ sed -i 's/"version": "1.2.3"/"version": "1.3.0"/' package.json
$ git add package.json
$ git commit -m "Bump to v1.3.0"
# 3. Update Changelog
$ nano CHANGELOG.md  # Add v1.3.0 section with all changes
$ git add CHANGELOG.md
$ git commit -m "Update CHANGELOG for v1.3.0"
# 4. Run full test suite
$ npm run test:full
$ npm run build

# 5. Create PR to main
$ git push origin release/1.3.0
# Create PR: release/1.3.0 → main on GitHub
# Get 2 approvals from leads
# 6. Merge to main
$ git checkout main
$ git merge --no-ff release/1.3.0
$ git tag -a v1.3.0 -m "Release v1.3.0"
$ git push origin main --tags

# 7. CI/CD automatically triggers
# - Builds Docker image: releaseflow-api:1.3.0

# - Pushes to ECR

# - Deploys to staging

# - Runs smoke tests

# 8. Blue-Green deployment to production
$ ssh ubuntu@prod.releaseflow.com
$ ./deploy.sh 1.3.0
# - Starts Green with v1.3.0

# - Health checks: ✓

# - Smoke tests: ✓

# - Monitors for 5 min: ✓

# - Deployment successful!
```

> **npm version minor:** automatically increments version, commits, creates annotated tag. npm tool for Node.js.
**Full test suite:** unit tests, integration tests, e2e tests. 30-60 minutes. Catches bugs before production.
**PR approval process:** humans review changelog, version bump, code. Catch mistakes before they reach main.
**--no-ff merge:** preserves release branch history. Clean Git log showing release boundaries.
**Push with --tags:** sends both commits and tags. GitHub webhook triggers CI/CD on tag push.
**Automated deployment:** no manual server work. Script handles health checks, testing, switching.

### 8. Post-Release Monitoring — Track Release Health

#### Release Dashboard Metrics

```
# Prometheus metrics scraped from running app
releaseflow_version{version="1.3.0"} 1

http_requests_total{version="1.3.0", status="200"} 145230
http_requests_total{version="1.3.0", status="500"} 42

http_request_duration_seconds{version="1.3.0", 
  endpoint="/api/projects", quantile="0.95"} 0.234

database_connection_pool_active{version="1.3.0"} 42
database_connection_pool_idle{version="1.3.0"} 8

errors_total{version="1.3.0", error_type="timeout"} 3
errors_total{version="1.3.0", error_type="validation"} 12
```

> **releaseflow_version:** which version is running. Grafana queries this to label dashboards.
**http_requests_total:** count of requests by status code. Error rate = 500s / total.
**http_request_duration_seconds (p95):** 95th percentile latency. If v1.3.0 is slower than v1.2.3, alerts fire.
**database_connection_pool:** active/idle connections. Spikes indicate connection leak or new feature using more DB.
**errors_total by type:** timeouts, validation, auth failures. Breakdown helps identify root cause.

#### Grafana Dashboard Alerts

```
# Alert if error rate > 2%
alert: HighErrorRate
expr: (http_requests_total{status="5xx"} / http_requests_total) > 0.02
for: "5m"
annotations:
  summary: "v{{ $labels.version }} error rate {{ $value | humanizePercentage }}"
action: "ROLLBACK"
# Alert if p95 latency > 500ms
alert: HighLatency
expr: http_request_duration_seconds{quantile="0.95"} > 0.5
for: "3m"
annotations:
  summary: "v{{ $labels.version }} p95 latency {{ $value }}s"
```

> **Error rate > 2% for 5 minutes:** automatically triggers rollback. No human decision needed.
**for: 5m:** alert must be true for 5 minutes before firing. Filters transient spikes (normal in production).
**Latency > 500ms for 3 min:** indicates performance regression. Alerts team to investigate. May not auto-rollback (less critical than errors).
**Metrics are labeled with version:** easy to compare v1.3.0 vs v1.2.3 side-by-side in Grafana.

**📍 Scene: ReleaseFlow's First Automated Release**

> **Amit**
> 
> "Release day for v1.3.0. Developers have committed features to develop for 2 weeks. We create release/1.3.0, bump the version, update CHANGELOG. The PR is reviewed and merged to main. We tag it v1.3.0."

> **Priya**
> 
> "GitHub sees the tag. CI/CD workflow triggers automatically. Within 10 minutes: tests pass, Docker image built, pushed to ECR, deployed to staging, smoke tests pass. Blue-green deployment starts on production."

> **Neeraj (QA)**
> 
> "Green container starts with v1.3.0. Health checks pass. Smoke tests verify critical paths: login, create project, add task. Error rate is 0.8% (normal). After 5 minutes of monitoring, v1.3.0 is live. All traffic flows to the new version."

> **Amit**
> 
> "And if there was a bug in v1.3.0? The monitoring alert would fire if error rate hit 2%. Rollback script would run automatically. Users would see v1.2.3 again within 30 seconds. They'd never notice the broken release."

> **Release Management — Core Takeaways for Freshers**

> - **Semantic versioning creates structure.** Version 1.2.3 means: major breaking change, 2 minor features, 3 patches. Predictable versioning prevents confusion.
> - **Git tags mark release boundaries.** v1.2.3 tag points to exact commit. 6 months later, you pull v1.2.3 and know exactly what code was live. Audit trail for compliance.
> - **CHANGELOG documents changes.** Users read CHANGELOG to understand what's new, what broke, what's fixed. No surprises. Transparency builds trust.
> - **CI/CD pipelines automate everything.** Humans should never manually run `npm install` on a production server. Pipelines do it: build, test, deploy. Consistency guaranteed.
> - **Blue-Green deployments = zero downtime.** Run old and new versions simultaneously. Test new version thoroughly. Switch traffic instantly. If bug: switch back in 30 seconds. No users see downtime.
> - **Automated rollback saves the day.** Error rate spikes? Automatic rollback. No waiting for on-call engineer. No 3 AM page. Old version re-activated instantly.
> - **Metrics are your window into production.** Error rate, latency, database connections, memory usage. Compared side-by-side: v1.3.0 vs v1.2.3. Performance regression caught immediately.
> - **Releases must be reproducible.** Same release process every time. No "oops I forgot to run tests." Checklist, PR approval, CI/CD gates. Process prevents disasters.

##### Release Management Standards — ReleaseFlow Production Rules

- Always use semantic versioning. Never use dates (2024.01.15), rolling versions (latest), or arbitrary strings. Semantic versioning is a contract with users.
- Tag every release in Git. Create annotated tags with `-a` flag, not lightweight tags. Include tagger info, date, message. Makes audit trails clean.
- Update CHANGELOG before every release. Separate sections: Added, Fixed, Changed, Deprecated, Security. Users read CHANGELOG to decide if they need to upgrade.
- Run full test suite on release branches. Unit tests, integration tests, e2e tests. If tests fail, don't release. Broken releases are worse than delayed releases.
- Use blue-green deployments for production. New version runs on Green while Blue serves traffic. Only switch when Green is proven healthy. Zero downtime, instant rollback.
- Include healthchecks in every release. /health endpoint must respond with status. Deploy script verifies health before switching. Dead apps are caught instantly.
- Automate rollback on metrics. If error rate > threshold for N minutes, automatic rollback. No human decision, no delay. Protect users from broken releases.
- Canary releases for major changes. Roll out v2.0.0 to 5% of users first. Monitor error rates. If healthy, increase to 50%, then 100%. Catch issues early on small segment.

### 9. Advanced Release Strategies — Canary, Feature Flags, Version Pinning

#### Canary Releases

```
    Canary Release Strategy

        v1.2.3 running on 100% of traffic
        
        Deploy v1.3.0 (breaking API changes):
        
        Hour 0: v1.3.0 on 5% of traffic (canary)
          ├─ Error rate: 2.1% (baseline: 1.8%) → Investigate
          ├─ Users affected: ~100 (out of 20,000)
          ├─ Quick rollback possible
        
        Hour 1: v1.3.0 on 20% of traffic
          ├─ Error rate: 1.9% (normal) → Proceed
          ├─ Latency: 215ms (baseline: 210ms) → Acceptable
        
        Hour 2: v1.3.0 on 50% of traffic
          ├─ No issues detected → Proceed
        
        Hour 4: v1.3.0 on 100% of traffic
          ├─ Full rollout complete
          ├─ Confidence: high (monitored for 4+ hours)
        
        If issues at any stage:
          └─ Rollback to v1.2.3 affecting small % only
        
```

#### Feature Flags — Release Without Deploying

```
// Code with feature flag
if (featureFlags.isEnabled('new-search-engine')) {
  // v1.3.0: New Elasticsearch search
  results = await elasticsearchClient.search(query);
} else {
  // v1.2.3: Old SQL LIKE search
  results = await database.query(
    'SELECT * FROM projects WHERE name LIKE ?', [query]
  );
}

// Feature flag config in database
{
  flag_name: 'new-search-engine',
  enabled: false,
  rollout: {
    percentage: 0,
    user_segments: ['internal-team']
  }
}

# Deploy v1.3.0 with new search engine disabled
# Users still use old search engine
# Update feature flag: percentage = 10
# 10% of users get new search engine (no redeploy!)
# Monitor: error_rate, latency
# If good: percentage = 100 (still no redeploy)
# If bad: percentage = 0 (instant rollback, no redeploy)
```

> **Deploy once, rollout gradually:** code for both old and new feature. Control which % of users see which version via flag.
**No redeployment needed:** feature flag changes are instant database updates. No CI/CD pipeline. No service restart.
**User segments:** A/B testing built in. "internal-team" always sees new feature. Regular users don't. Employees test first.
**Instant rollback:** set percentage to 0. All users see old feature. No container restart, no git rollback, no deployment. Instant.

##### 🏋️ Hands-On Exercises — Master Release Management

1. **Set up semantic versioning:** Create a package.json with version 1.0.0. Create CHANGELOG.md with the initial release notes. Add a git tag: git tag -a v1.0.0 -m "Initial release". Create a simple Node.js app with a /version endpoint that returns {version: "1.0.0"}. Use npm version minor to bump to 1.1.0 (it auto-commits and tags). Verify git tag v1.1.0 exists.
2. **Create a release workflow:** Set up .github/workflows/release.yml. Trigger on tags matching v*. Steps: checkout code, install dependencies, run tests, build artifacts, read version from package.json. Use outputs to pass version to later steps. Commit and push a tag v1.1.0 to see the workflow trigger in GitHub Actions.
3. **Implement blue-green deployment:** Create two directories: blue/ and green/. Each contains a simple Node.js app on different ports (3000 and 3001). Write a script that: starts a new container in green, runs health checks, switches an nginx reverse proxy to point to green, monitors for errors for 2 minutes. Test by starting blue with v1.0.0, deploying green with v1.1.0, verifying traffic switched.
4. **Write a rollback script:** Create rollback.sh that: reads current version from blue container, finds previous git tag, switches nginx back to blue, notifies Slack. Manually trigger it after a blue-green deployment and verify the traffic switches back instantly.
5. **Set up release monitoring:** Add Prometheus metrics to your app: version gauge, request counters by status, latency histograms. Expose /metrics endpoint. Write Prometheus alerts: error_rate > 2% triggers CRITICAL. Set up a Grafana dashboard showing error rate and latency side-by-side for v1.0.0 vs v1.1.0. Simulate a bad release (high error rate) and watch the alert fire.

### Release Management Project Complete 🎉

You have mastered release management — the discipline that turns chaos into predictable, safe deployments. You understand semantic versioning, Git workflows, CI/CD automation, blue-green deployments, automated rollbacks, and production monitoring. You can release new code to production with confidence: if it breaks, automatic rollback protects users. This is how enterprise companies release software safely, repeatedly, at scale.

> **Priya**
> 
> "Release management transformed our company. Before: releases were chaotic, manual, error-prone. Someone would forget to run tests. Someone else would manually copy files to production. Deployments took 4 hours. Rollbacks were painful. Now: completely automated. Tests run automatically. Docker images built automatically. Deployed automatically. Monitored automatically. If something breaks, automatic rollback. Releases are boring, predictable, safe. That's exactly what you want in production."

> **Amit**
> 
> "And the confidence it gives. We release every 2 weeks now. No fear. We know exactly what changed (CHANGELOG), we know it was tested (CI/CD), we know we can rollback instantly if needed (blue-green + automated monitoring). That's the power of release management. You've just learned how to deploy like a professional."

> **Next: Advanced DevOps & Infrastructure as Code**

> - Infrastructure as Code (Terraform, CloudFormation) — deploy entire infrastructure with `terraform apply`. Version control, code review, rollback.
> - Multi-region deployments — release to multiple AWS regions simultaneously. Handle regional failures gracefully.
> - Kubernetes releases — Helm charts version your Kubernetes deployments. Helm rollback if needed.
> - GitOps — Git is the source of truth. Push changes to Git, ArgoCD automatically deploys them. Declarative, auditable, rollback-friendly.
> - Feature management platforms — LaunchDarkly, Flagsmith. Enterprise feature flags with A/B testing, gradual rollouts, user targeting.
> - Observability at scale — Prometheus, Grafana, ELK stack, Datadog. Real-time insights into production behavior across 100+ services.
