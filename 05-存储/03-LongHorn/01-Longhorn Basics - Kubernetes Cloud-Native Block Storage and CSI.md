# Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI

Recommended Path: 05-Storage/03-LongHorn/01-Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI.md

Tags: #Longhorn #Kubernetes #CSI #PV #PVC #StorageClass #BlockStorage #CloudRawStorage #ApplyWithStatus #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the first article of the Longhorn module, focusing on understanding Longhorn's positioning within the Kubernetes storage architecture.

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

Node Planning:

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
    The production environment is not recommended to place Longhorn data directory on the system disk.
    Longhorn replicas will increase disk usage, for example, 1 10Gi Volume, 3 replicas require about 30Gi raw space.
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
    Redis AOF / RDB files
    Jenkins home directory
    Prometheus TSDB data
    GitLab data directory
    Application upload file directory

These data cannot be lost when the Pod is deleted.

---

### 4.2 Stateless and Stateful Applications

Stateless applications:

    Pods can be rebuilt directly after deletion.
    Data is not saved locally.
    State is usually stored in databases, caches, object storage, or external services.
    Typical applications: frontend, API services, gateways, ordinary microservices.

Stateful applications:

    Data needs to be preserved after Pod deletion.
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
    How to let applications use storage via PVC instead of directly caring about storage backend.

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
    Temporary directory for build processes

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
    May pose security risks.
    Not suitable for direct use by general applications.

Typical scenario: /think

Log Collection DaemonSet  
Node Monitoring Agent  
Access Host Node Socket  
Special System Components  

Production Warnings:

    HostPath is not equal to cloud-native dynamic storage.  
    Ordinary business should not rely on HostPath to save core data.  
    After Pod migration to other nodes, the original node's HostPath data will not be automatically followed.  

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

    User's request for storage resources.  

Can be understood as:  

    Application says: I need a 5Gi persistent storage.  
    Kubernetes is responsible for finding or creating a suitable PV.  
    After PVC is bound to PV, Pod can mount PVC.  

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
    PVC binds to PV  
      |  
      v  
    Pod mounts PVC  

After Longhorn is installed, it usually creates a StorageClass:  

    longhorn  

---

## SixI don't know.PVI don't know.PVCI don't know.StorageClass Relationship Diagram  

### 6.1 Static Provisioning Mode  

In static provisioning mode, administrators create PV first.  

    Administrator creates PV  
          |  
          v  
    User creates PVC  
          |  
          v  
    Kubernetes finds a suitable PV  
          |  
          v  
    PVC binds to PV  
          |  
          v  
    Pod mounts PVC  

Features:  

    Administrators need to prepare PV in advance.  
    Users cannot automatically create new volumes.  
    Suitable for small amounts of fixed storage.  
    Higher operation and maintenance management cost.  

---

### 6.2 Dynamic Provisioning Mode  

In dynamic provisioning mode, users only create PVC.  

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
    PVC binds to PV  
          |  
          v  
    Pod mounts PVC  

Longhorn is one of the storage systems in dynamic provisioning mode.  

---

## SevenI don't know.What is CSI  

### 7.1 CSI Definition  

CSI Full Name:  

    Container Storage Interface  

In Chinese, it can be understood as:  

    Container Storage Interface  

It is the standard interface for Kubernetes to connect with external storage systems.  

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

### 8.1 Longhorn's Position  

Longhorn is a Kubernetes cloud-native distributed block storage system.  

It mainly provides:  

    Dynamic PV/PVC provisioning  
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
    Mounted directory
    Block storage volume

MinIO provides:

    Bucket
    Object
    S3 API
    HTTP / HTTPS Endpoint

Differences:

| Comparison Item | Longhorn | MinIO |
|---|---|---|
| Type | Block storage | Object storage |
| Access Method | PVC mounting | S3 API |
| Form Seen by Application | File system directory | HTTP API |
| Kubernetes Relationship | Deep integration with CSI | Can be deployed on K8s, or deployed independently |
| Typical Use Cases | Database data disk, application persistent volume | Images, attachments, backups, archives |

---

### 8.3 Longhorn is Not a Universal Database Storage

Longhorn can provide PVC for databases, but this does not mean all core databases are suitable for direct operation on Longhorn.

Need to evaluate:

    Node disk performance
    Node network quality
    Replica count
    Write latency
    IOPS requirements
    Database importance
    Backup recovery capability
    Fault drill capability
    Monitoring alert capability

Production reminders:

    Longhorn is suitable for medium-scale Kubernetes stateful applications.
    Core high-concurrency databases require careful evaluation.
    Longhorn replicas are not database backups.
    Databases still need their own backups, such as mysqldump, xtrabackup, pg_dump, WAL archiving, etc.

