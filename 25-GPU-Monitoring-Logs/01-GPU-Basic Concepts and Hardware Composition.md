# 01-GPU-Foundations and Hardware Composition

## Document Explanation

This document establishes foundational understanding for GPU operations, GPU node management, Kubernetes GPU scheduling, and AI inference/training platform operations.

This document does not directly address GPU driver installation, CUDA installation, Device Plugin deployment, or Prometheus monitoring implementation. Instead, it first answers several fundamental questions:

- What exactly is a GPU;
- Why GPUs are suitable for AI training and inference;
- What key hardware capabilities compose a single GPU;
- Which metricsTransport engineers should focus on for GPUs;
- How GPUs become schedulable resources in Kubernetes clusters;
- What to check first when GPU nodes experience issues.

After mastering this document, subsequent content will be easier to understand:

- 02-GPU-BIOS and Hardware Optimization
- 03-NVIDIA-Driver Installation and Verification
- 04-CUDA-Installation and Testing
- 05-K8S-GPU-Resource Concepts and Scheduling Principles
- 06-NVIDIA-Device-Plugin-and-Operator-Installation
- 07-GPU-Pod-Deployment and Scheduling Practice
- 08-GPU-Monitoring and Alert Integration

## Tags

#GPU #NVIDIA #CUDA #Kubernetes #AiInfrastructure #TransportBarriers #Clouds. #SRE

## Recommended Path

Recommended path:

    06-GPU and AI Infrastructure/01-GPU Foundations/01-GPU-Foundations and Hardware Composition.md

---

## One, WhyTransport Engineers Need to Understand GPUs

In traditional operations scenarios, servers primarily focus on CPU, memory, disk, and network.

However, in AI training, AI inference, video rendering, scientific computing, and high-performance computing scenarios, GPUs become core computing resources.

ForTransport engineers, GPUs are not just "a graphics card," but a category of high-value, highly dependent, state-sensitive, and scheduling-constrained computing resources.

Common GPU operation issues include:

- Servers can see GPUs, but containers cannot;
- nvidia-smi is normal, but Kubernetes Pods cannot schedule GPUs;
- Pods have requested nvidia.com/gpu but remain Pending;
- Low GPU utilization but high memory usage;
- High GPU temperatures causing throttling or node anomalies;
- Mismatched driver versions, CUDA versions, and container image versions;
- Poor communication efficiency between multiple GPUs, leading to suboptimal training performance;
- High GPU node costs but unclear resource utilization;
- GPU metrics not integrated with Prometheus, making capacity planning and alerts impossible.

Therefore, GPU operations are not simply executing:

    nvidia-smi

But rather understanding:

    Hardware recognition
    -> BIOS/PCIe initialization
    -> NVIDIA driver
    -> CUDA Runtime
    -> Container Runtime
    -> NVIDIA Container Toolkit
    -> Kubernetes Device Plugin
    -> kubelet resource registration
    -> Scheduler scheduling
    -> Pod using GPU
    -> Metric collection and log troubleshooting

This is a complete chain.

---

## Two, Basic Concepts of GPUs

GPU, full name Graphics Processing Unit, graphics processor.

Originally, GPUs were mainly used for graphics rendering, such as game visuals, 3D modeling, and video processing.

Later, with the growth of parallel computing demands, GPUs were extensively used for:

- Deep learning training;
- AI inference;
- Large model training;
- Large model inference services;
- Scientific computing;
- Image processing;
- Video encoding/decoding;
- High-performance computing;
- Cloud desktops and graphics workstations;
- Digital twins and simulation computing.

The biggest difference between GPUs and CPUs is their computing model.

CPUs are suitable for complex logic, branch judgments, system scheduling, and serial tasks.

GPUs are suitable for large-scale repetitive, simple, and parallel computing tasks.

You can simply understand:

    CPU: Few strong cores, suitable for complex control logic
    GPU: Many parallel computing units, suitable for matrix calculations and batch parallel computing

AI training and inference heavily use matrix multiplication and tensor calculations, which are very suitable for GPUs.

---

## Three, Differences Between CPUs and GPUs

### 3.1 CPU Characteristics

CPUs are more suitable for general computing.

Typical features:

- Relatively few cores;
- Strong single-core performance;
- Suitable for complex logic judgment;
- Suitable for operating system scheduling;
- Suitable for network protocol stacks, file systems, and database transactions;
- Complex cache hierarchy;
- Strong branch prediction capability.

Common scenarios:

- Operating system operation;
- Web services;
- Database services;
- Middleware;
- Kubernetes control plane components;
- CI/CD task scheduling;
- Ordinary business services.

### 3.2 GPU Characteristics

GPUs are more suitable for large-scale parallel computing.

Typical features:

- Many computing cores;
- Suitable for batch parallel tasks;
- Suitable for matrix calculations;
- Suitable for tensor calculations;
- High memory bandwidth;
- High data throughput requirements;
- Strong dependency on drivers, CUDA, and runtime environments.

Common scenarios:

- PyTorch / TensorFlow model training;
- Stable Diffusion image generation;
- Large language model inference;
- Video transcoding;
- Image recognition;
- 3D rendering;
- Scientific simulation.

### 3.3 Differences from an Operations Perspective

From an operations perspective, the management of CPUs and GPUs differs significantly.

CPUs are generally managed directly by the operating system, and Kubernetes can natively recognize CPU resources.

For example:

    resources:
      requests:
        cpu: "500m"
      limits:
        cpu: "1"

GPUs are different.

Kubernetes does not natively know how many NVIDIA GPUs a machine has.

It requires NVIDIA Device Plugin or NVIDIA GPU Operator to register GPU resources with kubelet.

In Kubernetes, GPUs are typically represented as extended resources:

    nvidia.com/gpu: 1

Therefore, the GPU scheduling chain is longer than for CPUs and more prone to issues.

---

## Four, Common GPU Types and Use Cases

### 4.1 Consumer-Grade GPUs

Common models:

- RTX 3060
- RTX 3070
- RTX 3080
- RTX 3090
- RTX 4090

Features:

- Relatively low cost;
- Suitable for personal learning, small-scale inference, and image generation experiments;
- Not necessarily suitable for long-term full-load production tasks;
- Additional attention is needed for cooling, power supply, and stability;
- Weaker data center characteristics.

Common use cases:

- AI learning environments;
- Single-machine inference;
- Small-scale model testing;
- Image generation;
- Video rendering.

### 4.2 Data Center GPUs

Common models: /think

- NVIDIA T4
- NVIDIA L4
- NVIDIA A10
- NVIDIA A30
- NVIDIA A40
- NVIDIA A100
- NVIDIA H100
- NVIDIA H200
- NVIDIA H20
- NVIDIA A800

Features:

- Data Center Oriented;
- Stronger Stability;
- Support for Better Cooling Design;
- Support for ECC Memory;
- Support for Stronger Virtualization or Isolation Capabilities;
- More Suitable for Kubernetes GPU Clusters;
- More Suitable for AI Training, Inference, and High-Performance Computing.

Common Use Cases:

- Enterprise AI Inference Platform;
- Large Model Training;
- Large Model Inference;
- Multi-Tenant GPU Platform;
- High-Performance Computing Cluster;
- Cloud Provider GPU Instances.

### 4.3 Inference GPU vs Training GPU

GPUs can also be categorized by use case into inference and training types.

Inference GPUs focus more on:

- Request Latency per Call;
- Concurrency Capability;
- Memory Usage;
- Cost-Effectiveness;
- Service Stability;
- Batch Processing Capability.

Training GPUs focus more on:

- Memory Capacity;
- Memory Bandwidth;
- Multi-GPU Communication;
- NVLink / PCIe Topology;
- Compute Throughput;
- Long-Term Full-Load Stability.

Example:

    T4 / L4: commonly used for inference, light training, and video processing
    A10 / A30: suitable for medium-scale training and inference
    A100 / H100: suitable for large-scale training and high-performance inference
    H20: common in AI computing scenarios under specific market supply constraints

---

## FiveI don't know.GPU Core Hardware Components

For operations, a GPU cannot be judged solely by its "model".

You also need to pay attention to the following hardware dimensions:

- Compute Units;
- Memory Capacity;
- Memory Bandwidth;
- PCIe Connection;
- NVLink;
- Power Consumption;
- Temperature;
- ECC;
- MIG;
- Cooling;
- Power Supply;
- Driver Support;
- CUDA Support.

---

## SixI don't know.Compute Units: CUDA Core, SM, Tensor Core

### 6.1 CUDA Core

CUDA Core can be simply understood as the basic computing unit in NVIDIA GPUs for parallel computing.

The more CUDA Cores, the stronger the theoretical parallel computing capability.

But you cannot simply assume that more CUDA Cores mean better business performance.

Actual performance is also affected by the following factors:

- Memory Capacity;
- Memory Bandwidth;
- GPU Architecture;
- CUDA Version;
- Model Structure;
- batch size;
- Data Loading Speed;
- CPU to GPU Data Transfer;
- Multi-GPU Communication Efficiency;
- Framework Optimization Level.

### 6.2 SM Units

SM, full name Streaming Multiprocessor.

SM is a more important computing organizational unit inside the GPU.

You can simply understand:

    GPU does not directly use all CUDA Cores scattered,
    but organizes parallel computing capability through multiple SMs.

Many performance analysis tools and GPU architecture documents focus around SM.

Operations engineers do not need to write CUDA Kernels, but need to know:

- SM Utilization is related to GPU Compute Utilization;
- When the model is not fully parallelized, GPU utilization may not reach high levels;
- Low GPU utilization is not necessarily a hardware issue, it could also be that the application hasn't fully utilized the GPU;
- Slow data loading, slow CPU preprocessing, and slow network can all lead to GPU idle time.

### 6.3 Tensor Core

Tensor Core is a hardware unit in NVIDIA GPUs specifically used for matrix and tensor calculations.

AI training and inference heavily rely on matrix calculations, so Tensor Core is critical for AI performance.

Tensor Core is commonly used for:

- FP16;
- BF16;
- TF32;
- INT8;
- FP8;
- Mixed Precision Training;
- Inference Acceleration.

From an operations perspective, you need to pay attention to:

- Different GPU architectures support different precision types;
- Whether AI frameworks enable mixed precision affects performance;
- Whether inference services use TensorRT affects throughput;
- Model quantization may significantly reduce memory usage and improve inference speed.

---

## SevenI don't know.Memory: One of the Most Core Resources for GPU Operations

GPU memory is similar to the GPU's own high-speed memory.

When AI models run, model parameters, intermediate activation values, input data, and batch data all occupy GPU memory.

### 7.1 Memory Capacity

Common memory capacities:

- 8GB
- 16GB
- 24GB
- 40GB
- 48GB
- 80GB
- 96GB
- 141GB, etc.

When memory is insufficient, common phenomena include:

- Program reports CUDA out of memory;
- Pod exits immediately after startup;
- Training task batch size cannot be set too large;
- Inference service fails to load models;
- Multiple processes compete for the same GPU;
- nvidia-smi shows memory usage that doesn't release long-term.

Common error examples:

    RuntimeError: CUDA out of memory

    CUDA error: out of memory

    failed to allocate memory on device

### 7.2 Memory Bandwidth

Memory bandwidth determines the speed at which the GPU reads and writes memory data.

For AI training and inference, memory bandwidth is very important.

If the compute core is strong but memory bandwidth is insufficient, the GPU may experience waiting for data.

Such issues manifest as:

- GPU utilization fluctuation;
- High memory usage but low compute utilization;
- Unstable training throughput;
- Low GPU utilization in multi-GPU training.

### 7.3 ECC Memory

ECC, full name Error Correcting Code.

ECC is used to detect and correct some errors in memory.

Data center GPUs typically support ECC.

ECC is important for production environments, especially:

- Long-term training tasks;
- Financial calculations;
- Scientific computing;
- Tasks with high accuracy requirements;
- Multi-tenant GPU platforms.

Check ECC status:

    nvidia-smi -q | grep -i ecc -A 5

Production recommendations:

- Data center GPUs are recommended to enable ECC;
- If ECC errors continue to increase, hardware health needs to be monitored;
- Single occasional errors can be observed;
- Repeated errors may indicate issues with the GPU, motherboard, power supply, or cooling.

---

## EightI don't know.PCIe: Data Channel Between GPU and Host

GPUs are plugged into server motherboards and need to communicate with the CPU, memory, and other devices via PCIe.

PCIe is very important for GPUs, especially when data is transferred from CPU memory to GPU memory.

### 8.1 PCIe Generations

Common PCIe generations:

- PCIe Gen3
- PCIe Gen4
- PCIe Gen5

Higher generations offer higher theoretical bandwidth.

### 8.2 PCIe Lane Width

Common widths:

- x16
- x8
- x4

GPUs typically prefer to operate in x16 mode.

If the GPU slot, electrical lanes, motherboard configuration, or CPU PCIe Lane is insufficient, the GPU may run in x8 or lower mode.

Check PCIe information:

    lspci | grep -i nvidia /think

View detailed PCIe link capabilities:

    lspci -vvv -s <GPU_PCI_ID>

Example:

    lspci | grep -i nvidia

Possible output:

    65:00.0 3D controller: NVIDIA Corporation Device xxxx

Continue checking:

    lspci -vvv -s 65:00.0 | grep -i width
    lspci -vvv -s 65:00.0 | grep -i speed

### 8.3 Common Symptoms of PCIe Issues

PCIe anomalies may lead to:

- GPU not recognized;
- GPU card drop;
- nvidia-smi stuttering;
- Low training performance;
- Slow multi-card communication;
- PCIe AER errors in system logs;
- GPU abnormal exit during stress testing.

Check kernel logs:

    dmesg | grep -i pci
    dmesg | grep -i nvidia
    journalctl -k | grep -i nvidia

Production recommendations:

- Must confirm PCIe slot positions before GPU node deployment;
- Multi-GPU servers should follow vendor topology recommendations;
- Don't only focus on GPU count, but also PCIe bandwidth;
- When performance anomalies occur, check if GPU is throttled or bandwidth reduced;
- Above 4G Decoding is typically important in GPU node BIOS.

