# 03-Ingress Controller Basics: Why Just Ingress YAML Isn't Enough

## Document Notes
- Document Positioning: Core principles section of Ingress learning during the Kubernetes service discovery and traffic exposure phase
- Applicable Stage: After understanding Service, NodePort, and "why Ingress is needed", begin to deeply understand how Ingress actually works
- Recommended Path: `04-Kubernetes/07-Apply deployment/06-Service detection and traffic exposure/03-Ingress Controller Foundation: Why only Ingress YAML Not enough..md`

## Tags
#Kubernetes #Ingress #IngressController #ingress-nginx #Service #NodePort #7thFloorForward #TrafficEntrance #ReverseAgent #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## I. Why You Must Continue Learning Ingress Controller After Understanding Ingress's Role

The previous article solved one key question:

**Why do we need to continue learning Ingress after understanding Service and NodePort?**

We already know:

- Service solves the abstraction of backend services
- NodePort solves basic external access to the cluster
- Ingress solves the unified entry point for HTTP/HTTPS

But if we stop here, our understanding remains vague.  
Because many people encounter a very real problem when first encountering Ingress:

**Why isn't traffic flowing even though I've written an Ingress YAML?**

This is the core issue this article aims to solve.

The answer is:

**The Ingress resource object itself is just a "rule declaration". The Ingress Controller is what actually reads the rules, receives requests, and executes forwarding.**

In other words:

- Ingress: Rules
- Ingress Controller: The program that executes the rules

Without an Ingress Controller, the Ingress YAML you create is often just "an object in etcd" and won't automatically become a working business entry point.

---

## II. Conclusion First: What Is Ingress and What Is the Controller

To avoid confusion later, let's first present the core conclusions.

### 1. What Is Ingress
Ingress is a resource object in Kubernetes.  
It is mainly used to describe:

- Which domain
- Which path
- Which Service to forward traffic to

In essence, Ingress is:

**A seven-layer entry rule declaration.**

It describes "how to forward", not "to actually forward".

### 2. What Is an Ingress Controller
An Ingress Controller is an actual running program deployed in the cluster.  
It is responsible for:

- Listening to Ingress resources in Kubernetes
- Parsing these rules
- Loading these rules into its own entry proxy program
- Actually receiving external HTTP/HTTPS requests
- Then forwarding requests to the corresponding Service based on the rules

So the essence of an Ingress Controller is:

**The entry control program that actually executes Ingress rules.**

### 3. Why Can't We Just Create Ingress
Because the Ingress object is not a proxy program, it won't listen on ports or forward traffic itself.

This is like:

- You write a traffic distribution rule table
- But no program reads or executes this rule table

Requests will naturally not be forwarded.

---

## III. Why Ingress YAML Is Just a "Rule Declaration"

This statement is very important and must be thoroughly understood.

When you create the following YAML:

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: nginx-web-ingress
    spec:
      rules:
        - host: nginx.example.com
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: nginx-web-svc
                    port:
                      number: 80

Many beginners intuitively think:

- I created this object
- Kubernetes should automatically start forwarding

But in reality, Kubernetes won't magically create a seven-layer proxy program just because you created an Ingress object.

This YAML only tells the system:

- When the request domain is `nginx.example.com`
- And the path matches `/`
- It should forward traffic to `nginx-web-svc:80`

However:

**Who receives this request? Who judges the Host? Who judges the Path? Who actually forwards the request to the Service?**

None of these tasks are done by the Ingress resource object itself—they are all handled by the Ingress Controller.

So you can initially understand Ingress as:

**A "seven-layer routing rule sheet" in Kubernetes.**

---

## IV. What Happens Without an Ingress Controller

This question must be clearly answered, otherwise confusion will persist.

Assume you have:

- A Deployment
- A Service
- An Ingress YAML

But no Ingress Controller is deployed in the cluster.

Usually, this results in:

### 1. `kubectl get ingress` Can See the Object
In other words, from a Kubernetes resource perspective, it already exists.

### 2. But No Real Entry Program to Take Over
No program reads these rules, and no program listens for external requests.