---

## Nine, Longhorn Suitable Scenarios

### 9.1 Suitable Scenarios

Longhorn is suitable for:

    Medium-scale Kubernetes clusters
    Private deployment environment
    Bare metal Kubernetes
    Edge Kubernetes
    Development/test environment
    Business requiring dynamic PVC
    Ordinary stateful applications
    Applications requiring snapshot and recovery capabilities
    Clusters without cloud vendor cloud disk capabilities

Typical applications:

    Jenkins home
    Prometheus data disk
    Nacos data directory
    Redis persistence
    MySQL test environment
    PostgreSQL test environment
    GitLab small-scale experimental environment
    Ordinary business upload directory
    Middleware configuration and data directory

---

### 9.2 Cautionary Scenarios

Need to be cautious:

    Core high-concurrency MySQL
    High IOPS database
    Low-latency demanding business
    Super-large-scale production database
    Nodes with unstable network environment
    Nodes with significantly different disk performance
    Single-node Kubernetes
    No backup environment
    No monitoring environment
    All replicas located in the same failure domain

---

### 9.3 Not Recommended Scenarios

Not recommended:

    Use Longhorn to replace MinIO for storing object files.
    Use Longhorn to store massive images, attachments, and archive packages.
    Place Longhorn data directory on the system disk and directly go into production.
    Run critical production data without Backup Target.
    Consider data safe without recovery drills.
    Force high replica count when node count is insufficient.
    Run core databases on low-performance disks.

---

## Ten, Practical Operation One: Check Kubernetes Cluster Status

### 10.1 View Nodes

Execute:

    kubectl get nodes -o wide

Expected:

    All nodes are Ready.
    Master and worker node IPs are correct.
    Node versions are consistent or basically consistent.

Example focus points:

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
    Whether CNI components are Running.
    Whether kube-proxy is Running.
    Whether metrics-server (if installed) is Running.
    Whether there are many CrashLoopBackOff.
    Whether there are many Pending.

---

### 10.3 View Events

Execute:

    kubectl get events -A --sort-by=.lastTimestamp | tail -50

If the cluster has frequent abnormal events, they need to be resolved before installing Longhorn.

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

It is not recommended to continue installing Longhorn; node resource issues should be resolved first.

---

## Eleven, Practical Operation Two: Check Current Storage Resources

### 11.1 View StorageClass

Execute:

    kubectl get storageclass

Or:

    kubectl get sc

Possible result one: No StorageClass.

    No resources found

Explanation:

    The cluster currently has no default dynamic storage provisioning capability.
    Creating a regular PVC may result in Pending.
    After installing Longhorn, a new longhorn StorageClass will be added.

Possible result two: Existing StorageClass.

For example: /think

NAME                 PROVISIONER
nfs-client           k8s-sigs.io/nfs-subdir-external-provisioner
local-path           rancher.io/local-path
longhorn             driver.longhorn.io

Notes:

    If Longhorn is already present, it indicates that Longhorn may have been installed already.
    If there are other StorageClasses, you need to understand their differences from Longhorn.

---

### 11.2 Viewing the Default StorageClass

Execute:

    kubectl get sc

If a StorageClass shows:

    (default)

It indicates that it is the default StorageClass.

View details:

    kubectl describe sc <storageclass-name>

Focus on:

    Provisioner
    ReclaimPolicy
    VolumeBindingMode
    AllowVolumeExpansion
    Parameters

---

### 11.3 Viewing PV

Execute:

    kubectl get pv

If there are no PVs:

    No resources found

Notes:

    The current cluster has no created persistent volume resources.

---

### 11.4 Viewing PVC

Execute:

    kubectl get pvc -A

If there are no PVCs:

    No resources found

Notes:

    There are no namespace applications for persistent storage in the current cluster.

---

## TwelveI don't know.Practice Three: Creating a PVC Without Dynamic Provisioning Capability to Observe Pending

### 12.1 Creating an Experimental Namespace

Execute:

    kubectl create namespace storage-demo

View:

    kubectl get ns storage-demo

---

### 12.2 Creating PVC File

Create the file:

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

    This PVC does not explicitly specify a storageClassName.
    If the cluster has no default StorageClass, it will be Pending.
    If the cluster has a default StorageClass, it may automatically use the default StorageClass to create a PV.

---

### 12.3 Applying PVC

Execute:

    kubectl apply -f pvc-no-storageclass.yaml

View:

    kubectl get pvc -n storage-demo

View details:

    kubectl describe pvc demo-pvc-no-storageclass -n storage-demo

