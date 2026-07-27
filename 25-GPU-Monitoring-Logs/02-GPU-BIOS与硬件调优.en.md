# 02-GPU-BIOS and Hardware Tuning

## Document Description

This document outlines the BIOS/UEFI settings, PCIe topology, NUMA configuration, IOMMU settings, power supply, cooling, hardware recognition, and node acceptance procedures that need attention before installing an operating system on a GPU server, installing NVIDIA drivers, or adding a Kubernetes GPU node to a cluster.

This document does not directly cover the installation of NVIDIA drivers or the deployment of CUDA, Device Plugin, or GPU Operator. Instead, it focuses on more fundamental issues:

- Whether the server's BIOS/UEFI correctly recognizes the GPU;
- Whether there are sufficient PCIe resources;
- Whether the PCIe topology is appropriate for multi-GPU scenarios;
- Why Above 4G Decoding is important;
- The relationship between SR-IOV, IOMMU, and VT-d with GPU virtualization;
- How PCIe Speed/Width affects performance;
- The relationship between NUMA, GPUs, network cards, and CPU sockets;
- Why GPU power supply and cooling affect stability;
- How to perform hardware acceptance before a GPU node goes online;
- How to troubleshoot issues such as unrecognized GPUs, card failures, speed drops, and XID errors from hardware and BIOS levels.

After studying this document, you will find it easier to proceed with the following topics:

- 03-NVIDIA-Driver Installation and Verification
- 04-CUDA-Installation and Testing
- 05-K8S-GPU-Resource Concepts and Scheduling Principles
- 06-NVIDIA-Device-Plugin-and-Operator-Installation
- 07-GPU-Pod-Deployment and Scheduling Practice

## Tags

#GPU #BIOS #UEFI #PCIe #NUMA #NVIDIA #Server Hardware #AI Infrastructure #Kubernetes #SRE #Ops Troubleshooting

## Recommended Reading Path

Recommended reading path:

    06-GPU and AI Infrastructure/01-GPU Basics/02-GPU-BIOS and Hardware Tuning.md

---

## I. Why GPU Nodes Require BIOS and Hardware Tuning

In regular business servers, many Ops engineers rarely pay attention to the BIOS.

For Web services, databases, middleware, and ordinary Kubernetes Worker nodes, as long as the CPU, memory, disk, and network cards are recognized by the system, they can usually run smoothly.

However, GPU nodes are different.

GPUs are high-power-consuming, high-bandwidth, high-value computing resources that rely heavily on hardware.

The stability of a GPU server depends not only on the operating system and drivers but also on lower-level hardware initialization and BIOS/UEFI settings.

Common issues include:

- The NVIDIA GPU is not visible when executing `lspci` in the system;
- The BIOS recognizes the GPU, but the Linux system does not;
- The Linux system recognizes the GPU, but `nvidia-smi` reports an error;
- Only some of multiple GPUs are recognized;
- The GPU fails during stress testing;
- The GPU encounters XID errors after running for a while;
- The GPU is operating at PCIe x8 instead of the expected x16;
- The GPU is Gen3 instead of Gen4/Gen5;
- Multi-GPU training performance is significantly lower than expected;
- The GPU and high-speed network card are on different NUMA nodes, causing cross-Socket communication;
- The temperature rises too high when the node is at full load, causing the GPU to automatically reduce its speed;
- The order of GPUs changes after the server restarts, affecting scheduling and application binding;
- Secure Boot prevents the NVIDIA kernel module from loading successfully;
- In GPU virtualization or SR-IOV scenarios, devices cannot be passed through properly.

Therefore, before adding a GPU node to a Kubernetes cluster, it is essential to perform baseline checks at the hardware and BIOS levels first.

---

## II. GPU Node Initialization Process

The process by which a GPU node becomes available in Kubernetes goes through the following steps:

    Physical GPU installation
      ↓
    Power supply and cooling meet requirements
      ↓
    BIOS/UEFI initializes PCIe devices
      ↓
    PCIe resources are allocated
      ↓
    The Linux Kernel enumerates PCIe devices
      ↓
    The NVIDIA Driver loads kernel modules
      ↓
    `nvidia-smi` recognizes the GPU
      ↓
    The CUDA Runtime can access the GPU
      ↓
    The NVIDIA Container Toolkit exposes the GPU to containers
      ↓
    The Kubernetes Device Plugin registers `nvidia.com/gpu`
      ↓
    Pods request the GPU and scheduling is completed

