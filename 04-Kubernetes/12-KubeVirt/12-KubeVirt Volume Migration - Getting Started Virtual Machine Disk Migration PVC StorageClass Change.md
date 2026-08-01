# 12-KubeVirt Volume Migration: Getting Started with Virtual Machine Disk Migration, PVC, and StorageClass Changes

Recommended Path:

    04-Kubernetes/12-KubeVirt/12-KubeVirt Volume Migration: Getting Started with Virtual Machine Disk Migration, PVC, and StorageClass Changes.md

Tags:

    #Kubernetes
    #KubeVirt
    #VolumeMigration
    #StorageMigration
    #PVC
    #DataVolume
    #StorageClass
    #LiveMigration
    #VirtualDiskMigration
    #CloudlandVirtualization
    #PlatformEngineering

---

## I. Document Overview

This document records the basic concepts, experimental design, operational procedures, and troubleshooting methods of KubeVirt Volume Migration.

Volume Migration can be understood as:

    Migrating the virtual machine's disk volume from an old PVC/DataVolume to a new PVC/DataVolume during VM runtime.

It is commonly used for:

    1. The old StorageClass is about to be deprecated
    2. Need to migrate VM disks to a new storage backend
    3. Need to migrate VM disks to a higher-performance StorageClass
    4. Need to expand disk capacity and migrate to a new PVC
    5. Need to switch from an old PVC to a new PVC
    6. Platform storage transformation or storage pool migration

Note:

    Volume Migration is an advanced capability of KubeVirt.
    It depends on LiveMigration capability.
    It is not a simple PVC copy.
    It is not a regular Kubernetes PVC migration tool.
    It is also not a complete cross-cluster migration solution.

Document Objectives:

    1. Understand the difference between Volume Migration and LiveMigration
    2. Understand the role of updateVolumesStrategy: Migration
    3. Understand why the target PVC needs to be pre-created
    4. Create the source PVC and start the VM
    5. Create the target PVC
    6. Update the VM volume definition to trigger migration
    7. Observe VirtualMachineInstanceMigration
    8. Validate the VM disk migration results
    9. Learn the approach for migration cancellation and failure recovery
    10. Organize interview answers

Applicable Environment:

    Ubuntu 22.04
    Kubernetes v1.31.14
    KubeVirt is installed
    CDI is installed
    virtctl is installed
    At least two worker nodes supporting KVM
    Current KubeVirt version supports updateVolumesStrategy / Volume Migration

Important Reminder:

    If the current KubeVirt version does not support updateVolumesStrategy, this document serves as conceptual reference and experimental guide after upgrade.
    Before practical operations, verify whether the cluster API supports this field.

---

## II. First Confirm Current Version Support

Execute:

    kubectl explain vm.spec.updateVolumesStrategy

If you can see the field explanation, it indicates the API supports this field.

You can also check:

    kubectl explain virtualmachine.spec.updateVolumesStrategy

If it reports:

    field "updateVolumesStrategy" does not exist

It indicates the current version may not support this capability, or the CRD version is mismatched.

In this case:

    1. Do not forcibly copy the experimental YAML
    2. Upgrade or replace to a KubeVirt version that supports this capability first
    3. Or keep this document as theoretical notes only

---

## III. Difference Between Volume Migration and LiveMigration

### 3.1 LiveMigration

LiveMigration migrates:

    The location where the VM runs

In other words:

    VM moves from Node A to Node B

Core focus:

    1. VM runtime status
    2. Memory state
    3. Target node resources
    4. KVM capabilities
    5. Migration network
    6. virt-handler
    7. virt-launcher

Typical commands:

    virtctl migrate <vm-name> -n <namespace>

---

### 3.2 Volume Migration

Volume Migration migrates:

    The disk volume used by the VM

In other words:

    VM switches from an old PVC to a new PVC

Core focus:

    1. Source PVC
    2. Target PVC
    3. StorageClass
    4. Disk capacity
    5. DataVolume
    6. VM volume definition
    7. updateVolumesStrategy
    8. VirtualMachineInstanceMigration during migration
    9. VM state consistency

Volume Migration is not directly executed via a virtctl volume-migrate command.

It is more declarative:

    Update VM spec
        |
        v
    Set updateVolumesStrategy: Migration
        |
        v
    Change volume from old PVC to new PVC
        |
        v
    KubeVirt triggers storage migration

---

