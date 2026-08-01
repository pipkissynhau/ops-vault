# 13-MySQL Deployment Methods Supplement: Differences Between Handwritten YAML, Helm, and Operator

## Document Notes
- Document Focus: Understanding and choosing different delivery methods for MySQL in Kubernetes
- Applicable Stage: After completing MySQL single-instance deployment and advanced understanding, entering actual delivery method comparison
- Recommended Path: `04-Kubernetes/07-Apply deployment/03-Apply deployment in status/13-MySQL Deployment complement: handwritten YAMLI don't know.Helm and Operator The difference.`

## Tags
#Kubernetes #MySQL #YAML #Helm #Operator #StatefulSet #PVC #ConfigMap #Secret #DatabaseDeployment #ApplyDeployment #Clouds. #Transport

---

## I. Why Understanding Different Deployment Methods Matters

MySQL deployment in Kubernetes isn't limited to a single approach.

Common methods typically include:

- Handwritten YAML
- Using Helm
- Using Operator

These three approaches aren't simply "more advanced" or "less advanced" - they suit different stages, complexity levels, and team capabilities.

If only proficient in one method, common issues often arise:

- Only knowing handwritten YAML, easily remaining at practice level,Not good. rapid delivery
- Only knowing Helm install, easily remaining at "can install but can't understand"
- Only hearing about Operator, easily viewing it as "more advanced automation", but not understanding what it takes over

Therefore, understanding the differences between these three methods, the focus isn't memorizing terms, but answering these questions:

- When is handwritten YAML suitable?
- When is Helm suitable?
- When is Operator suitable?
- What problems do they respectively solve?
- Which should be prioritized in the current stage?

---

## II. First, Establish a Basic Judgment

We can first establish a fundamental conclusion:

### Handwritten YAML
More suitable for understanding object relationships and deployment basics.

### Helm
More suitable for standardized installation and parameterized delivery.

### Operator
More suitable for managing complex lifecycle and long-running database systems.

This conclusion can be remembered first, then expanded later.

---

## III. What Are the Core Characteristics of Handwritten YAML

Handwritten YAML means directly maintaining these objects:

- StatefulSet
- PVC
- ConfigMap
- Secret
- Service

Sometimes also including:

- Job
- CronJob
- Headless Service
- PodDisruptionBudget
- NetworkPolicy
- RBAC and other objects

### Its Most Direct Feature
Is:

> **Object boundaries are fully exposed, all relationships are organized by the deployer.**

### Common Manifestations
For example, deploying a single-instance MySQL requires explicitly writing:

- root password source from which Secret
- configuration file source from which ConfigMap
- data directory mounted to which volume
- how Service selects Pod
- how probes are written
- how PVC is applied

### What Does This Mean
It doesn't help "hide complexity", but fully displays the complexity.

---

## IV. Advantages of Handwritten YAML

### 1. Most Suitable for Learning Object Relationships
This is the biggest advantage.

Because through handwritten YAML, you can truly see:

- Why MySQL uses StatefulSet
- What's the relationship between PVC and data directory
- What's the relationship between ConfigMap and configuration files
- What's the relationship between Secret and passwords
- What's the relationship between Service and access entry

### 2. Most Clear Boundaries
Each object's responsibilities can be viewed separately.

### 3. Most Suitable for Building Minimal Understandable Models
For example:
- Single-instance MySQL
- Single-instance PostgreSQL
- Single-instance MinIO

These basic models are very suitable for establishing cognition first with handwritten methods.

### 4. Convenient for Troubleshooting Basic Issues
Because there's no encapsulation layer, you see the original objects directly.

---

## V. Limitations of Handwritten YAML

### 1. More Repetitive Work
With more objects, repetitive configuration becomes obvious.

### 2. High Change Cost
For example:
- Changing storage size
- Changing image version
- Changing password injection method
- Changing probe
- Changing Service exposure method

All require manual maintenance.

### 3. Not Suitable for Standardized Reuse
Same pattern database deployments often result in multiple inconsistent writing styles.

### 4. Unsuitable for Complex Database Lifecycle Management
For example:
- Automatic backup
- Automatic recovery
- Cluster scaling
- Fault handling
- Version upgrade orchestration

These would quickly become cumbersome with pure handwritten YAML.

### A Basic Judgment
Handwritten YAML is more suitable for:

> **Understanding and Practice**

But not inherently equal to:

> **The Most Effort-Saving Long-Term Delivery Method**

---

## VI. What Are the Core Characteristics of Helm

Helm's core idea isn't "inventing new Kubernetes objects", but:

> **Templating, parameterizing, and making reusable a group of Kubernetes objects.**

In other words, Helm essentially still generates:

- StatefulSet
- Service
- PVC
- ConfigMap
- Secret

These objects, just making them into a set of installable, upgradable, and reusable application packages.

### Helm's Most Common Usage
Deployment actions typically become:

    helm install
    helm upgrade
    helm rollback
    helm uninstall

### Key Problems It Solves
Primarily:

