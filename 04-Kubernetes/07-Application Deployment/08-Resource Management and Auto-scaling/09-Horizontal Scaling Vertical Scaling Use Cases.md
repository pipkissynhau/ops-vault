# 09-Horizontal and Vertical Scaling: HPA and VPA, Cross-Scaling, Vertical Scaling, and Use Cases

## Document Notes
- Document Positioning: Kubernetes resource management and elasticity scaling phase,Combination HPA and VPA relationship and scenario selection introduction
- Applicable Stage: After understanding requests/limits, Pending, QoS, Eviction, HPA basics and VPA basics, used to establish overall understanding of horizontal and vertical scaling
- Recommended Path: `04-Kubernetes/07-Apply deployment/08-Resource management and flexibility/09-HPA and VPA: horizontal, vertical and applicable scenarios.md`

## Tags
#Kubernetes #HPA #VPA #HorizontalPodAutoscaler #VerticalPodAutoscaler #HorizontalAmplification #VerticalAmplification #ResourceManagement #FlexibleStretch #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## I. Why Do We Need to Organize the Relationship Between HPA and VPA After Learning Them Separately

Previously, we have already learned:

- What is HPA
- How HPA automatically adjusts replica count based on metrics
- Why HPA might fail to start even after scaling Pods
- What is VPA
- Why VPA needs to prepare metrics-server and VPA components first
- Why `VerticalPodAutoscaler` is a custom resource
- Why VPA beginners should start with `updateMode: Off` first

If we stop here, although we have individually recognized HPA and VPA, we will still immediately face several practical issues in actual work:

- Is the current problem more like HPA or VPA?
- When should we first add replicas?
- When should we first adjust single Pod resource specifications?
- Is HPA replacing VPA or vice versa?
- Are they suitable for stateless and stateful respectively?
- Can they be used together?
- Should we start from "more Pods" or "more resources per Pod" for resource issues?

Therefore, the core of this article is no longer to explain a single object separately, but to truly combine the two elasticity approaches:

> **What does HPA solve, what does VPA solve, what scenarios are they suitable for, and why they cannot be merged into one concept.**

---

## II. First, Give the Most Core Conclusions

If this article only needs to remember a few sentences, it is recommended to first remember the following conclusions.

### 1. HPA is Horizontal Scaling
It mainly does:

> **Adjust replica count based on metric changes.**

### 2. VPA is Vertical Scaling
It mainly does:

> **Provide more reasonable CPU/Memory requests suggestions for single Pods, or push resource specification updates in appropriate modes.**

### 3. HPA and VPA Are Not Replacements
They solve two different dimensions of problems:

- HPA focuses more on quantity
- VPA focuses more on specifications

### 4. HPA is More Suitable for Horizontal Scaling Scenarios
Typically for many stateless Web/API services.

### 5. VPA is More Suitable for Single-Instance Resource Optimization Scenarios
Typically for:

- Single Pod resource specifications are long-term underestimated
- It's inconvenient to solve problems by simply adding replicas
- More common in some stateful components

### 6. "HPA is for stateless, VPA is for stateful" is not entirely wrong, but too absolute
A more accurate statement should be:

> **HPA is more suitable for horizontal scaling scenarios; VPA is more suitable for single-instance resource optimization scenarios, more common in stateful components, but not limited to stateful.**

---

## III. What is the Most Fundamental Difference Between HPA and VPA

The most fundamental difference can be understood from "what they change".

### What Does HPA Change
HPA changes:

- `replicas`
- That is, the replica count of the workload

For example:

- Originally 2 Pods
- Under high pressure becomes 4 Pods
- Under low pressure scales back to 2 Pods

So HPA's core action is:

> **Let more instances share the load, or reduce instance count during low traffic.**

---

### What Does VPA Change
VPA changes:

- Resource suggestions for single Pod containers
- For example CPU request, memory request
- In some modes, push these resource specifications to take effect

For example:

- Originally CPU request is `200m`
- VPA suggests increasing to `500m`

Or:

- Originally memory request is `256Mi`
- VPA suggests increasing to `1Gi`

So VPA's core action is:

> **Make single instances more reasonably configured.**

---

## IV. Why HPA Solves "Quantity Issues"

HPA's approach essentially is:

### When a Single Pod is Under Heavy Load
Don't assume "single Pod must be upgraded" first, but ask:

> **Can we add more Pods to share the load?**

This is particularly suitable for the following scenarios:

- Request volume increases
- CPU utilization continues to rise
- The service itself supports multi-replica load balancing
- Adding instances can naturally distribute the load

### Typical Understanding
Assume an API service now has 2 Pods:

- Each Pod is very busy
- CPU is continuously high

At this time, HPA's approach is:

- Not necessarily first give each Pod more CPU
- But scale Pod count to 3, 4, 5
- Let more replicas handle the traffic

### Operations Understanding Focus
Therefore, HPA is best at handling:

> **Concurrent pressure, traffic fluctuations, replica count elasticity**

Rather than:

> **Long-term optimization of single Pod resource specifications**

---

## V. Why VPA Solves "Specification Issues"

VPA's approach essentially is:

### When Single Pod Resource Declarations Are Long-Term Unreasonable
Don't continue to guess YAML first, but ask:

> **How much CPU and memory should this Pod have, can the system give more reliable suggestions?**

This is particularly suitable for the following scenarios:

- requests are long-term guessed by experience
- The same template is reused everywhere
- Some applications are not better with more replicas
- Under-provisioned resources easily cause OOM, performance jitter
- Over-provisioned resources cause long-term waste

### Typical Understanding
Assume a workload now only has 1 or few Pods:

- It's not the most typical traffic-based application
- Or even if a few more Pods are started, it can't directly solve the problem
- The core issue is currently inaccurate single-instance resource specifications

At this time, VPA's approach is:

- First check system suggestions
- Then decide the CPU/memory range for single Pod

### Operations Understanding Focus
Therefore, VPA is best at handling:

> **Single-instance resource profiling, resource declaration optimization, specification continuous calibration**

Rather than:

> **Quickly increasing replica count based on traffic instantaneous changes**

---

## VI. HPA and VPA Comparison: Which Dimensions Are Most Suitable to View

To avoid staying only at "horizontal/vertical" a few words, it's better to view from several actual dimensions.

### 1. Different Adjustment Objects

#### HPA
Adjusts:

- Replica count

#### VPA
Adjusts:

- Single Pod resource specifications

---

### 2. Different Problems Solved

#### HPA
More focused on solving:

- Whether replica count is sufficient
- Whether concurrent pressure needs more instances to share
- Whether expansion is needed during traffic peaks

#### VPA
More focused on solving:

- Are requests/limits long-term unreasonable  
- Is single-instance resource allocation too small or too large  
- Is resource configuration long-term deviating from actual usage  

---

### 3. Differences in Requirements for Application Types  

#### HPA  
Better suited for:  
- Applications that can naturally scale horizontally  
- Multi-replica collaborative work  
- Adding an instance can alleviate pressure  

#### VPA  
Better suited for:  
- Applications more dependent on single-instance specifications  
- Resource optimization value outweighs replica count changes  
- Not convenient to solve problems by simply increasing replicas  

---

### 4. Differences in Elastic Response Approaches  

#### HPA  
More inclined toward:  
- Responding to load changes with quantity elasticity  

#### VPA  
More inclined toward:  
- Responding to resource usage trends with specification optimization  

---

## Seven. Why HPA is Often Discussed Together with Stateless Applications  

This is mainly not because of the word "stateless" itself, but because stateless applications typically naturally meet HPA's prerequisites.  

### Characteristics of Typical Stateless Applications  
Examples:  
- Nginx  
- Web API  
- Most front-end/back-end interface services  
- Most typical Deployment business  

These applications often have:  
- Natural load balancing across multiple replicas  
- Adding a replica can directly alleviate traffic  
- No strong binding to local state  
- Minimal role differences between replicas  

Thus, the most natural elastic approach for these applications is:  

> **When traffic increases, start more Pods.**  

This is why HPA is often discussed together with stateless applications.  

### Operational Understanding Focus  
Therefore, a more accurate statement is not:  
- "HPA can only be used for stateless applications"  

Rather:  
> **Many stateless applications naturally suit HPA.**  

---

## Eight. Why VPA is More Frequently Mentioned in Stateful Components  

