# 02-GPU-BIOS and Hardware Tuning

## Document Overview

This document is used to organize the BIOS/UEFI configuration, PCIe topology, NUMA, IOMMU, power supply, cooling, hardware identification, and node acceptance methods that need attention before installing the operating system, NVIDIA drivers, and adding Kubernetes GPU nodes to the cluster.

This document does not directly address NVIDIA driver installation, nor does it directly discuss CUDA, Device Plugin, or GPU Operator deployment. Instead, it focuses on more fundamental issues:

- Does the server BIOS/UEFI correctly identify the GPU?
- Are PCIe resources sufficient?
- Is the PCIe topology reasonable in multi-GPU scenarios?
- Why is Above 4G Decoding important?
- What is the relationship between SR-IOV, IOMMU, VT-d, and GPU virtualization?
- How does PCIe Speed/Width affect performance?
- What is the relationship between NUMA and GPU, network card, and CPU Socket?
- Why does GPU power supply and cooling affect stability?
- How should hardware acceptance be performed before GPU node activation?
- How to troubleshoot hardware and BIOS layer issues when GPU is not recognized, card drops, performance degrades, or XID errors occur.

After studying this document, proceeding to the following content will be smoother:

- 03-NVIDIA-Driver Installation and Verification
- 04-CUDA Installation and Testing
- 05-K8S-GPU-Resource Concepts and Scheduling Principles
- 06-NVIDIA-Device-Plugin-and-Operator Installation
- 07-GPU-Pod Deployment and Scheduling Practical Guide

## Tags

#GPU #BIOS #UEFI #PCIe #NUMA #NVIDIA #ServerHardware #AiInfrastructure #Kubernetes #SRE #TransportBarriers

## Recommended Path

Recommended path:

    06-GPU-and-AI-Infrastructure/01-GPU-Base/02-GPU-BIOS-and-Hardware-Tuning.md

---

## I. Why GPU Nodes Need to Pay Attention to BIOS and Hardware Tuning

In ordinary business servers, many operations engineers rarely pay attention to BIOS.

For web services, databases, middleware, and ordinary Kubernetes Worker nodes, as long as CPU, memory, disk, and network card are recognized by the system, they can often run normally.

But GPU nodes are different.

GPU is a high-power, high-bandwidth, high-value, and strongly hardware-dependent computing resource.

The stability of a GPU server is not only dependent on the operating system and drivers, but also on the lower-level hardware initialization and BIOS/UEFI configuration.

Common issues include:

- `lspci` shows no NVIDIA GPU in the system;
- BIOS can see GPU, but Linux system cannot;
- Linux can see GPU, but `nvidia-smi` reports errors;
- Multiple GPUs only recognize part of them;
- GPU drops during stress testing;
- XID errors occur after GPU runs for a period;
- GPU operates at PCIe x8 instead of expected x16;
- GPU runs at Gen3 instead of Gen4/Gen5;
- Multi-GPU training performance is significantly lower than expected;
- GPU and high-speed network card are not on the same NUMA node, causing cross-socket communication;
- High temperature after node full load causes GPU auto-frequency reduction;
- GPU order changes after server reboot, affecting scheduling and application binding;
- Secure Boot causes NVIDIA kernel module loading failure;
- GPU virtualization or SR-IOV scenarios where devices cannot pass through.

Therefore, GPU nodes must complete baseline checks at the hardware and BIOS level before entering the Kubernetes cluster.

---

## II. GPU Node Initialization Chain

A GPU node from hardware to Kubernetes availability roughly goes through the following chain:

    Physical GPU Installation
      ↓
    Power and Cooling Requirements Met
      ↓
    BIOS/UEFI Initializes PCIe Devices
      ↓
    PCIe Resource Allocation
      ↓
    Linux Kernel Enumerates PCIe Devices
      ↓
    NVIDIA Driver Loads Kernel Module
      ↓
    `nvidia-smi` Recognizes GPU
      ↓
    CUDA Runtime Can Access GPU
      ↓
    NVIDIA Container Toolkit Exposes GPU to Containers
      ↓
    Kubernetes Device Plugin Registers `nvidia.com/gpu`
      ↓
    Pod Requests GPU and Completes Scheduling

This document focuses on the first half:

    Physical GPU
    -> Power and Cooling
    -> BIOS/UEFI
    -> PCIe
    -> NUMA
    -> Linux Device Recognition

If this layer is unstable, subsequent driver, CUDA, and Kubernetes scheduling will have anomalies.

---

## III. Hardware Checks Before GPU Server Activation

Before adjusting BIOS, first confirm there are no obvious hardware issues.

### 3.1 Confirm GPU Model and Quantity

Need to record:

- GPU model;
- GPU quantity;
- Memory capacity per GPU;
- Whether it's a data center card;
- Whether ECC is supported;
- Whether MIG is supported;
- Whether NVLink is supported;
- Whether external power supply is needed;
- Whether it's a passive-cooled card;
- Whether it's suitable for the current server chassis airflow.

Example record:

    Server model: Dell PowerEdge R750xa
    CPU: 2 x Intel Xeon
    GPU: 4 x NVIDIA A100 80GB PCIe
    Network card: 2 x 100GbE RoCE
    System disk: 2 x SSD RAID1
    Data disk: NVMe SSD
    Power supply: Dual power redundancy
    Cooling: Front and rear airflow, supports passive GPU cooling

### 3.2 Confirm GPU Compatibility with Current Server

Not all servers are suitable for high-power GPUs.

Need to confirm:

- Whether the server supports the GPU model;
- Whether the server supports the GPU's length and thickness;
- Whether PCIe slot meets x16;
- Whether PCIe slot is directly connected to CPU;
- Whether riser card is needed;
- Whether riser card supports the target GPU;
- Whether power supply capacity is sufficient;
- Whether power cable specifications are correct;
- Whether airflow is suitable for passive-cooled GPU;
- Whether server BIOS supports the GPU;
- Whether the vendor compatibility list includes the GPU.

Production recommendation:

    GPU servers should not only check "whether it can be plugged in".
    More importantly, check whether it is supported by the server vendor, and whether it meets power supply, cooling, PCIe Lane, and BIOS compatibility requirements.

### 3.3 Confirm GPU Power Supply

