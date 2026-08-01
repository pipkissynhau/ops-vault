# 10-GPU-Operations Experiment Environment Setup and Verification

## Document Notes

This document is used to organize a deployable, verifiable, and scalable Kubernetes GPU operations experiment environment.

This document does not focus on a single component, but rather connects the previous GPU basics, drivers, CUDA, Device Plugin, GPU Pod, monitoring alerts, and fault diagnosis content into a complete experimental workflow.

This document primarily answers the following questions:

- How to plan a Kubernetes GPU operations experiment environment;
- What system-level checks are needed before adding a GPU node to K8S;
- How to confirm that NVIDIA Driver, CUDA, containerd, and NVIDIA Container Toolkit are functioning normally;
- How to choose between Device Plugin or GPU Operator routes;
- How to verify whether Kubernetes recognizes `nvidia.com/gpu`;
- How to deploy a GPU test Pod;
- How to verify GPU, CUDA, and PyTorch capabilities within containers;
- How to integrate DCGM Exporter, Prometheus, and Grafana;
- How to design GPU alerts;
- How to conduct fault simulations for GPU Pod Pending, CUDA OOM, XID, high temperature, etc.;
- How to form a GPU node onboarding acceptance checklist;
- How to evolve the experimental environment into a production-grade GPU operations baseline.

This document is suitable for study after completing the following chapters:

- 01-GPU-Basics and Hardware Composition
- 02-GPU-BIOS and Hardware Optimization
- 03-NVIDIA-Driver Installation and Verification
- 04-CUDA-Installation and Testing
- 05-K8S-GPU-Resource Concepts and Scheduling Principles
- 06-NVIDIA-Device-Plugin-and-Operator-Installation
- 07-GPU-Pod-Deployment and Scheduling Practical Cases
- 08-GPU-Monitoring and Alert Integration
- 09-GPU-Fault Diagnosis Cases and Practical Applications

---

## Tags

#Kubernetes #GPU #NVIDIA #CUDA #DevicePlugin #GPUOperator #DCGM #Prometheus #Grafana #AlertManager #SRE #ExperimentalEnvironment #It'sABattleOfLuck.

---

## Recommended Path

Recommended path:

    06-GPU-and-AI-Infrastructure/04-GPU-Comprehensive-Experiment/10-GPU-Operations-Experiment-Environment-Setup-and-Verification.md

---

## One: Why a Dedicated GPU Operations Experiment Environment is Needed

GPU operations are not just executing:

    nvidia-smi

Nor just writing:

    resources:
      limits:
        nvidia.com/gpu: 1

True GPU operations capabilities require integrating the entire workflow:

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
    Node registration nvidia.com/gpu
      ↓
    GPU Pod scheduling
      ↓
    nvidia-smi / CUDA / PyTorch inside containers
      ↓
    DCGM Exporter
      ↓
    Prometheus
      ↓
    Grafana
      ↓
    AlertManager
      ↓
    Fault diagnosis and capacity management

Without a complete experimental environment, relying solely on documentation makes it difficult to truly understand issues like:

- Why the host `nvidia-smi` is normal, but Kubernetes cannot see the GPU;
- Why the Node has `nvidia.com/gpu`, but the Pod remains Pending;
- Why the Pod is Running, but the container execution of `nvidia-smi` fails;
- Why the container `nvidia-smi` is normal, but PyTorch cannot detect the GPU;
- Why low GPU utilization isn't necessarily abnormal;
- Why high memory isn't necessarily a fault;
- Why XID errors require analysis with `dmesg`, temperature, power consumption, and business logs;
- Why GPU monitoring cannot rely solely on GPU-Util;
- Why production environments need planning for Labels, Taints, Quotas, PriorityClass, image versions, and alert rules.

Thus, the goal of a GPU operations experiment environment is not to "run a test Pod," but to establish a complete capability from node delivery, resource registration, workload execution, monitoring alerts, fault simulation, to acceptance review.

---

## Two: Experiment Objectives

After completing this experiment environment, the following capabilities should be achieved.

### 2.1 Node-side Capabilities

    [ ] GPU node can be correctly recognized by Linux
    [ ] NVIDIA Driver installation completed
    [ ] nvidia-smi output is normal
    [ ] CUDA basic tests passed
    [ ] NVIDIA Container Toolkit is functioning normally
    [ ] containerd supports GPU containers
    [ ] No severe XID / PCIe / temperature anomalies on the node

### 2.2 Kubernetes-side Capabilities

    [ ] GPU node joined Kubernetes cluster
    [ ] GPU node is in Ready state
    [ ] Device Plugin or GPU Operator is running normally
    [ ] nvidia.com/gpu appears in Node Capacity / Allocatable
    [ ] GPU Pod can be scheduled successfully
    [ ] nvidia-smi is normal inside GPU Pod containers
    [ ] GPU Pod can run CUDA / PyTorch tests

### 2.3 Observability Capabilities

    [ ] DCGM Exporter is running normally
    [ ] Prometheus can collect GPU metrics
    [ ] Grafana can display GPU utilization, memory, temperature, power consumption
    [ ] AlertManager can receive GPU alerts
    [ ] Alert rules for GPU high temperature, high memory, XID, Exporter Down, etc., are available

### 2.4 Fault Simulation Capabilities /think

