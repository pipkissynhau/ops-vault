# 04-Rollout Undo and Helm Rollback: Use Cases

## Document Description
- Documentation Purpose: Introduction to application rollback capabilities
- Applicable Phase: After mastering basic application deployment, Deployment rolling updates, and understanding image and configuration changes, move on to learning about rollback
- Recommended Path: `04-Kubernetes/07-Application Deployment/10-Application Release, Update, and Rollback/04-Rollout Undo and Helm Rollback: Use Cases`

## Tags
#Kubernetes #Rollback #Deployment #Helm #Release #Application Deployment #Application Update #Cloud Native #Ops

---

## I. Why Learn about “Rollback” Separately

In the main application deployment process, updates and rollbacks must be understood together.

Previously, you have learned:

- How to perform rolling updates on Deployments
- How image changes trigger rollouts
- Why ConfigMap changes often require rebuilding Pods
- Why release verification cannot rely solely on Pod status being “Running”

These topics address how to move an application forward. However, in real-world scenarios, failures during changes are common. Common failure scenarios include:

- New images failing to start
- New versions having API issues
- New configurations causing application errors
- Improper probe configurations resulting in Pods never reaching the “Ready” state
- Abnormal application states after Helm upgrades
- Regression issues after new business features are deployed

Therefore, the deployment process should not only focus on how to upgrade but also include how to recover in case of failures.

---

## II. What is Rollback

For now, you can define rollback as:

> **Returning an application from its current abnormal or unexpected state back to a previously known and stable state.**

The “return” here can refer to various aspects:

- Returning to an older image version
- Returning to previous configuration settings
- Going back to an earlier Deployment revision
- Returning to an earlier Helm Release revision

### A Basic Understanding
You can think of the application’s state as follows:

- Current State: New version / new configuration / latest revision
- Historical States: Older versions / older configurations / previous revisions

The essence of rollback is moving from the “current state” back to a “historical state.”

### Key Points forOps Professionals
Rollback is not simply about “re-deploying an old version”; it’s more about:

> **Rapidly restoring the application to a known stable state based on historical records.**

---

## III. Why Rollback Capability Is as Important as Update Capability

### 1. Failures During Changes Are Normal, Not Exceptions
Any deployment process involves the possibility of failure. The question is not whether failures will occur but whether:

- You can quickly stop losses after a failure
- You can restore the system in an orderly manner

### 2. Updates Without Rollback Capability Pose High Risks
If, after an update fails, you can only:

- Manually modify the YAML files
- Temporarily find an old image tag
- Restore configurations manually
- Guess a stable state based on assumptions

the speed and accuracy of recovery will be significantly reduced.

### 3. Rollback Capability Makes Deployment a Closed Loop
A complete deployment process should include at least:

- Initiating changes
- Monitoring the status
- Verifying results
- Rolling back if expectations are not met

### A Basic Conclusion
Update capability determines whether you can move forward, while rollback capability ensures that you can recover if issues arise during the process.

---

## IV. Why Kubernetes Rollback Should Not Be Simply Seen as “Going Back to the Old State”

On the surface, rollback seems like a simple process of:

- Changing the image tag back to an old version
- Restoring the ConfigMap to previous settings
- Re-executing the apply command

While this approach can work in some cases, it is not ideal.

### Reason 1: Loss of Historical Context
Relying solely on memory to recall details such as:

- The previous image version
- The last revision number
- The content of the previous values file

can easily lead to errors.

### Reason 2: Slow Response Time
In troubleshooting, speed is crucial for restoring services and returning the system to a stable state. Manually searching for old configuration settings is usually inefficient.

### Reason 3: Lack of Operational Consistency
When multiple team members collaborate, manually “going back to the old state” can disrupt the change management process.

### Key Points forOps Professionals
A more reasonable approach to rollback is not to “re-create the old version from memory” but to:

> **Restore the system based on existing historical records in a systematic way.**

---

## V. What Are the Core Elements of Rollback for Native Kubernetes Workloads?

In the context of Deployments, rollback primarily relies on:

- rollout history
- revision numbers
- the `kubectl rollout undo` command

### Why These Three Elements?
During an update process, a Deployment creates:

- New### 3. View Current Values

    helm get values my-app -n app-demo

### 4. If Necessary, Use kubectl to Check Object Status

    kubectl get deploy,sts,ds,svc,pod -n app-demo

