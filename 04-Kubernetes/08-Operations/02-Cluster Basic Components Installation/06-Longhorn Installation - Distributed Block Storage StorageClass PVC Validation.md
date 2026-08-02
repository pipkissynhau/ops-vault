# 06-Longhorn Installation: Distributed Block Storage, StorageClass, and PVC Verification

Recommended path:

    04-Kubernetes/08-Operations/02-Cluster Base Component Installation/06-Longhorn Installation: Distributed Block Storage, StorageClass, and PVC Verification.md

Tags:

    #Kubernetes
    #Longhorn
    #StorageClass
    #PV
    #PVC
    #CSI
    #DistributiveBlockStorage
    #EnduringStorage
    #ClusterBasicComponents

---

## I. Document Description

This document records the installation, verification, and basic usage methods of Longhorn in a Kubernetes cluster.

Longhorn is a native distributed block storage system for Kubernetes, providing dynamic storage capabilities based on CSI for the cluster.

This document uses:

    Longhorn
    Helm
    StorageClass
    PVC
    Pod Mount Verification
    Longhorn UI

This document aims to:

    1. Install Longhorn prerequisites
    2. Check if nodes meet Longhorn requirements
    3. Install Longhorn using Helm
    4. Configure Longhorn default data directory
    5. Verify Longhorn StorageClass
    6. Create PVC
    7. Create Pod to mount Longhorn PVC
    8. View volume status through Longhorn UI
    9. Master common troubleshooting methods

Applicable scenarios:

    1. kubeadm self-built cluster
    2. Private environment
    3. Small-to-medium production environment
    4. Need stronger block storage capability than NFS
    5. Scenarios requiring replication, snapshots, backup, and volume migration capabilities

Notes:

    Longhorn is not simply an NFS shared directory.
    Longhorn is a distributed block storage.
    Before using Longhorn in production environments, careful planning of disks, network, node roles, backup, and disaster recovery drills is required.

---

## II. Differences Between Longhorn and NFS

| Item | NFS Dynamic Provisioning | Longhorn |
|---|---|---|
| Type | File-sharing storage | Distributed block storage |
| Backend | Single or highly available NFS Server | Distributed storage composed of multiple-node local disks |
| Access Mode | Common RWX | Common RWO, supports RWX with Share Manager |
| Suitable Scenarios | Shared directories, configuration, non-core data | Stateful applications, block storage, database-like light-to-medium scenarios |
| High Availability | Depends on NFS Server's own HA | Achieved through multi-replica |
| Management | Relatively simple | Need to pay attention to disks, replicas, rebuild, and backup |
| Operation Complexity | Low | Medium |

Simple understanding:

    NFS is more like a shared folder.
    Longhorn is more like providing distributed cloud disks for Kubernetes.

---

## III. Planning Information

### 3.1 Longhorn Planning

| Item | Planning |
|---|---|
| Namespace | longhorn-system |
| Installation Method | Helm |
| Longhorn Version | 1.11.1 |
| Default Data Directory | /data/longhorn |
| Default StorageClass | longhorn |
| Default Replica Count | 3 |
| UI Access Method | port-forward or Ingress |
| Suitable Nodes | Worker nodes as the main focus |

---

### 3.2 Node Disk Planning

It is recommended to prepare independent data disks or data directories on each Worker node:

    k8s-worker-01    /data/longhorn
    k8s-worker-02    /data/longhorn
    k8s-worker-03    /data/longhorn

Notes:

    /data/longhorn should ideally be located on an independent disk or large-capacity partition.
    If /data is just a regular directory under the root partition, it essentially still occupies the system disk.
    It is not recommended to mix Longhorn data with the system disk in production environments.

---

## IV. Pre-deployment Checks

### 4.1 Check Cluster Status

Execute:

    kubectl get nodes -o wide

Requirements:

    All nodes Ready.

---

### 4.2 Check Helm

Execute:

    helm version

---

### 4.3 Check Node Disks

