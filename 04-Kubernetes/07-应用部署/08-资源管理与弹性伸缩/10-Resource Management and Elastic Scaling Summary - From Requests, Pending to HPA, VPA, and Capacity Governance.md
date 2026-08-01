# 10 - Resource Management and Auto Scaling Stage Summary: From requests, Pending to HPA, VPA, and Capacity Governance

## Document Notes
- Document Positioning: Kubernetes Resource Management and Auto Scaling Stage Summary
- Applicable Stage: After completing requests/limits, resource scheduling, Pending, QoS, Eviction, HPA, VPA basics, used forStage review and consolidation
- Recommended Path: `04-Kubernetes/07-Apply deployment/08-Resource management and flexibility/10-Summary of the resource management and elasticity phase: requestsI don't know.Pending Present. HPAI don't know.VPA and capacity governance.md`

## Tags
#Kubernetes #ResourceManagement #FlexibleStretch #requests #limits #Pending #QoS #Eviction #HPA #VPA #CapacityGovernance #ApplyDeployment #Clouds. #SummaryOfPhases #Transport #InterviewNotes

---

## I. What is This Stage About

At first glance, this entire set of content appears to be several scattered topics:

- requests / limits
- Pod scheduling
- Pending
- QoS
- Eviction
- HPA
- VPA
- Relationship between HPA and VPA

But from an operations perspective, they are actually answering the same type of question throughout this stage:

> **How do resources in Kubernetes affect whether an application can be placed, run stably, avoid resource exhaustion, scale automatically, and whether scaling is effective.**

In other words, this stage isn't about isolated knowledge points, but building a complete main thread:

### 1. Applications first declare resource requirements
That is:

- requests
- limits

### 2. The scheduler decides whether to place the Pod on a node based on resource requirements
If it can't be placed, it will:

- Pending

### 3. After placement, Pods aren't treated equally in terms of resource guarantees
This involves:

- QoS

### 4. When node resource pressure is too high, not all Pods can stay stable
This involves:

- Eviction

### 5. When business load changes, it continues to involve automatic elasticity issues
This involves:

- HPA
- VPA

### 6. But elasticity isn't effective just because it's triggered
Because it will continue to be affected by:

- Scheduling
- Resource capacity
- Single instance resource specifications
- Node resource boundaries

These factors.

Therefore, this stage essentially establishes a very critical understanding:

> **Resource management isn't "writing a few fields", but a main thread that runs through scheduling, operation, eviction, scaling, and capacity governance.**

---

## II. How to Remember the Main Thread of This Stage

If we compress all the previous content into the minimal main thread, we can first remember it as:

> **Resource declaration → scheduling decision → Pending/Running → QoS hierarchy → node pressure → Eviction → HPA horizontal elasticity → VPA vertical governance → whether capacity can truly support**

In other words, the previous several articles, although with different themes, are actually answering questions in order of time and stage.

---

### First Stage: Before the Pod Actually Runs
Mainly answers:

- What are requests/limits
- What does the scheduler look at first
- Why a Pod might be Pending

That is:

> **Whether this Pod can be placed first.**

---

### Second Stage: After the Pod is Placed and Running
Mainly answers:

- What is QoS
- How node resource pressure affects different Pods
- Why Eviction happens

That is:

> **After this Pod is placed, can it stay stable during resource contention.**

---

### Third Stage: After Business Load Starts to Change
Mainly answers:

- How HPA automatically scales replicas
- Why HPA scaling doesn't necessarily mean the business can handle it
- How VPA governs single Pod resource specifications
- What problems HPA and VPA respectively solve

That is:

> **When business changes, should we scale replicas or adjust single instance resource specifications.**

---

## III. requests/limits Are the Starting Point of This Entire Stage

The most fundamental starting point of this stage is always:

- `requests`
- `limits`

Because many issues eventually come back to these two fields.

### 1. What is requests
It can be understood as:

> **The minimum resources the container hopes to be reserved.**

One of its most important functions is:

- Affecting whether the scheduler can place the Pod on a node

### 2. What is limits
It can be understood as:

> **The maximum resources allowed for the container to use during operation.**

It is more focused on:

- Operational resource boundaries
- CPU upper limit / memory upper limit
- Limitation or abnormal behavior when exceeding limits

### Operations Understanding Focus

These two fields are not static configurations that "end after being written", but the starting point for many subsequent mechanisms:

