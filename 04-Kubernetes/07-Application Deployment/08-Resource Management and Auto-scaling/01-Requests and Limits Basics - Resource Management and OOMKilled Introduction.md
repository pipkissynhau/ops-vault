# 01-requests and limits: Resource Management and OOMKilled Introduction

## Document Notes
- Document Positioning: Kubernetes resource management basics introduction
- Applicable Stage: After completing Probe system basics, entering resource requests, resource limits, and container resource anomalies understanding
- Recommended Path: `04-Kubernetes/07-Application Deployment/08-Resource Management and Elastic Scaling/08-requests and limits: Resource Management and OOMKilled Introduction`

## Tags
#Kubernetes #requests #limits #ResourceManagement #OOMKilled #CPU #Memory #Pod #Containers #ApplyDeployment #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## I. Why Start Learning requests and limits Now

The previous mainline has already established these foundational capabilities:

- Image building and pushing
- Deployment / Service / NodePort
- ConfigMap / Secret
- imagePullSecrets
- livenessProbe / readinessProbe / startupProbe

At this point, applications are no longer just "able to run," but have started to involve:

- Health checks
- Startup phase protection
- Service access control

However, in real production environments, application stability is not only determined by:

- Whether the image is correct
- Whether the YAML is correct
- Whether the Probe configuration is correct

It also heavily depends on another question:

> **How much resources does the application actually consume, and how much resources does the platform allocate to it?**

If resource configuration is unreasonable, common issues may arise:

- Pod cannot be scheduled
- Container is killed by the system after startup
- Application frequently OOMKilled
- CPU becomes fully utilized, leading to extremely slow responses
- Node resources are completely consumed by a few Pods
- Multiple services compete for resources, leading to decreased overall stability

Therefore, starting from this article, we will officially enter:

> **Kubernetes resource management basics.**

The two most core concepts are:

- `requests`
- `limits`

---

## II. What Do requests and limits Actually Solve

They solve:

> **How Kubernetes understands how much resources a container "at least needs" and "maximum allowed to use."**

This may seem like a configuration detail, but it is actually very core because it simultaneously affects:

- Scheduling
- Running stability
- Resource isolation
- Service availability
- Node capacity planning

---

## III. First, Remember the Core Conclusion

If you remember one sentence first, recommend this:

### `requests`
Indicates:

> **I need at least this much resources; please schedule me based on this minimum requirement.**

### `limits`
Indicates:

> **I can only use up to this much resources; exceeding this upper limit will result in being restricted or killed.**

### Simplified Understanding
- `requests`: Minimum guarantee
- `limits`: Maximum cap

---

## IV. Why Kubernetes Can't Just Look at "Node Idle Resources"

Because without requests and limits, the platform struggles to make precise judgments.

For example, if a node has some idle CPU and memory, but you don't know how much new applications will need, you may encounter these issues:

### 1. Too Arbitrary Scheduling
Looks like it can be placed, but actual resources are insufficient.

### 2. Severe Resource Competition
Some containers suddenly spike, pushing other applications down.

### 3. Unpredictable Stability
Whether the service will slow down, whether it will be killed, or whether the node will be overwhelmed becomes uncontrollable.

Therefore, Kubernetes needs a mechanism to inform itself:

- How much resources this container normally needs at minimum
- How much resources it is allowed to use at maximum

This is the value of requests and limits.

---

## V. What Are CPU and Memory Resources

In Kubernetes, the most common and core resource management objects are typically:

- CPU
- Memory (Memory)

### CPU
Primarily represents computational resource usage.

### Memory
Primarily represents memory usage.

These two types of resources affect Pod stability, but their manifestations differ:

### Common CPU Insufficiency Manifestations
- Application becomes slower
- Response latency increases
- Task execution time lengthens

### Common Memory Insufficiency Manifestations
- Container may be directly killed
- Appears `OOMKilled`
- Pod restarts

---

## VI. What Is requests

`requests` can be understood as:

> **The minimum resource requirement the container requests.**

In other words, Kubernetes will first check:

- How much CPU this container declares it needs at minimum
- How much memory it needs at minimum

