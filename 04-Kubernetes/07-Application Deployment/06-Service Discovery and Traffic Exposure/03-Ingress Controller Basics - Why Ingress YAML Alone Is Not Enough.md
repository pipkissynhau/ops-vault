# 03-Ingress Controller Basics: Why Ingress YAML Alone Is Not Enough

## Documentation Description
- Document Position: This section focuses on the core principles of Ingress in the context of Kubernetes service discovery and traffic routing.
- Applicable Phase: After understanding Services, NodePorts, and why Ingress is needed, this guide delves into how Ingress actually works.
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/06-Service Discovery and Traffic Routing/03-Ingress Controller Basics: Why Ingress YAML Alone Is Not Enough.md`

## Tags
#Kubernetes #Ingress #IngressController #ingress-nginx #Service #NodePort #Seven-Layer Forwarding #Traffic Entry #Reverse Proxy #Application Deployment #Cloud-Native #Ops #Interview Notes

---

## I. Why Do You Need to Learn About the Ingress Controller Even After Understanding Ingress
The previous section addressed one key question:

**Why do you still need to learn about Ingress after understanding Services and NodePorts?**

We already know that:

- Services abstract backend services.
- NodePorts handle external access to clusters.
- Ingress provides a unified entry point for HTTP/HTTPS requests.

However, this understanding is often insufficient. Many beginners face a practical problem when they first start using Ingress:

**Why doesn’t traffic flow even though I’ve written the Ingress YAML?**

This is precisely what this section aims to clarify.

The answer is:

**The Ingress resource object itself is merely a “rule declaration.” It’s the Ingress Controller that actually reads these rules, receives requests, and performs forwarding.**

In other words:

- Ingress: The rules.
- Ingress Controller: The program that executes these rules.

Without an Ingress Controller, the Ingress YAML you create would simply remain an object stored in etcd and wouldn’t become a functional business entry point.

---

## II. Conclusions First: What Exactly Is Ingress and What Is a Controller?
To avoid getting too complicated later on, let’s start with the most essential conclusions.

### 1. What Is Ingress
Ingress is a resource object in Kubernetes. Its main purpose is to specify:

- Which domain name or path should be used.
- To which Service the traffic should be forwarded.

In other words, Ingress essentially serves as a **seven-layer routing rule declaration**. It defines how traffic should be directed, not how it should be actually forwarded.

### 2. What Is an Ingress Controller
An Ingress Controller is a program that runs in the cluster. Its responsibilities include:

- Monitoring Ingress resources in Kubernetes.
- Parsing these rules and loading them into its own proxy program.
- Receiving external HTTP/HTTPS requests.
- Forwarding these requests to the corresponding Service according to the rules.

Therefore, the essence of an Ingress Controller is a **program that actually executes Ingress rules**.

### 3. Why You Can’t Rely Only on Ingress
The reason is that the Ingress resource object itself doesn’t act as a proxy program. It neither listens on ports nor forwards traffic automatically.

It’s like creating a set of traffic routing rules but having no program to read and apply them. Naturally, requests won’t be forwarded accordingly.

---

## III. Why Is Ingress YAML Just a “Rule Declaration”
This point is crucial and needs to be thoroughly understood.

When you create a YAML file like this:

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

Many beginners assume that once the object is created, Kubernetes will automatically start forwarding traffic. However, this isn’t the case. Just because you’ve defined an Ingress resource doesn’t mean Kubernetes will automatically add a seven-layer proxy program to handle requests.

This YAML file simply tells Kubernetes:

- When a request comes to the domain `nginx.example.com` and the path matches `/`, it should be forwarded to `nginx-web-svc:80`.

But who will actually receive these requests? Who will determine the Host and Path? Who will forward them to the Service? These tasks are not performed by the Ingress resource object itself; they are done by the Ingress Controller.

So, you can think of Ingress as a **“seven-layer routing rule sheet” within Kubernetes**.

---

## IV. What Happens Without an Ingress Controller
This question must be answered clearly to avoid confusion later on.

Suppose you have:

- A Deployment.
- A Service.
- An Ingress YAML file.

But no Ingress Controller is deployed in the cluster. What will usually happen?

### 1. The `kubectl get ingress` command shows that the object- Return HTTP response

### 2. It is the Controller that listens on ports
The actual program that listens on ports and receives requests is the proxy within the Ingress Controller, for example:

- Nginx in ingress-nginx
- The Traefik process in Traefik
- The actual gateway programs in other Controllers

### 3. Therefore, external requests first reach the Controller
For example, when accessing:

    http://nginx.example.com/

The actual process is:

- The domain name is first resolved to an entry address.
- This entry address is actually the address exposed by the Controller.
- After receiving the request, the Controller decides how to forward it based on the Ingress rules.

So remember this:

**Clients never “access the Ingress object” but access the entry point exposed by the Ingress Controller.**

---

## X. How is an Ingress Controller deployed in a cluster?

Since an Ingress Controller is a program, it must also be deployed like other applications.

Typically, a Controller will have its own Kubernetes resources in the cluster, such as:

- Deployment
- Pod
- Service
- ConfigMap
- Admission Webhook
- RBAC permissions
- IngressClass

What does this indicate?

It means that an Ingress Controller is not an inherent “invisible capability” of Kubernetes but a component that needs to be installed, run, exposed, and maintained.

Taking the most common ingress-nginx as an example, it usually runs in a dedicated namespace, such as:

    ingress-nginx

This namespace will contain:

- The ingress-nginx-controller Pod
- The ingress-nginx-controller Service

These resources themselves serve as the runtime environment for the Controller.

---

## XI. Why does the Controller also need its own Service?

This is a crucial point that is often overlooked.

Many people might ask at this point:

**Since the Ingress Controller is already running in the cluster, how does external traffic get to it?**

The answer is:

**The Controller usually also needs to be exposed through a Service.**

In other words, the Controller is also an application running within the cluster.  
As an application, it must have:

- A Pod
- A corresponding Service
- An external access point

For example, a common setup for the ingress-nginx Controller is:

### 1. In a testing environment/bare-metal environment
The Controller’s Service might be:

- NodePort

In this case, external requests first go to:

    NodeIP:NodePort

From there, the traffic is forwarded to the ingress-nginx Controller.

### 2. In a cloud-hosted environment
The Controller’s Service might be:

- LoadBalancer

In this scenario, the cloud provider will create an ELB/SLB/LB resource for the Controller. External requests first reach this load balancer before being directed to the Controller.

Therefore, you can see that:

**The Ingress Controller itself also needs an “entry point.”**

This is why, in a real-world environment, understanding how Ingress works requires looking not only at the business-related Ingress YAML but also at:

- Whether the Controller exists
- The status of the Controller’s Pod
- What type of Service the Controller uses
- What the Controller’s entry address is

---

## XII. Why is the Controller considered key to the functioning of Ingress?

The ability of Ingress to function properly depends not on whether the “object” has been created but on:

### 1. Whether a Controller is running
Without a Controller, there is no entity to execute the rules.

### 2. Whether the Controller has taken over this Ingress rule
Not all Ingress requests will be automatically processed by any given Controller.

### 3. Whether the Controller’s own entry point is accessible
Even if the rules are correct, if the Controller’s external entry point is unreachable, traffic will not flow through.

### 4. Whether the Controller correctly identifies the backend Service
Even if requests can reach the entry point, if the rules do not match the correct Service, traffic will still be blocked.

Therefore, the prerequisite for Ingress to work is not just having the YAML configuration but:

**Having a functional Controller that has taken over your rules and is accessible from outside.**

---

## XIII. What is ingress-nginx, and why is it the most suitable example?  

In the Kubernetes ecosystem, there are many implementations of Ingress Controllers, such as:

- ingress-nginx
- Traefik
- HAProxy Ingress
- Cloud providers’ built-in Ingress Controllers

For teaching purposes at this stage, **ingress-nginx** is the most appropriate choice for several reasons:

### 1. Most common
Many resources, tutorials, interview questions, and practical guides refer to ingress-nginx by default.

### 2. Easy to understand its working mechanism
Its architecture is relatively straightforward:

- You define Ingress rules in Kubernetes.
- The ingress-nginx Controller reads these rules and converts them into**Does the Controller layer really exist?**

---

## Eighteen, A Detailed Mental Model: How to Completely Separate Ingress from Controller

If you still find it a bit confusing, you can first use the following model to help you understand.

### 1. Ingress is a rule set
It defines:

- Which domain name
- Which path
- To which Service the traffic should be directed

### 2. The Ingress Controller is the executor
Its responsibilities include:

- Listening for rules
- Receiving requests
- Matching rules
- Forwarding traffic

### 3. External requests first reach the Controller
Not the Ingress object itself.

### 4. The Controller then forwards traffic according to the Ingress rules
Once a match is found, it directs the request to the Service.

### 5. The Service finally forwards the request to the Pod
This part is still the same as what you learned about Services before.

You can imagine the whole process as follows:

    Ingress = Route Table
    Controller = The person who checks the route table and actually directs traffic
    Service = Backend service entrance
    Pod = The business instance that actually processes requests

Once you establish this mental model, all subsequent configurations and troubleshooting will be much easier.

---

## Nineteen, How to Explain in Interviews “Why Just Ingress YAML Is Not Enough”

If an interviewer asks:

**Why isn’t creating an Ingress resource enough?**

You can answer like this:

The Ingress resource itself is just a layer-7 routing rule object in Kubernetes. It describes the mapping relationship between domain names, paths, and backend Services, but it doesn’t listen for ports or forward traffic itself.  
It is the Ingress Controller that actually receives external requests, parses the Ingress rules, and forwards the traffic to the backend Services.  
Therefore, if there is no available Ingress Controller deployed in the cluster, or if the Ingress is not managed by the corresponding Controller, then even if the Ingress YAML is correctly configured, the business entrance may still not work.

In this response, it’s important to remember these key terms:

- Rule object
- Does not listen for ports
- Does not directly forward traffic
- The real executor is the Controller
- Rules will not take effect without a Controller

---

## Twenty, The Most Important Conclusions of This Article

### 1. Ingress is not a proxy program, but a rule object
It simply describes how incoming traffic should be forwarded.

### 2. The Ingress Controller is the program that actually executes the rules
Its responsibilities include:

- Listening for Ingress resources
- Receiving external requests
- Matching rules
- Forwarding to Services

### 3. Clients do not access the Ingress object itself, but the Controller’s entrance address
The Controller is the first stop for incoming traffic.

### 4. The Controller also needs to be deployed and made accessible
It usually has its own:

- Pod
- Service
- External access methods

### 5. Without a Controller, Ingress often does not function properly
This is a common issue and a key point that beginners often overlook.

### 6. Understand the configuration relationship and request traffic flow separately
Configuration relationship:

    Ingress -> Service -> Pod

Real traffic flow:

    Client -> Controller entrance -> Ingress rule matching -> Service -> Pod

---

## Twenty-One, Summary of This Section

The most important thing here is not to memorize a specific YAML field, but to thoroughly understand the following concept:

**Ingress is just a set of rules; the Ingress Controller is the program that executes these rules.**

In other words, creating an Ingress does not mean that the entrance is automatically available.  
Only when there is a working Ingress Controller in the cluster, which properly manages your Ingress rules and can be accessed from external requests, does the entire layer-7 entry chain truly function.

By now, you should be able to answer these two questions:

### 1. Why just Ingress YAML is not enough
Because it only defines the rules; there is no executor.

### 2. Where do external requests first go
They first reach the Controller’s entrance, and then the Controller forwards them according to the Ingress rules.

Once you fully understand these two layers, further learning topics such as:

- The deployment example of ingress-nginx
- Your first practical use of Ingress
- Troubleshooting issues like Ingress not working, 404 errors, or 502 responses

will be much smoother.

---

## Key Terms for Quick Reference

- Ingress: Kubernetes layer-7 entry rule object
- Ingress Controller: The controller program that executes Ingress rules
- ingress-nginx: One of the most common implementations of Ingress Controllers
- Rule definition: Ingress only specifies how traffic should be forwarded
- Executor: The Controller is the one that actually receives and forwards requests
- Controller entrance: The first stop