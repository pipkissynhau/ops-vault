# 09-GPU-Fault Diagnosis Cases and Practical Applications

## Document Description

This document is used to organize common troubleshooting methods for GPU clusters in Kubernetes, including scenarios such as GPU nodes being unavailable, GPU resources not being registered, GPU Pods being in Pending state, inability to use GPU inside containers, CUDA OOM, abnormal GPU utilization, XID errors, memory anomalies, temperature/power anomalies, Device Plugin anomalies, GPU Operator anomalies, DCGM Exporter anomalies, etc.

The focus of this document is not on listing commands, but on establishing a methodology for GPU troubleshooting:

    Phenomenon Recognition
      ↓
    Fault Layering
      ↓
    Key Commands
      ↓
    Root Cause Judgment
      ↓
    Temporary Hemorrhage Control
      ↓
    Root Cause Repair
      ↓
    Post-Mortem Analysis

This document is suitable for study after completing the following chapters:

- 03-NVIDIA-Drive Installation and Verification
- 04-CUDA-Installation and Testing
- 05-K8S-GPU-Resource Concepts and Scheduling Principles
- 06-NVIDIA-Device-Plugin-and-Operator-Installation
- 07-GPU-Pod-Deployment and Scheduling Practical Applications
- 08-GPU-Monitoring and Alert Integration

---

## Tags

#Kubernetes #GPU #NVIDIA #CUDA #DevicePlugin #GPUOperator #DCGM #XID #Prometheus #SRE #FaultCheck. #It'sABattleOfLuck.

---

## Recommended Path

Recommended path:

    06-GPU-and-AI-Infrastructure/03-GPU-Monitoring-and-Problem-Solving/09-GPU-Fault Diagnosis Cases and Practical Applications.md

---

## One, Core Ideas of GPU Fault Diagnosis

GPU faults cannot be resolved by reinstalling drivers or directly judging "GPU insufficiency" when seeing Pod Pending. GPU faults typically span multiple layers:

    Hardware Layer
      ↓
    BIOS / PCIe Layer
      ↓
    Linux Kernel Layer
      ↓
    NVIDIA Driver Layer
      ↓
    CUDA / Runtime Layer
      ↓
    Container Runtime Layer
      ↓
    Kubernetes Device Plugin Layer
      ↓
    Scheduler Scheduling Layer
      ↓
    Pod / Container Layer
      ↓
    AI Framework / Business Application Layer
      ↓
    Prometheus / Loki / Log / Alert Layer

When troubleshooting, it is essential to first determine which layer the problem occurs in.

Different phenomena correspond to different layers:

    lspci cannot see GPU:
        Hardware, BIOS, PCIe, Power Supply, riser, Above 4G Decoding

    nvidia-smi fails:
        NVIDIA Driver, Kernel Module, Secure Boot, nouveau, DKMS, XID

    Node lacks nvidia.com/gpu:
        Device Plugin, GPU Operator, kubelet, GPU Health Status

    GPU Pod Pending:
        Scheduler, GPU Resource Insufficiency, Taint, Label, Quota, CPU/Memory

    Pod Running but no GPU inside container:
        NVIDIA Container Toolkit, containerd, Device Mounting, Pod Resource Declaration

    Container nvidia-smi normal but framework unavailable:
        CUDA Runtime, PyTorch/TensorFlow, cuDNN, Driver Compatibility

    CUDA out of memory:
        GPU Memory, Model, Batch Size, Concurrency, Multi-process, MIG/Shared Policy

    Low GPU Utilization:
        Data Loading, CPU, Network, Storage, Business Traffic, Application Logic

    XID Error:
        Application, Driver, Hardware, PCIe, Power Supply, Temperature, Memory, GPU Health

---

## Two, Total Flow of GPU Fault Diagnosis

### 2.1 First Stage: Confirming the Scope of Impact

First answer:

    Is it a single Pod anomaly?
    Is it a single Namespace anomaly?
    Is it a single GPU node anomaly?
    Is it an anomaly for a specific GPU model?
    Is it an anomaly for the entire GPU cluster?
    Is it a scheduling anomaly?
    Is it a runtime anomaly?
    Is it a business performance anomaly?

Common commands:

    kubectl get pod -A -o wide
    kubectl get nodes -o wide
    kubectl get events -A --sort-by=.lastTimestamp
    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia

If only one business Pod is abnormal, prioritize checking the Pod and business logs.

If multiple GPU Pods on the same node are abnormal, prioritize checking the GPU node and runtime.

If all GPU Pods cannot be scheduled, prioritize checking Device Plugin, GPU Operator, node resources, and Scheduler events.

### 2.2 Second Stage: Confirming if the Node GPU is Normal

Execute on the GPU node:

    lspci | grep -i nvidia
    nvidia-smi
    nvidia-smi -L
    nvidia-smi -q
    nvidia-smi topo -m
    lsmod | grep nvidia
    ls -l /dev/nvidia*
    dmesg | grep -i nvidia
    dmesg | grep -i xid
    journalctl -k | grep -i nvidia
    journalctl -k | grep -i xid

Judgment:

    lspci is abnormal:
        Check hardware and BIOS.

    lspci is normal, but nvidia-smi is abnormal:
        Check driver and kernel modules.

    nvidia-smi is normal, but K8S has no GPU:
        Check Device Plugin and kubelet.

    K8S has GPU, but Pod is abnormal:
        Check scheduling, runtime, and business container.

### 2.3 Third Stage: Confirming if Kubernetes Recognizes GPU

Check the node:

    kubectl describe node <gpu-node-name>

Focus on:

    Capacity:
      nvidia.com/gpu: <quantity>

    Allocatable:
      nvidia.com/gpu: <quantity>

Allocated resources:
  nvidia.com/gpu

If there is no `nvidia.com/gpu`, it indicates that the GPU has not been registered as a schedulable resource by Kubernetes.

Continue checking:

    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia
    kubectl logs <device-plugin-pod> -n <namespace>

If using GPU Operator:

    kubectl get pods -n gpu-operator -o wide
    kubectl get ds -n gpu-operator
    kubectl get clusterpolicy
    kubectl describe clusterpolicy

### 2.4 Fourth Stage: Confirm Pod Scheduling Events

Check the Pod:

    kubectl get pod <pod-name> -n <namespace> -o wide
    kubectl describe pod <pod-name> -n <namespace>

Focus on the Events section.

