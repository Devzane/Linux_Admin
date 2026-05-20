# SOP: MariaDB / MySQL Daemon Permission Restoration

## 📝 Objective

To restore service availability to a MariaDB/MySQL database that has crashed due to incorrect file ownership. The database daemon requires strict ownership of its storage directories to execute read/write operations.

## ⚙️ Target Infrastructure

- **Component**: MariaDB / MySQL Server
- **Default Storage Path**: `/var/lib/mysql`
- **Required Owner/Group**: `mysql:mysql`

## 🛠️ Execution Steps

### 1. Initial Triage

Before diving into advanced logs, always perform a baseline health check. Verify the current state of the daemon, and if it is stopped, attempt to start it to force the system to generate a fresh error state.

```bash
# Check the current status of the daemon
sudo systemctl status mariadb

# Attempt to start the service (this will fail, but it generates fresh logs)
sudo systemctl start mariadb
```

### 2. Deep Diagnosis (The SRE Way)

Standard status output often only tells you that the service failed, not why. To extract the exact error and the specific file path causing the permission denial, query the systemd journal with extended diagnostics.

```bash
# Query the end of the journal with extended explanations for the specific service
sudo journalctl -xeu mariadb.service
```

**Flag Breakdown**:
- `-x`: Extended mode (adds explanatory text to the logs, often pointing directly to the broken path).
- `-e`: Jumps straight to the end of the pager (the most recent crash logs).
- `-u mariadb.service`: Filters out all other OS noise, showing only the MariaDB unit.

Once you identify the broken path from the logs, verify its underlying directory ownership. Look for `root` (or another user) where `mysql` should be.

```bash
ls -ld /var/lib/mysql
```

### 3. Execution (The Fix)

Apply recursive ownership changes to the data directory, ensuring both the user owner and group owner are set to the database service account.

```bash
sudo chown -R mysql:mysql /var/lib/mysql
```

### 4. Verification

Restart the daemon and confirm the process remains active and listening.

```bash
sudo systemctl start mariadb
sudo systemctl status mariadb
```