### 3. Business Access Usually Still Doesn't Work
You might see:

- Domain access fails
- The entry has no response
- Rules are completely ineffective
- Access results don't match expectations

### 4. Core Reason
The core issue isn't a YAML syntax error—it's:

**There's no executor.**

This is why many people feel confused when first learning Ingress:

"Why does it still fail even though the object was created?"

The answer is:

**Having rules doesn't mean having a program to execute them.**

---

## V. What Role Does the Ingress Controller Play in the Entire Chain

This is the core layer of this article.

You can think of the Ingress Controller as:

**A gateway program deployed in the cluster that specifically handles entry proxying and rule execution.**

It mainly does four things.

### 1. Watch for Ingress Resource Changes
The Ingress Controller continuously watches the Kubernetes API, paying attention to: /think

- Ingress
- Service
- Endpoints
- Pod
- Secret (e.g. TLS certificate)
- IngressClass and related resources

Controller will recalculate the ingress configuration whenever any of these resources change.

### 2. Convert Ingress rules into proxy configuration
For example, if using ingress-nginx, it will convert Ingress resources into Nginx executable routing configuration.

In other words:

- Ingress is a rule at the Kubernetes object level
- ingress-nginx translates it into Nginx configuration

### 3. Actually listen to incoming traffic
The Ingress Controller itself provides an entry point, for example listening on:

- Port 80
- Port 443

External requests first arrive here, not directly "to a specific Ingress YAML".

### 4. Forward requests to backend Service according to rules
After receiving requests, the Controller will determine:

- What is the Host
- What is the Path
- Which rule should be matched
- Which Service should the request be forwarded to

Then the Service continues forwarding traffic to the backend Pod.

---

## Six. How to understand the Ingress chain

Many people often confuse two perspectives here.

Actually, when understanding Ingress, it's best to view it from two perspectives.

---

## Seven. First perspective: Configuration relationship perspective

From the configuration relationship perspective, it's usually written as:

    Ingress
    -> Service
    -> Pod

This chain emphasizes:

- Ingress's backend points to Service
- Service associates with backend Pod

This is the most common understanding when viewing YAML resource relationships.

This chain is fine, but it misses one layer:

**Who executes the Ingress rules.**

---

## Eight. Second perspective: Real request traffic perspective

From the perspective of real requests entering the system, the chain should be written as:

    Client
    -> Ingress Controller's entry address
    -> Ingress rule matching
    -> Service
    -> Pod
    -> Container process

This is a more accurate understanding of the runtime process.

### Why is it like this
Because external clients don't "request a YAML file".  
When a client sends a request, what actually arrives first is:

**The entry address exposed by the Ingress Controller.**

Only after the Controller receives the request will it match:

- Which Host
- Which Path
- Which backend Service to use

Therefore:

### Configuration relationship perspective
    Ingress -> Service -> Pod

### Request traffic perspective
    Client -> Controller entry -> Ingress rule matching -> Service -> Pod

These two chains are not conflicting, but rather different observation angles.

---

## Nine. Why external requests must first reach the Controller, not the Ingress

This is the most common confusion for beginners.

### 1. Ingress is not a process
Ingress is just a Kubernetes resource object, it will not:

- Listen on port 80
- Listen on port 443
- Accept TCP connections
- Return HTTP responses

### 2. The one that listens to ports is the Controller
The actual program that listens to ports and receives requests is the proxy program in the Ingress Controller, for example:

- Nginx in ingress-nginx
- Traefik process in Traefik
- Actual gateway program in other Controllers

### 3. Therefore, external requests first reach the Controller
For example, accessing:

    http://nginx.example.com/

The actual process is:

- The domain resolves to an entry address first
- This entry address is actually the address exposed by the Controller
- Only after the Controller receives the request will it decide how to forward based on Ingress rules

So remember this key point:

**Clients never "access the Ingress object", but always access the entry exposed by the Ingress Controller.**

---

## Ten. How is the Ingress Controller deployed in the cluster

Since the Ingress Controller is a program, it must also be deployed like other applications.

