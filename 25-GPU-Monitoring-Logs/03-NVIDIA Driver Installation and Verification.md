# 03-NVIDIA Driver Installation and Verification

## Document Description

This document provides guidance on installing, verifying, selecting NVIDIA drivers for Linux GPU nodes, checking kernel modules, troubleshooting common issues, and conducting preliminary checks for Kubernetes integration.

This document focuses on the "driver layer" within the GPU operations and maintenance process.

The path from GPU hardware to its availability in Kubernetes generally involves the following steps:

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
    Kubernetes Scheduling via nvidia.com/gpu
      ↓
    GPU Pods Utilizing the GPU

This document primarily covers:

    Linux Kernel
      ↓
    NVIDIA Driver
      ↓
    nvidia-smi
      ↓
    The interrelationships between CUDA, Containers, and Kubernetes.

This document does not delve into the installation of the CUDA Toolkit or the deployment of Device Plugins and GPU Operators. CUDA installation will be covered in subsequent notes, and Kubernetes GPU scheduling will be discussed in the dedicated K8S GPU chapter.

---

## Tags

#GPU #NVIDIA #Driver #Linux #Ubuntu #RockyLinux #CUDA #Kubernetes #Containerd #SRE #Operations and Maintenance Troubleshooting

---

## Recommended Reading Path

Recommended reading path:

    06-GPU and AI Infrastructure/01-GPU Basics/03-NVIDIA Driver Installation and Verification.md

---

## I. Why the NVIDIA Driver is a Critical Layer for GPU Nodes

A GPU cannot be directly used by applications just by being connected to a server.

The Linux system can recognize NVIDIA devices through PCIe, but this only indicates that the hardware has been detected; it does not mean the GPU is ready for use with CUDA, containers, or Kubernetes.

For a GPU node to be functional, at least three layers must be in place:

    First Layer: Hardware Visibility
        The NVIDIA device should be visible using `lspci`.

    Second Layer: Driver Availability
        `nvidia-smi` should be able to display the GPU correctly.

    Third Layer: Runtime Functionality
        Applications such as CUDA, containers, and Kubernetes Pods should be able to access the GPU.

The NVIDIA driver plays a crucial role in the second layer, acting as a bridge between the hardware and the rest of the system. If there are issues with the driver layer, common symptoms include:

- The GPU is visible through `lspci`, but `nvidia-smi` reports errors;
- `nvidia-smi` indicates "No devices were found";
- "NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver";
- The `lsmod` command shows no NVIDIA kernel modules;
    Secure Boot may prevent the driver modules from loading;
    Driver failures after kernel updates;
    Kubernetes nodes being unable to register with nvidia.com/gpu;
    Abnormalities in Device Plugin operations;
    GPU Pod scheduling failures;
    Inability to execute `nvidia-smi` inside containers;
    CUDA programs reporting incompatible driver versions.

Therefore, GPU operations and maintenance personnel should not only be able to run `nvidia-smi`, but also understand that:

- The visibility of the GPU through `lspci` indicates that the PCIe layer is functioning correctly;
- Proper operation of `nvidia-smi` suggests that the driver layer is available;
- A successful `nvcc -V` command confirms the presence of the CUDA Toolkit;
- The ability to run `nvidia-smi` in containers means that GPU integration at the container runtime level is working;
- Seeing `nvidia.com/gpu` listed in `kubectl describe node` indicates that Kubernetes resource registration is functioning properly.

---

## II. The Role of NVIDIA Drivers in the System

NVIDIA drivers enable the operating system and user-space applications to access GPUs.

They primarily include:

- Kernel modules;
- User-space libraries;
- Management tools;
- Device files;
- Support for the CUDA Runtime;
- GPU management interfaces;
    Memory management functions;
    Management capabilities for temperature, power consumption, clock speed, ECC, etc.

Common related components include:

    nvidia
    nvidia_uvm
    nvidia_drm
    nvidiamodeset
    nvidia-smi
    libcuda.so
    /dev/nvidia0
    /dev/nvidiactl
    /dev/nvidia-uvm

### 2.1 Kernel Modules

To check for NVIDIA kernel modules, use:

    `lsmod | grep nvidia`

Common output includes:

    nvidia
    nvidia_uvm
    nvidia_drm
    nvidiamodeset

