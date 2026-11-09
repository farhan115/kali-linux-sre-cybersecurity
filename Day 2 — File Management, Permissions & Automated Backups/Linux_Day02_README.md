# Day 02 — Linux File Management, Permissions & Automated Backups

**Series:** 30 Days of Linux & SRE Hands-On Labs  
**Platform:** Kali Linux in Oracle VirtualBox  
**Duration:** 30 minutes per day  
**Author:** Farhan Muntasir  
**GitHub:** [farhan115](https://github.com/farhan115)

## Project overview

This hands-on lab builds a small application directory tree, creates configuration and log files, practices Linux file permissions, and develops a Bash script that copies a configuration file into a timestamped backup. It is designed to demonstrate foundational Linux administration skills relevant to IT support, Linux administration, cybersecurity, DevOps, and site reliability engineering (SRE).

> **Lab status:** Completed. Terminal screenshots verified directory creation, file contents, permission changes, Bash script execution, and timestamped backup filenames. This is a learning lab, not a production-ready backup solution.

## Learning objectives

- Navigate Linux directories with `cd` and `pwd`.
- Create nested directories with `mkdir -p` and inspect them with `ls -R`.
- Create files with `touch` and inspect them with `cat`.
- Redirect output using `>` and `>>`.
- Understand `r`, `w`, `x` permissions and modify them with `chmod`.
- Edit a Bash script with `nano` and execute it.
- Copy files using `cp` and generate timestamps using `date`.
- Verify backup files and troubleshoot path errors.

## Project structure

```text
day02-linux-file-management-permissions/
├── README.md
├── app/
│   ├── backups/              # Generated backup files (ignored by Git)
│   ├── config/
│   │   └── app.conf
│   ├── data/
│   │   └── users.csv
│   ├── logs/
│   │   └── application.log
│   └── scripts/
│       └── backup.sh
├── screenshots/
│   └── README.md
└── .gitignore
```

## Lab 1 — Navigate and create directories

```bash
cd ~
mkdir -p ~/linux-sre-labs/day02
cd ~/linux-sre-labs/day02
pwd
mkdir -p app/{config,logs,data,backups,scripts}
ls -R app
```

**Explanation:** `cd ~` goes to your home directory; `~` represents your home path. `mkdir -p` creates missing parent directories and does not error if they already exist. Brace expansion `{config,logs,data,backups,scripts}` generates five paths. `ls -R` lists subdirectories recursively; uppercase `-R` differs from lowercase `-r`, which reverses sort order.

**Verification:** `pwd` should end with `/linux-sre-labs/day02`, and `ls -R app` should show five subdirectories.

## Lab 2 — Create and inspect files

```bash
touch app/config/app.conf
touch app/logs/application.log
touch app/data/users.csv
touch app/scripts/backup.sh
ls -R app
```

**Explanation:** `touch` creates an empty file if absent or updates its timestamps if it already exists. `ls -R` confirms the paths.

## Lab 3 — Redirect output and manage logs

```bash
echo "APP_ENV=development" > app/config/app.conf
cat app/config/app.conf

echo "Application started successfully" > app/logs/application.log
echo "User login successful" >> app/logs/application.log
cat app/logs/application.log

echo "id,name" > app/data/users.csv
```

**Explanation:** `echo` outputs text; `>` writes output to a file, replacing its previous contents; `>>` appends instead of replacing. `cat` displays file contents. The configuration file contains a development-only sample setting, not a secret.

**Expected log output:**

```text
Application started successfully
User login successful
```

## Lab 4 — Inspect and secure file permissions

```bash
ls -l app/config/app.conf
chmod 600 app/config/app.conf
ls -l app/config/app.conf
```

The lab initially showed permissions `-rw-rw-r--` (**664**), meaning owner read/write (6), group read/write (6), and others read-only (4). After `chmod 600`, the screenshot showed `-rw-------`: owner read/write, group no access, others no access.

| Symbol | Meaning | Numeric value |
|---|---|---:|
| `r` | Read | 4 |
| `w` | Write | 2 |
| `x` | Execute | 1 |
| `-` | Permission absent | 0 |

The first character of a long listing indicates file type (`-` for a regular file; `d` for a directory). Permission sets then apply to **owner**, **group**, and **others**. `chmod 600` demonstrates least privilege for a sensitive configuration file. It is not a replacement for secret management.

## Lab 5 — Write and execute a Bash script

```bash
chmod +x app/scripts/backup.sh
ls -l app/scripts/backup.sh
nano app/scripts/backup.sh
```

An initial practice script used:

```bash
#!/bin/bash
echo "Starting Linux backup..."
echo "Backup script executed successfully!"
```

Save Nano with **Ctrl+O**, **Enter**, then exit with **Ctrl+X**. Run the script from the Day 2 root directory:

```bash
./app/scripts/backup.sh
```

**Explanation:** `#!/bin/bash` is the shebang selecting Bash. `chmod +x` adds execute permission for all user classes subject to the process umask and existing mode. In the lab, `backup.sh` changed from `-rw-rw-r--` (664) to `-rwxrwxr-x` (775). `./` means execute a file via a path relative to the current directory.

## Lab 6 — Build a real file backup

The first functional version copied a configuration file to a fixed name:

```bash
cp app/config/app.conf app/backups/app.conf.bak
```

**Explanation:** `cp SOURCE DESTINATION` copies file content while leaving the original in place. A fixed destination is overwritten on later runs.

Verify the backup:

```bash
ls -l app/backups/
cat app/backups/app.conf.bak
```

The observed backup contained `APP_ENV=development`.

## Lab 7 — Add timestamps to backups

```bash
date
date +%Y%m%d_%H%M%S
```

The second command formats a timestamp such as `20261109_072912`.

| Token | Meaning |
|---|---|
| `%Y` | Four-digit year |
| `%m` | Month (01–12) |
| `%d` | Day of month |
| `%H` | Hour (00–23) |
| `%M` | Minute |
| `%S` | Second |

The final **lab-verified** script in `app/scripts/backup.sh` uses:

```bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
cp app/config/app.conf "app/backups/app.conf_${TIMESTAMP}.bak"
```

`$()` captures the output of `date`; `${TIMESTAMP}` inserts that value into the destination filename. Quoting the destination prevents accidental word splitting. Note: two runs in the **same second** can still overwrite a backup with the same timestamp.

**Run from the project root:**

```bash
./app/scripts/backup.sh
ls -lh app/backups/
```

**Observed results:** The terminal showed the script completing and listed the original `app.conf.bak` plus timestamped files `app.conf_20261109_073146.bak` and `app.conf_20261109_073741.bak`. The names were captured during the lab; they are not included as pre-generated backup files in this repository.

## Troubleshooting notes

| Problem | Cause | Resolution |
|---|---|---|
| `No such file or directory` for `app/conf/app.conf` | Directory typo: `conf` instead of `config` | Use `app/config/app.conf` |
| Script will not execute | Execute permission missing | Run `chmod +x app/scripts/backup.sh` |
| `cat` displays log entries rather than Bash code | Wrong file path used | Use `cat app/scripts/backup.sh` |
| Script contains duplicate backup commands | Commands accidentally pasted twice | Open in `nano` and remove duplicates |
| Terminal window closed | Shell session ended | Open terminal and `cd ~/linux-sre-labs/day02` |
| Relative paths fail from another directory | Script uses paths relative to current working directory | Run from project root; improve path handling in a future lab |

## Validation checklist

- [x] Created the Day 2 project directory and five application subdirectories.
- [x] Created configuration, log, data, and script files.
- [x] Wrote and inspected configuration and log contents.
- [x] Changed `app.conf` permissions from 664 to 600.
- [x] Added execute permission to `backup.sh`.
- [x] Executed a Bash script successfully.
- [x] Created and inspected a fixed-name configuration backup.
- [x] Created and listed timestamped backups.

## Security and production considerations

- This is a **training script**. It prints success after `cp` even if the copy fails. A production version should check the exit status, log failures, and exit nonzero.
- The script uses relative paths, so it must be executed from the project root.
- Timestamp resolution is one second; add higher-resolution timestamps or collision protection when needed.
- Real backups need retention policies, secure storage, monitoring, and periodic restore testing.
- Do not commit secrets, real credentials, production configurations, or sensitive logs to GitHub.
- The example `app.conf` contains only a harmless development setting.

## Interview questions and answers

**Q1. What does `mkdir -p` do?**  
Creates missing parent directories and avoids errors for directories that already exist.

**Q2. What is the difference between `ls -R` and `ls -r`?**  
`-R` recursively lists subdirectories; `-r` reverses sorting order.

**Q3. What is the difference between `>` and `>>`?**  
`>` replaces file contents; `>>` appends.

**Q4. What does `chmod 600` mean?**  
Owner can read/write; group and others have no permissions.

**Q5. What is a Bash shebang?**  
`#!/bin/bash` specifies the interpreter used to run the script.

**Q6. Why use timestamps in backup filenames?**  
To distinguish runs and retain multiple versions, provided filenames do not collide.

**Q7. How do you confirm a backup was created?**  
Check existence and permissions with `ls -l`, inspect sample content with `cat`, and for stronger assurance compare checksums or perform a restore test.

**Q8. Why is a success `echo` not sufficient proof?**  
The copy could fail while the script still prints success; production scripts should check command exit codes.

## Screenshot evidence

Add your actual Kali terminal screenshots to the `screenshots/` folder. Suggested names:

1. `01-directory-structure.png`
2. `02-created-files.png`
3. `03-configuration-and-logs.png`
4. `04-chmod-600.png`
5. `05-executable-script.png`
6. `06-backup-script.png`
7. `07-fixed-backup-verification.png`
8. `08-timestamped-backups.png`

See `screenshots/README.md` for a simple evidence checklist. Screenshots are not included in this starter archive; add the originals you want to publish.

## Next lab

**Day 03 — Linux Users, Groups & Ownership**: `whoami`, `id`, `groups`, `useradd`, `groupadd`, and `chown`.
