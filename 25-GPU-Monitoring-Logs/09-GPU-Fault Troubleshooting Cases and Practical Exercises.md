# 09-GPU-Fault Troubleshooting Cases and Practical Exercises

## Document Description

This document aims to compile common troubleshooting methods for Kubernetes GPU clusters, covering scenarios such as unavailable GPU nodes, unregistered GPU resources, pending GPU Pods, inability to use GPUs within containers, CUDA out-of-memory errors, abnormal GPU utilization rates, XID errors, video memory issues, temperature and power consumption anomalies, Device Plugin failures, GPU Operator problems, and DCGM Exporter errors.

The focus of this document is not on listing commands but on establishing a methodology for GPU troubleshooting:

    Phenomenon Identification
      ↓
    Fault Layering
      ↓
    Key Commands
      ↓
    Root Cause Determination
      ↓
    Temporary Solutions
      ↓
    Permanent Fixtures
      ↓
    Post-Troubleshooting Analysis

This document is recommended to be read after completing the following chapters:

- 03-NVIDIA-Driver Installation and Verification
- 04-CUDA-Installation and Testing
- 05-K8S-GPU-Resource Concepts and Scheduling Principles
- 06-NVIDIA-Device-Plugin- and-Operator-Installation
- 07-GPU-Pod-Deployment and Scheduling Practical Exercises
- 08-GPU-Monitoring and Alert Integration

---

## Tags

#Kubernetes #GPU #NVIDIA #CUDA #DevicePlugin #GPUOperator #DCGM #XID #Prometheus #SRE #Fault Troubleshooting #Ops Practice

---

## Recommended Reading Path

Recommended path:

    06-GPU and AI Infrastructure/03-GPU Monitoring and Troubleshooting/09-GPU-Fault Troubleshooting Cases and Practical Exercises.md

---

## I. Core Concepts of GPU Fault Troubleshooting

When dealing with GPU issues, one should not immediately assume it's a driver problem or simply conclude that "not enough GPUs are available" just because a Pod is pending.

GPU faults often involve multiple layers:

    Hardware Layer
      ↓
    BIOS/PCIe Layer
      ↓
    Linux Kernel Layer
      ↓
    NVIDIA Driver Layer
      ↓
    CUDA/Runtime Layer
      ↓
    Container Runtime Layer
      ↓
    Kubernetes Device Plugin Layer
      ↓
    Scheduler Scheduling Layer
      ↓
    Pod/Container Layer
      ↓
    AI Framework/Business Application Layer
      ↓
    Prometheus/Loki/Logging/Alerting Layer

When troubleshooting, it is essential to first determine which layer the issue lies in.

Different symptoms correspond to different layers:

    Unable to see GPU via `lspci`:
        Hardware, BIOS, PCIe, power supply, riser, Above 4G Decoding issues

    `nvidia-smi` fails:
        NVIDIA Driver, kernel module, Secure Boot, nouveau, DKMS, XID problems

    Node does not show `nvidia.com/gpu`:
        Device Plugin, GPU Operator, kubelet, GPU health status issues

    GPU Pod is pending:
        Scheduler, insufficient GPU resources, taints, labels, quotas, CPU/memory constraints

    Pod is running but no GPU inside the container:
        NVIDIA Container Toolkit, containerd, device mounting, Pod resource declarations

    `nvidia-smi` runs normally inside the container but the framework fails:
        CUDA Runtime, PyTorch/TensorFlow, cuDNN, driver compatibility issues

    CUDA out of memory:
        GPU video memory, model size, batch size, concurrency, multi-processing, MIG/sharing strategies

    Low GPU utilization:
        Data loading speed, CPU usage, network latency, storage performance, business traffic, application logic

    XID errors:
        Application, driver, hardware, PCIe, power supply, temperature, video memory, GPU health issues

---

## II. General GPU Fault Troubleshooting Process

### 2.1 Phase 1: Determine the Impact Scope

First, answer these questions:

    Is it a single Pod issue?
    Is it a single Namespace issue?
    Is it a single GPU node issue?
    Is it a specific GPU model issue?
    Is it an entire GPU cluster issue?
    Is it a scheduling issue?
    Is it a runtime issue?
    Is it a business performance issue?

Common commands:

    `kubectl get pod -A -o wide`
    `kubectl get nodes -o wide`
    `kubectl get events -A --sort-by=.lastTimestamp`
    `kubectl get pods -A | grep -i nvidia`
    `kubectl get ds -A | grep -i nvidia`

