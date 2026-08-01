# 05 - Topology Spread Constraints: topologySpreadConstraints Introduction

## Document Notes
- Document Location: Kubernetes Scheduling Strategy Advanced Basics
- Applicable Stage: 07 - Application Deployment / 09 - Scheduling Strategy and Workload Placement
- Learning Objectives:
  - Understand what `topologySpreadConstraints` solves
  - Understand the difference between `topologySpreadConstraints` and `podAntiAffinity`
  - Master the most basic topology spread YAML syntax
  - Be able to understand `maxSkew`, `topologyKey`, `whenUnsatisfiable`, `labelSelector`
  - Understand why multi-replica applications need "more balanced distribution"

## First, Build an Intuition

In the previous article, we learned `podAntiAffinity`.  
It solves:

**Don't let certain Pods cluster together.**

For example:

- Multiple replicas of the same Deployment shouldn't all land on the same node
- Certain critical Pods shouldn't be too close to similar Pods
- Hope replicas are distributed as evenly as possible to avoid single-node failure taking down all

But moving further, you'll find that simply "not clustering" is insufficient.

Because in real production environments, we often pursue not just simple "separation" but:

**Distribute as evenly as possible.**

For example:

- 6 replicas, hope 3 nodes are split as 2, 2, 2
- 4 replicas, hope 2 availability zones are split as 2, 2
- Not just "not clustering", but "as balanced as possible"

This is what `topologySpreadConstraints` solves.

## What is Topology Spread

Topology spread can be understood as:

**Distribute a group of Pods as evenly as possible across a topology dimension.**

The "topology dimension" could be:

- Nodes
- Availability zones
- Racks
- Other topology domains you define

So the core idea of this capability is:

**Not just controlling whether Pods can be close, but controlling whether their distribution across topology domains is balanced.**

## Why Need topologySpreadConstraints

Because many businesses can't achieve ideal distribution just with `podAntiAffinity`.

For example, you have 6 replicas and 3 nodes.

If only using `podAntiAffinity preferred`, the scheduler might achieve:

- node1: 3 Pods
- node2: 2 Pods
- node3: 1 Pod

From the perspective of "not clustering", it seems acceptable.  
But from the perspective of high availability and load balancing, this distribution is not ideal.

A more ideal distribution would be:

- node1: 2
- node2: 2
- node3: 2

At this point, you need:

**A mechanism to explicitly express the goal of "even distribution".**

This is what `topologySpreadConstraints` provides.

## Core Problem Solved by topologySpreadConstraints

You can think of it as answering this question:

**What's the maximum allowed difference in the number of Pods across different topology domains for the same type of Pod?**

In other words, it focuses on:

- Not whether individual Pods are close to each other
- But whether the overall distribution of a group of Pods is imbalanced

This is different from the anti-affinity approach.

## Difference with Pod Anti-Affinity

This is one of the most critical comparisons in this article.

### Pod Anti-Affinity

Emphasizes:

**Don't cluster with certain types of Pods.**

Its focus is on "avoiding".

### topologySpreadConstraints

Emphasizes:

**Distribute as evenly as possible across multiple topology domains.**

Its focus is on "balance".

So you can remember this first:

- `podAntiAffinity`: Prevent clustering
- `topologySpreadConstraints`: Achieve balance

## What is a Topology Domain

A "topology domain" can be understood as:

**The unit you use for distribution statistics.**

Common examples include:

- Nodes: `kubernetes.io/hostname`
- Availability zones: `topology.kubernetes.io/zone`

If `topologyKey` is:

    kubernetes.io/hostname

That means distributing by "nodes".

If `topologyKey` is:

    topology.kubernetes.io/zone

That means distributing by "availability zones".

## What is topologyKey

`topologyKey` is used to specify:

**Which dimension Pods should be spread and counted by.**

For example:

### 1. Spread by Nodes

    topologyKey: kubernetes.io/hostname

Means each host is considered one topology domain.

### 2. Spread by Availability Zones

    topologyKey: topology.kubernetes.io/zone

Means each availability zone is considered one topology domain.

So `topologyKey` determines:

**What is the "x-axis" for spread statistics.**

## What is labelSelector

`topologySpreadConstraints` doesn't count all Pods, but only a specific type of target Pods.

These Pods are selected by `labelSelector`.

For example:

    labelSelector:
      matchLabels:
        app: nginx-demo

Means:

**Only count the distribution of Pods labeled with `app=nginx-demo`.**

So topology spread typically focuses on:

**Whether the distribution of the same application, same group of replicas, or same type of business Pods is balanced.**

## What is maxSkew

This is one of the most core fields in this article.

`maxSkew` indicates:

**The maximum allowed difference in the number of Pods across different topology domains.**

For example:

    maxSkew: 1

Means the number of Pods in any two topology domains can't differ too much, maximum only allowed to differ by 1.

### Example Understanding

Assume 3 nodes, 6 replicas.

Ideal distribution:

- node1: 2
- node2: 2
- node3: 2

This is the most balanced, difference of 0.

Also acceptable:

- node1: 2
- node2: 2
- node3: 1

Maximum difference of 1.

But if it becomes:

- node1: 3
- node2: 1
- node3: 1

Maximum difference of 2.

If you set:

    maxSkew: 1

This distribution would not satisfy the constraint.

## What is whenUnsatisfiable

This field determines:

**What to do if the spread requirements cannot be met.**

The two most common values are:

- `DoNotSchedule`
- `ScheduleAnyway`

### 1. DoNotSchedule

Means:

**If the spread constraint cannot be met, don't schedule.**

This is a hard constraint.

### 2. ScheduleAnyway

Means:

**Even if the spread requirements cannot be perfectly met, try to schedule as much as possible.**

This is a soft constraint, a soft preference.

This approach is similar to what we learned earlier with required / preferred:

- `DoNotSchedule` is stricter
- `ScheduleAnyway` is more lenient

## Most Basic topologySpreadConstraints Example

Below is the most basic Deployment, making its 3 replicas spread as evenly as possible across nodes.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-spread
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-spread
  template:
    metadata:
      labels:
        app: nginx-spread
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: nginx-spread
      containers:
        - name: nginx
          image: nginx:1.25
