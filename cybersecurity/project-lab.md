# Cybersecurity Project Mastery

> **👋 Hey Fresher — Read This First!**

> Cybersecurity is the practice of protecting systems, networks, and applications from digital attacks. It's not just about "hacking" — most of the job is understanding how systems work, finding weaknesses before attackers do, and building defences. This document uses **short command blocks** — each one covers exactly one concept — with a plain-English explanation right beside it. No 50-line scripts to get lost in. One idea at a time, always explained in simple language.

> **Company in this project:** NovaPay — the same Mumbai fintech startup from the Kubernetes module. They've just moved to the cloud and their CISO has flagged serious security gaps. You join as a Junior Security Engineer. Your lead is Kavya. Let's audit and secure NovaPay's infrastructure from scratch.

#### What You Will Learn and Build in This Project

You will perform a full security audit of NovaPay's environment — learning every core security concept along the way, each explained with a real business reason.

Recon, Port Scanning, Web Vulns, SQLi & XSS, Linux Hardening, Firewalls, Encryption, Incident Response, OWASP Top 10

> **🔍 Phase 1 — Recon & Scanning**

> Learn how attackers gather information and how to find open ports and services before they do.

> Understand SQL Injection, XSS, and IDOR — the most common ways payment apps get breached.

> Harden the server OS, configure firewalls, and lock down SSH — the three pillars of server security.

> Protect data in transit with TLS, data at rest with encryption, and manage secrets the right way.

> Set up logging, detect anomalies, and follow a real incident response playbook.

**Scene 1 — NovaPay Security Review | "We've Been Exposed"**

> **Kavya** _Security Lead — NovaPay_
> 
> Our payment API was exposed on a public IP with port 22 open to the entire internet. No firewall rules. Last week a bot from Romania attempted 4,000 SSH login attempts in one hour. We caught it in the logs — but only by accident. We need a full security review starting today.

> **Arjun (You)** _Junior Security Engineer — Day 1 at NovaPay_
> 
> I know some Linux basics but I've never done a security audit. Where do I start?

> **Kavya** _Security Lead_
> 
> Start by thinking like an attacker. Before you can defend something, you need to know what's visible from the outside. What ports are open? What software versions are running? What URLs accept user input? Every answer is a potential entry point. We scan ourselves first so attackers don't get to do it first.

### 1. Phase 1 — Reconnaissance & Scanning

Reconnaissance ("recon") is the first step — understanding the attack surface. Before any attacker exploits a vulnerability, they scan to find what's running and where. As a defender, you run these same scans on your own systems regularly.

> **The Attacker's Mindset — Why You Must Think Like One**

> Security is about finding weaknesses before attackers do. That means actively scanning your own systems, probing your own APIs, and reading your own logs with suspicion. Every port you leave open unnecessarily is a door. Every input field that isn't validated is an unlocked window. Your job is to find and close those gaps — systematically, before someone else finds them.

```
NovaPay Attack Surface — What the Internet Can See
=====================================================

  INTERNET
     │
     ▼
  ┌──────────────────────────────────────────────────┐
  │  Public IP: 203.0.113.50                         │
  │                                                  │
  │  Port 22  → SSH (should be restricted!)          │
  │  Port 80  → HTTP (should redirect to HTTPS)      │
  │  Port 443 → HTTPS (payment API — intended)       │
  │  Port 3306 → MySQL (SHOULD NEVER BE PUBLIC!)     │
  └──────────────────────────────────────────────────┘
     │
     ▼
  Attacker sees Port 3306 open → tries default root password
  → gains database access → exports 2 lakh customer records
```

#### 1.1 Passive Recon — What Can You Find Without Touching the Target?

```
# Find subdomains using public DNS
nslookup novapay.in
nslookup api.novapay.in
```

**📖 What This Does**

**nslookup** queries DNS to find what IP addresses are behind a domain name. Attackers use this to discover subdomains like `staging.novapay.in` or `admin.novapay.in` — which are often less secured than the main site. Run this on your own domains first to see exactly what's publicly visible.

```bash
# Check what HTTP headers the server sends
curl -I https://api.novapay.in
```

**📖 Why Headers Matter**