If only one business Pod is affected, start by checking the Pod and business logs.

If multiple GPU Pods on the same node are problematic, focus on the GPU node and runtime components.

If all GPU Pods cannot be scheduled, examine the Device Plugin, GPU Operator, Node resources, and Scheduler events first.

### 2.2 Phase 2: Verify if the GPU| `nvidia-smi` Stuck | Driver/Hardware | XID, GPU Dropped, PCIe AER, Hardware Failure |
| Node Lacks `nvidia.com/gpu` | Device Plugin/kubelet | Plugin Not Running, Registration Failed, GPU Marked as Unhealthy |
| GPU Pod Pending | Scheduler | Insufficient GPUs, Taints Unacceptable, Tag Mismatch, Quota Issues, Insufficient CPU/Memory |
| Pod Running but No GPU | Runtime/Container | Toolkit, containerd, Device Mounting Issues, Pod Did Not Request a GPU |
| PyTorch Fails to Recognize GPU | Application/CUDA | Incompatible CPU Version of Framework, CUDA Runtime, Outdated Driver |
| CUDA Out of Memory | Video Memory/Application | Excessive Batch Size, Large Model, High Concurrency, Video Memory Leak |
| Low GPU Utilization | Application/Data Link | CPU Bottleneck, Slow Data Loading, Off-peak Hours, Program Not Using the GPU |
| High GPU Temperature | Hardware/Distribution | Data Center Temperature, Fan Strategies, Air Ducts, Full Load, Dust |
| XID Errors | Driver/Hardware/Application | Application Unauthorized Access, Driver Bugs, GPU Hardware Issues, PCIe, Power Supply, Temperature Problems |
| DCGM Lack of Metrics | Monitoring Link | Exporter Malfunctions, ServiceMonitor Errors, Prometheus Data Not Captured |
| GPU Operator Validator Failure | Operator Components | Issues with Driver, Toolkit, Plugin, or Any Part of the DCGM Process |

---

## Case 1: GPU Not Visible in lspci on a GPU Node

### 4.1 Symptoms

When executed on the GPU node:

    lspci | grep -i nvidia

No output is displayed.

At the same time:

    nvidia-smi

Is usually also unusable.

### 4.2 Preliminary Diagnosis

This indicates a fundamental hardware recognition issue with the GPU.

Do not prioritize reinstalling the driver at this stage, because if Linux cannot even enumerate the PCIe device, the driver layer likely has not had a chance to function properly.

### 4.3 Possible Causes

Common reasons include:

- The GPU is not properly inserted;
- The GPU power cable is disconnected;
- Insufficient power supply capacity;
- Damaged PCIe slot;
- Abnormal riser card;
- The server does not support the GPU;
- The PCIe slot is disabled in the BIOS;
- Above 4G Decoding is not enabled;
- Incorrect PCIe Bifurcation configuration;
- Outdated motherboard BIOS version;
- Hardware failure with the GPU.

### 4.4 Troubleshooting Commands

Check for PCIe devices:

    lspci
    lspci -tv

View kernel PCIe logs:

    dmesg | grep -i pci
    dmesg | grep -i "bar"
    dmesg | grep -i "aer"
    journalctl -k | grep -i pci

Obtain hardware information:

    dmidecode -t system
    dmidecode -t bios
    dmidecode -t baseboard

Check BMC events:

    ipmitool sel list
    ipmitool sensor

### 4.5 Resolution Steps

Follow this sequence to troubleshoot:

1. Verify through the BMC/BIOS whether the server recognizes the GPU.
2. Check if the GPU is securely inserted.
3. Inspect the GPU power cable.
4. Examine the riser card.
5. Try switching the PCIe slot.
6. Enable Above 4G Decoding if applicable.
7. Verify the PCIe Bifurcation configuration.
8. Update the BIOS/BMC to the latest version.
9. Perform a cross-test with a different GPU slot.
10. Contact the server manufacturer or the GPU manufacturer for further assistance.

### 4.6 Key Points to Record

Document the following information:

- Node model.
- GPU model and model number.
- PCIe slot location.
- BIOS version.
- Whether Above 4G Decoding is enabled.
- Riser card model.
- Power supply specifications.
- Serial number of the faulty GPU.
- Final resolution steps taken.

---

## Case 2: GPU Visible in lspci but nvidia-smi Fails

### 5.1 Symptoms

When executed:

    lspci | grep -i nvidia

NVIDIA devices are visible.

