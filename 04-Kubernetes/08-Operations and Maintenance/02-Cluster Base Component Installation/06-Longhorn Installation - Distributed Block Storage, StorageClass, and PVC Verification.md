# 06-Longhorn Installation: Distributed Block Storage, StorageClass, and PVC Verification

Recommended Path:

    04-Kubernetes/08-Operations/02-Cluster Basic Components Installation/06-Longhorn Installation: Distributed Block Storage, StorageClass, and PVC Verification.md

Tags:

    #Kubernetes
    #Longhorn
    #StorageClass
    #PV
    #PVC
    #CSI
    #Distributed Block Storage
    #Persistent Storage
    #Cluster Basic Components

---

## I. Document Description

This document records the installation, verification, and basic usage of Longhorn in a Kubernetes cluster.

Longhorn is a native distributed block storage system for Kubernetes that provides dynamic storage capabilities based on CSI.

This document uses:

    Longhorn
    Helm
    StorageClass
    PVC
    Pod Mount Verification
    Longhorn UI

Objectives of this document:

    1. Install the prerequisites for Longhorn.
    2. Check if nodes meet Longhorn requirements.
    3. Install Longhorn using Helm.
    4. Configure Longhorn's default data directory.
    5. Verify Longhorn StorageClass.
    6. Create PVCs.
    7. Create Pods that mount Longhorn PVCs.
    8. View volume status through the Longhorn UI.
    9. Understand how to troubleshoot common issues.

Applicable Scenarios:

    1. Self-built kubeadm clusters.
    2. Private environments.
    3. Medium and small-scale production environments.
    4. Situations requiring block storage capabilities stronger than NFS.
    5. Scenarios that require replication, snapshots, backup, and volume migration features.

Notes:

    Longhorn is not simply an NFS shared directory; it is a distributed block storage system.
    Before using Longhorn in a production environment, carefully plan the disk configuration, network setup, node roles, backup strategies, and disaster recovery plans.

---

## II. Differences Between Longhorn and NFS

| Item | NFS Dynamic Provisioning | Longhorn |
|---|---|---|
| Type | File-sharing storage | Distributed block storage |
| Backend | Single or highly available NFS Server | Distributed storage composed of multiple local disks on nodes |
| Access Mode | Commonly RWX | Commonly RWO; RWX requires a Share Manager |
| Suitable Scenarios | Shared directories, configuration files, non-core data | Stateful applications, block storage, light-to-midweight database scenarios |
    High Availability | Depends on the HA capabilities of the NFS Server itself | Achieved through multiple replicas |
| Management Complexity | Relatively simple | Requires attention to disk management, replication, rebuilding, and backup |
| Operational Complexity | Lower | Moderate |

In simple terms:

    NFS is more like a shared folder.
    Longhorn is more like providing distributed cloud storage for Kubernetes.

---

## III. Planning Information

### 3.1 Longhorn Planning

| Item | Planning Details |
|---|---|
| Namespace | longhorn-system |
| Installation Method | Helm |
| Longhorn Version | 1.11.1 |
| Default Data Directory | /data/longhorn |
| Default StorageClass | longhorn |
| Default Number of Replicas | 3 |
| Access UI Method | Port-forward or Ingress |
| Suitable Nodes | Primarily Worker nodes |

---

### 3.2 Node Disk Planning

It is recommended to prepare an independent data disk or directory for each Worker node:

    k8s-worker-01    /data/longhorn
    k8s-worker-02    /data/longhorn
    k8s-worker-03    /data/longhorn

Note:

    The /data/longhorn directory should preferably be located on an independent disk or a large-capacity partition.
    If /data is just an ordinary directory under the root partition, it will still occupy system disks.
    In a production environment, it is not recommended to mix Longhorn data with system disks.

---

## IV. Pre-Deployment Checks

### 4.1 Check Cluster Status

Execute:

    kubectl get nodes -o wide

Requirement:

    All nodes must be in the Ready state.

---

### 4.2 Check Helm

Execute:

    helm version

---

### 4.3 Check Node Disks

On each node where Longhorn will be installed, execute the following commands:

    df -h
    lsblk
    sudo mkdir -p /data/longhorn
    df -h /data/longhorn

Requirements:

    The partition where /data/longhorn is located must have sufficient capacity.
    It is not recommended to place the Longhorn data directory in a small root partition.

---

### 4.4 Check kubelet Mount Propagation

Longhorn relies on Kubernetes nodes supporting volume mounting properly.

First, confirm that kubelet is running normally:

    systemctl status kubelet --no-pager

Also```yaml
defaultSettings:
  defaultDataPath: /data/longhorn
  storageReservedPercentageForDefaultDisk: "10"
  storageOverProvisioningPercentage: "100"
  replicaAutoBalance: best-effort

persistence:
  defaultClass: true
  defaultClassReplicaCount: 3
  defaultFsType: ext4
  reclaimPolicy: Delete

longhornUI:
  replicas: 1
EOF
```

### Explanation:

- `defaultSettings.defaultDataPath`: The default data directory for Longhorn.
- `storageReservedPercentageForDefaultDisk`: The percentage of reserved space for the default disk.
- `storageOverProvisioningPercentage`: The proportion of extra storage allocated. Set it to 100 here to avoid excessive overprovisioning.
- `persistence.defaultClass`: Whether to create a default Longhorn StorageClass.
- `persistence.defaultClassReplicaCount`: The number of replicas for the default volume. It is recommended to set it to 3 for 3 Worker nodes.
- `reclaimPolicy`: The policy for reclaiming resources after a PVC is deleted.If the disk space is insufficient, it may also result in the failure to schedule replicas.

