# Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI

Recommended Path: 05-Storage/03-LongHorn/01-Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI.md

Tags: #Longhorn #Kubernetes #CSI #PV #PVC #StorageClass #Block Storage #Cloud-Native Storage #Stateful Applications #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the first in the Longhorn module, focusing on understanding Longhorn's role within the Kubernetes storage ecosystem.

Rather than directly jumping into installation, this article will first address the following questions:

    Why does Kubernetes require persistent storage?
    What are PVs, PVCs, and StorageClasses?
    What is CSI?
    Why is Longhorn considered a cloud-native block storage solution for Kubernetes?
    What problems can Longhorn help solve?
    What problems cannot it solve?
    How does Longhorn differ from MinIO and Ceph?
    What Kubernetes storage objects should be checked before installing Longhorn?

This article emphasizes practicality and will use `kubectl` commands to observe the current storage resources in the Kubernetes cluster, laying the foundation for subsequent Longhorn installation and PVC setup.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand why Pods in Kubernetes need persistent storage.
2. Differentiate between EmptyDir, HostPath, PVs, PVCs, and StorageClasses.
3. Comprehend the role of CSI in the Kubernetes storage system.
4. Recognize that Longhorn is a cloud-native distributed block storage solution for Kubernetes.
5. Distinguish Longhorn from object storage solutions like MinIO.
6. Understand the relationship and differences between Longhorn and Ceph RBD.
7. Learn how to view existing StorageClasses in a Kubernetes cluster.
8. Know how to view PV and PVC resources.
9. Create a PVC without a specified StorageClass and observe its Pending status.
10. Understand why PVCs cannot automatically bind to corresponding PVs without dynamic storage plugins.
11. Prepare for dynamically creating PVCs after installing Longhorn.

---

## III. Experimental Environment

### 3.1 Kubernetes Cluster

Default experimental environment:

    Kubernetes: Kubeadm cluster
    Operating system: Ubuntu Server 22.04.5 LTS
    Container runtime: containerd
    CNI: Calico
    Node IP range: 10.0.0.0/24

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

In a production environment, it is recommended to use an independent data disk for mounting:

    /data/longhorn

Check commands:

    df -hT
    lsblk
    mount | grep longhorn

Production note:

    In the experimental environment, you can use the system disk directory temporarily.
    In a production environment, it is not advised to place Longhorn data on the system disk.
    Longhorn copies will increase disk usage significantly; for example, 1 10Gi Volume with 3 replicas will require approximately 30Gi of original space.
    Longhorn's performance heavily depends on the node's disk and network quality.

---

## IV. Why Does Kubernetes Need Persistent Storage

### 4.1 Pods Are Temporary by Nature

Pods in Kubernetes have the following characteristics:

    Pods can be deleted.
    Pods can be recreated.
    Pods can be terminated.
    Pods can be scheduled to other nodes.
    The file system inside a Pod changes throughout its lifecycle.
    When a Deployment is updated, new Pods may be created to replace old ones.

If business data is stored only in the container's file system, it will be lost when the Pod is deleted or recreated.

Examples:

    MySQL data directory
    PostgreSQL data directory
    Redis AOF/RDB files
    Jenkins home directory
    Prometheus TSDB data
    GitLab data directory
    Application upload files directory

These data must not be lost when Pods are removed.

---

### 4.2 Stateless and Stateful Applications

Stateless applications:

    Can be recreated immediately after a Pod is deleted.
    Data is not stored locally.
    Information about the state is typically retained in databases, caches, object storage, or external services.
    Typical examples: Frontends, API services, gateways, general microservices.

Stateful applications:

    Require data to be preserved after a Pod is deleted.
    Usually need stable and persistent storage.
    Rebuilt Pods mustThe CSI Provisioner calls the storage system.
          |
          v
    Automatically creates a Volume.
          |
          v
    Automatically creates a PV.
          |
          v
    Binds the PVC to the PV.
          |
          v
    The Pod mounts the PVC.

