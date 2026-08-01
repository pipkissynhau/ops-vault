# 04 - Disk Partitioning, Mounting, and LVM Expansion

#Linux #Transport #TheBarrier. #DiskManagement #Division #Mount #LVM #fdisk #parted #mkfs #fstab #pvcreate #vgextend #lvextend

---

## Recommended Path

01-Linux Basics and Host Maintenance/01-Host Troubleshooting/04-Disk Partitioning, Mounting, and LVM Expansion.md

---

## Section 1: Document Explanation

This document organizes information about **disk partitioning, formatting, mounting, automatic mounting at boot, disk rescan, LVM management, and expansion** in Linux hosts.

Key focus areas include:

- Checking disks and partitions
- Checking file systems and UUID
- `lsblk`
- `fdisk -l`
- `blkid`
- `fdisk` partitioning
- `parted` managing GPT partitions
- `mkfs.xfs`
- `mkfs.ext4`
- Temporary mounting
- `/etc/fstab` automatic mounting at boot
- `mount -a` verifying mounting configuration
- Unmounting file systems
- Checking processes using mount points
- Rescanning disks
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
- Full LVM expansion process
- LVM shrinkage risk explanation

Goals:

- Identify newly added disks
- Complete partitioning, formatting, and mounting
- Configure `/etc/fstab` automatic mounting at boot
- Troubleshoot unmount failure and partition table refresh failure
- Understand PV / VG / LV relationships
- Complete common LVM online expansion
- Distinguish XFS and ext4 file system expansion methods

---

## Section 2: Disk Management Overview

After adding a new disk, the common workflow is:

```text
Recognize Disk

→ Division

→ Format File System

→ Create Mount directory

→ Temporary Mount

→ Authenticate reading and writing

→ Writing /etc/fstab

→ mount -a Validate starter automount

→ df -h / lsblk -f Authentication Results
```

For LVM expansion, the common workflow is:

```text
Recognize New Disk

→ Division

→ Create PV

→ Add VG

→ Expansion LV

→ Expanded Document System

→ Authentication df / pvs / vgs / lvs
```

One-sentence summary:

```text
Mount Normal Disk
→ Division, Formatting, Mounting

LVM Expansion
→ PVI don't know.VGI don't know.LVDocument system expansion
```

---

## Section 3: Basic Information on Disks and Partitions

---

## Scenario 1: Checking Block Device Structure

### Command

```bash
lsblk
```

### Purpose

View:

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

### Notes

`lsblk` is the first entry for troubleshooting newly added disks, partitions, and LVM.

Focus on:

```text
NAME
SIZE
TYPE
MOUNTPOINT
```

---

## Scenario 2: Checking File Systems and UUID

### Command

```bash
lsblk -f
```

### Purpose

On top of `lsblk`, it displays:

- File system type
- UUID
- LABEL
- Mount point

Focus on:

```text
FSTYPE
UUID
MOUNTPOINT
```

Suitable for determining:

```text
New disk formatted

Whether the partition has created a file system

/etc/fstab Medium UUID Is it right?

Whether mount points are expected
```

---

## Scenario 3: Checking Disk Partition Table

### Command

```bash
fdisk -l
```

Check a specific disk:

```bash
fdisk -l /dev/sdb
```

### Purpose

View moreBottom disk and partition information.

Commonly used for:

```text
Confirm if the new disk is recognized

Confirm disk size

Number of confirmed divisions

Confirm Division Type

Can not open message
```

---

## Scenario 4: Checking Device UUID and File System Type

### Command

```bash
blkid
```

Check a specific device:

```bash
blkid /dev/sdb1
```

### Purpose

View:

```text
UUID
TYPE
PARTUUID
```

Commonly used to confirm UUID before writing `/etc/fstab`.

---

## Section 4: fdisk Partitioning: MBR Scenario

---

## Scenario 5: Entering fdisk Partition Tool

### Command

```bash
fdisk /dev/sdb
```

### Notes

