# Complete Ubuntu Lab Manual – Linux Automation, Git, Agile, CI/CD and Prompt-Based Automation

A practical lab record and execution guide covering four experiments and their subquestions. The commands are intended for Ubuntu running on Windows 11 through WSL or VMware. Follow the steps in order and check outputs as you go.

## 1. Setup notes and previous issues

| Previous issue | How this guide avoids it |
|---|---|
| `readlink -f $(which java)` reported a missing operand | This command is not needed. |
| Java was not found | This lab does not require Java or Hadoop. |
| Temporary failure resolving `in.archive.ubuntu.com` | Check network connectivity before installing packages. |
| Confusion between VMware and WSL Ubuntu | Identify the Ubuntu environment before configuring services. |
| Complicated installation commands | Use Ubuntu packages where available. |

These four experiments do not require Hadoop, Java, Pig, or Hive.

### Integrated project

Instead of performing four unrelated experiments, we will build an **Automated Task Scheduling and Monitoring System** using Ubuntu, cron/systemd timers, Bash scripts, Git/GitHub, Agile tracking, GitHub Actions, logs, and reports.

## Experiment 1 – Job Scheduling and Automation in Linux

### Aim

Create and manage cron jobs and systemd timers in Ubuntu to automate recurring tasks, schedule scripts, maintain logs, handle job dependencies, and troubleshoot failed jobs.

### 1.1 Create and manage cron jobs

#### Step 1: Open Ubuntu and identify the environment

```bash
lsb_release -a
whoami
pwd
```

#### Step 2: Update packages and install required tools

Check network connectivity first if package installation has previously failed. Then run:

```bash
sudo apt update
sudo apt install cron git curl nano -y
sudo service cron start
sudo service cron status
```

Expected result: cron is active or running.

#### Step 3: Understand cron syntax

```text
* * * * * command
│ │ │ │ └── Day of week
│ │ │ └──── Month
│ │ └────── Day of month
│ └──────── Hour
└────────── Minute
```

| Cron expression | Meaning |
|---|---|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour |
| `0 9 * * *` | Every day at 9 AM |
| `0 8 * * 1` | Every Monday at 8 AM |
| `0 0 1 * *` | First day of every month |

#### Step 4: Create the project directory

```bash
mkdir -p ~/automation-lab
cd ~/automation-lab
mkdir -p scripts logs reports
ls
```

Expected output: `logs  reports  scripts`.

#### Step 5: Create and run a Bash script

```bash
nano scripts/hello.sh
```

Enter:

```bash
#!/bin/bash

echo "Automation executed successfully"
echo "Date: $(date)"
echo "User: $(whoami)"
```

Save with **Ctrl+O**, press **Enter**, then exit with **Ctrl+X**. Make it executable and run it:

```bash
chmod +x scripts/hello.sh
./scripts/hello.sh
```

Expected output includes the success message, date, and user.

#### Step 6: Schedule a cron job

Open the crontab editor:

```bash
crontab -e
```

Choose nano if prompted. Add this line, replacing `YOUR_USERNAME` with the result of `whoami`:

```cron
*/5 * * * * /bin/bash /home/YOUR_USERNAME/automation-lab/scripts/hello.sh >> /home/YOUR_USERNAME/automation-lab/logs/cron.log 2>&1
```

This runs every five minutes. `>>` appends output, and `2>&1` sends errors to the same log.

#### Step 7: Verify the cron job

```bash
crontab -l
```

Wait five minutes and inspect the log:

```bash
cat ~/automation-lab/logs/cron.log
```

### 1.2 Create and manage systemd timers

Systemd timers can run services at specified intervals and provide service dependencies, execution status, and journal logs.

#### WSL note

Check the init process:

```bash
ps -p 1 -o comm=
```

If it prints `systemd`, continue. If you use WSL and systemd is not running, enable it in `/etc/wsl.conf`:

```ini
[boot]
systemd=true
```

Save the file, then run `wsl --shutdown` in Windows PowerShell (not Ubuntu). Reopen Ubuntu and check again:

```bash
ps -p 1 -o comm=
```

#### Step 1: Create a systemd service

```bash
sudo nano /etc/systemd/system/automation.service
```

