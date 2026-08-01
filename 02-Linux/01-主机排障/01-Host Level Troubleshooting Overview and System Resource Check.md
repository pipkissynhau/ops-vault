# 01-Host-Level Troubleshooting Overview and System Resource Inspection

#Linux #Transport #TheBarrier. #HostTransport #SystemResources #CPU #Memory #Load #SystemBarriers

---

## Recommended Path

01-Linux Foundation and Host Maintenance/01-Host Troubleshooting/01-Host-Level Troubleshooting Overview and System Resource Inspection.md

---

## I. Document Description

This document compiles **Host-Level Troubleshooting Overview and System Resource Inspection** commonly used commands, written based on common troubleshooting scenarios in production environments.

This article focuses on:

- Host-level troubleshooting overall approach
- System resource inspection entry points
- Initial judgment of CPU / memory / load
- `top`
- `htop`
- `free`
- `uptime`
- `vmstat`
- Initial troubleshooting chain for system lag
- Summary of commonly used resource inspection commands

This article is the first in the host troubleshooting series, mainly addressing:

```text
One. Linux When the mainframe is not normal, what should we look at first?
```

The goal is:

To establish a host troubleshooting sequence

→ To quickly determine if the system has resource pressure

→ To preliminarily identify abnormal directions such as CPU, memory, load, IO wait, swap

→ To lay the foundation for subsequent CPU, memory, disk, network, and service troubleshooting

---

## II. Host-Level Troubleshooting Overall Approach

Do not restart services or kill processes immediately when troubleshooting host-level issues.

Recommended sequence:

```text
The phenomenon is confirmed.

→ Resource QueriesCPU / Memory / Load / Disk / Network)

→ Process Query

→ Port Check

→ Route / Firewall / DNS Check.

→ Log Check

→ Position Roots

→ Process and verify
```

Common approaches can be summarized as:

```text
First, resources.

→ Look at the process. / Services

→ Look at the port. / Network

→ Look at the rules. / Route

→ Last look at the log.
```

---

## III. Why Host Troubleshooting Should Start with Resource Inspection

Many production failures appear as business anomalies, for example:

```text
Slow interface

Services do not respond

SSH Login to Cardon.

Database connection timed out

Pod Or the container starts slow.

Nginx Back 502 / 504

Numerous applied logs timeout

There's been a lot of rebooting on the machine.
```

But the underlying cause may be:

```text
CPU Filled up.

Insufficient memory

swap Frequent page changes

Disk IO wait High

Disk Space Full

inode Full

Network bag or bandwidth full

Unusual occupation of resources by a process
```

Therefore, the first step in host troubleshooting is usually to check system resources.

---

## IV. Core Metrics for System Resource Inspection

Common metrics inspected for system resources:

```text
CPU Usage

load average

Memory used / available

buff/cache

swap Use

IO wait

Run the queue r

Do not interrupt the sleep process b

Process of maximum resource occupancy
```

Simple judgment directions:

```text
load High
→ Look. CPU Number,topI don't know.vmstat

CPU us High
→ User state program consumption CPU More

CPU sy High
→ Internal nuclear consumption CPU More

CPU wa High
→ Disk IO Waiting may be higher

available Low
→ Insufficient available memory

swap si / so High
→ The system is changing pages.

r High
→ CPU Run queue pressure high

b High
→ Maybe. IO Blocked or sleepless
```

---

## V. top: First Entry for System Resource Inspection

---

## Scenario 1: Use top to View System Status

### Command

```bash
top
```

### Purpose

`top` is the most commonly used tool for real-time system resource inspection.

Mainly used for viewing:

- CPU usage
- Memory usage
- Load average
- Process resource consumption
- Which process consumes the most CPU
- Which process consumes the most memory

---

## Scenario 2: Common Interactive Keys in top

After entering `top`, you can use the following shortcuts:

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

### Common Operation Examples

View system resources:

```bash
top
```

After entering, press:

```text
P
```

To sort by CPU.

After entering, press:

```text
M
```

To sort by memory.

After entering, press:

```text
1
```

To display CPU core usage.

After entering, press:

```text
c
```

To display full command line.

After entering, press:

```text
H
```

To display threads.

---

## Scenario 3: What to Focus on in top

Focus on:

```text
load average

CPU us / sy / wa

Memory used / free / buff/cache

Swap Use

Which process is occupied? CPU Or highest memory
```

### Load Average

`load average` represents the system's average load.

It generally displays 3 values:

```text
1 Average load of minutes

5 Average load of minutes

15 Average load of minutes
```

Judgment should not be based solely on numbers, but combined with CPU core count.

For example:

```text
4 Nuclear CPUI don't know.load Long term 4 Around
→ Pressure close to full.

4 Nuclear CPUI don't know.load Long term 10 Above
→ It's stressful. We need to keep checking.
```