## IV. Core Chain of Volume Migration

Core chain:

    Source PVC
      |
      v
    VM current disk
      |
      v
    Modify VM spec
      |
      v
    updateVolumesStrategy: Migration
      |
      v
    Target PVC
      |
      v
    VirtualMachineInstanceMigration
      |
      v
    Data replication / migration
      |
      v
    VM switches to target PVC

More specifically: /think

VirtualMachine
        |
        v
    updateVolumesStrategy: Migration
        |
        v
    template.spec.volumes From old-pvc Replace with new-pvc
        |
        v
    KubeVirt Other Organiser
        |
        v
    Move data from old to new volume
        |
        v
    After migration successful VM Use new PVC

---

## Five. Why the Target PVC Must Be Pre-created

Volume Migration itself does not handle complete storage planning.

You need to prepare the target volume in advance:

    1. Target PVC
    2. Or target DataVolume
    3. Or new dataVolumeTemplates

The target PVC must meet:

    1. Capacity not less than source PVC
    2. StorageClass available
    3. Access mode meets scenario requirements
    4. volumeMode matches expectations
    5. Backend storage is healthy
    6. Can be used by KubeVirt

Common errors:

    Only modifying VM spec without pre-creating target PVC

Result:

    VM fails to find target volume after update, migration fails or enters abnormal state

---

## Six. Scenarios Suitable for Volume Migration

Suitable for:

    1. Migrating from old StorageClass to new StorageClass
    2. Migrating from low-performance storage to high-performance storage
    3. Migrating from temporary storage to formal storage
    4. Storage backend upgrade replacement
    5. VM disk expansion with volume change
    6. Platform storage restructuring
    7. Migrating experimental VM disks to production storage pool
    8. Migrating VM data out of deprecated storage pool

Not suitable for:

    1. KubeVirt versions without this feature
    2. Unverified LiveMigration environments
    3. Unstable backend for source or target volume
    4. Production critical VMs without backup
    5. Multi-write shared disks
    6. Device passthrough VMs
    7. VMs using complex special disk types
    8. Production environments without maintenance window and rollback plan

---

## Seven. Common Limitations of Volume Migration

Remember in the initial stage:

    1. Not all volume types support migration
    2. Typically supports migration between PVC and DataVolume
    3. Target volume capacity should not be less than source volume
    4. Extra expanded capacity won't automatically take effect in Guest OS
    5. Guest OS may need to expand partition and file system
    6. Volume Migration depends on LiveMigration
    7. Migration usually requires source and target nodes participation
    8. Same-node storage live migration may not meet expectations
    9. Failure may require cancellation or manual recovery
    10. VM shutdown or VMI disappearance may trigger ManualRecoveryRequired

Not recommended for:

    1. shareable disk
    2. hotplug volume
    3. filesystem / virtiofs type shared directories
    4. lun type special disks
    5. hostDisk
    6. local disk
    7. GPU / PCI passthrough scenarios
    8. Unverified complex production VM scenarios

---

## Eight. Experiment Objectives

This document's experiment objectives:

    1. Create experimental namespace
    2. Create source DataVolume / PVC
    3. Start VM using source PVC
    4. Write test data inside VM
    5. Create target PVC
    6. Modify VM spec to trigger Volume Migration
    7. Observe VirtualMachineInstanceMigration
    8. Verify if VM switches to target PVC
    9. Verify if test data still exists
    10. Learn migration cancellation and failure recovery

Experimental VM:

    vm-volume-migration-demo

Experimental namespace:

    kubevirt-volume-migration-demo

Source PVC:

    source-rootdisk

Target PVC:

    target-rootdisk

Source StorageClass:

    longhorn

Target StorageClass:

    longhorn-rwx or other new StorageClass

Note:

    If your experimental environment doesn't have longhorn-rwx, replace it with an available target StorageClass.
    If current version doesn't support this feature, only read the concept and troubleshooting sections.

---

## Nine. Pre-Experiment Checks

### 9.1 Check KubeVirt

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

### 9.2 Check Field Support

Execute:

    kubectl explain vm.spec.updateVolumesStrategy

If the field doesn't exist, stop hands-on practice.

---

### 9.3 Check LiveMigration Configuration

View KubeVirt CR:

    kubectl -n kubevirt get kubevirt kubevirt -o yaml | grep -A80 -i workloadUpdateStrategy

