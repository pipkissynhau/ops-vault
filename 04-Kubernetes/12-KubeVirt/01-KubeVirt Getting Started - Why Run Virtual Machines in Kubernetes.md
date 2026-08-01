# 01-KubeVirt Getting Started: Why Run Virtual Machines in Kubernetes

Recommended Path:

    04-Kubernetes/12-KubeVirt/01-KubeVirt Getting Started: Why Run Virtual Machines in Kubernetes.md

Tags:

    #Kubernetes
    #KubeVirt
    #Virtual
    #KVM
    #QEMU
    #PlatformEngineering
    #SRE
    #CloudlandVirtualization
    #PrivateClouds

---

## I. Document Overview

This document is for learning the basic concepts of KubeVirt, focusing on understanding:

    1. What is KubeVirt
    2. Why run virtual machines in Kubernetes
    3. What problems KubeVirt solves
    4. Differences between KubeVirt and regular Pods
    5. Relationship between KubeVirt and vSphere / OpenStack
    6. Suitable scenarios for KubeVirt
    7. Unsuitable scenarios for KubeVirt
    8. How to answer KubeVirt-related questions in interviews

This document does not involve installation commands and is mainly for building an overall understanding.

---

## II. What is KubeVirt

KubeVirt is a virtualization extension for Kubernetes.

Its core goal is:

    To enable Kubernetes to manage not only containers but also virtual machines.

Simple understanding:

    Kubernetes originally mainly manages Pods.
    KubeVirt enables Kubernetes to also manage VMs.

More intuitively:

    Originally:
        vSphere manages virtual machines
        Kubernetes manages containers

    After introducing KubeVirt:
        Kubernetes can manage both containers and virtual machines

KubeVirt is not reinventing a virtualization platform but rather integrating virtual machine capabilities into Kubernetes' resource model.

---

## III. Why Run Virtual Machines in Kubernetes

Many enterprises are not fully cloud-native from the start.

Actual environments are typically:

    1. Old business runs on virtual machines
    2. New business runs on containers
    3. Some business cannot be containerized
    4. Some systems depend on full operating systems
    5. Platform teams want to unify resource management
    6. Operations teams want to reduce platform fragmentation

Traditional environments may be:

    vSphere / OpenStack / Private Cloud
        Manages virtual machines

    Kubernetes
        Manages containers

This creates several issues:

    1. Platform fragmentation
    2. Permission system fragmentation
    3. Monitoring system fragmentation
    4. Network model fragmentation
    5. Storage model fragmentation
    6. Deployment process fragmentation
    7. Operations entry fragmentation
    8. Automation capability fragmentation

The problem KubeVirt aims to solve is:

    To host both container and virtual machine workloads on a unified Kubernetes platform.

---

## IV. Core Problems Solved by KubeVirt

KubeVirt mainly solves the following issues:

    1. Virtual machine workloads are integrated into Kubernetes management
    2. Traditional business is gradually migrated to cloud-native platforms
    3. Unified scheduling of containers and virtual machines
    4. Unified use of Kubernetes network, storage, permissions, and resource objects by containers and virtual machines
    5. Reduce fragmentation between traditional virtualization platforms and container platforms
    6. Support legacy business that cannot be containerized to continue running
    7. Provide a unified foundation for private cloud platforms with containers + virtual machines

In one sentence:

    KubeVirt is a bridge between traditional virtualization and Kubernetes.

---

## V. Basic Working Mechanism of KubeVirt

KubeVirt extends virtual machine-related resources through Kubernetes CRDs.

Common resources include:

    VirtualMachine
    VirtualMachineInstance
    VirtualMachineInstanceReplicaSet
    DataVolume
    VirtualMachineInstanceMigration

The most fundamental ones are:

    VirtualMachine
    VirtualMachineInstance

You can understand this as:

    VirtualMachine:
        The desired configuration of a virtual machine.
        Similar to "what this virtual machine should look like".

    VirtualMachineInstance:
        The running virtual machine instance.
        Similar to "the current running state of this virtual machine".

Analogous to Kubernetes native resources:

    Deployment:
        Describes the desired application state.

    Pod:
        The actual running container instance.

