# 08-Linux Performance Analysis Methodology: USE, RED, and Troubleshooting Path Design

# Linux # Operations and Maintenance # SRE # Performance Analysis # USE Method # RED Method # Golden Signals # Troubleshooting Methodology # Fault Location # Host Troubleshooting

---

## Recommended Reading Path

01-Linux Basics and Host Operations and Maintenance/01-Host Troubleshooting/08-Linux Performance Analysis Methodology: USE, RED, and Troubleshooting Path Design.md

---

## I. Document Overview

This document focuses on the methodology aspects of Linux host performance analysis. Rather than individual commands, it covers:

- How to establish a troubleshooting path when encountering issues
- How to move from observed phenomena to relevant metrics
- How to locate resources based on metrics
- How to identify processes related to resources
- How to trace services through processes
- How to pinpoint the root cause of issues at the service level
- How to distinguish between resource, application, network, and storage-related problems
- How to use the USE method to analyze host resources
- How to apply the RED method to analyze service requests
- How to design troubleshooting approaches using golden signals
- How to avoid blind reboots, unnecessary scaling, or shifting blame

This document is part of an advanced methodology series on host troubleshooting.

The previous 01-07 articles focused on commands and specific scenarios, while this one emphasizes troubleshooting mindset.

The goal is:

- To link discrete commands into a coherent troubleshooting process
- To know what to check first upon observing a problem
- To determine the next steps based on metrics
- To distinguish between host resource bottlenecks and business service limitations
- To develop a high-level troubleshooting analysis framework commonly used by advanced SRE professionals

---

## II. Why Performance Analysis Methodology is Needed

In production troubleshooting, many problems arise not from a lack of knowledge about commands but from an unclear understanding of the proper sequence to take.

Common mistakes include:

```text
Rebooting a service when it runs slowly.
Killing processes when CPU usage is high.
Deleting logs when disk space is full.
Attributing network issues to firewalls.
Assuming insufficient CPU when load is high.
Thinking memory is low when free memory is low.
Considering the service normal once ports are connected.
attributing a 502 error to Nginx.
```

These practices have several drawbacks:

```text
They can disrupt the troubleshooting process.
They may lead to misjudgments about the root cause.
They often provide only temporary solutions rather than permanent fixes.
They increase the risk of causing secondary issues.
They prevent the establishment of a feedback loop for learning from mistakes.
```

More advanced troubleshooting should follow these steps:

```text
First, confirm the observed phenomenon.
→ Then, determine the scope of its impact.
→ Next, establish a timeline for the issue.
→ Analyze relevant metrics.
→ Locate the affected resources.
→ Identify the related processes.
→ Trace the service involved.
→ Verify your assumptions.
→ Implement corrective actions.
→ Finally, conduct a post-troubleshooting review.
```

---

## III. General Principles of Performance Analysis

Performance analysis should not rely solely on a single high metric to draw conclusions.

It is recommended to follow these principles:

```text
Start with the big picture before focusing on details.
Consider phenomena before examining metrics.
Focus on resources before analyzing processes.
Seek evidence before taking action.
Verify your findings before making repairs.
Restore normal operations before conducting a review.
```

During troubleshooting, avoid:

```text
Focusing only on a single metric.
Relying solely on instantaneous data.
Examining only one machine or service.
Considering only application logs.
Listening only to user descriptions.
Making judgments based on experience alone.
``

Instead, combine the following aspects:

```text
Business-related phenomena
System metrics
Service logs
Network connectivity
Resource utilization rates
Error rates
Latency values
Timeframes involved
Recent changes made
```

---

## IV. Four Levels of Linux Host Performance Analysis

Linux host troubleshooting can be divided into four levels:

```text
Level 1: System Layer
→ CPU, memory, disk, network, load, processes.
Level 2: Service Layer
→ systemd, ports, logs, configurations, dependent services.
Level 3: Application Layer
→ Request volumes, error rates, latency, threads, connection pools, slow queries.
Level 4: Architecture Layer
→ Load balancing, caching, databases, message queues, storage systems, network links.
```

When troubleshooting, avoid getting stuck at one level for too long.

For example:

```text
High CPU usage on the host.
→ It could be due to application code issues.
Interface timeouts.
→ It might be caused by slow database queries.
Nginx 502 errors.
→ It may indicate that the backend service has reached its connection limit.
High load values.
→ It could be related to disk I/O or NFS latency.
Container Out-of-Memory errors.
```bash
wc -l /var/log/nginx/access.log
``````bash
uptime
```

