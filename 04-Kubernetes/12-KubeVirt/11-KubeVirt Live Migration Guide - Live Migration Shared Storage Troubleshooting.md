# 11-KubeVirt LiveMigration Getting Started: Live Migration, Shared Storage, and Migration Troubleshooting

Recommended path:

    04-Kubernetes/12-KubeVirt/11-KubeVirt LiveMigration Getting Started: Live Migration, Shared Storage, and Migration Troubleshooting.md

Tags:

    #Kubernetes
    #KubeVirt
    #LiveMigration
    #VirtualMachineMigration
    #VMI
    #virt-launcher
    #SharedStorage
    #RWX
    #Longhorn
    #DataVolume
    #PlatformEngineering
    #CloudlandVirtualization

---

## I. Document Overview

This document records the basic concepts, prerequisites, experimental design, execution methods, and troubleshooting approaches for KubeVirt LiveMigration.

LiveMigration is an advanced virtual machine operation capability in KubeVirt.

Its goals are:

    Migrate a VM from one Kubernetes Node to another while the VM is running,
    Minimize business interruption time.

Similar to traditional virtualization's:

    vSphere vMotion

But note:

    KubeVirt LiveMigration is not simply equivalent to vMotion.
    It relies on collaboration between multiple components including Kubernetes, KubeVirt, node KVM, storage, network, virt-handler, and virt-launcher.
    Storage and network are the most prone to issues.

This document's objectives:

    1. Understand the differences between cold migration, live migration, and storage migration
    2. Understand the basic workflow of KubeVirt LiveMigration
    3. Understand why LiveMigration typically requires shared storage
    4. Understand the relationship between RWX storage and LiveMigration
    5. Create a migratable VM
    6. Execute migration using virtctl migrate
    7. Observe the VirtualMachineInstanceMigration resource
    8. Verify changes in the VM's node before and after migration
    9. Master common failure causes and troubleshooting paths for LiveMigration
    10. Organize interview response logic

Applicable environment:

    Ubuntu 22.04
    Kubernetes v1.31.14
    KubeVirt v1.4.0
    CDI is installed
    virtctl is installed
    At least two worker nodes supporting KVM
    Available shared storage or RWX-compatible StorageClass

---

## II. Distinguish Migration Types Clearly

When KubeVirt mentions "migration," don't assume it refers to a single type.

At least distinguish three categories:

    1. Cold Migration
    2. LiveMigration (Hot Migration)
    3. Volume Migration (Storage Migration)

---

### 2.1 Cold Migration

Cold migration refers to:

    Stop the VM first,
    Then restart it on another node.

Characteristics:

    1. VM will experience interruption
    2. Implementation is relatively simple
    3. Lower storage requirements
    4. Suitable for operations during maintenance windows
    5. Not considered true live migration

Common scenarios:

    Node maintenance
    Resource adjustment
    Manual migration testing
    Adjusting non-critical VM nodes

Basic approach:

    virtctl stop <vm>
    Adjust scheduling rules
    virtctl start <vm>

---

### 2.2 LiveMigration (Hot Migration)

LiveMigration refers to:

    Migrating a VM from the source node to the target node while it's running.

Goal:

    Minimize business interruption.

Characteristics:

    1. VM remains operational as much as possible
    2. Both source and target nodes participate in migration
    3. Memory state needs to be transferred
    4. Storage typically requires access from both nodes
    5. Higher network and storage requirements
    6. Closer to traditional virtualization's vMotion

This document focuses on:

    LiveMigration

---

### 2.3 Volume Migration (Storage Migration)

Volume Migration refers to:

    Migrating the disk volumes used by the VM,
    For example, from one PVC/StorageClass to another PVC/StorageClass.

It leans more toward storage-level migration.

Common scenarios:

    1. Migrating from an old StorageClass to a new one
    2. From NFS to Longhorn/Ceph
    3. From low-performance storage to high-performance storage
    4. Adjusting the backend where the VM's disk resides

Note:

    Volume Migration is more advanced than this document.
    This document doesn't expand on it; it will be covered separately later.

---

## III. Basic Principles of KubeVirt LiveMigration

The core workflow of KubeVirt LiveMigration can be understood as:

    Source node virt-launcher
          |
          | Transfer VM runtime state, memory state
          v
    Target node virt-launcher
          |
          v
    VM continues running on the target node

More complete workflow:

    VirtualMachine
        |
        v
    VirtualMachineInstance
        |
        v
    VirtualMachineInstanceMigration
        |
        v
    Source node virt-launcher Pod
        |
        v
    Target node virt-launcher Pod
        |
        v
    VMI node changes after migration completes

During migration, you'll typically see:

    1. Original virt-launcher Pod
    2. New target virt-launcher Pod
    3. VirtualMachineInstanceMigration object
    4. MigrationState information in VMI status
    5. Change in VMI's nodeName

---

## IV. Why LiveMigration Typically Requires Shared Storage

When migrating a VM, it's not just process migration.

The VM depends on:

    1. CPU state
    2. Memory state
    3. Disk data
    4. Network connection
    5. Device state