However, when trying to run:

    nvidia-smi

An error occurs:

    NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.

### 5.2 Preliminary Diagnosis

The hardware at the PCIe level appears to be functional, so the issue is likely related to the NVIDIA Driver or Linux Kernel module layer.

### 5.3 Possible Causes

Common reasons include:

- The NVIDIA Driver is not installed.
- Failed installation of the driver.
- The kernel module has not been loaded successfully.
- Secure Boot is preventing the module from loading.
- Conflicts with the nouveau driver.
- Lack of3. View Device Plugin logs
4. Confirm GPU node drivers and device files
5. Check the GPU Operator component
6. Restart the Device Plugin Pod
7. Restart kubelet if necessary
8. Recheck Node Capacity / Allocatable

To restart the Device Plugin Pod:

    kubectl delete pod <device-plugin-pod> -n <namespace>

To restart kubelet:

    systemctl restart kubelet

### 6.6 Key Points for Review

Record:

    Device Plugin version
    GPU Operator version
    Driver version
    kubelet logs
    Whether "nvidia.com/gpu" is displayed on the node
    Output of "Node describe" before and after fixes

---

## Section Seven: Case Four: GPU Pod Remains Pending

### 7.1 Observation

After creating a GPU Pod:

    kubectl get pod <pod-name> -n <namespace> -o wide

The status remains:

    Pending

### 7.2 Preliminary Analysis

The Pod has not been successfully scheduled to a node.

This is a problem at the Scheduler level, not a container runtime issue.

### 7.3 Check Events

    kubectl describe pod <pod-name> -n <namespace>

Focus on the Events section.

### 7.4 Common Event One: Insufficient "nvidia.com/gpu"

Example event:

    0/3 nodes are available: insufficient nvidia.com/gpu

Possible causes:

- The node has not registered a GPU;
- The GPU is already occupied by another Pod;
- The Pod requested more GPUs than the node can provide;
- Issues with the Device Plugin;
- The GPU is marked as unhealthy;
- Namespace quota limitations.

Troubleshooting steps:

    kubectl describe node <gpu-node-name>
    kubectl get pods -A -o wide | grep <gpu-node-name>
    kubectl describe nodes | grep -A10 -B5 "nvidia.com/gpu"
    kubectl describe resourcequota -n <namespace>

Actions to take:

- Release any unused GPU Pods;
- Add more GPU nodes if needed;
- Adjust the number of GPUs requested by the Pod;
- Repair any issues with the Device Plugin;
- Adjust ResourceQuotas if necessary;
- Check if the GPU is in good working order.

### 7.5 Common Event Two: Untolerated Taints

Example event:

    node(s) had untolerated taint {nvidia.com/gpu: true}

This indicates that the GPU node has a taint, but the Pod does not have the necessary tolerance setting.

To check node taints:

    kubectl describe node <gpu-node-name> | grep -i taints -A5

To add a tolerance for the GPU:

    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"

### 7.6 Common Event Three: Node Selector Mismatch

Example event:

    node(s) didn't match Pod's node affinity/selector

Check the Pod configuration:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A20 nodeSelector
    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A40 affinity

Also, check node labels:

    kubectl get node <gpu-node-name> --show-labels

Actions to take:

- Correct the nodeSelector or nodeAffinity settings;
- Add appropriate labels to the node;
- Ensure the GPU model matches the business requirements.

### 7.7 Common Event Four: Insufficient CPU or Memory

Example events:

    insufficient cpu
    insufficient memory

This indicates that the issue is not with the availability of GPUs.

Actions to take:

- Reduce CPU/memory requests in the Pod configuration;
- Clean up resources on the node;
- Adjust the node pool configuration;
- Use nodes with higher specifications;
- Split tasks if possible.

### 7.8 Key Points for Review

Record:

    Pod YAML configuration
    Pod event details
    Node capacity and allocation status
    Node taints and labels
    Namespace quota settings
- The specific reason why scheduling failed.python3 -c "import torch; print('torch:', torch.__version__); print('cuda:', torch.version.cuda); print('available:', torch.cuda.is_available()); print('count:', torch.cuda.device_count())"

Judgment:

If `torch.version cuda` is `None`, it is very likely a CPU version of PyTorch.