Enter the following and replace `YOUR_USERNAME` with your Ubuntu username:

```ini
[Unit]
Description=Automated Task Execution

[Service]
Type=oneshot
ExecStart=/bin/bash /home/YOUR_USERNAME/automation-lab/scripts/hello.sh
```

#### Step 2: Create a systemd timer

```bash
sudo nano /etc/systemd/system/automation.timer
```

Enter:

```ini
[Unit]
Description=Run Automation Every Five Minutes

[Timer]
OnBootSec=1min
OnUnitActiveSec=5min
Unit=automation.service

[Install]
WantedBy=timers.target
```

#### Step 3: Reload systemd and enable the timer

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now automation.timer
```

#### Step 4: Verify the timer and service

```bash
systemctl list-timers --all
sudo systemctl status automation.service
sudo journalctl -u automation.service
```

### 1.3 Schedule scripts for periodic execution and logging

Create `scripts/backup.sh`:

```bash
nano scripts/backup.sh
```

Enter:

```bash
#!/bin/bash

DATE=$(date '+%Y-%m-%d %H:%M:%S')

echo "Backup task started at $DATE"

mkdir -p "$HOME/automation-lab/backup"

cp "$HOME/automation-lab/scripts/hello.sh" \
"$HOME/automation-lab/backup/hello.sh"

echo "Backup completed successfully"
```

Save, make executable, and run it:

```bash
chmod +x scripts/backup.sh
./scripts/backup.sh
ls backup
```

Expected output in the backup directory: `hello.sh`.

Schedule the backup hourly by adding this line to `crontab -e` (replace `YOUR_USERNAME`):

```cron
0 * * * * /bin/bash /home/YOUR_USERNAME/automation-lab/scripts/backup.sh >> /home/YOUR_USERNAME/automation-lab/logs/backup.log 2>&1
```

### 1.4 Handle dependencies and troubleshoot failed jobs

#### A. Create a dependent report script

Create `scripts/report.sh`:

```bash
#!/bin/bash

if [ -f "$HOME/automation-lab/backup/hello.sh" ]; then
    echo "Backup exists"
    echo "Report generated successfully"
else
    echo "Backup not found"
    exit 1
fi
```

Run the backup first, then the report:

```bash
chmod +x scripts/report.sh
./scripts/backup.sh
./scripts/report.sh
```

Expected output:

```text
Backup exists
Report generated successfully
```

#### B. Execute dependent jobs together

Create `scripts/workflow.sh`:

```bash
#!/bin/bash

echo "Starting workflow"

if "$HOME/automation-lab/scripts/backup.sh"; then
    echo "Backup successful"
    if "$HOME/automation-lab/scripts/report.sh"; then
        echo "Report successful"
    else
        echo "Report failed"
        exit 1
    fi
else
    echo "Backup failed"
    exit 1
fi

echo "Workflow completed"
```

Run:

```bash
chmod +x scripts/workflow.sh
./scripts/workflow.sh
```

#### C. Troubleshoot failed jobs

| Problem | Command |
|---|---|
| Check cron status | `sudo service cron status` |
| List cron jobs | `crontab -l` |
| Edit cron jobs | `crontab -e` |
| Check systemd timers | `systemctl list-timers --all` |
| Check service status | `sudo systemctl status automation.service` |
| Check service logs | `sudo journalctl -u automation.service` |
| Check file permissions | `ls -l scripts/` |
| Give execute permission | `chmod +x scripts/hello.sh` |

Practice a missing-script error with:

```bash
bash scripts/missing.sh
```

Expected result: `No such file or directory`.

### 1.5 Use scheduling to update Git repositories or Agile project boards

Initialize Git:

```bash
cd ~/automation-lab
git init
```

Create `scripts/status.sh`:

```bash
#!/bin/bash

echo "Automation Status"
echo "Date: $(date)"
echo "System: $(uname -s)"
echo "User: $(whoami)"
```

Make it executable and run it:

```bash
chmod +x scripts/status.sh
./scripts/status.sh
```

Create `scripts/auto_update.sh`:

```bash
#!/bin/bash

cd "$HOME/automation-lab" || exit 1

./scripts/status.sh > reports/status.txt
git add reports/status.txt