Longhorn is one of the storage systems that use dynamic provisioning models.

---

## VII. What is CSI?

### 7.1 Definition of CSI

The full name of CSI is:

    Container Storage Interface

In Chinese, it can be understood as:

    Container Storage Interface

It is a standard interface for Kubernetes to connect with external storage systems.

The goal of CSI is:

    Kubernetes does not come with all storage drivers built-in.
    Storage vendors or open-source storage systems can integrate into Kubernetes through CSI plugins.
    Kubernetes can perform operations such as volume creation, mounting, unmounting, scaling, and snapshotting through a unified interface.

---

### 7.2 What Can CSI Do?

CSI can support:

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

    Longhorn provides its volumes for Kubernetes to use through CSI.

---

### 7.3 The Role of CSI in Longhorn

After Longhorn is installed, it deploys CSI-related components within Kubernetes.

These components are responsible for:

    Listening for PVC creation requests.
    Creating Longhorn volumes.
    Binding the volumes to Kubernetes PVs.
    Attaching the volumes to target nodes.
    Assisting kubelet in mounting volumes onto Pods.
    Supporting volume scaling.
    Providing snapshot capabilities.

Simple diagram:

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

It mainly provides:

    Dynamic provision of PVs/PVCs
    Distributed block storage volumes
    Multi-replica data protection
    Snapshots
    Backup and restoration capabilities
    Volume scaling
    Replica reconstruction after node failures
    A web-based management interface
    CSI integration

Its core purpose is to provide persistent block storage for stateful Pods in Kubernetes.

---

### 8.2 Longhorn Is Not Object Storage

Longhorn is not MinIO.

Longhorn does not directly upload objects to applications through the S3 API.

What Longhorn provides is:

    PVCs
    Mountable directories
    Block storage volumes

MinIO, on the other hand, provides:

    Buckets
    Objects
    S3 API
    HTTP/HTTPS endpoints

Differences:

| Comparison Item | Longhorn | MinIO |
|---|---|---|
| Type | Block storage | Object storage |
| Access Method | PVC mounting | S3 API |
| Form Seen by Applications | File system directories | HTTP API |
| Relationship with Kubernetes | Deeply integrated with CSI | Can be deployed on K8s or independently |
| Typical Uses | Database data disks, persistent application volumes | Images, attachments, backups, archiving |

---

### 8.3 Longhorn Is Not a Universal Database Storage Solution

While Longhorn can provide PVCs for databases, it does not mean that all core databases are suitable to run directly on Longhorn.

It is necessary to evaluate:

    Node disk performance
    Network quality of nodes
    Number of replicas
    Write latency
    IOPS requirements
    Importance of the database
    Backup and restoration capabilities
    Disaster recovery capabilities
    Monitoring and alerting systems

Production considerations:

    Longhorn is suitable for small to medium-sized Kubernetes applications with stateful components.
    Core, high-concurrency databases require careful evaluation before using it.
    Longhorn's replicas are not designed as a backup system for databases.
    Databases still need their own dedicated backup mechanisms, such as mysqldump, xtrabackup, pg_dump, or WAL backups.

---

## IX. Suitable Scenarios for Longhorn

### 9.1 Suitable Scenarios

Longhorn is suitable for:

    Small to medium-sized Kubernetes clusters
    Privately deployed environments
    Bare-metal Kubernetes setups
    Edge Kubernetes applications
    Development and testing environments
    Businesses that require dynamic PVCs
    General stateful applications
    Applications that need snapshotting and recovery capabilities
    Clusters without cloud provider's cloud disk services

Typical uses:

    Jenkins home directory
    Prometheus data disks
    Nacos data directories
    Redis persistent storage
    MySQL testing environments
    PostgreSQL testing environments
   ### 11.2 Viewing the Default StorageClass

Execute:

    kubectl get sc

