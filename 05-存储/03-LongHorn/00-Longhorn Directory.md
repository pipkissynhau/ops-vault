# Longhorn Directory Index

Recommended path: 05-Storage/03-LongHorn/00-Longhorn directory index.md

Tags: #Longhorn #Kubernetes #CSI #BlockStorage #PV #PVC #StorageClass #CloudRawStorage #AdvancedSre #ProductionTransport

---

## I. Module Overview

Longhorn is a cloud-native distributed block storage system for Kubernetes.

Its core functions include:

    Provide persistent block storage capabilities for Kubernetes
    Interface with Kubernetes PV/PVC/StorageClass via CSI
    Organize node-local disks into distributed volumes usable by Pods
    Provide replication, snapshot, backup, recovery, and failover capabilities for stateful applications

Longhorn is not an object storage system, nor is it a traditional shared file system.

It is more akin to:

    A distributed cloud hard disk system in Kubernetes scenarios

It can be simply understood as:

    MinIO solves S3 object storage issues
    Ceph RBD solves general distributed block storage issues
    Longhorn solves Kubernetes internal PVC block storage issues

---

## II. Learning Objectives

After completing the Longhorn module, you should be able to:

1. Understand Longhorn's positioning within the Kubernetes storage architecture.
2. Understand the relationship between Longhorn and PV, PVC, StorageClass, and CSI.
3. Understand the roles of Longhorn Manager, Engine, Replica, and Instance Manager.
4. Plan Longhorn nodes, disks, dependent components, and StorageClass.
5. Install Longhorn using Helm.
6. View Longhorn Chart, images, values.yaml, and version information.
7. Resolve image pull issues without disrupting containerd/Kubernetes runtime.
8. Create PVCs and verify Pod mounting of Longhorn volumes.
9. Understand Longhorn replica counts, node distribution, and data high availability.
10. Configure Backup Target.
11. Execute Snapshot, Backup, and Restore operations.
12. Troubleshoot Volume Degraded, Replica rebuild, and node anomalies.
13. Understand Longhorn's performance boundaries from a production perspective.
14. Establish Longhorn routine inspection, monitoring, backup, and disaster recovery methods.
15. Determine suitable and unsuitable scenarios for Longhorn.

---

## III. Experiment Environment Planning

### 3.1 Experiment Network Segment

This storage module uniformly uses:

    10.0.0.0/24

Existing nodes:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.10 | ops-server | GitLab / Jenkins / Harbor / Operations Services |
| 10.0.0.20 | k8s-master01 | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Longhorn is typically deployed within the current Kubernetes cluster.

---

### 3.2 Kubernetes Experiment Environment

Default experiment environment:

    Kubernetes: kubeadm cluster
    System: Ubuntu Server 22.04.5 LTS
    Container runtime: containerd
    CNI: Calico
    Node count: 1 Master + 2 Workers

Example nodes:

| Node | IP | Role | Recommended for Longhorn data |
|---|---|---|---|
| k8s-master01 | 10.0.0.20 | Control Plane | Experimental participation is allowed, production generally not recommended |
| k8s-worker01 | 10.0.0.21 | Worker | Recommended |
| k8s-worker02 | 10.0.0.22 | Worker | Recommended |

Production recommendations:

    Longhorn data replicas should prioritize Worker nodes.
    Whether Master nodes participate in storage depends on resources, taints, reliability, and cluster scale.
    Production environments should not concentrate all replicas on a single node.
    Production environments should not place Longhorn data directories on system disks.

---

### 3.3 Longhorn Data Directory Planning

Experimental environments can initially use the default directory:

    /var/lib/longhorn

Production recommendations use an independent data disk mount:

    /data/longhorn

Example planning:

| Node | Data Directory | Notes |
|---|---|---|
| k8s-worker01 | /data/longhorn | Independent data disk |
| k8s-worker02 | /data/longhorn | Independent data disk |

Check commands:

    df -hT
    lsblk
    mount | grep longhorn

Production principles:

    Do not mistakenly write Longhorn data directories to system disks.
    Data disk capacity should be planned in advance.
    Multiple replicas increase actual disk usage.
    Data disk I/O directly affects business Pod performance.

---

