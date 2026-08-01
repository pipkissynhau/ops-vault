# 04-CUDA-Installation and Testing

## Document Overview

This document is used to organize basic concepts of CUDA, installation methods, version selection, environment variable configuration, verification methods, containerized testing, boundary relationships in Kubernetes scenarios, and common CUDA compatibility troubleshooting.

This document focuses on answering the following questions:

- What is CUDA?
- What are CUDA Toolkit, CUDA Runtime, and NVIDIA Driver?
- Why does nvidia-smi being normal not necessarily mean CUDA Toolkit is installed?
- Why does nvcc -V being normal not necessarily mean a container or Kubernetes Pod can use GPU?
- Does the host machine really need to install the full CUDA Toolkit?
- How to match CUDA version and NVIDIA Driver version?
- How to install CUDA Toolkit on Ubuntu/Rocky Linux?
- How to test CUDA in containerized GPU scenarios?
- Who provides CUDA in Kubernetes GPU Pods?
- How to troubleshoot errors like CUDA out of memory and driver version is insufficient?

This document does not focus on NVIDIA driver installation. Driver installation should refer to:

    03-NVIDIA-Driver installation and authentication

This document also does not focus on Kubernetes Device Plugin, GPU Operator, or GPU Pod scheduling. Related content is placed in subsequent chapters.

---

## Tags

#GPU #CUDA #NVIDIA #Linux #Ubuntu #RockyLinux #Kubernetes #Containerd #Docker #AiInfrastructure #SRE #TransportBarriers

---

## Recommended Path

Recommended path:

    06-GPUandAIInfrastructure/01-GPUBasis/04-CUDA-Installation and testing.md

---

## One, Why Learn CUDA Separately

In GPU operations, many people mix the following components:

    NVIDIA Driver
    CUDA Toolkit
    CUDA Runtime
    cuDNN
    TensorRT
    PyTorch
    TensorFlow
    NVIDIA Container Toolkit
    NVIDIA Device Plugin

Although these components are all related to GPU, their positions are completely different.

If concepts are unclear, troubleshooting can easily lead to misjudgment.

For example:

    nvidia-smi is normal, but nvcc -V does not exist

This is not necessarily a fault, because nvidia-smi comes from NVIDIA Driver, while nvcc comes from CUDA Toolkit.

Another example:

    Host nvcc -V is normal, but torch.cuda.is_available() is False in Kubernetes Pod

This is not necessarily a CUDA Toolkit issue, it could be container image, NVIDIA Container Toolkit, Device Plugin, Pod resource allocation, or driver compatibility issues.

Another example:

    Container image uses CUDA 12.x Runtime, but host driver is too old

This may result in:

    CUDA driver version is insufficient for CUDA runtime version

So CUDA cannot be understood as just "installing a package".

Operations engineers need to understand CUDA's position in the entire GPU usage chain.

---

## Two, CUDA's Position in the GPU Usage Chain

From GPU hardware to business availability, the general chain is as follows:

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
    AI Framework or Business Program
      ↓
    Container Runtime
      ↓
    Kubernetes GPU Pod

CUDA is located above NVIDIA Driver and below business programs.

In bare-metal operation:

    Application
      ↓
    CUDA Runtime / CUDA Toolkit
      ↓
    NVIDIA Driver
      ↓
    GPU Hardware

In container operation:

    Application inside container
      ↓
    CUDA Runtime / AI Framework inside container image
      ↓
    Host NVIDIA Driver
      ↓
    NVIDIA Container Toolkit mounts devices and driver libraries
      ↓
    GPU Hardware

In Kubernetes:

    GPU Pod
      ↓
    CUDA Runtime / AI Framework in container image
      ↓
    NVIDIA Container Toolkit / Runtime
      ↓
    Host NVIDIA Driver
      ↓
    Device Plugin allocates GPU devices
      ↓
    GPU Hardware

Key conclusions:

    The host machine must have NVIDIA Driver.
    Container images typically include CUDA Runtime by default.
    The host machine does not necessarily need to install the full CUDA Toolkit.
    CUDA Toolkit is more needed for development and compilation of CUDA programs.
    Production inference/training containers usually focus more on driver and image Runtime compatibility.

---

## Three, Differences Between CUDA, CUDA Toolkit, CUDA Runtime, and Driver

### 3.1 CUDA

CUDA, full name Compute Unified Device Architecture.

It is a parallel computing platform and programming model provided by NVIDIA, used to let programs use NVIDIA GPU for general computing.

CUDA is commonly used for:

- AI training;
- AI inference;
- Image processing;
- Video processing;
- Scientific computing;
- High-performance computing;
- Matrix computing;
- Tensor computing.

### 3.2 NVIDIA Driver

NVIDIA Driver is the host driver.

It is responsible for allowing Linux kernel and user-space programs to access GPU.

Verification command:

    nvidia-smi

If nvidia-smi is not working properly, CUDA can basically not be used normally.

### 3.3 CUDA Toolkit

CUDA Toolkit is a development toolkit.

It typically includes:

- nvcc compiler;
- CUDA header files;
- CUDA example code;
- CUDA libraries;
- CUDA debugging tools;
- CUDA performance analysis tools;
- Some runtime components.

Verification command:

    nvcc -V

Note:

    nvcc comes from CUDA Toolkit.
    nvidia-smi being normal does not mean nvcc is definitely present.
    nvcc not existing does not mean NVIDIA Driver is abnormal.

### 3.4 CUDA Runtime

CUDA Runtime is the runtime library required to run CUDA programs.

In containerized scenarios, CUDA Runtime is typically included in container images.

Example:

    nvidia/cuda:12.2.0-base-ubuntu22.04
    nvidia/cuda:12.2.0-runtime-ubuntu22.04
    nvidia/cuda:12.2.0-devel-ubuntu22.04
    pytorch/pytorch:xxx-cuda12.x-cudnnx-runtime

### 3.5 cuDNN

cuDNN is NVIDIA's acceleration library for deep neural networks.

Commonly used in:

- PyTorch;
- TensorFlow;
- CNN;
- RNN;
- Transformer;
- Deep learning training and inference.

cuDNN is not equal to CUDA.

It is a deep learning library built on top of CUDA.

### 3.6 TensorRT

TensorRT is NVIDIA's inference optimization and deployment framework.

Commonly used in:

- Model inference acceleration;
- FP16 / INT8 optimization;
- Model graph optimization;
- Low-latency inference services;
- High-throughput inference services.

