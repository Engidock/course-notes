# Linux Project Mastery

> **👋 Hey Fresher — Read This First!**

> - Linux powers **96% of the world's servers**, all cloud infrastructure (AWS EC2, Azure VMs, GCP), all Docker containers, and all Kubernetes nodes. If you work in DevOps, cloud, or backend engineering, you will use Linux commands every single working day.
> - Linux knowledge is not optional for DevOps engineers — it is the **foundation under every other tool**: Kubernetes runs on Linux, Docker runs on Linux, Ansible configures Linux servers, Jenkins runs on Linux, every cloud VM is Linux.
> - Every command block in this document is short and focused. Every explanation is bullet points. Terminal output previews show you what each command actually displays.
> - **Company in this project:** TechAxis Solutions — a DevOps consulting firm in Bengaluru managing 60+ Linux servers for clients across banking, e-commerce, and SaaS. You just joined as a Junior DevOps Engineer. Your lead is Harish. You will manage real Ubuntu 22.04 servers from the command line — configuring services, managing users, writing scripts, and diagnosing production issues.

📦 Where Linux Lives in a Real DevOps / Cloud Stack

Ansible Playbook → SSH to Linux Server → apt / systemctl / chmod (Linux) → Configured Server

Docker Build → Dockerfile = Linux commands → Container (Linux kernel) → Kubernetes Pod

Jenkins / GitHub Actions → Shell Script (Linux) → Build → Test → Deploy

EC2 / Azure VM / GCP Instance → Ubuntu / Amazon Linux (Linux) → nginx / postgres / java running

Linux is the OS on which every tool in DevOps runs. Ansible runs shell commands on Linux. Docker containers are Linux processes. Kubernetes nodes are Linux VMs. CI/CD pipelines are shell scripts on Linux agents. Master Linux and you understand what every tool above it is actually doing.

#### What You Will Learn in This Project

> **📁 Phase 1 — File System & Navigation**

> Navigate the Linux directory structure, list files, create and delete directories, understand absolute vs relative paths — the foundation of every Linux session.

> Read files with cat/less/tail, search with grep, find files with find, edit with nano and vim. Used daily for reading logs and config files.

> Understand rwx permission bits, change ownership with chown, manage users and groups, use sudo — the security foundation every DevOps engineer must master.

> Monitor processes with ps and top, manage systemd services with systemctl, check ports with ss, trace network with curl and ping — production troubleshooting tools.

> Write bash scripts with variables, loops, and conditionals. Schedule jobs with cron. Search and tail logs with grep and journalctl. Automate everything.

Navigation, Permissions, grep / find, systemctl, Shell Script, cron, Users, Networking

**Scene 1 — TechAxis Solutions, Bengaluru | First Day on Production**

> **Harish** _Senior DevOps Engineer — TechAxis Solutions_
> 
> Deepak, welcome. Your first task today: a client's nginx web server is not serving requests. No GUI, no remote desktop. You get an SSH terminal and that's it. Everything — diagnosing the problem, fixing it, restarting the service — happens through Linux commands. This is every production incident: one terminal, one engineer, one broken server. Linux is not a nice-to-have for DevOps. It is the only interface that works at 2 AM on a crashed server.

> **Deepak (You)** _Junior DevOps Engineer — Day 1 at TechAxis_
> 
> I've used Linux GUI (Ubuntu desktop) but I'm not comfortable with the terminal. Where do I even start when I SSH into a server?

> **Harish** _Senior DevOps Engineer_
> 
> Three questions every time you SSH into a server: WHERE am I (pwd), WHAT is here (ls -la), and WHO am I (whoami). Then: is the service running (systemctl status nginx), what does the log say (tail /var/log/nginx/error.log), and what is using the port (ss -tlnp). Six commands answer 80% of production incidents. We'll learn all of them today, one at a time, with the actual output each command gives you.

> **Preetam** _Infrastructure Lead — TechAxis Solutions_
> 
> And never Google "how to delete all files in Linux" and run the first result. That is how people run rm -rf / on production. Every command we teach you today comes with "why" and "when NOT to run this." Linux commands are not dangerous — running them without understanding them is.

### 1. Phase 1 — File System and Navigation

🔧 Where File Navigation Is Used in the DevOps Stack

SSH into AWS EC2 / Azure VM → cd / ls / pwd (Linux navigation) → Find nginx config, app logs, ssl certs

Dockerfile (RUN commands) → mkdir / cp / mv / rm (Linux file ops) → Sets up container file system

Every Dockerfile, Ansible task, and CI/CD pipeline step that copies files, changes directories, or deletes build artifacts uses these exact Linux file commands.

> **The Linux Directory Structure — Know These Paths**

> - **/** — root of the entire file system. Everything starts here.
> - **/etc** — configuration files. nginx config, SSH config, cron jobs, hosts file — all here.
> - **/var/log** — log files. nginx logs, system logs, application logs — always check here first during incidents.
> - **/home/username** — each user's personal directory. Your files, scripts, and SSH keys live here.
> - **/usr/bin** — most installed programs (ls, grep, curl, python3). When you install a package, its binary goes here.
> - **/tmp** — temporary files. Cleared on reboot. Never store important data here.
> - **/opt** — manually installed software (Jenkins, custom apps). Separate from system packages.
> - **/proc** — virtual filesystem showing running processes and kernel state. `/proc/cpuinfo`, `/proc/meminfo` — read CPU and RAM info from here.

#### 1.1 Navigation — The Three Commands You Always Run First

```
# WHERE am I?
pwd
# WHAT is here? (detailed listing)
ls -la
# WHO am I?
whoami
# Go to a directory
cd /etc/nginx
# Go up one level
cd ..
# Go to your home directory
cd ~
```

**📖 Navigation Commands**

