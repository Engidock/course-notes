# Ansible Project Mastery

> **👋 Hey Fresher — Read This First!**

> Ansible is the tool that lets you manage **10 servers or 1,000 servers with exactly the same effort**. Instead of SSHing into each server and running commands one by one, you write a **playbook** — a YAML file that describes what you want done — and Ansible connects to all the servers simultaneously and does it. No agents to install on the servers. No complex setup. Just SSH access and Python. This document uses **short YAML blocks** — each one teaches one thing — with a plain-English explanation right beside it. Every example is from a real business problem at a real company.

> **Company in this project:** StackBridge — an IT consulting firm in Hyderabad managing 60 Linux servers for various clients. They deploy web applications, configure databases, manage users, and patch servers every week — all manually right now. You just joined as a Junior Automation Engineer. Your lead is Gopal. Let's automate StackBridge's entire operations with Ansible.

#### What You Will Learn and Build in This Project

You will write Ansible playbooks that automate StackBridge's real operational tasks — from setting up a web server from scratch to deploying applications and managing user accounts across 60 servers.

Inventory, Playbooks, Modules, Variables, Handlers, Roles, Vault, Templates

> **🗂️ Phase 1 — Foundations**

> Install Ansible, understand how it connects to servers, write your first inventory file, and run your first ad-hoc command.

> Write playbooks to install and configure nginx on web servers. Learn tasks, modules, and the most important modules you'll use daily.

> Use variables to write flexible playbooks that work for different environments. Use Jinja2 templates to generate config files dynamically.

> Restart services only when config changes. Run tasks only on certain OS types. Install a list of packages in one task using loops.

> Organise playbooks into reusable Roles. Encrypt passwords and secrets with Ansible Vault. Build a complete web server provisioning project.

**Scene 1 — StackBridge Server Room, Hyderabad | The Weekend That Broke the Team**

> **Gopal** _Senior Automation Engineer — StackBridge_
> 
> Last weekend we got a new client — 12 fresh Ubuntu servers, completely bare. The team spent Saturday and Sunday SSHing into each server, running apt-get install nginx, editing config files, creating users, setting firewall rules. Twelve servers. Same commands, twelve times. Sunday evening Ravi made a typo on server 9 — wrong nginx config. That server served 404 for three hours before anyone noticed. Two engineers, two days, one mistake. Ansible would have done all 12 in 4 minutes with zero mistakes.

> **Arjun (You)** _Junior Automation Engineer — Day 1 at StackBridge_
> 
> So Ansible connects to all 12 servers at once and runs the same commands on all of them? How does it know which servers to connect to, and how does it connect — does it need an agent installed on the servers?

> **Gopal** _Senior Automation Engineer_
> 
> No agents. Ansible connects over plain SSH — the same way you SSH manually, but to all servers simultaneously. You list the server IPs in an inventory file. You write the tasks you want done in a playbook YAML file. You run one command: ansible-playbook setup.yml. Ansible reads the inventory, SSHes into every server in parallel, runs your tasks, and prints the result. If a task fails on one server, it tells you exactly which server and which task failed — and stops only that server, continues the rest.

> **Meenakshi** _DevOps Lead — StackBridge_
> 
> And idempotency — that is the word you need to understand today. Every Ansible task is designed to be safe to run multiple times. If you run "install nginx" and nginx is already installed, Ansible checks, sees it's there, and does nothing. It does not install it again. It does not break anything. Run the same playbook 10 times and the result is always the same. That makes automation safe to run in production.

### 1. Phase 1 — Ansible Foundations

Before writing any automation, understand the three things that make Ansible work: the **control node** (your machine, where Ansible is installed), the **inventory** (the list of servers), and the **playbook** (the instructions). Ansible connects from your machine to the servers over SSH — no software needed on the servers except Python 3.

> **The Big Picture — How Ansible Works**

> You write a playbook describing what you want done. You run `ansible-playbook`. Ansible reads your inventory to get the list of servers, SSHes into each one, pushes tiny Python scripts (called modules) to them, runs those scripts to do the actual work, then removes them. The servers don't keep any Ansible software. Every time Ansible runs, it connects fresh, checks the current state of the server, and only does what needs to be done. That "only do what's needed" behaviour is called **idempotency** — the most important concept in Ansible.

```
How Ansible Connects and Runs Tasks
======================================

  Your Laptop / Control Node
  (Ansible installed here)
        │
        │  ansible-playbook webserver.yml
        │
        ├──── SSH ──────────► web-server-01 (192.168.1.10)
        │                     Pushes Python module
        │                     Runs task: install nginx
        │                     Reports: changed / ok / failed
        │
        ├──── SSH ──────────► web-server-02 (192.168.1.11)
        │                     Same tasks run in parallel
        │
        └──── SSH ──────────► web-server-03 (192.168.1.12)

  Key: No agent needed on servers.
  Servers only need: SSH access + Python 3 installed.
  Ansible runs tasks in parallel across all servers simultaneously.
```

#### 1.1 Install Ansible on Your Control Node

1. Install Ansible (Ubuntu/Debian)

```bash
# Install Ansible
sudo apt update
sudo apt install ansible -y

# Verify
ansible --version
```

> This installs Ansible on your laptop or control server. After installation, **ansible --version** prints the version and the location of config files. Ansible only needs to be installed on the machine you run it from — not on any of the servers you manage.

#### 1.2 The Inventory File — Tell Ansible Which Servers to Manage

The inventory is a file that lists the IP addresses or hostnames of all the servers Ansible can manage. It is the starting point of every Ansible project.

