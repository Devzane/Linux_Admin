**Topic:** Non-Interactive Text Substitution using `sed`
**Category:** Linux / DevOps / Cloud Engineering
**Difficulty:** Intermediate
**Last Updated:** May 2026
**Related SOPs:** Bash Scripting Basics, Automated Container Configuration

---

# SOP: Automated String Substitution in Configuration Files

## 📝 1. OBJECTIVE (The "Why")
- **What problem does this solve?** It eliminates the need to manually open text editors (`vim`, `nano`) to modify configuration files, enabling fully automated system deployments.
- **When would someone need this?** During CI/CD pipeline executions, Docker container initialization, or routine server administration tasks where template files require environment-specific values.
- **What's the business/practical context?** Automation scripts must execute without human intervention. Modifying files programmatically ensures consistency, speed, and zero manual typos across fleets of servers.
- **Summary:** This document teaches how to use the Stream Editor (`sed`) to find and replace text strings within template files programmatically, non-interactively, and safely.

---

## ⚙️ 2. TARGET INFRASTRUCTURE / PREREQUISITES (The "Where")
- **System/Environment:** Linux/Unix-based operating systems (Ubuntu, Alpine, Amazon Linux, etc.).
- **Software/Tools:** `sed` (GNU Stream Editor) – pre-installed on virtually all Linux distributions.
- **Assumptions:** You have sufficient file system permissions (e.g., `root` or via `sudo`) to modify the target configuration files.
- **Relevant Paths:** Applicable to any plain-text file, such as `.xml`, `.ini`, `.json`, `.yaml`, or `.conf`.

---

## 🛠️ 3. EXECUTION (The "How")

### Step 1: Standard Global Substitution
**Action Description:** Replace all occurrences of a specific string in a file, saving the changes directly to that file.
- **Command:** 

```bash
sed -i 's/Random/Cloud/g' /root/nautilus.xml
```

**What it does:**
- `sed`: Invokes the stream editor.
- `-i`: Instructs `sed` to edit the file in-place rather than printing to standard output.
- `s/`: Initiates the substitute operation.
- `Random/Cloud/`: Defines the target pattern to find (`Random`) and the replacement string (`Cloud`).
- `g`: The global flag ensures all instances on a single line are replaced, not just the first one.

**Why:** This is the core mechanism for updating templates without opening an interactive text editor.
**Expected output:** No console output. The command returns silently to the prompt if successful (exit code 0).

### Step 2: Substitution with an Automatic Backup
**Action Description:** Perform the substitution while simultaneously creating a backup of the original file.
- **Command:** 

```bash
sed -i.bak 's/Random/Cloud/g' /root/nautilus.xml
```

**What it does:** Appends `.bak` to the `-i` flag. Before writing changes to `nautilus.xml`, it copies the original state to `nautilus.xml.bak`.
**Why:** Critical for safe deployments. If the substitution breaks the configuration, you have an immediate rollback mechanism.
**Expected output:** No console output, but running `ls /root/` will now show both `nautilus.xml` and `nautilus.xml.bak`.

### Step 3: Handling File Paths (Custom Delimiters)
**Action Description:** Replace strings that contain forward slashes (like URLs or file directories) without breaking the `sed` syntax.
- **Command:** 

```bash
sed -i 's#/var/www/html#/opt/app/public#g' /etc/nginx/sites-available/default
```

**What it does:** Replaces the standard `/` delimiter with `#`.
**Why:** If your target string contains `/`, `sed` will misinterpret it as the end of the substitution block and throw an error. Changing the delimiter prevents this collision.
**Expected output:** Silently updates the document pathways.

---

## ✅ 4. VERIFICATION & TROUBLESHOOTING (The "Proof")

### Verification:
Confirm the substitution was successful without opening the file manually.
Check for the new string:

```bash
grep "Cloud" /root/nautilus.xml
```

Output should display the specific lines where "Cloud" now exists.
Verify the old string is gone:

```bash
grep "Random" /root/nautilus.xml
```

Output should be completely empty.

### Troubleshooting:

| Error/Symptom | Meaning | Solution |
|---|---|---|
| `sed: can't read... Permission denied` | You lack write access to the file. | Prepend `sudo` to the command. |
| `sed: -e expression #1, char X: unterminated 's' command` | You forgot the closing `/` or delimiter collision occurred. | Check syntax and switch to `#` or `\|` |
| File remains unchanged | The string didn't match exactly (case-sensitive). | Add the `I` flag for case-insensitivity: `s/random/Cloud/gI`. |

**Pro Tip for macOS/BSD users:** The default `sed` on macOS requires an explicit empty string for the `-i` flag if you don't want a backup. You must use `sed -i '' 's/old/new/g' file` or it will throw an error. GNU `sed` (standard Linux) does not require this.

---

## ⚠️ COMMON MISTAKES

- **Forgetting the `-i` flag:** Running `sed 's/old/new/g' config.txt` will print the modified text to your screen, but the actual file remains completely unchanged.
- **Forgetting the `g` flag:** If "Random" appears twice on line 4, omitting `g` means only the first instance is replaced.
- **Partial Word Matches:** `sed -i 's/Port/Cloud/g'` will accidentally turn the word "Portal" into "Cloudal". Use word boundaries (`\b`) to strictly match isolated words: `sed -i 's/\bPort\b/Cloud/g'`.

---

## ⚡ QUICK REFERENCE

| Task | Command Pattern |
|---|---|
| Basic Replace | `sed -i 's/OLD/NEW/g' filename` |
| Safe Replace (Backup) | `sed -i.bak 's/OLD/NEW/g' filename` |
| Replace Paths | `sed -i 's#/old/dir#/new/dir#g' filename` |
| Exact Word Match | `sed -i 's/\bOLD\b/NEW/g' filename` |

## 🔗 RELATED CONCEPTS

- **Regular Expressions (Regex):** Advanced pattern matching within `sed`.
- **Awk:** For manipulating columnar data and fields.
- **Environment Variable Injection:** `envsubst` for replacing variables in templates.
