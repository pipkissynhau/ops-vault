# 03-Pod Affinity and Anti-Affinity: Why Some Pods Need to Be Close or Spread Out

## Document Notes
- Document Positioning: Kubernetes Scheduling Strategy Advanced
- Applicable Stages: 07-Application Deployment / 09-Scheduling Strategy and Workload Placement
- Learning Objectives:
  - Understand what problems Pod affinity and anti-affinity solve
  - Master `podAffinity` and `podAntiAffinity` concepts
  - Understand the difference between "Pods being close" and "Pods being spread out"
  - Be able to read basic Pod affinity/anti-affinity YAML
  - Understand common scenarios like multi-replica high availability, same-node deployment, cross-node distribution

## Establishing an Intuition

The first two articles explained:

- `nodeSelector`
- `nodeAffinity`

They essentially answer the same question:

**Which nodes should this Pod be scheduled to.**

But sometimes scheduling problems aren't just about "looking at nodes" but also "looking at where other Pods are."

For example:

- A business Pod wants to be colocated with a cache Pod to reduce network hops
- Multiple replicas of a service shouldn't all be scheduled on the same node to avoid single point of failure
- Some middleware replicas want to be spread across different hosts
- A log collection component wants to be colocated with certain business Pods

At these times, the scheduler needs to reference not just "node labels" but:

**The location of other Pods.**

This is exactly what `podAffinity` and `podAntiAffinity` solve.

## What is Pod Affinity

Pod affinity (`podAffinity`) means:

**This Pod wants or must be scheduled closer to certain Pods.**

This "closeness" isn't arbitrary - it depends on a topology dimension, such as:

- Same node
- Same availability zone
- Same rack
- Same topology domain

At the beginner level, focus first on two most common ones:

- Same-node proximity
- Same-availability-zone proximity

## What is Pod Anti-Affinity

Pod anti-affinity (`podAntiAffinity`) means:

**This Pod wants or must not be scheduled too close to certain Pods.**

The most typical scenario is:

- Multiple replicas of a Deployment shouldn't all be scheduled on the same node
- Avoid having multiple replicas fail together due to single-node failure
- Spread high-availability replicas across different nodes

So you can simply remember:

- `podAffinity`: Proximity
- `podAntiAffinity`: Dispersion

## Why Kubernetes Needs Pod Affinity and Anti-Affinity

Because in real production environments, workloads have "relationships" with each other.

### Scenarios Requiring Proximity

For example:

- An application communicates frequently with local cache, wanting to reduce network latency
- Some collaborative services want to be deployed in the same topology domain
- Log or proxy components want to be as close as possible to target business workloads

### Scenarios Requiring Dispersion

For example:

- Multiple replicas of a web service shouldn't all be concentrated on a single node
- Database, message queue replicas want to be distributed across nodes
- A single-node failure shouldn't take down all service replicas at once

Thus, Pod affinity and anti-affinity essentially answer:

**Should Pods be close to each other or separated.**

## Difference from Node Affinity

This distinction must be clearly understood.

### Node Affinity

Looks at:

**What labels a node has.**

For example:

- `disktype=ssd`
- `env=prod`

### Pod Affinity/Anti-Affinity

Looks at:

**Where other Pods are, and what labels those Pods have.**

In other words:

- Node affinity: Pod selects a node
- Pod affinity: Pod references other Pods to select a location

## Core Dependencies for Pod Affinity and Anti-Affinity

Pod affinity and anti-affinity don't directly look at Pod names, but typically use:

**Pod labels (label) + topologyKey**

To determine scheduling relationships.

In other words, the scheduler will ask:

- Are there Pods with specific labels?
- Where are these Pods located (nodes or topology domains)?
- Should the new Pod be close to them or avoid them?

## What is topologyKey

This is one of the most core fields in this article.

`topologyKey` is used to define:

**Which topology dimension "closeness" or "dispersion" is based on.**

The most common example is:

- `kubernetes.io/hostname`  
  Indicates judgment based on node dimension  
  That is, "same node" or "different nodes"

- `topology.kubernetes.io/zone`  
  Indicates judgment based on availability zone dimension  
  That is, "same availability zone" or "different availability zones"

At the beginner level, the most commonly used is:

    kubernetes.io/hostname

