# Longhorn Directory Index

Recommended path: 05-Storage/03-LongHorn/00-Longhorn Directory Index.md

Tags: #Longhorn #Kubernetes #CSI #Block Storage #PV #PVC #StorageClass #Cloud-Native Storage #Advanced SRE #Production Operations and Maintenance

---

## I. Module Overview

Longhorn is a cloud-native distributed block storage system designed for Kubernetes.

Its main functions include:

    Providing persistent block storage capabilities for Kubernetes
    Connecting with Kubernetes PV/PVC/StorageClass through CSI
    Organizing local node disks into distributed volumes that can be used by Pods
    Offering features such as replication, snapshots, backup, recovery, and failover for stateful applications

Longhorn is neither object storage nor a traditional shared file system. Instead, it is more akin to a distributed cloud disk system in the Kubernetes context. It can be simply understood as follows:

    MinIO addresses the needs of S3 object storage
    Ceph RBD handles general-purpose distributed block storage
    Longhorn solves the block storage challenges within Kubernetes

---

## II. Learning Objectives

After completing this Longhorn module, you should be able to:

1. Understand Longhorn's role in the Kubernetes storage ecosystem.
2. Comprehend the relationship between Longhorn and PV, PVC, StorageClass, and CSI.
3. Know the functions of Longhorn Manager, Engine, Replica, and Instance Manager.
4. Be capable of planning Longhorn nodes, disks, dependent components, and StorageClasses.
5. Install Longhorn using Helm.
6. View Longhorn charts, images, values.yaml files, and version information.
7. Resolve image retrieval issues without affecting the containerd/Kubernetes underlying runtime.
8. Create PVCs and verify that Pods can mount Longhorn volumes successfully.
9. Understand how the number of Longhorn replicas, node distribution, and data high availability are related to each other.
10. Configure Backup Targets.
11. Perform Snapshot, Backup, and Restore operations.
12. Diagnose issues such as Volume degradation, Replica reconstruction, and node abnormalities.
13. Analyze the performance limitations of Longhorn from a production perspective.
14. Establish methods for routine inspection, monitoring, backup, and failover of Longhorn systems.
15. Determine suitable and unsuitable use cases for Longhorn.

---

## III. Experimental Environment Planning

### 3.1 Experimental IP Range

This storage module will uniformly use the following IP range:

    10.0.0.0/24

Existing nodes:

| IP | Host Name | Purpose |
|---|---|---|
| 10.0.0.10 | ops-server | GitLab, Jenkins, Harbor, operations services |
| 10.0.0.20 | k8s-master01 | Kubernetes Master node |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker node |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker node |

Longhorn is typically deployed within the current Kubernetes cluster.

---

### 3.2 Kubernetes Experimental Environment

Default experimental environment settings:

    Kubernetes: Kubeadm cluster
    Operating system: Ubuntu Server 22.04.5 LTS
    Container runtime: containerd
    CNI: Calico
    Number of nodes: 1 Master + 2 Workers

Example node allocation:

| Node | IP | Role | Recommended for Longhorn data storage |
|---|---|---|---|
| k8s-master01 | 10.0.0.20 | Control Plane | Suitable for experiments; not recommended in production |
| k8s-worker01 | 10.0.0.21 | Worker node | Recommended |
| k8s-worker02 | 10.0.0.22 | Worker node | Recommended |

Production recommendations:

    Longhorn data replicas should preferably be distributed across Worker nodes.
    Whether Master nodes should participate in storage should be determined based on factors such as resources, resource contamination, reliability, and cluster size.
    In production environments, it is not advisable to concentrate all replicas on a single node.
    It is also not recommended to store Longhorn data directories on the system disk.

---

### 3.3 Longhorn Data Directory Planning

For the experimental environment, you can initially use the default directory:

    /var/lib/longhorn

In production scenarios, it is recommended to mount an independent data disk for storage:

    /data/longhorn

Example configuration:

| Node | Data Directory | Notes |
|---|---|---|
| k8s-worker01 | /data/longhorn | Independent data disk |
| k8s-worker02 | /data/longhorn | Independent data disk |

Verification commands:

    df -hT
    lsblk
    mount | grep longhorn

    kubectl -n longhorn-system get volumes.longhorn.io