if ! git diff --cached --quiet; then
    git commit -m "Automated status update"
fi
```

Run it and optionally schedule it hourly:

```bash
chmod +x scripts/auto_update.sh
./scripts/auto_update.sh
```

Add to crontab (replace `YOUR_USERNAME`):

```cron
0 * * * * /bin/bash /home/YOUR_USERNAME/automation-lab/scripts/auto_update.sh >> /home/YOUR_USERNAME/automation-lab/logs/git-update.log 2>&1
```

The script generates a status report, stages it, and commits it if it changed.

## Experiment 2 – Git and GitHub Integration

### Aim

Initialize Git repositories, track changes in automation scripts, commit outputs, push changes to GitHub, manage branches, and implement Git hooks to validate scripts.

### 2.1 Initialize a repository and track changes

```bash
git --version
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
git config --list
cd ~/automation-lab
git init
ls -a
git status
git add .
git status
git commit -m "Initial automation project"
git log --oneline
```

A commit records a snapshot of the project; `ls -a` should show `.git`.

### 2.2 Commit generated job outputs automatically

Create `scripts/git_backup.sh`:

```bash
#!/bin/bash

cd "$HOME/automation-lab" || exit 1

DATE=$(date '+%Y-%m-%d %H:%M:%S')

echo "Automated Git Backup" > reports/backup.txt
echo "Execution Time: $DATE" >> reports/backup.txt

git add reports/backup.txt

if ! git diff --cached --quiet; then
    git commit -m "Automated backup: $DATE"
    echo "Changes committed successfully"
else
    echo "No changes to commit"
fi
```

Run and verify:

```bash
chmod +x scripts/git_backup.sh
./scripts/git_backup.sh
cat reports/backup.txt
git log --oneline
```

### 2.3 Push to GitHub and manage branches

Install GitHub CLI if available in your Ubuntu package sources:

```bash
sudo apt install gh -y
gh --version
```

If Ubuntu cannot locate `gh`, stop rather than downloading an unofficial package; consult GitHub's official installation instructions for your Ubuntu version.

Authenticate:

```bash
gh auth login
gh auth status
```

Choose GitHub.com, HTTPS, authenticate Git with your GitHub credentials, and use the browser login flow.

Create a repository and push the project:

```bash
cd ~/automation-lab
gh repo create automation-lab --public --source=. --remote=origin
git branch -M main
git push -u origin main
```

Use `--private` instead of `--public` if appropriate.

Create task branches:

```bash
git switch -c backup-automation
git switch main
git switch -c report-automation
git branch
```

Make and push a report task on the report branch:

```bash
echo "Report automation task" > reports/task.txt
git add .
git commit -m "Add report automation task"
git push -u origin report-automation
```

Merge it into main:

```bash
git switch main
git merge report-automation
git push origin main
```

### 2.4 Implement Git hooks

Local Git hooks live in `.git/hooks` and are not automatically transferred when cloning a repository.

#### A. Pre-commit hook

Create `.git/hooks/pre-commit`:

```bash
#!/bin/bash

echo "Running pre-commit validation..."

