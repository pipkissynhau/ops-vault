# 04 - Taints and Tolerations Basics: Introduction to taints and tolerations

## Document Overview
- Document positioning: Core foundation of Kubernetes scheduling strategies
- Applicable stages: 07-Application Deployment / 09-Scheduling Strategies and Workload Placement
- Learning objectives:
  - Understand what problems taint and toleration respectively solve
  - Master the scheduling control concept of "node rejecting Pod"
  - Be able to read basic taint/toleration YAML
  - Understand the differences between `NoSchedule`, `PreferNoSchedule`, `NoExecute`
  - Be able to establish complete scheduling awareness by combining with previous nodeSelector, nodeAffinity

## First establish an intuitive understanding

The previous few sections covered:

- `nodeSelector`
- `nodeAffinity`
- `podAffinity / podAntiAffinity`

These approaches essentially resemble:

**Pods actively selecting suitable nodes.**

But in real environments, there's another approach:

**Instead of Pods choosing nodes, nodes first declare: which Pods I don't welcome.**

This is the core idea of taint and toleration.

You can initially understand it as:

- `taint`: "Denial mark" on nodes
- `toleration`: "Pass permit" on Pods

If a node is tainted and the Pod lacks corresponding toleration, the Pod cannot be scheduled to this node, and existing Pods may also be evicted.

## Why need taint and toleration

Because some nodes don't want all Pods to run on them.

Common scenarios include:

- control-plane/master nodes don't want to run regular business Pods
- some high-performance nodes are reserved only for critical workloads
- some GPU nodes are reserved only for Pods needing GPU
- some nodes are under maintenance and don't want new Pods scheduled
- some nodes have failed and want existing Pods to migrate quickly

In other words, taint and toleration mainly solve:

**How nodes actively reject unsuitable Pods.**

## What is taint

`taint` is a "taint rule" applied to nodes, telling the scheduler:

**This node has special restrictions, not all Pods can come.**

You can think of it as a sign hanging at the node's entrance:

- Regular people are prohibited from entering
- Only special Pods can enter
- Some Pods already inside may also have to leave

Taints are applied to nodes, not Pods.

## What is toleration

`toleration` is a "toleration rule" written on Pods.

Its purpose isn't "force scheduling", but rather:

**Tell Kubernetes: This Pod can accept certain tainted nodes, and won't be directly rejected because of the taint.**

Be particularly careful:

`toleration` only allows "I can come in",  
not "guarantee I will be scheduled".

In other words:

- With toleration: Pod gets entry qualification
- Whether it can be finally scheduled depends on other conditions, such as resources, affinity, node status, etc.

## The most important relationship

This relationship must be remembered first:

- Node applies taint
- Pod writes toleration
- Pods without toleration will be blocked by taint
- Pods with toleration are only "not filtered", doesn't mean they will be selected

So taint/toleration is more like:

**Admission control**
rather than
**Final scheduling instruction**

## Basic structure of taint

Taints typically consist of three parts:

- `key`
- `value`
- `effect`

Example:

    dedicated=middleware:NoSchedule

Can be broken down into:

- key: `dedicated`
- value: `middleware`
- effect: `NoSchedule`

It expresses the meaning:

**This node is marked as a dedicated middleware node, and Pods without corresponding toleration are not allowed to be scheduled.**

## Three most important effects

Learning taints, the core is understanding the 3 effects.

### 1. NoSchedule

Means:

**Pods without corresponding toleration are not allowed to be scheduled to this node.**

This is the most common one.

Its characteristics:

- Only affects "new scheduling"
- Pods already running on this node won't be immediately evicted because of this taint

You can understand it as:

**From now on, new regular Pods are not welcome.**

### 2. PreferNoSchedule

Means:

**Try not to schedule Pods without corresponding toleration to this node.**

It's not an absolute prohibition, but a soft restriction.

You can understand it as:

**Best not to come, but if there's no better option, may still be scheduled.**

### 3. NoExecute

Means:

**Pods without corresponding toleration cannot be newly scheduled, and existing Pods on this node may also be evicted.**

This is the strongest effect.

You can understand it as:

**Not only not welcome new Pods, existing Pods also have to leave.**

## A simple comparison

### NoSchedule
- Prohibit new Pod scheduling
- Don't evict existing Pods

