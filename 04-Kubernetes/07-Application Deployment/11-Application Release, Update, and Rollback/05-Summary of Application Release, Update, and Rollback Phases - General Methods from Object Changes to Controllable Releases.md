# 05-Summary of Application Release, Update, and Rollback Phases: General Methods from Object Changes to Controllable Releases

## Document Description
- Document Purpose: To provide a summary and method guide for application release, update, and rollback phases.
- Applicable Phase: After completing the basics of application release, Deployment rolling updates, image and configuration changes, and rollback fundamentals, this document serves as a phase-specific summary.
- Recommended Path: `04-Kubernetes/07-Application Deployment/10-Application Release, Update, and Rollback/05-Summary of Application Release, Update, and Rollback Phases: General Methods from Object Changes to Controllable Releases`

## Tags
#Kubernetes #Application Release #Application Update #Application Rollback #Deployment #Rollout #Helm #Release #ConfigMap #Change Management #Release Validation #Cloud-Native #Operations and Maintenance

---

## I. What Was Achieved in This Phase

The focus of the application release, update, and rollback section is not to redefine Deployment, Service, ConfigMap, or Helm themselves, but rather to integrate the objects and delivery methods learned earlier into the context of “change management.”

This phase has covered the following key areas:

- Why release, update, and rollback capabilities are still necessary after an application is deployed in Kubernetes.
- The basics of Deployment rolling updates.
- The differences between image changes and ConfigMap changes.
- The basic usage scenarios of `kubectl rollout undo` and `helm rollback`.

The main problem solved by this section can be summarized as:

> **How to ensure that an application running in Kubernetes can be safely changed, observed, and rolled back if necessary.**

The focus here is no longer just on:

- How to define objects.
- How to deploy applications.
- How to use Helm for installation.

Instead, it shifts more towards:

- How to initiate changes.
- How to monitor the change process.
- How to verify the results of the changes.
- How to roll back in case of failures.
- How to turn an object modification into a controllable release.

---

## II. The Most Important Overall Understanding in This Phase

Application management in Kubernetes should not stop at just “deployment.”

Deployment addresses the issue of:

> **How an application enters the cluster.**

However, release, update, and rollback deal with:

> **How to ensure that an application continues to change within the cluster without getting out of control.**

This means that the main line of application management can be divided into at least three layers:

### Layer 1: Deployment
Examples include:
- Deployment
- StatefulSet
- DaemonSet
- Service
- Ingress
- ConfigMap
- Secret

### Layer 2: Application Package Management
Examples include:
- Helm
- Chart
- Release
- values.yaml

### Layer 3: Change Management
Examples include:
- rollout
- revision
- image updates
- configuration changes
- release validation
- rollback recovery

### A Basic Conclusion
By the end of this phase, you should have established the following overall understanding:

> **Deployment is not the end; change is the norm.**

---

## III. From What Perspective Was “Release” Clarified in This Phase

In many learning paths, “release” can be understood as a vague term:

- Launching a new version.
- Going live.
- Updating an application.

However, this phase has broken down “release” into more specific components.

### Current Understanding of Release
Release is not a single command but a controlled change process.

It includes at least the following steps:

- What the current state is.
- What the desired new state is.
- How to implement the change.
- How to verify the results of the change.
- How to revert back to the previous state if necessary.

### Therefore, the focus of release is not:
- Directly `apply`.
- Directly `set image`.
- Directly `upgrade`.

But rather:

> **To ensure that the new state enters the production environment in a controlled manner.**

---

## IV. What Object-Level Release Understanding Was Established Through Deployment

Deployment is the most common stateless business workload, making it an ideal first case for object-level release management.

Through Deployment, the following understandings were established:

### 1. Changes to Pod Template Trigger New Updates
For example:
- Changes in image tag.
- Changes in environment variables.
- Changes in resource limits.
- Changes in probes.
- Changes in annotations.

