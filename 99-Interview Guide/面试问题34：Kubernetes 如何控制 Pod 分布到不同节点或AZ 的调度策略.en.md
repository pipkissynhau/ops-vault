# Interview Question 34: How Does Kubernetes Control the Scheduling Strategy for Pods to Be Distributed across Different Nodes or AZs

#kubernetes #scheduling #affinity #highavailability #interview

---

## I. Core Summary (One-sentence Answer for Interviews)

In Kubernetes, if you want the pods created by controllers such as Deployments to be distributed across different nodes or zones, you can use:

- **podAntiAffinity**: To prevent pods from being on the same node.
- **topologySpreadConstraints**: To achieve even distribution (recommended).
- **nodeAffinity**: To control which nodes pods should run on.

---

## II. The Essence of the Question (What Interviewers Are Trying to Test)

The essence of such questions is:

> ❗How to prevent pods from concentrating on the same node or zone, thereby enhancing system high availability?

---

## III. The Most Common Solution: podAntiAffinity

### 1. Core Function

It prevents pods of the same application from being scheduled to the same node.

---

### 2. Example (Preventing on the Same Node)

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

- `topologyKey = kubernetes.io/hostname` → This refers to the node level.
- **Meaning**: Pods with the same label are not allowed to be on the same node.

---

## IV. Cross-AZ Scheduling (Key in Cloud Environments)

### 1. How AZs Are Represented in K8s

AZs are indicated through Node Labels:

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

> Pods will be distributed across different availability zones as much as possible.

---

## V. Recommended Solution: topologySpreadConstraints (More Modern)

### 1. Function

It ensures that pods are **evenly distributed** across different nodes or AZs.

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
|maxSkew|Maximum allowed imbalance in distribution|
|topologyKey|Distribution dimension (node/zone)|
|whenUnsatisfiable|Behavior when the constraints are not met|

---

## VI. nodeAffinity (Supplementary)

### Function

It controls which nodes pods should run on.

---

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
      - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values
```