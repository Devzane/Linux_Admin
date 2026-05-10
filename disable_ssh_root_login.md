# SOP: Disabling Direct SSH Root Login

## 📝 Objective

To enhance server security by disabling direct root user access via SSH.

Direct root login is a major security vulnerability. By disabling it, attackers cannot simply brute-force the root password. Instead, engineers must log in using their personal, unprivileged accounts and escalate privileges using `sudo`. This limits the blast radius of compromised credentials and creates a verifiable audit trail of who executed administrative commands.

## ⚙️ Target Infrastructure

- **OS:** Linux (RHEL/CentOS, Debian/Ubuntu)
- **Service:** OpenSSH Server Daemon (`sshd`)

## 🛠️ Execution Steps

### 1. Connect to the Target Server

SSH into the target application server using your standard user account.

```bash
ssh <username>@<server-hostname-or-ip>
```

### 2. Locate and Edit the SSH Server Configuration

The configuration file that controls the SSH daemon is located at `/etc/ssh/sshd_config`.

*(Note: Do not confuse this with `ssh_config`, which configures outbound client connections.)*

Open the file using a text editor with sudo privileges:

```bash
sudo vi /etc/ssh/sshd_config
```

*(Or use nano if preferred: `sudo nano /etc/ssh/sshd_config`)*

### 3. Modify the PermitRootLogin Directive

Locate the `PermitRootLogin` parameter within the file.

- If it is set to `yes`, change it to `no`.
- If it is commented out (starts with a `#`), remove the `#` and ensure it is set to `no`.

**Before:**

```text
#PermitRootLogin yes
```

**After:**

```text
PermitRootLogin no
```

Save the file and exit the text editor.

### 4. Verify Configuration Syntax (SRE Best Practice)

Before restarting the service, always test the configuration file for syntax errors to ensure you don't accidentally lock everyone out of the server.

```bash
sudo sshd -t
```

If the command returns no output, the syntax is perfectly valid.

### 5. Restart the SSH Daemon

The running `sshd` process must be restarted to read the updated configuration file from disk. Restarting the service will not drop your current active SSH session.

**For RHEL/CentOS/Amazon Linux:**

```bash
sudo systemctl restart sshd
```

**For Debian/Ubuntu:**

```bash
sudo systemctl restart ssh
```

## ✅ Verification

Attempt to SSH directly into the server as the root user from your local machine:

```bash
ssh root@<server-hostname-or-ip>
```

The connection should be actively refused or prompt for a password that will never be accepted, confirming the security boundary is active.
