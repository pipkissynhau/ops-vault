# 07-KubeVirt Storage Basics: PVC, DataVolume, Longhorn, and Virtual Machine Disks

Recommended Path:

    04-Kubernetes/12-KubeVirt/07-KubeVirt Storage Basics: PVC, DataVolume, Longhorn, and Virtual Machine Disks.md

Tags:

    #Kubernetes
    #KubeVirt
    #PVC
    #PV
    #StorageClass
    #DataVolume
    #CDI
    #Longhorn
    #Virtual Machine Disks
    #Cloud-Native Virtualization
    #Platform Engineering

---

## I. Document Overview

This document explains the basic concepts of virtual machine storage in KubeVirt, common disk types, the relationships between PVC, DataVolume, and Longhorn, as well as practical exercises for verification.

Previous steps have included:

    1.Installing KubeVirt
    2.Installing virtctl
    3.Creating the first VM using containerDisk
    4.Installing CDI
    5.Importing images using DataVolume
    6.Using DataVolume as the virtual machine system disk

This document continues to cover the basics of KubeVirt storage.

Objectives:

    1. Understand why KubeVirt virtual machines rely on PVCs.
    2.Distinguish between system disks, data disks, temporary disks, and cloud-init disks.
    3.Comprehend the relationship between DataVolume and PVC.
    4.Know the role of Longhorn in KubeVirt.
    5.Create a blank DataVolume as a data disk.
    6.Mount both the system disk and data disk to the VM.
    7.View disk devices inside the VM.
    8 Verify that PVCs are retained after the VM shuts down.
    9.Familiarize yourself with common troubleshooting approaches for virtual machine disks.

Applicable Environment:

    Ubuntu 22.04
    Kubernetes v1.31.14
    KubeVirt v1.4.0
    CDI installed
    Longhorn installed
    StorageClass: longhorn
    virtctl installed

---

## II. Differences Between KubeVirt Storage and Regular Pod Storage

When regular Pods use PVCs, they are typically mounted to the container directory.

Example:

    PVC -> Pod -> /data

In KubeVirt, PVCs are usually treated as virtual machine disks.

Example:

    PVC -> VM -> /dev/vda
    PVC -> VM -> /dev/vdb

In other words:

    Regular Pods see directories.
    Virtual machines see disk devices.

Regular Pod:

    volumeMounts:
      mountPath: /data

KubeVirt VM:

    disks:
    - name: rootdisk
      disk:
        bus: virtio

    volumes:
    - name: rootdisk
      persistentVolumeClaim:
        claimName: vm-root-pvc

Inside the virtual machine, you might see:

    /dev/vda
    /dev/vdb
    /dev/sda
    /dev/sdb

The specific device names depend on the disk bus type and the Guest OS.

---

## III. Common Disk Types in KubeVirt

Common sources of disks in KubeVirt include:

    1. containerDisk
    2. persistentVolumeClaim
    3. dataVolume
    4. cloudInitNoCloud
    5. emptyDisk
    6. configMap / secret

For beginners, focus on mastering:

    containerDisk
    DataVolume
    PVC
    cloudInitNoCloud
    emptyDisk

---

## IV. containerDisk

containerDisk involves packaging a virtual machine disk image into a container image.

Example:

    volumes:
    - name: containerdisk
      containerDisk:
        image: quay.io/kubevirt/cirros-container-disk-demo:latest

Suitable for:

    1. Quick trials
    2.Demos
    3.Testing whether KubeVirt can start VMs
    4.Learning about the relationships between VM, VMI, and virt-launcher

Not suitable for:

    1.System disks in production environments
    2.VMs that require long-term data storage
    3.VMs whose system disk content needs to be updated frequently
    4.Real business virtual machines

Reason:

    containerDisk is more like a "boot disk distributed with the image."
    It is not suitable as a persistent, changeable system disk.

---

## V. DataVolume

DataVolume is a resource provided by CDI.

It is used to import virtual machine images into PVCs.

Common sources of import include:

    1.HTTP
    2 Registry
    3 PVC Clone
    4 Upload
    5 Blank

Typical process:

    Image URL
       |
       v
   ## Chapter Eleven, volumeMode: Filesystem and Block