```

This YAML means:

- Counting object: `app=nginx-spread` Pods
- Counting dimension: by node
- Hope these Pods are as evenly distributed across different nodes as possible
- Maximum imbalance difference does not exceed 1
- If it cannot be achieved, still attempt to schedule

## Understanding This Example

Assume the cluster has 3 nodes and replica count is also 3.  
The scheduler will preferentially schedule:

- node1: 1
- node2: 1
- node3: 1

If there are 4 replicas, it may preferentially schedule:

- node1: 2
- node2: 1
- node3: 1

Because the maximum difference remains 1.

This is the meaning of "as evenly distributed as possible".

## DoNotSchedule Example: Enforce Node-Level Balanced Distribution

If you change `whenUnsatisfiable` to:

    whenUnsatisfiable: DoNotSchedule

Example:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-spread-hard
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx-spread-hard
      template:
        metadata:
          labels:
            app: nginx-spread-hard
        spec:
          topologySpreadConstraints:
            - maxSkew: 1
              topologyKey: kubernetes.io/hostname
              whenUnsatisfiable: DoNotSchedule
              labelSelector:
                matchLabels:
                  app: nginx-spread-hard
          containers:
            - name: nginx
              image: nginx:1.25

This indicates:

- Must strive to maintain balance
- If scheduling would cause distribution imbalance exceeding `maxSkew`
- This Pod cannot be scheduled

This approach is stricter, but also more likely to result in Pod Pending when resources are insufficient or node count is inadequate.

## When to Use ScheduleAnyway

This is more commonly used in the following scenarios:

- General business applications
- Want to strive for balance but don't want to block scheduling
- Cluster resources may be unstable
- Prioritize "being able to run" over perfect distribution

In other words:

**First ensure business can run, then strive for better distribution.**

## When to Use DoNotSchedule

This is more commonly used in the following scenarios:

- Strict high availability requirements
- Want to enforce replica distribution control
- Cannot accept excessive skew
- Very sensitive to single fault domain risks

But note:

**The stricter the rules, the more likely scheduling will fail.**

## Example of Spreading Across Availability Zones

If your cluster is multi-zone deployed, you can spread by zone.

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-zone-spread
    spec:
      replicas: 4
      selector:
        matchLabels:
          app: nginx-zone-spread
      template:
        metadata:
          labels:
            app: nginx-zone-spread
        spec:
          topologySpreadConstraints:
            - maxSkew: 1
              topologyKey: topology.kubernetes.io/zone
              whenUnsatisfiable: ScheduleAnyway
              labelSelector:
                matchLabels:
                  app: nginx-zone-spread
          containers:
            - name: nginx
              image: nginx:1.25

This indicates:

- 4 replicas strive to be evenly distributed across availability zones
- If there are 2 zones, ideal distribution would approach 2, 2
- If there are 3 zones, may approach 2, 1, 1

## Why It's More Suitable for "Balanced Distribution" Than Anti-Affinity

Because `podAntiAffinity` is more like a restriction to "avoid placingSimilar Pods in the same domain".

But `topologySpreadConstraints` will explicitly consider:

**How manySimilar Pods are already in each topology domain.**

Thus, it is more direct and clear in achieving "numerical balance".

For example:

- Anti-affinity emphasizes: Don't crowd together
- Topology spread emphasizes: Distribute as evenly as possible

These two have related goals but are not the same thing.

## A More Production-Ready Understanding

You can think of `podAntiAffinity` as:

**Don't put all eggs in one basket.**

And think of `topologySpreadConstraints` as:

**Not only can't put all eggs in one basket, but also strive to distribute them as evenly as possible across multiple baskets.**

This analogy is very suitable for memorization.

## Typical Production Scenarios

### 1. Multi-Replica Web Service Distribution

Multiple frontend services want to be distributed across different nodes and zones to improve availability and load balancing.

### 2. Middleware Replica Distribution

For example, some service replicas want to avoid being concentrated in a single fault domain.

### 3. Multi-AZ Disaster Recovery

Business replicas want to be relatively evenly distributed across different AZs to avoid most instances being affected by a single AZ failure.

### 4. Reduce Node Hotspots

Avoid having a single node carry too many similar Pods, causing excessive resource pressure.

## labelSelector Must Match the Target Pod Exactly

This is a common pitfall in practice.

For example, if you define:

    labelSelector:
      matchLabels:
        app: nginx-demo

The scheduler will only count `app=nginx-demo` Pods.  
If the Pod template's actual labels are not this, the desired effect will not be achieved.

Therefore, Deployments typically maintain consistency among:

- `.spec.selector.matchLabels`
- `.spec.template.metadata.labels`
- `topologySpreadConstraints.labelSelector`

These three sections should have consistent semantics.

## A Common Deployment Writing Pattern

The following is a common basic distribution template in production:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web-app
    spec:
      replicas: 4
      selector:
        matchLabels:
          app: web-app
      template:
        metadata:
          labels:
            app: web-app
        spec:
          topologySpreadConstraints:
            - maxSkew: 1
              topologyKey: kubernetes.io/hostname
              whenUnsatisfiable: ScheduleAnyway
              labelSelector:
                matchLabels:
                  app: web-app
          containers:
            - name: nginx
              image: nginx:1.25

This example is very suitable as an introductory template.

## Relationship with nodeAffinity, taint/toleration

At this point, you should start viewing the scheduling controls as interconnected rules.

### nodeAffinity
Solves:

**Where do I prefer to go?**

### taint / toleration
Solves:

**Which nodes are willing to accept me?**

### podAntiAffinity
Solves:

**I don't want to be too close to certain pods.**

### topologySpreadConstraints
Solves:

**Our group of Pods should be as evenly distributed as possible across different topology domains.**

Therefore, the scheduler considers not a single condition but the combined result of multiple rules.

## Common Issue: Why Pods Are Still Not Evenly Distributed

This is completely normal.

Possible reasons include:

### 1. Insufficient Node Count

For example, only 2 nodes exist, yet you want to evenly distribute 5 Pods, which is inherently impossible.

### 2. Resource Insufficiency

Some nodes may be topologically suitable, but lack sufficient resources, preventing the scheduler from placing Pods there.

### 3. Overlapping Scheduling Rules

For example:

- nodeAffinity
- taint / toleration
- podAntiAffinity

These combined conditions reduce the number of available nodes.

### 4. whenUnsatisfiable is ScheduleAnyway

It only means "try to schedule", not "guarantee".

### 5. labelSelector is Incorrectly Defined

Causing the scheduler to count Pods that are not the intended group.

## Troubleshooting Steps

If you think Pods are not distributed reasonably, suggest checking in this order.

### Step 1: Check Pod Distribution

    kubectl get pods -o wide

First, see which nodes these Pods are actually on.

### Step 2: Review YAML

Focus on confirming:

- `topologyKey`
- `maxSkew`
- `whenUnsatisfiable`
- `labelSelector`

### Step 3: Confirm Nodes Have Corresponding Topology Labels

For example, when distributing by zone, nodes must have:

    topology.kubernetes.io/zone

Otherwise, scheduling behavior may differ from expectations.

Check:

    kubectl describe node <node-name>

### Step 4: Check for Overlapping Scheduling Rules

For example, are any of these also defined:

- nodeSelector
- nodeAffinity
- taint / toleration
- podAntiAffinity

### Step 5: Review describe Events

    kubectl describe pod <pod-name>

If strict constraints caused scheduling failure, clues often appear in the Events.

## A Minimal Experiment Suggestion

### Experiment Goal

Observe the even distribution effect of 4 replicas across multiple nodes.

### Example YAML /think

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-topology-test
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx-topology-test
  template:
    metadata:
      labels:
        app: nginx-topology-test
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: nginx-topology-test
      containers:
        - name: nginx
          image: nginx:1.25
```

