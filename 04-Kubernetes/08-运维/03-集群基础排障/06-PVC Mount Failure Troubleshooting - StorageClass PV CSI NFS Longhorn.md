# 06-PVC Mount Failure Troubleshooting: StorageClass, PV, CSI, NFS, and Longhorn

Recommended Path:

    04-Kubernetes/08-Operations/03-Cluster Basic Troubleshooting/06-PVC Mount Failure Troubleshooting: StorageClass, PV, CSI, NFS, and Longhorn.md

Tags:

    #Kubernetes
    #PVC
    #PV
    #StorageClass
    #CSI
    #NFS
    #Longhorn
    #VolumeMount
    #EnduringStorage
    #ClusterInfrastructureBarriers

---

## I. Document Overview

This document records basic troubleshooting methods for PVC mount failures, Pod failures to start due to storage issues, and PV/PVC binding anomalies in Kubernetes.

PVC-related issues are very common in stateful applications, such as:

    MySQL
    Redis
    MongoDB
    PostgreSQL
    Elasticsearch
    MinIO
    Jenkins
    GitLab
    Harbor
    Prometheus
    Grafana
    Loki

Common Phenomena:

    1. PVC remains in Pending state
    2. Pod remains in Pending state
    3. Pod remains in ContainerCreating state
    4. FailedMount appears in Pod Events
    5. FailedAttachVolume appears in Pod Events
    6. PVC cannot be Bound
    7. PV is in Released / Failed state
    8. StorageClass does not exist
    9. CSI Controller / Node Plugin anomalies
    10. NFS cannot be mounted
    11. Longhorn Volume cannot be Attached
    12. StatefulSet Pod creation fails

Document Objectives:

    1. Understand the relationship between PVC, PV, StorageClass, and CSI
    2. Be able to distinguish between PVC binding failure and mount failure
    3. Be able to troubleshoot PVC Pending
    4. Be able to troubleshoot Pod FailedMount
    5. Be able to troubleshoot non-existent or misconfigured StorageClass
    6. Be able to troubleshoot NFS dynamic provisioning issues
    7. Be able to troubleshoot Longhorn volume mounting issues
    8. Be able to troubleshoot StatefulSet volumeClaimTemplates issues
    9. Establish a standardized storage troubleshooting path

Applicable Scenarios:

    1. kubeadm self-hosted cluster
    2. Private Kubernetes cluster
    3. NFS StorageClass
    4. Longhorn StorageClass
    5. CSI storage plugin
    6. Stateful application mount failure
    7. Pod Pending / ContainerCreating issues related to PVC

---

## II. PVC Access Chain

Kubernetes persistent storage basic chain:

    Pod
     |
     v
    volumeMounts
     |
     v
    volumes
     |
     v
    PVC
     |
     v
    PV
     |
     v
    StorageClass / CSI Provisioner
     |
     v
    Backend Storage
        NFS
        Longhorn
        Ceph
        Cloud Disk
        Local Disk
        Commercial Storage

Dynamic Provisioning Scenario:

    User creates PVC
       |
       v
    StorageClass finds provisioner
       |
       v
    Provisioner creates backend storage
       |
       v
    Creates PV
       |
       v
    PVC is Bound
       |
       v
    Pod schedules and mounts PVC
       |
       v
    kubelet calls CSI / mounting tool to complete mounting

Core Judgment:

    Before PVC is Bound, Pod may be Pending.
    After PVC is Bound, Pod may still be stuck in ContainerCreating due to mounting failure.
    PVC being Bound does not guarantee Pod will successfully mount.
    Pod Running does not mean storage read/write is definitely normal; need to verify inside the container.

---

## III. Basic Relationship Between PVC/PV/StorageClass

| Object | Function |
|---|---|
| StorageClass | Defines dynamic provisioning method, e.g. nfs-client, longhorn |
| PVC | User requests storage |
| PV | Actual persistent volume object in Kubernetes |
| CSI Controller | Responsible for creating, deleting, Attaching, Detaching volumes |
| CSI Node Plugin | Responsible for node-side mounting and unmounting volumes |
| Pod | Uses storage via PVC |
| kubelet | Executes mounting and container startup on the node |