This is also not because "stateful" inherently equals VPA, but because many stateful components often face these real-world constraints:  
- Inconvenient to simply increase replicas  
- Adding replicas doesn't immediately solve business pressure  
- Role differences between replicas  
- More complex mechanisms like data, storage, master-slave, sharding  
- Some scenarios prioritize single-instance stability and resource boundaries  

In these scenarios, the first issue to address is often not:  
- "Add more Pods"  

But rather:  
- "How much resource should this Pod or this type of Pod get?"  

Thus, VPA is more likely to be discussed in these scenarios.  

### Operational Understanding Focus  
Therefore, a more accurate statement is not:  
- "VPA is only suitable for stateful applications"  

Rather:  
> **Many stateful or single-instance-sensitive scenarios prioritize VPA first.**  

---

## Nine. Why "One for Stateless, One for Stateful" is Inadequate  

This phrasing, though easy to remember, leads to several misconceptions.  

### Misconception 1: Assuming HPA Can't Be Used in Any Stateful Scenario  
This is incorrect.  
Some stateful workloads may also have limited horizontal scaling, though with higher complexity and not as natural as stateless applications.  

### Misconception 2: Assuming VPA Can't Be Used in Stateless Applications  
This is also incorrect.  
Even stateless applications may face:  
- Clearly suboptimal requests  
- Severe resource misalignment in single Pods  
- Need for resource recommendations and specification calibration  

In these cases, VPA still holds value.  

### Misconception 3: Categorizing Only by "Stateful / Stateless" and Ignoring True Problem Dimensions  
The key question to ask first is actually:  
- Is this a replica count issue or a specification issue?  
- Is this a concurrency distribution issue or a single-instance resource issue?  
- Is this a traffic fluctuation issue or a long-term inaccurate resource declaration?  

### Operational Understanding Focus  
Thus, a more mature approach is not to first ask:  
- "Is it stateful or stateless?"  

But rather to first ask:  
> **Is it lacking "more instances" or "more suitable single-instance resource specifications"?**  

---

## Ten. When is HPA Most Suitable?  

Currently, we can summarize the most suitable scenarios for HPA as follows:  

### 1. Traffic-Based Business  
Examples:  
- Web services  
- API services  
- Request-facing business entry points  

### 2. Business with Obvious Peaks and Valleys  
Examples:  
- Busy during the day, idle at night  
- Traffic spikes during events  
- Periodic traffic fluctuations  

### 3. Adding Replicas Significantly Improves Overall Load Capacity  
That is:  
- Adding one replica can noticeably offload load  

### 4. Applications Naturally Suitable for Multi-Replica Deployment  
Typically many Deployment scenarios.  

### Current Summary  
> **When the main issue is "high concurrency pressure requiring more replicas toDiversion", prioritize HPA first.**  

---

## Eleven. When is VPA Most Suitable?  

Currently, we can summarize the most suitable scenarios for VPA as follows:  

### 1. Single Pod Resource Specifications Long-Term Misestimated  
Examples:  
- Requests written by guesswork every deployment  
- Too small leads to instability, too large causes waste  
- No long-term optimization basis  

### 2. Workloads Inconvenient to Solve by Simply Adding Replicas  
Examples:  
- Single instance is more critical  
- Multi-replica benefits are not obvious  
- Horizontal scaling is constrained  

### 3. Want to Continuously Optimize requests/limits  
Examples:  
- Reduce resource waste  
- Reduce frequent OOMs  
- Make resource allocation closer to reality  

### 4. More Concerned About Resource Profiles and Specification Governance Rather Than Instant Replica Changes  
In these scenarios, VPA is often more aligned with the essence than HPA.  

### Current Summary  
> **When the main issue is "single Pod resource specifications are inaccurate and need continuous calibration", prioritize VPA first.**  

---

## Twelve. Can HPA and VPA Be Used Together?  

This question is often asked, but currently, we should first establish a conservative and cautious understanding.  

### Conclusion First  
> **HPA and VPA cannot be simply and crudely used together.**  

Especially when:  
- Both are making automatic decisions around CPU/memory  
- No clear boundaries or governance strategies exist  

It's easy to cause mutual interference and conflicting behaviors.  

### Why Is This the Case?  
Because:  

#### HPA Focuses on Metric Changes  
Examples:  
- High CPU utilization means wanting to expand Pods  

#### VPA Focuses on Resource Specifications  
Examples:  
- Current requests are unreasonable, wanting to adjust up or down  

