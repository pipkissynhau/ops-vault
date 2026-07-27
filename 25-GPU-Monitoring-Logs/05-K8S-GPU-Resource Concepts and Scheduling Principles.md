# 05-K8S-GPU-Resource Concepts and Scheduling Principles

## Documentation Overview

This document outlines the fundamental concepts of GPU resources in Kubernetes, including the resource registration process, scheduling mechanisms, methods for Pod requests, the design of node labels and taints, Namespace-based resource quotas, common reasons for scheduling failures, and strategies for planning GPU scheduling in production environments.

This article addresses the following key questions:

- Why can't Kubernetes directly schedule NVIDIA GPUs by default?
- How does `nvidia.com/gpu` appear in Node resources?
- What role does the NVIDIA Device Plugin play in the scheduling process?
- What is the relationship between kubelet, Device Plugin, Scheduler, and Pods?
- Why are GPUs classified as Extended Resources?
- Why are only `limits` specified for GPU resources?
- Why can't we specify values like `500m` for GPUs, unlike CPUs?
- How does the Scheduler select nodes when a Pod requests a GPU?
- How should one troubleshoot issues when a GPU Pod is in a Pending state?
- Why are labels and taints applied to GPU nodes?
- How can Namespaces limit the usage of GPU resources?
- What are the boundaries between different types of GPUs, MIGs, shared GPUs, and time-slicing?
- How should one plan GPU node pools and scheduling strategies in production environments?

This document does not delve into the installation of NVIDIA drivers, CUDA, Device Plugins, or GPU Operators. Related information can be found in the following sections:

- 03-NVIDIA-Driver Installation and Verification
- 04-CUDA-Installation and Testing
- 06-NVIDIA-Device-Plugin-and-Operator-Installation
- 07-GPU-Pod-Deployment and Scheduling Practices
- 08-GPU-Monitoring and Alert Integration

---

## Tags

#Kubernetes #GPU #NVIDIA #DevicePlugin #Scheduler #ExtendedResource #AIInfrastructure #CloudNative #SRE #OpsTroubleshooting

---

## Recommended Reading Path

Recommended reading path:

    06-GPUandAIInfrastructure/02-Kubernetes-GPUScheduling/05-K8S-GPU-ResourceConceptsandSchedulingPrinciples.md

---

## I. Why Does Kubernetes Need Special Mechanisms to Manage GPUs

The most common resources in Kubernetes are:

    CPU
    Memory
    Ephemeral Storage

For example, a regular Pod can request CPU and memory as follows:

    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"

CPU and memory are built-in Kubernetes resources.

kubelet can directly obtain the CPU and memory capabilities of a node through the operating system and cgroups, and report this information to the apiserver.

However, GPUs are different.

By default, Kubernetes does not know how many NVIDIA GPUs are on a server, nor does it know about the GPU's health status, device path, driver capability, or how containers can mount the GPU.

Therefore, GPUs need to be integrated into Kubernetes through a device plugin mechanism.

In Kubernetes, NVIDIA GPUs are typically represented as Extended Resources:

    nvidia.com/gpu

Pods request GPUs by specifying this resource.

Example:

    resources:
      limits:
        nvidia.com/gpu: 1

This indicates that the Pod requires 1 GPU.

---

## II. The Complete Integration Process of GPUs in Kubernetes

For a GPU node to be scheduled by Kubernetes, it must go through the following steps:

    Physical GPU
      ↓
    BIOS / PCIe Recognized Correctly
      ↓
    NVIDIA Device Visible via Linux lspci
      ↓
    NVIDIA Driver Functional
      ↓
    nvidia-smi Working Properly
      ↓
    NVIDIA Container Toolkit Operational
      ↓
    kubelet Running Normally
      ↓
    NVIDIA Device Plugin Running as a DaemonSet
      ↓
    Device Plugin Registers GPU Resources with kubelet
      ↓
    kubelet Updates Node Status
      ↓
    `nvidia.com/gpu` Appears in Node Capacity / Allocatable
      ↓
    Pod Declares resources.limits.nvidia.com/gpu
      ↓
    Scheduler Selects a GPU-node Based on Resources and Constraints
      ↓
    kubelet Starts the Pod
      ↓
    Container Runtime Mounts the GPU Device and Driver Libraries
      ↓
    Applications Inside the Container Use CUDA to Access the GPU