If no output is displayed, it means the NVIDIA driver modules have not been loaded.

### 2Select a verified and stable version of the driver.
After verification on the test nodes, deploy it in batches.
Try to keep the driver versions consistent across the same batch of GPU nodes.

### 4.2 Prefer Data Center Driver for Data Center GPUs

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

It is recommended to prefer the NVIDIA Data Center Driver branch.

Reasons:

- It is more suitable for server and data center environments;
- It places greater emphasis on stability;
- It is better suited for AI training and inference;
- It is more compatible with Kubernetes GPU nodes;
- It works well with components such as DCGM and GPU Operator.

### 4.3 Compatibility between Drivers and CUDA

Whether a CUDA application can run depends on the compatibility between the required CUDA Runtime and the host NVIDIA Driver.

General principle:

The host driver version must support the CUDA Runtime needed by the container or application.

For example:

If the container image uses CUDA 12.x Runtime, the host driver must meet the minimum requirements of CUDA 12.x for the driver version.

If the driver is too old, issues such as:

- The CUDA driver version is insufficient for the CUDA runtime version
- No kernel image is available for execution on the device
- Or the application framework cannot detect the GPU may occur.

### 4.4 Suggestions for Choosing Driver Versions

It is recommended to determine the version in the following order:

1. Confirm the GPU model.
2. Confirm the operating system version.
3. Confirm the CUDA version used by the business image.
4. Check the NVIDIA official CUDA Compatibility.
5. Check the support for NVIDIA Data Center Driver.
6. Install and verify on test nodes.
7. Run nvidia-smi, CUDA examples, and container GPU tests.
8. Run tests with the business image or training/inference frameworks.
9. Finalize the version and deploy it in batches.

---

## V. Selection of Installation Methods

There are three common ways to install NVIDIA drivers:

1. Installation through the operating system's package manager.
2. Installation via the NVIDIA CUDA Repository.
3. Installation using a .run file downloaded from NVIDIA's official website.

For production environments, it is recommended to use the package manager method.

### 5.1 Installation through Package Managers

Advantages:

- Better maintainability;
- Easy to upgrade and uninstall;
- Consistent with the system's package management framework;
- Suitable for batch deployment;
- Ideal for automated operations and maintenance;
- Easier management of DKMS and Kernel Header dependencies.

Disadvantages:

- Choosing the right version requires understanding the source of the repository;
- Package names may vary across different distributions;
- In corporate intranet environments, software sources or mirror sources need to be configured.

### 5.2 Installation via the NVIDIA CUDA Repository

Advantages:

- Comes from NVIDIA's official repository;
- Offers a wide range of versions;
- Suitable for data center GPUs;
- Can be installed in conjunction with cuda-drivers or nvidia-driver packages.

Disadvantages:

- Access may be unstable in domestic network environments;
- Requires configuration of GPG keys and repositories;
- Version management requires careful attention;
- It is recommended to create an internal mirror source for batch environments.

### 5.3 Installation using a .run File

A .run file is an installation package downloaded from NVIDIA's official website.

Advantages:

- Direct and straightforward;
- Suitable for offline or special environments;
- Does not rely on the system repository.

Disadvantages:

- Not conducive to package management;
- Prone to becoming ineffective after kernel upgrades;
- Uninstalling and upgrading are less straightforward than with package managers;
- Higher costs for automated batch maintenance;
- Likely to conflict with system packages;
- Not recommended for long-term maintenance in production environments.

Production recommendations:

- Use .run files for testing or temporary environments.
- For production environments, Kubernetes GPU nodes, and batch GPU nodes, prefer the package manager method or NVIDIA's official repository.

---

## VI. Pre-installation Checks

Before installing NVIDIA drivers, do not rush to execute the installation command.

First, confirm the status of your hardware, system, and kernel.

### 6.1 Checking the Operating System Version

Ubuntu:

    cat /etc/os-release
    lsb_release -a

Rocky Linux / RHEL / CentOS Stream:

    cat /etc/os-release
    hostnamectl

### 6.2 Checking the Kernel Version

    uname -r

Record the current kernel version:

    uname -a

### 6.3 Checking if the GPU is Recognized by PCIe

    lspci | grep -i nvidia

