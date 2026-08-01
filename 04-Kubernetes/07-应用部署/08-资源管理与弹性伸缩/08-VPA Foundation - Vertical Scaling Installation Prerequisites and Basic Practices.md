# 08-VPA Basics: Vertical Scaling, Installation Prerequisites, and Fundamental Practices

## Document Overview
- **Document Positioning**: Kubernetes resource management and elasticity scaling phase, introducing VPA (Vertical Pod Autoscaler, vertical auto-scaling) fundamentals
- **Applicable Stage**: After understanding requests/limits, Pending, QoS, Eviction, HPA basics, begin learning VPA installation prerequisites, core components, installation methods, verification methods, and minimal experiments
- **Recommended Path**: `04-Kubernetes/07-Apply deployment/08-Resource management and flexibility/08-VPA Foundation: vertical expansion, installation preconditions and basic practice.md`

## Tags
#Kubernetes #VPA #VerticalPodAutoscaler #metrics-server #CRD #CustomResourceDefinition #VerticalAmplification #ResourceManagement #FlexibleStretch #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## I. Why Learn VPA After HPA

Previously, we've completed several key understandings in this series:

- `requests / limits` affects scheduling and runtime stability
- Pods may be `Pending` due to resource shortages
- `QoS` affects resource guarantee levels during resource tension
- Node resource pressure may cause `Eviction` when high
- `HPA` can automatically increase or decrease replica counts based on metrics
- However, expanding Pods via HPA doesn't guarantee new Pods will successfully schedule and handle traffic

At this point, we can understand a key elasticity concept in Kubernetes:

> **Horizontal scaling means adding more Pods.**

But in real production environments, not all issues are suitable for solving by "adding more Pods."  
We often encounter another type of problem:

- This Pod replica count can't easily increase
- This component depends more on single-instance resource size
- Current requests for this business are clearly underestimated
- Pods can run, but resource configuration is long-term too small or too large
- Resource allocation is overly conservative, causing obvious waste
- Resource allocation is overly aggressive, leading to OOM, restarts, or unstable performance

At this point, we need:

- **Not more Pods**
- **More reasonable single-Pod resource specifications**

This leads to another elasticity direction:

> **Vertical scaling**

VPA is one of the most common vertical auto-scaling mechanisms in the Kubernetes ecosystem.

---

## II. What is "Vertical Scaling"

In Kubernetes, when talking about auto-scaling, HPA is often the first thing that comes to mind.  
Because HPA is very intuitive:

- Originally 2 Pods
- Under pressure, becomes 4 Pods
- Under low pressure, scales back

This is called:

> **Horizontal scaling**

Which means:

- Changing replica counts
- Not necessarily changing single-instance resource specifications

But there's another approach:

### Not increasing Pod count, but adjusting single-Pod resource specifications

For example:

- Original CPU request is `200m`
- Now changed to `500m`

Or:

- Original memory request is `256Mi`
- Now changed to `1Gi`

The focus of this approach isn't "more," but:

> **Larger or smaller**

This is:

> **Vertical scaling**

### You can simply distinguish them like this

#### Horizontal Scaling
- Adjust replica counts
- Representative solution: HPA

#### Vertical Scaling
- Adjust single-Pod resource specifications
- Representative solution: VPA

---

## III. What is VPA

The full name of VPA is:

> **Vertical Pod Autoscaler**

You can first understand it simply as:

> **A mechanism that automatically provides or applies more suitable CPU/Memory requests based on Pod resource usage.**

VPA's core solution is for these issues:

- How much resource should a single Pod have?
- Is current requests significantly underestimated?
- Is current requests significantly overestimated?
- Can we reduce manual long-term resource guessing costs through automatic recommendations or updates?

### Remember this core sentence for now

> **HPA mainly solves "whether replica counts are sufficient," while VPA mainly solves "whether single-Pod resource allocation is reasonable."**

---

## IV. Why Must We Clearly Explain VPA Installation Prerequisites When Learning VPA

This is extremely critical.

Many people first learn VPA and mistakenly think it's:

- Like Deployment, just write a YAML and use it directly
- Like HPA, inherently available in most Kubernetes clusters

This understanding is incomplete.

### Remember this conclusion first

> **VPA is not a capability natively available in most Kubernetes clusters.**