If any step in this process fails, the GPU Pod may not function correctly.

---

## III. Differences in Resource Models between GPUs, CPUs, and Memories

### 3.1 CPUs Are Compressible Resources

CPUs can be over-allocated.

For example, a node with 8 cores can run multiple Pods, each requesting a portion of the CPU resources.

CPUs support specifying fractional cores:

    cpu: "500m"

This means**Capacity:**  
The total number of GPUs available on the node.

**Allocatable:**  
The number of GPUs that can be scheduled and used by Pods.

If `Capacity` does not include `nvidia.com/gpu`, it indicates that the GPU resources have not been successfully registered with Kubernetes.

**Common Causes:**  
- The node does not have a NVIDIA GPU;  
- There is an issue with the NVIDIA Driver;  
- `nvidia-smi` is not functioning correctly;  
- The Device Plugin is not running;  
- There are errors in the Device Plugin logs;  
- The kubelet plugin directory is incorrect;  
- The container runtime configuration is flawed;  
- The node is not a Linux GPU node;  
- The GPU has been marked as unavailable during health checks.

---

## VII. How to Request GPU Resources for Pods  

### 7.1 Example of a Minimum GPU Pod  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
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
```

**Explanation:**  
`limits.nvidia.com/gpu: 1` indicates that 1 GPU is requested for this Pod.

**Deployment:**  
```bash
kubectl apply -f gpu-test.yaml
```

**Viewing Results:**  
```bash
kubectl get pod gpu-test -o wide
kubectl logs gpu-test
kubectl describe pod gpu-test
```

### 7.2 Why Only `limits` Are Typically Specified for GPUs  

GPUs are considered expandable resources in Kubernetes. Therefore, only specifying `limits` is sufficient.

For example:  
```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```
Kubernetes will use this limit as the required amount of GPU resources.

If both `requests` and `limits` are specified, they must be equal:  
```yaml
resources:
  requests:
    nvidia.com/gpu: 1
  limits:
    nvidia.com/gpu: 1
```

Specifying only `requests` without `limits` is not allowed, as it violates the rules for GPU resource management.

### 7.3 Why It Is Not Recommended to Specify Decimal Values for GPUs  

By default, Kubernetes schedules GPU resources based on whole numbers of devices. Therefore, specifying a decimal value like `nvidia.com/gpu: 0.5` is incorrect.

If you need to share a GPU among multiple Pods, specialized solutions are required, such as:  
- NVIDIA MIG;  
- NVIDIA Device Plugin time-slicing;  
- NVIDIA MPS;  
- Third-party vGPU scheduling solutions;  
- Custom GPU resource management platforms.

These approaches cannot be achieved simply by specifying a decimal value in `nvidia.com/gpu`.  

---

## VIII. How the Scheduler Schedules GPU Pods  

When scheduling Pods, the Scheduler considers multiple factors. For GPU Pods, these include:  
- Whether the node is ready for use;  
- Whether the node has sufficient CPU and memory resources;  
- Whether the node has enough `nvidia.com/gpu` resources;  
- Whether the node meets the requirements specified by `nodeSelector` and `nodeAffinity`;  
- Whether the node’s taints are tolerated by the Pod;  
- Whether the Namespace ResourceQuota allows further requests for GPU resources;  
- The Pod’s priority and preemption settings;  
- Whether the node satisfies other resource and constraint requirements.

### 8.1 Simplified Scheduling Process  

The simplified process is as follows:  
1. A Pod is created.  
2. The Scheduler reads the Pod’s resource requirements.  
3. It checks if the Pod requires `nvidia.com/gpu: 1`.  
4. It searches for available nodes.  
5. It filters out nodes without GPUs or with insufficient GPU resources.  
6. It further filters out nodes that do not meet other required conditions.  
7. It scores the remaining nodes based on various criteria and selects the most suitable one.  
8. The Pod is assigned to the selected node.  
9. The kubelet starts the container on that node, and the Device Plugin/runtime allocates the GPU device.

### 8.2 The Scheduler Does Not Directly Execute `nvidia-smi`  

The Scheduler does not directly execute `nvidia-smi` on nodes. Instead, it relies on the Node Resource Status stored in the apiserver. If the Device Plugin has not registered the GPU information in the Node Status, the Scheduler will be unable to schedule the GPU Pod correctly.

Therefore, when troubleshooting a pending GPU Pod, you should check:  
```bash
kubectl describe node <gpu-nodeBecause the actual QoS of a GPU Pod is determined by both CPU and memory resources. Simply specifying only GPU resources may lead to unexpected QoS behavior due to insufficient or excessive allocation of other resources. By explicitly defining both CPU and memory requests/limits, you can ensure that the Pod receives an appropriate amount of resources to maintain the desired QoS level.```bash
kubectl get nodes | grep -o "nvidia.com/gpu"
```

