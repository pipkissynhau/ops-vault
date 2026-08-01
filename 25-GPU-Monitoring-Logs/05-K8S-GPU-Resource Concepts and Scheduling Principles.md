# 05-K8S-GPU-Resource Concepts and Scheduling Principles

## Document Overview

This document is used to organize the basic concepts of GPU resources in Kubernetes, resource registration flow, scheduling principles, Pod request methods, node label and taint design, Namespace resource quotas, common scheduling failure reasons, and GPU scheduling planning methods in production environments.

This document focuses on answering the following questions:

- Why does Kubernetes default not support direct scheduling of NVIDIA GPUs?
- How does `nvidia.com/gpu` appear in Node resources?
- What role does the NVIDIA Device Plugin play in the scheduling flow?
- What is the relationship between kubelet, Device Plugin, Scheduler, and Pod?
- Why is GPU considered an Extended Resource?
- Why is `limits` typically used for GPU?
- Why is `500m` not used like CPU for GPU?
- How does the Scheduler select nodes after a Pod requests GPU?
- How to troubleshoot GPU Pod Pending status?
- Why do GPU nodes need Label and Taint?
- How does Namespace limit GPU usage?
- What are the boundaries of multi-GPU models, MIG, shared GPU, and time slicing?
- How to plan GPU node pools and scheduling strategies in production environments.

This document does not focus on NVIDIA drivers, CUDA, Device Plugin installation, and GPU Operator installation. Related content is placed in the following chapters:

- 03-NVIDIA-Drive Installation and Verification
- 04-CUDA-Installation and Testing
- 06-NVIDIA-Device-Plugin-and-Operator-Installation
- 07-GPU-Pod-Deployment and Scheduling Practice
- 08-GPU-Monitoring and Alert Integration

---

## Tags

#Kubernetes #GPU #NVIDIA #DevicePlugin #Scheduler #ExtendedResource #AiInfrastructure #Clouds. #SRE #TransportBarriers

---

## Recommended Path

Recommended path:

    06-GPU-and-AI-Infrastructure/02-Kubernetes-GPU-Scheduling/05-K8S-GPU-Resource-Concepts-and-Scheduling-Principles.md

---

## One: Why Kubernetes Needs Special Management for GPU

The most common native resources in Kubernetes are:

    CPU
    Memory
    Ephemeral Storage

For example, a regular Pod can request CPU and memory like this:

    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"

CPU and memory are built-in resources in Kubernetes.

kubelet can directly obtain node CPU and memory capacity through the operating system and cgroup, and report it to the apiserver.

However, GPU is different.

Kubernetes does not natively know how many NVIDIA GPUs are on a server, their health status, device paths, driver capabilities, or how containers mount GPUs.

Therefore, GPUs need to be integrated into Kubernetes through the device plugin mechanism.

In Kubernetes, NVIDIA GPUs typically appear as extended resources:

    nvidia.com/gpu

Pods request GPUs by declaring this resource.

Example:

    resources:
      limits:
        nvidia.com/gpu: 1

This indicates the Pod needs 1 GPU.

---

## Two: Complete Integration Flow of GPU in Kubernetes

For a GPU node to be schedulable by Kubernetes, it must go through the following flow at least:

    Physical GPU
      ↓
    BIOS / PCIe normal recognition
      ↓
    Linux lspci can see NVIDIA devices
      ↓
    NVIDIA Driver normal
      ↓
    nvidia-smi normal
      ↓
    NVIDIA Container Toolkit normal
      ↓
    kubelet normal operation
      ↓
    NVIDIA Device Plugin runs as DaemonSet
      ↓
    Device Plugin registers GPU resources with kubelet
      ↓
    kubelet updates Node Status
      ↓
    Node Capacity / Allocatable shows nvidia.com/gpu
      ↓
    Pod declares resources.limits.nvidia.com/gpu
      ↓
    Scheduler selects GPU node based on resources and constraints
      ↓
    kubelet starts Pod
      ↓
    Container Runtime mounts GPU device and driver library
      ↓
    Application in container uses CUDA to call GPU

If any layer in this flow fails, the GPU Pod may not run normally.

---

## Three: Differences in Resource Models Between GPU and CPU/Memory

### 3.1 CPU is a Compressible Resource

CPU can be over-allocated.

For example, a node with 8-core CPU can run multiple Pods, each requesting part of the CPU.

CPU supports millicore:

    cpu: "500m"

Means 0.5 core.

When CPU usage is high, containers can compete for CPU time slices.

### 3.2 Memory is an Incompressible Resource

Memory cannot be shared like CPU through time slicing.

If a Pod uses memory exceeding its limit, it may be OOMKilled.

### 3.3 GPU is an Extended Device Resource

GPU is closer to "exclusive devices".

By default, Kubernetes schedules NVIDIA GPUs as whole devices:

    nvidia.com/gpu: 1
    nvidia.com/gpu: 2

It cannot natively write:

    nvidia.com/gpu: 0.5

Nor can it be written like CPU:

    nvidia.com/gpu: 500m

Reasons:

- GPU is a device resource;
- The default scheduling unit is a complete device;
- Kubernetes native scheduling only knows resource quantity, not memory granularity usage;
- Memory limits are not part of the native GPU resource model in Kubernetes;
- GPU sharing requires MIG, time-slicing, MPS, or third-party vGPU solutions.

### 3.4 GPU Scheduling Only Solves "Allocation to Which Node"

Kubernetes default GPU scheduling mainly solves:

    Whether a Pod can be scheduled to a node with sufficient GPU count

It does not directly solve:

- GPU memory usage upper limit within a Pod;
- Memory contention among multiple processes;
- GPU utilization optimization;
- GPU memory fragmentation;
- Model inference concurrency;
- Multi-GPU communication efficiency;
- GPU and NIC topology affinity;
- NVLink topology awareness between GPUs;
- Whether the business truly utilizes the GPU.

Therefore, GPU scheduling is only part of the GPU platform, not equivalent to complete GPU resource governance.

---

## Four. What is Extended Resource

In Kubernetes, Extended Resource is used to describe non-built-in resources.

Common examples:

    nvidia.com/gpu
    amd.com/gpu
    example.com/fpga
    example.com/device

GPU is a typical Extended Resource.

Characteristics of Extended Resource:

- Registered by device plugins or external components;
- Usually integer resources;
- Cannot be compressed like CPU;
- Does not support default overselling;
- Scheduler schedules based on Node allocatable quantity;
- kubelet collaborates with Device Plugin to allocate devices locally.

For NVIDIA GPU, the most common extended resource name is:

    nvidia.com/gpu

---

## Five. Role of NVIDIA Device Plugin

NVIDIA Device Plugin is a core component in the Kubernetes GPU scheduling pipeline.

It typically runs as a DaemonSet on GPU nodes.

Main functions:

- Discover NVIDIA GPUs on the node;
- Check GPU health status;
- Register GPU devices with kubelet;
- Make `nvidia.com/gpu` appear in Node Status;
- Assist kubelet in allocating GPU devices when Pod starts;
- Control which GPUs containers can see;
- Support advanced capabilities like MIG, time-slicing, depending on configuration and version.

Device Plugin is not Scheduler.

It does not decide which node a Pod should be scheduled to.

It informs kubelet:

    Which GPUs are available for allocation on this node.

Scheduler makes scheduling decisions based on Node resource information reported to apiserver by kubelet.

---

## Six. GPU Resource Registration Process

### 6.1 Component Relationships

Simplified relationships:

    NVIDIA Driver
      ↓
    Device Plugin DaemonSet
      ↓
    kubelet Device Plugin Manager
      ↓
    kubelet updates Node Status
      ↓
    apiserver saves Node resources
      ↓
    scheduler uses Node resources for scheduling

### 6.2 Post-Resource Registration Behavior

After deploying Device Plugin, check the node:

    kubectl describe node <gpu-node-name>

Expected to see:

    Capacity:
      nvidia.com/gpu: 1

    Allocatable:
      nvidia.com/gpu: 1

Meaning:

    Capacity:
        Total GPU count on the node.

    Allocatable:
        Number of GPUs available for Pod scheduling.

If `Capacity` does not contain `nvidia.com/gpu`, it indicates GPU resources failed to register with Kubernetes.

Common causes:

- Node lacks NVIDIA GPU;
- NVIDIA Driver malfunction;
- nvidia-smi not functioning properly;
- Device Plugin not running;
- Device Plugin logs abnormal;
- kubelet plugin directory abnormal;
- Container runtime configuration abnormal;
- Node is not a Linux GPU node;
- GPU marked as unavailable by health check.

---

## Seven. How GPU Pods Request Resources

### 7.1 Minimal GPU Pod Example

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

Explanation:

    limits.nvidia.com/gpu: 1
        Indicates requesting 1 GPU.

Deployment:

    kubectl apply -f gpu-test.yaml

Check:

    kubectl get pod gpu-test -o wide
    kubectl logs gpu-test
    kubectl describe pod gpu-test

### 7.2 Why GPU is Usually Only Written with limits

GPU is an extended resource.

In Kubernetes, GPU resources can only be written with limits.

Example:

    resources:
      limits:
        nvidia.com/gpu: 1

Kubernetes treats this limit as the request.

If both requests and limits are written, they must be equal:

    resources:
      requests:
        nvidia.com/gpu: 1
      limits:
        nvidia.com/gpu: 1

Writing only GPU requests without limits is not allowed.

Error example:

    resources:
      requests:
        nvidia.com/gpu: 1

This format violates the rules for GPU extended resources.

### 7.3 Why Fractional GPU is Not Recommended

By default, Kubernetes schedules GPU resources as integer devices.

Correct:

    nvidia.com/gpu: 1

Incorrect:

    nvidia.com/gpu: 0.5

If shared GPU usage is indeed required, specialized solutions are needed, such as: /think

- NVIDIA MIG;
- NVIDIA Device Plugin time-slicing;
- NVIDIA MPS;
- Third-party vGPU scheduling solution;
- Self-developed GPU resource scheduling platform.

These are not something that ordinary `nvidia.com/gpu: 0.5` can resolve.

---

## VIII. How Scheduler Schedules GPU Pods

When Scheduler schedules Pods, it comprehensively considers multiple conditions.

For GPU Pods, at least includes:

- Whether the node is Ready;
- Whether the node has sufficient CPU request;
- Whether the node has sufficient Memory request;
- Whether the node has sufficient `nvidia.com/gpu`;
- Whether the node meets nodeSelector;
- Whether the node meets nodeAffinity;
- Whether the node's taint is tolerated by Pod toleration;
- Whether Namespace ResourceQuota allows further GPU requests;
- Pod Priority and preemption strategy;
- Whether the node meets other resource and constraint requirements.

### 8.1 Simplified Scheduling Judgment Process

Simplified process:

    Pod creation
      ↓
    Scheduler reads Pod resource requests
      ↓
    Detects Pod needs nvidia.com/gpu: 1
      ↓
    Traverses available nodes
      ↓
    Filters nodes without GPU
      ↓
    Filters nodes with insufficient GPU
      ↓
    Filters nodes with mismatched taint
      ↓
    Filters nodes with mismatched label/affinity
      ↓
    Filters nodes with insufficient CPU/Memory
      ↓
    Scores remaining nodes
      ↓
    Selects the most suitable node
      ↓
    Pod binds to Node
      ↓
    kubelet starts container on the node
      ↓
    Device Plugin/Runtime allocates GPU devices