- **pwd** (print working directory) — shows your current location in the file system. Run this whenever you're lost.
- **ls -la** — `-l` = long format (permissions, size, date), `-a` = show hidden files (names starting with `.`). Always use `ls -la` not just `ls`.
- **whoami** — shows your current username. Critical — running commands as root vs a regular user has very different consequences.
- **cd /absolute/path** — go to exact path from root. Always works.
- **cd ../relative** — navigate relative to current location. `..` = parent directory, `.` = current directory.
- **cd ~** — go to your home directory. Shortcut: just type `cd` with no arguments.

deepak@techaxis-server:~$ pwd /home/deepak deepak@techaxis-server:~$ ls -la total 48 drwxr-xr-x 6 deepak deepak 4096 Mar 29 09:15 . drwxr-xr-x 12 root root 4096 Mar 28 22:00 .. -rw------- 1 deepak deepak 220 Mar 28 22:00 .bash_history -rw-r--r-- 1 deepak deepak 3771 Mar 28 22:00 .bashrc drwx------ 2 deepak deepak 4096 Mar 29 09:10 .ssh -rwxr-xr-x 1 deepak deepak 512 Mar 29 09:00 deploy.sh deepak@techaxis-server:~$ whoami deepak

#### 1.2 Creating and Deleting Files and Directories

```
# Create a directory
mkdir app
# Create nested directories at once
mkdir -p /opt/techaxis/logs/2026
# Create an empty file
touch deploy.log
# Copy a file
cp nginx.conf nginx.conf.bak
# Move / rename
mv old-app.jar app.jar
# Delete a file (no recycle bin!)
rm deploy.log
# Delete a directory and everything inside
rm -rf /opt/old-app
```

**📖 File Operations**

- **mkdir -p** — creates parent directories automatically. Without `-p`, mkdir fails if parent doesn't exist.
- **touch** — creates an empty file, or updates the timestamp if it already exists. Used to create placeholder files and trigger watchers.
- **cp file dest** — copy file. `cp -r dir/ dest/` to copy a whole directory recursively.
- **mv** — moves (or renames) a file. Unlike cp, the original is gone.
- **rm -rf** — deletes a directory and ALL contents recursively, with no confirmation. **There is no undo.** Double-check the path before pressing Enter. Never run `rm -rf /` or `rm -rf /*`.

### 2. Phase 2 — Viewing, Searching and Editing Files

**Business Problem:** The TechAxis client's application crashed. The first thing you do is read the log file to find the error. Knowing how to quickly search a 50,000-line log file for a specific error is the difference between a 5-minute fix and a 2-hour investigation.

🔧 Where File Reading / Search Is Used in the DevOps Stack

Production incident at 2 AM → tail -f /var/log/nginx/error.log → Find the error in real time

Ansible task: update nginx config → vim /etc/nginx/nginx.conf → Edit config, reload service

CI/CD pipeline build failure → grep "ERROR" build.log → Find root cause in build output

#### 2.1 Reading Files

```
# Print entire file (small files)
cat /etc/hostname
# Read large files page-by-page (q to quit)
less /var/log/syslog
# Show last 20 lines of a log
tail -n 20 /var/log/nginx/error.log
# Watch log in real time (follow mode)
tail -f /var/log/nginx/access.log
# Show first 5 lines
head -n 5 /etc/nginx/nginx.conf
```

**📖 Reading Commands**

- **cat** — dumps the entire file to screen. Good for config files. Bad for 50MB log files (will flood the terminal).
- **less** — opens a scrollable viewer. Arrow keys or Page Up/Down to scroll. Press `/` to search, `q` to quit. Use this for large files.
- **tail -n 20** — shows the last 20 lines. Change the number as needed. Default is 10 lines.
- **tail -f** (follow mode) — keeps watching the file and prints new lines as they're written. The most useful command during a live incident. Press Ctrl+C to stop.
- **head -n 5** — shows the first N lines. Useful to peek at a CSV or log file format without reading everything.

#### 2.2 grep — Search Inside Files

```
# Find all ERROR lines in a log
grep "ERROR" /var/log/app.log
# Case-insensitive search
grep -i "error" app.log
# Show 3 lines BEFORE and AFTER each match
grep -B3 -A3 "OutOfMemoryError" app.log
# Count how many lines match
grep -c "404" /var/log/nginx/access.log
# Search recursively in all files in a folder
grep -r "password" /etc/nginx/
```

**📖 grep Options**

- **grep "pattern" file** — prints every line containing the pattern. grep stands for "global regular expression print."
- **-i** — case-insensitive. "error", "ERROR", "Error" all match.
- **-B3 -A3** — show 3 lines Before and After each match. Gives context around the error — see what happened just before and after the problem line.
- **-c** — count of matching lines. How many 404 errors did we serve today?
- **-r** — recursive. Search all files inside a directory. Use `-rn` to also show line numbers.
- Combine with pipes: `cat app.log | grep "ERROR" | grep "database"` — chain multiple filters.

#### 2.3 find — Locate Files by Name, Type, or Age

```
# Find files by name
find /etc -name "nginx.conf"
# Find all .log files
find /var/log -name "*.log"
# Find files larger than 100MB
find /var -size +100M
# Find files modified in the last 24 hours
find /opt/app -mtime -1
# Find and delete old log files (over 30 days)
find /var/log -name "*.log" -mtime +30 -delete
```

**📖 find Command**

- **find /path -name "pattern"** — searches recursively from the starting path for files matching the name pattern.
- **-size +100M** — files larger than 100MB. Use `+` for greater than, `-` for less than. Units: `k`=KB, `M`=MB, `G`=GB.
- **-mtime -1** — modified within the last 1 day. `-mtime +30` = older than 30 days.
- **-delete** — deletes matching files. **Always run without -delete first** to verify what will be deleted before adding -delete.
- Find + grep together: `find /etc -name "*.conf" | xargs grep "listen 80"` — search inside all .conf files for a specific line.

#### 2.4 Text Editing — nano for Beginners, vim for Production