You can understand it as:

**Judging whether Pods are together based on host granularity.**

## Basic Semantics of Pod Affinity

Similar to `nodeAffinity`, Pod affinity/anti-affinity also fall into two categories:

### 1. requiredDuringSchedulingIgnoredDuringExecution

Means:

**Must be satisfied during scheduling.**

If not satisfied, the Pod will remain in Pending state.

### 2. preferredDuringSchedulingIgnoredDuringExecution

Means:

**Should be satisfied during scheduling, but can be skipped if not possible.**

The understanding is similar to node affinity:

- required: Hard constraint
- preferred: Soft preference

## First Intuitive Scenario: Let Pod Be Close to Certain Pods

Suppose you have a Redis Pod with the label:

    app=redis

Now you create a new business Pod that wants to be scheduled as close as possible to Redis to reduce access latency.

At this time, you can use `podAffinity`.

## Pod Affinity Example: Try to Place with Redis on Same Node

    apiVersion: v1
    kind: Pod
    metadata:
      name: app-with-redis-affinity
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - redis
                topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.25

This YAML means:

- Hope this Pod is scheduled near Pods with `app=redis` label  
- "Near" means the same `hostname`  
- In other words, try to run on the same node  

Note that `preferred` is used here, so:  

- If possible to be near Redis, try to be near it  
- If it's really impossible, don't block Pod scheduling  

## Understanding This Rule  

Translated into plain language:  

**If a node already has a Pod with `app=redis`, the scheduler prefers to schedule me on that node.**  

This is the basic logic of Pod affinity.  

## Common Scenario for Pod Anti-Affinity: Replica Dispersion  

This is the most important scenario you should understand in production.  

Assume a Deployment has 3 replicas.  
If all 3 replicas run on the same machine, what problems would occur?  

- If the machine fails, all 3 replicas are lost at once  
- Although you specified multiple replicas, high availability is fake  
- Resource contention becomes more concentrated  

So a more reasonable approach is:  

**Distribute multiple replicas across different nodes as much as possible.**  

This is the classic use case for `podAntiAffinity`.  

## Pod Anti-Affinity Example: Avoid Running with Same-Type Pods on Same Node  

    apiVersion: apps/v1  
    kind: Deployment  
    metadata:  
      name: nginx-anti-affinity  
    spec:  
      replicas: 3  
      selector:  
        matchLabels:  
          app: nginx-demo  
      template:  
        metadata:  
          labels:  
            app: nginx-demo  
        spec:  
          affinity:  
            podAntiAffinity:  
              preferredDuringSchedulingIgnoredDuringExecution:  
                - weight: 100  
                  podAffinityTerm:  
                    labelSelector:  
                      matchExpressions:  
                        - key: app  
                          operator: In  
                          values:  
                            - nginx-demo  
                    topologyKey: kubernetes.io/hostname  
          containers:  
            - name: nginx  
              image: nginx:1.25  

This rule means:  

- Try not to schedule this Pod on the same node as existing `app=nginx-demo` Pods  
- `topologyKey: kubernetes.io/hostname` indicates judgment by node dimension  
- If there are enough nodes, the scheduler will try to spread out the 3 replicas  

## Why preferred is More Common  

Because if you use:  

    requiredDuringSchedulingIgnoredDuringExecution  

It means:  

**Must not run with same-type Pods on the same node.**  

This can directly cause some Pods to fail to schedule when node count is insufficient.  

For example:  
- You have 2 nodes  
- Deployment replica count is 3  
- You require each replica to run on different nodes  

The third Pod will definitely fail to start and remain in Pending state.  

So in general business scenarios:  
- Want to spread deployment but don't want to block scheduling, preferred is commonly used  
- Only consider required when high availability requirements are extremely strict  

## required Anti-Affinity Example: Must Be Spread to Different Nodes  

    apiVersion: apps/v1  
    kind: Deployment  
    metadata:  
      name: nginx-hard-anti-affinity  
    spec:  
      replicas: 3  
      selector:  
        matchLabels:  
          app: nginx-hard  
      template:  
        metadata:  
          labels:  
            app: nginx-hard  
        spec:  
          affinity:  
            podAntiAffinity:  
              requiredDuringSchedulingIgnoredDuringExecution:  
                - labelSelector:  
                    matchExpressions:  
                      - key: app  
                        operator: In  
                        values:  
                          - nginx-hard  
                  topologyKey: kubernetes.io/hostname  
          containers:  
            - name: nginx  
              image: nginx:1.25  

