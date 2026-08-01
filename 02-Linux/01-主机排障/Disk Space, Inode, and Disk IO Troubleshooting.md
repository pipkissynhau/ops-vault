# 03-Disk Space, Inode, and Disk IO Troubleshooting

#Linux #Transport #TheBarrier. #DiskSpace #inode #DiskIo #df #du #iostat #iotop #pidstat #dmesg #FileSystem

---

## Recommended Path

01-Linux Foundation and Host Maintenance/01-Host Troubleshooting/03-Disk Space, Inode, and Disk IO Troubleshooting.md

---

## I. Document Description

This document organizes troubleshooting commands related to **disk space, inode, and disk IO** in Linux hosts.

Key focuses of this document include:

- Disk space troubleshooting
- Inode usage troubleshooting
- Large directory identification
- Large file identification
- Log explosion troubleshooting
- Disk and mount relationship inspection
- File system type inspection
- Disk IO troubleshooting
- Identifying processes with heavy disk read/write
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

The goal is:

To quickly determine if the disk is full

→ Distinguish between space exhaustion and inode exhaustion

→ Locate which directory or file is consuming space

→ Determine if disk IO is abnormal

→ Identify which process is performing heavy disk read/write

→ Lay the foundation for subsequent disk expansion, LVM expansion, and log management

---

## II. Common Disk Issues in Production

In production environments, disk-related issues often manifest as:

```text
Service cannot log

Application startup failed

Database cannot be written

Nginx Back 500 / 502

Failed to upload file

or Pod Starting Failed

Slower system login

Order to execute Carden.

Machine load High though. CPU Not high.

Log directory rapidly expanding

Time job failed

Disk alert trigger.
```

Common root causes include:

```text
Disk Space Full

inode Exhaustion

Log file too big

Provisional documents not cleared

Backup File Stack

Database data catalogues are too large

Docker / containerd Data directories are too big

Disk IO Full

Disk hardware anomaly

File system anomaly

NFS / Network Storage Carden
```

---

## III. Overall Disk Troubleshooting Approach

Recommended disk troubleshooting sequence:

```text
df -h
→ Look which mounts are full.

df -hi
→ Look. inode Full

du -h --max-depth=1
→ Which one's bigger?

find
→ Looking for big papers or old ones.

lsblk -f
→ Look at disks, partitions, file systems, mount points.

iostat -x 1 5
→ Look at the disk. IO High

iotop -o -P
→ Look which process is reading and writing disks.

dmesg
→ Look at disks, file systems, hardware errors.
```

CommonDiversion:

```text
df -h Full
→ Space capacity issues

df -hi Full
→ inode Problem. Too many files.

du Found Large Directory
→ Keep drilling down.

iostat %util / await High
→ Disk IO Pressure or storage bottlenecks

iotop A process IO High
→ Combining business and log judgements

dmesg Yes. disk / filesystem error
→ Focus on hardware, drive, file system anomalies
```

---

## IV. df: View Disk Space and Inode Usage

---

## Scenario 1: View Disk Space Usage

### Command

```bash
df -h
```

### Purpose

View disk space usage for each filesystem.

Key focus:

```text
Filesystem

Size

Used

Avail

Use%

Mounted on
```

Common judgment:

```text
Use% Over 80%
→ Needing attention.

Use% Over 90%
→ We need to check.

Use% Close 100%
→ High risk, which may have affected operations
```

---

## Scenario 2: Display File System Type

### Command

```bash
df -hT
```

### Purpose

Display file system type based on `df -h`.

Common file systems:

```text
xfs

ext4

tmpfs

overlay

nfs

cifs
```

Suitable for determining:

```text
Is this local disk or network storage?

Is this a normal file system or something? tmpfs

This is... Docker overlay Relevant directory or host directory
```

---

## Scenario 3: View Inode Usage

### Command

```bash
df -hi
```

### Purpose

View inode usage.

Inode can be simply understood as:

```text
Documentation volume resources
```

Even if there is remaining disk space, if inodes are exhausted, new files cannot be created.

Common phenomena:

```text
No space left on device
```

But executing:

```bash
df -h
```

shows that space is not full.

At this point, check:

```bash
df -hi
```

---

## Scenario 4: Difference Between Space Exhaustion and Inode Exhaustion

### Space Exhaustion

```text
df -h Medium Use% Close 100%
```

Common causes:

```text
Big Log File

Big Backup File

Database data growth

Docker Mirror and container layer growth

Too many upload files
```

### Inode Exhaustion

```text
df -hi Medium IUse% Close 100%
```

Common causes:

```text
A lot of small files.

Mass session Documentation

Lots of cache files

A large number of temporary documents

Lots of small log files

Program Abnormal Create File
```

---

## Scenario 5: View Mount Point of Specified Directory

### Command

```bash
df -h /var/log
```

```bash
df -h /data
```

```bash
df -h /var/lib/docker
```

### Purpose

Confirm which mount point a directory belongs to.

For example:

```bash
df -h /var/log
```

can confirm whether `/var/log` is on the root partition `/`.

Common issues in production:

```text
Log directory in root partition
→ The log goes up and the system is full.
```

---

## V. du: Locate Large Directories and Files

---

## Scenario 6: View Size of Items in Current Directory

### Command

```bash
du -sh *
```

### Purpose

View the total size of each file or directory in the current directory.

Common usage:

```bash
cd /
```

```bash
du -sh *
```

Used to locate the largest top-level directory from the root directory.

---

## Scenario 7: View Size of Items in Specified Directory

### Command

```bash
du -sh /var/log/*
```

```bash
du -sh /data/*
```

```bash
du -sh /home/*
```

### Purpose

Locate which subdirectory or file in the specified directory consumes the most space.

Common troubleshooting:

```text
/var/log Is the log too big?

/data Excessive operational data

/home Too many user files

/var/lib/docker Whether or not Docker Data is too big.
```

---

## Scenario 8: Limit Directory Depth to View Size

### Command

```bash
du -h --max-depth=1 /data
```

```bash
du -h --max-depth=1 /var
```

```bash
du -h --max-depth=1 /var/log
```

### Purpose

Only view space usage of the first level under the specified directory.

Suitable for drilling down step by step:

```text
Look first. /
→ Look again. /var
→ Look again. /var/log
→ Look at the specific service catalogue.
```

---

## Scenario 9: Sort by Size to View Directories

### Command

```bash
du -h --max-depth=1 /var | sort -h
```

Reverse order:

```bash
du -h --max-depth=1 /var | sort -hr
```

Take top 20:

```bash
du -h --max-depth=1 /var | sort -hr | head -n 20
```

### Purpose

Quickly identify the largest directory.

---

## Scenario 10: Discrepancy Between du and df Results

Sometimes it appears:

```text
df Show disk full
du I couldn't find the file.
```

Common causes:

```text
The file has been deleted, but the process still occupies the handle of the document

Mount point overwrite caused du Could not close temporary folder: %s

Due to inadequate authority du No complete statistics

Filesystem Save Space

Containers overlay or special file system impact
```

Focus on files that are deleted but still being used by processes.

Command:

```bash
lsof | grep deleted
```

If there is no `lsof`, need to install or use other methods to confirm.

---

## VI. find: Search for Large Files, Old Files, and Small File Accumulation

---

## Scenario 11: Find Files Larger than 100M

### Command

```bash
find /var/log -type f -size +100M
```

Find files larger than 1G in the root directory:

```bash
find / -type f -size +1G 2>/dev/null
```

Show detailed information:

```bash
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

### Purpose

Quickly locate large files.

---

## Scenario 12: Find Logs Modified in the Last 7 Days

### Command

```bash
find /var/log -type f -name "*.log" -mtime -7
```

### Purpose

Check recently growing or modified log files.

---

## Scenario 13: Find Logs from 7 Days Ago

### Command

```bash
find /var/log -type f -name "*.log" -mtime +7
```

### Purpose

Suitable for investigating whether old logs have accumulated.

---

## Scenario 14: Find Large Files from 30 Days Ago

### Command

```bash
find /data -type f -mtime +30 -size +100M -exec ls -lh {} \;
```

### Purpose

Suitable for finding historical backups, historical logs, temporary packages, etc., that can be archived or cleaned up.

---

## Scenario 15: Locate Directories withMass Small Files

### Command

Count the number of files in the current directory's first level:

```bash
for dir in /*; do echo "$dir"; find "$dir" -xdev -type f 2>/dev/null | wc -l; done
```

View file counts in `/var` directories:

```bash
for dir in /var/*; do echo "$dir"; find "$dir" -xdev -type f 2>/dev/null | wc -l; done
```

### Purpose

Suitable for locating directories with the most small files when inodes are exhausted.

Note:

```text
-xdev
→ Do not cross filesystem statistics to avoid scanning to other mount points
```

---

## Scenario 16: Count Specific File Types

### Command

```bash
find /var/log -type f -name "*.gz" | wc -l
```

```bash
find /tmp -type f | wc -l
```

```bash
find /data/cache -type f | wc -l
```

### Purpose

Confirm if the number of specific file types is abnormal.

---

## VII. lsblk, fdisk, blkid: View Disk and Mount Relationships

---

## Scenario 17: View Block Device Structure

### Command

```bash
lsblk
```

### Purpose

View:

```text
Disk

Division

LVM

Mount Point
```

Common output structure:

```text
sda
├─sda1
└─sda2
  └─vg-root
```

---

## Scenario 18: View File System and UUID

### Command /think

```bash
lsblk -f
```

### Purpose

Displays:

```text
File System Type

UUID

Mount Point
```

Suitable for troubleshooting:

```text
New Disk Formatted

Whether the partition has a file system

Whether the mount point is correct

/etc/fstab Medium UUID Correct?
```

---

## Scenario 19: View Disk Partition Table

### Command

```bash
fdisk -l
```

View specified disk:

```bash
fdisk -l /dev/sdb
```

### Purpose

More deeply view disk and partition information.

Suitable for:

```text
Confirm if the new disk is recognized

Confirm partition size

Confirm partition table type

Confirm disk device name
```

---

## Scenario 20: View Device UUID and File System Type

### Command

```bash
blkid
```

View specified device:

```bash
blkid /dev/sdb1
```

### Purpose

Commonly used to confirm UUID before configuring `/etc/fstab`.

---

## VIII. iostat: Disk IO Troubleshooting Entry Point

---

## Scenario 21: View Disk IO

### Command

```bash
iostat -x 1 5
```

### Parameter Description

```text
-x
→ Extension of statistics

1 5
→ Every 1 Sampling in seconds 1 Number of times, sampled 5 Minor
```

### Purpose

Check if there is a bottleneck in disk IO.

Key fields:

```text
r/s
→ Request per second

w/s
→ Request per second

rkB/s
→ Read traffic per second

wkB/s
→ Write flow per second

await
→ Average waiting time

%util
→ Disk Utilization
```

---

## Scenario 22: Display Disk IO in MB

### Command

```bash
iostat -dxm 1 5
```

### Parameter Description

```text
-d
→ Only disk devices

-x
→ Extension of statistics

-m
→ Press MB Show
```

---

## Scenario 23: Common iostat Judgment

### %util High

```text
%util High
→ The disk's busy.
```

If long-term close to:

```text
100%
```

Indicates the disk may be approaching full load.

### await High

```text
await High
→ IO Average waiting time
→ Possible disk or storage bottlenecks
```

### r/s, w/s High

```text
r/s High
→ Read many requests

w/s High
→ Write many requests
```

### rkB/s, wkB/s High

```text
rkB/s High
→ It's high.

wkB/s High
→ Write high
```

---

## Scenario 24: What to Do if iostat is Not Installed

`iostat` usually comes from `sysstat` package.

Ubuntu / Debian:

```bash
apt install -y sysstat
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y sysstat
```

Or:

```bash
dnf install -y sysstat
```

Install and verify:

```bash
iostat -x 1 5
```

---

## IX. iotop: Locate Which Process Occupies IO

---

## Scenario 25: View Real-time IO Processes

### Command

```bash
iotop
```

### Purpose

Real-time view which process is reading/writing disk.

---

## Scenario 26: Only Display Processes with IO

### Command

```bash
iotop -o -P
```

### Parameter Description

```text
-o
→ Show only emerging IO Process

-P
→ Look at the process, don't spread the thread.
```

---

## Scenario 27: Refresh Every 1 Second

### Command

```bash
iotop -o -P -d 1
```

### Purpose

More suitable for dynamic observation of short-term IO fluctuations.

---

## Scenario 28: Cumulative IO Observation

### Command

```bash
iotop -a -o -P
```

### Parameter Description

```text
-a
→ Show cumulative IO
```

Suitable for:

```text
Observe which processes are cumulatively most written over time
```

---

## Scenario 29: What to Do if iotop is Not Installed

Ubuntu / Debian:

```bash
apt install -y iotop
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y iotop
```

Or:

```bash
dnf install -y iotop
```

If production environment does not allow installation, can prioritize use:

```bash
pidstat -d 1 5
```

---

## X. pidstat -d: View IO by Process

---

## Scenario 30: View IO by Process Dimension

### Command

```bash
pidstat -d 1 5
```

### Purpose

View disk read/write status by process dimension.

Common fields:

```text
kB_rd/s
→ Read per second KB

kB_wr/s
→ Writing every second KB

kB_ccwr/s
→ Cancel writing per second KB

Command
→ Process command
```

---

## Scenario 31: View IO of Specified Process

### Command

```bash
pidstat -d -p PID 1 5
```

Example:

```bash
pidstat -d -p 12345 1 5
```

### Purpose

Continuously observe IO behavior of a specific process.

---

## Scenario 32: What to Do if pidstat is Not Installed

`pidstat` also comes from `sysstat` package.

Ubuntu / Debian:

```bash
apt install -y sysstat
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y sysstat
```

Or:

```bash
dnf install -y sysstat
```

---

## XI. dstat: Simultaneously Monitor CPU, Disk, and Network

---

## Scenario 33: Real-time View System Comprehensive Status

### Command

```bash
dstat
```

### Common Commands

```bash
dstat -d
```

```bash
dstat -n
```

```bash
dstat -d -n
```

```bash
dstat -cdngy
```

### Parameter Description

```text
-c
→ CPU

-d
→ Disk Read and Write

-n
→ Network traffic

-g
→ pagememory page

-y
→ System statistics
```

### Purpose

Suitable for one-screen simultaneous observation of:

```text
CPU

Disk Read and Write

Network traffic

Memory Page

System Status
```

---

## Scenario 34: What to Do if dstat is Not Installed

Ubuntu / Debian:

```bash
apt install -y dstat
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y dstat
```

Or:

```bash
dnf install -y dstat
```

If no `dstat`, can separately use:

```bash
vmstat 1 5
```

```bash
iostat -x 1 5
```

```bash
sar -n DEV 1 5
```

---

## XII. dmesg: View Disk and File System Abnormalities

---

## Scenario 35: View Kernel Logs

### Command

```bash
dmesg
```

View recent logs:

```bash
dmesg | tail
```

With readable time:

```bash
dmesg -T | tail -n 50
```

### Purpose

`dmesg` is suitable for viewing kernel-level hardware, driver, disk, and file system abnormalities.

---

## Scenario 36: Filter Disk-related Errors

### Command

```bash
dmesg -T | grep -i disk
```

```bash
dmesg -T | grep -i error
```

```bash
dmesg -T | grep -i fail
```

```bash
dmesg -T | grep -i ext4
```

```bash
dmesg -T | grep -i xfs
```

```bash
dmesg -T | grep -i nvme
```

```bash
dmesg -T | grep -i scsi
```

### Purpose

Troubleshoot:

```text
Disk hardware anomaly

Filesystem Error

Driver anomaly

Anomalous piece of equipment

Disk Drop

I/O error

File system read-only
```

---

## XIII. Common Troubleshooting Scenarios

---

## Scenario 37: Disk Space Full

### Phenomenon

```text
No space left on device

Service Writing Log Failed

Database writing failed

Upload failed

df -h Medium Use% Close 100%
```

### Troubleshooting Commands

```bash
df -h
```

```bash
df -hT
```

```bash
du -h --max-depth=1 /
```

```bash
du -h --max-depth=1 /var | sort -hr | head
```

```bash
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

### Troubleshooting Approach

```text
Let's see which mount points are full first.
→ Let's see which one's big on the mounted point.
→ We'll drill down the line.
→ Found Big Files or Big Catalogues
→ Adjudication, archiving, deletion or expansion
```

---

## Scenario 38: Inode Full

### Phenomenon

```text
No space left on device

df -h It's not full.

Cannot create new file

Massive small file piles
```

### Troubleshooting Commands

```bash
df -hi
```

```bash
for dir in /*; do echo "$dir"; find "$dir" -xdev -type f 2>/dev/null | wc -l; done
```

```bash
find /tmp -type f | wc -l
```

```bash
find /var/log -type f | wc -l
```

### Troubleshooting Approach

```text
Confirm which mount point inode Full
→ Which directories have the most small files?
→ View source
→ Clear expired small files
→ Refurbish programs that generate large volumes of small files
```

---

## Scenario 39: High Disk IO

### Phenomenon

```text
System Carton.

load High

top Medium wa High

vmstat Medium wa High

iostat Medium %util High

Application response slow

Increased number of slow database queries
```

### Troubleshooting Commands

```bash
top
```

```bash
vmstat 1 5
```

```bash
iostat -x 1 5
```

```bash
iotop -o -P
```

```bash
pidstat -d 1 5
```

### Troubleshooting Approach

```text
Let's make sure. IO wait High
→ Make sure you know which plate it is. util / await High
→ And see which process is reading and writing.
→ Then we'll see if it's a log, database, backup or an anomaly.
```

---

## Scenario 40: df and du Inconsistency

### Phenomenon

```text
df -h Showing space occupied.

du -sh No large files found
```

### Common Causes

```text
Big files deleted but process still occupied

Mount Point Overwrite

Insufficient Permissions

Special File System
```

### Troubleshooting Commands

```bash
lsof | grep deleted
```

View processes:

```bash
ps -fp PID
```

Handling approach:

```text
Find occupation deleted Process of documentation
→ Confirm whether or not to restart or reload
→ Release Document Thread
```

Note:

```text
Don't be direct. kill Production core process
First confirm the extent of the impact.
```

---

## Scenario 41: Log File Too Large

### Troubleshooting Commands

```bash
du -sh /var/log/*
```

```bash
find /var/log -type f -size +100M -exec ls -lh {} \;
```

```bash
ls -lh /var/log
```

### Handling Approach

```text
Confirm Log Source

→ Whether or not to compress the archive

→ Judge whether expired logs can be cleared

→ Configure logrotate

→ Fix application log level or abnormal screenbrush problem
```

---

## Scenario 42: Docker Data Directory Too Large

### Troubleshooting Commands

```bash
du -sh /var/lib/docker
```

```bash
docker system df
```

```bash
docker system df -v
```

```bash
docker images
```

```bash
docker ps -a
```

```bash
docker volume ls
```

### Troubleshooting Approach

```text
It's a mirror, a container layer,volume Or is the log occupied?

→ Too many mirrors to clean up old ones.

→ Stop filling too many containers to clean up useless ones

→ volume Validation of data values

→ Packaging Log Configuration Rotation

→ Planning as necessary data-root To Data Disk
```

Note:

```text
Do not just delete /var/lib/docker
Don't be brainless. docker volume prune
```

---

## Scenario 43: File System Becomes Read-Only

### Phenomenon

```text
Read-only file system
```

### Troubleshooting Commands

```bash
mount | grep " ro,"
```

```bash
dmesg -T | tail -n 100
```

```bash
dmesg -T | grep -i error
```

```bash
dmesg -T | grep -i xfs
```

```bash
dmesg -T | grep -i ext4
```

### Common Causes

```text
Disk hardware anomaly

Filesystem Error

Base storage anomaly

kernel remount file system to read-only for data protection
```

### Handling Approach

```text
Protect data first

→ View kernel log

→ Confirm if hardware or file system error

→ Assessment of the need for parking checks

→ Don't be blind. remount rw
```

---

## Fourteen. Production Handling Precautions

---

## 1. Do Not Delete Files Immediately

When disk is full, first confirm:

```text
What service does the file belong to?

Whether to be written

Whether or not to retain

Is there a backup?

Archival

Impact on operations
```

Then proceed with handling.

---

## 2. Do Not Directly Clear Logs Being Written

Not recommended to directly:

```bash
rm -f /path/to/running.log
```

If processes still hold file handles, space may not be released.

A more secure approach:

```bash
truncate -s 0 /path/to/running.log
```

But in production, still need to confirm if logs can be cleared.

---

## 3. Do Not Arbitrarily Delete Volume and Database Directories

The following directories require special caution:

```text
/var/lib/mysql

/var/lib/postgresql

/data/mysql

/data/postgres

/data/mongodb

/var/lib/docker/volumes

Business Upload Directory

Backup Directory
```

Must confirm data value before handling.

---

## 4. High IO Does Not Necessarily Mean Disk Failure

High IO could be:

```text
Backup Tasks

Log Brush

Database big query

Batch compression

Large file transfer

Porter mirror pull

Normal peak operations

Disk performance bottlenecks
```

Need to combine:

```text
iostat

iotop

pidstat

Business log

Mission plan

Monitor Curve
```

For comprehensive judgment.

---

## 5. Handle File System Errors with Caution

If `dmesg` shows:

```text
I/O error

EXT4-fs error

XFS error

Buffer I/O error

read-only filesystem
```

Do not only perform cleanup.

Should further confirm:

```text
Disk Health

Bottom Storage

RAID Status

Cloud Disk Status

File System State

Whether windows need to be maintained
```

---

## Fifteen. Common Commands Summary in This Article

---

## Disk Space

```bash
df -h
```

```bash
df -hT
```

```bash
df -h /var/log
```

```bash
df -h /data
```

```bash
df -h /var/lib/docker
```

---

## Inode

```bash
df -hi
```

```bash
for dir in /*; do echo "$dir"; find "$dir" -xdev -type f 2>/dev/null | wc -l; done
```

```bash
find /tmp -type f | wc -l
```

```bash
find /var/log -type f | wc -l
```

---

## Directory Size

```bash
du -sh *
```

```bash
du -sh /var/log/*
```

```bash
du -sh /data/*
```

```bash
du -h --max-depth=1 /data
```

```bash
du -h --max-depth=1 /var | sort -hr | head -n 20
```

---

## Large File Search

```bash
find /var/log -type f -size +100M
```

```bash
find / -type f -size +1G 2>/dev/null
```

```bash
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

```bash
find /data -type f -mtime +30 -size +100M -exec ls -lh {} \;
```

---

## Disk and Partition Information

```bash
lsblk
```

```bash
lsblk -f
```

```bash
fdisk -l
```

```bash
fdisk -l /dev/sdb
```

```bash
blkid
```

```bash
blkid /dev/sdb1
```

---

## Disk IO

```bash
iostat -x 1 5
```

```bash
iostat -dxm 1 5
```

```bash
iotop
```

```bash
iotop -o -P
```

```bash
iotop -o -P -d 1
```

```bash
iotop -a -o -P
```

```bash
pidstat -d 1 5
```

```bash
pidstat -d -p PID 1 5
```

---

## Comprehensive Status

```bash
vmstat 1 5
```

```bash
dstat
```

```bash
dstat -d
```

```bash
dstat -d -n
```

```bash
dstat -cdngy
```

---

## Kernel Logs

```bash
dmesg
```

```bash
dmesg | tail
```

```bash
dmesg -T | tail -n 50
```

```bash
dmesg -T | grep -i disk
```

```bash
dmesg -T | grep -i error
```

```bash
dmesg -T | grep -i fail
```

```bash
dmesg -T | grep -i xfs
```

```bash
dmesg -T | grep -i ext4
```

```bash
dmesg -T | grep -i nvme
```

```bash
dmesg -T | grep -i scsi
```

---

## Deleted Files Occupying Space

```bash
lsof | grep deleted
```

```bash
ps -fp PID
```

---

## Tool Installation

Ubuntu/Debian install sysstat:

```bash
apt install -y sysstat
```

RHEL/CentOS/Rocky/AlmaLinux install sysstat:

```bash
yum install -y sysstat
```

Or:

```bash
dnf install -y sysstat
```

Ubuntu/Debian install iotop:

```bash
apt install -y iotop
```

RHEL/CentOS/Rocky/AlmaLinux install iotop:

```bash
yum install -y iotop
```

Or:

```bash
dnf install -y iotop
```

Ubuntu/Debian install dstat:

```bash
apt install -y dstat
```

RHEL/CentOS/Rocky/AlmaLinux install dstat:

```bash
yum install -y dstat
```

Or:

```bash
dnf install -y dstat
```

---

## Sixteen. One-Sentence Summary

Disk troubleshooting core is:

```text
It's about space.inode Question, still. IO Problem
```

Space troubleshooting chain:

```text
df -h

→ du -h --max-depth=1

→ find Big document

→ Deciding whether logs, backups, data directories,Docker Contents occupation
```

Inode troubleshooting chain:

```text
df -hi

→ find Number of statistical documents

→ Position a large number of small file directories

→ Clean out obsolete small files and repair the source Head
```

Disk IO troubleshooting chain:

```text
top / vmstat Look. wa

→ iostat -x 1 5 Watch the equipment.

→ iotop -o -P Look at the process.

→ pidstat -d Read and write at the process level.

→ Integration of business assignments, logs, databases, backup analysis
```

Production recommendations:

```text
Don't delete files as soon as you get here.
Don't be brainless. volume or database directory
Do not just delete the writing log
Don't do it manually. /var/lib/docker
IO Highlights combine operational mandate judgement
Filesystem Error Prioritize Data Protection
```