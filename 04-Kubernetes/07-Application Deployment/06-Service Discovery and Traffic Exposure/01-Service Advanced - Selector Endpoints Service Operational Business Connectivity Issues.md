# 01-Service Advanced: selector, Endpoints, and Service Normal but Business Unreachable

## Document Notes
- Document Positioning: Service Access Chain Advanced and Basic Troubleshooting Precondition
- Applicable Stage: After completing Deployment, Service, NodePort, Probe, and Resource Management Basics, enter Service Pod Selection and Business Unreachable Scenario Understanding
- Recommended Path: `04-Kubernetes/07-Apply deployment/06-Service detection and traffic exposure/01-Service Progress:selectorI don't know.Endpoints and Service Normal but non-operational basis`

## Tags
#Kubernetes #Service #selector #Endpoints #NodePort #ClusterIP #ServiceDiscovery #FlowForward #ApplyDeployment #GroundsForExclusion #OperationalContainerization #Clouds. #Transport #InterviewNotes

---

## I. Why We Need to Organize Service Advanced Topics Now

In the previous mainline, you've already encountered:

- Deployment creates Pod
- Service provides stable access entry
- NodePort exposes externally
- Pod's Ready status and Probe
- requests/limits and OOMKilled

At this point, many people form an initial impression:

> **If Service is created successfully, business should be reachable.**

But in real environments, a highly frequent type of issue is:

- Service object exists
- Ports are written
- Pods appear to be Running
- Even NodePort is opened
- But business remains unreachable

This type of issue is extremely common in interviews and production environments.

And their core often revolves around these points:

- selector
- labels
- Endpoints
- Pod Ready status
- targetPort
- Application's actual listening port

So the goal of this article is to thoroughly explain:

> **Why Service being normal doesn't mean business is necessarily normal.**

---

## II. What Exactly Does Service Do

Let's return to the most fundamental level.

Service's core function isn't "running applications," but rather:

> **Provides a stable access entry for a group of Pods that meet certain conditions.**

Because Pods have several inherent characteristics:

- Pod IP changes
- Pods may be deleted and rebuilt
- Pods aren't suitable as long-term fixed entry points

So Kubernetes designed Service to accomplish two things:

### 1. Provide a stable external entry
For example:

- A fixed ClusterIP
- A fixed Service name
- Or a NodePort

### 2. Forward traffic to a group of backend Pods
This group of Pods is associated through selector and labels.

### Operations Understanding Focus
Service is essentially not "the service program itself," but rather:

> **Traffic entry point + backend Pod selection mechanism.**

---

## III. What is selector

selector can be understood as:

> **The condition Service uses to select backend Pods.**

For example:

    selector:
      app: nginx-web

This means:

- This Service will find all Pods with `app=nginx-web` label
- Then use them as its backend

### Operations Understanding Focus
selector doesn't create Pods, it only:

> **Finds Pods.**

This is why when selector is configured incorrectly, Service appears "normal" but actually has no backend.

---

## IV. What are labels

labels are tags on resource objects.

Pods commonly have some labels, for example:

    labels:
      app: nginx-web
      tier: frontend

One of their purposes is to be matched by selector.

### You can understand it this way
- Pods have tags
- Service uses selector to find Pods with corresponding tags

### The Most Critical Relationship
If:

- Pod label = `app: nginx-web`
- Service selector = `app: nginx-web`

Then Service can correctly select this Pod.

---

## V. Why selector and labels are so important

Because Service and Pods are not bound by names or automatically matched by "looking similar."

Their most core association method is:

> **Label matching.**

So if this relationship fails, it's easy to encounter:

- Service has been created successfully
- Pods are present
- But Service actually has no backend Pods

### Operations Understanding Focus
This is a very typical "objects are normal but the chain isn't connected" issue in Kubernetes.

---

## VI. What is Endpoints

Endpoints can be understood as:

> **The actual backend address collection Service currently selects.**

In other words, Service doesn't "select whoever it wants," but ultimately forms an actual backend list, which is usually Endpoints.

### You can understand it this way

#### selector is "filtering condition"
For example:

- `app=nginx-web`

#### Endpoints is "filtering result"
For example, the final selected:

- PodA's IP:80
- PodB's IP:80
- PodC's IP:80

### Operations Understanding Focus
Whether Service has backend depends not only on Service object existence, but also:

> **Whether Endpoints actually contains usable addresses.**

---

## VII. Why Service being normal doesn't mean business is normal

This is the core issue of the entire article.

### Service being normal usually only means:
- YAML was created successfully
- Service object exists
- Port field was written

But this is far from sufficient.

Because business connectivity depends at least on these conditions:

### 1. selector must select Pods
### 2. Pods must be Ready
### 3. Endpoints must be generated
### 4. targetPort must match actual listening port
### 5. Application must actually work on target port
### 6. NodePort/Ingress/external network chain must be normal

