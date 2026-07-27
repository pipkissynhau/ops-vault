# Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI

Recommended Path: 05-Storage/03-LongHorn/01-Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI.md

Tags: #Longhorn #Kubernetes #CSI #PV #PVC #StorageClass #Block Storage #Cloud-Native Storage #Stateful Applications #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the first in the Longhorn series, focusing on understanding Longhorn's role within the Kubernetes storage ecosystem.

Rather than diving straight into installation, this document will first address the following key questions:

    Why does Kubernetes require persistent storage?
    What are PVs, PVCs, and StorageClasses?
    What is CSI?
    Why is Longhorn considered a cloud-native block storage solution for Kubernetes?
    What problems can Longhorn help solve?
    What problems is Longhorn not suitable for solving?
    How does Longhorn differ from solutions like MinIO and Ceph?
    What Kubernetes storage objects should be checked before installing Longhorn?

This article emphasizes practicality, providing step-by-step instructions using kubectl commands to observe the current storage resources in your Kubernetes cluster, laying the foundation for subsequent Longhorn installation and PVC setup.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand why Pods in Kubernetes need persistent storage.
2. Distinguish between EmptyDir, HostPath, PVs, PVCs, and StorageClasses.
3. Comprehend the role of CSI in the Kubernetes storage framework.
4. Recognize that Longhorn is a cloud-native distributed block storage solution for Kubernetes.
5. Differentiate Longhorn from object storage solutions like MinIO.
6. Understand the relationship and differences between Longhorn and Ceph RBD.
7. Learn how to view existing StorageClasses in a Kubernetes cluster.
8. Know how to access PV and PVC resources.
9. Create a PVC without a specified StorageClass and observe its Pending status.
10. Explain why PVCs cannot automatically bind to PVs without dynamic storage plugins.
11. Be prepared for dynamically creating PVCs after installing Longhorn.

---

## III. Experimental Environment

### 3.1 Kubernetes Cluster

Default experimental environment:

    Kubernetes: Kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node IP Range: 10.0.0.0/24

Node configuration:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane |
| 10.0.0.21 | k8s-worker01 | Worker |
| 10.0.0.22 | k8s-worker02 | Worker |

---

### 3.2 Longhorn Data Directory Planning

For experimental purposes, the default directory for Longhorn is:

    /var/lib/longhorn

In a production environment, it is recommended to use an independent data disk:

    /data/longhorn

Check the available directories using these commands:

    df -hT
    lsblk
    mount | grep longhorn

Production note:

    In an experimental setting, you can use the system's default data directory.
    However, in a production environment, it is not advisable to store Longhorn data on the system disk.
    Longhorn copies will increase disk usage significantly; for example, 10Gi of storage space may require approximately 30Gi of raw space when three replicas are created.
    Longhorn's performance heavily depends on the node's disk capacity and network quality.

---

## IV. Why Does Kubernetes Need Persistent Storage

### 4.1 Pods Are Temporary by Nature

Pods in Kubernetes have the following characteristics:

    Pods can be deleted.
    Pods can be recreated.
    Pods can be drained.
    Pods can be scheduled to different nodes.
    The file systems inside Pods change throughout their lifecycle.
    When a Deployment is updated, new Pods may be created to replace old ones.

If business data is stored only within the container's file system, it will be lost when the Pod is deleted or recreated.

Examples of data that should not be lost:

    MySQL data directories
    PostgreSQL data directories
    Redis AOF/RDB files
    Jenkins home directories
    Prometheus TSDB data
    GitLab data directories
    Application upload folders

---

### 4.2 Stateless and Stateful Applications

Stateless applications:

    Can be recreated immediately after deletion.
    Do not retain data locally.
    Their state is usually stored in databases, caches, object storage, or external services.
    Examples: Frontends, API services, gateways, general microservices.

Stateful applications:

    Require data to be retained after a Pod is deleted.
    Usually need stable and persistent storage### CSI Provisioner Calls the Storage System
          |
          v
    Automatically creates Volume
          |
          v
    Automatically creates PV
          |
          v
    PVC is bound to PV
          |
          v
    Pod mounts the PVC

Longhorn is one of the storage systems that use a dynamic provisioning model.

---

## VII. What is CSI?