### 8.2 Scheduler Does Not Directly Call nvidia-smi

Scheduler will not execute:

    nvidia-smi

on the node.

Scheduler relies on the Node resource status stored in apiserver.

If Device Plugin fails to register GPU to Node Status, even if the host `nvidia-smi` is normal, Scheduler cannot correctly schedule GPU Pod to this node.

Therefore, when troubleshooting GPU Pod Pending, must check:

    kubectl describe node <gpu-node-name>

instead of only checking the host:

    nvidia-smi

---

## IX. GPU Node Label Design

In production environments, GPU nodes generally need to be labeled.

Reasons:

- Distinguish CPU nodes from GPU nodes;
- Distinguish different GPU models;
- Distinguish inference nodes from training nodes;
- Distinguish different business pools;
- Facilitate scheduling constraints;
- Facilitate monitoring, statistics, and capacity planning.

### 9.1 Basic Labels

Example:

    kubectl label node <gpu-node-name> node-role.kubernetes.io/gpu=true
    kubectl label node <gpu-node-name> accelerator=nvidia
    kubectl label node <gpu-node-name> gpu.vendor=nvidia

### 9.2 Labels by GPU Model

Example:

    kubectl label node <gpu-node-name> gpu.model=a100
    kubectl label node <gpu-node-name> gpu.memory=80gb

Or:

    kubectl label node <gpu-node-name> gpu.model=l4
    kubectl label node <gpu-node-name> gpu.memory=24gb

### 9.3 Labels by Business Purpose

Inference node:

    kubectl label node <gpu-node-name> gpu.workload=inference

Training node:

    kubectl label node <gpu-node-name> gpu.workload=training

Experiment node:

    kubectl label node <gpu-node-name> gpu.workload=dev

### 9.4 Pod Uses nodeSelector

Example:

    nodeSelector:
      accelerator: nvidia
      gpu.workload: inference

Complete example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: gpu-inference-test
    spec:
      restartPolicy: Never
      nodeSelector:
        accelerator: nvidia
        gpu.workload: inference
      containers:
        - name: cuda
          image: nvidia/cuda:12.2.0-base-ubuntu22.04
          command: ["nvidia-smi"]
          resources:
            limits:
              nvidia.com/gpu: 1

---

## X. GPU Node Taint and Toleration

### 10.1 Why GPU Nodes Should Be Tainted

GPU nodes are costly and should not allow ordinary Pods to schedule arbitrarily.

Without restrictions, ordinary Pods may occupy GPU nodes' resources:

- CPU;
- Memory;
- Disk;
- Network;
- Image cache;
- I/O;
- Pod count.

Although ordinary Pods don't request GPU, they may affect GPU tasks' data loading, inference latency, and node stability.

Therefore, taints are typically applied to GPU nodes in production environments.

### 10.2 Adding GPU Taint

Example:

    kubectl taint node <gpu-node-name> nvidia.com/gpu=true:NoSchedule

Meaning:

    Pods without corresponding toleration cannot be scheduled to this node.

### 10.3 GPU Pod Adds Toleration

Example:

tolerations:
  - key: "nvidia.com/gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"

Complete Example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: gpu-test
    spec:
      restartPolicy: Never
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      containers:
        - name: cuda
          image: nvidia/cuda:12.2.0-base-ubuntu22.04
          command: ["nvidia-smi"]
          resources:
            limits:
              nvidia.com/gpu: 1

### 10.4 Difference Between Label and Taint

Label / nodeSelector solves:

    I want to schedule to certain types of nodes

Taint / toleration solves:

    Which Pods are allowed to be scheduled on this node

Production environments typically use both together:

    GPU Node:
      Label: accelerator=nvidia
      Taint: nvidia.com/gpu=true:NoSchedule

    GPU Pod:
      nodeSelector: accelerator=nvidia
      tolerations: nvidia.com/gpu=true:NoSchedule

This is more secure.

---

## 11. Node Affinity in GPU Scheduling

nodeSelector is simple matching.

nodeAffinity is more flexible.

### 11.1 requiredDuringSchedulingIgnoredDuringExecution

This is a hard constraint.

Example:

    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: gpu.model
                  operator: In
                  values:
                    - a100
                    - h100

Meaning:

    The Pod must be scheduled on a node with gpu.model as a100 or h100.

### 11.2 preferredDuringSchedulingIgnoredDuringExecution

This is a soft constraint.

Example:

    affinity:
      nodeAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
                - key: gpu.workload
                  operator: In
                  values:
                    - inference

Meaning:

    Prefer scheduling to inference nodes, but it's not mandatory.

### 11.3 Production Recommendations

Training tasks:

    Better suited for required hard constraints to avoid scheduling to incorrect GPU models.

Inference tasks:

    Can combine with preferred for prioritized scheduling while retaining some flexibility.

Multi-GPU cluster:

    Must plan gpu.model, gpu.memory, gpu.workload labels properly.

---

## 12. GPU and ResourceQuota

Production environments typically need to limit the number of GPUs used by different Namespaces.

For example, a team can only use up to 4 GPUs.

### 12.1 GPU ResourceQuota Example

    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: gpu-quota
      namespace: ai-team-a
    spec:
      hard:
        requests.nvidia.com/gpu: "4"

Note:

    Quotas for Extended Resources typically use requests.<resource-name>.

