# 03-Nginx Exposing Static Pages to the Outside: NodePort Basics Practice

## Document Notes
- Document Positioning: First step for stateless applications to access externally
- Applicable Stage: After completing Deployment and Service basics practice for external exposure introduction
- Recommended Path: `04-Kubernetes/07-Apply deployment/02-No status application deployment/02-Nginx Static page exposure:NodePort Basic practice`

## Tags
#Kubernetes #NodePort #Service #Nginx #StaticPage #NoStatusApplication #ApplyDeployment #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## I. Why Continue Learning NodePort

The previous article completed:

- Deployment creates Pod
- Service selects backend Pod via labels
- ClusterIP provides stable access entry within the cluster

But ClusterIP has an obvious characteristic:

**Default only suitable for internal cluster access.**

This means:

- Pod internally can access
- Other services within the cluster can access
- But external users typically cannot directly access

If you want to directly access a simplest test business from outside the cluster, NodePort is the easiest way to start with.

Therefore, the significance of learning NodePort is:

> **Expose the cluster internal Service further to the outside, establishing the first link for external access to Kubernetes applications.**

---

## II. What Problem Does NodePort Solve

NodePort solves:

**How to allow external clients to access cluster internal Service through node IP and fixed port.**

It can be simply understood as:

- ClusterIP: Only accessible within the cluster
- NodePort: Maps Service to a port on each node
- External users access the business through `NodesIP:NodePort`

This is the first step from "internal cluster access" to "external cluster access."

---

## III. Basic Access Chain of NodePort

In ClusterIP scenario, the access chain is usually:

**Client (internal cluster) → Service → Pod → Container port**

In NodePort scenario, the access chain becomes:

**Client (external) → NodeIP:NodePort → Service → Pod → Container port**

The core change here is:

- Service is no longer just an internal entry
- It also occupies a port on each node
- External users can access through node IP and this port

---

## IV. What is NodePort

NodePort is a type of Service.

When Service type is set to:

    type: NodePort

Kubernetes will do two things:

### 1. Continue creating a ClusterIP
That is, NodePort is not replacing ClusterIP, but adding external access capability on its basis.

### 2. Open a port on each node
This port is called NodePort.

External access can reach backend Pod through any node's IP and this port.

---

## V. When is NodePort Most Suitable

NodePort is typically suitable for the following scenarios:

### 1. Test environment or experimental environment
For example, local cluster, self-built test cluster, learning environment.

### 2. Temporary external verification service
For example, temporarily viewing pages, verifying interfaces.

### 3. Learning Kubernetes Service external exposure mechanism
It is the foundation for understanding more complex exposure methods like Ingress, LoadBalancer, etc.

---

## VI. When is NodePort Not Suitable

NodePort has obvious limitations.

### 1. Not elegant
Access method is usually:

    http://NodesIP:NodePort

Not suitable for formal business entry.

### 2. Port exposure is rough
Each service occupies a node port, management can easily become chaotic.

### 3. Not suitable for unified external service entry
In production, it's usually preferred to use:

- LoadBalancer
- Ingress
- Gateway API

### 4. Security control is relatively coarse
If nodes directly expose NodePort, firewall and access control need to be combined.

---

## VII. A Simplest NodePort Service Example for Nginx

Below is a NodePort Service example for Nginx static page:

    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-web-svc
    spec:
      selector:
        app: nginx-web
      type: NodePort
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
          nodePort: 30080

---

## VIII. How to Understand This YAML

### 1. `type: NodePort`
Indicates this Service should provide access externally via NodePort.

### 2. `port: 80`
Indicates the Service's internal service port is 80.

### 3. `targetPort: 80`
Indicates the backend Pod container's actual listening port is 80.

### 4. `nodePort: 30080`
Indicates an additional 30080 port is opened on each node.

External access generally uses:

    http://NodesIP:30080

Traffic will be forwarded to:

- Service 80
- Pod 80
- Nginx container 80

---

## IX. How to Distinguish `port`, `targetPort`, `nodePort`

This is a high-frequency focus in NodePort learning.

### 1. `targetPort`
Indicates:

**The port where the container in the backend Pod truly listens.**

For example, Nginx container listens on 80.

### 2. `port`
Indicates:

**The port the Service provides internally.**

When accessing via Service name internally, this port is usually used.

### 3. `nodePort`
Indicates:

**The port exposed externally by the node.**

When external clients access, this port is usually used.

### Simplified Memory
- `targetPort`: Pod's port
- `port`: Service's port
- `nodePort`: Node's external port

---

## X. Why Access is to Node IP, Not Pod IP

Because Pod IP has several inherent issues:

- Pod may be rebuilt
- Pod IP may change
- Pod is not suitable as a stable external entry

The NodePort approach is:

- External does not directly find Pod
- External first hits the node
- The node forwards traffic to backend Pod via Service rules

Thus, the core of NodePort is not "opening Pod", but: /think

**Extending Service Capabilities to the Cluster's External Through Node Ports.**

---

## 11. On Which Nodes Does NodePort Open

A crucial understanding point is:

> **NodePort is typically accessible on every node.**

This means, if your cluster has multiple nodes, theoretically, you can access the service through any reachable node's IP plus NodePort.

For example:

- `http://NodesA:30080`
- `http://NodesB:30080`
- `http://NodesC:30080`

All may be forwarded to the backend Pod.

### Operations Understanding Focus
NodePort does not require the business Pod to run on the accessed node.  
As long as the Service rules are normal, the node can continue forwarding traffic to Pods on other nodes.

---

## 12. Why Is the Port Range for NodePort Important

In Kubernetes, NodePort generally uses a specific port range, with a common default range being:

    30000-32767

### Why Is This Range Important

#### 1. Not All Ports Can Be Directly Set as nodePort
If it exceeds the allowed range, creation may fail.

#### 2. Avoid Conflicts with Existing NodePorts
The same cluster cannot reuse the same NodePort.

#### 3. Coordinate with Firewall Policies
If external access is restricted, even if NodePort is configured, it may still be inaccessible.

---

## 13. What Essentially Happens When Changing from ClusterIP to NodePort

You can understand this change as:

### Originally Only Cluster Internal Entry
The Service could only be accessed internally within the cluster.

### Now with an Additional Node-Level Entry
Kubernetes opens a port on each node, allowing external traffic to first enter the node and then be forwarded to the Service and Pod.

In other words, NodePort is not a replacement for Service, but rather:

**Adds a node-level entry on top of the Service.**

---

## 14. How to Understand a Minimal Complete Access Chain

Taking an Nginx page example, the complete chain in a NodePort scenario can be understood as:

### 1. Browser Accesses Node IP and NodePort
For example:

    http://192.168.1.100:30080

### 2. Node Receives the Request
The node's NodePort rule catches this traffic.

### 3. Traffic is Forwarded to the Service
The Service finds the backend Pod based on the selector.

### 4. Service Forwards to the Target Pod
Then, through `targetPort`, the traffic is sent to the Nginx container's 80 port.

### 5. Nginx Returns the Page
Finally, the browser sees your static page.

---

## 15. Do You Need to Modify Deployment at This Point

Usually not.

The reason is:

- Deployment is responsible for creating and maintaining Pods
- NodePort is just a change in the Service's external exposure method
- The Pod's runtime logic remains unchanged

### In Other Words
If your Deployment was previously working normally, to add external access capabilities, you typically only need to adjust the Service type.

This is a crucial layered understanding:

- Deployment manages "application runtime"
- Service manages "service access"
- NodePort is a "external exposure method" for Service

---

## 16. What Should You Check First After Deployment

After NodePort is configured, it's recommended to check the following in order.

### 1. Check if Deployment is Normal
Focus on:

- Whether Pod is Running
- Whether Pod is Ready
- Whether the image is pulled normally

### 2. Check if Service is Normal
Focus on:

- Whether the type is NodePort
- `port`, `targetPort`, `nodePort` are correct

### 3. Check if Endpoints Exist
Focus on:

- Whether the Service has correctly selected backend Pods

### 4. Check Node Network Accessibility
Focus on:

- Whether the node IP is accessible from the outside
- Whether the node firewall allows NodePort

---

## 17. Why Can't External Access Be Reached Even Though NodePort Is Configured

This is a highly frequent issue.

### 1. Node IP Is Not Accessible
For example:

- Node is in the internal network
- Local machine cannot access the node's network segment
- Cloud security group is not open
- Routing is not available

### 2. Node Firewall Is Not Open for NodePort
For example, the node's system firewall blocks 30080.

### 3. Service Has Not Selected Pod
Typically manifested as:

- NodePort can connect to the node
- But there is no actual business response

### 4. Pod Is Not Actually Listening on the Port
For example, the container hasn't started properly, or Nginx isn't actually listening on 80.

### 5. NodePort Is Wrong or Conflicting
Port is not in the allowed range, or it's occupied by another service.

---

## 18. What Should You Check First When Troubleshooting NodePort

It's recommended to troubleshoot in this order.

### 1. Check Pod Health First
If the Pod hasn't started, there's no need to check further.

### 2. Check if Service Has Selected Pod
This is the key to determining whether the Service can actually forward traffic.

