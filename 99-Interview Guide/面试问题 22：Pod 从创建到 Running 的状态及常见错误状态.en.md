---
tags: [Kubernetes, Pod, Status, Lifecycle, Errors, Interviews]
---

# Interview Question 24: Pod States from Creation to Running and Common Error Conditions

## Explanation
The lifecycle of a Pod is managed by the Kubernetes controller and kubelet.  
From creation to Running, a Pod goes through several states, each reflecting a different phase of its development.

---

## Main Pod States

| State | Description |
|------|-----------|
| **Pending** | The Pod has been accepted by the API Server but has not yet been scheduled to a node or is waiting for resources. |
| **Scheduled** | The scheduler has selected a node, but the container has not yet been created. |
| **ContainerCreating** | kubelet is pulling the image and creating the container. |
| **Running** | At least one container in the Pod has started and is running. |
| **Succeeded** | All containers have successfully terminated, and the Pod has ended (commonly used with Jobs). |
| **Failed** | All containers have terminated, and at least one failed. |
| **Unknown** | The Pod's status cannot be determined (e.g., node unreachability or network issues).

---

## Common Error States and Their Causes

| Error State | Common Causes |
|-----------|-----------------|
| **Pending** | Insufficient resources (CPU/memory), unavailable nodes, PVC not bound to PV, scheduling constraints not met. |
| **CrashLoopBackOff** | The container crashes immediately after starting, possibly due to missing images, configuration errors, environment variable issues, or program anomalies. |
| **ImagePullBackOff / ErrImagePull** | Image retrieval fails, possibly because the image does not exist, repository authentication is unsuccessful, or there are network problems. |
| **CreateContainerConfigError** | Container configuration errors, such as missing volume mounts, incorrect commands, or missing environment variables. |
| **OOMKilled** | The container uses more memory than allowed and is terminated by the kernel. |
| **NodeAffinity / TaintToleration mismatch** | Pod scheduling fails because node labels or taints do not meet the Pod's requirements. |
| **Unknown** | The node crashes, or the API Server cannot retrieve the Pod's status.

---

## Commands for Operating on/Viewing Pod Status

```bash
# View Pod status
kubectl get pod <pod-name> -o wide

# View detailed events
kubectl describe pod <pod-name>
```

---

## Key Points Summary

- The main lifecycle states of a Pod are: Pending → Scheduled → ContainerCreating → Running → Succeeded/Failed.
- Common error states are often related to **insufficient resources, image retrieval failures, configuration errors, or scheduling constraints**.
- Using `kubectl describe pod` to view events helps quickly identify issues.
- Understanding these statuses and their causes is essential for troubleshooting Pod creation or runtime anomalies.

---

## Sample Interview Answer

> “A Pod goes through several stages from creation to Running: Pending, Scheduled, ContainerCreating, and finally Running. During the Pending phase, the Pod has been accepted by the API Server but hasn’t been assigned to a node; in the Scheduled phase, a node is selected; during ContainerCreating, kubelet retrieves the image and creates the container; and when it reaches the Running state, at least one container has started and is running.  
> Common error states include CrashLoopBackOff (container crashes), ImagePullBackOff (image retrieval failure), CreateContainerConfigError (configuration issues), OOMKilled (memory limit exceeded), Pending (insufficient resources or scheduling constraints), and Unknown (node unreachability). Being familiar with these conditions helps in quickly diagnosing and resolving Pod creation or runtime problems.”