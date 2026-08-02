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
    #Persistent Storage
    #Cluster Basic Troubleshooting

---

## I. Document Description

This document records the basic troubleshooting methods for PVC mount failures in Kubernetes, Pod startup issues due to storage problems, and abnormal PV/PVC binding.

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

    1. PVC remains Pending
    2. Pod remains Pending
    3. Pod gets stuck at ContainerCreating
    4. FailedMount appears in Pod Events
    5. FailedAttachVolume appears in Pod Events
    6. PV cannot be Bound
    7. PV is in the Released/Failed state
    8. StorageClass does not exist
    9. CSI Controller/Node Plugin errors
    10. NFS cannot be mounted
    11. Longhorn Volume cannot be attached
    12. StatefulSet Pods cannot be created

Objectives of This Document:

    1. Understand the relationships between PVC, PV, StorageClass, and CSI.
    2. Be able to determine whether a PVC binding or mount failure has occurred.
    3. Troubleshoot PVC Pending issues.
    4. Troubleshoot FailedMount in Pods.
    5. Identify problems with non-existent or incorrectly configured StorageClasses.
    6. Investigate NFS dynamic provisioning issues.
    7. Diagnose Longhorn volume mounting problems.
    8. Check StatefulSet volumeClaimTemplates for errors.
    9. Establish a standard storage troubleshooting process.

Applicable Scenarios:

    1. kubeadm self-built clusters
    2. Private Kubernetes clusters
    3. NFS StorageClass configurations
    4. Longhorn StorageClass implementations
    5. CSI storage plugins
    6. Stateful application mount failures
    7. Pod Pending/ContainerCreating issues related to PVC

---

## II. PVC Access Chain

Basic persistence storage chain in Kubernetes:

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
    StorageClass/CSI Provisioner
     |
     v
    Backend Storage
        NFS
        Longhorn
        Ceph
        Cloud Disk
        Local Disk
        Commercial Storage

Dynamic Provisioning Process:

    User creates a PVC
       |
       v
    StorageClass finds the provisioner
       |
       v
    Provisioner creates backend storage
       |
       v
    PV is created
       |
       v
    PVC is Bound
       |
       v
    Pod is scheduled and PVC is mounted
       |
       v
    kubelet calls CSI/mount tools to complete the mounting

Key Points:

    Before PVC is Bound, the Pod may remain Pending.
    Even after PVC is Bound, the Pod might still get stuck at ContainerCreating due to mount failures.
    PVC being Bound does not guarantee successful mounting of the Pod.
    A Running Pod does not necessarily mean normal storage read/write; further verification within the container is required.

---

## III. Basic Relationships Between PVC/PV/StorageClass

| Object | Function |
|---|---|
| StorageClass | Defines the dynamic provisioning method, e.g., nfs-client, longhorn |
| PVC | Represents the user's request for storage resources |
| PV | The actual persistent volume object in Kubernetes |
| CSI Controller | Responsible for creating, deleting, attaching, and detaching volumes |
| CSI Node Plugin | Handles volume mounting and unmounting at the node level |
| Pod | Uses storage through the PVC |
| kubelet | Executes mount operations and starts containers on nodes |

Simple Explanation:

    StorageClass specifies how storage will be provided.
    PVC indicates how much storage the user requires.
    PV represents the actual storage allocated.
    The Pod determines where the stored data should be mounted.

---

## IV. Common Status Descriptions

### 4.1 PVC Pending

Indicates that the PVC has not yet been bound to a PV.

Common Causes:

    1. StorageClass does not exist.
    2. StorageClass provisioner isThe pod contains unbound PersistentVolumeClaims, which is causing issues with mounting volumes. Attempts to mount or attach volumes have failed, including errors such as "FailedMount" and "Unable to attach or mount volumes." Additionally, there were timeouts while waiting for the necessary conditions to be met.

Specific errors include:
- The persistentvolumeclaim "xxx" was not found.
- The volume is already exclusively attached to one node and cannot be attached to another.
- RPC errors occurred with code "Internal desc".
- Access was denied by the server when trying to mount using NFS.
- The connection timed out during the mounting process.

To diagnose and resolve these storage-related issues, you should follow these steps:

1. Check the Pod Events to identify the specific causes of the problems.
2. Ensure that the PersistentVolumeClaims are properly bound to the corresponding volumes.
3. Verify that the volumes are not already in use by another Pod or node.
4. Check the status of the PVs and StorageClasses to ensure they are correct and available for use.
5. If necessary, install or configure the required StorageClasses, modify PVC settings, or repair any related components.