Among these, disk data is most critical.

If the VM's system disk is local to the source node, the target node can't access this disk, making migration difficult.

Thus, LiveMigration typically expects:

Source nodes and target nodes can access the same virtual machine disk.

This is the importance of shared storage.

In Kubernetes, the corresponding PVC accessModes are commonly:

    ReadWriteMany

Abbreviated as:

    RWX

---

## Five, RWX, RWO, and Migration Relationship

### 5.1 RWO

RWO:

    ReadWriteOnce

Meaning:

    Typically can only be mounted in read-write mode by a single node.

Commonly used in:

    Longhorn default block storage
    Cloud disks
    Ceph RBD
    Some block storage CSI

RWO is suitable for:

    VM systems disk running on a single node
    Ordinary VM boot disk
    Ordinary stateful applications

But for LiveMigration:

    RWO is not the simplest experimental choice.

Reason:

    During migration, both source and target nodes may need to access the virtual machine disk.
    If the storage does not support migration scenarios, it may lead to VM being non-migratable or migration failure.

---

### 5.2 RWX

RWX:

    ReadWriteMany

Meaning:

    Multiple nodes can mount and access simultaneously.

Commonly used in:

    NFS
    CephFS
    Longhorn RWX
    Some distributed file storage

More friendly to LiveMigration.

For beginner experiments, it is recommended to prioritize using:

    RWX StorageClass

---

### 5.3 Longhorn Scenario Description

Longhorn is commonly used as default RWO block storage.

If you want to support KubeVirt LiveMigration, you usually need to pay extra attention to:

    1. Whether Longhorn supports RWX
    2. Whether share-manager is functioning normally
    3. Whether nodes have nfs-common installed
    4. Whether the StorageClass is suitable for migration scenarios
    5. Whether PVC accessModes use ReadWriteMany
    6. Whether the current Longhorn version supports relevant KubeVirt migration capabilities

Production recommendations:

    Do not directly use the default RWO system disk to test LiveMigration.
    Prepare a dedicated RWX StorageClass or a StorageClass explicitly supporting migration first.
    Specific parameters should be based on the current Longhorn version and company storage specifications.

---

## Six, Common Limitations of LiveMigration

In the initial stage, remember the following limitations:

    1. VM needs to be judged as migratable by KubeVirt
    2. VM disks must meet migration requirements
    3. Nodes need to support KVM
    4. Both source and target nodes must have sufficient resources
    5. Source and target nodes must have network connectivity
    6. virt-handler must be functioning normally
    7. virt-launcher migration ports must not be blocked
    8. Some network binding modes are unsuitable for LiveMigration
    9. Using HostDisk, local disks, device passthrough, GPU, SR-IOV, etc., may limit migration
    10. Migration is not a universal capability; it must be validated before production

Common scenarios unsuitable for direct migration:

    1. Using hostPath / HostDisk
    2. Using local node disks
    3. Using non-shareable RWO disks
    4. Using PCI device passthrough
    5. Using GPU passthrough
    6. Using SR-IOV network card passthrough
    7. Using certain special network bindings
    8. Target node has insufficient resources
    9. Target node lacks /dev/kvm

---

## Seven, Differences Between LiveMigration and vSphere vMotion

| Comparison Item | vSphere vMotion | KubeVirt LiveMigration |
|---|---|---|
| Management Entry | vCenter | Kubernetes API / virtctl |
| Virtualization Layer | ESXi | KVM / QEMU |
| Scheduling System | vSphere Cluster / DRS | Kubernetes Scheduler |
| Storage System | Datastore / vSAN / SAN | PVC / CSI / StorageClass |
| Network System | vSwitch / dvSwitch | CNI / Service / Multus |
| Migration Object | VM | VMI / virt-launcher |
| Operation Method | vCenter UI / PowerCLI | kubectl / virtctl |
| Dependent Components | vCenter / ESXi | virt-api / virt-controller / virt-handler |
| Troubleshooting Objects | VM, Host, Datastore | VM, VMI, Pod, PVC, Node |

In one sentence:

    vMotion is a mature migration capability in traditional virtualization platforms.
    KubeVirt LiveMigration is a virtual machine migration capability in the Kubernetes ecosystem,
    It relies more on Kubernetes' scheduling, storage, and network infrastructure.

---

## Eight, Experiment Objectives

This article's experiment objectives:

    1. Create an experimental namespace
    2. Prepare RWX PVC / DataVolume
    3. Create a VM with evictionStrategy
    4. Start the VM
    5. Record the node where the VM was located before migration
    6. Execute migration using virtctl
    7. Observe VirtualMachineInstanceMigration
    8. Observe changes in virt-launcher Pod
    9. Verify changes in the node where the VM is located after migration
    10. Test node drain triggering migration
    11. Summarize the migration failure troubleshooting path

---

## Nine, Experiment Environment Planning

Example nodes:

    k8s-master-01     10.0.0.20
    k8s-master-02     10.0.0.21
    k8s-master-03     10.0.0.22
    k8s-worker-01     10.0.0.23
    k8s-worker-02     10.0.0.24
    k8s-worker-03     10.0.0.25