- [ ] Can simulate GPU Pod Pending
- [ ] Can simulate Taint/Toleration mismatch
- [ ] Can simulate nodeSelector mismatch
- [ ] Can simulate CUDA OOM
- [ ] Can simulate DCGM Exporter Down
- [ ] Can simulate Device Plugin abnormality
- [ ] Can execute cordon / drain / uncordon maintenance flow
- [ ] Can form fault analysis records

---

## Three. Experimental Environment Planning

### 3.1 Recommended Experimental Topology

This experiment can be based on an existing kubeadm Kubernetes cluster extended with GPU nodes.

Recommended topology:

    +-----------------------------+
    | ops-server                  |
    | 10.0.0.10                   |
    | Harbor / GitLab / Jenkins   |
    +---------------+-------------+
                    |
                    |
    +---------------+-------------+
    | k8s-master                  |
    | 10.0.0.20                   |
    | Control Plane               |
    +---------------+-------------+
                    |
        -----------------------------
        |                           |
    +---+----------------+      +---+----------------+
    | k8s-worker01       |      | k8s-worker02       |
    | 10.0.0.21          |      | 10.0.0.22          |
    | CPU Workloads      |      | CPU Workloads      |
    +--------------------+      +--------------------+

                    |
                    |
    +---------------+-------------+
    | k8s-gpu-node01              |
    | 10.0.0.30                   |
    | NVIDIA GPU / GPU Pods       |
    +-----------------------------+

Monitoring components can be deployed in:

    monitoring namespace

Including:

    Prometheus
    Grafana
    AlertManager
    kube-state-metrics
    node-exporter

GPU monitoring components:

    DCGM Exporter
    NVIDIA Device Plugin
    or GPU Operator

---

## Four. Experimental Node Planning

### 4.1 IP Address Planning

Recommend using an independent GPU node to avoid confusion with regular Workers.

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

Add to all nodes `/etc/hosts`:

    10.0.0.20 k8s-master
    10.0.0.21 k8s-worker01
    10.0.0.22 k8s-worker02
    10.0.0.30 k8s-gpu-node01

### 4.3 System Version Recommendation

Recommended:

    Ubuntu Server 22.04.5 LTS

Alternatively:

    Rocky Linux 9

This document uses Ubuntu Server 22.04 as the main line, Rocky Linux 9 can refer to corresponding installation commands.

### 4.4 Kubernetes Version Recommendation

Experimental environment recommends using the same version as the existing cluster.

Check version:

    kubectl version
    kubelet --version

Recommended record:

    Kubernetes Version:
    containerd Version:
    CNI:
    Kernel Version:
    NVIDIA Driver Version:
    CUDA Version:
    Device Plugin / GPU Operator Version:

---

## Five. Experimental Route Selection

GPU integration with Kubernetes has two common routes.

### 5.1 Route One: Manual Driver + NVIDIA Container Toolkit + Device Plugin

Suitable for:

- Learning;
- Small-scale experiments;
- Wanting to understand each layer;
- Already manually installed drivers;
- Not wanting Operator to manage drivers;
- Only needing basic GPU scheduling.

Link:

    Manually install NVIDIA Driver
      ↓
    Manually install NVIDIA Container Toolkit
      ↓
    Configure containerd
      ↓
    Helm install NVIDIA Device Plugin
      ↓
    Verify nvidia.com/gpu
      ↓
    Deploy GPU Pod
      ↓
    Integrate DCGM Exporter

Advantages:

    Clear hierarchy
    Easy to understand principles
    Suitable for learning and troubleshooting training

Disadvantages:

    High maintenance cost with multiple nodes
    Driver / Toolkit / DCGM need to be managed manually

### 5.2 Route Two: NVIDIA GPU Operator

Suitable for:

- Production environment;
- Multi-GPU nodes;
- Desire to unify management of NVIDIA components;
- Need for DCGM Exporter;
- Need for GPU Feature Discovery;
- Need for MIG;
- Need for more complete lifecycle management.

Link:

    Helm install GPU Operator
      ↓
    Operator manages Driver / Toolkit / Device Plugin / DCGM
      ↓
    Node registers nvidia.com/gpu
      ↓
    Deploy GPU Pod
      ↓
    Prometheus collects DCGM metrics

Advantages:

    High standardization
    Unified component management
    More suitable for production

Disadvantages:

    High initial understanding cost
    Values configuration needs caution
    Need to clarify whether to let Operator manage drivers and runtime

### 5.3 Recommended experimental route

In the learning phase, it is recommended to prioritize:

    Route 1: Manual driver + Toolkit + Device Plugin

Reasons:

    Can clearly understand each layer.
    Know where to check when problems occur.
    Will not hide all issues in the Operator.

After completing basic experiments, expand:

    Route 2: GPU Operator

This aligns better with the operations learning path.

---

## SixI don't know.Pre-Cluster Join Checks for GPU Nodes

Before GPU nodes join Kubernetes, they must first complete hardware, driver, and runtime baseline verification.

### 6.1 Check System Version

    cat /etc/os-release
    uname -a
    hostnamectl

### 6.2 Check if GPU is recognized by the system

    lspci | grep -i nvidia

If no output, do not install the driver yet, prioritize checking:

- GPU is properly inserted;
- Power supply;
- PCIe slot;
- riser;
- BIOS;
- Above 4G Decoding;
- Whether the server supports the GPU.

### 6.3 Check PCIe topology

    lspci -tv

Save:

    lspci -tv > pci-tree.txt

### 6.4 Check PCIe link speed and width

First get the PCI ID:

    lspci | grep -i nvidia

Example:

    65:00.0 3D controller: NVIDIA Corporation Device xxxx