### 12.2 Creating Namespace

    kubectl create namespace ai-team-a

Apply quota:

    kubectl apply -f gpu-quota.yaml

Check:

    kubectl describe resourcequota gpu-quota -n ai-team-a

### 12.3 Behavior When Quota is Reached

If a Namespace has already used 4 GPUs, creating new GPU Pods may fail or be rejected.

Check:

    kubectl describe pod <pod-name> -n ai-team-a
    kubectl get events -n ai-team-a --sort-by=.lastTimestamp

### 12.4 Production Recommendations

By team planning:

    ai-team-a: 4 GPUs
    ai-team-b: 8 GPUs
    ai-platform: 2 GPUs

By environment planning:

    dev: 1 GPU
    test: 2 GPUs
    prod: 8 GPUs

By task type planning:

    inference: Independent Namespace
    training: Independent Namespace
    experiment: Independent Namespace

GPU is a high-cost resource, and quotas must be managed in production environments.

---

## 13. GPU and LimitRange Boundary

LimitRange is commonly used to set default request/limit for CPU and memory.

However, it is not recommended to rely solely on LimitRange for GPU auto-injection.

Reasons:

- GPU is an expensive device;
- GPU should be explicitly requested by business;
- Implicit default GPU may lead to resource waste;
- GPU resource usage should be controlled through review or quotas;
- Different businesses require different GPU models and quantities.

Recommendations:

    GPU must be explicitly declared.
    It is not recommended to automatically add nvidia.com/gpu to Namespace by default.
    GPU usage should be standardized through templates, platforms, CI/CD, or admission control.

---

## FourteenI don't know.GPU and PriorityClass / Preemption

GPU resources are expensive, and in production environments, scenarios where high-priority tasks need to preempt low-priority tasks may occur.

Examples:

- Online inference services have high priority;
- Offline training tasks have low priority;
- Experimental tasks have the lowest priority;
- Emergency production tasks require priority scheduling.

### 14.1 PriorityClass Example

    apiVersion: scheduling.k8s.io/v1
    kind: PriorityClass
    metadata:
      name: gpu-prod-high
    value: 100000
    globalDefault: false
    description: "High priority for production GPU workloads"

Pod Usage:

    priorityClassName: gpu-prod-high

### 14.2 Usage Notes

Preemption is not a universal solution.

Notes to be aware of:

- The preempted Pod will be terminated;
- Training tasks may lose progress;
- Must be combined with checkpointing;
- Inference services should have multiple replicas;
- Preemption strategies must be confirmed by business;
- It is not recommended to enable high priority arbitrarily in ungoverned clusters.

---

## FifteenI don't know.GPU and Pod QoS

Kubernetes Pod QoS includes:

- Guaranteed;
- Burstable;
- BestEffort.

GPU resources themselves do not directly determine QoS.

QoS is mainly determined by CPU and Memory requests/limits.

### 15.1 Recommended Resource Writing for GPU Pods

Production GPU Pods should not only specify GPU.

CPU and memory requests/limits should be specified simultaneously.

Example:

    resources:
      requests:
        cpu: "4"
        memory: "16Gi"
      limits:
        cpu: "8"
        memory: "32Gi"
        nvidia.com/gpu: 1

A stricter Guaranteed example:

    resources:
      requests:
        cpu: "8"
        memory: "32Gi"
        nvidia.com/gpu: 1
      limits:
        cpu: "8"
        memory: "32Gi"
        nvidia.com/gpu: 1

### 15.2 Why GPU Pods Should Also Specify CPU/Memory

GPU tasks do not only consume GPU.

They also require:

- CPU for data preprocessing;
- Memory to cache models and data;
- Network to pull data;
- Disk to read models;
- Process management;
- Log output;
- Sidecar or monitoring processes.

If only GPU is requested without sufficient CPU/memory, the following may occur:

- Low GPU utilization;
- Slow data loading;
- Pod being OOMKilled;
- Low training throughput;
- Inference latency fluctuations;
- Node resource overcompetition.

---

## SixteenI don't know.GPU Scheduling and VRAM

Kubernetes defaults to scheduling `nvidia.com/gpu` by only knowing GPU count, not VRAM details.

Example:

    nvidia.com/gpu: 1

Only indicates requesting 1 GPU.

It does not indicate:

    Requesting 10Gi VRAM
    Limiting maximum usage to 20Gi VRAM
    Allocating half a GPU
    Automatic VRAM isolation

### 16.1 Common Signs of VRAM Insufficiency

Application logs:

    CUDA out of memory

PyTorch:

    RuntimeError: CUDA out of memory

nvidia-smi:

    Memory-Usage approaching full capacity

### 16.2 Why Kubernetes Cannot Directly Limit GPU VRAM

Default Device Plugin allocates devices, not VRAM cgroups.

GPU VRAM is not a native memory cgroup control item in Kubernetes.

To govern at the VRAM level, you need:

- MIG;
- Time-slicing;
- MPS;
- Third-party vGPU;
- AI platform layer control;
- Application-side batch size and model control;
- Inference service concurrency limits.

### 16.3 Production Recommendations

Do not mistakenly assume that:

    nvidia.com/gpu: 1
    memory: 8Gi

Here, `memory: 8Gi` refers to container memory, not GPU VRAM.

GPU VRAM needs to be governed separately through GPU metrics and application constraints.

---

## SeventeenI don't know.GPU Scheduling and MIG

MIG, full name Multi-Instance GPU.

It allows certain NVIDIA data center GPUs to split a single physical GPU into multiple hardware-isolated GPU instances.

Common in A100, H100, and other MIG-supported GPUs.

### 17.1 What MIG Solves

MIG can solve:

- Multiple small tasks sharing a large GPU;
- Improving GPU utilization;
- Providing hardware-level isolation;
- Partitioning instances by different VRAM and compute specifications;
- Multi-tenant isolation for inference services.

### 17.2 Differences Between MIG and Regular GPU Scheduling

Regular full GPU:

    nvidia.com/gpu: 1

In MIG scenarios, resource names may become similar to:

    nvidia.com/mig-1g.10gb
    nvidia.com/mig-2g.20gb
    nvidia.com/mig-3g.40gb

The exact names depend on Device Plugin/GPU Operator configuration and GPU model.

### 17.3 Notes for MIG Usage

MIG is not supported on all GPUs.

Need to pay attention to: /think

- Does the GPU model support it;
- Does the driver version support it;
- Is the Device Plugin configured with MIG policy;
- Does the GPU Operator manage MIG;
- Is the MIG partitioning strategy consistent;
- How does a Pod request MIG resources;
- Does monitoring recognize MIG instances;
- Can fault diagnosis locate MIG instances.

### 17.4 Production Recommendations

MIG is more suitable for:

- Multi-tenant inference;
- Medium-sized model services;
- GPU utilization improvement;
- Scenarios with high resource isolation requirements.

MIG may not be suitable for:

- Training requiring full-card large memory;
- Large model tasks requiring complete GPU computing power;
- Scenarios requiring frequent switching of partitioning specifications.

---

## EighteenI don't know.GPU Sharing: Time-Slicing and MPS

By default, Kubernetes GPU scheduling allocates a single GPU to one or more containers, but resource models typically request whole cards as integers.

To improve utilization, shared solutions can be used.

### 18.1 Time-Slicing

Time-Slicing is time-based sharing.

Multiple workloads can be scheduled on the same GPU, taking turns using GPU time.

Suitable for:

- Lightweight inference;
- Development and testing;
- Low utilization tasks;
- Non-strong isolation scenarios.

Notes:

- Not hardware isolation;
- Memory may still compete;
- An abnormal task may affect other tasks;
- Not suitable for strong SLA scenarios;
- Requires additional configuration from Device Plugin / GPU Operator.

### 18.2 MPS

MPS, full name Multi-Process Service.

It allows multiple CUDA processes to share GPU more efficiently.

Suitable for some high-performance computing and multi-process sharing scenarios.

Notes:

- Requires additional configuration;
- Not all businesses are suitable;
- Caution is needed for isolation and stability;
- Needs to be paired with Kubernetes platform governance.

### 18.3 Comparison of MIG, Time-Slicing, and MPS

    MIG:
        Hardware-level partitioning, stronger isolation, suitable for multi-tenant inference.

    Time-Slicing:
        Time-based sharing, improves utilization, but weaker isolation.

    MPS:
        CUDA multi-process sharing optimization, suitable for specific computing scenarios.

Production Recommendations:

    Prioritize MIG for strong isolation in production.
    Consider Time-Slicing for development testing or low-priority tasks.
    MPS requires evaluation based on business and CUDA program characteristics.
    Do not mix production inference and training tasks sharing the same GPU without understanding boundaries.

---

## NineteenI don't know.Multi-GPU Cluster Scheduling

Production clusters may have:

- T4;
- L4;
- A10;
- A30;
- A100;
- H100;
- H20.

The capabilities of different GPUs vary greatly.

If only `nvidia.com/gpu` is used, the Scheduler only knows the number of GPUs on a node, but not what model the business actually needs.

### 19.1 Problem Example

A large model training task requires A100 80GB.

But without additional constraints, it may be scheduled to an L4 node.

Result:

- Insufficient memory;
- Performance not meeting standards;
- Program startup failure;
- CUDA OOM;
- Business misjudges as code issues.

### 19.2 Solution

Label nodes:

    kubectl label node gpu-node-a100-01 gpu.model=a100
    kubectl label node gpu-node-a100-01 gpu.memory=80gb
    kubectl label node gpu-node-l4-01 gpu.model=l4
    kubectl label node gpu-node-l4-01 gpu.memory=24gb

Add constraints to Pod:

    nodeSelector:
      gpu.model: a100
      gpu.memory: 80gb

Or use nodeAffinity.

### 19.3 Production Recommendations

Different GPU models should be managed in separate pools:

    gpu-pool-l4-inference
    gpu-pool-a10-inference
    gpu-pool-a100-training
    gpu-pool-h100-training

Do not mix all GPU nodes into a single unlabeled pool.

---

## TwentyI don't know.GPU Scheduling and Topology Awareness

Kubernetes has limited awareness of GPU topology when scheduling GPUs by default.

For ordinary single-card inference tasks, the impact may not be significant.

For multi-card training, high-performance computing, and large model training, topology is crucial.

### 20.1 Topology to Pay Attention To

Includes:

- Whether GPUs are connected via NVLink;
- Whether GPUs span NUMA;
- Whether GPUs and network cards are on the same NUMA;
- Distance from GPU to RDMA network card;
- PCIe Switch topology;
- CPU Socket distribution;
- Whether multi-card tasks are assigned to appropriate GPUs.

### 20.2 Viewing Topology

Run on the node:

    nvidia-smi topo -m

View PCIe tree:

    lspci -tv

View NUMA:

    lscpu | grep -i numa

View device NUMA:

    cat /sys/bus/pci/devices/0000:<PCI_ID>/numa_node

### 20.3 Production Recommendations

Ordinary inference:

    Prioritize GPU model, memory, and utilization.

Multi-card training:

    Must pay attention to GPU-GPU topology, NVLink, and NUMA.

Multi-node training:

    Must pay attention to GPU-NIC topology, RDMA, network bandwidth, and NCCL.

