# 03-NVIDIA Driver Installation and Verification

## Document Overview

This document is used to organize the installation, verification, version selection, kernel module checks, common troubleshooting, and Kubernetes prerequisite checks for NVIDIA drivers on Linux GPU nodes.

This document focuses on the "driver layer" in the GPU operations chain.

The path from GPU hardware to Kubernetes availability roughly goes through the following layers:

    GPU Hardware
      ↓
    BIOS / UEFI
      ↓
    PCIe Device Recognition
      ↓
    Linux Kernel
      ↓
    NVIDIA Driver
      ↓
    nvidia-smi
      ↓
    CUDA Runtime
      ↓
    NVIDIA Container Toolkit
      ↓
    Container Runtime
      ↓
    NVIDIA Device Plugin / GPU Operator
      ↓
    Kubernetes Scheduling nvidia.com/gpu
      ↓
    GPU Pod Using GPU

This document primarily covers:

    Linux Kernel
      ↓
    NVIDIA Driver
      ↓
    nvidia-smi
      ↓
    CUDA / Container / Kubernetes Prerequisites

This document does not focus on CUDA Toolkit installation or Device Plugin/GPU Operator deployment. CUDA installation is covered in subsequent notes, and Kubernetes GPU scheduling is covered in the K8S GPU section.

---

## Tags

#GPU #NVIDIA #Driver #Linux #Ubuntu #RockyLinux #CUDA #Kubernetes #Containerd #SRE #TransportBarriers

---

## Recommended Path

Recommended path:

    06-GPUandAIInfrastructure/01-GPUBasis/03-NVIDIA-Driver installation and authentication.md

---

## One: Why NVIDIA Driver is the Critical Layer for GPU Nodes

A GPU cannot be directly used by business applications just by being plugged into a server.

The Linux system can identify NVIDIA devices via PCIe, but this only indicates that the system has discovered hardware and does not mean the GPU is ready for use by CUDA, containers, or Kubernetes.

A GPU node must at least meet three layers:

    First Layer: Hardware Visibility
        lspci can see NVIDIA devices

    Second Layer: Driver Availability
        nvidia-smi can normally display GPUs

    Third Layer: Runtime Availability
        CUDA, containers, Kubernetes Pods can access GPUs

The NVIDIA driver resides in the second layer and is the core linking upper and lower layers.

If the driver layer is abnormal, common manifestations include:

- lspci can see GPU, but nvidia-smi reports errors;
- nvidia-smi shows "No devices were found";
- NVIDIA-SMI failed because it couldn't communicate with the NVIDIA driver;
- lsmod doesn't show nvidia kernel modules;
- Secure Boot prevents driver module loading;
- Driver fails after kernel upgrade;
- Kubernetes nodes cannot register nvidia.com/gpu;
- Device Plugin runs abnormally;
- GPU Pod scheduling fails;
- Cannot execute nvidia-smi inside containers;
- CUDA programs report driver version incompatibility.

Therefore, GPU operations cannot only execute:

    nvidia-smi

But also understand:

    lspci seeing GPU indicates PCIe layer visibility;
    nvidia-smi working normally indicates driver layer availability;
    nvcc -V working normally indicates CUDA Toolkit existence;
    nvidia-smi working normally inside containers indicates container runtime GPU integration;
    kubectl describe node showing nvidia.com/gpu indicates Kubernetes resource registration.

---

## Two: The Role of NVIDIA Driver in the System

The NVIDIA driver enables the operating system and user-space programs to access the GPU.

It generally includes:

- Kernel modules;
- User-space libraries;
- Management tools;
- Device files;
- CUDA Runtime support;
- GPU management interface;
- Memory management;
- Temperature, power, clock, ECC, and other management capabilities.

Common related components include:

    nvidia
    nvidia_uvm
    nvidia_drm
    nvidia_modeset
    nvidia-smi
    libcuda.so
    /dev/nvidia0
    /dev/nvidiactl
    /dev/nvidia-uvm

### 2.1 Kernel Modules

Check NVIDIA kernel modules:

    lsmod | grep nvidia

Common outputs include:

    nvidia
    nvidia_uvm
    nvidia_drm
    nvidia_modeset

If there is no output, it indicates the NVIDIA driver modules are not loaded.

### 2.2 Device Files

Check NVIDIA device files:

    ls -l /dev/nvidia*

Common outputs include:

    /dev/nvidia0
    /dev/nvidiactl
    /dev/nvidia-uvm
    /dev/nvidia-uvm-tools

If device files do not exist, containers and CUDA programs typically cannot access the GPU.

### 2.3 nvidia-smi

nvidia-smi is the most commonly used NVIDIA GPU management tool.

It can view:

- GPU model;
- Driver version;
- CUDA version;
- GPU temperature;
- GPU power consumption;
- Memory usage;
- GPU utilization;
- Processes using GPU;
- MIG status;
- ECC status;
- GPU topology;
- XID error troubleshooting.

Basic command:

    nvidia-smi

Continuous observation:

    watch -n 2 nvidia-smi

Detailed information:

    nvidia-smi -q

Topology information:

    nvidia-smi topo -m

---

## Three: Relationship Between Driver, CUDA Toolkit, and CUDA Runtime

Many GPU beginners often confuse NVIDIA Driver, CUDA Toolkit, and CUDA Runtime.

These three concepts must be distinguished.

### 3.1 NVIDIA Driver /think

NVIDIA Driver is the host machine driver.

It is responsible for enabling Linux kernel and user-space programs to access GPU.

Verification command:

    nvidia-smi

If nvidia-smi is not functioning properly, it indicates a problem with the driver layer.

### 3.2 CUDA Toolkit

CUDA Toolkit is a development toolkit that includes:

- nvcc compiler;
- CUDA header files;
- CUDA examples;
- some development libraries;
- debugging and analysis tools.

Verification command:

    nvcc -V

Notes:

    nvidia-smi being normal does not guarantee that nvcc exists.
    nvcc -V not existing does not mean the GPU driver is unavailable.

In production inference or training containers, it is often unnecessary to install the full CUDA Toolkit on the host machine. Installing only the NVIDIA Driver on the host machine and having the CUDA Runtime included in the container image is sufficient.

### 3.3 CUDA Runtime

CUDA Runtime is the runtime library required for CUDA applications to run.

In containerized scenarios, CUDA Runtime typically comes from the container image, for example:

    nvidia/cuda:12.2.0-runtime-ubuntu22.04
    nvidia/cuda:12.2.0-base-ubuntu22.04
    pytorch/pytorch:<version>-cuda<version>-cudnn<version>-runtime

