# SOP: Advanced File Security via Access Control Lists (ACLs)

## 📝 Objective

To implement granular, micro-targeted file permissions. While standard Linux permissions (`chmod`) only allow for broad categorization (Owner, Group, Others), **Access Control Lists (ACLs)** allow Infrastructure Engineers to explicitly grant or deny permissions to specific, individual users or secondary groups without altering the file's primary ownership.

---

## ⚙️ Target Infrastructure

- **OS:** Linux (All modern distributions)
- **Core Binaries:** `setfacl` (Set File ACL), `getfacl` (Get File ACL)

---

## 🛠️ Execution Steps

### 1. The Syntax Architecture (`setfacl`)

The `setfacl` command is used to inject specific rules into a file's hidden access list.

**The Base Command:**

```bash
sudo setfacl -m  /path/to/file
```

- **`-m`:** Instructs the system to **Modify** the ACL (add or update a rule)

---

### Constructing the Rule

Rules are constructed using a colon-separated syntax:
`target_type:target_name:permissions`

**Components:**
- **Target Type:** `u` (user) or `g` (group)
- **Target Name:** The specific username (e.g., `garrett`)
- **Permissions:** `r` (read), `w` (write), `x` (execute), or `-` (deny all)

---

### 2. Execution Scenarios

#### Scenario A: Explicit Read Access

To grant a specific user (`garrett`) read-only access to a file, regardless of what the "Others" permission is set to:

```bash
sudo setfacl -m u:garrett:r-- /etc/hosts
```

---

#### Scenario B: Explicit Denial (Blacklisting)

To explicitly strip all permissions from a specific user (`ravi`), effectively blinding them to the file:

```bash
sudo setfacl -m u:ravi:--- /etc/hosts
```

---

#### Scenario C: Removing an ACL

If a user leaves the company or changes roles, you remove their specific ACL rule using the **`-x`** flag:

```bash
sudo setfacl -x u:garrett /etc/hosts
```

---

## ✅ Verification & Auditing

You **cannot** use the standard `ls -l` command to read ACLs. (If a file has an ACL applied, `ls -l` will just show a **`+`** symbol at the end of the permission block, like `-rw-r--r--+`).

To audit the exact rules applied to a file, you must use `getfacl`:

```bash
getfacl /etc/hosts
```

**Expected Output Breakdown:**
```text
# file: etc/hosts
# owner: root
# group: root
user::rw-
user:garrett:r--     <-- ACL Rule Active
user:ravi:---        <-- ACL Rule Active
group::r--
mask::r--
other::r--
```

---

## 📋 Quick Reference: Common ACL Operations

| Operation | Command | Description |
|-----------|---------|-------------|
| Grant read access to user | `setfacl -m u:username:r-- /path/file` | Allow specific user to read file |
| Grant read+write to user | `setfacl -m u:username:rw- /path/file` | Allow specific user to read and modify |
| Grant read+execute to user | `setfacl -m u:username:r-x /path/file` | Allow user to read and execute script |
| Deny all access to user | `setfacl -m u:username:--- /path/file` | Blacklist specific user |
| Grant access to group | `setfacl -m g:groupname:r-- /path/file` | Allow entire group read access |
| Remove user's ACL | `setfacl -x u:username /path/file` | Delete specific ACL entry |
| Remove all ACLs | `setfacl -b /path/file` | Strip all ACL rules from file |
| View file ACLs | `getfacl /path/file` | Display all active ACL rules |

---

## 🔍 Advanced ACL Features

### Recursive ACL Application

Apply ACLs to entire directory trees:

```bash
sudo setfacl -R -m u:garrett:r-x /var/www/html
```

**`-R`:** Recursively apply to all files and subdirectories

---

### Default ACLs for Directories

Set default permissions for **new files** created within a directory:

```bash
sudo setfacl -d -m u:garrett:rw- /shared/projects
```

**`-d`:** Default ACL (applies to future files created in this directory)

