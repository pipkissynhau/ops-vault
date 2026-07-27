# 05-StorageClass Basic Installation: NFS Dynamic Provisioning and Default Storage Class

Recommended Path:

    04-Kubernetes/08-Operations/02-Cluster Basic Components Installation/05-StorageClass Basic Installation: NFS Dynamic Provisioning and Default Storage Class.md

Tags:

    #Kubernetes
    #StorageClass
    #NFS
    #PV
    #PVC
    #Dynamic Provisioning
    #nfs-subdir-external-provisioner
    #Cluster Basic Components
    #Persistent Storage

---

## I. Document Description

This document records the installation method of NFS-based dynamic storage classes in a Kubernetes cluster.

Components Used:

    NFS Server
    nfs-subdir-external-provisioner
    StorageClass
    PVC
    Pod Mount Verification

Achievement:

    1. Users create a PVC.
    2. The StorageClass automatically triggers the NFS dynamic provider.
    3. A PV is created automatically.
    4. A subdirectory is automatically created within the NFS shared directory.
    5. The Pod mounts the PVC.
    6. Data written by the Pod is stored in the NFS backend directory.

Applicable Scenarios:

    1. Self-built Kubernetes clusters using kubeadm.
    2. Private environments.
    3. Experimental environments.
    4. Small to medium-sized non-core production environments.
    5. Scenarios where rapid provision of ReadWriteMany storage capacity is required.

Notes:

    NFS is suitable for basic shared storage verification and persistent storage for some non-core services.
    High-performance databases, highly consistent storage, and core production data are not recommended to rely solely on a single-node NFS system.
    In production environments, solutions such as Longhorn, Ceph CSI, cloud disk CSI, and commercial storage CSI should be further evaluated.

---

## II. Component Relationship Description

The dynamic storage provision relationship in Kubernetes is as follows:

    Pod
     |
     v
    PVC
     |
     v
    StorageClass
     |
     v
    nfs-subdir-external-provisioner
     |
     v
    NFS Server
     |
     v
    /data/nfs/k8s/<namespace-pvc-pv-subdirectory>

Key Components:

| Component | Function |
|-----------|-----------|
| NFS Server | Provides the backend shared storage directory. |
| nfs-common | Client tools required for Kubernetes nodes to mount NFS. |
| nfs-subdir-external-provisioner | Dynamically creates NFS subdirectories and PVs. |
| StorageClass | The entry point for dynamic storage provision in Kubernetes. |
| PVC | Users apply for storage resources. |
| PV | The actually bound persistent volume. |
| Pod | Mounts the PVC to use the stored data. |

---

## III. Planning Information

### 3.1 NFS Planning

| Item        | Planning     |
|-------------|--------------|
| NFS Server   | ops-server     |
| NFS Server IP | 10.0.0.10    |
| NFS Shared Directory | /data/nfs/k8s    |
| Allowed Access Range | 10.0.0.0/24    |
| StorageClass Name | nfs-client     |
| Set as Default StorageClass | Yes           |
| Recycling Policy | Delete         |
| Volume Binding Mode | Immediate       |

---

### 3.2 Node Planning

It is assumed in this document that:

    ops-server        10.0.0.10    NFS Server
    k8s-master-01     10.0.0.20
    k8s-master-02     10.0.0.21
    k8s-master-03     10.0.0.22
    k8s-worker-01     10.0.0.23
    k8s-worker-02     10.0.0.24
    k8s-worker-03     10.0.0.25

Note:

    The NFS Server can be deployed on the ops-server.
    Alternatively, it can be hosted on a dedicated storage server.
    It is not recommended to deploy the NFS Server on ordinary business Worker nodes.

---

## IV. Pre-deployment Checks

### 4.1 Check the Status of the Kubernetes Cluster

Execute on k8s-master-01:

    kubectl get nodes -o wide

Requirements:

    Master Ready
    Worker Ready

---

### 4.2 Check Helm

