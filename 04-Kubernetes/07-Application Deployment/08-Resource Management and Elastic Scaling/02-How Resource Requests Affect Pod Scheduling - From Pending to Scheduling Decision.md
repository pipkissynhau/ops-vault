# 02-How Resource Requests Affect Pod Scheduling: From Pending to Scheduling Decision

## Document Description
- Document Location: Basics of Resource-Driven Scheduling
- Applicable Phase: After understanding the basic concepts of requests/limits, move on to how resource requests affect Pod scheduling and troubleshooting Pending status
- Recommended Path: `04-Kubernetes/07-Application Deployment/08-Resource Management and Auto Scaling/02-How Resource Requests Affect Pod Scheduling: From Pending to Scheduling Decision`

## Tags
#Kubernetes #Scheduler #Pending #requests #limits #Resource Scheduling #Pod Scheduling #Resource Management #Application Deployment #Cloud-Native #Ops #Interview Notes

---

## I. Why Understand “How Resource Requests Affect Scheduling” Separately

Many people tend to confuse these concepts when first learning Kubernetes:

- How much resources the container actually uses
- How many `requests` are specified in the YAML file
- How many `limits` are specified in the YAML file
- How much resources are still available on the node
- Why a Pod remains in the `Pending` state

However, in Kubernetes, **the scheduler’s primary focus is not on how much the container “will actually use”, but on the `requests` declared by the Pod.**

This is also why common issues occur in practical environments:

- A node appears to have idle CPU, but the Pod still cannot be scheduled
- A Pod remains in the `Pending` state indefinitely
- HPA expands new replicas, but the new Pods fail to start
- An application fails at the scheduling stage even though it hasn’t started running yet

Therefore, the core goal of this section is to establish a key understanding:

> **Whether a Pod can be scheduled to a node depends mainly on the resource requests, not the limits or the current instantaneous resource usage.**

---

## II. What Is “Resource-Driven Scheduling”

The “resource-driven scheduling” discussed here does not refer to:

- `nodeSelector`
- `nodeAffinity`
- `taints / tolerations`
- `podAffinity / podAntiAffinity`

These are examples of “policy-based scheduling”.

Instead, this section focuses on another main aspect:

> **How the scheduler determines which node to place a Pod on based on the resource requests declared by the Pod.**

In other words, it addresses questions such as:

- How much CPU does this Pod request?
- How much memory does this Pod request?
- Can a certain node accommodate this Pod?
- If not, why does the Pod remain in the `Pending` state?

This represents the most direct connection between resource management and scheduling decisions.

---

## III. Why Does the Scheduler Primarily Consider Requests, Not Limits

This is the most important point in this section.

### 1. `Requests` Are the Basis for the Scheduler’s Resource Placement Decision

Before scheduling a Pod, the scheduler first checks the Pod’s resource requirements, which are specified by:

- `resources.requests.cpu`
- `resources_requests.memory`

Then, it determines how much **available resources** are left on the node.

If the available resources on the node cannot meet the Pod’s requests, the node will not be considered as a candidate for scheduling.

---

### 2. `Limits` Are Mainly Not Designed for the Scheduler

The primary purpose of `limits` is to enforce runtime resource constraints, such as:

- The maximum amount of CPU that can be used
- The maximum amount of memory that can be allocated
- The risk of OOMKilling if the memory limit is exceeded

Therefore, `limits` are more related to:

> **What a container can use during operation**, rather than:

> **Whether a Pod can be scheduled to a node before it starts running.**

---

### 3. Why Is Kubernetes Designed This Way

This design ensures that the scheduler has a “predictable, declarative, and calculable” resource requirement value.

The actual usage of resources presents several challenges:

- The scheduler cannot know the future real resource usage before the application starts
- Actual usage is dynamic and not suitable for static scheduling decisions
- Resource usage varies significantly over time and cannot be used directly as a basis for scheduling