## IV. Relationship Between Longhorn and Other Storage Modules

### 4.1 Longhorn vs Ceph

| Comparison Item | Longhorn | Ceph |
|---|---|---|
| Primary Focus | Kubernetes cloud-native block storage | General-purpose distributed storage platform |
| Access Method | CSI / PVC | RBD / CephFS / RGW / CSI |
| Deployment Complexity | Relatively low | Higher |
| Operation Complexity | Medium | High |
| Suitable Scenarios | K8s internal stateful application PVC | Large-scale unified storage platform |
| Data Model | Volume / Replica | Pool / PG / OSD / CRUSH |
| Learning Focus | PV/PVC/CSI/Replica/Recovery | RADOS/OSD/PG/CRUSH/Pool |

Simple understanding:

    Ceph is a general-purpose distributed storage foundation.
    Longhorn is a lighter cloud-native block storage solution for Kubernetes internals.

| Comparison Item | Longhorn | MinIO |
|---|---|---|
| Type | Block Storage | Object Storage |
| Access Method | PVC Mounted to Pod | S3 API |
| Typical Use Cases | MySQL, PostgreSQL, Redis, Application Data Disk | Images, Attachments, Backups, Logs, Artifacts |
| Data Unit | Volume | Bucket / Object |
| Protocol | CSI / Block Volume | HTTP / HTTPS S3 |
| Does It Provide Direct File Path to Application | Yes, Mounted as Directory | No, Accessed via API |

Simple Understanding:

    Pod needs a persistent data disk, use Longhorn.
    Application needs to upload images, attachments, backup packages, use MinIO.

---

### 4.3 Longhorn vs RustFS

| Comparison Item | Longhorn | RustFS |
|---|---|---|
| Type | Kubernetes Block Storage | S3 Object Storage |
| Target Objects | Pod / PVC | App / SDK / mc |
| Core Capabilities | Volume Replication, Snapshots, Backup, Recovery | Bucket, Object, S3 API |
| Deployment Location | Kubernetes Internal | Docker / VM / Cluster |
| Learning Focus | CSI and Stateful Applications | S3 and Object Storage Cluster |

---

## Five, Longhorn Core Concepts

### 5.1 CSI

CSI Full Name:

    Container Storage Interface

Function:

    Enables Kubernetes to call external storage systems via standard interface.

Longhorn provides via CSI:

    PV Creation
    PVC Binding
    Volume Attach
    Volume Mount
    Volume Expand
    Snapshot
    Restore

---

### 5.2 StorageClass

StorageClass is a template for dynamically creating PV in Kubernetes.

Longhorn typically creates a StorageClass after installation:

    longhorn

Check command:

    kubectl get storageclass

Example:

    kubectl get sc

Usage:

    PVC specifies storageClassName: longhorn
    Kubernetes automatically calls Longhorn CSI to create volume

---

### 5.3 PVC

PVC is a user's storage request.

Example Meaning:

    I need a 5Gi persistent data disk
    Use storageClass: longhorn
    Access mode is ReadWriteOnce

Check command:

    kubectl get pvc
    kubectl describe pvc <pvc-name>

---

### 5.4 PV

PV is the actual persistent volume resource bound in Kubernetes.

After Longhorn dynamically creates PV, Kubernetes will bind PVC with PV.

Check command:

    kubectl get pv
    kubectl describe pv <pv-name>

---

### 5.5 Volume

Longhorn Volume is the data volume managed by Longhorn.

One PVC usually corresponds to one Longhorn Volume.

Check Location:

    Longhorn UI
    kubectl -n longhorn-system get volumes.longhorn.io

---

### 5.6 Engine

Engine is the data read/write engine for Longhorn Volume.

Can be understood as:

    The data plane control component for each Volume

It is responsible for:

    Receiving read/write requests
    Synchronizing writes to multiple Replicas
    Managing Volume runtime status
    Coordinating replica rebuild

---

### 5.7 Replica

Replica is the data copy of Longhorn Volume.

For example, with 3 replicas:

    A Volume will have 3 Replicas
    Replicas will be distributed across different nodes as much as possible
    If a node fails, the Volume can still be used or recovered as long as there are healthy replicas