Common events:

    insufficient nvidia.com/gpu
    node(s) had untolerated taint
    node(s) didn't match Pod's node affinity/selector
    exceeded quota
    insufficient cpu
    insufficient memory
    ImagePullBackOff
    CreateContainerError
    RunContainerError

### 2.5 Fifth Stage: Confirm GPU Visibility Inside Container

Enter the container:

    kubectl exec -it <pod-name> -n <namespace> -- sh

Execute:

    nvidia-smi
    ls -l /dev/nvidia*
    echo $CUDA_VISIBLE_DEVICES
    echo $NVIDIA_VISIBLE_DEVICES
    echo $NVIDIA_DRIVER_CAPABILITIES

If the image does not contain `nvidia-smi`, do not directly conclude that the GPU is unavailable.

You can check:

    ls -l /dev/nvidia*
    python3 -c "import torch; print(torch.cuda.is_available())"

Or retest using the official CUDA image.

---

## Three, GPU Fault Layered Troubleshooting Table

| Fault Phenomenon | Priority Troubleshooting Layer | Common Causes |
|---|---|---|
| `lspci`I can't see it. GPU | Hardware / BIOS / PCIe | GPU not properly inserted, power supply, riser, Above 4G Decoding, PCIe slot |
| `nvidia-smi` Failed | Driver / Kernel | Driver not loaded, Secure Boot, nouveau, DKMS, kernel mismatch |
| `nvidia-smi` Stuck | Driver / Hardware | XID, GPU card failure, PCIe AER, hardware failure |
| Node has no `nvidia.com/gpu` | Device Plugin / kubelet | Plugin not running, registration failure, GPU marked as unhealthy |
| GPU Pod Pending | Scheduler | GPU insufficient, taints not tolerated, label mismatch, Quota, CPU/memory insufficient |
| Pod Running but no GPU | Runtime / Container | Toolkit, containerd, device mounting, Pod not requesting GPU |
| PyTorch cannot recognize GPU | Application / CUDA | CPU version framework, CUDA Runtime incompatibility, outdated driver |
| CUDA OOM | VRAM / Application | Batch size too large, model too big, high concurrency, VRAM leak |
| Low GPU utilization | Application / Data Pipeline | CPU bottleneck, slow data loading, low business peak, program not using GPU |
| High GPU temperature | Hardware / Cooling | Data center temperature, fan strategy, airflow, full load, dust |
| XID error | Driver / Hardware / Application | Application illegal access, driver bug, GPU hardware, PCIe, power supply, temperature |
| DCGM has no metrics | Monitoring pipeline | Exporter anomaly, ServiceMonitor error, Prometheus not scraping |
| GPU Operator Validator failed | Operator component | Driver, Toolkit, Plugin, DCGM any link abnormal |

---

## Four, Case Study One: GPU Node lspci shows no GPU

### 4.1 Phenomenon

Execute on the GPU node:

    lspci | grep -i nvidia

No output.

At the same time:

    nvidia-smi

Typically also cannot be used.

### 4.2 Initial Judgment

This is a fundamental hardware recognition issue with the GPU.

Do not prioritize reinstalling the driver.

Because Linux hasn't even enumerated the PCIe device, the driver layer typically hasn't had a chance to operate yet.

### 4.3 Possible Causes

Common causes:

- GPU not properly inserted;
- GPU power cable not connected;
- Insufficient power supply;
- Damaged PCIe slot;
- Riser card abnormal;
- Server does not support the GPU;
- PCIe slot disabled in BIOS;
- Above 4G Decoding not enabled;
- PCIe Bifurcation configuration error;
- Outdated motherboard BIOS version;
- GPU hardware failure.

### 4.4 Troubleshooting Commands

Check PCIe devices:

    lspci
    lspci -tv

Check kernel PCIe logs:

    dmesg | grep -i pci
    dmesg | grep -i "bar"
    dmesg | grep -i "aer"
    journalctl -k | grep -i pci

Check hardware information:

    dmidecode -t system
    dmidecode -t bios
    dmidecode -t baseboard

Check BMC events:

    ipmitool sel list
    ipmitool sensor

### 4.5 Handling Methods

Handling order: /think

1. Confirm via BMC/BIOS whether the server recognizes the GPU  
2. Check if the GPU is securely seated  
3. Check the GPU power supply cables  
4. Check the riser card  
5. Test by switching PCIe slots  
6. Enable Above 4G Decoding  
7. Check PCIe Bifurcation  
8. Update BIOS / BMC  
9. Single-card cross-testing  
10. Contact the server vendor or GPU vendor  

### 4.6 Review Key Points  

Need to record:  

    Node model  
    GPU model  
    Slot position  
    BIOS version  
    Whether Above 4G Decoding is enabled  
    Riser model  
    Power supply specifications  
    Faulty GPU serial number  
    Final remediation action  

---

## Five, Case Two: lspci can see GPU, but nvidia-smi fails  

### 5.1 Phenomenon  

Execute:  

    lspci | grep -i nvidia  

You can see NVIDIA devices.  

But executing:  

    nvidia-smi  

Returns an error:  

    NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.  

### 5.2 Initial Judgment  

The PCIe layer is basically visible, and the issue is likely in the NVIDIA Driver or Linux Kernel module layer.  

### 5.3 Possible Causes  

Common causes:  

- NVIDIA Driver not installed  
- Driver installation failed  
- Kernel module not loaded  
- Secure Boot prevents module loading  
- nouveau conflict  
- kernel headers missing  
- DKMS build failed  
- Driver became invalid after kernel upgrade  
- Current driver does not support GPU model  
- Driver version and kernel are incompatible  

### 5.4 Troubleshooting Commands  

Check driver modules:  

    lsmod | grep nvidia  

Check device files:  

    ls -l /dev/nvidia*  

Check Secure Boot:  

    mokutil --sb-state  

Check nouveau:  

    lsmod | grep nouveau  
    dmesg | grep -i nouveau  

Check kernel logs:  

    dmesg | grep -i nvidia  
    dmesg | grep -i nvrm  
    dmesg | grep -i signature  
    journalctl -k | grep -i nvidia  

Check driver packages:  

Ubuntu:  

    dpkg -l | grep -i nvidia  

Rocky / RHEL:  

    rpm -qa | grep -i nvidia  

Check DKMS:  

    dkms status  

### 5.5 Handling Procedures  

Handling order:  

    1. Confirm whether nouveau is disabled  
    2. Confirm Secure Boot status  
    3. Confirm kernel headers / kernel-devel match  
    4. Confirm NVIDIA driver version supports current GPU  
    5. Reinstall or switch driver version  
    6. Reboot the node  
    7. Re-validate nvidia-smi  