On each node prepared to run Longhorn, execute:

    df -h

    lsblk

    sudo mkdir -p /data/longhorn

    df -h /data/longhorn

Requirements:

    The partition where /data/longhorn resides has sufficient capacity.
    It is not recommended to place the Longhorn data directory in a small root partition.

---

### 4.4 Check kubelet Mount Propagation

Longhorn relies on Kubernetes nodes supporting normal volume mounting capabilities.

First confirm kubelet is running normally:

    systemctl status kubelet --no-pager

Confirm nodes have Pods that can normally mount regular volumes.

---

## V. Install Longhorn Dependencies on All Nodes

The following operations are recommended to be executed on all Worker nodes.

If Master nodes also allow running Longhorn components or business Pods, they also need to be executed.

Node range example:

    k8s-worker-01
    k8s-worker-02
    k8s-worker-03

---

### 5.1 Install Dependencies

Execute:

    sudo apt update

    sudo apt install -y \
      open-iscsi \
      nfs-common \
      cryptsetup \
      dmsetup \
      util-linux \
      curl \
      bash \
      grep \
      gawk

Notes:

    open-iscsi
        Longhorn uses iSCSI to mount volumes to nodes.

    nfs-common
        Longhorn RWX and backup capabilities require NFSv4 client support.

    cryptsetup
        Longhorn volume encryption capabilities depend on this.

    dmsetup
        User-space tool for device-mapper.

    util-linux
        Provides findmnt, lsblk, blkid, and other tools.

### 5.2 Start iscsid

Execute:

    sudo systemctl enable --now iscsid

    sudo systemctl enable --now open-iscsi

Check:

    systemctl status iscsid --no-pager

    systemctl status open-iscsi --no-pager

If a service does not exist, use the actual service name on the current system.

---

### 5.3 Create Longhorn Data Directory

Execute on each storage node:

    sudo mkdir -p /data/longhorn

    sudo chmod 0755 /data/longhorn

Check:

    ls -ld /data/longhorn

    df -h /data/longhorn

---

## SixI don't know.Use longhornctl for Pre-check

The following operations are executed on k8s-master-01.

---

### 6.1 Download longhornctl

Create directory:

    mkdir -p /root/k8s-yaml/longhorn

    cd /root/k8s-yaml/longhorn

Download:

    curl -sSfL -o longhornctl https://github.com/longhorn/cli/releases/download/v1.11.1/longhornctl-linux-amd64

Add executable permission:

    chmod +x longhornctl

Move to system PATH:

    sudo mv longhornctl /usr/local/bin/longhornctl

Check:

    longhornctl version

If downloading from GitHub is slow, you can download it in advance and upload it to this directory.

---

### 6.2 Execute preflight Check

Execute:

    longhornctl check preflight

If the check fails, fix node dependencies according to the output prompt.

Common check items:

    1. Whether open-iscsi is installed
    2. Whether iscsid is running
    3. Whether nfs-common is installed
    4. Whether cryptsetup is installed
    5. Whether dmsetup is available
    6. Whether the kernel supports NFSv4
    7. Whether mount propagation is normal

---

## SevenI don't know.Install Longhorn Using Helm

The following operations are executed on k8s-master-01.

---

### 7.1 Add Helm Repository

Execute:

    helm repo add longhorn https://charts.longhorn.io

    helm repo update

Check versions:

    helm search repo longhorn/longhorn --versions | head

---

### 7.2 Create values File

Create directory:

    mkdir -p /root/k8s-yaml/longhorn

    cd /root/k8s-yaml/longhorn

Create values file:

    cat <<EOF > values-longhorn.yaml
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

Explanation:

    defaultSettings.defaultDataPath
        Longhorn default data directory.

    storageReservedPercentageForDefaultDisk
        Default disk reserved space ratio.

    storageOverProvisioningPercentage
        Storage over-provisioning ratio. Set to 100 here to avoid excessive over-provisioning.

    persistence.defaultClass
        Whether to create the default Longhorn StorageClass.

    persistence.defaultClassReplicaCount
        Default volume replica count. Suggest 3 when there are 3 Worker nodes.

    reclaimPolicy
        Reclaim policy for PV after PVC is deleted.

