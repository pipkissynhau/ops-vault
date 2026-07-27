# 01-Application Release Basics: Why Is the Ability to Release, Update, and Roll Back Needed Even After Deployment

## Document Description
- Document Purpose: Introduction to application release, update, and rollback processes
- Applicable Phase: After completing stateless and stateful application deployments, node-level applications, and basic Helm management, move on to application change and release management
- Recommended Path: `04-Kubernetes/07-Application Deployment/10-Application Release, Update, and Rollback/01-Application Release Basics: Why Is the Ability to Release, Update, and Roll Back Needed Even After Deployment`

## Tags
#Kubernetes #Application Release #Application Update #Application Rollback #Deployment #Rollout #Helm #Release #Change Management #Cloud-Native #Operations

---

## I. Why “Completed Application Deployment” Does Not Mean “End of Application Management”

In the previous sections on application deployment, the following tasks were completed:

- Stateless application deployment
- Stateful application deployment
- Node-level application deployment
- Helm and package management

These steps primarily addressed:

- How to deploy applications in Kubernetes
- Which workload models should be used for different types of components
- How Helm manages a set of objects as an application

However, in real-world scenarios, applications are never “deployed once and done.”

After an application goes live, many changes will continue to occur, such as:

- Image version updates
- Configuration adjustments
- Number of replicas changes
- Resource limit modifications
- Domain name and entry point changes
- Dependency component version updates
- Security fixes
- Rolling back to a previous version

Therefore, application management in Kubernetes includes not only “deployment” but also another equally important aspect:

> **How to release, update, verify, and roll back applications.**

---

## II. What Is Application Release

For now, we can define “application release” as follows:

> **Delivering a prepared application version into the target environment in a controlled manner.**

The term “version” here does not necessarily refer only to the image version but may also include:

- Image tag
- Configuration version
- Contents of the values file
- Changes in resource object definitions
- Updates to Secret/ConfigMap
- Changes in Helm Chart versions

Thus, application release is not a one-time action but more of a change process.

### Basic Understanding
You can think of application release as follows:

- The current state of the application is State A.
- It needs to be changed to State B.
- This change must be done in a controlled manner.

### Key Points for Operations Professionals
The focus of “release” is not just applying objects but ensuring that:

> **The application can safely switch from the old state to the new state.**

---

## III. Why Is the Ability to Release Needed Even After Deployment

This is one of the most important questions at this stage.

Simply deploying an application usually only addresses:

- The initial deployment of the application into the cluster

However, common real-world issues include:

- How to update an already running application
- How to confirm that updates are successful
- How to roll back to a previous version if problems occur
- How to prevent a single change from causing service disruptions
- How to verify after a Helm release upgrade
- How to monitor the status during a Deployment rollout

These issues fall under the category of “release and change management,” not just “initial deployment.”

### Basic Conclusion
Application deployment ensures that:

> **The application exists**

Application release, update, and rollback ensure that:

> **The application can continue to change in a controlled manner.**

---

## IV. Why Are Application Updates Frequent in Kubernetes

Once an application enters the normal iteration phase, updates become routine.

### Common Update Scenarios Include

#### 1. Image Updates
For example:
- Bug fixes
- Framework version upgrades
- Security patch applications
- New feature releases

#### 2. Configuration Updates
For example:
- Changes to environment variables
- Modifications to ConfigMap
- Adjustments to log levels
- Changes in resource limits
- Modifications to probe parameters

#### 3. Changes in the Number of Running Replicas
For example:
- Increasing replicas from 2 to 4
- Temporary scaling out
- Scaling out during peak usage
- Shrinking after a failure is resolved

#### 4. Changes in Delivery Methods
For example:
- Moving from manual YAML configuration to Helm management
- Updates to Helm values files
- Switching between Chart versions

### Key Points for Operations Professionals
In Kubernetes application management, the most frequent activities are often not “initial deployments” but:

> **Continuous updates.**

---

## V. Why “Update” Cannot Be Simply Understood as “Reapplying Once”

At first glance, many changes seem achievable through:

    kubectl apply -f xxx.yaml

However, if we only focus on this step, we may overlook the following critical issues:

-### 3. The ability to only perform updates without rollback capabilities is an incomplete release capability
This is the third understanding.

### 4. Release, update, and rollback should be understood as a closed loop
This is the fourth understanding.

### 5. Helm and native object capabilities will converge at this stage
This is the fifth understanding.

---

## Thirteen, Key Points of This Article

This article does not delve into the specific details of rollout commands but establishes the following main idea:

### 1. Why deploying an application alone is far from sufficient
Because applications are constantly evolving.

### 2. Why updates must be controllable
Because businesses cannot afford the risks associated with abrupt replacements.

### 3. Why rollback capabilities are essential
Because failures in changes are inevitable.

### 4. Why verification is part of the release closed loop
Because what truly matters is whether the results are usable.

### 5. Why this stage is ideally placed after Helm
Because we now have:
- A foundation in objects
- A foundation in application package management

---

## Fourteen, Phase Summary

Application deployment addresses:

- How to deploy applications into Kubernetes

Helm addresses:

- How to install and manage applications as packages

While application release, update, and rollback deal with:

- How to manage ongoing changes in applications
- How to verify these changes
- How to revert to previous stable versions in case of issues

Therefore, the core focus of this phase is not about learning additional commands but about establishing a comprehensive understanding of change management:

> **Release is not just about submitting changes; it’s about ensuring that changes are introduced into production in a controlled manner.**

---

## Fifteen, Key Terms for Quick Reference

- Application release: Delivering a specific version to the target environment
- Application update: Changing the current state to a new one
- Application rollback: Returning the system to a previous working state
- Change closed loop: Update, verify, and rollback in case of failure
- Gradual change: Replacing components gradually rather than abruptly
- Release verification: Confirming whether the changes meet expectations
- Version control: Making changes traceable and reversible

---

## Sixteen, Extended Understanding for Operations and Maintenance

In the main thread of Kubernetes application management, the learning process generally goes through three stages:

### First Stage: Object Layer
Focus on:
- Deployment
- StatefulSet
- DaemonSet
- Service
- ConfigMap
- Secret

### Second Stage: Application Package Layer
Focus on:
- Helm
- Chart
- Release
- values

### Third Stage: Change Management Layer
Focus on:
- Release
- Update
- Rollback
- Verification
- Version history

The current stage marks the transition from being able to deploy applications to managing their continuous changes.

It also represents a shift for operations and maintenance personnel from being mere users of objects to becoming managers of application deliveries.

---

## References
- Kubernetes Official Documentation
- Helm Official Documentation

---

## Next Day's Suggestion
For the next article, it is recommended to organize content around:

[[02-Deployment Rolling Updates Basics: Image Updates, Rollout, Revision, and Status Checks]]