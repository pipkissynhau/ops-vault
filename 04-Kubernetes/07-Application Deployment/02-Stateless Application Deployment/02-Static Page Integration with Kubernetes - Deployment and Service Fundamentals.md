# 02-Nginx Static Page into Kubernetes: Deployment and Service Basics Practice

## Documentation Notes
- Documentation positioning: First application deployment practice entering Kubernetes
- Applicable stage: Kubernetes beginner practice after completing local image building and container running
- Recommended path: `04-Kubernetes/07-Apply deployment/02-No status application deployment/02-Nginx Static Page Entry Kubernetes:Deployment and Service Basic practice`

## Tags
#Kubernetes #Deployment #Service #Nginx #StaticPage #NoStatusApplication #ApplyDeployment #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## One, Why This Step Is Important

The previous learningMain mainly stayed at:

- Page file preparation
- Dockerfile building image
- Local container running
- Accessing pages through port mapping

From this step onward, the focus shifts to:

**Putting the locally running business under Kubernetes unified management.**

This means we need to start understanding two very core issues:

### 1. How Kubernetes manages stateless applications
This involves:

- Deployment
- ReplicaSet
- Pod

### 2. How Kubernetes allows cluster access to this application
This involves:

- Service
- Pod label selection
- Stable access entry

Therefore, the significance of this hands-on practice isn't "re-running Nginx again," but using the simplest stateless application to establish the most basic Kubernetes application deployment loop:

**Image → Deployment → Pod → Service → Cluster access**

---

## Two, Objectives of This Hands-on Practice

After completing this hands-on practice, it is recommended to at least achieve the following:

### 1. Be able to deploy a custom Nginx static page image to Kubernetes
### 2. Understand the basic role of Deployment
### 3. Understand how Pods are created by Deployment
### 4. Understand why Service can provide a stable access entry for Pods
### 5. Distinguish the relationship between container ports, Pods, and Service ports
### 6. Perform basic troubleshooting for common issues with Deployment and Service

---

## Three, Why Choose Nginx Static Page to Enter Kubernetes

Nginx static pages are very suitable as the first Kubernetes deployment object, reasons include:

- Business is simple enough
- Minimal startup dependencies
- Easy to build container images
- Page access results are intuitive
- Very suitable for observing basic mechanisms like Pods, Services, and label selection

More importantly, this type of business belongs to the typical:

**Stateless application.**

It naturally suits management by Deployment and is suitable for subsequent multi-replica expansion.

---

## Four, First Understand the Two Core Objects Used This Time

### 1. Deployment
The main role of Deployment is:

**To manage stateless application Pods in a declarative way.**

It is responsible for:

- Creating Pods
- Maintaining replica count
- Restarting Pods when they exit abnormally
- Performing rolling updates when the image is updated

You can understand Deployment as:

**The standard controller for stateless applications.**

---

### 2. Service
The main role of Service is:

**To provide a stable access entry for a group of Pods.**

Because Pods have several inherent characteristics:

- Pod IP changes
- Pods may be deleted and recreated
- Pods are not suitable as long-term fixed access addresses

Kubernetes needs a stable object to wrap the backend Pods, which is Service.

You can understand Service as:

**A stable access entry in front of Pods.**

---

## Five, What Changes in the Access Chain After Entering Kubernetes

In local Docker practice, the access chain is usually:

**Browser → Host port → Container port**

After entering Kubernetes, the most basic access chain becomes:

**Client → Service → Pod → Container port**

The most important cognition to establish here is:

- Containers still run inside Pods
- Pods are still where the business truly runs
- But access is usually not directly to Pods
- Instead, it's through Service to access Pods

---

## Six, Preparation Prerequisites

Before doing this hands-on practice, it's usually necessary to meet the following prerequisites.

### 1. Have an available Kubernetes cluster
For example, your own test cluster.

### 2. Have a pullable Nginx page image
For example:

    harbor.example.com/demo/nginx-web:v1

Or other naming in your repository.