Check:

    lspci -vvv -s 65:00.0 | grep -i "LnkCap"
    lspci -vvv -s 65:00.0 | grep -i "LnkSta"

Focus on:

    LnkCap:
        Theoretical capability

    LnkSta:
        Current actual working state

### 6.5 Check BIOS and hardware information

    dmidecode -t bios
    dmidecode -t system
    dmidecode -t baseboard

### 6.6 Check BMC sensors

If ipmitool is installed:

    ipmitool sensor
    ipmitool sel list

Used to check:

- Fans;
- Power;
- Temperature;
- Hardware alerts;
- Chassis status.

---

## SevenI don't know.Install and Verify NVIDIA Driver

### 7.1 Pre-installation Preparation

Ubuntu:

    apt-get update
    apt-get install -y build-essential dkms linux-headers-$(uname -r)
    apt-get install -y pciutils lshw mokutil

Rocky Linux:

    dnf install -y gcc make dkms kernel-devel-$(uname -r) kernel-headers-$(uname -r)
    dnf install -y pciutils lshw elfutils-libelf-devel

### 7.2 Disable nouveau

Create configuration:

    cat > /etc/modprobe.d/blacklist-nouveau.conf <<'EOF'
    blacklist nouveau
    options nouveau modeset=0
    EOF

Ubuntu:

    update-initramfs -u

Rocky Linux:

    dracut --force

Reboot:

    reboot

Verify:

    lsmod | grep nouveau

No output indicates nouveau is not loaded.

### 7.3 Install NVIDIA Driver

In experimental environments, you can use system packages or NVIDIA official repository.

Ubuntu example:

    apt-get update
    apt-get install -y ubuntu-drivers-common
    ubuntu-drivers devices

Install recommended driver:

    ubuntu-drivers install

Or install specific version:

    apt-get install -y nvidia-driver-<version>-server

Note:

    <version> should be selected based on GPU model, CUDA compatibility, and actual repository.
    Production environments must fix the version; do not use the latest version blindly.

### 7.4 Reboot and Verify

    reboot

Verify:

    nvidia-smi
    nvidia-smi -L
    nvidia-smi -q
    lsmod | grep nvidia
    ls -l /dev/nvidia*

### 7.5 Check XID errors

    dmesg | grep -i xid
    journalctl -k | grep -i xid

If XID appears, record and analyze it first; do not proceed to subsequent experiments directly.

---

## EightI don't know.CUDA Basic Verification

### 8.1 Is CUDA Toolkit Required on the Host

If it's just a Kubernetes GPU Worker, the host does not necessarily need the full CUDA Toolkit.

Host focus is on:

    NVIDIA Driver
    NVIDIA Container Toolkit
    containerd
    kubelet
    Device Plugin / GPU Operator

CUDA Runtime is typically inside container images.

However, for experimentation, you can install CUDA Toolkit for basic verification.

### 8.2 Verify CUDA Version in nvidia-smi

    nvidia-smi

Note:

The CUDA Version displayed by nvidia-smi indicates the driver's supported capabilities. It does not mean the host has already installed the CUDA Toolkit.

### 8.3 Verifying nvcc

If CUDA Toolkit is installed:

    nvcc -V

If not installed, the "command not found" error does not indicate driver issues.

### 8.4 Using Containers to Verify CUDA

It is recommended to verify via containers:

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

If not using Docker but Kubernetes + containerd, verify via GPU Pod later.

---

## Nine. Installing NVIDIA Container Toolkit

### 9.1 Purpose

NVIDIA Container Toolkit enables containers to use GPU.

It provides:

- GPU device mounting;
- Driver library injection;
- NVIDIA runtime configuration;
- GPU visibility inside containers;
- Integration with Device Plugin.

If not properly configured, common symptoms:

    Host nvidia-smi works normally
    Node may have nvidia.com/gpu
    But Pod nvidia-smi fails

### 9.2 Installing Toolkit

Actual installation commands should follow NVIDIA's official documentation.

Common approach for Ubuntu:

    apt-get update
    apt-get install -y nvidia-container-toolkit

Common approach for Rocky Linux:

    dnf install -y nvidia-container-toolkit

### 9.3 Configuring containerd

Use:

    nvidia-ctk runtime configure --runtime=containerd

Restart:

    systemctl restart containerd
    systemctl restart kubelet

### 9.4 Verifying Toolkit

    nvidia-container-cli info

Check containerd configuration:

    containerd config dump | grep -i nvidia -A30 -B10

If no NVIDIA runtime content is seen, recheck Toolkit configuration.

---

## Ten. Adding GPU Nodes to Kubernetes Cluster

### 10.1 kubeadm join

Generate join command on master node:

    kubeadm token create --print-join-command

Execute the output command on GPU node.

Example:

    kubeadm join 10.0.0.20:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

### 10.2 Verifying Node Join

On master node execute:

    kubectl get nodes -o wide

Expected:

    k8s-gpu-node01   Ready

### 10.3 Labeling GPU Nodes

    kubectl label node k8s-gpu-node01 node-role.kubernetes.io/gpu=true
    kubectl label node k8s-gpu-node01 accelerator=nvidia
    kubectl label node k8s-gpu-node01 gpu.vendor=nvidia

Add model-specific labels:

    kubectl label node k8s-gpu-node01 gpu.model=<gpu-model>
    kubectl label node k8s-gpu-node01 gpu.workload=dev