### 3.4 Relationship Between the Three

It can be simply understood as:

    NVIDIA Driver:
        Must be present on the host machine, responsible for truly driving the GPU.

    CUDA Toolkit:
        Required for developing and compiling CUDA programs, not all production nodes must install it.

    CUDA Runtime:
        Required for running CUDA applications, typically provided by the container image.

In Kubernetes GPU nodes, common practices are:

    Host machine:
        NVIDIA Driver
        NVIDIA Container Toolkit
        containerd / Docker
        kubelet
        Device Plugin / GPU Operator

    Container image:
        CUDA Runtime
        cuDNN
        PyTorch / TensorFlow
        Business application

---

## Four, Driver Version Selection Principles

NVIDIA driver version selection cannot be arbitrary.

When selecting a driver, the following must be considered:

- GPU model;
- Operating system version;
- Kernel version;
- CUDA version;
- AI framework version;
- Container image CUDA Runtime version;
- Kubernetes node runtime;
- NVIDIA Device Plugin / GPU Operator version;
- Whether MIG is needed;
- Whether vGPU is needed;
- Whether long-term stable support is needed;
- Whether it meets enterprise security and compliance requirements.

### 4.1 Not Recommended to Blindly Install Latest

Production environments do not recommend directly installing the latest driver.

Reasons:

- New versions may introduce compatibility issues;
- AI framework images may not yet be compatible;
- CUDA Runtime may not match;
- GPU Operator versions may have constraints;
- Kernel modules may have issues with the current Kernel;
- Inconsistent versions across batch nodes increase troubleshooting difficulty.

Production recommendations:

    Select a stable driver version that has been verified.
    Validate on test nodes before batch deployment.
    Keep driver versions consistent across the same batch of GPU nodes.

### 4.2 Data Center GPUs Should Prefer Data Center Driver

For data center GPUs, such as:

- T4
- L4
- A10
- A30
- A40
- A100
- H100
- H200
- H20
- A800

It is recommended to prioritize the NVIDIA Data Center Driver branch.

Reasons:

- More suitable for servers and data center environments;
- More focused on stability;
- More suitable for AI training and inference;
- More suitable for Kubernetes GPU nodes;
- More suitable for integration with DCGM, GPU Operator, and other components.

### 4.3 Driver and CUDA Compatibility

Whether a CUDA application can run depends on the compatibility between the required CUDA Runtime and the NVIDIA Driver on the host machine.

General principle:

    The host machine driver version must support the CUDA Runtime required by the container or application.

For example:

    The container image uses CUDA 12.x Runtime
    The host machine driver must meet the minimum driver version requirement for CUDA 12.x

If the driver is too old, the following issues may occur:

    CUDA driver version is insufficient for CUDA runtime version

Or:

    no kernel image is available for execution on the device

Or the application framework detects no GPU.

### 4.4 Driver Version Selection Recommendations

It is recommended to determine the version in the following order:

    1. Confirm GPU model
    2. Confirm operating system version
    3. Confirm the CUDA version used by the business image
    4. Check NVIDIA official CUDA Compatibility
    5. Check NVIDIA Data Center Driver support
    6. Install and validate on test nodes
    7. Run nvidia-smi, CUDA examples, and container GPU tests
    8. Run business image or training/inference framework tests
    9. Solidify the version and promote it in batches

---

## Five, Installation Method Selection

Common NVIDIA driver installation methods include three categories:

    1. Installation via OS package manager
    2. Installation via NVIDIA CUDA Repository
    3. Installation via NVIDIA .run file

Production environments are recommended to use the package manager method.

### 5.1 Package Manager Installation

Advantages:

- Better maintainability;
- Easier upgrades and uninstallation;
- Consistent with system package management;
- More suitable for batch deployment;
- More suitable for automated operations;
- DKMS and Kernel Header dependencies are easier to manage.

Disadvantages:

- Version selection requires understanding the repository source;
- Different distributions have different package names;
- In enterprise internal network environments, software source or mirror source configuration is needed.

### 5.2 NVIDIA CUDA Repository Installation

Advantages:

- From NVIDIA official repository;
- More comprehensive versions;
- Suitable for data center GPUs;
- Can be installed with cuda-drivers or nvidia-driver packages.

Disadvantages:

- Domestic network environments may have unstable access;
- Requires configuring GPG key and repo;
- Version management needs caution;
- In batch environments, it is recommended to set up internal mirror sources.

### 5.3 .run File Installation /think

.run files are installation packages downloaded from the NVIDIA official website.

Advantages:

- Direct;
- Suitable for offline or special environments;
- Not dependent on system repositories.

Disadvantages:

- Not conducive to package management;
- More likely to become invalid after kernel upgrades;
- Uninstallation and upgrades are less clear compared to package management methods;
- High cost for automated batch maintenance;
- Prone to conflicts with system packages;
- Not recommended for long-term maintenance in production environments.

Production Recommendations:

    .run files can be used for test or temporary environments.
    Production environments, Kubernetes GPU nodes, and batch GPU nodes are recommended to prioritize using package managers or NVIDIA official repositories.

---

## Six. Pre-Installation Checks

Before installing NVIDIA drivers, do not rush to execute the installation command.

First, confirm the hardware, system, and kernel status.

### 6.1 Check System Version

Ubuntu:

    cat /etc/os-release
    lsb_release -a

Rocky Linux / RHEL / CentOS Stream:

    cat /etc/os-release
    hostnamectl

### 6.2 Check Kernel Version

    uname -r

Record current kernel:

    uname -a

### 6.3 Check if GPU is Recognized by PCIe

    lspci | grep -i nvidia

If there is no output, do not proceed with driver installation. Return to the hardware and BIOS layer for troubleshooting.

Priority checks:

- Is the GPU properly inserted;
- Is the power supply normal;
- Is the BIOS recognizing it;
- Is Above 4G Decoding enabled;
- Is the PCIe slot correct;
- Is the riser compatible;
- Does the server support the GPU.

### 6.4 Check PCIe Topology

    lspci -tv

Save:

    lspci -tv > pci-tree.txt

### 6.5 Check PCIe Link Capabilities

First, get the PCI ID:

    lspci | grep -i nvidia

Example:

    65:00.0 3D controller: NVIDIA Corporation Device xxxx

Check the link:

    lspci -vvv -s 65:00.0 | grep -i "LnkCap"
    lspci -vvv -s 65:00.0 | grep -i "LnkSta"

Pay attention to:

    LnkCap: Theoretical capability
    LnkSta: Current status