### 3. Check if NodePort Was Created Successfully
Confirm that this Service has indeed been assigned a NodePort.

### 4. Check Node Network and Firewall
Often, business unavailability is not due to Kubernetes configuration errors, but because the node access chain hasn't been opened.

### 5. Finally Check Application Logs
Confirm that Nginx has indeed started normally and returned content.

---

## 19. What Similarities Are There Between NodePort and Docker's Local Port Mapping

They share a very similar aspect:

### Docker Local Run
For example:

    -p 8080:80

Means:

- Host 8080
- Mapped to container 80

### Kubernetes NodePort
For example:

- Node 30080
- Forwarded to Service 80
- Then to Pod 80

### Similarity
Essentially both are:

**External port → Internal service port**

### Difference
Docker is more about direct container mapping.  
NodePort involves forwarding through Kubernetes Service rules.

---

## 20. How to Understand the Relationship Between NodePort and Ingress

NodePort is not Ingress, but it's an important prerequisite for understanding Ingress.

### NodePort Characteristics
- Accessed via node IP and port
- Exposed in a more direct way
- More suitable for testing or basic practice

### Ingress Characteristics
- More suitable for HTTP/HTTPS seven-layer traffic
- More suitable for domain access
- More suitable for a unified entry point for multiple services

### Learning Order Recommendation
Learn first:

- ClusterIP
- NodePort

Then learn:

- Ingress
- Domain routing
- Seven-layer forwarding

This understanding will be smoother.

---

## 21. The Most Important Recognitions in This Practice /think

### 1. NodePort is a type of Service  
It is not a new object, but rather a method of exposing a Service.

### 2. NodePort does not replace ClusterIP  
It adds node-facing entry points on top of ClusterIP.

### 3. External access uses node IP and nodePort  
Instead of directly accessing Pod IP.

### 4. Service still selects Pods as core functionality  
Even if NodePort is configured, mismatched selector will block traffic.

### 5. External access path is affected by node network and firewall  
This is a common issue in real environments.

---

## Twenty-two, What level should you master this section?

At this stage, it's recommended to reach the following level:

### 1. Understand why NodePort allows external access to cluster services  
### 2. Distinguish `port`, `targetPort`, `nodePort`  
### 3. Understand the external access path: node → Service → Pod  
### 4. Understand why NodePort still depends on Service selector  
### 5. Perform basic troubleshooting for "NodePort configured but inaccessible"

---

## Twenty-three, Common interview follow-up questions

Common questions in this area include:

- What's the difference between ClusterIP and NodePort  
- Why does NodePort allow external access to cluster services  
- What's the difference between `nodePort`, `port`, `targetPort`  
- Why can't you access services even if Service is NodePort type  
- What scenarios is NodePort suitable for  
- Why is NodePort not ideal as a formal production entry point  
- What's the relationship between NodePort and Ingress  
- On which nodes does NodePort open

---

## Twenty-four, Stage summary

NodePort is one of the most fundamental ways to expose services externally in Kubernetes.

The most important takeaway from this section is not memorizing YAML, but establishing these core understandings:

- NodePort is a type of Service  
- It adds node port entry on top of ClusterIP  
- External access uses node IP and nodePort  
- Service still finds backend Pods via selector  
- When business fails, check both Kubernetes object relationships and node network/firewall

Understanding these relationships clearly will make future learning about Ingress, domain access, and layer-7 routing much easier.

---

## Twenty-five, Keyword quick notes

- NodePort: A type of Service for external exposure  
- nodePort: The externally open port on the node  
- port: The service port provided by Service  
- targetPort: The container listening port of backend Pod  
- Node IP: External client access entry address  
- ClusterIP: Cluster internal access entry  
- Endpoints: The set of actual backend addresses selected by Service

---

## Twenty-six, Operations perspective understanding

From an operations perspective, NodePort practice is simple but starts introducing the basic thinking of "platform external exposure capability."

At this stage, you need to gradually build these troubleshooting awareness:

- Is the application healthy  
- Is Service configured correctly  
- Are Endpoints present  
- Is node network reachable  
- Are firewall/security group rules allowing access  
- Is the access path hitting the correct node port

This awareness will be reused when learning Ingress, cloud load balancing, and API gateways later.

---

## References  
- Kubernetes Service: https://kubernetes.io/docs/concepts/services-networking/service/  
- Kubernetes Service Types: https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types  
- Kubernetes Labels and Selectors: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/  

---

## Next day suggestion  
Next article suggestion to organize:

[[04-Nginx Static Page Update Practice - Deployment Replica Scaling and Rolling Updates]]