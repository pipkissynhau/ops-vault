# 08-VPA Basics: Vertical Scaling, Prerequisites for Installation, and Basic Practices

## Document Description
- Document Position: An introductory guide to VPA (Vertical Pod Autoscaler) within the context of Kubernetes resource management and auto-scaling.
- Applicable Stage: After understanding the basics of requests/limits, Pending status, QoS, eviction, and HPA, this section covers the prerequisites for installing VPA, its core components, installation methods, validation approaches, and simple experiments.
- Recommended Path: `04-Kubernetes/07-Application Deployment/08-Resource Management and Auto-scaling/08-VPA Basics: Vertical Scaling, Prerequisites for Installation, and Basic Practices.md`

## Tags
#Kubernetes #VPA #VerticalPodAutoscaler #metrics-server #CRD #CustomResourceDefinition #vertical scaling #resource management #auto-scaling #application deployment #cloud-native #ops #interview notes

---

## I. Why Learn VPA After HPA
We have already covered several key concepts in this series:

- `requests/limits` affect scheduling and operational stability.
- Pods may enter the `Pending` state due to insufficient resources.
- `QoS` determines the priority of resource allocation under high demand.
- Evictions can occur when node resources are strained.
- HPA automatically adjusts the number of replicas based on metrics.

However, even with HPA, it's not guaranteed that new Pods will successfully handle increased traffic. This leads us to consider another approach:

> **Vertical scaling**, which involves adjusting the resource specifications of individual Pods rather than simply increasing their numbers.

In real production environments, not all issues can be resolved by adding more Pods. Other scenarios include:

- It might not be feasible to easily increase the number of Pod replicas.
- Some components rely heavily on specific single-instance resources.
- The current requests for certain services are difficult to predict accurately.
- Pod configurations may be either too small or too large over time.
- Excessively conservative or aggressive resource allocation can lead to OOM issues, restarts, or performance instability.

In these cases, we need:

- **Optimize the resource specifications of individual Pods**, rather than simply adding more of them.

This brings us to VPA, one of the most common vertical scaling mechanisms in the Kubernetes ecosystem.

---

## II. What is “Vertical Scaling”
When discussing auto-scaling in Kubernetes, HPA is often the first that comes to mind. It's straightforward:

- Initially, there are 2 Pods.
- When demand increases, 4 Pods are created.
- When demand decreases, the number of Pods is reduced.

This is called **horizontal scaling**, which involves changing the total number of instances without altering the resource specifications of individual instances.

However, another approach is to adjust the resource specifications of a single Pod, rather than increasing its number. For example:

- The original CPU request was `200m`, and now it's changed to `500m`.
- Or the memory request was `256Mi`, and now it's changed to `1Gi`.

The focus here is not on **increasing** resources but on **adjusting them** to optimize performance.

This is known as **vertical scaling**. We can simply distinguish between the two as follows:

#### Horizontal Scaling
- Adjusts the number of replicas.
- Representative solution: HPA

#### Vertical Scaling
- Adjusts the resource specifications of a single Pod.
- Representative solution: VPA

---

## III. What Exactly is VPA
VPA stands for **Vertical Pod Autoscaler**. It can be understood as a mechanism that automatically recommends or applies more appropriate CPU/memory requests based on a Pod's actual resource usage.

VPA primarily addresses the following issues:

- How much resources should a single Pod have?
- Are the current resource requests too low or too high?
- Can automating these recommendations reduce the time and effort required to manually adjust resource configurations?

### For now, remember this key point:
> **HPA focuses on determining whether there are enough replicas, while VPA focuses on ensuring that individual Pods have appropriate resource configurations.**

---

## IV. Why Is It Important to Explain the “Prerequisites for Installation” When Learning VPA
This is crucial. Many people initially assume that VPA can be used directly by writing a YAML file, just like Deployment or HPA. However, this is not the case.

### Key Conclusion
> **VPA is not a built-in capability in most Kubernetes clusters by default.**

For VPA to function effectively, two prerequisites are essential:

### 1. Resource Metric Source
This refers to the **metrics-server**.

