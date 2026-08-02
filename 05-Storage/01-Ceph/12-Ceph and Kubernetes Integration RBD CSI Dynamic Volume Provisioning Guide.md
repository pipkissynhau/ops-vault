# Ceph and Kubernetes Integration: RBD CSI Dynamic Volume Provisioning Practice

Recommended path: 05-Storage/01-Ceph/12-Ceph and Kubernetes Integration: RBD CSI Dynamic Volume Provisioning Practice.md

Tags: #Ceph #Kubernetes #RBD #CSI #StorageClass #PVC #PV #DynamicVolumeSupply #ReadWriteOnce #CloudRawStorage #SRE #AdvancedSre

---

## I. Document Overview

This is the twelfth article of the Ceph Advanced SRE Storage Module, focusing on the practical implementation of Ceph RBD integration with Kubernetes.

Previously completed:

- Ceph cluster deployment
- OSD management
- Pool and PG
- CRUSH rules
- RBD block storage practice
- CephFS file storage practice
- RGW object storage practice

From this article, we begin the integration between Ceph and Kubernetes:

    Kubernetes uses Ceph as an external persistent storage

This article focuses on:

    Using Ceph RBD CSI to provide dynamic block storage volumes for Kubernetes

Typical objectives:

    Pod requests PVC
      |
      v
    StorageClass calls RBD CSI
      |
      v
    Ceph automatically creates RBD Image
      |
      v
    Pod mounts as block device filesystem
      |
      v
    Application writes persistent data

The most common usage pattern for RBD in Kubernetes is:

    ReadWriteOnce

That is:

    A RBD volume is typically exclusively mounted by one or more Pods on a single node.
    Not suitable for multiple nodes to share read/write access to the same regular filesystem.

For multi-Pod multi-node shared directory scenarios, use:

    CephFS CSI

---

## II. Experiment Objectives

After completing this article, you should be able to:

1. Understand why Kubernetes needs CSI.
2. Understand the component structure of RBD CSI.
3. Understand the relationship between StorageClass, PVC, PV, Pod, and Ceph RBD Image.
4. Deploy Ceph RBD CSI in Kubernetes.
5. Create a dedicated Ceph RBD Pool.
6. Create a minimum-privilege user for Ceph CSI usage.
7. Create Kubernetes Secret.
8. Create RBD StorageClass.
9. Create PVC and dynamically generate PV.
10. Verify the automatically created RBD Image in Ceph.
11. Create Pod mounting PVC.
12. Write data and verify persistence.
13. After deleting Pod, verify data remains after remounting.
14. Expand PVC and verify filesystem expansion.
15. Troubleshoot issues like PVC Pending, Pod mount failure, RBD Image creation failure, Secret errors, and CSI Pod anomalies.
16. Understand security, permissions, image, monitoring, and recycling policies for production environment RBD CSI usage.

---

## III. Experimental Environment

### 3.1 Ceph Cluster

The Ceph cluster is independently deployed on the following nodes:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation (optional) |

Ceph cluster status requirements:

    MON quorum normal
    MGR normal
    OSD up/in
    PG active+clean
    ceph -s is HEALTH_OK or explainable HEALTH_WARN

---

### 3.2 Kubernetes Cluster

The Kubernetes cluster is independently deployed on the following nodes:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Runtime environment:

    kubeadm cluster
    containerd runtime
    Calico CNI
    Ubuntu Server 22.04.5 LTS

Notes:

    Ceph nodes and Kubernetes nodes remain independent.
    Kubernetes nodes only use RBD volumes as Ceph clients.
    Do not deploy Ceph OSD on Kubernetes nodes.
    Do not arbitrarily modify containerdBottom configuration for Ceph CSI.

---

### 3.3 Access Relationships

Kubernetes nodes need to access Ceph MON:

    10.0.0.31:3300
    10.0.0.31:6789
    10.0.0.32:3300
    10.0.0.32:6789
    10.0.0.33:3300
    10.0.0.33:6789

Kubernetes nodes also need to access Ceph OSD ports:

    6800-7300/tcp

All nodes in the experimental environment are located in:

    10.0.0.0/24

---

## IV. Why Kubernetes Needs CSI

### 4.1 Kubernetes Storage Object Relationships

Common storage objects in Kubernetes:

| Object | Purpose |
|---|---|
| StorageClass | Defines dynamic volume provisioning method |
| PVC | User requests storage |
| PV | Actual storage volume in the cluster |
| Pod volumeMount | Pod mounts storage |
| CSI Driver | Interface between Kubernetes and external storage systems |

Basic workflow:

    StorageClass
      |
      v
    PVC
      |
      v
    PV
      |
      v
    CSI Driver
      |
      v
    Ceph RBD Image
      |
      v
    Pod uses mounted storage

---

### 4.2 What is CSI

CSI, full name:

    Container Storage Interface

CSI is a standard interface between container orchestration systems and storage systems.

Kubernetes can interface with: /think

- Ceph RBD
- CephFS
- Longhorn
- NFS
- Cloud Disk
- SAN / NAS
- Commercial Storage

CSI is responsible for:

- Creating volumes
- Deleting volumes
- Mounting volumes
- Unmounting volumes
- Expanding volumes
- Snapshots, depending on driver capabilities
- Volume state management

---

### 4.3 The Role of RBD CSI

Ceph RBD CSI is responsible for converting Kubernetes PVCs into Ceph RBD Images.

Example:

    User creates PVC: mysql-data, size 10Gi
      |
      v
    RBD CSI creates an RBD Image in a Ceph Pool
      |
      v
    Pod is scheduled to a Worker node
      |
      v
    CSI Node plugin maps RBD on this node
      |
      v
    kubelet mounts the volume into the Pod
      |
      v
    Application writes data

---

## Five, Understanding RBD CSI Components

RBD CSI typically includes two types of components:

### 5.1 Controller Component

Controller components typically run as Deployments.

Main responsibilities:

- CreateVolume
- DeleteVolume
- ControllerExpandVolume
- Snapshot, depending on installation content
- Coordinate PV/PVC lifecycle with Kubernetes API

Common containers:

- csi-rbdplugin
- csi-provisioner
- csi-resizer
- csi-snapshotter, depending on deployment
- csi-attacher

---

### 5.2 Node Component

Node components typically run as DaemonSets on each Kubernetes node.

Main responsibilities:

- NodeStageVolume
- NodePublishVolume
- NodeUnstageVolume
- NodeUnpublishVolume
- Map/unmap RBD on this node
- Mount block device to Pod's required path

Simple understanding:

    Controller is responsible for creating volumes.
    Node is responsible for attaching volumes to specific nodes and Pods.

---

## Six, Applicable Scenarios for RBD in Kubernetes

### 6.1 Suitable Scenarios

RBD CSI is suitable for:

- MySQL data disk
- PostgreSQL data disk
- Redis persistent volume
- MongoDB data disk
- Elasticsearch data disk, requires performance evaluation
- Harbor data directory, some components
- GitLab data directory, requires architecture splitting
- Jenkins data directory
- Data disk for stateful applications with single Pod exclusivity

Typical access mode:

    ReadWriteOnce

---

### 6.2 Unsuitable Scenarios

RBD CSI is not suitable for:

- Multiple Pods on multiple nodes sharing read/write access to the same directory
- Web multi-replica shared uploads directory
- Multiple tasks sharing the same dataset
- Scenarios requiring ReadWriteMany

These scenarios are better suited for:

    CephFS CSI

---

## Seven, Pre-Operation Checks

### 7.1 Check Ceph Cluster

Run on Ceph management node:

    ceph -s
    ceph health detail
    ceph osd tree
    ceph pg stat
    ceph df

Requirements:

    OSD up/in
    PG active+clean
    Sufficient capacity
    No nearfull/full
    No unexplained HEALTH_ERR

---

### 7.2 Check Kubernetes Cluster

Run on k8s-master:

    kubectl get nodes -o wide

Expected:

    k8s-master    Ready
    k8s-worker01  Ready
    k8s-worker02  Ready

Check system Pods:

    kubectl get pods -A

Requirements:

    CoreDNS is normal
    CNI is normal
    kube-system has no abnormal Pods

---

### 7.3 Check Kubernetes Node to Ceph MON Network

Run on each Kubernetes node:

    ping -c 3 10.0.0.31
    ping -c 3 10.0.0.32
    ping -c 3 10.0.0.33

Check MON ports:

    nc -vz 10.0.0.31 3300
    nc -vz 10.0.0.31 6789
    nc -vz 10.0.0.32 3300
    nc -vz 10.0.0.32 6789
    nc -vz 10.0.0.33 3300
    nc -vz 10.0.0.33 6789

If no nc:

Ubuntu:

    apt install -y netcat-openbsd

---

### 7.4 Check Kubernetes Node Kernel Modules

RBD depends on Linux kernel rbd module.

Run on each Kubernetes node:

    modprobe rbd
    lsmod | grep rbd

Expected to see:

    rbd

If no output, check if the kernel module is available.

---

### 7.5 Check kubelet and containerd

Run on Kubernetes node:

    systemctl status kubelet
    systemctl status containerd

Requirements:

    kubelet is normal
    containerd is normal

Note:

    Do not arbitrarily modify containerd configuration for Ceph CSI.
    If image acceleration is needed, prioritize private registry, image synchronization, and imagePullPolicy management.

---

## Eight, Experiment Task List

| Experiment | Objective | Risk Level |
|---|---|---|
| Experiment 1 | Create a Dedicated Pool for Ceph RBD CSI | Medium |
| Experiment 2 | Create a Ceph CSI Minimal Privilege User | Medium |
| Experiment 3 | Prepare Kubernetes Namespace | Low |
| Experiment 4 | Deploy Ceph RBD CSI | Medium |
| Experiment 5 | Create Secret | Medium |
| Experiment 6 | Create StorageClass | Medium |
| Experiment 7 | Create PVC and Dynamically Generate PV | Medium |
| Experiment 8 | Create Pod Mounting PVC | Medium |
| Experiment 9 | Validate Data Persistence | Low |
| Experiment 10 | Validate Automatically Created RBD Image in Ceph | Low |
| Experiment 11 | Expand PVC | Medium |
| Experiment 12 | Clean Up Test Resources | High |
| Experiment 13 | Common Troubleshooting | Medium-High |

