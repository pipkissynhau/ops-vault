# 05-Processes and systemd Service Troubleshooting

#Linux #Transport #TheBarrier. #Process #systemd #systemctl #journalctl #ServiceManagement #ProcessStatus #FaultLocation.

---

## Recommended Path

01-Linux Foundation and Host Maintenance/01-Host Troubleshooting/05-Processes and systemd Service Troubleshooting.md

---

## One: Document Overview

This document organizes commands related to **process and systemd service troubleshooting** on Linux hosts.

Key focus areas include:

- View processes
- Sort processes by CPU/memory
- View process tree
- Find specific processes
- View process startup command
- View process working directory
- View files opened by process
- View process status
- Zombie processes
- D state processes
- Terminate processes
- View systemd service status
- Start/stop/restart/reload services
- Boot-up management
- View failed services
- View service logs
- `systemctl`
- `journalctl`
- `ps`
- `pstree`
- `pgrep`
- `pidof`
- `kill`
- `lsof`

Goals:

- Quickly determine if a process exists
→ Locate which service the process belongs to
→ View process resource usage
→ Determine if a service failed to start
→ View service logs
→ Distinguish between process issues and systemd service issues
→ Safely handle abnormal processes and services

---

## Two: Process and Service Troubleshooting Approach

When a Linux host has service anomalies, do not only check ports or directly restart.

Recommended sequence:

```text
Confirmation of the existence of services

→ View systemd Service Status

→ View service logs

→ Can not open message

→ View process resource occupancy

→ View process startup command

→ View process work directories and open files

→ Whether the service failed to start up, the process went off abnormally, was under-resourced or was wrongly configured

→ Then decide. reload / restart / stop / kill
```

Common troubleshooting flow:

```text
systemctl status Service Name

→ journalctl -u Service Name

→ ps / pgrep / pidof Check process

→ top / ps Sort View Resources

→ lsof See file or port occupation

→ By reason
```

---

## Three: Relationship Between Processes and Services

Many business components in Linux can be viewed from both process and service perspectives.

For example Nginx:

```text
systemd Service Name
→ nginx.service

Process Name
→ nginx

Port
→ 80 / 443

Profile
→ /etc/nginx/nginx.conf
```

Troubleshooting should not focus on only one dimension.

For example:

```text
systemctl status nginx Show active
```

Only indicates that systemd considers the service as running.

Need to further confirm:

```text
Existence of the process

Port listening

Is the configuration correct?

Whether the log is wrong

Normality of operational visits
```

---

## Four: ps - View Processes

---

## Scenario 1: View Current System Processes

### Command

```bash
ps aux
```

### Purpose

View all processes in the current system.

Common fields:

```text
USER
→ Process Own Users

PID
→ Process ID

%CPU
→ CPU Usage

%MEM
→ Memory Usage

VSZ
→ Virtual Memory Size

RSS
→ Actual occupancy of physical memory

STAT
→ Process Status

START
→ Start Time

TIME
→ Cumulative CPU Time

COMMAND
→ Start Command
```

---

## Scenario 2: View Processes with Parent Process

### Command

```bash
ps -ef
```

### Purpose

`ps -ef` is more suitable for viewing parent-child process relationships.

Common fields:

```text
UID
→ User

PID
→ Process ID

PPID
→ Parent Process ID

C
→ CPU Use

STIME
→ Start Time

TTY
→ Terminal

TIME
→ Cumulative CPU Time

CMD
→ Command
```

---

## Scenario 3: View Processes Sorted by CPU

### Command

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head
```

View more:

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head -n 20
```

### Purpose

Quickly locate high CPU usage processes.

---

## Scenario 4: View Processes Sorted by Memory

### Command

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%mem | head
```

View more:

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%mem | head -n 20
```

### Purpose

Quickly locate high memory usage processes.

---

## Scenario 5: View Detailed Information of a Specific PID

### Command

```bash
ps -fp PID
```

Example:

```bash
ps -fp 12345
```

### Purpose

View detailed information of a specific process, including:

```text
Users

Parent Process

Start Time

Start Command
```

---

## Five: Find Specific Processes