Ubuntu common dependencies:  

    sudo apt-get install -y build-essential dkms linux-headers-$(uname -r)  

Rocky common dependencies:  

    sudo dnf install -y gcc make dkms kernel-devel-$(uname -r) kernel-headers-$(uname -r)  

### 5.6 Review Key Points  

Record:  

    Kernel version  
    Driver version  
    Secure Boot status  
    nouveau status  
    DKMS status  
    Installation method  
    Whether kernel upgrade occurred  
    Final fix method  

---

## Six, Case Three: Node has no nvidia.com/gpu  

### 6.1 Phenomenon  

On a GPU node, execute:  

    nvidia-smi  

Works normally.  

But in Kubernetes, execute:  

    kubectl describe node <gpu-node-name>  

Cannot see:  

    nvidia.com/gpu  

### 6.2 Initial Judgment  

The host driver layer is normal, but the Kubernetes Device Plugin registration chain is abnormal.  

### 6.3 Possible Causes  

Common causes:  

- NVIDIA Device Plugin not deployed  
- GPU Operator not running normally  
- Device Plugin Pod CrashLoopBackOff  
- Device Plugin not scheduled to GPU node  
- Device Plugin logs show errors  
- kubelet device plugin manager abnormal  
- GPU marked as unhealthy by Device Plugin  
- Driver is usable, but runtime or device files are abnormal  
- Node taint prevents Device Plugin from running  
- GPU Operator Validator failed  

### 6.4 Troubleshooting Commands  

Check NVIDIA components:  

    kubectl get pods -A | grep -i nvidia  
    kubectl get ds -A | grep -i nvidia  

Check Device Plugin:  

    kubectl get pods -A | grep -i device-plugin  
    kubectl logs <device-plugin-pod> -n <namespace>  
    kubectl describe pod <device-plugin-pod> -n <namespace>  

If using GPU Operator:  

    kubectl get pods -n gpu-operator -o wide  
    kubectl get ds -n gpu-operator  
    kubectl get clusterpolicy  
    kubectl describe clusterpolicy  

Check kubelet:  

    systemctl status kubelet  
    journalctl -u kubelet -f  

Local on the node:  

    nvidia-smi  
    ls -l /dev/nvidia*  
    nvidia-container-cli info  

### 6.5 Handling Procedures  

Handling order:

1. Confirm Device Plugin is deployed  
2. Confirm Device Plugin Pod is Running  
3. Check Device Plugin logs  
4. Confirm GPU node drivers and device files  
5. Check GPU Operator components  
6. Restart Device Plugin Pod  
7. Restart kubelet if necessary  
8. Recheck Node Capacity / Allocatable  

Restart Device Plugin Pod:  

    kubectl delete pod <device-plugin-pod> -n <namespace>  

Restart kubelet:  

    systemctl restart kubelet  

### 6.6 Post-Mortem Key Points  

Record:  

    Device Plugin version  
    GPU Operator version  
    Driver version  
    kubelet logs  
    Whether Node has nvidia.com/gpu  
    Pre- and post-repair Node describe outputs  

---

## Seven, Case Four: GPU Pod Always Pending  

### 7.1 Phenomenon  

After creating a GPU Pod:  

    kubectl get pod <pod-name> -n <namespace> -o wide  

Status remains:  

    Pending  

### 7.2 Initial Judgment  

Pod failed to schedule to a node.  

This is a Scheduler phase issue, not a container runtime issue.  

### 7.3 Check Events  

    kubectl describe pod <pod-name> -n <namespace>  

Focus on Events.  

### 7.4 Common Event One: insufficient nvidia.com/gpu  

Event example:  

    0/3 nodes are available: insufficient nvidia.com/gpu  

Possible causes:  

- Node not registered with GPU  
- GPU occupied by other Pods  
- Pod requests exceed single-node available GPU count  
- Device Plugin anomaly  
- GPU marked as unhealthy  
- Namespace quota limits  

Troubleshoot:  

    kubectl describe node <gpu-node-name>  
    kubectl get pods -A -o wide | grep <gpu-node-name>  
    kubectl describe nodes | grep -A10 -B5 "nvidia.com/gpu"  
    kubectl describe resourcequota -n <namespace>  

Resolution:  

- Release unused GPU Pods  
- Add more GPU nodes  
- Adjust Pod GPU request count  
- Fix Device Plugin  
- Adjust ResourceQuota  
- Check GPU health  

### 7.5 Common Event Two: untolerated taint  

Event example:  

    node(s) had untolerated taint {nvidia.com/gpu: true}  

Indicates GPU node has taint, but Pod lacks toleration.  

Check node taints:  

    kubectl describe node <gpu-node-name> | grep -i taints -A5  

Add toleration to Pod:  

    tolerations:  
      - key: "nvidia.com/gpu"  
        operator: "Equal"  
        value: "true"  
        effect: "NoSchedule"  

### 7.6 Common Event Three: node selector mismatch  

Event example:  

    node(s) didn't match Pod's node affinity/selector  

Check Pod:  

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A20 nodeSelector  
    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A40 affinity  

Check node labels:  

    kubectl get node <gpu-node-name> --show-labels  

Resolution:  

- Fix nodeSelector  
- Fix nodeAffinity  
- Add correct labels to nodes  
- Check GPU model and business requirements match  

### 7.7 Common Event Four: CPU or memory insufficient  

Event example:  

    insufficient cpu  
    insufficient memory  

Indicates GPU is not the issue.  

Resolution:  

- Reduce CPU / Memory requests  
- Clean up nodes  
- Adjust node pools  
- Use larger node specifications  
- Split tasks  

### 7.8 Post-Mortem Key Points  

Record:  

    Pod YAML  
    Pod Events  
    Node Capacity / Allocatable  
    Node Taints  
    Node Labels  
    Namespace Quota  
    Final scheduling failure reason  

---

## Eight, Case Five: Pod Running but cannot use GPU inside container  

### 8.1 Phenomenon  

Pod status is:  

    Running  

But inside the container:  

    nvidia-smi  

Fails.  

### 8.2 Initial Judgment  

Scheduler successfully scheduled Pod, but GPU devices not properly exposed to container.  

Issue likely in:  

- Pod resource declarations  
- NVIDIA Container Toolkit  
- containerd runtime  
- Device Plugin allocation  
- Image environment  
- RuntimeClass  
- Device mounting  

### 8.3 Check if Pod requested GPU  

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A20 resources  

