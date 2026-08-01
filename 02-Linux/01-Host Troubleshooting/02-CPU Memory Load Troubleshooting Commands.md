# 02-CPU, Memory, and Load Troubleshooting Commands

#Linux #Transport #TheBarrier. #CPU #Memory #Load #top #ps #mpstat #pidstat #pmap #vmstat #PerformanceCheck.

---

## Recommended Path

01-Linux Foundation and Host Maintenance/01-Host Troubleshooting/02-CPU, Memory, and Load Troubleshooting Commands.md

---

## Section 1: Document Description

This document organizes Linux host commands related to **CPU, memory, and load** troubleshooting.

Key focuses include:

- CPU usage rate troubleshooting
- High CPU process location
- Multi-core CPU usage
- Thread-level CPU troubleshooting
- Memory usage troubleshooting
- High memory process location
- Process memory mapping viewing
- Load average understanding
- Judgment of high load but low CPU
- `top`
- `ps`
- `mpstat`
- `pidstat`
- `free`
- `pmap`
- `vmstat`
- `uptime`

The goal is:

To quickly determine if CPU is abnormal

→ Locate high CPU processes

→ Determine if memory is insufficient

→ Locate high memory processes

→ Understand the relationship between load average and CPU core count

→ Distinguish between CPU pressure, memory pressure, and IO wait causing high load

---

## Section 2: CPU, Memory, and Load Troubleshooting Overview

Troubleshooting CPU, memory, and load should not rely on a single command.

Recommended order:

```text
uptime
→ Look. load average

top
→ Look. CPU, memory, process occupancy

free -h
→ See memory and swap

vmstat 1 5
→ Look. rI don't know.bI don't know.siI don't know.soI don't know.wa

ps Sort
→ Find specific heights. CPU / High Memory Process

mpstat / pidstat
→ Look further. CPU Core and process-level sampling
```

CommonDiversion:

```text
load High + CPU us High
→ Rate CPU Calculate pressure

load High + wa High
→ Probably a disk. IO Wait

load High + b High
→ There's probably a lot of uninterrupted sleep.

available Low + swap si/so High
→ Memory pressure is high.

Single process CPU High
→ Enter process level check

High memory of individual processes
→ Access to memory leaks or cache anomalies
```

---

## Section 3: Load Troubleshooting: uptime and load average

---

## Scenario 1: View System Load

### Command

```bash
uptime
```

### Purpose

`uptime` Used to view:

- System uptime
- Current logged-in user count
- Load average

Common output:

```text
load average: 0.52, 0.60, 0.70
```

Meaning:

```text
First value
→ Recent 1 Average load of minutes

Second value
→ Recent 5 Average load of minutes

Third value
→ Recent 15 Average load of minutes
```

---

## Scenario 2: View CPU Core Count

### Command

```bash
nproc
```

Or:

```bash
lscpu
```

### Notes

Load average must be judged in combination with CPU core count.

Do not judge abnormality just by seeing load is 8.

For example:

```text
2 The nuclear machine,load Long term 8
→ It's stressful.

16 The nuclear machine,load Long term 8
→ It doesn't have to be unusual. It needs to be combined. CPUI don't know.IOMemory judgement
```

---

## Scenario 3: Load Average Judgment Method

### Judgment Logic

```text
load Long term less than CPU Numerical
→ Usually pressure is acceptable.

load Long-term proximity CPU Numerical
→ Needing attention.

load Long-term significant greater than CPU Numerical
→ System pressure is high. We need to keep checking.
```

### Example

```text
4 The nuclear machine,load Long term 1~2
→ Usually normal.

4 The nuclear machine,load Long term 4 Around
→ Approach full load

4 The nuclear machine,load Long term 10 Above
→ It's stressful. We need to keep checking.
```

---

## Scenario 4: High Load Does Not Necessarily Mean High CPU

High load may come from:

```text
CPU Calculate pressure
Disk IO Block
A lot of sleepless processes.
NFS / Storage anomaly
Too many processes
Thread block
kernel waiting.
```

Therefore, when load is high, do not only check CPU next. Also check:

```bash
top
```

```bash
vmstat 1 5
```

If disk IO is suspected, then check:

```bash
iostat -x 1 5
```

---

## Section 4: top: CPU and Memory Troubleshooting Entry Point

---

## Scenario 5: Use top to View Overall Resources