---

### Copying ACLs Between Files

Preserve ACLs when copying files:

```bash
getfacl /source/file | setfacl --set-file=- /destination/file
```

---

## 🔐 Security Best Practices

### 1. **Audit Before Modification**
Always check existing ACLs before making changes:
```bash
getfacl /path/file
```

### 2. **Document ACL Changes**
Maintain a log of ACL modifications for compliance and troubleshooting:
```bash
echo "$(date): Added read access for garrett to /etc/hosts" >> /var/log/acl-changes.log
```

### 3. **Use ACLs Sparingly**
ACLs add complexity. Use standard permissions (`chmod`) when possible and reserve ACLs for exceptions.

### 4. **Understand the Mask**
The **mask** limits the maximum effective permissions for named users and groups:
```bash
# Set maximum effective permissions
sudo setfacl -m m::r-- /path/file
```

### 5. **Backup ACLs**
Before major changes, backup ACLs:
```bash
getfacl -R /path/directory > acl-backup.txt
```

Restore if needed:
```bash
setfacl --restore=acl-backup.txt
```

---

## 🐛 Troubleshooting

### Problem: ACLs Not Working

**Check 1:** Verify filesystem supports ACLs
```bash
mount | grep acl
```

**Solution:** Remount with ACL support:
```bash
sudo mount -o remount,acl /
```

**Check 2:** Verify ACL packages are installed
```bash
# Debian/Ubuntu
sudo apt install acl

# RHEL/CentOS
sudo yum install acl
```

---

### Problem: User Still Cannot Access File Despite ACL

**Check the mask:**
```bash
getfacl /path/file
```

Look for the `mask::` line. If it's more restrictive than your ACL, it limits effective permissions.

**Fix the mask:**
```bash
sudo setfacl -m m::rwx /path/file
```

---

### Problem: `ls -l` Shows `+` But `getfacl` Shows Nothing

The file system may not have ACL support enabled, or ACLs were removed.

**Verify:**
```bash
getfacl /path/file
```

If output shows only standard permissions, the `+` indicator is stale. Remove it:
```bash
sudo setfacl -b /path/file  # Remove all ACLs
```

---

## 📊 ACL vs Standard Permissions

| Feature | Standard Permissions (`chmod`) | ACLs (`setfacl`) |
|---------|-------------------------------|------------------|
| **Granularity** | Owner, Group, Others (3 categories) | Unlimited individual users/groups |
| **Complexity** | Simple, easy to audit | Complex, requires `getfacl` |
| **Performance** | Faster | Slight overhead |
| **Use Case** | General permission management | Exception-based access control |
| **Portability** | Universal across all Unix/Linux | Requires filesystem support |

---

## 💡 Real-World Use Cases

### Use Case 1: Shared Development Server
Multiple developers need access to different project directories without being in the same group:

```bash
# Grant Alice read access to Bob's project
sudo setfacl -m u:alice:r-x /home/bob/project

# Grant Bob read access to Alice's project
sudo setfacl -m u:bob:r-x /home/alice/project
```

---

### Use Case 2: Audit Log Protection
Allow security team to read logs, but prevent everyone else (including the application owner) from modifying them:

```bash
# Grant security team read access
sudo setfacl -m g:security:r-- /var/log/app.log

# Prevent app owner from deleting logs
sudo setfacl -m u:appuser:r-- /var/log/app.log
```

---

### Use Case 3: Temporary Contractor Access
Grant temporary access to a contractor that automatically expires:

```bash
# Grant access
sudo setfacl -m u:contractor:rwx /project/data

# Revoke when contract ends
sudo setfacl -x u:contractor /project/data
```

---

## 📚 Additional Resources

- **Man Pages:** `man setfacl`, `man getfacl`
- **Filesystem Check:** `tune2fs -l /dev/sda1 | grep "Default mount options"`
- **ACL Documentation:** `/usr/share/doc/acl/`