---

## Scenario 4: Understanding the CPU Field in top

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

## Scenario 5: Understanding the Memory Field in top

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
Linux Use memory as much as possible for cache
buff/cache High isn't necessarily a problem.
available It's all the more important to be alert.
```

---

## VI. htop: A More Intuitive Resource Inspection Tool

---

## Scenario 6: Use htop to View Resources

### Command

```bash
htop
```

### Purpose

`htop` is more intuitive than `top`.

Common advantages:

- Clearer display
- Easier to view CPU cores
- Easier to filter processes
- Easier to sort
- Interactive process termination

### Notes

`htop` may not be pre-installed by default.

If the command is not available, installation may be required.

Ubuntu / Debian:

```bash
apt install -y htop
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y htop
```

Or:

```bash
dnf install -y htop
```

### Usage Recommendations

In production environments for temporary troubleshooting, `top` is more general.

If tool installation is allowed, `htop` is more suitable for quick observation.

---

## VII. free: View Memory Usage

---

## Scenario 7: View Memory Usage

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

Parameter explanations:

```text
-h
→ Human Readable Display

-m
→ Press MB Show

-g
→ Press GB Show
```

---

## Scenario 8: Focus on Key Items in free Output

Focus on:

```text
used

free

buff/cache

available

swap
```

### Field Understanding

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

## Scenario 9: Is High buff/cache Always a Problem

Conclusion:

```text
buff/cache High doesn't necessarily matter.
```

Reason:

```text
Linux The temporarily unused memory will be used to cache filesystem data
It makes reading and writing more efficient.
```

What should be more focused on is:

```text
available Sufficient
swap Sustained growth
Is there a carton in the system?
Is there any? OOM Records
```

---

## Scenario 10: How to View Swap Usage

If `swap` has minimal usage, it's not necessarily a serious issue immediately.

But if the following occurs:

```text
swap Sustained growth

si / so Longer

The system's obvious.

Slower application response

load Raise
```

We need to be cautious about memory pressure.

The next step can combine:

```bash
vmstat 1 5
```

Continue observation:

```text
si

so
```

---

## VIII. uptime: View Runtime and Load

---

## Scenario 11: View System Runtime and Load

### Command

```bash
uptime
```

### Purpose

`uptime` is used to view:

- System runtime
- Current number of logged-in users
- Load average

Common output content:

```text
load average: 0.52, 0.60, 0.70
```

Meaning:

```text
Recent 1 Average load of minutes

Recent 5 Average load of minutes

Recent 15 Average load of minutes
```

---

## Scenario 12: How to Judge Load Average

Load cannot be judged alone; it should be combined with CPU core count.

Check CPU core count:

```bash
nproc
```

Or:

```bash
lscpu
```

Simple judgment:

```text
load Long term less than CPU Numerical
→ Usually pressure is acceptable.

load Long-term proximity CPU Numerical
→ Needing attention.

load Long-term significant greater than CPU Numerical
→ System pressure is high. We need to keep checking.
```

Example:

```text
2 The nuclear machine,load Long term 6
→ It's stressful.

8 The nuclear machine,load Long term 6
→ It doesn't have to be unusual. It needs to be combined. CPUI don't know.IOThe process continues.
```

---

## Scenario 13: High Load Doesn't Necessarily Mean CPU Issues

High load may come from:

```text
CPU Calculate pressure

Disk IO Block

A lot of sleepless processes.

Too many processes

Apply thread block

Storage or network file system Carton.
```

Therefore, when load is high, don't only focus on CPU.

It is recommended to continue checking:

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

This command will be organized separately in the subsequent disk IO troubleshooting article.

---

## IX. vmstat: System-wide Status Sampling

---

## Scenario 14: Use vmstat to Observe System Status

### Command

```bash
vmstat 1 5
```

Meaning:

```text
Every 1 Sampling in seconds 1 Minor

Cosampling 5 Minor
```

You can also use:

```bash
vmstat -S M 1 5
```

Parameter explanation: /think

```text
-S M
→ Here. MB Show Partial Memory Data
```

---

## Scene 15: vmstat Key Fields

Focus on:

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

Field Description:

```text
r
→ Number of processes waiting to run

b
→ Non-interruptible sleep processes, common in IO Wait

si
→ swap inFrom swap Transfer Memory

so
→ swap out, switch from memory to swap

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

## Scene 16: vmstat Common Judgments

### wa High

```text
wa High
→ Possible Disk IO Pressure.
```

Next Steps:

```text
Enter Disk IO Check.
```

Subsequent Commands:

```bash
iostat -x 1 5
```

```bash
iotop -o -P
```

---

