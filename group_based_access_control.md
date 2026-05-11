# SOP: Group-Based Access Control & Identity Management

## 📝 Objective

To implement scalable access management across a fleet of servers by creating standardized permission groups and securely appending users to them without destructive overwrites.

## ⚙️ Target Infrastructure

* **OS:** Linux
* **Component:** System Identity Management (`/etc/group`, `/etc/passwd`)

## 🛠️ Execution Steps

### 1. Create the Global Target Group

Create the standard permission group on the target machine. Use the low-level `groupadd` binary for reliable automation.

**Example Scenario:** Create a group named `nautilus_noc`.

```bash
sudo groupadd nautilus_noc
```

### 2. Conditional User Provisioning

Before assigning a user, you must verify their current state on the system to avoid execution errors or destructive overwrites.

#### Step 2A: Check System State

```bash
id kano
```

#### Step 2B: State A (User Does Not Exist)
If the system returns "no such user", provision the user and assign the supplementary group simultaneously.

```bash
sudo useradd -G nautilus_noc kano
```

*(Note: Use `-G` for supplementary groups, not `-g`, to preserve their default primary workspace).*

#### Step 2C: State B (User Already Exists)
If the user exists, you must append the new group to their existing permissions.

```bash
sudo usermod -aG nautilus_noc kano
```

> [!WARNING]
> **CRITICAL WARNING:** You must include the `-a` (append) flag alongside `-G`. Running `usermod -G` without `-a` will silently delete the user from all their other groups, potentially locking them out of the system.

## ✅ Verification

Verify the user was successfully added to the target group:

```bash
id kano
```

**Expected output:** Should list `nautilus_noc` among the user's assigned groups.
