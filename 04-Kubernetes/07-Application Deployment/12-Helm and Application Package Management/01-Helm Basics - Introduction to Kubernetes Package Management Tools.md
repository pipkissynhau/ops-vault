# 01-Helm Basics: Introduction to Kubernetes Package Management Tools

## Document Description
- Document Purpose: An introduction to Helm's core concepts and use cases
- Target Audience: Those who have completed basic deployments of stateless, stateful, and node-level applications in Kubernetes and are ready to advance to managing application packages
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/11-Helm and Application Package Management/01-Helm Basics: Introduction to Kubernetes Package Management Tools`

## Tags
#Kubernetes #Helm #Chart #Release #Values #Application Package Management #Deployment Delivery #Cloud-Native #Ops

---

## I. Why Does Kubernetes Need Helm?

In the previous sections on application deployment, you have already encountered many Kubernetes objects, such as:

- Deployment
- StatefulSet
- DaemonSet
- Service
- ConfigMap
- Secret
- PVC
- Ingress

These objects can all be created directly by writing YAML files and using `kubectl apply`.  
This approach is suitable for learning about object relationships and for basic practice with small numbers of resources.

However, in a real-world environment, a complete application typically consists of more than just one or two objects. It includes a whole set of resources, such as:

- Workload objects
- Configuration objects
- Secret objects
- Services
- Ingresses
- Persistent storage
- RBAC settings
- ServiceAccounts
- PodDisruptionBudgets
- HPA configurations
- Monitoring-related objects

If all these objects are manually written and maintained over time, common issues will become increasingly apparent:

### 1. Large number of objects
A moderately complex application usually cannot be managed with just one YAML file.

### 2. Different environmental parameters
In development, testing, pre-production, and production environments, there may be differences in:

- Image versions
- Resource limits
- Service types
- Number of replicas
- Domain names
- Storage sizes

### 3. High cost of repetitive maintenance
When multiple environments, components, and versions are managed simultaneously, the number of duplicate YAML files tends to increase significantly.

### 4. Inconsistent upgrade, rollback, and uninstallation processes
Relying solely on scattered YAML files allows for deployment, but it makes change management and version control less efficient.

Therefore, in Kubernetes, in addition to knowing "how to write objects," there is also a need for a mechanism that is closer to the actual delivery process:

> **How to package a set of related objects, parameterize them, and manage them as a single application.**

Helm is designed to address these challenges.

---

## II. What is Helm?

Helm is a very common application package management tool in Kubernetes.

Here is a basic understanding of it:

> **Helm is used to organize a set of Kubernetes resources into an installable, upgradable, roll-backable, and uninstallable application package.**

There are several key points in this definition:

### 1. It manages groups of objects, not individual ones
For example, a MySQL Chart might include:

- A StatefulSet
- A Service
- Secrets
- ConfigMap configurations
- Related PVC settings

### 2. It focuses on "applications" rather than single YAML files
When using Helm, you usually focus on:

- Whether the application has been installed
- What the current version is
- What the parameters are
- How to upgrade it
- How to roll it back

### 3. It parameterizes Kubernetes objects
For instance:

- The number of replicas
- The image tag
- Whether persistent storage is enabled
- The Service type
- Resource limits

All these can be managed through values files, rather than by copying multiple YAML files.

### 4. It provides a unified interface for operations
Common actions include:

- Installation
- Querying
- Upgrading
- Rolling back
- Uninstalling

---

## III. How Can Helm Be Interpreted?

Helm can be most easily understood as:

> **A Kubernetes application package management tool.**

If you think of Kubernetes objects as individual parts, then Helm is like:

- Packing these parts into an installable package
- Allowing customization of the installation results through parameters
- Enabling unified upgrades, rollbacks, and uninstallations later on

### A simple analogy
You can think of it this way for easier understanding:

- `kubectl apply` is more like "directly submitting object definitions"
- Helm is more like "installing a configurable application package"

### What does this mean?
Helm does not replace Kubernetes' native objects;  
it still operates on the basis of:

- Deployment
- StatefulSet
- Service
- ConfigMap
- Secret

However, its role is to:

> **Organize these objects at the application level.**

---

## IV. What Problems Does Helm Mainly Solve?

The value of Helm does not lie in "inventing new objects> **The deployment results can be customized through values.**

---

## Eleven: What is the relationship between Helm and handwritten YAML?

This question is very important.

Helm does not replace Kubernetes' native objects; rather, it serves as an application delivery tool built on top of these native objects.

### When is handwritten YAML more suitable?
It is more suitable for:
- Learning about object relationships
- Understanding how Deployment, StatefulSet, Service, ConfigMap, and Secret work together
- Building minimal models
- Conducting small-scale exercises

### When is Helm more suitable?
It is more suitable for:
- Managing a set of objects
- Parameterized deployment
- Reusing deployment templates
- Standardizing the installation of middleware and platform components
- Performing upgrades, rollbacks, and uninstallations

### A basic conclusion
Handwritten YAML focuses more on:

> **Understanding the objects themselves**

Helm focuses more on:

> **Organizing and managing objects at the application level**

### Key points for operations engineers to understand
Without a foundation in object concepts, merely knowing how to use Helm commands often leads to:
- Being able to install but not understand or modify anything
- Being unable to troubleshoot issues

Therefore, a more logical learning order is usually:

> **First, understand Kubernetes objects, then learn Helm.**

---

## Twelve: How can we understand the differences between Helm and `kubectl apply`?

Both can deploy resources into Kubernetes, but they focus on different aspects.

### `kubectl apply`
It tends to be more focused on:
- Submitting object definitions
- Managing a single or a few YAML files
- Focusing on the resource objects themselves

### Helm
It tends to be more focused on:
- Installing an application package
- Managing a set of objects
- Managing the entire lifecycle of an application

### A comparative example
If you are only deploying a simple test Deployment, using:

    kubectl apply -f demo.yaml

is likely sufficient.

However, if you are deploying a middleware that involves:

- StatefulSet
- Service
- Secret
- ConfigMap
- PVC
- RBAC
- Monitoring objects

then the value of Helm becomes much clearer.

### Key points for operations engineers to understand
You can think of the difference between the two this way:

> `kubectl apply` is more like “submitting objects”, while Helm is more like “installing applications”.

---

## Thirteen: Why is it appropriate to learn Helm at this stage?

By now, you have already completed:

- Deploying stateless applications
- Deploying stateful applications
- Deploying node-level applications

At this point, you should have a clear understanding of:
- How to write objects
- How objects work together
- What different workload models solve what problems

Introducing Helm at this stage allows you to reorganize the objects you have learned so far into a delivery method that is closer to real-world work.

### Key points for learning Helm at this stage
The focus should not be on:

- Developing complex Charts from scratch
- Delving deeply into all template syntax

Instead, the focus should be on:

- What Helm is
- What Helm manages
- What Chart, Release, Repository, and Values are
- Why middleware and platform components are commonly managed using Helm
- How to use Helm for unified installation, upgrades, and rollbacks in the future

### A basic conclusion
At this stage, the focus of learning Helm should be on:

> **Understanding how Helm organizes a set of Kubernetes objects into a manageable application.**

---

## Fourteen: The most important insights from this article

### 1. Helm is a Kubernetes application package management tool
This is the first key insight.

### 2. Helm manages not individual objects but a set of objects
This is the second key insight.

### 3. Chart, Release, Repository, and Values are Helm's four most core concepts
This is the third key insight.

### 4. The focus of Helm lies in application delivery and lifecycle management
This is the fourth key insight.

### 5. Helm is built on top of Kubernetes' native objects but does not replace the basic understanding of these objects
This is the fifth key insight.

---

## Fifteen: Summary of this chapter

The value of Helm lies not in creating new types of Kubernetes workloads but in reorganizing existing Kubernetes objects into application packages that are more suitable for deployment, change management, and reuse.

To fully understand Helm, you need to establish the following basic concepts:

- Helm is a package management tool
- Chart is an application package
- Release is the actual installed instance of a Chart
- Repository is where Charts are stored
- Values are parameters used in Chart templates
- Helm is particularly useful for managing middleware, platform components, and groups of objects

Once these concepts are solid, learning further topics such as:

- Common Helm commands
- The practical role of values.yaml
- Practical experiences with installing, upgrading, rolling