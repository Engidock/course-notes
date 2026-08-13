# Shell Scripting Project Mastery

> **👋 Hey Fresher — Read This First!**

> Shell scripting is the **superpower of every Linux/DevOps engineer**. Every company running Linux servers needs someone who can write bash scripts. You don't need to memorise hundreds of commands — you need to understand how to combine simple commands to solve real problems. This project walks you through building 5 scripts step by step, explaining every line. By the end, you'll have scripts you can actually show in interviews and use on the job.

> **Company in this project:** InfraCore Technologies — an IT infrastructure company in Bangalore with 150 engineers, managing 30+ Linux servers. You just joined as a Junior Linux/DevOps Engineer. Your senior is Harish. Let's begin.

#### What You Will Build in This Project

You will build **5 production-quality Bash scripts** that InfraCore Technologies needs to automate their Linux server operations. These scripts are the types of things senior engineers write on their first week — and that Junior engineers are tested on in interviews.

Server Health Check, Log Rotation & Cleanup, Automated Backup, User Account Manager, Deployment Script, cron, awk & sed, grep & find

> **🖥️ Script 1 — Server Health Monitor**

> Check CPU, memory, disk usage, and running services every 5 minutes. Write a report and send an alert if anything crosses the danger threshold.

> Find log files older than 7 days, compress them with gzip, move to archive folder, and delete files older than 30 days — all automatically.

> Take timestamped backups of critical directories, compress them, copy to backup server via rsync, and verify backup integrity with checksums.

> Create, disable, and audit user accounts from a CSV file. Set passwords, assign groups, create home directories — all in one script run.

> Pull latest code from Git, run tests, stop the old service, deploy the new version, start the service, verify it's running, and rollback if anything fails.

**Scene 1 — Server Room, InfraCore Technologies, Bangalore | Your First Day**

> **Harish** _Senior DevOps Engineer — 7 years at InfraCore_
> 
> Welcome, Vikram. We manage 30 Linux servers here — Ubuntu, CentOS, and RHEL. Every day, our ops team manually logs into servers to check disk space, restarts services that have crashed, and copies logs to a backup location. That is three engineers spending four hours every morning doing the same thing. Shell scripting fixes this. But you need to write scripts that actually work in production — not textbook examples.

> **Vikram (You)** _Junior DevOps Engineer — Day 1 at InfraCore_
> 
> I've used basic commands like ls, cd, cp in college labs. I know what a shell is. But I've never written an actual script that runs on a real server. Where do I start?

> **Harish** _Senior DevOps Engineer_
> 
> A shell script is just a text file with Linux commands inside it — executed top to bottom, exactly like typing them manually in the terminal. The difference is you write it once, and the server runs it automatically a hundred times. Start with Script 1: server health check. I want it checking CPU, memory, disk, and key services every 5 minutes. If anything is wrong, write to a log file. That's your first task.

> **Preetam** _Infrastructure Lead — InfraCore Technologies_
> 
> And learn to read error messages. If your script fails at 3 AM, the log is the only thing that tells you what happened. Every script must log what it does — timestamp, action, result. No logging means no debugging. That rule is non-negotiable here.

### 1. Shell Scripting Foundations — Before You Write a Single Script

Before building the five scripts, every fresher needs to understand how Bash scripts are structured. These fundamentals appear in every script you will ever write.

> **What is a Shell Script? (Simple Explanation)**

> The shell is the program that takes your commands and talks to the Linux operating system. Bash (Bourne Again Shell) is the most common shell. A shell script is simply a plain text file containing a list of Bash commands, saved with a `.sh` extension. When you run it, the shell executes each line in order — exactly as if you had typed them yourself in the terminal. The power comes from adding variables, loops, conditions, and functions around those commands.

```
InfraCore Shell Scripts — Project Structure
============================================

  infracore-scripts/
  ├── scripts/
  │   ├── health_check.sh        ← Script 1: Server health monitor
  │   ├── log_rotation.sh        ← Script 2: Log rotation & cleanup
  │   ├── backup.sh              ← Script 3: Automated backup
  │   ├── user_manager.sh        ← Script 4: User account management
  │   └── deploy.sh              ← Script 5: Application deployment
  ├── config/
  │   └── infracore.conf         ← Shared config (paths, thresholds, IPs)
  ├── logs/
  │   └── automation.log         ← All script activity logged here
  ├── data/
  │   └── new_users.csv          ← Input file for Script 4
  └── crontab.txt                ← Cron schedule for all scripts
```

#### Essential Bash Concepts — The Building Blocks

1. The Shebang Line — Every Script Must Start With This
The very first line of every shell script must be `#!/bin/bash`. This tells Linux which interpreter to use. Without it, Linux doesn't know what type of script it is.

```bash
#!/bin/bash
# The shebang line ↑ always goes on line 1. It tells the OS to use /bin/bash.
# Lines starting with # are comments — they are ignored by the shell.
# Comments explain WHY you're doing something, not just WHAT.
# Good comment:
# Check disk usage — alert if above 85% to prevent app crashes from full disk
df -h /

# Bad comment (just repeats the command, adds no value):
# Run df command
df -h /
```

2. Variables — Storing Values to Reuse
Variables let you store values and reuse them. No spaces around the = sign. Access with $. Use UPPERCASE for configuration variables as a convention.

```bash
#!/bin/bash
# Variables — storing values
SERVER_NAME="prod-server-01" # String variable (no spaces around =)
DISK_THRESHOLD=85           # Number variable
LOG_FILE="/var/log/health.log" # Path variable
# Access a variable with $ prefix
echo "Checking server: $SERVER_NAME"
echo "Disk alert threshold: ${DISK_THRESHOLD}%" # {} are optional but good practice
# Capture command output into a variable using $( )
CURRENT_DATE=$(date "+%Y-%m-%d %H:%M:%S")
HOSTNAME=$(hostname)
CURRENT_USER=$(whoami)

echo "Timestamp: $CURRENT_DATE"
echo "Running on: $HOSTNAME as $CURRENT_USER"
```

3. If-Else Conditions — Making Decisions
If-else lets your script take different actions based on conditions. In Bash, conditions use square brackets [ ] and specific comparison operators.

```bash
#!/bin/bash
# If-else conditions in Bash
DISK_USAGE=87  # pretend we measured this
# Numeric comparisons: -gt (greater than), -lt (less than), -eq (equal), -ge (≥), -le (≤)
if [ $DISK_USAGE -gt 85 ]; then
echo "WARNING: Disk usage is ${DISK_USAGE}% — above threshold!"
elif [ $DISK_USAGE -gt 70 ]; then
echo "CAUTION: Disk at ${DISK_USAGE}% — monitor closely"
else
echo "OK: Disk at ${DISK_USAGE}% — healthy"
fi # 'fi' closes every if block (if spelled backwards)
# String comparisons: = (equal), != (not equal), -z (empty), -n (not empty)
STATUS="running"
if [ "$STATUS" = "running" ]; then
echo "Service is running"
fi
# File/directory checks: -f (file exists), -d (directory exists), -r (readable)
if [ -f "/etc/nginx/nginx.conf" ]; then
echo "Nginx config exists"
fi
```

4. Loops — Repeating Actions
Loops let you apply the same action to multiple items — multiple servers, multiple files, multiple users. The for loop and while loop are the two you will use most.

```bash
#!/bin/bash
# Loops — repeating actions
# FOR loop — iterate over a list of items
for SERVICE in nginx mysql redis sshd; do
if systemctl is-active --quiet $SERVICE; then
echo "[UP]   $SERVICE"
else
echo "[DOWN] $SERVICE — attempting restart..."
systemctl start $SERVICE
fi
done
# FOR loop over files in a directory
for LOG_FILE in /var/log/*.log; do
echo "Processing: $LOG_FILE"
done
# WHILE loop — repeat as long as a condition is true
COUNT=1
while [ $COUNT -le 5 ]; do
echo "Attempt $COUNT..."
COUNT=$(( COUNT + 1 ))  # arithmetic in bash uses $(( ))
done
```

