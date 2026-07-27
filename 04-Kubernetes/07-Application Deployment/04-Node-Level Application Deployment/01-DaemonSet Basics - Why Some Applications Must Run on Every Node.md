# 01-DaemonSet Basics: Why Some Applications Must Run on Every Node

## Document Description
- Documentation Location: Introduction to the Core Mechanisms of DaemonSet
- Applicable Stage: After summarizing stateful application deployment, move onto node-level application deployment
- Recommended Path: `04-Kubernetes/07-Application Deployment/04-Node-Level Application Deployment/01-DaemonSet Basics: Why Some Applications Must Run on Every Node`

## Tags
#Kubernetes #DaemonSet #node-exporter #node-level applications #monitoring #Prometheus #NodeExporter #application deployment #cloud-native #ops

---

## I. Why Learn DaemonSet Now

The main focus of application deployment previously was on two types of scenarios:

- Stateless business applications
- Stateful middleware and databases

These two types have one thing in common:

- They usually focus on whether a specific service is available.
- They often use the concept of "replica count" as their core approach.
- They are more commonly deployed using Deployment or StatefulSet.

However, in real clusters, there are also applications that do not fall into the category of "service replicas" or "database instances." Instead, they are:

> **system components that need to run on every node and distribute according to the node layout.**

Examples of such components include:

- node-exporter
- log collectors
- CNI-related components
- node agents in storage plugins
- security agents
- NodeLocal DNSCache, etc.

In these scenarios, it is no longer appropriate to think in terms of "running multiple replicas of a service." Instead, DaemonSet is a better choice.

---

## II. What is DaemonSet

DaemonSet is a controller in Kubernetes specifically designed for "node-distributed applications."

Its most fundamental role can be summarized as:

> **Ensuring that a specified Pod runs on eligible nodes.**

Here, the key focus is not on "replica count" but on "node coverage."

In other words, DaemonSet is more concerned with:

- Which nodes need to run this component.
- Whether each node has an instance of it.
- Whether new instances are automatically added when a new node is added.
- Whether instances are automatically removed when a node is deleted.

### A Basic Understanding
Deployment is more like:

> Determining how many replicas are needed

StatefulSet is more like:

> Ensuring that there are several identity-based instances

DaemonSet is more like:

> Ensuring that every node has at least one instance

---

## III. Why Some Applications Must Run on Every Node

This is the first key point to understanding DaemonSet.

The responsibilities of certain components determine that they must "run closely tied to the nodes."

### Typical Reasons Include

#### 1. Need to Collect Local Node Information
For example:
- CPU usage
- Memory usage
- File system information
- Network devices
- System load

If these are not collected directly on the node, it will be very difficult to obtain this information.

#### 2. Need to Access Host Machine Files or Directories
For example:
- Host machine log directories
- Host machine proc/sys
- Container runtime directories
- Local plugin directories

#### 3. Need to Perform the Same Tasks on Every Node
For example:
- Metric collection
- Log collection
- Security checks
- Network proxy functions
- Storage mount proxy services

#### 4. Automatic Expansion When the Number of Nodes Changes
If a new node is added to the cluster, such components should typically also automatically start running on the new node.

### Key Points forOps Professionals
What these components care about is not "how many service replicas there are in total," but:

> **Whether each node has the capability to perform these tasks.**

---

## IV. Why node-exporter Is a Natural Example for Understanding DaemonSet

node-exporter is a very typical example of a node-level application.

### What Is Its Purpose?
The main purpose of node-exporter is to expose node-level monitoring metrics, such as:

- CPU usage
- Memory usage
- File system capacity
- Disk I/O
- Network interface metrics
- System load
- System runtime

### Why It Is Suitable for DaemonSet
Because node-exporter collects "node-specific" metrics, not metrics from a particular business Pod.

This means that:

- Every node needs to have its own node-exporter.
- When a new node is added, a node-exporter should also be automatically started on it.
- Prometheus ultimately retrieves the metrics exposed by "each individual node."

### An Intuitive Understanding
If a cluster has 5 nodes, a more reasonable goal would not be:

- Randomly running 2 copies of node-exporter

but rather:

- Running one node-exporter on each of the 5 nodes.

### Key Points forOps Professionals
The deployment goal of node-exporter is not "ensuring that service replicas are available," but rather:

> **Ensuring### Summary of This Chapter

DaemonSet is a crucial controller in Kubernetes that addresses the deployment of components that require coverage across all nodes rather than managing a specific number of replicas. Taking node-exporter as an example, this controller ensures that there is one instance running on each node, enabling it to collect metrics and logs from the host perspective. This makes DaemonSet particularly useful for node-level monitoring, logging, security, and other purposes.

### Key Takeaways

1. **Node Coverage**: DaemonSet focuses on ensuring that every node has an instance of the component, not just managing a certain number of replicas.
2. **Host-Integrated Functions**: Components like node-exporter often rely on `hostPath` to access host resources such as `/proc`, `/sys`, and the root directory.
3. **Close Integration with Host Network**: Using `hostNetwork` allows the Pod to use the same network as the host, which is beneficial for monitoring systems like Prometheus.
4. **Role of Service**: While DaemonSet manages the instances on nodes, a Service is often used to provide a unified access point for these components, especially for integration with monitoring tools.

### Practical Applications

Understanding these concepts helps in making informed decisions when choosing between different controllers and components in Kubernetes, ensuring that applications are deployed efficiently and effectively.[[Differences between Node-Level Applications and Ordinary Business Applications: Comparison of Deployment, StatefulSet, and DaemonSet]]