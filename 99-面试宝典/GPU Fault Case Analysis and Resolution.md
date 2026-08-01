---
tags: "[GPU, Troubleshooting, Operation and Maintenance Cases, Kubernetes, Interview]"
---

# GPU Fault Case Analysis and Handling

## Fault Scenario
- There is a GPU node in the cluster, and the user's submitted deep learning Pod cannot start, with the container status long being `Pending`.  
- Logs show: `Requested device nvidia.com/gpu not found`.  
- The command `nvidia-smi` running on the node cannot normally output GPU information.  

---

## Troubleshooting Steps

### 1. Hardware Layer Check
```bash
# View GPU Is it systematically recognized?
lspci | grep -i nvidia

# Inspection GPU Status and temperature
nvidia-smi
```

- Discovery: The system does not recognize the GPU, hardware seems normal, but the fan does not spin, temperature display is abnormal.  

---

### 2. Driver / Software Layer Check
```bash
# View NVIDIA Driver Version
cat /proc/driver/nvidia/version

# Check kernel module
lsmod | grep nvidia
```

- Discovery: Driver version is incompatible with the operating system kernel, nvidia module is not loaded.  

---

### 3. Container Access Check
```bash
docker run --rm --gpus all nvidia/cuda:12.1-base nvidia-smi
```

- Result error, indicating GPU device not found.  

---

### 4. Kubernetes Layer Check
```bash
# View Nodes GPU Resources
kubectl describe node gpu-node

# View Device Plugin Pod Status
kubectl get pods -n kube-system -o wide | grep nvidia
```

- Discovery: Device Plugin Pod CrashLoopBackOff, unable to register GPU resources  

---

## Analysis

1. Hardware temporarily not damaged, but fan failure leads to abnormal GPU temperature  
2. NVIDIA driver not loaded, causing device unrecognizable  
3. Kubernetes Device Plugin crashes due to unavailable device  
4. Pod request GPU resources cannot be scheduled  

---

## Handling Process

1. **Hardware Handling**  
   - Check GPU fan, clean dust and restore power  
   - Reboot node, observe if GPU is recognized  

2. **Driver Handling**  
   - Uninstall old NVIDIA driver  
   - Install latest NVIDIA driver compatible with operating system kernel  
   - Load NVIDIA kernel module: `modprobe nvidia`  

3. **Container and Kubernetes Check**  
   - Confirm `nvidia-smi` normally outputs GPU information  
   - Restart Device Plugin Pod to re-register GPU resources  
   - Verify Pod can successfully schedule to GPU node  

---

## Results

- GPU node `nvidia-smi` outputs normally, temperature and fan status normal  
- Device Plugin Pod runs normally  
- User's deep learning Pod successfully scheduled and running  

---

## Key Takeaways

- GPU fault troubleshooting requires **hardware → driver → container → Kubernetes scheduling**  
- Pay attention to fans, power supply and temperature, prevent hardware overheating  
- Driver and kernel version mismatch is a common issue  
- Kubernetes Device Plugin status is a key troubleshooting indicator  
- The troubleshooting process can serve as an interview example, demonstrating systematic fault diagnosis capability  

---

## Interview Answer Example

> "In a GPU node fault case, the user's Pod was long pending with error 'Requested device nvidia.com/gpu not found'.  
> I followed the troubleshooting sequence: first checked hardware, found fan not spinning, temperature abnormal; then checked driver, found kernel module not loaded; then validated container GPU availability, found nvidia-smi error; finally checked Kubernetes Device Plugin, Pod could not register GPU.  
> Handling process: fixed fan issue, reinstalled NVIDIA driver compatible with kernel, restarted Device Plugin, validated GPU normal. Final Pod successfully scheduled. This case demonstrates systematic GPU fault troubleshooting from hardware to K8s."