# Ceph Integration with Kubernetes: Practical Applications of CephFS CSI File Sharing Storage

Recommended Path: 05-Storage/01-Ceph/13-Ceph Integration with Kubernetes: Practical Applications of CephFS CSI File Sharing Storage.md

Tags: #Ceph #Kubernetes #CephFS #CSI #StorageClass #PVC #PV #ReadWriteMany #Shared Storage #MDS #Cloud-Native Storage #SRE #Advanced SRE

---

## I. Document Overview

This article is the thirteenth in the Ceph Advanced SRE storage series, focusing on the practical integration of CephFS with Kubernetes.

Previous steps have included:

- Deploying a Ceph cluster
- Managing OSDs
- Setting up Pools and PGs
- Configuring CRUSH rules
- Implementing RBD block storage
- Utilizing CephFS for file storage
- Applying RGW for object storage
- Creating dynamic volume provisioning using RBD CSI in Kubernetes

This article further explores the combination of Ceph and Kubernetes:

    Using CephFS as shared file storage in Kubernetes

The main focus of this article is:

    Providing ReadWriteMany shared file storage for Kubernetes using CephFS CSI

Typical use cases include:

- Multiple Pods simultaneously mounting the same PVC
- CephFS CSI creating a Subvolume for shared access
- Pods reading and writing data through shared directories
- Persistent data storage in CephFS

The most common access pattern for CephFS in Kubernetes is:

    ReadWriteMany

This means that:

- Multiple Pods can mount the same PVC simultaneously.
- Pods on different nodes can share the same file directory.
- It is ideal for sharing upload directories, configuration files, and datasets.

Differences between CephFS CSI and RBD CSI:

- RBD CSI is better suited for ReadWriteOnce scenarios, similar to cloud block storage.
- CephFS CSI is designed for ReadWriteMany, similar to shared file systems.

---

## II. Experimental Objectives

After completing this article, you should be able to:

1. Understand the concept of ReadWriteMany in Kubernetes.
2. Distinguish between CephFS CSI and RBD CSI.
3. Recognize the roles of the Controller and Node plugins in CephFS CSI.
4. Create or reuse a CephFS file system.
5. Set up a SubvolumeGroup for CephFS CSI.
6. Configure a minimal-privilege user for CephFS CSI.
7. Deploy CephFS CSI in Kubernetes.
8. Create the necessary ConfigMap and Secret resources for CephFS CSI.
9. Define a CephFS StorageClass.
10. Create a ReadWriteMany PVC.
11. Mount multiple Pods to the same PVC simultaneously.
12. Verify shared file access among multiple Pods.
13. Ensure data persistence after Pod deletion and recreation.
14. Test PVC scaling.
15. Check the created subvolumes in Ceph.
16. Troubleshoot issues such as Pending PVCs, FailedMount Pods, MDS exceptions, permission errors, and CSI Pod failures.
17. Understand the security, mirroring, high availability, and recycling strategies for CephFS CSI in production environments.

---

## III. Experimental Environment

### 3.1 Ceph Cluster

The Ceph cluster is deployed independently on the following nodes:

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Testing (optional) |

Requirements for the Ceph cluster:

- Normal MON quorum
- Functional MGR
- All OSDs up and running
- Active+clean PGs
- Normal active/standby MDS configuration
- `ceph -s` output showing HEALTH_OK or interpretable HEALTH_WARN

---

### 3.2 Kubernetes Cluster

The Kubernetes cluster is deployed independently on the following nodes:

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Operating environment:

- Kubeadm cluster
- Containerd runtime
- Calico CNI
- Ubuntu Server 22.04.5 LTS

Notes:

- The Ceph nodes and Kubernetes nodes are kept separate.
- Kubernetes nodes only serve as clients for mounting the Ceph---

## VII. Pre-Operation Checks

### 7.1 Check the Ceph Cluster Status

Execute the following commands on the Ceph management node:

    ceph -s
    ceph health detail
    ceph osd tree
    ceph pg stat
    ceph df

Requirements:

    OSDs should be up or in progress.
    PGs must be active and clean.
    There should be sufficient capacity.
    No nearfull or full status observed.
    No unexplained HEALTH_ERR messages.

---

### 7.2 Check the CephFS Status

Verify if a CephFS exists:

    ceph fs ls