Recommended VM runtime nodes:

    k8s-worker-01
    k8s-worker-02
    k8s-worker-03

Experimental namespace:

    kubevirt-migration-demo

Experimental VM:

    vm-live-migrate-demo

Recommended StorageClass:

    longhorn-rwx

If there is no longhorn-rwx, it can be replaced with a StorageClass that actually supports RWX, for example:

    nfs-client
    cephfs
    rook-cephfs

---

## Ten. Pre-Experiment Checks

### 10.1 Check KubeVirt

Execute:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

Requirements:

    KubeVirt Available
    virt-api Running
    virt-controller Running
    virt-handler Running
    virt-operator Running

---

### 10.2 Check virtctl

Execute:

    virtctl version

---

### 10.3 Check CDI

Execute:

    kubectl -n cdi get pods -o wide

    kubectl get dv -A

If DataVolume resources are unavailable, CDI needs to be installed first.

---

### 10.4 Check Node KVM

Execute on each Worker node:

    egrep -c '(vmx|svm)' /proc/cpuinfo

    ls -l /dev/kvm

    sudo kvm-ok

Requirements:

    vmx / svm greater than 0
    /dev/kvm exists
    kvm-ok passes

---

### 10.5 Check Worker Node Count

Execute:

    kubectl get nodes -o wide

At least two available Worker nodes are required.

If only one running node exists, cross-node migration cannot be verified.

---

### 10.6 Check Node Resources

Execute:

    kubectl top nodes

If metrics-server is not installed, view:

    kubectl describe node <node-name> | grep -A8 "Allocated resources"

Requirements:

    Source and target nodes have sufficient CPU / Memory.

---

### 10.7 Check StorageClass

Execute:

    kubectl get storageclass

Confirm there is a StorageClass available for RWX.

Examples:

    longhorn-rwx
    nfs-client
    cephfs

If only the default longhorn RWO is available, it is recommended to avoid LiveMigration experiments.

---

## Eleven. Prepare RWX StorageClass (Longhorn Scenario Optional)

If the cluster already has a usable RWX StorageClass, skip this section.

If using Longhorn, confirm the current Longhorn version and configuration support RWX.

Example StorageClass (only for experimental reference):

    cat <<EOF > sc-longhorn-rwx.yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: longhorn-rwx
    provisioner: driver.longhorn.io
    allowVolumeExpansion: true
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    parameters:
      numberOfReplicas: "3"
      staleReplicaTimeout: "30"
      fsType: ext4
      migratable: "true"
    EOF

Apply:

    kubectl apply -f sc-longhorn-rwx.yaml

Check:

    kubectl get storageclass longhorn-rwx

Notes:

    1. Longhorn RWX depends on share-manager capabilities.
    2. Nodes typically need nfs-common.
    3. Whether the migratable parameter applies depends on the current Longhorn version.
    4. Do not arbitrarily create StorageClass in production environments; follow company storage guidelines.
    5. If this StorageClass cannot RWX Bind after creating PVC, further configuration of Longhorn RWX is needed in the environment.

Install NFS client on nodes:

    sudo apt update

    sudo apt install -y nfs-common

Check Longhorn:

    kubectl -n longhorn-system get pods -o wide

    kubectl -n longhorn-system get nodes.longhorn.io

    kubectl -n longhorn-system get volumes.longhorn.io

---

## Twelve. Create Experiment Namespace

Create:

    kubectl create namespace kubevirt-migration-demo --dry-run=client -o yaml | kubectl apply -f -

Check:

    kubectl get ns kubevirt-migration-demo

Create directory:

    mkdir -p /root/k8s-yaml/kubevirt/migration-demo

    cd /root/k8s-yaml/kubevirt/migration-demo

---

## Thirteen. Experiment One: Create RWX PVC Pre-Check

First create a regular RWX PVC to confirm the StorageClass works properly.

Create:

    cat <<EOF > pvc-rwx-precheck.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: rwx-precheck-pvc
      namespace: kubevirt-migration-demo
    spec:
      accessModes:
      - ReadWriteMany
      resources:
        requests:
          storage: 1Gi
      storageClassName: longhorn-rwx
    EOF

If your RWX StorageClass is not longhorn-rwx, replace:

    storageClassName: <your RWX StorageClass>

Apply:

    kubectl apply -f pvc-rwx-precheck.yaml

Check:

    kubectl -n kubevirt-migration-demo get pvc rwx-precheck-pvc

Expected:

    STATUS: Bound
    ACCESS MODES: RWX

Check PV:

    kubectl get pv | grep rwx-precheck-pvc

If PVC is Pending:

    kubectl -n kubevirt-migration-demo describe pvc rwx-precheck-pvc

Continue troubleshooting:

    StorageClass
    Longhorn / NFS / CephFS
    provisioner
    share-manager
    node nfs-common

Clean up pre-check PVC:

    kubectl -n kubevirt-migration-demo delete pvc rwx-precheck-pvc

---

## FourteenI don't know.Experiment Two: Create System Disk DataVolume

