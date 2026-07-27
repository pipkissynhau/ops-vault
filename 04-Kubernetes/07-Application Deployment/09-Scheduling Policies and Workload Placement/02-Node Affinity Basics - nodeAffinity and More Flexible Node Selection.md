# 02-Node Affinity Basics: nodeAffinity and More Flexible Node Selection

## Document Description
- Documentation Location: Advanced Basics of Kubernetes Scheduling Policies
- Applicable Phase: 07-Application Deployment / 09-Scheduling Policies and Workload Placement
- Learning Objectives:
  - Understand why `nodeAffinity` is more flexible than `nodeSelector`
  - Master the basic syntax and common fields of node affinity
  - Comprehend the difference between "must be met" and "preferably met"
  - Be able to write basic affinity scheduling rules based on node labels
  - Be able to troubleshoot Pod Pending issues caused by mismatched node affinities

## Establish an Intuition First

In the previous section, we learned about `nodeSelector`. Its advantage lies in its simplicity and directness, but its capabilities are relatively limited.

For example, it is very suitable for expressing:

- This Pod can only be placed on nodes with `disktype=ssd`
- This Pod can only be placed on nodes with `env=prod`

However, if you want to express the following requirements, `nodeSelector` becomes insufficient:

- It must be placed on production environment nodes and preferably on high-performance ones
- It cannot be placed on certain types of nodes
- It can be placed on any node that matches one of multiple labels
- It is preferred to be placed on a specific type of node, but other nodes are also acceptable if necessary

In such cases, a more flexible approach is needed: `nodeAffinity`.

You can think of it as:

**A node selection mechanism that is stronger, more detailed, and more suitable for real production environments than `nodeSelector`.**

## What is nodeAffinity

`nodeAffinity` is a node affinity scheduling mechanism provided by Kubernetes. It is used to specify:

**On which nodes a Pod wishes or must be run.**

It also operates based on node labels, but compared to `nodeSelector`, it offers greater expressiveness:

- Supports "must be met"
- Supports "preferably met"
- Allows for complex matching expressions
- Supports multiple operators
- Provides more flexible node filtering logic

## Why is it called "Affinity"?

The term "affinity" can be understood as:

**A Pod having a higher "preference" or greater "dependence" on certain nodes.**

For example:

- A database Pod is better suited to run on SSD nodes
- A business Pod prefers to run on production environment nodes
- Certain computational tasks perform better on nodes with special hardware capabilities

In essence, affinity expresses:

**A more suitable placement relationship between a Pod and certain nodes.**

## The Relationship Between nodeAffinity and nodeSelector

You can think of the two in the following way:

- `nodeSelector`: Basic version, precise matching, only supports "must be met"
- `nodeAffinity`: Enhanced version, richer expression capabilities, supports both "must be met" and "preferably met"

In other words:

**`nodeAffinity` is not a replacement for labels; it provides more flexible scheduling description capabilities based on those labels.**

## The Two Core Semantics of nodeAffinity

When learning `nodeAffinity`, it is crucial to first understand its two scheduling semantics.

### 1. requiredDuringSchedulingIgnoredDuringExecution

This means:

**It must be met during scheduling.**

In other words, when a Pod is created, the nodes must meet this rule; otherwise, the Pod will remain in a Pending state.

You can think of it as:

**A hard requirement.**

### 2. preferredDuringSchedulingIgnoredDuringExecution

This means:

**It is preferable to be met during scheduling.**

In other words, the scheduler will give priority to nodes that meet this condition, but if no suitable nodes are available, the Pod can still be scheduled on other nodes as long as it can still function.

You can think of it as:

**A soft preference.**

## Understanding "IgnoredDuringExecution"

These two fields have long names, but for now, you just need to grasp the key points.

`IgnoredDuringExecution` means that:

**Once a Pod has started running, even if the node's labels change later on, Kubernetes will not automatically relocate the Pod due to this affinity rule.**

In other words, these rules mainly apply during scheduling and are not continuously enforced after the Pod starts running.

For beginners, just remember:

- `DuringScheduling`: Checked during creation and scheduling
- `IgnoredDuringExecution`: Not automatically relocated due to label changes after the Pod starts running

## The Most Basic Structure of nodeAffinity

Let's first look at a basic structure. You don't need to memorize it all at once; just get familiar with it.

    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key:Operator: In  
Values:  
- ssd  

Containers:  
- Name: nginx  
  Image: nginx:1.25  

The meaning of this rule is:  

- The scheduler gives priority to nodes with `disktype=ssd`.  
- If no such nodes are available, the Pod can still be scheduled onto other suitable nodes.  
- This preference will not prevent the Pod from running if it cannot be fulfilled.  

## Understanding Weight  

In `preferredDuringSchedulingIgnoredDuringExecution`, there is a `weight` field.  

For example:  
- weight: 100  

This indicates the priority of this preference, with values typically ranging from 1 to 100.  
A higher weight means that this preference is more important.  
If a Pod has multiple preferred rules defined, the scheduler will consider these weights and select the node with the highest score.  

For beginners, remember:  
- `weight` merely indicates the strength of the preference.  
- It does not turn a preference into a requirement.  
- It only affects the “priority” but not whether the Pod can be scheduled at all.  