This document focuses on the first half of this process:

    Physical GPU
    -> Power supply and cooling
    -> BIOS/UEFI
    -> PCIe
    -> NUMA
    -> Linux device recognition

If any step in this process is unstable, subsequent driver, CUDA, or Kubernetes scheduling will be affected.

---

## III. Hardware Checks Before a GPU Server Goes Online

BeforeAbove 4G Decoding, also known as decoding addresses above 4GB, allows the system to allocate MMIO resources for PCIe devices that exceed a 4GB address space. Devices such as GPUs, NVMe drives, and high-speed network cards often require larger address spaces. In multi-GPU servers, not enabling Above 4G Decoding may result in insufficient PCIe address space.

### 6.2 Why GPU Nodes Need It

GPUs, especially those used in data centers, typically require a large BAR (Base Address Register) address mapping space. In scenarios with multiple GPUs, if the BIOS cannot allocate enough MMIO address space for all GPUs, it may lead to issues such as:

- Some GPUs not being recognized;
- System startup failures;
- Only some GPUs appearing in the lspci output under Linux;
- Errors when using nvidia-smi;
- Failed PCIe resource allocation;
- Kernel logs showing "BAR allocation failed";
- GPU driver loading failures.

You might see messages similar to these in kernel logs:

    BAR 0: no space for
    BAR 1: failed to assign
    pci 0000:xx:xx.x: BAR: no space
    not enough MMIO resources

### 6.3 Recommended Configuration

It is recommended to enable Above 4G Decoding, especially in multi-GPU servers with 4, 8, or 10 GPUs.

### 6.4 Verification Methods

To check the number of GPUs after booting up the system:

    lspci | grep -i nvidia

To count the total number of GPUs:

    lspci | grep -i nvidia | wc -l

To view information through the NVIDIA management interface:

    nvidia-smi

To check for kernel PCIe errors:

    dmesg | grep -i "bar"
    dmesg | grep -i "pci"
    journalctl -k | grep -i "bar"

---

## VII. Resizable BAR

### 7.1 What is Resizable BAR

Resizable BAR is a feature of PCIe that allows the CPU to access larger areas of GPU memory mapping. In certain scenarios, it can improve data transfer efficiency between the CPU and the GPU. However, for server GPUs, AI training systems, and Kubernetes GPU environments, whether to enable Resizable BAR should be determined based on factors such as the server manufacturer, GPU model, driver version, and test results.

### 7.2 Differences Between Resizable BAR and Above 4G Decoding

Above 4G Decoding addresses the issue of allocating sufficient address space for PCIe devices, while Resizable BAR focuses on adjusting the size of the BAR mapping window. Simply put:

    Above 4G Decoding: Enables the system to allocate more than 4GB of address space for large PCIe devices.
    Resizable BAR: Allows the adjustment of the BAR mapping window size.

These two features are not interchangeable.

### 7.3 Production Recommendations

It is advised to follow the recommendations provided by both the server manufacturer and NVIDIA. Do not enable these features blindly just in hopes of potential performance improvements. Conduct stress tests after making any changes, pay attention to driver compatibility, and consider how virtualization and passthrough scenarios may be affected. To check BAR information:

    lspci -vvv -s <GPU_PCI_ID> | grep -i BAR

---

## VIII. SR-IOV

### 8.1 What is SR-IOV

SR-IOV, or Single Root I/O Virtualization, enables a physical PCIe device to be split into multiple virtual functions that can be used by virtual machines or different environments. This technology is commonly used in network cards but may also be applied in GPU scenarios depending on the specific requirements of the hardware and software stack.

### 8.2 When SR-IOV Is Needed for GPUs

SR-IOV might be required in the following situations:

- When GPUs are directly connected to virtual machines;
- In NVIDIA vGPU solutions;
- For cloud platform GPU instances;
- In multi-tenant virtualization environments;
- With KubeVirt and GPU passthrough;
- In bare metal cloud GPU resource pools.

For ordinary Kubernetes bare metal GPU nodes where Pods directly use GPUs, enabling SR-IOV is usually not necessary.

### 8.3 Recommended Configuration

If GPU virtualization or PCIe device virtualization is required:

    SR-IOV: Enabled

For ordinary bare metal Kubernetes GPU nodes:

    Follow the manufacturer's recommendations.
    Do not assume that all GPU nodes must have SR-IOV enabled.

### 8.4 Related Verification

To check the IOMMU groups on the system:

    find /sys/kernel/iommu_groups/ -type l