### 21.5 排查故障

根据上述信息，逐一排查问题：

- 如果缺少 GPU，增加相应的资源申请；
- 如果节点有污点，清除污点；
- 如果节点不匹配 Pod 的亲和性/选择器，调整它们；
- 检查 CPU 和内存是否足够；
- 如果超过配额，减少资源申请。

---

## 二十二、GPU 调度与容器编排工具的集成

Kubernetes 是一个强大的容器编排工具，但它本身并不直接管理 GPU 资源。

因此，需要与其他工具集成来更好地利用 GPU。

### 22.1 Docker Swarm 和 Kubernetes 的集成

Docker Swarm 支持 GPU 调度，但与 Kubernetes 集成时，需要一些额外的配置和步骤。

### 22.2 KubeVirt 和 Kubernetes 的集成

KubeVirt 是一个虚拟化平台，它可以将 Kubernetes 容器运行在虚拟机上，从而提供更多的资源管理和控制能力，包括 GPU 调度。

### 22.3 AWS ECS 和 Kubernetes 的集成

AWS ECS 是 Amazon 提供的一个容器编排服务，它也支持 GPU 调度，并且可以与 Kubernetes 集成。

### 22.4 GKE on VMware vSphere 和 Kubernetes 的集成

GKE on VMware vSphere 是 Google 提供的一种在 VMware vSphere 上运行 Kubernetes 的方案，它也支持 GPU 调度。

---

## 二十三、GPU 调度与云平台策略

在云平台上使用 GPU 时，还需要考虑一些额外的策略和限制。

### 23.1 成本控制

云平台通常会提供一些成本控制机制，例如按使用量计费、设置上限等，以帮助用户合理控制 GPU 使用成本。

### 23.2 容量规划

根据业务需求和资源可用性，合理规划 GPU 资源的分配和使用情况，以避免浪费和不足。

### 23.3 负载均衡

对于多节点集群，需要考虑如何进行负载均衡，以确保所有节点都能得到公平的 GPU 使用机会。

### 23.4 故障恢复

制定相应的故障恢复策略，以应对可能的 GPU 相关故障，确保业务的连续性和稳定性。### 21.5 Viewing Device Plugins

    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia

To view logs:

    kubectl logs <nvidia-device-plugin-pod> -n <namespace>

The specific Namespace depends on the installation method.

It may be:

    kube-system
    gpu-operator
    gpu-operator-resources
    nvidia-device-plugin

### 21.6 Viewing Node Labels and Taints

Labels:

    kubectl get node <gpu-node-name> --show-labels

Taints:

    kubectl describe node <gpu-node-name> | grep -i taints -A5

