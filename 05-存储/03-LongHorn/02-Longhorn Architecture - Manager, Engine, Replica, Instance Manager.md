# Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI

Recommended path: 05-Storage/03-LongHorn/01-Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI.md

Tags: #Longhorn #Kubernetes #CSI #PV #PVC #StorageClass #BlockStorage #CloudRawStorage #ApplyWithStatus #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the first article of the Longhorn module, focusing on understanding Longhorn's positioning in the Kubernetes storage architecture.

This article does not directly enter installation, but first clarifies the following questions:

    Why does Kubernetes need persistent storage?
    What are PV, PVC, and StorageClass?
    What is CSI?
    Why is Longhorn considered Kubernetes cloud-native block storage?
    What problems is Longhorn suitable for solving?
    What problems is Longhorn not suitable for solving?
    What are the differences between Longhorn, MinIO, and Ceph?
    What Kubernetes storage objects should be checked before installing Longhorn?

This article emphasizes practical operations, using kubectl commands to observe storage resources in the current Kubernetes cluster, laying the foundation for subsequent Longhorn installation and PVC practice.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand why Pods need persistent storage in Kubernetes.
2. Understand the differences between EmptyDir, HostPath, PV, PVC, and StorageClass.
3. Understand the role of CSI in the Kubernetes storage architecture.
4. Understand that Longhorn is a Kubernetes cloud-native distributed block storage.
5. Understand the differences between Longhorn and object storage MinIO.
6. Understand the relationship and differences between Longhorn and Ceph RBD.
7. Be able to view existing StorageClass in the Kubernetes cluster.
8. Be able to view PV and PVC resources.
9. Be able to create a PVC without a StorageClass and observe the Pending state.
10. Understand why PVCs cannot automatically bind PVs without a dynamic storage plugin.
11. Prepare for dynamically creating PVCs after Longhorn installation.

---

## III. Experimental Environment

### 3.1 Kubernetes Cluster

Default experimental environment:

    Kubernetes: kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node Network Segment: 10.0.0.0/24

Node planning:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane |
| 10.0.0.21 | k8s-worker01 | Worker |
| 10.0.0.22 | k8s-worker02 | Worker |

---

### 3.2 Longhorn Data Directory Planning

Longhorn will default use:

    /var/lib/longhorn

Production recommendation: mount an independent data disk directory:

    /data/longhorn

Check commands:

    df -hT
    lsblk
    mount | grep longhorn

Production reminder:

    The experimental environment can first use the system disk directory.
    It is not recommended to place the Longhorn data directory on the system disk in production.
    Longhorn replicas will increase disk usage, for example, 1 10Gi Volume with 3 replicas requires about 30Gi raw space.
    Longhorn performance strongly depends on node disk and inter-node network quality.

---

## IV. Why Kubernetes Needs Persistent Storage

### 4.1 Pods Are Temporary

Kubernetes Pods have the following characteristics:

    Pods can be deleted.
    Pods can be rebuilt.
    Pods can be evicted.
    Pods can be scheduled to other nodes.
    Container filesystems in Pods change with container lifecycle.
    New Pods may replace old Pods during Deployment updates.

If business data is only written to the container's internal filesystem, data may be lost when the Pod is deleted or rebuilt.

Examples:

    MySQL data directory
    PostgreSQL data directory
    Redis AOF/RDB files
    Jenkins home directory
    Prometheus TSDB data
    GitLab data directory
    Application upload file directory

This data cannot be lost when the Pod is deleted.

---

### 4.2 Stateless and Stateful Applications

Stateless applications:

    Pods can be rebuilt directly after deletion.
    Data is not saved locally.
    State is usually stored in databases, caches, object storage, or external services.
    Typical applications: frontend, API services, gateways, ordinary microservices.

Stateful applications:

    Data must be retained after Pod deletion.
    Usually require stable storage.
    Need to continue using original data after rebuilding.
    Typical applications: MySQL, PostgreSQL, Redis, Prometheus, Jenkins, GitLab.

Longhorn mainly serves:

    Stateful applications that need PVC in Kubernetes.

---

### 4.3 Problems Kubernetes Storage Solves

