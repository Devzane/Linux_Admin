# SOP: systemd Service Execution Lifecycle

## 📝 Objective

To manage, troubleshoot, and architect complex application startup sequences using `systemd`.

In enterprise environments, applications rarely boot up in isolation. A web application often needs to ensure the database is reachable, apply database migrations, or pull dynamic secrets from a vault before the main code executes. `systemd` uses the `ExecStart` family of directives to enforce a strict, synchronous startup sequence, preventing race conditions where an app fails because its dependencies aren't ready.

## ⚙️ Target Infrastructure

*   **OS:** Linux (All modern distributions using `systemd`)
*   **Component:** System and Service Manager (`systemctl`, `.service` unit files)

## 🏗️ The Execution Directives

When reading or writing a `/etc/systemd/system/*.service` file, look at the `[Service]` block. The execution order strictly follows this sequence:

| Order | Directive | Purpose | SRE Real-World Example |
| :--- | :--- | :--- | :--- |
| 1 | `ExecStartPre` | Runs before the main service starts. Must exit with a status of 0 (Success) or the entire service boot aborts. | Running a database migration script (`configure_db.sh`), checking network availability, or pulling AWS secrets. |
| 2 | `ExecStart` | The primary command that keeps the service running. This is the actual daemon or application. | Starting the Gunicorn web server, running `python3 /opt/code/my_app.py`. |
| 3 | `ExecStartPost` | Runs immediately after the main service (`ExecStart`) successfully launches. | Sending a "Service Online" webhook to a Slack channel, or registering the new server with an AWS Target Group. |

## 🛠️ Execution & Troubleshooting Steps

### 1. Identify the Failing Service

If an application is failing to start, first check its status.

```bash
sudo systemctl status <service_name>
```

Look closely at the error logs. If the service is dead, note whether it failed during the `Pre` step or the main `Start` step.

### 2. Inspect the Configuration (Without a text editor)

To understand what scripts the service is actually trying to run, use the `cat` command built into `systemctl`. This safely prints the active unit file to your terminal without the risk of accidentally editing it.

```bash
sudo systemctl cat <service_name>.service
```

### 3. Example Output Breakdown

```ini
[Unit]
Description=My Python Web Application

[Service]
ExecStartPre=/bin/bash /opt/code/configure_db.sh
ExecStart=/usr/bin/python3 /opt/code/my_app.py
ExecStartPost=/bin/bash /opt/code/email_status.sh
Restart=always
```

**Analysis:** If the `configure_db.sh` script fails (e.g., wrong database password), `systemd` will immediately halt the sequence. The `python3` application will never be launched, and no email will be sent.

## 🔐 SRE Best Practices

*   **Absolute Paths Only:** Notice that the configuration uses `/bin/bash` and `/usr/bin/python3`, not just `bash` or `python3`. `systemd` does not have access to your user's `$PATH` environment variable. You must always use absolute paths to binaries in unit files.
*   **The `ExecStartPre` Gatekeeper:** Use `ExecStartPre` to fail fast. It is better for a service to refuse to start (because a config file is missing) than to start up in a broken state and serve errors to customers.
*   **Multiple Pre-Scripts:** You can have multiple `ExecStartPre` lines in a single file. They will execute sequentially from top to bottom.

## 🐛 Common Errors

*   **Problem:** `systemctl status` shows `code=exited, status=203/EXEC`
*   **Solution:** The script referenced in `ExecStart` or `ExecStartPre` does not have executable permissions (missing `chmod +x`) or the absolute path to the binary is incorrect.