5. Functions — Reusable Blocks of Commands
Functions let you name a block of commands and call it multiple times. This prevents copy-pasting the same code in five places. Define first, then call.

```bash
#!/bin/bash
# Functions — reusable blocks
# Define a logging function — used in every script we write
log_message() {
    TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")
    LEVEL=$1 # $1 is the first argument passed to the function
MESSAGE=$2 # $2 is the second argument
echo "[$TIMESTAMP] [$LEVEL] $MESSAGE" | tee -a $LOG_FILE
# tee prints to screen AND appends (-a) to the log file at the same time
}

# Call the function — pass level and message as arguments
log_message "INFO" "Health check started"
log_message "WARN" "Disk at 87% on /dev/sda1"
log_message "ERROR" "Service nginx is DOWN"
```

> **💡 Fresher Tip — chmod +x and Exit Codes**

> After creating a script file, you must make it executable: `chmod +x script.sh`. Without this, Linux won't run it. Then execute it with `./script.sh` (the ./ means "in current directory"). Every command in Linux returns an **exit code**: 0 means success, anything non-zero means failure. Check with `echo $?` after any command. In scripts, we use `exit 0` at the end for success and `exit 1` (or any non-zero) when an error occurs.

### 2. Script 1 — Server Health Monitor

**Business Problem:** InfraCore's ops team manually checks each server every morning — logging in one by one, running `df -h`, `free -m`, `top` — for 30 servers. This takes 3 engineers two hours. You need a script that checks CPU usage, memory usage, disk usage, and whether critical services are running, then writes a report and alerts if anything is above threshold.

**Scene 2 — Operations Centre, InfraCore | The Morning Ritual**

> **Deepak** _Operations Engineer — InfraCore Technologies_
> 
> Every morning at 8 AM, I SSH into server-01, run df -h, note down the disk percentages on a notepad, then SSH into server-02 and do the same. Then check if nginx is running, if MySQL is running, if our monitoring agent is running. Then do it for all 30 servers. It is 2 AM when something goes wrong and I have no idea because nobody checked since 6 PM yesterday.

> **Harish** _Senior DevOps Engineer_
> 
> Vikram, write a script that does exactly what Deepak does manually — but automatically, every 5 minutes, for every server, and logs the result. Use df to check disk, free to check memory, top to get CPU, and systemctl to check service status. If disk is above 85%, memory above 90%, or CPU above 95%, write a WARNING. If a service is down, write a CRITICAL. I want a timestamped log file after each run.

```
Script 1 — Health Check Logic Flow
=====================================

  health_check.sh runs every 5 minutes (via cron)
  │
  ├── Check CPU Usage
  │   └── top -bn1 | grep "Cpu(s)" | awk '{print $2}'
  │       ├── If CPU > 95%  → log WARNING
  │       └── If CPU OK     → log INFO
  │
  ├── Check Memory Usage
  │   └── free -m | awk '/Mem:/ {print int($3/$2 * 100)}'
  │       ├── If MEM > 90%  → log WARNING
  │       └── If MEM OK     → log INFO
  │
  ├── Check Disk Usage (each mounted partition)
  │   └── df -h | awk 'NR>1 {gsub("%","",$5); if($5>85) print $0}'
  │       ├── If DISK > 85% → log WARNING + send mail alert
  │       └── If DISK OK    → log INFO
  │
  └── Check Services: nginx, mysql, redis, sshd
      ├── systemctl is-active nginx
      │   ├── Active  → log OK
      │   └── Inactive → log CRITICAL + attempt restart
      └── Repeat for each service

  Output: /var/log/infracore/health_YYYY-MM-DD.log
```

```bash
#!/bin/bash
# =============================================================================
# Script: health_check.sh
# Purpose: Monitor server health — CPU, memory, disk, services
# Company: InfraCore Technologies
# Run by: cron every 5 minutes
# Author: Vikram (Junior DevOps Engineer)
# =============================================================================
# ── CONFIGURATION ─────────────────────────────────────────────────────────────
# Load shared config file if it exists, otherwise use defaults
CONFIG_FILE="/opt/infracore/config/infracore.conf"
[ -f "$CONFIG_FILE" ] && source "$CONFIG_FILE" # source = load the config file
LOG_DIR="/var/log/infracore"
LOG_FILE="${LOG_DIR}/health_$(date +%Y-%m-%d).log"
CPU_THRESHOLD=${CPU_THRESHOLD:-95} # use config value, or default to 95
MEM_THRESHOLD=${MEM_THRESHOLD:-90}
DISK_THRESHOLD=${DISK_THRESHOLD:-85}
SERVICES=("nginx" "mysql" "redis" "sshd")  # Bash array of services to check
ALERT_EMAIL="ops@infracore.com"
HOSTNAME=$(hostname)

# ── FUNCTIONS ──────────────────────────────────────────────────────────────────
setup_log_dir() {
    # Create log directory if it doesn't exist. -p creates parent dirs too.
mkdir -p "$LOG_DIR"
}

log_message() {
    # $1 = level (INFO/WARN/CRITICAL/OK), $2 = message
local TIMESTAMP
TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")
    echo "[$TIMESTAMP] [$1] $2" | tee -a "$LOG_FILE"
# tee -a = print to stdout AND append to log file simultaneously
}

check_cpu() {
    # top -bn1: run top in batch mode (-b), for 1 iteration (-n1), non-interactive
    # grep "Cpu(s)": find the CPU usage line
    # awk '{print $2}': print 2nd field (user CPU %)
    # The output looks like: "2.5%us" so we remove the % with tr
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | tr -d '%us,')
    CPU_INT=$(echo "$CPU_USAGE" | cut -d. -f1)  # take integer part only (85.3 → 85)
if [ "$CPU_INT" -gt "$CPU_THRESHOLD" ]; then
log_message "WARN" "CPU usage is ${CPU_USAGE}% — above ${CPU_THRESHOLD}% threshold"
send_alert "HIGH CPU" "CPU at ${CPU_USAGE}% on $HOSTNAME"
else
log_message "OK" "CPU usage: ${CPU_USAGE}% — within limit"
fi
}

check_memory() {
    # free -m: show memory in megabytes
    # awk '/Mem:/: match line containing "Mem:"
    # {print int($3/$2*100)}: calculate used/total as percentage
MEM_USAGE=$(free -m | awk '/Mem:/ {print int($3/$2*100)}')

    if [ "$MEM_USAGE" -gt "$MEM_THRESHOLD" ]; then
log_message "WARN" "Memory usage is ${MEM_USAGE}% — above ${MEM_THRESHOLD}% threshold"
send_alert "HIGH MEMORY" "Memory at ${MEM_USAGE}% on $HOSTNAME"
else
log_message "OK" "Memory usage: ${MEM_USAGE}% — within limit"
fi
}

check_disk() {
    # df -h: disk free with human-readable sizes
    # awk 'NR>1': skip the header row (row 1), process all others
    # gsub("%","",$5): remove % sign from column 5 (usage percentage)
    # if($5 > threshold): only print partitions that are over threshold
DISK_ISSUES=$(df -h | awk 'NR>1 {gsub("%","",$5); if($5+0 > '"$DISK_THRESHOLD"') print $5"% on "$6}')

    if [ -n "$DISK_ISSUES" ]; then
# -n checks if the variable is NOT empty
while IFS= read -r line; do
log_message "WARN" "Disk: $line — above ${DISK_THRESHOLD}% threshold"
done <<< "$DISK_ISSUES"
send_alert "HIGH DISK" "Disk issues on $HOSTNAME: $DISK_ISSUES"
else
log_message "OK" "All disks below ${DISK_THRESHOLD}% threshold"
fi
}

check_services() {
    # Loop through the SERVICES array and check each one
for SERVICE in "${SERVICES[@]}"; do # "${SERVICES[@]}" expands all array elements
if systemctl is-active --quiet "$SERVICE"; then
# is-active --quiet: returns exit code 0 if running, non-zero if stopped
log_message "OK" "Service $SERVICE is running"
else
log_message "CRITICAL" "Service $SERVICE is DOWN — attempting restart"
systemctl start "$SERVICE"
# Check if restart succeeded
if systemctl is-active --quiet "$SERVICE"; then
log_message "INFO" "Service $SERVICE restarted successfully"
else
log_message "CRITICAL" "Service $SERVICE FAILED to restart — manual intervention needed!"
send_alert "SERVICE DOWN" "$SERVICE failed to restart on $HOSTNAME"
fi
fi
done
}

send_alert() {
    # $1 = alert type, $2 = message
    # mail command sends email from the command line
echo "[$1] on $HOSTNAME at $(date): $2" | \
        mail -s "[InfraCore ALERT] $1 — $HOSTNAME" "$ALERT_EMAIL"
log_message "INFO" "Alert email sent to $ALERT_EMAIL"
}

# ── MAIN EXECUTION ────────────────────────────────────────────────────────────
setup_log_dir
log_message "INFO" "===== Health Check Started on $HOSTNAME ====="
check_cpu
check_memory
check_disk
check_services
log_message "INFO" "===== Health Check Complete ====="
exit 0  # exit code 0 = success
```

