# 01-KubeVirt Introduction: Why Run Virtual Machines in Kubernetes

Recommended path:

    04-Kubernetes/12-KubeVirt/01-KubeVirt Introduction: Why Run Virtual Machines.md

Tags:

    #Kubernetes
    #KubeVirt
    #Virtualization
    #KVM
    #QEMU
    #Platform Engineering
    #SRE
    #Cloud-Native Virtualization
    #Private Cloud

---

## I. Document Overview

This article aims to help you understand the basic concepts of KubeVirt, focusing on the following aspects:

    1. What KubeVirt is.
    2. Why virtual machines are still needed in Kubernetes.
    3. The problems KubeVirt solves.
    4. Differences between KubeVirt and regular Pods.
    5. The relationship between KubeVirt and vSphere/OpenStack.
    6. Suitable use cases for KubeVirt.
    7. Inappropriate scenarios for KubeVirt.
    8. How to answer KubeVirt-related questions in interviews.

This article does not cover installation instructions; it is primarily intended to provide an overall understanding of the topic.

---

## II. What is KubeVirt

KubeVirt is a virtualization extension for Kubernetes.

Its main goal is to enable Kubernetes to manage not only containers but also virtual machines.

In simple terms:

    - Kubernetes is originally designed to manage Pods.
    - KubeVirt adds the ability to manage virtual machines as well.

To put it more clearly:

    Before KubeVirt:
        vSphere manages virtual machines.
        Kubernetes manages containers.

    With KubeVirt:
        Kubernetes can now manage both containers and virtual machines.

Essentially, KubeVirt does not create a new virtualization platform; instead, it integrates virtual machine capabilities into Kubernetes' resource model.

---

## III. Why Are Virtual Machines Still Needed in Kubernetes?

Many enterprises do not start out using a completely cloud-native approach. The typical environment might include:

    1. Legacy applications running on virtual machines.
    2. New applications built with containers.
    3. Some applications that cannot be containerized.
    4. Systems that require a full operating system.
    5. A desire to unify resource management across the platform.
    6. The need to reduce fragmentation between different platforms.

In traditional environments, virtual machines are often managed by tools like vSphere/OpenStack, while containers are managed by Kubernetes. This can lead to several issues:

    - Fragmentation of platforms and their underlying systems.
    - Disparate permission models.
    - Inconsistent monitoring and management tools.
    - Different network and storage architectures.
    - Varied deployment processes.
    - Separate operations and maintenance interfaces.
    - Lack of unified automation capabilities.

KubeVirt aims to address these challenges by allowing both containers and virtual machines to run on the same Kubernetes platform, thus reducing fragmentation.

---

## IV. Core Problems Solved by KubeVirt

KubeVirt primarily addresses the following issues:

    1. Incorporating virtual machine workloads into Kubernetes management.
    - Facilitating the gradual migration of legacy applications to a cloud-native environment.
    - Enabling unified scheduling of containers and virtual machines.
    - Providing common access to Kubernetes networking, storage, permissions, and resources for both types of workloads.
    - Reducing the gap between traditional virtualization platforms and container-based solutions.
    - Supporting the continued operation of non-containerizable legacy applications.
    - Offering a unified foundation for both containers and virtual machines in private cloud environments.

In short, KubeVirt acts as a bridge between traditional virtualization and Kubernetes.

---

## V. How KubeVirt Works

KubeVirt extends Kubernetes by using Custom Resource Definitions (CRDs) to define virtual machine-related resources. Common resources include:

    - VirtualMachine
    - VirtualMachineInstance
    - VirtualMachineInstanceReplicaSet
    - DataVolume
    - VirtualMachineInstanceMigration

The most fundamental ones are:

    - VirtualMachine: Describes the desired configuration of a virtual machine.
    - VirtualMachineInstance: Represents an actually running virtual machine instance.

You can think of it this way:

    - VirtualMachine: Similar to a “specification” for how a virtual machine should be configured.
    - VirtualMachineInstance: Similar to an “instance” that represents the actual running state of that virtual machine.

Comparing it to Kubernetes' native resources:

    - Deployment: Describes the desired state of a deployed application.
    - Pod: Represents an actually running container instance.

In KubeVirt terms:

    - VirtualMachine: Describes the desired state of a virtual machine.
    - VirtualMachineInstance: Represents an actually running virtual machine instance.

---

## VI. Where Do Virtual Machines Actually Run?

Virtual machines in KubeVirt do not exist| Authentication | Keystone | Kubernetes RBAC |
| Operational Complexity | Relatively high | Dependent on the Kubernetes ecosystem |

Simple explanation:

    OpenStack is a complete private cloud platform.
    KubeVirt enhances virtual machine capabilities within Kubernetes.

---

## Section 10: Use Cases for KubeVirt

