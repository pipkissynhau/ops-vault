# 04-CUDA-Installation and Testing

## Documentation Overview

This document aims to provide an overview of fundamental concepts related to CUDA on GPU nodes, including installation methods, version selection, environment variable configuration, verification techniques, containerized testing, and the role in Kubernetes environments. It also covers common troubleshooting for CUDA compatibility issues.

This document addresses the following key questions:

- What is CUDA?
- What are the CUDA Toolkit, CUDA Runtime, and NVIDIA Driver?
- Why does a normal nvidia-smi output not necessarily mean that the CUDA Toolkit is installed?
- Why does a normal nvcc -V result not guarantee that a container or Kubernetes Pod can use the GPU?
- Is it necessary to install the full CUDA Toolkit on the host machine?
- How should the CUDA version be matched with the NVIDIA Driver version?
- How to install the CUDA Toolkit on Ubuntu/Rocky Linux?
- How to test CUDA in a containerized GPU environment?
- Who is responsible for providing CUDA in a Kubernetes GPU Pod?
- How to troubleshoot errors such as "CUDA out of memory" and "driver version is insufficient"?

This document does not focus on the installation of NVIDIA drivers. For driver installation, please refer to:

    03-NVIDIA-Driver Installation and Verification

Nor does it cover Kubernetes Device Plugins, GPU Operators, or GPU Pod scheduling in detail. Relevant information will be provided in subsequent chapters.

---

## Tags

#GPU #CUDA #NVIDIA #Linux #Ubuntu #RockyLinux #Kubernetes #Containerd #Docker #AIInfrastructure #SRE #Troubleshooting

---

## Recommended Reading Path

Recommended reading path:

    06-GPU and AI Infrastructure/01-GPU Basics/04-CUDA-Installation and Testing.md

---

## I. Why Learn CUDA Separately

In GPU operations and maintenance, many people tend to confuse the following components:

    NVIDIA Driver
    CUDA Toolkit
    CUDA Runtime
    cuDNN
    TensorRT
    PyTorch
    TensorFlow
    NVIDIA Container Toolkit
    NVIDIA Device Plugin

Although these components are all related to GPUs, they serve different purposes.

If the concepts are not clear, it can lead to misjudgments during troubleshooting.

For example:

    nvidia-smi works normally, but nvcc -V returns an error message indicating that the command is not available.

This is not necessarily a problem, because nvidia-smi comes from the NVIDIA Driver, while nvcc is part of the CUDA Toolkit.

Another example:

    The nvcc -V command works on the host machine, but torch.cuda.is_available() in a Kubernetes Pod returns False.

This might not be due to the CUDA Toolkit. It could be related to issues with the container image, NVIDIA Container Toolkit, Device Plugin, Pod resource allocation, or driver compatibility.

Another example:

    A container image uses CUDA 12.x Runtime, but the host machine's driver is too old.

In this case, you might encounter an error message like "CUDA driver version is insufficient for CUDA runtime version".

Therefore, CUDA should not be simply understood as a single package to install. Operations and maintenance engineers need to understand the role of CUDA in the entire GPU usage process.

---

## II. The Role of CUDA in the GPU Usage Process

The general process from hardware to practical use of a GPU is as follows:

    GPU Hardware
      ↓
    BIOS / PCIe / NUMA
      ↓
    Linux Kernel
      ↓
    NVIDIA Driver
      ↓
    CUDA Driver API / libcuda
      ↓
    CUDA Runtime / CUDA Toolkit
      ↓
    AI Framework or Business Application
      ↓
    Container Runtime
      ↓
    Kubernetes GPU Pod

CUDA is located above the NVIDIA Driver and below the business application.

When running on a bare machine:

    Application
      ↓
    CUDA Runtime / CUDA Toolkit
      ↓
    NVIDIA Driver
      ↓
    GPU Hardware