There are two common volumeMode options for PVCs:

    Filesystem
    Block

At the beginner stage, **Filesystem** is usually used.

In some virtualization scenarios, **Block** is also employed.

Differences:

    Filesystem:
        The PVC is formatted as a file system before use.
        It is common, simple, and easy to get started with.

    Block:
        It is used directly as a block device.
        It is more closely related to block storage, but it requires clearer understanding of the storage and usage methods.

In this experiment, **Filesystem** will be used.

Whether to use **Block** in a production environment depends on the storage backend, performance requirements, and platform specifications.

---

## Chapter Twelve, Experiment Objectives

This experiment aims to achieve the following:

    1. Verify the Longhorn StorageClass.
    2. Check if CDI is available.
    3. Create a DataVolume for the CirrOS system disk.
    4. Create an empty DataVolume to use as a data disk.
    5. Create a VM and mount both the system disk and the data disk.
    6. Start the VM.
    7. Enter the console.
    8. View the disk devices inside the VM.
    9. (Optional) Format and mount the data disk.
    10. Stop the VM and confirm that the PVC remains intact.

---

## Chapter Thirteen, Pre-Experiment Checks

### 13.1 Checking KubeVirt

Execute:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

Requirements:

    KubeVirt must be available.
    Core components should be running.

---

### 13.2 Checking CDI

Execute:

    kubectl -n cdi get pods -o wide

    kubectl get crd | grep datavolumes

    kubectl get dv -A

Requirements:

    The CDI component must be running.
    Executing `kubectl get dv` should not result in any errors.

---

### 13.3 Checking Longhorn

Execute:

    kubectl -n longhorn-system get pods -o wide

    kubectl get storageclass longhorn

    kubectl -n longhorn-system get nodes.longhorn.io

Requirements:

    The Longhorn component must be running.
    The `longhorn StorageClass` should exist.
    Longhorn nodes must be available.

---

### 13.4 Checking Node KVM

On Worker nodes, execute:

    egrep -c '(vmx|svm)' /proc/cpuinfo

    ls -l /dev/kvm

    sudo kvm-ok

---

## Chapter Fourteen, Creating an Experiment Namespace

Create:

    kubectl create namespace kubevirt-storage-demo --dry-run=client -o yaml | kubectl apply -f -

Verify:

    kubectl get ns kubevirt-storage-demo

Create a directory:

    mkdir -p /root/k8s-yaml/kubevirt/storage-demo

    cd /root/k8s-yaml/kubevirt/storage-demo

---

## Chapter Fifteen, Experiment One: Creating a System Disk DataVolume

The system disk uses an HTTP source to import the CirrOS image.

Create:

    cat <<EOF > dv-rootdisk-cirros.yaml
    apiVersion: cdi.kubevirt.io/v1beta1
    kind: DataVolume
    metadata:
      name: cirros-rootdisk
      namespace: kubevirt-storage-demo
    spec:
      source:
        http:
          url: "https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"
      pvc:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 2Gi
        storageClassName: longhorn
    EOF

Apply:

    kubectl apply -f dv-rootdisk-cirros.yaml

Verify the DataVolume:

    kubectl -n kubevirt-storage-demo get dv

Verify the PVC:

    kubectl -n kubevirt-storage-demo get pvc

Check the importer Pod:

    kubectl -n kubevirt-storage-demo get pods

View the import logs:

    kubectl -n kubevirt-storage-demo logs <importer-pod-name> --tail=100

Wait for the import to complete:

    kubectl -n kubevirt-storage-demo get dv cirros-rootdisk

Expected result:

    PHASE: Succeeded

For the PVC, the expected status is:

    STATUS: Bound

Note:

    If downloading from the public internet fails, you can download the image to an internal HTTP server, for example:

        http://10.0.0.10/images/cirros-0.6.2-x86_64```markdown
                - name: ssh
                  port: 22
          networks:
          - name: default
            pod: {}
          volumes:
          - name: rootdisk
            dataVolume:
              name: cirros-rootdisk
          - name: datadisk
            dataVolume:
              name: cirros-datadisk
          - name: cloudinitdisk
            cloudInitNoCloud:
              userData: |
                #cloud-config
                password: kubevirt
                chpasswd:
                  expire: false
                ssh_pwauth: true
    EOF

