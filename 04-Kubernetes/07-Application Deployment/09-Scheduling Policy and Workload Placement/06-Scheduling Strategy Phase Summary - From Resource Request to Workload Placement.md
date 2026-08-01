# 06-Scheduling Strategy Phase Summary: From Resource Requests to Workload Placement

## Document Notes
- Document Positioning: Kubernetes Scheduling Strategy Phase Summary
- Applicable Phase: 07-Application Deployment / 09-Scheduling Strategy and Workload Placement
- Learning Objectives:
  - Connect resource requests, node selection, affinity, taint tolerance, and topology spread into a complete understanding
  - Understand the key judgments Pod undergoes from creation to successful scheduling
  - Establish a systematic troubleshooting approach for "Pod Pending" issues
  - Understand why workloads are placed on specific nodes from a scheduling rule perspective
  - Lay the foundation for subsequent learning on deployment, rollback, access control, and Helm

## What This Phase Is Actually About

At first glance, Phase 09 seems to cover many scattered terms:

- `nodeSelector`
- `nodeAffinity`
- `podAffinity`
- `podAntiAffinity`
- `taints`
- `tolerations`
- `topologySpreadConstraints`

But from a higher level, they all address the same core issue:

**Where should a Pod be placed, and why it ends up there.**

In other words, this phase's core isn't memorizing fields, but understanding:

**How the Kubernetes scheduler filters nodes, scores them, and decides the final placement of a Pod.**

## What Is the Essence of Scheduling

The essence of scheduling can be summarized as:

**Find a list of candidate nodes that can run this Pod from all cluster nodes, then select the most suitable one.**

So scheduling isn't random, nor is it based on a single condition—it's a process of **filtering + comparison + selection**.

You can think of it as a multi-round interview:

1. First check if the node qualifies
2. Then verify if it meets hard requirements
3. Finally pick the best candidate from the qualified ones

## Understanding Pod Scheduling from Creation to Success

At the beginner level, you can roughly understand the Pod scheduling process as this main line:

1. Pod is created
2. Scheduler starts looking for suitable nodes
3. Filter out nodes that clearly don't meet requirements
4. Score remaining nodes by priority
5. Select the node with higher scores
6. Pod is bound to the node
7. kubelet actually pulls the container on the target node

This article focuses on steps 3 and 4:

- Which conditions will filter out nodes
- Which conditions will affect node priority

## The Most Fundamental Layer of Scheduling: Resource Availability

Before discussing various scheduling strategies, you must remember one thing:

**Without resources, everything is off the table.**

If a Pod defines resource requests, such as:

- CPU requests
- Memory requests

The scheduler first checks if the node has enough available resources.

For example:

- Pod needs 2 CPUs
- A node only has 1 CPU available

This node is immediately eliminated.

So the first layer of scheduling is very fundamental:

**Does this node have the capacity to host this Pod.**

This is why the previous phase "How resource requests affect Pod scheduling" is a prerequisite for scheduling strategies.

## The First Major Category of Scheduling Issues: Where Does the Pod Want to Go

These capabilities mainly include:

- `nodeSelector`
- `nodeAffinity`

They solve:

**What requirements does the Pod have for nodes.**

### nodeSelector

The simplest node selection method.

Suitable for expressing:

- I want to go to `disktype=ssd`
- I want to go to `env=prod`

Features:

- Simple
- Exact match
- Can only express hard conditions

### nodeAffinity

A more flexible node selection method.

Suitable for expressing:

- Must go to `env=prod`
- Prefer to go to `disktype=ssd`
- Avoid going to `env=test`
- As long as a certain label exists

Features:

- Supports required
- Supports preferred
- Supports expressions
- More suitable for production environments

You can summarize these two as:

**They both express the Pod's "preference or hard requirements" for nodes.**

## The Second Major Category of Scheduling Issues: Pod's Proximity and Avoidance

These capabilities mainly include:

- `podAffinity`
- `podAntiAffinity`

They solve:

**What spatial relationship should exist between the Pod and other Pods.**

### podAffinity

Indicates:

**Desire or requirement to be close to certain Pods.**

Common scenarios:

- A business Pod wants to be close to a cache Pod
- Low-latency collaboration within the same topology domain

### podAntiAffinity