- requests affect scheduling
- limits affect operational boundaries
- requests/limits jointly affect QoS
- Whether resource configuration is reasonable will continue to affect Eviction risk
- requests will also continue to affect HPA judgment and overall resource utilization
- Long-term unreasonable requests/limits will lead to the governance value of VPA

---

## IV. Why the Scheduler Prioritizes requests First

This is one of the most important foundational understandings of this entire stage.

The scheduler's problem to solve is:

> **Whether this Pod can be placed on a certain node first.**

It needs a:

- Resource requirement value that can be obtained before scheduling
- Declared
- Calculable
- Comparable

This value is most critical:

- `requests.cpu`
- `requests.memory`

### Why isn't limits checked first
Because limits are more focused on:

- How much the container can consume at most after running
- It's an operational boundary
- Not the most direct placement basis before scheduling

### Operations Understanding Focus

This is why you often see in many scenarios:

- Nodes "seem to have available resources"
- But the Pod is still Pending

Because the scheduler looks at what Kubernetes considers:

- allocatable
- Declared requests
- Remaining available space

So this conclusion must be remembered firmly:

> **The scheduler prioritizes requests first.**

---

## V. Why a Pod Might Be Pending

Pending is the first phenomenon in this group of resource management that enters the troubleshooting perspective.

Many people new to Kubernetes, when they see a Pod not starting, their first reaction is:

- Image issues
- Startup command issues
- Configuration issues
- Program startup failure

But the first distinction to establish in this group is:

> **A Pod in Pending status doesn't necessarily mean the application failed to start, but it might be that the scheduling hasn't completed yet.**

### The most typical category of Pending reasons is:
- CPU requests too large
- Memory requests too large
- Node overall capacity insufficient
- Resource can't fit after replica count aggregation
- New Pods expanded by HPA also lack placement

### Operations Understanding Focus

Therefore, the troubleshooting order for Pending should be to first check:

    kubectl describe pod <pod-name>

Focus on: /think

- `PodScheduled`
- `Events`
- Did it appear:
  - `Insufficient cpu`
  - `Insufficient memory`

What's actually being established at this level isn't a specific command, but rather a habit:

> **Before a Pod starts, first distinguish whether it's "not yet scheduled" or "failed after being scheduled."**

---

## Six. QoS Connects Resource Configuration and Runtime Guarantees

Once a Pod has been scheduled and started running, the issue is no longer just:

- Can it be scheduled

It becomes:

- Why some Pods are more stable when node resources are tight
- Why some Pods are more prone to issues
- Why formal business operations don't recommend completely omitting resource configuration

This is when we enter:

> **QoS (Quality of Service, Resource Guarantee Level)**

### Three QoS Categories

#### 1. Guaranteed
Most clearly defined resource boundaries, strongest guarantees

#### 2. Burstable
Has resource declarations but hasn't met the maximum guarantee conditions, the most common category

#### 3. BestEffort
Completely no requests/limits declared, weakest guarantees

### Operations Understanding Focus

This QoS section truly establishes not "remembering three names," but understanding:

> **Pods aren't starting from the same line when competing for resources.**

Kubernetes assigns different resource guarantee levels to Pods based on their resource declaration methods.  
This also directly sets the stage for the following Eviction discussion.

---

## Seven. Eviction Fully Explains the "Node Stability First" Principle

When learning QoS earlier, you might think it's just a classification system.  
It's only when reaching Eviction that this classification's value truly begins to show.

### What is Eviction
You can initially understand it as:

> **When node resource pressure becomes too high, kubelet will actively evict certain Pods from the node to protect the node itself.**

The most important understanding here is:

- Not the application itself voluntarily exiting
- Not the container itself crashing
- Not just a probe failure

Rather:

> **The node itself has entered a dangerous zone, and the system begins prioritizing node protection.**

### Common Node Pressures Include:
- `MemoryPressure`
- `DiskPressure`
- `PIDPressure`

### Relationship Between QoS and Eviction
At this stage, we can first establish a basic direction:

- `BestEffort` is more fragile
- `Burstable` is in the middle
- `Guaranteed` is typically more stable

### Operations Understanding Focus

This Eviction section truly aims to establish the "node perspective":

> **If the node itself can't sustain, all Pods on the node will eventually have issues together.**

So in resource governance, Kubernetes' approach is often:

- Prioritize node protection first
- Then try to protect business as much as possible

---

## Eight. HPA Advances Resource Management from Static Configuration to Dynamic Replica Elasticity

Up to this point, this group of content has mostly been discussing:

- How to declare resources
- How scheduling occurs
- Why Pods are Pending
- Why node pressure causes Pod evictions