### si / so High

```text
si / so High
→ There may be a shortage of memory and the system is changing pages frequently
```

Next Steps:

```text
Enter memory check
```

Subsequent Commands:

```bash
free -h
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

---

### r Very High

```text
r High
→ Maybe. CPU Pressure.
```

Next Steps:

```text
Enter CPU Check.
```

Subsequent Commands:

```bash
top
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

---

### b Very High

```text
b High
→ There's probably a lot of uninterrupted sleep.
→ Common on disks IO, storage,NFSAnomalous scene.
```

Next Steps:

```text
Combined dmesgI don't know.iostatProcess status continues to be checked.
```

---

## Ten. Initial Troubleshooting Chain for System Lag

---

## Scene 17: What to Check First When Machine is Clearly Lagging

Recommended Order:

```text
top

→ free -h

→ uptime

→ vmstat 1 5

→ Proceed with results. CPU / Memory / Disk IO / Process Query
```

Corresponding Commands:

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

## Scene 18: How to Split Based on Results

### CPU Pressure Direction

Phenomenon:

```text
top Medium CPU us High

vmstat Medium r High

load Longer CPU Numerical
```

Next Steps:

```text
Check it out. CPU Process
```

Subsequent Commands:

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

---

### Memory Pressure Direction

Phenomenon:

```text
free Medium available Low

swap Continued growth in use

vmstat Medium si / so High
```

Next Steps:

```text
Check for high memory processes
```

Subsequent Commands:

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head
```

---

### Disk IO Pressure Direction

Phenomenon:

```text
top Medium wa High

vmstat Medium wa High

vmstat Medium b High
```

Next Steps:

```text
Check Disk IO
```

Subsequent Commands:

```bash
iostat -x 1 5
```

```bash
iotop -o -P
```

---

### High System Load but Low CPU

Phenomenon:

```text
load High

CPU us Not high.

wa or b High
```

Possible Directions:

```text
Disk IO Block

Storage anomaly

NFS Carton.

Mass D Status Process

Document system issues
```

Subsequent Commands to Check:

```bash
vmstat 1 5
```

```bash
dmesg -T | tail -n 50
```

---

## Eleven. Production Troubleshooting Notes

---

## 1. Do Not Rely on Single Metric

For example:

```text
load High
```

Cannot Directly Equal:

```text
CPU Filled up.
```

Also Combine With:

```text
CPU Numerical

top Medium us / sy / wa

vmstat Medium r / b

Disk IO

Memory swap
```

---

## 2. Do Not Restart Immediately

Production troubleshooting should first preserve the scene:

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

If restarted directly, critical scene information may be lost.

---

## 3. Distinguish Between "Short-Term Fluctuation" and "Persistent Anomaly"

Short-term high load doesn't necessarily indicate a fault.

More importantly, observe:

```text
Persistence high

Impact on operations

Whether to accompany log errors

Do we have the resources to run out?

User side feedback available
```

---

## 4. Use Basic Commands When Tools Are Missing

Some production environments may not have installed:

```text
htop

iostat

iotop

mpstat

pidstat
```

But generally have:

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
vmstat
```

Therefore, system resource troubleshooting should first master basic commands.

---

## Twelve. Summary of Common Commands in This Article

---

## System Overall Status

```bash
top
```

```bash
htop
```

```bash
uptime
```

```bash
vmstat 1 5
```

```bash
vmstat -S M 1 5
```

---

## Memory Check

```bash
free -h
```

```bash
free -m
```

```bash
free -g
```

---

## CPU Core Count Check

```bash
nproc
```

```bash
lscpu
```

---

## Initial Troubleshooting Combination for System Lag

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

## Thirteen. One-Sentence Summary

The first step in host-level troubleshooting is:

```text
Look at the whole system.
```

Common Entry Points:

```text
top
→ Look. CPUMemory,loadProcess

free -h
→ See memory and swap

uptime
→ Look at running time and time load average

vmstat 1 5
→ Look. rI don't know.bI don't know.siI don't know.soI don't know.wa System status
```

System Lag Troubleshooting Chain:

```text
top

→ free -h

→ uptime

→ vmstat 1 5

→ Proceed with results. CPU / Memory / Disk IO / Process Query
```

Core Judgment:

```text
us High
→ Application consumption CPU

sy High
→ Higher kernel consumption

wa High
→ Disk IO Waiting high

available Low
→ Insufficient available memory

si / so High
→ swap The change is obvious.

r High
→ CPU Run queue pressure high

b High
→ Could exist. IO Blocked or sleepless
```

Production Recommendations:

```text
Watch and then act.
Stay on site, then deal with it.
Don't look at single indicators.
Don't. load High directly equals CPU High
Don't come up and restart the service.
```