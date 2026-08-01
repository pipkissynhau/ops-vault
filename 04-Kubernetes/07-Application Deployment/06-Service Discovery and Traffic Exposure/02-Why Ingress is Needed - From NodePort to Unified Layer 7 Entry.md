# 02-Why Ingress is Needed: From NodePort to a Unified Layer 7 Entry

## Documentation Notes
- Document positioning: In the Kubernetes service discovery and traffic exposure phase, this is the motivation chapter for learning Ingress
- Applicable phase: After understanding Deployment, Service, NodePort, selector, Endpoints basics, entering the unified entry governance awareness
- Recommended path: `04-Kubernetes/07-Apply deployment/06-Service detection and traffic exposure/02-Why? Ingress: From NodePort To the seven-level entrance..md`

## Tags
#Kubernetes #Ingress #Service #NodePort #7thFloorForward #TrafficEntrance #ReverseAgent #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## One: Why Continue Learning Ingress After Service and NodePort

Previously in the application deployment mainline, we usually already established this basic access chain:

    Client
    -> Access NodeIP:NodePort
    -> Service
    -> Pod
    -> Container process

This chain already allows external cluster access to internal cluster applications, so many people will raise a question when learning here:

**If Service and NodePort can already access the business, why introduce Ingress?**

The answer is:

**Service and NodePort solve the "can we access" problem, while Ingress solves the "how to access in a unified, standardized, and manageable way like a formal business system" problem.**

In other words:

- Service solves service abstraction and cluster internal access issues
- NodePort solves basic external cluster access issues
- Ingress solves the unified entry point issue for HTTP/HTTPS business

Therefore, Ingress is not to replace the previous content learned, but on the basis of the previous content, further advance the entry capabilities to a more production-ready level.

---

## Two: Recap - What Did Service and NodePort Actually Solve

To understand Ingress, we must first know the boundaries of the previous capabilities.

### 1. What Does Deployment Solve
Deployment is responsible for:

- Managing stateless Pod replicas
- Ensuring replica count
- Supporting rolling updates
- Supporting Pod reconstruction after anomalies

### 2. What Does Service Solve
Service is responsible for:

- Providing a stable access entry for a group of Pods
- Shielding Pod IP changes
- Associating backend Pods through selector
- Routing traffic to backend Pods through Endpoints

In other words, Service's core value is:

**Making the fact that "backend Pods change" as transparent as possible to the caller.**

### 3. What Does NodePort Solve
If a Service is set to NodePort, it will also open a port on each node, for example:

    30080

At this point, external cluster can access through similar ways:

    http://NodesIP:30080

This transforms the "only accessible within the cluster" Service into an "entry point accessible externally".

---

## Three: NodePort Already Allows Access, Where is the Problem

NodePort is not unusable, but it is more suitable for:

- Teaching demonstrations
- Test environments
- Temporary verification
- Exposing a small number of services externally

Once entering scenarios closer to formal business, NodePort will quickly reveal obvious problems.

### 1. Unfriendly Access Methods
Common access methods for NodePort are:

    http://NodesIP:30080

Issues with this method include:

- Users need to remember node IP
- Users need to remember port number
- URLs lack business semantics
- Not conducive to formal external release

Real business prefers:

    http://www.example.com
    http://api.example.com
    http://app.example.com/admin

That is:

- Access via domain name
- Minimize user concern about ports
- Entry address more like a formal website or API service

### 2. Port Chaos When Multiple Services Are Exposed
Assume you now have multiple services:

- Frontend service
- API service
- Backend management service
- File service

If each service is exposed separately via NodePort, you may encounter:

- Frontend: 30080
- API: 30081
- Management backend: 30082
- File service: 30083

This will bring several issues:

- Increasing number of ports
- Difficult to manage uniformly
- Hard to form a standardized entry point
- Unfavorable for unified governance and troubleshooting

### 3. NodePort is More Suitable for Four-Layer Entry, Not Complex HTTP Routing
The essence of NodePort's approach is more akin to:

- Opening a port
- Routing traffic to a certain Service

However, for many web businesses, the demand is often not "changing a port," but:

- Different domains route to different backends
- Different paths under the same domain route to different services
- Unified termination of HTTPS certificates
- Entry point with enhanced capabilities like redirection, rate limiting, and authentication

These requirements are no longer just "opening a port," but more towards:

**Traffic distribution based on HTTP/HTTPS rules.**