By carefully analyzing these logs and error messages, you should be able to locate and fix the underlying issues that are preventing the Pod from starting successfully.```markdown
mount.nfs: Connection timed out
mount.nfs: No such file or directory
bad option; for several filesystems you might need a /sbin/mount.<type> helper program
permission denied
stale file handle

Common reasons:

1. The node does not have nfs-common installed.
2. The NFS Server is unreachable.
3. The NFS shared directory has not been exported.
4. The permissions in /etc/exports do not allow the current node to access it.
5. The NFS directory does not exist.
6. The permissions on the NFS directory are insufficient.
7. The firewall is blocking port 2049.
8. The NFS Server is experiencing an error.

---

### 13.2 Checking the NFS Server

On the NFS Server, execute the following commands:

```bash
systemctl status nfs-server --no-pager
sudo exportfs -v
showmount -e 127.0.0.1
sudo ss -lntup | grep -E "2049|111"
```

Check the shared directory:

```bash
ls -ld /data/nfs/k8s
df -h /data/nfs/k8s
sudo du -sh /data/nfs/k8s
```

---

### 13.3 Checking the NFS Client on the Kubernetes Node

On the node where the Pod is located, execute the following commands:

```bash
dpkg -l | grep nfs-common
```

If it is not installed, install it:

```bash
sudo apt update
sudo apt install -y nfs-common
```

Check the NFS exports:

```bash
showmount -e 10.0.0.10
```

Perform a manual mount test:

```bash
sudo mkdir -p /mnt/nfs-test
sudo mount -t nfs 10.0.0.10:/data/nfs/k8s /mnt/nfs-test
echo "nfs mount test from $(hostname)" | sudo tee /mnt/nfs-test/test-$(hostname).txt
sudo umount /mnt/nfs-test
```

If the manual mount fails, it is likely that the Kubernetes Pod will also fail to mount.

---

### 13.4 Checking nfs-subdir-external-provisioner

View the Pod:

```bash
kubectl -n storage-system get pods -o wide
```

Check the logs:

```bash
kubectl -n storage-system logs deploy/nfs-subdir-external-provisioner --tail=100
```

View the Helm configuration:

```bash
helm get values nfs-subdir-external-provisioner -n storage-system
```

Pay special attention to the following settings:

```bash
nfs.server
nfs.path
storageClass.name
```

Common issues:

1. The value of nfs.server is incorrect.
2. The value of nfs.path is incorrect.
3. The corresponding directory on the NFS Server has not been exported.
4. The provisioner Pod cannot mount the NFS volume.
5. The name of the StorageClass provisioner does not match.

---

### 13.5 NFS Permission Issues

On the NFS Server, check:

```bash
ls -ld /data/nfs/k8s
```

Common configurations in experimental environments:

```bash
sudo chown -R nobody:nogroup /data/nfs/k8s
sudo chmod 0777 /data/nfs/k8s
```

In production environments, note that setting permissions to 0777 is not considered a secure practice. In production, permissions should be set according to the business's UID/GID, permission models, and security requirements. However, during troubleshooting, this can be used as a temporary indicator to determine if permissions are the cause of the issue.

---

## Chapter XIV: Troubleshooting Longhorn PVCs

### 14.1 Common Issues with Longhorn

Common phenomena related to Longhorn include:

1. PVC Pending
2. Volume creation failure
3. Volume unable to be attached
4. Volume stuck in the Detached/Attaching state
5. Pod FailedMount
6. Replicas unhealthy
7. open-iscsi not installed
8. iscsid not running
9. Insufficient space in /data/longhorn
10. Number of replicas exceeding the number of available nodes

---

### 14.2 Checking Longhorn Components

View the Pods:

```bash
kubectl -n longhorn-system get pods -o wide
```

Key components include:

```bash
longhorn-manager
longhorn-driver-deployer
longhorn-csi-plugin
csi-attacher
csi-provisioner
csi-resizer
lsi-snapshotter
instance-manager
```

View the Longhorn StorageClass:

```bash
kubectl get storageclass longhorn -o yaml
```

View the Longhorn Volumes:

```bash
```bash
kubectl get volumeattachment

kubectl describe volumeattachment <name>

kubectl -n longhorn-system get volumes.longhorn.io
```du -sh /data/longhorn

Common Issues:

1. Root partition is full
2. Inode space is exhausted
3. containerd data directory is full
4. Longhorn data directory is full
5. kubelet pod directory is abnormally expanding

If the node has DiskPressure:

kubectl describe node <node-name>

Further investigation:

DiskPressure=True

---

## Section 23: Read/Write Verification After Successful Mounting

After the Pod starts running, it is necessary to verify its read and write capabilities.

Enter the Pod:

kubectl exec -it <pod-name> -n <namespace> -- sh

Check the mount point:

df -h

mount | grep <mount-path>

Perform a write test:

echo "pvc test $(date)" > /data/test.txt

Read the data:

cat /data/test.txt

If the data persists after the Pod restarts, it indicates that persistence is functioning correctly.

