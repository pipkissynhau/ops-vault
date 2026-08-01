# 04-QoS Basics: Guaranteed, Burstable, and BestEffort

## Document Notes
- Document Positioning: Kubernetes Resource Quality of Service Level Introduction Practice
- Applicable Stage: After understanding requests / limits, resource-driven scheduling, and Pod Pending, entering QoS, resource contention, and node resource pressure basics
- Recommended Path: `04-Kubernetes/07-Apply deployment/08-Resource management and flexibility/04-QoS Basis:GuaranteedI don't know.Burstable and BestEffort`

## Tags
#Kubernetes #QoS #Guaranteed #Burstable #BestEffort #requests #limits #Eviction #ResourceManagement #PodMovement #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## One, Why Learn QoS Now

The previous set of mainlines have established several key cognitions:

- `requests` affects Pod scheduling
- Pod may be `Pending` due to resource insufficiency
- `limits` is more biased toward runtime resource constraints
- When node resources are tight, Pods may not be treated equally

But here, you will still encounter a very practical problem:

- Why are some Pods more "stable" when running on the same node
- Why some Pods are more prone to issues when resources are tight
- Why formal business generally does not recommend completely omitting resource configuration
- Why requests / limits are not only about scheduling but also about runtime resource guarantees

The core of these questions will fall to:

> **QoS (Quality of Service, Service Level)**

This is not about network QoS, nor about QoS in traffic shaping.  
This refers to:

> **Kubernetes automatically assigns resource guarantee levels to Pods based on how containers in Pods declare resources.**

The significance of learning QoS lies in:

- Understanding why different Pods have different "treatment" when resources are tight
- Understanding why some Pods are more likely to be evicted
- Understanding how proper resource configuration affects business stability
- Laying the foundation for further learning about Eviction, OOM, HPA, and resource governance

---

## Two, What Exactly Is QoS

QoS can be simply understood as:

> **Kubernetes automatically calculates resource guarantee levels for Pods based on the requests / limits configuration of containers in Pods.**

It is mainly strongly related to these fields:

- `resources.requests.cpu`
- `resources.requests.memory`
- `resources.limits.cpu`
- `resources.limits.memory`

Kubernetes will classify Pods into one of three QoS levels based on these configurations:

- `Guaranteed`
- `Burstable`
- `BestEffort`

### Operations Understanding Focus

QoS is not a field you manually write in YAML.  
You won't directly write:

    qosClass: Guaranteed

Instead:

> **Kubernetes calculates it automatically based on resource configuration.**

---

## Three, Why Is QoS Important

QoS is important not because it's just a "label," but because it affects the priority tendency during resource contention in runtime.

For example:

### 1. Which Pods are more likely to be prioritized when node resources are tight
This is strongly related to QoS.

### 2. Why some temporary test Pods fail immediately under resource pressure
This is often related to their low QoS level.

### 3. Why critical business recommends more standardized requests / limits configuration
Because this affects not only scheduling but also resource guarantee intensity during resource contention.

### 4. Why "not writing resource configuration to get started first" is highly risky in production environments
Because this usually falls into the lowest guarantee level.

### Operations Understanding Focus

You can initially understand QoS as:

> **Kubernetes's "layered classification" for Pods in resource governance.**

---

## Four, What Are the Three QoS Categories

The three most common QoS levels in Kubernetes are:

### 1. `Guaranteed`
The category with the highest resource guarantee.

### 2. `Burstable`
Intermediate, also the most common category.

### 3. `BestEffort`
The category with the lowest resource guarantee.

### You can initially roughly remember as

- `Guaranteed`: Most clear resource boundaries, strongest guarantee
- `Burstable`: Has resource declarations but hasn't met the highest guarantee conditions
- `BestEffort`: Basically no resource declarations, weakest guarantee

---

## Five, What Is Guaranteed

### First, the conclusion

For a Pod to become `Guaranteed`, you can initially understand it as:

> **Each container in the Pod has simultaneously set CPU and memory requests and limits, and the corresponding values are completely equal.**

For example:

    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"

### Why This Type of Pod Has Higher Guarantee

Because its resource boundaries are the most clear:

- At least how much
- Maximum how much
- Requests and limits are completely consistent

Kubernetes can more easily view this as a workload with very clear resource boundaries.

### Operations Understanding Focus

The key point of `Guaranteed` is not "requesting a lot of resources," but:

> **The resource declarations are the most complete, strict, and aligned.**

---

## Six, What Is Burstable

### First, the conclusion