- Too many objects to maintain manually
- Similar deployments hoping for reuse
- Parameters hoping to be managed via values
- Installation, upgrade, and rollback hoping for unification

---

## VII. Advantages of Helm

### 1. More Suitable for Standardized Installation
For common databases, middleware, and components, Helm usually delivers faster.

### 2. More Suitable for Parameterized Management
For example, you can control via values:

- Image version
- Storage size
- Resource limits
- Service type
- Probe parameters
- Authentication method

### 3. More Suitable for Environment Reuse
The same chart can be reused across:
- Development environment
- Testing environment
- Pre-production environment
- Production environment

### 4. More Suitable for Upgrades and Rollbacks
Helm inherently provides installation, upgrade, and rollback processes, offering a more unified management experience than scattered YAMLs.

---

## VIII. Limitations of Helm

### 1. Easy to "Can Install but Can't Understand"
This is one of the most common issues.

Many people will:

    helm install mysql ...

But don't know what objects the chart actually creates behind the scenes.

### 2. Changing values doesn't mean true understanding of deployment models
Without understanding the relationships between StatefulSet, PVC, ConfigMap, and Secret, just changing values will leave you passive when encountering issues.

### 3. Deep encapsulation makes it hard to quickly locate issues
Especially when:
- Many templates
- Many conditional judgments
- Deep values hierarchy
- Many rendered objects

This increases reading costs.

### 4. Helm primarily solves "delivery packaging issues"
It doesn't inherently solve database lifecycle management issues.

For example:
- Automatic failover
- Master-slave role orchestration
- Complex backup strategies
- Runtime state coordination

These aren't just "template installation" can solve.

## IX. What Are the Core Features of an Operator

The core idea of an Operator is not "YAML templating", but:

> **Putting the operational logic of a complex system into a controller, managed continuously by the controller.**

In other words, an Operator is not just helping "install a database", but rather trying to manage further:

- Cluster state
- Membership relationships
- Lifecycle
- Operational actions
- Backup and recovery
- Upgrade process
- Fault handling logic

---

### A More Accurate Understanding

Helm is more like:

> "Helping to put objects into the cluster"

An Operator is more like:

> "Continuing to monitor it after installation and managing it according to rules"

---

## X. Advantages of an Operator

### 1. More Suitable for Complex Database Systems
Especially when the database is no longer a "single-instance exercise", but:

- Clustered
- High availability
- Requires backup and recovery
- Requires membership state management
- Requires complex upgrade processes

At this point, an Operator becomes more meaningful.

### 2. More Suitable for Long-Term Lifecycle Management
An Operator's focus is not just one-time object creation, but continuous reconciliation of the target system.

### 3. More Suitable for Embedding Domain Knowledge into Controllers
For example, database-related:
- Initialization logic
- Membership relationships
- Fault recovery logic
- Backup and recovery logic
- Upgrade logic

All of these may be taken over by an Operator.

### 4. Closer to the Real-World Operation of Complex Stateful Systems
Database clusters, messaging systems, storage systems, and similar components often fit this model better than typical applications.

---

## XI. Limitations of an Operator

### 1. Higher Learning Curve
If the basic object relationships aren't clear, looking directly at an Operator may result in "knowing how to use it but not understanding what's happening."

### 2. Deeper Encapsulation
Compared to Helm, an Operator often adds another layer of controller behavior.

### 3. Troubleshooting No Longer Just Involves the Object Itself
You also need to look at:
- Operator Pod logs
- Custom Resource status
- Controller reconciliation behavior
- CRD field meanings

### 4. Not Suitable for Replacing Basic Understanding
An Operator is suitable after understanding object relationships, not as a replacement for foundational learning.

---

## XII. How to Remember the Core Differences Between the Three Approaches

You can use the following table to quickly grasp the concepts:

| Method | Core Function | Best for Solving What |
|---|---|---|
| Handwritten YAML | Directly define objects | Learning object relationships, minimal model practice |
| Helm | Templating to install objects | Standardized installation, parameterized delivery |
| Operator | Controller continuously manages systems | Complex lifecycle, stateful system management |

Another more straightforward way to put it:

### Handwritten YAML
Focus is on:

> **What the object is**

### Helm
Focus is on:

> **How to deliver objects in bulk and standardized ways**

### Operator
Focus is on:

> **How the system is continuously managed after object delivery**

---

## XIII. In the MySQL Scenario, Which Approach Is Suitable for Which Stage

### 1. When Is Handwritten YAML Suitable?
More suitable for:
- Learning the basic deployment model of MySQL
- Understanding the relationships between StatefulSet, PVC, ConfigMap, Secret, and Service
- Single-instance practice
- Understanding the basic object boundaries of stateful deployments

### 2. When Is Helm Suitable?
More suitable for:
- Rapid installation of a standardized MySQL solution
- Managing parameters with values
- Reuse across environments
- Learning the organization of mature charts

### 3. When Is an Operator Suitable?
More suitable for:
- Managing more complex MySQL clusters
- Understanding long-term lifecycle management of databases
- Exposure to more production-like database delivery methods