If `torch-version cuda` has a value, but `available` is `False`, there may be issues with driver compatibility, CUDA Runtime, or device visibility.```markdown
nvidia-smi -q

BMC:

    ipmitool sensor
    ipmitool sel list

System:

    journalctl -k | grep -i thermal
    dmesg | grep -i thermal

Kubernetes:

    kubectl get pod -A -o wide | grep <gpu-node-name>

Prometheus:

    DCGM_FI_DEV_GPU_TEMP
    DCGM_FI_DEV_POWER_usage
    DCGM_FIDEV_THERMAL_VIOLATION

### 13.4 Common Causes

- GPU is under full load for an extended period;
- High temperature in the data center;
- Incorrect fan strategy;
- Blocked air ducts;
- Abnormal air intake for the server;
- Inappropriate installation environment for passively cooled GPUs;
- Insufficient spacing between GPUs;
- Dust accumulation;
- Fan failures;
- Hotspots in the cabinet where the node is located.

### 13.5 Solutions

Temporary fixes:

    1. Move low-priority GPU Pods;
    2. Reduce task concurrency;
    3. Isolate the node with a cordon;
    4. Pause training tasks;
    5. Notify the data center to check the temperature.

Long-term solutions:

    1. Adjust the fan strategy;
    2. Optimize the air ducts in the cabinet;
    3. Clean up dust;
    4. Check the server's cooling configuration;
    5. Adjust the position of nodes in the rack;
    6. Improve task scheduling;
    7. Implement additional high-temperature alerts and automated handling processes.

### 13.6 Key Points for Review

Record the following information:

    - Time of high temperature occurrence;
    - GPU model;
    - Maximum temperature reached;
    - Pods running at that time;
    - Business load;
    - Data center temperature;
    Fan status;
    Whether there was any frequency reduction;
    Whether any XID errors occurred;
    Final actions taken.

---
## Chapter Fourteen: Case Eleven: GPU XID Errors

### 14.1 Symptoms

Monitoring will show:

    DCGM_FI_DEV_XID_ERRORS > 0

Node logs may contain:

    dmesg | grep -i xid

Possible messages include:

    NVRM: Xid

### 14.2 What is an XID?

An XID is a GPU error message reported by the NVIDIA driver.

It is recorded in either the system kernel logs or event logs.

An XID could indicate:

- Issues with user applications;
- Driver problems;
- Abnormalities in CUDA programs;
- Hardware faults with the GPU;
- Memory errors;
    PCIe-related issues;
    Temperature problems;
    Power supply issues;
    GPU disconnection from the bus.

It is not appropriate to immediately assume that the GPU is damaged just because an XID error appears. A comprehensive analysis considering the type of XID, its frequency of occurrence, business logs, temperature readings, power consumption, PCIe status, and node conditions is necessary.

### 14.3 Diagnostic Commands

To check for XID errors:

    dmesg | grep -i xid
    journalctl -k | grep -i xid
    journalctl -k | grep -i nvrm

To inspect the GPU:

    nvidia-smi
    nvidia-smi -q
    nvidia-smi -L

To check temperature and power consumption:

    nvidia-smi -q | grep -i temperature -A20
    nvidia-smi -q | grep -i power -A20

To examine PCIe status:

    dmesg | grep -i aer
    dmesg | grep -i pci

To view the currently running Pods:

    kubectl get pod -A -o wide | grep <gpu-node-name>

To review business logs:

    kubectl logs <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace> --previous
```

### 14.4 Assessing the Severity of XID Errors

Pay attention to the following situations:

- Single, occasional occurrences:
    These can be recorded and observed.

- Short-term bursts:
    They may indicate driver crashes, PCIe issues, hardware problems, or application anomalies.

- Repeated occurrences with the same GPU:
    This suggests potential hardware health issues that require closer investigation.

- Occurrences alongside high temperatures:
    In this case, focus on cooling and power consumption issues first.

- Occurrences related to specific Pods:
    Investigate business programs, CUDA frameworks, or related components.

- If a GPU is detected as "fallen off the bus":
    This indicates a high-risk situation; isolate the node immediately.

### 14.5 Temporary Solutions

If the issue affects business operations:

    kubectl cordon <gpu-node-name>

Migrate movable tasks from this node:

    kubectl- GPU Feature Discovery.

### 17.3 Troubleshooting Commands

    kubectl logs <validator-pod> -n gpu-operator
    kubectl describe pod <validator-pod> -n gpu-operator
    kubectl get pods -n gpu-operator -o wide
    kubectl get ds -n gpu-operator
    kubectl describe clusterpolicy

### 17.4 Resolution Methods

Do not simply restart the validator.

Identify the specific component based on the validator logs.

