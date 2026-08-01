# 09-Advanced CPU Troubleshooting: Threads, Context Switches, Soft Interrupts, and perf

#Linux #Transport #SRE #CpuCheck #PerformanceAnalysis #ContextSwitch #SoftBreak #HardBreak #perf #FlamesChart #Linework

---

## Recommended Path

01-LinuxBase and Host Movement/01-Host barriers/09-Advanced CPU Scanning: threads, context switching, soft break and perf.md

---

## One: Document Overview

This document organizes advanced CPU troubleshooting methods for Linux hosts, focusing not just on CPU utilization rates but further analyzing:

- Where the high CPU usage is concentrated
- How to investigate high user-space CPU usage
- How to investigate high kernel-space CPU usage
- How to determine single-core saturation
- How to locate high CPU threads in multi-threaded programs
- How to determine high context switches
- How to determine high soft interrupts
- How to determine high hard interrupts
- How to analyze CPU usage caused by network packet processing
- `perf top`
- `perf record`
- `perf report`
- Basic understanding of flame graphs
- Java high CPU thread localization approach
- Production CPU troubleshooting considerations

This document is part of the advanced performance analysis series in the host troubleshooting series.

The previous 02nd article has already organized basic commands for CPU, memory, and load, and this article continues to delve into CPU troubleshooting.

The goal is:

- To distinguish between user-space, kernel-space, soft interrupts, and hard interrupts CPU consumption
- To determine whether it's a single-core bottleneck or overall CPU pressure
- To locate high CPU processes and threads
- To determine whether context switches are abnormal
- To determine whether soft interrupts are abnormal
- To use perf for function-level hot spot analysis
- To possess advanced SRE interview CPU deep troubleshooting thinking

---

## Two: High CPU Does Not Equal the Same Problem

Seeing high CPU usage does not mean you can draw conclusions directly.

High CPU usage may come from:

```text
Business process calculation pressure high

Request surged.

Code Dead Cycle

Log Brush

Compression / Encryption / Encoding Task

GC Frequent

There's a lot of threads, which leads to a lot of movement costs.

System calls are too many.

High kernel consumption

Network soft is high

Disk IO Related kernel expenses

Containers or virtualized fighting CPU

One-way program full of single cores

Multiple processes simultaneously. CPU
```

Therefore, CPU troubleshooting should first break down:

```text
CPU Overall height

→ Is it all core high or single core? High

→ Is it a high user state or a kernel? High

→ Is the normal process high or soft? High

→ Is it a high process or a whole system? High

→ Is it a high line in the process or is it all high?

→ Is it a short puncture or is it going on?
```

---

## Three: Advanced CPU Troubleshooting Path

Advanced CPU troubleshooting recommended path:

```text
top
→ Look at the whole thing. CPU and load

mpstat -P ALL
→ Look at each. CPU Core

vmstat
→ See running queues, context switching, system call trends

pidstat
→ Look at the process level. CPU

pidstat -t
→ Look at the linear scale. CPU

pidstat -w
→ Toggle in context

mpstat -I
→ Look for interruptions and soft breaks.

/proc/softirqs
→ Watching soft break distribution

/proc/interrupts
→ Look at the hard break distribution.

perf top / perf record
→ Look at the function heat.
```

CommonDiversion:

```text
us High
→ Application consumption CPU

sy High
→ Internal nuclear consumption CPU

si High
→ Soft break consumption CPU

hi High
→ Hard break consumption CPU

st High
→ Virtual Environment CPU It was stolen by the host.

Monochrome 100%
→ One-way bottlenecks or interruption of concentration

cs High
→ Context Switch Frequent

r High
→ CPU Run Queue
```

---

## Four: CPU Field Review

---

## Scenario 1: CPU Fields in top

View:

```bash
top
```

Key fields:

```text
us
→ User State CPU Usage

sy
→ kernel CPU Usage

ni
→ nice Priority process CPU

id
→ CPU Free

wa
→ IO wait

hi
→ Hard Break

si
→ Soft Break

st
→ The virtual environment was stolen by the host. CPU
```

Common judgments:

```text
us High
→ Apply self-counting

sy High
→ High internal nuclear consumption, possible system calls, network, disks, internal nuclear pathways

wa High
→ CPU Wait IONot necessarily. CPU The math problem.

si High
→ Soft break high, common for network package processing

hi High
→ Hard break high, possibly connected to cybercards, interruptions, hardware equipment

st High
→ There's a clear competition for cloud or virtual host resources.
```

---

## Scenario 2: CPU Fields in vmstat

View:

```bash
vmstat 1 5
```

Key fields:

```text
r
→ Number of processes waiting to run

b
→ No sleep breaks

in
→ Number of breaks per second

cs
→ Number of context transitions per second

us
→ User State CPU

sy
→ kernel CPU

id
→ CPU Free

wa
→ IO wait

st
→ It was stolen by the host. CPU
```

Common judgments:

```text
r Longer CPU Numerical
→ CPU Run queue pressure high

cs High
→ Context Switch Frequent

in High
→ Frequent interruptions

sy High
→ The kernel consumption is clear.

wa High
→ Maybe. IO Wait

st High
→ Virtual host CPU Competition
```

---

## Five: Single-Core Saturation and Multi-Core Saturation

---

## Scenario 3: View Each CPU Core

Use `top`:

```bash
top
```

Enter and press:

```text
1
```

You can also use:

```bash
mpstat -P ALL 1 5
```

---

## Scenario 4: What Does Single-Core Saturation Indicate

If you see:

```text
CPU0 100%
CPU1 10%
CPU2 8%
CPU3 12%
```

It may indicate a single-core bottleneck.

Common causes:

```text
One-way program

A linear loop.

The process is tied. CPU Kindness

The interruption is concentrated in one place. CPU

There's an imbalance in the menu.

Application does not take full advantage of polynuclear

Lock competition leads to only one line of effective work.
```

Next steps:

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

## Scenario 5: What Does All-Core High Indicate

If all CPU cores are very high:

```text
CPU0 90%
CPU1 95%
CPU2 88%
CPU3 92%
```

Common causes:

```text
Request increase

Multi-process or multi-linear calculated pressure

Batch Tasks

Compression / Encryption / Encoded

Log processing

High even application

Containers or multiple businesses CPU
```

Next steps:

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -n 20
```

```bash
pidstat 1 5
```

```bash
perf top
```

---

## Six: High CPU Process Localization

---

## Scenario 6: Sort Processes by CPU

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head
```

View more:

```bash
ps -eo pid,ppid,user,cmd,%mem,%cpu --sort=-%cpu | head -n 20
```

Purpose:

```text
Fast track which process is the most important CPU
```

---

## Scenario 7: Continuous Observation of Process CPU

```bash
pidstat 1 5
```

View a specific process:

```bash
pidstat -p PID 1 5
```

Example:

```bash
pidstat -p 12345 1 5
```

Purpose:

```text
ps It's instantaneous.
pidstat It's a continuous sample.
```

In production, it's more recommended to use continuous sampling to determine trends.

---

## Scenario 8: View Complete Process Startup Command

```bash
ps -fp PID
```

```bash
tr '\0' ' ' < /proc/PID/cmdline
```

View working directory:

```bash
ls -l /proc/PID/cwd
```

View executable file:

```bash
ls -l /proc/PID/exe
```

Purpose:

```text
确认进程归属

确认启动参数

确认业务目录

确认是否能重启

确认是否属于容器或宿主机服务
```

---

## Seven: Thread-Level CPU Troubleshooting

---

## Scenario 9: Why Look at Threads

Many services are multi-threaded programs, for example:

```text
Java 服务

Go 服务

Nginx worker

数据库

消息队列

高并发 API 服务
```