```
[2026-01-15 09:00:01] [INFO] ===== Health Check Started on prod-server-01 =====
[2026-01-15 09:00:02] [OK]   CPU usage: 34% — within limit
[2026-01-15 09:00:02] [OK]   Memory usage: 68% — within limit
[2026-01-15 09:00:03] [WARN] Disk: 88% on /dev/sda1 — above 85% threshold
[2026-01-15 09:00:03] [INFO] Alert email sent to ops@infracore.com
[2026-01-15 09:00:04] [OK]   Service nginx is running
[2026-01-15 09:00:04] [CRITICAL] Service redis is DOWN — attempting restart
[2026-01-15 09:00:05] [INFO] Service redis restarted successfully
[2026-01-15 09:00:05] [OK]   Service mysql is running
[2026-01-15 09:00:05] [OK]   Service sshd is running
[2026-01-15 09:00:05] [INFO] ===== Health Check Complete =====
```

### 3. Script 2 — Log Rotation & Cleanup

**Business Problem:** InfraCore's application servers write log files every day. After 30 days, the `/var/log/app/` folder has hundreds of gigabytes of logs filling up the disk. The team manually deletes old logs — but sometimes deletes the wrong files. You need a script that automatically compresses logs older than 7 days and deletes logs older than 30 days.

**Scene 3 — Server Room | Disk Full Emergency**

> **Harish** _Senior DevOps Engineer_
> 
> Vikram, prod-server-03 just went down. Root cause: disk full. /var/log/app/ has 198 GB of log files — some from 6 months ago. Nobody cleaned them up. The application couldn't write its logs, panicked, and crashed. Write a log rotation script today. I want anything older than 7 days compressed with gzip, and anything older than 30 days deleted entirely. And log every action so we know exactly what was touched.

> **Preetam** _Infrastructure Lead_
> 
> The find command is what you need. find /path -mtime +7 finds files modified more than 7 days ago. Combine it with gzip for compression. Use find again with -mtime +30 and -delete for deletion. Never run find -delete without first running it without -delete to see what it would remove. That is the rule — always dry-run first.

```bash
#!/bin/bash
# =============================================================================
# Script: log_rotation.sh
# Purpose: Compress old logs (>7 days) and delete very old logs (>30 days)
# Company: InfraCore Technologies
# =============================================================================
LOG_DIRS=("/var/log/app" "/var/log/nginx" "/var/log/mysql")
ARCHIVE_DIR="/var/log/archive"
COMPRESS_AFTER_DAYS=7   # compress logs older than this
DELETE_AFTER_DAYS=30    # delete logs older than this
SCRIPT_LOG="/var/log/infracore/log_rotation_$(date +%Y-%m-%d).log"
DRY_RUN=false # set to true to test without actually doing anything
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$SCRIPT_LOG"
}

compress_old_logs() {
    local DIR=$1
log "Compressing logs older than $COMPRESS_AFTER_DAYS days in: $DIR"
# find: search in $DIR for regular files (-type f)
    # that do NOT already end in .gz (-not -name "*.gz")
    # that were last modified more than 7 days ago (-mtime +7)
while IFS= read -r -d '' FILE; do
if [ "$DRY_RUN" = "true" ]; then
log "[DRY-RUN] Would compress: $FILE"
else
gzip "$FILE" # gzip replaces FILE with FILE.gz
log "Compressed: $FILE → ${FILE}.gz"
fi
done < <(find "$DIR" -type f -not -name "*.gz" -mtime +$COMPRESS_AFTER_DAYS -print0)
    # -print0 separates results with null character instead of newline
    # This safely handles filenames with spaces
}

move_to_archive() {
    local DIR=$1
MONTH=$(date "+%Y-%m")  # e.g. "2026-01" — archive into month folders
DEST="${ARCHIVE_DIR}/${MONTH}"
mkdir -p "$DEST"
# Move all .gz files older than COMPRESS_AFTER_DAYS to archive folder
while IFS= read -r -d '' FILE; do
if [ "$DRY_RUN" = "true" ]; then
log "[DRY-RUN] Would move: $FILE → $DEST/"
else
mv "$FILE" "$DEST/"
log "Archived: $FILE → $DEST/"
fi
done < <(find "$DIR" -type f -name "*.gz" -mtime +$COMPRESS_AFTER_DAYS -print0)
}

delete_old_archives() {
    log "Deleting archived logs older than $DELETE_AFTER_DAYS days from $ARCHIVE_DIR"
# Count how many files will be deleted BEFORE deleting (safety check)
DELETE_COUNT=$(find "$ARCHIVE_DIR" -type f -mtime +$DELETE_AFTER_DAYS | wc -l)
    log "Files to be deleted: $DELETE_COUNT"
if [ "$DRY_RUN" = "true" ]; then
log "[DRY-RUN] Would delete $DELETE_COUNT files"
else
find "$ARCHIVE_DIR" -type f -mtime +$DELETE_AFTER_DAYS -delete
        log "Deleted $DELETE_COUNT old archive files"
fi
}

report_disk_savings() {
    # du -sh: disk usage, summary (-s), human-readable (-h)
ARCHIVE_SIZE=$(du -sh "$ARCHIVE_DIR" 2>/dev/null | cut -f1)
    log "Archive directory size after rotation: ${ARCHIVE_SIZE:-0}"
log "Current disk usage:"
df -h / | tail -1 | tee -a "$SCRIPT_LOG"
}

# ── MAIN EXECUTION ────────────────────────────────────────────────────────────
mkdir -p "/var/log/infracore"
log "===== Log Rotation Started ====="
[ "$DRY_RUN" = "true" ] && log "*** DRY-RUN MODE — no files will be changed ***"
for DIR in "${LOG_DIRS[@]}"; do
if [ -d "$DIR" ]; then # -d checks if it's a directory that exists
compress_old_logs "$DIR"
move_to_archive "$DIR"
else
log "WARNING: Directory not found: $DIR — skipping"
fi
done
delete_old_archives
report_disk_savings
log "===== Log Rotation Complete ====="
exit 0
```

> **💡 Fresher Tip — What is awk?**

> **awk** is one of the most powerful text-processing tools in Linux. It reads input line by line and processes each line. `NR` means "current row number" (`NR>1` skips the header). Fields are numbered `$1, $2, $3...` ($0 is the whole line). The pattern `/Mem:/` means "only process lines containing Mem:". So `awk '/Mem:/ {print int($3/$2*100)}'` reads: "On lines containing 'Mem:', print column3 divided by column2 as a percentage." awk is used in virtually every professional shell script that processes command output.

### 4. Script 3 — Automated Backup System

**Business Problem:** InfraCore's critical data — application code, database exports, and configuration files — must be backed up daily. Currently an engineer manually runs `tar` and `rsync` commands. If they're on leave, the backup doesn't happen. You need a fully automated backup script with timestamps, verification, and error handling.