---

## Scenario 6: Use grep to Find Processes

### Command

```bash
ps -ef | grep nginx
```

### Note

This is the most common way to find processes.

However, it will also match `grep nginx` itself.

You can filter like this:

```bash
ps -ef | grep nginx | grep -v grep
```

---

## Scenario 7: Use pgrep to Find Process PID

### Command

```bash
pgrep nginx
```

Display process name and PID:

```bash
pgrep -a nginx
```

### Purpose

Better suited for scripts and quick location than `ps | grep`.

---

## Scenario 8: Use pidof to Find Process PID

### Command

```bash
pidof nginx
```

### Purpose

Find PID by process name.

Suitable for:

```text
Quick find a service process ID
```

---

## Scenario 9: View Java Processes

### Command

```bash
ps -ef | grep java | grep -v grep
```

Or:

```bash
pgrep -a java
```

### Note

Java services often have long startup commands.

You can combine with:

```bash
tr '\0' ' ' < /proc/PID/cmdline
```

To view full startup parameters.

---

## Six: pstree - View Process Tree

---

## Scenario 10: View Process Tree

### Command

```bash
pstree
```

### Purpose

View process parent-child relationships in tree structure.

Suitable for determining:

```text
Who started a process?

Whether a script has given birth to a child.

Is there a multi-level subprocess in the service

Which parent is the abnormal process?
```

---

## Scenario 11: Display PID in Process Tree

### Command

```bash
pstree -p
```

### Purpose

Display process tree and PID.

---

## Scenario 12: View Specific Process Tree

### Command

```bash
pstree -p PID
```

Example:

```bash
pstree -p 12345
```

---

## Scenario 13: What to Do if pstree is Not Installed

Ubuntu/Debian:

```bash
apt install -y psmisc
```

RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y psmisc
```

Or:

```bash
dnf install -y psmisc
```

---

## Seven: /proc - View ProcessBottom Information

---

## Scenario 14: View Process Startup Command

### Command

```bash
cat /proc/PID/cmdline
```

Example:

```bash
cat /proc/12345/cmdline
```

Formatted output:

```bash
tr '\0' ' ' < /proc/12345/cmdline
```

### Purpose

View full startup command of a process.

Suitable for determining:

```text
Service Start Parameters

Profile Path

Java Start Parameter

Python Script Path

Directory of business processes
```

---

## Scenario 15: View Process Working Directory

### Command

```bash
ls -l /proc/PID/cwd
```

Example:

```bash
ls -l /proc/12345/cwd
```

### Purpose

View current working directory of a process.

Suitable for determining:

```text
Which application directory the process belongs to

Script started from which directory

Whether the service runs from the abnormal directory
```

---

## Scenario 16: View Executable File of a Process

### Command

```bash
ls -l /proc/PID/exe
```

Example:

```bash
ls -l /proc/12345/exe
```

### Purpose

View the path of the binary file actually executed by the process.

---

## Scenario 17: View Environment Variables of a Process

### Command

```bash
cat /proc/PID/environ
```

Formatted output:

```bash
tr '\0' '\n' < /proc/PID/environ
```

### Purpose

View environment variables of a process.

Suitable for troubleshooting:

```text
Validity of environmental variables

Which variables were read on service startup

PATH Correct?

JAVA_HOME Correct?

Configure Center Address Correct
```

Note:

```text
Environmental variables may contain sensitive information and do not disclose it at will.
```

---

## Scenario 18: View Process Status

### Command

```bash
cat /proc/PID/status
```

Example:

```bash
cat /proc/12345/status
```

### Key Fields

```text
Name
→ Process Name

State
→ Process Status

Pid
→ Process ID

PPid
→ Parent Process ID

Uid
→ User ID

Gid
→ User Group ID

VmRSS
→ Permanent Memory

Threads
→ Threads
```

---

## Eight: Understanding Process Status

---

## Scenario 19: STAT Field in ps

View:

```bash
ps aux
```

Common process states:

```text
R
→ Running, running or running

S
→ Sleeping, breaks your sleep

D
→ Uninterruptible sleepNo interruption of sleep.