### 4. NodePort is Unfavorable for Forming a Unified Entry Mindset
If all your businesses are exposed through different NodePorts, the entry layer will become:

- A bunch of node IPs
- A bunch of port numbers
- A bunch of manually remembered relationships

This is unfavorable for forming:

- Unified entry
- Unified domain
- Unified certificate
- Unified governance
- Unified troubleshooting

Therefore, the problem with NodePort is not "unable to access," but:

**It can expose the business, but it's not elegant and not suitable for formal business entry governance.**

---

## Four: What Exactly Does Ingress Solve

Ingress's core goal can be summarized in one sentence:

**Use domain names, paths, and other layer 7 rules to uniformly forward external HTTP/HTTPS traffic to different Services within the cluster.**

Here are several key terms that must be grasped.

### 1. External HTTP/HTTPS Traffic
Ingress mainly targets:

- HTTP
- HTTPS

That is, web websites, frontend/backend interfaces, management backends, and microservice gateway frontends.

### 2. Unified Entry
Ingress's approach is not to open a port for each service, but to let multiple services share a unified entry layer.

### 3. Domain Rules
Ingress can differentiate traffic based on domains, for example:

    app.example.com
    api.example.com
    admin.example.com

Different domains can be forwarded to different Services.

### 4. Path Rules
Ingress can also differentiate traffic based on paths, for example:

    /app
    /api
    /admin

Under the same domain, different paths can be forwarded to different services.

### 5. Backend is Usually Still a Service
Ingress generally does not directly use a bunch of Pod IPs as the business entry point, but usually routes traffic first to a Service, then to the Pod.

So the typical chain remains:

    Client
    -> Ingress
    -> Service
    -> Pod

---

## Five: What Does "Layer 7 Entry" or "Layer 7 Forwarding" Mean

The easiest term to hear in Ingress learning is:

**Layer 7** /think

Here, the term "seven layers" typically refers to the application layer in the OSI model. In the daily context of Kubernetes, it can more directly be understood as:

**Rule processing for the HTTP/HTTPS layer.**

That is, Ingress isn't just looking at:

- What IP address it is
- What port it is

It also examines:

- What domain the request is accessing
- What request path it is
- Whether it's an HTTP/HTTPS request
- Whether TLS termination is needed

This is very different from NodePort.

### What is NodePort more like
NodePort is more like:

- Opening a port on a node
- Forwarding traffic received to a Service

It emphasizes more on:

- Port
- Node
- Basic connectivity

### What is Ingress more like
Ingress is more like:

- A unified web entry layer
- Routing requests by domain and path
- Making the entry more like a formal business website or API gateway frontend

### Simplified understanding
You can remember with this comparison:

- NodePort: More like "opening a port"
- Ingress: More like "routing by domain and path"

---

## Six. Why Ingress is closer to a formal business entry

In real business scenarios, users or callers typically don't want to access services like this:

    http://10.0.0.21:30080
    http://10.0.0.22:30081
    http://10.0.0.23:30082

But rather prefer:

    http://www.example.com
    http://api.example.com
    http://www.example.com/admin

This access method is closer to a formal business system because it has the following characteristics.

### 1. Unified domain semantics
The entry address is no longer just a node IP, but a domain with business meaning.

### 2. Unified entry layer
Multiple services can be managed under the same entry layer, rather than each service exposing its own port.

### 3. Better scalability
After unification, it's easier to integrate later:

- HTTPS
- Certificate management
- Forced redirection
- Whitelist
- Rate limiting
- Enhanced reverse proxy capabilities

### 4. More favorable for operations and governance
After unification, it's easier to converge on the entry layer's monitoring, logs, policies, troubleshooting, and release validation.

Thus, the value of Ingress isn't just "being able to access by domain," but it upgrades the entry from "being able to connect" to "being more manageable."

---

## Seven. What is the relationship between Ingress and Service

This point must be firmly understood.

Many people who first learn Ingress mistakenly think:

- With Ingress, you don't need Service
- Ingress can directly replace Service

This understanding is incorrect.

### 1. Service handles backend service abstraction
Service handles:

- Providing a stable entry for backend Pods
- Finding a group of Pods via selector
- Maintaining a list of backend instances via Endpoints
- Shielding Pod IP changes

### 2. Ingress handles entry rules
Ingress handles:

- Receiving external HTTP/HTTPS entry traffic
- Routing traffic based on domain and path rules
- Forwarding traffic to different Services

### 3. They are not alternatives, but upstream/downstream relationships
The typical access flow is usually:

    Client
    -> Ingress
    -> Service
    -> Pod