---

### 7.3 Install Longhorn

Create namespace:

    kubectl create namespace longhorn-system

Install:

    helm install longhorn longhorn/longhorn \
      -n longhorn-system \
      -f values-longhorn.yaml \
      --version 1.11.1

Check Helm Release:

    helm list -n longhorn-system

---

## EightI don't know.Check Longhorn Status

### 8.1 View Pods

Execute:

    kubectl -n longhorn-system get pods -o wide

Common components:

    longhorn-manager
    longhorn-driver-deployer
    longhorn-ui
    longhorn-csi-plugin
    csi-attacher
    csi-provisioner
    csi-resizer
    csi-snapshotter
    instance-manager

Requirements:

    Key Pods must be in Running status.

---

### 8.2 View StorageClass

Execute:

    kubectl get storageclass

Expect to see:

    longhorn

If configured as default StorageClass, should see:

    longhorn (default)

---

### 8.3 View Longhorn CRD

Execute:

    kubectl get crd | grep longhorn

Should see Longhorn-related CRDs.

---

### 8.4 View Longhorn Nodes

Run:

    kubectl -n longhorn-system get nodes.longhorn.io

Or:

    kubectl -n longhorn-system get lhn

If the command does not support abbreviation, use the actual CRD name.

---

## 9. Accessing Longhorn UI

### 9.1 Temporarily Access via port-forward

Run:

    kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80 --address 0.0.0.0

Access:

    http://<k8s-master-01-IP>:8080

Example:

    http://10.0.0.20:8080

Note:

    port-forward is suitable for temporary access.
    Not recommended for production use.

---

### 9.2 Expose UI via Ingress (Optional)

If ingress-nginx is already installed, you can create an Ingress.

Create:

    cat <<EOF > longhorn-ui-ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: longhorn-ui
      namespace: longhorn-system
    spec:
      ingressClassName: nginx
      rules:
      - host: longhorn.ops.local
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: longhorn-frontend
                port:
                  number: 80
    EOF

Apply:

    kubectl apply -f longhorn-ui-ingress.yaml

Access:

    curl -H "Host: longhorn.ops.local" http://10.0.0.23:30080/

Production Note:

    Longhorn UI is not recommended to be exposed directly to the public internet.
    Production environments must add authentication, access control, or access via internal bastion host.

---

## 10. Create PVC to Verify Longhorn

### 10.1 Create Test PVC

Create:

    cat <<EOF > pvc-longhorn-test.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: longhorn-test-pvc
      namespace: default
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: longhorn
    EOF

Apply:

    kubectl apply -f pvc-longhorn-test.yaml

Check:

    kubectl get pvc longhorn-test-pvc

Expected:

    STATUS should be Bound

Check PV:

    kubectl get pv

---

### 10.2 Create Pod to Mount PVC

Create:

    cat <<EOF > pod-longhorn-test.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: longhorn-test-pod
      namespace: default
    spec:
      containers:
      - name: busybox
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "hello from longhorn pvc" > /data/hello.txt
          sleep 3600
        volumeMounts:
        - name: longhorn-data
          mountPath: /data
      volumes:
      - name: longhorn-data
        persistentVolumeClaim:
          claimName: longhorn-test-pvc
    EOF

Apply:

    kubectl apply -f pod-longhorn-test.yaml

Check:

    kubectl get pod longhorn-test-pod -o wide

---

### 10.3 Verify Write

Run:

    kubectl exec -it longhorn-test-pod -- cat /data/hello.txt

Expected Output:

    hello from longhorn pvc

Check in Longhorn UI:

    Volume
    longhorn-test-pvc corresponding volume
    Replica status
    Node location
    Health status

---

## 11. Verify Replica Scheduling