High-Risk Warning:

    Deleting PVC may cause the underlying RBD Image to be deleted, depending on reclaimPolicy.
    Deleting StorageClass will not delete existing PV, but will affect subsequent dynamic provisioning.
    Deleting Secret will cause new volume creation or mounting to fail.
    Production environments must clearly define recycling policies and data retention policies.

---

## IX. Experiment 1: Create RBD CSI Dedicated Pool

### 9.1 Create Pool

Execute on Ceph management node:

    ceph osd pool create k8s-rbd 64

Explanation:

    k8s-rbd is the Kubernetes RBD CSI dedicated pool.
    Small-scale experimental environment uses 64 PGs.
    Production environment should combine OSD count, pool count, and autoscaler to plan PGs.

---

### 9.2 Enable RBD application

    ceph osd pool application enable k8s-rbd rbd

---

### 9.3 Set Replication Factor

    ceph osd pool set k8s-rbd size 3
    ceph osd pool set k8s-rbd min_size 2

Check:

    ceph osd pool get k8s-rbd size
    ceph osd pool get k8s-rbd min_size

---

### 9.4 Initialize RBD Pool

    rbd pool init k8s-rbd

Check:

    ceph osd pool ls
    ceph df

---

## X. Experiment 2: Create Ceph CSI Minimal Privilege User

### 10.1 Create User

Execute on Ceph management node:

    ceph auth get-or-create client.k8s-rbd \
      mon 'profile rbd' \
      mgr 'profile rbd pool=k8s-rbd' \
      osd 'profile rbd pool=k8s-rbd' \
      -o /etc/ceph/ceph.client.k8s-rbd.keyring

Explanation:

    client.k8s-rbd is the Ceph user used by Kubernetes RBD CSI.
    This user only has RBD-related permissions for the k8s-rbd pool.
    Production environments should not directly use client.admin.

---

### 10.2 Check User Permissions

    ceph auth get client.k8s-rbd

Expected output similar to:

    [client.k8s-rbd]
        key = xxxxx
        caps mgr = "profile rbd pool=k8s-rbd"
        caps mon = "profile rbd"
        caps osd = "profile rbd pool=k8s-rbd"

---

### 10.3 Get User Key

    ceph auth get-key client.k8s-rbd

Save variable:

    K8S_RBD_KEY=$(ceph auth get-key client.k8s-rbd)

Check:

    echo $K8S_RBD_KEY

Security Reminder:

    Key is sensitive information.
    Do not commit to Git.
    Do not write to public documentation.
    Production environments should use Secret management and access control.

---

## XI. Obtain Ceph Cluster FSID and MON Address

### 11.1 Obtain FSID

Execute on Ceph management node:

    ceph fsid

Example:

    11111111-2222-3333-4444-555555555555

Record as:

    clusterID

RBD CSI's clusterID typically uses Ceph FSID.

---

### 11.2 Obtain MON Address

Check:

    ceph mon dump

Record MON addresses:

    10.0.0.31:6789
    10.0.0.32:6789
    10.0.0.33:6789

You can also use v2 port:

    10.0.0.31:3300
    10.0.0.32:3300
    10.0.0.33:3300

Experiment commonly uses 6789 for better compatibility.

---

## XII. Experiment 3: Prepare Kubernetes Namespace

### 12.1 Create Namespace

Execute on k8s-master:

    kubectl create namespace ceph-csi

Check:

    kubectl get ns ceph-csi

---

### 12.2 Create Test Business Namespace

    kubectl create namespace storage-demo

Check:

    kubectl get ns storage-demo

Explanation:

    ceph-csi is used for deploying CSI components.
    storage-demo is used for creating PVC and Pod testing.

---

## XIII. Experiment 4: Deploy Ceph RBD CSI

### 13.1 Deployment Method Overview

Common deployment methods for Ceph CSI:

1. Official YAML Manifest
2. Helm Chart
3. GitOps Management
4. Platform-Built-in Storage Plugin

This document explains core resources using YAML manifest approach.

Production recommendations:

    Fix Ceph CSI version.
    Fix image version.
    Synchronize images to internal Harbor or Alibaba Cloud repository.
    Do not use latest.
    Include YAML in Git management.

---

### 13.2 Image and Domestic Network Optimization

Ceph CSI may involve multiple images, for example:

- cephcsi image
- csi-provisioner
- csi-attacher
- csi-resizer
- csi-snapshotter (if snapshot feature is enabled)
- csi-node-driver-registrar

Domestic network environment recommendations:

    First pull fixed version images from official image repository.
    Then re-tag to internal Harbor or Alibaba Cloud image repository.
    Use internal image address uniformly in Kubernetes manifests.
    Do not modify Kubernetes nodeBottom containerd as primary solution.

Check image pull issues:

    kubectl describe pod -n ceph-csi <pod-name>

Common events:

    ImagePullBackOff
    ErrImagePull

---

### 13.3 Creating ceph-csi-config ConfigMap

