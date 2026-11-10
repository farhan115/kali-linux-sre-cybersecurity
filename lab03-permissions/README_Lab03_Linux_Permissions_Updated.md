# Lab 03 — Linux File Permissions, Ownership & Security

**Repository:** `linux-sre-labs`  
**Lab directory:** `lab03-permissions`  
**Environment:** Kali Linux  
**Status:** ✅ Completed and verified

## Overview

This hands-on lab explores Linux file access controls, numeric permissions, file ownership, and the use of elevated privileges. The exercises were performed in a Kali Linux terminal and verified using `ls -l` and `ls -la`.

## Learning objectives

- Create and inspect a file using terminal commands.
- Interpret owner, group, and others permission bits.
- Modify permissions using `chmod` and octal modes.
- Change file ownership with `sudo chown`.
- Restore secure ownership and permissions after testing.
- Verify final file state.

## Lab environment

- OS: Kali Linux
- Shell: Linux terminal
- Working directory: `~/linux-sre-labs/lab03-permissions`
- Test file: `secret.txt`
- User and group: `farhan`

## Step 1 — Create the workspace

```bash
mkdir -p ~/linux-sre-labs/lab03-permissions
cd ~/linux-sre-labs/lab03-permissions
```

`mkdir -p` creates the directory (and missing parent directories) without error if it already exists. `cd` enters the workspace.

## Step 2 — Create and inspect the file

```bash
touch secret.txt
echo "Confidential Linux data" > secret.txt
ls -l secret.txt
```

- `touch` creates the file if absent.
- `echo` prints text; `>` redirects it to the file, replacing existing content.
- `ls -l` shows permissions, link count, owner, group, size, modification time, and filename.

**Observed initial permissions:** `-rw-rw-r--` (664). These are the actual permissions seen in the lab, not a universal default; creation permissions depend on the system's umask.

## Step 3 — Restrict access to the owner

```bash
chmod 600 secret.txt
ls -l secret.txt
```

**Verified permissions:** `-rw-------`

Octal `600` means the owner has read (4) and write (2) access; group and others have no permissions.

## Step 4 — Explore executable permissions

```bash
chmod 755 secret.txt
ls -l secret.txt
```

**Verified permissions:** `-rwxr-xr-x`

Octal `755` means owner = read/write/execute (7), group = read/execute (5), and others = read/execute (5). This is useful for executable scripts and directories, but generally inappropriate for a confidential text file: it exposes the file for reading to other users and marks it executable unnecessarily.

## Step 5 — Change owner and group

```bash
sudo chown root:root secret.txt
ls -l secret.txt
```

**Verified ownership:** `root root`.

`sudo` runs the ownership change with elevated privileges. `chown OWNER:GROUP FILE` changes the owner and group. The permission bits remain unchanged unless modified separately.

## Step 6 — Restore ownership and secure permissions

```bash
sudo chown farhan:farhan secret.txt
chmod 600 secret.txt
ls -l secret.txt
ls -la
```

**Verified final state:**

```text
-rw------- 1 farhan farhan 24 Nov 10 07:16 secret.txt
```

The final screenshot also verified that `secret.txt` appears in the lab directory's `ls -la` listing. The 24-byte file is owned by `farhan`, belongs to the `farhan` group, and is accessible only to its owner through standard file permission bits.

## Understanding permission notation

```text
-rwxr-xr-x
│└─┬┘└┬┘└┬┘
│  │  │  └── Others: read + execute
│  │  └───── Group: read + execute
│  └──────── Owner: read + write + execute
└─────────── File type (- = regular file, d = directory)
```

| Mode | Symbolic | Meaning |
|---|---|---|
| `600` | `rw-------` | Owner can read/write; no group/other access |
| `644` | `rw-r--r--` | Owner can read/write; group/others can read |
| `755` | `rwxr-xr-x` | Owner can read/write/execute; group/others can read/execute |
| `700` | `rwx------` | Only owner can read/write/execute |
| `777` | `rwxrwxrwx` | Everyone can read/write/execute; usually unsafe |

Permission values: **read = 4**, **write = 2**, **execute = 1**.

## Security lessons

1. Follow **least privilege**: grant only the access required.
2. Use `600` for private files when only the owner needs access.
3. Avoid `777` because it allows other users to modify the file.
4. Use `sudo` only when administrative privileges are needed.
5. Check permissions and ownership after any change.
6. Execute permission on a directory allows traversal; on a regular file it marks the file executable, subject to other system controls.

## Troubleshooting

| Symptom | Likely cause | Check or fix |
|---|---|---|
| `cd: no such file or directory` | Misspelled directory | Check `pwd` and `ls ~`; use `linux-sre-labs`, not `linux-sare-labs` |
| `Permission denied` when changing ownership | Ordinary users cannot assign ownership to root | Use `sudo chown ...` when appropriate |
| File appears green in `ls` | Executable permission enabled and terminal color settings | Check actual bits with `ls -l` |
| File is accessible by others | Mode is too permissive | Review `ls -l` and apply `chmod 600` when appropriate |

## Verification checklist

- [x] Created the lab directory.
- [x] Created `secret.txt` and wrote sample content.
- [x] Inspected original permissions (`664`).
- [x] Applied and verified `chmod 600`.
- [x] Applied and verified `chmod 755`.
- [x] Changed ownership to `root:root` and verified it.
- [x] Restored ownership to `farhan:farhan`.
- [x] Restored permissions to `600`.
- [x] Verified final state with `ls -l` and `ls -la`.

## Interview questions and answers

**1. What is the difference between `chmod` and `chown`?**  
`chmod` changes file permission bits; `chown` changes file ownership and optionally group ownership.

**2. What does `chmod 600` mean?**  
The owner can read and write; the group and others have no permissions.

**3. What is the difference between `644` and `755`?**  
`644` allows owner read/write and everyone else read; `755` additionally allows execute for owner, group, and others.

**4. Why is `chmod 777` risky?**  
It allows any local user to modify the file, violating least privilege.

**5. Why was `sudo` needed for `chown root:root`?**  
Changing a file's owner to root requires elevated privileges on a typical Linux system.

**6. What does the first character of `ls -l` mean?**  
It indicates file type, such as `-` for regular file and `d` for directory.

**7. Does `chown` automatically change permissions?**  
No. Ownership and permissions are separate properties, though Linux may clear special privilege bits in some circumstances.

## Skills demonstrated

`Linux` · `Kali Linux` · `chmod` · `chown` · `sudo` · `ls` · `File Permissions` · `Ownership` · `Least Privilege` · `Security Hardening`

---

**Next lab:** Lab 04 — Linux Users & Groups Management.
