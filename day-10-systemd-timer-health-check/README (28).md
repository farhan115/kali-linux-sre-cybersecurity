# Day 10 — Linux Health Checks with a systemd Timer

**Author:** Farhan Muntasir | **Environment:** Kali Linux | **Scope:** User-level systemd units

## Objective

Schedule a Bash health check with systemd, verify that it appends to a local report, and recover from a service launch failure using journal evidence.

## Implementation

| Component | Purpose |
|---|---|
| `health-check.sh` | Accepts an output directory as its first argument. Appends an ISO timestamp, uptime and load, memory and swap usage, and root filesystem usage to `health.log`. |
| `linux-lab-health.service` | Runs the script once per activation with `Type=oneshot`. `ExecStart` uses `%h` to reference the user's home directory and launches the script directly. |
| `linux-lab-health.timer` | Schedules the first run about 15 seconds after activation and another run about one minute after the service's last activation. |
| `journalctl --user` | Shows service launches, script confirmations, and execution errors. |

### Bash script

Save as `~/linux-labs/day-10-systemd-timer/health-check.sh`.

```bash
#!/usr/bin/env bash
set -euo pipefail

report_dir="$1"
mkdir -p "$report_dir"

{
    printf '\n=== Health check at %s ===\n' "$(date -Is)"
    uptime
    free -h
    df -h /
} >> "$report_dir/health.log"

echo "Health report updated: $report_dir/health.log"
```

The grouped commands share one append redirection, preserving earlier reports. The final confirmation stays on standard output so it appears in the service journal.

### Service and timer

Create the unit directory with `mkdir -p ~/.config/systemd/user`. Save the following files there.

**`linux-lab-health.service`**

```ini
[Unit]
Description=Write the Linux lab health report

[Service]
Type=oneshot
ExecStart=%h/linux-labs/day-10-systemd-timer/health-check.sh %h/linux-labs/day-10-systemd-timer/reports
```

**`linux-lab-health.timer`**

```ini
[Unit]
Description=Schedule the Linux lab health report

[Timer]
OnActiveSec=15s
OnUnitActiveSec=1min
AccuracySec=1s

[Install]
WantedBy=timers.target
```

The units share a base name, so the timer activates the corresponding service automatically.

## Results

All times are local to the Kali VM on November 18, 2026.

| Time | Action | Evidence |
|---|---|---|
| 09:44:41 | Ran the script directly | The first timestamped report showed uptime, memory, swap, and filesystem usage. |
| 09:45:07 | Timer activated the service | The journal recorded successful completion and a second report appeared. |
| 09:45:21 | Removed execute permission and started the service | The journal recorded `Permission denied` and `status=203/EXEC`. |
| 09:45:31 | Restored execute permission and started the service | The journal recorded successful completion and a fresh report was appended. |

**Repeat interval still needs verification.** The final screenshot shows the existing 09:45:07 scheduled entry and the 09:45:31 manual recovery entry. It contains no new report timestamp after restarting the timer.

## Troubleshooting and recovery

With the timer stopped, removing execute permission reproduced a launch failure. The service executed the script directly, so restoring the execute bit resolved the issue.

```bash
cd ~/linux-labs/day-10-systemd-timer
chmod +x health-check.sh
systemctl --user start linux-lab-health.service
journalctl --user -u linux-lab-health.service -n 10 --no-pager
tail -n 9 reports/health.log
```

The journal retains earlier failures. Use the latest timestamp and a newly appended report to confirm recovery.

### Verify two consecutive scheduled runs

Run these commands after creating the script and units. The wait allows time for the initial 15-second delay and the next one-minute interval.

```bash
cd ~/linux-labs/day-10-systemd-timer
chmod +x health-check.sh
systemctl --user daemon-reload
systemctl --user stop linux-lab-health.timer
before=$(grep -c 'Health check at' reports/health.log 2>/dev/null || true)
systemctl --user start linux-lab-health.timer
sleep 85
systemctl --user stop linux-lab-health.timer
after=$(grep -c 'Health check at' reports/health.log 2>/dev/null || true)
printf 'New report entries: %s\n' "$(( ${after:-0} - ${before:-0} ))"
grep 'Health check at' reports/health.log | tail -n 2
```

Expect **two new report entries** approximately one minute apart. Keep manual service starts out of this test so each new report can be attributed to the timer.

## Interview questions

**Why use separate service and timer units?** The service defines the command and can run on demand. The timer defines when it runs automatically.

**How do you troubleshoot a timer that appears to run without updating its report?** Check the timer with `systemctl --user list-timers --all`, check the service journal for its latest execution, and verify the report path and timestamp. These checks distinguish scheduling problems from execution or output problems.

**What did `203/EXEC` mean in this lab?** systemd could not execute the command in `ExecStart`. The accompanying permission error identified the missing execute bit on the script.

## Resume bullet

- Automated Linux health reporting with Bash and a user-level systemd timer on Kali Linux, captured uptime, memory, and disk usage, and diagnosed and resolved a `203/EXEC` permission failure using the systemd journal.