These changes essentially lead to modifications of the Pod Template, which in turn trigger new ReplicaSets and rollout processes.

### 2. Release Does Not Directly Replace Pods but Involves ReplicaSet Handover
The essence of Deployment updates is not “directly replacing old Pods with new ones” but rather:

- Creating a new ReplicaSet.
- Gradually handing over replicas between the old and new ReplicaSets.
- Eventually having the new ReplicaSet handle all traffic.

### 3. Rollout Is a### 5. Understanding the Differences Between Object-Level and Application-Level Rollbacks
Know when to use:
- `kubectl rollout undo`
- `helm rollback`

### Key Points for Operations and Maintenance Professionals
The essence of this capability is:

> **Evolving from "knowing how to issue commands" to "being able to manage changes."**

---

## XI. How This Phase Connects with the Previous Helm Section
The Helm section addressed:

- How applications are packaged
- How applications are installed, upgraded, and uninstalled
- How values affect deployment outcomes

This part of application release, update, and rollback further addresses:

- What to do after an upgrade
- When to use a rollback
- How object-level and application-level processes correspond
- How to integrate Helm and kubectl perspectives

### A Unified Understanding
The relationship between these two parts can be summarized as:

- Helm handles the delivery of applications
- The release process manages the entire change management cycle

This is why this phase naturally follows Helm.

---

## XII. How This Phase Connects with Subsequent Practical Applications
Up to this point, the content has focused on "concepts and basic methods."

We have already covered:

- Stateless application deployment
- Stateful application deployment
- Node-level application deployment
- Helm and application package management
- Application release, update, and rollback

The natural next step is to move on to:

- Practical containerized deployment of common middleware and platforms
- Application deployment troubleshooting
- Case studies of business and platform deployments

### Why This Connection Makes Sense
Because we now possess these foundational capabilities:

- We can analyze workload models
- We understand object relationships
- We can deliver applications using Helm
- We can handle basic updates and rollbacks

Next, we can apply these skills in real-world components and scenarios.

---

## XIII. The Final Conclusion of This Phase
The ultimate goal of application release, update, and rollback is not to memorize individual commands, but to develop a set of portable deployment methods.

This method includes at least:

- Identifying the type of change
- Choosing the appropriate change approach
- Monitoring the rollout process
- Verifying object-level, application-level, and business-level outcomes
- Performing a rollback if expectations are not met
- Reverifying the results after rollback

The core objective of this method is not to "force changes," but to ensure that changes occur in a controlled manner.

With this, the **10-Application Release, Update, and Rollback** phase can be considered complete for now.

---

## XIV. Key Terms for Quick Reference
- Release: Delivering a new state to the target environment
- Update: Changing the current state to a new version or configuration
- Rollback: Returning to a previous stable state
- Rollout: The process of delivering updates in stages
- Revision: A historical change in a Deployment
- Release revision: A historical change in a Helm Release
- Release verification: Confirming that the new state is functional
- Controlled release: A change process that is observable, verifiable, and reversible

---

## XV. Further Insights for Operations and Maintenance Professionals
By this stage of Kubernetes learning, the main focus has shifted significantly.

Initially, we focused on:

- What objects are and how to define them
- How applications are deployed into clusters

Later, we learned about:

- What Charts and Releases are and how to use them
- How to modify values and package applications

Now, as we move onto release, update, and rollback, our focus shifts to:

- How applications can be continuously changed
- How to monitor the change process
- How to verify the results of changes
- How to recover from failed changes

This indicates that the learning focus has gradually shifted from **understanding deployment objects** to **comprehending the application lifecycle and change management processes.**

This step is crucial because in practical applications, troubleshooting, and case studies, most issues arise in systems that are constantly changing, rather than in statically defined objects.

---

## References
- Kubernetes Deployment Official Documentation
- Helm Official Documentation
- Kubernetes Official Documentation

---

## Next Steps
It is recommended to proceed to:

**《11-Practical Containerized Deployment of Common Middleware and Platforms》**