Check PVC corresponding PV:

    kubectl get pv

Check Longhorn volume:

    kubectl -n longhorn-system get volumes.longhorn.io

Check replicas:

    kubectl -n longhorn-system get replicas.longhorn.io

Note:

    When default replica count is 3, Longhorn will attempt to spread the volume replicas across different nodes.
    If there are fewer than 3 nodes, replicas may not be scheduled successfully.
    Insufficient disk space may also cause replica scheduling failure.

---

## 12. Delete Test Resources

Delete Pod:

    kubectl delete pod longhorn-test-pod

Delete PVC:

    kubectl delete pvc longhorn-test-pvc

Check PV:

    kubectl get pv

If reclaimPolicy is Delete, the PV will be deleted after PVC is removed.

In the Longhorn UI, confirm whether the corresponding Volume has been deleted.

---

## Thirteen. Common Troubleshooting

### 13.1 Longhorn Pod ImagePullBackOff

Check:

    kubectl -n longhorn-system get pods -o wide

Check events:

    kubectl -n longhorn-system describe pod <pod-name>

Common causes:

    1. Unable to access docker.io
    2. Unable to pull longhornio image
    3. Internal Harbor has not synchronized the image
    4. containerd network or proxy anomalies

Handling approach:

    1. Check the image configuration in Helm values
    2. Synchronize Longhorn image to internal Harbor
    3. Modify values and perform helm upgrade

---

### 13.2 longhornctl check preflight failure

Execute:

    longhornctl check preflight

Common causes:

    1. open-iscsi not installed
    2. iscsid not running
    3. nfs-common not installed
    4. cryptsetup not installed
    5. dmsetup not available
    6. Node kernel does not support NFSv4
    7. Mount propagation anomalies

Handling:

    sudo apt install -y open-iscsi nfs-common cryptsetup dmsetup util-linux

    sudo systemctl enable --now iscsid

---

### 13.3 PVC remains Pending

Check PVC:

    kubectl describe pvc longhorn-test-pvc

Check StorageClass:

    kubectl get storageclass

Check Longhorn Pod:

    kubectl -n longhorn-system get pods -o wide

Check Longhorn volume:

    kubectl -n longhorn-system get volumes.longhorn.io

Common causes:

    1. Longhorn CSI components not running
    2. StorageClass name written incorrectly
    3. Node disk not schedulable
    4. Replica count exceeds available storage nodes
    5. Longhorn manager anomalies

---

### 13.4 Pod unable to mount PVC

Check Pod:

    kubectl describe pod longhorn-test-pod

Check kubelet logs:

    journalctl -u kubelet -f

Check Longhorn CSI:

    kubectl -n longhorn-system get pods -o wide | grep csi

Common causes:

    1. open-iscsi anomalies
    2. iscsid not running
    3. Longhorn CSI plugin anomalies
    4. Volume not successfully attached
    5. Node unable to access Longhorn engine or replica

---

### 13.5 Longhorn volume replicas insufficient

Check Longhorn UI:

    Volume
    Replicas
    Events

Or check CRD:

    kubectl -n longhorn-system get replicas.longhorn.io

Common causes:

    1. Insufficient worker node count
    2. Some nodes have insufficient disk space
    3. Nodes marked as unschedulable
    4. Disk path does not exist
    5. Replica count set too high

Handling approach:

    1. Add more available storage nodes
    2. Reduce replica count
    3. Check /data/longhorn
    4. Check Longhorn Node and Disk status

---

### 13.6 Longhorn UI inaccessible

Check Service:

    kubectl -n longhorn-system get svc longhorn-frontend

Check Pod:

    kubectl -n longhorn-system get pod -l app=longhorn-ui

Use port-forward:

    kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80 --address 0.0.0.0

If using Ingress:

    kubectl -n longhorn-system describe ingress longhorn-ui

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=100

---

## Fourteen. Upgrade and Rollback

### 14.1 Check current version