### PreferNoSchedule
- Try not to schedule new Pods
- Not a mandatory prohibition

### NoExecute
- Prohibit new Pod scheduling
- Also evict intolerant existing Pods

## Apply taint to nodes

The most basic command format is as follows:

    kubectl taint nodes <node-name> <key>=<value>:<effect>

Example, applying a taint to `k8s-node1`:

    kubectl taint nodes k8s-node1 dedicated=middleware:NoSchedule

Meaning:

- This node is a dedicated middleware node
- Regular Pods are by default not allowed to be scheduled here

## View node taints

The most common way is to check node details:

    kubectl describe node k8s-node1

Find the `Taints` field in the output.

You can also first check all nodes:

    kubectl get nodes

Then view detailed information for a specific node.

## Remove node taints

The syntax for removing taints is similar to removing labels, with a minus sign added at the end:

    kubectl taint nodes k8s-node1 dedicated=middleware:NoSchedule-

This removes the taint.

## First intuitive experiment: After tainting a node, regular Pods cannot be scheduled

First, apply a taint to the node:

    kubectl taint nodes k8s-node1 dedicated=middleware:NoSchedule

Then create a regular Pod without writing toleration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-no-toleration
spec:
  containers:
    - name: nginx
      image: nginx:1.25
```

**Application:**

```bash
kubectl apply -f nginx-no-toleration.yaml
```

If your cluster has few optional nodes, or the scheduler happens to select this node, you may find the Pod cannot be scheduled to this taint-protected node.

If there are other untainted nodes in the cluster, the scheduler may place it on another node.

## Add a toleration to the Pod

If you want a Pod to be schedulable on a tainted node, you need to add a toleration.

Here's a basic example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-toleration
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "middleware"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx:1.25
```

This YAML means:

- This Pod can tolerate
- `dedicated=middleware:NoSchedule`
- This taint will not prevent it from being scheduled to the corresponding node

## Core fields of toleration

### key

Corresponds to the taint's key.

Example:

```yaml
key: "dedicated"
```

### operator

There are two common options:

- `Equal`
- `Exists`

#### Equal
Indicates that both the key and value must match.

Example:

```yaml
- key: "dedicated"
  operator: "Equal"
  value: "middleware"
  effect: "NoSchedule"
```

This means it will tolerate:

```
dedicated=middleware:NoSchedule
```

#### Exists
Indicates that the key exists is sufficient, without caring about the value.

Example:

```yaml
- key: "dedicated"
  operator: "Exists"
  effect: "NoSchedule"
```

This means any Pod with a taint key of `dedicated` can be tolerated.

### value

Only needed when `Equal`.

### effect

Must correspond to the taint's effect.

## A common misunderstanding: having a toleration doesn't mean it will definitely be scheduled

This is a particularly misunderstood point.

Many people see that a Pod has a toleration and think:

**It will go to that tainted node.**

Actually, it's not.

`toleration` simply means:

**This node won't block me due to the taint.**

But whether it gets scheduled ultimately depends on:

- Whether there's enough resources
- Whether there's nodeSelector / nodeAffinity
- Whether there are other more optimal nodes
- The scheduler's overall scoring result

So if your goal is:

**Not just "allowing it to go", but "trying or must go"**

You usually need to combine it with:

- `nodeSelector`
- `nodeAffinity`

## Common combination: taint + toleration + nodeSelector

This is a typical combination in production.

### Scenario

You have a batch of middleware-specific nodes:

- Taint the nodes to prevent regular business Pods from coming
- Label the nodes to help middleware Pods select precisely
- Middleware Pods use both toleration and nodeSelector

This enables "blocking others" while "letting our own come precisely".

### Node Preparation

Label the node:

```bash
kubectl label node k8s-node1 dedicated=middleware
```

Taint the node:

```bash
kubectl taint nodes k8s-node1 dedicated=middleware:NoSchedule
```

### Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-middleware
spec:
  nodeSelector:
    dedicated: middleware
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "middleware"
      effect: "NoSchedule"
  containers:
    - name: redis
      image: redis:7