Replace <cluster-id> with ceph fsid.

    cat > csi-config-map.yaml <<'EOF'
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: ceph-csi-config
      namespace: ceph-csi
    data:
      config.json: |-
        [
          {
            "clusterID": "<cluster-id>",
            "monitors": [
              "10.0.0.31:6789",
              "10.0.0.32:6789",
              "10.0.0.33:6789"
            ]
          }
        ]
    EOF

Apply:

    kubectl apply -f csi-config-map.yaml

Check:

    kubectl get configmap -n ceph-csi ceph-csi-config -o yaml

---

### 13.4 Creating ceph-config ConfigMap

    cat > ceph-config-map.yaml <<'EOF'
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: ceph-config
      namespace: ceph-csi
    data:
      ceph.conf: |
        [global]
        auth_cluster_required = cephx
        auth_service_required = cephx
        auth_client_required = cephx
    EOF

Apply:

    kubectl apply -f ceph-config-map.yaml

---

### 13.5 Deploying RBD CSI Components

In actual production, it is recommended to obtain corresponding version YAML from official ceph-csi release, and perform image address replacement.

Typical resources include:

    csi-rbdplugin-provisioner Deployment
    csi-rbdplugin DaemonSet
    ServiceAccount
    ClusterRole
    ClusterRoleBinding
    Role
    RoleBinding
    CSIDriver

Check deployed resources after deployment:

    kubectl get pods -n ceph-csi -o wide
    kubectl get deploy -n ceph-csi
    kubectl get ds -n ceph-csi
    kubectl get csidriver

Expected:

    csi-rbdplugin-provisioner is normally Running
    csi-rbdplugin DaemonSet is normally Running on each K8s node
    rbd.csi.ceph.com exists in CSIDriver

---

### 13.6 Post-deployment Checks

Check Pods:

    kubectl get pods -n ceph-csi -o wide

Check DaemonSet:

    kubectl get ds -n ceph-csi

Expected:

    DESIRED equals READY

Check CSI Driver:

    kubectl get csidriver

Expected to include:

    rbd.csi.ceph.com

If Pods are not normal:

    kubectl describe pod -n ceph-csi <pod-name>
    kubectl logs -n ceph-csi <pod-name> -c csi-rbdplugin

---

## FourteenI don't know.Experiment Five: Creating Kubernetes Secret

### 14.1 Creating Secret YAML

Replace <user-key> with client.k8s-rbd's key.

Note:

    userID does not include client. prefix.
    client.k8s-rbd corresponds to userID: k8s-rbd

    cat > csi-rbd-secret.yaml <<'EOF'
    apiVersion: v1
    kind: Secret
    metadata:
      name: csi-rbd-secret
      namespace: ceph-csi
    stringData:
      userID: k8s-rbd
      userKey: <user-key>
    EOF

Apply:

    kubectl apply -f csi-rbd-secret.yaml

Check:

    kubectl get secret -n ceph-csi csi-rbd-secret

---

### 14.2 Secret Naming Explanation

StorageClass will reference this Secret: /think

csi.storage.k8s.io/provisioner-secret-name
csi.storage.k8s.io/provisioner-secret-namespace
csi.storage.k8s.io/controller-expand-secret-name
csi.storage.k8s.io/controller-expand-secret-namespace
csi.storage.k8s.io/node-stage-secret-name
csi.storage.k8s.io/node-stage-secret-namespace

If the Secret name or namespace is incorrect, it will cause:

    PVC creation failure
    Pod mount failure
    CSI permission errors

---

## Fifteen, Experiment Six: Creating RBD StorageClass

### 15.1 StorageClass YAML

Replace <cluster-id> with ceph fsid.

    cat > storageclass-rbd.yaml <<'EOF'
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: ceph-rbd
    provisioner: rbd.csi.ceph.com
    parameters:
      clusterID: <cluster-id>
      pool: k8s-rbd
      imageFeatures: layering
      csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
      csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi
      csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
      csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi
      csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
      csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi
    reclaimPolicy: Delete
    allowVolumeExpansion: true
    volumeBindingMode: Immediate
    mountOptions:
      - discard
    EOF

Apply:

    kubectl apply -f storageclass-rbd.yaml

Check:

    kubectl get storageclass
    kubectl describe storageclass ceph-rbd

---

### 15.2 Parameter Explanation

| Parameter | Explanation |
|---|---|
| provisioner | CSI Driver name |
| clusterID | Ceph cluster FSID |
| pool | RBD Image pool |
| imageFeatures | RBD Image features |
| reclaimPolicy | PVC deletion PV recycling policy |
| allowVolumeExpansion | Whether volume expansion is allowed |
| volumeBindingMode | Volume binding mode |
| mountOptions | Mount parameters |

This document uses:

    imageFeatures: layering

Reason:

    layering has better compatibility.
    Complex features can be evaluated later based on kernel, CSI version, and production requirements.

---

### 15.3 reclaimPolicy Explanation

This experiment uses:

    reclaimPolicy: Delete

Meaning:

    After deleting PVC, dynamically created PV and underlying RBD Image may be deleted.

Use with caution in production.

If you want to retain PV and underlying data after deleting PVC, use:

    reclaimPolicy: Retain

