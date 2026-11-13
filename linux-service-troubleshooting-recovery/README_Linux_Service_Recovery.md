# Linux Service Troubleshooting and Recovery Lab

**Author:** Farhan Muntasir | [GitHub](https://github.com/farhan115)  
**Environment:** Kali Linux VM, systemd, Python 3.13.11, Bash/zsh, curl  
**Scope:** Personal lab; approximately 30 minutes

## Overview

Created a systemd-managed Python web server, verified its initial operation, and deliberately introduced an invalid working directory. Diagnosed the resulting `200/CHDIR` startup failure using service status and log messages. This lab practices service configuration, incident investigation, and recovery validation.

**Evidence status:** Initial service operation and HTTP page retrieval are confirmed by screenshots. The injected `200/CHDIR` failure is also confirmed. The recovery procedure is documented below; final post-fix status and HTTP verification are pending.

## Objectives

- Create 1 custom systemd service running under a non-root account.
- Serve 1 HTML page on localhost TCP port 8080.
- Reproduce and diagnose 1 controlled configuration failure.
- Distinguish process health from application health.
- Restore the configuration and verify service and HTTP recovery.

## Environment and locations

| Item | Value / purpose |
|---|---|
| User | `farhan` |
| Unit file | `/etc/systemd/system/lab-web.service` |
| Website directory | `/home/farhan/linux-labs/service-recovery` |
| Website file | `index.html` |
| Endpoint | `http://127.0.0.1:8080` |
| Service process | `/usr/bin/python3 -m http.server` |
| Failure introduced | Nonexistent `WorkingDirectory` |

The server binds to loopback, so requests must originate inside the VM. Python's built-in HTTP server is used for this learning exercise.

## 1. Prepare the website

```bash
mkdir -p ~/linux-labs/service-recovery
cd ~/linux-labs/service-recovery
echo "Linux service recovery lab - Farhan Muntasir" > index.html
python3 --version
pwd
ls -l
```

Observed Python version: **3.13.11**. Confirmed the working directory and the presence of `index.html`.

## 2. Create the service

```bash
sudo nano /etc/systemd/system/lab-web.service
```

Enter this configuration. If your account or home directory differs, adjust `User` and `WorkingDirectory`.

```ini
[Unit]
Description=Linux Lab Web Server
After=network.target

[Service]
User=farhan
WorkingDirectory=/home/farhan/linux-labs/service-recovery
ExecStart=/usr/bin/python3 -m http.server 8080 --bind 127.0.0.1
Restart=no

[Install]
WantedBy=multi-user.target
```

Save with **Ctrl+O**, **Enter**, then exit with **Ctrl+X**.

| Directive | Meaning |
|---|---|
| `Description` | Human-readable service name. |
| `After=network.target` | Orders startup after this target when both participate in the transaction; does not guarantee network connectivity. |
| `User=farhan` | Runs the server as `farhan`, rather than root. |
| `WorkingDirectory` | Directory systemd enters before executing the server; Python serves files from here. |
| `ExecStart` | Program and arguments that start the service. |
| `--bind 127.0.0.1` | Listens only on the VM's loopback address. |
| `Restart=no` | Disables automatic restart, making the failure easy to inspect. |
| `WantedBy=multi-user.target` | Specifies the enablement target; this line alone does not enable boot startup. |

## 3. Start and check the baseline

Run each command separately:

```bash
sudo systemctl daemon-reload
sudo systemctl start lab-web
systemctl status lab-web --no-pager -l
curl http://127.0.0.1:8080
```

**Observed:** `active (running)` and the response body:

```text
Linux service recovery lab - Farhan Muntasir
```

The service was `disabled` for boot startup, which does not prevent manual startup.

## 4. Back up and introduce a controlled failure

Back up the working unit:

```bash
sudo cp /etc/systemd/system/lab-web.service /etc/systemd/system/lab-web.service.bak
sudo nano /etc/systemd/system/lab-web.service
```

Change only this line:

```ini
WorkingDirectory=/home/farhan/linux-labs/service-recovery-missing
```

Save, then apply the change:

```bash
sudo systemctl daemon-reload
sudo systemctl restart lab-web
systemctl status lab-web --no-pager -l
sudo journalctl -u lab-web -n 15 --no-pager
```

The separate journal command is provided for investigation; the submitted screenshot confirms the failure through logs displayed by `systemctl status`.

## 5. Diagnose the failure

Observed messages included:

```text
Active: failed (Result: exit-code)
status=200/CHDIR
Changing to the requested working directory failed: No such file or directory
Failed at step CHDIR spawning /usr/bin/python3: No such file or directory
```

**Root cause:** The configured working directory did not exist. systemd failed while changing directories, before starting Python. The message mentioning Python does not, by itself, mean the Python executable is missing; the `CHDIR` stage identifies the failure.

Optional diagnostic checks:

```bash
systemctl cat lab-web
ls -ld /home/farhan/linux-labs/service-recovery
ls -ld /home/farhan/linux-labs/service-recovery-missing
```

The valid directory should exist; the deliberately incorrect path should not. A working-directory failure can also result from insufficient permissions, so verify access when the directory exists.

## 6. Recover and verify

**Pending final screenshot verification.**

```bash
sudo nano /etc/systemd/system/lab-web.service
```

Restore:

```ini
WorkingDirectory=/home/farhan/linux-labs/service-recovery
```

Save and run:

```bash
sudo systemctl daemon-reload
sudo systemctl restart lab-web
systemctl status lab-web --no-pager -l
curl -i http://127.0.0.1:8080
```

Expected recovery evidence:

- Service state: `active (running)`.
- HTTP response: `200 OK`.
- Response body: the original website message.

A running process alone does not prove that an application responds correctly. Check both service state and the HTTP response.

Alternative rollback, if the original backup is intact:

```bash
sudo cp /etc/systemd/system/lab-web.service.bak /etc/systemd/system/lab-web.service
sudo systemctl daemon-reload
sudo systemctl restart lab-web
```

## Command explanations: who, what, where, why, when

| Command | What / where | Who / why / when |
|---|---|---|
| `mkdir -p PATH` | Creates the directory and missing parents. | Normal user; prepares the lab workspace. |
| `cd PATH` | Changes the shell's current directory. | Normal user; moves into the website folder. |
| `echo "text" > index.html` | Writes text to the file in the current directory; overwrites existing contents. | Normal user; creates the test page. |
| `sudo nano FILE` | Edits a system-owned file. | Administrator privileges; use to change the service configuration. |
| `sudo cp SOURCE DEST` | Copies the service configuration. | Administrator privileges; use before edits or for rollback. |
| `sudo systemctl daemon-reload` | Rereads systemd unit definitions. | Administrator privileges; run after editing a unit file. Does not restart the server. |
| `sudo systemctl start lab-web` | Starts the service if it is inactive. | Administrator privileges; starts the baseline server. |
| `sudo systemctl restart lab-web` | Stops and starts the service, or starts it if inactive. | Administrator privileges; applies the new startup configuration. |
| `systemctl status lab-web --no-pager -l` | Shows state and recent logs; full lines, without a pager. | Usually readable as a normal user; inspect after startup or failure. |
| `sudo journalctl -u lab-web -n 15 --no-pager` | Shows the latest 15 journal entries for the unit. | `sudo` ensures access to system logs; use for diagnosis. |
| `curl -i URL` | Sends an HTTP request and includes response headers. | Normal user; checks application behavior. |

Absolute paths, service names, and URLs allow the management commands to work from any directory. The relative `index.html` creation command depends on the shell's current directory.

## Troubleshooting encountered

| Symptom | Explanation | Action |
|---|---|---|
| `Unit lab-web.service not found` | systemd did not discover a valid unit at that stage. | Recreate the unit with nano, save to the exact path, then reload systemd. |
| Literal `^[[200~` before a pasted command | Paste-control characters reached the shell. | Cancel with Ctrl+C and type the command manually. |
| Shell remains at `heredoc>` | The here-document is still waiting for its exact closing delimiter. | Cancel with Ctrl+C; use nano. If using a heredoc, the closing delimiter must be `EOF` alone, not `EOF~`. |
| `curl: (7) Failed to connect` | No successful connection to the endpoint. | Inspect service status, configured address/port, and startup logs. |
| `200/CHDIR` | systemd could not enter the working directory. | Correct the path and verify directory access; reload and restart. |

## Evidence checklist

| Evidence | Status |
|---|---|
| Python version, working directory, and HTML file | Confirmed |
| Initial `active (running)` service | Confirmed |
| Initial HTTP page retrieval | Confirmed |
| Controlled `200/CHDIR` failure and diagnostic logs | Confirmed |
| Corrected service active after recovery | Pending |
| HTTP 200 response after recovery | Pending |

For GitHub, upload your terminal screenshots into an `images/` folder and insert relative Markdown image links using the actual filenames. Capture the final recovery results before marking this lab fully verified.

## Resume bullet

**Accurate for the evidence currently available:**

- Built 1 systemd-managed Python web service on Kali Linux, verified localhost HTTP access on TCP port 8080, and diagnosed 1 induced `200/CHDIR` startup failure using service status and logs.

**Use after successful recovery is verified:**

- Built and recovered 1 systemd-managed Python web service on Kali Linux; diagnosed an invalid working-directory failure from systemd logs, corrected the configuration, and validated recovery with active service status and an HTTP 200 response.

## Interview questions and answers

### 1. What did you build in this lab?

I created a custom systemd service that runs Python's HTTP server as a non-root user. It serves one HTML page on localhost port 8080. I verified initial operation, deliberately broke the working-directory setting, and diagnosed the resulting startup failure. Final recovery verification is still pending.

### 2. What is systemd?

systemd is a system and service manager used by many Linux distributions. It manages services and their startup ordering, tracks processes, and integrates with the system journal.

### 3. What is a systemd unit file?

It is a configuration file describing a resource managed by systemd. A `.service` unit describes how a service starts, its execution environment, and related settings.

### 4. Why store this unit in `/etc/systemd/system/`?

This location holds administrator-managed system units and local overrides. It is suitable for a custom service created for this VM.

### 5. What does `daemon-reload` do?

It makes systemd reread unit files and rebuild its dependency information. It does not restart the application; I restart the service separately to apply startup changes.

### 6. What is the difference between start, restart, and enable?

`start` launches a service now. `restart` stops and starts it. `enable` configures activation through the unit's installation settings, commonly for startup at boot, and does not start it immediately unless combined with `--now`.

### 7. Can a service be disabled and running?

Yes. Disabled describes enablement, while running describes the current runtime state. My baseline service was manually started without being enabled at boot.

### 8. What caused `200/CHDIR`?

systemd could not change into the configured `WorkingDirectory`. In this lab, I deliberately configured a nonexistent directory, so Python never started.

### 9. Does `200/CHDIR` mean HTTP 200?

No. `200/CHDIR` is a systemd execution failure code. HTTP `200 OK` is a successful application response. They belong to different systems.

### 10. How did you identify the root cause?

I inspected `systemctl status`, which showed `200/CHDIR` and a message stating that changing to the working directory failed. I connected that message to the deliberately changed path in the service file.

### 11. Why inspect journal logs?

They provide startup errors, timestamps, and application output. `journalctl -u lab-web` filters entries to this service and helps reconstruct what happened.

### 12. What is the difference between `User` and `WorkingDirectory`?

`User` selects the account running the process. `WorkingDirectory` sets the process's initial directory. The selected account needs sufficient access to that directory and its files.

### 13. Why run the server as `farhan` instead of root?

This lab server does not require administrator privileges to listen on port 8080. Running it as a regular account limits the privileges of the application process.

### 14. What does `127.0.0.1` mean?

It is an IPv4 loopback address referring to the local machine. In this lab it refers to the Kali VM, not the Windows host. Binding there limits listening to local connections.

### 15. Why use `curl -i`?

It displays the response headers and body. This lets me check the HTTP status and verify that the server returns the intended page.

### 16. Is active service status enough to prove recovery?

No. It proves systemd considers the service running at that moment. I also need an application check, such as an HTTP 200 response with the expected page content.

### 17. What if the directory exists but startup still fails?

I would check the exact configured path, directory permissions and traversal access for the service account, then review current logs. I would also investigate other reported errors instead of assuming the original cause persists.

### 18. What if port 8080 is already in use?

I would inspect logs for an address-in-use error and identify the listener with `sudo ss -ltnp 'sport = :8080'`. Then I would choose an unused lab port or stop a confirmed conflicting lab process.

### 19. Why create a backup before changing the configuration?

It preserves the known working unit and provides a simple rollback. Restoring the backup still requires `daemon-reload` and restarting the service.

### 20. How would you explain this incident briefly?

A service startup failed after its working-directory setting pointed to a nonexistent path. The status output identified `200/CHDIR`, so I isolated the failure to the startup environment. The documented fix restores the valid path, reloads systemd, and restarts the service; post-fix status and HTTP checks remain to be captured.

## Optional cleanup

After collecting evidence, stop the lab server:

```bash
sudo systemctl stop lab-web
```

To remove only this lab's unit and backup:

```bash
sudo rm /etc/systemd/system/lab-web.service /etc/systemd/system/lab-web.service.bak
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

Keep the website files and screenshots for your portfolio. These cleanup steps are optional and were not part of the verified execution.