Check the CephFS status:

    ceph fs status

Inspect the MDS:

    ceph mds stat

Expected results:

    CephFS should be present.
    At least one active MDS should be available.
    It is recommended to have at least one standby MDS.
    The metadata and data pools should be functioning normally.

---

### 7.3 Check the Kubernetes Cluster

Execute the following commands on the k8s-master node:

    kubectl get nodes -o wide
    kubectl get pods -A

Requirements:

    All nodes must be in the Ready state.
    CoreDNS and CNI should be functioning correctly.
    There should be no abnormal Pods in the kube-system namespace.

---

### 7.4 Check the Network Connectivity between Kubernetes Nodes and Ceph MON

Execute the following commands on each Kubernetes node:

    ping -c 3 10.0.0.31
    ping -c 3 10.0.0.32
    ping -c 3 10.0.0.33

Check the MON ports:

    nc -vz 10.0.0.31 3300
    nc -vz 10.0.0.31 6789
    nc -vz 10.0.0.32 3300
    nc -vz 10.0.0.32 6789
    nc -vz 10.0.0.33 3300
    nc -vz 10.0.0.33 6789

If nc is not available, install it using:

    apt install -y netcat-openbsd

---

### 7.5 Verify if the Kubernetes Node Kernel Supports CephFS

Execute the following command on each Kubernetes node:

    modprobe ceph
    lsmod | grep ceph

If the ceph-fuse mounter is used, the user-space fuse tool is required.

In production environments, it is generally recommended to use the kernel mounter. However, this decision should be based on a thorough evaluation of the kernel version and compatibility.

---

## VIII. Experimental Task List

| Experiment | Objective | Risk Level |
|---|---|---|
| Experiment 1 | Create or confirm a CephFS file system | Medium |
| Experiment 2 | Create a CephFS SubvolumeGroup | Medium |
| Experiment 3 | Create a CephFS CSI user with minimal permissions | Medium |
| Experiment 4 | Prepare a Kubernetes Namespace | Low |
| Experiment 5 | Deploy a CephFS CSI | Medium |
| Experiment 6 | Create a Secret | Medium |
| Experiment 7 | Create a CephFS StorageClass | Medium |
| Experiment 8 | Create an RWX PVC | Medium |
| Experiment 9 | Create multiple Pods and mount them with the same PVC | Medium |
| Experiment 10 | Verify multi-Pod shared read and write access | Low |
| Experiment 11 | Verify data persistence after Pod deletion and reconstruction | Low |
| Experiment 12 | Verify CephFS Subvolume functionality | Low |
| Experiment 13 | Expand a PVC | Medium |
| Experiment 14 | Clean up test resources | High |
| Experiment 15 | Troubleshoot common issues | Medium to High |

High-Risk Notes:

    Deleting a PVC may result in the deletion of the underlying CephFS Subvolume, depending on the reclaimPolicy setting.
    Deleting a CephFS Pool will erase all data associated with that file system.
    Deactivating a CephFS file system will affect all services that rely on it.
    In production environments, clear recovery and data retention policies must be established.

---

## IX. Experiment 1: Create or Confirm a CephFS File System

If you have already completed the CephFS practice in Chapter 10 and already have a CephFS file system set up, you can skip the creation steps and proceed directly with the checks.

### 9.1 View the CephFS Status

Execute the following commands:

    ceph fs ls
    ceph fs status
    ceph mds stat

