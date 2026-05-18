# SOP: Linux Package & Dependency Management

## 📝 Objective

To standardize the provisioning, updating, and removal of software binaries and dependencies across Linux infrastructure.

In Site Reliability Engineering (SRE), software installation must be idempotent, reproducible, and automated. Relying on manual downloads or unpinned software versions leads to configuration drift, where `stapp01` runs a different version of a service than `stapp02`, causing unpredictable production outages.

## ⚙️ Target Infrastructure

- **Debian-based (Ubuntu, Debian)**: `apt`, `apt-get`, `dpkg`
- **RedHat-based (RHEL, CentOS, Rocky)**: `dnf`, `yum`, `rpm`
- **Language-Specific Environments**: Python (`pip`), Node.js (`npm`)

## 🏗️ The Architectural Framework

Linux does not use standard `.exe` installers. Software is distributed as compressed archives containing binaries, configuration files, and metadata.

There is a strict two-tier hierarchy to Linux software management:

1. **Low-Level Package Managers (`dpkg` / `rpm`)**: These interact directly with the local hard drive to unpack and install a physical file (`.deb` or `.rpm`). They do not understand the internet and cannot download missing dependencies.
2. **High-Level Package Managers (`apt` / `dnf` / `yum`)**: These are front-end wrappers. They connect to remote web servers (Repositories), download the software, calculate exactly which dependencies are missing, download those too, and then hand everything over to the low-level manager to install.

## 🛠️ Core Execution Scenarios

### Scenario A: Automated Installation (The CI/CD Standard)

When writing bash scripts or automated pipelines, you must suppress interactive human prompts (like "Are you sure? [y/N]").

**Debian/Ubuntu**:
*(Note: `apt-get` is preferred over `apt` in scripts because its output format is stable and guaranteed not to change).*

```bash
# 1. Always sync the local database with the remote repository first
sudo apt-get update -y

# 2. Install the package without human intervention
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y nginx
```

**RHEL/CentOS**:

```bash
sudo yum install -y nginx
# or on newer systems:
sudo dnf install -y nginx
```

### Scenario B: Version Pinning (Strict Parity)

SREs never just install "nginx". They install the exact version that was tested in the staging environment.

```bash
# Ubuntu/Debian syntax
sudo apt-get install -y nginx=1.18.0-0ubuntu1

# CentOS/RHEL syntax
sudo yum install -y nginx-1.18.0
```

### Scenario C: Offline / Direct File Installation

If a vendor provides a custom `.deb` or `.rpm` file that is not in a public repository, you must bypass apt/yum and use the low-level installer.

```bash
# Debian/Ubuntu (Install file)
sudo dpkg -i custom-app_1.0_amd64.deb

# RHEL/CentOS (Install file)
sudo rpm -ivh custom-app-1.0.x86_64.rpm
```

## 🚀 Advanced SRE Operations

### 1. Repository Management (Adding new sources)

Often, the default OS repositories contain outdated software. To get the newest tools (like HashiCorp Terraform or modern Ansible), you must add the vendor's cryptographic key and repository URL to your system.

Example: Adding EPEL (Extra Packages for Enterprise Linux) on CentOS:

```bash
sudo yum install -y epel-release
```

### 2. Ephemeral Storage Cleanup (Docker/Containers)

When building cloud images or Docker containers, leftover downloaded package caches bloat the image size and cost money to store. Always clean up in the same execution layer.

```bash
# Debian/Ubuntu cleanup
sudo apt-get clean && sudo rm -rf /var/lib/apt/lists/*

# CentOS/RHEL cleanup
sudo yum clean all && sudo rm -rf /var/cache/yum
```

### 3. Finding Which Package Owns a File

If you find a random configuration file (e.g., `/etc/httpd/conf/httpd.conf`) and need to know exactly which software package placed it there:

- **Debian**: `dpkg -S /etc/httpd/conf/httpd.conf`
- **RHEL**: `rpm -qf /etc/httpd/conf/httpd.conf`

## 📋 Quick Reference: Command Translation Matrix

| Action | Debian/Ubuntu (`apt-get`) | RHEL/CentOS (`yum` / `dnf`) |
| :--- | :--- | :--- |
| Refresh database | `apt-get update` | `yum check-update` |
| Install package | `apt-get install <pkg>` | `yum install <pkg>` |
| Remove package | `apt-get remove <pkg>` | `yum remove <pkg>` |
| Remove + purge configs | `apt-get purge <pkg>` | `yum remove <pkg>` *(Deletes configs by default)* |
| Search for package | `apt-cache search <pkg>` | `yum search <pkg>` |
| Upgrade all OS apps | `apt-get upgrade` | `yum upgrade` |

## 🐛 Troubleshooting & Edge Cases

### Problem 1: Could not get lock /var/lib/dpkg/lock-frontend (Ubuntu/Debian)

- **Root Cause**: Another software installation is currently running in the background (often the OS's automatic background updater), locking the database to prevent corruption.
- **Resolution**: 
  1. Find the rogue process: `lsof /var/lib/dpkg/lock-frontend`
  2. Wait for it to finish. If it is permanently hung, kill the PID and reconfigure the package manager: `sudo kill -9 <PID>` followed by `sudo dpkg --configure -a`.

### Problem 2: error: externally-managed-environment (pip3)

- **Root Cause**: Introduced in modern Linux (PEP 668), the OS blocks `sudo pip3 install` to prevent Python modules from overwriting system packages managed by `apt` or `dnf`.
- **Resolution**: The SRE standard is to create an isolated Virtual Environment (`python3 -m venv app_env`). If global execution is absolutely mandatory for a system-wide CI/CD tool (like Ansible on a Jump Host), append `--break-system-packages`.

### Problem 3: GPG Key or Signature Verification Failed

- **Root Cause**: The cryptographic signature on the remote repository has expired or changed, meaning the system can no longer mathematically verify the software hasn't been tampered with.
- **Resolution**: You must fetch the updated GPG key from the vendor and update your trusted keys list.
