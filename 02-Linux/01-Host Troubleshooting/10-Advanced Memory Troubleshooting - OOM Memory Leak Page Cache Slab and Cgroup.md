# 10-Advanced Memory Troubleshooting: OOM, Memory Leaks, page cache, slab, and cgroup

#Linux #Transport #SRE #MemoryCheck #OOM #MemoryLeak #pagecache #slab #cgroup #ContainerMemory #PerformanceAnalysis

---

## Recommended Path

01-Linux Foundation and Host Maintenance/01-Host Troubleshooting/10-Advanced Memory Troubleshooting: OOM, Memory Leaks, page cache, slab, and cgroup.md

---

## I. Document Explanation

This document organizes advanced memory troubleshooting methods for Linux hosts, focusing on:

- Linux memory usage model
- How to correctly interpret `free` results
- Relationships between `available`, `buff/cache`, and `swap`
- How to view process memory usage
- Differences between VIRT / RES / SHR
- Basic understanding of RSS / PSS / USS
- OOM Killer troubleshooting
- Differences between host OOM and container OOM
- Initial judgment of memory leaks
- page cache troubleshooting
- slab memory troubleshooting
- cgroup memory limit troubleshooting
- Docker / Kubernetes memory limit troubleshooting
- drop_caches risks
- Production memory troubleshooting precautions

This document is part of the advanced performance analysis series in the host troubleshooting series.

The previous 02nd article has already organized basic memory commands, and this article continues to delve into memory mechanisms and production troubleshooting paths.

The goal is:

- To determine whether Linux memory is truly insufficient
- → To explain why `free` being low doesn't necessarily indicate an anomaly
- → To locate high memory consuming processes
- → To troubleshoot the causes of OOM occurrences
- → To distinguish between host OOM and container OOM
- → To initially judge memory leaks
- → To understand page cache and slab usage
- → To troubleshoot anomalies caused by cgroup memory limits

---

## II. Common Symptoms of Memory Issues

In production environments, memory issues often manifest as:

```text
The system's obvious.

SSH Login Slow

Slower application response

The service suddenly quits.

Packagings by kill

Pod Status OOMKilled

Java Process killed.

System log appearance Out of memory

swap Continued increase in usage

load Raise but CPU Not high.

Disk IO ♪ Suddenly rising ♪

Slower database connection

Nginx / Application abnormally restarted

Overuse of surveillance memory
```

Common root causes include:

```text
Real memory deficit

Apply memory leaks

Cache Unlimited Growth

Too many processes

Too many threads.

page cache Overoccupied but recoverable

slab Unnormal growth

swap Frequent page changes

cgroup memory limit Too small.

Unjustified container memory limit

JVM Unreasonable memory parameters

Big Query / Large object / Big document processing

An anomaly exists in the system
```

---

## III. Overall Memory Troubleshooting Path

Recommended advanced memory troubleshooting path:

```text
free -h
→ Look. availableI don't know.buff/cacheI don't know.swap

vmstat 1 5
→ Look. si / so Whether to change pages frequently

top / ps
→ High Memory Process

/proc/meminfo
→ Look at the memory breakdown.

dmesg / journalctl
→ Cha. OOM Records

pmap / smem
→ See process memory distribution

slabtop / /proc/slabinfo
→ Cha. slab Occupation

cgroup
→ Check container or service memory limits

docker stats / kubectl describe
→ Check the containers. OOM
```

CommonDiversion:

```text
available Low + swap si/so High
→ Real memory pressure is high.

available Not low. + buff/cache High
→ Most of the time, it's normal.

A process RSS Sustained growth
→ Possible memory leakage or cache growth

dmesg Yes. OOM
→ Need to identify the killing process and the trigger

Pod OOMKilled
→ Check the containers first. limitIt's not necessarily the host's memory.

slab Sustained growth
→ Possible kernel cache,dentry/inode Or kernel object abnormal.

page cache High
→ It's probably a lot of paperwork to read and write. It's not necessarily a malfunction.
```

---

## IV. Correct Understanding of free

---

## Scenario 1: View Memory Usage

```bash
free -h
```

Common output fields:

```text
total
→ Total RAM

used
→ Used memory

free
→ Full Free Memory

shared
→ Share Memory

buff/cache
→ buffer and cache

available
→ Estimate memory available for new processes
```

Focus on:

```text
available

swap

buff/cache
```

---

## Scenario 2: Don't Only Look at free

Many people see:

```text
free Low
```

and think memory is insufficient.

This is inaccurate.

Linux will try to use idle memory as cache, for example:

```text
File Cache

Directory Item Cache

inode Cache

Block Device Cache
```

So:

```text
free Low
→ Not necessarily.

buff/cache High
→ Not necessarily.

available Low
→ More alarming.
```

---

## Scenario 3: available is More Important

`available` represents the estimated memory available for new processes.

If:

```text
available Sufficient
swap No significant growth.
The system isn't Carton.
Nothing. OOM
```

you usually cannot simply judge memory insufficiency.

