# 01-nodeSelector Basics: Scheduling Pods to Specific Nodes

## Document Notes
- Document Location: Kubernetes Scheduling Strategy Introduction
- Applicable Stage: 07-Application Deployment / 09-Scheduling Strategies and Workload Placement
- Learning Objectives:
  - Understand why Pods need to be scheduled to specific nodes
  - Master the basic relationship between node labels and nodeSelector
  - Be able to write the most basic YAML for node label-based scheduling
  - Be able to troubleshoot Pod Pending issues caused by mismatched node selection conditions

## Core Concepts

In Kubernetes, Pods are typically scheduled to nodes automatically by the scheduler.  
But in real environments, not all Pods are suitable for "running on any node".

Common scenarios include:

- Certain business Pods can only run on SSD nodes
- GPU applications can only run on nodes with GPUs
- Certain log, monitoring, and caching components prefer dedicated nodes
- Test and production workloads want physical isolation
- Certain middleware needs to be fixed on nodes with specific hardware capabilities

At this point, we need to manually give the scheduler a restriction condition to prevent it from scheduling arbitrarily.

The most basic implementation method is `nodeSelector`.

## What Are Node Labels

Node labels are essentially key-value tags on Kubernetes nodes that describe node characteristics.

Examples:

- `disktype=ssd`
- `env=prod`
- `zone=az1`
- `gpu=true`
- `workload=middleware`

You can think of them as labeling nodes:

- This machine has SSD disks
- This machine belongs to the production environment
- This machine is in a specific availability zone
- This machine supports GPU
- This machine is dedicated to certain types of workloads

Pods can then decide where to run based on these labels during scheduling.

## What Is nodeSelector

`nodeSelector` is a field in Pod specifications that declares:

**This Pod can only be scheduled to nodes with specified labels.**

Its characteristics are:

- Simple and direct
- Easy to understand
- Suitable for beginners
- Only supports exact matches
- Does not support complex expressions

Therefore, it is very suitable as the first step in learning Kubernetes scheduling.

## nodeSelector Working Logic

The overall process can be understood as follows:

1. First, label nodes
2. Pod declares `nodeSelector` in YAML
3. Scheduler checks all nodes during scheduling
4. Only nodes meeting label conditions are eligible to receive this Pod
5. If no nodes meet the conditions, the Pod will remain in Pending state

In other words, `nodeSelector` essentially does:

**Filter eligible scheduling targets by node labels.**

## Viewing Node Labels

First check which nodes are in the cluster:

    kubectl get nodes

View all nodes and their labels:

    kubectl get nodes --show-labels

If you want to see detailed information about a specific node:

    kubectl describe node <node name>

Example:

    kubectl describe node k8s-node1

Focus on the `Labels` section in the output.

## Labeling Nodes

Example: Add an SSD label to node `k8s-node1`:

    kubectl label node k8s-node1 disktype=ssd

Another example: Add a business purpose label:

    kubectl label node k8s-node1 workload=frontend

Check the result:

    kubectl get node k8s-node1 --show-labels

## Modifying Existing Labels

If a label already exists, use `--overwrite` to overwrite the value:

    kubectl label node k8s-node1 disktype=hdd --overwrite

## Removing Labels

To delete a label, add a minus sign after the key:

    kubectl label node k8s-node1 disktype-

## First nodeSelector Example

The following is the simplest Pod that requires scheduling only to nodes with `disktype=ssd` label.

    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-ssd
    spec:
      nodeSelector:
        disktype: ssd
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80

Save as `pod-nodeselector.yaml` and apply:

    kubectl apply -f pod-nodeselector.yaml

Check which node the Pod is running on:

    kubectl get pod -o wide

If scheduling is successful, you'll see it placed on a node meeting the conditions.

## Verifying Pod Scheduling Destination

Check detailed information:

    kubectl describe pod nginx-ssd

Focus on:

- `Node`
- `Events`

Example output:

    Node:         k8s-node1/192.168.1.21

This indicates the Pod was scheduled to `k8s-node1`.

## Practice Scenario 1: Scheduling Pod to Specific Business Node