T
→ StoppedStop!

Z
→ ZombieThe zombie process.

I
→ Idlekernel idle thread
```

Common additional flags:

```text
<
→ High Priority

N
→ Low Priority

L
→ A page locked in memory

s
→ session leader

l
→ Multi-line

+
→ Front desk process group
```

---

## Scenario 20: R State

```text
R
→ Process is running, or waiting CPU Run
```

If many processes are in R state, it may indicate CPU run queue pressure.

Continue to check:

```bash
top
```

```bash
vmstat 1 5
```

Focus on:

```text
r
```

---

## Scenario 21: S State

```text
S
→ Interrupted sleep
```

Many normal processes are in the S state.

For example:

```text
Waiting for Network Request

Waiting for user connection

Waiting for a scheduled task

Waiting for event to trigger
```

The S state itself is not necessarily a problem.

---

## Scenario 22: D State

```text
D
→ No interruption of sleep.
```

Commonly seen in:

```text
Disk IO Wait

NFS Carton.

Anomalous piece of equipment

Storage anomaly

File system anomaly
```

If a large number of D state processes appear, it usually indicates issues with IO or storage.

Check D state processes:

```bash
ps -eo pid,ppid,user,stat,cmd | awk '$4 ~ /D/ {print}'
```

Continue checking kernel logs:

```bash
dmesg -T | tail -n 100
```

Check IO:

```bash
iostat -x 1 5
```

---

## Scenario 23: Z State Zombie Processes

```text
Z
→ ZombieThe zombie process.
```

Zombie processes indicate:

```text
Subprocess has been withdrawn

But the parent process has not recovered its exit status.
```

Check zombie processes:

```bash
ps aux | awk '$8 ~ /Z/ {print}'
```

Check parent process:

```bash
ps -o pid,ppid,stat,cmd -p PID
```

Check process tree:

```bash
pstree -p PPID
```

Handling approach:

```text
A few zombie processes may not be serious.

There's a lot of zombie processes that indicate that the father's process may be problematic.

It's usually the father process, not the zombie process itself.
```

---

## IX. lsof: View Open Files by Process

---

## Scenario 24: View Files Opened by a Process

### Command

```bash
lsof -p PID
```

Example:

```bash
lsof -p 12345
```

### Purpose

View files, directories, sockets, etc. opened by a specific process.

Suitable for troubleshooting:

```text
Which log file is the process writing?

Which configuration file is occupied by the process

Whether the process is still occupied deleted Documentation

Whether the process opens a large number of files
```

---

## Scenario 25: View Which Process is Using a File

### Command

```bash
lsof /path/to/file
```

Example:

```bash
lsof /var/log/app.log
```

---

## Scenario 26: View Processes Using a Directory

### Command

```bash
lsof +D /data
```

### Purpose

Suitable for troubleshooting failed unmounts:

```text
umount: target is busy
```

---

## Scenario 27: View Deleted File Usage

### Command

```bash
lsof | grep deleted
```

### Purpose

Troubleshoot:

```text
df Show disk full
du Big file not found
```

Common causes:

```text
Large log file deleted
But the process still occupies the handle.
Space is not released.
```

Handling approach:

```text
Find the corresponding process
→ Whether or not. reload / restart
→ Release Document Thread
```

Note:

```text
Don't be direct. kill Production core process
```

---

## Scenario 28: What to Do if lsof is Not Installed

Ubuntu/Debian:

```bash
apt install -y lsof
```

RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y lsof
```

Or:

```bash
dnf install -y lsof
```

---

## X. kill: Terminate Processes

---

## Scenario 29: Gracefully Terminate a Process

### Command

```bash
kill PID
```

Equivalent to sending:

```text
SIGTERM
```

Example:

```bash
kill 12345
```

### Notes

`SIGTERM` requests the process to exit normally.

In production environments, prefer using:

```bash
kill PID
```

Rather than directly using:

```bash
kill -9 PID
```

---

## Scenario 30: Forcefully Terminate a Process

### Command

```bash
kill -9 PID
```

Example:

```bash
kill -9 12345
```

### Notes

`kill -9` sends:

```text
SIGKILL
```

