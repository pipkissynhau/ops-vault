# Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass

Recommended path: 05-Storage/03-LongHorn/03-Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass.md

Tags: #Longhorn #Kubernetes #CSI #StorageClass #PV #PVC #open-iscsi #NFS #BlockStorage #NodalPlanning #DiskPlanning #AdvancedSre #ProductionTransport

---

## I. Document Overview

This is the third article of the Longhorn module, focusing on pre-installation environment planning and verification for Longhorn.

Previously completed:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager

This article does not directly install Longhorn, but instead completes essential pre-production checks:

    Is the Kubernetes cluster healthy?
    Do node systems meet requirements?
    Are open-iscsi / iscsid installed and running?
    Is NFSv4 client installed?
    How to plan data disks and data directories?
    Will Longhorn data mistakenly write to system disks?
    Does the node count support replica distribution?
    How should StorageClass be designed?
    Should longhorn be set as the default StorageClass?
    What checks should be retained before installation?

Longhorn is a distributed block storage system within Kubernetes. Pre-installation planning of nodes, disks, dependencies, and StorageClass will directly affect subsequent PVC, Volume, Replica, Backup, Restore, and fault recovery capabilities.

In production, it is not recommended to "helm install directly", but rather to perform pre-installation checks first.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Clearly identify what needs to be checked before Longhorn installation.
2. Check Kubernetes cluster node status.
3. Check kube-system component status.
4. Check current StorageClass, PV, PVC status.
5. Check node system version, kernel, containerd, kubelet status.
6. Install open-iscsi and nfs-common on Ubuntu 22.04.
7. Install iscsi-initiator-utils and nfs-utils on Rocky Linux 9.
8. Confirm iscsid service is running normally.
9. Plan Longhorn data directories.
10. Determine if /data/longhorn is truly mounted to an independent data disk.
11. Understand the relationship between replica count, node count, and disk capacity.
12. Design basic parameters for Longhorn StorageClass.
13. Determine whether to set longhorn as the default StorageClass.
14. Output pre-installation check reports.
15. Prepare for subsequent Helm installation of Longhorn.

---

## III. Experiment Environment Planning

### 3.1 Current Kubernetes Cluster

Default experimental environment:

    Kubernetes: kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node Network Segment: 10.0.0.0/24

Node Planning:

| IP | Hostname | Role | Notes |
|---|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane | Control plane node |
| 10.0.0.21 | k8s-worker01 | Worker | Longhorn data node |
| 10.0.0.22 | k8s-worker02 | Worker | Longhorn data node |

---

### 3.2 Longhorn Data Node Planning

Experimental environment recommendations:

| Node | Participate in Longhorn | Data Directory | Notes |
|---|---|---|---|
| k8s-master01 | Optional | /data/longhorn | Experimental participation is allowed, production generally cautious |
| k8s-worker01 | Yes | /data/longhorn | Recommended |
| k8s-worker02 | Yes | /data/longhorn | Recommended |

Production recommendations:

    Prioritize having Worker nodes participate in Longhorn data storage.
    Whether Control Plane nodes participate in Longhorn depends on cluster scale, taints, resources, and reliability.
    Production environments recommend at least 3 available data nodes for 3-replica distribution.
    If only 2 Worker nodes are available, recommend using 2 replicas or have Master participate in testing.
    Avoid concentrating all replicas on the same node.
    Do not place Longhorn data directories directly on system disks for production.

---

### 3.3 Longhorn Data Directory

Common default data directory for Longhorn:

    /var/lib/longhorn

This article recommends using:

    /data/longhorn

Reasons:

    More clear directory semantics.
    Facilitates independent data disk mounting.
    Facilitates inspections, capacity statistics, and fault diagnosis.
    Avoids accidental system disk filling.

Production principles:

    /data/longhorn should be mounted to an independent data disk.
    Each node's data disk capacity and performance should be as consistent as possible.
    Data disk file systems should use stable mature file systems like xfs or ext4.
    Data disks should not be confused with system disks.
    Data directories should not be mixed with other applications.
    Do not place business logs, image caches, and Longhorn data in the same directory.

---

## IV. Pre-Installation Checklist

Before installing Longhorn, at least check the following items:

| Check Item | Mandatory | Notes |
|---|---|---|
| Kubernetes Node Ready | Mandatory | Do not install if node is abnormal |
| kube-system Components Normal | Mandatory | CoreDNS, CNI, kube-proxy abnormalities need to be resolved first |
| containerd Normal | Mandatory | Longhorn Pod depends on container runtime |
| kubelet Normal | Mandatory | Volume mounting depends on kubelet |
| open-iscsi / iscsid | Mandatory | Longhorn v1 data volumes depend on iSCSI capability |
| NFSv4 Client | Recommended | Required for RWX and some backup scenarios |
| Data Directory | Must be planned | Recommended: /data/longhorn |
| Dedicated Data Disk | Production Mandatory | Experimental use system disk |
| Node Disk Capacity | Mandatory | Replicas will increase capacity usage |
| Node Network | Mandatory | Replica synchronization depends on node network |
| StorageClass Policy | Mandatory | Determines PVC dynamic creation behavior |
| Backup Target Planning | Production Mandatory | Replicas are not backups |
| UI Access Control | Production Mandatory | Longhorn UI is the management entry point |

---

## Five. Practical Operation One: Check Kubernetes Cluster Status

### 5.1 Check Node Status

Run:

    kubectl get nodes -o wide

Focus on:

    STATUS whether all are Ready.
    INTERNAL-IP whether correct.
    VERSION whether differences are too large.
    Whether worker nodes exist.
    Whether NotReady nodes exist.

Example:

    kubectl get nodes -o wide

Expected:

    k8s-master01    Ready    control-plane
    k8s-worker01    Ready    <none>
    k8s-worker02    Ready    <none>

If nodes are NotReady:

    kubectl describe node <node-name>
    journalctl -u kubelet --since "30 minutes ago"
    systemctl status kubelet

Do not proceed with Longhorn installation until nodes recover.

---

### 5.2 Check Node Pressure

Run:

    kubectl describe nodes | grep -E "Name:|DiskPressure|MemoryPressure|PIDPressure|Ready"

Focus on:

    DiskPressure whether False.
    MemoryPressure whether False.
    PIDPressure whether False.
    Ready whether True.

If DiskPressure=True:

    Not recommended to install Longhorn.
    Need to resolve disk space, image cache, log, or mount issues first.

---

### 5.3 Check kube-system Components

Run:

    kubectl get pods -n kube-system -o wide

Focus on:

    CoreDNS whether Running.
    kube-proxy whether Running.
    Calico whether Running.
    metrics-server if installed, whether Running.
    Whether CrashLoopBackOff exists.
    Whether ImagePullBackOff exists.
    Whether Pending exists.

---

### 5.4 Check Global Abnormal Pods

Run:

    kubectl get pods -A | grep -Ev "Running|Completed"

Notes:

    If there are many abnormal pods, need to determine if it will affect Longhorn installation.
    Especially kube-system, CNI, DNS, kubelet related issues must be prioritized.

---

### 5.5 Check Recent Events

Run:

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

If there are many FailedMount or storage-related errors, need to resolve existing storage issues first.

---

## Six. Practical Operation Two: Check Current Storage Resources

### 6.1 Check StorageClass

Run:

    kubectl get storageclass

Or:

    kubectl get sc

Possible scenario one: No StorageClass.

    No resources found

Notes:

    Cluster has no dynamic storage provisioning capability.
    Longhorn installation will add longhorn StorageClass.

Possible scenario two: Existing StorageClass.

Example:

    local-path
    nfs-client
    rook-ceph-block
    longhorn

Need to confirm:

    Whether a default StorageClass exists.
    Whether current business is using other storage.
    Whether Longhorn can be set as default StorageClass later.

---

### 6.2 Check Default StorageClass

Run:

    kubectl get sc

If you see:

    (default)

Indicates this StorageClass is the default storage class.

Details:

    kubectl describe sc <storageclass-name>

Focus on:

    Provisioner
    ReclaimPolicy
    VolumeBindingMode
    AllowVolumeExpansion
    Parameters

Production recommendations:

    When multiple storage systems coexist, do not fully rely on default StorageClass.
    Business PVCs should explicitly specify storageClassName.
    Whether to set Longhorn as default StorageClass should be determined before installation.

---

### 6.3 Check PV

Run:

    kubectl get pv

If there are existing PVs, need to confirm: /think

Is there any business using it.  
Is it part of another storage system.  
Is there a Released / Failed status.  
Are there residual PVs.

Detailed inspection:

    kubectl describe pv <pv-name>

---

### 6.4 Check PVC

Execute:

    kubectl get pvc -A

Focus on:

    Is there an existing production PVC.  
    Is there a Pending PVC.  
    Is there a PVC using the default StorageClass.  
    Is there a Bound PVC with unknown business usage.

Detailed inspection:

    kubectl describe pvc <pvc-name> -n <namespace>

---

## SevenI don't know.Practical Operation Three: Check Node System Basic Information

The following commands need to be executed on all nodes planned to participate in Longhorn.

