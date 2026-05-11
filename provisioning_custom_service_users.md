# SOP: Provisioning Custom Service Users for Application Isolation

## 📝 Objective

To enhance web application security by provisioning custom service users with explicitly defined User IDs (UIDs) and specific home directories mapping to the application's web root. This limits the "blast radius" if the web application is compromised.

## ⚙️ Target Infrastructure

* **OS:** Linux
* **Use Case:** Apache / Nginx / Custom Web Applications

## 🛠️ Execution Steps

### 1. Provision the User

Use the `useradd` binary to create the user. We must bypass the system defaults to enforce security requirements.

**Example Scenario:** Create a user `john` with UID `1714` mapped to `/var/www/john`.

```bash
sudo useradd -u 1714 -d /var/www/john -m john
```

**Flag Breakdown:**

* `-u 1714`: Hardcodes the User ID (UID) across the cluster to prevent permission mismatches.
* `-d /var/www/john`: Overrides the default `/home/username` path, locking the user directly into their application directory.
* `-m`: Instructs the OS to physically construct the home directory on the filesystem. (Without this, the directory will not exist, breaking the application).

### 2. Retrofitting an Existing User (If necessary)

If the user was accidentally created with default settings, use `usermod` to mutate their existing record into the correct state:

```bash
sudo usermod -u 1714 -d /var/www/john -m john
```

## ✅ Verification

### 1. Verify User ID:

```bash
id john
```

**Expected output:** `uid=1714(john) gid=1714(john) groups=1714(john)`

### 2. Verify Directory Creation & Permissions:

```bash
ls -ld /var/www/john
```

Ensure the directory exists and is owned by the designated user.
