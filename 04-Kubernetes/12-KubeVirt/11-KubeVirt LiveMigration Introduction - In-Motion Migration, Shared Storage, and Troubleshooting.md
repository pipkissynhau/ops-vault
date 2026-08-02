# 11-KubeVirt LiveMigration Introduction: In-Motion Migration, Shared Storage, and Troubleshooting

Recommended Path:

    04-Kubernetes/12-KubeVirt/11-KubeVirt LiveMigration Introduction: In-Motion Migration, Shared Storage, and Troubleshooting.md

Tags:

    #Kubernetes
    #KubeVirt
    #LiveMigration
    #Virtual Machine Migration
    #VMI
    #virt-launcher
    #Shared Storage
    #RWX
    #Longhorn
    #DataVolume
    #Platform Engineering
    #Cloud-Native Virtualization

---

## I. Document Description

This document outlines the basic concepts, prerequisites, experimental design, execution methods, and troubleshooting approaches for KubeVirt LiveMigration.

LiveMigration is an advanced virtual machine operations capability within KubeVirt.

Its objectives include:

    Migrating a virtual machine from one Kubernetes Node to another while it is still running,
    with minimal disruption to business operations.

This is similar to:

    vSphere vMotion in traditional virtualization,

but it's important to note that:

    KubeVirt LiveMigration is not simply equivalent to vMotion. It relies on the coordinated operation of multiple components such as Kubernetes, KubeVirt, node KVM, storage, networking, virt-handler, and virt-launcher. Among these, storage and networking are the most likely areas to encounter issues.

The goals of this document are:

    1. To understand the differences between cold migration, live migration, and volume migration.
    2. To comprehend the basic workflow of KubeVirt LiveMigration.
    3. To grasp why shared storage is typically required for LiveMigration.
    4. To understand the relationship between RWX storage and LiveMigration.
    5. To create a virtual machine suitable for migration.
    6. To execute a migration using virtctl migrate.
    7. To observe the VirtualMachineInstanceMigration resources.
    8. To verify the changes in the nodes where the virtual machine resides before and after migration.
    9. To identify common causes of LiveMigration failures and know how to troubleshoot them.
    10. To develop a clear approach for answering related interview questions.

## II. Differentiate Between Migration Types

When discussing "migration" in KubeVirt, it's important to recognize that there are more than one type:

    At least three types need to be distinguished:

        1. Cold migration
        2. Live Migration (hot migration)
        3. Volume migration (storage migration)

---

### 2.1 Cold Migration

Cold migration refers to:

    Stopping the virtual machine first,
    and then restarting it on another node.

Characteristics:

    1. The virtual machine will be interrupted.
    2. It is relatively simple to implement.
    3. It has lower requirements for storage.
    4. It is suitable for maintenance tasks during scheduled downtime.
    5. It does not represent true in-motion migration.

Common scenarios:

    Node maintenance
    Resource reallocation
    Manual migration testing
    Adjusting the nodes of non-core virtual machines

Basic steps:

    virtctl stop <vm>
    Adjust scheduling rules
    virtctl start <vm>

---

### 2.2 Live Migration (Hot Migration)

Live Migration means:

    Migrating a virtual machine while it is still running from the source node to the target node.

Objective:

    To minimize business interruptions as much as possible.

Characteristics:

    1. The virtual machine remains running during the migration process.
    2. Both the source and target nodes participate in the migration.
    3. The memory state of the virtual machine needs to be transferred.
    4. The storage must be accessible from both nodes.
    5. It requires more stringent networking and storage capabilities.
    6. It is closest to vMotion in traditional virtualization.

This document focuses on:

    Live Migration.

---

### 2.3 Volume Migration (Storage Migration)

Volume migration refers to:

    Moving the disk volume used by a virtual machine,
    for example, from one PVC/StorageClass to another.

It is more related to storage-related operations.

Common scenarios:

    1. Migrating from an old StorageClass to a new one.
    2. Switching from NFS to Longhorn/Ceph.
    3. Moving data from low-performance storage to high-performance storage.
    4. Adjusting the backend of a virtual machine's disk.