```bash
#!/bin/bash
# =============================================================================
# Script: backup.sh
# Purpose: Automated timestamped backup of critical directories to backup server
# Company: InfraCore Technologies
# =============================================================================
BACKUP_SOURCES=("/etc/nginx" "/opt/app/config" "/var/www/html")
BACKUP_DEST_LOCAL="/mnt/backup/daily"
BACKUP_SERVER="backup-01.infracore.com"
BACKUP_SERVER_PATH="/backups/prod-server"
RETENTION_DAYS=14       # keep backups for 14 days
TIMESTAMP=$(date "+%Y%m%d_%H%M%S")  # e.g. 20260115_090000
BACKUP_LOG="/var/log/infracore/backup_${TIMESTAMP}.log"
ERRORS=0  # track total error count
log() { echo "[$(date '+%H:%M:%S')] $1" | tee -a "$BACKUP_LOG"; }

create_backup() {
    local SOURCE=$1
# basename extracts the last part of a path: /opt/app/config → config
local NAME=$(basename "$SOURCE")
    local BACKUP_FILE="${BACKUP_DEST_LOCAL}/${NAME}_${TIMESTAMP}.tar.gz"
log "Backing up: $SOURCE → $BACKUP_FILE"
# tar: tape archive (despite the name, it's how Linux compresses things)
    # -c: create archive  -z: compress with gzip  -f: output filename
    # 2>&1: redirect stderr to stdout so errors appear in our log
if tar -czf "$BACKUP_FILE" "$SOURCE" 2>&1 | tee -a "$BACKUP_LOG"; then
FILESIZE=$(du -sh "$BACKUP_FILE" | cut -f1)
        log "Backup created: $BACKUP_FILE ($FILESIZE)"
# Create a checksum file to verify integrity later
sha256sum "$BACKUP_FILE" > "${BACKUP_FILE}.sha256"
log "Checksum written: ${BACKUP_FILE}.sha256"
else
log "ERROR: Backup FAILED for $SOURCE"
ERRORS=$(( ERRORS + 1 ))
    fi
}

sync_to_backup_server() {
    log "Syncing backups to $BACKUP_SERVER..."
# rsync: efficient file sync tool
    # -a: archive mode (preserves permissions, timestamps, etc.)
    # -v: verbose output  --progress: show transfer progress
    # --delete: remove files from destination that no longer exist at source
if rsync -av --progress "$BACKUP_DEST_LOCAL/" \
            "backup-user@${BACKUP_SERVER}:${BACKUP_SERVER_PATH}/" \
            >> "$BACKUP_LOG" 2>&1; then
log "Sync to backup server: SUCCESS"
else
log "ERROR: rsync to backup server FAILED"
ERRORS=$(( ERRORS + 1 ))
    fi
}

cleanup_old_backups() {
    log "Removing backups older than $RETENTION_DAYS days..."
REMOVED=$(find "$BACKUP_DEST_LOCAL" -type f -mtime +$RETENTION_DAYS | wc -l)
    find "$BACKUP_DEST_LOCAL" -type f -mtime +$RETENTION_DAYS -delete
    log "Removed $REMOVED old backup files"
}

verify_backups() {
    log "Verifying backup checksums..."
for CHECKSUM_FILE in "$BACKUP_DEST_LOCAL"/*.sha256; do
        [ -f "$CHECKSUM_FILE" ] || continue # skip if no .sha256 files found
if sha256sum -c "$CHECKSUM_FILE" --quiet 2>&1; then
log "Checksum OK: $(basename $CHECKSUM_FILE)"
else
log "CHECKSUM MISMATCH: $(basename $CHECKSUM_FILE) — backup may be corrupt!"
ERRORS=$(( ERRORS + 1 ))
        fi
done
}

# ── MAIN ──────────────────────────────────────────────────────────────────────
mkdir -p "$BACKUP_DEST_LOCAL" "/var/log/infracore"
log "===== InfraCore Backup Started — $TIMESTAMP ====="
for SOURCE in "${BACKUP_SOURCES[@]}"; do
create_backup "$SOURCE"
done
verify_backups
sync_to_backup_server
cleanup_old_backups
log "===== Backup Complete — Errors: $ERRORS ====="
[ "$ERRORS" -gt 0 ] && exit 1 || exit 0  # exit 1 if any errors occurred
```

### 5. Script 4 — User Account Manager

**Business Problem:** InfraCore hires 5–10 new engineers every month. The sysadmin manually creates each account: `useradd`, sets password, assigns group, creates home directory, sets SSH key. With 8 new hires this month, that's 8 × 10 commands = 80 manual commands, each one typed carefully. One typo means a wrong group or wrong home directory. A script reads from a CSV file and does it all automatically.

**Scene 4 — Sysadmin Desk, InfraCore | New Batch Onboarding**

> **Geetha** _Linux Sysadmin — InfraCore Technologies_
> 
> We have 8 new engineers starting Monday. HR gave me a CSV with their names, usernames, and which group they belong to — dev, ops, or qa. I have to create all 8 accounts by Sunday evening. Last time I did this manually I gave one person the wrong group and they had root-level access they shouldn't have. Can we automate this from the CSV?

> **Harish** _Senior DevOps Engineer_
> 
> Vikram, write a user management script. It reads a CSV with columns: username, fullname, group, email. For each row: create the group if it doesn't exist, create the user with useradd, set an initial password, force them to change it on first login with chage -d 0, and log every action. Add a disable mode too — when someone leaves, call the script with --disable username and it locks the account immediately.

```
Input: data/new_users.csv
==========================
username,fullname,group,email
vikram_k,Vikram Kumar,dev,vikram.k@infracore.com
sneha_r,Sneha Reddy,ops,sneha.r@infracore.com
ajith_m,Ajith Menon,qa,ajith.m@infracore.com

Script 4 — User Creation Flow:
================================
  For each CSV row:
  │
  ├── Does group exist?  No → groupadd [groupname]
  │
  ├── Does user exist?   Yes → log "already exists, skip"
  │
  ├── Create user:
  │   useradd -m -g [group] -c "[fullname]" -s /bin/bash [username]
  │   (-m = create home dir, -g = primary group, -c = comment/fullname)
  │
  ├── Set initial password (username + "@InfraCore2026")
  │   echo "[username]:InfraCore2026@" | chpasswd
  │
  ├── Force password change on first login:
  │   chage -d 0 [username]
  │
  └── Log: "Created user [username] in group [group]"
```