Simple Understanding:

    StorageClass explains "how storage comes"
    PVC explains "how much storage I need"
    PV explains "the actual allocated storage"
    Pod explains "where to mount this storage"

---

## IV. Common Status Explanations

### 4.1 PVC Pending

Indicates PVC has not been bound to a PV yet.

Common Causes:

    1. StorageClass does not exist
    2. StorageClass provisioner anomaly
    3. Backend storage unavailable
    4. Dynamic provisioning failure
    5. No matching static PV available
    6. accessModes mismatch
    7. Storage capacity not met
    8. volumeBindingMode waits for first consumer

---

### 4.2 PVC Bound

Indicates PVC has been bound to a PV.

Note:

    PVC Bound only means the volume has been allocated.
    Does not mean Pod has successfully mounted.
    Need to check Pod Events later.

---

### 4.3 Pod Pending

If Pod depends on PVC and PVC is Pending, Pod may also be Pending.

Common Events:

    pod has unbound immediate PersistentVolumeClaims

---

### 4.4 Pod ContainerCreating

If a PVC is already Bound but the mounting phase fails, the Pod may become stuck in ContainerCreating.

Common events:

    FailedMount
    FailedAttachVolume
    MountVolume.SetUp failed
    Unable to attach or mount volumes
    timed out waiting for the condition

---

### 4.5 PV Released

Indicates that the PVC has been deleted, but the PV has not been automatically deleted or reclaimed.

Common scenarios:

    reclaimPolicy: Retain

Manual intervention is required to determine whether to clean up or rebind.

---

## Five. Troubleshooting Overview

PVC/storage troubleshooting recommendations: check in the following order:

    1. Check Pod status
    2. describe Pod to view Events
    3. Check PVC status
    4. describe PVC to view Events
    5. Check PV status
    6. Check StorageClass
    7. Check provisioner / CSI components
    8. Check backend storage status
    9. Check kubelet logs on the node where the Pod resides
    10. Manually verify mount dependencies on the node
    11. Reobserve the Pod after fixing the issue to check if it is Running
    12. Enter the Pod to validate read/write operations

Troubleshooting branches:

    Pod storage abnormality
        |
        |-- PVC Pending
        |       |
        |       |-- StorageClass does not exist
        |       |-- provisioner abnormality
        |       |-- backend storage unavailable
        |       |-- accessModes / capacity mismatch
        |
        |-- PVC Bound but Pod mounting failed
                |
                |-- FailedMount
                |-- FailedAttachVolume
                |-- CSI Node Plugin abnormality
                |-- Missing NFS client
                |-- Longhorn iSCSI abnormality
                |-- Node network or permission issues

---

## Six. Step 1: Check Pod Status

Check Pod:

    kubectl get pod -n <namespace> -o wide

Example:

    kubectl get pod -n default -o wide

Focus on:

    STATUS
    READY
    NODE
    AGE

Common statuses:

    Pending
    ContainerCreating
    Running
    CrashLoopBackOff

If the Pod is in Pending or ContainerCreating status, continue with describe.

---

## Seven. Step 2: Check Pod Events

Execute:

    kubectl describe pod <pod-name> -n <namespace>

Focus on Events.

Common PVC/storage-related events:

    pod has unbound immediate PersistentVolumeClaims

    FailedMount

    FailedAttachVolume

    MountVolume.SetUp failed

    Unable to attach or mount volumes

    timed out waiting for the condition

    persistentvolumeclaim "xxx" not found

    volume is already exclusively attached to one node and can't be attached to another

    rpc error: code = Internal desc

    mount.nfs: access denied by server

    mount.nfs: Connection timed out

Example:

    Warning  FailedMount  kubelet  MountVolume.SetUp failed for volume "data": mount failed

Note:

    Pod Events is the primary entry point for diagnosing storage issues.
    Do not only check PVC status.
    Even after PVC is Bound, the Pod may still fail to mount.

---

## Eight. Step 3: Check PVC Status

Check current namespace PVC:

    kubectl get pvc -n <namespace>

Example:

    kubectl get pvc -n default

Check all namespace PVC:

    kubectl get pvc -A

Common statuses:

    Pending
    Bound
    Lost