Check Method:

    Longhorn UI
    kubectl -n longhorn-system get replicas.longhorn.io

---

### 5.8 Instance Manager

Instance Manager is used to manage running instances of Longhorn Engine and Replica.

Can be understood as:

    Longhorn data plane instance manager

Check Command:

    kubectl -n longhorn-system get pods | grep instance-manager

---

### 5.9 Longhorn Manager

Longhorn Manager is the core control plane component of Longhorn.

It is responsible for:

    Managing nodes
    Managing disks
    Managing Volumes
    Managing Replicas
    Managing Engines
    Managing backups and recovery
    Collaborating with Kubernetes API and Longhorn CRD

Check Command:

    kubectl -n longhorn-system get pods | grep longhorn-manager

---

## Six, Longhorn Module Notes Directory

### 00-Longhorn Directory Index

Files:

    00-Longhorn Directory Index.md

Objective:

    Overview Longhorn module learning path.
    Clarify experiment environment, node planning, learning boundaries, and hands-on goals.
    Establish relationship between Longhorn, Ceph, MinIO, and RustFS.

---

### 01-Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI

Files:

    01-Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI.md

Focus:

    What is Longhorn.
    What is cloud-native block storage.
    Why Kubernetes needs CSI.
    Relationship between PV / PVC / StorageClass.
    Longhorn suitable scenarios.
    Longhorn unsuitable scenarios.

Hands-on Goals:

    Check Kubernetes cluster storage resources.
    Check StorageClass.
    Create minimal PVC.
    Understand PVC to PV binding process.

### 02-Longhorn Architecture: Manager, Engine, Replica, Instance Manager

File:

    02-Longhorn Architecture: Manager, Engine, Replica, Instance Manager.md

Focus:

    Longhorn control plane and data plane.
    Role of Manager.
    Role of Engine.
    Role of Replica.
    Role of Instance Manager.
    Full chain from PVC creation to Pod mounting.

Hands-on Objectives:

    View longhorn-system namespace.
    View Longhorn Pods.
    View CRD.
    Observe Volume, Engine, Replica changes after PVC creation.
    Understand Longhorn internal objects via kubectl.

---

### 03-Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass

File:

    03-Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass.md

Focus:

    What must be checked before installation.
    open-iscsi / iscsid.
    NFSv4 client.
    Mount propagation.
    Data disk path.
    Node resources.
    Default behavior of StorageClass.
    Why not to blindly install in production.

Hands-on Objectives:

    Check Kubernetes version.
    Check node system.
    Install dependencies.
    Check iscsid.
    Check data directory.
    Check node disks and mount points.
    Plan /data/longhorn.

---

### 04-Longhorn Helm Installation Methodology: Chart, Images, values.yaml, and Version Management

File:

    04-Longhorn Helm Installation Methodology: Chart, Images, values.yaml, and Version Management.md

Focus:

    Why Helm methodology is recommended.
    How to view Longhorn Chart.
    How to view values.yaml.
    How to filter images.
    How to fix versions.
    How to define values.yaml.
    How to handle domestic network and image repositories.
    Why not directly modify containerd configuration.

Hands-on Objectives:

    helm repo add longhorn.
    helm show chart.
    helm show values.
    Extract image list.
    Generate values.yaml.
    Install Longhorn.
    Check Pod, Service, CRD, StorageClass.

---

### 05-Longhorn Dynamic Volume Practice: PVC, PV, Pod Mounting, and Data Persistence

File:

    05-Longhorn Dynamic Volume Practice: PVC, PV, Pod Mounting, and Data Persistence.md

Focus:

    How PVC triggers Longhorn dynamic Volume creation.
    How Pod mounts PVC.
    How data is persisted.
    Whether data remains after Pod deletion and recreation.
    Differences between Deployment/StatefulSet using Longhorn.

Hands-on Objectives:

    Create PVC.
    Create Pod mounting PVC.
    Write test file.
    Delete Pod.
    Recreate Pod.
    Verify data remains.
    Check PV, PVC, Volume, Replica status.

---

### 06-Longhorn Replica Mechanism: Replica Count, Node Distribution, and Data High Availability

File:

    06-Longhorn Replica Mechanism: Replica Count, Node Distribution, and Data High Availability.md