The Kubernetes storage system mainly solves:

    How to provide persistent directories for Pods.
    How to retain data after Pod deletion.
    How to remount original data for new Pods.
    How to dynamically create storage volumes.
    How to abstract differences in underlying storage systems.
    How to let applications use storage via PVC instead of directly caring about storage backends.

---

## V. Common Kubernetes Storage Types

### 5.1 EmptyDir

EmptyDir is a temporary directory for Pods.

Features:

    Created when the Pod is created.
    Deleted when the Pod is deleted.
    Multiple containers in the same Pod can share it.
    Not suitable for persistent data storage.

Typical scenarios:

    Temporary cache
    Temporary files
    Sidecar shared directory
    Build process temporary directory

Example understanding:

    Data remains while the Pod exists.
    Data disappears when the Pod is gone.

---

### 5.2 HostPath

HostPath mounts a node's local directory into the Pod.

Features:

    Uses node-local directories.
    Data paths differ when the Pod is scheduled to different nodes.
    Strongly depends on nodes.
    May introduce security risks.
    Not suitable for direct use by ordinary applications.

Typical scenario: /think

Log Collection DaemonSet  
Node Monitoring Agent  
Access Host Node Socket  
Special System Components  

Production Notice:

    HostPath is not equal to cloud-native dynamic storage.  
    Ordinary business should not rely on HostPath to save core data.  
    After Pod migrates to other nodes, the original node's HostPath data will not be automatically followed.  

---

### 5.3 PV  

PV Full Name:  

    PersistentVolume  

Meaning:  

    Persistent volume resource in Kubernetes.  
    Represents an existing or newly created persistent storage by a storage system.  

PV is a cluster-level resource.  

Check Command:  

    kubectl get pv  

PV can come from:  

    NFS  
    Ceph RBD  
    CephFS  
    Longhorn  
    Cloud vendor cloud disks  
    Local disks  
    Other CSI storage systems  

---

### 5.4 PVC  

PVC Full Name:  

    PersistentVolumeClaim  

Meaning:  

    User's application for storage resources.  

Can be understood as:  

    Application says: I need a 5Gi persistent storage.  
    Kubernetes is responsible for finding or creating a suitable PV.  
    After PVC is bound with PV, Pod can mount PVC.  

PVC is a namespace-level resource.  

Check Command:  

    kubectl get pvc -A  

---

### 5.5 StorageClass  

StorageClass is a template for dynamically creating PV.  

It defines:  

    Which storage plugin to use.  
    What type of volume to create.  
    What parameters to use.  
    Whether expansion is allowed.  
    What recycling policy to use.  
    What volume binding mode to use.  

Check Command:  

    kubectl get storageclass  

Abbreviation:  

    kubectl get sc  

With StorageClass, users don't need to manually create PV when creating PVC.  

Process becomes:  

    User creates PVC  
      |  
      v  
    PVC specifies storageClassName  
      |  
      v  
    Kubernetes calls CSI Provisioner  
      |  
      v  
    Storage system dynamically creates Volume  
      |  
      v  
    Kubernetes creates PV  
      |  
      v  
    PVC binds with PV  
      |  
      v  
    Pod mounts PVC  

After Longhorn is installed, it usually creates a StorageClass:  

    longhorn  

---

## SixI don't know.PVI don't know.PVCI don't know.StorageClass Relationship Diagram  

### 6.1 Static Supply Mode  

In static supply mode, administrators create PV first.  

    Administrator creates PV  
          |  
          v  
    User creates PVC  
          |  
          v  
    Kubernetes finds a suitable PV  
          |  
          v  
    PVC binds with PV  
          |  
          v  
    Pod mounts PVC  

Features:  

    Administrators need to prepare PV in advance.  
    Users cannot automatically create new volumes.  
    Suitable for small amounts of fixed storage.  
    Higher operation and maintenance costs.  

---

### 6.2 Dynamic Supply Mode  

In dynamic supply mode, users only create PVC.  

    User creates PVC  
          |  
          v  
    PVC specifies StorageClass  
          |  
          v  
    CSI Provisioner calls storage system  
          |  
          v  
    Automatically creates Volume  
          |  
          v  
    Automatically creates PV  
          |  
          v  
    PVC binds with PV  
          |  
          v  
    Pod mounts PVC  

