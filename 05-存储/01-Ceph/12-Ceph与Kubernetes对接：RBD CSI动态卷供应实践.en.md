# Integrating Ceph with Kubernetes: Practical Approaches to RBD CSI Dynamic Volume Provisioning

Recommended Path: 05-Storage/01-Ceph/12-Integrating Ceph with Kubernetes: Practical Approaches to RBD CSI Dynamic Volume Provisioning.md

Tags: #Ceph #Kubernetes #RBD #CSI #StorageClass #PVC #PV #Dynamic Volume Provisioning #ReadWriteOnce #Cloud-Native Storage #SRE #Advanced SRE

---

## I. Document Overview

This article is the twelfth in the Ceph Advanced SRE storage series, focusing on practical methods for integrating Ceph RBD with Kubernetes.

Previous topics include:

- Deploying a Ceph cluster
- Managing OSDs
- Creating pools and PGs
- Configuring CRUSH rules
- Practicing RBD block storage
- Using CephFS for file storage
- Implementing RGW object storage

This article delves into the integration of Ceph and Kubernetes:

    - Using Ceph as an external persistent storage solution in Kubernetes

The main focus of this article is:

    - Providing dynamic block storage volumes through Ceph RBD CSI for Kubernetes

Typical objectives include:

- A Pod requesting a PVC
      |
      v
    A StorageClass leveraging RBD CSI
      |
      v
    Automatic creation of an RBD Image in Ceph
      |
      v
    The Pod mounting the volume as a block device file system
      |
      v
    Applications writing persistent data

The most common usage pattern for RBD in Kubernetes is:

    ReadWriteOnce

In other words, an RBD volume is typically mounted exclusively by one or more Pods on a single node. It is not suitable for multiple nodes sharing the same file system simultaneously.

For scenarios where multiple Pods and nodes need to share directories, CephFS CSI should be used instead:

---

## II. Experimental Objectives

After completing this article, you should be able to:

1. Understand why Kubernetes requires CSI.
2. Comprehend the component structure of RBD CSI.
3. Grasp the relationships between StorageClass, PVC, PV, Pod, and Ceph RBD Image.
4. Deploy Ceph RBD CSI in Kubernetes.
5. Create a dedicated Ceph RBD pool.
6. Set up a user with minimum permissions for using Ceph CSI.
7. Generate a Kubernetes Secret.
8. Define a RBD StorageClass.
9. Create a PVC and dynamically generate a PV.
10. Verify the automatic creation of the RBD Image in Ceph.
11. Create a Pod that mounts the PVC.
12. Write data and confirm its persistence.
13. Delete the Pod and re-mount it to verify that the data remains.
14. Expand the PVC and check if the file system scales accordingly.
15. Troubleshoot issues such as pending PVCs, failed Pod mounting, RBD Image creation errors, Secret failures, or CSI Pod anomalies.
16. Understand the security, permissioning, imaging, monitoring, and recycling strategies for using RBD CSI in production environments.

---

## III. Experimental Environment

### 3.1 Ceph Cluster

The Ceph cluster is deployed independently on the following nodes:

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Failover, optional |

The Ceph cluster must meet the following requirements:

- The MON quorum is functioning correctly.
- The MGR is operational.
- All OSDs are up and running.
- The PGs are active and clean.
- The `ceph -s` command should display HEALTH_OK or a tolerable HEALTH_WARN status.

---

### 3.2 Kubernetes Cluster

The Kubernetes cluster is deployed independently on the following nodes:

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

The operating environment includes:

- A `kubeadm` cluster
- The `containerd` runtime
- Calico CNI for network communication
- Ubuntu Server 22.04.5 LTS

Note:

- The Ceph nodes and the Kubernetes nodes are kept separate.
- The Kubernetes nodes only use RBD volumes as external storage.
- No Ceph OSDs### Requirements:

    OSDs should be up and running.
    The PGs must be active and in a clean state.
    There should be sufficient capacity.
    No nearfull or full status should be reported.
    No unexplained HEALTH_ERR messages should exist.

---

### 7.2 Checking the Kubernetes Cluster

Execute the following command on k8s-master:

    kubectl get nodes -o wide

Expected output:

    k8s-master    Ready
    k8s-worker01  Ready
    k8s-worker02  Ready

Check the system Pods by executing:

    kubectl get pods -A

Requirements:

    CoreDNS should be functioning normally.
    CNI should be working correctly.
    There should be no a large number of abnormal Pods in the kube-system namespace.

---

### 7.3 Checking the Network Connection Between Kubernetes Nodes and Ceph MON

Execute the following commands on each Kubernetes node:

    ping -c 3 10.0.0.31
    ping -c 3 10.0.0.32
    ping -c 3 10.0.0.33