```bash
#!/bin/bash
# =============================================================================
# Script: user_manager.sh
# Purpose: Create/disable user accounts from a CSV file
# Usage:   ./user_manager.sh --create data/new_users.csv
#          ./user_manager.sh --disable username
# Company: InfraCore Technologies
# =============================================================================
USER_LOG="/var/log/infracore/user_management.log"
DEFAULT_SHELL="/bin/bash"
INITIAL_PASS_SUFFIX="@InfraCore2026"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$USER_LOG"; }

ensure_group_exists() {
    local GROUP=$1
# getent group checks if a group exists in the system
if ! getent group "$GROUP" >/dev/null 2>&1; then
# >/dev/null 2>&1 silences all output (stdout and stderr)
groupadd "$GROUP"
log "Created group: $GROUP"
else
log "Group already exists: $GROUP"
fi
}

create_user() {
    local USERNAME=$1
local FULLNAME=$2
local GROUP=$3
local EMAIL=$4
# id command: checks if user already exists; returns non-zero if not found
if id "$USERNAME" >/dev/null 2>&1; then
log "SKIP: User '$USERNAME' already exists"
return 0
    fi
ensure_group_exists "$GROUP"
# useradd: creates the user account
    # -m: create home directory at /home/username
    # -g: primary group
    # -c: comment (usually full name)
    # -s: login shell
if useradd -m -g "$GROUP" -c "$FULLNAME" -s "$DEFAULT_SHELL" "$USERNAME"; then
log "Created user: $USERNAME (Group: $GROUP, Name: $FULLNAME)"
# Set initial password: "username@InfraCore2026"
        # chpasswd reads "username:password" from stdin via pipe (|)
INITIAL_PASS="${USERNAME}${INITIAL_PASS_SUFFIX}"
echo "${USERNAME}:${INITIAL_PASS}" | chpasswd
log "Password set for: $USERNAME"
# Force password change on first login (-d 0 sets last-change date to epoch)
chage -d 0 "$USERNAME"
log "Forced password change on first login for: $USERNAME"
# Create .ssh directory for future SSH key deployment
mkdir -p "/home/${USERNAME}/.ssh"
chmod 700 "/home/${USERNAME}/.ssh"
chown -R "${USERNAME}:${GROUP}" "/home/${USERNAME}/.ssh"
log "SSH directory created for: $USERNAME"
else
log "ERROR: Failed to create user: $USERNAME"
return 1
    fi
}

process_csv() {
    local CSV_FILE=$1
if [ ! -f "$CSV_FILE" ]; then
log "ERROR: CSV file not found: $CSV_FILE"
exit 1
    fi
LINE_NUM=0
    # Read CSV line by line. IFS=, sets comma as field separator.
while IFS=, read -r USERNAME FULLNAME GROUP EMAIL; do
LINE_NUM=$(( LINE_NUM + 1 ))

        # Skip the header row (line 1)
        [ "$LINE_NUM" -eq 1 ] && continue
# Skip empty lines
        [ -z "$USERNAME" ] && continue
# trim whitespace from variables using parameter substitution
USERNAME=$(echo "$USERNAME" | tr -d ' \r')
        GROUP=$(echo "$GROUP" | tr -d ' \r')

        create_user "$USERNAME" "$FULLNAME" "$GROUP" "$EMAIL"
done < "$CSV_FILE" # < reads lines from the file into the while loop
}

disable_user() {
    local USERNAME=$1
if ! id "$USERNAME" >/dev/null 2>&1; then
log "ERROR: User '$USERNAME' does not exist"
exit 1
    fi
# usermod -L: lock the account (add ! to password hash — login impossible)
    # usermod -e 1: expire account immediately (belt-and-suspenders approach)
usermod -L -e 1 "$USERNAME"
log "Account DISABLED: $USERNAME (locked + expired)"
# Kill all active sessions for this user
pkill -u "$USERNAME" 2>/dev/null
    log "All sessions terminated for: $USERNAME"
}

# ── ARGUMENT PARSING ─────────────────────────────────────────────────────────
# $1 is the first argument given when running the script
# e.g.: ./user_manager.sh --create data/new_users.csv
mkdir -p "/var/log/infracore"
log "===== User Management Script Started ====="
case "$1" in
    --create)
        [ -z "$2" ] && { log "Usage: $0 --create <csv_file>"; exit 1; }
        process_csv "$2"
        ;;
    --disable)
        [ -z "$2" ] && { log "Usage: $0 --disable <username>"; exit 1; }
        disable_user "$2"
        ;;
    *)
        echo "Usage: $0 --create <csv_file> | --disable <username>"
exit 1
        ;;
esac
log "===== User Management Complete ====="
exit 0
```

### 6. Script 5 — Application Deployment Script

**Business Problem:** Deploying a new version of InfraCore's internal portal involves: pulling code from Git, building, stopping the old service, copying files, restarting, and testing. Done manually by developers, this takes 20 minutes and sometimes leaves the server in a broken state when a step is skipped. A deployment script does it in 90 seconds — and automatically rolls back if anything fails.

**Scene 5 — Dev War Room | The Failed Deployment**

> **Rajan** _CTO — InfraCore Technologies_
> 
> Last Friday's deployment was a disaster. The developer forgot to stop the old service before copying new files. The new files were half-copied when the old service tried to read them. The app crashed for 45 minutes. We lost 3000 user sessions. I need a deployment script that: stops the service first, deploys atomically, starts it back up, and — most importantly — rolls back automatically if the new version doesn't start within 30 seconds.

> **Harish** _Senior DevOps Engineer_
> 
> Vikram, this is your most important script. The rollback mechanism is the key: before deploying, keep a copy of the current version. If the new version doesn't respond to health check within 30 seconds, restore the old version automatically and exit with an error code. The pipeline catches the non-zero exit code and sends an alert. Zero downtime, automatic recovery.

```
Script 5 — Deployment Flow with Auto-Rollback
================================================

  ./deploy.sh --version v2.3.1
        │
        ├── 1. Pre-checks (disk space, Git access, service exists)
        │
        ├── 2. Create backup of CURRENT version
        │        cp -r /opt/app /opt/app_backup_20260115_090000
        │
        ├── 3. Pull new code from Git
        │        git pull origin main / git checkout v2.3.1
        │
        ├── 4. Stop current service
        │        systemctl stop infracore-portal
        │
        ├── 5. Deploy new files (atomic swap using mv)
        │        mv /opt/app_new /opt/app
        │
        ├── 6. Start service
        │        systemctl start infracore-portal
        │
        ├── 7. Health check (retry 6 times, 5s apart = 30s total)
        │        curl -sf http://localhost:8080/health
        │        │
        │        ├── 200 OK → Deployment SUCCESS ✓
        │        │
        │        └── Timeout/Error → ROLLBACK!
        │               ├── systemctl stop infracore-portal
        │               ├── mv /opt/app_backup → /opt/app
        │               ├── systemctl start infracore-portal
        │               └── exit 1 → pipeline alert fires    
```