### 3. Cluster nodes can pull this image
If it's a private registry, you need to handle:

- Registry address
- Authentication
- Network connectivity
- Certificate trust

---

## Seven, A Simplest Deployment Example

The following is a simplest Deployment example:

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
              image: harbor.example.com/demo/nginx-web:v1
              ports:
                - containerPort: 80

---

## Eight, How to Understand This Deployment

### 1. `kind: Deployment`
Indicates this is a Deployment object.

### 2. `metadata.name: nginx-web`
This is the name of the Deployment itself.

### 3. `replicas: 1`
Indicates the expected number of Pod replicas to run is 1.

### 4. `selector.matchLabels`
Indicates which Pods with specified labels this Deployment will manage.

Here it is:

    app: nginx-web

### 5. `template.metadata.labels`
Indicates what labels the Pods created by it will have.

Here it is also:

    app: nginx-web

### 6. `containers.image`
Indicates which image the container in the Pod uses.

### 7. `containerPort: 80`
Indicates the container application listens on port 80.

---

## Nine, Why `selector` and `labels` Are Important

This is a very critical foundational point for both Deployment and Service.

### How Does Deployment Manage Pods?
It relies on label matching.

### How Does Service Find Backend Pods?
It also relies on label matching.

Labels are a core layer of association mechanism in Kubernetes.

### In This Example
The logic of Deployment is:

- Create Pods with `app=nginx-web` label
- And continuously manage these Pods

If the selector and template labels are inconsistent, Deployment behavior will be abnormal or unable to manage Pods normally.

---

## Ten. A Simplest Service Example

Here is a basic Service example:

    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-web-svc
    spec:
      selector:
        app: nginx-web
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: ClusterIP

---

## Eleven. How to Understand This Service

### 1. `kind: Service`
Indicates this is a Service object.

### 2. `metadata.name: nginx-web-svc`
This is the name of the Service itself.

### 3. `selector`
Indicates which Pods the Service will forward traffic to.

Here is the selection:

    app: nginx-web

This means it will find all Pods with this label.

### 4. `port: 80`
Indicates the access port provided by the Service is 80.

### 5. `targetPort: 80`
Indicates the Service forwards traffic to the 80 port of the Pod container.

### 6. `type: ClusterIP`
Indicates this is a Service accessible within the cluster.

---

## Twelve. How Deployment and Service Work Together

This step must be clearly understood.

### What Does Deployment Handle?
- Create and manage Pods
- Ensure replica count
- Ensure application continuous operation

### What Does Service Handle?
- Find target Pods
- Provide a unified, stable access entry for these Pods
- Shield Pod IP changes

### How Do They Connect?
Through labels.

In other words:

- Deployment adds labels when creating Pods
- Service selects these labeled Pods via selector
- Then forwards traffic to them

---

## Thirteen. How to Understand the Minimal Complete Chain

Using this Nginx page example, the complete chain can be understood as:

### 1. The image contains your static pages
This is the source of business content.

### 2. Deployment creates Pods using this image
Pods actually run Nginx containers.

### 3. Nginx in the Pod listens on port 80
This is the actual listening port of the business.

### 4. Service selects these Pods
Service finds backend Pods based on labels.

### 5. When accessing Service, traffic is forwarded to Pods
Finally, the Nginx in the Pod returns the page content.

---

## Fourteen. How to Apply These Two YAMLs

The usual practice is to save them as separate files, for example:

- `deployment.yaml`
- `service.yaml`

Then apply them to the cluster, for example:

    kubectl apply -f deployment.yaml
    kubectl apply -f service.yaml

### Operation Understanding Focus

The focus here is not the commands themselves, but forming a habit:

- Declare objects first
- Then apply
- Then check status
- Then verify access
- Then troubleshoot anomalies

---

## Fifteen. What to Check After Deployment

After completing deployment, it is recommended to check the following in order.

### 1. Check Deployment
Focus on:

- Whether it was created successfully
- Whether replica count reached expected