With HPA, the question becomes:

> **If business load changes, can replica numbers automatically adjust?**

### HPA's Core Capabilities
- Adjust Pod replica counts based on CPU, memory, or other metrics
- Automatically scale replica counts

### It Solves:
> **Quantity issues**

That is:

- Are enough Pods available
- Should more Pods be created during traffic peaks
- Should replicas be scaled back during traffic troughs

### Operations Understanding Focus

HPA doesn't make individual Pods stronger, but enables more Pods to share the load together.  
So its core is:

> **Horizontal scaling**

That is:

- Increase or decrease replica counts
- Rather than changing individual Pod resource specifications

---

## Nine. Why HPA May Still Fail to Start Pods Even After Expansion

This is one of the most critical real-world conclusions in this phase.

Many people, after learning HPA, naturally form an impression:

- Metrics are high
- HPA expands
- More Pods are created
- The problem is solved

But the reality isn't necessarily this way.

### Typical Chain of Events is:

### 1. HPA detects high metrics
### 2. HPA increases Deployment replica count
### 3. New Pods are created
### 4. Scheduler continues to find nodes for these new Pods
### 5. Node resources are insufficient
### 6. New Pods are Pending
### 7. The result is "it looks like it expanded," but business benefits are limited

### Operations Understanding Focus

This section truly establishes an important real-world understanding:

> **HPA only handles expanding Pods, not creating node capacity.**

So effective horizontal elasticity isn't just the existence of HPA objects, but:

- New replicas actually get created
- New replicas actually get scheduled successfully
- New replicas actually become Ready
- New replicas actually handle traffic

This is why HPA ultimately comes back to:

- requests
- scheduling
- node capacity
- Pending

These topics we've already studied earlier.

---

## Ten. VPA Advances Resource Governance from "Replica Count" to "Single Instance Specification"

After HPA, many people naturally assume:

- Automatic scaling isn't just HPA

But in real production, there's another type of issue that isn't resolved by "adding more replicas."

For example:

- requests are long guessed by experience
- Some Pods are configured too small and often experience resource tension
- Some Pods are configured too large and waste resources long-term
- These loads aren't best addressed by adding more replicas

At this point, what truly needs to be solved isn't:

- Whether to add more replicas

But rather:

- **How much resource specification should a single Pod have to be more reasonable**

This is when we enter:

> **VPA (Vertical Pod Autoscaler)**

### VPA's Core Capabilities
- Provide or push applications for more reasonable CPU/Memory resource recommendations based on usage

### It Solves:
> **Specification issues**

That is:

- Whether single Pod resource specifications are reasonable
- Whether requests/limits have long deviated from actual usage

### Operations Understanding Focus

If HPA answers:

> **Are enough replicas available**

Then VPA answers:

> **Are single Pod resource specifications reasonable**

---

## Eleven. Why VPA Must Emphasize "Installation Prerequisites" and "Custom Resources" Separately

This is a key difference between VPA and HPA.

HPA is easier to understand as part of the cluster's existing capabilities in daily learning paths.  
But VPA can't be simply understood this way.

### The Most Critical Conclusion at This Stage is:

> **VPA isn't a native object you can use directly by default.**

It typically depends on:

- Additional installed components
- CRD
- recommender/updater/admission capabilities

The most critical point among these is:

> **`VerticalPodAutoscaler` is a custom resource introduced into the cluster through CRD.**

In other words, first having:

- `CustomResourceDefinition`

The cluster only recognizes:

- `VerticalPodAutoscaler`

Plus controller components, it's not just "able to create objects," but "truly functional."

### Operations Understanding Focus

This VPA section truly establishes this layer of understanding:

- Installing VPA capabilities
- And creating business VPA objects

Are two different things.

---

## Twelve, Understanding the Relationship Between HPA and VPA

This is the final thing that must be concluded in this phase.

Many people will ask:

- Is HPA going to replace VPA?
- Is one for stateless and the other for stateful?
- Can they be configured together and that's it?

The most stable understanding at this stage should be:

### HPA focuses on:
- Quantity elasticity
- Whether multiple Pods can share traffic

### VPA focuses on:
- Resource governance
- Whether the resource allocation of a single Pod is reasonable

So they are not replacements for each other, but:

> **Different focus areas**

---

## Thirteen, Why "One is for stateless, one is for stateful" is not rigorous enough

This phrase you often see in learning and interviews:

- HPA is suitable for stateless
- VPA is suitable for stateful

The direction is not wrong, but it's too absolute.

