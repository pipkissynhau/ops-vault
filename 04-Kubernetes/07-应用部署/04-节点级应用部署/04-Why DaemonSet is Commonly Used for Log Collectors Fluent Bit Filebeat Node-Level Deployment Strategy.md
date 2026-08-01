# 04-Why Log Collectors Commonly Use DaemonSet: Node-Level Deployment Strategies for Fluent Bit and Filebeat

## Document Notes
- Document Positioning: Introduction to node-level deployment strategies for log collectors in Kubernetes
- Applicable Stage: After understanding DaemonSet basics, workload model differences, and node-exporter node-level deployment methods, move to understanding node-level deployment for log collector components
- Recommended Path: `04-Kubernetes/07-Apply deployment/04-Application deployment at node level/04-Why do log collectors often use them? DaemonSet:Fluent Bit / Filebeat The idea of nodal deployment`

## Tags
#Kubernetes #DaemonSet #FluentBit #Filebeat #LogCollection #ApplicationAtNodeLevel #hostPath #LogPlatform #ContainerLog #ApplyDeployment #Clouds. #Transport

---

## I. Why Log Collectors Are Well Suited for Node-Level Application Mainline

In node-level applications, log collectors are the most typical type of component.

The node-exporter represents:

- Node metrics collection
- Host resource monitoring
- One monitoring agent per node

Log collectors represent another very common node-level capability:

- Container log collection
- Host log collection
- Log forwarding
- Unified log aggregation

Although these components share the same common deployment method of DaemonSet as node-exporter, they solve completely different problems.

### What node-exporter Focuses On
More focused on:
- CPU
- Memory
- File system
- Network interfaces
- Node resource metrics

### What Log Collectors Focus On
More focused on:
- Pod log files
- Host log directories
- Locations where container runtime generates logs
- Forwarding logs to downstream platforms

Therefore, log collectors are very suitable as the second typical case in DaemonSet mainline.

---

## II. Why Log Collectors Are Usually DaemonSet, Not Deployment

The core of this question isn't about "how official examples are written", but about the working mechanism of log collection itself.

### 1. Logs Are Naturally Dispersed Across Nodes
In Kubernetes, container logs are typically not centralized in one place, but dispersed across various nodes.

For example:
- Business Pod logs on Node A usually reside on Node A
- Business Pod logs on Node B usually reside on Node B

This means that log collectors cannot efficiently collect logs from all nodes if they don't run closely with the nodes.

### 2. The Goal Isn't "Running Several Collector Replicas", But "Covering All Nodes"
If log collectors only run 2 Deployment replicas, but the cluster has 8 nodes, it would result in:

- Some nodes have log collectors
- Some nodes don't have log collectors
- Incomplete node coverage

### 3. New Nodes Should Automatically Get Log Collectors
If a new node is added, as long as there are business Pods that might be scheduled to it, that node should also have the corresponding log collector.

### Operations Understanding Focus
The goal of log collectors isn't:

> Running a certain number of instances

But rather:

> All nodes must have log collection capabilities

This is precisely the workload model that DaemonSet is most suitable for.

---

## III. What Log Collectors Typically Collect in Kubernetes

At this stage, there's no need to expand all features of Fluent Bit or Filebeat. Just need to establish an overall sense of "what is collected".

### Common Collection Targets Include

#### 1. Container Standard Output Logs
This is the most common type.

Many business containers print logs to standard output, and Kubernetes and container runtimes will write these logs to log files on the node.

#### 2. Host System Logs
For example:
- `/var/log/messages`
- `/var/log/syslog`
- `/var/log/dmesg`
- Audit logs
- Security-related logs

#### 3. Specific Application Log Directories
If business containers or host processes write logs to specific directories, log collectors may directly read these paths.

### A Core Understanding
The focus of log collectors isn't just "looking at container internal stdout", but:

> **Reading log files from the node perspective and forwarding them to the log platform.**

---

