# 04-Disk Partitioning, Mounting, and LVM Expansion

# Linux # Operations and Maintenance # Troubleshooting # Disk Management # Partitioning # Mounting # LVM # fdisk # parted # mkfs # fstab # pvcreate # vgextend # lvextend

---

## Recommended Path

01-Linux Basics and Server Operations and Maintenance/01-Server Troubleshooting/04-Disk Partitioning, Mounting, and LVM Expansion.md

---

## I. Document Description

This document compiles information on **disk partitioning, formatting, mounting, automatic startup mounting, disk re-scanning, LVM management, and expansion** in Linux servers.

Key topics include:

- Viewing disks and partitions
- Checking file systems and UUIDs
- `lsblk`
- `fdisk -l`
- `blkid`
- Partitioning with `fdisk`
- Managing GPT partitions with `parted`
- `mkfs.xfs`
- `mkfs.ext4`
- Temporary mounting
- Automatic startup mounting in `/etc/fstab`
- Verifying mounting configurations with `mount -a`
- Unmounting file systems
- Checking processes using mount points
- Re-scanning disks
- `partprobe`
- `partx`
- LVM basic structure
- `pvs`
- `vgs`
- `lvs`
- `pvcreate`
- `vgextend`
- `lvextend`
- `xfs_growfs`
- `resize2fs`
- Complete process of LVM expansion
- Risks associated with LVM reduction

Objectives:

- Be able to identify new disks.
- Perform partitioning, formatting, and mounting.
- Configure automatic startup mounting in `/etc/fstab`.
- Troubleshoot failures in unmounting and partition table refreshing.
- Understand the relationships between PVs, VGs, and LVs.
- Complete common LVM online expansions.
- Distinguish between XFS and ext4 file system expansion methods.

---

## II. General Disk Management Process

After adding a new disk, the typical steps are:

```text
Identify the disk.
→ Partition it.
→ Format its file system.
→ Create a mount directory.
→ Temporarily mount it.
→ Verify read and write operations.
→ Write the configuration to `/etc/fstab`.
→ Use `mount -a` to verify automatic startup mounting.
→ Check results with `df -h` and `lsblk -f`.
```

For LVM expansion, the typical steps are:

```text
Identify the new disk.
→ Partition it.
→ Create PVs.
→ Add them to a VG.
→ Expand LVs.
→ Expand the file system.
→ Verify configurations with `df`, `pvs`, `vgs`, and `lvs`.
```

In summary:

```text
For regular disk mounting:
→ Partition, format, and mount.
For LVM expansion:
→ Create PVs, VGs, LVs, and expand the file system.
```

---

## III. Viewing Basic Disk and Partition Information

---

## Scenario 1: Viewing Block Device Structure

### Command

```bash
lsblk
```

### Function

It displays information about:

- Disks
- Partitions
- LVM logical volumes
- Mount points

Common output structure:

```text
sda
├─sda1
└─sda2
  └─ubuntu--vg-ubuntu--lv
sdb
└─sdb1
```

### Note

`lsblk` is the primary tool for identifying new disks, partitions, and LVM configurations.

Focus on:

```text
NAME
SIZE
TYPE
MOUNTPOINT
```

---

## Scenario 2: Viewing File Systems and UUIDs

### Command

```bash
lsblk -f
```

### Function

It adds details such as:

- File system type
- UUID
- LABEL
- Mount point

Useful for:

```text
Checking if a new disk is already formatted.
Verifying if a partition has a file system created.
Ensuring the UUID in `/etc/fstab` is correct.
Confirming that the mount point matches expectations.
```

---

## Scenario 3: Viewing Disk Partition Tables

### Command

```bash
fdisk -l
```

To view specific disks:

```bash
fdisk -l /dev/sdb
```

### Function

It provides deeper information about disks and partitions.

Common uses include:

```text
Verifying if a new disk is recognized.
Checking the disk size.
Determining the number of partitions.
Identifying partition types.
Reviewing partition table details.
```

---

## Scenario 4: Viewing Device UUIDs and File System Types

### Command

```bash
blkid
```

To view specific devices:

```bash
blkid /dev/sdb1
```

### Function

It displays the following information for block devices:

```
/dev/sdb1: UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="xfs"
It is recommended to prefer using the UUID when writing to `/etc/fstab` rather than directly specifying `/dev/sdb1`.
```

Reasons:

```text
Device names may change.
UUIDs are more stable.
```

---

## Scenario 20: Editing /etc/fstab

### Command

```bash
vi /etc/fstab
```

### Example for XFS

```fstab
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /data xfs defaults 0 0
```

### Example for ext4

```fstab
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /data ext4 defaults 0 0
```

Field explanations:

```text
Column 1
→ Device or UUID

Column 2
→ Mount point

Column 3
→ File system type

Column 4
→ Mount parameters

Column 5
→ Dump backup flag, usually 0

Column 6
→ fsck check order; for data disks, it is usually 0 or 2. This depends on actual strategies.
```

---

## Scenario 21: Verifying fstab Configuration

### Command

```bash
mount -a
```

### Purpose

This command verifies whether the mount configurations in `/etc/fstab` are correct without restarting the system.

If there are configuration errors, error messages will be displayed.

To check the mounting results:

```bash
df -h
```

```bash
df -h /data
```

```bash
lsblk -f
```

---

## Scenario 22: Risks of Incorrect fstab Configurations

Incorrect `/etc/fstab` configurations may lead to:

```text
Entering emergency mode during startup
Slower system startup
Mounting failures
Failure to mount essential business data directories
Applications writing data to empty, unmounted directories
Data path discrepancies
```

Production recommendations:

```text
Back up the existing `/etc/fstab` configuration before making any changes.
After modifications, always execute `mount -a` to verify the settings.
Only consider the configuration complete after successful verification.
```

For backup:

```bash
cp -a /etc/fstab /etc/fstab.$(date +%F-%H%M%S).bak
```

---

## IX. Uninstalling File Systems

---

## Scenario 23: Unmounting a Mount Point

### Command

```bash
umount /data
```

You can also uninstall the device directly:

```bash
umount /dev/sdb1
```

### Verification

```bash
df -h
```

```bash
mount | grep /data
```

```bash
lsblk
```

---

## Scenario 24: Uninstallation Failure: "Target is busy"

If you encounter an error message like "target is busy" during uninstallation, it indicates that a process is currently using the mount point.

To identify the occupying process:

```bash
lsof +D /data
```

Or:

```bash
fuser -vm /data
```

Notes:

```text
lsof +D /data
→ Displays processes accessing files in the /data directory

fuser -vm /data
→ Lists processes using the mount point
```

Before taking action, confirm:

```text
Whether these processes can be stopped.
Whether they are related to critical production services.
Whether they are currently writing data.
Whether it is possible to schedule a maintenance window to perform the uninstallation.
```

---

## X. Rescanning Disks and Refreshing Partition Tables

---

## Scenario 25: System Fails to Recognize Newly Added Disks

In some environments, newly added cloud disks or virtual disks may not be immediately recognized by the system.

You can first check:

```bash
lsblk
```

```bash
fdisk -l
```

If the new disk is not listed, you can try scanning the SCSI host.

---

## Scenario 26: Scanning SCSI Buses

### Command

```bash
echo "- - -" > /sys/class/scsi_host/host0/scan
```

If there are multiple SCSI hosts, you can scan them individually:

```bash
ls /sys/class/scsi_host/
```

Examples:

```bash
echo "- - -" > /sys/class/scsi_host/host0/scan
```

```bash
echo "- - -" > /sys/class/scsi_host/host1/scan
```

```bash
echo "- - -" > /sys/class/scsi_host/host2/scan
```

After scanning, check again:

```bash
lsblk
```

```bash
fdisk -l
```

---

## Scenario 27: Rereading Partition Tables

### Command

```bash
partprobe /dev/sdb
```

Or:

```bash
ext4 often involves scaling out logical volume devices.
mkfs.xfs /dev/sdb1
```

```bash
mkfs.ext4 /dev/sdb1
```

---

## Mounting and Unmounting

```bash
mkdir -p /data
```

```bash
mount /dev/sdb1 /data
```

```bash
df -h /data
```

```bash
mount | grep sdb1
```

```bash
umount /data
```

```bash
umount /dev/sdb1
```

---

## fstab

To view the UUID:

```bash
blkid /dev/sdb1
```

To edit it:

```bash
vi /etc/fstab
```

Example for XFS:

```fstab
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /data xfs defaults 0 0
```

Example for ext4:

```fstab
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /data ext4 defaults 0 0
```

To verify the settings:

```bash
mount -a
```

---

## Checking Disk Usage

```bash
lsof +D /data
```

```bash
fuser -vm /data
```

```bash
lsof /dev/sdb
```

```bash
fuser -v /dev/sdb
```

---

## Rescanning and Refreshing

```bash
ls /sys/class/scsi_host/
```

```bash
echo "- - -" > /sys/class/scsi_host/host0/scan
```

```bash
partprobe /dev/sdb
```

```bash
partx -u /dev/sdb
```

---

## Checking LVM Information

```bash
pvs
```

```bash
pvdisplay
```

```bash
vgs
```

```bash
vgdisplay
```

```bash
vgdisplay vgdata
```

```bash
lvs
```

```bash
lvdisplay
```

---

## LVM Expansion

To create a new PV:

```bash
pvcreate /dev/sdb1
```

To expand a VG:

```bash
vgextend vgdata /dev/sdb1
```

To expand an LV:

```bash
lvextend -L +100G /dev/vgdata/lvdata
```

To use all available space:

```bash
lvextend -l +100%FREE /dev/vgdata/lvdata
```

To automatically expand the file system:

```bash
lvextend -r -L +100G /dev/vgdata/lvdata
```

```bash
lvextend -r -l +100%FREE /dev/vgdata/lvdata
```

To expand an XFS file system:

```bash
xfs_growfs /data
```

To expand an ext4 file system:

```bash
resize2fs /dev/vgdata/lvdata
```

To verify the changes:

```bash
df -h
```

```bash
pvs
```

```bash
vgs
```

```bash
lvs
```

---

## Summary

The core steps for handling regular disk partitions, mounting them, and expanding LVM volumes are as follows:

For regular disks:
- Identify the disk.
- Partition it.
- Format it.
- Mount it.
- Add it to the fstab.
- Verify by running `mount -a`.

For LVM volumes:
- Identify the new disk.
- Partition it.
- Create a PV.
- Expand the VG.
- Expand the LV.
- Expand the file system using appropriate commands (e.g., `xfs_growfs` or `resize2fs`).
- Verify the changes by checking `/df`, `pvs`, `vgs`, and `lvs`.

Note that XFS does not support volume reduction, while ext4 does. In production environments, it is generally recommended to expand volumes rather than reduce them. Always ensure to back up your configuration files before making any modifications and verify the results after applying changes.