---

## Twenty-oneI don't know.GPU Pod Pending Troubleshooting Process

The most common issue with GPU Pods is Pending.

### 21.1 Check Pod Status

    kubectl get pod <pod-name> -n <namespace> -o wide

### 21.2 Check Detailed Events

    kubectl describe pod <pod-name> -n <namespace>

Focus on Events.

Common events:

    0/3 nodes are available: insufficient nvidia.com/gpu
    node(s) had untolerated taint
    node(s) didn't match Pod's node affinity/selector
    insufficient cpu
    insufficient memory
    exceeded quota

### 21.3 Check Node GPU Resources

    kubectl describe node <gpu-node-name>

Focus on:

    Capacity:
      nvidia.com/gpu

Allocatable:
  nvidia.com/gpu

Allocated resources:
  nvidia.com/gpu

### 21.4 View All Node GPU Resources

You can use:

    kubectl get nodes

Then describe each node individually.

If plugins or custom scripts are installed, you can also aggregate the view.

Simple method:

    kubectl describe nodes | grep -A5 -B5 "nvidia.com/gpu"

### 21.5 View Device Plugin

    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia

Check logs:

    kubectl logs <nvidia-device-plugin-pod> -n <namespace>

The specific namespace depends on the installation method.

It could be:

    kube-system
    gpu-operator
    gpu-operator-resources
    nvidia-device-plugin

### 21.6 View Node Labels and Taints

Labels:

    kubectl get node <gpu-node-name> --show-labels

Taints:

    kubectl describe node <gpu-node-name> | grep -i taints -A5

### 21.7 View Namespace Quotas

    kubectl describe resourcequota -n <namespace>

### 21.8 Troubleshooting Judgment

If the event is:

    insufficient nvidia.com/gpu

Prioritize checking:

- Does the node have GPU?
- Is the Device Plugin registered?
- Is the GPU already occupied by another Pod?
- Is the Namespace quota full?
- Does the Pod request too much GPU?

If the event is:

    untolerated taint

Check:

- GPU node taint
- Pod tolerations

If the event is:

    didn't match node selector

Check:

- nodeSelector
- nodeAffinity
- Node label

If the event is:

    insufficient cpu / memory

This indicates it's not GPU shortage, but CPU or memory request not met.

---

## Twenty-two, Troubleshooting GPU Not Available After Pod Runs

The Pod is already Running, but the container cannot use GPU.

### 22.1 Check if Pod Requests GPU

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A10 resources

You should see:

    limits:
      nvidia.com/gpu: 1

### 22.2 Check GPU in Container

Enter the container:

    kubectl exec -it <pod-name> -n <namespace> -- bash

Execute:

    nvidia-smi

If the image doesn't have bash, use sh.

### 22.3 Does the Image Include nvidia-smi

Some business images may not include nvidia-smi.

In this case, you cannot directly determine GPU unavailability.

You can check:

    ls -l /dev/nvidia*
    echo $CUDA_VISIBLE_DEVICES

Or use a CUDA test image with nvidia-smi to reproduce.

### 22.4 Common Causes

Pod Running but GPU unavailable, possible causes:

- Pod did not request GPU
- Device Plugin allocation anomaly
- Runtime did not mount GPU device
- NVIDIA Container Toolkit anomaly
- containerd configuration anomaly
- Image CUDA Runtime incompatible with driver
- Application uses CPU version framework
- CUDA_VISIBLE_DEVICES is overridden
- Pod uses incorrect RuntimeClass
- GPU node driver anomaly

---

## Twenty-three, Relationship Between GPU Scheduling and Container Runtime

Scheduler only responsible for scheduling Pod to nodes.

The actual container startup and GPU device mounting is handled by node-local components:

    kubelet
      ↓
    containerd / Docker / CRI-O
      ↓
    NVIDIA Container Toolkit / CDI
      ↓
    /dev/nvidia*
      ↓
    Container

### 23.1 Why Scheduler Success Doesn't Guarantee GPU Availability

Pod scheduled to GPU node only indicates Scheduler thinks resources are sufficient.

But GPU availability inside container depends on:

- Node driver
- NVIDIA Container Toolkit
- Container runtime configuration
- Device Plugin allocation
- Image CUDA Runtime
- Application dependencies

Therefore, troubleshooting should distinguish:

    Pending:
        Scheduling phase issue.

    Running but GPU unavailable:
        Node runtime or application environment issue.

---

## Twenty-four, Relationship Between GPU Scheduling and Image

GPU Pod image needs to include correct runtime environment.

Examples:

    nvidia/cuda:12.2.0-base-ubuntu22.04
    nvidia/cuda:12.2.0-runtime-ubuntu22.04
    nvidia/cuda:12.2.0-devel-ubuntu22.04
    pytorch/pytorch:xxx-cuda12.x-cudnnx-runtime

### 24.1 Differences Between base / runtime / devel

    base:
        Basic CUDA environment, suitable for simple testing.

    runtime:
        Includes runtime libraries for CUDA applications, suitable for production services.

    devel:
        Includes compilation tools, such as nvcc, suitable for building and development.

### 24.2 Production Recommendations

Run services:

    Use runtime image.

Build image:

    Use devel image.

Test GPU:

    Use official nvidia/cuda base image to execute nvidia-smi.

Not recommended: /think

Production runtime images should directly use latest.
Production services should long-term use devel images.
Different businesses arbitrarily use chaotic CUDA versions.

---

## 25. GPU Scheduling and Monitoring Metrics

GPU scheduling needs to be combined with monitoring, otherwise you can only know "whether a Pod can be scheduled", but not "how the GPU is being used".

