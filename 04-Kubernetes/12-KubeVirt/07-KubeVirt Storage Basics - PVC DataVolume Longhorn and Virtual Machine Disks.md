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
    #VirtualMachineDisk
    #CloudlandVirtualization
    #PlatformEngineering

---

## I. Document Overview

This document records the basic concepts of storage in KubeVirt, common disk types, the relationship between PVC/DataVolume/Longhorn, and basic operational verification.

Previously completed:

    1. KubeVirt installation
    2. virtctl installation
    3. containerDisk creation of the first VM
    4. CDI installation
    5. DataVolume image import
    6. Using DataVolume as the VM root disk

This document continues to complete the basics of KubeVirt storage.

Document objectives:

    1. Understand why KubeVirt VMs depend on PVC
    2. Understand the differences between root disk, data disk, temporary disk, and cloud-init disk
    3. Understand the relationship between DataVolume and PVC
    4. Understand Longhorn's role in KubeVirt
    5. Create an empty DataVolume as a data disk
    6. Mount root disk and data disk simultaneously to VM
    7. View disk devices inside VM
    8. Verify that PVC remains after VM shutdown
    9. Master common troubleshooting paths for VM disks

Applicable environment:

    Ubuntu 22.04
    Kubernetes v1.31.14
    KubeVirt v1.4.0
    CDI already installed
    Longhorn already installed
    StorageClass: longhorn
    virtctl already installed

---

## II. Difference Between KubeVirt Storage and Regular Pod Storage

When regular Pods use PVC, it's typically mounted to container directories.

Example:

    PVC -> Pod -> /data

When KubeVirt uses PVC, it's typically treated as a VM disk.

Example:

    PVC -> VM -> /dev/vda
    PVC -> VM -> /dev/vdb

That is:

    Regular Pods see directories.
    VMs see disk devices.

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

Inside the VM, you might see:

    /dev/vda
    /dev/vdb
    /dev/sda
    /dev/sdb

The specific device names depend on the disk bus type and Guest OS.

---

## III. Common Disk Types in KubeVirt

Common disk sources in KubeVirt:

    1. containerDisk
    2. persistentVolumeClaim
    3. dataVolume
    4. cloudInitNoCloud
    5. emptyDisk
    6. configMap / secret

Focus areas in the beginner stage:

    containerDisk
    DataVolume
    PVC
    cloudInitNoCloud
    emptyDisk

---

## IV. containerDisk

containerDisk packages virtual machine disk images into container images.

Example:

    volumes:
    - name: containerdisk
      containerDisk:
        image: quay.io/kubevirt/cirros-container-disk-demo:latest

Suitable for:

    1. Quick experience
    2. Demo
    3. Testing if KubeVirt can start VM
    4. Learning VM / VMI / virt-launcher relationships

Not suitable for:

    1. Production root disks
    2. VMs needing long-term data preservation
    3. VMs requiring system disk content updates
    4. Real business VMs

Reason:

    containerDisk is more like a "boot disk distributed with the image".
    It's not suitable as a long-term, modifiable, and persistent root disk.

---

## V. DataVolume

DataVolume is a resource provided by CDI.

It is responsible for importing virtual machine images into PVC.

Common import sources:

    1. HTTP
    2. Registry
    3. PVC Clone
    4. Upload
    5. Blank

Typical flow:

    Image URL
       |
       v
    DataVolume
       |
       v
    importer Pod
       |
       v
    PVC
       |
       v
    VM boot disk

DataVolume is closer to real virtual machine disk management.

---

## VI. PVC

PVC is an object for requesting storage in Kubernetes.

In KubeVirt, PVC is often used for:

    1. VM root disk
    2. VM data disk
    3. Persistent disk after image import
    4. Data retention after VM restart
    5. Data retention/deletion based on policy after VM deletion

PVC may come from:

    Longhorn
    Ceph RBD
    NFS
    Cloud disk CSI
    Commercial storage CSI
    Local disk

This document focuses on:

    Longhorn

---

## VII. Longhorn's Role in KubeVirt