### Step 1: Label the Node

Assume you want to fix frontend business to `k8s-node2`:

    kubectl label node k8s-node2 apparea=frontend

### Step 2: Create Pod

    apiVersion: v1
    kind: Pod
    metadata:
      name: web-frontend
    spec:
      nodeSelector:
        apparea: frontend
      containers:
        - name: nginx
          image: nginx:1.25

### Step 3: Apply and Verify

    kubectl apply -f web-frontend.yaml
    kubectl get pod -o wide
    kubectl describe pod web-frontend

## Practice Scenario 2: Deployment with nodeSelector

In actual production, Deployments are more commonly used than single Pods.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-deploy
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      nodeSelector:
        disktype: ssd
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

**Application:**

```bash
kubectl apply -f nginx-deploy.yaml
```

**Check results:**

```bash
kubectl get pods -o wide
```

If scheduling conditions are met, replicas will be placed on nodes with `disktype=ssd` label.

## What happens if no matching nodes are found

If a Pod specifies the following scheduling condition:

```yaml
nodeSelector:
  disktype: ssd
```

But there are no nodes with this label in the cluster, the Pod will remain in Pending state.

**Check:**

```bash
kubectl get pod
```

You might see:

```
NAME        READY   STATUS    RESTARTS   AGE
nginx-ssd   0/1     Pending   0          20s
```

**View detailed information:**

```bash
kubectl describe pod nginx-ssd
```

In `Events` you'll typically see similar information:

```
0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.
```

This indicates that no nodes currently meet this Pod's scheduling requirements.

## Troubleshooting nodeSelector causing Pending

When a Pod fails to schedule due to `nodeSelector`, follow these steps in order.

### 1. Check Pod details

```bash
kubectl describe pod <pod-name>
```

Focus on `Events`.

### 2. Verify nodeSelector syntax in YAML

Confirm:

- Is the key written correctly?
- Is the value written correctly?
- Are the case sensitivities matched?

For example, this will fail:

```yaml
nodeSelector:
  disktype: SSD
```

While the node's actual label is:

```
disktype=ssd
```

Because label values are case-sensitive.

### 3. Check actual node labels

```bash
kubectl get nodes --show-labels
```

Or:

```bash
kubectl describe node <node-name>
```

### 4. Verify labels are applied to correct nodes

Sometimes the labels are correct, but applied to the wrong machine.

### 5. Check for other scheduling constraints

For example, additional constraints may include:

- Resource insufficiency
- Taints / tolerations
- Affinity
- Topology spread

However, at this introductory stage, focus first on `nodeSelector` itself.

## Characteristics of nodeSelector matching

`nodeSelector` uses "all conditions must be met" logic.

For example:

```yaml
nodeSelector:
  env: prod
  disktype: ssd
```

Indicates nodes must simultaneously satisfy:

- `env=prod`
- `disktype=ssd`

It's not sufficient to meet just one, both conditions must be met.

## Multi-condition example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-prod-ssd
spec:
  nodeSelector:
    env: prod
    disktype: ssd
  containers:
    - name: nginx
      image: nginx:1.25
```

Only nodes with both these labels will accept this Pod.

## Common use cases for nodeSelector

### 1. Scheduling by hardware capabilities

For example:

- SSD nodes
- GPU nodes
- High-memory nodes

### 2. Environment isolation

For example:

- `env=test`
- `env=prod`

### 3. Node purpose isolation

For example:

- `workload=logging`
- `workload=middleware`
- `workload=frontend`

### 4. Simple node selection in small clusters

For experimental environments, testing environments, and small-scale business environments, `nodeSelector` is very practical.

## Limitations of nodeSelector

Although simple, it has notable limitations.

### 1. Only supports exact matching

It can only express "must match this label".

Cannot express:

- Prefer certain types of nodes
- Match A or B
- Avoid certain types of nodes

### 2. Does not support complex expressions

Complex scheduling scenarios typically use `nodeAffinity`.

### 3. Limited flexibility

When your scheduling rules start to become complex, `nodeSelector` often becomes insufficient.

The typical learning order is:

- Learn `nodeSelector` first
- Then learn `nodeAffinity`

## Difference between nodeSelector and nodeName

These concepts are often confused.

### nodeSelector

Selects nodes by labels.

For example:

```yaml
nodeSelector:
  disktype: ssd
