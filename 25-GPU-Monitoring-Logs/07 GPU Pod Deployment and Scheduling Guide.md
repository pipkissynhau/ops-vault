# 07-GPU Pod Deployment and Scheduling Practical Guide

## Document Overview

This document is used to organize the deployment, scheduling, verification, troubleshooting, and production usage methods of GPU Pods in Kubernetes clusters.

This document focuses on answering the following questions:

- What prerequisites should be checked before deploying GPU Pods?
- How to confirm that Kubernetes nodes have recognized `nvidia.com/gpu`?
- How to write the minimal GPU Pod configuration?
- How to verify if the container can access GPUs?
- How to use `nodeSelector`, `nodeAffinity`, and `taint/toleration` to control GPU Pod scheduling?
- How to deploy GPU inference services using Deployment?
- How to deploy GPU training tasks using Job?
- How to troubleshoot GPU Pod Pending status?
- How to troubleshoot Pod Running but no GPU access inside the container?
- How to verify if CUDA, PyTorch, and TensorFlow can use GPUs?
- How to understand the relationship between GPU Pod CPU, memory, and GPU memory?
- How to write GPU Pod YAML in production environments?
- How to clean up, reclaim, and govern GPU Pod resources?

This document is suitable for study after completing the following chapters:

- 03-NVIDIA Driver Installation and Verification
- 04-CUDA Installation and Testing
- 05-K8S GPU Resource Concepts and Scheduling Principles
- 06-NVIDIA Device Plugin and Operator Installation

This document does not expand on Device Plugin / GPU Operator installation details, focusing instead on practical deployment and scheduling verification of GPU Pods.

---

## Tags

#Kubernetes #GPU #NVIDIA #CUDA #DevicePlugin #GPUOperator #PodMovement #AiInfrastructure #SRE #TransportBarriers

---

## Recommended Path

Recommended path:

    06-GPU and AI Infrastructure/02-Kubernetes GPU Scheduling/07-GPU Pod Deployment and Scheduling Practical Guide.md

---

## One: Core Goals of GPU Pod Deployment Practical Guide

GPU Pod deployment is not simply writing a YAML snippet:

    resources:
      limits:
        nvidia.com/gpu: 1

Real GPU Pod practical deployment requires verifying an end-to-end chain:

    GPU hardware is normal
      ↓
    NVIDIA Driver is normal
      ↓
    nvidia-smi is normal
      ↓
    NVIDIA Container Toolkit is normal
      ↓
    Device Plugin / GPU Operator is normal
      ↓
    Node has nvidia.com/gpu
      ↓
    Pod correctly requests GPU
      ↓
    Scheduler schedules Pod to GPU node
      ↓
    kubelet starts container
      ↓
    containerd mounts GPU device
      ↓
    nvidia-smi can run inside container
      ↓
    CUDA / PyTorch / TensorFlow can recognize GPU
      ↓
    Business tasks actually use GPU

Therefore, GPU Pod deployment practical guide cannot only check:

    kubectl get pod

But also check:

    kubectl describe pod
    kubectl describe node
    kubectl logs
    nvidia-smi
    /dev/nvidia*
    CUDA_VISIBLE_DEVICES
    PyTorch cuda.is_available
    Prometheus GPU metrics

---

## Two: Pre-deployment Checks for GPU Pod

Before deploying GPU Pod, do not directly write YAML.

First confirm cluster and node status.

### 2.1 Check Kubernetes Nodes

    kubectl get nodes -o wide

Expected:

    GPU nodes are in Ready state.

If GPU nodes are NotReady, troubleshoot kubelet, containerd, CNI, and node resource pressure issues first.

### 2.2 Check GPU Node Labels

    kubectl get node <gpu-node-name> --show-labels

Recommended labels include at least:

    accelerator=nvidia
    node-role.kubernetes.io/gpu=true
    gpu.vendor=nvidia
    gpu.model=<model>
    gpu.workload=<training|inference|dev>

If missing, these can be added later.

### 2.3 Check Node Taints

    kubectl describe node <gpu-node-name> | grep -i taints -A5

Common GPU node taints:

    nvidia.com/gpu=true:NoSchedule

If the node has this taint, GPU Pod must configure toleration.

### 2.4 Check if Node Recognizes GPU Resources

    kubectl describe node <gpu-node-name>

Focus on:

    Capacity:
      nvidia.com/gpu: 1

    Allocatable:
      nvidia.com/gpu: 1

If `nvidia.com/gpu` is missing, Kubernetes cannot schedule GPU.

Prioritize troubleshooting:

- NVIDIA Driver
- NVIDIA Container Toolkit
- NVIDIA Device Plugin
- GPU Operator
- kubelet
- Node GPU health status

### 2.5 Check NVIDIA Components

If using Device Plugin:

    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia

If using GPU Operator:

    kubectl get pods -n gpu-operator -o wide
    kubectl get ds -n gpu-operator
    kubectl get clusterpolicy

### 2.6 Check Device Plugin Logs

First find the Pod:

    kubectl get pods -A | grep -i device-plugin

View logs: /think

kubectl logs <device-plugin-pod> -n <namespace>

If the Device Plugin is not functioning properly, GPU Pods are likely to fail scheduling.

### 2.7 Node Local Checks

Execute the following commands on the GPU node:

    lspci | grep -i nvidia
    nvidia-smi
    nvidia-smi -L
    nvidia-smi topo -m
    lsmod | grep nvidia
    ls -l /dev/nvidia*
    dmesg | grep -i xid
    nvidia-container-cli info
    containerd config dump | grep -i nvidia -A30 -B10

These commands are used to confirm:

- Hardware visibility;
- Driver functionality;
- Correct GPU count;
- GPU device files exist;
- containerd supports NVIDIA runtime;
- No obvious XID errors.

---

## Three, GPU Pod Minimum Example

### 3.1 Minimum YAML

Create a file:

    cat > gpu-test.yaml <<'EOF'
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
    EOF

