# 04- Rollback Basics: Using kubectl rollout undo and Helm rollback

## Document Notes
- Document Location: Application Rollback Fundamentals
- Applicable Stage: After completing application deployment basics, Deployment rolling updates, and understanding image/configuration changes, enter rollback mainline
- Recommended Path: `04-Kubernetes/07-Apply deployment/10-Apply release, update and rollback/04-Rollback Foundation:kubectl rollout undo and Helm rollback Use scene`

## Tags
#Kubernetes #RollBack #Rollout #Helm #Rollback #Deployment #Release #ApplyPublication #ApplyUpdate #Clouds. #Transport

---

## I. Why Rollback Needs Separate Learning

In the application deployment flow, updates and rollbacks must be understood as a pair.

Previously learned:

- Deployment rolling updates
- How image changes trigger rollout
- Why ConfigMap changes often require Pod recreation
- Why pod running status alone isn't sufficient for deployment validation

These address:

> **How applications move forward.**

But in real environments, failed changes are common.  
Common failure scenarios include:

- New image startup failure
- New version interface anomalies
- New configuration causing application errors
- Unreasonable probe configuration leading to Pod not Ready
- Helm upgrade resulting in abnormal application status
- Regression issues after business feature launch

Therefore, the deployment flow can't just learn "how to upgrade", but must also complete:

> **How to roll back after upgrade failure.**

This is exactly what rollback solves.

---

## II. What is Rollback

At this stage, give "rollback" a very practical definition:

> **Reverting an application from its current abnormal or unexpected state back to a previously known working state.**

The "reversion" can be at multiple levels:

- Reverting to old image version
- Reverting to old configuration
- Reverting to old Deployment revision
- Reverting to old Helm Release revision

### A Basic Understanding
You can initially think of application state as:

- Current state: new version / new configuration / new revision
- Historical state: old version / old configuration / old revision

Rollback's essence is moving from "current state" back to "a historical state".

### Operations Focus
Rollback isn't simply "redeploying an old version", but more akin to:

> **Quickly restoring to a known stable state based on existing history.**

---

## III. Why Rollback and Update Capabilities Are Equally Important

### 1. Change failure is normal, not an exception
Any deployment may fail.  
The difference isn't whether failure will happen, but:

- Can we quickly contain the failure after it happens
- Can we orderly recover

### 2. Updates without rollback capabilities carry high risk
If an update fails, you might have to:
- Manually edit YAML
- Temporarily find old image tag
- Manually restore configuration
- Temporarily guess the last stable state

This would significantly reduce handling speed and accuracy.

### 3. Rollback capability makes deployment a closed loop
A complete deployment loop should at least include:

- Initiating change
- Observing status
- Validating results
- Rolling back if unexpected

### A Basic Conclusion
Update capability determines:

> **Whether we can move forward**

Rollback capability determines:

> **Whether we can return after encountering problems while moving forward**

---

## IV. Why Kubernetes Rollback Can't Be Simply Understood as "Reverting Back"

At first glance, rollback seems to simply mean:

- Changing image tag back to old version
- Changing ConfigMap back to old content
- Reapplying the change

This approach can work in some cases, but it's not ideal.

### Reason 1: Easy to lose historical context
If relying solely on manual memory:
- What was the previous image version
- What was the previous revision
- What was the previous values file content

Errors are easily made.

### Reason 2: Slow response time
In fault handling, the most important is:
- Quickly restoring service
- Quickly returning to a known stable state

Manually searching for old configurations is typically inefficient.

### Reason 3: Poor operational consistency
In team collaboration, temporarily "reverting back" can disrupt the change chain.

### Operations Focus
A more reasonable rollback isn't "manually redoing old versions from memory", but:

> **Recovering based on existing historical records.**

---

## V. What to Focus On for Rollback in Native Kubernetes Workloads

In Deployment scenarios, rollback mainly depends on:

- rollout history
- revision
- rollout undo

### Why These Three
Because Deployment creates:
- New and old ReplicaSet
- Historical revision records
- Reversible version history

This gives Deployment its most basic native rollback capability.

### What This Means
For Deployments, rollback isn't an additional patch, but:

> **A natural extension of the rolling update mechanism.**

