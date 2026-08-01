# Longhorn Phase Summary: From K8s Storage Plugin to Production Operations

Recommended path: 05-Storage/03-LongHorn/99-Longhorn Phase Summary: From K8s Storage Plugin to Production Operations.md

Tags: #Longhorn #Kubernetes #CSI #PV #PVC #StorageClass #BlockStorage #Replica #BackupTarget #Snapshot #FaultCheck. #PerformanceOptimization #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This document is a phase summary for the Longhorn module, used to conclude the entire Longhorn learning phase.

Previously completed:

- 00-Longhorn Directory Index
- 01-Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- 02-Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- 03-Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass
- 04-Longhorn Helm Installation Methodology: Chart, Images, values.yaml, and Version Management
- 05-Longhorn Dynamic Volume Practice: PVC, PV, Pod Mounting, and Data Persistence
- 06-Longhorn Replica Mechanism: Replica Count, Node Distribution, and Data High Availability
- 07-Longhorn Backup and Recovery: Backup Target, Snapshot, and Volume Restore
- 08-Longhorn Troubleshooting: Volume Degraded, Replica Rebuilding, and Node Anomalies
- 09-Longhorn Performance and Production Recommendations: Disks, Network, Scheduling, and Resource Limits

This document focuses on summarizing:

    What problems Longhorn actually solves
    Longhorn's positioning in the Kubernetes storage ecosystem
    The relationship between PV/PVC/StorageClass/CSI and Longhorn
    How Longhorn's core components collaborate
    Understanding Longhorn's replica mechanism
    Distinguishing between Snapshot, Backup, and Restore
    How to plan for Longhorn in production environments
    How to troubleshoot Longhorn issues
    Where Longhorn's performance bottlenecks lie
    What scenarios are suitable for Longhorn
    What scenarios are not suitable for Longhorn
    How to advance from "knowing how to install" to "being able to perform production operations"

---

## II. Phase Positioning

Longhorn is a Kubernetes cloud-native distributed block storage system.

Its core positioning is:

    Providing dynamic PV/PVC capabilities for Kubernetes
    Connecting with Kubernetes storage ecosystem via CSI
    Organizing node-local disks into persistent block storage volumes usable by Pods
    Offering Volume, Replica, Snapshot, Backup, and Restore capabilities for stateful applications

One-sentence understanding:

    Longhorn is Kubernetes's internal "distributed cloud hard disk system."

It is NOT:

    Object storage
    Traditional NAS
    Ordinary NFS
    High-end SAN storage
    A universal database high-performance storage
    A complete replacement for Ceph

It primarily solves:

    The problem of persistent data needed by Pods in Kubernetes
    The issue of no cloud disks in bare-metal/private K8s clusters
    The problem of dynamic PVC supply for medium-scale stateful applications
    The issue of Volume replica recovery when nodes or disks fail
    The problem of snapshots, backups, and recovery within K8s internal storage

---

## III. Relationship Between Longhorn and Kubernetes Storage Ecosystem

### 3.1 Kubernetes Core Storage Objects

Kubernetes core storage objects include:

| Object | Meaning |
|---|---|
| StorageClass | Template for dynamically creating PV |
| PVC | User's request for storage resources |
| PV | Persistent volume resource in Kubernetes |
| CSI | Standard interface for Kubernetes to connect to external storage systems |
| Pod | Business runtime unit using PVC |

Longhorn accesses Kubernetes via CSI.

Complete flow:

    User creates PVC
        |
        v
    PVC specifies storageClassName: longhorn
        |
        v
    Kubernetes calls Longhorn CSI
        |
        v
    Longhorn creates Volume
        |
        v
    Kubernetes automatically creates PV
        |
        v
    PVC and PV are bound
        |
        v
    Pod mounts PVC
        |
        v
    Application reads/writes to mounted directory
        |
        v
    Data is written to Longhorn Volume

---

### 3.2 Longhorn Dynamic Volume Creation Flow

Dynamic volume flow:

    PVC
      |
      v
    StorageClass: longhorn
      |
      v
    CSI Provisioner
      |
      v
    Longhorn Manager
      |
      v
    Longhorn Volume
      |
      v
    Engine + Replica
      |
      v
    Kubernetes PV
      |
      v
    Pod Mount

When troubleshooting operations, reverse tracing is required:

    Pod
      |
      v
    PVC
      |
      v
    PV
      |
      v
    Longhorn Volume
      |
      v
    Engine / Replica
      |
      v
    Node / Disk

---

## IV. Summary of Longhorn Core Architecture

### 4.1 Architecture Layers

Longhorn can be divided into three layers:

    Kubernetes Access Layer
    Longhorn Control Plane
    Longhorn Data Plane

Architecture diagram: /think