Deploy:

    kubectl apply -f gpu-test.yaml

Check the Pod:

    kubectl get pod gpu-test -o wide

Check logs:

    kubectl logs gpu-test

Expected logs should include:

    NVIDIA-SMI
    Driver Version
    CUDA Version
    GPU Name
    Memory-Usage
    GPU-Util

### 3.2 Why This Pod Validates GPU

This Pod validates:

    Whether the Pod can schedule to a GPU node
    Whether the container can see the GPU
    Whether the NVIDIA runtime correctly injects devices and driver libraries
    Whether nvidia-smi can access the GPU inside the container

It does NOT validate:

- CUDA compilation capability;
- Whether nvcc exists;
- Whether PyTorch can use GPU;
- Whether TensorFlow can use GPU;
- Whether business models can load;
- Whether inference services are normal;
- Whether multi-GPU communication is normal.

Because `nvidia/cuda:12.2.0-base-ubuntu22.04` is a base image, it may not include complete development tools.

---

## Four, GPU Pod Resource Declaration Rules

### 4.1 GPU Usage with limits

GPU Pod requests GPU:

    resources:
      limits:
        nvidia.com/gpu: 1

Kubernetes has special rules for extended resources like GPU:

    GPU can only write limits.
    Kubernetes treats limit as request.
    If both requests and limits are written, they must be equal.
    Cannot only write requests without writing limits.

### 4.2 Correct Way

Method one:

    resources:
      limits:
        nvidia.com/gpu: 1

Method two:

    resources:
      requests:
        nvidia.com/gpu: 1
      limits:
        nvidia.com/gpu: 1

### 4.3 Not Recommended Way

Incorrect example:

    resources:
      requests:
        nvidia.com/gpu: 1

Problem:

    Only writing GPU requests without writing GPU limits violates extended resource usage rules.

### 4.4 GPU Does Not Support CPU-style Decimal Notation

Incorrect example:

    resources:
      limits:
        nvidia.com/gpu: 0.5

By default, Kubernetes allocates GPU resources as integer devices.

If sharing GPU is needed, use:

- MIG;
- Time-slicing;
- MPS;
- Third-party vGPU / GPU Share schemes;
- AI platform layer resource governance.

Cannot simply write `0.5`.

---

## Five, GPU Pod and CPU / Memory Resources

GPU Pod should not only declare GPU.

Production environments must consider CPU and memory simultaneously.

### 5.1 Why GPU Pod Needs CPU

GPU tasks typically require CPU for:

- Data preprocessing;
- Image decoding;
- Text tokenization;
- Network request handling;
- Model pre/post-processing;
- Log output;
- Process scheduling;
- Data transfer from disk/network to GPU.

If CPU is insufficient, may appear:

- Low GPU utilization;
- Slow training;
- High inference latency;
- Data loading blocking;
- GPU idle.

### 5.2 Why GPU Pod Needs Memory

Memory is used for:

- Loading model files;
- Caching input data;
- Python process runtime;
- Framework execution;
- Data preprocessing;
- Logs and intermediate objects.

If memory is insufficient, Pod may be:

    OOMKilled

Check:

    kubectl describe pod <pod-name> -n <namespace>
    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -i oom -A5 -B5

### 5.3 Production GPU Pod Resource Example

    resources:
      requests:
        cpu: "4"
        memory: "16Gi"
      limits:
        cpu: "8"
        memory: "32Gi"
        nvidia.com/gpu: 1

A more strict Guaranteed example:

resources:
  requests:
    cpu: "8"
    memory: "32Gi"
    nvidia.com/gpu: 1
  limits:
    cpu: "8"
    memory: "32Gi"
    nvidia.com/gpu: 1

Note:

    memory: 32Gi is container memory, not GPU memory.
    GPU memory is not directly restricted by Kubernetes memory limit.

---

## Six. Using nodeSelector to Control GPU Pod Scheduling

### 6.1 Label GPU Nodes

    kubectl label node <gpu-node-name> accelerator=nvidia
    kubectl label node <gpu-node-name> gpu.workload=inference
    kubectl label node <gpu-node-name> gpu.model=a100

Check:

    kubectl get node <gpu-node-name> --show-labels

### 6.2 Pod Using nodeSelector

Example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: gpu-test-nodeselector
      namespace: default
    spec:
      restartPolicy: Never
      nodeSelector:
        accelerator: nvidia
      containers:
        - name: cuda
          image: nvidia/cuda:12.2.0-base-ubuntu22.04
          command: ["nvidia-smi"]
          resources:
            limits:
              nvidia.com/gpu: 1

Deploy:

    kubectl apply -f gpu-test-nodeselector.yaml

Check:

    kubectl get pod gpu-test-nodeselector -o wide

### 6.3 Applicable Scenarios for nodeSelector

Suitable:

- Simple GPU node specification;
- Distinguish CPU nodes and GPU nodes;
- Experimental environments;
- Small clusters;
- Simple model isolation.

Not suitable:

- Complex multi-condition scheduling;
- Multiple GPU models;
- Training/inference complex hybrid;
- Soft constraint priority scheduling.

Complex scenarios are better suited for nodeAffinity.

---

## Seven. Using nodeAffinity to Control GPU Pod Scheduling

### 7.1 Hard Constraint nodeAffinity

Example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: gpu-test-affinity
      namespace: default
    spec:
      restartPolicy: Never
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
      containers:
        - name: cuda
          image: nvidia/cuda:12.2.0-base-ubuntu22.04
          command: ["nvidia-smi"]
          resources:
            limits:
              nvidia.com/gpu: 1

Meaning:

    The Pod must be scheduled to a node with gpu.model as a100 or h100.

### 7.2 Soft Constraint nodeAffinity

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

    Prefer scheduling to nodes with gpu.workload=inference.
    If no matching nodes are available, the Scheduler can choose other suitable nodes.

### 7.3 Production Recommendations

Training tasks:

    Better suited for hard constraints to avoid scheduling to incorrect GPU models.