For example:

    Driver verification failed:
        Check nvidia-driver-daemonset and the node's nvidia-smi.

    Toolkit verification failed:
        Check nvidia-container-toolkit-daemonset and containerd configuration.

    Plugin verification failed:
        Check nvidia-device-plugin-daemonset and the Node's nvidia.com/gpu.

    DCGM verification failed:
        Check dcgm-exporter and /metrics.

---

## Case 15: DCGM Exporter Not Reporting Metrics

### 18.1 Issue

The Grafana GPU dashboard shows no data.

Prometheus queries return:

    DCGM_FI_DEV_GPU_UTIL

or the DCGM Exporter target in Prometheus is marked as DOWN.

### 18.2 Troubleshooting Commands

Check Pods:

    kubectl get pods -A | grep -i dcgm

Check Services:

    kubectl get svc -A | grep -i dcgm

Review logs:

    kubectl logs <dcgm-exporter-pod> -n <namespace>

Test metrics:

    kubectl port-forward -n <namespace> <dcgm-exporter-pod> 9400:9400
    curl http://127.0.0.1:9400/metrics

Check Prometheus Targets:

    Status -> Targets

### 18.3 Common Causes

- The DCGM Exporter Pod is not running;
- The DCGM Exporter cannot access the GPU;
- ServiceMonitor selector error;
- Service port name mismatch;
- Prometheus lacks permissions to collect data;
- NetworkPolicy restrictions;
- Namespace selector issue;
- Metric names do not match Dashboard configurations;
- The GPU Operator has not enabled the DCGM Exporter.

### 18.4 Resolution Steps

- Fix up the DCGM Exporter Pod;
- Adjust the Service configuration;
- Repair the ServiceMonitor setting;
- Verify Prometheus RBAC permissions;
- Check Target settings;
- Modify Grafana PromQL queries based on actual metrics;
- Ensure the DCGM Exporter is deployed on GPU-enabled nodes.

---

## Case 16: GPU Memory Not Released After Pod Deletion

### 19.1 Issue

After deleting a Pod:

    kubectl delete pod <pod-name> -n <namespace>

The memory usage on the node still shows as occupied by nvidia-smi.

### 19.2 Preliminary Assessment

Possible causes include residual processes, unfinished containers, runtime errors, or delayed display by nvidia-smi.

### 19.3 Troubleshooting Commands

Check GPU processes:

    nvidia-smi

Check host process:

    ps -ef | grep <PID>

Inspect containers:

    crictl ps -a
    crictl inspect <container-id>

Monitor kubelet/containerd status:

    systemctl status kubelet
    systemctl status containerd
    journalctl -u kubelet -f
    journalctl -u containerd -f

### 19.4 Resolution Steps

- Wait briefly to see if memory is freed;
- Terminate any remaining abnormal containers;
- End residual processes;
- Restart containerd;
    Restart kubelet;
    In severe cases, isolate the node and restart it.

Note:

    Do not arbitrarily terminate processes without confirming their IDs.
    Ensure the terminated processes are not part of active business Pods.

---

## Case 17: Multiple GPU Pods Cannot Be Scheduled

### 20.1 Issue

A Pod requests 2 GPUs:

    nvidia.com/gpu: 2

But it remains in a Pending state.

### 20.2 Preliminary Assessment

Kubernetes requires that the same GPUs requested by a Pod must be located on the same node.

If there are 2 available GPUs across two nodes, the Pod still cannot be scheduled.

### 20.3 Troubleshooting Commands

Check the Pod:

    kubectl describe pod <pod-name> -n <namespace>

Inspect nodes:

    kubectl describe nodes | grep -A10 -B5 "nvidia.com/gpu"

View Pods running on each node:

    kubectl get pod -A -o wide | grep <gpu-node-name>

### 20.4 Common Causes