Check the MON ports by executing:

    nc -vz 10.0.0.31 3300
    nc -vz 10.0.0.31 6789
    nc -vz 10.0.0.32 3300
    nc -vz 10.0.0.32 6789
    nc -vz 10.0.0.33 3300
    nc -vz 10.0.0.33 6789

If nc is not available on Ubuntu, install it using:

    apt install -y netcat-openbsd

---

### 7.4 Checking the Kubernetes Node Kernel Modules

RBD depends on the Linux kernel's rbd module.

Execute the following command on each Kubernetes node:

    modprobe rbd
    lsmod | grep rbd

Expected output:

    rbd

If no output is displayed, check whether the kernel module is available.

---

### 7.5 Checking kubelet and containerd

Execute the following commands on each Kubernetes node:

    systemctl status kubelet
    systemctl status containerd

Requirements:

    kubelet and containerd should be running normally.

Note:

    It is not recommended to modify the containerd configuration arbitrarily for Ceph CSI purposes.
    If image acceleration is required, it is advisable to use private image repositories, image synchronization, and the imagePullPolicy setting instead.

---

## Section 8: Experimental Task List

| Experiment | Objective | Risk Level |
|---------------|------------|-------------|
| Experiment 1 | Create a dedicated RBD CSI pool | Medium |
| Experiment 2 | Create a Ceph CSI user with minimal permissions | Medium |
| Experiment 3 | Prepare Kubernetes namespaces | Low |
| Experiment 4 | Deploy Ceph RBD CSI | Medium |
| Experiment 5 | Create a Secret | Medium |
| Experiment 6 | Create a StorageClass | Medium |
| Experiment 7 | Create a PVC and dynamically generate PVs | Medium |
| Experiment 8 | Create Pods and mount them with PVCs | Medium |
| Experiment 9 | Verify data persistence | Low |
| Experiment 10 | Verify automatically created RBD images in Ceph | Low |
| Experiment 11 | Expand PVCs | Medium |
| Experiment 12 | Clean up test resources | High |
| Experiment 13 | Troubleshoot common issues | Medium to High |

High-risk warnings:

    Deleting a PVC may result in the deletion of the underlying RBD image, depending on the reclaimPolicy setting.
    Deleting a StorageClass will not remove existing PVs but will affect subsequent dynamic provisioning.
    Deleting a Secret can cause failures when creating or mounting new volumes.
    In a production environment, clear recovery and data retention policies must be established.

---

## Section 9: Experiment 1: Creating a Dedicated RBD CSI Pool

### 9.1 Creating the Pool

Execute the following command on the Ceph management node:

    ceph osd pool create k8s-rbd 64

Explanation:

    k8s-rbd is a dedicated pool for Kubernetes RBD CSI.
    In small experimental environments, 64 PGs are used.
    In production environments, the number of PGs should be planned based on the number of OSDs, the number of pools, and the autoscaler settings.

---

### 9.2 Enabling the RBD Application

Execute the following command:

    ceph osd pool application enable k8s-rbd rbd

---

### 9.3 Setting the Number of Replicas

Execute the following commands to set the number of replicas:

    ceph osd pool    ImagePullBackOff
    ErrImagePull

---

### 13.3 Creating a ceph-csi-config ConfigMap

Replace the following <cluster-id> with the ceph fsid.

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

Application:

    kubectl apply -f csi-config-map.yaml

View:

    kubectl get configmap -n ceph-csi ceph-csi-config -o yaml

---

### 13.4 Creating a ceph-config ConfigMap

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

Application:

    kubectl apply -f ceph-config-map.yaml

---

### 13.5 Deploying the RBD CSI Components

In actual production, it is recommended to obtain the corresponding version YAML from the official ceph-csi release and replace the image addresses accordingly.

Typical resources include:

    csi-rbdplugin-provisioner Deployment
    csi-rbdplugin DaemonSet
    ServiceAccount
    ClusterRole
    ClusterRoleBinding
    Role
    RoleBinding
    CSIDriver

View the deployed resources:

    kubectl get pods -n ceph-csi -o wide
    kubectl get deploy -n ceph-csi
    kubectl get ds -n ceph-csi
    kubectl get csidriver

Expected results:

    csi-rbdplugin-provisioner should be Running normally.
    The csi-rbdplugin DaemonSet should be Running normally on each K8s node.
    The CSIDriver should include rbd.csi.ceph.com in its list.

---

### 13.6 Post-Deployment Checks

View the Pods:

    kubectl get pods -n ceph-csi -o wide

View the DaemonSet:

    kubectl get ds -n ceph-csi

Expected results:

    DESIRED should equal READY.

Check the CSI Driver:

    kubectl get csidriver

Expected output should include:

    rbd.csi.ceph.com

