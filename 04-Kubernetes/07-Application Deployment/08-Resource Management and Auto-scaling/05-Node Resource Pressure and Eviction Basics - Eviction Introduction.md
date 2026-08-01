# 05 - Node Resource Pressure and Eviction Mechanism Basics: Eviction Getting Started

## Document Notes
- Document Positioning: Node Resource Pressure and Pod Eviction Basic Practice
- Applicable Stage: After understanding requests / limits, Pod Pending, and QoS, entering node resource pressure, eviction phenomena, and basic troubleshooting
- Recommended Path: `04-Kubernetes/07-Apply deployment/08-Resource management and flexibility/05-Node Resource Pressure and Basis for Expulsion Mechanisms:Eviction Introduction`

## Tags
#Kubernetes #Eviction #NodePressure #MemoryPressure #DiskPressure #PIDPressure #QoS #requests #limits #ResourceManagement #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## One, Why Learn Eviction Now

The previous mainline has established these key understandings:

- `requests` affects Pod scheduling
- Resource shortages may cause Pods to `Pending`
- `QoS` affects Pod resource guarantee level

But here, you'll still encounter a very practical problem:

- Pods have successfully run on the node
- The business has been running normally for some time
- Later, node resources become increasingly tight
- Some Pods are suddenly evicted

At this point, many people's first reaction is:

- The application itself crashed
- The probe failed
- The image is problematic
- The container exited on its own

But the actual situation may not be the case.  
Sometimes what really happens is:

> **Node resource pressure is too high, and kubelet actively evicts some Pods to protect the node itself.**

This is:

> **Eviction (Eviction)**

The significance of learning Eviction is:

- Understand why Pods may be "evicted" by the system while running
- Understand the relationship between node resource pressure and business stability
- Understand why QoS is not just a classification label
- Learn to identify eviction signs from `describe pod`, `describe node`
- Lay the foundation for understanding OOM, HPA stability, and node capacity governance

---

## Two, First Talk About Phenomena: What Does Eviction Look Like

When encountering Eviction for the first time, you often see the phenomenon rather than the definition.

Common phenomena usually include these:

### 1. Running Pods Disappear
You execute:

    kubectl get pod -n <namespace>

And find that the original Pods are gone, with the controller restarting a new Pod.

---

### 2. Brief Business Downtime
Especially when:

- Replica count is low
- New Pod startup is slow
- ReadinessProbe is strict

The business may experience brief unavailability.

---

### 3. Seeing `Evicted` When Describing Pod
For example:

    kubectl describe pod <pod-name> -n <namespace>

You can see eviction-related information in the status or events.

---

### 4. Discovering Pressure Status When Describing Node
For example:

- `MemoryPressure=True`
- `DiskPressure=True`
- `PIDPressure=True`

### Operations Understanding Focus

Eviction is rarely a "random failure," more like:

> **The node has entered a dangerous zone, and Kubernetes starts actively cleaning up some loads.**

---

## Three, What Is Eviction Exactly

Eviction can be initially understood as:

> **When node resource pressure is too high, Kubernetes actively evicts some Pods from the node to protect the node itself.**

The key point is not:

- Application itself exiting
- Container itself crashing
- Deployment itself scaling down

But rather:

> **The node or kubelet believes resource pressure is too high, and if not handled, the entire machine stability will be affected.**

### How to Understand It

You can first understand it as:

- The node is like a ship
- Pods are cargo on the ship
- When the ship is about to sink, some cargo is thrown off to save the ship

Kubernetes prioritizes:

> **Protecting the node first, then trying to protect the business.**

Because if the node itself can't sustain, all Pods on the node will have problems.

---

## Four, Is Eviction the Same as OOMKilled

No, but they are both related to "resource tension," so they are often confused.

### 1. What Is OOMKilled
It leans more toward:

> **Container-level memory issues**

Usually indicates:

- Container actual memory usage exceeds limits
- Or being killed in memory-related scenarios

It's more like:

- A single container issue
- The container runtime being terminated

---

### 2. What Is Eviction
It leans more toward:

> **Node-level resource pressure issues**

Usually indicates:

- Node overall resource tension
- kubelet actively cleaning up some Pods to protect the entire machine

### Operations Understanding Focus

At this stage, just distinguish them as:

- `OOMKilled`: More container-specific memory scenarios
- `Eviction`: More node-level resource pressure scenarios

---

## Five, What Are the Common Node Pressure Types

At this stage, remember the three most common types:

### 1. `MemoryPressure`
Indicates:

> **Node memory pressure is too high**

This is the most common and needs to be understood first.