## IV. Why Fluent Bit/Filebeat Commonly Use DaemonSet Deployment

Fluent Bit and Filebeat are both very common log collectors. Their implementations differ, but their node-level deployment strategies in Kubernetes are very similar.

### Commonality 1: Need Node Coverage
Each node must have collection capabilities.

### Commonality 2: Often Read Host Path Logs via hostPath
Because container log files are typically located on the node.

### Commonality 3: More Like Node Agents Rather Than Business Services
Their role isn't to directly provide functionality to business services, but to take on the responsibility of log reporting for nodes.

### Commonality 4: Often Serve as "Collection Layer"
They will later connect to:
- Elasticsearch
- Loki
- Kafka
- Logstash
- OpenSearch
- Other log platforms

### Operations Understanding Focus
The role of Fluent Bit/Filebeat here is more like:

> **Log transport agents on nodes.**

---

## V. Where Container Logs Typically Reside in Kubernetes

Before understanding log collectors, first understand where container logs typically reside.

Details may vary slightly across environments, but in Kubernetes, common paths include:

### 1. `/var/log/containers`
This is one of the most common and frequently encountered paths in collector configurations.

This typically contains:
- Links to container log files
- Log entries named by Pod/Namespace/Container

### 2. `/var/log/pods`
This usually contains Pod-level log directory structures.

### 3. Container Runtime Itself Log Directory
For example, different runtimes may place real log files in moreBottom paths.

### Why This Matters
Because log collectors don't magically obtain logs, but:

- Read log files on the host
- Identify log sources based on paths and naming conventions
- Then perform parsing and forwarding

### Operations Understanding Focus
Log collectors typically don't directly "enter each Pod to view logs", but:

> **Read container log files on the node.**

---

## VI. Why Log Collectors Typically Need hostPath

This is similar to node-exporter's approach, but with different reading targets.

### Why node-exporter Uses hostPath
To view:
- `/proc`
- `/sys`
- Host root directory

### Why Log Collectors Use hostPath
To view:
- `/var/log/containers`
- `/var/log/pods`
- Other host log directories

### A Simplified Example

    volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: containers
        hostPath:
          path: /var/lib/docker/containers

Or more commonly, directly mount: /think

