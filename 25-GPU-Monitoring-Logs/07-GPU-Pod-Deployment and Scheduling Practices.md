# 07-GPU-Pod-Deployment and Scheduling Practices

## Document Description

This document outlines how to deploy, schedule, verify, troubleshoot, and use GPU Pods in a Kubernetes cluster.

It addresses the following key questions:

- What prerequisites should be checked before deploying a GPU Pod?
- How to confirm that Kubernetes nodes recognize `nvidia.com/gpu`?
- How to create a minimal GPU Pod configuration?
- How to verify that the GPU is accessible within the container?
- How to use `nodeSelector`, `nodeAffinity`, and `taint/toleration` to control GPU Pod scheduling?
- How to deploy GPU inference services using Deployments?
- How to deploy GPU training tasks using Jobs?
- How to troubleshoot GPU Pods that remain in a "Pending" state?
- How to resolve issues where a Pod is running but the GPU is not visible?
- How to verify whether CUDA, PyTorch, or TensorFlow can utilize the GPU?
- How to understand the relationship between CPU, memory, and GPU resources in a GPU Pod?
- What should the YAML configuration for GPU Pods look like in a production environment?
- How to clean up, reclaim, and manage GPU Pod resources effectively?

This document is recommended after completing the following chapters:

- 03-NVIDIA-Driver Installation and Verification
- 04-CUDA-Installation and Testing
- 05-K8S-GPU-Resource Concepts and Scheduling Principles
- 06-NVIDIA-Device-Plugin-and-Operator-Installation

This document does not delve into the details of installing Device Plugins or GPU Operators, but focuses on practical deployment and scheduling of GPU Pods.

---

## Tags

#Kubernetes #GPU #NVIDIA #CUDA #DevicePlugin #GPUOperator #PodScheduling #AIInfrastructure #SRE #OperationsTroubleshooting

---

## Recommended Reading Path

Recommended path:

    06-GPUandAIInfrastructure/02-Kubernetes-GPUScheduling/07-GPU-Pod-Deployment-and-Scheduling Practices.md

---

## I. Core Objectives of GPU Pod Deployment Practices

Deploying a GPU Pod is not just about writing a YAML file:

    resources:
      limits:
        nvidia.com/gpu: 1

The real goal of practical GPU Pod deployment is to verify the entire chain of operations:

- The GPU hardware is functioning correctly.
- The NVIDIA Driver is installed and working properly.
- `nvidia-smi` can be executed successfully.
- The NVIDIA Container Toolkit is configured correctly.
- The Device Plugin/GPU Operator are operational.
- The node recognizes `nvidia.com/gpu`.
- The Pod successfully requests the GPU resource.
- The scheduler assigns the Pod to a GPU-enabled node.
- The kubelet launches the container.
- `containerd` mounts the GPU device.
- `nvidia-smi` can be run within the container.
- CUDA, PyTorch, or TensorFlow can recognize and utilize the GPU.
- The business tasks can effectively use the GPU.

Therefore, when evaluating GPU Pod deployment, it is essential to check not only:

    kubectl get pod

but also:

    kubectl describe pod
    kubectl describe node
    kubectl logs
    nvidia-smi
    /dev/nvidia*
    CUDA_VISIBLE_devices
    PyTorch cuda.is_available
    Prometheus GPU metrics

---

## II. Pre-deployment Prerequisites for GPU Pods

Before deploying a GPU Pod, do not start writing the YAML file directly.

First, verify the status of the cluster and nodes.

### 2.1 Checking Kubernetes Nodes

    kubectl get nodes -o wide

Expected outcome:

- GPU nodes should be in the "Ready" state.

If any GPU node is in the "NotReady" state, troubleshoot issues related to kubelet, containerd, CNI, or node resource constraints.

### 2.2 Checking GPU Node Labels

    kubectl get node <gpu-node-name> --show-labels

At least the following labels should be present:

    accelerator=nvidia
    node-role.kubernetes.io/gpu=true
    gpu.vendor=nvidia
    gpu.model=<model>
    gpu.workload=<training|inference|dev>

If these labels are missing, add them later.

### 2.3 Checking GPU Node Taints

    kubectl describe node <gpu-node-name> | grep -i taints -A5

Common GPU node taints include:

    nvidia.com/gpu=true:NoSchedule

If a node has this taint, the GPU Pod must be configured with a tolerance.

### 2.4 Checking if the Node Recognizes GPU Resources

    kubectl describe node <gpu-node-name>

Focus on the following fields:

    Capacity:
      nvidia.com/gpu: 1

    Allocatable:
      nvidia.com/gpu: 1