Typically, a Controller will also have its own Kubernetes resources in the cluster, for example:

- Deployment
- Pod
- Service
- ConfigMap
- Admission Webhook
- RBAC permissions
- IngressClass

What does this indicate?

It indicates that the Ingress Controller is not a "hidden capability" automatically possessed by Kubernetes, but rather a component that needs to be installed, run, exposed, and maintained.

Taking the most common ingress-nginx as an example, it usually runs in a dedicated namespace, for example:

    ingress-nginx

Inside there will be:

- ingress-nginx-controller Pod
- ingress-nginx-controller Service

These resources are the runtime carrier of the Controller itself.

---

## Eleven. Why the Controller itself also needs a Service

This is a very critical but often overlooked point.

Many people will ask here:

**Since the Ingress Controller is already running in the cluster, how does external traffic reach it?**

The answer is:

**The Controller itself usually also needs to expose itself through a Service.**

In other words, the Controller is an application running in the cluster.  
As an application, it needs to have:

- Pod
- Corresponding Service
- Access method

For example, a common practice for ingress-nginx Controller is:

### 1. In test environments / bare-metal environments
The Controller's Service might be:

- NodePort

At this point, external requests actually first access:

    NodeIP:NodePort

Then this NodePort traffic enters the ingress-nginx Controller.

### 2. In cloud-hosted environments
The Controller's Service might be:

- LoadBalancer

At this point, the cloud provider will create an ELB/SLB/LB resource for the Controller, and external requests first hit this load balancer before entering the Controller.

So you'll find:

**The Ingress Controller itself actually needs an "entry point".**

This is why, in real environments, understanding Ingress requires not only looking at the business Ingress YAML, but also checking:

- Whether the Controller exists
- Whether the Controller's Pod is healthy
- What type of Service the Controller has
- What is the entry address of the Controller

--- /think

## Twelve. Why is the Controller the key to whether Ingress works

Because whether Ingress can truly work depends not on "whether the object has been created," but on:

### 1. Is there a running Controller
Without a Controller, there is no executor.

### 2. Has the Controller taken over this Ingress rule
Not all Ingress rules are automatically handled by any Controller.

### 3. Is the Controller's entry point reachable
Even if the rule is written correctly, if the Controller's external entry point is unreachable, the service remains inaccessible.

### 4. Does the Controller correctly read the backend Service
Even if the entry can receive requests, if the rule doesn't match the correct Service, traffic remains blocked.

So the prerequisite for Ingress to work is not "having YAML," but:

**Having a runnable Controller, and it really takes over your rules, and can be accessed externally.**

---

## Thirteen. What is ingress-nginx, and why is it the most suitable for use as an example

In the Kubernetes ecosystem, there are many implementations of Ingress Controller, such as:

- ingress-nginx
- Traefik
- HAProxy Ingress
- Cloud provider's built-in Ingress Controller

For the current stage, the most suitable one to use as a teaching example is:

**ingress-nginx**

The reasons are mainly several.

### 1. It's the most common
Many resources, tutorials, interview questions, and practical articles default to talking about ingress-nginx.

### 2. It's easiest to visualize
Its working model is relatively easy to understand:

- Write Ingress rules in Kubernetes
- ingress-nginx Controller reads these rules
- Converts them into Nginx configuration
- Nginx forwards requests according to the rules after receiving them

### 3. It's convenient for migration understanding
Later, when you learn cloud provider's ALB Ingress, SLB Ingress, or managed cluster entry points, you'll find the underlying logic is still:

- The rule object exists in Kubernetes
- A certain Controller is responsible for implementing these rules

So now using ingress-nginx to build a foundation is the most stable.

---

## Fourteen. Is the Ingress Controller the same as the Nginx Pod

No, this is a common confusion.

Assume you will create a minimal example later:

- A business Deployment: nginx-web
- A business Service: nginx-web-svc
- An Ingress Controller: ingress-nginx

Here are at least two different concepts of "nginx."

### 1. Business Nginx
For example, your business Pod is:

- nginx-web

It is responsible for providing web content and is your business application itself.