---

## VI. What is `kubectl rollout undo`

### Command Purpose
Used to rollback a Deployment to a previous historical revision.

### Most Common Usage

    kubectl rollout undo deployment/demo-web -n app-demo

### What This Command Means
Indicates:
- Rolling back `demo-web` Deployment
- Returning to the previous revision

### If Specifying Revision

    kubectl rollout undo deployment/demo-web --to-revision=2 -n app-demo

This indicates:
- Not just returning to "the previous"
- But explicitly returning to revision 2

### Operations Focus
The value of `rollout undo` lies in:

> **Using Deployment history to quickly revert to an old version.**

---

## VII. What is Helm rollback

If the application is managed by Helm, rollback typically focuses more on Releases.

### Command Purpose
Used to rollback a Helm Release to a historical revision.

### Common Usage

    helm rollback my-app 1 -n app-demo

This indicates:
- Release name: `my-app`
- Rolling back to revision `1`

### Difference from Deployment Rollback
Deployment rollback focuses more on object layer, paying attention to:
- Deployment
- ReplicaSet
- rollout history

Helm rollback focuses more on application layer, paying attention to:
- Release
- Chart
- values
- Historical revisions of the entire group of objects

### Operations Focus
Helm rollback's focus isn't individual Deployments, but:

> **Restoring the entire Release to a historical revision.**

---

## VIII. What's the Fundamental Difference Between `kubectl rollout undo` and `helm rollback`

This is one of the most important comparisons in this document.

### `kubectl rollout undo`
More focused on:
- Kubernetes native object layer
- Restoring historical revisions of individual Deployments
- Rolling back workload update history

### `helm rollback`
More biased toward:
- Helm application layer
- Restoration of the entire Release's historical revisions
- Rolling back the state of a group of objects managed by Helm

### A more intuitive comparison

| Dimension | kubectl rollout undo | helm rollback |
|---|---|---|
| Focus level | Deployment object layer | Release application layer |
| Rollback objects | Single Deployment | Entire Helm Release |
| Dependency history | Deployment revision | Helm release revision |
| Common scenarios | Handwritten YAML / native Deployment management | Helm-managed applications |

### Operations understanding focus
If the application is primarily managed by native Deployments, prioritize thinking of:

- `kubectl rollout undo`

If the application is primarily managed by Helm, prioritize thinking of:

- `helm rollback`

---

## IX. Why you must check history before rolling back

Whether it's a Deployment or Helm, rolling back should never be done blindly.

### Reason 1: Confirm what versions are in the history
Don't guess based on memory:
- Which stable version was the last one
- Whether the current issue started from the most recent revision

### Reason 2: Confirm the target version is truly a known stable version
Not all old versions are necessarily suitable for rollback.

### Reason 3: Facilitate post-mortem analysis
Checking history before rolling back makes it easier to trace the source of issues after rolling back.

### A basic conclusion
The first step before rolling back should not be to immediately execute undo, but rather:

> **Check the history first.**

---

## X. What to typically check before rolling back a Deployment

### 1. View rollout history

    kubectl rollout history deployment/demo-web -n app-demo

### What this step shows
Typically you'll see:
- Revision number
- History of revisions

### 2. Check current Deployment status

    kubectl get deploy demo-web -n app-demo
    kubectl describe deploy demo-web -n app-demo

### 3. Check current Pod and ReplicaSet status

    kubectl get rs -n app-demo
    kubectl get pod -n app-demo -o wide

### 4. Check logs and events
If troubleshooting before rolling back, you should also check:

    kubectl logs -n app-demo <pod-name>
    kubectl describe pod -n app-demo <pod-name>

### Operations understanding focus
Before rolling back a Deployment, you need to confirm two things:

- The current issue is indeed related to this revision
- The target rollback version is clearly defined

---

## XI. What to typically check before rolling back a Helm release

### 1. View release history

    helm history my-app -n app-demo

### What this step shows
Typically you'll see:
- Revision number
- Update time
- Status
- Chart version
- Notes information

### 2. Check current release status

    helm status my-app -n app-demo

### 3. Check current values

    helm get values my-app -n app-demo

