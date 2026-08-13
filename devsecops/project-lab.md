# 🛡️ DevSecOps Project Mastery

> **👋 Hey Fresher — Read This First!**

> DevSecOps means security checks run automatically inside your CI/CD pipeline, on every single commit — not as a separate audit that happens once a quarter, weeks after the code already shipped. Instead of a security team manually reviewing code before a release (slow, and it happens too late to fix things cheaply), the pipeline itself scans your source code, your dependencies, your secrets, and your container images every time you push — and blocks the merge if it finds something dangerous. The goal is "shift left": catch the vulnerability while you're still writing the code, not after it's running in production with real patient data flowing through it.

> **Company in this project:** MediSetu — a healthtech startup in Pune building a telemedicine platform that lets patients book video consultations and get e-prescriptions from doctors. Because MediSetu handles medical records and prescription data, a security breach isn't just embarrassing — it's a regulatory and legal disaster. You just joined as a Junior DevSecOps Engineer. Your mentor is Rohit, and the engineering director pushing this initiative after a scare with a leaked API key is Sunita. Let's build a CI/CD pipeline that catches vulnerabilities before they ever reach production.

#### What You Will Learn and Build in This Project

You will build a security-hardened CI/CD pipeline for MediSetu's telemedicine platform — layering static analysis, dependency scanning, secrets detection, container image scanning, and dynamic testing into GitHub Actions, so the pipeline itself becomes the security gatekeeper instead of a human doing it manually after the fact.

SAST, SCA, Secrets Scanning, Container Image Scanning, DAST, Software Bill of Materials, Shift-Left Security, Pipeline Gating, Vulnerability Severity Triage

> **📦 Phase 1 — Static Analysis (SAST)**
>
> Scan MediSetu's source code for insecure patterns — SQL injection, hardcoded credentials, unsafe deserialization — before the code is even merged.

> **📦 Phase 2 — Software Composition Analysis (SCA)**
>
> Scan every third-party dependency MediSetu pulls in for known CVEs, because most vulnerabilities live in libraries you didn't write, not in your own code.

> **📦 Phase 3 — Secrets Scanning**
>
> Catch API keys, database passwords, and tokens before they're committed to Git history — because once a secret is in Git history, it's compromised forever, even if you delete it later.

> **📦 Phase 4 — Container Image Scanning**
>
> Scan the Docker image itself, including the OS packages inside it, before it's allowed to be pushed to the registry or deployed.

> **📦 Phase 5 — Dynamic Analysis (DAST)**
>
> Attack MediSetu's running staging environment like a real attacker would — SAST can't catch everything, because some vulnerabilities only exist when the app is actually running.

> **📦 Phase 6 — Gate the Pipeline and Generate an SBOM**
>
> Wire every scanner's severity output into a single pass/fail gate, and produce a Software Bill of Materials for compliance and incident response.

**Scene 1 — MediSetu Engineering Office, Pune | The Leaked Razorpay Key**

> **Sunita** _Engineering Director — MediSetu_
>
> Ananya, before you write any pipeline code, you need to understand why this project exists. Three weeks ago, a contractor committed a Razorpay secret key directly into a config file, pushed it to our public documentation repo by mistake, and it sat there for six days before anyone noticed. Anyone browsing GitHub could have used that key to process fraudulent refunds against our account. We got lucky — nobody exploited it before we rotated it. We are not relying on luck again.

> **Rohit** _Senior DevSecOps Engineer — MediSetu_
>
> And that's just one category of vulnerability — a leaked secret. We also have zero automated checks for SQL injection in our Django code, zero checks for vulnerable versions of the Python packages we depend on, and we've never scanned our Docker images for OS-level CVEs. Right now, our only "security review" is a senior engineer eyeballing the diff in a pull request, and humans miss things — especially at 11 PM before a release deadline.

> **Ananya (You)** _Junior DevSecOps Engineer — Day 1_
>
> That sounds like a lot of ground to cover. Where do we even start?

> **Rohit** _Senior DevSecOps Engineer_
>
> One layer at a time, and we automate each one directly into the GitHub Actions pipeline so it runs on every single pull request — no one has to remember to run it manually. We start with source code (SAST), then dependencies (SCA), then secrets, then the container image itself, and finally we attack our own staging environment before an actual attacker gets the chance.

### 1. Phase 1 — Static Application Security Testing (SAST)

