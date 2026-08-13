# Python Scripting Project Mastery

> **👋 Hey Fresher — Read This First!**

> This project is designed specially for you. You don't need to know everything about Python before starting. We will explain every concept step by step, using a real company story so you understand **why** we are writing each script — not just **how**. By the end, you'll have 5 working Python scripts that solve real problems, which you can proudly show in any job interview.

> **Company in this project:** TechZen Solutions — a mid-sized IT services company in Hyderabad with 200 employees. You just joined as a Junior Python Developer. Let's begin.

#### What You Will Build in This Project

You will build **5 Python automation scripts** that TechZen Solutions actually uses in day-to-day operations. These are not toy examples — these are the exact types of scripts that real companies need and pay engineers to build.

File Organiser, CSV Report Generator, API Health Monitor, Email Alert System, Log Analyser, os module, requests, smtplib, csv & json

> **📁 Script 1 — File Organiser**

> Automatically sort thousands of files in a folder into subfolders by type — PDFs, images, Excel, logs. Runs every morning at 9 AM.

> Read employee attendance from CSV, calculate present/absent/late counts, and produce a daily summary report automatically.

> Check if the company's web services are online every 5 minutes. If any service is down, log the failure and trigger an alert.

> Send automatic email notifications to the IT manager when a server goes down or disk space exceeds 90%. No manual steps needed.

> Parse application log files, count errors by type, identify the top 3 error patterns, and write a clean summary report.

**Scene 1 — TechZen Solutions Office, Hyderabad | Your First Day**

> **Meghana** _Engineering Manager — 8 years in IT automation_
> 
> Welcome to TechZen! I'm glad you joined us. You said you know Python basics — variables, loops, functions. Great. Now let me tell you what we actually need from a Python developer here. Our team wastes 3 hours every morning doing things manually that a Python script could do in 10 seconds. We need someone to fix that.

> **Ravi (You)** _Junior Python Developer — Day 1 at TechZen_
> 
> I have learned Python basics in college — for loops, if-else, functions. But I'm not sure how to build something useful for a real company. What exactly do you mean by automation?

> **Karthik** _Senior Python Developer — 4 years at TechZen_
> 
> I'll give you a real example. Every morning, Preethi from Admin opens the Downloads folder — it has 800 files mixed together. PDFs, Excel sheets, images, log files — all dumped in one place. She spends 45 minutes moving them into separate folders. That is your first assignment: write a Python script that does that in under 3 seconds. That's automation.

> **Meghana** _Engineering Manager_
> 
> And that is just one task. We have API services, attendance records, server logs, email notifications — all done manually right now. By the time you finish this project, you will have automated all five of these pain points. Ready?

### 1. Setup — Python Environment for Real Projects

Before writing a single line of code, a professional Python developer sets up the environment properly. This means using virtual environments (so your project's libraries don't conflict with other projects), and organising code into separate files instead of one big file.

> **Why Virtual Environments? (Simple Explanation for Freshers)**

> Imagine you have two projects. Project A needs version 1.0 of a library. Project B needs version 2.0 of the same library. If you install both globally, they fight each other and break. A virtual environment is like a separate box for each project — it has its own Python and its own libraries. Always create a virtual environment for every project. This is standard practice in all professional companies.

```
TechZen Python Automation — Project Structure
==============================================

  techzen-automation/
  ├── venv/                    ← Virtual environment (never commit to Git)
  ├── scripts/
  │   ├── file_organiser.py    ← Script 1
  │   ├── report_generator.py  ← Script 2
  │   ├── api_monitor.py       ← Script 3
  │   ├── email_alerts.py      ← Script 4
  │   └── log_analyser.py      ← Script 5
  ├── data/
  │   ├── downloads/           ← Sample files for Script 1
  │   ├── attendance.csv       ← Sample data for Script 2
  │   └── app.log              ← Sample log for Script 5
  ├── reports/                 ← Generated reports go here
  ├── config.py                ← Company settings (server URLs, emails)
  ├── requirements.txt         ← List of libraries needed
  └── README.md                ← How to run the project
```

1. Create the project folder and virtual environment
Open your terminal and run these commands. Each command does exactly one thing.

```bash
# Step 1: Create project folder
mkdir techzen-automation
cd techzen-automation

# Step 2: Create virtual environment (creates a folder called venv)
python -m venv venv

# Step 3: Activate the virtual environment
# On Windows:
venv\Scripts\activate

# On Mac/Linux:
source venv/bin/activate

# You will see (venv) appear in your terminal prompt — this means it is active
# (venv) C:\Users\Ravi\techzen-automation>
# Step 4: Install required libraries
pip install requests       # for making HTTP requests (Script 3)
pip install schedule       # for scheduling scripts to run automatically
# Step 5: Save your requirements (so others can install the same libraries)
pip freeze > requirements.txt
```

2. Create the config.py file (company settings in one place)
Good practice: never hardcode company-specific values inside your scripts. Put them in one config file. If anything changes, you update one file — not ten scripts.

```
# config.py — TechZen company configuration
# Folder paths
DOWNLOADS_FOLDER = "data/downloads"
REPORTS_FOLDER = "reports"
ATTENDANCE_FILE = "data/attendance.csv"
LOG_FILE = "data/app.log"
# Company API services to monitor
COMPANY_APIS = [
    {"name": "TechZen HR Portal",    "url": "https://hr.techzen.com/health"},
    {"name": "TechZen CRM System",   "url": "https://crm.techzen.com/ping"},
    {"name": "TechZen Payroll API", "url": "https://payroll.techzen.com/status"},
]

# Email settings
SMTP_SERVER = "smtp.gmail.com"
SMTP_PORT = 587
SENDER_EMAIL = "automation@techzen.com"
ALERT_RECIPIENT = "it.manager@techzen.com"
# Thresholds
DISK_USAGE_THRESHOLD = 90   # Send alert if disk is more than 90% full
API_TIMEOUT_SECONDS = 10    # Wait max 10 seconds for API response
```

> **💡 Fresher Tip — What is import?**

> Python has thousands of built-in tools (called **modules**) that you don't have to write yourself. When you write `import os`, you're telling Python: "Give me access to all the operating system tools." The `os` module lets you create folders, list files, delete files, check file sizes — all the things you would normally click on in Windows Explorer. Python scripting is mostly about knowing which modules exist and using them smartly.

### 2. Script 1 — Automatic File Organiser

