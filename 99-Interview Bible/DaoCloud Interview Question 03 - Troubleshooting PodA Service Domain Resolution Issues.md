# DaoCloud Interview Question 03: Troubleshooting PodA's Failure to Resolve Service Domain Name

## Question Description
Interview Question: PodA and PodB communication fails, and when using dig/nslookup inside PodA, the Service domain name cannot be resolved. How should we troubleshoot?

---

## Core Conclusion
This question clearly indicates a key phenomenon:

- dig/nslookup inside PodA fails to resolve the Service domain name

Therefore, the problem should first be categorized as:

- Prioritize troubleshooting the Kubernetes DNS resolution chain
- Temporarily do not focus on business port, application logs, or PodB process status

Kubernetes officially states that the cluster creates DNS records for Services and Pods, and Pods resolve via DNS components using kubelet-configured DNS settings. ([kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/?utm_source=chatgpt.com))

---

## Correct Understanding
Do not immediately check:
- Whether PodB application is listening on ports
- Whether Service selector matches
- Whether container logs have errors

Because the question explicitly states **domain name resolution failure**, the first step should be to troubleshoot along the DNS chain:

1. Confirm Service name and namespace are correct
2. Check if PodA's `/etc/resolv.conf` is normal
3. Verify if Pod's `dnsPolicy` is correct
4. Check if CoreDNS/kube-dns is normal
5. Check if CoreDNS configuration is abnormal
6. Confirm existence of Service/Endpoints/EndpointSlice
7. Verify reachability from PodA to DNS service
8. Check if NetworkPolicy blocks DNS port 53
9. Check if NodeLocal DNSCache is enabled
10. Finally check kube-proxy, CNI, and node network

Kubernetes official DNS debugging documentation explicitly lists checking `resolv.conf`, CoreDNS Pod, and CoreDNS logs as critical steps for DNS fault localization. ([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/?utm_source=chatgpt.com))

---

## How to Answer in an Interview

### One-Sentence Version
If dig/nslookup in PodA cannot resolve the Service domain name, I would first define it as a DNS layer failure rather than a business port issue; prioritize checking Service name and namespace, PodA's resolv.conf, dnsPolicy, and CoreDNS status, then further check NetworkPolicy, EndpointSlice, and node-side network. ([kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/?utm_source=chatgpt.com))

### More Detailed Version
Since dig/nslookup in PodA cannot resolve the Service domain name, I would prioritize troubleshooting along the Kubernetes DNS chain.  
I would first confirm if the Service name is correct, especially whether it's cross-namespace, and test directly with full FQDN if needed.  
Then enter PodA to check `/etc/resolv.conf`, confirm `nameserver`, `search domain`, and `dnsPolicy` are normal.  
Next, check CoreDNS or kube-dns services and Pods in kube-system, verify if there are error logs, and confirm CoreDNS ConfigMap has no configuration anomalies.  
After that, check if the target Service, Endpoints/EndpointSlice exist, and validate reachability from PodA to DNS service; if DNS service itself is unreachable, further troubleshoot NetworkPolicy blocking UDP/TCP 53, and issues with NodeLocal DNSCache, kube-proxy, CNI, or node network. ([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/?utm_source=chatgpt.com))

---

## Specific Troubleshooting Steps

### 1. First confirm Service name and namespace are correct
Kubernetes creates DNS records for Services, but short domain name resolution depends on Pod's namespace and search domain configuration.  
Same namespace can use just Service name, but cross-namespace typically requires namespace, and full FQDN should be used when necessary. ([kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/?utm_source=chatgpt.com))

Common commands:
    kubectl get svc -A | grep <svc-name>
    kubectl get svc -n <ns> <svc-name>

Cross-namespace scenarios can directly test:
    dig <svc-name>.<ns>.svc.cluster.local
    nslookup <svc-name>.<ns>.svc.cluster.local

#### Issues to Pay Attention To
- Service name spelling error
- Wrong namespace checked
- Whether FQDN should be used originally

---

### 2. Check PodA's /etc/resolv.conf
Kubernetes documentation states that kubelet configures Pod's DNS to resolve cluster Service names. ([kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/?utm_source=chatgpt.com))

Enter PodA to check:
    cat /etc/resolv.conf

Focus on:
- `nameserver` pointing to cluster DNS address
- `search` containing:
  - `<namespace>.svc.cluster.local`
  - `svc.cluster.local`
  - `cluster.local`
- `options ndots` abnormality