When running in a container:

    Container Application
      ↓
    Container CUDA Runtime / AI Framework
      ↓
    Host Machine NVIDIA Driver
      ↓
    NVIDIA Container Toolkit (mounted devices and driver libraries)
      ↓
    GPU Hardware

In Kubernetes:

    GPU Pod
      ↓
    CUDA Runtime / AI Framework in the container image
      ↓
    NVIDIA Container Toolkit / Runtime
      ↓
    Host Machine NVIDIA Driver
      ↓
    Device Plugin assigns GPU resources
      ↓
    GPU Hardware

Key conclusions:

- The host machine must have the NVIDIA Driver installed.
- Container images usually include the CUDA Runtime.
- It is not necessary to install the full CUDA Toolkit on the host machine.
- The CUDA Toolkit is mainly needed when developing and compiling CUDA programs.
- Production-grade inference/training containers focus more on driver and image runtime compatibility.

---

## III. Differences Between CUDA, CUDA Toolkit, CUDA Runtime, and Driver

### 3.1 CUDA

CUDA stands for Compute Unified Device Architecture.

It is a parallel computing platform and programming```bash
python3 -c "import torch; print(torch.cuda.is_available()); print(torch.version.cuda); print(torch.cuda.device_count())"

TensorFlow:

    python3 -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

Note:

Whether a Python framework can use a GPU depends not only on the CUDA Toolkit but also on the framework version, CUDA Runtime, cuDNN, host driver, container runtime, and device mounting.```markdown
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_<version>_all.deb
sudo dpkg -i cuda-keyring_<version>_all.deb
sudo apt-get update

Note:

The <version> above should be based on the NVIDIA official website.
Do not fix a keyring package name that may change in the future in production documents.
Before actually executing, go to the CUDA Downloads page, select the OS, architecture, distribution, and version, and then copy the commands.

### 9.4 Installing the CUDA Toolkit

Example of installing a specific major version:

    sudo apt-get install -y cuda-toolkit-12-2

Or install the default version from the repository:

    sudo apt-get install -y cuda-toolkit

Production recommendations:

    Prefer to install a specific version, such as cuda-toolkit-12-2.
It is not recommended to blindly install the latest default version of cuda-toolkit in a production environment.
The version should match the compatibility matrix of your business framework and drivers.

### 9.5 Configuring Environment Variables

If the installation path is:

    /usr/local/cuda-12.2

You can configure it as follows:

    echo 'export PATH=/usr/local/cuda/bin:$PATH' | sudo tee /etc/profile.d/cuda.sh
    echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' | sudo tee -a /etc/profile.d/cuda.sh

To load the settings:

    source /etc/profile.d/cuda.sh

If /usr/local/cuda is a symlink, check:

    ls -l /usr/local/cuda

### 9.6 Verifying the CUDA Toolkit

    nvcc -V

Check the CUDA installation path:

    which nvcc
    ls -l /usr/local/cuda
    ls -l /usr/local/cuda/bin/nvcc

### 9.7 Verifying Dynamic Libraries

    ldconfig -p | grep cuda