Should see:  

    limits:  
      nvidia.com/gpu: 1  

If not, Pod is ordinary and won't get GPU.  

### 8.4 Check devices inside container  

Enter container:  

    kubectl exec -it <pod-name> -n <namespace> -- sh  

Check: /think

ls -l /dev/nvidia*
echo $CUDA_VISIBLE_DEVICES
echo $NVIDIA_VISIBLE_DEVICES
echo $NVIDIA_DRIVER_CAPABILITIES

If `/dev/nvidia*` is not present, it indicates the device is not mounted into the container.

### 8.5 Troubleshooting Node Runtime

Execute on the node:

    nvidia-container-cli info
    containerd config dump | grep -i nvidia -A30 -B10
    systemctl status containerd
    systemctl status kubelet

If containerd configuration has been modified, restart:

    systemctl restart containerd
    systemctl restart kubelet

Then rebuild the Pod.

### 8.6 Case Where Image Lacks nvidia-smi

Some business images lack `nvidia-smi`.

In such cases, you cannot directly determine GPU unavailability.

Check:

    ls -l /dev/nvidia*
    python3 -c "import torch; print(torch.cuda.is_available())"

You can also deploy the official test image:

    nvidia/cuda:12.2.0-base-ubuntu22.04

### 8.7 Handling Procedures

Handling order:

    1. Confirm Pod requests nvidia.com/gpu
    2. Confirm Node has nvidia.com/gpu
    3. Confirm Device Plugin is normal
    4. Confirm containerd configuration NVIDIA runtime
    5. Confirm NVIDIA Container Toolkit is normal
    6. Use official CUDA image for reproduction
    7. Re-examine business image

### 8.8 Post-mortem Key Points

Record:

    Pod resources
    containerd configuration
    nvidia-container-cli info
    Device Plugin logs
    /dev/nvidia* inside container
    Test image results
    Business image differences

---

## Nine, Case Six: PyTorch / TensorFlow Cannot Detect GPU

### 9.1 Phenomenon

Inside the container:

    nvidia-smi

Is normal.

But PyTorch:

    python3 -c "import torch; print(torch.cuda.is_available())"

Outputs:

    False

Or TensorFlow:

    python3 -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"

Outputs empty.

### 9.2 Initial Judgment

GPU devices are exposed to the container, but AI framework layer cannot use them.

The issue is likely in:

- Python package versions;
- CPU version PyTorch;
- CUDA Runtime;
- cuDNN;
- TensorFlow version;
- Driver compatibility with CUDA Runtime;
- Image build errors;
- Environment variables overriding.

### 9.3 Troubleshoot PyTorch

Execute:

    python3 -c "import torch; print('torch:', torch.__version__); print('cuda:', torch.version.cuda); print('available:', torch.cuda.is_available()); print('count:', torch.cuda.device_count())"

Judge:

    If torch.version.cuda is None:
        Likely CPU version PyTorch.

    If torch.version.cuda has a value but available is False:
        Possibly driver compatibility, CUDA Runtime, or device visibility issues.

### 9.4 Troubleshoot TensorFlow

Execute:

    python3 -c "import tensorflow as tf; print(tf.__version__); print(tf.config.list_physical_devices('GPU'))"

Also check logs for:

    Could not load dynamic library
    libcudart.so not found
    libcudnn.so not found
    CUDA driver version is insufficient

### 9.5 Troubleshoot Driver Compatibility

On the host:

    nvidia-smi

Inside the container:

    nvidia-smi

Check container CUDA:

    python3 -c "import torch; print(torch.version.cuda)"

If it reports:

    CUDA driver version is insufficient for CUDA runtime version

It indicates the host driver is too old to support the container CUDA Runtime.

### 9.6 Handling Procedures

Handling order:

    1. Confirm framework is GPU version
    2. Confirm CUDA Runtime version
    3. Confirm cuDNN compatibility
    4. Confirm host driver supports CUDA Runtime
    5. Use official framework image for testing
    6. Standardize enterprise base image

### 9.7 Post-mortem Key Points

Record:

    Image name and tag
    Python version
    PyTorch/TensorFlow version
    CUDA Runtime version
    cuDNN version
    Driver version
    Whether CPU version packages are used
    Fixed standard image

---

## Ten, Case Seven: CUDA out of memory

### 10.1 Phenomenon

Business logs show:

    CUDA out of memory

PyTorch commonly shows:

    RuntimeError: CUDA out of memory

The Pod may still be Running or enter CrashLoopBackOff.

### 10.2 Initial Judgment

This is a GPU memory shortage issue, not Kubernetes container memory shortage.

Note: /think

Kubernetes memory limit restricts container memory.
GPU memory is not directly limited by memory limit.
nvidia.com/gpu: 1 indicates requesting one GPU, not limiting memory size.

### 10.3 Troubleshooting Memory

Inside container or host:

    nvidia-smi

Check:

    Memory-Usage

Prometheus:

    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_FB_FREE

Check occupied processes:

    nvidia-smi

Host reverse lookup:

    ps -ef | grep <PID>

Kubernetes reverse lookup Pod:

    kubectl get pod -A -o wide | grep <gpu-node-name>
    crictl ps
    crictl inspect <container-id>

### 10.4 Common Causes

Common causes:

- Batch size too large;
- Model too large;
- Inference concurrency too high;
- Multiple workers reloading model repeatedly;
- Multiple processes sharing same GPU;
- Memory leak;
- Framework caching memory;
- Abnormal input data;
- MIG instance memory too small;
- Time-Slicing scenario memory contention;
- Pod requested GPU, but business started multiple sub-processes.

### 10.5 Handling Methods

Temporary quick fix:

    1. Reduce batch size
    2. Reduce concurrency
    3. Restart abnormal Pod
    4. Clean up residual processes
    5. Migrate to larger memory GPU
    6. Suspend low-priority tasks

Long-term fix:

    1. Optimize model
    2. Use FP16 / BF16
    3. Use quantization
    4. Optimize memory release logic
    5. Control worker count
    6. Establish memory usage baseline
    7. Use MIG for resource isolation
    8. Standardize inference service resource templates

### 10.6 Post-Mortem Points

Record:

    Model name
    Batch size
    Concurrency count
    GPU model
    GPU memory
    Peak memory
    Pod replica count
    Worker count
    OOM timestamp
    Whether related to traffic peak
    Final optimization strategy

---

## Eleven, Case Eight: Low GPU Utilization

### 11.1 Phenomenon

