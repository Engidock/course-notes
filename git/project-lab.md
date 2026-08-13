# Git Project Mastery

> **👋 Hey Fresher — Read This First!**

> Git is the tool every software team on the planet uses to track changes to code. Without Git, every edit you make can overwrite someone else's work, there's no history of what changed or why, and rolling back a broken release means manually deleting files. Git solves all of that — every change is saved, labelled, reversible, and shareable. This document uses **short command blocks** — each one covers exactly one concept — with a plain-English explanation right beside it. No 30-command sequences to get lost in. One idea at a time, always explained in simple language.

> **Company in this project:** NovaPay — the fintech startup in Mumbai you already know from the Kubernetes project. They have a Node.js payment API team (3 developers), a React frontend team, and a QA team. They are using a single shared folder on a USB drive right now. Your lead is Tanvi. You are about to introduce Git and save them from themselves.

#### What You Will Learn and Build in This Project

You will set up a real Git workflow for NovaPay — from `git init` on day one to pull requests, protected branches, conflict resolution, and CI hooks — learning every core Git concept along the way, each explained with a real business reason.

Init & Config, Staging & Commits, Branching, Merging, Remote / GitHub, Pull Requests, Conflict Resolution, Rebase, Stash & Tags

> **📦 Phase 1 — Foundations**

> Understand what Git is, install and configure it, understand the three areas (working dir, staging, repository), and make your first commit.

> Create feature branches, switch between them, understand HEAD, and merge branches without losing any work.

> Push your repository to GitHub, set up origin, push and pull, understand tracking branches, and work with teammates.

> Pull requests, code review workflow, merge conflicts, rebase, cherry-pick, stash, tags, and Git hooks for CI/CD.

### 1. Phase 1 — Git Foundations

**Business Problem:** NovaPay's three backend developers are emailing ZIP files of their code to each other. Last week, Dev B's email overwrote Dev A's payment processing fix. The fix was lost. The build broke. Payments failed. Git ends this immediately — every developer has the full history, and no one can accidentally overwrite anyone else's work.

**Scene 1 — NovaPay Engineering Room | Day of the Lost Fix**

> **Roshan** _DevOps Architect — NovaPay_
> 
> Priya, look at this. Kiran sent a ZIP file of the payment module to the shared folder yesterday. Dev B — Arjun — downloaded it, added his UPI integration, and re-uploaded it. Kiran's critical null-pointer fix is gone. Completely overwritten. We have no record of what Kiran changed. This is a ₹15 lakh bug because it's sitting in production right now and we don't even know what the fix looked like.

> **Tanvi** _Senior Platform Engineer_
> 
> Priya, your first task: set up Git for the entire backend team. Every change gets committed. Every commit has a message explaining what changed. Every developer works on their own branch. No more ZIPs. No more overwriting. And if we ever need to know what Kiran changed on Tuesday, we just run git log.

#### 1.1 Install and Configure Git

Before anything else, Git needs to know who you are — every commit is stamped with your name and email so the team knows who made each change.

```bash
# Install (Ubuntu/Debian)
sudo apt install git
# Install (macOS)
brew install git
# Check version
git --version
```

**📖 Installing Git**

Git is a command-line tool. Once installed, every `git` command runs from your terminal inside a project folder.  
  
On Windows, install **Git for Windows** from git-scm.com — it includes Git Bash, a terminal that understands all Git commands.  
  
`git --version` confirms the installation worked. You should see something like `git version 2.43.0`.

```bash
# Set your name (appears in every commit)
git config --global user.name "Priya Sharma"
# Set your email
git config --global user.email "priya@novapay.in"
# Confirm your settings
git config --list
```

**📖 Global Config — Set Once, Used Always**

**--global** means this setting applies to every Git repository on your computer — you only run these two commands once per machine.  
  
Your name and email are embedded into every commit you make. On GitHub, your email links your commits to your profile, so your contributions show up on your activity graph.  
  
`git config --list` prints all your current settings — a good way to verify everything is correct before starting.

#### 1.2 The Three Areas of Git

This is the single most important concept in Git. Everything else makes sense once you understand these three areas.

```
  WORKING DIRECTORY          STAGING AREA             REPOSITORY
  (your files on disk)       (changes ready to         (saved snapshots /
                              be committed)              commits)

  payment.js  ──git add──►  payment.js  ──git commit──►  commit abc123
  utils.js                                               commit def456
  index.js                                               commit ghi789

  You edit files here.      You choose WHICH           Permanent history.
  Git sees changes but      changes to include          Can never be lost.
  hasn't saved them yet.    in the next commit.         git log shows this.
```

> **💡 Why Three Areas? Can't Git Just Save Automatically?**

> The staging area (also called the "index") exists so you can be precise about what goes into each commit. Imagine you fixed a payment bug AND added a logging feature in the same editing session. You want two separate, clearly named commits — not one giant "changed stuff" commit. The staging area lets you `git add payment.js` and commit just the bug fix first, then `git add logger.js` and commit the feature separately. Clean history = easy debugging later.

#### 1.3 Initialise a Repository

```bash
# Go into your project folder
cd novapay-api
# Turn this folder into a Git repository
git init
```

**📖 git init — Birth of a Repository**

`git init` creates a hidden `.git/` folder inside your project. That folder is the entire database — every commit, every branch, every piece of history lives inside it.  
  
You only run `git init` once per project. After that, all `git` commands automatically know they're operating on this repository.  
  
If you delete the `.git/` folder, you delete all Git history. The files remain but the version control is gone.

```
Initialized empty Git repository in /home/priya/novapay-api/.git/
```

#### 1.4 Your First Commit

```bash
# See what Git sees in your project
git status
```

**📖 git status — The Most Useful Command**

`git status` is your dashboard. Run it constantly. It tells you: which files have changed, which are staged, and which are untracked (new files Git has never seen before).  
  
Get in the habit: before every `git add` and before every `git commit`, run `git status` first. It prevents mistakes.

```
On branch main

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        payment.js
        utils.js
        package.json

nothing added to commit but untracked files present
```

```bash
# Stage one specific file
git add payment.js
# Stage ALL changed files at once
git add .
```

**📖 git add — Move Files to Staging**

`git add payment.js` stages one file — moves it from "working directory" to the "staging area."  
  
`git add .` stages everything in the current folder — useful when you want all your changes in one commit.  
  
Staging is reversible. If you accidentally stage the wrong file, `git restore --staged payment.js` unstages it without losing your edits.

```bash
# Save staged changes as a commit
git commit -m "Add payment processing module"
```

**📖 git commit — Saving a Snapshot**

A commit is a permanent snapshot of all staged files. Git gives it a unique ID (a hash like `abc1234`) and records your name, email, timestamp, and message.  
  
**-m** means "message" — always write a clear message. Bad: `"fix"`. Good: `"Fix null pointer in UPI payment validation"`. Your team reads these messages every day.  
  
Once committed, the snapshot is permanent — you can always come back to this exact state.

```
[main (root-commit) a3f9c21] Add payment processing module
 3 files changed, 142 insertions(+)
 create mode 100644 payment.js
 create mode 100644 utils.js
 create mode 100644 package.json
```

#### 1.5 Viewing History

```bash
# Full log — author, date, message
git log
# Compact one-line-per-commit view
git log --oneline
# See what changed inside a commit
git show a3f9c21
```

**📖 Reading Git History**

`git log` shows every commit — newest first. Each entry shows the commit hash (unique ID), author, date, and message.  
  
`git log --oneline` is the same thing compressed to one line per commit — much easier to scan when there are 500 commits.  
  
`git show <hash>` shows the exact diff (line-by-line changes) of that commit. Use this when a bug appeared and you want to see exactly what changed.

```
a3f9c21 Add payment processing module
b7e2d14 Initial project setup
c9a3f88 Add package.json dependencies
```

#### 1.6 The .gitignore File

```
# .gitignore — tell Git what NOT to track
node_modules/
.env
dist/
*.log
.DS_Store
```

**📖 .gitignore — What to Exclude**

Create a file named `.gitignore` in your project root. Every file or folder listed here will be completely ignored by Git — they won't show up in `git status` and can never be accidentally committed.  
  