---

## XII. Deleting Test Resources

Delete the Pod:

    kubectl delete pod longhorn-test-pod

Delete the PVC:

    kubectl delete pvc longhorn-test-pvc

Check the PV:

    kubectl get pv

If the reclaimPolicy is set to Delete, the PV will be deleted after the PVC is removed.

Confirm in the Longhorn UI whether the corresponding Volume has been deleted.

---

## XIII. Troubleshooting Common Issues

### 13.1 Longhorn Pod ImagePullBackOff

Check:

    kubectl -n longhorn-system get pods -o wide

Check events:

    kubectl -n longhorn-system describe pod <pod-name>

Common causes:

    1. Unable to access docker.io
    2. Unable to pull the longhornio image
    3. The internal Harbor has not synchronized the image
    4. Abnormalities with containerd network or proxy

Solution:

    1. Check the image configuration in the Helm values
    2. Synchronize the Longhorn image to the internal Harbor
    3. Update the values and then perform a helm upgrade

---

### 13.2 longhornctl check preflight fails

Execute:

    longhornctl check preflight

Common causes:

    1. open-iscsi is not installed
    2. iscsid is not running
    3. nfs-common is not installed
    4. cryptsetup is not installed
    5. dmsetup is unavailable
    6. The node kernel does not support NFSv4
    7. Abnormalities with mount propagation

Solution:

    sudo apt install -y open-iscsi nfs-common cryptsetup dmsetup util-linux

    sudo systemctl enable --now iscsid

---

### 13.3 PVC remains in the Pending state

Check the PVC:

    kubectl describe pvc longhorn-test-pvc

Check the StorageClass:

    kubectl get storageclass

Check the Longhorn Pod:

    kubectl -n longhorn-system get pods -o wide

Check the Longhorn Volume:

    kubectl -n longhorn-system get volumes.longhorn.io

Common causes:

    1. The Longhorn CSI component is not running
    2. The StorageClass name is incorrect
    3. The node disk is unavailable for scheduling
    4. The number of replicas exceeds the available storage nodes
    5. Abnormalities with the Longhorn manager

---

### 13.4 The Pod cannot mount the PVC

Check the Pod:

    kubectl describe pod longhorn-test-pod

Check the kubelet logs:

    journalctl -u kubelet -f

Check the Longhorn CSI:

    kubectl -n longhorn-system get pods -o wide | grep csi

Common causes:

    1. Abnormalities with open-iscsi
    2. iscsid is not running
    3. Abnormalities with the Longhorn CSI plugin
    4. The volume was not successfully attached
    5. The node cannot access the Longhorn engine or replicas

---

### 13.5 Insufficient Longhorn Volume replicas

Check the Longhorn UI:

    Volume
    Replicas
    Events

Or check the CRD:

    kubectl -n longhorn-system get replicas.longhorn.io

Common causes:

    1. The number of worker nodes is insufficient
    2. Some node disks have insufficient space
    3. Nodes are marked as unavailable for scheduling
    4. The disk path does not exist
    5. The number of replicas is set too high

Solution:

    1. Increase the number of available storage nodes
    2. Reduce the number of replicas
    3. Check the /data/longhorn directory
    4. Verify the status of Longhorn Nodes and disks

---

### 13.6 The Longhorn UI is inaccessible

Check the Service:

    kubectl -n longhorn-system get svc longhorn-frontend

Check the Pod:

    kubectl -n longhorn-system get pod -l app=longhorn-ui

Use port-forwarding:

    kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80 --address 0.0.0.0

If using Ingress:

    kubectl -n longhorn-system describe ingress longhorn-ui

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=100

---

## XIV. Upgrading and Rolling Back

### 14.1 Checking the Current Version

Execute:

    helm list -n longhorn-system

    helm status longhorn -n longhorn-system

Check history:

   11. The Longhorn UI allows you to view the health status of volumes.

---

## Section Seventeen: Summary

This document outlines the basic installation and verification process for Longhorn.

Key Points:

    1. Install prerequisites for Longhorn
    2. Start iscsid
    3. Create the /data/longhorn data directory
    4. Use longhornctl to perform preflight checks
    5. Deploy Longhorn using Helm
    6. Configure the default data directory for Longhorn
    7. Create a Longhorn StorageClass
    8. Create PVCs to verify dynamic provisioning
    9. Create Pods to test mounting and reading/writing capabilities
    10. View volumes, replicas, and health status through the UI
    11. Troubleshoot issues such as ImagePullBackOff failures, pending PVCs, mounting errors, and insufficient replicas

Production Recommendations:

    1. Place the Longhorn data directory on a separate disk or large-capacity partition.
    2. Avoid using it alongside the system disk.
    3. Determine the number of replicas based on the number of nodes and disk capacity.
    4. Ensure that a backup target is configured in a production environment.
    5. Always follow the official Longhorn upgrade guidelines for any production upgrades.
    6. Restrict access to the Longhorn UI to prevent public exposure.
    7. Conduct thorough failure and recovery drills before deploying core data services with Longhorn.