Longhorn is one of the storage systems in dynamic supply mode.  

---

## SevenI don't know.What is CSI  

### 7.1 CSI Definition  

CSI Full Name:  

    Container Storage Interface  

In Chinese, it can be understood as:  

    Container Storage Interface  

It is a standard interface for Kubernetes to connect with external storage systems.  

CSI's goals:  

    Kubernetes does not natively include all storage drivers.  
    Storage vendors or open-source storage systems access Kubernetes through CSI plugins.  
    Kubernetes completes volume creation, mounting, unmounting, expansion, snapshots, etc., through a unified interface.  

---

### 7.2 What CSI Can Do  

CSI can support:  

    Dynamically create volumes  
    Delete volumes  
    Mount volumes  
    Unmount volumes  
    Expand volumes  
    Create snapshots  
    Restore from snapshots  
    Query volume status  

For Kubernetes:  

    Only needs to call storage plugins through CSI standard.  

For Longhorn:  

    Longhorn provides its Volume to Kubernetes through CSI.  

---

### 7.3 CSI's Role in Longhorn  

After Longhorn is installed, it deploys CSI-related components in Kubernetes.  

These components are responsible for:  

    Listening for PVC creation requests.  
    Creating Longhorn Volumes.  
    Binding Volumes as Kubernetes PV.  
    Attaching Volumes to target nodes.  
    Assisting kubelet to mount Volumes to Pods.  
    Supporting Volume expansion.  
    Supporting Snapshot capabilities.  

Simple flow:  

    PVC  
     |  
     v  
    Kubernetes CSI  
     |  
     v  
    Longhorn CSI Driver  
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
    Pod mounts and uses  

---

## EightI don't know.What is Longhorn  

### 8.1 Longhorn Positioning  

Longhorn is a Kubernetes cloud-native distributed block storage system.  

It mainly provides:  

    Dynamic PV/PVC supply  
    Distributed block storage volumes  
    Multi-replica data protection  
    Snapshots  
    Backups  
    Recovery  
    Volume expansion  
    Replica reconstruction after node failure  
    Web UI management interface  
    CSI integration  

Core positioning:  

    Provide persistent block storage for stateful Pods in Kubernetes.

### 8.2 Longhorn is Not Object Storage

Longhorn is not MinIO.

Longhorn does not directly upload objects to applications via S3 API.

Longhorn provides:

    PVC
    Mounted Directory
    Block Storage Volume

MinIO provides:

    Bucket
    Object
    S3 API
    HTTP / HTTPS Endpoint

Differences:

| Comparison Item | Longhorn | MinIO |
|---|---|---|
| Type | Block Storage | Object Storage |
| Access Method | PVC Mount | S3 API |
| Form Seen by Application | File System Directory | HTTP API |
| Kubernetes Relationship | Deep Integration with CSI | Can be deployed on K8s or independently |
| Typical Use Cases | Database Data Disk, Application Persistent Volumes | Images, Attachments, Backups, Archives |

---

### 8.3 Longhorn is Not a Universal Database Storage

Longhorn can provide PVC for databases, but this does not mean all core databases are suitable for direct operation on Longhorn.

Need to evaluate:

    Node Disk Performance
    Node Network Quality
    Replica Count
    Write Latency
    IOPS Requirements
    Database Importance
    Backup Recovery Capability
    Fault Drill Capability
    Monitoring Alert Capability

Production Notes:

    Longhorn is suitable for medium-scale Kubernetes stateful applications.
    Core high-concurrency databases require cautious evaluation.
    Longhorn replicas are not database backups.
    Databases still require their own backups, such as mysqldump, xtrabackup, pg_dump, WAL archiving, etc.

---

## Nine, Longhorn Suitable Scenarios

### 9.1 Suitable Scenarios

Longhorn is suitable for:

    Medium-scale Kubernetes Clusters
    Private Deployment Environments
    Bare Metal Kubernetes
    Edge Kubernetes
    Development/Test Environments
    Businesses Needing Dynamic PVC
    Ordinary Stateful Applications
    Applications Needing Snapshot and Recovery Capabilities
    Clusters Without Cloud Vendor Cloud Disk Capabilities

