# 05-Topological Spread Constraints: An Introduction to `topologySpreadConstraints`

## Documentation Overview
- Document Category: Advanced Basics of Kubernetes Scheduling Policies
- Applicable Phases: 07-Application Deployment / 09-Scheduling Policies and Workload Placement
- Learning Objectives:
  - Understand what `topologySpreadConstraints` addresses.
  - Distinguish it from `podAntiAffinity`.
  - Master the basic YAML syntax for topology spread settings.
  - Comprehend terms like `maxSkew`, `topologyKey`, `whenUnsatisfiable`, and `labelSelector`.
  - Recognize why multi-replica applications require more balanced distribution.

## Building an Intuitive Understanding First

In the previous section, we learned about `podAntiAffinity`. It aims to prevent certain Pods from being concentrated in one place. For example:

- Multiple replicas of the same Deployment should not all reside on a single node.
- Critical Pods should not be placed too close to similar Pods.
- Replicas should be distributed as evenly as possible to mitigate the impact of single-node failures.

However, this approach is often insufficient in real production scenarios. What we actually strive for is not just avoiding clustering but achieving **as even a distribution as possible**. For instance:

- With 6 replicas, it’s ideal if they are spread across 3 nodes in a pattern like 2, 2, 2.
- Or with 4 replicas, if they are evenly distributed across 2 availability zones.

This is where `topologySpreadConstraints` comes into play.

## What is Topological Spread

Topological spread can be understood as ensuring that a group of Pods is **evenly distributed** across certain logical dimensions such as nodes, availability zones, or racks. The key idea here is not just preventing clustering but ensuring overall balance in distribution.

## Why Are `topologySpreadConstraints` Needed?

Many use cases cannot achieve the desired distribution with only `podAntiAffinity`. For example, consider a cluster with 3 nodes and 6 replicas:

- If only `podAntiAffinity preferred` is used, the scheduler might assign the replicas as follows:
    - node1: 3 Pods
    - node2: 2 Pods
    - node3: 1 Pod

While this avoids concentrating all replicas on one node, it may not be optimal for high availability and load balancing. A better distribution would be:

- node1: 2 Pods
- node2: 2 Pods
- node3: 2 Pods

This requires a mechanism to explicitly define the goal of balanced distribution, and that’s where `topologySpreadConstraints` comes in.

## The Core Problem That `topologySpreadConstraints` Solves

You can think of it as addressing the question: **What is the maximum allowable difference in the number of Pods across different logical dimensions for the same type of Pod?** In other words, it focuses on whether the overall distribution of a group of Pods is balanced or not, rather than individual Pod interactions.

## Differences Between `topologySpreadConstraints` and Pod Anti-Affinity

This is one of the most crucial comparisons in this chapter:

### Pod Anti-Affinity

It emphasizes avoiding clustering with specific types of Pods. Its focus is on **avoidance**.

### `topologySpreadConstraints`

It focuses on achieving balance across multiple logical dimensions. Its goal is **equilibrium**.

To remember it simply:

- `podAntiAffinity` prevents clustering.
- `topologySpreadConstraints` ensures balanced distribution.

## What Are Topological Dimensions?

“Topological dimensions” refer to the units used for analyzing and distributing Pods. Common examples include nodes and availability zones. For example, if ` topologyKey` is set to `kubernetes.io/hostname`, then distribution will be based on nodes; if it’s set to `topology.kubernetes.io/zone`, then distribution will be based on availability zones.

## What Is `topologyKey`?

`topologyKey` specifies the dimension along which Pods should be distributed and analyzed. For example:

### 1. Distribution by Node

    topologyKey: kubernetes.io/hostname

This means that each node will be considered a separate topological dimension.

### 2. Distribution by Availability Zone

    topologyKey: topology.kubernetes.io/zone

This means that each availability zone will be considered a separate topological dimension.

Therefore, `topologyKey` determines the **axis along which distribution is calculated**.

## What Is `labelSelector`?

`topologySpreadConstraints` does not apply to all Pods but only to a specific group of target Pods identified by a `labelSelector`. For example:

    labelSelector:
      matchLabels:
        app: nginx-demo

This means that the distribution analysis will only consider Pods with the label `app=nginx-demo`.

Therefore, topology spread usually focuses on analyzing and balancing the distribution of **the same application, the same group of replicas, or pods```markdown
labelSelector:
  matchLabels:
    app: nginx-spread-hard
containers:
  - name: nginx
    image: nginx:1.25
```

This means that:

- It is necessary to maintain as even a distribution as possible.
- If scheduling results in a skew greater than `maxSkew`, the Pod will not be scheduled.

This approach is more stringent, but it also increases the likelihood of Pods remaining in the "Pending" state when resources are insufficient or the number of nodes is inadequate.

## When to Use ScheduleAnyway

ScheduleAnyway is commonly used in the following scenarios:

- For general business applications where a balanced distribution is desired without delaying scheduling.
- When cluster resources may be unstable.
- Where priority is given to ensuring that services can start and run, with distribution being a secondary concern.

In other words:

**Ensure that services can operate first, and then strive for an even distribution.**

## When to Use DoNotSchedule

DoNotSchedule is typically used in situations where:

- High availability requirements are strict.
- It is necessary to strictly control the distribution of replicas.
- Excessive skew is not acceptable.
- There is a high sensitivity to risks within a single fault domain.

However, it is important to note that more stringent rules increase the likelihood of scheduling failures.

## Example of Distribution by Availability Zone

If your cluster is deployed across multiple availability zones, you can use `topologySpreadConstraints` to distribute Pods evenly across these zones.

```yaml
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
```

This means that:

- The 4 replicas will be distributed as evenly as possible across the available zones.
- If there are 2 zones, the distribution would ideally be 2 in one zone and 2 in the other.
- If there are 3 zones, the distribution might be 2 in one zone, 1 in another, and 1 in the third.

## Why It Is More Effective Than Anti-Affinity for Achieving Even Distribution

While `podAntiAffinity` aims to prevent Pods of the same type from being placed in the same zone, `topologySpreadConstraints` explicitly takes into account the current number of Pods of the same type within each zone. As a result, it is more direct and effective at achieving an even distribution.

For example:

- Anti-affinity focuses on preventing clustering.
- Topology spread constraints focus on ensuring an even distribution across zones.

These two approaches serve related but different purposes.

## A Practical Understanding

You can think of `podAntiAffinity` as a rule that says, "Don't put all your eggs in one basket," while `topologySpreadConstraints` is more like saying, "Not only should you not put them all in one basket, but you should also distribute them evenly across multiple baskets."

This analogy helps to understand the difference between these two scheduling mechanisms.

## Typical Production Scenarios

### 1. Balanced Distribution of Multiple Replicas for Web Services

For front-end services with multiple replicas, it is important to distribute them across different nodes and zones to improve availability and load balancing.

### 2. Distribution of Middleware Replicas

In some cases, it is necessary to prevent middleware replicas from being concentrated in a single fault domain.

### 3. Disaster Recovery Across Multiple Availability Zones

It is crucial to ensure that business replicas are evenly distributed across different AZs to avoid losing most instances in the event of a single AZ failure.

### 4. Preventing Node Hot Spots

It is important to prevent a single node from handling too many Pods of the same type, which could lead to excessive resource usage.

## The Importance of Ensuring That `labelSelector` Matches the Target Pods

This is a common issue in practice. For example, if you define:

```yaml
labelSelector:
  matchLabels:
    app: nginx-demo
```

The scheduler will only consider Pods with `app=nginx-demo`. If the actual labels of the Pod template do not match this, the desired distribution will not be achieved.

Therefore, it is important to ensure that:

- `.spec.selector.matchLabels`
- `.spec.template.metadata.labels`
- `topologySpreadConstraints.labelSelector`

are consistent in meaning.

## A Common Deployment Template

Here is a basic deployment template commonly used in production for achieving even distribution:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
               whenUnsatisfiable: ScheduleAnyway
              labelSelector:
                matchLabels:
                  app: nginx-topology-test
          containers:
            - name: nginx
              image: nginx:1.25

Application:

    kubectl apply -f nginx-topology-test.yaml

View:

    kubectl get pods -o wide

You can observe whether these replicas are distributed as evenly as possible across different nodes.

If your cluster has 3 or more nodes, this experiment will be more apparent.

## Key Points to Remember

You should keep the following key points in mind:

1. `topologySpreadConstraints` ensures that a group of Pods is distributed as evenly as possible within a topology domain.
2. It focuses on achieving overall distribution balance, not just preventing clusters of Pods from forming.
3. The `topologyKey` determines the dimension along which the Pods should be spread, such as nodes or availability zones.
4. The `labelSelector` specifies which type of Pods to consider for this distribution.
5. `maxSkew` defines the maximum allowed difference in the number of Pods across different topology domains.
6. The setting `whenUnsatisfiable: DoNotSchedule` is more stringent, while `whenUnsatisfiable: ScheduleAnyway` is more lenient.
7. Similar to `podAntiAffinity`, `topologySpreadConstraints` aims for balance rather than simply avoiding clustering.

## Common Commands

    kubectl get pods
    kubectl get pods -o wide
    kubectl describe pod <pod_name>
    kubectl get nodes --show-labels
    kubectl describe node <node_name>
    kubectl apply -f deployment.yaml

## Summary

`topologySpreadConstraints` essentially instructs the scheduler to distribute a group of Pods not only across different nodes but also to achieve balance within these dimensions such as nodes and availability zones.

## Tags

#Kubernetes #Application Deployment #Scheduling Strategies #TopologySpreadConstraints #Topological Distribution #High Availability #Replica Balancing

## Additional Resources for Operations

- Kubernetes Official Documentation: Pod Topology Spread Constraints
  https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/
- Kubernetes Official Documentation: Assigning Pods to Nodes
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
- Kubernetes Official Documentation: Well-Known Labels, Annotations and Taints
  https://kubernetes.io/docs/reference/labels-annotations-taints/

## Next Steps

- Study [[06-Scheduling Strategies Phase Summary: From Resource Requests to Workload Placement]]
- Integrate `nodeSelector`, `nodeAffinity`, `podAffinity/AntiAffinity`, `taint/toleration`, and `topologySpreadConstraints` into a comprehensive scheduling framework.
- Develop a systematic approach to identifying why Pods might remain in the "Pending" state.