To view PCIe devices:

    lspci | grep -i nvidia

To check kernel parameters:

    cat /proc/cmdline

Common IOMMU parameters for Intel platforms:

    intel_iommu=on

For AMD platforms:

    amd_iommu=    x16
    x8/x8
    x4/x4/x4/x4

These configurations are commonly encountered in scenarios involving GPUs, NVMe expansion cards, and riser cards.

### 11.2 Why Do GPU Nodes Care About Bifurcation?

Multi-GPU servers may connect multiple GPUs through risers or PCIe switches.

If the PCIe Bifurcation configuration in the BIOS is incorrect, it may result in:

- Some GPUs not being recognized;
- Only a portion of devices being detected;
- Abnormal order of PCIe devices;
- The GPUs operating at an unexpected link width;
- Slow system startup or freezes.

### 11.3 Recommendations

Production tips:

- Do not modify the Bifurcation configuration unless you are familiar with the motherboard topology.
- Refer to the GPU installation documentation provided by the server manufacturer.
- Use original risers.
- Revalidate the lspci and LnkSta settings after replacing GPUs or risers.
- When delivering multi-GPU nodes, record the PCIe position of each GPU.

---

## Section XII: NUMA and CPU Socket Affinity

### 12.1 What is NUMA?

NUMA stands for Non-Uniform Memory Access.

In servers with multiple CPU sockets, different CPU sockets have varying latency when accessing local and remote memory.

GPUs, network cards, and NVMe devices are typically connected to the PCIe Root Complex of a specific CPU socket.

If an application process runs on CPU Socket 0, but the GPU is connected to CPU Socket 1, cross-NUMA access may occur.

### 12.2 Why Do GPU Scenarios Care About NUMA?

In multi-GPU training or high-performance inference tasks, NUMA has a significant impact.

Key considerations include:

- Which NUMA node the GPU belongs to;
- Which NUMA node the high-speed network card is connected to;
- Whether CPU processes are bound to the appropriate socket;
- Whether there is cross-Socket communication between multiple GPUs;
- Whether the connection between the GPU and the network card crosses sockets;
- Whether distributed training communications involve additional delays.

To view the NUMA topology:

    lscpu | grep -i numa

To check the NUMA node of a device:

    cat /sys/bus/pci/devices/0000:65:00.0/numa_node

To view the GPU topology:

    nvidia-smi topo -m

### 12.3 GPU and Network Card Topology

In AI training scenarios, especially for multi-node training, the topology between GPUs and high-speed network cards is crucial.

If the distance between the GPU and the RDMA network card is too great, communication efficiency will decrease.

To view PCIe devices:

    lspci | grep -i nvidia
    lspci | grep -i ethernet
    lspci | grep -i mellanox
    lspci | grep -i broadcom
    lspci | grep -i intel

To view the topology:

    nvidia-smi topo -m

Common identifiers in the output include:

    PIX: Indicates devices connected through the same PCIe switch.
    PXB: Indicates devices connected through multiple PCIe bridges.
    PHB: Indicates devices connected through a PCIe host bridge.
    SYS: Indicates cross-CPU socket or NUMA connections.
    NV#: Indicates devices connected via NVLink.

Production tips:

- Place GPUs and high-speed network cards as close to each other as possible.
- For distributed training nodes, pay attention to the GPU-NIC topology.
- For multi-GPU training, focus on the GPU-GPU topology.
- Do not rely solely on the number of GPUs; also consider their topology relative to network cards and CPUs.

---

## Section XIII: Power Profile and Performance Modes

### 13.1 BIOS Power Strategies

Server BIOS usually includes power and performance settings.

Common options include:

    Performance
    Balanced Performance
    Balanced Power
    Power Saving
    Custom

Different manufacturers may use different names for these options.

It is generally not recommended to use excessively power-saving strategies for GPU nodes.

### 13.2 Why Do Power Strategies Affect GPUs?

Although GPUs have their own independent power management capabilities, factors such as CPU power strategies, PCIe ASPM, and C-State can also affect overall performance and latency.

Possible consequences include:

- Fluctuations in CPU frequency;
    Slower data preprocessing;
    Increased PCIe latency;
    Fluctuations in inference latency;
    Unstable training throughput;
    Unstable full-load performance of the node.

### 13.3 Recommended Configurations

For GPU training or inference nodes, it is recommended to set:

    System Profile: Performance
    Power Regulator: Static High Performance or Maximum Performance
    CPU Power Management: Maximum Performance
    Memory Frequency: Maximum Performance
    PCIe ASPM: Disabled or turned off according to the manufacturer's performance recommendations
    C-State: Configure according to the requirements of business latency.