Production recommendation:

    Database and critical business data should use Delete cautiously.
    Before production, clearly define data retention policy after PVC deletion.

---

## Sixteen, Experiment Seven: Create PVC and Dynamically Generate PV

### 16.1 Create PVC

    cat > pvc-rbd-demo.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: rbd-demo-pvc
      namespace: storage-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: ceph-rbd
      resources:
        requests:
          storage: 5Gi
    EOF

Apply:

    kubectl apply -f pvc-rbd-demo.yaml

---

### 16.2 Check PVC

    kubectl get pvc -n storage-demo

Expected:

    NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
    rbd-demo-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWO            ceph-rbd

---

### 16.3 Check PV

    kubectl get pv

Check details:

    kubectl describe pvc -n storage-demo rbd-demo-pvc
    kubectl describe pv <pv-name>

Focus on:

- StorageClass
- CSI Driver
- VolumeHandle
- Events
- Capacity
- AccessModes

---

## Seventeen, Experiment Eight: Verify Automatically Created RBD Image in Ceph

### 17.1 View RBD Image

Execute on Ceph management node:

    rbd ls -p k8s-rbd

Expected to see similar output:

    csi-vol-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

Check details:

    rbd info k8s-rbd/<image-name>

Notes:

    After Kubernetes PVC dynamic provisioning, RBD CSI will create corresponding RBD Image in the k8s-rbd Pool.

---

### 17.2 Check RBD Status

    rbd status k8s-rbd/<image-name>

If the PVC hasn't been mounted by a Pod yet, there may be no watcher.

After the Pod mounts, check again to potentially see watcher information.

---

## EighteenI don't know.Experiment Nine: Create Pod Mounting PVC

### 18.1 Create Test Pod

    cat > pod-rbd-demo.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: rbd-demo-pod
      namespace: storage-demo
    spec:
      containers:
        - name: app
          image: registry.cn-hangzhou.aliyuncs.com/pub-syq/busybox:latest
          command:
            - sh
            - -c
            - "while true; do sleep 3600; done"
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: rbd-demo-pvc
    EOF

Notes:

    If the busybox image isn't available, replace it with an existing busybox image from your Harbor or Alibaba Cloud registry.
    It's recommended to use a controlled image source in both production and experimental environments to avoid ImagePullBackOff.

Apply:

    kubectl apply -f pod-rbd-demo.yaml

Check:

    kubectl get pod -n storage-demo -o wide

Expected:

    rbd-demo-pod Running

---

### 18.2 Check Pod Events

    kubectl describe pod -n storage-demo rbd-demo-pod

Focus on:

- Whether scheduling was successful
- Whether PVC mounting was successful
- Whether FailedMount occurred
- Whether ImagePullBackOff occurred
- Whether MountVolume.MountDevice failed occurred

---

### 18.3 Write Data Inside Pod

    kubectl exec -n storage-demo rbd-demo-pod -- sh -c "echo 'hello rbd csi' > /data/hello.txt"

Read:

    kubectl exec -n storage-demo rbd-demo-pod -- cat /data/hello.txt

Expected:

    hello rbd csi

Check files:

    kubectl exec -n storage-demo rbd-demo-pod -- ls -l /data

---

## NineteenI don't know.Experiment Ten: Verify Data Persistence

### 19.1 Delete Pod

    kubectl delete pod -n storage-demo rbd-demo-pod

Confirm Pod deletion:

    kubectl get pod -n storage-demo

---

### 19.2 Recreate Pod

    kubectl apply -f pod-rbd-demo.yaml

Wait for Running status:

    kubectl get pod -n storage-demo -o wide

---

### 19.3 Read Previously Written Data

    kubectl exec -n storage-demo rbd-demo-pod -- cat /data/hello.txt

Expected:

    hello rbd csi

Notes:

    After the Pod is deleted, the PVC and underlying RBD Image still exist.
    When a new Pod mounts the same PVC, the data remains preserved.

---

## TwentyI don't know.Experiment Eleven: PVC Expansion

### 20.1 Prerequisites

StorageClass must enable:

    allowVolumeExpansion: true

Check:

    kubectl get storageclass ceph-rbd -o yaml | grep allowVolumeExpansion

---

### 20.2 Expand PVC

Expand the PVC from 5Gi to 10Gi:

    kubectl patch pvc rbd-demo-pvc -n storage-demo \
      -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

Check:

    kubectl get pvc -n storage-demo

Wait until capacity becomes:

    10Gi

---

### 20.3 Check File System Capacity Inside Pod

    kubectl exec -n storage-demo rbd-demo-pod -- df -h /data

Expected:

    /data capacity gradually becomes 10Gi

If no immediate change, check PVC:

    kubectl describe pvc -n storage-demo rbd-demo-pvc

Check CSI Pod logs:

    kubectl get pods -n ceph-csi
    kubectl logs -n ceph-csi <rbd-provisioner-pod> -c csi-rbdplugin

---

### 20.4 Verify RBD Image Size in Ceph

Execute on Ceph management node:

    rbd ls -p k8s-rbd

Check corresponding Image:

    rbd info k8s-rbd/<image-name>

Expected:

    size 10 GiB