Indicates:

**Desire or requirement to avoid certain Pods.**

Common scenarios:

- Multiple replicas of the same application shouldn't be on the same node
- Avoid single points of failure being taken down together
- Reduce resource contention

Here's a key distinction to grasp:

- Node affinity: checks node labels
- Pod affinity/anti-affinity: checks other Pods' labels and positions

## The Third Major Category of Scheduling Issues: Node's Willingness to Accept the Pod

These capabilities mainly include:

- `taints`
- `tolerations`

They solve:

**Node's admission control for the Pod.**

The previous `nodeSelector` / `nodeAffinity` are more like:

**Pod actively selecting nodes.**

While taint / toleration is more like:

**Nodes actively filtering Pods.**

### taint

Applied to nodes, indicating:

- I don't welcome regular Pods
- I'm a special node
- No permission, don't enter

### toleration

Written on the Pod, indicating:

- I can tolerate this taint
- This taint won't block me

The most important point about this capability pair is:

**Toleration only means "allowing entry", not guaranteeing placement here.**

In production, it's often used together with:

- `toleration`
- `nodeSelector` or `nodeAffinity`

## The Fourth Major Category of Scheduling Issues: How This Group of Pods Should Be Distributed

This capability mainly focuses on:

- `topologySpreadConstraints`

It solves:

**How a group of Pods should be more evenly distributed across multiple topology domains.**

This is similar to `podAntiAffinity` but with different focus.

### podAntiAffinity

Tends to:

**Avoid crowding together.**

### topologySpreadConstraints

Tends to:

**Distribute as evenly as possible.**

For example:

- Anti-affinity just says "don't all be on the same node"
- Topology spread goes further, aiming for 2, 2, 2 instead of 3, 2, 1

This is more valuable for high availability and load balancing.

## You Can Divide the Entire Scheduling System into Four Questions

At this point, you can summarize this phase's content into four major questions.

### 1. Does this node have enough resources?
For example:
- Is CPU sufficient?
- Is memory sufficient?

### 2. Does this Pod want to go to this node?
For example:
- `nodeSelector`
- `nodeAffinity`

### 3. Does this node want to accept this Pod?
For example:
- `taints`
- `tolerations`

### 4. How should this Pod be placed relative to other Pods?
For example:

- `podAffinity`
- `podAntiAffinity`
- `topologySpreadConstraints`

These four questions basically tie together the scheduling core you've learned so far.

## A Comprehensive Understanding Model of Scheduling

You can imagine the scheduler asking itself these questions:

### Step 1: Is there enough resource?
- Can this node fit the CPU and memory?

### Step 2: Is the Pod allowed to come?
- Does `nodeSelector` match?
- Does `nodeAffinity required` meet?

### Step 3: Is the node allowing you to come?
- Does the node have taint?
- Does the Pod have corresponding toleration?

### Step 4: Are the relationships with other Pods satisfied?
- Do you need to be close to certain Pods?
- Do you need to avoid certain Pods?
- Does it meet the topology spread requirements?

### Step 5: If multiple nodes can be selected, which is better?
- preferred nodeAffinity
- preferred podAffinity / antiAffinity
- topology spread scoring preference
- Other comprehensive scheduling scores

Finally, the scheduler picks a more suitable node from the "available nodes".

## Why Would a Pod Be Pending?

This is the most practical question in this phase.

**Pod Pending does not equal only one reason.**

From a scheduling perspective, the most common reasons can be summarized as follows.

### 1. Resource Insufficiency

Examples:
- CPU is insufficient
- Memory is insufficient
- The node's allocatable resources are insufficient

This is the most common and fundamental reason.

### 2. nodeSelector Does Not Match

Examples:
- YAML specifies `disktype=ssd`
- But there are no nodes with this label in the cluster

### 3. nodeAffinity Required Conditions Are Too Strict

Examples:
- Must be `env=prod`
- Must be `disktype=ssd`
- Must be `zone=az1`

Resulting in no nodes meeting all conditions.

### 4. Taint Without Corresponding Toleration

Examples:
- The node has `NoSchedule`
- The Pod does not have the corresponding toleration

This node is directly unavailable.

### 5. podAntiAffinity / podAffinity Conditions Not Met