As long as a Pod is not `Guaranteed`, but also not completely missing resource configuration, it will usually fall into:

> **Burstable**

This is the most common category in actual environments.

### Common Scenarios

#### Scenario 1: Wrote requests and limits, but they are not equal

    resources:
      requests:
        cpu: "200m"
        memory: "256Mi"
      limits:
        cpu: "1"
        memory: "512Mi"

#### Scenario 2: Wrote requests but not limits

    resources:
      requests:
        cpu: "200m"
        memory: "256Mi"

#### Scenario 3: Wrote partial resource fields but not complete

### How to Understand This

This type of Pod can initially be understood as:

> **Has some resource guarantees but does not belong to the highest guarantee level.**

### Operations Understanding Focus

Burstable does not equal "poor configuration."  
Quite the opposite, many real businesses, to balance:

- Resource utilization
- Minimum guarantees
- Peak elasticity

Will fall into Burstable.

---

## Seven, What Is BestEffort

### First, the conclusion /think

If all containers in a Pod **have no requests or limits set**, the Pod is typically classified as:

> **BestEffort**

Example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-pod
    spec:
      containers:
        - name: nginx
          image: nginx:1.27

Here, **nothing is written**:

- requests
- limits

So it's the most typical example of `BestEffort`.

### How to Understand This

You can think of it as:

> **This Pod hasn't explicitly told Kubernetes how much resource it needs at minimum or how much it's allowed to use at maximum.**

Thus, its resource guarantees are at the lowest level.

---

## VIII. The Simplest Way to Judge the Three QoS Classes

Currently, you can use a very practical judgment mnemonic:

### 1. `Guaranteed`
**Each container has both CPU and memory requests and limits, and the corresponding values are equal**

### 2. `Burstable`
**Resource configurations are written, but they don't meet the strict conditions for Guaranteed**

### 3. `BestEffort`
**No requests/limits are written at all**

---

## IX. A Guaranteed Example

    apiVersion: v1
    kind: Pod
    metadata:
      name: guaranteed-pod
    spec:
      containers:
        - name: app
          image: nginx:1.27
          resources:
            requests:
              cpu: "500m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

### Why It's Guaranteed

Because:

- It has written `cpu requests`
- It has written `cpu limits`
- It has written `memory requests`
- It has written `memory limits`
- CPU request = CPU limit
- memory request = memory limit

Thus, it meets the typical conditions of `Guaranteed`.

---

## X. A Burstable Example

    apiVersion: v1
    kind: Pod
    metadata:
      name: burstable-pod
    spec:
      containers:
        - name: app
          image: nginx:1.27
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"

### Why It's Burstable

Because although resource configurations are written, it:

- CPU request ≠ CPU limit
- memory request ≠ memory limit

Thus, it doesn't meet the strict conditions for Guaranteed.  
But it's not completely missing resource fields, so it's not BestEffort.

Therefore, it belongs to:

> **Burstable**

---

## XI. A BestEffort Example

    apiVersion: v1
    kind: Pod
    metadata:
      name: besteffort-pod
    spec:
      containers:
        - name: app
          image: nginx:1.27

### Why It's BestEffort

Because here, **nothing is written**:

- requests
- limits

So Kubernetes will classify it as:

> **BestEffort**

---

## XII. How to Check a Pod's QoS Class

The two most commonly used methods are:

### 1. Check Pod Details

    kubectl describe pod <pod-name> -n <namespace>

The output usually shows:

    QoS Class: Burstable

Or:

    QoS Class: Guaranteed

---

### 2. Check YAML

    kubectl get pod <pod-name> -n <namespace> -o yaml

You'll usually see a field like:

    qosClass: Burstable

### Operations Focus

When troubleshooting, don't guess "which QoS class this Pod belongs to"—just check directly for the most reliable information.

---

## XIII. A Minimal Experiment: Creating Three QoS Classes Simultaneously

Below is a minimal experiment that places all three QoS classes in a single file.

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

### Applying the Experiment

    kubectl apply -f qos-demo.yaml

---

## Fourteen. How to Verify the Results

### 1. Check if the Pods Were Created Successfully

    kubectl get pod

You should see:

- `qos-guaranteed`
- `qos-burstable`
- `qos-besteffort`

---

### 2. Check the QoS Class for Each

    kubectl describe pod qos-guaranteed
    kubectl describe pod qos-burstable
    kubectl describe pod qos-besteffort

You typically see:

#### Guaranteed Pod
    QoS Class: Guaranteed

#### Burstable Pod
    QoS Class: Burstable

#### BestEffort Pod
    QoS Class: BestEffort

