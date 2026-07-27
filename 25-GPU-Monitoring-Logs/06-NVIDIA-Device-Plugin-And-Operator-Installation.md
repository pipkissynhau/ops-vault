# 06-NVIDIA-Device-Plugin-And-Operator-Installation

## Document Description

This document provides guidance on the installation, selection, prerequisites for deployment, validation methods, troubleshooting of common issues, and recommendations for implementing NVIDIA Device Plugin and NVIDIA GPU Operator in Kubernetes clusters.

This document addresses the following key questions:

- Why is the NVIDIA Device Plugin necessary for Kubernetes?
- What problems do the Device Plugin and GPU Operator solve respectively?
- In what situations should only the Device Plugin be installed?
- Under what circumstances should the GPU Operator be used?
- Which components does the GPU Operator manage?
- How to configure the NVIDIA Container Toolkit in a containerd environment?
- If a GPU driver is already installed on a GPU node, is it still necessary to install the GPU Operator?
- After deploying the Device Plugin, how can `nvidia.com/gpu` be verified?
- After deploying the GPU Operator, which Pods, DaemonSets, and ClusterPolicies should be checked?
- After a Pod requests a GPU, how can `nvidia-smi` inside the container be verified?
- How to troubleshoot issues such as installation failures, nodes not displaying GPUs, Pods being in a Pending state, or no GPUs available in containers?
- What considerations should be taken in domestic network environments, private image repositories, and production clusters?

This document is suitable for reading after completing the following prerequisites:

- 01-GPU-Basic Concepts and Hardware Composition
- 02-GPU-BIOS and Hardware Tuning
- 03-NVIDIA-Driver Installation and Verification
- 04-CUDA-Installation and Testing
- 05-K8S-GPU-Resource Concepts and Scheduling Principles

This document does not repeat the details of NVIDIA driver installation and CUDA setup but focuses on GPU integration at the Kubernetes layer.

---

## Tags

#Kubernetes #GPU #NVIDIA #DevicePlugin #GPUOperator #Containerd #Helm #DCGM #CUDA #SRE #Operation and Maintenance Troubleshooting

---

## Recommended Reading Path

Recommended reading path:

    06-GPU and AI Infrastructure/02-Kubernetes-GPU Scheduling/06-NVIDIA-Device-Plugin-And-Operator-Installation.md

---

## I. Why Are the Device Plugin and GPU Operator Needed?

Kubernetes can inherently recognize built-in resources such as CPU, Memory, and Ephemeral Storage.

However, Kubernetes does not know how many NVIDIA GPUs are available on a node by default.

Even if running `nvidia-smi` on the host machine reveals the GPUs, it does not mean that the Kubernetes Scheduler can detect them.

To enable Kubernetes to schedule GPU resources, the kubelet must register the GPUs as extended resources in the Node Status.

This results in the following display:

    Capacity:
      nvidia.com/gpu: 1

    Allocatable:
      nvidia.com/gpu: 1

This step is typically accomplished by the NVIDIA Device Plugin.

The primary functions of the Device Plugin are:

- To detect NVIDIA GPUs on a node;
- To register these GPUs with kubelet;
- To include `nvidia.com/gpu` in the Node Status;
- To allow Pods to request GPU resources via `resources.limits.nvidia.com/gpu`.

However, a GPU node consists of more than just the Device Plugin.

A complete Kubernetes GPU node may also include:

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
- containerd/Docker configurations;
- Prometheus monitoring;
- Grafana dashboards;
- AlertManager alerts.

If only the Device Plugin is installed, many of these components would need to be managed manually.

Using the GPU Operator allows for unified management of all these NVIDIA-related components.

---

## II. Differences Between the Device Plugin and the GPU Operator

### 2.1 NVIDIA Device Plugin

The Device Plugin is a relatively lightweight solution.

Its main responsibilities include:

