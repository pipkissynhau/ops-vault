# 01-Application Deployment Basics: Why Deployment Completion Still Requires Release, Update, and Rollback Capabilities

## Document Notes
- Document Positioning: Application release, update, and rollback mainline entry
- Applicable Stage: After completing stateless application deployment, stateful application deployment, node-level application deployment, and Helm basics, enter the application change and release management phase
- Recommended Path: `04-Kubernetes/07-Apply deployment/10-Apply release, update and rollback/01-Application of the release basis: why there is a need for a release, update and rollback capability after deployment is completed`

## Tags
#Kubernetes #ApplyPublication #ApplyUpdate #ApplyRollback #Deployment #Rollout #Helm #Release #ChangeManagement #Clouds. #Transport

---

## One, Why "Application Deployment Completion" Doesn't Equal "Application Management End"

In the previous application deployment mainline, the following content has been completed:

- Stateless application deployment
- Stateful application deployment
- Node-level application deployment
- Helm and application package management

These contents mainly solve:

- How to put applications into Kubernetes
- What workload models different types of components should use
- How Helm installs and manages a group of objects as an application

But in real environments, applications are never "deployed once and finished".

After application goes online, many changes will continue to occur, such as:

- Image version updates
- Configuration adjustments
- Replica count adjustments
- Resource limit adjustments
- Domain and entry adjustments
- Dependency component version changes
- Security fixes
- Rolling back to previous versions

Therefore, application management in Kubernetes doesn't only include "deployment", but must also include another equally important mainline:

> **How to release, update, verify, and rollback applications.**

---

## Two, What Is Application Release

At this stage, we can first give "application release" a practical definition:

> **Deliver a prepared application version to the target environment in a controlled manner.**

Here, "version" doesn't necessarily only refer to image version, but may also include:

- Image tag
- Configuration version
- Values file content
- Resource object definition changes
- Secret / ConfigMap updates
- Helm Chart version changes

Therefore, application release is not a single action, but more like a change process.

### A Basic Understanding
We can first understand application release as the following:

- The application is currently in state A
- Now it needs to become state B
- It needs to be changed in a controlled way

### Operations Understanding Focus
The focus of "release" is not simply applying objects, but:

> **To safely switch the application from old state to new state.**

---

## Three, Why Deployment Capabilities Must Be Complemented with Release Capabilities

This is one of the most important questions at this stage.

Because only deployment capabilities usually solve:

- Putting the application into the cluster for the first time

But in real environments, more common issues are actually:

- How to update an already running application
- How to confirm it's fine after update
- How to roll back to the previous version if problems occur
- How to avoid a single change directly breaking the business
- How to verify after Helm release upgrade
- How to monitor status during Deployment rolling update

These issues all belong to the "release and change management" scope, not the "initial deployment" scope.

### A Basic Conclusion
Application deployment solves:

> **How to make the application exist**

Application release, update, and rollback solve:

> **How to make the application continuously change and remain as controlled as possible**

---

## Four, Why Application Updates Are Frequent in Kubernetes

As soon as the application enters the normal iteration phase, updates will become the norm.

### Common Update Scenarios Include

#### 1. Image Updates
For example:
- Fixing bugs
- Upgrading framework versions
- Applying security patches
- Releasing new features

#### 2. Configuration Updates
For example:
- Modifying environment variables
- Modifying ConfigMap
- Modifying log levels
- Modifying resource limits
- Modifying probe parameters

#### 3. Running Replica Adjustments
For example:
- Changing replica count from 2 to 4
- Temporary scaling up
- Active period scaling up
- Shrinking after fault recovery

#### 4. Delivery Method Changes
For example:
- Switching from handwritten YAML to Helm management
- Updating Helm values
- Switching Chart versions

### Operations Understanding Focus
In Kubernetes application management, the truly frequent action is often not "initial deployment", but:

> **Continuous updates.**

---

## Five, Why "Update" Cannot Simply Be Understood as "Re-applying Once"

At first glance, many changes seem to be completed through:

    kubectl apply -f xxx.yaml