Longhorn is a native distributed block storage for Kubernetes.

In KubeVirt scenarios, Longhorn can provide PVC disks for VMs.

Relationship: /think

Longhorn StorageClass
        |
        v
    PVC
        |
        v
    KubeVirt VM Disk
        |
        v
    Guest OS Disk Device

Reasons why Longhorn is suitable for KubeVirt beginners and small-scale private scenarios:

    1. Relatively simple installation
    2. Native support for Kubernetes PVC
    3. Support for multi-replica
    4. Has UI
    5. Can view Volume status
    6. Easy to learn the relationship between VM disks and PVC

But note the following when using in production:

    1. Node disk planning
    2. Replica number planning
    3. Performance evaluation
    4. Backup targets
    5. Fault recovery drills
    6. Upgrade risks
    7. Whether it meets the performance requirements of business database workloads

---

## VIII. System Disk, Data Disk, and Temporary Disk Differences

### 8.1 System Disk

The system disk is used to boot the operating system.

Examples:

    Ubuntu
    Rocky Linux
    CentOS
    Debian
    Windows
    CirrOS

In KubeVirt, it typically comes from:

    DataVolume
    PVC
    containerDisk

Production recommendation:

    Use DataVolume / PVC as the system disk.

---

### 8.2 Data Disk

The data disk is used to store business data.

Examples:

    /data
    /var/lib/mysql
    /opt/app/data

In KubeVirt, it typically comes from:

    PVC
    blank DataVolume

Production recommendation:

    Separate system disk and data disk.
    Plan data disk capacity, performance, and backup strategy separately.

---

### 8.3 Temporary Disk

Temporary disks are typically used for caching or temporary data.

In KubeVirt, you can use:

    emptyDisk

Features:

    Data may be lost after VM lifecycle ends.
    Not suitable for storing important data.

---

### 8.4 cloud-init Disk

The cloud-init disk is used to inject initialization configurations.

Examples:

    Username
    Password
    SSH Key
    Hostname
    Initialization commands
    Network configuration

It is not a business data disk.

Common configuration:

    cloudInitNoCloud

---

## IX. Disk bus Types

Common disk bus types in KubeVirt:

    virtio
    sata
    scsi

Recommendation for beginners:

    virtio

Example:

    disks:
    - name: rootdisk
      disk:
        bus: virtio

Note:

    virtio is a high-performance paravirtualized device model commonly used in virtualization scenarios.
    Linux Guest OS typically has good support.

---

## X. accessModes and VM Disks

Common PVC accessModes:

    ReadWriteOnce
    ReadWriteMany
    ReadOnlyMany
    ReadWriteOncePod

Common for VM system disks:

    ReadWriteOnce

Reason:

    A regular system disk should typically be mounted in read-write mode by only one VM.

Common NFS support:

    ReadWriteMany

Longhorn common default:

    ReadWriteOnce

Note:

    Do not allow multiple VMs to write to the same regular system disk PVC simultaneously.
    This can easily lead to file system corruption.
    Multi-replica VMs should use their own independent system disks.

---

## XI. volumeMode: Filesystem vs Block

Common PVC volumeMode types:

    Filesystem
    Block

Common for beginners:

    Filesystem

Some virtualization scenarios also use Block.

Differences:

    Filesystem:
        Used after the PVC is formatted as a filesystem.
        Common, simple, and easy to get started.

    Block:
        Used directly as a block device.
        Closer to block storage, but requires more specific storage and usage specifications.

This experiment uses:

    Filesystem

Whether to use Block in production environments depends on the storage backend, performance requirements, and platform specifications.

---

## XII. Experiment Objectives

This experiment completes the following:

    1. Check Longhorn StorageClass
    2. Check CDI availability
    3. Create a CirrOS system disk DataVolume
    4. Create a blank DataVolume as a data disk
    5. Create a VM with both system and data disks mounted
    6. Start the VM
    7. Enter the console
    8. View disk devices inside the VM
    9. Optionally format and mount the data disk
    10. Confirm PVC retention after stopping the VM

