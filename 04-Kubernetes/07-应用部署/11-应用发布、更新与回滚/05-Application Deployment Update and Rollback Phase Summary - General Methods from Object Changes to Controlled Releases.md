# 05 - Summary of the Application Deployment, Update, and Rollback Stage: General Methods from Object Changes to Controlled Releases

## Document Notes
- Document Positioning: Summary and method consolidation of the application deployment, update, and rollback stage
- Applicable Stage: After completing the foundation of application deployment, Deployment rolling updates, image and configuration changes, and rollback basics, enter the stage summary
- Recommended Path: `04-Kubernetes/07-Apply deployment/10-Apply release, update and rollback/05-Apply summary of the publication, update and rollback phase: from object to controlled release`

## Tags
#Kubernetes #ApplyPublication #ApplyUpdate #ApplyRollback #Deployment #Rollout #Helm #Release #ConfigMap #ChangeManagement #Organisation #Clouds. #Transport

---

## I. What This Stage Has Accomplished

The application deployment, update, and rollback section focuses not on defining Deployment, Service, ConfigMap, or Helm itself, but on pulling previously learned objects and delivery methods onto the "change management" main line for understanding.

This stage has covered the following core content:

- Why application deployment completion still requires release, update, and rollback capabilities
- Basic rolling update of Deployment
- Differences between image changes and ConfigMap changes
- Basic usage scenarios of `kubectl rollout undo` and `helm rollback`

The final problem solved by this section can be summarized as:

> **After an application is running in Kubernetes, how to make it change safely, observably, and roll back when necessary.**

The focus here is no longer just on:

- How to write objects
- How to install the application
- How to use Helm

But more on:

- How to initiate changes
- How to observe the change process
- How to verify the change results
- How to rollback after failure
- How to turn an object modification into a controlled release

---

## II. The Most Important Overall Understanding of This Stage

Application management in Kubernetes cannot stop at "deployment" alone.

Deployment solves:

> **How to get the application into the cluster.**

Release, update, and rollback solve:

> **How to continuously change the application after it's in the cluster while keeping it under control.**

This means the application management main line can be divided into at least three layers:

### First Layer: Deployment
For example:
- Deployment
- StatefulSet
- DaemonSet
- Service
- Ingress
- ConfigMap
- Secret

### Second Layer: Application Package Management
For example:
- Helm
- Chart
- Release
- values.yaml

### Third Layer: Change Management
For example:
- rollout
- revision
- Image updates
- Configuration changes
- Release verification
- Rollback recovery

### A Fundamental Conclusion
By the end of this stage, the following overall understanding should be established:

> **Deployment is not the end; change is the norm.**

---

## III. How This Stage Clarifies the "Release" Perspective

In many learning paths, "release" is often understood as a vague phrase:

- Release
- Go live
- Update the application

But this stage has broken down "release" into clearer aspects.

### Current Stage's Basic Understanding of Release
Release is not a single command, but a controlled change process.

It at least includes:

- What is the current state
- What is the new desired state
- How to push the change
- How to verify the change results
- How to roll back to the old state if it doesn't meet expectations

### Therefore, the focus of release is not
- Directly `apply`
- Directly `set image`
- Directly `upgrade`

But rather:

> **How to let the new state enter the runtime environment in a controlled way.**

---

## IV. What Object Layer Release Understanding This Stage Established Through Deployment

Deployment is the most common stateless workload, making it ideal as the first object layer case for the release main line.

Through Deployment, this stage has established the following understandings.

### 1. Pod Template Changes Trigger a New Update
For example:
- Image tag changes
- Environment variable changes
- Resource limit changes
- Probe changes
- Annotation changes

These essentially cause Pod Template changes, triggering a new ReplicaSet and a new rollout.

### 2. Release Is Not Direct Pod Replacement, But ReplicaSet Handover
The essence of Deployment update is not "directly changing the old Pod to a new Pod," but:

- Generating a new ReplicaSet
- Gradual handover between old and new ReplicaSet
- Finally, the new ReplicaSet takes over all traffic

### 3. Rollout Is a Change Progression Process
It focuses on:
- Whether the change has started
- Whether the change is complete
- Whether it's stuck
- Whether the release has successfully progressed

### 4. Revision Provides Historical Traces
It allows Deployment changes to track:
- Which revision was effective
- Which revision caused issues
- Which historical version can be rolled back