### Operational Understanding Focus

The value of this experiment isn't "getting the Pod running," but:

> **Transforming QoS from an abstract concept into an observable object property.**

---

## Fifteen. Why Many Businesses Are Not Guaranteed

Although Guaranteed looks the most stable, in actual environments many businesses do not configure them as Guaranteed from the start.

Common reasons include:

### 1. Business Resource Usage Fluctuates
If requests and limits are set identically, it may appear overly rigid.

### 2. Wanting to Separate Baseline and Peak
For example:

- requests give a baseline value
- limits allow it to spike during peak times

### 3. Need to Balance Resource Utilization
If all businesses pursue Guaranteed, resource utilization may not be optimal.

### Operational Understanding Focus

So many formal businesses ultimately fall into:

> **Burstable**

This isn't a configuration failure, but a common compromise.

---

## Sixteen. Why BestEffort Is Riskiest

The biggest issue with BestEffort isn't "being unable to run," but:

> **It has almost no clear resource guarantees.**

### Common Risks Include

#### 1. More Difficult to Predict in Scheduling and Runtime
Because there's no boundary on resource needs.

#### 2. More Vulnerable When Node Resources Are Tense
Because it belongs to the lowest resource guarantee level.

#### 3. Hard to Stabilize in Production Environments
Because resource reservations, capacity planning, and risk control are more difficult.

### Operational Understanding Focus

BestEffort is more suitable for:

- Temporary experiments
- Lightweight testing
- Very unimportant auxiliary workloads

For formal business, it's generally not recommended to completely omit resource configuration for long periods.

---

## Seventeen. Relationship Between QoS and Scheduling

This section is easy to confuse, so we need to clarify first.

### 1. Scheduling Phase Primarily Considers requests
That is, whether a Pod can be placed on a node first depends on:

- requests.cpu
- requests.memory

### 2. QoS Focuses More on Resource Guarantee Classification During Runtime
It mainly affects:

- PriorityOrientation during resource contention
- Eviction priorityOrientation under node resource pressure
- Runtime resource guarantee level

### Operational Understanding Focus

So we cannot confuse these two issues:

#### Scheduling Issue
Focuses on:

> **Can this Pod be scheduled first?**

#### QoS Issue
Focuses on:

> **What resource guarantee level does the Pod belong to when it's running and resources are tight?**

---

## Eighteen. Relationship Between QoS and Eviction (Drain)

This is one of the most important points in learning QoS.

Currently, we can first establish a basic understanding:

When node resource pressure occurs, for example:

- Memory pressure
- Disk pressure
- PID pressure

Kubernetes may trigger:

> **Eviction (Drain)**

Under node resource pressure scenarios, QoS typically influences the eviction priorityOrientation.

### Currently, we can first remember this direction

We can roughly remember:

- `BestEffort` is more likely to be processed first
- `Burstable` is next
- `Guaranteed` usually has stronger guarantees

### Operational Understanding Focus

This doesn't mean:

- Guaranteed will never be evicted
- BestEffort will definitely be the first to die

Rather, it means:

> **QoS is an important priority reference factor under resource pressure.**

---

## Nineteen. Relationship Between QoS and OOMKilled

These two cannot be directly equated, but they are related.

### First, clarify

`OOMKilled` is more directly associated with:

- Container runtime memory usage exceeding limits
- Or containers being killed due to memory issues

While QoS is associated with:

- Pod's resource guarantee hierarchy
- PriorityOrientation under node resource pressure

### You can understand it this way

- `limits.memory` being too small may lead to containers being OOMKilled more easily
- Lower QoS level may make Pods more likely to be evicted first under node resource pressure

### Operational Understanding Focus

Don't simply remember:

- Lower QoS = Definitely OOMKilled

A more accurate understanding is:

> **QoS, limits, node pressure, and eviction behavior collectively impact Pod stability.**

---

## Twenty, the Minimal Closed Loop of QoS and requests / limits

You can summarize this article with one sentence:

> **Kubernetes automatically classifies Pods into Guaranteed, Burstable, and BestEffort QoS tiers based on their requests / limits configuration; this tier further influences resource guarantees and priority during resource pressure.**

In other words, this chain can be simplified as:

### 1. Write resource configuration  
### 2. Kubernetes automatically calculates QoS  
### 3. Pod receives corresponding resource guarantee level  
### 4. When node resources are tight, QoS affects stability and priority preference

---

## Twenty-one, Common Misunderstandings in This Article

### 1. Think QoS is a manually configured field  
Actually, it's automatically calculated by Kubernetes.

