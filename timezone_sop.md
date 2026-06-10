# SOP: Standardizing Linux Server Timezones via Timedatectl

## 📝 1. OBJECTIVE (The "Why")
**Objective:** To synchronize the timezone settings across Linux application servers to precisely match the physical datacenter's local timezone (in this case, `Asia/Macau`).
**Context:** In distributed infrastructure, time discrepancies between servers cause cascading failures in scheduled tasks (`cron`), state locking for Infrastructure as Code, and distributed logging. For teams executing overnight deployments to leverage timezone arbitrage, ensuring the datacenter logs perfectly align with the target local execution window is critical for accurate asynchronous debugging and auditing.

---

## ⚙️ 2. TARGET INFRASTRUCTURE / PREREQUISITES (The "Where")
- **OS:** Linux (Systemd-based distributions like CentOS, RHEL, Ubuntu)
- **Target Systems:** Nautilus Application Servers (Stratos Datacenter)
- **Required Tools:** `timedatectl` (native systemd utility)
- **Prerequisites:**
  - SSH access to the target application servers.
  - Sudo/administrative privileges.

---

## 🛠️ 3. EXECUTION (The "How")

Perform these steps sequentially on each required application server.

### Step 1: Check the Current Timezone State
Before making changes, document the current configuration to ensure you understand the delta.
```bash
timedatectl
```
**What it does:** Outputs the current Local time, Universal time (UTC), RTC time (hardware clock), and the currently active Time zone.
**Why:** Confirms whether the server is currently drifting or set to a default like UTC.

### Step 2: (Optional) Search for the Exact Timezone String
If you are unsure of the exact formatting for the required region, query the system database.
```bash
timedatectl list-timezones | grep -i macau
```
**What it does:** Lists all available timezones and filters the output for "macau".
**Expected output:** `Asia/Macau`

### Step 3: Set the New Timezone
Execute the change using the strict Continent/City nomenclature.
```bash
sudo timedatectl set-timezone Asia/Macau
```
**What it does:** Instantly updates the system clock configuration and alters the `/etc/localtime` symbolic link to point to the correct zone data file.
**Why:** This modifies the system state immediately without requiring a system reboot. All newly generated logs and scheduled cron jobs will instantly reflect the Macau timezone.

---

## ✅ 4. VERIFICATION & TROUBLESHOOTING (The "Proof")

### Verification:

**Verify State:**
```bash
timedatectl
```
**Expected Output:** The `Time zone:` field must explicitly read `Asia/Macau (CST, +0800)`.

**Verify System Date:**
```bash
date
```
**Expected Output:** The standard output should append the timezone abbreviation (e.g., CST for China Standard Time) at the end of the timestamp.

### Troubleshooting:

| Error | Meaning | Solution |
| :--- | :--- | :--- |
| `Failed to set time zone: Invalid time zone` | You misspelled the timezone or did not use the Continent/City format. | Run `timedatectl list-timezones` to find the exact string syntax. Linux is case-sensitive. |
| `Failed to set time zone: Interactive authentication required.` | You forgot to run the command with administrative privileges. | Prepend the command with `sudo`. |
| Logs still show the old time | The specific application (like Tomcat or Nginx) caches time on boot. | Restart the specific application service (`sudo systemctl restart <service_name>`). |
