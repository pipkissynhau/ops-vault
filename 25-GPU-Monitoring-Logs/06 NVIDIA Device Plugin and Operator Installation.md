# 06-NVIDIA-Device-Plugin-and-Operator-Installation

## Document Description

This document is used to organize the installation, selection, deployment prerequisites, verification methods, common troubleshooting, and production environment deployment recommendations for NVIDIA Device Plugin and NVIDIA GPU Operator in Kubernetes clusters.

This document focuses on answering the following questions:

- Why does Kubernetes need the NVIDIA Device Plugin;
- What problems do Device Plugin and GPU Operator respectively solve;
- In what scenarios should only the Device Plugin be installed;
- In what scenarios should the GPU Operator be used;
- What components does the GPU Operator manage;
- How to configure NVIDIA Container Toolkit in containerd environments;
- Whether the GPU Operator still needs to install drivers when GPU nodes already have drivers;
- How to verify `nvidia.com/gpu` after Device Plugin deployment;
- What Pods, DaemonSets, and ClusterPolicies should be checked after GPU Operator deployment;
- How to verify `nvidia-smi` inside containers after Pod requests GPU;
- How to troubleshoot installation failures, nodes not showing GPUs, Pod Pending, and no GPU inside containers;
- What to pay attention to in domestic network environments, private image repositories, and production clusters.

This document is suitable for study after completing the following prerequisites:

- 01-GPU-Basic-Concepts-and-Hardware-Composition
- 02-GPU-BIOS-and-Hardware-Tuning
- 03-NVIDIA-Driver-Installation-and-Verification
- 04-CUDA-Installation-and-Testing
- 05-K8S-GPU-Resource-Concepts-and-Scheduling-Principles

This document does not repeat details on NVIDIA driver and CUDA installation, focusing instead on GPU access at the Kubernetes layer.

---

## Tags

#Kubernetes #GPU #NVIDIA #DevicePlugin #GPUOperator #Containerd #Helm #DCGM #CUDA #SRE #TransportBarriers

---

## Recommended Path

Recommended path:

    06-GPU-and-AI-Infrastructure/02-Kubernetes-GPU-Scheduling/06-NVIDIA-Device-Plugin-and-Operator-Installation.md

---

## One, Why Need Device Plugin and GPU Operator

Kubernetes can natively recognize built-in resources such as CPU, Memory, and Ephemeral Storage.

However, Kubernetes does not inherently know how many NVIDIA GPUs are present on a node.

Even if executing:

    nvidia-smi

shows GPUs on the host, it does not mean the Kubernetes Scheduler can see GPUs.

To enable Kubernetes to schedule GPUs, kubelet must register GPUs as extended resources to the Node Status.

This ultimately manifests as:

    Capacity:
      nvidia.com/gpu: 1

    Allocatable:
      nvidia.com/gpu: 1

This step is typically completed by the NVIDIA Device Plugin.

The core function of the Device Plugin is:

    Discover NVIDIA GPUs on the node
    ↓
    Register GPU devices with kubelet
    ↓
    Make nvidia.com/gpu appear on the Node
    ↓
    Pods can request GPUs via resources.limits.nvidia.com/gpu

However, a GPU node involves more than just the Device Plugin.

A complete GPU Kubernetes node may also involve:

- NVIDIA Driver;
- NVIDIA Container Toolkit;
- NVIDIA Container Runtime;
- NVIDIA Device Plugin;
- GPU Feature Discovery;
- Node Feature Discovery;
- DCGM Exporter;
- MIG Manager;
- Driver Toolkit;
- GPU Operator;
- RuntimeClass;
- containerd / Docker configuration;
- Prometheus monitoring;
- Grafana Dashboard;
- AlertManager alerts.

If only the Device Plugin is installed, many components need manual maintenance.

If the GPU Operator is used, it can centrally manage these NVIDIA components via the Operator.

---

## Two, Differences Between Device Plugin and GPU Operator

### 2.1 NVIDIA Device Plugin

The Device Plugin is a lightweight approach.

It mainly handles:

- Discovering GPUs;
- Registering GPUs with kubelet;
- Making `nvidia.com/gpu` appear on the Node;
- Coordinating with kubelet and Runtime to allocate GPUs to containers;
- Supporting basic GPU Pod scheduling;
- Supporting some advanced configurations, such as MIG and time-slicing, depending on version and configuration.

It typically runs as a DaemonSet.

The Device Plugin does not handle:

- Installing NVIDIA Driver;
- Installing CUDA;
- Installing NVIDIA Container Toolkit;
- Configuring containerd;
- Managing DCGM Exporter;
- Managing GPU node labels;
- Managing MIG lifecycle;
- Automatically fixing GPU node environments;
- Central management of the entire NVIDIA software stack.

### 2.2 NVIDIA GPU Operator

The GPU Operator is a more comprehensive automated management solution.

It manages GPU node-related components via Kubernetes Operator pattern.

It can typically manage:

- NVIDIA Driver;
- NVIDIA Container Toolkit;
- NVIDIA Device Plugin;
- NVIDIA DCGM;
- DCGM Exporter;
- GPU Feature Discovery;
- Node Feature Discovery;
- MIG Manager;
- Validator;
- Runtime configuration;
- GPU node labels;
- GPU monitoring components.

The GPU Operator acts as a controller for the GPU node software stack.

It is suitable for:

- Multi-GPU nodes;
- Multi-cluster;
- Production Kubernetes;
- Want to unify management of NVIDIA components;
- Want to reduce manual maintenance;
- Need MIG;
- Need DCGM monitoring;
- Need standardized GPU node delivery;
- Need subsequent upgrades and version management.

---

## III. Choosing Between Two Deployment Routes

### 3.1 Scenario of Installing Only Device Plugin

Suitable for:

- Experimental environments;
- Small-scale GPU nodes;
- Nodes already manually installed with NVIDIA Driver;
- NVIDIA Container Toolkit manually configured;
- Only need Kubernetes to schedule GPU;
- Do not want Operator to manage drivers and runtime;
- Operations team wants full control over all node software versions;
- Temporarily do not need MIG, DCGM, GPU Feature Discovery, etc. complete capabilities.

Typical workflow:

    Manually install NVIDIA Driver
    ↓
    Manually install NVIDIA Container Toolkit
    ↓
    Manually configure containerd
    ↓
    Deploy NVIDIA Device Plugin
    ↓
    Node appears nvidia.com/gpu
    ↓
    Deploy GPU Pod test

### 3.2 Scenario of Using GPU Operator

Suitable for:

- Production GPU clusters;
- Multiple GPU nodes;
- Multiple GPU models;
- Need unified component versions;
- Need automatic deployment of Device Plugin;
- Need DCGM Exporter;
- Need GPU Feature Discovery;
- Need MIG management;
- Need unified operations for GPU nodes;
- Want to reduce manual configuration;
- Want GPU nodes to have more complete lifecycle management.

