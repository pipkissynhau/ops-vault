# 03 - Why Pod is Pending: Resource Insufficiency and Scheduling Failure Troubleshooting

## Document Notes
- Document Location: Pod Pending issue beginner troubleshooting
- Applicable Stage: After understanding how requests affect scheduling, enter the basic troubleshooting of Pending phenomena, resource insufficiency, and scheduling failure
- Recommended Path: `04-Kubernetes/07-Apply deployment/08-Resource management and flexibility/03-Why? Pod Yes. Pending: Insufficient resources and failure of scheduling`

## Tags
#Kubernetes #Pending #Scheduler #InadequateResources #ScheduleFailed #requests #PodBarrier. #ResourceManagement #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## I. Why Learn About Pod Pending Separately

In Kubernetes, `Pending` is a highly frequent and critical status.

Many newcomers to application deployment, upon seeing a Pod not coming up, often immediately think:

- Image issues
- Startup command issues
- Application configuration issues
- Program startup failure

In reality, many Pods haven't even reached the "actual container startup" stage when they get stuck.

That is to say:

> **Pod being in Pending status doesn't necessarily mean application startup failure - it might have failed to complete scheduling or preparation phase altogether.**

The significance of learning about Pod Pending lies in:

- Learning to distinguish between "scheduling phase failure" and "runtime failure"
- Learning to first check scheduling, then image, then application
- Establishing the correct troubleshooting order for Kubernetes

---

## II. What Does Pod Pending Mean

From the most basic perspective, we can roughly understand the Pod lifecycle as:

### 1. Pod object has been created
YAML has been submitted to Kubernetes.

### 2. Pod hasn't yet entered stable operation
At this point, it may not have successfully bound to a node, or completed some necessary pre-start conditions.

### 3. So the status shows `Pending`
This indicates:

> **Pod has been received by Kubernetes, but hasn't completed scheduling or initialization to the runnable phase.**

### Operations Understanding Focus

You can first remember this way:

- `Pending`: Not yet truly running
- `Running`: At least has been scheduled and entered the running phase
- `CrashLoopBackOff`: Usually indicates runtime anomalies
- `ImagePullBackOff`: Usually indicates image pull issues

So `Pending` is more about pre-phase issues.

---

## III. The Two Main Directions of Pod Pending

Although this article mainly focuses on resource insufficiency, we need to first establish overall understanding:

Common causes of Pod Pending can typically be divided into two categories.

### 1. Scheduler can't find suitable nodes
This is the most typical category.

Examples include:

- CPU insufficient
- Memory insufficient
- Requests set too high
- Node selection conditions too strict
- Taint mismatch
- Scheduling constraints not met

The core of these issues is:

> **Pod hasn't been successfully placed on any node yet.**

---

### 2. Pod has been scheduled but some prerequisites are not met
Examples include:

- PVC not yet bound successfully
- Dependent resources not ready
- Some initialization processes not completed

These can also manifest as Pending.

### Operations Understanding Focus

So when seeing Pending, the first step isn't guessing, but to first determine:

> **Is it "unscheduled", or "scheduled but other prerequisites not met"?**

---

## IV. Why This Article Focuses on "Resource Insufficiency" First

Because in your current learning path, the first thing you need to master is the most common, most basic, and easiest to encounter type of Pending:

> **Pending due to resource insufficiency causing scheduling failure.**

That is, the events we most commonly see:

- `Insufficient cpu`
- `Insufficient memory`
- `0/3 nodes are available`

This type of issue is directly connected to the previous section's:

- requests
- limits
- Resource-driven scheduling

So this article first establishes this troubleshootingMain:

> **Pod Pending → describe Pod → check Events → determine if it's resource scheduling failure**

---

## V. Why Resource Insufficiency Causes Pending

This is the core causal relationship.

When the scheduler is choosing a node for a Pod, it checks the resource requests of the Pod, such as:

- `requests.cpu`
- `requests.memory`

Then checks if the node can still meet these requests.

If no node can meet the request, the scheduler can't find a suitable node for the Pod.

At this point, the Pod will remain in:

- `Pending`

### Essentially What Happens

You can simply understand it as:

### 1. Pod submitted to the cluster
### 2. Scheduler starts looking for nodes
### 3. Finds all nodes can't accommodate
### 4. Pod can't complete binding
### 5. So remains in Pending

### Operations Understanding Focus

At this point, the Pod isn't "failed to run", but:

> **Hasn't even had the chance to start running yet.**

---

## VI. What Are the Most Common Resource-Related Pending Scenarios