### Command

```bash
top
```

### Purpose

`top` Used for real-time viewing of:

- Load average
- CPU usage rate
- Memory usage
- Swap usage
- Process CPU usage
- Process memory usage

---

## Scenario 6: Common Interactive Keys in top

After entering `top`, common keys:

```text
P
→ Press CPU Sort Usage

M
→ Sort by memory usage

1
→ Show Every CPU Core

c
→ Show full command line

H
→ Show threads
```

---

## Scenario 7: Understanding the CPU Field in top

Common fields:

```text
us
→ User State CPU Usage

sy
→ kernel CPU Usage

id
→ CPU Free Proportion

wa
→ IO wait, wait for disk IO Yes. CPU Time

hi
→ Hard Break

si
→ Soft Break

st
→ The virtual environment was stolen by the host. CPU Time
```

Common judgments:

```text
us High
→ Application consumption CPU More

sy High
→ More kernel consumption, possibly associated with system calls, networks, disks, drivers

wa High
→ CPU Waiting for disk IOMaybe the disk's under pressure.

id Low
→ CPU Little time.

st High
→ The competition for virtual host resources may be more pronounced.
```

---

## Scenario 8: Understanding the Memory Field in top

Common fields:

```text
total
→ Total RAM

free
→ Full Free Memory

used
→ Used memory

buff/cache
→ Buffer and Cache

available
→ Estimate Available Memory
```

Focus:

```text
Don't just look. free
More than that. available
```

Reason:

```text
Linux Use empty memory as much as possible for cache
buff/cache High isn't necessarily a problem.
available It's all the more important to be alert.
```

---

## Section 5: CPU Troubleshooting: Locating High CPU Processes

---

## Scenario 9: View Processes Sorted by CPU Usage

### Command

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

### Purpose

View processes in descending order of CPU usage rate.

Suitable for quickly locating:

```text
Which process is most important? CPU
```

### Field Explanation

```text
pid
→ Process ID

ppid
→ Parent Process ID

cmd
→ Start Command

%mem
→ Memory ratio

%cpu
→ CPU Percentage
```

---

## Scenario 10: View More High CPU Processes

### Command

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -n 20
```

### Notes

Default `head` only shows the first 10 lines.

If you want to view more processes, you can use:

```bash
head -n 20
```

Or:

```bash
head -n 30
```

---

## Scenario 11: View Details of a Specific Process

### Command

```bash
ps -fp PID
```

Example:

```bash
ps -fp 12345
```

### Purpose

View detailed information of the specified PID.

---

## Scenario 12: View Process Startup Command

### Command

```bash
cat /proc/PID/cmdline
```

Example:

```bash
cat /proc/12345/cmdline
```

If the output is crowded, you can use:

```bash
tr '\0' ' ' < /proc/12345/cmdline
```

### Purpose

Suitable for confirming:

```text
Which service started the process?
What's the start parameter?
Is it a script or something? Java Process
```

---

## Scenario 13: View Process Working Directory

### Command

```bash
ls -l /proc/PID/cwd
```

Example:

```bash
ls -l /proc/12345/cwd
```

### Purpose

View the current working directory of the process.

Suitable for determining which application directory the process belongs to.

---

## Scenario 14: View Executable File of a Process

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

## Section 6: mpstat: View Multi-core CPU Usage

---

## Scenario 15: View CPU Core Usage Rate

### Command

```bash
mpstat -P ALL 1 5
```

### Common Parameters

```text
-P ALL
→ View All CPU Core

1 5
→ Every 1 Sampling in seconds 1 Number of times, sampled 5 Minor
```

### Purpose

Suitable for viewing:

```text
Is it something? CPU The core is full.
Whether or not CPU Uneven use
Is it a whole? CPU They're busy.
```

---

## Scenario 16: What to Do if mpstat Is Not Installed

`mpstat` Usually comes from the `sysstat` package.

Ubuntu/Debian:

```bash
apt install -y sysstat
```

RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y sysstat
```

Or:

```bash
dnf install -y sysstat
```

Install and then execute:

```bash
mpstat -P ALL 1 5
```

---

## Scenario 17: Common Judgments in mpstat

### A Single Core Is High

```text
Some CPU Core long term 100%
Other core low
→ Could be a one-way program bottleneck.
```

### All Cores Are High

