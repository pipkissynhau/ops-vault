# DaoCloud Interview Question 05: What are the working modes of CoreDNS, and how does it resolve domain names?

## Question Description
Interview question: What are the different working modes of CoreDNS? How does it resolve domain names?

---

## Key Points
In a Kubernetes context, a more accurate way to answer this question is:

CoreDNS can generally be understood to operate in two main modes:

- **Authoritative Resolution**
- **Forwarding Resolution**

For:

- Domain names within the cluster, CoreDNS responds directly.
- Requests for domain names outside the cluster are forwarded to upstream DNS.

According to CoreDNS's official documentation, it is a plugin-based DNS server. In Kubernetes, the `kubernetes` plugin is responsible for resolving cluster-domain names, while the `forward` plugin forwards requests to external DNS servers.

---

## Correct Understanding
Avoid answering with purely theoretical DNS concepts such as "recursive query and iterative query."

In Kubernetes interviews, interviewers are more interested in knowing:

- How CoreDNS receives requests within the cluster.
- Which domain names it resolves itself.
- Which domain names it forwards.
- What data it uses to resolve Service domain names.

Kubernetes officially states that DNS records for Services and Pods are created within the cluster, and CoreDNS typically serves as the DNS provider in such clusters.

---

## How to Answer During an Interview

### One-Sentence Version
In Kubernetes, I generally consider CoreDNS to operate in two main modes: authoritative resolution for domain names within the cluster, which is handled by the `kubernetes` plugin based on Service, Pod, and EndpointSlice information; and forwarding resolution for external or non-cluster-domain names, which is done through the `forward` plugin.

### More Detailed Version
CoreDNS is essentially a plugin-based DNS server. In Kubernetes, it primarily functions in two ways:
1. **Authoritative Resolution**: It directly resolves domain names within the cluster, such as `service.namespace.svc.cluster.local`. This is handled by the `kubernetes` plugin, which uses data from Kubernetes APIs like Services, Pods, and EndpointSlices.
2. **Forwarding Resolution**: For domain names outside the cluster, such as public or internal enterprise domains, CoreDNS forwards the requests to upstream DNS servers using the `forward` plugin.

When a Pod initiates a DNS query, CoreDNS first checks its Corefile and zone configuration to determine which plugin should handle the request. If it’s a cluster-domain name, the `kubernetes` plugin processes it; otherwise, it is forwarded to another plugin like `forward`.

---

## How CoreDNS Resolves Domain Names

### Step 1: The Pod Initiates a DNS Query
An application inside a Pod accesses a domain name, for example:

    nginx.default.svc.cluster.local

The Pod sends the request to the cluster DNS according to the `nameserver` and `search` settings in its `/etc/resolv.conf`. Kubernetes officially states that kubelet configures DNS for Pods so they can resolve Service names.

---

### Step 2: CoreDNS Matches Zones and Plugin Chains Based on the Corefile
CoreDNS is not a fixed-functionality server; instead, it uses its Corefile to match different zones and then invokes the corresponding plugin chain. The Corefile specifies which plugins should handle various domain names.

For example:

- `cluster.local` is processed by the `kubernetes` plugin.
- Other domain names are handled by the `forward` plugin.

---

### Step 3: If It’s a Cluster Domain Name, It’s Handled by the kubernetes Plugin
If the queried domain name is within the cluster, such as `service.namespace.svc.cluster.local`, CoreDNS uses the `kubernetes` plugin. This plugin generates the DNS response based on Kubernetes API data.

---

### Step 4: If It’s Not a Cluster Domain Name, It’s Weiter Processed According to Rules
If the domain name is not within the cluster, CoreDNS continues processing it according to rules defined in its Corefile, such as forwarding it to the `forward` plugin or using other custom plugins. The most common scenario here is forwarding to an external DNS server.

---

### Step 5: The Result Is Returned to the Pod
Once CoreDNS has processed the query, it returns the DNS response to the Pod. The application inside the Pod then uses this IP address to access the target service.

---

## Specific Examples

### Example 1: Resolving a Cluster-Internal Service
PodA queries:

    redis.default.svc.cluster.local

CoreDNS processes it as follows:

1. PodA initiates the query.
2. CoreDNS identifies that it’s a `cluster.local` domain name.
3. The `kubernetes` plugin looks up the Service and EndpointSlice information.
4. It returns the corresponding ClusterIP.

This is **authoritative resolution**.

---

### Example 2: Resolving an External Domain Name
PodA queries:

    www.baidu.com

CoreDNS processes it as follows:

1. PodAIn the parsing process, the Pod first sends DNS requests to the cluster's DNS server according to the `/etc/resolv.conf` file. Upon receiving these requests, CoreDNS first matches the zones and plugin chains based on the Corefile. If it is a cluster domain name like `cluster.local`, the `kubernetes` plugin handles the response; otherwise, it proceeds with processing according to rules such as `forward` before returning the result to the Pod.

---

## Memory Templates
First, remember these two methods:

- Authoritative resolution
- Forwarding resolution

Then, recall the main flow of the resolution process:

Pod sends a request  
→ CoreDNS receives the query  
→ Corefile matches zones / plugin chains  
→ Cluster domains are handled by the kubernetes plugin  
→ Non-cluster domains are processed through forwarding  
→ Result is returned

---

## Tags
#Kubernetes
#CoreDNS
#DNS
#Service
#Pod
#kubernetes-plugin
#forward-plugin
#Cloud-Native-Interview
#Ops-Interview
#Infrastructure