---

### 12.4 Observing Result One: PVC Pending

If the PVC status is Pending, common reasons:

    The cluster has no default StorageClass.
    There are no bindable static PVs.
    There is no dynamic Provisioner.
    The PVC cannot find available storage.

You may see similar information in events:

    no persistent volumes available for this claim and no storage class is set

Operations Notes:

    Kubernetes itself does not create storage out of thin air.
    There must be a static PV, or a StorageClass + CSI Provisioner.
    Longhorn can dynamically create PVs via StorageClass after installation.

---

### 12.5 Observing Result Two: PVC Bound

If the PVC status is Bound, it indicates:

    The cluster already has a default StorageClass.
    Kubernetes has created or bound a PV via the default StorageClass.
    You need to check which StorageClass it uses.

Execute:

    kubectl get pvc demo-pvc-no-storageclass -n storage-demo -o wide

View details:

    kubectl describe pvc demo-pvc-no-storageclass -n storage-demo

Check the corresponding PV:

    kubectl get pv

Then view the PV details:

    kubectl describe pv <pv-name>

Focus on:

    StorageClass
    CSI Driver
    Reclaim Policy
    VolumeHandle

---

### 12.6 Cleaning Up Experimental PVC

If you're just observing PVC Pending, you can clean up:

    kubectl delete -f pvc-no-storageclass.yaml

If the PVC is already Bound, note:

    Deleting a PVC may trigger the PV's reclaim policy.
    If a real volume was created using the default StorageClass, confirm whether it's an experimental resource before deletion.

Check the PV reclaim policy:

    kubectl get pv

If confirmed as an experimental resource, delete it:

    kubectl delete -f pvc-no-storageclass.yaml

---

## ThirteenI don't know.Practice Four: Viewing the Impact of Default StorageClass on PVC

### 13.1 Why Pay Attention to the Default StorageClass

The default StorageClass affects PVCs that do not specify a storageClassName.

For example, if a PVC does not include:

    storageClassName: longhorn

And the cluster has a default StorageClass, the PVC will automatically use the default StorageClass.

This may lead to:

Originally intended to use Longhorn, ended up using other storage.
Originally intended to test Pending, it automatically became Bound.
Originally intended to use NFS, ended up using Longhorn.
Confusion easily occurs when multiple storage systems coexist.

---

### 13.2 Viewing the Default StorageClass

Execute:

    kubectl get sc

Observe if the NAME column contains:

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

After Longhorn installation, if you want longhorn to become the default StorageClass:

    kubectl patch storageclass longhorn \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

Production reminder:

    Setting Longhorn as the default StorageClass should be done cautiously.
    When multiple storage systems coexist, it's recommended to explicitly specify storageClassName for business PVCs.
    It's not advisable for all PVCs to blindly use the default StorageClass.

---

## FourteenI don't know.Practice 5: Creating PVC Template with Explicit StorageClass Specification

### 14.1 PVC Template After Longhorn Installation

After Longhorn installation, you can create the following PVC:

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
    If Longhorn is installed and the longhorn StorageClass exists, this PVC should automatically become Bound.
    A full validation will be covered in 05-Longhorn Dynamic Volume Practice.

---

### 14.2 Why It's Recommended to Explicitly Specify StorageClass

Recommended:

    storageClassName: longhorn

Reasons:

    Avoid misusing the default StorageClass.
    Clearer when multiple storage systems coexist.
    Facilitates migration and troubleshooting.
    Enables different storage strategies for different business uses.

---

## Fifteen、Longhorn and PV/PVC Workflow Chain

### 15.1 Complete Workflow Chain

Longhorn dynamic volume creation chain:

    User creates PVC
        |
        v
    PVC specifies storageClassName: longhorn
        |
        v
    Kubernetes detects need for dynamic provisioning
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

### 15.2 Troubleshooting Chain

If PVC cannot be used, troubleshoot along the chain:

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
    Are node disks normal?
    Are node networks normal?

---

## Sixteen、Understanding Comparison Between Longhorn and Ceph RBD

### 16.1 Commonalities

Longhorn and Ceph RBD can both provide block storage capabilities for Kubernetes.

Commonalities:

    Both can provide PVC via CSI.
    Both can dynamically create PVs.
    Both can mount as file system directories for Pods.
    Both are suitable for stateful applications.
    Both require attention to replicas, fault recovery, monitoring, and backups.

---

### 16.2 Differences