- Detecting GPUs;
- Registering GPUs with kubelet;
- Including `nvidia.com/gpu` in the Node Status;
- Collaborating with kubelet and the runtime to allocate GPUs to containers;
- Supporting basic GPU Pod scheduling;
- Enabling some advanced configurations, such as MIG and time-slicing, depending on the version and settings.

It typically runs as a DaemonSet.

The Device Plugin does not handle:

- Installing NVIDIA Drivers;
- Setting up CUDA;
- Configuring the NVIDIA Container Toolkit;
- Managing containerd;
- Managing the DCGM Exporter;
- Managing GPU node labels;
- Managing the entire lifecycle of MIG instances;
- Automatically repairing the GPU node environment;
- Providing unified management of the entire NVIDIA software stack.

### 2.2 NVIDIA GPU Operator

The GPU Operator represents a more comprehensive automated management solution.

It manages GPU node-related components using the Kubernetes Operator framework.

It can    systemctl status kubelet

Check the status of containerd:

    systemctl status containerd

Production recommendations:

Before deploying GPU components, the cluster itself must be stable.
If there are issues with CNI, CoreDNS, or kubelet itself, do not rush to deploy GPU components.

### 4.2 GPU Node Hardware Identification

Execute the following on the GPU node:

    lspci | grep -i nvidia

If no output is displayed, it means the system has not recognized the GPU.

Priority troubleshooting areas include:

- Whether the GPU is properly inserted;
- Whether the BIOS has recognized it;
- Issues with Above 4G Decoding;
- PCIe slots;
- Riser cards;
- Power supply;
- Server compatibility.

### 4.3 NVIDIA Driver Status

Run the following on the GPU node:

    nvidia-smi

If `nvidia-smi` is not functioning correctly, repair the driver first.

Check kernel modules:

    lsmod | grep nvidia

Check device files:

    ls -l /dev/nvidia*

Check XID:

    dmesg | grep -i xid
    journalctl -k | grep -i xid

### 4.4 containerd Status

Check the status of containerd:

    systemctl status containerd

Check the configuration:

    containerd config dump | grep -i nvidia -A20 -B5

If NVIDIA Runtime is not configured yet, containers may not be able to access the GPU later on.

### 4.5 Helm Tool

Both the GPU Operator and Device Plugin can be installed using Helm.

Check the Helm version:

    helm version

If Helm is not installed, it needs to be installed first.

Example for Ubuntu:

    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

Production environment recommendations:

Download Helm installation packages from trusted sources.
In enterprise intranet environments, it is recommended to maintain an internal Helm repository or artifact repository.

---

## Section 5: containerd and NVIDIA Container Toolkit

### 5.1 Why You Need the NVIDIA Container Toolkit

Just because the host's `nvidia-smi` is working properly does not mean that the driver layer is functioning correctly.

For containers to use the GPU, the following are also required:

- Mount `/dev/nvidia*`;
- Inject NVIDIA driver-related libraries;
- Set GPU visibility;
- Ensure that the container runtime supports the NVIDIA runtime;
- Coordinate with Device Plugins for device allocation.

These functionalities are provided by the NVIDIA Container Toolkit.

### 5.2 Installing the NVIDIA Container Toolkit

The actual installation commands should follow NVIDIA's official documentation.

Common steps include:

    1. Configure the NVIDIA Container Toolkit software repository
    2. Install nvidia-container-toolkit
    3. Use nvidia-ctk to configure containerd
    4. Restart containerd
    5. Verify whether containers can access the GPU

For Ubuntu/Debian:

    sudo apt-get update
    sudo apt-get install -y nvidia-container-toolkit

For Rocky/RHEL:

    sudo dnf install -y nvidia-container-toolkit

### 5.3 Configuring containerd

It is recommended to use the following command in a containerd environment:

    sudo nvidia-ctk runtime configure --runtime=containerd

Restart containerd:

    sudo systemctl restart containerd

Also, restart kubelet:

    sudo systemctl restart kubelet

