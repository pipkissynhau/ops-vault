# Ceph and Kubernetes Integration: CephFS CSI File Shared Storage Practice

Suggested path: 05-Storage/01-Ceph/13-Ceph and Kubernetes Integration: CephFS CSI File Shared Storage Practice.md

Tags: #Ceph #Kubernetes #CephFS #CSI #StorageClass #PVC #PV #ReadWriteMany #SharedStorage #MDS #CloudRawStorage #SRE #AdvancedSre

---

## I. Document Explanation

This document is the thirteenth article of the Ceph advanced SRE storage module, focusing on the practice of integrating CephFS with Kubernetes.

Previously completed:

- Ceph cluster deployment
- OSD management
- Pool and PG
- CRUSH rules
- RBD block storage practice
- CephFS file storage practice
- RGW object storage practice
- Kubernetes integration with RBD CSI dynamic volume provisioning practice

This document continues to the integration part between Ceph and Kubernetes:

    Kubernetes uses CephFS as shared file storage

This document focuses on practicing:

    Using CephFS CSI to provide ReadWriteMany shared file storage for Kubernetes

Typical objectives:

    Multiple Pods
      |
      v
    Mounting the same PVC simultaneously
      |
      v
    CephFS CSI creates a CephFS Subvolume
      |
      v
    Pods read/write in shared directory mode
      |
      v
    Data persistence saved to CephFS

The most typical access mode for CephFS in Kubernetes is:

    ReadWriteMany

That is:

    Multiple Pods can mount the same PVC simultaneously.
    Pods on multiple nodes can share the same file directory.
    Suitable for scenarios like shared upload directories, shared configurations, and shared datasets.

Difference from RBD CSI:

    RBD CSI is suitable for ReadWriteOnce, like cloud disks.
    CephFS CSI is suitable for ReadWriteMany, like shared file systems.

---

## II. Experiment Objectives

After completing this document, you should be able to:

1. Understand the meaning of ReadWriteMany in Kubernetes.
2. Understand the differences between CephFS CSI and RBD CSI.
3. Understand the role of CephFS CSI's Controller and Node plugins.
4. Create or reuse a CephFS filesystem.
5. Create a SubvolumeGroup for CephFS CSI usage.
6. Create a CephFS CSI minimal privilege user.
7. Deploy CephFS CSI in Kubernetes.
8. Create ConfigMap required for CephFS CSI.
9. Create Secret for CephFS CSI.
10. Create CephFS StorageClass.
11. Create ReadWriteMany PVC.
12. Create multiple Pods mounting the same PVC.
13. Verify shared file read/write between multiple Pods.
14. Verify data persistence after Pod deletion/recreation.
15. Verify PVC expansion.
16. View CSI-created subvolume in Ceph.
17. Troubleshoot issues like PVC Pending, Pod FailedMount, MDS anomalies, permission errors, and CSI Pod anomalies.
18. Understand CephFS CSI's permissions, security, images, MDS high availability, and recycling policies in production environments.

---

## III. Experiment Environment

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
    MDS active/standby normal
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
    Kubernetes nodes only act as CephFS client mounting filesystem.
    Do not deploy Ceph OSD on Kubernetes nodes.
    Do not arbitrarily destroy containerdBottom configuration for CephFS CSI.

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

CephFS also requires:

    MDS service normal
    CephFS filesystem created
    CephFS metadata/data pool normal

All nodes in the experiment environment are located in:

    10.0.0.0/24

---

## IV. Differences between CephFS CSI and RBD CSI

### 4.1 RBD CSI

RBD CSI provides block storage capabilities.

Typical access mode:

    ReadWriteOnce

Suitable for:

- MySQL
- PostgreSQL
- Redis
- MongoDB
- Single Pod exclusive data disk
- Scenarios similar to cloud disks

Features:

A PVC is typically mounted to a Pod on a single node.
Not suitable for general file systems that need to be shared for simultaneous writing across multiple nodes.

---

### 4.2 CephFS CSI

CephFS CSI provides file sharing capabilities.

Typical access mode:

    ReadWriteMany

Suitable for:

- Multiple Pods sharing an upload directory
- Web multi-replica shared files
- Multiple tasks sharing a dataset
- Shared configuration directory
- AI / data processing shared directory
- Multi-node shared result output directory

Features:

    Multiple Pods can mount the same PVC simultaneously.
    Pods on multiple nodes can share the same directory.
    Depends on CephFS and MDS at the underlying layer.

---

### 4.3 Selection Mnemonic

    Use like a cloud disk: RBD CSI
    Use like a shared directory: CephFS CSI
    Use like an object storage: RGW / MinIO

---

## FiveI don't know.CephFS CSI Component Understanding

CephFS CSI typically includes two types of components.

### 5.1 Controller Component

The Controller usually runs as a Deployment.

Main responsibilities:

- Create CephFS Subvolume
- Delete CephFS Subvolume
- Expand Volume
- Handle PVC / PV lifecycle
- Coordinate with Kubernetes API for dynamic volume provisioning

Common containers:

- csi-cephfsplugin
- csi-provisioner
- csi-resizer
- csi-snapshotter, if snapshot feature is enabled
- csi-attacher, may not be needed in some scenarios

---

### 5.2 Node Component

Node plugins usually run as DaemonSet on each Kubernetes node.

Main responsibilities:

- Mount CephFS on the node
- Mount CephFS subdirectory to Pod
- Unmount CephFS path used by Pod
- Handle node-side mounting anomalies

Simple understanding:

    Controller is responsible for creating shared volumes.
    Node is responsible for mounting shared volumes to specific nodes and Pods.

---

## SixI don't know.CephFS CSI and Subvolume

### 6.1 What is Subvolume

When CephFS CSI dynamically creates PVCs, it typically creates a Subvolume in CephFS.

It can be understood as:

    Each PVC corresponds to an independent subvolume directory in CephFS.

Illustration:

    CephFS: cephfs
      |
      v
    SubvolumeGroup: csi
      |
      ├── csi-vol-xxxxxx
      ├── csi-vol-yyyyyy
      └── csi-vol-zzzzzz