Typical workflow:

    Kubernetes cluster is ready
    ↓
    GPU nodes join the cluster
    ↓
    Helm install GPU Operator
    ↓
    Operator creates ClusterPolicy
    ↓
    Automatically deploy related DaemonSet / Pod
    ↓
    Node appears nvidia.com/gpu
    ↓
    DCGM Exporter exposes GPU metrics
    ↓
    Deploy GPU Pod test

### 3.3 Production Recommendations

If it's for learning or small-scale experiments, you can start with Device Plugin first, understanding how GPU becomes `nvidia.com/gpu`.

If it's for production environment, especially with multiple GPU nodes, it's recommended to prioritize evaluating GPU Operator first.

However, when using GPU Operator in production environments, pay special attention to:

- Whether the driver is managed by Operator;
- Whether the driver is already installed on nodes;
- Whether containerd is configured by Operator;
- Whether Operator is allowed to modify node runtime;
- Whether images can be pulled;
- How to configure private image registry;
- Whether versions are fixed;
- Whether there is a rollback plan;
- Whether it complies with enterprise change management process.

---

## IV. Installation Prerequisites

Regardless of using Device Plugin or GPU Operator, you need to first confirm the basic status of Kubernetes and nodes.

### 4.1 Kubernetes Cluster Status

Check nodes:

    kubectl get nodes -o wide

Check system Pods:

    kubectl get pods -A

Check kubelet status:

    systemctl status kubelet

Check containerd status:

    systemctl status containerd

Production recommendations:

    GPU components deployment must start with a stable cluster.
    If CNI, CoreDNS, or kubelet itself are abnormal, do not rush to deploy GPU components.

### 4.2 GPU Node Hardware Recognition

Run on GPU node:

    lspci | grep -i nvidia

If there is no output, it means the system has not recognized the GPU.

Prioritize troubleshooting:

- Whether GPU is properly inserted;
- Whether BIOS recognizes it;
- Above 4G Decoding;
- PCIe slot;
- riser;
- Power supply;
- Server compatibility.

### 4.3 NVIDIA Driver Status

Run on GPU node:

    nvidia-smi

If `nvidia-smi` is not normal, fix the driver first.

Check kernel modules:

    lsmod | grep nvidia

Check device files:

    ls -l /dev/nvidia*

Check XID:

    dmesg | grep -i xid
    journalctl -k | grep -i xid

### 4.4 containerd Status

Check containerd:

    systemctl status containerd

Check configuration:

    containerd config dump | grep -i nvidia -A20 -B5

If NVIDIA Runtime is not yet configured, subsequent containers may not be able to access GPU.

### 4.5 Helm Tool

Both GPU Operator and Device Plugin can be installed via Helm.

Check Helm:

    helm version

If Helm is not installed, it needs to be installed first.

Ubuntu example:

    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

Production recommendations:

    Helm installation packages should be downloaded from trusted sources.
    For enterprise internal network environments, it's recommended to maintain an internal Helm repository or artifact repository.

---

## V. containerd and NVIDIA Container Toolkit

### 5.1 Why NVIDIA Container Toolkit is Needed

Normal `nvidia-smi` on the host only indicates that the driver layer is normal.

For containers to use GPU, they also need:

- Mount `/dev/nvidia*`;
- Inject NVIDIA driver-related libraries;
- Set GPU visibility;
- Enable container runtime to support NVIDIA runtime;
- Work with Device Plugin to allocate devices.

These capabilities are provided by NVIDIA Container Toolkit.

### 5.2 Installing NVIDIA Container Toolkit

The actual installation commands should follow NVIDIA's official documentation.

Common workflow:

    1. Configure NVIDIA Container Toolkit software source
    2. Install nvidia-container-toolkit
    3. Use nvidia-ctk to configure containerd
    4. Restart containerd
    5. Verify if containers can access GPU

Ubuntu / Debian Approach:

    sudo apt-get update
    sudo apt-get install -y nvidia-container-toolkit

Rocky / RHEL Approach:

    sudo dnf install -y nvidia-container-toolkit

### 5.3 Configure containerd

containerd scenario recommends using:

    sudo nvidia-ctk runtime configure --runtime=containerd

Restart containerd:

    sudo systemctl restart containerd

Restart kubelet:

    sudo systemctl restart kubelet

Check configuration:

    containerd config dump | grep -i nvidia -A30 -B10

### 5.4 Verify NVIDIA Container Toolkit

Check tools:

    nvidia-container-cli info

If using Docker for testing:

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

If using Kubernetes with containerd, Docker is not required.

containerd scenario is more commonly verified through Kubernetes GPU Pods.

### 5.5 Production Notes

If using GPU Operator to manage toolkit, need to confirm:

    toolkit.enabled=true

If nodes have already been manually configured with NVIDIA Container Toolkit, can disable toolkit management during Operator installation:

    --set toolkit.enabled=false

Whether to disable depends on enterprise operations policy.

Do not allow manual configuration and Operator auto-configuration to conflict.

---

## SixI don't know.Route One: Manual Installation of NVIDIA Device Plugin

### 6.1 Prerequisites

Before using Device Plugin route, nodes should already have:

    [ ] GPU hardware functioning normally
    [ ] lspci can see NVIDIA GPU
    [ ] NVIDIA Driver installed
    [ ] nvidia-smi functioning normally
    [ ] NVIDIA Container Toolkit installed
    [ ] containerd configured with NVIDIA Runtime
    [ ] kubelet functioning normally
    [ ] Kubernetes cluster functioning normally

Device Plugin does not handle driver installation.

### 6.2 Create Namespace

Can be placed in `kube-system`, or create a separate namespace.

Recommended to create separately:

    kubectl create namespace nvidia-device-plugin

### 6.3 Install Device Plugin with Helm

Add Helm repository:

    helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
    helm repo update

Check available versions:

    helm search repo nvdp/nvidia-device-plugin --versions

Recommend specifying a version during installation, not recommend production without version.

Example:

    helm install nvidia-device-plugin nvdp/nvidia-device-plugin \
      --namespace nvidia-device-plugin \
      --version <CHART_VERSION>

Notes:

    <CHART_VERSION> should be selected based on helm search repo output.
    Production environments must fix the version.
    Do not recommend directly copying old version numbers from the internet.

### 6.4 Check DaemonSet

    kubectl get ds -n nvidia-device-plugin
    kubectl get pods -n nvidia-device-plugin -o wide

Expected:

    One nvidia-device-plugin Pod running on each GPU node.

If CPU nodes also run, need to check nodeSelector or tolerations design.

### 6.5 Check Logs

    kubectl logs -n nvidia-device-plugin -l app.kubernetes.io/name=nvidia-device-plugin

If label mismatch, first check Pod name:

    kubectl get pods -n nvidia-device-plugin

Then check specific Pod:

    kubectl logs <device-plugin-pod-name> -n nvidia-device-plugin

### 6.6 Verify Node Resources

Check node:

    kubectl describe node <gpu-node-name>

Focus on:

    Capacity:
      nvidia.com/gpu: 1

    Allocatable:
      nvidia.com/gpu: 1

If no `nvidia.com/gpu`, indicates resource registration failure.