When a process has high CPU usage, not all threads may be high.

It could be:

```text
某一个线程死循环

少数线程执行重任务

GC 线程消耗 CPU

日志线程异常

网络线程异常

业务线程池打满
```

Therefore, thread-level troubleshooting is needed.

---

## Scenario 10: View Threads with top

```bash
top -H -p PID
```

Example:

```bash
top -H -p 12345
```

Explanation:

```text
-H
→ 显示线程

-p PID
→ 只看指定进程
```

After entering, you can press:

```text
P
```

To sort by CPU.

---

## Scenario 11: View Thread-Level CPU with pidstat

```bash
pidstat -t -p PID 1 5
```

Example:

```bash
pidstat -t -p 12345 1 5
```

Focus on:

```text
TID
→ 线程 ID

%usr
→ 用户态 CPU

%system
→ 内核态 CPU

%CPU
→ 总 CPU
```

---

## Scenario 12: View Threads with ps

```bash
ps -T -p PID
```

Example:

```bash
ps -T -p 12345
```

Custom fields:

```bash
ps -T -p PID -o pid,tid,psr,pcpu,pmem,stat,comm
```

Field explanations:

```text
pid
→ 进程 ID

tid
→ 线程 ID

psr
→ 当前运行在哪个 CPU 核心

pcpu
→ CPU 使用率

stat
→ 线程状态

comm
→ 线程名
```

---

## Eight: Java High CPU Thread Localization

---

## Scenario 13: Java High CPU Troubleshooting Chain

Common Java high CPU troubleshooting chain:

```text
找到 Java 进程 PID

→ 找到高 CPU 线程 TID

→ TID 转十六进制

→ jstack 中搜索 nid

→ 定位线程栈

→ 结合代码判断原因
```

---

## Scenario 14: Find Java Process

```bash
ps -ef | grep java | grep -v grep
```

Or:

```bash
pgrep -a java
```

Assume PID is:

```text
12345
```

---

## Scenario 15: Find High CPU Threads

```bash
top -H -p 12345
```

Or:

```bash
pidstat -t -p 12345 1 5
```

Assume thread ID is:

```text
12388
```

---

## Scenario 16: Convert Thread ID to Hexadecimal

```bash
printf "%x\n" 12388
```

Assume output:

```text
3064
```

---

## Scenario 17: Use jstack to View Thread Stack

```bash
jstack 12345 > /tmp/jstack-12345.txt
```

Search for nid:

```bash
grep -n "3064" /tmp/jstack-12345.txt
```

Or:

```bash
grep -n "nid=0x3064" /tmp/jstack-12345.txt
```

View context:

```bash
grep -n -A 30 -B 5 "nid=0x3064" /tmp/jstack-12345.txt
```

---

## Scenario 18: Common Causes of Java High CPU

Common causes:

```text
Dead circulation.

Regular expression performance issues

Frequent GC

Mass JSON Sequenced / Inverse sequence

Encryption

Compression decompression

Log Brush

Thread pool abnormal.

Lock competition

Large object processing

There's too much data coming together.
```

Continue combining:

```text
Apply Log

GC Log

Request Volume

Code changes

Recent

Thread Inn
```

To determine the root cause.

---

## Nine: Context Switch Troubleshooting

---

## Scenario 19: What is Context Switching

Context switching can be simply understood as:

```text
CPU Switch from one task to another
```

Common types:

```text
Process Switch

Thread Switch

User State / kernel exchange

Interrupt to switch
```

Context switching is not necessarily a problem.

But when it's too high, it brings overhead:

```text
CPU Time spent on scheduling.

Reduction in real business computing time

System response slow down.

load Raise

sy CPU Raise
```

---

## Scenario 20: View Context Switches with vmstat

```bash
vmstat 1 5
```

Key fields:

```text
cs
→ Number of context transitions per second

in
→ Number of breaks per second
```