### Operations Understanding Focus
The Deployment layer has clarified:

> **Object layer changes are not simple replacements, but have history, process, and state.**

---

## V. How This Stage Clarifies "Update Type Differences" Through Image and Configuration Changes

A key point of this stage is splitting "application update" into different types rather than treating it as a single action.

### 1. Image Change
Image change is more like:
- Program body change
- Code version change
- Feature version change

It usually directly triggers:
- New Pod
- New ReplicaSet
- A new rollout

### 2. Configuration Change
Configuration change is more like:
- Runtime condition change
- Behavior parameter change
- External dependency change
- Environment variable or configuration file change

It doesn't necessarily mean:
- New version is active
- The running application has used the new configuration

### Why This Is Important
Because many configuration changes still require:
- Pod recreation
- rollout restart
- Explicit triggering of a new application startup

### A Fundamental Conclusion
Through this content, the following judgment should be formed:

> **Image changes update the program body, configuration changes update the program runtime conditions.**

Both belong to "application updates," but the verification focus differs.

---

## VI. How This Stage Clarifies "What to Check After a Change" Through Release Verification

If you can only initiate changes but cannot verify the results, the release process is incomplete.

This stage has at least structured the verification approach into three layers.

### 1. Object Layer Verification
For example:
- Is Deployment Available
- Is rollout complete
- Is ReplicaSet handover normal
- Is Pod Running / Ready

### 2. Application Layer Verification
For example:
- Are application logs normal
- Is configuration truly loaded
- Are pages or interfaces returning normally
- Are probes passing

### 3. Business Layer Verification
For example:
- Are key business requests successful
- Is error rate abnormal
- Is latency significantly increased
- Are core functions restored

### A Fundamental Conclusion
Through this content, the following habit should be formed:

> **The end of a release action does not equal confirmation of the release result.**

A true closure must include verification.

---

## VII. How This Stage Clarifies "What to Do After a Failure" Through Rollback

Update and rollback capabilities must be understood together.

This stage has split the basic rollback path into two layers:

### 1. Deployment Object Layer Rollback
Through:
- `kubectl rollout history`
- `kubectl rollout undo`

To revert to a specific Deployment historical revision version.

This is suitable for:
- Native Deployment management scenarios
- Quick rollback of single workload objects

### 2. Helm Application Layer Rollback
Through:
- `helm history`
- `helm rollback`

To revert to a specific Release historical revision version.

This is suitable for:
- Applications managed by Helm
- Full rollback of a group of objects
- Historical recovery at the Release level

### Why This Layer Is Important
Because it demonstrates that application rollback is not a vague concept, but rather:

- Has historical basis
- Has object layer entry points
- Has application layer entry points
- Has verification closure

### A Fundamental Conclusion
Through the rollback layer, we should form a judgment:

> **Controlled release is not just about upgrades, but also about being able to roll back.**

---

## VIII. What is the complete release cycle finally solidified in this stage

By this point, we should already be able to connect the releaseMain into a complete cycle.

### Step 1: Identify Change Type
First determine what kind of change this is:

- Image change
- Configuration change
- Helm values change
- Resource parameter change

### Step 2: Initiate the Change
Common methods include:

- `kubectl apply`
- `kubectl set image`
- `kubectl rollout restart`
- `helm upgrade`

### Step 3: Observe the Change Process
Common focus areas:

- `kubectl rollout status`
- `kubectl get deploy`
- `kubectl get rs`
- `kubectl get pod`
- `helm status`

### Step 4: Verify the Change Results
At least verify:
- Object status
- Application logs
- Interface or page access
- Business critical paths

### Step 5: Rollback if Necessary
Common methods include:

- `kubectl rollout undo`
- `helm rollback`

### Step 6: Verify Again After Rollback
Ensure:
- Object status recovery
- Application access recovery
- Business normalization

### Operations Understanding Focus
This chain is the most important method closure at this stage:

> **A change is not a command, but a complete controlled release cycle.**

---

## IX. What are the several commonly overlooked issues in this stage

### 1. Only Initiate Changes Without Observing Rollout
The change action itself does not mean the change was successful.

### 2. Only Check Pod Running, Not Application Behavior
Object layer recovery does not equal business layer recovery.

### 3. Treat ConfigMap Changes and Image Changes as the Same Type of Thing
They have different impact paths and verification methods.