### 7.1 Definition of CSI

CSI stands for:

    Container Storage Interface

In Chinese, it can be understood as:

    Container Storage Interface

It is the standard interface through which Kubernetes connects with external storage systems.

The purpose of CSI is:

    Kubernetes does not come with all storage drivers built-in.
    Storage vendors or open-source storage systems can integrate with Kubernetes through CSI plugins.
    Kubernetes uses a unified interface to perform operations such as creating, mounting, unmounting, scaling, and taking snapshots of volumes.

---

### 7.2 What Can CSI Do?

CSI supports:

    Dynamically creating volumes
    Deleting volumes
    Mounting volumes
    Unmounting volumes
    Scaling volumes
    Creating snapshots
    Restoring from snapshots
    Querying volume status

For Kubernetes:

    It only needs to call storage plugins through the CSI standard.

For Longhorn:

    Longhorn provides its volumes for Kubernetes to use via CSI.

---

### 7.3 The Role of CSI in Longhorn

After Longhorn is installed, it deploys CSI-related components within Kubernetes.

These components are responsible for:

    Listening for PVC creation requests.
    Creating Longhorn volumes.
    Binding volumes to Kubernetes PVs.
    Attaching volumes to target nodes.
    Helping kubelet mount volumes onto Pods.
    Supporting volume scaling.
    Supporting snapshot functionality.

Simple workflow:

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
    Pod mounts and uses the volume

---

## VIII. What is Longhorn?

### 8.1 Positioning of Longhorn

Longhorn is a cloud-native distributed block storage system for Kubernetes.

It primarily offers:

    Dynamic provision of PVs/PVCs
    Distributed block storage volumes
    Multi-replica data protection
    Snapshotting
    Backup and recovery capabilities
    Volume scaling
    Replica reconstruction after node failures
    A web-based management interface
    Integration with CSI

Its core purpose is to provide persistent block storage for stateful Pods in Kubernetes.

---

### 8.2 Longhorn is Not an Object Storage System

Longhorn is not similar to MinIO.

It does not directly use the S3 API to upload objects to applications.

Instead, Longhorn provides:

    PVCs
    Mountable directories
    Block storage volumes

MinIO, on the other hand, offers:

    Buckets
    Objects
    S3 API
    HTTP/HTTPS endpoints

Differences:

| Comparison Item | Longhorn | MinIO |
|---|---|---|
| Type | Block storage | Object storage |
| Access Method | PVC mounting | S3 API |
| Form seen by applications | File system directories | HTTP API |
    Relationship with Kubernetes | Deeply integrated with CSI | Can be deployed on K8s or independently |
| Typical Uses | Database data disks, persistent volumes for applications | Images, attachments, backups, archiving |

---

### 8.3 Longhorn is Not a Universal Database Storage Solution

While Longhorn can provide PVCs for databases, not all core databases are suitable to run directly on Longhorn.

It is necessary to evaluate:

    Node disk performance
    Network quality of nodes
    Number of replicas
    Write latency
    IOPS requirements
    Importance of the database
    Backup and recovery capabilities
    Disaster recovery readiness
    Monitoring and alerting systems

Production considerations:

    Longhorn is suitable for small to medium-sized Kubernetes applications with stateful components.
    Core, high-concurrency databases require careful evaluation before using Longhorn.
    Longhorn's replicas do not serve as a backup for databases.
    Databases still need their own dedicated backup mechanisms, such as mysqldump, xtrabackup, pg_dump, or WAL archives.

---

## IX. Suitable Scenarios for Longhorn

### 9.1 Suitable Scenarios

Longhorn is suitable for:

    Small to medium-sized Kubernetes clusters
    Private deployment environments
    Bare-metal Kubernetes setups
    Edge Kubernetes applications
    Development and testing environments
    Applications that require dynamic PVCs
    General stateful applications
    Applications that need snapshotting and recovery capabilities
    Clusters without cloud provider's built-in storage solutions

Typical use cases include:

    Jenkins home directories
    Prometheus data disks
    Nacos data directories
    Redis persistent storage
    MySQL test environments
    PostgreSQL test environments
    Small-scale GitLab experiment setups
### 11.2 Checking the Default StorageClass

Execute:

    kubectl get sc