For energy-efficient offline    GPU0    GPU1    GPU2    GPU3    CPU Affinity    NUMA Affinity
    GPU0     X      NV2     SYS     SYS     0-31            0
    GPU1    NV2      X      SYS     SYS     0-31            0
    GPU2    SYS     SYS      X      NV2     32-63           1
    GPU3    SYS     SYS     NV2      X      32-63           1

### 18.2 How to Understand Topology

Common designations:

    X      Self
    PIX    Within the same PCIe Switch
    PXB    Multiple-level PCIe Bridge
    PHB    PCIe Host Bridge
    SYS    Across CPU Sockets / NUMA
    NV#    NVLink Connection

Generally speaking:

    NVLink is superior to PCIe;
    Devices within the same Socket are better than those across different sockets;
    PIX usually performs better than SYS;
    The closer the GPU is to the network card, the more beneficial it is for distributed training communications.

### 18.3 Production Recommendations

When multiple GPU nodes are deployed, record the following:

- The PCI address of each GPU;
- The NUMA node for each GPU;
- The GPU-GPU topology;
- The GPU-NIC topology;
- The CPU socket where the GPU is located;
- Whether there is an NVLink connection;
- Whether there is cross-socket communication.

It is recommended to save the following output:

    nvidia-smi topo -m > gpu-topology.txt
    lspci -tv > pci-tree.txt
    lscpu > cpu-topology.txt

---

## Section 19: Summary of BIOS Configuration Recommendations for GPU Nodes

The following are common BIOS/UEFI configuration recommendations for GPU nodes. In actual production, the settings should be based on the server manufacturer's recommendations, as well as those from the GPU manufacturer and business performance testing results.

### 19.1 Basic GPU Nodes

Suitable for:

- Kubernetes GPU Workers;
- AI inference tasks;
- Small-scale training scenarios;
- Bare-metal GPU nodes.

Recommendations:

    Above 4G Decoding: Enable
    PCIe Speed: Auto or the highest stable generation
    PCIe ASPM: Disable
    Power Profile: Performance
    Fan Profile: Performance / High Performance
    Secure Boot: Decide based on driver installation strategy
    SR-IOV: Use as needed
    VT-d / IOMMU: Use as needed
    NUMA: Keep default settings or follow manufacturer recommendations
    Hyper-Threading: Determine based on business performance testing results

### 19.2 GPU Virtualization Nodes

Suitable for:

- vGPUs;
- GPU passthrough;
- KubeVirt;
- Cloud platform GPU instances.

Recommendations:

    Above 4G Decoding: Enable
    SR-IOV: Enable
    VT-d / AMD-Vi: Enable
    IOMMU: Enable
    PCIe ACS: Configure according to virtualization requirements
    Secure Boot: Combine with driver signature strategy
    Power Profile: Performance
    Fan Profile: Performance

### 19.3 Training-oriented Multi-GPU Nodes

Suitable for:

- Multi-card training;
- Large model training;
- Distributed training;
- High-speed network card communications.

Recommendations:

    Above 4G Decoding: Enable
    PCIe Speed: Auto / Gen4 / Gen5, choose based on stability tests
    PCIe ASPM: Disable
    Power Profile: Maximum Performance
    Fan Profile: High Performance
    C-State: Determine based on latency and throughput tests
    NUMA: Pay special attention to GPU-NIC affinity
    SR-IOV: Not required if there is no virtualization need
    VT-d / IOMMU: Configure according to platform specifications

---

## Section 20: Operating System-Level Verification Commands

After completing the BIOS configuration and restarting the system, it is necessary to verify the settings at the operating system level.

### 20.1 Checking if the GPU is Recognized by PCIe

    lspci | grep -i nvidia

If there is no output, it means that the system has not detected the NVIDIA PCIe device.

Priority troubleshooting steps:

- Check if the GPU is properly inserted;
- Verify if the BIOS recognizes it;
- Ensure that Above 4G Decoding is enabled;
- Confirm if the PCIe slot is correct;
- Check if the riser card is compatible;
- Verify if the power supply is functioning correctly;
- Determine if there is any hardware failure with the GPU.

### 20.2 Viewing the PCIe Tree

    lspci -tv

To save the topology information:

    lspci -tv > pci-tree.txt

### 20.3 Checking PCIe Speed and Width