---

## Nine. NVLink and Multi-GPU Communication

In multi-GPU training scenarios, GPUs need frequent data exchange.

If all communication goes through PCIe, it may become a bottleneck.

NVLink is NVIDIA's high-speed GPU interconnect technology.

### 9.1 Role of NVLink

NVLink can enhance data transfer capabilities between GPUs.

Common scenarios:

- Multi-card training;
- Large model training;
- Model parallelism;
- Tensor parallelism;
- Parameter synchronization;
- High-performance computing.

### 9.2 Check GPU Topology

Check GPU topology:

    nvidia-smi topo -m

Output may show:

    GPU0    GPU1    CPU Affinity
    GPU0     X      NV2
    GPU1    NV2      X

Common identifier meanings:

    X       Represents self
    PIX     Through same PCIe Switch
    PXB     Through multiple PCIe Bridges
    PHB     Through PCIe Host Bridge
    SYS     Cross CPU Socket or NUMA
    NV#     Indicates NVLink connection

### 9.3 Operational Focus Points

When performing multi-GPU training, operations need to focus on:

- Whether GPUs are on the same NUMA node;
- Whether NVLink exists between GPUs;
- Topology relationship between GPUs and network cards;
- Whether distributed training spans nodes;
- Whether cross-node communication relies on high-speed network;
- Whether NCCL communication is normal;
- Whether RDMA or RoCE is enabled;
- Whether containers can access corresponding devices.

---

## Ten. Power and Cooling

GPUs are high-power devices.

Data center GPUs may consume very high power when fully loaded.

Operations must focus on:

- Power capacity;
- Cabinet power supply;
- PDU load;
- Airflow;
- Server fans;
- Data center temperature;
- GPU temperature;
- Power limits;
- Whether frequency reduction occurs.

Check GPU temperature and power:

    nvidia-smi

Check more detailed information:

    nvidia-smi -q

Continuous monitoring:

    watch -n 2 nvidia-smi

Common metrics:

    Temperature
    Power Draw
    Power Limit
    Performance State
    Clocks
    Fan Speed

### 10.1 Abnormal GPU Temperature Manifestations

Temperature anomalies may lead to:

- GPU automatic frequency reduction;
- Reduced training speed;
- Node reboots;
- Driver anomalies;
- Task failures in Pods;
- Slow nvidia-smi response;
- Reduced hardware lifespan.

### 10.2 Production Recommendations

Production environment recommendations:

- GPU nodes must be connected to temperature monitoring;
- High-temperature alerts should not rely on single instant values;
- Alerts can be set to trigger after 5 or 10 minutes of exceeding thresholds;
- Need to troubleshoot in combination with data center temperature, fan status, and node location;
- GPU nodes should not operate long-term near power limits;
- Pressure testing should be done before deployment.

---

## Eleven. GPU Recognition Chain in Linux Systems

A Linux server recognizes a GPU through multiple layers.

Basic chain:

    Physical GPU
      ↓
    BIOS / UEFI Initialization
      ↓
    PCIe Device Enumeration
      ↓
    Linux Kernel Recognizes PCI Device
      ↓
    NVIDIA Driver Loading
      ↓
    nvidia-smi Can View GPU
      ↓
    CUDA Runtime Can Access GPU
      ↓
    Container Runtime Can Mount GPU Device
      ↓
    Kubernetes Device Plugin Registers GPU
      ↓
    Pod Requests nvidia.com/gpu to Use GPU

If any layer has issues, upper layers may become unavailable.

### 11.1 Check if PCIe Recognizes GPU

    lspci | grep -i nvidia

If no GPU is visible here, prioritize suspecting:

- GPU not properly inserted;
- BIOS configuration anomalies;
- PCIe slot issues;
- Motherboard incompatibility;
- Insufficient power supply;
- GPU hardware failure.

### 11.2 Check if Driver is Normal

    nvidia-smi

If lspci can see GPU but nvidia-smi is abnormal, prioritize suspecting:

- NVIDIA driver not installed;
- Driver version mismatch;
- Kernel module not loaded;
- Secure Boot blocking driver module loading;
- DKMS build failure;
- Driver and kernel version incompatibility.

Check kernel modules:

    lsmod | grep nvidia

Check driver-related logs:

    dmesg | grep -i nvidia
    journalctl -k | grep -i nvidia

### 11.3 Check if CUDA is Normal

    nvcc -V

Notes:

- The CUDA Version shown by nvidia-smi does not necessarily mean the full CUDA Toolkit is installed;
- The CUDA Version shown by nvidia-smi is the highest CUDA Runtime version supported by the driver;
- nvcc -V depends on CUDA Toolkit;
- Whether CUDA can be used in containers also depends on image and runtime configuration.

