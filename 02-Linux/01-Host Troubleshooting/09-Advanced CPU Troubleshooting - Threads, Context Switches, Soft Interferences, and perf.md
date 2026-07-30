# 09-Advanced CPU Troubleshooting: Threads, Context Switches, Soft Interferences, and perf

# Linux # Operations and Maintenance # SRE # CPU Troubleshooting # Performance Analysis # Context Switches # Soft Interferences # Hard Interferences # perf # Flame Graphs # Thread Troubleshooting

---

## Recommended Reading Path

01-Linux Basics and Host Operations and Maintenance/01-Host Troubleshooting/09-Advanced CPU Troubleshooting: Threads, Context Switches, Soft Interferences, and perf.md

---

## I. Document Overview

This document outlines advanced methods for troubleshooting high CPU usage on Linux hosts. The focus is not just on checking the overall CPU utilization but also on analyzing:

- Where exactly the high CPU usage originates
- How to diagnose high user-space CPU usage
- How to diagnose high kernel-space CPU usage
- How to determine if a single core is being fully utilized
- How to identify threads in multi-threaded programs that are causing high CPU usage
- How to determine if context switches are excessive
- How to determine if soft interferences are high
- How to determine if hard interferences are high
- How to analyze high CPU usage caused by network packet processing
- Using `perf top`, `perf record`, and `perf report`
- Basic understanding of flame graphs
- Approaches for identifying high-CPU-consuming threads in Java applications
- Precautions for troubleshooting high CPU usage in production environments

This document is part of the advanced performance analysis series within the host troubleshooting category.

In the previous article #02, basic commands for monitoring CPU, memory, and load were covered. This article delves deeper into CPU troubleshooting.

The objectives are:

- To distinguish between user-space, kernel-space, soft-interference, and hard-interference CPU consumption
- To determine whether the bottleneck lies in a single core or overall CPU pressure
- To identify processes and threads that are consuming high amounts of CPU
- To assess if context switches are abnormal
- To evaluate if there are issues with soft interferences
- To use perf for function-level hot-spot analysis
- To develop the skills required for in-depth CPU troubleshooting in advanced SRE interviews

---

## II. High CPU Usage Does Not Always Indicate the Same Problem

When you notice high CPU usage, it is not sufficient to draw immediate conclusions.

High CPU usage can be caused by various factors:

```text
High computational load in business applications
Sudden increase in request volume
Code infinite loops
Excessive log output
Compression, encryption, or encoding tasks
Frequent garbage collection
Too many threads resulting in high scheduling overhead
Excessive system calls
High kernel-space consumption
High network soft interferences
Kernel-related disk I/O overhead
Containerization or virtualization processes competing for CPU resources
A single-threaded program using up a entire core
Multiple processes competing for CPU resources simultaneously
```

Therefore, when troubleshooting high CPU usage, it is necessary to break down the issue into smaller components:

```text
Is the overall CPU usage high?
→ Is it all cores that are affected, or just one core?
→ Is it user-space or kernel-space consumption?
→ Are ordinary processes causing the high usage, or are soft interferences at fault?
→ Is it a specific process or the entire system?
→ Is it a particular thread within a process, or are all threads affected?
→ Is the high usage temporary or persistent?
```

---

## III. General Path for CPU Troubleshooting

Here is a recommended sequence for advanced CPU troubleshooting:

```text
top
→ Check overall CPU and load levels
mpstat -P ALL
→ View CPU usage for each core
vmstat
→ Monitor trends in the run queue, context switches, and system calls
pidstat
→ Analyze process-level CPU usage
pidstat -t
→ Analyze thread-level CPU usage
pidstat -w
→ Examine context switch statistics
mpstat -I
→ Check for interrupt and soft interference counts
/proc/softirqs
→ View distribution of soft interferences
/proc/interrupts
→ View distribution of hard interferences
perf top / perf record
→ Identify function-level hot spots
```

Common branches in the troubleshooting process:

```text
High user-space usage
→ The application is consuming a lot of CPU resources
High kernel-space usage
→ The kernel is using up significant CPU time, possibly due to system calls, network activities, disk operations, or kernel-related processes
High soft-interference usage
→ Soft interferences are causing high CPU consumption, often related to network packet processing
High hard-interference usage
→ Hard interferences are consuming a lot of CPU time, possibly related to network cards, interrupts, or hardware devices
High virtualization environment CPU usage
→ The host machine is stealing CPU resources from the virtualized instances
100% utilization of a single core
→ Either a single-threaded bottleneck or concentrated interrupt activities
High context## Scenario 15: Identifying High-CPU-Consuming Threads
```

```bash
top -H -p 12345
```

Or:

```bash
pidstat -t -p 12345 1 5
```

Assume the thread ID is:

```text
12388
```

---

## Scenario 16: Converting Thread ID to Hexadecimal

```bash
printf "%x\n" 12388
```

Assume the output is:

```text
3064
```

---

## Scenario 17: Using jstack to View Thread Stack

```bash
jstack 12345 > /tmp/jstack-12345.txt
```

Search for the nid:

```bash
grep -n "3064" /tmp/jstack-12345.txt
```

Or:

```bash
grep -n "nid=0x3064" /tmp/jstack-12345.txt
```

View the context:

```bash
grep -n -A 30 -B 5 "nid=0x3064" /tmp/jstack-12345.txt
```

---

## Scenario 18: Common Causes of High CPU Usage in Java

Common causes include:

```text
 Infinite loops

 Performance issues with regular expressions

 Frequent garbage collection (GC)

 Large amounts of JSON serialization/deserialization

 Encryption/decryption operations

 Compression/decompression tasks

 Excessive log output

 Abnormalities in thread pools

 Lock contention

 Handling large objects

 Iterating through massive data sets in collections
```

Further consider:

```text
 Application logs

 GC logs

 Number of requests

 Code changes

 Recently released updates

 Thread stack information
```

To determine the root cause.

---

## IX. Troubleshooting Context Switching Issues

---

## Scenario 19: What is Context Switching?

Context switching can be simply understood as:

```text
The CPU switching from one task to another
```

Common types include:

```text
 Process switching

 Thread switching

 User mode/kernel mode switching

 Interrupt-induced switching
```

Context switching itself is not necessarily a problem, but excessive context switching can lead to additional overhead:

```text
 More CPU time is spent on scheduling

 Actual business computation time is reduced

 System response times slow down

 The system load increases

 The percentage of CPU time spent on scheduling also rises
```

---

## Scenario 20: Using vmstat to Monitor Context Switching

```bash
vmstat 1 5
```

Key fields include:

```text
cs
→ Number of context switches per second

in
→ Number of interrupts per second
```

If you observe:

```text
 Continuously high values for cs
→ Possible frequent context switching

 Continuously high values for in
→ Possible frequent interruptions
```

Note:

```text
 Whether cs is abnormally high should be evaluated in conjunction with the number of CPU cores, the type of application, and historical baseline data
```

---

## Scenario 21: Using pidstat to Monitor Process Context Switching

```bash
pidstat -w 1 5
```

To monitor a specific process:

```bash
pidstat -w -p PID 1 5
```

Example:

```bash
pidstat -w -p 12345 1 5
```

Field explanations include:

```text
cswch/s
→ Voluntary context switches per second

nvcswch/s
→ Involuntary context switches per second
```

---

## Scenario 22: Voluntary vs. Involuntary Context Switches

```text
 Voluntary context switches (cswch/s)
→ The process voluntarily relinquishes control of the CPU
→ Commonly occurs while waiting for I/O operations, locks, network requests, or during sleep states

 Involuntary context switches (nvcswch/s)
→ The system forces the process to switch contexts
→ Often happens when a CPU time slice expires or due to intense CPU competition
```

Common reasons for high values:

```text
 High levels of cswch/s may indicate frequent waits, lock conflicts, I/O delays, or network issues

 High levels of nvcswch/s could suggest intense CPU competition, too many threads, or insufficient CPU resources
```