Inference tasks:

    Can choose between hard or soft constraints based on SLA.

Multi-model clusters:

    Recommend using gpu.model, gpu.memory, gpu.workload, etc. labels with affinity.

---

## Eight. Using Taint / Toleration to Protect GPU Nodes

### 8.1 Add Taint to GPU Nodes

    kubectl taint node <gpu-node-name> nvidia.com/gpu=true:NoSchedule

Meaning:

    Pods without corresponding toleration cannot be scheduled to this node.

### 8.2 GPU Pod Adds Toleration

Example: /think

### 8.3 Using nodeSelector and toleration together

Production recommendation:

    apiVersion: v1
    kind: Pod
    metadata:
      name: gpu-test-node-taint
      namespace: default
    spec:
      restartPolicy: Never
      nodeSelector:
        accelerator: nvidia
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

Meaning:

    nodeSelector:
        Specifies to schedule to GPU nodes.

    toleration:
        Allows the Pod to be scheduled to nodes with GPU taints.

The two solve different problems and are recommended to be used together.

---

## NineI don't know.Deployment for GPU Inference Services

GPU inference services are typically managed by Deployment.

Suitable scenarios:

- Model inference API;
- Image recognition service;
- Large model inference service;
- Embedding service;
- OCR inference;
- Audio/video inference.

### 9.1 GPU Deployment Example

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: gpu-inference-demo
      namespace: ai-prod
      labels:
        app: gpu-inference-demo
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gpu-inference-demo
      template:
        metadata:
          labels:
            app: gpu-inference-demo
            workload-type: inference
        spec:
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

Deployment:

    kubectl apply -f gpu-inference-deployment.yaml

Check:

    kubectl get deploy -n ai-prod
    kubectl get pod -n ai-prod -o wide
    kubectl logs <pod-name> -n ai-prod

### 9.2 Relationship between replicas and GPU Count

If Deployment sets:

    replicas: 3

And each Pod requests:

    nvidia.com/gpu: 1

Then a total of 3 available GPUs are needed.

If the cluster only has 2 idle GPUs, the third Pod will be Pending.

Check:

    kubectl describe pod <pending-pod> -n ai-prod

May see:

    insufficient nvidia.com/gpu

### 9.3 Production Considerations for GPU Inference Services

Production inference services also need to consider:

- readinessProbe;
- livenessProbe;
- startupProbe;
- model loading time;
- preheating;
- graceful shutdown;
- rolling update;
- maximum concurrency;
- VRAM usage;
- P95 / P99 latency;
- HPA / KEDA;
- GPU utilization;
- canary release;
- fixed image version;
- model version management.

---

## 10. Job Deployment for GPU Training Tasks

GPU training tasks are typically managed using Jobs or more advanced training frameworks.

Suitable scenarios:

- offline training;
- model fine-tuning;
- batch processing;
- one-time GPU tasks;
- experimental tasks.

### 10.1 GPU Job Example

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
                  nvidia.com/gpu: 1

Deployment:

    kubectl apply -f gpu-training-job.yaml

Check:

    kubectl get job -n ai-training
    kubectl get pod -n ai-training -o wide
    kubectl logs <pod-name> -n ai-training

### 10.2 Multi-GPU Job

If a training task requires 2 GPUs:

    resources:
      limits:
        nvidia.com/gpu: 2

Note:

    This indicates that the same Pod needs to be scheduled on a node with 2 GPUs available.
    If no single node has 2 GPUs available, the Pod will remain in Pending state.
    Kubernetes by default will not distribute 2 GPUs of a Pod across two nodes.

### 10.3 Training Task Production Notes

Training tasks should focus on:

- checkpoint;
- failure retry;
- data mounting;
- log retention;
- model output path;
- training duration;
- GPU utilization;
- VRAM usage;
- multi-GPU communication;
- distributed training;
- automatic cleanup after task completion;
- low-priority experimental tasks avoiding production GPU usage.

---

## 11. GPU Usage in Multi-Container Pods

### 11.1 Sidecar Should Not Arbitrarily Request GPU

A Pod can have multiple containers.

If only the main business container needs GPU, do not let sidecar containers also request GPU.

Example:

    containers:
      - name: app
        image: nvidia/cuda:12.2.0-runtime-ubuntu22.04
        resources:
          limits:
            nvidia.com/gpu: 1

      - name: sidecar
        image: busybox
        command: ["sh", "-c", "sleep 3600"]

### 11.2 Be Aware of Resource Ownership

GPU resources are declared at the container level.

Do not assume that Pod-level resources are automatically allocated to all containers.

In production, clearly define:

- which container needs GPU;
- which container is only for logging, proxy, monitoring;
- whether sidecar affects CPU/memory;
- whether sidecar needs access to model files;
- whether sidecar increases node resource pressure.

---

## 12. Verify GPU Visibility Inside Container

### 12.1 Enter Pod

    kubectl exec -it <pod-name> -n <namespace> -- bash

If no bash is available:

    kubectl exec -it <pod-name> -n <namespace> -- sh

### 12.2 Check GPUs

    nvidia-smi

### 12.3 Check Device Files

    ls -l /dev/nvidia*

Expected output may include:

    /dev/nvidia0
    /dev/nvidiactl
    /dev/nvidia-uvm

### 12.4 Check Environment Variables

    echo $CUDA_VISIBLE_DEVICES
    echo $NVIDIA_VISIBLE_DEVICES
    echo $NVIDIA_DRIVER_CAPABILITIES

Notes:

    CUDA_VISIBLE_DEVICES:
        Controls which GPU devices are visible to CUDA applications.

    NVIDIA_VISIBLE_DEVICES:
        Device visibility variable used by NVIDIA Container Runtime.

    NVIDIA_DRIVER_CAPABILITIES:
        Controls which driver capabilities are exposed, such as compute, utility, etc.

### 12.5 Checking Processes

Inside the container:

    nvidia-smi

On the host machine:

    nvidia-smi