Check detailed information:

    kubectl describe pvc <pvc-name> -n <namespace>

Focus on:

    Status
    Volume
    StorageClass
    Capacity
    Access Modes
    VolumeMode
    Events

---

## Nine. Step 4: Check PV Status

Check PV:

    kubectl get pv

Check detailed information:

    kubectl describe pv <pv-name>

Focus on:

    Status
    Claim
    StorageClass
    Capacity
    Access Modes
    Reclaim Policy
    Source
    CSI
    NFS
    Node Affinity

Common statuses:

    Available
    Bound
    Released
    Failed

Note:

    Bound indicates it is bound to a PVC.
    Released indicates the original PVC has been deleted, but the PV remains.
    Failed indicates failure in recycling or deletion.

---

## Ten. Step 5: Check StorageClass

Check StorageClass:

    kubectl get storageclass

Check detailed information:

kubectl describe storageclass <storageclass-name>

Key Focus:

    Provisioner
    ReclaimPolicy
    VolumeBindingMode
    AllowVolumeExpansion
    Parameters

Example:

    NAME                   PROVISIONER
    nfs-client (default)   cluster.local/nfs-subdir-external-provisioner
    longhorn               driver.longhorn.io

Common Issues:

    1. PVC specifies a non-existent storageClassName
    2. No default StorageClass exists
    3. Default StorageClass is not expected
    4. Provisioner name is incorrect
    5. VolumeBindingMode causes delayed binding
    6. ReclaimPolicy does not match expectations

---

## Eleven. PVC Pending Troubleshooting

### 11.1 StorageClass Does Not Exist

PVC Example:

    storageClassName: longhorn

But the cluster has no longhorn:

    kubectl get storageclass

PVC Events may show:

    storageclass.storage.k8s.io "longhorn" not found

Resolution:

    1. Install the corresponding StorageClass
    2. Modify PVC storageClassName
    3. If it's a StatefulSet, need to modify volumeClaimTemplates and restart
    4. Confirm if a default StorageClass exists

---

### 11.2 No Default StorageClass

If PVC does not specify storageClassName:

    spec:
      resources:
        requests:
          storage: 1Gi

But the cluster has no default StorageClass, PVC will be Pending.

Check:

    kubectl get storageclass

If none:

    (default)

You can set the default StorageClass:

    kubectl patch storageclass <storageclass-name> \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

Cancel default:

    kubectl patch storageclass <storageclass-name> \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

---

### 11.3 Dynamic Provisioner Abnormality

PVC Events may show:

    failed to provision volume with StorageClass

Troubleshoot:

    kubectl describe pvc <pvc-name> -n <namespace>

    kubectl get storageclass

    kubectl describe storageclass <storageclass-name>

Then find the corresponding component based on Provisioner.

NFS:

    kubectl -n storage-system get pods -o wide

    kubectl -n storage-system logs deploy/nfs-subdir-external-provisioner --tail=100

Longhorn:

    kubectl -n longhorn-system get pods -o wide

    kubectl -n longhorn-system get volumes.longhorn.io

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=100

---

### 11.4 volumeBindingMode: WaitForFirstConsumer

Check StorageClass:

    kubectl describe storageclass <storageclass-name>

If you see:

    VolumeBindingMode: WaitForFirstConsumer

It means PVC may wait until a Pod uses it before binding.

This is not necessarily an error.

Common in:

    Local disks
    Cloud disks
    Storage requiring Pod scheduling topology

Troubleshoot:

    1. Check if any Pod uses this PVC
    2. Check if Pod is Pending
    3. Describe Pod to check scheduling events
    4. Verify node topology and storage topology compatibility

---

## Twelve. Pod FailedMount Troubleshooting

If PVC is already Bound but Pod still fails to start, focus on Pod Events.

Check:

    kubectl describe pod <pod-name> -n <namespace>

Common events:

    FailedMount
    Unable to attach or mount volumes
    MountVolume.SetUp failed

Continue investigation:

    1. Pod's node
    2. kubelet logs
    3. CSI Node Plugin
    4. Backend storage connectivity
    5. Whether node has mount dependencies installed

Check Pod's node:

    kubectl get pod <pod-name> -n <namespace> -o wide