Typical Applications:

    Jenkins Home
    Prometheus Data Disk
    Nacos Data Directory
    Redis Persistence
    MySQL Test Environment
    PostgreSQL Test Environment
    GitLab Small-Scale Experimental Environment
    Ordinary Business Upload Directory
    Middleware Configuration and Data Directory

---

### 9.2 Cautionary Scenarios

Need to be cautious:

    Core High-Concurrency MySQL
    High IOPS Databases
    Low-Latency-Required Business
    Super Large-Scale Production Databases
    Nodes with Unstable Network Environments
    Nodes with Significant Disk Performance Differences
    Single-Node Kubernetes
    No Backup Environment
    No Monitoring Environment
    All Replicas in the Same Fault Domain

---

### 9.3 Not Recommended Scenarios

Not recommended:

    Use Longhorn to Replace MinIO for Storing Object Files.
    Use Longhorn to Store Massive Images, Attachments, and Archive Packages.
    Place Longhorn Data Directory on System Disk and Directly Go to Production.
    Run Critical Production Data Without Backup Target.
    Assume Data Safety Without Recovery Drills.
    Force High Replica Count When Node Count is Insufficient.
    Run Core Databases on Low-Performance Disks.

---

## Ten, Operational Practice One: Check Kubernetes Cluster Status

### 10.1 View Nodes

Execute:

    kubectl get nodes -o wide

Expected:

    All nodes are Ready.
    Master and worker node IPs are correct.
    Node versions are consistent or basically consistent.

Example Focus Points:

    NAME            STATUS   ROLES           INTERNAL-IP
    k8s-master01    Ready    control-plane   10.0.0.20
    k8s-worker01    Ready    <none>          10.0.0.21
    k8s-worker02    Ready    <none>          10.0.0.22

---

### 10.2 View System Components

Execute:

    kubectl get pods -A

Focus on:

    Whether CoreDNS in kube-system is Running.
    Whether CNI Components are Running.
    Whether kube-proxy is Running.
    Whether metrics-server (if installed) is Running.
    Whether there are many CrashLoopBackOff.
    Whether there are many Pending.

---

### 10.3 View Events

Execute:

    kubectl get events -A --sort-by=.lastTimestamp | tail -50

If the cluster has frequent abnormal events, address them before installing Longhorn.

Focus on:

    FailedScheduling
    FailedMount
    ImagePullBackOff
    NodeNotReady
    DiskPressure
    MemoryPressure
    NetworkUnavailable

---

### 10.4 View Node Resource Pressure

Execute:

    kubectl describe nodes | grep -E "Name:|DiskPressure|MemoryPressure|PIDPressure|Ready"

If nodes have:

    DiskPressure=True
    MemoryPressure=True
    Ready=False

Do not recommend continuing with Longhorn installation; address node resource issues first.

---

## Eleven, Operational Practice Two: Check Current Storage Resources

### 11.1 Check StorageClass

Execute:

    kubectl get storageclass

Or:

    kubectl get sc

Possible Result One: No StorageClass.

    No resources found

Explanation:

    The cluster currently has no default dynamic storage provisioning capability.
    Creating a regular PVC may result in Pending.
    After installing Longhorn, a new longhorn StorageClass will be added.

Possible Result Two: Existing StorageClass.

Example: /think

NAME                 PROVISIONER
nfs-client           k8s-sigs.io/nfs-subdir-external-provisioner
local-path           rancher.io/local-path
longhorn             driver.longhorn.io

Notes:

    If longhorn already exists, Longhorn may have been installed.
    If there are other StorageClass entries, understand their differences from Longhorn.

---

### 11.2 Check Default StorageClass

Execute:

    kubectl get sc

If a StorageClass shows:

    (default)

It indicates it is the default StorageClass.

Check details:

    kubectl describe sc <storageclass-name>

Focus on:

    Provisioner
    ReclaimPolicy
    VolumeBindingMode
    AllowVolumeExpansion
    Parameters

---

### 11.3 Check PV

Execute:

    kubectl get pv

If no PVs are found:

    No resources found

Notes:

    The current cluster has no created persistent volume resources.

---

### 11.4 Check PVC

Execute:

    kubectl get pvc -A