```text
All CPU Core us Both tall.
→ Apply whole to calculate pressure
```

### sy Is High

```text
sy High
→ More internal nuclear consumption
→ Possible and system calls, network package processing, disks IODriver-related
```

### wa Is High

```text
wa High
→ CPU Wait IO
→ Keep checking the disk. IO
```

---

## Section 7: pidstat: Process-level CPU Sampling

---

## Scenario 18: View Process CPU Usage

### Command

```bash
pidstat 1 5
```

### Meaning

```text
Every 1 Sampling in seconds 1 Minor
Cosampling 5 Minor
```

### Purpose

Compared to the instant results of `ps`, `pidstat` is more suitable for observing the CPU usage trend of a process over a period of time.

---

## Scenario 19: View CPU Usage of a Specific Process

### Command

```bash
pidstat -p PID 1 5
```

Example:

```bash
pidstat -p 12345 1 5
```

### Purpose / Explanation

# Monitoring CPU Usage for a Specific Process Continuously

---

## Scenario 20: Viewing Thread-Level CPU

### Command

```bash
pidstat -t -p PID 1 5
```

Example:

```bash
pidstat -t -p 12345 1 5
```

### Purpose

Suitable for troubleshooting:

```text
Which threads are the most important inside a process? CPU
```

Commonly seen in:

```text
Java Process
Multi-line service
High even application
```

---

## VIII. Thread-Level CPU Troubleshooting

---

## Scenario 21: top Display Threads

### Command

```bash
top
```

After entering, press:

```text
H
```

### Purpose

Displays thread-level CPU usage.

Suitable for confirming:

```text
Is there a line full? CPU
```

---

## Scenario 22: View Threads of a Specific Process

### Command

```bash
ps -T -p PID
```

Example:

```bash
ps -T -p 12345
```

### Purpose

Views threads under a specific process.

---

## Scenario 23: Thread ID to Hexadecimal Conversion

In Java troubleshooting, it's often necessary to convert thread ID to hexadecimal to match `jstack`'s `nid`.

### Command

```bash
printf "%x\n" ThreadID
```

Example:

```bash
printf "%x\n" 12346
```

### Notes

Suitable for Java high CPU troubleshooting.

Common path:

```text
top -H or pidstat -t Find high. CPU Thread ID
→ printf Turn Hexadecimal
→ jstack Organisation nid
→ Positioning Code.
```

---

## IX. Memory Troubleshooting: free and ps

---

## Scenario 24: View Memory Usage

### Command

```bash
free -h
```

### Common Parameters

```bash
free -h
```

```bash
free -m
```

```bash
free -g
```

Parameter Explanation:

```text
-h
→ Human Readable Display

-m
→ Press MB Show

-g
→ Press GB Show
```

---

## Scenario 25: Focus on free Output

Focus on:

```text
available

buff/cache

swap
```

Field Understanding:

```text
used
→ Used memory

free
→ Full Free Memory

buff/cache
→ Buffer and Cache

available
→ System estimates can also provide memory for new processes

swap
→ Use of exchange of partitions or exchange of documents
```

---

## Scenario 26: What Does Low available Mean

If `available` is very low, and the system experiences:

```text
Reaction slow.
SSH Login Slow
Apply delay elevation
swap Sustained growth
OOM Records
```

It may indicate memory pressure.

Next Steps:

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

---

## Scenario 27: View Processes by Memory Usage

### Command

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

### Purpose

Views processes sorted by memory usage in descending order.

Suitable for quickly locating:

```text
Which process accounts for most memory
```

---

## Scenario 28: View More High-Memory Processes

### Command

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -n 20
```

---

## Scenario 29: View Process Memory Mapping

### Command

```bash
pmap PID
```

Example:

```bash
pmap 12345
```

View summary:

```bash
pmap -x 12345 | tail
```

### Purpose

`pmap` is used to view process memory mapping.

Suitable for analyzing:

```text
Which libraries did the process load?
Different Sector Occupancy
Process approximate memory distribution
```

---

## Scenario 30: What to Do if pmap Is Not Installed

`pmap` usually comes from `procps` or `procps-ng` packages.

Ubuntu/Debian:

```bash
apt install -y procps
```

RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y procps-ng
```

Or:

```bash
dnf install -y procps-ng
```

---

## X. Swap Troubleshooting

---

