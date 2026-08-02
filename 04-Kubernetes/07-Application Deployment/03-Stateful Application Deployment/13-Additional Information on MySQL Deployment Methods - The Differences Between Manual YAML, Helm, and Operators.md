# 13-Additional Information on MySQL Deployment Methods: The Differences Between Manual YAML, Helm, and Operators

## Document Description
- Document Purpose: Understanding and choosing different deployment methods for MySQL in Kubernetes
- Applicable Stage: After completing basic single-instance MySQL deployment and gaining a deeper understanding, this section compares actual deployment methods
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/03-Stateful Application Deployment/13-Additional Information on MySQL Deployment Methods: The Differences Between Manual YAML, Helm, and Operators`

## Tags
#Kubernetes #MySQL #YAML #Helm #Operator #StatefulSet #PVC #ConfigMap #Secret #Database Deployment #Application Deployment #Cloud-Native #Operations

---

## I. Why Understand Different Deployment Methods

There is more than one way to implement MySQL in Kubernetes.

Common methods include:

- Manual YAML
- Using Helm
- Using Operators

These three methods are not simply about which one is more advanced or less; they are each suitable for different stages, complexities, and team capabilities.

If you only know one method, common issues can arise:

- If you only know how to write manual YAML, it might be limited to practice and not conducive to rapid deployment.
- If you only know how to use Helm install, you might only be able to deploy but not understand the configuration.
- If you have only heard of Operators, you might think of them as “more advanced automation” without understanding what they actually manage.

Therefore, understanding the differences between these three methods is not about memorizing terms but about answering the following questions:

- When is manual YAML appropriate?
- When is Helm appropriate?
- When are Operators appropriate?
- What problems do they each help solve?
- Which one should be prioritized at this stage?

---

## II. A General Conclusion First

You can start with a basic judgment:

### Manual YAML
It is more suitable for understanding object relationships and the basics of deployment.

### Helm
It is more suitable for standardized installation and parameterized delivery.

### Operators
It is more suitable for managing complex lifecycles and database systems that need long-term operation.

Remember this conclusion for now, and we will elaborate on it later.

---

## III. What Are the Core Features of Manual YAML

Manual YAML means directly maintaining these objects yourself:

- StatefulSet
- PVC
- ConfigMap
- Secret
- Service

When necessary, you may also include:

- Job
- CronJob
- Headless Service
- PodDisruptionBudget
- NetworkPolicy
- RBAC, and other objects.

### Its Most Direct Feature
It is that:

> **The boundaries of each object are completely exposed, and all relationships are organized by the deployer.**

### Common Applications
For example, when deploying a single-instance MySQL, you need to explicitly specify:

- Where the root password comes from in a Secret
- Which ConfigMap contains the configuration files
- Which volume the data directory should be mounted on
- How the Service selects Pods
- How to write the probes
- How to request the PVC

### What This Means
It doesn’t hide complexity; instead, it presents it completely.

---

## IV. Advantages of Manual YAML

### 1. Most Suitable for Learning Object Relationships
This is its greatest advantage.

By writing manually, you can clearly see:

- Why MySQL needs a StatefulSet
- The relationship between PVC and the data directory
- The relationship between ConfigMap and configuration files
- The relationship between Secret and passwords
- The relationship between Service and access points

### 2. Clearer Boundaries
The responsibilities of each object can be examined individually.

### 3. Ideal for Building Minimum Understandable Models
For example:

- Single-instance MySQL
- Single-instance PostgreSQL
- Single-instance MinIO

Such basic models are well-suited for building understanding through manual YAML.

### 4. Easy to Debug Basic Issues
Since there is no encapsulation layer, you see the original objects themselves.

---

## V. Disadvantages of Manual YAML

### 1. Much Repetitive Work
When there are many objects, repetitive configuration becomes obvious.

### 2. High Cost of Changes
For example:

- Changing the storage size
- Updating the image version
- Modifying the password injection method
- Adjusting the probes
- Changing how the Service is exposed

All these require manual maintenance.

### 3. Not Suitable for Standardized Reuse
The same pattern of database deployment can easily lead to multiple inconsistent implementations.

### 4. Inappropriate for Managing Complex Database Lifecycles
For example:

- Automatic backups
- Automatic recovery
- Cluster scaling
- Fault handling
- Version upgrade orchestration

These tasks become cumbersome when done manually with YAML.

### A Basic Judgment
Manual YAML is more suitable for:

> **Understanding and practice**

But it doesn’t necessarily mean it’s the most efficient long-term deployment method.

---

## VI. What Are### Why is a StatefulSet Needed?
### Why are PVCs Required?
### Why Use ConfigMap for Configuration?
### Why Employ Secrets for Passwords?
### Why Access Databases Through Services in Business Applications?

### Step Two: Focus on Helm
By this point, you already understand what objects are being rendered behind the chart, so you won’t just stop at knowing how to install them without understanding their purpose.

### Step Three: Finally, Understand Operators
With a solid foundation in:
- basic object relationships,
- Helm’s packaging approach,
- and the database operation model,
you can better grasp what operators handle.

### A Fundamental Conclusion
A more logical learning path is not:

> Operator > Helm > YAML

but rather:

> YAML > Helm > Operator

---

## Fifteen: Why You Should Not Skip Writing YAML to Learn Helm or Operators Directly
Skipping the object-level understanding will lead to typical problems later on.

### 1. Not Understanding What the Chart Generates
You’ll only be able to execute Helm commands without knowing what objects are being created.

### 2. Not Comprehending How values Control Configuration
For example, terms like persistence, auth, primary, secondary, service, and resources will lose their clear meaning if you don’t understand how they are configured.

### 3. Not Seeing the Scope of Operators’ Control
You won’t know:
- which resources they help create,
- which lifecycle phases they manage,
- and their relationships with StatefulSet, Service, and PVCs.

### The Value of Writing YAML by Hand
The goal is not to write everything manually forever, but to build a clear understanding of the underlying object relationships.

---

## Sixteen: A Practical Approach Based on Real Work Scenarios
In real-world settings, the best choice is often not “always use one method,” but to choose based on the specific task at hand.

### Scenario 1: Learning and Experimenting
优先选择 writing YAML manually
to clearly understand object relationships.

### Scenario 2: Standardized Installation of Middleware
Prefer Helm
for its suitability for standardized installation and parameter management.

### Scenario 3: Complex Database Clusters and Long-Term Management
Choose Operators
for their capability to manage complex state systems over time.

### A Practical Judgment Criterion
Ask yourself not:

> “Which is the most advanced?”

but rather:

> “Does this current issue involve object definition, deployment packaging, or lifecycle management?”

---

## Seventeen: Key Understandings from This Article
### 1. Writing YAML, Helm, and Operators Are Not Mutually Exclusive Hierarchies
They each address different aspects of application deployment.

### 2. Writing YAML Is Most Useful for Understanding Basic Object Relationships
This is its most important value at this stage.

### 3. Helm Mainly Handles Standardized Installation and Parameter Management
It makes deployments more efficient, but it does not necessarily mean a deeper understanding.

### 4. Operators Are Better for Managing Complex Stateful Systems
The focus is on continuous management, not just installation.

### 5. A More Reasonable Learning Order Is YAML → Helm → Operators
This sequence helps build a stable foundation of knowledge.

---

## Eighteen: Phase Summary
The three common ways to deploy MySQL in Kubernetes can be simplified as three layers:

- Writing YAML manually: Understand basic object relationships.
- Using Helm: Learn about standardized installation and parameterized delivery.
- Employing Operators: Comprehend complex lifecycle management and continuous operations.

From a learning perspective, writing YAML should not be seen as a “backward method,” but rather as a necessary step in understanding the underlying logic of stateful application deployment. Helm and Operators can then help you move towards more practical deployment methods.

---

## Nineteen: Key Terms for Quick Reference
- Writing YAML manually: Directly manage object relationships.
- Helm: Template-based, parameterized delivery.
- Operators: Controllers that manage lifecycles continuously.
- StatefulSet: A stateful object carrier for databases.
- values: The entry point for Helm parameters.
- CRD: The basis for custom operators in Kubernetes.
- reconcile: The core mechanism of operators to ensure consistency.

---

## Twenty: Further Understanding for Operations and Maintenance
The differences in database deployment methods essentially reflect who bears the complexity:

### Writing YAML manually
The complexity is mainly borne by the deployer.

### Using Helm
Part of the complexity is handled by the chart templates.

### Employing Operators
The controller takes on even more of the complexity.

Therefore, learning these methods should not focus solely on “knowing how to use commands,” but also on understanding:

- which complexities are exposed,
- which are encapsulated,
- and which still need to be understood and managed personally.

Only by clarifying this can you make more informed decisions when choosing a deployment method in real-world scenarios.

---

## References
- Kubernetes StatefulSet Official Documentation
- Helm Official Documentation
- Kubernetes Operator Pattern Official Documentation
- MySQL-related Helm Chart/Operator Documentation

---

## Next Day's Suggestion
