# 07-HPA Why Pods Might Still Fail to Start After Expansion: The Relationship Between Scheduling and Resource Capacity

## Document Notes
- Document Positioning: Introductory Troubleshooting for HPA Expansion Failure Scenarios
- Applicable Stage: After understanding HPA basics, the impact of requests on scheduling, and Pod Pending states under node resource pressure, entering the typical scenario of "HPA triggered but new replicas still fail to start"
- Recommended Path: `04-Kubernetes/07-Apply deployment/08-Resource management and flexibility/07-HPA Why did it expand? Pod It may also be impossible: the relationship between movement and resource capacity`

## Tags
#Kubernetes #HPA #HorizontalPodAutoscaler #Pending #ScheduleFailed #requests #ResourceCapacity #ClusterAutoscaler #AutomaticallyAmplified #ResourceManagement #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## One, Why Write This Article Separately

Many people after learning HPA form a natural but incomplete impression:

- CPU is high
- HPA will automatically scale out
- Pods increase
- Business issues are resolved

But in real environments, a typical scenario often occurs:

- HPA has indeed triggered
- Deployment replica count has indeed increased
- New Pods haven't truly started
- Business still can't handle traffic

If you only memorize HPA definitions, you'll be confused.

This article aims to solve the core question:

> **Why can new Pods still fail to start even though HPA has decided to scale out?**

This article isn't just about concepts, but to truly connect the following chain:

### 1. HPA detects metrics exceeding thresholds
### 2. HPA increases replica count
### 3. New Pod is created
### 4. Scheduler attempts to find a node
### 5. Node resources are insufficient
### 6. New Pod remains in Pending state
### 7. "Scaling out appears to have happened," but business benefits are limited

---

## Two, Describe the Phenomenon: What You'll See

The most typical phenomenon for this issue is usually as follows.

### 1. HPA is indeed working
Execute:

    kubectl get hpa

You'll see:

- `TARGETS` exceeds target value
- `REPLICAS` begins to increase

---

### 2. Deployment replica count has changed
Execute:

    kubectl get deploy

You'll see:

- desired replicas changes from 2 to 3, 4, 5

---

### 3. But new Pods haven't fully started
Execute:

    kubectl get pod

You'll see:

- Some new Pods are in `Pending`
- Or new Pods are stuck

---

### 4. After describing the Pod, you find it's a scheduling issue
Execute:

    kubectl describe pod <pod-name>

Events may show:

- `Insufficient cpu`
- `Insufficient memory`
- `0/3 nodes are available`

### Operations Understanding Focus

The key insight for this scenario is:

> **HPA only decides whether to add more Pods, but does not guarantee these Pods will have a place to run.**

---

## Three, Conclusion First: HPA ≠ Automatic Resource Sufficiency

This is the core sentence of this article:

> **HPA only handles scaling Pods, not creating node capacity.**

That means:

- HPA sees high load
- It increases replica count
- But the scheduler still needs to determine if nodes can accommodate these new Pods

If nodes are already strained, the result will be:

- HPA scales out
- Pods are created
- But scheduling fails
- New Pods remain in Pending state

### Operations Understanding Focus

So HPA addresses:

> **Whether to add more Pods**

It does not directly address:

> **Whether the cluster has sufficient resources to accommodate these new Pods**

---

## Four, Why "HPA Scales Out, But Pods Fail to Start"

The most common causes are actually two directions:

### 1. Node resources are insufficient
This is the most typical category.

For example:

- CPU requests are too high
- Memory requests are too high
- Nodes are already quite full
- New replicas have no place to land

---

### 2. Issues in the Pod startup chain itself
For example:

- Image pull is slow
- Probes are stuck
- PVC hasn't been ready
- Application startup is particularly slow

However, in this article, we focus first on the most common and core category:

> **New Pods created by HPA scaling fail to schedule due to insufficient resource capacity, resulting in Pending status.**

---

## Five, Minimum Experiment Goal

This article recommends using a minimal experiment to reproduce the phenomenon.

The experiment goal isn't to perform high-complexity stress testing, but:

> **Trigger HPA scaling, then intentionally make new Pods fail to schedule due to resource insufficiency, ultimately observing Pending status.**

You should observe the following chain, not the application's business logic:

### 1. Pod load increases
### 2. HPA determines scaling is needed
### 3. Deployment replica count increases
### 4. New Pods are created
### 5. New Pods fail to schedule due to resource insufficiency
### 6. describe Pod shows `Insufficient cpu` or `Insufficient memory`

---

## Six, Experiment Prerequisites

To make the experiment more successful, several prerequisites are needed:

### 1. Cluster has metrics-server installed and operational
First confirm:

    kubectl top node
    kubectl top pod -A

If you can't get metrics, HPA has no basis for resource data.

---