`/dev/sdb` needs to be replaced with the actual disk.

Confirm the target disk before execution to avoid accidental operations on production data disks.

Confirm disk:

```bash
lsblk
```

```bash
fdisk -l
```

---

## Scenario 6: Common fdisk Interactive Commands

After entering `fdisk /dev/sdb`, common interactive commands:

```text
m
→ View Help

p
→ View the current partition table

n
→ New Partition

d
→ Remove Partition

t
→ Modify partition type

w
→ Save and exit

q
→ Do Not Save Exit
```

---

## Scenario 7: Common fdisk Partitioning Process

### 1. Select Target Disk

```bash
fdisk /dev/sdb
```

### 2. View Current Partitions

```text
p
```

### 3. Create New Partition

```text
n
```

### 4. Select Partition Type, Partition Number, Start Sector, End Sector

Usually select based on actual needs.

If creating a single partition for the entire disk, many scenarios directly use default start/end positions.

### 5. Save and Exit

```text
w
```

### 6. Let Kernel Rescan Partition Table

```bash
partprobe /dev/sdb
```

### 7. Verify Partition

```bash
lsblk
```

```bash
fdisk -l /dev/sdb
```

---

## Scenario 8: Setting LVM Partition Type in fdisk

If the partition is to be used for LVM, modify the partition type in `fdisk`.

Enter fdisk:

```bash
fdisk /dev/sdb
```

Modify partition type:

```text
t
```

Common Linux LVM type in MBR:

```text
8e
```

Save:

```text
w
```

Refresh partition table:

```bash
partprobe /dev/sdb
```

Notes:

```text
8e Yes. MBR Common under Partition Table Linux LVM Type
GPT There's a different way to manage the partition type in the scene.
```

---

## Section 5: parted Managing GPT Partitions

---

## Scenario 9: Why Use parted

For large-capacity disks, especially those over 2TB, GPT partition tables are more commonly used.

`parted` is more suitable for managing GPT partitions.

Common scenarios:

```text
Large Capacity Data Disk

Rolling clouds.

Over 2TB Disk

Yes. GPT The scene of the partition table
```

---

## Scenario 10: Entering parted

### Command

```bash
parted /dev/sdb
```

### Common Interactive Commands

```text
print
→ View partition table

mklabel gpt
→ Create GPT Partition Table

mkpart
→ Create Partition

quit
→ Exit
```

---

## Scenario 11: Non-Interactive GPT Partition Creation with parted

### Create GPT Partition Table

```bash
parted /dev/sdb --script mklabel gpt
```

### Create a Primary Partition Filling the Entire Disk

```bash
parted /dev/sdb --script mkpart primary xfs 0% 100%
```

### Rescan Partition Table

```bash
partprobe /dev/sdb
```

### Verification

```bash
lsblk
```

```bash
parted /dev/sdb print
```

---

## Section 6: Formatting File Systems

---

## Scenario 12: Formatting as XFS

### Command

```bash
mkfs.xfs /dev/sdb1
```

### Notes

XFS is a common file system in many production Linux systems.

Suitable for:

```text
Big document
Large Volume Disk
Data Disk
Logboard
Database data disc scene
```

Note:

```text
mkfs We'll clear the original data from the target partition.
The equipment must be verified before execution.
```

---

## Scenario 13: Formatting as ext4

### Command

```bash
mkfs.ext4 /dev/sdb1
```

### Notes

ext4 is also a common Linux file system.

Suitable for:

```text
General file system
Normal Data Disk
It's a high compatibility requirement.
```

---

## Scenario 14: Confirming Device Before Formatting

Before formatting, confirm:

```bash
lsblk
```

```bash
blkid /dev/sdb1
```

```bash
fdisk -l /dev/sdb
```

Production note:

```text
Do not execute without confirming the use of the equipment mkfs
Do not format existing data partitions
Do not misformulate the partition of the system as a new disk
```

---

## Section 7: Mounting File Systems

---