for file in scripts/*.sh
do
    if [ -f "$file" ]; then
        bash -n "$file"
        if [ $? -ne 0 ]; then
            echo "Syntax error found in $file"
            exit 1
        fi
    fi
done

echo "All scripts are valid"
exit 0
```

Make executable and test:

```bash
chmod +x .git/hooks/pre-commit
git add .
git commit -m "Test pre-commit validation"
```

The hook permits a commit only if the shell scripts pass `bash -n`.

#### B. Post-commit hook

Create `.git/hooks/post-commit`:

```bash
#!/bin/bash

mkdir -p "$HOME/automation-lab/logs"

echo "Commit completed successfully" \
>> "$HOME/automation-lab/logs/commit-history.log"

git log -1 --pretty=format:"%h %s" \
>> "$HOME/automation-lab/logs/commit-history.log"

echo "" >> "$HOME/automation-lab/logs/commit-history.log"
```

Make executable, test, and inspect:

```bash
chmod +x .git/hooks/post-commit
echo "Testing post commit" > reports/test.txt
git add .
git commit -m "Test post commit hook"
cat logs/commit-history.log
```

## Experiment 3 – Agile Project Management with Git

### Aim

Create an Agile project board, manage work using GitHub Issues, connect scheduled jobs with user stories, track status, and maintain traceability between issues and commits.

### 3.1 Create a project board

On GitHub, open your account's **Projects** page and create a **Board** named **Automation Workflow Management**. Configure these statuses:

| Status | Meaning |
|---|---|
| Backlog | Tasks identified but not yet planned |
| To Do | Tasks ready to begin |
| In Progress | Tasks currently executing |
| Done | Successfully completed tasks |

Add sample tasks/issues:

| Task ID | Task description |
|---|---|
| T1 | Create Linux cron job |
| T2 | Implement backup script |
| T3 | Integrate GitHub repository |
| T4 | Configure CI/CD pipeline |
| T5 | Generate automated reports |

### 3.2 Map cron/systemd tasks to user stories or issues

Create a GitHub issue titled **Automate recurring Linux backup** with this user story:

> As a system administrator, I want to schedule automated backup jobs, so that important project files are backed up regularly.

Acceptance criteria:

1. Create a backup script.
2. Schedule the script using cron.
3. Generate execution logs.
4. Verify backup completion.

Add the issue to the project board and move it to **To Do**. Example mapping:

| Scheduled task | GitHub issue |
|---|---|
| Cron backup | Linux backup automation |
| Report generation | Automated report generation |
| Git commit | Repository update |
| CI/CD execution | Pipeline validation |

### 3.3 Track task status

Create or use labels named `Backlog`, `To Do`, `In Progress`, `Done`, and `Failed`. Issue labels can track status; connect them to the Project board separately if you want the board's Status field to update automatically.

Create commonly used labels with GitHub CLI:

```bash
gh label create "In Progress" --color "FBCA04" --force
gh label create "Done" --color "0E8A16" --force
gh label create "Failed" --color "D73A4A" --force
```

For example, assuming the backup issue is #1:

```bash
gh issue edit 1 --add-label "In Progress"
gh issue edit 1 --remove-label "In Progress" --add-label "Done"
gh issue edit 1 --remove-label "In Progress" --add-label "Failed"
gh issue view 1
```

Replace `1` with your actual issue number.

### 3.4 Link commits to Agile tasks

For example, for Issue #1:

```bash
git switch -c issue-1-backup
```

Add a comment such as `# Issue #1: Automated backup implementation` to `scripts/backup.sh`, then:

```bash
git add scripts/backup.sh
git commit -m "Implement automated backup for #1"
git push -u origin issue-1-backup
gh pr create --base main --head issue-1-backup --title "Implement automated backup" --body "Implements the backup task tracked in #1."
```

The issue number in the commit and pull request provides traceability.

## Experiment 4 – CI/CD Pipeline and Prompt-Based Automation

### Aim

Configure GitHub Actions to test and run scripts, update Agile issue statuses based on pipeline results, and generate reports using a prompt-based analysis workflow.

### 4.1 Configure a CI/CD pipeline with GitHub Actions

The workflow will run on pushes to `main`, every six hours, and manual dispatch. It validates scripts, runs the automation, generates a report, uploads reports as artifacts, and updates an issue's labels.

Create `.github/workflows/automation.yml`:

```yaml
name: Automation CI CD Pipeline

on:
  push:
    branches:
      - main
  schedule:
    - cron: '0 */6 * * *'
  workflow_dispatch:

permissions:
  contents: read
  issues: write