```
# nano — simple editor (shortcuts at bottom)
nano /etc/nginx/sites-available/default
# Ctrl+O = save, Ctrl+X = exit, Ctrl+W = search
# vim — powerful, steep learning curve
vim /etc/nginx/nginx.conf
# i = insert mode (type text)
# Esc = back to command mode
# :w = save, :q = quit, :wq = save+quit
# :q! = quit WITHOUT saving
# Quick file edit trick: use sed for one-liners
sed -i 's/listen 80/listen 8080/g' nginx.conf
```

**📖 Text Editors**

- **nano** — beginner-friendly. Keyboard shortcuts shown at the bottom. Use for quick edits on servers that have it installed. Not available on all minimal Linux installs.
- **vim** — present on virtually every Linux system. Two modes: command mode (navigate, run commands) and insert mode (type text). Press `i` to start typing, `Esc` to stop. `:wq` to save and exit.
- Most common vim mistake: typing without pressing `i` first — commands get misinterpreted. Always check the bottom of the screen — it shows `-- INSERT --` when in insert mode.
- **sed -i 's/old/new/g' file** — find and replace in a file without opening an editor. The `-i` edits in place. `g` = replace all occurrences on each line. Used in Ansible tasks and shell scripts to automate config edits.

### 3. Phase 3 — Permissions, Users and Groups

**Business Problem:** The TechAxis deployment script fails with "Permission denied." A developer can't read the application config. A service can't write to its log directory. Understanding Linux permissions is mandatory — it is the most common source of problems on freshly configured servers.

🔧 Where Permissions & Users Are Used in the Stack

Ansible role: deploy application → chown / chmod (Linux permissions) → App can write to /var/log/app

Docker: RUN in Dockerfile → useradd / chmod 755 (Linux users) → Non-root container user

Every Dockerfile best practice says "don't run as root." Every Kubernetes security policy says "run as non-root user." Both are enforced through Linux user and permission commands.

#### 3.1 Understanding Permission Bits

```
Reading ls -la Output
=======================

-rwxr-xr-- 1 deepak devops 2048 Mar 29 09:00 deploy.sh
│ │││ │││ │││
│ │││ │││ └── other: r-- = read only
│ │││ └────── group (devops): r-x = read + execute
│ └────────── owner (deepak): rwx = read + write + execute
└──────────── file type: - = file, d = directory, l = symlink

r = read  (4)    w = write (2)    x = execute (1)

chmod 755 = rwxr-xr-x = owner:rwx(7) group:r-x(5) other:r-x(5)
chmod 644 = rw-r--r-- = owner:rw-(6) group:r--(4) other:r--(4)
chmod 600 = rw------- = owner:rw-(6) group:---(0) other:---(0)
```

```
# Change permissions
chmod 755 deploy.sh # rwxr-xr-x
chmod 644 config.yml # rw-r--r--
chmod 600 ~/.ssh/id_rsa # rw------- (private key!)
# Symbolic mode (easier to read)
chmod +x script.sh # add execute for all
chmod u+x,g-w file # user+execute, group-write
# Change ownership
chown deepak:devops deploy.sh
chown -R www-data:www-data /var/www/html
```

**📖 chmod and chown**

- **chmod 755** — numeric mode. Each digit = owner, group, other. 7=rwx, 5=r-x, 4=r--. Standard for executable scripts and directories.
- **chmod 644** — standard for config files and web content. Owner can edit, everyone can read.
- **chmod 600** — for SSH private keys. Only owner can read/write. SSH refuses to use a key file with looser permissions.
- **chmod +x** — symbolic mode. Add execute permission without changing anything else.
- **chown user:group file** — change who owns the file. Requires sudo for files you don't own.
- **chown -R** — recursive: change ownership of a directory and everything inside it. Used after copying app files to give the web server user ownership.

#### 3.2 User and Group Management

```
# Create a new user
sudo useradd -m -s /bin/bash deepak

# Set user password
sudo passwd deepak

# Create a group
sudo groupadd devops

# Add user to a group
sudo usermod -aG devops deepak
sudo usermod -aG sudo deepak   # grant sudo access
# List groups a user belongs to
groups deepak

# Delete a user (keep home dir)
sudo userdel olduser
```

**📖 User Management**

