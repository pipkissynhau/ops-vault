# 05-StorageClass Basic Installation: NFS Dynamic Provisioning and Default Storage Class

Recommended path:

    04-Kubernetes/08-Operations/02-Cluster Basic Component Installation/05-StorageClass Basic Installation: NFS Dynamic Provisioning and Default Storage Class.md

Tags:

    #Kubernetes
    #StorageClass
    #NFS
    #PV
    #PVC
    #DynamicSupply
    #nfs-subdir-external-provisioner
    #ClusterBasicComponents
    #EnduringStorage

---

## I. Document Description

This document records the installation method of NFS-based dynamic StorageClass in Kubernetes clusters.

This document uses:

    NFS Server
    nfs-subdir-external-provisioner
    StorageClass
    PVC
    Pod mount verification

Achieved effects:

    1. User creates PVC
    2. StorageClass automatically triggers NFS dynamic provisioner
    3. Automatically creates PV
    4. Automatically creates subdirectory in NFS shared directory
    5. Pod mounts PVC
    6. Data written by Pod falls into NFS backend directory

Applicable scenarios:

    1. kubeadm self-built cluster
    2. Private environment
    3. Experimental environment
    4. Small-to-medium scale non-core production environment
    5. Scenarios requiring rapid provision of ReadWriteMany storage capability

Notes:

    NFS is suitable for basic shared storage verification and some non-core business persistence.
    High-performance databases, strong consistency storage, and core production data should not rely simply on single-node NFS.
    Production environments should further evaluate solutions like Longhorn, Ceph CSI, cloud disk CSI, and commercial storage CSI.

---

## II. Component Relationship Description

Kubernetes dynamic storage provisioning relationship:

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
    /data/nfs/k8s/<namespace-pvc-pv subdirectory>

Core objects:

| Object | Purpose |
|---|---|
| NFS Server | Provides backend shared storage directory |
| nfs-common | Client tools required for Kubernetes nodes to mount NFS |
| nfs-subdir-external-provisioner | Dynamically creates NFS subdirectories and PV |
| StorageClass | Kubernetes dynamic provisioning entry point |
| PVC | User storage request |
| PV | Actually bound persistent volume |
| Pod | Mounts PVC to use storage |

---

## III. Planning Information

### 3.1 NFS Planning

| Item | Planning |
|---|---|
| NFS Server | ops-server |
| NFS Server IP | 10.0.0.10 |
| NFS Shared Directory | /data/nfs/k8s |
| Allowed Access Network Segment | 10.0.0.0/24 |
| StorageClass Name | nfs-client |
| Is Set as Default StorageClass | Yes |
| Recycling Policy | Delete |
| Volume Binding Mode | Immediate |

---

### 3.2 Node Planning

This document assumes:

    ops-server        10.0.0.10    NFS Server
    k8s-master-01     10.0.0.20
    k8s-master-02     10.0.0.21
    k8s-master-03     10.0.0.22
    k8s-worker-01     10.0.0.23
    k8s-worker-02     10.0.0.24
    k8s-worker-03     10.0.0.25

Notes:

    NFS Server can be deployed on ops-server.
    It can also use an independent storage server.
    It is not recommended to deploy NFS Server arbitrarily on ordinary business Worker nodes.

---

## IV. Pre-Deployment Checks

### 4.1 Check Kubernetes Cluster Status

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

### 4.3 Check NFS Server Network Connectivity

Execute on Kubernetes nodes:

    ping -c 3 10.0.0.10

Requirements:

    Kubernetes nodes can access NFS Server.

---

## V. Install NFS Server

The following operations are executed on ops-server.

---

### 5.1 Install NFS Server

Execute:

    sudo apt update

    sudo apt install -y nfs-kernel-server

Start and set to boot:

    sudo systemctl enable --now nfs-server

Check status:

    systemctl status nfs-server --no-pager

---

### 5.2 Create NFS Shared Directory

Create directory:

    sudo mkdir -p /data/nfs/k8s

Set permissions:

    sudo chown -R nobody:nogroup /data/nfs/k8s

    sudo chmod 0777 /data/nfs/k8s

Notes:

    0777 is suitable for experimental environments and internal network basic verification.
    Production environments should tighten permissions based on business UID/GID, access boundaries, and security requirements.
    If permissions are too strict, Pods may encounter issues writing to PVCs.

---

### 5.3 Configure NFS Export Directory

Backup configuration:

    sudo cp /etc/exports /etc/exports.bak.$(date +%F-%H%M%S)

Write export configuration:

    cat <<EOF | sudo tee -a /etc/exports

    /data/nfs/k8s 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)
    EOF

Load configuration:

    sudo exportfs -rav

View exports:

sudo exportfs -v

Check ports:

    sudo ss -lntup | grep -E "2049|111"

Explanation:

    rw
        Allows read and write.

    sync
        Synchronous writes for better data consistency.

    no_subtree_check
        Avoids issues caused by subtree checks.

    no_root_squash
        Client root user is not mapped to nobody.
        Convenient for experimental environments, but requires careful evaluation for production.

