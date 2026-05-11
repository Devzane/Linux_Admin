# SOP: Log Analysis & Incident Response Data Pipelines

## 📝 Objective

To rapidly extract, sort, and count anomalous data (such as identifying DDOS attackers or high-frequency visitors) from raw, unstructured server log files using standard Linux CLI text processing tools.

## ⚙️ Target Infrastructure

* **Tools:** `awk`, `sort`, `uniq`, `head`
* **Target:** Nginx/Apache `access.log` or generic space-separated log files.

## 🛠️ Execution Steps

**Scenario:** Find the top 5 most frequent IP addresses hitting a web server.

Assume an Nginx access log where the IP address is the first column in the text file.

### The Full Pipeline:

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 5
```

### Pipeline Architectural Breakdown:

#### `awk '{print $1}' /var/log/nginx/access.log`
* **Action:** Parses the text horizontally.
* **Result:** Extracts only the first column (the IP address) from every single line in the log, ignoring the user-agents and request paths.

#### `| sort`
* **Action:** Groups identical IP addresses together sequentially.
* **Result:** Required because the next tool (`uniq`) only compares adjacent lines.

#### `| uniq -c`
* **Action:** Deduplicates the list and counts the occurrences.
* **Result:** Outputs a list like `    45 192.168.1.15`.

#### `| sort -nr`
* **Action:** Sorts the new list based on the count.
* **Result:** `-n` ensures it sorts numerically (so 10 is higher than 2). `-r` reverses the order so the highest counts are pushed to the top of the output.

#### `| head -n 5`
* **Action:** Parses the text vertically.
* **Result:** Chops off everything below the 5th line, returning only the top 5 offenders.

## 🔧 Advanced Isolation

If you need to pass the output of this pipeline into a firewall rule, you must strip away the count numbers and isolate only the IP string.

Append another `awk` command to the end to grab the second column of the final output:

```bash
... | head -n 5 | awk '{print $2}'
```