- **useradd -m -s /bin/bash** — `-m` creates a home directory, `-s` sets the login shell. Without these flags, the user gets no home dir and no shell (can't log in).
- **passwd username** — interactively set a password. Prompts for new password twice.
- **usermod -aG group user** — `-a` means APPEND to groups (critical — without `-a`, the user is removed from all other groups). `-G` specifies the group list.
- **usermod -aG sudo** — grants sudo access (Ubuntu). On CentOS/RHEL: add to the `wheel` group instead.
- Group membership changes take effect at next login. Current session still uses old groups. Run `newgrp devops` or log out and back in.

#### 3.3 sudo — Running Commands as Root

```
# Run one command as root
sudo apt update
# Open a root shell (use with caution)
sudo -i
# Run command as a specific user (e.g. www-data)
sudo -u www-data ls /var/www/html
# Check what you can run with sudo
sudo -l
# Edit sudoers safely (do NOT edit directly)
sudo visudo
```

**📖 sudo — Principle of Least Privilege**

- **sudo command** — runs one specific command as root. Logs the action to `/var/log/auth.log`. This is the audit trail.
- **sudo -i** — opens a root shell. Every subsequent command runs as root until you `exit`. Use sparingly — mistakes as root have no safety net.
- **sudo -u www-data** — run as a specific user, not root. Used to test if the web server user can access files it needs.
- **sudo -l** — lists what sudo commands your current user is allowed to run. Useful when you need to know your permissions on an unfamiliar server.
- **visudo** — the ONLY safe way to edit /etc/sudoers. It validates the syntax before saving. A syntax error in sudoers locks everyone out of sudo permanently.

### 4. Phase 4 — Processes, Services and Networking

**Business Problem:** The TechAxis client's server is responding slowly. Something is consuming all CPU. The application port is not open. You need to diagnose and fix production issues in real time — using only the terminal.

🔧 Where Process / Network Commands Are Used in the Stack

AWS EC2 instance high CPU alert → top / ps aux / kill (Linux processes) → Identify and stop runaway process

Kubernetes: nginx pod not reachable → ss -tlnp / curl / netstat (Linux networking) → Check port binding and connectivity

Ansible: restart failed service → systemctl status / restart (Linux services) → Service back online

#### 4.1 Process Management

```
# Show all running processes
ps aux
# Filter for a specific process
ps aux | grep nginx

# Live CPU/memory view (q to quit)
top
# Better live view (install: apt install htop)
htop
# Kill a process by ID (PID)
kill 1234
# Force kill (SIGKILL — process cannot ignore this)
kill -9 1234
# Kill all processes with a name
pkill -f "gunicorn"
```

**📖 Processes**

- **ps aux** — lists every running process: PID, CPU%, MEM%, command. `a`=all users, `u`=user-oriented format, `x`=include processes without terminal.
- **ps aux | grep nginx** — pipe the full process list into grep to find a specific process. Shows PID, CPU, memory usage.
- **top** — real-time process monitor. Updates every 3 seconds. Shows load average, CPU%, memory. Press `M` to sort by memory, `P` to sort by CPU, `q` to quit.
- **kill PID** — sends SIGTERM (signal 15): "please shut down gracefully." The process can catch this and clean up.
- **kill -9 PID** — sends SIGKILL: "die immediately." The OS forces termination. Use when a process is frozen and ignoring SIGTERM.

#### 4.2 systemctl — Managing Services

```
# Check if nginx is running
sudo systemctl status nginx

# Start a service
sudo systemctl start nginx

# Stop a service
sudo systemctl stop nginx

# Restart a service
sudo systemctl restart nginx

# Reload config without restart
sudo systemctl reload nginx

# Enable auto-start on boot
sudo systemctl enable nginx

# List all failed services
systemctl --failed
```

**📖 systemctl — systemd Service Manager**

- **systemctl status service** — shows running state, PID, recent log lines, and whether it's enabled on boot. Run this FIRST for any service problem.
- **systemctl start/stop** — starts or stops the service immediately. Does not affect auto-start on boot.
- **systemctl restart** — stops then starts. Use when you've changed a config file that requires a full restart (e.g. binding to a new port).
- **systemctl reload** — sends SIGHUP to reload configuration without restarting. Zero downtime. Not all services support this — nginx and apache do.
- **systemctl enable/disable** — controls whether the service starts automatically when the server boots. A service that starts but is not enabled will not come back after a server restart.
- **systemctl --failed** — lists all services that failed to start. First thing to check after a server reboot.

deepak@techaxis-server:~$ sudo systemctl status nginx ● nginx.service - A high performance web server Loaded: loaded (/lib/systemd/system/nginx.service; enabled ; preset: enabled) Active: active (running) since Sat 2026-03-29 09:00:12 IST; 3h 21min ago Docs: man:nginx(8) Process: 1234 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS) Main PID: 1242 (nginx) Tasks: 3 (limit: 4096) Memory: 12.4M CPU: 1.238s CGroup: /system.slice/nginx.service ├─1242 "nginx: master process /usr/sbin/nginx -g daemon off;" └─1243 "nginx: worker process" Mar 29 09:00:12 techaxis-server systemd[1]: Starting nginx...

#### 4.3 Networking — Check Ports and Connectivity

```bash
# Show all listening ports and which process owns them
ss -tlnp
# Check if port 80 is open on a remote host
curl -I http://192.168.1.10

# Test if a port is reachable
telnet 192.168.1.10 5432
# DNS lookup
nslookup api.techaxis.in
dig api.techaxis.in
# Trace network hops to a host
traceroute 8.8.8.8
# Show network interfaces and IPs
ip addr
```

**📖 Networking Commands**

- **ss -tlnp** — replacement for netstat. `-t`=TCP, `-l`=listening, `-n`=numeric ports (no DNS), `-p`=show process. Shows exactly which program is listening on which port.
- **curl -I URL** — sends an HTTP HEAD request and shows response headers. Useful to check if a web server responds and what status code it returns without downloading the body.
- **nslookup / dig** — DNS resolution. Confirms a domain resolves to the expected IP. `dig` gives more detailed DNS information.
- **ip addr** — shows all network interfaces and their IP addresses. Replacement for the deprecated `ifconfig`.
- **ping hostname** — basic connectivity test. If ping fails, the host is unreachable or blocking ICMP. In cloud environments, security groups often block ping — port connectivity matters more.

#### 4.4 Package Management — Installing Software

```
# Ubuntu / Debian (apt)
sudo apt update # refresh package list
sudo apt install nginx  # install
sudo apt remove nginx   # remove (keep config)
sudo apt upgrade # upgrade all packages
# CentOS / RHEL / Amazon Linux (yum/dnf)
sudo yum install nginx
sudo dnf install nginx   # newer systems
# Check if a package is installed
dpkg -l | grep nginx    # Debian/Ubuntu
rpm -qa | grep nginx   # CentOS/RHEL
```

**📖 Package Managers**

- **apt update** — downloads the latest package list from repositories. Does NOT upgrade anything — just refreshes what's available. Always run before install.
- **apt install** — downloads and installs a package and all its dependencies. The `-y` flag skips the "Do you want to continue?" prompt — essential for automation scripts.
- **apt upgrade** — upgrades all installed packages to latest versions. Run on a test server first in production environments.
- Ubuntu/Debian uses **apt/dpkg**. CentOS/RHEL/Amazon Linux uses **yum/rpm** (or **dnf** on newer versions). Know which distro you're on — wrong package manager = command not found.

### 5. Phase 5 — Shell Scripting, Cron Jobs and Log Management

**Business Problem:** TechAxis runs the same five commands every morning to check server health. Every Friday they need to compress and archive old log files. On every deployment they need to backup the config, deploy, and restart the service. All of this is written once as a shell script and automated with cron.

🔧 Where Shell Scripts Are Used in the Stack

Jenkins CI/CD Pipeline → Shell Script: build.sh, deploy.sh → Builds, tests, and deploys app