---

### 2. `DiskPressure`
Indicates:

> **Node disk pressure is too high**

For example:

- Too many images
- Container logs are too large
- Temporary files accumulate
- `nodefs` / `imagefs` are full

---

### 3. `PIDPressure`
Indicates:

> **Node available PID count pressure is too high**

For example:

- Too many processes
- Abnormal thread/process surge
- Some services continuously spawn child processes

### Operations Understanding Focus

These three types of pressure essentially indicate:

> **The node itself has started to be unhealthy or is approaching the dangerous boundary.**

---

## Six, Why Does Node Resource Pressure Trigger Eviction

Because Kubernetes nodes are not just for running business Pods.

Nodes usually run many base components, such as:

- Operating system itself
- kubelet
- container runtime
- Network plugins
- Log-related processes
- Monitoring processes
- Other system daemons

If business Pods keep consuming resources, it may eventually lead to:

- kubelet itself becoming unstable
- Runtime anomalies
- Node slowdown
- Node disconnection
- All business being affected

So Kubernetes' approach is not:

> "Wait until it completely crashes before doing anything"

But rather:

> "Once the node enters a dangerous zone, clean up some Pods first to preserve the basic runtime capability of the entire machine."

---

## Seven, What Is the Relationship Between Eviction and QoS

This is one of the most important points in this article.

Previously, you've learned:

- `Guaranteed`
- `Burstable`
- `BestEffort`

When the node enters a resource pressure state, QoS affects the priority of Pods being processed.

### Current Stage Understanding

#### 1. `BestEffort`
Lowest resource guarantee, usually the most vulnerable.

#### 2. `Burstable`
Middle layer, many business operations are located here.

#### 3. `Guaranteed`
Resource boundaries are most clearly defined, typically with stronger guarantees.

### Can be roughly remembered as

In node resource pressure scenarios, you can generally remember this direction first:

- `BestEffort` is more likely to be evicted first
- `Burstable` is next
- `Guaranteed` is usually more stable

### Operations Understanding Focus

This is not saying:

- Guaranteed will never be evicted
- BestEffort will always be evicted first

Rather, it means:

> **QoS is one of the key factors Kubernetes uses to evaluate the resource guarantee priority of Pods.**

---

## VIII. A Minimal Observation Experiment: Creating Three Types of QoS Pods

If you've already done a QoS experiment in the previous article, you can directly reuse it.

For example, prepare three Pods:

### 1. Guaranteed

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

### 2. Burstable

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

### 3. BestEffort

    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-besteffort
    spec:
      containers:
        - name: app
          image: nginx:1.27

### Apply First

    kubectl apply -f qos-demo.yaml

### Then Check the QoS Class of the Three

    kubectl describe pod qos-guaranteed
    kubectl describe pod qos-burstable
    kubectl describe pod qos-besteffort

You will typically see:

- `QoS Class: Guaranteed`
- `QoS Class: Burstable`
- `QoS Class: BestEffort`

### Operations Understanding Focus

The significance of this experiment is not to immediately create evictions, but to first clarify:

> **Different QoS levels → Different resource guarantee levels**

This premise.

---

## IX. How to Check if a Node Already Has Resource Pressure

One of the most critical commands is:

    kubectl describe node <node-name>

### Focus on Conditions

You will typically see something like:

    Conditions:
      Type             Status
      MemoryPressure   False
      DiskPressure     False
      PIDPressure      False
      Ready            True

If a resource pressure appears, you might see:

    MemoryPressure   True

Or:

    DiskPressure     True

### What This Means

#### `MemoryPressure=True`
The node has significant memory pressure.

#### `DiskPressure=True`
The node has significant disk pressure.

#### `PIDPressure=True`
The node has tight PID resources.

### Operations Understanding Focus

When troubleshooting nodes:

> **`kubectl describe node` is one of the first entry points to understand Eviction scenarios.**

---

## X. How to Check if a Pod Was Evicted

The two most commonly used methods are:

### 1. Check the Pod List

    kubectl get pod -n <namespace>

If a Pod has been evicted, you might see:

- The original Pod is no longer visible
- A new Pod is restarted by the controller
- Some Pods have abnormal statuses

---

### 2. Check Pod Details

    kubectl describe pod <pod-name> -n <namespace>

Focus on:

- Status
- Events
- Any eviction-related explanations

In some scenarios, you might see:

- `Evicted`
- Resource pressure-related warnings

### Operations Understanding Focus

If you find:

- It's not an application error
- It's not a probe failure
- It's not an image issue
- The Pod was actively removed by the system

You should start thinking about:

> **Node resource pressure / Eviction**

This direction.

---

## XI. A Real-World Troubleshooting Perspective: What a Evicted Pod Looks Like

Assume a node's memory pressure is increasing.

You first execute:

    kubectl get pod -n test -o wide

You might see a Pod that was previously on this node:

- `worker-abc123` on `node-1`

Later, when you check again:

- The Pod disappears
- The controller restarts a new `worker-xyz456`

At this point, when you describe the original Pod, you might see traces related to Eviction.

Then check:

    kubectl describe node node-1

You're likely to see:

- `MemoryPressure=True`
- Or resource pressure-related events

### Operations Understanding Focus

This scenario is about building a troubleshooting chain, not just command memory:

### 1. Discover abnormal Pod disappearance / restart
### 2. Rule out image, probe, or application self-failure
### 3. Check Pod describe
### 4. Check Node describe
### 5. Combine with pressure conditions to determine if it's Eviction

---

## 12. What's the relationship between Eviction and requests / limits

This section is very important.

### 1. requests affects resource guarantee baseline
If a Pod has declared requests, it at least indicates:

- Resource demand is declared
- Kubernetes can better understand its baseline needs

This will further affect QoS classification, and also influence priority tendencies during resource scarcity.

---

### 2. limits affects runtime upper bounds
limits determine:

- How much resource a container can use at most

If limits are too high, too loose, or if resource governance is chaotic, it may indirectly exacerbate node resource pressure.

---

### 3. requests / limits together affect QoS
And QoS further affects:

- Resource guarantee hierarchy during node pressure scenarios

### Operations Understanding Focus

Therefore, requests / limits are not only affecting:

- Scheduling
- OOM

They will also continue to affect:

- QoS
- Eviction risk under node pressure

---

## 13. Is Eviction and Pending a single phase issue

No.

### `Pending`
More biased toward:

> **The Pod hasn't been placed yet**

That is, issues during scheduling or scheduling process.

---

### `Eviction`
More biased toward:

> **The Pod has already started running, but was later evicted due to node resource pressure**

This is an issue during runtime phase.

### You can remember it this way

- `Pending`: Can't be placed
- `Eviction`: Placed but later lost

This distinction is very important.

---

## 14. What's the relationship between Eviction and HPA

This is a very practical issue.

### 1. HPA can scale Pods
It addresses:

> **Whether to increase replica count**

---

### 2. Eviction may evict Pods
It handles:

> **Whether the node can stay stable**

### So in real environments, this phenomenon may occur

- HPA sees high load and starts scaling Pods
- The node was already resource-constrained
- New Pods haven't stabilized yet
- Old Pods are evicted due to node pressure

### Operations Understanding Focus

Therefore, HPA is not a universal solution.  
If the underlying node resource pressure is already high, relying solely on "adding more replicas" may not solve the problem, and could even make the situation worse.

---

## 15. A more practical observation habit

After you suspect "the Pod isn't crashing on its own", try to form this observation sequence:

### Step 1: Check the Pod

    kubectl get pod -n <namespace> -o wide

Focus on:

- Whether the Pod is being restarted
- Whether the Pod has been moved to another node
- Whether the original Pod disappeared and a new Pod appeared

---

### Step 2: Check Pod details

    kubectl describe pod <pod-name> -n <namespace>

Focus on:

- Events
- Status changes
- Any Evicted-related clues

---

### Step 3: Check the node

    kubectl describe node <node-name>

Focus on:

- `MemoryPressure`
- `DiskPressure`
- `PIDPressure`
- Node events

---

### Step 4: Review resource configuration

Focus on verifying:

- requests
- limits
- QoS level

### Operations Understanding Focus

Don't treat Eviction as just a Pod-specific issue.  
It's often the result of:

> **The node's overall resource governance problem.**

---

## 16. Why resource governance is important in production environments

The act of Eviction itself is reminding you:

> **Kubernetes is not an infinite container platform; node capacity and resource governance must be taken seriously.**

In production environments, at least the following should be prioritized:

### 1. Avoid long-term heavy use of BestEffort Pods
Otherwise, these Pods will be very fragile when node pressure rises.

### 2. For critical workloads, clearly define requests / limits
Make resource boundaries clearer.

### 3. Pay attention to log, image, and temporary file usage
Avoid `DiskPressure`.

### 4. Don't only look at Pod count, but also node capacity
Often, it's not "can we start another Pod" but "can the node stay stable."

### 5. Monitor node pressure status
Especially:

- Memory
- Disk
- PID
- Eviction events

---

## 17. Common misunderstanding in this article