### 1. CPU requests too high
For example, in a test environment with small nodes, but the Pod requests:

- `cpu: 2`
- `cpu: 4`

The scheduler might directly find no available nodes.

---

### 2. Memory requests too high
For example:

- `memory: 4Gi`
- `memory: 8Gi`

While the test node only has very little available memory.

---

### 3. Single Pod isn't large, but too many replicas
For example:

- A Pod requests `500m CPU + 512Mi Memory`
- `replicas: 10`

The first few Pods might schedule normally, but later ones start Pending.

---

### 4. Many Pods have already consumed all requests
Although the node's current instantaneous usage isn't high, the scheduler thinks resources are already reserved.

---

### 5. HPA expansion causes new replicas to fail scheduling
Existing replicas are still running, but new expanded Pods fail due to resource insufficiency and remain in Pending.

---

## VII. What to Check First When Seeing Pending

Don't rush to check logs.

Because in many Pending scenarios:

- Containers haven't even started
- No business logs exist

### Correct First Step

Check the Pod list first:

    kubectl get pod -n <namespace>

For example, you might see:

    NAME            READY   STATUS    RESTARTS   AGE
    nginx-web-1     0/1     Pending   0          2m

### What to Focus On at This Point

- Is `STATUS` `Pending`?
- Is `RESTARTS` still `0`?
- Has it been ongoing for a long time?

### Operations Understanding Focus

If it's:

- `Pending`
- `RESTARTS = 0`

That's likely:

> **Hasn't even entered the container runtime phase yet.**

## VIII. What is the most critical command for troubleshooting Pending

The most critical command is:

    kubectl describe pod <pod-name> -n <namespace>

This is the first entry point for troubleshooting Pending.

Because this typically directly tells you:

- Whether scheduling has been successful
- What happened in Events
- Why the scheduler rejected it
- Whether it's due to resource insufficiency
- Whether it's related to PVC, taints, affinity, etc.

---

## IX. What sections to focus on when describing a Pod

### 1. `Status`
Check whether it is indeed:

- `Pending`

---

### 2. `Node`
Check whether it has been assigned to a node.

If scheduling hasn't been successful yet, this is often:

- Empty
- Or lacks valid node information

---

### 3. `Conditions`
Focus on:

- `PodScheduled`

If it is:

- `PodScheduled=False`

It usually indicates that scheduling hasn't been successful yet.

---

### 4. `Events`
This is the most critical section.

### Operations Understanding Focus

Many Pending issues don't require guessing.  
Events often clearly indicate the direction.

---

## X. What do Events commonly show when resource insufficiency occurs

If the scheduling failure is resource-related, the most common events are:

- `0/3 nodes are available`
- `Insufficient cpu`
- `Insufficient memory`
- `node(s) didn't have enough cpu`
- `node(s) didn't have enough memory`

### How to understand these events

#### `0/3 nodes are available`
Indicates the scheduler checked 3 nodes but none met the criteria.

#### `Insufficient cpu`
Indicates the node doesn't have enough CPU request space to satisfy this Pod.

#### `Insufficient memory`
Indicates the node doesn't have enough memory request space to satisfy this Pod.

### Operations Understanding Focus

Whenever you see these events, prioritize determining:

> **Whether resources.requests are set too high, or the node's overall resources are insufficient.**

---

## XI. A basic example of resource insufficiency causing Pending

For example, you create a Pod like this:

    apiVersion: v1
    kind: Pod
    metadata:
      name: cpu-memory-heavy-pod
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

If your test node resources are inherently small, the result is likely:

    kubectl get pod

You'll see:

    NAME                   READY   STATUS    RESTARTS   AGE
    cpu-memory-heavy-pod   0/1     Pending   0          1m

Then executing:

    kubectl describe pod cpu-memory-heavy-pod

May show Events like:

- `Insufficient cpu`
- `Insufficient memory`

---

## XII. Why a node "appears to have resources" but a Pod still remains Pending

This is a highly frequent practical confusion.

Many people check:

- `top`
- `htop`
- Grafana
- Node CPU usage
- Node memory usage

Then say:

> "This node isn't fully utilized, why can't it schedule?"

Here we must re-emphasize a core understanding:

> **The scheduler looks at allocatable resources and already allocated requests from the Kubernetes perspective, not instantaneous usage.**

### Distinguish several concepts

#### 1. Node total resources
How much CPU and memory the machine has in total.

#### 2. allocatable
Resources actually available to Pods.

#### 3. Already allocated requests
Resources already requested and occupied by existing Pods.

