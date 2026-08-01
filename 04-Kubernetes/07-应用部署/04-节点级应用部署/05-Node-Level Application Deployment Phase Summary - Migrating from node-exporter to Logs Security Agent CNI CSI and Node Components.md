# 05-Node-Level Application Deployment Phase Summary: Migrating from node-exporter to Log, Security Agent, CNI, and CSI Node Components

## Document Notes
- Document Purpose: Summary of node-level application deployment phase and migration methods
- Applicable Stage: After completing DaemonSet basics, workload model comparison, and node-exporter/log collector deployment strategies, enter the main summary of node-level application deployment
- Recommended Path: `04-Kubernetes/07-Apply deployment/04-Application deployment at node level/05-Summary of application deployment phase at node level: from node-exporter Move to log, secure AgentI don't know.CNI / CSI Node Component`

## Tags
#Kubernetes #DaemonSet #node-exporter #FluentBit #Filebeat #Clearagent #CNI #CSI #ApplicationAtNodeLevel #TaskLoadModel #hostPath #hostNetwork #ApplyDeployment #Clouds. #Transport

---

## I. What This Phase Has Accomplished

The node-level application deployment phase focuses not on learning the details of a specific component, but on establishing a new way to understand workloads.

The main achievements of this phase include:

- DaemonSet basics
- Comparison of three workload models: Deployment, StatefulSet, DaemonSet
- Dissection of node-exporter's node-level deployment
- Why log collectors commonly use DaemonSet

Through these contents, a relatively clear judgment can be established:

> **Some components aren't meant to provide business services or maintain a group of stateful members, but rather to provide some local capability on each node.**

These types of components are called node-level applications.

---

## II. Why node-exporter is a Natural Starting Point for Node-Level Applications

node-exporter is the most natural starting point for understanding node-level applications because its goals are very clear:

- Collect node-native metrics
- Expose metrics interface
- Provide metrics for Prometheus scraping
- Each node needs an instance

Through node-exporter, the core value of DaemonSet can be naturally understood:

### 1. It focuses more on node coverage rather than replica count
If the cluster has 6 nodes, the expectation is typically:

- 6 nodes each have 1 node-exporter instance

Rather than:

- Total of 2 or 3 replicas

### 2. It is closer to the host
Commonly involves:

- `/proc`
- `/sys`
- Host root directory
- `hostNetwork`
- `hostPort`

### 3. It is not a business service, but a node agent
Its responsibility is not to process business requests, but to represent the node and output monitoring data.

### Operations Understanding Focus
This example shows that deploying node-level applications first requires understanding whether:

> **The component's responsibilities are inherently bound to each node.**

---

## III. What Really Changes When Migrating from node-exporter to Log Collectors

Log collectors are the second most common case of node-level applications.

Common representatives include:

- Fluent Bit
- Filebeat

They, like node-exporter, often use DaemonSet, but with different collection targets.

### What node-exporter Collects
Mainly collects:

- CPU
- Memory
- File systems
- Network interfaces
- System load and other node metrics

### What Log Collectors Collect
Mainly collects:

- `/var/log/containers`
- `/var/log/pods`
- Host system logs
- Some application-local log files

### Commonalities
- Both are more suitable for DaemonSet
- Both typically require node coverage
- Both are closer to the host perspective
- Both are platform agent-type components

### Differences
- node-exporter outputs metrics
- Log collectors output logs
- node-exporter is more focused on reading system state
- Log collectors are more focused on reading log files and forwarding to log platforms

### Operations Understanding Focus
When migrating from node-exporter to log collectors, there's no need to relearn the entire DaemonSet model - just understand:

> **It's still a node agent, but the objects being read and output differ.**

---

## IV. What to Look At First When Migrating from node-exporter to Security Agent

Security Agent doesn't need to be expanded into a full topic in this phase, but it's very suitable for validating node-level application migration methods.

### You Can First Look at Four Questions