### 6.6 Check for nouveau Driver

nouveau is the open-source NVIDIA driver for Linux.

In production GPU computing scenarios, it is typically recommended to disable nouveau to avoid conflicts with the official NVIDIA driver.

Check:

    lsmod | grep nouveau

If there is output, it indicates that nouveau is loaded.

### 6.7 Check Secure Boot Status

Ubuntu commonly uses:

    mokutil --sb-state

If Secure Boot is enabled, the NVIDIA kernel module may fail to load due to lack of signing.

### 6.8 Check Existing NVIDIA Drivers

Debian / Ubuntu:

    dpkg -l | grep -i nvidia

Rocky / RHEL:

    rpm -qa | grep -i nvidia

Check if the command exists:

    which nvidia-smi

---

## Seven. Clean Up Old Drivers Before Installation

If the node previously installed NVIDIA drivers, it is recommended to clean up the old version to avoid conflicts.

### 7.1 Clean Up Old Drivers on Ubuntu

Check installed packages:

    dpkg -l | grep -i nvidia
    dpkg -l | grep -i cuda

Clean NVIDIA packages:

    sudo apt-get purge -y 'nvidia-*'
    sudo apt-get purge -y 'cuda-*'
    sudo apt-get autoremove -y
    sudo apt-get autoclean

Note:

    If there are existing business environments on the node, confirm whether cleaning will affect existing CUDA applications before proceeding.
    Do not arbitrarily purge on production nodes. Always perform maintenance windows and back up change records first.

### 7.2 Clean Up Old Drivers on Rocky / RHEL

Check:

    rpm -qa | grep -i nvidia
    rpm -qa | grep -i cuda

Clean:

    sudo dnf remove -y '*nvidia*'
    sudo dnf remove -y '*cuda*'

Or uninstall precisely based on actual package names.

### 7.3 Clean Up Drivers Installed via .run

If previously installed via .run files, typically use:

    sudo /usr/bin/nvidia-uninstall

Or:

    sudo nvidia-uninstall

Depending on the installation path.

---

## Eight. Disable nouveau

### 8.1 Create blacklist Configuration

General approach for Ubuntu / Rocky:

    sudo tee /etc/modprobe.d/blacklist-nouveau.conf > /dev/null <<'EOF'
    blacklist nouveau
    options nouveau modeset=0
    EOF

### 8.2 Update initramfs

Ubuntu:

    sudo update-initramfs -u

Rocky / RHEL:

    sudo dracut --force

### 8.3 Reboot the System

    sudo reboot

### 8.4 Verify nouveau is Not Loaded

    lsmod | grep nouveau

No output indicates nouveau is not loaded.

If output still exists, continue checking:

- Whether initramfs is updated;
- Kernel boot parameters;
- Whether graphical desktop dependencies on nouveau;
- Whether correctly rebooted to the target kernel.

---

## Nine. NVIDIA Driver Installation Methods for Ubuntu 22.04 / 24.04

The following uses Ubuntu Server as an example.

Production environments are recommended to prioritize package management methods to avoid using .run files arbitrarily.

### 9.1 Install Basic Dependencies

    sudo apt-get update
    sudo apt-get install -y build-essential dkms linux-headers-$(uname -r)
    sudo apt-get install -y pciutils lshw mokutil

Note: /think

build-essential: Compilation dependencies  
dkms: Dynamic building of kernel modules  
linux-headers: Header files for the current kernel  
pciutils: Provides lspci  
mokutil: Check Secure Boot status  

### 9.2 Using ubuntu-drivers to view recommended drivers  

Install the tool:  

    sudo apt-get install -y ubuntu-drivers-common  

View devices and recommended drivers:  

    ubuntu-drivers devices  

If there are recommended drivers in the output, you can install them according to the recommendation.  

Automatically install recommended drivers:  

    sudo ubuntu-drivers install  

Or install a specific version, for example:  

    sudo apt-get install -y nvidia-driver-<version>-server  

Example:  

    sudo apt-get install -y nvidia-driver-550-server  

Note:  

    <version> must be selected based on the current repository, GPU model, CUDA compatibility, and business requirements.  
    Do not directly copy the version number to all environments.  
    In production environments, verify first on test nodes.  

### 9.3 Installing drivers using NVIDIA CUDA Repository  

Suitable for scenarios where you want to use the NVIDIA official repository to manage drivers and CUDA-related packages uniformly.  

Example variables:  

    DISTRO=ubuntu2204  
    ARCH=x86_64  

The method of adding the NVIDIA CUDA repository keyring varies with CUDA versions and distributions; refer to the NVIDIA official repo page for the latest information.  

Common approach:  

    1. Download the cuda-keyring package for the corresponding distribution  
    2. Install cuda-keyring  
    3. apt-get update  
    4. Install cuda-drivers or the specified nvidia-driver package  

Example format:  

    wget https://developer.download.nvidia.com/compute/cuda/repos/${DISTRO}/${ARCH}/cuda-keyring_<版本>_all.deb  
    sudo dpkg -i cuda-keyring_<version>_all.deb  
    sudo apt-get update  
    sudo apt-get install -y cuda-drivers  

If you need a fixed version, use the explicit version package instead of blindly installing the latest.  

### 9.4 Reboot  

After installation, it is recommended to reboot:  

    sudo reboot  

### 9.5 Verification  

    nvidia-smi  
    lsmod | grep nvidia  
    ls -l /dev/nvidia*  

---  

## TenI don't know.Rocky Linux 9 / RHEL 9 Driver Installation Methods  

Rocky Linux 9 or RHEL 9 is commonly used in enterprise server environments.  

Before installation, focus on:  

- Whether kernel-devel matches the current kernel;  
- Whether gcc and make are installed;  
- Secure Boot;  
- Whether nouveau is disabled;  
- SELinux and security policies;  
- NVIDIA official repository or internal enterprise mirror.  

### 10.1 Install Basic Dependencies  

    sudo dnf install -y gcc make dkms kernel-devel-$(uname -r) kernel-headers-$(uname -r)  
    sudo dnf install -y pciutils lshw elfutils-libelf-devel  

If the current kernel version's kernel-devel is not found, it may be:  

- The current running kernel and the kernel-devel in the repository are inconsistent;  
- The system updated the kernel but hasn't rebooted;  
- The software source is incomplete;  
- A custom kernel is used.  

Check:  

    uname -r  
    rpm -qa | grep kernel-devel  

### 10.2 Disable nouveau  