Relationship between Kubernetes PVC and CephFS Subvolume:

    PVC
      |
      v
    PV
      |
      v
    CephFS CSI
      |
      v
    CephFS Subvolume

---

### 6.2 Why Use SubvolumeGroup

SubvolumeGroup is used to centrally manage CSI dynamically created subvolumes.

Benefits:

- Easy to distinguish Kubernetes-created volumes
- Easy to view and clean up
- Easy for future quota, isolation, and management
- Easy to distinguish from manually created CephFS directories

This document uses:

    SubvolumeGroup: csi

---

## SevenI don't know.Pre-Operation Checks

### 7.1 Check Ceph Cluster Status

Execute on Ceph management node:

    ceph -s
    ceph health detail
    ceph osd tree
    ceph pg stat
    ceph df

Requirements:

    OSD up/in
    PG active+clean
    Sufficient capacity
    No nearfull / full
    No unexplained HEALTH_ERR

---

### 7.2 Check CephFS Status

Check if CephFS exists:

    ceph fs ls

Check CephFS status:

    ceph fs status

Check MDS:

    ceph mds stat

Expected:

    cephfs exists
    At least 1 active MDS
    At least 1 standby MDS, recommended for production
    metadata/data pool normal

---

### 7.3 Check Kubernetes Cluster

Execute on k8s-master:

    kubectl get nodes -o wide
    kubectl get pods -A

Requirements:

    All nodes Ready
    CoreDNS normal
    CNI normal
    kube-system has no large number of abnormal Pods

---

### 7.4 Check Kubernetes Node to Ceph MON Network

Execute on each Kubernetes node:

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

If nc is not available:

    apt install -y netcat-openbsd

---

### 7.5 Check Kubernetes Node Kernel Support for CephFS

Execute on each Kubernetes node:

    modprobe ceph
    lsmod | grep ceph

If using ceph-fuse mounter, depends on user-space fuse tools.

In production, kernel mounter is usually preferred, but need to evaluate kernel version and compatibility.

---

## EightI don't know.Experimental Task List

| Experiment | Objective | Risk Level |
|---|---|---|
| Experiment 1 | Create or Confirm CephFS File System | Medium |
| Experiment 2 | Create CephFS SubvolumeGroup | Medium |
| Experiment 3 | Create CephFS CSI Minimum Privilege User | Medium |
| Experiment 4 | Prepare Kubernetes Namespace | Low |
| Experiment 5 | Deploy CephFS CSI | Medium |
| Experiment 6 | Create Secret | Medium |
| Experiment 7 | Create CephFS StorageClass | Medium |
| Experiment 8 | Create RWX PVC | Medium |
| Experiment 9 | Create Multiple Pods with PVC Mount | Medium |
| Experiment 10 | Validate Multi-Pod Shared Read/Write | Low |
| Experiment 11 | Validate Data Persistence After Pod Deletion/Recreation | Low |
| Experiment 12 | Validate CephFS Subvolume | Low |
| Experiment 13 | PVC Expansion | Medium |
| Experiment 14 | Clean Up Test Resources | High |
| Experiment 15 | Common Troubleshooting | Medium-High |

High-Risk Warning:

    Deleting PVC may delete underlying CephFS Subvolume depending on reclaimPolicy.
    Deleting CephFS Pool will delete file system data.
    Deleting CephFS File System will affect all business using this file system.
    Production environment must have clear recycling policy and data retention policy.

---

## 9. Experiment 1: Create or Confirm CephFS File System

If you have completed the 10th CephFS practice and already have a cephfs file system, you can skip the creation step and only perform verification.

### 9.1 View CephFS

    ceph fs ls
    ceph fs status
    ceph mds stat

If it exists:

    cephfs

And MDS active is normal, you can proceed to the next section.

---

### 9.2 Create metadata pool

If you don't have CephFS yet, first create metadata pool:

    ceph osd pool create cephfs_metadata 32

Enable application:

    ceph osd pool application enable cephfs_metadata cephfs

Set replica count:

    ceph osd pool set cephfs_metadata size 3
    ceph osd pool set cephfs_metadata min_size 2

---

### 9.3 Create data pool

    ceph osd pool create cephfs_data 64

Enable application:

    ceph osd pool application enable cephfs_data cephfs

Set replica count:

    ceph osd pool set cephfs_data size 3
    ceph osd pool set cephfs_data min_size 2

---

### 9.4 Create CephFS

    ceph fs new cephfs cephfs_metadata cephfs_data

View:

    ceph fs ls
    ceph fs get cephfs

---

### 9.5 Deploy MDS

Deploy MDS:

    ceph orch apply mds cephfs --placement="2 ceph-node01 ceph-node02 ceph-node03"

View:

    ceph orch ps --daemon_type mds
    ceph fs status
    ceph mds stat

Expected:

    1 active MDS
    At least 1 standby MDS

---

## 10. Experiment 2: Create CephFS SubvolumeGroup

### 10.1 Create SubvolumeGroup

Execute on Ceph management node:

    ceph fs subvolumegroup create cephfs csi

View:

    ceph fs subvolumegroup ls cephfs

Expected:

    csi

---

### 10.2 SubvolumeGroup Explanation

The StorageClass will specify:

    subvolumeGroup: csi

This ensures subsequent Kubernetes PVC automatically created CephFS subvolume will be grouped under csi.

---

## 11. Experiment 3: Create CephFS CSI Minimum Privilege User

### 11.1 Create User

Execute on Ceph management node:

    ceph auth get-or-create client.k8s-cephfs \
      mon 'allow r' \
      mgr 'allow rw' \
      mds 'allow rw fsname=cephfs' \
      osd 'allow rw tag cephfs data=cephfs' \
      -o /etc/ceph/ceph.client.k8s-cephfs.keyring

Explanation:

    client.k8s-cephfs is the Ceph user used by Kubernetes CephFS CSI.
    mon allow r is for reading cluster information.
    mgr allow rw is for CSI calling CephFS subvolume management capabilities.
    mds allow rw fsname=cephfs is for accessing specified CephFS.
    osd allow rw tag cephfs data=cephfs is for accessing CephFS data.

---

### 11.2 View User Permissions

    ceph auth get client.k8s-cephfs

