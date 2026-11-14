# Linux Lab 04: Process Management and Resource Monitoring

**Author:** Farhan Muntasir  
**GitHub:** [farhan115](https://github.com/farhan115)  
**Environment:** Kali Linux virtual machine in VirtualBox  
**Duration:** Approximately 30 minutes

## Overview

In this hands-on lab, I started and stopped a background process, verified termination, ranked processes by CPU and memory usage, and inspected disk space, RAM, swap, and system load. These tasks build foundational Linux troubleshooting skills for IT support, data center, and SRE roles.

## Objectives

- Understand PIDs, parent PIDs, and background jobs.
- Send the default termination signal to a practice process.
- Verify that the process has exited.
- Identify processes with the highest CPU and memory usage.
- Distinguish filesystem usage from directory usage.
- Interpret available RAM, swap usage, and load averages.

## Commands and Walkthrough

### 1. Identify the system

```bash
whoami
hostname
uptime
```

These commands display the current user, hostname, and system uptime/load.

### 2. Start a background process

Run these commands in the same terminal session:

```bash
sleep 600 &
LAB_PID=$!
echo "Practice process PID: $LAB_PID"
jobs -l
ps -p "$LAB_PID" -o pid,ppid,user,stat,etime,cmd
```

| Item | Meaning |
|---|---|
| `sleep 600` | Wait for 600 seconds |
| `&` | Run the command in the background |
| `$!` | PID of the most recent background command |
| `LAB_PID` | Shell variable storing the practice PID |
| `jobs -l` | Jobs managed by the current shell, including PIDs |
| `PPID` | Parent process ID |
| `STAT` | Process state and additional flags |
| `ETIME` | Elapsed time since the process started |

### 3. Inspect live activity

```bash
top
```

Observe CPU and memory activity. Press `q` to exit. This is an optional repeatable check; the saved screenshots do not include a full `top` summary.

### 4. Stop and verify the practice process

```bash
kill "$LAB_PID"
sleep 1
ps -p "$LAB_PID"
```

By default, `kill` sends SIGTERM, allowing a process to handle termination. In this lab, the shell reported `terminated sleep 600`, and the final `ps` output contained only its header. This verified that the practice PID was no longer running at that check.

![Practice process termination](images/01-process-termination.png)

### 5. Rank processes by CPU and memory

```bash
ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head -n 6
ps -eo pid,comm,%cpu,%mem --sort=-%mem | head -n 6
```

`-e` selects all processes, `-o` specifies columns, and the minus sign in the sort key requests descending order. Six lines include one header and five process rows.

`ps` CPU percentage is averaged over a process's lifetime. It is not the same as an instantaneous CPU reading.

![CPU and memory process rankings](images/02-process-rankings.png)

### 6. Check filesystem and directory usage

```bash
df -h /
du -sh ~/Downloads ~/Documents 2>/dev/null
```

- `df` reports filesystem capacity and usage.
- `du` measures space used by selected files/directories.
- `-h` uses human-readable units.
- `du -s` summarizes each requested directory.
- `2>/dev/null` discards standard error; omit it when troubleshooting missing directories or permission issues.

![Disk and directory usage](images/03-disk-usage.png)

### 7. Check memory and system load

```bash
free -h
uptime
nproc
```

`free` shows RAM and swap information. The `available` column estimates memory available for new applications without swapping. `uptime` shows load averages for the previous 1, 5, and 15 minutes. `nproc` reports processing units available to the current process.

The last screenshot shows `nproc` typed at the prompt, but its output is not captured. CPU count remains unverified in this documentation.

![Memory and load checks](images/04-memory-load.png)

## Observed Results

Values represent this VM at the time of the screenshots, not permanent system specifications or performance guarantees.

| Check | Observed result |
|---|---|
| Practice process | `sleep 600` terminated; verification showed no process row |
| Highest CPU entry in captured list | `Xorg`: 1.3% CPU |
| Highest memory entry in captured list | `Xorg`: 4.1% memory |
| Root filesystem | `/dev/sda1`, mounted at `/` |
| Root capacity | 38 GB as displayed by `df -h` |
| Root usage | 28 GB used; 78% |
| Root available space | 8.1 GB |
| Downloads directory | 16 KB as displayed by `du -sh` |
| Documents directory | 4.0 KB as displayed by `du -sh` |
| Total RAM | 3.8 GiB |
| Used RAM | 935 MiB |
| Available RAM | Approximately 2.9 GiB; output wrapped |
| Swap | 2.1 GiB total; 0 B used |
| Uptime | 27 minutes |
| Load averages | 0.15, 0.13, 0.16 |
| Available CPU count | Output not captured |

The captured values show low load and substantial available RAM. The process rankings alone do not establish overall CPU health. Root filesystem usage merits monitoring as the VM grows; no cleanup was performed.

## Troubleshooting Encountered

### `df: invalid option -- '/'`

**Cause:** I entered `df -h/`. Without a space, `/` was parsed as part of the options.

**Correction:**

```bash
df -h /
```

**Validation:** The corrected command displayed the root filesystem capacity, usage, and available space.

### Wrapped memory output

**Cause:** The terminal was too narrow, splitting headings and values across lines.

**Correction:** Maximize the terminal or reduce its font size, then rerun `free -h`.

## Interview Questions and Answers

### 1. What is a process?
A running instance of a program, identified by a PID and associated with resources such as memory and open files.

### 2. What is the difference between PID and PPID?
PID identifies a process. PPID identifies its parent process.

### 3. How do you run a command in the background?
Append `&`, for example `sleep 600 &`. Capture `$!` immediately if you need that background command's PID.

### 4. How are `jobs` and `ps` different?
`jobs` lists jobs tracked by the current shell. `ps` lists selected operating system processes, including processes outside that shell.

### 5. What does `kill` do by default?
It sends SIGTERM. Despite its name, it sends a signal; successful signal delivery does not guarantee immediate process exit.

### 6. How do you verify termination?
Inspect the target PID with `ps -p PID`. No process row means that PID is absent at the time of the check.

### 7. What is the difference between SIGTERM and SIGKILL?
SIGTERM can be handled so the program can clean up. SIGKILL cannot be caught or ignored and prevents application cleanup. Use it only when necessary after checking the target.

### 8. What does `ps --sort=-%mem` do?
It sorts selected processes by memory percentage, highest first.

### 9. Why use `head -n 6`?
To show one header plus the first five processes in this output.

### 10. What is the difference between `df` and `du`?
`df` reports filesystem-level allocation. `du` totals space associated with files/directories it can inspect. Their values may differ because of metadata, deleted open files, permissions, and filesystem behavior.

### 11. Why is available memory more useful than free memory alone?
Linux uses RAM for caches. Available memory estimates what applications can use without swapping, including reclaimable memory.

### 12. What is swap?
Storage used for memory pages moved out of RAM. Sustained swap activity can indicate memory pressure; nonzero swap usage by itself does not prove a current problem.

### 13. What do the three load averages represent?
Average runnable tasks and tasks in uninterruptible sleep over 1, 5, and 15 minutes. Load is not a CPU percentage.

### 14. How does CPU count help interpret load?
It provides capacity context. Sustained runnable load above available processing units can indicate CPU contention, but load may also include tasks waiting on I/O.

### 15. How would you explain this lab in an interview?
“I used a Kali Linux VM to manage a background process and verify its termination. I ranked processes by CPU and memory, checked filesystem and directory usage, and interpreted RAM, swap, and load averages. I also diagnosed a command-spacing error and documented my results with screenshots.”

## Troubleshooting Questions and Answers

These are practice scenarios. Except for the spacing and terminal-width issues above, they were not encountered or reproduced in this lab.

### 1. `kill "$LAB_PID"` fails. What should you check?
Check that the variable is populated in the same shell and inspect the target before sending a signal:

```bash
printf 'LAB_PID=%s\n' "$LAB_PID"
```

If it is empty, restart the practice process and capture `$!` again. If it contains a valid PID, inspect it with `ps -p "$LAB_PID" -o pid,user,stat,cmd`.

### 2. `kill` reports “No such process.” Why?
The process may have already exited, the PID may be wrong, or the practice sleep may have reached its timeout. Recheck the PID and command; do not guess another target.

### 3. `kill` reports “Operation not permitted.” What does this mean?
Your user lacks permission to signal that process. Verify its owner and purpose. Use an authorized administrative account only when the task requires it.

### 4. A process remains after SIGTERM. What next?
Allow time for cleanup, inspect its state, and check relevant application logs. A zombie requires parent reaping; a task in uninterruptible sleep may be waiting on I/O. SIGKILL is a last resort for a verified process when its impact is understood.

### 5. A process has state `Z`. Can `kill -9` remove it?
No. A zombie has exited but its parent has not collected its exit status. Investigate the parent process and its reaping behavior.

### 6. A process shows high CPU. How do you investigate?
Observe it over several `top` updates, identify its command and owner, and determine whether the workload is expected. Check application logs and correlate the activity with user impact before deciding on action.

### 7. Load is high but CPU usage is low. Why?
Tasks may be waiting in uninterruptible sleep, often related to I/O. Inspect process states with `ps -eo pid,stat,comm` and use `vmstat 1 5` if available to investigate further. High load does not necessarily mean CPU saturation.

### 8. Free RAM is low. Is the system out of memory?
Not necessarily. Check `available` RAM, cache, swap activity, and symptoms. Use `free -h` and, if available, `vmstat 1 5` to distinguish cache use from sustained pressure.

### 9. `df` reports a full disk but `du` cannot explain it. What next?
Check access errors, mount boundaries, filesystem metadata, and deleted files still held open. If installed and authorized, `sudo lsof +L1` can help identify deleted open files. Avoid deleting files before identifying the cause.

### 10. There is free disk space, but creating files fails. What else might be full?
Inodes may be exhausted. Run `df -i /`. Also check permissions, quotas, and whether the filesystem is mounted read-only.

### 11. `du` prints nothing when errors are redirected. What should you do?
Remove `2>/dev/null` and rerun the command. Error messages may reveal missing directories or permission problems.

### 12. Why does `jobs` show nothing while `ps` shows the process?
The process may belong to a different shell, have been detached, or no longer be tracked as a job by the current shell.

## Skills Practiced

Linux CLI • Process inspection • Background jobs • Signal handling • CPU and RAM monitoring • Filesystem usage • Troubleshooting • Technical documentation

## Repository Details

**Suggested repository name:** `linux-lab04-process-monitoring`

**Repository description:** Hands-on Kali Linux lab covering background processes, SIGTERM, CPU and memory monitoring, disk usage, system load, and troubleshooting with screenshots and interview Q&A.

**Suggested topics:** `linux`, `kali-linux`, `linux-administration`, `process-management`, `troubleshooting`, `it-support`, `sre`, `homelab`

## Publishing on GitHub

1. Create a repository using the suggested name and description.
2. Extract the ZIP package.
3. Upload the contents of the extracted project folder so `README.md` is at the repository root and `images` is beside it.
4. Commit the files and verify that all four screenshots render in the README.

## Project Update

Completed Linux Lab 04: Process Management and Resource Monitoring in my Kali Linux VM. I practiced background jobs, process termination, CPU and memory rankings, disk usage checks, and RAM/load interpretation. The documentation includes four screenshots, 15 interview questions, and 12 troubleshooting scenarios with answers.
