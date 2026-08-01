# 01-Helm Basics: Kubernetes Package Management Tool Introduction

## Document Notes
- Document Focus: Helm core concepts and usage scenarios introduction
- Applicable Stage: After completing stateless application, stateful application, and node-level application deployment basics, enter Kubernetes application package management mainline
- Recommended Path: `04-Kubernetes/07-Apply deployment/11-Helm and application package management/01-Helm Basis:Kubernetes Introduction to Package Management Tool`

## Tags
#Kubernetes #Helm #Chart #Release #Values #ApplyPackageManagement #DeploymentDelivery #Clouds. #Transport

---

## I. Why Kubernetes Still Needs Helm

In the previous application deployment mainline, many Kubernetes objects have already been encountered, such as:

- Deployment
- StatefulSet
- DaemonSet
- Service
- ConfigMap
- Secret
- PVC
- Ingress

These objects can all be directly written in YAML and created via `kubectl apply`.  
This approach is suitable for learning object relationships and also suitable for basic practice with small-scale resources.

But in real environments, a complete application often doesn't just contain one or two objects, but a whole set of resources, such as:

- Workload objects
- Configuration objects
- Secret objects
- Service
- Ingress
- Persistent storage
- RBAC
- ServiceAccount
- PodDisruptionBudget
- HPA
- Monitoring-related objects

If all these objects are handwritten and maintained long-term, common issues will gradually become apparent:

### 1. Large number of objects
A medium-complexity application often cannot be resolved by a single YAML file.

### 2. Different environment parameters
Development, testing, pre-release, and production environments have different:
- Image versions
- Resource limits
- Service types
- Replica counts
- Domains
- Storage sizes

### 3. High cost of repeated maintenance
When multiple environments, multiple components, and multiple versions are running in parallel, repeated YAML files are easy to proliferate.

### 4. Inadequate standardization for upgrades, rollbacks, and uninstallation
Although deploying via scattered YAML files can still complete deployment, change management and version control are not centralized enough.

Therefore, in Kubernetes, besides "how to write objects," there's a need for a layer closer to actual delivery capabilities:

> **How to package a group of related objects, parameterize them, and manage them as an application.**

Helm solves exactly these kinds of problems.

---

## II. What is Helm

Helm is a very common application package management tool in Kubernetes.

You can first have a basic understanding of it:

> **Helm is used to organize a group of Kubernetes resources into an installable, upgradable, rollable, and uninstallable application package.**

This statement contains several key points:

### 1. It manages not individual objects but a group of objects
For example, a MySQL Chart may include:

- StatefulSet
- Service
- Secret
- ConfigMap
- PVC-related configurations

### 2. It emphasizes "applications" rather than "individual YAML"
When using Helm, you typically no longer just care about a specific Deployment, but rather:

- Whether the application is installed
- What the current version is
- What the parameters are
- How to upgrade
- How to rollback

### 3. It parameterizes Kubernetes objects
For example:
- Replica count
- Image tag
- Whether persistent storage is enabled
- Service type
- Resource limits

All can be managed through values instead of duplicating YAML files.

### 4. It provides a unified operation entry point
Common actions include:
- Installation
- Query
- Upgrade
- Rollback
- Uninstallation

---

## III. How to Understand Helm

The most intuitive understanding of Helm is:

> **Kubernetes's application package management tool.**

If you view Kubernetes objects as scattered components, Helm is more like:

- Packaging components into an installation package
- Allowing parameterized customization of the installation result
- Allowing unified upgrades, rollbacks, and uninstallations later

### A Simple Analogy
You can first memorize it in a not strictly accurate but easy-to-understand way:

- `kubectl apply` is more like "directly submitting object definitions"
- Helm is more like "installing a configurable application package"

### What This Means
Helm does not replace Kubernetes native objects,  
It still operates based on:

- Deployment
- StatefulSet
- Service
- ConfigMap
- Secret

These objects.

What it does is more akin to:

> **Organizing these objects by application dimensions.**

---

## IV. What Problems Does Helm Mainly Solve

Helm's value doesn't lie in "inventing new objects," but in managing object collections.

You can first summarize the problems it solves as follows.

### 1. Object Packaging Problem
Organizing a group of originally scattered Kubernetes resources into an application package.

### 2. Parameterization Problem
Allowing the same template to adapt to different environments without duplicating YAML files.

### 3. Version and Change Management Problem
Providing clearer operation methods for application installation, upgrades, and rollbacks.

### 4. Reusability Problem
Many common components don't need to be written from scratch in YAML, but can directly use existing Charts.

### 5. Application Lifecycle Management Problem
Treating the application as a whole for:
- Installation
- Inspection
- Upgrade
- Rollback
- Uninstallation

### A Fundamental Conclusion
Helm's focus is not "writing more complex YAML," but:

> **Making Kubernetes application delivery and management more like a complete application process.**

---

## V. Why Helm Is Particularly Suitable for Middleware and Platform Components

Although Helm can also package ordinary business applications, it is especially common in middleware and platform component scenarios.

### Common Reasons

#### 1. Middleware Objects Usually Number Many
For example, deploying a common component often involves not just one workload object, but also:
- Configuration
- Secrets
- Service
- Persistent storage
- Permissions
- Probes
- Monitoring objects

#### 2. Middleware Parameters Usually Number Many
For example:
- Whether persistent storage is enabled
- Storage size
- Replica count
- Service type
- Resource limits
- Security configuration
- Default password
- Data initialization parameters

#### 3. Middleware Has Many Repetitive Deployment Scenarios
Different environments may all need:
- Redis
- MySQL
- Nginx Ingress
- Prometheus
- Loki
- Elasticsearch
- MinIO

#### 4. More Need for Version Management
These components often aren't installed once and finished, but need:
- Upgrading versions
- Adjusting configurations
- Doing rollbacks
- Unified maintenance

### Operations Understanding Focus
Helm is commonly used in middleware scenarios because it is very suitable for solving:

> **The delivery problems of components with many objects, parameters, environments, and changes.**

---

## VI. Which Core Objects of Helm Should Be Remembered First

Helm has few core concepts, but they must be clearly distinguished.

At this stage, focus on the following four:

- Chart
- Release
- Repository
- Values

These four concepts will almostThrough the entire Helm usage process.

---

## VII. What Is a Chart

Chart is one of the most core concepts in Helm.

You can first understand it as: /think

> **Helm Application Packages**

A Chart typically includes:

- Template files
- Default parameters
- Chart metadata
- Relevant Kubernetes object definition templates

### A Basic Understanding
If Helm is a package management tool, then a Chart is the "package" it manages.

### Example
Common statements would be:

- Installing an nginx Chart
- Installing a mysql Chart
- Installing a prometheus Chart

At this point, the term "Chart" essentially refers to:

- A packaged definition of a set of objects
- A collection of templates for an application

### Operations Focus
A Chart is not "a single Deployment file," but rather:

> **A complete set of application templates.**

---

## VIII. What is a Release

Release is another crucial concept in Helm.

You can initially understand it as:

> **An actual installation instance of a Chart in a cluster.**

### Why Distinguish Between Chart and Release
Because the same Chart can be installed multiple times.

For example, the same nginx Chart can be installed as:

- `web-a`
- `web-b`
- `test-nginx`

These installation instances essentially share the same Chart, but differ in the cluster's actual name, parameters, and state.

### A More Intuitive Understanding

- Chart: The installation package itself
- Release: The actual instance after the package is deployed

### Common Scenarios
For example, executing:

    helm install my-nginx bitnami/nginx

Here:
- `bitnami/nginx` is the Chart
- `my-nginx` is the Release name

### Operations Focus
Many subsequent Helm operations are actually centered around Releases, such as:

- Viewing a Release
- Upgrading a Release
- Rolling back a Release
- Uninstalling a Release

---

## IX. What is a Repository

A Repository can initially be understood as:

> **A repository that stores Helm Charts.**

It is not the same as an image repository, but has a similar positioning:

- Image repositories store images
- Helm repositories store Charts

### What Problems Does a Repository Solve
It enables common Charts to be centrally maintained and distributed.

For example:
- nginx
- mysql
- redis
- prometheus

These common components often don't require writing full Chart from scratch, but can instead be obtained from existing repositories.

### Common Actions
Common operations related to repositories include:

- Adding a repository
- Updating repository indexes
- Searching for Charts in a repository

### Operations Focus
The value of a repository lies in:

> **Providing a source for ready-made Charts.**

---

## X. What are Values

Values are a critical layer in Helm.

You can initially understand them as:

> **A collection of configuration parameters used during Chart template rendering.**

### Why Values Are Important
Because the same Chart often needs to adapt to different environments, and these environmental differences shouldn't be resolved by duplicating template files.

For example, an application may have different requirements in different environments:

- Different replica counts
- Different image tags
- Different resource limits
- Different Service types
- Different domains
- Whether to enable persistence

These differences are typically controlled through values.

### A Basic Understanding
A Chart provides templates and default values,  
Values provide:

- The actual parameters to fill in
- Overrides to default values
- Customization of deployment results

### Operations Focus
The true focus of Helm usage isn't just `install`, but rather:

> **Customizing deployment results through values.**

---

## XI. What is the Relationship Between Helm and Handwritten YAML

This question is very important.

Helm is not a replacement for Kubernetes native objects, but rather a layer of application delivery tool built on top of native objects.