## Scene 15: Creating a Mount Point

### Command

```bash
mkdir -p /data
```

### Explanation

A mount point is essentially a directory.

Common mount points:

```text
/data
/data1
/backup
/logs
/appdata
```

---

## Scene 16: Temporary Mounting

### Command

```bash
mount /dev/sdb1 /data
```

### Verification

```bash
df -h
```

```bash
df -h /data
```

```bash
mount | grep sdb1
```

```bash
lsblk -f
```

---

## Scene 17: Testing Read/Write

### Command

```bash
touch /data/testfile
```

```bash
ls -lh /data/testfile
```

```bash
rm -f /data/testfile
```

### Purpose

Confirm that the mount point can be written to normally.

---

## VIII. Configuring Automatic Mounting at Boot: /etc/fstab

---

## Scene 18: Why to Write fstab

Manual execution:

```bash
mount /dev/sdb1 /data
```

This is a temporary mount.

The system will not automatically mount after reboot.

To achieve automatic mounting at boot, you need to write:

```bash
/etc/fstab
```

---

## Scene 19: Viewing UUID

### Command

```bash
blkid /dev/sdb1
```

Example output:

```text
/dev/sdb1: UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="xfs"
```

It is recommended to prioritize using UUID in `/etc/fstab` rather than directly writing `/dev/sdb1`.

Reason:

```text
Possible change in equipment name
UUID More stable.
```

---

## Scene 20: Editing /etc/fstab

### Command

```bash
vi /etc/fstab
```

### XFS Example

```fstab
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /data xfs defaults 0 0
```

### ext4 Example

```fstab
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /data ext4 defaults 0 0
```

Field explanations:

```text
I don't think so. 1 Columns
→ Equipment or UUID

I don't think so. 2 Columns
→ Mount Point

I don't think so. 3 Columns
→ File System Type

I don't think so. 4 Columns
→ Mount Parameters

I don't think so. 5 Columns
→ dump Backup tags, usually 0

I don't think so. 6 Columns
→ fsck Check order, disks usually 0 or 2, needs to be combined with practical strategies
```

---

## Scene 21: Verifying fstab Configuration

### Command

```bash
mount -a
```

### Purpose

Verify whether the mount configuration in `/etc/fstab` is correct without rebooting the system.

If the configuration is incorrect, it may report errors.

Verify the mount result:

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

## Scene 22: Risks of fstab Configuration Errors

`/etc/fstab` configuration errors may lead to:

```text
Get in. emergency mode

The system is slowing down.

Mount Failed

Business Data Directory Not Mounted

Apply empty directories written to unmounted

Data path disorder
```

Production recommendations:

```text
Modify fstab Backup First

Must be implemented after modification mount -a Authentication

Consider configuration complete after validation
```

Backup:

```bash
cp -a /etc/fstab /etc/fstab.$(date +%F-%H%M%S).bak
```

---

## IX. Unmounting File Systems

---

## Scene 23: Unmounting a Mount Point

### Command

```bash
umount /data
```

You can also unmount the device:

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

## Scene 24: Unmount Failed: target is busy

If an error occurs during unmounting:

```text
target is busy
```

It indicates that a process is using the mount point.

Check the usage:

```bash
lsof +D /data
```

Or:

```bash
fuser -vm /data
```

Explanation:

```text
lsof +D /data
→ View Access /data Process of documents under directory

fuser -vm /data
→ View processes that occupy the mount
```

Before handling, confirm:

```text
Can these processes stop?

Production services

Whether to write data

Whether to arrange maintenance windows
```

---

## X. Rescanning Disks and Refreshing Partition Table

---

## Scene 25: System Not Recognizing New Disk After Addition

In some environments, after adding a cloud disk or virtual disk, the system may not recognize it immediately.

You can first check:

```bash
lsblk
```

```bash
fdisk -l
```

If the new disk is not visible, you can try scanning the SCSI host.

---

## Scene 26: SCSI Bus Scanning

### Command