| Comparison Item | Longhorn | Ceph RBD |
|---|---|---|
| Deployment Complexity | Low | High |
| Operations Complexity | Medium | High |
| Underlying Architecture | Volume / Engine / Replica | RADOS / OSD / Pool / PG |
| Kubernetes Integration | Native for K8s | CSI Integration |
| Suitable Scale | Medium/Small K8s Scenarios | Medium/Large Unified Storage Platform |
| Learning Focus | CSI, Volume, Replica, Backup | OSD, PG, CRUSH, Pool, RBD |
| UI Management | Longhorn UI is Intuitive | Ceph Dashboard is More Complex |
| Production Capabilities | Easy to Use, but Requires Understanding of Boundaries | Powerful, but High Complexity |

Simple Understanding:

    Longhorn is more like an internal cloud disk for Kubernetes.
    Ceph RBD is more like a general-purpose distributed block storage foundation.

---

## Seventeen, Understanding Comparison Between Longhorn and MinIO

| Comparison Item | Longhorn | MinIO |
|---|---|---|
| Storage Type | Block Storage | Object Storage |
| Kubernetes Usage | PVC | S3 API or Application Access |
| Data Unit | Volume | Object |
| Top-Level Container | PVC / PV / Volume | Bucket |
| Access Protocol | CSI / Mount File System | HTTP / HTTPS S3 |
| Typical Ports | Kubernetes Internal | 9000 / 9001 |
| Typical Applications | MySQL, Redis, Prometheus | Images, Attachments, Backup Packages, Log Archives |
| Data Protection | Replica, Replica Reconstruction, Backup | Erasure Coding, Bucket Backup, mirror |

One-Sentence Differentiation:

    Use Longhorn when an application needs a single persistent data disk.
    Use MinIO when an application needs an object upload/download interface.

---

## Eighteen, Basic Principles of Longhorn from a Production Perspective

### 18.1 Do Not Treat Replicas as Backups

Longhorn Replica Resolves:

    Node Failure
    Disk Failure
    Replica Reconstruction
    Volume Availability

It Cannot Resolve:

    Accidental PVC Deletion
    Accidental Volume Deletion
    Application Corruption
    Accidental File Deletion
    Cluster-Wide Failure
    All Replicas Destroyed

Therefore:

    Replica is Not Backup.
    Snapshot is Not Fully Equivalent to异地 Backup.
    Production Must Configure Backup Target.
    Important Data Must Perform Recovery Drills.

---

### 18.2 Do Not Treat System Disk as Data Disk

System Disk Can Temporarily Be Used in Experimental Environments.

Production Recommendations:

    Each Worker Node Should Mount Independent Data Disk.
    Use /data/longhorn as Longhorn Data Directory.
    Confirm df -hT Shows Data Directory Mounted on Expected Disk.
    Do Not Let Longhorn Data Fill Up System Disk.

---

### 18.3 Do Not Blindly Set High Replica Count

Higher Replica Count:

    May Improve Availability.
    Consumes More Disk Space.
    Write Amplification is More Obvious.
    Network Synchronization Pressure is Greater.
    Replica Reconstruction Takes Longer.

Example:

    10Gi Volume, 3 Replicas Occupy ~30Gi Raw Space.
    Writing Data Requires Synchronizing to Multiple Replicas.
    Replica Distribution is Ineffective When Node Count is Insufficient.

Production Recommendations:

    Common Replica Count is 2 or 3.
    At Least Enough Worker Nodes to Support Replica Distribution.
    Do Not Blindly Set 3 Replicas on 2 Nodes and Assume Full High Availability.

---

### 18.4 Do Not Ignore iSCSI Dependencies

Longhorn Volume Mount Relies on Node-Side Capabilities.

If Nodes Lack open-iscsi or iscsid is Not Running, Issues May Occur:

    Pod Mount Failure.
    Volume Attach Failure.
    MountVolume Errors in kubelet Events.
    PVC Bound but Pod Cannot Be Used.

Check:

    systemctl status iscsid
    iscsiadm --version

---

## Nineteen, Common Issue Troubleshooting

### 19.1 PVC Stays Pending

Troubleshooting Commands:

    kubectl get pvc -n storage-demo
    kubectl describe pvc demo-pvc-no-storageclass -n storage-demo
    kubectl get sc
    kubectl get pv
    kubectl get events -A --sort-by=.lastTimestamp | tail -50

Possible Causes:

    No Default StorageClass.
    Specified StorageClass Does Not Exist.
    Longhorn Not Installed.
    Longhorn CSI Components Abnormal.
    Node Disk Insufficient.
    StorageClass Parameters Abnormal.

---

### 19.2 PVC Bound but Pod Mount Failure