If a certain StorageClass shows:

    (default)

it indicates that it is the default StorageClass.

To view details:

    kubectl describe sc <storage-class-name>

Pay attention to:

    Provisioner
    ReclaimPolicy
    VolumeBindingMode
    AllowVolumeExpansion
    Parameters

---

### 11.3 Checking PVs

Execute:

    kubectl get pv

If no PVs are displayed:

    No resources found

This means:

    There are no created persistent volume resources in the current cluster.

---

### 11.4 Checking PVCs

Execute:

    kubectl get pvc -A

If no PVCs are displayed:

    No resources found

This indicates:

    No namespace in the current cluster has requested persistent storage.

---

## Section Twelve: Practical Exercise Three:Creating a PVC Without Dynamic Provisioning Capability to Observe Pending Status

### 12.1 Creating an Experimental Namespace

Execute:

    kubectl create namespace storage-demo

To check:

    kubectl get ns storage-demo

---

### 12.2 Creating a PVC File

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

Explanation:

    This PVC does not explicitly specify a storageClassName.
    If the cluster does not have a default StorageClass, it will remain in Pending status.
    If the cluster has a default StorageClass, it may automatically use that class to create a PV.

---

### 12.3 Applying the PVC

Execute:

    kubectl apply -f pvc-no-storageclass.yaml

To check:

    kubectl get pvc -n storage-demo

For detailed information:

    kubectl describe pvc demo-pvc-no-storageclass -n storage-demo

---

### 12.4 Observation Result One: PVC in Pending Status

If the PVC status is Pending, common reasons include:

    The cluster does not have a default StorageClass.
    There are no static PVs available for binding.
    No dynamic Provisioner is configured.
    The PVC cannot find available storage.

You may see similar messages in the events:

    no persistent volumes available for this claim and no storage class is set

Operational understanding:

    Kubernetes itself does not create storage resources out of thin air.
    There must be either static PVs, or a StorageClass combined with a CSI Provisioner.
    After installing Longhorn, it is possible to dynamically create PVs using the StorageClass feature.

---

### 12.5 Observation Result Two: PVC Bound

If the PVC status is Bound, it means:

    The current cluster has a default StorageClass.
    Kubernetes has already created or bound a PV using the default StorageClass.
    You need to check which StorageClass is being used.

Execute:

    kubectl get pvc demo-pvc-no-storageclass -n storage-demo -o wide

For detailed information:

    kubectl describe pvc demo-pvc-no-storageclass -n storage-demo

To view the corresponding PV:

    kubectl get pv

Then, to check details of the PV:

    kubectl describe pv <pv-name>

Pay attention to:

    StorageClass
    CSI Driver
    ReclaimPolicy
    VolumeHandle

---

### 12.6 Cleaning Up the Experimental PVC

If you only want to observe the Pending status of the PVC, you can clean it up:

    kubectl delete -f pvc-no-storageclass.yaml

If the PVC is already Bound, be aware that:

    Deleting the PVC may trigger its ReclaimPolicy.
    If a real volume was created using the default StorageClass, make sure to confirm whether it is an experimental resource before deleting it.

To check the PV Reclaim Policy:

    kubectl get pv

If you confirm it is an experimental resource, then delete it:

    kubectl delete -f pvc-no-storageclass.yaml

---

## Section Thirteen: Practical Exercise Four: Understanding the Impact of the Default StorageClass on PVCs

### 13.1 Why Pay Attention to the Default StorageClass

The default StorageClass affects PVCs that do not explicitly specify a storageClassName.

For example, if the PVC does not contain:

    storageClassName: longhorn

and the cluster has a default StorageClass, the PVC will automatically use that class.

This can lead to:

    Using a different storage system than intended.
    The PVC being Bound instead of remaining in Pending status.
    Using Longhorn when NFS was intended.
    Potential confusion when multiple storage systems coexist.

---

### 13.2 Checking the Default StorageClass

Execute:

    kubectl get sc        v
    Kubernetes identifies the need for dynamic provisioning
        |
        v
    Calls the Longhorn CSI Provisioner
        |
        v
    Longhorn creates a Volume
        |
        v
    Longhorn creates an Engine and Replica
        |
        v
    Kubernetes creates a PV
        |
        v
    The PVC is bound to the PV
        |
        v
    The Pod mounts the PVC
        |
        v
    The application writes data
        |
        v
    Data is written to the Longhorn Volume
        |
        v
    Data is synchronized to multiple Replicas