For VPA to work, it usually depends on two prerequisites:

### 1. Resource metric source
That is:

- **metrics-server**

### 2. VPA's own components and custom resources
That is:

- **CRD**
- **recommender**
- **updater**
- **admission-controller**

So learning VPA shouldn't only focus on `VerticalPodAutoscaler` itself. You must first understand:

- Whether the cluster has resource metric sources
- Whether the cluster has installed VPA components
- Whether the cluster recognizes `VerticalPodAutoscaler` as a resource

---

## V. Why Must We First Install metrics-server

This must be clearly explained separately.

`metrics-server` is one of the core components in Kubernetes resource metric chains. It is responsible for:

- Collecting resource usage data from kubelets on each node
- Exposing Pod/Node CPU and memory metrics via `metrics.k8s.io` API
- Supplying `kubectl top`
- Supplying HPA
- Supplying VPA

### You can first understand it like this

#### kubelet
Responsible for providing resource usage information on nodes

#### metrics-server
Responsible for aggregating this information and exposing it via API

#### kubectl top / HPA / VPA
Responsible for consuming these data

### So we must establish this sequential understanding

> **metrics-server must be in place before HPA/VPA can have standard resource metric sources.**

### Most practical judgment method at this stage

First execute:

    kubectl top nodes
    kubectl top pods -A

If you can't get data at all, don't rush toShit. VPA yet. First ensure metrics-server is installed and working properly.

---

## VI. Why Recommend "Clone First, Then Install"

This aligns perfectly with realTransport habits.

Not recommended to directly online `kubectl apply -f Some address` at first because although it's fast, there are several issues:

- Can't see exactly what resources are being installed
- Can't pre-check YAML
- When facing issues like image addresses, certificate parameters, ports, or address selection, it's inconvenient to modify directly
- Not suitable for internal network environments or scenarios where direct GitHub access is not possible
- Not conducive to subsequent cost-accumulation inventory lists and Git management

So the recommended approach at this stage is:

> **Clone the official repository first, review the inventory, modify as needed, and then install locally.**

This approach is more like actual production operations.

## VII. How to Generally Install metrics-server

### Step 1: Clone the Repository

    git clone https://github.com/kubernetes-sigs/metrics-server.git
    cd metrics-server

### Step 2: Check the Installation Manifests Directory

    ls
    ls manifests

Your current focus should be on:

- `manifests/`

This is typically the resource manifest provided by the metrics-server official.

### Step 3: Local Installation

    kubectl apply -f manifests/

### Step 4: Check if Pod is Running

    kubectl get pods -n kube-system | grep metrics-server

### Step 5: Verify APIService Registration

    kubectl get apiservice | grep metrics
    kubectl describe apiservice v1beta1.metrics.k8s.io

### Step 6: Validate Resource Metrics

    kubectl top nodes
    kubectl top pods -A

### Current Stage's Critical Validation Criteria

Don't just check if Pod `Running` is running, but ensure both of these are true:

#### 1. Metrics API is Available
That is, `v1beta1.metrics.k8s.io` is functioning normally

#### 2. `kubectl top` Can Retrieve Data
This indicates the resource metrics pipeline is basically connected

---

## VIII. Common Bottlenecks When Installing metrics-server on Self-Managed Clusters

This is a very common issue in actual operations.

Many self-managed clusters aren't "installation failed", but rather:

- Pod appears to be running
- But `kubectl top` has no data
- Or APIService is unavailable
- Or logs show kubelet TLS, address selection related errors

### Most Common Directions

#### 1. kubelet Certificate Validation Issues
This is the most common category.

#### 2. metrics-server Selects Wrong Address When Connecting to Nodes
For example, it selects an unreachable address instead of the node's internal network address.

#### 3. `10250` Between Control Plane and Node kubelet is Unreachable
Even if YAML is correct, metrics cannot be pulled.

---

## IX. Understanding Common Correction Parameters for metrics-server

If you're managing a self-hosted cluster, you often add some startup parameters to metrics-server in experimental environments.

The most common ones are:

    args:
      - --kubelet-preferred-address-types=InternalIP,Hostname
      - --kubelet-insecure-tls

### Understanding These Two Parameters

#### `--kubelet-preferred-address-types=InternalIP,Hostname`
Means:

- Prioritize using the node's `InternalIP`
- If that's not suitable, then consider `Hostname`

This parameter's purpose is:

> **To make metrics-server use the actual reachable node address to access kubelet.**

#### `--kubelet-insecure-tls`
Means:

- Skip strict TLS validation when accessing kubelet

This parameter is commonly used in:

- Experimental environments
- Self-hosted cluster troubleshooting
- Scenarios where kubelet serving cert configuration is not standardized

### But Must Establish Correct Understanding

> **It's more suitable for testing or troubleshooting, not production environment preferred solution.**

In production environments, the more ideal approach is still to configure kubelet certificate chain and access chain properly.

---

## X. After metrics-server is Installed, Next Step is Installing VPA

This order should not be reversed:

### Step 1
First confirm `kubectl top nodes` and `kubectl top pods -A` are normal

### Step 2
Then proceed to install VPA

Because VPA requires a resource metrics source, and metrics-server is one of the most common and standard sources.

---

## XI. Why VPA Isn't Default Available

This must be clearly understood.

Many learners at this stage will ask:

- I already have metrics-server
- Why can't I just create `VerticalPodAutoscaler` directly?

It's still not possible.

Because `VerticalPodAutoscaler` is not a regular core resource, but:

> **A custom resource introduced via CRD.**

That is:

### Before Installing VPA
The cluster typically doesn't recognize:

- `VerticalPodAutoscaler`

### After Installing VPA
The cluster will have the corresponding:

- CRD
- Controller components
- admission-controller
- recommender
- updater

So here must establish a complete understanding:

> **metrics-server solves the resource metrics source problem, while VPA components solve the vertical scaling capability itself.**

---

## XII. What Components Does VPA Typically Include

At this stage, you don't need to dive into source code, but must first understand the component structure.

### 1. CRD
Used to let the cluster recognize:

- `VerticalPodAutoscaler`

### 2. Recommender
Responsible for:

- Calculating more suitable CPU/Memory requests based on historical and current resource usage

You can think of it as:

> **Resource recommendation calculator**

### 3. Updater
Responsible for:

- Pushing existing Pods to be rebuilt or updated according to recommendations when needed

You can think of it as:

> **The execution component that makes recommendations take effect**

### 4. Admission Controller
Responsible for:

- Participating in resource recommendation injection or update process during Pod creation

You can think of it as:

> **The entry component for resource recommendations to enter the actual object creation process**

---

## XIII. How to Generally Install VPA

### Step 1: Clone the autoscaler Official Repository

    git clone https://github.com/kubernetes/autoscaler.git
    cd autoscaler/vertical-pod-autoscaler

### Step 2: Check Directory Structure

    ls
    ls hack
    ls examples

Your current focus should be on:

- `hack/`
- `examples/`

### Step 3: Check Installation Scripts

    ls hack

You'll typically see:

- `vpa-up.sh`
- `vpa-down.sh`

### Step 4: Execute Installation

    ./hack/vpa-up.sh

### Step 5: Check CRD

    kubectl get crd | grep -i verticalpodautoscaler

### Step 6: Check Components

    kubectl get pods -A | grep -i vpa
    kubectl get deployment -A | grep -i vpa

### You should at least see components similar to these
- recommender
- updater
- admission-controller

### The most important judgment criteria at this stage

Installing VPA successfully is not just about the script executing without errors, but:

#### 1. The cluster already has VPA-related CRD
#### 2. VPA-related Pod / Deployment is already running
#### 3. It can successfully create `VerticalPodAutoscaler` objects later

---

## Fourteen, Installing VPA and creating business VPA objects are two different things

This point must be remembered separately.

### First layer: Installing VPA capability
This step does:

- Install CRD
- Install recommender
- Install updater
- Install admission-controller

The result of this step is:

> **The cluster now has VPA capability.**

### Second layer: Creating a VPA object for a specific business workload
This step is:

- For a specific Deployment / StatefulSet
- Creating `VerticalPodAutoscaler` rules

The result of this step is:

> **A specific business workload is now being managed by VPA.**

Do not confuse these two steps into one sentence:

- "Just create a VPA is enough"

More accurately, it should be:

1. First install metrics-server  
2. Then install VPA components  
3. Then create the specific `VerticalPodAutoscaler` business object  