Ansible task: run custom command → Shell: /opt/scripts/health-check.sh → Returns 0 (success) or 1 (failure)

Docker ENTRYPOINT → Shell script: entrypoint.sh → Starts application in container

Shell scripts are the glue of DevOps. Every CI/CD pipeline, every Ansible task that runs custom logic, every Docker entrypoint, every cron job is a shell script. Writing clean, reliable shell scripts is one of the highest-value skills for a junior DevOps engineer.

#### 5.1 Shell Script Basics

```bash
#!/bin/bash
# deploy.sh — TechAxis deployment script
# Exit immediately if any command fails
set -e
# Variables
APP_DIR="/opt/techaxis/app"
LOG_FILE="/var/log/techaxis/deploy.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# Print with timestamp
echo "[$TIMESTAMP] Starting deployment"
# Make script executable: chmod +x deploy.sh
# Run it: ./deploy.sh  OR  bash deploy.sh
```

**📖 Script Structure**

- **#!/bin/bash** — shebang line. Must be the very first line. Tells the OS which interpreter to use. Always include this.
- **set -e** — "exit on error." If any command fails (returns non-zero exit code), the script stops immediately instead of continuing and making things worse. Always add this to production scripts.
- **VARIABLE=value** — no spaces around `=`. Use uppercase for constants. Access with `$VARIABLE` or `${VARIABLE}` (curly braces needed when adjacent to other text).
- **$(command)** — command substitution. Runs the command and captures its output. Here, `$(date ...)` captures the current date as a string.
- **echo** — prints text to standard output. Redirect to a file: `echo "message" >> logfile`.

#### 5.2 Conditionals and Loops

```
# if/else — check if a service is running
if systemctl is-active --quiet nginx; then
echo "nginx is running"
else
echo "nginx is DOWN — restarting"
sudo systemctl start nginx
fi
# for loop — process a list of servers
for SERVER in web-01 web-02 web-03; do
echo "Checking $SERVER"
ssh $SERVER "systemctl status nginx"
done
```

**📖 Control Flow**

- **if command; then ... fi** — the condition is ANY command. If the command exits with 0 (success), the if-block runs. If non-zero (failure), the else-block runs.
- **systemctl is-active --quiet** — returns exit code 0 if service is active, non-zero if not. The `--quiet` suppresses output — we only care about the exit code, not the text.
- **for var in list; do ... done** — loops over space-separated items. The variable `$SERVER` takes each value in turn.
- Compare values: `if [ "$VAR" = "expected" ]; then` — spaces inside `[ ]` are mandatory.
- Check if file exists: `if [ -f /path/to/file ]; then`. Check if directory exists: `if [ -d /path/ ]; then`.

#### 5.3 A Real Deployment Script

```bash
#!/bin/bash
# Full deployment script — TechAxis production deploy
set -euo pipefail   # -u=error on undefined var, -o pipefail=pipe errors count
APP_DIR="/opt/techaxis/app"
BACKUP_DIR="/opt/techaxis/backups"
GIT_REPO="https://github.com/techaxis/webapp.git"
```

> **set -euo pipefail** — the professional safety triple: `-e` exit on error, `-u` treat undefined variables as errors, `-o pipefail` a failed pipe command counts as failure. Always use all three in production scripts.

```bash
# Step 1: Backup current app
echo "Creating backup..."
cp -r $APP_DIR "${BACKUP_DIR}/app-$(date +%Y%m%d-%H%M%S)"
# Step 2: Pull latest code
cd $APP_DIR
git pull origin main

# Step 3: Restart service
sudo systemctl restart techaxis-app

# Step 4: Verify it came up
sleep 3
if curl -sf http://localhost:8080/health; then
echo "✅ Deployment successful"
else
echo "❌ Health check failed — check logs"
exit 1
fi
```

> **Backup before deploy** — always. If the deploy breaks the app, you need to restore. The backup name includes a timestamp so each backup is uniquely named.
**curl -sf** — `-s`=silent (no progress output), `-f`=fail with non-zero exit code if HTTP status is 4xx/5xx. This makes curl work as an if-condition correctly.
**exit 1** — exits the script with a failure exit code. CI/CD pipelines check exit codes — a non-zero exit marks the pipeline as failed and sends alerts.
**sleep 3** — wait 3 seconds for the service to fully start before health checking. Services don't become ready the instant systemctl says they're running.

#### 5.4 Cron Jobs — Schedule Automated Tasks

```
# Edit your cron jobs
crontab -e
# Cron format:
# MIN  HOUR  DAY  MONTH  WEEKDAY  COMMAND
#  *    *     *     *      *      = every minute

# Run backup every day at 2 AM
0 2 * * * /opt/scripts/backup.sh
# Run health check every 5 minutes
*/5 * * * * /opt/scripts/health-check.sh
# Run log cleanup every Sunday at midnight
0 0 * * 0 /opt/scripts/log-cleanup.sh
```

**📖 Cron Syntax**