Examples:
- Must be with certain Pods on the same node
- But those Pods simply don't exist

Or:
- Must be dispersed to different nodes with same type Pods
- But there are not enough nodes

### 6. topologySpreadConstraints Are Too Strict

Examples:
- `maxSkew` is too strict
- `whenUnsatisfiable: DoNotSchedule`
- Not enough nodes to maintain balance

### 7. No Available Nodes After Multiple Conditions Combined

This is a very common scenario in production.

Each individual condition may seem fine, but combined they result in no nodes passing all filters.

Example:
- Requires `env=prod`
- Requires `disktype=ssd`
- Nodes cannot have certain taint, or Pods must tolerate certain taint
- Also needs anti-affinity dispersion
- Also needs to meet topology spread requirements

Eventually, you'll get "rules are all reasonable, but no node can meet all conditions".

## Pod Pending Troubleshooting Order

This section is very important and you'll use it repeatedly later.

### Step 1: Check Pod Status

    kubectl get pod

Confirm if it's `Pending`.

### Step 2: Check Pod Details

    kubectl describe pod <pod-name>

Focus on:
- `Events`
- Scheduling failure messages
- Whether `didn't match` appears
- Whether `had taint` appears
- Whether `Insufficient cpu` / `Insufficient memory` appears

This is usually the entry point for troubleshooting.

### Step 3: Check Node Resource Status

    kubectl describe node <node-name>

Focus on:
- Allocatable
- Allocated resources
- Remaining capacity

### Step 4: Check Node Labels

    kubectl get nodes --show-labels

Confirm:
- `nodeSelector` matches
- `nodeAffinity` actually exists on nodes

### Step 5: Check Node Taints

    kubectl describe node <node-name>

Confirm:
- Whether taint exists
- Whether the Pod has corresponding toleration

### Step 6: Check Pod's Scheduling Rules

Focus on reviewing YAML for:
- `nodeSelector`
- `affinity`
- `tolerations`
- `topologySpreadConstraints`

### Step 7: Check Cluster-wide Distribution

    kubectl get pods -o wide

Check:
- Which nodes are running similar Pods
- Whether scheduling is restricted due to anti-affinity or topology spread
- Whether replica count conflicts naturally with node count

## A Typical Troubleshooting Thought Chain

Assume you have a Pod that can't start.

Don't jump to conclusions about Kubernetes being broken. Instead, ask yourself these questions in this order:

1. Is there enough resource?
2. Does the label match?
3. Are the taints tolerated?
4. Are the affinity/anti-affinity conditions too strict?
5. Are the topology spread requirements too strict?
6. Are there any candidate nodes left after combining multiple conditions?

This approach is more important than memorizing commands.

## Common Scheduling Design Patterns in Production

### 1. Dedicated Node Pool Pattern

Objective:
- Certain nodes run only specific workloads

Approach:
- Nodes are labeled
- Nodes are tainted
- Pods specify nodeSelector + toleration

Examples:
- GPU nodes
- Middleware nodes
- High-performance database nodes

### 2. Multi-replica High Availability Pattern

Objective:
- Don't cluster all replicas on one node
- Distribute replicas across different fault domains

Approach:
- `podAntiAffinity`
- `topologySpreadConstraints`

### 3. Prioritize High-performance Nodes Pattern

Objective:
- Prefer high-performance nodes if possible
- Still run if conditions aren't met

Approach:
- `nodeAffinity preferred`

### 4. Strict Production Node Pattern

Objective:
- Certain workloads can only run on production nodes
- Can't run on test nodes

Approach:
- `nodeAffinity required`
- Or use `nodeSelector` for simple scenarios

## A Complete Example: Combining Multiple Scheduling Strategies

Here's an example that closely resembles production understanding.

Scenario:
- The workload must run on production environment nodes
- Prefer SSD nodes
- The node pool is a dedicated middleware pool with taint protection
- Multiple replicas should be spread across different nodes
- And distribute as evenly as possible

apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-complex-scheduling
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-complex-scheduling
  template:
    metadata:
      labels:
        app: app-complex-scheduling
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: env
                    operator: In
                    values:
                      - prod
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                      - ssd
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: app-complex-scheduling
              topologyKey: kubernetes.io/hostname
      tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "middleware"
          effect: "NoSchedule"
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: app-complex-scheduling
      containers:
        - name: nginx
          image: nginx:1.25