TensorRT also depends on CUDA and NVIDIA Driver.

---

## Four, Several Easily Confused Commands

### 4.1 nvidia-smi

Command:

    nvidia-smi

Source:

    NVIDIA Driver

Function:

- View GPU;
- View driver version;
- View CUDA Version supported by driver;
- View memory;
- View temperature;
- View power consumption;
- View GPU utilization;
- View process usage.

Note:

    The CUDA Version displayed in nvidia-smi does not mean the system has installed a complete CUDA Toolkit.
    It indicates the highest CUDA Runtime capability range supported by the current driver.

### 4.2 nvcc -V

Command:

    nvcc -V

Source:

    CUDA Toolkit

Function:

- View CUDA compiler version;
- Verify if CUDA Toolkit is installed;
- Used for compiling CUDA examples and custom CUDA programs.

Note:

    nvcc not existing does not mean GPU cannot be used by containers.
    Production Kubernetes GPU nodes can lack nvcc.
    Development machines or nodes needing to compile CUDA programs would need nvcc more.

### 4.3 ldconfig -p | grep cuda

Command:

    ldconfig -p | grep cuda

Function:

- Check if CUDA-related libraries exist in system dynamic libraries;
- Assist in checking if libcuda.so, libcudart.so, etc., libraries exist.

### 4.4 Python framework check

PyTorch:

    python3 -c "import torch; print(torch.cuda.is_available()); print(torch.version.cuda); print(torch.cuda.device_count())"

TensorFlow:

    python3 -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"

Note:

    Whether Python frameworks can use GPU depends not only on CUDA Toolkit.
    It also depends on framework version, CUDA Runtime, cuDNN, host driver, container runtime, and device mounting.

---

## Five, Should the Host Install CUDA Toolkit

This is a critical question in GPU operations.

### 5.1 Bare-metal Development Environment

If it's a development machine requiring CUDA program compilation, it's recommended to install CUDA Toolkit.

Typical scenarios:

- Compiling CUDA C/C++ programs;
- Using nvcc;
- Compiling CUDA samples;
- Doing CUDA development and debugging;
- Local development of AI operators;
- Building source code projects dependent on CUDA.

Such environments require:

    NVIDIA Driver
    CUDA Toolkit
    nvcc
    CUDA samples
    Relevant development libraries

### 5.2 Production Inference Nodes

If it's a Kubernetes GPU inference node, the host typically does not need a complete CUDA Toolkit.

More commonly:

    Host:
        NVIDIA Driver
        NVIDIA Container Toolkit
        containerd / Docker
        kubelet
        Device Plugin / GPU Operator

    Container image:
        CUDA Runtime
        cuDNN
        TensorRT
        PyTorch / TensorFlow
        Inference service code

This approach is more conducive to:

- Standardized images;
- Isolated business environments;
- Avoiding host pollution;
- Different businesses using different CUDA Runtimes;
- Keeping the host only responsible for drivers and scheduling.

### 5.3 Production Training Nodes

Training nodes are also typically recommended to be containerized.

The host should focus on maintaining:

    Stable driver
    Stable container runtime
    Stable device plugin
    Stable DCGM Exporter monitoring

Training dependencies should be placed inside the image.

### 5.4 Operation Recommendations

Recommended principles:

    If it's just a Kubernetes GPU Worker:
        The host can avoid installing a complete CUDA Toolkit.

    If local CUDA compilation and debugging are needed on the node:
        Install CUDA Toolkit.

    If only verifying if GPU can run CUDA:
        Prioritize using official CUDA containers for testing to avoid polluting the host.

    For production environments:
        Do not let the host install a lot of business dependencies.
        Try to manage business runtime environments through standard CUDA base images.

---

## Six, CUDA Version Selection Principles

CUDA version cannot be selected arbitrarily.

It needs to consider:

- NVIDIA Driver version;
- GPU model;
- Operating system version;
- AI framework version;
- PyTorch / TensorFlow supported CUDA version;
- cuDNN version;
- TensorRT version;
- Business image version;
- Kubernetes GPU node uniformity;
- Whether new architecture GPU support is needed;
- Whether there are enterprise security compliance requirements.

### 6.1 Do not blindly install the latest CUDA

It is not recommended to install the latest CUDA in production environments.

Reasons:

- AI frameworks may not be fully compatible yet;
- Business images may depend on older versions;
- Driver versions may not meet requirements;
- New versions may introduce compatibility issues;
- Inconsistent versions across nodes will increase troubleshooting costs;
- CUDA Toolkit upgrades do not guarantee improved business performance.

### 6.2 Prioritize deriving CUDA version from business framework

In actual production, the common order is:

    1. Confirm the PyTorch / TensorFlow / TensorRT version used by the business
    2. Confirm which CUDA versions the framework supports
    3. Select the corresponding CUDA Runtime image
    4. Confirm the host NVIDIA Driver supports the CUDA Runtime
    5. Standardize GPU node driver versions
    6. Finalize the base image

Do not reverse this:

    Install CUDA randomly first
    Then force the business to adapt

### 6.3 Driver and CUDA Runtime compatibility

Basic principles:

    The host NVIDIA Driver must support the CUDA Runtime required by the container or application.

If the driver is too old, common errors include:

    CUDA driver version is insufficient for CUDA runtime version

Or:

    RuntimeError: CUDA error

Or:

    torch.cuda.is_available() returns False

Production recommendations:

    Establish a Driver / CUDA / Framework version matrix.
    Validate on test nodes before each upgrade.

---

## Seven, CUDA Installation Method Selection

Common CUDA Toolkit installation methods:

    1. Package manager installation
    2. NVIDIA CUDA Repository installation
    3. runfile installation
    4. Container image usage of CUDA Runtime

### 7.1 Package manager / official repository installation

Suitable for:

- Need to install CUDA Toolkit on the host;
- Want system maintainability;
- Want to manage versions via apt/dnf;
- Batch node uniform deployment;
- Enterprise internal mirror source management.

Advantages:

- Clear upgrade and uninstallation;
- Suitable for automation;
- Dependency management is possible;
- Suitable for production standardization.

### 7.2 runfile installation

Suitable for:

- Offline environments;
- Special versions;
- Experimental environments;
- Scenarios where repositories cannot be used.

Disadvantages:

- Not conducive to package management;
- Upgrade and uninstallation are troublesome;
- Easy to conflict with driver installations from package managers;
- High maintenance cost in production environments.