This rule means:  

- Multiple `app=nginx-hard` Pods cannot run on the same node  
- If node count is insufficient, some Pods will definitely remain in Pending state  

## When is Pod Affinity Suitable  

### 1. Deploy closely for services with strong dependencies and frequent communication  

For example:  
- An application frequently communicates with cache, proxy, or auxiliary services  
- Want to reduce cross-node network access  

### 2. Want certain Pods to work collaboratively in the same topology domain  

For example:  
- Low-latency communication within the same availability zone  
- Sharing certain local characteristics on the same node  

But note:  
**Pod affinity concentrates workloads.**  
Concentration has latency advantages but may also bring risks like resource contention and concentrated failures.  

## When is Pod Anti-Affinity Suitable  

### 1. High availability dispersion for multi-replica applications  

This is the most common use case.  

### 2. Avoid clustering same-type business on a single node  

Prevent single point failures from affecting all replicas.  

### 3. Spread middleware replicas  

For example, multiple replicas of some stateful applications want to be distributed across different nodes.  

### 4. Reduce resource contention  

If multiple heavy Pods are placed together, they may compete for CPU, memory, and disk IO.  

## How to Understand labelSelector /think
</think>

### How to Understand labelSelector  

The `labelSelector` defines which Pods should be considered for affinity/anti-affinity rules. It uses `matchExpressions` to specify:  

- **Key**: The label key to match (e.g., `app`)  
- **Operator**: The matching logic (e.g., `In` for exact match)  
- **Values**: The specific label values to match (e.g., `nginx-demo`)  

This ensures the scheduler applies rules only to Pods with the specified labels.

Pod affinity/anti-affinity isn't about directly writing "I want to be near Pod A", but selecting target Pods through labels.

Example:

    labelSelector:
      matchExpressions:
        - key: app
          operator: In
          values:
            - redis

This means:

**Find all Pods with labels matching `app=redis`.**

Then determine which topology domains these Pods fall into based on `topologyKey`.

## A more practical understanding

You can think of Pod affinity and anti-affinity as "social rules".

### Pod Affinity

This Pod says:

**I want to be neighbors with a certain type of Pod.**

### Pod Anti-Affinity

This Pod says:

**I don't want to be crowded with a certain type of Pod.**

The standard for "neighbors" or "separation" is defined by `topologyKey`.

## Relationship between Pod Affinity/Anti-Affinity and Replica Count

This is a crucial understanding in production environments.

Assume you have 3 replicas:

- Without anti-affinity, all 3 replicas might be scheduled to the same node
- With preferred anti-affinity, the scheduler will try to spread them out
- With required anti-affinity, they must be spread out, otherwise the Pods can't start

So high availability of replicas isn't just about `replicas=3`.  
You also need to check:

**Whether these replicas are actually spread across different failure domains.**

## Difference with topologySpreadConstraints

You'll learn about `topologySpreadConstraints` later.  
Let's first establish a fuzzy understanding:

- `podAntiAffinity` is more like "don't crowd with certain Pods"
- `topologySpreadConstraints` is more like "distribute Pods evenly across topology domains"

In other words:

- Anti-affinity emphasizes "avoidance"
- Topology spread emphasizes "even distribution"

At this stage, just thoroughly understand Pod affinity/anti-affinity.

## Common Anti-Affinity Practice in Deployments

In actual production, a typical base writing is to spread the same Deployment's Pods across different nodes:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web-app
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: web-app
      template:
        metadata:
          labels:
            app: web-app
        spec:
          affinity:
            podAntiAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
                - weight: 100
                  podAffinityTerm:
                    labelSelector:
                      matchLabels:
                        app: web-app
                    topologyKey: kubernetes.io/hostname
          containers:
            - name: nginx
              image: nginx:1.25

This is a very typical writing, recommended to be familiar with.

Its goal is simple:

**Try to prevent multiple replicas of the same application from being scheduled to the same node.**