```bash
echo "- - -" > /sys/class/scsi_host/host0/scan
```

If there are multiple hosts, you can scan them separately:

```bash
ls /sys/class/scsi_host/
```

Example:

```bash
echo "- - -" > /sys/class/scsi_host/host0/scan
```

```bash
echo "- - -" > /sys/class/scsi_host/host1/scan
```

```bash
echo "- - -" > /sys/class/scsi_host/host2/scan
```

Check again:

```bash
lsblk
```

```bash
fdisk -l
```

---

## Scene 27: Re-reading Partition Table

### Command

```bash
partprobe /dev/sdb
```

Or:

```bash
partx -u /dev/sdb
```

### Applicable Scenarios

```text
System not recognized immediately after new partition

Additional kernel unupdated partition table

fdisk/parted Device node not updated after operation
```

---

## Scene 28: Partition Table Refresh Fails

First check if the disk is in use:

```bash
lsof /dev/sdb
```

```bash
fuser -v /dev/sdb
```

If the device is in use, you need to unmount the relevant mount point or stop the using process first, then execute:

```bash
partprobe /dev/sdb
```

You can also try:

```bash
partx -u /dev/sdb
```

If it still doesn't work, you may need:

```text
Re-scan equipment

Or restart the system in the maintenance window to re-identify the partition table for the kernel
```

---

## XI. Basic Understanding of LVM

---

## Scene 29: LVM Three-Layer Structure

Common three-layer structure of LVM:

```text
PV
→ Physical VolumePhysical rolls.

VG
→ Volume GroupVolume

LV
→ Logical VolumeLogical roll
```

The relationship can be understood as:

```text
Disk / Division
→ Done. PV

Multiple PV
→ Add VG

From VG Split Space between
→ Create or expand LV

LV Create Filesystem Up
→ Mount to directory use
```

---

## Scene 30: Why LVM is Suitable for Production Expansion

Advantages of LVM:

```text
You can synthesize multiple disk spaces into a roll. Group

It's flexible enough to expand the flow of logic.

An online extended common file system

More flexible than traditional fixed divisions

Fits to data disk and business directory growth scenario
```

Common uses:

```text
/data

/var/lib/mysql

/var/lib/docker

Log Directory

Backup Directory

Directory of business data
```

---

## XII. Viewing LVM Structure

---

## Scene 31: Viewing Physical Volumes (PV)

### Command

```bash
pvs
```

Detailed view:

```bash
pvdisplay
```

### Purpose

View the status, size, and associated VG of the physical volume.

---

## Scene 32: Viewing Volume Groups (VG)

### Command

```bash
vgs
```

Detailed view:

§

### Purpose

Focus on:

```text
VG Size
Free
```

Used to confirm how much remaining space is available in the volume group.

---

## Scene 33: Viewing Logical Volumes (LV)

### Command

```bash
lvs
```

Detailed view:

```bash
lvdisplay
```

### Purpose

View the size, path, and associated VG of the logical volume.

Common LV paths:

```text
/dev/vgdata/lvdata
/dev/mapper/vgdata-lvdata
```

---

## XIII. Adding New Disk to LVM

---

## Scene 34: Confirming New Disk

### Command

```bash
lsblk
```

```bash
fdisk -l
```

Assume the new disk is:

```text
/dev/sdb
```

---

## Scene 35: Partitioning New Disk

### Command

```bash
fdisk /dev/sdb
```

Common interaction:

```text
n
→ New Partition

t
→ Change Type

8e
→ MBR Down Linux LVM Type

w
→ Save
```

Refresh partition table:

```bash
partprobe /dev/sdb
```

Verification:

```bash
lsblk
```

Assume the new partition is:

```text
/dev/sdb1
```

---

## Scene 36: Creating Physical Volume (PV)

### Command

```bash
pvcreate /dev/sdb1
```

Verification:

```bash
pvs
```

```bash
pvdisplay
```

---

## Scene 37: Expanding Volume Group (VG)