So a more accurate understanding should be:

- Service solves backend service access abstraction
- Ingress solves frontend unified entry rules

### 4. Why can't you bypass Service to directly access Pod
From an operations perspective, directly using Pod as a business entry has obvious issues:

- Pod may be rebuilt
- Pod IP may change
- Number of replicas may change
- Backend instance list needs dynamic maintenance

Service exists precisely to handle these changes.

Thus, Ingress typically relies on Service, rather than replacing it.

---

## Eight. A simple business scenario: What happens without Ingress

Assume you now have two services:

- Frontend website: frontend-svc
- API service: api-svc

### If only using NodePort
You might need to expose them like this:

- frontend-svc -> 30080
- api-svc -> 30081

Access methods might become:

    http://NodesIP:30080
    http://NodesIP:30081

You would encounter these issues:

- Users have to remember two different ports
- Not elegant for external publishing
- Difficult to unify HTTPS
- Difficult to unify entry rule governance

### If introducing Ingress
You can organize the entry into the following form:

    http://www.example.com       -> frontend-svc
    http://www.example.com/api   -> api-svc

Or:

    http://www.example.com       -> frontend-svc
    http://api.example.com       -> api-svc

The entry layer becomes clearer:

- Users only care about the domain
- Path or subdomain determines which service to access
- Multiple Services can be unified under one entry layer

This is the typical value of Ingress.

---

## Nine. In which scenarios is Ingress most suitable

Ingress isn't required for all scenarios, but the following are very suitable.

### 1. Web website entry
For example:

- Official website
- Management backend
- Frontend page application

### 2. API service entry
For example:

- Front-end and back-end separated projects
- Microservice API entry
- HTTP interfaces provided to external systems

### 3. Unified external entry for multiple services
For example, under the same domain:

- /web
- /api
- /admin

Forwarding to different Services.

### 4. Scenarios requiring HTTPS
For example:

- Unified TLS certificate access
- HTTPS termination at the entry layer

### 5. Scenarios requiring subsequent expansion and governance capabilities
For example:

- Redirection
- Whitelist
- Rate limiting
- Pre-authentication
- More standardized access log collection

---

## Ten. What problems is Ingress not suitable for solving

To avoid misunderstanding, it's also important to clarify the boundaries of Ingress.

### 1. Ingress is not a universal entry for all protocols
Ingress mainly targets HTTP/HTTPS seven-layer traffic.

If your business is:

- MySQL
- Redis
- TCP middleware
- UDP service

It may not be suitable to directly use Ingress as an entry.

### 2. Ingress is not the business program itself
Ingress is not your application, nor is it the Pod itself. It's just the entry rule layer.

### 3. Ingress is not automatically effective after creation
This is very critical.

Creating an Ingress YAML does not automatically mean traffic starts forwarding. It usually still depends on the actual execution of these rules, which is the Ingress Controller to be focused on later:

**Ingress Controller**

So for now, just understand Ingress as:

**Entry rule layer**

---

## Eleven. Why this article emphasizes understanding "roles" first, not rushing to memorize YAML

Many people learn Ingress by immediately looking at YAML, for example:

- host
- path
- backend
- pathType /think

But if you don't first understand "Why Ingress is needed," these fields can easily become rote memorization.

The most important thing here isn't memorizing configurations—it's building the following key understandings:

### 1. Service Already Solves Basic Service Access  
### 2. NodePort Already Solves the Most Basic External Access  
### 3. But NodePort Is Not Suitable for Formal Business Unified Entry  
### 4. Ingress Emerged to Address HTTP/HTTPS Unified Entry Governance  
### 5. Ingress Typically Adds an Extra Layer of Entry Rules in Front of Service  

As long as these role relationships are clear, you won't feel lost when looking at specific YAML, Controller, or troubleshooting later.

---

## Twelve. From an Operations Perspective, Why Must You Understand Ingress?

For operations or cloud-native roles, Ingress is not an optional topic—it's a highly frequent entry theme.

Because in actual work, you often encounter issues like:

- Domain access is not working  
- Business can only access via NodePort, but not via domain  
- Frontend pages can open, but API returns 404  
- Entry rules are configured, but backend has no traffic  
- HTTPS is not properly configured, causing certificate or redirect anomalies  
- Backend Pod is normal, but business entry still fails  

If you lack understanding of Ingress's role, you're likely to blame:

- Network issues  
- Kubernetes issues  
- Service issues  
- Domain issues  