Note:

    Volume migration is more advanced than what is covered in this document. It will be discussed in a separate article later on.

---

## III. Basic Principles of KubeVirt LiveMigration

The core workflow of KubeVirt LiveMigration can be understood as follows:

    Source node virt-launcher
          |
          | Transfers the virtual machine's running state9. The target node does not have /dev/kvm

---

## VII. Differences Between LiveMigration and vSphere vMotion

| Comparison Item | vSphere vMotion | KubeVirt LiveMigration |
|---|---|---|
| Management Interface | vCenter | Kubernetes API / virtctl |
| Underlying Virtualization | ESXi | KVM / QEMU |
| Scheduling Mechanism | vSphere Cluster / DRS | Kubernetes Scheduler |
| Storage System | Datastore / vSAN / SAN | PVC / CSI / StorageClass |
| Networking System | vSwitch / dvSwitch | CNI / Service / Multus |
| Migration Object | VM | VMI / virt-launcher |
| Operation and Maintenance Method | vCenter UI / PowerCLI | kubectl / virtctl |
| Dependent Components | vCenter / ESXi | virt-api / virt-controller / virt-handler |
| Objects to Be Checked | VM, Host, Datastore | VM, VMI, Pod, PVC, Node |

In short:

    vMotion is a mature migration capability in traditional virtualization platforms.
    KubeVirt LiveMigration is a machine migration capability within the Kubernetes ecosystem; it relies more heavily on Kubernetes's scheduling, storage, and networking infrastructure.

---

## VIII. Experimental Objectives

The objectives of this experiment are:

    1. Create an experimental namespace.
    2. Prepare RWX PVCs and DataVolumes.
    3. Create a VM with an evictionStrategy configured.
    4. Start the VM.
    5. Record the node on which the VM was originally located.
    6. Execute the migration using virtctl migrate.
    7. Observe the changes related to VirtualMachineInstanceMigration.
    8. Monitor the changes in the virt-launcher Pod.
    9. Verify the new node where the migrated VM is located.
    10. Test whether node drain triggers the migration process.
    11. Summarize the troubleshooting steps for any migration failures.

---

## IX. Experimental Environment Planning

Example nodes:

    k8s-master-01     10.0.0.20
    k8s-master-02     10.0.0.21
    k8s-master-03     10.0.0.22
    k8s-worker-01     10.0.0.23
    k8s-worker-02     10.0.0.24
    k8s-worker-03     10.0.0.25

Recommended VM nodes for the experiment:

    k8s-worker-01
    k8s-worker-02
    k8s-worker-03

Experimental namespace:

    kubevirt-migration-demo

Experimental VM:

    vm-live-migrate-demo

Recommended StorageClass:

    longhorn-rwx

If longhorn-rwx is not available, you can use another StorageClass that supports RWX, such as nfs-client, cephfs, or rook-cephfs.

---

## X. Pre-experiment Checks

### 10.1 Checking KubeVirt

Execute the following commands:

    kubectl -n kubevirt get kv kubevirt

    kubectl -n kubevirt get pods -o wide

Requirements:

    KubeVirt must be available.
    The virt-api, virt-controller, virt-handler, and virt-operator services must all be running.

---

### 10.2 Checking virtctl

Execute:

    virtctl version

---

### 10.3 Checking CDI

Run these commands:

    kubectl -n cdi get pods -o wide

    kubectl get dv -A

If the DataVolume resources are not available, you need to install CDI first.

---

### 10.4 Checking KVM on Worker Nodes

On each Worker node, execute the following commands:

    egrep -c '(vmx|svm)' /proc/cpuinfo

    ls -l /dev/kvm

    sudo kvm-ok

Requirements:

    The vmx or svm flag must be set to 1.
    The /dev/kvm directory must exist.
    The kvm-ok command must execute successfully.

---

### 10.5 Checking the Number of Worker Nodes

Execute:

    kubectl get nodes -o wide

At least two available Worker nodes are required for this experiment.

If there is only one runnable node, cross-node migration cannot be verified.

---

### 10.6 Checking Node Resources

Run:

    kubectl top nodes

If the metrics-server is not installed, you can check the following information:

    kubectl describe node <node-name> | grep -A8 "Allocated resources"