Login to the node and check kubelet logs:

    sudo journalctl -u kubelet -n 200 --no-pager

Real-time view:

    sudo journalctl -u kubelet -f

---

## Thirteen. NFS PVC Troubleshooting

### 13.1 NFS Common Issues

NFS-related common errors: /think

mount.nfs: access denied by server  
mount.nfs: Connection timed out  
mount.nfs: No such file or directory  
bad option; for several filesystems you might need a /sbin/mount.<type> helper program  
permission denied  
stale file handle  

Common causes:  

    1. The node has not installed nfs-common  
    2. NFS Server is unreachable  
    3. NFS shared directory is not exported  
    4. /etc/exports permissions do not allow current node access  
    5. NFS directory does not exist  
    6. NFS directory permissions are insufficient  
    7. Firewall blocks port 2049  
    8. NFS Server is abnormal  

---

### 13.2 Check NFS Server  

Execute on NFS Server:  

    systemctl status nfs-server --no-pager  

    sudo exportfs -v  

    showmount -e 127.0.0.1  

    sudo ss -lntup | grep -E "2049|111"  

Check shared directory:  

    ls -ld /data/nfs/k8s  

    df -h /data/nfs/k8s  

    sudo du -sh /data/nfs/k8s  

---

### 13.3 Check Kubernetes Node NFS Client  

Execute on Pod node:  

    dpkg -l | grep nfs-common  

If not installed:  

    sudo apt update  

    sudo apt install -y nfs-common  

Check NFS exports:  

    showmount -e 10.0.0.10  

Manual mount test:  

    sudo mkdir -p /mnt/nfs-test  

    sudo mount -t nfs 10.0.0.10:/data/nfs/k8s /mnt/nfs-test  

Write test:  

    echo "nfs mount test from $(hostname)" | sudo tee /mnt/nfs-test/test-$(hostname).txt  

Unmount:  

    sudo umount /mnt/nfs-test  

If manual mount fails, Kubernetes Pod mount likely fails.  

---

### 13.4 Check nfs-subdir-external-provisioner  

Check Pod:  

    kubectl -n storage-system get pods -o wide  

Check logs:  

    kubectl -n storage-system logs deploy/nfs-subdir-external-provisioner --tail=100  

Check Helm configuration:  

    helm get values nfs-subdir-external-provisioner -n storage-system  

Focus on:  

    nfs.server  
    nfs.path  
    storageClass.name  

Common issues:  

    1. nfs.server is incorrect  
    2. nfs.path is incorrect  
    3. NFS Server has not exported the directory  
    4. provisioner Pod cannot mount NFS  
    5. StorageClass provisioner name mismatch  

---

### 13.5 NFS Permission Issues  

Check on NFS Server:  

    ls -ld /data/nfs/k8s  

Common configuration in test environments:  

    sudo chown -R nobody:nogroup /data/nfs/k8s  

    sudo chmod 0777 /data/nfs/k8s  

Production environment note:  

    0777 is not a security best practice.  
    Production should tighten permissions based on business UID/GID, permission model, and security requirements.  
    However, temporary permission testing can help diagnose issues.  

---

## FourteenI don't know.Longhorn PVC Troubleshooting  

### 14.1 Common Longhorn Issues  

Common phenomena related to Longhorn:  

    1. PVC Pending  
    2. Volume creation failure  
    3. Volume cannot be attached  
    4. Volume stuck in Detached / Attaching  
    5. Pod FailedMount  
    6. Replica unhealthy  
    7. open-iscsi not installed  
    8. iscsid not running  
    9. /data/longhorn space insufficient  
    10. Replica count exceeds available nodes  

---

### 14.2 Check Longhorn Components  

Check Pod:  

    kubectl -n longhorn-system get pods -o wide  

Key components:  

    longhorn-manager  
    longhorn-driver-deployer  
    longhorn-csi-plugin  
    csi-attacher  
    csi-provisioner  
    csi-resizer  
    csi-snapshotter  
    instance-manager  

Check Longhorn StorageClass:  

    kubectl get storageclass longhorn -o yaml  

Check Longhorn Volume:  

    kubectl -n longhorn-system get volumes.longhorn.io  