```markdown
K8S_CEPHFS_KEY=$(ceph auth get-key client.k8s-cephfs)

View:

echo $K8S_CEPHFS_KEY

Security Reminder:

The key is considered sensitive information.
Do not commit it to Git.
Do not include it in public documentation.
In a production environment, use Secrets for management and access control.
---

## Section Twelve: Obtaining the Ceph Cluster FSID and MON Addresses

### 12.1 Obtaining the FSID

Execute on the Ceph management node:

ceph fsid

Example:

11111111-2222-3333-4444-555555555555

Record it as:

clusterID

The clusterID for CephFS CSI is usually the same as the Ceph FSID.

---

### 12.2 Obtaining the MON Addresses

View:

ceph mon dump

Record the MON addresses:

10.0.0.31:6789
10.0.0.32:6789
10.0.0.33:6789

You can also use the v2 ports:

10.0.0.31:3300
10.0.0.32:3300
10.0.0.33:3300

Using port 6789 is more intuitive for experiments.

---

## Section Thirteen: Experiment Four: Preparing the Kubernetes Namespace

### 13.1 Creating the ceph-csi Namespace

If you have already created the ceph-csi namespace in Chapter Twelve, you can skip this step.

kubectl create namespace ceph-csi

View:

kubectl get ns ceph-csi

---

### 13.2 Creating the Testing Business Namespace

kubectl create namespace cephfs-demo

View:

kubectl get ns cephfs-demo

Explanation:

The ceph-csi namespace is used for deploying CSI components and Secrets.
The cephfs-demo namespace is used for creating PVCs and Pods for testing purposes.

---

## Section Fourteen: Experiment Five: Deploying CephFS CSI

### 14.1 Explanation of Deployment Methods

Common methods for deploying CephFS CSI include:

1. Official YAML manifests
2. Helm Charts
3. GitOps management
4. Built-in storage plugins in the platform

This document uses YAML manifests to explain the core resources.

In production, it is recommended to:

- Fix the version of CephFS CSI.
- Use a fixed image version.
- Synchronize the images to an internal network Harbor or Alibaba Cloud repository.
- Avoid using the "latest" version.
- Manage the YAML files through Git.

---

### 14.2 Images and Optimization for Domestic Networks

CephFS CSI may involve multiple images, such as:

- cephcsi image
- csi-provisioner
- csi-resizer
- csi-snapshotter (depending on whether snapshots are enabled)
- csi-node-driver-registrar

For domestic network environments, it is suggested to:

- First pull fixed version images from the official repository.
- Then re-tag them for use in the internal network Harbor or Alibaba Cloud image repository.
- Use the internal image addresses consistently in the Kubernetes manifests.
- Avoid modifying the underlying containerd on Kubernetes nodes as a primary solution.

To check image retrieval issues, use:

kubectl describe pod -n ceph-csi <pod-name>

Common errors include:

ImagePullBackOff
ErrImagePull

---

### 14.3 Creating the ceph-csi-config ConfigMap

If you have already created the ceph-csi-config ConfigMap in Chapter Twelve for RBD CSI, and if the clusterID and monitors are the same, you can reuse it.

Otherwise, create a new one.

Replace <cluster-id> with the Ceph FSID:

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

Apply the configuration:

kubectl apply -f csi-config-map.yaml

View the result:

kubectl get configmap -n ceph-csi ceph-csi-config -o yaml

---

### 14.4 Creating the ceph-config ConfigMap

If you have already created the ceph-config ConfigMap in Chapter Twelve, you can reuse it.

cat >    NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
    cephfs-pod-a        Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWX            cephfs-rwx
    cephfs-pod-b        Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWX            cephfs-rwx

---

### 18.3 查看 Pod 的 VolumeMounts

    kubectl get pod -n cephfs-demo -o wide | grep shared-data

预期输出：

    volumeMounts:
      - name: shared-data
        mountPath: /data

---

## 十九、实验十：创建多个 Pod 使用不同的 StorageClass

### 19.1 创建两个测试 Pod

使用不同的 StorageClass：

- one with `cephfs-rwx`
- another with `cephfs-read-only`

写入 YAML：

```yaml
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
    storageClassName: cephfs-rwx

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
    storageClassName: cephfs-read-only
```

应用：

```bash
kubectl apply -f pods-cephfs-rwx-demo.yaml
kubectl apply -f pods-cephfs-read-only.yaml
```

查看：

```bash
kubectl get pod -n cephfs-demo -o wide
```

预期输出：

```yaml
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
cephfs-pod-a        Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWX            cephfs-rwx
cephfs-pod-b        Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RO           cephfs-read-only
```

---

## Twenty, Experiment Eleven: Creating a Pod with Multiple CSI Drivers

### 21.1 Creating a Test Pod

Use one CSI Driver:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: csi-driver-test-pod
  namespace: csi-drivers-demo
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
    csiDriver: csi.storage.k8s.io/cephfs.csi.ceph.com
```

Use two CSI Drivers:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: csi-driver-test-pod-2
  namespace: csi-drivers-demo
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
    csiDriver: csi.storage.k8s.io/cephfs.csi.ceph.com
  volumeMounts:
  - name: shared-data
    mountPath: /data
    csiDriver: csi.storage.k8s.io/cephfs.csi.gcp.com