Prometheus / Grafana shows:

    GPU utilization long-term below 5%

But GPU memory may be occupied.

### 11.2 Initial Judgment

Low GPU utilization doesn't necessarily indicate a fault.

Need to judge based on business context.

### 11.3 Normal Situations

The following may be normal:

- Inference service in low-traffic period;
- Model resident in GPU waiting for requests;
- Experimental environment unused;
- Scheduled tasks not started;
- Service needs low latency but low throughput;
- GPU used for occasional tasks.

### 11.4 Abnormal Situations

The following need attention:

- Training task GPU utilization long-term low;
- High memory usage but no business requests;
- Pod long-term occupies GPU but no logs;
- High business QPS but low GPU utilization;
- GPU utilization periodically drops to 0;
- Multi-card task only partial GPUs have utilization.

### 11.5 Troubleshooting Commands

Check GPU:

    nvidia-smi

Check Pod:

    kubectl get pod -A -o wide | grep <gpu-node-name>
    kubectl logs <pod-name> -n <namespace>

Check application:

    QPS
    P95/P99 latency
    Data loading time
    Batch size
    Worker count
    Business logs

Check CPU:

    kubectl top pod -n <namespace>
    kubectl top node

### 11.6 Common Causes

- Data loading slow;
- CPU preprocessing slow;
- Network storage slow;
- Batch size too small;
- Low request volume;
- Business not actually using GPU;
- Framework fallback to CPU;
- Multiple processes waiting for lock;
- Data set reading failure;
- Training task stuck;
- Model service not receiving traffic.

### 11.7 Handling Methods

- Optimize data loading;
- Increase CPU request;
- Adjust batch size;
- Use DataLoader with multiple workers;
- Check storage throughput;
- Check network;
- Optimize business concurrency;
- Recycle long-idle Pods;
- Scale down inference service based on traffic;
- Add monitoring and heartbeat for training tasks.

### 11.8 Post-Mortem Points

Record:

    Business type
    GPU utilization trend
    Memory usage
    QPS
    Latency
    CPU usage
    Data loading time
    Whether normal low-traffic
    Whether need resource recycling

---

## Twelve, Case Nine: High GPU Utilization but Poor Business Performance

### 12.1 Phenomenon

GPU utilization long-term near:

    90% - 100%

But business performance is poor:

- High inference latency;
- Low QPS;
- Low training throughput;
- Request queuing;
- Increased error rate.

### 12.2 Initial Judgment

High GPU utilization doesn't mean high business efficiency.

It could be model, batch, concurrency, CPU, network, storage, or framework configuration issues.

### 12.3 Troubleshooting Directions

Business metrics:

    QPS
    P50 / P95 / P99
    Error rate
    Queue length
    Timeout count

GPU metrics:

    GPU utilization
    Memory usage
    Power consumption
    Temperature
    XID

CPU / Memory:

    kubectl top pod
    kubectl top node

Logs:

    kubectl logs <pod-name> -n <namespace>

### 12.4 Common Causes

- Unreasonable batch size;
- Single request computation too heavy;
- Model not optimized;
- Not using TensorRT;
- Not using FP16/BF16;
- CPU preprocessing/bottleneck;
- Frequent data copy;
- Network request queuing;
- Process contention;
- Temperature/power consumption causing throttling;
- GPU sharing causing jitter.

### 12.5 Handling Methods

- Adjust batch size;
- Model quantization;
- Use TensorRT;
- Use FP16 / BF16;
- Increase replicas;
- Optimize request queue;
- Optimize CPU preprocessing;
- Increase CPU request;
- Check temperature and power consumption;
- Avoid critical business sharing GPU with low-priority tasks.

---

## Thirteen, Case Ten: High GPU Temperature

### 13.1 Phenomenon

Monitoring alerts:

    GPU temperature > 80°C for 5 minutes continuously

Or:

    GPU temperature > 90°C

On node:

    nvidia-smi

Displaying High Temperature

### 13.2 Preliminary Judgment

High temperature may originate from normal full load or cooling failure.

Key considerations:

- Is it persistent?
- Is it accompanied by throttling?
- Is it accompanied by power limitation?
- Is it accompanied by XID?
- Are multiple cards simultaneously overheating?
- Are multiple nodes simultaneously overheating?
- Is the data center environment abnormal?

### 13.3 Troubleshooting Commands

GPU:

    nvidia-smi
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
    DCGM_FI_DEV_POWER_USAGE
    DCGM_FI_DEV_THERMAL_VIOLATION

### 13.4 Common Causes

- Long-term full load on GPU;
- High data center temperature;
- Incorrect fan strategy;
- Blocked airflow;
- Abnormal server intake airflow;
- Inappropriate installation environment for passive-cooled GPU;
- Too small GPU spacing;
- Dust accumulation;
- Fan failure;
- Hotspot in the cabinet where the node is located.

### 13.5 Handling Methods

Temporary mitigation:

    1. Migrate low-priority GPU Pods
    2. Reduce task concurrency
    3. Cordon the node
    4. Suspend training tasks
    5. Notify the data center to check temperature

Long-term resolution:

    1. Adjust fan strategy
    2. Optimize cabinet airflow
    3. Clean dust
    4. Check server cooling configuration
    5. Adjust node rack position
    6. Optimize task scheduling
    7. Add high-temperature alerts and automated handling

### 13.6 Post-Incident Review Points

Record:

    High temperature time
    GPU model
    Highest temperature
    Pods running at the time
    Business load
    Data center temperature
    Fan status
    Did throttling occur?
    Did XID occur?
    Final handling action

---

## Fourteen, Case Eleven: GPU XID Error

### 14.1 Symptoms

Monitoring shows:

    DCGM_FI_DEV_XID_ERRORS > 0

Node logs:

    dmesg | grep -i xid

You may see:

    NVRM: Xid

### 14.2 What is XID

XID is an error message reported by the NVIDIA driver for the GPU.

It is written to the system kernel log or event log.

XID may indicate:

- User application issues;
- Driver issues;
- CUDA program anomalies;
- GPU hardware issues;
- Memory errors;
- PCIe issues;
- Temperature issues;
- Power supply issues;
- GPU card detachment.

Do not immediately conclude the GPU is faulty just from seeing XID.

Comprehensive judgment is needed based on XID type, occurrence frequency, business logs, temperature, power consumption, PCIe, and node status.

### 14.3 Troubleshooting Commands

Check XID:

    dmesg | grep -i xid
    journalctl -k | grep -i xid
    journalctl -k | grep -i nvrm