---

## XIII. Pre-Experiment Checks

### 13.1 Check KubeVirt

Run:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

Requirements:

    KubeVirt Available
    Core components Running

---

### 13.2 Check CDI

Run:

    kubectl -n cdi get pods -o wide

    kubectl get crd | grep datavolumes

    kubectl get dv -A

Requirements:

    CDI components Running
    kubectl get dv does not return errors

---

### 13.3 Check Longhorn

Run:

    kubectl -n longhorn-system get pods -o wide

    kubectl get storageclass longhorn

    kubectl -n longhorn-system get nodes.longhorn.io

Requirements:

    Longhorn components Running
    longhorn StorageClass exists
    Longhorn nodes available

---

### 13.4 Check Node KVM

Run on Worker node:

    egrep -c '(vmx|svm)' /proc/cpuinfo

    ls -l /dev/kvm

    sudo kvm-ok

---

## XIV. Create Experiment Namespace

Create: /think

kubectl create namespace kubevirt-storage-demo --dry-run=client -o yaml | kubectl apply -f -

View:

    kubectl get ns kubevirt-storage-demo

Create directory:

    mkdir -p /root/k8s-yaml/kubevirt/storage-demo

    cd /root/k8s-yaml/kubevirt/storage-demo

---

## Fifteen, Experiment One: Creating a System Disk DataVolume

The system disk uses HTTP to import a CirrOS image.

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

View DataVolume:

    kubectl -n kubevirt-storage-demo get dv

View PVC:

    kubectl -n kubevirt-storage-demo get pvc

View importer Pod:

    kubectl -n kubevirt-storage-demo get pods

View import log:

    kubectl -n kubevirt-storage-demo logs <importer-pod-name> --tail=100

Wait for import completion:

    kubectl -n kubevirt-storage-demo get dv cirros-rootdisk

Expected:

    PHASE: Succeeded

PVC expected:

    STATUS: Bound

Note:

    If public internet download fails, you can download the image to an internal HTTP service, for example:

        http://10.0.0.10/images/cirros-0.6.2-x86_64-disk.img

    Then modify source.http.url.

---

## Sixteen, Experiment Two: Creating a Blank Data Disk DataVolume

Create a blank disk as a VM data disk.

Create:

    cat <<EOF > dv-datadisk-blank.yaml
    apiVersion: cdi.kubevirt.io/v1beta1
    kind: DataVolume
    metadata:
      name: cirros-datadisk
      namespace: kubevirt-storage-demo
    spec:
      source:
        blank: {}
      pvc:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
        storageClassName: longhorn
    EOF

Apply:

    kubectl apply -f dv-datadisk-blank.yaml

View:

    kubectl -n kubevirt-storage-demo get dv

    kubectl -n kubevirt-storage-demo get pvc

Wait:

    kubectl -n kubevirt-storage-demo get dv cirros-datadisk

Expected:

    PHASE: Succeeded

Note:

    source.blank indicates creating a blank disk.
    It can be used as a data disk inside the VM later.

---

## Seventeen, Experiment Three: Creating a VM that mounts both the system disk and data disk

Create VM: /think

```bash
cat <<EOF > vm-cirros-storage.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-cirros-storage
  namespace: kubevirt-storage-demo
  labels:
    app: vm-cirros-storage
spec:
  runStrategy: Manual
  template:
    metadata:
      labels:
        app: vm-cirros-storage
        kubevirt.io/domain: vm-cirros-storage
    spec:
      terminationGracePeriodSeconds: 0
      domain:
        resources:
          requests:
            memory: 512Mi
        devices:
          disks:
          - name: rootdisk
            disk:
              bus: virtio
          - name: datadisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
            ports:
            - name: ssh
              port: 2
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
```

**Apply:**

```bash
kubectl apply -f vm-cirros-storage.yaml
```

**Check:**

```bash
kubectl -n kubevirt-storage-demo get vm
```

At this point the VM should be **Stopped**.

---

## Eighteen. Experiment Four: Start VM and Observe Disk Status