If any Pods are not functioning properly:

    kubectl describe pod -n ceph-csi <pod-name>
    kubectl logs -n ceph-csi <pod-name> -c csi-rbdplugin

---

## Chapter Fourteen: Experiment Five: Creating a Kubernetes Secret

### 14.1 Creating the Secret YAML

Replace <user-key> with the key for client.k8s-rbd.

Note:

    The userID does not include the "client." prefix.
    The userID corresponding to client.k8s-rbd is k8s-rbd.

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

Application:

    kubectl apply -f csi-rbd-secret.yaml

View:

    kubectl get secret -n ceph-csi csi-rbd-secret

---

### 14.2 Explanation of Secret Naming

The StorageClass will reference this Secret in the following fields:

    csi.storage.k8s.io/provisioner-secret-name
    csi.storage.k8s.io/provisioner-secret-namespace
    csi.storage.k8s.io/controller-expand-secret-name
    csi.storage.k8s.io-controller-expand-secret-namespace
    csi.storage.k8s.io/node-stage-secret-name
    csi.storage.k8s.io/node-stage-secret-namespace

If the Secret name or Namespace is incorrect, it may result in:

    Failed PVC creation
    Failed Pod mounting
    CSI permission errors

---

## Chapter Fifteen: Experiment Six: Creating an RBD StorageClass

### 15.1 StorageClass YAML

Replace <cluster-id> with the ceph fsid.

    cat > storageclass-rbd.yaml <<'EOF'
    apiVersion:reclaimPolicy: Retain

Production Recommendations:

It is recommended to use the Delete option with caution for databases and critical business data. Before proceeding with production, it is essential to determine the data retention policy after deleting a PVC.

---

## Section Sixteen: Experiment Seven: Creating a PVC and Dynamically Generating a PV

### 16.1 Creating a PVC

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

Application:

    kubectl apply -f pvc-rbd-demo.yaml

---

### 16.2 Viewing the PVC

    kubectl get pvc -n storage-demo

Expected Output:

    NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
    rbd-demo-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   5Gi        RWO            ceph-rbd

---

### 16.3 Viewing the PV

    kubectl get pv

For more details:

    kubectl describe pvc -n storage-demo rbd-demo-pvc
    kubectl describe pv <pv-name>

Key Points to Check:

- StorageClass
- CSI Driver
- VolumeHandle
- Events
- Capacity
- AccessModes

---

## Section Seventeen: Experiment Eight: Verifying the Automatically Created RBD Image in Ceph

### 17.1 Viewing the RBD Image

Execute on the Ceph management node:

    rbd ls -p k8s-rbd

Expected Output:

    csi-vol-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

For more details:

    rbd info k8s-rbd/<image-name>

Explanation:

After a Kubernetes PVC is dynamically provisioned, the RBD CSI will create the corresponding RBD Image in the k8s-rbd Pool.

---

### 17.2 Checking the RBD Status

    rbd status k8s-rbd/<image-name>

If the PVC has not been mounted by a Pod, there may be no watcher information.

After the Pod mounts the volume, you should be able to see the watcher details.

---

## Section Eighteen: Experiment Nine: Creating a Pod to Mount a PVC

### 18.1 Creating a Test Pod

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

Explanation:

If the busybox image is not available, you can replace it with an existing busybox image from your Harbor or Alibaba Cloud repository. It is recommended to use controllable image sources in both production and experimental scenarios to avoid issues with ImagePullBackOff.

Application:

    kubectl apply -f pod-rbd-demo.yaml

Verification:

    kubectl get pod -n storage-demo -o wide

Expected Output:

    rbd-demo-pod Running

---

### 18.2 Checking Pod Events

    kubectl describe pod -n storage-demo rbd-demo-pod

Key Points to Check:

- Whether the Pod was successfully scheduled.
- Whether it successfully mounted the PVC.
- Any FailedMount events.
- Any ImagePullBackOff events.
- Any MountVolume.MountDevice failed events.

---

### 18.3 Writing Data Inside the Pod

    kubectl exec -n storage-demo rbd-demo-pod -- sh -c "echo 'hello rbd csi' > /data/hello.txt"

Reading the Data:

    kubectl exec -n storage-demo rbd-demo-pod -- cat /data/hello.txt

Expected Output:

    hello rbd csi

Checking the File:

    kubectl exec -n storage-demo rbd-demo-pod -- ls -l /data

---

## Section Nineteen: Experiment Ten: Verifying Data Persistence

### 19.1 Deleting the Pod

    kubectl delete pod -n storage-demo rbd-demo-pod

Confirmation of Pod Deletion:

    kubectl get pod -n storage-demo

---

### 19.2 Recreating the Pod

    kubectlIf reclaimPolicy is set to Delete, the associated CSI-vol Image should be deleted. If it still exists, it is necessary to check the PV status and CSI logs.