## Scenario 31: View Swap Usage

### Command

```bash
free -h
```

Or:

```bash
swapon --show
```

### Purpose

Checks if swap is enabled and views swap usage.

---

## Scenario 32: vmstat to View Swap Paging

### Command

```bash
vmstat 1 5
```

Focus on:

```text
si
so
```

Field Explanation:

```text
si
→ swap inFrom swap Transfer Memory

so
→ swap out, switch from memory to swap
```

---

## Scenario 33: Judging Swap Usage

### Minor Swap Usage

```text
swap Small use
→ Not necessarily a serious problem.
```

### Continuous High si/so

```text
si / so Longer
→ The system is changing pages.
→ Memory pressure is high.
```

### Common Impact

```text
The system's obvious.
Slower application response
load Raise
Disk IO Higher
```

---

## Scenario 34: View OOM Records

### Command

```bash
dmesg -T | grep -i oom
```

Or:

```bash
journalctl -k | grep -i oom
```

You can also check kill:

```bash
dmesg -T | grep -i kill
```

### Purpose

Checks if OOM Killer killed a process.

---

## XI. vmstat: Comprehensive Judgment of CPU/Memory/Load

---

## Scenario 35: Sample System Status

### Command

```bash
vmstat 1 5
```

### Field Focus

```text
r
→ Number of processes waiting to run

b
→ Non-interruptible sleep processes, common in IO Wait

si
→ swap in

so
→ swap out

us
→ User State CPU Usage

sy
→ kernel CPU Usage

id
→ CPU Free Proportion

wa
→ IO wait
```

---

## Scenario 36: How to Judge High r

```text
r High
→ Wait CPU Run many processes
→ Maybe. CPU It's stressful.
```

Next Steps:

```bash
top
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
mpstat -P ALL 1 5
```

---

## Scenario 37: How to Judge High b

```text
b High
→ You can't interrupt sleep.
→ Common on disks IO, storage,NFSAnomalous piece of equipment
```

Next Steps:

```bash
iostat -x 1 5
```

```bash
dmesg -T | tail -n 50
```

---

## Scenario 38: How to Judge High wa

```text
wa High
→ CPU Waiting. IO
→ Not necessarily. CPU Not enough.
→ More likely to be disk or storage slow.
```

Next step into disk I/O troubleshooting:

```bash
iostat -x 1 5
```

```bash
iotop -o -P
```

---

## XII. Common Troubleshooting Scenarios

---

## Scenario 39: High CPU Usage

### Phenomenon

```text
top Medium us High
load Higher
Slower application response
```

### Troubleshooting Commands

```bash
top
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
mpstat -P ALL 1 5
```

```bash
pidstat 1 5
```

### Troubleshooting Approach

```text
Look at the whole thing first. CPU
→ Let's go higher. CPU Process
→ Let's see if it's full.
→ Let's see if it's an anomaly.
→ Final integration of application log and business behaviour analysis
```

---

## Scenario 40: High Memory Usage

### Phenomenon

```text
free Medium available Low
swap Start Growth
System slow down.
It's possible. OOM
```

### Troubleshooting Commands

