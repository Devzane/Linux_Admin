# SOP: Machine-to-Machine (M2M) Passwordless SSH Trust

## 📝 Objective

To establish secure, automated communication channels between infrastructure components.

In a CI/CD or automated DevOps environment, central orchestrators (like Ansible, Jenkins, or custom cron scripts on a Jump Host) must execute commands across hundreds of remote servers. Relying on password-based authentication breaks automation because it requires interactive human input. This operation provisions a public/private key pair with no passphrase, distributing the public "lock" to remote servers to allow seamless, cryptographically verified script execution.

---

## ⚙️ Target Infrastructure

- **Source Machine:** The Orchestrator / Jump Host (User: `thor`)
- **Target Machines:** Fleet Application Servers (`stapp01`, `stapp02`, `stapp03`)
- **Core Binaries:** `ssh-keygen`, `ssh-copy-id`

---

## 🏗️ The Architectural Framework

This process requires two distinct phases:

1. **Key Generation:** Creating the cryptographic math on the source machine.
2. **Trust Distribution:** Pushing the Public Key over the network into the remote server's `~/.ssh/authorized_keys` file. The `ssh-copy-id` tool automates this injection, preventing manual copy-paste errors.

---

## 🛠️ Execution Steps

### Phase 1: Generate the Automation Key Pair

Ensure you are logged into the Jump Host as the correct user (in this scenario, `thor`). Do not do this as `root` unless the script runs as root.

Run the key generator:

```bash
ssh-keygen -t rsa
```

> **⚠️ The Automation Trap (CRITICAL):**
> The prompt will ask you three questions.
> 1. **File to save the key:** Press `Enter` to accept the default (`/home/thor/.ssh/id_rsa`).
> 2. **Enter passphrase:** Press `Enter` to leave it completely empty!
> 3. **Enter same passphrase again:** Press `Enter` again.
>
> **If you type a password here, your automated scripts will fail.**

---

### Phase 2: Distribute the Trust (Public Key)

You must now push your newly created public key to the target servers using `ssh-copy-id`.

*Note: You will be prompted for the remote user's password exactly once. This is required to gain initial access to inject the key.*

#### Target 1: App Server 1 (Tony)

```bash
ssh-copy-id tony@stapp01
```
*(Enter Tony's password when prompted: `Ir0nM@n`)*

#### Target 2: App Server 2 (Steve)

```bash
ssh-copy-id steve@stapp02
```
*(Enter Steve's password when prompted: `Am3ric@`)*

#### Target 3: App Server 3 (Banner)

```bash
ssh-copy-id banner@stapp03
```
*(Enter Banner's password when prompted: `BigGr33n`)*

---

## ✅ Verification & Auditing (The SRE Proof)

Do not assume the keys were injected correctly. You must verify that the trust relationship is established by attempting to log in. **If you are prompted for a password during this test, the operation failed.**

Run these three commands. You should instantly drop into the remote shell. Type `exit` immediately to return to the Jump Host after each test.

```bash
# Test Server 1
ssh tony@stapp01
exit

# Test Server 2
ssh steve@stapp02
exit

# Test Server 3
ssh banner@stapp03
exit
```

---

## 📋 Quick Reference: KodeKloud Auth Matrix

If you forget the default passwords required during the `ssh-copy-id` injection phase:

| Server | User | Password |
|--------|------|----------|
| `stapp01` | `tony` | `Ir0nM@n` |
| `stapp02` | `steve` | `Am3ric@` |
| `stapp03` | `banner` | `BigGr33n` |

---

## 🐛 Troubleshooting & Edge Cases

### Problem 1: `ssh-copy-id: command not found`

- **Root Cause:** Extremely minimal Linux distributions might not have this helper script installed.
- **Resolution:** You must inject the key manually using standard pipes:
  ```bash
  cat ~/.ssh/id_rsa.pub | ssh tony@stapp01 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
  ```

### Problem 2: The system still asks for a password after `ssh-copy-id` was successful.

- **Root Cause:** The remote server has strict file permissions. If the remote `~/.ssh` directory or the `authorized_keys` file has permissions that are too open (e.g., `777`), the remote SSH daemon will actively reject the key for security reasons.
- **Resolution:** SSH into the remote server with the password, and fix the directory permissions:
  ```bash
  chmod 700 ~/.ssh
  chmod 600 ~/.ssh/authorized_keys
  ```
