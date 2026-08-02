# 06-HPA Basics: Introduction to Pod Automatic Scaling

## Document Description
- Document Focus: Basic practices of horizontal automatic scaling in Kubernetes
- Target Audience: Those who have understood concepts such as requests/limits, Pod Pending status, QoS, and eviction, ready to learn about metric-based Pod auto-scaling
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/08-Resource Management and Auto Scaling/06-HPA Basics: Introduction to Pod Automatic Scaling`

## Tags
#Kubernetes #HPA #HorizontalPodAutoscaler #Automatic Scaling #Auto Scaling #CPU Utilization #metrics-server #requests #Resource Management #Application Deployment #Cloud-Native #Ops #Interview Notes

---

## I. Why Learn HPA Now

In the previous sections, we have established the following key understandings:

- `requests` affect Pod scheduling
- Pods may enter the `Pending` state when resources are insufficient
- `QoS` determines the level of resource guarantees for Pods
- Nodes may experience eviction when resources are scarce

With these concepts, we can now understand:

- How Pods are scheduled
- Why node resources might become limited
- How resource configuration impacts business stability

However, in real production environments, there is another practical issue:

- Higher traffic during the day and lower traffic at night
- High CPU usage for certain interfaces during peak periods
- A situation where 2 Pods were previously sufficient but suddenly are not enough
- The time-consuming and unstable process of manually adjusting the number of replicas

This is where Kubernetes provides a mechanism:

> **Automatically adjust the number of workload replicas based on metric changes.**

This mechanism is known as:

> **HPA (Horizontal Pod Autoscaler)**

The significance of learning HPA lies in:

- Understanding why the number of replicas can change automatically
- Comprehending the relationship between HPA, Deployment, requests, and metrics
- Learning to observe scaling phenomena rather than merely memorizing concepts
- Laying a foundation for understanding more advanced topics like "why HPA might not succeed in scaling Pods"

---

## II. Understanding the Phenomenon: What Does HPA Look Like?

When first encountering HPA, it is best to start by understanding its practical effects rather than the technical definitions.

A typical scenario is as follows:

### 1. You originally have a Deployment
For example:

- `replicas: 2`

---

### 2. Business load increases
For example:

- Continuous increase in CPU usage
- A single Pod experiencing significant stress

---

### 3. HPA starts working
It will detect that:

- Current metrics exceed the target values
- The existing number of replicas may be insufficient

Then, it automatically increases the number of replicas.

---

### 4. You check the Deployment/Pod again
You will find that:

- The number of replicas has increased
- New Pods have been created

### Key Points for Ops Professionals
The most intuitive aspect of HPA is not its complexity but rather:

> **The number of replicas is no longer fixed manually but dynamically adjusted based on load changes.**

---

## III. What Exactly Is HPA?

HPA can be simply understood as:

> **A mechanism that automatically increases or decreases the number of Pod replicas based on changes in metrics such as CPU or memory usage.**

The term "horizontal" here refers to:

- **Dynamically increasing or decreasing the number of Pods**
- Rather than allocating more resources to a single Pod

Therefore, HPA addresses the issue of:

> **Whether there are enough replicas**

rather than:

> **Whether individual Pods have sufficient resources**

---

## IV. Why Is It Called "Horizontal" Scaling?

In resource management, two main directions are commonly encountered:

### 1. Vertical Scaling
Allocating more resources to a single Pod/container, for example:

- Increasing CPU from `500m` to `1`
- Increasing memory from `512Mi` to `2Gi`

This is more like:

> **Making a single instance more powerful**

---

### 2. Horizontal Scaling
Rather than enhancing a single Pod, it involves:

- Changing the number of Pods from 2 to 4 or back
- Distributing traffic among multiple Pods

HPA falls into the second category.

---

## V. What Problems Does HPA Solve?

HPA is most suitable for solving the following issues:

### 1. Stateless applications with fluctuating traffic
For example:

- Nginx
- API services
- Java Web applications
- Go/Python interface services

---

### 2. Services where CPU or request load varies significantly with traffic
For example:

- High activity during the day and low activity at night
- Sudden spikes in traffic
- Constant high CPU usage during peak periods

---

### 3. Situations where you want the number of replicas to change automatically without manual intervention
For example:

- Automatically scaling up during busy timesmemory: "256Mi"

### Application Deployment

    kubectl apply -f nginx-deploy.yaml

### First, confirm that the Deployment is working properly

    kubectl get deploy
    kubectl get pod -o wide

### Key Points for Operations and Maintenance

It's important to note here:

> **CPU requests have already been specified**

Because the subsequent CPU-based HPA settings will be based on these requests.

---

## Section Fourteen: Basic HPA Example YAML

Here is a basic example of HPA configuration in YAML format:

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

### Applying HPA Configuration

    kubectl apply -f nginx-hpa.yaml

---

## Section Fifteen: What This HPA YAML Does

### 1. The target object is the `nginx-web` Deployment

The following line:

    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: nginx-web

indicates that the HPA is managing:

- The `Deployment/nginx-web`

It does not directly control individual Pods but manages the number of replicas of this Deployment.

---

### 2. `minReplicas: 2`
This means:

> **A minimum of 2 replicas must always be maintained**

Even when the workload is low, the number of replicas will not be reduced below 2.

---

### 3. `maxReplicas: 5`
This means:

> **The maximum number of replicas can be increased to 5**

No matter how high the load becomes, the number of replicas will not increase indefinitely.

---

### 4. `averageUtilization: 70`
This sets the target average CPU utilization at 70%.

You can simply understand this as:

- If the current average CPU utilization consistently exceeds 70%,
- HPA will tend to increase the number of replicas.

---

## Section Sixteen: How to Monitor the Current Status of HPA

The most commonly used command for monitoring HPA is:

    kubectl get hpa

For example, you might see output like this:

    NAME           REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
    nginx-web-hpa  Deployment/nginx-web   15%/70%   2         5         2          1m

### Key Columns to Check

#### `TARGETS`
For example, if the value is `15%/70%`, it means:

- The current average CPU utilization is approximately 15%,
- The target value is 70%.

#### `REPLICAS`
This column shows the current number of replicas.

### Key Points for Operations and Maintenance

This is the direct way to check whether HPA is functioning correctly.

---

## Section Seventeen: How to Generate CPU Pressure on Pods

The key to conducting HPA experiments is to ensure that the relevant metrics actually increase.

Some common approaches include:

### 1. Running tests inside containers
If the necessary tools are available in the container image, simple stress tests can be performed.

### 2. Running additional load generators
For example, using specialized load testing images or services to continuously request resources from the application.

### 3. Conducting external load testing
For instance, continuously sending requests to the Nginx service to increase its CPU usage.

### The focus at this stage is not on choosing a specific tool but on achieving the experiment objective:

> **To ensure that the average CPU utilization of the Deployment's Pods consistently exceeds the HPA target value.**

---

## Section Eighteen: Which Monitoring Commands Should Be Run During Experiments

It is recommended to run the following commands simultaneously during experiments:

### 1. Monitor HPA
    kubectl get hpa -w

Check:

- Whether the `TARGETS` value is increasing,
- Whether the number of replicas is changing.

---

### 2. Monitor Pods
    kubectl get pod -w

Check:

- Whether new Pods are being created,
- Whether the total number of Pods is increasing.

---

### 3. Monitor Deployment
    kubectl get deploy -w

Check:

- The status of `READY` and `AVAILABLE` pods,
- Any changes in the number of replicas.

---

### 4. Monitor resource usage
    kubectl top pod

Check:

- Whether the CPU usage of Pods has actually increased.

### Key Points for Operations and Maintenance

When conducting HPA experiments, do not focus on just one set of monitoring data. At### 2. Cluster Autoscaler is Responsible for Adding Nodes
The question it asks is:

> **If there are no more available nodes, should we add some more?**

### Typical Relationships

The following chain of events may occur:

### 1. HPA Detects High CPU Usage and Decides to Expand Pods
### 2. New Pods Are Created
### 3. However, Due to Insufficient Node Resources, the New Pods Remain Pending
### 4. At This Point, Cluster Autoscaler Is Needed to Add Nodes
### 5. Only After Nodes Are Added Can the New Pods Be Actually Scheduled

### Key Points for Ops Understanding

Therefore, HPA only addresses:

- “Whether we should add more Pods”

It does not directly handle:

- “Where these Pods will be placed”

---

## Twenty-seven. Several Premises That Are Most Likely Overlooked Before HPA Takes Effect

### 1. The Cluster Lacks the Ability to Collect Metrics
For example, if `metrics-server` is not correctly installed or running.

### 2. The Deployment Does Not Specify Reasonable Requests
Especially for CPU-based HPAs, this directly affects the accuracy of the judgment.

### 3. The Business Is Not Suitable for Horizontal Scaling
Even increasing the number of replicas may not truly solve the problem.

### 4. The Cluster Already Has Insufficient Node Resources
Although HPA adds more replicas, new Pods will still remain Pending.

### 5. New Pods Take Too Long to Start
Theoretically, scaling has occurred, but during peak times, the new replicas may not be ready in time.

---

## Twenty-eight. Several Common Misunderstandings About This Topic

### 1. Thinking That HPA “Automatically Improves Performance”
In fact, it:

> **Automatically Increases or Decreases the Number of Replicas**

### 2. Assuming That HPA Ensures Stability Once Configured
It is just part of an elastic system and is not a universal solution.

### 3. Believing That HPA Has No Relationship with Requests
In reality, CPU-based HPAs are closely related to requests.

### 4. Thinking That HPA Directly Manipulates Pod Processes
Essentially, it affects Deployments by changing the number of replicas.

### 5. Assuming That HPA Can Solve Insufficient Node Resources
It is not responsible for adding nodes; this requires Cluster Autoscaler or manual intervention.

---

## Twenty-nine. Several Key Understandings About This Topic

### 1. HPA Achieves Horizontal Automatic Scaling
The core mechanism is changing the number of replicas, not adjusting individual Pod resources.

### 2. HPA Is Most Suitable for Stateless Services That Can Be Horizontally Scaled
This type of service benefits most from HPA.

### 3. HPA Depends on a Metric System
Basic scenarios typically rely on `metrics-server`.

### 4. CPU-based HPAs Are Closely Related to Requests
Without appropriate requests, HPA will struggle to function effectively.

### 5. HPA Only Determines “Whether We Should Add More Pods”
It does not guarantee that these Pods will necessarily be effective.

### 6. HPA and Deployment Work Together
HPA dynamically adjusts the number of replicas, while Deployment is responsible for managing the Pods.

### 7. HPA Is Not the Same as Cluster Autoscaler
One focuses on Pod scaling, while the other focuses on node scaling.

---

## Thirty. What Level of Understanding Is Recommended at This Stage

For now, it is suggested to achieve the following:

### 1. Be able to clearly explain what HPA is.
### 2. Understand the difference between “horizontal scaling” and “vertical scaling.”
### 3. Be able to read a basic HPA YAML configuration.
### 4. Conduct a basic experiment to observe HPA in action.
### 5. Use `kubectl get hpa` and `kubectl top pod` to observe scaling phenomena.
### 6. Understand why HPA relies on metrics and requests.
### 7. Recognize that increasing Pod numbers with HPA may not immediately yield results.

---

## Thirty-one. Common Follow-up Questions in Interviews

Common questions include:

- What is HPA?
- How does HPA differ from VPA?
- On what indicators does HPA operate?
- Why does HPA usually rely on metrics-server?
- What is the relationship between HPA and requests?
- Why might Pods expanded by HPA still remain Pending?
- What is the relationship between HPA and Deployment?
- How does HPA differ from Cluster Autoscaler?

---

## Thirty-two. Summary of This Section

HPA is one of the core capabilities in Kubernetes for resource management and automatic scaling.

The most important thing to grasp here is not memorizing all the details of the autoscaling API, but establishing the following key understandings:

- The essence of HPA is to automatically adjust the number of Pod replicas.
- It is best suited for