- A single node does not have 2 available GPUs;
- GPUs are already allocated to other Pods;
- The nodeSelector limits accessible nodes;
    Taint/toleration settings do not match;
    Insufficient CPU/memory```markdown
nvidia-smi topo -m
lsmod | grep nvidia
ls -l /dev/nvidia*

Logs:

dmesg | grep -i nvidia
dmesg | grep -i nvrm
dmesg | grep -i xid
dmesg | grep -i aer
journalctl -k | grep -i nvidia
journalctl -k | grep -i xid

Runtime:

nvidia-container-cli info
containerd config dump | grep -i nvidia -A30 -B10
crictl ps -a
crictl inspect <container-id>

System:

systemctl status kubelet
systemctl status containerd
journalctl -u kubelet -f
journalctl -u containerd -f

BMC:

ipmitool sensor
ipmitool sel list

---

## Section 23: GPU Fault Emergency Response Measures

### 23.1 Cordon a Node

Prevent new Pods from being scheduled:

kubectl cordon <gpu-node-name>

### 23.2 Drain a Node

Migrate movable Pods:

kubectl drain <gpu-node-name> --ignore-daemonsets --delete-emptydir-data

Note:

- Drain will evict Pods.
- Training tasks may lose progress if there are no checkpoints.
- Stateful services require careful handling.

### 23.3 Restore Scheduling

kubectl uncordon <gpu-node-name>

### 23.4 Delete Abnormal Pods

kubectl delete pod <pod-name> -n <namespace>

### 23.5 Restart the Device Plugin

kubectl delete pod <device-plugin-pod> -n <namespace>

### 23.6 Restart Node Components

systemctl restart containerd
systemctl restart kubelet

### 23.7 Restart the Node

Consider restarting in the following scenarios:

- GPU card failure;
- nvidia-smi freezing;
- Duplicate XID values;
- Unrecoverable issues with containerd/kubelet;
- Residual GPU memory that cannot be cleared;
- Abnormal driver status.

Before restarting in production, ensure to:

1. Cordon the node.
2. Drain movable services.
3. Notify relevant teams.
4. Save all logs.
5. Verify if there is a maintenance window available.

---

## Section 24: GPU Fault Review Template

It is recommended to document each GPU-related production fault using this template.

| Fault Number | Fault Time | Recovery Time | Impact Scope | Affected Services | Affected Nodes | GPU Model | GPU UUID | Kubernetes Namespace | Pod Name | Description of Issue | Alarm Name | First Detection Method | Key Logs | Key Metrics | XID Number | High Temperature? | Abnormal Power Consumption? | PCIe Issues? | Driver Issues? | Device Plugin Issues? | Business Code Issues? | Emergency Measures Taken | Root Cause Analysis | Final Fix Steps | Need for Hardware Replacement? | Need for Driver Upgrade? | Need for Image Repair? | Need for Scheduling Policy Changes? | Need for Additional Alarms? | Need to Update Runbook? | Responsible Person | Review Date |
---

## Section 25: Production-Environment GPU Troubleshooting Recommendations

### 25.1 Establish Node Baselines

Each GPU node should have the following information saved:

lspci | grep -i nvidia
nvidia-smi
nvidia-smi -q
nvidia-smi -L
nvidia-smi topo -m
lspci -tv
lscpu
dmidecode -t bios
ipmitool sensor

### 25.2 Maintain a Version Matrix

Record the following versions:

OS Version
Kernel Version
NVIDIA Driver Version
CUDA Runtime Version
Container Runtime Version
NVIDIA Container Toolkit Version
Device Plugin Version
GPU Operator Version
DCGM Exporter Version
PyTorch/TensorFlow Version

### 25.3 Set Up Monitoring Baselines

Monitor the following key metrics:

- GPU Utilization
- GPU Memory
- GPU Temperature
- GPU Power Consumption
- XID
- ECC
- Device Plugin Status
- DCGM Exporter Performance
- GPU Operator Health
- Number of Pending GPU Pods
- GPU Pod Restart Events
- Namespace-Level GPU Usage

### 25.4 Create Runbooks for Each Alarm

Prepare documentation for handling each type of alarm:

GPUHighTemperature
GPUXIDError
GPUMemoryUsageHigh
GPUPodPending
DevicePluginDown
DCGMExporterDown
GPUNodeNotReady

### 25.5 Establish Regular Drills

Recommend regular exercises for the following scenarios:

- Pending GPU Pods
- Device Plugin failures
- DCGM Exporter interruptions
- High GPU memory usage
- High temperatures
- Node cordon/drain operations
- Node restarts
- Business Pod migrations

---

## Section 26: Conclusion

The key to effective GPU troubleshooting is not memorizing individual commands,https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

- Kubernetes GPU Scheduling:
  https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/

- Kubernetes Device Plugins:
  https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/

- Kubernetes Taints and Tolerations:
  https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

- Kubernetes Resource Quotas:
  https://kubernetes.io/docs/concepts/policy/resource-quotas/

- Prometheus Documentation:
  https://prometheus.io/docs/

- Grafana Documentation:
  https://grafana.com/docs/