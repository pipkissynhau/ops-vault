# 06-Horizontal Pod Autoscaler (HPA) Basics: Getting Started with Pod Auto Scaling

## Documentation Overview
- Documentation Focus: Kubernetes Horizontal Auto Scaling Basics Practice
- Applicable Stage: After understanding requests/limits, Pod Pending, QoS, and Eviction, entering the basics of Pod auto scaling based on metrics
- Recommended Path: `04-Kubernetes/07-Apply deployment/08-Resource management and flexibility/06-HPA Basis:Pod Automatically amplification entry`

## Tags
#Kubernetes #HPA #HorizontalPodAutoscaler #AutomaticallyAmplified #FlexibleStretch #CpuUtilization #metrics-server #requests #ResourceManagement #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## I. Why Learn HPA Now

The previous series has established these key insights:

- `requests` affects Pod scheduling
- Pods may `Pending` when resources are insufficient
- `QoS` affects Pod resource guarantee level
- `Eviction` may occur when node resources are strained

At this point, you can understand:

- How Pods are scheduled
- Why nodes become strained
- Why resource configuration affects business stability

But in real production environments, there's a very practical issue:

- High traffic during the day, low traffic at night
- A specific API endpoint's CPU spikes during peak times
- Two Pods could handle it, but suddenly they can't
- Manually adjusting replica counts is slow and unstable

At this point, Kubernetes provides a mechanism:

> **Adjust workload replica counts automatically based on metric changes.**

This is:

> **HPA (Horizontal Pod Autoscaler, Horizontal Auto Scaling)**

The significance of learning HPA lies in:

- Understanding why replica counts can change automatically
- Understanding the relationship between HPA, Deployment, requests, and metrics
- Learning to observe scaling phenomena rather than just memorizing concepts
- Laying the foundation for future learning about "why HPA might fail to scale Pods even after scaling"

---

## II. First, Understand the Phenomenon: What HPA Looks Like

When first encountering HPA, it's best to start with "phenomenon" rather than definitions.

A typical scenario is:

### 1. You originally have a Deployment
For example:

- `replicas: 2`

---

### 2. Business pressure increases
For example:

- CPU continuously rises
- Single Pod pressure is clearly too high

---

### 3. HPA starts working
It will detect:

- Current metrics exceed target values
- Existing replicas may be insufficient

Then automatically increase replica counts.

---

### 4. You check Deployment/Pod again
You'll notice:

- Replica counts have increased
- New Pods have been created

### Operations Understanding Focus

The most direct phenomenon of HPA isn't about "complex principles," but:

> **Replica counts are no longer fixed manually, but dynamically adjusted based on load changes.**

---

## III. What Exactly Is HPA

HPA can be simply understood as:

> **A mechanism that automatically increases or decreases Pod replica counts based on changes in CPU, memory, or other metrics.**

The "horizontal" here refers to:

- **Increasing or decreasing Pod counts horizontally**
- Not allocating more CPU/memory to individual Pods

So HPA solves the problem of:

> **Whether replica counts are sufficient**

Not:

> **Whether individual Pod resources are sufficient**

---

## IV. Why It's Called "Horizontal" Scaling

In resource management, you commonly encounter two directions:

### 1. Vertical Scaling
Allocate more resources to a single Pod/container, for example:

- CPU from `500m` to `1`
- Memory from `512Mi` to `2Gi`

This is more like:

> **Making a single instance stronger**

---

### 2. Horizontal Scaling
Instead of enhancing a single Pod, you:

- Start with 2 Pods
- Scale to 4 Pods
- Or scale back to 2 Pods

This is more like:

> **Having more instances share the traffic load together**

HPA does the second type.

---

## V. What Problems Does HPA Mainly Solve

HPA is best suited for solving:

### 1. Stateless applications with fluctuating traffic
For example:

- Nginx
- API services
- Java Web applications
- Go/Python interface services

---

### 2. Businesses where CPU or request pressure changes significantly with traffic
For example:

- Busy during the day, idle at night
- Sudden traffic spikes
- High CPU usage during peak times

---

### 3. Wanting replica counts to change automatically rather than manually maintaining them
For example:

- Automatically scale up during busy times
- Automatically scale down during idle times
- Maintain within a reasonable range

### Operations Understanding Focus

HPA's value isn't "letting Pods increase infinitely," but:

> **Letting replica counts adjust automatically within a set range as load changes.**

---

## VI. What Metrics Does HPA Base Its Decisions On

The most basic HPA typically uses these metrics:

### 1. CPU Utilization
This is the most common and suitable for beginners.