First, identify the PCI ID:

    lspci | grep -i nvidia

---

## Twenty-Eight, Common Fault Three: Only Some GPUs Are Recognized

### 28.1 Phenomenon

A server is equipped with 4 GPUs, but the system only recognizes 2 of them.

When executing:

    lspci | grep -i nvidia
    nvidia-smi

The displayed number is incomplete.

### 28.2 Possible Causes

Possible reasons include:

- Above 4G Decoding is not enabled;
- Insufficient PCIe address space;
- Incorrect riser configuration;
- Incorrect PCIe Bifurcation configuration;
- A certain slot is disabled;
- Inadequate GPU power supply;
- A single GPU is malfunctioning;
- The BIOS version does not support the current configuration;
- Insufficient CPU PCIe lanes;
- The combination of slots and devices is not supported by the manufacturer.

### 28.3 Troubleshooting Steps

Check the BIOS:

    Above 4G Decoding
    PCIe Slot Enable
    PCIe Bifurcation
    PCIe Speed
    System Event Log

Check the system:

    lspci -tv
    dmesg | grep -i pci
    dmesg | grep -i bar
    ipmitool sel list

Perform cross-testing:

    Swap GPU slots;
    Swap risers;
    Test with a single GPU;
    Add GPUs one by one and test.

### 28.4 Handling Suggestions

Solutions include:

- Enable Above 4G Decoding;
- Use the slot combination recommended by the manufacturer;
- Restore the recommended configuration for Bifurcation;
- Upgrade the BIOS;
- Check the riser;
- Verify the GPU power supply;
- Contact the manufacturer to confirm the maximum number of GPUs supported.

---

## Twenty-Five, Common Fault Four: PCIe Speed or Width Reduction

### 25.1 Phenomenon

Expected:

    PCIe Gen4 x16

Actual:

    PCIe Gen3 x8

To check:

    lspci -vvv -s <GPU_PCI_ID> | grep -i "LnkCap"
    lspci -vvv -s <GPU pci_ID> | grep -i "LnkSta"

### 25.2 Possible Causes

Possible reasons include:

- The electrical specifications of the slot are only x8;
- The riser only supports x8;
- Channel sharing on the motherboard;
- The BIOS sets a low speed;
- Quality issues with the PCIe signal;
- The GPU is not securely inserted;
- Design limitations imposed by the server manufacturer;
- Insufficient CPU PCIe lanes;
- Automatic downgrading from Gen4 to Gen3 due to compatibility issues.

### 25.3 Handling Suggestions

Solutions include:

- Refer to the server hardware manual;
- Replace with an x16 slot;
- Verify the riser specifications;
- Set the PCIe Speed to Auto or the target Gen in the BIOS;
- Update the BIOS/Firmware;
- Replug the GPU;
- Compare results of stress tests;
- If fixing Gen4 is unstable, temporarily fix it at Gen3 to verify stability.

---

## Twenty-Six, Common Fault Five: GPU Drops During Stress Testing

### 26.1 Phenomenon

nvidia-smi works normally when idle, but during training or stress testing, the following issues occur:

- The GPU disappears;
- nvidia-smi freezes;
- XID errors appear;
- The node restarts;
- Training tasks fail;
- Pods exit abnormally;
- The GPU temperature becomes too high;
- PCIe AER errors are logged in the system logs.

### 26.2 Possible Causes

Possible reasons include:

- Insufficient power supply;
- Abnormalities with the GPU power cable;
- Inadequate cooling;
- Contact issues with the PCIe slot;
- Problems with the riser;
- Hardware failure of the GPU;
- Driver issues;
- BIOS power management settings;
- Compatibility issues with PCIe ASPM;
- Excessively high room temperature.

### 26.3 Troubleshooting Commands

Check for XID errors:

    dmesg | grep -i xid
    journalctl -k | grep -i xid

Check for PCIe AER errors:

    dmesg | grep -i aer
    journalctl -k | grep -i aer

Check temperature and power consumption:

    nvidia-smi
    nvidia-smi -q

Check BMC events:

    ipmitool sel list
    ipmitool sensor

Monitor continuously:

    watch -n 2 nvidia-smi

### 26.4 Handling Suggestions

Solutions include:

- Check the room temperature;
- Switch fans to performance mode;
- Inspect the GPU power cable;
- Verify the power module;
- Replace the riser;
- Swap PCIe slots;
- Temporarily fix Gen4 if it causes instability and verify stability at Gen3;
- Update the BIOS and firmware;
- Update or revert the NVIDIA driver;
-VT-d / IOMMU:  
Secure Boot:  
PCIe Speed:  
PCIe Width:  
NVIDIA Driver:  
CUDA Version:  
Container Runtime:  
Kubernetes Version:  
Device Plugin / GPU Operator Version:  
Reviewer:  
Review Date:  

It is recommended to save the output of the following commands:  

```bash
lspci | grep -i nvidia > gpu-lspci.txt  
lspci -tv > pci-tree.txt  
lscpu > lscpu.txt  
nvidia-smi > nvidia-smi.txt  
nvidia-smi -q > nvidia-smi-query.txt  
nvidia-smi topo -m > gpu-topology.txt  
dmidecode -t bios > bios-info.txt  
dmidecode -t system > system-info.txt  
ipmitool sensor > ipmi-sensor.txt  
ipmitool sel list > ipmi-sel.txt  
```  

---

## Section 29: Preliminary Checks for Kubernetes GPU Nodes  

Although this document does not delve into Kubernetes GPU scheduling, the following items must be confirmed before adding a GPU node to the cluster.  

### 29.1 Node Naming Recommendations  

It is recommended that GPU nodes be named in a way that makes them easily identifiable:  
`k8s-gpu-node01`, `k8s-gpu-node02`, `ai-gpu-a100-01`, `ai-gpu-l4-01`.  
Avoid using completely meaningless names.  

### 29.2 Node Label Planning  

It is recommended to set labels for nodes after they are added to the cluster:  
```bash
kubectl label node <node-name> node-role.kubernetes.io/gpu=true  
kubectl label node <node-name> accelerator=nvidia  
kubectl label node <node-name> gpu.vendor=nvidia  
kubectl label node <node-name> gpu.model=a100  
```  
If there are different GPU models, you can add additional labels:  
`gpu.memory=80gb`, `gpu.partition=mig-enabled`, `gpu.workload=inference`, `gpu.workload=training`.  

### 29.3 Node Taint Planning  

It is recommended to add a taint to GPU nodes to prevent them from being scheduled for regular Pods:  
```bash
kubectl taint node <node-name> nvidia.com/gpu=true:NoSchedule  
```  
GPU Pods need to explicitly specify tolerations to run on such nodes.  

### 29.4 Resource Planning  

Before adding a GPU node to Kubernetes, confirm the following:  
- [ ] Whether there are sufficient CPU resources for GPU task data preprocessing.  
- [ ] Whether there is enough memory.  
- [ ] Whether there is sufficient local disk space for caching models and data.  
- [ ] Whether the network supports image pulling and model downloading.  
- [ ] Whether the GPU monitoring ports have been configured.  
- [ ] Whether the node maintenance process is clear.  
```  

---

## Section 30: Precautions in a Production Environment  

### 30.1 Do Not Use GPU Nodes as Ordinary Workers  

GPU nodes are expensive, and it is not recommended to use them for ordinary workloads.  
Ordinary Pods can consume CPU, memory, disk, and network resources, which may affect GPU tasks.  
It is recommended to control the use of GPU nodes through:  
- Node labels;  
- Taints/tolerations;  
- Resource quotas;  
- Priority classes;  
- Namespace isolation;  
- Node pool planning.  

### 30.2 Do Not Rely Only on the Number of GPUs  

The capabilities of a GPU server depend not only on the number of GPUs but also on factors such as:  
- GPU model;  
- Video memory capacity;  
- PCIe generation;  
- PCIe width;  
- NVLink support;  
- CPU sockets;  
- NUMA configuration;  
- Network card topology;  
- Local disk space;  
- Cooling system;  
- Power supply;  
- Driver version;  
- CUDA compatibility.  

### 30.3 Do Not Ignore the BIOS Version  

The BIOS version can affect:  
- GPU compatibility;  
- PCIe device recognition;  
- Power management strategies;  
- Fan control settings;  
- NUMA behavior;  
- SR-IOV support;  
- IOMMU configuration;  
- Device startup order.  
In a production environment, it is recommended to:  
- Standardize the BIOS and BMC versions before deploying GPU nodes in batches.  
- Avoid having different BIOS versions among GPU nodes within the same batch.  

### 30.4 Do Not Skip Stress Testing Before Deployment  

Just because a GPU node works fine when idle does not mean it will remain stable under full load.  
It is essential to verify the following:  
- Single-card performance under full load;  
- Multi-card performance under full load;  
- Temperature levels;  
- Power