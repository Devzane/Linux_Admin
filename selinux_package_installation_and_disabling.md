# SOP: SELinux Package Installation & Permanent Disabling

## 1️⃣ The Objective (The "Why")

Security-Enhanced Linux (SELinux) is a powerful kernel-level access control mechanism. However, during specific application testing or when migrating legacy applications, SELinux policies can aggressively block necessary file executions or network ports. The objective of this operation is to provision the required SELinux management utilities on a bare-metal/virtual server and permanently disable the SELinux kernel module to prevent it from interfering with application deployments upon the next system boot.

## 2️⃣ Target Infrastructure (The "Where")

- **OS:** RedHat-based Linux Distributions (CentOS, RHEL, Fedora, Rocky Linux)
- **Component:** Linux Kernel (SELinux Module)
- **Package Manager:** `yum` / `dnf`

## 3️⃣ Execution (The "How")

### Step 1: Install Core SELinux Utilities

Unlike Ubuntu/Debian which uses `apt`, RedHat-based systems use `yum` or `dnf`. Furthermore, SELinux utilities are not bundled into a single package named "selinux". You must install the specific policy and utility packages.

```bash
sudo yum install policycoreutils selinux-policy-targeted
```

### Step 2: Permanently Disable SELinux

Because SELinux operates inside the kernel (not as a standard background service), you cannot simply use `systemctl stop selinux`. You must instruct the bootloader not to load it into memory during the next startup.

Open the core configuration file:

```bash
sudo vi /etc/selinux/config
```

Locate the `SELINUX=` directive and modify it to `disabled`:

```ini
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
```

Save and exit the file (`:wq`).

*(Note: The system requires a scheduled reboot for this kernel-level change to take physical effect. No immediate runtime changes are made in this step).*

## 4️⃣ Verification & Troubleshooting (The "Proof")

### Verification

Since a reboot is required to fully disable the module, you cannot immediately run `sestatus` to verify the disabled state. Instead, you must verify that the configuration file was written correctly.

```bash
cat /etc/selinux/config | grep SELINUX=
```

**Expected Output:** `SELINUX=disabled`

### Troubleshooting

- **Problem:** `yum install SELinux` returns `Error: Unable to find a match: SELinux.`
  - **Solution:** Linux package managers are strictly case-sensitive. Additionally, the meta-package name is not "selinux". You must use the exact lowercase utility names: `policycoreutils` and `selinux-policy-targeted`.

- **Problem:** You need SELinux disabled immediately without waiting for the scheduled maintenance reboot.
  - **Solution:** Use the runtime toggle command `sudo setenforce 0` to temporarily place SELinux into "Permissive" mode until the server can be rebooted.