If a certain StorageClass displays as:

    (default)

it indicates that it is the default StorageClass.

To view details:

    kubectl describe sc <storage-class-name>

Pay special attention to:

    Provisioner
    ReclaimPolicy
    VolumeBindingMode
    AllowVolumeExpansion
    Parameters

---

### 11.3 Viewing PVs

Execute:

    kubectl get pv

If no PVs are displayed:

    No resources found

This means that there are no persistent volume resources created in the current cluster.

---

### 11.4 Viewing PVCs

Execute:

    kubectl get pvc -A

If no PVCs are displayed:

    No resources found

This indicates that no namespace has requested persistent storage in the current cluster.

---

## Chapter Twelve: Practical Exercise Three: Creating a PVC Without Dynamic Provisioning to Observe Pending Status

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
    If the cluster does not have a default StorageClass, it will remain in the Pending state.
    If there is a default StorageClass, it may automatically use that class to create a PV.

---

### 12.3 Applying the PVC

Execute:

    kubectl apply -f pvc-no-storageclass.yaml

To check:

    kubectl get pvc -n storage-demo

For detailed information:

    kubectl describe pvc demo-pvc-no-storageclass -n storage-demo

---

### 12.4 Observing Result One: PVC in Pending State

If the PVC status is Pending, common reasons include:

    The cluster does not have a default StorageClass.
    There are no static PVs available for binding.
    No dynamic Provisioner is configured.
    The PVC cannot find available storage.

You may see similar messages in the events:

    no persistent volumes available for this claim and no storage class is set

Operational understanding:

    Kubernetes itself does not create storage resources out of thin air.
    There must be either static PVs or a StorageClass combined with a CSI Provisioner.
    After installing Longhorn, it is possible to dynamically create PVs using the StorageClass feature.

---

### 12.5 Observing Result Two: PVC Bound

If the PVC status is Bound, it means that:

    The current cluster has a default StorageClass.
    Kubernetes has already created or bound a PV using the default StorageClass.
    It is necessary to check which specific StorageClass is being used.

Execute:

    kubectl get pvc demo-pvc-no-storageclass -n storage-demo -o wide

For detailed information:

    kubectl describe pvc demo-pvc-no-storageclass -n storage-demo

To view the corresponding PV:

    kubectl get pv

Then, to check the details of the PV:

    kubectl describe pv <pv-name>

Pay special attention to:

    StorageClass
    CSI Driver
    ReclaimPolicy
    VolumeHandle

---

### 12.6 Clearing the Experimental PVC

If you only want to observe the PVC in the Pending state, you can clear it:

    kubectl delete -f pvc-no-storageclass.yaml

If the PVC is already Bound, be cautious:

    Deleting a PVC may trigger its ReclaimPolicy.
    If a real volume was created using the default StorageClass, make sure it is an experimental resource before deleting it.

To check the PV Reclaim Policy:

    kubectl get pv

If you confirm it is an experimental resource, then delete it:

    kubectl delete -f pvc-no-storageclass.yaml

---

## Chapter Thirteen: Practical Exercise Four: Understanding the Impact of the Default StorageClass on PVCs

### 13.1 Why Pay Attention to the Default StorageClass

The default StorageClass affects PVCs that do not explicitly specify a storageClassName.

For example, if the PVC does not contain:

    storageClassName: longhorn

and the cluster has a default StorageClass, the PVC will automatically use that class.

This can lead to:

    Using a different storage system than intended.
    The PVC being Bound instead of remaining in the Pending state.
    Using Longhorn when NFS was expected.
    Potential confusion when multiple storage systems coexist.

---

### 13.2 Viewing the Default StorageClass