### 4. Rollback Without Checking History
This makes the rollback action lack basis.

### 5. No Verification After Rollback
This leads to the problem of "command success but business not recovered" being ignored.

### 6. Confuse Deployment Rollback and Helm Rollback Layers
One is object layer focused, one is application layer focused, they should not be conflated.

---

## X. What is the most important capability formed in this stage

This part truly establishes not just several command memories, but a more stable change judgment capability.

### 1. Able to View Application Changes by Type
Know that image changes and configuration changes are not completely the same type of update.

### 2. Able to View Releases as a Process, Not Just an Action
Know that `apply`, `upgrade` are just the beginning, not the end.

### 3. Able to Treat Verification as Part of the Release
Know that release success depends on verification closure confirmation.

### 4. Able to View Rollback as a Normal Means
Know that rollback is not "something you think of only when failed," but part of the standard change flow.

### 5. Able to Distinguish Between Object Layer and Application Layer Rollback Methods
Know when to use:
- `kubectl rollout undo`
- `helm rollback`

### Operations Understanding Focus
The essence of these capabilities is:

> **Evolve from "knowing how to issue commands" to "knowing how to manage changes."**

---

## XI. How does this stage connect with the previous Helm section

The Helm section solves:

- How applications are packaged
- How applications are installed/updated/uninstalled
- How values affect deployment results

The application release, update, and rollback section continues to solve:

- What to do after an upgrade
- When to use rollback
- How object layer and application layer correspond
- How to integrate Helm perspective with kubectl perspective

### A Unified Understanding
You can remember the relationship between these two parts as:

- Helm handles application delivery entry points
- The releaseMain handles application change management closure

This is why this stage naturally follows the Helm section.

---

## XII. How does this stage connect with the practicalMain ahead

By this point, the content leaning toward "concepts and basic methods" has basically been concluded.

Previously completed:

- Stateless application deployment
- Stateful application deployment
- Node-level application deployment
- Helm and application packaging management
- Application release, update, and rollback

The more natural direction ahead is typically to enter:

- Common middleware and platform containerization deployment practicals
- Application deployment troubleshooting
- Business and platform deployment cases

### Why This Connection Makes Sense
Because we now have these foundational capabilities:

- Able to judge workload models
- Able to understand object relationships
- Able to deliver applications with Helm
- Able to handle basic updates and rollbacks

We can then apply these capabilities in real components and real scenarios.

---

## XIII. Final Conclusion of This Stage

The application release, update, and rollback section ultimately solidifies not a specific command usage, but a transferable release method.

This method at least includes:

- First identify change type
- Then select appropriate change entry points
- Then observe rollout process
- Then perform object layer, application layer, and business layer verification
- If expectations are not met, rollback based on history
- Verify recovery results after rollback

The core goal of this method is not "to make changes happen," but:

> **To make changes happen in a controlled manner.**

At this point, the **10-Application Release, Update, and Rollback** section can be concluded.

---

## XIV. Keyword Quick Notes

- **Release**: Deliver new state to target environment
- **Update**: Transition current state to new version or configuration
- **Rollback**: Revert to old stable state
- **Rollout**: A release progression process
- **Revision**: Deployment historical revision
- **Release Revision**: Helm Release historical revision
- **Release Verification**: Confirm new state is truly usable
- **Controlled Release**: Change method that is observable, verifiable, and rollable

---

## XV. Operations Extended Understanding

After learning Kubernetes to this stage, the main line has undergone a noticeable change.

Initially focused on:
- What objects are
- How to write objects
- How to deploy applications into the cluster

Later focused on:
- What a Chart is
- What a Release is
- How to modify values
- How to deploy applications

At the release, update, and rollback layer, the focus shifts further to:
- How applications continuously change
- How to observe change processes
- How to verify change results
- How to roll back when changes fail

This indicates the learning focus has shifted from:
> **Understanding deployment objects**

to:
> **Understanding application lifecycle and change management**

This step is crucial because the actual high-frequency issues that occur during later practical exercises, troubleshooting, and case studies often belong to "systems in change," rather than "staticly defined objects."

---

## Reference Materials
- Kubernetes Deployment Official Documentation
- Helm Official Documentation
- Kubernetes Official Documentation

---

## Next Stage Recommendation
The next stage recommends entering:

**Yes.11-Common Middleware and Platform Containerization Deployment Practical GuideI don't know.**