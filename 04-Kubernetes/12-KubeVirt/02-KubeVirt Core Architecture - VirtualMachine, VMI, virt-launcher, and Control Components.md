# 02-KubeVirt Core Architecture: VirtualMachine, VMI, virt-launcher, and Control Components

Recommended Path:

    04-Kubernetes/12-KubeVirt/02-KubeVirt Core Architecture: VirtualMachine, VMI, virt-launcher, and Control Components.md

Tags:

    #Kubernetes
    #KubeVirt
    #VirtualMachine
    #VMI
    #virt-launcher
    #virt-api
    #virt-controller
    #virt-handler
    #virt-operator
    #Cloud-Native Virtualization
    #Platform Engineering

---

## I. Document Overview

This document is designed to help you understand the core architecture of KubeVirt.

Key Points to Master:

    1. The overall architecture of KubeVirt
    2. The differences between VirtualMachine and VirtualMachineInstance
    3. What a virt-launcher Pod is
    4. The functions of virt-api, virt-controller, virt-handler, and virt-operator
    5. The basic process of creating and running a KubeVirt virtual machine
    6. How to monitor KubeVirt resources using kubectl
    7. How to troubleshoot virtual machine issues through events and Pod relationships

This document focuses on:

    Architectural understanding + Practical observation.
    It is not an installation guide.
    The actual installation of KubeVirt will be covered in subsequent practical guides.

Note:

    If your current cluster does not have KubeVirt installed, you can read the conceptual section first. After installing it, return to this document and execute the experimental commands to deepen your understanding.

---

## II. KubeVirt Architecture Overview

KubeVirt is a virtualization extension for Kubernetes.

It integrates virtual machines into the Kubernetes resource model through CRDs and Controllers.

Overall Relationship:

    User / Operations Personnel
          |
          v
    kubectl / virtctl
          |
          v
    Kubernetes API Server
          |
          v
    KubeVirt CRD
          |
          v
    KubeVirt Control Components
          |
          v
    virt-launcher Pod
          |
          v
    KVM / QEMU
          |
          v
    Virtual Machine

From a Kubernetes perspective:

    KubeVirt consists of a set of CRDs, Controllers, and Pods.

From a virtualization perspective:

    KubeVirt is a virtual machine management solution based on KVM/QEMU.

From a platform perspective:

    KubeVirt enables Kubernetes to manage both containers and virtual machines uniformly.

---

## III. Key KubeVirt Resource Objects

Common KubeVirt resource objects include:

    VirtualMachine
    VirtualMachineInstance
    VirtualMachineInstanceReplicaSet
    VirtualMachineInstanceMigration
    DataVolume
    VirtualMachineSnapshot
    VirtualMachineRestore

For beginners, the most important ones are:

    VirtualMachine
    VirtualMachineInstance
    DataVolume

In this document, we will focus on:

    VirtualMachine
    VirtualMachineInstance
    virt-launcher

---

## IV. What is a VirtualMachine?

A VirtualMachine, often abbreviated as VM, represents the desired state and configuration of a virtual machine.

It specifies:

    1. The name of the virtual machine
    2. Whether it is running or not
    3. CPU configuration
    4. Memory configuration
    5. Disk configuration
    6. Network configuration
    7. Boot strategy
    8. cloud-init settings
    9. Scheduling rules
    10. Runtime template

In other words:

    VirtualMachine = A virtual machine definition object

Similar to Deployment in Kubernetes:

    Deployment defines the desired state of an application.
    VirtualMachine defines the desired state of a virtual machine.

A VirtualMachine is not the actual virtual machine process itself. It is more like:

    "I want a virtual machine with these specifications in my cluster."

---

## V. What is a VirtualMachineInstance?

A VirtualMachineInstance, often abbreviated as VMI, represents an actual running virtual machine instance.

It can be understood as:

    VMI = A running virtual machine instance

Analogous to native Kubernetes objects:

    Deployment -> Pod
    VirtualMachine -> VirtualMachineInstance

Relationship:

    VirtualMachine defines the desired state.
    VirtualMachineInstance represents the actual running state.

When a VM is started, it becomes a VMI. When a VM stops, the VMI typically disappears.

Therefore:

    A VM can exist, but a VMI may not always be present.
    The presence of a VMI indicates that the virtual machine is running.

---

## VI. Differences Between VM and VMI

| Object | Function | Persistence | Analogy |
|---|---|---|---|
| VirtualMachine | Definition and desired state of a virtual machine | Usually persistent | Deployment |
| Virtual4. Managing the virt-controller
5. Managing the virt-handler
6. Handling KubeVirt component upgrades
7. Ensuring that KubeVirt components are in the desired state

This can be understood as:

virt-operator = The operator responsible for managing KubeVirt itself.

If the virt-operator experiences an exception, it may affect the upgrade, modification, and self-healing of KubeVirt components.

To check:

    kubectl get pods -n kubevirt | grep virt-operator

    kubectl logs -n kubevirt deploy/virt-operator --tail=100