Confirm at least supports:

    workloadUpdateMethods:
    - LiveMigrate

If not, need to confirm based on current KubeVirt version.

Example configuration, do not directly copy for production:

    kubectl -n kubevirt patch kubevirt kubevirt --type=merge \
      -p '{"spec":{"configuration":{"vmRolloutStrategy":"LiveUpdate"},"workloadUpdateStrategy":{"workloadUpdateMethods":["LiveMigrate"]}}}'

If version requires feature gate, also need to confirm: /think

# VolumesUpdateStrategy

View:

    kubectl -n kubevirt get kubevirt kubevirt -o yaml | grep -A20 featureGates

Notes:

    The fields may vary across different KubeVirt versions.
    Use `kubectl explain` and current CRD as reference.

---

### 9.4 Check CDI

Execute:

    kubectl -n cdi get pods -o wide

    kubectl get dv -A

---

### 9.5 Check StorageClass

Execute:

    kubectl get storageclass

Confirm both source and target StorageClass exist.

Example:

    longhorn
    longhorn-rwx

---

### 9.6 Check Node KVM

Execute on Worker node:

    egrep -c '(vmx|svm)' /proc/cpuinfo

    ls -l /dev/kvm

    sudo kvm-ok

---

## Ten. Create Experimental Namespace

Create:

    kubectl create namespace kubevirt-volume-migration-demo --dry-run=client -o yaml | kubectl apply -f -

Create directory:

    mkdir -p /root/k8s-yaml/kubevirt/volume-migration-demo

    cd /root/k8s-yaml/kubevirt/volume-migration-demo

---

## Eleven. Experiment One: Create Source DataVolume

Create source DataVolume:

    cat <<EOF > dv-source-rootdisk.yaml
    apiVersion: cdi.kubevirt.io/v1beta1
    kind: DataVolume
    metadata:
      name: source-rootdisk
      namespace: kubevirt-volume-migration-demo
    spec:
      source:
        http:
          url: "https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"
      pvc:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 2Gi
        storageClassName: longhorn
    EOF

Apply:

    kubectl apply -f dv-source-rootdisk.yaml

View:

    kubectl -n kubevirt-volume-migration-demo get dv

    kubectl -

View importer logs:

    kubectl -n kubevirt-volume-migration-demo get pods

    kubectl -n kubevirt-volume-migration-demo logs <importer-pod-name> --tail=100

Wait:

    kubectl -n kubevirt-volume-migration-demo get dv source-rootdisk

Expected:

    PHASE: Succeeded

PVC Expected:

    source-rootdisk Bound

If public download fails, switch to internal HTTP mirror address:

    http://10.0.0.10/images/cirros-0.6.2-x86_64-disk.img

---

## Twelve. Experiment Two: Create VM Using Source PVC

Note:

    The VM does not directly reference dataVolume.
    To make subsequent volume updates more intuitive, this uses persistentVolumeClaim to reference the source PVC.

Create VM: /think

```bash
cat <<EOF > vm-volume-migration-demo.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-volume-migration-demo
  namespace: kubevirt-volume-migration-demo
  labels:
    app: vm-volume-migration-demo
spec:
  runStrategy: Manual
  template:
    metadata:
      labels:
        app: vm-volume-migration-demo
        kubevirt.io/domain: vm-volume-migration-demo
    spec:
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
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
            ports:
            - name: ssh
              port: 23
      networks:
      - name: default
        pod: {}
      volumes:
      - name: rootdisk
        persistentVolumeClaim:
          claimName: source-rootdisk
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

**Apply:**

```bash
kubectl apply -f vm-volume-migration-demo.yaml
```

**Start:**

```bash
virtctl start vm-volume-migration-demo -n kubevirt-volume-migration-demo
```

**Check:**

```bash
kubectl -n kubevirt-volume-migration-demo get vm
kubectl -n kubevirt-volume-migration-demo get vmi -o wide
kubectl -n kubevirt-volume-migration-demo get pods -o wide | grep virt-launcher
```

---

## Thirteen. Experiment Three: Writing Test Data Inside VM

**Enter console:**

```bash
virtctl console vm-volume-migration-demo -n kubevirt-volume-migration-demo
```

**Login:**

```
Username: cirros
Password: kubevirt
```

**Write test data:**

```bash
echo "before volume migration" > /home/cirros/migration-test.txt
```

**Check:**

```bash
cat /home/cirros/migration-test.txt
```

**Check disk:**

```bash
df -h
mount
```

**Exit:**

```
Ctrl + ]
```

**Note:** This step is for verifying data persistence after migration.

---

## Fourteen. Experiment Four: Creating Target PVC

The target PVC must be created in advance.

**Create target PVC:**

```bash
cat <<EOF > pvc-target-rootdisk.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: target-rootdisk
  namespace: kubevirt-volume-migration-demo
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 4Gi
  storageClassName: longhorn-rwx