### 6.7 Deploy GPU Test Pod

Create test file:

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

Check:

    kubectl get pod gpu-test -o wide
    kubectl logs gpu-test
    kubectl describe pod gpu-test

Expected logs can see `nvidia-smi` output.

### 6.8 Uninstalling Device Plugin

If installed via Helm:

    helm uninstall nvidia-device-plugin -n nvidia-device-plugin

Delete Namespace:

    kubectl delete namespace nvidia-device-plugin

Note:

    After uninstalling Device Plugin, the nvidia.com/gpu resource on the Node will disappear.
    Running GPU Pods may be affected.
    Before uninstalling in production environments, it is necessary to evict GPU workloads first.

---

## Seven、Route Two: Installing NVIDIA GPU Operator

### 7.1 Prerequisites for GPU Operator

GPU Operator is suitable for more comprehensive GPU node management.

Confirm before installation:

    [ ] Kubernetes cluster is normal
    [ ] GPU nodes have joined the cluster
    [ ] GPU nodes can access the image repository
    [ ] Helm is available
    [ ] Network policies allow communication between related components
    [ ] Confirmed whether Driver is managed by Operator
    [ ] Confirmed whether Toolkit is managed by Operator
    [ ] Confirmed containerd / Docker / CRI-O runtime
    [ ] Confirmed Kubernetes version compatibility with Operator version
    [ ] Confirmed whether MIG is needed
    [ ] Confirmed whether DCGM Exporter is needed

### 7.2 Adding NVIDIA Helm Repository

    helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
    helm repo update

Check versions:

    helm search repo nvidia/gpu-operator --versions

Production recommendations:

    Specify chart version in production environments.
    It is not recommended to install the latest version without specifying a version.
    Upgrade must be validated in a test environment first.

### 7.3 Creating Namespace

    kubectl create namespace gpu-operator

### 7.4 Standard Installation of GPU Operator

Basic installation example:

    helm install gpu-operator nvidia/gpu-operator \
      --namespace gpu-operator \
      --version <CHART_VERSION> \
      --wait

Notes:

    <CHART_VERSION> select based on helm search repo output.
    --wait waits for resources to be ready.
    It is not recommended to omit the version number in production environments.

### 7.5 Scenario Where Nodes Have Manually Installed Drivers

If nodes have already manually installed NVIDIA Driver, it is usually not desired for Operator to reinstall the driver.

You can use:

    helm install gpu-operator nvidia/gpu-operator \
      --namespace gpu-operator \
      --version <CHART_VERSION> \
      --set driver.enabled=false \
      --wait

Suitable for:

- Operations team maintains drivers themselves;
- Driver version has been standardized;
- Intranet environment does not want Operator to pull driver images;
- Node drivers are installed via Ansible, image templates, or manual installation.

### 7.6 Scenario Where Nodes Have Manually Configured NVIDIA Container Toolkit

If nodes have already manually configured toolkit and containerd, consider:

    helm install gpu-operator nvidia/gpu-operator \
      --namespace gpu-operator \
      --version <CHART_VERSION> \
      --set driver.enabled=false \
      --set toolkit.enabled=false \
      --wait

Note:

    Disabling toolkit requires caution.
    If toolkit is disabled but the node runtime is not properly configured, GPU Pods may not access GPUs.
    Production environments must have unified policies.

### 7.7 Viewing Resources After GPU Operator Installation

Check Namespace:

    kubectl get all -n gpu-operator

Check Pods:

    kubectl get pods -n gpu-operator -o wide

Check DaemonSet:

    kubectl get ds -n gpu-operator

Check ClusterPolicy:

    kubectl get clusterpolicy

Check detailed information:

    kubectl describe clusterpolicy

Resource names may vary slightly by version, but you will typically see similar entries:

    gpu-operator
    nvidia-driver-daemonset
    nvidia-container-toolkit-daemonset
    nvidia-device-plugin-daemonset
    nvidia-dcgm-exporter
    nvidia-operator-validator
    gpu-feature-discovery
    node-feature-discovery

### 7.8 Viewing Operator Logs

First check Pods:

    kubectl get pods -n gpu-operator

View logs:

    kubectl logs <gpu-operator-pod> -n gpu-operator

View related component logs:

    kubectl logs <nvidia-device-plugin-pod> -n gpu-operator
    kubectl logs <nvidia-container-toolkit-pod> -n gpu-operator
    kubectl logs <nvidia-dcgm-exporter-pod> -n gpu-operator
    kubectl logs <nvidia-operator-validator-pod> -n gpu-operator

Actual Pod names are determined by cluster output.

---

## EightI don't know.Common Components of GPU Operator

### 8.1 ClusterPolicy /think

ClusterPolicy is the core custom resource of GPU Operator.

It defines which components GPU Operator should manage and their configurations.

Check:

    kubectl get clusterpolicy
    kubectl describe clusterpolicy

In production, modifying GPU Operator behavior is typically done through Helm values or ClusterPolicy-related configurations rather than directly editing DaemonSets.

### 8.2 nvidia-driver-daemonset

Responsible for installing or managing NVIDIA Driver on nodes.

If set:

    driver.enabled=false

Driver installation is usually disabled.

Suitable for scenarios where nodes already have drivers installed.

### 8.3 nvidia-container-toolkit-daemonset

Responsible for configuring NVIDIA Container Toolkit to enable GPU usage by container runtimes.

If manual runtime configuration is done, consider disabling it but ensure node configurations are correct.

### 8.4 nvidia-device-plugin-daemonset

Responsible for registering GPU resources to kubelet.

This is the key component for nodes to appear as `nvidia.com/gpu`.

### 8.5 nvidia-dcgm-exporter

Responsible for exposing GPU metrics for Prometheus to scrape.

Common metrics include:

- GPU utilization;
- Memory usage;
- GPU temperature;
- GPU power consumption;
- ECC errors;
- XID errors;
- MIG metrics.

### 8.6 gpu-feature-discovery

Responsible for discovering GPU features and labeling nodes.

Examples include:

- GPU model;
- GPU count;
- MIG capabilities;
- CUDA capabilities;
- GPU product information.

These labels can be used for subsequent scheduling.

### 8.7 node-feature-discovery

Node Feature Discovery is used to discover node hardware, CPU, kernel, PCI devices, etc., and add node labels.

GPU Operator can rely on it to provide richer hardware labels for GPU nodes.

### 8.8 nvidia-operator-validator

Used to verify if components managed by GPU Operator are functioning properly.

If validator fails, it usually indicates a broken link.

---

## NineI don't know.GPU Operator Installation Verification Process

### 9.1 Verify Pod Status

    kubectl get pods -n gpu-operator -o wide

Expected:

    Relevant Pods are Running or Completed.

If some Pods are CrashLoopBackOff, check logs.

### 9.2 Verify Node GPU Resources

    kubectl describe node <gpu-node-name>

Check:

    Capacity:
      nvidia.com/gpu: <count>

    Allocatable:
      nvidia.com/gpu: <count>