In reality, the problems often stem from:

- Unclear understanding of entry rules  
- No unified entry layer  
- Layer 7 routing not matching  
- Unclear understanding of the responsibilities boundary between Ingress and Service  

Therefore, although this article doesn't dive into deep configuration, it's a prerequisite for further understanding:

- Ingress Controller  
- Ingress YAML  
- Ingress Troubleshooting  

---

## Thirteen. How to Describe "Why Ingress Is Needed" in an Interview

If the interviewer asks:

**Why do we need Ingress when we already have Service/NodePort?**

You can answer according to the following logic:

Ingress mainly solves the problem of unified entry for HTTP/HTTPS business.  
Service provides stable access abstraction for backend Pods, and NodePort allows external access to services via node ports, but NodePort is more suitable for basic testing or simple exposure.  
In formal business, it's usually preferred to manage multiple service entries through domains, paths, and HTTPS, rather than exposing each service via a node port.  
Thus, Ingress essentially adds a layer of Layer 7 entry rules in front of Service, allowing multiple HTTP/HTTPS services to be distributed via unified entry through domains and paths.

This passage doesn't need to be memorized verbatim, but the following keywords are recommended to remember:

- Unified entry point  
- HTTP/HTTPS  
- Domain and path distribution  
- Add a layer of rules in front of Service  
- More suitable for formal business entry  

---

## Fourteen. The Most Important Conclusions of This Article

### 1. NodePort Solves "External Access from Cluster"  
But it's not elegant and unsuitable for large-scale formal business exposure.

### 2. Ingress Solves "How to Manage HTTP/HTTPS Entry More Standardly"  
It emphasizes:

- Domain  
- Path  
- Layer 7 rules  
- Unified entry  

### 3. Ingress Is Not a Replacement for Service  
The standard chain is usually still:

    Client  
    -> Ingress  
    -> Service  
    -> Pod  

### 4. Ingress Is Closer to Formal Business System Entry  
It upgrades the entry from "Node IP + Port" to "Domain + Path + Unified Governance."

### 5. This Article Focuses on Roles, Next Article Enters Controller  
At this stage, the most important thing is to clearly understand "Why Ingress Is Needed," rather than memorizing all fields urgently.

---

## Fifteen. Stage Summary

Before learning Ingress, you must complete a cognitive shift:

**Kubernetes exposing services externally isn't just about making business "reachable," but also considering whether the entry is unified, standardized, and suitable for governance.**

NodePort can already expose business, but it's more suitable as:

- Learning entry  
- Temporary testing entry  
- Simple environment entry  

Ingress is more suitable as:

- Unified entry for HTTP/HTTPS business  
- Domain and path forwarding entry  
- Layer 7 entry layer closer to formal business systems  

Thus, the core of this article isn't configuration details, but answering a question:

**Why continue learning Ingress after already learning Service and NodePort?**

The answer is:

**Because formal business entry needs to not only be reachable but also resemble a manageable, scalable, and unified system entry.**

---

## Keyword Quick Notes

- Service: Stable access abstraction for backend Pods  
- NodePort: Exposes Service to each node port  
- Ingress: Layer 7 entry rules for HTTP/HTTPS  
- Unified Entry: Multiple services share one external entry layer  
- Domain Forwarding: Traffic diversion by Host  
- Path Forwarding: Traffic diversion by Path  
- Layer 7 Forwarding: Traffic distribution based on HTTP/HTTPS rules  
- Formal Business Entry: More standardized external access method than NodePort  

---

## Operational Extended Understanding

From an operations perspective, Ingress's true value isn't just "making URLs look better," but the entry layer evolving from "pure service exposure" to "unified service governance."

If only NodePort is used, entry management can easily become:

- Port table  
- Node table  
- Manual memory table  

Introducing Ingress brings entry to possess:

- Domain semantics  
- Path semantics  
- Unified entry layer  
- Better scalability  
- Stronger governance foundation  

This is why Ingress is often a key step from "business can run" to "business resembling a formal production system" in cloud-native application deployments.

---

## References
- Kubernetes Services, Load Balancing, and Networking  
  https://kubernetes.io/docs/concepts/services-networking/
- Kubernetes Ingress  
  https://kubernetes.io/docs/concepts/services-networking/ingress/
- Ingress Controllers  
  https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

---

## Next Day Suggestions
Next article suggests organizing:

[[03-Ingress Controller Basics - Why Only an Ingress YAML Isn't Enough]]