Check the configuration:

    containerd config dump | grep -i nvidia -A30 -B10

### 5.4 Verifying the NVIDIA Container Toolkit

Use the following command to check the toolkit:

    nvidia-container-cli info

If testing with Docker, run the following command:

    docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

If using Kubernetes with containerd, Docker may not be installed.

In a containerd environment, it is more common to verify by creating a Kubernetes GPU Pod.

### 5.5 Production Considerations

If using the GPU Operator to manage the toolkit, make sure to set:

    toolkit.enabled=true

If the NVIDIA Container Toolkit has already been manually configured on the node, you can disable toolkit management during the Operator installation:

    --set toolkit.enabled=false

Whether to disable it depends on the enterprise's operations and maintenance policies.

Do not allow manual configurations and Operator automatic configurations to override each other simultaneously.

---

## Section 6: Manual Installation of the NVIDIA Device Plugin

### 6.1 Prerequisites

Before using the Device Plugin approach, ensure that the node meets the following requirements:

    [ ] The GPU hardware is functioning properly.
    [ ] NVIDIA GPUs can be detected using lspci.
    [ ] The NVIDIA driver has been installed.
    [ ] `nvidia-smi` is working correctly.
    [ ] The NVIDIA Container Toolkit has been installed.
    [ ]```markdown
kubectl get pod gpu-test -o wide
kubectl logs gpu-test
kubectl describe pod gpu-test

Expected logs to contain `nvidia-smi` output.

### 6.8 Uninstalling the Device Plugin

If installed via Helm:

    helm uninstall nvidia-device-plugin -n nvidia-device-plugin

Delete the Namespace:

    kubectl delete namespace nvidia-device-plugin

Note:

- After uninstalling the Device Plugin, the `nvidia.com/gpu` resources on the Node will be removed.
- Running GPU Pods may be affected.
- In a production environment, GPU services should be shut down before uninstallation.

---

## Section 7: Installation of the NVIDIA GPU Operator

### 7.1 Prerequisites for Using the GPU Operator

The GPU Operator is suitable for more comprehensive management of GPU nodes.

Before installation, confirm:

    [ ] The Kubernetes cluster is functioning normally.
    [ ] GPU nodes have been added to the cluster.
    [ ] GPU nodes have access to the image repository.
    [ ] Helm is available.
    [ ] Network policies allow communication between relevant components.
    [ ] It has been confirmed whether the Driver and Toolkit are managed by the Operator.
    [ ] The containerd/Docker/CRI-O runtime is confirmed to be installed.
    [ ] The Kubernetes version is compatible with the Operator version.
    [ ] It has been determined whether MIG or DCGM Exporter is required.

### 7.2 Adding the NVIDIA Helm Repository

    helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
    helm repo update

Check the versions:

    helm search repo nvidia/gpu-operator --versions

Production recommendations:

- In a production environment, specify a version of the chart.
- It is not recommended to install the latest version without specifying a version number.
- Always verify in a test environment before upgrading.

### 7.3 Creating a Namespace

    kubectl create namespace gpu-operator

### 7.4 Standard Installation of the GPU Operator

Basic installation example:

    helm install gpu-operator nvidia/gpu-operator \
      --namespace gpu-operator \
      --version <CHART_VERSION> \
      --wait

Explanation:

- `<CHART_VERSION>` should be selected based on the output of `helm search repo`.
- `--wait` will wait for the resources to be ready. In a production environment, it is not recommended to omit this option.

### 7.5 Scenarios Where Drivers Have Been Manually Installed on Nodes

If drivers have already been manually installed on nodes, it is generally not desired for the Operator to install them again.

Use the following command:

    helm install gpu-operator nvidia/gpu-operator \
      --namespace gpu-operator \
      --version <CHART_VERSION> \
      --set driver.enabled=false \
      --wait

This option is suitable for:

- Operations teams that maintain drivers themselves;
- Cases where driver versions are standardized;
- Private network environments where the Operator should not pull driver images;
- Nodes where drivers have been installed through Ansible, image templates, or manual processes.

### 7.6 Scenarios Where the NVIDIA Container Toolkit Has Been Manually Configured on Nodes

If the toolkit and containerd have already been manually configured on nodes, consider using the following command:

    helm install gpu-operator nvidia/gpu-operator \
      --namespace gpu-operator \
      --version <CHART_VERSION> \
      --set driver.enabled=false \
      --set toolkit.enabled=false \
      --wait

Note:

- Deciding to disable the toolkit requires careful consideration.
- If the toolkit is disabled and the node runtime is not correctly configured, GPU Pods may fail to access the GPU.
- A consistent policy must be enforced in production environments.

### 7.7 Checking Resources After Installing the GPU Operator

Check the Namespace:

    kubectl get all -n gpu-operator

Check Pods:

    kubectl get pods -n gpu-operator -o wide

Check DaemonSets:

    kubectl get ds -n gpu-operator

Check ClusterPolicy:

    kubectl get clusterpolicy

For detailed information:

    kubectl describe clusterpolicy

Resource names may vary slightly between versions, but typical entries include:

    gpu-operator
    nvidia-driver-daemonset
    nvidia-container-toolkit-daemonset
    nvidia-device-plugin-daemonset
    nvidia-dcgm-exporter
    nvidia-operator-validator
    gpu-feature-discovery
    node-feature-discovery

### 7.8 Checking Operator Logs

First, check the Pods:

    kubectl get pods -n gpu-operator

To view logs:

    kubectl logs <gpu-operator-pod> -n gpu-operator

To check related component logs:

    kubectl logs <nvidia-device-plugin-pod> -n gpu-operator
    kubectl logs <nvidia-container-toolkit-pod> -n gpu-operator
    kubectl logs <nvidia-dcgm-exporter-pod> -n gpu-operator
    k### Capacity:
      nvidia.com/gpu: <quantity>

    Allocatable:
      nvidia.com/gpu: <quantity>

### 9.3 Verifying Node Labels

To view labels:

    kubectl get node <gpu-node-name> --show-labels

You can filter for NVIDIA-related labels:

    kubectl get node <gpu-node-name> --show-labels | grep -i nvidia

Alternatively:

    kubectl describe node <gpu-node-name> | grep -i nvidia

### 9.4 Verifying GPU Test Pod

Create a test Pod:

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

Deploy it:

    kubectl apply -f gpu-test.yaml

To check it:

    kubectl get pod gpu-test -o wide
    kubectl logs gpu-test
    kubectl describe pod gpu-test

### 9.5 Verifying DCGM Exporter

View the Service:

    kubectl get svc -n gpu-operator

View the Pod:

    kubectl get pods -n gpu-operator | grep -i dcgm

For local port testing, use the appropriate Service and port:

    kubectl port-forward -n gpu-operator <dcgm-exporter-pod> 9400:9400

Then access it:

    curl http://127.0.0.1:9400/metrics

If you have a Prometheus Operator, you can configure ServiceMonitor later.

### 9.6 Verifying CUDA Inside Containers

If the test image only runs `nvidia-smi`, it will only verify basic GPU functionality.

You can use the devel image for more detailed checks:

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

Deploy it:

    kubectl apply -f cuda-devel-test.yaml

To check it:

    kubectl logs cuda-devel-test

Note:

- The base image may not include nvcc.
- The devel image usually includes nvcc.
- It is generally not recommended to use the devel image in production environments; runtime images should be more streamlined.

---

## Section X: Device Plugins and GPU Operator Should Not Be Installed Together to Avoid Conflicts

### 10.1 It Is Not Recommended to Deploy Two Device Plugins Duplicates

If you are already using a GPU Operator, it typically manages its own Device Plugin.

In this case, it is not advisable to manually deploy another NVIDIA Device Plugin.

Otherwise, you may encounter:

- Multiple Device Plugins on the same node competing for resources;
- Confused logs;
- Inconsistent versions;
- Issues with kubelet registration;
- Increased troubleshooting difficulties.

### 10.2 Checking for Duplicate Deployments

View all NVIDIA-related DaemonSets:

    kubectl get ds -A | grep -i nvidia

View all NVIDIA-related Pods:

    kubectl get pods -A | grep -i nvidia

If you see both:

    nvidia-device-plugin in kube-system
    nvidia-device-plugin in gpu-operator

you need to confirm whether there is a duplicate deployment.

### 10.3 Handling Recommendations

If using a GPU Operator:

    Uninstall any manually installed Device Plugins.

If using a manual Device Plugin:

    Do not install a GPU Operator, or disable related components when installing it.

In production environments, it is important to maintain a clear and consistent setup to avoid confusion and overlapping configurations.

---

## Section XI: Design of GPU Node Labels and Taints

After installing GPU components, it is recommended to plan the scheduling strategy for GPU nodes.

### 11.1 Labeling GPU Nodes

Examples:

    kubectl label node <gpu-node-name> node-role.kubernetes.io/gpu=true
    kubectl label node <gpu-node-name> accelerator=nvidia
    kubectl label node <gpu-node-name> gpu.vendor=nvidia

By model:

    kubectl label node <gpu-node-name> gpu.model=a100
    kubectl label node <gpu-node-name> gpu.memory=80gb

By purpose:

    kubectl label node <gpu-node-name> gpu.workload=inference

- Can Prometheus collect metrics at the MIG level?
- Can fault troubleshooting pinpoint specific MIG instances?

### 12.4 Examples of MIG Resources

Once MIG is enabled, resource names may look like this:

    nvidia.com/mig-1g.10gb
    nvidia.com/mig-2g.20gb
    nvidia.com/mig-3g.40gb

The specific name is determined by the GPU model, MIG segmentation, and Device Plugin configuration.

Example of Pod resource requirements:

    resources:
      limits:
        nvidia.com/mig-1g.10gb: 1

Note:

    Do not directly copy the MIG resource names.
    First, use `kubectl describe node` to check the actual resource names exposed by the node.

### 12.5 Viewing MIG Resources

    Use `kubectl describe node <gpu-node-name>` to view resources in the Capacity/Allocatable section:

        nvidia.com/mig-xxx

To view MIG details on a node, use:

    nvidia-smi -L
    nvidia-smi mig -lgi
    nvidia-smi mig -lci

---

## Chapter Thirteen: GPU Sharing: Introduction to Time-Slicing

### 13.1 Why GPU Sharing is Needed

By default, `nvidia.com/gpu: 1` requests a full GPU.

For lightweight inference, development testing, and small-model tasks, if each Pod uses an entire GPU, the utilization rate may be very low.

In such cases, GPU sharing can be considered.

Common solutions include:

- MIG;
- Time-Slicing;
- MPS;
- Third-party vGPU solutions.

### 13.2 Characteristics of Time-Slicing

Time-Slicing is a form of time-sharing for GPUs.

Key features include:

- Multiple Pods can share one GPU;
- Suitable for lightweight tasks;
- Can improve utilization rates;
- Lower isolation compared to MIG;
- Memory competition may still occur;
- An issue with one task could affect others;
- Not suitable for highly isolated production scenarios.

### 13.3 Production Recommendations

For strong isolation in production:

    Prefer MIG.

For development testing or lightweight inference:

    Consider Time-Slicing.

For serious training or core inference:

    Be cautious when sharing GPUs.

Do not mix all tasks on the same GPU just to improve utilization rates.

---

## Chapter Fourteen: Considerations for Domestic Networks and Private Image Repositories

### 14.1 Common Issues

When installing Device Plugins or GPU Operators, it may be necessary to pull images from remote repositories.

In domestic network environments, you might encounter:

- Slow image downloads;
- Image download failures;
- Unstable access to NGC;
- Unstable access to GitHub;
- Unstable access to Helm repos;
- Nodes unable to access the public internet;
- Corporate security policies prohibiting public network access.

### 14.2 Production Recommendations

Here are some suggestions:

    1. List all required images in advance.
    2. Pull them in a networked environment.
    3. Retag the images for use in your internal repository.
    4. Push them to Harbor, Nexus, or Alibaba Cloud image repositories.
    5. Modify Helm values to reference the internal repository.
    6. Fix the image tags consistently.
    7. Keep a record of image versions.

### 14.3 Viewing GPU Operator Images

You can check these using Helm values:

    `helm show values nvidia/gpu-operator --version <CHART_VERSION> > gpu-operator-values.yaml`

To search for specific images, use:

    `grep -i "repository" gpu-operator-values.yaml`
    `grep -i "image" gpu-operator-values.yaml`
    `grep -i "tag" gpu-operator-values.yaml`

You can also install the Operator in a test cluster first and then check the images:

    `kubectl get pods -n gpu-operator -o yaml | grep image:`

### 14.4 Modifying values.yaml

It is recommended to use values.yaml to manage configuration in production environments, rather than using multiple `--set` commands at the command line.

Example:

    `helm show values nvidia/gpu-operator --version <CHART_VERSION> > values-gpu-operator.yaml`

After editing, install it using:

    `helm install gpu-operator nvidia/gpu-operator \
      --namespace gpu-operator \
      --version <CHART_VERSION> \
      -f values-gpu-operator.yaml \
      --wait`

To upgrade, use:

    `helm upgrade gpu-operator nvidia/gpu-operator \
      --namespace gpu-operator \
      --version <CHART_VERSION> \
      -f values-gpu-operator.yaml \
      --wait`

### 14.5 Avoid Direct Dependence on the Public Internet

It is not recommended to rely heavily on public internet images in production clusters.

Reasons include:

- Potential```markdown
containerd config dump | grep -i nvidia -A20 -B5