Example:

    kubectl label node k8s-gpu-node01 gpu.model=a10
    kubectl label node k8s-gpu-node01 gpu.workload=dev

### 10.4 Adding Taints to GPU Nodes

To prevent regular Pods from scheduling on GPU nodes:

    kubectl taint node k8s-gpu-node01 nvidia.com/gpu=true:NoSchedule

Subsequent GPU Pods need to add toleration.

---

## Eleven. Deploying NVIDIA Device Plugin

This section uses the manual Device Plugin approach.

### 11.1 Creating Namespace

    kubectl create namespace nvidia-device-plugin

### 11.2 Installing with Helm

Add repository:

    helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
    helm repo update

Check versions:

    helm search repo nvdp/nvidia-device-plugin --versions

Install specific version:

    helm install nvidia-device-plugin nvdp/nvidia-device-plugin \
      --namespace nvidia-device-plugin \
      --version <CHART_VERSION>

Note:

    <CHART_VERSION> should be selected based on helm search repo output.
    Production environments must fix the version.

### 11.3 Checking DaemonSet

    kubectl get ds -n nvidia-device-plugin
    kubectl get pods -n nvidia-device-plugin -o wide

Expected:

    nvidia-device-plugin Pod running on GPU nodes.

### 11.4 Checking Logs

    kubectl get pods -n nvidia-device-plugin
    kubectl logs <device-plugin-pod> -n nvidia-device-plugin

### 11.5 Verifying GPU Registration

    kubectl describe node k8s-gpu-node01

Focus on:

    Capacity:
      nvidia.com/gpu: 1

    Allocatable:
      nvidia.com/gpu: 1

If you don't see `nvidia.com/gpu`, it indicates that the Device Plugin registration failed.

---

## Twelve. Deploy GPU Test Pod

### 12.1 Create Minimal GPU Test Pod

    cat > gpu-test.yaml <<'EOF'
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
    EOF

Deployment:

    kubectl apply -f gpu-test.yaml

Check:

    kubectl get pod gpu-test -o wide
    kubectl logs gpu-test
    kubectl describe pod gpu-test

### 12.2 Verify Results

If successful, the logs should show:

    NVIDIA-SMI
    Driver Version
    CUDA Version
    GPU Name
    Memory-Usage

Explanation:

    Kubernetes can schedule GPU Pod
    The container can see GPU
    Device Plugin is basically normal
    NVIDIA Container Toolkit is basically normal
    Runtime successfully mounts GPU device

### 12.3 Common Failure Judgments

If Pod is Pending:

    kubectl describe pod gpu-test

Check Events.

If:

    insufficient nvidia.com/gpu

It indicates GPU resources are insufficient or not registered.

If:

    untolerated taint

It indicates toleration mismatch.

If:

    didn't match Pod's node affinity/selector

It indicates nodeSelector or node label mismatch.

If Pod is Running but logs fail:

    Check container startup errors, image, runtime, and device mounting.

---

## Thirteen. Deploy CUDA devel Test Pod

### 13.1 Why Need devel Image

`base` or `runtime` images may not include `nvcc`.

If you want to verify CUDA Toolkit compilation tools, you need to use `devel` image.

### 13.2 Create Test Pod

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

Deployment:

    kubectl apply -f cuda-devel-test.yaml

Check:

    kubectl logs cuda-devel-test

### 13.3 Verification Focus

Confirm:

    nvidia-smi is normal
    nvcc -V is normal

If `nvidia-smi` is normal but `nvcc` does not exist, it indicates the image is not devel or CUDA Toolkit is incomplete.

---

## Fourteen. Deploy PyTorch GPU Test Pod

### 14.1 Test Purpose

Verify if the AI framework layer can recognize GPU.

`nvidia-smi` being normal only indicates driver and device visibility, but does not guarantee PyTorch is usable.

### 14.2 Create Test Pod

The actual image version should be selected according to business needs. Here, a placeholder format is used, and it must be replaced with a specific version when executed.

```yaml
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
      image: pytorch/pytorch:<明确版本>
      command:
        - bash
        - -lc
        - |
          python3 -c "import torch; print('torch:', torch.__version__); print('cuda version:', torch.version.cuda); print('cuda available:', torch.cuda.is_available()); print('device count:', torch.cuda.device_count())"
      resources:
        limits:
          nvidia.com/gpu: 1
EOF
```

**Note:**

- Do not use `latest`.
- Must select a specific PyTorch + CUDA version.
- The CUDA Runtime image must be compatible with the host NVIDIA Driver.

### 14.3 Expected Output

**Expected:**
```
cuda available: True
device count: 1
```

**If output is False, troubleshoot:**
- Whether CPU version PyTorch;
- Whether host Driver is too old;
- Whether CUDA Runtime image is incompatible;
- Whether Pod has requested GPU;
- Whether `/dev/nvidia*` exists in container;
- Whether `CUDA_VISIBLE_DEVICES` is abnormal.

---

## FifteenI don't know.GPU Operator Route Validation

If using GPU Operator, you can skip manual Device Plugin deployment and directly validate Operator-managed components.

### 15.1 Add Helm Repository

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
```

Check versions:

```bash
helm search repo nvidia/gpu-operator --versions
```

### 15.2 Create Namespace

```bash
kubectl create namespace gpu-operator
```

### 15.3 Installation Method for Nodes with Manually Installed Drivers

If GPU nodes have already manually installed NVIDIA Driver, use:

```bash
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --version <CHART_VERSION> \
  --set driver.enabled=false \
  --wait
