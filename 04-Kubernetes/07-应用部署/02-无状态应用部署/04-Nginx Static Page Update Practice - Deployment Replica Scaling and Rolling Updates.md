# 04-Nginx Static Page Update Practice: Deployment Replica Scaling and Rolling Update Basics

## Document Notes
- Document Positioning: Basic Practice of Stateless Application Updates and Scaling
- Applicable Stage: After completing basic practices of Deployment, Service, and NodePort
- Recommended Path: `04-Kubernetes/07-Apply deployment/02-No status application deployment/03-Nginx Static page update practice:Deployment Copy amplification and rolling update base`

## Tags
#Kubernetes #Deployment #ScrollUpdate #CopyAmplification #Nginx #StaticPage #NoStatusApplication #ApplyDeployment #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## I. Why Learn Replica Scaling and Rolling Updates

The previous practices have established this foundational workflow:

- Image building
- Container runtime
- Deployment creation of Pods
- Service providing access entry
- NodePort enabling external cluster access

But a truly operational business system will not remain in the "single replica + immutable version" state forever.

In real production environments, at least two very common actions will be encountered:

### 1. Scaling
For example:

- Single replica insufficient to support traffic volume
- Need to improve availability
- Need to implement redundant deployment

### 2. Updates
For example:

- Page content has changed
- Image version has changed
- Business code has fixed a bug
- Need to release a new version

Therefore, learning Deployment replica scaling and rolling updates is learning:

> **How Kubernetes enables stateless applications to complete scaling and version switching without downtime or minimal disruption.**

---

## II. What Problems Does This Practice Solve

After completing this practice, it is recommended to at least achieve the following:

### 1. Understand the role of `replicas`
### 2. Understand why stateless applications are suitable for multiple replicas
### 3. Understand why Deployment can perform rolling updates
### 4. Distinguish between "creating a Pod" and "updating a Pod"
### 5. Be able to troubleshoot basic anomalies in scaling and rolling updates
### 6. Be able to explain this logic as an interview answer

---

## III. Why Nginx Static Pages Are Suitable for This Exercise

Nginx static pages are typical stateless applications with these characteristics:

- Do not depend on local critical state
- Any replica can provide the same page content
- Replicas have no identity differences
- A Pod deleted can be replaced by a newly launched one
- Very suitable for scaling
- Very suitable for rolling updates

Therefore, it is very suitable for observing:

- How Deployment maintains multiple replicas
- How Service distributes traffic to multiple Pods
- How old and new Pods switch during image updates

---

## IV. What Is Replica Scaling

Replica scaling essentially means:

**Increasing the number of application instances from a small number to a larger number.**

For example:

- Scaling from 1 replica to 2
- Scaling from 2 replicas to 3

In Deployment, the most direct manifestation is:

    replicas: 3

It indicates the desired number of Pods running that match the template definition.

### Operations Understanding Focus
Replica scaling is not manually running several containers, but rather:

- Informing Kubernetes of the desired number of replicas through declarative configuration
- Deployment continuously maintaining this number
- Adding when fewer, removing when more

---

## V. Why Stateless Applications Are Suitable for Scaling

Stateless applications are naturally suitable for multi-replica deployment because they typically have the following characteristics:

### 1. No Identity Differences Between Replicas
Any replica can be replaced by another.

### 2. No Dependence on Local Critical Data
Pod deletion will not cause business unrecoverability due to local data loss.

### 3. Easier Load Balancing
Service can distribute requests to multiple Pods.

### 4. Easier to Improve Availability
Even if a Pod fails, other Pods can continue providing services.

### Example with Nginx Static Pages
If page content comes from the same image, three Pods will return nearly identical content.  
This is a standard stateless scaling scenario.

---

## VI. A Simple Multi-Replica Deployment Example

Below is a 3-replica Deployment example:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-web
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx-web
      template:
        metadata:
          labels:
            app: nginx-web
        spec:
          containers:
            - name: nginx-web
              image: harbor.example.com/demo/nginx-web:v1
              ports:
                - containerPort: 80

### Key Changes in This YAML
Compared to the single-replica version, the most core change is:

    replicas: 3

This indicates the Deployment expects to maintain 3 replicas.

---

## VII. What Changes in the Access Chain After Scaling

In a single-replica scenario, Service typically has only one Pod behind it.

In a multi-replica scenario, the access chain becomes:

**Client → Service → One of Multiple Backend Pods → Container Port**

That is:

- The Service still has a stable entry point
- But there is no longer just one Pod behind it
- Service distributes traffic to multiple backend instances

### Operations Understanding Focus
After scaling, Service does not need to change the selector or re-define access methods.  
Because it was originally "facing a group of Pods."

---

## VIII. What Is Most Worth Observing After Scaling

When performing replica scaling, it is recommended to focus on observing the following:

### 1. Whether Deployment Generated Multiple Pods as Expected
For example, from 1 to 3.

### 2. Whether All Pods Are Running and Ready
It's not enough to just have the right number; all replicas must be truly available.