**Business Problem:** Preethi from Admin spends 45 minutes every morning manually sorting files in the Downloads folder. The folder has PDFs, Excel files, images, log files, and Word documents all mixed together. You need to write a script that automatically moves each file to the correct subfolder based on its extension.

**Scene 2 — Admin Desk, TechZen Office | The File Problem**

> **Preethi** _Admin Executive — TechZen Solutions_
> 
> Look at this folder — 847 files. I have to find every PDF and move it to "Documents", every .xlsx to "Excel Reports", every .jpg and .png to "Images", every .log file to "Logs". I do this every single day. My wrist hurts from clicking and dragging. Is there seriously no way to automate this?

> **Karthik** _Senior Python Developer_
> 
> Ravi, this is your first task. The os module has everything you need. os.listdir() gives you all files in a folder. os.path.splitext() gives you the file extension. shutil.move() moves the file. That's literally the whole script — 30 lines of Python. Go.

```
Script 1 — File Organiser Logic
=================================

  downloads/                        After Script Runs:
  ├── report_jan.pdf         ──►    downloads/
  ├── photo_team.jpg                ├── PDFs/
  ├── attendance.xlsx                │   └── report_jan.pdf
  ├── server.log             ──►    ├── Images/
  ├── resume_john.docx               │   └── photo_team.jpg
  ├── banner.png             ──►    ├── Excel/
  ├── error_trace.log                │   └── attendance.xlsx
  └── budget.xlsx            ──►    ├── Logs/
                                     │   ├── server.log
                             ──►    │   └── error_trace.log
                                     ├── Word/
                             ──►    │   └── resume_john.docx
                                     └── Others/
                                         └── banner.png
```

```python
# scripts/file_organiser.py
# Script 1: Automatic File Organiser for TechZen Downloads Folder
# This script reads all files in the downloads folder and moves
# each file to the correct subfolder based on its file type.
import os        # for listing files and creating folders
import shutil    # for moving files from one folder to another
import datetime # for adding timestamps to our log messages
from config import DOWNLOADS_FOLDER, REPORTS_FOLDER

# ─── STEP 1: Define which file types go into which folder ───────────────────
# This dictionary maps file extensions to folder names.
# Key = file extension (always lowercase), Value = folder name to create
FILE_TYPE_MAP = {
    ".pdf":  "PDFs",
    ".xlsx": "Excel",
    ".xls":  "Excel",
    ".csv":  "Excel",
    ".jpg":  "Images",
    ".jpeg": "Images",
    ".png":  "Images",
    ".gif":  "Images",
    ".log":  "Logs",
    ".txt":  "TextFiles",
    ".docx": "Word",
    ".doc":  "Word",
    ".pptx": "Presentations",
    ".zip":  "Archives",
    ".py":   "PythonScripts",
}

def create_folder_if_not_exists(folder_path):
    """Create a folder only if it doesn't already exist."""
if not os.path.exists(folder_path):
        os.makedirs(folder_path)  # makedirs creates all parent folders too
        print(f"  Created folder: {folder_path}")

def organise_files(source_folder):
    """
    Main function: reads all files from source_folder,
    determines the correct destination subfolder,
    and moves each file there.
    """
    print(f"\n[{datetime.datetime.now()}] Starting File Organiser...")
    print(f"Source folder: {source_folder}")

    # Check if the source folder exists before doing anything
if not os.path.exists(source_folder):
        print(f"ERROR: Folder '{source_folder}' does not exist. Stopping.")
        return
# os.listdir() gives us a list of all file/folder names in the directory
    all_items = os.listdir(source_folder)

    moved_count = 0       # how many files we moved
    skipped_count = 0     # how many files we skipped (folders, unknown types)
for filename in all_items:

        # Build the full path: "data/downloads/report_jan.pdf"
        full_path = os.path.join(source_folder, filename)

        # Skip if it's a folder (we only want to move files)
if os.path.isdir(full_path):
            skipped_count += 1
            continue
# os.path.splitext splits "report_jan.pdf" into ("report_jan", ".pdf")
        _, extension = os.path.splitext(filename)
        extension = extension.lower()  # convert to lowercase for consistency
# Look up which folder this extension belongs to
# .get() returns "Others" if the extension is not in our map
        destination_folder_name = FILE_TYPE_MAP.get(extension, "Others")

        # Build the destination folder path
        destination_folder = os.path.join(source_folder, destination_folder_name)

        # Create the destination folder if it doesn't exist yet
        create_folder_if_not_exists(destination_folder)

        # Move the file from source to destination
        destination_path = os.path.join(destination_folder, filename)
        shutil.move(full_path, destination_path)
        print(f"  Moved: {filename}  →  {destination_folder_name}/")
        moved_count += 1

    print(f"\nDone! Moved {moved_count} files. Skipped {skipped_count} items.")
    print(f"[{datetime.datetime.now()}] File Organiser complete.\n")

# ─── MAIN EXECUTION ─────────────────────────────────────────────────────────
if __name__ == "__main__":
    organise_files(DOWNLOADS_FOLDER)
```

```
[2026-01-15 09:00:12] Starting File Organiser...
Source folder: data/downloads
  Created folder: data/downloads/PDFs
  Moved: report_jan.pdf  →  PDFs/
  Created folder: data/downloads/Images
  Moved: photo_team.jpg  →  Images/
  Created folder: data/downloads/Excel
  Moved: attendance.xlsx  →  Excel/
  Created folder: data/downloads/Logs
  Moved: server.log  →  Logs/
  Moved: error_trace.log  →  Logs/
  Created folder: data/downloads/Word
  Moved: resume_john.docx  →  Word/
  Created folder: data/downloads/Others
  Moved: banner.png  →  Images/

Done! Moved 7 files. Skipped 3 items.
[2026-01-15 09:00:12] File Organiser complete.
```

> **💡 Fresher Tip — What is if __name__ == "__main__"?**

> This is one of the most important patterns in Python. When you run a file directly (python file_organiser.py), Python sets `__name__` to `"__main__"`. When another file imports this file, `__name__` is set to the file's name instead. So this check ensures the `organise_files()` function only runs when you execute this file directly — not when someone imports it from another script. Always use this pattern in professional Python scripts.

### 3. Script 2 — Attendance Report Generator

**Business Problem:** TechZen's HR team receives a CSV file every day from the badge scanning system. The CSV has one row per employee per day with their status (Present, Absent, Late). HR needs a daily summary report — how many were present, absent, late, and which employees were absent. This is done manually with Excel filters, taking 30 minutes each day.