Includes:

    k8s-master01  
    k8s-worker01  
    k8s-worker02

### 7.1 Check System Version

Execute:

    hostname  
    cat /etc/os-release  
    uname -a

Record:

    Operating system version  
    Kernel version  
    Hostname  
    Node IP

---

### 7.2 Check Container Runtime

Execute:

    systemctl status containerd  
    containerd --version  
    crictl info | head -50

If crictl is unavailable, check first:

    which crictl

Production notes:

    It is not recommended to directly destroy containerdBottom configuration for Longhorn image pull issues.  
    Image issues should be resolved first by specifying the image repository through Helm values.yaml.  
    Image management will be detailed in 04-Longhorn Helm Installation Methodology.

---

### 7.3 Check kubelet Status

Execute:

    systemctl status kubelet  
    journalctl -u kubelet --since "30 minutes ago" | tail -100

Focus on:

    Is kubelet active.  
    Are there FailedMount errors.  
    Are there CSI-related errors.  
    Is there a NodeNotReady status.  
    Is there disk pressure.

---

### 7.4 Check Node Resources

Execute:

    free -h  
    uptime  
    df -hT  
    lsblk  
    mount

Focus on:

    Is the system disk nearly full.  
    Are there independent data disks.  
    Is /data mounted to an independent disk.  
    File system type.  
    Is node load abnormal.  
    Is memory too low.

---

### 7.5 Check Time Synchronization

Execute:

    timedatectl

Focus on:

    Is System clock synchronized yes.  
    Is the time zone as expected.  
    Is NTP service active.

Time inconsistency may lead to:

    Certificate issues.  
    Confused log timestamps.  
    Difficulties in distributed system troubleshooting.  
    Inaccurate monitoring data.

---

## EightI don't know.Practical Operation Four: Install and Check open-iscsi

### 8.1 Why Need open-iscsi

Longhorn v1 data volume mounting depends on node-side iSCSI capabilities.

If nodes lack open-iscsi or iscsid is not running, it may result in:

    PVC is already Bound, but Pod mount fails.  
    Volume attach failure.  
    MountVolume failure in kubelet events.  
    Pod stuck in ContainerCreating for a long time.  
    Abnormal Volume status in Longhorn UI.

Therefore, all nodes planned to run Longhorn Volume must install and run iSCSI-related services.

---

### 8.2 Ubuntu 22.04 Install open-iscsi

Execute on each node:

    apt update  
    apt install -y open-iscsi

Start and set to boot:

    systemctl enable --now iscsid

Check service:

    systemctl status iscsid  
    systemctl is-active iscsid  
    systemctl is-enabled iscsid

Check commands:

    which iscsiadm  
    iscsiadm --version

---

### 8.3 Rocky Linux 9 Install iSCSI Dependencies

Execute on Rocky Linux 9 nodes:

    dnf install -y iscsi-initiator-utils

Start and set to boot:

    systemctl enable --now iscsid

Check:

    systemctl status iscsid  
    systemctl is-active iscsid  
    systemctl is-enabled iscsid  
    which iscsiadm  
    iscsiadm --version

---

### 8.4 Batch Check iscsid Status

Execute on each node separately:

    systemctl is-active iscsid && echo "iscsid is running"

Or:

    systemctl status iscsid --no-pager

If not running:

    systemctl enable --now iscsid

---

### 8.5 Common Issues

#### Issue One: iscsiadm Not Found

Phenomenon:

    iscsiadm: command not found

Resolution:

Ubuntu:

    apt install -y open-iscsi

Rocky Linux 9:

    dnf install -y iscsi-initiator-utils

---

#### Issue Two: iscsid Not Running

Phenomenon:

    systemctl is-active iscsid  
    inactive

Resolution:

    systemctl enable --now iscsid

---

#### Issue Three: Pod Mount PVC Failed

Troubleshoot: /think

kubectl describe pod <pod-name> -n <namespace>
kubectl get events -A --sort-by=.lastTimestamp | tail -100
systemctl status iscsid
journalctl -u kubelet --since "30 minutes ago" | tail -100

---

## IX. Hands-on 5: Installing and Checking NFSv4 Client

### 9.1 Why NFSv4 Client is Needed

Longhorn's RWO block volumes primarily rely on iSCSI.

However, the following scenarios require NFSv4 client capabilities:

    RWX volume support
    Shared file access scenarios
    Some backup and recovery related scenarios
    Node compatibility checks when using NFS Backup Target later

It is recommended to uniformly install NFS client tools on Longhorn nodes to reduce troubleshooting costs in the future.

---