---

## Section Seventeen: Common Issue One: Node Lacks nvidia.com/gpu

### 17.1 Observation

When executing:

    kubectl describe node <gpu-node-name>

the following is not displayed:

    nvidia.com/gpu

### 17.2 Possible Causes

Possible reasons include:

- The node does not have a GPU;
- The GPU is not detected by lspci;
- Issues with the NVIDIA Driver;
- Abnormalities with nvidia-smi;
- The Device Plugin is not running;
- The Device Plugin is running on the wrong node;
- Error logs from the Device Plugin;
- The GPU Operator component is not ready;
- Abnormal registration of kubelet plugins;
- Node taints or selectors preventing plugin scheduling;
- Issues with runtime configuration;
- The GPU is marked as unhealthy in health checks.

### 17.3 Troubleshooting Commands

On the node:

    lspci | grep -i nvidia
    nvidia-smi
    lsmod | grep nvidia
    ls -l /dev/nvidia*
    dmesg | grep -i xid

In the cluster:

    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia
    kubectl describe node <gpu-node-name>
    kubectl get events -A --sort-by=.lastTimestamp

To check Device Plugin logs:

    kubectl logs <device-plugin-pod> -n <namespace>

### 17.4 Solution Approach

Follow these steps in order:

    1. Verify at the hardware level with lspci;
    2. Check the driver layer using nvidia-smi;
    3. Confirm that the Device Plugin Pod is running;
    4. Ensure there are no error logs in the Device Plugin;
    5. Verify that kubelet is functioning normally;
    6. Restart the Device Plugin Pod if necessary;
    7. If needed, restart kubelet;
    8. Recheck Node Capacity and Allocatable resources.

