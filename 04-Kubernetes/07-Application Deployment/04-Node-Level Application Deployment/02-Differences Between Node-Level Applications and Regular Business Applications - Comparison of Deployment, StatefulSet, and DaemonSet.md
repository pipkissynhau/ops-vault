# 02-Differences Between Node-Level Applications and Regular Business Applications: Comparison of Deployment, StatefulSet, and DaemonSet

## Document Description
- Document Purpose: An introductory comparison of three common Kubernetes workload controllers.
- Applicable Stage: After understanding the basics of DaemonSet, further understand the suitable application scenarios for Deployment, StatefulSet, and DaemonSet.
- Recommended Path: `04-Kubernetes/07-Application Deployment/04-Node-Level Application Deployment/02-Differences Between Node-Level Applications and Regular Business Applications: Comparison of Deployment, StatefulSet, and DaemonSet`

## Tags
#Kubernetes #Deployment #StatefulSet #DaemonSet #node-exporter #MySQL #Workload Models #Node-Level Applications #Stateful Applications #Stateless Applications #Application Deployment #Cloud-Native #Operation and Maintenance

---

## I. Why Compare Deployment, StatefulSet, and DaemonSet Together

When learning about Kubernetes workloads, it's easy to first remember the names of these three objects:

- Deployment
- StatefulSet
- DaemonSet

However, if one only understands them at the level that "they all create Pods," it's easy to have a misconception:

> **They are just controllers with different names.**

This understanding is not accurate enough.

Although these three controllers all create Pods, they address different issues:

- Deployment mainly addresses "how to manage business replicas."
- StatefulSet mainly addresses "how to manage stateful components."
- DaemonSet mainly addresses "how to distribute node-level agents."

Therefore, the focus of this section is not to compare their syntax but rather to examine:

- What deployment goals they each serve.
- What types of components they are more suitable for.
- Why node-exporter is not suitable for Deployment.
- Why MySQL is not suitable for DaemonSet.
- Why it's important to consider beyond just "whether they can run" and also whether the workload model matches.

---

## II. A Core Summary in One Sentence

You can start by remembering these three statements:

### Deployment
> **It is more suitable for managing a set of replaceable business replicas.**

### StatefulSet
> **It is more suitable for managing a set of members with unique identities, defined sequences, and storage relationships.**

### DaemonSet
> **It is more suitable for managing a set of agent-type components distributed across nodes.**

Remembering these three will help you make clearer judgments in many scenarios.

---

## III. What Problem Does Deployment Mainly Solve?

Deployment is one of the most common workload controllers.

It is particularly suitable for the following goals:

- Requiring multiple replicas.
- Where replicas are largely interchangeable.
- Not requiring unique identities.
- Not relying on local persistent data.
- Focusing more on the number of replicas and rolling updates.

### What Deployment Is More Like

Deployment is more like expressing:

> **How many instances of this business service are needed in total.**

### Common Scenarios

For example:

- Java API services.
- Nginx web services.
- Front-end static page services.
- Ordinary microservices.
- Gateway services.

### A Typical Example

For instance, if a backend API service is configured with `replicas: 3`, it usually means:

- There should be 3 replicas.
- If one replica fails, another will take its place.
- Which replica handles requests is generally not important.
- The more identical the backend replicas are, the better.

### Key Points for Operations and Maintenance

The core of Deployment is not "who the instances are" but rather:

> **How many working replicas there are in total.**

---

## IV. What Problem Does StatefulSet Mainly Solve?

StatefulSet is more suitable for stateful applications.

It is particularly suitable for the following goals:

- Instances have unique identities.
- Instances cannot be completely replaced by others.
- Instances require stable names.
- Instances need to maintain a relationship with their respective volumes.
- There may be specific startup sequences or membership relationships between instances.

### What StatefulSet Is More Like

StatefulSet is more like expressing:

> **I don't want several replicas; I want several members with clear identities.**

### Common Scenarios

For example:

- MySQL.
- PostgreSQL.
- Redis clusters.
- Kafka.
- ZooKeeper.
- Elasticsearch.

### An Intuitive Example

Consider a 3-replica MySQL StatefulSet. The typical instances would be named:

- `mysql-0`
- `mysql-1`
- `mysql-2`

The key point here is not "there are 3 MySQL containers" but rather:

- They have numbered identities.
- They may perform different roles.
- Each may be bound to its own volume.
- They are not completely interchangeable.

### Key Points for Operations and Maintenance

StatefulSet focuses on:

> **Member relationships.**

---