**Start:**

```bash
virtctl start vm-cirros-storage -n kubevirt-storage-demo
```

**Check VM:**

```bash
kubectl -n kubevirt-storage-demo get vm
```

**Check VMI:**

```bash
kubectl -n kubevirt-storage-demo get vmi
```

**Check virt-launcher:**

```bash
kubectl -n kubevirt-storage-demo get pods -o wide | grep virt-launcher
```

**Check VMI details:**

```bash
kubectl -n kubevirt-storage-demo describe vmi vm-cirros-storage
```

**Focus on:**

- Volumes
- Disks
- Conditions
- Events

**Check PVC:**

```bash
kubectl -n kubevirt-storage-demo get pvc
```

**Expected:**

```
cirros-rootdisk Bound
cirros-datadisk Bound
VM Running
VMI Running
virt-launcher Pod Running
```

---

## Nineteen. Experiment Five: Enter VM to View Disk Devices

**Enter console:**

```bash
virtctl console vm-cirros-storage -n kubevirt-storage-demo
```

**Login:**

- Username: cirros
- Password: kubevirt

**After login, execute:**

```bash
hostname
ip addr
lsblk
fdisk -l
df -h
```

**Expected to see at least two disks:**

- System disk
- Data disk

**Possible device names:**

```
/dev/vda
/dev/vdb
```

**Note:**
- Device names may vary depending on the system and driver.
- Do not memorize /dev/vdb.
- Determine through lsblk / fdisk -l.

**Exit console:**

```
Ctrl + ]
```

---

## Twenty. Experiment Six: Optional Format and Mount Data Disk

**Note:**
- This step will format the data disk.
- Only allowed to execute in experimental environments.
- Do not directly mkfs on unknown disks of production VMs.

**In VM, confirm the data disk device name.**

**Assume data disk is:**
```
/dev/vdb
```

**Check:**
```bash
lsblk
```

**Format:**
```bash
sudo mkfs.ext4 /dev/vdb
```

**Create mount directory:**
```bash
sudo mkdir -p /data
```

**Mount:**
```bash
sudo mount /dev/vdb /data
```

**Write test:**
```bash
echo "hello from kubevirt datadisk" | sudo tee /data/test.txt
```

**Check:**
```bash
cat /data/test.txt
```

**Check mount:** `/data`