### 9.2 Installing nfs-common on Ubuntu 22.04

Execute:

    apt update
    apt install -y nfs-common

Check:

    which mount.nfs
    mount.nfs -V

---

### 9.3 Installing nfs-utils on Rocky Linux 9

Execute:

    dnf install -y nfs-utils

Check:

    which mount.nfs
    mount.nfs -V

---

### 9.4 Checking Kernel NFS Support

Execute:

    cat /boot/config-$(uname -r) | grep CONFIG_NFS_V4

If the file does not exist, try:

    zcat /proc/config.gz | grep CONFIG_NFS_V4

Note:

    The kernel configuration file paths may differ across distributions.
    The absence of the command does not necessarily mean it is unsupported; actual system checks are needed.

---

## X. Hands-on 6: Planning and Preparing Longhorn Data Directory

### 10.1 Creating Data Directory

On each node planned to participate in Longhorn:

    mkdir -p /data/longhorn

Check:

    ls -ld /data/longhorn

---

### 10.2 Checking Disk Where the Directory is Located

Execute:

    df -hT /data/longhorn

Example output if:

    Filesystem     Type  Size  Used Avail Use% Mounted on
    /dev/sda2      ext4   50G   20G   30G  40% /

Note:

    /data/longhorn is still on the system disk.
    Only suitable for learning experiments.
    Not recommended for direct production use.

If output is similar to:

    /dev/sdb1      xfs   500G  10G  490G   2% /data

Note:

    /data/longhorn is on an independent data disk or under the /data mount point.
    More suitable for production planning.

---

### 10.3 Viewing Disk Structure

Execute:

    lsblk -f

Focus on:

    Which disk is the system disk.
    Which disk is the data disk.
    Whether the data disk is formatted.
    Whether the data disk is mounted.
    What file system type it is.
    Whether the mount point is stable.

---

### 10.4 Example of Mounting Independent Data Disk

Warning:

    The following formatting commands will erase disk data.
    Only execute on new disks or confirmed data-free experimental disks.
    Must confirm device names before production execution; dual-person verification is recommended.
    Do not mistakenly format the system disk.

Assume the new data disk is:

    /dev/sdb

Confirm the disk:

    lsblk

Example of creating file system:

    mkfs.xfs /dev/sdb

Create mount directory:

    mkdir -p /data

Get UUID:

    blkid /dev/sdb

Edit /etc/fstab:

    vi /etc/fstab

Add similar content:

    UUID=<replace with actual UUID> /data xfs defaults,noatime 0 0

Mount:

    mount -a

Check:

    df -hT /data
    lsblk -f

Create Longhorn data directory:

    mkdir -p /data/longhorn

Check:

    df -hT /data/longhorn

---

### 10.5 ext4 Example

If using ext4:

    mkfs.ext4 /dev/sdb

fstab example:

    UUID=<replace with actual UUID> /data ext4 defaults,noatime 0 0

Check:

    mount -a
    df -hT /data

---

### 10.6 Directory Permission Check

Execute:

    ls -ld /data/longhorn

Generally, root ownership is sufficient.

Do not arbitrarily execute:

    chmod 777 /data/longhorn

It is not recommended to use crude permission methods to resolve mount issues in production.

If Longhorn reports permission-related errors after installation, troubleshoot by combining Longhorn logs, Pod security context, and node directory status.

---

## XI. Replica Count and Capacity Planning

### 11.1 Meaning of Replica Count

Longhorn Volume can configure replica count.

Example:

    numberOfReplicas: 3

Means:

    A Volume will try to create 3 Replicas.
    Replicas will be distributed across different nodes and disks as much as possible.
    When a node or replica fails, other replicas are still available.

---

### 11.2 Replica Count and Node Count

Replica count should match node count.

| Node Count | Recommended Replica Count | Notes |
|---|---|---|
| 1 node | 1 | Only for learning, lacks node high availability |
| 2 nodes | 2 | Suitable for basic high availability experiments |
| 3 nodes | 3 | More suitable for production common replica distribution |
| 4 or more | 3 | Common production choice, can adjust based on business needs |

Current experimental cluster:

    1 Master + 2 Worker

If only allowing Worker nodes to participate in Longhorn:

    Available data nodes are 2.
    Recommend using 2 replicas first.

If allowing Master node to participate in experiments:

    Available data nodes are 3.
    Can verify 3 replica distribution.

Production recommendations:

    Do not force 3 replicas to be considered fully highly available with only 2 data nodes.
    Replica count should not be much greater than the number of available schedulable data nodes, otherwise replica scheduling may be insufficient.
    Replica distribution should be determined based on node anti-affinity policies, disk space, and node status.

