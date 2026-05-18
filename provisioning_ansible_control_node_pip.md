# SOP: Provisioning an Ansible Control Node via Python PIP

## 📝 Objective

To transform a standard Linux Jump Host into an Ansible Control Node.

In Infrastructure as Code (IaC) environments, Ansible is utilized for fleet-wide configuration management. This operation provisions the core Ansible engine. Because the deployment requires strict version parity (v4.10.0) to match existing infrastructure playbooks, the installation must bypass the OS-level package manager (yum/apt) and utilize Python's pip3 to guarantee exact semantic versioning and global availability.

## ⚙️ Target Infrastructure

- **Role**: Ansible Control Node (The Orchestrator)
- **Location**: Jump Host
- **Core Binaries**: `python3`, `pip3`, `ansible`
- **Architecture Constraints**: Agentless (Requires only standard SSH on target nodes).

## 🏗️ The Architectural Framework: pip3 vs OS Package Managers

Why use `pip3` instead of `sudo yum install ansible`?

- **Version Pinning**: `yum` and `apt` pull whatever version of Ansible the OS repository deems "stable" (which might be months or years out of date). `pip3` interacts directly with the Python Package Index (PyPI), allowing SREs to pin the exact release required (`==4.10.0`) to prevent playbook syntax deprecation errors.
- **Global Namespace**: By default, running `pip3 install <package>` as a standard user installs the binary into that specific user's `~/.local/bin` directory. To satisfy the requirement that all users on the system can execute Ansible, the installation must be elevated to root so the binary is placed in a globally accessible `$PATH` (like `/usr/local/bin`).

## 🛠️ Execution Steps

### Step 1: Verify Python Environment Dependencies

Ansible is written in Python. Before installation, verify the Jump Host has Python 3 and PIP installed.

```bash
python3 --version
pip3 --version
```

### Step 2: Global Installation via Elevated PIP

Execute the installation using `sudo` to inject the Ansible binary into the global binary path, explicitly declaring the required version.

```bash
sudo pip3 install ansible==4.10.0
```

*Note: In some modern Python environments, running PIP as root is discouraged in favor of Virtual Environments (venvs). However, for a dedicated system-wide CI/CD Jump Host where all users require access, an elevated global install is the standard approach.*

### Step 3: Validate Global Availability & Version Parity

Do not assume the installation succeeded. Verify the binary is globally registered in the system path and matches the exact version required.

```bash
# Check the binary location (Should return a global path like /usr/local/bin/ansible)
which ansible

# Verify the strict version constraint
ansible --version
```

## 🚀 Advanced SRE Operations

### 1. The Ansible Configuration File (ansible.cfg)

Once the binary is installed, Ansible looks for its configuration file in a strict hierarchical order:

1. `$ANSIBLE_CONFIG` (Environment variable)
2. `./ansible.cfg` (Current working directory - SRE Preferred)
3. `~/.ansible.cfg` (User home directory)
4. `/etc/ansible/ansible.cfg` (Global default)

*Best Practice: Always keep an `ansible.cfg` file in the same Git repository as your playbooks so the configuration travels with the code.*

### 2. Python Virtual Environments (The Enterprise Alternative)

If you operate on a shared server where different teams need different versions of Ansible (e.g., Team A needs v4.10.0, Team B needs v6.0.0), a global sudo install causes a version conflict. Instead, SREs provision isolated Python Virtual Environments:

```bash
python3 -m venv ~/ansible_v4_env
source ~/ansible_v4_env/bin/activate
pip3 install ansible==4.10.0
```

## 🐛 Troubleshooting & Edge Cases

### Problem 1: ansible: command not found after successful installation.

- **Root Cause**: The directory where PIP placed the binary (usually `/usr/local/bin` or `/usr/bin`) is not in your shell's `$PATH` environment variable.
- **Resolution**: Find where PIP placed it (`find / -name ansible 2>/dev/null`) and add that directory to the global path: `export PATH=$PATH:/usr/local/bin`.

### Problem 2: error: externally-managed-environment during sudo pip3 install.

- **Root Cause**: Introduced in PEP 668, newer Linux distributions (like Ubuntu 24.04 or Debian 12) block `sudo pip3` by default to prevent PIP from overwriting Python packages required by the core OS.
- **Resolution**: If you absolutely must install globally on a modern OS, append the override flag:

```bash
sudo pip3 install ansible==4.10.0 --break-system-packages
```

*(Note: This flag will likely not be required in the CentOS/RHEL 9 environment used in KodeKloud, but is critical for future reference).*