```markdown
df -h

mount | grep /data

**Note:**

CirrOS image is small and may lack some tools.
If commands like mkfs.ext4, lsblk, fdisk, etc., are missing, you can first observe the disk devices.
Later, perform a full file system mount experiment using Ubuntu / Rocky Linux image.

---

## 21. Experiment 7: Verify Data Disk Persistence After VM Restart

Write data and exit console inside the VM.

Restart VM:

    virtctl restart vm-cirros-storage -n kubevirt-storage-demo

Wait:

    kubectl -n kubevirt-storage-demo get vmi

Re-enter console:

    virtctl console vm-cirros-storage -n kubevirt-storage-demo

If data disk was previously mounted and data was written, re-mount the data disk:

    sudo mkdir -p /data

    sudo mount /dev/vdb /data

Check:

    cat /data/test.txt

If you can still see:

    hello from kubevirt datadisk

It indicates that the PVC data disk persistence is normal.

**Note:**
If /etc/fstab is not configured, the disk will not be automatically mounted after reboot.
This is an internal system configuration issue of the Guest OS, not a loss of KubeVirt PVC.

---

## 22. Experiment 8: Confirm PVC Retention After Stopping VM

Stop VM:

    virtctl stop vm-cirros-storage -n kubevirt-storage-demo

Check VM:

    kubectl -n kubevirt-storage-demo get vm

Check VMI:

    kubectl -n kubevirt-storage-demo get vmi

Check Pod:

    kubectl -n kubevirt-storage-demo get pods

Check PVC:

    kubectl -n kubevirt-storage-demo get pvc

**Expected:**
VM exists
VMI disappears
virt-launcher Pod disappears
PVC still exists

**Note:**
Stopping VM does not equal deleting the disk.
PVC still retains the system disk and data disk content.

---

## 23. Experiment 9: Observe VM Disk via Longhorn UI

If Longhorn UI is exposed, you can view the Volume in Longhorn UI.

You can also check via command:

    kubectl -n longhorn-system get volumes.longhorn.io

Find the Longhorn Volume corresponding to the PVC.

Check PVC corresponding PV:

    kubectl -n kubevirt-storage-demo get pvc

    kubectl get pv

Check PV details:

    kubectl describe pv <pv-name>

In Longhorn, pay attention to:

Volume status
Replica count
Node location
Health status
Attached node
Data locality
Replica rebuild status

**Note:**
When KubeVirt VM disk anomalies occur, in addition to checking KubeVirt, also check Longhorn Volume status.

---

## 24. KubeVirt Disk Troubleshooting Path

When VM disk anomalies occur, it is recommended to troubleshoot in the following order:

    1. kubectl get vm
    2. kubectl describe vm
    3. kubectl get vmi
    4. kubectl describe vmi
    5. kubectl get pods | grep virt-launcher
    6. kubectl describe pod virt-launcher
    7. kubectl get dv
    8. kubectl describe dv
    9. kubectl get pvc
    10. kubectl describe pvc
    11. kubectl get pv
    12. kubectl describe pv
    13. kubectl get volumeattachment
    14. Check Longhorn Volume
    15. Check node kubelet logs
    16. Check virt-launcher logs

---

## 25. Common Issue 1: DataVolume Pending

Check:

    kubectl -n kubevirt-storage-demo get dv

    kubectl -n kubevirt-storage-demo describe dv <dv-name>

Common causes:

    1. PVC Pending
    2. StorageClass does not exist
    3. CDI is abnormal
    4. Longhorn is abnormal
    5. Storage capacity is insufficient

Continue troubleshooting:

    kubectl -n kubevirt-storage-demo get pvc

    kubectl -n kubevirt-storage-demo describe pvc <pvc-name>

    kubectl get storageclass

    kubectl -n cdi get pods -o wide

    kubectl -n longhorn-system get pods -o wide

---

## 26. Common Issue 2: DataVolume Import Failure

Check importer Pod:

    kubectl -n kubevirt-storage-demo get pods | grep importer

Check logs:

    kubectl -n kubevirt-storage-demo logs <importer-pod-name> --tail=100

Common causes:

    1. Image URL is inaccessible
    2. HTTPS certificate issue
    3. Corporate network cannot access public internet
    4. Image file is too large, PVC capacity is insufficient
    5. Image format is abnormal
    6. PVC mount failure

**Resolution:** 
```

1. Switch to internal HTTP mirror address  
2. Increase PVC capacity  
3. Check StorageClass  
4. Check CDI Pod  
5. Check importer logs  

---

## 27. Common Issue Three: PVC Bound but VM Fails to Start  

Check:  

    kubectl -n kubevirt-storage-demo describe vm vm-cirros-storage  

    kubectl -n kubevirt-storage-demo describe vmi vm-cirros-storage  

    kubectl -n kubevirt-storage-demo get pods -o wide  

    kubectl -n kubevirt-storage-demo describe pod <virt-launcher-pod-name>  

    kubectl -n kubevirt-storage-demo logs <virt-launcher-pod-name> --tail=100  

Common causes:  

    1. Image cannot be started  
    2. System disk format error  
    3. Data disk configuration error  
    4. cloud-init configuration error  
    5. Node does not support KVM  
    6. Insufficient memory  
    7. PVC mount anomaly  
    8. Longhorn Volume attach anomaly  

---

## 28. Common Issue Four: Volume Cannot Attach  

Check:  

    kubectl get volumeattachment  

    kubectl describe volumeattachment <name>  

Longhorn check:  

    kubectl -n longhorn-system get volumes.longhorn.io  

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>  

Node check:  

    systemctl status iscsid --no-pager  

    systemctl status open-iscsi --no-pager  

    dpkg -l | grep open-iscsi  

