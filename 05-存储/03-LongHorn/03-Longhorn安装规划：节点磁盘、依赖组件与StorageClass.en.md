# Longhorn Installation Planning: Node Disks, Dependent Components, and StorageClass

Recommended Path: 05-Storage/03-LongHorn/03-Longhorn Installation Planning: Node Disks, Dependent Components, and StorageClass.md

Tags: #Longhorn #Kubernetes #CSI #StorageClass #PV #PVC #open-iscsi #NFS #Block Storage #Node Planning #Disk Planning #Advanced SRE #Production Operations

---

## I. Document Description

This article is the third part of the Longhorn module, focusing on the environmental planning and checks required before installing Longhorn.

What has been covered previously includes:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager

This article does not directly perform the installation of Longhorn but instead addresses the necessary preparations before a production-level installation:

    Whether the Kubernetes cluster is healthy
    Whether the node system meets the requirements
    Whether open-iscsi / iscsid are installed and running
    Whether the NFSv4 client is installed
    How to plan the data disks and directories
    Whether Longhorn data might accidentally overwrite the system disk
    Whether the number of nodes supports the required replica distribution
    How to design the StorageClass
    Whether to set longhorn as the default StorageClass
    What check records should be retained before installation

Longhorn is a distributed block storage system within Kubernetes. The planning of nodes, disks, dependent components, and StorageClass before installation will directly affect subsequent operations such as PVC, Volume, Replica, Backup, Restore, and disaster recovery capabilities.

In production environments, it is not recommended to simply perform "helm install" without conducting these preliminary checks first.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Identify the necessary checks before installing Longhorn.
2. Verify the status of Kubernetes cluster nodes.
3. Check the status of kube-system components.
4. Assess the current configuration of StorageClass, PVs, and PVCs.
5. Verify the node system version, kernel, containerd, and kubelet status.
6. Install open-iscsi and nfs-common on Ubuntu 22.04.
7. Install iscsi-initiator-utils and nfs-utils on Rocky Linux 9.
8. Ensure that the iscsid service is running correctly.
9. Plan the Longhorn data directory.
10. Determine whether /data/longhorn is actually mounted on an independent data disk.
11. Understand the relationship between the number of replicas, nodes, and disk capacity.
12. Design the basic parameters for Longhorn StorageClass.
13. Decide whether to set longhorn as the default StorageClass.
14. Generate a pre-installation check report.
15. Prepare for subsequent Helm-based installations of Longhorn.

---

## III. Experimental Environment Planning

### 3.1 Current Kubernetes Cluster

Default experimental environment:

    Kubernetes: Kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node IP Range: 10.0.0.0/24

Node Planning:

| IP | Host Name | Role | Description |
|---|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane | Control plane node |
| 10.0.0.21 | k8s-worker01 | Worker | Longhorn data node |
| 10.0.0.22 | k8s-worker02 | Worker | Longhorn data node |

---

### 3.2 Longhorn Data Node Planning

Experimental environment recommendations:

| Node | Whether to Participate in Longhorn | Data Directory | Description |
|---|---|---|---|
| k8s-master01 | Optional | /data/longhorn | Can be used for experiments; cautious in production |
| k8s-worker01 | Yes | /data/longhorn | Recommended |
| k8s-worker02 | Yes | /data/longhorn | Recommended |

Production recommendations:

    Preferentially use Worker nodes for Longhorn data storage.
    Whether Control Plane nodes participate in Longhorn should depend on the cluster size, resource constraints, and reliability requirements.
    In production environments, it is recommended to have at least 3 available data nodes to support 3 replicas.
    If there are only 2 Worker nodes, consider using 2 replicas or involving the Master node in the experiment.
    Avoid concentrating all replicas on a single node.
    Do not place the Longhorn data directory on the system disk before deploying it in production.

---

### 3.3 Longhorn Data Directory

The common default location for Longhorn data directories is:

    /var/lib/longhorn

