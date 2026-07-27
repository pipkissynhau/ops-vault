---
tags: [Kubernetes, PVC, PV, StorageClass, Storage, Interview]
---

# Interview Question 13: The Relationship, Scope, and Automatic Binding of PVCs, PVs, and StorageClasses

## Explanation
In Kubernetes, storage resources mainly involve three objects:

| Object | Full Name | Function | Scope |
|------|--------------|-----------|----------|
| **PV** | PersistentVolume | Actual storage volume, created by administrators or dynamic provisioning | Cluster level |
| **PVC** | PersistentVolumeClaim | User's request for storage, specifying required capacity and access mode | Namespace level |
| **SC** | StorageClass | Defines storage type, parameters, and dynamic provisioning rules | Cluster level |

### Relationship
1. **PVC Requests PV**: When a user creates a PVC in a namespace, Kubernetes attempts to find a suitable PV to bind it to.
2. **StorageClass for Dynamic Provisioning**: If no suitable PV is available, the StorageClass specified by the PVC can be used to dynamically create and bind a PV.
3. **Binding Process**:
   - PVC status: `Pending` → Waiting for a PV
   - Finding a matching PV or dynamically creating one through the StorageClass
   - Binding the PVC to the PV, with the status changing to `Bound`

## Example Operations

```yaml
# Example of StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2

# Example of PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: dev
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-storage
```

```bash
# Checking PVC status
kubectl get pvc -n dev

# Checking PV status
kubectl get pv
```

## Key Points Summary

- **PV**: Cluster-level, actual volume resource
- **PVC**: Namespace-level, user request for storage
- **StorageClass**: Cluster-level, defines dynamic provisioning rules
- Automatic binding of PVC to PV: Kubernetes matches the requested capacity, access mode, and StorageClass. If no suitable PV exists, a new PV is dynamically created based on the StorageClass.
- Status transition: `Pending` → `Bound`

## Sample Interview Answer

> “In Kubernetes, a PV represents the actual storage volume at the cluster level, while a PVC is a user-defined request for storage within a namespace. A StorageClass defines the type of storage and specifies how PVs should be dynamically created when needed. When a user creates a PVC, Kubernetes tries to find an existing PV that matches the requirements. If none is found, it uses the StorageClass to create a new PV and bind it to the PVC. The status of the PVC changes from `Pending` to `Bound` once the binding is successful. This mechanism allows users to manage storage resources without having to worry about the details of volume creation and management.”