You can see which process is using the GPU.

If it's a Kubernetes container process, you can reverse-check using containerd / crictl.

---

## Thirteen, Verifying CUDA Capabilities

### 13.1 Using base image to verify nvidia-smi

    image: nvidia/cuda:12.2.0-base-ubuntu22.04
    command: ["nvidia-smi"]

Verification:

- Whether GPU devices are visible;
- Whether the driver library is injected;
- Whether nvidia-smi is available.

### 13.2 Using devel image to verify nvcc

    image: nvidia/cuda:12.2.0-devel-ubuntu22.04
    command: ["bash", "-lc", "nvidia-smi && nvcc -V"]

Verification:

- Whether the CUDA Toolkit compiler exists;
- Whether nvcc is available.

Note:

    Base / runtime images may not include nvcc.
    Devel images typically include nvcc.
    It's not recommended to use devel images for production services.

### 13.3 Using runtime image to verify runtime environment

    image: nvidia/cuda:12.2.0-runtime-ubuntu22.04

Suitable for running services.

If your business program only runs CUDA applications without compilation, the runtime image is more suitable.

---

## Fourteen, Verifying PyTorch Uses GPU

### 14.1 PyTorch Test Command

Execute inside the container:

    python3 -c "import torch; print('torch:', torch.__version__); print('cuda version:', torch.version.cuda); print('cuda available:', torch.cuda.is_available()); print('device count:', torch.cuda.device_count())"

Expected output:

    cuda available: True
    device count: 1

### 14.2 PyTorch Test Pod Example

The actual image version should be selected according to business needs.

Example structure:

    apiVersion: v1
    kind: Pod
    metadata:
      name: pytorch-gpu-test
      namespace: default
    spec:
      restartPolicy: Never
      nodeSelector:
        accelerator: nvidia
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      containers:
        - name: pytorch
          image: pytorch/pytorch:<explicit version>
          command:
            - bash
            - -lc
            - |
              python3 -c "import torch; print('torch:', torch.__version__); print('cuda version:', torch.version.cuda); print('cuda available:', torch.cuda.is_available()); print('device count:', torch.cuda.device_count())"
          resources:
            limits:
              nvidia.com/gpu: 1

Note:

    Do not use latest in production documentation.
    PyTorch image versions must be compatible with CUDA Runtime and host driver.

### 14.3 Common Reasons PyTorch Cannot Detect GPU

Possible reasons:

- Installed CPU version of PyTorch;
- PyTorch CUDA version mismatch with the image;
- Host driver is too old;
- Container has no GPU mounted;
- Pod has not requested `nvidia.com/gpu`;
- Device Plugin anomaly;
- `CUDA_VISIBLE_DEVICES` is overridden;
- Image entry script modifies environment variables.

---

## Fifteen, Verifying TensorFlow Uses GPU

### 15.1 TensorFlow Test Command

    python3 -c "import tensorflow as tf; print(tf.__version__); print(tf.config.list_physical_devices('GPU'))"

If you can see GPU devices, it means TensorFlow can recognize the GPU.

### 15.2 Common TensorFlow Issues

Possible reasons:

- TensorFlow version mismatch with CUDA/cuDNN;
- Used CPU version;
- Host driver incompatibility;
- Image missing necessary runtime libraries;
- Container has no GPU device;
- Pod has not requested GPU;
- Runtime configuration anomaly.

Production recommendations:

    TensorFlow / PyTorch images should be included in the platform's base image version matrix.
    Do not allow each business team to arbitrarily select incompatible CUDA images.

---

## Sixteen, Troubleshooting GPU Pod Pending

GPU Pod Pending is the most common issue.

### 16.1 Check Pod Status

    kubectl get pod <pod-name> -n <namespace> -o wide

If the Node column is empty, it means the scheduling hasn't succeeded yet.

### 16.2 Check Pod Events

    kubectl describe pod <pod-name> -n <namespace>

Focus on the Events section.

Common information: /think

0/3 nodes are available: insufficient nvidia.com/gpu
node(s) had untolerated taint
node(s) didn't match Pod's node affinity/selector
insufficient cpu
insufficient memory
exceeded quota

### 16.3 insufficient nvidia.com/gpu

Meaning:

    The cluster currently has insufficient allocatable GPUs.

Troubleshooting:

    kubectl describe node <gpu-node-name>
    kubectl get pods -A -o wide | grep <gpu-node-name>
    kubectl describe nodes | grep -A10 -B5 "nvidia.com/gpu"

Check:

- Does the Node have `nvidia.com/gpu`;
- Has the GPU been occupied by other Pods;
- Did the Pod request excessive GPUs;
- Does the single node meet multi-card requests;
- Is the Device Plugin functioning normally;
- Is the GPU marked as unhealthy.

### 16.4 untolerated taint

Meaning:

    The node has taints, but the Pod lacks corresponding toleration.

View node taints:

    kubectl describe node <gpu-node-name> | grep -i taints -A5

Pod addition:

    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"

### 16.5 nodeSelector / affinity mismatch

Check Pod:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A20 nodeSelector
    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A40 affinity

Check node labels:

    kubectl get node <gpu-node-name> --show-labels

Resolution:

- Modify the Pod's nodeSelector / affinity;
- Or add correct labels to the node;
- Or confirm whether the business should be scheduled to this GPU model.

### 16.6 CPU / Memory Insufficient

If the event is:

    insufficient cpu
    insufficient memory

This indicates the issue is not with the GPU, but with CPU or memory request.

Resolution:

- Reduce requests;
- Increase node resources;
- Clean up node load;
- Switch to larger GPU node specifications;
- Optimize resource allocation.

### 16.7 exceeded quota

Check quota:

    kubectl describe resourcequota -n <namespace>

If the GPU quota is full, you need:

- Remove unused GPU Pods;
- Adjust ResourceQuota;
- Switch Namespace;
- Follow the resource application process.

---

## SeventeenI don't know.Pod Running but No GPU Visible in Container Troubleshooting