```

Application:

```bash
kubectl apply -f csi-driver-test-pod.yaml
kubectl apply -f csi-driver-test-pod-2.yaml
```

View:

```bash
kubectl get pod -n csi-drivers-demo
```

Expected output:

```yaml
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   CSI DRIVER
csicephfs-pod-a is Running.
cephfs-pod-b is Running.

---

### 18.2 Checking Pod Mount Events

    kubectl describe pod -n cephfs-demo cephfs-pod-a
    kubectl describe pod -n cephfs-demo cephfs-pod-b

Key points to check:

- Whether it was successfully scheduled.
- Whether the PVC was successfully mounted.
- If there are any FailedMount events.
- If there are any ImagePullBackOff events.
- If there are any MountVolume.SetUp failures.

---

## Chapter 19: Experiment 10: Verifying Multi-Pod Shared Read/Write

### 19.1 Pod A Writes to a File

    kubectl exec -n cephfs-demo cephfs-pod-a -- sh -c "echo 'written from pod A' > /data/a.txt"

Check:

    kubectl exec -n cephfs-demo cephfs-pod-a -- cat /data/a.txt

Expected result:

    written from pod A

---

### 19.2 Pod B Reads the File Written by Pod A

    kubectl exec -n cephfs-demo cephfs-pod-b -- cat /data/a.txt

Expected result:

    written from pod A

---

### 19.3 Pod B Writes to a File

    kubectl exec -n cephfs-demo cephfs-pod-b -- sh -c "echo 'written from pod B' > /data/b.txt"

Check:

    kubectl exec -n cephfs-demo cephfs-pod-b -- cat /data/b.txt

Expected result:

    written from pod B

---

### 19.4 Pod A Lists the Files in Pod B

    kubectl exec -n cephfs-demo cephfs-pod-a -- ls -l /data

    kubectl exec -n cephfs-demo cephfs-pod-a -- cat /data/b.txt

Expected result:

    written from pod B

---

### 19.5 Experimental Conclusion

This experiment demonstrates that:

- The same CephFS PVC can be mounted by multiple pods simultaneously.
- Data written by Pod A can be read by Pod B.
- This is a typical capability of the ReadWriteMany feature in Kubernetes.

---

## Chapter 20: Experiment 11: Verifying Data Persistence After Pod Deletion and Recreation

### 20.1 Deleting Pods

    kubectl delete pod -n cephfs-demo cephfs-pod-a
    kubectl delete pod -n cephfs-demo cephfs-pod-b

Confirm:

    kubectl get pod -n cephfs-demo

---

### 20.2 Recreating Pods

    kubectl apply -f pods-cephfs-rwx-demo.yaml

Wait for the pods to start running:

    kubectl get pod -n cephfs-demo -o wide

---

### 20.3 Verifying That Data Still Exists

    kubectl exec -n cephfs-demo cephfs-pod-a -- ls -l /data

    kubectl exec -n cephfs-demo cephfs-pod-a -- cat /data/a.txt

    kubectl exec -n cephfs-demo cephfs-pod-a -- cat /data/b.txt

Expected result:

    a.txt and b.txt still exist.
    The contents are correct.

Conclusion:

- Deleting a pod does not erase the data associated with its PVC.
- The CephFS Subvolume corresponding to the PVC remains intact.

---

## Chapter 21: Experiment 12: Verifying CephFS Subvolumes

### 21.1 Checking the SubvolumeGroup

Execute on the Ceph management node:

    ceph fs subvolumegroup ls cephfs

Expected result:

    csi

---

### 21.2 Checking Subvolumes

    ceph fs subvolume ls cephfs --group_name csi

Expected output similar to:

    csi-vol-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

---

### 21.3 Checking Subvolume Information

Replace <subvolume-name> with the appropriate value:

    ceph fs subvolume info cephfs <subvolume-name> --group_name csi

Key points to check:

- path
- bytes_quota
- mode
- uid
- gid
- created_at
- data_pool

Explanation:

- Behind a Kubernetes PVC, there is a corresponding CephFS Subvolume.
- Whether the subvolume is deleted when the PVC is deleted depends on the StorageClass recycling policy and the CSI behavior.

---

### 21.4 Checking CephFS Status

    ceph fs status
    ceph mds stat
    ceph -s