### 3. Whether Service Selected All Replicas
If the selector is normal, it should theoretically select all tagged Pods.

### 4. Whether Page Access Remains Continuous
Scaling should not affect the continuity of existing services.

---

## IX. What Is Rolling Update

Rolling Update is a common update method for Deployment by default.

Its core idea is:

> **Not deleting all old Pods first and then starting new ones, but gradually switching versions by starting new replicas and decommissioning old ones.**

The benefits of this approach are:

- Avoiding complete service interruption
- Reducing update risks
- More suitable for stateless applications to maintain continuous service

---

## X. How Is Rolling Update Usually Triggered

For Deployment, rolling update is typically triggered whenever the Pod Template changes.

The most common changes include:

### 1. Image tag changes
For example:

- From `nginx-web:v1`
- Updated to `nginx-web:v2`

### 2. Container parameter changes
Such as command, environment variable changes.

### 3. Label and configuration changes in Pod templates
Any changes at the template layer may trigger the creation of new ReplicaSet and new Pod.

---

## 11. How to Perform a Simple Rolling Update Scenario

Assume you already have a running Deployment:

- Current image: `nginx-web:v1`
- Current replica count: 3

Now you have rebuilt a new image:

- New image: `nginx-web:v2`

Just change the image in the Deployment to:

    image: harbor.example.com/demo/nginx-web:v2

Then reapply the YAML, and the Deployment will typically start rolling updates.

---

## 12. What Usually Happens During a Rolling Update

During a rolling update, common behaviors include:

### 1. A new ReplicaSet is created
This indicates the Deployment is starting to use the new Pod template.

### 2. New Pods are gradually launched
These Pods use the new image or new template.

### 3. Old Pods are gradually reduced
They are not deleted all at once, but replaced in batches.

### 4. Service continues to point to available Pods
As long as the Pod is Ready, the Service will continue to forward traffic.

---

## 13. Why Rolling Updates Are Particularly Suitable for Stateless Applications

Stateless applications are particularly suitable for rolling updates, reasons include:

### 1. Any replica can be replaced
After the old Pod is decommissioned, the new Pod can take over traffic as long as its content is consistent or version-compatible.

### 2. No dependency on fixed identity
Unlike some stateful services, Pod identity changes do not bring cluster relationship issues.

### 3. Easier to gradually replace
As long as the new replica is Ready, the old replica can be exited.

### 4. Service itself works based on "replica groups"
The business side does not need to perceive Pod replacement details.

---

## 14. What Role Does Service Play During a Rolling Update

Service plays a critical role.

### Without Service
Clients may directly depend on a Pod IP. Once the Pod is replaced, access may be interrupted or become invalid.

### With Service
Clients always access a stable entry point.  
Whether the backend Pod is old or new version, Service is responsible for tracking available replicas and forwarding traffic.

---

## 15. What Issues Are Most Likely to Occur During a Rolling Update

### 1. New image pull failure
For example:

- Tag written incorrectly
- Image not pushed successfully
- Private registry authentication failure

The result is usually:

- New Pod cannot start
- Update gets stuck

### 2. New Pod startup failure
For example:

- Page content not included in the image
- Nginx startup anomaly
- Dockerfile issues

### 3. Pod Ready not passed
If probes are added later, new Pod may remain Not Ready, causing traffic to fail smooth transition.

### 4. Image changed but content remains the same
For example, new tag actually hasn't updated page content, leading to the appearance of "update not taking effect".

### 5. Page inconsistency after rolling update
For example, mixed use of old and new pages in multiple replicas, which may be due to incomplete update or image version management chaos.

---

## 16. What's the Difference Between Scaling and Rolling Update

These two actions are often confused, but they are fundamentally different.

### Scaling
The goal is:

**Increase replica count.**

For example:

- 1 → 3

Focus is on capacity and availability improvement.

### Rolling Update
The goal is:

**Gradually replace old version replicas with new version replicas.**

For example:

- v1 → v2

Focus is on version switching.

### They may also happen simultaneously
For example:

- Replica count changes from 2 to 3
- Image changes from v1 to v2

In this case, Deployment will still converge to the target state gradually.

---

## 17. How to Determine if Scaling is Successful

Recommend checking at least the following dimensions.

### 1. Pod count reaches expected number
For example, it actually changes from 1 to 3.

### 2. All Pods are Ready
Quantity is sufficient but not Ready, it's not truly available.

### 3. Service selects all Pods
Confirm the Service backend includes the new replicas.

### 4. Page access remains normal
Scaling itself should not cause page unavailability.

---

## 18. How to Determine if Rolling Update is Successful

Recommend checking at least the following dimensions.

### 1. New image version is referenced
Deployment configuration should point to the new image.

### 2. New Pod is created successfully
It's not just changing YAML that counts.

### 3. Old Pod is gradually exited
Indicates replacement process is happening.

### 4. Page content is updated to expected version
This is the most direct business verification.

### 5. Service remains continuously accessible
Avoid obvious interruptions during the update process.

---

## 19. What to Check First When Rolling Update Fails