### 17.1 Confirm if the Pod has requested GPU

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A15 resources

Should see:

    limits:
      nvidia.com/gpu: 1

If not, it indicates the Pod is a regular Pod.

### 17.2 Enter the container for inspection

    kubectl exec -it <pod-name> -n <namespace> -- sh

Check:

    nvidia-smi
    ls -l /dev/nvidia*
    echo $CUDA_VISIBLE_DEVICES
    echo $NVIDIA_VISIBLE_DEVICES

### 17.3 Image lacks nvidia-smi

Some business images do not include `nvidia-smi`.

This does not mean the GPU is unavailable.

You can check:

    ls -l /dev/nvidia*
    python3 -c "import torch; print(torch.cuda.is_available())"

You can also switch to the official CUDA test image to verify node connectivity.

### 17.4 Node runtime troubleshooting

On the GPU node, execute:

    nvidia-container-cli info
    containerd config dump | grep -i nvidia -A30 -B10
    systemctl status containerd
    systemctl status kubelet

Restart if necessary:

    systemctl restart containerd
    systemctl restart kubelet

After restart, you need to recreate the Pod.

### 17.5 Device Plugin Troubleshooting

    kubectl get pods -A | grep -i nvidia
    kubectl logs <device-plugin-pod> -n <namespace>

If using GPU Operator:

    kubectl get pods -n gpu-operator -o wide
    kubectl logs <nvidia-device-plugin-pod> -n gpu-operator

---

## EighteenI don't know.GPU Pod Running After CUDA OOM Troubleshooting

### 18.1 Phenomenon

Application logs:

    CUDA out of memory

Or PyTorch error:

    RuntimeError: CUDA out of memory

### 18.2 Check GPU memory

Execute in the host or container:

    nvidia-smi

Focus on:

    Memory-Usage

### 18.3 Common Causes

- Batch size too large;
- Model too large;
- Multiple processes using the same GPU;
- Inference service loads multiple models;
- GPU memory not released;
- Framework caches GPU memory;
- MIG instance GPU memory too small;
- Shared GPU tasks competing for GPU memory;
- Pod requests 1 GPU, but internal business starts multiple workers.

### 18.4 Resolution Methods

- Reduce batch size;
- Use a smaller model;
- Use FP16 / BF16;
- Use quantization;
- Limit concurrency;
- Reduce the number of workers;
- Clean up abnormal processes;
- Use a GPU with larger memory;
- Use MIG for reasonable partitioning;
- Use a model service framework to control memory;
- Add checkpoints to training tasks to avoid significant loss from failures.

Note:

    Kubernetes memory limit restricts container memory, not GPU memory.
    nvidia.com/gpu: 1 indicates requesting one GPU, not limiting memory size.

---

## NineteenI don't know.GPU Pod and CUDA_VISIBLE_DEVICES

### 19.1 Purpose

`CUDA_VISIBLE_DEVICES` Controls which GPUs are visible to CUDA applications.

In Kubernetes + Device Plugin scenarios, Device Plugin and runtime typically control which GPUs are visible to containers.

### 19.2 Inspection

Inside the container:

    echo $CUDA_VISIBLE_DEVICES
    nvidia-smi

### 19.3 Notes

Do not hardcode:

    CUDA_VISIBLE_DEVICES=0

in business images, because in Kubernetes, the GPU numbering seen by containers may have been remapped by the runtime.

If hardcoded incorrectly, it may lead to:

- Application not seeing GPUs;
- Application using the wrong GPU;
- Multi-GPU training anomalies;
- Debug results inconsistent with the host.

Production recommendations:

    Let the platform and Device Plugin manage GPU visibility.
    Do not arbitrarily override CUDA_VISIBLE_DEVICES on the business side.
    Explicitly manage it only when doing multi-GPU training according to framework requirements.

---

## TwentyI don't know.Multi-GPU Pod Practical Guide

### 20.1 Request 2 GPUs

    apiVersion: v1
    kind: Pod
    metadata:
      name: multi-gpu-test
      namespace: default
    spec:
      restartPolicy: Never
      nodeSelector:
        accelerator: nvidia
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
              nvidia.com/gpu: 2

Meaning:

    This Pod requires a node with 2 available GPUs.

### 20.2 Common Pending Reasons for Multi-GPU Pods

- Cluster total GPUs are sufficient, but single-node GPUs are not;
- Single-node has 2 GPUs, but one is occupied;
- Insufficient available nodes after nodeSelector restrictions;
- Taint/toleration mismatch;
- CPU / Memory insufficiency;
- Namespace GPU quota insufficiency.

### 20.3 Multi-GPU Training Notes

Multi-GPU training also needs to pay attention to:

- NCCL;
- GPU topology;
- NVLink;
- NUMA;
- Distance between GPU and network card;
- RDMA;
- Distributed training framework;
- Container shared memory;
- Data loading performance.

Check topology:

    nvidia-smi topo -m

---

## Twenty-OneI don't know.MIG Pod Practical Guide

### 21.1 Check MIG Resources

If MIG is enabled, first check which resources the node exposes:

    kubectl describe node <gpu-node-name>

You might see:

    nvidia.com/mig-1g.10gb
    nvidia.com/mig-2g.20gb
    nvidia.com/mig-3g.40gb

The actual resource name is based on the node output.

### 21.2 MIG Pod Example

Example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: mig-test
      namespace: default
    spec:
      restartPolicy: Never
      containers:
        - name: cuda
          image: nvidia/cuda:12.2.0-base-ubuntu22.04
          command: ["nvidia-smi"]
          resources:
            limits:
              nvidia.com/mig-1g.10gb: 1

Note:

    Do not copy MIG resource names directly.
    Must use the actual names shown by kubectl describe node.

### 21.3 MIG Suitable Scenarios

Suitable for:

- Multi-tenant inference;
- Small model inference;
- Fixed specification GPU instances;
- Improve large GPU utilization;
- Shared scenarios requiring strong isolation.