## The Difference Between Required and Preferred  

This is one of the most critical concepts in this topic.  

### Required  
- Must be met.  
- If not met, the Pod will remain in a Pending state.  
- It represents a hard constraint.  

### Preferred  
- Ideally met, but not necessary for successful scheduling.  
- Even if not met, the Pod may still be scheduled.  
- It represents a soft preference.  

You can think of it this way:  
- Required: This condition is essential; without it, nothing can proceed.  
- Preferred: Having this condition is better, but it’s not a dealbreaker if missing.  

## An Example Using Both Required and Preferred  

Here’s an example that reflects real-world scenarios:  
- The Pod must be scheduled onto a production node.  
- Ideally, it should be on an SSD node.  

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

The scheduling logic for this Pod is as follows:  
1. First, nodes with `env=prod` will be identified.  
2. Among these nodes, those with `disktype=ssd` will be given priority.  
3. If no SSD nodes are available, the Pod can still run on a regular prod node.  

This shows how `nodeAffinity` is more practical in production scenarios than `nodeSelector`.  

## Understanding matchExpressions  

`matchExpressions` can be thought of as a list of expressions for matching node labels.  
Each expression consists of:  
- `key`  
- `operator`  
- `values`  

For example:  
- key: env  
  operator: In  
  values:  
    - prod  

This means that the node’s `env` label must belong to the set `prod`.  

## Understanding nodeSelectorTerms  

The name `nodeSelectorTerms` might seem a bit confusing at first, but it simply refers to multiple sets of node selection criteria.  
Within a single `nodeSelectorTerm`, multiple `matchExpressions` are usually connected by the AND operator.  
In other words, all conditions within this set must be met simultaneously.  

For example:  
```yaml
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
```

This means that the node must meet both conditions: `env=prod` and `disktype=ssd`.  

## Multiple matchExpressions  

Here’s an example with multiple `matchExpressions`:  
```yaml
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
```

This means that the Pod must be scheduled onto a node that meets both                - containerPort: 80

This indicates that all Pods of the Deployment:

- must be on nodes with `env=prod`
- it is preferable that they be on nodes with `disktype=ssd`

## Troubleshooting: Why Does a Pod Remain Pending Due to NodeAffinity?

If a Pod remains Pending, first check:

    kubectl get pod
    kubectl describe pod <pod name>

You will usually see similar information in the `Events` section:

    0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.

This suggests that the issue is likely one of the following:

### 1. Incorrect tag key or value

For example, if the YAML specifies:

    env=production

but the actual node label is:

    env=prod

then they will not match.

### 2. The node does not have the required tag

For instance, if you specify:

    - key: disktype
      operator: In
      values:
        - ssd

but none of the nodes have the `disktype` tag, then the Pod will not be scheduled.

### 3. The required conditions are too strict

If you require:

- `env=prod`
- `disktype=ssd`
- `zone=az1`

and no node in the cluster meets all three conditions, the Pod will remain Pending.

### 4. Combined with other scheduling constraints

Other factors such as taints, anti-affinity, and topology spread can also affect scheduling.

## Recommended Steps for Troubleshooting NodeAffinity

### Step 1: Check Pod Details

    kubectl describe pod <pod name>

### Step 2: Verify the Affinity Specification

Check whether it is set to `required` or `preferred`, and ensure the key, operator, and values are correct.

### Step 3: Check Node Labels

    kubectl get nodes --show-labels
    or
    kubectl describe node <node name>

### Step 4: Ensure at Least One Node Meets the Required Conditions

If none of the nodes meet the required conditions, the Pod will remain Pending.

### Step 5: Consider Other Combined Constraints

This step is more relevant after learning about taints, anti-affinity, and topology spread.

## A Simple Experiment Recommendation

You can perform a simple experiment to separate the effects of `required` and `preferred` affinity.

### Experiment 1: Required Affinity

Label a node:

    kubectl label node k8s-node1 disktype=ssd

Deploy a Pod with required affinity:

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

Apply the Pod:

    kubectl apply -f test-required-affinity.yaml
    kubectl get pod -o wide

Then remove the node label and try again:

    kubectl delete pod test-required-affinity
    kubectl label node k8s-node1 disktype-
    kubectl apply -f test-required-affinity.yaml
    kubectl describe pod test_required-affinity

### Experiment 2: Preferred Affinity

Deploy a Pod with preferred affinity:

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

Apply the Pod:

    kubectl apply -f test-preferred-affinity.yaml
    kubectl get pod -o wide
    kubectl describe pod test-preferred-affinity

You will observe that:

- If the required condition is not met, the Pod cannot run.
- If the preferred condition is not met, the Pod may still be scheduled, but it will not be on the preferred node.

## Key Points to Remember

- `nodeAffinity` uses node labels to schedule Pods more flexibly than `nodeSelector`.
- `requiredDuringSchedulingIgnoredDuringExecution` means the condition must be met.
- `preferredDuringSchedulingIgnoredDuringExecution` means it is preferable but not mandatory.
- The `weight` parameter affects the priority, not whether the Pod can be scheduled.
- Common operators include `In`, `NotIn`, `Exists`, and `DoesNotExist`.
- When troubleshooting Pending Pods, first confirm if any node meets the