```
# inventory.ini — list all servers
[webservers]
192.168.1.10
192.168.1.11
192.168.1.12

[dbservers]
192.168.1.20
192.168.1.21

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

**📖 What This Inventory File Does**

**[webservers]** — a group name in square brackets. You can run a playbook on "all webservers" by targeting this group name.  
  
**[dbservers]** — another group. You can have as many groups as you need.  
  
**[all:vars]** — variables that apply to ALL servers in this inventory.  
  
**ansible_user** — the SSH username Ansible uses to connect.  
  
**ansible_ssh_private_key_file** — path to your SSH private key.

#### 1.3 Your First Ad-Hoc Command — Test the Connection

Before writing a playbook, test that Ansible can reach all servers using an ad-hoc command. Ad-hoc commands are one-off tasks you run directly from the terminal — no playbook needed.

```bash
# Test if all webservers are reachable
ansible webservers -i inventory.ini -m ping
```

> **webservers** — the group to target from the inventory.  
**-i inventory.ini** — which inventory file to use.  
**-m ping** — the module to run. The `ping` module SSHes to each server and checks connectivity (it doesn't send a network ping — it checks SSH works and Python is installed). If you see **"pong"** from each server, Ansible can manage them.

```
192.168.1.10 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
192.168.1.11 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
192.168.1.12 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

> **💡 Fresher Tip — changed vs ok in Ansible Output**

> Every Ansible task reports one of three results: **ok** (task ran, nothing needed to change — server was already in the correct state), **changed** (task ran and actually changed something on the server — like installed a package), or **failed** (task could not complete). The goal of good Ansible is to see **ok** on repeat runs — meaning the server is already in the desired state and Ansible had nothing to do. This "do-nothing on repeat runs" is idempotency, and it's what makes Ansible safe to run in CI/CD pipelines daily.

### 2. Phase 2 — Playbooks and Modules

**Business Problem:** StackBridge needs to set up nginx on 12 new client servers — same configuration on every server. Manual approach: SSH into each server, run 4 commands, edit a config file, restart nginx. 12 times. 1 mistake per 5 attempts statistically. Ansible approach: write a playbook once, run it against all 12 servers. Done in 3 minutes.

**Scene 2 — StackBridge Operations | "Set Up 12 Servers by Monday"**

> **Gopal** _Senior Automation Engineer_
> 
> Arjun, the client just sent access to 12 new Ubuntu 22.04 servers. They need nginx installed, a custom index.html deployed, and the ufw firewall configured to allow port 80 and 443. Monday morning 9 AM. Today is Friday 4 PM. Manually, that is 6 hours of work for two engineers over the weekend. With Ansible, write one playbook this afternoon, test on one server, run on all 12. Done by 5 PM today. That's the difference.

#### 2.1 Playbook Structure — The Big Picture First

```bash
---  # Every YAML file starts with ---
- name: Set up nginx web servers
hosts: webservers
become: true
tasks:
    - name: Install nginx
ansible.builtin.apt:
        name: nginx
state: present
```

**📖 Anatomy of a Playbook**

**---** marks the start of a YAML file.  
  
**name:** a human description of this play — shows in output.  
  
**hosts: webservers** — run this play on all servers in the "webservers" group from the inventory.  
  
**become: true** — use sudo. Without this, you can't install packages or edit system files.  
  
**tasks:** the list of things to do, in order, on every matched server.

#### 2.2 The Most Important Modules

Modules are the building blocks of Ansible. Each module does one specific job — install a package, copy a file, manage a service, run a command. You don't write shell scripts — you use modules.

```bash
# apt module — install/remove packages (Ubuntu/Debian)
    - name: Install nginx
ansible.builtin.apt:
        name:  nginx
state: present
update_cache: true
```

**📖 apt Module**

**state: present** — "make sure nginx is installed." If it already is, do nothing (idempotent).  
  
**state: absent** — "make sure nginx is NOT installed." Removes it if present.  
  
**state: latest** — install and keep at the latest version.  
  
**update_cache: true** — equivalent to running `apt-get update` before installing.

```bash
# service module — start/stop/enable services
    - name: Start and enable nginx
ansible.builtin.service:
        name:    nginx
state:   started
enabled: true
```

**📖 service Module**

**state: started** — ensure the service is running. If already running, do nothing.  
  
**state: stopped** — ensure the service is stopped.  
  
**state: restarted** — always restart (not idempotent — use handlers instead).  
  
**enabled: true** — ensure it starts automatically on server reboot (equivalent to `systemctl enable nginx`).

```bash
# copy module — copy files to remote servers
    - name: Deploy custom index page
ansible.builtin.copy:
        src:   files/index.html
dest:  /var/www/html/index.html
owner: www-data
mode:  '0644'
```

**📖 copy Module**

**src** — path to the file on your control node (local machine).  
  
**dest** — where to put it on the remote server.  
  
**owner** — sets file ownership (www-data is the nginx user).  
  
**mode: '0644'** — file permissions (readable by owner and group, not executable). Always quote mode values as strings to avoid octal issues.

```bash
# file module — create directories, set permissions
    - name: Create app directory
ansible.builtin.file:
        path:  /opt/stackbridge/app
state: directory
owner: deploy
mode:  '0755'
```

**📖 file Module**

**state: directory** — create this path as a directory if it doesn't exist.  
  
**state: file** — ensure this is a regular file.  
  
**state: absent** — ensure this path does not exist (delete it).  
  
**state: link** — create a symbolic link.  
  