Focus:

    What is Replica count.
    Why replica count is not equal to backup.
    How replicas are distributed across nodes.
    How Volume status changes after node failure.
    How replica rebuild occurs.
    Issues when node count is insufficient.

Hands-on Objectives:

    Set replica count.
    View Replica distribution.
    Stop a Worker node or evict Longhorn components.
    Observe Volume Degraded.
    Recover node.
    Observe Replica Rebuild.

---

### 07-Longhorn Backup and Recovery: Backup Target, Snapshot, and Volume Restore

File:

    07-Longhorn Backup and Recovery: Backup Target, Snapshot, and Volume Restore.md

Focus:

    What is Snapshot.
    What is Backup.
    Differences between Snapshot and Backup.
    How to configure Backup Target.
    NFS Backup Target.
    S3 Backup Target.
    How to restore Volume from Backup.
    Why replica is not backup.

Hands-on Objectives:

    Create Volume Snapshot.
    Rollback Snapshot.
    Configure NFS or S3 Backup Target.
    Execute Backup.
    Delete test data.
    Restore Volume.
    Verify recovery results.

---

### 08-Longhorn Troubleshooting: Volume Degraded, Replica Rebuild, and Node Anomalies

File:

    08-Longhorn Troubleshooting: Volume Degraded, Replica Rebuild, and Node Anomalies.md

Focus:

    What is Volume Degraded.
    What is Replica Failed.
    How to troubleshoot Engine anomalies.
    How to troubleshoot Instance Manager anomalies.
    How to troubleshoot node disk unavailability.
    How to troubleshoot PVC Pending.
    How to troubleshoot Pod mount failure.
    How to troubleshoot data directory mistakenly written to system disk.

Hands-on Objectives:

    Troubleshoot PVC Pending.
    Troubleshoot Pod Mount failure.
    Troubleshoot Volume Degraded.
    Troubleshoot Replica Rebuild.
    View Longhorn Manager logs.
    View kubelet events.
    View node iSCSI status.
    View disk capacity and mount.

---

### 09-Longhorn Performance and Production Recommendations: Disks, Network, Scheduling, and Resource Limits

File:

    09-Longhorn Performance and Production Recommendations: Disks, Network, Scheduling, and Resource Limits.md

Focus:

Longhorn Performance Bottlenecks  
Why Network and Disk Are Critical  
Why Database Scenarios Require Caution  
Impact of Replica Count on Write Performance  
Node Scheduling and Taint Tolerance  
CPU / Memory Requests  
Disk Pressure and Rebuild Pressure  
Longhorn Suitable and Unsuitable Production Scenarios  

Hands-on Objectives:  

- View Node Resources  
- View Longhorn Pod Resources  
- Observe Write Testing  
- Observe Resource Changes During Replica Rebuild  
- Create a Production Recommendation Checklist  

---  

### 99-Longhorn Stage Summary: From Kubernetes Storage Plugin to Production Operations  

Files:  

- 99-Longhorn Stage Summary: From Kubernetes Storage Plugin to Production Operations.md  

Key Points:  

- Summarize the Longhorn Learning Path  
- Summarize the Relationship Between Longhorn and PV/PVC/CSI  
- Summarize Deployment, Backup, Failure, Performance, and Production Boundaries  
- Summarize Interview Expression  
- Establish a Longhorn Production Usability Acceptance Checklist  

---  

## VII. Longhorn Hands-on Mainline  

The Longhorn module hands-on mainline is as follows:  

1. Understand Longhorn and CSI  
2. Check Kubernetes Cluster  
3. Check Node Dependencies  
4. Plan Data Disks and Data Directories  
5. Use Helm to View Chart and values.yaml  
6. Fix Longhorn Version  
7. Handle Image and Domestic Network Issues  
8. Install Longhorn  
9. Check longhorn-system Components  
10. Check StorageClass  
11. Create PVC  
12. Create Pod Mounted PVC  
13. Validate Data Persistence  
14. View Volume / Replica / Engine  
15. Simulate Node Failure  
16. Observe Volume Degraded  
17. Recover Node and Observe Replica Rebuild  
18. Configure Backup Target  
19. Execute Snapshot / Backup / Restore  
20. Establish Monitoring, Inspection, and Production Recommendations  