### 25.1 Common GPU Metrics

Through DCGM Exporter, the following can be collected:

- GPU utilization;
- Memory usage;
- GPU temperature;
- GPU power consumption;
- ECC errors;
- XID errors;
- GPU health status;
- MIG instance metrics.

### 25.2 Scheduling-Related Metrics

Also pay attention to:

- GPU node Ready;
- Device Plugin Pod status;
- GPU Pod Pending count;
- GPU Pod Restart;
- Namespace GPU usage;
- GPU allocation rate;
- GPU utilization;
- GPU memory utilization.

### 25.3 Production Recommendations

GPU platforms should not only look at:

    nvidia.com/gpu allocated amount

But also check:

    Whether GPU-Util is actually high
    Whether memory is long-term occupied
    Whether there are Pods occupying GPUs without usage
    Whether GPU Pods are Pending
    Whether GPU nodes are overheating
    Whether there are XID errors
    Whether GPU usage is balanced across teams

---

## 26. GPU Resource Utilization and Scheduling Efficiency

GPU is an expensive resource.

Common issues in production environments:

- GPU is requested but utilization remains 0 long-term;
- Memory is occupied but no actual computation;
- Experimental Pods occupy entire cards without releasing;
- Too many inference service replicas leading to resource waste;
- Training tasks without checkpoints suffer severe losses when preempted;
- GPU nodes have insufficient CPU/memory causing GPU idling;
- Slow data loading leading to low GPU utilization;
- Lack of quotas in multi-tenancy causing resources to be monopolized by a few teams.

### 26.1 Identification of Idle GPU Usage

Can combine:

    nvidia-smi
    DCGM_FI_DEV_GPU_UTIL
    DCGM_FI_DEV_FB_USED
    Pod runtime
    Namespace
    Owner
    Job status

To judge:

    High memory + low GPU utilization + long-running Pod
        Could be model-resident inference or idle GPU usage.

    Low memory + low GPU utilization + long-running Pod
        Could be resource waste.

    High GPU utilization + normal business throughput
        Could be normal training or inference.

Do not draw conclusions based on a single metric.

### 26.2 Production Governance Methods

Can use:

- ResourceQuota;
- PriorityClass;
- TTLAfterFinished;
- Job completion auto-cleanup;
- Platform approval;
- Low-priority experiment pools;
- GPU utilization reports;
- Idle GPU alerts;
- Namespace cost statistics;
- Tenant quotas;
- Reservation and queuing mechanisms;

To improve GPU utilization.

---

## 27. Common GPU Scheduling Misconceptions

### 27.1 Misconception 1: If nvidia-smi is normal, Kubernetes can definitely schedule GPU

Incorrect.

nvidia-smi being normal only indicates the host driver layer is normal.

Whether Kubernetes can schedule GPU also requires:

- Device Plugin being normal;
- kubelet registering resources;
- Node Status having `nvidia.com/gpu`;
- Scheduler seeing the resource;
- Pod correctly requesting GPU.

### 27.2 Misconception 2: Pod Running means GPU is already used by the business

Incorrect.

Pod Running only means the container has started.

Whether the business is actually using GPU also needs to check:

- nvidia-smi inside the container;
- Application logs;
- PyTorch / TensorFlow detection;
- GPU utilization;
- Memory usage;
- CUDA_VISIBLE_DEVICES;
- Business performance metrics.

### 27.3 Misconception 3: GPU Pod Pending is definitely due to insufficient GPU

Incorrect.

Pending could be:

- Insufficient GPU;
- Insufficient CPU;
- Insufficient memory;
- Taint mismatch;
- nodeSelector mismatch;
- affinity not met;
- Namespace quota exceeded;
- Node NotReady;
- Image pull issues;
- PVC issues;
- RuntimeClass issues.

Must check:

    kubectl describe pod

### 27.4 Misconception 4: Requesting 1 GPU card can limit memory

Incorrect.

    nvidia.com/gpu: 1

Indicates requesting one GPU device, not limiting memory size.

Memory needs to be governed through application, MIG, sharing solutions, or platform policies.

### 27.5 Misconception 5: All GPU nodes can be mixed used

Incorrect.

Different GPU models have huge differences.

For example:

- Different memory capacities;
- Different Tensor Core capabilities;
- Different inference performance;
- Different training capabilities;
- Different support for MIG;
- Different support for NVLink;
- Different power consumption and cooling.

Production must distinguish GPU models and purposes.

---

## 28. GPU Node Pool Planning in Production Environments

### 28.1 Split by Purpose

Recommended splits:

    gpu-inference-pool
    gpu-training-pool
    gpu-dev-pool
    gpu-batch-pool

Different node pools have different settings:

- Label;
- Taint;
- ResourceQuota;
- PriorityClass;
- Monitoring alerts;
- Auto-cleanup policies;
- Image admission policies.

### 28.2 Split by GPU Model

Examples:

    gpu-l4-inference
    gpu-a10-inference
    gpu-a100-training
    gpu-h100-training

### 28.3 Split by Environment

Examples:

    dev-gpu
    test-gpu
    prod-gpu

Production recommendations:

    Not recommended to mix development/test tasks and production inference services in the same GPU node pool.
    Not recommended to have small model inference long-term occupy large memory training cards.
    Not recommended to open GPU access to all Namespaces without quotas.

---

## 29. GPU Pod YAML Production Template

### 29.1 Single-GPU Inference Pod Template /think

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

### 29.2 Multi-GPU Training Job Template

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
                  nvidia.com/gpu: 2

---

## 30. GPU Scheduling Troubleshooting Command Summary