---

### 5.4 Verify NFS Server

Execute on ops-server:

    showmount -e 127.0.0.1

Expected output:

    /data/nfs/k8s 10.0.0.0/24

---

## Six. Install NFS Client on All Kubernetes Nodes

The following operations are performed on all Master and Worker nodes.

Nodes include:

    k8s-master-01
    k8s-master-02
    k8s-master-03
    k8s-worker-01
    k8s-worker-02
    k8s-worker-03

Install:

    sudo apt update

    sudo apt install -y nfs-common

Check NFS Server exports:

    showmount -e 10.0.0.10

Expected output:

    /data/nfs/k8s 10.0.0.0/24

---

## Seven. Manual Mount Test

Execute on any Kubernetes node, for example k8s-worker-01.

Create test mount directory:

    sudo mkdir -p /mnt/nfs-test

Mount:

    sudo mount -t nfs 10.0.0.10:/data/nfs/k8s /mnt/nfs-test

Write test file:

    echo "nfs test from $(hostname)" | sudo tee /mnt/nfs-test/test-$(hostname).txt

View:

    ls -lh /mnt/nfs-test/

Unmount:

    sudo umount /mnt/nfs-test

Check on ops-server:

    ls -lh /data/nfs/k8s/

If you can see the test file, it indicates normal NFS Server and Kubernetes node mounting.

Clean up test files:

    sudo rm -f /data/nfs/k8s/test-*.txt

---

## Eight. Install nfs-subdir-external-provisioner

The following operations are performed on k8s-master-01.

---

### 8.1 Add Helm Repository

Add repository:

    helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

Update repository:

    helm repo update

Check:

    helm search repo nfs-subdir-external-provisioner

---

### 8.2 Create values File

Create directory:

    mkdir -p /root/k8s-yaml/nfs-subdir-external-provisioner

    cd /root/k8s-yaml/nfs-subdir-external-provisioner

Create values file:

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
        NFS Server address.

    nfs.path
        Shared path exported by NFS Server.

    storageClass.name
        StorageClass name.

    storageClass.defaultClass
        Whether to set as default StorageClass.

    archiveOnDelete
        Whether to archive and retain backend directories after PVC deletion.

    reclaimPolicy: Delete
        PV will also be deleted after PVC deletion.

    accessModes: ReadWriteMany
        NFS supports multiple Pods reading and writing the same volume.

---

### 8.3 Install provisioner

Create namespace:

    kubectl create namespace storage-system

Install:

    helm install nfs-subdir-external-provisioner \
      nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
      -n storage-system \
      -f values-nfs-subdir.yaml

Check Helm Release:

    helm list -n storage-system

Check Pods:

    kubectl -n storage-system get pods -o wide

Check StorageClass:

    kubectl get storageclass

Expected output:

    nfs-client

If set as default StorageClass, should see:

    nfs-client (default)

---

## Nine. Verify Dynamic PVC

### 9.1 Create Test PVC

Create test file: /think

```bash
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
```

**Apply:**

```bash
kubectl apply -f pvc-nfs-test.yaml
```

**Check PVC:**

```bash
kubectl get pvc
```

**Expected:**
STATUS is Bound

**Check PV:**

```bash
kubectl get pv
```

**Check PVC details:**

```bash
kubectl describe pvc nfs-test-pvc
```

---

### 9.2 Check NFS backend directory

Execute on ops-server:

```bash
find /data/nfs/k8s -maxdepth 2 -type d
```

You should see subdirectories automatically created by the provisioner.

---

## Ten. Create Pod mounting PVC for verification

### 10.1 Create test Pod

Create Pod:

```bash
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
```

**Apply:**

```bash
kubectl apply -f pod-nfs-test.yaml
```

**Check:**

```bash
kubectl get pod nfs-test-pod -o wide
```

---

### 10.2 Verify writing in Pod

Execute:

```bash
kubectl exec -it nfs-test-pod -- cat /data/hello.txt
```

**Expected output:**
hello from nfs pvc

---

### 10.3 Verify file on NFS Server

Execute on ops-server:

```bash
find /data/nfs/k8s -name hello.txt -type f
```

Check content:

```bash
cat $(find /data/nfs/k8s -name hello.txt -type f | head -n 1)
```

If you see:
hello from nfs pvc

**Explanation:**
- PVC dynamic provisioning is working
- Pod mounting is working
- NFS backend writing is working

---

## Eleven. Verify default StorageClass

If StorageClass is set as default, PVC can omit storageClassName.

Create PVC:

```bash
cat <<EOF > pvc-default-sc-test.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: default-sc-test-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
```

**Apply:**

```bash
kubectl apply -f pvc-default-sc-test.yaml
```

**Check:**

```bash
kubectl get pvc default-sc-test-pvc
```

**Check bound StorageClass:**