---

## Section Eleven: virt-api

virt-api provides the API capabilities for KubeVirt.

It is responsible for handling requests related to virtual machine operations.

For example:

    virtctl console
    virtctl start
    virtctl stop
    virtctl restart
    virtctl vnc

These commands usually interact with KubeVirt through virt-api.

This can be understood as:

virt-api = The API entry component for KubeVirt.

If virt-api encounters an issue, the following may occur:

    1. virtctl may fail to connect to the console.
    2. Commands like start/stop may fail.
    3. API calls for virtual machine operations may fail.

To check:

    kubectl get pods -n kubevirt | grep virt-api

    kubectl logs -n kubevirt deploy/virt-api --tail=100

---

## Section Twelve: virt-controller

The virt-controller is one of the core controllers in KubeVirt.

It is responsible for controlling VirtualMachine and VirtualMachineInstance according to their desired states.

Main responsibilities include:

    1. Monitoring VirtualMachine
    2. Monitoring VirtualMachineInstance
    3. Creating or deleting VMI
    4. Coordinating with the virt-launcher Pod
    5. Handling VM startup, shutdown, and status changes
    6. Updating the status of VirtualMachine/VMI
    7. Managing migration, status synchronization, and other control logic

This can be understood as:

virt-controller = The controller that manages the lifecycle of virtual machines.

In analogy to Kubernetes:

    The Deployment Controller creates ReplicaSet/Pod based on a Deployment.
    The virt-controller creates VMI/virt-launcher-related resources based on VirtualMachine.

To check:

    kubectl get pods -n kubevirt | grep virt-controller

    kubectl logs -n kubevirt deploy/virt-controller --tail=100

---

## Section Thirteen: virt-handler

virt-handler is a node-side component.

It usually runs as a DaemonSet on each node in the cluster.

Main responsibilities include:

    1. Running on every node that can host virtual machines
    2. Managing the lifecycle of virtual machines at the node level
    3. Interacting with the local virtualization capabilities
    4. Managing VM startup, shutdown, and status synchronization
    5. Monitoring VMI on the current node
    6. Coordinating between virt-launcher and the host machine's virtualization capabilities

This can be understood as:

virt-handler = The virtual machine manager for each node in the cluster.

If the virt-handler on a certain node fails, the virtual machines running on that node may not function correctly.

To check:

    kubectl get pods -n kubevirt -o wide | grep virt-handler

    To check the virt-handler on a specific node:

    kubectl get pods -n kubevirt -o wide | grep <node-name>

    To view logs:

    kubectl logs -n kubevirt <virt-handler-pod-name> --tail=100

---

## Section Fourteen: virt-launcher

virt-launcher is the Pod that corresponds to each running virtual machine.

It is not necessarily in the kubevirt namespace; it usually shares the same namespace as the virtual machine.

For example, if a virtual machine is in the default namespace:

    The virt-launcher Pod will also typically be in the default namespace.

To check:

    kubectl get pods -n default | grep virt-launcher

The main responsibilities of virt-launcher include:

    1. Hosting the virtual machine process
    2. Mounting the virtual machine's disks
    3. Connecting the Pod to the network
    4. Starting QEMU/KVM
    5. Enabling Kubernetes to manage the virtual machine through the Pod

When troubleshooting a virtual machine, focus on the following:

    1. The virtual machine itself
    2. The VMI
    3. The virt-launcher Pod
    4. Pod Events
    5. virt-launcher logs
    6. The node where the Pod is located
    7. The /dev/kvm directory on the node
    8. PVCs and DataVolumes

---

## Section Fifteen: The Basic Process of KubeV```markdown
🔤 Common resources include:
virtualmachines.kubevirt.io
virtualmachineinstances.kubevirt.io
virtualmachineinstancereplicasets.kubevirt.io
virtualmachineinstancemigrations.kubevirt.io
kubevirts.kubevirt.io

To view API resources:
kubectl api-resources | grep kubevirt
Focus on:
virtualmachines
virtualmachineinstances
Shortcuts:
kubectl api-resources | grep -E "virtualmachines|virtualmachineinstances"
Possible shortcuts include:
vm
vmi

Verification:
kubectl get vm -A
kubectl get vmi -A
Expected result:
vm and vmi resources should be recognized correctly.

Note:
If kubectl get vm reports no resources, it means the KubeVirt CRD is not installed successfully or the current cluster does not have KubeVirt.
---

## Experiment 2: Checking KubeVirt Control Components
Experiment objective:
To observe whether KubeVirt control plane components are running normally.

Execution:
kubectl get pods -n kubevirt -o wide

Common components include:
virt-api
virt-controller
virt-handler
virt-operator

View Deployments:
kubectl get deploy -n kubevirt

View DaemonSets:
kubectl get ds -n kubevirt

Key understanding:
virt-api and virt-controller are usually Deployments.
virt-operator is usually a Deployment.
virt-handler is usually a DaemonSet.