### Key Points for Operations and Maintenance Understanding
Before rolling back with Helm, it is essential to confirm at least the following:

- What is the current status of the Release?
- Which historical revision is stable?
- Whether the current issue is caused by the most recent upgrade?

---

## Section XII: Common Use Cases for Deployment Rollbacks

### Case 1: New Pod Fails to Start After Image Update
Examples include:
- The image does not exist.
- The startup command contains errors.
- The new version of the program crashes.

### Case 2: Business Interface Issues After Image Update
Examples include:
- Logical bugs in the new version.
- Incompatibility with dependencies.
- An increase in request failure rates.

### Case 3: Abnormal Behavior of Pods Despite Configuration Changes
Examples include:
- Incorrect environment variable settings.
- Misconnected application dependencies.
- Unreasonable probe parameters.

### A Basic Principle
As long as the new revision of a Deployment clearly does not meet expectations and the issue cannot be quickly resolved, rolling back should be considered as a priority.

### Key Points for Operations and Maintenance Understanding
Rollback is not an “ultimate measure,” but rather:

> **A normal method to restore business availability.**

---

## Section XIII: Common Use Cases for Helm Rollbacks

### Case 1: Overall Application Issues After helm upgrade
Examples include:
- Incorrect new values configuration.
- Unexpected Chart rendering results.
- Multiple objects being affected negatively.

### Case 2: Incompatibilities After Middleware Updates
Examples include:
- Changes in image versions.
- Changes in Service types.
- Modifications to probe parameters.
- Changes in persistence or authentication settings.

### Case 3: Need for a Full Rollback at the Release Level
If a Helm upgrade affects not just a Deployment but an entire set of objects, it is more appropriate to perform a full rollback at the Release level.

### Key Points for Operations and Maintenance Understanding
Helm rollbacks are particularly suitable for situations where:

> **The issue is not with a single workload but with the entire application upgrade.**

---

## Section XIV: What to Check After Performing a Deployment Rollback

After executing:

    kubectl rollout undo deployment/demo-web -n app-demo

it is necessary to continue verification.

### 1. Check the Rollout Status

    kubectl rollout status deployment/demo-web -n app-demo

### 2. Review the Rollout History

    kubectl rollout history deployment/demo-web -n app-demo

### 3. Verify Pods and ReplicaSets

    kubectl get rs -n app-demo
    kubectl get pod -n app-demo -o wide

### 4. Perform Application Access Verification
Examples include:
- Whether the pages have returned to normal.
- Whether the interfaces are functioning correctly.
- Whether error rates have decreased.
- Whether new alerts have disappeared.

### Key Points for Operations and Maintenance Understanding
The rollback action itself does not guarantee success;
verification after rollback is as important as verification after an upgrade.

---

## Section XV: What to Check After Performing a Helm Rollback

After executing:

    helm rollback my-app 1 -n app-demo

it is usually necessary to further check the following aspects.

### 1. Check the Release Status

    helm status my-app -n app-demo

### 2. Review the Release History

    helm history my-app -n app-demo

### 3. View Current Values Again

    helm get values my-app -n app-demo

### 4. Re-check Objects Using kubectl

    kubectl get deploy,sts,ds,svc,pod -n app-demo

### 5. Perform Application Access Verification
Examples include:
- Whether the pages have returned to normal.
- Whether middleware connections are functioning correctly.
- Whether service entrances are operational.
- Whether application logs have returned to a normal state.

### Key Points for Operations and Maintenance Understanding
Although Helm rollback is initiated at the application layer, verification must still focus on:

- Release status.
- Kubernetes object status.
- Actual application availability.

---

## Section XVI: Why “Successful Rollback” Cannot Be Determined Only by Command Execution Success

### Reason 1: Successful Command Execution Does Not Mean Business Recovery
Sometimes, even though the command executes without errors, issues may still exist:
- Pods may not be fully ready.
- Services may not have fully recovered.
- Dependency chains may still be malfunctioning.

### Reason 2: Object Recovery Does Not Equal Business Recovery
Examples include:
- The Deployment may have been restored, but problems caused by ConfigMap or Secret may still persist.
- External dependencies may have changed.

### Reason 3: Rollback Itself May Trigger New Issues
Examples include:
- The old version of the image may no