### 2. ingress-nginx Controller
It is also an entry controller implemented based on the Nginx technology route, but its responsibility is not to provide your business pages, but:

- Receives incoming traffic
- Matches Host/Path rules
- Forwards traffic to the backend Service

So never confuse:

- Business Nginx
- ingress-nginx Controller

One is:

**Backend business application**

The other is:

**Frontend entry proxy controller**

---

## Fifteen. Why creating an Ingress might still not work at all

We won't expand on comprehensive troubleshooting here, but focus on typical reasons related to the Controller.

### 1. No Ingress Controller is installed at all
The most basic issue, many test environments lack this.

### 2. The Controller is not running normally
For example:

- Pod CrashLoopBackOff
- Pod not Ready
- Controller service anomalies

### 3. The Controller's Service is not correctly exposed
For example:

- NodePort not open
- LoadBalancer hasn't allocated an address
- Network security policies block traffic

### 4. The Ingress is not taken over by this Controller
Even if the object exists, the Controller may not process it.

### 5. The domain name hasn't resolved to the Controller's entry address
Even if the rule is completely correct, the request never reaches the Controller, so it remains inaccessible.

So you should now establish this understanding:

**Whether Ingress works first depends not on the business Pod, but first on whether the Controller layer is valid.**

---

## Sixteen. What is IngressClass, and why is it related to the Controller

At this stage, first establish a basic understanding without diving too deep.

In a cluster, there may be multiple different Ingress Controllers.
For example:

- One ingress-nginx
- One cloud provider's Ingress Controller

This creates a problem:

**Which Ingress should be handled by whom?**

This requires a "ownership relationship" to tell the system:

- This Ingress rule should be taken over by which Controller

This relationship is usually reflected through:

- `IngressClass`
- `ingressClassName`

You should remember this sentence at this stage:

**The purpose of IngressClass is to establish an ownership relationship between Ingress rules and a specific Controller.**

Later, when doing examples, you'll often see such a field:

    ingressClassName: nginx

This usually means:

- This Ingress rule should be handled by the Controller corresponding to the IngressClass `nginx`

If this layer doesn't match, it may result in:

- The Ingress object is created
- But the Controller doesn't take it over
- The business access remains inaccessible

---

## Seventeen. From an operations perspective, why understanding the Controller is particularly important

Because many Ingress issues are essentially not problems with the business itself, but the entry layer isn't connected.

For example, these phenomena:

- The business Pod is normal
- The Service is normal
- Endpoints exist
- But domain name access is still inaccessible

Without understanding the Controller, it's easy to misdiagnose as:

- The application has issues
- The Service has issues
- The network has issues

But the real situation may be:

- No Controller is installed
- The Controller Pod is abnormal
- The Controller entry address isn't reachable
- The Ingress isn't properly taken over
- The domain name hasn't resolved to the Controller

So from an operations perspective, the first reaction to troubleshooting Ingress shouldn't be "first modify the business," but rather first confirm:

**Whether the Controller layer is valid.**

---

## Eighteen. A minimal mental model: How to completely separate Ingress and Controller

If you still find it confusing, you can first use the following model to remember.

### 1. Ingress is a Rule Set
It defines:

- This domain
- This path
- Which Service to forward to

### 2. Ingress Controller is the Executor
It is responsible for:

- Listening for rules
- Receiving requests
- Matching rules
- Forwarding traffic

### 3. External requests first go to the Controller
Not to the Ingress object.

### 4. The Controller then forwards based on Ingress rules
After a match is successful, it forwards the request to the Service.

### 5. The Service then forwards the request to the Pod
This part is still the Service capability you've already learned.

You can imagine the entire process as:

    Ingress = Routing table
    Controller = The person who looks at the routing table and actually directs traffic
    Service = Backend service entry point
    Pod = The actual business instance handling the request

Once this mental model is established, all subsequent configurations and troubleshooting will be much smoother.

---

## 19. How to explain "Why just Ingress YAML is not enough" in an interview

If the interviewer asks:

**Why is creating an Ingress resource not enough?**

You can answer:

The Ingress resource itself is just a seven-layer routing rule object in Kubernetes. It describes the mapping relationship between domain names, paths, and backend Services, but it does not listen on ports or directly forward traffic.  
The actual receiver of external requests, parser of Ingress rules, and forwarder of traffic to the backend Service is the Ingress Controller.  
Therefore, if there is no deployed and functional Ingress Controller in the cluster, or if the Ingress is not taken over by the corresponding Controller, then even if the Ingress YAML is created successfully, the business entry point may still be unreachable.

This answer suggests remembering these key terms:

- Rule object
- Does not listen on ports
- Does not directly forward traffic
- The true executor is the Controller
- Without a Controller, rules will not take effect

---

## 20. The most important conclusions from this article

### 1. Ingress is not a proxy program, but a rule object
It only describes how incoming traffic "should be forwarded."

### 2. The Ingress Controller is the program that truly executes the rules
It is responsible for:

- Listening for Ingress resources
- Receiving external requests
- Matching rules
- Forwarding to the Service

### 3. Clients access the Controller's entry point, not the Ingress object
The Controller is where incoming traffic truly first arrives.

### 4. The Controller itself also needs to be deployed and exposed
It typically also has:

- Pod
- Service
- External entry method

### 5. Without a Controller, Ingress often does not take effect
This is a very common issue and a key point often overlooked when starting with Ingress.

### 6. Understand the configuration relationship and request traffic perspective separately
Configuration relationship:

    Ingress -> Service -> Pod

Real traffic flow:

    Client -> Controller entry point -> Ingress rule matching -> Service -> Pod

---

## 21. Summary of the stage

The most important thing from this article is not remembering a specific YAML field, but thoroughly understanding this layer:

**Ingress is just a rule, and the Ingress Controller is the entry program that executes the rule.**

In other words, creating an Ingress does not automatically make the entry available.  
Only when there is a working Ingress Controller in the cluster and it truly takes over your Ingress rules and can be accessed by external requests does this seven-layer entry chain truly establish.

So by now, you should already be able to answer these two questions:

### 1. Why is just Ingress YAML not enough
Because it's just a rule declaration, without an executor.

### 2. Who does external traffic first reach
It first reaches the Ingress Controller's entry point, then the Controller forwards it to the Service based on the Ingress rules.

Once these two layers are thoroughly understood, subsequent learning will be much smoother:

- ingress-nginx example deployment
- First Ingress practical implementation
- Troubleshooting Ingress issues, 404, 502 errors

---

## Key Terms for Quick Recall

- Ingress: Kubernetes seven-layer entry rule object
- Ingress Controller: Controller program that executes Ingress rules
- ingress-nginx: One of the most common Ingress Controller implementations
- Rule declaration: Ingress only describes "how to forward"
- Executor: The Controller is the one that truly receives and forwards requests
- Controller entry point: Where client requests first arrive
- IngressClass: Which Controller the Ingress rule belongs to
- Configuration relationship: Ingress -> Service -> Pod
- Request traffic: Client -> Controller entry point -> Ingress rule matching -> Service -> Pod

---

## Operational Perspective Understanding

From an operational perspective, the Ingress Controller is the key bridge that turns "entry rules" into "operational entry capabilities."

If there is only an Ingress object and no Controller, the system just has an additional rule configuration;  
Only when the Controller exists and is functioning properly will this rule truly become:

- An entry point that can receive requests
- A proxy layer that can route based on domain and path
- A seven-layer entry layer with extensible TLS, logging, access control, and unified governance capabilities

Therefore, many Ingress issues are not due to the business application being down, but rather:

**The entry control layer has not been established, taken over, exposed, or correctly accessed.**

---

## References
- Kubernetes Ingress  
  https://kubernetes.io/docs/concepts/services-networking/ingress/
- Ingress Controllers  
  https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/
- ingress-nginx official documentation  
  https://kubernetes.github.io/ingress-nginx/

---

## Tomorrow's Suggestion
Next article suggestion:  
[[04-ingress-nginx Installation and Ingress Preparation - Controller Installation and NodePort Exposure for kubeadm Cluster]]