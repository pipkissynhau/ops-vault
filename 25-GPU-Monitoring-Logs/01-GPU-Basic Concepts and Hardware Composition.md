# 01-GPU-Basic Concepts and Hardware Composition

## Document Description

This document aims to establish a foundational understanding for GPU operations, GPU node management, Kubernetes GPU scheduling, and the operation and maintenance of AI inference/training platforms.

This document does not directly cover the installation of GPU drivers, CUDA, or Device Plugins, nor the implementation of Prometheus monitoring. Instead, it addresses several fundamental questions:

- What exactly is a GPU?
- Why are GPUs suitable for AI training and inference?
- What key hardware components make up a GPU?
- Which metrics should operations engineers pay attention to when managing GPUs?
- How do GPUs become schedulable resources within Kubernetes clusters?
- What should be checked first when issues arise with GPU nodes?

Mastering this content will make it easier to proceed with the following topics:

- 02-GPU-BIOS and Hardware Tuning
- 03-NVIDIA-Driver Installation and Verification
- 04-CUDA-Installation and Testing
- 05-K8S-GPU-Resource Concepts and Scheduling Principles
- 06-NVIDIA-Device-Plugin-and-Operator-Installation
- 07-GPU-Pod-Deployment and Scheduling Practices
- 08-GPU-Monitoring and Alarm Integration

## Tags

#GPU #NVIDIA #CUDA #Kubernetes #AIInfrastructure #OperationsTroubleshooting #CloudNative #SRE

## Recommended Reading Path

Recommended path:

    06-GPU and AI Infrastructure/01-GPU Basics/01-GPU-Basic Concepts and Hardware Composition.md

---

## I. Why Operations Engineers Need to Understand GPUs

In traditional operations scenarios, servers mainly focus on CPU, memory, disk, and network resources.

However, in applications such as AI training, inference, video rendering, scientific computing, and high-performance computing, GPUs have become essential computing resources.

For operations engineers, a GPU is not just a "graphics card" but a type of high-value, highly dependent, state-sensitive computing resource with strict performance requirements.

Common issues in GPU operations include:

- The server can detect the GPU, but it is not visible inside containers;
- nvidia-smi appears normal, but Kubernetes Pods cannot schedule GPUs;
- A Pod has requested GPU resources via nvidia.com/gpu, but it remains pending;
- The GPU utilization rate is low, but the video memory usage is high;
- Excessive GPU temperature causes frequency reduction or node failures;
- Mismatched driver versions, CUDA versions, or container image versions;
- Poor communication between multiple GPUs results in suboptimal training performance;
- High costs for GPU nodes, yet unclear resource utilization rates;
- Lack of GPU metrics integrated with Prometheus, making capacity planning and alarm settings difficult.

Therefore, GPU operations involves more than just executing commands like `nvidia-smi`. It requires a deep understanding of various aspects, including:

- Hardware identification
  - -> BIOS/PCIe initialization
  - -> NVIDIA driver installation
  - -> CUDA Runtime configuration
  - -> Container runtime management
  - -> NVIDIA Container Toolkit usage
  - -> Kubernetes Device Plugin integration
  - -> kubelet resource registration
  - -> Scheduler configuration
  - -> Pod GPU allocation
  - -> Metric collection and log analysis for troubleshooting

This represents a comprehensive process involving multiple components.- Whether the inference service uses TensorRT will affect throughput;
- Quantizing the model may significantly reduce memory usage and increase inference speed.      ↓
    NVIDIA Driver
      ↓
    NVIDIA Container Toolkit
      ↓
    NVIDIA Device Plugin
      ↓
    kubelet
      ↓
    The "nvidia.com/gpu" entry appears in Node Status
      ↓
    Pods request GPUs through resources.limits
      ↓
    The Scheduler schedules Pods to GPU nodes

To check if a node has registered a GPU:

    kubectl describe node <gpu-node-name>

Pay special attention to the following fields:

    Capacity:
      nvidia.com/gpu: 1

    Allocatable:
      nvidia.com/gpu: 1

If "nvidia.com/gpu" is not listed, it means Kubernetes has not yet detected the GPU.

### 12.2 How Pods Request GPUs

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

- GPU resources are typically specified only in the limits field;
- Kubernetes requires that requests and limits for expandable resources be equal;
- If only limits are provided, Kubernetes usually sets requests to the same value automatically;
- GPUs cannot be configured with values like 500m like CPUs can;
- "nvidia.com/gpu: 1" indicates requesting one full GPU;
- Sharing GPUs requires additional solutions such as MIG, time slicing, or specialized scheduling strategies.

---

## Section Thirteen: Common Labels and Taints for GPU Nodes

In production environments, it is generally not recommended to mix GPU nodes with regular business Pods.

Common practices include labeling GPU nodes:

    kubectl label node <gpu-node-name> node-role.kubernetes.io/gpu=true
    kubectl label node <gpu-node-name> accelerator=nvidia