---

## XIV. Which Approach Should Be Prioritized at This Stage

The most suitable order at this stage is:

### Step 1: Master Handwritten YAML First
Because this step determines whether you truly understand:
- Why StatefulSet is needed
- Why PVC is needed
- Why configuration should use ConfigMap
- Why passwords should use Secret
- Why applications access the database through Service

### Step 2: Then Look at Helm
Because you already know what objects the chart is rendering behind the scenes, you won't just "install it but not explain it."

### Step 3: Finally Understand Operators
Because you now have a solid foundation in:
- Basic object relationships
- Helm's encapsulation logic
- Database operation models

### A Fundamental Conclusion
A more reasonable learning path is not:

> Operator > Helm > YAML

But rather:

> YAML > Helm > Operator

---

## XV. Why It's Not Recommended to Skip Handwritten YAML and Directly Learn Helm or Operator

Because skipping the object layer will lead to very typical problems later.

### 1. Unable to Understand What Charts Generate
Only executing Helm commands without knowing what objects are created.

### 2. Unable to Understand What Values Control
For example:
- persistence
- auth
- primary
- secondary
- service
- resources

The meaning of these configuration items becomes unclear.

### 3. Unable to Understand the Control Boundaries of Operators
Not knowing:
- What resources they help create
- What lifecycle they take over
- What relationship they have with StatefulSet, Service, and PVC

### Operational Understanding Focus
The value of handwritten YAML is not requiring long-term manual writing of everything, but to establish:

> **A clear understanding of the underlying object relationships.**

---

## XVI. A More Practical Choice Method

In real environments, a more reasonable choice is usually not "always using one method", but selecting based on the target.

### Scenario 1: Learning and Experimentation
Prioritize:
- Handwritten YAML

Because you need to see the object relationships clearly.

### Scenario 2: Standardized Installation of Middleware
Prioritize:
- Helm

Because it's more suitable for standardized installation and parameter management.

### Scenario 3: Complex Database Clusters and Long-Term Management
Prioritize:
- Operator

Because it's more suitable for continuous management of complex stateful systems.

### A More Practical Judgment Standard
Instead of asking:

> "Which is the most advanced?"

Ask:

> "Is the current problem more about object definition, delivery packaging, or lifecycle management?"

---

## XVII. The Most Important Cognitive Takeaways from This Article

### 1. Handwritten YAML, Helm, and Operators Are Not Mutually Exclusive in Terms of Rank
They solve problems at different levels.

### 2. Handwritten YAML Is Most Suitable for Understanding Basic Object Relationships
This is the most important value at this stage.

### 3. Helm Mainly Solves Standardized Installation and Parameterized Management Issues
It makes delivery more efficient, but doesn't automatically mean deeper understanding.

### 4. Operators Are More Suitable for Continuous Management of Complex Stateful Systems
The focus is not just "installing it", but "continuously managing it."

### 5. The Reasonable Learning Order at This Stage Is YAML → Helm → Operator
This makes it easier to build stable understanding.

---

## XVIII. Stage Summary

The three common delivery methods for MySQL in Kubernetes can be simplified into three layers:

- Handwritten YAML: Understanding Basic Object Relationships
- Helm: Understanding Standardized Installation and Parameterized Delivery
- Operator: Understanding Complex Lifecycle and Continuous Management

From the learning path perspective, handwritten YAML should not be viewed as a "backward method," but rather as:

> **The necessary stage for understanding the underlying logic of stateful application deployment.**

Helm and Operator are more suitable for extending toward "more realistic delivery methods" based on this foundation.

---

## Nineteen. Keyword Mnemonics

- Handwritten YAML: Directly Maintaining Object Relationships
- Helm: Templating and Parameterized Delivery
- Operator: Controller-Driven Continuous Lifecycle Management
- StatefulSet: Stateful Database Hosting Object
- values: Helm Parameter Entry Point
- CRD: Common Custom Resource Foundation for Operator
- reconcile: Core Concept of Operator Control Loop

---

## Twenty. Operational Extension Understanding

The differences in database deployment approaches essentially reflect "who bears the complexity."

### Handwritten YAML
The complexity is primarily borne by the deployer themselves.

### Helm
The complexity is partially handled by the chart template.

### Operator
The complexity is further managed by the controller.

Therefore, the focus of learning these methods should not merely be "whether you can use the commands," but rather understanding:

- Which complexity is exposed
- Which complexity is encapsulated
- Which complexity still requires your understanding and bearing

Only by clearly seeing this layer will your judgment be more stable when choosing delivery methods in real environments.

---

## References
- Kubernetes StatefulSet Official Documentation
- Helm Official Documentation
- Kubernetes Operator Pattern Official Documentation
- MySQL Related Helm Chart / Operator Documentation

---

## Next Day Recommendation
Next post suggestion to organize:

[[Stateful App Deployment Summary - Migrating to Middleware (MySQL Example)]]