---

## Section Eighteen: Common Issue Two: Device Plugin Pod Crashes in a Loop

### 18.1 Observation

When executing:

    kubectl get pods -A | grep -i nvidia

you will see the Device Plugin Pod with status:

    CrashLoopBackOff
    Error

### 18.2 Troubleshooting

Check the logs:

    kubectl logs <device-plugin-pod> -n <namespace>

Review the description:

    kubectl describe pod <device-plugin-pod> -n <namespace>

Inspect the node:

    kubectl get pod <device-plugin-pod> -n <namespace> -o wide

Go to the corresponding node and check:

    nvidia-smi
    ls -l /dev/nvidia*
    systemctl status kubelet
    systemctl status containerd

### 18.3 Possible Causes

Possible reasons include:

- Abnormalities with node drivers;
- The `/dev/nvidia*` directory does not exist;
- Permission issues;
- The container runtime cannot mount the device;
- Failed image pull;
- Incompatibility between Device Plugin version and node environment;
- Configuring an unsupported MIG policy;
- Attempting to run the plugin on a node without a GPU;
- SELinux / AppArmor / PodSecurity restrictions;
- Incorrect runtime configuration.

### 18.4 Solutions

Possible actions include:

- Repairing the NVIDIA Driver;
- Fixing the NVIDIA Container Toolkit;
- Checking the image;
- Reviewing Helm values;
- Verifying MIG configuration;
- Determining if the node should run the plugin;
- Checking Pod security policies;
- If necessary, deleting the Pod and rebuilding the DaemonSet with:

    kubectl delete pod <device-plugin-pod> -n <namespace>