Execute:

    kubectl get        v
    Kubernetes determines a need for dynamic provisioning.
        |
        v
    It invokes the Longhorn CSI Provisioner.
        |
        v
    Longhorn creates the Volume.
        |
        v
    Longhorn generates the Engine and Replica.
        |
        v
    Kubernetes establishes the PV.
        |
        v
    The PVC is bound to the PV.
        |
        v
    The Pod mounts the PVC.
        |
        v
    The application writes data.
        |
        v
    The data is stored in the Longhorn Volume.
        |
        v
    The data is synchronized across multiple Replicas.

---

### 15.2 Troubleshooting Steps

If a PVC becomes unusable, follow this sequence to diagnose the issue:

    - Has the PVC been created?
    - Is the PVC bound to a PV?
    - Does the corresponding StorageClass exist?
    - Are all CSI components functioning correctly?
    - Is the Longhorn Manager operational?
    - Has the Longhorn Volume been successfully created?
    - Are all Replicas intact?
    - Has the Pod been successfully scheduled?
    - Has the kubelet successfully mounted the Volume?
    - Is the iscsid service running?
    - Are the node disks in good condition?
    - Is the node network functioning properly?

---

## Chapter Sixteen: Comparing Longhorn with Ceph RBD

### 16.1 Similarities

Both Longhorn and Ceph RBD can provide block storage for Kubernetes.

Similarities:

    - Both support creating PVCs through CSI.
    - Both allow for dynamic creation of PVs.
    - Both can be mounted as file system directories in Pods.
    - Both are suitable for stateful applications.
    - Both require attention to replication, fault recovery, monitoring, and backup strategies.

---

### 16.2 Differences

| Comparison Item | Longhorn | Ceph RBD |
|---|---|---|
| Deployment Complexity | Lower | Higher |
| Operational Complexity | Moderate | High |
| Underlying Architecture | Volume / Engine / Replica | RADOS / OSD / Pool / PG |
| Kubernetes Integration | Natively integrated for K8s | Requires CSI mediation |
| Suitable Scale | Ideal for small to medium-sized K8s deployments | Designed for large-scale unified storage solutions |
| Key Learning Areas | CSI, Volume Management, Replication, Backup | OSD, PG, CRUSH, Pool Management, RBD |
| User Interface | Longhorn offers a user-friendly GUI | Ceph's Dashboard is more complex |
| Productivity | Easy to use but requires understanding of limitations | Powerful but demanding in terms of setup and management |

In summary:

    - Longhorn is more akin to an internal cloud disk within Kubernetes.
    - Ceph RBD serves as a more general-purpose distributed block storage solution.

---

## Chapter Seventeen: Comparing Longhorn with MinIO

| Comparison Item | Longhorn | MinIO |
|---|---|---|
| Storage Type | Block storage | Object storage |
| Usage in Kubernetes | Through PVCs | Via S3 APIs or application integration |
    Data Unit | Volumes | Objects |
    Top-Level Containers | PVCs/PVs/Volumes | Buckets |
    Access Protocols | CSI/mount as file system | HTTP/HTTPS via S3 |
    Typical Ports | Internal to Kubernetes cluster | 9000/9001 |
    Common Use Cases | MySQL, Redis, Prometheus | Image storage, file sharing, backup archiving |
    Data Protection Mechanisms | Replication, data restoration, backup | Erasure Coding, bucket backups, mirroring |

To put it simply:

    - Choose Longhorn if you need a persistent block storage solution for your Kubernetes applications.
    - Opt for MinIO if you require an object storage service with flexible access methods.

---

## Chapter Eighteen: Basic Principles for Using Longhorn in Production

### 18.1 Do Not Consider Replicas as Backups

Longhorn’s Replica mechanism is designed to handle:

    - Node failures
    - Disk failures
    - Replica reconstitution
    - Ensuring Volume availability

However, it cannot prevent:

    - Accidental deletion of PVCs or Volumes
    - Data corruption caused by application errors
    - Complete system failures that affect all replicas

Therefore:

    - Replicas are not a substitute for backups.
    - Snapshots alone are not sufficient for off-site data protection.
    - It is essential to configure backup targets in production environments.
    - Regular recovery tests should be conducted for critical data.

