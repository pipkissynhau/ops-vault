# 02-Deployment Rolling Update Basics: Image Updates, Rollout, Revision, and Status Checks

## Document Notes
- Document Focus: Basics of Deployment Rolling Updates and Version Changes
- Applicable Stage: After completing foundational knowledge of application publishing, enter the mainline of Kubernetes native workload updates and rollbacks
- Recommended Path: `04-Kubernetes/07-Apply deployment/10-Apply release, update and rollback/02-Deployment Scroll update basis: mirror update,rolloutI don't know.revision and status check`

## Tags
#Kubernetes #Deployment #Rollout #Revision #ScrollUpdate #MirrorUpdate #ApplyPublication #ApplyRollback #Clouds. #Transport

---

## I. Why Deployment Updates Need Separate Learning

In Kubernetes, Deployment is one of the most common stateless workload controllers.  
Many business services, such as:

- Java API
- Go services
- Python Web services
- Nginx
- Frontend services

often run as Deployments.

Therefore, the part about application publishing, updates, and rollbacks naturally starts with:

> **How Deployment completes version updates.**

The most common change actions in this phase typically include:

- Updating image versions
- Adjusting replica counts
- Modifying environment variables
- Modifying resource limits
- Modifying probe parameters
- Modifying labels or annotations

However, in the mainline of publishing, the most typical and observable action remains:

> **Image updates.**

Because image updates almost represent the most basic application version release.

---

## II. What is Deployment Rolling Update

The default update method for Deployment is usually rolling update.

We can first give it a direct understanding:

> **Not deleting all old Pods at once and then starting new Pods, but rather gradually creating new version Pods and gradually replacing old version Pods.**

The goal of this approach is:

- To make the application update process smoother
- To reduce the risk of interruption
- To maintain a certain number of available replicas
- To make the release process observable and controllable

### A Direct Example
Assume a Deployment currently has 3 replicas:

- web-v1 Pod 1
- web-v1 Pod 2
- web-v1 Pod 3

If updating to a new image version, the rolling update approach is typically not:

- Deleting all 3 old Pods at once
- Then restarting 3 new Pods

But rather:

- Start one new version Pod first
- Wait for it to be Ready
- Then gradually replace old version Pods
- Until all become the new version

### Operations Understanding Focus
The core of rolling update is not "updating the object definition," but:

> **Gradually switching between new and old versions within controlled limits.**

---

## III. Why Rolling Update is More Reasonable than Full Replacement

### 1. Can Reduce Business Interruption Risk
If old Pods are deleted all at once and new Pods are started, the business may be temporarily unavailable during the update.

### 2. Can Gradually Validate New Version Health
During rolling update, you can observe:
- Whether new Pods start normally
- Whether Readiness Probe passes
- Whether application logs are normal
- Whether error rates rise

### 3. Easier to Locate Issues When Problems Occur
Because new and old versions coexist temporarily, issue localization is easier focused on this change.

### 4. More Aligned with Real Production Requirements
Production environments typically don't accept "stop first, then start gradually" brute force methods, unless it's a clear downtime release scenario.

### Operations Understanding Focus
Rolling update is not "a more complex replacement method," but:

> **A more suitable replacement method for online services.**

---

## IV. What is the Most Common Trigger for Deployment Updates

Deployment updates, in the end, essentially mean:

> **Pod Template changes.**

As long as the Pod template part of the Deployment changes, Kubernetes will consider generating a new ReplicaSet and starting a new rolling update.

### Most Common Triggers Include

#### 1. Image Version Changes
For example:

- `nginx:1.27.0` changed to `nginx:1.27.1`

#### 2. Environment Variable Changes
For example, adding or modifying:

- `ENV`
- `LOG_LEVEL`

#### 3. Resource Configuration Changes
For example:
- CPU request
- memory limit

#### 4. Probe Changes
For example:
- readinessProbe
- livenessProbe

#### 5. Annotation or Label Changes
Sometimes even if the business logic hasn't changed, just changing any relevant fields in the Pod Template will trigger a rolling update.

### A Basic Conclusion
Deployment doesn't care about "whether you want to update," but:

> **Whether the Pod template has changed.**

---

## V. Let's Look at a Minimal Deployment Example

Below is the most basic Deployment example for subsequent understanding of the update process.

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: demo-web
      namespace: app-demo
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: demo-web
      template:
        metadata:
          labels:
            app: demo-web
        spec:
          containers:
            - name: web
              image: nginx:1.27.0
              ports:
                - containerPort: 80

### Key Points of This YAML

#### 1. `replicas: 3`
Indicates the Deployment wants to maintain 3 replicas.

#### 2. `image: nginx:1.27.0`
Indicates the current version uses this image tag.

#### 3. `selector` and `template.labels`
Used to establish the relationship between Deployment, ReplicaSet, and Pod.

### Operations Understanding Focus
If you want to make the most basic release update later, the most intuitive action is:

> **Change `image` from the old version to the new version.**

---

## VI. What Happens Internally During Deployment Update

Understanding Deployment updates shouldn't just stay at the command level; you should know the resource chain that occurs internally.

### 1. The Original Deployment Corresponds to an Old ReplicaSet
For example, currently running:

- `demo-web` Deployment
- `demo-web-OldRS`
- 3 old version Pods

### 2. After Pod Template Changes, a New ReplicaSet is Generated
For example, the image changes from:

- `nginx:1.27.0`

to:

- `nginx:1.27.1`

Deployment will consider the Pod template has changed, thus generating:

- `demo-web-NewRS`

### 3. Old and New ReplicaSet Begin Replica Handover
The Deployment controller will gradually:

- Increase the number of Pods in the new ReplicaSet
- Decrease the number of Pods in the old ReplicaSet

### 4. Final New ReplicaSet Takes Over All Replicas
When the update completes:
- The new RS holds all replicas
- The old RS retains history but replica count drops to 0

### Operational Understanding Focus
The essence of Deployment rolling update is not "directly replacing Pods," but:

> **The process of replica handover between the old and new ReplicaSet.**

---

## Seven, What is Rollout

In Kubernetes context, rollout can be initially understood as:

> **The process of gradually advancing workloads from old versions to new versions.**

For Deployment, rollout most commonly corresponds to:

- Rolling update process
- Update process status checks
- Viewing update history
- Rollback after update failure

Therefore rollout is not a standalone resource object, but more like a set of operations and status expressions around the release process.

### Common Related Commands Include
- `kubectl rollout status`
- `kubectl rollout history`
- `kubectl rollout undo`

### Operational Understanding Focus
Rollout focuses not on "YAML content itself," but:

> **Whether the change process is progressing as expected.**

---

## Eight, Why Image Update is the Most Typical Rollout Example

Because image tag changes most directly demonstrate the concept of "version release."

### A Simple Example
Current Deployment uses:

    image: nginx:1.27.0

Now upgrade to:

    image: nginx:1.27.1

At this point:
- Pod template changes
- Deployment triggers new ReplicaSet
- New and old Pods gradually switch
- Rollout begins

### Why It's Suitable as an Introductory Example
Because it has several advantages:

- Single change point
- Easy to observe effects
- Clear distinction between old and new versions
- More aligned with daily "application release" actions

### Operational Understanding Focus
Image update can be seen as:

> **The minimal observable example of Deployment release and change capability.**

---

## Nine, How to Update Deployment Image

The most common methods are divided into two categories.

### Method One: Directly Modify YAML and Apply
For example, change:

    image: nginx:1.27.0

to:

    image: nginx:1.27.1

Then execute:

    kubectl apply -f deployment.yaml

### Method Two: Use set image
For example:

    kubectl set image deployment/demo-web web=nginx:1.27.1 -n app-demo

This command means:
- Update `demo-web` Deployment
- Container name `web`
- New image is `nginx:1.27.1`

### Differences Between the Two Methods
#### Modify YAML and Apply
More suitable for:
- ConfigurationInclusion version management
- GitOps or regular configuration management workflows
- Changes can be reviewed

#### `kubectl set image`
More suitable for:
- Temporary testing
- Quick verification
- Demo update process