**Scene 3 — HR Department, TechZen Office | The Daily Report Problem**

> **Divya** _HR Executive — TechZen Solutions_
> 
> Ravi, every morning I open this attendance CSV, apply filters in Excel, count the Present rows, count the Absent rows, write the numbers into an email, and send it to the manager. I do this for 200 employees. It takes me half an hour and I make mistakes sometimes. Can Python do this automatically?

> **Karthik** _Senior Python Developer_
> 
> Absolutely. Python's csv module reads CSV files line by line. You loop through the rows, count the statuses, and write the summary. Add the datetime module for the date, write the output to a new CSV or text file, and Divya never has to open Excel again for this task.

```
Script 2 — Attendance Report Flow
===================================

  attendance.csv (Input)           daily_report_2026-01-15.txt (Output)
  ──────────────────────           ────────────────────────────────────
  Date,EmployeeID,Name,Status      TechZen Solutions — Daily Attendance
  2026-01-15,E001,Aarav,Present    Report Date: 2026-01-15
  2026-01-15,E002,Brinda,Late      ──────────────────────────────────────
  2026-01-15,E003,Chetan,Absent    Total Employees: 200
  2026-01-15,E004,Deepa,Present    Present:         172 (86.0%)
  ...                      ──►     Late:             18 ( 9.0%)
                                   Absent:           10 ( 5.0%)

                                   ABSENT EMPLOYEES:
                                   - E003 | Chetan Kumar
                                   - E017 | Farida Noor
                                   ...8 more...
```

```python
# scripts/report_generator.py
# Script 2: Attendance Report Generator for TechZen HR Department
# Reads daily attendance CSV, calculates summary, and writes report file.
import csv          # built-in module for reading and writing CSV files
import os           # for creating the reports folder
import datetime     # for today's date in the report filename
from config import ATTENDANCE_FILE, REPORTS_FOLDER

def read_attendance_csv(file_path):
    """
    Read attendance CSV file and return a list of dictionaries.
    Each dictionary represents one row: {'Date':..., 'EmployeeID':..., 'Name':..., 'Status':...}
    """
    attendance_records = []

    try:
        # open() opens the file. 'r' means read-only. encoding='utf-8' handles special characters.
with open(file_path, 'r', encoding='utf-8') as csv_file:
            # DictReader reads each row as a dictionary using the header row as keys
            reader = csv.DictReader(csv_file)
            for row in reader:
                attendance_records.append(row)

        print(f"Read {len(attendance_records)} records from {file_path}")
        return attendance_records

    except FileNotFoundError:
        print(f"ERROR: File not found: {file_path}")
        return []

def generate_report(records):
    """
    Takes a list of attendance records and generates a report dictionary
    with counts and lists of absent/late employees.
    """
# Initialise counters
    report = {
        "total": 0,
        "present": 0,
        "absent": 0,
        "late": 0,
        "absent_employees": [],   # list of dicts: {id, name}
"late_employees": [],     # list of dicts: {id, name}
    }

    for record in records:
        report["total"] += 1
        status = record["Status"].strip()  # .strip() removes leading/trailing spaces
if status == "Present":
            report["present"] += 1

        elif status == "Absent":
            report["absent"] += 1
            report["absent_employees"].append({
                "id": record["EmployeeID"],
                "name": record["Name"]
            })

        elif status == "Late":
            report["late"] += 1
            report["late_employees"].append({
                "id": record["EmployeeID"],
                "name": record["Name"]
            })

    return report

def write_report_file(report, output_folder):
    """Write the report dictionary to a text file in the reports folder."""
# Create reports folder if it doesn't exist
    os.makedirs(output_folder, exist_ok=True)

    today = datetime.date.today()  # gives us 2026-01-15
    report_filename = f"attendance_report_{today}.txt"
    report_path = os.path.join(output_folder, report_filename)

    total = report["total"]

    # Safe percentage calculation — avoid dividing by zero
    pct = lambda n: round((n / total) * 100, 1) if total > 0 else 0

    # 'w' means write-mode — creates file or overwrites if it already exists
with open(report_path, 'w', encoding='utf-8') as f:
        f.write("=" * 50 + "\n")
        f.write("     TechZen Solutions — Daily Attendance\n")
        f.write(f"     Report Date: {today}\n")
        f.write("=" * 50 + "\n\n")
        f.write(f"Total Employees : {total}\n")
        f.write(f"Present         : {report['present']} ({pct(report['present'])}%)\n")
        f.write(f"Late            : {report['late']} ({pct(report['late'])}%)\n")
        f.write(f"Absent          : {report['absent']} ({pct(report['absent'])}%)\n")

        if report["absent_employees"]:
            f.write("\nABSENT EMPLOYEES:\n")
            for emp in report["absent_employees"]:
                f.write(f"  - {emp['id']} | {emp['name']}\n")

        if report["late_employees"]:
            f.write("\nLATE EMPLOYEES:\n")
            for emp in report["late_employees"]:
                f.write(f"  - {emp['id']} | {emp['name']}\n")

        f.write(f"\nGenerated at: {datetime.datetime.now()}\n")

    print(f"Report written to: {report_path}")
    return report_path

if __name__ == "__main__":
    records = read_attendance_csv(ATTENDANCE_FILE)
    if records:
        report = generate_report(records)
        write_report_file(report, REPORTS_FOLDER)
```

### 4. Script 3 — API Health Monitor

**Business Problem:** TechZen runs 3 critical web services: the HR Portal, CRM System, and Payroll API. The IT team only finds out these services are down when employees complain — sometimes 30 minutes after the outage starts. The IT manager wants a Python script that checks all three services every 5 minutes and logs the results.

**Scene 4 — IT Operations Room, TechZen | The Outage Discovery**

> **Suresh** _IT Operations Manager — TechZen Solutions_
> 
> This morning, the Payroll API was down from 8 AM to 8:45 AM. I found out at 8:44 AM — when the finance team came to my desk shouting that they couldn't process salaries. 44 minutes of downtime and nobody told me! I need something that checks these services automatically, every few minutes, and tells me the moment something goes wrong.

> **Karthik** _Senior Python Developer_
> 
> Ravi, this is the requests library's job. requests.get(url) sends an HTTP request to a URL. If the service responds with status code 200, it's up. If it throws an exception or returns 5xx, it's down. Wrap it in a loop with time.sleep(300) for 5-minute intervals. Log the results. Done.

```
Script 3 — API Monitor Logic
==============================

  Every 5 minutes:
  ─────────────────

  For each company API URL:
    │
    ├── Send GET request (with 10s timeout)
    │
    ├── If response status == 200 ──►  Log: [UP] Service name - response time
    │
    ├── If response status != 200 ──►  Log: [DOWN] Service name - status code
    │                                  Trigger: email alert!
    │
    └── If connection error ──────►   Log: [ERROR] Service name - error message
                                      Trigger: email alert!

  monitor.log:
  ─────────────
  [2026-01-15 09:00:00] [UP]    TechZen HR Portal    200  45ms
  [2026-01-15 09:00:00] [UP]    TechZen CRM System   200  82ms
  [2026-01-15 09:00:00] [DOWN]  TechZen Payroll API  503  —
  [2026-01-15 09:05:00] [DOWN]  TechZen Payroll API  503  —
```

```python
# scripts/api_monitor.py
# Script 3: API Health Monitor for TechZen Services
# Checks each company API every 5 minutes and logs results.
import requests    # third-party library for HTTP requests (pip install requests)
import time        # for sleep() — pauses the script for N seconds
import datetime   # for timestamps in log messages
import os
from config import COMPANY_APIS, API_TIMEOUT_SECONDS, REPORTS_FOLDER

# ─── CONSTANTS ───────────────────────────────────────────────────────────────
CHECK_INTERVAL_SECONDS = 300   # 300 seconds = 5 minutes
LOG_FILE = os.path.join(REPORTS_FOLDER, "api_monitor.log")

def log_message(message):
    """Write a message to both the console and the log file."""
    timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    full_message = f"[{timestamp}] {message}"
    print(full_message)

    os.makedirs(REPORTS_FOLDER, exist_ok=True)
    # 'a' means append-mode — adds to end of file without deleting existing content
with open(LOG_FILE, 'a', encoding='utf-8') as f:
        f.write(full_message + "\n")

def check_api(api_info):
    """
    Send an HTTP GET request to one API URL.
    Returns a dictionary with the check result.
    """
    name = api_info["name"]
    url  = api_info["url"]

    try:
        # time.time() gives current time in seconds since 1970 (a float)
        start_time = time.time()

        # Send GET request. timeout prevents script from hanging forever.
        response = requests.get(url, timeout=API_TIMEOUT_SECONDS)

        # Calculate how long the request took (in milliseconds)
        response_time_ms = round((time.time() - start_time) * 1000)

        # HTTP 200 means "OK" — service is healthy
if response.status_code == 200:
            log_message(f"[UP]    {name:<30} {response.status_code}  {response_time_ms}ms")
            return {"status": "up", "name": name, "code": response.status_code}
        else:
            # Status 500, 503 etc means server error
            log_message(f"[DOWN]  {name:<30} {response.status_code}  --")
            return {"status": "down", "name": name, "code": response.status_code}

    except requests.exceptions.ConnectionError:
        # This happens when the server is completely unreachable
        log_message(f"[ERROR] {name:<30} Connection refused")
        return {"status": "error", "name": name, "code": None}

    except requests.exceptions.Timeout:
        # This happens when the server takes more than 10 seconds to respond
        log_message(f"[ERROR] {name:<30} Timeout after {API_TIMEOUT_SECONDS}s")
        return {"status": "error", "name": name, "code": None}

def run_monitor():
    """
    Main monitoring loop. Runs forever until the user presses Ctrl+C.
    Checks all APIs, then waits CHECK_INTERVAL_SECONDS before checking again.
    """
    log_message("=" * 55)
    log_message("TechZen API Monitor started. Press Ctrl+C to stop.")
    log_message("=" * 55)

    while True:   # infinite loop — keeps running until we stop it
        log_message("--- Checking all APIs ---")
        failed_services = []

        for api in COMPANY_APIS:
            result = check_api(api)
            if result["status"] != "up":
                failed_services.append(result["name"])

        # If any service failed, this is where you would call the email alert
if failed_services:
            log_message(f"ALERT: {len(failed_services)} service(s) DOWN: {', '.join(failed_services)}")
            # send_alert_email(failed_services)  ← we build this in Script 4

        log_message(f"Next check in {CHECK_INTERVAL_SECONDS // 60} minutes...\n")
        time.sleep(CHECK_INTERVAL_SECONDS)  # pause for 5 minutes
if __name__ == "__main__":
    try:
        run_monitor()
    except KeyboardInterrupt:
        # KeyboardInterrupt is raised when user presses Ctrl+C
        print("\nMonitor stopped by user.")
```

### 5. Script 4 — Email Alert System

**Business Problem:** When the API monitor detects a service is down, or when disk usage exceeds 90%, the IT manager needs an immediate email alert. Currently someone has to manually send these emails. The alert email should include the service name, time of failure, and what action is needed.

**Scene 5 — IT Operations Room | "Why Didn't Anyone Tell Me?"**

> **Suresh** _IT Operations Manager_
> 
> Your monitor script is great — it logs everything. But I can't stare at a log file all day. I need an email the moment something breaks. When any service goes down, or when the server disk hits 90%, send an email to me and my deputy immediately. Subject should say URGENT. Body should say exactly which service, what time, and what I should check first.

> **Karthik** _Senior Python Developer_
> 
> Python's smtplib handles email. SMTP is the protocol email uses to travel from server to server. You connect to Gmail's SMTP server, login with the sender credentials, and use EmailMessage to build the email object. The whole function is about 25 lines. Ravi, you can call this function from the monitor script when a failure is detected.