jobs:
  automation:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup Environment
        run: echo "Automation environment ready"

      - name: Update Agile Status
        env:
          GH_TOKEN: ${{ github.token }}
          ISSUE_NUMBER: ${{ vars.AGILE_ISSUE_NUMBER }}
          GH_REPO: ${{ github.repository }}
        run: |
          if [ -n "$ISSUE_NUMBER" ]; then
            gh issue edit "$ISSUE_NUMBER" \
              --remove-label "Done" \
              --remove-label "Failed" \
              --add-label "In Progress"
          fi

      - name: Validate Bash Scripts
        run: |
          for file in scripts/*.sh
          do
            bash -n "$file"
          done
          echo "All scripts validated"

      - name: Execute Automation
        run: |
          chmod +x scripts/*.sh
          bash scripts/workflow.sh

      - name: Generate Report
        if: success()
        run: bash scripts/analyze_logs.sh

      - name: Save Generated Reports
        if: success()
        uses: actions/upload-artifact@v4
        with:
          name: automation-reports
          path: reports/

      - name: Mark Agile Task Done
        if: success()
        env:
          GH_TOKEN: ${{ github.token }}
          ISSUE_NUMBER: ${{ vars.AGILE_ISSUE_NUMBER }}
          GH_REPO: ${{ github.repository }}
        run: |
          if [ -n "$ISSUE_NUMBER" ]; then
            gh issue edit "$ISSUE_NUMBER" \
              --remove-label "In Progress" \
              --remove-label "Failed" \
              --add-label "Done"
          fi

      - name: Mark Agile Task Failed
        if: failure()
        env:
          GH_TOKEN: ${{ github.token }}
          ISSUE_NUMBER: ${{ vars.AGILE_ISSUE_NUMBER }}
          GH_REPO: ${{ github.repository }}
        run: |
          if [ -n "$ISSUE_NUMBER" ]; then
            gh issue edit "$ISSUE_NUMBER" \
              --remove-label "In Progress" \
              --add-label "Failed"
          fi
```

The job uses checkout, Bash syntax validation, the workflow script, report generation, artifact upload, and issue-label updates. The provided `GITHUB_TOKEN` is used; no personal access token is needed for this example.

Configure the issue number as a repository Actions variable: **Settings → Secrets and variables → Actions → Variables → New repository variable**. Set name `AGILE_ISSUE_NUMBER` and value to the issue number. The workflow token needs the declared `issues: write` permission, and the labels should exist.

Commit and push the workflow:

```bash
cd ~/automation-lab
git add .github/workflows/automation.yml
git commit -m "Add GitHub Actions CI CD pipeline"
git push origin main
```

Verify it under **Repository → Actions → Automation CI CD Pipeline**. A green check indicates success; a red cross indicates a failed step. To test manually, choose the workflow, click **Run workflow**, select `main`, and start it.

### 4.2 Update Agile task status from pipeline results

Create the labels if needed:

```bash
gh label create "In Progress" --color "FBCA04" --force
gh label create "Done" --color "0E8A16" --force
gh label create "Failed" --color "D73A4A" --force
```

The workflow adds `In Progress` when it starts, then removes it and adds `Done` after success or `Failed` after failure. Verify with:

```bash
gh issue view 1
```

Replace `1` with the actual issue number. This updates issue labels; configure Project board workflows separately if you want the board's Status column to follow those labels.

### 4.3 Use prompt-based analysis to summarize results and detect errors

This local demonstration uses a prompt template and Bash; it does not call an AI API and requires no API key.

Create `scripts/prompt.txt`:

```text
You are an automation monitoring assistant.

Analyze the following automation logs.

Your tasks:

1. Summarize the execution results.
2. Identify errors and failures.
3. Find possible reasons for failure.
4. Suggest corrective actions.
5. Provide the final execution status.

Logs:
```

Create `scripts/analyze_logs.sh`:

```bash
#!/bin/bash

cd "$HOME/automation-lab" || exit 1

mkdir -p reports
REPORT="reports/automation-report.txt"

echo "AUTOMATION EXECUTION REPORT" > "$REPORT"
echo "Date: $(date)" >> "$REPORT"
echo "------------------------" >> "$REPORT"

if [ -f logs/cron.log ]; then
    cat logs/cron.log >> "$REPORT"
fi
if [ -f logs/backup.log ]; then
    cat logs/backup.log >> "$REPORT"
fi
if [ -f logs/git-update.log ]; then
    cat logs/git-update.log >> "$REPORT"
fi

echo "" >> "$REPORT"
echo "ERROR ANALYSIS" >> "$REPORT"
if grep -Eiq "error|failed|failure" logs/*.log 2>/dev/null; then
    echo "Errors detected in automation logs" >> "$REPORT"
else
    echo "No errors detected in automation logs" >> "$REPORT"
fi

echo "" >> "$REPORT"
echo "ACTIONABLE RECOMMENDATIONS" >> "$REPORT"
echo "1. Verify cron service status" >> "$REPORT"
echo "2. Check script permissions" >> "$REPORT"
echo "3. Verify Git repository status" >> "$REPORT"
echo "4. Review failed job logs" >> "$REPORT"
echo "" >> "$REPORT"
echo "REPORT GENERATED SUCCESSFULLY" >> "$REPORT"
cat "$REPORT"
```

Make it executable and run it:

```bash
chmod +x scripts/analyze_logs.sh
./scripts/analyze_logs.sh
cat reports/automation-report.txt
```

This is a basic text scan for common error indicators, not a complete diagnosis. Combine the template and report to make a prompt that can be copied into an AI assistant:

```bash
cat scripts/prompt.txt reports/automation-report.txt > reports/ai-analysis-prompt.txt
cat reports/ai-analysis-prompt.txt
```

### 4.4 Share reports through GitHub, artifacts, or an issue

Commit and push reports to GitHub:

```bash
git add reports/
git commit -m "Generate automation execution report"
git push origin main
```

The workflow uploads `reports/` as an `automation-reports` artifact after successful execution. Retrieve it from the completed workflow's **Artifacts** section.

You can also comment on the issue (replace `1`):

```bash
gh issue comment 1 --body "Automation report generated. Check the reports directory in the repository."
```

Email integration requires an authenticated email service or configured SMTP client. For this lab, GitHub reports and artifacts demonstrate sharing without adding email credentials.

## 5. Final integrated execution and verification

Earlier examples use the fixed `~/automation-lab` path. GitHub Actions checks the repository out into a different directory, so update the scripts below to use paths relative to their own location before running them in CI.

### 5.1 Make scripts portable

Replace `scripts/backup.sh` with:

```bash
#!/bin/bash

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
ROOT_DIR="$(dirname "$SCRIPT_DIR")"

DATE=$(date '+%Y-%m-%d %H:%M:%S')
echo "Backup started at $DATE"

mkdir -p "$ROOT_DIR/backup"
cp "$SCRIPT_DIR/hello.sh" "$ROOT_DIR/backup/hello.sh"

echo "Backup completed successfully"
```

Replace `scripts/report.sh` with:

```bash
#!/bin/bash

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
ROOT_DIR="$(dirname "$SCRIPT_DIR")"

if [ -f "$ROOT_DIR/backup/hello.sh" ]; then
    echo "Backup exists"
    echo "Report generated successfully"
else
    echo "Backup not found"
    exit 1
fi
```

Replace `scripts/workflow.sh` with:

```bash
#!/bin/bash

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

echo "Starting workflow"

if bash "$SCRIPT_DIR/backup.sh"; then
    echo "Backup successful"
    if bash "$SCRIPT_DIR/report.sh"; then
        echo "Report successful"
    else
        echo "Report failed"
        exit 1
    fi
else
    echo "Backup failed"
    exit 1
fi

echo "Workflow completed"
```

Replace `scripts/analyze_logs.sh` with:

```bash
#!/bin/bash

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
ROOT_DIR="$(dirname "$SCRIPT_DIR")"

mkdir -p "$ROOT_DIR/reports"
REPORT="$ROOT_DIR/reports/automation-report.txt"

echo "AUTOMATION EXECUTION REPORT" > "$REPORT"
echo "Date: $(date)" >> "$REPORT"
echo "------------------------" >> "$REPORT"

for file in "$ROOT_DIR"/logs/*.log
do
    if [ -f "$file" ]; then
        cat "$file" >> "$REPORT"
    fi
done

echo "" >> "$REPORT"
echo "ERROR ANALYSIS" >> "$REPORT"
if grep -Eiq "error|failed|failure" "$ROOT_DIR"/logs/*.log 2>/dev/null; then
    echo "Errors detected" >> "$REPORT"
else
    echo "No errors detected in available logs" >> "$REPORT"
fi

echo "" >> "$REPORT"
echo "ACTIONABLE RECOMMENDATIONS" >> "$REPORT"
echo "1. Check cron service" >> "$REPORT"
echo "2. Check script permissions" >> "$REPORT"
echo "3. Verify Git repository" >> "$REPORT"
echo "4. Review execution logs" >> "$REPORT"
echo "Report generated successfully" >> "$REPORT"

cat "$REPORT"
```

Make shell scripts executable:

```bash
chmod +x scripts/*.sh
```

### 5.2 Run and verify the project

```bash
cd ~/automation-lab
for file in scripts/*.sh; do bash -n "$file" || exit 1; done
bash scripts/workflow.sh
bash scripts/analyze_logs.sh
ls backup/
cat reports/automation-report.txt
git status
git add .
git commit -m "Complete integrated automation workflow"
git push origin main
```

If Git says there is nothing to commit, the changes are already committed. Finally, check **GitHub → Your repository → Actions** and confirm the pipeline result.

## 6. Ubuntu troubleshooting guide

| Error | Solution |
|---|---|
| Permission denied | `chmod +x filename.sh` |
| Command not found | Check spelling and install the required package. |
| No such file or directory | Check location with `pwd` and `ls`. |
| Cron not running | `sudo service cron start` |
| Git identity unknown | Configure `git config --global user.name` and `git config --global user.email`. |
| Git push rejected | Run `git pull --rebase origin main` and resolve conflicts if any. |
| GitHub authentication failed | Run `gh auth login`. |
| systemd not running in WSL | Enable systemd in `/etc/wsl.conf` and restart WSL. |
| CI/CD workflow fails | Open Actions and inspect the failed step's logs. |
| Issue update fails | Check the issue number, labels, and Actions permissions. |

A cron job scheduled inside WSL runs only while the WSL environment is running. For dependable background scheduling, keep Ubuntu running or use a native Ubuntu VM with systemd.

## 7. Practical record – evidence and screenshots

Capture the following evidence for your lab record.

### Experiment 1 – Linux Automation

- Cron job configuration
- Cron execution log
- Systemd timer status
- Successful backup output

### Experiment 2 – Git and GitHub

- Git initialization
- Git commit history
- GitHub repository
- Pre-commit and post-commit execution

### Experiment 3 – Agile Project Management

- GitHub Project board
- Created issues
- Status transitions
- Commit linked to an issue

### Experiment 4 – CI/CD and Reporting

- Workflow YAML file
- Successful GitHub Actions run
- Agile issue status update
- Generated automation report

## 8. Final lab revision

| Experiment | Main concepts | Important commands |
|---|---|---|
| 1. Linux Automation | Cron, systemd, logging, dependencies | `crontab -e`, `systemctl`, `journalctl` |
| 2. Git Integration | Init, add, commit, push, branches, hooks | `git init`, `git add`, `git commit`, `git push` |
| 3. Agile Management | User stories, issues, Kanban, traceability | `gh issue create`, `gh issue edit`, `gh issue view` |
| 4. CI/CD | GitHub Actions, testing, deployment, reporting | `git push`, GitHub Actions, `gh issue edit` |

## 9. Practical completion checklist

### Experiment 1

- [ ] Install required Ubuntu packages
- [ ] Create cron job
- [ ] Configure systemd timer
- [ ] Generate and maintain logs
- [ ] Execute dependent scripts
- [ ] Automate Git updates

### Experiment 2

- [ ] Initialize Git repository
- [ ] Commit automated job outputs
- [ ] Create GitHub repository
- [ ] Push and manage branches
- [ ] Implement pre-commit hook
- [ ] Implement post-commit hook

### Experiment 3

- [ ] Create GitHub Project board
- [ ] Create Agile user stories and issues
- [ ] Configure status labels
- [ ] Track task status
- [ ] Link commits to issues

### Experiment 4

- [ ] Create GitHub Actions workflow
- [ ] Configure automatic testing
- [ ] Update Agile issue status
- [ ] Generate prompt-based reports
- [ ] Publish reports to GitHub

## 10. Before starting: confirm your Ubuntu environment

Open your Ubuntu terminal and execute these three commands:

```bash
lsb_release -a
ps -p 1 -o comm=
whoami
```

Record the outputs. The Ubuntu version, init process, and username determine the correct setup steps, especially for cron and systemd. Proceed through Experiment 1 in order and verify each output before moving on.