Analogous to KubeVirt:

    VirtualMachine:
        Describes the desired virtual machine state.

    VirtualMachineInstance:
        The actual running virtual machine instance.

---

## VI. Where Do Virtual Machines Ultimately Run

Virtual machines in KubeVirt are not running in a vacuum.

They ultimately run as a special Pod on Kubernetes nodes.

This Pod is usually called:

    virt-launcher

The general relationship is as follows:

    VirtualMachine
          |
          v
    VirtualMachineInstance
          |
          v
    virt-launcher Pod
          |
          v
    KVM / QEMU
          |
          v
    Virtual machine process

In other words:

    Kubernetes sees a Pod.
    KubeVirt runs virtual machines inside the Pod.
    Virtual machines rely on KVM / QEMU capabilities on the node.

This is why KubeVirt has requirements for nodes:

    Nodes need to support hardware virtualization.
    Nodes typically need /dev/kvm.
    Node kernels and systems must support KVM.
    Node resources must be sufficient to host virtual machines.

---

## VII. Differences Between KubeVirt and Regular Pods

| Comparison Item | Regular Pod | KubeVirt Virtual Machine |
|---|---|---|
| Runtime Object | Container Process | Virtual Machine |
| Isolation Method | Container Isolation | Virtualization Isolation |
| Boot Speed | Usually Fast | Relatively Slow |
| Operating System | Shared Host Kernel | Independent Guest OS |
| Image Format | Container Image | Virtual Machine Disk Image |
| Storage | volume / PVC | PVC / DataVolume / Virtual Disk |
| Network | Pod Network | Pod Network or Multi-Interface Network |
| Access Method | kubectl exec | virtctl console / ssh |
| Suitable Scenarios | Cloud-Native Applications | Traditional Systems, OS-Dependent Applications |

Core Differences:

    Pod runs containers.
    KubeVirt runs virtual machines.

Regular Pod shares the host kernel.

KubeVirt VM has its own Guest OS, for example:

    Ubuntu
    CentOS
    Debian
    Windows
    Rocky Linux
    AlmaLinux

This is also the key reason it can support traditional applications.

---

## VIII. Relationship Between KubeVirt and vSphere

vSphere is a traditional virtualization platform.

It is responsible for:

    1. Creating virtual machines
    2. Managing virtual machines
    3. Allocating CPU / Memory / Disk
    4. Managing virtual networks
    5. Taking snapshots
    6. Performing migrations
    7. Implementing HA
    8. Managing resource pools

KubeVirt also manages virtual machines, but the management entry has become Kubernetes.

Comparison:

| Comparison Item | vSphere | KubeVirt |
|---|---|---|
| Management Entry | vCenter | Kubernetes API |
| Resource Model | VM, Host, Cluster, Datastore | CRD, Pod, PVC, Node |
| Scheduling System | vSphere DRS etc. | Kubernetes Scheduler |
| Virtualization Layer | ESXi | KVM / QEMU |
| Storage | Datastore / vSAN / SAN | PVC / CSI / Longhorn / Ceph |
| Network | vSwitch / dvSwitch | CNI / Multus |
| Operation Method | vCenter UI / PowerCLI | kubectl / virtctl / YAML |
| Platform Positioning | Traditional Virtualization | Cloud-Native Virtualization |

Simple Understanding:

    vSphere is a mature traditional virtualization platform.
    KubeVirt is the virtualization capability within the Kubernetes ecosystem.

---

## IX. Relationship Between KubeVirt and OpenStack

OpenStack is a traditional private cloud platform.

It typically includes:

    Nova
    Neutron
    Cinder
    Glance
    Keystone
    Horizon

Corresponding Capabilities:

    Nova manages compute
    Neutron manages network
    Cinder manages block storage
    Glance manages images
    Keystone manages authentication

Both KubeVirt and OpenStack can manage virtual machines, but they have different design philosophies.

| Comparison Item | OpenStack | KubeVirt |
|---|---|---|
| Platform Architecture | Independent private cloud architecture | Kubernetes extension |
| Virtual Machine Management | Nova | KubeVirt CRD |
| Network | Neutron | CNI / Multus |
| Storage | Cinder | PVC / CSI |
| Image | Glance | CDI / DataVolume / PVC |
| Authentication | Keystone | Kubernetes RBAC |
| Operation Complexity | High | Depends on Kubernetes ecosystem |

