# Linux System Health Monitor — Lab 04

**Author:** Farhan Muntasir | **GitHub:** [farhan115](https://github.com/farhan115)

A Bash project that collects Linux system health information and saves a timestamped text report. Built as hands-on practice in Linux administration, monitoring, scripting, and troubleshooting.

## Objectives

- Inspect uptime, load averages, memory, disks, and processes.
- Use variables, command substitution, pipelines, and output redirection.
- Generate repeatable reports and flag root filesystem usage at 80% or higher.
- Diagnose a mismatch between the working directory and output path.

## Requirements

Linux with Bash, GNU coreutils, procps, awk, and sed. Kali Linux was used for the original lab. No sudo is needed. This script targets GNU/Linux, not macOS.

## Run the project

Download this repository, open its folder in a terminal, then run:

```bash
bash -n health-check.sh
chmod u+x health-check.sh
./health-check.sh
ls -lh reports/
cat reports/health-*.txt
```

An empty response from `bash -n` means the syntax check passed. The script prints the absolute location of the saved report. Reports are stored beside the script even when it is launched from another directory.

## Repository contents

| Path | Purpose |
|---|---|
| `health-check.sh` | Monitoring script |
| `reports/` | Generated reports; excluded from Git |
| `screenshots/` | Original lab troubleshooting evidence |
| `.gitignore` | Keeps local system reports out of commits |

## Checks performed

| Command | Purpose |
|---|---|
| `hostname` | Identify the machine |
| `date -Is` | Record time with timezone offset |
| `uptime` | Show running time and 1-, 5-, and 15-minute load averages |
| `free -h` | Show memory and swap usage |
| `df -h` | Show filesystem capacity and utilization |
| `ps aux --sort=-%cpu` | Sort processes by reported CPU utilization |
| `sed -n '1,6p'` | Display the header and first five process rows |
| `df --output=pcent /` | Retrieve root filesystem usage |

Load average is not a CPU percentage. The `ps` CPU figure is not an instantaneous sample; use `top` for an interactive view.

## Important Bash concepts

```bash
set -euo pipefail
```

- `-e`: exits on many unhandled command failures, with exceptions such as condition tests.
- `-u`: treats references to unset variables as errors.
- `-o pipefail`: makes a pipeline fail when one of its commands fails.

```bash
script_dir="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
report_dir="$script_dir/reports"
mkdir -p "$report_dir"
report_file="$report_dir/health-$(date +%Y%m%d-%H%M%S).txt"
```

`BASH_SOURCE[0]` identifies the script path. `dirname` extracts its directory; `cd` and `pwd` resolve it to an absolute path. `$(...)` inserts command output. `mkdir -p` creates missing directories and accepts an existing directory.

The timestamp format contains year, month, day, hour, minute, and second. For example: `health-20261112-081908.txt`. The filename assignment alone does not create a file. The final `> "$report_file"` writes the command group's standard output to it. Errors still appear in the terminal.

Double quotes preserve spaces in paths. Variable assignments have no spaces around `=`.

## Troubleshooting encountered

### Reports directory not found

The first attempt ran `chmod u+x health-check.sh`, followed by `ls -lh reports/`. Changing permissions does not execute the script, so no reports directory had been created yet.

**Resolution:** run `./health-check.sh` before listing reports.

### Directory-name mismatch

The working directory was `~/linux-labs/labs04-system-health`, while the original script saved reports under `~/linux-labs/lab04-system-health/reports`.

The supplied screenshot confirms that the original script printed a saved report path in the second directory, while a relative `ls reports/` in the first directory failed.

**Improvement in this version:** calculate the output directory from the script location instead of hard-coding the project folder name. This updated version should be rerun locally; the screenshots show the earlier version.

![Initial missing reports directory](screenshots/01-missing-reports.png)

![Report saved in a different project directory](screenshots/02-path-mismatch.png)

## Verify the result

1. Run the script and confirm the printed report path.
2. Open the file and confirm all six sections appear: uptime/load, memory, disk usage, top processes, root filesystem check, plus the report header.
3. Confirm the root usage message matches the percentage from `df --output=pcent /`.
4. Launch the script from another directory using its absolute path. Confirm reports still appear beside the script.

The warning is a capacity threshold, not a complete health diagnosis. The script collects one snapshot and does not send alerts or run continuously. Two runs within the same second can use the same filename and overwrite that report.

## Interview questions

**Why use `mkdir -p`?** It creates missing parent directories and succeeds if the directory already exists.

**What is the difference between `>` and `>>`?** `>` creates or overwrites a file; `>>` appends.

**Why quote variables?** To prevent word splitting and filename expansion when values contain spaces or special characters.

**Why use `pipefail`?** It helps detect failures earlier in a pipeline, rather than relying only on the last command.

**Why use `sed` instead of `head` here?** `head` can close the pipe early, causing an upstream SIGPIPE that may stop a script using `pipefail`. This `sed` command consumes the complete stream.

**Why derive paths from the script location?** Relative paths depend on the caller's working directory; deriving a base directory makes output placement consistent.

## Skills demonstrated

Bash scripting, Linux resource inspection, process inspection, filesystem monitoring, conditional logic, text processing, and path troubleshooting.

## Publish on GitHub

Suggested repository name: `linux-system-health-monitor`

Suggested description: `Bash system health monitor that generates timestamped Linux reports for uptime, memory, disk usage, and processes, with disk-capacity warnings and troubleshooting documentation.`

Create an empty repository on GitHub. From this project directory, run:

```bash
git init
git add README.md health-check.sh .gitignore screenshots/
git commit -m "Add Linux system health monitoring lab"
git branch -M main
git remote add origin https://github.com/farhan115/linux-system-health-monitor.git
git push -u origin main
```

Use your normal GitHub authentication when prompted. Review screenshots before publishing. Generated reports may contain local usernames, hostnames, and process arguments, so review and sanitize any report you choose to share.