If:

```text
available Low
swap Sustained growth
vmstat si / so High
System Carton.
Come on. OOM
```

it's more likely to indicate real memory pressure.

---

## Scenario 4: Check Swap

```bash
free -h
```

Check swap devices:

```bash
swapon --show
```

Check more detailed memory:

```bash
cat /proc/meminfo
```

Key fields:

```text
SwapTotal

SwapFree

SwapCached
```

---

## V. vmstat: Determine Frequent Page Swapping

---

## Scenario 5: Check Swap Page Swapping

```bash
vmstat 1 5
```

Key fields:

```text
si
→ swap inFrom swap Transfer Memory

so
→ swap out, switch from memory to swap
```

Judgment:

```text
si / so Long term 0
→ No apparent page change pressure.

si / so Sometimes it's worth it.
→ Not necessarily.

si / so Constantly high
→ The system is changing pages more frequently, and memory pressure is clear.
```

---

## Scenario 6: Impact of High Swap

Frequent swap page swapping leads to:

```text
The system is clearly slowing down.

Disk IO Increase

load Raise

Slower application response

Database performance is down

Container scheduling or process running anomalies
```

Because swap essentially moves some memory pages to disk.

Disk speed is much slower than memory.

---

## Scenario 7: Swap Usage Doesn't Necessarily Indicate Anomalies

If it's just:

```text
swap Small use
si / so Basic 0
available Still enough.
There's no carton on the system.
```

it's not necessarily a serious issue.

What truly needs attention is:

```text
swap Sustained growth

si / so Maintained high

Impact on operational response

Whether or not to accompany OOM or load Raise
```

---

## VI. /proc/meminfo: Memory Breakdown

---

## Scenario 8: View Detailed Memory Information

```bash
cat /proc/meminfo
```

Common key fields:

```text
MemTotal
→ Total RAM

MemFree
→ Full Free Memory

MemAvailable
→ Available Memory Estimates

Buffers
→ Block device buffer

Cached
→ File Page Cache

SwapCached
→ Cache that was replaced and replaced

Active
→ Active Memory

Inactive
→ Inactive memory

Unevictable
→ Non-recoverable memory

Dirty
→ Dirty Pages

Writeback
→ Replying pages

Slab
→ Core slab Occupation

SReclaimable
→ Recoverable slab

SUnreclaim
→ Unrecoverable slab

PageTables
→ Table occupancy

Committed_AS
→ Pledged to allocate memory

CommitLimit
→ Maximum commitment memory
```

---

## Scenario 9: Quick View of Key Fields

```bash
egrep "MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree|Dirty|Writeback|Slab|SReclaimable|SUnreclaim|PageTables|Committed_AS|CommitLimit" /proc/meminfo
```

Suitable for quick judgment:

```text
Is the memory really insufficient?

cache Is it high?

slab Is it unusual?

swap Is there a small amount left?

dirty Whether there is a backlog of pages

Is the page too large?
```

---

## Scenario 10: Dirty and Writeback

Field meanings:

```text
Dirty
→ Modified but not written back to disk memory Page

Writeback
→ Writing memory pages back to disk
```

If `Dirty` is consistently high, it may indicate:

```text
Extensive Writing

We can't keep up with the disk.

Storage Slow

Apply writing pressure high
```

Continue troubleshooting:

```bash
iostat -x 1 5
```

```bash
pidstat -d 1 5
```

```bash
iotop -o -P
```

---

## VII. Process Memory Basics: VIRT, RES, SHR

---

## Scenario 11: top to View Process Memory

```bash
top
```

After entering, press:

```text
M
```

Sort by memory.

Common fields:

```text
VIRT
→ Virtual Memory

RES
→ Permanent Physical Memory

SHR
→ Share Memory

%MEM
→ Use ratio of physical memory
```

---

## Scenario 12: VIRT Doesn't Equal Real Usage

`VIRT` being large doesn't necessarily mean real usage is high.

It includes:

```text
Virtual address space for process application

Map Files

Shared Library

Unused Memory Area

Memory Map Area
```

More commonly used to judge real physical usage is:

```text
RES
RSS
```

---

## Scenario 13: RES / RSS is Closer to Actual Physical Usage

`RES` represents the physical memory that the process resides in.

Common fields in `ps` are:

```text
RSS
```

Check:

```bash
ps -eo pid,ppid,user,rss,vsz,cmd --sort=-rss | head
```

RSS unit is typically KB.

A more readable way:

```bash
ps -eo pid,ppid,user,%mem,%cpu,rss,vsz,cmd --sort=-rss | head -n 20
```

---

## Scenario 14: SHR is Shared Memory

`SHR` represents the shared memory portion, for example:

```text
Shared Library

Shared memory session

Memory shared by multiple processes Page
```

Note:

```text
Multiple processes RSS Adding may double-count shared memory
```

Therefore, to more precisely analyze a process's exclusive memory, you need to understand PSS / USS.

---

## VIII. Basic Understanding of RSS, PSS, USS