### 9.3 Verify Node Labels

View labels:

    kubectl get node <gpu-node-name> --show-labels

Filter NVIDIA-related labels:

    kubectl get node <gpu-node-name> --show-labels | grep -i nvidia

Alternatively:

    kubectl describe node <gpu-node-name> | grep -i nvidia

### 9.4 Verify GPU Test Pod

Create test Pod:

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

Check:

    kubectl get pod gpu-test -o wide
    kubectl logs gpu-test
    kubectl describe pod gpu-test

### 9.5 Verify DCGM Exporter

Check Service:

    kubectl get svc -n gpu-operator

Check Pod:

    kubectl get pods -n gpu-operator | grep -i dcgm

Local port testing can be performed based on actual Service and port:

    kubectl port-forward -n gpu-operator <dcgm-exporter-pod> 9400:9400

Then access:

    curl http://127.0.0.1:9400/metrics

If Prometheus Operator is present, ServiceMonitor can be configured later.

### 9.6 Verify CUDA in Container

If the test image only executes `nvidia-smi`, it can only verify basic GPU visibility.

Further, use the devel image:

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

Deploy:

    kubectl apply -f cuda-devel-test.yaml

Check: /think

kubectl logs cuda-devel-test

Note:

    The base image may not include nvcc.
    The devel image typically includes nvcc.
    It is generally not recommended to use the devel image for production services; the runtime image should be more streamlined.

---

## 10. Device Plugin and GPU Operator Should Not Be Conflictingly Installed

### 10.1 It Is Not Recommended to Deploy Two Device Plugins Repeatedly

If you are already using GPU Operator, it usually manages its own Device Plugin.

In this case, it is not recommended to manually deploy another NVIDIA Device Plugin.

Otherwise, the following issues may occur:

- Multiple Device Plugins on the same node competing for resources;
- Confusing logs;
- Inconsistent versions;
- kubelet registration anomalies;
- Increased troubleshooting difficulty.

### 10.2 Check for Duplicate Deployments

View all NVIDIA-related DaemonSets:

    kubectl get ds -A | grep -i nvidia

View all NVIDIA-related Pods:

    kubectl get pods -A | grep -i nvidia

If you see both:

    nvidia-device-plugin in kube-system
    nvidia-device-plugin in gpu-operator

You need to confirm if there is a duplicate deployment.

### 10.3 Recommended Actions

If you are using GPU Operator:

    Uninstall the manually installed Device Plugin.

If you are using a manually installed Device Plugin:

    Do not install GPU Operator, or disable related components during installation.

Production environments should maintain a clear and consistent approach, avoiding mixed configurations.

---

## 11. GPU Node Label and Taint Design

After installing GPU components, it is recommended to plan the GPU node scheduling strategy.

### 11.1 Label GPU Nodes

Example:

    kubectl label node <gpu-node-name> node-role.kubernetes.io/gpu=true
    kubectl label node <gpu-node-name> accelerator=nvidia
    kubectl label node <gpu-node-name> gpu.vendor=nvidia

By model:

    kubectl label node <gpu-node-name> gpu.model=a100
    kubectl label node <gpu-node-name> gpu.memory=80gb

By purpose:

    kubectl label node <gpu-node-name> gpu.workload=inference

Or:

    kubectl label node <gpu-node-name> gpu.workload=training

### 11.2 Add Taints to GPU Nodes

Prevent regular Pods from scheduling to GPU nodes:

    kubectl taint node <gpu-node-name> nvidia.com/gpu=true:NoSchedule

GPU Pods need to add tolerations:

    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"

### 11.3 Complete GPU Pod Example

    apiVersion: v1
    kind: Pod
    metadata:
      name: gpu-scheduled-test
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

---

## 12. MIG Support Notes

### 12.1 What Is MIG

MIG stands for Multi-Instance GPU.

It can split supported NVIDIA data center GPUs into multiple isolated GPU instances.

Commonly used in:

- A100;
- H100;
- Some data center GPUs.

MIG is suitable for:

- Multi-tenant inference;
- Small-to-medium model services;
- Improving utilization of large GPUs;
- Isolating different workloads;
- Avoiding small tasks long-term occupying entire GPUs.

### 12.2 MIG in Device Plugin

NVIDIA Device Plugin supports MIG policy configuration.

Common policies:

    none
    single
    mixed

Meanings are roughly:

    none:
        Do not expose MIG resources.

    single:
        The node uses a unified MIG policy.

    mixed:
        Allow exposing multiple MIG instance resources.

Specific configuration details should refer to the current Device Plugin documentation.

### 12.3 MIG in GPU Operator

GPU Operator can work with MIG Manager to manage MIG.

In production, pay attention to:

- Whether the GPU model supports MIG;
- Whether the driver version supports;
- Whether the GPU Operator version supports;
- Whether the MIG partitioning policy is consistent;
- How business Pods request MIG resources;
- Whether Prometheus can collect MIG-level metrics;
- Whether fault diagnosis can locate to specific MIG instances.

### 12.4 MIG Resource Examples

After MIG is enabled, resource names may look like:

    nvidia.com/mig-1g.10gb
    nvidia.com/mig-2g.20gb
    nvidia.com/mig-3g.40gb

Specific names are determined by GPU model, MIG partitioning, and Device Plugin configuration.

Pod request example: /think

resources:
  limits:
    nvidia.com/mig-1g.10gb: 1

Note:

  Do not directly copy MIG resource names.
  First check the actual exposed resource names using kubectl describe node.

### 12.5 Viewing MIG Resources

  kubectl describe node <gpu-node-name>

Check the resources in Capacity / Allocatable:

  nvidia.com/mig-xxx

Check MIG on the node:

  nvidia-smi -L
  nvidia-smi mig -lgi
  nvidia-smi mig -lci

---

## ThirteenI don't know.GPU Sharing: Time-Slicing Introduction

### 13.1 Why GPU Sharing is Needed

By default `nvidia.com/gpu: 1` is requested as a full GPU.

For lightweight inference, development testing, and small model tasks, if each Pod exclusively uses an entire card, utilization may be very low.

In such cases, consider GPU sharing.

Common solutions:

- MIG;
- Time-Slicing;
- MPS;
- Third-party vGPU solutions.

### 13.2 Features of Time-Slicing

Time-Slicing is time-based sharing.

Features:

- Multiple Pods can share a single GPU;
- Suitable for lightweight tasks;
- Can improve utilization;
- Weaker isolation than MIG;
- Memory may still compete;
- A task failure may affect other tasks;
- Not suitable for strong isolation production scenarios.

### 13.3 Production Recommendations

Strong isolation in production:

  Prioritize MIG.

Development testing or lightweight inference:

  Can evaluate Time-Slicing.

Serious training or core inference:

  Be cautious about GPU sharing.

Do not mix all workloads onto a single card just to improve utilization.

---

## FourteenI don't know.Domestic Network and Private Image Registry Notes

### 14.1 Common Issues

When installing Device Plugin or GPU Operator, you may need to pull images.

Common issues in domestic network environments:

- Slow image pulling;
- Image pulling failure;
- NGC access instability;
- GitHub access instability;
- Helm repo access instability;
- Nodes unable to access public internet;
- Corporate security policies prohibit public internet pulling.

### 14.2 Production Recommendations

Recommendations:

  1. List all required images in advance
  2. Pull images in an internet-connected environment
  3. Retag to internal image registry
  4. Push to Harbor / Nexus / Alibaba Cloud image registry
  5. Modify Helm values to use internal registry
  6. Fix image tag
  7. Keep image version list

### 14.3 Viewing GPU Operator Images

You can view via Helm values:

  helm show values nvidia/gpu-operator --version <CHART_VERSION> > gpu-operator-values.yaml

Search for image:

  grep -i "repository" gpu-operator-values.yaml
  grep -i "image" gpu-operator-values.yaml
  grep -i "tag" gpu-operator-values.yaml

Alternatively, check after installing on a test cluster:

  kubectl get pods -n gpu-operator -o yaml | grep image:

### 14.4 Modifying values.yaml

Recommend using values.yaml to manage configurations in production, rather than stacking many `--set` in command line.

Example:

  helm show values nvidia/gpu-operator --version <CHART_VERSION> > values-gpu-operator.yaml

Install after editing:

  helm install gpu-operator nvidia/gpu-operator \
    --namespace gpu-operator \
    --version <CHART_VERSION> \
    -f values-gpu-operator.yaml \
    --wait

Upgrade:

  helm upgrade gpu-operator nvidia/gpu-operator \
    --namespace gpu-operator \
    --version <CHART_VERSION> \
    -f values-gpu-operator.yaml \
    --wait

### 14.5 Not Recommended to Depend on Public Internet

Production clusters should not strongly depend on public internet images.

Reasons:

- Node expansion may fail;
- Risk of image tag drift;
- Difficulty in security audits;
- Difficulty in rollback;
- Uncontrolled recovery during failures.

---

## FifteenI don't know.GPU Operator Upgrade and Rollback

### 15.1 Pre-Upgrade Checks

Save current information before upgrading:

  helm list -n gpu-operator
  helm get values gpu-operator -n gpu-operator > old-values.yaml
  helm get manifest gpu-operator -n gpu-operator > old-manifest.yaml
  kubectl get pods -n gpu-operator -o wide > old-pods.txt
  kubectl get ds -n gpu-operator -o wide > old-ds.txt
  kubectl get clusterpolicy -o yaml > old-clusterpolicy.yaml

Check current node GPU:

  kubectl describe node <gpu-node-name> > old-node-describe.txt

### 15.2 Upgrade Process

Recommendations:

  1. Validate in test cluster
  2. Validate on single GPU node
  3. Schedule during production maintenance window
  4. Cordon GPU node
  5. Drain business Pods
  6. Upgrade GPU Operator
  7. Validate Operator Pod
  8. Validate nvidia.com/gpu
  9. Validate GPU Pod
  10. Resume scheduling

Node maintenance:

kubectl cordon <gpu-node-name>  
kubectl drain <gpu-node-name> --ignore-daemonsets --delete-emptydir-data  

**Recovery:**  

kubectl uncordon <gpu-node-name>  

### 15.3 Helm Upgrade  

helm upgrade gpu-operator nvidia/gpu-operator \  
  --namespace gpu-operator \  
  --version <NEW_CHART_VERSION> \  
  -f values-gpu-operator.yaml \  
  --wait  

### 15.4 Rollback  

**Check history:**  

helm history gpu-operator -n gpu-operator  

**Rollback:**  

helm rollback gpu-operator <REVISION> -n gpu-operator  

**Verify after rollback:**  

kubectl get pods -n gpu-operator -o wide  
kubectl describe node <gpu-node-name>  
kubectl logs gpu-test  

**Production recommendations:**  

Do not upgrade GPU Operator without testing and rollback plans.  
GPU component upgrades may affect running GPU workloads.  

---  

## SixteenI don't know.Uninstall GPU Operator  

### 16.1 Pre-uninstall checks  

**Confirm before uninstalling:**  

[ ] GPU workloads have been migrated or stopped  
[ ] Nodes have been cordoned  
[ ] Key Pods have been drained  
[ ] Helm values have been backed up  
[ ] ClusterPolicy has been backed up  
[ ] Confirmed whether driver and runtime will be affected  

### 16.2 Uninstall commands  

helm uninstall gpu-operator -n gpu-operator  

**Check for residual resources:**  

kubectl get all -n gpu-operator  
kubectl get clusterpolicy  

**Delete Namespace:**  

kubectl delete namespace gpu-operator  

### 16.3 Post-uninstall impacts  

**Potential impacts:**  

- `nvidia.com/gpu` disappears from Node resources;  
- GPU Pods cannot be scheduled further;  
- DCGM Exporter disappears;  
- GPU monitoring is interrupted;  
- Automatic node labels disappear;  
- MIG management becomes ineffective;  
- If driver is managed by Operator, driver status needs to be confirmed separately.  

**Check on nodes after uninstall:**  

nvidia-smi  
lsmod | grep nvidia  
ls -l /dev/nvidia*  
containerd config dump | grep -i nvidia -A20 -B5  

---  

## SeventeenI don't know.Common Issue One: Node lacks nvidia.com/gpu  

### 17.1 Symptoms  

Execute:  

kubectl describe node <gpu-node-name>  

Cannot see:  

nvidia.com/gpu  

### 17.2 Possible causes  

**Possible causes:**  

- Node has no GPU;  
- lspci cannot see GPU;  
- NVIDIA Driver is abnormal;  
- nvidia-smi is not functioning properly;  
- Device Plugin is not running;  
- Device Plugin runs on wrong node;  
- Device Plugin logs show errors;  
- GPU Operator components are not ready;  
- kubelet plugin registration is abnormal;  
- Node taint or selector prevents plugin scheduling;  
- Runtime configuration is abnormal;  
- GPU is marked as abnormal by health checks.  

### 17.3 Troubleshooting commands  

**On node:**  

lspci | grep -i nvidia  
nvidia-smi  
lsmod | grep nvidia  
ls -l /dev/nvidia*  
dmesg | grep -i xid  

**In cluster:**  

kubectl get pods -A | grep -i nvidia  
kubectl get ds -A | grep -i nvidia  
kubectl describe node <gpu-node-name>  
kubectl get events -A --sort-by=.lastTimestamp  

**Check Device Plugin logs:**  

kubectl logs <device-plugin-pod> -n <namespace>  

### 17.4 Troubleshooting steps  

In order:  

1. Confirm hardware layer lspci  
2. Confirm driver layer nvidia-smi  
3. Confirm Device Plugin Pod is running  
4. Confirm Device Plugin logs have no errors  
5. Confirm kubelet is functioning  
6. Restart Device Plugin Pod  
7. Restart kubelet if necessary  
8. Recheck Node Capacity / Allocatable  

---  

## EighteenI don't know.Common Issue Two: Device Plugin Pod CrashLoopBackOff  

