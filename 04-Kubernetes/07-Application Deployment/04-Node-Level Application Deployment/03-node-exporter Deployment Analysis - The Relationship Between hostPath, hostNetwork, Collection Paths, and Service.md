# 03-node-exporter Deployment Analysis: The Relationship Between hostPath, hostNetwork, Collection Paths, and Service

## Document Description
- Document Focus: Detailed analysis of the node-level deployment of node-exporter in Kubernetes.
- Target Audience: Those who have already understood the basics of DaemonSet and the differences between Deployment, StatefulSet, and DaemonSet workloads, and wish to delve into the specific deployment of node-exporter.
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/04-Node-Level Application Deployment/03-node-exporter Deployment Analysis: The Relationship Between hostPath, hostNetwork, Collection Paths, and Service`

## Tags
#Kubernetes #DaemonSet #node-exporter #Prometheus #hostPath #hostNetwork #Monitoring #Node-Level Applications #Service #Application Deployment #Cloud-Native #Operations and Maintenance

---

## I. Why Analyze node-exporter Separately

node-exporter is one of the most typical and beginner-friendly examples in node-level applications.

In the previous section, we established the fundamental differences among the three types of workloads:

- Deployment focuses more on the number of replicas.
- StatefulSet emphasizes member relationships between instances.
- DaemonSet aims for node coverage across the cluster.

node-exporter provides a concrete example that illustrates how DaemonSet principles can be applied in practice.

The focus of this section is not to provide an exhaustive overview of the Prometheus ecosystem but to address several practical questions:

- Why is node-exporter typically deployed using DaemonSet?
- Why does it often need to access host directory paths?
- Why is `hostNetwork` frequently used in its deployment?
- Whose metrics does it actually collect?
- What role does Service play here?
- What are the key differences in deploying such components compared to regular business applications?

---

## II. What Exactly Is node-exporter

node-exporter is a commonly used component in the Prometheus ecosystem for collecting node-level metrics.

Its primary function is:

> **To expose the system metrics of the host node as metrics that Prometheus can retrieve.**

### Commonly Collected Metrics Include

- CPU usage
- Memory usage
- File system capacity
- Disk I/O statistics
- Network interface information
- System load
- Startup time
- Various kernel and system operation details

### What It Doesn't Mainly Do

node-exporter is not designed to:

- Collect business logs
- Gather application traces
- Collect custom business metrics defined by applications
- Analyze container application errors directly

Instead, its focus is on:

> **Collecting resource metrics at the host and node levels.**

---

## III. Why Is node-exporter Suitable for DaemonSet

The key reason lies not in industry best practices but in the nature of the component's role.

### 1. It Collects Node-Level Metrics
Since it collects data from the host node itself, each node requires its own collector.

### 2. Its Goal is Node Coverage, Not Replica Quantity
If a cluster has 6 nodes, the desired outcome is usually:

- One node-exporter instance on each of the 6 nodes

rather than:

- Only 2 or 3 instances running across the entire cluster.

### 3. New Nodes Are Automatically Added with Collectors
When a new node joins the cluster, DaemonSet automatically creates a Pod on that node, which aligns perfectly with node-exporter's purpose.

### 4. Collectors Are Removed When Nodes Are Deleted
If a node leaves the cluster, the corresponding node-exporter instance becomes irrelevant.

### Key Points for Operations and Maintenance
node-exporter is not a “replica of a monitoring service” but rather:

> **A metric proxy on each host node.**

---

## IV. A Simplified Example of node-exporter

Here’s a simplified example that still illustrates the basic structure:

### DaemonSet Example

    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: node-exporter
      namespace: monitoring
    spec:
      selector:
        matchLabels:
          app: node-exporter
      template:
        metadata:
          labels:
            app: node-exporter
        spec:
          hostNetwork: true
          hostPID: true
          tolerations:
            - operator: Exists
          containers:
            - name: node-exporter
              image: prom/node-exporter:v1.8.1
              ports:
                - containerPort: 9100
                  hostPort: 9100
              args:
                - --path.procfs=/host/proc
                - --path.sysfs=/host/sys
                - --path.rootfs=/host/root
              volumeMounts:
                - name: proc
                  mountPath: /host/proc
                  readOnly: true
                - name: sys
                  mountPath: /host/sys
                  readOnly: true
                - name: root
                  mountPath: /host/root
                 --path.procfs=/host/proc

This indicates telling the node-exporter:

- Not to read the container's own `/proc`
- Instead, to read the host machine's `/proc` that has been mounted.

### Key Points for Operations and Maintenance
Without this setting, the perspective collected by the node-exporter may be incomplete or biased towards the container itself, rather than the entire node.### 1. The purpose of node-exporter is not to provide business services but to collect node metrics.  
This is the first key understanding.

### 2. DaemonSet is suitable for node-exporter because it ensures one instance per node.  
This is the second key understanding.

### 3. The essence of hostPath is to allow containers to access data from the host machine's perspective.  
This is the third key understanding.

### 4. hostNetwork, hostPort, and hostPID reflect a runtime approach that is closer to that of the host machine.  
This is the fourth key understanding.

### 5. The Service associated with node-exporter serves more as an entry point for monitoring and data collection rather than a business service interface.  
This is the fifth key understanding.

---

## Twenty, Phase Summary

node-exporter provides a natural example for understanding how node-level applications are deployed. Through it, we can focus on several specific aspects of DaemonSet:

- Why it’s necessary to have one instance per node.
- Why it’s important to mount host directories.
- Why it’s crucial to use the host network.
- Why Service acts as an entry point for monitoring and data collection here.
- Why components like this shouldn’t be considered regular business replicas.

The goal of this discussion is to establish a broader understanding:

> **The focus of node-level applications is not on replicating business services but on collecting host capabilities and ensuring node coverage.**

---

## Twenty-One, Key Terms Summary

- **node-exporter**: A component for collecting node metrics.
- **DaemonSet**: A controller designed for node coverage.
- **hostPath**: Used to mount host directories into containers.
- `/proc`: Provides runtime information of the host machine.
- `/sys`: Contains system and device information of the host machine.
- `hostNetwork`: Utilizes the host network for communication.
- `hostPort`: Occupies a port on the host machine.
- **Service**: Serves as an entry point for monitoring and data collection.
- **Node coverage**: Ensures that there is one instance of the component per node.

---

## Twenty-Two, Extended Operations Understanding

From the perspective of workload models, node-exporter clearly demonstrates the difference between node-level applications and business applications. Business applications focus more on:

- Traffic volume.
- Number of replicas.
- Deployment strategies.

Stateful applications are more concerned with:

- Data storage.
- Identity management.
- Recovery mechanisms and membership relationships.

In contrast, node-level applications like node-exporter prioritize:

- Host visibility.
- Metric collection.
- Node coverage.
- Ensuring the component runs directly on the node.

Therefore, understanding how node-exporter is deployed lays a foundation for further studying related components such as:

- Fluent Bit / Filebeat.
- Security Agents.
- CNI / CSI node-side components.

---

## References
- Kubernetes DaemonSet official documentation.
- node-exporter official documentation.
- Prometheus official documentation.

---

## Next Steps
It is recommended to organize the following topic in the next article:

[[04-Why DaemonSets are Commonly Used for Log Collectors: The Node-Level Deployment Approach of Fluent Bit and Filebeat]]