Any layer failing can manifest as:

> **Service exists, but business is unreachable.**

---

## VIII. A Basic Service Example

Below is a basic Service example: /think

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

## IX. How to Understand This Service YAML

### 1. `metadata.name`
The Service name is:

    nginx-web-svc

### 2. `selector`
It indicates it will look for:

    app=nginx-web

Pods.

### 3. `port: 80`
It indicates the Service itself provides access via port 80.

### 4. `targetPort: 80`
It indicates traffic is ultimately forwarded to the Pod container's port 80.

### 5. `type: ClusterIP`
It indicates it defaults to being accessible only within the cluster.

### Operations Understanding Focus
This configuration only describes "how to route," but does not automatically guarantee "the backend exists."

---

## X. Why Making the selector Correct is the First Step, Not the Last

Many people think:

- selector is correct
- it's done

Actually, making the selector correct only means:

> **The Service has a chance to find Pods.**

But whether it can actually forward business traffic still needs to be checked:

### 1. Whether the Pod actually has this label
### 2. Whether the Pod is in Ready state
### 3. Whether the Pod actually listens on targetPort
### 4. Whether the service inside the container is actually available

So the selector is just an entry point, not the final answer.

---

## XI. Why Endpoints Might Still Be Missing Even If Pod is Running

This is directly related to readinessProbe.

### If the Pod is not Ready
Even if it's already Running, Kubernetes might not include it in the Service's available backend list.

### Common Phenomenon
- Pod status looks "alive"
- But it's missing from Endpoints
- Service backend is empty or incorrect
- Business access fails

### Operations Understanding Focus
This is why the previous Probe articles are important prerequisites for Service troubleshooting.

You must first understand:

> **Running does not equal Ready, Ready is closer to "being able to accept traffic."**

---

## XII. What Does "Service Exists But No Backend" Mean

This phenomenon is very common.

### Surface View
- Service exists
- ClusterIP is available
- NodePort is also available
- Object creation has no errors

### Actually
- selector didn't match any Pod
- Or matched, but Pod is not Ready
- So Endpoints is empty

### Final Effect
From the access perspective, business is unreachable.

### Operations Understanding Focus
The essence of this issue is:

> **Traffic entry exists, but backend targets are empty.**

---

## XIII. What Problems Can TargetPort Being Wrong Cause

This is another highly frequent issue.

### Example
- Pod container actually listens on 8080
- Service is written as `targetPort: 80`

What happens?

### Surface View
- selector is fine
- Endpoints may have Pods
- Service object is normal

### Actually
Traffic is forwarded to a port on the Pod that is not listening.

### Result
- Business is unreachable
- Connection fails
- Timeout or connection refused

### Operations Understanding Focus
This shows:

> **Endpoints not being empty does not guarantee business is reachable.**
You still need to confirm port chain consistency.

---

## XIV. How to Reinforce Understanding of port and targetPort

This point must be thoroughly understood.

### `port`
Indicates:

> **The port the Service itself provides externally.**

### `targetPort`
Indicates:

> **The port the Pod container actually receives traffic on.**

### Simplified Memory
- `port`: The port that hits the Service
- `targetPort`: The port that hits the Pod

### Critical for Troubleshooting
Many "Service is normal but business is unreachable" issues ultimately stem from:

- `targetPort` not matching the actual listening port of the application

---

## XV. A Typical Error Example

Assume Nginx in the Pod actually listens on:

- port 80

But the Service is written as:

    ports:
      - port: 80
        targetPort: 8080

Then when accessed:

- Client accesses Service:80
- Service forwards to Pod:8080
- But Nginx is not listening on 8080
- Business is unreachable

### Operations Understanding Focus
This kind of issue is particularly suitable for interviews to test "whether you can trace the chain of issues."

---

## XVI. How to Visualize Service Access Chain in Your Mind

It's recommended to form a fixed access chain:

### Cluster Internal Access
**Client → Service:port → Pod:targetPort → Container's actual listening port**

### NodePort External Access
**Client → NodeIP:nodePort → Service:port → Pod:targetPort → Container port**

### Ingress Scenario
**Client → Ingress → Service:port → Pod:targetPort → Container port**

### Operations Understanding Focus
If any layer in the chain doesn't match, business may appear "unreachable."

---

## XVII. Why You Must Check Endpoints When Troubleshooting Service

Because Endpoints is a critical "intermediate state evidence."

### If Endpoints is empty
The issue is likely in:

- selector
- labels
- Pod Ready status

### If Endpoints is not empty
It indicates at least:

- Service has selected backend Pods

The issue is more likely in:

- targetPort
- Application port listening
- Container internal service status
- Upper-layer network access chain

### Operations Understanding Focus
So Endpoints is like a watershed, helping you quickly narrow down the troubleshooting scope.

---

## XVIII. Why "Service is Normal But Business is Unreachable" is a Common Troubleshooting Question

