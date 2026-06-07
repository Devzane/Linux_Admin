---
**Topic:** Default GUI Boot Configuration (Systemd Targets)
**Category:** Linux System Administration
**Difficulty:** Beginner
**Last Updated:** June 7, 2026
**Related SOPs:** SSH Access Management, Systemctl Service Management
---

# SOP: Configuring Default GUI Boot via Systemd

## 📝 1. OBJECTIVE (The "Why")
**Objective:** To change the default boot state (runlevel) of a Linux server to automatically load the Graphical User Interface (GUI) upon its next startup, without interrupting currently running services.
**Context:** The deployment of new tools on the Stratos Datacenter App Servers requires graphical access. By modifying the `systemd` default target from `multi-user.target` (CLI) to `graphical.target` (GUI), we prepare the servers for graphical access upon their next scheduled maintenance reboot, while ensuring zero downtime today.

---

## ⚙️ 2. TARGET INFRASTRUCTURE / PREREQUISITES (The "Where")
- **OS:** Linux (Systemd-based distributions like CentOS, RHEL, Ubuntu)
- **Target Systems:** App Server 1 (`stapp01`), App Server 2 (`stapp02`), App Server 3 (`stapp03`)
- **Required Tools:** `systemctl`, `ssh`
- **Prerequisites:**
  - Network access to the servers via SSH.
  - Sudo/administrative privileges on the target machines.
- **Relevant File Paths:**
  - `/etc/systemd/system/default.target` (The configuration symlink)

---

## 🛠️ 3. EXECUTION (The "How")

Perform the following steps sequentially for **each** target application server. 

### Step 1: Connect to the Target Server
```bash
ssh <username>@<server_hostname>
# Example: ssh steve@stapp02
```
* **What it does:** Opens a secure shell session into the target server.
* **Why:** You must be locally authenticated on the machine to modify its boot parameters.

### Step 2: Verify the Current Boot Target
```bash
sudo systemctl get-default
```
* **What it does:** Queries the systemd manager for the current default boot path.
* **Expected output:** `multi-user.target`

### Step 3: Set the New Graphical Target
```bash
sudo systemctl set-default graphical.target
```
* **What it does:** Removes the old symbolic link pointing to the CLI environment and creates a new one pointing to `/usr/lib/systemd/system/graphical.target`.
* **Why:** This leaves a "sticky note" for the operating system, explicitly instructing it to load the display manager (GUI) the next time the server is turned on.

> **CRITICAL WARNING:** Do not execute the `reboot` or `sudo systemctl isolate graphical.target` commands. The requirement strictly dictates not initiating a reboot. Altering the default target satisfies the requirement without immediate disruption.

### Step 4: Verify the Configuration Change
```bash
sudo systemctl get-default
```
* **What it does:** Confirms the symbolic link was updated successfully.
* **Expected output:** `graphical.target`

---

## ✅ 4. VERIFICATION & TROUBLESHOOTING (The "Proof")

**Verification:**
The execution is considered successful when `sudo systemctl get-default` returns `graphical.target` on all designated servers. No processes will visually change in your current SSH session.

**Troubleshooting:**

| Error / Issue | Meaning | Solution |
| :--- | :--- | :--- |
| `Could not resolve hostname stapp003` | DNS cannot find the server. | Check for typos in your SSH command. (e.g., typing `stapp003` instead of `stapp03`). |
| `Host key verification failed` | You answered "no" or typed a blank response when SSH asked to verify the new server fingerprint. | Re-run the SSH command and explicitly type `yes` when prompted to trust the ED25519 key. |
| `sudo: get-default: command not found` | Syntax error. You forgot the `systemctl` tool in the command. | Run the full command: `sudo systemctl get-default`. |
| `Permission denied` / Prompts for root password | The user does not have sudo privileges. | Escalate to the system administrator or use an authorized account (e.g., `steve`, `banner`). |