Insufficient GPU power supply may lead to:

- System fails to boot;
- GPU not recognized;
- Card drops during stress testing;
- `nvidia-smi` hangs;
- GPU XID errors;
- Server abnormal reboot;
- Power module alarm.

Check focus: /think

- Total power consumption of the power supply module;
- Whether a single power supply can handle full load;
- Whether dual power supplies are redundant;
- Whether the GPU power cable is original factory-made;
- Whether incompatible power cables are mixed;
- Whether PDU and cabinet power supply are sufficient;
- Whether it approaches the power supply limit under full load.

Production Recommendations:

    GPU nodes should not operate near power limits long-term.
    AI training tasks may run at full load for extended periods, insufficient power margin can cause hidden stability issues.

### 3.4 Confirming Cooling Conditions

Many data center GPUs use passive cooling, relying on server airflow.

If passive-cooled GPUs are installed in ordinary tower cases or poorly ventilated enclosures, they may overheat severely.

Check Points:

- Whether the GPU uses active or passive cooling;
- Whether the server supports GPU airflow;
- Whether fans are fully equipped;
- Whether the fan policy is set to performance mode;
- Whether cabinet front/back ventilation is good;
- Whether the data center temperature is stable;
- Whether GPU temperature under full load is controllable.

Check GPU temperature:

    nvidia-smi

Continuous monitoring:

    watch -n 2 nvidia-smi

Check detailed temperature and power consumption:

    nvidia-smi -q

Common temperature issues:

- GPU temperature consistently high;
- GPU automatic throttling;
- Unstable performance;
- Training speed fluctuations;
- Hardware errors in system logs;
- Server fans running at high speed;
- Node restarts after full load.

---

## Four. Preparations Before Entering BIOS/UEFI

### 4.1 Common Ways to Enter BIOS

Different vendors have different entry methods.

Common keys:

    Dell: F2 to enter System Setup, F10 to enter Lifecycle Controller
    HPE: F9 to enter System Utilities
    Lenovo: F1 to enter Setup
    Supermicro: Delete to enter BIOS
    Ordinary motherboard: Delete / F2

For remote data center servers, typically access via BMC:

    Dell iDRAC
    HPE iLO
    Lenovo XClarity Controller
    Supermicro IPMI
    Inspur BMC
    Huawei iBMC

### 4.2 Principles Before Modifying BIOS

Modifying BIOS should not be done arbitrarily, especially in production environments.

Recommended principles:

    1. Record current configuration first
    2. Modify one type of critical configuration at a time
    3. Save screenshots or export configuration after modification
    4. Verify after reboot
    5. Validate lspci, nvidia-smi, PCIe Width, PCIe Speed
    6. Only deploy after passing stress testing
    7. Evict business Pods before modifying production cluster nodes

Kubernetes node maintenance recommendations:

    kubectl cordon <gpu-node-name>
    kubectl drain <gpu-node-name> --ignore-daemonsets --delete-emptydir-data

After maintenance:

    kubectl uncordon <gpu-node-name>

---

## Five. Key BIOS/UEFI Configuration Items

Different server vendors may have different BIOS names, but core configuration items are generally similar.

Common categories:

- PCIe resource configuration;
- CPU virtualization configuration;
- IOMMU configuration;
- Power performance configuration;
- NUMA configuration;
- Secure boot configuration;
- Fan and cooling configuration;
- SR-IOV configuration;
- Resizable BAR configuration.

---

## Six. Above 4G Decoding

### 6.1 What is Above 4G Decoding

Above 4G Decoding, also known as 4G+ address decoding, allows the system to allocate MMIO resources exceeding 4GB for PCIe devices.

GPUs, NVMe, and high-speed network cards may require large address spaces.

In multi-GPU servers, not enabling Above 4G Decoding may cause PCIe address space shortages.

### 6.2 Why GPU Nodes Need to Enable It

GPUs, especially data center GPUs, typically require large BAR address mapping space.

In multi-GPU scenarios, if BIOS cannot allocate sufficient MMIO address space for all GPUs, it may lead to:

- Partial GPUs not recognized;
- System boot stuck;
- Linux only sees partial GPUs via lspci;
- nvidia-smi errors;
- PCIe resource allocation failure;
- Kernel logs showing BAR allocation failed;
- GPU driver loading failure.

Kernel logs may show similar messages:

    BAR 0: no space for
    BAR 1: failed to assign
    pci 0000:xx:xx.x: BAR: no space
    not enough MMIO resources

### 6.3 Recommended Configuration

Recommend:

    Above 4G Decoding: Enabled

For multi-GPU servers, especially 4-card, 8-card, and 10-card GPU nodes, it's typically recommended to enable it.

### 6.4 Verification Methods

After entering the system, check GPU count:

    lspci | grep -i nvidia

Count the number:

    lspci | grep -i nvidia | wc -l

Check NVIDIA management interface:

    nvidia-smi

Check kernel PCIe errors:

    dmesg | grep -i "bar"
    dmesg | grep -i "pci"
    journalctl -k | grep -i "bar"

---

## Seven. Resizable BAR

### 7.1 What is Resizable BAR

Resizable BAR is a PCIe capability allowing the CPU to access a larger GPU memory-mapped region.

In some scenarios, Resizable BAR can improve data access efficiency between CPU and GPU.

However, in server GPUs, AI training, and Kubernetes GPU scenarios, whether to enable it should be determined based on server vendor, GPU model, driver version, and business testing results.

### 7.2 Difference from Above 4G Decoding

Above 4G Decoding solves PCIe device address space allocation issues.

Resizable BAR focuses on BAR mapping window size.

Simplified understanding:

    Above 4G Decoding: Enables the system to allocate address space over 4GB for large PCIe devices
    Resizable BAR: Allows adjustment of BAR mapping window size

They are not the same thing.

### 7.3 Production Recommendations

Recommend:

- Prioritize configurations based on server vendor and NVIDIA official recommendations;
- Do not blindly enable for "potential performance gains";
- Mandatory stress testing after modification;
- Pay attention to driver compatibility;
- Monitor impact on virtualization and passthrough scenarios.

Check BAR information: /think

---
lspci -vvv -s <GPU_PCI_ID> | grep -i BAR

---

## VIII. SR-IOV