Requirements:

    Both the source and target nodes must have sufficient CPU and Memory resources.

---

### 10.7 Checking the StorageClass

Execute:

    kEOF

If your RWX StorageClass is not longhorn-rwx, replace it with:

storageClassName: <your RWXStorageClass>

Apply the configuration:

kubectl apply -f pvc-rwx-precheck.yaml

Check the result:

kubectl -n kubevirt-migration-demo get pvc rwx-precheck-pvc

The expected output should be:

STATUS: Bound
ACCESS MODES: RWX

To view the PV, use:

kubectl get pv | grep rwx-precheck-pvc

If the PVC is still Pending, use:

kubectl -n kubevirt-migration-demo describe pvc rwx-precheck-pvc

Continue to investigate issues related to the StorageClass, provisioner, node, etc.

To clean up the pre-check PVC, use:

kubectl -n kubevirt-migration-demo delete pvc rwx-precheck-pvc

---

## Experiment 14: Creating a System Disk DataVolume

Create the DataVolume configuration:

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

If you are using a different RWX StorageClass, replace it with:

storageClassName: <your RWXStorageClass>

Apply the configuration:

kubectl apply -f dv-live-migrate-rootdisk.yaml

To check the DataVolume, use:

kubectl -n kubevirt-migration-demo get dv

To view the associated PVC, use:

kubectl -n kubevirt-migration-demo get pvc

To check the importer Pod, use:

kubectl -n kubevirt-migration-demo get pods

To view the import logs, use:

kubectl -n kubevirt-migration-demo logs <importer-pod-name> --tail=100

Once the import is successful, check the status of the DataVolume:

kubectl -n kubevirt-migration-demo get dv live-migrate-rootdisk

The expected output should be:

PHASE: Succeeded

For the PVC, the expected status should be:

STATUS: Bound
ACCESS MODES: RWX

If accessing the public URL is slow, you can use an internal HTTP address instead:

http://10.0.0.10/images/cirros-0.6.2-x86_64-disk.img

---

## Experiment 15: Creating a Migratable VM

Create the VM configuration:

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

Explanation:

- evictionStrategy: LiveMigrate: This ensures that when the node tries to evict the VMI, it will attempt a LiveMigration first.
- rootdisk uses a DataVolume: This DataVolume is associated with an RWX PVC, which facilitates migration.
- masquerade: Using the default Pod network is suitable for beginner migration experiments.

Apply the configuration:

kubectl apply -f vm-live-migrate-demo.yaml

To check the VM, use:

kubectl -n kubevirt-migration-demo get vm

To view the VMI details, use:

kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo -o wide

Record the following information:

- Current status of the VM
- Current nodeIf it indicates that migration is not possible, you usually need to check the following:

    1. Whether the disk is set to RWX mode.
    2. If a disk type that does not support migration is being used.
    3. Whether hostDisk or local storage is being utilized.
    4. If special device passthrough is in use.
    5. Whether network binding supports the migration.
    6. Whether KubeVirt's migration configuration allows it.

---

## Experiment Eighteen: Performing LiveMigration

To execute the migration:

    virtctl migrate vm-live-migrate-demo -n kubevirt-migration-demo

Explanation:

    The `virtctl migrate` command creates a migration request for the running VMI.

To view the migration objects:

    kubectl -n kubevirt-migration-demo get virtualmachineinstancemigrations

If abbreviations are supported, you can also try:

    kubectl -n kubevirt-migration-demo get vmim

For more details:

    kubectl -n kubevirt-migration-demo describe virtualmachineinstancemigration <migration-name>

At the same time, observe the VMI:

    kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o wide

To monitor the virt-launcher:

    kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher

It is recommended to open two windows for observation:

Window one:

    watch -n 1 "kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o wide"

Window two:

    watch -n 1 "kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher"

During the migration process, you may see the following:

    1. The VirtualMachineInstanceMigration object appears.
    2. A new virt-launcher Pod is created on the target node.
    3. There might be two related virt-launcher Pods for a short time.
    4. The VMI's NODE changes from the source node to the target node.
    5. The migration object eventually shows "Succeeded".