Create configuration:  

    sudo tee /etc/modprobe.d/blacklist-nouveau.conf > /dev/null <<'EOF'  
    blacklist nouveau  
    options nouveau modeset=0  
    EOF  

Rebuild initramfs:  

    sudo dracut --force  

Reboot:  

    sudo reboot  

Verify:  

    lsmod | grep nouveau  

### 10.3 Configure NVIDIA CUDA Repository  

The actual repository address and key installation method should follow the NVIDIA official documentation.  

Common approach:  

    1. Add NVIDIA CUDA repo  
    2. Refresh dnf cache  
    3. Install nvidia-driver or cuda-drivers  
    4. Reboot  
    5. nvidia-smi to verify  

Example format:  

    sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo  
    sudo dnf clean all  
    sudo dnf makecache  
    sudo dnf install -y nvidia-driver  

If the enterprise internal network cannot access the internet, synchronize the NVIDIA repo to the internal yum/dnf source.  

### 10.4 Reboot and Verify  

    sudo reboot  

Verify:  

    nvidia-smi  
    lsmod | grep nvidia  
    ls -l /dev/nvidia*  

---  

## Eleven、.run File Installation Method  

.run files are suitable for experiments, offline, or special environments, but are not recommended as the preferred method for production clusters.  

### 11.1 Download Drivers  

Select from the NVIDIA official driver download page:  

- GPU model;  
- Operating system;  
- Linux 64-bit;  
- Data Center Driver or corresponding driver branch.  

After downloading, the file is similar to:  

    NVIDIA-Linux-x86_64-<version>.run  

### 11.2 Install Dependencies  

Ubuntu:  

    sudo apt-get update  
    sudo apt-get install -y build-essential dkms linux-headers-$(uname -r)  

Rocky / RHEL:  

    sudo dnf install -y gcc make dkms kernel-devel-$(uname -r) kernel-headers-$(uname -r)  

### 11.3 Stop Graphical Interface  

Servers typically do not have a graphical interface.  

If there is a desktop environment, you need to stop the display manager.  

Common example:  

    sudo systemctl isolate multi-user.target  

Or: /think

sudo systemctl stop gdm
sudo systemctl stop lightdm
sudo systemctl stop sddm

### 11.4 Execute Installation

    chmod +x NVIDIA-Linux-x86_64-<version>.run
    sudo ./NVIDIA-Linux-x86_64-<version>.run

Recommended during installation:

- Allow DKMS;
- Do not install 32-bit compatibility libraries unless absolutely necessary;
- Server compute nodes typically do not need OpenGL desktop components;
- Avoid mixing with system package managers.

### 11.5 Reboot and Verify

    sudo reboot

Verification:

    nvidia-smi

### 11.6 .run Installation Maintenance Issues

After .run installation, if the system upgrades the Kernel, it may be necessary to recompile or reinstall the driver.

Common issues:

- nvidia-smi fails after kernel upgrade;
- DKMS did not correctly rebuild;
- Secure Boot prevents new module loading;
- Incomplete uninstallation;
- Conflicts with apt/dnf packages.

Uninstallation:

    sudo nvidia-uninstall

Production recommendations:

    Unless there is a clear reason, do not prioritize .run files as long-term maintenance for Kubernetes GPU nodes.

---

## TwelveI don't know.Secure Boot Handling

### 12.1 Secure Boot Impact on Drivers

When Secure Boot is enabled, the Linux kernel may reject loading unsigned NVIDIA kernel modules.

Manifestations:

    lspci can see the GPU
    nvidia-smi reports errors
    lsmod does not show nvidia modules
    dmesg contains signature / key / module verification information

Check status:

    mokutil --sb-state

### 12.2 Common Handling Methods

Method one: Disable Secure Boot.

Suitable for experimental environments, internal network testing environments, or scenarios without strict Secure Boot requirements.

Method two: Sign NVIDIA kernel modules.

Suitable for production environments with security compliance requirements.

Method three: Use officially signed driver packages from the distribution.

Suitable for scenarios strictly following system distribution package management.

### 12.3 Troubleshooting Commands

    mokutil --sb-state
    lsmod | grep nvidia
    dmesg | grep -i secure
    dmesg | grep -i signature
    dmesg | grep -i nvidia

Production recommendations:

    Do not assume "driver installation success" means the module is loaded.
    In Secure Boot scenarios, verify whether the nvidia module is truly loaded.

---

## ThirteenI don't know.Basic Post-Installation Verification

After driver installation, verify in sequence.

### 13.1 Check if GPU is Visible

    nvidia-smi

Focus on:

- Driver Version;
- CUDA Version;
- GPU Model;
- Number of GPUs;
- Temperature;
- Power Usage;
- Memory Usage;
- Processes.

### 13.2 Check Detailed Information

    nvidia-smi -q

Recommended to save baseline:

    nvidia-smi > nvidia-smi.txt
    nvidia-smi -q > nvidia-smi-query.txt

### 13.3 Check Kernel Modules

    lsmod | grep nvidia

Common modules:

    nvidia
    nvidia_uvm
    nvidia_drm
    nvidia_modeset

### 13.4 Check Device Files

    ls -l /dev/nvidia*

Expected to see:

    /dev/nvidia0
    /dev/nvidiactl
    /dev/nvidia-uvm

Multi-GPU nodes may see:

    /dev/nvidia0
    /dev/nvidia1
    /dev/nvidia2
    /dev/nvidia3

### 13.5 Check Driver Version

    cat /proc/driver/nvidia/version

Or:

    nvidia-smi --query-gpu=driver_version --format=csv,noheader

### 13.6 Check GPU List

    nvidia-smi -L

Example:

    GPU 0: NVIDIA A100-SXM4-80GB (UUID: GPU-xxxx)
    GPU 1: NVIDIA A100-SXM4-80GB (UUID: GPU-yyyy)

### 13.7 Check Topology

    nvidia-smi topo -m

### 13.8 Check PCIe Status

    nvidia-smi -q | grep -i "PCI" -A 30

Or:

    lspci -vvv -s <GPU_PCI_ID> | grep -i "LnkCap"
    lspci -vvv -s <GPU_PCI_ID> | grep -i "LnkSta"

---

## FourteenI don't know.CUDA Basic Verification

After driver installation, you can perform basic CUDA layer verification.

Note:

    CUDA Toolkit is not required on all production GPU nodes.
    If it's just a Kubernetes GPU Worker, CUDA Runtime is usually provided by container images.
    However, for node delivery verification, you can temporarily install or use CUDA containers for testing.

### 14.1 Check nvcc

If CUDA Toolkit is installed:

    nvcc -V