### 1. Think Eviction means the application crashed on its own
In fact, it's often the node actively removing the Pod.

### 2. Think OOMKilled and Eviction are the same
They are related but not the same thing.

### 3. Think a Pod can run stably if it can be scheduled
Scheduling success is just the first step; runtime will still face node resource pressure issues.

### 4. Think Guaranteed Pods will never be evicted
They won't be evicted absolutely, but usually have stronger guarantees and higher priority.

### 5. Think Eviction is only related to memory
Actually, it also includes:
- `DiskPressure`
- `PIDPressure`

---

## 18. Key understandings from this article

### 1. Eviction is an active eviction mechanism under node resource pressure
It's not a random failure.

### 2. Eviction and OOMKilled are not the same thing
One is more node-level resource pressure, the other is more container-level memory issues.

### 3. Common node pressure types include
- `MemoryPressure`
- `DiskPressure`
- `PIDPressure`

### 4. QoS affects priority tendencies under resource pressure scenarios
Typically:
- `BestEffort` is more fragile
- `Burstable` is in the middle
- `Guaranteed` is more stable

### 5. requests / limits don't only affect scheduling, but also continue to affect runtime stability
This is a very critical point in resource governance.

### 6. Eviction isn't just about the Pod itself, but the node's overall state
This is a particularly important perspective for troubleshooting.

---

## 19. What level should you master to understand this article

At this stage, it's recommended to reach the following level:

### 1. Be able to explain what Eviction is
### 2. Be able to distinguish Eviction and OOMKilled
### 3. Be able to name three common node pressure types
### 4. Be able to check pressure status via `kubectl describe node`
### 5. Be able to understand the basic relationship between QoS and eviction priority
### 6. Be able to establish a basic troubleshooting sequence for eviction-related issues

---

## 20. Common follow-up questions in interviews

Common questions in this area include: /think

- What is Eviction in Kubernetes
- Why would a node evict a Pod
- What's the difference between Eviction and OOMKilled
- What are common resource pressures on nodes
- `MemoryPressure` / `DiskPressure` / `PIDPressure` represent what
- What's the relationship between QoS and eviction priority
- Why are BestEffort Pods more prone to issues
- How to troubleshoot if a Pod was evicted

---

## 21. Summary

Eviction is a critical component in Kubernetes resource management.

The most important takeaway from this article isn't memorizing various thresholds, but first establishing these core understandings:

- When node resource pressure is too high, Kubernetes will actively evict some Pods
- The primary goal of Eviction is to protect the node itself
- OOMKilled is related to Eviction but not the same thing
- QoS affects a Pod's "survivability" under resource pressure
- requests/limits aren't just scheduling fields - they also continue to impact runtime stability
- When troubleshooting Eviction issues, you must examine Pod, node, and resource configuration from three levels

As long as these understandings are established, future learning will be more coherent:

- HPA
- Node capacity planning
- Resource governance
- Business high availability

Your thinking framework will become more complete.

---

## 22. Key Terms Memorization

- Eviction: Active Pod eviction under node resource pressure
- MemoryPressure: Node memory pressure
- DiskPressure: Node disk pressure
- PIDPressure: Node PID pressure
- QoS: Resource guarantee level
- Guaranteed: High guarantee
- Burstable: Intermediate guarantee
- BestEffort: Low guarantee
- OOMKilled: Container-level memory-related abnormal exit
- describe node: Important entry point for checking node pressure

---

## 23. Operational Insights

From an operational perspective, Eviction is Kubernetes reminding you of a very practical fact:

> **Platform stability always takes precedence over individual Pod survival.**

If a node is already nearing its limits, Kubernetes's first priority isn't "keep every Pod running", but rather:

- Protect the node
- Protect kubelet
- Protect the basic runtime environment
- Keep the entire system from losing control as much as possible

Thus, Eviction isn't an "extra cold knowledge" point - it's a key node in understanding Kubernetes' resource governance mindset.

It truly connects all the previously learned content:

- requests/limits
- QoS
- Node capacity
- Business stability
- Resource pressure
- Runtime governance

Precisely for this reason, the Eviction article isn't a standalone knowledge point - it's the key step in transitioning from "knowing how to deploy Pods" to "understanding the relationship between nodes and business."

---

## References
- Kubernetes Node-pressure Eviction
- Kubernetes Pod Quality of Service Classes
- Common outputs troubleshooting for kubectl describe node
- Kubernetes Resource Management for Pods and Containers

---

## Next Day Suggestions
Next article suggestion: organize

[[06-HPA Basics - Pod Auto Scaling Getting Started]]