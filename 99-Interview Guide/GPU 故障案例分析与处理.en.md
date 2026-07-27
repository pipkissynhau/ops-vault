---
tags: [GPU, troubleshooting, Ops cases, Kubernetes, interview]
---

# Analysis and Handling of GPU Failure Cases

## Failure Scenarios
- One GPU node in the cluster is preventing users from launching deep learning Pods, with containers remaining in a `Pending` state for extended periods.  
- Logs indicate: `Requested device nvidia.com/gpu not found`.  
- Running `nvidia-smi` on the node yields no valid GPU information.  

---

## Troubleshooting Steps

### 1. Hardware Inspection
```bash
# Check if the GPU is recognized by the system
lspci | grep -i nvidia

# Verify GPU status and temperature
nvidia-smi
```

- Result: The system does not recognize the GPU; the hardware appears normal, but the fan is not spinning, and the temperature readings are abnormal.  

---

### 2. Driver/Software Verification
```bash
# Check NVIDIA driver version
cat /proc/driver/nvidia/version

# Verify kernel modules
lsmod | grep nvidia
```

- Result: The driver version is incompatible with the operating system kernel, and the NVIDIA module is not loaded.  

---

### 3. Container Access Test
```bash
docker run --rm --gpus all nvidia/cuda:12.1-base nvidia-smi
```

- Error message indicates that the GPU device cannot be found.  

---

### 4. Kubernetes Verification
```bash
# Check node GPU resources
kubectl describe node gpu-node

# Verify Device Plugin Pod status
kubectl get pods -n kube-system -o wide | grep nvidia
```

- Result: The Device Plugin Pod is in a CrashLoopBackOff state, indicating that it cannot register the GPU resources.  

---

## Analysis
1. The hardware itself is not damaged, but the fan failure causes abnormal GPU temperatures.  
2. The NVIDIA driver is not loaded, preventing device recognition.  
3. The Kubernetes Device Plugin fails to register the GPU due to the issue with the driver.  
4. As a result, Pod requests for GPU resources cannot be scheduled.  

---

## Resolution Steps
1. **Hardware Repair**  
   - Inspect and clean the GPU fan, then restore power supply.  
   - Restart the node to check if the GPU is now recognized.  

2. **Driver Installation**  
   - Uninstall the outdated NVIDIA driver.  
   - Install the latest version of the NVIDIA driver compatible with the kernel.  
   - Load the NVIDIA kernel module: `modprobe nvidia`.  

3. **Container and Kubernetes Verification**  
   - Ensure that `nvidia-smi` displays correct GPU information.  
   - Restart the Device Plugin Pod to re-register the GPU resources.  
   - Verify that Pods can be successfully scheduled to the GPU node.  

---

## Outcome
- The `nvidia-smi` command on the GPU node now outputs normal results, and both temperature and fan status are normal.  
- The Device Plugin Pod is running correctly.  
- User deep learning Pods can now be scheduled and executed successfully on the GPU node.  

---

## Key Points
- GPU troubleshooting follows a systematic approach: **hardware → driver → container → Kubernetes scheduling**.  
- Pay attention to factors such as fan performance, power supply, and temperature to prevent overheating.  
- Driver incompatibility with the kernel is a common issue.  
- Monitoring the status of Kubernetes Device Plugins is crucial for identifying failures.  
- This case study can be used as an example in interviews to demonstrate systematic troubleshooting skills.  

---

## Interview Answer Example
> “During a GPU node failure, user Pods experienced prolonged `Pending` states with the error ‘Requested device nvidia.com/gpu not found’. I followed these steps: First, I checked the hardware and found that the fan was faulty, causing abnormal temperatures. Next, I verified the driver and discovered that the kernel module was missing. Then, I tested container access and saw that `nvidia-smi` reported an issue. Finally, I examined the Kubernetes Device Plugin and found that it could not register the GPU.  
> To resolve this, I repaired the fan, installed a compatible NVIDIA driver, and restarted the Device Plugin. After verification, Pods were successfully scheduled to the GPU node. This case demonstrates a comprehensive approach to GPU troubleshooting from hardware to Kubernetes.”