The issue is that HPA's calculations are related to requests.  
If VPA is simultaneously adjusting requests, and HPA uses these data for scaling decisions, it can lead to:  
- Metric baseline changes  
- Judgment logic fluctuations  
- Strategy mutual interference  

### Current Most Stable Understanding  
- **Do not directly equate HPA and VPA as "installing two automations makes it stronger"**  
- Should first clarify:  
  - Who is responsible for quantity elasticity  
  - Who is responsible for specification governance  
  - Whether there is metric coupling  

### Operational Understanding Focus  
In the early stages, treat them as:  
- Two different approaches  
- Not a default combination to be fully automated  

This is the most stable approach.  

---

## Thirteen. Why Many Teams Start with HPA and Then Carefully Evaluate VPA  

Because from implementation difficulty and scenario universality, HPA often sees business benefits more quickly.  

### Direct Benefits of HPA  
- Replica count automatically increases when traffic rises  
- Replica count automatically decreases when traffic drops  
- Very intuitive for many stateless services  

### Value of VPA is More Governance and Optimization Oriented  
- More reasonable resource configuration  
- Requests closer to real needs  
- Reduce long-term waste and guesswork configuration  

But it requires higher maturity in resource governance and needs careful understanding of update patterns, business impact, and control methods.  

### Operational Understanding Focus  
Thus, many teams' paths are more like:

1. First, implement HPA  
2. Then gradually standardize requests / limits  
3. Then cautiously introduce VPA for recommendations or vertical optimization  

This path is usually more stable.

---

## Fourteen. How HPA and VPA Issues Appear from a Troubleshooting Perspective

Understanding their relationship isn't just for selection, but also to avoid unnecessary detours during troubleshooting.

### Common HPA-Related Issues
- Why hasn't the replica count expanded?  
- Why isn't the HPA metric changing?  
- Why are new replicas Pending after HPA scales Pods?  
- Is there an issue with metrics-server?  
- Are requests written reasonably?

### Common VPA-Related Issues
- Why doesn't the cluster recognize `VerticalPodAutoscaler`?  
- Is the VPA component not installed at all?  
- Why isn't the recommendation coming out?  
- Why isn't `updateMode` taking effect as expected?  
- Why is the current suggested value so different from the expected value?

### Operational Understanding Focus
You can initially distinguish them as follows:

#### HPA Issues Are More Like
- Issues with metric-driven replica elasticity

#### VPA Issues Are More Like
- Resource recommendation and resource specification governance issues

---

## Fifteen. A Most Practical Judgment Method: Ask Yourself Three Questions

When you're hesitating between HPA and VPA in real scenarios, you can first ask yourself three questions.

### Question 1: Is the current issue about insufficient replica count or unreasonable single Pod resource allocation?
If it's clearly:
- Insufficient replica count  
- Inadequate concurrent load sharing  
More HPA-oriented.

If it's clearly:
- Inaccurate single Pod configuration  
- Long-term inaccurate requests  
More VPA-oriented.

---

### Question 2: Is this workload suitable for horizontal scaling to alleviate pressure?
If:
- Suitable for horizontal scaling  
- Adding replicas provides significant benefits  
More HPA-oriented.

If:
- Not convenient for horizontal scaling  
- More dependent on single instance specifications  
More VPA-oriented.

---

### Question 3: Do we currently need "quick elasticity" more, or "long-term resource governance"?
If we need more:
- Automatic scaling up during peaks and down during troughs  
More HPA-oriented.

If we need more:
- Resource specification optimization  
- Reduce waste  
- Reduce arbitrary resource allocation  
More VPA-oriented.

### Operational Understanding Focus
Often, the real basis for choice isn't "whether there's state," but the answers to these three questions.

---

## Sixteen. A More Stable Standard Answer for Interviews

If the interviewer asks:

**What's the difference between HPA and VPA, and what scenarios are they suitable for?**

You can answer like this:

HPA is horizontal scaling, primarily adjusting Pod replica counts based on changes in CPU, memory, or other metrics, making it more suitable for scenarios where horizontal scaling can alleviate pressure, many stateless web/API services are well-suited.  
VPA is vertical scaling, primarily providing or applying more suitable CPU/memory resource recommendations based on resource usage, making it more suitable for scenarios where single Pod resource specifications are hard to estimate long-term or where it's inconvenient to solve problems by simply adding replicas.  
They are not simple substitutes; one focuses more on quantity elasticity, and the other on specification governance.  
Additionally, VPA is more commonly discussed in stateful components, but that doesn't mean it's only suitable for stateful; HPA is also more common in stateless services, but that doesn't mean it can only be used for stateless.

### Keywords to Remember in This Answer
- HPA: Horizontal, adjust replica count  
- VPA: Vertical, adjust single Pod resource specifications  
- One manages quantity, one manages specifications  
- HPA leans toward stateless, horizontally scalable scenarios  
- VPA leans toward single-instance resource optimization scenarios  
- Not a simple substitute relationship

---

## Seventeen. The Most Important Conclusions from This Article

### 1. HPA and VPA Address Two Different Dimensions of Issues
- HPA: Quantity issues  
- VPA: Specification issues

### 2. HPA is More Suitable for Horizontally Scalable Scenarios
Many stateless traffic-based applications are naturally suited.

### 3. VPA is More Suitable for Single-Instance Resource Optimization Scenarios
Many stateful or inconvenient horizontally scalable scenarios are more common.

### 4. "HPA is suitable for stateless, VPA for stateful" direction isn't wrong, but too absolute
A more accurate statement is:
- HPA leans toward horizontally scalable  
- VPA leans toward single-instance resource optimization

### 5. They Are Not Mutually Exclusive
They focus on different aspects.

### 6. Don't Simply and Automatically Use Both Together
At least in the initial stages, clarify boundaries and dimensions of operation.

---

## Eighteen. Summary of the Stage

This article aims to establish not just the "naming difference" between HPA and VPA, but a more mature elasticity mindset:

### When There's a Problem with Business, Don't Rush to Ask
- Can we start more Pods?

Also don't just ask:
- Can we give Pods more CPU/memory?

Instead, first determine:
> **Is the current issue about "more instances" or "more suitable single-instance resource specifications"?**

If it's the former, it leans more toward HPA.  
If it's the latter, it leans more toward VPA.

So by now, you should be able to connect the entire resource management and elasticity scaling phase into a more complete understanding:
- requests/limits affect resource boundaries  
- requests affect scheduling  
- If it doesn't fit, it will be Pending  
- Resource contention has QoS and Eviction  
- HPA handles quantity elasticity  
- VPA handles specification governance  
- True resource governance isn't just about one object, but determining which category the issue belongs to

---

## Keyword Mnemonics

- HPA: Horizontal Pod Autoscaler, horizontal scaling  
- VPA: Vertical Pod Autoscaler, vertical scaling  
- Horizontal scaling: Adjust Pod replica counts  
- Vertical scaling: Adjust single Pod resource specifications  
- HPA: More about quantity elasticity  
- VPA: More about specification governance  
- Stateless: Usually more suitable for HPA  
- Stateful: Many scenarios more commonly discuss VPA  
- Not a binary choice  
- First determine if the issue is about quantity or specification

---

## Operational Extended Understanding

From an operational perspective, the relationship between HPA and VPA essentially represents two completely different approaches to resource governance.

HPA is more like answering:
> **When business peaks come, should I call more people to work together?**

VPA is more like answering:
> **What size of workspace and tools should these people have to be more reasonable?**

One leans toward:
- Traffic and concurrent load sharing

One leans toward:
- Resource configuration and specification calibration

True mature resource governance isn't just memorizing HPA and VPA definitions, but being able to judge based on the problem itself:
- Do we need more instances now?  
- Or more reasonable single-instance resource specifications?  
- Or something else, like scheduling, capacity, or architecture issues?

This is why in more realistic production scenarios, elasticity capabilities aren't just about "whether you can configure an autoscaler," but:
> **Can you first determine which layer the issue belongs to?**

---

## References
- Horizontal Pod Autoscaler  
  https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

- Vertical Pod Autoscaler  
  https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler

- Kubernetes Autoscaling Concepts  
  https://kubernetes.io/docs/concepts/workloads/autoscaling/

---

## Next Day Suggestions
Next article suggestion:  
[[10-Resource Management and Elastic Scaling Summary - From Requests, Pending to HPA, VPA, and Capacity Governance]]