Then decide whether the node has enough available resources to accept it.

### Example
If a container declares:

- CPU request = 200m
- Memory request = 256Mi

The scheduler will consider:

> This container needs at least this much resources; scheduling must account for it.

---

## VII. What Is limits

`limits` can be understood as:

> **The maximum resource usage allowed for the container during operation.**

In other words, after the container runs:

- How much CPU it can use at maximum
- How much memory it can use at maximum

Exceeding the limit will trigger different consequences.

### CPU Exceeding limits
Usually gets throttled, i.e., "rate-limited."

### Memory Exceeding limits
Usually more dangerous, common results are:

> **The container is killed.**

Which is the frequent occurrence of:

- `OOMKilled`

---

## VIII. Why requests and limits Cannot Be Merged Into One Concept

This is a common confusion point for beginners.

### What Does requests Affect
Primarily affects:

> **Scheduling**

Kubernetes highly focuses on requests when "deciding which node to place the Pod."

---

### What Does limits Affect
Primarily affects:

> **Runtime boundaries**

After the container actually runs, exceeding the limit will trigger restrictions or killing.

---

### Simplified Distinction
- `requests`: Determines whether it can be scheduled
- `limits`: Determines how much it can consume at maximum

---

## IX. A Simplest requests / limits Example

Below is a basic Deployment snippet:

---
## 10. How to Understand This YAML

### 1. `resources`
Defines resource management rules for this container.

### 2. `requests.cpu: 200m`
Indicates this container requires at least 0.2 CPU cores.

### 3. `requests.memory: 256Mi`
Indicates this container requires at least 256Mi memory.

### 4. `limits.cpu: 500m`
Indicates this container can use up to 0.5 CPU cores.

### 5. `limits.memory: 512Mi`
Indicates this container can use up to 512Mi memory.

### Operations Understanding Focus
This means:

- Scheduling reserves 200m CPU + 256Mi memory
- Runtime limits are capped at 500m CPU + 512Mi memory

---

## 11. What Does `m` Mean in CPU Units

This is a common foundational concept.

### `m`
Represents millicore, which is:

> **One-thousandth of a core**

Examples:

- `1000m` = 1 core
- `500m` = 0.5 cores
- `200m` = 0.2 cores

### Simplified Understanding
If you see:

    cpu: "250m"

You can understand it as:

> Requesting 0.25 CPU cores

---

## 12. What Does `Mi` Mean in Memory Units

In Kubernetes, memory is typically written as:

- `128Mi`
- `256Mi`
- `512Mi`
- `1Gi`

### `Mi`
Represents Mebibyte.

### `Gi`
Represents Gibibyte.

At this stage, you don't need to focus excessively on binary conversion details. Just understand it as:

- `Mi`: Memory unit, close to common MB scale
- `Gi`: Memory unit, close to common GB scale

### Practical Focus
The key is to develop a sense of scale, for example:

- `128Mi`: Small
- `256Mi`: Common starting point for lightweight applications
- `512Mi`: More lenient
- `1Gi` and above: Common for medium to heavy applications

---

## 13. What Happens If You Don't Specify requests

If you don't specify requests, common issues include:

### 1. Scheduling Lacks Clear Resource Estimation
Scheduler has inaccurate perception of actual needs.

### 2. Containers Easily Compete with Other Pods for Resources
Because the platform lacks reasonable reservation awareness.

### 3. Nodes Are More Likely to Be "Overloaded"
Multiple Pods may seem schedulable, but resources are actually very tight.

### Operations Understanding Focus
Not specifying requests makes it harder for the platform to make "minimum resource allocation" judgments.

---

## 14. What Happens If You Don't Specify limits

If you don't specify limits, common issues include:

### 1. Containers May Unrestrictedly Consume Resources
An abnormal service might consume large amounts of CPU or memory.

### 2. Affects Other Business Stability
Especially noticeable in multi-tenant or multi-service nodes.

### 3. Overall Node Risk Increases
A single container anomaly could potentially crash the entire machine.

### Operations Understanding Focus
Not specifying limits means the platform lacks "ceiling protection".

---