---  

## VIII. Longhorn Installation Prerequisites  

### 8.1 Kubernetes Requirements  

Ensure the Kubernetes cluster is functioning properly first.  

Checks:  

- kubectl get nodes -o wide  
- kubectl get pods -A  
- kubectl get sc  
- kubectl version  

Requirements:  

- Nodes Ready  
- CoreDNS Normal  
- kube-proxy / CNI Normal  
- Cluster Network Normal  
- kubelet Normal  
- kubectl Can Access Cluster  

---  

### 8.2 Node System Requirements  

Each node planned for Longhorn deployment must be checked:  

- hostname  
- uname -a  
- cat /etc/os-release  
- df -hT  
- lsblk  
- mount  
- free -h  
- systemctl status kubelet  
- systemctl status containerd  

Focus:  

- Whether Data Disks Are Independent  
- Whether System Disk Is Sufficient  
- Whether Nodes Have Disk Pressure  
- Whether Nodes Have Enough CPU and Memory  
- Whether Time Is Synchronized  
- Whether containerd Is Normal  

---  

### 8.3 open-iscsi Dependency  

Longhorn requires nodes to have iSCSI capabilities.  

Ubuntu Installation:  

- apt update  
- apt install -y open-iscsi  
- systemctl enable --now iscsid  
- systemctl status iscsid  

Checks:  

- which iscsiadm  
- iscsiadm --version  
- systemctl is-active iscsid  

Rocky Linux 9 Installation:  

- dnf install -y iscsi-initiator-utils  
- systemctl enable --now iscsid  
- systemctl status iscsid  

---  

### 8.4 NFSv4 Client Dependency  

If RWX support is needed, nodes must have NFSv4 client capabilities.  

Ubuntu:  

- apt install -y nfs-common  

Rocky Linux 9:  

- dnf install -y nfs-utils  

Checks:  

- mount.nfs -V  

Notes:  

- RWO block volumes mainly depend on iSCSI  
- RWX scenarios involve NFS clients  
- Even if RWX is not done initially, it's recommended to record this dependency in learning notes  

---  

### 8.5 Data Disk Directory  

Recommend creating in Worker nodes:  

- mkdir -p /data/longhorn  

Checks:  

- df -hT /data/longhorn  
- ls -ld /data/longhorn  

Production Reminder:  

- If /data/longhorn remains on the system disk, it can only be used for learning  
- Production should mount an independent data disk  
- Data disk capacity and performance will directly affect PVC performance  
- Longhorn replicas will consume multiple times the capacity  

---  

## IX. Image and Domestic Network Methodology  

The Longhorn module must reflect Helm methodology and image management approach.  

Core Principles:  

- Do not directly disrupt containerd or KubernetesBottom runtime  
- Do not arbitrarily modify clusterBottom configuration for pulling images  
- First check the Chart  
- Then check values.yaml  
- Then filter images  
- Then decide whether to synchronize to domestic repositories  
- Then specify image repository and tag via values.yaml  

Typical Workflow:  

- helm repo add longhorn https://charts.longhorn.io  
- helm repo update  
- helm show chart longhorn/longhorn  
- helm show values longhorn/longhorn > longhorn-values-default.yaml  
- grep -n "repository\\|image\\|tag" longhorn-values-default.yaml  

Then based on the image list: /think

docker pull official image fixed version  
docker tag to your own Alibaba Cloud repository or Harbor  
docker push to your own repository  
modify values.yaml  
helm install / helm upgrade  

This reflects the advanced operations mindset:

    Read the Chart first  
    Control the version  
    Control the image  
    Install afterward  
    Do not blindly execute one-click commands  
    Do not break the underlying runtime  

---

## TenI don't know.Longhorn Production Boundaries

### 10.1 Suitable Scenarios

Longhorn is suitable for:

    Small to medium-sized Kubernetes clusters  
    Private deployment environments  
    Edge clusters  
    Development/test clusters  
    Small to medium-sized stateful applications  
    Scenarios requiring PVC dynamic provisioning  
    Scenarios requiring simple snapshots and backup recovery  
    Bare-metal clusters without cloud vendor block storage capabilities  

