---
tags: "[Kubernetes, StatefulSet, PVC, PV, Storage, Interview]"
---

# Interview Question 16: StatefulSet and PV Binding Mechanism

## Explanation
StatefulSet is used to manage stateful applications, where each Pod has **unique identity** and typically requires **persistent storage**.  
Kubernetes implements storage binding for StatefulSet Pods through **PVC (namespace-level) and PV (cluster-level)**.

### Key Features
1. Each Pod has unique name (`<statefulset-name>-<ordinal>`)  
2. Each Pod corresponds to an independent PVC  
3. PVC is one-to-one bound with PV, ensuring data isolation and persistence  

## Operation Example

```yaml
# StatefulSet Example:
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
      storageClassName: fast-storage
```

```bash
# View StatefulSet Pod Corresponding PVC
kubectl get pvc -n <namespace>

# View PV Tie Status
kubectl get pv
```

## Key Takeaways

- Each Pod in a StatefulSet has a unique PVC, ensuring independent persistent storage  
- PVC automatically binds to PV, which can be dynamically created via StorageClass  
- PVC remains unchanged when Pod is deleted and recreated, maintaining data persistence  
- In interviews, emphasize the difference between StatefulSet and Deployment: Deployment does not guarantee Pod unique identity or data persistence  

## Interview Answer Example

> "StatefulSet is used for stateful applications, ensuring each Pod has a unique identity. Each Pod creates an independent PVC, binding to PV through PVC to achieve persistent storage. PVC can specify StorageClass; if no existing PV is available, Kubernetes will dynamically create one.  
> The advantage of StatefulSet is that PVC remains unchanged even after Pod recreation or migration, preserving data persistence. Compared to Deployment, StatefulSet is more suitable for stateful services like databases and message queues."