## 15. Why Do CPU and Memory Overlimit Behaviors Differ

This is a critical point, frequently encountered in troubleshooting.

### CPU Exceeds limits
Typically manifests as:

> **Restricted usage speed**

The application is still running, but:

- Slower
- Delayed responses
- Reduced throughput

### Memory Exceeds limits
Typically manifests as:

> **Immediately killed**

This is because memory is a more rigid resource.  
Once the limit is exceeded, the system usually terminates the container to protect overall stability.

### Operations Understanding Focus
Therefore, during troubleshooting:

- CPU issues often appear as "slowness"
- Memory issues often appear as "crashes"

---

## 16. What Is OOMKilled

`OOMKilled` is one of the most frequent Kubernetes/container abnormal states.

It usually indicates:

> **The container was killed by the system due to memory overlimit or memory shortage.**

### The OOM Here
Means:

> Out Of Memory

Which means memory is insufficient.

### Common Scenarios
- Container actual memory usage exceeds `limits.memory`
- Memory leaks in the program
- Configuration set too small
- Startup memory peak higher than expected

---

## 17. What Phenomena Does OOMKilled Typically Show

In practice, you'll typically see these phenomena:

### 1. Pod Continuously Restarts
Especially in Deployment-managed environments, old containers are killed and then restarted.

### 2. OOMKilled Appears in Container Status
This is a direct indication.

### 3. Intermittent Service Unavailability
Due to containers being repeatedly killed, service instability occurs.

### 4. More Issues During Startup
Some applications consume more memory during startup than during stable operation. If the limit is too small, it's easy to be killed during startup.

---

## 18. Why Is OOMKilled Important

Because it's not just a simple "program exit", but:

> **A forced intervention by the platform layer when resource control is lost.**

In other words, when troubleshooting OOMKilled, you shouldn't only focus on the code, but also consider:

- Whether the limit is too small
- Whether the request is reasonable
- Whether the application has memory peaks
- Whether there's a leak
- Whether it's a startup phase with high instantaneous usage

### Operations Understanding Focus
This type of issue clearly demonstrates:

> **Resource allocation itself is part of application deployment.**

---

## 19. Problems Caused by Setting requests Too Small

### 1. Underestimating Scheduling Demands
The scheduler thinks nodes are sufficient, but actual runtime pressure is significant.

### 2. Nodes May Be Overpacked
Too many Pods are placed, leading to intense resource contention.

### 3. Service Stability Deteriorates
Especially when multiple services "underreport" resource needs, the node layer becomes increasingly unstable.

### Operations Understanding Focus
Setting requests too low is essentially "overstating costs".

---

## Twenty, Problems When Limits Are Too Small

### 1. CPU Limit Too Small
Applications frequently get throttled, manifested as:

- Slow
- Stuck
- High latency

### 2. Memory Limit Too Small
Applications are more likely to be directly:

- `OOMKilled`

### 3. Startup Period Prone to Misjudgment
Memory peak during startup may temporarily be higher. If the limit is too tight, startup may fail.

### Operations Understanding Focus
Setting limits too low appears to save resources, but may actually directly crash the application.

---

## Twenty-One, Is It Better to Set Requests and Limits Higher?
Of course not.

### Setting Too High Also Causes Problems

#### 1. Requests Too High
Causes scheduling difficulties.  
Nodes may still have resources, but the scheduler thinks they can't accommodate.

#### 2. Limits Too High
Results in overly broad resource boundaries. Some abnormal applications may consume too much resources, affecting other services.

### Operations Understanding Focus
The goal of resource configuration is not "as large as possible" or "as small as possible", but:

> **As close as possible to actual business needs, with reasonable redundancy reserved.**

---

## Twenty-Two, A Common Practice Approach for Requests and Limits

At this stage, establishing a basic approach is sufficient:

### Requests
Estimate based on:

- The foundational needs for normal stable operation

### Limits
Set based on:

- Normal fluctuations + certain redundancy space

### Simplified Understanding
- Request: Daily baseline
- Limit: Peak cap

---

## Twenty-Three, How Is This Topic Related to the Subsequent TroubleshootingMain?
When entering `14-Apply deployment barriers`, you will definitely encounter these issues:

- Pods can't start
- Containers repeatedly restart
- Services are very slow
- Even when the image and configuration are correct, the business is unstable
- Containers are OOMKilled
- Node resource tension causes scheduling failures

Without the prerequisite understanding of requests/limits, these issues are easily misjudged as:

- Application code issues
- Kubernetes bugs
- Image issues

In reality, it's often:

> **Resource configuration and actual application needs are mismatched.**

---

## Twenty-Four, The Most Important Understandings in This Topic

### 1. Requests Mainly Affects Scheduling
It tells the scheduler the minimum resources this container needs.

### 2. Limits Mainly Affects Runtime Boundaries
It determines the maximum resources the container can use.

### 3. CPU and Memory Overlimit Behaviors Differ
- CPU: More commonly manifests as slowness
- Memory: More commonly results in being killed

### 4. OOMKilled Is a Highly Frequent Practical Abnormality
Must establish basic judgment awareness.

### 5. Resource Configuration Should Neither Be Arbitrarily Small Nor Arbitrarily Large
Essentially, it's a balance between "actual needs + redundancy".

---

## Twenty-Five, What Level Should You Master to Learn This Topic?
At this stage, it's recommended to reach the following levels:

### 1. Be able to explain the difference between requests and limits
### 2. Understand why requests affect scheduling
### 3. Understand why limits affect runtime resource upper limits
### 4. Understand roughly where OOMKilled comes from
### 5. Be able to make initial resource-level judgments for "slow applications" and "applications being killed"

---

## Twenty-Six, Common Interview Follow-up Questions
Common questions in this area include:

- What's the difference between requests and limits?
- Why does requests affect scheduling?
- What's the difference between CPU limit and memory limit overlimits?
- What is OOMKilled?
- Why are containers OOMKilled?
- What problems occur when requests/limits are too small?
- What problems occur when requests/limits are too large?
- Why should you also suspect resource configuration issues when an application is slow?

---

## Twenty-Seven, Stage Summary
Requests and limits are the most fundamental and important concepts in Kubernetes resource management.

The most important part of this article isn't to learn all tuning methods, but to first establish the following core understandings:

- `requests` determines resource baseline and scheduling baseline
- `limits` determines resource upper limit and runtime boundaries
- CPU and memory overlimit behaviors differ
- OOMKilled is a high-frequency signal for memory-related issues
- Resource configuration itself is part of application deployment quality

As long as these understandings are established, when continuing to learn:

- Service / Endpoints
- Ingress
- PVC / Storage
- Application deployment troubleshooting

Many seemingly complex issues will be easier to trace back to whether there's a resource layer problem.

---

## Twenty-Eight, Keyword Quick Notes
- requests: Minimum resource needs for the container
- limits: Resource usage upper limit for the container
- CPU request: CPU needs reserved during scheduling
- Memory request: Memory needs reserved during scheduling
- CPU limit: CPU usage upper limit
- Memory limit: Memory usage upper limit
- OOMKilled: Container killed due to memory insufficiency or overlimit
- m: CPU's millicore unit
- Mi / Gi: Common memory units

---

## Twenty-Nine, Operations Extended Understanding
From an operations perspective, requests and limits are not "optional optimization items", but a resource covenant between the platform and business.

Without this covenant:

- The scheduler doesn't know how to reasonably place Pods
- Node resources are easily exhausted
- Services are more likely to interfere with each other
- It's harder to determine whether the issue is configuration or resource-related when problems occur

Therefore, learning requests/limits isn't just for writing a YAML, but to start truly understanding:

> **Kubernetes isn't just helping you run applications, but also helping you stably manage applications under limited resources.**

This is also a necessary resource perspective you must have when entering application deployment troubleshooting later.

---

## References
- Kubernetes Resource Management for Pods and Containers: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- Assign CPU Resources to Containers and Pods: https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/
- Assign Memory Resources to Containers and Pods: https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/

---

## Next Day Suggestions
Next article suggests organizing:

[[02-Resource Requests Impact on Pod Scheduling - From Pending to Scheduling Decision]]
 /think