Judgment:

```text
cs It's still high.
→ Could be context-switched.

in It's still high.
→ Could be a lot of interruptions.
```

Note:

```text
cs Is it unusual to combine? CPU Number, type of operation and historical baseline judgement
```

---

## Scenario 21: View Process Context Switches with pidstat

```bash
pidstat -w 1 5
```

View a specific process:

```bash
pidstat -w -p PID 1 5
```

Example:

```bash
pidstat -w -p 12345 1 5
```

Field explanations:

```text
cswch/s
→ Voluntary context switch per second

nvcswch/s
→ Irradiated context switch per second
```

---

## Scenario 22: Voluntary Context Switches and Involuntary Context Switches

```text
Voluntary Context Switch cswch/s
→ It's a process. CPU
→ Often waiting. IOWaiting for locks, waiting for networks,sleep

Involuntary context switch nvcswch/s
→ Processes are forced out of the system.
→ Commonly CPU We're out of time.CPU Competition is intense.
```

Common Determinations:

```text
cswch/s High
→ Could be a lot of waiting, locking,IONetwork blocking

nvcswch/s High
→ Maybe. CPU Strong competition, too many threads or... CPU Insufficient
```

---

## Scenario 23: Common Causes of High Context Switches

Common Causes:

```text
Too many threads.

The thread pool is too large

Frequent creation and destruction of threads

It's very competitive.

A lot of short connections.

A large number of system calls

High even network request

Frequent IO Wait

Containers CPU limit Too low.

Apply sleep / wakeup Frequent
```

Troubleshooting Commands:

```bash
vmstat 1 5
```

```bash
pidstat -w 1 5
```

```bash
pidstat -t -w -p PID 1 5
```

Check Thread Count:

```bash
cat /proc/PID/status | grep Threads
```

Or:

```bash
ps -T -p PID | wc -l
```

---

## Ten. Soft Interrupt Troubleshooting

---

## Scenario 24: What is a Soft Interrupt

A soft interrupt is a mechanism in the Linux kernel for handling asynchronous events.

Common soft interrupts include:

```text
NET_RX
→ Network Pack

NET_TX
→ Network Package

TIMER
→ Timer

SCHED
→ Movement control

BLOCK
→ Block Devices

RCU
→ Core RCU Mechanisms
```

When network traffic is high, you often see:

```text
si CPU High
NET_RX Grow fast.
ksoftirqd Process CPU High
```

---

## Scenario 25: Checking si in top

```bash
top
```

Focus on:

```text
si
```

If `si` rises significantly, it indicates that soft interrupts are consuming a lot of CPU.

---

## Scenario 26: Checking Soft Interrupts with mpstat

```bash
mpstat -P ALL 1 5
```

Focus on each CPU's:

```text
%soft
```

If a particular CPU's `%soft` is very high, it may indicate that soft interrupts are concentrated on that core.

---

## Scenario 27: Checking Soft Interrupt Statistics

```bash
cat /proc/softirqs
```

Focus on:

```text
NET_RX
NET_TX
TIMER
SCHED
BLOCK
```

If `NET_RX` grows rapidly, it is typically related to network packet reception.

You can continuously observe:

```bash
watch -n 1 cat /proc/softirqs
```

---

## Scenario 28: Checking ksoftirqd

```bash
ps -ef | grep ksoftirqd
```

Check CPU:

```bash
top
```

If you see something like:

```text
ksoftirqd/0
ksoftirqd/1
```

High CPU usage may indicate significant soft interrupt pressure.

---

## Scenario 29: Common Causes of High Soft Interrupts in Networking

Common Causes:

```text
Network traffic surged.

There's a lot of buns.

Too many connections

DDoS Or unusual traffic.

There's an imbalance in the menu.

Interruption concentrated on a single CPU

High cost of retransmitting container network

iptables / conntrack Pressure.

Nginx / Overflow of gateway-type services
```