Check GPU:

    nvidia-smi
    nvidia-smi -q
    nvidia-smi -L

Check temperature and power consumption:

    nvidia-smi -q | grep -i temperature -A20
    nvidia-smi -q | grep -i power -A20

Check PCIe:

    dmesg | grep -i aer
    dmesg | grep -i pci

Check the Pod at the time:

    kubectl get pod -A -o wide | grep <gpu-node-name>

Check business logs:

    kubectl logs <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace> --previous

### 14.4 Judging XID Severity

Pay attention to:

    Single occurrence:
        Record and observe.

    Short-term burst:
        May indicate driver crash, PCIe, hardware, or application anomalies.

    Same GPU recurring:
        Focus on hardware health.

    Occurring with high temperature:
        Prioritize cooling and power consumption checks.

    Occurring with a specific Pod:
        Investigate business program, CUDA, and framework.

    Occurring with GPU fallen off the bus:
        High risk, prioritize isolating the node.

### 14.5 Temporary Mitigation

If affecting business:

    kubectl cordon <gpu-node-name>

Migrate movable tasks:

    kubectl drain <gpu-node-name> --ignore-daemonsets --delete-emptydir-data

Note:

    Drain will evict Pods, training tasks may lose progress.
    In production environments, execute based on business priority and checkpoint status.

### 14.6 Handling Methods

Choose based on the situation:

- Restart abnormal Pods;
- Restart Device Plugin;
- Restart kubelet;
- Restart the node;
- Update driver version;
- Check temperature and power supply;
- Check PCIe riser;
- Single-card stress test;
- Decommission the node;
- Contact vendor;
- Replace GPU.

### 14.7 Post-Incident Review Points

Record:

    XID number
    Occurrence time
    GPU UUID
    GPU model
    Node name
    Pods running at the time
    Temperature
    Power consumption
    AER present?
    GPU fallen off the bus?
    Recurring?
    Handling action
    Hardware replacement needed?

---

## Fifteen, Case Twelve: GPU Fallen Off the Bus

### 15.1 Symptoms

Logs show:

    GPU has fallen off the bus

May also appear:

- nvidia-smi is stuck;
- GPU disappears;
- Pod anomalies;
- XID errors;
- Node reboot;
- GPU resources not recovered.

### 15.2 Preliminary Judgment

This is a high-risk failure.

Typically, investigation is needed from hardware, PCIe, power supply, temperature, and driver perspectives.

### 15.3 Troubleshooting Commands /think

dmesg | grep -i "fallen"
dmesg | grep -i xid
dmesg | grep -i aer
dmesg | grep -i pci
journalctl -k | grep -i nvrm
nvidia-smi
ipmitool sel list
ipmitool sensor

### 15.4 Common Causes

- PCIe connection issues;
- riser failure;
- GPU not securely inserted;
- insufficient power supply;
- excessive temperature;
- motherboard slot anomaly;
- driver issues;
- GPU hardware failure;
- BIOS / Firmware issues.

### 15.5 Handling Procedures

Recommendations:

    1. Immediately cordon the node
    2. Migrate migratable workloads
    3. Save logs
    4. Restart the node for observation
    5. If recurring, stop scheduling on this node
    6. Perform single-GPU stress testing
    7. Check riser, slot, and power supply
    8. Upgrade BIOS / BMC / Firmware
    9. Contact vendor for assistance

### 15.6 Post-Mortem Focus Points

This type of issue must be recorded in hardware health records.

Record:

    Whether recurring
    Whether same GPU
    Whether same slot
    Whether same riser
    Whether triggered under full load
    Whether temperature-triggered
    Whether recovered after hardware replacement

---

## SixteenI don't know.Case Thirteen: Device Plugin CrashLoopBackOff

### 16.1 Symptoms

Check:

    kubectl get pods -A | grep -i nvidia

Discover Device Plugin Pod:

    CrashLoopBackOff

### 16.2 Initial Judgment

Device Plugin fails to function normally, potentially causing the node to lack `nvidia.com/gpu` or GPU resources becoming unavailable.

### 16.3 Troubleshooting Commands

Check Pod:

    kubectl describe pod <device-plugin-pod> -n <namespace>
    kubectl logs <device-plugin-pod> -n <namespace>

Check node:

    kubectl get pod <device-plugin-pod> -n <namespace> -o wide

Go to corresponding node:

    nvidia-smi
    ls -l /dev/nvidia*
    nvidia-container-cli info
    systemctl status kubelet
    systemctl status containerd

### 16.4 Common Causes

- Node driver anomalies;
- `/dev/nvidia*` does not exist;
- Unhealthy GPU;
- Device Plugin image pull failure;
- MIG configuration error;
- Node has no GPU but plugin runs forcibly;
- Container runtime configuration anomaly;
- Pod security policy restrictions;
- GPU Operator conflicts with manually deployed Device Plugin.

### 16.5 Handling Procedures

- Repair drivers;
- Check device files;
- Repair container runtime;
- Check MIG configuration;
- Check Helm values;
- Avoid duplicate Device Plugin deployment;
- Delete faulty Pod to allow DaemonSet to recreate;
- Restart kubelet if necessary.

---

## SeventeenI don't know.Case Fourteen: GPU Operator Validator Failure

### 17.1 Symptoms

Check:

    kubectl get pods -n gpu-operator

Discover:

    nvidia-operator-validator

Status is abnormal.

### 17.2 Initial Judgment

Validator failure typically indicates that a component managed by GPU Operator failed verification.

Possible causes:

- Driver;
- Toolkit;
- Device Plugin;
- DCGM;
- Runtime;
- MIG;
- Node Feature Discovery;
- GPU Feature Discovery.

### 17.3 Troubleshooting Commands

    kubectl logs <validator-pod> -n gpu-operator
    kubectl describe pod <validator-pod> -n gpu-operator
    kubectl get pods -n gpu-operator -o wide
    kubectl get ds -n gpu-operator
    kubectl describe clusterpolicy

### 17.4 Handling Procedures

Do not simply restart validator.

Locate the corresponding component based on validator logs.

Examples:

    Driver validation failure:
        Check nvidia-driver-daemonset and node nvidia-smi.

    Toolkit validation failure:
        Check nvidia-container-toolkit-daemonset and containerd configuration.

    Plugin validation failure:
        Check nvidia-device-plugin-daemonset and Node nvidia.com/gpu.

    DCGM validation failure:
        Check dcgm-exporter and /metrics.

---