In this plan, we will use:

    /data```markdown
kubectl get events -A --sort-by=.lastTimestamp | tail -100

Focus on:

FailedScheduling
FailedMount
FailedAttachVolume
ImagePullBackOff
NodeNotReady
DiskPressure
MemoryPressure
NetworkUnavailable

If there are many FailedMount or storage-related errors, address these storage issues first.

---

## Section 6: Practical Operation 2: Checking Current Storage Resources

### 6.1 Viewing StorageClasses

Execute:

kubectl get storageclass

or:

kubectl get sc

Possible Scenario 1: No StorageClass found.

"No resources found"

Explanation:

The current cluster does not have the capability to provide dynamic storage.
After installing Longhorn, a new longhorn StorageClass will be added.

Possible Scenario 2: Other StorageClasses exist.

For example:

local-path
nfs-client
rook-ceph-block
longhorn

It is necessary to confirm:

- Whether there is a default StorageClass already in use.
- If other storage systems are currently being used by the business.
- Whether Longhorn should be set as the default StorageClass in the future.

---

### 6.2 Viewing the Default StorageClass

Execute:

kubectl get sc

If you see:

“(default)”

it indicates that this StorageClass is the default one.

View details:

kubectl describe sc <storage-class-name>

Focus on:

Provisioner
ReclaimPolicy
VolumeBindingMode
AllowVolumeExpansion
Parameters

Production Recommendation:

When multiple storage systems coexist, it is not recommended to rely entirely on the default StorageClass.
It is best to explicitly specify the storageClassName for business PVCs.
The decision to set Longhorn as the default StorageClass should be made before installation.

---

### 6.3 Viewing PVs

Execute:

kubectl get pv

If there are PVs already, check:

- Whether any services are currently using them.
- If they belong to other storage systems.
- Their status (Released/Failed).
- If there are any remaining unused PVs.

View details:

kubectl describe pv <pv-name>

---

### 6.4 Viewing PVCs

Execute:

kubectl get pvc -A

Focus on:

- Whether any production PVCs exist.
- Any Pending PVCs.
- PVCs that use the default StorageClass.
- PVCs that are long-term Bound but not clearly used by any service.

View details:

kubectl describe pvc <pvc-name> -n <namespace>

---

## Section 7: Practical Operation 3: Checking Basic Node System Information

The following commands need to be executed on all nodes planned to participate in Longhorn, including:

k8s-master01
k8s-worker01
k8s-worker02

### 7.1 Viewing System Versions

Execute:

hostname
cat /etc/os-release
uname -a

Record:

- Operating system version.
- Kernel version.
- Hostname.
- Node IP address.

---

### 7.2 Checking the Container Runtime

Execute:

systemctl status containerd
containerd --version
crictl info | head -50

If crictl is not available, check first:

which crictl

Production Note:

It is not recommended to directly modify the underlying configuration of containerd just to resolve Longhorn's image pulling issues.
Image-related problems should be prioritized by specifying the image repository through Helm values.yaml files.
Image management will be covered in detail in the 04-Longhorn Helm Installation Methodology section.

---

### 7.3 Checking the kubelet Status

Execute:

systemctl status kubelet
journalctl -u kubelet --since "30 minutes ago" | tail -100

Focus on:

- Whether kubelet is active.
- Any FailedMount events.
- Any CSI-related errors.
- Any NodeNotReady issues.
- Any disk pressure indications.

---

### 7.4 Checking Node Resources

Execute:

free -h
uptime
df -hT
lsblk
mount

Focus on:

- Whether the system disk is nearly full.
- The presence of any dedicated data disks.
- Whether /data is mounted on a separate disk.
- The type of file system in use.
- Any signs of abnormal node load.
- Whether memory levels are too low.

---

### 7.5 Checking Time Synchronization

Execute:

timedatectl

Focus on:

- Whether the System clock synchronization is set to "yes".
- Whether the time zone is correct.
- Whether the NTP service is active.

Inconsistent times can lead to:

- Certificate issues.
- Confused log timestamps.
- Difficulties in troubleshooting distributed systems.
- Inaccurate monitoring data.

---

## Section 8: Practical Operation 4: Installing and Checking open-iscsi

### 8.1 Why open-iscsi is Needed

Longhorn v1 volume mounting relies on the node's iSCSI capabilities.

If a node does not have open-iscsi or if iscsid is### 9.3 Installing nfs-utils on Rocky Linux 9

Execute:

    dnf install -y nfs-utils

Check:

    which mount.nfs
    mount.nfs -V

---

### 9.4 Checking Kernel NFS Support

Execute:

    cat /boot/config-$(uname -r) | grep CONFIG_NFS_V4

If the file does not exist, you can try:

    zcat /proc/config.gz | grep CONFIG_NFS_V4

Note:

    The path to the kernel configuration file may vary depending on the distribution.
    The absence of this command does not necessarily mean that support is unavailable; it should be checked based on the actual system configuration.

---

## Section Ten: Practical Exercise Six: Planning and Preparing Longhorn Data Directories

### 10.1 Creating a Data Directory

Execute on each node planned to participate in Longhorn:

    mkdir -p /data/longhorn

Check:

    ls -ld /data/longhorn

---

### 10.2 Checking the Disk Where the Directory is Located

Execute:

    df -hT /data/longhorn

If the output looks like this:

    Filesystem     Type  Size  Used Avail Use% Mounted on
    /dev/sda2      ext4   50G   20G   30G  40% /

It means that /data/longhorn is located on the system disk. This setup is suitable only for learning and experimentation and is not recommended for production use.

If the output looks like this:

    /dev/sdb1      xfs   500G  10G  490G   2% /data

It means that /data/longhorn is located on an independent data disk or under a /data mount point, which is more suitable for production use.

---

### 10.3 Checking the Disk Structure

Execute:

    lsblk -f

Pay attention to:

    Which disk is the system disk.
    Which disk is the data disk.
    Whether the data disk is formatted.
    Whether the data disk is mounted.
    What the file system type is.
    Whether the mount point is stable.

---

### 10.4 Example of Mounting an Independent Data Disk

High-risk warning:

    The following formatting commands will erase all data on the disk. They should only be used on new disks or experimental disks where no data exists. Before using them in production, make sure you know the device name and have someone else review the process. Do not accidentally format the system disk.

Assume a new data disk is /dev/sdb:

Check the disk:

    lsblk

Example of creating a file system:

    mkfs.xfs /dev/sdb

Create a mount directory:

    mkdir -p /data

Get the UUID:

    blkid /dev/sdb

Edit /etc/fstab:

    vi /etc/fstab

Add content similar to this:

    UUID=<replace with the actual UUID> /data xfs defaults,noatime 0 0

Mount the disk:

    mount -a

Check:

    df -hT /data
    lsblk -f

Create the Longhorn data directory:

    mkdir -p /data/longhorn

Check:

    df -hT /data/longhorn

---

### 10.5 Example Using ext4

If you want to use ext4:

    mkfs.ext4 /dev/sdb

Example of fstab:

    UUID=<replace with the actual UUID> /data ext4 defaults,noatime 0 0

Check:

    mount -a
    df -hT /data

---

### 10.6 Checking Directory Permissions

Execute:

    ls -ld /data/longhorn

In most cases, root ownership is sufficient.

Do not arbitrarily change permissions to 777 for the /data/longhorn directory. In production, it is not recommended to use such drastic permission settings to resolve mounting issues. If you encounter permission-related errors after installing Longhorn, check the Longhorn logs, Pod security context, and node directory status to troubleshoot.

---

## Section Eleven: Planning the Number of Replicas and Capacity

### 11.1 The Meaning of the Number of Replicas

Longhorn Volume allows you to configure the number of replicas.

For example:

    numberOfReplicas: 3

This means that:

    A Volume will attempt to create 3 replicas.
    The replicas will be distributed across different nodes and disks.
    If one node or replica fails, other replicas will still be available.

---

### 11.2 Matching the Number of Replicas with the Number of Nodes

The number of replicas should match the number of nodes.

| Number of Nodes | Recommended Number of Replicas | Notes |
|-----------------|-----------------------------------|-----------|
| 1 node           | 1                                   | Suitable only for learningSuggestions:

During the learning phase, it is advisable to set defaults for convenience in experiments. In production scenarios, it is recommended to explicitly specify the `storageClassName` for business PVCs. When multiple storage systems coexist, it is not recommended to easily set Longhorn as the default.

Examples for setting defaults:

To set the default value:

```bash
kubectl patch storageclass longhorn \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