Because it particularly tests whether someone only "looks at object existence" or can truly trace the chain.

It usually involves:

- Kubernetes object relationships
- Label matching
- Pod Ready concept
- Service and Pod port mapping
- Application actual listening status
- Sometimes also involves network and Ingress

This is a very typical type of comprehensive question.

---

## 19. How to Establish a Basic Troubleshooting Order

It is recommended to form a fixed order now.

### 1. First check if the Pod exists and is normal
Focus on:

- Running
- Ready
- Whether there are restarts

### 2. Then check if the Service selector is correct
Confirm whether it matches the Pod labels.

### 3. Then check if Endpoints are generated
If there is no backend, don't rush to suspect the application.

### 4. Then check if targetPort matches the application's listening port
This is a common issue.

### 5. Then check if the application itself is truly accessible
Confirm whether the program inside the container is working normally.

### 6. Finally check the external link
If it's a NodePort / Ingress scenario, continue to check nodes, ingress, network policies, and firewalls.

---

## 20. How to Understand a Common Troubleshooting Scenario

### Phenomenon
- Service has been created
- NodePort is open
- Pod is also Running
- But the page is inaccessible

### How to think about it
Don't end with "The Service is problematic", but instead break it down:

1. Does the selector match?  
2. Is the Pod Ready?  
3. Are there Endpoints?  
4. Is targetPort correct?  
5. Is the application listening port correct?  
6. Is the NodePort link working?

### Key Understanding for Operations
This is the starting point of "structured troubleshooting".

---

## 21. The Most Important Understandings in This Topic

### 1. The core of Service is not "object existence", but "whether the backend can be correctly selected and forwarded"
This is the first understanding.

### 2. Selector is a condition, Endpoints is the result
Don't only look at selector, but also the final backend set.

### 3. Pod Running does not equal being able to enter Service backend
Ready status is very critical.

### 4. If targetPort is mismatched, the business still won't work
Even if Service and Endpoints look normal.

### 5. Service troubleshooting is essentially link troubleshooting
Must trace through layer by layer.

---

## 22. What Level Should You Reach After Learning This

At this stage, it is recommended to reach the following level:

### 1. Be able to explain the relationship between selector, labels, and Endpoints
### 2. Understand why Service being normal doesn't equal business being normal
### 3. Understand the impact of Running and Ready status on Service backend
### 4. Be able to distinguish `port` and `targetPort`
### 5. Be able to establish a basic Service troubleshooting order

---

## 23. Common Follow-up Questions in Interviews

Common questions in this area include:

- How does Service find backend Pods?
- What is the relationship between selector and labels?
- What is Endpoints?
- How to troubleshoot when Service is normal but business is not working?
- Why is Pod Running but no traffic is entering Service backend?
- What is the difference between `port` and `targetPort`?
- What does empty Endpoints usually indicate?
- Why is business still not working even if selector is normal?

---

## 24. Stage Summary

The core of Service advancement is not learning a new object, but connecting previously learned objects into a truly troubleshootable chain.

The most important part of this article is not memorizing several fields, but establishing the following core understandings:

- Service provides a stable access entry
- Selector is responsible for selecting Pods
- Endpoints reflects the final backend result
- Pod Ready status affects whether it enters the backend
- `targetPort` must match the application's actual listening port
- Service being normal does not equal business being definitely normal

As long as these relationships are clarified, many "service unreachable" issues will not remain on the surface when officially entering `09-Apply deployment barriers`.

---

## 25. Keyword Quick Notes

- Service: Stable access entry for Pods
- selector: Condition for Service to select backend Pods
- labels: Labels on Pods
- Endpoints: Current actual backend address set of Service
- port: Service's external service port
- targetPort: Actual port on Pod container receiving traffic
- Ready: Whether Pod can accept traffic
- ClusterIP: Cluster internal access entry
- NodePort: Node-level external exposure entry

---

## 26. Operational Extended Understanding

From an operational perspective, Service-related issues are frequent because it exactly sits between "platform objects" and "business access".

Images, Deployments, and Pods appear to belong to the "resource object layer", while whether a page can be opened or an interface can be accessed belongs to the "business experience layer".  
Service is exactly the bridge connecting these two layers.

So often, the surface issue is:

- Page cannot be opened
- Interface is unreachable
- NodePort has no response

But the real root cause may lie in:

- Selector not matching
- Pod not Ready
- Endpoints empty
- targetPort error
- Application simply not listening on that port

This is why Service troubleshooting capability is almost a mandatory skill in cloud-native operations and Kubernetes interviews.

---

## References
- Kubernetes Service: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes Labels and Selectors: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
- Kubernetes Endpoints: https://kubernetes.io/docs/concepts/services-networking/service/#endpoints

---

## Next Day Recommendation
Next article suggestion:Collate [[02-Why Ingress is Needed - From NodePort to Unified Layer 7 Entry]]