Execute:

    helm version

---

### 4.3 Check Network Connectivity to the NFS Server

Execute on a Kubernetes node:

    ping -c 3 10.0.0.10

Requirement:

    Kubernetes nodes should be able to access the NFS Server.

---

## V. Install the NFS Server

The```markdown
mkdir -p /root/k8s-yaml/nfs-subdir-external-provisioner

cd /root/k8s-yaml/nfs-subdir-external-provisioner

Create the values file:

    cat <<EOF > values-nfs-subdir.yaml
    replicaCount: 1

    nfs:
      server: 10.0.0.10
      path: /data/nfs/k8s

    storageClass:
      create: true
      name: nfs-client
      defaultClass: true
      archiveOnDelete: true
      reclaimPolicy: Delete
      accessModes: ReadWriteMany
      pathPattern: "\${.PVC.namespace}-\${.PVC.name}-\${.PV.name}"

    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 256Mi
    EOF

Explanation:

    nfs.server
        The address of the NFS Server.

    nfs.path
        The shared path exported by the NFS Server.

    storageClass.name
        The name of the StorageClass.

    storageClass.defaultClass
        Whether it is set as the default StorageClass.

    archiveOnDelete
        Whether to archive the backend directory after the PVC is deleted.

    reclaimPolicy: Delete
        When the PVC is deleted, the PV will also be deleted.

    accessModes: ReadWriteMany
        NFS supports multiple Pods reading and writing from the same volume.
```

---

### 8.3 Install the Provisioner

Create a namespace:

    kubectl create namespace storage-system

Install:

    helm install nfs-subdir-external-provisioner \
      nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
      -n storage-system \
      -f values-nfs-subdir.yaml

View the Helm Release:

    helm list -n storage-system

View Pods:

    kubectl -n storage-system get pods -o wide

View StorageClasses:

    kubectl get storageclass

Expected output:

    nfs-client

If it is set as the default StorageClass, you should see:

    nfs-client (default)

---

## Section 9: Verify Dynamic PVCs

### 9.1 Create a Test PVC

Create a test file:

    cat <<EOF > pvc-nfs-test.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nfs-test-pvc
      namespace: default
    spec:
      accessModes:
      - ReadWriteMany
      resources:
        requests:
          storage: 1Gi
      storageClassName: nfs-client
    EOF

Apply:

    kubectl apply -f pvc-nfs-test.yaml

View the PVC:

    kubectl get pvc

Expected output:

    STATUS: Bound

View the PV:

    kubectl get pv

View detailed information about the PVC:

    kubectl describe pvc nfs-test-pvc
```

---

### 9.2 Check the NFS Backend Directory

Execute on the ops-server:

    find /data/nfs/k8s -maxdepth 2 -type d

You should see the subdirectories automatically created by the provisioner.
```markdown
## Section 10: Create Pods to Mount PVCs and Verify

### 10.1 Create a Test Pod

Create a Pod configuration file:

    cat <<EOF > pod-nfs-test.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nfs-test-pod
      namespace: default
    spec:
      containers:
      - name: busybox
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "hello from nfs pvc" > /data/hello.txt
          sleep 3600
        volumeMounts:
        - name: nfs-data
          mountPath: /data
      volumes:
      - name: nfs-data
        persistentVolumeClaim:
          claimName: nfs-test-pvc
    EOF

Apply the configuration:

    kubectl apply -f pod-nfs-test.yaml

View the Pod:

    kubectl get pod nfs-test-pod -o wide