Create DataVolume:

    cat <<EOF > dv-live-migrate-rootdisk.yaml
    apiVersion: cdi.kubevirt.io/v1beta1
    kind: DataVolume
    metadata:
      name: live-migrate-rootdisk
      namespace: kubevirt-migration-demo
    spec:
      source:
        http:
          url: "https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"
      pvc:
        accessModes:
        - ReadWriteMany
        resources:
          requests:
            storage: 2Gi
        storageClassName: longhorn-rwx
    EOF

If using other RWX StorageClass, replace:

    storageClassName: <your RWX StorageClass>

Apply:

    kubectl apply -f dv-live-migrate-rootdisk.yaml

Check DataVolume:

    kubectl -n kubevirt-migration-demo get dv

Check PVC:

    kubectl -n kubevirt-migration-demo get pvc

Check importer Pod:

    kubectl -n kubevirt-migration-demo get pods

Check import logs:

    kubectl -n kubevirt-migration-demo logs <importer-pod-name> --tail=100

Wait for import success:

    kubectl -n kubevirt-migration-demo get dv live-migrate-rootdisk

Expected:

    PHASE: Succeeded

PVC Expected:

    STATUS: Bound
    ACCESS MODES: RWX

If public URL access is slow, you can switch to internal HTTP address:

    http://10.0.0.10/images/cirros-0.6.2-x86_64-disk.img

---

## FifteenI don't know.Experiment Three: Create Migratable VM

Create VM: /think

```yaml
cat <<EOF > vm-live-migrate-demo.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-live-migrate-demo
  namespace: kubevirt-migration-demo
  labels:
    app: vm-live-migrate-demo
spec:
  runStrategy: Manual
  template:
    metadata:
      labels:
        app: vm-live-migrate-demo
        kubevirt.io/domain: vm-live-migrate-demo
    spec:
      evictionStrategy: LiveMigrate
      terminationGracePeriodSeconds: 0
      domain:
        resources:
          requests:
            memory: 512Mi
        devices:
          disks:
          - name: rootdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virt
          interfaces:
          - name: default
            masquerade: {}
            ports:
            - name: ssh
              port: 22
      networks:
      - name: default
        pod: {}
      volumes:
      - name: rootdisk
        dataVolume:
          name: live-migrate-rootdisk
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
            password: kubevirt
            chpasswd:
              expire: false
            ssh_pwauth: true
EOF
```

**Explanation:**

- `evictionStrategy: LiveMigrate`
  Indicates that when the node evicts the VMI, it will prioritize LiveMigration.

- `rootdisk` uses DataVolume:
  This DataVolume corresponds to a RWX PVC, facilitating migration.

- `masquerade`:
  Uses the default Pod network, suitable for introductory migration experiments.

**Application:**

```bash
kubectl apply -f vm-live-migrate-demo.yaml
```

**Viewing:**

```bash
kubectl -n kubevirt-migration-demo get vm
```

---

## Sixteen, Experiment Four: Starting VM and Recording the Source Node

**Start:**

```bash
virtctl start vm-live-migrate-demo -n kubevirt-migration-demo
```

**View VM:**

```bash
kubectl -n kubevirt-migration-demo get vm
```

**View VMI:**

```bash
kubectl -n kubevirt-migration-demo get vmi -o wide
```

**View virt-launcher:**

```bash
kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher
```

**Record:**
- VM current status
- VMI current node
- virt-launcher Pod current node
- VMI IP

**Example:**

```bash
kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o wide
```

```
NAME                   AGE   PHASE     IP           NODE
vm-live-migrate-demo   1m    Running   10.244.x.x   k8s-worker-01
```

Here, `k8s-worker-01` is the source node.

---

## Seventeen, Checking if VMI is Migratable

**View VMI details:**

```bash
kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo
```

Focus on whether there is a condition like:

```
LiveMigratable
```

**Alternatively, view YAML:**

```bash
kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o yaml | grep -A20 -i migrat
```

If it shows not migratable, typically check:

1. Whether the disk is RWX
2. Whether it uses a disk type not supported for migration
3. Whether it uses hostDisk / local storage
4. Whether it uses special device passthrough
5. Whether the network binding is supported
6. Whether KubeVirt migration configuration allows it

---

## Eighteen, Experiment Five: Executing LiveMigration

**Execute migration:**

```bash
virtctl migrate vm-live-migrate-demo -n kubevirt-migration-demo
```

**Explanation:**
`virtctl migrate` creates a migration request for a running VMI.

**View migration object:**

```bash
kubectl -n kubevirt-migration-demo get virtualmachineinstancemigrations
```

If shorthand is supported, you can also try:

    kubectl -n kubevirt-migration-demo get vmim

For details:

    kubectl -n kubevirt-migration-demo describe virtualmachineinstancemigration <migration-name>

Monitor the VMI:

    kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o wide

Monitor the virt-launcher:

    kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher

Recommend opening two windows for monitoring:

Window one:

    watch -n 1 "kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o wide"

Window two:

    watch -n 1 "kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher"