#### 1. Is it providing capabilities to the node itself?
Usually yes.  
Examples:
- Host security checks
- Process monitoring
- File integrity monitoring
- Audit log reporting
- Anomaly behavior detection

#### 2. Does it need one instance per node?
Usually yes.  
Because missing one node means missing one security coverage.

#### 3. Does it need host visibility?
Usually yes.  
May involve:
- Host file system
- Host processes
- Host network connections
- Audit log directory

#### 4. Is it more sensitive than node-exporter?
Usually yes.  
Because Security Agents often involve higher privileges and deeper host access capabilities.

### Migration Conclusion
Security Agent and node-exporter are both node agent-type components, but typically:

- Higher privileges
- More sensitive risks
- More direct impact on the host

### Operations Understanding Focus
When migrating to Security Agent, the focus isn't just changing the component name, but extending the analysis methods from node-exporter toward "higher privileges and stronger host involvement".

---

## V. What to Look At First When Migrating from node-exporter to CNI Node Components

CNI node components are also very suitable as objects for DaemonSet migration analysis.

### You Can First Look at Four Questions

#### 1. Is it providing node network capabilities?
Yes.  
Because Pod network access, network device handling, routing or forwarding capabilities often occur on the node side.

#### 2. Does it need one instance per node?
Usually yes.  
Because each node may schedule Pods, and each node needs network access capabilities.

#### 3. Does it need host directories or namespaces?
Usually yes.  
Examples may involve:
- CNI configuration directory
- Host network namespace
- Network interface card
- Routing and forwarding tables
- kubelet-related directories

#### 4. Is it more closely aligned with the underlying network stack than node-exporter?
Usually yes.  
Because CNI node components operate closer to the Linux network stack.

### Migration Conclusion
CNI node components also fit the node-level application model, but emphasize:

- Network capabilities
- Host network participation
- Node-side plugin execution logic

### Operations Understanding Focus
When migrating from node-exporter to CNI node components, the focus is on extending the "node agent" concept toward "node network capability agent".

---

## VI. What to Look At First When Migrating from node-exporter to CSI Node Components

CSI node-side components are also very suitable as migration targets for node-level applications.

### You Can First Look at Four Questions

#### 1. Is it providing node storage capabilities?
Yes.  
Because volume mounting, device access, and node-local storage processing typically occur on the node side.

#### 2. Does it require an instance per node  
It is usually required.  
Because each node may mount a volume, each node needs local mounting capability.

#### 3. Does it require access to the host's directories  
It is usually required.  
For example, it may involve:  
- kubelet volume directory  
- device path  
- host mount point  
- plugin directory  

#### 4. Does it emphasize interaction with the node's local storage environment  
It is usually the case.  
It is closer to:  
- device access  
- volume mounting  
- storage plugin call chain  

### Migration Conclusion  
CSI node components and CNI node components are both "platform capability node agents," one focusing on networking and the other on storage.

### Operations Understanding Focus  
When migrating from node-exporter to CSI node components, the analysis path remains unchanged, but the focus shifts from "monitoring collection" to "node mounting capability."

---

## Seven, The most important thing when migrating out of node-exporter is not component details, but migration methodology  

What needs to be formed in this phase is not a collection of isolated component knowledge points, but a fixed analysis framework.  

When facing new node-level components in the future, you can consistently ask the following questions first:  

### 1. Is it providing capability to the node itself  
For example:  
- node metrics  
- node logs  
- node security  
- node network  
- node storage  

### 2. Does it require an instance per node  
If the goal is node coverage, DaemonSet is usually the first thought.  

### 3. What host resources does it need  
For example:  
- `/proc`  
- `/sys`  
- `/var/log`  
- kubelet directory  
- device directory  
- network namespace  

### 4. Does it need to be closer to the host's runtime environment  
For example:  
- `hostNetwork`  
- `hostPID`  
- `hostPath`  
- privileged permissions  
- taint tolerance  