---

## Scenario 23: Common Causes of Frequent Context Switching

Common causes include:

```text
 Excessive number of threads

 Overly large thread pool configurations

 Frequent creation and destruction of threads

 Severe lock contention

 A large number of short-lived connections

 Numerous system calls

 High-concurrency network requests

 Frequent I/O delays

 Insufficient container CPU limits

 Frequent sleep/wake-up events within the application
```

Troubleshooting commands includeRegular ps/top only allows you to view processes, not functions.Sample data in as short a time as possible. Avoid prolonged sampling:

```bash
strace -p PID
```

This will attach to the core process online.

---

## 4. Single-data points may not be reliable

For CPU issues, continuous sampling is necessary:

```bash
vmstat 1 5
```

```bash
mpstat -P ALL 1 5
```

```bash
pidstat 1 5
```

```bash
pidstat -w 1 5
```

---

## 5. High CPU usage must be considered in context with business activities

Check for the following:

```text
Was it just released?
Has there been an increase in traffic?
Were any scheduled tasks initiated?
Are backup or compression tasks running?
Is there a log overflow?
Could slow database queries be causing application threads to wait?
Are there any exceptions in external calls being retried?
```

---

## Nineteen: Summary of Common Commands

---

## Overall CPU Usage

```bash
top
```

```bash
top -b -n 1 | head -n 50
```

```bash
uptime
```

```bash
nproc
```

```bash
lscpu
```

```bash
vmstat 1 5
```

---

## Multi-core CPU Usage

```bash
mpstat -P ALL 1 5
```

```bash
top
```

In top, press:

```text
1
```

---

## Process-specific CPU Usage

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head -n 20
```

```bash
pidstat 1 5
```

```bash
pidstat -p PID 1 5
```

---

## Thread-specific CPU Usage

```bash
top -H -p PID
```

```bash
pidstat -t -p PID 1 5
```

```bash
ps -T -p PID
```

```bash
ps -T -p PID -o pid,tid,psr,pcpu,pmem,stat,comm
```

---

## High CPU Usage in Java Applications

```bash
ps -ef | grep java | grep -v grep
```

```bash
pgrep -a java
```

```bash
top -H -p PID
```

```bash
printf "%x\n" Thread ID
```

```bash
jstack PID > /tmp/jstack-PID.txt
```

```bash
grep -n "nid=0xHexadecimal Thread ID" /tmp/jstack-PID.txt
```

---

## Context Switching

```bash
vmstat 1 5
```

```bash
pidstat -w 1 5
```

```bash
pidstat -w -p PID 1 5
```

```bash
pidstat -t -w -p PID 1 5
```

```bash
cat /proc/PID/status | grep Threads
```

```bash
ps -T -p PID | wc -l
```

---

## Soft Interrupts

```bash
mpstat -P ALL 1 5
```

```bash
cat /proc/softirqs
```

```bash
watch -n 1 cat /proc/softIRQs
```

```bash
ps -ef | grep ksoftirqd
```

```bash
sar -n DEV 1 5
```

```bash
iftop -n
```

```bash
ss -s
```

```bash
ip -s link
```

---

## Hard Interrupts

```bash
cat /proc/interrupts
```

```bash
watch -n 1 cat /proc/interrupts
```

```bash
systemctl status irqbalance
```

---

## Perf Tool Usage

```bash
perf --version
```

```bash
perf top
```

```bash
perf top -p PID
```

```bash
perf record -p PID -g -- sleep 30
```

```bash
perf record -F 99 -p PID -g -- sleep 30
```

```bash
perf record -a -g -- sleep 30
```

```bash
perf report
```

```bash
perf script > out.perf
```

---

## Strace Tool Usage

```bash
strace -c -p PID
```

---

## Container CPU Usage

```bash
docker stats
```

```bash
docker top Container ID
```

```bash
docker inspect -f "{{.State.Pid}}" Container ID
```

```bash
ps -fp $()
```