For example:

- Target CPU utilization at 70%
- Current value consistently exceeds the target
- HPA will tend to increase replica counts

---

### 2. Memory Utilization
You can also base decisions on memory, but understanding and tuning are usually more sensitive than CPU.

---

### 3. Custom Metrics/External Metrics
For example:

- QPS
- Queue length
- Request count
- Business metrics from external monitoring systems

This section won't expand on these yet. First, clarify the basics of CPU-based HPA.

---

## VII. What Is HPA's Operational Logic

You can first establish a basic understanding chain:

### 1. HPA continuously observes a workload
For example, a Deployment.

### 2. It reads target metrics
For example, average Pod CPU utilization.

### 3. It compares current values with target values

#### If current value is higher than target
Indicates replicas may be insufficient, consider scaling up.

#### If current value is lower than target
Indicates replicas may be excessive, consider scaling down.

### 4. It adjusts replica counts
Essentially, it modifies the `replicas` of the target workload.

### Operations Understanding Focus

HPA doesn't directly operate on container processes, but:

> **Adjusts replica counts dynamically to bring overall load back to a more reasonable range.**

---

## VIII. On Which Objects Does HPA Act

HPA generally doesn't act directly on "raw Pods," but on:

- Deployment
- StatefulSet
- ReplicaSet
- And workloads that support the scale subresource

Currently, the most common and suitable for beginners is:

> **Deployment + HPA**

Because it naturally suits auto scaling for stateless applications.

---

## IX. What Prerequisites Does HPA Depend On

This is a critical insight.

Many first-time HPA configurations find that YAML is written but nothing happens.  
The most common reason is:

> **HPA can't access metrics.**

### The most basic prerequisites usually include

### 1. Cluster has a metric source
Most commonly:

- `metrics-server`

---

### 2. Target workload supports scaling
For example, Deployment.

### 3. Resource Configuration Should Be Reasonably Set
Especially for CPU-type HPA, it is strongly related to `requests`.

### Operations Understanding Focus

HPA is not "just create an object", it at least depends on:

- Metric source
- Scalable target object
- Reasonable resource declarations

---

## 10. Why HPA Has a Direct Relationship with requests

This is the easiest point to overlook in HPA, yet it is very critical.

### Let's Start with the Conclusion

If HPA works based on **CPU utilization percentage**, then:

> **Its judgment is usually relative to requests.**

In other words, CPU utilization is not an absolute value, but typically references:

- Current actual CPU usage of the Pod
- CPU requests declared by the Pod

### Why This Is Important

If you don't write CPU requests or write them very unreasonably, then:

- HPA's judgment basis will become distorted
- Scaling effects will be unstable
- It may even fail to reach the expected outcome

### Operations Understanding Focus

Therefore, HPA is not an isolated function; it is strongly associated with:

- requests
- Metric system
- Deployment

---

## 11. Minimum Experimental Objective

This article doesn't aim for complexity first; let's set the most basic experimental objective first:

> **Make a Deployment automatically increase replica count from low to high when CPU pressure rises.**

You need to observe the chain of this process, not the "code logic":

### 1. Run the Deployment first
### 2. Bind HPA to the Deployment
### 3. Create CPU pressure for the Pod
### 4. HPA detects high CPU usage
### 5. HPA increases replica count
### 6. New Pod is created

---

## 12. Preparations Before Experiment

### 1. Confirm metrics-server is available

First execute:

    kubectl top node
    kubectl top pod -A

If you can already see metrics normally, it indicates the basic metrics capability is likely good.

If you get errors like:

- metrics API not available

It suggests the metric chain HPA depends on may not be ready yet.

---

### 2. Prepare a Deployment
Here we use a simple Nginx example as the carrier.

---

## 13. Experimental Deployment Example

Below is a basic Deployment example:

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
                  cpu: "100m"
                  memory: "128Mi"
                limits:
                  cpu: "500m"
                  memory: "256Mi"

### Apply Deployment

    kubectl apply -f nginx-deploy.yaml

### Confirm Deployment is Normal First

    kubectl get deploy
    kubectl get pod -o wide

### Operations Understanding Focus

Here you must pay attention:

> **CPU requests have already been written**

Because the CPU-type HPA will base its judgment on this.

---

## 14. HPA Example YAML

Below is the simplest HPA example:

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
      maxReplicas: 5
      metrics:
        - type: Resource
          resource:
            name: cpu
            target:
              type: Utilization
              averageUtilization: 70

### Apply HPA

    kubectl apply -f nginx-hpa.yaml