```python
# scripts/email_alerts.py
# Script 4: Email Alert System for TechZen IT Operations
# Sends formatted alert emails when services fail or disk usage is high.
import smtplib                     # built-in Python module for SMTP email sending
from email.message import EmailMessage  # for building structured email objects
import datetime
import os
import shutil                      # has disk_usage() function
from config import SMTP_SERVER, SMTP_PORT, SENDER_EMAIL, ALERT_RECIPIENT, DISK_USAGE_THRESHOLD

def send_alert_email(subject, body, sender_password):
    """
    Send an alert email via Gmail SMTP.
    
    Parameters:
        subject (str): Email subject line
        body (str): Email body text
        sender_password (str): App password for the sender Gmail account
    """
# EmailMessage is a Python object that represents an email
    msg = EmailMessage()
    msg["From"] = SENDER_EMAIL
    msg["To"] = ALERT_RECIPIENT
    msg["Subject"] = subject
    msg.set_content(body)

    try:
        # smtplib.SMTP connects to the mail server
# 'with' ensures the connection is properly closed even if an error occurs
with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as smtp:
            # starttls() upgrades the connection to encrypted (TLS)
            smtp.starttls()
            # Login with your Gmail sender account
            smtp.login(SENDER_EMAIL, sender_password)
            # Send the email object we built above
            smtp.send_message(msg)
            print(f"Alert email sent to {ALERT_RECIPIENT}")

    except smtplib.SMTPAuthenticationError:
        print("ERROR: Email authentication failed. Check your Gmail App Password.")
    except smtplib.SMTPException as e:
        print(f"ERROR: Could not send email: {e}")

def send_service_down_alert(failed_services, sender_password):
    """
    Build and send an alert email specifically for service outages.
    Called by api_monitor.py when a service is detected as DOWN.
    """
    now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    service_list = "\n".join([f"  - {s}" for s in failed_services])

    subject = f"[URGENT] TechZen Service Down Alert — {now}"

    body = f"""
TechZen IT Alert System
========================
Alert Type : SERVICE DOWN
Detected   : {now}
Affected   :
{service_list}

Immediate Actions:
  1. Log into the server dashboard
  2. Check application logs at /var/log/techzen/
  3. Restart the failed service if necessary
  4. Confirm recovery by checking service status
  5. Reply to this email with root cause once resolved

This is an automated alert from TechZen Python Monitor.
Do NOT reply to this email — it is unmonitored.
"""

    send_alert_email(subject, body, sender_password)

def check_disk_and_alert(sender_password, path="/"):
    """
    Check disk usage on the server.
    If usage exceeds the threshold (90%), send an alert email.
    """
# shutil.disk_usage() returns named tuple with total, used, free (in bytes)
    usage = shutil.disk_usage(path)

    # Calculate percentage used
    percent_used = (usage.used / usage.total) * 100
    percent_used = round(percent_used, 1)

    print(f"Disk usage on {path}: {percent_used}%")

    if percent_used >= DISK_USAGE_THRESHOLD:
        now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        # Convert bytes to GB for readable numbers
        total_gb = round(usage.total / (1024**3), 1)
        used_gb  = round(usage.used  / (1024**3), 1)
        free_gb  = round(usage.free  / (1024**3), 1)

        subject = f"[URGENT] Disk Space Alert — {percent_used}% Used on TechZen Server"
        body = f"""
TechZen IT Alert System
========================
Alert Type : HIGH DISK USAGE
Detected   : {now}
Path       : {path}

Disk Statistics:
  Total : {total_gb} GB
  Used  : {used_gb} GB ({percent_used}%)
  Free  : {free_gb} GB

Threshold  : {DISK_USAGE_THRESHOLD}%

Recommended Actions:
  1. Delete old log files in /var/log/
  2. Archive old report files to storage
  3. Check for large temp files: du -sh /tmp/*
  4. Consider expanding disk volume if consistently above 85%
"""
        send_alert_email(subject, body, sender_password)
    else:
        print(f"Disk OK — below {DISK_USAGE_THRESHOLD}% threshold.")

if __name__ == "__main__":
    # For testing — get password from environment variable (NEVER hardcode passwords!)
    password = os.environ.get("EMAIL_PASSWORD", "")
    if not password:
        print("Set EMAIL_PASSWORD environment variable first.")
    else:
        check_disk_and_alert(password)
```

> **🔐 Security Rule — Never Hardcode Passwords in Code!**

> You might be tempted to write `password = "MyGmailPass123"` directly in your script. **Never do this.** If you push this to GitHub, anyone who finds the repository can steal your password. The correct approach is to store the password as an **environment variable** on the server (`export EMAIL_PASSWORD="yourpass"`) and read it in your script with `os.environ.get("EMAIL_PASSWORD")`. This is standard practice in all professional companies.

### 6. Script 5 — Application Log Analyser

**Business Problem:** TechZen's backend application writes thousands of log lines every day. When something goes wrong, the development team needs to know: how many errors occurred, what types of errors are happening most, and at what times. Manually reading 50,000 lines of logs is impossible. A Python script can analyse the entire log file in seconds.

**Scene 6 — Development Team Meeting | "The App is Throwing Errors"**

> **Ananya** _Backend Developer — TechZen Solutions_
> 
> Our app.log file has 60,000 lines from yesterday. I know there are errors in there — users complained about payment failures and login timeouts. But I can't read 60,000 lines by hand. I need to know: how many ERRORs, which error message appears most often, and at what hours did most errors happen.

> **Karthik** _Senior Python Developer_
> 
> Ravi, this is classic log analysis. Open the file, read line by line, check if "ERROR" appears in the line, extract the error message, use a dictionary to count how many times each message appeared. Then sort by count — highest first. Ten minutes of Python saves three hours of manual reading. Write it.

```
Sample app.log format:
=========================
2026-01-15 08:01:22 INFO  Request received: GET /api/employees
2026-01-15 08:01:23 INFO  Response 200 OK in 45ms
2026-01-15 08:03:11 ERROR DatabaseConnectionError: timeout after 30s
2026-01-15 08:03:12 ERROR DatabaseConnectionError: timeout after 30s
2026-01-15 08:15:44 WARN  Slow query detected: 8200ms
2026-01-15 08:17:02 ERROR PaymentGatewayError: invalid card token
2026-01-15 08:19:55 ERROR DatabaseConnectionError: timeout after 30s

Script 5 Output (analysis_report_2026-01-15.txt):
===================================================
TechZen — Log Analysis Report: 2026-01-15
Total Lines    : 60,000
ERROR lines    : 847
WARN lines     : 234
INFO lines     : 58,919

TOP 3 ERROR PATTERNS:
  1. DatabaseConnectionError: timeout after 30s   →  312 occurrences
  2. PaymentGatewayError: invalid card token       →  198 occurrences
  3. NullPointerException in UserService.java      →  89 occurrences

ERROR HOURLY DISTRIBUTION:
  08:00 – 09:00  : 245 errors  ████████████
  09:00 – 10:00  : 31 errors   ██
  13:00 – 14:00  : 412 errors  ████████████████████
```