Check Longhorn Node:  

    kubectl -n longhorn-system get nodes.longhorn.io  

Check Longhorn Replica:  

    kubectl -n longhorn-system get replicas.longhorn.io  

---

### 14.3 Check open-iscsi  

Longhorn nodes require iSCSI support.  

Execute on Pod node:  

    dpkg -l | grep open-iscsi  

    systemctl status iscsid --no-pager  

    systemctl status open-iscsi --no-pager  

If not installed:  

    sudo apt update

sudo apt install -y open-iscsi

Start:

    sudo systemctl enable --now iscsid

    sudo systemctl enable --now open-iscsi

Check again:

    systemctl status iscsid --no-pager

---

### 14.4 Check Longhorn Data Directory

Default directory:

    /data/longhorn

Execute on Longhorn storage node:

    ls -ld /data/longhorn

    df -h /data/longhorn

    sudo du -sh /data/longhorn

Common issues:

    1. Directory does not exist
    2. Directory is on system disk with insufficient space
    3. Disk is full
    4. Node is disabled for scheduling
    5. Disk status is abnormal in Longhorn UI

---

### 14.5 Check Volume Status

Check PVC corresponding PV:

    kubectl get pvc <pvc-name> -n <namespace>

    kubectl get pv

After finding PV, check:

    kubectl describe pv <pv-name>

Longhorn Volume name is typically related to PV/PVC.

Check Longhorn Volume:

    kubectl -n longhorn-system get volumes.longhorn.io

Check details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Focus on:

    State
    Robustness
    Current Node
    Replicas
    Conditions
    Events

Common states:

    detached
    attaching
    attached
    faulted
    degraded
    healthy

---

### 14.6 Insufficient Replicas

Longhorn default replica count may be 3.

If only 1 or 2 storage nodes are available, the volume may not reach the expected replica count.

Check StorageClass:

    kubectl get storageclass longhorn -o yaml | grep -i replica -A5

Check Longhorn UI:

    Volume
    Replicas
    Nodes
    Disks

Handling directions:

    1. Add storage nodes
    2. Reduce replica count
    3. Fix abnormal nodes
    4. Clean up disk space
    5. Confirm node and disk scheduling availability

---

### 14.7 Volume Already Attached to Another Node

Common error:

    volume is already exclusively attached to one node and can't be attached to another

Common causes:

    1. RWO volume cannot be mounted on multiple nodes simultaneously
    2. Old Pod not fully deleted
    3. Node anomaly causing volume not to detach properly
    4. Volume still on old node during StatefulSet node migration

Troubleshoot:

    kubectl get pod -A -o wide | grep <pvc-name>

    kubectl get volumeattachment

    kubectl describe volumeattachment <name>

    kubectl -n longhorn-system get volumes.longhorn.io

Handling:

    1. Wait for automatic detachment
    2. Confirm old Pod has been deleted
    3. Confirm old node status
    4. Carefully handle detachment in Longhorn UI
    5. Do not forcibly delete critical business volumes unless risks are clearly understood

---

## Fifteen, CSI Component Troubleshooting

If using CSI storage, check Controller and Node Plugin.

### 15.1 Check CSI Pods

Check all CSI components:

    kubectl get pods -A | grep -i csi

Common namespaces:

    longhorn-system
    kube-system
    rook-ceph
    storage-system

Common components:

    csi-provisioner
    csi-attacher
    csi-resizer
    csi-snapshotter
    csi-node
    csi-plugin

---

### 15.2 Provisioner Abnormality

PVC Pending status often relates to provisioner.

Check logs:

    kubectl -n <namespace> logs <csi-provisioner-pod> --tail=100

Common issues:

    1. Unable to create volume
    2. Backend storage API abnormality
    3. Insufficient permissions
    4. StorageClass parameter error
    5. Insufficient capacity

---

### 15.3 Attacher Abnormality

Pod mounting block storage may require Attach.

Check:

    kubectl get volumeattachment

Check details:

    kubectl describe volumeattachment <name>

Common issues:

    1. Volume cannot be attached to node
    2. Volume already mounted on another node
    3. Node unreachable
    4. CSI attacher abnormality
    5. Backend storage response timeout