```bash
nproc
```

```bash
top
```

```bash
vmstat 1 5
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
ps -eo pid,ppid,user,stat,cmd | awk '$4 ~ /D/ {print}'
```

```bash
iostat -x 1 5
```

---

## Scenario 24: High CPU Usage

Troubleshooting Steps:

```text
Check the overall CPU usage.

→ Check if there is high usage on a single core.

→ Identify the process with high CPU usage.

→ Locate the thread with high CPU usage.

→ Determine whether it's in user space, kernel space, or due to soft interrupts.

→ Consider application logs and traffic patterns.
```

Commands:

```bash
top
```

```bash
mpstat -P ALL 1 5
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
pidstat -t -p PID 1 5
```

```bash
top -H -p PID
```

---

## Scenario 25: Insufficient Memory

Troubleshooting Steps:

```text
Check the available memory.

→ Check the swap space usage.

→ Monitor for Out-of-Memory (OOM) events.

→ Identify processes with high memory consumption.

→ Examine the process's memory map.

→ Determine if there are memory leaks, cache issues, or normal business growth.
```

Commands:

```bash
free -h
```

```bash
vmstat 1 5
```

```bash
dmesg -T | grep -i oom
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

```bash
pmap -x PID | tail
```

```bash
cat /proc/meminfo
```

---

## Scenario 26: Full Disk Space

Troubleshooting Steps:

```text
Identify which mount point is full.

→ Check if the inode count is high.

→ Look for large directories or files.

→ Consider log files, backups, databases, and Docker usage.

→ Back up or archive data first, then clean up the space.
```

Commands:

```bash
df -h
```

```bash
df -hi
```

```bash
du -h --max-depth=1 / | sort -hr | head
```

```bash
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

```bash
lsof | grep deleted
```

---

## Scenario 27: Port Unreachable

Troubleshooting Steps:

```text
Check if the local machine is listening on that port.

→ Verify if the listening address is correct.

→ Ensure the local firewall allows access.

→ Check if the security group permits it.

→ Confirm the routing settings are correct.

→ Verify if the request reaches the target machine.

→ Check if the service responds.
```

Commands:

```bash
ss -tunlp | grep 端口号
```

```bash
nc -zv -w 2 目标IP 端口号
```

```bash
iptables -L INPUT -n -v
```

```bash
firewall-cmd --list-all
```

```bash
ip route get 目标IP
```

```bash
tcpdump -i any -nn 端口号
```

---

## Section XII: Analyzing from Indicators to Root Causes

---

## Scenario 28: Indicators Point in a Direction but Are Not the Root Cause

For example:

```text
High CPU usage
```

Possible causes:

```text
Sudden increase in business traffic.

Code infinite loop.

Frequent garbage collection.

Compression tasks.

Log processing.

Encryption/decryption operations.

Many kernel system calls.

High number of soft interrupts.
```

Another example:

```text
High disk I/O usage
```

Possible causes:

```text
Large database queries.

Backup tasks.

Log flushing.

Image downloads.

Batch compression.

Swap paging.

Storage issues.
```

Therefore, when indicators show abnormalities, further questions are necessary:

```text
Who caused this issue?

Why is it happening now?

Is it related to any recent changes?

Is it linked to traffic patterns?

Does it depend on certain dependencies?

Can it be reproduced?

Do the indicators return to normal after fixes?
```