### 8.1 What is SR-IOV

SR-IOV, full name Single Root I/O Virtualization.

It allows a physical PCIe device to be split into multiple virtual functions for use by virtual machines or different environments.

In the context of network cards, SR-IOV is very common.

In GPU scenarios, whether SR-IOV is used depends on the GPU model, virtualization scheme, vGPU software stack, and vendor support.

### 8.2 When SR-IOV is Needed in GPU Scenarios

The following scenarios may involve SR-IOV or similar virtualization capabilities:

- GPU passthrough to virtual machines;
- NVIDIA vGPU;
- Cloud platform GPU instances;
- Multi-tenant virtualization environments;
- KubeVirt + GPU passthrough;
- Bare-metal cloud GPU resource pools.

If it's a regular Kubernetes bare-metal GPU node where Pods directly use GPU, SR-IOV is typically not needed.

### 8.3 Recommended Configuration

If GPU virtualization or PCIe device virtualization is needed:

    SR-IOV: Enabled

If it's a regular bare-metal Kubernetes GPU node:

    Follow vendor recommendations
    Do not blindly assume all GPU nodes must have SR-IOV enabled

### 8.4 Related Verification

Check device IOMMU grouping:

    find /sys/kernel/iommu_groups/ -type l

Check PCIe devices:

    lspci | grep -i nvidia

Check kernel parameters:

    cat /proc/cmdline

Common IOMMU parameters for Intel platforms:

    intel_iommu=on

Common IOMMU parameters for AMD platforms:

    amd_iommu=on

---

## IX. VT-d / AMD-Vi / IOMMU

### 9.1 What is IOMMU

IOMMU is used to manage device DMA address mapping, allowing virtual machines to securely access physical devices.

Intel platforms typically call it VT-d.

AMD platforms typically call it AMD-Vi.

### 9.2 Why Pay Attention to IOMMU in GPU Operations

The following scenarios require attention to IOMMU:

- GPU passthrough to virtual machines;
- KubeVirt using GPU;
- Cloud platform virtualized GPU;
- SR-IOV;
- PCIe device isolation;
- Secure DMA mapping;
- VFIO device binding.

If it's regular container use of GPU, IOMMU is typically not needed for container scenarios, but it's common for servers to have it enabled by default.

### 9.3 Recommended Configuration

Virtualization or passthrough scenarios:

    Intel VT-d: Enabled
    AMD-Vi / IOMMU: Enabled

Regular bare-metal Kubernetes GPU scenarios:

    Follow vendor recommendations
    Maintain consistency
    Do not have inconsistent configurations in the same batch of GPU nodes

### 9.4 Linux Verification

Check if IOMMU is enabled:

    dmesg | grep -e DMAR -e IOMMU

Intel:

    dmesg | grep -i dmar

AMD:

    dmesg | grep -i amd-vi

Check IOMMU Group:

    find /sys/kernel/iommu_groups/ -type l

---

## X. PCIe Speed and PCIe Width

### 10.1 PCIe Speed

PCIe Speed represents the PCIe generation speed.

Common ones include:

    Gen3
    Gen4
    Gen5

Different generations have different theoretical bandwidths.

Generally:

    Gen5 > Gen4 > Gen3

If a GPU is plugged into a server that supports Gen4 but runs at Gen3, it may affect the data transfer performance between CPU and GPU.

### 10.2 PCIe Width

PCIe Width represents the number of lanes.

Common ones include:

    x16
    x8
    x4
    x1

GPUs typically prefer to operate at x16.

If a GPU actually operates at x8 or x4, possible reasons include:

- The slot itself only supports x8;
- Riser card limitations;
- Insufficient CPU PCIe lanes;
- Multiple devices sharing the lane;
- Motherboard slot electrical specification limitations;
- Incorrect BIOS PCIe bifurcation configuration;
- Hardware contact issues;
- GPU or slot failure.

### 10.3 BIOS Recommendations

Common configurations:

    PCIe Speed: Auto or fixed to the highest stable generation supported by hardware
    PCIe Slot Configuration: Follow vendor recommendations
    PCIe Link Training: Auto
    PCIe Bifurcation: Configure according to actual riser / GPU topology

Production recommendations:

- Prioritize Auto by default;
- If compatibility issues occur, temporarily fix Gen4 or Gen3 for troubleshooting;
- Do not blindly fix to the highest generation;
- Multi-GPU nodes should refer to server vendor GPU installation guides.

### 10.4 Linux Verification of PCIe Speed / Width

Check GPU PCI ID:

    lspci | grep -i nvidia

Example:

    65:00.0 3D controller: NVIDIA Corporation Device xxxx

Check link status:

    lspci -vvv -s 65:00.0 | grep -i "LnkCap"
    lspci -vvv -s 65:00.0 | grep -i "LnkSta"

Focus on the following in the output:

    LnkCap: Speed 16GT/s, Width x16
    LnkSta: Speed 16GT/s, Width x16

Explanation:

    LnkCap: Hardware theoretical capability
    LnkSta: Current actual working state

If you see:

    LnkCap: Speed 16GT/s, Width x16
    LnkSta: Speed 8GT/s, Width x8

It indicates that the theoretical capability is not fully utilized, and further troubleshooting is needed.

### 10.5 Common Issue Diagnosis

Case 1:

    LnkCap x16
    LnkSta x16

Indicates the link width is normal.

Case 2:

    LnkCap x16
    LnkSta x8

Possible causes:

- Slot or riser issues;
- PCIe lane sharing;
- GPU not seated properly;
- Incorrect BIOS configuration;
- Motherboard design limitations.

Case 3:

    LnkCap Gen4
    LnkSta Gen3

Possible causes:

- BIOS fixed to a lower generation;
- Device negotiation failure;
- Riser not supporting Gen4;
- Signal quality issues;
- Compatibility issues.

## 11. PCIe Bifurcation

### 11.1 What is PCIe Bifurcation

PCIe Bifurcation refers to splitting PCIe lanes.

For example, an x16 lane can be split into:

    x16
    x8/x8
    x4/x4/x4/x4

This is commonly encountered in GPU, NVMe expansion card, and riser card scenarios.

### 11.2 Why GPU Nodes Care About Bifurcation

Multi-GPU servers may connect multiple GPUs via riser cards or PCIe switches.