Which forcefully kills the process.

Risks:

```text
Process cannot do cleanup

Possible data unpaved

It could lead to the locking of files.

Possible disruption of operations

It could destroy the scene.
```

Do not immediately use `kill -9` in production.

---

## Scenario 31: Terminate by Process Name

### Command

```bash
pkill Process Name
```

Example:

```bash
pkill nginx
```

Forcefully:

```bash
pkill -9 nginx
```

### Risks

Killing processes by name may accidentally terminate multiple processes.

Use with caution in production.

Recommend checking first:

```bash
pgrep -a nginx
```

Before deciding to take action.

---

## Scenario 32: Common Signals

View signal list:

```bash
kill -l
```

Common signals:

```text
TERM
→ Elegance.

KILL
→ Force Termination

HUP
→ Always used for reloading configuration

INT
→ Interrupt

QUIT
→ Exit and possible generation core
```

---

## XI. systemctl: Basic Service Management

---

## Scenario 33: Check Service Status

### Command

```bash
systemctl status Service Name
```

Example:

```bash
systemctl status nginx
```

Or:

```bash
systemctl status nginx.service
```

### Purpose

Check service:

```text
Whether or not active

Whether or not failed

Main Process PID

Recent Log

Start Time

Exit Code

unit File Path
```

---

## Scenario 34: Start a Service

### Command

```bash
systemctl start Service Name
```

Example:

```bash
systemctl start nginx
```

---

## Scenario 35: Stop a Service

### Command

```bash
systemctl stop Service Name
```

Example:

```bash
systemctl stop nginx
```

---

## Scenario 36: Restart a Service

### Command

```bash
systemctl restart Service Name
```

Example:

```bash
systemctl restart nginx
```

### Notes

Restarting will interrupt current processes.

In production environments, confirm:

```text
Operational impact

Is there a lead?

Whether to allow short interruptions

Changed Window
```

Before proceeding.

---

## Scenario 37: Reload Service Configuration

### Command

```bash
systemctl reload Service Name
```

Example:

```bash
systemctl reload nginx
```

### Notes

`reload` typically has less impact than `restart`.

Suitable for:

```text
Service supports smooth load configuration
```

But not all services support reload.

Check:

```bash
systemctl status Service Name
```

Or check if the unit file defines `ExecReload`.

---

## Scenario 38: reload-or-restart

### Command

```bash
systemctl reload-or-restart Service Name
```

### Purpose

If the service supports reload, perform reload.

If not, perform restart.

Still exercise caution in production as it may degrade to restart.

---

## XII. systemctl: Boot Autostart Management

---

## Scenario 39: Set Boot Autostart

### Command

```bash
systemctl enable Service Name
```

Example:

```bash
systemctl enable nginx
```

---

## Scenario 40: Disable Boot Autostart

### Command

```bash
systemctl disable Service Name
```

Example:

```bash
systemctl disable nginx
```

---

## Scenario 41: Check Boot Autostart Status

### Command

```bash
systemctl is-enabled Service Name
```

Example:

```bash
systemctl is-enabled nginx
```

Possible results:

```text
enabled
disabled
static
masked
```

---

## Scenario 42: Check if Service is Running

### Command

```bash
systemctl is-active Service Name
```

Example:

```bash
systemctl is-active nginx
```

Possible results:

```text
active
inactive
failed
activating
deactivating
```

---

## XIII. View Service List and Failed Services

---

## Scenario 43: View Running Services

### Command

```bash
systemctl list-units --type=service --state=running
```

---

## Scenario 44: View All Services

### Command

```bash
systemctl list-units --type=service --all
```

---

## Scenario 45: View Failed Services

### Command

```bash
systemctl --failed
```

### Purpose

Quickly view failed units in the current system.

Common output:

```text
failed service
failed mount
failed timer
```

---

## Scenario 46: Reset Failed Status

### Command

```bash
systemctl reset-failed
```

Reset specified service:

```bash
systemctl reset-failed Service Name
```

### Notes

`reset-failed` only clears the failed status, not equivalent to fixing the issue.

Before fixing, check logs:

```bash
journalctl -u Service Name
```

---

## XIV. journalctl: View Service Logs

---

## Scenario 47: View Logs for a Specific Service

### Command

```bash
journalctl -u Service Name
```

Example:

```bash
journalctl -u nginx
```

---

## Scenario 48: Real-time View Service Logs

### Command

```bash
journalctl -u Service Name -f
```

Example:

```bash
journalctl -u nginx -f
```

---

## Scenario 49: View Recent Logs

### Command

```bash
journalctl -u Service Name -n 100
```

Example:

```bash
journalctl -u nginx -n 100
```

Real-time view of recent logs:

```bash
journalctl -u nginx -n 100 -f
```

---

## Scenario 50: View Logs by Time

### Command

View today's logs:

```bash
journalctl -u nginx --since today
```

View recent 1 hour:

```bash
journalctl -u nginx --since "1 hour ago"
```

View logs for a specified time range:

```bash
journalctl -u nginx --since "2026-04-25 10:00:00" --until "2026-04-25 11:00:00"
```

---

## Scenario 51: View Logs Since Last Boot

### Command

```bash
journalctl -b
```

View logs for a specific service since last boot:

```bash
journalctl -u nginx -b
```

---

## Scenario 52: View Logs from Previous Boot

### Command

```bash
journalctl -b -1
```

View logs for a specific service during previous boot:

```bash
journalctl -u nginx -b -1
```

Suitable for troubleshooting:

```text
Why is the service unusual before system restart?

Last Start Failed

Log before and after machine abnormal restart
```

---

## Scenario 53: Output Logs Without Pagination

### Command

```bash
journalctl -u Service Name --no-pager
```

Example:

```bash
journalctl -u nginx --no-pager
```

---

## FifteenI don't know.systemd Unit File Troubleshooting

---

## Scenario 54: View Service Unit File Path

### Command

```bash
systemctl status Service Name
```

The output typically shows the unit file path.

You can also use:

```bash
systemctl cat Service Name
```

Example:

```bash
systemctl cat nginx
```

### Purpose

View the actual service configuration loaded by systemd.

---

## Scenario 55: Common Unit File Paths

Common paths:

```text
/usr/lib/systemd/system/

/lib/systemd/system/

/etc/systemd/system/
```

Description:

```text
/etc/systemd/system/
→ Always for administrator custom or overwrite configuration

/usr/lib/systemd/system/ or /lib/systemd/system/
→ Regularly provided by packages
```

---

## Scenario 56: Reload systemd After Modifying Unit

If you modify a unit file, execute:

```bash
systemctl daemon-reload
```

Then restart the service:

```bash
systemctl restart Service Name
```

Check status:

```bash
systemctl status Service Name
```

---

## Scenario 57: View Service Dependencies

### Command

```bash
systemctl list-dependencies Service Name
```

Example:

```bash
systemctl list-dependencies nginx
```

View reverse dependencies:

```bash
systemctl list-dependencies --reverse Service Name
```

---

## Scenario 58: View Service Properties

### Command

```bash
systemctl show Service Name
```

Example:

```bash
systemctl show nginx
```

View specific property:

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

## SixteenI don't know.Service Startup Failure Troubleshooting

---

## Scenario 59: Service Startup Failure

### Phenomenon

```bash
systemctl start nginx
```

Failed.

Check:

```bash
systemctl status nginx
```

Status may show:

```text
failed
```

---

## Scenario 60: Service Startup Failure Troubleshooting Commands

### Troubleshooting Flow

```bash
systemctl status nginx
```

```bash
journalctl -u nginx -n 100
```

```bash
journalctl -u nginx -f
```

Check if the configuration file is correct.

Take Nginx as an example:

```bash
nginx -t
```

Check for port conflicts:

```bash
ss -tunlp
```

Check for residual processes:

```bash
ps -ef | grep nginx | grep -v grep
```

---

## Scenario 61: Service Repeated Restarts

### Check Status

```bash
systemctl status Service Name
```

Check logs:

```bash
journalctl -u Service Name -n 200
```

Check unit configuration:

```bash
systemctl cat Service Name
```

Focus on:

```text
Restart
RestartSec
StartLimitBurst
StartLimitIntervalSec
ExecStart
```

Check properties:

```bash
systemctl show Service Name -p Restart
```

```bash
systemctl show Service Name -p RestartSec
```

---

## Scenario 62: Service Starts Very Slowly

Troubleshoot:

```bash
systemctl status Service Name
```

```bash
journalctl -u Service Name -b
```

Check system startup time:

```bash
systemd-analyze
```

Check service startup time sorting:

```bash
systemd-analyze blame
```

Check startup chain:

```bash
systemd-analyze critical-chain
```

---

## SeventeenI don't know.Common Troubleshooting Scenarios

---

## Scenario 63: Service Status is active, but Business is Abnormal

### Possible Causes

```text
Service processes exist, but operational ports are not listening

Service port listening local 127.0.0.1

Profile loading error

Dependency on services unusual

Process card is dead.

Log error

Firewall or network issues

Disk fill leads to write failure
```

### Troubleshooting Commands

```bash
systemctl status Service Name
```

```bash
journalctl -u Service Name -n 100
```

```bash
ps -ef | grep Service keyword | grep -v grep
```

```bash
ss -tunlp
```

```bash
df -h
```

```bash
top
```

---

## Scenario 64: Service is inactive

### Troubleshooting Commands

```bash
systemctl status Service Name
```

```bash
journalctl -u Service Name -n 100
```

```bash
systemctl is-enabled Service Name
```

Possible Causes:

```text
Service not started

Service stopped manually.

It's not on.

Exit after service startup failed

It's not service. systemd Management
```

---

## Scenario 65: Service is failed

### Troubleshooting Commands

```bash
systemctl status Service Name
```

```bash
journalctl -u Service Name -n 200
```

```bash
systemctl cat Service Name
```

Common Causes:

```text
Profile Error

Port Conflict

Insufficient Permissions

Reliance directory does not exist

Environmental variables are missing

Starting command error

Disk Full

Reliance on services not available
```

---

## Scenario 66: Service Exits Immediately After Startup

Troubleshoot:

```bash
systemctl status Service Name
```

```bash
journalctl -u Service Name -n 100
```

```bash
systemctl cat Service Name
```

Common Causes:

```text
Program started later, but unit Type mismatch

ExecStart Exit after execution

The configuration error caused the application to exit

Other Organiser

Insufficient Permissions

Environmental variables are missing
```

---

## Scenario 67: Service Configuration Changes Not Taking Effect

Troubleshoot:

```bash
systemctl status Service Name
```

```bash
systemctl cat Service Name
```

If you modified the unit file:

```bash
systemctl daemon-reload
```

Then:

```bash
systemctl restart Service Name
```

If you only modified application configuration, prioritize checking if the service supports reload:

```bash
systemctl reload Service Name
```

If not supported, consider restart.

---

## EighteenI don't know.Production Handling Notes

---

## 1. Do Not Immediately Use kill -9

Priority order:

```text
systemctl stop Service Name

→ kill PID

→ kill -9 PID
```

`kill -9` Should be the last resort.

---

## 2. Prefer Using systemctl to Manage systemd Services

If the service is managed by systemd, it's not recommended to directly kill the process and manually start it.

Recommended:

```bash
systemctl restart Service Name
```

Reason:

```text
systemd Maintain service status

systemd I can press it. unit Configure Startup

systemd Logging

systemd Could handle dependency and restart strategy
```

---

## 3. Check Logs Before Restarting the Service

Before restarting, it's recommended to check:

```bash
systemctl status Service Name
```

```bash
journalctl -u Service Name -n 100
```

To avoid losing context after restart.

---

## 4. Prefer reload Over restart

If the service supports smooth configuration reload, prefer:

```bash
systemctl reload Service Name
```

Instead of:

```bash
systemctl restart Service Name
```

Reason:

```text
reload The impact is usually smaller.
restart Could interrupt the current process
```

---

## 5. Confirm Impact Scope Before Handling Processes

Before handling, confirm:

```text
Which service does the process belong to?

Core business or not

Is there a lead?

Is there automatic pull-up?

Need to inform the operator

Whether windows need to be maintained
```