If `nvidia.com/gpu` is not listed, it means Kubernetes cannot schedule GPU### 4.1 Declaration of GPU Usage Limits

When applying for a GPU in a GPU Pod:

    resources:
      limits:
        nvidia.com/gpu: 1

Kubernetes has special rules for extended resources like GPUs:

    Only limits can be specified for GPUs.
    Kubernetes will use the limits as requested values.
    If both requests and limits are specified, they must be equal.
    It is not allowed to specify only requests without limits.

### 4.2 Correct Ways to Write

Ways 1 and 2:

    resources:
      limits:
        nvidia.com/gpu: 1

Ways 3 and 4 (incorrect):

    resources:
      requests:
        nvidia.com/gpu: 1
    resources:
      requests:
        nvidia.com/gpu: 1
      limits:

Problem:

    Specifying only GPU requests without limits violates the usage rules for extended GPU resources.

### 4.3 Unrecommended Ways to Write

Incorrect example:

    resources:
      limits:
        nvidia.com/gpu: 0.5

By default, Kubernetes allocates GPU resources as whole units.

To share a GPU, you need to use:

- MIG;
- Time-slicing;
- MPS;
- Third-party vGPU/GPU Share solutions;
- AI platform-level resource management.

You cannot simply specify `0.5`.

---

## Section 5: GPU Pods and CPU/Memory Resources

A GPU Pod should not rely solely on GPUs.

In a production environment, both CPU and memory must be considered.

### 5.1 Why GPU Pods Also Need CPUs

GPU tasks often require CPUs for:

- Data preprocessing;
- Image decoding;
- Text tokenization;
- Network request handling;
- Model preprocessing and post-processing;
- Log output;
- Process scheduling;
- Loading data from disks or networks into GPUs.

Insufficient CPU resources can lead to:

- Low GPU utilization;
- Slow training speeds;
- High inference delays;
- Data loading bottlenecks;
- Idle GPUs.

### 5.2 Why GPU Pods Also Need Memory

Memory is essential for:

- Loading model files;
- Caching input data;
- Running Python processes and frameworks;
- Performing data preprocessing;
- Storing logs and intermediate objects.

Insufficient memory can cause a Pod to experience:

    OOMKilled

To check this, use the following commands:

    kubectl describe pod <pod-name> -n <namespace>
    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -i oom -A5 -B5

### 5.3 Examples of Production GPU Pod Resources

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

    The value `32Gi` refers to container memory, not GPU video memory.
    GPU video memory is not directly limited by Kubernetes' memory limits.

---

## Section 6: Using nodeSelector to Control GPU Pod Scheduling

### 6.1 Tagging GPU Nodes

    kubectl label node <gpu-node-name> accelerator=nvidia
    kubectl label node <gpu-node-name> gpu.workload=inference
    kubectl label node <gpu-node-name> gpu.model=a100

To check the tags, use:

    kubectl get node <gpu-node-name> --show-labels

### 6.2 Using nodeSelector in a Pod Specification

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

To deploy this Pod, use:

    kubectl apply -f gpu-test-nodeselector.yaml

To view the deployed Pod, use:

    kubectl get pod gpu-test-nodeselector -o wide

### 6.3 Appropriate Scenarios for nodeSelector

Suitable for:

- Simply specifying GPU nodes;
- Distinguishing between CPU and GPU nodes;
- Experimental environments;
- Small clusters;
- Simple model isolation.

Not suitable for:

- Complex multi-condition scheduling;
- Multiple GPU models;
- Mixed training/inference scenarios;
- Scheduling with soft constraints as```yaml
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
```

### 8.3 Using nodeSelector and toleration Together

It is more recommended in production scenarios to use both:

```yaml
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
```

**Explanation:**

- `nodeSelector` specifies that the Pod should be scheduled on nodes with NVIDIA accelerators.
- `toleration` allows the Pod to be scheduled on nodes that have GPU-related taints, even if it conflicts with the `nodeSelector`.

Using both ensures that Pods requiring GPUs are placed on suitable nodes while also accommodating potential node constraints. It is recommended to use them together for optimal performance and reliability.### Kubernetes Does Not Distribute Two GPUs in a Pod Across Two Nodes by Default

---

### 10.3 Notes on Producing Training Tasks

For training tasks, it is recommended to pay attention to the following aspects:

- Checkpoints;
- Retry upon failure;
- Data mounting;
- Log saving;
- Model output path;
- Training duration;
- GPU usage rate;
- Video memory consumption;
- Multi-GPU communication;
- Distributed training;
- Automatic cleanup after task completion;
- Avoiding that low-priority experimental tasks consume production GPUs.