---

### 5.6 Engine

The Engine is the data read and write component of Longhorn Volume.

It can be understood as:

    The control component for the data side of each Volume

Its responsibilities include:

    Receiving read and write requests
    Synchronizing writes to multiple Replicas
    Managing the operational status of the Volume
    Coordinating replica reconstruction

---

### 5.7 Replica

A Replica is a copy of the data in a Longhorn Volume.

For example, if there are 3 replicas:

    A Volume will have 3 Replicas
    The Replicas are distributed across different nodes as much as possible
    If a node fails, as long as there are healthy replicas, the Volume can still be used or restored

Methods to view them:

    Longhorn UI
    kubectl -n longhorn-system get replicas.longhorn.io

---

### 5.8 Instance Manager

The Instance Manager is used to manage the running instances of Longhorn Engines and Replicas.

It can be understood as:

    The instance manager for the Longhorn data side

Command to view:

    kubectl -n longhorn-system get pods | grep instance-manager

---

### 5.9 Longhorn Manager

The Longhorn Manager is the core component of the Longhorn control plane.

Its responsibilities include:

    Managing nodes
    Managing disks
    Managing Volumes
    Managing Replicas
    Managing Engines
    Managing backup and recovery
    Collaborating with the Kubernetes API and Longhorn CRDs

Command to view:

    kubectl -n longhorn-system get pods | grep longhorn-manager

---

## VI. Longhorn Module Notes Directory

### 00-Longhorn Directory Index

File:

    00-Longhorn Directory Index.md

Objective:

    To provide an overview of the learning path for Longhorn modules.
    To clarify the experimental environment, node planning, learning boundaries, and practical objectives.
    To establish the relationships between Longhorn and Ceph, MinIO, RustFS.

---

### 01-Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI

File:

    01-Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI.md

Key Points:

    What is Longhorn.
    What is cloud-native block storage.
    Why does Kubernetes need CSI.
    The relationship between PV, PVC, and StorageClass.
    Scenarios where Longhorn is suitable.
    Scenarios where Longhorn is not suitable.

Practical Objectives:

    To view the storage resources in a Kubernetes cluster.
    To view StorageClasses.
    To create the smallest possible PVC.
    To understand the process of binding PVC to PV.

---

### 02-Longhorn Architecture: Manager, Engine, Replica, Instance Manager

File:

    02-Longhorn Architecture: Manager, Engine, Replica, Instance Manager.md

Key Points:

    The distinction between Longhorn's control plane and data plane.
    The role of the Manager.
    The role of the Engine.
    The role of Replicas.
    The role of the Instance Manager.
    The complete process of creating a Volume from a PVC to its mounting in a Pod.

Practical Objectives:

    To view the longhorn-system namespace.
    To view Longhorn Pods.
    To view CRDs.
    To observe changes in Volume, Engine, and Replica after creating a PVC.
    To understand Longhorn's internal objects using kubectl.

---

### 03-Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass

File:

    03-Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass.md

Key Points:

    What must be checked before installation.
    open-iscsi / iscsid.
    NFSv4 client.
    mount propagation.
    Data disk paths.
    Node resources.
    Default behavior of StorageClass.
    Why not blindly proceed with installation.

Practical Objectives:

    To check the Kubernetes version.
    To inspect the node system.
    To install dependencies.
    To verify iscsid.
    To check the data directory.
    To examine the node disks and mount points.
    To plan the /data/longhorn directory structure.

---

### 04-Longhorn Helm Installation Methodology: Charts, Images, values.yaml, and Version Management

File:

    04-Longhorn Helm Installation Methodology: Charts, Images, values.yaml, and Version Management.md

Key Points:

    Why the Helm methodology is recommended.
    How to view a Longhorn Chart.
    How to view values.yaml.
    How to filter images.
    How to fix versions.
    How to define values.yaml.
    How to handle domestic networks and image repositories.
    Why not directly modify containerd configurations.

