# 03-Disk Space, Inodes, and Disk I/O Troubleshooting

#Linux #Ops #Troubleshooting #Disk Space #Inodes #Disk I/O #df #du #iostat #iotop #pidstat #dmesg #File System

---

## Recommended Path

01-Linux Basics and Host Ops/01-Host Troubleshooting/03-Disk Space, Inodes, and Disk I/O Troubleshooting.md

---

## I. Document Description

This document compiles Linux host-related troubleshooting commands for **disk space, inodes, and disk I/O**.

Key points include:

- Disk space troubleshooting
- Inode usage analysis
- Large directory identification
- Large file detection
- Log spike investigation
- Checking disk and mount relationships
- Determining file system type
- Disk I/O analysis
- Identifying processes with heavy disk read/write activities
- `df`
- `du`
- `find`
- `lsblk`
- `fdisk`
- `blkid`
- `iostat`
- `iotop`
- `pidstat -d`
- `dmesg`

Objectives:

- Quickly determine if the disk is full.
- Differentiate between space exhaustion and inode exhaustion.
- Locate which directories or files are consuming significant space.
- Identify abnormal disk I/O patterns.
- Determine which processes are heavily using the disk.
- Lay the groundwork for subsequent disk expansion, LVM configuration adjustments, and log management.

---

## II. Common Disk Issues in Production Environments

In production settings, disk-related problems often manifest as:

```text
Services unable to write logs
Applications failing to start
Databases unable to store data
Nginx returning 500/502 errors
File uploads failing
Containers or Pods failing to launch
Slow system login processes
Command execution delays
High machine load but low CPU usage
Rapid expansion of log directories
Scheduled tasks failing to execute
Disk alerts triggered
```

Common causes include:

```text
Insufficient disk space
Exhausted inodes
Large log files
Untempted temporary files
Accumulated backup files
Over-sized database data directories
Docker/containerd data directories becoming full
High disk I/O load
Physical disk hardware failures
File system malfunctions
NFS/network storage delays
```

---

## III. General Approach to Disk Troubleshooting

The recommended order for disk troubleshooting is:

```text
df -h
→ Check which mount point has run out of space.
df -hi
→ Verify if inodes are exhausted.
du -h --max-depth=1
→ Identify the largest first-level directory.
find
→ Locate large or old files.
lsblk -f
→ List disks, partitions, file systems, and mount points.
iostat -x 1 5
→ Assess disk I/O performance.
iotop -o -P
→ Determine which processes are accessing the disk.
dmesg
→ Check for any disk, file system, or hardware errors.
```

Common branches of investigation include:

```text
If `df -h` shows full space:
→ Focus on storage capacity issues.
If `df -hi` shows full inodes:
→ Investigate excessive small files.
If `du` identifies large directories:
→ Further drill down into subdirectories.
If `iostat` shows high %util or await values:
→ Check for disk I/O bottlenecks or storage limitations.
If `iotop` highlights a particular process with high I/O:
→ Evaluate its role based on business context and logs.
If `dmesg` reports disk/file system errors:
→ Pay attention to potential hardware, driver, or file system issues.
```

---

## IV. df: Checking Disk Space and Inodes

---

## Scenario 1: Checking Disk Space Usage

### Command

```bash
df -h
```

### Purpose

Display the disk space usage for each mounted file system.

Key details to focus on:

```text
Filesystem

Size

Used

Avail

Use%

Mounted on
```

Common interpretations:

```text
If Use% exceeds 80%:
→ Requires attention.
If Use% exceeds 90%:
→ Requires further investigation.
If Use% is close to 100%:
→ High risk; may impact business operations.
```

---

## Scenario 2: Displaying File System Types

### Command

```bash
df -hT
```

### Purpose

Show the file system type in addition to disk space information.

Common file systems include:

```text
xfs

ext4

tmpfs

overlay

nfs

cifs
```

Use cases:

```text
Determine if it's a local disk or network storage.
Identify if it's a standard file system or tmpfs.
Check if it belongs to Docker overlay or the host directory.
```

---

## Scenario 3:```bash
find /var/log -type f -name "*.gz" | wc -l
``````bash
for dir in /*; do echo "$dir"; find "$dir" -xdev -type f 2>/dev/null | wc -l; done
```---

## Tool Installation

To install sysstat on Ubuntu/Debian:

```bash
apt install -y sysstat
```

To install sysstat on RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y sysstat
```

Or:

```bash
dnf install -y sysstat
```

To install iotop on Ubuntu/Debian:

```bash
apt install -y iotop
```

To install iotop on RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y iotop
```

Or:

```bash
dnf install -y iotop
```

To install dstat on Ubuntu/Debian:

```bash
apt install -y dstat
```

To install dstat on RHEL/CentOS/Rocky/AlmaLinux:

```bash
yum install -y dstat
```

Or:

```bash
dnf install -y dstat
```

---

## Summary in Sixteen Sentences

The core of disk troubleshooting is to:

```text
First, determine whether the issue lies with space, inodes, or I/O.
```

For space-related troubleshooting:

```text
df -h

→ du -h --max-depth=1

→ Identify large files

→ Check if logs, backups, data directories, or Docker folders are consuming space.
```

For inode-related troubleshooting:

```text
df -hi

→ Use find to count the number of files

→ Locate directories with many small files

→ Clean up expired small files and address their origin.
```

For I/O-related troubleshooting:

```text
Use top/vmstat to check wa values

→ Run iostat -x 1 5 to analyze devices

→ Use iotop -o -P to monitor processes

→ Check pidstat -d for process-level read/write activities

→ Combine these with business tasks, logs, databases, and backup information for a comprehensive analysis.
```

Production recommendations:

```text
Do not delete files without careful consideration.

Avoid indiscriminately clearing volume or database directories.

Never remove logs that are still being written to.

Do not manually alter /var/lib/docker folders.

When I/O performance is low, analyze it in context with business operations.

In case of file system errors, prioritize data protection.
```