EOF
```

If your environment doesn't have `longhorn-rwx`, replace:

```yaml
storageClassName: <your target StorageClass>
```

If the target storage only supports RWO, confirm that your KubeVirt version and backend support this migration scenario.

**Apply:**

```bash
kubectl apply -f pvc-target-rootdisk.yaml
```

**Check:**

```bash
kubectl -n kubevirt-volume-migration-demo get pvc
```

**Expected:**

```
target-rootdisk Bound
```

**Key verifications:**
1. `target-rootdisk` capacity ≥ `source-rootdisk`
2. `target-rootdisk` is Bound
3. StorageClass is normal
4. Backend storage is healthy

If `target-rootdisk` is Pending, do not update VM yet.

---

## Fifteen. Experiment Five: Updating VM to Trigger Volume Migration

Create updated VM YAML:

```bash
cat <<EOF > vm-volume-migration-demo-target.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-volume-migration-demo
  namespace: kubevirt-volume-migration-demo
  labels:
    app: vm-volume-migration-demo
spec:
  updateVolumesStrategy: Migration
  runStrategy: Manual
  template:
    metadata:
      labels:
        app: vm-volume-migration-demo
        kubevirt.io/domain: vm-volume-migration-demo
    spec:
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
              bus: virtio
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
        persistentVolumeClaim:
          claimName: target-rootdisk
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

**Apply:**

```bash
kubectl apply -f vm-volume-migration-demo-target.yaml
```

**Explanation:**

The key changes are:

1. `spec.updateVolumesStrategy: Migration`
2. `rootdisk` from `source-rootdisk` to `target-rootdisk`

KubeVirt will attempt to execute volume migration based on VM spec changes.

---

## Sixteen. Experiment Six: Observing the Migration Process

**Check VM:**

```bash
kubectl -n kubevirt-volume-migration-demo get vm vm-volume-migration-demo
```

**Check VMI:**

```bash
kubectl -n kubevirt-volume-migration-demo get vmi vm-volume-migration-demo -o wide
```

**Check migration objects:**

```bash
kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations
```

**Try filtering with labels:**

```bash
kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations \
  -l kubevirt.io/volume-update-in-progress=vm-volume-migration-demo
```

If the actual version uses different labels, you can view all:

```bash
kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations --show-labels
```

**View details:**

```bash
kubectl -n kubevirt-volume-migration-demo describe virtualmachineinstancemigration <migration-name>
```

**Observe VMI migrationState:**

```bash
kubectl -n kubevirt-volume-migration-demo get vmi vm-volume-migration-demo -o yaml | grep -A100 -i migration
```

**Observe VM volumeMigrationState:**

```bash
kubectl -n kubevirt-volume-migration-demo get vm vm-volume-migration-demo -o yaml | grep -A100 -i volume
```

**Observe Pod:**

```bash
kubectl -n kubevirt-volume-migration-demo get pods -o wide | grep virt-launcher
```

**Recommended to open two windows:**

**Window one:**

```bash
watch -n 1 "kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations"
```

**Window two:**

```bash
watch -n 1 "kubectl -n kubevirt-volume-migration-demo get vmi vm-volume-migration-demo -o wide"
```

## 17. Experiment 7: Verifying Migration Results

After migration is complete, check:

    kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations

Check VM YAML:

    kubectl -n kubevirt-volume-migration-demo get vm vm-volume-migration-demo -o yaml | grep -A20 volumes

Confirm rootdisk has been redirected to:

    target-rootdisk

Check PVC:

    kubectl -n kubevirt-volume-migration-demo get pvc

Source PVC:

    source-rootdisk

Target PVC:

    target-rootdisk

Enter console:

    virtctl console vm-volume-migration-demo -n kubevirt-volume-migration-demo

After logging in, check test file:

    cat /home/cirros/migration-test.txt

Expected output:

    before volume migration