```bash
free -h
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

```bash
pmap -x PID | tail
```

```bash
vmstat 1 5
```

```bash
dmesg -T | grep -i oom
```

### Troubleshooting Approach

```text
Look first. available
→ Look again. swap
→ Find high memory processes.
→ Let's see if we can. OOM
→ Whether it's business cache, memory leak or normal occupation
```

---

## Scenario 41: High Load but Low CPU

### Phenomenon

```text
uptime Medium load High
top Medium CPU us Not high.
But the system is still Carton.
```

### Possible Causes

```text
Disk IO Waiting high
NFS / Store Carton
Mass D Status Process
swap Page Break
kernel waiting.
```

### Troubleshooting Commands

```bash
vmstat 1 5
```

```bash
top
```

```bash
dmesg -T | tail -n 50
```

If suspecting disk I/O:

```bash
iostat -x 1 5
```

---

## Scenario 42: System Lag

### Troubleshooting Order

```text
top
→ free -h
→ uptime
→ vmstat 1 5
→ ps Sort
→ Proceed with results. CPU / Memory / IO Check.
```

### Commands

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

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

---

## XIII. Production Troubleshooting Notes

---

## 1. Do Not Directly Expand CPU When Seeing High Load

High load may be:

```text
CPU Not enough.
IO Wait
Memory Change Page
D Status Process
Storage anomaly
```

Should first determine:

```text
r Are you tall?
wa Are you tall?
b Are you tall?
si / so Are you tall?
```

---

## 2. Do Not Assume Memory Is Insufficient Just Because free Is Low

Linux uses memory for caching.

More attention should be paid to:

```text
available
swap
si / so
OOM Records
Operational performance
```

---

## 3. Do Not Directly Kill High CPU Processes

High CPU processes may be core business processes.

Before handling, confirm:

```text
Which service does the process belong to?
Can Restart
Is there a lead?
Impact on production
Is it necessary to keep the scene?
```

First check:

```bash
ps -fp PID
```

```bash
tr '\0' ' ' < /proc/PID/cmdline
```

Then decide whether to handle.

---

## 4. Sample First, Then Judge

Single command results may be accidental.

Recommend using sampling commands:

```bash
vmstat 1 5
```

```bash
mpstat -P ALL 1 5
```

```bash
pidstat 1 5
```

Continuous observation is more reliable.

---

## 5. Use Basic Commands First When Tools Are Missing

If you don't have:

```text
mpstat
pidstat
iostat
pmap
```

Use first:

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

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

---

## Fourteen, Common Commands in This Article

---

## Load Check

```bash
uptime
```

```bash
nproc
```

```bash
lscpu
```

---

## Overall Resource Check

```bash
top
```

```bash
htop
```

```bash
vmstat 1 5
```

```bash
vmstat -S M 1 5
```

---

## CPU Troubleshooting

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -n 20
```

```bash
mpstat -P ALL 1 5
```

```bash
pidstat 1 5
```

```bash
pidstat -p PID 1 5
```

```bash
pidstat -t -p PID 1 5
```

```bash
ps -T -p PID
```

---

## Memory Troubleshooting

```bash
free -h
```

```bash
free -m
```

```bash
free -g
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -n 20
```

```bash
pmap PID
```

```bash
pmap -x PID | tail
```

---

## Swap / OOM Troubleshooting

```bash
swapon --show
```

```bash
vmstat 1 5
```

```bash
dmesg -T | grep -i oom
```

```bash
journalctl -k | grep -i oom
```

```bash
dmesg -T | grep -i kill
```

---

## Process Details

```bash
ps -fp PID
```

```bash
cat /proc/PID/cmdline
```

```bash
tr '\0' ' ' < /proc/PID/cmdline
```

```bash
ls -l /proc/PID/cwd
```

```bash
ls -l /proc/PID/exe
```

---

## Tool Installation

Install sysstat on Ubuntu/Debian:

```bash
apt install -y sysstat
```

Install sysstat on RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y sysstat
```

Or:

```bash
dnf install -y sysstat
```

Install procps on Ubuntu/Debian:

```bash
apt install -y procps
```

Install procps-ng on RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y procps-ng
```

Or:

```bash
dnf install -y procps-ng
```

---

## Fifteen, One-Sentence Summary

The core of CPU, memory, and load troubleshooting is:

```text
Look at the whole thing first.
→ Find another process.
→ Look at the sample.
→ Finalized with business and log judgements
```

Common troubleshooting entry points:

```text
uptime
→ Look. load average

top
→ Look. CPU, memory, process

free -h
→ Look. available and swap

vmstat 1 5
→ Look. rI don't know.bI don't know.siI don't know.soI don't know.wa

ps --sort=-%cpu
→ Find Gao. CPU Process

ps --sort=-%mem
→ High Memory Process

mpstat
→ Look at the nucleus. CPU Use

pidstat
→ See process or linear level CPU
```

Judgment mnemonic:

```text
us High
→ Apply CPU High

sy High
→ High kernel consumption

wa High
→ IO Waiting high

r High
→ CPU Run queue pressure high

b High
→ Maybe. IO Blocked or otherwise D Status Process

available Low
→ Insufficient available memory

si / so High
→ swap The change is obvious.

load Gaudan. CPU Not high.
→ Priority suspicion IOI don't know.swapI don't know.D Status or storage issues
```

Production recommendations:

```text
Don't just look. load
Don't just look. free
Don't be direct. kill Process
Don't come up and restart the machine.
Take a sample, then decide.
Stay on site, then deal with it.
```