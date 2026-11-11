# Day 4: Linux Users, Groups, and Shared Directory Permissions

## Overview

Configured a shared help desk directory on Kali Linux using group ownership, setgid, and umask. Verified that authorized users could collaborate while an unauthorized user was denied access.

## Lab Environment

- OS: Kali Linux
- Group: `it_support`
- Authorized users: `labtech1`, `labtech2`
- Unauthorized test user: `laboutsider`
- Shared directory: `/srv/helpdesk-lab`

## 1. Create Users and Group

```bash
sudo groupadd it_support
sudo useradd -m labtech1
sudo useradd -m labtech2
sudo usermod -aG it_support labtech1
sudo usermod -aG it_support labtech2

id labtech1
id labtech2
```

- `-m` creates a home directory.
- `-aG` adds supplementary group membership without removing existing groups.

## 2. Configure the Shared Directory

```bash
sudo mkdir -p /srv/helpdesk-lab
sudo chown root:it_support /srv/helpdesk-lab
sudo chmod 2770 /srv/helpdesk-lab

ls -ld /srv/helpdesk-lab
```

Expected permissions and ownership:

```text
drwxrws--- root it_support
```

| Setting | Purpose |
|---|---|
| `2` | Enables setgid so new files inherit the directory's group |
| Owner `7` | Read, write, and enter |
| Group `7` | Read, write, and enter |
| Others `0` | No access |

## 3. Create a Shared Ticket File

```bash
sudo -u labtech1 bash -c 'umask 0007; echo "Ticket 001: DNS issue investigated" > /srv/helpdesk-lab/tickets.txt'

sudo ls -l /srv/helpdesk-lab/tickets.txt
```

Verified file ownership: `labtech1:it_support`

Verified permissions: `660` (`-rw-rw----`)

Setgid supplies group inheritance. The umask allows a newly created regular file to retain owner/group read and write permissions while removing access for others.

## 4. Verify Collaboration

```bash
sudo -u labtech2 bash -c 'echo "Ticket 001: Connectivity restored" >> /srv/helpdesk-lab/tickets.txt'

sudo -u labtech2 cat /srv/helpdesk-lab/tickets.txt
```

Verified output:

```text
Ticket 001: DNS issue investigated
Ticket 001: Connectivity restored
```

`>` overwrites a file; `>>` appends to it.

## 5. Verify Unauthorized Access Is Blocked

```bash
sudo useradd -m laboutsider
sudo -u laboutsider cat /srv/helpdesk-lab/tickets.txt
```

Verified result:

```text
Permission denied
```

## Troubleshooting

### Permission denied for an authorized user

Checked group membership and added the users to `it_support`:

```bash
sudo usermod -aG it_support labtech1
sudo usermod -aG it_support labtech2
id labtech1
```

Retried using a new `sudo -u` command.

### Shell displayed `quote>`

The command contained mismatched quotes. Pressed `Ctrl+C` and re-entered it with matching outer single quotes.

### Current user could not inspect the file

The current user was not authorized through the shared group. Used `sudo` for administrative verification without broadening directory permissions.

## Validation Results

| Test | Result |
|---|---|
| labtech1 creates the file | Passed |
| File inherits it_support group | Passed |
| File permissions are 660 | Passed |
| labtech2 appends and reads | Passed |
| laboutsider is denied access | Passed |

## Interview Questions

**What is setgid on a directory?**  
It makes newly created files and subdirectories inherit the directory's group.

**Does setgid automatically grant group write access?**  
No. File creation permissions, umask, and any applicable ACLs determine access.

**What is the difference between chmod and chown?**  
`chmod` changes permissions; `chown` changes ownership.

**Why use sudo -u during testing?**  
It runs a command as a specific user so access can be tested.

**What does umask 0007 do?**  
It removes all permissions for others from newly created items. With typical creation modes, files become 660 and directories become 770.

## Skills Demonstrated

Linux user administration, group management, permissions, setgid, umask, access validation, and troubleshooting.

## Author

Farhan Muntasir  
GitHub: [farhan115](https://github.com/farhan115)