If there is no output, do not continue with the driver installation. First, check the hardware and BIOS settings.

Priority checks include```markdown
🔤 Install ubuntu-drivers-common using sudo apt-get:

    sudo apt-get install -y ubuntu-drivers-common

Check devices and recommended drivers:

    ubuntu-drivers devices

If recommended drivers are listed, install them as suggested.

Automatically install recommended drivers:

    sudo ubuntu-drivers install

Or install a specific version, for example:

    sudo apt-get install -y nvidia-driver-<version_number>-server

Example:

    sudo apt-get install -y nvidia-driver-550-server

Note:

    <version_number> should be chosen based on your current repository, GPU model, CUDA compatibility, and business requirements.
    Do not copy the version number directly into all environments.
    In a production environment, test it first on a staging node.

### 9.3 Using the NVIDIA CUDA Repository for Driver Installation

This method is suitable for those who want to use the official NVIDIA repository to manage drivers and CUDA-related packages uniformly.

Example variables:

    DISTRO=ubuntu2204
    ARCH=x86_64

The process of adding the NVIDIA CUDA repository keyring may vary depending on the CUDA version and distribution. Refer to the NVIDIA official repo documentation for specific instructions.

Common steps:

    1. Download the corresponding cuda-keyring package for your distribution.
    2. Install the cuda-keyring.
    3. Update your package list with apt-get update.
    4. Install the nvidia-drivers or specified nvidia-driver packages.

Example command:

    wget https://developer.download.nvidia.com/compute/cuda/repos/${DISTRO}/${ARCH}/cuda-keyring_<version>_all.deb
    sudo dpkg -i cuda-keyring_<version>_all.deb
    sudo apt-get update
    sudo apt-get install -y cuda-drivers

If you need a specific version, use the exact version number instead of installing the latest version.

### 9.4 Restart After Installation

It is recommended to restart your system after installation:

    sudo reboot

### 9.5 Verification

    nvidia-smi
    lsmod | grep nvidia
    ls -l /dev/nvidia*

---

## Ten: Driver Installation Methods for Rocky Linux 9 / RHEL 9

Rocky Linux 9 or RHEL 9 are commonly used in enterprise server environments.

Key considerations before installation:

- Ensure that kernel-devel matches the current kernel version.
- Make sure gcc and make are installed.
- Check Secure Boot settings.
- Disable nouveau if necessary.
- Configure SELinux and security policies appropriately.
- Use the official NVIDIA repository or an internal corporate mirror.

### 10.1 Install Basic Dependencies

    sudo dnf install -y gcc make dkms kernel-devel-$(uname -r) kernel-headers-$(uname -r)
    sudo dnf install -y pciutils lshw elfutils-libelf-devel

If you receive a message indicating that kernel-devel for your current kernel version is not available, it may be due to:

- The kernel currently in use does not match the kernel-devel in the repository.
- The system has updated its kernel without restarting.
- The software sources are incomplete.
- You are using a custom kernel.

Verify the following:

    uname -r
    rpm -qa | grep kernel-devel

### 10.2 Disable nouveau

Create a configuration file:

    sudo tee /etc/modprobe.d/blacklist-nouveau.conf > /dev/null <<'EOF'
    blacklist nouveau
    options nouveau modeset=0
    EOF

Rebuild the initramfs:

    sudo dracut --force

Restart your system:

    sudo reboot

Verify installation:

    lsmod | grep nouveau

### 10.3 Configure the NVIDIA CUDA Repository

Refer to NVIDIA's official documentation for the actual repository address and key installation methods.

Common steps:

    1. Add the NVIDIA CUDA repository.
    2. Refresh the dnf cache.
    3. Install nvidia-driver or cuda-drivers.
    4. Restart your system.
    5. Verify installation with nvidia-smi.

Example command:

    sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo
    sudo dnf clean all
    sudo dnf makecache
    sudo dnf install -y nvidia-driver

If your internal network does not have access to the internet, you will need to synchronize the NVIDIA repository with your local yum/dnf sources.

### 10.4 Restart and Verify Installation

    sudo reboot

Verify installation:

    nvidia-smi
    lsmod | grep nvidia
    ls -l /dev/nvidia*

---

## Eleven: Using .run Files for Driver Installation

.run files are suitable for experimental, offline, or special-use environments but are not recommended as a primary method for production clusters.

### 11.1 Download```bash
nvidia-smi --force
nvidia-smi -v
lsmod | grep nvidia
ldconfig -p | grep libcuda
``````bash
lsmod | grep nvidia
dkms status
mokutil --sb-state
dmesg | grep -i nvidia
dmesg | grep -i secure
dmesg | grep -i signature
dmesg | grep -i nouveau
dpkg -l | grep -i nvidia
rpm -qa | grep -i nvidia
``````
sudo apt-get install --reinstall -y nvidia-driver-<version number>-server