Typical applications:

    MySQL test environment  
    PostgreSQL test environment  
    Redis persistence  
    Nacos data directory  
    Prometheus data volume  
    Jenkins data volume  
    GitLab small-scale test environment  
    General business application upload directory  

---

### 10.2 Cautionary Scenarios

The following scenarios require caution:

    High-concurrency core databases  
    Large-scale high IOPS scenarios  
    Ultra-low latency databases  
    Super-large production storage  
    Clusters with unstable network quality  
    Nodes with significantly different disk performance  
    Too few nodes but high replica requirements  
    Production environments without backup targets  
    Production environments without monitoring alerts  

Notes:

    Longhorn can run databases.  
    However, whether Longhorn is suitable for production core databases needs comprehensive evaluation based on disk, network, IOPS, latency, backup, recovery, and SLA.  
    Longhorn should not be simply understood as "installing equals production-grade high-performance storage".  

---

### 10.3 Not Recommended Scenarios

Not recommended:

    Single-node production high availability  
    All replicas on the same node  
    Data directory placed on the system disk  
    No backup  
    No monitoring  
    No disaster recovery drills  
    Using Longhorn to replace object storage  
    Using Longhorn to store large amounts of object files instead of MinIO  
    Running critical databases on nodes with poor resources  

---

## ElevenI don't know.Longhorn Daily Inspection Directions

Daily checks:

    kubectl -n longhorn-system get pods  
    kubectl get sc  
    kubectl get pvc -A  
    kubectl get pv  
    kubectl get nodes  
    df -hT  
    lsblk  

Longhorn object checks:

    kubectl -n longhorn-system get volumes.longhorn.io  
    kubectl -n longhorn-system get replicas.longhorn.io  
    kubectl -n longhorn-system get engines.longhorn.io  
    kubectl -n longhorn-system get nodes.longhorn.io  

Focus on:

    Whether Volume is Healthy  
    Whether there are Degraded  
    Whether there are Faulted  
    Whether Replica is normally distributed  
    Whether there is reconstruction  
    Whether node disk capacity is sufficient  
    Whether longhorn-manager is normal  
    Whether instance-manager is normal  
    Whether CSI components are normal  
    Whether Backup Target is available  

---

## TwelveI don't know.Longhorn Fault Diagnosis Entry Points

### 12.1 PVC Pending

Diagnosis:

    kubectl describe pvc <pvc-name>  
    kubectl get sc  
    kubectl -n longhorn-system get pods  
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=100  

Common causes:

    StorageClass does not exist.  
    Longhorn CSI anomaly.  
    Node disk insufficient.  
    Longhorn not installed.  
    Default StorageClass configuration does not meet expectations.  

---

### 12.2 Pod Mount Failure

Diagnosis:

    kubectl describe pod <pod-name>  
    kubectl describe pvc <pvc-name>  
    kubectl get events -A --sort-by=.lastTimestamp  
    systemctl status iscsid  
    iscsiadm --version  

Common causes:

    open-iscsi not installed.  
    iscsid not running.  
    kubelet mount failure.  
    Volume not attached.  
    Node anomaly.  
    Longhorn Engine / Instance Manager anomaly.  

---

### 12.3 Volume Degraded

Diagnosis:

    Longhorn UI  
    kubectl -n longhorn-system get volumes.longhorn.io  
    kubectl -n longhorn-system get replicas.longhorn.io  
    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>  
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200  

Common causes:

    A replica failed.  
    A node is offline.  
    A disk is unavailable.  
    Disk capacity insufficient.  
    Replica reconstruction in progress.  
    Insufficient replica count.  

---

### 12.4 Node Disk Anomaly

Diagnosis:

    kubectl get nodes  
    kubectl describe node <node-name>  
    df -hT  
    lsblk  
    dmesg | tail -100  
    journalctl -k --since "1 hour ago"  
    kubectl -n longhorn-system get nodes.longhorn.io  

---

## ThirteenI don't know.Longhorn Security and Production Notes

### 13.1 UI Access Security

Longhorn UI should not be directly exposed to the public.

Recommendation: /think

