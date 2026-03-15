
---

# Scenario #1: The Spooky Full Disk (Saint-Malo)

*   **Lab Source:** [SadServers](https://sadservers.com)
*   **Difficulty:** Medium
*   **Topic:** Linux Filesystems & Inodes

## 1. The Problem Statement
The scenario prompt is simple but frustrating: 
> "There is a file `/home/admin/data/file` that cannot be created. Find the reason and fix it."

Upon logging in, I tried to manually create the file to see the error for myself:

```bash
touch /home/admin/data/file
# Output: touch: cannot touch '/home/admin/data/file': No space left on device
```

## 2. Initial Investigation
My first instinct was to check the disk space using `df -h`. Usually, "No space left on device" means exactly what it says.

```bash
df -h
```

**Observations:**
*   `/dev/sda1` is mounted on `/`.
*   Size: 8GB, Used: 2GB, **Available: 6GB**.
*   Wait... the disk shows **6GB free**, so why is it claiming it's full?

## 3. Digging Deeper (The "Aha!" Moment)
If the disk has physical space but won't allow new files, there are usually two suspects: **Deleted files held by processes** or **Inode exhaustion**.

I checked the Inodes usage next:

```bash
df -i
```

**Result:**
```text
Filesystem     Inodes  IUsed  IFree IUse% Mounted on
/dev/sda1       15000  15000      0  100% /
```

**Bingo.** The filesystem has run out of **Inodes**. 
> **Concept:** Every file and directory on Linux requires an Inode. If you have millions of tiny 0-byte files, you will run out of Inodes long before you run out of actual disk space (GB).

## 4. Identifying the Culprit
I needed to find where all these tiny files were hiding. I ran a command to count files in subdirectories:

```bash
find / -xdev -type d -exec sh -c 'echo "$(find "$1" -maxdepth 1 | wc -l) $1"' _ {} \; | sort -rn | head -n 10
```

The output pointed directly to:
`/var/lib/shared/tmp/` which contained over 10,000 tiny files.

## 5. The Resolution
To fix the issue immediately, I cleared out the temporary files to free up Inodes:

```bash
# Deleting thousands of files via 'rm' can sometimes fail if the argument list is too long
# Using 'find -delete' is more efficient
find /var/lib/shared/tmp/ -type f -delete
```

**Verification:**
```bash
df -i
# Result: IUse% is now down to 15%.

touch /home/admin/data/file
# Success!
```

---

## 6. DevOps Retrospective
**What did I learn?**
Monitoring only "Disk Space Used %" is a trap. In a production environment, you must also monitor **Inode Usage**.

**Prevention Strategy:**
1.  **Monitoring:** Add an alert in Prometheus/Grafana for `node_filesystem_files_free`.
2.  **Cleanup Jobs:** Implement a `logrotate` or a `cron` job to clean up temporary session files/cache files that are older than X days.
3.  **App Level:** Investigate the application code—why is it creating thousands of small files without cleaning them up?

---

### 🚀 Technical Toolkit Used:
| Command | Purpose |
| :--- | :--- |
| `touch` | Validated the error message |
| `df -h` | Checked human-readable disk space |
| `df -i` | Checked Inode count (The solution) |
| `find` | Located the directory with the most files |

---