```

---

## Section Nineteen: Common Issue Three: GPU Pods Remain Pending

### 19.1 Observation

GPU Pods remain in a pending state.

To check:

    kubectl get pod <pod-name> -n <namespace> -o wide

### 19.2 Troubleshooting Events

Check the events:

    kubectl describe pod <pod-name> -n <namespace>

Common error messages include:

    insufficient nvidia.com/gpu
    Node(s) had untolerated taint
    Node(s) didn't match Pod's node affinity/selector
    Exceeded quota
    Insufficient CPU
    Insufficient memory

### 19.3 Solution Approach

If the issue is "insufficient nvidia.com/gpu":

Check:

    kubectl describe node <gpu-node-name>
    kubectl get pods -A -o wide | grep <gpu-node-name>

If it's "untolerated taint":

Add a toleration for the Pod.

If it### 27.1 Pre-Installation Checks

    [ ] The Kubernetes cluster is functioning properly.
    [ ] GPU nodes are ready for use.
    [ ] NVIDIA GPUs can be detected using lspci.
    [ ] nvidia-smi is running correctly.
    [ ] containerd is operational.
    [ ] kubelet is functioning normally.[ ] Helm is available.
[ ] Decided whether to use a Device Plugin or a GPU Operator.
[ ] Determined whether the Driver will be managed by an Operator.
[ ] Decided whether the Toolkit will be managed by an Operator.
[ ] Confirmed that the image can be pulled.
[ ] Confirmed the version number.
[ ] Prepared values.yaml.
[ ] Prepared a rollback plan.

### 27.2 Installation in Progress

[ ] Namespace created successfully.
[ ] Helm repo is accessible.
[ ] Helm installation was successful.
[ ] Pods are pulling images normally.
[ ] DaemonSet is running normally.
[ ] No error logs from the Operator.
[ ] Device Plugin is running normally.
[ ] Validator passed.

### 27.3 After Installation

[ ] Node Capacity shows nvidia.com/gpu.
[ ] Node Allocatable shows nvidia.com/gpu.
[ ] GPU test Pod can run.
[ ] Logs of the GPU test Pod show nvidia-smi.
[ ] /dev/nvidia* exists in the container.
[ ] DCGM Exporter is working properly.
[ ] Prometheus can collect GPU metrics.
[ ] Grafana can display GPU metrics.
[ ] GPU node Labels are correct.
[ ] GPU node Taints are correct.
[ ] ResourceQuotas have been planned.
[ ] Business image tests passed.

---

## Chapter 28: Summary of Common Commands

### 28.1 View GPU Nodes

    kubectl get nodes -o wide
    kubectl describe node <gpu-node-name>
    kubectl get node <gpu-node-name> --show-labels

### 28.2 View NVIDIA Components

    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia
    kubectl get pods -n gpu-operator -o wide
    kubectl get ds -n gpu-operator

### 28.3 View the GPU Operator

    helm list -n gpu-operator
    helm get values gpu-operator -n gpu-operator
    kubectl get clusterpolicy
    kubectl describe clusterpolicy

### 28.4 View the Device Plugin

    kubectl get pods -A | grep -i device-plugin
    kubectl logs <device-plugin-pod> -n <namespace>

### 28.5 View GPU Resources

    kubectl describe node <gpu-node-name> | grep -A10 -B5 "nvidia.com/gpu"

### 28.6 Test the GPU Pod

    kubectl apply -f gpu-test.yaml
    kubectl get pod gpu-test -o wide
    kubectl logs gpu-test
    kubectl describe pod gpu-test

### 28.7 Local Check of Nodes

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

## Chapter 29: Troubleshooting Hierarchy for Device Plugin and Operator

| Phenomenon | Priority Layer for Troubleshooting | Common Causes |
|---|---|---|
| `lspci` does not show the GPU | Hardware/BIOS | GPU not properly inserted, power supply issues, PCIe connection problems, Above 4G Decoding issues |
| Failure of `nvidia-smi` | Driver layer | Driver issues, nouveau, Secure Boot settings, DKMS problems |
| Node does not display `nvidia.com/gpu` | Device Plugin/kubelet | Plugin not running, registration failure, abnormal GPU health |
| Device Plugin crashes in a loop | Plugin/node environment | Driver errors, missing device files, image issues |
| Failure to pull the GPU Operator Pod | Image/network | NGC connectivity issues, private repository authentication problems, DNS issues |
| Validator fails | Operator components | Issues with the Driver, Toolkit, Plugin, or any part of DCGM |
| GPU Pod remains in a pending state | Scheduler | Insufficient GPUs, Taints, Labels, Quotas, CPU/Memory constraints |
| Pod runs but does not have a GPU | Runtime/container | Issues with the Toolkit, containerd, image, or the Pod not requesting a GPU |
| DCGM does not provide metrics | Monitoring link | Exporter failures, ServiceMonitor issues, Prometheus configuration problems |
| MIG resources do not appear | MIG configuration | GPU not supported, incorrect policies, Operator not configured properly |

---

## Chapter 30: Recommended Implementation Paths for Production Environments

### 30.1 Learning and Experimental Path

Suitable for individual learning:

    1. Manually install the NVIDIA Driver.
    2. Manually install the NVIDIA Container Toolkitnvidia-smi is not functioning correctly:
    Check the driver.

The node does not have nvidia.com/gpu information:
    Verify the Device Plugin or GPU Operator settings.

Pods are pending deployment:
    Check the Scheduler, Labels, Taints, Quotas, and whether there are insufficient resources.

Pods are running but lack a GPU:
    Verify the Container Toolkit, containerd, image specifications, and Pod resource declarations.

Metrics are not available:
    Ensure that the DCGM Exporter, ServiceMonitor, and Prometheus are properly configured.

In a production environment, it is not recommended to simply connect GPUs to Kubernetes using a `kubectl apply` command. Implementing a proper GPU solution requires attention to the following aspects:

- Driver version;
- CUDA compatibility;
- containerd configuration;
- Device Plugin settings;
- GPU Operator functionality;
- Image repository management;
- Node labeling and tainting;
- Resource quota management;
- MIG support;
- GPU monitoring tools;
- Alerting mechanisms;
- Upgrade and rollback procedures;
- Cost and utilization optimization.

Only by addressing all these aspects can a GPU node be transformed from something that merely allows the host machine to recognize the GPU into a production-ready resource pool that Kubernetes can schedule, utilize for business purposes, monitor effectively, and troubleshoot efficiently.

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