### 11.3 Replica Count and Capacity

Replicas will amplify capacity usage.

Example:

| PVC Size | Replica Count | Estimated Raw Capacity Usage |
|---|---|---|
| 10Gi | 1 | Approximately 10Gi |
| 10Gi | 2 | Approximately 20Gi |
| 10Gi | 3 | Approximately 30Gi |
| 100Gi | 3 | Approximately 300Gi |

Notes:

    This is a rough understanding.
    You also need to consider snapshots, backups, metadata, and temporary usage during rebuild processes.
    Capacity planning for production cannot be based solely on PVC request volume.
    You should calculate based on actual replica count and growth trends.

---

### 11.4 Capacity Watermark Recommendations

Production recommendations:

| Usage Rate | Recommended Actions |
|---|---|
| 70% | Start paying attention to growth trends |
| 80% | Develop an expansion plan |
| 85% | Clarify expansion time and plan |
| 90% | Handle with high priority |
| 95% | Severe risk, may affect writes and replica rebuild |

Longhorn disk space insufficiency may lead to:

    PVC creation failure.
    Replica scheduling failure.
    Replica rebuild failure.
    Volume Degraded remaining unrecovered for a long time.
    Pod write failure.

---

## TwelveI don't know.Node Scheduling Planning

### 12.1 Whether to Allow Master to Participate in Longhorn

Check Master node taints:

    kubectl describe node k8s-master01 | grep -i taint

Common:

    node-role.kubernetes.io/control-plane:NoSchedule

If the Master has NoSchedule taint, regular Pods will not be scheduled to the Master by default.

Production recommendations:

    Control Plane nodes generally do not recommend hosting business and storage data.
    In small-scale experiments, you can temporarily allow Master to participate in Longhorn.
    Whether to participate should be combined with resource availability, stability, and taint tolerance configuration.

---

### 12.2 Worker Node Planning

Worker node recommendations:

    Has independent data disks.
    Sufficient CPU and memory resources.
    Stable network.
    No DiskPressure.
    kubelet is normal.
    containerd is normal.
    open-iscsi is normal.
    NFS client is normal.

Check:

    kubectl get nodes -o wide
    kubectl describe node k8s-worker01
    kubectl describe node k8s-worker02

---

### 12.3 Longhorn Node Label Planning

In production, you can use node labels to identify storage nodes.

Example:

    kubectl label node k8s-worker01 storage.longhorn.io/enabled=true
    kubectl label node k8s-worker02 storage.longhorn.io/enabled=true

Check:

    kubectl get nodes --show-labels | grep storage.longhorn.io

Notes:

    Labels themselves will not automatically restrict Longhorn.
    Subsequently, you can combine with Longhorn node settings, scheduling policies, Helm values, or operation specifications.
    This document first records the planning, and specific scheduling control will be expanded in subsequent installation and production recommendations.

---

## ThirteenI don't know.StorageClass Planning

### 13.1 Longhorn Default StorageClass

After Longhorn installation, it usually creates:

    longhorn

Check:

    kubectl get sc

Details:

    kubectl describe sc longhorn

StorageClass mainly focuses on:

    provisioner
    reclaimPolicy
    volumeBindingMode
    allowVolumeExpansion
    parameters

---

### 13.2 Whether to Set as Default StorageClass

If you set Longhorn as the default StorageClass:

    PVCs without specified storageClassName may automatically use Longhorn.

Advantages:

    Convenient to use.
    Application YAML can omit storageClassName.
    Suitable for small clusters with a single storage system.

Risks:

    May be misused in multi-storage system environments.
    Some PVCs that shouldn't use Longhorn may automatically use it.
    Business may not know the underlying storage type.
    Subsequent migration and troubleshooting may be unclear.

Recommendations:

    In learning phases, you can set it as default for experimentation.
    In production, it's recommended to explicitly specify storageClassName for business PVCs.
    In multi-storage system environments, it's not recommended to easily set Longhorn as default.

Example to set as default:

    kubectl patch storageclass longhorn \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

Example to cancel default:

    kubectl patch storageclass longhorn \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

---

### 13.3 ReclaimPolicy Planning

ReclaimPolicy determines how PV and backend data are handled after PVC deletion.

Common:

    Delete
    Retain

Delete:

    After PVC deletion, PV and backend Longhorn Volume may be deleted.
    Suitable for temporary environments or data that can be rebuilt.
    Risk is accidental PVC deletion may lead to data loss.

Retain:

    After PVC deletion, PV and backend data are retained.
    More secure but requires manual cleanup.
    Suitable for important data scenarios.