---

## Scenario 15: What is RSS

RSS:

```text
Resident Set Size
```

Represents the size of the process's current physical memory residency.

Features:

```text
Easy to view.

top / ps Common

But shared memory can be double calculated
```

Check:

```bash
ps -o pid,rss,vsz,cmd -p PID
```

---

## Scenario 16: What is PSS

PSS:

```text
Proportional Set Size
```

Represents the usage after sharing memory is proportionally allocated.

For example:

```text
Part 100MB Shared library by 10 Process Sharing
Each process PSS Assessment only 10MB
```

PSS is more suitable for evaluating the actual usage of multiple processes collectively.

Check:

```bash
cat /proc/PID/smaps_rollup
```

If the system supports it, you'll see:

```text
Pss
```

---

## Scenario 17: What is USS

USS:

```text
Unique Set Size
```

Represents the exclusive memory of the process.

It can be understood as:

```text
If the process exits, a real release of internal stocks
```

Common tools:

```text
smem
```

---

## Scenario 18: Use smem to View PSS / USS

Install:

Ubuntu / Debian:

```bash
apt install -y smem
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y smem
```

Or:

```bash
dnf install -y smem
```

View process memory:

```bash
smem -r -k
```

Sort by PSS:

```bash
smem -r -k -s pss
```

Explanation:

```text
smem Default installation not necessarily in the production environment
You can use it if you don't have one. psI don't know.topI don't know.pmapI don't know./proc/PID/smaps_rollup
```

---

## 9. Locating High Memory Processes

---

## Scenario 19: Sort by Memory

```bash
ps -eo pid,ppid,user,%mem,%cpu,rss,vsz,cmd --sort=-rss | head -n 20
```

Or:

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -n 20
```

Purpose:

```text
Rapidly locate high memory processes
```

---

## Scenario 20: View Memory of a Specific Process

```bash
ps -fp PID
```

```bash
ps -o pid,ppid,user,%mem,%cpu,rss,vsz,cmd -p PID
```

Check Status:

```bash
cat /proc/PID/status
```

Key Fields:

```text
VmSize
→ Virtual Memory

VmRSS
→ Permanent Physical Memory

VmData
→ Data band

VmStk
→ Inn

VmExe
→ Executable

VmLib
→ Library

Threads
→ Threads
```

Quick View:

```bash
egrep "VmSize|VmRSS|VmData|VmStk|VmExe|VmLib|Threads" /proc/PID/status
```

---

## Scenario 21: View Process Memory Mapping

```bash
pmap PID
```

Check Details:

```bash
pmap -x PID | tail
```

Example:

```bash
pmap -x 12345 | tail
```

Purpose:

```text
View the memory mapping of the process segments

Approximation of anonymous memory, library, document mapping, etc.

Auxiliary positioning memory growth direction
```

---

## Scenario 22: View smaps_rollup

```bash
cat /proc/PID/smaps_rollup
```

Quick Filter:

```bash
egrep "Rss|Pss|Private_Clean|Private_Dirty|Shared_Clean|Shared_Dirty|Swap" /proc/PID/smaps_rollup
```

Suitable for Viewing:

```text
Process RSS

Process PSS

Private Memory

Share Memory

Process swap Use
```

---

## 10. OOM Killer Troubleshooting

---

## Scenario 23: What is OOM

OOM:

```text
Out Of Memory
```

When system or cgroup memory is insufficient, the kernel may kill a process to free memory.

Common Phenomena:

```text
The service suddenly quits.

Process without visible application log

System log appearance Out of memory

Container Status OOMKilled

Java Process is being killed

dmesg There you go. Killed process
```

---

## Scenario 24: View OOM Records on Host

```bash
dmesg -T | grep -i oom
```

```bash
dmesg -T | grep -i "killed process"
```

```bash
journalctl -k | grep -i oom
```

```bash
journalctl -k | grep -i "killed process"
```

---

## Scenario 25: Key Points in OOM Logs

OOM logs should focus on:

```text
Trigger OOM Process

By kill Process

Process PID

Process Name

total-vm

anon-rss

file-rss

shmem-rss

oom_score_adj

Memory cgroup Information
```

Common Fragments:

```text
Out of memory: Killed process 12345 (java)
```

Explanation:

```text
The kernel finally killed him. PID 12345 Yes. java Process
```

---

## Scenario 26: View oom_score of a Process

If the process is still running, check:

```bash
cat /proc/PID/oom_score
```

```bash
cat /proc/PID/oom_score_adj
```

Explanation:

```text
oom_score
→ OOM The possibility rating is selected

oom_score_adj
→ OOM Rating adjustment value
```

`oom_score_adj` higher, the more likely it is to be selected by OOM Killer.

---

## Scenario 27: OOM is Not Necessarily Triggered by the Process with the Highest Memory

Note:

```text
Trigger OOM Process
It's not necessarily the process of the ultimate murder.

