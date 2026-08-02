# 02-Nginx Static Pages Entering Kubernetes: Basic Practices with Deployment and Service

## Document Description
- Document Position: The first practical guide on deploying applications in Kubernetes.
- Applicable Stage: Beginners who have completed local image building and container running.
- Recommended Path: `04-Kubernetes/07-Application Deployment/02-Stateless Application Deployment/02-Nginx Static Pages Entering Kubernetes: Basic Practices with Deployment and Service`

## Tags
#Kubernetes #Deployment #Service #Nginx #Static Pages #Stateless Applications #Application Deployment #Business Containerization #Cloud Native #Ops #Interview Notes

---

## I. Why This Step Is Important

The previous learning focus was mainly on:

- Preparing page files
- Building images with Dockerfile
- Running containers locally
- Accessing pages via port mapping

From this step onward, the emphasis shifts to:

**Handing over locally running services to Kubernetes for unified management.**

This means understanding two fundamental concepts:

### 1. How Kubernetes manages stateless applications
This involves:

- Deployment
- ReplicaSet
- Pod

### 2. How Kubernetes ensures access to these applications within the cluster
This involves:

- Service
- Pod label selection
- Stable access points

Therefore, the purpose of this practical guide is not just to “run Nginx again” but to use a simple stateless application to establish the basic workflow for Kubernetes application deployment:

**Image → Deployment → Pod → Service → Access within the cluster**

---

## II. Goals of This Practical Guide

After completing this guide, you should be able to:

### 1. Deploy custom Nginx static page images in Kubernetes
### 2. Understand the basic function of Deployment
### 3. Comprehend how Pods are created by Deployment
### 4. Recognize why Service provides a stable access point for Pods
### 5. Distinguish between container ports, Pod ports, and Service ports
### 6. Perform basic troubleshooting for common issues with Deployment and Service

---

## III. Why Choose Nginx Static Pages for Kubernetes

Nginx static pages make an excellent first choice for Kubernetes deployment due to the following reasons:

- The business logic is simple enough.
- It requires few dependencies for startup.
- Container images are easy to build.
- The page delivery results are straightforward.
- It’s ideal for understanding basic concepts such as Pods, Services, and label selection.

More importantly, this type of application represents a typical **stateless application**. It naturally suits management using Deployment and can be easily scaled by adding more replicas.

---

## IV. Understanding the Two Core Objects Used in This Guide

### 1. Deployment
The primary function of Deployment is to:

**Manage stateless application Pods in a declarative manner.**

It handles:

- Creating Pods
- Maintaining the required number of replicas
- Restarting Pods when they fail
- Rolling out updates smoothly

You can think of Deployment as:

**The standard controller for stateless applications.**

---

### 2. Service
Service serves to:

**Provide a stable access point for a group of Pods.**

This is necessary because Pods have the following inherent limitations:

- Their IP addresses may change.
- They might be deleted and recreated.
- They are not suitable as permanent access addresses.

Therefore, Kubernetes requires a stable component to wrap around these backend Pods, and that’s where Service comes in.

You can consider Service as:

**A stable front-end interface for Pods.**

---

## V. Changes in the Access Chain After Entering Kubernetes

During local Docker practice, the access chain typically follows this path:

**Browser → Host machine port → Container port**

In Kubernetes, the basic access chain becomes:

**Client → Service → Pod → Container port**

It’s important to understand that:

- Containers still run inside Pods.
- Pods are where the actual business logic executes.
- However, access requests usually go through Service rather than directly to Pods.

---

## VI. Prerequisites

Before starting this guide, you need to meet the following requirements:

### 1. A usable Kubernetes cluster
For example, your own test cluster.

### 2. An available Nginx page image
For instance:

    harbor.example.com/demo/nginx-web:v1

Or any other name from your repository.

### 3. The cluster nodes must be able to pull this image
If using a private repository, ensure the following are set up correctly:

- Repository address
- Authentication credentials
- Network connectivity
- Certificate trust settings

---

## VII. A Simple Deployment Example

Here’s a basic example of Deployment:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-web
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
            - name: nginx-web
             The page content is ultimately returned by Nginx inside the Pod.

---