If the BIOS PCIe Bifurcation configuration is incorrect, it may lead to:

- Some GPUs not being recognized;
- Only part of the devices being recognized;
- Abnormal PCIe device order;
- GPUs operating on non-expected link widths;
- Slow system boot or system hang.

### 11.3 Recommendations

Production recommendations:

- Do not arbitrarily modify Bifurcation if unfamiliar with the motherboard topology;
- Refer to the server vendor's GPU installation documentation;
- Use original factory risers;
- Re-validate lspci and LnkSta after replacing GPUs or risers;
- Record the PCIe positions of each GPU when delivering multi-GPU nodes.

---

## 12. NUMA and CPU Socket Affinity

### 12.1 What is NUMA

NUMA, full name Non-Uniform Memory Access.

In multi-CPU Socket servers, different CPU Sockets have different latency accessing local and remote memory.

GPUs, network cards, and NVMe devices are typically attached to a specific CPU Socket's PCIe Root Complex.

If an application process runs on CPU Socket 0 but the GPU is attached to CPU Socket 1, cross NUMA access may occur.

### 12.2 Why GPU Scenarios Care About NUMA

NUMA has significant impact during multi-GPU training or high-performance inference.

Key considerations:

- Which NUMA node the GPU belongs to;
- Which NUMA node the high-speed network card belongs to;
- Whether CPU processes are bound to appropriate Sockets;
- Whether multiple GPUs span Sockets;
- Whether the GPU to network card spans Sockets;
- Whether distributed training communication takes a long path.

Check NUMA topology:

    lscpu | grep -i numa

Check device NUMA node:

    cat /sys/bus/pci/devices/0000:65:00.0/numa_node

Check GPU topology:

    nvidia-smi topo -m

### 12.3 GPU and Network Card Topology

In AI training scenarios, especially multi-node training, the topology between GPUs and high-speed network cards is critical.

If the GPU and RDMA network card are too far apart, communication efficiency will decrease.

Check PCIe devices:

    lspci | grep -i nvidia
    lspci | grep -i ethernet
    lspci | grep -i mellanox
    lspci | grep -i broadcom
    lspci | grep -i intel

Check topology:

    nvidia-smi topo -m

Common identifiers in output:

    PIX: Through the same PCIe Switch
    PXB: Through multiple PCIe Bridges
    PHB: Through PCIe Host Bridge
    SYS: Across CPU Socket or NUMA
    NV#: Through NVLink connection

Production recommendations:

- Keep GPUs and high-speed network cards as close as possible;
- Distributed training nodes should pay attention to GPU-NIC topology;
- Multi-GPU training should pay attention to GPU-GPU topology;
- Do not only focus on GPU count, but also the topology relationship between GPUs, network cards, and CPUs.

---

## 13. Power Profile and Performance Modes

### 13.1 BIOS Power Policy

Servers typically have power and performance policies in their BIOS.

Common options:

    Performance
    Balanced Performance
    Balanced Power
    Power Saving
    Custom

Different vendors have different naming conventions.

GPU nodes typically do not recommend using overly power-saving policies.

### 13.2 Why Power Policies Affect GPUs

Although GPUs have independent power management, CPU, power policies, PCIe ASPM, C-State, etc., also affect overall performance and latency.

Potential impacts:

- CPU frequency fluctuations;
- Slower data preprocessing;
- Increased PCIe latency;
- Inconsistent inference latency;
- Unstable training throughput;
- Unstable full-load performance of nodes.

### 13.3 Recommended Configuration

For GPU training or inference nodes:

    System Profile: Performance
    Power Regulator: Static High Performance or Maximum Performance
    CPU Power Management: Maximum Performance
    Memory Frequency: Maximum Performance
    PCIe ASPM: Disabled or turn off according to vendor performance recommendations
    C-State: Configure according to business latency requirements

For energy-efficient offline training environments, you can also choose Balanced Performance based on actual testing.

Production recommendations:

    Do not select BIOS policies solely based on "energy saving".
    GPU nodes themselves are costly, and if power policies cause GPUs to wait for CPUs or data input, it will reduce overall resource utilization.

---

## 14. C-State, P-State, and CPU Frequency Policies

### 14.1 C-State

C-State is the CPU's idle power-saving state.

The deeper the C-State, the more power is saved, but wake-up latency may be higher.

For GPU inference services sensitive to latency, deep C-State may cause latency jitter.

### 14.2 P-State

P-State relates to CPU performance states and frequencies.

Frequent CPU frequency changes may affect data preprocessing, inference preprocessing/postprocessing, and network protocol stack processing.

### 14.3 Recommendations

For low-latency inference services:

    Disable deep C-State
    Use Performance mode
    Fix CPU frequency policy to performance

For offline training tasks:

    Can retain some energy-saving strategies
    But need to validate with actual training throughput

Check CPU frequency policy in Linux:

    cpupower frequency-info

Set performance:

    cpupower frequency-set -g performance

Note:

    cpupower is an OS-level configuration, not BIOS configuration.
    However, BIOS power policies affect the frequency management capabilities available to the OS.

---

## 15. PCIe ASPM

### 15.1 What is PCIe ASPM

ASPM, full name Active State Power Management.

It is used to reduce power consumption when the PCIe link is idle.

### 15.2 Why GPU Nodes Care About ASPM

ASPM may introduce latency or compatibility issues in certain scenarios.

For high-performance GPU nodes, especially in training, inference, high-speed network cards, and low-latency scenarios, stability and performance are typically more prioritized.

### 15.3 Recommendations

Common recommendations:

    PCIe ASPM: Disabled

Or configure according to the server vendor's performance optimization recommendations.

Linux check related logs:

    dmesg | grep -i aspm

Sometimes you may also see:

    pcie_aspm=off

Production recommendations:

    It is not recommended to sacrifice GPU node stability for power savings.
    Whether to disable ASPM should be combined with vendor recommendations and stress test results.

---

## SixteenI don't know.Secure Boot

### 16.1 Secure Boot and NVIDIA Drivers

Secure Boot is not a GPU performance tuning item, but it often affects NVIDIA driver installation.

If Secure Boot is enabled and the NVIDIA kernel module is not properly signed, it may lead to the driver module failing to load.

Manifestations:

- lspci can see the GPU;
- nvidia-smi fails;
- lsmod does not show the nvidia module;
- dmesg shows module signature-related errors;
- The driver installation appears to be completed, but is actually unavailable.

### 16.2 Troubleshooting Commands

Check Secure Boot status:

    mokutil --sb-state

Check NVIDIA modules:

    lsmod | grep nvidia

Check kernel logs:

    dmesg | grep -i nvidia
    dmesg | grep -i secure
    dmesg | grep -i signature

### 16.3 Handling Recommendations

Optional solutions:

    1. Disable Secure Boot
    2. Sign the NVIDIA kernel module
    3. Use the official signed driver package from the distribution

Secure boot can typically be disabled in experimental environments.

In production environments, it should be decided based on enterprise security policies, and it is not recommended to disable it directly.

---

## SeventeenI don't know.Fan Strategy and Thermal Control Mode

### 17.1 Fan Strategy

The server BIOS or BMC usually allows configuring fan strategies.

Common modes:

    Standard
    Optimal
    Performance
    Full Speed
    Acoustic
    Power Saving

For GPU nodes, it is recommended to prioritize performance or GPU-optimized modes.

### 17.2 Why Fan Strategy Matters

GPU generates significant heat when fully loaded.

If the fan strategy is biased towards energy saving, it may lead to:

- Increased GPU temperature;
- GPU underclocking;
- Reduced training performance;
- Sudden fan speed increase;
- Unstable long-term operation;
- Reduced GPU lifespan.

### 17.3 Verification Methods

Check GPU temperature:

    nvidia-smi

Continuous observation:

    watch -n 2 nvidia-smi

Check BMC sensors:

    ipmitool sensor

Check system event logs:

    ipmitool sel list

If using a vendor BMC, you can also view it through the web console:

    Fan speed
    Inlet temperature
    Outlet temperature
    Power status
    GPU temperature
    PCIe device alert

---

## EighteenI don't know.GPU Topology Check

### 18.1 View GPU Topology

Command:

    nvidia-smi topo -m

Example output may include:

    GPU0    GPU1    GPU2    GPU3    CPU Affinity    NUMA Affinity
    GPU0     X      NV2     SYS     SYS     0-31            0
    GPU1    NV2      X      SYS     SYS     0-31            0
    GPU2    SYS     SYS      X      NV2     32-63           1
    GPU3    SYS     SYS     NV2      X      32-63           1

### 18.2 Understanding the Topology

Common identifiers:

    X      Self
    PIX    Same PCIe Switch
    PXB    Multi-level PCIe Bridge
    PHB    PCIe Host Bridge
    SYS    Cross CPU Socket / NUMA
    NV#    NVLink connection

Generally:

    NVLink is preferred over PCIe
    Same Socket is preferred over cross Socket
    PIX is typically preferred over SYS
    The closer the GPU is to the network card, the more beneficial distributed training communication is

### 18.3 Production Recommendations

When deploying multi-GPU nodes, it is recommended to record:

- PCI address of each GPU;
- NUMA node of each GPU;
- GPU-GPU topology;
- GPU-NIC topology;
- CPU Socket where the GPU resides;
- Whether NVLink exists;
- Whether cross-socket communication exists.

Recommend saving the output:

    nvidia-smi topo -m > gpu-topology.txt
    lspci -tv > pci-tree.txt
    lscpu > cpu-topology.txt

---

## NineteenI don't know.BIOS Configuration Recommendations Summary

The following are common BIOS/UEFI configuration recommendations for GPU nodes. In actual production, it should be based on the server vendor, GPU vendor, and business stress test results.

### 19.1 Basic GPU Nodes

Suitable for:

- Kubernetes GPU Worker;
- AI inference;
- Small-scale training;
- Bare-metal GPU nodes.

Recommendations:

    Above 4G Decoding: Enabled
    PCIe Speed: Auto or highest stable generation
    PCIe ASPM: Disabled
    Power Profile: Performance
    Fan Profile: Performance / High Performance
    Secure Boot: Decided based on driver installation strategy
    SR-IOV: As needed
    VT-d / IOMMU: As needed
    NUMA: Keep default or follow vendor recommendations
    Hyper-Threading: Decided based on business stress tests

### 19.2 GPU Virtualization Nodes

Suitable for:

- vGPU;
- GPU passthrough;
- KubeVirt;
- Cloud platform GPU instances.

Recommendations:

Above 4G Decoding: Enabled  
SR-IOV: Enabled  
VT-d / AMD-Vi: Enabled  
IOMMU: Enabled  
PCIe ACS: Configure according to virtualization requirements  
Secure Boot: Combined with driver signing strategy  
Power Profile: Performance  
Fan Profile: Performance  

### 19.3 Training Multi-GPU Node  

Suitable for:  
- Multi-card training;  
- Large model training;  
- Distributed training;  
- High-speed network card communication.  

Recommendations:  
    Above 4G Decoding: Enabled  
    PCIe Speed: Auto / Gen4 / Gen5, decide based on stability testing  
    PCIe ASPM: Disabled  
    Power Profile: Maximum Performance  
    Fan Profile: High Performance  
    C-State: Decide based on latency and throughput testing  
    NUMA: Focus on checking GPU and NIC affinity  
    SR-IOV: Can be disabled if there is no virtualization requirement  
    VT-d / IOMMU: Configure according to platform specifications  

---

## Twenty, OS Layer Verification Commands  

After completing BIOS configuration and rebooting, enter the operating system for verification.  

### 20.1 Check if GPU is recognized by PCIe  

    lspci | grep -i nvidia  

If there is no output, it indicates the system has not enumerated the NVIDIA PCIe device.  

Prioritize checking:  
- Whether the GPU is properly inserted;  
- Whether the BIOS recognizes it;  
- Whether Above 4G Decoding is enabled;  
- Whether the PCIe slot is correct;  
- Whether the riser card matches;  
- Whether the power supply is normal;  
- Whether the GPU is faulty.  

### 20.2 Check PCIe Tree  

    lspci -tv  

Save topology:  
    lspci -tv > pci-tree.txt  

### 20.3 Check PCIe Speed and Width  

First check PCI ID:  
    lspci | grep -i nvidia  