**node_modules/** — thousands of dependency files; anyone can regenerate them with `npm install`.  
**.env** — contains API keys and passwords. Never commit this.  
***.log** — log files change constantly and have no value in history.

> **💡 Fresher Tip — The .env File Must Always Be in .gitignore**

> NovaPay's `.env` file contains `RAZORPAY_SECRET_KEY=rzp_live_abc123`. If you commit this file and push it to GitHub, even for one minute, bots scan GitHub 24/7 and will find and use that key before you can delete it. Hundreds of companies have had production secrets stolen this way. **The very first thing you add to .gitignore is .env.** Before you make your first commit. Non-negotiable.

> **Phase 1 Key Takeaways**

> - Git has three areas: working directory (edits), staging area (chosen changes), repository (permanent commits)
> - git status — run this constantly. It tells you exactly where every file is
> - git add → git commit is the two-step save process. Never skip the message
> - git log --oneline shows your project's complete history in a readable format
> - Always create .gitignore before your first commit — especially to exclude .env and node_modules

**Quiz: ❓ Quick Check — Which area holds changes that are ready to be committed but not yet saved permanently?**

- A) The working directory
- B) The staging area (index)
- C) The repository
- D) The .git folder

> **Answer/explanation:** ✅ **Answer: B.** The staging area is the middle step between editing a file and permanently saving it. git add moves files into staging. git commit saves everything in staging as a permanent commit. The working directory is where you type. The repository is the permanent history.

### 2. Phase 2 — Branching

**Business Problem:** NovaPay is running a live payment system. Kiran needs to add a new EMI feature — which will take two weeks and involve breaking changes to the payment module. Arjun needs to fix a critical bug in production today. If both of them work on the same codebase at the same time, Kiran's half-built EMI code will break Arjun's urgent fix. Branches solve this: every developer works in isolation, and code only merges when it's finished and reviewed.

**Scene 2 — NovaPay Standup | The Simultaneous Crisis**

> **Kiran** _Backend Developer — NovaPay_
> 
> I need to refactor the entire payments module to support EMIs. I'll be touching payment.js, utils.js, and the database schema. It'll take at least a week and nothing will work during that time.

> **Arjun** _Backend Developer — NovaPay_
> 
> I found a UPI callback validation bug that's failing 3% of all transactions right now. I need to fix payment.js in the next hour and deploy it. But if Kiran starts refactoring, the file will be broken...

> **Tanvi** _Senior Platform Engineer_
> 
> Priya, this is exactly what branches are for. Arjun, create a branch called hotfix/upi-callback, fix the bug there, and merge it to main. Kiran, create a branch called feature/emi-support and work there for the whole week. They'll never touch each other. Main stays clean and deployable at all times.

#### 2.1 What Is a Branch?

```
  main branch:  A ── B ── C
                               \
  feature branch:               D ── E ── F
                                          (you are here)

  A, B, C = existing commits on main (production-safe)
  D, E, F = your new commits (work in progress, safe to break things)
  When feature is done, merge F back into main.
```

> **💡 What Exactly Is a Branch?**

> A branch is just a lightweight pointer to a commit. When you create a branch, Git creates a new label that points to the current commit. As you make new commits on that branch, the label moves forward. The original branch (main) stays where it was. No files are duplicated — Git is smart enough to share the unchanged history. Creating a branch costs almost nothing — it's instantaneous and takes up barely any storage.

#### 2.2 Create and Switch Branches

```bash
# Create a new branch AND switch to it
git switch -c feature/emi-support
# Switch back to main
git switch main
# See all branches (* = current branch)
git branch
```

**📖 Creating Branches**

`git switch -c feature/emi-support` creates the branch AND switches to it in one command. The `-c` flag means "create."  
  
The **branch name** is a convention: `feature/` prefix for new features, `hotfix/` for urgent production fixes, `release/` for release preparation.  
  
`git branch` lists all local branches. The branch with `*` in front is your current branch — every commit you make goes here.

```
* feature/emi-support
  hotfix/upi-callback
  main
```

#### 2.3 HEAD — Where You Are Right Now

```
  main:     A ── B ── C
                               \
  feature:                      D ── E
                                     ▲
                                   HEAD  ← you are at commit E on feature branch

  HEAD is just a pointer to "the commit you are currently on."
  When you switch branches, HEAD moves.
  When you make a new commit, HEAD moves forward on the current branch.
```

```bash
# See where HEAD is pointing
git log --oneline --decorate
```

**📖 Understanding HEAD**

HEAD is Git's way of saying "where are you right now?" It always points to the current branch, which points to the latest commit on that branch.  
  
`--decorate` adds branch names and HEAD to the log output so you can see where each branch label is sitting relative to the commits.  
  
If you see `HEAD -> feature/emi-support` in the log, that confirms you're on the feature branch and your next commit will go there.

#### 2.4 Merging — Bring Changes Back to Main

When a feature is finished, you merge its branch back into main. The commits from the feature branch join the main branch's history.

```bash
# 1. Switch to the branch you want to merge INTO
git switch main
# 2. Merge the feature branch into main
git merge feature/emi-support
```

**📖 git merge — Joining Branches**

The golden rule: **you merge into the branch you're currently on.** Switch to main first, then merge the feature branch into it.  
  
If there are no conflicts, Git creates a "merge commit" that ties the two branch histories together. You'll be prompted to write a merge commit message.  
  
After merging, the feature branch still exists. Delete it when you're done: `git branch -d feature/emi-support`.

```
Updating c9a3f88..e7f1b44
Fast-forward
 payment.js | 87 ++++++++++++++++++++++++++++++++++++++++
 utils.js   | 12 ++++--
 2 files changed, 95 insertions(+), 4 deletions(-)
```

#### 2.5 Fast-Forward vs Merge Commit

**Two Types of Merge**

- **⚡ Fast-Forward** — Happens when main hasn't changed since you branched off. Git just moves the main pointer forward to the latest feature commit. No merge commit is created. Clean linear history.

- **🔀 Merge Commit (3-Way)** — Happens when both main and the feature branch have new commits since branching. Git creates a new "merge commit" with two parents. History shows both lines of work meeting.

> **Phase 2 Key Takeaways**

> - Branches are free — create one for every feature, bug fix, or experiment. They cost nothing
> - git switch -c <name> creates and switches to a new branch in one command
> - HEAD points to where you are right now — your next commit goes on the current branch
> - Always switch to the destination branch BEFORE merging (git switch main, then git merge feature)
> - Delete merged branches to keep the repository tidy: git branch -d <name>

### 3. Phase 3 — Remote Repositories & GitHub

**Business Problem:** NovaPay's Git repository currently exists only on Priya's laptop. If the laptop dies, all code history is gone. And the three developers can't share their branches with each other. GitHub (or GitLab/Bitbucket) is a cloud host for your Git repository — a central place everyone can push to and pull from.

**Scene 3 — NovaPay IT Room | "It's All on One Laptop"**

> **Roshan** _DevOps Architect_
> 
> Priya, the Git repo you set up is great, but it only exists on your machine. Kiran is working on the EMI branch on his laptop, Arjun on his own machine — they can't share their branches with each other. And if your machine has a problem, we lose the central history. We need this on GitHub today.

> **Tanvi** _Senior Platform Engineer_
> 
> Once you push to GitHub, every developer clones the repository to their machine. They pull the latest changes every morning, push their branches when they're ready for review, and open Pull Requests so the team can review code before it merges to main. That's the standard industry workflow — feature branch → push → pull request → review → merge.

#### 3.1 Connect Your Local Repo to GitHub

```bash
# Add GitHub as the "origin" remote
git remote add origin https://github.com/novapay/payment-api.git
# Verify the remote was added
git remote -v
```

**📖 What Is a Remote?**

A **remote** is a URL pointing to a copy of the repository somewhere else (GitHub, GitLab, a company server). **origin** is just the conventional name for the main remote — you could call it anything, but everyone calls it origin.  
  
`git remote -v` shows all configured remotes with their fetch and push URLs — useful for confirming you're connected to the right place.

```
origin  https://github.com/novapay/payment-api.git (fetch)
origin  https://github.com/novapay/payment-api.git (push)
```

#### 3.2 Push — Send Your Work to GitHub

```bash
# First push — set upstream tracking
git push -u origin main
# After -u is set, all future pushes are just:
git push
```

**📖 git push — Upload to GitHub**

`git push -u origin main` pushes your local main branch to GitHub and sets up "tracking" — your local main now knows it corresponds to `origin/main` on GitHub.  
  
The **-u flag** (short for --set-upstream) only needs to be used once per branch. After that, plain `git push` knows where to send the commits.  
  
Push often — at least at the end of every working day. It serves as a backup and lets teammates see your progress.

#### 3.3 Clone — Get a Repository from GitHub

```bash
# Kiran downloads the repo to his laptop
git clone https://github.com/novapay/payment-api.git
# Clone into a specific folder name
git clone https://github.com/novapay/payment-api.git novapay-backend
```

**📖 git clone — Download a Full Repository**

`git clone` downloads the entire repository — all commits, all branches, all history — to your machine. It also automatically sets `origin` to point back to GitHub.  
  
Clone is a one-time setup. After cloning, you use `git pull` to get updates, not clone again.  
  
Any new developer joining NovaPay's team runs one `git clone` command and they have the full project history from day one.

#### 3.4 Pull — Get Updates from GitHub

```bash
# Every morning — get the latest from GitHub
git pull
# Pull a specific branch
git pull origin main
```

**📖 git pull — Download + Merge**

`git pull` does two things: **fetch** (download new commits from GitHub) and **merge** (bring them into your current local branch).  
  
The golden rule: **always pull before you start working each morning.** If Arjun merged a fix to main last night and you don't pull, you're building on outdated code. Then when you push, Git will reject it because your history has diverged from GitHub's.

```
  PUSH / PULL FLOW

  Your Machine                           GitHub (origin)
  ────────────────────────────────────────────────────────
  local main: A──B──C ──git push──► remote main: A──B──C

  Arjun pushes a new commit D to main on GitHub:
  remote main: A──B──C──D

  You run git pull:
  local main: A──B──C──D  ← now up to date
```

#### 3.5 Fetch — Download Without Merging

```bash
# Download changes but DON'T merge yet
git fetch origin
# See what's on GitHub vs your local
git log main..origin/main --oneline
```

**📖 fetch vs pull**

`git fetch` is the cautious version of pull. It downloads new commits from GitHub and stores them as `origin/main` (a remote-tracking branch) but does NOT change your local main branch.  
  
This lets you inspect what changed on GitHub before merging it in. The `git log main..origin/main` command shows commits that are on GitHub but not yet in your local main. Useful for reviewing what your teammates shipped overnight.

> **Phase 3 Key Takeaways**

> - origin is the conventional name for the GitHub remote — just a URL with a short name
> - git push -u origin main on the first push sets up tracking; plain git push for all future pushes
> - git clone downloads the full repository. Run once per machine
> - git pull = fetch + merge. Run every morning before starting work
> - git fetch downloads without merging — use it to inspect remote changes before applying them

### 4. Phase 4 — Pull Requests & Code Review

**Business Problem:** Kiran finished the EMI feature on his branch and wants to merge it to main. But main is production — a mistake there could fail real payments. No code should go to main without at least one other developer reviewing it. Pull Requests (PRs) on GitHub formalise this: Kiran shows his changes, the team reviews and comments, and only when it's approved does it merge.

**Scene 4 — NovaPay GitHub | The First Pull Request**

> **Kiran** _Backend Developer — NovaPay_
> 
> The EMI feature is done on my branch. I'm going to merge it to main now.

> **Tanvi** _Senior Platform Engineer_
> 
> Not directly — never merge to main directly. Push your branch to GitHub and open a Pull Request. Tag Arjun and me as reviewers. We'll go through every changed line. If something looks wrong, we'll comment. You fix it, push again. Once both of us approve, the PR merges. Main is protected — nobody merges to it without at least one approval, not even me.

#### 4.1 Push a Feature Branch to GitHub

```bash
# Make sure you're on your feature branch
git switch feature/emi-support
# Push the branch to GitHub
git push -u origin feature/emi-support
```

**📖 Pushing a Feature Branch**

You push a feature branch the same way you push main — just use the branch name. GitHub will create the branch on the remote side automatically if it doesn't exist.  
  
After pushing, GitHub will show you a banner: **"feature/emi-support had recent pushes — Compare & pull request."** Click that to open a PR in seconds.  
  
You can push to a feature branch as many times as you want — each push updates the PR automatically.

#### 4.2 What Is a Pull Request?

> **Pull Request = "I am requesting that you pull my branch into main"**

> A Pull Request is a GitHub UI feature (not a Git command). It shows a diff of every changed line, lets reviewers add inline comments on specific lines, tracks approvals, runs CI checks automatically, and records the full conversation history. When approved, merging takes one click. The PR stays in history forever — 6 months later you can open it and see exactly why a change was made and who approved it.

#### 4.3 Protected Branches — Prevent Accidental Merges

**Branch Protection Rules (Set on GitHub → Settings → Branches)**

- **✅ Require pull request reviews** — Set minimum 1 (or 2) approvals required before merging. Nobody — not even the repo owner — can bypass this.

- **✅ Require status checks to pass** — Block merging if CI tests fail. Broken code cannot reach main no matter how many approvals it has.

- **✅ Restrict who can push** — Prevent developers from pushing directly to main. All changes must come through a PR — no exceptions.

- **✅ Require linear history** — Enforces squash or rebase merges — keeps the main branch history clean and readable.

> **🔒 NovaPay Branch Protection Policy**

> In NovaPay's GitHub repository, the **main** branch has these rules: 1 required reviewer approval, all CI checks must pass, direct pushes to main are blocked for all users including admins, and branch must be up to date with main before merging. These rules mean: even if Kiran is having a bad day and commits broken code, the CI catches it and the PR can't merge. Main is always deployable. Always.

### 5. Phase 5 — Merge Conflicts

**Business Problem:** Kiran and Arjun both edited `payment.js` on their separate branches. When their branches try to merge, Git can't figure out which version to keep — the same lines were changed two different ways. This is a merge conflict. Git stops and asks you to decide. Every developer hits this eventually — and it's not scary once you understand it.

**Scene 5 — NovaPay Terminal | "CONFLICT: payment.js"**

> **Kiran** _Backend Developer — NovaPay_
> 
> I tried to merge my EMI branch into main and got this: CONFLICT (content): Merge conflict in payment.js. What do I do?

> **Tanvi** _Senior Platform Engineer_
> 
> Open payment.js. Git has put conflict markers in it — your version and Arjun's version are both shown, separated by those arrows. You just need to read both, decide what the final code should look like, remove the markers, and commit. That's it. The file is just a text file. Git made it ugly to show you the conflict — you make it clean again.

#### 5.1 What a Conflict Looks Like

```
# Git writes this into payment.js:

<<<<<<< HEAD
  const maxRetries = 3;
=======
  const maxRetries = 5;
>>>>>>> feature/emi-support
```

**📖 Reading Conflict Markers**

**<<<<<<< HEAD** — everything below this, down to `=======`, is what your current branch (HEAD) has.  
  
**=======** — divides the two versions.  
  
**>>>>>>> feature/emi-support** — everything above this, after `=======`, is what the incoming branch has.  
  
Your job: delete all three marker lines, keep whichever version is correct (or combine them), and save the file.

```bash
# After resolving — the fixed file looks like:

  const maxRetries = 5;

# Now stage the resolved file
git add payment.js
# Complete the merge
git commit -m "Resolve conflict: use maxRetries=5 for EMI"
```

**📖 Resolving a Conflict**

After you edit the file to remove all conflict markers, Git doesn't automatically know you're done. You have to explicitly tell it:  
  
1. `git add payment.js` — marks this file as "conflict resolved"  
2. `git commit` — completes the merge with a commit  
  
Always write a clear commit message explaining which version you chose and why.

> **💡 Fresher Tip — Use VS Code to Resolve Conflicts Visually**

> You don't have to resolve conflicts manually in a text editor. VS Code shows conflict markers as colour-coded blocks with buttons: **Accept Current Change**, **Accept Incoming Change**, **Accept Both Changes**, and **Compare Changes**. Click the right button, save, and you're done. The underlying Git commands are the same — VS Code just makes the visual clearer. GitHub's web editor also has a conflict resolver UI for simple cases.

#### 5.2 Abort a Merge if Things Go Wrong

```bash
# Panic button — undo the entire merge
git merge --abort
# Check status — should be clean
git status
```

**📖 It's Always Safe to Abort**

If you're in the middle of a conflict and feel overwhelmed, `git merge --abort` is your escape hatch. It rewinds everything back to exactly the state before you started the merge — no changes, no conflict markers, clean working directory.  
  
Nothing is lost. You can try again after talking to your teammate about which version is correct. Abort is not failure — it's the responsible thing to do when you're unsure.

### 6. Phase 6 — Rebase

**Business Problem:** Kiran's EMI feature branch was created 2 weeks ago from an old version of main. In those 2 weeks, Arjun merged 15 hotfixes to main. Kiran's branch is now behind. If Kiran tries to open a PR, GitHub says "This branch is 15 commits behind main." Merging with that many diverged commits creates a messy history. Rebase solves this — it replays Kiran's work on top of the latest main as if he'd started his branch today.

**Scene 6 — NovaPay PR Review | "Your Branch Is Behind"**

> **Kiran** _Backend Developer — NovaPay_
> 
> My PR says it's 15 commits behind main. Tanvi said I need to rebase. What does that actually do?

> **Roshan** _DevOps Architect_
> 
> Think of it like this: you started building the EMI feature on version 1.0 of a wall. Arjun then made 15 changes to the wall. Instead of gluing your work onto the old wall and then merging the two messy versions, rebase picks up your EMI commits, applies all 15 of Arjun's changes first, and then replays your commits on top of the result. The final history looks like you started the EMI feature today, on the latest code.

#### 6.1 Rebase vs Merge

```
  MERGE result:                      REBASE result:

  main:    A──B──C──────M            main:    A──B──C──D──E
                   \   /                                   \
  feature:  D──E───╯                 feature:               D'──E'

  M = merge commit. History shows                D', E' = your commits replayed
  two parallel lines meeting.                    on top of latest main.
  Looks complex. Honest.                         Clean linear history.
                                                 Looks like you never diverged.
```

```bash
# On your feature branch
git switch feature/emi-support
# Replay your commits on top of latest main
git rebase main
```

**📖 git rebase — Replay Your Commits**

`git rebase main` does the following: it finds the point where your branch diverged from main, temporarily removes your commits, fast-forwards your branch to the tip of main, then replays your commits one by one on top.  
  
Result: your branch looks like it was just created from the latest main. The PR will say "up to date with main" and will merge cleanly with a linear history.

##### ⚠️ The Golden Rule of Rebase

**Never rebase a branch that other people are also working on.** Rebase rewrites commit history — it creates new commit hashes. If a teammate has based their work on your old commits, rebase will cause serious confusion. Only rebase branches that are exclusively yours. Once a branch has been merged and multiple people have pulled it, never rebase it.

### 7. Phase 7 — Undoing Things

**Business Problem:** Mistakes happen. Arjun committed a debug `console.log` with NovaPay's internal API key. Kiran accidentally committed the wrong file. Roshan deployed a bad release. Git has a tool for every type of undo — you just need to know which one to use when.

**Scene 7 — NovaPay Slack | "I Committed the Wrong Thing"**

> **Arjun** _Backend Developer — NovaPay_
> 
> I just committed and pushed a file that had our internal database password in a debug log. It's on GitHub. What do I do?!

> **Tanvi** _Senior Platform Engineer_
> 
> Step one: rotate the database password right now — assume it's compromised. Step two: we'll fix the Git history. But rotating the credential is the real fix. Git history changes won't help if someone already cloned and saw the password. This is why .env is always in .gitignore before your first commit.

#### 7.1 Undo the Last Commit (Keep Changes)

```bash
# Undo last commit — keep files staged
git reset --soft HEAD~1
# Undo last commit — keep files unstaged
git reset --mixed HEAD~1
```

**📖 Soft vs Mixed Reset**

`HEAD~1` means "one commit before HEAD" — i.e. the previous commit.  
  
**--soft**: Removes the commit but keeps all your changes staged. You can fix the problem and recommit immediately.  
  
**--mixed** (the default): Removes the commit and unstages the changes but keeps the file edits in your working directory. More room to review before restaging.

#### 7.2 Undo a Commit That's Already Pushed

```bash
# Create a new commit that reverses the bad one
git revert a3f9c21
# Push the revert commit to GitHub
git push
```

**📖 git revert — Safe Undo for Pushed Commits**

`git revert` is the safe way to undo a commit that's already on GitHub and that others may have pulled.  
  
It does NOT delete the original commit. Instead, it creates a new commit that does the exact opposite — if the original commit added 5 lines, the revert commit removes those 5 lines. History is preserved.  
  
This is the only way to safely undo a public commit without breaking other developers' local copies.

#### 7.3 Discard Unstaged Changes

```bash
# Discard edits in one file (permanent!)
git restore payment.js
# Discard all unstaged changes
git restore .
```

**📖 git restore — Reset a File to Last Commit**

`git restore payment.js` throws away all edits to that file since the last commit — restoring it to exactly how it looked in the most recent commit.  
  
**This is permanent.** The edits are gone — not staged, not anywhere. Use it when you've made a mess of a file and want to start fresh from the last known good version.  
  
Run `git status` first to confirm which files have changes before restoring.

**Undo Cheat Sheet — Which Command to Use**

- **Undo staged file** — `git restore --staged file.js`  
Unstages the file, keeps your edits in working directory.

- **Undo working dir edits** — `git restore file.js`  
Throws away edits. Permanent — can't recover.

- **Undo last commit (not pushed)** — `git reset --soft HEAD~1`  
Removes commit, keeps changes staged.

- **Undo pushed commit** — `git revert <hash>`  
Creates a new undo commit. Safe for public history.

### 8. Phase 8 — Stash, Tags & Cherry-Pick

**Business Problem:** These three tools solve specific, real situations that come up every week on a working team. Stash: you're in the middle of a feature when Tanvi calls — urgent hotfix needed now. Tags: you need to mark the exact commit that went to production for v1.3. Cherry-pick: there's one specific bug fix commit on a branch that needs to go to main without bringing the entire unfinished branch along.

#### 8.1 Stash — Save Unfinished Work Temporarily

```bash
# Save current work-in-progress
git stash
# Switch to main, do the urgent hotfix, come back
git switch main
# ... fix the bug, commit, push ...
# Come back and restore your work
git switch feature/emi-support
git stash pop
```

**📖 git stash — The Quick Save**

`git stash` saves your uncommitted changes to a temporary stack and resets your working directory to the last commit — like pressing "save and pause."  
  
`git stash pop` restores those saved changes back to your working directory — like pressing "resume."  
  
You can have multiple stashes. `git stash list` shows all of them. `git stash pop` applies and removes the most recent one.

#### 8.2 Tags — Mark Important Releases

```bash
# Create a tag for the current commit
git tag v1.3.0
# Tag with a message (annotated tag)
git tag -a v1.3.0 -m "Release v1.3.0 - EMI support"
# Push tags to GitHub
git push origin --tags
```

**📖 git tag — Bookmark a Commit**

A tag is a permanent, human-readable name for a specific commit. Unlike branch names that move with new commits, tags never move — `v1.3.0` will always point to the exact same commit.  
  
NovaPay tags every release — `v1.0.0`, `v1.1.0`, etc. If a production issue is reported, the team can instantly check out exactly what was deployed: `git checkout v1.2.0`.  
  
GitHub turns tags into "Releases" — you can attach release notes and download ZIPs for each version.

#### 8.3 Cherry-Pick — Bring One Commit to Another Branch

```bash
# On main — apply one specific commit from another branch
git switch main
git cherry-pick d7e4a12
```

**📖 git cherry-pick — Take One Commit**

Cherry-pick applies one specific commit from any branch onto your current branch — without bringing all the other commits from that branch.  
  
Use case: Arjun found a critical bug and fixed it on his feature branch (commit `d7e4a12`). The feature isn't ready, but the bug fix is needed in production today. Cherry-pick grabs just that one commit and applies it to main. The feature work stays on Arjun's branch.

### 9. Phase 9 — Git Hooks & CI/CD Integration

**Business Problem:** Developers sometimes forget to run tests before pushing. Once bad code reaches GitHub, the CI pipeline fails and blocks the PR — but the developer has already pushed, created the PR, and now has to fix and re-push. Git hooks let you run automatic checks before a commit or push even happens, catching errors locally before they leave the developer's machine.

#### 9.1 What Are Git Hooks?

> **Git Hooks = Scripts That Run Automatically at Git Events**

> Git hooks are shell scripts stored in `.git/hooks/`. Git runs them automatically at specific points — before a commit, after a commit, before a push, after a merge. You use them to enforce standards: run tests before every commit, validate commit message format, run a linter before every push. If the script exits with a non-zero code, Git stops the operation.

```bash
# Create a pre-commit hook
nano .git/hooks/pre-commit
# Content of the hook script:
#!/bin/bash
npm test
if [ $? -ne 0 ]; then
  echo "Tests failed. Commit blocked."
  exit 1
fi
```

**📖 pre-commit Hook — Test Before Every Commit**

The `pre-commit` hook runs every time you type `git commit` — before the commit is created.  
  
This script runs `npm test`. If tests fail (`$? -ne 0` means "exit code was not zero"), the script prints an error and exits with code 1, which tells Git to abort the commit.  
  
Make the file executable: `chmod +x .git/hooks/pre-commit`. After that, no developer on this machine can commit broken code.

#### 9.2 GitHub Actions — CI on Every Push

Git hooks run locally on each developer's machine. GitHub Actions run on GitHub's servers on every push — they are the second line of defence (and the more reliable one, since hooks can be bypassed with `--no-verify`).

```
# .github/workflows/ci.yml
name: CI Pipeline
on: [push, pull_request]
jobs:
test:
runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm install
      - run: npm test
```

**📖 GitHub Actions — Automatic CI**

This YAML file in `.github/workflows/` defines a CI pipeline that runs on every push and pull request.  
  
**on: [push, pull_request]** — triggers on both events.  
  
The pipeline checks out the code, installs dependencies, and runs tests. If any step fails, GitHub marks the PR with a red ✗ and blocks merging (if branch protection requires status checks to pass).  
  
This is the industry standard — every professional team uses some form of this.

### 10. Phase 10 — Git Workflows

**Business Problem:** As NovaPay grows from 3 to 30 developers, they need a formal agreement on how branches are named, when merges happen, and what the process is for releases. Without a workflow, everyone invents their own process and the repository becomes chaos. There are three popular workflows — understanding them helps you join any team and immediately understand their process.

#### 10.1 The Three Main Git Workflows

**Workflow Comparison — Which to Use When**

- **🔹 GitHub Flow** — **Best for:** Web apps, continuous deployment.  
  
Simple: main is always deployable. Create a feature branch → push → PR → merge to main → deploy. No release branches. Used by most startups and SaaS companies.

- **🔹 Git Flow** — **Best for:** Products with scheduled releases (mobile apps, SDKs).  
  
Two long-lived branches: main (production) and develop (integration). Feature branches merge to develop. Releases branch off develop and then merge to both main and develop.

- **🔹 Trunk-Based Dev** — **Best for:** Large teams with strong CI/CD (Google, Facebook).  
  
Everyone commits to main (trunk) multiple times per day. Feature flags hide unfinished work. Branches are short-lived (hours, not days). Fastest release cycle possible.

#### 10.2 NovaPay's Workflow — GitHub Flow

```
  NOVAPAY GITHUB FLOW

  main ─────────────────────────────────────────────────────►  (always deployable)
         \                       /          \              /
          feature/emi-support ──              hotfix/upi ─
          (Kiran, 2 weeks)                   (Arjun, 2 hours)

  Steps:
  1. git switch -c feature/emi-support   ← create branch
  2. make commits (push often)           ← work on branch
  3. git push -u origin feature/emi      ← share to GitHub
  4. Open Pull Request on GitHub         ← request review
  5. Team reviews, CI passes             ← gate checks
  6. Merge PR to main                    ← one click
  7. Deploy main to production           ← automated
  8. git branch -d feature/emi-support   ← clean up
```

### 11. Phase 11 — Commit Messages: The Right Way

**Business Problem:** NovaPay's git log currently looks like this: "fix", "update", "changes", "wip", "done", "asdfgh". This is useless. When a payment bug hits production at midnight and Tanvi has to git log to find which commit broke things, she can't tell "fix" from "fix" from "fix". A well-written commit history is the most valuable debugging tool a team has.

**Scene 11 — NovaPay Production Incident | "Which Commit Broke It?"**

> **Tanvi** _Senior Platform Engineer_
> 
> We have a production incident right now — UPI payments are failing for amounts over ₹50,000. I'm looking at git log to find when this broke and I see: "fix", "update", "more changes", "final fix", "final fix 2". I cannot tell which of these 8 commits from today is the problem. I'm going to have to read every diff. This is costing us 10 minutes for every minute of downtime. Commit messages are not optional.

> **Roshan** _DevOps Architect_
> 
> Priya, enforce Conventional Commits from today. Every message must follow the format: type(scope): description. feat(payments): add UPI high-value transaction validation. fix(auth): correct JWT expiry on mobile sessions. chore(deps): upgrade axios to 1.6.2. When it's written like this, I can filter git log --grep="fix(payments)" and instantly find every payment bug fix ever committed. That's the value.

#### 11.1 Conventional Commits Format

```bash
# Format: type(scope): short description
git commit -m "feat(payments): add EMI plan selection"
git commit -m "fix(upi): handle null amount in callback"
git commit -m "chore(deps): upgrade express to 4.19.2"
git commit -m "docs(readme): add API endpoint reference"
git commit -m "refactor(auth): extract token validation"
```

**📖 Conventional Commits Types**

**feat** — a new feature added to the codebase.  
  
**fix** — a bug fix.  
  
**chore** — maintenance work that doesn't affect users (dependency updates, build config).  
  
**docs** — documentation changes only.  
  
**refactor** — code restructuring with no behaviour change.  
  
**test** — adding or updating tests.  
  
**ci** — changes to CI/CD configuration.  
  
The **scope** in parentheses is optional but recommended — it names the module or feature area.

#### 11.2 Multi-Line Commit Messages

```bash
# Open editor for a full commit message
git commit
# What you type in the editor:
fix(payments): reject UPI transactions over 2L

RBI regulation requires extra verification for UPI
transactions above Rs 2,00,000. Without this check,
high-value transactions were silently failing at the
bank's end rather than returning a clear error.

Resolves: NP-891
Tested-by: Arjun
```

**📖 When to Write a Long Commit**

For simple changes, one line is fine. For changes that affect behaviour, add a blank line after the summary and then a body explaining **why** the change was made — not what (the diff shows what).  
  
The **why** is what makes a commit message valuable. "Reject UPI transactions over 2L" is the what. "RBI regulation requires verification for high-value UPI" is the why — and that context would take hours to rediscover without the commit message.  
  
Link to tickets and PRs in the footer for full traceability.

#### 11.3 Search Through Commit History

```bash
# Find all commits that touched payment.js
git log --oneline -- payment.js
# Find commits matching a keyword
git log --grep="UPI" --oneline
# Find who last changed each line
git blame payment.js
```

**📖 Finding Specific Changes**

`git log -- payment.js` shows only commits that changed that file — useful for tracing when a bug was introduced.  
  
`git log --grep="UPI"` searches commit messages — finds every commit related to UPI. Only works well if messages are meaningful.  
  
`git blame payment.js` shows every line of the file with the commit hash, author, and date of the last change to that line. Invaluable for "who wrote this and why?" debugging sessions.

```
a3f9c21 (Kiran  2026-01-14 10:23)  const maxRetries = 5;
b7e2d14 (Arjun  2026-01-12 16:45)  const timeout = 30000;
c9a3f88 (Priya  2026-01-10 09:12)  const apiUrl = process.env.API_URL;
```

### 12. Phase 12 — Interactive Rebase: Cleaning Up Commits

**Business Problem:** Kiran worked on the EMI feature for 2 weeks and made 47 commits — including "wip", "typo", "fix typo", "actually this time fix", "done finally maybe". Before opening a PR, Tanvi wants to see clean, logical commits that tell the story of the feature clearly. Interactive rebase lets Kiran squash the 47 messy commits into 4 well-written ones without losing any code changes.

**Scene 12 — NovaPay PR Review | "47 Commits for a 200-Line Feature?"**

> **Tanvi** _Senior Platform Engineer_
> 
> Kiran, before I review this PR, can you clean up the commit history? Right now it has 47 commits. I see 12 "fix" commits and 8 "wip" commits. I want to see 4 commits: one for the database schema change, one for the API endpoint, one for the business logic, and one for the tests. Use interactive rebase to squash and rename them. The goal is a history that explains the feature, not a history that documents your afternoon.

#### 12.1 Interactive Rebase — Squash, Rename, Reorder

```bash
# Interactively edit the last 4 commits
git rebase -i HEAD~4
# Git opens an editor showing:
pick a1b2c3 Add EMI database schema
pick d4e5f6 wip
pick g7h8i9 fix
pick j0k1l2 done for real

# Change "pick" to "squash" (s) to combine:
pick a1b2c3 Add EMI database schema
squash d4e5f6 wip
squash g7h8i9 fix
squash j0k1l2 done for real
```

**📖 Interactive Rebase Commands**

`git rebase -i HEAD~4` opens an editor with the last 4 commits listed, oldest at the top.  
  
Each line starts with a command word. Change it to modify what happens to that commit:  
  
**pick** — keep as-is (default).  
**squash (s)** — combine with the commit above it.  
**reword (r)** — keep the commit but change the message.  
**drop (d)** — delete this commit entirely.  
  
Save and close — Git will apply your instructions.

```
# After squashing, Git opens the message editor.
# Delete all the "wip" and "fix" lines.
# Write one clean message for the combined commit:

feat(payments): add EMI plan selection and database schema

- New emi_plans table with plan_id, months, interest_rate
- POST /payments/emi endpoint returns available plans
- Validates EMI eligibility based on transaction amount
Resolves: NP-312
```

**📖 Writing the Combined Message**

When squashing, Git shows you all the original commit messages together — delete the ones you don't want and write a clean summary for the combined commit.  
  
The result: all the code changes from all 4 commits are preserved in a single commit with a single, meaningful message. The PR reviewer sees one commit that tells the full story — not 47 messages documenting the developer's stream of consciousness.

##### ⚠️ Only Rebase Commits That Aren't Shared Yet

Interactive rebase rewrites commit history — the commits get new hashes. This is safe on your own feature branch that only you are using. It is NOT safe on shared branches or after a PR has been opened and others have reviewed specific commits. If you rebase after others have already reviewed, their review comments will no longer point to existing commits. Rule: squash and clean up before opening the PR, not after.

### 13. Phase 13 — Git Bisect: Find the Bug Automatically

**Business Problem:** NovaPay's payment API was working perfectly in release v1.2.0. It's broken in the current main. There are 87 commits between v1.2.0 and now. Reading 87 diffs to find the bug commit would take hours. Git bisect performs a binary search through the commits — you just tell it "this commit is good" and "this commit is bad," and it will find the exact breaking commit in just 7 steps (log₂(87) ≈ 7).

#### 13.1 Using Git Bisect

```bash
# Start bisect session
git bisect start
# Tell Git the current commit is BAD
git bisect bad
# Tell Git where it was GOOD (v1.2.0 tag)
git bisect good v1.2.0
```

**📖 Starting the Search**

Git bisect uses binary search. With 87 commits between good and bad, it will check the commit at position 43 first. You test whether the bug exists there and tell Git "good" or "bad." It then checks position 21 or 65 depending on your answer. Each round cuts the search space in half.  
  
After about 7 rounds, Git will pinpoint the exact commit that introduced the bug — showing you the commit hash, author, message, and diff.

```bash
# Git checks out a middle commit automatically.
# Test your application. Bug present?
# If bug IS present at this commit:
git bisect bad
# If bug is NOT present at this commit:
git bisect good
# After 7 rounds, Git prints:
# d4e5f6 is the first bad commit
# End the bisect session
git bisect reset
```

**📖 Running the Search**

Each time you answer "good" or "bad," Git checks out the next commit in the binary search automatically. You just test your application and answer.  
  
When Git identifies the first bad commit, it prints the full commit details — you can now read the diff (`git show <hash>`) to understand exactly what change caused the bug.  
  
`git bisect reset` returns you to your original branch after the search is complete.

### 14. Phase 14 — Managing Large Repositories

As NovaPay's codebase grows, certain repository management habits become important for keeping the team's workflow fast and the repository clean.

#### 14.1 git diff — See Changes Before Staging

```bash
# See unstaged changes in all files
git diff
# See staged changes (what's about to be committed)
git diff --staged
# See changes between two branches
git diff main..feature/emi-support
```

**📖 git diff — Review Before You Commit**

`git diff` shows every line changed in your working directory since the last commit — lines removed in red with `-`, lines added in green with `+`.  
  
`git diff --staged` shows the same but for files that are already staged. Always run this before committing to confirm you're committing exactly what you think you are.  
  
`git diff main..feature/emi-support` shows all changes between two branches — essentially a preview of what a PR would show.

#### 14.2 git shortlog — See Team Contributions

```bash
# Summary of commits per author
git shortlog -sn
# Contributions in last 30 days
git shortlog -sn --since="30 days ago"
```

**📖 Viewing Team Activity**

`git shortlog -sn` prints each author's name and number of commits, sorted by commit count. Useful for understanding contribution patterns — though commit count is a very rough metric (a 1-commit refactor can be more valuable than 50 typo fixes).  
  
The `--since` flag filters to a time window. `--until` sets an end date. These are useful for sprint retrospectives or release summaries.

```
   142  Kiran Reddy
    87  Arjun Singh
    63  Priya Sharma
```

#### 14.3 Cleaning Up Old Local Branches

```bash
# Delete a merged local branch
git branch -d feature/emi-support
# Force-delete an unmerged branch
git branch -D experiment/test-idea
# Delete a remote branch on GitHub
git push origin --delete feature/emi-support
# Remove stale remote-tracking refs
git fetch --prune
```

**📖 Branch Hygiene**

`git branch -d` deletes a local branch that's already been merged. Git refuses if the branch has unmerged commits — a safety net.  
  
`-D` force-deletes regardless. Use for experimental branches you definitely don't need.  
  
`git push origin --delete <branch>` deletes the branch on GitHub too. After a PR merges, the GitHub UI will offer an "Delete branch" button — click it every time.  
  
`git fetch --prune` removes local references to remote branches that no longer exist on GitHub.

#### 14.4 git reflog — The Ultimate Safety Net

```bash
# Show every HEAD movement in the last 90 days
git reflog
# Recover a "lost" commit
git checkout a3f9c21
git switch -c recovered-work
```

**📖 reflog — Nothing Is Ever Truly Lost**

The reflog is Git's safety net for yourself. It records every move HEAD has made — every checkout, rebase, reset, merge — for the last 90 days.  
  
Even if you run `git reset --hard` and lose commits, those commits still exist in Git's object database. The reflog shows their hashes. You can check out the old commit and create a new branch from it — recovering your "lost" work.  
  
This is why Git is safe to experiment in. Almost nothing is permanently unrecoverable unless you wait 90 days and Git's garbage collector runs.

```
a3f9c21 HEAD@{0}: commit: fix(payments): handle zero-amount UPI
b7e2d14 HEAD@{1}: rebase: feat(emi): add plan selection endpoint
c9a3f88 HEAD@{2}: checkout: moving from main to feature/emi-support
d4e5f6a HEAD@{3}: merge: Merge pull request #47 from hotfix/upi-fix
```

### 15. Phase 15 — SSH Keys & Secure GitHub Access

**Business Problem:** Every developer on the NovaPay team is typing their GitHub username and password (or a personal access token) every time they push. This is slow, error-prone, and insecure. SSH keys solve this: your computer has a private key, GitHub has your public key, and they authenticate each other automatically — no password required, more secure than passwords, and faster every single time.

#### 15.1 Generate an SSH Key

```
# Generate an ed25519 SSH key pair
ssh-keygen -t ed25519 -C "priya@novapay.in"
# Press Enter to accept default file path
# Set a passphrase (optional but recommended)
# Print your public key (this goes to GitHub)
cat ~/.ssh/id_ed25519.pub
```

**📖 SSH Key Pairs**

SSH generates two files: `~/.ssh/id_ed25519` (your **private key** — never share this) and `~/.ssh/id_ed25519.pub` (your **public key** — safe to share).  
  
The email in `-C` is just a label so you know which key is which when you have multiple machines.  
  
Copy the output of `cat ~/.ssh/id_ed25519.pub` — the entire line starting with `ssh-ed25519` — and paste it into GitHub Settings → SSH and GPG keys → New SSH key.

```bash
# Test your SSH connection to GitHub
ssh -T git@github.com
# Update remote URL to use SSH instead of HTTPS
git remote set-url origin git@github.com:novapay/payment-api.git
```

**📖 Switching to SSH**

`ssh -T git@github.com` verifies the connection — you should see "Hi priya! You've successfully authenticated." If not, check that you added the public key to GitHub correctly.  
  
`git remote set-url` updates the remote URL from HTTPS (`https://github.com/...`) to SSH (`git@github.com:...`). After this, all push and pull operations use SSH — no more password prompts.

```
Hi priya! You've successfully authenticated, but GitHub does not provide shell access.
```

#### 15.2 Personal Access Tokens (When SSH Isn't Available)

> **When to Use Personal Access Tokens vs SSH**

> **SSH keys** are the best choice for your personal development machine — set up once, no passwords ever again.  
  
**Personal Access Tokens (PATs)** are used in: CI/CD pipelines (GitHub Actions, Jenkins), scripts that need to interact with the GitHub API, environments where you can't install SSH keys (some cloud IDEs). Generate a PAT at GitHub Settings → Developer settings → Personal access tokens. Treat it like a password — never commit it to a repository, always store it in environment variables or a secrets manager.

### 16. Phase 16 — Forking & Open Source Contributions

**Real-world context:** NovaPay uses several open-source libraries. Arjun found a bug in the `razorpay-node` SDK that causes incorrect decimal handling for amounts in paise. He wants to fix it and contribute the fix back. But he's not a member of the razorpay org on GitHub — he can't push to their repository. Forking is the answer.

#### 16.1 The Fork and PR Workflow

```
  FORK AND CONTRIBUTE WORKFLOW

  razorpay/razorpay-node  (the original — you can't push here)
         |
         | Click "Fork" on GitHub
         ▼
  arjun-singh/razorpay-node  (your copy — you OWN this, push freely)
         |
         | git clone your fork
         | create a branch: git switch -c fix/paise-decimal
         | make the fix, commit, push
         | open a Pull Request FROM your fork TO the original
         ▼
  razorpay/razorpay-node  ← maintainers review and merge your PR
```

```bash
# 1. Fork on GitHub (click the Fork button)
# 2. Clone YOUR fork
git clone git@github.com:arjun-singh/razorpay-node.git
# 3. Add the original as "upstream"
git remote add upstream git@github.com:razorpay/razorpay-node.git
# 4. Sync with the original before branching
git fetch upstream
git merge upstream/main
```

**📖 Fork Workflow**

A fork is your personal copy of someone else's repository on GitHub. You can do anything to it — it doesn't affect the original.  
  
**origin** = your fork (you push here).  
**upstream** = the original repository (you pull updates from here).  
  
Always sync with upstream before creating your branch — otherwise your fix will be based on old code and the PR may have conflicts. After syncing, create your branch, fix the bug, push to origin, and open a PR from origin → upstream.

#### 16.2 Keeping Your Fork Updated

```bash
# Stay in sync with the original project
git fetch upstream
git switch main
git merge upstream/main
git push origin main
```

**📖 Why Keep Your Fork Updated?**

The original project keeps evolving after you fork it. If you create a bug fix branch 3 weeks after forking without syncing, your PR will be based on outdated code — it may conflict with recent changes in the original.  
  
This 4-line sequence: fetch new commits from the original, switch to your main, merge them in, push to your fork. Run this every time before creating a new branch for a contribution.

### 17. Phase 17 — Monorepo vs Multi-repo

As NovaPay grows, Tanvi needs to decide: should all services (payment API, user service, notification service, frontend) live in one Git repository (monorepo) or in separate repositories (polyrepo)? Each has real tradeoffs.

**Monorepo vs Polyrepo — Real Trade-offs**

- **📦 Monorepo** — **All services in one repo.**  
  
✅ One PR can update 3 services atomically.  
✅ Shared code (utils, types) without npm publish.  
✅ Single CI run catches cross-service breaks.  
❌ Repository gets large over time.  
❌ CI must be smart enough to only rebuild changed services.  
  
**Tools:** Nx, Turborepo, Lerna.

- **🗂️ Polyrepo** — **One repo per service.**  
  
✅ Teams fully independent — no conflicts.  
✅ Each service has its own release cycle.  
✅ Simpler CI per repo.  
❌ Cross-service changes need multiple PRs.  
❌ Shared code requires publishing packages.  
❌ Hard to enforce standards across repos.  
  
**Used by:** most larger orgs, microservice teams.

> **💡 What Should I Use as a Fresher?**

> When you join a company, the decision is already made — you work with whatever structure the team has. Understanding the trade-offs helps you contribute to architectural discussions later. For personal projects or small startups, start with a monorepo — it's simpler to manage when the team is small. Split into polyrepo when team independence and separate release cycles become more valuable than the coordination benefits of a single repo.

### 18. Phase 18 — Common Mistakes and How to Fix Them

Every developer makes these mistakes. The goal isn't to never make them — it's to know the fix immediately so you recover in under 2 minutes instead of 2 hours.

#### 18.1 Committed to the Wrong Branch

```bash
# You committed to main by mistake
# Move the commit to the correct branch
# 1. Note the commit hash
git log --oneline
# 2. Create/switch to the correct branch
git switch -c feature/my-feature
# 3. Remove the commit from main
git switch main
git reset --hard HEAD~1
```

**📖 Fix: Wrong Branch Commit**

When you create the new branch from main (step 2), it includes the accidental commit — because the branch starts at the current HEAD of main, which includes that commit.  
  
Then you remove it from main with `git reset --hard HEAD~1`. The commit now exists only on your feature branch — exactly where it should be.  
  
Only safe if you haven't pushed the accidental commit to GitHub yet. If you have, use `git revert` instead of `git reset`.

#### 18.2 Pushed to the Wrong Remote Branch

```bash
# Delete the wrongly created remote branch
git push origin --delete wrong-branch-name
# Push to the correct branch
git push origin correct-branch-name
```

**📖 Fix: Pushed to Wrong Remote Branch**

If you pushed to the wrong branch on GitHub (e.g., pushed to `feature/old` instead of `feature/new`), just delete it remotely and push again to the right name.  
  
If nobody has based work on the wrong branch and no PR is open, this is a safe 2-command fix. The commits aren't lost — they still exist on your local branch and you're just pushing them to the correct remote branch name.

#### 18.3 Forgot to Pull Before Working — Now History Diverged

```bash
# Your push is rejected because of diverged history:
# ! [rejected] main → main (non-fast-forward)
# Option 1: Pull and merge (creates merge commit)
git pull
# Option 2: Pull and rebase (cleaner history)
git pull --rebase
```

**📖 Fix: Diverged History**

When GitHub rejects your push with "non-fast-forward," it means GitHub has commits your local doesn't have — someone else pushed while you were working.  
  
`git pull` merges their commits with yours (creates a merge commit — ugly but safe).  
  
`git pull --rebase` is cleaner: it fetches the new commits, replays your commits on top of them, giving a linear history. Use `--rebase` when working on a shared branch for cleaner history.

#### 18.4 Added node_modules or .env to Git

```bash
# Remove from Git tracking WITHOUT deleting the file
git rm --cached .env
git rm -r --cached node_modules/
# Add to .gitignore
echo ".env" >> .gitignore
echo "node_modules/" >> .gitignore
# Commit the removal
git commit -m "chore: remove .env and node_modules from tracking"
```

**📖 Fix: Untrack Already-Committed Files**

`git rm --cached` removes a file from Git's tracking (it will no longer appear in commits) WITHOUT deleting the actual file from your disk — it still exists locally.  
  
After running this and committing, add the file to `.gitignore` so it's never accidentally tracked again.  
  
Important: if `.env` was already pushed to GitHub, the secret is compromised — rotate the credentials immediately. Removing from Git doesn't remove it from GitHub's history.

### Quick Command Reference — All the Commands You Need

Command

What It Does

git init

Initialise a new Git repository in the current folder

git config --global user.name "Name"

Set your name — appears in every commit you make

git status

Show which files are changed, staged, or untracked

git add <file>

Stage a specific file for the next commit

git add .

Stage ALL changed files at once

git commit -m "message"

Save staged changes as a permanent commit with a message

git log --oneline

Show all commits — one per line — newest first

git diff

Show line-by-line changes not yet staged

git switch -c <branch>

Create a new branch and switch to it

git switch <branch>

Switch to an existing branch

git branch

List all local branches (* = current)

git branch -d <branch>

Delete a merged branch

git merge <branch>

Merge a branch into your current branch

git merge --abort

Cancel a merge in progress

git remote add origin <url>

Connect your local repo to a GitHub remote

git push -u origin <branch>

Push branch to GitHub and set upstream tracking

git push

Push current branch to GitHub (after -u is set)

git pull

Fetch and merge latest changes from GitHub

git fetch origin

Download remote changes without merging

git clone <url>

Download a full repository to your machine

git rebase main

Replay your commits on top of the latest main

git revert <hash>

Undo a pushed commit by creating a new reverse commit

git reset --soft HEAD~1

Undo last commit, keep changes staged

git restore <file>

Discard working-directory edits to a file

git restore --staged <file>

Unstage a file without losing edits

git stash

Save unfinished changes temporarily

git stash pop

Restore the most recently stashed changes

git tag -a v1.0.0 -m "msg"

Create an annotated tag for a release

git cherry-pick <hash>

Apply one specific commit to the current branch

git show <hash>

See the diff of a specific commit

git blame <file>

See who last changed each line of a file

##### ❓ Freshers' Most Asked Git Questions

**Q: Q: What's the difference between git fetch and git pull?**

A: git fetch downloads new commits from GitHub and stores them in origin/main — but does NOT change your local main branch. You can inspect what changed before applying. git pull = git fetch + git merge in one command. It downloads AND immediately merges the changes into your current branch. For daily use, git pull is fine. Use git fetch when you want to review what changed before merging.

**Q: Q: I committed to main by mistake instead of my feature branch. What do I do?**

A: If you haven't pushed yet: run git reset --soft HEAD~1 to undo the commit but keep your changes staged. Then git switch -c feature/my-feature and git commit to commit on the right branch. If you have already pushed to main (and if main isn't protected), run git revert HEAD to create a revert commit, push it, then redo the commit on the correct branch. This is why protected branches matter — they make it impossible to push directly to main.

**Q: Q: Can I rename a branch?**

A: Yes. To rename your current branch: git branch -m new-name. To rename a different branch: git branch -m old-name new-name. If the branch is on GitHub, you need to delete the old remote branch and push the new name: git push origin --delete old-name, then git push -u origin new-name.

**Q: Q: What does "detached HEAD" mean?**

A: Detached HEAD means your HEAD is pointing directly at a commit rather than at a branch. This happens when you run git checkout <hash> to look at an old commit. Any commits you make in this state are "floating" — they don't belong to any branch and can be lost. If you want to do work from this state, immediately create a branch: git switch -c recovery-branch. Then your work is safe on a named branch.

**Q: Q: Should I use git merge or git rebase?**

A: For integrating a feature branch into main — use merge (via a Pull Request). It preserves the complete history of when the feature was developed. For bringing main's latest changes into your own feature branch — use rebase. It keeps your branch up to date with a clean, linear history before you open a PR. The rule: rebase to update your branch, merge to complete the PR. Never rebase a branch that other people are using.

**Q: Q: How do I see which branch a commit is on?**

A: Run git branch --contains <commit-hash>. This lists all branches that contain that commit. Useful when a commit shows up in your log and you're not sure which branch it came from originally.

##### Git Best Practices — What Every Senior Developer Does

- Write meaningful commit messages — not "fix" or "update", but "Fix null pointer in UPI callback when amount is zero". Future you (and your team) will be grateful at 2am during an incident.
- Commit small and often — one logical change per commit. It makes reverting surgical: you can revert the exact bug fix without reverting three other things you changed on the same day.
- Always put .env, node_modules, *.log, and dist/ in .gitignore before your first commit. Once secrets are in Git history, they're effectively public even if you delete them — bots cache GitHub continuously.
- Pull before you push, every single time. Run git pull at the start of every working session to avoid diverged history and rejected pushes.
- Never force-push to main or any shared branch. git push --force rewrites history on GitHub, potentially destroying teammates' work and making the PR history unreadable.
- Name branches clearly with a prefix: feature/, hotfix/, release/, chore/. This makes the branch list instantly readable and lets CI/CD pipelines apply different rules based on prefix.
- Delete branches after merging. A repository with 200 stale branches is impossible to navigate. Enable "auto-delete head branches after PR merge" in GitHub repository settings.
- Tag every production release. git tag -a v1.3.0 -m "Payment v1.3.0 with EMI" and push tags with git push origin --tags. This lets you instantly check out any past release to investigate a production issue.

##### Git Standards — NovaPay Engineering Rules

- Main branch is protected — no direct pushes, minimum 1 approval required on all PRs, all CI checks must pass before merging. Even the engineering lead follows this rule.
- Feature branches must be named feature/<ticket-id>-short-description (e.g. feature/NP-142-emi-support). The ticket ID links the branch to the Jira story so anyone can trace code changes back to business requirements.
- Commit messages follow Conventional Commits format: feat: add EMI payment support, fix: correct null check in UPI callback, chore: update Node.js dependencies. This enables automated changelog generation.
- PR descriptions must include: what changed, why it changed, how to test it, and a link to the Jira ticket. A PR with no description gets sent back — reviewers can't approve what they can't understand.
- All PRs must be reviewed and approved within 24 hours of opening. Unreviewed PRs that sit for more than 48 hours are escalated to the engineering lead.
- Any PR touching payment.js, auth.js, or database migrations requires 2 approvals, not 1. High-risk files get extra scrutiny.

##### 🏋️ Hands-On Exercises — Build NovaPay's Git Workflow

1. **Full workflow end-to-end:** Initialise a new repository, add a .gitignore (exclude node_modules, .env), create an index.js with a simple function, stage and commit it with a meaningful message, push to a new GitHub repository. Confirm the commit appears on GitHub.
2. **Branch and merge:** Create a branch called feature/logging, add a logger.js file, commit it, switch back to main, create a hotfix/typo-fix branch, fix a typo in index.js, merge hotfix to main first (simulating an urgent fix), then merge feature/logging to main. Resolve any conflicts that arise. Delete both feature branches after merging.
3. **Simulate a conflict:** Create two branches from the same commit — branch-a and branch-b. Edit line 1 of the same file differently on each branch. Merge branch-a to main, then try to merge branch-b. Resolve the conflict in VS Code (or manually), complete the merge with a descriptive commit message.
4. **Practice rebase:** Create main with 3 commits. Create a feature branch. Add 2 more commits to main. On the feature branch, run git rebase main. Observe how the feature branch commits are replayed on top of the latest main. Push the rebased branch with git push --force-with-lease (the only safe time to force-push — your own unreviewed feature branch).
5. **Set up a pre-commit hook:** Write a pre-commit hook in .git/hooks/pre-commit that checks whether any staged .js file contains the string "console.log" and blocks the commit with a message "Remove console.log before committing." Test it by staging a file with a console.log. Fix the file and confirm the commit goes through.

### 19. Phase 19 — Git Aliases: Work Faster

Senior developers configure shortcuts — aliases — for the Git commands they type dozens of times a day. Learning to set these up makes your terminal workflow significantly faster and is a sign of developer maturity.

#### 19.1 Setting Up Useful Aliases

```bash
# Short aliases for common commands
git config --global alias.st status
git config --global alias.co switch
git config --global alias.br branch
git config --global alias.cm "commit -m"
# A beautiful one-line log with branch graph
git config --global alias.lg "log --oneline --decorate --graph --all"
```

**📖 Git Aliases — Fewer Keystrokes**

Once set up, `git st` runs `git status`, `git co main` runs `git switch main`, and `git lg` shows a beautiful ASCII graph of your entire branch history.  
  
Aliases are stored in `~/.gitconfig` under the `[alias]` section. You can edit that file directly instead of running config commands.  
  
Many developers share their `~/.gitconfig` in their dotfiles repository on GitHub so they can set up a new machine in seconds.

```
* a3f9c21 (HEAD -> feature/emi-support) feat: add EMI plan selection
* b7e2d14 (origin/main, main) fix: handle null amount in UPI callback
* c9a3f88 fix: correct Razorpay webhook signature validation
| * d4e5f6a (hotfix/upi-fix) hotfix: reject UPI amounts over 2L
|/
* e7f1b44 feat: add payment retry logic
```

#### 19.2 The .gitconfig File

```
# Example ~/.gitconfig
[user]
    name = Priya Sharma
    email = priya@novapay.in
[core]
    editor = code --wait
[alias]
    st = status
    lg = log --oneline --decorate --graph --all
    cm = commit -m
[pull]
    rebase = false
```

**📖 .gitconfig — Your Git Profile**

All global Git settings live in `~/.gitconfig`. Edit it directly with any text editor.  
  
**core.editor = code --wait** — opens VS Code for commit messages and interactive rebase instead of vim.  
  
**pull.rebase = false** — makes `git pull` merge instead of rebase by default. Set to `true` for cleaner linear history on shared branches.

### 20. Phase 20 — Git for DevOps: CI/CD Pipelines

**Business Problem:** NovaPay currently deploys by SSH-ing into the production server manually. This means deployments happen at random times, anyone can deploy anything, and there is no audit trail. Git-triggered CI/CD pipelines automate this: every merge to main automatically triggers a tested, audited deployment.

**Scene 20 — NovaPay Production | "Who Deployed That at 11 PM?"**

> **Roshan** _DevOps Architect_
> 
> Production went down at 11 PM Friday. Arjun had SSH'd into prod at 10:45 PM and manually deployed a half-tested feature. No tests, no review, no approval. The entire payment API was down for 35 minutes. From today: no manual SSH deployments. All deployments come from CI/CD triggered by Git. If code passes tests and gets PR approval, it deploys automatically. If not, it doesn't. No exceptions.

#### 20.1 GitHub Actions — Test and Deploy Pipeline

```
# .github/workflows/deploy.yml
name: Test and Deploy
on:
push:
branches: [main]
jobs:
test:
runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm install
      - run: npm test
```

**📖 Trigger on Push to Main**

`on: push: branches: [main]` — this pipeline runs every time a commit reaches main (either by direct push or PR merge).  
  
The test job runs first. If tests fail, the workflow stops — the deploy job never runs. Broken code literally cannot reach production.  
  
This is the first gate in a two-gate system: test in CI, deploy only if tests pass.

```
# Second job — deploys ONLY after tests pass
deploy:
needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to production
        run: |
          ssh user@prod.novapay.in \
            "cd /app && git pull && npm install && pm2 restart api"
```

**📖 The Deploy Job**

**needs: test** — this job only runs if the test job finished successfully. If tests failed, deploy is skipped entirely.  
  
The deploy step SSHs into the production server and runs commands: pull latest code, install dependencies, restart the app process.  
  
The SSH private key is stored as a GitHub Secret — never in the YAML file itself. GitHub injects it at runtime from an encrypted store.

**Branch → Environment Mapping — NovaPay Strategy**

- **feature/* branches** — No auto-deploy. CI runs tests only — results shown in the PR. Developers test locally against dev environment.

- **develop branch** — Auto-deploys to **staging** on every merge. QA team tests here. Real URLs but not customer-facing.

- **main branch** — Auto-deploys to **production** after tests pass. Requires PR approval + all CI checks green. Tagged as release.

- **hotfix/* branches** — Merged directly to main with fast-track 1-approval review. Immediate production deployment via the same pipeline.

### 21. Phase 21 — Common Mistakes and How to Fix Them

Every developer makes these mistakes. The goal isn't to never make them — it's to know the fix immediately so you recover in under 2 minutes instead of 2 hours.

#### 21.1 Committed to the Wrong Branch

```bash
# You committed to main by mistake
# Note the commit hash first
git log --oneline
# Create the correct branch (includes the commit)
git switch -c feature/my-feature
# Remove the commit from main
git switch main
git reset --hard HEAD~1
```

**📖 Fix: Wrong Branch Commit**

When you create the new branch from main (step 2), it includes the accidental commit — the branch starts at the current HEAD of main, which includes that commit.  
  
Then reset main to remove it. The commit now exists only on the feature branch — exactly where it should be.  
  
Only safe before you push. If already pushed to main (and main isn't protected), use `git revert` instead of `git reset`.

#### 21.2 Added .env or node_modules to Git

```bash
# Remove from Git tracking WITHOUT deleting the file
git rm --cached .env
git rm -r --cached node_modules/
# Now add to .gitignore
echo ".env" >> .gitignore
echo "node_modules/" >> .gitignore
# Commit the removal
git commit -m "chore: untrack .env and node_modules"
```

**📖 Fix: Untrack Already-Committed Files**

`git rm --cached` removes a file from Git tracking WITHOUT deleting the actual file from your disk — it still exists locally, Git just stops watching it.  
  
After committing the removal, add to `.gitignore` so it's never tracked again.  
  
**Critical:** if `.env` was already pushed to GitHub, the secret is compromised — rotate your credentials immediately regardless of the Git fix.

#### 21.3 Push Rejected: Non-Fast-Forward

```bash
# Push rejected — GitHub has commits you don't
# ! [rejected] main → main (non-fast-forward)
# Option 1: Pull and merge (creates merge commit)
git pull
# Option 2: Pull and rebase (cleaner history)
git pull --rebase
```

**📖 Fix: Diverged History**

"Non-fast-forward" means GitHub has commits your local doesn't — someone else pushed while you were working.  
  
`git pull` merges their commits with yours (creates a merge commit — safe but slightly messy).  
  
`git pull --rebase` fetches the new commits and replays your commits on top of them — cleaner, linear history. Use `--rebase` on shared branches for best results.

#### 21.4 Accidentally Deleted a Branch

```bash
# Find the last commit of the deleted branch
git reflog
# Look for "checkout: moving from feature/emi..."
# Note the hash that was HEAD before deletion
# Recreate the branch from that commit
git switch -c feature/emi-support a3f9c21
```

**📖 Fix: Recover Deleted Branch**

When you delete a branch, the commits don't disappear immediately — they're orphaned (no branch pointing to them) but still in Git's object database for 90 days.  
  
The reflog shows every HEAD position — including the last commit on the deleted branch. Find it, copy the hash, and recreate the branch pointing to that hash. All your work is restored.

### 22. Scenario-Based Learning: Real NovaPay Situations

Apply each scenario in a test repository — muscle memory is built by doing, not reading.

#### Scenario A — Friday Afternoon Hotfix

1. Alert: 3% of UPI payments failing
You're mid-feature on `feature/emi-support`. Run `git stash` to save your unfinished work without committing it.

2. Create the hotfix branch from main
`git switch main` → `git pull` → `git switch -c hotfix/upi-null-amount`. Now you're on a clean branch from the latest production code.

3. Fix, commit, push, open PR
Fix the bug. `git commit -m "fix(upi): reject null amount before validation"`. `git push -u origin hotfix/upi-null-amount`. Open PR. Tanvi approves in 5 minutes. Merge.

4. Tag the release and return to your feature
`git tag -a v1.2.1 -m "Hotfix: UPI null amount fix"` → `git push origin --tags`. Switch back: `git switch feature/emi-support` → `git stash pop`. Continue exactly where you left off.

#### Scenario B — New Developer Joins NovaPay

1. Set up Git and SSH on a new machine
Install Git. `git config --global user.name` and `user.email`. Generate SSH key: `ssh-keygen -t ed25519 -C "dev@novapay.in"`. Add public key to GitHub Settings.

2. Clone and explore the repository
`git clone git@github.com:novapay/payment-api.git`. Run `git log --oneline` to see project history. `git branch` to see all local branches. `git branch -r` to see remote branches.

3. First ticket: feature/NP-201-add-gpay-support
`git switch -c feature/NP-201-add-gpay-support`. Make changes. Commit with Conventional Commits format. Push. Open PR. Get reviewed. First merged PR — welcome to the team.

### 23. Phase 23 — Commit Messages: The Right Way

**Business Problem:** NovaPay's git log looks like: "fix", "update", "changes", "wip", "done". When a payment bug hits production at midnight and Tanvi needs to find which commit broke things, she can't tell one "fix" from another. A well-written commit history is the most valuable debugging tool a team has.

**Scene 23 — NovaPay Production Incident | "Which Commit Broke It?"**

> **Tanvi** _Senior Platform Engineer_
> 
> UPI payments are failing for amounts over ₹50,000. I'm looking at git log and I see: "fix", "update", "more changes", "final fix", "final fix 2". Eight commits today. I cannot tell which one is the problem — I have to read every diff. This is costing ten minutes for every minute of downtime. Commit messages are not optional. They are documentation.

> **Roshan** _DevOps Architect_
> 
> Priya, enforce Conventional Commits from today. Format: type(scope): description. Example: fix(upi): reject null amount before validation. When every message follows this, I can run git log --grep="fix(upi)" and instantly find every UPI bug fix ever committed. That is the value — months of work becomes searchable in two seconds.

#### 23.1 Conventional Commits Format

```bash
# Format: type(scope): short description
git commit -m "feat(payments): add EMI plan selection"
git commit -m "fix(upi): handle null amount in callback"
git commit -m "chore(deps): upgrade express to 4.19.2"
git commit -m "docs(readme): add API endpoint reference"
git commit -m "refactor(auth): extract token validation"
```

**📖 Conventional Commits Types**

**feat** — a new feature.  
  
**fix** — a bug fix.  
  
**chore** — maintenance work: deps, build config, no user-visible change.  
  
**docs** — documentation changes only.  
  
**refactor** — code restructure, no behaviour change.  
  
**test** — adding or updating tests.  
  
**ci** — CI/CD config changes.  
  
The **scope** in parentheses names the module — optional but recommended for searchability.

#### 23.2 Multi-Line Commit Messages

```bash
# Open VS Code to write a full commit message
git commit
# Write in the editor (blank line separates subject from body):

fix(payments): reject UPI transactions over 2L

RBI regulation requires extra verification for UPI
transactions above Rs 2,00,000. Without this check,
high-value transactions failed silently at the bank
without returning a clear error to the user.

Resolves: NP-891
Tested-by: Arjun Singh
```

**📖 When to Write a Long Message**

For simple changes, one line is fine. For behaviour-changing fixes, add a blank line after the subject and then a body explaining **why** — not what (the diff shows what).  
  
The **why** is invaluable. "RBI requires verification for high-value UPI" — that context would take hours to rediscover without the commit message.  
  
Link tickets in the footer: `Resolves: NP-891`. GitHub and Jira both detect these and close tickets automatically on merge.

#### 23.3 Search Through Commit History

```bash
# Find all commits that touched payment.js
git log --oneline -- payment.js
# Find commits by keyword in the message
git log --grep="fix(upi)" --oneline
# See who last edited each line of a file
git blame payment.js
```

**📖 Finding Specific Changes**

`git log -- payment.js` shows only commits that changed that file — trace when a bug was introduced.  
  
`git log --grep="fix(upi)"` searches commit messages — only useful when messages are meaningful.  
  
`git blame payment.js` shows every line with commit hash, author, and date of the last change to it. Classic for "who wrote this and why?" debugging. Invaluable at 2am during an incident.

```
a3f9c21 (Kiran  2026-01-14 10:23)  const maxRetries = 5;
b7e2d14 (Arjun  2026-01-12 16:45)  const timeout = 30000;
c9a3f88 (Priya  2026-01-10 09:12)  const apiUrl = process.env.API_URL;
```

### 24. Phase 24 — Interactive Rebase: Clean Up History Before a PR

**Business Problem:** Kiran worked on the EMI feature for 2 weeks and made 47 commits — "wip", "typo", "fix typo", "actually fix", "done finally maybe". Before opening a PR, Tanvi wants 4 clean logical commits. Interactive rebase lets Kiran squash the 47 messy commits into 4 well-named ones without losing any code.

**Scene 24 — NovaPay PR Review | "47 Commits for a 200-Line Feature?"**

> **Tanvi** _Senior Platform Engineer_
> 
> Kiran, clean up the commit history before I review. 47 commits, 12 of which say "fix" and 8 say "wip". I want four commits: database schema, API endpoint, business logic, tests. Use interactive rebase. The goal is a history that explains the feature — not a diary of your afternoon.

#### 24.1 Interactive Rebase — Squash Multiple Commits

```bash
# Open interactive editor for last 4 commits
git rebase -i HEAD~4
# Git shows this in the editor (oldest at top):
pick a1b2c3 Add EMI database schema
pick d4e5f6 wip
pick g7h8i9 fix
pick j0k1l2 done for real

# Change "pick" to "squash" (s) to combine:
pick a1b2c3 Add EMI database schema
squash d4e5f6 wip
squash g7h8i9 fix
squash j0k1l2 done for real
```

**📖 Interactive Rebase Commands**

`git rebase -i HEAD~4` opens an editor listing the last 4 commits.  
  
Change the command word at the start of each line:  
**pick** — keep as-is (default).  
**squash (s)** — combine with the commit above.  
**reword (r)** — keep code, edit the message.  
**drop (d)** — delete this commit entirely.  
  
Save and close — Git applies your instructions.

```
# Git opens the message editor after squashing.
# Delete the "wip", "fix" lines.
# Write one clean message for the combined commit:

feat(payments): add EMI plan selection and DB schema

- emi_plans table: plan_id, months, interest_rate
- POST /payments/emi returns available plans
- Validates EMI eligibility by transaction amount

Resolves: NP-312
```

**📖 Writing the Combined Message**

Git shows all original messages — delete the noise and write one clean summary.  
  
All code changes from all 4 commits are preserved in the single resulting commit with one meaningful message. The PR reviewer sees one commit that tells the full story.

##### ⚠️ Only Rebase Commits That Aren't Shared Yet

Interactive rebase rewrites commit history — commits get new hashes. This is safe on your own feature branch that only you are using. It is NOT safe on shared branches or after a PR is open and colleagues have left review comments on specific commits. Rule: squash and clean up **before** opening the PR, not after.

### 25. Phase 25 — Git Bisect: Find the Bug Automatically

**Business Problem:** NovaPay's payment API worked in v1.2.0. It's broken now. There are 87 commits between them. Reading 87 diffs manually would take hours. Git bisect uses binary search — you say good or bad, and it finds the exact breaking commit in 7 steps (log₂(87) ≈ 7).

#### 25.1 Running a Bisect Session

```bash
# Start bisect
git bisect start
# Current state is broken
git bisect bad
# v1.2.0 was working
git bisect good v1.2.0
# Git checks out a middle commit.
# Test your app — bug present?
git bisect bad # or: git bisect good
# Repeat 5-6 more times. Git announces the culprit.
# End the session and return to your branch
git bisect reset
```

**📖 Binary Search in Action**

With 87 commits to search, Git checks commit 43 first. You test and answer. Then 21 or 65. Each round halves the search space.  
  
After 7 rounds, Git identifies the exact bad commit — showing author, date, message, and diff. Run `git show <hash>` to read the offending change in full.  
  
`git bisect reset` returns you to your original branch after the session.

```
Bisecting: 43 revisions left to test after this (roughly 6 steps)
[c9a3f88] fix(auth): correct JWT expiry on mobile sessions

# After 7 rounds:
d4e5f6a is the first bad commit
Author: Arjun Singh <arjun@novapay.in>
Date:   Thu Jan 14 16:45 2026
    feat(payments): add high-value UPI validation
```

### 26. Phase 26 — SSH Keys and Secure GitHub Access

**Business Problem:** Every developer types their GitHub token on every push. SSH keys solve this: your machine and GitHub authenticate automatically — no password, more secure, faster every time.

#### 26.1 Generate and Add an SSH Key

```bash
# Generate an ed25519 SSH key pair
ssh-keygen -t ed25519 -C "priya@novapay.in"
# Print your PUBLIC key — paste this into GitHub
cat ~/.ssh/id_ed25519.pub
# Test the connection
ssh -T git@github.com
# Switch your repo from HTTPS to SSH
git remote set-url origin git@github.com:novapay/payment-api.git
```

**📖 SSH Key Pairs**

Two files generated: `id_ed25519` (private — NEVER share) and `id_ed25519.pub` (public — safe to share).  
  
Copy the full output of `cat ~/.ssh/id_ed25519.pub` and paste into GitHub → Settings → SSH keys → New SSH key.  
  
After adding it, `ssh -T git@github.com` should say "Hi priya! You've successfully authenticated." Then switch your remote URL with `git remote set-url` and you'll never type a password again.

```
Hi priya! You've successfully authenticated, but GitHub does not provide shell access.
```

**Quiz: ❓ Final Quiz — You push to main by accident and need to undo the commit. Other developers have already pulled it. What is the correct command?**

- A) git reset --hard HEAD~1
- B) git revert HEAD
- C) git restore .
- D) git stash

> **Answer/explanation:** ✅ **Answer: B — git revert HEAD.** When other developers have already pulled the commit, resetting is dangerous — it rewrites history and will break their local copies. git revert creates a new commit that undoes the previous one, leaving the original commit in history intact. It's the safe, collaborative undo. git reset --hard is only safe before you push and before others have pulled.

### Git Project Complete 🎉

You have set up NovaPay's complete Git workflow — from git init on day one to commits, branches, remote repositories on GitHub, pull requests with protected branches, merge conflict resolution, rebase, stash, tags, cherry-pick, and CI hooks. You now speak the language of every engineering team in the world.

> **Tanvi**
> 
> "Six weeks ago, Arjun's ZIP file overwrote Kiran's critical payment fix. It cost us 40 lakhs. This week, Kiran opened a PR for the EMI feature, Arjun reviewed it, CI ran the tests automatically, and it merged to main with one click. That same night, we deployed to production with zero issues. The code, the reviewer, the timestamp — all of it is in Git history forever. That's not just version control. That's accountability, collaboration, and safety in one tool."

> **Roshan**
> 
> "The mental model that changes everything: Git is not a backup tool. It's a communication tool. Every commit message is a note to your future teammates explaining what you changed and why. A clean Git history is the highest form of documentation a team can have."

> **Arjun**
> 
> "And git blame? I used to dread that command. Now I love it — not to blame people, but because when I'm debugging a weird behavior, I can see exactly who changed that line, when, and what their commit message said. Half the time the commit message explains the context I needed. Three hours of debugging saved in three seconds."

> **Next: Advanced Git — Monorepos, Submodules, Signing Commits & Git Internals**

> - Git internals — how commits, trees, and blobs are stored as objects inside .git/objects/; understanding pack files and garbage collection
> - Interactive rebase — git rebase -i to squash, reorder, edit, and drop commits before a PR to clean up messy work-in-progress history
> - Git submodules and subtrees — manage multiple repositories as components of one larger project (common in microservice architectures)
> - Monorepo workflows — tools like Nx, Turborepo, and Lerna for managing many packages in a single Git repository with intelligent CI caching
> - Commit signing with GPG — cryptographically sign commits so GitHub shows a "Verified" badge, proving a commit genuinely came from you
> - Git bisect — binary search through commit history to automatically find the exact commit that introduced a bug
> - Advanced git log — filtering history by author, date, file path, commit message pattern, and generating statistics with git shortlog and git diff --stat