Continue Troubleshooting:

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

```bash
tcpdump -i any -nn
```

---

## Eleven. Hard Interrupt Troubleshooting

---

## Scenario 30: What is a Hard Interrupt

Hard interrupts typically come from hardware devices, for example:

```text
Cybercard

Disk controller

NVMe Equipment

SCSI Equipment

USB Equipment

Virtualization equipment
```

If `hi` is high, it indicates that hard interrupts are consuming significant CPU resources.

---

## Scenario 31: Checking Hard Interrupts

```bash
cat /proc/interrupts
```

Continuously observe:

```bash
watch -n 1 cat /proc/interrupts
```

Focus on:

```text
Which break is growing fast?

Whether or not to focus on a certain CPU

Whether it comes from cybercards, disks,virtioI don't know.nvme
```

---

## Scenario 32: Checking hi in top

```bash
top
```

Focus on:

```text
hi
```

If `hi` remains consistently high, you need to combine it with `/proc/interrupts` to determine the source device.

---

## Scenario 33: Interrupts Concentrated on a Single Core

If `/proc/interrupts` shows that interrupts for a particular device are concentrated on a single CPU, it may cause that core to become fully utilized.

Common Directions:

```text
CyberCardo Queue Configuration

irqbalance Whether to run

Interrupting kinship and sexual configuration

Virtual Net Card Queue

RSS / RPS / XPS Configure
```

Check irqbalance:

```bash
systemctl status irqbalance
```

---

## Twelve. perf Basics

---

## Scenario 34: What is perf

`perf` is a commonly used performance analysis tool in Linux.

It can be used to analyze:

```text
CPU Hotspot Functions

Internal nuclear hotspots

User-state hotspots

System call costs

Lock Competition Threads

Function Call Path

Performance bottlenecks
```

Common Usage:

```text
perf top
→ View hotspot functions in real time

perf record
→ Sample log performance data

perf report
→ Analysis of sampling results
```

---

## Scenario 35: Installing perf

Ubuntu / Debian:

```bash
apt install -y linux-tools-common linux-tools-$(uname -r)
```

If the above doesn't match, you can search for the corresponding kernel tool package.

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y perf
```

Or:

```bash
dnf install -y perf
```

Verification:

```bash
perf --version
```

---

## Scenario 36: perf Permission Issues

Some systems restrict regular users from using perf.

Check Restrictions:

```bash
cat /proc/sys/kernel/perf_event_paranoid
```

Temporary Adjustment:

```bash
sysctl -w kernel.perf_event_paranoid=1
```

More Lenient:

```bash
sysctl -w kernel.perf_event_paranoid=0
```

Production Note:

```text
Don't keep security restrictions down for too long.
The original configuration should be restored after a temporary check.
```

---

## Thirteen. perf top Real-time Hotspot Analysis

---

## Scenario 37: Checking System-level Hot Functions

```bash
perf top
```

Purpose:

```text
View the current in real time CPU Hotspot Functions
```

Common Output Fields:

```text
Overhead
→ Percentage

Shared Object
→ From whom, like the kernel,libcBinary application

Symbol
→ Hotspot Functions
```

---

## Scenario 38: Checking Hotspots for a Specific Process

```bash
perf top -p PID
```

Example:

```bash
perf top -p 12345
```

Suitable For:

```text
Some process CPU High

I need to know. CPU on which functions

Normal ps/top Only see processes, no functions
```

---

## Scenario 39: Common Judgments for perf top

If the hotspot is in application functions:

```text
It could be a business code calibration hotspot.
```

If the hotspot is in libc:

```text
Could be string processing, memory copying, sorting, regularization, serialization, etc.
```

If the hotspot is in kernel functions:

```text
Could be system calls, network, disks, locks, dispatch related.
```

If the hotspot is in encryption functions:

```text
Maybe. TLSdecryption of encryption CPU
```

If the hotspot is in compression functions:

```text
Maybe. gzipCompressed task consumption CPU
```

---

## Fourteen. perf record and perf report

---

## Scenario 40: Sampling a Specific Process

```bash
perf record -p PID -g -- sleep 30
```

Explanation:

```text
-p PID
→ Specify Process