### 2. VPA’s Own Components and Custom Resources
These include **CRDs**, **recommender**, **updater**, and **admission-controller**.

Therefore, when learning VPA, it's important to understand whether your cluster has these components and whether they are configured correctly.

---

## V. Why Install metrics-server#### `--kubelet-insecure-tls`
This indicates that:

- Strict TLS verification is skipped when accessing the kubelet.

This parameter is commonly used in:

- Experimental environments
- Troubleshooting self-built clusters
- Scenarios where the kubelet serving cert configuration is non-standard

### However, it's important to understand correctly

> **It is more suitable for testing or troubleshooting and is not the preferred option for production environments.**

In a production environment, it is still best to configure the kubelet certificate chain and access links properly.

---

## Ten: If the metrics-server is already installed, the next step is to install VPA

It's best not to reverse this order:

### Step One
First, confirm that `kubectl top nodes` and `kubectl top pods -A` are working correctly.

### Step Two
Then start installing VPA.

This is because VPA requires a source for resource metrics, and the metrics-server is one of the most common and standard sources.

---

## Eleven: Why VPA is not available by default

This point must also be clear.

Many people ask at this stage:

- I already have a metrics-server. Can't I just create a `VerticalPodAutoscaler` directly?

No, you can't.

Because the `VerticalPodAutoscaler` is not an ordinary core resource but:

> **a custom resource introduced through a CRD.**

In other words:

### Before installing VPA
The cluster usually doesn't recognize:

- `VerticalPodAutoscaler`

### After installing VPA
The cluster will have additional components such as:

- CRD
- Controller components
- Admission-controller
- Recommender
- Updater

So it's essential to understand that:

> **metrics-server addresses the issue of where resource metrics come from, while the VPA components handle the actual vertical scaling capability.**

---

## Twelve: What components does VPA generally include?

You don't need to dive into the source code at this stage, but you should have an understanding of the components.

### 1. CRD
This allows the cluster to recognize:

- `VerticalPodAutoscaler`

### 2. Recommender
This is responsible for:

- Calculating more appropriate CPU / Memory requests based on historical and current resource usage

You can think of it as:

> **a resource recommendation calculator**

### 3. Updater
This is responsible for:

- Prompting existing Pods to be rebuilt or updated according to the recommended values when needed

You can think of it as:

> **the component that makes the recommendations take effect**

### 4. Admission Controller
This participates in the process of injecting resource recommendations or updating resources during Pod creation.

You can think of it as:

> **the entry point for resource recommendations to enter the actual object creation process**

---

## Thirteen: How to install VPA

### Step One: Clone the official autoscaler repository

    git clone https://github.com/kubernetes/autoscaler.git
    cd autoscaler/vertical-pod-autoscaler

### Step Two: Check the directory structure

    ls
    ls hack
    ls examples

At this stage, focus on:

- `hack/`
- `examples/`

### Step Three: Look at the installation scripts

    ls hack

You will usually see:

- `vpa-up.sh`
- `vpa-down.sh`

### Step Four: Execute the installation

    ./hack/vpa-up.sh

### Step Five: Check the CRD

    kubectl get crd | grep -i verticalpodautoscaler

### Step Six: Check the components

    kubectl get pods -A | grep -i vpa
    kubectl get deployment -A | grep -i vpa

### You should see components such as:
- recommender
- updater
- admission-controller

### The most important criteria for success

Success in installing VPA is not just about whether the scripts execute without errors but also:

#### 1. The cluster already has the relevant VPA CRD.
#### 2. VPA-related Pods / Deployments are running.
#### 3. You can successfully create a `VerticalPodAutoscaler` object later on.

---

## Fourteen: Installing VPA and creating a specific business VPA object are two different things

This point must be remembered separately.

### First step: Install the VPA capability
This involves:

- Installing the CRD
-Installing the recommender
-Installing the updater
-Installing the admission-controller

The result of this step is:

> **The cluster now has the ability to use VPA.**

### Second step: Create a VPA object for a specific business workload
This step involves:

- Creating `VerticalPodAutoscaler` rules for a particular Deployment / StatefulSet

The result of this step is:

> **A specific business begins to be managed by VPA.**

So don't confuse the two steps into one statement like:

- “Just install### 2. Check if the VPA components are functioning correctly

    kubectl get pods -A | grep -i vpa
    kubectl logs -n kube-system deployment/vpa-recommender
    kubectl logs -n kube-system deployment/vpa-updater
    kubectl logs -n kube-system deployment/vpa-admission-controller

If your actual namespace is different from `kube-system`, adjust the commands accordingly based on your installation location.

### 3. Verify if the cluster recognizes the `VerticalPodAutoscaler`

    kubectl get crd | grep -i verticalpodautoscala
    kubectl api-resources | grep -i verticalpodautoscaler

### 4. Ensure the `targetRef` is set correctly

In many cases, the issue is not with the VPA itself, but rather:

- The namespace is incorrect.
- The name of the Deployment is wrong.
- The `targetRef` points to the wrong object.

### 5. Give it some time

VPA recommendations do not appear immediately after you apply them. Observe for a while and then check the results using `kubectl describe`.

---

## Section 21: Recommended Learning Sequence at This Stage

To avoid getting confused at the beginning, it is suggested to follow this sequence:

### Step 1: Verify the metrics-server first
Ensure that the resource metric pipeline is established.

### Step 2: Install the VPA component
Enable the cluster's capability for `VerticalPodAutoscaler`.

### Step 3: Conduct a minimal experiment with `updateMode: Off`
Observe the recommendations first.

### Step 4: Understand the recommendations
First, comprehend what the recommendations suggest.

### Step 5: Learn about the automatic update mode
Only after you understand all the previous steps should you explore modes such as `Initial` and `Recreate`.

---

## Section 22: How to Answer “How Do You Implement VPA” in an Interview

You can answer this question by structuring your response as follows:

Before using VPA, we first ensure that the cluster has a source for resource metrics. This typically involves installing and verifying the metrics-server to ensure that `kubectl top` can correctly display CPU and memory data for Pods and Nodes.  
Next, we install the VPA components, including the CRD, recommender, updater, and admission-controller.  
After installation, we do not immediately enable automatic adjustments to business resources. Instead, we create a `VerticalPodAutoscaler` object with `updateMode: Off` for a test workload to observe the recommendations first. Only after confirming that everything is working properly do we consider transitioning to more automated update modes.

### Key terms to remember in this response:

- Verify metrics-server first
- Install VPA components later
- `VerticalPodAutoscaler` is a custom resource
- Start with `updateMode: Off`
- Observe recommendations before automating updates

---

## Section 23: Key Takeaways from This Article

### 1. VPA is about vertical scaling
It focuses on:

- The resource specifications of individual Pods
- Whether CPU/Memory requests are reasonable

### 2. VPA is not ready out of the box
It typically requires the following components:

- metrics-server
- CRD
- recommender
- updater
- admission-controller

### 3. Install the metrics-server before VPA
This order should not be reversed.

### 4. It is recommended to clone the official repository and install locally at this stage
This approach makes it easier to review the list of components, modify parameters, adapt to internal network environments, and prepare for future use.

### 5. Installing VPA and creating a business VPA object are different steps
First, establish the capability, and then create the rule objects.

### 6. It is advisable to start with `updateMode: Off` for minimal experiments
This allows you to observe the recommendations before making automatic changes to resources.

---

## Section 24: Summary of This Article

The most important thing here is not to try to learn all the details of VPA at once, but rather to ensure that the following steps are completed:

1. Understand why VPA is more than just writing a YAML file.
2. Recognize the importance of installing the metrics-server first.
3. Learn why it is recommended to clone the official repository and install locally.
4. Be able to install and verify both the metrics-server and VPA components.
5. Know how to create a `updateMode: Off` minimal experiment.
6. Understand how to use `kubectl describe vpa` to view recommendations.

By doing this, you will have established a comprehensive foundational understanding of VPA:

> **VPA is not just a simple concept; it represents a complete process from resource metrics sources to custom resources, recommendation calculations, and practical application in business scenarios.**

Its true value lies not in automatically adjusting resources, but in helping us address a very real question:

