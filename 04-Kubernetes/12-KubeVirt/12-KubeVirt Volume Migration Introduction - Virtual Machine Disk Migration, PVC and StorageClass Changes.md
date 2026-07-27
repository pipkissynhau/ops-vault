# 12-KubeVirt Volume Migration Introduction: Virtual Machine Disk Migration, PVC and StorageClass Changes

Recommended Path:

    04-Kubernetes/12-KubeVirt/12-KubeVirt Volume Migration Introduction: Virtual Machine Disk Migration, PVC and StorageClass Changes.md

Tags:

    #Kubernetes
    #KubeVirt
    #VolumeMigration
    #StorageMigration
    #PVC
    #DataVolume
    #StorageClass
    #LiveMigration
    #Virtual Machine Disk Migration
    #Cloud-Native Virtualization
    #Platform Engineering

---

## I. Document Description

This document records the basic concepts, experimental design, operation process, and troubleshooting methods of KubeVirt Volume Migration.

Volume Migration can be understood as:

    During the operation of a VM, migrating the disk volume used by the VM from an old PVC / DataVolume to a new PVC / DataVolume.

It is commonly used in:

    1. When an old StorageClass is about to be deprecated
    2. When it is necessary to migrate the VM disk to a new storage backend
    3. When it is necessary to migrate the VM disk to a higher-performance StorageClass
    4. When it is necessary to expand the disk capacity and migrate it to a new PVC
    5. When it is necessary to switch from an old PVC to a new PVC
    6. During platform storage reconfiguration or storage pool migration

Note:

    Volume Migration is an advanced feature of KubeVirt.
    It relies on the LiveMigration capability.
    It is not simply a copy of a PVC.
    It is not an ordinary Kubernetes PVC migration tool.
    It is also not a complete cross-cluster migration solution.

Objectives of this document:

    1. Understand the differences between Volume Migration and LiveMigration
    2. Understand the role of updateVolumesStrategy: Migration
    3. Understand why the target PVC needs to be created in advance
    4. Create a source PVC to start the VM
    5. Create a target PVC
    6. Update the VM volume definition to trigger migration
    7. Monitor VirtualMachineInstanceMigration
    8. Verify the results of the VM disk migration
    9. Learn about how to cancel and recover from migration failures
    10. Organize interview responses

Applicable Environment:

    Ubuntu 22.04
    Kubernetes v1.31.14
    KubeVirt installed
    CDI installed
    virtctl installed
    At least two Worker nodes that support KVM
    The current KubeVirt version supports updateVolumesStrategy / Volume Migration

Important Reminder:

    If the current KubeVirt version does not support updateVolumesStrategy, this document can be used as a reference for concepts and subsequent experiments after upgrading.
    Before conducting actual operations, it is necessary to verify whether the current cluster API supports this field.

---

## II. First, Confirm Whether the Current Version Supports It

Execute:

    kubectl explain vm.spec.updateVolumesStrategy

If you can see the field description, it means that the current API supports this field.

You can also check:

    kubectl explain virtualmachine/spec.updateVolumesStrategy

If an error is reported:

    field "updateVolumesStrategy" does not exist

This indicates that the current version may not support this capability, or the CRD version may not match.

In such cases:

    1. Do not forcibly copy the experimental YAML.
    2. First, upgrade or switch to a KubeVirt version that supports this capability.
    3. Or simply use this document as a theoretical reference.

---

## III. Differences between Volume Migration and LiveMigration

### 3.1 LiveMigration

LiveMigration migrates:

    The running location of the VM

In other words:

    The VM is moved from Node A to Node B

Key considerations:

    1. The running status of the virtual machine
    2. Memory status
    3. Resources of the target node
    4. KVM capabilities
    5. Migration network
    6. virt-handler
    7. virt-launcher

Typical command:

    virtctl migrate <vm-name> -n <namespace>

---

### 3.2 Volume Migration

Volume Migration migrates:

    The disk volume used by the VM

In other words:

    The VM is switched from an old PVC to a new PVC

Key considerations:

    1. Source PVC
    2. Target PVC
    3. StorageClass
    4. Disk capacity
    5. DataVolume
    6. VM volume definition
    7. updateVolumesStrategy
    8. VirtualMachineInstanceMigration during migration
    9. Consistency of the VM's status

Volume Migration does not simply execute a1. Create an experimental namespace.
2. Create a source DataVolume/PVC.
3. Launch a VM using the source PVC.
4. Write test data inside the VM.
5. Create a target PVC.
6. Modify the VM specification to trigger Volume Migration.
7. Observe the VirtualMachineInstanceMigration process.
8. Verify whether the VM has switched to the target PVC.
9. Confirm that the test data is still present.
10. Learn about how to cancel the migration and recover in case of failures.

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

