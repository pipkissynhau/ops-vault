# Introduction to 01-nodeSelector: Scheduling Pods to Specific Nodes

## Documentation Overview
- Location: Kubernetes Scheduling Strategy Basics
- Applicable Phase: 07 - Application Deployment / 09 - Scheduling Strategies and Workload Placement
- Learning Objectives:
  - Understand why Pods need to be scheduled on specific nodes.
  - Grasp the fundamental relationship between node labels and nodeSelector.
  - Be able to create basic YAML files for scheduling based on node labels.
  - Know how to troubleshoot Pod Pending issues caused by mismatched node selection criteria.

## Core Concepts

In Kubernetes, Pods are typically automatically assigned nodes by the scheduler.  
However, in real-world scenarios, not all Pods can run on any node.  

Common use cases include:
- Certain business Pods must run on SSD-equipped nodes.
- GPU-powered applications require nodes with GPUs.
- Logs, monitoring, and caching components should be placed on dedicated nodes.
- Testing and production services need to be physically separated.
- Some middleware requires specific hardware configurations.

In such cases, it’s necessary to provide explicit constraints to the scheduler to prevent random assignments.  
The simplest way to achieve this is through `nodeSelector`.

## What are Node Labels?

Node labels are essentially key-value pairs associated with Kubernetes nodes, used to describe their characteristics.  
Examples include:
- `disktype=ssd`
- `env=prod`
- `zone=az1`
- `gpu=true`
- `workload=middleware`

You can think of them as tags attached to nodes, indicating:
- This machine has an SSD.
- It belongs to the production environment.
- It is located in a specific zone.
- It supports GPUs.
- It is designed for running certain types of workloads.

When scheduling Pods, these labels help determine where they should be placed.

## What is nodeSelector?

`nodeSelector` is a field in a Pod’s specification that specifies:  
**This Pod can only be scheduled on nodes with the specified labels.**

Its key features are:
- Simple and easy to understand.
- Suitable for beginners.
- Requires exact matches only.
- Does not support complex expressions.

Therefore, it makes an excellent starting point for learning Kubernetes scheduling.

## How nodeSelector Works

The process is as follows:
1. Labels are applied to nodes.
2. The Pod specifies `nodeSelector` in its YAML file.
3. The scheduler checks all nodes during scheduling.
4. Only nodes that meet the label criteria are eligible to host the Pod.
5. If no node meets the conditions, the Pod remains in a Pending state.

In other words, `nodeSelector` filters available nodes based on their labels.

## Viewing Node Labels

To list all nodes in the cluster:
    kubectl get nodes

To view labels of all nodes:
    kubectl get nodes --show-labels

For detailed information about a specific node:
    kubectl describe node <node-name>

For example:
    kubectl describe node k8s-node1
Pay special attention to the `Labels` section in the output.

## Applying Labels to Nodes

To add an SSD label to the `k8s-node1` node:
    kubectl label node k8s-node1 disktype=ssd

To add a business-related label:
    kubectl label node k8s-node1 workload=frontend

To check the labels:
    kubectl get node k8s-node1 --show-labels

## Removing Labels

To remove a label, use the `-` operator after the key:
    kubectl label node k8s-node1 disktype-

## First Example of nodeSelector

Here’s a basic Pod that requires scheduling only on nodes with `disktype=ssd` labels:

```yaml
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
```

Save this as `pod-nodeselector.yaml` and apply it:
    kubectl apply -f pod-nodeselector.yaml

To check which node the Pod is running on:
    kubectl get pod -o wide

If successful, you will see that the Pod has been assigned to a node with the required label.

## Verifying Pod Placement

View detailed information about the Pod:
    kubectl describe pod nginx-ssd
Pay attention to the `Node` and `Events` sections.  
For example, if it shows `Node: k8s-node1/192.168.1.21`, then the Pod is running on `k8s-node1`.

## Practice Scenario 1: Scheduling a Pod to a Specific Business Node