## 21. Experiment 12: Clean Up Test Resources

### 21.1 Delete Pod

    kubectl delete pod -n storage-demo rbd-demo-pod

---

### 21.2 Delete PVC

> High-risk warning:
>
> If StorageClass reclaimPolicy is Delete, deleting PVC may result in the underlying RBD Image being deleted.

Delete:

    kubectl delete pvc -n storage-demo rbd-demo-pvc

Check:

    kubectl get pvc -n storage-demo
    kubectl get pv

---

### 21.3 Verify RBD Image Deletion

Execute on Ceph management node:

    rbd ls -p k8s-rbd

If reclaimPolicy is Delete, the related csi-vol Image should be deleted.

If it still exists, check PV status and CSI logs.

---

### 21.4 Delete StorageClass

    kubectl delete storageclass ceph-rbd

---

### 21.5 Delete Secret

    kubectl delete secret -n ceph-csi csi-rbd-secret

---

### 21.6 Delete Test Namespace

    kubectl delete namespace storage-demo

If no longer testing Ceph CSI, you may also delete ceph-csi Namespace, but this will remove CSI components:

    kubectl delete namespace ceph-csi

Do not arbitrarily delete ceph-csi Namespace in production environments.

---

### 21.7 Whether to Delete Ceph Pool

If confirmed no longer needed in test environment:

    ceph osd pool ls
    rbd ls -p k8s-rbd

Enable deletion:

    ceph config set mon mon_allow_pool_delete true

Delete Pool:

    ceph osd pool rm k8s-rbd k8s-rbd --yes-i-really-really-mean-it

Disable deletion:

    ceph config set mon mon_allow_pool_delete false

Production warning:

    Deleting Pool will delete all RBD Images within.
    Do not execute arbitrarily in production environments.

---

## 22. Common Issues and Troubleshooting

### 22.1 CSI Pod Image Pull Failure

Symptoms:

    ImagePullBackOff
    ErrImagePull

Check:

    kubectl get pods -n ceph-csi
    kubectl describe pod -n ceph-csi <pod-name>

Resolution:

- Check image address
- Check domestic network
- Check imagePullSecrets
- Synchronize image to Harbor or Alibaba Cloud repository
- Modify image address in YAML
- Not recommended to directly modify containerd global configuration

---

### 22.2 CSI Pod Not Running

Check:

    kubectl get pods -n ceph-csi -o wide
    kubectl describe pod -n ceph-csi <pod-name>
    kubectl logs -n ceph-csi <pod-name> -c csi-rbdplugin

Common causes:

- Missing RBAC
- ConfigMap error
- Secret error
- Image pull failure
- Node taints not tolerated
- Kubelet plugin directory permission anomaly

---

### 22.3 PVC Stuck in Pending

Check:

    kubectl describe pvc -n storage-demo rbd-demo-pvc

Focus on Events.

Common causes:

- StorageClass does not exist
- Provisioner name error
- CSI Controller not working
- Secret name or namespace error
- ClusterID error
- Pool does not exist
- Ceph user permissions insufficient
- Ceph MON unreachable

Troubleshoot:

    kubectl get storageclass
    kubectl get pods -n ceph-csi
    kubectl logs -n ceph-csi <provisioner-pod>
    ceph -s
    ceph auth get client.k8s-rbd
    ceph osd pool ls

---

### 22.4 Pod Mount Failed (FailedMount)

Check:

    kubectl describe pod -n storage-demo rbd-demo-pod

Common events:

    MountVolume.MountDevice failed
    rbd image not found
    permission denied
    context deadline exceeded
    failed to get secret
    failed to connect to monitors

Troubleshoot:

    kubectl get secret -n ceph-csi csi-rbd-secret
    kubectl get configmap -n ceph-csi ceph-csi-config -o yaml
    kubectl logs -n ceph-csi <node-plugin-pod> -c csi-rbdplugin
    ceph -s
    rbd ls -p k8s-rbd

Check on the node where Pod resides:

    modprobe rbd
    lsmod | grep rbd
    dmesg | tail -50

---

### 22.5 Ceph Permission Error

Symptoms:

    permission denied
    operation not permitted

Check Ceph user:

    ceph auth get client.k8s-rbd

Confirm caps:

    mon 'profile rbd'
    mgr 'profile rbd pool=k8s-rbd'
    osd 'profile rbd pool=k8s-rbd'

If Pool name is incorrect, permissions will fail.

---

### 22.6 ClusterID Error

Symptoms: /think

# CSI Cannot Find Corresponding Cluster Configuration

## Check:

- `ceph fsid`
- `kubectl get configmap -n ceph-csi ceph-csi-config -o yaml`
- `kubectl get storageclass ceph-rbd -o yaml`

## Requirements:

The `clusterID` in the ConfigMap, the `clusterID` in the StorageClass, and the Ceph fsid must all be consistent.

---

### 22.7 RBD Image Created But Pod Cannot Mount

## Troubleshoot:

- `rbd ls -p k8s-rbd`
- `rbd info k8s-rbd/<image-name>`
- `rbd status k8s-rbd/<image-name>`

Check Kubernetes:

- `kubectl describe pod -n storage-demo rbd-demo-pod`
- `kubectl logs -n ceph-csi <node-plugin-pod> -c csi-rbdplugin`

## Common Causes:

- Node plugin abnormality
- RBD kernel module issue
- Secret error
- Network connectivity issue between node and Ceph
- RBD feature incompatibility
- VolumeAttachment abnormality

Check VolumeAttachment:

- `kubectl get volumeattachment`
- `kubectl describe volumeattachment <name>`

---

### 22.8 PVC Deleted But RBD Image Not Deleted

## Possible Causes:

- `reclaimPolicy` is set to Retain
- CSI Controller abnormality
- Finalizer not cleaned
- Ceph deletion permission insufficient
- PV still exists

## Check:

- `kubectl get pv`
- `kubectl describe pv <pv-name>`
- `kubectl get pvc -A`
- `rbd ls -p k8s-rbd`

## Handling Approach:

- First confirm whether data needs to be retained.
- Do not directly delete RBD Image.
- Handle after confirming PV/PVC status and reclaimPolicy.

---

### 22.9 PVC Expansion Failed

## Check:

- `kubectl describe pvc -n storage-demo rbd-demo-pvc`
- `kubectl get storageclass ceph-rbd -o yaml`
- `kubectl logs -n ceph-csi <provisioner-pod>`
- `kubectl logs -n ceph-csi <node-plugin-pod> -c csi-rbdplugin`

## Common Causes:

- StorageClass not enabled with `allowVolumeExpansion`
- `csi-resizer` not deployed
- Ceph user permission insufficient
- RBD Image expansion succeeded but filesystem expansion failed
- Pod not remounted or node expansion failed

---

## Twenty-Three, RBD CSI Common Commands Summary

### 23.1 Ceph Side

- `ceph -s`
- `ceph osd pool create k8s-rbd 64`
- `ceph osd pool application enable k8s-rbd rbd`
- `ceph osd pool set k8s-rbd size 3`
- `ceph osd pool set k8s-rbd min_size 2`
- `rbd pool init k8s-rbd`
- `rbd ls -p k8s-rbd`
- `rbd info k8s-rbd/<image-name>`
- `rbd status k8s-rbd/<image-name>`
- `ceph auth get client.k8s-rbd`

---

### 23.2 Kubernetes Side

- `kubectl get nodes -o wide`
- `kubectl get pods -n ceph-csi -o wide`
- `kubectl get storageclass`
- `kubectl get pvc -n storage-demo`
- `kubectl get pv`
- `kubectl describe pvc -n storage-demo rbd-demo-pvc`
- `kubectl describe pod -n storage-demo rbd-demo-pod`
- `kubectl get volumeattachment`
- `kubectl get csidriver`

---

### 23.3 Log Troubleshooting

- `kubectl logs -n ceph-csi <provisioner-pod>`
- `kubectl logs -n ceph-csi <node-plugin-pod> -c csi-rbdplugin`
- `kubectl describe pod -n ceph-csi <pod-name>`
- `kubectl describe pvc -n storage-demo <pvc-name>`
- `kubectl describe pod -n storage-demo <pod-name>`

---

## Twenty-Four, Production Environment Notes

### 24.1 Do Not Use client.admin

Admin key can be used for verification in experiments.

In production, use a user with minimal permissions:

- `client.k8s-rbd`

And restrict to a specific Pool:

- `osd 'profile rbd pool=k8s-rbd'`

---

### 24.2 Clearly Define reclaimPolicy

In production, clearly specify:

- Delete or Retain

Delete:

- PVC deletion will remove the underlying RBD Image.
- Suitable for temporary environments or discardable data.

Retain:

- PVC deletion will retain PV and underlying data.
- Suitable for critical business, but requires manual recovery process.

PVCs for databases in production should typically use Delete cautiously.

---

### 24.3 Image Must Use Fixed Version

In production, do not use `latest`.

Recommendations:

- Fix Ceph CSI version.
- Fix sidecar version.
- Sync image to internal repository.
- YAML in Git management.
- Validate upgrades in test cluster before deployment.

---

### 24.4 Do Not Break Kubernetes Underlying Runtime

To pull CSI images, avoid arbitrarily modifying containerd global configuration.

Recommended: `/think`

- Internal Harbor
- Alibaba Cloud Image Repository
- imagePullSecrets
- Unified Image Synchronization Process
- Fixed Image Tag

---

### 24.5 RBD is suitable for RWO, not RWX

RBD is inherently suitable for:

    ReadWriteOnce

If multiple Pods across multiple nodes need to share a directory, use:

    CephFS CSI

Do not force multiple nodes to perform regular file system read/write operations on the same RBD volume.

---

### 24.6 Monitoring must cover CSI

In production, monitor:

- ceph-csi Pod status
- PVC Pending count
- PV abnormal status
- VolumeAttachment abnormalities
- RBD Image count
- RBD Pool capacity
- Ceph OSD status
- Ceph PG status
- CSI error logs
- kubelet FailedMount events

---

### 24.7 Storage changes require a process

The following operations are considered high-risk in production:

- Delete PVC
- Delete PV
- Modify StorageClass
- Modify Secret
- Modify Ceph user permissions
- Delete RBD Image
- Delete k8s-rbd Pool
- Upgrade Ceph CSI
- Scale up large PVCs

Must have:

    Backup
    Change window
    Rollback plan
    Monitoring observation
    Business confirmation

---

## Twenty-five, Advanced SRE Methodology

### 25.1 RBD CSI Troubleshooting requires four layers

First layer: Kubernetes objects

    PVC
    PV
    Pod
    StorageClass
    Secret
    VolumeAttachment
    Events

Second layer: CSI components

    csi-rbdplugin-provisioner
    csi-rbdplugin DaemonSet
    csi-provisioner
    csi-attacher
    csi-resizer
    node-driver-registrar

Third layer: Kubernetes nodes

    kubelet
    containerd
    rbd kernel module
    Node to Ceph network
    dmesg
    mount status

Fourth layer: Ceph backend

    Ceph health
    k8s-rbd Pool
    RBD Image
    CephX permissions
    OSD / PG status

---

### 25.2 PVC Pending is not necessarily a Kubernetes issue

PVC Pending may originate from:

- StorageClass error
- CSI Controller abnormality
- Secret error
- Ceph permission error
- Pool does not exist
- MON unreachable
- clusterID error

Troubleshooting cannot focus solely on `kubectl get pvc`.

Must check:

    kubectl describe pvc
    CSI Controller logs
    Ceph cluster status
    Ceph auth permissions
    RBD Pool status

---

### 25.3 Pod FailedMount is not necessarily a Ceph issue

Pod FailedMount may originate from:

- kubelet abnormality
- Node CSI plugin abnormality
- rbd module missing
- Node to Ceph network abnormality
- Secret error
- Image feature incompatibility
- VolumeAttachment residue

Thus, analysis must combine node and CSI logs.

---

## Twenty-six, Interview Answer Framework

If asked in an interview:

    How does Kubernetes use Ceph RBD as persistent storage?

Respond as follows:

    Kubernetes typically uses Ceph CSI to leverage Ceph RBD. First, create a dedicated RBD Pool in Ceph, such as k8s-rbd, and enable the rbd application, set replica count and min_size. Then create a minimal-privilege CephX user, such as client.k8s-rbd, granting only RBD permissions for that Pool.
    Deploy Ceph RBD CSI in Kubernetes, including Controller components and Node DaemonSet. The Controller handles dynamic creation/deletion of RBD Images, while the Node plugin maps RBD and mounts it to Pods.
    Create a Secret to store the Ceph user's userID and userKey, then create a StorageClass with clusterID, monitors, pool, imageFeatures, Secret name, and reclaimPolicy. After business creates a PVC, CSI automatically creates an RBD Image in the Ceph Pool and generates a PV. When a Pod mounts the PVC, the Node plugin completes RBD mapping and filesystem mounting on the node.
    Troubleshooting involves multi-layer checks: PVC, PV, StorageClass, Secret, CSI Pods, VolumeAttachment, kubelet, node rbd module, Ceph auth, RBD Pool, and Ceph health.
    In production, note minimal permissions, reclaimPolicy, fixed image versions, internal image repositories, CSI monitoring, PVC deletion protection, and Ceph backend capacity thresholds.

---

## Twenty-seven, Summary of This Chapter

This article primarily organizes practices for integrating Ceph RBD CSI with Kubernetes:

1. RBD CSI is responsible for dynamically converting Kubernetes PVCs into Ceph RBD Images.
2. RBD CSI is typically used in ReadWriteOnce scenarios.
3. Ceph clusters and Kubernetes clusters should maintain a clear boundary.
4. Kubernetes nodes act as Ceph clients accessing RBD, and do not directly handle Ceph OSDs.
5. It must be confirmed before deployment that K8s nodes can access Ceph MON and OSD.
6. A dedicated Pool: k8s-rbd must be created.
7. A user with minimal permissions: client.k8s-rbd must be created.
8. RBD CSI Controller and Node plugin must be deployed.
9. ConfigMap, Secret, and StorageClass must be created.
10. After creating PVC, RBD CSI automatically creates PV and underlying RBD Image.
11. After Pod mounts PVC, it can write persistent data.
12. Deleting Pod does not delete PVC data.
13. PVC expansion requires StorageClass to enable allowVolumeExpansion, and depends on CSI resizer.
14. In production environments, the reclaimPolicy must be explicitly set to Delete or Retain.
15. For advanced SRE troubleshooting RBD CSI issues, analysis must be conducted from four layers: Kubernetes, CSI, nodes, and Ceph backend.

---

## 28. Reference Documents

Ceph CSI Project:

    https://github.com/ceph/ceph-csi

Ceph CSI RBD Documentation:

    https://github.com/ceph/ceph-csi/tree/devel/docs

Ceph RBD Official Documentation:

    https://docs.ceph.com/en/latest/rbd/

Ceph RBD Kubernetes Documentation:

    https://docs.ceph.com/en/latest/rbd/rbd-kubernetes/

Kubernetes CSI Documentation:

    https://kubernetes.io/docs/concepts/storage/volumes/#csi

Kubernetes StorageClass Documentation:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes Persistent Volumes Documentation:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/