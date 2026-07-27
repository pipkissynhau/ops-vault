# 04 - Basics of Taints and Tolerations: An Introduction to Taints and Toleration Rules

## Document Description
- Document Focus: Core foundations of Kubernetes scheduling strategies
- Applicable Phases: 07 - Application Deployment / 09 - Scheduling Strategies and Workload Placement
- Learning Objectives:
  - Understand what taints and tolerations address respectively
  - Grasp the scheduling control concept of "nodes rejecting Pods"
  - Be able to interpret basic YAML files for taints/tolerations
  - Differentiate between `NoSchedule`, `PreferNoSchedule`, and `NoExecute`
  - Combine previous concepts like nodeSelector and nodeAffinity to build a comprehensive understanding of scheduling

## Establishing an Initial Intuition

In previous sections, we learned about:

- `nodeSelector`
- `nodeAffinity`
- `podAffinity / podAntiAffinity`

These methods essentially involve:

**Pods actively selecting suitable nodes.**

However, in real-world scenarios, there is another approach:

**Rather than Pods choosing nodes, nodes can declare which Pods they do not want to host.**

This is the core idea behind taints and tolerations.

You can think of it this way:

- `taint`: A "reject mark" applied to a node
- `toleration`: A "permission badge" associated with a Pod

If a node has a taint and a Pod does not have the corresponding tolerance, then the Pod will not be scheduled to that node. Even Pods already running on that node may be evicted.

## Why Are Taints and Tolerations Needed?

Some nodes are not suitable for hosting all types of Pods. Common use cases include:

- Control-plane/master nodes should not run general business Pods
- High-performance nodes should be reserved for critical services
- GPU-enabled nodes should only host Pods that require GPUs
- Nodes under maintenance should avoid scheduling new Pods
- Faulty nodes need to quickly dislodge existing Pods

In other words, taints and tolerations primarily address:

**How nodes can proactively reject inappropriate Pods.**

## What Is a Taint?

A taint is a rule that applies a "reject mark" to a node, informing the scheduler:

**This node has special restrictions; not all Pods are allowed here.**

You can imagine it as a sign at the door of a node:

- Ordinary people are prohibited from entering
- Only specific Pods are allowed
- Some Pods already inside may also need to leave

Taints are applied to nodes, not Pods.

## What Is a Tolerance?

A tolerance is a rule associated with a Pod, indicating its ability to accept certain taints without being directly rejected.

It's important to note that tolerations:

**Do not guarantee scheduling; they only indicate permission.**

In other words:

- Having a tolerance gives a Pod the opportunity to be scheduled
- Whether it will actually be scheduled depends on other factors such as resources, affinity rules, and node status.

## A Critical Relationship

Remember this key relationship:

- Nodes are applied with taints
- Pods have tolerations defined
- Pods without tolerations will be blocked by taints
- Pods with tolerations will only be "not filtered out"; they may still not be selected

Therefore, taints and tolerations are more like:

**Access control mechanisms**
rather than
**final scheduling directives**

## The Basic Structure of a Taint

A taint typically consists of three parts:

- `key`
- `value`
- `effect`

For example:

    dedicated=middleware:NoSchedule

This can be broken down into:

- key: `dedicated`
- value: `middleware`
- effect: `NoSchedule`

It means:

**This node is marked as a dedicated middleware node; Pods without the corresponding tolerance cannot be scheduled here.**

## The Three Most Important Effects

Understanding taints revolves around these three effects:

### 1. NoSchedule

This means:

**Pods without the corresponding tolerance are not allowed to be scheduled to this node.**

It is the most common effect.

Characteristics:

- It only affects **new scheduling attempts**
- Pods already running on the node will not be immediately removed due to this taint

You can think of it as:

**From now on, new ordinary Pods are not welcome here.**

### 2. PreferNoSchedule

This means:

**Try not to schedule Pods without the corresponding tolerance to this node.**

It is not an absolute ban but a soft restriction.

You can interpret it as:

**It's preferable not to have them here, but if no better option exists, they might still be scheduled.**

### 3. NoExecute

This means:

**Pods without the corresponding tolerance are not only not allowed to be newly scheduled, but also existing Pods on this node may be evicted.**

This is the strongest effect.

You can think of it- `nodeAffinity` should be used in conjunction with these other concepts.

## Common combinations: taint + toleration + nodeSelector

This is a very typical combination in production use cases.

### Scenarios

You have a group of nodes dedicated specifically for middleware:

- You apply a taint to the nodes to prevent regular business Pods from being scheduled there.
- You add labels to the nodes to help middleware Pods select them precisely.
- The middleware Pods are configured with both tolerations and nodeSelector.

This approach ensures that "unwanted Pods are kept out" while "required Pods can be accurately placed."

### Node preparation

Label the nodes:

    kubectl label node k8s-node1 dedicated=middleware

Apply a taint to the nodes:

    kubectl taint nodes k8s-node1 dedicated=middleware:NoSchedule

### Pod YAML configuration

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

This configuration means:

- The Pod wants to run only on nodes labeled `dedicated=middleware`.
- Even if those nodes have the `NoSchedule` taint, this Pod will tolerate it.
- Regular Pods do not have these tolerations and will not be scheduled to these nodes.

This is a typical example of using a "dedicated node pool."

## Taints on control-plane nodes