Not suitable for:

- Training requiring full large memory;
- Tasks requiring full card performance;
- Chaotic environments with frequent specification changes.

---

## Twenty-TwoI don't know.GPU Time-Slicing Pod Guide

### 22.1 Significance of Time-Slicing

Time-Slicing allows multiple workloads to share a single GPU.

Suitable for:

- Development and testing;
- Low utilization tasks;
- Light inference;
- Experimental environments;
- Non-strong isolation scenarios.

NVIDIA GPU Operator supports GPU time-slicing through extended Device Plugin configuration, enabling multiple workloads to alternate running on oversubscribed GPUs.

### 22.2 Usage Notes

Time-Slicing is not equal to MIG.

It is not hardware isolation.

Risks:

- Memory may compete with each other;
- An abnormal task may affect other tasks;
- Performance cannot be fully guaranteed;
- Not suitable for strong SLA business;
- Not suitable for core production training tasks.

### 22.3 Production Recommendations

Production strong isolation:

    Prioritize MIG.

Development and testing:

    Can evaluate Time-Slicing.

Core inference:

    Use cautiously, requiring pressure testing and isolation strategies.

---

## Twenty-ThreeI don't know.GPU Pod and Image Version

### 23.1 Do Not Use latest

Not recommended: /think

image: nvidia/cuda:latest

Reasons:

- Version is uncontrollable;
- Fault reproduction is difficult;
- Driver compatibility is uncontrollable;
- Production rollback is difficult;
- Security scanning is unstable.

Recommendations:

    image: nvidia/cuda:12.2.0-runtime-ubuntu22.04

Or enterprise internal image:

    image: registry.example.com/ai/cuda:12.2.0-runtime-ubuntu22.04

### 23.2 base / runtime / devel Selection

    base:
        For basic testing, such as nvidia-smi.

    runtime:
        For running CUDA applications, suitable for production services.

    devel:
        Includes development tools like nvcc, suitable for building and debugging.

Production Recommendations:

    Use runtime for inference services.
    Use devel for building images.
    Use base for testing pipelines.
    Do not use devel for long-term production services.

### 23.3 Private Image Registry

Production environments are recommended to synchronize images to internal repositories:

    Harbor
    Nexus
    Alibaba Cloud Image Registry
    Private Registry

Reasons:

- Stable domestic network;
- Controllable versions;
- Security scanning;
- Reliable rollback;
- Avoid expansion failure due to public network unavailability.

---

## Twenty-Four, GPU Pod and Service Exposure

GPU inference services typically need to be exposed through a Service.

Example:

    apiVersion: v1
    kind: Service
    metadata:
      name: gpu-inference-demo
      namespace: ai-prod
    spec:
      type: ClusterIP
      selector:
        app: gpu-inference-demo
      ports:
        - name: http
          port: 8080
          targetPort: 8080

Notes:

    GPU is just a computing resource.
    Service exposure method is the same as for regular Pods.
    Service does not care whether the Pod uses GPU.
    Whether the business is normal depends on container ports, probes, logs, and model loading status.

If public access is required, it is typically combined with:

- Ingress;
- Gateway API;
- LoadBalancer;
- NodePort;
- Reverse proxy;
- API gateway.

---

## Twenty-Five, GPU Inference Service Probe Design

GPU inference service startup is usually slow because it needs:

- Downloading models;
- Loading models;
- Initializing CUDA;
- Initializing TensorRT;
- Preheating;
- Memory allocation.

Therefore, probe design should be cautious.

### 25.1 startupProbe

Suitable for handling slow startups.

Example:

    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 60
      periodSeconds: 10

Meaning:

    Wait up to 600 seconds for startup.

### 25.2 readinessProbe

Indicates whether the service can accept traffic.

Example:

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10

### 25.3 livenessProbe

Indicates whether the process is healthy.

Example:

    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 60
      periodSeconds: 20

### 25.4 Production Recommendations

GPU service probes should not be overly aggressive.

Otherwise, it may cause:

- Killing the service before model loading completes;
- Frequent memory allocation and release;
- Pod restart storms;
- Abnormal GPU node load;
- Inference service failure to start stably.

---

## Twenty-Six, GPU Pod Log and Event Troubleshooting

### 26.1 View Pod Logs

    kubectl logs <pod-name> -n <namespace>

If multiple containers:

    kubectl logs <pod-name> -n <namespace> -c <container-name>

View logs from the previous crash:

    kubectl logs <pod-name> -n <namespace> --previous

### 26.2 View Pod Events

    kubectl describe pod <pod-name> -n <namespace>

Focus on:

    Events

### 26.3 View Namespace Events

    kubectl get events -n <namespace> --sort-by=.lastTimestamp

### 26.4 View Node Events

    kubectl describe node <gpu-node-name>

Focus on:

- Node Conditions;
- Allocated resources;
- Events;
- Taints;
- Capacity;
- Allocatable.

### 26.5 View Device Plugin Logs

    kubectl get pods -A | grep -i nvidia
    kubectl logs <device-plugin-pod> -n <namespace>

---

## Twenty-Seven, GPU Pod Resource Cleanup

### 27.1 Delete Test Pod

    kubectl delete pod gpu-test

### 27.2 Delete Job

    kubectl delete job gpu-training-demo -n ai-training

Note:

    Deleting a Job will default delete the Pods it created.
    If you want to retain logs, collect them first or configure a logging system.

### 27.3 Delete Deployment

    kubectl delete deployment gpu-inference-demo -n ai-prod

### 27.4 Check if GPU is Released

View node resources:

    kubectl describe node <gpu-node-name>

View allocated resources:

Allocated resources:

Check the host node:

    nvidia-smi

If GPU memory is still occupied after a Pod is deleted, possible causes may include:

- Residual processes on the host node;
- Containers not fully exiting;
- Runtime anomalies;
- Driver status anomalies;
- Delayed display from nvidia-smi;
- Business processes detaching from container management.