Rocky:

sudo dnf install -y kernel-devel-$(uname -r) kernel-headers-$(uname -r)
sudo dnf reinstall -y nvidia-driver

If the current kernel is too new or incompatible:

1. Switch to a supported kernel.
2. Change the driver version.
3. Wait for official support.
4. Use a verified enterprise kernel version.

---

## Chapter 26: Common Issue 9: GPU Not Visible in Containers

### 26.1 Symptoms

On the host machine:

nvidia-smi

Works normally.

Inside the container:

nvidia-smi

Fails to execute.

### 26.2 Possible Causes

Possible reasons include:

- The NVIDIA Container Toolkit is not installed.
- Docker does not use the --gpus all option.
- containerd is not configured with the NVIDIA runtime.
- Kubernetes has not deployed the Device Plugin.
- The Pod has not requested access to nvidia.com/gpu.
- The container image does not contain nvidia-smi.
- The /dev/nvidia* directories are not mounted.
- The driver and the container's CUDA Runtime are incompatible.

### 26.3 Docker Troubleshooting

Run the following command in a container:

docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

If this fails, check the following:

nvidia-container-cli info
docker info | grep -i runtime

### 26.4 Kubernetes Troubleshooting

To check if a node has registered its GPU:

kubectl describe node <gpu-node-name>

To check the Device Plugin:

kubectl get pods -A | grep -i nvidia
kubectl get ds -A | grep -i nvidia

To check a specific Pod:

kubectl describe pod <gpu-pod-name> -n <namespace>

The Pod should have settings like this in its resources section:

limits:
  nvidia.com/gpu: 1