### 2. Cluster resources aren't overly abundant
If your cluster is very large with many nodes and ample resources, the experiment might not reliably reproduce Pending states.

More suitable scenarios are typically:

- Test cluster
- Few nodes
- Limited per-node resources
- Easier to create "scaled out but no place to land" phenomena

---

### 3. Deployment must specify requests
Because we need the scheduler to make decisions based on requests.

---

## Seven, Experiment Deployment Example

Below is a basic Deployment example.  
To more easily reproduce "scaled out but no place to land," we intentionally set requests higher than typical Nginx.

# Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web
spec:
  replicas: 2
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
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "500m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
```

### Apply Deployment

```bash
kubectl apply -f nginx-deploy.yaml
```

### Verify Deployment Status

```bash
kubectl get deploy
kubectl get pod -o wide
```

### Operations Understanding Focus

The most critical point here is:

- Do not arbitrarily write requests to extreme values
- Instead, write to "enough to create capacity pressure in a small cluster"

This makes it easier to observe phenomena.

---

## Eight, HPA Example Experiment

Here's a basic CPU-type HPA:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-web
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

### Apply HPA

```bash
kubectl apply -f nginx-hpa.yaml
```

### Check HPA Initial Status

```bash
kubectl get hpa
```

You'll typically see:

- `REPLICAS` is 2
- `TARGETS` may be relatively low

---

## Nine, Why Set averageUtilization to 50

Because we want:

> **Easier to trigger scaling**

If the target value is set too high, such as:

- 80%
- 90%

It may not be easy to trigger scaling in the experiment.

Setting it to 50 means:

- When average CPU utilization exceeds 50%
- HPA will more easily start scaling

### Operations Understanding Focus

The experimental parameters are not for "production optimization", but:

> **To make phenomena easier to observe.**

---

## Ten, How to Create CPU Pressure

The goal here is not to pursue a specific stress testing tool, but:

> **To make the CPU metrics of the nginx-web Pod group continuously higher than the HPA target value.**

Common approaches include:

### 1. Use Continuous Requests to Target the Service
For example, through a stress testing Pod or external tool, continuously request the nginx-web Service.

### 2. Use a Stress Testing Image
For example, run a dedicated stress testing container in the cluster to continuously generate request traffic.

At this stage, the core is not to use a specific tool, but to achieve the following effect:

- `kubectl top pod` see nginx-web CPU increase
- `kubectl get hpa`'s `TARGETS` increase
- HPA starts scaling

---

## Eleven, Recommended Observation Commands to Run Simultaneously During Experiment

It's recommended to run 4 groups of commands simultaneously to clearly see the phenomenon chain.

### 1. Watch HPA

```bash
kubectl get hpa -w
```

Focus on:

- `TARGETS`
- `REPLICAS`

---

### 2. Watch Pod

```bash
kubectl get pod -w
```

Focus on:

- Whether new Pods are created
- Whether new Pods are stuck at `Pending`

---

### 3. Watch Deployment

```bash
kubectl get deploy -w
```

Focus on:

- `DESIRED`
- `AVAILABLE`
- `READY`

---

### 4. Watch Resource Metrics

```bash
kubectl top pod
```

Focus on:

- Whether nginx-web's CPU continues to rise

### Operations Understanding Focus

For HPA experiments, if you only watch one command, it's easy to miss the full picture.  
At least observe simultaneously:

- Metrics
- HPA
- Deployment
- Pod

---

## Twelve, What Phenomenon You Ideally See

If the experiment is successful, the typical chain of events is usually as follows.

### Step 1: Initial State
Deployment has 2 replicas.

Execute:

```bash
kubectl get deploy
kubectl get pod
kubectl get hpa
```

You'll see:

- Pod is normally Running
- HPA target value is normal
- Replica count is 2

---

### Step 2: Start Applying Pressure
After continuous requests to the service, execute:

```bash
kubectl top pod
kubectl get hpa
```

You'll see:

- Pod CPU starts to rise
- HPA's `TARGETS` approaches or exceeds the target value

---

### Step 3: HPA Starts Scaling
Continue observing:

```bash
kubectl get hpa -w
kubectl get deploy -w
```

You'll see:

- HPA's `REPLICAS` increases
- Deployment's desired replica count starts to rise

---

### Step 4: New Pod is Created
Execute:

```bash
kubectl get pod -w
```

You'll see:

- New Pod name appears  
- But not necessarily all Running  

---

### Step 5: New Pods May Be Pending  
If node capacity is insufficient, a common phenomenon is:  

- Some Pods are Running  
- Some newly added Pods remain `Pending`  

### Operations Understanding Focus  

This step is the core phenomenon you're meant to see in this article:  

> **HPA expanded, but new replicas did not all take effect effectively.**  

---

## Step 13: What to Do First When Pods Are Pending  

When you see new Pods in `Pending` status, do not check application logs first.  

Because often:  

- Containers haven't started yet  
- Business logs are not meaningful  

### Correct First Step  

First describe the Pending Pod:  

    kubectl describe pod <pending-pod-name>  

Focus on:  

- `Conditions`  
- `Events`  

### You May See the Following Content  

For example:  

- `Insufficient cpu`  
- `Insufficient memory`  
- `0/3 nodes are available`  

### Operations Understanding Focus  

If you see such events, you should first judge:  

> **Is it node capacity insufficiency, not application configuration issues?**  

---

## Step 14: Why HPA-Expanded Pods May Be Pending  

The root cause is actually straightforward:  

### 1. HPA Only Adjusts Replica Target  
It increases the `replicas` of the Deployment.  

### 2. Deployment Creates Pods According to New Replica Count  
So new Pod objects appear.  

### 3. Scheduler Assigns Nodes to These Pods  
The problem lies here.  

### 4. If Node Resources Are Insufficient, New Pods Have No Place to Run  
Final result:  

- Pod Created  
- But Scheduling Failed  
- Status Becomes `Pending`  

### You Can Initially Understand It As  

> **HPA is responsible for "having more children," the scheduler is responsible for "finding rooms for the children."**  
>  
> The children were born, but there are not enough rooms, so they can't stay.  

---

## Step 15: What Relationship Does This Have With Previous "requests" Knowledge  

The core root cause of this article is actually content you've already learned:  

> **The scheduler mainly looks at requests, not limits.**  

If your Deployment writes:  

    resources:  
      requests:  
        cpu: "500m"  
        memory: "256Mi"  

Each new Pod means:  

- Another request space is occupied  

If node resources are already tight, then:  

- HPA wants to scale  
- But the scheduler can't accommodate it  

### Operations Understanding Focus  

Therefore, the essence of this article is not just "HPA issues," but:  

> **The combined result of HPA + scheduling + requests + node capacity.**  

---

## Step 16: A More Complete Troubleshooting Order  

When encountering "HPA scaled but Pods can't start," suggest following this order.  

### Step 1: Confirm HPA Actually Triggered  

    kubectl get hpa  

Focus on:  

- `TARGETS`  
- `REPLICAS`  

Confirm:  

- Whether HPA actually increased replica count  

---

### Step 2: Confirm Deployment Replica Count Changed  

    kubectl get deploy  

Confirm:  

- Whether desired replicas have increased  

---

### Step 3: Confirm New Pods Were Actually Created  

    kubectl get pod  

Confirm:  

- Whether new Pods appeared  

---

### Step 4: Check If New Pods Are Running or Pending  

If Pending, proceed to next step.  

---

### Step 5: Describe Pending Pod  

    kubectl describe pod <pod-name>  

Focus on:  

- `Events`  
- Whether there is `Insufficient cpu / memory`  

---

### Step 6: Check Node Capacity  

    kubectl describe node <node-name>  

Focus on:  

- Allocatable  
- Node Pressure  
- Resource Allocation Status  

### Operations Understanding Focus  

The troubleshooting chain for such issues is definitely:  

> **First confirm HPA behavior, then confirm scheduling results, and finally check node capacity.**  

---

## Step 17: What Relationship Does This Have With Cluster Autoscaler  

This is also a particularly important point.  

### HPA Handles Pod Scaling  
It solves:  

> **Whether to have more replicas**  

---

### Cluster Autoscaler Handles Node Scaling  
It solves:  

> **If existing nodes can't accommodate, whether to add more nodes**  

### So Typical Relationship Is As Follows  

### 1. HPA Detects High Load → Scale Pods  
### 2. New Pods Are Created  
### 3. New Pods Are Pending Because Nodes Can't Accommodate  
### 4. If Cluster Autoscaler Is Enabled  
### 5. CA May Continue to Scale Nodes  
### 6. New Nodes Appear, and Pending Pods Are Finally Scheduled  

### Operations Understanding Focus  

If there is no Cluster Autoscaler or CA can't scale nodes, the result will stop at:  

- HPA Scaled  
- Pods Pending  
- Limited Business Benefits  

---

## Step 18: Why This Article Is Closer to Real Production Than "HPA Basics"  

Because in production, the real difficulty is not:  

- Whether HPA YAML can be written  

But:  

- Whether the new replicas from HPA can actually become business capabilities  

In real environments, you're more likely to encounter:  

- HPA Took Effect  
- New Pods Start Slowly  
- Or Pending  
- Or Ready Slowly  
- Or Nodes Already Can't Accommodate  

### Operations Understanding Focus  

Therefore, the focus of this article is not "repeating HPA definitions," but:  

> **To help you understand the integrity of the elasticity chain.**  

---

## Step 19: Common Misunderstandings in This Article  

### 1. Assume HPA Scaling = Business Definitely Strengthened  
Not necessarily.  
Scaling is just the first step; new replicas must be truly Running/Ready to be meaningful.  

### 2. Assume Replica Count Increased = Scaling Succeeded  
Incomplete.  
You also need to check if new Pods were actually scheduled successfully.  

### 3. Check Business Logs First When Seeing Pending  
Often containers haven't started yet; check scheduling first.  

### 4. Assume HPA Can Solve Node Resource Insufficiency  
No.  
HPA doesn't scale nodes; it only scales Pods.  

### 5. Assume This Is HPA Itself Broken  
Often HPA is fine, the issue is:  

- Requests Too Large  
- Nodes Too Small  
- Cluster Capacity Insufficient  

---

## Step 20: Most Important Understandings in This Article  

### 1. HPA Only Decides Whether to Increase Replica Count  
Does not guarantee these replicas will be realized.

### 2. Whether a new Pod can actually work depends on the scheduler and node capacity
This is the most core practical limitation.

### 3. If node resources are insufficient, HPA-expanded new Pods are likely to be Pending
This is a very typical production phenomenon.

### 4. Troubleshooting such issues requires checking
- HPA
- Deployment
- Pod
- describe Pod
- Node capacity

### 5. Essentially, such issues result from
- HPA
- requests
- Scheduling
- Node capacity

The convergence of several main lines.

---

## Twenty-one, What Level Should You Master This Article

At this stage, it's recommended to reach the following level:

### 1. Be able to explain why HPA-expanded Pods may still fail to start
### 2. Be able to design a minimal experiment to observe this phenomenon
### 3. Be able to use `kubectl get hpa`, `kubectl get pod`, `kubectl describe pod` for basic observation
### 4. Be able to understand the relationship between Pending and HPA expansion
### 5. Be able to understand the responsibilities boundary between HPA and Cluster Autoscaler
### 6. Be able to connect requests, scheduling, node capacity, and HPA for understanding

---

## Twenty-two, Common Interview Extensions

Common questions in this area include:

- Why might Pods still fail to start after HPA expansion
- What's the relationship between HPA and scheduler
- Why might HPA-expanded new Pods be Pending
- What's the difference between HPA and Cluster Autoscaler
- Why might HPA take effect but business not improve noticeably
- What should be checked first when troubleshooting "HPA expanded but no effect"
- Why do requests affect the outcome of HPA expansion

---

## Twenty-three, Stage Summary

"Why HPA-expanded Pods may still fail to start" is a very critical lesson in Kubernetes elasticity system.

The most important thing about this article isn't memorizing a conclusion, but establishing these core understandings:

- HPA only handles Pod expansion, not node expansion
- Newly expanded Pods still need to go through scheduling and resource matching
- Requests directly affect whether new Pods can be placed on nodes
- When node capacity is insufficient, HPA expansion may only show as "replica count increased" on the surface
- True effective elasticity isn't just HPA taking effect, but making new replicas truly become Running/Ready business capabilities

As long as these understandings are established, subsequent topics will have clearer thinking:

- Cluster Autoscaler
- HPA stability optimization
- Resource capacity planning
- Peak elasticity governance

---

## Twenty-four, Keyword Memorization

- HPA: Horizontal Pod Autoscaling
- Pending: Pod has been created but not successfully scheduled or not completed prerequisite stages
- requests: Resource requests, important basis for scheduling
- scheduler: Scheduler, responsible for finding nodes for Pods
- Insufficient cpu: CPU resource insufficiency
- Insufficient memory: Memory resource insufficiency
- Cluster Autoscaler: Node auto-scaling
- Elasticity chain: Metric → HPA → Replica change → Scheduling → Startup → Ready

---

## Twenty-five, Operation Extension Understanding

From an operations perspective, this article truly aims to establish a "complete chain of thought".

Many people learning HPA only learn:

- Metric is high
- Replica count increases

But in real production, whether the business truly benefits depends on whether the entire chain is connected:

- Whether metrics are correctly collected
- Whether HPA makes expansion decisions
- Whether Deployment actually increases replicas
- Whether new Pods are scheduled successfully
- Whether new Pods start promptly
- Whether new Pods pass readiness checks and receive traffic

If any link in this chain breaks, the final result will be:

> **It looks like expansion happened, but the business capability hasn't truly improved.**

This is why this content actually ties together previously learned concepts:

- requests
- Pending
- QoS
- Node capacity
- HPA

into a complete practical closed-loop.

---

## References
- Kubernetes Horizontal Pod Autoscaling
- Kubernetes Resource Management for Pods and Containers
- metrics-server official documentation
- Cluster Autoscaler official documentation
- kubectl describe pod / describe node common output troubleshooting

---

## Next Day Recommendation
Next article suggestion: organize

[[08-VPA Foundation - Vertical Scaling Installation Prerequisites and Basic Practices]]