**Apply:**

```bash
kubectl apply -f nginx-topology-test.yaml
```

**Check:**

```bash
kubectl get pods -o wide
```

You can observe whether these replicas are distributed as evenly as possible across different nodes.

If your cluster has 3 or more nodes, this experiment will be more intuitive.

## Key Takeaways Recap

You need to remember these core points:

1. `topologySpreadConstraints` is used to evenly distribute a group of Pods across topology domains  
2. It focuses on "overall distribution balance", not just "not crowding together"  
3. `topologyKey` determines the dimension for spreading, such as nodes or availability zones  
4. `labelSelector` determines which type of Pod to count  
5. `maxSkew` represents the maximum allowed difference in quantity between different topology domains  
6. `whenUnsatisfiable: DoNotSchedule` is stricter  
7. `whenUnsatisfiable: ScheduleAnyway` is more lenient  
8. It shares similar goals with `podAntiAffinity`, but emphasizes "balance" rather than just "avoiding clustering"

## Common Command Quick Reference

```bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl get nodes --show-labels
kubectl describe node <node-name>
kubectl apply -f deployment.yaml
```

## One-Sentence Summary

`topologySpreadConstraints` essentially tells the scheduler: **Don't just spread Pods apart, but strive to distribute this group of Pods more evenly across topology domains like nodes and availability zones.**

## Tags
#Kubernetes #ApplyDeployment #SchedulePolicy #topologySpreadConstraints #TakutoSpread. #HighAvailable #CopyBalance

## Operations Extension Understanding
- Kubernetes official documentation: Pod Topology Spread Constraints  
  https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/
- Kubernetes official documentation: Assigning Pods to Nodes  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
- Kubernetes official documentation: Well-Known Labels, Annotations and Taints  
  https://kubernetes.io/docs/reference/labels-annotations-taints/

## Next Day Plan
- Study [[06-Scheduling Strategy Phase Summary - From Resource Request to Workload Placement]]
- Connect `nodeSelector`, `nodeAffinity`, `podAffinity/AntiAffinity`, `taint/toleration`, `topologySpreadConstraints` into a complete scheduling view
- Build a systematic troubleshooting framework for "why Pods are Pending"