Process of being killed
It's usually kernel-based. oom_score Selected sacrifice process
```

Therefore, troubleshooting should combine:

```text
OOM Pre-system storage status

Process of killing

Other high memory processes at the time

cgroup Limits

Containers limit

Business timeline
```

---

## 11. Host OOM vs Container OOM

---

## Scenario 28: Host OOM

Host OOM indicates:

```text
Memory pressure triggered at the whole host level OOM
```

Common Phenomena:

```text
dmesg / journalctl Yes. OOM Records

Some host process is being kill

Multiple services may be affected

System-wide memory available Low

swap Possible frequent use
```

Troubleshooting:

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
ps -eo pid,ppid,user,%mem,%cpu,rss,cmd --sort=-rss | head -n 20
```

---

## Scenario 29: Container OOM

Container OOM indicates:

```text
Packages Over cgroup memory limit
```

It does not necessarily mean the host memory is insufficient.

For example:

```text
There's plenty of memory on the host.
But the container. limit Only 512Mi
Apply More Use 512Mi
Packagings by OOMKilled
```

Docker View:

```bash
docker ps -a
```

```bash
docker inspect ContainersID | grep -i oom -A 10
```

```bash
docker stats
```

---

## Scenario 30: Kubernetes Pod OOMKilled

View Pod:

```bash
kubectl get pod -n Namespace
```

View Details:

```bash
kubectl describe pod PodName -n Namespace
```

View YAML:

```bash
kubectl get pod PodName -n Namespace -o yaml
```

Focus on:

```text
State
Last State
Reason: OOMKilled
Exit Code: 137
resources.requests.memory
resources.limits.memory
```

View Logs:

```bash
kubectl logs PodName -n Namespace --previous
```

Explanation:

```text
--previous
→ View log for last exit container instance
```

---

## 12. Initial Judgment of Memory Leaks

---

## Scenario 31: What is a Memory Leak

A memory leak can be simply understood as:

```text
The program applied for memory.
But they didn't release after they stopped using them.
Resulting in sustained growth of the process memory
```

Common Manifestations:

```text
Process RSS Rising

No decline in memory after recovery of business flows

Final trigger OOM

Memory restored after restarting service

Run for some time and rise again
```

---

## Scenario 32: Memory Increase is Not Necessarily a Leak

Process memory increase may also be normal behavior:

```text
Apply Cache Preheat

JVM Growing piles

Database buffer pool

Connect pool increase

Increase in business flows

page cache

Object Pool

Go runtime Memory management

Memory dispenser not returned to system immediately
```

Therefore, you cannot judge a leak just by memory increase.

---

## Scenario 33: Continuous Sampling of Process RSS

Check RSS of a process:

```bash
ps -o pid,rss,vsz,%mem,cmd -p PID
```

Continuous Observation:

```bash
while true; do date; ps -o pid,rss,vsz,%mem,cmd -p PID; sleep 10; done
```

You can also record to a file:

```bash
while true; do date; ps -o pid,rss,vsz,%mem,cmd -p PID; sleep 10; done >> /tmp/mem-watch-PID.log
```

---

## Scenario 34: Judging Whether It's Suspected of Leaking

Suspected leak characteristics:

```text
RSS Keep going one-way.

It's a low business.

GC And then it didn't drop.

Restore after restart

Run for some time and grow again

Growth does not fully match requests

No specified cache ceiling
```

Need to combine:

```text
Applied languages

Run-time

GC Log

Business flows

Version Change

Type of request

Dump

Code Analysis
```

---

## Scenario 35: Java Memory Leak Clues

Java processes need to distinguish:

```text
JVM Stack

Non stacking

Direct Memory

Metaspace

Thread Inn

Code Cache

JNI / native Memory
```

Basic Check:

```bash
jcmd PID VM.flags
```

```bash
jcmd PID VM.native_memory summary
```

If Native Memory Tracking is not enabled, you may not be able to use full NMT.

Check Heap:

```bash
jmap -heap PID
```

Export Heap Dump:

```bash
jmap -dump:format=b,file=/tmp/heap-PID.hprof PID
```

Production Notes:

```text
jmap dump Possible suspension of process and generation of large files
Production use must be carefully identified for disk space and operational effects
```

---

## Scenario 36: Go / Python / Node.js Memory Leaks

Different languages have different tools.

Go common:

```text
pprof

runtime metrics

heap profile
```

Python common:

```text
tracemalloc

objgraph

memory_profiler
```

Node.js common:

```text
heap snapshot

clinic.js

--inspect
```

Host-level troubleshooting can only first confirm:

```text
Which process memory growth

What are the growth trends?

Whether to accompany changes in flows and releases

Whether or not to trigger OOM
```

The root cause usually requires language runtime tools and code analysis.

---

## 13. Page Cache Troubleshooting

---

## Scenario 37: What is Page Cache

Page cache is a mechanism Linux uses to cache file data in memory.

Purpose:

```text
Improve file reading performance

Decrease Disk IO

Use free memory
```

Common Phenomena:

```text
buff/cache High

free Low

available Still high.
```

This is usually a normal phenomenon.

---

## Scenario 38: View Page Cache Related Fields

```bash
cat /proc/meminfo
```

Quick View:

```bash
egrep "Cached|Buffers|MemAvailable|Dirty|Writeback" /proc/meminfo
```

Field Explanation:

```text
Cached
→ File Page Cache

Buffers
→ Block device buffer

Dirty
→ Dirty Pages

Writeback
→ Writing back
```

---

## Scenario 39: Whether High Page Cache Needs Cleaning

General Conclusion:

```text
Don't be mad. cache Clean it up.
```

Reason:

```text
cache Yes. Linux Normal mechanisms

Recoverable when memory is tense

Clear cache May lead to subsequent reading IO Increase

It could slow down business.
```

Actually, we should focus on:

```text
available Is it low?

Is there any? swap Page Break

Is there any? OOM

Whether or not Dirty / Writeback Unusual

Whether Disk IO High
```

---

## Scenario 40: Risks of drop_caches

Command to clean page cache:

```bash
sync
```

```bash
echo 3 > /proc/sys/vm/drop_caches
```

Explanation:

```text
sync
→ First, brush the dirty data.

drop_caches
→ Release page cacheI don't know.dentryI don't know.inode cache
```

Production Risks:

```text
Could cause the cache to fail.

Follow-up requests for rereading disks

Disk IO Suspended

Slower operational response

Cannot solve the application memory leak

Not as a long-term solution
```

Production Recommendations:

```text
Don't. drop_caches It's a regular clean-up.
Only for temporary validation or special maintenance
Impact scope must be confirmed before implementation
```

---

## 14. Slab Memory Troubleshooting

---

## Scenario 41: What is Slab

Slab is a mechanism Linux kernel uses to manage kernel object caching.

Common cached objects:

```text
dentry

inode

kmalloc

buffer_head

Network Related Object

Filesystem Object
```

View `/proc/meminfo`:

```bash
egrep "Slab|SReclaimable|SUnreclaim" /proc/meminfo
```

Field Explanation:

```text
Slab
→ slab Total occupation

SReclaimable
→ Recoverable slab

SUnreclaim
→ Unrecoverable slab
```

---

## Scenario 42: View Slab with slabtop

```bash
slabtop
```

Common Focus:

```text
CACHE
→ Cache Name

OBJS
→ Number of Objects

OBJ/SLAB
→ Each slab Middle Objects

CACHE SIZE
→ Cache Occupancy Size
```

Common Large Items:

```text
dentry
inode_cache
xfs_inode
ext4_inode_cache
kmalloc
```

---

## Scenario 43: View /proc/slabinfo

```bash
cat /proc/slabinfo
```

Filter by Keyword:

```bash
grep -E "dentry|inode|xfs|ext4" /proc/slabinfo
```

---

## Scenario 44: High dentry / inode Cache

Common Causes:

```text
Extensive file access

A lot of small files.

Directory goes through a lot

Too many log directory files

Cache system generates a lot of files

Multiple container mirror layers and filesystem objects

Backup or scan programs run through a lot of files
```

Troubleshooting Directions:

```bash
df -hi
```

```bash
find /var/log -type f | wc -l
```

```bash
find /tmp -type f | wc -l
```

```bash
du -h --max-depth=1 /var | sort -hr | head
```

---

## Scenario 45: How to Judge Slab Anomalies

Need to combine historical baselines.

Suspicious Situations:

```text
Slab Sustained growth

SUnreclaim Sustained growth

available Continuous decline

No visible application process high memory

System end OOM

dentry / inode cache Very large.
```

Next Steps:

```text
Confirm if there are many small files

Confirm if there is a scanning, backup, log program

Confirm file system type

Confirm kernel. bug or drivers

Combining kernel versions and vendor support, as necessary
```

---

## 15. cgroup Memory Troubleshooting

---

## Scenario 46: Why to Check cgroup

In container and systemd environments, processes may be restricted by cgroup.

Manifested as:

```text
Host has sufficient memory

But the container or service still OOM

The process is restricted to one. memory limit Internal

Kubernetes Pod OOMKilled

Docker Containers OOMKilled
```

Therefore, distinguish between:

```text
Total host memory insufficient

Still? cgroup Inadequate restrictions
```

---

## Scenario 47: Check cgroup Version

```bash
stat -fc %T /sys/fs/cgroup
```

Common results:

```text
cgroup2fs
→ cgroup v2

tmpfs
→ Maybe. cgroup v1
```

Check mount points:

```bash
mount | grep cgroup
```

---

## Scenario 48: cgroup v2 Memory Files

Common files in cgroup v2:

```text
memory.current
→ Current Memory Usage

memory.max
→ Memory ceiling

memory.high
→ High water level limit

memory.events
→ Memory Event

memory.stat
→ Memory statistics
```

Check example:

```bash
cat /sys/fs/cgroup/memory.current
```

```bash
cat /sys/fs/cgroup/memory.max
```