## EighteenI don't know.Case Fifteen: DCGM Exporter Has No Metrics

### 18.1 Symptoms

Grafana GPU dashboard has no data.

Prometheus cannot query:

    DCGM_FI_DEV_GPU_UTIL

Or DCGM Exporter is DOWN in Prometheus Targets.

### 18.2 Troubleshooting Commands

Check Pod:

    kubectl get pods -A | grep -i dcgm

Check Service:

    kubectl get svc -A | grep -i dcgm

Check logs:

    kubectl logs <dcgm-exporter-pod> -n <namespace>

Test metrics:

    kubectl port-forward -n <namespace> <dcgm-exporter-pod> 9400:9400
    curl http://127.0.0.1:9400/metrics

Check Prometheus Targets:

    Status -> Targets

### 18.3 Common Causes

- DCGM Exporter Pod is not running;
- DCGM Exporter cannot access GPU;
- ServiceMonitor selector error;
- Service port name mismatch;
- Prometheus lacks permission to scrape;
- NetworkPolicy blocks access;
- Namespace selector error;
- Metric name and Dashboard mismatch;
- GPU Operator has not enabled DCGM Exporter.

### 18.4 Resolution Methods

- Fix DCGM Exporter Pod;
- Fix Service;
- Fix ServiceMonitor;
- Check Prometheus RBAC;
- Check Target;
- Adjust Grafana PromQL according to actual metrics;
- Ensure DCGM Exporter is deployed on GPU nodes.

---

## NineteenI don't know.Case Sixteen: GPU Memory Not Released After Pod Deletion

### 19.1 Phenomenon

After deleting a Pod:

    kubectl delete pod <pod-name> -n <namespace>

But on the node:

    nvidia-smi

Still shows memory is occupied.

### 19.2 Initial Judgment

There may be residual processes, containers not fully exited, runtime anomalies, or nvidia-smi display delay.

### 19.3 Troubleshooting Commands

Check GPU processes:

    nvidia-smi

Check host processes:

    ps -ef | grep <PID>

Check containers:

    crictl ps -a
    crictl inspect <container-id>

Check kubelet / containerd:

    systemctl status kubelet
    systemctl status containerd
    journalctl -u kubelet -f
    journalctl -u containerd -f

### 19.4 Resolution Methods

- Wait for a short time to confirm if memory is released;
- Clean up abnormal containers;
- Terminate residual processes;
- Restart containerd;
- Restart kubelet;
- In severe cases, cordon the node and restart the node.

Note:

    Do not kill processes randomly without confirming the PID.
    First confirm whether the process belongs to a still-running business Pod.

---

## TwentyI don't know.Case Seventeen: Multi-GPU Pod Cannot Schedule

### 20.1 Phenomenon

Pod requests:

    nvidia.com/gpu: 2

But it remains Pending.

### 20.2 Initial Judgment

Kubernetes requires that the GPU requested by a Pod comes from the same node.

If the cluster has 2 idle GPUs, but they are distributed across two nodes, the Pod still cannot schedule.

### 20.3 Troubleshooting Commands

Check Pod:

    kubectl describe pod <pod-name> -n <namespace>

Check nodes:

    kubectl describe nodes | grep -A10 -B5 "nvidia.com/gpu"

Check Pods running on each node:

    kubectl get pod -A -o wide | grep <gpu-node-name>

### 20.4 Common Causes

- Single node has less than 2 idle GPUs;
- GPUs are occupied by other Pods;
- nodeSelector restricts available nodes;
- taint/toleration mismatch;
- CPU / Memory insufficient;
- Namespace quota insufficient;
- Some GPUs are marked as unhealthy.

### 20.5 Resolution Methods

- Release GPUs on the same node;
- Adjust task to single GPU;
- Use a multi-Pod architecture supporting distributed training;
- Add multi-GPU nodes;
- Adjust scheduling constraints;
- Check quotas.

---

## Twenty-oneI don't know.Case Eighteen: Poor Performance in Multi-GPU Training

### 21.1 Phenomenon

Multi-GPU training tasks can run, but performance does not meet expectations.

Manifestations:

- Multi-GPU performance improvement is not significant compared to single GPU;
- GPU utilization is unbalanced;
- Some GPUs have long-term low utilization;
- Low training throughput;
- NCCL errors;
- Multi-node training gets stuck.

### 21.2 Troubleshooting Directions

Check GPU topology:

    nvidia-smi topo -m

Check PCIe:

    lspci -tv
    lspci -vvv -s <GPU_PCI_ID> | grep -i "Lnk"

Check NUMA:

    lscpu | grep -i numa
    cat /sys/bus/pci/devices/0000:<PCI_ID>/numa_node

Check network:

    ibstat
    ibv_devinfo
    ethtool <interface>
    ip addr
    ip route

Check NCCL logs:

    NCCL_DEBUG=INFO

### 21.3 Common Causes

- No NVLink between GPUs;
- Cross NUMA communication;
- GPUs and network cards are far apart;
- PCIe speed reduced;
- RDMA not enabled;
- NCCL configuration error;
- Data loading bottleneck;
- Insufficient CPU;
- Low storage throughput;
- Inappropriate batch size.

### 21.4 Resolution Methods

- Choose GPU nodes with better topology;
- Adjust Pod scheduling;
- Optimize NCCL parameters;
- Check RDMA;
- Increase CPU request;
- Optimize data loading;
- Use local caching;
- Adjust batch size;
- Use same model and topology nodes for training tasks.

---

## Twenty-twoI don't know.Common GPU Troubleshooting Commands

### 22.1 Kubernetes Side

Check Pod:

    kubectl get pod -A -o wide
    kubectl describe pod <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace> --previous

Check Node:

    kubectl get nodes -o wide
    kubectl describe node <gpu-node-name>
    kubectl get node <gpu-node-name> --show-labels

Check Events:

    kubectl get events -A --sort-by=.lastTimestamp
    kubectl get events -n <namespace> --sort-by=.lastTimestamp

Check NVIDIA components: /think

kubectl get pods -A | grep -i nvidia
kubectl get ds -A | grep -i nvidia
kubectl get pods -n gpu-operator -o wide
kubectl logs <nvidia-pod> -n <namespace>

Check resource quotas:

    kubectl describe resourcequota -n <namespace>

### 22.2 Node Side

Hardware:

    lspci | grep -i nvidia
    lspci -tv
    lspci -vvv -s <GPU_PCI_ID> | grep -i "Lnk"