- Five fields: **minute (0-59) hour (0-23) day (1-31) month (1-12) weekday (0-7, 0 and 7 = Sunday)**.
- ***** means "every." `* * * * *` = every minute of every hour of every day.
- ***/5** = every 5 units. `*/5 * * * *` = every 5 minutes.
- **0 2 * * *** = at 02:00 every day (field 1=minute 0, field 2=hour 2, rest=every).
- Use [crontab.guru](https://crontab.guru) to visualise cron expressions. Cron errors are hard to spot without a validator.
- Always redirect cron output to a log: `/opt/scripts/backup.sh >> /var/log/backup.log 2>&1`. Without this, you never know if cron jobs succeed or fail.

#### 5.5 Log Management — journalctl and logrotate

```
# View systemd service logs
journalctl -u nginx

# Follow logs in real time
journalctl -u nginx -f
# Show logs from last 1 hour
journalctl -u nginx --since "1 hour ago"
# Show logs since a specific time
journalctl -u nginx --since "2026-03-29 08:00"
# Show only errors and above
journalctl -u nginx -p err

# Check disk used by journal logs
journalctl --disk-usage
```

**📖 journalctl — systemd Logs**

- **journalctl -u service** — shows all logs for a specific systemd service. Every service started by systemd writes to the journal automatically.
- **-f** — follow mode (like tail -f). Shows new log entries as they arrive. Press Ctrl+C to stop.
- **--since "1 hour ago"** — time filters. Also supports: `today`, `yesterday`, `2026-03-29`, `2026-03-29 09:00:00`.
- **-p err** — filter by priority. Levels: debug, info, notice, warning, err, crit, alert, emerg. `-p err` shows errors and above (more critical levels).
- Journal logs are stored in binary format in `/var/log/journal/`. They do not show in `tail /var/log/syslog` — you must use journalctl.

### 6. Linux in Real DevOps and Cloud Stacks

- **🐳 Docker / Container Stack** — 

- **⚙️ Ansible / Configuration Mgmt** — 

- **🚀 CI/CD Pipelines** — 

- **☁️ Cloud Infrastructure** — 

### 7. Linux Commands Quick Reference

Command

What It Does

Common Use in DevOps

pwd

Show current directory

First thing to run after SSH

ls -la

List files with permissions and hidden files

Check file permissions, find hidden config files

cd /path

Change directory

Navigate to /etc, /var/log, /opt/app

mkdir -p /path

Create directory (and parents)

Create log dirs, app dirs in Ansible/Docker

cp -r src dst

Copy file or directory

Backup config before editing

mv old new

Move or rename file

Rename build artifacts, move configs

rm -rf /path

Delete directory and contents (NO UNDO)

Clean build artifacts, remove old versions

cat file

Print file content

Read small config files

less file

Scroll through large file

Read large log files

tail -f file

Follow log in real time

Watch nginx/app logs during incidents

grep "text" file

Search for text in file

Find errors in logs, search configs

grep -r "text" /dir

Recursive search in directory

Find config value across all nginx configs

find /path -name "*.log"

Find files matching pattern

Find all log files, find large files

sed -i 's/old/new/g' file

Find and replace in file

Update config values in Ansible/scripts

chmod 755 file

Set file permissions

Make scripts executable, secure configs

chown user:group file

Change file ownership

Give web server user ownership of web root

sudo command

Run command as root

Install packages, manage services, edit system files

useradd -m -s /bin/bash user

Create a Linux user

Create app user, deploy user in Ansible

usermod -aG group user

Add user to group

Grant sudo or docker group access

ps aux | grep process

Find a running process

Check if nginx/java/node is running

kill -9 PID

Force kill a process

Kill a hung process during incident

top / htop

Real-time process monitor

Find high CPU/memory process

systemctl status nginx

Check service status

First check during any service incident

systemctl restart nginx

Restart a service

Apply config changes

systemctl enable nginx

Enable auto-start on boot

After installing a new service

ss -tlnp

Show listening ports and processes

Verify app is listening on expected port

curl -I http://host

HTTP header check

Verify web server response

journalctl -u service -f

Follow service logs in real time

Watch logs during deployment

df -h

Show disk usage (human readable)

Check if disk is full (common incident cause)

free -h

Show memory usage

Check available RAM during performance issues

du -sh /path

Show directory size

Find what's using all disk space

crontab -e

Edit current user's cron jobs

Schedule backups, health checks, cleanups

| (pipe)

Send output of one command to another

ps aux | grep nginx, cat log | grep ERROR

> file

Redirect output to file (overwrites)

Save command output to a log file

>> file

Append output to file

Add to existing log without erasing

2>&1

Redirect stderr to stdout

Capture errors in same log as output

### 8. Interview Questions — Linux

##### Interview Q&A — Fresher Level (0–1 Year Linux / DevOps Experience)

**Q: Q1. What does chmod 755 mean and when would you use it?**

A: chmod 755 sets permissions: **owner = rwx (7)**, **group = r-x (5)**, **others = r-x (5)**.
**r=4, w=2, x=1** — add the numbers: 4+2+1=7 (rwx), 4+0+1=5 (r-x), 4+0+0=4 (r--).
755 is the standard permission for **shell scripts and directories**: the owner can read/write/execute, everyone else can read and execute but not modify.
Common uses: making a deploy script executable (`chmod 755 deploy.sh`), setting directory permissions so web server can serve files from it, setting up application binaries.
Contrast: **644** for config files (owner edits, others read, no execute), **600** for SSH private keys (only owner reads, no one else), **400** for read-only secrets.

**Q: Q2. What is the difference between kill and kill -9?**

A: **kill PID** — sends SIGTERM (signal 15). Politely asks the process to terminate. The process can catch this signal, save its state, release resources, and shut down gracefully. Most well-written programs handle SIGTERM properly.
**kill -9 PID** — sends SIGKILL (signal 9). The OS immediately terminates the process. The process cannot catch or ignore this signal. No cleanup, no graceful shutdown. Open files may be left in a corrupt state.
**Always try kill first**, wait a few seconds, then use kill -9 only if the process is still running (frozen, stuck in a loop, or actively ignoring SIGTERM).
A Java or Node.js application receiving SIGTERM can finish processing current requests before exiting. kill -9 cuts them off mid-request, potentially causing data corruption or incomplete transactions.

**Q: Q3. What does set -e and set -o pipefail do in a shell script?**

A: **set -e** — "exit on error." If any command in the script returns a non-zero exit code (indicating failure), the script immediately stops. Without this, a script continues running even after a failed command, often making the situation worse.
**set -o pipefail** — by default in bash, a pipe chain like `grep "error" file | wc -l` only checks the exit code of the LAST command (wc -l). If grep fails (file doesn't exist), pipefail ensures the whole pipe is considered failed.
**set -u** — treats undefined variables as errors. Without this, `$UNDEFINED_VAR` silently expands to an empty string, which can cause dangerous commands like `rm -rf $APPDIR/` to become `rm -rf /` if APPDIR is undefined.
Always use `set -euo pipefail` at the top of every production bash script. It makes scripts fail loudly and early rather than silently corrupting state.

**Q: Q4. How do you find which process is using a specific port (e.g. port 80)?**

A: **ss -tlnp | grep :80** — most reliable modern approach. Shows which process (with PID and program name) is listening on port 80.
**lsof -i :80** — lists open files and network connections. Shows the PID and program name for port 80. Requires `sudo lsof -i :80` to see processes owned by other users.
**fuser 80/tcp** — simply prints the PID using port 80. Minimal output. Add `-v` for verbose output including program name.
Real-world scenario: "Port 80 is already in use — nginx fails to start." Run `ss -tlnp | grep :80` to find the culprit process, then either kill it or change nginx's port configuration.

**Q: Q5. What is the difference between absolute and relative paths in Linux?**

A: **Absolute path** — starts from the root `/`. Always works regardless of where you currently are. Example: `/etc/nginx/nginx.conf`, `/home/deepak/deploy.sh`.
**Relative path** — starts from your current directory. `../config/nginx.conf` means "go up one level, then into config folder." Only works if you're in the right starting directory.
In **shell scripts and cron jobs**, always use absolute paths. Cron jobs run without your shell's PATH and current directory — relative paths will fail. `/opt/scripts/backup.sh` not `./backup.sh`.
In **Dockerfiles**, use absolute paths for COPY and WORKDIR commands for the same reason — container build context doesn't have a working directory concept.

**Q: Q6. A production server has no disk space left. How do you diagnose and fix it?**

A: **Step 1: Confirm the problem** — `df -h` shows disk usage for all filesystems. Find which partition is at 100%.
**Step 2: Find the large files/directories** — `du -sh /* 2>/dev/null | sort -rh | head -20` shows top 20 largest directories from root. Drill down: `du -sh /var/log/*` to find large logs.
**Step 3: Common culprits** — large log files in `/var/log`, Docker images in `/var/lib/docker`, old package caches in `/var/cache/apt`, application cores and temp files.
**Step 4: Clean safely** — compress old logs: `gzip /var/log/app.log.1`. Clean apt cache: `sudo apt autoremove && sudo apt clean`. Clean Docker: `docker system prune`. Delete old temp files: `find /tmp -mtime +7 -delete`.
**Prevention**: configure logrotate for all application logs, set up cron jobs to clean temp files, configure Docker log driver with size limits, add disk usage monitoring alert at 80%.

**Quiz: Quiz 1 — You run: rm -rf /opt/app . (notice the space before the dot). What happens?**

- A) Deletes only the /opt/app directory
- B) Deletes /opt/app AND then tries to delete the current directory (.) — which on some systems deletes everything in your current working directory
- C) The dot is ignored — only /opt/app is deleted
- D) The command fails because rm cannot delete a dot

