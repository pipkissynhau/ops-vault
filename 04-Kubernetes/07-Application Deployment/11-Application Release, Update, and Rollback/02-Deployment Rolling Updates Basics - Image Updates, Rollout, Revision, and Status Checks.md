# 02-Deployment Rolling Updates Basics: Image Updates, Rollout, Revision, and Status Checks

## Document Description
- Document Focus: Basics of Deployment rolling updates and version changes
- Applicable Phase: After understanding the fundamentals of application deployment, move on to updating and rolling back native Kubernetes workloads
- Recommended Path: `04-Kubernetes/07-Application Deployment/10-Application Release, Updates, and Rollbacks/02-Deployment Rolling Updates Basics: Image Updates, Rollout, Revision, and Status Checks`

## Tags
#Kubernetes #Deployment #Rollout #Revision #Rolling Updates #Image Updates #Application Release #Application Rollback #Cloud-Native #Operations

---

## I. Why Learn About Deployment Updates Separately

In Kubernetes, Deployment is one of the most common stateless workload controllers. Many business services, such as:

- Java APIs
- Go services
- Python web services
- Nginx
- Frontend services

often run using Deployments.

Therefore, the natural starting point for understanding application release, updates, and rollbacks is:

> **How Deployment handles version changes.**

The most common change actions at this stage typically include:

- Updating image versions
- Adjusting the number of replicas
- Modifying environment variables
- Changing resource limits
- Adjusting probe parameters
- Altering labels or annotations

However, in the main deployment process, the most typical and easily observable action is still:

> **Image updates.**

Because an image update essentially represents a basic application version release.

---

## II. What Is Deployment Rolling Update?

The default update method for Deployments is rolling update.

Here’s a straightforward understanding:

> **Instead of deleting all old Pods and creating new ones at once, new versions of Pods are created gradually while the old ones are phased out.**

The goal of this approach is to:

- Make the application update process smoother
- Reduce the risk of interruptions
- Maintain a certain number of available replicas
- Ensure that the release process can be observed and controlled

### An Intuitive Example

Suppose a current Deployment has 3 replicas:

- `web-v1 Pod 1`
- `web-v1 Pod 2`
- `web-v1 Pod 3`

To update to a new image version, the rolling update approach would not be:

- Deleting all 3 old Pods at once
- Creating 3 new Pods

But rather:

- Starting a new version of a Pod first
- Waiting for it to become Ready
- Gradually replacing the old Pods
- Until all are new versions

### Key Points for Operations Professionals

The core of rolling updates is not “updating the object definition” but:

> **Gradually switching between old and new versions within a controlled range.**

---

## III. Why Rolling Updates Are More Reasonable Than Full Replacements

### 1. Reduced Business Interruption Risk

If all old Pods are deleted and new ones created immediately, the service is likely to be temporarily unavailable during the update.

### 2. Gradual Verification of New Versions

During rolling updates, you can observe:

- Whether new Pods start up normally
- Whether Readiness Probes pass
- If application logs are normal
- If error rates increase

### 3. Easier Problem Identification

Since old and new versions coexist briefly, it’s easier to pinpoint issues related to the specific change.

### 4. More Aligned with Real-Production Requirements

Production environments generally cannot tolerate a “stop-and-restart” approach unless it’s a deliberate downtime release scenario.

### Key Points for Operations Professionals

Rolling updates are not “more complex replacement methods” but:

> **A more suitable way to replace online services.**

---

## IV. What Are the Most Common Triggers for Deployment Updates?

Ultimately, any update to a Deployment is triggered by:

> **Changes to the Pod Template.**

As long as there are changes to the Pod template part of a Deployment, Kubernetes will recognize the need to create a new ReplicaSet and begin a new rolling update process.

### Common Triggers Include

#### 1. Image version changes
For example:

- Changing `nginx:1.27.0` to `nginx:1.27.1`

#### 2. Environment variable changes
Adding or modifying:

- `ENV`
- `LOG_LEVEL`

#### 3. Resource configuration changes
For example:

- Adjusting CPU requests
- Modifying memory limits

#### 4. Probe changes
For example:

- Changing readinessProbes
- Adjusting livenessProbes

#### 5. Annotation or label changes

Sometimes, even if the business logic remains unchanged, changes to related fields in the Pod Template will trigger a rolling update.

### A Basic Conclusion

Deployments focus not on “whether you want to update” but on:

> **Whether there have been changes to the Pod template.**

---

## V. Let’s Look at a Minimal Deployment Example

### Quick Verification
### Demonstration of the Update Process