**curl -I** fetches only the HTTP headers — not the page body. Headers often reveal the server software version (like `Server: Apache/2.4.1`), which tells an attacker exactly which known CVEs apply. You should suppress or randomize server version headers in production.

```
HTTP/2 200
server: nginx/1.18.0
x-powered-by: Express
content-type: application/json

⚠️  Problem: "server: nginx/1.18.0" tells attackers the exact version.
    Solution: Set "server_tokens off;" in nginx config.
```

#### 1.2 Port Scanning — What's Open on the Server?

```
# Scan the 1000 most common ports
nmap 203.0.113.50
```

**📖 nmap Basics**

**nmap** is the industry-standard port scanner. It sends packets to each port and checks if something responds. A "open" result means a service is listening there. Run this against your own servers to see what's exposed. Every open port that isn't needed is a risk — close it.

```
# Detect service versions on open ports
nmap -sV 203.0.113.50
```

**📖 Version Detection**

**-sV** tells nmap to probe each open port and identify the service and version. Output like `3306/tcp open mysql MySQL 5.7.38` tells you MySQL is public-facing — which it should never be. If your database port is open to the internet, that's a critical vulnerability.

```
PORT     STATE  SERVICE   VERSION
22/tcp   open   ssh       OpenSSH 8.2
80/tcp   open   http      nginx 1.18.0
443/tcp  open   https     nginx 1.18.0
3306/tcp open   mysql     MySQL 5.7.38   ← CRITICAL: Database is public!
```

> **🚨 What NovaPay Must Fix Immediately**

> Port 3306 (MySQL) is open to the internet. This means anyone on earth can attempt to connect to NovaPay's database directly. The fix: add a firewall rule to block port 3306 from all external IPs. Databases should only accept connections from application servers inside the same private network — never from the public internet.

> **💡 Fresher Tip — Only Scan Systems You Own or Have Permission to Scan**

> Running nmap against a system you don't own is illegal in most countries. In a job, you'll be given written permission (a "scope of engagement") before scanning. In practice labs, use dedicated environments like HackTheBox, TryHackMe, or your own local VMs. Always get explicit written permission before scanning any system.

### 2. Phase 2 — Web Application Vulnerabilities

**Business Problem:** NovaPay's payment API accepts user input — payment amounts, user IDs, search queries. Every input field is a potential attack surface. The OWASP Top 10 lists the most common and critical web application vulnerabilities. As a junior security engineer, you must know the top three cold.

**Scene 2 — NovaPay Code Review | "The Login Page Has a Hole"**

> **Kavya** _Security Lead_
> 
> Look at this login endpoint. It takes the username directly from the form and puts it into the SQL query with string concatenation. That's textbook SQL Injection. An attacker types "admin' --" as their username and the query becomes "SELECT * FROM users WHERE username='admin'--" — the "--" comments out the password check. They're in as admin with no password.

> **Arjun (You)** _Junior Security Engineer_
> 
> So the fix is to never build SQL queries by joining strings?

> **Kavya** _Security Lead_
> 
> Exactly. Always use parameterized queries — also called prepared statements. The database driver separates the data from the SQL code, so user input can never be interpreted as SQL commands. That's the golden rule of SQL Injection prevention.

#### 2.1 SQL Injection (SQLi) — The #1 Web Attack

```
# ❌ VULNERABLE: user input in SQL string
query = "SELECT * FROM users
  WHERE user='" + username + "'"
```

**📖 Why This Is Dangerous**

When `username` is `admin' --`, the query becomes: `WHERE user='admin'--'`. The `--` comments out the rest — including the password check. The attacker logs in as admin with any password. This is SQL Injection.

```
# ✅ SAFE: parameterized query
query = "SELECT * FROM users
  WHERE user = ?"
db.execute(query, [username])
```

**📖 Why This Is Safe**

The `?` is a placeholder. The database driver sends the SQL and the data *separately*. Even if the user types SQL commands, they're treated as plain text data — never as executable code. This is called a **parameterized query** or **prepared statement**.

🧪. How to Test for SQL Injection (on your own app)
In any login form or search box, type a single quote `'` and submit. If the app shows a database error like `syntax error near ''''`, it's vulnerable — the quote broke the SQL query. A safe app should show a validation error, not a database error.

#### 2.2 Cross-Site Scripting (XSS) — Injecting Code Into the Browser

