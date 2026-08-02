# Longhorn Phase Summary: From K8s Storage Plugins to Production Operations

Recommended Path: 05-Storage/03-LongHorn/99-Longhorn Phase Summary: From K8s Storage Plugins to Production Operations.md

Tags: #Longhorn #Kubernetes #CSI #PV #PVC #StorageClass #Block Storage #Replica #BackupTarget #Snapshot #Troubleshooting #Performance Optimization #Advanced SRE #Production Operations

---

## I. Document Description

This document serves as a phase summary for the Longhorn module, providing a comprehensive review of the entire learning process related to Longhorn.

Previous sections have covered:

- 00-Longhorn Directory Index
- 01-Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- 02-Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- 03-Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass
- 04-Longhorn Helm Installation Methodology: Charts, Images, values.yaml, and Version Management
- 05-Longhorn Dynamic Volumes in Practice: PVC, PV, Pod Mounting, and Data Persistence
- 06-Longhorn Replica Mechanism: Number of Replicas, Node Distribution, and Data High Availability
- 07-Longhorn Backup and Recovery: Backup Target, Snapshot, and Volume Restore
- 08-Longhorn Troubleshooting: Volume Degradation, Replica Rebuilding, and Node Exceptions
- 09-Longhorn Performance and Production Recommendations: Disks, Networks, Scheduling, and Resource Limits

This section focuses on the following key points:

    What problems Longhorn addresses
    Longhorn's role within the Kubernetes storage ecosystem
    The relationship between PV/PVC/StorageClass/CSI and Longhorn
    How Longhorn's core components work together
    Understanding the Longhorn replica mechanism
    Distinguishing between Snapshot, Backup, and Restore
    How to plan for Longhorn in a production environment
    How to troubleshoot Longhorn issues
    Identifying performance bottlenecks in Longhorn
    Scenarios where Longhorn is suitable or not suitable
    How to progress from being able to install Longhorn to managing it in a production setting

---

## II. Phase Positioning

Longhorn is a Kubernetes cloud-native distributed block storage system.

Its primary purpose is:

    To provide Kubernetes with the capability to dynamically manage PV/PVC resources
    To integrate with the Kubernetes storage framework through CSI
    To organize node local disks into persistent block storage volumes that can be used by Pods
    To offer stateful applications features such as Volume, Replica, Snapshot, Backup, and Restore capabilities

In simple terms:

    Longhorn acts as a "distributed cloud disk system" within Kubernetes.

It is not:

    An object storage solution
    A traditional NAS system
    A standard NFS service
    A high-end SAN storage system
    A high-performance storage alternative for general-purpose databases
    A complete replacement for Ceph

Its main objectives are to address the following challenges in Kubernetes:

    The need for persistent data storage for Pods
    The lack of cloud disk support in bare metal or privatized K8s clusters
    The dynamic allocation of PVC resources for small and medium-sized stateful applications
    The recovery of Volume replicas in case of node or disk failures
    The provision of snapshot, backup, and restoration capabilities within Kubernetes' internal storage system

---

## III. The Relationship Between Longhorn and the Kubernetes Storage Ecosystem

### 3.1 Key Objects in Kubernetes Storage

The core objects in Kubernetes storage include:

| Object | Definition |
|---|-------------------|
| StorageClass | A template for dynamically creating PVs |
| PVC | A user's request for storage resources |
| PV | A persistent volume resource within Kubernetes |
| CSI | The standard interface for Kubernetes to connect with external storage systems |
| Pod | The business execution unit that uses PVCs |

Longhorn integrates into Kubernetes via the CSI interface.

The complete process is as follows:

    User creates a PVC
        |
        v
    PVC specifies `storageClassName: longhorn`
        |
        v
    Kubernetes calls Longhorn's CSI component
        |
        v
    Longhorn creates a Volume
        |
        v
    Kubernetes automatically generates a PV
        |
        v
    The PVC is bound to the PV
        |
        v
    The Pod mounts the PVC
        |
        v
    The application reads/writes data from the mounted directory
        |
        v
    Data is stored in the Longhorn Volume

---

### 3.2 Longhorn's Dynamic Volume Creation Process