---

## Twelve. Role of GPU in Kubernetes /think

</think>

## Twelve. Role of GPU in Kubernetes

GPU serves as a critical compute resource in Kubernetes environments, enabling high-performance computing tasks. Proper integration of GPU resources into Kubernetes requires careful configuration of several components:

1. **Node-level GPU recognition**: Ensuring the GPU is properly detected at the hardware and driver level.
2. **Driver installation**: Installing and maintaining the NVIDIA driver to enable GPU functionality.
3. **CUDA Toolkit**: Providing the necessary libraries and tools for GPU-accelerated computing.
4. **Kubernetes Device Plugin**: Registering GPU resources with the Kubernetes API for scheduling and resource management.
5. **Pod scheduling**: Allowing workloads to request and utilize GPU resources effectively.

The full GPU utilization chain in Kubernetes includes:

- **Physical GPU** → **BIOS/UEFI initialization** → **PCIe enumeration** → **Linux kernel PCI device recognition** → **NVIDIA driver loading** → **nvidia-smi visibility** → **CUDA runtime access** → **Container runtime GPU device mounting** → **Kubernetes Device Plugin GPU registration** → **Pod GPU resource request**.

Any failure in this chain can prevent GPU resources from being utilized by workloads.

Kubernetes can natively manage CPU and memory, but cannot inherently manage NVIDIA GPUs.

GPUs are typically integrated into Kubernetes through the following components:

- NVIDIA Driver
- NVIDIA Container Toolkit
- NVIDIA Device Plugin
- NVIDIA GPU Operator
- kubelet
- Scheduler

### 12.1 How GPU Becomes a Kubernetes Resource

Simplified flow:

    GPU hardware
      ↓
    NVIDIA Driver
      ↓
    NVIDIA Container Toolkit
      ↓
    NVIDIA Device Plugin
      ↓
    kubelet
      ↓
    nvidia.com/gpu appears in Node Status
      ↓
    Pod requests GPU via resources.limits
      ↓
    Scheduler schedules Pod to GPU node

Check if node has registered GPU:

    kubectl describe node <gpu-node-name>

Focus on:

    Capacity:
      nvidia.com/gpu: 1

    Allocatable:
      nvidia.com/gpu: 1

If nvidia.com/gpu is missing, Kubernetes has not yet recognized the GPU.

### 12.2 How Pod Requests GPU

Pod example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: gpu-test
      namespace: default
    spec:
      restartPolicy: Never
      containers:
        - name: cuda-container
          image: nvidia/cuda:12.2.0-base-ubuntu22.04
          command: ["nvidia-smi"]
          resources:
            limits:
              nvidia.com/gpu: 1

Notes:

- GPU resources are typically only specified with limits;
- Kubernetes requires requests and limits to be equal for extended resources;
- If only limits are specified, Kubernetes usually automatically sets requests to the same value;
- GPU cannot be specified like CPU with 500m;
- nvidia.com/gpu: 1 indicates requesting 1 full GPU;
- Shared GPU requires additional solutions, such as MIG, time slicing, or specific scheduling schemes.

---

## Thirteen, Common Labels and Taints for GPU Nodes

In production environments, GPU nodes typically should not run mixed with regular business Pods.

Common practice is to label GPU nodes:

    kubectl label node <gpu-node-name> node-role.kubernetes.io/gpu=true
    kubectl label node <gpu-node-name> accelerator=nvidia

Taints can also be applied to prevent regular Pods from scheduling:

    kubectl taint node <gpu-node-name> nvidia.com/gpu=true:NoSchedule

GPU Pods need to add tolerations:

    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"

This can be combined with nodeSelector:

    nodeSelector:
      accelerator: nvidia

Production recommendations:

- GPU nodes should be distinctly identified;
- Regular business Pods should not be arbitrarily scheduled to GPU nodes;
- GPU nodes are resource-intensive, so avoid unrelated Pods consuming CPU, memory, disk, and network;
- GPU inference services can be planned in dedicated node pools;
- GPU training tasks can be planned in dedicated node pools;
- If it's a multi-tenant platform, consider strategies like Namespace, ResourceQuota, PriorityClass, etc.

---

## Fourteen, Core Metrics in GPU Operations

At least the following metrics should be monitored for GPU operations.

### 14.1 GPU Utilization

Represents the busyness of GPU computing units.

Common command:

    nvidia-smi

Metrics example:

    GPU-Util

Prometheus metrics may come from DCGM Exporter, such as:

    DCGM_FI_DEV_GPU_UTIL

Focus points:

- Low GPU utilization over time may indicate tasks not fully utilizing GPU;
- High GPU utilization over time isn't necessarily abnormal, it could be normal for training tasks;
- High GPU utilization with high business latency requires analysis with memory, CPU, network, and logs;
- Low GPU utilization with high memory usage may indicate model loading idle or task blocking on data loading.

### 14.2 Memory Usage

Command:

    nvidia-smi

Focus on:

    Memory-Usage

Prometheus metrics may include:

    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_FB_FREE

Common issues:

- Memory occupied by processes but tasks are already abnormal;
- Memory contention when multiple containers share GPU;
- Memory pressure from loading multiple models in inference services;
- Large batch size causing OOM;
- Models not releasing memory.

Check occupying processes:

    nvidia-smi

Combine with host processes when necessary:

    ps -ef | grep <PID>

In Kubernetes, also trace back to Pod:

    crictl ps
    crictl inspect <container-id>
    kubectl get pod -A -o wide | grep <node-name>

### 14.3 Temperature

Command:

    nvidia-smi

Focus on:

    Temp

Production alert recommendations:

- Warning: Continuous 5 minutes over 75°C;
- Critical: Continuous 5 minutes over 85°C;
- Thresholds should be adjusted based on GPU model, manufacturer recommendations, and data center environment.

### 14.4 Power Consumption

Command:

    nvidia-smi

Focus on:

    Power Draw
    Power Limit

Potential issues:

- Power consumption approaching the upper limit causing throttling;
- Power module pressure too high;
- Cabinet power supply insufficient;
- Instability after node full load.

### 14.5 XID Errors

XID errors often occur in NVIDIA driver anomalies.

Check:

    dmesg | grep -i xid
    journalctl -k | grep -i xid

XID may indicate:

- Driver anomalies;
- GPU hardware errors;
- PCIe issues;
- VRAM errors;
- Application-triggered illegal access;
- Temperature or power supply issues.

Production recommendations:

- Occasional XID requires recording;
- Repeated XID requires focused attention;
- XID with Pod anomalies needs time correlation;
- Severe XID may require node isolation, node restart, or hardware replacement.

---

## FifteenI don't know.GPU Topology Diagram

### 15.1 Single Node Single GPU

    +-----------------------------+
    |        GPU Node             |
    |                             |
    |  +-----------------------+  |
    |  | CPU / Memory          |  |
    |  +-----------------------+  |
    |             |               |
    |           PCIe              |
    |             |               |
    |  +-----------------------+  |
    |  | NVIDIA GPU            |  |
    |  | VRAM / CUDA / Tensor  |  |
    |  +-----------------------+  |
    |                             |
    +-----------------------------+

Suitable for:

- Learning environment;
- Small model inference;
- Single machine testing;
- GPU operation and maintenance basics experiments.

### 15.2 Single Node Multiple GPUs

    +--------------------------------------------------+
    |                    GPU Node                      |
    |                                                  |
    |   +-------------+        +-------------------+    |
    |   | CPU/Memory  |--------| PCIe Switch       |    |
    |   +-------------+        +-------------------+    |
    |                              |       |       |     |
    |                            GPU0    GPU1    GPU2   |
    |                              |       |       |     |
    |                         GPU Memory / CUDA         |
    |                                                  |
    +--------------------------------------------------+

Focus points:

- PCIe topology;
- NUMA affinity;
- Multi-card communication;
- Distance from GPU to network card;
- Support for NVLink;
- Power and cooling.

### 15.3 Kubernetes GPU Cluster

    +-----------------------------+
    |      Control Plane           |
    |  apiserver / scheduler       |
    |  controller-manager / etcd   |
    +---------------+-------------+
                    |
        Kubernetes API
                    |
    +---------------+-------------------------------+
    |                                               |
+---+-------------------+             +-------------+-------------+
| CPU Worker Node       |             | GPU Worker Node           |
|                       |             |                           |
| common pods           |             | nvidia driver             |
| node-exporter         |             | nvidia-container-toolkit   |
| kubelet               |             | device-plugin             |
| containerd            |             | dcgm-exporter             |
|                       |             | gpu pods                  |
+-----------------------+             +---------------------------+

---

## SixteenI don't know.GPU Node First-Level Inspection Commands

### 16.1 Check if system recognizes NVIDIA devices

    lspci | grep -i nvidia

### 16.2 Check GPU driver status

    nvidia-smi

### 16.3 Check GPU detailed information

    nvidia-smi -q

### 16.4 Check GPU topology

    nvidia-smi topo -m

### 16.5 Check driver modules

    lsmod | grep nvidia

### 16.6 Check kernel logs