If it prompts "command not found", it means the system does not have CUDA Toolkit installed or PATH is not configured.

This does not necessarily indicate driver issues.

### 14.2 Check libcuda

    ldconfig -p | grep libcuda

### 14.3 Python Framework Verification

If the node has Python and PyTorch, you can test:

    python3 -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.device_count())"

Do not recommend polluting the host Python environment for driver verification without installing PyTorch.

Container verification is more recommended.

---

## FifteenI don't know.Container Layer Verification

Kubernetes GPU nodes ultimately typically use containers for GPU access.

Therefore, after driver installation, you also need to verify whether the container layer can access the GPU.

This step depends on NVIDIA Container Toolkit.

### 15.1 Docker Scenario Testing

If the node uses Docker:

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

If successful, it indicates:

- The host driver is available;
- NVIDIA Container Toolkit is functional;
- Docker can expose GPUs to containers;
- The nvidia-smi inside the container can access GPUs.

### 15.2 containerd Scenario Explanation

Kubernetes production environments more commonly use containerd.

In the containerd scenario, you need to install and configure NVIDIA Container Toolkit to enable containerd support for NVIDIA runtime.

Common verification steps:

    1. The host nvidia-smi is functioning normally
    2. nvidia-container-cli info is functioning normally
    3. containerd has been configured with NVIDIA runtime
    4. Kubernetes Device Plugin is functioning normally
    5. nvidia-smi inside GPU Pods is functioning normally

Check nvidia-container-cli:

    nvidia-container-cli info

If this command does not exist, it indicates that NVIDIA Container Toolkit may not yet be installed.

### 15.3 Common Reasons for Container Testing Failures

Unable to use GPU inside containers, common reasons:

- Host driver anomalies;
- NVIDIA Container Toolkit not installed;
- Docker not configured with --gpus support;
- containerd runtime not configured;
- CUDA Runtime in container image incompatible with driver;
- /dev/nvidia* not properly mounted into container;
- Kubernetes Device Plugin not deployed;
- Pod not requesting nvidia.com/gpu.

---

## Sixteen, Kubernetes Pre-Access Validation

After driver installation, it does not mean Kubernetes can schedule GPUs.

Before accessing Kubernetes, perform node-side validation first.

### 16.1 Node-Side Checklist

    [ ] lspci can see all NVIDIA GPUs
    [ ] nvidia-smi is functioning normally
    [ ] GPU count matches expectations
    [ ] Driver Version has been recorded
    [ ] CUDA compatibility has been confirmed
    [ ] lsmod can see nvidia modules
    [ ] /dev/nvidia* device files exist
    [ ] nouveau is not loaded
    [ ] Secure Boot policy has been confirmed
    [ ] PCIe Speed / Width matches expectations
    [ ] nvidia-smi topo -m has been saved
    [ ] XID error checks show no serious anomalies
    [ ] Container layer GPU testing passed
    [ ] Node stress testing passed

### 16.2 Required Components in Kubernetes

Kubernetes typically requires:

- NVIDIA Container Toolkit;
- NVIDIA Device Plugin;
- Or NVIDIA GPU Operator;
- containerd / Docker runtime configuration;
- Node labels;
- Node taints;
- GPU Pod resources.limits;
- Monitoring components, such as DCGM Exporter.

Subsequent Kubernetes verification:

    kubectl describe node <gpu-node-name>

You need to see:

    Capacity:
      nvidia.com/gpu: <count>

    Allocatable:
      nvidia.com/gpu: <count>

If nvidia.com/gpu is not visible, it indicates that GPUs have not yet been registered as schedulable resources in Kubernetes.

---

## Seventeen, Pressure Testing After Driver Installation

After GPU driver installation, do not only check nvidia-smi once.

Normal idle state does not guarantee stability under full load.

### 17.1 Monitor Temperature and Power Consumption

    watch -n 2 nvidia-smi

Pay attention to:

- GPU-Util;
- Memory-Usage;
- Temp;
- Power Draw;
- Pstate;
- Processes.

### 17.2 Monitor System Logs

Open a window to continuously monitor kernel logs:

    journalctl -k -f

Or:

    dmesg -w

Focus on:

- NVRM;
- XID;
- PCIe AER;
- GPU fallen off the bus;
- BAR;
- IOMMU;
- thermal;
- power.

### 17.3 Check XID Errors

    dmesg | grep -i xid
    journalctl -k | grep -i xid

XID errors may be related to:

- Driver anomalies;
- GPU hardware failure;
- Application illegal access;
- VRAM errors;
- PCIe issues;
- Power supply issues;
- Temperature issues;
- Kernel compatibility issues.

### 17.4 Pressure Testing Recommendations

Optional testing methods:

- CUDA samples;
- gpu-burn;
- Real training tasks;
- Real inference services;
- Multi-GPU communication testing;
- Business image startup testing.

Production recommendations:

    At least one continuous pressure test should be completed before new GPU nodes go live.
    Only executing nvidia-smi cannot prove the node can support production tasks.

---

## Eighteen, Common Fault: lspci Can See GPU, but nvidia-smi Fails

### 18.1 Symptoms

Execute:

    lspci | grep -i nvidia

You can see NVIDIA devices.

Execute:

    nvidia-smi

Error:

    NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.

### 18.2 Possible Causes

Possible causes:

- NVIDIA driver not installed;
- Driver installation failed;
- Kernel module not loaded;
- Secure Boot prevents module loading;
- kernel headers missing;
- DKMS build failed;
- nouveau conflict;
- Current driver does not support the GPU;
- Driver module not rebuilt after kernel upgrade;
- Driver and kernel version incompatibility.

### 18.3 Troubleshooting Commands /think

lsmod | grep nvidia  
dkms status  
mokutil --sb-state  
dmesg | grep -i nvidia  
dmesg | grep -i secure  
dmesg | grep -i signature  
dmesg | grep -i nouveau  
dpkg -l | grep -i nvidia  
rpm -qa | grep -i nvidia  

### 18.4 Handling Approach  

Handling order:  

    1. Confirm whether nouveau is disabled  
    2. Confirm Secure Boot status  
    3. Confirm kernel headers match  
    4. Confirm DKMS success  
    5. Reinstall or switch driver version  
    6. Reboot the node  
    7. Re-verify nvidia-smi  

---  

## NineteenI don't know.Common Issue Two: Driver Fails After Reboot  

### 19.1 Phenomenon  

nvidia-smi works normally after driver installation.  

After reboot or system upgrade:  

    nvidia-smi fails  

### 19.2 Possible Causes  