> **Answer/explanation:** ✅ Answer: **B — Deletes /opt/app then the current directory contents**
The space makes `.` a separate argument. rm receives two arguments: `/opt/app` and `.` (current directory).
Deleting `.` (current directory) with -rf deletes everything inside it — potentially wiping your home directory or /opt or wherever you currently are.
This is a real accident that has happened in production. Always double-check rm commands. Use `echo rm -rf /opt/app` first (prints without executing) to verify what will be deleted.
Modern versions of rm refuse to delete `.` or `/` — but you cannot rely on this safety on all systems.

**Quiz: Quiz 2 — You add a user to the docker group: sudo usermod -aG docker deepak. But deepak still gets "permission denied" when running docker commands. Why?**

- A) The command was wrong — it should be usermod -G docker deepak (without -a)
- B) Group membership changes only take effect after the user logs out and back in (or runs newgrp docker)
- C) You need to restart the docker daemon after adding a user
- D) Docker requires sudo regardless of group membership

> **Answer/explanation:** ✅ Answer: **B — Must log out and back in for group changes to take effect**
Linux reads group membership at login time. An already-running session keeps the old group list even after the user is added to a new group.
**Immediate fix without logout**: run `newgrp docker` to start a new shell with the updated group membership. Or run `su - deepak` to open a fresh login shell.
This is a very common confusion for freshers — "I added myself to the group but it still says permission denied!" The answer is always: log out and back in.
Note: `-a` in `usermod -aG` means APPEND. Without `-a`, the command replaces the user's supplementary groups with only the docker group, removing all other group memberships.

**Quiz: Quiz 3 — A cron job runs daily at 2 AM: 0 2 * * * /opt/scripts/backup.sh. It seems to not be running. What are the two most likely causes?**

- A) Cron only runs on weekdays — add a weekday field
- B) The script has no execute permission (chmod +x missing) OR the script path in crontab uses a relative path that cron can't resolve
- C) Cron requires root to run scripts in /opt
- D) The cron expression is wrong — the hour field should be 2AM not 2

> **Answer/explanation:** ✅ Answer: **B — Missing execute permission or relative path**
**Cause 1: chmod** — cron cannot run a script that doesn't have execute permission. Run `ls -la /opt/scripts/backup.sh` — you need to see `-rwxr-xr-x` or at least `-rwx`. Fix: `chmod +x /opt/scripts/backup.sh`.
**Cause 2: Absolute path** — cron runs with a minimal environment and no working directory. A relative path like `./backup.sh` or `backup.sh` will fail. Always use the full absolute path: `/opt/scripts/backup.sh`.
**Debugging cron**: redirect output to a log file: `0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1`. Without this, silent failures go completely unnoticed.
The cron expression `0 2 * * *` is correct: minute=0, hour=2 (2 AM), every day, every month, every weekday.

> **Linux Project — Core Takeaways for Freshers**