Confirm:

- The MDS is functioning normally.
- The PG is in an active+clean state.
- There are### Production Environment Restrictions on Deleting CephFS and Pools

In a production environment, it is strictly prohibited to delete CephFS and pools arbitrarily.

If you confirm that you no longer need the entire CephFS in an experimental environment, follow these steps:

1. Delete the CephFS:
   ```bash
   ceph fs fail cephfs
   ceph fs rm cephfs --yes-i-really-mean-it
   ```

2. Then delete the pool:
   ```bash
   ceph config set mon mon_allow_pool_delete true
   ceph osd pool rm cephfs_metadata cephfs_metadata --yes-i-really-really-mean-it
   ceph osd pool rm cephfs_data cephfs_data --yes-i-really-really-mean-it
   ceph config set mon mon_allow_pool_delete false
   ```

**High-Risk Warning:**
Deleting CephFS and pools will result in the loss of all file system data. This operation is strictly forbidden in a production environment. It is also recommended to confirm that no data remains before proceeding in an experimental environment.

---

## Section 24: Common Faults and Troubleshooting

### 24.1 Failure to Pull CephFS CSI Pod Images

**Phenomenon:**
- `ImagePullBackOff`
- `ErrImagePull`

**Inspection Steps:**
- `kubectl get pods -n ceph-csi`
- `kubectl describe pod -n ceph-csi <pod-name>`

**Troubleshooting Steps:**
- Check the image URL.
- Verify the domestic network connection.
- Verify the `imagePullSecrets`.
- Synchronize the images to a Harbor or Alibaba Cloud repository.
- Update the image URL in the YAML file.
- It is not recommended to directly modify the global configuration of `containerd`.

---

### 24.2 CephFS CSI Pod Not Running

**Inspection Steps:**
- `kubectl get pods -n ceph-csi -o wide`
- `kubectl describe pod -n ceph-csi <pod-name>`
- `kubectl logs -n ceph-csi <pod-name> -c csi-cephfsplugin`

**Common Causes:**
- RBAC permissions are missing.
- There is an error with the `ConfigMap`.
- The `Secret` configuration is incorrect.
- Image pull failed.
- The node has a stain that cannot be tolerated.
- Permissions in the kubelet plugin directory are abnormal.
- The CSI Driver is not registered.

**Check the CSI Driver:**
```bash
kubectl get csidriver
```
Ensure that `cephfs.csi.ceph.com` is listed.

---

### 24.3 PVC Remains Pending

**Inspection Steps:**
- `kubectl describe pvc -n cephfs-demo cephfs-demo-pvc`
- Pay special attention to the `Events`.

**Common Causes:**
- The `StorageClass` does not exist.
- The name of the provisioner is incorrect.
- The CSI Controller is not functioning properly.
- There is an error with the `Secret` name or namespace.
- The `clusterID` is incorrect.
- The `fsName` is incorrect.
- CephFS is not installed.
- The `SubvolumeGroup` does not exist.
- The Ceph user does not have sufficient permissions.
- Communication with the Ceph MON is interrupted.
- The MDS is not active.

**Troubleshooting Steps:**
- `kubectl get storageclass`
- `kubectl get pods -n ceph-csi`
- `kubectl logs -n ceph-csi <provisioner-pod> -c csi-cephfsplugin`
- `ceph -s`
- `ceph fs ls`
- `ceph fs status`
- `ceph auth get client.k8s-cephfs`
- `ceph fs subvolumegroup ls cephfs`

---

### 24.4 Pod Failed to Mount

**Inspection Steps:**
- `kubectl describe pod -n cephfs-demo cephfs-pod-a`
- Common error messages include:
  - `MountVolume.SetUp failed`
  - `permission denied`
  - `context deadline exceeded`
  - `failed to get secret`
  - `failed to connect to monitors`
  - `error mounting ceph filesystem`

**Troubleshooting Steps:**
- `kubectl get secret -n ceph-csi csi-cephfs-secret`
- `kubectl get configmap -n ceph-csi ceph-csi-config -o yaml`
- `kubectl logs -n ceph-csi <node-plugin-pod> -c csi-cephfsplugin`
- `ceph -s`
- `ceph fs status`