Simple Understanding:

    OpenStack is a complete private cloud platform.
    KubeVirt is adding virtual machine capabilities to Kubernetes.

---

## X. Scenarios Where KubeVirt Is Suitable

KubeVirt is suitable for the following scenarios:

    1. Enterprises already have a Kubernetes platform
    2. There are still traditional virtual machine workloads needing hosting
    3. Want to unify management of containers and virtual machines
    4. Want to gradually migrate traditional workloads to cloud-native platforms
    5. Need to run applications that cannot be containerized
    6. Need to run complete operating systems in Kubernetes
    7. Private cloud platforms want to evolve toward cloud-native architecture
    8. Edge computing scenarios need to unify management of lightweight virtual machines and containers
    9. Development/test environments need to quickly create VMs
    10. Platform teams want to manage VM lifecycle via Kubernetes API

Typical Scenarios:

    Traditional workload migration
    Hybrid container and virtual machine platform
    Private cloud alternative exploration
    Development/test virtual machines
    Special workloads requiring complete OS
    Legacy applications unsuitable for containerization

---

## XI. Scenarios Where KubeVirt Is Not Suitable

KubeVirt is not suitable for blindly replacing all virtualization platforms.

Unsuitable Scenarios:

    1. Pure containerized applications that don't need virtual machines
    2. Traditional virtualization environments highly dependent on vSphere enterprise features
    3. Scenarios with extremely complex virtualization capabilities
    4. Teams without Kubernetes operation capabilities
    5. Teams without storage and network troubleshooting capabilities
    6. Nodes without hardware virtualization support
    7. No stable CSI storage
    8. No VM network planning
    9. Just want to run a few containers
    10. Production environments without backup, monitoring, migration, and disaster recovery solutions

Note:

    KubeVirt is not for making everything a virtual machine.
    KubeVirt is for enabling workloads that must retain virtual machine form to enter Kubernetes management.

---

## XII. Overview of KubeVirt Core Components

After installation, KubeVirt includes multiple core components.

Common Components:

    virt-api
    virt-controller
    virt-handler
    virt-launcher
    virt-operator

Simple Understanding:

### 12.1 virt-operator

Responsible for KubeVirt's installation, upgrade, and lifecycle management.

Equivalent to:

    Operator for managing KubeVirt components.

---

### 12.2 virt-api

Provides KubeVirt-related API capabilities.

Examples:

    virtctl console
    virtctl start
    virtctl stop
    virtctl restart

These operations require interaction with virt-api.

---

### 12.3 virt-controller

Responsible for creating and managing corresponding resources based on the desired state of VirtualMachine / VMI.

Similar to the role of a Kubernetes controller.

---

### 12.4 virt-handler

Runs on each node that supports virtualization.

Responsible for managing the lifecycle of virtual machines on the node.

It interacts with the virtualization capabilities on the node, such as:

    KVM
    QEMU
    libvirt-related capabilities

---

### 12.5 virt-launcher

Each running virtual machine typically corresponds to a virt-launcher Pod.

It is the Pod that actually hosts the virtual machine process.

When troubleshooting VMs, it's often necessary to check:

    virt-launcher Pod
    virt-launcher logs
    VMI events
    The node where the VM resides

---

## Thirteen, Key Objects in KubeVirt

### 13.1 VirtualMachine

VirtualMachine is the definition object for a virtual machine.

It describes:

    1. Virtual machine name
    2. CPU
    3. Memory
    4. Disk
    5. Network
    6. Boot strategy
    7. Whether it is running

It can be understood as:

    The desired state of the virtual machine.

---

### 13.2 VirtualMachineInstance

VirtualMachineInstance represents a running virtual machine instance.

Abbreviated as:

    VMI

It can be understood as:

    The virtual machine currently running.

If the VM stops, the VMI typically disappears.

---

### 13.3 DataVolume

DataVolume is usually provided by CDI for importing virtual machine images and preparing disks.

Common uses include:

    1. Importing a qcow2 image from HTTP
    2. Importing an image from a registry
    3. Cloning a PVC
    4. Creating a virtual machine boot disk