**Business Problem:** MediSetu's Django backend handles patient search, e-prescription generation, and appointment booking. A single unparameterized SQL query or an unsafe use of `eval()` could expose thousands of patient records. SAST tools read source code without running it, looking for known-dangerous patterns.

#### 1.1 Adding Semgrep to the GitHub Actions Pipeline

```yaml
# .github/workflows/security.yml
name: Security Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep SAST scan
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/python
            p/django
            p/secrets
            p/owasp-top-ten
        env:
          SEMGREP_SEND_METRICS: "off"
```

> **📖 What Semgrep Is Actually Doing**
>
> `config: p/python p/django p/owasp-top-ten` — these are Semgrep's public rule packs. `p/django` specifically catches Django anti-patterns like raw SQL built with string formatting instead of the ORM's parameterized queries, `mark_safe()` calls that could enable XSS, and Django settings with `DEBUG = True` left on. `p/owasp-top-ten` checks for the OWASP Top 10 web vulnerability categories generally (injection, broken auth, sensitive data exposure). Semgrep parses your code into an abstract syntax tree and pattern-matches against known-bad code shapes — it does not run your code, so it's fast enough to run on every single pull request in under a minute.

#### 1.2 A Real Finding — SQL Injection in the Patient Search Endpoint

```python
# BEFORE — flagged by Semgrep as sql-injection (High)
def search_patients(request):
    name = request.GET.get("name")
    query = f"SELECT * FROM patients WHERE name LIKE '%{name}%'"
    cursor.execute(query)  # ⚠️ raw string interpolation into SQL

# AFTER — fixed using parameterized query
def search_patients(request):
    name = request.GET.get("name")
    cursor.execute(
        "SELECT * FROM patients WHERE name LIKE %s",
        [f"%{name}%"],
    )
```

> **📖 Why the "Before" Version Is Dangerous**
>
> In the "before" version, if someone searches for `'; DROP TABLE patients; --`, that string gets concatenated directly into the SQL statement and executed as-is — a classic SQL injection. The "after" version passes `name` as a separate parameter (`%s` placeholder), so the database driver treats it strictly as *data*, never as executable SQL syntax, no matter what characters it contains. This is exactly the kind of pattern Semgrep's `p/django` ruleset is built to catch automatically, before a human reviewer even reads the diff.

**Quiz: Semgrep flags a finding as "High" severity for a raw SQL string built with an f-string. A developer argues "but the `name` variable always comes from an internal admin tool, never from a public user — can we ignore this?"**
- Yes, ignore it — internal-only inputs can never be malicious
- No — treat all unvalidated string concatenation into SQL as a defect regardless of current caller, because code gets reused and callers change over time; fix it or add a documented, reviewed suppression with a clear justification
- Delete the Semgrep rule from the config so it stops flagging this pattern everywhere
- Only fix it if the tool also finds a matching DAST finding

> **Answer/explanation:** The second option is the correct security engineering discipline. "This input is currently safe because of how it's called today" is exactly the kind of assumption that breaks silently months later — someone adds a new public-facing caller to `search_patients()`, or copies the pattern into a new function, and the injection risk is live again with nobody remembering the original safety argument. The right move is either to fix the code with a parameterized query (cheap, permanent) or, if there's a genuine false positive, add a scoped, documented suppression comment explaining exactly why — never silently disable the rule project-wide, which would blind the pipeline to the same bug appearing anywhere else in the codebase.

### 2. Phase 2 — Software Composition Analysis (SCA)

**Business Problem:** MediSetu's Django backend depends on over 140 third-party Python packages. Most of MediSetu's own code has been reviewed — but nobody has audited whether `Pillow==9.0.0` or `PyYAML==5.3.1` (both pulled in transitively) have known critical vulnerabilities. SCA tools compare your exact dependency versions against public vulnerability databases.

**Scene 2 — Standup | "We're Running a CVE From 2022"**

> **Rohit** _Senior DevSecOps Engineer_
>
> I ran `pip-audit` locally against our requirements.txt out of curiosity. We're running `PyYAML 5.3.1`, which has a known arbitrary code execution vulnerability — CVE-2020-14343 — that was patched over three years ago. Nobody bumped the version because nothing told us to. That's the whole argument for SCA in CI: humans don't proactively re-check dependency versions against CVE databases. A scanner does it automatically, every single build.