### 21.7 Viewing Namespace Quotas

    kubectl describe resourcequota -n <namespace>

### 21.8 Troubleshooting

If the issue is:

    insufficient nvidia.com/gpu

Check first:

- Whether the node has a GPU;
- Whether the Device Plugin is registered;
- Whether the GPU is already occupied by another Pod;
- Whether the Namespace quota is full;
- Whether the Pod is requesting too many GPUs.

If the issue is:

    untolerated taint

Check:

- The taint on the GPU node;
- The Pod's tolerations.

If the issue is:

    didn't match node selector

Check:

- The nodeSelector;
- The nodeAffinity;
- The node labels.

If the issue is:

    insufficient cpu / memory

It means there is not a shortage of GPUs, but rather that the CPU or memory requests are not being met.

---

## Chapter 22: Troubleshooting GPU Issues When GPU Pods Are Running But Cannot Use the GPU

The Pod is running, but the GPU inside the container cannot be used.

### 22.1 Checking Whether the Pod Has Requested a GPU

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A10 resources

You should see:

    limits:
      nvidia.com/gpu: 1

### 22.2 Checking the GPU Inside the Container

Enter the container:

    kubectl exec -it <pod-name> -n <namespace> -- bash

Execute:

    nvidia-smi

If the image does not include bash, you can use sh.

### 22.3 Checking If the Image Contains nvidia-smi

Some business images do not contain nvidia-smi.

In this case, it is not possible to directly determine that the GPU is unavailable.

You can check:

    ls -l /dev/nvidia*
    echo $CUDA_VISIBLE_devices

Or use a CUDA test image with nvidia-smi to reproduce the issue.

### 22.4 Common Causes

If a Pod is running but the GPU cannot be used, possible reasons include:

- The Pod has not requested a GPU;
- Abnormal distribution of Device Plugins;
- The Runtime has not mounted the GPU device;
- Issues with the NVIDIA Container Toolkit;
- Abnormal containerd configuration;
- Incompatibility between the image's CUDA Runtime and drivers;
- The application is using a CPU-based framework;
- The CUDA_VISIBLE_devices setting has been overridden;
- The Pod is using an incorrect RuntimeClass;
- Abnormalities in the GPU node driver.

---

## Chapter 23: The Relationship Between GPU Scheduling and Container Runtime

The Scheduler is only responsible for scheduling Pods to nodes.

It is the local components on the node that actually start the container and mount the GPU device:

    kubelet
      ↓
    containerd / Docker / CRI-O
      ↓
    NVIDIA Container Toolkit / CDI
      ↓
    /dev/nvidia*
      ↓
    Container

### 23.1 Why Successful Scheduling by the Scheduler Does Not Mean the GPU Is Available

Just because a Pod is scheduled to a GPU node does not mean that the scheduler believes the resources are sufficient.

Whether the GPU inside the container is available also depends on:

- The node driver;
- The NVIDIA Container Toolkit;
- The configuration of the container runtime;
- The distribution of Device Plugins;
- The CUDA Runtime in the image;
- The application's own dependencies.

Therefore, when troubleshooting, it is important to distinguish between:

    Pending:
        Issues during the scheduling phase.

    Running but GPU Not Available:
        Problems with the node runtime or the application environment.

---

## Chapter 24: The Relationship Between GPU Scheduling and Images

GPU Pod images need to contain the correct runtime environment.

For example:

    nvidia/cuda:12.2.0-base-ubuntu22.04
    nvidia/cuda:12.2.0-runtime-ubuntu22.04
    nvidia/cuda:12.2.0-devel-ubuntu22.04
    pytorch/pytorch:xxx-cuda12.x-cudnnx-runtime

###- Business Performance Metrics.

### 27.3 Myth Three: If a GPU Pod is Pending, It Must Be Due to Insufficient GPUs

Wrong.

Pending could be caused by:

- Insufficient GPUs;
- Insufficient CPUs;
- Insufficient memory;
- Mismatch in taint labels;
- Mismatch in nodeSelector;
- Incompatibility with affinity settings;
- Exceeding Namespace quota limits;
- Nodes being in the NotReady state;
- Issues with image retrieval;
- Problems with PVCs;
- Issues with RuntimeClass configurations.

It is essential to check:

    `kubectl describe pod`

### 27.4 Myth Four: Applying for One GPU Will Limit the Amount of Video Memory Available

Wrong.

`nvidia.com/gpu: 1` indicates requesting one GPU device, but it does not mean that the video memory capacity will be restricted.

Video memory needs to be managed through application settings, MIG configurations, shared resource solutions, or platform-specific policies.

### 27.5 Myth Five: All GPU Nodes Can Be Used Indiscriminately

Wrong.

Different GPU models have significant differences:

- Different video memory capacities;
- Varying Tensor Core capabilities;
- Differing inference and training performance;
- Differences in support for MIG technology;
- Variations in NVLink availability;
- Different power consumption and cooling requirements.

In production environments, it is crucial to distinguish between different GPU models and use them according to their specific purposes.

---

## Chapter 28: Planning GPU Node Pools in Production Environments

### 28.1 Partitioning by Purpose

It is recommended to create separate pools for:

    `gpu-inference-pool`
    `gpu-training-pool`
    `gpu-dev-pool`
    `gpu-batch-pool`

Each pool can have different configurations, such as:

- Labels;
- Taint labels;
- ResourceQuota settings;
- PriorityClass values;
- Monitoring and alert settings;
- Automatic cleanup policies;
- Image access control rules.

### 28.2 Partitioning by GPU Model

Examples include:

    `gpu-l4-inference`
    `gpu-a10-inference`
    `gpu-a100-training`
    `gpu-h100-training`

### 28.3 Partitioning by Environment

Examples are:

    `dev-gpu`
    `test-gpu`
    `prod-gpu`

Production recommendations include:

- Avoid running development and testing tasks on the same GPU pool as production inference services.
- Prevent small-model inference tasks from consuming large, high-memory training GPUs for extended periods.
- Do not allocate GPUs without setting appropriate quotas to all namespaces.

---

## Chapter 29: Production YAML Templates for GPU Pods

### 29.1 Single-GPU Inference Pod Template

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: gpu-inference-demo
      namespace: ai-prod
      labels:
        app: gpu-inference-demo
        workload-type: inference
    spec:
      restartPolicy: Always
      nodeSelector:
        accelerator: nvidia
        gpu.workload: inference
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      containers:
        - name: inference
          image: nvidia/cuda:12.2.0-runtime-ubuntu22.04
          command: ["bash", "-lc", "nvidia-smi && sleep 3600"]
          resources:
            requests:
              cpu: "4"
              memory: "8Gi"
            limits:
              cpu: "8"
              memory: "16Gi"
              nvidia.com/gpu: 1
    ```
    
### 29.2 Multi-GPU Training Job Template

    ```yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: gpu-training-demo
      namespace: ai-training
    spec:
      backoffLimit: 1
      template:
        metadata:
          labels:
            app: gpu-training-demo
            workload-type: training
        spec:
          restartPolicy: Never
          nodeSelector:
            accelerator: nvidia
            gpu.workload: training
          tolerations:
            - key: "nvidia.com/gpu"
              operator: "Equal"
              value: "true"
              effect: "NoSchedule"
          containers:
            - name: trainer
              image: nvidia/cuda:12.2.0-devel-ubuntu22.04
              command: ["bash", "-lc", "nvidia-smi && sleep 3600"]
              resources:
                requests:
                  cpu: "8"
                  memory: "32Gi"
                limits:
                  cpu: "16"
                  memory: "64Gi"
                  nvidia.com- Labeling GPU nodes;
- Applying taints to GPU nodes;
- Organizing by model and purpose;
- Recording baseline information for GPU nodes, including model, memory, driver, CUDA, and topology;
- Integrating GPU nodes into monitoring systems;
- Implementing cordon/drain procedures before maintaining GPU nodes;
- Preventing ordinary Pods from being scheduled on GPU nodes.