This example, though having many fields, actually answers several questions simultaneously:

- Must be scheduled on prod nodes
- Prefer SSD nodes
- Can enter middleware dedicated nodes
- Avoid clustering with same-type replicas
- Overall distribution should be balanced

This is a typical "multi-rule overlay scheduling" scenario in production environments.

## Why This Stage is Important

Because the following topics you'll learn later:

- Deployment updates and rollbacks
- Application permission control
- Helm deployment
- Middleware cloud migration to K8s
- Application deployment troubleshooting

All assume you already have this prerequisite:

**You know why a Pod gets scheduled on a specific node, and why it sometimes can't be scheduled at all.**

Without understanding this step, many issues like "application fails to start", "replica imbalance", and "stuck after update" will appear confusing.

## What You Should Truly Remember in This Stage is the Thinking Framework, Not the Fields

At this point, you don't need to memorize field names by heart.  
But you should at least form this thinking framework:

- Resources pass the line first
- Pod expresses where it wants to go
- Node expresses whether it accepts the Pod
- Pod considers relationships with other Pods
- Finally check if the overall distribution is reasonable

As long as this framework is established, you'll never get confused when looking at YAML, troubleshooting Pending states, or analyzing scheduling events.

## Scheduling Strategy Stage Knowledge Summary Table

### 1. nodeSelector
- Simplest node selection
- Exact matching
- Suitable for beginners and simple scenarios

### 2. nodeAffinity
- More flexible node selection
- Supports required/preferred
- Supports expressions

### 3. podAffinity
- Let Pod stay close to certain Pods

### 4. podAntiAffinity
- Let Pod avoid certain Pods
- Commonly used for multi-replica dispersion

### 5. taints
- Nodes reject certain Pods

### 6. tolerations
- Pod tolerates node taints
- Just allows entry, not forced scheduling

### 7. topologySpreadConstraints
- Let a group of Pods distribute evenly
- Stronger overall balance than simple anti-affinity

## Key Concepts to Master in This Stage

You need to truly grasp these core understandings:

1. Kubernetes scheduling is essentially "filtering candidate nodes, then selecting the more optimal one"  
2. Resource requests are the prerequisite for scheduling  
3. `nodeSelector` and `nodeAffinity` address where a Pod wants to go  
4. `podAffinity` and `podAntiAffinity` address the relationship between Pods and other Pods  
5. `taints` and `tolerations` address whether a node is willing to accept a Pod  
6. `topologySpreadConstraints` addresses the overall balanced distribution of a group of Pods  
7. Pod Pending is often not a single-point issue, but a result of multiple conditionsOverlay  
8. When troubleshooting Pending, `kubectl describe pod` is the first entry point  
9. Learning scheduling is not about memorizing YAML, but understanding the logic of workload placement  

## Common Troubleshooting Commands Quick Reference

    kubectl get pod
    kubectl get pod -o wide
    kubectl describe pod <pod name>
    kubectl get nodes
    kubectl get nodes --show-labels
    kubectl describe node <node name>
    kubectl get pods --show-labels
    kubectl get pods -l app=<label value> -o wide

## One-Sentence Summary

In this phase of Kubernetes scheduling strategy, the core question being answered is: **Why is a Pod placed on a specific node, and why sometimes it can't be placed at all.**

## Tags
#Kubernetes #ApplyDeployment #SchedulePolicy #PodMovement #nodeAffinity #podAntiAffinity #Taint #topologySpreadConstraints #Pending.

## Operations Extension Understanding
- Kubernetes Official Documentation: Scheduling, Preemption and Eviction  
  https://kubernetes.io/docs/concepts/scheduling-eviction/
- Kubernetes Official Documentation: Assigning Pods to Nodes  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
- Kubernetes Official Documentation: Taints and Tolerations  
  https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
- Kubernetes Official Documentation: Pod Topology Spread Constraints  
  https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/

## Next Day Plan
- Enter `09-Apply Identity and Permission Control`
- First learn [[01-ServiceAccount Basics - Pod Identity in Cluster]]
- Connect "scheduling successfully placing Pods" with "how to stably publish, update, and rollback later"