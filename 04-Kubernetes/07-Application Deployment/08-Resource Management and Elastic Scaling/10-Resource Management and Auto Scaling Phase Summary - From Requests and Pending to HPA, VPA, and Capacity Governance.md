# 10-Resource Management and Auto Scaling Phase Summary: From Requests and Pending to HPA, VPA, and Capacity Governance

## Document Description
- Document Purpose: A summary of Kubernetes resource management and auto scaling phases
- Target Audience: Those who have completed foundational studies on requests/limits, resource scheduling, Pending, QoS, eviction, HPA, and VPA, for use in periodic review
- Recommended Path: `04-Kubernetes/07-Application Deployment/08-Resource Management and Auto Scaling/10-Resource Management and Auto Scaling Phase Summary: From Requests and Pending to HPA, VPA, and Capacity Governance.md`

## Tags
#Kubernetes #ResourceManagement #AutoScaling #requests #limits #Pending #QoS #Eviction #HPA #VPA #CapacityGovernance #ApplicationDeployment #CloudNative #PhaseSummary #Ops #InterviewNotes

---

## I. What Is Learned in This Phase

At first glance, this set of topics seems disparate:

- requests/limits
- Pod scheduling
- Pending
- QoS
- Eviction
- HPA
- VPA
- The relationship between HPA and VPA

However, from an operational perspective, they all address the same core question:

> **In Kubernetes, how do resources affect whether an application can be deployed, run stably, avoid resource exhaustion, scale effectively, and maintain its functionality?**

In other words, this phase is not about learning individual concepts but about establishing a coherent framework:

### 1. Applications declare their resource requirements
This includes:

- requests
- limits

### 2. The scheduler determines whether the application can be placed on a node
If it cannot, the Pod will enter:

- Pending state

### 3. Even after deployment, Pods are not treated equally in terms of resource allocation
This involves:

- QoS policies

### 4. When node resources become insufficient, not all Pods can survive
This leads to:

- Eviction mechanisms

### 5. Changes in business load require further auto-scaling adjustments
This includes:

- HPA and VPA

### 6. However, auto-scaling is not always effective
Other factors such as:

- Scheduling algorithms
- Resource capacity
- Individual Pod specifications
- Node resource limits

also play a role.

Therefore, the essence of this phase is to understand that:

> **Resource management is not about filling out forms but about creating a cohesive framework that spans scheduling, operation, eviction, scaling, and capacity governance.**

---

## II. How to Remember the Main Thread of This Phase

If we summarize all these concepts into one main thread, it can be remembered as:

> **Resource Declaration → Scheduling Decision → Pending/Running → QoS Hierarchical Protection → Node Pressure → Eviction → HPA for Horizontal Scaling → VPA for Vertical Resource Management → Ensuring Capacity Adequacy**

In other words, although each topic focuses on a different aspect, they all follow a chronological and phased approach to solving similar problems.

---

### Phase 1: Before the Pod Actually Starts Running
This phase addresses:

- What requests and limits are.
- What the scheduler considers first.
- Why Pods enter the Pending state.

In other words:

> **Whether the Pod can even be deployed on a node.**

---

### Phase 2: After the Pod Has Been Deployed and Is Running
This phase focuses on:

- What QoS policies are and how they affect resource allocation.
- How node resource pressure impacts different Pods.
- Why Eviction may occur.

In other words:

> **Whether the Pod can maintain stability under resource competition.**

---

### Phase 3: When Business Load Changes
This phase explores:

- How HPA automatically adjusts the number of replicas.
- Why scaling with HPA does not always guarantee improved performance.
- How VPA manages individual Pod resource specifications.
- The differences between HPA and VPA in addressing scaling challenges.

In other words:

> **How to decide whether to increase replica counts or adjust individual Pod specifications when business needs change.**

---

## IV. Why the Scheduler Starts with Requests

This is one of the most fundamental concepts in this phase.

The scheduler’s primary goal is to determine:

> **Whether a Pod can be deployed on a node.**

To achieve this, it needs a resource requirement value that:

- Can be declared beforehand.
- Is calculable.
- Can be compared.

The most critical values here are:

- `requests.cpu`
- `requests.memory`

### Why Not Start with Limits?
Limits are more about:

- The maximum amount of resources allowed during runtime.
- They represent operational boundaries.
- They are not the primary basis for scheduling decisions.

### Operational Insight
This is why you often see situations where a node appears to have sufficient resources, but a Pod still remains Pending. The scheduler considers factors such as:

- `allocatable` resources.
- Declared resource requests.
- Remaining available space.

Therefore, it## Nine, Why Expanding Pods with HPA May Not Solve the Problem

This is one of the crucial practical points to understand at this stage.

Many people, after learning about HPA, naturally form the impression that:

- When metrics increase,
- HPA expands the number of pods,
- And more pods are created,
- The problem will be solved.

But in reality, it's not always the case.

### The typical process is as follows:

### 1. HPA detects high metrics.
### 2. HPA increases the number of replicas in the Deployment.
### 3. New pods are created.
### 4. The scheduler attempts to assign nodes to these new pods.
### 5. If there aren't enough node resources,
### 6. The new pods enter a Pending state.
### 7. As a result, even though the number of pods has increased, business performance may not improve significantly.

### Key Points for Ops Professionals to Understand

What this understanding helps establish is an important reality:

> **HPA is only responsible for expanding the number of pods; it does not create additional node capacity.**

Therefore, true horizontal scaling depends not just on the existence of HPA objects but also on whether:

- New replicas are actually created,
- They are successfully scheduled,
- They become Ready,
- And they can handle the traffic.

This is why, in the end, issues related to HPA often boil down to:

- Requests,
- Scheduling,
- Node capacity,
- and Pending status.

These are all concepts that were covered earlier.

---

## Ten, How VPA Advances Resource Management Beyond Simply Adjusting the Number of Replicas

With HPA in place, many people assume that automatic scaling is simply about increasing the number of replicas. However, in real production scenarios, there are other issues that cannot be resolved by just adding more pods.

For example:

- Requests may often need to be estimated based on experience.
- Some pods may be configured too small and frequently face resource shortages.
- Other pods may be configured too large, resulting in unnecessary waste of resources.
- For such loads, increasing the number of pods is not always the best solution.

What needs to be addressed in these cases is not whether more pods are needed but rather:

- **What the optimal resource specifications for a single pod should be.**

This leads us to VPA:

> **Vertical Pod Autoscaler**

### Core Capabilities of VPA
- Based on resource usage, it provides or recommends more reasonable CPU/Memory resource settings for applications.

### What It Solves:
> **Resource specification issues**

In other words:

- Whether the resource configuration for a single pod is appropriate.
- Whether the requests/limits deviate significantly from actual usage patterns.

### Key Points for Ops Professionals to Understand

If HPA addresses whether there are enough replicas, then VPA focuses on whether the resource specifications for individual pods are reasonable.

---

## Eleven, Why It's Important to Emphasize “Installation Prerequisites” and “Custom Resources” for VPA

This is a significant difference between VPA and HPA.

In everyday learning contexts, HPA is often perceived as part of the existing cluster capabilities. However, VPA cannot be understood in this way.

### The most crucial conclusion at this stage is:

> **VPA is not an object that can be used directly out of the box.**

Its functionality usually relies on:

- Additional components that need to be installed.
- CRDs.
- And mechanisms such as recommender/updater/admission.

The most important point here is that:

> `VerticalPodAutoscaler` is a custom resource introduced into the cluster through CRDs.

In other words, before VPA can be used effectively, you first need to:

- Define the `CustomResourceDefinition`.
- Then the cluster will recognize `VerticalPodAutoscaler`.
- Only after adding the necessary controller components will it become truly functional.

### Key Points for Ops Professionals to Understand

Installing VPA and creating actual business-related VPA objects are two different processes.

---

## Twelve, How to Properly Understand the Relationship Between HPA and VPA

This is a crucial point that must be clarified at this stage.

Many people ask:

- Do HPA and VPA replace each other?
- Is one designed for stateless applications and the other for stateful ones?
- Can they simply be used together to solve all problems?

The most accurate understanding at this stage is:

### HPA focuses on:
- Quantitative scaling.
- Whether adding more pods can help distribute traffic.

### VPA focuses on:
- Specification governance.
- Whether the resource specifications for individual pods are appropriate.

Therefore, they do not replace each other but rather address different aspects of resource management.

---

## Thirteen, Why the Statement “One Is Suitable for Stateless Applications, and the Other for Stateful Ones” Is Inaccurate

You might often hear this statement during learning or interviews:

- HPA is suitable for stateless applications.
- VPA is### 7. True resource governance capability does not lie in knowing how to handle several objects, but in identifying which layer a problem belongs to. This is the most core ability at this stage.

---

## Nineteen, Phase Summary

Although this set of content covers many topics, it essentially revolves around a very practical issue:

> **Applications in Kubernetes should not only "start running," but also be able to be deployed smoothly, run stably, ensure resource control, adapt to business changes, and truly be useful once deployed.**

In other words, what is truly accomplished at this stage is not just an introduction to resource fields, but a complete transformation in thinking:

From:

- Only knowing how to create Deployments/Services
- Only checking whether Pods have started running

To:

- Understanding requests and limits
- Distinguishing between Pending states and runtime failures
- Comprehending QoS and Eviction mechanisms
- Realizing why HPA is not always the perfect solution for elasticity
- Understanding why VPA represents a capability for managing resource specifications
- Knowing whether to address problems from the perspective of quantity or specification

If we were to summarize the value of this set of content in one sentence, it would be:

> **This stage helps you move from simply "deploying applications" to truly understanding how resources determine an application's availability, stability, and elasticity.**

---

## Key Terms Summary

- requests: Scheduling minimum resource requirements
- limits: Maximum resource limits during runtime
- Pending: A common scheduling phase issue
- QoS: Resource quality assurance level
- Eviction: Active eviction of Pods under node pressure
- HPA: Horizontal scaling, adjusting the number of replicas
- VPA: Vertical scaling, adjusting individual Pod resource specifications
- Quantity issues: Consider HPA first
- Specification issues: Consider VPA first
- Capacity governance: Focus on the entire resource chain, not just individual objects

---

## Operations and Maintenance Insights

From an operations and maintenance perspective, what is truly established at this stage is not just knowledge of resource fields, but a way of thinking more akin to that of a production environment.

In production, many issues may have similar surface symptoms, but their root causes lie at different levels:

- If a Pod does not start running, it might be because it is in the Pending state, not because the application is broken.
- If a Pod is terminated, it might be due to Eviction, not because the program crashed.
- Even if HPA is enabled, new Pods may still remain in the Pending state, indicating that elasticity has not taken effect.
- If resources are constantly misallocated, the real issue might not be needing more replicas, but rather a capability like VPA for managing resource specifications.

Therefore, a mature approach to resource governance is not about simply modifying YAML files when problems arise, but about first identifying whether the issue lies at the scheduling layer, in resource competition during runtime, under node pressure, in quantity elasticity, or in individual instance specifications.

This is also why, although this stage focuses on resources, it is essentially training an important ability:

> **To understand how resources affect business availability at each level of the application's lifecycle.**

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

## Next Steps
Suggested next topic: [[01-nodeSelector Basics: Scheduling Pods to Specific Nodes]]