If no PVCs are found:

    No resources found

Notes:

    There are no namespace PVC applications in the current cluster.

---

## TwelveI don't know.Practice Three: Create a PVC Without Dynamic Provisioning Capability to Observe Pending

### 12.1 Create Experimental Namespace

Execute:

    kubectl create namespace storage-demo

Check:

    kubectl get ns storage-demo

---

### 12.2 Create PVC File

Create file:

    cat > pvc-no-storageclass.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: demo-pvc-no-storageclass
      namespace: storage-demo
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
    EOF

Notes:

    This PVC does not explicitly specify storageClassName.
    If the cluster has no default StorageClass, it will be Pending.
    If the cluster has a default StorageClass, it may automatically use the default StorageClass to create PV.

---

### 12.3 Apply PVC

Execute:

    kubectl apply -f pvc-no-storageclass.yaml

Check:

    kubectl get pvc -n storage-demo

Check details:

    kubectl describe pvc demo-pvc-no-storageclass -n storage-demo

---

### 12.4 Observation Result One: PVC Pending

If PVC status is Pending, common reasons:

    The cluster has no default StorageClass.
    No available static PV.
    No dynamic Provisioner.
    PVC cannot find available storage.

Events may show similar information:

    no persistent volumes available for this claim and no storage class is set

Operations understanding:

    Kubernetes itself does not create storage out of nowhere.
    There must be static PV, or StorageClass + CSI Provisioner.
    Longhorn can dynamically create PV via StorageClass after installation.

---

### 12.5 Observation Result Two: PVC Bound

If PVC status is Bound, it indicates:

    The cluster has a default StorageClass.
    Kubernetes has created or bound PV via the default StorageClass.
    Need to check which StorageClass it uses.

Execute:

    kubectl get pvc demo-pvc-no-storageclass -n storage-demo -o wide

Check details:

    kubectl describe pvc demo-pvc-no-storageclass -n storage-demo

Check corresponding PV:

    kubectl get pv

Then check PV details:

    kubectl describe pv <pv-name>

Focus on:

    StorageClass
    CSI Driver
    Reclaim Policy
    VolumeHandle

---

### 12.6 Clean Up Experimental PVC

If just observing PVC Pending, you can clean up:

    kubectl delete -f pvc-no-storageclass.yaml

If PVC is already Bound, note:

    Deleting PVC may trigger PV reclaim policy.
    If real volumes were created via default StorageClass, confirm if it's experimental resource before deletion.

Check PV reclaim policy:

    kubectl get pv

If confirmed as experimental resource, then delete:

    kubectl delete -f pvc-no-storageclass.yaml

---

## ThirteenI don't know.Practice Four: Check Impact of Default StorageClass on PVC

### 13.1 Why Pay Attention to Default StorageClass

Default StorageClass affects PVCs that do not specify storageClassName.

For example, if PVC does not include:

    storageClassName: longhorn

And the cluster has a default StorageClass, PVC will automatically use the default StorageClass.

This may lead to:

I wanted it. Longhorn, the results are stored in other storages.  
I wanted to test it. Pending, result automatically BoundI don't know.  
I wanted it. NFSIt worked. LonghornI don't know.  
Multiple storage systems are easily misused when stored together.

---

### 13.2 Viewing the Default StorageClass

Execute:

    kubectl get sc

Observe whether the NAME column contains:

    (default)

Check details:

    kubectl describe sc <storageclass-name>

---

### 13.3 Example of Removing the Default StorageClass

If you need to remove a default StorageClass:

    kubectl patch storageclass <storageclass-name> \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

Check again:

    kubectl get sc

---

### 13.4 Example of Setting Longhorn as the Default StorageClass

After Longhorn is installed, if you want longhorn to become the default StorageClass:

    kubectl patch storageclass longhorn \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

Production reminder:

    Setting Longhorn as the default StorageClass should be done with caution.  
    When multiple storage systems coexist, it is recommended to explicitly specify storageClassName for business PVCs.  
    It is not advisable to let all PVCs blindly use the default StorageClass.

---

## FourteenI don't know.Practice 5: PVC Template with Explicitly Specified StorageClass

### 14.1 PVC Template After Longhorn Installation