If the test file still exists, it indicates data has been migrated from the source volume to the target volume, and the VM can continue to be used.

Check disk capacity:

    df -h

Note:

    Even if the target PVC changes from 2Gi to 4Gi, the Guest OS may not immediately show 4Gi.
    Expansion of the partition and filesystem must be performed inside the Guest OS.
    CirrOS is not suitable for full expansion demonstrations; Ubuntu/Rocky Linux can be used for separate practice later.

---

## 18. Experiment 8: Post-Migration Expansion Notes

If the target PVC is larger than the source PVC, Kubernetes has already been updated to a larger capacity:

    kubectl -n kubevirt-volume-migration-demo get pvc target-rootdisk

However, the Guest OS may not automatically see the newly added space.

In a real Linux VM, typically you need:

    1. Check the disk
    2. Expand the partition
    3. Expand the filesystem

Common commands:

    lsblk

    growpart /dev/vda 1

    resize2fs /dev/vda1

Or for XFS:

    xfs_growfs /

Note:

    Commands vary by image, partition table, and filesystem.
    Backups must be performed before expansion in production.
    Do not execute growpart or resize2fs without confirming the disk device.

---

## 19. Volume Migration Cancellation Approach

Volume Migration is a declarative update.

If cancellation is needed during migration, the usual approach is:

    Restore the VM's original volume definition.

This means reverting the VM spec back to:

    rootdisk -> source-rootdisk

And removing or restoring the corresponding updateVolumesStrategy configuration.

For example, reapply the original VM YAML:

    kubectl apply -f vm-volume-migration-demo.yaml

Check:

    kubectl -n kubevirt-volume-migration-demo get vm vm-volume-migration-demo -o yaml | grep -A20 volumes

Note:

    Cancellation is not simply deleting the migration Pod.
    Cancellation should restore the original volume set from the VM spec level.

---

## 20. Volume Migration Failure Recovery

If migration fails, KubeVirt may continue to retry.

Check:

    kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations

    kubectl -n kubevirt-volume-migration-demo describe virtualmachineinstancemigration <migration-name>

Check VM:

    kubectl -n kubevirt-volume-migration-demo describe vm vm-volume-migration-demo

If:

    ManualRecoveryRequired

Appears, the VM may be in a state requiring manual recovery.

Common causes:

    1. VM was shut down during migration
    2. VMI was deleted
    3. virt-launcher Pod disappeared abnormally
    4. Target volume data replication was incomplete
    5. VM spec switched to target volume, but target volume data is incomplete
    6. Migration interruption caused inconsistent state

Handling approach:

    1. Do not rush to delete PVCs
    2. First confirm if the source PVC is still intact
    3. Check migratedVolumes information in VM status
    4. Check migrationState
    5. Check if target PVC has completed replication
    6. If confirmed incomplete, restore VM spec to source PVC
    7. If confirmed target volume is complete, handle updateVolumesStrategy field cautiously according to official recommendations
    8. Backups must be performed before handling in production environments

Basic approach to restore to source PVC:

    kubectl apply -f vm-volume-migration-demo.yaml

If needing to remove updateVolumesStrategy:

    kubectl -n kubevirt-volume-migration-demo patch vm vm-volume-migration-demo --type='json' \
      -p='[{"op": "remove", "path": "/spec/updateVolumesStrategy"}]'

Note:

    The above patch should only be executed after confirming the current state is suitable.
    Do not execute blindly in production environments.
    ManualRecoveryRequired scenarios must be judged based on VM status, source volume, target volume, and migration object state.

---

## 21. Common Issue 1: updateVolumesStrategy Field Not Exists

Phenomenon:

    kubectl explain vm.spec.updateVolumesStrategy

Error:

    field does not exist

Causes:

    1. KubeVirt version does not support it
    2. CRD has not been updated
    3. The current cluster has not enabled related capabilities

Solution:

1. Do not proceed with hands-on operations  
2. Confirm KubeVirt version  
3. Check official version notes  
4. Test upgrade in a test environment  
5. Do not force patch unsupported fields in production  

---

## 22. Common Issue Two: Target PVC Pending  

Check:  

    kubectl -n kubevirt-volume-migration-demo describe pvc target-rootdisk  

Common causes:  

    1. StorageClass does not exist  
    2. Provisioner abnormality  
    3. Storage capacity insufficient  
    4. RWX backend not configured  
    5. Longhorn share-manager abnormality  
    6. CephFS / NFS / CSI abnormality  

