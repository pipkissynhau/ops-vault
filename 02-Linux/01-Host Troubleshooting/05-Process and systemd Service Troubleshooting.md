# 05 - Process and systemd Service Troubleshooting

# Linux # Operations # Troubleshooting # Processes # systemd # systemctl # journalctl # Service Management # Process Status # Fault Location

---

## Recommended Path

01-Linux Basics and Host Operations/01-Host Troubleshooting/05-Process and systemd Service Troubleshooting.md

---

## I. Document Description

This document compiles commands related to **process and systemd service troubleshooting** on Linux hosts.

Key points include:

- Viewing processes
- Sorting processes by CPU/memory
- Viewing process trees
- Finding specific processes
- Checking process startup commands
- Viewing process working directories
- Listing files opened by processes
- Monitoring process status
- Identifying zombie processes
- D-state processes
- Terminating processes
- Checking systemd service status
- Starting/stopping/reloading/restarting services
- Managing automatic startup at boot
- Investigating failed services
- Reviewing service logs
- Using `systemctl`, `journalctl`, `ps`, `pstree`, `pgrep`, `pidof`, `kill`, `lsof`

Goals:

- Quickly determine if a process exists.
- Identify which service a process belongs to.
- Check process resource usage.
- Determine if a service failed to start.
- Review service logs.
- Distinguish between process issues and systemd service problems.
- Safely handle abnormal processes and services.

---

## II. General Approach to Process and Service Troubleshooting

When a service on a Linux host encounters an issue, don’t just focus on ports or restart the system immediately.

Recommended sequence:

```text
Confirm if the service exists.
→ Check the systemd service status.
→ Review service logs.
→ Verify if the process exists.
→ Evaluate process resource usage.
→ Examine the process startup command.
→ Check the process working directory and opened files.
→ Determine whether the issue is due to a failed service start, abnormal process exit, insufficient resources, or configuration errors.
→ Then decide whether to reload/restart/stop/kill the service.
```

Common troubleshooting steps:

```text
systemctl status service_name
→ journalctl -u service_name
→ ps/pgrep/pidof to find the process
→ top/ps to sort processes by resource usage
→ lsof to check file or port usage
→ Take appropriate action based on the findings.
```

---

## III. The Relationship Between Processes and Services

Many Linux components can be viewed from both process and service perspectives.

For example, Nginx:

```text
systemd service_name
→ nginx.service
Process name
→ nginx
Port
→ 80/443
Configuration file
→ /etc/nginx/nginx.conf
```

Troubleshooting should consider multiple dimensions.

For instance:

```text
systemctl status nginx shows "active"
```

This only indicates that systemd considers the service running. Further checks are needed:

```text
Verify if the process exists.
Check if the port is being listened on.
Confirm if the configuration is correct.
Review logs for errors.
Ensure normal business access.
```

---

## IV. ps: Viewing Processes

---

## Scenario 1: Viewing Current System Processes

### Command

```bash
ps aux
```

### Purpose

Shows all current processes on the system.

Common fields:

```text
USER
→ Process owner
PID
→ Process ID
%CPU
→ CPU usage percentage
%MEM
→ Memory usage percentage
VSZ
→ Virtual memory size
RSS
→ Actual physical memory usage
STAT
→ Process status
START
→ Start time
TIME
→ Total CPU time used
COMMAND
→ Command used to start the process
```

---

## Scenario 2: Viewing Processes and Their Parent Processes

### Command

```bash
ps -ef
```

### Purpose

`ps -ef` is more useful for displaying parent-child process relationships.

Common fields:

```text
UID
→ User ID
PID
→ Process ID
PPID
→ Parent process ID
C
→ CPU usage percentage
STIME
→ Start time
TTY
→ Terminal
TIME
→ Total CPU time used
CMD
→ Command used to start the process
```

---

## Scenario 3: Sorting Processes by CPU Usage

### Command

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head
```

To view more results:

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head -n 20
```

### Purpose

