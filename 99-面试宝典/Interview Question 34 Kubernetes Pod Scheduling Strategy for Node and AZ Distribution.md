# Interview Question 34: How Kubernetes Controls Pod Distribution Across Different Nodes or AZs in Scheduling Strategies

#kubernetes #MovementControl #affinity #HighAvailable #Interviews

---

## I. Core Summary (One-Sentence Interview Version)

In Kubernetes, to distribute Pods created by controllers (such as Deployments) across different nodes or availability zones (AZs), you can use:

- **podAntiAffinity (Anti-Affinity)**: Prevents Pods from being scheduled on the same node
    
- **topologySpreadConstraints**: Achieves even distribution (recommended)
    
- **nodeAffinity**: Controls which nodes Pods run on
    

---

## II. Core Essence (What the Interviewer is Testing)

The essence of this question is:

> ❗How to avoid Pods concentrating on the same node or same AZ, improving system high availability

---

## III. Most Common Solution: podAntiAffinity (Anti-Affinity)

### 1. Core Function

Prevents Pods of the same application from being scheduled on the same node

---

### 2. Example (Avoid Same Node)

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: my-app
        topologyKey: kubernetes.io/hostname
```

---

### 3. Explanation

- topologyKey = kubernetes.io/hostname → Node-level dimension
    
- Meaning: Pods with the same label are not allowed on the same Node
    

---

## IV. Cross-AZ Scheduling (Cloud Environment Focus)

### 1. AZ Representation in K8s

AZ is represented through Node Labels:

```bash
kubectl get nodes --show-labels
```

Example:

```text
topology.kubernetes.io/zone=ap-southeast-1a
```

---

### 2. Cross-AZ Example

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: my-app
        topologyKey: topology.kubernetes.io/zone
```

---

### 3. Meaning

> Pods will be distributed across different availability zones (AZs) as much as possible

---

## V. Recommended Solution: topologySpreadConstraints (More Modern)

### 1. Function

Evenly distributes Pods across different nodes or AZs

---

### 2. Example

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app
```

---

### 3. Parameter Explanation

|Parameter|Meaning|
|---|---|
|maxSkew|Maximum imbalance value|
|topologyKey|Distribution dimension (node / AZ)|
|whenUnsatisfiable|Behavior when constraints are not met|

---

## VI. nodeAffinity (Supplement)

### Function

Controls which nodes Pods run on (node filtering)

---

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
      - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values
``` /think