---

## 15. What Does This HPA YAML Do

### 1. Target Object is `nginx-web` Deployment

This segment:

    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: nginx-web

Indicates HPA manages:

- `Deployment/nginx-web`

It doesn't directly manage a specific Pod, but manages the replica count of this Deployment.

---

### 2. `minReplicas: 2`
Indicates:

> **Minimum 2 replicas will be retained**

Even if the business is very idle, it won't scale down below 2.

---

### 3. `maxReplicas: 5`
Indicates:

> **Maximum scale up to 5 replicas**

Even if the load is extremely high, it won't scale infinitely.

---

### 4. `averageUtilization: 70`
Indicates the target average CPU utilization is 70%.

You can first simply understand it as:

- If the current average CPU utilization remains above 70%
- HPA will tend to scale up

---

## 16. How to Observe HPA Current Status

The most commonly used command is:

    kubectl get hpa

For example, you might see:

NAME           REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-web-hpa  Deployment/nginx-web   15%/70%   2         5         2          1m

### Which Columns to Focus On

#### `TARGETS`
For example:

- `15%/70%`

Can be understood as:

- Current average CPU utilization is approximately 15%
- Target is 70%

#### `REPLICAS`
Represents the current number of replicas.

### Key Operational Insights

This is the entry point to directly observe whether HPA is working.

---

## Seventeen. How to Create CPU Pressure for Pods

For HPA experiments, the key is to make the metrics actually rise.

A common approach is:

### 1. Enter the Container
If the image has sufficient tools, you can perform simple stress testing.

### 2. Or run an additional stress container
For example, use a dedicated stress-testing image or a stress-testing image to continuously request the service.

### 3. Or perform continuous external stress testing
For example, continuously request the Nginx service to keep Pod CPU utilization rising.

### The core of this phase is not to choose a specific tool, but to achieve the experimental goal:

> **Keep the target Deployment's Pod average CPU utilization consistently above the HPA target value.**

---

## Eighteen. Which Observation Commands to Run Simultaneously During Experiments

It is recommended to run these groups of commands simultaneously.

### 1. Monitor HPA

    kubectl get hpa -w

Observe:

- Whether TARGETS increase
- Whether REPLICAS change

---

### 2. Monitor Pods

    kubectl get pod -w

Observe:

- Whether new Pods are created
- Whether the number of Pods increases

---

### 3. Monitor Deployment

    kubectl get deploy -w

Observe:

- READY
- AVAILABLE
- Changes in replica count

---

### 4. Monitor Resource Metrics

    kubectl top pod

Observe:

- Whether Pod CPU utilization actually rises

### Key Operational Insights

When performing HPA experiments, do not focus on just one command.  
At least observe simultaneously:

- Metrics
- HPA
- Pod count

This allows you to connect the "phenomenon chain."

---

## Nineteen. What Phenomena Should You Observe

If the experiment is successful, typical phenomena are usually:

### Step 1
Initial state:

- Deployment has 2 replicas
- HPA current replica count is 2
- `TARGETS` is relatively low

---

### Step 2
After continuous pressure is applied:

- `kubectl top pod` shows CPU rising
- `kubectl get hpa`'s `TARGETS` begins to approach or exceed the target value

---

### Step 3
After a period of time:

- HPA starts increasing replica count from 2 to 3
- Or continues adjusting toward 4, 5

---

### Step 4
`kubectl get pod -w` will show:

- New Pods are created
- New Pods enter Running / Ready

### Key Operational Insights

In HPA experiments, what you truly need to confirm is not "whether YAML is written correctly," but:

> **Metrics rise → HPA detects → replica count increases → new Pods appear**

Whether this chain of events actually occurs.

---

## Twenty. What Happens When HPA Scales Pods

Assume currently:

- Deployment has 2 replicas
- CPU utilization is consistently above the target value
- HPA determines scaling is needed

The actual events that occur can be simply understood as:

### 1. HPA detects metrics are too high
### 2. HPA calculates the need for more replicas
### 3. HPA updates the target Deployment's replica count
### 4. Deployment creates new Pods
### 5. Scheduler assigns nodes to these new Pods
### 6. New Pods start and join traffic handling

### Key Operational Insights

HPA is not a one-step solution for all problems; it merely:

> **Makes the decision to "increase the number of replicas."**

Further steps still need to go through:

- Scheduling
- Resource matching
- Pod startup
- Readiness checks

These stages.

---

## Twenty-one. Why HPA Cannot Be Understood Separately from Deployment

