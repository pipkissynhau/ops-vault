# 02-Differences Between Node-Level Applications and Regular Business Applications: Deployment, StatefulSet, DaemonSet Comparison

## Document Notes
- Document positioning: A beginner's guide to comparing three common Kubernetes workload controllers
- Applicable stage: After completing DaemonSet basics, further understanding the suitable application models for Deployment, StatefulSet, and DaemonSet
- Recommended path: `04-Kubernetes/07-Apply deployment/04-Application deployment at node level/02-Distinction between nodal level application and general business application:DeploymentI don't know.StatefulSetI don't know.DaemonSet Comparison`

## Tags
#Kubernetes #Deployment #StatefulSet #DaemonSet #node-exporter #MySQL #TaskLoadModel #ApplicationAtNodeLevel #ApplyWithStatus #NoStatusApplication #ApplyDeployment #Clouds. #Transport

---

## I. Why Compare Deployment, StatefulSet, and DaemonSet Together

When learning Kubernetes workloads, it's easy to remember the three object names first:

- Deployment
- StatefulSet
- DaemonSet

If you only stay at the level of "they can all create Pods," it's easy to fall into a misconception:

> **It's just three controllers with different syntax.**

This understanding is inaccurate.

Although these three controllers will all create Pods, they solve different problems:

- Deployment mainly addresses "how to manage business replicas"
- StatefulSet mainly addresses "how to manage stateful members"
- DaemonSet mainly addresses "how to distribute node-level agents"

Therefore, this article's focus isn't on comparing syntax, but rather on comparing:

- What deployment goals they respectively express
- What types of components they respectively suit
- Why node-exporter is unsuitable for Deployment
- Why MySQL is unsuitable for DaemonSet
- Why you shouldn't just look at "whether it can run," but rather "whether the workload model matches"

---

## II. First, Remember the Core Summary

You can first remember the following three sentences:

### Deployment
> **More suitable for managing a group of replaceable business replicas.**

### StatefulSet
> **More suitable for managing a group of members with identity, order, and storage relationships.**

### DaemonSet
> **More suitable for managing a group of agent components distributed by node.**

If you remember these three sentences first, many scenario judgments will become clearer later.

---

## III. What Problems Does Deployment Mainly Solve

Deployment is one of the most common workload controllers.

It mainly suits these targets:

- Needs several replicas
- Replicas are largely replaceable
- Doesn't depend on fixed identity
- Doesn't depend on local persistent data
- More concerned with replica count and rolling updates

### What Is Deployment More Like
Deployment is more like expressing:

> **This business service needs several instances.**

### Common Scenarios
For example:

- Java API service
- Nginx Web service
- Frontend static page service
- Regular microservices
- Gateway services

### A Typical Example
For example, a backend API service writes:

    replicas: 3

This usually indicates:

- Needs 3 replicas
- Replaces a missing one
- Which replica is hit by requests generally doesn't matter
- Backend replicas should be as close to "indistinguishable" as possible

### Operations Understanding Focus
Deployment's core isn't "who the instances are," but rather:

> **How many working replicas there are in total.**

---

## IV. What Problems Does StatefulSet Mainly Solve

StatefulSet is more suitable for stateful applications.

It mainly suits these targets:

- Instances have identity differences
- Instances are not fully replaceable
- Instances need stable names
- Instances need to maintain relationships with their own volumes
- Instances may have startup order or membership relationships

### What Is StatefulSet More Like
StatefulSet is more like expressing:

> **I don't want several replicas, but several members with clear identities.**

### Common Scenarios
For example:

- MySQL
- PostgreSQL
- Redis Cluster
- Kafka
- ZooKeeper
- Elasticsearch

### An Intuitive Example
For example, a 3-replica MySQL StatefulSet, common members would appear as:

- `mysql-0`
- `mysql-1`
- `mysql-2`

Here, the focus isn't "having 3 MySQL containers," but rather:

- They have numbering
- They may take on different roles
- They may each bind their own volumes
- They are not fully interchangeable

### Operations Understanding Focus
StatefulSet focuses on:

> **Member relationships.**

---

## V. What Problems Does DaemonSet Mainly Solve

DaemonSet is suitable for node-level applications.

It mainly suits these targets:

- Needs to run an instance on each node
- Component responsibilities are close to the host
- Component is a node agent, not a business replica
- Automatically adds instances when nodes increase
- Automatically reduces instances when nodes decrease