Drivers:

    nvidia-smi
    nvidia-smi -L
    nvidia-smi -q
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

## Twenty-Three, GPU Emergency Response Actions

### 23.1 Cordon Node

Prevent new Pods from scheduling:

    kubectl cordon <gpu-node-name>

### 23.2 Drain Node

Migrate migratable Pods:

    kubectl drain <gpu-node-name> --ignore-daemonsets --delete-emptydir-data

Notes:

    Drain will evict Pods.
    Training tasks may lose progress without checkpoints.
    Stateful services require careful handling.

### 23.3 Restore Scheduling

    kubectl uncordon <gpu-node-name>

### 23.4 Delete Abnormal Pod

    kubectl delete pod <pod-name> -n <namespace>

### 23.5 Restart Device Plugin

    kubectl delete pod <device-plugin-pod> -n <namespace>

### 23.6 Restart Node Components

    systemctl restart containerd
    systemctl restart kubelet

### 23.7 Restart Node

Consider this in the following scenarios:

- GPU card failure;
- nvidia-smi freeze;
- XID duplication;
- containerd/kubelet unable to recover;
- GPU memory residue cannot be cleaned;
- Driver status anomaly.

Before production restart, must:

    1. Cordon node
    2. Drain migratable workloads
    3. Notify business teams
    4. Save logs
    5. Confirm maintenance window

---

## Twenty-Four, GPU Fault Review Template

After every GPU production fault, record according to the template.

    Fault ID:
    Fault Time:
    Recovery Time:
    Impact Scope:
    Affected Business:
    Affected Nodes:
    GPU Model:
    GPU UUID:
    Kubernetes Namespace:
    Pod Name:
    Phenomenon Description:
    Alarm Name:
    First Discovery Method:
    Key Logs:
    Key Metrics:
    XID Number:
    High Temperature:
    Power Abnormality:
    PCIe Abnormality:
    Driver Abnormality:
    Device Plugin Abnormality:
    Business Code Abnormality:
    Temporary Emergency Actions:
    Root Cause Analysis:
    Final Fix Actions:
    Hardware Replacement Needed:
    Driver Upgrade Needed:
    Image Fix Needed:
    Scheduling Policy Adjustment Needed:
    New Alarm Needed:
    Runbook Update Needed:
    Responsible Person:
    Review Date:

---

## Twenty-Five, Production Environment GPU Troubleshooting Recommendations

### 25.1 Establish Node Baseline

Each GPU node should save:

    lspci | grep -i nvidia
    nvidia-smi
    nvidia-smi -q
    nvidia-smi -L
    nvidia-smi topo -m
    lspci -tv
    lscpu
    dmidecode -t bios
    ipmitool sensor

### 25.2 Establish Version Matrix

Record:

    OS Version
    Kernel Version
    NVIDIA Driver Version
    CUDA Runtime Version
    Container Runtime Version
    NVIDIA Container Toolkit Version
    Device Plugin Version
    GPU Operator Version
    DCGM Exporter Version
    PyTorch / TensorFlow Version

### 25.3 Establish Monitoring Baseline

Must monitor:

- GPU utilization;
- GPU memory;
- GPU temperature;
- GPU power consumption;
- XID;
- ECC;
- Device Plugin;
- DCGM Exporter;
- GPU Operator;
- GPU Pod Pending;
- GPU Pod Restart;
- Namespace GPU Usage.

### 25.4 Establish Runbook

Each alarm must have a handling document: /think

GPUHighTemperature
GPUXIDError
GPUMemoryUsageHigh
GPUPodPending
DevicePluginDown
DCGMExporterDown
GPUNodeNotReady

### 25.5 Establishing Drill Mechanisms

Recommended regular drills:

- GPU Pod Pending;
- Device Plugin anomalies;
- DCGM Exporter Down;
- High GPU memory usage;
- High GPU temperature;
- Node cordon / drain;
- GPU node reboot;
- Business Pod migration.

---

## Twenty-Six, Summary

The key to GPU fault troubleshooting is not memorizing a single command, but building a layered judgment capability.

Troubleshooting steps should follow this order:

    1. First confirm the impact scope
    2. Then determine the fault level
    3. Confirm if hardware is visible
    4. Confirm if drivers are normal
    5. Confirm if Kubernetes has registered GPU
    6. Confirm if Pod scheduling was successful
    7. Confirm if GPU is visible inside container
    8. Confirm if CUDA / AI framework is available
    9. Judge business status by combining metrics and logs
    10. Perform bleeding control, repair, and post-mortem analysis

Common judgment mnemonics:

    lspci is abnormal:
        Check hardware, BIOS, PCIe, power supply.

    nvidia-smi is abnormal:
        Check drivers, kernel modules, Secure Boot, nouveau, XID.

    Node has no nvidia.com/gpu:
        Check Device Plugin, GPU Operator, kubelet.

    Pod Pending:
        Check Scheduler, GPU resources, Taint, Label, Quota, CPU/Memory.

    Pod Running but no GPU:
        Check NVIDIA Container Toolkit, containerd, device mounting.

    Framework has no GPU:
        Check PyTorch/TensorFlow, CUDA Runtime, cuDNN, driver compatibility.

    CUDA OOM:
        Check memory, model, batch size, concurrency, worker, MIG.

    GPU utilization anomaly:
        Check business traffic, data loading, CPU, network, storage, application logs.

    XID error:
        Check dmesg, journalctl, temperature, power consumption, PCIe, hardware and business timestamps.

In production environments, GPU fault troubleshooting must form a closed-loop:

    Alarm detection
      ↓
    Metric localization
      ↓
    Log analysis
      ↓
    Command verification
      ↓
    Temporary bleeding control
      ↓
    Root cause repair
      ↓
    Post-mortem record
      ↓
    Runbook update
      ↓
    Monitoring rule optimization

Only in this way can GPU clusters evolve from "being able to run tasks" to becoming production-grade AI infrastructure that is observable, troubleshootable, recoverable, and governable.

---

## Reference Documents

- NVIDIA Xid Errors:
  https://docs.nvidia.com/deploy/xid-errors/

- NVIDIA GPU Operator Troubleshooting:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/troubleshooting.html

- NVIDIA DCGM Exporter:
  https://docs.nvidia.com/datacenter/dcgm/latest/gpu-telemetry/dcgm-exporter.html

- NVIDIA Kubernetes Device Plugin:
  https://github.com/NVIDIA/k8s-device-plugin

- NVIDIA GPU Operator:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/

- NVIDIA Container Toolkit:
  https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

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