### 2. Think writing requests alone makes a Pod Guaranteed  
Incorrect.  
Guaranteed has stricter conditions.

### 3. Think Burstable is always bad  
Not true.  
It's the most common and practical category.

### 4. Think BestEffort is just "simpler"  
Actually, it means the lowest resource guarantee.

### 5. Confuse QoS with scheduling  
Scheduling mainly depends on requests; QoS focuses more on resource guarantees during runtime.

---

## Twenty-two, Key Understandings in This Article

### 1. QoS is an automatically calculated resource guarantee level  
Not a manually written field in YAML.

### 2. QoS has three categories  
- `Guaranteed`  
- `Burstable`  
- `BestEffort`

### 3. Guaranteed has the strictest conditions  
Requires both CPU/memory requests and limits to be fully specified and equal.

### 4. Burstable is the most common category  
Many formal workloads fall here.

### 5. BestEffort has the lowest resource guarantee  
Usually means no requests / limits are specified.

### 6. QoS focuses more on runtime resource guarantees and eviction priority  
Cannot be completely mixed with scheduling.

### 7. Formal workloads should not long-term omit resource configuration  
Otherwise, they may fall into BestEffort, which is riskier.

---

## Twenty-three, What Level Should You Master for This Article

At this stage, it's recommended to reach the following level:

### 1. Be able to explain what QoS is  
### 2. Be able to distinguish Guaranteed, Burstable, BestEffort  
### 3. Be able to understand three basic YAML examples  
### 4. Be able to check QoS Class via `kubectl describe pod`  
### 5. Be able to understand the basic relationship between QoS, resource pressure, and eviction  
### 6. Be able to understand why BestEffort is riskier

---

## Twenty-four, Common Interview Follow-up Questions

Common questions in this area include:

- What are the three QoS classes in Kubernetes?  
- What are the differences between Guaranteed, Burstable, and BestEffort?  
- What types of Pods are classified as Guaranteed?  
- Why are many business Pods Burstable?  
- What risks does BestEffort carry?  
- What's the relationship between QoS and requests / limits?  
- What's the relationship between QoS and Eviction?  
- How to check a Pod's QoS level?

---

## Twenty-five, Summary of This Section

QoS is a critical layer in Kubernetes' resource management system.

The most important takeaway from this article isn't memorizing definitions, but establishing these core understandings:

- requests / limits affect not only scheduling but also runtime resource guarantee levels  
- Kubernetes automatically classifies Pods into Guaranteed, Burstable, BestEffort  
- Higher QoS typically means stronger resource guarantees  
- Lower QoS is more vulnerable under resource pressure  
- Formal workloads should not long-term rely on BestEffort, which has almost no explicit guarantees

Once these understandings are established, future learning will flow more smoothly:

- Eviction  
- Node resource pressure  
- OOMKilled  
- HPA and resource fluctuations

---

## Twenty-six, Keyword Mnemonics

- QoS: Kubernetes Resource Quality of Service Level  
- Guaranteed: Highest guarantee level, requests and limits are fully specified and equal  
- Burstable: Middle level, most common  
- BestEffort: Lowest guarantee level, usually no resource declarations  
- requests: Resource request, scheduling basis  
- limits: Resource upper limit, runtime constraint  
- Eviction: Eviction behavior under node resource pressure  
- QoS Class: Pod's current QoS level

---

## Twenty-seven, Operational Perspective

From an operational perspective, QoS marks Kubernetes' transition from "only checking if a Pod can run" to "beginning to differentiate workload guarantee levels."

Without the QoS perspective, you might only focus on:

- Whether a Pod is running  
- Whether requests are specified  
- Whether limits are specified  
- Whether OOM occurs

With the QoS perspective, you'll truly understand:

- Why different Pods behave differently under resource pressure  
- Why critical workloads need stricter resource boundaries  
- Why test workloads shouldn't be written like formal business  
- Why resource configuration affects not only scheduling but also runtime stability

Thus, although this article seems to focus on "classification," it actually connects:

- requests / limits  
- Scheduling  
- Runtime resource contention  
- Node pressure  
- Eviction priority  
- Business stability

It's a crucial step in understanding Kubernetes' resource governance system.

---

## References
- Kubernetes Pod Quality of Service Classes  
- Kubernetes Resource Management for Pods and Containers  
- Common troubleshooting of kubectl describe pod outputs

---

## Next Day Recommendation
Next article suggestion:  

[[05-Node Resource Pressure and Eviction Basics - Eviction Introduction]]