Possible causes:  

- System boots into a new kernel;  
- New kernel lacks corresponding NVIDIA module;  
- DKMS automatic build fails;  
- kernel-devel/linux-headers mismatch;  
- Secure Boot blocks new module loading;  
- Driver package incompatible with current kernel;  
- .run installation method fails to handle new kernel properly.  

### 19.3 Troubleshooting Commands  

Check current kernel:  

    uname -r  

Check DKMS:  

    dkms status  

Ubuntu:  

    dpkg -l | grep linux-headers  
    dpkg -l | grep nvidia  

Rocky:  

    rpm -qa | grep kernel-devel  
    rpm -qa | grep nvidia  

Check logs:  

    dmesg | grep -i nvidia  
    journalctl -k | grep -i nvidia  

### 19.4 Handling Recommendations  

Handling methods:  

- Install headers matching current kernel;  
- Re-trigger DKMS build;  
- Roll back to previously working kernel;  
- Reinstall driver;  
- Fix kernel version;  
- Production GPU nodes should avoid uncontrolled kernel upgrades.  

Production recommendations:  

    GPU nodes should not upgrade kernel without control.  
    Kernel upgrades should first verify NVIDIA driver compatibility.  

---  

## TwentyI don't know.Common Issue Three: Nouveau Conflict  

### 20.1 Phenomenon  

Driver installation fails or nvidia-smi abnormal.  

Logs may contain nouveau-related information:  

    dmesg | grep -i nouveau  

### 20.2 Troubleshooting  

    lsmod | grep nouveau  
    dmesg | grep -i nouveau  

### 20.3 Handling  

Create blacklist:  

    sudo tee /etc/modprobe.d/blacklist-nouveau.conf > /dev/null <<'EOF'  
    blacklist nouveau  
    options nouveau modeset=0  
    EOF  

Ubuntu:  

    sudo update-initramfs -u  

Rocky:  

    sudo dracut --force  

Reboot:  

    sudo reboot  

Verify:  

    lsmod | grep nouveau  

---  

## Twenty-one、Common Issue Four: Secure Boot Prevents Module Loading  

### 21.1 Phenomenon  

nvidia-smi reports errors.  

lsmod shows no nvidia modules.  

Logs contain signature, verification, key-related information.  

### 21.2 Troubleshooting  

    mokutil --sb-state  
    lsmod | grep nvidia  
    dmesg | grep -i signature  
    dmesg | grep -i secure  
    dmesg | grep -i nvidia  

### 21.3 Handling  

Optional solutions:  

    1. Disable Secure Boot  
    2. Sign NVIDIA module  
    3. Use distribution-signed driver  

Experimental environments typically disable Secure Boot directly.  

Production environments must follow enterprise security policies.  

---  

## Twenty-two、Common Issue Five: CUDA Runtime Incompatible with Driver  

### 22.1 Phenomenon  

Container or application startup errors:  

    CUDA driver version is insufficient for CUDA runtime version  

Or:  

    torch.cuda.is_available() returns False  

Or:  

    TensorFlow cannot detect GPU  

### 22.2 Causes  

Host NVIDIA Driver too old to support CUDA Runtime in container images.  

Example:  

    Host driver version too old  
    Container image uses newer CUDA Runtime  
    Runtime detection failure  

### 22.3 Troubleshooting  

Check host:  

    nvidia-smi  

Check inside container:  

    nvidia-smi  
    python -c "import torch; print(torch.version.cuda); print(torch.cuda.is_available())"  

Check image tag:  

    docker image inspect <image>  

Or check Pod image:  

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep image:  

### 22.4 Handling  

Handling methods:  

- Upgrade host NVIDIA Driver;  
- Use container image with lower CUDA Runtime version;  
- Standardize business image CUDA version;  
- Select driver according to CUDA Compatibility;  
- Avoid different business teams using numerous different CUDA version images.  

Production recommendations:  

    GPU platforms should maintain standard base images.  
    Do not allow each business team to freely select CUDA image versions, as driver compatibility and troubleshooting costs will rise rapidly.  

---  

## Twenty-threeI don't know.Common Issue Six: nvidia-smi Shows No Devices Found  

### 23.1 Phenomenon  

Execute:  

    nvidia-smi  

Output:  

    No devices were found  

### 23.2 Possible Causes  

Possible causes:

- Driver is loaded, but not properly bound to GPU;  
- GPU is occupied by other drivers;  
- GPU is in an abnormal state;  
- PCIe layer anomaly;  
- GPU has fallen off the bus;  
- MIG or virtualization configuration anomaly;  
- Permission or device file anomaly;  
- GPU hardware failure.  

### 23.3 Troubleshooting  

    lspci | grep -i nvidia  
    lsmod | grep nvidia  
    ls -l /dev/nvidia*  
    dmesg | grep -i nvidia  
    dmesg | grep -i xid  
    journalctl -k | grep -i nvidia  

### 23.4 Handling  

Handling methods:  

- Confirm whether lspci can still see the GPU;  
- If lspci cannot see the GPU, go back to hardware and BIOS layer troubleshooting;  
- If lspci can see the GPU, check the driver module and logs;  
- Reboot the node for verification;  
- Check for XID or GPU fallen off the bus;  
- Replace the driver version or troubleshoot hardware if necessary.  

---  

## Twenty-four, Common Fault Seven: GPU fallen off the bus  

### 24.1 Phenomenon  

Logs show similar entries:  

    GPU has fallen off the bus  

or:  

    NVRM: Xid  

nvidia-smi may get stuck or fail to display the GPU.  

### 24.2 Possible Causes  

Possible causes:  

- PCIe connection anomaly;  
- GPU has fallen off the bus;  
- Insufficient power supply;  
- Overheating;  
- Riser card failure;  
- Motherboard slot issue;  
- BIOS / Firmware problem;  
- Driver bug;  
- GPU hardware failure.  

### 24.3 Troubleshooting  

    dmesg | grep -i "fallen"  
    dmesg | grep -i xid  
    dmesg | grep -i aer  
    journalctl -k | grep -i nvrm  
    ipmitool sel list  
    ipmitool sensor  
    nvidia-smi -q  

### 24.4 Handling  

Handling methods:  

- Check GPU temperature;  
- Check power supply and power cables;  
- Check riser card;  
- Check PCIe slot;  
- Update BIOS / BMC / Firmware;  
- Replace driver version;  
- Test with lower PCIe generation;  
- Single-card stress test;  
- Replace GPU for cross-validation.  

---  

## Twenty-five, Common Fault Eight: DKMS Build Failure  