But if we only stay at this level, we may overlook the following key issues:

- Is the update a full replacement or rolling replacement?
- Does the business experience interruption during the update?
- Will the new and old versions coexist temporarily?
- Will configuration updates trigger Pod recreation?
- Will StatefulSet updates proceed sequentially?
- How to judge the status after Helm upgrade?
- Can we quickly rollback if the update fails?

Therefore, the truly important aspect of "update" is not the submission action, but:

> **Whether the update process is controllable.**

---

## Six, Why Rollback Capabilities Are As Important As Update Capabilities

Discussing only updates without rollback makes the release process incomplete.

### Why Must There Be Rollback

Because changes are not always successful.  
Common issues in reality include:

- New image startup failure
- New configuration causing application anomalies
- New version logic has bugs
- Resource parameter errors
- Unreasonable probe configuration causing Pod continuous restart
- Service unavailable after Helm upgrade

Without rollback capabilities, we can only:
- Temporarily adjust configurations
- Manually rollback
- Search for old versions
- Panic in the face of failures

### Core Problem Solved by Rollback
It doesn't solve whether "change failure will happen", but:

> **Whether we can quickly return to a known available state after change failure.**

### A Basic Conclusion
Update capabilities determine:

> **Whether we can move forward**

Rollback capabilities determine:

> **Whether we can revert after errors**

---

## Seven, Why Release, Update, and Rollback Should Be Understood Together

These three are essentially different stages of the same mainline.

### Release
Deliver a specific version to the environment.

### Update
Switch the current running state to a new version or configuration.

### Rollback
Revert to the old available state when the new state doesn't meet expectations.

### A Unified Understanding
We can understand them as a complete chain:

    Current State
       ->
    Release / Update
       ->
    Verification Result
       ->
    Success Continue
       ->
    Failure Rollback

### Operations Understanding Focus
These three should not be isolated, but should be understood as:

> **Application Change Closed-Loop**

---

## Eight, Why Kubernetes Emphasizes "Incremental Changes"

Application release in Kubernetes generally doesn't recommend "direct full replacement", but emphasizes more:

- Rolling updates
- Status checks
- Readiness probe coordination
- Phased confirmation
- Rollback capability on failure

### Why This Is Necessary
Because in real environments, applications aren't better with faster replacement, but with more controllability.

### Example: Deployment Rolling Update
The value of Deployment lies in its support for:
- Gradual switching between new and old Pods
- Gradually launching new versions
- Gradually phasing out old versions

This makes the update process smoother.

### Example: Helm Upgrade
The value of Helm isn't just "changes take effect immediately", but:
- Records changes by Release
- Allows viewing history
- Supports rollback to old revisions /think

### Operational Understanding Focus
Kubernetes application changes emphasize:

> **Progressive, observable, and reversible**

Rather than simple, crude replacement.

---

## IX. Why "Verification" is Part of the Release Process

Many people mistakenly view release as:

- Modify
- Apply
- Complete

This understanding is incomplete.

Because the most critical issue after release is actually:

> **Whether the application is truly working as expected.**

### Common Post-Release Verification Contents Include

#### 1. Object Layer Success
For example:
- Pod is Running
- Deployment is Available
- Service is normal
- Ingress is effective

#### 2. Application Layer Normalcy
For example:
- Page accessibility
- Interface response
- Log normalcy
- Probe success

#### 3. Business Layer Normalcy
For example:
- Login availability
- Data read/write normalcy
- Key functionality normalcy
- Error rate increase

### Operational Understanding Focus
The release action itself does not signify success,  
Verification determines whether the release is truly complete.

---

## X. Why Application Release Capabilities Naturally Follow Helm Section

The Helm section has already covered these contents:

- What is Helm
- Common commands
- values.yaml
- Installation, viewing, upgrading, and uninstalling practices

This means two foundational layers are now established:

### First Layer: Native Kubernetes Object Foundation
Already know:
- Deployment
- StatefulSet
- DaemonSet
- Service
- ConfigMap
- Secret