Because HPA itself does not "create replica mechanisms"; it relies on the upper-level controller that actually manages Pod counts.

For example, Deployment itself is responsible for:

- Defining Pod templates
- Maintaining replica counts
- Rolling updates
- Rebuilding abnormal Pods

While HPA is responsible for:

- Dynamically modifying `replicas` based on metrics

So the relationship between the two can be initially understood as:

### Deployment
Responsible for "how to manage these Pods"

### HPA
Responsible for "what the replica count should be"

---

## Twenty-two. Why HPA Expanding Pods Doesn't Necessarily Take Effect Immediately

This is one of the most commonly misunderstood aspects of HPA.

Many people think:

- Configured HPA
- High traffic automatically scales
- The business is definitely stable

But it's not necessarily the case.

Because HPA only solves one thing:

> **Whether more Pods should be added**

But it does not guarantee:

- Nodes have enough resources
- New Pods can be scheduled successfully
- New Pods can start quickly
- New Pods can be Ready immediately
- The application itself supports horizontal scaling

### Key Operational Insights

Therefore, HPA is just one link in the elasticity chain, not a universal switch.

---

## Twenty-three. Why Some Businesses Are Not Suitable for HPA from the Start

HPA is most suitable for:

- Stateless
- Can scale horizontally
- No strong state binding between replicas
- Clear relationship between metric changes and load

But some businesses are not suitable for direct HPA, for example:

### 1. Stateful applications
Such as services with strong local state dependencies, single-point writes, or complex state synchronization.

### 2. Applications with very slow startup
Although HPA can scale Pods, new Pods take a long time to become Ready, which reduces the actual benefit.

### 3. Applications where metrics don't match real business pressure
Such as low CPU utilization but poor connection counts, queue lengths, or response times.

### Key Operational Insights

Whether HPA is suitable depends not only on "whether it can be configured" but also on:

> **Whether this business can truly be solved by "adding more replicas."**

---

## Twenty-four. What's the Relationship Between HPA and Manually Changing Replicas

### Without HPA
Typically maintained manually:

- `replicas: 2`
- `replicas: 4`

---

### With HPA
Replica counts are dynamically adjusted by HPA.

That is:

- Deployment remains the same Deployment
- But `replicas` is no longer a static value
- HPA will modify it based on metrics

### Key Operational Insights

After using HPA, establish this understanding:

> **Replica count is no longer a fixed number in the configuration file, but a dynamic result.**

---

## Twenty-five. Is HPA the Same as VPA?

No.

### HPA
Solves the question of:

> **Should the number of replicas increase or decrease**

Which means:

- Horizontal scaling

---

### VPA
Solves the question of:

> **Should the CPU/memory of a single Pod be increased or decreased**

Which means:

- Vertical resource adjustment

### Operations Understanding Focus

At this stage, it's sufficient to first understand HPA clearly. Do not mix HPA, VPA, and CA together from the beginning.

---

## Twenty-six, What is the Relationship Between HPA and Cluster Autoscaler

### 1. HPA is responsible for scaling Pods
It asks:

> **Should we add more replicas?**

---

### 2. Cluster Autoscaler is responsible for scaling nodes
It asks:

> **Do we need to add more nodes since the existing nodes are full?**

### Typical Relationship

The following chain of events may occur:

### 1. HPA detects high CPU usage and decides to scale Pods
### 2. New Pod is created
### 3. Resulting in insufficient node resources, new Pod is Pending
### 4. At this point, Cluster Autoscaler is needed to scale nodes
### 5. After nodes are expanded, new Pod can be scheduled

### Operations Understanding Focus

Therefore, HPA only solves:

- **Should we have more Pods**

It does not directly solve:

- **Whether there is space to place these Pods**

---

## Twenty-seven, Key Precautions Before HPA Takes Effect

### 1. The cluster lacks metrics capability
For example, metrics-server is not correctly installed or running `metrics-server`.

### 2. Deployment does not specify reasonable requests
Especially for CPU-based HPA, it directly affects the judgment baseline.

### 3. The business itself is unsuitable for horizontal scaling
Even if replica count increases, it may not truly solve the problem.

### 4. Cluster node resources are inherently insufficient
HPA may scale replicas, but new Pods will be Pending.

### 5. New Pod startup is too slow
Theoretically scaled, but during peak hours, new replicas may not be Ready.

---

## Twenty-eight, Common Misunderstandings in This Section

### 1. Believing HPA is "automatically becoming stronger"
Actually, it is:

> **Automatically increasing or decreasing**

### 2. Believing HPA is stable once configured
It is just part of the elasticity system, not a universal solution.