```python
# scripts/log_analyser.py
# Script 5: Application Log Analyser for TechZen Development Team
# Reads app.log, counts error types, identifies patterns, writes summary.
import os
import re           # built-in module for regular expressions (pattern matching)
import datetime
from collections import Counter, defaultdict  # powerful counting tools
from config import LOG_FILE, REPORTS_FOLDER

def analyse_log(log_file_path):
    """
    Read and analyse the application log file.
    Returns a dictionary with all analysis results.
    """
# Counters for each log level
    level_counts = Counter()          # Counter is like a dict but counts automatically
    error_messages = Counter()        # counts each unique error message
    hourly_errors = defaultdict(int) # errors per hour of day {8: 245, 9: 31, ...}
    total_lines = 0

    # Log line format: "2026-01-15 08:03:11 ERROR DatabaseConnectionError: timeout"
    # We use a regular expression to capture the 4 parts of each line.
# Regex pattern explanation:
    # (\d{4}-\d{2}-\d{2})  → captures date like "2026-01-15"
    # (\d{2}:\d{2}:\d{2})  → captures time like "08:03:11"
    # (\w+)                 → captures level like "ERROR"
    # (.+)                  → captures rest of line (the message)
    log_pattern = re.compile(
        r"(\d{4}-\d{2}-\d{2})\s+(\d{2}:\d{2}:\d{2})\s+(\w+)\s+(.+)"
    )

    try:
        with open(log_file_path, 'r', encoding='utf-8', errors='ignore') as f:
            for line in f:         # reads one line at a time (memory efficient!)
                total_lines += 1
                line = line.strip()  # remove newline at end
if not line:         # skip empty lines
continue
# Try to match the pattern against this log line
                match = log_pattern.match(line)
                if not match:
                    continue
# Extract the 4 captured groups
                date_str, time_str, level, message = match.groups()

                # Count log levels: INFO, WARN, ERROR, DEBUG
                level_counts[level] += 1

                # For ERROR lines, do additional analysis
if level == "ERROR":
                    # Count each unique error message
                    error_messages[message] += 1

                    # Extract the hour from the time string "08:03:11" → 8
                    hour = int(time_str.split(":")[0])
                    hourly_errors[hour] += 1

    except FileNotFoundError:
        print(f"ERROR: Log file not found: {log_file_path}")
        return None
return {
        "total_lines": total_lines,
        "level_counts": level_counts,
        "top_errors": error_messages.most_common(10),  # top 10 errors
"hourly_errors": dict(sorted(hourly_errors.items())),
    }

def write_analysis_report(analysis, output_folder):
    """Write the analysis results to a human-readable report file."""
if not analysis:
        return

    os.makedirs(output_folder, exist_ok=True)
    today = datetime.date.today()
    report_path = os.path.join(output_folder, f"log_analysis_{today}.txt")

    with open(report_path, 'w', encoding='utf-8') as f:
        f.write("=" * 60 + "\n")
        f.write(f"     TechZen — Log Analysis Report: {today}\n")
        f.write("=" * 60 + "\n\n")
        f.write(f"Total Lines     : {analysis['total_lines']:,}\n")

        for level in ["ERROR", "WARN", "INFO", "DEBUG"]:
            count = analysis["level_counts"].get(level, 0)
            f.write(f"{level:<15} : {count:,}\n")

        f.write("\nTOP ERROR PATTERNS:\n")
        f.write("-" * 60 + "\n")
        for i, (msg, count) in enumerate(analysis["top_errors"], start=1):
            f.write(f"  {i}. {msg[:55]:<55}  → {count} occurrences\n")

        f.write("\nERROR HOURLY DISTRIBUTION:\n")
        f.write("-" * 60 + "\n")
        for hour, count in analysis["hourly_errors"].items():
            bar = "█" * min(count // 10, 30)  # visual bar (max 30 chars)
            f.write(f"  {hour:02d}:00  {count:>5} errors  {bar}\n")

        f.write(f"\nReport generated at: {datetime.datetime.now()}\n")

    print(f"Log analysis report written to: {report_path}")
    return report_path

if __name__ == "__main__":
    analysis = analyse_log(LOG_FILE)
    if analysis:
        write_analysis_report(analysis, REPORTS_FOLDER)
```

### 7. Putting It All Together — Schedule Scripts to Run Automatically

Writing scripts is only half the job. The real value comes when scripts run automatically without anyone having to remember to start them. In a company, this is done using a **scheduler**. Python's `schedule` library makes this simple.

**Scene 7 — Meghana's Office | The Final Review**

> **Meghana** _Engineering Manager_
> 
> Ravi, these five scripts are excellent. But I'm not going to log in every morning and run them manually — that defeats the purpose. I need the file organiser to run at 8 AM, the attendance report at 9 AM, the API monitor checking every 5 minutes all day, and the log analyser running at 6 PM after the business day ends. Can you wire this into one master scheduler?

> **Karthik** _Senior Python Developer_
> 
> Absolutely. The schedule library makes this clean. You define what runs when, then call schedule.run_pending() in a loop with time.sleep(60) to check every minute. One master script, all five automations running on their own schedule. On Linux servers, you'd use cron for production — but schedule works perfectly for development and Windows environments.

```python
# main_scheduler.py — Master automation scheduler for TechZen
# Run this once and all 5 scripts execute on their own schedule.
# pip install schedule   (if not already installed)
import schedule
import time
import datetime

# Import our automation scripts as modules
from scripts.file_organiser  import organise_files
from scripts.report_generator import read_attendance_csv, generate_report, write_report_file
from scripts.api_monitor      import check_api
from scripts.log_analyser     import analyse_log, write_analysis_report
from config import DOWNLOADS_FOLDER, ATTENDANCE_FILE, COMPANY_APIS, LOG_FILE, REPORTS_FOLDER

def job_file_organiser():
    print(f"[{datetime.datetime.now()}] Running: File Organiser")
    organise_files(DOWNLOADS_FOLDER)

def job_attendance_report():
    print(f"[{datetime.datetime.now()}] Running: Attendance Report")
    records = read_attendance_csv(ATTENDANCE_FILE)
    if records:
        report = generate_report(records)
        write_report_file(report, REPORTS_FOLDER)

def job_api_monitor():
    print(f"[{datetime.datetime.now()}] Running: API Health Check")
    for api in COMPANY_APIS:
        check_api(api)

def job_log_analyser():
    print(f"[{datetime.datetime.now()}] Running: Log Analysis")
    analysis = analyse_log(LOG_FILE)
    if analysis:
        write_analysis_report(analysis, REPORTS_FOLDER)

# ─── DEFINE SCHEDULES ────────────────────────────────────────────────────────
schedule.every().day.at("08:00").do(job_file_organiser)    # 8 AM daily
schedule.every().day.at("09:00").do(job_attendance_report)  # 9 AM daily
schedule.every(5).minutes.do(job_api_monitor)               # every 5 minutes
schedule.every().day.at("18:00").do(job_log_analyser)       # 6 PM daily

print("TechZen Automation Scheduler started.")
print("Schedules:")
print("  08:00 — File Organiser")
print("  09:00 — Attendance Report Generator")
print("  Every 5 min — API Health Monitor")
print("  18:00 — Log Analyser")
print("Press Ctrl+C to stop.\n")

while True:
    schedule.run_pending()  # check if any scheduled job needs to run right now
    time.sleep(60)          # check every 60 seconds
```