### 2. Check Pod
Focus on:

- Whether it is Running
- Whether it is Ready
- Whether there were restarts
- Whether image pull failed

### 3. Check Service
Focus on:

- Whether it was created successfully
- Whether port configuration is correct

### 4. Check Endpoints
Focus on:

- Whether Service actually selected backend Pods

This check is very critical because:

**Service existing does not mean it has definitely selected Pods.**

---

## Sixteen. Why Service Exists, But Business May Still Not Work

This is a highly frequent issue.

### 1. Selector Mismatch
Service's selected labels and Pod's actual labels are inconsistent.

Result is:

- Service created successfully
- But no backend Pods

### 2. Pod Not Ready
Even if Pod is Running, if not Ready, it may not receive traffic normally.

### 3. targetPort Written Incorrectly
For example, container actually listens on 80, but Service configured to another port.

### 4. Application Not Actually Listening Port
Container started, but Nginx not working normally, or image itself has issues.

### 5. Access Method Incorrect
ClusterIP type Service defaults only suitable for cluster internal access, not directly from outside cluster.

---

## Seventeen. How `port` and `targetPort` Are Easily Confused

This is a common confusion point for beginners.

### `targetPort`
Indicates:

**The port the container in Pod actually listens on.**

In this practical case, Nginx container listens on 80.

### `port`
Indicates:

**The port Service itself provides.**

Clients access Service by first hitting `port`, then forwarded to `targetPort`.

### Simplified Understanding
- `targetPort`: Backend Pod port
- `port`: Service's exposed port

---

## Eighteen. Why ClusterIP Is Used Here

Because the goal of this article is to first understand the basic collaboration between Deployment and Service, not to handle external exposure issues urgently.

### ClusterIP Characteristics
- Default type
- Only accessible within the cluster
- Suitable for first verifying service discovery and internal forwarding logic

### Learning Order Recommendation
First learn:

- Pod can start normally
- Service can correctly select Pods
- Cluster internal access works

Then continue learning:

- NodePort
- Ingress
- Domain access
- Layer 7 traffic exposure

---

## Nineteen. Why This Is a Typical Stateless Application Practice

Nginx static pages have typical stateless application characteristics:

- No dependency on local critical business data  
- Pod is recreated after deletion, behavior remains consistent  
- No identity differences between multiple replicas  
- Any replica can provide the same page content  
- Well-suited for Deployment management  

Thus, one of the key focuses of this practice is to help establish a judgment standard:  

> **Stateless applications typically prioritize Deployment.**  

---  

## Twenty, Key Observations in This Practical  

### 1. Whether Deployment successfully creates Pod  
This is the first step in determining whether the application is managed by Kubernetes.  

### 2. Whether Pod is truly Running and Ready  
This is the foundational judgment for determining if the business is actually operational.  

### 3. Whether Service selects Pod  
This is the critical judgment for determining if traffic can be correctly forwarded.  

### 4. Whether the page returns expected content  
This is the final verification that the business is running as expected.  

---  

## Twenty-one, Common Issues and Basic Troubleshooting Approaches  

### 1. Pod cannot start up  
Common causes:  

- Incorrect image address  
- Private registry authentication failure  
- Image does not exist  
- Node cannot pull image  

Troubleshooting directions:  

- Check Pod status  
- Check events  
- Check `ErrImagePull` / `ImagePullBackOff`  

---  

### 2. Pod is running, but page access is blocked  
Common causes:  

- Service selector mismatch  
- targetPort error  
- Pod not Ready  
- Container not actually listening on port 80  

Troubleshooting directions:  

- Check Service configuration  
- Check Pod labels  
- Check Endpoints  
- Check container logs  

---  

### 3. Service created successfully, but no backend  
Common causes:  

- Selector label error  
- Deployment template label error  

Troubleshooting directions:  

- Cross-check:  
  - Deployment template labels  
  - Service selector  

---  

### 4. Page returns content that is not its own  
Common causes:  