Kubernetes DNS debugging documentation also treats checking Pod's `resolv.conf` as a direct troubleshooting step. ([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/?utm_source=chatgpt.com))

---

### 3. Check if Pod's dnsPolicy is normal
By default, Pods typically use:

- `dnsPolicy: ClusterFirst`

If changed to:
- `Default`
- `None`

Or combined with `dnsConfig` for incorrect customizations, it may cause Pods to bypass cluster DNS.  
Kubernetes DNS configuration documentation clearly explains `dnsPolicy` and custom DNS behavior. ([kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/?utm_source=chatgpt.com))

---

### 4. Confirm CoreDNS/kube-dns is normal
In Kubernetes clusters, Service/Pod DNS resolution is typically provided by CoreDNS.  
Official DNS debugging documentation explicitly recommends checking CoreDNS Pod and logs. ([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/?utm_source=chatgpt.com))

Common Commands:
    kubectl get svc -n kube-system
    kubectl get pod -n kube-system -l k8s-app=kube-dns
    kubectl get pod -n kube-system -l k8s-app=coredns
    kubectl logs -n kube-system -l k8s-app=coredns --tail=100

Focus on:
- CoreDNS Pod status: Running / Ready
- Frequent restarts
- Error logs
- Existence of kube-dns / CoreDNS Service

---

### 5. Check CoreDNS Configuration
CoreDNS behavior is determined by Corefile.  
Kubernetes allows customizing cluster DNS behavior, so corrupted CoreDNS configuration is a common failure point.([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/?utm_source=chatgpt.com))

Common commands:
    kubectl get cm -n kube-system coredns -o yaml

Focus on:
- `kubernetes cluster.local` configuration is normal
- `forward` upstream configuration is abnormal
- Presence of incorrect rewrite / hosts / stub domain configurations