Recommend checking in this order.

### 1. Check Deployment status
Determine if the update has started or is stuck.

### 2. Check Pod status
Confirm new Pod is:

- Pending
- ImagePullBackOff
- CrashLoopBackOff
- NotReady

### 3. Check events
Events often directly expose:

- Image pull failure
- Scheduling failure
- Mount failure
- Startup failure

### 4. Check container logs
Confirm new version truly starts successfully.

### 5. Check Service and Endpoints
Confirm traffic has switched to new replicas.

---

## 20. How to Answer About Replica Scaling in Interviews

You can answer:

> Deployment is suitable for stateless applications, replica count is controlled by replicas. When scaling, it's essentially increasing the desired replica count, and Deployment automatically fills new Pods. Since stateless applications typically have no identity differences between replicas, they are very suitable for load distribution and multi-replica fault tolerance via Service.

---

## 21. How to Answer About Rolling Update in Interviews

You can answer:

> Deployment natively supports rolling updates. For stateless applications, when image or Pod template changes, Deployment creates new ReplicaSet and gradually launches new Pods, reduces old Pods, trying to complete version replacement without service interruption. Service continues to forward traffic to Ready Pods, so clients don't need to perceive backend replica replacement process.

## Twenty-Two: The Most Important Cognitive Aspects of This Practice

### 1. Replica Scaling and Rolling Updates Are Core Capabilities of Deployment
This is also the key reason why Deployment is suitable for stateless applications.

### 2. Service Makes Multi-Replica and Update Processes Transparent to Clients
Clients always face a stable entry point, not a single Pod.

### 3. Scaling Focuses on Replica Count, Updates Focus on Version Replacement
These two concepts must be clearly distinguished.

### 4. Not All Deployment Changes Guarantee Success
Validation must combine Pod status, logs, events, and access results.

### 5. Stateless Applications Are Particularly Suitable for Multi-Replica and Rolling Updates
Because replicas are interchangeable, there's no fixed identity, and state dependency is weak.

---

## Twenty-Three: What Level Should You Master This Topic?

At this stage, it's recommended to reach the following levels:

### 1. Understand the purpose of `replicas`
### 2. Explain why stateless applications are suitable for multi-replica deployment
### 3. Understand why Deployment supports rolling updates
### 4. Understand Service's role during update processes
### 5. Perform basic troubleshooting for replica scaling and rolling updates
### 6. Be able to explain this logic as a clear interview answer

---

## Twenty-Four: Common Interview Follow-Up Questions

Common questions in this area include:

- Why is Deployment suitable for stateless applications?
- What's the significance of multi-replica deployment?
- What role does Service play in multi-replica scenarios?
- What is rolling update?
- Why is rolling update suitable for stateless applications?
- What's the difference between rolling update and full-replacement update?
- Why does the page not change after updating the image?
- How to troubleshoot when rolling update gets stuck?
- Why is the access entry usually not changed during scaling?

---

## Twenty-Five: Stage Summary

Replica scaling and rolling updates for Nginx static pages are key practices for stateless applications to enter the "sustainable operation and sustainable release" stage.

The most important aspect of this article isn't memorizing several fields, but establishing these core cognitive frameworks:

- Deployment doesn't just create Pods, but also maintains replica counts and handles version updates
- Stateless applications are suitable for multi-replica scaling
- Stateless applications are suitable for rolling updates
- Service makes replica changes and version replacements more transparent to clients
- Scaling focuses on quantity, updates focus on version
- Whether an update succeeds must be verified through object status, logs, and final business results

As long as these relationships are clearly understood, subsequent learning about Probe, ConfigMap, Ingress, release strategies, and canary updates will be smoother.

---

## Twenty-Six: Keyword Mnemonics

- replicas: Desired replica count
- Scaling: Increase application replica count
- Rolling update: Gradually replace old Pods with new ones
- ReplicaSet: Intermediate control layer for Deployment managing replicas
- Ready: Pod is ready to receive traffic
- Stateless application: Replicas are interchangeable, no dependency on fixed local state
- Version replacement: Gradually switch from old image to new image

---

## Twenty-Seven: Operational Perspective Understanding

From an operational perspective, replica scaling and rolling updates aren't "advanced features," but the most fundamental and common daily operations for stateless applications.

From this point onward, you need to gradually develop true application release thinking:

- A service can't just run, it must also scale
- A version can't just go live, it must also smoothly replace
- An entry point can't depend on a specific Pod, but must depend on Service
- An update can't just check YAML changes, but must also verify if Pods, logs, and access results meet expectations

These mindsets will naturally extend to:

- Health checks
- Rollback
- Configuration updates
- Canary release
- Ingress layer-7 routing
- CI/CD continuous delivery

---

## References
- Kubernetes Deployment: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes ReplicaSet: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/
- Kubernetes Service: https://kubernetes.io/docs/concepts/services-networking/service/

---

## Next Day Recommendation
Next article suggestion: 

[[01-Nginx Static Page Configuration Decoupling - ConfigMap Mounting Fundamentals]]