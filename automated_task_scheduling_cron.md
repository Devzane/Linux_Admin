# SOP: Automated Task Scheduling via Cron (crond)

📋 Document Metadata
- **Role Focus:** Site Reliability Engineering (SRE) / Infrastructure Automation
- **Component:** Time-based job scheduler (crond)
- **Environment:** RedHat Enterprise Linux (RHEL) / CentOS Stream 9
- **Execution Scope:** Fleet-wide Deployment (All Application Servers)

## 1️⃣ The Objective (The "Why")

In scalable infrastructure, relying on manual human intervention for routine tasks (e.g., database backups, log rotation, temporary file cleanup) introduces the risk of human error and operational toil. The objective of this operation is to provision, activate, and configure the crond daemon across the server fleet to execute automated scripts on a strict, time-based schedule without user interaction.

## 2️⃣ Target Infrastructure (The "Where")

- **The Daemon:** crond (The background service that wakes up every minute to check for scheduled tasks).
- **The Package:** cronie and cronie-anacron. (Modern RHEL distributions use cronie as the standard implementation of cron, with anacron ensuring that daily jobs still run even if the server was powered off during their scheduled time).
- **Storage Location:** User-specific cron schedules are physically stored in the `/var/spool/cron/` directory.

## 3️⃣ Execution (The "How")

### Phase 1: Provisioning the Dependencies

By default, some minimal container or cloud images do not ship with the cron daemon pre-installed to save space. We must ensure the correct binaries are present.

```bash
sudo yum install cronie cronie-anacron -y
```

*(Note: The `-y` flag is used in automation to bypass the user confirmation prompt during installation).*

### Phase 2: Activating the Daemon

Installing the package does not automatically start the process in the background. We must use systemd to start the service and ensure it survives a system reboot.

```bash
# Start the service for the current session
sudo systemctl start crond

# Enable the service to start automatically on hardware boot
sudo systemctl enable crond
```

### Phase 3: Injecting the Job Schedule

Every user on a Linux system can have their own isolated cron table (crontab). For system-level administrative tasks, jobs are typically injected into the root user's table.

To edit the root user's crontab safely:

```bash
sudo crontab -u root -e
```

- `-u root`: Explicitly specifies the target user. If omitted, it defaults to the currently logged-in user.
- `-e`: Opens the cron table in the system's default text editor (usually vi or vim) while checking for syntax errors upon saving.

#### The Syntax Implementation:
Once the editor is open, append the exact execution schedule. In this scenario, we are executing a command every 5 minutes.

```bash
*/5 * * * * echo hello > /tmp/cron_text
```

#### Architectural Breakdown of the Syntax:

- `*/5` (Minute): The `*/` operator denotes a step value. This triggers execution every 5 minutes (e.g., :00, :05, :10).
- `*` (Hour): Every hour.
- `*` (Day of Month): Every day of the month.
- `*` (Month): Every month.
- `*` (Day of Week): Every day of the week.
- `echo hello > /tmp/cron_text`: The payload command. It writes "hello" into a temporary file, overwriting its previous contents.

Save and exit the editor (`:wq` in vi). The crond daemon will automatically detect the updated file and apply the new schedule immediately.

## 4️⃣ Verification & Troubleshooting (The "Proof")

### Level 1: Configuration Verification

Do not assume the file was saved correctly. Verify the table exists in the system memory:

```bash
sudo crontab -u root -l
```

*(The `-l` flag lists the contents of the target user's crontab).*

### Level 2: Daemon State Verification

Confirm that crond is actively listening and has not crashed:

```bash
sudo systemctl status crond
```

Expected output must show `Active: active (running)`.

### Level 3: Execution Verification (Checking the Logs)

The ultimate proof of concept is verifying that the system is actually firing the job. In RedHat-based systems, cron logs its actions to a dedicated file.

```bash
sudo tail -f /var/log/cron
```

You should see a log entry every 5 minutes stating `(root) CMD (echo hello > /tmp/cron_text)`.

### Common SRE Troubleshooting Vectors

- **Absolute Paths:** The crond environment is stripped down. It does not load the user's `$PATH` variables. If scheduling complex bash scripts, always use absolute paths (e.g., `/usr/bin/python3 /opt/scripts/backup.py` instead of just `python3`).
- **Permission Denied:** If the cron job fails to execute a script, check the script's execution permissions (`chmod +x /path/to/script.sh`).