Handling:  

    kubectl get storageclass  

    kubectl -n longhorn-system get pods -o wide  

    kubectl -n longhorn-system get volumes.longhorn.io  

    kubectl -n storage-system get pods -o wide  

Do not update VM before the target PVC is Bound to trigger migration.  

---

## 23. Common Issue Three: Target PVC Smaller than Source PVC  

Phenomenon:  

    Migration failure  
    Or data replication anomaly  

Cause:  

    Target PVC capacity is smaller than source PVC.  

Handling:  

    1. Delete the erroneous target PVC  
    2. Recreate a larger target PVC  
    3. Target capacity must be at least equal to source capacity  
    4. Trigger Volume Migration again  

Check source PVC:  

    kubectl -n kubevirt-volume-migration-demo get pvc source-rootdisk  

Check target PVC:  

    kubectl -n kubevirt-volume-migration-demo get pvc target-rootdisk  

---

## 24. Common Issue Four: VirtualMachineInstanceMigration Failed  

Check:  

    kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations  

    kubectl -n kubevirt-volume-migration-demo describe virtualmachineinstancemigration <migration-name>  

Common causes:  

    1. Target node resource insufficient  
    2. LiveMigration configuration not met  
    3. PVC does not meet migration requirements  
    4. Source volume or target volume abnormality  
    5. virt-handler abnormality  
    6. Source node or target node KVM abnormality  
    7. Migration network abnormality  
    8. VM high load causing migration not to converge  

Continue troubleshooting:  

    kubectl -n kubevirt-volume-migration-demo describe vmi vm-volume-migration-demo  

    kubectl -n kubevirt get pods -o wide | grep virt-handler  

    kubectl -n kubevirt logs <virt-handler-pod-name> --tail=200  

---

## 25. Common Issue Five: Migration Completed but VM Disk Capacity Unchanged  

Cause:  

    Target PVC expansion only affects Kubernetes storage layer.  
    Guest OS partition and filesystem do not automatically expand.  

Handling:  

    1. Enter VM  
    2. lsblk to check disk  
    3. Expand partition  
    4. Expand filesystem  
    5. df -h to verify  

Note:  

    This issue is not a Volume Migration failure.  
    It is due to incomplete Guest OS disk expansion steps.  

---

## 26. Common Issue Six: VM Closed During Migration  

Risk:  

    May cause VM spec and disk migration state inconsistency.  

Phenomenon:  

    VM shows ManualRecoveryRequired  
    VM cannot start  
    Migration object Failed  
    Target volume data may be incomplete  

Handling principles:  

    1. Do not delete source PVC  
    2. Do not delete target PVC  
    3. Check VM status first  
    4. Check migratedVolumes  
    5. Check migrationState  
    6. Determine if target volume is complete  
    7. Restore VM spec to source PVC if necessary  
    8. Clean up failed target volume last  

---

## 27. Common Issue Seven: DataVolumeTemplate Inconsistency  

If VM uses dataVolumeTemplates to create the system disk, be especially cautious during migration.  

Because:  

    Volume references DataVolume name  
    DataVolumeTemplate defines DataVolume name  

Must remain consistent.  

Common errors:  

    Only modify volumes  
    No change to dataVolumeTemplates  

Or:  

    Only modify dataVolumeTemplates  
    No change to volumes  

Result:  

    VM spec inconsistency  
    Migration fails to execute as expected  

Entry suggestions:  

    When starting with Volume Migration, first use persistentVolumeClaim to reference existing PVC.  
    Do not mix dataVolumeTemplates initially.  
    Only perform DataVolumeTemplate update experiments after becoming familiar.  

---

## 28. Standard Troubleshooting Path  

When Volume Migration fails, troubleshoot in the following order:

1. kubectl explain vm.spec.updateVolumesStrategy  
2. kubectl get vm  
3. kubectl describe vm  
4. kubectl get vmi  
5. kubectl describe vmi  
6. kubectl get virtualmachineinstancemigrations  
7. kubectl describe virtualmachineinstancemigration  
8. kubectl get pvc  
9. kubectl describe pvc source-rootdisk  
10. kubectl describe pvc target-rootdisk  
11. kubectl get pv  
12. kubectl describe pv  
13. kubectl -n kubevirt logs deploy/virt-controller  
14. kubectl -n kubevirt get pods -o wide | grep virt-handler  
15. kubectl -n kubevirt logs <virt-handler-pod>  
16. View source and target nodes kubelet Log  
17. Inspection KVM, storage, network  

