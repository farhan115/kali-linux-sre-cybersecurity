# Linux Lab 08: Disk Usage Investigation & Log Cleanup

**Author:** Farhan Muntasir  
**GitHub:** [farhan115](https://github.com/farhan115)  
**Environment:** Kali Linux virtual machine  
**Date:** November 16, 2026

## Project overview

Investigated filesystem usage, identified large practice files, compressed them into an archive, removed the originals, measured recovered disk space, and tested restoration. This lab demonstrates a complete cleanup workflow with verification before and after deletion.

The files were created inside `~/linux-labs/day08-disk-cleanup`. They contain zeros to simulate log sizes; they are not actual application logs. All deletion commands below target only these practice files.

## Verified results

| Check | Observed result |
|---|---|
| Root filesystem at the start | 78% used; approximately 8.1 GB available |
| Practice files | 3 files totaling 35 MiB of logical data |
| Initial human-readable allocated usage | `36M` for the logs directory |
| Compressed archive file size | Approximately `36K` in `ls -lh` |
| Before cleanup, logs and archive directories combined | 35.01 MiB, measured before archive creation |
| After cleanup, logs and archive directories combined | 0.04 MiB |
| Net allocated space recovered | **34.96 MiB** |
| Archive integrity | PASSED |
| Restored files compared with archive | PASSED |
| Final allocated directory usage | logs: 4K; archive: 40K; reports: 28K |

The recovered-space calculation covers `logs` and `archive`, including their directory overhead. It excludes reports. Rounded displayed values can differ slightly from arithmetic using the underlying byte totals.

## Skills practiced

- Filesystem capacity checks with `df`.
- Directory usage analysis with `du`.
- Large-file discovery with `find`.
- Practice-file generation with `dd`.
- Archiving, compression, extraction, and comparison with `tar` and `gzip`.
- Targeted cleanup with `rm` and empty-directory removal with `rmdir`.
- Report capture with `tee` and calculations with `awk`.

## 1. Check capacity and create the workspace

```bash
df -h
mkdir -p ~/linux-labs/day08-disk-cleanup/{logs,archive,reports}
cd ~/linux-labs/day08-disk-cleanup
pwd
ls -l
```

`df -h` reports total, used, and available capacity for mounted filesystems. `mkdir -p` creates missing parent directories. Brace expansion creates the three subdirectories in one command. `pwd` confirms the working directory before creating or deleting files.

## 2. Create and investigate sample logs

Run the generation commands once in a fresh lab directory; `dd` overwrites an existing output file with the same name.

```bash
dd if=/dev/zero of=logs/application.log bs=1M count=20 status=none
dd if=/dev/zero of=logs/access.log bs=1M count=10 status=none
dd if=/dev/zero of=logs/debug.log bs=1M count=5 status=none

ls -lh logs/
du -sh logs archive reports
find logs/ -type f -size +8M -exec ls -lh {} \;
du -sh logs archive | tee reports/before-cleanup.txt
du -sB1 logs archive > reports/before-bytes.txt
```

`dd` reads from `/dev/zero` and writes blocks to each output file. Here, `bs=1M` means 1,048,576 bytes per block. `count` controls the number of blocks, and `status=none` suppresses routine transfer statistics.

| File | Logical size | Exact bytes |
|---|---:|---:|
| application.log | 20 MiB | 20,971,520 |
| access.log | 10 MiB | 10,485,760 |
| debug.log | 5 MiB | 5,242,880 |

`ls -lh` shows logical file sizes. `du -sh` summarizes allocated disk space, which includes allocation overhead and human-readable rounding. `find -type f -size +8M` selects regular files larger than 8 MiB: the application and access logs. `{}` represents each matching path; the escaped semicolon ends `-exec`.

`tee` displays output and saves it to a file. `du -sB1` records allocated usage in byte units for later calculations. Take the baseline before creating the archive to include the archive's storage cost in net savings.

## 3. Create and check the archive

```bash
tar -czf archive/practice-logs.tar.gz logs/
gzip -t archive/practice-logs.tar.gz && echo "Compression check: PASSED"
tar -tzf archive/practice-logs.tar.gz
du -sh logs/
ls -lh archive/practice-logs.tar.gz
```

| Flag | Meaning |
|---|---|
| `-c` | Create an archive |
| `-z` | Use gzip compression |
| `-f` | Use the specified archive filename |
| `-t` | List archive contents |
| `-x` | Extract archive contents |
| `-d` | Compare archive entries with files on disk |
| `-C` | Change directory for the tar operation |

`gzip -t` checks the compressed stream's integrity. It does not prove that all intended source files were included, so also inspect the archive listing. This lab's archive contained all three logs.

The archive was approximately 36 KiB because repeated zeros compress extremely well. This compression ratio should not be used to predict savings for real logs or already-compressed files.

## 4. Delete practice originals and calculate savings

After confirming the archive listing, run:

```bash
if gzip -t archive/practice-logs.tar.gz; then
    echo "Archive integrity: PASSED"
    rm -- logs/application.log logs/access.log logs/debug.log

    du -sh logs archive | tee reports/after-cleanup.txt
    du -sB1 logs archive > reports/after-bytes.txt

    awk '
        NR == FNR { before += $1; next }
        { after += $1 }
        END {
            printf "Before: %.2f MiB\n", before / 1048576
            printf "After: %.2f MiB\n", after / 1048576
            printf "Recovered: %.2f MiB\n", (before-after) / 1048576
        }
    ' reports/before-bytes.txt reports/after-bytes.txt \
      | tee reports/space-recovered.txt

    ls -lh logs/
else
    echo "Archive check FAILED. Original logs were kept."
fi
```

The conditional prevents deletion if the gzip integrity check fails. `rm --` ends option parsing and removes only the named files. The `awk` program sums the first column of the baseline and post-cleanup reports, subtracts the totals, and converts bytes to MiB.

Observed output:

```text
Archive integrity: PASSED
Before: 35.01 MiB
After: 0.04 MiB
Recovered: 34.96 MiB
total 0
```

For important data, complete a restore test before deleting originals. This exercise used disposable generated data and tested recovery immediately afterward. A local archive also does not protect against loss of the entire virtual disk.

## 5. Restore and compare

```bash
mkdir -p restore-test
tar -xzf archive/practice-logs.tar.gz -C restore-test/
ls -lh restore-test/logs/

if tar -dzf archive/practice-logs.tar.gz -C restore-test/; then
    echo "Restore verification: PASSED"
else
    echo "Restore verification: FAILED"
fi

find restore-test/logs/ -type f -printf '%f: %s bytes\n' \
    | tee reports/restore-verification.txt
```

`-C restore-test/` places extracted contents under the restore directory. Because the archive includes the `logs/` path, the resulting files appear in `restore-test/logs/`.

`tar -d` compares archive entries with the extracted files, including contents and relevant metadata. No differences and exit status 0 indicate a successful comparison. The observed result was **Restore verification: PASSED**, and all three byte sizes matched the table above.

## 6. Remove temporary restored copies and finish

```bash
rm -- restore-test/logs/application.log \
      restore-test/logs/access.log \
      restore-test/logs/debug.log
rmdir restore-test/logs restore-test

du -sh logs archive reports
gzip -t archive/practice-logs.tar.gz && echo "Final archive check: PASSED"
ls -lh reports/
```

`rmdir` removes only empty directories, making unexpected remaining files visible through an error. The final archive check passed. The archive and six reports were retained.

## Saved evidence

| Path | Purpose |
|---|---|
| `archive/practice-logs.tar.gz` | Compressed practice logs |
| `reports/before-cleanup.txt` | Readable baseline usage |
| `reports/before-bytes.txt` | Baseline allocated bytes |
| `reports/after-cleanup.txt` | Readable usage after cleanup |
| `reports/after-bytes.txt` | Allocated bytes after cleanup |
| `reports/space-recovered.txt` | Calculated space savings |
| `reports/restore-verification.txt` | Restored filenames and byte sizes |

The restore report records sizes; the successful comparison was shown separately in terminal output. Keep screenshots of the comparison and final integrity check as additional evidence.

## Troubleshooting questions and answers

### 1. Why did the report have a trailing `~` in its filename?

A trailing `~` was accidentally typed in the output path. At the end of this filename it is a literal character. Confirm it with `ls -lb reports/`, then rename it:

```bash
mv reports/before-cleanup.txt~ reports/before-cleanup.txt
```

Only run this correction if that incorrectly named file exists.

### 2. Why does `ls` show 35M of file data while `du` shows 36M?

`ls` reports logical file sizes. `du` measures allocated blocks, and `-h` rounds values for display. Directory allocation and filesystem allocation details also contribute. Compare precise values with `stat -c '%n %s bytes' logs/*.log` and `du -sB1 logs/` before cleanup.

### 3. Why does an empty logs directory still use 4K?

The directory itself stores filesystem metadata and directory entries. `ls -lh logs/` showing `total 0` means there are no allocated file blocks listed inside it; the directory can still occupy a block.

### 4. Why did `df -h` barely change after recovering about 35 MiB?

The recovered amount is small relative to the filesystem, and `df -h` rounds its display. Use `df -B1 .` for finer units. Other system activity can also change filesystem usage between measurements.

### 5. What should I do if `find` returns no large files?

Check `pwd`, `ls -lh logs/`, and whether the generation commands completed. After cleanup, no matching files is expected. The 5 MiB debug log is intentionally excluded by `-size +8M`.

### 6. What does “No such file or directory” mean here?

The current directory or filename may be wrong, or the file may already have been removed. Run `pwd` and `ls -l` before retrying. Return to the lab with `cd ~/linux-labs/day08-disk-cleanup`.

### 7. What if `gzip -t` fails?

Keep any originals. Inspect the archive path and error message; the archive may be incomplete, corrupted, or not gzip data. Recreate it from intact originals and repeat both integrity and contents checks. If originals are gone, recover from another known-good copy.

### 8. Why did `tar` report “Cannot stat”?

One of the supplied source paths is missing or inaccessible. Check `pwd`, `ls -ld logs`, and `ls -lh logs/`. If this happens during comparison, confirm that extraction created `restore-test/logs/` and that `-C restore-test/` is present.

### 9. Why does the archive pass integrity checking but contain the wrong files?

Integrity checks validate the compressed stream, not the intended file selection. Inspect `tar -tzf archive/practice-logs.tar.gz` and compare it with the source file list. Correct the source path and recreate the archive before removing originals.

### 10. What if restore verification fails?

Read the differences printed by `tar -d`. Confirm the extraction directory, file sizes, and whether restored files were modified. Extract into a fresh test directory and compare again. Do not claim successful recovery until the comparison passes.

### 11. Why does `rmdir` say “Directory not empty”?

Files remain, possibly including hidden files. Inspect with `ls -la restore-test/logs/` and remove only identified practice artifacts. Avoid broad recursive deletion to hide the error.

### 12. Why did deleting a real log fail to free space?

A running process may still hold the deleted file open. On a system you administer, `sudo lsof +L1` can help identify deleted-but-open files if `lsof` is installed. Use the service's supported rotation or reopen procedure; do not restart services blindly. This situation was not observed in this lab.

### 13. What if the system reports “No space left on device” despite free capacity?

Check both `df -h .` and `df -i .`. The filesystem may have exhausted inodes, or the write may target a different mount. Investigate the affected filesystem before deleting anything. This issue was not observed in the exercise.

### 14. Why does the space calculation show negative savings?

Check that the baseline was captured before archive creation, the same directories were measured both times, and restored copies are not included in only one measurement. Never overwrite the baseline after cleanup. Repeating the workflow in a fresh workspace produces a new measurement, not a reconstruction of the original evidence.

### 15. Can these commands be used directly on active production logs?

Active logs need the application's supported rotation and retention procedure. A process may keep writing during archiving or retain an open deleted file. This lab used static disposable files; production cleanup requires checking service behavior, retention requirements, and recovery copies.

## Interview questions

**What is the difference between `df` and `du`?**  
`df` reports filesystem capacity and usage. `du` measures allocated usage for selected files and directories.

**What is the difference between archiving and compression?**  
Archiving groups files and metadata into one container. Compression reduces the representation's size. `tar -czf` combines tar archiving with gzip compression.

**How did you verify successful recovery?**  
I extracted the archive into a separate directory, checked all three file sizes, and used `tar -d` to compare the restored files against the archive. The comparison passed.

**How did you measure net savings?**  
I captured allocated byte totals for the logs and archive directories before archiving and after deleting originals, then subtracted the post-cleanup total. This included the retained archive's storage cost and showed 34.96 MiB recovered.

**Why use a restore test if the archive passes an integrity check?**  
A valid compressed stream can still contain an incomplete or incorrect selection of files. A restore test checks whether the intended data can actually be recovered.

**What was the main limitation of your test data?**  
The files contained repeated zeros, so they compressed much better than typical real-world data. The workflow and verification are transferable; the compression ratio is not representative.

## Project summary

Created and investigated three simulated Linux logs totaling 35 MiB, archived and verified them, measured 34.96 MiB of net allocated space recovered, and successfully tested restoration. Retained the compressed archive and six reports as evidence.