### 4. Optionally combine with kubectl to check object status

    kubectl get deploy,sts,ds,svc,pod -n app-demo

### Operations understanding focus
Before rolling back a Helm release, you should at least confirm:

- What state the current release is in
- Which version in the history is stable
- Whether the current issue was introduced by the latest upgrade

---

## XII. Most common use cases for Deployment rollback

### Scenario 1: New Pod fails after image update
For example:
- Image does not exist
- Start command error
- New version application crashes

### Scenario 2: Business interface anomalies after image update
For example:
- New version logic bug
- Incompatible dependencies
- Increased request failure rate

### Scenario 3: Pod starts but behaves abnormally after configuration changes
For example:
- Environment variable configuration error
- Application connects to wrong dependencies
- Probe parameters are unreasonable

### A basic judgment
If the new revision of a Deployment is clearly not meeting expectations and the cause cannot be quickly resolved, rollback should be prioritized.

### Operations understanding focus
Rollback is not a "last-resort action", but rather:

> **A normal means to restore business availability.**

---

## XIII. Most common use cases for Helm rollback

### Scenario 1: Application state anomalies after helm upgrade
For example:
- New values configured incorrectly
- Chart rendering results do not meet expectations
- Multiple objects are broken together

### Scenario 2: Incompatibility after middleware upgrade
For example:
- Image version change
- Service type change
- Probe parameter change
- Persistence or authentication parameter change

### Scenario 3: Rollback at the Release level when changes affect multiple objects
If a Helm upgrade modifies more than just a Deployment and affects a group of objects, it's more suitable to rollback at the Release level.

### Operations understanding focus
Helm rollback is more suitable for scenarios where:

> **The issue isn't a single workload, but a problem with the entire application upgrade.**

---

## XIV. What to typically check after rolling back a Deployment

After executing:

    kubectl rollout undo deployment/demo-web -n app-demo

You shouldn't end here, but continue verification.

### 1. Check rollout status

    kubectl rollout status deployment/demo-web -n app-demo

### 2. Check rollout history

    kubectl rollout history deployment/demo-web -n app-demo

### 3. Check Pods and ReplicaSets

    kubectl get rs -n app-demo
    kubectl get pod -n app-demo -o wide

### 4. Perform application access verification
For example:
- Whether the page has recovered
- Whether the interface has recovered
- Whether the error rate has decreased
- Whether new alerts have disappeared

### Operations understanding focus
The rollback action itself does not mean recovery has succeeded,  
Verification after rollback is as important as verification after an upgrade.

---

## XV. What to typically check after rolling back a Helm release

After executing:

    helm rollback my-app 1 -n app-demo

You should typically continue checking the following layers.

helm status my-app -n app-demo

### 2. View Release History

    helm history my-app -n app-demo

### 3. Recheck Current Values

    helm get values my-app -n app-demo

### 4. Return to kubectl to Check Objects

    kubectl get deploy,sts,ds,svc,pod -n app-demo

### 5. Perform Application Access Verification
For example:
- Whether the page has recovered
- Whether middleware connections have recovered
- Whether service entry points have recovered
- Whether application logs have returned to normal

### Operations Understanding Focus
Helm rollback, although initiated from the application layer, still requires verification at:

- Release status
- Kubernetes object status
- Application actual availability

---

## Sixteen, Why "Rollback Success" Cannot Be Judged Only by Command Execution Success

### Reason 1: Command Success ≠ Business Recovery
Sometimes the command executes without errors, but:
- Pods are not yet all Ready
- Service has not truly recovered
- Dependency chains are still abnormal

### Reason 2: Object Recovery ≠ Business Recovery
For example:
- Deployment has recovered
- But issues caused by ConfigMap or Secret still exist
- External dependencies have changed

### Reason 3: Rollback Itself May Trigger New Problems
For example:
- Old version image is no longer pullable
- Old configuration is no longer compatible with current environment
- Data structure changes make old version programs unable to run directly

### Operations Understanding Focus
The true standard for successful rollback is not "command exit code is normal", but:

> **The application must return to a working business state.**

---

## Seventeen, A Minimalized Deployment Rollback Example Process

### 1. Check Current History

    kubectl rollout history deployment/demo-web -n app-demo

