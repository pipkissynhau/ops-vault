# 02-How Resource Requests Affect Pod Scheduling: From Pending to Scheduling Decisions

## Document Notes
- Document Location: Resource-Driven Scheduling Basics
- Applicable Stage: After understanding requests / limits basics, entering resource requests' impact on Pod scheduling and Pending troubleshooting
- Recommended Path: `04-Kubernetes/07-Apply deployment/08-Resource management and flexibility/02-How the resource requests affect Pod Movement control: from Pending To dispatch decision-making`

## Tags
#Kubernetes #Scheduler #Pending #requests #limits #ResourceMovement #PodMovement #ResourceManagement #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## I. Why Understand "How Resource Requests Affect Scheduling" Separately

Many beginners in Kubernetes often confuse these concepts:

- How much resource a container actually uses
- How much is written in YAML for `requests`
- How much is written in YAML for `limits`
- How much resource a node currently has left
- Why a Pod is in `Pending`

In Kubernetes, **the scheduler's first priority is not how much a container will actually use, but the resource requests declared by the Pod.**

This is why we often see these phenomena in real environments:

- A node appears to have idle CPU, but the Pod still can't be scheduled
- A Pod is stuck in `Pending`
- HPA scales out new replicas, but new Pods can't start
- The application hasn't started yet, but it failed during scheduling

This article's core goal is to establish a key understanding:

> **Whether a Pod can be scheduled to a node first depends on the resource requests (requests), not limits, nor the current instantaneous resource usage.**

---

## II. What is "Resource-Driven Scheduling"

The "resource-driven scheduling" discussed here is NOT:

- `nodeSelector`
- `nodeAffinity`
- `taints / tolerations`
- `podAffinity / podAntiAffinity`

These are "strategy-based scheduling" approaches.

Here we're discussing a different main line:

> **How the scheduler places a Pod on a node with enough available resources based on the resource requests declared by the Pod.**

In other words, it mainly answers:

- How much CPU does this Pod request?
- How much memory does this Pod request?
- Can this node accommodate this Pod?
- Why is the Pod in Pending if it can't?

This is the most direct connection between resource management and scheduling decisions.

---

## III. Why the Scheduler Focuses on requests, Not limits

This is the most important sentence in this article.

### 1. `requests` is the basis for the scheduler's resource placement decisions

Before scheduling, the scheduler first checks the Pod's resource requirements, i.e.:

- `resources.requests.cpu`
- `resources.requests.memory`

Then it checks how much **available resources** the node has left.

If the node's remaining available resources can't meet this Pod's requests, that node won't be a candidate.

---

### 2. `limits` is mainly not for the scheduler

`limits`'s main purpose is more about runtime resource constraints, such as:

- How much CPU can be used at most
- How much memory can be occupied at most
- The Pod might be OOMKilled when exceeding memory limits

So it's more about:

> **What's the maximum usage level a container can reach after it's running.**

Rather than:

> **Whether this Pod can be placed before scheduling.**

---

### 3. Why Kubernetes is designed this way

Because the scheduler needs a "predictable, declarable, calculable" resource requirement value.

Actual usage has several issues:

- The application hasn't started yet, so the scheduler can't know future actual resource usage
- Actual usage is dynamically changing, unsuitable for static scheduling decisions
- Actual resource usage fluctuates greatly over time, can't be used directly as placement basis

So Kubernetes uses:

> **Use requests to describe "how much resource needs to be reserved at least", allowing the scheduler to make placement decisions based on this value.**

---

## IV. What Happens from Pod Creation to Successful Scheduling

From an operations perspective, a Pod's journey from submission to successful node placement roughly goes through this process:

### 1. User Submits Pod / Deployment / StatefulSet

For example, executing:

    kubectl apply -f deployment.yaml

---

### 2. Controller Creates Pod Object

For example, Deployment first creates ReplicaSet, then ReplicaSet creates Pod.

---

### 3. Scheduler Detects This Pod Has No Bound Node

Such Pods usually are in:

- `Pending`
- `PodScheduled=False`

---

### 4. Scheduler Begins Node Filtering

It considers many dimensions, but resource is very core:

- Does the node have enough CPU request space?
- Does the node have enough memory request space?
- Does it meet other constraints?

---

### 5. After Finding a Suitable Node, Completes Binding

The Pod is assigned to a specific node.

---

### 6. kubelet Starts Pulling Images, Creating Containers, and Launching Pod on the Node

At this point, the Pod enters the subsequent running phase.

### Operations Understanding Focus

If the resource check fails, the Pod hasn't even started yet and will be stuck in:

> **Pending phase**

This is why many "application startup failures" actually have their root cause not in the application itself, but:

> **The Pod hasn't been scheduled successfully yet.**

---

## V. Why a Pod is Pending: Common Resource-Related Reasons