---

## XI. Using GPUs in Multi-Container Pods

### 11.1 Sidecars Should Not Request GPUs Arbitrarily

A Pod can contain multiple containers.

If only the main business container requires a GPU, do not allow the sidecar to also request one.

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

### 11.2 Pay Attention to Resource Attribution

GPU resources are declared at the container level.

Do not assume that they are automatically allocated to all containers at the Pod level.

In production, it is essential to clarify:

- Which container needs a GPU;
- Which containers are only used for logging, proxying, or monitoring;
- Whether the sidecar will affect CPU/memory usage;
- Whether the sidecar requires access to model files;
- Whether the sidecar will increase the resource load on the node.

---

## XII. Verifying GPU Visibility Within Containers

### 12.1 Entering the Pod

    kubectl exec -it <pod-name> -n <namespace> -- bash

If bash is not available:

    kubectl exec -it <pod-name> -n <namespace> -- sh

### 12.2 Checking GPUs

    nvidia-smi

### 12.3 Viewing Device Files

    ls -l /dev/nvidia*

Expected output:

    /dev/nvidia0
    /dev/nvidiactl
    /dev/nvidia-uvm

### 12.4 Checking Environment Variables

    echo $CUDA_VISIBLE_devices
    echo $NVIDIAVISIBLE_DEVICES
    echo $NVIDIA_DRIVER_CAPABILITIES

Explanation:

    CUDA_VISIBLE Devices:
        Controls which GPU devices are visible to CUDA applications.

    NVIDIA_VISIBLE_DEVICES:
        The device visibility variable used by the NVIDIA Container Runtime.

    NVIDIA DRIVER_CAPABILITIES:
        Determines which driver capabilities are exposed, such as compute or utility functions.

### 12.5 Viewing Processes

Within the container:

    nvidia-smi

On the host machine:

    nvidia-smi

This can help identify which process is using the GPU.

If it is a Kubernetes container process, you can also use containerd/crictl to investigate further.

---

## XIII. Verifying CUDA Capabilities

### 13.1 Using the Base Image to Verify nvidia-smi

    image: nvidia/cuda:12.2.0-base-ubuntu22.04
    command: ["nvidia-smi"]

Verification points:

- Whether GPU devices are visible;
- Whether the driver library has been successfully installed;
- Whether nvidia-smi is functional.

### 13.2 Using the Developer Image to Verify nvcc

    image: nvidia/cuda:12.2.0-devel-ubuntu22.04
    command: ["bash", "-lc", "nvidia-smi && nvcc -V"]

Verification points:

- Whether the CUDA Toolkit compilation tools are present;
- Whether nvcc is available.

Note:

    The base/runtime images may not include nvcc.
    Developer images usually contain nvcc.
    It is not recommended to use developer images in production services.

### 13.3 Using the Runtime Image to Verify the Running Environment

    image: nvidia/cuda:12.2.0-runtime-ubuntu22.04

This image is suitable for running production services.

If your business application only requires running CUDA applications without compilation, a runtime image is more appropriate.

---

## XIV. Verifying GPU Usage in PyTorch

### 14.1 PyTorch Test Command

Execute within the container:

    python3 -c "import torch; print('torch:', torch.__version__); print('cuda version:', torch.version.cuda); print('cuda available:', torch.cuda.is_available()); print('device count:', torch.cuda.device_count())"

Expected output:

    cuda available: True
    device count: 1

### 14.2 Example of a PyTorch Test Pod

The actual image version should be chosen based on your specific requirements.

Example Pod configuration### 16.3 Insufficient nvidia.com/gpu

Meaning:

    The cluster currently does not have enough available GPUs to allocate.

Troubleshooting:

    kubectl describe node <gpu-node-name>
    kubectl get pods -A -o wide | grep <gpu-node-name>
    kubectl describe nodes | grep -A10 -B5 "nvidia.com/gpu"

Check:

- Whether the node has `nvidia.com/gpu`;
- Whether the GPU is already occupied by another Pod;
- Whether the Pod has requested too many GPUs;
- Whether a single node can support multiple GPUs;
- Whether the Device Plugin is functioning correctly;
- Whether the GPU is marked as unhealthy.

### 16.4 Untolerated Taint

Meaning:

    The node has a taint, and the Pod does not have the corresponding tolerance setting.

View node taints:

    kubectl describe node <gpu-node-name> | grep -i taints -A5

Add tolerance to the Pod:

    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"

### 16.5 NodeSelector / Affinity Mismatch

