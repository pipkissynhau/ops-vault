# 05 - Basics of Node Resource Pressure and Eviction Mechanism: Introduction to Eviction

## Document Description
- Document Location: Practical basics of node resource pressure and Pod eviction
- Applicable Stage: After understanding requests/limits, Pod Pending, and QoS, move on to node resource pressure, eviction phenomena, and basic troubleshooting
- Recommended Path: `04-Kubernetes/07-Application Deployment/08-Resource Management and Auto Scaling/05-Basics of Node Resource Pressure and Eviction Mechanism: Introduction to Eviction`

## Tags
#Kubernetes #Eviction #NodePressure #MemoryPressure #DiskPressure #PIDPressure #QoS #requests #limits #ResourceManagement #ApplicationDeployment #CloudNative #Ops #InterviewNotes

---

## I. Why Learn about Eviction Now

The previous main content has established these key understandings:

- `requests` affect Pod scheduling
- When resources are insufficient, Pods may become `Pending`
- `QoS` affects the resource guarantee level of Pods

But at this point, you might encounter a very practical issue:

- A Pod has successfully started on a node
- The service has been running normally for some time
- Later, the node's resources become increasingly strained
- Suddenly, some Pods are evicted

In such cases, many people's first reaction might be:

- The application itself crashed
- The probes failed
- There is a problem with the image
- The container exited on its own

But the real situation may be different.  
Sometimes, what actually happens is:

> **The node's resources are under too much pressure, and kubelet actively evicts some Pods to protect the node itself.**

This is what we call:

> **Eviction.**

The significance of learning about Eviction lies in:

- Understanding why a Pod might be "evicted" by the system while it is still running
- Comprehending the relationship between node resource pressure and service stability
- Realizing that QoS is more than just a classification label
- Learning to identify eviction signs in `describe pod` and `describe node` commands
- Laying the foundation for understanding OOM, HPA stability, and node capacity management

---

## II. First, Let's Look at the Phenomena: What Does Eviction Look Like?

When you first encounter Eviction, it is usually not through its definition but rather through its observable phenomena.

Common phenomena include:

### 1. A Running Pod Suddenly Disappears
You execute:

    kubectl get pod -n <namespace>

and find that the original Pod is gone, and the controller has started a new one in its place.

---

### 2. Temporary Service Disruption
Especially when:

- The number of replicas is small
- New Pods take a long time to start
- The readinessProbe requirements are strict

The service may experience temporary unavailability at this time.

---

### 3. "Evicted" Appears in `describe Pod` Results
For example:

    kubectl describe pod <pod-name> -n <namespace>

You can find eviction-related information in the status or events section.

---

### 4. High Pressure Levels are Reported in `describe Node`
For example:

- `MemoryPressure=True`
- `DiskPressure=True`
- `PIDPressure=True`

### Key Points for Ops Professionals
Eviction is rarely a "random failure"; it is more often a sign that:

> **The node has entered a critical state, and Kubernetes has started to proactively remove some loads.**

---

## III. What Exactly Is Eviction?

Eviction can be simply understood as:

> **When a node's resources are under too much pressure, Kubernetes actively removes some Pods from that node to protect the node itself.**

The key point here is not that:

- The application crashes on its own
- The container terminates unexpectedly
- The Deployment scales in automatically

but rather that:

> **The node or kubelet determines that the resource pressure is too high, and if nothing is done, it will affect the overall stability of the system.**

### How to Understand It
You can think of it this way:

- A node is like a ship
- Pods are the cargo on that ship
- When the ship is about to sink, some cargo must be discarded first to save the entire ship

In such situations, Kubernetes prioritizes:

> **Saving the node before trying to preserve the services running on it.**

Because if the node itself cannot function properly, all the Pods on it will be affected.

---

## IV. Are Eviction and OOMKilled the Same Thing?

They are not the same thing, but they are both related to "resource constraints," so they are often confused.

### 1. What is OOMKilled?
It relates more to:

> **Memory issues at the container level**

It usually means that:

- The actual memory usage of a container exceeds its limit
- Or the container is terminated due> Different QoS levels → Different levels of resource assurance

Make sure you understand this premise clearly.

---

## Nine: How to Check if a Node Is Under Resource Pressure

One of the most crucial commands is:

    kubectl describe node <node-name>

### Focus on Conditions

You will usually see something like this:

    Conditions:
      Type             Status
      MemoryPressure   False
      DiskPressure     False
      PIDPressure      False
      Ready            True

If a certain type of resource pressure occurs, you might see:

    MemoryPressure   True

Or:

    DiskPressure     True

### What Does This Mean?

#### `MemoryPressure=True`
The node is experiencing high memory pressure.

#### `DiskPressure=True`
The node is under significant disk pressure.

#### `PIDPressure=True`
The node is running out of PID resources.

### Key Points for Operations and Maintenance Professionals

When troubleshooting a node:

> **`kubectl describe node` is one of the first steps in understanding Eviction scenarios.**

---

## Ten: How to Check if a Pod Has Been Evicted

There are two common methods:

### 1. View the Pod List

    kubectl get pod -n <namespace>

If a Pod has been evicted, you might observe:

- The original Pod is no longer present.
- A new Pod has been created by the controller.
- Some Pods may show abnormal status.

---

### 2. View Pod Details

    kubectl describe pod <pod-name> -n <namespace>