Common causes:  

    1. Longhorn Volume already mounted to another node  
    2. RWO volume cannot be used by multiple nodes simultaneously  
    3. Node iscsid anomaly  
    4. Longhorn CSI anomaly  
    5. Node NotReady  
    6. Longhorn Volume status anomaly  

---

## 29. Common Issue Five: Guest OS Cannot See Data Disk  

Kubernetes side check:  

    kubectl -n kubevirt-storage-demo describe vmi vm-cirros-storage  

    kubectl -n kubevirt-storage-demo get vm vm-cirros-storage -o yaml  

Confirm:  

    disks contains datadisk  
    volumes contains datadisk  
    names are completely consistent  

VM internal check:  

    lsblk  

    fdisk -l  

Common causes:  

    1. disks and volumes names are inconsistent  
    2. VM has not restarted to load new disks  
    3. Guest OS lacks drivers  
    4. Device name is not /dev/vdb  
    5. Data disk has not been partitioned or formatted  

---

## 30. Common Issue Six: /data Not Automatically Mounted After Reboot  

Cause:  

    Only executed mount command, but did not write to /etc/fstab.  

This is an internal Guest OS configuration issue, not a Kubernetes PVC loss.  

Handling:  

    1. View disk UUID  
    2. Write to /etc/fstab  
    3. Validate with mount -a  

Example:  

    sudo blkid /dev/vdb  

    sudo vi /etc/fstab  

Example content:  

    UUID=<uuid> /data ext4 defaults 0 0  

Validation:  

    sudo mount -a  

Note:  

    CirrOS is a minimal system, not suitable for full production disk mounting experiments.  
    Later use Ubuntu / Rocky Linux images are more suitable for practice.  

---

## 31. Production Environment Disk Design Recommendations  

### 31.1 Separate System Disk and Data Disk  

Recommendation:  

    rootdisk:  
        System disk  

    datadisk:  
        Business data disk  

Benefits:  

    1. Clear boundary between system and data  
    2. Data disk can be expanded separately  
    3. Data disk can be backed up separately  
    4. Data can be retained more easily during system reinstallation  
    5. Fault recovery is clearer  

---

### 31.2 Do Not Share a RWO System Disk Among Multiple VMs  

Do not allow multiple VMs to write to the same RWO PVC simultaneously.  

Risks:  

    1. File system corruption  
    2. Data inconsistency  
    3. VM startup anomalies  
    4. Difficult troubleshooting  

Each VM should have its own system disk PVC.  

---

### 31.3 Distinguish Between Image Disk and Runtime Disk  

Image is a template.  

Runtime disk is the PVC actually used by a VM.  

Do not treat the base image as a shared system disk for multiple VMs.  

Correct approach:  

    Base image  
       |  
       v  
    Clone / Import  
       |  
       v  
    Independent PVC for each VM  
       |  
       v  
    VM startup  

---

### 31.4 Data Disk Should Have Backup Strategy  

Virtual machine data disks are not safe just because they have PVCs.  

Additional requirements:  

    1. Snapshots  
    2. Backups  
    3. Recovery drills  
    4. Cross-node replication  
    5. Storage backend health checks  
    6. Protection against accidental deletion  
    7. Access control  

Longhorn scenarios should pay special attention to:  

    Backup Target  
    Snapshot  
    Recurring Job  
    Volume Health  
    Replica Rebuild  

---

## 32. Virtual Machine Disk Expansion Explanation  

Prerequisites for PVC expansion support:  

    StorageClass allowVolumeExpansion=true  

Check:  

    kubectl get storageclass longhorn -o yaml | grep allowVolumeExpansion  

If expansion is supported, expand the PVC.  

Example: /think

kubectl -n kubevirt-storage-demo patch pvc cirros-datadisk \
  -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'

Then you also need to expand the filesystem inside the VM.

Note:

  - Expansion steps vary depending on the Guest OS, filesystem, and disk partitioning method.
  - Backups must be performed before expansion in production environments.
  - VM disk expansion is recommended to be documented separately.

---