-g
→ Record Call Inn

sleep 30
→ Sample 30 sec
```

After sampling, a file will be generated:

```text
perf.data
```

---

## Scenario 41: Viewing Sampling Reports

```bash
perf report
```

Purpose:

```text
View perf record Collected Hotspot Functions and Call Inn
```

---

## Scenario 42: System-level Sampling

```bash
perf record -a -g -- sleep 30
```

Explanation:

```text
-a
→ System-wide sampling
```

Suitable For:

```text
I don't know which process caused CPU High

We need a global view of hot spots.
```

Production Note:

```text
System-wide sampling costs are higher
Don't take too long to sample.
```

---

## Scenario 43: Specifying Sampling Frequency

```bash
perf record -F 99 -p PID -g -- sleep 30
```

Explanation:

```text
-F 99
→ Per second 99 Subsampling
```

Production Recommendation:

```text
Don't make too high a sample.
Avoiding significant additional costs for production machines
```

---

## Fifteen. Flame Graph Basics

---

## Scenario 44: What is a Flame Graph

A flame graph is a visualization of CPU sampling call stacks.

It can be understood as:

```text
Widener
→ Organisation CPU The more you consume.

Deeper
→ Means the deeper the Call Inn
```

Flame graphs are suitable for answering:

```text
CPU What are the main functions on which time is spent?

Business code, library function, or kernel?

Which call chain is the most expensive? CPUWhat?

Where should code optimization begin?
```

---

## Scenario 45: Use Cases for Flame Graphs

Suitable For:

```text
CPU Sustained height

Single process CPU High

Positioning performance bottlenecks during pressure measurements

Optimizing interface performance

Positioning Hotspot Functions

Location anomaly cycle
```

Not Suitable For:

```text
It's a temporary phenomenon that can't be repeated.

No sample window

Production machines are not allowed to install tools

The problem is obviously the disk's full, the port doesn't make sense.DNS Error
```

---

## Scenario 46: Basic Flame Graph Workflow

Common Workflow:

```text
perf record Sample

→ perf script Export Call Inn

→ stackcollapse-perf.pl Collapse

→ flamegraph.pl Generate SVG

→ Browser to view flame maps
```

Command Example:

```bash
perf record -F 99 -p PID -g -- sleep 30
```

```bash
perf script > out.perf
```

Subsequent use of the FlameGraph tool to generate the graph:

```bash
stackcollapse-perf.pl out.perf > out.folded
```

```bash
flamegraph.pl out.folded > cpu-flamegraph.svg
```

Explanation:

```text
stackcollapse-perf.pl and flamegraph.pl From FlameGraph Toolset
The production environment is not necessarily pre-assembled.
It can be handled in the test environment or by a special analyser.
```

---

## Sixteen. Typical CPU High Scenarios

---

## Scenario 47: High CPU in User-space

Phenomenon:

```text
top Medium us High
A business process CPU High
```

Troubleshooting Commands:

```bash
top
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head
```

```bash
pidstat -p PID 1 5
```

```bash
pidstat -t -p PID 1 5
```

```bash
perf top -p PID
```

Possible Causes:

```text
High volume of business requests

Code Loop

High volume of data processing

Right, right.JSONCompression, encryption

GC

Algorithmic complexity issues

Log processing
```

---

## Scenario 48: High CPU in Kernel-space

Phenomenon:

```text
top Medium sy High
vmstat Medium sy High
```

Troubleshooting Commands:

```bash
vmstat 1 5
```

```bash
pidstat 1 5
```

```bash
strace -c -p PID
```

```bash
perf top
```

Possible Causes:

```text
System calls are too many.