```
┌───────────────────────────────────────────────┐
│ Kubernetes                                    │
│                                               │
│ Pod -> PVC -> PV -> StorageClass -> CSI       │
└───────────────────────┬───────────────────────┘
                        │
                        v
┌───────────────────────────────────────────────┐
│ Longhorn Control Plane                        │
│                                               │
│ longhorn-manager                              │
│ longhorn-driver-deployer                      │
│ CSI sidecar                                   │
│ Longhorn UI                                   │
│ Longhorn CRD                                  │
└───────────────────────┬───────────────────────┘
                        │
                        v
┌───────────────────────────────────────────────┐
│ Longhorn Data Plane                           │
│                                               │
│ Engine                                        │
│ Replica                                       │
│ Instance Manager                              │
│ Node Disk: /data/longhorn                     │
└───────────────────────────────────────────────┘

---

### 4.2 Core Component Functions

| Component | Function |
|---|---|
| longhorn-manager | Longhorn Control Plane core, manages Volume, Replica, Node, Disk, Backup |
| longhorn-driver-deployer | Deploys CSI-related components |
| longhorn-csi-plugin | Node-side CSI plugin responsible for mounting operations |
| csi-provisioner | Dynamically creates PV/Volume based on PVC |
| csi-attacher | Handles Volume attach/detach |
| csi-resizer | Handles PVC expansion |
| csi-snapshotter | Handles snapshot capabilities |
| Engine | Read/write engine for Volume |
| Replica | Data replica for Volume |
| Instance Manager | Manages Engine/Replica instances |
| Longhorn UI | Management interface |
| Longhorn CRD | Resource representation of Longhorn internal objects in Kubernetes |

---

### 4.3 Longhorn Object Relationships

A PVC may correspond to:

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

Common viewing commands:

    kubectl get pvc -A
    kubectl get pv
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system get engines.longhorn.io -o wide
    kubectl -n longhorn-system get instancemanagers.longhorn.io
    kubectl -n longhorn-system get nodes.longhorn.io

---

## V. Experimental Environment Summary

### 5.1 Cluster Planning

Default experimental environment:

    Kubernetes: kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node Network Segment: 10.0.0.0/24

Node planning:

| IP | Hostname | Role | Longhorn Recommendation |
|---|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane | Experimental participation is possible, production caution advised |
| 10.0.0.21 | k8s-worker01 | Worker | Recommended participation |
| 10.0.0.22 | k8s-worker02 | Worker | Recommended participation |

---

### 5.2 Data Directory Planning

Longhorn's default data directory is commonly:

    /var/lib/longhorn

This module recommends planning as:

    /data/longhorn

Production recommendation: /think

/data/longhorn should use a dedicated data disk.
It is not recommended to place the Longhorn data directory on the system disk.
Using symbolic links instead of proper mounting is not recommended.
The data disk is recommended to use xfs or ext4.
The data disk should be written to /etc/fstab to ensure automatic mounting after reboot.

Check commands:

    df -hT /data/longhorn
    lsblk -f
    mount | grep /data
    du -sh /data/longhorn

---

### 5.3 Summary of Node Dependencies

All nodes planned to run Longhorn Volume must have:

Ubuntu 22.04:

    apt update
    apt install -y open-iscsi nfs-common
    systemctl enable --now iscsid

Rocky Linux 9:

    dnf install -y iscsi-initiator-utils nfs-utils
    systemctl enable --now iscsid

Check:

    systemctl is-active iscsid
    iscsiadm --version
    mount.nfs -V

Notes:

    open-iscsi / iscsid are critical for Longhorn RWO volume mounting.
    NFSv4 client is mainly used for RWX or related shared access scenarios.
    When Pod ContainerCreating / FailedMount occurs, iscsid is a key troubleshooting item.

---

## Six. Helm Installation Methodology Summary

### 6.1 Installation Principles

Longhorn recommends using Helm for installation management.

Production installation should not:

    Install without checking the version
    Install without exporting values.yaml
    Install without checking the image list
    Arbitrarily modify containerd global configuration when image pull fails
    Not save Helm installation records
    Assume installation is complete without verifying PVC

Recommended workflow:

    1. Check Kubernetes cluster status.
    2. Check node open-iscsi / iscsid.
    3. Check data disk and /data/longhorn.
    4. helm repo add longhorn.
    5. helm search repo longhorn/longhorn --versions.
    6. Fix Longhorn version.
    7. helm show values to export default values.yaml.
    8. helm template to render final YAML.
    9. Extract image list.
    10. Synchronize images to private registry in domestic network environment.
    11. Write longhorn-values-prod.yaml.
    12. helm install.
    13. Verify Pod, CRD, StorageClass, PVC, Pod mounting.

---

### 6.2 Image Management Methodology

Domestic network environment recommendation:

    First check the Chart
    Then export values.yaml
    Then use helm template to extract images
    Then synchronize to private registry
    Then specify image repository via values.yaml
    Do not directly modify containerd global configuration

Common commands:

    helm show values longhorn/longhorn --version ${LONGHORN_VERSION} > longhorn-values-default.yaml

    helm template longhorn longhorn/longhorn \
      --namespace longhorn-system \
      --version ${LONGHORN_VERSION} \
      -f longhorn-values-default.yaml \
      > longhorn-rendered-default.yaml

    grep -oE 'image: [^ ]+' longhorn-rendered-default.yaml | awk '{print $2}' | sort -u

---

### 6.3 Post-Installation Verification

Post-installation checks:

    kubectl -n longhorn-system get pods -o wide
    kubectl -n longhorn-system get svc
    kubectl get crd | grep longhorn
    kubectl api-resources | grep -i longhorn
    kubectl get sc
    kubectl describe sc longhorn

Create test PVC:

    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: install-demo-pvc
      namespace: longhorn-install-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn
      resources:
        requests:
          storage: 1Gi

Verification:

    kubectl get pvc -n longhorn-install-demo
    kubectl get pv
    kubectl -n longhorn-system get volumes.longhorn.io
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

---

## Seven. Dynamic Volume Practice Summary

### 7.1 PVC Persistence Verification Path

Dynamic volume verification workflow:

    1. Create PVC.
    2. Confirm PVC Bound.
    3. Confirm PV auto-creation.
    4. Create Pod mounting PVC.
    5. Write file to /data.
    6. Delete Pod.
    7. Rebuild Pod.
    8. Read old file.
    9. Verify data still exists.

Common commands: /think

kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
kubectl get pv
kubectl describe pv <pv-name>
kubectl get pod -n <namespace> -o wide
kubectl exec -n <namespace> <pod-name> -- sh -c "echo test > /data/test.txt"
kubectl exec -n <namespace> <pod-name> -- cat /data/test.txt

---

### 7.2 Differences Between Deployment and StatefulSet

Deployment + PVC:

    Suitable for single-replica applications.
    Multiple replicas sharing the same RWO PVC may cause Multi-Attach issues.
    Not suitable for most stateful multi-replica applications.

StatefulSet + volumeClaimTemplates:

    Each Pod has an independent PVC.
    Suitable for stateful multi-replica applications.
    More suitable for scenarios like MySQL, Redis, Prometheus, ZooKeeper, Kafka, etc.

Core Principles:

    RWO PVC is not recommended for multiple Pods to read/write mount simultaneously across nodes.
    StatefulSet + independent PVC is preferred for stateful multi-replica applications.

---

## VIII. Summary of Replica Mechanism

### 8.1 Problems Solved by Replica

Replica is used to enhance Volume availability.

Can address:

    Single-replica failure
    Single-node failure
    Single-disk failure
    Node temporary offline
    Replica reconstruction
    Recovery after Volume Degraded

Cannot address:

    User accidental file deletion
    Application data corruption
    Accidental PVC deletion
    Accidental Volume deletion
    Cluster-wide failure
    All replicas simultaneously failing

Conclusion:

    Replica is a high availability capability.
    Replica is not backup.

---

### 8.2 Replica Count Planning

| Replica Count | Suitable Scenarios | Risks |
|---|---|---|
| 1 | Learning, temporary, data-rebuildable, applications with built-in replication | No node-level high availability |
| 2 | Two-node experiments, small-to-medium business, capacity-sensitive scenarios | Larger risk window after one replica failure |
| 3 | Common production configuration for three-node or more | Higher capacity consumption and write pressure |

Capacity Estimation:

    Actual raw capacity ≈ PVC requested capacity × replica count + Snapshot reservation + Rebuild reservation + Growth reservation

Example:

    100Gi PVC
    3 replicas
    At least about 300Gi raw capacity
    Also needs to reserve space for Snapshot, Backup, Rebuild, and growth

---

### 8.3 Understanding Volume Status

| Status | Meaning | Handling |
|---|---|---|
| Healthy | Volume is normal, replicas meet expectations | Normal inspection |
| Degraded | Volume availability has decreased, insufficient or abnormal replicas | Restore replicas as soon as possible |
| Faulted | Volume has severe anomalies, possibly unavailable | Preserve the scene, assess Backup / Restore |

Common causes for Degraded:

    Node NotReady
    Disk unavailable
    Replica failure
    Replica under reconstruction
    Replica count exceeds available nodes
    Node disk space insufficient
    Longhorn Node disallows scheduling

---

## IX. Summary of Snapshot, Backup, and Restore

### 9.1 Differences Between the Three

| Capability | Function | Does it replace backup |
|---|---|---|
| Replica | High availability for node/disk failure | No |
| Snapshot | Local point-in-time snapshot of Volume, suitable for quick rollback | No |
| Backup | Backup to external Backup Target | Close to true backup |
| Restore | Restore from Backup to new Volume | Recovery capability |

Core Conclusion:

    Replica is not backup.
    Snapshot is not cross-site backup.
    Backup Target is the foundation for cross-fault-domain recovery.
    Backup must be validated through recovery to be considered a closed loop.

---

### 9.2 Backup Target Selection

Longhorn Backup Target can use:

    NFS
    S3-compatible object storage
    MinIO
    SMB/CIFS
    Cloud object storage

This module's focus practice:

    NFS Backup Target
    MinIO / S3 Backup Target

Production recommendations:

    Backup Target should be separated from Longhorn data nodes.
    Backup Target should not be in the same fault domain.
    When using MinIO as Backup Target, use independent Bucket, independent user, and minimal permission Policy.
    Do not use MinIO root account as Longhorn backup credentials.

---

### 9.3 Key Points for Using MinIO as Backup Target

Bucket example:

    longhorn-backup

Secret example:

    apiVersion: v1
    kind: Secret
    metadata:
      name: longhorn-s3-backup-secret
      namespace: longhorn-system
    type: Opaque
    stringData:
      AWS_ACCESS_KEY_ID: "longhorn-backup-user"
      AWS_SECRET_ACCESS_KEY: "LonghornBackup@123456"
      AWS_ENDPOINTS: "http://10.0.0.41:9000"
      AWS_REGION: "us-east-1"

Backup Target example:

    s3://longhorn-backup@us-east-1/

Security requirements:

    Secret should not be submitted to public Git.
    AccessKey should have minimal permissions.
    Backup Bucket should not be mixed with business Bucket.
    Do not manually delete objects inside backupstore.
    Backup lifecycle should be managed through Longhorn primarily.

---

### 9.4 Standard Recovery Drill Process

1. Create a test Backup.
2. Restore to a new Volume from the Backup.
3. Create a new PVC.
4. Create a validation Pod.
5. Verify critical files.
6. Record recovery duration.
7. Validate if the application can start.
8. Clean up exercise resources.
9. Output a recovery exercise report.

Recovery Principles:

    Do not directly overwrite the original business.
    Prioritize recovery to a new Volume.
    Switch the business only after verification.
    Original PVC and original Volume are retained for a period.

---

## Ten, Troubleshooting Summary

### 10.1 Troubleshooting Layers

Longhorn troubleshooting recommends layered approach:

    1. Business layer: Is Pod Running, is the application readable/writable.
    2. Kubernetes layer: PVC / PV / StorageClass status.
    3. CSI layer: csi-provisioner / csi-attacher / csi-plugin status.
    4. Longhorn control plane: longhorn-manager status.
    5. Longhorn data plane: Engine / Replica / Instance Manager status.
    6. Node layer: kubelet / containerd / iscsid / disk / network status.
    7. Backup layer: Snapshot / Backup / Backup Target availability.

---

### 10.2 Common Troubleshooting Table

| Fault Phenomenon | First Entry Point | Deeper Investigation Directions |
|---|---|---|
| PVC Pending | kubectl describe pvc | StorageClassI don't know.CSII don't know.node diskI don't know.manager |
| Pod ContainerCreating | kubectl describe pod | FailedMountI don't know.iscsidI don't know.CSI Plugin |
| FailedMount | Events / kubelet logs | iSCSII don't know.Volume attachI don't know.CSI |
| Multi-Attach | Pod distribution / Events | RWO PVC used by multiple nodes |
| Volume Degraded | volumes.longhorn.io | ReplicaI don't know.NodeI don't know.DiskI don't know.Rebuild |
| Replica Rebuilding Stuck | replicas.longhorn.io | DiskI don't know.networkI don't know.manager logs |
| Volume Faulted | Volume status | Preserve sceneI don't know.check BackupI don't know.Restore |
| Backup Failed | Longhorn UI / manager logs | NFS / S3 / Secret / Permissions |
| DiskPressure | describe node / df | System diskI don't know.data diskI don't know.containerdI don't know.logs |

---

### 10.3 High-Frequency Command Summary

Kubernetes:

    kubectl get nodes -o wide
    kubectl describe node <node-name>
    kubectl get pods -A -o wide
    kubectl get pvc -A
    kubectl get pv
    kubectl get sc
    kubectl get events -A --sort-by=.lastTimestamp | tail -100

Longhorn:

    kubectl -n longhorn-system get pods -o wide
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
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

## Eleven, Performance and Production Recommendations Summary

### 11.1 Longhorn Performance Factors

Longhorn performance mainly affected by:

    Disk performance
    Node-to-node network
    Replica count
    Replica Rebuild
    Snapshot count
    Workload I/O model
    Node CPU / Memory
    Longhorn component resources
    Business Pod resource contention

Impact path:

    Pod write
      |
      v
    Longhorn Engine
      |
      +--> Replica 1
      +--> Replica 2
      +--> Replica 3
      |
      v
    Multi-node disk and network

Higher replica count:

    Higher capacity usage.
    More pronounced write amplification.
    Higher network synchronization pressure.
    Potential higher write latency.
    Usually better availability.

---

### 11.2 Independent Data Disk Principle

Production recommendation: /think

Use an independent data disk.  
Do not use the system disk to carry production Longhorn data.  
Use /data/longhorn or a specific path.  
Data disks use xfs or ext4.  
Mount write to /etc/fstab.  
Set disk capacity alerts.  
Monitor disk I/O.  

Risks:  

    System disk fullness will affect kubelet.  
    Containerd anomalies will affect all Pods.  
    DiskPressure will cause Pod eviction.  
    Longhorn Replica may be abnormal.  
    Volume may be Degraded.  

---

### 11.3 StorageClass Tiering Recommendations  

Recommend creating different StorageClass for different purposes:  

| StorageClass | Number of Replicas | Scenario |  
|---|---|---|  
| longhorn-standard | 2 | General business |  
| longhorn-ha | 3 | More important business |  
| longhorn-ssd | 2 or 3 | High-performance disk business |  
| longhorn-local-performance | 1 | Performance scenarios where the application itself has replication capabilities |  

Example parameters:  

    numberOfReplicas  
    dataLocality  
    diskSelector  
    nodeSelector  
    staleReplicaTimeout  
    fsType  
    recurringJobSelector  

Production recommendations:  

    Not recommended to use default longhorn for all business.  
    Not recommended to arbitrarily modify the default StorageClass.  
    Different businesses should explicitly specify storageClassName.  
    StorageClass changes usually only affect newly created PVCs and do not automatically change old Volumes.  

---

### 11.4 fio Test Understanding  

fio can be used for basic I/O testing.  

Test content:  

    Sequential write  
    Sequential read  
    Random write  
    Random read  
    Mixed read/write  

Notes:  

    fio will generate I/O pressure.  
    Do not directly stress-test production PVCs.  
    Test results are affected by disk, network, replica count, cache, and load.  
    A single test is not equivalent to an official performance report.  
    Official stress testing requires a fixed environment, repeated testing, and monitoring support.  

---

## Twelve, Longhorn Use Case Summary  

### 12.1 Recommended Scenarios  

Longhorn is suitable for:  

    Medium-to-small Kubernetes clusters  
    Private deployment environments  
    Bare metal K8s  
    Edge clusters  
    Development/test clusters  
    General stateful applications  
    Jenkins home  
    Nacos data directory  
    Redis persistence  
    Prometheus medium-to-small instances  
    Small-scale MySQL / PostgreSQL  
    Clusters needing PVC dynamic provisioning  

---

### 12.2 Cautionary Scenarios  

Need to be cautious:  

    Core high-concurrency databases  
    High IOPS low-latency business  
    Large-scale GitLab  
    High-write Prometheus  
    Large-scale Elasticsearch  
    Kafka high-throughput scenarios  
    Cross-datacenter replica synchronization  
    Nodes with poor network quality  
    HDD carrying high random writes  
    Production environments without Backup Target  

---

### 12.3 Not Recommended Scenarios  

Not recommended:  

    Use Longhorn to replace MinIO for object storage.  
    Use Longhorn to store massive images, attachments, and archive objects.  
    Use Longhorn to store large-scale artifacts.  
    Use single-node Longhorn to fake high availability.  
    Use system disk to carry production data.  
    Production environments without backup targets for core databases.  
    Production environments without monitoring alerts.  
    Carry critical business without recovery drills.  

---

## Thirteen, Longhorn Comparison with Other Storage Modules  

### 13.1 Longhorn vs MinIO  

| Comparison Item | Longhorn | MinIO |  
|---|---|---|  
| Type | Block storage | Object storage |  
| Access Method | PVC / CSI | S3 API |  
| Usage Objects | Pod / StatefulSet | Application / SDK / mc |  
| Data Unit | Volume | Object |  
| Top-level Container | PVC / PV | Bucket |  
| Typical Use Cases | Database data disk, application data directory | Images, attachments, backup packages, log archives |  
| Protocol | Kubernetes CSI / File system mounting | HTTP / HTTPS S3 |  
| Suitable for Massive Objects | Not suitable | Suitable |  

One-sentence summary:  

    Pods need a persistent data disk, use Longhorn.  
    Applications need object upload/download interfaces, use MinIO.  

---

### 13.2 Longhorn vs Ceph RBD  

| Comparison Item | Longhorn | Ceph RBD |  
|---|---|---|  
| Positioning | K8s cloud-native block storage | General-purpose distributed block storage |  
| Deployment Complexity | Low | High |  
| Operation Complexity | Medium | High |  
| Architecture Core | Volume / Engine / Replica | RADOS / OSD / PG / CRUSH |  
| K8s Integration | Native around Kubernetes | Through CSI integration |  
| Suitable Scale | Medium-to-small K8s | Medium-to-large unified storage platform |  
| Learning Focus | CSI, Replica, Backup | OSD, PG, Pool, CRUSH |  

One-sentence summary:  

    Longhorn is lighter and closer to Kubernetes.  
    Ceph is more general-purpose, more complex, and better suited for unified storage foundation.  

---

### 13.3 Longhorn vs NFS  

| Comparison Item | Longhorn | NFS |  
|---|---|---|  
| Type | Distributed block storage | Network file system |  
| K8s Access | CSI / PVC | NFS PV / NFS CSI |  
| Data Model | Volume | Shared directory |  
| High Availability | Longhorn replicas | Depends on NFS Server high availability |  
| Typical Use Cases | RWO block volumes | RWX shared files |  
| Risks | Replicas, reconstruction, node network | NFS single point, permissions, performance bottlenecks |  

One-sentence summary:  

    Longhorn is more suitable for block volumes.  
    NFS is more suitable for shared directories, but NFS high availability needs to be resolved separately.  

---

## Fourteen, Production Launch Acceptance Checklist  

### 14.1 Architecture Acceptance /think

| Check Item | Requirement | Result |
|---|---|---|
| Node Count | At least 2 data nodes, production recommends 3 or more |  |
| Data Disk | Independent data disk |  |
| Data Directory | /data/longhorn or explicit path |  |
| System Disk | Does not carry production data |  |
| Network | Stable, low-latency between nodes |  |
| iscsid | All related nodes are running |  |
| NFS Client | RWX scenario is installed |  |
| Longhorn UI | Not exposed to public internet |  |

---

### 14.2 Installation Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| Helm Version | Recorded |  |
| Longhorn Version | Fixed |  |
| values.yaml | Saved |  |
| Image List | Extracted |  |
| Private Image Registry | Planned or synchronized |  |
| longhorn-system Pod | Running |  |
| CRD | Created |  |
| StorageClass | longhorn exists |  |
| Test PVC | Can be Bound |  |
| Test Pod | Can mount PVC |  |

---

### 14.3 Data Protection Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| Backup Target | Configured |  |
| S3/NFS Access | Normal |  |
| Minimum Permissions | Configured |  |
| Recurring Backup | Configured |  |
| Snapshot Policy | Planned |  |
| Restore Drill | Completed |  |
| Data Validation | Passed |  |
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
| Backup Failure | Alerted |  |
| DiskPressure | Alerted |  |
| PVC Pending | Alerted |  |

---

### 14.5 Security Acceptance

| Check Item | Requirement | Result |
|---|---|---|
| UI Access | Intranet/VPN/Fortress Machine |  |
| UI Authentication | Configured |  |
| UI HTTPS | Configured |  |
| Regular User Permissions | Cannot delete Volume |  |
| Secret Management | Not submitted to public Git |  |
| MinIO Backup User | Minimum permissions |  |
| High-Risk Operations | Requires approval |  |
| Operation Records | Traceable |  |

---

## FifteenI don't know.Daily Inspection Checklist

### 15.1 Daily Inspection

Execute:

    kubectl get nodes -o wide
    kubectl -n longhorn-system get pods -o wide
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system get nodes.longhorn.io
    kubectl get pvc -A
    kubectl get events -A --sort-by=.lastTimestamp | tail -100

Node side:

    df -hT /data/longhorn
    systemctl is-active iscsid
    systemctl is-active kubelet
    systemctl is-active containerd

---

### 15.2 Weekly Inspection

Recommended checks:

    Whether backup succeeded.
    Backup Target capacity.
    Snapshot accumulation.
    Volume long-term Degraded.
    Replica Rebuild anomalies.
    Data disk capacity growth trend.
    Longhorn UI access security.
    Abnormal PVC/PV.
    Long-time Pending/FailedMount events.

---

### 15.3 Monthly Inspection

Recommended execution:

    Restore drill.
    Backup policy review.
    StorageClass usage review.
    Node disk capacity planning review.
    Longhorn version and upgrade announcement assessment.
    High-risk operation records review.
    Failure drill review.
    Monitoring alert effectiveness test.

---

## SixteenI don't know.High-Risk Operations Summary

The following operations must be cautious in production:

    Delete PVC
    Delete PV
    Delete Longhorn Volume
    Delete Replica
    Delete Engine
    Delete Snapshot
    Delete Backup
    Clear Backup Target
    Delete longhorn-system namespace
    Delete Longhorn CRD
    Delete /data/longhorn
    Format Longhorn data disk
    Modify default StorageClass
    Modify replica count
    Modify dataLocality
    Simultaneously restart multiple Longhorn data nodes
    Continue drain other nodes when Volume is Degraded
    Perform fio stress test on production PVC
    Conduct failure drill without backup

Before execution, must confirm:

    Whether it affects business.
    Which business the PVC belongs to.
    Which Volume the PV corresponds to.
    Whether the Volume has Backup.
    Whether restore drill is completed.
    Whether there is a maintenance window.
    Whether there is a rollback plan.
    Whether business confirmation is obtained.
    Whether dual-person review is passed.
    Whether on-site information is preserved.

| Item | Content |
|---|---|
| Fault Time |  |
| Discovery Method | Alert / Patrol / Business Feedback |
| Impact on Business |  |
| Namespace |  |
| PVC |  |
| PV |  |
| Longhorn Volume |  |
| Volume Status | Healthy / Degraded / Faulted |
| Replica Status |  |
| Affected Nodes |  |
| Does It Affect Read | Yes / No |
| Does It Affect Write | Yes / No |
| Is There a Snapshot | Yes / No |
| Is There a Backup | Yes / No |
| Backup Target | NFS / S3 / MinIO |
| Handling Actions |  |
| Recovery Time |  |
| Root Cause |  |
| Subsequent Improvements |  |

---

## EighteenI don't know.Interview Answering Strategy

If asked in an interview:

    What level of proficiency do you have with Longhorn?

You can respond:

    I have systematically studied and organized the complete chain of Longhorn from basics, architecture, installation, dynamic volumes, replicas, backup recovery, fault diagnosis, and production recommendations.
    Longhorn is a Kubernetes cloud-native distributed block storage system, primarily providing dynamic PV/PVC capabilities to Kubernetes through CSI. After users create a PVC and specify the longhorn StorageClass, Kubernetes will create a Longhorn Volume via Longhorn CSI, automatically generate a PV, and Pods mounting PVCs can read/write data like ordinary directories.
    Architecturally, I understand Longhorn can be divided into Kubernetes integration layer, control plane, and data plane. The control plane core is longhorn-manager, responsible for managing Volumes, Replicas, Nodes, Disks, Snapshots, and Backups; the data plane includes Engine, Replica, and Instance Manager. Engine handles Volume read/write, Replica is the data replica of a Volume, and Instance Manager manages Engine and Replica instances.
    For installation, I prefer Helm over direct one-click YAML. Before installation, I check Kubernetes nodes, kube-system components, containerd, kubelet, open-iscsi/iscsid, NFS client, data disks, and StorageClass. In domestic network environments, I first use helm show values and helm template to extract image lists, then synchronize them to Harbor or Alibaba Cloud image repositories, specifying image repositories via values.yaml rather than arbitrarily modifying containerd global configurations.
    For dynamic volume practice, I create PVCs, confirm PVC Bound, PV creation, Longhorn Volume creation, then use Pods to mount PVCs and write data. After deleting Pods and rebuilding them, I verify data persistence. For multi-replica stateful applications, I prioritize StatefulSet + volumeClaimTemplates over multiple Deployment replicas sharing an RWO PVC.
    Regarding replica mechanisms, I understand Longhorn Replicas provide high availability to handle single-node or single-disk failures. Volumes may have Healthy, Degraded, or Faulted states. Degraded indicates insufficient or abnormal replicas, which cannot be ignored long-term; Faulted indicates severe anomalies requiring on-site preservation and assessment of recovery from Backup. Replica counts should be planned based on node numbers, capacity, and performance, not necessarily higher is better.
    For backup recovery, I clearly distinguish between Replica, Snapshot, and Backup. Replica is not a backup, and Snapshot is notAlien.backup. Production must configure Backup Targets like NFS or S3/MinIO. During recovery, I prioritize restoring from Backup to a new Volume, then creating a new PVC and validating Pods to check data, rather than directly overwriting original business.
    For fault diagnosis, I troubleshoot layer by layer: business layer, Kubernetes layer, CSI layer, Longhorn control plane, data plane, node layer, and backup layer. PVC Pending checks StorageClass, CSI, manager, and node disks; Pod FailedMount checks Events, kubelet, iscsid, and CSI Plugin; Volume Degraded checks Volume, Replica, Node, Disk, Rebuild, and manager logs.
    For production recommendations, I don't treat Longhorn as a universal high-performance storage. Production should use dedicated data disks, stable node networks, reasonable replica counts, Backup Targets, monitoring alerts, and recovery drills. Longhorn suits small-to-medium K8s stateful apps, Jenkins, Prometheus, Nacos, Redis, small databases, etc. Core high-concurrency databases and high IOPS low-latency scenarios require cautious evaluation combined with database-specific backup and replication mechanisms.

---

## NineteenI don't know.Capabilities Gained in This Phase

After completing the Longhorn phase, you should possess the following capabilities:

1. Explain what Longhorn is.
2. Differentiate Longhorn, MinIO, Ceph, and NFS.
3. Understand the relationship between PV, PVC, StorageClass, and CSI.
4. Plan Longhorn nodes, disks, and data directories.
5. Install open-iscsi / iscsid.
6. Use Helm to install Longhorn.
7. View Chart, values.yaml, and image lists.
8. Manage image repositories and default parameters via values.yaml.
9. Create PVCs and validate dynamic PV creation.
10. Create Pods mounting PVCs and validate data persistence.
11. Understand differences between Deployment and StatefulSet in PVC usage.
12. Understand Replica counts, node distribution, and Volume Degraded states.
13. Observe node failures and Replica Rebuild.
14. Configure Backup Targets.
15. Understand boundaries between Snapshot, Backup, and Restore.
16. Recover new Volumes from Backup.
17. Troubleshoot PVC Pending, FailedMount, Multi-Attach, and Volume Degraded.
18. Perform basic fio testing.
19. Design StorageClass tiering.
20. Create a Longhorn production deployment acceptance checklist.
21. Establish inspection, monitoring, alerting, backup, and recovery drill processes.

---

## TwentyI don't know.Summary of This Section

This article completes the summary of the Longhorn phase:

1. Longhorn is a Kubernetes cloud-native distributed block storage system.  
2. Longhorn provides dynamic PV/PVC capabilities through CSI.  
3. PVC, PV, StorageClass, and CSI are the core components of Longhorn's usage chain.  
4. The control plane core of Longhorn is longhorn-manager.  
5. The data plane core of Longhorn consists of Engine, Replica, and Instance Manager.  
6. Helm is the more suitable method for production deployment of Longhorn.  
7. Pre-installation checks must include nodes, disks, iscsid, StorageClass, and Kubernetes cluster status.  
8. In domestic network environments, image repositories should be managed via values.yaml; avoid arbitrarily modifying containerd.  
9. Pod deletion does not equate to data deletion; PVC and Longhorn Volume are the actual data carriers.  
10. RWO PVC is unsuitable for simultaneous read/write mounting across multiple nodes.  
11. Stateful applications with multiple replicas are better suited for StatefulSet + volumeClaimTemplates.  
12. Replica provides high availability, not backup.  
13. Snapshot offers local point-in-time protection, not cross-region backup.  
14. Backup Target is critical for Longhorn's cross-failure-domain recovery.  
15. Restore should prioritize recovery to a new Volume, then validate before switching business operations.  
16. Volume Degraded status should not be ignored for extended periods.  
17. When Volume Faulted, the first principle is to preserve the scene.  
18. Longhorn performance is influenced by disks, network, replica count, and workload.  
19. Production environments must use dedicated data disks, monitoring alerts, backup strategies, and recovery drills.  
20. Longhorn is suitable for medium-scale K8s stateful applications but not for replacing object storage or blindly hosting core high-concurrency databases.  
21. At this point, the Longhorn stage has completed the full production operation cycle from Kubernetes storage plugin to production management.  

---

## Twenty-one, Reference Documents  

Longhorn official documentation:  

    https://longhorn.io/docs/latest/  

What is Longhorn:  

    https://longhorn.io/docs/latest/what-is-longhorn/  

Longhorn installation documentation:  

    https://longhorn.io/docs/latest/deploy/install/  

Longhorn Helm Chart:  

    https://github.com/longhorn/longhorn/tree/master/chart  

Longhorn nodes and volumes:  

    https://longhorn.io/docs/latest/nodes-and-volumes/  

Longhorn StorageClass parameters:  

    https://longhorn.io/docs/latest/references/storage-class-parameters/  

Longhorn Volume Conditions:  

    https://longhorn.io/docs/latest/nodes-and-volumes/volumes/volume-conditions/  

Longhorn Replica Rebuilding:  

    https://longhorn.io/docs/latest/advanced-resources/rebuilding/  

Longhorn Backup and Restore:  

    https://longhorn.io/docs/latest/snapshots-and-backups/  

Longhorn Backup Target:  

    https://longhorn.io/docs/latest/snapshots-and-backups/backup-and-restore/set-backup-target/  

Longhorn Recurring Snapshot and Backup:  

    https://longhorn.io/docs/latest/snapshots-and-backups/scheduling-backups-and-snapshots/  

Longhorn Troubleshooting:  

    https://longhorn.io/docs/latest/troubleshoot/troubleshooting/  

Longhorn Best Practices:  

    https://longhorn.io/docs/latest/best-practices/  

Longhorn Monitoring:  

    https://longhorn.io/docs/latest/monitoring/  

Kubernetes Persistent Volumes:  

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/  

Kubernetes Storage Classes:  

    https://kubernetes.io/docs/concepts/storage/storage-classes/  

Kubernetes CSI:  

    https://kubernetes-csi.github.io/docs/  

Kubernetes StatefulSet:  

    https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/  

Kubernetes Node Maintenance:  

    https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/  

Helm official documentation:  

    https://helm.sh/docs/  

fio official repository:  

    https://github.com/axboe/fio