```
dmesg | grep -i nvidia
dmesg | grep -i xid
journalctl -k | grep -i nvidia

### 16.7 Checking Kubernetes Node Resources

    kubectl describe node <gpu-node-name>

Key Focus Areas:

    Capacity
    Allocatable
    nvidia.com/gpu
    Taints
    Labels
    Conditions
    Events

### 16.8 Checking GPU-Related Pods

If using GPU Operator:

    kubectl get pods -n gpu-operator -o wide
    kubectl get pods -n gpu-operator-resources -o wide

If using NVIDIA Device Plugin:

    kubectl get daemonset -A | grep -i nvidia
    kubectl get pods -A | grep -i nvidia

### 16.9 Checking GPU Pod Scheduling Status

    kubectl get pods -A -o wide | grep -i gpu
    kubectl describe pod <gpu-pod-name> -n <namespace>

---

## Seventeen, GPU Fault Layered Troubleshooting Approach

Do not reinstall drivers immediately when encountering GPU issues.

Recommend troubleshooting by layers.

### 17.1 First Layer: Hardware Layer

Checkpoints:

- Is the GPU properly inserted?
- Is the power cable connected?
- Is the PCIe slot functioning normally?
- Does the server support this GPU?
- Does the chassis cooling meet requirements?
- Is the GPU recognized by BIOS?
- Is Above 4G Decoding required to be enabled?

Commands:

    lspci | grep -i nvidia
    dmesg | grep -i pci

If lspci cannot detect the GPU, prioritize checking hardware and BIOS.

### 17.2 Second Layer: Driver Layer

Checkpoints:

- Is NVIDIA driver installed?
- Are kernel modules loaded?
- Does Secure Boot affect module loading?
- Is the driver version compatible with the current GPU?
- Is the driver compatible with the kernel version?

Commands:

    nvidia-smi
    lsmod | grep nvidia
    dmesg | grep -i nvidia

If lspci can detect the GPU but nvidia-smi fails, prioritize checking the driver.

### 17.3 Third Layer: CUDA Layer

Checkpoints:

- Is CUDA Toolkit installed?
- Is CUDA Runtime available?
- Does the application's required CUDA version match?
- Does the container image's CUDA version match the host driver?
- Can PyTorch / TensorFlow recognize GPU?

Commands:

    nvcc -V
    python -c "import torch; print(torch.cuda.is_available())"

Notes:

    nvidia-smi being normal does not guarantee Python frameworks can use GPU.
    nvcc not existing does not mean the driver is unavailable.
    Compatibility between container image CUDA Runtime and host driver is critical.

### 17.4 Fourth Layer: Container Runtime Layer

Checkpoints:

- Is NVIDIA Container Toolkit installed?
- Is NVIDIA runtime configured for containerd / Docker?
- Can the container access /dev/nvidia* devices?
- Does the container image include correct CUDA runtime libraries?

Command Examples:

    ls -l /dev/nvidia*

If testing with Docker:

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

If using containerd, check with cluster runtime configuration.

### 17.5 Fifth Layer: Kubernetes Layer

Checkpoints:

- Is Device Plugin running?
- Does kubelet register nvidia.com/gpu?
- Does the node have GPU allocatable?
- Does the Pod correctly declare resources.limits?
- Do NodeSelector / Affinity / Toleration match?
- Is the GPU already occupied by another Pod?

Commands:

    kubectl describe node <gpu-node-name>
    kubectl get pods -A -o wide
    kubectl describe pod <gpu-pod-name> -n <namespace>

---

## Eighteen, Common Misconceptions in GPU Operations

### 18.1 Misconception 1: nvidia-smi being normal means the GPU environment is normal

Not necessarily.

nvidia-smi being normal only indicates:

- The system recognizes the GPU;
- The driver is generally functional;
- NVIDIA management interface is accessible.

But it does not indicate:

- CUDA Toolkit is definitely installed;
- The container environment is definitely accessible;
- Kubernetes definitely recognizes GPU;
- Device Plugin is definitely functional;
- Pod scheduling is definitely possible;
- AI frameworks can definitely use GPU.

### 18.2 Misconception 2: Low GPU utilization means resource waste

Not necessarily.

Low GPU utilization could be:

- Low business request volume;
- Inference service during off-peak hours;
- Too small batch size;
- Slow CPU data preprocessing;
- Slow data loading;
- Network bottleneck;
- Model structure not suitable for parallelism;
- Application not properly using GPU.

Need to combine with:

- QPS;
- Latency;
- GPU memory usage;
- CPU utilization;
- Data loading time;
- Application logs;
- Prometheus metrics;
- Actual business traffic.

### 18.3 Misconception 3: High GPU memory usage means abnormality

Not necessarily.

Model memory may be persistently occupied after loading.

For example, after inference service startup, model residency in GPU memory is normal.

Abnormality is determined by:

- Whether GPU memory continuously increases;
- Whether OOM occurs;
- Whether it affects new task startup;
- Whether zombie processes exist;
- Whether multiple Pods compete for the same GPU;
- Whether business has stopped but memory is not released.

### 18.4 Misconception 4: GPU Pod Pending means insufficient GPU

Not necessarily.

GPU Pod Pending may be caused by: /think
```