### 18.1 Symptoms  

kubectl get pods -A | grep -i nvidia  

See Device Plugin Pod:  

CrashLoopBackOff  
Error  

### 18.2 Troubleshooting  

**Check logs:**  

kubectl logs <device-plugin-pod> -n <namespace>  

**Check description:**  

kubectl describe pod <device-plugin-pod> -n <namespace>  

**Check node:**  

kubectl get pod <device-plugin-pod> -n <namespace> -o wide  

**Check on corresponding node:**  

nvidia-smi  
ls -l /dev/nvidia*  
systemctl status kubelet  
systemctl status containerd  

### 18.3 Possible causes  

**Possible causes:**

- Node driver failure;
- `/dev/nvidia*` does not exist;
- Permission error;
- Container runtime cannot mount device;
- Image pull failed;
- Device Plugin version is incompatible with node environment;
- Unsupported MIG policy is configured;
- Node has no GPU but plugin is forced to run;
- SELinux / AppArmor / PodSecurity restrictions;
- Runtime configuration error.

### 18.4 Resolution

Resolution methods:

- Fix NVIDIA Driver;
- Fix NVIDIA Container Toolkit;
- Check image;
- Check Helm values;
- Check MIG configuration;
- Check if node should run plugin;
- Check Pod security policy;
- Delete Pod if necessary to let DaemonSet rebuild:

    kubectl delete pod <device-plugin-pod> -n <namespace>

---

## Nineteen, Common Fault Three: GPU Pod Pending

### 19.1 Phenomenon

GPU Pod remains in Pending state.

Check:

    kubectl get pod <pod-name> -n <namespace> -o wide

### 19.2 Troubleshooting Events

    kubectl describe pod <pod-name> -n <namespace>

Common Events:

    insufficient nvidia.com/gpu
    node(s) had untolerated taint
    node(s) didn't match Pod's node affinity/selector
    exceeded quota
    insufficient cpu
    insufficient memory

### 19.3 Resolution Approach

If:

    insufficient nvidia.com/gpu

Check:

    kubectl describe node <gpu-node-name>
    kubectl get pods -A -o wide | grep <gpu-node-name>

If:

    untolerated taint

Add toleration to Pod.

If:

    node selector mismatch

Check:

    kubectl get node <gpu-node-name> --show-labels

If:

    exceeded quota

Check:

    kubectl describe resourcequota -n <namespace>

If:

    insufficient cpu / memory

Indicates GPU is not the issue, but CPU / Memory request is too large or node resources are insufficient.

---

## Twenty, Common Fault Four: GPU Pod Running but No GPU Visible Inside Container

### 20.1 Phenomenon

Pod is already Running.

Enter container to execute:

    nvidia-smi

Failed.

### 20.2 Possible Causes

Possible causes:

- Pod did not request `nvidia.com/gpu`;
- NVIDIA Container Toolkit is not configured;
- containerd runtime configuration is abnormal;
- Device Plugin allocation is abnormal;
- Image does not have nvidia-smi;
- CUDA Runtime inside container is incompatible with host driver;
- RuntimeClass configuration is wrong;
- `/dev/nvidia*` is not mounted;
- `CUDA_VISIBLE_DEVICES` is overridden.

### 20.3 Troubleshooting

Check Pod YAML:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A15 resources

Check devices inside container:

    kubectl exec -it <pod-name> -n <namespace> -- sh
    ls -l /dev/nvidia*
    echo $CUDA_VISIBLE_DEVICES

Check node runtime:

    containerd config dump | grep -i nvidia -A30 -B10

Check toolkit:

    nvidia-container-cli info

### 20.4 Resolution

Resolution methods:

- Confirm Pod requests GPU;
- Fix NVIDIA Container Toolkit;
- Fix containerd configuration;
- Use official CUDA test image for verification;
- Check Device Plugin logs;
- Check RuntimeClass;
- Check driver compatibility with CUDA Runtime.

---

## Twenty-one, Common Fault Five: GPU Operator Pod Image Pull Failed

### 21.1 Phenomenon

Pod status:

    ImagePullBackOff
    ErrImagePull

### 21.2 Troubleshooting

Check Pod:

    kubectl describe pod <pod-name> -n gpu-operator

Check events:

    kubectl get events -n gpu-operator --sort-by=.lastTimestamp

Check image:

    kubectl get pod <pod-name> -n gpu-operator -o yaml | grep image:

### 21.3 Possible Causes

Possible causes:

- Node cannot access external network;
- Cannot access NGC;
- DNS issue;
- Proxy issue;
- Image tag does not exist;
- Private registry authentication failed;
- imagePullSecret not configured;
- Domestic network instability.

### 21.4 Recommendations

- Synchronize image to internal repository;
- Modify repository in Helm values;
- Configure imagePullSecrets;
- Fix image tag;
- Ensure all nodes can access internal repository;
- Test pull with crictl / nerdctl before installation.

containerd pull test:

    crictl pull <image>

or:

    nerdctl -n k8s.io pull <image>

---

## Twenty-two, Common Fault Six: Operator Validator Failed

### 22.1 Phenomenon

nvidia-operator-validator Pod failed or CrashLoopBackOff.

### 22.2 Troubleshooting

Check Pod:

    kubectl get pods -n gpu-operator | grep validator

View Logs:

    kubectl logs <validator-pod> -n gpu-operator

Check Related Components:

    kubectl get pods -n gpu-operator -o wide
    kubectl get ds -n gpu-operator

### 22.3 Possible Causes

Possible Causes:

- Driver not installed successfully;
- Toolkit configuration failure;
- Device Plugin not registered GPU;
- Runtime configuration anomaly;
- DCGM component anomaly;
- Node has no GPU;
- Image pull failure;
- MIG configuration anomaly.

### 22.4 Resolution

Locate the specific failure item based on the validator logs.

Do not restart validator solely.

Should check the components it validates:

    Driver
    Toolkit
    Device Plugin
    DCGM
    Runtime
    Node Feature Discovery
    GPU Feature Discovery

---

## Twenty-Three, Common Issue Seven: GPU Operator After Installation Overwrites Node Driver

### 23.1 Phenomenon

The node originally manually installed a stable NVIDIA Driver.

After installing GPU Operator, the driver version changed or the driver Pod attempts to manage the driver.

### 23.2 Cause

Not setting:

    driver.enabled=false

During installation, the Operator may default to trying to manage the driver.

### 23.3 Resolution

If you want to retain the existing driver on the node, use:

    --set driver.enabled=false

Or set in values.yaml:

    driver:
      enabled: false

If already installed, confirm the current status and redeploy Operator if necessary.

### 23.4 Production Recommendation

In production, must clearly define:

    Who manages the driver?

Optional answers:

    1. Managed manually by operations or Ansible
    2. Managed by GPU Operator
    3. Pre-installed by cloud vendor node image
    4. Pre-installed by bare metal image template

Do not have multiple parties managing simultaneously.

---

## Twenty-Four, Common Issue Eight: containerd Configuration Still Cannot Use GPU