---

### 18.2 Do Not Use System Disks as Data Disks

In experimental settings, system disks may be temporarily used for Longhorn storage.

However, in production:

    - Each Worker node should have its own dedicated data disk.
    /data/longhorn directory should be used specifically for Longhorn data.
    Ensure that this directory is mounted on a suitable disk using df -hT.
```bash
rm -f pvc-no-storageclass.yaml
rm -f pvc-longhorn-demo.yaml

---

## 21. Interview Answer Guidelines

If you are asked in an interview:

What is Longhorn? How does it relate to PV, PVC, and CSI?

You can answer as follows:

Longhorn is a cloud-native distributed block storage system for Kubernetes that provides dynamic PV/PVC capabilities through CSI. It allows multiple local disks on different nodes to be organized into persistent Volumes that can be used by Pods. Longhorn enhances data availability by using the Replica mechanism.
In Kubernetes, Pods should not need to worry directly about the underlying storage system. Users typically create PVCs, specifying the required capacity, access mode, and StorageClass. If a PVC specifies the Longhorn StorageClass, Kubernetes will use Longhorn CSI to interact with Longhorn, creating a Longhorn Volume in the backend, automatically generating a PV, and binding the PVC to that PV. Once a Pod mounts the PVC, it can use it just like any regular directory for data storage and retrieval.
CSI is the standard interface in Kubernetes for integrating with external storage systems. Longhorn utilizes CSI to enable various operations such as volume creation, mounting, unmounting, scaling, and snapshotting.
Longhorn differs from MinIO in that Longhorn is designed as a block storage system, primarily providing PVCs for Pods, while MinIO is an object storage system that uses the S3 API for data upload and download. Longhorn is more suitable for small to medium-sized Kubernetes applications that require stateful storage, such as Jenkins, Prometheus, Redis, or test environments with MySQL. However, for critical high-concurrency databases, it's essential to carefully evaluate the combination of disk performance, network reliability, IOPS, backup capabilities, and recovery strategies.
When using Longhorn in a production environment, key considerations include node data disks, open-iscsi dependencies, StorageClass settings, the number of replicas, Volume status, replica distribution, Backup Targets, monitoring and alerting mechanisms, and disaster recovery plans. It's important to note that Longhorn Replicas are not intended as backups; critical data should still be backed up using dedicated targets and regularly verified for integrity.

---

## 22. Summary of This Article

This article has provided an overview of the basics of Longhorn:

1. In Kubernetes, Pods are temporary by design, so stateful applications require persistent storage solutions.
2. EmptyDir is a temporary directory whose contents are lost when the Pod is terminated.
3. HostPath binds to the host machine's files and directories, making it unsuitable for long-term use in production scenarios.
4. PVs are persistent volume resources in Kubernetes that provide storage for Pods.
5. PVCs represent user requests for persistent storage resources.
6. StorageClasses serve as templates for dynamically creating PVs based on specific requirements.
7. CSI is the standard interface in Kubernetes for integrating with external storage systems.
8. Longhorn utilizes CSI to offer dynamic PV/PVC management within Kubernetes.
9. Longhorn is a cloud-native distributed block storage system designed for Kubernetes.
10. It provides various features such as Volume management, replication, snapshots, backup, and recovery.
11. Unlike MinIO, which is an object storage system, Longhorn specializes in block storage for Pods.
12. Issues with PVCs, such as "PVC Pending," are often related to StorageClasses, PVs, CSI, or provisioners.
13. In a production environment using Longhorn, attention should be paid to node disks, open-iscsi configurations, data directories, the number of replicas, and backup strategies.
14. While Longhorn Replicas help in data availability, critical data still requires dedicated backup solutions.
15. The next article will delve into the architecture of Longhorn, including components such as Manager, Engine, Replica, and Instance Manager.

---

## 23. References

Official Longhorn Documentation:

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