Application:

    kubectl apply -f vm-cirros-storage.yaml

View:

    kubectl -n kubevirt-storage-demo get vm

At this point, the VM should be in the Stopped state.

---

## Experiment 18: Start the VM and Check Disk Status

Start:

    virtctl start vm-cirros-storage -n kubevirt-storage-demo

View the VM:

    kubectl -n kubevirt-storage-demo get vm

View the VMI:

    kubectl -n kubevirt-storage-demo get vmi

View the virt-launcher:

    kubectl -n kubevirt-storage-demo get pods -o wide | grep virt-launcher

View detailed VMI information:

    kubectl -n kubevirt-storage-demo describe vmi vm-cirros-storage

Pay attention to:

    Volumes
    Disks
    Conditions
    Events

Check the PVCs:

    kubectl -n kubevirt-storage-demo get pvc

Expected results:

    cirros-rootdisk Bound
    cirros-datadisk Bound
    VM Running
    VMI Running
    virt-launcher Pod Running

---

## Experiment 19: Enter the VM and View Disk Devices

Enter the console:

    virtctl console vm-cirros-storage -n kubevirt-storage-demo

Login:

    Username: cirros
    Password: kubevirt

After logging in, execute the following commands:

    hostname
    ip addr
    lsblk
    fdisk -l
    df -h

You should see at least two disks:

    System Disk
    Data Disk

The device names may be similar to:

    /dev/vda
    /dev/vdb

Note:

    Device names can vary depending on the system and drivers.
    Do not memorize /dev/vdb; instead, use lsblk or fdisk -l to identify them.

Exit the console:

    Ctrl + ]

---

## Experiment 20: Optionally Format and Mount the Data Disk

Note:

    This step will format the data disk.
    It should only be performed in a test environment.
    Do not directly perform mkfs on unknown disks of production VMs.

Inside the VM, confirm the data disk device name.

Assume the data disk is:

    /dev/vdb

Check:

    lsblk

Format the disk:

    sudo mkfs.ext4 /dev/vdb

Create a mount directory:

    sudo mkdir -p /data

Mount the disk:

    sudo mount /dev/vdb /data

Write a test message:

    echo "hello from kubevirt datadisk" | sudo tee /data/test.txt

View the message:

    cat /data/test.txt

Check the mount status:

    df -h

    mount | grep /data

Note:

    The CirrOS image may lack some tools.
    If commands like mkfs.ext4, lsblk, or fdisk are not available, you can first just observe the disk devices.
    You can perform a complete file system mounting experiment later using an Ubuntu or Rocky Linux image.

---

## Experiment 21: Restart the VM to Verify Data Disk Persistence

After writing data inside the VM, exit the console.

Restart the VM:

    virtctl restart vm-cirros-storage -n kubevirt-storage-demo

Wait:

    kubectl -n kubevirt-storage-demo get vmi

Re-enter the console:

    virtctl console vm-cirros-storage -n kubevirt-storage-demo

If you previously mounted and wrote data, you need to re-mount the data disk:

    sudo mkdir -p /data

    sudo mount /dev/vdb /data

View the content:

    cat /data/test.txt

If you can still see:

    hello from kubevirt datadisk

it means that the data disk PVC is persistently mounted.

Note:

    If /etc/fstab is not configured, the disk will not be automatically mounted after a restart.
    This is an issue with the Guest OS's internal configuration and does not indicate a loss of the KubeVirt PVC.

---

## Experiment 22: Stop the VM and Verify that the PVCs are Retained

Stop the VM:

    virtctl stop vm-cirros-storage -n kubevirt-storage-demo

View the VM:

    kubectl -n kubevirt-storage-demo get vm