### 24.1 Phenomenon

Executed:

    nvidia-ctk runtime configure --runtime=containerd

But the Pod still cannot use GPU internally.

### 24.2 Troubleshooting

Check containerd configuration:

    containerd config dump | grep -i nvidia -A30 -B10

Check if containerd has restarted:

    systemctl status containerd

Check if kubelet has restarted:

    systemctl status kubelet

Check toolkit:

    nvidia-container-cli info

Check Pod:

    kubectl describe pod <gpu-pod> -n <namespace>

### 24.3 Possible Causes

Possible Causes:

- containerd not restarted;
- kubelet not restarted;
- containerd configuration file path is not the actual effective path;
- RuntimeClass not matched;
- Device Plugin not running normally;
- Pod not requested GPU;
- Multiple containerd configurations on the node;
- Conflict between GPU Operator and manual configuration.

### 24.4 Resolution

Resolution steps:

    sudo systemctl restart containerd
    sudo systemctl restart kubelet

Recreate GPU Pod:

    kubectl delete pod <gpu-pod> -n <namespace>
    kubectl apply -f gpu-test.yaml

Check kubelet's CRI endpoint if necessary:

    ps -ef | grep kubelet
    crictl info

---

## Twenty-Five, GPU Monitoring Integration Guide

GPU Operator typically deploys DCGM Exporter by default.

### 25.1 DCGM Exporter Function

DCGM Exporter is used to expose NVIDIA GPU metrics as Prometheus format.

Common metrics include:

- GPU utilization;
- GPU memory usage;
- GPU temperature;
- GPU power consumption;
- SM utilization;
- Memory bandwidth;
- ECC errors;
- XID errors;
- MIG instance metrics.

### 25.2 Verify Metrics

Check Pod:

    kubectl get pods -n gpu-operator | grep dcgm

Port forwarding:

    kubectl port-forward -n gpu-operator <dcgm-exporter-pod> 9400:9400

Access:

    curl http://127.0.0.1:9400/metrics

### 25.3 Prometheus Integration

If using Prometheus Operator, configure ServiceMonitor.

For regular Prometheus, configure scrape_configs.

Example:

    scrape_configs:
      - job_name: 'dcgm-exporter'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace]
            action: keep
            regex: gpu-operator
          - source_labels: [__meta_kubernetes_pod_label_app]
            action: keep
            regex: nvidia-dcgm-exporter

Specific labels need to be adjusted according to actual Pod labels.

### 25.4 Production Recommendation

GPU clusters must integrate GPU metrics.

Otherwise, only know:

    Whether Pod is Running

But don't know:

    Whether GPU is actually used
    Whether memory is long-term occupied
    Whether GPU is overheating
    Whether XID errors exist
    Whether GPU is idle
    Whether expansion or recycling is needed

---

## Twenty-Six, Production Installation Recommendations

### 26.1 Version Fixing

Production environment must fix:

- GPU Operator chart version  
- Device Plugin version  
- NVIDIA Driver version  
- NVIDIA Container Toolkit version  
- DCGM Exporter version  
- CUDA Runtime base image version  
- containerd version  
- Kubernetes version  

Do not use:  
```  
latest  
```  

Do not install without version:  
```  
helm install gpu-operator nvidia/gpu-operator  
```  

Recommended:  
```  
helm install gpu-operator nvidia/gpu-operator \  
  --version <CHART_VERSION> \  
  -f values-gpu-operator.yaml  
```  

### 26.2 Test Before Deployment  

Process:  
1. Single-node testing  
2. Cluster validation testing  
3. Business image validation  
4. Monitoring validation  
5. Alert validation  
6. Rollback validation  
7. Production phased rollout  

### 26.3 Do Not Test Directly in Production Clusters  

GPU nodes are costly, and business workloads are typically critical.  

Before production operations:  
```  
kubectl cordon <gpu-node-name>  
kubectl drain <gpu-node-name> --ignore-daemonsets --delete-emptydir-data  
```  

After maintenance:  
```  
kubectl uncordon <gpu-node-name>  
```  

### 26.4 Internal Repository  

Recommendations:  
- Cache Helm charts in an internal repository  
- Synchronize images to an internal image registry  
- Pull nodes only from the internal registry  
- Record image inventory  
- Perform security scans on images  
- Retain rollback versions  

### 26.5 Node Baseline  

GPU nodes should record:  
- OS Version  
- Kernel Version  
- NVIDIA Driver Version  
- CUDA Compatibility  
- Container Runtime Version  
- NVIDIA Container Toolkit Version  
- Device Plugin Version  
- GPU Operator Version  
- DCGM Exporter Version  
- GPU Model  
- GPU Count  
- MIG Status  
- Node Labels  
- Node Taints  

---

## Twenty-Seven, Complete Installation Checklist  

### 27.1 Pre-Installation  

```  
[ ] Kubernetes cluster is normal  
[ ] GPU node is Ready  
[ ] lspci can see NVIDIA GPU  
[ ] nvidia-smi is normal  
[ ] containerd is normal  
[ ] kubelet is normal  
[ ] Helm is available  
[ ] Determined whether to use Device Plugin or GPU Operator  
[ ] Determined whether the Operator manages the Driver  
[ ] Determined whether the Operator manages the Toolkit  
[ ] Confirmed image pullability  
[ ] Confirmed version numbers  
[ ] Prepared values.yaml  
[ ] Prepared rollback plan  
```  

### 27.2 During Installation  

```  
[ ] Namespace creation succeeded  
[ ] Helm repo is accessible  
[ ] Helm install succeeded  
[ ] Pod pulls images normally  
[ ] DaemonSet runs normally  
[ ] Operator has no error logs  
[ ] Device Plugin runs normally  
[ ] Validator passed  
```  

### 27.3 Post-Installation  

```  
[ ] Node Capacity shows nvidia.com/gpu  
[ ] Node Allocatable shows nvidia.com/gpu  
[ ] GPU test Pod can run  
[ ] GPU test Pod logs show nvidia-smi  
[ ] /dev/nvidia* exists in the container  
[ ] DCGM Exporter is normal  
[ ] Prometheus can collect GPU metrics  
[ ] Grafana can display GPU metrics  
[ ] GPU node Labels are correct  
[ ] GPU node Taints are correct  
[ ] ResourceQuota is planned  
[ ] Business image testing passed  
```  

---

## Twenty-Eight, Common Command Summary  

### 28.1 View GPU Nodes  

```  
kubectl get nodes -o wide  
kubectl describe node <gpu-node-name>  
kubectl get node <gpu-node-name> --show-labels  
```  

### 28.2 View NVIDIA Components  

```  
kubectl get pods -A | grep -i nvidia  
kubectl get ds -A | grep -i nvidia  
kubectl get pods -n gpu-operator -o wide  
kubectl get ds -n gpu-operator  
```  

### 28.3 View GPU Operator  

```  
helm list -n gpu-operator  
helm get values gpu-operator -n gpu-operator  
kubectl get clusterpolicy  
kubectl describe clusterpolicy  
```  

