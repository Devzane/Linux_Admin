# SOP: Managing Linux File Execution Permissions

## 📝 Objective

To properly assign execution permissions to automation scripts (Bash, Python, etc.) deployed across application servers.

In Linux, a file is not automatically executable just because it contains code or ends in `.sh` or `.py`. The operating system relies on explicit "Permission Bits" to determine if a file is allowed to be run as a program.

**Critical Architecture Rule:** A text-based script (like Bash or Python) cannot be executed if it cannot be read. The system's interpreter must be able to open and read the file's contents to execute its commands. Therefore, Read (`r`) and Execute (`x`) permissions must always be paired for scripts.

## ⚙️ Target Infrastructure

*   **OS:** Linux (All Distributions)
*   **Core Binary:** `chmod` (Change Mode)

## 🛠️ Execution Steps

### 1. Connect to the Target Server

SSH into the specific application server where the script resides.

```bash
ssh <username>@<server-hostname-or-ip>
```

### 2. Evaluate the Current Permission State

Before modifying a file, always check its current permission bits to understand the baseline.

```bash
ls -l /tmp/xfusioncorp.sh
```

**Output Example:**
```text
-rw-r--r-- 1 root root 1024 May 12 10:00 /tmp/xfusioncorp.sh
```
*(Notice the absence of the 'x' character. This file is strictly read/write).*

### 3. Apply the Execution Permission

Use the `chmod` command to modify the permission bits. You can use either Symbolic (letters) or Numerical/Octal (numbers) syntax.

#### Method A: Symbolic Permissions (Relative Change)

Use this to add or subtract permissions relative to what already exists.

*   `u` = User (Owner)
*   `g` = Group
*   `o` = Others
*   `a` = All

**Scenario 1: Strict Isolation (Owner Only)**

```bash
sudo chmod u+x /tmp/xfusioncorp.sh
```

**Scenario 2: Global Execution (System-Wide)**
To ensure all users can both read and execute the file (without stripping the owner's existing write permission):

```bash
sudo chmod a+rx /tmp/xfusioncorp.sh
```

#### Method B: Numerical/Octal Permissions (Absolute Change - SRE Preferred)

Senior engineers prefer numerical permissions because they define the exact, absolute state of the file in a single command, avoiding unpredictable relative changes.

*   `4` = Read (`r`)
*   `2` = Write (`w`)
*   `1` = Execute (`x`)

**The SRE Standard for Global Scripts: 755**

```bash
sudo chmod 755 /tmp/xfusioncorp.sh
```

**Permission Breakdown:**

*   **7 (Owner):** 4+2+1 = `rwx` (Owner retains full control to read, write/edit, and execute).
*   **5 (Group):** 4+1 = `r-x` (Group can read and execute, but cannot overwrite).
*   **5 (Others):** 4+1 = `r-x` (Everyone else can read and execute, but cannot overwrite).

> ⚠️ **Warning:** Avoid using `555`. Setting a script to `555` removes the Write (`w`) permission from everyone, meaning even the root owner will be denied permission to edit their own script later!

## ✅ Verification

Never assume a permission change was successful. Verify the new state.

```bash
ls -l /tmp/xfusioncorp.sh
```

**Expected Output for 755 or a+rx:**
```text
-rwxr-xr-x 1 root root 1024 May 12 10:00 /tmp/xfusioncorp.sh
```

**How to read the result (`-rwxr-xr-x`):**

*   **Block 1 (`rwx`):** The Owner can Read, Write, and eXecute.
*   **Block 2 (`r-x`):** The Group can Read and eXecute (but not overwrite).
*   **Block 3 (`r-x`):** All Others can Read and eXecute (but not overwrite).

The presence of the `r` and `x` in the target blocks confirms the script is now fully operational.

## 📚 Quick Reference Table

| Permission | Octal | Symbolic | Description |
| :--- | :--- | :--- | :--- |
| **Owner full control, others read/execute** | `755` | `u=rwx,go=rx` | SRE Standard for shared scripts |
| **Owner only** | `700` | `u=rwx,go=` | Private automation scripts |
| **Everyone full access** | `777` | `a=rwx` | ⚠️ Security risk - avoid in production |
| **Read-only for all** | `444` | `a=r` | Configuration files, logs |

## 🔐 Security Best Practices

*   **Principle of Least Privilege:** Only grant execute permissions to users who genuinely need to run the script.
*   **Never use 777 in production environments.** This allows any user on the system to inject malicious code into your scripts.
*   **Audit regularly:** Use `find /path -perm 777` to locate overly permissive files on your servers.
*   **Use `sudo` judiciously:** Only modify system-wide scripts with elevated privileges when absolutely necessary.

## 🐛 Troubleshooting

**Problem:** Script still shows "Permission denied" after running `chmod 755`.

**Solutions:**

1.  **Filesystem Constraints:** Check if the filesystem was mounted with the `noexec` flag (often done on `/tmp` for security).
    ```bash
    mount | grep noexec
    ```
2.  **Interpreter Issues:** Verify you're executing with the correct interpreter.
    ```bash
    bash /tmp/xfusioncorp.sh  # Explicit interpreter bypasses some shell restrictions
    ```
3.  **SELinux (RHEL/CentOS):** Security-Enhanced Linux might be blocking execution regardless of file permissions. Check the contexts:
    ```bash
    ls -Z /tmp/xfusioncorp.sh
    ```