---

### 21.4 Deleting StorageClass

    kubectl delete storageclass ceph-rbd

---

### 21.5 Deleting Secret

    kubectl delete secret -n ceph-csi csi-rbd-secret

---

### 21.6 Deleting the Test Namespace

    kubectl delete namespace storage-demo

If Ceph CSI is no longer being tested, the ceph-csi Namespace can also be deleted, but this will remove all CSI components:

    kubectl delete namespace ceph-csi

Do not delete the ceph-csi Namespace in a production environment without caution.

---

### 21.7 Whether to Delete the Ceph Pool

In a test environment, if it is confirmed that the pool is no longer needed:

    ceph osd pool ls
    rbd ls -p k8s-rbd

To proceed with deletion:

    ceph config set mon mon_allow_pool_delete true

To delete the pool:

    ceph osd pool rm k8s-rbd k8s-rbd --yes-i-really-really-mean-it

To disable deletion:

    ceph config set mon mon_allow_pool_delete false

Note for production:

    Deleting a pool will also remove all RBD Images within it.
    Such deletions are strictly prohibited in a production environment.

---

## Chapter Twenty-Two: Common Faults and Troubleshooting

### 22.1 Failure to Pull CSI Pod Images

Symptoms:

    ImagePullBackOff
    ErrImagePull

Inspection Steps:

    kubectl get pods -n ceph-csi
    kubectl describe pod -n ceph-csi <pod-name>

Possible Solutions:

- Check the image URL.
- Verify the domestic network connectivity.
- Ensure that the imagePullSecrets are correct.
- Synchronize the images to a Harbor or Alibaba Cloud repository.
- Update the image URL in the YAML configuration file.
- It is not recommended to directly modify the containerd global settings.

---

### 22.2 CSI Pod Not Running

Inspection Steps:

    kubectl get pods -n ceph-csi -o wide
    kubectl describe pod -n ceph-csi <pod-name>
    kubectl logs -n ceph-csi <pod-name> -c csi-rbdplugin

Common Causes:

- RBAC permissions are missing.
- There is an error with the ConfigMap.
- The Secret is incorrect.
- Image pull failed.
- The node has stains that are not tolerated.
- Permissions in the kubelet plugin directory are incorrect.

---

### 22.3 PVC Remaining in a Pending State

Inspection Steps:

    kubectl describe pvc -n storage-demo rbd-demo-pvc
    Pay special attention to the Events section.

Possible Causes:

- The StorageClass does not exist.
- The provisioner name is incorrect.
- The CSI Controller is not functioning properly.
- The Secret name or Namespace is wrong.
- The clusterID is incorrect.
- The pool does not exist.
- Ceph user permissions are insufficient.
- Communication with the Ceph MON is lost.

Troubleshooting Steps:

    kubectl get storageclass
    kubectl get pods -n ceph-csi
    kubectl logs -n ceph-csi <provisioner-pod>
    ceph -s
    ceph auth get client.k8s-rbd
    ceph osd pool ls

---

### 22.4 Pod Mount Failure: FailedMount

Inspection Steps:

    kubectl describe pod -n storage-demo rbd-demo-pod
    Common error messages include:
        MountVolume.MountDevice failed
        rbd image not found
        permission denied
        context deadline exceeded
        failed to get secret
        failed to connect to monitors

Troubleshooting Steps:

    kubectl get secret -n ceph-csi csi-rbd-secret
    kubectl get configmap -n ceph-csi ceph-csi-config -o yaml
    kubectl logs -n ceph-csi <node-plugin-pod> -c csi-rbdplugin
    ceph -s
    rbd ls -p k8s-rbd

On the node where the Pod is running:

    modprobe rbd
    lsmod | grep rbd
    dmesg | tail -50

---

### 22.5 Ceph Permission Issues

Symptoms:

    permission denied
    operation not permitted

Check the Ceph user permissions:

    ceph auth get client.k8s-rbd

Verify the configured caps:

    mon 'profile rbd'
    mgr 'profile rbd pool=k8s-rbd'
    osd 'profile rbd pool=k8s-rbd'

If the pool name is```bash
kubectl get pods -n ceph-csi -o wide
kubectl get storageclass
kubectl get pvc -n storage-demo
kubectl get pv
kubectl describe pvc -n storage-demo rbd-demo-pvc
kubectl describe pod -n storage-demo rbd-demo-pod
kubectl get volumeattachment
kubectl get csidriver
```Ceph RBD for Kubernetes Documentation:

    https://docs.ceph.com/en/latest/rbd/rbd-kubernetes/

Kubernetes CSI Documentation:

    https://kubernetes.io/docs/concepts/storage/volumes/#csi

Kubernetes StorageClass Documentation:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes Persistent Volumes Documentation:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/