It can be understood as:

    The process of preparing disk data for a virtual machine.

---

### 13.4 PVC

Virtual machine disks typically reside on PVCs.

For example:

    Ubuntu virtual machine system disk
    Windows virtual machine system disk
    Data disk

In KubeVirt, storage depends on Kubernetes' storage system.

In other words:

    KubeVirt's disk capabilities cannot exist without StorageClass, PV, PVC, and CSI.

---

### 13.5 virtctl

virtctl is the command-line tool for KubeVirt.

Common operations include:

    Starting a virtual machine
    Stopping a virtual machine
    Restarting a virtual machine
    Connecting to console
    Uploading an image
    Viewing VNC
    Triggering migration

It can be understood as:

    kubectl manages Kubernetes resources.
    virtctl manages KubeVirt virtual machine operations.

---

## Fourteen, Understanding Storage in KubeVirt

A virtual machine must have a disk.

In KubeVirt, common sources for virtual machine disks include:

    1. PVC
    2. DataVolume
    3. ContainerDisk
    4. CloudInitNoCloud
    5. EmptyDisk

### 14.1 PVC

The most common production method.

Virtual machine disk data resides in a PVC.

Suitable for:

    Production VMs
    VMs requiring persistent data
    VMs needing to retain data after restart

---

### 14.2 DataVolume

Used for importing images or preparing disks.

For example:

    Creating a boot disk from a qcow2 image.

Common chain:

    HTTP image address
        |
        v
    DataVolume
        |
        v
    PVC
        |
        v
    VM boot disk

---

### 14.3 ContainerDisk

Packages the virtual machine disk image into a container image.

Suitable for:

    Testing
    Demo
    Quick experience

Not suitable for:

    Production persistent virtual machine system disks

---

### 14.4 CloudInitNoCloud

Used to inject initialization configuration into a virtual machine.

Similar to cloud-init for cloud hosts.

Can configure:

    Username
    Password
    SSH key
    Hostname
    Initialization commands

---

## Fifteen, Understanding Networking in KubeVirt

KubeVirt virtual machines also need networking.

Common networking modes include:

    1. Default Pod network
    2. Service exposure
    3. Multus multi-NIC
    4. Bridge network
    5. Masquerade network
    6. SR-IOV high-performance network

In the initial stages, understand:

    VMs can default access Kubernetes Pod network.
    VMs can expose ports via Service.
    If traditional Layer 2 network or multi-NIC access is needed, Multus is typically used.

Simple relationship:

    VM
     |
     v
    virt-launcher Pod network
     |
     v
    CNI network
     |
     v
    Service / Ingress / Gateway / External access

If a company asks about multi-NIC:

    KubeVirt often combines with Multus to attach multiple NICs to a VM.
    One for management network.
    One for business network.
    One for storage or dedicated network.

---

## Sixteen, Understanding Scheduling in KubeVirt

KubeVirt virtual machines are still scheduled by Kubernetes.

A virtual machine will be scheduled to a specific node.

Scheduling considers:

    1. CPU request
    2. Memory request
    3. Whether the node is Ready
    4. Whether the node supports KVM
    5. nodeSelector
    6. affinity
    7. taints / tolerations
    8. storage topology
    9. network capabilities
    10. Resource availability

If a VM fails to start, common reasons include:

    1. Node does not support virtualization
    2. /dev/kvm does not exist
    3. Insufficient resources
    4. PVC is not Bound
    5. Image import failed
    6. virt-handler abnormal
    7. virt-launcher Pod abnormal
    8. Network configuration error

---

## Seventeen, Comparison of KubeVirt and Traditional Virtual Machine Lifecycle

Common operations in traditional vSphere:

# Creating a Virtual Machine  
# Power On  
# Power Off  
# Reboot  
# Mount ISO  
# Add Disk  
# Modify CPU / Memory  
# Take Snapshot  
# Migrate  
# Delete Virtual Machine  

KubeVirt also has similar concepts:  

    Create VirtualMachine  
    start VM  
    stop VM  
    restart VM  
    console VM  
    Add disk using PVC  
    Prepare system disk using DataVolume  
    LiveMigration  
    Delete VM  