### Operational Understanding Focus
In production, it's recommended to:
- Change configuration files
- Git management
- Re-apply

But for learning rollout process, `set image` is very intuitive.

---

## Ten, How to Observe Rollout Status

After Deployment update triggers, the most common first observation command is:

    kubectl rollout status deployment/demo-web -n app-demo

### Purpose of This Command
Used to check whether the Deployment's rolling update has completed.

### Common Output Semantics
Usually see similar:

- Waiting for deployment rollout to finish
- Deployment successfully rolled out

### Why This Command is Important
Because it directly corresponds to:

> **Where this change is currently in the process.**

### Common Commands to Observe Together
In addition to rollout status, you can also simultaneously check:

    kubectl get deploy -n app-demo
    kubectl get rs -n app-demo
    kubectl get pod -n app-demo -o wide

### Operational Understanding Focus
When releasing updates, `kubectl apply` is just the starting point,  
the truly important thing is whether the rollout status successfully completes.

---

## Eleven, How to Understand Revision

During Deployment updates, version history will be formed.  
This version history is typically represented in the form of revision.

You can initially understand revision as:

> **A historical revision version formed by each effective change of Deployment.**

### Why Revisions Exist
Because Deployment doesn't only save the current state, it also retains certain historical information for:
- Rollback
- Auditing
- Tracking version changes

### Common Ways to View

    kubectl rollout history deployment/demo-web -n app-demo

### What This Command Usually Shows
Usually see:
- Revision number
- Corresponding change records

### A Basic Understanding
If a Deployment has undergone multiple changes, you may see:

- revision 1
- revision 2
- revision 3

This indicates:
- Current is not the first deployment
- Previous historical versions are available for reference

### Operational Understanding Focus
The significance of revision lies in:

> **Giving changes a historical trace.**

---

## Twelve, Why Rollout History is Important

### 1. Facilitates Confirmation of Change History
Can know what revisions this Deployment has undergone previously.

### 2. Facilitates Version Selection for Rollback
If rollback is needed later, you can't guess blindly, you need to first check what revisions are available.

### 3. Facilitates Troubleshooting and Post-Mortem Analysis
If issues start after a release, history can help confirm:
- Whether the issue started from a specific revision
- What was the last stable revision before the current version

### Command Example

    kubectl rollout history deployment/demo-web -n app-demo

### Operational Understanding Focus
The purpose of history is not "just to look," but:

> **To provide basis for subsequent rollbacks and post-mortem analysis.**

---

## Thirteen, The Most Common Status Checks When Deployment Updates
During the release update process, it's common to check the following layers of status.

### 1. Deployment Status

    kubectl get deploy -n app-demo
    kubectl describe deploy demo-web -n app-demo

Focus on:
- Whether desired / current / available are normal
- Whether there are exceptions in events

### 2. ReplicaSet Status

    kubectl get rs -n app-demo

Focus on:
- Whether both new and old RS exist
- Whether new RS replica count is increasing
- Whether old RS replica count is decreasing

### 3. Pod Status

    kubectl get pod -n app-demo -o wide
    kubectl describe pod -n app-demo <pod-name>

Focus on:
- Whether new Pod is created successfully
- Whether it is Ready
- Whether it is in CrashLoopBackOff
- Whether image pull failed

### 4. Log Status

    kubectl logs -n app-demo <pod-name>

Focus on:
- Whether new version starts normally
- Whether errors occur
- Whether configuration anomalies exist

### Operations Understanding Focus
Rollout's core is not about a single command, but about:

> **Whether the three-layer status of Deployment, ReplicaSet, and Pod progresses consistently forward.**

---

## Fourteen, When Might Rollout Get Stuck

Kubernetes rolling updates do not always complete successfully.  
Common reasons for getting stuck include:

### 1. New Pod Cannot Start
For example:
- Image does not exist
- Startup command error
- Configuration anomalies
- Dependency connection failure

### 2. New Pod Not Ready
For example:
- readinessProbe failure
- Application port not listening normally
- Startup time too long
- Backend dependencies not ready

### 3. Resource Insufficiency
For example:
- Node resources insufficient
- New Pod cannot be scheduled