---

## Scenario 29: From Resources to Processes

After identifying resource abnormalities, the next step is usually to locate the relevant processes.

CPU:

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%---

## Scenario 40: Recommendations for Fault Troubleshooting Records

It is recommended to record the following information:

```text
Fault Title:

Occurrence Time:

Discovery Method:

Impact Scope:

User Symptoms:

Involved Services:

Involved Hosts:

Recent Changes:

Initial Assessment:

Troubleshooting Steps:

Key Evidence:

Temporary Recovery Measures:

Recovery Time:

Root Cause Analysis:

Long-Term Solutions:

Responsible Person:

Deadline:
```

---

## Scenario 41: Saving Command Output During Troubleshooting

You can save relevant information during troubleshooting.

Create a directory:

```bash
mkdir -p /tmp/troubleshooting-$(date +%F-%H%M%S)
```

Save basic system information:

```bash
uptime > /tmp/troubleshooting-$(date +%F-%H%M%S)/uptime.txt
```

A more convenient way is to define variables first:

```bash
TS=$(date +%F-%H%M%S)
```

```bash
mkdir -p /tmp/troubleshooting-$TS
```

```bash
uptime > /tmp/troubleshooting-$TS/uptime.txt
```

```bash
free -h > /tmp/troubleshooting-$TS/free.txt
```

```bash
df -h > /tmp/troubleshooting-$TS/df.txt
```

```bash
vmstat 1 5 > /tmp/troubleshooting-$TS/vmstat.txt
```

```bash
ss -s > /tmp/troubleshooting-$TS/ss-summary.txt
```

```bash
journalctl -p err -n 200 > /tmp/troubleshooting-$TS/journal-errors.txt
```

---

## XVIII. Summary of Common Troubleshooting Commands

---

## System Overview

```bash
uptime
```

```bash
top
```

```bash
free -h
```

```bash
vmstat 1 5
```

---

## CPU

```bash
mpstat -P ALL 1 5
```

```bash
pidstat 1 5
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

---

## Memory

```bash
free -h
```

```bash
cat /proc/meminfo
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

```bash
dmesg -T | grep -i oom
```

---

## Disk

```bash
df -h
```

```bash
df -hi
```

```bash
du -h --max-depth=1 / | sort -hr | head
```

```bash
iostat -x 1 5
```

```bash
iotop -o -P
```

---

## Network

```bash
ip a
```

```bash
ip route
```

```bash
ss -tunlp
```

```bash
ss -s
```

```bash
sar -n DEV 1 5
```

```bash
tcpdump -i any -nn
```

---

## Services

```bash
systemctl status 名称服务
```

```bash
journalctl -u 名称服务 -n 100
```

```bash
journalctl -u 名称服务 -f
```

```bash
ss -tunlp | grep 端口
```

---

## Logs

```bash
journalctl -p err
```

```bash
dmesg -T | tail -n 100
```

```bash
tail -n 100 /var/log/messages
```

```bash
tail -n 100 /var/log/syslog
```

---

## XIX. Notes for Production Troubleshooting

---

## 1. Do Not Focus Only on One Metric

For example, if you see:

```text
High load
```

You need to further check:

```text
Is the CPU usage high?

Is there a lot of IO wait?

Are there many processes in D-state?

Is swap memory frequently used?

Are there any storage issues?
```

---

## 2. Do Not Reboot Immediately to Hide Issues

Before rebooting, collect at least the following information:

```bash
uptime
```

```bash
top -b -n 1 | head -n 30
```

```bash
free -h
```

```bash
vmstat 1 5
```

```bash
journalctl -u 名称服务 -n 200
```

---

## 3. Do Not Consider Recovery as the Root Cause

For example, if a system recovers after a reboot, it only means that the reboot alleviated the problem. It does not mean that the root cause has been resolved. You still need to analyze