```
# ❌ VULNERABLE: renders raw HTML
page.innerHTML = userComment
# Attacker submits this as a comment:
<script>document.cookie</script>
```

**📖 What XSS Does**

The attacker stores a `<script>` tag as a comment. When other users view the page, that script runs in their browser — it can steal their session cookies, redirect them to a fake login page, or make API calls on their behalf. This is how many banking frauds start.

```
# ✅ SAFE: escape HTML before rendering
page.textContent = userComment
# OR sanitize with a library
DOMPurify.sanitize(userComment)
```

**📖 The Fix**

**textContent** treats everything as plain text — it never interprets HTML tags. **DOMPurify** is a library that strips dangerous tags while keeping safe ones like `<b>`. Use `textContent` when no HTML is needed; use DOMPurify when you need to allow some formatting.

#### 2.3 Insecure Direct Object Reference (IDOR) — Accessing Other Users' Data

```
# ❌ VULNERABLE: no ownership check
GET /api/transaction/1042
GET /api/transaction/1043  ← another user's!
```

**📖 What IDOR Is**

The API returns transaction 1043 to anyone who asks — even if it belongs to a different user. An attacker loops through IDs (1000, 1001, 1002...) and reads all transactions in the system. This is called an **IDOR** — Insecure Direct Object Reference.

```
# ✅ SAFE: always check ownership
if transaction.user_id != current_user.id:
    return 403 Forbidden
```

**📖 The Fix**

Before returning any resource, check that the logged-in user owns it. Return **403 Forbidden** if they don't — not 404, because 404 reveals the object exists. This check must happen server-side — never trust the client to enforce access control.

> **💡 Fresher Tip — OWASP Top 10**

> OWASP (Open Worldwide Application Security Project) publishes the most critical web security risks as the "OWASP Top 10." As a fresher, memorize the top 3: **Injection** (SQLi, command injection), **Broken Access Control** (IDOR), and **XSS**. Every interview for a security or backend role will ask about at least one of them. Visit `owasp.org/Top10` to read the full list with real examples.

### 3. Phase 3 — Linux & Network Hardening

**Business Problem:** NovaPay's cloud server has a default configuration — meaning every service that Ubuntu installed by default is still running, SSH accepts password logins from any IP, and the firewall allows everything. Default configurations are attacker-friendly. Hardening means turning off everything you don't need and locking down everything that must stay on.

**Scene 3 — NovaPay Server Audit | "Hardening Begins"**

> **Kavya** _Security Lead_
> 
> The server has 14 unnecessary services running — including a print spooler. We're a payment API, not a printer. Every running service is code that can have bugs, can be exploited. The principle of minimal attack surface: if you don't need it, remove it. Let's start with SSH, firewall rules, and user accounts.

#### 3.1 SSH Hardening — Lock the Front Door

```
# Disable password login — keys only
PasswordAuthentication no
# in /etc/ssh/sshd_config
```

**📖 Why Keys Instead of Passwords**

SSH passwords can be brute-forced — attackers try millions of common passwords automatically. SSH keys are mathematically impossible to brute-force with current hardware. Disabling password login means even if an attacker knows your username, they can't get in without the private key file on their machine.

```
# Disable root login over SSH
PermitRootLogin no
# in /etc/ssh/sshd_config
```

**📖 Why Block Root SSH**

The root account exists on every Linux machine with the same name. If root can log in via SSH, every attacker in the world knows the username — they just need the password. Disable root SSH and use a normal user account with `sudo` for admin tasks. This alone stops a huge class of attacks.

```
# Restart SSH to apply changes
sudo systemctl restart sshd

# Verify the changes took effect
sudo sshd -T | grep -E "passwordauth|permitroot"
```

> **systemctl restart sshd** reloads the SSH daemon with the new config. Always run **sshd -T** after editing the config — it prints the effective settings and catches typos before you get locked out. Keep your current SSH session open while testing in a second session, so you're never locked out of your own server.

#### 3.2 Firewall Rules with UFW — Allow Only What You Need