Production recommendations:

    Use Delete cautiously for critical business.
    For important data, consider Retain or establish clear deletion approval processes.
    Do not allow ordinary business personnel to delete PVCs arbitrarily.

---

### 13.4 AllowVolumeExpansion Planning

AllowVolumeExpansion indicates whether PVC expansion is allowed.

Recommendation: /think

Enable expansion capability.
Before expansion, confirm the underlying disk capacity.
After expansion, confirm whether the file system has been successfully expanded.
Do not only expand PVC, but also check Longhorn Volume and node disk.

Check:

    kubectl describe sc longhorn | grep -i AllowVolumeExpansion

---

### 13.5 VolumeBindingMode Planning

Common modes:

    Immediate
    WaitForFirstConsumer

Immediate:

    A PV and backend Volume are created immediately after PVC creation.

WaitForFirstConsumer:

    Bind when the Pod uses PVC and is scheduled.

Longhorn's default behavior should be based on the actual StorageClass after installation.

Production understanding:

    For strong binding to local disks, WaitForFirstConsumer is important.
    For distributed volumes like Longhorn, also understand the impact of binding timing on scheduling and replica distribution.
    Actual settings should be based on Longhorn Chart and StorageClass parameters.

---

### 13.6 Replica Count Parameter Planning

Longhorn StorageClass can define replica count via parameters.

Example parameter idea:

    numberOfReplicas: "2"

If the current experimental environment has only two Worker nodes, it is recommended to plan:

    numberOfReplicas: "2"

If there are three data nodes, production or full experimental environments can plan:

    numberOfReplicas: "3"

Note:

    Replica count is not better the higher.
    Higher replica count leads to more obvious write amplification.
    Higher replica count consumes more capacity.
    When node count is insufficient, replicas cannot be effectively distributed.

---

## Fourteen. Pre-Installation Check Report Generation

### 14.1 Create Check Script

Create a simple check script on the management node:

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

Permissions:

    chmod +x longhorn-precheck.sh

Execution:

    ./longhorn-precheck.sh

View:

    ls -lh longhorn-precheck-*.log
    less longhorn-precheck-*.log

---

### 14.2 Node-side Check Commands

Execute on each node:

    hostname
    cat /etc/os-release
    uname -a
    systemctl status kubelet --no-pager
    systemctl status containerd --no-pager
    systemctl status iscsid --no-pager
    which iscsiadm
    iscsiadm --version
    which mount.nfs
    df -hT
    lsblk -f
    timedatectl

You can also save to a file:

    NODE=$(hostname)
    REPORT="${NODE}-longhorn-node-precheck-$(date +%F-%H%M%S).log"

    {
      echo "===== Node: ${NODE} ====="
      date
      echo

      echo "===== OS ====="
      cat /etc/os-release
      uname -a
      echo

      echo "===== Services ====="
      systemctl status kubelet --no-pager || true
      systemctl status containerd --no-pager || true
      systemctl status iscsid --no-pager || true
      echo

      echo "===== iSCSI ====="
      which iscsiadm || true
      iscsiadm --version || true
      echo /think

echo "===== NFS ====="
    which mount.nfs || true
    mount.nfs -V || true
    echo

    echo "===== Disk ====="
    df -hT
    lsblk -f
    echo

    echo "===== Time ====="
    timedatectl
    } > "${REPORT}" 2>&1

    echo "Report saved to ${REPORT}"

---

## FifteenI don't know.Pre-Installation Conclusion Template

It is recommended to output a conclusion before installation.

Example:

    Kubernetes Cluster Status: Normal / Abnormal
    Node Count: 3
    Planned Longhorn Nodes: k8s-worker01, k8s-worker02
    Allow Master Nodes to Participate in Experiment: Yes / No
    Data Directory: /data/longhorn
    Is Data Directory on a Dedicated Data Disk: Yes / No
    open-iscsi Installed: Yes / No
    iscsid Running: Yes / No
    NFS Client Installed: Yes / No
    Current StorageClass: None / Exists
    Default StorageClass Exists: Yes / No
    Longhorn Planned Replica Count: 2 / 3
    Longhorn Set as Default StorageClass: Yes / No
    Installation Conditions Met: Yes / No

If "Installation Conditions Not Met", address:

    Node NotReady
    DiskPressure
    iscsid Not Running
    Data Disk Not Mounted
    System Disk Space Insufficient
    kube-system Component Abnormal
    StorageClass Conflict or Default StorageClass Uncertain

---

## SixteenI don't know.Special Notes for Production Installation

### 16.1 Do Not Destroy containerd Underlying Runtime