After Longhorn is installed, you can create the following PVC:

    cat > pvc-longhorn-demo.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: demo-pvc-longhorn
      namespace: storage-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn
      resources:
        requests:
          storage: 1Gi
    EOF

Apply:

    kubectl apply -f pvc-longhorn-demo.yaml

Check:

    kubectl get pvc -n storage-demo
    kubectl get pv

Check details:

    kubectl describe pvc demo-pvc-longhorn -n storage-demo

Notes:

    If Longhorn is not yet installed, this PVC will be Pending.  
    If Longhorn is installed and the longhorn StorageClass exists, this PVC should automatically Bound.  
    A full validation will be conducted in 05-Longhorn Dynamic Volume Practice.

---

### 14.2 Why It Is Recommended to Explicitly Specify StorageClass

Recommended:

    storageClassName: longhorn

Reasons:

    Avoid misusing the default StorageClass.  
    More clear when multiple storage systems coexist.  
    Facilitates migration and troubleshooting.  
    Enables different storage strategies for different business applications.

---

## Fifteen、Longhorn and PV/PVC Workflow

### 15.1 Complete Workflow

Longhorn dynamic volume creation workflow:

    User creates PVC
        |
        v
    PVC specifies storageClassName: longhorn
        |
        v
    Kubernetes detects the need for dynamic provisioning
        |
        v
    Calls Longhorn CSI Provisioner
        |
        v
    Longhorn creates Volume
        |
        v
    Longhorn creates Engine and Replica
        |
        v
    Kubernetes creates PV
        |
        v
    PVC binds with PV
        |
        v
    Pod mounts PVC
        |
        v
    Application writes data
        |
        v
    Data writes to Longhorn Volume
        |
        v
    Data synchronizes to multiple Replicas

---

### 15.2 Troubleshooting Workflow

If PVC cannot be used, troubleshoot along the workflow:

    Is PVC created?
    Is PVC Bound?
    Does StorageClass exist?
    Are CSI components normal?
    Is Longhorn Manager normal?
    Is Longhorn Volume created?
    Are Replicas normal?
    Is Pod scheduling successful?
    Is kubelet mounting successful?
    Is iscsid running?
    Is node disk normal?
    Is node network normal?

---

## Sixteen、Understanding and Comparison of Longhorn and Ceph RBD

### 16.1 Commonalities

Longhorn and Ceph RBD can both provide block storage capabilities for Kubernetes.

Commonalities:

    Both can provide PVC via CSI.  
    Both can dynamically create PV.  
    Both can mount as file system directories for Pods.  
    Both are suitable for stateful applications.  
    Both require attention to replicas, fault recovery, monitoring, and backups.

---

### 16.2 Differences

| Comparison Item | Longhorn | Ceph RBD |
|---|---|---|
| Deployment Complexity | Low | High |
| Operation Complexity | Medium | High |
| Underlying Architecture | Volume / Engine / Replica | RADOS / OSD / Pool / PG |
| Kubernetes Integration | Native for K8s | CSI integration |
| Suitable Scale | Medium-to-small K8s scenarios | Medium-to-large unified storage platform |
| Learning Focus | CSI, Volume, Replica, Backup | OSD, PG, CRUSH, Pool, RBD |
| UI Management | Longhorn UI is intuitive | Ceph Dashboard is more complex |
| Production Capabilities | Easy to use, but requires understanding of boundaries | Powerful, but complex |

Simple Understanding:

    Longhorn is more like a persistent disk inside Kubernetes.
    Ceph RBD is more like a general-purpose distributed block storage foundation.

---

## Seventeen, Understanding Comparison Between Longhorn and MinIO

| Comparison Item | Longhorn | MinIO |
|---|---|---|
| Storage Type | Block Storage | Object Storage |
| Kubernetes Usage | PVC | S3 API or application access |
| Data Unit | Volume | Object |
| Top-level Container | PVC / PV / Volume | Bucket |
| Access Protocol | CSI / Mount File System | HTTP / HTTPS S3 |
| Typical Port | Kubernetes internal | 9000 / 9001 |
| Typical Use Cases | MySQL, Redis, Prometheus | Images, attachments, backup packages, log archives |
| Data Protection | Replica, replica rebuild, Backup | Erasure Coding, Bucket backup, mirror |

