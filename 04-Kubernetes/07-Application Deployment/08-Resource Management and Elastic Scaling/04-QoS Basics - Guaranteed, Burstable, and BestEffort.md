# 04-QoS Basics: Guaranteed, Burstable, and BestEffort

## Document Description
- Document Purpose: An introductory guide to Kubernetes resource quality of service levels
- Target Audience: Those who have already understood requests/limits, resource-driven scheduling, and Pod Pending status, moving on to QoS, resource competition, and node resource pressure basics
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/08-Resource Management and Scaling/04-QoS Basics: Guaranteed, Burstable, and BestEffort`

## Tags
#Kubernetes #QoS #Guaranteed #Burstable #BestEffort #requests #limits #Eviction #ResourceManagement #PodScheduling #ApplicationDeployment #CloudNative #Ops #InterviewNotes

---

## I. Why Learn QoS Now

The previous sections have established several key concepts:

- `requests` affect Pod scheduling
- Pods may become `Pending` due to insufficient resources
- `limits` are more about runtime resource constraints
- When node resources are tight, not all Pods are treated equally

However, a practical issue arises:

- Why do some Pods running on the same node perform more stably?
- Why do certain Pods face issues more easily when resources become scarce?
- Why is it generally not recommended to omit resource configurations in production environments?
- Why do requests/limits impact not only scheduling but also runtime resource availability?

The core of these questions lies in:

> **QoS (Quality of Service)**

Here, we are not referring to network QoS or traffic shaping. Instead, we mean:

> **Kubernetes' automatic classification of Pod resource guarantees based on their configuration settings.**

Learning QoS is important because it helps you understand:

- Why different Pods receive varying treatment during resource shortages
- Why some Pods are more likely to be evicted
- How proper resource configurations affect business stability
- It lays the foundation for understanding concepts like Eviction, OOM, HPA, and resource governance.

---

## II. What Exactly is QoS

QoS can be simply understood as:

> **Kubernetes' automatically calculated resource guarantee level for a Pod based on its container's requests/limits settings.**

It is closely related to the following fields:

- `resources.requests.cpu`
- `resourcesrequests.memory`
- `resources.limits.cpu`
- `resources.limits.memory`

Based on these configurations, Kubernetes assigns Pods to one of three QoS levels:

- `Guaranteed`
- `Burstable`
- `BestEffort`

### Key Points for Ops Professionals

QoS is not a field you manually add to YAML. You won’t write something like:

    qosClass: Guaranteed

Instead, it is automatically determined by Kubernetes based on your resource configurations.

---

## III. Why QoS Matters So Much

QoS is crucial because it influences the priority during runtime resource competition.

For example:

### 1. Which Pods receive preferential treatment when node resources are tight
This is closely related to QoS levels.

### 2. Why some temporary test Pods fail quickly under resource pressure
This often stems from their lower QoS levels.

### 3. Why it’s recommended to configure requests/limits properly for critical services
It affects both scheduling and resource availability during peak times.

### 4. The high risks of omitting resource configurations in production environments
It usually results in the Pod being assigned the lowest level of resource guarantees.

### Key Points for Ops Professionals

Think of QoS as:

> **Kubernetes’ way of categorizing Pods based on their resource management practices.**

---

## IV. The Three QoS Levels in Kubernetes

The three most common QoS levels are:

### 1. `Guaranteed`
This level offers the highest level of resource guarantees.

### 2. `Burstable`
It falls in between and is the most commonly encountered level.

### 3. `BestEffort`
This level provides the lowest level of resource guarantees.

### Brief Recall

- `Guaranteed`: Clear resource boundaries with strong guarantees
- `Burstable`: Some resource declarations, but not at the highest level
- `BestEffort`: No specific resource declarations, resulting in minimal guarantees

---

## V. What is Guaranteed QoS

### Quick Summary

For a Pod to be classified as `Guaranteed`, each container must have:

- Equal `requests` and `limits` values for both CPU and memory

For example:

    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"

### Why Higher Guarantees

This configuration ensures clear resource boundaries, making it easier for Kubernetes to manage the Pod’s resources.

### Key Points for Ops Professionals

The focus of `Guaranteed` is not on the amount of resources requested but on:

> **Complete, strict, and aligned resource declarations.**

            limits:
              cpu: "1"
              memory: "512Mi"

### Why is it Burstable?

Although resource configurations are specified, the following conditions apply:

- CPU request ≠ CPU limit
- Memory request ≠ memory limit

Therefore, it does not meet the strict requirements of Guaranteed. However, since some resource fields are still defined, it is not classified as BestEffort.

Thus, it belongs to:

> **Burstable**

---

## Example 11: A BestEffort Pod

    apiVersion: v1
    kind: Pod
    metadata:
      name: besteffort-pod
    spec:
      containers:
        - name: app
          image: nginx:1.27

### Why is it BestEffort?

Because the following fields are completely absent:

- requests
- limits

As a result, Kubernetes categorizes it as:

> **BestEffort**

---

## How to Check a Pod’s QoS Level

There are two common methods:

### 1. View Pod Details

    kubectl describe pod <pod-name> -n <namespace>

The output usually includes:

    QoS Class: Burstable

or:

    QoS Class: Guaranteed

---

### 2. View YAML Configuration

    kubectl get pod <pod-name> -n <namespace> -o yaml

You can typically find the following field:

    qosClass: Burstable

### Key Points for Operations Engineers

When troubleshooting, don’t just guess which QoS category a Pod belongs to; directly check its actual resource allocation.

---

## Example 13: Creating Pods with Three Different QoS Levels

Here is a simple experiment where three types of QoS configurations are defined in the same YAML file:

    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-guaranteed
    spec:
      containers:
        - name: app
          image: nginx:1.27
          resources:
            requests:
              cpu: "200m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-burstable
    spec:
      containers:
        - name: app
          image: nginx:1.27
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-besteffort
    spec:
      containers:
        - name: app
          image: nginx:1.27

### Applying the Configurations

    kubectl apply -f qos-demo.yaml

---

## How to Verify the Results

### 1. Check if Pods were Created Successfully

    kubectl get pod

You should see:

- `qos-guaranteed`
- `qos-burstable`
- `qos-besteffort`

---

### 2. Verify Each Pod’s QoS Class

    kubectl describe pod qos-guaranteed
    kubectl describe pod qos-burstable
    kubectl describe pod qos-besteffort

You will typically observe:

#### Guaranteed Pod
    QoS Class: Guaranteed

#### Burstable Pod
    QoS Class: Burstable

#### BestEffort Pod
    QoS Class: BestEffort

### Key Points for Operations Engineers

The value of this experiment lies not in just running the Pods but in:

> **transforming QoS from an abstract concept into a tangible attribute that you can observe directly.**

---

## Why Many Services Do Not Use Guaranteed QoS

Although Guaranteed seems the most reliable option, many real-world services do not configure it initially.

Common reasons include:

### 1. Fluctuating Resource Usage

If requests and limits are set identically, it may become too rigid.

### 2. Need to Separate Baseline and Peak Requirements

For example:

- Set a baseline value for requests
- Allow limits to increase during peak periods

### 3. Need to Balance Resource Utilization

If all services were guaranteed the same level of resources, overall utilization might not be optimal.

### Key Points for Operations Engineers

Therefore, many production services ultimately opt for:

> **Burstable**

This is not a configuration error but a common compromise.

---

## Why BestEffort Represents the Highest Risk

The main issue with BestEffort is not that it cannot run at all, but rather that:

> **it offers almost no guaranteed resource allocation.**

### Potential Risks Include:

#### 1. More Uncertain Scheduling and Operation

Since there are no fixed limits on resource requests, scheduling and operation become more unpredictable.

#### 2. Higher Vulnerability## 22. Key Understandings in This Chapter

### 1. QoS is a level of resource assurance automatically calculated by Kubernetes
It is not a field that can be manually specified in YAML.

### 2. QoS is divided into three categories:
- `Guaranteed`
- `Burstable`
- `BestEffort`

### 3. The `Guaranteed` category has the strictest requirements
Both the `requests` and `limits` for CPU/memory must be fully specified and equal.

### 4. The `Burstable` category is the most common
Many production services are assigned to this category.

### 5. The `BestEffort` category provides the lowest level of resource assurance
Usually, neither `requests` nor `limits` are specified.

### 6. QoS is more related to runtime resource assurance and eviction priorities
It should not be confused with the scheduling phase.

### 7. It is generally not recommended to leave resource configurations unspecified for a long time in production services
Otherwise, the service is likely to be assigned to the `BestEffort` category, which carries higher risks.

---

## 23. What Level of Mastery Is Expected After Studying This Chapter

At this stage, it is recommended to achieve the following:

### 1. Be able to clearly explain what QoS is.
### 2. Distinguish between `Guaranteed`, `Burstable`, and `BestEffort`.
### 3. Understand three basic YAML examples.
### 4. Use `kubectl describe pod` to view the QoS Class of a Pod.
### 5. Comprehend the fundamental relationship between QoS, resource pressure, and eviction.
### 6. Understand why `BestEffort` carries higher risks.

---

## 24. Common Follow-up Questions in Interviews

Common questions include:

- What are the categories of Kubernetes QoS?
- What is the difference between `Guaranteed`, `Burstable`, and `BestEffort`?
- What kind of Pods are assigned to the `Guaranteed` category?
- Why are many production service Pods classified as `Burstable`?
- What are the risks associated with `BestEffort`?
- What is the relationship between QoS and `requests/limits`?
- How is QoS related to eviction?
- How can I check the QoS level of a Pod?

---

## 25. Summary of This Chapter

QoS is a crucial aspect of Kubernetes' resource management system.

The most important thing to understand from this chapter is not to memorize definitions, but to establish the following key concepts:

- `requests/limits` affect not only scheduling but also the level of runtime resource assurance.
- Kubernetes automatically classifies Pods into `Guaranteed`, `Burstable`, and `BestEffort`.
- Generally, higher QoS means stronger resource assurance, while lower QoS makes a Pod more vulnerable under resource constraints.
- Production services should not rely on `BestEffort` for a long time, as it provides almost no guaranteed resources.

With these concepts in place, further studies on topics such as eviction, node resource pressure, OOM killing, and HPA for resource fluctuations will be easier to understand.

---

## 26. Key Terms for Quick Reference

- QoS: Level of Kubernetes resource service quality.
- Guaranteed: Highest level of assurance; `requests` and `limits` are complete and equal.
- Burstable: Intermediate level; most common.
- BestEffort: Lowest level of assurance; usually no resource declarations.
- requests: Resource requirements, used for scheduling.
- limits: Resource upper limits, used for runtime constraints.
- Eviction: Process of removing a Pod due to node resource pressure.
- QoS Class: Current QoS level of a Pod.

---

## 27. Operational Perspectives on QoS

From an operational standpoint, QoS marks an important step in Kubernetes' evolution from simply ensuring that Pods can run to distinguishing different levels of support for various workloads.

Without the concept of QOS, you might only focus on whether a Pod is running, whether there are `requests` and `limits`, or whether it will encounter OOM issues. However, with QOS, you can understand why different Pods behave differently under resource constraints, why critical services require more stringent resource management, why testing workloads cannot be configured as casually as production ones, and why resource configurations affect both scheduling and runtime stability.

Therefore, although this chapter appears to focus on classification, it actually connects various concepts such as `requests/limits`, scheduling, runtime resource competition, node pressure, eviction priorities, and service stability. It is a fundamental step in understanding Kubernetes' resource governance system.

---

## References
- Kubernetes Pod Quality of Service Classes
- Kubernetes Resource Management for Pods and Containers
- Common output analysis using `kubectl describe pod`

---

## Next Steps
It is recommended to review the following material next:

[[05-Node Resource Pressure and