```bash
#!/bin/bash
# =============================================================================
# Script: deploy.sh
# Purpose: Deploy application with automatic rollback on failure
# Usage:   ./deploy.sh --version v2.3.1
#          ./deploy.sh --rollback  (manual rollback)
# Company: InfraCore Technologies
# =============================================================================
APP_DIR="/opt/infracore-portal"
APP_REPO="git@github.com:infracore/portal.git"
APP_TEMP="/opt/infracore-portal-temp"
APP_BACKUP="/opt/infracore-portal-backup"
SERVICE_NAME="infracore-portal"
HEALTH_URL="http://localhost:8080/health"
HEALTH_RETRIES=6
HEALTH_INTERVAL=5    # seconds between health check retries
DEPLOY_LOG="/var/log/infracore/deploy_$(date +%Y%m%d_%H%M%S).log"
VERSION="latest"
log() {
    echo "[$(date '+%H:%M:%S')] $1" | tee -a "$DEPLOY_LOG"
}

abort() {
    # Called on any fatal error. Logs the reason and exits with code 1.
log "ABORT: $1"
exit 1
}

pre_flight_checks() {
    log "Running pre-flight checks..."
# Check disk space — need at least 1GB free (1048576 KB)
FREE_KB=$(df /opt | awk 'NR==2 {print $4}')
    [ "$FREE_KB" -lt 1048576 ] && abort "Insufficient disk space (need 1GB, have $(( FREE_KB/1024 ))MB)"
# Check Git is accessible
git ls-remote "$APP_REPO" >/dev/null 2>&1 || abort "Cannot reach Git repo: $APP_REPO"
# Check if the service unit file exists (can we manage it with systemctl?)
systemctl list-unit-files "$SERVICE_NAME.service" | grep -q "$SERVICE_NAME" \
        || abort "Service unit '$SERVICE_NAME' not found"
log "Pre-flight checks passed"
}

backup_current_version() {
    if [ -d "$APP_DIR" ]; then
log "Backing up current version to $APP_BACKUP..."
rm -rf "$APP_BACKUP" # remove old backup first
cp -r "$APP_DIR" "$APP_BACKUP" # copy current to backup
log "Current version backed up"
fi
}

pull_new_code() {
    log "Cloning version $VERSION from $APP_REPO..."
rm -rf "$APP_TEMP"
if ! git clone --depth 1 --branch "$VERSION" "$APP_REPO" "$APP_TEMP"; then
abort "Git clone failed for version $VERSION"
fi
log "Code cloned: $VERSION"
}

stop_service() {
    log "Stopping service: $SERVICE_NAME"
systemctl stop "$SERVICE_NAME"
sleep 2  # give the service 2 seconds to fully stop
log "Service stopped"
}

deploy_new_version() {
    log "Deploying new version..."
# Atomic swap: mv is atomic on the same filesystem (near-instantaneous)
    # This avoids the half-deployed state that causes the type of crash Rajan described
rm -rf "$APP_DIR"
mv "$APP_TEMP" "$APP_DIR"
log "New version deployed to $APP_DIR"
}

start_service() {
    log "Starting service: $SERVICE_NAME"
if ! systemctl start "$SERVICE_NAME"; then
log "ERROR: systemctl start failed — initiating rollback"
rollback
fi
log "Service start command issued"
}

health_check() {
    log "Running health checks (${HEALTH_RETRIES} attempts, ${HEALTH_INTERVAL}s apart)..."
for i in $(seq 1 $HEALTH_RETRIES); do # seq generates: 1 2 3 4 5 6
log "Health check attempt $i/$HEALTH_RETRIES..."
# curl -sf: silent (-s) and fail (-f) on HTTP errors
        # -o /dev/null: discard response body, only care about exit code
        # --max-time 5: timeout after 5 seconds
if curl -sf -o /dev/null --max-time 5 "$HEALTH_URL"; then
log "Health check PASSED on attempt $i"
return 0  # return 0 = success from this function
fi
log "Attempt $i failed — waiting ${HEALTH_INTERVAL}s..."
sleep "$HEALTH_INTERVAL"
done
# If we reach here, all attempts failed
log "Health check FAILED after $HEALTH_RETRIES attempts — initiating rollback"
rollback
}

rollback() {
    log "===== ROLLBACK INITIATED ====="
if [ ! -d "$APP_BACKUP" ]; then
abort "No backup found at $APP_BACKUP — cannot rollback!"
fi
systemctl stop "$SERVICE_NAME" 2>/dev/null
    rm -rf "$APP_DIR"
cp -r "$APP_BACKUP" "$APP_DIR"
systemctl start "$SERVICE_NAME"
log "Previous version restored from backup"
if systemctl is-active --quiet "$SERVICE_NAME"; then
log "Rollback SUCCESSFUL — old version is running"
else
log "CRITICAL: Rollback FAILED — service won't start with old version either!"
fi
exit 1  # exit 1 tells the CI/CD pipeline that deployment failed
}

# ── ARGUMENT PARSING ─────────────────────────────────────────────────────────
mkdir -p "/var/log/infracore"
case "$1" in
    --version)
        [ -z "$2" ] && abort "Specify a version: --version v2.3.1"
VERSION="$2"
        ;;
    --rollback)
        log "Manual rollback requested"
rollback
        ;;
    *)
        echo "Usage: $0 --version <tag> | --rollback"
exit 1
        ;;
esac
log "===== Deployment Started: $VERSION on $(hostname) ====="
pre_flight_checks
backup_current_version
pull_new_code
stop_service
deploy_new_version
start_service
health_check
log "===== Deployment SUCCESSFUL: $VERSION ====="
exit 0
```

### 7. Scheduling with Cron — Run Scripts Automatically

Writing scripts is only useful if they run automatically. On Linux servers, **cron** is the scheduler. It reads a configuration file called the **crontab** and runs specified commands at scheduled times. Every sysadmin and DevOps engineer must know how to write cron jobs.

> **Cron Schedule Format — The 5 Fields**

> Every cron entry has 5 time fields followed by the command. Reading left to right: **minute (0-59)**, **hour (0-23)**, **day of month (1-31)**, **month (1-12)**, **day of week (0-7, where 0 and 7 = Sunday)**. A `*` means "every". So `*/5 * * * *` means "every 5 minutes". `0 9 * * 1-5` means "9:00 AM, Monday to Friday".

```
# InfraCore Automation Crontab
# Edit with: crontab -e
# View current: crontab -l
# Format: minute  hour  day-of-month  month  day-of-week  command
# ─────────────────────────────────────────────────────────────────
# *       = every (minute/hour/day etc.)
# */5     = every 5 (units)
# 0       = at exactly the 0th minute (i.e. at the start of the hour)
# 1-5     = Monday to Friday
# 0,6     = Sunday and Saturday
# ── Script 1: Health Check — every 5 minutes, 24/7
*/5 * * * * /opt/infracore/scripts/health_check.sh >> /var/log/infracore/cron.log 2>&1

# ── Script 2: Log Rotation — every night at 2 AM (off-peak hours)
0 2 * * * /opt/infracore/scripts/log_rotation.sh >> /var/log/infracore/cron.log 2>&1

# ── Script 3: Backup — every day at 3 AM (after log rotation finishes)
0 3 * * * /opt/infracore/scripts/backup.sh >> /var/log/infracore/cron.log 2>&1

# ── Weekly disk usage report — every Monday at 8 AM
0 8 * * 1 df -h | mail -s "InfraCore Weekly Disk Report - $(hostname)" ops@infracore.com

# ── Monthly cleanup of old script logs (older than 90 days) — 1st of each month
0 4 1 * * find /var/log/infracore -type f -mtime +90 -delete

# ── Useful cron commands:
# crontab -e         → open crontab editor for current user
# crontab -l         → list current user's crontab
# crontab -r         → remove all crontabs (careful!)
# sudo crontab -e    → edit root's crontab (for scripts needing root)
# cat /etc/cron.d/   → system-wide cron jobs
# journalctl -u cron → view cron service logs
```

> **💡 Fresher Tip — Always Use Full Paths in Cron**

> Cron runs in a very minimal environment — it doesn't load your `.bashrc` or set up the same `$PATH` you have when logged in interactively. This is why cron jobs fail even though the same command works when you type it manually. The solution: always use **full absolute paths** in cron jobs. Write `/usr/bin/find` not just `find`. Write `/opt/infracore/scripts/backup.sh` not just `backup.sh`. Also add `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` at the top of your crontab as a safety net.

### 8. Essential Commands Reference for Shell Scripting

These are the Linux commands that appear most often in real company shell scripts. Know what each one does and when to use it — these are frequently tested in DevOps interviews.

Command

What It Does

Real Use in Scripts

grep

Search for text patterns in files or output

grep "ERROR" app.log — find error lines

awk

Process text column by column, row by row

awk '{print $5}' — extract 5th field from df output

sed

Stream editor — find & replace text in files

sed -i 's/old/new/g' config.conf — replace in-place

find

Search filesystem for files matching criteria

find /log -mtime +7 -name "*.log" — logs older than 7 days

xargs

Convert stdin into command arguments

find . -name "*.tmp" | xargs rm — delete found files

cut

Extract specific fields from lines

cut -d: -f1 /etc/passwd — extract usernames

wc

Word/line/character count

grep "ERROR" app.log | wc -l — count error lines

sort

Sort lines alphabetically or numerically

sort -rn — reverse numeric sort (largest first)

uniq

Remove or count duplicate lines

sort file | uniq -c | sort -rn — count occurrences

tail

Show last N lines of a file

tail -f /var/log/syslog — follow live log output

head

Show first N lines of a file

head -5 file.csv — see first 5 lines of CSV

tee

Read from stdin, write to stdout AND file

command | tee -a logfile.txt — log and display output

df

Disk free — show disk usage per partition

df -h — human-readable disk space summary

du

Disk usage — show size of directories/files