To unset the default value:

```bash
kubectl patch storageclass longhorn \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

---

### 13.3 Planning ReclaimPolicy

The `ReclaimPolicy` determines how the PV and its underlying data are handled after the PVC is deleted.

Common options:

- **Delete**: After the PVC is deleted, both the PV and the associated Longhorn Volume may be removed. This option is suitable for temporary environments or situations where data can be recreated. The risk is that accidental deletion of a PVC may result in data loss.
- **Retain**: After the PVC is deleted, both the PV and the underlying data are retained. This option is more secure but requires manual cleanup, making it suitable for handling important data.

Production recommendations:

- Use the `Delete` option with caution for critical business applications. For important data, consider using the `Retain` option or establishing a clear deletion approval process to prevent accidental removal of PVCs by ordinary users.

---

### 13.4 Planning AllowVolumeExpansion

The `AllowVolumeExpansion` parameter determines whether it is allowed to expand the size of a PVC.

Suggestions:

- Enable volume expansion capabilities.
- Verify the underlying disk capacity before expanding.
- Check whether the file system has successfully expanded after the operation.
- Do not simply expand the PVC without confirming the status of both the Longhorn Volume and the node disks.

Verification method:

```bash
kubectl describe sc longhorn | grep -i AllowVolumeExpansion
```

---

### 13.5 Planning VolumeBindingMode

Common modes include:

- **Immediate**: The PV and the underlying Volume are created immediately after the PVC is created.
- **WaitForFirstConsumer**: The Volume is bound only when a Pod uses the PVC and it is scheduled.

The default behavior of Longhorn depends on the actual StorageClass configuration installed later. In production scenarios, it is important to understand how the binding timing affects scheduling and replica distribution, especially for distributed volumes like Longhorn. The final setting should be based on the specific parameters defined by the Longhorn Chart and the StorageClass.

---

### 13.6 Planning the Number of Replicas

The number of replicas can be defined through parameters for a Longhorn StorageClass.

Example parameter configuration:

```bash
numberOfReplicas: "2"
```

If there are only two Worker nodes in the current experimental environment, it is recommended to set `numberOfReplicas: "2"`. If there are three data nodes, a value of `numberOfReplicas: "3"` can be considered for production or more comprehensive experiments.

Note:

- Having more replicas does not always mean better performance. An increased number of replicas leads to greater write amplification and higher capacity usage. If the number of nodes is insufficient, replicas may not be distributed effectively.

---

## Section 14: Generating a Pre-Installation Check Report

### 14.1 Creating a Check Script

You can create a simple check script on the management node:

```bash
cat > longhorn-precheck.sh <<'EOF'
#!/bin/bash