Network package processing

Disk IO Path

A lot of small file operations

Frequent creation of destruction process

Container network forwarding

iptables / conntrack Pressure

kernel lock competition
```

Explanation:

```text
strace Additional impact on production processes
Do not mount at high load core process long Let's go.
```

---

## Scenario 49: High CPU due to Soft Interrupts

Phenomenon: /think

```text
top Medium si High
mpstat Medium %soft High
ksoftirqd CPU High
```

Troubleshooting Commands:

```bash
mpstat -P ALL 1 5
```

```bash
cat /proc/softirqs
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

Possible Causes:

```text
Network traffic surged.

There's too many buns.

Too many connections

Unusual flow

There's an uneven break in the card.

iptables / conntrack Pressure

Container network forwarding costs
```

---

## Scenario 50: High Context Switching

Phenomenon:

```text
vmstat Medium cs High
pidstat -w Medium cswch/s or nvcswch/s High
sy CPU High
```

Troubleshooting Commands:

```bash
vmstat 1 5
```

```bash
pidstat -w 1 5
```

```bash
pidstat -t -w -p PID 1 5
```

```bash
cat /proc/PID/status | grep Threads
```

Possible Causes:

```text
Too many lines.

It's too big.

Lock competition

A lot of short connections.

Frequent IO Wait

Frequent sleep/wakeup

CPU limit Too low.

Process control competition
```

---

## Scenario 51: High VM st

Phenomenon:

```text
top Medium st High
```

Meaning:

```text
Virtual machine to use. CPU
But... CPU Time occupied by host or other virtual machine
```

Common Causes:

```text
The cloud host is oversold.

Room host neighbor noise

Low specification

Resource constraints on virtual platforms
```

Resolution Direction:

```text
Observation of sustainability

Replacement of host

Upgrade instance specification

Contact the cloud factory.

Error Handle Task
```

---

## Seventeen. CPU Troubleshooting in Container Scenarios

---

## Scenario 52: Determine Whether to Check Host or Container First for High Container CPU

Recommended Order:

```text
Host top
→ Confirm overall CPU

docker stats
→ Look at the container. CPU

docker top
→ Look inside the container.

Enter the container.
→ Use top / ps Look at the internal process.

Host pidstat/perf
→ If necessary, analyze the truth. PID
```

---

## Scenario 53: Check Container Resources

```bash
docker stats
```

Check Container Processes:

```bash
docker top ContainersID
```

Enter Container:

```bash
docker exec -it ContainersID /bin/sh
```

---

## Scenario 54: Container Processes and Host PID

Get Container Main Process Host PID:

```bash
docker inspect -f '{{.State.Pid}}' ContainersID
```

Check Processes:

```bash
ps -fp $(docker inspect -f '{{.State.Pid}}' ContainersID)
```

Note:

```text
The process inside the container is essentially a host process
You can see it from the host when you're doing the advanced check. PID
```

---

## Scenario 55: Container CPU Limit Causes Queuing

If the container CPU limit is set too low, it may result in:

```text
Slow application in containers

Host CPU There's room.

Line inside the container

Apply delay elevation
```

Check Container Limits:

```bash
docker inspect ContainersID | grep -i cpu -A 20
```

In Kubernetes scenarios, check Pod resources:

```bash
kubectl get pod PodName -n Namespace -o yaml
```

Focus on:

```text
resources.requests.cpu

resources.limits.cpu
```

---

## Eighteen. Production CPU Troubleshooting Notes

---

## 1. Do Not Kill Processes Immediately When Seeing High CPU

First confirm:

```text
Which service does the process belong to?

Core business

Is there a lead?

Whether the assignment is ongoing

Whether to restart

Is it necessary to keep the scene?
```

First collect:

```bash
top -b -n 1 | head -n 50
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -n 20
```

```bash
pidstat 1 5
```

---

## 2. perf Has Overhead, Use It Carefully in Production