### 28.4 View Device Plugin  

```  
kubectl get pods -A | grep -i device-plugin  
kubectl logs <device-plugin-pod> -n <namespace>  
```  

### 28.5 View GPU Resources  

```  
kubectl describe node <gpu-node-name> | grep -A10 -B5 "nvidia.com/gpu"  
```  

### 28.6 Test GPU Pod

kubectl apply -f gpu-test.yaml
kubectl get pod gpu-test -o wide
kubectl logs gpu-test
kubectl describe pod gpu-test

### 28.7 Node Local Checks

    lspci | grep -i nvidia
    nvidia-smi
    nvidia-smi -L
    nvidia-smi topo -m
    lsmod | grep nvidia
    ls -l /dev/nvidia*
    dmesg | grep -i xid
    nvidia-container-cli info
    containerd config dump | grep -i nvidia -A30 -B10

---

## Twenty-Nine, Device Plugin and Operator Troubleshooting Layers

| Phenomenon | Priority Troubleshooting Layer | Common Causes |
|---|---|---|
| `lspci` Can't see GPU | Hardware / BIOS | GPU not properly inserted, power supply, PCIe, Above 4G Decoding |
| `nvidia-smi` Failure | Driver Layer | Driver, nouveau, Secure Boot, DKMS |
| Node has no `nvidia.com/gpu` | Device Plugin / kubelet | Plugin not running, registration failure, GPU health anomaly |
| Device Plugin CrashLoop | Plugin / Node Environment | Driver anomaly, missing device files, image issues |
| GPU Operator Pod pull failure | Image / Network | NGC unreachable, private registry authentication, DNS |
| Validator failure | Operator Component | Driver, Toolkit, Plugin, DCGM any link anomaly |
| GPU Pod Pending | Scheduler | GPU insufficient, Taint, Label, Quota, CPU/Memory |
| Pod Running but no GPU | Runtime / Container | Toolkit, containerd, image, Pod not requesting GPU |
| DCGM no metrics | Monitoring Chain | exporter anomaly, ServiceMonitor, Prometheus configuration |
| MIG resources not appearing | MIG Configuration | GPU not supported, policy error, Operator not configured |

---

## Thirty, Production Environment Recommended Deployment Path

### 30.1 Learning and Experimentation Route

Suitable for personal learning:

    1. Manually install NVIDIA Driver
    2. Manually install NVIDIA Container Toolkit
    3. Configure containerd
    4. Helm install Device Plugin
    5. Verify nvidia.com/gpu
    6. Deploy gpu-test Pod
    7. Further learn GPU Operator

Advantages:

    Clear chain
    Easier to understand each layer's role
    Stronger troubleshooting capability

### 30.2 Small-Scale Production Route

Suitable for small number of GPU nodes:

    1. Driver installed by operations team in standardized manner
    2. Container Toolkit installed by operations team in standardized manner
    3. Device Plugin deployed with fixed version via Helm
    4. DCGM Exporter deployed separately
    5. Prometheus integrated with metrics
    6. Manually maintain node labels and taints

Advantages:

    Controllable
    Simple
    Lower change risk

Disadvantages:

    Maintenance cost increases with more components

### 30.3 Medium-to-Large Production Route

Suitable for formal GPU platform:

    1. Use GPU Operator
    2. Clearly define whether Driver is managed by Operator
    3. Clearly define whether Toolkit is managed by Operator
    4. Fix Helm chart and image version
    5. Use internal image repository
    6. Use values.yaml to manage configuration
    7. Integrate DCGM Exporter
    8. Integrate Prometheus / Grafana / AlertManager
    9. Configure GPU node Label / Taint
    10. Plan ResourceQuota
    11. Establish upgrade and rollback process

Advantages:

    High standardization
    Suitable for scaling
    Easy to manage uniformly
    Easy to integrate MIG and monitoring

Disadvantages:

    High initial understanding cost
    Values configuration needs careful handling
    Operator permissions and behavior must be fully understood

---

## Thirty-One, Summary

NVIDIA Device Plugin and NVIDIA GPU Operator are two important layers in Kubernetes GPU platform.

Device Plugin solves:

    How to let Kubernetes see GPU
    How to make Node appear nvidia.com/gpu
    How to let Pod request GPU

GPU Operator solves:

    How to automate management of complete NVIDIA GPU software stack
    How to unify driver, runtime, Device Plugin, DCGM, node labels and MIG components

The relationship can be understood as:

    Device Plugin is one of the core components for GPU scheduling.
    GPU Operator is a more complete GPU node software stack management solution.
    GPU Operator usually includes and manages Device Plugin.

When learning, it's recommended to first understand Device Plugin:

    NVIDIA Driver
      ↓
    NVIDIA Container Toolkit
      ↓
    Device Plugin
      ↓
    kubelet
      ↓
    nvidia.com/gpu
      ↓
    GPU Pod

When in production, it's recommended to evaluate GPU Operator:

    GPU Operator
      ↓
    Driver / Toolkit / Device Plugin / DCGM / GFD / MIG
      ↓
    Unified GPU node management
      ↓
    Monitoring and scheduling closed-loop

When troubleshooting, must troubleshoot by layer:

    lspci not normal:
        Check hardware and BIOS

    nvidia-smi not normal:
        Check driver

    Node has no nvidia.com/gpu:
        Check Device Plugin or GPU Operator

Pod Pending:
    Check Scheduler, Label, Taint, Quota, Resource Insufficient

Pod Running but no GPU:
    Check Container Toolkit, containerd, Image and Pod Resource Claims

Metrics not present:
    Check DCGM Exporter, ServiceMonitor, Prometheus

It is not recommended to integrate GPU into Kubernetes in production environments via a single `kubectl apply` command.

A real GPU platform deployment requires simultaneous attention to:

- Driver version;
- CUDA compatibility;
- containerd configuration;
- Device Plugin;
- GPU Operator;
- Image repository;
- Node labels;
- Node taints;
- Resource quotas;
- MIG;
- GPU monitoring;
- Alerts;
- Upgrades;
- Rollbacks;
- Cost and utilization governance.

Only by connecting these layers together can GPU nodes evolve from "the host can see the GPU" to a production-grade GPU resource pool that is "Kubernetes-schedulable, business usable, monitorable, and fault diagnosable".

---

## Reference Documents

- NVIDIA Kubernetes Device Plugin:
  https://github.com/NVIDIA/k8s-device-plugin

- NVIDIA GPU Operator Documentation:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/

- NVIDIA GPU Operator Getting Started:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html

- NVIDIA Container Toolkit:
  https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

- NVIDIA DCGM Exporter:
  https://github.com/NVIDIA/dcgm-exporter

- Kubernetes Device Plugins:
  https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/

- Kubernetes GPU Scheduling:
  https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/

- Kubernetes Taints and Tolerations:
  https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

- Kubernetes Resource Quotas:
  https://kubernetes.io/docs/concepts/policy/resource-quotas/

- Helm Documentation:
  https://helm.sh/docs/