```

Indicates all nodes meeting the condition are acceptable.

### nodeName

Directly specifies a specific node.

For example:

```yaml
spec:
  nodeName: k8s-node1
```

Indicates the Pod is forcibly scheduled to a specific node.

The difference is:

- `nodeSelector` is more flexible and better suited for daily use
- `nodeName` binds to a specific machine, typically not used as a regular scheduling method

## Experimental recommendation

You can create a minimal experiment to observe nodeSelector's effects.

### Experimental objective

Have a Pod run only on nodes with a specific label.

### Experimental steps

#### 1. Check nodes

```bash
kubectl get nodes
```

#### 2. Label a node /think

kubectl label node k8s-node1 disktype=ssd

#### 3. Create Test Pod

    apiVersion: v1
    kind: Pod
    metadata:
      name: test-node-selector
    spec:
      nodeSelector:
        disktype: ssd
      containers:
        - name: nginx
          image: nginx:1.25

#### 4. Apply YAML

    kubectl apply -f test-node-selector.yaml

#### 5. Check Pod Scheduling Result

    kubectl get pod -o wide

#### 6. Test Pending After Removing Label

    kubectl delete pod test-node-selector
    kubectl label node k8s-node1 disktype-
    kubectl apply -f test-node-selector.yaml
    kubectl get pod
    kubectl describe pod test-node-selector

Through this experiment, you can clearly observe:

- When a node has matching labels, the Pod is scheduled normally
- When a node lacks matching labels, the Pod remains in Pending state

## Production Understanding Supplement

In production environments, node labels are typically not arbitrarily assigned but should follow unified planning.

Common label dimensions include:

- Hardware capabilities: `disktype=ssd`
- Environment type: `env=prod`
- Availability zone: `topology.kubernetes.io/zone=az1`
- Node role: `node-role=middleware`
- Special capabilities: `gpu=true`

A chaotic label system can lead to:

- Difficult YAML maintenance
- Inconsistent scheduling rules
- Hard-to-trace issues
- Increased team collaboration costs

From a higher level, scheduling is not just a YAML writing issue, but part of node planning and resource placement strategy.

## Key Points Recap

You need to remember the following:

1. `nodeSelector` is used to select scheduling targets based on node labels
2. Nodes must have labels first for Pods to select them based on labels
3. Multiple `nodeSelector` conditions must be met simultaneously
4. When no matching nodes exist, the Pod will remain in Pending state
5. Check `kubectl describe pod` Events first for troubleshooting
6. `nodeSelector` is suitable for basic scenarios, complex scenarios should use `nodeAffinity`

## Common Command Quick Reference

    kubectl get nodes
    kubectl get nodes --show-labels
    kubectl describe node k8s-node1
    kubectl label node k8s-node1 disktype=ssd
    kubectl label node k8s-node1 disktype=hdd --overwrite
    kubectl label node k8s-node1 disktype-
    kubectl apply -f pod-nodeselector.yaml
    kubectl get pod -o wide
    kubectl describe pod nginx-ssd

## One-Sentence Summary

`nodeSelector` is the most fundamental node scheduling method in Kubernetes, essentially telling the scheduler: **This Pod should go to these nodes and should not go to these nodes.**

## Tags
#Kubernetes #ApplyDeployment #SchedulePolicy #nodeSelector #NodeTab #PodMovement

## Operations Extension Understanding
- Kubernetes Official Documentation: Assign Pods to Nodes  
  https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
- Kubernetes Official Documentation: Labels and Selectors  
  https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
- Kubernetes Official Documentation: kubectl label  
  https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#label

## Next Day Plan
- Learn [[02- Node Affinity Basics - nodeAffinity and More Flexible Node Selection]]
- Compare the capability differences between `nodeSelector` and `nodeAffinity`
- Understand the scheduling semantics of "must satisfy" vs "try to satisfy"