set -euo pipefail

REPORT="longhorn-precheck-$(date +%F-%H%M%S).log"

{
  echo "===== Longhorn Precheck Report ====="
  echo "Time: $(date)"
  echo

  echo "===== 1. Kubernetes Nodes ====="
  kubectl get nodes -o wide || true
  echo

  echo "===== 2. kube-system Pods ====="
  kubectl get pods -n kube-system -o wide || true
  echo

  echo "===== 3. StorageClass ====="
  kubectl get sc || true
  echo

  echo "===== 4. PV ====="
  kubectl get pv || true
  echo

  echo "===== 5. PVC All Namespaces ====="
  kubectl get pvc -A || true
  echo

  echo "===== 6. Recent Events ====="
  kubectl get events -A --sort-by=.lastTimestamp | tail -100 || true
  echo

  echo "===== 7. Node Pressure ====="
  kubectl describe nodes | grep -E "Name:|DiskPressure|MemoryPressure|PIDPressure|Ready" || true
  echo

  echo "===== Report Finished ====="
} > "${REPORT}" 2>&1

echo "Report saved to ${REPORT}"
EOF
``If "installation conditions are not met," the following issues should be addressed first:

    Node NotReady
    DiskPressure
    iscsid not running
    Data disk not mounted
    Insufficient system disk space
    Abnormalities in kube-system components
    Storage class conflicts or unspecified default StorageClass

---

## Section Sixteen: Special Precautions Before Production Installation

### 16.1 Do Not Damage the Underlying Containerd Runtime

If the Longhorn image retrieval fails, do not immediately modify the global containerd configuration.

It is recommended to:

    First, check the Longhorn Chart.
    Then, examine the values.yaml file.
    Next, retrieve the list of images.
    Synchronize the images to a domestic repository.
    Use Helm's values.yaml file to specify the image repository.
    Finally, proceed with the installation.

This approach is more controllable and will not affect the overall operation of the Kubernetes cluster.

---

### 16.2 Do Not Write Longhorn Data to the System Disk

Filling up the system disk may lead to:

    Abnormalities in kubelet and containerd.
    DiskPressure on nodes.
    Pod eviction.
    Issues with Longhorn replicas.
    Volume degradation.
    Inability to rebuild replicas.

In production, ensure that:

    `df -hT /data/longhorn` shows that /data/longhorn is located on the expected data disk.

---

### 16.3 Do Not Ignore the Backup Target

Longhorn replicas are not intended for backup purposes.

Plan in advance:

    Whether to use NFS or S3 as the Backup Target.
    Whether the backup target should be isolated from the K8s cluster.
    Whether regular backups will be performed.
    Whether recovery tests have been conducted.
    How to monitor for backup failures.