Recommend:

```text
Don't take too long to sample.

Prioritize Assign PID

Avoid HF sampling

Be careful at peak.

Do not fill disks with sample files
```

Recommended:

```bash
perf record -F 99 -p PID -g -- sleep 30
```

---

## 3. Do Not Use strace on Production Processes for Long Periods

`strace` will affect process performance.

If must use:

```bash
strace -c -p PID
```

Sample briefly.

Avoid long-term:

```bash
strace -p PID
```

Attaching to core production processes.

---

## 4. Single Data Point May Not Be Reliable

CPU issues require continuous sampling:

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

## 5. High CPU Should Be Combined with Business Timeline

Confirm:

```text
Is it just published?

Is the flow rising?

It's the negative when the mission starts.

Whether to back up compression task run

Whether or not to draw a log

Whether slow database queries make applications busy, etc.

Whether to use an abnormal retry outside
```

---

## Nineteen. Summary of Common Commands in This Document

---

## Overall CPU

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

## Multi-core CPU

```bash
mpstat -P ALL 1 5
```

```bash
top
```

After entering top, press:

```text
1
```

---

## Process CPU

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

## Thread CPU

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

## High CPU in Java

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
printf "%x\n" ThreadID
```

```bash
jstack PID > /tmp/jstack-PID.txt
```

```bash
grep -n "nid=0xHexadecimal ThreadID" /tmp/jstack-PID.txt
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
watch -n 1 cat /proc/softirqs
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

## perf

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

## strace

```bash
strace -c -p PID
```

---

## Container CPU

```bash
docker stats
```

```bash
docker top ContainersID
```

```bash
docker inspect -f '{{.State.Pid}}' ContainersID
```

```bash
ps -fp $(docker inspect -f '{{.State.Pid}}' ContainersID)
```

```bash
docker inspect ContainersID | grep -i cpu -A 20
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

Install perf on Ubuntu/Debian:

```bash
apt install -y linux-tools-common linux-tools-$(uname -r)
```

Install perf on RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y perf
```

Or:

```bash
dnf install -y perf
```

---

## Twenty. One-Sentence Summary

The core of advanced CPU troubleshooting is not just looking at CPU usage rate, but breaking down the sources of CPU consumption:

```text
us
→ User-state application consumption

sy
→ Internal nuclear consumption

si
→ Soft break consumption

hi
→ Hard break consumption

st
→ Virtual host competition

wa
→ IO Waiting, not mere. CPU The math problem.
```

CPU Troubleshooting Path:

```text
top Look at the whole thing.

→ mpstat Look at every core.

→ ps / pidstat Search process

→ pidstat -t / top -H Find the thread.

→ vmstat / pidstat -w Toggle in context

→ /proc/softirqs Watch the soft break.

→ /proc/interrupts Look at the break.

→ perf Look at the function heat.
```

Java High CPU Path:

```text
Found it. Java PID

→ top -H Find Gao. CPU Thread

→ Thread ID Turn Hexadecimal

→ jstack Search. nid

→ Positioning Code.
```

Soft Interrupt Troubleshooting Path:

```text
top Look. si

→ mpstat Look. %soft

→ /proc/softirqs Look. NET_RX / NET_TX

→ sar / iftop / ss / tcpdump Look at the traffic and the connections.
```

Context Switching Troubleshooting Path:

```text
vmstat Look. cs

→ pidstat -w See process transition

→ pidstat -t -w Watch the thread switch

→ I'm trying to figure out the lengths, lock the competition,IO Wait.CPU Competition
```

Production Recommendations:

```text
Don't look. CPU High is straight. kill Process
Not just once. top I'm gonna come to a conclusion.
Don't. wa High and Wrong CPU I'm not good enough.
Not for long. strace Online Core Process
perf Sampling takes time and scope.
CPU High must be judged in conjunction with business flows, timing, logs and dependency
```