Tainting nodes can also prevent regular Pods from being scheduled on them:

    kubectl taint node <gpu-node-name> nvidia.com/gpu=true:NoSchedule

GPU Pods require tolerations to ensure they can be scheduled correctly:

    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"

Node selectors can also be used for additional control:

    nodeSelector:
      accelerator: nvidia

Production recommendations include:

- Identifying GPU nodes separately;
- Preventing regular Pods from scheduling on GPU nodes;
- Considering that GPU nodes are expensive resources and should not be unnecessarily occupied by unrelated Pods consuming CPU, memory, disk, or network bandwidth;
- Planning dedicated node pools for GPU inference services and training tasks;
- In multi-tenant environments, using strategies such as Namespaces, ResourceQuotas, and PriorityClasses to manage resource allocation.

---

## Section Fourteen: Core Metrics for GPU Operations and Maintenance

At least the following metrics should be monitored regularly for GPU operations and maintenance.

### 14.1 GPU Utilization

This indicates how busy the GPU's computing units are.

Common command:

    nvidia-smi

Example metric:

    GPU-Util

Metrics available in Prometheus may come from the DCGM Exporter, such as:

    DCGM_FI_DEV_GPU_UTIL

Key considerations:

- If GPU utilization is consistently low, it may indicate that tasks are not making full use of the GPU;
- High GPU utilization is not necessarily abnormal; it could mean that training tasks are running at full capacity;
- High GPU utilization combined with high latency may require further analysis of memory usage, CPU performance, network issues, and logs;
- Low GPU utilization but high memory usage might suggest that the model is idle after loading or that tasks are stuck during data loading.

### 14.2 Memory Usage

Command:

    nvidia-smi

Key focus:

    Memory-Usage

Prometheus metrics may include:

    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_FB_FREE

Common issues:

- Processes consuming memory but causing task failures;
- Memory contention when multiple containers share a GPU;
- Inference services loading multiple models leading to high memory usage;
- Out-of-memory errors (OOM) due to excessive batch sizes;
- Models not releasing allocated memory.

To identify memory-consuming processes:

    nvidia-smi

If necessary, check the host process list:

    ps -ef | grep <PID>

In Kubernetes, you can also inspect related Pods:

    crictl ps
    crictl inspect <container-id>
    kubectl get pod -A -o wide | grep <node-name>

### 14.3 Temperature

Command:

    nvidia-smi

Key consideration:

    Temp

Production alert thresholds:

- Warning: Temperature exceeds 75°C- Multi-card communication;
- Distance between the GPU and the network card;
- Support for NVLink;
- Power supply and cooling.

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

## Sixteen, First-Level Inspection Commands for GPU Nodes

### 16.1 Check if the system recognizes NVIDIA devices

    lspci | grep -i nvidia

### 16.2 Check the status of the GPU driver

    nvidia-smi

### 16.3 View detailed information about the GPU

    nvidia-smi -q

### 16.4 View the GPU topology

    nvidia-smi topo -m

### 16.5 View the driver modules

    lsmod | grep nvidia

### 16.6 Check the kernel logs

    dmesg | grep -i nvidia
    dmesg | grep -i xid
    journalctl -k | grep -i nvidia

### 16.7 View Kubernetes node resources

    kubectl describe node <gpu-node-name>

Focus on:

    Capacity
    Allocatable
    nvidia.com/gpu
    Taints
    Labels
    Conditions
    Events

### 16.8 Check GPU-related Pods

If using the GPU Operator:

    kubectl get pods -n gpu-operator -o wide
    kubectl get pods -n gpu-operator-resources -o wide

If using the NVIDIA Device Plugin:

    kubectl get daemonset -A | grep -i nvidia
    kubectl get pods -A | grep -i nvidia

### 16.9 Check the scheduling status of GPU Pods

    kubectl get pods -A -o wide | grep -i gpu
    kubectl describe pod <gpu-pod-name> -n <namespace>

---

## Seventeen, Layered Troubleshooting Approach for GPU Failures

Do not immediately reinstall the driver when a GPU failure occurs.

It is recommended to troubleshoot layer by layer.

### 17.1 First Layer: Hardware Layer

Checkpoints:

- Is the GPU properly inserted?
- Are the power cables connected?
- Is the PCIe slot functioning correctly?
- Does the server support this GPU?
- Does the chassis cooling meet requirements?
- Is it recognized by the BIOS?
- Is "Above 4G Decoding" needed to be enabled?

Commands:

    lspci | grep -i nvidia
    dmesg | grep -i pci

If the GPU is not visible in lspci, first check the hardware and BIOS.

### 17.2 Second Layer: Driver Layer

Checkpoints:

- Is the NVIDIA driver installed?
- Are the kernel modules loaded?
- Does Secure Boot affect module loading?
- Is the driver version compatible with the current GPU?
- Is the driver compatible with the kernel version?

Commands:

    nvidia-smi
    lsmod | grep nvidia
    dmesg | grep -i nvidia