### 25.1 Phenomenon  

DKMS error occurs during driver installation.  

Or the module is not loaded after installation.  

### 25.2 Possible Causes  

Possible causes:  

- Missing kernel headers;  
- kernel-devel version mismatch;  
- gcc version issue;  
- Kernel version is too new;  
- Driver version does not support current kernel;  
- Secure Boot;  
- Residual old driver files.  

### 25.3 Troubleshooting  

    dkms status  
    uname -r  

Ubuntu:  

    dpkg -l | grep linux-headers  

Rocky:  

    rpm -qa | grep kernel-devel  

Check DKMS logs:  

    ls /var/lib/dkms/  
    find /var/lib/dkms/ -name make.log  

### 25.4 Handling  

Ubuntu:  

    sudo apt-get install -y linux-headers-$(uname -r)  
    sudo apt-get install --reinstall -y nvidia-driver-<version>-server  

Rocky:  

    sudo dnf install -y kernel-devel-$(uname -r) kernel-headers-$(uname -r)  
    sudo dnf reinstall -y nvidia-driver  

If current kernel is too new or incompatible:  

    1. Switch to supported kernel  
    2. Replace driver version  
    3. Wait for official support  
    4. Use verified enterprise kernel version  

---  

## Twenty-six, Common Fault Nine: No GPU Visibility in Container  

### 26.1 Phenomenon  

Host:  

    nvidia-smi  

Works normally.  

Container:  

    nvidia-smi  

Fails.  

### 26.2 Possible Causes  

Possible causes:  

- NVIDIA Container Toolkit not installed;  
- Docker not using --gpus all;  
- containerd not configured with NVIDIA runtime;  
- Kubernetes not deployed Device Plugin;  
- Pod not requesting nvidia.com/gpu;  
- Container image lacks nvidia-smi;  
- /dev/nvidia* not mounted;  
- Driver and container CUDA Runtime incompatibility.  

### 26.3 Docker Troubleshooting  

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi  

If failed:  

    nvidia-container-cli info  
    docker info | grep -i runtime  

### 26.4 Kubernetes Troubleshooting  

Check if GPU is registered on node:  

    kubectl describe node <gpu-node-name>  

Check Device Plugin:  

    kubectl get pods -A | grep -i nvidia  
    kubectl get ds -A | grep -i nvidia  

Check Pod:  

    kubectl describe pod <gpu-pod-name> -n <namespace>  

Pod must include similar entries:  

    resources:  
      limits:  
        nvidia.com/gpu: 1  

---  

## Twenty-seven, Driver Upgrade and Rollback  

### 27.1 Driver Upgrade Principles  

Do not arbitrarily upgrade drivers on production GPU nodes.  

Before upgrading, confirm:  

- Whether new driver supports current GPU;  
- Whether supports current Kernel;  
- Whether compatible with business CUDA Runtime;  
- Whether compatible with PyTorch / TensorFlow;  
- Whether compatible with NVIDIA Container Toolkit;  
- Whether compatible with Device Plugin / GPU Operator;  
- Whether tested on test nodes;  
- Whether rollback plan exists.  

### 27.2 Recommended Upgrade Process  

Recommended process: /think

1. Select a test GPU node  
2. Cordon the node  
3. Drain the node's business Pods  
4. Backup the current version information  
5. Upgrade the driver  
6. Reboot the node  
7. Validate `nvidia-smi`  
8. Validate CUDA containers  
9. Validate GPU Pods  
10. Run business testing  
11. Observe for 24 hours or a business cycle  
12. Batch rolling upgrade  

Kubernetes node maintenance commands:  

    kubectl cordon <gpu-node-name>  
    kubectl drain <gpu-node-name> --ignore-daemonsets --delete-emptydir-data  

Recovery:  

    kubectl uncordon <gpu-node-name>  

### 27.3 Save Baseline Before Upgrade  

    nvidia-smi > before-nvidia-smi.txt  
    nvidia-smi -q > before-nvidia-smi-query.txt  
    nvidia-smi topo -m > before-gpu-topology.txt  
    uname -a > before-kernel.txt  
    lsmod | grep nvidia > before-nvidia-modules.txt  
    dpkg -l | grep -i nvidia > before-nvidia-packages.txt  
    rpm -qa | grep -i nvidia > before-nvidia-rpms.txt  

### 27.4 Rollback Strategy  

Rollback methods:  

- Use the package manager to install the old version;  
- Use system snapshots;  
- Use image rollback;  
- Use configuration management tools to restore;  
- Reinstall the original driver version;  
- Rollback Kernel.  

Production recommendations:  

    GPU driver upgrades must have a rollback version and rollback window.  
    Do not batch upgrade production GPU nodes without backup and validation.  

---

## Twenty-Eight, Driver Version and Node Baseline Record  

It is recommended to record the driver baseline for each GPU node.  

Template:  

    Node name:  
    Asset number:  
    Operating system:  
    Kernel version:  
    GPU model:  
    GPU count:  
    NVIDIA Driver Version:  
    nvidia-smi CUDA Version:  
    CUDA Toolkit Version:  
    Container Runtime:  
    NVIDIA Container Toolkit Version:  
    Device Plugin / GPU Operator Version:  
    Secure Boot status:  
    nouveau status:  
    Installation method:  
    Installation time:  
    Installer:  
    Validation result:  
    Stress test result:  

Recommended command outputs to save:  

    hostnamectl > host-info.txt  
    uname -a > kernel-info.txt  
    lspci | grep -i nvidia > gpu-lspci.txt  
    nvidia-smi > nvidia-smi.txt  
    nvidia-smi -q > nvidia-smi-query.txt  
    nvidia-smi -L > gpu-list.txt  
    nvidia-smi topo -m > gpu-topology.txt  
    lsmod | grep nvidia > nvidia-modules.txt  
    ls -l /dev/nvidia* > nvidia-devices.txt  
    dmesg | grep -i nvidia > dmesg-nvidia.txt  
    dmesg | grep -i xid > dmesg-xid.txt  

---

## Twenty-Nine, Production Environment Driver Management Recommendations  

### 29.1 Fixed Version  

Production environments are recommended to fix the driver version.  

Do not allow nodes to automatically drift to different versions.  

Recommendations:  
- Use internal software repositories;  
- Fix package versions;  
- Record version matrices;  
- Follow change procedures before upgrades;  
- Batch rolling upgrades;  
- Avoid unattended automatic GPU driver upgrades.  

### 29.2 Fixed Kernel  

GPU drivers depend on kernel modules.  