Then view detailed status:  
    lspci -vvv -s <GPU_PCI_ID> | grep -i "LnkCap"  
    lspci -vvv -s <GPU_PCI_ID> | grep -i "LnkSta"  

Example:  
    lspci -vvv -s 65:00.0 | grep -i "Lnk"  

### 20.4 Check NUMA Information  

    lscpu | grep -i numa  

Check device NUMA:  
    cat /sys/bus/pci/devices/0000:<PCI_ID>/numa_node  

Note PCI ID format:  
    lspci output: 65:00.0  
    sysfs path: 0000:65:00.0  

### 20.5 Check Kernel PCIe Logs  

    dmesg | grep -i pci  
    dmesg | grep -i nvidia  
    dmesg | grep -i "bar"  
    dmesg | grep -i "aer"  
    journalctl -k | grep -i pci  

### 20.6 Check BIOS and Hardware Information  

    dmidecode -t bios  
    dmidecode -t system  
    dmidecode -t baseboard  
    dmidecode -t processor  
    dmidecode -t memory  

### 20.7 Check Sensors and Hardware Alarms  

If ipmitool is installed:  
    ipmitool sensor  
    ipmitool sel list  

---

## Twenty-one, nvidia-smi Layer Verification  

After passing BIOS and PCIe verification, check the NVIDIA driver layer.  

### 21.1 Check GPU Basic Status  

    nvidia-smi  

Focus on:  
- Number of GPUs;  
- GPU model;  
- Driver Version;  
- CUDA Version;  
- Temperature;  
- Power Usage;  
- Memory Usage;  
- GPU Utilization;  
- Processes.  

### 21.2 Check Detailed Information  

    nvidia-smi -q  

Focus on:  
- ECC Mode;  
- Retired Pages;  
- Power Readings;  
- Clocks;  
- Performance State;  
- PCIe Generation;  
- PCIe Link Width;  
- Accounting Mode;  
- Compute Mode.  

### 21.3 Check PCIe Information  

    nvidia-smi -q | grep -i "PCI" -A 20  

### 21.4 Check Topology  

    nvidia-smi topo -m  

### 21.5 Check XID Errors  

    dmesg | grep -i xid  
    journalctl -k | grep -i xid  

If XID errors appear, analyze them together with NVIDIA XID documentation, business logs, temperature, power, and PCIe AER errors.  

---

## Twenty-two, Common Fault: lspci Doesn't Show GPU  

### 22.1 Phenomenon  

Execute:  
    lspci | grep -i nvidia  

No output.  

### 22.2 Possible Causes  

Possible causes:  
- GPU not properly inserted;  
- Power cable not connected;  
- Insufficient power supply;  
- Damaged PCIe slot;  
- Incompatible riser card;  
- BIOS not recognizing the device;  
- Above 4G Decoding not enabled;  
- PCIe Bifurcation configuration error;  
- Outdated motherboard BIOS version;  
- GPU hardware failure.  

### 22.3 Troubleshooting Steps  

Step 1: Check if BMC/BIOS can see PCIe devices.  

Step 2: Check physical installation:  
    Is the GPU securely inserted?  
    Is the power cable connected?  
    Is the riser card correct?  
    Does the slot meet requirements?  

Step 3: Check BIOS:  
    Is Above 4G Decoding enabled?  
    Is PCIe Slot enabled?  
    Is PCIe Bifurcation correctly configured?  
    Does the BIOS need an update?  

Step 4: Check Linux logs:  
    dmesg | grep -i pci  
    dmesg | grep -i "bar"  
    journalctl -k | grep -i pci  

### 22.4 Handling Recommendations  

Handling method: /think

- Replug the GPU;
- Replace the PCIe slot;
- Replace the riser;
- Check the power cables;
- Enable Above 4G Decoding;
- Restore PCIe Bifurcation to default;
- Upgrade the BIOS;
- Contact the server vendor to confirm compatibility.

---

## 23. Common Fault Two: lspci can see GPU, but nvidia-smi is abnormal

### 23.1 Phenomenon

Execute:

    lspci | grep -i nvidia

You can see the GPU.

But executing:

    nvidia-smi

Returns an error.

Common errors:

    NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.

### 23.2 Possible Causes

Possible causes:

- NVIDIA driver not installed;
- Failed driver installation;
- Kernel module not loaded;
- Secure Boot blocks module loading;
- DKMS compilation failed;
- Current kernel and driver are incompatible;
- GPU model and driver version mismatch;
- nouveau driver conflict;
- System kernel upgrade without driver module rebuild.

### 23.3 Troubleshooting Commands

Check driver modules:

    lsmod | grep nvidia

Check Secure Boot:

    mokutil --sb-state

Check kernel logs:

    dmesg | grep -i nvidia
    dmesg | grep -i nouveau
    dmesg | grep -i signature

Check installed packages:

    dpkg -l | grep -i nvidia

or:

    rpm -qa | grep -i nvidia

### 23.4 Handling Recommendations

Handling methods:

- Install the correct version of NVIDIA driver;
- Disable nouveau;
- Disable Secure Boot or sign kernel modules;
- Install kernel headers;
- Rebuild DKMS;
- Reboot the node;
- Confirm driver version supports current GPU.

---

## 24. Common Fault Three: GPU only recognizes partial cards

### 24.1 Phenomenon

The server has 4 GPUs installed, but the system only sees 2.

Execute:

    lspci | grep -i nvidia
    nvidia-smi

Displays incomplete counts.

### 24.2 Possible Causes

Possible causes:

- Above 4G Decoding not enabled;
- PCIe address space insufficient;
- riser configuration error;
- PCIe Bifurcation configuration error;
- A slot disabled;
- GPU power supply insufficient;
- Single GPU failure;
- BIOS version does not support current configuration;
- CPU PCIe Lane insufficient;
- Slot and device combination not supported by vendor.

### 24.3 Troubleshooting Steps

Check BIOS:

    Above 4G Decoding
    PCIe Slot Enable
    PCIe Bifurcation
    PCIe Speed
    System Event Log

Check system:

    lspci -tv
    dmesg | grep -i pci
    dmesg | grep -i bar
    ipmitool sel list

Cross-testing:

    Swap GPU slots
    Swap riser
    Single-card boot test
    Incrementally add GPUs