### A more accurate understanding should be:

#### HPA is more suitable for:
- Horizontally scalable applications
- Scenarios where multiple replicas collaborate to handle traffic
- Many stateless Web/API services naturally fit this

#### VPA is more suitable for:
- Single-instance resource specification optimization
- Scenarios where it's inconvenient to solve problems by simply adding replicas
- Some stateful components are more common

### Operations Understanding Focus

Therefore, a more mature way of judgment is not to first ask:

- Is it stateful or stateless?

But to first ask:

> **Is it currently lacking more instances, or more reasonable single-instance resource specifications?**

---

## Fourteen, What is the core logic loop in this phase

By now, this entire set of content has already formed a very complete logic loop.

You can compress it into the following chain:

### 1. Applications first declare requests / limits
### 2. Scheduler makes placement decisions based on requests
### 3. If it can't be placed, it becomes Pending
### 4. After being placed, different Pods are categorized into different QoS levels
### 5. When node resource pressure is high, Eviction may occur
### 6. When business load changes, HPA automatically adjusts replica count
### 7. But the new Pods created by expansion still need to go through scheduling and node capacity validation
### 8. If the issue is not replica count, but single Pod specification inaccuracies, it enters VPA
### 9. HPA solves quantity elasticity, VPA solves specification governance
### 10. The final focus is not whether to configure a certain object, but whether to determine which layer the problem belongs to

### Operations Understanding Focus

The real thing this main line aims to build is not "knowing several fields", but:

> **Knowing how to judge the problem layer by stage.**

---

## Fifteen, The most common misunderstanding in this phase

### 1. Thinking requests / limits are just two ordinary YAML fields
In reality, they will continue to affect:

- Scheduling
- QoS
- Eviction
- HPA stability
- VPA value

---

### 2. Thinking Pending means the application is broken
Often, the application hasn't even entered the startup phase yet.

---

### 3. Thinking QoS is just a classification question
In fact, it affects resource competition and the hierarchy of guarantees under node pressure.

---

### 4. Thinking Eviction means the business itself failed
Often, it's the node actively cleaning up Pods to protect itself.

---

### 5. Thinking HPA expansion equals solving the problem
Not necessarily, because new Pods still need to check:

- requests
- Scheduling
- Node capacity
- Ready status

---

### 6. Thinking VPA is "automatically adding resources to Pods"
In fact, it's more like a:

- Recommendation
- Resource governance
- Specification continuous optimization

mechanism, not a simple button.

---

### 7. Thinking HPA and VPA are sufficient as one for stateless and one for stateful
This is just a rough memory, not rigorous enough.
The real thing to determine first is:

- Quantity issues
- Or specification issues

---

## Sixteen, What level should you reach after this phase

After this phase, the goal is not "mastering resource governance", but at least reaching a solid beginner-to-practical level.

You should be able to at least do the following things.

### 1. Be able to explain what requests / limits affect separately
- requests is more biased towards scheduling
- limits is more biased towards runtime boundaries

### 2. When seeing Pending, know to check scheduling first instead of blaming the application first
Will first check:

- `describe pod`
- `Events`
- `Insufficient cpu / memory`

### 3. Be able to understand the relationship between QoS and node resource pressure
Know:

- Whether resource configuration is standardized affects the hierarchy of guarantees during resource tension

### 4. Be able to distinguish phenomena like OOM, Pending, Eviction, which are not in the same layer
At least not mix them into one category of problems.

### 5. Be able to explain what HPA solves
Know it's more biased towards:

- Horizontal scaling of replicas
- Quantity elasticity

### 6. Be able to explain why HPA expansion may not be useful
Because new Pods still need to go through:

- requests
- Scheduling
- Node capacity

### 7. Be able to explain what VPA solves
Know it's more biased towards:

- Single-instance resource governance
- Specification optimization
- Resource recommendations

### 8. Be able to explain the relationship between HPA and VPA
At least know:

- HPA manages quantity
- VPA manages specifications
- Neither replaces the other

---

## Seventeen, How to answer "How do you understand Kubernetes resource management and elastic scaling" in an interview

You can organize it according to the following logic:

Kubernetes resource management is not just a few fields, but a complete chain. First, Pods declare resource needs through requests and limits, where requests directly affect whether the scheduler can place the Pod on a node, so resource shortages may result in Pending. After Pods run, Kubernetes will classify Pods into QoS levels based on resource declarations, and under node resource tension, different levels of Pods have different guarantee hierarchies, and severe cases may result in Eviction. In terms of elasticity, HPA mainly adjusts replica count based on metric changes, which is horizontal scaling; VPA mainly provides or pushes single Pod resource specification optimization based on resource usage, which is vertical scaling. So this entire set of content essentially answers: whether the application can be placed, whether it can run stably after running, whether to scale replicas or adjust single-instance resource specifications when traffic changes, and whether the cluster capacity can support these changes.

### These keywords are recommended to remember in this answer
- requests influence scheduling
- Pending is an important phenomenon in scheduling before and after
- QoS and Eviction reflect resource competition during runtime
- HPA is horizontal scaling
- VPA is vertical resource governance
- The key is to determine which layer the problem belongs to

---

## Eighteen, The most important conclusions in this phase

### 1. Resource management spans the entire application lifecycle
From scheduling, runtime, to eviction and elasticity, resources are always involved.

### 2. requests is the core entry point for understanding scheduling and Pending
The scheduler's most critical first step is to look at requests.

### 3. QoS and Eviction truly connect resource configuration with runtime stability
Resource configuration doesn't only affect scheduling, but also the guarantee hierarchy during resource competition.

### 4. HPA solves quantity elasticity
Its core is: /think

- More Pods  
- Fewer Pods  

### 5. VPA Solves Specification Governance  
Its core is:  

- Whether the resource allocation for a single Pod is reasonable  

### 6. HPA and VPA Are Not Alternatives  
One focuses on quantity, the other on specifications.  

### 7. True Resource Governance Capability Isn't About Knowing Several Objects, But Judging Which Layer the Problem Belongs To  
This is the core capability of this stage.  

---  

## Nineteen. Phase Summary  

Although this group of content has many names, it fundamentally revolves around one very practical issue:  

> **Applications in Kubernetes aren't just about "running," but also about "being able to scale, running stably, resource control, and having elasticity when business changes, with the elasticity actually being effective."**  

In other words, what this stage truly completes isn't just an introduction to resource fields, but a complete shift in thinking:  

From:  
- Only knowing how to write Deployment / Service  
- Only knowing how to check if Pods are running  

To:  
- Knowing how to check requests / limits  
- Distinguishing between Pending and runtime failures  
- Understanding QoS and Eviction  
- Understanding why HPA isn't equal to universal elasticity  
- Understanding why VPA is a specification governance capability  
- Judging whether to solve the problem from the quantity or specification perspective  

If we compress the value of this group into one sentence, it could be said:  

> **This stage marks the transition from "knowing how to deploy applications" to "understanding how resources determine an application's operability, stability, and elasticity."**  

---  

## Keyword Quick Notes  

- requests: Scheduling baseline resources  
- limits: Resource upper bounds during runtime  
- Pending: Common scheduling phase issue  
- QoS: Resource guarantee level  
- Eviction: Active eviction under node pressure  
- HPA: Horizontal scaling, adjusts replica count  
- VPA: Vertical scaling, adjusts single Pod resource specifications  
- Quantity issues: Prioritize HPA  
- Specification issues: Prioritize VPA  
- Capacity governance: Not just looking at objects, but the entire resource chain  

---  

## Operational Extended Understanding  

From an operational perspective, this stage truly establishes not "resource field knowledge points," but a way of thinking closer to production environments.  

In production, many issues have similar surface phenomena but completely different root causes:  

- Pod not starting up could be Pending, not the app being broken  
- Pod being evicted could be Eviction, not the program crashing itself  
- HPA scaling could result in new Pods still being Pending, not elasticity truly taking effect  
- Resources always being misconfigured might actually lack VPA-style specification governance capabilities  

A mature resource governance perspective focuses not on "changing YAML when problems occur," but first determining:  

- Is it a scheduling layer issue?  
- Is it a runtime resource contention issue?  
- Is it a node pressure issue?  
- Is it a quantity elasticity issue?  
- Or is it a single instance specification issue?  

This is why this stage, although discussing resources, fundamentally trains an important capability:  

> **Understanding how resources influence business availability layer by layer along an application's real lifecycle.**  

---  

## References  
- Resource Management for Pods and Containers  
  https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/  
- Horizontal Pod Autoscaling  
  https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/  
- Vertical Pod Autoscaler  
  https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler  
- Kubernetes QoS for Pods  
  https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/  
- Node-pressure Eviction  
  https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/  

---  

## Tomorrow's Suggestions  
Next post suggestion to organize:  

[[01-NodeSelector Basics - Pod Scheduling to Specified Nodes]]