---

### 15.4 Node Plugin Abnormality

Pod's node needs CSI Node Plugin to run normally.

Check Pod's node:

    kubectl get pod <pod-name> -n <namespace> -o wide

Check CSI Node Pod on the node:

    kubectl get pods -A -o wide | grep <node-name> | grep -i csi

If the node lacks CSI Node Plugin or it's abnormal, Pod may fail to mount.

---

## Sixteen, StatefulSet PVC Troubleshooting

StatefulSet often creates PVCs automatically via volumeClaimTemplates.

Check StatefulSet:

    kubectl get sts -n <namespace>

View Pod:

    kubectl get pod -n <namespace> -o wide

View PVC:

    kubectl get pvc -n <namespace>

Example:

    mysql-0
    data-mysql-0

Troubleshooting Order:

    1. describe pod mysql-0
    2. describe pvc data-mysql-0
    3. get pv
    4. describe storageclass
    5. Check provisioner / CSI
    6. Check backend storage

Note:

    PVCs created by volumeClaimTemplates in StatefulSet are typically not automatically deleted when the StatefulSet is deleted.
    PVCs may still exist after deleting the StatefulSet.
    This is usually to protect data.

---

## SeventeenI don't know.accessModes Troubleshooting

Common accessModes for PVC:

    ReadWriteOnce
    ReadOnlyMany
    ReadWriteMany
    ReadWriteOncePod

Meaning:

    ReadWriteOnce
        Mounted for read/write on a single node.

    ReadWriteMany
        Mounted for read/write on multiple nodes.

Common NFS Support:

    ReadWriteMany

Common Longhorn Default:

    ReadWriteOnce

Longhorn supports RWX requires additional Share Manager capability.

Common Issues:

    1. Application needs multi-replica shared write, but uses RWO
    2. RWO volume is used by multiple Pods on different nodes simultaneously
    3. Backend storage does not support RWX
    4. StatefulSet replicas incorrectly share the same PVC

View PVC:

    kubectl get pvc <pvc-name> -n <namespace> -o yaml | grep -A5 accessModes

Resolution:

    1. Clarify whether the business requires shared read/write
    2. Use NFS or RWX-supporting storage for RWX scenarios
    3. Avoid multiple Pods sharing the same PVC across nodes for RWO scenarios
    4. Each Pod in StatefulSet should have an independent PVC

---

## EighteenI don't know.volumeMode Troubleshooting

Default volumeMode for PVC:

    Filesystem

It can also be:

    Block

View:

    kubectl get pvc <pvc-name> -n <namespace> -o yaml | grep volumeMode

Common Issues:

    1. Application expects filesystem, but PVC is Block
    2. Application expects block device, but mounted as Filesystem
    3. Storage plugin does not support certain volumeMode

Basic scenarios typically use:

    volumeMode: Filesystem

---

## NineteenI don't know.Secret / ConfigMap Mount Failure

Sometimes Pod mount failure is not related to PVC, but Secret / ConfigMap.

Pod Events may show:

    secret "xxx" not found

    configmap "xxx" not found

Check Pod volumes:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A40 volumes

Check Secret:

    kubectl get secret -n <namespace>

Check ConfigMap:

    kubectl get configmap -n <namespace>

Resolution:

    1. Create missing Secret
    2. Create missing ConfigMap
    3. Correct referenced names
    4. Confirm namespace is correct

Note:

    These issues can also cause Pod to stall in ContainerCreating.
    Do not only check PVC when seeing FailedMount.

---

## TwentyI don't know.hostPath Mount Failure

If Pod uses hostPath:

    hostPath:
      path: /data/app
      type: Directory

Common Issues:

    1. Path does not exist on the node
    2. Type specified as Directory, but actual is not a directory
    3. Insufficient permissions
    4. Pod scheduled to a node without the path
    5. SELinux / AppArmor / security policy restrictions

Check Pod's node:

    kubectl get pod <pod-name> -n <namespace> -o wide

Login to node and check:

    ls -ld /data/app

    df -h /data/app

Resolution:

    1. Create directory on all possible scheduling nodes
    2. Use nodeSelector to fix node
    3. Avoid production business relying on hostPath arbitrarily
    4. Prefer using PVC

