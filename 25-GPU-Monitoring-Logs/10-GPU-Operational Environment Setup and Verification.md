# 10-GPU-Operational Environment Setup and Verification

## Document Description

This document aims to outline a practical, verifiable, and scalable Kubernetes operational environment for managing GPUs.

Rather than focusing on individual components, this document integrates previous sections on GPU basics, drivers, CUDA, Device Plugins, GPU Pods, monitoring and alerts, and troubleshooting into a comprehensive experimental workflow.

This document addresses the following key questions:

- How to plan a Kubernetes operational environment for managing GPUs;
- What system-level checks are required before adding GPU nodes to K8S;
- How to ensure that NVIDIA Drivers, CUDA, containerd, and the NVIDIA Container Toolkit are functioning correctly;
- How to choose between using Device Plugins or GPU Operators;
- How to verify whether Kubernetes recognizes `nvidia.com/gpu`;
- How to deploy GPU test Pods;
- How to confirm the capabilities of GPUs, CUDA, and PyTorch within containers;
- How to integrate with DCGM Exporter, Prometheus, and Grafana;
- How to design effective GPU alerts;
- How to conduct fault tolerance tests for scenarios such as GPU Pod Pending, CUDA Out Of Memory, XID errors, and high temperatures;
- How to establish an acceptance checklist for GPU nodes before deployment;
- How to evolve the experimental environment into a production-grade operational baseline for managing GPUs.

This document is recommended for reading after completing the following chapters:

- 01-GPU-Basic Concepts and Hardware Composition
- 02-GPU-BIOS and Hardware Tuning
- 03-NVIDIA-Driver Installation and Verification
- 04-CUDA-Installation and Testing
- 05-K8S-GPU-Resource Concepts and Scheduling Principles
- 06-NVIDIA-Device-Plugin-And-Operator-Installation
- 07-GPU-Pod-Deployment and Scheduling Practices
- 08-GPU-Monitoring and Alert Integration
- 09-GPU-Troubleshooting Cases and Practices

---

## Tags

#Kubernetes #GPU #NVIDIA #CUDA #DevicePlugin #GPUOperator #DCGM #Prometheus #Grafana #AlertManager #SRE #ExperimentalEnvironment #OperationalPractices

---

## Recommended Reading Path

Recommended reading path:

    06-GPU and AI Infrastructure/04-GPU Comprehensive Experiments/10-GPU-Operational Environment Setup and Verification.md

---

## I. Why a Dedicated GPU Operational Environment is Needed

Managing GPUs involves more than just running `nvidia-smi` or configuring resource limits like:

    resources:
      limits:
        nvidia.com/gpu: 1

True operational expertise requires establishing a complete chain of capabilities that includes:

    GPU Hardware
      ↓
    BIOS / PCIe
      ↓
    Linux Kernel
      ↓
    NVIDIA Driver
      ↓
    CUDA Runtime / Toolkit
      ↓
    NVIDIA Container Toolkit
      ↓
    containerd
      ↓
    Kubernetes kubelet
      ↓
    NVIDIA Device Plugin / GPU Operator
      ↓
    Node Registration with `nvidia.com/gpu`
      ↓
    GPU Pod Scheduling
      ↓
    Execution of `nvidia-smi` / CUDA / PyTorch within containers
      ↓
    Integration with DCGM Exporter
      ↓
    Prometheus and Grafana for data collection
      ↓
    AlertManager for notifications
    ↑
    Comprehensive troubleshooting and capacity management

Without a dedicated experimental environment, it is difficult to truly understand issues such as:

- Why the host machine's `nvidia-smi` works fine, but Kubernetes cannot detect the GPU;
- Why a Node has `nvidia.com/gpu` listed, but Pods remain Pending;
- Why a Pod seems to be Running, but `nvidia-smi` inside the container fails;
- Why `nvidia-smi` is working within the container, but PyTorch cannot recognize the GPU;
- Why low GPU utilization might not necessarily indicate an issue;
- Why high video memory usage is not always a fault;
- Why analyzing XID errors requires considering `dmesg`, temperature, power consumption, and business logs;
- Why GPU monitoring should go beyond just checking `GPU-Util`;
- Why labeling, tainting, quotas, priority classes, image versions, and alert rules are essential in a production environment.