---

## Twenty-Nine, Standard Command List  

### 29.1 API Capability Check  

    kubectl explain vm.spec.updateVolumesStrategy  

    kubectl -n kubevirt get kubevirt kubevirt -o yaml | grep -A80 -i workloadUpdateStrategy  

    kubectl -n kubevirt get kubevirt kubevirt -o yaml | grep -A20 featureGates  

---

### 29.2 VM / VMI  

    kubectl -n kubevirt-volume-migration-demo get vm  

    kubectl -n kubevirt-volume-migration-demo describe vm vm-volume-migration-demo  

    kubectl -n kubevirt-volume-migration-demo get vmi -o wide  

    kubectl -n kubevirt-volume-migration-demo describe vmi vm-volume-migration-demo  

---

### 29.3 Migration Objects  

    kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations  

    kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations --show-labels  

    kubectl -n kubevirt-volume-migration-demo describe virtualmachineinstancemigration <migration-name>  

---

### 29.4 PVC / PV  

    kubectl -n kubevirt-volume-migration-demo get pvc  

    kubectl -n kubevirt-volume-migration-demo describe pvc source-rootdisk  

    kubectl -n kubevirt-volume-migration-demo describe pvc target-rootdisk  

    kubectl get pv  

    kubectl describe pv <pv-name>  

---

### 29.5 KubeVirt Components  

    kubectl -n kubevirt get pods -o wide  

    kubectl -n kubevirt logs deploy/virt-controller --tail=200  

    kubectl -n kubevirt logs deploy/virt-api --tail=200  

    kubectl -n kubevirt get pods -o wide | grep virt-handler  

    kubectl -n kubevirt logs <virt-handler-pod-name> --tail=200  

---

### 29.6 Nodes  

    kubectl get nodes -o wide  

    kubectl describe node <node-name>  

Node native:  

    ls -l /dev/kvm  

    egrep -c '(vmx|svm)' /proc/cpuinfo  

    systemctl status kubelet --no-pager  

    systemctl status containerd --no-pager  

    journalctl -u kubelet -n 200 --no-pager  

---

## Thirty, Production Environment Notes  

Before executing Volume Migration in production environment, must confirm:  

    1. Current KubeVirt version supports this capability  
    2. Verified in test environment  
    3. VM has complete backup  
    4. Source PVC will not be deleted prematurely  
    5. Target PVC is already Bound  
    6. Target PVC capacity is not less than source PVC  
    7. Target StorageClass is stable  
    8. VM can LiveMigration  
    9. Source and target nodes have sufficient resources  
    10. Rollback plan for migration failure  
    11. Has maintenance window  
    12. Do not stop VM during migration  
    13. Do not delete migration objects, virt-launcher, PVC  
    14. Plan old PVC cleanup after migration completes  

Prohibited:  

    1. Migrate core VM without backup  
    2. Delete source PVC during migration  
    3. Delete target PVC during migration  
    4. Force delete VMI during migration  
    5. Drain source node during migration  
    6. Blindly delete resources after migration failure  
    7. Delete old volumes without confirming data integrity  

---

## Thirty-One, Interview Answer: What is KubeVirt Volume Migration  

You can answer: /think

# KubeVirt Volume Migration is the capability for virtual machine disk migration.
It can migrate the disk used by a VM from an old PVC or DataVolume to a new PVC or DataVolume while the VM is running.
Common scenarios include the retirement of an old StorageClass, storage backend upgrades, migrating disks to higher-performance storage, or migrating to a larger PVC.
It is triggered by updating the VM spec and setting updateVolumesStrategy: Migration. KubeVirt will create a migration process to transfer data from the old volume to the new volume.
This capability depends on LiveMigration, PVC, StorageClass, and underlying storage capabilities, so validation is required before production use.

---

## 32. Interview Answer: Difference between Volume Migration and LiveMigration

You can answer like this:

LiveMigration primarily migrates the runtime location of a VM, such as moving it from one node to another.
Volume Migration primarily migrates the disk volume used by a VM, such as migrating from an old PVC to a new PVC, or from an old StorageClass to a new StorageClass.
Volume Migration also relies on LiveMigration mechanisms, so it's not just a storage issue—it also involves node resources, KVM, virt-handler, virt-launcher, and networking.
In short, LiveMigration migrates the runtime location, while Volume Migration migrates the disk volume.