During migration, you may observe:

    1. A VirtualMachineInstanceMigration object appears
    2. A new virt-launcher Pod appears on the target node
    3. There may be two related virt-launcher Pods briefly
    4. The VMI's NODE changes from the source node to the target node
    5. The migration object eventually shows Succeeded

---

## Nineteen. Experiment Six: Verifying Migration Results

Check the VMI:

    kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o wide

Check the virt-launcher:

    kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher

Check the migration object:

    kubectl -n kubevirt-migration-demo get virtualmachineinstancemigrations

Check the VMI migrationState:

    kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o yaml | grep -A80 migrationState

Determine if migration succeeded:

    1. The VMI is still Running
    2. The VMI's NODE has changed to the new node
    3. The virt-launcher Pod on the new node is Running
    4. The migration-related Pods on the old node are cleaned up
    5. The VirtualMachineInstanceMigration shows Succeeded
    6. Console or SSH access remains available

Enter the console:

    virtctl console vm-live-migrate-demo -n kubevirt-migration-demo

Login:

    Username: cirros
    Password: kubevirt

Check:

    hostname

    ip addr

    uptime

Exit:

    Ctrl + ]

---

## Twenty. Experiment Seven: Verifying Access During Migration via Service

Create an SSH Service:

    cat <<EOF > svc-live-migrate-ssh.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: vm-live-migrate-ssh
      namespace: kubevirt-migration-demo
    spec:
      type: NodePort
      selector:
        kubevirt.io/domain: vm-live-migrate-demo
      ports:
      - name: ssh
        protocol: TCP
        port: 22
        targetPort: 22
        nodePort: 30025
    EOF

Apply:

    kubectl apply -f svc-live-migrate-ssh.yaml

Check:

    kubectl -n kubevirt-migration-demo get svc

    kubectl -n kubevirt-migration-demo get endpoints vm-live-migrate-ssh

Access from outside:

    ssh cirros@10.0.0.23 -p 30025

Password:

    kubevirt

During migration, you can continuously ping or SSH from another terminal to observe.

Note:

    CirrOS is lightweight, and this experiment is only for understanding the migration process.
    In production, you should use real business connections and monitoring to verify the impact of migration.

---

## Twenty-one. Experiment Eight: Triggering Migration via Drain

Explanation:

    evictionStrategy: LiveMigrate is an important use case for node maintenance.
    When a node is drained, KubeVirt can attempt to perform LiveMigration on VMs.

First confirm the current node of the VM:

    kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o wide

Assume the current node is:

    k8s-worker-01

Execute cordon:

    kubectl cordon k8s-worker-01

Execute drain:

    kubectl drain k8s-worker-01 \
      --ignore-daemonsets \
      --delete-emptydir-data

Monitor the migration:

    kubectl -n kubevirt-migration-demo get virtualmachineinstancemigrations

    kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o wide

    kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher

After migration completes, the VMI should be running on another Worker node.

Restore node scheduling: /think

kubectl uncordon k8s-worker-01

Note:

- Drain is a high-risk operation.
- It is only allowed in experimental environments or during explicitly defined maintenance windows.
- Before draining a node in production environments, evaluate all business Pods and VMs on that node.

---

## 22. View KubeVirt Migration Configuration

Check KubeVirt CR:

    kubectl -n kubevirt get kubevirt kubevirt -o yaml | grep -A80 -i migration

The field names may vary slightly across versions.

Common migration-related configurations include:

    parallelMigrationsPerCluster
    parallelOutboundMigrationsPerNode
    bandwidthPerMigration
    completionTimeoutPerGiB
    progressTimeout
    allowAutoConverge
    allowPostCopy

Note:

- These configurations affect migration concurrency, bandwidth, timeouts, and auto-convergence behaviors.
- It is not recommended to modify these configurations directly during the initial phase.
- Before modifying in production environments, validation must be done in a test environment.

Example patch (only for record, not recommended for production execution):

    kubectl -n kubevirt patch kubevirt kubevirt --type=merge \
      -p '{"spec":{"configuration":{"migrations":{"parallelMigrationsPerCluster":5,"parallelOutboundMigrationsPerNode":2,"bandwidthPerMigration":"64Mi","completionTimeoutPerGiB":800,"progressTimeout":150}}}}'

After modification, check:

    kubectl -n kubevirt get kubevirt kubevirt -o yaml | grep -A80 -i migration

Note:

- The patch fields must match the actual KubeVirt version CRD.
- If the fields are incompatible, use the actual supported fields of the current version.

---

## 23. Common LiveMigration States

Check migration objects:

    kubectl -n kubevirt-migration-demo get virtualmachineinstancemigrations

Common phases may include:

    Scheduling
    PreparingTarget
    TargetReady
    Running
    Succeeded
    Failed

The displayed fields may vary slightly across versions; refer to actual output for accuracy.

Check details:

    kubectl -n kubevirt-migration-demo describe virtualmachineinstancemigration <migration-name>

Focus on:

    Phase
    Conditions
    Source Node
    Target Node
    VMI Name
    Events

Check VMI:

    kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo

Focus on:

    Migration State
    Conditions
    Events

---

## 24. Mainline Troubleshooting for Migration Failures