Continue troubleshooting:

    crictl ps -a
    ps -ef | grep <PID>
    nvidia-smi

---

## Twenty-EightI don't know.GPU Pod and ResourceQuota

### 28.1 Creating GPU Namespace

    kubectl create namespace ai-team-a

### 28.2 Creating GPU Quota

    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: gpu-quota
      namespace: ai-team-a
    spec:
      hard:
        requests.nvidia.com/gpu: "2"
        requests.cpu: "16"
        requests.memory: "64Gi"

Apply:

    kubectl apply -f gpu-quota.yaml

Check:

    kubectl describe resourcequota gpu-quota -n ai-team-a

### 28.3 Quota Effect

If the Namespace quota is set to 2 GPUs, and there are already two Pods each requesting 1 GPU, creating a third GPU Pod may be rejected.

Check events:

    kubectl get events -n ai-team-a --sort-by=.lastTimestamp

### 28.4 Production Recommendations

GPU resources must undergo quota governance.

Recommend allocating resources based on:

- Team;
- Environment;
- Project;
- Training/Inference;
- Priority;

---

## Twenty-NineI don't know.GPU Pod and PriorityClass

GPU resources are expensive, and production environments may require priority levels.

### 29.1 Creating PriorityClass

    apiVersion: scheduling.k8s.io/v1
    kind: PriorityClass
    metadata:
      name: gpu-prod-high
    value: 100000
    globalDefault: false
    description: "High priority for production GPU workloads"

### 29.2 Pod Using PriorityClass

    priorityClassName: gpu-prod-high

### 29.3 Notes

Preemption may cause lower-priority Pods to be terminated.

Training tasks without checkpoints may result in significant losses.

Production recommendations:

- Core inference services have high priority;
- Offline training tasks have medium priority;
- Experimental tasks have low priority;
- Preemption strategies must be confirmed by business;
- Training tasks must support checkpoints.

---

## ThirtyI don't know.GPU Pod Production Template

The following is a relatively complete GPU inference service template.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-inference-demo
  namespace: ai-prod
  labels:
    app: gpu-inference-demo
    workload-type: inference
spec:
  replicas: 1
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: gpu-inference-demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: gpu-inference-demo
        workload-type: inference
    spec:
      priorityClassName: gpu-prod-high
      nodeSelector:
        accelerator: nvidia
        gpu.workload: inference
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      terminationGracePeriodSeconds: 6
      containers:
        - name: inference
          image: registry.example.com/ai/inference-service:1.0.0-cuda12.2
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: NVIDIA_DRIVER_CAPABILITIES
              value: "compute,utility"
          resources:
            requests:
              cpu: "4"
              memory: "16Gi"
            limits:
              cpu: "8"
              memory: "32Gi"
              nvidia.com/gpu: 1
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 60
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 20

Notes:

- The image must be fixed to a specific version.
- GPU nodes are specified via nodeSelector.
- GPU node taints are tolerated via tolerations.
- CPU/memory request/limit must be reasonable.
- GPU is requested via nvidia.com/gpu.
- startupProbe should allow sufficient time for model loading.
- readinessProbe controls whether traffic is accepted.
- livenessProbe should not be overly aggressive.

---

## 31. Common GPU Pod Fault Layer Table

| Phenomenon | Priority Troubleshooting Layer | Common Causes |
|---|---|---|
| Pod remains in Pending state | Scheduler | Insufficient GPU, Taint, Label, Quota, CPU/Memory insufficient |
| Pod Running but nvidia-smi unavailable | Runtime / Container | Toolkit, containerd, device mounting, image issues |
| Host nvidia-smi normal, Node has no nvidia.com/gpu | Device Plugin | Plugin not running, registration failure, GPU health abnormal |
| Container torch.cuda.is_available is False | Framework / Image | CPU version framework, CUDA version mismatch, driver incompatibility |
| CUDA out of memory | Memory / Application | Batch size too large, model too big, multi-process memory contention |
| Low GPU utilization | Application / Data Pipeline | Slow data loading, CPU bottleneck, business idle |
| Multiple GPU Pod Pending | Scheduling / Node Capacity | Single node lacks sufficient GPU |
| Inference service repeatedly restarts | Application / Probes | Slow model loading, overly aggressive probes, memory insufficient |
| Pod memory not released after deletion | Runtime / Residual Processes | Container not exited, residual processes, runtime anomaly |
| Too many ordinary Pods on GPU node | Scheduling Governance | No Taint set, lack of node pool isolation |

## 32. Common Commands for GPU Pod Maintenance

### 32.1 View Pod

    kubectl get pod -A -o wide
    kubectl describe pod <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace> --previous

### 32.2 View Node

    kubectl get nodes -o wide
    kubectl describe node <gpu-node-name>
    kubectl get node <gpu-node-name> --show-labels

### 32.3 View GPU Resources

    kubectl describe node <gpu-node-name> | grep -A10 -B5 "nvidia.com/gpu"

### 32.4 View NVIDIA Components

    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia
    kubectl logs <nvidia-pod-name> -n <namespace>

### 32.5 View Events

    kubectl get events -A --sort-by=.lastTimestamp
    kubectl get events -n <namespace> --sort-by=.lastTimestamp

### 32.6 Container Verification

    kubectl exec -it <pod-name> -n <namespace> -- sh
    nvidia-smi
    ls -l /dev/nvidia*
    echo $CUDA_VISIBLE_DEVICES
    echo $NVIDIA_VISIBLE_DEVICES

### 32.7 Node Local Verification

    nvidia-smi
    nvidia-smi -L
    nvidia-smi topo -m
    dmesg | grep -i xid
    nvidia-container-cli info
    containerd config dump | grep -i nvidia -A30 -B10

---

## 33. Deployment Recommendations for Production GPU Pods

### 33.1 Image Standards

Recommendations:

- Prohibit use of latest;
- Use fixed versions;
- Use internal image registry;
- Distinguish between runtime and devel;
- Perform vulnerability scanning;
- Record CUDA / cuDNN / TensorRT / PyTorch versions;
- Ensure traceability of image build process.