### 7.3 Container image method

Suitable for:

- Kubernetes GPU Pod;
- Containerized training;
- Containerized inference;
- Multiple businesses sharing GPU nodes;
- Environments needing isolation of different CUDA Runtimes.

Example:

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

This method does not require the host to install a complete CUDA Toolkit.

---

## Eight, Pre-installation Checks for CUDA

Before installing CUDA Toolkit, confirm the driver layer is normal.

### 8.1 Check GPU hardware

    lspci | grep -i nvidia

If no GPU is visible here, do not continue installing CUDA; troubleshoot hardware and BIOS first.

### 8.2 Check NVIDIA Driver

    nvidia-smi

If nvidia-smi is not functioning properly, do not continue installing CUDA; troubleshoot the driver first.

### 8.3 Check Driver Version

    nvidia-smi

Record:

    Driver Version
    CUDA Version
    GPU model
    GPU count

Note:

    The CUDA Version displayed by nvidia-smi is the driver's supported capability, not the CUDA Toolkit installation version.

### 8.4 Check system version

Ubuntu:

    cat /etc/os-release
    uname -r

Rocky Linux:

    cat /etc/os-release
    uname -r

### 8.5 Check old CUDA

    which nvcc
    nvcc -V

Check installed packages:

Ubuntu:

    dpkg -l | grep -i cuda
    dpkg -l | grep -i nvidia

Rocky Linux:

    rpm -qa | grep -i cuda
    rpm -qa | grep -i nvidia

### 8.6 Check environment variables

    echo $PATH
    echo $LD_LIBRARY_PATH

If CUDA paths were manually configured before, confirm whether it will affect the new version.

Common paths:

    /usr/local/cuda
    /usr/local/cuda-12.2
    /usr/local/cuda-12.4

---

## Nine, Ubuntu Installation of CUDA Toolkit

The following uses Ubuntu Server 22.04 / 24.04 as an example.

The actual version should be based on the command generated by NVIDIA's official CUDA Downloads page.

### 9.1 Install basic dependencies

    sudo apt-get update
    sudo apt-get install -y wget curl gnupg ca-certificates
    sudo apt-get install -y build-essential dkms linux-headers-$(uname -r)
    sudo apt-get install -y pciutils

### 9.2 Confirm driver is normal

    nvidia-smi

If this fails, do not continue installing CUDA Toolkit.

### 9.3 Configure NVIDIA CUDA Repository

The actual command should be generated based on the target CUDA version and Ubuntu version from NVIDIA's CUDA Downloads page.

Common process: /think

1. Download the cuda-keyring package
    2. Install cuda-keyring
    3. apt-get update
    4. Install cuda-toolkit-specified version

Example format:

    wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_<版本>_all.deb
    sudo dpkg -i cuda-keyring_<version>_all.deb
    sudo apt-get update

Note:

    The <version> above must be based on NVIDIA's official website.
    Do not fix a keyring package name that may change in the future in production documentation.
    Before actual execution, select OS, architecture, distribution, and version on the CUDA Downloads page and copy the command.

### 9.4 Installing CUDA Toolkit

Example for installing a specific major version:

    sudo apt-get install -y cuda-toolkit-12-2

Or install the default version from the repository:

    sudo apt-get install -y cuda-toolkit

Production recommendation:

    Prioritize installing a specific version, such as cuda-toolkit-12-2.
    It is not recommended to install the default latest version of cuda-toolkit in production environments.
    The version should be consistent with the business framework and compatibility matrix of the driver.

### 9.5 Configuring Environment Variables

If the installation path is:

    /usr/local/cuda-12.2

You can configure:

    echo 'export PATH=/usr/local/cuda/bin:$PATH' | sudo tee /etc/profile.d/cuda.sh
    echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' | sudo tee -a /etc/profile.d/cuda.sh

Load:

    source /etc/profile.d/cuda.sh

If /usr/local/cuda is a symlink, check:

    ls -l /usr/local/cuda

### 9.6 Verifying CUDA Toolkit

    nvcc -V

Check the CUDA installation path:

    which nvcc
    ls -l /usr/local/cuda
    ls -l /usr/local/cuda/bin/nvcc

### 9.7 Verifying Dynamic Libraries

    ldconfig -p | grep cuda