### 3. Believing HPA has no relation to requests
In fact, CPU-based HPA is strongly related to requests.

### 4. Believing HPA directly operates Pod processes
It essentially adjusts replica count to influence Deployment.

### 5. Believing HPA can solve node resource insufficiency
It does not handle node expansion; this depends on Cluster Autoscaler or manual scaling.

---

## Twenty-nine, Key Understandings in This Section

### 1. HPA is horizontal auto-scaling
Core is adjusting replica count, not changing single Pod resource size.

### 2. HPA is most suitable for stateless, horizontally scalable workloads
Such scenarios benefit most from HPA.

### 3. HPA relies on metric systems
Basic scenarios typically depend on `metrics-server`.

### 4. CPU-based HPA is strongly related to requests
Without reasonable requests, HPA struggles to work effectively.

### 5. HPA only decides **"should we have more Pods"**
Does not guarantee these Pods will function properly.

### 6. HPA and Deployment have a collaborative relationship
HPA dynamically adjusts replica count, Deployment manages Pods.

### 7. HPA is not equal to Cluster Autoscaler
One scales Pods, the other scales nodes.

---

## Thirty, What Level of Understanding Should You Reach to Learn This Section

At this stage, it's recommended to reach the following level:

### 1. Be able to explain what HPA is
### 2. Understand the difference between "horizontal scaling" and "vertical scaling"
### 3. Be able to read a basic HPA YAML
### 4. Be able to complete a basic HPA experiment observation
### 5. Be able to use `kubectl get hpa`, `kubectl top pod` to observe scaling phenomena
### 6. Understand why HPA depends on metrics and requests
### 7. Understand why HPA scaling may not take effect immediately

---

## Thirty-one, Common Interview Follow-up Questions

Common questions in interviews include:

- What is HPA?
- What's the difference between HPA and VPA?
- What metrics does HPA base its work on?
- Why does HPA typically rely on metrics-server?
- What's the relationship between HPA and requests?
- Why might HPA-scaled Pods still be Pending?
- What's the relationship between HPA and Deployment?
- What's the difference between HPA and Cluster Autoscaler?

---

## Thirty-two, Stage Summary

HPA is one of the core capabilities in Kubernetes resource management and elasticity scaling.

The most important part of this section is not memorizing all details of the autoscaling API, but establishing these core understandings:

- HPA's essence is automatically adjusting Pod replica count
- It's most suitable for stateless, horizontally scalable workloads
- It relies on metric systems, not just writing YAML will automatically work
- CPU-based HPA is strongly related to requests
- HPA isn't a universal elasticity solution; it only solves **"should we have more Pods"**
- Whether new Pods can truly function depends on scheduling, resources, startup speed, and the workload itself

As long as these understandings are established, further learning will be smooth:

- Why HPA-scaled Pods may not take effect immediately
- VPA
- Cluster Autoscaler
- Elasticity system optimization

---

## Thirty-three, Keyword Quick Notes

- HPA: Horizontal Pod Autoscaler, horizontal auto-scaling
- scaleTargetRef: Target object managed by HPA
- minReplicas: Minimum replica count
- maxReplicas: Maximum replica count
- averageUtilization: Target average resource utilization
- metrics-server: Basic source of resource metrics
- Horizontal scaling: Increase Pod count
- Vertical scaling: Increase single Pod resources
- Cluster Autoscaler: Node auto-scaling

---

## Thirty-four, Operations Perspective Understanding

From an operations perspective, HPA is a key step in Kubernetes transitioning from "static deployment" to "dynamic elasticity."

Without HPA, your understanding of replica count is typically:

- What's written in the configuration file
- What's always online

With HPA, replica count becomes a dynamic result:

- More when traffic is high
- Fewer when traffic is low
- The platform starts automatically participating in replica control based on metrics

This means your understanding of application deployment also needs to evolve:

- No longer just "can Pods run"
- Also consider "is the Pod count sufficient"
- No longer just "how to write Deployment"
- Also consider "is the workload suitable for auto-scaling"

So HPA seems to just add one object, but in reality, it connects:

- Workload
- Metric system
- Replica control
- Scheduling capability
- Node capacity

into a true elasticity chain.

## References
- Kubernetes Horizontal Pod Autoscaling
- Kubernetes Metrics API
- metrics-server official documentation
- HorizontalPodAutoscaler v2 API

---

## Next Day Suggestions
Next article suggestion to organize:

[[07-HPA Why Expanded Pods Might Still Fail to Start - Relationship Between Scheduling and Resource Capacity]]