### Command

```bash
vgextend vgdata /dev/sdb1
```

Explanation:

```text
vgdata
→ Replace with the actual volume group name
```

Verification:

```bash
vgs
```

```bash
vgdisplay vgdata
```

---

## XIV. Expanding Logical Volume (LV)

---

## Scene 38: Expanding with Specified Capacity

### Command

```bash
lvextend -L +100G /dev/vgdata/lvdata
```

Meaning:

```text
-L +100G
→ Existing LV Increase 100G
```

---

## Scene 39: Expanding Using All Remaining VG Space

### Command

```bash
lvextend -l +100%FREE /dev/vgdata/lvdata
```

Meaning:

```text
-l +100%FREE
→ Use all remaining space in the volume group
```

---

## Scene 40: lvextend Parameter Explanation

```text
-L
→ Increased by volume

-l
→ Press extent Expansion

+100G
→ Increase 100G

+100%FREE
→ Use a scroll group of all free spaces
```

Note:

```text
lvextend It's just an amplifier. / Logical volume
The file system also needs to be expanded.
```

---

## XV. Expanding File System

---

## Scene 41: Confirming File System Type

### Command

```bash
df -hT /data
```

Or:

```bash
lsblk -f
```

You need to first confirm:

```text
XFS
Still? ext4
```

Different file systems have different expansion commands.

## Scenario 42: XFS File System Expansion

### Commands

```bash
xfs_growfs /data
```

Explanation:

```text
XFS The amplification usually executes the mount point
Not directly to a piece of equipment.
```

Verification:

```bash
df -h /data
```

---

## Scenario 43: ext4 File System Expansion

### Commands

```bash
resize2fs /dev/vgdata/lvdata
```

Explanation:

```text
ext4 Regular expansion of logical volume equipment
```

Verification:

```bash
df -h
```

---

## Scenario 44: lvextend -r Automatic File System Expansion

Some environments can use:

```bash
lvextend -r -L +100G /dev/vgdata/lvdata
```

Or:

```bash
lvextend -r -l +100%FREE /dev/vgdata/lvdata
```

Explanation:

```text
-r
→ Expansion LV Auto-expand file system after
```

Production Recommendations:

```text
Although... -r Easy, but file system type, backup and rollback should be confirmed before formal production operations
```

---

## Sixteen. Complete LVM Expansion Process

---

## Scenario 45: Existing VG Has Free Space

### 1. Check LVM Structure

```bash
pvs
```

```bash
vgs
```

```bash
lvs
```

### 2. Check Mount and File System

```bash
df -hT /data
```

```bash
lsblk -f
```

### 3. Expand Logical Volume

Specify adding 100G:

```bash
lvextend -L +100G /dev/vgdata/lvdata
```

Or use all remaining space:

```bash
lvextend -l +100%FREE /dev/vgdata/lvdata
```

### 4. Expand File System

XFS:

```bash
xfs_growfs /data
```

ext4:

```bash
resize2fs /dev/vgdata/lvdata
```

### 5. Verify Results

```bash
df -h
```

```bash
lvs
```

```bash
vgs
```

---

## Scenario 46: Expand LVM After Adding New Disk

### 1. Confirm New Disk

```bash
lsblk
```

```bash
fdisk -l
```

### 2. Partition

```bash
fdisk /dev/sdb
```

### 3. Refresh Partition Table

```bash
partprobe /dev/sdb
```

### 4. Create PV

```bash
pvcreate /dev/sdb1
```

### 5. Add to Volume Group

```bash
vgextend vgdata /dev/sdb1
```

### 6. Check VG Free Space

```bash
vgs
```

```bash
vgdisplay vgdata
```

### 7. Expand Logical Volume

```bash
lvextend -l +100%FREE /dev/vgdata/lvdata
```

### 8. Expand File System

XFS:

```bash
xfs_growfs /data
```

ext4:

```bash
resize2fs /dev/vgdata/lvdata
```

### 9. Verify

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