Therefore, Kubernetes uses `requests` to specify “at least how much resources should be reserved”, allowing the scheduler to make placement decisions based on this value.

---

## IV. What Happens Between Creating a Pod and Successfully Scheduling It

From an operations perspective, the process of submitting a Pod from creation until it successfully lands on a node generally involves the following steps:

### 1. User Submits a Pod / Deployment / StatefulSet

For example, executing:

    kubectl apply -f deployment.yaml

---

### 2. The Controller Creates the Pod Object

For instance, a Deployment first creates a ReplicaSet, which then generates Pods.

---

### 3. The Scheduler Notices That the Pod Has Not Been Bound to a Node

Such### It's Important to Distinguish Several Concepts

#### 1. Total Node Resources
How much CPU and memory the machine has in total.

#### 2. Allocatable Resources
The resources that can actually be used by Pods; not all machine resources are available for running Pods.

#### 3. Allocated Requests
Resources that have already been “reserved” by existing Pods.

#### 4. Current Instantaneous Utilization
The actual usage of CPU/memory at a particular moment.

### Key Points for Operations and Maintenance Professionals to Understand

Even if a node's current CPU utilization is low, it may still be unable to host new Pods due to:

- Many Pods having already reserved resources.
- The scheduler considering these resources as “reserved”.

This is a typical scenario in resource scheduling:

> **The machine seems idle, but the scheduler believes it cannot accommodate more Pods.**

---

## Example of a Basic Resource Request

Here’s an example of a basic Deployment YAML file:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-web
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx-web
      template:
        metadata:
          labels:
            app: nginx-web
        spec:
          containers:
            - name: nginx-web
              image: nginx:1.27
              ports:
                - containerPort: 80
              resources:
                requests:
                  cpu: "100m"
                  memory: "128Mi"
                limits:
                  cpu: "500m"
                  memory: "256Mi"

### How to Understand This YAML File

#### `requests`
Indicates that during scheduling, at least the following resources should be reserved for this container:

- 100 million CPU cycles
- 128 MiB of memory

#### `limits`
Indicate the maximum amount of these resources that the container is allowed to use during runtime:

- 500 million CPU cycles
- 256 MiB of memory

### Key Points for Operations and Maintenance Professionals to Understand

If a node does not have even 100 million CPU cycles or 128 MiB of memory available, this Pod may fail during the scheduling process.

---

## Example of a Pod Pending Due to Insufficient Resources

Suppose a testing node has limited resources, and you try to create a Pod with the following configuration:

    apiVersion: v1
    kind: Pod
    metadata:
      name: big-memory-pod
    spec:
      containers:
        - name: app
          image: nginx:1.27
          resources:
            requests:
              cpu: "2"
              memory: "4Gi"
            limits:
              cpu: "2"
              memory: "4Gi"

If no node in the cluster has enough resources to meet these requirements (2 CPU cores and 4 GiB of memory), this Pod will likely remain in the **Pending** state.

### Common Phenomena

When you execute `kubectl get pod`, you might see:

    NAME             READY   STATUS    RESTARTS   AGE
    big-memory-pod   0/1     Pending   0          2m

At this point, many people might assume that there are issues with the image download, container startup, or the application itself. However, in reality, these problems have not yet occurred.

---

## How to Determine If the Issue Is Related to Resource Scheduling Using `describe` and `events`

This is the most important step in troubleshooting such issues.

When a Pod is in the **Pending** state, first check:

    `kubectl describe pod <pod-name>`

Pay special attention to:

### 1. `Events`
If it’s a resource scheduling issue, you are likely to see messages like:

- `Insufficient cpu`
- `Insufficient memory`
- `0/3 nodes are available`
- `node(s) didn't have enough memory`
- `node(s) didn't have enough cpu`

### 2. `Conditions`
You might also see:

- `PodScheduled=False`

This indicates that the Pod has not been successfully assigned to any node.