### 24.4 Handling Recommendations

Handling methods:

- Enable Above 4G Decoding;
- Use vendor-recommended slot combinations;
- Restore Bifurcation recommended configuration;
- Upgrade BIOS;
- Check riser;
- Check GPU power supply;
- Contact vendor to confirm maximum GPU support count.

---

## 25. Common Fault Four: PCIe downspeed or downwidth

### 25.1 Phenomenon

Expected:

    PCIe Gen4 x16

Actual:

    PCIe Gen3 x8

Check:

    lspci -vvv -s <GPU_PCI_ID> | grep -i "LnkCap"
    lspci -vvv -s <GPU_PCI_ID> | grep -i "LnkSta"

### 25.2 Possible Causes

Possible causes:

- Slot electrical specifications only x8;
- riser only supports x8;
- Motherboard channel sharing;
- BIOS fixed low speed;
- PCIe signal quality issues;
- GPU not securely inserted;
- Server vendor design limitations;
- CPU PCIe Lane insufficient;
- Gen4 compatibility issue automatically downgraded to Gen3.

### 25.3 Handling Recommendations

Handling methods:

- Confirm server hardware manual;
- Switch to x16 slot;
- Check riser specifications;
- Set PCIe Speed to Auto or target Gen in BIOS;
- Update BIOS / Firmware;
- Replug GPU;
- Compare stress test results;
- If Gen4 instability is fixed, temporarily fix Gen3 for stability verification.

---

## 26. Common Fault Five: GPU drops during stress testing

### 26.1 Phenomenon

Idle nvidia-smi is normal, but after running training or stress testing, the following occurs:

- GPU disappears;
- nvidia-smi hangs;
- XID error;
- Node reboot;
- Training task failure;
- Pod abnormal exit;
- GPU temperature too high;
- PCIe AER error appears in system logs.

### 26.2 Possible Causes

Possible causes:

- Insufficient power;
- Abnormal GPU power cable;
- Insufficient cooling;
- PCIe slot contact issues;
- riser issues;
- GPU hardware failure;
- Driver issues;
- BIOS power policy issues;
- PCIe ASPM compatibility issues;
- Overheated data center.

### 26.3 Troubleshooting Commands

Check XID:

    dmesg | grep -i xid
    journalctl -k | grep -i xid

Check PCIe AER:

    dmesg | grep -i aer
    journalctl -k | grep -i aer

Check temperature and power consumption:

    nvidia-smi
    nvidia-smi -q

Check BMC events:

    ipmitool sel list
    ipmitool sensor

Continuous observation:

    watch -n 2 nvidia-smi

### 26.4 Handling Recommendations

Handling methods:

- Check data center temperature;
- Adjust fans to performance mode;
- Check GPU power cables;
- Check power supply modules;
- Replace riser;
- Replace PCIe slot;
- Fix PCIe Gen downgrade verification;
- Update BIOS and firmware;
- Update or roll back NVIDIA driver;
- Single-card stress test to locate faulty card.

---

## 27. GPU Node Onboarding Acceptance Process

GPU nodes should undergo the following acceptance checks before joining a Kubernetes cluster.

### 27.1 Hardware Identification Acceptance

Check:

    lspci | grep -i nvidia
    lspci -tv
    dmidecode -t bios
    dmidecode -t system

Acceptance Criteria:

    [ ] GPU count matches expectations
    [ ] All GPU PCIe devices are visible
    [ ] No obvious PCIe BAR allocation failures
    [ ] No obvious AER errors
    [ ] BIOS version has been recorded
    [ ] Server model has been recorded

### 27.2 PCIe Link Acceptance

Check:

    lspci -vvv -s <GPU_PCI_ID> | grep -i "LnkCap"
    lspci -vvv -s <GPU_PCI_ID> | grep -i "LnkSta"

Acceptance Criteria:

    [ ] PCIe Speed matches expectations
    [ ] PCIe Width matches expectations
    [ ] No abnormal downspeed
    [ ] No abnormal downwidth
    [ ] Multi-GPU topology has been saved

### 27.3 NUMA Topology Acceptance

Check:

    lscpu | grep -i numa
    nvidia-smi topo -m
    lspci -tv

Acceptance Criteria:

    [ ] GPU NUMA distribution has been recorded
    [ ] GPU and NIC topology has been recorded
    [ ] Multi-GPU training nodes have confirmed communication topology
    [ ] Cross-socket risks have been evaluated

### 27.4 Pre-Installation Acceptance

Confirm before installing drivers:

    [ ] lspci can see all GPUs
    [ ] BIOS configuration is complete
    [ ] Above 4G Decoding has been enabled
    [ ] PCIe link is normal
    [ ] Secure Boot policy has been confirmed
    [ ] Fan policy has been confirmed
    [ ] Power policy has been confirmed

### 27.5 Post-Installation Acceptance

Check after driver installation:

    nvidia-smi
    nvidia-smi -q
    nvidia-smi topo -m
    dmesg | grep -i xid

Acceptance Criteria:

    [ ] nvidia-smi is normal
    [ ] GPU count is correct
    [ ] Temperature is normal
    [ ] Power consumption is normal
    [ ] No XID errors
    [ ] ECC status matches expectations
    [ ] Topology matches expectations

### 27.6 Stress Test Acceptance

Recommended tests:

- Single-GPU stress test
- Multi-GPU stress test
- Memory stress test
- Long-term stability test
- Temperature observation
- Power consumption observation
- XID error observation

Check:

    watch -n 2 nvidia-smi
    dmesg -w
    journalctl -k -f
    ipmitool sensor

Acceptance Criteria:

    [ ] Full-load temperature is stable
    [ ] No GPU card drop
    [ ] No severe XID errors
    [ ] No node reboots
    [ ] No massive PCIe AER errors
    [ ] Performance meets expectations
    [ ] Fans and power have no abnormal alerts

---

## Twenty-Eight, GPU Node Hardware Baseline Record Template

It is recommended to establish a baseline record for each GPU node.