More details on backup and restoration will be provided in section 07-Longhorn Backup and Recovery.

---

### 16.4 Do Not Blindly Set the Default StorageClass

If Longhorn is the only storage system, setting it as the default may be acceptable.

However, if the cluster already has:

    NFS StorageClass
    local-path
    Ceph CSI
    Cloud vendor cloud disk CSI

It is recommended to explicitly specify `storageClassName: longhorn` in business PVC configurations to avoid misusing the default storage.

---

## Section Seventeen: Common Issues and Troubleshooting

### 17.1 Node Lacks iscsid

Symptoms:

    Pod mounting of Longhorn PVC fails.
    kubelet events display iSCSI-related errors.
    The `iscsiadm` command is not found.

Troubleshooting:

    `which iscsiadm`
    `systemctl status iscsid`

Resolution:

    For Ubuntu:
    `apt update`
    `apt install -y open-iscsi`
    `systemctl enable --now iscsid`

    For Rocky Linux 9:
    `dnf install -y iscsi-initiator-utils`
    `systemctl enable --now iscsid`

---

### 17.2 Data Directory Not Mounted on a Separate Disk

Symptom:

    `df -hT /data/longhorn` shows that the mount point is `/`.

Risk:

    Longhorn data will be written to the system disk.
    The system disk may become full easily.
    Node DiskPressure may occur.

Resolution:

    Plan to use a separate data disk.
    Mount it to `/data`.
    Create `/data/longhorn`.
    Specify the default data path during Longhorn installation or configure it in the UI/node settings.

---

### 17.3 StorageClass Conflicts

Symptom:

    A PVC uses an unexpected storage system without explicitly specifying `storageClassName`.
    The PVC automatically uses the default StorageClass.

Troubleshooting:

    `kubectl get sc`
    `kubectl describe pvc <pvc-name> -n <namespace>`
    `kubectl describe pv <pv-name>`

Resolution:

    Determine whether to retain the default StorageClass.
    Explicitly specify `storageClassName` in the business YAML file.
    Remove unnecessary default StorageClasses if needed.

---

### 17.4 Insufficient Replica Scheduling

Symptom:

    Volume degradation.
    Insufficient number of replicas.
    The Longhorn UI indicates that replicas cannot be scheduled.

Possible causes:

    Insufficient number of data nodes.
    Insufficient node disk space.
    Nodes are not allowed to schedule replicas.
    Data directory configuration issues.
    Excessively high number of replicas set.

Resolution:

    Reduce the number of experimental replicas.
    Add more data nodes.
    Increase the number of data disks.
    Check the status of Longhorn nodes and disks.

---

### 17.5 Node DiskPressure

Troubleshooting:

    `kubectl describe node <node-name>`
    `df -hT`
    `du -sh /var/lib/containerd`
    `du -sh /data/longhorn`

Possible11. In a production environment, the /data/longhorn directory should be mounted on an independent data disk.
12. It is not recommended to write Longhorn data directly onto the system disk.
13. The number of replicas must be planned in conjunction with the number of nodes and their capacity.
14. For 2 data nodes, it is more appropriate to start with 2 replicas for experimentation.
15. For 3 or more data nodes, using 3 replicas is recommended for verification purposes.
16. Replicas will increase both the capacity usage and the write load.
17. Whether to set Longhorn as the default StorageClass should be decided in advance.
18. When multiple storage systems coexist, it is suggested to explicitly specify the storageClassName for PVCs.
19. Longhorn replicas are not backups; a Backup Target must be planned for production use.
20. The next article will cover the installation methodology of Longhorn Helm: Charts, images, values.yaml files, and version management.

---

## Section 22: Reference Documents

Longhorn Official Documentation:

    https://longhorn.io/docs/latest/

Longhorn Installation Documentation:

    https://longhorn.io/docs/latest/deploy/install/

Longhorn Installation Requirements:

    https://longhorn.io/docs/latest/deploy/install/#installation-requirements

Longhorn Nodes and Volumes:

    https://longhorn.io/docs/latest/nodes-and-volumes/

Longhorn StorageClass:

    https://longhorn.io/docs/latest/references/storage-class-parameters/

Longhorn Backup and Recovery:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Longhorn Troubleshooting:

    https://longhorn.io/kb/troubleshooting/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Storage Classes:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes CSI Documentation:

    https://kubernetes-csi.github.io/docs/