---

### 15.2 Troubleshooting Steps

If a PVC becomes unavailable, troubleshoot along the following chain:

    Has the PVC been created?
    Is the PVC bound?
    Does the StorageClass exist?
    Are the CSI components functioning properly?
    Is the Longhorn Manager working correctly?
    Has the Longhorn Volume been created?
    Are the Replicas functioning normally?
    Has the Pod been successfully scheduled?
    Has the kubelet successfully mounted it?
    Is iscsid running?
    Are the node disks in good condition?
    Is the node network functioning properly?

---

## Chapter Sixteen: Comparison between Longhorn and Ceph RBD

### 16.1 Similarities

Both Longhorn and Ceph RBD can provide block storage capabilities for Kubernetes.

Similarities:

    Both can offer PVCs through CSI.
    Both can dynamically create PVs.
    Both can be mounted as file system directories in Pods.
    Both are suitable for stateful applications.
    Both require attention to replication, fault recovery, monitoring, and backup.

---

### 16.2 Differences

| Comparison Item | Longhorn | Ceph RBD |
|---|---|---|
| Deployment Complexity | Lower | Higher |
| Operational Complexity | Medium | High |
| Underlying Architecture | Volume / Engine / Replica | RADOS / OSD / Pool / PG |
    Kubernetes Integration | Natively designed for K8s | Connected via CSI |
    Suitable Scale | Small to medium-sized K8s environments | Large-scale unified storage platforms |
| Key Learning Areas | CSI, Volume, Replica, Backup | OSD, PG, CRUSH, Pool, RBD |
| UI Management | Longhorn UI is intuitive | Ceph Dashboard is more complex |
    Productivity | Easy to use but requires understanding of limitations | Powerful but more complex |

In simple terms:

    Longhorn is more like an internal cloud disk within Kubernetes.
    Ceph RBD serves as a general-purpose distributed block storage foundation.

---

## Chapter Seventeen: Comparison between Longhorn and MinIO

| Comparison Item | Longhorn | MinIO |
|---|---|---|
| Storage Type | Block storage | Object storage |
    How it is Used in Kubernetes | PVCs | S3 API or application integration |
    Data Unit | Volume | Object |
    Top-Level Container | PVC / PV / Volume | Bucket |
    Access Protocol | CSI / Mounted file system | HTTP / HTTPS S3 |
    Typical Ports | Internal to Kubernetes cluster | 9000 / 9001 |
    Common Use Cases | MySQL, Redis, Prometheus | Images, attachments, backup files, log archiving |
    Data Protection | Replica replication, backup recovery | Erasure Coding, bucket backups, mirroring |

In one sentence:

    Use Longhorn if your application requires a persistent block storage drive.
    Choose MinIO if you need an object storage interface for data upload and download.

---

## Chapter Eighteen: Basic Principles of Longhorn in Production Environments

### 18.1 Do Not Consider Replicas as Backups

Longhorn Replicas are designed to handle:

    Node failures
    Disk failures
    Replica reconstruction
    Ensuring Volume availability

However, they cannot prevent:

    Accidental deletion of PVCs or Volumes
    Data corruption caused by application errors
    File deletions
    Complete cluster failures
    Destruction of all replicas

Therefore:

    Replicas are not equivalent to backups.
    Snapshots are not always a reliable form of off-site backup.
    It is essential to configure Backup Targets in production environments.
    Regular recovery tests must be conducted for critical data.

---

### 18.2 Do Not Use the System Disk as a Data Disk

In a testing environment, using the system disk may be temporary.

In a production environment, it is recommended:

    Each Worker node should have its own dedicated data disk.
    The /data/longhorn directory should be used specifically for Longhorn data.
    Ensure that df -hT shows the data directory mounted on the expected disk.
    Prevent Longhorn data from filling up the system disk.

---

### 18.3 Avoid Setting an Excessively High Number of Replicas

More replicas generally mean:

    Better availability
    Increased disk usage
    More significant write amplification
    Greater network synchronization load
    Longer replica reconstruction times