```markdown
volumes:
  - name: containers
    hostPath:
      path: /var/log/containers

Corresponding inside the container:

    volumeMounts:
      - name: containers
        mountPath: /var/log/containers
        readOnly: true

### Operations Understanding Focus
The significance of hostPath in the log collector is not to "access all files on the host node", but to:

> **Allow the collector to see the log files on the node.**

---

## VII. Why These Log Directories Are Usually Mounted Read-Only

The responsibility of a log collector is to collect and forward logs, not to modify log files.

Therefore, a more reasonable mounting approach is typically:

    readOnly: true

### Why This Is More Reasonable
- Reduce the risk of accidentally writing to host node logs
- Lower the privilege scope
- Better align with the role positioning of the collector

### A Fundamental Principle
For node-level agent components, if data is only being read, it's typically advisable to:

> **Access in read-only mode.**

---

## VIII. Similarities and Differences Between Log Collectors and node-exporter

Both components are well-suited as DaemonSet examples, but their differences should be clearly understood.

### Commonalities
- Both typically use DaemonSet
- Both run closer to the host node
- Both commonly use hostPath
- Neither is an ordinary business service
- Both emphasize node coverage

### Differences

#### node-exporter
Focuses on collecting:
- Node resource metrics
- System runtime status

#### Log Collector
Focuses on collecting:
- Container logs
- Host node logs
- Contents of specific log files

### A More Intuitive Understanding

| Component | Main Read Object | Main Output Content |
|---|---|---|
| node-exporter | `/proc`, `/sys`, etc. system information | metrics |
| Fluent Bit / Filebeat | `/var/log/...`, etc. log files | logs |

### Operations Understanding Focus
Both belong to node agents, but one leans toward "metrics" while the other leans toward "logs."

---

## IX. Why Log Collectors Usually Also Need to Forward Logs

Collecting logs is just the first step; the true value of a log collector lies in forwarding.

### Common Downstream Destinations Include
- Elasticsearch / OpenSearch
- Loki
- Kafka
- Logstash
- Cloud vendor log services
- Self-built log platforms

### What Does This Mean
A log collector is not a "local log viewing tool," but rather:

> **Collect logs from nodes and send them to a unified platform.**

### A Fundamental Pipeline
You can initially understand it as:

    Node log files
        ->
    Fluent Bit / Filebeat
        ->
    Log platform
        ->
    Search / Analysis / Alerting

### Operations Understanding Focus
The role of a log collector is more like:
- Collection layer
- Forwarding layer
- Data transportation layer before the unified entry point

---

## X. Why Log Collectors Usually Carry More "Parsing" and "Filtering" Logic

This is also quite different from node-exporter.

### node-exporter Output Is Usually More Standardized
Metrics themselves have a high degree of structurization.

### Logs Faced by Log Collectors Are Usually More Diverse
They may include:
- Plain text logs
- JSON logs
- Multi-line exception stacks
- Container stdout
- Application custom format logs

### Therefore, Commonly Added Logic Includes
- Parsing timestamps
- Parsing JSON fields
- Identifying Namespace / Pod / Container
- Adding Kubernetes metadata
- Log filtering
- Merging multi-line logs

### Operations Understanding Focus
The complexity of a log collector comes not only from "how to deploy," but also from:

> **How to turn raw logs into usable logs.**

At this stage, understanding why it's deployed as a DaemonSet is sufficient; parsing details can be explored later.

---

## XI. How to Understand a Teaching-Type Fluent Bit / Filebeat Deployment Plan

At this stage, you don't need to memorize fluent-bit.yaml or filebeat.yaml entirely, but you should be able to understand what they typically express.

### Common Content You'll See

#### 1. `kind: DaemonSet`
Indicates the target is one instance per node.

#### 2. `hostPath`
Usually mounts:
- `/var/log`
- `/var/log/containers`
- `/var/log/pods`
- Other log paths

#### 3. `ConfigMap`
Used for placing:
- Input configuration
- Parser configuration
- Output configuration

#### 4. `ServiceAccount` / RBAC
Because many log collectors also query Kubernetes API to supplement Pod metadata.

#### 5. Output Destination Configuration
For example:
- Elasticsearch address
- Loki address
- Kafka address

### Operations Understanding Focus
Log collector YAMLs are usually more complex than node-exporter, but the core can still be summarized in one sentence:

> **Place a proxy that reads local logs and forwards them on each node.**

---

## XII. Why Log Collectors Often Need Kubernetes Metadata

This is a very important point in the log collection scenario.

If only raw log files are obtained, you typically only know:
- Log content
- File path

But in actual operations, it's more desirable to know:
- Which Namespace
- Which Pod
- Which container
- Which node
- Which application generated the logs

Therefore, many log collectors supplement Kubernetes metadata to make the final logs in the platform more searchable, more filterable, and more associable.

### Operations Understanding Focus
A log collector isn't simply "copying file content," but rather trying to associate logs with the Kubernetes runtime context as much as possible.

---

## XIII. Why Log Collectors Often Need to Tolerate More Nodes

Like node-exporter, log collectors usually also want to cover more nodes.

### Common Reasons
- Work nodes need to collect container logs
- Some control plane nodes may also run system Pods
- Special role nodes may also need to collect local logs

### Therefore, common configurations are usually broader
For example, using:

    tolerations:
      - operator: Exists

### What Does This Mean
It means:
- Try to avoid missing nodes due to taints
- Try to achieve complete node coverage

### Operations Understanding Focus
The goal of a log collector is usually not "collect logs from part of the nodes," but rather:

> **To cover log sources across the cluster as comprehensively as possible.**

---

## XIV. Why Log Collectors Are Not Naturally Equivalent to "Log Platforms"

This is a very common point of confusion.

### What Log Collectors Are Responsible For
- Reading logs from nodes
- Doing basic processing
- Forwarding to downstream

### What Log Platforms Are Responsible For
- Storage
- Indexing
- Search
- Visualization
- Alerting
- Multi-dimensional analysis
```