### What Is DaemonSet More Like
DaemonSet is more like expressing:

> **Which nodes must have this component.**

### Common Scenarios
For example:

- node-exporter
- Fluent Bit
- Filebeat
- Security Agent
- CNI node component
- CSI node component
- NodeLocal DNSCache

### Taking node-exporter as an Example
If the cluster has 5 nodes, DaemonSet's goal is usually:

- Each of the 5 nodes has 1 node-exporter

Rather than:

- Running 2 or 3 node-exporter replicas in total

### Operations Understanding Focus
DaemonSet focuses on:

> **Node coverage.**

---

## VI. Core Thinking Differences Among the Three Controllers

If you place the three together on a line, you can understand them this way:

### Core Problem of Deployment
- How many replicas are needed

### Core Problem of StatefulSet
- Who are these instances

### Core Problem of DaemonSet
- Which nodes must have an instance

These three problems seem related to Pods, but they are essentially three different deployment goals.

---

## VII. Use a Unified Table to Build an Overall Sense

| Controller | Focuses On | Instance Relationships | Common Scenarios |
|---|---|---|---|
| Deployment | Replica Count | Generally Replaceable | Web Services, API Services, Frontend Services |
| StatefulSet | Identity, Order, Volume Relationships | Not Fully Replaceable | MySQL, PostgreSQL, Kafka, ZooKeeper |
| DaemonSet | Node Coverage | Distributed by Node | node-exporter, Log Collectors, Security Agents |

This table is recommended to be viewed multiple times, as it can almost solve most initial judgments about "which controller to use" in subsequent scenarios.

---

## VIII. Why node-exporter Is Unsuitable for Deployment

This is a natural comparison example.

### What Problems Would Arise If You Run node-exporter with Deployment /think
</think>

> [!note] What problems would arise if you run node-exporter with Deployment

#### 1. Only guarantees replica count, not node coverage
For example:

    replicas: 3

Only indicates a total of 3 node-exporter instances running, but does not guarantee coverage across all nodes.

#### 2. New nodes won't automatically get instances
If a new node is added to the cluster, the Deployment won't automatically add a node-exporter instance just because "a new node appeared".

#### 3. Some nodes may not be monitored at all
The final result could be:

- node-1 has node-exporter
- node-2 has node-exporter
- node-3 has none
- node-4 has none
- node-5 has node-exporter

This would result in incomplete node metrics collected by Prometheus.

### Why DaemonSet is more reasonable
Because node-exporter's goal isn't "run several service replicas", but rather:

> **Every node must have native metrics collection capability.**

---

## IX. Why MySQL is unsuitable for DaemonSet

Looking at it from the opposite angle, MySQL is also unsuitable for DaemonSet.

### What problems would arise if running MySQL with DaemonSet

#### 1. Running a MySQL instance on every node has no clear business meaning
Databases are typically not "one per node proxies", but rather a business system or database system.

#### 2. Data and membership relationships won't automatically become reasonable due to "one per node"
MySQL cares about:
- Data directory
- Membership relationships
- Master-slave or master-replica structure
- Initialization and recovery

Not node coverage.

#### 3. Node changes will force database instance distribution
If a new node is added, DaemonSet will automatically add a MySQL Pod, but the database may not need a new member.

### Why StatefulSet is more reasonable
Because MySQL requires:

- Stable identity
- Stable data directory
- Stable membership relationships

Not "one per node".

### Operations understanding focus
MySQL is a business middleware, not a node proxy.

---

## X. Why ordinary business services are unsuitable for StatefulSet or DaemonSet

Looking at another comparison: why ordinary business services are typically better suited for Deployment.

### For example, a Java API service
It typically has these characteristics:

- Multiple replicas are generally interchangeable
- Doesn't require fixed identity
- Doesn't need one per node
- Doesn't rely on local critical data
- More concerned with rolling updates and replica elasticity

### If using StatefulSet
Although it can run, it would introduce many unnecessary semantics:

- Fixed identity
- Order
- Volume relationships

This would make the model heavier.

### If using DaemonSet
It would become:
- Each node has an API Pod

This usually has no business meaning and would waste resources.

### Why Deployment is more reasonable
Deployment directly expresses the core business goal:

> **Provide several interchangeable replicas to handle traffic.**

---

## XI. What are the most common judgment criteria for these three controllers

When determining which controller to use for a component later, you can first ask the following three questions.

### Question 1: Is the focus on replica count?
If the focus is "how many total replicas", typically consider Deployment first.