#### 2.1 Dependency Scanning with pip-audit

```yaml
# .github/workflows/security.yml (continued)
  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install pip-audit
        run: pip install pip-audit

      - name: Scan dependencies for known CVEs
        run: |
          pip-audit -r requirements.txt \
            --format json \
            --output pip-audit-report.json \
            --strict

      - name: Upload SCA report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: pip-audit-report
          path: pip-audit-report.json
```

> **📖 pip-audit — Reading the Output**
>
> `pip-audit` cross-references every pinned version in `requirements.txt` against the Python Packaging Advisory Database (PyPA's own CVE feed, sourced from OSV). `--strict` makes the command exit with a non-zero code if any vulnerability is found — this is what actually fails the GitHub Actions job. `--format json` produces a machine-readable report that's uploaded as a build artifact with `upload-artifact`, so anyone can download and inspect exactly which package, which version, and which CVE ID triggered the finding, even after the pipeline run.

#### 2.2 Fixing the Vulnerable Dependency

```bash
# Confirm the fixed version and available range
pip index versions pyyaml

# Update the pin in requirements.txt
# BEFORE: PyYAML==5.3.1
# AFTER:  PyYAML==6.0.1

pip install -r requirements.txt
pip-audit -r requirements.txt --strict   # should now exit 0
```

**SAST vs SCA vs DAST — Three Different Jobs**

- **SAST (Static Application Security Testing)** — scans *your own source code* without running it. Catches bugs like SQL injection and hardcoded secrets that you wrote yourself. Fast, runs on every commit.
- **SCA (Software Composition Analysis)** — scans your *third-party dependencies* (and their transitive dependencies) against public CVE databases. You didn't write this code, but you're still responsible for shipping it — most real-world breaches come from a known, unpatched CVE in a dependency, not a bug in first-party code.
- **DAST (Dynamic Application Security Testing)** — attacks your *running application* from the outside, like a real attacker with no access to source code. Catches things SAST structurally cannot see, like misconfigured HTTP headers, broken authentication flows, or vulnerabilities that only appear at runtime. Slower, usually run against a staging environment rather than on every commit.

### 3. Phase 3 — Secrets Scanning

**Business Problem:** The incident that triggered this whole project — a Razorpay key committed to Git. Once a secret lands in Git history, deleting the file in a later commit does not remove it; the secret is still retrievable from any earlier commit unless history is rewritten. The fix is to catch it *before* it's ever pushed, or at minimum before it's merged.

#### 3.1 Gitleaks in the Pipeline

```yaml
# .github/workflows/security.yml (continued)
  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # full history, not just the latest commit

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITLEAKS_ENABLE_UPLOAD_ARTIFACT: true
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

> **📖 Why fetch-depth: 0 Matters Here**
>
> GitHub Actions checks out a shallow clone (just the latest commit) by default to save time. Gitleaks needs `fetch-depth: 0` — the full commit history — because a secret could have been committed several commits back in this same pull request and then "removed" in a later commit; without full history, Gitleaks would never see the commit where the secret was actually introduced. Gitleaks scans every commit's diff for regex patterns matching known credential formats — AWS access keys, Razorpay/Stripe API keys, private key blocks (`-----BEGIN RSA PRIVATE KEY-----`), and generic high-entropy strings that look like tokens.

#### 3.2 Pre-Commit Hook — Catch It Before It Even Leaves Your Laptop

```bash
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks

# Install the hook locally
pip install pre-commit
pre-commit install
```

> **📖 Shift Left Even Further — Pre-Commit vs CI**
>
> The GitHub Actions job in 3.1 catches a secret when a pull request is opened — but by then, it's already been pushed to GitHub, even if the PR is never merged. A pre-commit hook runs Gitleaks locally, on your machine, before `git commit` even completes — if it finds a secret, the commit is blocked and nothing ever reaches GitHub's servers. Both layers matter: pre-commit hooks can be skipped with `--no-verify`, so the CI-level Gitleaks scan in the pipeline is the real, unbypassable enforcement point. Pre-commit is a courtesy that saves you an embarrassing PR comment; CI is the actual gate.

> **Key takeaways**
> - SAST, SCA, and secrets scanning each catch a different category of risk — none of them substitutes for the others, and a mature pipeline runs all three.
> - Deleting a committed secret in a later commit does not remove it from Git history — rotate the credential immediately regardless of whether the file was later removed.
> - `fetch-depth: 0` is required for any tool that needs to inspect full Git history, including secrets scanners and some SBOM generators.
> - Pre-commit hooks are a convenience layer, not a security boundary — they can be bypassed. The CI pipeline is the actual enforcement point.

### 4. Phase 4 — Container Image Scanning

**Business Problem:** MediSetu's Django app runs in a Docker container built `FROM python:3.11-slim`. Even if MediSetu's own code and Python dependencies are clean, the base image itself ships with OS packages (glibc, openssl, apt packages) that can have their own CVEs — and those need to be scanned too, before the image is pushed to the registry and deployed.

#### 4.1 Trivy Image Scan Before Registry Push

```yaml
# .github/workflows/security.yml (continued)
  image-scan:
    runs-on: ubuntu-latest
    needs: [sast, sca, secrets-scan]
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t medisetu/api:${{ github.sha }} .

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: medisetu/api:${{ github.sha }}
          format: table
          severity: CRITICAL,HIGH
          exit-code: "1"
          ignore-unfixed: true

      - name: Push image (only if scan passed)
        if: success()
        run: |
          docker tag medisetu/api:${{ github.sha }} \
            registry.medisetu.in/api:${{ github.sha }}
          docker push registry.medisetu.in/api:${{ github.sha }}
```

> **📖 Trivy Scan — What Each Flag Controls**
>
> `needs: [sast, sca, secrets-scan]` — the image build and scan only run after the earlier, faster checks pass, so you don't waste time building and scanning an image for a commit that already failed a cheaper check. `severity: CRITICAL,HIGH` — only fail on the most dangerous findings; Trivy would otherwise also report LOW and MEDIUM findings that would create noise and alert fatigue. `exit-code: "1"` — this is what actually fails the GitHub Actions job when a CRITICAL or HIGH vulnerability is found; without it, Trivy just prints a report and the pipeline continues regardless. `ignore-unfixed: true` — don't fail the build on a CVE that has no available patched version yet; there's nothing actionable you can do about it today, and blocking every deploy on it would just freeze the pipeline. The `docker push` step is gated behind the scan passing — an image with unresolved CRITICAL findings never reaches the registry MediSetu's Kubernetes cluster pulls from.

**Trivy vs Grype for Image Scanning**

- **Trivy** — broader coverage: OS packages, language dependencies, misconfigurations (via `trivy config`), and even Kubernetes manifests, all in one tool. MediSetu picked Trivy because one tool covers image scanning and later IaC scanning too, reducing pipeline complexity.
- **Grype** — narrower focus on vulnerability matching specifically, often cited as having a slightly different (sometimes more precise) vulnerability database matching for certain ecosystems. A reasonable choice if you already use Syft for SBOM generation, since Grype consumes Syft's SBOM format natively.

### 5. Phase 5 — Dynamic Application Security Testing (DAST)

**Business Problem:** SAST and dependency scanning both look at code without running it. But some vulnerabilities only exist when the application is actually live — missing security headers, a login endpoint that doesn't rate-limit password attempts, session cookies without the `Secure` flag. MediSetu needs to attack its own staging environment before a real attacker does.

**Scene 3 — Pre-Launch Review | Attacking Our Own App**

> **Rohit** _Senior DevSecOps Engineer_
>
> SAST would never have caught this, because it's not a code pattern — it's a runtime configuration. Our staging login endpoint accepts unlimited password attempts with no lockout or rate limiting. That's a textbook brute-force vulnerability, and the only way to find it is to actually attack the running app, exactly like an attacker would.

#### 5.1 OWASP ZAP Baseline Scan Against Staging

```yaml
# .github/workflows/security.yml (continued)
  dast:
    runs-on: ubuntu-latest
    needs: [image-scan]
    steps:
      - name: ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.12.0
        with:
          target: "https://staging.medisetu.in"
          rules_file_name: ".zap/rules.tsv"
          cmd_options: "-a"
          fail_action: true
```

> **📖 ZAP Baseline Scan — What It Actually Does**
>
> `action-baseline` spins up OWASP ZAP (Zed Attack Proxy) and crawls `https://staging.medisetu.in` like a browser would, then passively analyzes every response for missing security headers (`Content-Security-Policy`, `X-Frame-Options`, `Strict-Transport-Security`), cookies missing the `Secure` or `HttpOnly` flags, and other runtime-observable issues. `-a` includes ZAP's spider in ajax mode to crawl JavaScript-rendered pages, which matters because MediSetu's booking flow is a React SPA. `rules_file_name: .zap/rules.tsv` lets the team downgrade specific rules to informational if they're confirmed false positives for MediSetu's setup, instead of failing the build on noise. `fail_action: true` makes the GitHub Actions job fail if ZAP finds anything at WARN level or above.

#### 5.2 A Real DAST Finding — Missing Rate Limiting on Login

```python
# BEFORE — settings.py, no throttling configured
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
    ],
}

# AFTER — add throttling scoped to the login endpoint
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
    ],
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.ScopedRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "login": "5/minute",
    },
}
```

> **📖 Rate Limiting Stops Credential Stuffing**
>
> `"login": "5/minute"` limits each client to 5 login attempts per minute on the endpoint tagged with that throttle scope. This doesn't just stop a manual brute-force attempt — it substantially slows down automated credential-stuffing attacks, where an attacker tries millions of leaked username/password pairs from other data breaches against MediSetu's login form, hoping patients reused passwords. ZAP's DAST scan flagged the *absence* of this control, something no SAST tool scanning source code would catch, because there's no "bad pattern" to match — it's a missing runtime behavior.

### 6. Phase 6 — Gate the Pipeline and Generate an SBOM

**Business Problem:** MediSetu now has five separate scanners (SAST, SCA, secrets, image, DAST) each producing their own findings. Without a single clear gate, a developer could see five different reports and still not know whether it's actually safe to merge. And when a customer's compliance team asks "what exact software is running in production," MediSetu needs a precise, generatable answer — a Software Bill of Materials.

#### 6.1 The Combined Gate Job

```yaml
# .github/workflows/security.yml (continued)
  security-gate:
    runs-on: ubuntu-latest
    needs: [sast, sca, secrets-scan, image-scan, dast]
    if: always()
    steps:
      - name: Evaluate all scan results
        run: |
          if [ "${{ needs.sast.result }}" != "success" ] || \
             [ "${{ needs.sca.result }}" != "success" ] || \
             [ "${{ needs.secrets-scan.result }}" != "success" ] || \
             [ "${{ needs.image-scan.result }}" != "success" ] || \
             [ "${{ needs.dast.result }}" != "success" ]; then
            echo "❌ One or more security checks failed. Blocking merge."
            exit 1
          fi
          echo "✅ All security gates passed."
```

> **📖 One Required Status Check, Five Underlying Jobs**
>
> This `security-gate` job is set as the single **required status check** in MediSetu's GitHub branch protection rules for `main` — not each individual scanner job. That means engineers see one clear pass/fail signal on their PR, while `needs: [sast, sca, secrets-scan, image-scan, dast]` still means this job only runs after all five finish, and `if: always()` ensures it runs and evaluates results even if an earlier job failed (so the gate itself always reports accurately, rather than getting skipped).

#### 6.2 Generating a Software Bill of Materials (SBOM)

```yaml
# .github/workflows/security.yml (continued)
  sbom:
    runs-on: ubuntu-latest
    needs: [security-gate]
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM with Syft
        uses: anchore/sbom-action@v0
        with:
          image: medisetu/api:${{ github.sha }}
          format: cyclonedx-json
          output-file: medisetu-api-sbom.json

      - name: Upload SBOM as release artifact
        uses: actions/upload-artifact@v4
        with:
          name: sbom-${{ github.sha }}
          path: medisetu-api-sbom.json
```

> **📖 Why an SBOM Matters for a Healthtech Company**
>
> An SBOM is a complete, machine-readable inventory of every package, library, and version inside the deployed image — in this case in `cyclonedx-json` format, an industry-standard schema. When a new CVE is announced for, say, `openssl`, MediSetu can grep every stored SBOM for that package instead of re-scanning every historical release from scratch — answering "are we affected?" in minutes instead of days. For a healthtech company handling patient data, this is often a direct compliance requirement (HIPAA-equivalent frameworks and enterprise customer security questionnaires routinely ask for an SBOM), and it's exactly the kind of artifact that turns a frantic CVE-day scramble into a five-minute search.

**Quiz: A new CVE is announced today for a specific version of the `requests` Python library. MediSetu has shipped 40 different releases over the past year. What's the fastest way to find out which of those 40 releases are affected?**
- Re-clone and re-run pip-audit against each of the 40 releases' source code, one at a time
- Search the stored SBOM artifact from each release for the exact `requests` version and its known dependency tree
- Ask each engineer if they remember which version of `requests` was used in which release
- Assume all releases are affected and redeploy everything immediately

> **Answer/explanation:** The second option is correct and is precisely why Phase 6 generates and stores an SBOM for every build. Because each SBOM is a structured, searchable record of exact package versions at the time of that specific build, MediSetu can programmatically grep or query all 40 stored SBOMs for the vulnerable `requests` version in seconds, and get a precise, evidence-backed list of exactly which releases are affected — no guesswork, no re-scanning old code, no relying on anyone's memory. Re-running pip-audit against 40 historical checkouts would work eventually but is far slower and doesn't scale as the CVE count grows; asking engineers to remember is unreliable; and redeploying everything "just in case" wastes effort on releases that were never actually affected.

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Add IaC scanning:** MediSetu's Terraform provisions an S3-compatible bucket for storing prescription PDFs. Add `trivy config` (or Checkov) as a pipeline stage that scans the Terraform files for a publicly readable bucket policy before `terraform apply` ever runs.
2. **Tune false positives with a `.semgrepignore`:** Deliberately trigger a Semgrep false positive (e.g. a test fixture with a fake, clearly non-functional "password" string) and add a scoped suppression with a comment explaining why, rather than disabling the rule globally.
3. **Add a severity-based Slack alert:** Wire the `security-gate` job to post to a MediSetu Slack channel only when a CRITICAL finding is detected, so the on-call security engineer isn't paged for every LOW-severity noise finding.
4. **Rotate a secret end-to-end:** Deliberately commit a fake API key to a test branch, watch Gitleaks catch it in the PR, then practice the full incident response: revoke/rotate the real credential, rewrite Git history with `git filter-repo`, and force-push the cleaned branch.
5. **Build a vulnerability dashboard:** Pull the JSON output from Trivy, pip-audit, and Gitleaks across the last 10 pipeline runs into a simple script that produces a weekly trend chart of open findings by severity, so Sunita's leadership team can track MediSetu's security posture over time instead of only seeing pass/fail per PR.

### DevSecOps Project Complete 🎉

You built a five-layer automated security pipeline for MediSetu's telemedicine platform — static analysis catching insecure code patterns, dependency scanning catching known CVEs in third-party libraries, secrets scanning stopping credentials before they reach Git history, container image scanning blocking vulnerable base images from reaching the registry, and dynamic scanning attacking the live staging environment the way a real adversary would. All five now gate every single pull request through one required status check, and every build produces a searchable SBOM.

> **Sunita**
>
> "Three weeks ago, a leaked key sat undetected in our repo for six days. Last week, Gitleaks caught a test engineer's accidental commit of a staging database password within ninety seconds of the push, before the PR was even opened for review. That's not a marginal improvement — that's the entire risk model changing, from 'we hope someone notices' to 'the pipeline guarantees someone notices.'"

> **Rohit**
>
> "The SBOM step felt like overkill to you at first, I remember. Then last month's OpenSSL CVE dropped, and we answered our biggest hospital partner's 'are you affected?' email in four minutes by grepping our stored SBOMs, instead of the two-day fire drill it would have been six months ago."

> **Ananya (You)**
>
> "What changed my thinking most was realizing SAST, SCA, secrets scanning, and DAST aren't redundant — they're each blind to what the others catch. The SQL injection fix, the PyYAML CVE, the leaked key, the missing rate limit — four completely different bugs, and each one needed a different kind of scanner to ever be found automatically."

> **Next: Cloud Security & Infrastructure Hardening**

> - Infrastructure-as-Code scanning (Checkov, tfsec) — catch misconfigured cloud resources before `terraform apply` ever runs
> - Runtime security monitoring (Falco) — detect anomalous behavior inside running containers, not just at build time
> - Secrets management (HashiCorp Vault, AWS Secrets Manager) — stop secrets from living in environment variables and CI config at all
> - Policy-as-Code (OPA / Conftest) — enforce organization-wide security rules automatically across every repo, not just MediSetu's API
> - Zero Trust networking and mTLS between services — assume every network is hostile, even your own internal cluster network