---

## Twenty-oneI don't know.Node-level Troubleshooting kubelet

Pod mounting is ultimately executed by node kubelet.

Check logs on Pod's node:

    sudo journalctl -u kubelet -n 200 --no-pager

Real-time view:

    sudo journalctl -u kubelet -f

Filter keywords:

    sudo journalctl -u kubelet --since "1 hour ago" | grep -i "mount"

    sudo journalctl -u kubelet --since "1 hour ago" | grep -i "volume"

    sudo journalctl -u kubelet --since "1 hour ago" | grep -i "csi"

Common information:

    MountVolume.SetUp failed
    Unable to attach or mount volumes
    operationExecutor.MountVolume failed
    rpc error
    timed out waiting for the condition

---

## Twenty-twoI don't know.Node Disk and Inode Troubleshooting

Storage mounting or container startup may also be affected by node disk.

Execute on Pod's node:

    df -h

    df -ih

    du -sh /var/lib/kubelet

du -sh /data/containerd

du -sh /data/longhorn

Common Issues:

    1. Root partition full
    2. inode full
    3. containerd data directory full
    4. Longhorn data directory full
    5. kubelet pod directory abnormal expansion

If node DiskPressure:

    kubectl describe node <node-name>

Continue investigation:

    DiskPressure=True

---

## 23. Post-mount Read/Write Verification

After Pod Running, verify read/write operations.

Enter Pod:

    kubectl exec -it <pod-name> -n <namespace> -- sh

Check mount point:

    df -h

    mount | grep <mount-path>

Write test:

    echo "pvc test $(date)" > /data/test.txt

Read:

    cat /data/test.txt

If data still exists after Pod restart, persistence is basically normal.

Delete Pod and verify recreation:

    kubectl delete pod <pod-name> -n <namespace>

Wait for recreation and check again:

    cat /data/test.txt

---

## 24. Precautions Before Deleting PVC

PVC deletion may lead to data deletion.

First confirm StorageClass reclaimPolicy:

    kubectl get storageclass <storageclass-name> -o yaml | grep reclaimPolicy

Common policies:

    Delete
        PVC deletion may also delete backend volume.

    Retain
        PVC deletion retains PV and backend data, requiring manual handling.

Production environment notes:

    1. Do not arbitrarily delete business PVC
    2. Deleting StatefulSet does not necessarily delete PVC
    3. PVC deletion may trigger backend data deletion
    4. Longhorn / NFS / cloud disk behavior should align with StorageClass
    5. Must confirm backup and business impact before deletion

---

## 25. Quick Troubleshooting Guide

| Phenomenon | Common Causes | Priority Checks |
|---|---|---|
| PVC Pending | StorageClass does not exist | kubectl get storageclass |
| PVC Pending | provisioner anomaly | provisioner / CSI logs |
| Pod Pending | PVC not Bound | kubectl get pvc |
| Pod ContainerCreating | Mount failure | describe pod Events |
| FailedMount | NFS unreachable or permission issues | showmount / mount |
| FailedAttachVolume | Block storage Attach anomaly | volumeattachment |
| Longhorn PVC anomaly | open-iscsi not installed | iscsid status |
| Longhorn volume anomaly | Insufficient replicas or disk full | Longhorn Volume / Node |
| Secret not found | Secret reference error | kubectl get secret |
| ConfigMap not found | ConfigMap reference error | kubectl get cm |
| RWO multi-node mount failure | Volume already mounted on other node | volumeattachment / Pod distribution |
| hostPath failure | Node path does not exist | Check path on node |

---

## 26. Standard Troubleshooting Command List

### 26.1 Pod and PVC

    kubectl get pod -n <namespace> -o wide

    kubectl describe pod <pod-name> -n <namespace>

    kubectl get pvc -n <namespace>

    kubectl describe pvc <pvc-name> -n <namespace>

---

### 26.2 PV and StorageClass

    kubectl get pv

    kubectl describe pv <pv-name>

    kubectl get storageclass

    kubectl describe storageclass <storageclass-name>

---

### 26.3 CSI Components

    kubectl get pods -A | grep -i csi

    kubectl get volumeattachment

    kubectl describe volumeattachment <name>

