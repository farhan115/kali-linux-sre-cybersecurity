# Day 01 — Linux System Inventory & Security Baseline Audit

## Project Overview

In this hands-on project, I performed a Linux system inventory and baseline audit using **Kali Linux running in Oracle VirtualBox**. The objective was to collect, analyze, and document critical operating system information, including CPU architecture, memory utilization, disk capacity, filesystem partitions, system uptime, and virtualization details.

This project simulates a foundational responsibility of a **Site Reliability Engineer (SRE), Linux System Administrator, or Cybersecurity Analyst**: establishing a system baseline before implementing monitoring, troubleshooting performance issues, or performing security assessments.

## Lab Environment

- **Operating System:** Kali Linux Rolling
- **Virtualization:** Oracle VirtualBox
- **Linux Kernel:** 6.18.9+kali-amd64
- **System RAM:** 3.8 GiB
- **Virtual Disk:** 40 GB
- **Swap Space:** 2.1 GiB
- **User Account:** farhan

## Linux Commands Practiced

| Command | Purpose |
|---|---|
| `hostnamectl` | Identify operating system, hostname, and virtualization platform |
| `uname -a` | Display Linux kernel information |
| `lscpu` | Inspect CPU architecture and processor characteristics |
| `free -h` | Analyze RAM and swap utilization |
| `df -h` | Monitor filesystem capacity and available disk space |
| `lsblk` | Identify disks, partitions, and mount points |
| `uptime` | Check system uptime and load averages |
| `cat` | Read system inventory reports |
| `tail` | Inspect the latest lines of a report |
| `>` and `>>` | Redirect and append command output to report files |

## Lab Results

Successfully collected and documented the following system metrics:

- **Root filesystem:** 37.9 GB
- **Disk utilization:** 77%
- **Available disk space:** Approximately 8.1 GB
- **Total RAM:** 3.8 GiB
- **RAM in use:** Approximately 1.0 GiB
- **Swap utilization:** 0 B
- **Virtualization environment:** Oracle VirtualBox

Identified that the root filesystem was approaching a proposed 80% warning threshold, demonstrating the importance of proactive disk capacity monitoring.

## Technical Implementation

Created a dedicated project directory and organized reports, scripts, and evidence.

```bash
mkdir -p ~/sre-cybersecurity-labs/day01/{reports,scripts,evidence}
```

Collected system information and saved it to a baseline report:

```bash
hostnamectl > reports/system-baseline.txt
lscpu >> reports/system-baseline.txt
free -h >> reports/system-baseline.txt
df -h >> reports/system-baseline.txt
lsblk >> reports/system-baseline.txt
```

Verified the generated report:

```bash
cat reports/system-baseline.txt
tail -12 reports/system-baseline.txt
```

## SRE and Cybersecurity Skills Demonstrated

- Linux system administration
- Infrastructure inventory and baseline documentation
- CPU and memory resource inspection
- Disk utilization and capacity monitoring
- Filesystem and partition analysis
- Command-line troubleshooting
- Technical reporting and evidence collection
- Foundational operational monitoring

## Real-World Application

Site Reliability Engineers use system baselines to understand infrastructure capacity, investigate resource exhaustion, identify configuration changes, and establish monitoring thresholds.

This lab provides foundational skills for maintaining reliable Linux systems and preparing for advanced projects involving Prometheus, Grafana, Bash automation, alerting, incident response, and performance troubleshooting.

## Project Outcome

**Status: Completed Successfully**

Created and verified `reports/system-baseline.txt` and documented the lab in `README.md`.

This is Project 01 of my 30-Day Kali Linux Cybersecurity and Site Reliability Engineering portfolio.

## Author

**Farhan Muntasir**  
B.S. Computer Science — Texas Southern University  
CompTIA Security+ | Microsoft SC-300 | Full Stack Network Associate (FSNA)

GitHub: https://github.com/farhan115