```
# Start with deny all
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

**📖 Default Deny — The Right Starting Point**

By default, deny all incoming connections. Then explicitly allow only the ports you need. This is the **allowlist** (whitelist) model — everything is blocked unless you say otherwise. The opposite (allow all, block known bad) is called blocklisting and is far weaker.

```
# Allow only what NovaPay needs
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow from 10.0.0.0/8 to any port 22
sudo ufw enable
```

**📖 Restrict SSH by IP Range**

Port 443 (HTTPS) must be open to everyone — it's the payment API. But SSH (port 22) should only be open from your office VPN or internal IP range. `from 10.0.0.0/8` means "only from private IP addresses" — the entire public internet is blocked from SSH. This stops 99% of automated SSH attacks.

#### 3.3 User Account Hardening — Least Privilege

```
# List all users with login shells
cat /etc/passwd | grep "/bin/bash"
```

**📖 Find Unnecessary Accounts**

Every account that can log in is a potential entry point. Run this to see all human-login accounts. Old employee accounts, test accounts, default accounts — remove them. The fewer accounts that exist, the fewer ways in for an attacker.

```
# Lock a user account immediately
sudo passwd -l oldemployee
```

**📖 Locking vs Deleting**

**passwd -l** locks the account instantly — the user can no longer log in. Use this when an employee leaves NovaPay. It's faster and safer than deleting, because it preserves their files and audit history. After confirming nothing is needed, delete with `userdel -r`.

### 4. Phase 4 — Encryption & Secrets Management

**Business Problem:** NovaPay processes UPI payments. Every transaction contains account numbers, amounts, and user identities. If data travels unencrypted, anyone on the network path can read it. If secrets like database passwords are stored in environment variables on the server, a single compromised process exposes the entire database.

**Scene 4 — NovaPay Security Review | "Our API Was on HTTP"**

> **Kavya** _Security Lead_
> 
> The mobile app was calling the payment API over plain HTTP on the staging server. The developer said "it's just staging." But staging uses real-looking test data. And we found the staging API key is identical to the production key. Someone on any coffee shop WiFi could have intercepted those calls and seen both the key and the request payload.

#### 4.1 TLS — Encrypting Data in Transit

```
# Get a free TLS certificate
sudo certbot --nginx -d api.novapay.in
```

**📖 What TLS Does**

TLS (Transport Layer Security) encrypts all data between the client and server. Without it, anyone on the same network — a coffee shop WiFi, an ISP, a compromised router — can read every byte transmitted. **Certbot** from Let's Encrypt gets a free, trusted certificate in one command and configures nginx automatically.

```bash
# Verify TLS is working correctly
curl -I https://api.novapay.in
openssl s_client -connect api.novapay.in:443
```

**📖 Testing TLS**

**curl -I** should return HTTP/2 200, not a certificate error. **openssl s_client** shows the full TLS handshake — you can see the certificate chain, expiry date, and cipher suite. Use this to verify TLS is working and the certificate is valid before going to production.

#### 4.2 Hashing Passwords — Never Store Plain Text

```
# ❌ NEVER store passwords like this
db.save(user, password="secret123")
```

**📖 The Problem**

If the database is ever leaked — which happens even to major companies — every user's plain-text password is exposed. An attacker can log into every account immediately and also try those passwords on Gmail, banking apps, and other sites (password reuse attacks).

```
# ✅ Hash with bcrypt before storing
hashed = bcrypt.hash(password, salt=12)
db.save(user, password=hashed)
```

**📖 Why bcrypt**

**bcrypt** is a slow hashing algorithm designed for passwords. The **salt=12** (also called cost factor) makes it intentionally slow — 100ms per hash. This is fine for login (users don't notice 100ms) but makes brute-force attacks take centuries. Never use MD5 or SHA-256 directly — they're too fast for passwords.

#### 4.3 Environment Variables & Secret Files — Don't Hardcode Credentials

```
# ❌ NEVER hardcode secrets in code
DB_PASS = "novapay_prod_2024!"
```

**📖 Why This Is Dangerous**

If this code is pushed to GitHub — even a private repo — the secret is in git history forever. Even after you delete it, anyone with access to the old commits can find it. Secrets in code are the single most common source of credential leaks.

```python
# ✅ Read from environment at runtime
import os
DB_PASS = os.environ["DB_PASSWORD"]
```

**📖 The Safe Pattern**

The actual secret lives in the environment (set by a secrets manager, a Kubernetes Secret, or a `.env` file that is in `.gitignore`). The code only reads from the environment — the source code itself has no secrets. This way, the same codebase works in dev and prod with different secrets, and the repo is safe to share.

> **💡 Fresher Tip — Check Your .gitignore**

> Before every git commit, run `git status` and make sure `.env`, `*.pem`, `*.key`, and any config files with passwords are listed in `.gitignore`. A tool called **git-secrets** or **truffleHog** can scan your repo history for accidentally committed secrets. Many companies also use **GitHub Secret Scanning** — it alerts you automatically when a secret-like pattern is pushed.

### 5. Phase 5 — Monitoring & Incident Response

**Business Problem:** Security is not just about preventing attacks — it's about detecting them when they happen and responding fast. A breach that is detected in 10 minutes and contained costs a fraction of one that isn't noticed for 3 months. NovaPay needs logs, alerts, and a clear playbook for when something goes wrong.

**Scene 5 — NovaPay 2 AM | "Something Is Wrong With the Logs"**

> **Kavya** _Security Lead_
> 
> Arjun, look at the auth logs. 3,000 failed login attempts on the /api/login endpoint in the last 10 minutes, all from the same IP range. And then at 1:47 AM, one successful login from the same range — but we have no customer from that IP geolocation. That's a credential stuffing attack. They used a leaked password list and hit one valid credential. We need to block that IP range and force a password reset for that account right now.

#### 5.1 Reading System Logs — Your First Line of Detection

```
# Watch SSH login attempts live
sudo tail -f /var/log/auth.log
```

**📖 auth.log**

Every SSH login attempt — success or failure — is written to `auth.log`. The **-f** flag "follows" the file in real time. You'll see lines like `Failed password for root from 185.x.x.x`. Many such lines from different IPs in minutes = automated brute-force attack in progress.

```
# Count failed SSH attempts by IP
grep "Failed password" /var/log/auth.log \
  | awk '{print $11}' | sort | uniq -c | sort -rn