---

## Experiment Nineteen: Verifying Migration Results

To check the VMI:

    kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o wide

To view the virt-launcher:

    kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher

To view the migration objects:

    kubectl -n kubevirt-migration-demo get virtualmachineinstancemigrations

To check the VMI's migrationState:

    kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o yaml | grep -A80 migrationState

To determine if the migration was successful:

    1. The VMI is still running.
    2. The VMI's NODE has changed to the new node.
    3. The virt-launcher Pod on the new node is running.
    4. The migration-related Pods on the old node have been cleaned up.
    5. The VirtualMachineInstanceMigration shows "Succeeded".
    6. The console or SSH can still be accessed.

To enter the console:

    virtctl console vm-live-migrate-demo -n kubevirt-migration-demo

To log in:

    Username: cirros
    Password: kubevirt

To check information:

    hostname
    ip addr
    uptime

To exit:

    Ctrl + ]

---

## Experiment Twenty: Verifying Access During Migration Using a Service

To create an SSH Service:

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

To apply the configuration:

    kubectl apply -f svc-live-migrate-ssh.yaml

To check:

    kubectl -n kubevirt-migration-demo get svc

    kubectl -n kubevirt-migration-demo get endpoints vm-live-migrate-ssh

To access from outside:

    ssh cirros@10.0.0.23 -p 30025

Password:

    kubevirt

During the migration, you can continue to monitor with ping or SSH in another terminal.

Note:

    CirrOS is a lightweight operating system and is only used for understanding the migration process in this experiment.
    In production, real business connections and monitoring must be used to verify the impact of migration.

---

## Experiment Twenty-One: Triggering Migration Using Drain

Explanation:

The `patch` field must match the actual CRD of the KubeVirt version. If the fields are incompatible, the fields actually supported by the current version will be used.```markdown
kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo

Common reasons:

    1. The VMI is not eligible for migration.
    2. Schedule failure on the target node.
    3. The PVC does not meet the migration requirements.
    4. Migration connection failure.
    5. Failure to start the target virt-launcher.
    6. Abnormalities with the virt-handler on the source or target node.
    7. Migration timeout.
    8. The VM was stopped or deleted during migration.

---

## Issue 32: Common Problem 8: VM Not Migrated During Drain
Check the VM YAML:

    kubectl -n kubevirt-migration-demo get vm vm-live-migrate-demo -o yaml | grep -A5 evictionStrategy

It should contain:

    evictionStrategy: LiveMigrate

Check if the VMI is eligible for migration:

    kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo

Check the node drain output.

Common reasons:

    1. The VM does not have the evictionStrategy set to LiveMigrate.
    2. The VMI is not eligible for migration.
    3. No available target nodes.
    4. Insufficient resources on the target node.
    5. The PVC does not meet the migration requirements.
    6. KubeVirt migration configuration limitations.
    7. PodDisruptionBudget or eviction restrictions.

Solutions:

    1. Set the evictionStrategy for the VM to LiveMigrate.
    2. Use an RWX PVC.
    3. Add more available nodes.
    4. Reduce the VM's resource requirements.
    5. Manually perform a virtctl migrate operation first to verify the process.
---

## Issue 33: List of Standard LiveMigration Commands

### 33.1 View VMs/VMIs

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

Alternative shortened form:

    kubectl -n kubevirt-migration-demo get vmim

---

### 33.4 View Pods

    kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher

    kubectl -n kubevirt-migration-demo describe pod <virt-launcher-pod-name>

    kubectl -n kubevirt-migration-demo logs <virt-launcher-pod-name> --tail=100

---

### 33.5 View PVCs

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

For local node details:

    ls -l /dev/kvm

    egrep -c '(vmx|svm)' /proc/cpuinfo

    systemctl status kubelet --no-pager

    systemctl status containerd --no-pager

    journalctl -u kubelet -n 200 --no-pager

---

## Issue 34: Precautions for Production Environments

Before deploying KubeVirt LiveMigration in a production environment, it is essential to confirm the following:

    1. Whether the VMs are suitable for live migration.
    2. If the storage supports migration.
    3. If the network can handle the migration traffic.
    4. If there are sufficient node resources.
    5. How migration will affect business connectivity.
    6. If there is a backup plan in case of migration failure.
    7. Whether there are scheduled maintenance periods.
    8. If monitoring and alert mechanisms are in place.
    9. If migration tests have been conducted successfully.
    10. If clear node maintenance proceduresHowever, due to different implementation frameworks, KubeVirt's migration process relies more heavily on Kubernetes' storage, networking, and scheduling capabilities. Therefore, when addressing migration issues with KubeVirt, it is not sufficient to approach them using traditional virtualization techniques; one must also examine logs related to Pods, PVCs, Nodes, Events, as well as KubeVirt components.

---

## Thirty-Nine: Clearing Up Experimental Resources

Stop the Virtual Machine:

    virtctl stop vm-live-migrate-demo -n kubevirt-migration-demo

Delete the Service:

    kubectl delete -f svc-live-migrate-ssh.yaml --ignore-not-found

Delete the Virtual Machine:

    kubectl delete -f vm-live-migrate-demo.yaml --ignore-not-found

Delete the DataVolume:

    kubectl delete -f dv-live-migrate-rootdisk.yaml --ignore-not-found

Check the PVC:

    kubectl -n kubevirt-migration-demo get pvc

If it is confirmed that there is no need to retain it, delete the residual PVC:

    kubectl -n kubevirt-migration-demo delete pvc live-migrate-rootdisk --ignore-not-found

Delete the pre-check resources:

    kubectl delete -f pvc-rwx-precheck.yaml --ignore-not-found

Delete the namespace:

    kubectl delete namespace kubevirt-migration-demo

If an experimental StorageClass was created and is no longer needed, delete it:

    kubectl delete -f sc-longhorn-rwx.yaml --ignore-not-found

Note:

    Before deleting PVCs or DataVolumes, make sure that there is no business data remaining on them. In a production environment, resources should not be deleted in the same manner as during experiments.

---

## Forty: Summary of This Article

KubeVirt LiveMigration is an important advanced operational capability within KubeVirt.

Key understandings:

    1. Cold migration involves shutting down the system before proceeding.
    2. LiveMigration allows for migration while the system is running.
    3. Volume Migration specifically refers to the process of moving disks between systems.
    4. LiveMigration relies on components such as KVM, virt-handler, virt-launcher, shared storage, and networking.
    5. During migration, a VirtualMachineInstanceMigration object is created.
    6. After successful migration, the node where the VMI resides will change.
    7. The evictionStrategy: LiveMigrate can be used in scenarios involving node drain.
    8. RWX PVCs are more suitable for introductory LiveMigration experiments.
    9. Special scenarios such as RWO, local disks, device passthrough, or multiple network cards may limit migration possibilities.
    10. Migration drills must be conducted before deploying LiveMigration in a production environment.

Key processes:

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
    Switching of the VMI's running node

Key commands:

    virtctl migrate vm-live-migrate-demo -n kubevirt-migration-demo

    kubectl -n kubevirt-migration-demo get virtualmachineinstancemigrations

    kubectl -n kubevirt-migration-demo describe virtualmachineinstancemigration <migration-name>

    kubectl -n kubevirt-migration-demo get vmi vm-live-migrate-demo -o wide

    kubectl -n kubevirt-migration-demo get pods -o wide | grep virt-launcher

    kubectl -n kubevirt-migration-demo describe vmi vm-live-migrate-demo

Practical tips:

    1. Before starting a migration, check whether the VMI is eligible for LiveMigration.
    2. In case of a migration failure, first examine the Migration object and related Events.
    3. Insufficient storage capacity is one of the most common issues.
    4. Insufficient resources on the target node are also a frequent problem.
    5. If a certain node experiences abnormal migration behavior, check virt-handler and /dev/kvm.
    6. Before triggering a drain-based migration, ensure that the evictionStrategy is set correctly.
    7. In production environments, do not rely on LiveMigration as a default solution; always verify its suitability based on actual storage and networking conditions.

Future areas for expansion:

    12-KubeVirt Volume Migration Introduction: Virtual Machine Disk Migration, PVC and StorageClass Management.md