```bash
cat /sys/fs/cgroup/memory.events
```

```bash
cat /sys/fs/cgroup/memory.stat
```

Explanation:

```text
The actual path depends on the process. cgroup
```

---

## Scenario 49: Find Process cgroup Path

```bash
cat /proc/PID/cgroup
```

Example:

```bash
cat /proc/12345/cgroup
```

Then locate the corresponding cgroup path based on the output.

---

## Scenario 50: cgroup v1 Memory Files

Common paths in cgroup v1:

```text
/sys/fs/cgroup/memory/
```

Common files:

```text
memory.usage_in_bytes
memory.limit_in_bytes
memory.max_usage_in_bytes
memory.failcnt
memory.stat
```

Check example:

```bash
cat /sys/fs/cgroup/memory/memory.usage_in_bytes
```

```bash
cat /sys/fs/cgroup/memory/memory.limit_in_bytes
```

```bash
cat /sys/fs/cgroup/memory/memory.failcnt
```

---

## 16. Docker Memory Troubleshooting

---

## Scenario 51: Check Container Memory Usage

```bash
docker stats
```

Check container details:

```bash
docker inspect ContainersID
```

Filter memory-related metrics:

```bash
docker inspect ContainersID | grep -i memory -A 20
```

Check for OOM:

```bash
docker inspect ContainersID | grep -i oom -A 10
```

---

## Scenario 52: Docker Memory Limit Parameters

Common runtime parameters:

```bash
docker run -d -m 512m nginx
```

Explanation:

```text
-m 512m
→ Limit maximum memory of container to 512MB
```

May also use:

```text
--memory
--memory-swap
--memory-reservation
```

Check:

```bash
docker inspect ContainersID
```

Key fields:

```text
HostConfig.Memory
HostConfig.MemorySwap
HostConfig.MemoryReservation
State.OOMKilled
```

---

## Scenario 53: Common Causes of Container OOM

```text
memory limit Set too small

Apply memory overuse limit

JVM Stack size not set by container limit

Flows surged

Batch Tasks

Memory Leak

It's too high inside.

Cache No Limits

Mirror or start parameters are not reasonable
```

Troubleshooting directions:

```text
Confirm if OOMKilled

Confirm. limit Reasonable

Confirm the application of memory parameters

Identification of recent traffic and version changes

Optimizing memory use

Adjustments if necessary limit
```

---

## 17. Kubernetes Memory Troubleshooting

---

## Scenario 54: Check Pod Resource Configuration

```bash
kubectl get pod PodName -n Namespace -o yaml
```

Focus on:

```text
resources.requests.memory

resources.limits.memory
```

---

## Scenario 55: Check if Pod was OOMKilled

```bash
kubectl describe pod PodName -n Namespace
```

Focus on:

```text
State

Last State

Reason: OOMKilled

Exit Code: 137

Events
```

---

## Scenario 56: Check Previous Container Logs

```bash
kubectl logs PodName -n Namespace --previous
```

If it's a multi-container Pod:

```bash
kubectl logs PodName -c Container Name -n Namespace --previous
```

---

## Scenario 57: Check Node Memory Pressure

```bash
kubectl describe node Node Name
```

Focus on:

```text
MemoryPressure

Allocated resources

Non-terminated Pods

Events
```

Check node resources:

```bash
kubectl top node
```

Check Pod resources:

```bash
kubectl top pod -n Namespace
```

Explanation:

```text
kubectl top Dependency metrics-server
```

---

## 18. Typical Memory Troubleshooting Scenarios

---

## Scenario 58: low free, high buff/cache

Phenomenon:

```text
free Low

buff/cache High

available Still high.

There's no sign of Carton.
```

Judgment:

```text
Not usually.
This is... Linux Normal use of empty memory for cache
```

Troubleshooting:

```bash
free -h
```

```bash
egrep "MemAvailable|Cached|Buffers|SwapTotal|SwapFree" /proc/meminfo
```

Not recommended to directly:

```bash
echo 3 > /proc/sys/vm/drop_caches
```

---

## Scenario 59: low available, high swap

Phenomenon:

```text
available Low

swap Use high

vmstat si / so High

System Carton.
```

Troubleshooting:

```bash
free -h
```

```bash
vmstat 1 5
```

```bash
ps -eo pid,ppid,user,%mem,%cpu,rss,cmd --sort=-rss | head -n 20
```

```bash
dmesg -T | grep -i oom
```

Judgment:

```text
Real memory pressure is high.
```

Troubleshooting directions:

```text
Find High Memory Process

Confirm if it's abnormal.

Temporary bleeding.

Optimizing Configuration

Expanded Memory

Limit abnormal processes

Reasons for the growth of duplicate memory
```

---

## Scenario 60: Process Memory Continuously Rising

Troubleshooting:

```bash
ps -o pid,rss,vsz,%mem,cmd -p PID
```

Continuously monitor:

```bash
while true; do date; ps -o pid,rss,vsz,%mem,cmd -p PID; sleep 10; done
```