View the VMI:

    kubectl -n kubevirt-storage-demo get```bash
kubectl -n kubevirt-storage-demo describe pvc <pvc-name>

kubectl get storageclass

kubectl -n cdi get pods -o wide

kubectl -n longhorn-system get pods -o wide
---
## Issue 26: DataVolume Import Failure

Check the importer Pod:

kubectl -n kubevirt-storage-demo get pods | grep importer

View logs:

kubectl -n kubevirt-storage-demo logs <importer-pod-name> --tail=100

Common causes:

1. The image URL is unreachable.
2. HTTPS certificate issues.
3. The company network cannot access the internet.
4. The image file is too large, and the PVC capacity is insufficient.
5. Abnormal image format.
6. PVC mounting failure.

Solution methods:

1. Replace with an internal HTTP image address.
2. Increase the PVC capacity.
3. Check the StorageClass.
4. Verify the CDI Pod.
5. Review the importer logs.
---
## Issue 27: PVC Bound but VM Failure to Start

Check:

kubectl -n kubevirt-storage-demo describe vm vm-cirros-storage

kubectl -n kubevirt-storage-demo describe vmi vm-cirros-storage

kubectl -n kubevirt-storage-demo get pods -o wide

kubectl -n kubevirt-storage-demo describe pod <virt-launcher-pod-name>

kubectl -n kubevirt-storage-demo logs <virt-launcher-pod-name> --tail=100

Common causes:

1. The image is not bootable.
2. Incorrect system disk format.
3. Wrong data disk configuration.
4. Issues with cloud-init settings.
5. The node does not support KVM.
6. Insufficient memory.
7. Abnormal PVC mounting.
8. Longhorn Volume attachment errors.
---
## Issue 28: Volume Unable to Be Attached

Check:

kubectl get volumeattachment

kubectl describe volumeattachment <name>

For Longhorn:

kubectl -n longhorn-system get volumes.longhorn.io

kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Node checks:

systemctl status iscsid --no-pager

systemctl status open-iscsi --no-pager

dpkg -l | grep open-iscsi

Common causes:

1. The Longhorn Volume is already mounted on another node.
2. RWO volumes cannot be used simultaneously by multiple nodes.
3. Issues with the node's iscsid service.
4. Abnormalities with Longhorn CSI.
5. Node NotReady status.
6. Abnormal Longhorn Volume status.
---
## Issue 29: Guest OS Cannot See the Data Disk

Kubernetes checks:

kubectl -n kubevirt-storage-demo describe vmi vm-cirros-storage

kubectl -n kubevirt-storage-demo get vm vm-cirros-storage -o yaml

Verify:

- The "disks" list includes "datadisk".
- The "volumes" list includes "datadisk".
- The names match exactly.

VM internal checks:

lsblk

fdisk -l

Common causes:

1. Inconsistent names between "disks" and "volumes".
2. The VM has not been restarted to reload the new disk.
3. The Guest OS lacks the necessary drivers.
4. The device name is different from expected (/dev/vdb).
5. The data disk has not been partitioned or formatted.
---
## Issue 30: /data Not Automatically Mounted After Restart

Reason:

Only the mount command was executed; /etc/fstab was not updated.

This is a configuration issue within the Guest OS, not related to the Kubernetes PVC.

Solution:

1. Retrieve the disk UUID.
2. Update /etc/fstab.
3. Verify the mount -a command.

Examples:

sudo blkid /dev/vdb

sudo vi /etc/fstab

Example content:

UUID=<uuid> /data ext4 defaults 0 0

Verify:

sudo mount -a

Note:

CirrOS is a minimal system and not suitable for detailed disk mounting experiments. Using Ubuntu or Rocky Linux images is more appropriate for practice.
---
## Issue 31: Disk Design Recommendations for Production Environments

### 31.1 Separating System Disks and Data Disks

Recommendations:

rootdisk:
- System disk.

datadisk:
- Business data disk.

Benefits:

1. Clear distinction between system and data areas.
2. Data disks can be expanded independently.
3. Data disks can be backed up separately.
4. Easier data retention during system reinstallation.
5. Simplified fault recovery processes.
---

### 31.2 Avoid Multiple VMs Sharing the Same RWO System Disk

Do not let multiple VMs write to the same RWO PVC simultaneously.

Risks:

1. File system corruption.
2. Data inconsistency.
3. VM startup failures.
4. Difficult troubleshooting.

Each VM should have its own system disk PVC.
---

### 31.3 Distinguish Between Image Dis```markdown
kubectl -n kubevirt-storage-demo describe pvc cirros-datadisk