View the Pod:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A20 nodeSelector
    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A40 affinity

View node labels:

    kubectl get node <gpu-node-name> --show-labels

Solution:

- Modify the Pod's nodeSelector / affinity settings;
- Add the correct labels to the node;
- Determine if the business logic requires scheduling on that specific GPU model.

### 16.6 Insufficient CPU / Memory

If the issue is:

    insufficient cpu
    insufficient memory

It indicates that the problem lies not with the GPU, but with the CPU or memory requirements of the application.

Solution:

- Reduce the resource requests;
- Increase the node's available resources;
- Optimize the load on the node;
- Replace the node with one equipped with a larger GPU;
- Rebalance resource allocation.

### 16.7 Exceeded Quota

Check the quota settings:

    kubectl describe resourcequota -n <namespace>

If the GPU quota has been exhausted, you need to:

- Remove any unnecessary GPU-related Pods;
- Adjust the ResourceQuota settings;
- Switch to a different Namespace if possible;
- Submit a formal request for additional resources.

---

## Section Seventeen: Troubleshooting When a Pod is Running but the GPU is Not Visible Inside the Container

### 17.1 Verify Whether the Pod Has Requested a GPU

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A15 resources

You should see:

    limits:
      nvidia.com/gpu: 1

If this option is missing, it means the Pod is not configured to use a GPU.

### 17.2 Check Inside the Container

    kubectl exec -it <pod-name> -n <namespace> -- sh

Run the following commands inside the container:

    nvidia-smi
    ls -l /dev/nvidia*
    echo $CUDA_VISIBLE_devices
    echo $NVIDIA_VISIBLE_DEVICES

### 17.3 Check if the Image Includes nvidia-smi

Some application images do not include `nvidia-smi`.

This does not necessarily mean that the GPU is unavailable.

You can still check:

    ls -l /dev/nvidia*
    python3 -c "import torch; print(torch.cuda.is_available())"

Alternatively, use an official CUDA test image to verify the GPU connection.

### 17.4 Troubleshoot with the Node Runtime

On a GPU-enabled node, execute the following commands:

    nvidia-container-cli info
    containerd config dump | grep -i nvidia -A30 -B10
    systemctl status containerd
    systemctl status kubelet

If necessary, restart the services:

    systemctl restart containerd
    systemctl restart kubelet

After restarting, you will need to recreate the Pod.

### 17.5 Check the Device Plugin

    kubectl get pods -A | grep -i nvidia
    kubectl logs <device-plugin-pod> -n <namespace>

If a GPU Operator is being used:

    kubectl get pods -n gpu-operator -o wide
    kubectl logs <nvidia-device-plugin-pod> -n gpu-operator

---

## Section Eighteen: Troubleshooting CUDA Out of Memory Issues in GPU Pods

### 18.1 Symptoms

In application logs, you may see:

    CUDA out of memory

Or in PyTorch, you might encounter an error:

    RuntimeError: CUDA out of memory

### 18.2 Check Available Video## Twenty-Eight, GPU Pods and ResourceQuotas### 28.1 Creating a GPU Namespace

    kubectl create namespace ai-team-a

### 28.2 Creating a GPU Quota

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

Application:

    kubectl apply -f gpu-quota.yaml

Verification:

    kubectl describe resourcequota gpu-quota -n ai-team-a

### 28.3 Quota Effects

If the namespace quota specifies 2 GPUs, and two pods have already each requested one GPU, attempting to create a third GPU Pod will likely result in rejection.

Monitoring Events:

    kubectl get events -n ai-team-a --sort-by=.lastTimestamp

### 28.4 Production Recommendations

It is essential to manage GPU resources through quotas.

It is recommended to allocate resources based on:

- Team;
- Environment;
- Project;
- Training/Inference Purpose;
- Priority Level;

---

## Chapter Twenty-Nine: GPU Pods and PriorityClasses

Since GPU resources are expensive, priority settings may be necessary in a production environment.

### 29.1 Creating a PriorityClass

    apiVersion: scheduling.k8s.io/v1
    kind: PriorityClass
    metadata:
      name: gpu-prod-high
    value: 100000
    globalDefault: false
    description: "High priority for production GPU workloads"

### 29.2 Using a PriorityClass in a Pod

    priorityClassName: gpu-prod-high

### 29.3 Precautions

Preempting other pods with higher priorities may cause them to terminate.

Training tasks without checkpoints can result in significant data loss.

Production recommendations:

- Core inference services should have high priority;
- Offline training tasks should be given medium priority;
- Experimental tasks should have low priority;
- Preemption strategies must be approved by the business team;
- Training tasks must support checkpoints.

