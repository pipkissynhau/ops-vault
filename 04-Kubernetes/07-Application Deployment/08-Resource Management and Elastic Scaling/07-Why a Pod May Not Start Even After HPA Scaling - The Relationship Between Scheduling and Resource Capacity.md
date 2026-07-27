# 07-Why a Pod May Not Start Even After HPA Scaling: The Relationship Between Scheduling and Resource Capacity

## Document Description
- Document Purpose: Introduction to troubleshooting scenarios where HPA scaling fails
- Applicable Phase: For those who have understood the basics of HPA, how requests affect scheduling, and what it means for Pods to be in a `Pending` state due to node resource constraints, this section addresses the typical scenario where "HPA has been triggered, but new replicas still cannot take effect"
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/08-Resource Management and Auto Scaling/07-Why a Pod May Not Start Even After HPA Scaling: The Relationship Between Scheduling and Resource Capacity`

## Tags
#Kubernetes #HPA #HorizontalPodAutoscaler #Pending #Scheduling Failure #requests #Resource Capacity #ClusterAutoscaler #Auto Scaling #Resource Management #Application Deployment #Cloud-Native #Ops #Interview Notes

---

## I. Why Write This Article Separately

Many people, after learning about HPA, form a natural but incomplete understanding:

- When CPU usage increases,
- HPA will automatically scale up,
- More Pods will be created,
- And the business issue will be solved.

However, in real-world scenarios, it is common to encounter such a situation:

- HPA has indeed been triggered,
- The number of Deployment replicas has increased,
- But new Pods are not actually running,
- And the business still cannot handle the traffic.

If you only know the definition of HPA at this point, you will be confused. Therefore, the core issue this article aims to address is:

> **Why might a new Pod not start even after HPA has decided to scale up?**

This article is not just about explaining concepts; it focuses on connecting the following steps together:

### 1. HPA detects high metrics
### 2. HPA increases the number of replicas
### 3. New Pods are created
### 4. The scheduler attempts to find nodes
### 5. Nodes lack sufficient resources
### 6. New Pods remain in a `Pending` state
### 7. Scaling seems to have occurred, but business benefits are limited

---

## II. Observing the Phenomenon: What You Will See

The most typical symptoms of this issue are as follows:

### 1. HPA is indeed working
Run:

    `kubectl get hpa`

You will see that:

- `TARGETS` is higher than the target value,
- `REPLICAS` has started increasing.

---

### 2. The number of Deployment replicas has also changed
Run:

    `kubectl get deploy`

You will see that:

- The desired number of replicas has increased from 2 to 3, 4, or 5.

---

### 3. But new Pods are not fully running
Run:

    `kubectl get pod`

You will see that:

- Some new Pods are in the `Pending` state,
- Or new Pods seem stuck during startup.

---

### 4. Upon checking Pod details, it turns out to be a resource scheduling issue
Run:

    `kubectl describe pod <pod-name>`

The Events section may show:

- `Insufficient cpu`
- `Insufficient memory`
- `0/3 nodes are available`

### Key Points for Ops Professionals

The most critical thing to understand in this scenario is that:

> **HPA only decides whether to add more Pods; it does not guarantee that there will be enough resources to host these Pods.**

---

## III. Concluding: HPA Does Not Equal Automatically Sufficient Capacity

This is the most fundamental point of this article:

> **HPA is responsible for scaling up Pods, but it does not create additional node capacity.**

In other words:

- When HPA detects high load,
- It will increase the number of replicas,
- But the scheduler still needs to determine whether there are enough nodes to accommodate these new Pods.

If nodes are already under heavy strain, the result will be:

- HPA has scaled up,
- New Pods have been created,
- But scheduling fails,
- New Pods remain in a `Pending` state.

### Key Points for Ops Professionals

Therefore, what HPA addresses is:

> **Whether to add more Pods**

It does not directly solve the issue of:

> **Whether the cluster has enough resources to handle these new Pods**

---

## IV. Why Does "HPA Has Scaled Up, But Pods Still Cannot Start" Happen?

The most common reasons fall into two categories:

### 1. Insufficient node resources
This is the most typical case.

For example:

- The CPU requests are too high,
- The memory requests are too high,
- Nodes are already quite full,
- New replicas have no place to land.

---

### 2. Other issues within the Pod startup process
For example:

- Slow image retrieval,
- Probe failures### At this stage, the focus is not on which specific tool to use, but rather on achieving the following objectives:

- When you run `kubectl top pod`, you should see an increase in the CPU usage of the nginx-web Pod.
- The `TARGETS` value in `kubectl get hpa` should also increase.
- The HPA mechanism should start scaling out.

---

## XI. Observation commands recommended during experiments

It is suggested to execute 4 sets of commands simultaneously to clearly understand the entire process.

### 1. Checking HPA settings

    kubectl get hpa -w

Focus on:

- `TARGETS`
- `REPLICAS`

---

### 2. Checking Pods

    kubectl get pod -w

Pay attention to:

- Whether new Pods have been created.
- If the new Pods are stuck in the `Pending` state.

---

### 3. Checking Deployment settings

    kubectl get deploy -w

Focus on:

- `DESIRED`
- `AVAILABLE`
- `READY`

---

### 4. Monitoring resource metrics

    kubectl top pod

Check mainly:

- Whether the CPU usage of the nginx-web Pod continues to increase.

---

## Key points for operations engineers to understand

When conducting experiments with HPA, it is easy to miss important details if you only focus on one command.  
At minimum, you should observe the following aspects simultaneously:

- Metrics
- HPA settings
- Deployment status
- Pod behavior

---

## XII. Expected outcomes in a successful experiment

If the experiment is successful, the typical sequence of events will be as follows.

### Step 1: Initial state
The Deployment has 2 replicas.

Execute the following commands:

    kubectl get deploy
    kubectl get pod
    kubectl get hpa

You will see:

- The Pods are running normally.
- The HPA target value is normal.
- The number of replicas is 2.

---

### Step 2: Applying load
After continuously requesting services, execute the following commands again:

    kubectl top pod
    kubectl get hpa

You will observe:

- The CPU usage of the Pods starts to increase.
- The `TARGETS` value of the HPA approaches or exceeds the target level.

---

### Step 3: HPA starts scaling out
Continue monitoring and execute:

    kubectl get hpa -w
    kubectl get deploy -w

You will see:

- The number of `REPLICAS` of the Deployment increases.
- The desired number of replicas also begins to rise.

---

### Step 4: New Pods are created
Execute:

    kubectl get pod -w

You will notice:

- New Pod names appear.
- However, not all of them may be running immediately.

---

### Step 5: Some new Pods remain Pending
If the node capacity is insufficient, a common scenario is:

- Some Pods are running normally.
- Some newly created Pods remain in the `Pending` state.

---

## Key points for operations engineers to understand

This step highlights the most critical observation in this experiment:

> **Even though the HPA has scaled out, not all new replicas have been successfully deployed.**

---

## XV. How to troubleshoot when Pods remain Pending
When you see that new Pods are in the `Pending` state, do not immediately check the application logs.

In many cases:

- The containers have not even started yet.
- Checking the business logs is not very informative.

### The correct first step

First, describe the Pending Pod using the following command:

    kubectl describe pod <pending-pod-name>

Focus on:

- `Conditions`
- `Events`

### Common issues you may encounter

For example:

- `Insufficient cpu`
- `Insufficient memory`
- `0/3 nodes are available`

---

## XVII. The relationship between this and Cluster Autoscaler
This is also an important point to understand.

### HPA is responsible for scaling out Pods
Its role is to determine whether to increase the number of replicas.

---

### Cluster Autoscaler is responsible for scaling out nodes
Its function is to decide whether to add more nodes when the existing ones are insufficient.

### The typical relationship is as follows

### 1. HPA detects high load → Scales out Pods
### 2. New Pods are created
### 3. New Pods remain Pending because there are not enough nodes
### 4. If Cluster Autoscaler is enabled in the cluster
### 5. CA may add more nodes
### 6. Only after new nodes are available will the Pending Pods be successfully scheduled.

---

## XVIII. Why this article is closer to real production scenarios than "HPA Basics"
In production, the real challenge is not about knowing how to write HPA YAML configuration files, but rather whether the newly added replicas can actually contribute to business performance.

In a real environment, you are more likely to encounter situations where:

- The HPA takes effect, but### 6. Understanding Requests, Scheduling, Node Capacity, and HPA as a Connected Concept

---

## Twenty-Two: Common Follow-up Questions in Interviews

Common questions in this area include:

- Why might a Pod still fail to start up after HPA scaling?
- What is the relationship between HPA and the scheduler?
- Why does a newly scaled Pod remain in the Pending state?
- What are the differences between HPA and Cluster Autoscaler?
- If HPA is activated, why isn't there an obvious improvement in service performance?
- What should be checked first when troubleshooting the issue of "HPA scaling having no effect"?
- Why can requests affect the outcome after HPA scaling?

---

## Twenty-Three: Phase Summary

Understanding why a Pod might still fail to start up even after HPA scaling is a crucial concept in Kubernetes' elasticity framework.

The most important thing here is not to memorize a conclusion, but to establish the following key understandings:

- HPA is only responsible for scaling Pods, not nodes.
- Newly scaled Pods still need to go through scheduling and resource allocation processes.
- Resource requests directly affect whether new Pods can be allocated to nodes.
- When node capacity is insufficient, HPA scaling may only result in an increase in the number of replicas at the surface level.
- True elasticity means that not only should HPA take effect, but the additional replicas also need to become Running/Ready and contribute to service performance.

Once these concepts are firmly established, understanding other related topics such as Cluster Autoscaler, HPA stability optimization, resource capacity planning, and peak-load elasticity management will become much clearer.

---

## Twenty-Four: Key Terms for Quick Reference

- HPA: Horizontal Pod Autoscaling
- Pending: A Pod has been created but has not yet been successfully scheduled or completed the preliminary stages.
- Requests: Resource requests, which are an important basis for scheduling.
- Scheduler: The component responsible for assigning Pods to nodes.
- Insufficient cpu: Insufficient CPU resources.
- Insufficient memory: Insufficient memory resources.
- Cluster Autoscaler: Node autoscaling mechanism.
- Elasticity chain: Metrics → HPA → Change in the number of replicas → Scheduling → Startup → Ready status.

---

## Twenty-Five: Operational Perspective for Deeper Understanding

From an operational perspective, the goal is to develop a "complete-chain thinking" approach.

Many people only learn about HPA in terms of how it increases the number of replicas when metrics rise. However, in real production environments, whether the service truly benefits depends on whether the entire chain functions smoothly:

- Are the metrics being collected correctly?
- Has HPA made a decision to scale up?
- Has the Deployment actually increased the number of replicas?
- Has the new Pod been successfully scheduled?
- Has the new Pod started up promptly?
- Has the new Pod passed the readiness checks and begun handling traffic?

If any link in this chain is broken, the final result will be:

> **It seems like scaling has occurred, but the actual service performance hasn't improved.**

This is why this content aims to integrate previously learned concepts such as requests, Pending status, QoS, node capacity, and HPA into a comprehensive practical framework.

---

## References
- Kubernetes Horizontal Pod Autoscaling
- Kubernetes Resource Management for Pods and Containers
- Official documentation on metrics-server
- Official documentation on Cluster Autoscaler
- Common output checks using `kubectl describe pod`/`describe node`

---

## Next Steps
It is recommended to organize the following content in the next article:

[[08-VPA Basics: Vertical Scaling, Prerequisites for Installation, and Basic Practices]]