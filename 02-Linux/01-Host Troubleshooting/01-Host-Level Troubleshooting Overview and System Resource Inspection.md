# 01-Host-Level Troubleshooting Overview and System Resource Inspection

#Linux #Operations and Maintenance #Troubleshooting #Host Operations and Maintenance #System Resources #CPU #Memory #Load #System Troubleshooting

---

## Recommended Path

01-Linux Basics and Host Operations and Maintenance/01-Host Troubleshooting/01-Host-Level Troubleshooting Overview and System Resource Inspection.md

---

## I. Document Description

This document compiles **common commands for host-level troubleshooting overview and system resource inspection**, based on frequently encountered troubleshooting scenarios in production environments.

The focus of this article is on:

- The overall approach to host-level troubleshooting
- Entry points for system resource inspection
- Preliminary assessment of CPU/memory/load
- Tools such as `top`, `htop`, `free`, `uptime`, `vmstat`
- Initial investigation pathways for system lag
- Summary of commonly used resource inspection commands

This article is the first in the host troubleshooting series, aiming to address:

```text
When a Linux host is malfunctioning, what should be checked first?
```

The objectives are:

- To establish an order of host troubleshooting steps
- To quickly determine if there is any resource pressure on the system
- To initially identify potential issues with CPU, memory, load, IO wait, swap, etc.
- To lay the foundation for further investigations into CPU, memory, disk, network, and service-related problems

---

## II. Overall Approach to Host-Level Troubleshooting

When troubleshooting at the host level, do not immediately restart services or terminate processes.

Recommended sequence:

```text
Confirm the issue

→ Inspect resources (CPU/memory/load/disk/network)

→ Check processes

→ Examine ports

→ Verify routing/firewall/DNS settings

→ Analyze logs

→ Identify the root cause

→ Apply solutions and verify results
```

The general approach can be summarized as:

```text
First, check resources

→ Then examine processes/services

→ Next, inspect ports/network

→ Follow up with rules/routing checks

→ Finally, review logs
```

---

## III. Why Check Resources First in Host Troubleshooting

Many production issues may seem like business-related problems on the surface, such as:

```text
Slow interfaces

Unresponsive services

Delayed SSH logins

Database connection timeouts

Slow Pod or container startup

Nginx returning 502/504 errors

Numerous application timeout logs

Frequent service restarts on the machine
```

However, the underlying causes could be:

```text
CPU being fully utilized

Insufficient memory

Frequent swap usage

High disk IO wait times

Full disk space

Excessive inode counts

Network packet loss or high bandwidth usage

A particular process consuming excessive resources
```

Therefore, the first step in host troubleshooting is usually to check system resources.

---

## IV. Core Indicators for System Resource Inspection

These indicators are commonly used when inspecting system resources:

```text
CPU utilization rate

Load average

Used/Available memory

Buffer/cache

Swap usage

IO wait time

Running queue (r)

Interruptible sleep processes (b)

Process with the highest resource consumption
```

Simple guidelines for interpretation:

```text
High load average
→ Check CPU cores, `top`, `vmstat`

High CPU user space utilization
→ User-space programs are consuming a lot of CPU

High CPU kernel space utilization
→ The kernel is using up significant CPU resources

High CPU wait time
→ Possible high disk IO wait times

Very low available memory
→ Insufficient free memory

High swap in/out usage
→ Frequent swapping is occurring

High running queue value (r)
→ High pressure on the CPU

High number of interruptible sleep processes (b)
→ Possible IO blocking or interruptible sleep issues
```

---

## V. `top`: The First Entry Point for System Resource Inspection

---

## Scenario 1: Using `top` to View Overall System Status

### Command

```bash
top
```

### Function

`top` is the most commonly used tool for real-time system resource monitoring.

It is primarily used to view:

- CPU usage
- Memory usage
- Load average
- Process resource consumption
- Which process is consuming the most CPU or memory

---