### 8. Debugging — When Your Script Breaks (And It Will)

Every script breaks at some point. A professional developer doesn't panic — they use a systematic approach to find and fix the problem. Here are the most common errors freshers face and exactly how to fix them.

**Common Python Errors and How to Fix Them**

- **🔴 FileNotFoundError** — **What it means:** The file path you gave doesn't exist on the computer.

- **🔴 KeyError** — **What it means:** You tried to access a dictionary key that doesn't exist.

- **🔴 ConnectionError** — **What it means:** Python couldn't reach the URL — network issue or wrong URL.

- **🔴 IndentationError** — **What it means:** Your code has mixed tabs and spaces, or wrong indentation.

- **🔴 PermissionError** — **What it means:** Python tried to move/delete a file it doesn't have permission for.

- **🔴 UnicodeDecodeError** — **What it means:** The file has special characters that Python can't decode.

```python
# The Debugging Toolkit — use these to find problems quickly
# 1. Print the value of any variable to see what's in it
print(f"filename = '{filename}'")
print(f"extension = '{extension}'")
print(f"records type: {type(records)}, count: {len(records)}")

# 2. Check if a file/folder actually exists before using it
if os.path.exists(DOWNLOADS_FOLDER):
    print("Folder found!")
else:
    print(f"ERROR: Folder not found: {DOWNLOADS_FOLDER}")

# 3. Print the current working directory (where Python is looking for files)
print(f"Current directory: {os.getcwd()}")

# 4. Print all keys in a CSV row to check column names
with open(ATTENDANCE_FILE) as f:
    reader = csv.DictReader(f)
    first_row = next(reader)
    print(f"CSV columns: {list(first_row.keys())}")  # see exact column names
# 5. Use try-except to catch exactly what went wrong
try:
    result = check_api(api)
except Exception as e:
    print(f"Error type : {type(e).__name__}")
    print(f"Error message: {e}")
```

### 9. Interview Questions — Python Scripting

These are the questions you will face in technical interviews when you apply for Python Developer, Automation Engineer, or DevOps roles. The answers are based on what you built in this project.

##### Interview Q&A — Fresher Level (0–1 Year Experience)

**Q: Q1. What is the difference between the os module and the shutil module?**

A: The `os` module provides low-level operating system interactions — listing files (`os.listdir()`), creating directories (`os.makedirs()`), checking if a file exists (`os.path.exists()`), splitting file extensions (`os.path.splitext()`). The `shutil` module provides higher-level file operations — copying files (`shutil.copy()`), moving files (`shutil.move()`), deleting entire directory trees (`shutil.rmtree()`), and checking disk usage (`shutil.disk_usage()`). In our file organiser, we used `os` to list and inspect files, and `shutil.move()` to move them to the correct subfolder.

**Q: Q2. What is a virtual environment and why should you always use one?**

A: A virtual environment is an isolated Python installation that has its own set of installed libraries, separate from your system Python and other projects. You should always use one because different projects often need different versions of the same library — without virtual environments, they conflict and break each other. The command `python -m venv venv` creates it and `venv\Scripts\activate` (Windows) or `source venv/bin/activate` (Linux) activates it. Professional teams never install libraries globally — every project has its own venv.

**Q: Q3. What does the requests library do and what is an HTTP status code?**

A: The `requests` library lets Python programs send HTTP requests to web servers, just like a browser does when you visit a website. `requests.get(url)` sends a GET request and returns a response object. An HTTP status code is a 3-digit number that tells you what happened: 200 means "OK - success", 404 means "Not Found", 500 means "Internal Server Error", 503 means "Service Unavailable". In our API monitor, we check `response.status_code == 200` to determine if a service is healthy.

**Q: Q4. What is a Python dictionary and how did you use it in this project?**

A: A dictionary is a data structure that stores key-value pairs — like a real dictionary where the word is the key and the definition is the value. In our project we used dictionaries in multiple ways: the `FILE_TYPE_MAP` in Script 1 maps file extensions to folder names (`{".pdf": "PDFs", ".xlsx": "Excel"}`); the `report` dictionary in Script 2 stores all report data; and the `api_info` dictionaries in Script 3 store each service's name and URL. Dictionaries are the most commonly used data structure in Python scripting.

**Q: Q5. What is the purpose of try-except in Python?**

A: The `try-except` block handles errors gracefully instead of letting the script crash. Code inside the `try` block runs normally. If an error occurs, Python jumps to the matching `except` block instead of crashing the entire program. In production scripts, this is essential — if one API check fails due to a network error, you don't want the entire monitoring script to stop. You catch the specific exception (`except requests.exceptions.ConnectionError`), log the error, and continue with the next check. Never use bare `except:` — always specify the exception type.

**Q: Q6. What is the difference between 'r', 'w', and 'a' when opening files?**

A: These are file modes. `'r'` (read) opens the file for reading only — if the file doesn't exist, it raises FileNotFoundError. `'w'` (write) creates the file if it doesn't exist, or overwrites it completely if it does. `'a'` (append) opens the file and adds new content at the end without deleting existing content — used in our log_message() function so monitor logs accumulate over time instead of being overwritten every minute. There's also `'rb'` and `'wb'` for binary files (images, PDFs).

**Quiz: Quiz 1 — What does os.path.splitext("report_2026.pdf") return?**

- A) "report_2026"
- B) ".pdf"
- C) ("report_2026", ".pdf")
- D) ["report_2026.pdf"]