Verify again after deleting and recreating the Pod:

kubectl delete pod <pod-name> -n <namespace>

Check again after recreation:

cat /data/test.txt

---

## Section 24: Precautions Before Deleting a PVC

Deleting a PVC may result in data loss.

First, confirm the reclaimPolicy of the StorageClass:

kubectl get storageclass <storageclass-name> -o yaml | grep reclaimPolicy

Common policies include:

Delete
        After the PVC is deleted, the underlying volume may also be removed.

Retain
        The PV and underlying data are retained after the PVC is deleted, requiring manual intervention.

Important notes for production environments:

1. Do not delete business-related PVCs arbitrarily.
2. Deleting a StatefulSet does not necessarily mean deleting the associated PVCs.
3. Deleting a PVC may trigger the deletion of underlying data.
4. The behavior of Longhorn, NFS, and cloud disks depends on the configured StorageClass.
5. Always ensure backup and assess potential impacts before deleting any PVC.

---

## Section 25: Quick Reference for Common Issues

| Issue | Possible Causes | Priority Check Items |
|---|---|---|
| PVC Pending | Missing StorageClass | kubectl get storageclass |
| PVC Pending | Provisioner error | Check provisioner/CSI logs |
| Pod Pending | PVC not bound | kubectl get pvc |
| ContainerCreating Failed in Pod | Mounting issue | Check pod Events |
| FailedMount | NFS connection issues or permissions | Showmount/mount commands |
| FailedAttachVolume | Block storage attachment errors | volumeattachment checks |
| Longhorn PVC-related issues | open-iscsi not installed | iscsid status check |
| Longhorn volume problems | Insufficient replicas or full disk space | Longhorn Volume/Node checks |
| Secret not found | Incorrect Secret reference | kubectl get secret |
| ConfigMap not found | Incorrect ConfigMap reference | kubectl get cm |
| RWO multi-node mounting failure | Volume already mounted on another node | volumeattachment/Pod distribution checks |
| hostPath failure | Node path does not exist | Verify path on the node |

---

## Section 26: Standard Troubleshooting Commands

### 26.1 Pod and PVC Operations

    kubectl get pod -n <namespace> -o wide

    kubectl describe pod <pod-name> -n <namespace>

    kubectl get pvc -n <namespace>

    kubectl describe pvc <pvc-name> -n <namespace>

---

### 26.2 PV and StorageClass Operations

    kubectl get pv

    kubectl describe pv <pv-name>

    kubectl get storageclass

    kubectl describe storageclass <storageclass-name>

---

### 26.3 CSI Component Checks

    kubectl get pods -A | grep -i csi

    kubectl get volumeattachment

    kubectl describe volumeattachment <name>

---

### 26.4 NFS Operations

    showmount -e 10.0.0.10

    sudo mount -t nfs 10.0.0.10:/data/nfs/k8s /mnt/nfs-test

    kubectl -n storage-system get pods -o wide

    kubectl -n storage-system logs deploy/nfs-subdir-external-provisioner --tail=100

---

### 26.5 Longhorn Operations

    kubectl -n longhorn-system get pods -o wide

    kubectl -n longhorn-system get volumes.longhorn.io

    kubectl -n longhorn-system get nodes.longhorn.io

    kubectl -n longhorn-system get replicas.longhorn.io

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=100

---

### 26.6 Node Local Checks

    systemctl status kubelet --no-pager

    journalctl -u kubelet -n 200 --no-pager

    df -h

    df -ihCan a PVC be Bound to a PV?

2. Mounting Phase:
    Can a Pod mount the bound volume onto nodes and containers?

If the PVC is in a Pending state:

    Check these priorities first:
    1. StorageClass
    2. Provisioner
    3. Backend storage
    4. accessModes
    5. volumeBindingMode

If the PVC is Bound but the Pod fails to mount it:

    Check these priorities first:
    1. Pod Events
    2. kubelet logs
    3. CSI Node Plugin
    4. NFS Client and Server
    5. Longhorn open-iscsi / Volume status
    6. Node disk and network configuration

Most important commands:

    kubectl describe pod <pod-name> -n <namespace>

    kubectl describe pvc <pvc-name> -n <namespace>

    kubectl get pv

    kubectl describe storageclass <storageclass-name>

Practical tips for troubleshooting:

    1. If a Pod has an unbound PVC, check the PVC first.
    2. In case of FailedMount, start by checking Pod Events and the node's kubelet logs.
    3. For NFS access issues, verify exports and directory permissions.
    4. If Longhorn attachment fails, inspect open-iscsi, Volume, and VolumeAttachment settings.
    5. When encountering multi-node mounting failures with RWO, check if the volume is already mounted on other nodes.
    6. Be cautious when troubleshooting storage-related issues to avoid accidentally deleting critical business data.