---

### 26.4 NFS

    showmount -e 10.0.0.10

    sudo mount -t nfs 10.0.0.10:/data/nfs/k8s /mnt/nfs-test

    kubectl -n storage-system get pods -o wide

    kubectl -n storage-system logs deploy/nfs-subdir-external-provisioner --tail=100

---

### 26.5 Longhorn

    kubectl -n longhorn-system get pods -o wide

    kubectl -n longhorn-system get volumes.longhorn.io

    kubectl -n longhorn-system get nodes.longhorn.io

    kubectl -n longhorn-system get replicas.longhorn.io

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=100

---

### 26.6 Node Native

    systemctl status kubelet --no-pager

    journalctl -u kubelet -n 200 --no-pager

    df -h

    df -ih

    systemctl status iscsid --no-pager

dpkg -l | grep nfs-common

dpkg -l | grep open-iscsi

---

## 27. Recommended Troubleshooting Path

### 27.1 PVC Always Pending

Execution order:

    kubectl get pvc -n <namespace>

    kubectl describe pvc <pvc-name> -n <namespace>

    kubectl get storageclass

    kubectl describe storageclass <storageclass-name>

    Check provisioner / CSI components

    Check backend storage status

---

### 27.2 Pod Always ContainerCreating

Execution order:

    kubectl describe pod <pod-name> -n <namespace>

    kubectl get pvc -n <namespace>

    kubectl get pv

    Check Pod's node

    Check kubelet logs on the node

    Check CSI Node Plugin / NFS / iSCSI

---

### 27.3 NFS Mount Failure

Execution order:

    kubectl describe pod <pod-name> -n <namespace>

    showmount -e <nfs-server>

    Manually mount on the Pod's node

    Check nfs-common

    Check NFS Server /etc/exports

    Check NFS directory permissions

---

### 27.4 Longhorn Mount Failure

Execution order:

    kubectl describe pod <pod-name> -n <namespace>

    kubectl -n longhorn-system get pods -o wide

    kubectl -n longhorn-system get volumes.longhorn.io

    kubectl get volumeattachment

    Check open-iscsi / iscsid on the Pod's node

    Check /data/longhorn space

---

## 28. Handling Recommendations

PVC/Storage Troubleshooting Recommendations:

    1. First check Pod Events
    2. Then check PVC Events
    3. For PVC Pending, check StorageClass and provisioner
    4. For PVC Bound but Pod failing to start, check mounting and kubelet
    5. For NFS issues, manually mount on the node to verify
    6. For Longhorn issues, check open-iscsi and Volume status
    7. For StatefulSet issues, check both Pod and corresponding PVC
    8. Do not delete PVCs arbitrarily
    9. Do not delete PVs arbitrarily
    10. Do not clean storage directories without confirming backend data
    11. Before handling storage issues, confirm business impact and data backup

---

## 29. Summary

The core of PVC mounting failure troubleshooting is to distinguish between two phases:

    1. Binding phase:
        Can PVC bind to PV

    2. Mounting phase:
        Can Pod mount the bound volume to the node and container

If PVC is Pending:

    Prioritize checking:
        1. StorageClass
        2. provisioner
        3. Backend storage
        4. accessModes
        5. volumeBindingMode

If PVC is Bound but Pod FailedMount:

    Prioritize checking:
        1. Pod Events
        2. kubelet logs
        3. CSI Node Plugin
        4. NFS client and server
        5. Longhorn open-iscsi / Volume status
        6. Node disk and network

Most important commands:

    kubectl describe pod <pod-name> -n <namespace>

    kubectl describe pvc <pvc-name> -n <namespace>

    kubectl get pv

    kubectl describe storageclass <storageclass-name>

Experience-based judgment:

    1. pod has unbound immediate PVC: check PVC first
    2. FailedMount: check Pod Events and node kubelet first
    3. NFS access denied: check exports and directory permissions first
    4. Longhorn attach failure: check open-iscsi, Volume, and VolumeAttachment first
    5. RWO multi-node mount failure: check if the volume is already mounted on other nodes
    6. Storage troubleshooting must be cautious to avoid accidental deletion of business data