Troubleshooting Commands:

    kubectl describe pod <pod-name> -n <namespace>
    kubectl describe pvc <pvc-name> -n <namespace>
    kubectl get events -A --sort-by=.lastTimestamp | tail -100
    systemctl status iscsid
    iscsiadm --version

Possible Causes:

    Node Lacks open-iscsi.
    iscsid Not Running.
    kubelet Mount Failure.
    Longhorn Engine Abnormal.
    Instance Manager Abnormal.
    Node Network Issues.

---

### 19.3 Used Wrong StorageClass

Symptoms:

    PVC Bound but Not Created by Longhorn.
    PV's CSI Driver is Not Longhorn.
    PVC Used Default StorageClass.

Troubleshooting:

    kubectl get pvc -n <namespace> -o wide
    kubectl describe pvc <pvc-name> -n <namespace>
    kubectl describe pv <pv-name>
    kubectl get sc

Resolution: /think

# 19. Node Disk Pressure

## Troubleshooting:

    kubectl describe node <node-name>
    df -hT
    lsblk
    du -sh /data/longhorn
    dmesg | tail -100

## Risks:

    Volume write failure.
    Replica cannot be rebuilt.
    Node DiskPressure.
    Pod evicted.
    Longhorn data directory full.

---

## 20. Experiment Cleanup

### 20.1 Delete Experimental PVC

If PVC was created:

    kubectl delete -f pvc-no-storageclass.yaml

If Longhorn PVC template was created:

    kubectl delete -f pvc-longhorn-demo.yaml

---

### 20.2 Delete Namespace

After confirming no important resources exist:

    kubectl delete namespace storage-demo

Check:

    kubectl get ns storage-demo

---

### 20.3 Clean Up Local YAML Files

If no longer needed:

    rm -f pvc-no-storageclass.yaml
    rm -f pvc-longhorn-demo.yaml

---

## 21. Interview Answer Approach

If asked in an interview:

    Longhorn is a Kubernetes cloud-native distributed block storage system, primarily providing dynamic PV/PVC capabilities through CSI. It can organize local disks across multiple nodes into persistent Volumes usable by Pods, and enhance data availability through Replica replication mechanisms.
    In Kubernetes, Pods should not directly concern themselves with underlying storage systems. Users typically create PVCs, specifying capacity, access mode, and StorageClass. If a PVC specifies Longhorn StorageClass, Kubernetes will use Longhorn CSI to call Longhorn, creating a Longhorn Volume backend, then automatically creating a PV and binding the PVC to the PV. After mounting the PVC, Pods can read/write data as if using a regular directory.
    CSI is the standard interface for Kubernetes to connect with external storage systems. Longhorn implements volume creation, mounting, unmounting, expansion, and snapshot capabilities through CSI.
    Longhorn differs from MinIO. Longhorn is block storage, primarily providing PVCs for Pods; MinIO is object storage, mainly uploading/downloading objects via S3 API. Longhorn is more suitable for medium-scale Kubernetes stateful applications like Jenkins, Prometheus, Redis, and test environment MySQL, but core high-concurrency databases require careful evaluation of disk, network, IOPS, backup, and recovery capabilities.
    When using Longhorn in production, I will focus on node data disks, open-iscsi dependencies, StorageClass parameters, replica count, volume status, replica distribution, backup targets, monitoring alerts, and recovery drills. Longhorn replicas are not backups; important data still requires Backup Targets and regular recovery verification.

---

## 22. Summary of This Chapter

This chapter completed the basics of Longhorn learning:

1. Pods in Kubernetes are ephemeral; stateful applications need persistent storage.
2. EmptyDir is a temporary directory; data disappears when the Pod is deleted.
3. HostPath binds to the host path, unsuitable for long-term dependency by regular business.
4. PV is a persistent volume resource in Kubernetes.
5. PVC is a user's request for persistent storage.
6. StorageClass is a template for dynamically creating PVs.
7. CSI is the standard interface for Kubernetes to connect with external storage systems.
8. Longhorn provides dynamic PV/PVC capabilities to Kubernetes through CSI.
9. Longhorn is a Kubernetes cloud-native distributed block storage system.
10. Longhorn provides Volume, Replica, Snapshot, Backup, and Restore capabilities.
11. Longhorn is block storage, not object storage.
12. MinIO is object storage; Longhorn is PVC block storage.
13. PVC Pending is usually related to StorageClass, PV, CSI, and Provisioner.
14. In production environments, Longhorn requires attention to node disks, open-iscsi, data directories, replica count, and backups.
15. Longhorn Replica is not a backup; important data still requires Backup Target.
16. The next chapter will learn Longhorn architecture: Manager, Engine, Replica, Instance Manager.

---

## 23. Reference Documents

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