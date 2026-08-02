# 06-Summary of Scheduling Strategy Phase: From Resource Requests to Workload Placement

## Document Description
- Document Location: Summary of Kubernetes Scheduling Strategy Phase
- Applicable Phases: 07-Application Deployment / 09-Scheduling Strategy and Workload Placement
- Learning Objectives:
  - Understand how resource requests, node selection, affinity, taint tolerance, and topology spread work together.
  - Comprehend the steps involved in scheduling a Pod from creation to successful placement.
  - Develop a systematic approach to troubleshooting why a Pod might remain in the Pending state.
  - Analyze why a workload is assigned to a specific node from the perspective of scheduling rules.
  - Lay a foundation for subsequent topics on deployment, rollback, permission control, and Helm.

## What Is Learned in This Phase

At first glance, this phase covers many individual terms:

- `nodeSelector`
- `nodeAffinity`
- `podAffinity`
- `podAntiAffinity`
- `taints`
- `tolerations`
- `topologySpreadConstraints`

However, at a higher level, they all address one core issue:

**Where should a Pod be placed, and why?**

In other words, the focus is not on memorizing these terms but on understanding how Kubernetes' scheduler selects nodes, evaluates them, and determines where a Pod should land.

## What Is the Essence of Scheduling

The essence of scheduling can be summarized as follows:

**Identify candidate nodes in the cluster that can host a Pod, and then select the most suitable one.**

Scheduling is not random or based on a single criterion; it involves a process of **filtering, comparing, and selecting**.

You can think of it as multiple rounds of evaluation:

1. First, check if the candidate nodes meet basic requirements.
2. Then, assess whether they fulfill any specific conditions.
3. Finally, prioritize among the eligible candidates to find the best option.

## Understanding the Pod Scheduling Process from Creation to Success

For beginners, the scheduling process can be roughly described as follows:

1. A Pod is created.
2. The scheduler begins searching for suitable nodes.
3. Nodes that clearly do not meet the requirements are eliminated immediately.
4. Remaining nodes are scored based on their suitability.
5. The node with the highest score is chosen.
6. The Pod is assigned to this node.
7. The kubelet then initiates the container on the target node.

In this section, we focus on steps 3 and 4:

- What conditions cause a node to be excluded?
- How do certain conditions affect a node's priority?

## The Most Fundamental Aspect of Scheduling: Are There Enough Resources?

Before discussing various scheduling strategies, it is essential to remember one thing:

**Without sufficient resources, nothing else matters.**

If a Pod specifies resource requirements, such as CPU or memory amounts, the scheduler will first check whether any available nodes have enough resources to meet these demands.

For example, if a Pod requires 2 CPUs but only one node has 1 CPU available, that node will not be considered for hosting the Pod.

Therefore, the most basic criterion for scheduling is:

**Whether a node has the capacity to support the Pod.**

This explains why understanding how resource requests affect scheduling is a prerequisite for studying advanced scheduling strategies.

## The First Major Category of Scheduling Issues: Where Does the Pod Want to Go?

This category primarily involves the following concepts:

- `nodeSelector`
- `nodeAffinity`

These mechanisms determine **what requirements the Pod has regarding nodes**.

### nodeSelector

The simplest way to select nodes for a Pod.

Suitable for specifying conditions like:

- I want to be placed on a node with `disktype=ssd`.
- I prefer a node where `env=prod` is set.

Characteristics:

- Simple to use.
- Provides precise matching.
- Only allows specifying hard requirements.

### nodeAffinity

A more flexible way to select nodes.

Suitable for expressing conditions such as:

- It must be placed on a node with `env=prod`.
- Preferably on a node with `disktype=ssd`.
- It should not be placed on a node where `env=test` is set.
- It is sufficient if a certain label exists on the node.

Characteristics:

- Supports both required and preferred settings.
- Allows for complex expression of requirements.
- More suitable for real-world production scenarios.

In summary, these two mechanisms help define **the Pod's preferences or rigid requirements regarding nodes**.

## The Second Major Category of Scheduling Issues: Which Nodes Should the Pod Be Close To or Avoid?

This category primarily involves the following concepts:

- `podAffinity`
- `podAntiAffinity`

These mechanisms determine **how other Pods should be positioned relative to the target Pod**.

### podAffinity

Indicates a desire or requirement to be close to certain types of Pods.

Common use cases include:

- Ensuring that business-related Pods are- Whether there are issues with `Insufficient cpu` or `Insufficient memory`

This is usually the starting point for troubleshooting.

### Step 3: Check node resource status

    Use `kubectl describe node <node name>`

Focus on:

- Allocatable resources
- Allocated resources
- Remaining capacity

### Step 4: Examine node labels

    Run `kubectl get nodes --show-labels`

Verify:

- Whether `nodeSelector` matches
- Whether there are any nodes that meet the `nodeAffinity` requirements

### Step 5: Check node taints

    Use `kubectl describe node <node name>`

Confirm:

- Whether any taints exist
- Whether the Pod has corresponding tolerations defined

### Step 6: Review Pod scheduling rules

Carefully check if the YAML file defines:

- `nodeSelector`
- `affinity`
- `tolerations`
- `topologySpreadConstraints`

### Step 7: Assess overall cluster distribution

    Use `kubectl get pods -o wide`

Observe:

- On which nodes similar Pods are running
- Whether scheduling is restricted due to anti-affinity or topology spread constraints
- Whether there are conflicts between the number of replicas and available nodes

## A typical troubleshooting process

If you encounter an issue where a Pod fails to start, don't assume immediately that Kubernetes is malfunctioning. Instead, follow this sequence of questions:

1. Are sufficient resources available?
2. Do the labels match correctly?
3. Are the taints being tolerated by the Pod?
4. Are the affinity/anti-affinity conditions too strict?
5. Are the topology spread constraints too demanding?
6. Are there still eligible nodes after considering all these factors?

This approach is far more valuable than memorizing specific commands.

## Common scheduling design patterns in production

### 1. Dedicated node pool pattern

Objective:

- Ensure that certain types of tasks run on dedicated nodes.

Implementation:

- Apply labels to nodes.
- Apply taints to nodes.
- Define `nodeSelector` and `tolerations` in the Pod YAML.

Examples:

- GPU nodes
- Middleware nodes
- High-performance database nodes

### 2. Multi-replica high availability pattern

Objective:

- Distribute replicas across different fault domains for better reliability.

Implementation:

- Use `podAntiAffinity`.
- Apply `topologySpreadConstraints`.

### 3. Prefer high-performance nodes pattern

Objective:

- Try to run tasks on high-performance nodes, but allow exceptions if necessary.

Implementation:

- Use `nodeAffinity preferred` during scheduling.

### 4. Strict production node restriction pattern

Objective:

- Ensure that certain tasks only run on production nodes.

Implementation:

- Use `nodeAffinity required`.
- Or use `nodeSelector` in simpler cases.

## A comprehensive example: Combining multiple scheduling techniques

Here’s an example that reflects typical production practices:

Scenario:

- The task must run on production environment nodes.
- It prefers SSD-based nodes.
- There is a dedicated middleware node pool with taint protection.
- Replicas should be distributed across different nodes for balance.

```yaml
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
```

This example addresses multiple requirements:

- It ensures that the task runs on prod nodes and prefers SSDs.
- It uses a dedicated middleware node pool with taint protection.
- It distributes replicas across different nodes for balance.

This is the kind of “multi-rule scheduling” commonly seen in production environments```bash
kubectl get nodes --show-labels
kubectl describe node <node-name>
kubectl get pods --show-labels
kubectl get pods -l app=<label-value> -o wide
```

## Summary in One Sentence

During the Kubernetes scheduling phase, the fundamental question being addressed is: **Why are Pods placed on certain nodes, and why sometimes cannot they be placed there at all?**

## Tags
#Kubernetes #Application Deployment #Scheduling Strategy #Pod Scheduling #nodeAffinity #podAntiAffinity #Taint #topologySpreadConstraints #Pending Investigation

## Additional Operations and Understanding
- Kubernetes Official Documentation: Scheduling, Preemption, and Eviction  
  https://kubernetes.io/docs/concepts/scheduling-eviction/
- Kubernetes Official Documentation: Assigning Pods to Nodes  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
- Kubernetes Official Documentation: Taints and Tolerations  
  https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
- Kubernetes Official Documentation: Pod Topology Spread Constraints  
  https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/

## Next Steps
- Move on to `09-Application Identity and Permission Control`
- First, learn about [[01-ServiceAccount Basics: The Identity of Pods within the Cluster]]
- Then, connect the concepts of "successfully scheduling Pods" with "how to stably deploy, update, and roll back applications".