Practical Objectives:

    To add the longhorn Helm repository.
    To display a Longhorn Chart.
    ToCheck the node configuration for any pending PVC creation tasks. Ensure that there are no network issues preventing the creation of PVs. If necessary, adjust the resource allocation or resolve any dependencies.```markdown
kubectl describe pvc <pvc-name>
kubectl get sc
kubectl -n longhorn-system get pods
kubectl -n longhorn-system logs -l app=longhorn-manager --tail=100

Common causes:

    The StorageClass does not exist.
    Longhorn CSI encounters an exception.
    The node's disk space is insufficient.
    Longhorn has not been fully installed.
    The default StorageClass configuration does not meet expectations.

---

### 12.2 Pod mounting failure

Troubleshooting:

    kubectl describe pod <pod-name>
    kubectl describe pvc <pvc-name>
    kubectl get events -A --sort-by=.lastTimestamp
    systemctl status iscsid
    iscsiadm --version

Common causes:

    open-iscsi is not installed.
    iscsid is not running.
    The kubelet fails to mount the volume.
    The Volume has not been attached.
    The node is experiencing an issue.
    There is a problem with the Longhorn Engine/Instance Manager.

---

### 12.3 Volume degradation

Troubleshooting:

    Use the Longhorn UI.
    kubectl -n longhorn-system get volumes.longhorn.io
    kubectl -n longhorn-system get replicas.longhorn.io
    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

Common causes:

    A certain replica has failed.
    A node is offline.
    A disk is unavailable.
    The disk capacity is insufficient.
    Replica reconstruction is in progress.
    The number of replicas is too low.

---

### 12.4 Node disk issues

Troubleshooting:

    kubectl get nodes
    kubectl describe node <node-name>
    df -hT
    lsblk
    dmesg | tail -100
    journalctl -k --since "1 hour ago"
    kubectl -n longhorn-system get nodes.longhorn.io

---
## Section Thirteen: Longhorn Security and Production Considerations

### 13.1 UI access security

The Longhorn UI should not be directly exposed to the public network.

Recommendations:

    Only allow access within the private network.
    Implement authentication when exposing it through Ingress.
    Restrict access only to the operations network segment.
    Do not grant management permissions to regular business personnel.
    Use HTTPS for production access.
    Configure RBAC and access controls.

---

### 13.2 Data security

Note:

    Longhorn replicas are not considered backups.
    Snapshots are not equivalent to offsite backups.
    The Backup Target provides true backup capabilities.
    Backups must be configured for production use.
    Regular recovery tests should be performed on backups.
    Avoid storing all backups in the same failure domain.

---

### 13.3 Operation safety

High-risk operations include:

    Deleting PVCs
    Deleting PVs
    Deleting Longhorn Volumes
    Deleting Replicas
    Clearing /data/longhorn
    Formatting data disks
    Changing the number of replicas
    Batch restarting Longhorn components
    Uninstalling Longhorn
    Modifying the default StorageClass

Production requirements:

    Confirm the impact on business before making any changes.
    Ensure backups are in place before proceeding.
    Prepare a rollback plan in advance.
    High-risk operations should be reviewed by two people.
    Keep detailed records of all operations.

---
## Section Fourteen: Longhorn and Interview Answers

If asked in an interview:

    What is Longhorn? How do you understand it?

You can answer:

    Longhorn is a cloud-native distributed block storage system for Kubernetes. It provides dynamic PV/PVC capabilities through CSI, enabling the organization of local disks across multiple nodes to create persistent volumes for Pods. The replica mechanism ensures data availability and resilience in case of failures.
    From a Kubernetes perspective, when a user creates a PVC and specifies a Longhorn StorageClass, Kubernetes dynamically generates a PV using Longhorn CSI. Longhorn then manages the creation of Volume, Engine, and replicas in the backend. Once a Pod mounts this PVC, data is stored on the Longhorn Volume, with Longhorn handling replica synchronization and fault recovery.
    Longhorn is suitable for small to medium-sized Kubernetes clusters, private environments, edge scenarios, and situations where dynamic PV supply is required. It is not the same as object storage and should not replace solutions like MinIO for S3 use cases.
    In production, I would pay close attention to factors such as node disk planning, open-iscsi dependencies, StorageClass settings, the number of replicas, whether data directories are on separate disks, Volume degradation status, replica reconstruction processes, Backup Target configuration, and the effectiveness of monitoring and alerting mechanisms.
    Additionally, it is important to emphasize that Longhorn replicas are not backups. Even with multiple replicas, errors like accidental PVC