All operations are idempotent — if the directory already exists with correct permissions, nothing changes.

```bash
# user module — manage system user accounts
    - name: Create deploy user
ansible.builtin.user:
        name:     deploy
groups:   sudo
shell:    /bin/bash
state:    present
```

**📖 user Module**

Creates or manages a Linux user account.  
  
**name** — the username to create.  
  
**groups** — add this user to the sudo group (for admin access).  
  
**shell** — the user's default shell.  
  
**state: absent** — remove the user account.

```bash
# command module — run a shell command
    - name: Check nginx config is valid
ansible.builtin.command:
        cmd: nginx -t
register: nginx_test
changed_when: false
```

**📖 command Module**

Runs a command on the remote server when no dedicated module exists.  
  
**register: nginx_test** — saves the command output into a variable so you can check it in later tasks.  
  
**changed_when: false** — tells Ansible this command never changes state (it's a read-only check), so always report "ok" not "changed".  
  
**Use the command module sparingly** — prefer dedicated modules (apt, service, file) which are idempotent. command is not idempotent by default.

#### 2.3 Run the Playbook

```bash
# Run the playbook against the webservers group
ansible-playbook -i inventory.ini webserver.yml

# Dry run — show what WOULD change, without changing anything
ansible-playbook -i inventory.ini webserver.yml --check

# Run on just one server to test first
ansible-playbook -i inventory.ini webserver.yml --limit 192.168.1.10
```

> **--check** (dry run) is one of Ansible's best features — it simulates the run and shows you what would change without actually doing anything. Always run --check first on production servers.  
**--limit** restricts execution to a subset of the inventory — use this to test on one server before running on all 12.  
**-v, -vv, -vvv** increase verbosity — add these when debugging a failing task to see exactly what Ansible is doing.

```
PLAY [Set up nginx web servers] ********************

TASK [Gathering Facts] *****************************
ok: [192.168.1.10]
ok: [192.168.1.11]
ok: [192.168.1.12]

TASK [Install nginx] ********************************
changed: [192.168.1.10]
changed: [192.168.1.11]
changed: [192.168.1.12]

TASK [Start and enable nginx] ***********************
changed: [192.168.1.10]
changed: [192.168.1.11]
changed: [192.168.1.12]

PLAY RECAP ******************************************
192.168.1.10  : ok=3  changed=2  unreachable=0  failed=0
192.168.1.11  : ok=3  changed=2  unreachable=0  failed=0
192.168.1.12  : ok=3  changed=2  unreachable=0  failed=0
```

### 3. Phase 3 — Variables, Facts and Templates

**Business Problem:** StackBridge manages 5 different clients. Each client needs a slightly different nginx configuration — different domain names, different server names, different port numbers. Instead of writing 5 separate playbooks, you write one playbook with variables — and each client gets the right config automatically.

**Scene 3 — Client Meeting, StackBridge | "Each Client Has Different Settings"**

> **Meenakshi** _DevOps Lead — StackBridge_
> 
> Arjun, I've been copying the same playbook and editing the domain name and port for each client. It works, but now I have 5 nearly identical playbooks to maintain. If I need to change the nginx install command, I change it 5 times. That's how bugs get introduced. The right approach: one playbook, variables for everything that differs between clients. Variables go in separate files per client. The playbook stays the same forever.

#### 3.1 Defining and Using Variables

```
# In the playbook — vars section
- name: Configure nginx for client
hosts: webservers
become: true
vars:
    app_name:   client-portal
http_port:  80
server_name: portal.acme.com
```

**📖 Defining Variables in the Playbook**

**vars:** section defines variables inline in the playbook. These are available to all tasks below it.  
  
Variables are referenced with double curly braces: `{{ app_name }}`, `{{ http_port }}`.  
  
For real projects, put variables in separate files (vars/main.yml) or pass them at runtime — not hardcoded in the playbook. Inline vars are fine for learning and simple cases.

```bash
# Use variables in tasks with {{ }}
    - name: Create app directory
ansible.builtin.file:
        path:  /var/www/{{ app_name }}
state: directory

    - name: Show the server name
ansible.builtin.debug:
        msg: "Configuring {{ server_name }}"
```

**📖 Using Variables**

**{{ app_name }}** — double curly braces reference a variable. Ansible replaces this with the variable's value at runtime.  
  
Variables can be used in almost any field — path, name, command, message, etc.  
  
**ansible.builtin.debug** — prints a message during playbook execution. Very useful for debugging: show what a variable contains during a run.

#### 3.2 Group Variables — Different Settings Per Group

```
# group_vars/webservers.yml
# Variables for ALL servers in the webservers group
http_port:        80
nginx_user:       www-data
max_connections:  1024
log_path:         /var/log/nginx
```

**📖 group_vars — Auto-loaded Variables**

Create a folder called `group_vars/` next to your inventory file. Inside it, create a file named after each inventory group (e.g. `webservers.yml`). Ansible automatically loads these variables for all hosts in that group — no need to reference them manually.  
  
This is the standard way to organise variables in production: one file per group, lives next to the inventory.

#### 3.3 Ansible Facts — Automatic Server Info

Ansible automatically collects information about each server before running tasks — OS type, IP address, memory, disk space, hostname. These are called **facts**. You can use them in tasks and templates without defining them yourself.

```bash
# Facts are automatically available as variables
    - name: Show server OS details
ansible.builtin.debug:
        msg: "{{ ansible_hostname }} runs
              {{ ansible_distribution }}
              {{ ansible_distribution_version }}"
```

**📖 Useful Built-in Facts**

**ansible_hostname** — the server's hostname.  
**ansible_distribution** — OS name (Ubuntu, CentOS, Debian).  
**ansible_distribution_version** — OS version (22.04, 8, etc.).  
**ansible_default_ipv4.address** — primary IP address.  
**ansible_memtotal_mb** — total RAM in MB.  
**ansible_os_family** — OS family (Debian, RedHat).  
  
Run `ansible webservers -m setup` to see ALL facts for a server.

#### 3.4 Jinja2 Templates — Generate Config Files Dynamically

Instead of copying a static config file to every server (which gives every server the exact same file), use a **template**. A template has placeholders like `{{ server_name }}` that Ansible fills in with the right variable value for each server.

```
# templates/nginx.conf.j2 — the template file
server {
    listen {{ http_port }};
    server_name {{ server_name }};

    root /var/www/{{ app_name }};

    access_log /var/log/nginx/{{ app_name }}_access.log;
    error_log  /var/log/nginx/{{ app_name }}_error.log;
}
```

**📖 Jinja2 Template (.j2 file)**

Template files have the **.j2** extension (Jinja2 = the templating language Ansible uses).  
  
Any **{{ variable }}** placeholder gets replaced with the variable's actual value when Ansible renders the template.  
  
For server1: `server_name portal.acme.com` → the rendered file has `server_name portal.acme.com;`  
For server2 with `server_name api.xyz.co` → different rendered file, different name.

```bash
# Use template module to deploy the rendered file
    - name: Deploy nginx config from template
ansible.builtin.template:
        src:   templates/nginx.conf.j2
dest:  /etc/nginx/sites-available/app.conf
owner: root
mode:  '0644'
```

**📖 template Module**

The **template** module renders the .j2 file (substitutes all variables) and copies the result to `dest` on the remote server.  
  
The difference from **copy**: copy sends the file as-is; template first fills in all the `{{ }}` placeholders with values appropriate for each server, then sends the rendered version.  
  
Ansible also detects if the rendered output changed since last run — if nothing changed, it reports "ok" (idempotent).

### 4. Phase 4 — Handlers, Conditions and Loops

**Business Problem:** When the nginx config file changes, nginx must be restarted to apply the new config. But you only want to restart it if the config actually changed — not every time the playbook runs. Handlers solve this. And sometimes a task should only run on Ubuntu servers, not CentOS. And sometimes you need to install 10 packages — you don't want to write 10 separate tasks. Conditions and loops solve these.

#### 4.1 Handlers — Run Only When Something Changes

```bash
# Task that notifies a handler
tasks:
    - name: Deploy nginx config
ansible.builtin.template:
        src:  templates/nginx.conf.j2
dest: /etc/nginx/sites-available/app.conf
notify: Restart nginx
```

**📖 notify — Trigger a Handler**

**notify: Restart nginx** — "if this task made a change (reported 'changed'), queue the handler named 'Restart nginx' to run at the end of the play."  
  
If the template task reported "ok" (config didn't change) — the handler is NOT triggered. Nginx is NOT restarted.  
  
If the config changed — handler runs. Nginx restarts once at the end (even if 5 tasks all notified the same handler).

```bash
# handlers section — define the handler
handlers:
    - name: Restart nginx
ansible.builtin.service:
        name:  nginx
state: restarted
```

**📖 Handlers Section**

**handlers:** is a section at the same level as `tasks:` in the playbook. Handlers look exactly like tasks but they only run when **notified**.  
  
The handler name (`Restart nginx`) must exactly match the name in `notify:` — case-sensitive.  
  
Even if 10 tasks all notify the same handler, it runs only once at the end of the play. This prevents nginx from restarting 10 times unnecessarily.

#### 4.2 when — Run Tasks Conditionally

```bash
# Run this task only on Ubuntu servers
    - name: Install nginx (Debian/Ubuntu)
ansible.builtin.apt:
        name:  nginx
state: present
when: ansible_os_family == "Debian"
```

**📖 when — Conditional Execution**

**when:** takes a condition. If the condition is true, the task runs. If false, Ansible skips it (shows "skipping" in output).  
  
This is how you write one playbook that works on both Ubuntu and CentOS. Use apt for Debian/Ubuntu, use yum for RedHat/CentOS — controlled by `when: ansible_os_family == "Debian"` vs `when: ansible_os_family == "RedHat"`.  
  
`ansible_os_family` is an Ansible fact gathered automatically.

```bash
# Check result of a previous task
    - name: Check if app is running
ansible.builtin.command:
        cmd: systemctl is-active myapp
register: app_status
ignore_errors: true

    - name: Start app only if stopped
ansible.builtin.service:
        name:  myapp
state: started
when: app_status.rc != 0
```

**📖 register + when Together**

**register: app_status** — saves the output of the command into a variable named `app_status`.  
  
**app_status.rc** — the return code (0 = success, non-zero = failure).  
  
**ignore_errors: true** — if the command fails (app not running), don't stop the playbook.  
  
Together: "check if the app is running, and only start it if it's not" — a safe conditional start pattern used in production deployments.

#### 4.3 loop — Repeat a Task Over a List

```bash
# Install many packages with one task
    - name: Install required packages
ansible.builtin.apt:
        name:  "{{ item }}"
state: present
loop:
        - nginx
        - git
        - curl
        - unzip
        - python3-pip
```

**📖 loop — Iterate Over a List**

**loop:** makes Ansible run the task once for each item in the list.  
  
**{{ item }}** — the special variable that holds the current loop value. On the first iteration, item = nginx. Second: git. And so on.  
  
Without loop, you'd write 5 separate tasks. With loop, one task installs all 5 packages. The output shows each package as a separate sub-task.

```bash
# Loop over dictionaries (more complex data)
    - name: Create multiple users
ansible.builtin.user:
        name:   "{{ item.name }}"
groups: "{{ item.group }}"
state:  present
loop:
        - { name: arjun,  group: devops }
        - { name: meena,  group: devops }
        - { name: ravi,   group: ops    }
```

**📖 Looping Over Dictionaries**

When each item has multiple fields, use a list of dictionaries (key-value pairs in curly braces).  
  
**{{ item.name }}** — access the `name` key of the current dictionary item.  
  
**{{ item.group }}** — access the `group` key.  
  
This creates 3 users with 3 different group assignments — all in one task. Compare to writing 3 separate user tasks!

### 5. Phase 5 — Roles, Vault and a Complete Project

**Business Problem:** StackBridge's playbooks have grown to hundreds of tasks across multiple files — all mixed together. And database passwords are sitting in plaintext in variable files. Two problems: one architectural (messy, hard to reuse), one security (passwords in plain text). Ansible Roles solve the first. Ansible Vault solves the second.

**Scene 5 — StackBridge Code Review | "This Is Getting Impossible to Maintain"**

> **Gopal** _Senior Automation Engineer_
> 
> Arjun, look at our deploy.yml — it's 400 lines of mixed nginx setup, database config, user creation, and firewall rules. When a new client comes in, we copy the whole thing and start editing. If the nginx section needs a fix, we fix it in 5 copies. Roles fix this. A Role is a folder structure that packages one complete job — like "set up nginx" or "configure postgres". Each Role is self-contained: its own tasks, variables, templates, and handlers. You call roles from the playbook like functions.

> **Meenakshi** _DevOps Lead — StackBridge_
> 
> And the passwords — db_password, api_secret_key — they're in our vars files in plain text, committed to Git. Anyone with repo access can read the production database password. Ansible Vault encrypts those files. You edit them normally, but they're stored as encrypted ciphertext in Git. Only someone with the vault password can decrypt them. No plaintext secrets ever in Git again.

#### 5.1 Roles — Organising Playbooks at Scale

```
Ansible Role Directory Structure
===================================

  roles/
  └── nginx/                 ← Role name
      ├── tasks/
      │   └── main.yml       ← All tasks for this role
      ├── handlers/
      │   └── main.yml       ← All handlers for this role
      ├── templates/
      │   └── nginx.conf.j2  ← Jinja2 templates
      ├── files/
      │   └── index.html     ← Static files to copy
      ├── vars/
      │   └── main.yml       ← Role-specific variables
      └── defaults/
          └── main.yml       ← Default values (can be overridden)

  Calling a Role from a Playbook:
  - hosts: webservers
    roles:
      - nginx        ← runs roles/nginx/tasks/main.yml automatically
```

```bash
# roles/nginx/tasks/main.yml
---
- name: Install nginx
ansible.builtin.apt:
    name:  nginx
state: present

- name: Deploy config from template
ansible.builtin.template:
    src:  nginx.conf.j2
dest: /etc/nginx/sites-available/app.conf
notify: Restart nginx
```

**📖 Role Tasks File**

Inside a role, tasks go in `tasks/main.yml`. Ansible automatically runs this file when the role is called — you don't need to reference it explicitly.  
  
Paths to templates and files are automatically resolved relative to the role's own `templates/` and `files/` folders — no need for full paths.  
  
Handlers in `handlers/main.yml` are also automatically loaded — the `notify: Restart nginx` will find the handler in the role's own handlers file.

```
# site.yml — call roles from a playbook
---
- name: Configure all web servers
hosts: webservers
become: true
roles:
    - nginx
    - firewall
    - monitoring
```

**📖 Calling Multiple Roles**

The playbook becomes very clean — it just says which roles to apply to which hosts.  
  
Ansible runs the roles in order: first `nginx`, then `firewall`, then `monitoring`.  
  
Each role is completely self-contained — you can reuse the nginx role in a different playbook for a different client without copying any code. Just list it under `roles:`.

#### 5.2 Ansible Vault — Encrypt Sensitive Data

```bash
# Create an encrypted vault file
ansible-vault create vars/secrets.yml

# Edit an existing encrypted file
ansible-vault edit vars/secrets.yml

# Encrypt an existing plain-text file
ansible-vault encrypt vars/secrets.yml

# View contents without editing
ansible-vault view vars/secrets.yml
```

> These commands open your default editor (vim or nano). You write normal YAML — Ansible encrypts it when you save. The file in Git looks like `$ANSIBLE_VAULT;1.1;AES256\n3434343...` — completely unreadable without the vault password. Only the vault password (which you never commit) can decrypt it.

```
# vars/secrets.yml (decrypted — what you write)
db_password:    SuperSecret@2026!
api_secret_key: rzpkey_live_abc123xyz
smtp_password:  mailpass_789
```

**📖 What the Vault File Contains**

Inside a vault file, you write perfectly normal YAML key-value pairs. These become regular Ansible variables once decrypted at runtime.  
  
Reference them the same way as any variable: `{{ db_password }}`, `{{ api_secret_key }}`.  
  
Never put the vault **password** in the repo. Store it in a password manager, CI/CD secrets (GitHub Actions secrets, Jenkins credentials), or use a vault password file.

```bash
# Run playbook with vault — prompts for vault password
ansible-playbook site.yml --ask-vault-pass

# Or use a vault password file (for CI/CD pipelines)
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

> **--ask-vault-pass** prompts you to type the vault password. Ansible decrypts all vault-encrypted files in memory and uses the variables — the encrypted files on disk are never modified.  
**--vault-password-file** reads the password from a file — use this in CI/CD so the pipeline doesn't need human input. Store that password file as a CI/CD secret, never in the repo.

#### 5.3 Complete Project — Web Server Provisioning Playbook

Putting all the concepts together: the complete site.yml that provisions a new client's web servers from scratch using roles, variables, templates, handlers, and vault.

```
StackBridge Complete Project Structure
========================================

  stackbridge-infra/
  ├── site.yml               ← Master playbook: calls all roles
  ├── inventory.ini          ← Server groups: webservers, dbservers
  ├── group_vars/
  │   ├── webservers.yml     ← Variables for web server group
  │   └── dbservers.yml      ← Variables for DB server group
  ├── vars/
  │   └── secrets.yml        ← Ansible Vault encrypted file
  └── roles/
      ├── common/            ← Tasks for ALL servers (updates, base tools)
      ├── nginx/             ← Install and configure nginx
      ├── postgresql/        ← Install and configure PostgreSQL
      ├── deploy_app/        ← Clone repo, run migrations, start app
      └── firewall/          ← Configure ufw rules

  Run order:
  1. common  → runs on all hosts (update packages, create deploy user)
  2. nginx   → runs on webservers (install, template config, enable)
  3. postgresql → runs on dbservers (install, create DB and user)
  4. deploy_app → runs on webservers (pull code, install deps, start)
```

```
# site.yml — the master playbook
---
- name: Common setup for all servers
hosts: all
become: true
vars_files:
    - vars/secrets.yml
roles:
    - common

- name: Set up web servers
hosts: webservers
become: true
roles:
    - nginx
    - deploy_app
    - firewall
```

**📖 Multiple Plays in One Playbook**

A playbook can have multiple **plays** (items starting with `- name:`). Each play targets different hosts and runs different roles.  
  
**vars_files: vars/secrets.yml** — loads the vault-encrypted secrets file. Ansible asks for the vault password once and decrypts all referenced vault files.  
  
The first play runs on `all` servers. The second runs only on `webservers`. Each play is independent and runs in order.

### 6. Essential Ansible Commands Reference

Command

What It Does

When to Use

ansible all -m ping

Test SSH connectivity to all servers

First thing after setting up inventory

ansible webservers -m setup

Show all gathered facts for a host group

Find variable names for conditions/templates

ansible all -m command -a "uptime"

Run an ad-hoc command on all servers

Quick one-off checks without a playbook

ansible-playbook site.yml

Run a playbook

Deploy or configure servers

ansible-playbook site.yml --check

Dry run — show changes without applying

Always before running on production

ansible-playbook site.yml --diff

Show file diffs when templates/files change

See exactly what will change in config files

ansible-playbook site.yml -v

Verbose output (add more v's for more detail)

Debug failing tasks

ansible-playbook site.yml --tags nginx

Run only tasks tagged with "nginx"

Apply changes to one component only

ansible-playbook site.yml --skip-tags deploy

Skip tasks with this tag

Run everything except deployment steps

ansible-playbook site.yml --limit 192.168.1.10

Run only on specific host(s)

Test on one server before running on all

ansible-vault create secrets.yml

Create a new encrypted file

Store new secrets securely

ansible-vault edit secrets.yml

Edit an encrypted vault file

Update or add secrets

ansible-vault view secrets.yml

Read vault file without editing

Check what's stored in the vault

ansible-galaxy role init myrole

Create the role folder structure automatically

Start a new role from scratch

ansible-lint site.yml

Check playbooks for best practices and errors

Before committing playbooks to Git

### 7. Interview Questions — Ansible

##### Interview Q&A — Fresher Level (0–1 Year Ansible Experience)

**Q: Q1. What is Ansible and how is it different from shell scripts?**

A: Ansible is an agentless IT automation tool that manages servers over SSH by running YAML-based playbooks. The key differences from shell scripts: (1) **Idempotency** — Ansible tasks check the current state before acting. If nginx is already installed, the apt task does nothing. A shell script with `apt-get install nginx` would run every time. (2) **No agents** — shell scripts require you to copy and execute them on each server; Ansible uses SSH. (3) **Readability** — Ansible YAML describes what you want (declarative); shell scripts describe how to do it (imperative). (4) **Error handling** — Ansible automatically stops on failure and reports which server and task failed. Shell scripts require manual error checking with `$?`.

**Q: Q2. What is idempotency in Ansible and why does it matter?**

A: Idempotency means running the same task multiple times produces the same result as running it once — it's safe to run repeatedly. For example, the Ansible `apt` module with `state: present` first checks if the package is already installed. If it is, it does nothing and reports "ok". If not, it installs and reports "changed". This matters because in production, you often run the same playbook multiple times — to verify state, after adding new servers, or in a CI/CD pipeline that runs daily. Idempotent playbooks are safe to run without fear of causing unintended changes. The `command` and `shell` modules are NOT idempotent by default — that's why you should prefer dedicated modules (apt, service, file, user) whenever possible.

**Q: Q3. What is the difference between a task, a play, and a playbook?**

A: A **task** is a single action to perform on a server — like "install nginx" or "copy this file". Each task calls one Ansible module. A **play** is a set of tasks that run against a specific group of hosts — it has a `hosts:` field, optional variables, and a `tasks:` list. A **playbook** is a YAML file that contains one or more plays. For example, a playbook might have one play targeting webservers (install nginx, deploy app) and another play targeting dbservers (install postgres, create database). You run a playbook with `ansible-playbook`. You never run individual tasks directly — tasks always belong to a play inside a playbook (or to a Role).

**Q: Q4. What is a Handler in Ansible and how is it different from a regular task?**

A: A handler is a special task that only runs when it is explicitly notified by another task's `notify:` directive. Regular tasks run every time the playbook runs. Handlers run only when triggered by a change. For example: a "Restart nginx" handler is notified by the "Deploy nginx config" task. If the config file didn't change during this run, the task reports "ok" and the handler is never triggered — nginx is not restarted unnecessarily. If the config changed, the task reports "changed", the handler is queued, and it runs once at the end of the play. Even if 10 tasks all notify the same handler, it runs only once. This prevents unnecessary restarts and is more efficient than calling `state: restarted` in a task.

**Q: Q5. What is Ansible Vault and when would you use it?**

A: Ansible Vault is a feature that encrypts sensitive data at rest using AES-256 encryption. You use it to store secrets like database passwords, API keys, and SSL certificate private keys in your Ansible variable files without exposing them in plaintext in Git. Commands: `ansible-vault create secrets.yml` to create an encrypted file; `ansible-vault edit secrets.yml` to update it; `ansible-vault view secrets.yml` to read it. At runtime, you provide the vault password via `--ask-vault-pass` or `--vault-password-file`. Ansible decrypts in memory and uses the variables normally. Vault-encrypted files look like unreadable ciphertext in Git — safe to commit. The vault password itself should never be in the repo — store it in CI/CD secrets (GitHub Actions secrets, Jenkins credentials).

**Q: Q6. What is an Ansible Role and why should you use them instead of one big playbook?**

A: An Ansible Role is a standardised directory structure that packages all the components of one job — tasks, handlers, templates, files, variables, and defaults — into a reusable unit. For example, an "nginx" role contains everything needed to install and configure nginx. You call it by listing it under `roles:` in a playbook. Roles solve the scalability problem of large playbooks: (1) **Reusability** — the nginx role can be used for any client's playbook without copying code; (2) **Separation of concerns** — nginx logic lives in the nginx role, postgres logic lives in the postgres role; (3) **Maintainability** — fix a bug in the nginx role once and it's fixed everywhere it's used; (4) **Shareability** — Ansible Galaxy (galaxy.ansible.com) hosts thousands of public roles you can download and use in your project.

**Quiz: Quiz 1 — A task shows "ok" instead of "changed" when you run your playbook the second time. What does this mean?**

- A) The task failed silently — "ok" means error
- B) Ansible skipped the task because of a when condition
- C) The server was already in the desired state — the task ran but found nothing needed to change. This is idempotency working correctly.
- D) The task did not run at all

> **Answer/explanation:** ✅ Answer: C. "ok" in Ansible output means the task ran, checked the server's state, found it was already correct, and did nothing. "changed" means the task ran and actually modified something. On the first run: nginx isn't installed → task installs it → "changed". On the second run: nginx is already installed → task checks, does nothing → "ok". This is idempotency — safe to run repeatedly. In a healthy CI/CD pipeline that runs daily, most tasks should show "ok" because the servers are already in the desired state.

**Quiz: Quiz 2 — What is the difference between "copy" and "template" modules?**

- A) copy is for binary files, template is for text files
- B) copy sends the file exactly as-is; template first fills in {{ variable }} placeholders with values, then sends the rendered result
- C) copy is faster; template uses more memory
- D) template requires root access; copy does not

> **Answer/explanation:** ✅ Answer: B. The copy module transfers a static file unchanged — every server gets the exact same file. The template module processes a .j2 (Jinja2) file first: it substitutes all `{{ variable }}` placeholders with their values for each specific server, then sends the rendered version. Use copy for static files that are identical on all servers (like a script or an image). Use template for config files that differ per server (nginx.conf with different server_name, database.conf with different hostname) — one template, many different rendered outputs.

**Quiz: Quiz 3 — When should you use a Handler instead of a task with "state: restarted"?**

- A) Always — handlers are always better than tasks
- B) Never — tasks are simpler and clearer
- C) When you only want to restart a service IF something actually changed — handlers are triggered only when a notifying task reports "changed", preventing unnecessary restarts
- D) When you need the restart to happen immediately in the middle of a play, not at the end

> **Answer/explanation:** ✅ Answer: C. A task with `state: restarted` restarts the service every single time the playbook runs — even if the config didn't change. This causes unnecessary downtime and is not idempotent. A handler with `notify:` only restarts when the notifying task actually changed something. If the nginx config template task reports "ok" (config unchanged), the "Restart nginx" handler is never triggered and nginx keeps running uninterrupted. This is both safer and more efficient. Note from option D: handlers always run at the END of the play by default — if you need immediate restart in the middle, use `meta: flush_handlers`.

> **Ansible Project — Core Takeaways for Freshers**

> - Ansible is agentless — it connects over SSH and requires nothing pre-installed on the servers except Python 3. This is its biggest advantage over tools like Puppet and Chef which require agents on every managed server.
> - Idempotency is the most important concept — every Ansible module is designed to check the current state before acting. Run the same playbook 10 times and only the first run makes changes. This makes Ansible safe to run in production daily.
> - Always use dedicated modules (apt, service, file, user, template) over the command or shell modules. Dedicated modules are idempotent. command and shell are not — they run every time and can cause unintended side effects.
> - Use --check (dry run) before running any playbook on production. It shows exactly what would change without touching the servers. Use --diff alongside --check to see the exact content of file changes.
> - Use Handlers to restart services — never use state: restarted in a task. Handlers run only when something actually changed, preventing unnecessary service restarts which cause downtime.
> - Never commit secrets in plaintext to Git. Use Ansible Vault to encrypt sensitive variable files. The vault password lives in your CI/CD secrets manager — never in the repository.
> - Organise large projects into Roles. One role per responsibility (nginx, postgresql, firewall). Roles make your automation reusable, testable, and maintainable as the number of clients and servers grows.
> - Test every playbook with --limit on a single server before running against your entire inventory. One failing task on one server is far better than the same failing task on 60 servers simultaneously.

##### Ansible Code Standards — StackBridge Engineering Rules

- Name every task descriptively — "Install nginx" not "task1". The name appears in output and is how you find failing tasks in long playbook runs
- Use ansible.builtin.module_name (FQCN — Fully Qualified Collection Name) instead of just module_name to avoid ambiguity and ensure the correct module runs when multiple collections are installed
- Add tags to tasks and roles so you can run specific parts of a playbook without running everything: `tags: [nginx, config]` on a task enables `--tags nginx` at runtime
- Store variables in group_vars/ and host_vars/ directories next to the inventory — not hardcoded in playbooks. This separates what to do (playbook) from the values used (variables)
- Run ansible-lint on all playbooks before committing to Git — it catches common mistakes, deprecated syntax, and style violations that could cause problems later
- Use become: true only at the task level when needed, not at the play level for everything — principle of least privilege: only escalate to sudo when the task actually requires it

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Build a complete LAMP stack role:** Create a role called `lamp` that installs Apache (or nginx), MySQL, and PHP on a single server. Use the apt module for each package, the service module to start and enable each service, and a template to deploy a test PHP info page. The role should be callable with a single line under `roles:` in any playbook.
2. **Write a user management playbook:** Create a playbook that reads a list of users from a variable (as a list of dictionaries with name, group, and ssh_key fields) and uses loop to create all users, assign them to the correct group, and deploy their SSH public key using the `authorized_key` module. Test removing users by changing `state: absent` for one user and verifying they're removed.
3. **Add error handling with blocks:** Ansible's `block/rescue/always` is equivalent to try-catch-finally in programming. Wrap a deployment task in a block, add a rescue section that rolls back the deployment if the task fails, and an always section that sends a Slack notification (using the `community.general.slack` module) regardless of success or failure.
4. **Create a dynamic inventory:** Instead of a static inventory.ini file, write a Python or shell script that queries your cloud provider's API (AWS EC2, DigitalOcean, etc.) and outputs a JSON inventory. Ansible accepts dynamic inventories via `-i inventory_script.py`. This means new servers appear in your inventory automatically when they're launched.
5. **Write a patching playbook with a maintenance window:** Create a playbook that: (1) checks if a reboot is required after `apt upgrade` by registering the output of `cat /var/run/reboot-required`, (2) reboots the server only if that file exists using a `when` condition, (3) waits for the server to come back online using `ansible.builtin.wait_for_connection`, and (4) verifies the new kernel version with a debug task. This is a real patching workflow used in production.

### Ansible Project Complete 🎉

You have built StackBridge's complete Ansible automation framework — inventory files, playbooks with tasks, the most important modules, variables and group_vars, Jinja2 templates, handlers, when conditions, loops, reusable roles, and Ansible Vault for secrets. What used to take two engineers two days of SSH commands now runs in under 5 minutes, consistently, across any number of servers.

> **Gopal**
> 
> "Arjun, the new client's 12 servers — fully configured, nginx running, firewall set, deploy user created, app deployed. Total time: 4 minutes 22 seconds. One command. Last month this took two engineers a full weekend and we still had a config error on server 9. You wrote the playbook in one afternoon and tested it on one server. Applied to all 12. Zero errors. That is what automation looks like. That is why StackBridge hired an automation engineer."

> **Meenakshi**
> 
> "The vault file you set up means no plaintext secrets in Git for the first time in this company's history. The security audit that failed last quarter — it would pass now. The auditor asked specifically about secrets management. The answer is Ansible Vault, encrypted at AES-256, vault password stored in Jenkins credentials, never in the repository."

> **Ravi**
> 
> "And those roles — the nginx role, the firewall role, the common role — I used them for the new client onboarding yesterday. Different inventory, different group_vars, same roles. New client's 8 servers configured in 3 minutes. The role I didn't write, on servers I've never touched. That is what reusable automation means."

> **Next: Advanced Ansible — Dynamic Inventories, Molecule Testing & AWX/Tower**

> - Dynamic inventories — query AWS EC2, Azure VMs, or GCP automatically so new servers appear without editing inventory files
> - Ansible Galaxy — download and use community-written roles for databases, monitoring agents, Docker, and more
> - Molecule — test framework for Ansible roles: spin up a Docker container, run your role inside it, verify the result, destroy — all automated
> - AWX / Ansible Tower — web UI and API for Ansible: run playbooks from a browser, manage credentials securely, schedule runs, and audit who ran what and when
> - Ansible collections — the modern way to package and distribute Ansible content (roles + modules + plugins) from community.general, amazon.aws, kubernetes.core
> - Error handling with blocks — block/rescue/always for try-catch-finally patterns in playbooks