```bash
kubectl get pvc default-sc-test-pvc -o yaml | grep storageClassName
```

**Expected:**
storageClassName: nfs-client

**Cleanup:**

```bash
kubectl delete pvc default-sc-test-pvc
```

---

## Twelve. Reclaim Policy Verification

Delete Pod:

```bash
kubectl delete pod nfs-test-pod
```

Delete PVC:

```bash
kubectl delete pvc nfs-test-pvc
```

Check PV:

```bash
kubectl get pv
```

If reclaimPolicy is Delete, PV will be deleted after PVC deletion.

Check backend directory on ops-server:

```bash
find /data/nfs/k8s -maxdepth 2 -type d
```

If archiveOnDelete is enabled, backend directory may be renamed for archiving instead of direct deletion.

**Production recommendation:**
- For important data, do not immediately delete backend data when PVC is deleted.
- Using archiving is safer.

---

## Thirteen. Common Issues Troubleshooting

### 13.1 PVC remains in Pending state

**Troubleshoot:**
```bash
kubectl get pvc
kubectl describe pvc <pvc-name>
kubectl get storageclass
kubectl -n storage-system get pods -o wide
kubectl -n storage-system logs deploy/nfs-subdir-external-provisioner
```

**Common causes:**

1. StorageClass name is incorrect  
2. provisioner Pod is not running  
3. NFS Server is unreachable  
4. NFS path configuration is incorrect  
5. Kubernetes nodes do not have nfs-common installed  
6. RBAC or provisioner configuration is abnormal  

---

### 13.2 Pod Cannot Mount PVC  

Check the Pod:  

    kubectl describe pod <pod-name>  

Focus on the Events section.  

Common error messages:  

    mount.nfs: access denied by server  
    mount.nfs: Connection timed out  
    mount.nfs: No such file or directory  
    MountVolume.SetUp failed  

Troubleshooting steps:  

    showmount -e 10.0.0.10  

    sudo mount -t nfs 10.0.0.10:/data/nfs/k8s /mnt/nfs-test  

Common causes:  

    1. NFS Server has not authorized the current network segment  
    2. NFS Server firewall rules are not open  
    3. NFS path does not exist  
    4. nfs-common is not installed  
    5. NFS directory permissions are insufficient  

---

### 13.3 Default StorageClass Not Active  

Check StorageClass:  

    kubectl get storageclass  

If you do not see:  

    (default)  

You can manually set the default StorageClass:  

    kubectl patch storageclass nfs-client \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'  

Cancel default status:  

    kubectl patch storageclass nfs-client \
      -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'  

---

## Fourteen. Uninstallation  

Check PVC:  

    kubectl get pvc -A  

Confirm there are no business dependencies before uninstalling:  

    helm uninstall nfs-subdir-external-provisioner -n storage-system  

Check StorageClass:  

    kubectl get storageclass  

If confirmed no longer needed:  

    kubectl delete storageclass nfs-client  

Delete namespace:  

    kubectl delete namespace storage-system  

Notes:  
    Helm uninstallation does not automatically clean up all historical data on NFS Server.  
    NFS backend directory requires manual verification before cleanup based on actual conditions.  

---

## Fifteen. Post-Installation Checklist  

After installation, execute:  

    showmount -e 10.0.0.10  

    kubectl -n storage-system get pods -o wide  

    kubectl get storageclass  

    kubectl get pvc  

    kubectl get pv  

    kubectl get pod nfs-test-pod -o wide  

    find /data/nfs/k8s -maxdepth 2 -type d  

Requirements:  
    1. NFS Server exports /data/nfs/k8s normally  
    2. All Kubernetes nodes have nfs-common installed  
    3. nfs-subdir-external-provisioner Pod is Running  
    4. StorageClass nfs-client exists  
    5. nfs-client is the default StorageClass  
    6. PVC can be automatically Bound  
    7. PV can be automatically created  
    8. Pod can mount PVC  
    9. Pod write operations are visible on NFS Server  

---

## Sixteen. Summary  

This document completes the installation of a Kubernetes dynamic storage class based on NFS.  

Core content:  
    1. Install NFS Server  
    2. Configure NFS shared directory  
    3. Install nfs-common on all Kubernetes nodes  
    4. Use Helm to install nfs-subdir-external-provisioner  
    5. Create StorageClass nfs-client  
    6. Set default StorageClass  
    7. Create PVC to verify dynamic provisioning  
    8. Create Pod to verify mount and read/write  
    9. Check NFS backend directory  
    10. Troubleshoot PVC Pending, Pod mount failure, NFS permission issues  

Production recommendations:  
    1. NFS is suitable for basic shared storage and some non-core business scenarios  
    2. Core database business should not directly depend on single-node NFS  
    3. Monitor NFS Server high availability in production environments  
    4. Important data must have a backup strategy  
    5. Confirm whether backend data needs to be retained before deleting PVC  

Suggested next steps:  
    06-Longhorn Installation: Distributed block storage, StorageClass, and PVC verification.md