#### 4. Actual runtime usage
Just the real usage at a certain moment, not equal to "whether scheduling is possible."

### Operations Understanding Focus

So when troubleshooting Pending, you can't rely solely on system monitoring, you must also understand:

> **The scheduler makes decisions based on the "resource request perspective."**

---

## XIII. Why you need to check the Pod YAML's requests

If Events indicate:

- `Insufficient cpu`
- `Insufficient memory`

The next step shouldn't be focusing only on the node, but also looking back:

> **How much resources this Pod actually requested.**

Check the Deployment / Pod YAML's:

    resources:
      requests:
        cpu: ...
        memory: ...

### Many issues fundamentally fall into these categories

#### 1. Requests set too high
Normal business usually requires:

- `100m`
- `128Mi`

But it's set to:

- `2 CPU`
- `4Gi`

---

#### 2. Requests don't match the test environment capacity
Notes examples or production templates are directly copied to small clusters, causing resource overflows.

---

#### 3. Single Pod looks fine, but total replicas exceed capacity
Individual requests are fine, but total exceeds.

---

## XIV. A basic troubleshooting order for resource-related Pending

Recommend a fixed troubleshooting approach first.

### Step 1: Check Pod status

    kubectl get pod -n <namespace>

Confirm:

- Whether `Pending`

---

### Step 2: Check Pod details

    kubectl describe pod <pod-name> -n <namespace>

Focus on:

- `PodScheduled`
- `Events`
- Whether there is `Insufficient cpu / memory`

---

### Step 3: Check resource configuration

Review the YAML's:

- `requests.cpu`
- `requests.memory`

---

### Step 4: Check node resources

You can check:

    kubectl describe node <node-name>

Focus on:

- `Allocatable`
- Already allocated resources
- Node pressure

---

### Step 5: Determine resolution direction

Common resolution directions are typically only two types:

#### 1. Adjust Pod Resource Requests
For example, reduce the requests.

#### 2. Increase Cluster Available Capacity
For example, add nodes, scale nodes, let Cluster Autoscaler take over.

---

## Fifteen, Other Issues That Can Cause Pending Besides Resource Insufficiency

Although this article mainly discusses resource insufficiency, you need to first understand:

> **Pending does not mean it's always a resource issue.**

You may later encounter these directions:

### 1. PVC Not Successfully Bound
For example, the storage class is not yet ready.

### 2. Node Selection Conditions Not Met
For example:

- `nodeSelector`
- `nodeAffinity`

Too restrictive.

### 3. Taint Mismatch
Nodes have taint, but the Pod lacks toleration.

### 4. Pod Affinity/Anti-affinity Conditions Too Strict
Causing no nodes to meet the conditions.

### Operations Understanding Focus

So when troubleshooting, you cannot mechanically assume:

- Pending = Definitely resource insufficiency

But in your current stage, resource insufficiency is the first thing to master and the most common type.

---

## Sixteen, Why This Article Follows "How Requests Affect Scheduling"

Because these two articles are naturally connected.

The previous article solves:

- Scheduler mainly looks at requests
- How requests affect Pod placement

This article solves:

- What happens when requests cannot be met
- Why Pod becomes Pending
- How to transition from phenomenon to troubleshooting

In other words, these two articles together form the most basic loop:

### 1. Pod has requests
### 2. Scheduler finds nodes based on requests
### 3. No suitable nodes found
### 4. Pod Pending
### 5. Troubleshoot via describe / events

This is the most basic entry point for resource scheduling and troubleshooting.

---

## Seventeen, Common Mistakes in This Type of Issue

### 1. Immediately Check Container Logs When Seeing Pending
Often, the container hasn't even started yet, so checking logs is meaningless.

---

### 2. Only Look at `kubectl get pod`, Not `describe`
This prevents seeing events and scheduling failure reasons.

---

### 3. Only Check Current Node CPU / Memory Usage
Ignored:

- allocatable
- Already allocated requests
- Scheduler's perspective

---

### 4. Mistakenly Think Limits Determine Scheduling
At this stage, remember:

> **The most critical thing is to look at requests first.**

---

### 5. Only Suspect Node Resources, Not Recheck YAML
Often, the issue lies in:

- requests written too large
- Too many replicas

---

## Eighteen, How to Answer "Why Pod is Pending" in Interviews

You can first give a layered answer:

### First Layer: Give the General Answer
Pod Pending means the Pod has been received by the cluster but hasn't completed scheduling or entered the actual running phase.

### Second Layer: Mention Common Directions
Common reasons include:

- Node resource insufficiency
- requests too large
- Node selection conditions not met
- Taint mismatch
- PVC not successfully bound

### Third Layer: Mention Troubleshooting Methods
Prioritize checking:

- `kubectl describe pod`
- `Events`
- `PodScheduled`
- Whether there is `Insufficient cpu / memory`

This answer is more complete than just saying "resource insufficiency."

---

## Nineteen, Key Understandings in This Article

### 1. Pending Is Not Synonymous with "Application Startup Failure"
Often, the application hasn't even started running yet.

### 2. Resource Insufficiency Is the Most Common Cause of Pending
Especially when CPU/memory requests cannot be met.

### 3. describe Pod Is the First Entry Point for Troubleshooting Pending
Not logs, not guessing.

### 4. Events Often Directly Tell You the Direction
For example:

- `Insufficient cpu`
- `Insufficient memory`

### 5. Scheduler Looks at Resource Requests and Allocatable Resources, Not Instant Usage
This is the key premise for troubleshooting resource-related issues.

### 6. Don't Only Look at Nodes, Also Recheck Pod's Own requests
Many Pending issues are fundamentally due to unreasonable YAML.

---

## Twenty, What Level Should You Master to Learn This Article

At this stage, it's recommended to reach the following level:

### 1. Be able to explain the basic meaning of Pending
### 2. Understand why resource insufficiency causes Pending
### 3. Be able to check Events via `kubectl describe pod`
### 4. Be able to identify `Insufficient cpu / memory` and similar common events
### 5. Establish a basic troubleshooting order for Pending
### 6. Distinguish between "scheduling failure" and "container runtime failure" as two different issues

---

## Twenty-one, Common Follow-up Questions in Interviews

Common questions in this area include:

- Why does a Pod become Pending?
- What's the difference between Pending and CrashLoopBackOff?
- How to troubleshoot a Pending Pod?
- What does `Insufficient cpu` indicate?
- What does `Insufficient memory` indicate?
- Why can't a Pod be scheduled even though there's idle resource on the node?
- Why is there no log but the Pod remains Pending?
- What's the relationship between requests and Pending?

---

## Twenty-two, Stage Summary

Pod Pending is one of the most important entry states in Kubernetes application deployment and troubleshooting.

The most important part of this article isn't memorizing all possible causes at once, but first establishing these core understandings:

- Pending indicates the Pod hasn't yet entered a stable running phase
- The most common cause is resource insufficiency leading to scheduling failure
- The first entry point for troubleshooting Pending is `kubectl describe pod`
- Events often directly indicate the direction
- Scheduler looks at requests, allocatable, and allocated resources, not just node instantaneous usage rates
- When judging resource-related Pending, both nodes and Pod YAML should be checked

As long as these understandings are established, further learning will be more stable:

- QoS
- Eviction
- HPA
- More complex scheduling constraints

---

## Twenty-three, Keyword Quick Notes

- Pending: Pod has been created but hasn't completed scheduling or runtime preparation
- scheduler: Scheduler, responsible for selecting nodes for Pods
- PodScheduled: Whether the Pod has successfully scheduled to a node
- Insufficient cpu: CPU resource insufficiency
- Insufficient memory: Memory resource insufficiency
- requests: Resource requests, important basis for scheduling
- allocatable: Resources available for Pods on the node
- Events: First information source for troubleshooting Pending

---

## Twenty-four, Operational Extended Understanding

From an operational perspective, Pending is a very important "pre-state" that deserves attention.

It reminds you:

- Issues may not occur during the application's runtime phase  
- Many deployment failures actually stem from problems in the early stages of deployment  
- Scheduling, resources, storage, and node constraints — these foundational conditions determine a Pod's fate before business logic even starts  

This is why, in Kubernetes, mature troubleshooting approaches typically aren't:  

- Jumping straight to logs  
- Immediately suspecting application code  

Instead, the focus is first on determining:  

- Where exactly the Pod is in its process  
- Whether scheduling failed  
- Whether the image couldn't be pulled  
- Whether the container crashed after starting  

The value of this "Pending" article isn't just learning a state name — it's about establishing a:  

> **Layered troubleshooting mindset by phase.**  

---

## References  
- Kubernetes Pod Lifecycle  
- Kubernetes Scheduler Fundamentals  
- Common Outputs of `kubectl describe pod` for Troubleshooting  
- Kubernetes Resource Management for Pods and Containers  

---

## Tomorrow's Suggestions  
The next article suggests organizing:  

[[04-QoS Basics - Guaranteed Burstable and BestEffort]]