## Scenario 2: Common `top` Interaction Keys

After entering `top`, you can use the following shortcut keys:

```text
P
→ Sort by CPU utilization rate

M
→ Sort by memory usage rate

1
→ Display information for each CPU core

c
→ Show the full command line

H
→ Display thread information
```

### Example Operations

To view system resources:

```bash
top
```

After entering, press:

```text
P
```

to sort by CPU utilization.

Press:

```text
M
```

to sort by memory usage.

Press:

```text
1
```

to display details for each CPU core## Scenario 14: Observing System Status Using vmstat

### Command

```bash
vmstat 1 5
```

Explanation:

```text
Samples are taken every 1 second.
A total of 5 samples will be collected.
```

It can also be used as follows:

```bash
vmstat -S M 1 5
```

Parameter Explanation:

```text
-S M
→ Displays some memory data in MB units.
```

---

## Scenario 15: Key Fields of vmstat

Pay special attention to the following fields:

```text
r
b
si
so
us
sy
id
wa
```

Field Explanation:

```text
r
→ Number of processes waiting to run
b
→ Number of processes in uninterruptible sleep, commonly due to IO waits
si
→ Swap in, processes swapped from swap into memory
so
→ Swap out, processes swapped from memory back to swap
us
→ User-mode CPU usage
sy
→ Kernel-mode CPU usage
id
→ CPU idle percentage
wa
→ IO wait time
```

---

## Scenario 16: Common Interpretations of vmstat Results

### High wa value

```text
High wa value
→ Possible high disk IO load
```

Next step:

```text
Investigate disk IO issues
```

You can also use the following commands:

```bash
iostat -x 1 5
```

```bash
iotop -o -P
```

---

### High si / so values

```text
High si / so values
→ Possible memory shortage; the system is frequently swapping pages
```

Next step:

```text
Check for memory issues
```

You can use the following commands:

```bash
free -h
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

---

### High r value

```text
High r value
→ Possible high CPU load
```

Next step:

```text
Investigate CPU issues
```

You can use the following commands:

```bash
top
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

---

### High b value

```text
High b value
→ There may be a large number of processes in uninterruptible sleep.
→ This is commonly seen in cases involving disk IO, storage issues, NFS problems, or block device anomalies.
```

Next step:

```text
Continue investigating by checking dmesg, iostat, and process status
```

---

## Section 10: Preliminary Troubleshooting Steps for System Lag

---

## Scenario 17: What to Check First When the Machine Shows Obvious Lag

Recommended order of checks:

```text
top
→ free -h
→ uptime
→ vmstat 1 5
→ Proceed to investigate CPU/memory/disk IO/process issues based on the results
```

Corresponding commands:

```bash
top
```

```bash
free -h
```

```bash
uptime
```

```bash
vmstat 1 5
```

---

## Scenario 18: How to Determine the Cause Based on the Results

### CPU Load Issue

Phenomena:

```text
High us value in top
High r value in vmstat
Load level consistently higher than the number of CPU cores
```

Next step:

```text
Investigate processes with high CPU usage
```

Follow-up commands:

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

---

### Memory Load Issue

Phenomena:

```text
Very low available memory in free output
Continuous increase in swap usage
High si / so values in vmstat
```

Next step:

```text
Check for processes with high memory consumption
```

Follow-up commands:

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

---

### Disk IO Load Issue

Phenomena:

```text
High wa value in top and vmstat
High b value in vmstat
```

Next step:

```text
Investigate disk IO issues
```

Follow-up commands:

```bash
iostat -x 1 5
```

```bash
iotop -o -P
```

---

### High System Load but Not High CPU Usage

Phenomena:

```text
High load level
Low us value in CPU metrics
High wa or b values
```

Possible causes:

```text
Disk IO bottlenecks
Storage issues
NFS delays
A large number of processes in D-state
File system problems
```

You can also check the following commands:

```bash
vmstat 
```