One-sentence Differentiation:

    Use Longhorn if an application needs a single persistent data disk.
    Use MinIO if an application needs an object upload/download interface.

---

## Eighteen, Basic Principles of Longhorn from a Production Perspective

### 18.1 Don't Treat Replicas as Backups

Longhorn Replica solves:

    Node failure
    Disk failure
    Replica rebuild
    Volume availability

It cannot solve:

    Accidental deletion of PVC
    Accidental deletion of Volume
    Application corrupting data
    Accidental file deletion
    Cluster-wide failure
    All replicas being destroyed

Therefore:

    Replica is not Backup.
    Snapshot is not entirely equivalent to异地 backup.
    Production must configure Backup Target.
    Important data must undergo recovery drills.

---

### 18.2 Don't Treat System Disk as Data Disk

System disk can be temporarily used in experimental environments.

Production recommendations:

    Each Worker node should mount an independent data disk.
    Use /data/longhorn as Longhorn data directory.
    Confirm df -hT shows data directory mounted on expected disk.
    Do not let Longhorn data fill up system disk.

---

### 18.3 Don't Blindly Set High Replica Count

Higher replica count:

    May improve availability.
    Consumes more disk space.
    Write amplification becomes more noticeable.
    Network synchronization pressure increases.
    Replica rebuild takes longer.

Example:

    10Gi Volume, 3 replicas occupy about 30Gi raw space.
    Writing data requires synchronizing to multiple replicas.
    Insufficient node count prevents effective replica distribution.

Production recommendations:

    Common replica count is 2 or 3.
    At least sufficient Worker nodes to support replica distribution.
    Don't blindly set 3 replicas on 2 nodes and assume full high availability.

---

### 18.4 Don't Ignore iSCSI Dependencies

Longhorn Volume mounting depends on node-side capabilities.

If nodes lack open-iscsi or iscsid is not running, issues may occur:

    Pod mounting failure.
    Volume attach failure.
    MountVolume-related errors in kubelet events.
    PVC Bound but Pod cannot be used.

Check:

    systemctl status iscsid
    iscsiadm --version

---

## Nineteen, Common Issue Troubleshooting

### 19.1 PVC Stays Pending

Troubleshooting commands:

    kubectl get pvc -n storage-demo
    kubectl describe pvc demo-pvc-no-storageclass -n storage-demo
    kubectl get sc
    kubectl get pv
    kubectl get events -A --sort-by=.lastTimestamp | tail -50

Possible causes:

    No default StorageClass.
    PVC specified StorageClass does not exist.
    Longhorn not installed.
    Longhorn CSI components abnormal.
    Node disk space insufficient.
    StorageClass parameter anomalies.

---

### 19.2 PVC Bound but Pod Mount Failure

Troubleshooting commands:

    kubectl describe pod <pod-name> -n <namespace>
    kubectl describe pvc <pvc-name> -n <namespace>
    kubectl get events -A --sort-by=.lastTimestamp | tail -100
    systemctl status iscsid
    iscsiadm --version

Possible causes:

    Node lacks open-iscsi.
    iscsid not running.
    kubelet mounting failure.
    Longhorn Engine abnormal.
    Instance Manager abnormal.
    Node network issues.

---

### 19.3 Used the Wrong StorageClass

Symptoms:

    PVC is Bound but not created by Longhorn.
    PV's CSI Driver is not Longhorn.
    PVC used default StorageClass.

Troubleshooting:

    kubectl get pvc -n <namespace> -o wide
    kubectl describe pvc <pvc-name> -n <namespace>
    kubectl describe pv <pv-name>
    kubectl get sc

Resolution:

Business PVCs should explicitly specify storageClassName.
Do not fully rely on default StorageClass when multiple storage systems coexist.
Adjust the default StorageClass as needed.

---

### 19.4 Node Disk Pressure

Troubleshooting:

    kubectl describe node <node-name>
    df -hT
    lsblk
    du -sh /data/longhorn
    dmesg | tail -100

Risks:

    Volume write failure.
    Replica cannot be rebuilt.
    Node DiskPressure.
    Pod evicted.
    Longhorn data directory full.