### 5. What is its output or downstream consumer  
For example:  
- Prometheus  
- log platform  
- security platform  
- network control plane  
- storage control plane  

### Operations Understanding Focus  
As long as these five questions are clearly answered first, the deployment direction of most node-level components can be captured effectively.

---

## Eight, Common characteristics of node-level applications  

Through examples like node-exporter, log collectors, security agents, CNI, and CSI, we can summarize several common features of node-level applications.  

### 1. Usually deployed as DaemonSet  
Because the focus is on node coverage, not total replica count.  

### 2. Often requires host visibility  
For example:  
- reading host directories  
- using host network  
- using host namespaces  
- perceiving node-local resources  

### 3. Usually not an entry point for business traffic  
They are typically not directly used by business requests, but more like:  
- collection layer  
- proxy layer  
- node capability layer  
- platform infrastructure layer  

### 4. Usually requires higher node coverage  
Because missing one node means losing one platform capability.  

### 5. Permissions and security boundaries are usually more sensitive  
Because they are closer to the host, common risk surfaces are also larger.  

---

## Nine, Node-level applications should now be clearly distinguishable from the previous two types  

By the end of this phase, the three workload models should be relatively clear.  

| Type | Typical Examples | Core Questions | More Common Controller |  
|---|---|---|---|  
| Stateless Business Application | Web API / Nginx | Replica count, traffic handling | Deployment |  
| Stateful Business Application | MySQL | Data, identity, recovery, membership | StatefulSet |  
| Node-Level Application | node-exporter / Fluent Bit / Security Agent | Node coverage, host visibility, platform proxy | DaemonSet |  

### Operations Understanding Focus  
What's truly important is not memorizing object names, but:  

> **After seeing a component, first determine which runtime model it belongs to.**  

---

## Ten, Final conclusion of this phase  

The deployment of node-level applications truly establishes not specific knowledge about a single monitoring or logging component, but an analysis method for "node agent components."  

The core of this method includes:  

- First determine if node coverage is needed  
- Then determine if host visibility is required  
- Then determine if closer access to host network or namespaces is needed  
- Finally determine its output target and platform relationship  

Through node-exporter as the starting point, and then migrating to log collectors, security agents, CNI, and CSI node components, this method has already been fully demonstrated.  

At this point, the **04-Node-Level Application Deployment** section can be concluded.  

---

## Eleven, Keyword Quick Notes  

- Node-Level Application: Platform agent component running per node  
- DaemonSet: Node coverage controller  
- node-exporter: Node metrics collection agent  
- Fluent Bit / Filebeat: Node log collection agent  
- Security Agent: Node security inspection and audit agent  
- CNI Node Component: Node network capability component  
- CSI Node Component: Node storage mounting capability component  
- hostPath: Host directory mounting  
- Node Coverage: An instance per node  
- Host Visibility: Component can read or perceive node resources  

---

## Twelve, Extended Operations Understanding  

The deployment challenges in Kubernetes often lie not in the YAML itself, but in the underlying runtime model of the components.  

- MySQL represents a stateful business system  
- node-exporter represents a node monitoring agent  
- Fluent Bit / Filebeat represent node log agents  
- Security Agents, CNI, and CSI node components represent platform-side node capability components  

Although these components all run in Kubernetes, they solve different problems:  

- How to scale business services  
- How to maintain data and identity for stateful services  
- How node agents run per node  
- How platform components work closely with the host  

After understanding these models in layers, subsequent learning of Helm, application deployment, updates, and rollbacks will have a more stable main line.  

---

## References  
- Kubernetes DaemonSet Official Documentation  
- Prometheus node-exporter Official Documentation  
- Fluent Bit Official Documentation  
- Filebeat Official Documentation  
- Kubernetes CNI / CSI Related Documentation  

---

## Next Day Recommendation  
The next section is recommended to enter:  

**Yes.09-Helm Basics: Kubernetes Package Management Tool IntroductionI don't know.**