---

## 33. Interview Answer: How to troubleshoot Volume Migration failure

You can answer like this:

I will first confirm if the current KubeVirt version supports updateVolumesStrategy.
Then check the status of VM and VMI to confirm if the VM is still Running.
Next, check the VirtualMachineInstanceMigration object and Events to determine at which stage the migration failed.
Focus on the source PVC, target PVC, PV, StorageClass, and backend storage status from the storage side.
If the migration process failed, also check the logs of virt-controller, virt-handler, virt-launcher, and the resource status of the source and target nodes.
If ManualRecoveryRequired appears, do not blindly delete the PVC. Instead, first determine if the data of the source and target volumes is complete, then restore the VM spec to a consistent state.

---

## 34. Clean up experimental resources

Stop VM:

virtctl stop vm-volume-migration-demo -n kubevirt-volume-migration-demo

Delete VM:

kubectl delete -f vm-volume-migration-demo-target.yaml --ignore-not-found

kubectl delete -f vm-volume-migration-demo.yaml --ignore-not-found

Check VMI:

kubectl -n kubevirt-volume-migration-demo get vmi

Check PVC:

kubectl -n kubevirt-volume-migration-demo get pvc

Delete PVC/DataVolume after confirming data is no longer needed:

kubectl delete -f dv-source-rootdisk.yaml --ignore-not-found

kubectl delete -f pvc-target-rootdisk.yaml --ignore-not-found

If PVC still remains:

kubectl -n kubevirt-volume-migration-demo delete pvc source-rootdisk --ignore-not-found

kubectl -n kubevirt-volume-migration-demo delete pvc target-rootdisk --ignore-not-found

Delete namespace:

kubectl delete namespace kubevirt-volume-migration-demo

Note:

Experimental environments can be cleaned.
Production environments cannot directly delete resources using experimental cleanup methods.
Before deleting PVCs, confirm that data is no longer needed.

---

## 35. Summary of this article

KubeVirt Volume Migration is an advanced capability for virtual machine disk migration in KubeVirt.

Core understanding:

1. LiveMigration migrates the runtime location of a VM
2. Volume Migration migrates the disk volume used by a VM
3. Volume Migration is triggered by updateVolumesStrategy: Migration
4. Target PVC/DataVolume must be pre-prepared
5. Target volume capacity should be no smaller than the source volume
6. Updating the volumes in VM spec triggers migration
7. Migration progress can be observed via VirtualMachineInstanceMigration
8. After migration, the VM uses the new PVC
9. Guest OS internal expansion requires separate handling
10. Migration failure may require cancellation or ManualRecovery

Core chain:

Source PVC
|
v
VM rootdisk
|
v
updateVolumesStrategy: Migration
|
v
Target PVC
|
v
VirtualMachineInstanceMigration
|
v
Data migration completed
|
v
VM uses target PVC

Core commands:

kubectl explain vm.spec.updateVolumesStrategy

kubectl apply -f vm-volume-migration-demo-target.yaml

kubectl get virtualmachineinstancemigrations -n kubevirt-volume-migration-demo

kubectl describe virtualmachineinstancemigration <migration-name> -n kubevirt-volume-migration-demo

```bash
kubectl get vm vm-volume-migration-demo -n kubevirt-volume-migration-demo -o yaml

kubectl get pvc -n kubevirt-volume-migration-demo
```

Experience Judgment:

1. When the current version does not support updateVolumesStrategy, do not force experimentation
2. Do not update the VM to trigger migration before the target PVC is Bound
3. The target PVC capacity must be no smaller than the source PVC
4. When migration fails, first check VMIM and Events
5. Do not rush to delete PVCs after storage migration fails
6. ManualRecoveryRequired requires careful handling
7. Must perform backups and drills before production migration

Here we go.KubeVirt The introductory to advanced basic series has been covered:

01 Introduction Concepts  
02 Core Architecture  
03 Pre-Installation Preparation  
04 KubeVirt Installation  
05 First VM  
06 CDI / DataVolume  
07 Storage Basics  
08 Network Basics  
09 Operations Troubleshooting  
10 Interview Preparation  
11 LiveMigration  
12 Volume Migration