## Troubleshooting: Why Pod Fails to Schedule Due to Affinity/Anti-Affinity Rules

If a Pod fails to schedule, first check:

    kubectl describe pod <pod-name>

You might see scheduling failure information related to affinity/anti-affinity in the Events.

Common causes include:

### 1. Required conditions are too strict

Example:

- Must be on the same node as certain Pods
- But there are no such Pods in the cluster

Or:

- Must be spread across different nodes with same type Pods
- But there aren't enough nodes

### 2. labelSelector doesn't match target Pods

For example, you write:

    app=redis

But the target Pod's actual label is:

    app=redis-master

The scheduler can't find the Pods you're referencing.

### 3. topologyKey is unreasonable

Example:

- Nodes don't have the corresponding topology label
- Selected the wrong topology dimension
- Thought it was by node, but actually wrote by zone

### 4. Conflicts with other scheduling constraints

For example, there are also:

- nodeAffinity
- taints / tolerations
- resources requests
- topology spread

It's possible that "each rule alone is correct, but combined there's no node available".

## Recommended Troubleshooting Steps

### Step 1: Check Pod details

    kubectl describe pod <pod-name>

First look at Events.

### Step 2: Confirm target Pods with the correct labels exist

For example:

    kubectl get pods --show-labels

Or:

    kubectl get pods -l app=redis -o wide

Confirm whether `labelSelector` selected Pods actually exist.

### Step 3: Confirm topologyKey is reasonable

If you write:

    topologyKey: kubernetes.io/hostname

You're essentially judging by the "node" dimension.

### Step 4: Confirm required rules aren't too strict

Especially required anti-affinity, which is easy to lock yourself in small clusters.

### Step 5: Consider node count and replica count together

For example:

- 2 nodes
- 3 replicas
- Required anti-affinity by hostname

The third replica will inevitably fail to schedule.

## A Minimal Experiment Suggestion: Observe Anti-Affinity Spread Effect

You can do a simple experiment yourself.

### Experiment Goal

Spread 3 Nginx replicas across different nodes as much as possible.

### Experiment YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-spread-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-spread-test
  template:
    metadata:
      labels:
        app: nginx-spread-test
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: nginx-spread-test
                topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.25

Application:

    kubectl apply -f nginx-spread-test.yaml

Check distribution:

    kubectl get pods -o wide

You can observe whether multiple replicas are spread across different nodes.

If your cluster has enough nodes, the effect will be more noticeable.

## Key Points Recap

You need to remember these key points:

1. `podAffinity` indicates that the Pod should be placed near certain Pods  
2. `podAntiAffinity` indicates that the Pod should avoid certain Pods  
3. They refer to "other Pods' labels and locations", not just node labels  
4. `topologyKey` determines the "nearby or spread" evaluation dimension  
5. `kubernetes.io/hostname` is the most common node-based evaluation  
6. The most common practice for multi-replica applications is to use `podAntiAffinity` for spreading deployment  
7. `preferred` is more suitable for general business, `required` is stricter but more likely to cause Pending  

## Common Command Quick Reference

    kubectl get pods
    kubectl get pods -o wide
    kubectl get pods --show-labels
    kubectl get pods -l app=redis -o wide
    kubectl apply -f pod-affinity.yaml
    kubectl apply -f pod-anti-affinity.yaml
    kubectl describe pod <pod名>

## One-Sentence Summary

Pod affinity and anti-affinity solve the problem of **whether Pods should be placed near or spread apart from each other**, where affinity emphasizes "co-location for collaboration" and anti-affinity emphasizes "spread for high availability".

## Tags
#Kubernetes #ApplyDeployment #SchedulePolicy #PodFamily #PodAgainstFamily #HighAvailable #Duplicate

## Operations Extension Understanding
- Kubernetes official documentation: Assigning Pods to Nodes  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
- Kubernetes official documentation: Inter-pod affinity and anti-affinity  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity
- Kubernetes official documentation: Well-Known Labels, Annotations and Taints  
  https://kubernetes.io/docs/reference/labels-annotations-taints/

## Next Day Plan
- Learn [[04-Taints and Tolerations Basics - Getting Started]]
- Understand the difference between "active node selection" and "rejecting Pod on node"
- Lay the foundation for more comprehensive scheduling control methods in future learning