If the Longhorn image pull fails, do not immediately modify containerd global configuration.

Recommended steps:

    Check Longhorn Chart first.
    Then check values.yaml.
    Then extract image list.
    Then synchronize images to domestic repositories.
    Then specify image repository via Helm values.yaml.
    Finally install.

This approach is more controlled and won't affect the entire Kubernetes cluster runtime.

---

### 16.2 Do Not Write Longhorn Data to System Disk

System disk full may cause:

    kubelet Abnormal.
    containerd Abnormal.
    Node DiskPressure.
    Pod Evicted.
    Longhorn Replica Abnormal.
    Volume Degraded.
    Replica Cannot Rebuild.

Production must confirm:

    df -hT /data/longhorn

Confirm /data/longhorn is on the expected data disk.

---

### 16.3 Do Not Ignore Backup Targets

Longhorn replicas are not backups.

Plan before installation:

    Backup Target Uses NFS or S3.
    Is Backup Target Isolated from K8s Cluster.
    Has Regular Backup.
    Has Recovery Drill.
    Monitors Backup Failures.

This will be detailed in section 07-Longhorn Backup Recovery.

---

### 16.4 Do Not Blindly Set Default StorageClass

If Longhorn is the only storage system, consider setting as default.

If cluster already has:

    NFS StorageClass
    local-path
    Ceph CSI
    Cloud Vendor Cloud Disk CSI

Recommend business PVC explicitly specify:

    storageClassName: longhorn

Avoid accidental use of default storage.

---

## SeventeenI don't know.Common Issue Troubleshooting

### 17.1 Node Lacks iscsid

Symptoms:

    Pod Fails to Mount Longhorn PVC.
    iSCSI Related Errors in kubelet Events.
    iscsiadm Command Not Found.

Troubleshoot:

    which iscsiadm
    systemctl status iscsid

Resolution:

Ubuntu:

    apt update
    apt install -y open-iscsi
    systemctl enable --now iscsid

Rocky Linux 9:

    dnf install -y iscsi-initiator-utils
    systemctl enable --now iscsid

---

### 17.2 Data Directory Not Mounted to Dedicated Disk

Symptoms:

    df -hT /data/longhorn Shows Mount Point as /

Risk:

    Longhorn Data Written to System Disk.
    System Disk Likely to Fill Up.
    Node May Experience DiskPressure.

Resolution:

    Plan Dedicated Data Disk.
    Mount to /data.
    Create /data/longhorn.
    Specify Default Data Path When Installing Longhorn or Set in UI/Node Configuration.

---

### 17.3 StorageClass Conflict

Symptoms:

    PVC Not Specified storageClassName, Yet Used Unexpected Storage.
    PVC Automatically Used Default StorageClass.

Troubleshoot:

    kubectl get sc
    kubectl describe pvc <pvc-name> -n <namespace>
    kubectl describe pv <pv-name>

Resolution:

    Clarify Whether to Retain Default StorageClass.
    Explicitly Write storageClassName in Business YAML.
    Cancel Unneeded Default StorageClass When Necessary.

---

### 17.4 Insufficient Replica Scheduling

Symptoms:

    Volume Degraded.
    Insufficient Replica Count.
    Longhorn UI Shows Unable to Schedule Replica.

Possible Causes:

    Insufficient Data Nodes.
    Node Disk Space Insufficient.
    Node Disallows Scheduling.
    Data Directory Not Configured.
    Replica Count Set Too High.

Resolution:

    Reduce Experimental Replica Count.
    Add More Data Nodes.
    Add More Data Disks.
    Check Longhorn Node and Disk Status.

---

### 17.5 Node DiskPressure

Troubleshoot: /think

kubectl describe node <node-name>
df -hT
du -sh /var/lib/containerd
du -sh /data/longhorn

Handling Direction:

    Clean up unused images.
    Clean up logs.
    Expand system disk or data disk.
    Confirm Longhorn data is not mistakenly written to system disk.
    Avoid creating new Volumes on nodes with DiskPressure.

---

## EighteenI don't know.High-Risk Operation Reminder

The following operations must be cautious in production environments:

    Format disk
    Modify /etc/fstab
    Delete /data/longhorn
    Delete /var/lib/longhorn
    Delete PVC
    Delete PV
    Delete StorageClass
    Modify default StorageClass
    Modify Longhorn default replica count
    Uninstall Longhorn
    Batch restart Longhorn components
    Adjust data directory without backup

Before execution, you must confirm:

    Whether the device name is correct.
    Whether data will be cleared.
    Whether there is a backup.
    Whether there is a maintenance window.
    Whether there is a rollback plan.
    Whether you know which PVCs are affected.
    Whether it has been business confirmed.
    Whether there is dual-person review.