## 33. Standard Troubleshooting Command List

### 33.1 KubeVirt Objects

  kubectl -n kubevirt-storage-demo get vm

  kubectl -n kubevirt-storage-demo describe vm vm-cirros-storage

  kubectl -n kubevirt-storage-demo get vmi

  kubectl -n kubevirt-storage-demo describe vmi vm-cirros-storage

  kubectl -n kubevirt-storage-demo get pods -o wide

  kubectl -n kubevirt-storage-demo logs <virt-launcher-pod-name> --tail=100

---

### 33.2 DataVolume / PVC / PV

  kubectl -n kubevirt-storage-demo get dv

  kubectl -n kubevirt-storage-demo describe dv cirros-rootdisk

  kubectl -n kubevirt-storage-demo describe dv cirros-datadisk

  kubectl -n kubevirt-storage-demo get pvc

  kubectl -n kubevirt-storage-demo describe pvc cirros-rootdisk

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

### 33.6 Inside the VM

  lsblk

  fdisk -l

  df -h

  mount

  cat /etc/fstab

  blkid

---

## 34. Clean Up Experimental Resources

Stop the VM:

  virtctl stop vm-cirros-storage -n kubevirt-storage-demo

Delete the VM:

  kubectl delete -f vm-cirros-storage.yaml --ignore-not-found

Check VMI and Pod:

  kubectl -n kubevirt-storage-demo get vmi

  kubectl -n kubevirt-storage-demo get pods

Delete DataVolume:

  kubectl delete -f dv-rootdisk-cirros.yaml --ignore-not-found

  kubectl delete -f dv-datadisk-blank.yaml --ignore-not-found

Check PVC:

  kubectl -n kubevirt-storage-demo get pvc

If the PVC still exists, confirm it's not needed before deletion:

  kubectl -n kubevirt-storage-demo delete pvc cirros-rootdisk --ignore-not-found

  kubectl -n kubevirt-storage-demo delete pvc cirros-datadisk --ignore-not-found

Delete namespace:

  kubectl delete namespace kubevirt-storage-demo

Note:

  - Experimental environments can be cleaned up.
  - Production environments must confirm data retention before deleting DataVolume / PVC.

---

## 35. Summary of This Article

This article completed the basic learning and hands-on practice of KubeVirt storage.

Core content:

  1. KubeVirt VM disks typically come from PVC
  2. DataVolume is used to import images into PVC
  3. Longhorn can provide PVC block storage capabilities for VM
  4. System disk and data disk should be understood separately
  5. containerDisk is suitable for experience, DataVolume + PVC is closer to real usage
  6. blank DataVolume can create blank data disks
  7. The VM sees disk devices, not regular container mount directories
  8. PVC remains after VM stops
  9. Disk issues require checking VM, VMI, virt-launcher, DataVolume, PVC, PV, and Longhorn simultaneously

Core object relationships:

  DataVolume
     |
     v
  PVC
     |
     v
  VM Disk
     |
     v
  VMI / virt-launcher
     |
     v
  Guest OS Disk Device

Most important commands:

  kubectl get dv -n kubevirt-storage-demo

kubectl get pvc -n kubevirt-storage-demo

kubectl describe vmi vm-cirros-storage -n kubevirt-storage-demo

kubectl get volumeattachment

kubectl -n longhorn-system get volumes.longhorn.io

virtctl console vm-cirros-storage -n kubevirt-storage-demo

Experience-based judgment:

    1. DataVolume failed, check importer Pod logs first
    2. PVC Pending, check StorageClass and storage backend first
    3. VM fails to start, must check both VMI and virt-launcher
    4. Longhorn Volume anomaly, check Volume, Replica, Node and iscsid
    5. Guest OS cannot see disk, check if disks and volumes names match
    6. Data disk not auto-mounted after reboot, usually due to Guest OS internal fstab issue
    7. Production environment must plan system disk, data disk, backup, snapshot and recovery

Next suggested learning: 

    08-KubeVirt network basics: Pod network, Service exposure, Multus and multi-NIC scenarios.md