Check memory mapping:

```bash
pmap -x PID | tail
```

Check smaps:

```bash
cat /proc/PID/smaps_rollup
```

Judgment direction:

```text
Whether memory leaks

Cache growth

Whether business flows result in

Is there a recent introduction issue?

Whether or not to run memory management mechanisms resulting in
```

---

## Scenario 61: Service OOM Killed

Troubleshooting:

```bash
dmesg -T | grep -i oom
```

```bash
journalctl -k | grep -i "killed process"
```

```bash
ps -eo pid,ppid,user,%mem,%cpu,rss,cmd --sort=-rss | head -n 20
```

If it's a container:

```bash
docker inspect ContainersID | grep -i oom -A 10
```

If it's Kubernetes:

```bash
kubectl describe pod PodName -n Namespace
```

```bash
kubectl logs PodName -n Namespace --previous
```

---

## Scenario 62: Host Memory Normal, Container OOM

Troubleshooting:

```bash
free -h
```

```bash
docker stats
```

```bash
docker inspect ContainersID | grep -i memory -A 20
```

```bash
docker inspect ContainersID | grep -i oom -A 10
```

Judgment:

```text
Probably the container. memory limit Too small or application exceeding container limits
```

Kubernetes check:

```bash
kubectl get pod PodName -n Namespace -o yaml
```

```bash
kubectl describe pod PodName -n Namespace
```

---

## Scenario 63: Abnormal Slab Usage

Troubleshooting:

```bash
egrep "Slab|SReclaimable|SUnreclaim" /proc/meminfo
```

```bash
slabtop
```

```bash
cat /proc/slabinfo | head
```

If dentry / inode are high, continue:

```bash
df -hi
```

```bash
find /tmp -type f | wc -l
```

```bash
find /var/log -type f | wc -l
```

```bash
find /data -xdev -type f | wc -l
```

---

## 19. Production Memory Troubleshooting Notes

---

## 1. Don't Only Check free

Correct order:

```text
Look first. available

Look again. swap

Look again. si / so

Look again. OOM

Look at the process. RSS
```

---

## 2. Don't Arbitrarily drop_caches

`drop_caches` is not a conventional optimization method.

Risks:

```text
Cause Cache to expire

Read IO Suspended

Slower operations

Cover up real problems.
```

---

## 3. Don't Restart Services Directly When Memory is High

At least collect before restarting:

```bash
free -h
```

```bash
vmstat 1 5
```

```bash
ps -eo pid,ppid,user,%mem,%cpu,rss,cmd --sort=-rss | head -n 20
```

```bash
cat /proc/meminfo
```

```bash
dmesg -T | grep -i oom
```

---

## 4. After OOM, Investigate Root Cause, Not Just Restart Services

Need to confirm:

```text
It's the host. OOM Still? cgroup OOM

Is the memory limit too small or is the application abnormal?

Is there a surge in traffic?

Is there a version published?

Whether there are batches

Existence of memory leakage

Need to adjust resource constraints
```

---

## 5. Java / Go / Python / Node.js Need to Combine with Runtime Tools

Host layer can only answer:

```text
Which process is RAM

What's your share?

Sustained growth

Whether or not OOM

Whether or not cgroup Limits
```

To truly locate code-level memory leaks, need language runtime tools.

---

## 6. Container Memory Must Check Limit

Container troubleshooting cannot only check host.

Must check:

```text
Current Use of Containers

Containers memory limit

Whether or not OOMKilled

Pod requests / limits

Nodes MemoryPressure
```

---

## 20. Common Commands in This Article

---

## Basic Memory

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
swapon --show
```

```bash
vmstat 1 5
```

---

## /proc/meminfo

```bash
cat /proc/meminfo
```

```bash
egrep "MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree|Dirty|Writeback|Slab|SReclaimable|SUnreclaim|PageTables|Committed_AS|CommitLimit" /proc/meminfo
```

```bash
egrep "Cached|Buffers|MemAvailable|Dirty|Writeback" /proc/meminfo
```

```bash
egrep "Slab|SReclaimable|SUnreclaim" /proc/meminfo
```

---

## High Memory Processes

```bash
top
```

```bash
ps -eo pid,ppid,user,%mem,%cpu,rss,vsz,cmd --sort=-rss | head -n 20
```

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -n 20
```

```bash
ps -o pid,ppid,user,%mem,%cpu,rss,vsz,cmd -p PID
```

```bash
cat /proc/PID/status
```

```bash
egrep "VmSize|VmRSS|VmData|VmStk|VmExe|VmLib|Threads" /proc/PID/status
```

---

## Process Memory Mapping

```bash
pmap PID
```

```bash
pmap -x PID | tail
```

```bash
cat /proc/PID/smaps_rollup
```

```bash
egrep "Rss|Pss|Private_Clean|Private_Dirty|Shared_Clean|Shared_Dirty|Swap" /proc/PID/smaps_rollup
```

---

## smem