Pay attention to:

- Status
- Events
- Any notes related to eviction

In some cases, you might see:

- `Evicted`
- Messages indicating resource pressure issues.

### Key Points for Operations and Maintenance Professionals

If you notice that a Pod is being terminated not due to an application error, probe failure, or image issue, then you should consider the possibility of:

> **Node resource pressure / Eviction**

---

## Eleven: A Real-World Troubleshooting Example: What Does an Evicted Pod Look Like?

Suppose a node is experiencing increasing memory pressure.

First, execute:

    kubectl get pod -n test -o wide

You might see that a certain Pod was previously running on this node:

- `worker-abc123` was on `node-1`

Later, when you check again, you may find:

- The Pod is no longer there.
- A new `worker-xyz456` has been created by the controller.

If you then describe the original Pod, you might see evidence of eviction.

Next, check:

    kubectl describe node node-1

You are likely to see:

- `MemoryPressure=True`
- Or related events indicating resource pressure.

### Key Points for Operations and Maintenance Professionals

The goal here is not to memorize commands, but to establish a troubleshooting process:

### 1. Notice any abnormalities in Pod disappearance/recreation.
### 2. Rule out issues with images, probes, or application failures.
### 3. Check Pod details and node status.
### 4. Analyze resource pressure conditions to determine if eviction occurred.

---

## Twelve: What Is the Relationship Between Eviction and requests/limits?

This section is very important.

### 1. requests Affect the Resource Assurance Baseline
If a Pod specifies requests, it indicates:

- A declared resource requirement.
- Kubernetes can better understand its minimum resource needs.

This will affect both QoS classification and priority during resource shortages.

---

### 2. limits Determine the Maximum Resources Available
limits determine:

- The maximum amount of resources a container can use.

If limits are set too high or too loosely, or if resource management is poor, it may indirectly increase node resource pressure.

---

### 3. requests/limits Together Affect QoS
QoS, in turn, affects:

- The level of resource assurance under different node pressure scenarios.

### Key Points for Operations and Maintenance Professionals

Thus, requests/limits affect not only:

- Scheduling.
- Out-of-memory killings (OOM).

They also influence:

- QoS.
- The risk of eviction under high-pressure conditions.

---

## Thirteen: Are Eviction and Pending the Same Thing?

No.

### `Pending`
Refers more to:

> **The Pod has not yet been deployed.**

That is, it’s a problem before or during scheduling.

---

### `Eviction`
Refers to:

> **The Pod has already started running, but the node becomes too resource-constrained and forces it to be terminated.**

This occurs during runtime.

### You Can Remember It This Way:

- `Pending`: The Pod hasn’t been deployed yet.
- `Eviction`: The Pod has been deployed but later terminated due to resource constraints.

This distinction is very important.

---

## Fourteen: What Is the Relationship Between Eviction and HPA?

This is a practical issue.

### 1. HPA Can Expand Pods
It addresses:

> **Whether to increase the number of replicas.**

---

The most important thing in this section is not to memorize various thresholds by rote, but rather to establish the following core understandings first:

- When node resources are under excessive pressure, Kubernetes will proactively evict some Pods.
- The primary goal of eviction is to protect the node itself.
- OOMKilled is related to eviction, but it is not the same thing.
- QoS affects a Pod's "survivability" under resource constraints.
- requests/limits are not just scheduling fields; they also continue to impact operational stability during runtime.
- When troubleshooting eviction issues, it is essential to consider three aspects: the Pod, the node, and resource configuration.

Once these understandings are established, you can then proceed with learning about the following topics:

- HPA
- Node capacity planning
- Resource governance
- Business high availability

This will provide a more comprehensive perspective.

---

## 22. Key Term Summary

- Eviction: Proactive removal of Pods when node resources are under pressure.
- MemoryPressure: Node memory stress.
- DiskPressure: Node disk stress.
- PIDPressure: Node process ID stress.
- QoS: Resource quality assurance level.
- Guaranteed: High-level assurance.
- Burstable: Medium-level assurance.
- BestEffort: Low-level assurance.
- OOMKilled: Abnormal exit due to container-level memory issues.
- describe node: An important command for checking node status.

---

## 23. Operational Insights

From an operational perspective, eviction serves as a reminder from Kubernetes that:

> **Platform stability always takes precedence over the survival of individual Pods.**

If a node is on the verge of failing, Kubernetes' primary goal is not to "keep every Pod running," but rather to:

- Preserve the node.
- Maintain the kubelet.
- Ensure the basic operating environment remains stable.
- Try to prevent the entire system from collapsing.

Therefore, eviction is not some "extra obscure fact" but a critical component in understanding Kubernetes's resource governance approach. It ties together all the concepts we have learned so far:

- requests/limits
- QoS
- Node capacity
- Business stability
- Resource pressure
- Operational management

For this reason, eviction is not just a standalone topic; it marks a crucial step in moving from simply deploying Pods to understanding the relationship between nodes and business operations.

---

## References

- Kubernetes Node-pressure Eviction
- Kubernetes Pod Quality of Service Classes
- Common outputs of kubectl describe node for troubleshooting
- Kubernetes Resource Management for Pods and Containers

---

## Next Step Suggestion

For the next article, it is recommended to organize the following content:

[[06-HPA Basics: Introduction to Pod Auto Scaling]]