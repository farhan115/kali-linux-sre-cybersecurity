# Linux Day 15 — Backup and Restore Verification

**Lab folder:** `Day-15-Backup-Restore`  
**Environment:** Kali Linux · Bash · `tar` · `sha256sum` · `diff`

I backed up my Day 13 web service files, simulated a bad edit in a temporary copy, and recovered the original health response. The archive passed its SHA-256 check, and the restored directory matched the original with exit code **0**.

## 1. Stage a working copy

Run the commands in the same Bash session so `$lab` remains available. This lab uses the Day 13 site and its regular `health.txt` file.

```bash
lab=$(mktemp -d)
mkdir -p "$lab/source/site" "$lab/backup" "$lab/restore"

cp -a "$HOME/linux-labs/day-13-auto-recovery/site/." "$lab/source/site/"
cat "$lab/source/site/health.txt"
printf 'Lab folder: %s\n' "$lab"
```

- **`mktemp -d`** creates a fresh workspace. **`$(...)`** captures its path for reuse.
- **`mkdir -p`** creates the staging, backup, and restore directories, along with missing parents.
- **`cp -a`** copies the site while preserving attributes such as permissions and timestamps. The trailing **`/.`** copies the contents, including hidden entries, directly into the destination.
- **`cat`** shows the baseline response. **`printf`** prints the temporary path.

## 2. Create and check the archive

```bash
tar -czf "$lab/backup/site.tar.gz" -C "$lab/source" site
tar -tzf "$lab/backup/site.tar.gz"

(
    cd "$lab/backup" &&
    sha256sum site.tar.gz > site.tar.gz.sha256 &&
    sha256sum -c site.tar.gz.sha256
)
```

- **`tar -czf`** creates a gzip-compressed archive. **`-C`** sets the working directory for the archive operation so files are stored under the relative path `site/`.
- **`tar -tzf`** lists archive entries so I can confirm the health file is included.
- **`sha256sum`** records the archive digest and filename. **`sha256sum -c`** recalculates the digest and checks it against the saved value.
- **`(...)`** runs the checksum commands in a subshell, leaving the parent shell in its original directory. **`&&`** proceeds only after the previous command succeeds.

![Terminal output showing the staged health response, the archive listing, and a successful checksum check](images/01-backup-checksum.png)

**Screenshot 1.** The staged file prints **Day 13 health check OK**. The archive lists **`site/`** and **`site/health.txt`**, and the checksum reports **`site.tar.gz: OK`**. The bottom of the screenshot shows the simulated edit and extraction commands.

## 3. Simulate a bad edit and restore

```bash
printf 'BROKEN\n' > "$lab/source/site/health.txt"
tar -xzf "$lab/backup/site.tar.gz" -C "$lab/restore"

diff -u "$lab/source/site/health.txt" "$lab/restore/site/health.txt"
cat "$lab/restore/site/health.txt"
```

- **`>`** replaces the staged health file with `BROKEN`. The staged copy lets me simulate the edit while preserving the original Day 13 site.
- **`tar -xzf`** extracts the archive into the separate restore directory so the recovered files are available for inspection.
- **`diff -u`** displays the exact line change. **`cat`** confirms the restore contains the original text.

## 4. Verify the full directory

```bash
diff -r "$HOME/linux-labs/day-13-auto-recovery/site" "$lab/restore/site"
echo "Comparison exit code: $?"
```

**`diff -r`** compares both directory trees and their file contents. **`$?`** captures the immediately preceding exit code. Code **0** means no differences found, **1** means differences found, and **2** means the comparison encountered an error.

![Terminal output showing BROKEN replaced by Day 13 health check OK and comparison exit code 0](images/02-restore-comparison.png)

**Screenshot 2.** The unified diff shows **`-BROKEN`** from the edited copy and **`+Day 13 health check OK`** from the restored copy. The recursive comparison against the untouched Day 13 site reports **`Comparison exit code: 0`**.

## Interview and troubleshooting Q&A

**1. Why check the checksum and test the restore?**  
The checksum verifies that the archive matches the recorded digest. The restore confirms that the archive can be extracted and contains the files and contents I need.

**2. Why extract into a separate directory?**  
I can inspect recovered files and compare them with a baseline before replacing anything.

**3. Why did the two diff commands have different expected results?**  
The first compares the deliberately edited file with the recovered copy, so a difference gives exit code 1. The second compares the original site with the restored site, so matching trees give exit code 0.

**4. What would you check if the checksum failed?**  
Check that the digest belongs to this archive and that the files are in the expected backup directory. If the archive differs from the recorded digest, use a verified backup or recreate it from a trusted source.

**5. What if the checksum passes but a file is missing after extraction?**  
Use **`tar -tzf`** to confirm the file was included, check the archive's relative paths, and verify the extraction destination and any extraction errors.

**6. Does a recursive diff verify permissions and ownership?**  
A recursive diff checks the tree and file contents. For a metadata check, compare the relevant attributes separately with **`stat`**.

## Resume bullet

Created and validated a Linux backup and restore workflow using gzip-compressed tar archives, SHA-256 integrity checks, and recursive file comparison, confirming successful recovery with exit code 0.