kubectl get pv

---

### 33.3 CDI

kubectl -n cdi get pods -o wide

kubectl -n cdi get cdi

kubectl get crd | grep cdi

---

### 33.4 Longhorn

kubectl -n longhorn-system get pods -o wide

kubectl -n longhorn-system get volumes.longhorn.io

kubectl -n longhorn-system get nodes.longhorn.io

kubectl -n longhorn-system get replicas.longhorn.io

---

### 33.5 VolumeAttachment

kubectl get volumeattachment

kubectl describe volumeattachment <name>

---

### 33.6 VM 内部

lsblk

fdisk -l

df -h

mount

cat /etc/fstab

blkid

---

## 34. Clearing Experimental Resources

Stop the VM:

    virtctl stop vm-cirros-storage -n kubevirt-storage-demo

Delete the VM:

    kubectl delete -f vm-cirros-storage.yaml --ignore-not-found

Check VMI and Pods:

    kubectl -n kubevirt-storage-demo get vmi

    kubectl -n kubevirt-storage-demo get pods

Delete DataVolumes:

    kubectl delete -f dv-rootdisk-cirros.yaml --ignore-not-found

    kubectl delete -f dv-datadisk-blank.yaml --ignore-not-found

Check PVCs:

    kubectl -n kubevirt-storage-demo get pvc

If the PVCs are still present, delete them if they are no longer needed:

    kubectl -n kubevirt-storage-demo delete pvc cirros-rootdisk --ignore-not-found

    kubectl -n kubevirt-storage-demo delete pvc cirros-datadisk --ignore-not-found

Delete the namespace:

    kubectl delete namespace kubevirt-storage-demo

Note:

    The experimental environment can be cleared. In a production environment, make sure to confirm whether data needs to be retained before deleting DataVolumes/PVCs.

---

## 35. Summary of This Article

This article covers the basic concepts and practical operations of KubeVirt storage.

Key Points:

    1. Virtual machine disks in KubeVirt typically come from PVCs.
    2. DataVolumes are used to import images into PVCs.
    3. Longhorn can provide PVC block storage for virtual machines.
    4. System disks and data disks should be treated separately.
    5. containerDisk is suitable for experimentation, while DataVolume + PVC is closer to real-world usage.
    6. Blank DataVolumes can be used to create empty data disks.
    7. Inside a virtual machine, the devices seen are disk devices, not ordinary container mount directories.
    8. PVCs are retained even after the virtual machine is stopped.
    9. When dealing with disk issues, check all related components including the virtual machine, VMI, virt-launcher, DataVolume, PVC, PV, and Longhorn.

Key Object Relationships:

    DataVolume
       |
       v
    PVC
       |
       v
    Virtual Machine Disk
       |
       v
    VMI/virt-launcher
       |
       v
    Guest OS Disk Device

Most Important Commands:

    kubectl get dv -n kubevirt-storage-demo

    kubectl get pvc -n kubevirt-storage-demo

    kubectl describe vmi vm-cirros-storage -n kubevirt-storage-demo

    kubectl get volumeattachment

    kubectl -n longhorn-system get volumes.longhorn.io

    virtctl console vm-cirros-storage -n kubevirt-storage-demo

Tips and的经验:

    1. If a DataVolume fails, check the importer Pod logs first.
    2. If a PVC is in the "Pending" state, check the StorageClass and storage backend.
    3. If a virtual machine fails to start, examine both the VMI and virt-launcher.
    4. For Longhorn Volume issues, check the Volume, Replica, Node, and iscsid.
    5. If the guest OS cannot see the disk, verify that the names of the disks and volumes match.
    6. If a data disk does not mount automatically after restarting, it is likely due to an issue with the guest OS's fstab configuration.
    7. In a production environment, carefully plan for system disks, data disks, backups, snapshots, and recovery mechanisms.

Suggested Next Topic:

    08-KubeVirt Network Basics: Pod Networking, Service Exposure, Multus, and Multi-NIC Scenarios.md
```