- Image is not the expected version  
- Running image is not the new one  
- Page files are not correctly included in the image  

Troubleshooting directions:  

- Check image tag  
- Check actual image used by Pod  
- Troubleshoot at the image build layer  

---  

## Twenty-two, Extensions from This Practical  

After completing this practice, you can naturally proceed to the following topics.  

### 1. Replica Scaling  
Change `replicas` from 1 to 2, 3, and observe the behavior of stateless application with multiple replicas.  

### 2. NodePort  
Allow external access to this Nginx page.  

### 3. Ingress  
Expose the page via domain or path.  

### 4. ConfigMap  
Decouple Nginx configuration or page content from the image.  

### 5. Probe  
Add health checks for Nginx.  

### 6. Rolling Update  
Modify image tag and observe Deployment update behavior.  

---  

## Twenty-three, Learning Objectives for This Section  

At this stage, it is recommended to reach the following level:  

### 1. Understand the basic structure of Deployment  
### 2. Understand the basic structure of Service  
### 3. Understand the role of labels in Deployment and Service  
### 4. Understand the difference between `port` and `targetPort`  
### 5. Explain what Deployment and Service are responsible for  
### 6. Perform basic troubleshooting for "Service exists but business is unreachable"  

---  

## Twenty-four, Common Interview Extensions  

Common questions in interviews include:  

- What's the relationship between Deployment and Pod  
- What's the difference between Deployment and StatefulSet  
- Why can't Service be omitted  
- How does Service find backend Pod  
- What's the role of selector and labels  
- What's the difference between `port` and `targetPort`  
- Why is Service normal but business is not  
- What's the difference between ClusterIP, NodePort, LoadBalancer  
- Why stateless applications prioritize Deployment  

---  

## Twenty-five, Stage Summary  

Deploying an Nginx static page into Kubernetes is the first step of business containerization entering the orchestration layer.  

The most important thing at this stage is not memorizing YAML, but establishing the following core understandings:  

- Deployment is responsible for creating and maintaining stateless application Pods  
- Pod is where the business actually runs  
- Service provides a stable access entry for a group of Pods  
- Deployment and Service associate through labels  
- Service existence does not guarantee it has selected backend Pods  
- Cluster internal access chain is typically: Service → Pod → container port  

As long as you clearly understand these relationships, subsequent learning about multi-replica, rolling updates, NodePort, Ingress, Probe, and ConfigMap will be much smoother.  

---  

## Twenty-six, Keyword Quick Notes  

- Deployment: Controller for stateless applications  
- Pod: Basic unit of application runtime in Kubernetes  
- Service: Provides stable access entry for Pods  
- selector: Label condition for selecting target Pods  
- labels: Labels on resource objects  
- ClusterIP: Service type for internal cluster access  
- targetPort: Container listening port of Pod  
- port: Access port provided by Service  

---  

## Twenty-seven, Operational Perspective Understanding  

From an operational perspective, although this practice is simple, it already covers the most fundamental four-layer relationships of Kubernetes application deployment:  

- Image layer: Whether page content is included in the image  
- Orchestration layer: Whether Deployment correctly creates Pod  
- Service layer: Whether Service correctly selects backend  
- Access layer: Whether traffic actually reaches container port  

When encountering more complex business systems, gateway services, or middleware deployments later, the essence is still to continue adding configurations, probes, storage, release strategies, and traffic governance capabilities on these layers.  

Therefore, it is more effective to clearly complete this simplest stateless application practice first, rather than directly starting with complex middleware.  

---  

## References  
- Kubernetes Deployment: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/  
- Kubernetes Service: https://kubernetes.io/docs/concepts/services-networking/service/  
- Kubernetes Labels and Selectors: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/  
- Kubernetes Pods: https://kubernetes.io/docs/concepts/workloads/pods/  

---  

## Next Day Suggestions  
Next post suggestion to organize:  

[[03-Nginx Exposing Static Pages - NodePort Basics]]