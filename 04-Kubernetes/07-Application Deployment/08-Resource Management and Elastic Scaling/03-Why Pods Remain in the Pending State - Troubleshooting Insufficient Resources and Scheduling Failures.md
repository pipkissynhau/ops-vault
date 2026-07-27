# 03-Why Pods Remain in the Pending State: Troubleshooting Insufficient Resources and Scheduling Failures

## Document Description
- Document Purpose: Introduction to troubleshooting Pod Pending issues
- Applicable Stage: After understanding how requests affect scheduling, this section covers basic investigations into insufficient resources and scheduling failures
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/08-Resource Management and Auto Scaling/03-Why Pods Remain in the Pending State: Troubleshooting Insufficient Resources and Scheduling Failures`

## Tags
#Kubernetes #Pending #Scheduler #Insufficient Resources #Scheduling Failure #requests #Pod Troubleshooting #Resource Management #Application Deployment #Cloud-Native #Operations #Interview Notes

---

## I. Why Study Pod Pending Separately

In Kubernetes, `Pending` is a very common and critical state.

Many people starting with application deployment often assume that if a Pod doesn't start, it's because:

- There's an issue with the image
- The startup command is incorrect
- The application configuration is wrong
- The program failed to start

However, in many cases, Pods get stuck before even reaching the step of actually launching containers.

In other words:

> **When a Pod is in the Pending state, it doesn't necessarily mean that the application failed to start; rather, it may indicate that scheduling or preparation steps haven't been completed yet.**

Therefore, understanding Pod Pending is important because it helps you:

- Distinguish between “scheduling failures” and “runtime failures”
- Learn to check scheduling first, then the image, and finally the application
- Establish the correct order of troubleshooting in Kubernetes

---

## II. What Exactly Does Pod Pending Mean?

From a fundamental perspective, the lifecycle of a Pod can be roughly understood as follows:

### 1. The Pod object has been created
The YAML configuration has been submitted to Kubernetes.

### 2. The Pod has not yet started running properly
At this stage, it may not have successfully bound to any nodes, or some necessary prerequisites before startup may not have been met.

### 3. Therefore, its status is shown as `Pending`
This means:

> **Kubernetes has received the Pod, but scheduling or initialization to a runnable state has not been completed yet.**

### Key Points for Operations Professionals to Remember

You can remember it this way:

- `Pending`: The Pod hasn't started running yet
- `Running`: It has been scheduled and entered the running phase
- `CrashLoopBackOff`: Usually indicates repeated abnormal exits during runtime
- `ImagePullBackOff`: Typically indicates problems during the image retrieval process

So, `Pending` is more related to issues that occur in earlier stages of the Pod lifecycle.

---

## III. The Two Most Common Causes of Pod Pending

Although this article focuses on insufficient resources, it’s important to establish a general understanding first:

The common reasons for Pod Pending can generally be divided into two main categories.

### 1. The scheduler cannot find a suitable node
This is the most typical case.

For example:

- Insufficient CPU resources
- Insufficient memory
- Excessively large resource requests
- Too strict node selection criteria
- Mismatch between pod taints and node labels
- Failure to meet scheduling constraints

The core issue with these cases is:

> **The Pod has not been successfully assigned to any node.**

---

### 2. Although the Pod has been scheduled, some prerequisites are still unmet
For example:

- The PVC has not been bound successfully
- Dependent resources are not yet available
- Some initialization processes have not completed

These situations can also result in a Pending status.

### Key Points for Operations Professionals to Remember

When you see a Pod in the Pending state, the first step should not be guessing. Instead, you need to determine whether:

> **Scheduling was unsuccessful, or there are other prerequisites that need to be met after scheduling.**

---

## IV. Why This Article Focuses on “Insufficient Resources” First

In your current learning path, the most common, fundamental, and practical issue related to Pod Pending is:

> **Scheduling failures caused by insufficient resources.**

This is what you often see in event logs:

- `Insufficient cpu`
- `Insufficient memory`
- `0/3 nodes are available`

These issues are directly related to the concepts discussed in the previous section, such as:

- Resource requests and limits
- Resource-driven scheduling

Therefore, this article focuses on establishing this troubleshooting approach first:

> **Pod Pending → Describe Pod → Check Events → Determine if it’s a resource scheduling failure**

---

## V. Why Insufficient Resources Lead to Pending

This is the core causal relationship.

When the scheduler tries to assign a node to a Pod, it considers the resource requests specified in the Pod’s YAML configuration, such as:

- `requests.cpu`
- `requests.memory`

It then checks whether any available nodes can meet these requirements.

    cpu-memory-heavy-pod   0/1     Pending   0          1m

Execute again:

    kubectl describe pod cpu-memory-heavy-pod

You may see in the Events:

- `Insufficient cpu`
- `Insufficient memory`

---

## Section Twelve: Why Does a Pod Remain Pending Even Though the Node Seems to Have Resources?

This is a very common practical issue that confuses many people.

Many people will check:

- `top`
- `htop`
- Grafana
- Node CPU usage
- Node memory usage

And then say:

> "This node isn't even fully utilized, so why can't the Pod be scheduled?"

Here, it's important to emphasize another key concept:

> **The scheduler doesn't look at instantaneous usage rates but rather the allocatable resources and requested resources from the Kubernetes perspective.**

### It's Necessary to Distinguish Several Concepts

#### 1. Total Node Resources
How much CPU and memory does the machine have in total?

#### 2. Allocatable Resources
Resources that can actually be allocated to Pods.

#### 3. Allocated Requests
Resource requests that have been declared and occupied by existing Pods.

#### 4. Actual Usage at Run Time
Just the actual amount used at a particular moment; it doesn't mean whether scheduling is still possible.

### Key Points for Ops Professionals to Understand

So when troubleshooting Pending issues, you can't just rely on system monitoring. You also need to understand:

> **The scheduler makes its decisions based on the 'resource request perspective.'**

---

## Section Thirteen: Why Should You Also Check the Requests in the Pod YAML?

If the Events indicate:

- `Insufficient cpu`
- `Insufficient memory`

Then your next step should not be just focusing on the node but also looking back at:

> **How much resource has this Pod actually requested?**

Check the Deployment/Pod YAML for:

    resources:
      requests:
        cpu: ...
        memory: ...

### Many Problems Essentially Stem from These Issues

#### 1. Excessively High Requests
For ordinary services, requests of:

- `100m`
- `128Mi`

are usually sufficient, but if they are set to:

- `2 CPU`
- `4Gi`,

issues may arise.

---

#### 2. Requests That Don't Match the Capacity of the Test Environment
Copying configuration examples from production templates into smaller clusters can lead to resource overruns.

---

#### 3. A Single Pod May Seem Fine, but Too Many Replicas Can Cause Problems
The individual requests might be within limits, but the total amount exceeds what the node can handle.

---

## Section Fourteen: A Basic Order of Steps for Troubleshooting Resource-Related Pending Issues

It's recommended to follow this basic sequence when troubleshooting:

### Step One: Check the Pod Status

    kubectl get pod -n <namespace>

Confirm whether it is in the `Pending` state.

---

### Step Two: View Pod Details

    kubectl describe pod <pod-name> -n <namespace>

Focus on:

- `PodScheduled`
- `Events`
- Whether there are issues with `Insufficient cpu/memory`.

---

### Step Three: Examine Resource Configurations

Check the YAML for:

- `requests.cpu`
- `requests.memory`

---

### Step Four: Check Node Resources

You can use:

    kubectl describe node <node-name>

Pay attention to:

- `Allocatable`
- Allocated resources
- Node load.

---

### Step Five: Determine the Solution

There are usually two main approaches:

#### 1. Adjust Pod resource requests
For example, reduce the requested amounts.

#### 2. Increase the cluster's available capacity
This might involve adding more nodes, scaling out nodes, or using the Cluster Autoscaler.

---

## Section Fifteen: What Other Issues Can Cause a Pod to Remain Pending Besides Insufficient Resources?

Although this article focuses on resource shortages, it's important to know that:

> **Pending doesn't always mean there is only a resource issue.**

Other possible causes include:

### 1. PVC Not Bound Successfully
For example, the storage resources might not be available yet.

### 2. Node Selection Conditions Not Met
For instance:

- `nodeSelector`
- `nodeAffinity` settings might be too strict.

### 3. Taints and Tolerances Not Matching
The node might have taints, but the Pod doesn't have corresponding tolerations.

### 4. Excessive Affinity/Anti-Affinity Conditions Between Pods
This can prevent any nodes from meeting the requirements.

### Key Points for Ops Professionals to Understand

So when troubleshooting, don't assume that:

- Pending = Definitely a resource issue

However, at this stage, resource shortages are the most common and primary cause to consider.

---

## Section Sixteen: Why Is This Article Following Right After "How Requests Affect Scheduling"?

Because these two topics are closely related.

The## 24. Extended Understanding of Operations and Maintenance

From the perspective of operations and maintenance, "Pending" is a state that deserves great attention.

It reminds us that:

- Problems do not necessarily occur during the application's runtime.
- Many deployment failures are actually due to issues that arise in the early stages of the deployment process.
- Underlying factors such as scheduling, resources, storage, and node constraints determine the fate of a Pod even before the business logic starts running.

This is also why, in Kubernetes, a mature troubleshooting approach typically does not involve:

- Immediately examining logs or suspecting the application code at the start.

Instead, it focuses on first determining:

- What stage the Pod has reached.
- Whether there were issues with scheduling, image retrieval, or container startup.

Therefore, the value of understanding the "Pending" state is not just about learning its name but also about developing a:

> **phase-based troubleshooting mindset.**

---

## References
- Kubernetes Pod Lifecycle
- Basic Principles of the Kubernetes Scheduler
- Common Output Analysis Using `kubectl describe pod`
- Kubernetes Resource Management for Pods and Containers

---

## Next Steps
For the next article, it is recommended to organize the following content:

[[04-QoS Basics: Guaranteed, Burstable, and BestEffort]]