---

## Fifteen, Why the first experiment suggests using `updateMode: Off` first

Because VPA beginners often fear automatically modifying business resource configurations from the start.

The safest way during the learning phase is:

    updateMode: "Off"

Its meaning is:

- VPA continues to calculate recommendation values
- You can see the recommendation
- But it won't automatically modify resource configurations of running business pods

### Why this is suitable for beginners

Because what you currently need to understand first is:

- What a VPA object looks like
- How the recommendation appears
- What the recommended values are approximately
- Whether the cluster has actually run through this chain

Rather than letting it automatically modify pods from the start.

---

## Sixteen, How to do a minimal experiment

At this stage, it's recommended to do a minimal experiment, not complex, but focus on running through the chain.

Experiment goals:

- Prepare a simple Deployment
- Create a `VerticalPodAutoscaler` for it
- Use `updateMode: Off`
- Check if the recommendation appears

---

## Seventeen, Prepare a simple Deployment

You can first prepare a simple nginx Deployment.

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-web
      namespace: test
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx-web
      template:
        metadata:
          labels:
            app: nginx-web
        spec:
          containers:
            - name: nginx
              image: nginx:1.25
              ports:
                - containerPort: 80
              resources:
                requests:
                  cpu: 100m
                  memory: 128Mi
                limits:
                  cpu: 500m
                  memory: 256Mi

First create the namespace and Deployment:

    kubectl create namespace test
    kubectl apply -f nginx-web.yaml
    kubectl get pods -n test

---

## Eighteen, Create a minimal VPA object

Then create a minimal VPA example:

    apiVersion: autoscaling.k8s.io/v1
    kind: VerticalPodAutoscaler
    metadata:
      name: nginx-web-vpa
      namespace: test
    spec:
      targetRef:
        apiVersion: "apps/v1"
        kind: Deployment
        name: nginx-web
      updatePolicy:
        updateMode: "Off"

Apply this object:

    kubectl apply -f nginx-web-vpa.yaml
    kubectl get vpa -n test

---

## Nineteen, How to verify if the minimal VPA experiment was successful

### Step 1: First confirm the object was created successfully

    kubectl get vpa -n test

### Step 2: Check detailed information

    kubectl describe vpa nginx-web-vpa -n test

### What you should focus on

Most importantly, check:

- recommendation
- whether targetRef is correct
- whether CPU/Memory recommendations are being provided

### The most critical verification criteria at this stage

> **It's not whether it automatically changed resources, but whether it successfully generated a recommendation.**

If the recommendation can be seen, it indicates the following chain is likely working:

1. metrics-server is functioning normally  
2. VPA components are functioning normally  
3. VPA object was created successfully  
4. recommender is working  

---

## Twenty, What to check first if the recommendation never appears

This type of issue is very common and must be troubleshooted.

### 1. First check if metrics-server is functioning normally

    kubectl top nodes
    kubectl top pods -A
    kubectl get pods -n kube-system | grep metrics-server
    kubectl logs -n kube-system deploy/metrics-server

### 2. Then check if VPA components are functioning normally /think

kubectl get pods -A | grep -i vpa
kubectl logs -n kube-system deployment/vpa-recommender
kubectl logs -n kube-system deployment/vpa-updater
kubectl logs -n kube-system deployment/vpa-admission-controller

If your actual namespace is not `kube-system`, adjust according to the actual installation location.

### 3. Check if the cluster recognizes `VerticalPodAutoscaler`

    kubectl get crd | grep -i verticalpodautoscaler
    kubectl api-resources | grep -i verticalpodautoscaler

### 4. Check if targetRef is correctly specified

Often it's not the VPA that's broken, but:

- namespace is wrong
- Deployment name is wrong
- targetRef points to the wrong object

### 5. Give it some time

VPA recommendation does not appear immediately after you apply.  
Observe for a while, then describe to see the results.

---

## 21. Recommended learning order at this stage

To avoid getting confused at the beginning, follow this order.

### Step 1: First confirm metrics-server
Ensure the resource metrics chain is fully connected.

### Step 2: Then install VPA
Enable the cluster with `VerticalPodAutoscaler` capability.

### Step 3: First do a minimal experiment with `updateMode: Off`
Observe the recommendation first.

### Step 4: First check the recommendation
Understand what it recommends.