## Seventeen. LVM Shrinking Notes

In production environments, shrinking carries much higher risks than expanding.

Reason:

```text
The abbreviation requires the file system first, then the file system. LV
The error in the order of operation may directly damage the data
XFS Do not support a condensation
ext4 You need to be very careful.
Usually needs to unmount the file system
Need to do a file system check first
Full backup required
```

Production Recommendations:

```text
General transport for production is mainly capacity-building
It's easy to do online.
If a convulsion is required, it should be fully backed up and tested in the environment
```

---

## Eighteen. Common Issues and Troubleshooting

---

## Problem 1: New Disk Not Detected

### Troubleshooting Commands

```bash
lsblk
```

```bash
fdisk -l
```

```bash
dmesg -T | tail -n 50
```

Scan SCSI host:

```bash
echo "- - -" > /sys/class/scsi_host/host0/scan
```

Check host:

```bash
ls /sys/class/scsi_host/
```

If multiple hosts, scan each separately.

---

## Problem 2: /dev/sdb1 Not Appearing After Partitioning

### Troubleshooting Commands

```bash
lsblk
```

```bash
fdisk -l /dev/sdb
```

Refresh partition table:

```bash
partprobe /dev/sdb
```

Or:

```bash
partx -u /dev/sdb
```

Check if occupied:

```bash
lsof /dev/sdb
```

```bash
fuser -v /dev/sdb
```

---

## Problem 3: Mount Failed

### Troubleshooting Commands

```bash
lsblk -f
```

```bash
blkid /dev/sdb1
```

```bash
dmesg -T | tail -n 50
```

```bash
mount /dev/sdb1 /data
```

Common Causes:

```text
Partition not formatted

File system type does not match

Mount directory does not exist

Device Name Error

File system damage

Competence or SELinux Limits
```

---

## Problem 4: mount -a Error

### Troubleshooting Commands

```bash
cat /etc/fstab
```

```bash
blkid
```

```bash
mount -a
```

Common Causes:

```text
UUID Wrong

Mount point does not exist

Filesystem Type Error

Parameter Error

Device not recognized
```

Recommendations:

```text
Do not restart authentication fstab
It should be used first. mount -a Authentication
```

---

## Problem 5: umount Failed, Busy Prompt

### Troubleshooting Commands

```bash
lsof +D /data
```

```bash
fuser -vm /data
```

Troubleshooting Approach:

```text
Confirm occupation process

→ Discontinuation of service

→ Toggle Current shell Path, do not stop in /data Down

→ Maintain window processes as necessary
```

Note:

```text
Do not force the killing process without identifying operational impact
```

---

## Problem 6: df -h Not Changing After lvextend

Cause:

```text
It only expanded. LV
No expanded file system
```

Confirm:

```bash
lvs
```

```bash
df -hT /data
```

If XFS:

```bash
xfs_growfs /data
```

If ext4:

```bash
resize2fs /dev/vgdata/lvdata
```

---

## Problem 7: xfs_growfs Failed

Common Causes:

```text
Wrong object
Cannot initialise Evolution's mail component.
Not a filesystem. XFS
Filesystem Not Mounted
```

Correct Example:

```bash
xfs_growfs /data
```

First Confirm:

```bash
df -hT /data
```

---

## Problem 8: resize2fs Failed

Common Causes:

```text
Not a filesystem. ext4
Device Path Error
LV No successful expansion.
File system is abnormal.
```

First Confirm:

```bash
df -hT
```

```bash
lsblk -f
```

```bash
lvs
```

---

## Nineteen. Production Operation Notes

---

## 1. Confirm Devices Before Partitioning, Formatting, and LVM Operations

Dangerous commands include:

```bash
mkfs.xfs /dev/sdb1
```

```bash
mkfs.ext4 /dev/sdb1
```

```bash
pvcreate /dev/sdb1
```

These commands may destroy existing data.

Before execution, must confirm:

```bash
lsblk
```

```bash
fdisk -l
```

```bash
blkid
```

---

## 2. Backup Before Modifying fstab

Backup:

```bash
cp -a /etc/fstab /etc/fstab.$(date +%F-%H%M%S).bak
```

Verify after modification:

```bash
mount -a
```

Do not directly reboot to verify.

---

## 3. Confirm File System Type Before Expansion

```bash
df -hT /data
```

```bash
lsblk -f
```

Determine:

```text
XFS
→ xfs_growfs /data

ext4
→ resize2fs /dev/vgdata/lvdata
```

---

## 4. LVM Expansion is Generally Safer Than Shrinking

Expansion usually has lower risks, but still need:

```text
Confirm target. LV
Confirm target mount point
Confirm file system type
Confirm. VG Remaining space
Recognition of operational impact
Conditional backup of key data
```

Shrinking has high risks and is not recommended.

---

## 5. Do Not Delete Directories Without Knowing Mount Relationships

For example `/data` if it's a mount point:

```text
Delete /data File
→ The data from the mounted disk was actually deleted.
```

If `/data` is not mounted:

```text
Application may be written in root partition /data Empty directory
→ It's causing the system to be full.
```

Troubleshoot:

```bash
df -h /data
```

```bash
mount | grep /data
```

```bash
lsblk -f
```

---

## Twenty. Common Commands Summary in This Article

---

## Check Disks and Partitions

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

## fdisk Partitioning

```bash
fdisk /dev/sdb
```

fdisk Interactive Commands:

```text
m
p
n
d
t
w
q
```

Refresh Partition Table:

```bash
partprobe /dev/sdb
```

---

## parted GPT Partitioning

```bash
parted /dev/sdb
```

```bash
parted /dev/sdb --script mklabel gpt
```

```bash
parted /dev/sdb --script mkpart primary xfs 0% 100%
```

```bash
partprobe /dev/sdb
```

---

## Formatting

```bash
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

View UUID:

```bash
blkid /dev/sdb1
```

Edit:

```bash
vi /etc/fstab
```

XFS example:

```fstab
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /data xfs defaults 0 0
```

ext4 example:

```fstab
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /data ext4 defaults 0 0
```

Verify:

```bash
mount -a
```

---

## Usage Investigation

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

## Rescan and Refresh

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

## LVM Inspection

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

Create PV:

```bash
pvcreate /dev/sdb1
```

Expand VG:

```bash
vgextend vgdata /dev/sdb1
```

Expand LV:

```bash
lvextend -L +100G /dev/vgdata/lvdata
```

Use all remaining space:

```bash
lvextend -l +100%FREE /dev/vgdata/lvdata
```

Automatically expand filesystem:

```bash
lvextend -r -L +100G /dev/vgdata/lvdata
```

```bash
lvextend -r -l +100%FREE /dev/vgdata/lvdata
```

XFS expansion:

```bash
xfs_growfs /data
```

ext4 expansion:

```bash
resize2fs /dev/vgdata/lvdata
```

Verify:

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

## Twenty-one. One-Sentence Summary

The core chain of disk partitioning, mounting, and LVM expansion is:

```text
General disks use links:
Recognize Disk
→ Division
→ Formatting
→ Mount
→ Writing fstab
→ mount -a Authentication
```

LVM expansion chain:

```text
Recognize New Disk
→ Division
→ pvcreate
→ vgextend
→ lvextend
→ xfs_growfs / resize2fs
→ df / pvs / vgs / lvs Authentication
```

Differences between XFS and ext4 expansion:

```text
XFS
→ xfs_growfs /Mount Point

ext4
→ resize2fs /dev/Logical scroll path
```

Production recommendations:

```text
The disk must be confirmed before partitioning and formatting
Modify fstab Backup must be provided before
Modify fstab Afterward. mount -a Authentication
The file system type must be confirmed before expansion
LVM Don't forget to expand the file system.
XFS Do not support a condensation
The production environment generally recommends only expansion, not easy contraction
```