Therefore, the goal of a dedicated GPU operational environment is not just to run a single test Pod, but to establish a comprehensive capability that covers node deployment, resource registration, workload execution, monitoring and alerts, fault tolerance testing, and post-mortem analysis.

---

## II. Experimental Objectives

Upon completion of this experimental environment setup, the following capabilities should be achieved:

### 2.1 Node-Level Capabilities

    [ ] The GPU node is correctly recognized by Linux.
    [ ] NVIDIA Driver installation is complete.
    [ ] `nvidia-smi` outputs are normal.
    [ ] Basic CUDA tests pass successfully.
    [### 4.1 IP Address Planning

It is recommended to use dedicated GPU nodes to avoid confusion with regular Worker nodes.

    ops-server        10.0.0.10
    k8s-master        10.0.0.20
    k8s-worker01      10.0.0.21
    k8s-worker02      10.0.0.22
    k8s-gpu-node01    10.0.0.30

If there are multiple GPU nodes:

    k8s-gpu-node01    10.0.0.30
    k8s-gpu-node02    10.0.0.31
    k8s-gpu-node03    10.0.0.32

### 4.2 Hostname Planning

    hostnamectl set-hostname k8s-gpu-node01

Add the following entries to `/etc/hosts` on all nodes:

    10.0.0.20 k8s-master
    10.0.0.21 k8s-worker01
    10.0.0.22 k8s-worker02
    10.0.0.30 k8s-gpu-node01

### 4.3 System Version Recommendations

It is recommended to use:

    Ubuntu Server 22.04.5 LTS

Or you can also use:

    Rocky Linux 9

This document focuses on Ubuntu Server 22.04; for Rocky Linux 9, refer to the corresponding installation instructions.

### 4.4 Kubernetes Version Recommendations

For the experimental environment, it is recommended to use the same version as your existing cluster.

To check the version:

    kubectl version
    kubelet --version

It is advised to record the following versions:

    Kubernetes Version:
    containerd Version:
    CNI:
    Kernel Version:
    NVIDIA Driver Version:
    CUDA Version:
    Device Plugin / GPU Operator Version:

---

## V. Experimental Route Selection

There are two common approaches to integrating GPUs into Kubernetes.

### 5.1 Route One: Manual Drivers + NVIDIA Container Toolkit + Device Plugin

Suitable for:

- Learning;
- Small-scale experiments;
- Those who want to understand each component;
- Those who have already manually installed the drivers;
- Those who do not want the Operator to manage the drivers;
- Those who only need basic GPU scheduling.

Process:

    Manually install NVIDIA Driver
      ↓
    Manually install NVIDIA Container Toolkit
      ↓
    Configure containerd
      ↓
    Install NVIDIA Device Plugin using Helm
      ↓
    Verify functionality on nvidia.com/gpu
      ↓
    Deploy GPU Pods
      ↓
    Connect to DCGM Exporter

Advantages:

    Clear hierarchy;
    Easy to understand the principles;
    Suitable for learning and troubleshooting.

Disadvantages:

    Higher maintenance costs for multiple nodes;
    Need to manage the Driver, Toolkit, and DCGM yourself.

### 5.2 Route Two: NVIDIA GPU Operator

Suitable for:

- Production environments;
- Multiple GPU nodes;
- Those who want centralized management of NVIDIA components;
- Those who need DCGM Exporter;
    Those who require GPU Feature Discovery, MIG, or more comprehensive lifecycle management.

Process:

    Install the GPU Operator using Helm
      ↓
    Let the Operator manage the Driver, Toolkit, Device Plugin, and DCGM
      ↓
    Register nodes with nvidia.com/gpu
      ↓
    Deploy GPU Pods
      ↓
    Use Prometheus to collect DCGM metrics

Advantages:

    Higher level of standardization;
    Centralized management of components;
    More suitable for production use.

Disadvantages:

    Higher initial learning curve;
    Careful configuration of values is required;
    Need to decide whether the Operator should manage the drivers and runtime.

### 5.3 Recommended Route for This Experiment

During the learning phase, it is recommended to use:

    Route One: Manual Drivers + Toolkit + Device Plugin

Reasons:

    It helps in clearly understanding each component;
    Makes it easier to troubleshoot issues;
    Prevents all problems from being masked by the Operator.

After completing basic experiments, you can then move on to:

    Route Two: NVIDIA GPU Operator

This approach aligns better with the typical learning path for operations and maintenance.

---

## VI. Pre-checks Before Adding a GPU Node to the Cluster

Before integrating a GPU node into Kubernetes, it is essential to verify the hardware, drivers, and runtime components.

### 6.1 Check System Version

    cat /etc/os-release
    uname -a
    hostnamectl

### 6.2 Verify if the GPU is Recognized by the System

   ### 13.1 Creating a Minimum GPU Test Pod

Create a YAML file for the test pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
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

Deploy the pod:

```bash
kubectl apply -f gpu-test.yaml
```

Check the deployment status and logs:

```bash
kubectl get pod gpu-test -o wide
kubectl logs gpu-test
kubectl describe pod gpu-test
```

### 13.2 Verifying Results

If successful, the logs should display information about the NVIDIA SMI driver, CUDA version, GPU name, and memory usage.

This confirms that Kubernetes can schedule GPU pods, the container can recognize the GPU, the Device Plugin is functioning correctly, the NVIDIA Container Toolkit is properly configured, and the runtime has successfully mounted the GPU device.

### 13.3 Common Failure Scenarios

- **Pod Pending**: Check the Events for reasons such as insufficient `nvidia.com/gpu` resources or unmatched tolerations.
- **Pod Running but Logs Failed**: Investigate issues with container startup, image, runtime configuration, or device mounting.### 13.1 Why Do We Need the devel Image?

`base` or `runtime` images may not include `nvcc`.

If you need to verify the CUDA Toolkit compilation tools, you should use the `devel` image.

### 13.2 Creating a Test Pod

    cat > cuda-devel-test.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: cuda-devel-test
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
          image: nvidia/cuda:12.2.0-devel-ubuntu22.04
          command:
            - bash
            - -lc
            - |
              nvidia-smi
              nvcc -V
          resources:
            limits:
              nvidia.com/gpu: 1
    EOF

Deploy:

    kubectl apply -f cuda-devel-test.yaml

View logs:

    kubectl logs cuda-devel-test

### 13.3 Key Verification Points

Confirm:

    That `nvidia-smi` is functioning correctly.
    That `nvcc -V` returns valid output.

If `nvidia-smi` works fine but `nvcc` is not available, it indicates that the image is not a devel version or that the CUDA Toolkit is incomplete.

---

## Section Fourteen: Deploying PyTorch GPU Test Pod

### 14.1 Purpose of the Test

To verify whether the AI framework can recognize the GPU.

Just because `nvidia-smi` is working does not guarantee that PyTorch will be able to use the GPU.

### 14.2 Creating a Test Pod

The actual image version should be selected based on your specific requirements. Here, a placeholder version is used; you must replace it with the correct version when executing.

    cat > pytorch-gpu-test.yaml <<'EOF'
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
          image: pytorch/pytorch:<specific_version>
          command:
            - bash
            - -lc
            - |
              python3 -c "import torch; print('torch:', torch.__version__); print('cuda version:', torch.version.cuda); print('cuda available:', torch.cuda.is_available()); print('device count:', torch.cuda.device_count())"
          resources:
            limits:
              nvidia.com/gpu: 1
    EOF

Note:

    Do not use `latest`.
    Always specify a clear version of PyTorch and CUDA.
    Ensure that the CUDA Runtime version in the image is compatible with the NVIDIA Driver on the host machine.

### 14.3 Expected Output

The output should include:

    `cuda available: True`
    `device count: 1`

If the output shows `False`, investigate possible issues such as:

- Whether you are using a CPU-based version of PyTorch;
- Whether the host Driver is too old;
- Whether the CUDA Runtime in the image is incompatible;
- Whether the Pod has successfully requested access to the GPU;
- Whether there are `/dev/nvidia*` files inside the container;
- Whether `CUDA_VISIBLE_devices` returns unexpected values.

---

## Section Fifteen: Verifying the GPU Operator Route

If you use a GPU Operator, you can skip the manual deployment of the Device Plugin and directly verify the components managed by the Operator.

### 15.1 Adding a Helm Repository

    helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
    helm repo update

Check available versions:

    helm search repo nvidia/gpu-operator --versions

### 15.2 Creating a Namespace

    kubectl create namespace gpu-operator

### 15.3 Installation Method for Nodes Where Drivers Have Been Manually Installed

If GPU drivers have already been manually installed on the nodes, you can use the following command to install the GPU Operator:

    helm install gpu-operator nvidia/gpu-operator \
      --namespace gpu-operator \
      --version <CHART_VERSION> \
      --set driver.enabled=false \
      --wait

If NVIDIA Container Toolkit has also been manually configured on the nodes, you may need to adjust the installation parameters accordingly:

    --set toolkit.enabled=false

Note:

    Deciding whether to disable the toolkit requires careful consideration.
    Disabling the toolkit without proper configuration in```markdown
--namespace gpu-monitoring \
--version <CHART_VERSION>

View:

    kubectl get pods -n gpu-monitoring -o wide
    kubectl get ds -n gpu-monitoring
    kubectl logs <dcgm-exporter-pod> -n gpu-monitoring

Verify:

    kubectl port-forward -n gpu-monitoring <dcgm-exporter-pod> 9400:9400
    curl http://127.0.0.1:9400/metrics

---

## Section Seventeen: Integrating GPU Metrics with Prometheus

### 17.1 Using ServiceMonitor

If you are using the Prometheus Operator or kube-prometheus-stack, you can utilize ServiceMonitor.

First, check the DCGM Exporter Service:

    kubectl get svc -A | grep -i dcgm
    kubectl get svc -n <namespace> <dcgm-service-name> --show-labels

Example of creating a ServiceMonitor:

    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: dcgm-exporter
      namespace: monitoring
      labels:
        release: prometheus
    spec:
      namespaceSelector:
        matchNames:
          - gpu-operator
      selector:
        matchLabels:
          app.kubernetes.io/name: dcgm-exporter
      endpoints:
        - port: metrics
          path: /metrics
          interval: 30s

Notes:

    The `namespaceSelector` must match the actual namespace.
    The `selector.matchLabels` should correspond to the Service labels.
    The port name must be consistent with the Service’s port configuration.
    Whether to include the `release` label depends on the Prometheus Operator settings.

### 17.2 Using Regular Prometheus scrape_configs

If you are using standard Prometheus, you can configure scrapeConfigs as follows:

    scrape_configs:
      - job_name: 'dcgm-exporter'
        kubernetes_sdconfigs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_name]
            action: keep
            regex: .*dcgm.*exporter.*
          - source_labels: [__meta_kubernetes_pod_ip]
            target_label: __address__
            replacement: $1:9400

Adjust these configurations according to your actual Pod labels.

### 17.3 Verifying Targets

Go to the Prometheus page:

    Status
      ↓
    Targets

Search for:

    dcgm
    gpu
    nvidia

Check the status:

    UP

### 17.4 Common PromQL Queries

GPU Utilization:

    DCGM_FI_DEV_GPU_UTIL

Memory Usage:

    DCGM_FI_DEV_FB_USED

Free Memory:

    DCGM_FI_DEV_FB_FREE

Memory Utilization Percentage:

    DCGM_FI_dev_FB_used / (DCGM_FIDEVFBUSED + DCGM_FI_DEV_FB_FREE) * 100

GPU Temperature:

    DCGM_FI_DEV_GPU_TEMP

Power Consumption:

    DCGM_FI_DEV_POWER_usage

XID Errors:

    DCGM_FI_DEV_XID-errors

---

## Section Eighteen: Verifying Grafana Dashboard

### 18.1 Panels Recommended for the GPU Dashboard

It is suggested to include at least the following panels:

    GPU Overview
    GPU Node List
    GPU Utilization
    GPU Memory Usage
    GPU Temperature
    GPU Power Consumption
    GPU XID Errors
    GPU Pod Distribution
    Namespace GPU Usage
    Device Plugin Status
    DCGM Exporter Status

### 18.2 Verifying Data

Add Panel queries in Grafana:

    DCGM_FI_DEV_GPU_UTIL

If no data is displayed, check the following:

- Whether the Prometheus Target is UP;
- If the metric name exists;
- Whether the dashboard labels match;
- If the time range is correct;
- If the DCGM Exporter is functioning properly;
- Whether there are any GPU nodes under load.

### 18.3 Recommended Dashboard Variables

It is recommended to add variables such as:

    cluster
    node
    namespace
    pod
    gpu
    gpu_model

Adjust variable queries based on your actual labels, for example:

    label_values(DCGM_FI_DEV_GPU_UTIL, Hostname)

or:

    label_values(DCGM_FI_DEV_gpu_UTIL, instance)

---

## Section Nineteen: Verifying AlertManager Alerts

### 19.1 Example of GPU High Temperature Alert

    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: gpu-alert-rules
      namespace: monitoring
      labels:
        release: prometheus
    spec:
      groups:
        - name: gpu.rules
          rules:
            - alert: GPUHighdescription: "The DCGM Exporter target {{ $labels.instance }} has been down for more than 5 minutes."[ ] GPU node cordon/drain experiment completed
[ ] Review template has been organized

---

## Chapter 26: Experimental Record Template

It is recommended to record the following information for each GPU experiment.

    Experiment Date:
    Experimenter:
    Cluster Name:
    Kubernetes Version:
    containerd Version:
    CNI:
    GPU Node Name:
    GPU Node IP:
    OS Version:
    Kernel Version:
    GPU Model:
    Number of GPUs:
    NVIDIA Driver Version:
    nvidia-smi CUDA Version:
    CUDA Toolkit Version:
    NVIDIA Container Toolkit Version:
    Device Plugin Version:
    GPU Operator Version:
    DCGM Exporter Version:
    Prometheus Version:
    Grafana Version:
    AlertManager Version:
    GPU Pod Test Results:
    CUDA Test Results:
    PyTorch Test Results:
    Prometheus Target Status:
    Grafana Dashboard Status:
    Alarm Test Results:
    Fault Experiment Results:
    Remaining Issues:
    Further Optimization:

---

## Chapter 27: Recommendations for Production Environment Evolution

After the experimental environment is successfully established, you can gradually progress towards production-grade GPU operations.

### 27.1 Node Standardization

It is recommended to:

- Unify the OS;
- Unify the Kernel;
- Unify the Driver;
- Unify containerd;
- Unify the NVIDIA Container Toolkit;
- Unify Device Plugin/GPU Operator;
- Standardize node labels;
- Standardize node taints.

### 27.2 Image Standardization

It is recommended to maintain internal images:

    registry.example.com/ai/cuda:12.2.0-base-ubuntu22.04
    registry.example.com/ai/cuda:12.2.0-runtime-ubuntu22.04
    registry.example.com/ai/cuda:12.2.0-devel-ubuntu22.04
    registry.example.com/ai/pytorch:<version>-cuda12.2-runtime

Avoid using the latest image in production.

### 27.3 Resource Management

It is recommended to:

- Implement Namespace ResourceQuotas;
- Use GPU node pools;
- Separate training/inference pools;
- Create dev/test/prod pools;
- Utilize PriorityClass;
- Establish low-priority task preemption policies;
- Monitor idle GPUs;
- Set up alerts for long-term underutilization;
- Maintain daily/weekly reports on GPU usage.

### 27.4 Observability Management

It is recommended to integrate:

- Prometheus;
- Grafana;
- AlertManager;
- Loki or EFK;
- DCGM Exporter;
- kube-state-metrics;
- node-exporter;
- Business metrics;
- GPU cost statistics.

### 27.5 Operations Process Management

It is recommended to establish:

- GPU node deployment processes;
- GPU node maintenance procedures;
- Driver update procedures;
- GPU Operator upgrade processes;
- Fault emergency response scripts;
- XID handling guidelines;
- High-temperature management protocols;
- CUDA OOM handling procedures;
- Resource recycling processes;
- Fault review templates.

---

## Chapter 28: Common Issues Summary

### 28.1 The host machine's nvidia-smi is working, but the Node does not display "nvidia.com/gpu"

Check first:

    Device Plugin
    GPU Operator
    kubelet
    Device Plugin logs
    `kubectl describe node <gpu-node-name>`

Commands to use:

    `kubectl get pods -A | grep -i nvidia`
    `kubectl logs <device-plugin-pod> -n <namespace>`
    `kubectl describe node <gpu-node-name>`

### 28.2 The Node shows "nvidia.com/gpu", but the Pod is in a Pending state

Check first:

    Pod Events
    Whether there is a GPU shortage
    Taint/Toleration settings
    `nodeSelector`
    ResourceQuotas
    CPU/Memory usage

Commands to use:

    `kubectl describe pod <pod-name> -n <namespace>`

### 28.3 The Pod is running, but nvidia-smi inside the container fails

Check first:

    Whether the Pod has requested a GPU
    NVIDIA Container Toolkit
    containerd
    `/dev/nvidia*` directories
    Whether the image includes nvidia-smi

Commands to use:

    `kubectl get pod <pod-name> -o yaml | grep -A20 resources`
    `nvidia-container-cli info`
    `containerd config dump | grep -i nvidia -A30 -B10`

### 28.4 PyTorch cannot detect the GPU

Check first:

    Whether PyTorch is a GPU version
    CUDA Runtime
    cuDNN
    Driver compatibility
    The GPU device inside the container
    `CUDA_VISIBLE_devices`

Commands to use:

    `python3 -c "import torch; print(torch.__version__); print(torch13. Deploy the DCGM Exporter
14. Integrate with Prometheus
15. Integrate with Grafana
16. Configure AlertManager for alerts
17. Conduct fault tolerance tests
18. Generate an acceptance checklist
19. Organize the Runbook
20. Establish a production baseline

When troubleshooting, follow a hierarchical approach:

- If `lspci` reports issues: Check the hardware and BIOS.
- If `nvidia-smi` encounters problems: Verify the drivers and kernel modules.
- If a Node does not display `nvidia.com/gpu`: Investigate the Device Plugin and GPU Operator settings.
- For Pending Pods: Check the Scheduler, resources, Taints, Labels, and Quotas.
- If a Pod is running but lacks a GPU: Verify the NVIDIA Container Toolkit, containerd, and device mounting configurations.
- In case of CUDA or PyTorch errors: Check the Runtime, framework versions, and driver compatibility.
- If GPU metrics are missing: Verify the DCGM Exporter, Prometheus, and ServiceMonitor settings.
- For abnormal GPU alerts: Examine the PrometheusRule configuration, AlertManager settings, and notification channels.

The ultimate goal of the experimental environment should be:

- GPU-enabled Nodes are ready for deployment.
- GPU resources can be effectively scheduled.
- GPU Pods can run smoothly.
- GPU metrics are available for monitoring.
- Fault tolerance tests for GPUs can be conducted successfully.
- GPU alerts are promptly notified.
- GPU-related issues can be thoroughly analyzed and resolved.
- GPU operations can be standardized.

Only by completing this entire process can one truly demonstrate practical skills in Kubernetes GPU operations.

---

## Reference Documents

- Kubernetes GPU Scheduling:
  https://kubernetes.io/docs/tasksmanage-gpus/scheduling-gpus/

- Kubernetes Device Plugins:
  https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/

- NVIDIA Driver Installation Guide:
  https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/

- NVIDIA CUDA Installation Guide for Linux:
  https://docs.nvidia.com/cuda/cuda-installation-guide-linux/

- NVIDIA Container Toolkit:
  https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

- NVIDIA Kubernetes Device Plugin:
  https://github.com/NVIDIA/k8s-device-plugin

- NVIDIA GPU Operator:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/

- NVIDIA DCGM Exporter:
  https://docs.nvidia.com/datacenter/dcgm/latest/gpu-telemetry/dcgm-exporter.html

- NVIDIA DCGM Exporter GitHub:
  https://github.com/NVIDIA/dcgm-exporter

- Prometheus Documentation:
  https://prometheus.io/docs/

- Prometheus Alerting Rules:
  https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

- AlertManager Documentation:
  https://prometheus.io/docs ALERTING/latest/alertmanager/

- Grafana Documentation:
  https://grafana.com/docs/

- kube-state-metrics:
  https://github.com/kubernetes/kube-state-metrics

- Helm Documentation:
  https://helm.sh/docs/