```markdown

### 10.2 Verify Writing Inside the Pod

Execute the following command inside the Pod:

    kubectl exec -it nfs-test-pod -- cat /data/hello.txt

Expected output:

    hello from nfs pvc

---

### 10.3 Verify the File on the NFS Server

Execute on the ops-server:

    find /data/nfs/k8s -name hello.txt -type f

View the content:

    cat $(find /data/nfs/k8s -name hello.txt -type f | head -n 1)

If you see:

    hello from nfs pvc

It indicates that---

### 2. The provisioner Pod is not running.

### 3. The NFS Server is unreachable.

### 4. There is an error in the NFS path configuration.

### 5. The Kubernetes node does not have nfs-common installed.

### 6. There are abnormalities in RBAC or provisioner configuration.

---

### 13.2 Pod cannot mount a PVC

View the Pod:

    kubectl describe pod <pod-name>

Pay special attention to the Events.

Common error messages:

    mount.nfs: access denied by server
    mount.nfs: Connection timed out
    mount.nfs: No such file or directory
    MountVolume.SetUp failed

Troubleshooting:

    showmount -e 10.0.0.10

    sudo mount -t nfs 10.0.0.10:/data/nfs/k8s /mnt/nfs-test

Common causes:

    1. The NFS Server has not authorized the current network segment.
    2. The firewall on the NFS Server is not configured to allow access.
    3. The NFS path does not exist.
    4. nfs-common is not installed.
    5. The permissions on the NFS directory are insufficient.

---

### 13.3 The default StorageClass is not taking effect

View the StorageClass:

    kubectl get storageclass

If it does not show:

    (default)

You can manually set the default StorageClass:

    kubectl patch storageclass nfs-client \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

To revert to the default setting:

    kubectl patch storageclass nfs-client \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

---

## Section Fourteen: Uninstallation

View the PVCs:

    kubectl get pvc -A

Uninstall them after confirming that no services are using them:

    helm uninstall nfs-subdir-external-provisioner -n storage-system

View the StorageClass:

    kubectl get storageclass

If it is confirmed that it is no longer needed:

    kubectl delete storageclass nfs-client

Delete the namespace:

    kubectl delete namespace storage-system

Note:

    Uninstalling with Helm will not automatically clean up all historical data on the NFS Server.
    The NFS backend directories must be manually checked and cleaned up as appropriate.

---

## Section Fifteen: Post-Installation Verification Checklist

After installation, perform the following checks:

    showmount -e 10.0.0.10

    kubectl -n storage-system get pods -o wide

    kubectl get storageclass

    kubectl get pvc

    kubectl get pv

    kubectl get pod nfs-test-pod -o wide

    find /data/nfs/k8s -maxdepth 2 -type d

The following should be true:

    1. The NFS Server is exporting /data/nfs/k8s correctly.
    2. All Kubernetes nodes have nfs-common installed.
    3. The nfs-subdir-external-provisioner Pod is running.
    4. The StorageClass nfs-client exists.
    5. nfs-client is set as the default StorageClass.
    6. PVCs can be automatically bound.
    7. PVs can be created automatically.
    8. Pods can mount PVCs successfully.
    9. Files written by Pods in the PVCs can be accessed on the NFS Server.

---

## Section Sixteen: Summary

This document describes how to set up a dynamic storage class based on NFS in Kubernetes.

Key steps include:

    1. Installing the NFS Server.
    2. Configuring the NFS shared directory.
       3. Ensuring that all Kubernetes nodes have nfs-common installed.
       4. Using Helm to install the nfs-subdir-external-provisioner.
    5. Creating the StorageClass nfs-client.
    6. Setting it as the default StorageClass.
    7. Testing PVC creation and automatic binding.
    8. Verifying Pod mounting and read/write capabilities.
    9. Checking the NFS backend directory.
    10. Troubleshooting issues such as Pending PVCs, failed pod mounts, and NFS permission problems.

Production recommendations:

    1. NFS is suitable for basic shared storage and some non-core business scenarios.
    2. For core database applications, it is not recommended to rely solely on a single-node NFS setup.
    3. In a production environment, ensure that the NFS Server has high availability.
    4. Important data should have backup strategies in place.
    5. Always confirm whether backend data needs to be retained before deleting PVCs.

Future recommendations include:

    06-Longhorn Installation: Distributed Block Storage, StorageClass, and PVC Verification.md