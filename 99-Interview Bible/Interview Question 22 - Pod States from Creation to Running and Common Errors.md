---
tags: "[Kubernetes, Pod, Status, Lifecycle, Error, Interview]"
---

# Interview Question 24: Pod States from Creation to Running and Common Error States

## Explanation
The Pod lifecycle is managed by Kubernetes controllers and kubelet.  
From creation to Running, a Pod goes through multiple states, each reflecting a different stage of the Pod.

---

## Main Pod States

| Status | Description |
|--------|-------------|
| **Pending** | The Pod has been accepted by the API Server but has not yet been scheduled to a node or is waiting for resources |
| **Scheduled** | The Scheduler has selected a node, but the containers have not yet been created |
| **ContainerCreating** | kubelet is pulling the image and creating the container |
| **Running** | At least one container in the Pod has started and is running |
| **Succeeded** | All containers have exited successfully, and the Pod has terminated (typically used for Jobs) |
| **Failed** | All containers have terminated, and at least one has failed |
| **Unknown** | The Pod status cannot be determined (node unreachable or network issues) |

---

## Common Error States and Causes

| Error Status | Common Causes |
|--------------|---------------|
| **Pending** | Resource insufficiency (CPU/memory), unavailable nodes, PVC not bound to PV, unsatisfied scheduling constraints |
| **CrashLoopBackOff** | Container crashes immediately after startup, possible causes include missing image, configuration errors, environment variable errors, or program anomalies |
| **ImagePullBackOff / ErrImagePull** | Image pull failure, possibly due to non-existent image, repository authentication failure, or network issues |
| **CreateContainerConfigError** | Container configuration error, such as non-existent volume mount, command error, or missing environment variables |
| **OOMKilled** | Container is killed by the kernel due to exceeding memory limit |
| **NodeAffinity / TaintToleration Mismatch** | Pod scheduling failure, node labels or taints do not meet Pod requirements |
| **Unknown** | Node downtime or API Server unable to retrieve Pod status |

---

## Operation / View Pod Status Commands

```bash
# View Pod Status
kubectl get pod <pod-name> -o wide

# View Detailed Events
kubectl describe pod <pod-name>
```

---

## Key Takeaways

- Main Pod lifecycle states: Pending → Scheduled → ContainerCreating → Running → Succeeded/Failed  
- Error states are often related to **resource insufficiency, image pull failure, configuration errors, or scheduling constraints**  
- Use `kubectl describe pod` to view events, enabling quick issue identification  
- Understanding states and error causes helps troubleshoot Pod creation failures or scheduling anomalies  

---

## Interview Answer Example

> "A Pod progresses through Pending, Scheduled, ContainerCreating, and finally Running as it transitions from creation to running.  
> Pending indicates the Pod has been accepted by the API Server but has not yet been scheduled to a node; Scheduled means a node has been selected; ContainerCreating is the phase where kubelet pulls the image and creates the container; Running means the container has started successfully and is operational.  
> Common error states include CrashLoopBackOff (container crash), ImagePullBackOff (image pull failure), CreateContainerConfigError (configuration error), OOMKilled (memory limit exceeded), Pending (resource insufficiency or unsatisfied scheduling constraints), and Unknown (node unreachable). Understanding these states and their causes helps quickly diagnose Pod creation or runtime issues."