```bash
rm -f pvc-no-storageclass.yaml
rm -f pvc-longhorn-demo.yaml

---

## 21. Interview Response Strategies

If you are asked in an interview:

What is Longhorn? What is its relationship with PV, PVC, and CSI?

You can answer as follows:

Longhorn is a cloud-native distributed block storage system for Kubernetes that primarily provides dynamic PV/PVC capabilities through CSI. It allows multiple local disks on different nodes to be organized into persistent Volumes that can be used by Pods, and it enhances data availability through the Replica mechanism.
In Kubernetes, Pods should not directly concern themselves with the underlying storage system. Users typically create PVCs, specifying the required capacity, access mode, and StorageClass to use. If a PVC specifies the Longhorn StorageClass, Kubernetes will utilize Longhorn CSI to interact with Longhorn, creating a Longhorn Volume in the backend, automatically generating a PV, and binding it to the PVC. Once a Pod mounts the PVC, it can access data just like using a regular directory.
CSI is the standard interface in Kubernetes for connecting to external storage systems. Longhorn uses CSI to enable functions such as volume creation, mounting, unmounting, scaling, and snapshotting.
Longhorn is different from MinIO. While MinIO is an object storage system that provides object upload and download services via the S3 API, Longhorn is a block storage system designed specifically for Kubernetes to provide PVCs for Pods. Longhorn is more suitable for small to medium-sized Kubernetes applications with stateful components, such as Jenkins, Prometheus, Redis, or test environments using MySQL. However, for core high-concurrency databases, it's important to carefully evaluate the combination of disk performance, network capabilities, IOPS, backup and recovery options.
When deploying Longhorn in a production environment, key considerations include the node data disks, open-iscsi dependencies, StorageClass settings, the number of replicas, Volume status, Replica distribution, Backup Targets, monitoring and alerting mechanisms, and disaster recovery plans. It's important to note that Longhorn Replicas are not designed for backup purposes; critical data should still be backed up using dedicated targets and regularly verified through restoration tests.

---

## 22. Summary of This Article

This article has covered the basics of Longhorn:

1. In Kubernetes, Pods are temporary by nature, so stateful applications require persistent storage.
2. EmptyDir is a temporary directory whose contents are lost when the Pod is deleted.
3. HostPath binds to the host machine's path and is not suitable for long-term use in production scenarios.
4. PVs represent persistent volume resources in Kubernetes.
5. PVCs are user-defined requests for persistent storage resources.
6. StorageClasses serve as templates for dynamically creating PVs.
7. CSI is the standard interface in Kubernetes for integrating with external storage systems.
8. Longhorn utilizes CSI to provide dynamic PV/PVC capabilities to Kubernetes.
9. Longhorn is a cloud-native distributed block storage system for Kubernetes.
10. It offers features such as Volume management, Replica replication, snapshotting, backup, and restoration.
11. Unlike MinIO, which is an object storage system, Longhorn specializes in block storage for PVCs.
12. Issues with PVC Pending status are often related to StorageClass settings, PV configuration, CSI interactions, or Provisioner mechanisms.
13. In a production environment using Longhorn, attention should be paid to node disk performance, open-iscsi compatibility, data directory management, the number of replicas, and backup strategies.
14. While Longhorn Replicas are not backups, it is still essential to have dedicated Backup Targets for critical data.
15. The next article will delve into the architecture of Longhorn, including components such as Manager, Engine, Replica, and Instance Manager.

---
## 23. References

Longhorn Official Documentation:

https://longhorn.io/docs/latest/

What is Longhorn:

https://longhorn.io/docs/latest/what-is-longhorn/

Longhorn Installation Guide:

https://longhorn.io/docs/latest/deploy/install/

Longhorn Nodes and Volumes:

https://longhorn.io/docs/latest/nodes-and-volumes/

Longhorn Snapshots and Backups:

https://longhorn.io/docs/latest/snapshots-and-backups/

Longhorn Troubleshooting:

https://longhorn.io/kb/troubleshooting/

Kubernetes Persistent Volumes:

https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Storage Classes:

https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes CSI Documentation:

https://kubernetes-csi.github.io/docs/
```