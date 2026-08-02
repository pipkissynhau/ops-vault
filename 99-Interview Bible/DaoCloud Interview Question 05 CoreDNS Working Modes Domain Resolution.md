# DaoCloud Interview Question 05: How Many Working Modes Does CoreDNS Have, and How to Resolve Domains?

## Question Description
Interview Question: How many working modes does CoreDNS have? How to resolve domains?

---

## Core Conclusion
In Kubernetes scenarios, the more stable answer to this question is:

CoreDNS is commonly understood to have two main resolution methods:

- **Authoritative resolution (authoritative)**
- **Forwarding resolution (forwarding)**

Among them:

- For cluster internal domain names, CoreDNS directly responds
- For requests not belonging to the cluster domain, CoreDNS forwards to upstream DNS

CoreDNS official documentation states that CoreDNS is a DNS server based on a plugin chain; in Kubernetes, the `kubernetes` plugin handles cluster domain resolution, and the `forward` plugin forwards requests to upstream DNS.

---

## Correct Understanding
Do not answer with "recursive query and iterative query" - this is a pure DNS theory answer.

In Kubernetes interviews, the interviewer is more interested in:

- How CoreDNS receives requests in the cluster
- Which domains it answers directly
- Which domains it forwards
- What data it uses to answer Service domain names

Kubernetes official documentation states that the cluster creates DNS records for Services and Pods; CoreDNS is typically the DNS provider in a Kubernetes cluster.

---

## How to Answer in an Interview

### One-Sentence Version
In Kubernetes, I generally understand CoreDNS to have two main working modes: one is authoritative resolution for cluster internal domains, typically handled by the `kubernetes` plugin based on Service, Pod, EndpointSlice, etc. information directly responding; the other is forwarding mode, for requests not belonging to the cluster domain, such as external domains, CoreDNS forwards to upstream DNS via the `forward` plugin.

### More Complete Version
CoreDNS is essentially a plugin-based DNS server.  
In Kubernetes scenarios, it's common to understand it as having two main working modes:  
The first is authoritative resolution mode, where CoreDNS directly responds to cluster internal domain names, such as `service.namespace.svc.cluster.local`, typically handled by the `kubernetes` plugin;  
The second is forwarding resolution mode, for requests not belonging to the cluster domain, such as public domain names or enterprise internal domains, CoreDNS forwards the request to upstream DNS via the `forward` plugin.  
So after a Pod initiates a DNS query, CoreDNS first determines which plugin to use based on Corefile and zone matching rules: cluster domain names are answered by the `kubernetes` plugin, and non-cluster domain names are processed by plugins like `forward`.

---

## CoreDNS's Two Main Working Modes

### 1. Authoritative Resolution Mode (authoritative)
CoreDNS has authoritative answers for certain domains.

In Kubernetes, the most typical examples are:

- `*.svc.cluster.local`
- `*.pod.cluster.local`

These records are generated and answered by the `kubernetes` plugin based on Kubernetes API object information.  
CoreDNS official documentation clearly states that the `kubernetes` plugin reads Kubernetes Service, Pod, and EndpointSlice data to provide resolution for cluster domains.

#### Typical Example
For example, a Pod querying:

    redis.default.svc.cluster.local

CoreDNS processing steps:

1. Receives DNS query from the Pod
2. Detects the target domain belongs to `cluster.local`
3. Passes it to the `kubernetes` plugin
4. The `kubernetes` plugin generates DNS response based on cached Service / EndpointSlice information
5. Returns ClusterIP or related records

---

### 2. Forwarding Resolution Mode (forwarding)
When the queried domain does not belong to the cluster domain, CoreDNS typically does not answer directly but forwards to upstream DNS.

Examples include:

- `www.baidu.com`
- `api.github.com`
- Enterprise internal custom domains

Such requests are typically forwarded via the `forward` plugin.  
CoreDNS official documentation states that the `forward` plugin can forward DNS requests to one or more upstream DNS servers.

#### Typical Example
For example, a Pod querying:

    www.baidu.com

CoreDNS processing steps:

1. Receives the query
2. Detects it's not a cluster domain
3. Matches the `forward` rule in Corefile
4. Forwards to upstream DNS
5. Retrieves results and returns them to the Pod

---

## How CoreDNS Resolves Domains

### Step 1: Pod Initiates DNS Query
A Pod's application accesses a name, such as:

    nginx.default.svc.cluster.local

The Pod sends the request to the cluster DNS based on the `/etc/resolv.conf` configuration in `nameserver` and `search` domains.  
Kubernetes official documentation states that kubelet configures DNS for Pods, enabling them to look up services by Service name.

---