When LiveMigration fails, troubleshoot in the following order:

    1. Check if VMI is Running
    2. Check if VMI is LiveMigratable
    3. Check details of VirtualMachineInstanceMigration
    4. Check VMI Events
    5. Check virt-launcher Pods on source and target nodes
    6. Check virt-handler logs on source and target nodes
    7. Check PVC accessModes
    8. Check DataVolume / PVC / PV
    9. Check resources on target node
    10. Check /dev/kvm on target node
    11. Check network connectivity and migration ports
    12. Check component status in kubevirt namespace

---

## 25. Common Issue 1: VMI Not Migratable

Phenomenon:

- virtctl migrate fails
- describe vmi shows LiveMigratable=False

Troubleshoot:

    kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo

    kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o yaml | grep -A30 -i migrat

Common causes:

    1. PVC is not RWX
    2. Using hostDisk
    3. Using local storage
    4. Using devices not supported for migration
    5. Using special network binding
    6. Using SR-IOV / GPU / PCI passthrough
    7. VMI status is not Running

Resolution:

    1. Use RWX PVC
    2. Avoid hostDisk / local storage
    3. Avoid device passthrough configurations
    4. Simplify network mode to pod network + masquerade
    5. First validate migration chain with minimal CirrOS VM

---

## 26. Common Issue 2: PVC is not RWX

Check PVC:

    kubectl -n kubevirt-migration-demo get pvc

    kubectl -n kubevirt-migration-demo describe pvc live-migrate-rootdisk

If ACCESS MODES is:

    RWO

Instead of:

    RWX

It is not suitable as a root disk for an introductory LiveMigration experiment system.

Resolution:

    1. Use a StorageClass that supports RWX
    2. Recreate DataVolume / PVC
    3. Use ReadWriteMany
    4. Confirm the backend storage supports multi-node access
    5. Do not directly modify existing PVC accessModes

Note:

PVC accessModes are typically not directly modifiable after creation.
A suitable PVC / DataVolume needs to be recreated.

---

## 27. Common Issue Three: Target Node Resource Insufficiency

Migration requires the target node to have sufficient resources.

Check VMI:

    kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo

Check migration object:

    kubectl -n kubevirt-migration-demo describe virtualmachineinstancemigration <migration-name>

Check node resources:

    kubectl top nodes

Or:

    kubectl describe node <target-node> | grep -A8 "Allocated resources"

Common events:

    insufficient cpu
    insufficient memory
    0/3 nodes are available

Handling:

    1. Reduce VM memory request
    2. Clean up target node load
    3. Add more Worker nodes
    4. Confirm target node has no taint preventing scheduling
    5. Confirm VM has no nodeSelector restricting to source node

---

## 28. Common Issue Four: Target Node Lacks KVM

Check target node:

    ls -l /dev/kvm

    egrep -c '(vmx|svm)' /proc/cpuinfo

    sudo kvm-ok

If the target node does not support KVM, VM cannot migrate normally to this node.

Handling:

    1. Only schedule VM to nodes supporting KVM
    2. Label KVM nodes
    3. Use nodeSelector / affinity for VM
    4. Fix nested virtualization on node
    5. Do not add KVM-uncompatible nodes to VM node pool

---

## 29. Common Issue Five: virt-handler Abnormality

Check virt-handler:

    kubectl -n kubevirt get pods -o wide | grep virt-handler

Find the source node and target node corresponding virt-handler.

Check logs:

    kubectl -n kubevirt logs <virt-handler-pod-name> --tail=200

Common causes:

    1. virt-handler Pod abnormality
    2. Node kubelet abnormality
    3. Node containerd abnormality
    4. /dev/kvm unavailable
    5. KubeVirt component version or configuration abnormality

Handling:

    1. First confirm virt-handler is Running
    2. Then confirm node kubelet and containerd are normal
    3. Then confirm /dev/kvm
    4. Check kubevirt namespace Events

---

## 30. Common Issue Six: Migration Timeout or Non-Convergence

LiveMigration needs to synchronize VM memory state to the target node.

If VM memory changes frequently, migration may take a long time to converge.

Common causes:

    1. VM memory writes very frequently
    2. Migration network bandwidth insufficient
    3. bandwidthPerMigration too small
    4. progressTimeout too short
    5. completionTimeoutPerGiB not suitable
    6. Poor network quality between source and target nodes

Troubleshoot:

    kubectl -n kubevirt-migration-demo describe virtualmachineinstancemigration <migration-name>

    kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo

    kubectl -n kubevirt logs <source-virt-handler-pod> --tail=200

    kubectl -n kubevirt logs <target-virt-handler-pod> --tail=200

Handling direction:

    1. Adjust migration window
    2. Reduce VM load
    3. Adjust migration bandwidth
    4. Evaluate enabling auto-converge
    5. Check source/target node network
    6. Conduct migration behavior stress testing in test cluster first in production environment

---

## 31. Common Issue Seven: Migration Object Failed

Check migration object:

    kubectl -n kubevirt-migration-demo get virtualmachineinstancemigrations

    kubectl -n kubevirt-migration-demo describe virtualmachineinstancemigration <migration-name>