### Second Layer: Helm Application Delivery Foundation
Already know:
- Chart
- Release
- values
- install / upgrade / rollback / uninstall

Therefore, entering the "Application Release, Update, and Rollback" phase is natural, as this stage can simultaneously cover both lines:

### Native Object Release Line
For example:
- Deployment rolling update
- `kubectl rollout status`
- `kubectl rollout undo`

### Helm Release Line
For example:
- `helm upgrade`
- `helm history`
- `helm rollback`

### Operational Understanding Focus
The value of this stage lies in:

> **Unifying the object layer and application package layer under the "change management" main line.**

---

## XI. Key Problems This Stage Will Address

The focus of this section typically isn't continuing to repeat "what objects are," but gradually answering these more practical questions:

### 1. How to Perform Rolling Updates for Deployment
For example:
- Update image
- Check rollout status
- Check revision
- Check if update is complete

### 2. How Configuration Changes Affect the Application
For example:
- Does ConfigMap update automatically take effect
- Does Pod need to be rebuilt
- How to verify after changes

### 3. How to Perform Rollback
For example:
- `kubectl rollout undo`
- `helm rollback`
- How to confirm post-rollback status

### 4. How Helm and Native Rolling Updates Coordinate
For example:
- Helm upgrade ultimately drives object changes
- Helm rollback still requires object state verification

### A Fundamental Conclusion
This stage focuses on:

> **Whether the application can be installed**

Instead of:

> **How to safely change the application after installation.**

---

## XII. Key Cognitive Shifts to Establish at This Stage

### 1. Deployment is Not the End, Change is the Norm
This is the first cognition.

### 2. Application Updates Are Frequent, Not Occasional
This is the second cognition.

### 3. Only Updating Without Rollback Capability Is Incomplete Release Capability
This is the third cognition.

### 4. Release, Update, and Rollback Should Be Understood as a Closed Loop
This is the fourth cognition.

### 5. Helm and Native Object Capabilities Converge at This Stage
This is the fifth cognition.

---

## XIII. Learning Focus of This Section

This section does not expand on specific rollout command details, but first establishes this main line:

### 1. Why Deployment Completion Is Far From Enough
Because the application will continuously change.

### 2. Why Updates Must Be Controllable
Because business cannot afford the risks of crude replacement.

### 3. Why Rollback Capability Must Exist
Because change failure will always occur.

### 4. Why Verification Is Part of the Release Closed Loop
Because the truly important thing is "whether the result is usable."

### 5. Why This Stage Is Appropriate to Follow Helm
Because we now have:
- Object foundation
- Application package management foundation

---

## XIV. Stage Summary

Application deployment solves:

- How to get the application into Kubernetes

Helm solves:

- How to install and manage the application as a package

Application release, update, and rollback solve:

- How the application continuously changes
- How to verify after changes
- How to revert if problems occur

Therefore, the core of this stage is not "learning more commands," but establishing a complete change closed-loop understanding:

> **Release is not submitting a change, release is letting the change enter production in a controlled way.**

---

## XV. Keyword Mnemonics

- Application Release: Delivering a version to the target environment
- Application Update: Switching the current state to a new state
- Application Rollback: Reverting to an old, usable state
- Change Closed-Loop: Update, verification, rollback on failure
- Progressive Change: Gradual replacement instead of crude replacement
- Release Verification: Confirming the change result matches expectations
- Versioned Management: Making changes trackable and reversible

---

## XVI. Operational Extended Understanding

In the Kubernetes application management main line, the learning sequence roughly goes through three stages:

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

### Third Stage: Change Layer
Focus on:
- Release
- Update
- Rollback
- Verification
- Version history

At this step, we are transitioning from "knowing how to deploy applications" to "knowing how to manage application changes."

This is also a step in the operational shift from object user to application delivery manager.

---

## References
- Kubernetes Official Documentation
- Helm Official Documentation

---

## Next Day Suggestions
Next post suggestion: organize

[[02-Deployment Rolling Update Basics - Image Updates Rollout Revision Status Check]]