Quickly identify processes with high CPU usage.

---

## Scenario 4: Sorting Processes by Memory Usage

### Command

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%mem | head
```

To view more results:

```bash
ps -eo pid,ppid,user,cmd,%mem→ Multithreading

+
→ Foreground Process Group
```
## Section 14: journalctl: Viewing Service Logs

---

## Scenario 47: Viewing Specific Service Logs

### Command

```bash
journalctl -u service_name
```

Example:

```bash
journalctl -u nginx
```

---

## Scenario 48: Realtime Viewing of Service Logs

### Command

```bash
journalctl -u service_name -f
```

Example:

```bash
journalctl -u nginx -f
```

---

## Scenario 49: Viewing Recent Logs

### Command

```bash
journalctl -u service_name -n 100
```

Example:

```bash
journalctl -u nginx -n 100
```

For real-time viewing of recent logs:

```bash
journalctl -u nginx -n 100 -f
```

---

## Scenario 50: Viewing Logs by Time

### Command

Viewing today's logs:

```bash
journalctl -u nginx --since today
```

Viewing the last 1 hour:

```bash
journalctl -u nginx --since "1 hour ago"
```

Viewing a specific time period:

```bash
journalctl -u nginx --since "2026-04-25 10:00:00" --until "2026-04-25 11:00:00"
```

---

## Scenario 51: Viewing Logs Since the Last Start

### Command

```bash
journalctl -b
```

Viewing logs for a specific service since the last start:

```bash
journalctl -u nginx -b
```

---

## Scenario 52: Viewing Logs from the Previous Start

### Command

```bash
journalctl -b -1
```

To view logs of a certain service from the previous start:

```bash
journalctl -u nginx -b -1
```

This is useful for troubleshooting issues such as:

```text
Why did the service malfunction before the system restart?
What were the reasons for the previous failed startup?
Review the logs before and after the unexpected system reboot.
```

---

## Scenario 53: Viewing Logs Without Pagination

### Command

```bash
journalctl -u service_name --no-pager
```

Example:

```bash
journalctl -u nginx --no-pager
```

---

## Section 15: Troubleshooting with systemd Unit Files

---

## Scenario 54: Checking the Path to the Service Unit File

### Command

```bash
systemctl status service_name
```

The path to the unit file is usually displayed in the output.

You can also use:

```bash
systemctl cat service_name
```

Example:

```bash
systemctl cat nginx
```

### Purpose

To view the actual service configuration loaded by systemd.

---

## Scenario 55: Common Path Locations for Unit Files

Common paths include:

```text
/usr/lib/systemd/system/
/lib/systemd/system/
/etc/systemd/system/
```

Explanation:

```text
/etc/systemd/system/
→ Often used for administrators to customize or override configurations.