```bash
smem -r -k
```

```bash
smem -r -k -s pss
```

---

## OOM

```bash
dmesg -T | grep -i oom
```

```bash
dmesg -T | grep -i "killed process"
```

```bash
journalctl -k | grep -i oom
```

```bash
journalctl -k | grep -i "killed process"
```

```bash
cat /proc/PID/oom_score
```

```bash
cat /proc/PID/oom_score_adj
```

---

## Memory Trend Observation

```bash
ps -o pid,rss,vsz,%mem,cmd -p PID
```

```bash
while true; do date; ps -o pid,rss,vsz,%mem,cmd -p PID; sleep 10; done
```

```bash
while true; do date; ps -o pid,rss,vsz,%mem,cmd -p PID; sleep 10; done >> /tmp/mem-watch-PID.log
```

---

## Java Memory

```bash
jcmd PID VM.flags
```

```bash
jcmd PID VM.native_memory summary
```

```bash
jmap -heap PID
```

```bash
jmap -dump:format=b,file=/tmp/heap-PID.hprof PID
```

---

## page cache

```bash
sync
```

```bash
echo 3 > /proc/sys/vm/drop_caches
```

---

## slab

```bash
slabtop
```

```bash
cat /proc/slabinfo
```

```bash
grep -E "dentry|inode|xfs|ext4" /proc/slabinfo
```

---

## cgroup

```bash
stat -fc %T /sys/fs/cgroup
```

```bash
mount | grep cgroup
```

```bash
cat /proc/PID/cgroup
```

cgroup v2 Example:

```bash
cat /sys/fs/cgroup/memory.current
```

```bash
cat /sys/fs/cgroup/memory.max
```

```bash
cat /sys/fs/cgroup/memory.events
```

```bash
cat /sys/fs/cgroup/memory.stat
```

cgroup v1 Example:

```bash
cat /sys/fs/cgroup/memory/memory.usage_in_bytes
```

```bash
cat /sys/fs/cgroup/memory/memory.limit_in_bytes
```

```bash
cat /sys/fs/cgroup/memory/memory.failcnt
```

---

## Docker

```bash
docker stats
```

```bash
docker ps -a
```

```bash
docker inspect ContainersID
```

```bash
docker inspect ContainersID | grep -i memory -A 20
```

```bash
docker inspect ContainersID | grep -i oom -A 10
```

---

## Kubernetes

```bash
kubectl get pod -n Namespace
```

```bash
kubectl describe pod PodName -n Namespace
```

```bash
kubectl get pod PodName -n Namespace -o yaml
```

```bash
kubectl logs PodName -n Namespace --previous
```

```bash
kubectl logs PodName -c Container Name -n Namespace --previous
```

```bash
kubectl describe node Node Name
```

```bash
kubectl top node
```

```bash
kubectl top pod -n Namespace
```

---

## Tool Installation

Ubuntu / Debian Install smem:

```bash
apt install -y smem
```

RHEL / CentOS / Rocky / AlmaLinux Install smem:

```bash
yum install -y smem
```

Or:

```bash
dnf install -y smem
```

---

## Twenty-one, One-Sentence Summary

The core of advanced memory troubleshooting isn't just monitoring memory usage rate, but to determine:

```text
It's a real memory deficit.

Still? Linux Normal Cache

The host's memory is inadequate

Still? cgroup / Containers limit Too small.

It's an application memory leak.

Or is business cache growing normal?

Yes. page cache High

Still? slab Unusual growth
```

Basic judgment chain:

```text
free -h

→ Look. available / buff/cache / swap

→ vmstat Look. si / so

→ ps/top High Memory Process

→ /proc/meminfo Look at the breakdown.

→ dmesg / journalctl Cha. OOM
```

OOM troubleshooting chain:

```text
dmesg / journalctl Cha. OOM

→ Confirmed by kill Process

→ Distinct host OOM and cgroup OOM

→ View Process RSS

→ View container limit / Pod limit

→ Continue analysis of traffic, version, cache, leakage
```

Initial memory leak judgment:

```text
Process RSS Rising

→ It's a low business.

→ Restore after restart

→ Run for a while and rise.

→ Analyse further with the language running time tool
```

page cache judgment:

```text
buff/cache High is not necessarily unusual.

available It's all the more important to be alert.

Don't be silly. drop_caches

cache Yes. Linux Normal mechanisms
```

slab judgment:

```text
Slab Highlight. SReclaimable and SUnreclaim

dentry / inode More than a lot of small files.

SUnreclaim Sustained growth requires more attention
```

Container memory judgment:

```text
It's not like the container doesn't exist. OOM

Containers OOM Priority. memory limit

Kubernetes Priority. requests / limits / OOMKilled / Exit Code 137
```

Production recommendations:

```text
Don't just look. free

Don't be silly. drop_caches

Do not see the memory high and restart.

OOM Then you have to distinguish between the host and the host. cgroup

The container should be checked. limit

The memory leak is ultimately combined with the application of running time tools and code analysis
```