The dynamic volume creation process involves the following steps:

    PVC
      |
      v
    `StorageClass: longhorn`
      |
      v
    CSI Provisioner
      |
      v
    Longhorn Manager
      |
      v| csi-resizer | Handles PVC scaling |
| csi-snapshotter | Handles snapshot functionality |
| Engine | The read-write engine for volumes |
| Replica | A copy of volume data |
| Instance Manager | Manages Engine/Replica instances |
| Longhorn UI | Management interface |
| Longhorn CRD | Resource representation of Longhorn objects in Kubernetes |

---

### 4.3 Longhorn Object Relationships

A PVC may correspond to the following:

    PVC
      |
      v
    PV
      |
      v
    Longhorn Volume
      |
      +--> Engine
      |
      +--> Replica 1
      +--> Replica 2
      +--> Replica 3
      |
      v
    Instance Manager
      |
      v
    Node Disk

Common commands for viewing:

    kubectl get pvc -A
    kubectl get pv
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system get engines.longhorn.io -o wide
    kubectl -n longhorn-system get instancemanagers.longhorn.io
    kubectl -n longhorn-system get nodes.longhorn.io

---

## V. Summary of Experimental Environment Setup

### 5.1 Cluster Planning

Default experimental environment:

    Kubernetes: Kubeadm cluster
    Operating system: Ubuntu Server 22.04.5 LTS
    Container runtime: containerd
    CNI: Calico
    Node IP range: 10.0.0.0/24

Node planning:

| IP | Hostname | Role | Recommended for Longhorn |
|---|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane | Suitable for experimentation, cautious in production |
| 10.0.0.21 | k8s-worker01 | Worker | Highly recommended |
| 10.0.0.22 | k8s-worker02 | Worker | Highly recommended |

---

### 5.2 Data Directory Planning

The default data directory for Longhorn is typically:

    /var/lib/longhorn

It is recommended to organize it as follows:

    /data/longhorn

Production recommendations:

    Use a dedicated data disk for /data/longhorn.
    It is not advised to place the Longhorn data directory on the system disk.
    Avoid using symbolic links instead of direct mounting.
    It is recommended to use xfs or ext4 for the data disk.
    The data disk should be listed in /etc/fstab to ensure automatic mounting after restarts.

Verification commands:

    df -hT /data/longhorn
    lsblk -f
    mount | grep /data
    du -sh /data/longhorn

---

### 5.3 Node Dependency Summary

All nodes intended to host Longhorn Volumes must meet the following requirements:

For Ubuntu 22.04:

    Execute `apt update` and `apt install -y open-iscsi nfs-common`, then enable `iscsid` service.

For Rocky Linux 9:

    Install `isdci-initiator-utils` and `nfs-utils`, then enable `iscsid` service.

Verification steps:

    Check if `iscsid` is active: `systemctl is-active iscsid`
    Verify the `iscsiadm` version: `iscsiadm --version`
    Test NFS mounting: `mount.nfs -V`

Notes:

    `open-iscsi` and `iscsid` are essential for mounting Longhorn RWO volumes.
    The NFSv4 client is primarily used for RWX or shared access scenarios.
    In cases of `Pod ContainerCreating / FailedMount`, check the `iscsid` configuration first.

---

## VI. Summary of Helm Installation Best Practices

### 6.1 Installation Principles

Helm is recommended for managing Longhorn installations.

Production installations should avoid:

    Installing without checking versions
    Installing without exporting the `values.yaml` file
    Installing without reviewing the image list
    Arbitrarily modifying containerd global settings in case of image retrieval failures
    Failing to keep Helm installation records
    Assuming the installation is complete without verifying PVCs