Execute:

    helm list -n longhorn-system

    helm status longhorn -n longhorn-system

Check history:

    helm history longhorn -n longhorn-system

---

### 14.2 Backup values before upgrade

Execute:

    helm get values longhorn -n longhorn-system -o yaml > longhorn-values-backup.yaml

---

### 14.3 Upgrade

Update repository:

    helm repo update

Check version:

    helm search repo longhorn/longhorn --versions | head

Upgrade:

    helm upgrade longhorn longhorn/longhorn \
      -n longhorn-system \
      -f values-longhorn.yaml \
      --version <target version>

Note:

    Longhorn upgrade should read the corresponding version upgrade instructions.
    Direct cross-version upgrades are not recommended.
    Before production environment upgrades, confirm Volume health, backup availability, and clear maintenance window.

---

### 14.4 Rollback

Check history:

    helm history longhorn -n longhorn-system

Rollback:

    helm rollback longhorn <REVISION> -n longhorn-system

Note:

    Rolling back storage systems carries higher risks than rolling back stateless components.
    Do not arbitrarily roll back Longhorn in production environments; it must be combined with official upgrade instructions and on-site status evaluation.

---

## FifteenI don't know.Uninstallation

Note:

    Before uninstalling Longhorn, confirm all business PVCs have been migrated or deleted.
    Do not directly uninstall Longhorn in production environments.
    Longhorn uninstallation involves CRD, Volume, Finalizer, data directories, etc., and must be done with caution.

Check PVC:

    kubectl get pvc -A

Check Longhorn volumes:

    kubectl -n longhorn-system get volumes.longhorn.io

Delete test resources:

    kubectl delete pod longhorn-test-pod

    kubectl delete pvc longhorn-test-pvc

Uninstall Helm Release:

    helm uninstall longhorn -n longhorn-system

Check residual resources:

    kubectl get crd | grep longhorn

Note:

    Longhorn uninstallation and data cleanup are recommended to be written separately.
    It is not advisable to provide one-click forced cleanup commands in basic installation notes to avoid accidental deletion of production data.

---

## SixteenI don't know.Installation Completion Checklist

After installation, execute:

    longhornctl check preflight

    kubectl -n longhorn-system get pods -o wide

    kubectl get storageclass

    kubectl get pvc

    kubectl get pv

    kubectl -n longhorn-system get volumes.longhorn.io

    kubectl exec -it longhorn-test-pod -- cat /data/hello.txt

Should satisfy:

    1. All storage nodes have installed open-iscsi
    2. iscsid is running normally
    3. nfs-common is installed
    4. cryptsetup and dmsetup are installed
    5. /data/longhorn exists and has sufficient space
    6. Longhorn Pod is Running
    7. Longhorn StorageClass exists
    8. PVC can be Bound
    9. Pod can mount PVC
    10. Pod writes files normally
    11. Longhorn UI can view Volume health status

---

## SeventeenI don't know.Summary

This document completes the basic installation and verification of Longhorn.

Core content:

    1. Install prerequisite dependencies for Longhorn
    2. Start iscsid
    3. Create /data/longhorn data directory
    4. Use longhornctl for preflight checks
    5. Install Longhorn using Helm
    6. Configure Longhorn default data directory
    7. Create Longhorn StorageClass
    8. Create PVC to verify dynamic provisioning
    9. Create Pod to verify mounting and read/write
    10. View Volume, replicas, and health status via UI
    11. Troubleshoot issues like ImagePullBackOff, PVC Pending, mounting failure, insufficient replicas, etc.

Production recommendations:

    1. Longhorn data directory should be placed on an independent disk or large-capacity partition
    2. Not recommended to use with system disk
    3. Replica count should be planned based on node count and disk capacity
    4. Backup targets must be configured in production environments
    5. Production upgrades must follow Longhorn official upgrade path
    6. Longhorn UI must be restricted from access and cannot be exposed to public internet
    7. Fault and recovery drills must be conducted before core data business goes live