### 2. Observe Current Abnormal Status

    kubectl get pod -n app-demo -o wide
    kubectl logs -n app-demo <pod-name>

### 3. Execute Rollback

    kubectl rollout undo deployment/demo-web -n app-demo

### 4. Check Rollout Status

    kubectl rollout status deployment/demo-web -n app-demo

### 5. Check Object Status After Rollback

    kubectl get deploy,rs,pod -n app-demo

### 6. Perform Business Verification
For example:
- Access the page
- Call an interface
- Check whether error logs have recovered

---

## Eighteen, A Minimalized Helm Rollback Example Process

### 1. Check Release History

    helm history my-app -n app-demo

### 2. Check Current Release Status

    helm status my-app -n app-demo

### 3. Execute Rollback to Specified Revision

    helm rollback my-app 1 -n app-demo

### 4. Check Status After Rollback

    helm status my-app -n app-demo

### 5. Recheck Object Status

    kubectl get deploy,sts,svc,pod -n app-demo

### 6. Perform Application Verification
For example:
- Business access recovery
- Middleware connection recovery
- Logs returning to normal

---

## Nineteen, Common Misconceptions in Rollback

### 1. Blindly Undo Without Checking History
This may lead to reverting to an inappropriate version.

### 2. Understanding Rollback as "Ending After Command Execution"
In reality, complete verification is still required.

### 3. Confusing Deployment Rollback and Helm Rollback Layers
One is object layer, the other is application layer.

### 4. Assuming Any Problem Should Be Immediately Rolled Back
Some issues are transient fluctuations, others require immediate rollback. Whether to rollback should be judged based on business impact.

### 5. Assuming Rollback Is Always Risk-Free
Old versions may still have:
- Image not pullable
- Configuration incompatibility
- Dependency inconsistency
- Data format incompatibility

### Operations Understanding Focus
Rollback is an important capability, but it's not a "press and it's all good" button. It still requires judgment and verification.

---

## Twenty, The Most Important Recognitions in This Article

### 1. Rollback Is Part of the Release Cycle, Not an Additional Action
This is the first recognition.

### 2. `kubectl rollout undo` Mainly Targets Deployment Object Layer
This is the second recognition.

### 3. `helm rollback` Mainly Targets Release Application Layer
This is the third recognition.

### 4. Check History Before Rollback, Must Perform Verification After Rollback
This is the fourth recognition.

### 5. The Standard for Rollback Success Is Business Recovery, Not Command Success
This is the fifth recognition.

---

## Twenty-One, Stage Summary

In the application release process, rollback capability is as important as update capability.

Through this article, you should at least establish the following basic understandings:

- Why the release chain must include rollback
- Why rollback cannot rely solely on manual "reversion"
- What problems Deployment rollback and Helm rollback respectively solve
- Why you should check history before rollback
- Why you still need to check objects, logs, and business access results after rollback

As long as this layer of understanding is clear, the release, update, and rollback process will form a basic closed loop.

---

## Twenty-Two, Keyword Mnemonics

- Rollback: Return to historical stable state
- rollout undo: Deployment object layer rollback
- helm rollback: Release application layer rollback
- revision: Deployment historical revision version
- release revision: Helm Release historical revision version
- history: Check history before rollback
- Release Cycle: Update, verification, rollback on failure

---

## Twenty-Three, Operations Extended Understanding

The real difficulty in application release often lies not in "how to deploy", but in "what to do if deployment goes wrong".

As soon as you enter real change scenarios, you will inevitably encounter the following issues repeatedly:

- Whether the new version is really usable
- Whether the new configuration is really correct
- Whether to continue troubleshooting or immediately rollback
- Whether to rollback at the object layer or application layer

Therefore, the significance of this article is not just to supplement two commands, but to complete an important cognitive closed loop in the entire process:

> **Controllable release must include controllable rollback.**

---

## References
- Kubernetes Deployment Official Documentation
- Helm Official Documentation
- Kubernetes Official Documentation

---

## Next Day Suggestions
Next article suggestion to organize:

**Yes.05-Application Release, Update, and Rollback Stage Summary: General Methods from Object Changes to Controllable ReleaseI don't know.**