View component logs:
kubectl logs -n kubevirt deploy/virt-api --tail=50
kubectl logs -n kubevirt deploy/virt-controller --tail=50
kubectl logs -n kubevirt deploy/virt-operator --tail=50

View virt-handler:
kubectl get pods -n kubevirt -o wide | grep virt-handler
Select a virt-handler Pod:
kubectl logs -n kubevirt <virt-handler-pod-name> --tail=50

Expected result:
Key KubeVirt components should be in the Running state.
The virt-handler Pod should be distributed on nodes where running virtual machines are located.
---

## Experiment 3: Observing the Relationship Between VM, VMI, and virt-launcher
Experiment objective:
To observe which Kubernetes objects correspond to a running virtual machine.

Prerequisite:
A test VM has already been created and started.

Assume the VM name is:
testvm
Namespace:
default

View the VM:
kubectl get vm -n default

View the VMI:
kubectl get vmi -n default

View Pods:
kubectl get pods -n default -o wide | grep virt-launcher

View detailed information:
kubectl describe vm testvm -n default
kubectl describe vmi testvm -n default

Find the virt-launcher Pod:
kubectl get pods -n default -o wide | grep testvm

View the virt-launcher Pod:
kubectl describe pod <virt-launcher-pod-name> -n default

Expected relationship:
The VM exists.
The VMI exists.
The virt-launcher Pod exists.
The virt-launcher Pod is in the Running state.
The VMI and virt-launcher Pod are on the same node.

Record:
VM name
VMI name
virt-launcher Pod name
Node location
Pod IP
PVC name
---

## Experiment 4: Stopping a VM and Observing Changes to VMI and virt-launcher
Experiment objective:
To understand the changes that occur to VM, VMI, and virt-launcher when a VM is stopped.

Stop the VM:
virtctl stop testvm -n default

View the VM:
kubectl get vm testvm -n default

View the VMI:
kubectl get vmi -n default

View the virt-launcher Pod:
kubectl get pods -n default | grep virt-launcher

Expected result:
The VM object still exists.
The VMI object disappears.
The virt-launcher Pod disappears or enters the Terminating state before disappearing.
The PVC still exists.

View the PVC:
kubectl get pvc -n default

Understanding:
Stopping a VM does not mean deleting it.
Stopping a VM does not mean deleting its associated disks.
The VM definition and PVC data remain intact.
---

## Experiment 5: Starting a VM and Observing Object Rebuilding
Experiment objective:
To understand that VMI and virt-launcher are recreated when the VM is started.

Start the VM:
virtctl start testvm -n default

Observation:
kubectl get vm testvm -n default
kubectl get vmi testvm -n default
kubectl get pods -n default -o wide | grep virt-launcher

View events:
kubectl describe vm testvm -n default
kubectl describe vmi testvm -n default

Expected result:
The VMI reappears.
The virt-launcher Pod reappears.
The VM enters the Running state.

If starting fails, continue to check:
kubectl describe vmi testvm -n default
kubectl get events -n default --sort-by=.lastTimestamp
kubectl describe pod <virt-launcher-pod-name> -n default
---
## Experiment 6: InvestigatingIf the VMI has not been created:
Check the VM Events and virt-controller logs.

If the virt-launcher Pod is Pending:
Check the Pod Events, scheduling, resources, and PVCs.

If the Pod has been created but the VM has not started:
Check the virt-launcher logs, virt-handler logs, and the node's KVM capabilities.---

## Summary of This Article

The core architecture of KubeVirt can be summarized as follows:

- CRDs define virtual machine resources.
- Controllers manage the status of virtual machines.
- The virt-launcher Pod carries the virtual machine processes.
- KVM/QEMU provide the underlying virtualization capabilities.
- PVCs provide virtual machine disks.
- CNI provides virtual machine networking.

Key object relationships include:

- **VirtualMachine**: Describes the expected state of a virtual machine.
- **VirtualMachineInstance**: Represents a running virtual machine instance.
- **virt-launcher Pod**: Actually carries the virtual machine processes.
- **virt-handler**: Manages virtual machines at the node level.
- **virt-controller**: Controls the status of VMs and VMI.
- **virt-api**: Provides KubeVirt API functionality.
- **virt-operator**: Manages the lifecycle of KubeVirt components.

The most critical architecture links are:

- VM -> VMI -> virt-launcher Pod -> QEMU/KVM -> Guest OS

For troubleshooting, key commands include:

- `kubectl get vm`
- `kubectl describe vm`
- `kubectl get vmi`
- `kubectl describe pod | grep virt-launcher`
- `kubectl describe pod virt-launcher`
- `kubectl logs virt-launcher`
- `kubectl get pvc`
- `kubectl get events`

It is recommended to continue learning with the following topic:

- **03 - Preparations Before KubeVirt Installation: Checking KVM, Node Virtualization Capabilities, Storage, and Networking.md