KubeVirt is suitable for the following scenarios:

    1. Enterprises that already have a Kubernetes platform.
    2. Situations where traditional virtual machines still need to be managed.
    3. When it's necessary to manage both containers and virtual machines uniformly.
    4. For gradual migration of traditional services to cloud-native platforms.
    5. For running applications that cannot be containerized.
    6. When a complete operating system needs to run within Kubernetes.
    7. Private cloud platforms looking to evolve towards cloud-native architectures.
    8. In edge computing scenarios where it's required to manage lightweight virtual machines and containers together.
    9. In development and testing environments where VMs need to be quickly created.
    10. When the platform team wishes to manage the lifecycle of VMs through Kubernetes APIs.

Typical use cases:

    Transitioning from traditional services to cloud-based solutions.
    Hybrid platforms combining containers and virtual machines.
    Exploring alternatives for private clouds.
    Development and testing virtual machines.
    Special workloads that require a complete operating system.
    Legacy applications that are not suitable for containerization.

---

## Section 11: Scenarios Where KubeVirt Is Not Suitable

KubeVirt should not be blindly used to replace all virtualization platforms. It is not appropriate in the following cases:

    1. For applications that are purely containerized and do not require virtual machines.
    2. In traditional virtualization environments that rely heavily on enterprise-specific features of vSphere.
    3. In scenarios where very advanced virtualization capabilities are required.
    4. When a team lacks the necessary Kubernetes operational skills.
    5. When the team does not have the capability to troubleshoot storage and network issues.
    6. When nodes do not support hardware virtualization.
    7. In situations without stable CSI storage solutions.
    8. When there is no plan for managing VM networks.
    9. For those who simply want to run a few containers.
    10. In production environments where backup, monitoring, migration, and disaster recovery mechanisms are not in place.

It's important to note that KubeVirt is not intended to convert everything into virtual machines. Its purpose is to allow businesses that still require virtual machine models to be managed within the Kubernetes framework as well.

---

## Section 12: Overview of Key Components in KubeVirt

After installation, KubeVirt includes several core components. Some common ones are:

    virt-api
    virt-controller
    virt-handler
    virt-launcher
    virt-operator

Simple explanation:

### 12.1 virt-operator

Responsible for the installation, upgrade, and lifecycle management of KubeVirt itself.

Similar to:

    An Operator that manages KubeVirt components.

---

### 12.2 virt-api

Provides API functionality related to KubeVirt.

Examples:

    Operations like `virtctl console`, `virtctl start`, `virtctl stop`, `virtctl restart` require interaction with virt-api.

---

### 12.3 virt-controller

Responsible for creating and managing resources based on the desired state of VirtualMachine/VMI objects.

Similar to the role of a Kubernetes controller.

---

### 12.4 virt-handler

Runs on each node that supports virtualization.

It manages the lifecycle of virtual machines at the node level, interacting with the virtualization software on that node, such as KVM, QEMU, or libvirt-related components.

---

### 12.5 virt-launcher

Each running virtual machine typically corresponds to a virt-launcher Pod. This Pod is where the actual virtual machine processes are executed. When troubleshooting virtual machines, it is often necessary to check the virt-launcher Pod, its logs, VMI events, and the node on which the VM resides.

---

## Section 13: Key Objects in KubeVirt

### 13.1 VirtualMachine

The VirtualMachine object defines a virtual machine.

It includes information such as:

    1. Name of the virtual machine.
    2. CPU resources.
    3. Memory capacity.
    4. Disk configuration.
    5. Network settings.
    6. Startup parameters.
    7. Whether the virtual machine is currently running or not.

It can be understood as:

    The desired state of a virtual machine.

---

### 13.2 VirtualMachineInstance

The VirtualMachineInstance object represents an actual running virtual machine instance.

Abbreviation:

    VMI

It refers to:

    A virtual machine that is currently in operation. If the VM### Summary of KubeVirt

KubeVirt can be understood as a solution for running and managing virtual machines within Kubernetes. Its key features include:

1. Extending Kubernetes through Custom Resource Definitions (CRDs).
2. Using the `VirtualMachine` resource type to define virtual machines.
3. Representing running virtual machines with the `VirtualMachineInstance` resource type.
4. Using `virt-launcher` Pods to host virtual machine processes.
5. Relying on KVM/QEMU at the underlying layer for virtualization.
6. Typically using PVCs for storage.
7. Combining CDI and DataVolume for image import.
8. Allowing use of both Pod networks and Multus for networking.
9. Being suitable for unified management of containers and virtual machines.
10. Constituting an advanced platform capability within Kubernetes.

In summary, KubeVirt isn’t intended to convert all applications into virtual machines; rather, it enables traditional virtual machine workloads that aren’t yet compatible with containers to be integrated into Kubernetes’ unified management framework.