CoreDNS official documentation states that `kubernetes` plugin handles cluster domain resolution, and `forward` plugin forwards to upstream DNS.([coredns.io](https://coredns.io/plugins/kubernetes/?utm_source=chatgpt.com)) ([coredns.io](https://coredns.io/plugins/forward/?utm_source=chatgpt.com))

---

### 6. Check Service and Endpoints / EndpointSlice
Strictly speaking, "DNS resolution failure" and "Service has no backend" are different issues.  
However, it's advisable to check this step during interviews to demonstrate thoroughness.

Common commands:
    kubectl get svc -n <ns> <svc-name> -o wide
    kubectl get endpoints -n <ns> <svc-name>
    kubectl get endpointslice -n <ns>

CoreDNS's `kubernetes` plugin provides service discovery based on Services and EndpointSlices data.([coredns.io](https://coredns.io/plugins/kubernetes/?utm_source=chatgpt.com))

#### Key issues to focus on
- Existence of Service
- Normal ClusterIP
- Empty Endpoints / EndpointSlice

---

### 7. Directly test DNS service reachability from PodA
You can perform DNS server IP queries directly inside PodA to determine if the issue is "wrong name" or "DNS service unreachable".

First check DNS address:
    cat /etc/resolv.conf

Then test:
    nslookup kubernetes.default.svc.cluster.local <dns-ip>
    dig @<dns-ip> kubernetes.default.svc.cluster.local

`kubernetes.default.svc.cluster.local` is a commonly used benchmark target, and the official DNS debugging documentation also recommends using a known Service name to verify DNS functionality.([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/?utm_source=chatgpt.com))

#### Typical judgment
- If `kubernetes.default.svc.cluster.local` cannot be resolved, the issue is more likely with the DNS service itself being unreachable or abnormal
- If this can be resolved but the target Service cannot, the issue is more likely with the name, namespace, or Service record itself

---

### 8. Check NetworkPolicy blocking DNS
PodA to CoreDNS is essentially a network access.  
If there's a default deny egress policy without allowing access to kube-dns / CoreDNS on port 53, it would manifest as Pod being unable to resolve domain names.([kubernetes.io](https://kubernetes.io/docs/tasks/debug/?utm_source=chatgpt.com))

Common commands:
    kubectl get netpol -A
    kubectl describe netpol -n <ns>

Focus on:
- Whether PodA's namespace has default deny egress
- Whether access to CoreDNS / kube-dns is allowed
- Whether both UDP 53 and TCP 53 are allowed

---

### 9. Check if NodeLocal DNSCache is enabled
Some clusters don't let Pods directly access CoreDNS Service, but instead use NodeLocal DNSCache to access node-local DNS cache proxy.  
Kubernetes official documentation has specific explanations about NodeLocal DNSCache.([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/?utm_source=chatgpt.com))

If enabled, continue checking:
- DaemonSet status of NodeLocal DNSCache
- Whether Pod's `nameserver` points to local cache address
- Abnormality of node-side local DNS proxy

---

### 10. Finally check kube-proxy / CNI / node network
If all previous steps are normal, consider:

- kube-proxy rule anomalies
- CNI issues
- Node network issues
- Pod to DNS Service IP unreachable

If needed, continue checking:
    kubectl get pod -o wide -n kube-system
    kubectl describe pod <podA> -n <ns>

If necessary, proceed to node troubleshooting:
- kube-proxy
- iptables / nft
- CNI
- Node to DNS service connectivity

---

## Key Knowledge Points

### 1. This is a DNS link issue, not a business port issue
The problem statement says dig/nslookup cannot resolve domain names, so the first step should be to locate DNS failure, not application layer failure.

### 2. Pod DNS is configured by kubelet
Pod's `/etc/resolv.conf`, `dnsPolicy`, `search domain` are key for troubleshooting.([kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/?utm_source=chatgpt.com))

### 3. CoreDNS provides cluster domain resolution
CoreDNS's `kubernetes` plugin handles cluster internal domain resolution, and `forward` plugin forwards non-cluster domain requests to upstream DNS.([coredns.io](https://coredns.io/plugins/kubernetes/?utm_source=chatgpt.com)) ([coredns.io](https://coredns.io/plugins/forward/?utm_source=chatgpt.com))

### 4. NetworkPolicy affects DNS
If UDP/TCP 53 to DNS service is not allowed, even if the business itself is fine, domain names cannot be resolved.

---

## Common Mistakes /think

### Common Mistake 1: Start by checking the business port  
Inappropriate.  
The question already points to a DNS failure, so we should first check the DNS link.

### Common Mistake 2: Ignore namespace and FQDN  
In cross-namespace scenarios, short domains may not be available.  
We should prioritize using fully qualified domains for verification.

### Common Mistake 3: Ignore PodA's resolv.conf  
This is one of the first pieces of evidence for troubleshooting Kubernetes DNS failures.([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/?utm_source=chatgpt.com))

### Common Mistake 4: Skip checking NetworkPolicy  
Many real-world failures are not due to CoreDNS failing, but rather DNS traffic being blocked by policies.

### Common Mistake 5: Confuse "no Endpoints" with "domain resolution failure"  
These are not issues at the same level.  
However, in interviews, it's best to check both to demonstrate completeness.

---

## Interview Verbal Template  
If PodA and PodB communication fails, and I can't resolve the Service domain name using dig/nslookup in PodA, I'll first classify it as a DNS layer failure, not a business port failure.  
I'll first confirm whether the Service name and namespace are correctly written, and prioritize testing with fully qualified FQDNs in cross-namespace scenarios.  
Then I'll enter PodA to check `/etc/resolv.conf`, confirming `nameserver`, `search domain`, and `dnsPolicy` are normal.  
Next, I'll check the CoreDNS or kube-dns service and Pod in kube-system, ensuring they're normal and checking for error logs, then verifying whether CoreDNS's ConfigMap was mistakenly modified.  
At the same time, I'll check whether the target Service and corresponding Endpoints/EndpointSlice exist, and directly query the cluster DNS IP from PodA, such as resolving `kubernetes.default.svc.cluster.local`, to determine if it's a name issue or the DNS service itself is unreachable.  
If the DNS service itself is unreachable, I'll continue troubleshooting whether NetworkPolicy blocks UDP/TCP 53, and whether the cluster has enabled NodeLocal DNSCache, kube-proxy, or node network anomalies.([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/?utm_source=chatgpt.com))

---

## Memory Template  
First remember this main line:

Name / FQDN  
→ resolv.conf  
→ dnsPolicy  
→ CoreDNS  
→ Service / EndpointSlice  
→ DNS Reachability  
→ NetworkPolicy  
→ NodeLocal DNSCache  
→ Node Network

Also remember this phrase:

First classify it as a DNS issue, not a port issue

---

## Tags  
#Kubernetes  
#DNS  
#CoreDNS  
#Service  
#Pod  
#NetworkPolicy  
#EndpointSlice  
#NodeLocalDNSCache  
#FaultCheck.  
#TheYunwonInterview.  
#TransportInterview