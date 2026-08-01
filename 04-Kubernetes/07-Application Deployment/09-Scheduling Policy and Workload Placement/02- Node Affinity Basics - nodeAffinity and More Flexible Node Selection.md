# 02-Node Affinity Basics: nodeAffinity and More Flexible Node Selection

## Document Notes
- Document Positioning: Kubernetes Scheduling Strategy Intermediate Foundation
- Applicable Stage: 07-Application Deployment / 09-Scheduling Strategy and Workload Placement
- Learning Objectives:
  - Understand why `nodeAffinity` is more flexible than `nodeSelector`
  - Master the basic writing methods and common fields of node affinity
  - Understand the difference between "must satisfy" and "prefer to satisfy"
  - Be able to write basic affinity scheduling rules based on node labels
  - Be able to troubleshoot Pod Pending issues caused by affinity mismatches

## Build an Intuition First

The previous lesson learned about `nodeSelector`, which is simple and direct but has limited capabilities.

For example it's very suitable for expressing:

- This Pod can only go to `disktype=ssd` nodes
- This Pod can only go to `env=prod` nodes

But if you want to express the following requirements, `nodeSelector` is insufficient:

- Must go to production environment nodes, and preferably to high-performance nodes
- Cannot go to certain types of nodes
- Can go to any node with multiple labels
- Prefer to go to certain types of nodes, but can go to others if necessary

At this point you need a more flexible approach: `nodeAffinity`.

You can think of it as:

**A node selection mechanism stronger, more precise, and better suited for real production environments than nodeSelector.**

## What is nodeAffinity

`nodeAffinity` is Kubernetes' node affinity scheduling mechanism, used to describe:

**Where a Pod prefers or must run.**

It also works based on node labels, but compared to `nodeSelector`, it has stronger expression capabilities:

- Supports "must satisfy"
- Supports "prefer to satisfy"
- Supports complex matching expressions
- Supports multiple operators
- Supports more flexible node filtering logic

## Why is it called "affinity"

The term "affinity" can be understood as:

**A Pod's preference or dependency for certain nodes.**

For example:

- A database Pod is more suitable for running on SSD nodes
- A business Pod prefers to run on production environment nodes
- Some compute tasks prefer to run on nodes with special hardware capabilities

So affinity essentially expresses:

**A more suitable placement relationship between a Pod and certain nodes.**

## Relationship between nodeAffinity and nodeSelector

You can think of them as having the following relationship:

- `nodeSelector`: Base version, exact match, only "must satisfy"
- `nodeAffinity`: Enhanced version, richer expression, supports "must satisfy" and "prefer to satisfy"

In other words:

**nodeAffinity is not a replacement for labels, but provides more flexible scheduling description capabilities based on labels.**

## Two Core Semantics of nodeAffinity

When learning `nodeAffinity`, the most important thing is to first understand its two scheduling semantics.

### 1. requiredDuringSchedulingIgnoredDuringExecution

This means:

**Must be satisfied during scheduling.**

That is, when a Pod is created, the node must meet this rule, otherwise it won't be scheduled and the Pod will remain Pending.

You can think of it as:

**Hard condition.**

### 2. preferredDuringSchedulingIgnoredDuringExecution

This means:

**Try to satisfy during scheduling.**

That is, the scheduler will prioritize nodes that meet the criteria, but if there are no suitable nodes, it can still schedule to other nodes as long as the overall operation is still possible.

You can think of it as:

**Soft preference.**

## How to Understand `IgnoredDuringExecution`

These field names are long, but you only need to focus on the key points now.

`IgnoredDuringExecution` means:

**Once a Pod is running, even if the node's labels change later, Kubernetes will not automatically evict it because of this affinity rule.**

In other words, these rules mainly take effect "during scheduling", not continuous enforcement after runtime.

For the initial stage, just remember:

- `DuringScheduling`: Checked during creation and scheduling
- `IgnoredDuringExecution`: Not automatically evicted due to label changes after runtime