- Insufficient GPU;
- Device Plugin is not running;
- Node is not registered with nvidia.com/gpu;
- Node is tainted;
- Pod does not have toleration;
- nodeSelector does not match;
- affinity is not satisfied;
- Image pull failure;
- CPU / Memory insufficient;
- Namespace ResourceQuota limit;
- RuntimeClass configuration anomaly.

So first check:

    kubectl describe pod <pod-name>

Do not directly assume it's due to insufficient GPU.

---

## Nineteen. Production Environment GPU Node Baseline Recommendations

### 19.1 Hardware Baseline

Recommended to record:

- Server model;
- GPU model;
- Number of GPUs;
- VRAM capacity;
- PCIe topology;
- Network card model;
- RDMA support;
- Power capacity;
- BIOS version;
- BMC/IPMI address;
- Cabinet location;
- Asset number.

### 19.2 System Baseline

Recommended to record:

- Operating system version;
- Kernel version;
- NVIDIA Driver version;
- CUDA version;
- Container Runtime version;
- Kubernetes version;
- NVIDIA Container Toolkit version;
- Device Plugin / GPU Operator version.

### 19.3 Kubernetes Baseline

Recommended planning:

- GPU node labels;
- GPU node taints;
- GPU Namespace;
- ResourceQuota;
- LimitRange;
- PriorityClass;
- RuntimeClass;
- GPU monitoring;
- GPU alerts;
- Log collection;
- Node maintenance process.

### 19.4 Monitoring Baseline

At least monitor:

- GPU utilization;
- GPU VRAM usage;
- GPU temperature;
- GPU power consumption;
- GPU ECC errors;
- GPU XID errors;
- GPU Pod Pending;
- GPU Pod Restart;
- Device Plugin status;
- DCGM Exporter status;
- GPU node Ready status.

---

## Twenty. GPU Node Delivery Checklist

Before GPU node delivery, recommended checks:

    [ ] BIOS has completed basic configuration
    [ ] System can identify GPU via lspci
    [ ] NVIDIA driver installation completed
    [ ] nvidia-smi output is normal
    [ ] GPU temperature and power consumption are normal
    [ ] CUDA basic tests passed
    [ ] Can execute nvidia-smi inside container
    [ ] Kubernetes node is Ready
    [ ] Device Plugin or GPU Operator is normal
    [ ] kubectl describe node can see nvidia.com/gpu
    [ ] GPU test Pod can run successfully
    [ ] GPU metrics are integrated with Prometheus
    [ ] Grafana can view GPU dashboard
    [ ] GPU alert rules are configured
    [ ] GPU node labels and taints are set
    [ ] Regular Pod will not be mis-scheduled to GPU node
    [ ] GPU node maintenance and restart process is clear

---

## Twenty-one. Summary

GPU is a core computing resource in AI infrastructure.

From an operations perspective, GPU management is not a single-point issue, but a complete end-to-end chain:

    Hardware
    -> BIOS
    -> PCIe
    -> Linux Kernel
    -> NVIDIA Driver
    -> CUDA
    -> Container Runtime
    -> NVIDIA Container Toolkit
    -> Kubernetes Device Plugin
    -> kubelet
    -> Scheduler
    -> GPU Pod
    -> Prometheus / DCGM Exporter
    -> Grafana / AlertManager
    -> Logs and automated troubleshooting

When learning GPU operations, one should not just memorize commands, but establish a layered perspective.

When encountering issues, troubleshoot in the following order:

    1. Does lspci show GPU?
    2. Is nvidia-smi normal?
    3. Is CUDA available?
    4. Can GPU be seen inside container?
    5. Is nvidia.com/gpu registered on Kubernetes Node?
    6. Is Device Plugin normal?
    7. Does Pod correctly request GPU?
    8. Are scheduling constraints matched?
    9. Can monitoring and logs associate with the issue?

Only by managing GPU as a "high-value computing resource" rather than a regular graphics card, can you support subsequent AI inference platforms, training platforms, Kubernetes GPU clusters, and production-grade observability systems.

---

## Reference Documents

- NVIDIA Official Documentation: https://docs.nvidia.com/
- NVIDIA CUDA Toolkit Documentation: https://docs.nvidia.com/cuda/
- NVIDIA Data Center GPU Documentation: https://docs.nvidia.com/datacenter/
- Kubernetes GPU Scheduling: https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/
- NVIDIA Kubernetes Device Plugin: https://github.com/NVIDIA/k8s-device-plugin
- NVIDIA GPU Operator: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/
- NVIDIA DCGM Exporter: https://github.com/NVIDIA/dcgm-exporter