Kernel upgrades may cause driver failures.  

Recommendations:  
- Disable uncontrolled automatic kernel upgrades on GPU nodes;  
- Validate kernel upgrades on test nodes first;  
- Retain old kernel boot options;  
- Record the compatibility matrix between Kernel and Driver.  

### 29.3 Standardized Base Image  

If business uses containers, it is recommended to maintain standardized CUDA base images on the platform.  

Examples:  

    company/cuda:12.2-runtime-ubuntu22.04  
    company/pytorch:2.x-cuda12.2-runtime  
    company/tensorflow:cuda12.x-runtime  

Do not allow business teams to directly use various CUDA images from unknown sources.  

### 29.4 Use Automation Tools for Batch Management  

For batch GPU nodes, it is recommended to use:  

- Ansible;  
- SaltStack;  
- Terraform + cloud-init;  
- Image templates;  
- Internal yum/apt repositories;  
- Kubernetes GPU Operator;  
- CMDB to record versions.  

### 29.5 Driver Installation Does Not Equal GPU Platform Completion  

Drivers are just the foundation.  

A complete GPU platform also requires:  

- NVIDIA Container Toolkit;  
- containerd configuration;  
- Device Plugin / GPU Operator;  
- DCGM Exporter;  
- Prometheus;  
- Grafana;  
- AlertManager;  
- Log collection;  
- Node labels and taints;  
- ResourceQuota;  
- Multi-tenancy isolation;  
- Image standards;  
- Change and rollback procedures.  

---

## Thirty, Complete Installation Checklist for Drivers  

### 30.1 Before Installation  

    [ ] GPU hardware installation is complete  
    [ ] BIOS has been configured  
    [ ] Above 4G Decoding has been confirmed  
    [ ] lspci can see NVIDIA GPU  
    [ ] Operating system version has been confirmed  
    [ ] Kernel version has been confirmed  
    [ ] kernel headers / kernel-devel are available  
    [ ] nouveau has been disabled  
    [ ] Secure Boot policy has been confirmed  
    [ ] Driver version has been selected  
    [ ] CUDA compatibility has been confirmed  
    [ ] Installation method has been determined  
    [ ] Old driver has been cleaned or conflicts confirmed  
    [ ] Production nodes have entered maintenance window

### 30.2 Installation in Progress

    [ ] Dependencies installed successfully
    [ ] Driver package installed successfully
    [ ] DKMS build successful
    [ ] No obvious installation errors
    [ ] No nouveau conflicts
    [ ] No Secure Boot interception
    [ ] Node has been restarted

### 30.3 After Installation

    [ ] nvidia-smi works normally
    [ ] Correct number of GPUs
    [ ] Correct Driver Version
    [ ] nvidia module visible in lsmod
    [ ] /dev/nvidia* exists
    [ ] nvidia-smi -q works normally
    [ ] nvidia-smi topo -m works normally
    [ ] dmesg has no severe XID errors
    [ ] Container GPU test passed
    [ ] Kubernetes pre-check passed
    [ ] Monitoring and alerting ready for integration

---

## Thirty-one, Understanding the Driver Layer from a Troubleshooting Perspective

GPU troubleshooting should be layered, not start with reinstalling the driver.

### 31.1 lspci Does Not Show GPU

This likely indicates issues with:

    Hardware
    BIOS
    PCIe
    riser
    Power supply
    Above 4G Decoding

Do not prioritize reinstalling the driver.

### 31.2 lspci Can See It, but nvidia-smi Is Abnormal

This likely indicates issues with:

    NVIDIA Driver
    Kernel Module
    Secure Boot
    nouveau
    DKMS
    Kernel Headers
    Driver version compatibility

This is when you should focus on the driver layer.

### 31.3 nvidia-smi Works Normally, but Container Is Abnormal

This likely indicates issues with:

    NVIDIA Container Toolkit
    Docker / containerd runtime
    Container image
    CUDA Runtime compatibility
    /dev/nvidia* mount

### 31.4 Container Works Normally, but Kubernetes Is Abnormal

This likely indicates issues with:

    Device Plugin
    GPU Operator
    kubelet resource registration
    nvidia.com/gpu
    Pod resources.limits
    taint / toleration
    nodeSelector / affinity
    RuntimeClass

---

## Thirty-two, Summary

NVIDIA driver is the critical layer that transforms a GPU node from "hardware visible" to "computation available."

The core goal of GPU driver installation is not simply "getting nvidia-smi to run," but establishing a stable, verifiable, upgradable, and reversible node foundation.

The correct driver installation and verification process should be:

    1. BIOS and hardware check
    2. Confirm GPU visibility via lspci
    3. Choose appropriate driver version
    4. Disable nouveau
    5. Confirm Secure Boot policy
    6. Install kernel headers / kernel-devel
    7. Install driver via package manager or official repository
    8. Restart node
    9. Verify with nvidia-smi
    10. Verify kernel module with lsmod
    11. Verify device files /dev/nvidia*
    12. Verify with CUDA or container layer
    13. Stress test
    14. Integrate with Kubernetes Device Plugin / GPU Operator
    15. Integrate with Prometheus / DCGM Exporter monitoring

When troubleshooting, adhere to the layered approach:

    lspci is abnormal:
        Check hardware and BIOS

    nvidia-smi is abnormal:
        Check driver and kernel module

    Container is abnormal:
        Check NVIDIA Container Toolkit and runtime

    Kubernetes is abnormal:
        Check Device Plugin, kubelet, and scheduling configuration

The key to GPU operations is not memorizing commands, but connecting the relationships between hardware, driver, CUDA, containers, and Kubernetes.

Only with a stable driver layer can subsequent CUDA, Device Plugin, GPU Operator, Prometheus GPU monitoring, and GPU Pod scheduling have a reliable foundation.

---

## Reference Documents

- NVIDIA Driver Installation Guide:
  https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html

- NVIDIA Ubuntu Driver Installation Guide:
  https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/ubuntu.html

- NVIDIA CUDA Compatibility:
  https://docs.nvidia.com/deploy/cuda-compatibility/

- NVIDIA CUDA Toolkit Documentation:
  https://docs.nvidia.com/cuda/

- NVIDIA Container Toolkit Installation Guide:
  https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

- NVIDIA GPU Operator Documentation:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/

- Kubernetes GPU Scheduling:
  https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/

- NVIDIA Kubernetes Device Plugin:
  https://github.com/NVIDIA/k8s-device-plugin

- NVIDIA DCGM Exporter:
  https://github.com/NVIDIA/dcgm-exporter