### Step 1: Apply Labels to the Node

Assume you want to place frontend services on `k8s-node2`:
   However, at the current introductory stage, it is sufficient to thoroughly understand the `nodeSelector` itself.

## Matching Characteristics of `nodeSelector`

`nodeSelector` operates on an "all must be met" logic.

For example:

    nodeSelector:
      env: prod
      disktype: ssd

This means that a node must satisfy both conditions:

- `env=prod`
- `disktype=ssd`

It is not enough to meet just one of them; both must be fulfilled.

## Example with Multiple Conditions

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

Only nodes that possess both of these labels will receive this Pod.

## Common Use Cases for `nodeSelector`

### 1. Scheduling Based on Hardware Capabilities

For example:

- SSD nodes
- GPU nodes
- Nodes with large amounts of memory

### 2. Environmental Isolation

For example:

- `env=test`
- `env=prod`

### 3. Node Purpose-Based Isolation

For example:

- `workload=logging`
- `workload=middleware`
- `workload=frontend`

### 4. Simple Node Selection in Small-Scale Clusters

`nodeSelector` is particularly useful in experimental, testing, and small-scale business environments.

## Limitations of `nodeSelector`

Despite its simplicity, it has clear limitations:

### 1. Only Exact Matching

It can only specify that a node "must have this label."

It cannot express:

- Preferencing certain types of nodes
- Meeting either condition A or B
- Avoiding certain types of nodes

### 2. No Support for Complex Expressions

For more complex scheduling scenarios, `nodeAffinity` is typically used.

### 3. Limited Flexibility

As scheduling rules become more complex, `nodeSelector` often becomes insufficient.

Therefore, the recommended learning sequence is:

- First learn `nodeSelector`
- Then learn `nodeAffinity`

## Difference Between `nodeSelector` and `nodeName`

These two concepts are easily confused.

### `nodeSelector`

It selects nodes based on labels.

For example:

    nodeSelector:
      disktype: ssd

This means that any node that meets this condition can be selected.

### ` nodeName`

It directly specifies a particular node.

For example:

    spec:
      nodeName: k8s-node1

This means that the Pod is forcibly assigned to that specific node.

The difference is that:

- `nodeSelector` is more flexible and suitable for daily use
- `nodeName` binds to a specific machine and is usually not used as a regular scheduling method

## Experimental Suggestions

You can conduct a simple experiment to observe the effects of `nodeSelector`.

### Experimental Objective

Ensure that a Pod is only allocated to nodes with the specified label.

### Experimental Steps

#### 1. List Nodes

    kubectl get nodes

#### 2. Label a Node

    kubectl label node k8s-node1 disktype=ssd

#### 3. Create a Test Pod

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

#### 4. Apply the YAML Configuration

    kubectl apply -f test-node-selector.yaml

#### 5. Check Pod Scheduling Results

    kubectl get pod -o wide

#### 6. Remove the Label and Re-run the Test

    kubectl delete pod test-node-selector
    kubectl label node k8s-node1 disktype-
    kubectl apply -f test-node-selector.yaml
    kubectl get pod
    kubectl describe pod test-node-selector

Through this experiment, you can clearly see:

- When a node has the matching label, the Pod is scheduled successfully.
- When a node does not have the matching label, the Pod remains in the Pending state.

## Additional Insights for Production Use

In a production environment, node labels should not be assigned arbitrarily but should be part of a unified planning process.

Common label dimensions include:

- Hardware capabilities: `disktype=ssd`
- Environment types: `env=prod`
- Availability zones: `topology.kubernetes.io/zone=az1`
- Node roles: `node-role=middleware`
- Special capabilities: `gpu=true`

If the label system is disorganized, it can lead to:

- Difficulty in maintaining YAML configurations
- Inconsistent scheduling rules
- Increased troubleshooting complexity
- Higher costs for team collaboration

Therefore, at a higher level, scheduling is not just about how YAML is written but also an integral part of node planning and resource allocation strategies