---

## NineteenI don't know.Common Commands in This Article

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
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%mem | head
```

---

## Process Search

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

## Process Tree

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

## Process Status

Check D-state processes:

```bash
ps -eo pid,ppid,user,stat,cmd | awk '$4 ~ /D/ {print}'
```

Check zombie processes:

```bash
ps aux | awk '$8 ~ /Z/ {print}'
```

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

## Ending Processes

```bash
kill PID
```

```bash
kill -9 PID
```

```bash
pkill Process Name
```

```bash
pkill -9 Process Name
```

```bash
kill -l
```

---

## systemctl Service Management

```bash
systemctl status Service Name
```

```bash
systemctl start Service Name
```

```bash
systemctl stop Service Name
```

```bash
systemctl restart Service Name
```

```bash
systemctl reload Service Name
```

```bash
systemctl reload-or-restart Service Name
```

```bash
systemctl enable Service Name
```

```bash
systemctl disable Service Name
```

```bash
systemctl is-enabled Service Name
```

```bash
systemctl is-active Service Name
```

---

## Service List

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
systemctl reset-failed Service Name
```

---

## journalctl Logs

```bash
journalctl -u Service Name
```

```bash
journalctl -u Service Name -f
```

```bash
journalctl -u Service Name -n 100
```

```bash
journalctl -u Service Name -n 100 -f
```

```bash
journalctl -u Service Name --since today
```

```bash
journalctl -u Service Name --since "1 hour ago"
```

```bash
journalctl -u Service Name --since "2026-04-25 10:00:00" --until "2026-04-25 11:00:00"
```

```bash
journalctl -b
```

```bash
journalctl -u Service Name -b
```

```bash
journalctl -b -1
```

```bash
journalctl -u Service Name -b -1
```

```bash
journalctl -u Service Name --no-pager
```

---

## Unit Files

```bash
systemctl cat Service Name
```

```bash
systemctl daemon-reload
```

```bash
systemctl list-dependencies Service Name
```

```bash
systemctl list-dependencies --reverse Service Name
```

```bash
systemctl show Service Name
```

```bash
systemctl show Service Name -p ExecStart
```

```bash
systemctl show Service Name -p MainPID
```

```bash
systemctl show Service Name -p Restart
```

---

## Startup Time

```bash
systemd-analyze
```

```bash
systemd-analyze blame
```

```bash
systemd-analyze critical-chain
```

---

## Tool Installation

Ubuntu / Debian Install psmisc:

```bash
apt install -y psmisc
```

RHEL / CentOS / Rocky / AlmaLinux Install psmisc:

```bash
yum install -y psmisc
```

Or:

```bash
dnf install -y psmisc
```

Ubuntu / Debian Install lsof:

```bash
apt install -y lsof
```

RHEL / CentOS / Rocky / AlmaLinux Install lsof:

```bash
yum install -y lsof
```

Or:

```bash
dnf install -y lsof
```

---

## Twenty. One-Line Summary

The core of process and systemd service troubleshooting is:

```text
Look at the service. systemctl

Look at the service log. journalctl

Process exists. ps / pgrep / pidof

Look at the process. pstree

Look at the details of the process. /proc

File occupancy. lsof
```

Service anomaly troubleshooting chain:

```text
systemctl status Service Name

→ journalctl -u Service Name -n 100

→ systemctl cat Service Name

→ ps / pgrep Check process

→ ss / lsof Check port and file

→ Processing according to logs and status
```

Process anomaly troubleshooting chain:

```text
ps aux / ps -ef

→ ps Sort Search CPU / Memory anomaly process

→ ps -fp PID Read the details.

→ /proc/PID/cmdline Look at the start-up order.

→ /proc/PID/cwd Look at the work directory.

→ lsof -p PID Look at the file.

→ To determine if it's necessary. stop / restart / kill
```

Production recommendations:

```text
Don't come up. kill -9
Do not read the log before restarting
systemd Priority for managed services systemctl
reload Priority restart
Pre-process recognition of operational impact
failed It's not just that. reset-failed
```