```

This configuration expresses:

- I only want to go to `dedicated=middleware` nodes
- Even if that node has `NoSchedule` taint, I can tolerate it
- Regular Pods have neither toleration nor will they actively select this node

This is a very typical "dedicated node pool" approach.

## Taints on control-plane nodes

In many Kubernetes clusters, control-plane nodes have taints by default.

Common examples include:

```yaml
node-role.kubernetes.io/control-plane:NoSchedule
```

Or in older versions:

```yaml
node-role.kubernetes.io/master:NoSchedule
```

Its purpose is:

**To prevent regular business Pods from running on control plane nodes by default.**

This is a classic example of using taints in Kubernetes.

If you want regular Pods to run on master/control-plane nodes in a single-node experimental environment or resource-constrained environment, you usually need to:

- Remove the taint
- Or add toleration to specific Pods

Example of removing the taint:

```bash
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-
```

But it's generally not recommended to do this in production environments.

## Understanding PreferNoSchedule

This is a soft restriction.

For example, if you apply: /think

kubectl taint nodes k8s-node2 special=true:PreferNoSchedule

Means:

**It's best not to schedule ordinary Pods to this node.**

But if the scheduler finds other nodes unsuitable, it may still schedule the Pod here.

So `PreferNoSchedule` is more like:

- The scheduler's "preference suggestion"
- Rather than an absolute ban

In the early stages, you're more likely to see:

- `NoSchedule`
- `NoExecute`

## Understanding NoExecute

This is the strongest effect and needs special attention.

If a node has:

    unstable=true:NoExecute

Then Pods without corresponding toleration:

- Cannot be scheduled newly
- Existing Pods on it will be evicted

This effect is often related to node anomalies and failover.

## What is tolerationSeconds

When the effect is `NoExecute`, you can use `tolerationSeconds` together.

Example:

    tolerations:
      - key: "unstable"
        operator: "Equal"
        value: "true"
        effect: "NoExecute"
        tolerationSeconds: 60

Means:

- This Pod can temporarily tolerate `unstable=true:NoExecute`
- But only for 60 seconds
- After 60 seconds, if the taint still exists, the Pod will be evicted

This is very suitable for temporary tolerance, such as:

- Node short-term fluctuations
- Instant network issues
- Giving business a little buffer time instead of immediate eviction

## Intuitive Understanding of NoExecute

You can remember it like this:

- `NoSchedule`: Don't let new residents in
- `NoExecute`: Don't let new residents in, existing residents without permission must leave

This analogy is easy to remember.

## Common Production Scenarios

### 1. Control Plane Node Isolation

The most classic scenario.
Ordinary business Pods default not to run on control-plane.

### 2. GPU Node Specialization

GPU nodes are expensive, and we don't want ordinary business to occupy them.
So we can taint GPU nodes, allowing only Pods that truly need GPU to enter via toleration.

### 3. Middleware Dedicated Nodes

For example, Redis, Kafka, MySQL, etc., these middleware want to run in fixed node pools, avoiding mixing with ordinary business.

### 4. Maintenance Window

Temporarily taint nodes to block new business scheduling.

### 5. Node Anomaly Eviction

Combined with `NoExecute`, control eviction for nodes that are unsuitable to continue hosting business.

## How taint / toleration Connects with Previous Knowledge

By this point, you should start forming a complete scheduling picture.

### nodeSelector / nodeAffinity
Solve:

**Where the Pod wants to go.**

### podAffinity / podAntiAffinity
Solve:

**Who the Pod wants to be close to, or avoid.**

### taint / toleration
Solve:

**Whether the node allows the Pod to come.**

So scheduling isn't just one-way selection, but:

**The Pod is choosing the node, and the node is filtering the Pod.**

## Scheduling Failure Troubleshooting Approach

If a Pod can't start, check:

    kubectl describe pod <pod-name>

In Events, you might see hints related to taints, such as certain nodes having untolerated taints.

Common troubleshooting directions are as follows.

### 1. Node has taint, but Pod has no toleration

This is the most common case.

### 2. toleration's key / value / effect is written incorrectly

For example, the node taint is:

    dedicated=middleware:NoSchedule

But the Pod writes:

    key: dedicated
    value: redis
    effect: NoSchedule

This won't match.

### 3. Mistakenly believing toleration forces scheduling

Result: The Pod tolerates the taint, but without nodeSelector / nodeAffinity, it's still scheduled to other ordinary nodes.

### 4. NoExecute causes existing Pods to be evicted

Not a scheduling failure, but the Pod is evicted after running for a while.

### 5. Overlapping with other scheduling rules

For example:

- There is a taint
- There is also nodeAffinity
- There is also resource insufficiency

Finally, it becomes "toleration is written, but it still can't start".

## Recommended Experiment One: Observing NoSchedule

### Objective

Understand "Pods without toleration cannot be scheduled to tainted nodes".

### Steps

#### 1. Taint the node

    kubectl taint nodes k8s-node1 dedicated=test:NoSchedule

#### 2. Create an ordinary Pod

    apiVersion: v1
    kind: Pod
    metadata:
      name: test-no-toleration
    spec:
      containers:
        - name: nginx
          image: nginx:1.25

#### 3. Observe scheduling results

    kubectl apply -f test-no-toleration.yaml
    kubectl get pod -o wide
    kubectl describe pod test-no-toleration

#### 4. Create a Pod with toleration

    apiVersion: v1
    kind: Pod
    metadata:
      name: test-with-toleration
    spec:
      tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "test"
          effect: "NoSchedule"
      containers:
        - name: nginx
          image: nginx:1.25

#### 5. Observe results again

    kubectl apply -f test-with-toleration.yaml
    kubectl get pod -o wide
    kubectl describe pod test-with-toleration

## Recommended Experiment Two: Observing taint + nodeSelector Combination

### Objective

Understand "need to both enter and precisely select the target node".

#### 1. Label the node and apply taint / nodeSelector

kubectl label node k8s-node1 dedicated=middleware  
kubectl taint nodes k8s-node1 dedicated=middleware:NoSchedule  

#### 2. Create Pod  

    apiVersion: v1  
    kind: Pod  
    metadata:  
      name: middleware-test  
    spec:  
      nodeSelector:  
        dedicated: middleware  
      tolerations:  
        - key: "dedicated"  
          operator: "Equal"  
          value: "middleware"  
          effect: "NoSchedule"  
      containers:  
        - name: redis  
          image: redis:7  

#### 3. Observe Results  

    kubectl apply -f middleware-test.yaml  
    kubectl get pod -o wide  
    kubectl describe pod middleware-test  

Through this experiment, you'll gain clearer understanding:  

- toleration merely allows admission  
- nodeSelector is what tells the Pod where to go  
- combining both reflects more production-ready practices  

## Key Points Recap  

Remember these core concepts:  

1. `taint` is the rejection mark on nodes  
2. `toleration` is the Pod's tolerance declaration for taints  
3. Pods without toleration may be rejected by tainted nodes  
4. `NoSchedule` prohibits new Pod scheduling  
5. `PreferNoSchedule` indicates preference against scheduling  
6. `NoExecute` not only blocks new scheduling but may evict existing Pods  
7. `toleration` does not guarantee scheduling to the node  
8. Production common combination is `taint + toleration + nodeSelector/nodeAffinity`  

## Common Command Quick Reference  

    kubectl get nodes  
    kubectl describe node k8s-node1  
    kubectl taint nodes k8s-node1 dedicated=middleware:NoSchedule  
    kubectl taint nodes k8s-node1 dedicated=middleware:NoSchedule-  
    kubectl label node k8s-node1 dedicated=middleware  
    kubectl apply -f pod.yaml  
    kubectl get pod -o wide  
    kubectl describe pod <pod名>  

## One-Sentence Summary  

Taint and toleration essentially implement **"nodes reject unsuitable Pods, only Pods with permission can enter"**, serving as a critical "access control" mechanism in Kubernetes scheduling.  

## Tags  
#Kubernetes #ApplyDeployment #SchedulePolicy #Taint #Toleration #NoSchedule #NoExecute #NodeIsolation  

## Operations Extension Understanding  
- Kubernetes Official Documentation: Taints and Tolerations  
  https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/  
- Kubernetes Official Documentation: Assigning Pods to Nodes  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/  
- Kubernetes Official Documentation: Well-Known Labels, Annotations and Taints  
  https://kubernetes.io/docs/reference/labels-annotations-taints/  

## Next Day Plan  
- Study [[05-Topology Spread Constraints topologySpreadConstraints Getting Started]]  
- Understand the difference between "anti-affinity dispersion" and "more balanced topology distribution"  
- Build comprehensive scheduling control awareness for subsequent stages' summary article