### 4. Update Policy Constraints
If replica count is small and policy is conservative, update progress may slow down.

### Operations Understanding Focus
When rollout gets stuck, the issue is usually not in "Deployment not updating," but in:

> **New version Pod not reaching a state where it can take over traffic.**

---

## Fifteen, Why Status Checks Are Part of the Release Process

Only performing update actions without status checks makes the release chain incomplete.

### A Basic Release Cycle Should Include
- Initiate update
- Observe rollout status
- Check object status
- Check application access
- Rollback if failed

### What Happens If Only the First Step Is Done
Updating without checking can lead to:
- Thinking the release was successful
- Actual Pod continuously restarting
- Service still unavailable
- Business anomalies detected too late

### Operations Understanding Focus
Deployment release is not just "updating the image," but:

> **Confirming after update whether the target state is truly reached.**

---

## Sixteen, Relationship Between Deployment Rolling Update and Helm Upgrade

We've already learned Helm.  
Now entering the release update phase, we need to connect the relationship between the two.

### Deployment Rolling Update Focuses More on Object Layer
For example:
- Updating Deployment's Pod template
- Triggering new ReplicaSet
- Observing rollout status

### Helm Upgrade Focuses More on Application Layer
For example:
- Updating Chart parameters
- Updating image tag
- Updating Service configuration
- Updating ConfigMap content
- Re-rendering entire object group

### But They Are Not Disconnected
In many cases:

- `helm upgrade` will still ultimately fall to changes in Deployment template
- Changes in Deployment template will ultimately trigger rollout

### A Basic Conclusion
You can initially understand it as:

- Helm upgrade manages "application changes"
- Deployment rollout manages "workload update process"

### Operations Understanding Focus
Helm and rollout are not mutually exclusive, but:

> **One focuses on application layer, the other on object layer.**

---

## Seventeen, The Most Important Understandings in This Section

### 1. Deployment's Default Update Method Is Usually Rolling Update
This is the first understanding.

### 2. Rolling Update Focuses on Gradual Replacement, Not Full Replacement at Once
This is the second understanding.

### 3. Pod Template Changes Trigger New ReplicaSet and New Rollout
This is the third understanding.

### 4. Revision Represents Deployment's Historical Revision Version
This is the fourth understanding.

### 5. Release Updates Must Be Combined with Status Checks, Not Just Change Actions
This is the fifth understanding.

---

## Eighteen, Stage Summary

Deployment rolling update is the most core and common update method in Kubernetes application release.

Through this section, you should at least establish these basic understandings:

- What is rolling update
- Why Deployment is more suitable for gradual replacement
- Why image update is the most typical rollout example
- What roles rollout, revision, and history play in the release process
- Why status checks must continue after update actions

As long as this layer of understanding is clear, proceeding to:

- Configuration change and application update
- Rollback basics
- Relationship between Helm rollback and kubectl rollout undo

Will be more natural.

---

## Nineteen, Keyword Quick Notes

- Rolling update: Gradually replace old version Pod
- Rollout: A single version progress process
- Revision: Deployment's historical revision version
- ReplicaSet: Deployment's version carrier during update process
- Rollout status: Check if release is completed
- Rollout history: Check revision history
- Set image: Quickly update image
- Status check: Confirm if release was truly successful

---

## Twenty, Operational Extension Understanding

The significance of Deployment rolling update is not just learning several commands, but entering the theme of "application online change" for the first time.

In the previous deployment mainline, the focus was more on:

- How objects are created
- How applications enter the cluster
- How Helm installs applications

At rollout, the focus begins to shift to:

- How applications change online
- How old and new versions transition
- How to observe the release process
- How to handle issues based on revision history

In other words, the mainline has shifted from:

> **Deploying applications**

To:

> **Managing application version changes**

This is exactly the core of the application release, update, and rollback phase.

---

## References
- Kubernetes Deployment Official Documentation
- Kubernetes Official Documentation

---

## Next Day Suggestions
Next post suggests organizing:

[[03-Configuration Changes and Application Updates - Image Changes ConfigMap Changes Release Verification Strategy]]