Template:

    Node name:
    Asset number:
    Data center location:
    Rack location:
    Server vendor:
    Server model:
    BIOS version:
    BMC version:
    CPU model:
    CPU Socket count:
    Memory capacity:
    GPU model:
    GPU count:
    GPU memory:
    GPU PCI address:
    GPU NUMA node:
    GPU topology:
    NIC model:
    NIC PCI address:
    NIC NUMA node:
    System disk:
    Data disk:
    Power supply power:
    Fan policy:
    Power Profile:
    Above 4G Decoding:
    SR-IOV:
    VT-d / IOMMU:
    Secure Boot:
    PCIe Speed:
    PCIe Width:
    NVIDIA Driver:
    CUDA Version:
    Container Runtime:
    Kubernetes Version:
    Device Plugin / GPU Operator Version:
    Verifier:
    Verification date:

Recommended command outputs to save:

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

---

## Twenty-Nine, Kubernetes GPU Node Pre-Check

Although this document does not elaborate on Kubernetes GPU scheduling, the following items should be confirmed before adding GPU nodes to the cluster.

### 29.1 Node Naming Recommendation

Recommended GPU node names should be identifiable:

    k8s-gpu-node01
    k8s-gpu-node02
    ai-gpu-a100-01
    ai-gpu-l4-01

Avoid using completely meaningless names.

### 29.2 Node Label Planning

Recommended labels to set after joining the cluster:

    kubectl label node <node-name> node-role.kubernetes.io/gpu=true
    kubectl label node <node-name> accelerator=nvidia
    kubectl label node <node-name> gpu.vendor=nvidia
    kubectl label node <node-name> gpu.model=a100

If there are different GPU models, you can furtherBreakdown:

    gpu.memory=80gb
    gpu.partition=mig-enabled
    gpu.workload=inference
    gpu.workload=training

### 29.3 Node Taint Planning

It is recommended to add taints to GPU nodes to prevent regular Pods from scheduling:

    kubectl taint node <node-name> nvidia.com/gpu=true:NoSchedule

GPU Pods need to explicitly add toleration.

### 29.4 Resource Planning

Before joining Kubernetes, ensure:

    [ ] CPU resources are sufficient to support GPU task data preprocessing
    [ ] Memory is sufficient
    [ ] Local disk is sufficient for caching models and data
    [ ] Network meets requirements for image pulling and model downloads
    [ ] GPU monitoring port planning
    [ ] Node maintenance process is clearly defined

---

## Thirty, Production Environment Notes

### 30.1 Do Not Use GPU Nodes as Regular Worker Nodes

GPU nodes are costly and should not be used for arbitrary regular workloads.

Regular Pods consume CPU, memory, disk, and network resources, which may affect GPU tasks.

Recommendations include:

- Node Label;
- Taint / Toleration;
- ResourceQuota;
- PriorityClass;
- Namespace isolation;
- Node pool planning;

to control GPU node usage.

### 30.2 Do Not Rely Solely on GPU Count

GPU server capabilities are not solely determined by GPU count.

Also consider:

- GPU model;
- VRAM capacity;
- PCIe generation;
- PCIe width;
- NVLink;
- CPU Socket;
- NUMA;
- Network topology;
- Local disk;
- Cooling;
- Power supply;
- Driver version;
- CUDA compatibility.

### 30.3 Do Not Ignore BIOS Version

BIOS version may affect:

- GPU compatibility;
- PCIe device recognition;
- Power policy;
- Fan policy;
- NUMA behavior;
- SR-IOV;
- IOMMU;
- Device boot order.

Production recommendations:

    Before batch deploying GPU nodes, unify BIOS and BMC versions.
    It is not recommended to have inconsistent BIOS versions among the same batch of GPU nodes.

### 30.4 Do Not Skip Stress Testing Before Deployment

GPU nodes being idle normally does not guarantee stability under full load.

Must at least verify:

- Single GPU full load;
- Multi-GPU full load;
- Temperature;
- Power consumption;
- XID;
- PCIe AER;
- Node stability;
- Business container runtime stability.

---

## Thirty-One, GPU BIOS and Hardware Optimization Summary

GPU node stability does not start from Kubernetes, but from hardware and BIOS.

The correct GPU node deployment sequence should be:

    1. Confirm server and GPU compatibility
    2. Check power supply and cooling
    3. Configure BIOS / UEFI
    4. Enable Above 4G Decoding
    5. Configure SR-IOV, VT-d, IOMMU as needed
    6. Check PCIe Speed / Width
    7. Check NUMA and GPU topology
    8. Confirm Secure Boot policy
    9. Install operating system
    10. Validate lspci
    11. Install NVIDIA driver
    12. Validate nvidia-smi
    13. Perform stress testing
    14. Integrate with Kubernetes
    15. Deploy Device Plugin / GPU Operator
    16. Integrate with monitoring and alerts

When troubleshooting, also follow the layered approach:

    lspci does not show GPU:
        Prioritize checking hardware, BIOS, PCIe, power supply, riser, Above 4G Decoding

    lspci can show GPU but nvidia-smi is abnormal:
        Prioritize checking driver, Secure Boot, kernel modules, DKMS, nouveau

    nvidia-smi is normal but performance is poor:
        Prioritize checking PCIe Width, PCIe Speed, NUMA, temperature, power consumption, application data input

    Stress testing causes GPU drop:
        Prioritize checking power supply, cooling, PCIe AER, XID, riser, hardware stability

    Kubernetes cannot schedule GPU:
        Check Device Plugin, kubelet, nvidia.com/gpu, taint, label, runtime later

GPU operations are not just about installing drivers, nor just deploying Device Plugin.

True GPU operations capability requires establishing a complete chain from hardware layer, BIOS layer, system layer, driver layer, container layer, Kubernetes layer, and monitoring layer.

---

## Reference Documents

- NVIDIA Documentation:https://docs.nvidia.com/
- NVIDIA Data Center GPU Documentation:https://docs.nvidia.com/datacenter/
- NVIDIA CUDA Documentation:https://docs.nvidia.com/cuda/
- NVIDIA GPU Operator Documentation:https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/
- Kubernetes GPU Scheduling:https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/
- NVIDIA Kubernetes Device Plugin:https://github.com/NVIDIA/k8s-device-plugin
- NVIDIA DCGM Exporter:https://github.com/NVIDIA/dcgm-exporter
- Linux PCI Utilities:https://mj.ucw.cz/sw/pciutils/
- ipmitool Documentation:https://github.com/ipmitool/ipmitool