du -sh /var/log/* | sort -h — largest log dirs

free

Show RAM and swap usage

free -m — memory in megabytes

top/htop

Real-time CPU and memory usage

top -bn1 — single batch snapshot for scripting

ps

Process status — list running processes

ps aux | grep nginx — is nginx really running?

kill/pkill

Send signals to processes

pkill -u olduser — kill all processes of a user

curl

Make HTTP requests from command line

curl -sf http://localhost/health — check if app responds

rsync

Efficient file sync between directories or servers

rsync -av /source/ user@server:/dest/ — sync files

tar

Create and extract compressed archives

tar -czf backup.tar.gz /opt/app — create gzip archive

chmod

Change file permissions

chmod +x script.sh — make script executable

chown

Change file owner and group

chown -R app:app /opt/app — fix ownership recursively

systemctl

Control systemd services

systemctl status nginx — is it running? start/stop/restart

### 9. Debugging Shell Scripts — When Things Break

Shell scripts are particularly easy to break and sometimes hard to debug because errors can be silent. These tools and techniques will save you hours when a script isn't working as expected.

```bash
#!/bin/bash
# ── DEBUGGING TECHNIQUES ──────────────────────────────────────────────────────

# 1. BASH DEBUG MODE: Run with -x flag to see every command as it executes
#    bash -x health_check.sh
#    OR add 'set -x' at the top of the script to always debug

# 2. STRICT MODE: Add these 3 lines after #!/bin/bash in every script:
set -e   # Exit immediately if ANY command fails (non-zero exit code)
set -u   # Treat unset variables as errors (catches typos like $DISK_THREESHOLD)
set -o pipefail  # Catch failures in piped commands (cmd1 | cmd2 — if cmd1 fails, catch it)

# 3. CHECK $? — the exit code of the last command (0=success, non-zero=error)
df -h /nonexistent 2>/dev/null
echo "Exit code: $?"   # Prints: Exit code: 1 (failure)

# 4. PRINT VARIABLE VALUES — add echo statements to see what's inside variables
echo "DEBUG: CPU_USAGE=$CPU_USAGE"
echo "DEBUG: FILES=$(ls /var/log/ | wc -l)"
echo "DEBUG: Running as user: $(whoami)"

# 5. TEST WITHOUT DOING — add a DRY_RUN mode to every script
DRY_RUN=true
if [ "$DRY_RUN" = "true" ]; then
    echo "[DRY-RUN] Would delete: $FILE"
else
    rm "$FILE"
fi

# 6. CHECK IF COMMAND EXISTS before using it
if ! command -v rsync >/dev/null 2>&1; then
    echo "ERROR: rsync is not installed. Run: apt-get install rsync"
    exit 1
fi

# 7. TRAP — catch errors and clean up automatically
cleanup() {
    echo "Script interrupted or failed — cleaning up temp files..."
    rm -rf "$APP_TEMP" 2>/dev/null
}
trap cleanup EXIT   # cleanup() runs whenever the script exits (success OR failure)
trap cleanup INT    # also runs on Ctrl+C

# 8. SHELLCHECK — install this linter to catch common mistakes BEFORE running
#    sudo apt-get install shellcheck
#    shellcheck health_check.sh   ← will show warnings and suggestions
```

**Common Shell Scripting Mistakes & How to Fix Them**

- **🔴 Spaces around = in variables** — **Wrong:** `NAME = "Vikram"`  
**Right:** `NAME="Vikram"`  
Bash interprets spaces as command separators. With spaces, it tries to run a command called "NAME".

- **🔴 Forgetting quotes around variables** — **Wrong:** `rm $FILE`  
**Right:** `rm "$FILE"`  
If $FILE contains spaces (e.g. "my file.txt"), the unquoted version becomes two arguments: rm my file.txt → deletes "my" and "file.txt" separately.

- **🔴 Using == instead of = for strings** — **Wrong:** `if [ "$X" == "yes" ]`  
**Right:** `if [ "$X" = "yes" ]`  
Double == works in [[ ]] (bash-specific) but not in POSIX [ ]. Use single = in [ ] for portability.

- **🔴 Not making scripts executable** — **Wrong:** Running `./script.sh` and getting "Permission denied"  
**Right:** Run `chmod +x script.sh` first. Without execute permission, the OS refuses to run the file.

- **🔴 Hardcoding paths that only work on one machine** — **Wrong:** `/home/vikram/scripts/backup.sh`  
**Right:** Store in a config variable or use `/opt/infracore/` standard path. Scripts must work when run as root via cron.

- **🔴 No error handling after critical commands** — **Wrong:** Just running `tar -czf backup.tar.gz /opt/app`  
**Right:** Always check exit code: `if tar ...; then log "OK"; else log "FAILED"; exit 1; fi`

### 10. Interview Questions — Shell Scripting

These are the questions you will face in Linux/DevOps engineer interviews. The answers are based on concepts from the scripts you built in this project.

##### Interview Q&A — Fresher Level (0–1 Year Experience)

**Q: Q1. What is the difference between ./ and just typing the script name to run it?**

A: When you type `./script.sh`, the `./` tells the shell to look for `script.sh` in the current directory. Without `./`, the shell searches only through the directories listed in your `$PATH` variable (like `/usr/bin`, `/usr/local/bin`). Since the current directory is not in `$PATH` by default (for security reasons), typing just `script.sh` will fail with "command not found". You must either use `./script.sh` or provide the full absolute path like `/opt/infracore/scripts/script.sh`.

**Q: Q2. What does 2>&1 mean and when do you use it?**

A: In Linux, every program has three standard streams: stdin (0 = input), stdout (1 = normal output), stderr (2 = error output). By default, stdout and stderr go to different places. `2>&1` means "redirect stderr (2) to wherever stdout (1) is currently going". So `command > output.log 2>&1` sends both normal output and error messages to the same file. This is essential in cron jobs because cron cannot display errors on screen — without `2>&1`, error messages from your script would be silently lost and you'd never know why it failed.

**Q: Q3. What is the difference between single quotes ' ' and double quotes " " in bash?**

A: Double quotes `"$VAR"` allow variable expansion — variables inside are replaced with their values. Single quotes `'$VAR'` treat everything literally — `$VAR` is not expanded and appears exactly as typed. Example: if `NAME="Vikram"`, then `echo "Hello $NAME"` prints `Hello Vikram`, but `echo 'Hello $NAME'` prints `Hello $NAME`. Use double quotes around variables in scripts (always quote `"$VAR"` to handle spaces). Use single quotes when you want literal strings, like in sed patterns: `sed 's/$literal/text/'`.

**Q: Q4. What is the purpose of set -e and set -u in a shell script?**

A: `set -e` (errexit) makes the script exit immediately when any command returns a non-zero exit code, instead of continuing with the next line. Without it, a failed `tar` command would still run the `rsync` afterwards — syncing nothing. `set -u` (nounset) causes the script to exit if you try to use an undefined variable — this catches typos like `$DISK_THRESOLD` (missing an H) which would silently evaluate to empty string and cause confusing behavior. Adding both at the top of every script is a professional best practice that prevents entire classes of subtle bugs.

**Q: Q5. How do you read a CSV file line by line in bash?**

A: Use a `while` loop with `read` and set `IFS` (Internal Field Separator) to a comma: `while IFS=, read -r col1 col2 col3; do ... done < file.csv`. The `read` command reads one line at a time and splits it into variables at the IFS delimiter. The `-r` flag prevents backslash interpretation. Always skip the header row by checking `$LINE_NUM -eq 1` and using `continue` to skip to the next iteration. In our user_manager.sh script, we read username, fullname, group, and email from each CSV row and created accounts automatically — this is a pattern used in real onboarding automation at companies.

**Q: Q6. What is the exit code of a script and why does it matter for CI/CD pipelines?**

A: Every script exits with a numeric code: 0 means success, any non-zero number (1, 2, 127, etc.) means failure. Access the last command's exit code with `$?`. In CI/CD pipelines (Jenkins, GitLab CI, GitHub Actions), the pipeline checks the exit code of every step — if a step exits with non-zero, the pipeline stops and marks the build as failed. This is why our deploy.sh exits with `exit 1` when rollback occurs — it tells the pipeline "deployment failed, don't proceed to the next stage". Without proper exit codes, a completely failed deployment could look like a success to the pipeline.

**Quiz: Quiz 1 — What does the following command do? find /var/log -type f -mtime +7 -name "*.log" | wc -l**

- A) Deletes all .log files older than 7 days in /var/log
- B) Counts how many .log files in /var/log were modified more than 7 days ago
- C) Lists .log files and waits 7 minutes before showing count
- D) Shows the 7 largest log files

> **Answer/explanation:** ✅ Answer: B. `find /var/log` searches in that directory. `-type f` means files only (not directories). `-mtime +7` means modified more than 7 days ago. `-name "*.log"` matches only .log files. The output (a list of matching file paths) is piped into `wc -l` which counts the number of lines — giving us the count of matching files. No files are deleted — only `-delete` or `rm` would delete them.

**Quiz: Quiz 2 — In our health_check.sh script, why do we use systemctl is-active --quiet instead of just systemctl status?**

- A) is-active is faster than status
- B) is-active returns an exit code (0 if running, non-zero if not) that if-else can directly use; status just prints text that we'd have to parse
- C) status doesn't work inside scripts
- D) --quiet flag is required for all systemctl commands

> **Answer/explanation:** ✅ Answer: B. `systemctl is-active --quiet SERVICE` returns exit code 0 if the service is active, and exit code 3 if it's not. Bash if-else directly tests exit codes — so `if systemctl is-active --quiet nginx; then` reads perfectly: "if nginx is running, then...". Using `systemctl status nginx` would print human-readable text that you'd have to grep through. Exit codes are the correct way to check conditions in shell scripts — always prefer commands that communicate via exit codes over commands that communicate via text output.

**Quiz: Quiz 3 — What is wrong with this line in a shell script? if [ $DISK_USAGE > 85 ]; then**

- A) Nothing — it is correct bash syntax
- B) The > is a file redirection operator in bash — it creates a file named "85" instead of comparing. Use -gt for numeric comparison.
- C) $DISK_USAGE needs double quotes: "$DISK_USAGE"
- D) The if statement is missing fi at the end

> **Answer/explanation:** ✅ Answer: B. In bash, `>` inside single brackets [ ] is treated as output redirection, not greater-than comparison. It would create a file named "85" in your current directory. The correct operator for numeric greater-than in bash is `-gt`. The corrected line is: `if [ "$DISK_USAGE" -gt 85 ]; then`. For string comparisons: use `=` and `!=`. For numeric comparisons: use `-eq, -ne, -lt, -le, -gt, -ge`. This is one of the most common mistakes freshers make in bash scripts.

> **Shell Scripting Project — Core Takeaways for Freshers**

> - Every shell script must start with #!/bin/bash (the shebang), use full absolute paths, and exit with code 0 for success or non-zero for failure — these three rules are non-negotiable in professional scripts.
> - Always add set -euo pipefail after the shebang — it makes your script exit on any error, catch undefined variables, and catch failures in pipes. Scripts without this hide bugs silently.
> - Always log what your script does — timestamp, action taken, result. Console output disappears. Files persist. In production, the log file is the only way to know what happened at 3 AM.
> - Use functions to organise your script — one function for logging, one for each major task. A 200-line script with named functions is maintainable. A 200-line script with no functions is unreadable.
> - Always quote your variables: "$VAR" not $VAR. Unquoted variables break when the value contains spaces, and they silently do the wrong thing instead of throwing an error.
> - Test scripts with DRY_RUN=true before running on production data — especially any script that deletes, moves, or modifies files. One wrong find -delete command can destroy weeks of logs in milliseconds.
> - Cron jobs run in a minimal environment — always use full paths for both scripts and commands inside scripts. Test a cron job by temporarily running it as a one-minute job and checking the output before setting the real schedule.
> - The exit code is the contract between your script and the system calling it — pipelines, cron, monitoring tools all rely on exit codes. A script that always exits 0 regardless of what happened is useless in automation.

##### Shell Script Writing Standards — InfraCore Team Rules

- Name scripts with verbs that describe the action: backup.sh, deploy.sh, health_check.sh — not script1.sh or test.sh
- Store all configurable values as variables at the top of the script — never bury magic numbers like 85 or 30 inside if-else blocks deep in the code
- Every destructive operation (delete, disable, deploy) must log what it's about to do BEFORE doing it — so if the script is interrupted mid-way, you can read the log and know exactly what ran and what didn't
- Run shellcheck on every script before deploying: shellcheck script.sh — it catches 90% of common bash mistakes that compile-time checking would catch in other languages
- Scripts that make changes to the system should only be runnable by the correct user — check $(whoami) at the top and exit if not root (for system scripts) or if run as root (for application scripts)
- When removing files, always calculate and log the count BEFORE deleting — "Deleting 47 files" in the log is informative; "Deleted" with no count is useless for auditing
- Add a usage() function and check for --help argument — other engineers will use your scripts and need to know the correct syntax without reading the entire code

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Extend Script 1:** Add network connectivity checking to the health monitor — ping the gateway and an external DNS server (8.8.8.8). If both fail, log "CRITICAL: Network DOWN". If only external fails, log "WARN: Internet connectivity lost — internal network OK". Use the `ping -c 1 -W 2` command (1 packet, 2-second timeout).
2. **Extend Script 2:** Before deleting any file in log_rotation.sh, add a size check — only compress files larger than 1MB (1024KB). Files smaller than 1MB don't benefit much from compression. Use `du -k "$FILE" | cut -f1` to get the file size in kilobytes.
3. **Extend Script 3:** Add a weekly backup summary email — count total backups created this week, total size, and list the most recent backup for each source directory. Send it every Sunday at 6 PM using cron and the `mail` command.
4. **Extend Script 4:** Add an audit report mode — `./user_manager.sh --audit` should list all users in the dev, ops, and qa groups with their last login time (use `last -n 1 username` to get last login). Write the report to a file with today's date in the name.
5. **Bonus Challenge:** Write a Script 6 — SSL certificate expiry checker. For each domain in a list, use `echo | openssl s_client -connect domain:443 2>/dev/null | openssl x509 -noout -enddate` to get the expiry date, calculate days remaining, and send an alert if any certificate expires within 30 days. SSL expiry is a real production problem that companies have — your script prevents it.

### Shell Scripting Project Complete 🎉

You have designed and built five production-quality Bash scripts — Server Health Monitor, Log Rotation & Cleanup, Automated Backup System, User Account Manager, and Application Deployment with Auto-Rollback — the exact type of automation that DevOps and Linux engineers build at companies every day.

> **Harish**
> 
> "Vikram, when you joined two weeks ago you could cd into a directory and run ls. Today you have written scripts that are running on production servers, checking health every 5 minutes, rotating logs every night, and backing up our data every morning. That is exactly what a Junior DevOps engineer should be capable of after their first project. More importantly — you understand WHY each script exists. That understanding is what separates engineers who maintain systems from engineers who build them."

> **Preetam**
> 
> "The deployment script with automatic rollback prevented what would have been a 2-hour outage last Thursday. The new version's health check failed, the script rolled back in 45 seconds, and users never noticed. Rajan called it the best change we made this quarter. A shell script did that."

> **Geetha**
> 
> "And the user management script? I ran it for Monday's batch of 8 new engineers in 12 seconds. All 8 accounts created, correct groups, forced password change, SSH directories ready. Previously that took me 45 minutes and one mistake per batch. Shell scripting is the most underrated skill in DevOps — and you have it."

> **Next: Advanced Shell Scripting — Signals, Parallel Execution & Production Patterns**

> - Signal handling — trap SIGTERM, SIGINT, SIGHUP for graceful shutdown in long-running scripts
> - Parallel execution — run multiple checks or backups simultaneously with & and wait for performance
> - Here documents (heredoc) — write multi-line content to files and commands without external files
> - Regular expressions with grep -E and sed — advanced text processing for log analysis
> - Script locking with flock — prevent two cron instances of the same script running simultaneously
> - Integrating with APIs — use curl with JSON parsing via jq to send Slack/Teams alerts from shell scripts
> - Writing reusable function libraries — source a shared functions.sh file across all your scripts
