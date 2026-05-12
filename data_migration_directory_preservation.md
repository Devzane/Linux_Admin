# SOP: Data Migration & Directory Structure Preservation

## 📝 Objective

To securely locate, filter, and migrate specific user or application data across a filesystem while strictly preserving the original directory hierarchy.

During data mix-ups, forensic investigations, or user offboarding, System Administrators frequently need to extract specific files (e.g., owned by a specific user) from a shared directory. Simply copying the files into a flat folder destroys the organizational context. Preserving the directory path ensures the data remains structured and usable after the migration.

## ⚙️ Target Infrastructure

*   **OS:** Linux (All Distributions)
*   **Core Binaries:** `find`, `cp`

## 🛠️ Execution Steps

### 1. Connect to the Target Server

SSH into the specific application server where the data resides.

```bash
ssh <username>@<server-hostname-or-ip>
```

### 2. Dry Run the Search Query (SRE Best Practice)

Before executing a bulk copy or move operation, always run the search query by itself to verify it targets the correct data.

```bash
sudo find /path/to/source -type f -user <target_username>
```

*   `-type f`: Restricts the search to files only (ignores directories, symlinks, and sockets).
*   `-user <target_username>`: Filters by the specific file owner.

Verify the output list matches your expectations before proceeding.

### 3. Execute the Migration with Path Preservation

Use the `-exec` flag within `find` to pass the results directly into the `cp` (copy) command. The `--parents` flag is critical here.

```bash
sudo find /path/to/source -type f -user <target_username> -exec cp --parents {} /path/to/destination \;
```

**Command Breakdown:**

*   `-exec`: Instructs `find` to run a command on every matched item.
*   `cp`: The standard copy binary.
*   `--parents`: Forces `cp` to recreate the full absolute path of the source file inside the destination directory.
*   `{}`: A placeholder that `find` replaces with the current file path it is processing.
*   `\;`: The required terminator for the `-exec` command.

## ✅ Verification

Verify that the files were copied and that the directory structure was maintained in the new location.

```bash
sudo ls -R /path/to/destination
```

*(The `-R` flag lists the directory recursively, allowing you to see the newly built folder tree).*

## 📚 Quick Reference Table

| Objective | Command Syntax | Description |
| :--- | :--- | :--- |
| **Copy & Preserve Paths** | `cp --parents /src/file.txt /dest/` | Recreates the full `/src/` path inside `/dest/` |
| **Find by User** | `find /dir -user username` | Locates files owned by a specific user |
| **Find by Type** | `find /dir -type f` (files), `-type d` (dirs) | Filters by filesystem object type |
| **Find by Size** | `find /dir -size +100M` | Locates files larger than 100 Megabytes |

## 🔐 Security & Operational Best Practices

*   **Beware the Absolute Path:** The `cp --parents` command replicates the absolute path. If you search `/home/usersdata`, it will create `/destination/home/usersdata/...`. Ensure your target drive has enough capacity for the nested structure.
*   **Never blindly `-exec rm`:** When cleaning up data, never replace `cp` with `rm` (remove) on your first run. Always perform a dry run (Step 2) first, or redirect the output to a text file for review.
*   **Alternative (`rsync`):** For massive migrations (Gigabytes/Terabytes of data) across servers, prefer `rsync -avR` instead of `find`/`cp`. `rsync` handles network interruptions and delta-transfers much more safely.

## 🐛 Troubleshooting

**Problem:** `cp: cannot create regular file... No such file or directory`

**Solutions:**

1.  **Target Directory Missing:** The absolute final destination (e.g., `/beta`) might not exist yet. You must create the root destination folder before `cp --parents` can build the sub-folders inside it.
    ```bash
    sudo mkdir -p /path/to/destination
    ```
2.  **Permission Denied:** Ensure you are running the `find` command with `sudo` if you are scanning directories or writing to destinations owned by other users or root.