In many Kubernetes clusters, control-plane nodes are typically marked with a taint by default.

Common examples include:

    node-role.kubernetes.io/control-plane:NoSchedule

Or in older versions:

    node-role.kubernetes.io/master:NoSchedule

The purpose of this is to prevent regular business Pods from running on control-plane nodes.

This is a classic use case for taints in Kubernetes.

If you are in a single-node experimental environment or have limited resources and want to allow regular Pods to run on control-plane nodes, you usually need to:

- Remove the taint.
- Or add specific tolerations for those Pods.

For example, to remove the taint:

    kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-

However, this is generally not recommended in production environments.

## Understanding `PreferNoSchedule`

This is a soft restriction.

For example, if you apply a taint like this:

    kubectl taint nodes k8s-node2 special=true:PreferNoSchedule

It means that the scheduler will try to avoid scheduling regular Pods on this node. However, if no other suitable nodes are available, the Pod may still be scheduled there.

So `PreferNoSchedule` is more like a "recommendation" from the scheduler rather than an absolute rule.

At the beginner level, you are more likely to encounter:

- `NoSchedule`
- `NoExecute`

## Understanding `NoExecute`

This has the strongest effect and needs special attention.

If a node is marked with:

    unstable=true:NoExecute

Then Pods that do not have the corresponding toleration will:

- Not be scheduled to this node.
- If already running on this node, they will be evicted.

This type of effect is often used in cases where nodes are experiencing issues or need to be taken offline for maintenance.

## What is `tolerationSeconds`?

When the `effect` is `NoExecute`, you can use `tolerationSeconds` to specify how long a Pod can tolerate the taint before being evicted.

For example:

    tolerations:
      - key: "unstable"
        operator: "Equal"
        value: "true"
        effect: "NoExecute"
        tolerationSeconds: 60

This means that the Pod can tolerate the `unstable=true:NoExecute` taint for up to 60 seconds. After that time, if the taint is still present, the Pod will be evicted.

This is useful for temporary situations such as:

- Short-term node instability.
- Temporary network issues.
- Giving the business some extra time before evicting the Pod.

## A visual way to understand `NoExecute`

You can think of it this way:

- `NoSchedule`: Prevents new Pods from being scheduled to the node.
- `NoExecute`: Not only prevents new Pods from being scheduled, but also forces existing Pods to leave if they are on that node.

This analogy makes it easier to remember the difference between these two concepts.

## Common production scenarios

### 1. Isolating control-plane nodes

The most classic use case. Regular business Pods should not run on control-plane nodes by default.

### 2. Dedicated GPU nodes

GPU nodes are expensive, and it's important to prevent regular business workloads from using them. You can    kubectl taint nodes k8s-node1 dedicated=middleware:NoSchedule

#### 2. Creating a Pod

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

#### 3. Observing Results

    kubectl apply -f middleware-test.yaml
    kubectl get pod -o wide
    kubectl describe pod middleware-test

Through this experiment, you will gain a better understanding of the following concepts:

- `Toleration` merely allows a Pod to exist on a node with a specific taint.
- `NodeSelector` specifies which nodes a Pod should be scheduled to.
- Only when both `Toleration` and `NodeSelector` are properly configured does a Pod behave more closely to how it would in production.

## Key Points to Remember

You need to keep the following core concepts in mind:

1. `Taints` serve as rejection markers on nodes.
2. `Tolerations` indicate a Pod's tolerance for these taints.
3. Pods without any tolerations may be rejected by tainted nodes.
4. The `NoSchedule` effect prevents new Pods from being scheduled to a node with a taint.
5. `PreferNoSchedule` indicates that scheduling should be avoided if possible.
6. `NoExecute` not only prevents scheduling but may also evict existing Pods from a tainted node.
7. Having a `Tolerance` does not guarantee that a Pod will be scheduled to a specific node.
8. In production, common practices combine `taints`, `tolerations`, and `nodeSelector/nodeAffinity` rules.

## Common Commands for Quick Reference

    kubectl get nodes
    kubectl describe node k8s-node1
    kubectl taint nodes k8s-node1 dedicated=middleware:NoSchedule
    kubectl taint nodes k8s-node1 dedicated=middleware:NoSchedule-
    kubectl label node k8s-node1 dedicated=middleware
    kubectl apply -f pod.yaml
    kubectl get pod -o wide
    kubectl describe pod <pod-name>

## In One Sentence

Taints and tolerations work together to ensure that **“inappropriate Pods are prevented from accessing nodes with specific taints, allowing only those that meet the requirements to be scheduled”**, serving as a crucial mechanism in Kubernetes' scheduling system.

## Tags

#Kubernetes #ApplicationDeployment #SchedulingStrategy #Taint #Toleration #NoSchedule #NoExecute #NodeIsolation

## Additional Resources for Deepening Understanding

- Kubernetes Official Documentation: Taints and Tolerations
  https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
- Kubernetes Official Documentation: Assigning Pods to Nodes
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
- Kubernetes Official Documentation: Well-Known Labels, Annotations and Taints
  https://kubernetes.io/docs/reference/labels-annotations-taints/

## Next Steps

- Study [[05-Topology Spread Constraints: Introduction]]
- Understand the differences between "anti-affinity dispersion" and "more balanced topology spread".
- Prepare a comprehensive understanding of scheduling controls for the final summary chapter.