A Pod in `Pending` doesn't necessarily mean it's a resource issue.  
But in this chapter on resource management, the most common resource-related reasons usually include:

### 1. Node CPU Resources Insufficient

For example, the Pod requests:

- `cpu: 2`

But no node in the cluster has enough remaining available CPU request space.

---

### 2. Node Memory Resources Insufficient

For example, the Pod requests:

- `memory: 4Gi`

But all available nodes can't meet the requirement.

---

### 3. requests Set Too High

This is very common in test environments.

For example, a normal Nginx web service, but the YAML is written as:

- `cpu: 4`
- `memory: 8Gi`

Then the scheduler might not find any suitable node.

---

### 4. Resource Insufficient After Multiple Replicas

A single Pod's request may seem small, but if:

- `replicas: 10`
- Each Pod has fixed requests

The cumulative total may still cause subsequent Pod scheduling failures.

---

### 5. HPA Scaling Out, New Replicas Can't Find Placement

Existing replicas can run, but new Pods scaled out by HPA are in Pending due to resource insufficiency.

---

## VI. How CPU Requests Affect Scheduling

CPU requests typically indicate:

> **The amount of CPU resources the container expects to be reserved for scheduling.**

For example: /think

resources:
  requests:
    cpu: "500m"

This indicates the container has requested:

- `0.5` CPU cores

If a Pod contains multiple containers, the Pod-level CPU requests are typically understood as:

> **The sum of all container CPU requests**

The scheduler matches this total with the remaining available CPU on the node.

### Example

A node currently has available:

- `700m` CPU

While a new Pod requests:

- `800m` CPU

This Pod will not be scheduled on this node.

### Operations Understanding Focus

CPU requests affect:

- **Whether this Pod can be placed first**

It does not equal the application will definitely use this much CPU during runtime.

---

## VII. How Memory Requests Affect Scheduling

Memory requests typically indicate:

> **The amount of memory resources this container expects to be reserved for scheduling.**

For example:

    resources:
      requests:
        memory: "512Mi"

Indicates the container has requested 512Mi of memory.

The scheduler also matches Pod total memory requests against node available memory.

### A Key Difference with CPU

Memory is generally "harder" during runtime because:

- CPU can be compressed or preempted
- Memory exceeding by too much can easily trigger OOM or scheduling failure

So memory requests are often more sensitive in scheduling practices.

### Operations Understanding Focus

Many Pods being Pending is not necessarily due to CPU insufficiency, but:

> **Memory requests are too large, and the node simply can't accommodate them.**

---

## VIII. Why Limits Do Not Directly Participate in Scheduling Decisions

This is one of the most commonly misunderstood points when starting out.

### Let's Summarize First

In the most basic understanding of resource scheduling, you can initially understand:

> **The scheduler primarily looks at requests, not directly using limits for placement decisions.**

### Why People Often Confuse

Because people see:

- `requests`
- `limits`

Together written, it's easy to mistakenly assume:

- Both will equally affect scheduling

Actually, it's not.

### Limits Are More About Runtime Constraints

For example:

    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"

This can be initially understood as:

- Schedule by `500m / 512Mi` first
- At runtime, this container is allowed to use up to `1 CPU / 1Gi Memory`

### Operations Understanding Focus

In such cases, the first thing to ask is often:

> **Is the Pod unschedulable because requests are too large, not because limits are too large.**

---

## IX. Why a Node Seems to Have Resources, Yet a Pod Still Fails to Schedule

This is a highly frequent practical confusion.

Many people look at:

- `top`
- `htop`
- Grafana
- Node CPU current usage rate
- Node memory current usage rate

Then conclude:

> "This node clearly has plenty of resources, why can't the Pod be scheduled?"

The key reason here is:

> **The scheduler doesn't look at the instantaneous usage rate you see, but at Kubernetes' view of allocatable resources and already allocated resources.**

### Need to Distinguish Several Concepts

#### 1. Node Total Resources
How much CPU and memory the machine has in total.

#### 2. Allocatable
Resources truly available for Pods, not all machine resources are used for Pods.

#### 3. Already Allocated Requests
Resources already "reserved" by existing Pods.

#### 4. Current Instantaneous Usage Rate
Just the actual CPU/memory usage at a specific moment.

### Operations Understanding Focus

Even if a node's current CPU usage rate is low, it might still fail to schedule a new Pod because:

- Many Pods have already declared requests
- The scheduler thinks resources are already "reserved"
- So the new Pod still can't be scheduled

This is a very typical scenario in resource scheduling:

> **The machine appears idle, but the scheduler doesn't think it can accommodate another Pod.**

---

## X. A Basic Requests Example

Here's a basic Deployment example:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-web
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx-web
      template:
        metadata:
          labels:
            app: nginx-web
        spec:
          containers:
            - name: nginx-web
              image: nginx:1.27
              ports:
                - containerPort: 80
              resources:
                requests:
                  cpu: "100m"
                  memory: "128Mi"
                limits:
                  cpu: "500m"
                  memory: "256Mi"