```

---

## Chapter 27: Driver Updates and Rollbacks

### 27.1 Principles for Driver Updates

Do not update the drivers on production GPU nodes casually.

Before updating, ensure that:

- The new driver is compatible with the current GPU.
- It supports the current Kernel version.
- It is compatible with the business-specific CUDA Runtime.
- It is compatible with PyTorch/TensorFlow.
- It is compatible with the NVIDIA Container Toolkit.
- It is compatible with the Device Plugin/GPU Operator.
- The update has been tested on nodes.
- There is a plan for rolling back in case of issues.

### 27.2 Recommended Update Process

Follow these steps:

1. Select a test GPU node.
2. Isolate the node from the rest of the system.
3. Drain all business Pods from the node.
4. Back up the current configuration and driver version.
5. Update the driver.
6. Restart the node.
7. Verify that nvidia-smi works correctly.
8. Check if CUDA containers function properly.
9. Verify that GPU-related Pods are functioning correctly.
10. Run a business test.
11. Monitor the system for 24 hours or one business cycle.
12. Roll out the update to other nodes in batches.

For Kubernetes nodes, use these commands:

kubectl cordon <gpu-node-name>
kubectl drain <gpu-node-name> --ignore-daemonsets --delete-emptydir-data

To restore the node:

kubectl uncordon <gpu-node-name>

### 27.3 Saving Baseline Data Before Updates

Before updating the driver, save the following information:

nvidia-smi > before-nvidia-smi.txt
nvidia-smi -q > before-nvidia-smi-query.txt
nvidia-smi topo -m > before-gpu-topology.txt
uname -a > before-kernel.txt
lsmod | grep nvidia > before-nvidia-modules.txt
dpkg -l | grep -i nvidia > before-nvidia-packages.txt
rpm -qa | grep -i nvidia > before-nvidia-rpms.txt

### 27.4 Rollback Strategies

Possible rollback methods include:

- Using the package manager to install the previous driver version.
- Using a system snapshot.
- Rolling back using an image.
- Using configuration management tools to restore settings.
- Reinstalling the original driver version.
- Rolling back the Kernel.

For production environments, it is essential to have a backup version of the driver and a designated rollback period. Do not update production GPU nodes in batches without proper backups and testing.

---

## Chapter 28: Recording Driver Versions and Node Baseline Information

It is recommended to record detailed information about each GPU node, including its driver version and other configuration details.

Use this template:

Node Name:
Asset Number:
Operating System:
Kernel Version:
GPU Model:
Number[ ] No Secure Boot interference detected
[ ] Node has been restarted

### Post-Installation Steps (30.3)

[ ] nvidia-smi is functioning normally
[ ] The number of GPUs matches the actual count
[ ] The driver version is correct
[ ] The nvidia module is visible in lsmod
[ ] The /dev/nvidia* directory exists
[ ] nvidia-smi -q and nvidia-smi topo -m commands return expected results
[ ] No serious XID errors are reported in dmesg
[ ] Container GPU tests have passed
[ ] Pre-checks for Kubernetes have been successful
[ ] Monitoring and alert systems are ready to be integrated

---

## Section 31: Understanding the Driver Layer from a Troubleshooting Perspective

When troubleshooting GPU issues, it's important to approach them in a layered manner rather than immediately reinstalling the driver.

### 31.1 If the GPU is not visible in lspci

The problem is likely related to:

    Hardware
    BIOS
    PCIe connection
    Riser card
    Power supply
    Issues with 4G RAM decoding

In this case, reinstalling the driver should not be the first step.

### 31.2 If the GPU is visible in lspci but nvidia-smi is not working properly

The issue may lie with:

    NVIDIA Driver
    Kernel module
    Secure Boot settings
    nouveau driver
    DKMS configuration
    Kernel headers
    Driver version compatibility

Only at this point should you focus on checking the driver layer.

### 31.3 If nvidia-smi is working normally, but issues occur within containers

The problem could be related to:

    NVIDIA Container Toolkit
    Docker / containerd runtime
    Container image configuration
    CUDA Runtime compatibility
    Mounting of /dev/nvidia* directory

### 31.4 If the container is functioning correctly, but Kubernetes encounters problems

Possible causes include:

    Device Plugin configuration
    GPU Operator settings
    kubelet resource registration issues
    Issues with nvidia.com/gpu service
    Pod resource limits
    Taints and tolerances settings
    NodeSelector or affinity rules
    RuntimeClass configurations

---

## Section 32: Summary

The NVIDIA driver plays a crucial role in ensuring that GPU nodes are not only "hardware-visible" but also effectively usable for computing tasks.

The core goal of installing a NVIDIA driver is not just to get nvidia-smi running, but to establish a stable, verifiable, upgradable, and recoverable foundation for the node's functionality.

The correct procedure for installing and verifying a NVIDIA driver should include:

1. Checking the BIOS and hardware configuration.
2. Using lspci to confirm that the GPU is detected.
3. Selecting an appropriate driver version.
4. Disabling the nouveau driver if it is present.
5. Adjusting Secure Boot settings as needed.
6. Installing kernel headers and kernel-devel.
7. Using package managers or official repositories to install the driver.
8. Restarting the node.
9. Verifying the driver installation with nvidia-smi.
10. Checking the kernel module list using lsmod.
11. Confirming the presence of the /dev/nvidia* directory.
12. Testing CUDA functionality within containers, if applicable.
13. Performing stress tests to ensure stability.
14. Integrating the NVIDIA Driver with Kubernetes Device Plugin and GPU Operator.
15. Setting up Prometheus and DCGM Exporter for monitoring.

When troubleshooting, follow a structured approach based on the specific issue:

- If lspci shows no GPU, focus on hardware and BIOS issues.
- If nvidia-smi is faulty, examine the driver and kernel module configuration.
- If container-based issues arise, check the NVIDIA Container Toolkit and runtime environment.
- If Kubernetes problems occur, investigate Device Plugin settings, kubelet operations, and scheduling configurations.

The key to effective GPU operation and maintenance lies in understanding how hardware, drivers, CUDA, containers, and Kubernetes work together. Only when all these components are properly configured and functioning smoothly can you ensure reliable performance.