## Fourteen, How to Apply These Two YAML Files

The common practice is to save them as separate files, for example:

- `deployment.yaml`
- `service.yaml`

Then apply them to the cluster, like this:

    kubectl apply -f deployment.yaml
    kubectl apply -f service.yaml

### Key Points for Operations and Maintenance Professionals

The focus here is not on the commands themselves, but on developing a routine:

- Declare the objects first.
- Apply the configurations.
- Check the status.
- Verify access.
- Troubleshoot any issues.

---

## Fifteen, What to Check First After Deployment

After completing the deployment, it is recommended to check the following in order:

### 1. Check the Deployment
Focus on:

- Whether it was successfully created.
- Whether the number of replicas meets expectations.

### 2. Check the Pod
Focus on:

- Whether it is Running.
- Whether it is Ready.
- Whether there have been any restarts.
- Whether image pulling failed.

### 3. Check the Service
Focus on:

- Whether it was successfully created.
- Whether the port configuration is correct.

### 4. Check the Endpoints
Focus on:

- Whether the Service has correctly selected the backend Pod.

This check is very important because:

**Just because a Service exists does not mean it has already selected a Pod.**

---

## Sixteen, Why Business May Still Not Work Even Though a Service Is Present

This is a very common issue.

### 1. Selector Mismatch
The tags selected by the Service do not match the actual tags of the Pod.

As a result:

- The Service is created successfully.
- But there is no backend Pod.

### 2. Pod Is Not Ready
Even if the Pod is Running, if it is not Ready, traffic may not be properly forwarded.

### 3. TargetPort Is Incorrectly Entered
For example, if the container actually listens on port 80, but the Service is configured with a different port.

### 4. The Application Does Not Really Listen on the Port
The container starts up, but Nginx does not work correctly, or there is an issue with the image itself.

### 5. Incorrect Access Method
A ClusterIP type Service is only suitable for internal cluster access by default and is not suitable for direct external access.

---

## Seventeen, How Easy It Is to Confuse `port` and `targetPort`

This is a point that beginners often get confused about.

### `targetPort`
It represents:

**The port on which the container inside the Pod actually listens.**

In this practical example, the Nginx container listens on port 80.

### `port`
It represents:

**The port through which the Service provides its services.**

When clients access the Service, they usually first contact the `port`, and then it is forwarded to the `targetPort`.

### Simplified Understanding
- `targetPort`: The port of the backend Pod.
- `port`: The port exposed by the Service itself.

---

## Eighteen, Why Use ClusterIP Here

Since the goal of this article is to first understand the basic coordination between Deployment and Service, there is no need to rush into dealing with external exposure issues.

### Characteristics of ClusterIP
- Default type.
- Only accessible within the cluster.
- Suitable for verifying service discovery and internal forwarding logic first.

### Recommended Learning Order
First, learn:

- How to get the Pod up and running correctly.
- How to ensure the Service correctly selects the Pod.
- How to access it within the cluster.

Then move on to learning:

- NodePort.
- Ingress.
- Domain name access.
- Layer 7 traffic exposure.

---

## Nineteen, Why This Is a Typical Practice for Stateless Applications

Nginx static pages have typical characteristics of stateless applications:

- They do not rely on local critical business data.
- When the Pod is deleted and recreated, its behavior remains basically the same.
- There is no identity difference between multiple replicas.
- Any replica can provide the same page content externally.
- It is very suitable for management using Deployment.

Therefore, one of the key points of this practice is to help you establish a criterion:

> **Stateless applications usually give priority to Deployment.**

---

## Twenty, The Most Important Observation Points in This Practical Example

### 1. Whether the Deployment Successfully Creates Pods
This is the first step in determining “whether the application is being managed by Kubernetes.”

### 2. Whether the Pod Is Really Running and Ready
This is the basic criterion for determining “whether the business is actually running.”

### 3. Whether the Service Has Correctly Selected the Pod
This is the key to determining “whether traffic can be correctly forwarded.”

### 4. Whether the Page Returns the Expected Content
This is the ultimate verification of whether “the business is actually working as expected.”

---

## Twenty[[03-Nginx Static Pages Exposed to the Outside World: Basic Practices with NodePort]]