### Understanding This YAML

#### `requests`
Indicates the scheduler reserves at least:

- 100m CPU
- 128Mi memory

for this container

#### `limits`
Indicates the maximum allowed usage during runtime:

- 500m CPU
- 256Mi memory

### Operations Understanding Focus

If the node can't even provide:

- 100m CPU
- 128Mi memory

the Pod may fail during scheduling.

---

## XI. An Example of a Pod Being Pending Due to Resource Insufficiency

Assume a test node has relatively limited resources, and you write a Pod like this:

apiVersion: v1
kind: Pod
metadata:
  name: big-memory-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "2"
          memory: "4Gi"
        limits:
          cpu: "2"
          memory: "4Gi"

If none of the nodes in the current cluster can satisfy:

- 2 CPU cores
- 4Gi memory

This Pod is very likely to remain in:

- `Pending`

### Common Phenomenon

Running:

    kubectl get pod

May show:

    NAME             READY   STATUS    RESTARTS   AGE
    big-memory-pod   0/1     Pending   0          2m

At this point, many people mistakenly think:

- Image pull failure
- Container startup failure
- Application issue

But in reality, it hasn't even reached that stage.

---

## Twelve. How to Determine if It's a Resource Scheduling Issue via describe / events

This is the most critical troubleshooting entry point.

When a Pod is Pending, first check:

    kubectl describe pod <pod-name>

Focus on:

### 1. `Events`
If it's a resource scheduling issue, you'll commonly see similar information:

- `Insufficient cpu`
- `Insufficient memory`
- `0/3 nodes are available`
- `node(s) didn't have enough memory`
- `node(s) didn't have enough cpu`

### 2. `Conditions`
You'll commonly see:

- `PodScheduled=False`

This indicates the Pod hasn't been successfully scheduled to any node.

### Operations Understanding Focus

As soon as you see such events, you should first determine:

> **Are the requests set too high, or are the node resources inherently insufficient?**

Rather than immediately suspecting the image, application, or configuration files.

---

## Thirteen. The Minimal Closed Loop Between Resource Requests, Pending, and Scheduling Decisions

You can summarize this entire article with one sentence:

> **After a Pod is created, the scheduler will determine which node can accommodate it based on the requests; if no node can accommodate it, the Pod will remain Pending.**

In other words, this chain of events can be simplified to:

### 1. Submit Pod
### 2. Scheduler reads requests
### 3. Compare with each node's available resources
### 4. Find a node and schedule successfully
### 5. Fail to find a node and remain Pending

This is the most fundamental scheduling logic driven by resource requirements.

---

## Fourteen. How This Issue Relates to HPA and Cluster Autoscaler

Establish a basic understanding first, without going too deep.

### 1. Relationship with HPA
HPA is responsible for:

- Increasing or decreasing the number of Pod replicas

But HPA does not guarantee:

- New replicas will have a place to land

In practice, it's common to see:

- HPA has already scaled out
- New Pod has been created
- But remains Pending due to resource insufficiency

---

### 2. Relationship with Cluster Autoscaler
Cluster Autoscaler is responsible for:

- Attempting to add nodes when there are not enough

So it's a more upper-level remediation mechanism.

### Operations Understanding Focus

HPA addresses:

- "Should we have more Pods?"

Resource-driven scheduling addresses:

- "Do these Pods have a place to land?"

Cluster Autoscaler addresses:

- "Should we add more nodes if there's no place to land?"

---

## Fifteen. Common Misunderstandings in This Topic

### 1. Believing the scheduler looks at real-time resource usage
In reality, it focuses more on:

- requests
- allocatable
- already allocated resources

---

### 2. Believing limits determine if a Pod can be scheduled
In the early stages, remember firmly:

> **The most critical factor is requests.**

---

### 3. Immediately suspecting application issues when seeing a Pending Pod
In many cases, the application hasn't even started running yet.

---

### 4. Only looking at Pod status, not describe and events
This easily leads to misdiagnosis.

---

### 5. Arbitrarily writing requests
Writing too high causes scheduling failure.  
Writing too low may lead to resource contention, performance fluctuations, OOM issues later.

---

## Sixteen. What to Check First When Troubleshooting This Issue

Recommend forming a simple troubleshooting sequence.

### 1. First check Pod status

    kubectl get pod -n <namespace>

If it's:

- `Pending`

Start from the scheduling perspective first.

---

### 2. Then check Pod details

    kubectl describe pod <pod-name> -n <namespace>

Focus on:

- `Events`
- `PodScheduled`
- Whether there is `Insufficient cpu / memory`

---

### 3. Then check requests in the YAML

Focus on verifying:

- `resources.requests.cpu`
- `resources.requests.memory`

---

### 4. Then check node resource overview

For example:

    kubectl get nodes
    kubectl describe node <node-name>

Focus on:

- Allocatable
- Already allocated requests
- Node pressure status

---

### 5. Finally, consider whether to reduce requests or expand nodes

Don't blindly increase replica count, rebuild Deployment, or suspect the image at first.

---

## Seventeen. The Most Important Understandings in This Topic

### 1. The most critical factor for whether a Pod can be scheduled successfully is requests
This is the core understanding of this article.

### 2. requests affects whether the Pod can be placed
It doesn't equal the application will definitely use that much during runtime.

### 3. limits are more about "runtime upper limits"
They are not the most critical judgment criteria during scheduling.

### 4. Pod Pending often originates from resource scheduling failure
Not application startup failure.

### 5. Nodes appearing to have idle resources doesn't mean the scheduler thinks there's space for new Pods
Distinguish between "current usage rate" and "already allocated resources."

### 6. describe and events are the first entry point for troubleshooting Pending
Don't only focus on `kubectl get pod`.

---

## Eighteen. What Level of Understanding Should You Reach to Learn This Topic

At this stage, it's recommended to reach the following level:

### 1. Can explain why requests affect scheduling  
### 2. Can understand the relationship between Pod Pending and resource insufficiency  
### 3. Can distinguish the differences in scheduling phase roles between requests and limits  
### 4. Can interpret a basic resource configuration YAML  
### 5. Can preliminarily determine if it's a resource-related scheduling failure via describe / events  
### 6. Can understand why HPA-expanded Pods might still fail to start  

---

## Nineteen, Common Interview Extensions  

This section includes common questions:  

- Does Kubernetes scheduler primarily consider requests or limits?  
- Why would a Pod be Pending?  
- Why can't a Pod be scheduled even though the node still has resources?  
- What's the difference between requests and limits?  
- What do you check first when a Pod scheduling fails?  
- `Insufficient cpu` / `Insufficient memory` generally indicate what?  
- Why might HPA-expanded Pods still be Pending?  
- How to understand the difference between allocatable and total node resources?  

---

## Twenty, Stage Summary  

How resource requests affect Pod scheduling is a critical part of Kubernetes resource management.  

The most important takeaway from this article isn't memorizing scheduler implementation details, but establishing these core understandings:  

- When the scheduler makes resource placement decisions, it primarily looks at requests first  
- Requests determine whether a Pod is eligible to be placed on a node  
- Limits focus more on runtime resource upper bounds  
- Pod Pending often isn't due to program failure, but scheduling phase failure  
- The primary entry point for diagnosing resource-related scheduling issues is describe and events  
- When checking node resources, don't only look at instantaneous usage rates, but also allocatable and already requested resources  

Establishing these understandings will make subsequent learning much smoother:  

- QoS  
- Eviction  
- HPA  
- Resource troubleshooting  

---

## Twenty-one, Keyword Quick Notes  

- requests: Resource request, core basis for scheduling  
- limits: Resource upper bound, runtime resource constraints  
- Pending: Pod hasn't completed scheduling or startup  
- scheduler: Scheduler, responsible for selecting nodes for Pods  
- allocatable: Resources truly available for Pod use on a node  
- Insufficient cpu: CPU resource insufficiency, unable to meet scheduling  
- Insufficient memory: Memory resource insufficiency, unable to meet scheduling  
- HPA: Horizontal auto-scaling  
- Cluster Autoscaler: Node auto-scaling  

---

## Twenty-two, Operational Extensions  

From an operational perspective, resource-driven scheduling is the starting point for Kubernetes evolving from "being able to create Pods" to "being able to reasonably place workloads."  

Many beginners focus on:  

- Can the image be pulled down?  
- Can the container start?  
- Can Service be accessed?  

But in real environments, many issues occur much earlier:  

- The Pod hasn't even started running  
- The scheduler has already determined resource insufficiency  
- The workload is stuck in Pending from the moment of creation  

This is why resource requests, though just a small field in YAML, actually connects:  

- Pod scheduling  
- Node capacity  
- Resource planning  
- HPA scaling effectiveness  
- Cluster Autoscaler trigger conditions  
- Subsequent resource troubleshooting  

So this article appears to focus on requests, but actually establishes your first-level understanding of the relationship between Kubernetes "resources, scheduling, and capacity."  

---

## References  
- Kubernetes Resource Management for Pods and Containers  
- Assign Memory Resources to Containers and Pods  
- Assign CPU Resources to Containers and Pods  
- Pod Scheduling Readiness  
- kubectl describe pod / describe node common output troubleshooting  

---

## Next Day Suggestions  
Next article suggestion:  

[[03-Why Pod is Pending - Troubleshooting Resource Insufficiency and Scheduling Failure]]