/usr/lib/systemd/system/ or /lib/systemd/system/
→ Usually provided by software packages.
```

---

## Scenario 56: Reloading systemd After Modifying a Unit File

If you modify the unit file, you need to execute:

```bash
systemctl daemon-reload
```

Then restart the service:

```bash
systemctl restart service_name
```

Check the status:

```bash
systemctl status service_name
```

---

## Scenario 57: Viewing Service Dependencies

### Command

```bash
systemctl list-dependencies service_name
```

Example:

```bash
systemctl list-dependencies nginx
```

To view reverse dependencies:

```bash
systemctl list-dependencies --reverse service_name
```

---

## Scenario 58: Viewing Service Properties

### Command

```bash
systemctl show service_name
```

Example:

```bash
systemctl show nginx
```

To view a specific property:

```bash
systemctl show nginx -p ExecStart
```

```bash
systemctl show nginx -p MainPID
```

```bash
systemctl show nginx -p Restart
```

---

## Section 16: Troubleshooting Service Startup Failures

---

## Scenario 59: Service Startup Failure

### Phenomenon

```bash
systemctl start nginx
```

Fails.

Check:

```bash
systemctl status nginx
```

The status may show:

```text
failed
```

---

## Scenario 60: Commands for Troubleshooting Service Startup Failures

### Troubleshoot the Chain of Events

```bash
systemctl status nginx
```

```bash
journalctl -u nginx -n 100
```

journalIf a service is managed by systemd, it is not recommended to directly terminate the process and then start it manually. Instead, the following approach is advised:

```bash
systemctl restart service_name
```

Reasons:

- systemd can maintain the service state effectively.
- It allows for configuration-based startup of services.
- systemd keeps track of logs systematically.
- It handles dependencies and manages restart strategies efficiently.

---

## 3. Check Logs Before Restarting a Service

It is advisable to check the following before restarting a service:

```bash
systemctl status service_name
```

```bash
journalctl -u service_name -n 100
```

This helps prevent losing important information after a restart.

---

## 4. Use reload Instead of Restart When Possible

If a service supports smooth configuration reloading, it is better to use:

```bash
systemctl reload service_name
```

Rather than:

```bash
systemctl restart service_name
```

Reasons:

- Reloading usually has less impact on the system.
- Restarting will interrupt ongoing processes.

---

## 5. Determine the Impact Before Taking Action on a Process

Before dealing with a process, it is essential to consider the following factors:

- Which service the process belongs to.
- Whether it is critical to business operations.
- If there are any backups or automatic recovery mechanisms in place.
- Whether notifications need to be sent to relevant parties.
- Any required maintenance windows that may affect other services.

---

## Summary of Common Commands

---

## Process Viewing

```bash
ps aux
```

```bash
ps -ef
```

```bash
ps -fp PID
```

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
ps -eo pid,ppid:user,cmd,%mem,%cpu --sort=-%mem | head
```

---

## Process Searching

```bash
ps -ef | grep nginx
```

```bash
ps -ef | grep nginx | grep -v grep
```

```bash
pgrep nginx
```

```bash
pgrep -a nginx
```

```bash
pidof nginx
```

---

## Process Tree Display

```bash
pstree
```

```bash
pstree -p
```

```bash
pstree -p PID
```

---

## /proc Process Information

```bash
tr '\0' ' ' < /proc/PID/cmdline
```

```bash
ls -l /proc/PID/cwd
```

```bash
ls -l /proc/PID/exe
```

```bash
tr '\0' '\n' < /proc/PID/environ
```

```bash
cat /proc/PID/status
```

---

## Process Status Checking

To view processes in D-state:

```bash
ps -eo pid,ppid,user,stat,cmd | awk '$4 ~ /D/ {print}'
```

To identify zombie processes:

```bash
ps aux | awk '$8 ~ /Z/ {print}'
```

---

## Opening Files

```bash
lsof -p PID
```

```bash
lsof /path/to/file
```

```bash
lsof +D /data
```

```bash
lsof | grep deleted
```

---

## Terminating Processes

```bash
kill PID
```

```bash
kill -9 PID
```

```bash
pkill process_name
```

```bash
pkill -9 process_name
```

```bash
kill -l
```

---

## systemd Service Management

```bash
systemctl status service_name
```

```bash
systemctl start service_name
```

```bash
systemctl stop service_name
```

```bash
systemctl restart service_name
```

```bash
systemctl reload service_name
```

```bash
systemctl reload-or-restart service_name
```

```bash
systemctl enable service_name
```

```bash
systemctl disable service_name
```

```bash
systemctl is-enabled service_name
```

```bash
systemctl is-active service_name
```

---

## Service Listing

```bash
systemctl list-units --type=service --state=running
```

```bash
systemctl list-units --type=service --all
```

```bash
systemctl --failed
```

```bash
systemctl reset-failed
```

```bash
systemctl reset-failed service_name
```

---

## journalctl for Logs

```bash
journalctl -u service_name
```

```bash
journalctl -u service_name -f
```

```bash
journalctl -u service_name -n 100
```

```bash
journalctl -u service_name -n 100 -f
```