If the GPU is visible in lspci but nvidia-smi fails, first check the driver.

### 17.3 Third Layer: CUDA Layer

Checkpoints:

- Is the CUDA Toolkit installed?
- Is the CUDA Runtime available?
- Does the CUDA version required by the application match?
- Does the CUDA version in the container image match the host driver?
- Can PyTorch / TensorFlow recognize the GPU?

Commands:

    nvcc -V
    python -c "import torch; print(torch.cuda.is_available())"

Note:

- A normal nvidia-smi does not guarantee that Python frameworks can use the GPU.
- The absence of nvcc does not necessarily mean the driver is unavailable.
- Compatibility between the CUDA Runtime in the container image and the host driver is crucial.

### 17.4 Fourth Layer: Container Runtime Layer

Checkpoints:

- Is the NVIDIA Container Toolkit installed?
- Are containerd / Docker configured with the NVIDIA runtime?
- Can containers access /dev/nvidia* devices?
- Does the container image contain the correct CUDA runtime libraries?

Command examples:

    ls -l /dev/nvidia*

If using Docker for testing:

    docker run --rm --gpus all nvidia/cuda:1---

## Section 19: Baseline Recommendations for Production Environment GPU Nodes

### 19.1 Hardware Baseline

It is recommended to record the following:

- Server model;
- GPU model;
- Number of GPUs;
- Video memory capacity;
- PCIe topology;
- Network card model;
- Support for RDMA;
- Power supply capacity;
- BIOS version;
- BMC/IPMI address;
- Rack location;
- Asset number.

### 19.2 System Baseline

It is recommended to record the following:

- Operating system version;
- Kernel version;
- NVIDIA Driver version;
- CUDA version;
- Container Runtime version;
- Kubernetes version;
- NVIDIA Container Toolkit version;
- Device Plugin/GPU Operator version.

### 19.3 Kubernetes Baseline

It is recommended to plan the following:

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
- Node maintenance procedures.

### 19.4 Monitoring Baseline

At least monitor the following:

- GPU utilization;
- GPU video memory usage;
- GPU temperature;
- GPU power consumption;
- GPU ECC errors;
- GPU XID errors;
- GPU Pod Pending status;
- GPU Pod Restart status;
- Device Plugin status;
- DCGM Exporter status;
- GPU Node Ready status.

---

## Section 20: GPU Node Delivery Checklist

Before delivering a GPU node, it is recommended to check the following:

    [ ] Basic BIOS configuration is completed.
    [ ] The system can identify GPUs through lspci.
    [ ] NVIDIA Driver installation is complete.
    [ ] nvidia-smi outputs normally.
    [ ] GPU temperature and power consumption are within normal ranges.
    [ ] Basic CUDA tests pass.
    [ ] nvidia-smi can be executed inside containers.
    [ ] The Kubernetes node is marked as Ready.
    [ ] Device Plugin or GPU Operator is functioning correctly.
    [ ] kubectl describe node shows nvidia.com/gpu information.
    [ ] GPU test Pods can run successfully.
    [ ] GPU metrics are integrated into Prometheus.
    [ ] GPU panels can be viewed in Grafana.
    [ ] GPU alert rules have been configured.
    [ ] GPU node labels and taints have been set.
    [ ] Ordinary Pods are not mis-scheduled to GPU nodes.
    [ ] Maintenance and restart procedures for GPU nodes are clear.

---

## Section 21: Summary

GPUs are core computing resources in AI infrastructure.

From an operations perspective, managing GPUs is not a single issue but involves a complete chain of components:

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
    -> GPU Pods
    -> Prometheus/DCGM Exporter
    -> Grafana/AlertManager
    -> Logging and automated troubleshooting

When learning about GPU operations, it is important to adopt a layered approach rather than just memorizing commands. When encountering issues, follow this sequence for troubleshooting:

    1. Can GPUs be seen through lspci?
    2. Does nvidia-smi function correctly?
    3. Is CUDA available?
    4. Can GPUs be detected inside containers?
    5. Has the Kubernetes Node registered nvidia.com/gpu?
    6. Is the Device Plugin working properly?
    7. Do Pods request GPUs correctly?
    8. Are scheduling constraints matching?
    9. Can monitoring and logs help identify issues?

Only by treating GPUs as "high-value computing resources" rather than ordinary graphics cards can we ensure the success of subsequent AI inference platforms, training systems, Kubernetes GPU clusters, and production-grade observability frameworks.

---

## References

- NVIDIA Official Documentation: https://docs.nvidia.com/
- NVIDIA CUDA Toolkit Documentation: https://docs.nvidia.com/cuda/
- NVIDIA Data Center GPU Documentation: https://docs.nvidia.com/datacenter/
- Kubernetes GPU Scheduling: https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/
- NVIDIA Kubernetes Device Plugin: https://github.com/NVIDIA/k8s-device-plugin
- NVIDIA GPU Operator: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/
- NVIDIA DCGM Exporter: https://github.com/NVIDIA/dcgm-exporter