If the libraries are not recognized by the system, you can check:

    cat /etc/ld.so.conf.d/*cuda*
    sudo ldconfig
---

## Section 10: Installing the CUDA Toolkit on Rocky Linux 9

Rocky Linux 9/RHEL 9 is commonly used in enterprise server environments.

### 10.1Installing Basic Dependencies

    sudo dnf install -y wget curl ca-certificates
    sudo dnf install -y gcc make dkms kernel-devel-$(uname -r) kernel-headers-$(uname -r)
    sudo dnf install -y pciutils

### 10.2 Ensuring Drivers Are Working Properly

    nvidia-smi

If nvidia-smi does not work correctly, repair the NVIDIA Driver first.

### 10.3 Configuring the NVIDIA CUDA Repository

The actual repository should be based on the NVIDIA official CUDA Downloads page.

Common format:

    sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo
    sudo dnf clean all
    sudo dnf makecache

### 10.4 Installing the CUDA Toolkit

Example of installing a specific version:

    sudo dnf install -y cuda-toolkit-12-2

Or:

    sudo dnf install -y cuda-toolkit

Production recommendations:

    In Rocky/RHEL environments, it is preferable to use internal enterprise yum/dnf repositories.
In uncontrolled public network environments, it is not recommended to rely on the public NVIDIA repository for production nodes.
The version should be fixed to avoid discrepancies between different GPU nodes.

### 10.5 Configuring Environment Variables

    echo 'export PATH=/usr/local/cuda/bin:$PATH' | sudo tee /etc/profile.d/cuda.sh
    echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' | sudo tee -a /etc/profile.d/cuda.sh

To load the settings:

    source /etc/profile.d/cuda.sh

Verification:

    nvcc -V
    which nvcc
    ls -l /usr/local/cuda

---

## Section 11: Installing the CUDA Toolkit Using a Runfile

The runfile method is suitable for offline or special environments but is not recommended as the first choice for production use.

### 11.1 Downloading the Runfile

Select from the NVIDIA CUDA Downloads page:

    Operating System: Linux
    Architecture: x86_64
    Distribution: Ubuntu/RHEL/Rocky
    Version: The corresponding system version
    Installer Type: runfile

The downloaded file will be something like:

    cuda_<version>_linux.run

### 11.2 Executing the Installation

    chmod +x cuda_<version>_linux.run
    sudoResult = PASS

deviceQuery can verify the following:

- Whether the CUDA Runtime can access the GPU;
- GPU device properties;
- CUDA Capability;
- Video memory;
- Number of multi-processors;
- Maximum number of threads, and other information.

### 13.2 bandwidthTest Test

Enter the directory:

    cd /usr/local/cuda/samples/1_Utilities/bandwidthTest

Compile:

    make

Run:

    ./bandwidthTest

Upon success, you will usually see:

Result = PASS

bandwidthTest can preliminarily verify:

- Host to Device bandwidth;
- Device to Host bandwidth;
- Device to Device bandwidth.

Note:

    bandwidthTest is not a comprehensive performance test.
    It can only be used as a basic verification of connectivity and bandwidth.

### 13.3 Troubleshooting Compilation Failures

If make fails, common reasons include:

- gcc / make are not installed;
- The CUDA Toolkit is incomplete;
- PATH is not configured correctly;
- nvcc does not exist;
- Incompatible gcc version;
- The samples directory does not exist;
- Insufficient permissions.

Troubleshoot by checking:

    which nvcc
    nvcc -V
    gcc --version
    make --version
    echo $PATH
    echo $LD_LIBRARY_PATH

---

## Chapter Fourteen: Testing CUDA Using Containers

In a production Kubernetes environment, it is recommended to test the CUDA Runtime using containers.

Prerequisites:

- The host machine's nvidia-smi is functioning correctly;
- The NVIDIA Container Toolkit has been installed;
- Docker or containerd is configured properly;
- The CUDA Runtime container image is compatible with the host machine's drivers.

### 14.1 Testing nvidia-smi with Docker

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

If successful, it indicates that:

- The host machine's drivers are available;
- Docker's GPU parameters are functional;
- The NVIDIA Container Toolkit is usable;
- The GPU can be accessed within the container.

### 14.2 Testing the CUDA Runtime with Docker

You can enter the container using:

    docker run --rm -it --gpus all nvidia/cuda:12.2.0-devel-ubuntu22.04 bash

Inside the container, execute:

    nvidia-smi
    nvcc -V

Note:

- The base image may not include nvcc.
- The runtime image typically does not contain full development tools.
- The devel image usually includes nvcc and development headers.

### 14.3 Types of nvidia/cuda Images

Common image types include:

    base
    runtime
    devel

Differences:

    base:
        Provides the minimum foundation for running CUDA, suitable for simple environments.

    runtime:
        Includes libraries necessary for running CUDA applications, ideal for production use.

    devel:
        Contains development and compilation tools, such as nvcc, suitable for building and compiling.

Production recommendations:

    Use the runtime image for running services.
    Use the devel image for building images.
    It is not recommended to use the devel image for production services without a specific need, as it results in larger image sizes and increased security risks.

---

## Chapter Fifteen: Testing CUDA in Kubernetes

In Kubernetes, CUDA typically comes from container images rather than the host machine's CUDA Toolkit.

### 15.1 Prerequisites

Before a GPU Pod can run successfully, the following must be in place:

- The host machine's NVIDIA Driver is functioning correctly;
- The NVIDIA Container Toolkit is operational;
- The Device Plugin or GPU Operator is working properly;
- kubelet has registered with nvidia.com/gpu;
- The Pod has correctly requested a GPU;
- The CUDA Runtime in the image is compatible with the drivers.

### 15.2 Checking Node GPU Resources

    kubectl describe node <gpu-node-name>

Focus on:

    Capacity:
      nvidia.com/gpu: 1

    Allocatable:
      nvidia.com/gpu: 1

If "nvidia.com/gpu" is not listed, it means Kubernetes has not yet recognized the GPU resources.

### 15.3 Creating a CUDA Test Pod

Example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: cuda-test
      namespace: default
    spec:
      restartPolicy: Never
      containers:
        - name: cuda
          image: nvidia/cuda:12.2.0-base-ubuntu22.04
          command: ["nvidia-smi"]
          resources:
            limits:
              nvidia.com/gpu: 1

Save the file as:

    cuda-test.yaml

Deploy it using:

    kubectl apply -f cuda-test.yaml

View the Pod details:

    kubectl get pod cuda-test -o wide
    kubectl logs cuda-test
   company/pytorch:2.x-cuda12.2-cudnn-runtime

TensorRT Image:

company/tensorrt:cuda12.2-runtime

Basic Image for Inference Services:

company/inference-base:cuda12.2-runtime

### 17.3 Recommendations for Image Selection

For running services:

- Prefer the runtime image.

For building programs:

- Use the devel image.

For debugging environments:

- You can temporarily use the devel image.

For production environments:

- Avoid using large devel images to run services directly.
- Avoid using the latest version of images.
- Avoid using images with unknown origins.
- Avoid maintaining messy CUDA base images for each project separately.

---

## Chapter Eighteen: The Relationship Between CUDA and Kubernetes Scheduling

CUDA itself is not responsible for Kubernetes scheduling.

Kubernetes relies on the following components to schedule GPUs:

- NVIDIA Driver;
- NVIDIA Container Toolkit;
- NVIDIA Device Plugin;
- kubelet resource registration;
- Pod resources.limits;
- Scheduler.

CUDA provides the runtime environment for containers or applications to use GPUs.

### 18.1 Pod Application for GPU

Example of a GPU Pod configuration:

resources:
  limits:
    nvidia.com/gpu: 1

This indicates that the Pod is requesting 1 GPU.

Note:

- GPUs are considered extended resources.
- GPUs usually cannot be configured with limits like CPU resources (e.g., setting them to 500m).
- nvidia.com/gpu: 1 means one full GPU is requested.
- Solutions for multi-GPU sharing or MIG need to be designed separately.

### 18.2 CUDA_VISIBLE_DEVICES

A common environment variable in containers:

CUDAVISIBLE_devices

This variable controls which GPUs are visible to the application.

In a Kubernetes + Device Plugin setup, the Device Plugin typically sets the visible GPUs for the container.

To check:

- echo $CUDA_VISIBLE Devices
- Execute nvidia-smi inside the container

You may only see the GPU assigned to that specific container.

### 18.3 Multi-GPU Pods

To request multiple GPUs:

resources:
  limits:
    nvidia.com/gpu: 2

Business applications that require multiple GPUs should use appropriate frameworks, such as:

- PyTorch DistributedDataParallel;
- TensorFlow MirroredStrategy;
- NCCL;
- MPI;
- Horovod;
- DeepSpeed;
- Megatron-LM.

Operations and maintenance personnel need to pay attention to the following:

- Ensure the Pod has actually been allocated multiple GPUs.
- Verify if the GPUs are on the same node.
- Check if the GPU topology is appropriate.
- Confirm that NCCL communication is functioning correctly.
- Ensure even distribution of video memory among GPUs.
- Verify if the multi-GPU utilization rate is balanced.

---

## Chapter Nineteen: Explanation of Common CUDA Directories

### 19.1 /usr/local/cuda

This is usually a symbolic link to the current default CUDA version.

To check:

- ls -l /usr/local/cuda

### 19.2 /usr/local/cuda/bin

Common commands included in this directory:

- nvcc
- cuda-gdb
- nsight-related tools

To list available commands:

- ls /usr/local/cuda/bin

### 19.3 /usr/local/cuda/lib64

This is the CUDA library directory.

To check:

- ls /usr/local/cuda/lib64

### 19.4 /usr/local/cuda/include

This contains CUDA header files.

It is essential for developing and compiling CUDA programs.

### 19.5 /usr/local/cuda/samples

This directory contains sample CUDA code.

However, not all installation methods include the samples by default.

---

## Chapter Twenty: Complete Verification Process After CUDA Installation

### 20.1 Driver Layer Verification

- Execute nvidia-smi
- Run nvidia-smi -L to check version details
- Use nvidia-smi -q to view quick information
- Check lsmod | grep nvidia for driver installation
- List devices connected via /dev/nvidia*

### 20.2 CUDA Toolkit Verification

- Verify the nvcc command and its version with nvcc -V
- List installed CUDA tools in /usr/local/cuda
- Check $PATH and $LD_LIBRARY_PATH to ensure they include CUDA paths

### 20.3 Dynamic Library Verification

- Use ldconfig -p | grep cuda and cudart to confirm dynamic libraries are present

### 20.4 CUDA Samples Verification

- Navigate to /usr/local/cuda/samples/1_Utilities/deviceQuery
- Run make to compile the samples
- Execute ./deviceQuery to test them

Expected result:

Result = PASS

### 20.5 Container Verification

- Use docker run with --rm and --gpus all to run a basic CUDA test:
  docker run --rm --gpus all nvidia/cuda:12.2.0-base- The Pod requested a GPU, but multiple processes within the service competed for the same card;
- The MIG/shareable GPU allocation was too small.

### 24.3 Troubleshooting

Check video memory:

    nvidia-smi

Check processes:

    nvidia-smi

Check host machine processes:

    ps -ef | grep <PID>

Check Kubernetes Pod:

    kubectl get pod -A -o wide | grep <node-name>
    crictl ps
    crictl inspect <container-id>

### 24.4 Solutions

Possible solutions include:

- Reducing the batch size;
- Using smaller models;
- Switching to FP16/BF16;
- Implementing model quantization;
- Terminating abnormal processes;
- Limiting concurrent tasks per node;
- Splitting the model;
- Using GPUs with more video memory;
- Properly dividing resources using MIG;
- Optimizing application code to free up video memory.

Note:

    In Kubernetes, nvidia.com/gpu only indicates the number of GPUs requested.
    It does not automatically limit the amount of video memory available within processes.
    Managing video memory requires a comprehensive approach that takes into account applications, frameworks, MIG settings, and platform policies.

---

## Chapter Twenty-Five: Common Fault 5: PyTorch Fails to Detect a GPU

### 25.1 Symptoms

When executing:

    python3 -c "import torch; print(torch.cuda.is_available())"

The output is:

    False

### 25.2 Possible Causes

Possible reasons include:

- The installed version of PyTorch is the CPU edition;
- The CUDA version of PyTorch does not match the environment;
- The host machine driver does not support the CUDA Runtime image;
- The container is not configured to use a GPU;
- The Kubernetes Pod has not requested any GPUs through nvidia.com/gpu;
- The NVIDIA Container Toolkit is not properly configured;
- There are issues with the Device Plugin;
- The CUDA_VISIBLE_devices environment variable is set incorrectly.

### 25.3 Troubleshooting

Check PyTorch version:

    python3 -c "import torch; print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available()); print(torch.cuda.device_count())"

Check GPU status:

    nvidia-smi

Verify environment variables:

    echo $CUDA_VISIBLE_devices

For Kubernetes:

    kubectl describe pod <pod-name> -n <namespace>
    kubectl describe node <gpu-node-name>

### 25.4 Solutions

Possible solutions include:

- Installing the CUDA version of PyTorch;
- Using an official Docker image that includes the correct CUDA version;
- Ensuring the Pod has requested a GPU;
- Repairing any issues with the NVIDIA Container Toolkit or Device Plugin;
- Changing to an image that supports the correct CUDA Runtime;
- Upgrading the host machine driver.

---

## Chapter Twenty-Six: Common Fault 6: nvidia-smi Fails within a Container

### 26.1 Symptoms

On the host machine:

    nvidia-smi

Works normally.

Within the container:

    nvidia-smi

Fails to execute.

### 26.2 Possible Causes

Possible reasons include:

- The NVIDIA Container Toolkit is not installed;
- Docker has not been configured to use --gpus all;
- containerd is not set up to use the NVIDIA runtime;
- The Kubernetes Device Plugin is not running;
- The Pod has not requested any GPUs through nvidia.com/gpu;
- The container image does not contain the nvidia-smi tool;
- The necessary driver libraries are not mounted;
- There are configuration errors with the container runtime.

### 26.3 Docker Troubleshooting

Try running:

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

If this fails, check:

    nvidia-container-cli info
    docker info | grep -i runtime

### 26.4 Kubernetes Troubleshooting

Check:

    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia
    kubectl describe node <gpu-node-name>
    kubectl describe pod <gpu-pod-name> -n <namespace>

Ensure that the Pod has configured:

    resources:
      limits:
        nvidia.com/gpu: 1

---

## Chapter Twenty-Seven: Common Fault 7: deviceQuery Compilation Fails

### 27.1 Symptoms

When attempting to compile:

    make

A failure occurs.

### 27.2 Possible Causes

Possible reasons include:

- The make tool is not installed;
- The gcc/g++ tools are missing;
- The nvcc binary is not present;
- The CUDA Toolkit is not fully installed;
- The PATH environment variable is not set correctly;
- The version of gcc is---

## Section 31: CUDA Management Recommendations for Production Environments

### 31.1 Establishing a Version Matrix

It is recommended to record the following:

    NVIDIA Driver Version
    CUDA Runtime Version
    CUDA Toolkit Version
    cuDNN Version
    TensorRT Version
    PyTorch Version
    TensorFlow Version
    GPU Model
    OS Version
    Kernel Version
    Container Runtime Version
    Device Plugin / GPU Operator Version

Example:

    GPU Model: NVIDIA A100 80GB
    OS: Ubuntu 22.04
    Driver: 550.x
    CUDA Runtime: 12.2
    PyTorch: 2.x
    cuDNN: 8.x
    Container Runtime: containerd
    Kubernetes: 1.xx
    Device Plugin: v0.xx

### 31.2 Avoid Making the Host Machine Handle Business Dependencies

It is recommended that the host machine only maintain:

- NVIDIA Driver;
- NVIDIA Container Toolkit;
- containerd / Docker;
- kubelet;
- Device Plugin / GPU Operator;
- Monitoring Agent;
- Logging Agent.

Business dependencies should be placed in the image:

- CUDA Runtime;
- cuDNN;
- TensorRT;
- Python;
- PyTorch;
- TensorFlow;
- Business Code.

### 31.3 Fixing Image Versions

It is not recommended to use the "latest" version in production:

Instead, it is suggested to use specific versions such as:

    nvidia/cuda:12.2.0-runtime-ubuntu22.04
    company/pytorch:2.x-cuda12.2-cudnn-runtime

### 31.4 Establishing a Base Image Repository

It is recommended that enterprises maintain internally:

- CUDA base images;
- CUDA runtime images;
- CUDA devel images;
- PyTorch images;
- TensorFlow images;
- TensorRT images;
- Basic images for inference services.

These should be synchronized to internal image repositories, such as:

    Harbor
    Nexus
    Alibaba Cloud Image Repository
    Private Registry

### 31.5 Controlling the Pace of CUDA Updates

Before upgrading CUDA, it is necessary to verify:

- Driver compatibility;
- Framework compatibility;
- Compatibility with business images;
- Model loading;
- Inference performance;
- Training performance;
- Memory usage;
- Kubernetes GPU Pods;
- Monitoring metrics;
- Backup plans.

---

## Section 32: CUDA Installation and Test Checklist

### 32.1 Before Installation

    [ ] The GPU hardware has been recognized.
    [ ] The NVIDIA GPU is visible in lspci.
    [ ] The NVIDIA Driver has been installed.
    [ ] nvidia-smi is functioning normally.
    [ ] The Driver version has been recorded.
    [ ] The required CUDA version has been determined.
    [ ] The AI framework version has been confirmed.
    [ ] It has been determined whether the host machine needs the CUDA Toolkit.
    [ ] The installation method has been chosen.
    [ ] The old CUDA environment has been checked.
    [ ] PATH and LD_LIBRARY_PATH have been verified.

### 32.2 During Installation

    [ ] The NVIDIA CUDA Repository has been configured.
    [ ] apt/dnf caches have been updated.
    [ ] The CUDA Toolkit has been successfully installed.
    [ ] It has not overwritten any existing stable NVIDIA Driver.
    [ ] Environment variables have been set correctly.
    [ ] ldconfig has been executed or library paths have been confirmed.

### 32.3 After Installation

    [ ] nvcc -V returns a normal result.
    [ ] The path to nvcc is correct.
    [ ] The /usr/local/cuda symlink is correct.
    [ ] nvidia-smi is functioning normally.
    [ ] deviceQuery tests pass.
    [ ] bandwidthTest tests pass.
    [ ] Docker CUDA image tests pass.
    [ ] Kubernetes CUDA Pod tests pass.
    [ ] PyTorch / TensorFlow tests pass.
    [ ] There are no compatibility issues between CUDA Runtime and Driver.

---

## Section 33: Understanding CUDA Issues at Different Levels

When troubleshooting CUDA problems, do not immediately attempt to reinstall CUDA. Instead, approach the issue at different levels.

### 33.1 If lspci Reports Abnormalities

The problem may lie at the hardware or BIOS level.

Check:

    - Is the GPU properly plugged in?
    - Does the BIOS recognize it?
    - Are there any issues with Above 4G Decoding?
    - PCIe slots and risers are secure.
    - Power supply is adequate.

### 33.2 If nvidia-smi Reports Abnormalities

The problem may be with the driver layer.

Check:

    - NVIDIA Driver
    - nouveau
    - Secure Boot settings
    - DKMS configuration
    - Kernel headers
    - XID issues
    - PCIe errors

- NVIDIA CUDA Toolkit Documentation:
  https://docs.nvidia.com/cuda/

- NVIDIA CUDA Compatibility:
  https://docs.nvidia.com/deploy/cuda-compatibility/

- NVIDIA CUDA Downloads:
  https://developer.nvidia.com/cuda-downloads

- NVIDIA CUDA Container Images:
  https://hub.docker.com/r/nvidia/cuda

- NVIDIA NGC CUDA Container:
  https://catalog.ngc.nvidia.com/orgs/nvidia/containers/cuda

- NVIDIA Container Toolkit:
  https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

- NVIDIA GPU Operator:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/

- Kubernetes GPU Scheduling:
  https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/