Check Events:

    kubectl -n kubevirt-migration-demo get events --sort-by=.lastTimestamp

Check VMI:

    kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo

Common causes:

    1. VMI not migratable
    2. Target node scheduling failure
    3. PVC not meeting migration requirements
    4. Migration connection failure
    5. Target virt-launcher startup failure
    6. Source or target node virt-handler abnormality
    7. Migration timeout
    8. VM stopped or deleted during migration

---

## 32. Common Issue Eight: VM Not Migrated When Draining Node

Check VM YAML:

    kubectl -n kubevirt-migration-demo get vm vm-live-migrate-demo -o yaml | grep -A5 evictionStrategy

Should include:

    evictionStrategy: LiveMigrate

Check if VMI is migratable:

    kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo

Check node drain output.

Common causes:

1. VM is not configured with evictionStrategy: LiveMigrate  
2. VMI is not migratable  
3. No available target nodes  
4. Target node resources are insufficient  
5. PVC does not meet migration requirements  
6. KubeVirt migration configuration limitations  
7. PodDisruptionBudget or eviction restrictions  

**Handling:**  

1. Add evictionStrategy to the VM  
2. Use RWX PVC  
3. Increase available nodes  
4. Reduce VM resources  
5. Manually verify with virtctl migrate  

---

## 33. LiveMigration Standard Command List  

### 33.1 View VM / VMI  

    kubectl -n kubevirt-migration-demo get vm  

    kubectl -n kubevirt-migration-demo get vmi -o wide  

    kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo  

---

### 33.2 Execute Migration  

    virtctl migrate vm-live-migrate-demo -n kubevirt-migration-demo  

---

### 33.3 View Migration Resources  

    kubectl -n kubevirt-migration-demo get virtualmachineinstancemigrations  

    kubectl -n kubevirt-migration-demo describe virtualmachineinstancemigration <migration-name>  

If supported:  

    kubectl -n kubevirt-migration-demo get vmim  

---

### 33.4 View Pod  

    kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher  

    kubectl -n kubevirt-migration-demo describe pod <virt-launcher-pod-name>  

    kubectl -n kubevirt-migration-demo logs <virt-launcher-pod-name> --tail=100  

---

### 33.5 View PVC  

    kubectl -n kubevirt-migration-demo get pvc  

    kubectl -n kubevirt-migration-demo describe pvc live-migrate-rootdisk  

---

### 33.6 View KubeVirt Components  

    kubectl -n kubevirt get pods -o wide  

    kubectl -n kubevirt logs deploy/virt-controller --tail=200  

    kubectl -n kubevirt logs deploy/virt-api --tail=200  

    kubectl -n kubevirt get pods -o wide | grep virt-handler  

    kubectl -n kubevirt logs <virt-handler-pod-name> --tail=200  

---

### 33.7 View Nodes  

    kubectl get nodes -o wide  

    kubectl describe node <node-name>  

Node native:  

    ls -l /dev/kvm  

    egrep -c '(vmx|svm)' /proc/cpuinfo  

    systemctl status kubelet --no-pager  

    systemctl status containerd --no-pager  

    journalctl -u kubelet -n 200 --no-pager  

---

## 34. Production Environment Notes  

Before deploying KubeVirt LiveMigration in production, must confirm:  

    1. Whether the VM is truly suitable for live migration  
    2. Whether storage supports migration  
    3. Whether the network supports migration traffic  
    4. Whether node resources are sufficient  
    5. Whether migration affects business connectivity  
    6. Whether there is a rollback plan for migration failure  
    7. Whether there is a maintenance window  
    8. Whether there is monitoring and alerting  
    9. Whether migration drills have been conducted  
    10. Whether there is a clear node maintenance process  

Do not directly perform these in production:  

    1. Arbitrarily drain nodes  
    2. Arbitrarily migrate core VMs  
    3. Arbitrarily modify KubeVirt migration configuration  
    4. Arbitrarily change StorageClass  
    5. Arbitrarily delete migrating virt-launcher Pods  
    6. Arbitrarily delete PVC or VolumeAttachment  

Production recommendations:  

    First validate in test environment.  
    Then validate on non-core VMs.  
    Finally integrate into standard maintenance process.  

---

## 35. Interview Answer: What is KubeVirt LiveMigration  

You can answer:  

    KubeVirt LiveMigration is the ability to migrate running virtual machines in KubeVirt.  
    It can move a running VMI from one Kubernetes node to another, minimizing business disruption.  
    During migration, a VirtualMachineInstanceMigration object is created, and a new virt-launcher Pod is started on the target node.  
    After successful migration, the VMI's runtime node changes from the source node to the target node.  
    This capability depends on node KVM, virt-handler, shared storage, network connectivity, and target node resources.  

---

## 36. Interview Answer: Why Does KubeVirt LiveMigration Need Shared Storage  

