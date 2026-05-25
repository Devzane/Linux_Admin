# SOP: Automating Web Content Archives via Bash & SCP

## 📝 1. OBJECTIVE (The "Why")
**Objective:** To automate the creation of compressed backups of website data and securely transfer them to a remote storage server without human intervention.
**Context:** xFusionCorp requires regular, automated backups of static website content. By scripting the archive creation (`zip`) and remote transfer (`scp`) via Headless Machine-to-Machine Authentication (SSH Keys), the DevOps team ensures data redundancy without the script freezing to wait for a human password prompt. 

---

## ⚙️ 2. TARGET INFRASTRUCTURE / PREREQUISITES (The "Where")
- **OS:** Linux (App Server 1 & Nautilus Storage Server)
- **Required Tools:** `zip`, `scp`, `ssh-keygen`
- **Assumptions:** 
  - You are logged into **App Server 1** as the application user (e.g., `tony`).
  - You know the credentials for the **Nautilus Storage Server** user (`natasha` on `ststor01`).
- **Relevant Paths:**
  - Script location: `/scripts/news_archive.sh`
  - Source data: `/var/www/html/news`
  - Local backup destination: `/archives/`
  - Remote backup destination: `/archives/` (on Storage Server)

---

## 🛠️ 3. EXECUTION (The "How")

### Step 1: Install the Compression Utility
> **Note:** This command must be run with administrative privileges *before* creating the script, as the script itself cannot contain `sudo`.

```bash
sudo yum install zip -y   # For RHEL/CentOS systems
# OR
sudo apt install zip -y   # For Debian/Ubuntu systems
```
**What it does:** Installs the `zip` utility required for compressing the website files.
**Why:** The standard Linux installation does not always include `zip` by default.

### Step 2: Generate the Cryptographic Keys (The "Private Key")
Run this as the standard user (e.g., `tony`), **NOT** as root.
```bash
ssh-keygen -t rsa -b 4096
```
**What it does:** Creates a public/private cryptographic key pair.
**Why:** This replaces the need for a password.
> **CRITICAL GOTCHA:** When prompted for a passphrase, you **MUST press ENTER** to leave it blank. If you set a passphrase, the automated script will pause and ask for it at runtime, defeating the purpose of automation.

### Step 3: Install the Public Key (The "Padlock") on the Destination Server
```bash
ssh-copy-id natasha@ststor01
```
**What it does:** Logs into the Storage Server and places your new Public Key into the remote `~/.ssh/authorized_keys` file.
**Why:** This tells the Storage Server to trust App Server 1 moving forward, allowing `scp` commands to execute silently in the background.

### Step 4: Create the Script File
```bash
sudo mkdir -p /scripts
sudo chown tony:tony /scripts
touch /scripts/news_archive.sh
```
**What it does:** Creates the required `/scripts` directory, grants ownership to the executing user, and creates the empty bash file.

### Step 5: Write the Archive Script
Open the file (`vi /scripts/news_archive.sh`) and add the following logic:
```bash
#!/bin/bash

# Define variables for clean, reusable code
SOURCE_DIR="/var/www/html/news"
LOCAL_ARCHIVE_DIR="/archives"
ARCHIVE_NAME="xfusioncorp_news.zip"
REMOTE_USER="natasha"
REMOTE_SERVER="ststor01"
REMOTE_ARCHIVE_DIR="/archives/"

# Task A & B: Compress the directory (-r for recursive to get all contents inside)
# Note: No sudo is used here per the requirements. 
zip -r ${LOCAL_ARCHIVE_DIR}/${ARCHIVE_NAME} ${SOURCE_DIR}

# Task C & D: Copy the created archive to the remote Storage Server silently
scp ${LOCAL_ARCHIVE_DIR}/${ARCHIVE_NAME} ${REMOTE_USER}@${REMOTE_SERVER}:${REMOTE_ARCHIVE_DIR}
```
**What it does:** Recursively compresses the source directory, then securely copies the `.zip` file over SSH to the remote server.

### Step 6: Make the Script Executable
```bash
chmod +x /scripts/news_archive.sh
```
**What it does:** Sets the executable permission bit on the file.
**Why:** Without this, Linux treats the file as a basic text document and will throw a "permission denied" error if you try to run it.

---

## ✅ 4. VERIFICATION & TROUBLESHOOTING (The "Proof")

### Verification:
Execute the script manually to test:
```bash
/scripts/news_archive.sh
```

**Verify Local Creation:**
```bash
ls -l /archives/xfusioncorp_news.zip
```
*Expected Output:* You should see the zip file listed with a current timestamp.

**Verify Remote Transfer:**
```bash
ssh natasha@ststor01 "ls -l /archives/xfusioncorp_news.zip"
```
*Expected Output:* The command should return silently with the file listing from the remote server, proving both the file transfer and the passwordless SSH configuration were successful.

### Troubleshooting:

| Error | Meaning | Solution |
| :--- | :--- | :--- |
| `Permission denied (publickey,password).` | Passwordless SSH is not configured correctly. | Re-run `ssh-copy-id` and ensure you are acting as the correct user (not root). |
| `zip command not found` | The zip package wasn't installed. | Run Step 1 (`sudo yum install zip -y`). |
| `zip warning: Permission denied` | The user doesn't have read access to the source data. | Check source permissions: `ls -ld /var/www/html/news` and adjust if necessary. |
| `scp: /archives/: Permission denied` | The remote user doesn't have write access to the remote folder. | SSH into the storage server and ensure the `/archives/` directory is owned by `natasha`. |