## V. What Problem Does DaemonSet Mainly Solve?

DaemonSet is suitable for node-level applications.

It is particularly suitable for the following## Twelve. Re-examining the Typical Differences Among These Three From an Object Perspective

### Common Focus Areas for Deployment
More frequently considered:
- `replicas`
- `strategy`
- Rolling updates
- Exposure of stateless Services

### Common Focus Areas for StatefulSet
More frequently considered:
- `serviceName`
- `replicas`
- `volumeClaimTemplates`
- Headless Services
- Stable DNS / Member relationships

### Common Focus Areas for DaemonSet
More frequently considered:
- Node selection range
- `hostPath`
- `hostNetwork`
- Pollution tolerance
- Node distribution

### Key Points for Operations and Maintenance Understanding
Although all three are workload objects, their areas of focus are entirely different.

---

## Thirteen. Why Is node-exporter a DaemonSet, MySQL a StatefulSet, and Web Services a Deployment?

This can be summarized in very simple terms:

### Web Services
The goal is:

> **Having multiple replicas providing service is sufficient**

Therefore, they are more suitable for Deployment.

### MySQL
The goal is:

> **Identifying the instance, locating its data, and ensuring it remains the same original instance**

Thus, they are more suitable for StatefulSet.

### node-exporter
The goal is:

> **Ensuring that every node has the capability to collect local metrics**

Therefore, they are more suitable for DaemonSet.

These three statements can basically serve as a simplified judgment template.

---

## Fourteen. A More Practical Method of Judgment

In actual work, one should not necessarily start by looking at the object type but rather consider the **component's responsibilities** first.

### If the component's responsibility is to **provide external business capabilities**
Consider:
- Deployment

### If the component's responsibility is to **provide stateful system capabilities**
Consider:
- StatefulSet

### If the component's responsibility is to **provide local proxy capabilities on each node**
Consider:
- DaemonSet

### A Common Misunderstanding
Many people only think of Deployment when they see a “container”.  
However, the true basis for judgment should be:

> **What role it plays within the cluster.**

---

## Fifteen. The Most Important Understandings in This Article

### 1. The three controllers are not different due to their syntax but because they represent different workload models
This is the first key understanding.

### 2. Deployment is more suitable for services that require a specific number of replicas
This is the second key understanding.

### 3. StatefulSet is more suitable for services with member relationships
This is the third key understanding.

### 4. DaemonSet is more suitable for services that need to cover all nodes in the cluster
This is the fourth key understanding.

### 5. When judging which controller to use, one should first consider the component's responsibilities rather than memorizing the object definitions
This is the fifth key understanding.

---

## Sixteen. Phase Summary

Deployment, StatefulSet, and DaemonSet all create Pods, but they represent three completely different deployment goals.

In summary:

- Deployment: Solves the issue of “number of replicas”
- StatefulSet: Solves the issue of “member relationships”
- DaemonSet: Solves the issue of “node coverage”

Using node-exporter, MySQL, and ordinary Web services as examples helps to clearly define the boundaries between these three models:

- node-exporter is a node-level proxy, suitable for DaemonSet
- MySQL is a stateful member system, suitable for StatefulSet
- Web services are stateless replica systems, suitable for Deployment

Once these three models are clearly understood, choosing the right controller will become much more natural when learning about log collectors, security agents, CNI / CSI node components, and so on.

---

## Seventeen. Quick Keywords for Memorization

- Deployment: Replica Quantity-Based Controller
- StatefulSet: Member Relationship-Based Controller
- DaemonSet: Node Coverage-Based Controller
- node-exporter: Typical Node-Level Application
- MySQL: Typical Stateful Application
- Web Services: Typical Stateless Application
- Workload Model: The way applications run in Kubernetes

---

## Eighteen. Further Understanding for Operations and Maintenance

In the study of Kubernetes workloads, the real challenge is not memorizing object names but developing a **model-based understanding**.

Once you establish these three models:

- Business Replica Model
- Stateful Member Model
- Node Proxy Model

Many subsequent decisions regarding object selection, YAML reading, and Helm values interpretation will become much more consistent.

Because then, your focus will no longer be on:

- How to write this object

but rather on:

- Which category of runtime model this component belongs to

This is also a crucial step in moving from “being able to write objects” to “being able to understand deployment designs”.

---

## References
- Kubernetes Deployment Official Documentation
- Kubernetes StatefulSet Official Documentation
- Kubernetes DaemonSet Official Documentation

---

## Suggestions for the Next Day
It is recommended to organize the following