You can answer:  

    When migrating a VM, it's not only necessary to migrate the runtime state and memory, but also to ensure the target node can access the VM's disk.  
    If the system disk is only available on the source node locally, the target node cannot access the disk, making migration difficult.  
    Therefore, KubeVirt LiveMigration typically requires the PVC containing the VM's disk to support shared access, such as RWX.  
    In Kubernetes, this usually depends on a StorageClass that supports ReadWriteMany, such as NFS, CephFS, or Longhorn RWX.  
    Using regular RWO local disks or hostDisk often limits migration capabilities.

## Thirty-Seven, Interview Answer: How to Troubleshoot Failed KubeVirt LiveMigration

You can answer like this:

    I would first confirm if the VMI is Running and meets the LiveMigratable conditions.
    Then check the status and Events of the VirtualMachineInstanceMigration object.
    If the migration scheduling fails, check the target node resources, taint, nodeSelector, and KVM capabilities.
    If it's a storage issue, check PVC accessModes, DataVolume, PV, StorageClass, and backend storage status.
    If the migration fails during the process, check the virt-handler logs on the source and target nodes, the virt-launcher Pod status, and the VMI's migrationState.
    So the troubleshooting chain is VMI -> Migration object -> virt-launcher -> PVC/Storage -> virt-handler -> Node/KVM.

---

## Thirty-Eight, Interview Answer: Differences Between KubeVirt LiveMigration and vMotion

You can answer like this:

    vSphere vMotion is a very mature live migration capability in the vSphere platform, relying on vCenter, ESXi, vSwitch, Datastore, and other traditional virtualization systems.
    KubeVirt LiveMigration is a virtual machine migration capability in the Kubernetes ecosystem, relying on VM/VMI, virt-launcher, virt-handler, PVC, CSI, CNI, and Kubernetes Scheduler.
    Both have similar goals, aiming to minimize business interruption during migration.
    However, their implementation systems differ. KubeVirt migration relies more on Kubernetes' storage, network, and scheduling capabilities.
    Therefore, KubeVirt migration issues cannot be troubleshooted using traditional virtualization approaches alone; you must also check Pods, PVCs, Nodes, Events, and KubeVirt component logs.

---

## Thirty-Nine, Clean Up Experimental Resources

Stop VM:

    virtctl stop vm-live-migrate-demo -n kubevirt-migration-demo

Delete Service:

    kubectl delete -f svc-live-migrate-ssh.yaml --ignore-not-found

Delete VM:

    kubectl delete -f vm-live-migrate-demo.yaml --ignore-not-found

Delete DataVolume:

    kubectl delete -f dv-live-migrate-rootdisk.yaml --ignore-not-found

Check PVC:

    kubectl -n kubevirt-migration-demo get pvc

If confirmed that no data needs to be retained, delete residual PVC:

    kubectl -n kubevirt-migration-demo delete pvc live-migrate-rootdisk --ignore-not-found

Delete pre-check resources:

    kubectl delete -f pvc-rwx-precheck.yaml --ignore-not-found

Delete namespace:

    kubectl delete namespace kubevirt-migration-demo

If an experimental StorageClass was created and confirmed to no longer be used:

    kubectl delete -f sc-longhorn-rwx.yaml --ignore-not-found

Note:

    Must confirm there is no business data before deleting PVC/DataVolume.
    Production environments cannot directly delete resources using experimental cleanup methods.

---

## Forty, Summary of This Article

KubeVirt LiveMigration is an important advanced operation capability in KubeVirt.

Core understanding:

    1. Cold migration is a downtime migration
    2. LiveMigration is a live migration
    3. Volume Migration is disk migration
    4. LiveMigration relies on KVM, virt-handler, virt-launcher, shared storage, and network
    5. The migration process creates a VirtualMachineInstanceMigration object
    6. After successful migration, the VMI's node changes
    7. evictionStrategy: LiveMigrate can be used in node drain scenarios
    8. RWX PVC is more suitable for LiveMigration experiments
    9. RWO, local disks, device passthrough, and multi-NIC special scenarios may limit migration
    10. Migration drills must be conducted before production deployment

Core chain:

    VM
      |
      v
    VMI
      |
      v
    VirtualMachineInstanceMigration
      |
      v
    Source virt-launcher Pod
      |
      v
    Target virt-launcher Pod
      |
      v
    VMI node switch

Core commands:

    virtctl migrate vm-live-migrate-demo -n kubevirt-migration-demo

    kubectl -n kubevirt-migration-demo get virtualmachineinstancemigrations

    kubectl -n kubevirt-migration-demo describe virtualmachineinstancemigration <migration-name>

    kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o wide

    kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher

    kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo

Experience judgment:

1. Before migration, check if the VMI is LiveMigratable  
2. If migration fails, prioritize checking the Migration object and Events  
3. Storage not meeting requirements is one of the most common issues  
4. Insufficient resources on the target node is also common  
5. If a node has migration issues, check virt-handler and /dev/kvm  
6. Before drain triggers migration, confirm evictionStrategy  
7. In production, do not treat LiveMigration as a universal default capability; it must be validated based on actual storage and network  

Subsequent expansion:  

    12-KubeVirt Volume Migration Getting Started: Virtual Machine Disk Migration, PVC, and StorageClass Changes.md