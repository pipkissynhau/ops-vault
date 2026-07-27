# 10-Advanced Memory Troubleshooting: OOM, Memory Leaks, Page Cache, Slab, and Cgroups

# Linux # Operations and Maintenance # SRE # Memory Troubleshooting # OOM # Memory Leaks # Pagecache # Slab # Cgroup # Container Memory # Performance Analysis

---

## Recommended Reading Path

01-Linux Basics and Host Operations/01-Host Troubleshooting/10-Advanced Memory Troubleshooting: OOM, Memory Leaks, Page Cache, Slab, and Cgroups.md

---

## I. Document Overview

This document outlines advanced methods for troubleshooting memory issues on Linux hosts, focusing on:

- The Linux memory usage model
- How to correctly interpret `free` report results
- The relationship between `available`, `buff/cache`, and `swap`
- How to monitor process memory consumption
- Differences between VIRT, RES, and SHR
- Basic understanding of RSS, PSS, and USS
- Troubleshooting OOM Killer
- Distinguishing between host OOM and container OOM
- Preliminary identification of memory leaks
- Page cache troubleshooting
- Slab memory analysis
- Cgroup-based memory limitation issues
- Container memory restriction checks using Docker/Kubernetes
- Risks associated with `drop_caches`
- Key considerations for production-level memory troubleshooting

This document is part of the advanced performance analysis series in the host troubleshooting category.

In the previous section 02, basic memory commands were covered. This section delves deeper into memory mechanisms and practical troubleshooting approaches.

The objectives are:

- To be able to determine whether Linux truly has insufficient memory
- To understand why a low `free` value does not necessarily indicate an issue
- To identify processes with high memory usage
- To investigate the causes of OOM occurrences
- To distinguish between host OOM and container OOM
- To make preliminary judgments about memory leaks
- To comprehend the significance of page cache and slab usage
- To diagnose abnormalities caused by cgroup memory restrictions

---

## II. Common Symptoms of Memory Issues

In production environments, memory problems often manifest as follows:

```text
Notable system lag

Slow SSH logins

Reduced application response times

Sudden service crashes

Container termination

Pod status showing OOMKilled

Java processes being killed

System logs displaying "Out of memory"

Continuous increase in swap usage

High load levels but low CPU utilization

Sudden surge in disk I/O operations

Slow database connections

Abnormal restarts of Nginx/application processes

Monitoring alerts indicating high memory usage
```

Common root causes include:

```text
Insufficient physical memory

Application-level memory leaks

Uncontrolled cache growth

Excessive number of processes or threads

High page cache utilization that is not recyclable

Abnormal slab memory growth

Frequent swap paging

Too small cgroup memory limits

Inappropriate container memory restrictions

Improper JVM heap configuration parameters

 처리 of large queries, objects, or files

The presence of malfunctioning processes in the system
```

---

## III. General Memory Troubleshooting Process

Recommended advanced memory troubleshooting steps:

```text
free -h
→ Check available memory, buff/cache, and swap usage

vmstat 1 5
→ Verify if there is frequent swap paging

top / ps
→ Identify processes with high memory consumption

/proc/meminfo
→ View detailed memory statistics

dmesg / journalctl
→ Search for OOM-related messages

pmap / smem
→ Analyze process memory distribution

slabtop / /proc/slabinfo
→ Examine slab memory usage

cgroup
→ Check container or service memory limits

docker stats / kubectl describe
→ Investigate container OOM issues
```

Common troubleshooting paths based on initial observations include:

```text
Low available memory + high swap si/so
→ Indicating significant physical memory pressure

Notably high buff/cache
→ Usually due to normal caching activities

Continuous increase in a specific process's RSS
→ Possible memory leak or cache growth

OOM records in dmesg
→ Verify the terminated process and the trigger event

Pod OOMKilled
→ First check container limits; it may not be due to host memory shortages

Persistent slab memory growth
→ Possible issues with kernel caches, dentry/inodes, or kernel objects

High page cache usage
→ Possibly caused by extensive file I/O operations; not necessarily a fault
```

---

## IV. Correct Interpretation of `free` Reports

---

## Scenario 1: Checking Memory Usage

```bash
free -h
```

Common output fields:

```text
total
→ Total memory

used
→ Used memory

free
→ Free memory

shared
→ Shared memory

buff/cache
→ Buffer and cache memory

available
→ Estimated available memory for new processes
```

Key fields to focus on:

```text
available

swap

buff/cache
```

```text
ps -eo pid,rss,vsz,cmd --sort=-rss | head -n 20
```🔤 Translate the following Chinese text into English:

```bash
ps -o pid,rss,vsz,%mem,cmd -p PID
```

Continuous observation:

```bash
while true; do date; ps -o pid,rss,vsz,%mem,cmd -p PID; sleep 10; done
```

It can also be recorded to a file:

```bash
while true; do date; ps -o pid,rss,vsz,%mem,cmd -p PID; sleep 10; done >> /tmp/mem-watch-PID.log
```

---

## Scenario 34: Determining Suspected Memory Leaks

Characteristics of suspected leaks:

```text
The RSS value continuously increases monotonically.
It does not decrease during off-peak business hours.
There is no significant decrease after garbage collection (GC).
The issue reappears after a system restart.
The increase in memory usage is not entirely consistent with the number of requests.
There is no clear upper limit for cache usage.
```

It is necessary to consider the following factors in combination:

```text
Application language
Runtime environment
GC logs
Business traffic
Version changes
Type of requests
Heap dumps
Code analysis
```

---

## Scenario 35: Clues to Java Memory Leaks

In Java processes, it is important to distinguish between the following types of memory:

```text
JVM heap
Non-heap memory
Direct memory
Metaspace
Thread stacks
Code Cache
JNI/native memory
```

Basic checks include:

```bash
jcmd PID VM_flags
```

```bash
jcmd PID VM_native_memory summary
```

If Native Memory Tracking is not enabled, it may be impossible to use the full functionality of NMT.

To view the heap memory:

```bash
jmap -heap PID
```

To export a heap dump:

```bash
jmap -dump:format=b,file=/tmp/heap-PID.hprof PID
```

Production considerations:

```text
Running `jmap dump` may cause the process to pause and generate a large file.
It must be used with caution in production environments. Ensure that there is sufficient disk space and that the impact on business operations is minimal.
```

---

## Scenario 36: Memory Leaks in Go/Python/Node.js

Different programming languages require different tools for detecting memory leaks.

Common tools for Go:

```text
pprof
Runtime metrics
Heap profile
```

Common tools for Python:

```text
tracemalloc
objgraph
memory_profiler
```

Common tools for Node.js:

```text
heap snapshot
clinic.js
--inspect
```

At the host level, the first steps are to determine:

```text
Which process is experiencing increasing memory usage.
What is the trend of this increase?
Is it related to changes in traffic or software versions?
Has it caused out-of-memory errors (OOM)?
```

The true root cause usually requires the use of language-specific runtime tools and code analysis.

---

## Section 13: Investigating Page Cache Issues

---

## Scenario 37: What is Page Cache?

Page cache is a mechanism in Linux that uses memory to cache file data.

Its benefits include:

```text
Improving file read performance.
Reducing disk I/O operations.
Making use of available memory.
```

Common observations include:

```text
A high value in the `buff/cache` column.
A very low value in the `free` column.
An still relatively high value in the `available` column.
```

In most cases, these are normal phenomena.

---

## Scenario 38: Viewing Page Cache-related Fields

```bash
cat /proc/meminfo
```

For a quick check:

```bash
egrep "Cached|Buffers|MemAvailable|Dirty|Writeback" /proc/meminfo
```

Field explanations:

```text
Cached
→ File page cache.
Buffers
→ Block device buffers.
Dirty
→ Dirty pages.
Writeback
→ Pages that are currently being written back to disk.
```

---

## Scenario 39: Should Page Cache be Cleared When It Is High?

The general rule is:

```text
Do not clear the page cache just because its value is high.
```

Reasons include:

```text
Page cache is a normal mechanism in Linux.
It can be reclaimed when memory is scarce.
Clearing the cache may increase subsequent read I/O operations and slow down business processes.
```

The main concerns should be:

```text
Is the available memory amount low?
Is there any swap paging occurring?
Are there out-of-memory errors (OOM)?
Are there abnormal values in the `Dirty` or `Writeback` fields?
Is the disk I/O load high?
```

---

## Scenario 40: Risks of Using `drop_caches`

Commands to clear page cache include:

```bash
sync
```

```bashcat /sys/fs/cgroup/memory/memory.failcnt```bash
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
egrep "cached|buffers|MemAvailable|dirty|writeback" /proc/meminfo
```

```bash
egrep "slab|sreclaimable|sunreclaim" /proc/meminfo
```

---

## High-memory processes

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

## Process memory mapping

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

## Memory trend observation

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

## Java memory

```bash
jcmd PID VM.flags
```

```bash
jcmd PID VM_native_memory summary
```

```bash
jmap -heap PID
```

```bash
jmap -dump:format=b,file=/tmp/heap-PID.hprof PID
```

---

## Page cache

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

cgroup v2 examples:

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

cgroup v1 examples:

```bash
cat /sys/fs/cgroup/memory/memory_usage_in_bytes
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
docker inspect container_id
```

```bash
docker inspect container_id | grep -i memory -A 20
``