```

**📖 Finding the Attacker's IP**

This one-liner counts how many times each IP address had a failed SSH login. The output shows the worst offenders at the top — an IP with 500+ failures in the log is definitely a bot running a brute-force attack. This is the first command a security engineer runs during an SSH attack.

#### 5.2 Blocking an Attacker with fail2ban

```
# Install and enable fail2ban
sudo apt install fail2ban
sudo systemctl enable --now fail2ban
```

**📖 What fail2ban Does**

**fail2ban** watches log files and automatically adds firewall rules to ban IPs that have too many failed login attempts. After 5 failures in 10 minutes, the attacker's IP is banned for 1 hour. This happens without any manual intervention — it's automated defence that runs 24/7.

```
# Manually ban an IP right now
sudo fail2ban-client set sshd banip 185.x.x.x
```

**📖 Manual Ban**

During an active attack you identified from logs, you can ban an IP immediately without waiting for the automatic threshold. This creates a firewall rule that drops all packets from that IP. Use **fail2ban-client status sshd** to see all currently banned IPs.

#### 5.3 Incident Response — The 5-Step Playbook

1. Detect — Know something happened
Alert from monitoring, customer report, log anomaly, or automated tool. Time of detection matters — every minute counts. Assign an incident owner immediately.

2. Contain — Stop the bleeding
Isolate affected systems. Block attacker IPs. Revoke compromised credentials immediately. Do not delete logs — you'll need them for investigation.

3. Investigate — Understand what happened
Review logs to find the initial entry point. What vulnerability was exploited? What data was accessed? What time did the breach start? Build a full timeline.

4. Remediate — Fix the root cause
Patch the vulnerability, change all potentially compromised credentials, rotate API keys. Harden the system so the same attack cannot work again.

5. Report & Learn — Don't repeat it
Write an incident report (what happened, how long, what data affected, what was fixed). Share lessons with the team. Update runbooks. Comply with GDPR/RBI notification requirements if customer data was affected.

### 6. Essential Security Commands — Quick Reference

Command

What It Does

When to Use It

nmap -sV <ip>

Scan open ports and detect service versions

First step of any security audit on your own server

curl -I https://<url>

Fetch HTTP headers only

Check what server version is exposed in headers

sudo ufw status verbose

Show all active firewall rules

Verify which ports are open and to whom

sudo ufw deny <port>/tcp

Block a port at the firewall level

Close a port you don't need open to the internet

grep "Failed password" /var/log/auth.log

Show all failed SSH login attempts

During or after a suspected brute-force attack

sudo last

Show recent login history for all users

Check if any unexpected logins occurred

sudo netstat -tlnp

List all listening ports and the process using them

Find unexpected services running on your server

openssl s_client -connect <host>:443

Inspect the TLS certificate and handshake

Verify TLS is configured correctly before going live

sudo passwd -l <user>

Lock a user account instantly

When an employee leaves or an account is compromised

git log --all --full-history -- "*.env"

Check if a .env file was ever committed to git

Before open-sourcing a repo or sharing it

### 7. Interview Questions — Cybersecurity

##### Interview Q&A — Fresher Level (0–1 Year Security Experience)

**Q: Q1. What is the difference between authentication and authorization?**

A: Authentication is proving who you are — it answers "Are you really Arjun?" It's handled by login systems: passwords, OTPs, biometrics, SSH keys. Authorization is deciding what you're allowed to do — it answers "Is Arjun allowed to access this payment record?" These are two separate systems. A bug in authentication means an attacker can log in as someone else. A bug in authorization (like an IDOR vulnerability) means a legitimately logged-in user can access data that isn't theirs. Both must be implemented correctly for a system to be secure.

**Q: Q2. What is SQL Injection and how do you prevent it?**

A: SQL Injection is when an attacker inserts SQL commands into a form field or URL parameter, and the application executes that SQL as part of a database query. For example, entering "admin'--" as a username can bypass a login query if the app builds queries by string concatenation. Prevention: always use parameterized queries (also called prepared statements), where the SQL code and the user data are sent to the database separately, so user input can never be interpreted as SQL. Secondary defences include input validation, least-privilege database accounts, and a WAF (Web Application Firewall).

**Q: Q3. What is the principle of least privilege?**

A: Least privilege means every user, process, and service should have the minimum permissions needed to do its job — and nothing more. NovaPay's payment API needs to read and write to the payments table, so its database account should only have SELECT and INSERT on that specific table — not DROP TABLE, not access to the users table, not GRANT. If the API is compromised, the attacker inherits only those limited permissions. Applying least privilege slows down attackers and limits the blast radius of any breach. It applies to everything: OS users, database accounts, API keys, Kubernetes service accounts, and cloud IAM roles.

**Q: Q4. What is the difference between symmetric and asymmetric encryption?**

A: Symmetric encryption uses the same key to encrypt and decrypt data. It's fast and efficient — used for encrypting data at rest (entire disk, database fields). The challenge: how do you securely share the key with the other party? Asymmetric encryption uses a key pair — a public key (shareable with anyone) and a private key (never shared). Data encrypted with the public key can only be decrypted with the private key. TLS uses asymmetric encryption during the handshake to securely exchange a temporary symmetric key, then switches to symmetric encryption for the actual data transfer (because symmetric is much faster). HTTPS depends on this combination.

**Q: Q5. What would you do if you suspect a server has been compromised?**

A: First, don't panic and don't immediately shut down the server — you might destroy forensic evidence. Follow the incident response playbook: (1) Detect and confirm — check logs for unusual activity, unexpected logins, new user accounts, modified files. (2) Contain — isolate the server from the network if possible, revoke all credentials associated with it, block suspicious IPs. (3) Investigate — review auth.log, application logs, file modification timestamps (stat, find -newer), and running processes (ps aux, netstat). Look for signs of persistence like cron jobs, new SSH authorized_keys, or unfamiliar binaries. (4) Remediate — restore from a clean backup or rebuild fresh, patch the vulnerability, rotate all credentials. (5) Report — document the timeline, scope, and remediation steps.

**Q: Q6. What is a CVE?**

A: CVE stands for Common Vulnerabilities and Exposures — it's the global standard identifier for publicly disclosed security vulnerabilities. Each vulnerability gets a unique ID like CVE-2024-1234. When you hear "patch for CVE-2021-44228" (Log4Shell), that's a reference to a specific, documented vulnerability in a specific software version. As a security engineer, you track CVEs for all software your company runs. Tools like Trivy, Snyk, and Dependabot automatically scan your code and containers for known CVEs and alert you when a critical one is found. The CVSS score (0–10) rates severity — anything above 9.0 is Critical and needs immediate patching.

**Quiz: Quiz 1 — A developer commits a database password to a public GitHub repo by mistake. What are the first two actions you take?**

- A) Delete the commit and push again — the secret is gone now
- B) Immediately rotate (change) the database password, then remove the secret from git history using git filter-branch or BFG Repo Cleaner
- C) Make the repository private — that's enough
- D) Wait to see if anyone uses the leaked credential before acting

> **Answer/explanation:** ✅ Answer: B. Rotating the credential is the first action because GitHub indexes public repos within minutes — bots scan for leaked secrets continuously. By the time you notice, the secret has already been found. Making the repo private or deleting the commit does not help — the secret is in git history and was already indexed. After rotating, remove it from history with BFG Repo Cleaner or `git filter-repo`. Going forward, add `.env` to `.gitignore` and enable GitHub Secret Scanning on all repos.

**Quiz: Quiz 2 — What is the difference between a vulnerability scan and a penetration test?**

- A) They are the same thing — both involve running nmap
- B) A vulnerability scan automatically finds known weaknesses in software versions; a penetration test involves a human tester actively trying to exploit those weaknesses to demonstrate real-world impact
- C) Penetration tests are illegal; vulnerability scans are not
- D) Vulnerability scans are done by external companies only; penetration tests are done internally

> **Answer/explanation:** ✅ Answer: B. A vulnerability scan (using tools like Nessus, OpenVAS, or Trivy) is automated — it checks software versions against databases of known CVEs and reports findings. It's fast and cheap but can't chain vulnerabilities together or understand business context. A penetration test (pentest) has a skilled human tester actively trying to break into the system — they think creatively, combine multiple weaknesses, and demonstrate real-world attack paths. Companies typically run vulnerability scans continuously and full pentests annually or before major product launches. RBI mandates annual pentests for payment processors in India.

**Quiz: Quiz 3 — A user reports they see other customers' transaction history on the NovaPay app. What type of vulnerability is this and where is the fix?**

- A) SQL Injection — the fix is to sanitize the database query
- B) XSS — the fix is to escape HTML output
- C) IDOR (Insecure Direct Object Reference) — the fix is a server-side ownership check before returning any resource
- D) CSRF — the fix is to add CSRF tokens to forms

> **Answer/explanation:** ✅ Answer: C. When a user can access another user's data by simply changing an ID in the URL or API call, that's an IDOR vulnerability. The root cause is the server returning a resource without checking that the requesting user owns it. The fix is always server-side — add a check like `if transaction.user_id != current_user.id: return 403` before returning any data. Client-side fixes (hiding the button, not showing the ID) don't count — attackers call the API directly, bypassing any UI restrictions entirely.

> **Cybersecurity Project — Core Takeaways for Freshers**

> - Think like an attacker first — before defending, understand what an attacker sees. Run nmap against your own servers, review your own headers, read your own logs. You cannot defend what you haven't examined.
> - Never trust user input — every form field, URL parameter, and API body is a potential injection point. Validate and sanitize all input server-side. Never build SQL queries by string concatenation. Always use parameterized queries.
> - Least privilege everywhere — database accounts, OS users, API keys, cloud roles. Every account should have exactly the permissions it needs and no more. This limits the damage when (not if) something is compromised.
> - Secrets never belong in code — no passwords, API keys, or certificates in source code. Use environment variables, Kubernetes Secrets, or a dedicated secrets manager. Add .env and *.key to .gitignore on day one.
> - Default configurations are attacker-friendly — turn off services you don't need, disable password SSH, set your firewall to deny-all by default. Hardening is the process of removing everything unnecessary.
> - Logs are your security camera — if you're not collecting and reviewing logs, you're flying blind. Enable logging, centralize logs, and set up alerts for high-failure-rate patterns. You can only respond to what you detect.

##### Security Standards — NovaPay Payment Platform Rules

- All APIs must use HTTPS only — no exceptions, including internal staging environments. HTTP must redirect to HTTPS. TLS certificates must be renewed before expiry (Certbot auto-renews, but verify the cron job).
- Rotate all credentials every 90 days — database passwords, API keys, JWT signing secrets. Immediately rotate any credential that may have been exposed, without waiting for the rotation schedule.
- Every new feature that handles user data must go through a security review before merging — check for IDOR, injection, missing authentication, and insecure direct database access from the frontend.
- Use MFA (Multi-Factor Authentication) for all employee accounts — cloud console access, GitHub, admin dashboards, and the VPN. Passwords alone are not sufficient for admin-level access.
- Run dependency scans (Snyk, Dependabot) on every code repository — attackers frequently exploit outdated npm or pip packages with known CVEs, not custom zero-days.
- Store all security scripts, firewall rules, and hardening configs in Git — infrastructure security changes go through code review just like application code. Undocumented manual changes to servers are a security risk.

##### 🏋️ Hands-On Exercises — Practice in a Safe Environment

1. **Set up a vulnerable app and exploit it:** Install DVWA (Damn Vulnerable Web Application) using Docker (`docker run -p 80:80 vulnerables/web-dvwa`). Practice SQL Injection on the login form and XSS in the guestbook. Then read the source code to understand exactly why each attack worked and what the fix looks like.
2. **Harden an Ubuntu VM:** Spin up an Ubuntu VM on VirtualBox or on a free-tier AWS EC2. Disable SSH password auth, set up UFW with deny-all-incoming, remove unnecessary packages (`apt list --installed`), and install fail2ban. Then run nmap against it from another machine and verify only port 443 is accessible.
3. **Find a leaked secret in git history:** Create a test repo, commit a fake API key (`API_KEY=test123`) in a .env file, then delete it and commit again. Run `git log --all -p -- .env` to prove the secret is still in history. Then use BFG Repo Cleaner to remove it and verify it's gone.
4. **Analyze a real CVE:** Look up CVE-2021-44228 (Log4Shell) on the NVD website. Read the description, CVSS score, and affected versions. Find which version of Log4j fixes it. Practice explaining it to a non-technical person in 2 sentences — this is a common interview exercise.
5. **Set up log monitoring:** On your Ubuntu VM, install fail2ban and configure it to protect SSH with a ban threshold of 3 failures. Then deliberately fail 3 SSH logins from another terminal and run `sudo fail2ban-client status sshd` to confirm your IP is banned. Then unban it with `fail2ban-client set sshd unbanip <your-ip>`.

### Cybersecurity Project Complete 🎉

You have completed NovaPay's full security audit — reconnaissance and port scanning, SQL Injection and XSS defence, Linux and SSH hardening, firewall configuration, TLS setup, secrets management, log analysis, and a complete incident response playbook. You now speak the language of every security team in the industry.

> **Kavya**
> 
> "Arjun, when you joined we had a public database port, SSH accepting passwords from any IP, and API keys sitting in git history. Last week's external pentest came back: zero critical findings. The pentester specifically called out our parameterized queries, fail2ban configuration, and TLS setup as best-in-class. That's the work you did. Security is never done — but NovaPay is now a much harder target than it was."

> **Roshan**
> 
> "The mental model shift you made — from 'I hope nobody attacks us' to 'I assume someone is always trying, so I layer my defences' — that's the security engineering mindset. Assume breach. Minimize blast radius. Detect fast. Respond faster. Once that clicks, every security decision becomes obvious."

> **Next: Advanced Security — Cloud Security, SIEM, Threat Modelling & DevSecOps**

> - AWS/GCP IAM — Identity and Access Management: roles, policies, service accounts, and the principle of least privilege in the cloud
> - SIEM — Security Information and Event Management: centralize logs from all sources (CloudWatch, nginx, app logs) and set up automated alerts for anomalies
> - Threat Modelling — STRIDE methodology: systematically identify Spoofing, Tampering, Repudiation, Information Disclosure, DoS, and Elevation of Privilege risks in your architecture
> - DevSecOps — shift security left: SAST (Static Application Security Testing) in CI/CD pipelines so every pull request is automatically scanned for vulnerabilities before merge
> - Container Security — scan Docker images for CVEs with Trivy, run containers as non-root, use read-only filesystems, and apply Kubernetes Pod Security Standards
> - Zero Trust Architecture — never trust, always verify: even internal services must authenticate each other; no implicit trust based on network location
