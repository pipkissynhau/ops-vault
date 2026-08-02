---
tags: "[Kubernetes, PVC, PV, StorageClass, Storage, Interview]"
---

# Interview Question 13: Relationship, Scope, and Automatic Binding Between PVC, PV, and StorageClass

## Description
In Kubernetes, storage resources mainly involve three objects:

| Object | Full Name | Role | Scope |
|--------|-----------|------|-------|
| **PV** | PersistentVolume | Actual storage volume, created by administrators or via dynamic provisioning | Cluster-wide |
| **PVC** | PersistentVolumeClaim | User request for storage, declaring required capacity and access mode | Namespace-level |
| **SC** | StorageClass | Defines storage type, parameters, and dynamic provisioning rules | Cluster-wide |

### Relationship
1. **PVC Requests PV**: When users create a PVC within a namespace, Kubernetes attempts to find a matching PV and bind it.  
2. **StorageClass Enables Dynamic Provisioning**: If no matching PV exists, a PVC can specify a StorageClass, which automatically creates a PV and binds it via the StorageClass.  
3. **Binding Process**:
   - PVC Status: `Pending` → Waiting for PV  
   - Find matching PV or dynamically create PV via StorageClass  
   - PVC binds to PV, status becomes `Bound`

## Operation Examples

```yaml
# StorageClass Example:
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2

# PVC Example:
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
# View PVC Status
kubectl get pvc -n dev

# View PV Status
kubectl get pv
```

## Key Summary

- **PV**: Cluster-level, actual volume resource  
- **PVC**: Namespace-level, user request  
- **StorageClass**: Cluster-level, defines dynamic provisioning rules  
- PVC Automatically Binds PV: Kubernetes matches PV based on requested capacity, access mode, and StorageClass. If no match is found, StorageClass dynamically creates PV  
- Status Change: `Pending` → `Bound`  

## Interview Answer Example

> "In Kubernetes, PV is a cluster-level storage volume resource, PVC is a namespace-level storage request, and StorageClass defines storage types and dynamic provisioning rules. When users create a PVC, the system attempts to find a matching PV. If none exists, it dynamically creates a PV via the StorageClass specified in the PVC and binds it. The PVC status transitions from Pending to Bound. This mechanism allows users to abstract away the details of volume creation and management, simply declaring required capacity and access mode in the namespace to achieve automatic binding."