> **Answer/explanation:** ✅ Answer: C. os.path.splitext() always returns a tuple of two strings: the filename without extension, and the extension including the dot. This is why we write: `_, extension = os.path.splitext(filename)` — the underscore means "I don't need the first part, only the extension."

**Quiz: Quiz 2 — In our attendance script, why do we use csv.DictReader instead of csv.reader?**

- A) DictReader is faster for large files
- B) DictReader reads each row as a dictionary using the header row as keys, making code more readable
- C) DictReader can handle Excel files directly
- D) csv.reader doesn't work with Python 3

> **Answer/explanation:** ✅ Answer: B. With csv.reader, each row is a list: row[0] is Date, row[1] is EmployeeID, etc. — you have to remember position numbers. With DictReader, each row is a dictionary: row["Date"], row["EmployeeID"] — much more readable and less error-prone. If the column order in the CSV changes, your DictReader code still works; csv.reader code breaks.

**Quiz: Quiz 3 — What happens if you don't use a virtual environment and run pip install requests globally?**

- A) The script won't work at all
- B) It works fine — virtual environments are optional
- C) It installs for all users on the computer and may conflict with other projects needing different library versions
- D) Python will ask for admin permission and refuse

> **Answer/explanation:** ✅ Answer: C. Installing globally works in the short term but causes library version conflicts across projects. If Project A needs requests==2.28 and Project B needs requests==2.30 and you install globally, one of them will break. Virtual environments solve this completely. In professional environments, global pip installs are considered bad practice.

> **Python Scripting Project — Core Takeaways for Freshers**

> - Python scripting is about saving human time — every script you write should automate something a human is doing manually right now. The business value is always time saved × hourly cost × number of times per day/week.
> - Always use virtual environments — one per project, activate before installing any library, and commit requirements.txt to Git so anyone can recreate your exact setup with pip install -r requirements.txt.
> - Separate configuration from logic — company-specific values (paths, URLs, email addresses, thresholds) belong in config.py, not scattered across every script. One change in config.py updates all scripts.
> - Always use try-except around risky operations — file reads, HTTP requests, and email sending can all fail. Catching exceptions prevents one failure from crashing the entire automation.
> - Never hardcode passwords or API keys in your code — use environment variables (os.environ.get). This is the single most important security rule in professional Python development.
> - Always write the output to files — don't just print to console. Console output disappears. Files accumulate history, can be emailed, and serve as audit trails for your automation.
> - Use the Python Standard Library first — os, csv, shutil, datetime, re, collections are all built-in and require no installation. Only install third-party libraries when the standard library genuinely cannot do the job.
> - The if __name__ == "__main__" pattern is professional practice — always include it so your functions can be imported by other scripts without automatically running the entire program.

##### Python Script Writing Standards — TechZen Team Rules

- Every function must have a docstring explaining what it does, its parameters, and what it returns — future you and your teammates will thank present you
- Use f-strings for string formatting (f"Hello {name}") — they are more readable than .format() and % formatting and work in Python 3.6+
- Use specific exception types (except FileNotFoundError) not bare except: — catching all exceptions hides bugs and makes debugging impossible
- Open files using the "with" statement — it guarantees the file is properly closed even if an exception occurs inside the block
- Use os.path.join() for building file paths, never string concatenation — "data/" + "file.csv" breaks on Windows, os.path.join("data", "file.csv") works everywhere
- Print meaningful status messages as your script runs — timestamps, counts, and file paths help you understand exactly what happened when you read the output later
- Test your script on a small sample of real data before running it on 60,000 lines or 800 files — always have a data/test/ folder with 5–10 representative files for testing

##### 🏋️ Hands-On Exercise — Extend the Project (For Practice)

1. **Extend Script 1:** Modify the file organiser to also skip files that were modified more than 30 days ago — use `os.path.getmtime()` which returns the file's last modification timestamp. This prevents accidentally moving archived files.
2. **Extend Script 2:** Add a "streak" column to the attendance report — for each present employee, calculate how many consecutive days they have been present by reading the last 5 days of attendance CSV files from the data folder.
3. **Extend Script 3:** Add response time tracking — store the last 10 response times for each API and send an alert if the average response time crosses 2 seconds, even if the service returns 200 OK. Slow services are almost as bad as down services.
4. **Extend Script 4:** Make the email include an HTML body — instead of plain text, format the alert as an HTML table so the email looks professional. Look up `msg.add_alternative(html_body, subtype='html')` in the Python docs.
5. **Bonus Project:** Write a Script 6 that reads all report files from the reports/ folder, counts how many were generated per week, and emails a weekly summary every Friday at 5 PM — combining everything you've learned: file reading, date filtering, report writing, email sending, and scheduling.

### Python Scripting Project Complete 🎉

You have designed and built five real-world Python automation scripts — File Organiser, Attendance Report Generator, API Health Monitor, Email Alert System, and Log Analyser — the same types of tools used every day in IT companies across India and the world.

> **Meghana**
> 
> "Ravi, when you joined three weeks ago you weren't sure how to build something useful with Python. Today you have automated four of our biggest daily pain points and saved this team over two hours per day. That is what professional Python scripting looks like — not writing algorithms for coding competitions, but solving real problems that real people have in real companies. You are ready."

> **Karthik**
> 
> "Remember: every great automation engineer started exactly where you are now. The key skills you've built — reading files, calling APIs, sending emails, parsing logs, scheduling jobs — appear in almost every Python automation role you'll find on job portals. In your next interview, walk them through these five scripts. That's more impressive than any algorithm you could recite."

> **Ananya**
> 
> "And the log analyser? It found 312 database timeout errors clustered between 8 and 9 AM. We traced that to a batch job that was hitting the database at exactly 8 AM with no connection pooling. We fixed it this morning. Your script found in 3 seconds what would have taken me 4 hours to find manually."

> **Next: Advanced Python Scripting — APIs, Databases & Web Scraping**

> - REST API Integration — POST requests, authentication headers, JSON parsing, paginating API results
> - Database Scripting — connect to MySQL and PostgreSQL with psycopg2, run queries, insert batch data
> - Web Scraping Basics — BeautifulSoup and requests for extracting data from websites
> - JSON & YAML Config Files — reading complex nested config files for multi-environment deployments
> - Multithreading — run multiple API checks simultaneously instead of one at a time for 10x speed improvement
> - Packaging Scripts — converting your scripts into command-line tools with argparse for reuse across teams