### 32.2 Namespace Dimension

Recommendations:

- Organizing Namespaces by team;
- Setting ResourceQuotas;
- Establishing image access policies;
- Defining priorities;
- Distinguishing between dev/test/prod environments;
- Implementing audit trails for GPU usage.

### 32.3 Pod Dimension

Recommendations:

- Explicitly declare the need for a GPU;
- Also specify CPU and memory requirements;
- Use nodeSelector or affinity rules;
- Apply tolerations where necessary;
- Avoid using the latest images;
- Choose standard CUDA Runtime images;
- Use Jobs for training tasks and Deployments for inference services;
- Support checkpoints for long-running tasks;
- Ensure comprehensive logging and metrics collection.

### 32.4 Platform Dimension

Recommendations:

- Maintain a dashboard for GPU resources;
- Track allocation and utilization rates;
- Calculate the cost associated with Namespaces;
- Set up alerts for idle GPUs and pending GPU Pods;
- Establish health checks for GPU nodes and Device Plugins;
- Manage versions of drivers, CUDA, and frameworks;
- Develop processes for managing GPU changes and rollbacks.

---

## Summary

The core of Kubernetes' management of GPUs lies not in its direct recognition of GPUs but in the use of the Device Plugin mechanism to register GPUs as extended resources.

The key process is as follows:

    NVIDIA Driver functioning normally
      ↓
    Device Plugin detecting the GPU
      ↓
    Device Plugin registering the device with kubelet
      ↓
    kubelet updating the Node Status
      ↓
    The node displaying nvidia.com/gpu in its status
      ↓
    Pods declaring limits.nvidia.com/gpu in their configuration
      ↓
    The Scheduler scheduling Pods based on available resources and constraints
      ↓
    kubelet starting the scheduled Pods
      ↓
    The runtime environment mounting the GPU device
      ↓
    CUDA applications within containers utilizing the GPU

When troubleshooting, it is essential to analyze each layer separately:

- If nvidia-smi on the host machine is working properly, it indicates that the driver layer is functioning correctly.
- If the node shows nvidia.com/gpu in its status, it means Kubernetes has recognized the GPU resource.
- If Pods can be scheduled and run successfully, it confirms that scheduling has been successful.
- If nvidia-smi within containers is working fine, it indicates that both the runtime environment and device mounting are functioning correctly.
- Only when PyTorch/TensorFlow can recognize the GPU does it mean that the business framework is usable.

When scheduling production-grade GPUs, the following factors must be considered:

- Labeling;
- Tainting;
- ResourceQuotas;
- CPU/Memory configurations;
- GPU model and memory capacity;
- MIG support;
- Sharing strategies;
- Monitoring mechanisms;
- Utilization rates;
- Version management;
- Business priorities;
- Cost considerations.

Do not simplify GPU scheduling to merely adding a label like nvidia.com/gpu: 1. True Kubernetes capability in managing GPUs involves ensuring that hardware, drivers, CUDA, container runtime, Device Plugins, the Scheduler, Pods, monitoring systems, and business requirements are all seamlessly integrated into a cohesive system.

---

## References

- Kubernetes GPU Scheduling:
  https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/

- Kubernetes Extended Resources:
  https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources

- Kubernetes Resource Quotas:
  https://kubernetes.io/docs/concepts/policy/resource-quotas/

- Kubernetes Assigning Pods to Nodes:
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/

- Kubernetes Taints and Tolerations:
  https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

- NVIDIA Kubernetes Device Plugin:
  https://github.com/NVIDIA/k8s-device-plugin

- NVIDIA GPU Operator:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/

- NVIDIA Container Toolkit:
  https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

- NVIDIA DCGM Exporter:
  https://github.com/NVIDIA/dcgm-exporter

- NVIDIA GPU Sharing with GPU Operator:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html