But the operation entry point becomes:  

    kubectl  
    virtctl  
    YAML  
    Kubernetes API  

---

## 18. Requirements for Operations with KubeVirt  

Learning KubeVirt requires the following basics:  

    1. Kubernetes basic resource objects  
    2. Pod / Deployment / Service  
    3. PV / PVC / StorageClass  
    4. CNI network basics  
    5. Node scheduling basics  
    6. Linux system basics  
    7. Virtualization basics  
    8. KVM / QEMU basics  
    9. Image format basics  
    10. Event and log troubleshooting capabilities  

Thus, it is more advanced than deploying ordinary K8s applications.  

However, the learning order can reduce the difficulty:  

    First understand the concepts  
    Then install  
    Then create the first VM  
    Then learn storage  
    Then learn networking  
    Finally learn migration and advanced capabilities  

---

## 19. How to Answer "What is KubeVirt?" in an Interview  

You can answer like this:  

    KubeVirt is a virtualization extension on Kubernetes, which abstracts virtual machines into Kubernetes resources through CRD for management.  
    It allows Kubernetes to not only manage Pods but also VMs.  
    Virtual machines ultimately run as virt-launcher Pods on nodes, relying on KVM/QEMU at the bottom layer.  
    Storage typically uses PVC, and image import can combine CDI and DataVolume.  
    Networking can use the default Pod network or combine with Multus for multi-NIC or traditional Layer 2 network access.  
    It is suitable for enterprises to unify container and traditional VM workloads on the Kubernetes platform.  

This answer is suitable for most basic interview scenarios.  

---

## 20. How to Answer "Why is KubeVirt Needed?" in an Interview  

You can answer like this:  

    Many legacy business systems in enterprises still run on virtual machines and cannot be containerized immediately.  
    If virtual machines and containers are placed on separate platforms, it would lead to fragmented permissions, networking, storage, monitoring, and operations processes.  
    KubeVirt can integrate virtual machine workloads into Kubernetes for unified management, allowing the platform to simultaneously support containers and virtual machines.  
    It is more suitable as a transition solution for traditional virtualization to cloud-native platforms.  

---

## 21. How to Answer "What's the Difference Between KubeVirt and Pod?" in an Interview  

You can answer like this:  

    Ordinary Pods run container processes that share the host kernel.  
    KubeVirt runs complete virtual machines with their own Guest OS.  
    In Kubernetes, KubeVirt uses virt-launcher Pods to host virtual machine processes, relying on KVM/QEMU at the bottom layer.  
    So from a Kubernetes perspective, it is a Pod, but from a business perspective, it is a virtual machine.  
    It is suitable for running applications that require a full OS or cannot be containerized yet.  

---

## 22. How to Answer "What's the Difference Between KubeVirt and vSphere?" in an Interview  

You can answer like this:  

    vSphere is a traditional virtualization platform, with vCenter as the management entry point and ESXi as the underlying layer.  
    KubeVirt is a virtualization extension on Kubernetes, with Kubernetes API as the management entry point and KVM/QEMU on Linux nodes as the underlying layer.  
    vSphere is more oriented toward traditional virtualization management with mature capabilities.  
    KubeVirt is more oriented toward cloud-native platform integration, suitable for unifying virtual machines and containers under Kubernetes management.  
    They are not completely replaceable, but more depends on the enterprise platform architecture and business migration direction.  

---

## 23. How to Answer "What Are the Core Components of KubeVirt?" in an Interview  

You can answer like this:  

    KubeVirt's main components include virt-operator, virt-api, virt-controller, virt-handler, and virt-launcher.  
    virt-operator is responsible for managing the lifecycle of KubeVirt components.  
    virt-api provides KubeVirt API capabilities.  
    virt-controller is responsible for controlling based on the desired state of VM/VMI.  
    virt-handler runs on the node side, responsible for virtual machine lifecycle management.  
    virt-launcher is the Pod corresponding to each running virtual machine, hosting the virtual machine process.  

---

## 24. Learning Focus for KubeVirt  

