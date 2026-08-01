# 01-DaemonSet Basics: Why Some Applications Must Run on Every Node

## Document Notes
- Document Location: DaemonSet Core Mechanism Introduction
- Applicable Stage: After completing the summary of stateful application deployment, enter the mainline of node-level application deployment
- Recommended Path: `04-Kubernetes/07-Apply deployment/04-Application deployment at node level/01-DaemonSet Basis: Why do some applications have to run every node`

## Tags
#Kubernetes #DaemonSet #node-exporter #ApplicationAtNodeLevel #Monitor #Prometheus #NodeExporter #ApplyDeployment #Clouds. #Transport

---

## I. Why Learn DaemonSet Now

The previous application deployment mainline primarily focused on two scenarios:

- Stateless business applications
- Stateful middleware and databases

These two types of objects share the following characteristics:

- Typically focus on whether a business service is available
- Typically use "replica count" as the core concept
- More commonly use Deployment or StatefulSet

However, in real clusters, there's another type of application that doesn't belong to "business replicas" or "database instances," but rather:

> **Needs to follow node distribution and run one instance on each node.**

These components include:

- node-exporter
- Log collector
- CNI-related components
- Node agent in storage plugins
- Security Agent
- NodeLocal DNSCache, etc

This scenario is no longer suitable for understanding with the "start several replicas for a service" approach, but rather better suited for DaemonSet.

---

## II. What is DaemonSet

DaemonSet is a controller specifically designed for "node-distributed applications" in Kubernetes.

Its most core function can be summarized as:

> **Ensure that a specified Pod runs on every node that meets the conditions.**

The most important aspect here isn't "replica count," but "node coverage."

In other words, DaemonSet is more concerned with:

- Which nodes need to run this component
- Whether each node has the corresponding instance
- Whether instances are automatically added when new nodes are added
- Whether instances are automatically removed when nodes are deleted

### A Basic Understanding
Deployment is more like:

> Need several replicas

StatefulSet is more like:

> Need several instances with identities

DaemonSet is more like:

> Every node must have an instance

---

## III. Why Some Applications Must Run on Every Node

This is the first key point in understanding DaemonSet.

Because some components' responsibilities determine they must "run closely with nodes."

### Typical Reasons Include

#### 1. Need to collect node-local information
For example:
- CPU
- Memory
- File system
- Network devices
- System load

If not running on the node itself, it's difficult to directly collect these information.

#### 2. Need to access host files or directories
For example:
- Host log directory
- Host proc/sys
- Container runtime directory
- Local plugin directory

#### 3. Need to perform the same responsibilities on each node
For example:
- Metric collection
- Log collection
- Security checks
- Network proxy
- Storage mount proxy

#### 4. Need to automatically expand coverage when node count changes
If a new node is added to the cluster, such components should typically automatically run an instance on the new node.

### Operations Understanding Focus
These components care not about "how many service replicas there are," but rather:

> **Whether each node has the capability.**

---

## IV. Why node-exporter is a Natural Example for Understanding DaemonSet

node-exporter is a very typical node-level application.

### What is its purpose
node-exporter is mainly used to expose node-level monitoring metrics, such as:

- CPU usage
- Memory usage
- File system capacity
- Disk IO
- Network interface metrics
- System load
- System uptime

### Why it's suitable for DaemonSet
Because node-exporter collects "node itself" metrics, not metrics of a specific business Pod.

This means:

- Each node needs to have a node-exporter
- A new node added should automatically start a node-exporter
- Prometheus ultimately scrapes "metrics exposed by each node individually"

### An Intuitive Understanding
If the cluster has 5 nodes, a more reasonable goal is not:

- Randomly run 2 node-exporter replicas

But rather:

- Run 1 node-exporter on each of the 5 nodes

### Operations Understanding Focus
The deployment goal of node-exporter is not "service replica availability," but:

> **Each node must have the capability to collect metrics.**

---

## V. Why Deployment is Unsuitable for node-exporter Scenarios

From the "can it run" perspective, Deployment can certainly pull up the node-exporter container.  
But from the "workload model" perspective, it's not suitable.

### Deployment's Issues

#### 1. Only guarantees replica count, not node coverage
For example:

    replicas: 3

Only indicates a total of 3 Pods are running, but doesn't mean they'll be distributed across all nodes.

#### 2. New nodes won't automatically get an instance
If a new node is added, Deployment won't automatically start a node-exporter because of the "new node appears."

#### 3. Difficult to express "one per node"
Deployment is better at expressing:
- Total count
- How to do rolling updates

But not good at expressing:
- Every node must have one

### Issues for node-exporter
If only running on some nodes, it means:

- Some nodes have monitoring data
- Some nodes don't have monitoring data
- Prometheus scraping results are incomplete
- Node coverage is uneven

### Operations Understanding Focus
Deployment is more suitable for "replica-count-based workloads,"  
DaemonSet is more suitable for "node-coverage-based workloads."

