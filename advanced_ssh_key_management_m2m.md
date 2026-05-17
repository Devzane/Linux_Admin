# SOP: Advanced SSH Key Management & Machine-to-Machine (M2M) Authentication

## 📝 Objective

To establish secure, passwordless communication channels between infrastructure components.

In enterprise DevOps environments, orchestrators (like Ansible, Jenkins, or cron jobs) must execute commands across hundreds of remote servers. Relying on password-based authentication breaks automation and poses a security risk. This SOP covers the generation, distribution, and advanced configuration of Asymmetric SSH Keys to enable cryptographically verified, zero-touch authentication.

## ⚙️ Target Infrastructure

- **Protocols**: SSH (Secure Shell) via TCP Port 22
- **Core Binaries**: `ssh-keygen`, `ssh-copy-id`, `ssh-agent`, `ssh-add`
- **Configuration Files**: `~/.ssh/config`, `~/.ssh/authorized_keys`, `~/.ssh/known_hosts`

## 🏗️ The Architectural Framework

SSH Key authentication relies on two mathematically linked files:

1. **The Private Key** (`id_ed25519` / `id_rsa`): Never leaves the source machine. It is the physical "key" used to prove identity.
2. **The Public Key** (`id_ed25519.pub` / `id_rsa.pub`): The "lock". It is distributed to any target server you wish to access and appended to that server's `authorized_keys` file.

When a connection is initiated, the target server challenges the source machine to solve a cryptographic puzzle that can only be solved if the source possesses the matching Private Key.

## 🛠️ Core Execution Scenarios

### Scenario A: Generating the Key Pair

Modern SRE standard dictates the use of `ed25519` over `rsa` due to its superior performance and smaller footprint, unless connecting to legacy systems.

```bash
ssh-keygen -t ed25519 -C "automation_bot_jump_host"
```

**The Automation Trap (Passphrases)**:
- **For Human Users**: Always enter a secure passphrase.
- **For Machine/Script Users**: You MUST leave the passphrase blank (press Enter twice). If a key has a passphrase, background scripts will permanently hang waiting for human input.

### Scenario B: Distributing the Trust (The Standard Way)

The `ssh-copy-id` utility automatically handles the correct permissions and appends the public key to the remote server without overwriting existing keys.

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@target_server_ip
```

*(You will be prompted for the target user's password exactly once to authorize the injection).*

### Scenario C: Distributing the Trust (The Manual Way)

If `ssh-copy-id` is not installed on your minimal OS, you must inject it manually via standard pipes:

```bash
cat ~/.ssh/id_ed25519.pub | ssh user@target_server_ip "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

## 🚀 Advanced SRE Operations

### 1. The SSH Config File (`~/.ssh/config`)

Instead of typing `ssh -i /path/to/key user@10.0.0.55 -p 2222` every time, SREs use a configuration file to create aliases.

**Edit/Create `~/.ssh/config`**:

```ssh-config
Host web-prod-01
    HostName 10.0.0.55
    User deploy_admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_prod
```

**Usage**: You can now simply type `ssh web-prod-01` and SSH handles the rest.

### 2. SSH Agent Forwarding (`ssh-agent`)

If you need to SSH into Server A, and from Server A, SSH into Server B, you should never copy your private key to Server A. Instead, you forward your local agent.

```bash
# Start the agent and add your key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# SSH into Server A with Agent Forwarding (-A)
ssh -A user@Server_A
# Now, from Server A, you can SSH to Server B using your laptop's key!
```

### 3. ProxyJump / Bastion Hosts

To seamlessly route an SSH connection through a Jump Host directly to a private internal server:

```bash
ssh -J jump_user@jump_host internal_user@private_server
```

## 🔐 Strict Security & Permission Requirements

The SSH daemon (`sshd`) is notoriously strict. If your file permissions are too open, it will silently ignore your keys and fallback to password authentication.

**Mandatory Local Permissions**:
- `~/.ssh/` directory: `700` (`drwx------`)
- Private Key (`id_rsa`, `id_ed25519`): `600` (`-rw-------`)
- Public Key (`.pub`): `644` (`-rw-r--r--`)
- `known_hosts` file: `644` (`-rw-r--r--`)

**Mandatory Remote Permissions**:
- `~/.ssh/authorized_keys` file: `600` (`-rw-------`)

## 📋 Quick Reference: Command Matrix

| Operation | Command | Description |
| :--- | :--- | :--- |
| Generate (Modern) | `ssh-keygen -t ed25519` | Creates high-security elliptical curve key |
| Generate (Legacy) | `ssh-keygen -t rsa -b 4096` | Creates highly compatible RSA key |
| Distribute | `ssh-copy-id user@host` | Safely injects public key to remote host |
| Test Auth | `ssh -v user@host` | Verbose mode. Shows exactly which keys are offered |
| Load Agent | `ssh-add ~/.ssh/key` | Loads private key into memory |

## 🐛 Troubleshooting & Edge Cases

**Problem 1: Still being asked for a password after `ssh-copy-id`.**
- **Check 1 (Permissions)**: Run `ssh user@host "ls -ld ~/.ssh && ls -l ~/.ssh/authorized_keys"`. If they are not `700` and `600` respectively, fix them.
- **Check 2 (SELinux)**: On RHEL/CentOS, SELinux might be blocking read access to the keys. Run `restorecon -Rv ~/.ssh` on the target server.
- **Check 3 (Daemon Config)**: Ensure the remote server's `/etc/ssh/sshd_config` has `PubkeyAuthentication yes` enabled.

**Problem 2: `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`**
- **Root Cause**: The target server was rebuilt or its IP was reassigned, meaning its unique host fingerprint changed. This triggers a Man-In-The-Middle warning.
- **Resolution**: Remove the stale fingerprint from your local known hosts file:
  ```bash
  ssh-keygen -R <target_ip_or_hostname>
  ```

## 💡 Use Case: KodeKloud Jump Host Automation

To configure the `thor` jump host to automate tasks across the Stratos Datacenter fleet:

```bash
ssh thor@jump_host
ssh-keygen -t rsa # (Press Enter 3 times to skip the passphrase)
ssh-copy-id tony@stapp01
ssh-copy-id steve@stapp02
ssh-copy-id banner@stapp03
```