### Question 2: Is the focus on member identity?
If the focus is "who are these members", typically consider StatefulSet first.

### Question 3: Is the focus on node coverage?
If the focus is "which nodes must have one", typically consider DaemonSet first.

These three questions are more practical than memorizing object definitions.

---

## XII. Revisiting the typical differences from an object perspective

### Commonly focused fields for Deployment
More commonly focus on:
- `replicas`
- `strategy`
- Rolling updates
- Stateless Service exposure

### Commonly focused fields for StatefulSet
More commonly focus on:
- `serviceName`
- `replicas`
- `volumeClaimTemplates`
- Headless Service
- Stable DNS / Membership relationships

### Commonly focused fields for DaemonSet
More commonly focus on:
- Node selection scope
- hostPath
- hostNetwork
- Taint tolerance
- Node distribution

### Operations understanding focus
Although all three are workload objects, their focus points are completely different.

---

## XIII. Why node-exporter is DaemonSet, MySQL is StatefulSet, and web services are Deployment

You can summarize it with one simple sentence:

### Web service
Goal is:

> **Having several replicas to provide service is sufficient**

So it's more suitable for Deployment.

### MySQL
Goal is:

> **What is this instance's identity, where is its data, and is it the original instance**

So it's more suitable for StatefulSet.

### node-exporter
Goal is:

> **Every node must have native metrics collection capability**

So it's more suitable for DaemonSet.

These three sentences can basically serve as the simplest judgment template.

---

## XIV. A more practical judgment method in real work

In real work, it's not necessarily about looking at the object first, but rather about the "component's responsibility".

### If the component's responsibility is "providing business capabilities externally"
Prioritize thinking of:
- Deployment

### If the component's responsibility is "providing stateful system capabilities"
Prioritize thinking of:
- StatefulSet

### If the component's responsibility is "providing native proxy capabilities on each node"
Prioritize thinking of:
- DaemonSet

### A common misconception
Many people see "this is a container" and only think of Deployment.  
But the real judgment basis should be:

> **What role does it play in the cluster.**

---

## XV. The most important several understandings from this article

### 1. These three controllers are not different syntax, but different workload models
This is the first understanding.

### 2. Deployment is more suitable for replica count-based services
This is the second understanding.

### 3. StatefulSet is more suitable for membership relationship-based services
This is the third understanding.

### 4. DaemonSet is more suitable for node coverage-based services
This is the fourth understanding.

### 5. When judging controllers, determine the component's responsibility first, not memorize object definitions
This is the fifth understanding.

---

## XVI. Summary by stage

Deployment, StatefulSet, and DaemonSet all create Pods, but they express three completely different deployment goals.

It can be summarized as:

- Deployment: Solving "replica count"
- StatefulSet: Solving "membership relationships"
- DaemonSet: Solving "node coverage"

Using node-exporter, MySQL, and ordinary web services as comparisons, you can more clearly establish the boundaries of these three models:

- node-exporter is a node agent, suitable for DaemonSet
- MySQL is a stateful member system, suitable for StatefulSet
- Web services are stateless replica systems, suitable for Deployment

As long as these three models are clear, subsequent learning about log collectors, security agents, CNI/CSI node components will naturally lead to more natural controller selection.

---

## XVII. Keyword quick recall /think

- Deployment: Replica Count Controller
- StatefulSet: Member Relationship Controller
- DaemonSet: Node Coverage Controller
- node-exporter: Typical Node-Level Application
- MySQL: Typical Stateful Application
- Web Service: Typical Stateless Application
- Workload Model: How Applications Run in Kubernetes

---

## EighteenI don't know.Operational Extension Understanding

In Kubernetes workload learning, the real difficulty isn't remembering object names, but establishing "model sense".

Once the following three models are established:

- Business Replica Model
- Stateful Member Model
- Node Agent Model

Subsequent object selection, YAML reading, and Helm values understanding will become more stable.

Because the focus will no longer be on:

- How to write this object

But rather on:

- Which runtime model this component belongs to

This is also the key step from "knowing how to write objects" to "knowing how to read deployment designs".

---

## References
- Kubernetes Deployment Official Documentation
- Kubernetes StatefulSet Official Documentation
- Kubernetes DaemonSet Official Documentation

---

## Next Day Recommendation
Next post recommendation:Collate

[[03-node-exporter Deployment Breakdown - hostPath hostNetwork Collection Path Service Relationship]]