### Step 5: Then understand the automatic update mode
Only after the previous chain is clear, proceed to `Initial`, `Recreate`, etc.

---

## 22. How to answer "How to implement VPA" in an interview

Answer in this structure:

Before using VPA, we first confirm the cluster has a resource metrics source, typically install and verify metrics-server, ensuring `kubectl top` can return Pod and Node CPU/memory data.  
Then install the VPA components themselves, including CRD, recommender, updater, and admission-controller.  
After installation, we don't immediately let it automatically modify business resources, but first create a `updateMode: Off` `VerticalPodAutoscaler` object for a test workload, observe if the recommendation is reasonable, confirm the chain is normal, then decide whether to enter more automated update mode.

### Remember these keywords in this answer

- First metrics-server
- Then VPA components
- `VerticalPodAutoscaler` is a custom resource
- First `Off`
- First check recommendation
- Then consider automatic update

---

## 23. Key conclusions from this article

### 1. VPA is vertical autoscaling capability
It focuses on:

- Single Pod resource specifications
- Whether CPU/memory requests are reasonable

### 2. VPA is not out-of-the-box by default
It typically depends on:

- metrics-server
- CRD
- recommender
- updater
- admission-controller

### 3. Install metrics-server first, then VPA
This order should not be reversed.

### 4. At this stage, it's recommended to clone and install first
This makes it easier to review the manifest, modify parameters, adapt to intranet environments, and subsequentDeposition.

### 5. Installing VPA and creating business VPA objects are two different things
First install the capability, then create the rule objects.

### 6. Minimal experiment suggests using `updateMode: Off` first
Because it's best for observing recommendation, not automatically modifying business resources.

---

## 24. Stage Summary

The most important part of this article isn't to learn all VPA details at once, but to run through this chain:

1. Understand why VPA isn't just "write another YAML"
2. Understand why metrics-server should be installed first
3. Understand why it's recommended to clone the official repository and install locally first
4. Know how to install and verify metrics-server
5. Know how to install and verify VPA
6. Know how to create a minimal experiment with `updateMode: Off`
7. Know how to check recommendation via `kubectl describe vpa`

By now, you should have a relatively complete beginner's understanding:

> **VPA isn't just a conceptual object, but a complete chain from resource metrics source, to custom resources, to recommendation calculation, and finally to business practice.**

Its true value isn't just "automatically changing resources," but helping us gradually answer a very practical question:

> **How much resource should a single Pod have to be more reasonable?**

---

## Keyword Quick Notes

- VPA: Vertical Pod Autoscaler, vertical autoscaling
- metrics-server: Resource metrics source component
- `metrics.k8s.io`: Resource metrics API
- CRD: CustomResourceDefinition, custom resource definition
- `VerticalPodAutoscaler`: VPA's corresponding custom resource object
- recommender: Calculates resource recommendations
- updater: Drives resource recommendations to take effect
- admission-controller: Participates in the recommendation application process
- `updateMode: Off`: Only recommends, no automatic update
- recommendation: Resource recommendations provided by VPA

---

## Operational Extended Understanding

From an operations perspective, VPA's value isn't just "letting resources change automatically," but transitioning resource configuration from long-term reliance on manual experience to a more observable, verifiable, and optimizable governance approach.

Many teams truly get stuck not because there's no elasticity, but because:

- requests are written by guesswork
- limits haven't been touched for a long time
- Resources are too large, causing obvious waste
- Resources are too small, causing business instability
- When OOM or scheduling failures occur, there's no continuous optimization basis

VPA's first layer of value is often not "fully automatic," but:

> **First generate resource recommendations, making resource specifications no longer rely on guessing.**

The entire chain of metrics-server, VPA components, CRD, and recommendation is the first step from "guessing resource allocation" to "having metrics, recommendations, and basis."

---

## References
- Metrics Server official repository  
  https://github.com/kubernetes-sigs/metrics-server

- Kubernetes resource metrics chain  
  https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/

- VPA official repository  
  https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler

- VPA Quick Start  
  https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs/quickstart.md

- Kubernetes Vertical Pod Autoscaling  
  https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/

---

## Next Day Suggestions
Next article suggestions compilation:  

[[09-Horizontal Scaling Vertical Scaling Use Cases]]