### Step 2: CoreDNS Matches Zone and Plugin Chain Based on Corefile
CoreDNS is not a single-entity DNS with hard-coded logic, but rather matches different zones based on Corefile and follows the corresponding plugin chain.  
CoreDNS official documentation states that its functionality is implemented by plugins, and Corefile determines which plugins handle which domains.

A common approach is:

- `cluster.local` is handled by the `kubernetes` plugin
- Other domains are handled by `forward`

---

### Step 3: Cluster Domains Are Handled by the Kubernetes Plugin
If the query is for:

- `service.namespace.svc.cluster.local`
- `pod-ip.namespace.pod.cluster.local`

CoreDNS passes it to the `kubernetes` plugin.  
This plugin generates DNS responses based on Kubernetes API data.

---

### Step 4: Non-Cluster Domains Are Processed According to Forwarding Rules
If it's not a cluster domain like `cluster.local`, CoreDNS continues matching, such as passing it to:

- `forward`
- `hosts`
- Other custom plugins

The most common is `forward` forwarding to upstream DNS.

---

### Step 5: Return Results to the Pod
After obtaining results, CoreDNS returns the DNS response to the Pod.  
The Pod's application then uses the resolved IP to access the target service.

---

## Specific Examples

### Example 1: Resolving Cluster Internal Service
PodA queries:

    redis.default.svc.cluster.local

CoreDNS Processing Flow:

1. PodA initiates a query
2. CoreDNS determines this is `cluster.local` domain
3. `kubernetes` plugin queries Service / EndpointSlice information
4. Returns the corresponding ClusterIP

This is **authoritative resolution**. 

---

### Example 2: Resolving External Domains
PodA queries:

    www.baidu.com

CoreDNS processing flow:

1. PodA initiates a query
2. CoreDNS determines this is not a cluster domain
3. Matches `forward` rule
4. Forwards the request to upstream DNS
5. Returns the result to PodA

This is **forwarding resolution**. 

---

## Key Knowledge Points

### 1. CoreDNS is a plugin-based DNS server
Its behavior is not fixed, but determined by Corefile and plugin chain. 

### 2. Two most critical plugins in Kubernetes
- `kubernetes`: Resolves cluster internal domains
- `forward`: Forwards cluster external domains

### 3. CoreDNS answers cluster domains, not calculating on-demand
More accurately: `kubernetes` plugin watches/cache Kubernetes resources, then answers DNS queries based on these data. 

### 4. Pod's DNS requests first go to CoreDNS
Pods do not directly query external DNS, but first send requests to cluster DNS according to kubelet's DNS configuration. 

---

## Common Mistakes

### Mistake 1: Answering as pure DNS theory questions
For example, only saying "recursive query, iterative query".  
This isn't wrong, but not the most wanted answer in Kubernetes interviews.

### Mistake 2: Saying CoreDNS only resolves Service, ignoring external domains
Inaccurate.  
External domains are typically forwarded via `forward` plugin. 

### Mistake 3: Saying CoreDNS queries apiserver every time
More accurately: `kubernetes` plugin watches/cache Kubernetes resources, then answers based on these data. 

### Mistake 4: Ignoring Corefile and plugin chain
If your answer lacks Corefile/plugin chain concepts, it shows incomplete understanding of CoreDNS.

---

## Interview Spoken Template
In Kubernetes, I generally understand CoreDNS as two main working modes:  
One is authoritative resolution mode, directly answering cluster internal domains like Service and Pod domains, which is typically handled by `kubernetes` plugin based on Kubernetes Service, Pod, EndpointSlice information;  
The other is forwarding resolution mode, for requests not belonging to cluster domains like external domains or enterprise internal domains, CoreDNS forwards the request to upstream DNS via `forward` plugin.  
In the resolution process, Pod first sends DNS requests to cluster DNS via `/etc/resolv.conf`, CoreDNS receives the query, then matches zone/plugin chain in Corefile; if it's `cluster.local` type cluster domain, it's answered by `kubernetes` plugin; if not, it continues processing according to `forward` rules, finally returning the result to Pod. 

---

## Memory Template
First remember two methods:

- Authoritative resolution (authoritative)
- Forwarding resolution (forwarding)

Then remember the main flow:

Pod sends request  
→ CoreDNS receives query  
→ Corefile matches zone/plugin chain  
→ Cluster domain is handled by kubernetes plugin  
→ Non-cluster domain is handled by forward  
→ Returns result

---

## Tags
#Kubernetes
#CoreDNS
#DNS
#Service
#Pod
#KubernetesPlugin
#ForwardPlugin
#TheYunwonInterview.
#TransportInterview
#Infrastructure