**On the node where the pod is running:**
- `modprobe ceph`
- `lsmod | grep ceph`
- `dmesg | tail -5## Chapter 25: Summary of Common Commands for CephFS CSI

### 25.1 On the Ceph Side

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

### 25.2 On the Kubernetes Side

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

### 25.4 Testing Read and Write Operations

    kubectl exec -n cephfs-demo cephfs-pod-a -- sh -c "echo hello > /data/hello.txt"
    kubectl exec -n cephfs-demo cephfs-pod-b -- cat /data/hello.txt
    kubectl exec -n cephfs-demo cephfs-pod-a -- df -h /data

---

## Chapter 26: Precautions for Production Environments

### 26.1 Do Not Use client.admin

In experiments, you can use the admin key to verify the connection.

For production, you must use a user with minimal permissions:

    client.k8s-cephfs

And limit its usage to specific CephFS instances.

---

### 26.2 Clearly Define the ReclaimPolicy

In production, it is essential to specify whether to:

- Delete or Retain the PVC and its underlying CephFS Subvolume after termination.

Deleting the PVC may result in the deletion of the associated Subvolume, making it suitable for temporary environments or data that can be discarded. Retaining the PVC ensures that both the PV and its data are preserved, which is crucial for critical business operations. In production scenarios involving shared files, user-uploaded data, or business attachment directories, caution should be exercised when using the Delete option.

---

### 26.3 Ensure High Availability of MDS

CephFS relies heavily on MDS. For production use, it is recommended to have:

- At least 1 active MDS
- At least 1 standby MDS

You can check their status using commands like `ceph fs status` and `ceph mds stat`. Without a standby MDS, a failure in one MDS could impact the availability of CephFS.

---

### 26.4 The Metadata Pool Is Crucial

The CephFS metadata pool stores information such as:

- Directory structures
- File names
- Inodes
- Permissions
- Metadata itself

Any issues with the metadata pool can significantly affect CephFS performance. Therefore, it is important to:

- Use a reliable replication strategy for the metadata pool
- Monitor the capacity of the metadata pool
- Check the status of the PGs within the metadata pool
- Avoid letting the metadata pool get close to its maximum capacity
- Pay special attention to MDS performance in scenarios involving many small files

---

### 26.5 Use a Fixed Version for Images

In production, it is not advisable to use the latest version of Ceph CSI or related components. Instead, it is recommended to:

- Fix the version of Ceph CSI and its sidecar components
- Mirror these images to an internal repository
- Manage the image versions using Git
- Verify any updates in a test cluster before applying them in production

---

### 26.6 Avoid Altering the Kubernetes Base Runtime

To ensure smooth operation of CSI images, it is not recommended to make arbitrary changes to the global containerd configuration. Instead, consider using:

- An internal Harbor for image storage
- Alibaba Cloud’s image repository
- ImagePullSecrets for secure image retrieval
- A unifiedBefore deployment, it is necessary to prepare the CephFS file system within Ceph, which includes the metadata pool, data pool, and MDS service. In production environments, there must be at least one active and one standby MDS instance. Next, create a CephX user for use with CephFS CSI and grant the minimum required permissions to the mon, mgr, mds, and osd roles for the specified CephFS.

On the Kubernetes side, deploy the CephFS CSI components, including the Controller Deployment and Node DaemonSet. Then create a ceph-csi-config ConfigMap to configure the clusterID and MON address; create a Secret to store the Ceph credentials; and create a StorageClass with the provisioner set to cephfs.csi.ceph.com, specifying parameters such as fsName, pool, subvolumeGroup, Secret, and reclaimPolicy.

After a business entity creates a ReadWriteMany Persistent Volume (PVC), CephFS CSI will create the corresponding subvolume within CephFS and generate the PV. When multiple Pods mount the same PVC, they can share access to the same directory for read and write operations.

When troubleshooting issues, it is essential to examine various layers of information, including PVC/PV/Pod events, CSI Controller logs, CSI Node logs, Secret settings, StorageClass configurations, CephFS status, MDS status, SubvolumeGroup details, CephX permission settings, and OSD/PG status.

In production environments, special attention should be paid to ensuring high availability of the MDS, the health of the metadata pool, the proper setting of the reclaimPolicy, maintaining minimum required permissions, using fixed image versions, implementing monitoring and alert mechanisms, optimizing performance for a large number of small files, and establishing effective data backup strategies.