---

## Twenty, Experiment Cleanup

### 20.1 Delete Experimental PVC

If PVCs were created:

    kubectl delete -f pvc-no-storageclass.yaml

If Longhorn PVC templates were created:

    kubectl delete -f pvc-longhorn-demo.yaml

---

### 20.2 Delete Namespace

After confirming no important resources remain:

    kubectl delete namespace storage-demo

Check:

    kubectl get ns storage-demo

---

### 20.3 Clean Up Local YAML Files

If no longer needed:

    rm -f pvc-no-storageclass.yaml
    rm -f pvc-longhorn-demo.yaml

---

## Twenty-one, Interview Answer Approach

If asked in an interview:

    What is Longhorn? What is its relationship with PV, PVC, and CSI?

You can answer:

    Longhorn is a Kubernetes cloud-native distributed block storage system, primarily providing dynamic PV/PVC capabilities through CSI. It can organize local disks across multiple nodes into persistent Volumes usable by Pods, and enhance data availability through Replica replication mechanisms.
    In Kubernetes, Pods should not directly concern themselves with underlying storage systems. Users typically create PVCs, specifying capacity, access mode, and StorageClass. If a PVC specifies Longhorn StorageClass, Kubernetes will use Longhorn CSI to call Longhorn, creating a Longhorn Volume backend, then automatically creating a PV and binding the PVC to the PV. After mounting the PVC, Pods can read and write data as if using a regular directory.
    CSI is the standard interface for Kubernetes to connect with external storage systems. Longhorn implements volume creation, mounting, unmounting, expansion, and snapshot capabilities through CSI.
    Longhorn differs from MinIO. Longhorn is block storage, mainly providing PVCs for Pods; MinIO is object storage, primarily uploading and downloading objects via S3 API. Longhorn is more suitable for medium-scale Kubernetes stateful applications like Jenkins, Prometheus, Redis, and test environment MySQL, but core high-concurrency databases require careful evaluation of disk, network, IOPS, backup, and recovery capabilities.
    When using Longhorn in production, I will focus on node data disks, open-iscsi dependencies, StorageClass parameters, replica count, volume status, replica distribution, backup targets, monitoring alerts, and recovery drills. Longhorn replicas are not backups; important data still requires Backup Targets and regular recovery validation.

---

## Twenty-two, Summary of This Article

This article completed the basic learning of Longhorn:

1. Pods in Kubernetes are temporary; stateful applications need persistent storage.
2. EmptyDir is a temporary directory; data disappears when the Pod is deleted.
3. HostPath binds to the host path, unsuitable for long-term dependency by regular business.
4. PV is a persistent volume resource in Kubernetes.
5. PVC is a user's request for persistent storage.
6. StorageClass is a template for dynamically creating PVs.
7. CSI is the standard interface for Kubernetes to connect with external storage systems.
8. Longhorn provides dynamic PV/PVC capabilities for Kubernetes through CSI.
9. Longhorn is a Kubernetes cloud-native distributed block storage system.
10. Longhorn provides Volume, Replica, Snapshot, Backup, and Restore capabilities.
11. Longhorn is block storage, not object storage.
12. MinIO is object storage; Longhorn is PVC block storage.
13. PVC Pending is usually related to StorageClass, PV, CSI, and Provisioner.
14. In production environments, Longhorn requires attention to node disks, open-iscsi, data directories, replica count, and backups.
15. Longhorn Replica is not a backup; important data still requires Backup Target.
16. The next article will learn about Longhorn architecture: Manager, Engine, Replica, Instance Manager.

---

## Twenty-three, Reference Documents

Longhorn official documentation:

    https://longhorn.io/docs/latest/

What is Longhorn:

    https://longhorn.io/docs/latest/what-is-longhorn/

Longhorn installation documentation:

    https://longhorn.io/docs/latest/deploy/install/

Longhorn nodes and volumes:

    https://longhorn.io/docs/latest/nodes-and-volumes/

Longhorn snapshots and backups:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Longhorn troubleshooting:

    https://longhorn.io/kb/troubleshooting/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Storage Classes:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes CSI documentation:

    https://kubernetes-csi.github.io/docs/