### 30.1 View Pod

    kubectl get pod <pod-name> -n <namespace> -o wide
    kubectl describe pod <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace>

### 30.2 View Node

    kubectl get nodes -o wide
    kubectl describe node <gpu-node-name>
    kubectl get node <gpu-node-name> --show-labels

### 30.3 View GPU Resources

    kubectl describe node <gpu-node-name> | grep -A10 -B5 "nvidia.com/gpu"

### 30.4 View Device Plugin

    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia
    kubectl logs <device-plugin-pod> -n <namespace>

### 30.5 View Events

    kubectl get events -A --sort-by=.lastTimestamp

Specify Namespace:

    kubectl get events -n <namespace> --sort-by=.lastTimestamp

### 30.6 View Quotas

    kubectl describe resourcequota -n <namespace>

### 30.7 View Local GPU on Node

Enter the node:

    nvidia-smi
    nvidia-smi -L
    nvidia-smi topo -m
    lsmod | grep nvidia
    ls -l /dev/nvidia*

---

## 31. GPU Scheduling Fault Layer Table

| Phenomenon | Priority Troubleshooting Layer | Common Causes |
|---|---|---|
| `lspci` GPU Not Visible | Hardware / BIOS | GPU Not Properly Inserted, Power Supply, PCIe, Above 4G Decoding |
| `nvidia-smi` Failure | Driver Layer | Driver, nouveau, Secure Boot, DKMS |
| Node Without `nvidia.com/gpu` | Device Plugin / kubelet | Plugin Not Running, Driver Abnormality, Registration Failure |
| GPU Pod Pending | Scheduler | GPU Insufficient, Taint, Label, Quota, CPU/Memory |
| Pod Running But No GPU | Runtime / Container | Container Toolkit, Runtime, Image, Device Mounting |
| CUDA OOM | Application / Memory | Batch Size, Model, Memory Usage, Shared Conflict |
| Low GPU Utilization | Application / Data Pipeline | Slow Data Loading, CPU Bottleneck, Business Idle |
| Slow Multi-GPU Training | Topology / Network | NVLink, NUMA, RDMA, NCCL, PCIe |

---

## 32. Production Environment GPU Scheduling Design Recommendations

### 32.1 Node Dimension

Recommendations:

- Pool GPU Nodes Separately;
- Label GPU Nodes;
- Add Taint to GPU Nodes;
- Split by Model and Purpose;
- Record GPU Model, Memory, Driver, CUDA, Topology as Baseline;
- Monitor GPU Nodes;
- Cordon / Drain GPU Nodes Before Maintenance;
- Prohibit Ordinary Pods from Arbitrarily Scheduling on GPU Nodes.

### 32.2 Namespace Dimension

Recommendations:

- Split Namespaces by Team;
- Set ResourceQuota;
- Set Image Admission Policies;
- Set Priority;
- Distinguish dev/test/prod;
- Establish GPU Usage Audit.

### 32.3 Pod Dimension

Recommendations:

- Explicitly Declare GPU;
- Simultaneously Declare CPU / Memory;
- Use nodeSelector or Affinity;
- Use Toleration;
- Avoid Latest Image;
- Use Standard CUDA Runtime Image;
- Use Job for Training Tasks;
- Use Deployment for Inference Services;
- Support Checkpoint for Long Tasks;
- Fully Integrate Logs and Metrics.

### 32.4 Platform Dimension

Recommendations:

- Maintain GPU Resource Dashboard;
- Statistic Allocation Rate and Utilization;
- Statistic Namespace Costs;
- Establish Idle GPU Alert;
- Establish GPU Pod Pending Alert;
- Establish GPU Node Health Alert;
- Establish Device Plugin Status Alert;
- Manage Driver / CUDA / Framework Version Matrix;
- Establish GPU Change and Rollback Process.

---

## 33. Summary

The core of Kubernetes managing GPUs is not directly identifying GPUs itself, but registering GPUs as extended resources through the Device Plugin mechanism.

Core Chain:

    NVIDIA Driver Normal
      ↓
    Device Plugin Detects GPU
      ↓
    Device Plugin Registers Device to kubelet
      ↓
    kubelet Updates Node Status
      ↓
    Node Appears nvidia.com/gpu
      ↓
    Pod Declares limits.nvidia.com/gpu
      ↓
    Scheduler Schedules Based on Resources and Constraints
      ↓
    kubelet Launches Pod
      ↓
    Runtime Mounts GPU Device
      ↓
    CUDA Application in Container Uses GPU

Troubleshooting Must Be Layered:

    nvidia-smi on Host Node Normal:
        Only Indicates Driver Layer is Normal.

    Node Has nvidia.com/gpu:
        Indicates Kubernetes Has Recognized GPU Resources.

    Pod Can Schedule and Run:
        Indicates Scheduling Success.

    nvidia-smi in Container Normal:
        Indicates Runtime and Device Mounting are Basically Normal.

    PyTorch / TensorFlow Can Recognize GPU:
        Only Then Indicates Business Framework Layer is Basically Available.

Production GPU Scheduling Must Focus On:

- Label;
- Taint;
- ResourceQuota;
- CPU / Memory Complement;
- GPU Model;
- Memory;
- MIG;
- Sharing Policy;
- Monitoring;
- Utilization;
- Version Matrix;
- Business Priority;
- Cost Governance.

Do Not Simply Understand GPU Scheduling as:

    Write a nvidia.com/gpu: 1

True Kubernetes GPU Operations Capability is the Ability to Connect Hardware, Driver, CUDA, Container Runtime, Device Plugin, Scheduler, Pod, Monitoring, and Business Usage into a Complete Chain.

---

## Reference Documents

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