Only accessible via the internal network.
Authentication required when exposed via Ingress.
Only allow access from the operations network segment.
Do not grant regular business personnel management permissions.
Production entry requires HTTPS.
Integrate with RBAC and access control.

---

### 13.2 Data Security

Note:

    Longhorn replicas are not backups.
    Snapshots are not offsite backups.
    Backup Target is the more accurate backup capability.
    Production must configure backups.
    Backups must be regularly restored and validated.
    Do not store all backups in the same failure domain.

---

### 13.3 Operation Security

High-risk operations:

    Delete PVC
    Delete PV
    Delete Longhorn Volume
    Delete Replica
    Clear /data/longhorn
    Format data disks
    Modify replica count
    Batch restart Longhorn components
    Uninstall Longhorn
    Modify default StorageClass

Production requirements:

    Confirm business before changes.
    Confirm backups before changes.
    Confirm rollback plan before changes.
    Dual verification for high-risk operations.
    Retain operation records.

---

## FourteenI don't know.Longhorn and Interview Expression

If asked in an interview:

    What is Longhorn? How do you understand it?

Can answer:

    Longhorn is a Kubernetes cloud-native distributed block storage system, primarily providing dynamic PV/PVC capabilities to Kubernetes through CSI. It can organize local disks across multiple nodes to provide persistent volumes for Pods, and improve volume availability through replication mechanisms.
    From a Kubernetes perspective, users create PVCs and specify Longhorn StorageClass. Kubernetes dynamically creates PVs via Longhorn CSI, which then creates Volumes, Engines, and multiple Replicas in the backend. After Pods mount PVCs, data is actually written to Longhorn Volumes, managed by Longhorn for replica synchronization and failure recovery.
    Longhorn is suitable for medium-scale Kubernetes clusters, private environments, edge environments, and scenarios requiring dynamic PVC provisioning. It is not equivalent to object storage and should not replace S3 storage like MinIO.
    In production use, I will focus on node disk planning, open-iscsi dependencies, StorageClass parameters, replica count, whether data directories reside on independent disks, whether Volumes are Degraded, whether Replicas are properly rebuilding, Backup Target configuration, and whether monitoring alerts are complete.
    I will also emphasize that Longhorn replicas are not backups. Even with multiple Replicas, it cannot prevent accidental PVC deletion, data deletion, or cluster failures, so production must configure Backup Target and regularly perform recovery drills.

---

## FifteenI don't know.Phase Advancement Suggestions

Longhorn module is recommended to proceed in the following order:

    Step 1: First write 01, explaining Kubernetes cloud-native block storage and CSI.
    Step 2: Write 02, understanding Longhorn internal components.
    Step 3: Write 03, completing pre-installation environment checks.
    Step 4: Write 04, installing via Helm methodology.
    Step 5: Write 05, creating PVC and Pod, verifying data persistence.
    Step 6: Write 06, verifying replication mechanism and node failure.
    Step 7: Write 07, configuring Backup Target and recovery.
    Step 8: Write 08, systematic troubleshooting.
    Step 9: Write 09, summarizing performance and production recommendations.
    Step 10: Write 99, completing phase summary.

Current module's most important requirements:

    Every article must have operational commands.
    Every article must be verifiable in a Kubernetes environment.
    Every article must highlight production boundaries.
    Cannot only explain concepts.
    Cannot treat Longhorn as a universal storage.
    Cannot ignore backups, monitoring, and failure recovery.
    Helm installation article must demonstrate Chart, images, values.yaml, and domestic network optimization methodology.

---

## SixteenI don't know.Reference Documents

Longhorn official documentation:

    https://longhorn.io/docs/latest/

What is Longhorn:

    https://longhorn.io/docs/latest/what-is-longhorn/

Longhorn installation documentation:

    https://longhorn.io/docs/latest/deploy/install/

Longhorn Helm Chart:

    https://github.com/longhorn/longhorn/tree/master/chart

Longhorn Backup and Restore:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Longhorn Volumes and Nodes:

    https://longhorn.io/docs/latest/nodes-and-volumes/

Longhorn Troubleshooting:

    https://longhorn.io/kb/troubleshooting/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Storage Classes:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes CSI:

    https://kubernetes-csi.github.io/docs/