---

## NineteenI don't know.Completion Standard of This Article

After completing this article, you should at least have the following results:

| Item | Standard |
|---|---|
| Kubernetes Nodes | All Ready |
| kube-system | Core components Running |
| StorageClass | Current status is clearly defined |
| PV/PVC | Whether there are existing business is clearly defined |
| open-iscsi | Already installed |
| iscsid | Already running and set to start on boot |
| NFS Client | Already installed or recorded as not needed |
| Data Directory | /data/longhorn is created |
| Data Disk | Whether it is independently mounted is confirmed |
| Replica Count | Already planned according to node count |
| Default StorageClass | Whether it is set is clearly defined |
| Installation Conditions | Conclusion is formed |

---

## TwentyI don't know.Interview Answering Logic

If asked in an interview:

    What will you check before installing Longhorn?

You can answer:

    I won't install Longhorn directly. I will first perform pre-installation checks. First, check the Kubernetes cluster status, such as kubectl get nodes, kube-system components, CNI, CoreDNS, kubelet, containerd whether they are normal, confirm nodes have no DiskPressure, MemoryPressure, or a lot of FailedMount or scheduling anomalies.
    Then check all planned nodes for Longhorn installation whether open-iscsi is installed, and confirm iscsid service is already running and set to start on boot, because Longhorn v1 data volume mounting depends on node-side iSCSI capability. If RWX or NFS-related capabilities are needed, also install NFSv4 client, such as nfs-common on Ubuntu, nfs-utils on Rocky.
    For disks, I will plan independent data disks, such as mounted to /data/longhorn, and use df -hT, lsblk to confirm it is not a system disk directory. In production, it is not recommended to put Longhorn data on system disk, otherwise system disk full will cause kubelet, containerd, and Longhorn all to fail.
    Also need to plan replica count according to node count. If only two Worker nodes, you shouldn't blindly set 3 replicas and think it is fully HA. Replica count will amplify capacity usage and also increase network and write pressure.
    For StorageClass, I will first check existing StorageClass and default StorageClass in the cluster, confirm whether Longhorn is set as default. If the cluster has multiple storage systems, I prefer to let business PVC explicitly specify storageClassName: longhorn, avoid misusing default storage.
    Finally I will plan Backup Target, because Longhorn Replica is not backup. Production must consider NFS or S3 Backup Target, and do recovery drills.

---

## Twenty-oneI don't know.Summary of This Article

This article completes Longhorn installation planning:

1. Longhorn installation must first check Kubernetes cluster health before installation.
2. Nodes must be Ready, kube-system core components must be normal.
3. Pre-installation should check existing StorageClass, PV, PVC.
4. open-iscsi / iscsid is critical dependency on node side for Longhorn.
5. Ubuntu 22.04 uses open-iscsi.
6. Rocky Linux 9 uses iscsi-initiator-utils.
7. RWX and some backup scenarios recommend installing NFSv4 client.
8. Ubuntu uses nfs-common.
9. Rocky Linux 9 uses nfs-utils.
10. Longhorn data directory is recommended to plan as /data/longhorn.
11. In production environment, /data/longhorn should be mounted to independent data disk.
12. Not recommended to directly write Longhorn data to system disk.
13. Replica count must be planned according to node count and capacity.
14. 2 data nodes are more suitable for first using 2 replicas for experiment.
15. 3 or more data nodes are more suitable for verifying 3 replicas.
16. Replica will amplify capacity usage and write pressure.
17. Whether to set Longhorn as default StorageClass needs to be decided in advance.
18. When multiple storage systems coexist, it is recommended to explicitly specify storageClassName for PVC.
19. Longhorn replica is not backup, production must plan Backup Target.
20. Next article will enter Longhorn Helm installation methodology: Chart, images, values.yaml and version management.

---

## Twenty-twoI don't know.Reference Documents

Longhorn official documentation:

    https://longhorn.io/docs/latest/

Longhorn installation documentation:

    https://longhorn.io/docs/latest/deploy/install/

Longhorn installation requirements:

    https://longhorn.io/docs/latest/deploy/install/#installation-requirements

Longhorn nodes and volumes:

    https://longhorn.io/docs/latest/nodes-and-volumes/

Longhorn StorageClass:

    https://longhorn.io/docs/latest/references/storage-class-parameters/

Longhorn backup and recovery:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Longhorn troubleshooting:

    https://longhorn.io/kb/troubleshooting/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Storage Classes:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes CSI documentation:

    https://kubernetes-csi.github.io/docs/