### Key Points for Operations and Maintenance Professionals to Understand

Whenever you encounter such events, first consider whether the `requests` specified are too high or whether the node’s resources are insufficient. Don’t immediately assume there are problems with the image, application, or configuration files.

---

## The Minimum Cycle of Resource Requests, Pending Status, and Scheduling Decisions

You can summarize this process in one sentence:

> **After a Pod is created, the scheduler determines which node can accommodate it based on the specified requests. If no node meets these requirements, the Pod will remain in the Pending state.**

In other words, the process can be simplified as follows:

### 1. Submit the Pod
### 2. The scheduler checks the `requests`
### 3## Nineteen, Common Follow-up Questions in Interviews

Common questions in this area include:

- Does the Kubernetes scheduler primarily consider requests or limits?
- Why does a Pod remain in the Pending state?
- If a node has sufficient resources, why can't a Pod be scheduled?
- What is the difference between requests and limits?
- When a Pod scheduling fails, what should be checked first?
- What do errors like `Insufficient cpu` or `Insufficient memory` usually indicate?
- Why does a Pod expanded using HPA still remain in the Pending state?
- How can one understand the difference between allocatable resources and a node's total resources?

---

## Twenty, Summary of Key Points

How resource requests affect Pod scheduling is a crucial aspect of Kubernetes resource management. The most important thing here is not to memorize the detailed implementation of the scheduler, but to establish the following core concepts:

- When making resource allocation decisions, the scheduler first considers requests.
- Requests determine whether a Pod has the right to be placed on a certain node.
- Limits are more about the upper limit of resources during runtime.
- Often, when a Pod is in the Pending state, it's because the scheduling process failed, not because the program hasn't started yet.
- To diagnose resource-related scheduling issues, `describe` and `events` are the most useful tools.
- When evaluating node resources, one should consider not only the instantaneous usage rate but also the difference between allocatable resources and requested resources.

Once these concepts are clear, understanding subsequent topics such as QoS, Eviction, HPA, and resource troubleshooting will be much easier.

---

## Twenty-One, Quick Reference Terms

- requests: Resource requests, the primary basis for scheduling.
- limits: Resource upper limits, constraints during runtime.
- Pending: A Pod that has not yet been scheduled or started.
- scheduler: The component responsible for selecting nodes for Pods.
- allocatable: Resources on a node that are actually available for use by Pods.
- Insufficient cpu: Insufficient CPU resources to meet scheduling requirements.
- Insufficient memory: Insufficient memory resources to meet scheduling requirements.
- HPA: Horizontal auto-scaling.
- Cluster Autoscaler: Node auto-scaling mechanism.

---

## Twenty-Two, Operational Perspectives

From an operational standpoint, resource-driven scheduling marks the beginning of Kubernetes' transition from being able to create Pods to being able to place workloads efficiently. Many beginners focus on whether images can be pulled down, containers can start, and Services are accessible. However, in real-world scenarios, many issues arise much earlier:

- The Pod may not even have started running.
- The scheduler might already have determined that there are insufficient resources.
- The workload could get stuck in the Pending state from the moment it is created.

This shows why resource requests, although just a small section of YAML configuration, actually connect various aspects such as Pod scheduling, node capacity, resource planning, the effectiveness of HPA, the trigger conditions for Cluster Autoscaler, and subsequent resource troubleshooting. Therefore, while this topic appears to focus on requests, it actually helps build your foundational understanding of the relationship between resources, scheduling, and capacity in Kubernetes.

---

## References

- Kubernetes Resource Management for Pods and Containers
- Assign Memory Resources to Containers and Pods
- Assign CPU Resources to Containers and Pods
- Pod Scheduling Readiness
- Common output analysis using `kubectl describe pod` / `describe node`

---

## Next Steps

It is recommended to organize the following topic:

[[03-Why Does a Pod Remain in the Pending State: Troubleshooting Insufficient Resources and Scheduling Failures]]