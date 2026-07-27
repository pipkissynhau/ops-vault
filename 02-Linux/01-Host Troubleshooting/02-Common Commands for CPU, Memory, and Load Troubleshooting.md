# 02 - Common Commands for CPU, Memory, and Load Troubleshooting

# Linux # Operations and Maintenance # Troubleshooting # CPU # Memory # Load # top # ps # mpstat # pidstat # pmap # vmstat # Performance Troubleshooting

---

## Recommended Path

01 - Linux Basics and Host Operations and Maintenance / 01 - Host Troubleshooting / 02 - Common Commands for CPU, Memory, and Load Troubleshooting.md

---

## I. Document Description

This document compiles common troubleshooting commands related to **CPU, memory, and load** on Linux hosts.

Key points include:

- Checking CPU usage
- Identifying high-CPU-consuming processes
- Monitoring multi-core CPU utilization
- Conducting thread-level CPU analysis
- Checking memory usage
- Locating high-memory-consuming processes
- Viewing process memory maps
- Understanding the load average
- Determining if high load is due to high CPU usage or other factors
- Using `top`, `ps`, `mpstat`, and `pidstat`
- Using `free`, `pmap`, and `vmstat`
- Checking `uptime`

The goal is:

- To quickly determine if there are any CPU-related issues.
- To locate processes that are consuming a large amount of CPU resources.
- To identify memory shortages.
- To locate processes that are using up a lot of memory.
- To understand the relationship between the load average and the number of CPU cores.
- To distinguish between different types of load caused by CPU pressure, memory pressure, or IO wait.

---

## II. General Approach to CPU, Memory, and Load Troubleshooting

Don’t rely on just one command when troubleshooting CPU, memory, and load issues.

Recommended order:

```text
uptime
→ Check the load average.
top
→ View CPU usage, memory usage, and process occupancy.
free -h
→ Check memory and swap space.
vmstat 1 5
→ Monitor r, b, si, so, and wa values.
ps (sorted)
→ Identify specific high-CPU-/high-memory-consuming processes.
mpstat/pidstat
→ Get more detailed information about CPU core usage and process-level statistics.
```

Common scenarios for further investigation:

```text
High load + High CPU usage
→ Likely due to heavy computational tasks.
High load + High wa values
→ Possibly caused by disk I/O delays.
High load + High b values
→ There may be many processes in uninterruptible sleep mode.
Low available memory + High swap space usage
→ Indicates significant memory pressure.
A single process with high CPU usage
→ Further investigate at the process level.
A single process with high memory usage
→ Check for memory leaks or cache issues.
```

---

## III. Load Troubleshooting: uptime and load average

---

## Scenario 1: Checking System Load

### Command

```bash
uptime
```

### Purpose

`uptime` displays:

- The system’s running time.
- The number of currently logged-in users.
- The load average.

Example output:

```text
load average: 0.52, 0.60, 0.70
```

Explanation:

- The first value represents the average load in the last 1 minute.
- The second value represents the average load in the last 5 minutes.
- The third value represents the average load in the last 15 minutes.

---

## Scenario 2: Checking the Number of CPU Cores

### Command

```bash
nproc
```

or:

```bash
lscpu
```

### Note

The load average must be interpreted in conjunction with the number of CPU cores. Just because the load average is high doesn’t necessarily mean there’s a problem. For example:

- On a 2-core machine, a permanent load average of 8 may indicate high pressure.
- On a 16-core machine, a permanent load average of 8 might not be abnormal; further analysis of CPU usage, I/O, and memory is needed.

---

## Scenario 3: Interpreting the Load Average

### Judgment Logic

```text
If the load average is consistently lower than the number of CPU cores, the system’s performance is generally acceptable.
If the load average is close to the number of CPU cores, it requires attention.
If the load average is significantly higher than the number of CPU cores, the system is under significant pressure and further investigation is needed.
```

### Examples

```text
On a 4-core machine, a load average of 1–2 is usually normal.
On a 4-core machine, a load average of around 4 indicates near-full capacity.
On a 4-core machine, a load average of over 10 suggests high pressure and requires further analysis.
```

---

## Scenario 4: High Load Doesn’t Necessarily Mean High CPU Usage

High load can be caused by:

- Heavy computational tasks.
- Disk I/O delays.
-🔤 Compared to the instantaneous results provided by `ps`, `pidstat` is more suitable for observing the trend of a process's CPU usage over a period of time.```bash
vmstat 1 5
```