If the library is not recognized by the system, you can check:

    cat /etc/ld.so.conf.d/*cuda*
    sudo ldconfig

---

## TenI don't know.Rocky Linux 9 Installation of CUDA Toolkit

Rocky Linux 9 / RHEL 9 scenarios are typically used in enterprise server environments.

### 10.1 Installing Basic Dependencies

    sudo dnf install -y wget curl ca-certificates
    sudo dnf install -y gcc make dkms kernel-devel-$(uname -r) kernel-headers-$(uname -r)
    sudo dnf install -y pciutils

### 10.2 Confirming Driver Normality

    nvidia-smi

If nvidia-smi is not functioning properly, first fix the NVIDIA Driver.

### 10.3 Configuring NVIDIA CUDA Repository

The actual repository should be based on NVIDIA's official CUDA Downloads page.

Common format:

    sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo
    sudo dnf clean all
    sudo dnf makecache

### 10.4 Installing CUDA Toolkit

Example for installing a specific version:

    sudo dnf install -y cuda-toolkit-12-2

Or:

    sudo dnf install -y cuda-toolkit

Production recommendation:

    In Rocky / RHEL environments, it is recommended to use enterprise internal yum/dnf repositories.
    In environments where external networks are uncontrollable, it is not recommended to rely on public NVIDIA repo in production nodes.
    The version should be fixed to avoid drift between different GPU nodes.

### 10.5 Configuring Environment Variables

    echo 'export PATH=/usr/local/cuda/bin:$PATH' | sudo tee /etc/profile.d/cuda.sh
    echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' | sudo tee -a /etc/profile.d/cuda.sh

Load:

    source /etc/profile.d/cuda.sh

Verification:

    nvcc -V
    which nvcc
    ls -l /usr/local/cuda

---

## ElevenI don't know.runfile Method Installation of CUDA Toolkit

The runfile method is suitable for offline or special environments, but it is not recommended as the preferred choice for production environments.

### 11.1 Downloading runfile

Select from the NVIDIA CUDA Downloads page:

    Operating System: Linux
    Architecture: x86_64
    Distribution: Ubuntu / RHEL / Rocky
    Version: Corresponding system version
    Installer Type: runfile

After downloading, it will be similar to:

    cuda_<version>_linux.run

### 11.2 Executing Installation

    chmod +x cuda_<version>_linux.run
    sudo ./cuda_<version>_linux.run

During installation, note:

    If the host has already installed the NVIDIA Driver, you can cancel the Driver installation in the runfile installation interface and only install CUDA Toolkit.
    Avoid the runfile overwriting existing stable drivers.
    Do not mix drivers and CUDA packages from multiple sources.

### 11.3 Verification

    nvcc -V
    nvidia-smi

### 11.4 Maintenance Issues with runfile

Issues with runfile installation:

- Not conducive to apt/dnf management;
- Uninstallation and upgrades are not as clear as with package managers;
- Prone to overwriting existing drivers;
- Difficult for batch automation;
- High long-term maintenance cost for production nodes.

Production recommendation:

    Unless in an offline environment or with special requirements, it is not recommended to prioritize using runfile to install CUDA Toolkit.

---

## TwelveI don't know.CUDA Environment Variable Configuration

Common CUDA installation paths: /usr/local/cuda-12.2

/usr/local/cuda
/usr/local/cuda-12.2
/usr/local/cuda-12.4
/usr/local/cuda-13.0

### 12.1 PATH

PATH is used to locate commands like nvcc.

Configuration:

    export PATH=/usr/local/cuda/bin:$PATH

Verification:

    which nvcc
    nvcc -V

### 12.2 LD_LIBRARY_PATH

LD_LIBRARY_PATH is used to locate CUDA dynamic libraries at runtime.

Configuration:

    export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH

Verification:

    echo $LD_LIBRARY_PATH
    ldconfig -p | grep cuda

### 12.3 Writing to System Configuration

Recommended location:

    /etc/profile.d/cuda.sh

Example:

    sudo tee /etc/profile.d/cuda.sh > /dev/null <<'EOF'
    export PATH=/usr/local/cuda/bin:$PATH
    export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
    EOF

Load:

    source /etc/profile.d/cuda.sh

### 12.4 Managing Multiple CUDA Versions

If multiple CUDA versions exist on the system:

    /usr/local/cuda-11.8
    /usr/local/cuda-12.2
    /usr/local/cuda-12.4
    /usr/local/cuda -> /usr/local/cuda-12.2

Check symbolic link:

    ls -l /usr/local/cuda

Switch symbolic link example:

    sudo ln -sfn /usr/local/cuda-12.2 /usr/local/cuda

Verification after switch:

    nvcc -V
    which nvcc

Production recommendation:

    It is not recommended to frequently switch CUDA versions on production GPU nodes.
    For multi-tenant environments requiring multiple CUDA versions, prioritize container images.

---

## ThirteenI don't know.CUDA Samples Testing

After installing the CUDA Toolkit, you can verify it through CUDA Samples.

Sample paths may vary depending on CUDA versions and installation methods.

Common path:

    /usr/local/cuda/samples

If it doesn't exist, you may need to install samples separately or obtain them from the official repository.

### 13.1 deviceQuery Test

Enter directory:

    cd /usr/local/cuda/samples/1_Utilities/deviceQuery

Compile:

    make

Run:

    ./deviceQuery

Success typically shows:

    Result = PASS

deviceQuery can verify:

- Whether CUDA Runtime can access GPU;
- GPU device properties;
- CUDA Capability;
- Memory;
- Number of multiprocessors;
- Maximum thread count, etc.

### 13.2 bandwidthTest Test

Enter directory:

    cd /usr/local/cuda/samples/1_Utilities/bandwidthTest

Compile:

    make

Run:

    ./bandwidthTest

Success typically shows:

    Result = PASS

bandwidthTest can preliminarily verify:

- Host to Device bandwidth;
- Device to Host bandwidth;
- Device to Device bandwidth.

Note:

    bandwidthTest is not a complete performance benchmark.
    It can only serve as basic connectivity and bandwidth verification.

### 13.3 Compilation Failure Troubleshooting

If make fails, common causes include:

- No gcc / make installed;
- Incomplete CUDA Toolkit installation;
- PATH not configured;
- nvcc not found;
- Incompatible gcc version;
- Samples path missing;
- Insufficient permissions.

Troubleshoot:

    which nvcc
    nvcc -V
    gcc --version
    make --version
    echo $PATH
    echo $LD_LIBRARY_PATH

---

## FourteenI don't know.Container-based CUDA Testing

In production Kubernetes environments, it's recommended to test CUDA Runtime through containers.

Prerequisites:

- Host nvidia-smi works normally;
- NVIDIA Container Toolkit installed;
- Docker or containerd properly configured;
- Container CUDA Runtime image is compatible with host driver.

### 14.1 Docker Test nvidia-smi

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

Success indicates:

- Host driver is available;
- Docker GPU parameters are available;
- NVIDIA Container Toolkit is functional;
- GPU access is available inside the container.

### 14.2 Docker Test CUDA Runtime

Enter container:

    docker run --rm -it --gpus all nvidia/cuda:12.2.0-devel-ubuntu22.04 bash

Execute inside container:

    nvidia-smi
    nvcc -V

Note:

    Base image may not include nvcc.
    Runtime images typically don't include full development tools.
    Devel images usually include nvcc and development headers.

### 14.3 nvidia/cuda Image Types

Common image types:

    base
    runtime
    devel

Differences:

    base:
        Minimal CUDA runtime base, suitable for simple runtime environments.

    runtime:
        Includes libraries needed to run CUDA applications, suitable for production use.

    devel:
        Includes development and compilation tools, such as nvcc, suitable for building and compiling.

Production recommendation:

    Prioritize using runtime images for running services.
    Use devel images for building images.
    Avoid using devel images for production runtime without justification, as they are larger and have a larger attack surface.

---

## FifteenI don't know.Testing CUDA in Kubernetes

In Kubernetes, CUDA typically comes from container images rather than the host's CUDA Toolkit.

### 15.1 Prerequisites

Before a GPU Pod can run successfully, the following must be met:

- Host NVIDIA Driver is functioning properly;
- NVIDIA Container Toolkit is functioning properly;
- Device Plugin or GPU Operator is functioning properly;
- kubelet has registered nvidia.com/gpu;
- Pod correctly requests GPU;
- CUDA Runtime in the image is compatible with the driver.

### 15.2 View Node GPU Resources

    kubectl describe node <gpu-node-name>

Focus on:

    Capacity:
      nvidia.com/gpu: 1

    Allocatable:
      nvidia.com/gpu: 1

If nvidia.com/gpu is not present, it indicates Kubernetes has not yet recognized GPU resources.

### 15.3 Create CUDA Test Pod

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

Save:

    cuda-test.yaml

Deploy:

    kubectl apply -f cuda-test.yaml

Check:

    kubectl get pod cuda-test -o wide
    kubectl logs cuda-test
    kubectl describe pod cuda-test

### 15.4 Use devel Image to Test nvcc

If you want to test nvcc inside a Pod, use a devel image:

    apiVersion: v1
    kind: Pod
    metadata:
      name: cuda-devel-test
      namespace: default
    spec:
      restartPolicy: Never
      containers:
        - name: cuda
          image: nvidia/cuda:12.2.0-devel-ubuntu22.04
          command: ["bash", "-lc", "nvidia-smi && nvcc -V"]
          resources:
            limits:
              nvidia.com/gpu: 1

Check:

    kubectl logs cuda-devel-test

### 15.5 nvidia-smi Works Normally in Pod but Business Pod Fails

If the test Pod works normally but the business Pod fails, the GPU foundation chain is likely functioning properly.

Continue troubleshooting:

- Business image CUDA version;
- PyTorch / TensorFlow version;
- cuDNN version;
- TensorRT version;
- Business startup parameters;
- Model files;
- GPU memory usage;
- Permissions;
- Environment variables;
- Image entry script.

---

## SixteenI don't know.Python Framework CUDA Testing

### 16.1 PyTorch Test

Run in container or host:

    python3 -c "import torch; print('torch:', torch.__version__); print('cuda version:', torch.version.cuda); print('cuda available:', torch.cuda.is_available()); print('device count:', torch.cuda.device_count())"

If output:

    cuda available: True
    device count: 1

It indicates PyTorch can see the GPU.

### 16.2 TensorFlow Test

    python3 -c "import tensorflow as tf; print(tf.__version__); print(tf.config.list_physical_devices('GPU'))"

If GPU devices are visible, it indicates TensorFlow can recognize the GPU.

### 16.3 Common Misconceptions

Misconception 1:

    nvcc -V works normally, so PyTorch can definitely use GPU.

Not necessarily.

PyTorch's ability to use GPU depends on PyTorch build version, CUDA Runtime, cuDNN, driver compatibility, and device visibility.

Misconception 2:

    nvidia-smi works normally, so TensorFlow can definitely use GPU.

Not necessarily.

nvidia-smi only indicates the driver layer is functioning normally, not that framework dependencies are complete.

Misconception 3:

    Host CUDA Toolkit version must be exactly the same as container CUDA Runtime.

Not necessarily.

Container applications mainly depend on container CUDA Runtime and host driver compatibility.

---

## SeventeenI don't know.CUDA and Container Image Version Management

### 17.1 Why Standardize CUDA Images

If business teams use different CUDA images arbitrarily, it can lead to:

- Driver compatibility chaos;
- Uncontrolled image size;
- Difficult to manage security vulnerabilities;
- Inconsistent cuDNN / TensorRT versions;
- Difficult to reproduce online issues;
- Difficult to upgrade GPU nodes;
- Complex troubleshooting in multi-tenant platforms.

It is recommended that the platform maintain standard images.

### 17.2 Image Layering Recommendations

Build image:

    company/cuda:12.2-devel-ubuntu22.04

Run image:

    company/cuda:12.2-runtime-ubuntu22.04

PyTorch image:

    company/pytorch:2.x-cuda12.2-cudnn-runtime

TensorRT image:

    company/tensorrt:cuda12.2-runtime

Inference service base image:

    company/inference-base:cuda12.2-runtime

### 17.3 Image Selection Recommendations /think

# Running Service:

    Prioritize runtime

# Building Program:

    Use devel

# Debugging Environment:

    Temporarily use devel

# Production Environment:

    Avoid running services directly with large devel images
    Avoid using latest
    Avoid using images from unknown sources
    Avoid each project maintaining chaotic CUDA base images

---

## EighteenI don't know.CUDA and Kubernetes Scheduling Relationship

CUDA itself does not handle Kubernetes scheduling.

Kubernetes GPU scheduling depends on:

- NVIDIA Driver
- NVIDIA Container Toolkit
- NVIDIA Device Plugin
- kubelet resource registration
- Pod resources.limits
- Scheduler

CUDA is the runtime environment for containers or applications to use GPU.

### 18.1 Pod GPU Request

GPU Pod example:

    resources:
      limits:
        nvidia.com/gpu: 1

This indicates the Pod requests 1 GPU.

Note:

    GPU is an extended resource.
    GPU cannot be written as 500m like CPU.
    nvidia.com/gpu: 1 represents a full GPU.
    MIG or shared schemes are designed separately.

### 18.2 CUDA_VISIBLE_DEVICES

Common environment variables in containers:

    CUDA_VISIBLE_DEVICES

It controls which GPU devices are visible to the application.

In Kubernetes + Device Plugin scenarios, the Device Plugin typically sets visible devices for containers.

Check:

    echo $CUDA_VISIBLE_DEVICES

Execute inside the container:

    nvidia-smi

May only show GPUs allocated to the current container.

### 18.3 Multi-GPU Pod

Request multiple GPUs:

    resources:
      limits:
        nvidia.com/gpu: 2

The business application needs to correctly use multi-GPU, for example:

- PyTorch DistributedDataParallel
- TensorFlow MirroredStrategy
- NCCL
- MPI
- Horovod
- DeepSpeed
- Megatron-LM

Operations need to pay attention to:

- Whether the Pod actually gets multiple GPUs
- Whether GPUs are on the same node
- Whether GPU topology is suitable
- Whether NCCL communication is normal
- Whether memory is balanced
- Whether multi-GPU utilization is balanced

---

## NineteenI don't know.Common CUDA Directory Explanations

### 19.1 /usr/local/cuda

It's usually a symlink to the current default CUDA version.

Check:

    ls -l /usr/local/cuda

### 19.2 /usr/local/cuda/bin

Common commands:

    nvcc
    cuda-gdb
    nsight related tools

Check:

    ls /usr/local/cuda/bin

### 19.3 /usr/local/cuda/lib64

CUDA library directory.

Check:

    ls /usr/local/cuda/lib64

### 19.4 /usr/local/cuda/include

CUDA header file directory.

Used for development and compilation of CUDA programs.

### 19.5 /usr/local/cuda/samples

CUDA sample code directory.

Not all installation methods include samples by default.

---

## TwentyI don't know.Complete Validation Process After CUDA Installation

### 20.1 Driver Layer Validation

    nvidia-smi
    nvidia-smi -L
    nvidia-smi -q
    lsmod | grep nvidia
    ls -l /dev/nvidia*

### 20.2 CUDA Toolkit Validation

    which nvcc
    nvcc -V
    ls -l /usr/local/cuda
    echo $PATH
    echo $LD_LIBRARY_PATH

### 20.3 Dynamic Library Validation

    ldconfig -p | grep cuda
    ldconfig -p | grep cudart

### 20.4 CUDA Samples Validation

    cd /usr/local/cuda/samples/1_Utilities/deviceQuery
    make
    ./deviceQuery

Expected:

    Result = PASS

### 20.5 Container Validation

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

If nvcc is needed:

    docker run --rm --gpus all nvidia/cuda:12.2.0-devel-ubuntu22.04 nvcc -V

### 20.6 Kubernetes Validation

    kubectl describe node <gpu-node-name>
    kubectl apply -f cuda-test.yaml
    kubectl logs cuda-test
    kubectl describe pod cuda-test

---

## Twenty-oneI don't know.Common Fault: nvidia-smi is normal, but nvcc does not exist

### 21.1 Phenomenon

Execute:

    nvidia-smi

Normal.

Execute:

    nvcc -V

Error:

    command not found

### 21.2 Cause

nvidia-smi comes from NVIDIA Driver.

nvcc comes from CUDA Toolkit.

The host does not have CUDA Toolkit installed, or PATH is not configured.

### 21.3 Troubleshooting

    which nvcc
    ls -l /usr/local/cuda
    echo $PATH
    dpkg -l | grep -i cuda
    rpm -qa | grep -i cuda

### 21.4 Resolution

If nvcc is indeed needed:

    Install CUDA Toolkit
    Configure PATH
    Configure LD_LIBRARY_PATH

If it's just for Kubernetes GPU Worker:

    Can skip installing nvcc.
    Use container image to test CUDA Runtime instead.

---

## Twenty-twoI don't know.Common Fault: nvcc is normal, but nvidia-smi is abnormal

### 22.1 Phenomenon

Execute: /think

nvcc -V

Normal.

Execution:

    nvidia-smi

Failed.

### 22.2 Causes

nvcc only indicates that CUDA Toolkit exists.

nvidia-smi failure indicates an issue with the NVIDIA Driver layer.

Possible causes:

- Driver not installed;
- Driver module not loaded;
- Secure Boot blocks the module;
- nouveau conflict;
- Driver fails after kernel upgrade;
- GPU hardware or PCIe issues.

### 22.3 Troubleshooting

    lspci | grep -i nvidia
    lsmod | grep nvidia
    dmesg | grep -i nvidia
    dmesg | grep -i xid
    mokutil --sb-state

### 22.4 Resolution

First fix the NVIDIA Driver, then handle CUDA.

Do not assume that nvcc being normal means the GPU is available.

---

## Twenty-three, Common Fault Three: CUDA driver version is insufficient for CUDA runtime version

### 23.1 Phenomenon

Error reported when business program or container starts:

    CUDA driver version is insufficient for CUDA runtime version

### 23.2 Causes

The CUDA Runtime version in the container or application is newer, while the host's NVIDIA Driver version is too old.

### 23.3 Troubleshooting

Host:

    nvidia-smi

Check Driver Version.

Inside the container:

    nvidia-smi
    python3 -c "import torch; print(torch.version.cuda); print(torch.cuda.is_available())"

Check image:

    docker image inspect <image>

Kubernetes:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep image:

### 23.4 Resolution

Resolution steps:

    1. Upgrade host NVIDIA Driver
    2. Use a lower CUDA Runtime image
    3. Standardize business base image
    4. Establish a Driver / CUDA / Framework version matrix

Production recommendations:

    Do not allow business images to arbitrarily upgrade CUDA Runtime.
    GPU node driver upgrades must validate business images.

---

## Twenty-four, Common Fault Four: CUDA out of memory

### 24.1 Phenomenon

Error during training or inference:

    CUDA out of memory

Or:

    RuntimeError: CUDA out of memory

### 24.2 Causes

Insufficient GPU memory.

Possible causes:

- Batch size too large;
- Model too large;
- Multiple processes using the same GPU;
- Memory leak;
- Inference service loads multiple models;
- High memory usage for training intermediate activations;
- Memory not released;
- Pod requested GPU, but internal multi-process contention on the same card;
- MIG / shared GPU allocation too small.

### 24.3 Troubleshooting

Check memory:

    nvidia-smi

Check processes:

    nvidia-smi

Reverse lookup processes on host:

    ps -ef | grep <PID>

Kubernetes reverse lookup Pod:

    kubectl get pod -A -o wide | grep <node-name>
    crictl ps
    crictl inspect <container-id>

### 24.4 Resolution

Resolution steps:

- Reduce batch size;
- Use smaller models;
- Use FP16 / BF16;
- Use model quantization;
- Clean up abnormal processes;
- Limit concurrent per node;
- Split models;
- Use larger memory GPU;
- Use MIG for reasonable partitioning;
- Optimize business code to release memory.

Note:

    Kubernetes's nvidia.com/gpu only indicates requested GPU count.
    It does not automatically limit internal memory upper bounds.
    Memory governance requires combining application, framework, MIG, or platform policies.

---

## Twenty-five, Common Fault Five: PyTorch cannot detect GPU

### 25.1 Phenomenon

Execution:

    python3 -c "import torch; print(torch.cuda.is_available())"

Output:

    False

### 25.2 Possible Causes

Possible causes:

- Installed CPU-only PyTorch;
- PyTorch CUDA version mismatch with environment;
- Host driver does not support container CUDA Runtime;
- Container has no GPU mounted;
- Kubernetes Pod did not request nvidia.com/gpu;
- NVIDIA Container Toolkit not configured;
- Device Plugin abnormal;
- CUDA_VISIBLE_DEVICES setting abnormal.

### 25.3 Troubleshooting

Check PyTorch:

    python3 -c "import torch; print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available()); print(torch.cuda.device_count())"

Check GPU:

    nvidia-smi

Check environment variables:

    echo $CUDA_VISIBLE_DEVICES

Kubernetes:

    kubectl describe pod <pod-name> -n <namespace>
    kubectl describe node <gpu-node-name>

### 25.4 Resolution

Resolution steps:

- Install CUDA version PyTorch;
- Use official matched CUDA PyTorch image;
- Confirm Pod requests GPU;
- Fix NVIDIA Container Toolkit;
- Fix Device Plugin;
- Use compatible CUDA Runtime image;
- Upgrade host driver.

---

## Twenty-six, Common Fault Six: nvidia-smi fails inside container

### 26.1 Phenomenon

Host:

    nvidia-smi

Normal.

Inside container:

    nvidia-smi

Failed.

### 26.2 Possible Causes

Possible causes:

- NVIDIA Container Toolkit is not installed;
- Docker is not using --gpus all;
- containerd is not configured with NVIDIA runtime;
- Kubernetes Device Plugin is not running;
- Pod does not request nvidia.com/gpu;
- container image does not include nvidia-smi;
- driver library is not mounted;
- container runtime configuration is incorrect.

### 26.3 Docker Troubleshooting

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

If failed:

    nvidia-container-cli info
    docker info | grep -i runtime

### 26.4 Kubernetes Troubleshooting

    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia
    kubectl describe node <gpu-node-name>
    kubectl describe pod <gpu-pod-name> -n <namespace>

Confirm the presence of:

    resources:
      limits:
        nvidia.com/gpu: 1

---

## Twenty-SevenI don't know.Common Issue Seven: deviceQuery Compilation Failure

### 27.1 Phenomenon

Executing:

    make

Fails.

### 27.2 Possible Causes

Possible causes:

- make is not installed;
- gcc/g++ is not installed;
- nvcc does not exist;
- CUDA Toolkit installation is incomplete;
- PATH is not configured;
- gcc version is incompatible;
- samples path does not exist;
- insufficient permissions.

### 27.3 Troubleshooting

    which nvcc
    nvcc -V
    gcc --version
    g++ --version
    make --version
    echo $PATH
    ls -l /usr/local/cuda
    ls -l /usr/local/cuda/samples

### 27.4 Resolution

Ubuntu:

    sudo apt-get install -y build-essential

Rocky:

    sudo dnf install -y gcc gcc-c++ make

Reload environment variables:

    source /etc/profile.d/cuda.sh

Compile again:

    make clean
    make

---

## Twenty-EightI don't know.Common Issue Eight: Cannot Find libcudart.so or libcuda.so

### 28.1 Phenomenon

Program startup error:

    libcudart.so: cannot open shared object file

Or:

    libcuda.so: cannot open shared object file

### 28.2 Difference

libcuda.so:

    Typically comes from NVIDIA Driver.

libcudart.so:

    Typically comes from CUDA Runtime / CUDA Toolkit.

### 28.3 Troubleshooting

    ldconfig -p | grep libcuda
    ldconfig -p | grep libcudart
    echo $LD_LIBRARY_PATH
    find /usr/local/cuda -name "libcudart.so*"
    find /usr -name "libcuda.so*"

Also troubleshoot in containers:

    ldconfig -p | grep cuda
    echo $LD_LIBRARY_PATH

### 28.4 Resolution

Resolution methods:

- Install correct CUDA Runtime;
- Configure LD_LIBRARY_PATH;
- Execute ldconfig;
- Use correct CUDA image;
- Confirm NVIDIA Container Toolkit mounts driver library;
- Avoid overriding critical library paths in containers.

---

## Twenty-NineI don't know.Common Issue Nine: Multiple CUDA Versions Conflict

### 29.1 Phenomenon

Multiple CUDA versions exist in the system, and the program uses the wrong version.

Manifestations:

- nvcc -V shows version not expected;
- compilation uses wrong header files;
- runtime loads wrong dynamic libraries;
- PyTorch / TensorFlow reports version mismatch;
- multiple CUDA paths in LD_LIBRARY_PATH.

### 29.2 Troubleshooting

    which nvcc
    nvcc -V
    echo $PATH
    echo $LD_LIBRARY_PATH
    ls -l /usr/local/cuda
    ls -d /usr/local/cuda-*

### 29.3 Resolution

Clarify soft links:

    sudo ln -sfn /usr/local/cuda-12.2 /usr/local/cuda

Clean old paths in environment variables:

    vi /etc/profile.d/cuda.sh

Reload:

    source /etc/profile.d/cuda.sh

Production recommendations:

    It is not recommended to maintain multiple host CUDA versions long-term in production nodes.
    Multiple version requirements should be isolated through container images as much as possible.

---

## ThirtyI don't know.CUDA and GPU Performance Testing Boundaries

CUDA installation verification does not equal performance benchmarking.

### 30.1 Basic Verification

Basic verification includes:

    nvidia-smi
    nvcc -V
    deviceQuery
    bandwidthTest
    PyTorch cuda.is_available
    Docker CUDA image testing
    Kubernetes CUDA Pod testing

These can only indicate the basic CUDA chain is normal.

### 30.2 Performance Benchmarking

Performance benchmarking requires combining with business scenarios.

Training scenarios focus on:

- GPU utilization;
- memory usage;
- samples per second;
- multi-GPU communication;
- data loading speed;
- CPU preprocessing;
- network throughput;
- storage throughput.

Inference scenarios focus on:

- QPS;
- P50 / P95 / P99 latency;
- batch size;
- concurrency;
- memory usage;
- GPU utilization;
- model loading time;
- cold start time.

### 30.3 Do Not Only Look at GPU-Util

High GPU-Util does not necessarily mean business health.

Low GPU-Util does not necessarily indicate an anomaly.

Need to combine with: /think

- Business throughput;
- Business latency;
- VRAM;
- CPU;
- Network;
- Disk;
- Logs;
- Framework metrics;
- Prometheus metrics.

---

## 31. Production Environment CUDA Management Recommendations

### 31.1 Establish Version Matrix

Recommended to record:

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

### 31.2 Do Not Place Business Dependencies on Host

Host is recommended to maintain:

- NVIDIA Driver;
- NVIDIA Container Toolkit;
- containerd / Docker;
- kubelet;
- Device Plugin / GPU Operator;
- Monitoring Agent;
- Log Agent.

Business dependencies are recommended to be placed in the image:

- CUDA Runtime;
- cuDNN;
- TensorRT;
- Python;
- PyTorch;
- TensorFlow;
- Business code.

### 31.3 Fix Image Version

Not recommended for production use:

    latest

Recommended to use specific versions:

    nvidia/cuda:12.2.0-runtime-ubuntu22.04
    company/pytorch:2.x-cuda12.2-cudnn-runtime

### 31.4 Establish Base Image Repository

Recommended for enterprise internal maintenance:

- CUDA base image;
- CUDA runtime image;
- CUDA devel image;
- PyTorch image;
- TensorFlow image;
- TensorRT image;
- Inference service base image.

And synchronize to internal image repository, for example:

    Harbor
    Nexus
    Alibaba Cloud image repository
    Private Registry

### 31.5 Control CUDA Upgrade Pace

Before CUDA upgrade, need to verify:

- Driver compatibility;
- Framework compatibility;
- Business image compatibility;
- Model loading;
- Inference performance;
- Training performance;
- VRAM usage;
- Kubernetes GPU Pod;
- Monitoring metrics;
- Rollback plan.

---

## 32. CUDA Installation and Testing Checklist

### 32.1 Before Installation

    [ ] GPU hardware is recognized
    [ ] lspci can see NVIDIA GPU
    [ ] NVIDIA Driver is installed
    [ ] nvidia-smi is normal
    [ ] Driver Version is recorded
    [ ] CUDA version requirements are clear
    [ ] AI framework version is confirmed
    [ ] Whether host CUDA Toolkit is needed is confirmed
    [ ] Installation method is determined
    [ ] Old CUDA environment is checked
    [ ] PATH / LD_LIBRARY_PATH is checked

### 32.2 During Installation

    [ ] NVIDIA CUDA Repository is configured
    [ ] apt/dnf cache is updated
    [ ] CUDA Toolkit installation is successful
    [ ] No overwrite of existing stable NVIDIA Driver
    [ ] Environment variables are configured
    [ ] ldconfig is executed or library path is confirmed

### 32.3 After Installation

    [ ] nvcc -V is normal
    [ ] which nvcc path is correct
    [ ] /usr/local/cuda symlink is correct
    [ ] nvidia-smi is normal
    [ ] deviceQuery test passed
    [ ] bandwidthTest test passed
    [ ] Docker CUDA image test passed
    [ ] Kubernetes CUDA Pod test passed
    [ ] PyTorch / TensorFlow test passed
    [ ] No CUDA Runtime and Driver incompatibility issues

---

## 33. Understanding CUDA Layers from Troubleshooting Perspective

CUDA troubleshooting should not start with reinstalling CUDA immediately.

Should judge by layers.

### 33.1 lspci is abnormal

Issue is in hardware or BIOS layer.

Troubleshoot:

    GPU is properly inserted
    BIOS recognition
    Above 4G Decoding
    PCIe slot
    riser
    Power supply

### 33.2 nvidia-smi is abnormal

Issue is in driver layer.

Troubleshoot:

    NVIDIA Driver
    nouveau
    Secure Boot
    DKMS
    Kernel Headers
    XID
    PCIe error

### 33.3 nvcc is abnormal

Issue is in CUDA Toolkit or environment variable layer.

Troubleshoot:

    CUDA Toolkit is installed
    PATH
    /usr/local/cuda
    nvcc path
    Multiple version conflicts

### 33.4 CUDA is abnormal in container

Issue may be in container runtime layer.

Troubleshoot:

    NVIDIA Container Toolkit
    Docker --gpus
    containerd runtime
    Image CUDA Runtime
    Driver compatibility
    /dev/nvidia* mount

### 33.5 CUDA is abnormal in Kubernetes Pod

Issue may be in K8S GPU scheduling layer.

Troubleshoot:

    Device Plugin
    GPU Operator
    kubelet nvidia.com/gpu
    Pod resources.limits
    Taint / Toleration
    NodeSelector / Affinity
    RuntimeClass
    CUDA image version

---

## 34. Summary

CUDA is the core runtime and development capability in the NVIDIA GPU computing ecosystem, but in operations scenarios, CUDA cannot be simply understood as "installing a Toolkit."

The following relationships must be clearly understood:

    NVIDIA Driver:
        The host must be stable, as nvidia-smi depends on it.

    CUDA Toolkit:
        Required for developing and compiling CUDA programs, with nvcc coming from it.

    CUDA Runtime:
        Required for running CUDA applications, typically pre-installed in container images.

    cuDNN / TensorRT:
        Upper-level libraries focused on deep learning and inference optimization.

    NVIDIA Container Toolkit:
        Responsible for enabling containers to access the host's GPU and driver capabilities.

    Kubernetes Device Plugin:
        Responsible for registering GPUs as nvidia.com/gpu scheduling resources.

In a Kubernetes GPU cluster, the recommended approach is:

    Keep the host clean:
        Driver + Container Toolkit + Runtime + Device Plugin

    Place business dependencies in the image:
        CUDA Runtime + cuDNN + TensorRT + AI Framework + App

The correct order for CUDA installation and testing is:

    1. lspci to confirm GPU hardware visibility
    2. nvidia-smi to confirm driver functionality
    3. Confirm driver compatibility with the target CUDA Runtime
    4. Determine if the host needs CUDA Toolkit
    5. If installation is required, install the specified version via official repositories or package managers
    6. Configure PATH and LD_LIBRARY_PATH
    7. Use nvcc -V to verify Toolkit
    8. Use deviceQuery / bandwidthTest to verify basic CUDA
    9. Use nvidia/cuda container to verify runtime
    10. Use Kubernetes GPU Pod to verify cluster scheduling and container CUDA
    11. Use PyTorch / TensorFlow to verify business frameworks
    12. Establish a Driver / CUDA / Framework / image version matrix

When troubleshooting, maintain a layered approach:

    nvidia-smi fails:
        Check the Driver

    nvcc not found:
        Check CUDA Toolkit or PATH

    Failure inside the container:
        Check Container Toolkit and image

    Pod Pending:
        Check Device Plugin and scheduling

    CUDA out of memory:
        Check memory, model, batch size, and process usage

    Driver version insufficient:
        Check driver compatibility with CUDA Runtime

For production GPU platforms, the most important aspect is not installing many CUDA components on each host, but establishing a stable driver layer, standardized images, clear version matrices, and reproducible verification processes.

---

## Reference Documents

- NVIDIA CUDA Installation Guide for Linux:
  https://docs.nvidia.com/cuda/cuda-installation-guide-linux/

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