### 33.2 Resource Standards

Recommendations:

- Must specify CPU request;
- Must specify Memory request;
- Must specify GPU limit;
- Set appropriate replica count for inference services;
- Use Jobs for training tasks;
- Set low priority for experimental tasks;
- Set ResourceQuota for Namespace.

### 33.3 Scheduling Standards

Recommendations:

- Label GPU nodes;
- Add taints to GPU nodes;
- Add nodeSelector/affinity to GPU Pods;
- Add toleration to GPU Pods;
- Partition by GPU model;
- Partition inference and training;
- Partition production and experimental.

### 33.4 Observability Standards

Recommend monitoring:

- GPU utilization;
- Memory usage;
- GPU temperature;
- GPU power consumption;
- XID errors;
- GPU Pod Pending;
- GPU Pod Restart;
- Device Plugin status;
- DCGM Exporter status;
- Namespace GPU usage;
- Idle GPUs;
- GPU cost and utilization.

### 33.5 Operations Standards

Recommendations:

- Cordon/drain before node maintenance;
- Support checkpoint for training tasks;
- Support graceful shutdown for inference services;
- Integrate business logs with Loki/EFK;
- Integrate GPU metrics with Prometheus;
- Integrate alerts with AlertManager;
- Preserve deployment YAML;
- Preserve version matrix;
- Establish rollback process.

---

## 34. Complete Checklist for GPU Pod Deployment

### 34.1 Before Deployment

    [ ] GPU node Ready
    [ ] lspci can see NVIDIA GPU
    [ ] nvidia-smi normal
    [ ] NVIDIA Container Toolkit normal
    [ ] Device Plugin / GPU Operator normal
    [ ] Node shows nvidia.com/gpu
    [ ] GPU node Label correct
    [ ] GPU node Taint planned
    [ ] Namespace ResourceQuota confirmed
    [ ] Image CUDA version compatible with driver
    [ ] Image can be pulled from registry
    [ ] Pod YAML declared GPU
    [ ] CPU/Memory request reasonable
    [ ] nodeSelector/affinity correct
    [ ] toleration correct

### 34.2 During Deployment

    [ ] Pod creation successful
    [ ] Pod scheduled to GPU node
    [ ] No ImagePullBackOff
    [ ] No insufficient nvidia.com/gpu
    [ ] No taint/toleration error
    [ ] No nodeSelector/affinity error
    [ ] No quota error
    [ ] No CPU/Memory insufficient error

### 34.3 After Deployment

    [ ] Pod Running
    [ ] nvidia-smi normal inside container
    [ ] /dev/nvidia* exists
    [ ] CUDA_VISIBLE_DEVICES normal
    [ ] PyTorch/TensorFlow can recognize GPU
    [ ] GPU memory usage meets expectation
    [ ] GPU utilization meets expectation
    [ ] No CUDA errors in logs
    [ ] Prometheus can collect GPU metrics
    [ ] Grafana can view GPU dashboard
    [ ] Alert rules cover critical anomalies

---

## 35. Summary

# Core of GPU Pod Deployment

The core of GPU Pod deployment is not writing a YAML that can run, but verifying whether the Kubernetes GPU usage chain is complete.

Complete chain:

    GPU Hardware
      ↓
    NVIDIA Driver
      ↓
    NVIDIA Container Toolkit
      ↓
    Device Plugin / GPU Operator
      ↓
    Node registration nvidia.com/gpu
      ↓
    Pod declaration nvidia.com/gpu
      ↓
    Scheduler scheduling to GPU node
      ↓
    kubelet starts container
      ↓
    containerd mounts GPU device
      ↓
    nvidia-smi normal in container
      ↓
    CUDA / AI framework normal
      ↓
    Business truly uses GPU

Troubleshooting should clearly distinguish stages:

    Pod Pending:
        Check Scheduler, resources, Taint, Label, Quota.

    Pod Running but no GPU:
        Check Runtime, Toolkit, Device Plugin, image, device mounting.

    nvidia-smi normal but framework unavailable:
        Check PyTorch/TensorFlow, CUDA Runtime, cuDNN, driver compatibility.

    CUDA OOM:
        Check memory, model, batch size, multi-process, MIG/shared strategy.

    Low GPU utilization:
        Check data loading, CPU, network, storage, business traffic and application logic.

In production environments, GPU Pod must follow these principles:

- Fixed image version;
- Complete declaration of CPU / Memory / GPU resources;
- Clear GPU node Label / Taint;
- Clear Namespace quota;
- Separate inference and training pools;
- Separate test and production pools;
- Monitoring and log integration;
- Closed-loop alerts;
- Training tasks support checkpoint;
- Inference services support probes and graceful stop;
- Observable GPU utilization and cost.

Do not understand GPU Pod deployment as:

    resources:
      limits:
        nvidia.com/gpu: 1

This is just the entry point.

True GPU Pod operations capability is the ability to comprehensively judge whether GPU workload is stably running from YAML, scheduling, runtime, CUDA, AI framework, metrics, logs, and business performance.

---

## Reference Documents

- Kubernetes GPU Scheduling:
  https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/

- Kubernetes Assign Pods to Nodes:
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/

- Kubernetes Taints and Tolerations:
  https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

- Kubernetes Resource Quotas:
  https://kubernetes.io/docs/concepts/policy/resource-quotas/

- Kubernetes Jobs:
  https://kubernetes.io/docs/concepts/workloads/controllers/job/

- Kubernetes Deployments:
  https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

- NVIDIA Kubernetes Device Plugin:
  https://github.com/NVIDIA/k8s-device-plugin

- NVIDIA GPU Operator:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/

- NVIDIA Container Toolkit:
  https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

- NVIDIA CUDA Container Images:
  https://hub.docker.com/r/nvidia/cuda

- NVIDIA DCGM Exporter:
  https://github.com/NVIDIA/dcgm-exporter