Recommended process:

    1. Check the Kubernetes cluster status.
    2. Verify node configuration for `open-iscsi` and `iscsid`.
    3. Ensure the data disk and `/data/longhorn` are set up correctly.
    4. Add the Longhorn Helm repository.
    5. Search for available versions using `helm search repo longhorn/longhorn --versions`.
    6. Select a specific version for installation.
    7. Export default configuration with `helm show values`.
    8. Use `helm template` to generate the final YAML file```markdown
kubectl exec -n <namespace> <pod-name> -- sh -c "echo test > /data/test.txt"
kubectl exec -n <namespace> <pod-name> -- cat /data/test.txt

---

### 7.2 Differences Between Deployment and StatefulSet Usage

Deployment + PVC:

    Suitable for single-replica applications.
    Multiple replicas sharing the same RWO PVC can lead to Multi-Attach issues.
    Not suitable for most stateful, multi-replica applications.

StatefulSet + volumeClaimTemplates:

    Each Pod has its own independent PVC.
    Ideal for stateful, multi-replica applications.
    Particularly suitable for scenarios like MySQL, Redis, Prometheus, ZooKeeper, Kafka, etc.

Key Principles:

    It is not recommended to use RWO PVCs for simultaneous read and write mounting by multiple Pods on different nodes.
    For stateful, multi-replica applications, StatefulSet with independent PVCs is the preferred choice.

---

## VIII. Summary of Replica Mechanism

### 8.1 Problems Replicated By Replicas

Replicas are used to enhance Volume availability.

They can handle:

    Single-replica failures
    Single-node failures
    Single-disk failures
    Temporary node outages
    Replica reconstruction
    Recovery after Volume degradation

However, they cannot address:

    User-induced file deletions
    Application-generated data corruption
    Accidental deletion of PVCs or Volumes
    Complete cluster failures
    Simultaneous failure of all replicas

Conclusion:

    Replicas ensure high availability.
    However, they are not designed as backups.

---

### 8.2 Planning the Number of Replicas

| Number of Replicas | Suitable Scenarios | Risks |
|---|---|---|
| 1 | Learning, temporary use, data that can be recreated, applications with built-in replication capabilities | No node-level high availability |
| 2 | Two-node experiments, small to medium-sized businesses, scenarios where capacity is a concern | Larger risk window in the event of a single replica failure |
| 3 | Common production configuration for three or more nodes | Higher capacity usage and write load |

Capacity Estimation:

    Actual raw capacity ≈ PVC requested capacity × Number of replicas + Space reserved for snapshots + Recovery + Growth

Example:

    100Gi PVC
    3 replicas
    At least approximately 300Gi of raw capacity
    Additional space must be reserved for snapshots, backups, recovery, and future growth

---

### 8.3 Understanding Volume Status

| Status | Meaning | Action Required |
|---|---|---|
| Healthy | Volume is functioning normally, with sufficient and healthy replicas | Regular monitoring |
| Degraded | Volume availability is reduced; insufficient or faulty replicas | Restore replicas as soon as possible |
| Faulted | Volume is severely damaged and may be unavailable | Preserve the situation and assess backup/restoration options |

Common reasons for Degraded status:

    Node being NotReady
    Unavailable disk
    Failed replicas
    Replicas are in the process of reconstruction
    Number of replicas exceeds available nodes
    Insufficient node disk space
    Longhorn node not allowed to be scheduled

---

## IX. Summary of Snapshot, Backup, and Restore

### 9.1 Differences Among Them

| Capability | Purpose | Can Replace Backup? |
|---|---|---|
| Replica | High availability in the event of node/disk failures | No |
| Snapshot | Quick rollback at a specific point in time | No |
| Backup | Copies data to an external backup target | Nearly equivalent to a true backup |
| Restore | Restores data from a backup to create a new Volume | Provides recovery capability |

Key Conclusion:

    Replicas are not backups.
    Snapshots are not off-site backups.
    The backup target is essential for cross-fault-domain recovery.
    Backups must be verified after restoration to ensure effectiveness.

---

### 9.2 Selection of Backup Targets

Longhorn supports the following backup targets:

    NFS
    S3-compatible object storage
    MinIO
    SMB/CIFS
    Cloud object storage

Key practices in this module:

    Using NFS as a backup target
    Utilizing MinIO/S3 as backup targets

Production recommendations:

    Keep the backup target separate from Longhorn data nodes.
    Ensure the backup target is not located within the same fault domain.
    When using MinIO as a backup target, assign it a dedicated bucket, user, and minimal permission policy.
    Avoid using the MinIO root account for Longhorn backup purposes.

---

### 9.3 Key Points for Using MinIO as a Backup Target

Example for Bucket:

    longhorn-backup

Example for Secret:

    apiVersion: v1
    kind: Secret
    metadata:
      name: longhorn-s3-backup-secret
      namespace: longhorn-system
    type: Opaque
    stringData:
      AWS_ACCESS_KEY_ID: "    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system get engines.longhorn.io -o wide
    kubectl -n longhorn-system get instancemanagers.longhorn.io
    kubectl -n longhorn-system get nodes.longhorn.io

Logs:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=300
    kubectl -n longhorn-system logs <csi-pod-name> --tail=100
    kubectl -n longhorn-system logs <instance-manager-pod-name> --tail=100
    journalctl -u kubelet --since "30 minutes ago" | tail -200
    journalctl -u containerd --since "30 minutes ago" | tail -200

Nodes:

    systemctl status kubelet
    systemctl status containerd
    systemctl status iscsid
    iscsiadm --version
    df -hT
    lsblk -f
    dmesg | tail -100
    journalctl -k --since "1 hour ago" | tail -100

---

## Chapter 11: Performance and Production Recommendations Summary

### 11.1 Factors Affecting Longhorn Performance

Longhorn's performance is primarily influenced by:

    Disk performance
    Network between nodes
    Number of replicas
    Replica rebuild process
    Number of snapshots
    Workload I/O pattern
    Node CPU/memory capabilities
    Resources allocated to Longhorn components
    Resource competition among business pods

Impact pathway:

    Pod writes
      |
      v
    Longhorn Engine
      |
      +--> Replica 1
      +--> Replica 2
      +--> Replica 3
      |
      v
    Multi-node disk and network interactions

An increasing number of replicas results in:

    Higher capacity usage.
    More significant write amplification.
    Increased network synchronization load.
    Potential higher write latency.
    Generally better availability.

---

### 11.2 Principle of Using Separate Data Disks

Production recommendations include:

    Utilize dedicated data disks for Longhorn storage.
    Avoid using system disks to store production data.
    Specify the path as /data/longhorn or a clear directory.
    Choose filesystems like xfs or ext4 for data disks.
    Mount and configure the disk in /etc/fstab.
    Set up disk capacity alerts and monitor I/O performance.

Risks:

    A full system disk can affect kubelet operations.
    Containerd failures can impact all pods.
    High DiskPressure may cause pod eviction.
    Abnormalities in Longhorn replicas could occur.
    Volume degradation might be possible.

---

### 11.3 Suggestions for Layered StorageClass Management

It is recommended to create different StorageClasses based on usage:

| StorageClass | Number of Replicas | Use Cases |
|---------------|-----------------|-------------|
| longhorn-standard | 2 | General business use cases |
| longhorn-ha | 3 | Critical or high-value business scenarios |
| longhorn-ssd | 2 or 3 | High-performance disk-intensive applications |
| longhorn-local-performance | 1 | Performance-oriented scenarios where the application has its own replication mechanism |

Example parameters:

    numberOfReplicas
    dataLocality
    diskSelector
    nodeSelector
    staleReplicaTimeout
    fsType
    recurringJobSelector

Production advice:

    It is not advisable to use the default longhorn StorageClass for all applications.
    Do not arbitrarily modify the default settings.
    Clearly specify the storageClassName for different business needs.
    Changes to StorageClasses typically affect only newly created PVCs and do not automatically update existing volumes.

---

### 11.4 Understanding fio Testing

fio can be used for fundamental I/O performance testing.

Testing areas include:

    Sequential writing
    Sequential reading
    Random writing
    Random reading
    Mixed read and write operations

Notes:

    fio generates significant I/O stress.
    Do not use it to directly test production PVCs.
    Test results are affected by disk, network conditions, number of replicas, caching mechanisms, and workload.
    A single test does not equate to a formal performance evaluation report.
    Formal testing requires a controlled environment, repeated measurements, and comprehensive monitoring.

---

## Chapter 12: Summary of Longhorn's Applicable Scenarios

### 12.1 Recommended Scenarios

Longhorn is suitable for:

    Small to medium-sized Kubernetes clusters
    Private deployment environments
    Bare metal K8s setups
    Edge clusters
    Development and testing clusters
    General stateful applications
    Jenkins home directories
    Nacos data storage
    Redis persistence solutions
    Medium to small-scale Prometheus instances
    Small-scale MySQL/PostgreSQL| Longhorn UI | Not exposed to the public network |  |

---

### 14.2 Installation Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| Helm Version | Recorded |  |
| Longhorn Version | Fixed |  |
| values.yaml | Saved |  |
| Image List | Extracted |  |
| Private Image Repository | Planned or Synchronized |  |
| longhorn-system Pod | Running |  |
| CRD | Created |  |
| StorageClass | Longhorn exists |  |
| Test PVC | Bindable |  |
| Test Pod | Can mount PVC |  |

---

### 14.3 Data Protection Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| Backup Target | Configured |  |
| S3/NFS Access | Normal |  |
| Minimum Permissions | Configured |  |
| Recurring Backup | Configured |  |
| Snapshot Strategy | Planned |  |
| Restore Drill | Completed |  |
| Data Verification | Passed |  |
| Deletion Approval | Established |  |

---

### 14.4 Monitoring Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| Volume Healthy/Degraded/Faulted | Monitored |  |
| Replica Status | Monitored |  |
| Rebuild Status | Monitored |  |
| Node Status | Monitored |  |
| Disk Capacity | Monitored |  |
| CSI Components | Monitored |  |
| Backup Failure | Alarmed |  |
| DiskPressure | Alarmed |  |
| PVC Pending | Alarmed |  |

---

### 14.5 Security Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| UI Access | Private Network/VPN/Bastion Host |  |
| UI Authentication | Configured |  |
| UI HTTPS | Configured |  |
| Regular User Permissions | Cannot delete volumes |  |
| Secret Management | Does not submit to public Git |  |
| MinIO Backup User | Minimum permissions |  |
| High-Risk Operations | Require approval |  |
| Operation Logs | Traceable |  |

---

## Section Fifteen: Daily Inspection Checklist

### 15.1 Daily Inspections

Perform:

    kubectl get nodes -o wide
    kubectl -n longhorn-system get pods -o wide
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system get nodes.longhorn.io
    kubectl get pvc -A
    kubectl get events -A --sort-by=.lastTimestamp | tail -100

For node checks:

    df -hT /data/longhorn
    systemctl is-active iscsid
    systemctl is-active kubelet
    systemctl is-active containerd

---

### 15.2 Weekly Inspections

Recommended checks include:

    Whether backups were successful.
    Capacity of the Backup Target.
    Whether Snapshots have accumulated.
    Whether any volumes are in a degraded state for an extended period.
    Any abnormalities in Replica Rebuilds.
    Trend in data disk capacity growth.
    Security of Longhorn UI access.
    Presence of any abnormal PVC/PVs.
    Incidents of Pending or FailedMount events lasting for an extended time.

---

### 15.3 Monthly Inspections

Recommended actions include:

    Conducting restore drills.
    Reviewing backup strategies.
    Checking the usage of StorageClasses.
    Reassessing node disk capacity planning.
    Evaluating Longhorn versions and upgrade announcements.
    Reviewing records of high-risk operations.
    Conducting post-fault drill reviews.
    Testing the effectiveness of monitoring alerts.

---

## Section Sixteen: Summary of High-Risk Operations

The following operations must be carried out with extreme caution in a production environment:

    Deleting PVCs
    Deleting PVs
    Deleting Longhorn volumes
    Deleting Replicas
    Deleting Engines
    Deleting Snapshots
    Deleting Backups
    Clearing the Backup Target
    Deleting the longhorn-system namespace
    Deleting Longhorn CRDs
    Deleting /data/longhorn
    Formatting Longhorn data disks
    Modifying the default StorageClass
    Changing the number of replicas
    Adjusting dataLocality settings
    Restarting multiple Longhorn data nodes simultaneously
    Continuing to drain other nodes while a volume is in a degraded state
    Performing FIO stress tests on production PVCs
    Conducting fault drills without backups

Before executing any of these operations, it is essential to confirm:

    Whether the operation will affect business operations.
    To which business the specific PVC belongs.
    Which volume the PV corresponds to.
    Whether there is a backup for the Volume.
    Whether a restore drill has been completed.
    Whether there is a maintenance window### Fault Diagnosis

In terms of fault diagnosis, I will conduct a tiered investigation at the business layer, Kubernetes layer, CSI layer, Longhorn control plane, data plane, node layer, and backup layer. For PVC Pending issues, I will check StorageClass, CSI, manager, and node disks; for Pod FailedMount problems, I will examine Events, kubelet, iscsid, and CSI plugins; and for Volume Degraded cases, I will review Volume, Replica, Node, Disk, Rebuild, and manager logs.

### Production Recommendations

I would not recommend using Longhorn as a universal high-performance storage solution in production environments. Instead, independent data disks should be utilized, along with stable node networks, appropriate numbers of replicas, Backup Targets, monitoring and alert mechanisms, and recovery drills. Longhorn is suitable for small to medium-sized Kubernetes applications that require stateful storage, as well as scenarios involving Jenkins, Prometheus, Nacos, Redis, and smaller-scale databases. For core high-concurrency databases or systems requiring high IOPS and low latency, careful evaluation is necessary, taking into account the database's own backup and replication mechanisms.

---

## Chapter 19: Skills Acquired in This Phase

After completing the Longhorn phase, you should possess the following abilities:

1. Be able to explain what Longhorn is.
2. Distinguish between Longhorn, MinIO, Ceph, and NFS.
3. Understand the relationships between PV, PVC, StorageClass, and CSI.
4. Be capable of planning Longhorn nodes, disks, and data directories.
5. Know how to install open-iscsi/iscsid.
6. Be able to use Helm for installing Longhorn.
7. Know how to view Charts, values.yaml files, and the image list.
8. Use values.yaml to manage the image repository and default parameters.
9. Create PVCs and verify dynamic PV creation.
10. Create Pods that mount PVCs and confirm data persistence.
11. Understand the differences between Deployment and StatefulSet in using PVCs.
12. Comprehend concepts such as Replica count, node distribution, and Volume Degraded.
13. Be able to observe node failures and Replica Rebuild processes.
14. Configure Backup Targets.
15. Understand the boundaries between Snapshot, Backup, and Restore.
16. Restore new volumes from backups.
17. Diagnose issues such as PVC Pending, FailedMount, Multi-Attach, and Volume Degraded.
18. Perform basic fio tests.
19. Design StorageClass hierarchies effectively.
20. Develop an acceptance checklist for Longhorn in production environments.
21. Establish processes for inspection, monitoring, alerting, backup, and recovery drills.

---

## Chapter 20: Summary of This Article

This article summarizes the key points covered during the Longhorn phase:

1. Longhorn is a Kubernetes-native distributed block storage system.
2. It provides dynamic PV/PVC functionality through CSI.
3. PVCs, PVs, StorageClass, and CSI form the core components of Longhorn's storage framework.
4. The Longhorn control plane is primarily managed by longhorn-manager.
5. The data plane consists of Engine, Replica, and Instance Manager.
6. Helm is an efficient way to deploy Longhorn in production settings.
7. Before installation, it is essential to check the status of nodes, disks, iscsid, StorageClass, and the Kubernetes cluster itself.
8. In domestic network environments, values.yaml should be used to manage the image repository rather than altering containerd settings arbitrarily.
9. The deletion of a Pod does not necessarily result in the deletion of its data; only PVCs and Longhorn volumes store actual data.
10. RWO PVCs are not suitable for simultaneous read and write operations across multiple nodes.
11. Stateful applications with multiple replicas perform better using StatefulSet + volumeClaimTemplates.
12. Replicas serve as a means to enhance availability, not as a backup mechanism.
13. Snapshots provide protection at a specific point in time but are not considered off-site backups.
14. Backup Targets are crucial for ensuring cross-fault-domain recovery with Longhorn.
15. Restoring data should prioritize new volumes before verifying and switching back to the original service.
16. Volume Degraded issues must not be ignored for extended periods.
17. In the event of a volume failure, the primary objective is to preserve the current state of the data.
18. Longhorn's performance is influenced by disks, network conditions, the number of replicas, and the workload.
19. Production environments require independent data disks, monitoring and alert systems, backup strategies, and recovery drills.
20. While Longhorn is suitable for small to medium-sized Kubernetes applications with stateful storage needs, it is not appropriate as a substitute for object storage or for core high-concurrency databases.
21. To date, the Longhorn phase has completed a comprehensive