### When Handwritten YAML Is More Suitable
More suitable for:
- Learning object relationships
- Understanding how Deployment, StatefulSet, Service, ConfigMap, and Secret work together
- Building minimal models
- Small-scale practice

### When Helm Is More Suitable
More suitable for:
- Managing a group of objects
- Parameterized deployment
- Reusing deployment templates
- Standardizing installation of middleware and platform components
- Upgrading, rolling back, and uninstalling

### A Basic Conclusion
Handwritten YAML focuses more on:

> **Understanding the objects themselves**

Helm focuses more on:

> **Organizing and managing objects by application dimensions**

### Operations Focus
Without a foundation in object layers, relying only on Helm commands often results in:
- Knowing how to install
- Not knowing how to inspect
- Not knowing how to modify
- Not knowing how to troubleshoot

Therefore, a more reasonable learning sequence is usually:

> **First understand Kubernetes objects, then learn Helm.**

---

## XII. How to Understand the Differences Between Helm and kubectl apply

Both can put resources into Kubernetes, but they focus on different aspects.

### `kubectl apply`
More oriented toward:
- Submitting object definitions
- Managing individual or small YAML files
- Focused on resource objects themselves

### Helm
More oriented toward:
- Installing an application package
- Managing a group of objects
- Focused on the application's entire lifecycle

### A Comparative Example
If deploying a simple test Deployment, directly:

    kubectl apply -f demo.yaml

may already be sufficient.

But when deploying a middleware involving:

- StatefulSet
- Service
- Secret
- ConfigMap
- PVC
- RBAC
- Monitoring objects

Helm's value becomes more apparent.

### Operations Focus
You can remember the differences between the two with one sentence:

> `kubectl apply` is more like "submitting objects," Helm is more like "installing an application."

---

## XIII. Why Helm Is Suitable for Learning at This Stage

At this point, you've completed:

- Deploying stateless applications
- Deploying stateful applications
- Deploying node-level applications

At this stage, you have a clear understanding of:
- How to write objects
- How objects work together
- What different workload models solve

Entering Helm at this stage allows you to reorganize the previously learned objects into a delivery method closer to real-world work.

### Focus of Learning Helm at This Stage
Not focused on:
- Developing complex Charts from scratch
- Deep diving into all template syntax

But rather on:
- What Helm is
- What Helm manages
- What Charts, Releases, Repositories, and Values are
- Why middleware and platform components commonly use Helm
- How to use Helm to uniformly install, upgrade, and rollback later

### A Basic Conclusion
At this stage, the focus of learning Helm should be on:

> **Understanding how Helm organizes a group of Kubernetes objects into a manageable application.**

---

## XIV. The Most Important Several Understandings in This Article

### 1. Helm is a Kubernetes application package management tool
This is the first understanding.

### 2. Helm manages a group of objects, not individual objects
This is the second understanding.

### 3. Chart, Release, Repository, and Values are the four core concepts of Helm
This is the third understanding.

### 4. Helm focuses on application delivery and lifecycle management
This is the fourth understanding.

### 5. Helm is built on top of Kubernetes native objects and does not replace the understanding of object fundamentals
This is the fifth understanding.

---

## Fifteen, Phase Summary

The value of Helm lies not in inventing new Kubernetes workload types, but in reorganizing existing Kubernetes objects into application packages that are more suitable for delivery, change, and reuse.

Through this article, the following foundational understandings need to be established first:

- Helm is a package management tool
- Chart is an application package
- Release is an installed instance
- Repository is a Chart repository
- Values are deployment parameters
- Helm is more suitable for standardized management of middleware, platform components, and grouped objects

Once this conceptual layer is solidified, proceeding to the following topics will be much smoother:

- Helm common commands
- The actual role of values.yaml
- Helm installation, upgrade, rollback, and uninstallation practices

---

## Sixteen, Keyword Mnemonics

- Helm: Kubernetes application package management tool
- Chart: Helm application package
- Release: The actual installed instance of a Chart
- Repository: A repository storing Charts
- Values: Parameters used by Chart templates
- Application package management: Managing a group of objects as a unified application
- Parameterized deployment: Customizing deployment results through configuration items

---

## Seventeen, Operational Extended Understanding

After reaching a certain stage in Kubernetes learning, the focus will gradually shift from "how to write objects" to "how to deliver objects".

During this process:

- Handwritten YAML represents object-level understanding
- Helm represents application-level delivery
- Subsequent publishing, updating, and rolling back represent change-level management

These three layers are not replacement relationships, but progressive relationships.

Therefore, the significance of this part of Helm is not just learning a few commands, but completing a perspective shift within the overall framework:

> **From managing objects, to managing applications.**

---

## References
- Helm official documentation
- Kubernetes official documentation

---

## Next Day Suggestions
The next article suggests organizing:

[[02-Helm Common Commands - repo search install upgrade rollback and uninstall]]