Expected similar to:

    [client.k8s-cephfs]
        key = xxxxx
        caps mon = "allow r"
        caps mgr = "allow rw"
        caps mds = "allow rw fsname=cephfs"
        caps osd = "allow rw tag cephfs data=cephfs"

---

### 11.3 Permission Compatibility Note

Different Ceph versions and Ceph CSI versions may have slight differences in permission requirements for CephFS subvolume management.

If permission issues occur during dynamic PVC creation, you can temporarily verify in test environment: /think

STRICT RULES:
1. Preserve ALL Markdown syntax exactly: headers (#), lists (-, *, 1.), bold/italic (**/*), tables, blockquotes (>), horizontal rules (---), task checkboxes (- [ ] / - [x]).
2. Do NOT translate or alter content inside placeholder tokens like §§code_0§§, https://github.com/ceph/ceph-csi, §§inline_0§§, §§wlink_0§§, #Ceph — leave them completely unchanged, exactly as written, character-for-character, in the same position in the sentence. These are protected tokens standing in for code/links/tags, not real words.
3. Do NOT add commentary, notes, or explanations. Output ONLY the translated Markdown.
4. Keep technical terms (command names, config keys, product names, API names) in English/original form — do not force-translate proper nouns or code identifiers.
5. Keep the same line breaks and paragraph structure as the input.
6. Translate table cell contents but keep pipe (|) and dash (-) table syntax intact.
7. Obsidian callouts like > [!note] or > [!warning] must keep their exact [!type] syntax; only translate the callout title text after it and the body.

Output only the translated Markdown, nothing else.

ceph auth caps client.k8s-cephfs \
  mon 'allow r' \
  mgr 'allow rw' \
  mds 'allow rw' \
  osd 'allow rw tag cephfs *=*'

After verification success, gradually converge permissions according to production security requirements.

Production principles:

  Do not use client.admin.
  Do not grant excessive permissions to business.
  First satisfy functionality, then gradually converge to minimal permissions.

---

### 11.4 Get User Key

  ceph auth get-key client.k8s-cephfs

Save variable:

  K8S_CEPHFS_KEY=$(ceph auth get-key client.k8s-cephfs)

View:

  echo $K8S_CEPHFS_KEY

Security reminder:

  Key belongs to sensitive information.
  Do not commit to Git.
  Do not write to public documentation.
  Production environment should use Secret management and permission control.

---

## TwelveI don't know.Get Ceph Cluster FSID And MON Address

### 12.1 Get FSID

Execute on Ceph management node:

  ceph fsid

Example:

  11111111-2222-3333-4444-555555555555

Record as:

  clusterID

CephFS CSI's clusterID typically uses Ceph FSID.

---

### 12.2 Get MON Address

View:

  ceph mon dump

Record MON address:

  10.0.0.31:6789
  10.0.0.32:6789
  10.0.0.33:6789

You can also use v2 port:

  10.0.0.31:3300
  10.0.0.32:3300
  10.0.0.33:3300

In experiments, using 6789 is more intuitive.

---

## ThirteenI don't know.Experiment Four: Prepare Kubernetes Namespace

### 13.1 Create ceph-csi Namespace

If the 12th chapter has already created ceph-csi, skip.

  kubectl create namespace ceph-csi

View:

  kubectl get ns ceph-csi

---

### 13.2 Create Test Business Namespace

  kubectl create namespace cephfs-demo

View:

  kubectl get ns cephfs-demo

Note:

  ceph-csi is used for deploying CSI components and Secret.
  cephfs-demo is used for creating PVC and Pod testing.

---

## FourteenI don't know.Experiment Five: Deploy CephFS CSI

### 14.1 Deployment Method Explanation

Common deployment methods for CephFS CSI:

1. Official YAML manifest
2. Helm Chart
3. GitOps management
4. Platform built-in storage plugin

This article explains core resources with YAML manifest approach.

In production, recommend:

  Fix Ceph CSI version.
  Fix image version.
  Synchronize images to internal Harbor or Alibaba Cloud repository.
  Do not use latest.
  Include YAML in Git management.

---

### 14.2 Image And Domestic Network Optimization

CephFS CSI may involve multiple images, such as:

- cephcsi image
- csi-provisioner
- csi-resizer
- csi-snapshotter, depending on whether snapshot is enabled
- csi-node-driver-registrar

Domestic network environment recommendations:

  First pull fixed version images from official image source.
  Then re-tag to internal Harbor or Alibaba Cloud image repository.
  Use internal image address in Kubernetes manifest.
  Do not modify Kubernetes node's underlying containerd as the main solution.

Check image pull issues:

  kubectl describe pod -n ceph-csi <pod-name>

Common events:

  ImagePullBackOff
  ErrImagePull

---

### 14.3 Create ceph-csi-config ConfigMap

If the 12th chapter's RBD CSI has already created ceph-csi-config and clusterID and monitors are the same, can reuse.

If not, create.

Replace <cluster-id> with ceph fsid:

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

View:

  kubectl get configmap -n ceph-csi ceph-csi-config -o yaml

---

### 14.4 Create ceph-config ConfigMap

If the 12th chapter has already created ceph-config, can reuse.

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

View:

    kubectl get configmap -n ceph-csi ceph-config -o yaml

---

### 14.5 Deploying CephFS CSI Components

In actual production environments, it is recommended to obtain the corresponding version YAML from the official ceph-csi release and perform image address replacement.

Typical resources include:

    csi-cephfsplugin-provisioner Deployment
    csi-cephfsplugin DaemonSet
    ServiceAccount
    ClusterRole
    ClusterRoleBinding
    Role
    RoleBinding
    CSIDriver

Check deployed resources:

    kubectl get pods -n ceph-csi -o wide
    kubectl get deploy -n ceph-csi
    kubectl get ds -n ceph-csi
    kubectl get csidriver

Expected results:

    csi-cephfsplugin-provisioner is Running normally
    csi-cephfsplugin DaemonSet is Running normally on each K8s node
    cephfs.csi.ceph.com exists in CSIDriver

---

### 14.6 Post-Deployment Verification

Check Pods:

    kubectl get pods -n ceph-csi -o wide

Check DaemonSet:

    kubectl get ds -n ceph-csi

Expected results:

    DESIRED equals READY

Check CSI Driver:

    kubectl get csidriver

Expected to include:

    cephfs.csi.ceph.com

If Pods are not functioning normally:

    kubectl describe pod -n ceph-csi <pod-name>
    kubectl logs -n ceph-csi <pod-name> -c csi-cephfsplugin

---

## FifteenI don't know.Experiment Six: Creating Kubernetes Secret

### 15.1 Creating Secret YAML

Replace <user-key> with the key for client.k8s-cephfs.

Notes:

    userID does not include the client. prefix.
    client.k8s-cephfs corresponds to userID: k8s-cephfs

To improve compatibility with different Ceph CSI versions, the experiment writes both userID/userKey and adminID/adminKey.

In production, you can split the provisioner user and node user according to CSI version requirements.

    cat > csi-cephfs-secret.yaml <<'EOF'
    apiVersion: v1
    kind: Secret
    metadata:
      name: csi-cephfs-secret
      namespace: ceph-csi
    stringData:
      userID: k8s-cephfs
      userKey: <user-key>
      adminID: k8s-cephfs
      adminKey: <user-key>
    EOF

Apply:

    kubectl apply -f csi-cephfs-secret.yaml

Check:

    kubectl get secret -n ceph-csi csi-cephfs-secret

---

### 15.2 Secret Naming Explanation

StorageClass will reference this Secret:

    csi.storage.k8s.io/provisioner-secret-name
    csi.storage.k8s.io/provisioner-secret-namespace
    csi.storage.k8s.io/controller-expand-secret-name
    csi.storage.k8s.io/controller-expand-secret-namespace
    csi.storage.k8s.io/node-stage-secret-name
    csi.storage.k8s.io/node-stage-secret-namespace

If the Secret name or namespace is incorrect, it will cause:

    PVC creation failure
    Pod mounting failure
    CSI permission errors

---

## SixteenI don't know.Experiment Seven: Creating CephFS StorageClass

### 16.1 StorageClass YAML

Replace <cluster-id> with the ceph fsid.

    cat > storageclass-cephfs.yaml <<'EOF'
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: cephfs-rwx
    provisioner: cephfs.csi.ceph.com
    parameters:
      clusterID: <cluster-id>
      fsName: cephfs
      pool: cephfs_data
      subvolumeGroup: csi
      mounter: kernel
      csi.storage.k8s.io/provisioner-secret-name: csi-cephfs-secret
      csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi
      csi.storage.k8s.io/controller-expand-secret-name: csi-cephfs-secret
      csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi
      csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
      csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi
    reclaimPolicy: Delete
    allowVolumeExpansion: true
    volumeBindingMode: Immediate
    EOF

Apply:

kubectl apply -f storageclass-cephfs.yaml

Check:

    kubectl get storageclass
    kubectl describe storageclass cephfs-rwx

---

### 16.2 Parameter Explanation

| Parameter | Description |
|---|---|
| provisioner | CSI Driver name |
| clusterID | Ceph cluster FSID |
| fsName | CephFS filesystem name |
| pool | CephFS Data Pool |
| subvolumeGroup | CSI subvolume group |
| mounter | Mount method, kernel or fuse |
| reclaimPolicy | Recycling policy after PVC deletion |
| allowVolumeExpansion | Whether volume expansion is allowed |
| volumeBindingMode | Binding mode |

This document uses:

    mounter: kernel

If kernel mounter has compatibility issues, you can test:

    mounter: fuse

---

### 16.3 reclaimPolicy Explanation

This experiment uses:

    reclaimPolicy: Delete

Meaning:

    After PVC deletion, dynamically created PV and underlying CephFS Subvolume may be deleted.

Use with caution in production.

If you want to retain PV and underlying data after PVC deletion, use:

    reclaimPolicy: Retain

Production recommendation:

    Use Delete cautiously for critical business data.
    Must clearly define data retention policy after PVC deletion before production deployment.

---

## SeventeenI don't know.Experiment 8: Create ReadWriteMany PVC

### 17.1 Create PVC

    cat > pvc-cephfs-demo.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: cephfs-demo-pvc
      namespace: cephfs-demo
    spec:
      accessModes:
        - ReadWriteMany
      storageClassName: cephfs-rwx
      resources:
        requests:
          storage: 5Gi
    EOF

Apply:

    kubectl apply -f pvc-cephfs-demo.yaml

---

### 17.2 Check PVC

    kubectl get pvc -n cephfs-demo

Expected output:

    NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
    cephfs-demo-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWX            cephfs-rwx

---

### 17.3 Check PV

    kubectl get pv

Check details:

    kubectl describe pvc -n cephfs-demo cephfs-demo-pvc
    kubectl describe pv <pv-name>

Focus on:

- StorageClass
- CSI Driver
- VolumeHandle
- Events
- Capacity
- AccessModes

---

## EighteenI don't know.Experiment 9: Create Multiple Pods Mounting Same PVC

### 18.1 Create Two Test Pods

Using same PVC:

    cephfs-demo-pvc

Write YAML:

    cat > pods-cephfs-rwx-demo.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: cephfs-pod-a
      namespace: cephfs-demo
    spec:
      containers:
        - name: app
          image: registry.cn-hangzhou.aliyuncs.com/pub-syq/busybox:latest
          command:
            - sh
            - -c
            - "while true; do sleep 3600; done"
          volumeMounts:
            - name: shared-data
              mountPath: /data
      volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: cephfs-demo-pvc
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: cephfs-pod-b
      namespace: cephfs-demo
    spec:
      containers:
        - name: app
          image: registry.cn-hangzhou.aliyuncs.com/pub-syq/busybox:latest
          command:
            - sh
            - -c
            - "while true; do sleep 3600; done"
          volumeMounts:
            - name: shared-data
              mountPath: /data
      volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: cephfs-demo-pvc
    EOF

Note: /data

If there is no such busybox image, you can replace it with an existing busybox image from your Harbor or Alibaba Cloud repository.  
It is recommended to use a controlled image source in both production and experimental environments to avoid ImagePullBackOff.

**Application:**

    kubectl apply -f pods-cephfs-rwx-demo.yaml

**View:**

    kubectl get pod -n cephfs-demo -o wide

**Expected Output:**

    cephfs-pod-a Running
    cephfs-pod-b Running

---

### 18.2 View Pod Mount Events

    kubectl describe pod -n cephfs-demo cephfs-pod-a
    kubectl describe pod -n cephfs-demo cephfs-pod-b

Focus on:

- Whether scheduling was successful
- Whether PVC mounting was successful
- Whether FailedMount exists
- Whether ImagePullBackOff exists
- Whether MountVolume.SetUp failed

---

## Nineteen、Experiment Ten: Verify Multi-Pod Shared Read-Write

### 19.1 Pod A Writes a File

    kubectl exec -n cephfs-demo cephfs-pod-a -- sh -c "echo 'write from pod A' > /data/a.txt"

**View:**

    kubectl exec -n cephfs-demo cephfs-pod-a -- cat /data/a.txt

**Expected Output:**

    write from pod A

---

### 19.2 Pod B Reads Pod A's File

    kubectl exec -n cephfs-demo cephfs-pod-b -- cat /data/a.txt

**Expected Output:**

    write from pod A

---

### 19.3 Pod B Writes a File

    kubectl exec -n cephfs-demo cephfs-pod-b -- sh -c "echo 'write from pod B' > /data/b.txt"

**View:**

    kubectl exec -n cephfs-demo cephfs-pod-b -- cat /data/b.txt

**Expected Output:**

    write from pod B

---

### 19.4 Pod A Views Pod B's File

    kubectl exec -n cephfs-demo cephfs-pod-a -- ls -l /data

    kubectl exec -n cephfs-demo cephfs-pod-a -- cat /data/b.txt

**Expected Output:**

    write from pod B

---

### 19.5 Experiment Conclusion

This experiment demonstrates:

    The same CephFS PVC can be mounted by multiple Pods simultaneously.
    Data written by Pod A can be read by Pod B.
    This is the typical ReadWriteMany capability in Kubernetes.

---

## TwentyI don't know.Experiment Eleven: Verify Data Persistence After Pod Deletion and Recreation

### 20.1 Delete Pod

    kubectl delete pod -n cephfs-demo cephfs-pod-a
    kubectl delete pod -n cephfs-demo cephfs-pod-b

**Confirm:**

    kubectl get pod -n cephfs-demo

---

### 20.2 Recreate Pod

    kubectl apply -f pods-cephfs-rwx-demo.yaml

**Wait for Running:**

    kubectl get pod -n cephfs-demo -o wide

---

### 20.3 Verify Data Still Exists

    kubectl exec -n cephfs-demo cephfs-pod-a -- ls -l /data

    kubectl exec -n cephfs-demo cephfs-pod-a -- cat /data/a.txt

    kubectl exec -n cephfs-demo cephfs-pod-a -- cat /data/b.txt

**Expected Output:**

    a.txt and b.txt still exist.
    The content remains correct.

**Note:**

    Deleting a Pod does not delete PVC data.
    The CephFS Subvolume corresponding to the PVC still exists.

---

## Twenty-OneI don't know.Experiment Twelve: Verify CephFS Subvolume

### 21.1 View SubvolumeGroup

Execute on Ceph management node:

    ceph fs subvolumegroup ls cephfs

**Expected Output:**

    csi

---

### 21.2 View Subvolume

    ceph fs subvolume ls cephfs --group_name csi

**Expected to see similar:**

    csi-vol-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

---

### 21.3 View Subvolume Information

Replace <subvolume-name>:

    ceph fs subvolume info cephfs <subvolume-name> --group_name csi

Focus on:

- path
- bytes_quota
- mode
- uid
- gid
- created_at
- data_pool

**Note:**

    The Kubernetes PVC corresponds to the CephFS Subvolume.
    Whether the subvolume is deleted when the PVC is deleted depends on the StorageClass recycling policy and CSI behavior.

---

### 21.4 View CephFS Status

    ceph fs status
    ceph mds stat
    ceph -s

**Confirm:**

    MDS is normal.
    PG is active+clean.
    No abnormal alerts.

---

## Twenty-TwoI don't know.Experiment Thirteen: PVC Expansion

### 22.1 Prerequisites

The StorageClass must enable:

    allowVolumeExpansion: true

**View:**

    kubectl get storageclass cephfs-rwx -o yaml | grep allowVolumeExpansion

---

### 22.2 Expand PVC

Expand the PVC from 5Gi to 10Gi: /think

kubectl patch pvc cephfs-demo-pvc -n cephfs-demo \
  -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

Check:

  kubectl get pvc -n cephfs-demo

Expected:

  CAPACITY becomes 10Gi

---

### 22.3 View Pod Capacity

  kubectl exec -n cephfs-demo cephfs-pod-a -- df -h /data

If the capacity does not change immediately, check PVC events:

  kubectl describe pvc -n cephfs-demo cephfs-demo-pvc

Check CSI logs:

  kubectl get pods -n ceph-csi
  kubectl logs -n ceph-csi <cephfs-provisioner-pod> -c csi-cephfsplugin

---

### 22.4 Verify Subvolume Quotas in Ceph

Check subvolume:

  ceph fs subvolume ls cephfs --group_name csi

Check information:

  ceph fs subvolume info cephfs <subvolume-name> --group_name csi

Focus on:

  bytes_quota

---

## Twenty-Three, Experiment Fourteen: Clean Up Test Resources

### 23.1 Delete Test Pods

  kubectl delete pod -n cephfs-demo cephfs-pod-a
  kubectl delete pod -n cephfs-demo cephfs-pod-b

---

### 23.2 Delete PVC

High-risk warning:

  If the StorageClass reclaimPolicy is Delete, deleting the PVC may delete the underlying CephFS Subvolume.
  Do not directly delete PVCs for production data.

Delete:

  kubectl delete pvc -n cephfs-demo cephfs-demo-pvc

Check:

  kubectl get pvc -n cephfs-demo
  kubectl get pv

---

### 23.3 Verify Subvolume Deletion

Execute on Ceph management node:

  ceph fs subvolume ls cephfs --group_name csi

If reclaimPolicy is Delete, the related subvolume should be deleted.

If it still exists, check PV status and CSI logs.

---

### 23.4 Delete StorageClass

  kubectl delete storageclass cephfs-rwx

---

### 23.5 Delete Secret

  kubectl delete secret -n ceph-csi csi-cephfs-secret

---

### 23.6 Delete Test Namespace

  kubectl delete namespace cephfs-demo

If no longer testing Ceph CSI, you can also delete the ceph-csi Namespace, but it will delete CSI components:

  kubectl delete namespace ceph-csi

Do not arbitrarily delete the ceph-csi Namespace in production environments.

---

### 23.7 Whether to Delete CephFS and Pool

Do not arbitrarily delete CephFS and Pool in production environments.

If confirmed that the entire CephFS is no longer needed in test environments:

  ceph fs fail cephfs
  ceph fs rm cephfs --yes-i-really-mean-it

Then delete the Pool:

  ceph config set mon mon_allow_pool_delete true

  ceph osd pool rm cephfs_metadata cephfs_metadata --yes-i-really-really-mean-it
  ceph osd pool rm cephfs_data cephfs_data --yes-i-really-really-mean-it

  ceph config set mon mon_allow_pool_delete false

High-risk warning:

  Deleting CephFS and Pool will delete all filesystem data.
  Do not directly execute in production environments.
  Even in test environments, confirm there is no data before proceeding.

---

## Twenty-Four, Common Issues and Troubleshooting

### 24.1 CephFS CSI Pod Image Pull Failure

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
- Synchronize the image to Harbor or Alibaba Cloud repository
- Modify the image address in YAML
- Do not recommend directly modifying containerd global configuration

---

### 24.2 CephFS CSI Pod Not Running

Check:

  kubectl get pods -n ceph-csi -o wide
  kubectl describe pod -n ceph-csi <pod-name>
  kubectl logs -n ceph-csi <pod-name> -c csi-cephfsplugin

Common causes:

- Missing RBAC
- ConfigMap error
- Secret error
- Image pull failure
- Taints not tolerated
- kubelet plugin directory permission issues
- CSI Driver not registered

Check CSI Driver:

  kubectl get csidriver

Confirm existence:

  cephfs.csi.ceph.com

---

### 24.3 PVC Stays Pending

Check:

  kubectl describe pvc -n cephfs-demo cephfs-demo-pvc

Focus on Events.

Common causes:

- StorageClass does not exist  
- Provisioner name is incorrect  
- CSI Controller is not working  
- Secret name or namespace is incorrect  
- clusterID is incorrect  
- fsName is incorrect  
- CephFS does not exist  
- SubvolumeGroup does not exist  
- Ceph user permissions are insufficient  
- Ceph MON is unreachable  
- MDS is not active  

Troubleshooting:  

    kubectl get storageclass  
    kubectl get pods -n ceph-csi  
    kubectl logs -n ceph-csi <provisioner-pod>  
    ceph -s  
    ceph fs ls  
    ceph fs status  
    ceph auth get client.k8s-cephfs  
    ceph fs subvolumegroup ls cephfs  

---

### 24.4 Pod FailedMount  

Check:  

    kubectl describe pod -n cephfs-demo cephfs-pod-a  

Common events:  

    MountVolume.SetUp failed  
    permission denied  
    context deadline exceeded  
    failed to get secret  
    failed to connect to monitors  
    error mounting ceph filesystem  

Troubleshooting:  

    kubectl get secret -n ceph-csi csi-cephfs-secret  
    kubectl get configmap -n ceph-csi ceph-csi-config -o yaml  
    kubectl logs -n ceph-csi <node-plugin-pod> -c csi-cephfsplugin  
    ceph -s  
    ceph fs status  

Check on the node where the Pod is located:  

    modprobe ceph  
    lsmod | grep ceph  
    dmesg | tail -50  

---

### 24.5 CephFS MDS is not active  

Check:  

    ceph fs status  
    ceph mds stat  
    ceph orch ps --daemon_type mds  

Common causes:  

- MDS is not deployed  
- MDS container is abnormal  
- MDS node is faulty  
- MDS cannot connect to MON  
- metadata pool is abnormal  
- CephFS creation is incomplete  

Handling:  

    ceph orch apply mds cephfs --placement="2 ceph-node01 ceph-node02 ceph-node03"  
    ceph orch ps --daemon_type mds  
    ceph fs status  
    ceph -s  

---

### 24.6 Ceph Permission Errors  

Symptoms:  

    permission denied  
    operation not permitted  
    access denied  

Check Ceph user:  

    ceph auth get client.k8s-cephfs  

Confirm caps:  

    mon 'allow r'  
    mgr 'allow rw'  
    mds 'allow rw fsname=cephfs'  
    osd 'allow rw tag cephfs data=cephfs'  

If dynamic subvolume creation fails, temporarily relax validation in a test environment:  

    ceph auth caps client.k8s-cephfs \
      mon 'allow r' \
      mgr 'allow rw' \
      mds 'allow rw' \
      osd 'allow rw tag cephfs *=*'  

Verify again and converge to production security requirements afterward.  

---

### 24.7 clusterID is incorrect  

Symptoms:  

    CSI cannot find the corresponding cluster configuration  

Check:  

    ceph fsid  
    kubectl get configmap -n ceph-csi ceph-csi-config -o yaml  
    kubectl get storageclass cephfs-rwx -o yaml  

Requirements:  

    clusterID in ConfigMap  
    clusterID in StorageClass  
    Ceph fsid  

All three must be consistent.  

---

### 24.8 fsName is incorrect  

Symptoms:  

    PVC creation fails  
    "filesystem not found" appears in CSI logs  

Check:  

    ceph fs ls  
    kubectl get storageclass cephfs-rwx -o yaml  

Requirements:  

    fsName: cephfs  

Must match the filesystem name listed in:  

    ceph fs ls  

---

### 24.9 SubvolumeGroup does not exist  

Symptoms:  

    PVC creation fails  
    "subvolume group not found" appears in logs  

Check:  

    ceph fs subvolumegroup ls cephfs  

If there is no csi:  

    ceph fs subvolumegroup create cephfs csi  

---

### 24.10 PVC deleted but Subvolume not deleted  

Possible causes:  

- reclaimPolicy is Retain  
- CSI Controller is abnormal  
- Finalizer is not cleaned  
- Ceph lacks deletion permissions  
- PV still exists  
- PVC is not fully deleted  

Check:  

    kubectl get pv  
    kubectl describe pv <pv-name>  
    kubectl get pvc -A  
    ceph fs subvolume ls cephfs --group_name csi  

Handling approach:  

    First confirm whether data needs to be retained.  
    Do not directly delete CephFS subvolume.  
    Handle after confirming PV/PVC status and reclaimPolicy.  

---

### 24.11 PVC Expansion Failed  

Check:

kubectl describe pvc -n cephfs-demo cephfs-demo-pvc
kubectl get storageclass cephfs-rwx -o yaml
kubectl logs -n ceph-csi <provisioner-pod>
kubectl logs -n ceph-csi <node-plugin-pod> -c csi-cephfsplugin

Common Causes:

- StorageClass not enabling allowVolumeExpansion
- csi-resizer not deployed
- Ceph user permissions insufficient
- Subvolume quota modification failed
- CephFS status abnormal
- MDS abnormal

---

## 25. CephFS CSI Common Commands Summary

### 25.1 Ceph Side

    ceph -s
    ceph fs ls
    ceph fs status
    ceph mds stat
    ceph orch ps --daemon_type mds

    ceph fs subvolumegroup create cephfs csi
    ceph fs subvolumegroup ls cephfs
    ceph fs subvolume ls cephfs --group_name csi
    ceph fs subvolume info cephfs <subvolume-name> --group_name csi

    ceph auth get client.k8s-cephfs
    ceph auth get-key client.k8s-cephfs

---

### 25.2 Kubernetes Side

    kubectl get nodes -o wide
    kubectl get pods -n ceph-csi -o wide
    kubectl get storageclass
    kubectl get pvc -n cephfs-demo
    kubectl get pv
    kubectl describe pvc -n cephfs-demo cephfs-demo-pvc
    kubectl describe pod -n cephfs-demo cephfs-pod-a
    kubectl get csidriver

---

### 25.3 Log Troubleshooting

    kubectl logs -n ceph-csi <provisioner-pod>
    kubectl logs -n ceph-csi <node-plugin-pod> -c csi-cephfsplugin
    kubectl describe pod -n ceph-csi <pod-name>
    kubectl describe pvc -n cephfs-demo <pvc-name>
    kubectl describe pod -n cephfs-demo <pod-name>

---

### 25.4 Read/Write Testing

    kubectl exec -n cephfs-demo cephfs-pod-a -- sh -c "echo hello > /data/hello.txt"
    kubectl exec -n cephfs-demo cephfs-pod-b -- cat /data/hello.txt
    kubectl exec -n cephfs-demo cephfs-pod-a -- df -h /data

---

## 26. Production Environment Notes

### 26.1 Do Not Use client.admin

Admin key can be used for verification in experiments.

Production must use minimal privilege users:

    client.k8s-cephfs

And restrict to specific CephFS.

---

### 26.2 Define reclaimPolicy Explicitly

Production must specify:

    Delete or Retain

Delete:

    PVC deletion may delete underlying CephFS Subvolume.
    Suitable for temporary environments or discardable data.

Retain:

    PVC deletion retains PV and underlying data.
    Suitable for critical business, but requires manual recovery process.

Shared file data, user upload data, business attachment directories, etc., should be cautious with Delete in production scenarios.

---

### 26.3 MDS Must Be Highly Available

CephFS strongly depends on MDS.

Production recommendations:

    At least 1 active MDS
    At least 1 standby MDS

Check:

    ceph fs status
    ceph mds stat

If no standby exists, CephFS availability will be affected when MDS fails.

---

### 26.4 Metadata Pool Is Critical

CephFS metadata pool stores:

- Directory structure
- File names
- inode
- Permissions
- Metadata

If metadata pool fails, CephFS will be severely impacted.

Production recommendations:

- Use reliable replication policy for metadata pool
- Monitor metadata pool capacity
- Monitor metadata pool PG status
- Avoid metadata pool nearfull
- Focus on MDS performance for small file scenarios

---

### 26.5 Image Must Use Fixed Version

Production should not use latest.

Recommendations:

    Fix Ceph CSI version
    Fix sidecar version
    Sync images to internal repository
    YAML should be under Git management
    Validate upgrades in test clusters before deployment

---

### 26.6 Do Not Break Kubernetes Underlying Runtime

Avoid modifying containerd global configuration arbitrarily for pulling CSI images.

Recommendations:

- Internal Harbor
- Alibaba Cloud image repository
- imagePullSecrets
- Unified image synchronization process
- Fixed image tag

---

### 26.7 CephFS Is Suitable for RWX, Not All Data

CephFS is suitable for:

    Shared files
    Shared upload directories
    Multi-Pod shared datasets

Not recommended for:

    Database primary data disks

Database primary data disks are usually better suited for:

    RBD CSI

---

### 26.8 Large Amount of Small Files Requires Pre-evaluation

Large amounts of small files increase MDS pressure.

Monitor:

- MDS CPU
- MDS memory
- Metadata cache
- Directory item count
- File count
- Client count
- Small file creation/deletion frequency

If the business involves massive small files, pre-stress testing is required.

---

### 26.9 Monitoring Must Cover CephFS CSI

Production monitoring must include:

- ceph-csi Pod status
- PVC Pending count
- PV abnormal status
- Pod FailedMount events
- CephFS Subvolume count
- CephFS metadata/data pool capacity
- MDS active/standby status
- MDS CPU / memory
- Ceph OSD status
- Ceph PG status
- CSI error logs
- kubelet mount events

---

## 27. Advanced SRE Methodology

### 27.1 CephFS CSI Troubleshooting is Divided into Five Layers

**First Layer: Kubernetes Objects**

    PVC
    PV
    Pod
    StorageClass
    Secret
    Events

**Second Layer: CSI Components**

    csi-cephfsplugin-provisioner
    csi-cephfsplugin DaemonSet
    csi-provisioner
    csi-resizer
    node-driver-registrar

**Third Layer: Kubernetes Nodes**

    kubelet
    containerd
    Ceph kernel module
    Node to Ceph network
    dmesg
    mount status

**Fourth Layer: CephFS Services**

    CephFS filesystem
    MDS active/standby
    SubvolumeGroup
    Subvolume
    CephX permissions

**Fifth Layer: Ceph Backend**

    Ceph health
    OSD / PG status
    metadata/data pool
    capacity water level
    network and disk performance

---

### 27.2 PVC Pending is Not Necessarily a Kubernetes Issue

PVC Pending may originate from:

- StorageClass error
- CSI Controller abnormality
- Secret error
- Ceph permission error
- fsName error
- CephFS does not exist
- SubvolumeGroup does not exist
- MDS is not active
- MON is unreachable
- clusterID error

Troubleshooting cannot only focus on `kubectl get pvc`.

Must check:

    kubectl describe pvc
    CSI Controller logs
    CephFS status
    MDS status
    Ceph auth permissions
    SubvolumeGroup status

---

### 27.3 Pod FailedMount is Not Necessarily a CephFS Backend Issue

Pod FailedMount may originate from:

- kubelet abnormality
- Node CSI plugin abnormality
- Kernel Ceph module issue
- Node to Ceph network abnormality
- Secret error
- Mounter configuration incompatibility
- MDS abnormality
- Insufficient permissions

Therefore, analysis must combine node, CSI logs, and CephFS status.

---

### 27.4 CephFS Core Risks Lie in MDS and Metadata

RBD's core focus is more on block devices, RBD Image, and OSD performance.

CephFS also needs to pay attention to:

    MDS
    metadata pool
    small files
    large directories
    multi-client concurrency
    directory permissions
    subvolume lifecycle

CephFS operations cannot only rely on `ceph -s`, but also need to use:

    ceph fs status
    ceph mds stat
    ceph fs subvolume ls
    CephFS CSI logs

---

## 28. Interview Answer Structure

If asked:

    How does Kubernetes use CephFS as shared storage?

Can answer:

    Kubernetes typically uses CephFS CSI to leverage CephFS as shared file storage. CephFS is suitable for ReadWriteMany scenarios, allowing multiple Pods to mount the same PVC for shared read/write, common use cases include uploading directories, shared datasets, and multi-replica web applications sharing files.
    Before deployment, prepare CephFS filesystem in Ceph, including metadata pool, data pool, and MDS service. In production, MDS must have at least active and standby instances. Then create a CephX user for CephFS CSI usage, granting minimal permissions for mon, mgr, mds, osd to the specified CephFS.
    On Kubernetes side, deploy CephFS CSI including Controller Deployment and Node DaemonSet. Then create ceph-csi-config ConfigMap to configure clusterID and MON address; create Secret to store Ceph user; create StorageClass with provisioner set to cephfs.csi.ceph.com, specifying fsName, pool, subvolumeGroup, Secret, and reclaimPolicy.
    After business creates ReadWriteMany PVC, CephFS CSI will create corresponding subvolume in CephFS and generate PV. Multiple Pods mounting the same PVC can share read/write access to the same directory.
    Troubleshooting involves multi-layer checks including PVC/PV/Pod events, CSI Controller logs, CSI Node logs, Secret, StorageClass, CephFS status, MDS status, SubvolumeGroup, CephX permissions, OSD/PG status, etc.
    In production, special attention should be paid to MDS high availability, metadata pool health, reclaimPolicy, minimal permissions, fixed image version, monitoring alerts, performance of large numbers of small files, and data backup strategy.

---

## 29. Summary of This Article

This article mainly organizes the practice of CephFS CSI integration with Kubernetes:

1. CephFS CSI is responsible for providing Kubernetes with ReadWriteMany shared file storage.
2. CephFS CSI is suitable for multiple Pods to share read/write access to the same directory.
3. RBD CSI acts like a cloud disk, while CephFS CSI acts like a shared file system.
4. CephFS CSI depends on the CephFS file system and MDS.
5. Before deployment, ensure that CephFS exists and MDS is active and functioning normally.
6. A SubvolumeGroup must be created, for example, named `csi`.
7. A minimal-privilege user `client.k8s-cephfs` must be created.
8. Kubernetes requires deployment of CephFS CSI Controller and Node plugins.
9. The `ceph-csi-config`, `ceph-config`, Secret, and StorageClass must be created.
10. After creating an RWX PVC, CephFS CSI automatically creates a CephFS Subvolume.
11. Multiple Pods can mount the same PVC simultaneously.
12. Data written by Pod A can be read by Pod B.
13. Deleting a Pod does not delete PVC data.
14. PVC expansion requires the StorageClass to have `allowVolumeExpansion` enabled and depends on CSI resizer.
15. Whether deleting a PVC deletes the underlying subvolume depends on `reclaimPolicy`.
16. In production environments, focus must be given to MDS high availability, metadata pool, permission control, image version, monitoring alerts, and backup strategy.
17. Advanced SREs troubleshooting CephFS CSI issues must analyze from five layers: Kubernetes, CSI, nodes, CephFS service, and Ceph backend.

---

## 30. Reference Documents

Ceph CSI project: 

    https://github.com/ceph/ceph-csi

Ceph CSI CephFS documentation:

    https://github.com/ceph/ceph-csi/tree/devel/docs

CephFS official documentation:

    https://docs.ceph.com/en/latest/cephfs/

CephFS creating file system:

    https://docs.ceph.com/en/latest/cephfs/createfs/

CephFS client mounting:

    https://docs.ceph.com/en/latest/cephfs/mount-using-kernel-driver/

Ceph-FUSE mounting:

    https://docs.ceph.com/en/latest/cephfs/mount-using-fuse/

CephX permission management:

    https://docs.ceph.com/en/latest/rados/operations/user-management/

Kubernetes CSI documentation:

    https://kubernetes.io/docs/concepts/storage/volumes/#csi

Kubernetes StorageClass documentation:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes Persistent Volumes documentation:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/