> - **The three first commands on any server** — pwd (where am I), ls -la (what is here), whoami (who am I). Run these every single time you SSH into an unfamiliar server before touching anything.
> - **tail -f is your best friend during incidents** — open a second terminal, run `tail -f /var/log/nginx/error.log`, then trigger the request in the first terminal. Watch the error appear in real time.
> - **Never run rm -rf without verifying** — use `echo rm -rf /path` to print what would be deleted, or `ls /path` to see what's there, before executing the real delete command.
> - **chmod 755 for scripts, 644 for configs, 600 for keys** — memorise these three. They cover 90% of permission scenarios you'll encounter. The SSH client will refuse to use a private key file that's too permissive.
> - **systemctl status service_name** is the first command for any service problem — it shows the current state, whether it's enabled on boot, and the last few log lines. This single command answers most "why isn't my service running?" questions.
> - **Always use set -euo pipefail** in shell scripts — production scripts without these safety flags will continue running after a failed command and corrupt state silently. Add these three options to every script you write.
> - **Always redirect cron output to a log** — add `>> /var/log/myjob.log 2>&1` to every cron job. Without this, you have no way to know if the job succeeded, failed, or even ran at all.
> - **grep -B3 -A3 "ERROR" logfile** — never grep for errors without context lines. The error message alone rarely tells you what caused it. The three lines before the error usually do.

##### Linux Engineering Standards — TechAxis Rules

- Never log into production servers as root — always log in as a named user and use sudo. Root sessions are not attributable to an individual engineer in the audit log; named user sudo sessions are
- Always test config file changes with a validation command before restarting the service: `nginx -t` for nginx, `sshd -t` for SSH, `apache2ctl configtest` for Apache. A syntax error in the config makes the service fail to restart
- Before editing any config file: create a backup first — `cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak.$(date +%Y%m%d)`. Takes 2 seconds. Saves hours of recovery time
- Use `screen` or `tmux` for long-running operations on remote servers — if your SSH connection drops while a database migration is running, the migration continues inside the screen session instead of being killed mid-run
- Set up SSH key-based authentication and disable password authentication — password auth is brute-force attacked within minutes of a server being exposed. SSH keys are exponentially harder to compromise
- Configure logrotate for every application log — without logrotate, log files grow without bound until they fill the disk. This is one of the most common causes of production outages that DevOps engineers are called in to fix at 2 AM

##### 🏋️ Hands-On Exercises — TechAxis Production Scenarios

1. **Server Health Check Script:** Write a bash script `health-check.sh` that checks: (1) nginx is running (systemctl is-active), (2) port 80 is listening (ss -tlnp), (3) disk usage is below 80% (df -h, parse the percentage with awk), (4) available memory is above 500MB (free -m, parse with awk). Print "✅ PASS" or "❌ FAIL" for each check with the actual value. Exit with code 0 if all pass, code 1 if any fail. Schedule it with cron to run every 5 minutes and log output to /var/log/health-check.log.
2. **Log Analyser Script:** Write a script that analyses an nginx access log and reports: (1) total number of requests, (2) number of 404 errors, (3) number of 500 errors, (4) top 5 most requested URLs, (5) top 5 IP addresses by request count. Use only Linux tools: grep, awk, sort, uniq, head. This exact script is asked in DevOps technical interviews.
3. **User Provisioning Script:** Write a script that takes a username as argument (`./provision-user.sh deepak`) and: (1) creates the user with home directory and bash shell, (2) adds them to the devops group, (3) creates .ssh directory with correct permissions (700), (4) creates authorized_keys file with correct permissions (600), (5) copies a provided public key into authorized_keys. Test by SSHing as the new user with the key.
4. **Automated Backup with Rotation:** Write a backup script that: (1) compresses the /opt/app directory into a timestamped tar.gz file in /opt/backups, (2) lists the number of backup files currently in /opt/backups, (3) deletes any backups older than 7 days using find -mtime, (4) reports the total disk space used by backups. Add it to root's crontab at 3 AM daily. Verify it ran by checking /var/log/backup.log the next morning.
5. **Incident Simulation — nginx Not Responding:** Deliberately misconfigure nginx (change the listen port to 8080 in the config), restart it, then diagnose and fix the problem using only the commands covered in this module — without looking at the config file directly first. Use: systemctl status nginx, journalctl -u nginx, ss -tlnp, curl -I, cat error.log, nginx -t. This simulation mirrors an actual production incident investigation workflow.

### Linux Project Complete 🎉

You have mastered the Linux skills that power TechAxis's entire infrastructure — file system navigation, grep and find for log analysis, user and permission management, service control with systemctl, process management, network diagnostics, shell scripting with proper error handling, cron job scheduling, and log management with journalctl. These commands run on every cloud server, inside every Docker container, and in every CI/CD pipeline every minute of every day.

> **Harish**
> 
> "Deepak, the client called at 2:47 AM. nginx was down, the site was showing 502. You SSH'd in, ran systemctl status nginx, read the journal, saw the config syntax error, ran nginx -t to confirm, fixed the typo in nano, ran nginx -t again to validate, reloaded the service. Site was back up in 4 minutes and 12 seconds. The client never knew there was a problem. That is what Linux administration looks like in production. Every command you used tonight, you learned this week."

> **Preetam**
> 
> "The health-check cron script you wrote sends us an alert before the disk hits 90%. Last week it caught a runaway Java process writing a 40GB heap dump to /tmp. We had 45 minutes to clean it up before the server ran out of space. Without the script, we would have found out from the client when their site went down. Linux skills don't just fix incidents — they prevent them."

> **Next: Advanced Linux — Performance Tuning, Security Hardening & Kernel Management**

> - System performance — vmstat, iostat, sar for CPU/IO analysis; strace to trace system calls of any running process
> - Security hardening — SSH hardening (disable root login, key-only auth, fail2ban), UFW/iptables firewall rules, SELinux/AppArmor basics
> - Kernel and boot — understanding /proc/sys, sysctl for kernel parameter tuning, grub boot loader, kernel modules with modprobe
> - Network configuration — static IP with netplan/nmcli, /etc/hosts, /etc/resolv.conf, iptables NAT and port forwarding
> - Advanced text processing — awk for column-based log analysis, sed for complex substitutions, cut/tr/sort/uniq pipelines
> - Linux for containers — cgroups (CPU/memory limits), namespaces (process isolation), what Docker actually does under the hood on the Linux kernel