In the initial stage, focus on mastering:  

    1. What is KubeVirt  
    2. The difference between VM and VMI  
    3. What is virt-launcher Pod  
    4. Why virtual machine disks depend on PVC  
    5. What are CDI/DataVolume for  
    6. How VMs access networks  
    7. How to start, stop, and enter console for VM  
    8. Which resources to check when VM fails to start  
    9. The difference between KubeVirt and vSphere  
    10. What scenarios is KubeVirt suitable for  

Do not need to delve into these initially:  

    1. LiveMigration details  
    2. SR-IOV high-performance networking  
    3. NUMA/CPU pinning  
    4. Deep optimization for Windows VM  
    5. Large-scale production architecture  
    6. Multi-tenant virtualization platform design  
    7. Complex disaster recovery and backup strategies  

---

## 25. Learning Route Suggestions  

It is recommended to learn KubeVirt in the following order:  

    1. Understand what KubeVirt is  
    2. Understand VM/VMI/virt-launcher  
    3. Check node KVM capabilities  
    4. Install KubeVirt  
    5. Install virtctl  
    6. Create the first VM  
    7. Enter console  
    8. Use PVC as virtual machine disk  
    9. Use CDI/DataVolume to import images  
    10. Learn VM networking and Service exposure  
    11. Learn Multus multi-NIC  
    12. Learn VM startup failure troubleshooting  
    13. Learn advanced capabilities like migration, snapshots, and backups  

---

## 26. Common Misconceptions  

### 26.1 Misconception 1: KubeVirt is a replacement for Kubernetes  

Incorrect.

KubeVirt is an extension of Kubernetes, not a replacement for Kubernetes.

It relies on Kubernetes to run.

---

### 26.2 Misconception 2: KubeVirt is a replacement for all vSphere scenarios

Inaccurate.

KubeVirt can host virtual machines, but this does not mean all vSphere scenarios should be migrated to KubeVirt.

Mature virtualization platforms have extensive enterprise capabilities. Whether migration is appropriate depends on business scenarios, team capabilities, storage network conditions, and platform objectives.

---

### 26.3 Misconception 3: KubeVirt virtual machines are ordinary Pods

Inaccurate.

From a Kubernetes resource perspective, it hosts virtual machine processes through Pods.

However, from the perspective of runtime workloads, it is a complete virtual machine with an independent Guest OS.

---

### 26.4 Misconception 4: Just knowing Kubernetes means you can operate KubeVirt

Not necessarily.

KubeVirt also involves:

    KVM
    QEMU
    Virtual machine images
    PVC
    CSI
    Multiple network interfaces
    Virtual machine boot process
    Storage performance
    Network forwarding
    Migration and high availability

Thus, it is more complex than ordinary Kubernetes applications.

---

## Twenty-Seven, Summary of KubeVirt's Value

KubeVirt's value is not "showcasing virtual machine capabilities within Kubernetes."

Its true value lies in:

    1. Unified management entry for containers and virtual machines
    2. Support for smooth migration of traditional workloads
    3. Reducing fragmentation among multiple platforms
    4. Leveraging Kubernetes' scheduling, permissions, declarative API, and automation capabilities
    5. Providing a new cloud-native virtualization path for private cloud platforms

For those with a vSphere background, KubeVirt is an excellent transitional direction.

Because it connects:

    Traditional virtualization experience
    Kubernetes operations capabilities
    Cloud-native platform engineering

---

## Twenty-Eight, Summary of This Article

KubeVirt can be understood as:

    A solution for running and managing virtual machines within Kubernetes.

Its core features include:

    1. Extending Kubernetes through CRD
    2. Using VirtualMachine to describe virtual machines
    3. Using VirtualMachineInstance to represent running virtual machines
    4. Using virt-launcher Pod to host virtual machine processes
    5. Relying on KVM / QEMU at theBottom
    6. Storage typically relies on PVC
    7. Image import usually combines with CDI / DataVolume
    8. Networking can use Pod networking or combine with Multus
    9. Suitable for unified management scenarios of containers and virtual machines
    10. Belongs to advanced platform engineering capabilities of Kubernetes

One-sentence memory:

    KubeVirt is not making all applications virtual machines,
    but enabling virtual machine workloads that cannot be containerized to enter Kubernetes' unified management system.

Recommended next reading:

    02-KubeVirt Core Architecture: VirtualMachine, VMI, virt-launcher, and Control Components.md