---

## VI. What is the Core Idea of DaemonSet

DaemonSet's most core scheduling idea can be summarized as:

> **Ensure every node that meets the conditions has a corresponding Pod.**

### What Does This Mean Specifically

#### 1. When a new node is added
If a new node is added and it meets the scheduling conditions, DaemonSet will typically automatically create a Pod on this new node.

#### 2. When a node is deleted
If a node is removed from the cluster, the corresponding Pod will also disappear.

#### 3. When nodes are filtered
If node range is limited through `nodeSelector`, taint tolerance, or affinity restrictions, DaemonSet will only cover nodes that meet the conditions.

### A Simple Example
Assume the cluster has 3 nodes:

- node-1
- node-2
- node-3

After deploying a node-exporter DaemonSet, the expectation is usually:

- node-1 has 1 node-exporter
- node-2 has 1 node-exporter
- node-3 has 1 node-exporter

If a new `node-4` is added and meets scheduling conditions, it should automatically add:

- node-4 has 1 node-exporter

---

## VII. Let's Look at a Minimal DaemonSet Example

# A Minimal Teaching Example to Understand Structure

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
                  readOnly: true
          volumes:
            - name: proc
              hostPath:
                path: /proc
            - name: sys
              hostPath:
                path: /sys
            - name: root
              hostPath:
                path: /

This YAML is not a production-ready complete solution, but it's ideal for understanding how DaemonSet and node-exporter work together.

---

## VIII. Overview of What This YAML Represents

This YAML contains four key layers of information.

### 1. `kind: DaemonSet`
This indicates it's not a Deployment or StatefulSet, but a DaemonSet.

This means the target isn't a fixed number of replicas, but node coverage.

### 2. `selector` and `template.labels`
These two parts express:

- How DaemonSet identifies the Pods it manages
- What labels the Pod carries

For example:

    selector:
      matchLabels:
        app: node-exporter

and:

    labels:
      app: node-exporter

must remain consistent.

### 3. `hostPath`
Here, the following host directories are mounted:

- `/proc`
- `/sys`
- `/`

This shows node-exporter isn't just reading container internals, but accessing data from the host perspective.

### 4. `hostNetwork` / `hostPID`
This indicates the Pod runs closer to the host environment, aiding in collecting node-level metrics.

### Operational Understanding Focus
The essence of this YAML isn't "starting a monitoring container," but:

> **Deploying an agent that collects node-level metrics on each node.**

---

## IX. Why node-exporter Mounts `/proc`, `/sys`, and the Root Directory

This is the most natural part of the node-exporter example and key to understanding "node-level applications."

### 1. `/proc`
The host's `/proc` provides extensive information about processes, CPU, memory, kernel, etc.

### 2. `/sys`
The host's `/sys` provides information about hardware, kernel devices, file systems, etc.

### 3. `/`
Mounting the host root directory typically allows node-exporter to read file system information from the host perspective.

### Why These Directories Are Mounted
Because node-exporter focuses on:

- The node's actual state
- Host resources
- Node file systems
- Node network and system metrics

Rather than the container's own environment.

### Operational Understanding Focus
Without mounting these host-related directories, node-exporter would struggle to accurately reflect the node's state.

---

## X. Why node-exporter Often Requires `hostNetwork`

In many deployment scenarios, node-exporter commonly enables:

    hostNetwork: true

### Common Reasons for This
- Making the Pod's network closer to the node's network
- Facilitating Prometheus to scrape port 9100 by node address
- Avoiding additional complexity in certain network environments

### Key Understanding
The focus isn't "memorizing hostNetwork," but understanding:

> **Node-level agents often need closer proximity to the host's network environment.**

### Notes
Once using `hostNetwork`, be aware of:
- Port conflict risks
- Network exposure surface
- Security boundaries

---

## XI. Why node-exporter Often Works with Service

Although node-exporter is a node-level application, Prometheus still needs a way to discover and scrape it.

### Common Practice
A Service is typically created for node-exporter, for example:

apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    app: node-exporter
  ports:
    - name: metrics
      port: 9100
      targetPort: 9100

### What Does This Mean
DaemonSet is responsible for "one instance per node",  
Service is responsible for "providing a unified discovery entry or working with monitoring systems to scrape".

### But This Is Different From MySQL's Service
MySQL's Service focuses on:
- Business access entry

node-exporter's Service focuses more on:
- Monitoring system discovery targets
- Unified exposure of metrics ports
- Convenience for Prometheus configuration

### Operational Understanding Focus
Although both are Service, their roles can differ in different workload models.

---

## Twelve, How to Remember the Differences Between DaemonSet, StatefulSet, and Deployment

You can use the following table to quickly get a sense:

| Controller | What It Focuses On | Common Scenarios |
|---|---|---|
| Deployment | Number of replicas | Web services, API services, frontend services |
| StatefulSet | Identity, order, volume relationships | MySQL, PostgreSQL, Kafka, ZooKeeper |
| DaemonSet | Node coverage | node-exporter, log collectors, security agents |

Another more straightforward way of saying it:

### Deployment
Focuses on:

> How many total instances

### StatefulSet
Focuses on:

> Who are these instances

### DaemonSet
Focuses on:

> Which nodes must have one

---

## Thirteen, What Types of Components Are DaemonSet Most Suitable For

At this stage, we can divide suitable components for DaemonSet into several categories.

### 1. Node Monitoring
Examples:
- node-exporter

### 2. Log Collection
Examples:
- Fluent Bit
- Filebeat

These components typically read container logs or system logs on nodes.

### 3. Security and Audit
Examples:
- Node security agent
- Host audit agent

### 4. Network and Storage Node Agents
Examples:
- Some CNI/CSI node-side components
- NodeLocal DNSCache

### A Common Feature
These components are typically not "oriented toward a specific business service", but rather:

> **Provide capabilities to the node itself.**

---

## Fourteen, When Shouldn't You Use DaemonSet

Understanding controllers also requires knowing their boundaries.

### Common Situations Where DaemonSet Is Not Suitable

#### 1. Just Ordinary Business Services
Examples:
- Java API
- Nginx
- Frontend services

These are more suitable for Deployment.

#### 2. Business Components That Require Stable Identity and Independent Storage
Examples:
- MySQL
- PostgreSQL
- Kafka
- ZooKeeper

These are more suitable for StatefulSet.

#### 3. When Not Every Node Needs Coverage
If the application isn't "required on every node", you shouldn't force-fit DaemonSet just for "appearing uniform".

### Operational Understanding Focus
DaemonSet's value lies in "node coverage", not "more advanced scheduling".

---

## Fifteen, Basic Viewing Command Examples

After deploying DaemonSet, common viewing methods are as follows.

### 1. View DaemonSet

    kubectl get ds -n monitoring
    kubectl describe ds node-exporter -n monitoring

### 2. View Pod Distribution

    kubectl get pod -n monitoring -o wide

Here, focus on:

- Whether each node has a node-exporter Pod
- Whether Pods are distributed on expected nodes

### 3. View Service

    kubectl get svc -n monitoring
    kubectl describe svc node-exporter -n monitoring

### 4. View Logs

    kubectl logs -n monitoring <node-exporter-pod-name>

---

## Sixteen, The Most Important Concepts in This Article

### 1. DaemonSet Focuses on Node Coverage, Not Replica Count
This is the first key understanding.

### 2. node-exporter Is a Natural Example for Understanding DaemonSet
Because it naturally requires "one instance per node".

### 3. Node-Level Applications Are Typically Closer to the Host
Therefore, they are more commonly:
- hostPath
- hostNetwork
- Node-level resource access

### 4. DaemonSet Is Suitable for Node Agent Components, Not Ordinary Business Services
This is the fourth key understanding.

### 5. Switching from Stateful Applications to DaemonSet Requires Understanding Workload Model Changes
Rather than just remembering the new object name.

---

## Seventeen, Stage Summary

DaemonSet is a very important type of controller in Kubernetes, solving not the "business replica management" problem, but the "node coverage component deployment" problem.

Using node-exporter as an example, we can clearly see DaemonSet's several core features:

- One instance per node
- Collects metrics from the host perspective
- Usually requires mounting host directories
- Commonly used for monitoring, logging, security, and other node-level capability construction

Therefore, understanding DaemonSet's key is not to treat it as "another replica controller", but to understand:

> **Some applications are not distributed around business services, but around nodes.**

---

## Eighteen, Keyword Quick Notes

- DaemonSet: Node-level coverage controller
- node-exporter: Typical node monitoring collection component
- Node-level application: Component running on a node
- hostPath: Mount host directory
- hostNetwork: Use host network
- Node coverage: One instance per node
- Node agent: Pod that represents a node to provide certain capabilities

---

## Nineteen, Operational Extended Understanding

Switching from MySQL to node-exporter is an important turning point in learning Kubernetes application deployment.

MySQL represents:
- Business middleware
- Data and state
- Instance relationships
- Service access

node-exporter represents:
- Node agent
- Node observation
- Host capability collection
- Running on a node

This indicates that Kubernetes application deployment is not a single model, but at least includes:
- Replica count-based
- State relationship-based
- Node coverage-based /think

Separate these three types of models for clearer understanding. When studying node-level components like the log collector, CNI, CSI, and security Agent later, yourThinking. will be more organized.

---

## References
- Kubernetes DaemonSet official documentation
- node-exporter official documentation
- Prometheus official documentation

---

## Next Day's Suggestion
The next post suggests organizing:

[[02 Difference Between Node-Level and Regular Business Applications Deployment StatefulSet DaemonSet Comparison]]