longhorn-rwx or another available StorageClass

Note:

If your experimental environment does not have longhorn-rwx, you can replace it with a suitable target StorageClass.
If the current version does not support this feature, please only read the conceptual explanations and troubleshooting sections.```yaml
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

Application:

kubectl apply -f vm-volume-migration-demo.yaml

Startup:

virtctl start vm-volume-migration-demo -n kubevirt-volume-migration-demo

View:

kubectl -n kubevirt-volume-migration-demo get vm

kubectl -n kubevirt-volume-migration-demo get vmi -o wide

kubectl -n kubevirt-volume-migration-demo get pods -o wide | grep virt-launcher

---

## Experiment 13: Writing Test Data Inside the VM

Enter the console:

virtctl console vm-volume-migration-demo -n kubevirt-volume-migration-demo

Login:

Username: cirros
Password: kubevirt

Write test data:

echo "before volume migration" > /home/cirros/migration-test.txt

View:

cat /home/cirros/migration-test.txt

Check the disk:

df -h

Mount the disk:

Exit:

Ctrl + ]

Explanation:

This step is used to verify whether the system disk data is retained after the migration.

---

## Experiment 14: Creating the Target PVC

The target PVC must be created in advance.

Create the target PVC:

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

If your environment does not have longhorn-rwx, replace it with:

storageClassName: <your-target-storage-class>

If the target storage only supports RWO, confirm first whether the current KubeVirt version and backend support this migration scenario.

Application:

kubectl apply -f pvc-target-rootdisk.yaml

View:

kubectl -n kubevirt-volume-migration-demo get pvc

Expected result:

target-rootdisk is Bound

Key points to check:

1. The capacity of target-rootdisk is greater than or equal to that of source-rootdisk.
2. target-rootdisk is already Bound.
3. The StorageClass is correct.
4. The backend storage is healthy.

If target-rootdisk shows as Pending, do not update the VM yet.

---

## Experiment 15: Updating the VM to Trigger Volume Migration

Create the updated VM YAML:

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

Application:

kubectl apply -f vm-volume-migration-demo-target.yaml

Explanation:

There are two key changes:

1. spec.updateVolumesStrategy: Migration
2. rootdisk has been changed from source-rootdisk to target-rootdisk.

KubeVirt will attempt to execute the volume migration based on these changes in the VM spec.

---

## Experiment 16: Observing the Migration Process

View the VM:

kubectl -n kubevirt-volume-migration-demo get vm vm-volume-migration-demo

View the VMI:

kubectl -n kubevirt-volume-migration-demo get vmi vm-volume-migration-demo -o wide

View the migration objects:

kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations

Try filtering by labels:

kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations \
  -l kubevirt.io/volume-update-in-progress=vm-volume-migration-demo

Observe the `migrationState` of VMI:

    `kubectl -n kubevirt-volume-migration-demo get vmi vm-volume-migration-demo -o yaml | grep -A100 -i migration`

Observe the `volumeMigrationState` of the VM:

    `kubectl -n kubevirt-volume-migration-demo get vm vm-volume-migration-demo -o yaml | grep -A100 -i volume`

Check the Pods:

    `kubectl -n kubevirt-volume-migration-demo get pods -o wide | grep virt-launcher`

It is recommended to open two windows:

Window one:

    `watch -n 1 "kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations"`

Window two:

    `watch -n 1 "kubectl -n kubevirt-volume-migration-demo get vmi vm-volume-migration-demo -o wide"`

---

## Section Seventeen: Experiment Seven: Verifying Migration Results

After the migration is complete, check:

    `kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations`

View the VM YAML:

    `kubectl -n kubevirt-volume-migration-demo get vm vm-volume-migration-demo -o yaml | grep -A20 volumes`

Confirm that the `rootdisk` has been set to:

    `target-rootdisk`

Check the PVCs:

    `kubectl -n kubevirt-volume-migration-demo get pvc`
    Source PVC:
        `source-rootdisk`
    Target PVC:
        `target-rootdisk`

Enter the console:

    `virtctl console vm-volume-migration-demo -n kubevirt-volume-migration-demo`

After logging in, check the test file:

    `cat /home/cirros/migration-test.txt`

Expected result:

    `before volume migration`

If the test file still exists, it indicates that the data has been successfully migrated from the source volume to the target volume, and the VM can continue to be used.

Check the disk capacity:

    `df -h`

Note:

    Even if the target PVC size increases from 2Gi to 4Gi, the Guest OS may not immediately display the additional space.
    You will need to perform partition and file system expansion within the Guest OS.
    CirrOS is not suitable for conducting complete scale-out demonstrations; Ubuntu or Rocky Linux can be used instead for practice.

---

## Section Eighteen: Post-Migration Scale-Out Instructions

If the target PVC has a larger capacity than the source PVC, and Kubernetes already supports this size:

    `kubectl -n kubevirt-volume-migration-demo get pvc target-rootdisk`

However, the Guest OS may not automatically recognize the additional space.

In a real Linux VM, you typically need to follow these steps:

    1. Check the disk status.
    2. Expand the partition(s).
    3. Resize the file system accordingly.

Common commands include:

    `lsblk`
    `growpart /dev/vda 1`
    `resize2fs /dev/vda1`

For XFS, use:

    `xfs_growfs /`

Note:

    The specific commands may vary depending on the image used and the file system type.
    Always back up your data before performing any scale-out operations.
    Do not attempt to expand partitions or resize file systems without first verifying the disk status.

---

## Section Nineteen: Canceling Volume Migration

Volume Migration is a declarative process. If you need to cancel it, follow these steps:

    Restore the original volume definition for the VM.

    This means changing the `vm spec` back to use the `source-rootdisk`.

    Also, remove or restore the corresponding `updateVolumesStrategy` configuration.

    For example, reapply the original VM YAML:

    `kubectl apply -f vm-volume-migration-demo.yaml`

Check again:

    `kubectl -n kubevirt-volume-migration-demo get vm vm-volume-migration-demo -o yaml | grep -A20 volumes`

Remember:

    Canceling the migration does not simply involve deleting the related Pod.
    You must restore the original volume configuration at the VM spec level.

---

## Section Twenty: Recovering from a Failed Volume Migration

If the migration fails, KubeVirt may attempt to retry automatically. To check the status, use:

    `kubectl -n kubevirt-volume-migration-demo get virtualmachineinstancemigrations`
    `kubectl -n kubevirt-volume-migration-demo describe virtualmachineinstancemigration <migration-name>`

Check the status of the VM itself:

    `kubectl -n kubevirt-volume-migration-demo describe vm vm-volume-migration-demo`

If you encounter a message indicating `ManualRecoveryRequired`, it means that the VM may need to be manually restored. Possible causes include:

    1. The VM was shut down during the migration process.
    2. The VMI object was deleted.
    3. The `virt-launcher` Pod disappeared unexpectedly.
    4. The data replication to the target### 30. Precautions for Production Environments

Before performing Volume Migration in a production environment, it is essential to ensure the following:### Summary of This Article

KubeVirt Volume Migration is an advanced feature in KubeVirt that enables the migration of virtual machine disks. It allows a disk to be moved from an existing PVC or DataVolume to a new one while the VM remains running. Common use cases include upgrading storage backends, migrating disks to higher-performance storage solutions, or changing the size of the storage volume.

Here are some key points about this feature:

1. **LiveMigration** is required for moving the VM’s physical location, such as from one node to another.
2. Volume Migration specifically targets the disk volume used by the VM, allowing it to be transferred between different PVCs or StorageClasses.
3. The migration process is triggered by setting `updateVolumesStrategy: Migration` in the VM’s specification.
4. It is essential to ensure that the target PVC has sufficient capacity and is properly bound before starting the migration.
5. During migration, you can monitor the progress using the `VirtualMachineInstanceMigration` object and its related Events.
6. If the migration fails, there should be a backup plan in place for rolling back changes.

### Example Steps to Clean Up Experimental Resources

To clean up an experimental environment using KubeVirt Volume Migration, follow these steps:

1. **Stop the VM:**  
   ```bash
   virtctl stop vm-volume-migration-demo -n kubevirt-volume-migration-demo
   ```

2. **Delete the VM:**  
   ```bash
   kubectl delete -f vm-volume-migration-demo-target.yaml --ignore-not-found
   ```

3. **Check the VMI and PVC status:**  
   ```bash
   kubectl -n kubevirt-volume-migration-demo get vmi
   kubectl -n kubevirt-volume-migration-demo get pvc
   ```

4. **Delete unnecessary PVCs and DataVolumes:**  
   ```bash
   kubectl delete -f dv-source-rootdisk.yaml --ignore-not-found
   kubectl delete -f pvc-target-rootdisk.yaml --ignore-not-found
   ```

5. **If PVCs remain, delete them explicitly:**  
   ```bash
   kubectl -n kubevirt-volume-migration-demo delete pvc source-rootdisk --ignore-not-found
   kubectl -n kubevirt-volume-migration-demo delete pvc target-rootdisk --ignore-not-found
   ```

6. **Remove the namespace:**  
   ```bash
   kubectl delete namespace kubevirt-volume-migration-demo
   ```

It’s important to note that experimental resources can be cleared, but in a production environment, resources must be managed carefully and backed up before deletion. Always ensure that data is not lost before proceeding with any cleanup operations.