---

## Chapter Thirty: GPU Pod Production Template

Below is a relatively complete template for a GPU inference service.

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
          terminationGracePeriodSeconds: 60
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

    The image version must be fixed.
    GPU nodes are specified through the nodeSelector.
    GPU node taints are managed by tolerations.
    CPU/memory requests/limits must be appropriate.
    GPUs are requested using nvidia.com/gpu.
    The startupProbe ensures sufficient time for model loading.
    The readinessProbe controls whether traffic is allowed.
    The livenessProbe should not be set too aggressively.

---

## Chapter Thirty-One: Common Fault Layers for GPU Pods

| Phenomenon | Priority```markdown
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous

### 32.2 View Nodes

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

### 32.6 Verify Inside the Container

    kubectl exec -it <pod-name> -n <namespace> -- sh
    nvidia-smi
    ls -l /dev/nvidia*
    echo $CUDA_VISIBLE_devices
    echo $NVIDIAVISIBLE_DEVICES

### 32.7 Verify Locally on the Node

    nvidia-smi
    nvidia-smi -L
    nvidia-smi topo -m
    dmesg | grep -i xid
    nvidia-container-cli info
    containerd config dump | grep -i nvidia -A30 -B10

---
## Section 33: Recommendations for Deploying GPU Pods in a Production Environment

### 33.1 Image Specification

Recommendations:

- Avoid using the "latest" version;
- Use fixed versions;
- Utilize an internal image repository;
- Distinguish between runtime and devel images;
- Perform vulnerability scans;
- Record the CUDA, cuDNN, TensorRT, and PyTorch versions;
- Ensure the image build process is traceable.

### 33.2 Resource Specification

Recommendations:

- Always specify CPU and Memory requests;
- Set clear GPU limits;
- Configure appropriate numbers of replicas for inference services;
- Use Jobs for training tasks;
- Assign low priority to experimental tasks;
- Set ResourceQuotas in the Namespace.

### 33.3 Scheduling Specification

Recommendations:

- Label GPU nodes;
- Apply taints to GPU nodes;
- Add nodeSelector/affinity settings to GPU Pods;
- Include toleration rules for GPU Pods;
- Pool multiple types of GPUs;
- Separate inference and training tasks;
- Allocate different pools for production and experimental use.

### 33.4 Observability Specification

It is recommended to monitor the following:

- GPU utilization;
- Video memory usage;
- GPU temperature;
- GPU power consumption;
- XID errors;
- GPU Pod Pending status;
- GPU Pod Restart events;
- Device Plugin status;
- DCGM Exporter status;
- Namespace-level GPU usage;
- Available idle GPUs;
- GPU costs and efficiency.

### 33.5 Operation and Maintenance Specification

It is recommended to:

- Drain resources from nodes before maintenance;
- Support checkpointing for training tasks;
- Enable graceful shutdown of inference services;
- Integrate business logs into Loki/EFK;
- Connect GPU metrics to Prometheus;
- Set up AlertManager for alerts;
- Keep the deployment YAML files and version matrices;
- Establish a rollback process.
---

## Section 34: Comprehensive Checklist for Deploying GPU Pods

### 34.1 Before Deployment

    [ ] GPU nodes are ready for use.
    [ ] NVIDIA GPUs are visible through lspci.
    [ ] nvidia-smi is functioning correctly.
    [ ] The NVIDIA Container Toolkit is operational.
    [ ] Device Plugins and GPU Operators are working properly.
    [ ] Nodes display "nvidia.com/gpu" label.
    [ ] Correct labels have been applied to GPU nodes.
    [ ] Taints for GPU nodes have been planned accordingly.
    [ ] ResourceQuotas in the Namespace have been confirmed.
    [ ] The CUDA version in the image is compatible with the driver.
    [ ] Images can be pulled from the repository successfully.
    [ ] Pod YAML files explicitly declare the use of GPUs.
    [ ] CPU and Memory requests are reasonable.
    [ ] nodeSelector/affinity settings are correct.
    [ ] Toleration rules are properly configured.

### 34.2 During Deployment

    [ ] Pods are created successfully.
    [ ] Pods are scheduled to GPU nodes without issues.
    [ ] There are no ImagePullBackOff errors.
    [ ] No insufficient "nvidia.com/gpu" labels are detected.
    [ ] No taint/tolerance-related errors occur.
    [ ] No nodeSelector/affinity configuration errors exist.
    [ ] No quota-related issues are found.
    [https://kubernetes.io/docs/concepts/workloads/controllers/job/

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