```

If nodes have already manually installed and configured NVIDIA Container Toolkit, you can also evaluate based on actual conditions:

```bash
--set toolkit.enabled=false
```

**Note:**
- Disabling toolkit must be done cautiously.
- If toolkit is disabled but containerd is not properly configured, Pods may not be able to use GPU.

### 15.4 Check Operator Components

```bash
kubectl get pods -n gpu-operator -o wide
kubectl get ds -n gpu-operator
kubectl get clusterpolicy
kubectl describe clusterpolicy
```

Expected components may include:

- `nvidia-device-plugin`
- `nvidia-container-toolkit`
- `nvidia-dcgm-exporter`
- `gpu-feature-discovery`
- `node-feature-discovery`
- `nvidia-operator-validator`

### 15.5 Validate Node GPU Resources

```bash
kubectl describe node k8s-gpu-node01
```

Check:

```
nvidia.com/gpu
```

### 15.6 Continue Executing GPU Pod Test

Use the previous files:

- `gpu-test.yaml`
- `cuda-devel-test.yaml`
- `pytorch-gpu-test.yaml`

Verify whether the Operator route is complete.

---

## SixteenI don't know.Deploy DCGM Exporter

### 16.1 Use GPU Operator's Built-in DCGM Exporter

If using GPU Operator, DCGM Exporter is typically included.

Check:

```bash
kubectl get pods -n gpu-operator | grep -i dcgm
kubectl get svc -n gpu-operator | grep -i dcgm
```

Check logs:

```bash
kubectl logs <dcgm-exporter-pod> -n gpu-operator
```

Verify metrics:

```bash
kubectl port-forward -n gpu-operator <dcgm-exporter-pod> 9400:9400
curl http://127.0.0.1:9400/metrics
```

### 16.2 Install DCGM Exporter Separately

If not using GPU Operator, you can install it separately.

Add repository:

```bash
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update
```

Create Namespace:

```bash
kubectl create namespace gpu-monitoring
```

Check versions:

helm search repo gpu-helm-charts/dcgm-exporter --versions

Installation:

    helm install dcgm-exporter gpu-helm-charts/dcgm-exporter \
      --namespace gpu-monitoring \
      --version <CHART_VERSION>

Check:

    kubectl get pods -n gpu-monitoring -o wide
    kubectl get ds -n gpu-monitoring
    kubectl logs <dcgm-exporter-pod> -n gpu-monitoring

Verification:

    kubectl port-forward -n gpu-monitoring <dcgm-exporter-pod> 9400:9400
    curl http://127.0.0.1:9400/metrics

---

## 17. Prometheus Integration with GPU Metrics

### 17.1 Using ServiceMonitor

If using Prometheus Operator or kube-prometheus-stack, you can use ServiceMonitor.

First check DCGM Exporter Service:

    kubectl get svc -A | grep -i dcgm
    kubectl get svc -n <namespace> <dcgm-service-name> --show-labels

Create ServiceMonitor example:

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

    namespaceSelector must match the actual Namespace.
    selector.matchLabels must match the Service labels.
    port name must match the Service port name.
    release label is required based on Prometheus Operator configuration.

### 17.2 Regular Prometheus scrape_configs

If using regular Prometheus, configure:

    scrape_configs:
      - job_name: 'dcgm-exporter'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_name]
            action: keep
            regex: .*dcgm.*exporter.*
          - source_labels: [__meta_kubernetes_pod_ip]
            target_label: __address__
            replacement: $1:9400

Adjust configurations based on actual Pod labels.

### 17.3 Verify Target

Enter Prometheus page:

    Status
      ↓
    Targets

Search:

    dcgm
    gpu
    nvidia

Confirm status:

    UP

### 17.4 Common PromQL

GPU utilization:

    DCGM_FI_DEV_GPU_UTIL

Memory usage:

    DCGM_FI_DEV_FB_USED

Memory free:

    DCGM_FI_DEV_FB_FREE

Memory usage rate:

    DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE) * 100

GPU temperature:

    DCGM_FI_DEV_GPU_TEMP

GPU power consumption:

    DCGM_FI_DEV_POWER_USAGE

XID errors:

    DCGM_FI_DEV_XID_ERRORS

---

## 18. Grafana Dashboard Verification

### 18.1 Panels Should Include in GPU Dashboard

Recommended to include at least:

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

### 18.2 Verify Data

Add Panel query in Grafana:

    DCGM_FI_DEV_GPU_UTIL

If no data, check:

- Whether Prometheus Target is UP;
- Whether metric name exists;
- Whether Dashboard label matches;
- Whether time range is correct;
- Whether DCGM Exporter is working;
- Whether GPU nodes have workload.

### 18.3 Recommended Dashboard Variables

Recommended to add variables:

    cluster
    node
    namespace
    pod
    gpu
    gpu_model

Variable queries should be adjusted based on actual labels, for example:

    label_values(DCGM_FI_DEV_GPU_UTIL, Hostname)

or:

    label_values(DCGM_FI_DEV_GPU_UTIL, instance)

---

## 19. AlertManager Alert Verification

### 19.1 Example of GPU High Temperature Alert /think

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
        - alert: GPUHighTemperature
          expr: DCGM_FI_DEV_GPU_TEMP > 80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "GPU Temperature Too High"
            description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} temperature is {{ $value }}°C for more than 5 minutes."

### 19.2 GPU Memory High Alert

    - alert: GPUMemoryUsageHigh
      expr: |
        (
          DCGM_FI_DEV_FB_USED
          /
          (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE)
          * 100
        ) > 90
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "GPU Memory Usage Too High"
        description: "GPU memory usage is above 90% for more than 10 minutes."

### 19.3 XID Error Alert

    - alert: GPUXIDError
      expr: DCGM_FI_DEV_XID_ERRORS > 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "GPU XID Error Detected"
        description: "GPU XID error detected. Check dmesg, journalctl and GPU health immediately."

### 19.4 DCGM Exporter Down Alert

    - alert: DCGMExporterDown
      expr: up{job=~".*dcgm.*"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "DCGM Exporter Unavailable"
        description: "DCGM Exporter target {{ $labels.instance }} has been down for more than 5 minutes."

### 19.5 Verify Alert Rules

Check PrometheusRule:

    kubectl get prometheusrule -n monitoring

Check Prometheus page:

    Status
      ↓
    Rules

Check Alert page:

    Alerts

Confirm in AlertManager page whether alerts are received.

---

## Twenty, Fault Simulation One: GPU Pod Pending

### 20.1 Simulation Objective

Verify how to locate Pod Pending when GPU resources are insufficient or scheduling conditions are mismatched.

### 20.2 Method One: Requesting More GPUs Than Nodes

If nodes have only 1 GPU, create a Pod requesting 2:

    resources:
      limits:
        nvidia.com/gpu: 2

Check:

    kubectl get pod <pod-name> -o wide
    kubectl describe pod <pod-name>

Expected event:

    insufficient nvidia.com/gpu

### 20.3 Method Two: nodeSelector Mismatch

Pod YAML:

    nodeSelector:
      gpu.model: h100

But nodes lack this label.

Check:

    kubectl describe pod <pod-name>

Expected:

    node(s) didn't match Pod's node affinity/selector

### 20.4 Method Three: Missing Tolerations

GPU nodes have taint:

    nvidia.com/gpu=true:NoSchedule

Pod lacks toleration.

Expected:

    node(s) had untolerated taint

### 20.5 Post-Simulation Record

Record:

    Pod YAML
    Pod Events
    Node Label
    Node Taint
    Node Capacity
    Trigger Reason
    Fix Action

---

## Twenty-One, Fault Simulation Two: Pod Running but Cannot Use GPU Inside Container

### 21.1 Simulation Objective

Verify methods to troubleshoot when GPU devices are not visible in a running Pod.

### 21.2 Common Simulation Methods

Method One:

    Pod does not declare nvidia.com/gpu

Method Two:

    Use a regular image, such as busybox

Method Three:

    containerd NVIDIA runtime configuration anomaly

### 21.3 Troubleshooting Steps

Check Pod YAML:

    kubectl get pod <pod-name> -o yaml | grep -A20 resources

Enter container:

    kubectl exec -it <pod-name> -- sh

Check:

    ls -l /dev/nvidia*
    echo $CUDA_VISIBLE_DEVICES

Node side: /think

nvidia-container-cli info
containerd config dump | grep -i nvidia -A30 -B10

### 21.4 Remediation Actions

- Add GPU limit to the Pod;
- Fix NVIDIA Container Toolkit;
- Fix containerd;
- Use CUDA test image;
- Rebuild the Pod.

---

## 22. Fault Simulation Three: CUDA OOM

### 22.1 Simulation Objective

Understand the difference between GPU memory exhaustion and Kubernetes memory exhaustion.

### 22.2 Trigger Method

In actual experiments, you can create large tensors with PyTorch to trigger memory exhaustion.

Example idea:

    python3 - <<'PY'
    import torch
    x = []
    while True:
        x.append(torch.randn((1024, 1024, 1024), device="cuda"))
    PY

Note:

    This experiment will quickly consume all GPU memory.
    It is only recommended to run on dedicated experimental nodes.
    Do not run on production GPU nodes.

### 22.3 Observation Metrics

Container or host node:

    nvidia-smi

Prometheus:

    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_FB_FREE

Pod logs:

    kubectl logs <pod-name>

Expected error:

    CUDA out of memory

### 22.4 Handling Method

- Reduce batch size;
- Reduce concurrency;
- Clean up abnormal processes;
- Restart Pod;
- Use GPU with larger memory;
- Use MIG for isolation;
- Use FP16 / BF16 / quantization.

---

## 23. Fault Simulation Four: DCGM Exporter Down

### 23.1 Simulation Objective

Verify if alerts can be triggered when GPU monitoring targets are unavailable.

### 23.2 Operation

Delete DCGM Exporter Pod:

    kubectl delete pod <dcgm-exporter-pod> -n <namespace>

Or temporarily scale down DaemonSet.

Check Prometheus Targets:

    Status -> Targets

Expected:

    Target Down

AlertManager should trigger:

    DCGMExporterDown

### 23.3 Recovery

If it's a DaemonSet, the Pod will automatically be recreated.

Check:

    kubectl get pods -n <namespace> -o wide
    kubectl logs <dcgm-exporter-pod> -n <namespace>

---

## 24. Fault Simulation Five: GPU Node Maintenance

### 24.1 Simulation Objective

Master the GPU node maintenance process.

### 24.2 Cordon Node

    kubectl cordon k8s-gpu-node01

Effect:

    New Pods will no longer be scheduled to this node.

### 24.3 Drain Node

    kubectl drain k8s-gpu-node01 --ignore-daemonsets --delete-emptydir-data

Note:

    Training tasks may lose progress.
    Production environments must confirm business support for migration or checkpointing in advance.

### 24.4 Maintain Node

Can perform:

- Driver upgrade;
- containerd configuration adjustment;
- BIOS check;
- GPU stress test;
- Node reboot;
- Hardware check.

### 24.5 Resume Scheduling

    kubectl uncordon k8s-gpu-node01

Verification:

    kubectl get nodes
    kubectl describe node k8s-gpu-node01
    kubectl apply -f gpu-test.yaml
    kubectl logs gpu-test

---

## 25. Experiment Environment Acceptance Checklist

### 25.1 Hardware Layer

    [ ] GPU is correctly installed
    [ ] Power supply is normal
    [ ] Cooling is normal
    [ ] BIOS is configured
    [ ] Above 4G Decoding is confirmed
    [ ] lspci can see NVIDIA GPU
    [ ] PCIe Speed / Width meets expectations
    [ ] BMC has no severe hardware alerts

### 25.2 Driver Layer

    [ ] NVIDIA Driver installation is complete
    [ ] nvidia-smi is normal
    [ ] nvidia-smi -L is normal
    [ ] nvidia-smi -q is normal
    [ ] lsmod can see NVIDIA modules
    [ ] /dev/nvidia* exists
    [ ] No severe XID errors

### 25.3 Container Runtime Layer

    [ ] NVIDIA Container Toolkit is installed
    [ ] nvidia-container-cli info is normal
    [ ] containerd is configured with NVIDIA runtime
    [ ] containerd has been restarted
    [ ] kubelet has been restarted
    [ ] Runtime configuration is recorded

### 25.4 Kubernetes Layer

    [ ] GPU node is Ready
    [ ] GPU node labels are correct
    [ ] GPU node taints are correct
    [ ] Device Plugin or GPU Operator is normal
    [ ] Node shows nvidia.com/gpu
    [ ] GPU Pod can schedule successfully
    [ ] nvidia-smi inside container is normal
    [ ] CUDA test is normal
    [ ] PyTorch test is normal

### 25.5 Monitoring Layer

    [ ] DCGM Exporter is normal
    [ ] /metrics is accessible
    [ ] Prometheus Target is UP
    [ ] Grafana Dashboard has data
    [ ] GPU utilization is visible
    [ ] Memory is visible
    [ ] Temperature is visible
    [ ] Power consumption is visible
    [ ] XID metrics are visible
    [ ] AlertManager can receive GPU alerts

### 25.6 Fault Simulation Layer

[ ] GPU Pod Pending Exercise Completed
[ ] Taint/Toleration Exercise Completed
[ ] nodeSelector Exercise Completed
[ ] CUDA OOM Exercise Completed
[ ] DCGM Exporter Down Exercise Completed
[ ] GPU Node cordon/drain Exercise Completed
[ ] Post-mortem Template Prepared

---

## Twenty-six, Experiment Record Template

It is recommended to record the following information for each GPU experiment.

    Experiment Date:
    Experiment Personnel:
    Cluster Name:
    Kubernetes Version:
    containerd Version:
    CNI:
    GPU Node Name:
    GPU Node IP:
    OS Version:
    Kernel Version:
    GPU Model:
    GPU Count:
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
    GPU Pod Test Result:
    CUDA Test Result:
    PyTorch Test Result:
    Prometheus Target Status:
    Grafana Dashboard Status:
    Alert Test Result:
    Fault Drill Result:
    Outstanding Issues:
    Subsequent Optimization:

---

## Twenty-seven, Production Environment Evolution Suggestions

After the experiment environment is validated, you can gradually evolve to production-grade GPU operations capabilities.

### 27.1 Node Standardization

Recommendations:

- Unified OS
- Unified Kernel
- Unified Driver
- Unified containerd
- Unified NVIDIA Container Toolkit
- Unified Device Plugin / GPU Operator
- Unified Node Labels
- Unified Node Taints

### 27.2 Image Standardization

Recommend maintaining internal images:

    registry.example.com/ai/cuda:12.2.0-base-ubuntu22.04
    registry.example.com/ai/cuda:12.2.0-runtime-ubuntu22.04
    registry.example.com/ai/cuda:12.2.0-devel-ubuntu22.04
    registry.example.com/ai/pytorch:<version>-cuda12.2-runtime

Do not use in production:

    latest

### 27.3 Resource Governance

Recommendations:

- Namespace ResourceQuota
- GPU Node Pool
- Training/Inference Separated Pools
- dev/test/prod Separated Pools
- PriorityClass
- Low Priority Task Preemption Strategy
- Idle GPU Statistics
- Long-term Low Utilization Alert
- GPU Usage Daily/Weekly Reports

### 27.4 Observability Governance

Recommend connecting:

- Prometheus
- Grafana
- AlertManager
- Loki or EFK
- DCGM Exporter
- kube-state-metrics
- node-exporter
- Business Metrics
- GPU Cost Statistics

### 27.5 Operation Process Governance

Recommend establishing:

- GPU Node Onboarding Process
- GPU Node Maintenance Process
- Driver Upgrade Process
- GPU Operator Upgrade Process
- Fault Emergency Runbook
- XID Handling Specification
- High Temperature Handling Specification
- CUDA OOM Handling Specification
- Resource Recycling Process
- Fault Post-mortem Template

---

## Twenty-eight, Common Issues Summary

### 28.1 Host nvidia-smi is normal, but Node has no nvidia.com/gpu

Prior checks:

    Device Plugin
    GPU Operator
    kubelet
    Device Plugin Logs
    Node describe

Commands:

    kubectl get pods -A | grep -i nvidia
    kubectl logs <device-plugin-pod> -n <namespace>
    kubectl describe node <gpu-node-name>

### 28.2 Node has nvidia.com/gpu, but Pod is Pending

Prior checks:

    Pod Events
    GPU Availability
    Taint/Toleration
    nodeSelector
    ResourceQuota
    CPU/Memory

Commands:

    kubectl describe pod <pod-name> -n <namespace>

### 28.3 Pod is Running, but container nvidia-smi fails

Prior checks:

    Pod GPU Request
    NVIDIA Container Toolkit
    containerd
    /dev/nvidia*
    Image contains nvidia-smi

Commands:

    kubectl get pod <pod-name> -o yaml | grep -A20 resources
    nvidia-container-cli info
    containerd config dump | grep -i nvidia -A30 -B10

### 28.4 PyTorch detects no GPU

Prior checks:

    PyTorch GPU Version
    CUDA Runtime
    cuDNN
    Driver Compatibility
    Container GPU Devices
    CUDA_VISIBLE_DEVICES

Commands:

    python3 -c "import torch; print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available()); print(torch.cuda.device_count())"

### 28.5 Grafana has no GPU data

Prior checks:

    DCGM Exporter Pod
    /metrics
    Prometheus Target
    ServiceMonitor
    PromQL Metric Name
    Dashboard label

Commands:

    kubectl get pods -A | grep -i dcgm
    curl http://127.0.0.1:9400/metrics

---

## 29. Cleanup of Experimental Environment

### 29.1 Delete Test Pod

    kubectl delete pod gpu-test
    kubectl delete pod cuda-devel-test
    kubectl delete pod pytorch-gpu-test

### 29.2 Delete Test Job / Deployment

    kubectl delete job <job-name> -n <namespace>
    kubectl delete deployment <deployment-name> -n <namespace>

### 29.3 Uninstall Device Plugin

    helm uninstall nvidia-device-plugin -n nvidia-device-plugin
    kubectl delete namespace nvidia-device-plugin

### 29.4 Uninstall DCGM Exporter

    helm uninstall dcgm-exporter -n gpu-monitoring
    kubectl delete namespace gpu-monitoring

### 29.5 Uninstall GPU Operator

    helm uninstall gpu-operator -n gpu-operator
    kubectl delete namespace gpu-operator

Note:

    Before uninstalling GPU Operator, confirm whether it manages the Driver and Toolkit.
    If Operator manages the driver, uninstallation may affect node GPU capabilities.
    Do not arbitrarily uninstall in production environments.

### 29.6 Remove Node Taints and Labels

Remove taints:

    kubectl taint node k8s-gpu-node01 nvidia.com/gpu=true:NoSchedule-

Remove labels:

    kubectl label node k8s-gpu-node01 accelerator-
    kubectl label node k8s-gpu-node01 gpu.vendor-
    kubectl label node k8s-gpu-node01 gpu.model-
    kubectl label node k8s-gpu-node01 gpu.workload-

---

## 30. Summary

The value of GPU operations experiment environment is not simply verifying whether a GPU Pod can run, but fully establishing the production chain from hardware to monitoring alerts.

A complete experimental chain should include:

    1. Plan GPU nodes
    2. Check BIOS / PCIe / hardware
    3. Install NVIDIA Driver
    4. Verify nvidia-smi
    5. Install NVIDIA Container Toolkit
    6. Configure containerd
    7. Add GPU node to Kubernetes
    8. Deploy Device Plugin or GPU Operator
    9. Verify node has nvidia.com/gpu
    10. Deploy GPU Pod
    11. Verify nvidia-smi inside container
    12. Verify CUDA / PyTorch
    13. Deploy DCGM Exporter
    14. Integrate with Prometheus
    15. Integrate with Grafana
    16. Configure AlertManager alerts
    17. Conduct fault drills
    18. Output acceptance checklist
    19. Organize Runbook
    20. Establish production baseline

When troubleshooting, adhere to layered approach:

    lspci not normal:
        Check hardware and BIOS.

    nvidia-smi not normal:
        Check driver and kernel modules.

    Node has no nvidia.com/gpu:
        Check Device Plugin / GPU Operator.

    Pod Pending:
        Check Scheduler, resources, Taint, Label, Quota.

    Pod Running but no GPU:
        Check NVIDIA Container Toolkit, containerd, device mounting.

    CUDA / PyTorch anomalies:
        Check Runtime, framework version, driver compatibility.

    GPU metrics missing:
        Check DCGM Exporter, Prometheus, ServiceMonitor.

    GPU alert anomalies:
        Check PrometheusRule, AlertManager, notification channels.

The final state of experimental environment should be:

    GPU node is deliverable
    GPU resources are schedulable
    GPU Pod can run
    GPU metrics are observable
    GPU faults can be simulated
    GPU alerts are reachable
    GPU issues can be reviewed
    GPU operations are standardized

Only after completing this closed-loop process can one truly possess the practical skills for Kubernetes GPU operations.

---

## Reference Documents

- Kubernetes GPU Scheduling:
  https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/

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
  https://prometheus.io/docs/alerting/latest/alertmanager/

- Grafana Documentation:
  https://grafana.com/docs/

- kube-state-metrics:
  https://github.com/kubernetes/kube-state-metrics

- Helm Documentation:
  https://helm.sh/docs/