### A Simple Differentiation
Fluent Bit / Filebeat are more like:

> **Collection and Transport Layer**

Elasticsearch / Loki / OpenSearch are more like:

> **Storage and Query Layer**

### Operations Understanding Focus
Log collectors are part of the log system, but not the whole.

---

## Fifteen, Differences Between Log Collectors and Ordinary Business Applications

You can remember it with a simple comparison.

| Dimension | Ordinary Business Application | Log Collector |
|---|---|---|
| More Common Controller | Deployment | DaemonSet |
| Focus | Service Replicas | Node Coverage |
| Dependency on hostPath | Usually Not | Usually Yes |
| Proximity to Host | Usually Not | Usually Yes |
| Output Target | Provide Functionality to Business | Forward Data to Log Platform |

### Operations Understanding Focus
The goal of ordinary business applications is "to provide services externally",  
The goal of log collectors is "to complete log collection and forwarding on nodes".

---

## Sixteen, The Most Important Recognitions in This Section

### 1. Log Collectors Are Usually More Suitable for DaemonSet
Because logs are naturally dispersed across nodes.

### 2. hostPath in Log Collectors Is Primarily Used to Read Host Log Paths
This is the second recognition.

### 3. Log Collectors Are Usually Node Agents, Not Business Services
This is the third recognition.

### 4. The Service Role of Log Collectors Differs from Business Services
It serves more the platform pipeline, rather than business entry points.

### 5. Log Collectors Are Usually Just the Collection and Forwarding Layer of the Log System
They are not a complete log platform.

---

## Seventeen, Stage Summary

Why log collectors like Fluent Bit / Filebeat are commonly deployed with DaemonSet essentially comes down to a simple reason:

- Logs are dispersed by node
- Collectors need to be close to nodes
- Instances should automatically scale when nodes increase
- Instances should automatically disappear when nodes are deleted

These components, like node-exporter, are all typical node-level applications, but they target different objects:

- node-exporter targets node metrics
- log collectors target node logs

After understanding this, when facing more node-level components, you can first ask:

> **Is it to collect node-native capabilities, or to provide business services externally.**

---

## Eighteen, Keyword Mnemonics

- DaemonSet: Node Coverage Controller
- Fluent Bit: Common Log Collector
- Filebeat: Common Log Collector
- hostPath: Host Directory Mount
- `/var/log/containers`: Common Container Log Path
- Log Forwarding: Collectors send logs to downstream platforms
- Kubernetes Metadata: Namespace, Pod, Container, Node, etc. context information
- Log Platform: System responsible for storing, retrieving, and analyzing logs

---

## Nineteen, Operations Extended Understanding

Log collector components well demonstrate a core characteristic of "node-level applications":

> **It is not to directly provide services to business, but to enable the entire platform with some foundational capabilities.**

This foundational capability includes:
- Log collection
- Metric collection
- Security checks
- Network proxy
- Storage node-side capabilities

Therefore, from node-exporter to Fluent Bit / Filebeat, you can more clearly establish a platform perspective:

- What business services are running
- What capabilities node agents are supplementing
- How platform observability capabilities are gradually built

---

## References
- Kubernetes DaemonSet Official Documentation
- Fluent Bit Official Documentation
- Filebeat Official Documentation
- Prometheus Official Documentation

---

## Next Day Recommendation
Next post recommendation to organize:

[[05-Node-Level Application Deployment Phase Summary - Migrating from node-exporter to Logs Security Agent CNI CSI and Node Components]]