## Most Basic nodeAffinity Structure

Let's look at a basic structure first. You don't need to remember everything at once, just get a general idea.

    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                      - ssd

This rule means:

**The Pod must be scheduled to a node with `disktype=ssd` label.**

You'll notice it's still using labels for matching, just with more complex and flexible writing.

## Most Commonly Used Operators in nodeAffinity

One of the strengths of `nodeAffinity` is its support for operators.

### 1. In

Indicates the label value must be in a certain list.

For example:

    - key: disktype
      operator: In
      values:
        - ssd

Indicates the node label `disktype` must be `ssd`.

If there are multiple values, for example:

    - key: env
      operator: In
      values:
        - prod
        - staging

Indicates the node label value can be `prod` or `staging`.

### 2. NotIn

Indicates the label value cannot be in a certain list.

For example:

    - key: env
      operator: NotIn
      values:
        - test

Indicates this Pod doesn't want to go to `env=test` nodes.

### 3. Exists

Indicates the label just needs to exist, without caring about the value.

For example:

    - key: gpu
      operator: Exists

Indicates the node just needs to have the `gpu` label, without specific value requirements.

### 4. DoesNotExist

Indicates the label cannot exist.

For example:

    - key: maintenance
      operator: DoesNotExist

Indicates the Pod doesn't want to be scheduled to nodes with the `maintenance` label.

### 5. Gt / Lt

Indicates label value comparison, typically used for numeric scenarios.

For example:

- key: cpu-level
  operator: Gt
  values:
    - "4"

Indicates that the node label `cpu-level` must be greater than 4.

In the initial stage, the most commonly used ones are:

- `In`
- `NotIn`
- `Exists`
- `DoesNotExist`

## First required example: must schedule to SSD nodes

Assume nodes have the following label:

    kubectl label node k8s-node1 disktype=ssd

Write a Pod that requires it to be scheduled to `disktype=ssd` nodes.

    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-affinity-required
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                      - ssd
      containers:
        - name: nginx
          image: nginx:1.25

Apply:

    kubectl apply -f pod-affinity-required.yaml

Check results:

    kubectl get pod -o wide
    kubectl describe pod nginx-affinity-required

If there are nodes with `disktype=ssd` in the cluster, the Pod will be scheduled normally.

If not, the Pod will remain Pending.

## Understanding the essence of required

The essence of `requiredDuringSchedulingIgnoredDuringExecution` is:

**Do not schedule if not satisfied.**

This is very similar to `nodeSelector`.  
But it has stronger expressive power because it can:

- Use expression matching
- Support multiple values
- Support existence and non-existence judgments
- Support more complex logical structures

## Preferred example: prefer scheduling to SSD nodes

This example is not "must", but "best".

    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-affinity-preferred
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                      - ssd
      containers:
        - name: nginx
          image: nginx:1.25

This rule means:

- The scheduler prioritizes `disktype=ssd` nodes
- If there are no such nodes, it can still schedule to other available nodes
- It won't prevent the Pod from running because the preference cannot be satisfied

## How to understand weight

In `preferredDuringSchedulingIgnoredDuringExecution`, there is a `weight` field.

For example:

    - weight: 100

It represents the weight of this preference, typically ranging from 1 to 100.

Higher weight means this preference is more important.  
If a Pod defines multiple preferred rules, the scheduler will score them comprehensively and prioritize nodes with higher scores.

In the initial stage, remember:

- `weight` is just "preference strength"
- It won't turn preferred into required
- It only affects "priority", not "whether scheduling is possible"

## Difference between required and preferred

This is one of the most critical knowledge points in this article.

### required

- Must be satisfied
- Will remain Pending if not satisfied
- Is a hard constraint

### preferred

- Try to satisfy
- Can still successfully schedule if not satisfied
- Is a soft preference

You can understand it this way:

- required: Without this condition, it's impossible
- preferred: With this condition is better, without it can still work around

## An example using both required and preferred

The following example is more realistic:

- Must be on production nodes
- Prefer SSD nodes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-prod-ssd
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
  containers:
  - name: nginx
    image: nginx:1.25
```

This Pod's scheduling logic is:

1. First filter `env=prod` nodes
2. Among these nodes, prioritize `disktype=ssd` nodes
3. If there are no SSD nodes, but there are still regular prod nodes, the Pod can still run

This is why `nodeAffinity` is closer to production than `nodeSelector`.

## How to understand matchExpressions

`matchExpressions` can be understood as:

**A list of node label matching expressions.**

Each expression contains:

- `key`
- `operator`
- `values`

For example:

    - key: env
      operator: In
      values:
        - prod

Its meaning is:

**The node's env label value must be in the set prod.**

## How to understand nodeSelectorTerms

This layer's name can be confusing.

You can first simply understand it as:

**A group of node selection conditions.**

Within a `nodeSelectorTerm`, multiple `matchExpressions` are usually AND relationships.  
That is, the conditions in this group typically need to be satisfied simultaneously.

For example:

nodeSelectorTerms:
  - matchExpressions:
      - key: env
        operator: In
        values:
          - prod
      - key: disktype
        operator: In
        values:
          - ssd

Can be understood as:

- Must be `env=prod`
- And must be `disktype=ssd`

## Understanding Multiple matchExpressions

For example, the following example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: app-complex-affinity
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
                  - key: disktype
                    operator: In
                    values:
                      - ssd
      containers:
        - name: nginx
          image: nginx:1.25

This indicates the Pod must be scheduled on nodes that simultaneously meet the following two conditions:

- `env=prod`
- `disktype=ssd`

## Comparison Between nodeAffinity and nodeSelector

### nodeSelector

Advantages:

- Simple
- Intuitive
- Suitable for beginners

Disadvantages:

- Can only perform exact matches
- Cannot express soft preferences
- Does not support complex conditions

### nodeAffinity

Advantages:

- Supports required and preferred
- Supports expressions
- Supports more complex logic
- More suitable for real production

Disadvantages:

- Longer syntax
- Higher understanding cost
- Easy to be intimidated by field names when starting

## Common Use Cases

### 1. Business must be scheduled on production nodes

For example:

- `env=prod`

This belongs to required.

### 2. Prefer scheduling on high-performance nodes

For example:

- `disktype=ssd`
- `cpu-level=high`

This belongs to preferred.

### 3. Avoid scheduling on maintenance nodes

For example:

    - key: maintenance
      operator: DoesNotExist

### 4. Limit to nodes with certain hardware capabilities

For example:

- GPU nodes
- NVMe nodes
- High-memory nodes

## Using nodeAffinity in Deployment

In production, it's more common to use it in Deployment rather than single Pod.

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-affinity-deploy
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: nginx-affinity-deploy
      template:
        metadata:
          labels:
            app: nginx-affinity-deploy
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
          containers:
            - name: nginx
              image: nginx:1.25
              ports:
                - containerPort: 80

This indicates all Pods in the Deployment:

- Must be on `env=prod` nodes
- Prefer `disktype=ssd` nodes

## Troubleshooting: Why Pod is Always Pending Due to nodeAffinity

If a Pod is always pending, first check:

    kubectl get pod
    kubectl describe pod <pod名>

In `Events`, you'll typically see similar information:

    0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.

This indicates the issue is usually in one of the following directions.

### 1. Label key or value is written incorrectly

For example, YAML writes:

    env=production

But the node actually is:

    env=prod

This will not match.

### 2. The node has no corresponding label

For example, you wrote:

- key: disktype
  operator: In
  values:
    - ssd

But all nodes lack the label `disktype`.

### 3. required conditions are too strict

For example, requiring simultaneously:

- `env=prod`
- `disktype=ssd`
- `zone=az1`

Resulting in no nodes in the cluster satisfying all three conditions.

### 4. Overlapping with other scheduling constraints

For instance, also configured:

- Taints and tolerations
- Resource requests too large
- Pod anti-affinity
- Topology spread

This is not just a nodeAffinity issue, but the result of multiple constraints overlapping.

## Recommended Steps to Troubleshoot nodeAffinity

### Step 1: Check Pod details

    kubectl describe pod <pod name>

### Step 2: Confirm affinity syntax

Focus on:

- `required` or `preferred`
- Whether the key is correct
- Whether the operator is reasonable
- Whether the values match node labels

### Step 3: Check node labels

    kubectl get nodes --show-labels

Or:

    kubectl describe node <node name>

### Step 4: Determine if at least one node meets required conditions

If none exist, the Pod will definitely be Pending.

### Step 5: Consider overlapping constraints

This step is revisited after learning about taints, anti-affinity, and topology spread.

## Recommended Minimal Experiment

You can perform a small experiment to observe required and preferred separately.

### Experiment 1: required

First label a node:

    kubectl label node k8s-node1 disktype=ssd

Deploy a required Pod:

    apiVersion: v1
    kind: Pod
    metadata:
      name: test-required-affinity
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                      - ssd
      containers:
        - name: nginx
          image: nginx:1.25

Apply:

    kubectl apply -f test-required-affinity.yaml
    kubectl get pod -o wide

Then delete the node label and redeploy the Pod to observe Pending:

    kubectl delete pod test-required-affinity
    kubectl label node k8s-node1 disktype-
    kubectl apply -f test-required-affinity.yaml
    kubectl describe pod test-required-affinity

### Experiment 2: preferred

Create another preferred Pod:

    apiVersion: v1
    kind: Pod
    metadata:
      name: test-preferred-affinity
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                      - ssd
      containers:
        - name: nginx
          image: nginx:1.25

Apply:

    kubectl apply -f test-preferred-affinity.yaml
    kubectl get pod -o wide
    kubectl describe pod test-preferred-affinity

You will observe:

- The Pod cannot run if required conditions are unmet
- The Pod may still run normally if preferred conditions are unmet, but it won't be scheduled to the preferred node

## Key Points to Remember

Remember these core points:

1. `nodeAffinity` also schedules Pods based on node labels
2. It is more flexible than `nodeSelector`, supporting complex matching
3. `requiredDuringSchedulingIgnoredDuringExecution` indicates must be met
4. `preferredDuringSchedulingIgnoredDuringExecution` indicates should be met as much as possible
5. `weight` only affects preference strength, not whether scheduling is possible
6. Common operators include `In`, `NotIn`, `Exists`, `DoesNotExist`
7. When troubleshooting Pending status, first confirm if there are nodes that can meet required conditions

## Common Commands Quick Reference

kubectl get nodes  
kubectl get nodes --show-labels  
kubectl describe node k8s-node1  
kubectl label node k8s-node1 disktype=ssd  
kubectl label node k8s-node1 env=prod  
kubectl apply -f pod-affinity-required.yaml  
kubectl apply -f pod-affinity-preferred.yaml  
kubectl get pod -o wide  
kubectl describe pod nginx-affinity-required  
kubectl describe pod nginx-affinity-preferred  

## One-Sentence Summary  

`nodeAffinity` is a more flexible node scheduling mechanism than `nodeSelector`, as it can not only express **where Pods must go** but also **where Pods prefer to go**.  

## Tags  
#Kubernetes #ApplyDeployment #SchedulePolicy #nodeAffinity #NodesAndSex #PodMovement  

## Operational Extension Understanding  
- Kubernetes Official Documentation: Assigning Pods to Nodes  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/  
- Kubernetes Official Documentation: Node Affinity  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity  
- Kubernetes Official Documentation: Well-Known Labels, Annotations and Taints  
  https://kubernetes.io/docs/reference/labels-annotations-taints/  

## Plan for Tomorrow  
- Learn [[03-Pod Companion and anti-reconciliation: Why Pod To approach or disperse]]  
- Understand the difference between "node selection" and "Pod relationship scheduling"  
- Lay the foundation for future learning about taint tolerance and topology spreading