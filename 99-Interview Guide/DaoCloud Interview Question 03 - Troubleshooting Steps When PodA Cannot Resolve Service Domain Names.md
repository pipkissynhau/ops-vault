# DaoCloud Interview Question 03: Troubleshooting Steps When PodA Cannot Resolve Service Domain Names

## Problem Description
During an interview, it was asked how to troubleshoot the communication failure between PodA and PodB. When using `dig` or `nslookup` inside PodA, the Service domain name could not be resolved. What should be done in this case?

---

## Key Conclusion
The question clearly presents a key phenomenon:

- It is impossible to resolve Service domain names using `dig` or `nslookup` within PodA.

Therefore, the problem should first be identified as:

- Prioritize checking the Kubernetes DNS resolution chain.
- Do not immediately focus on business ports, application logs, or the status of PodB processes.

According to Kubernetes documentation, the cluster creates DNS records for Services and Pods. Pods send resolution requests to the cluster's DNS components using the DNS settings configured by kubelet. ([kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/?utm_source=chatgpt.com))

---

## Correct Understanding
Do not start by checking the following immediately:

- Whether PodB's application is listening on the correct port.
- Whether the Service selector matches correctly.
- Whether there are any error messages in container logs.

Since the problem is specifically about **domain name resolution failure**, the DNS chain should be checked first:

1. Verify whether the Service name and namespace are correctly specified.
2. Check if PodA's `/etc/resolv.conf` file is set up properly.
3. Confirm whether the Pod's `dnsPolicy` setting is correct.
4. Ensure that CoreDNS or kube-dns is running smoothly.
5. Check for any configuration errors in CoreDNS.
6. Verify whether the Service, Endpoints, and EndpointSlice exist.
7. Confirm that PodA can reach the DNS service.
8. Check if Network Policies are blocking port 53 for DNS requests.
9. Determine whether NodeLocal DNSCache is enabled.
10. Only if none of the above steps resolve the issue, consider checking kube-proxy, CNI, or the node network.

Kubernetes's official documentation on debugging DNS issues emphasizes checking `resolv.conf`, CoreDNS Pods, and CoreDNS logs as important steps in troubleshooting DNS problems. ([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/?utm_source=chatgpt.com))

---

## How to Answer During the Interview

### One-Sentence Version
If `dig` or `nslookup` inside PodA cannot resolve Service domain names, I would first consider it a DNS-related issue rather than a problem with business ports. I would start by checking whether the Service name and namespace are correct, as well as PodA's `resolv.conf` and `dnsPolicy` settings. Only if these steps do not help would I further investigate Network Policies, EndpointSlice, and the node network. ([kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/?utm_source=chatgpt.com))

### More Detailed Version
Since it is impossible to resolve Service domain names using `dig` or `nslookup` inside PodA, I would first follow the Kubernetes DNS resolution chain. I would:
1. Verify that the Service name and namespace are correct, especially if they span different namespaces. In such cases, use the full FQDN for testing.
2. Check PodA's `/etc/resolv.conf` file to ensure that `nameserver`, `search domain`, and `dnsPolicy` settings are appropriate.
3. Verify whether the CoreDNS or kube-dns service in the kube-system is running smoothly and whether there are any error logs.
4. Confirm that the Service, Endpoints, and EndpointSlice exist and that PodA can reach the DNS service.
5. If the DNS service itself is unavailable, check if Network Policies are blocking UDP/TCP port 53, as well as whether NodeLocal DNSCache, kube-proxy, CNI, or the node network are causing issues. ([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/?utm_source=chatgpt.com))

---

## Specific Troubleshooting Steps

### 1. Verify Service Name and Namespace
Kubernetes creates DNS records for Services, but whether short domain names can be resolved correctly depends on the namespace and search domain settings of the Pod.  
For pods within the same namespace, only the Service name needs to be specified; for cross-namespace cases, the full FQDN should be used. ([kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/?utm_source=chatgpt.com))

Common commands:
    `kubectl get svc -A | grep <svc-name>`
    `kubectl get svc -n <ns> <svc-name>`

For cross-namespace scenarios, you can directly test using:
    `dig <svc-name>.<nsKubernetes allows for the customization of cluster DNS behavior, making configuration errors in CoreDNS a common source of issues. ([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/?utm_source=chatgpt.com))

Common commands:
    kubectl get cm -n kube-system coredns -o yaml

Key points to check:
- Whether the `kubernetes cluster.local` configuration is correct
- If there are any abnormalities in the `forward` upstream configuration
- The presence of incorrect rewrite, /hosts, or stub domain configurations

According to CoreDNS's official documentation, the `kubernetes` plugin is responsible for cluster domain resolution, while the `forward` plugin handles forwarding requests to external DNS servers. ([coredns.io](https://coredns.io/plugins/kubernetes/?utm_source=chatgpt.com)) ([coredns.io](https://coredns.io/plugins/forward/?utm_source=chatgpt.com))

---

### 6. Checking Services and Endpoints / EndpointSlice
Strictly speaking, "DNS resolution failure" and "Service having no backend" are not the same issue.
However, it's a good practice to include this step in interviews to ensure a comprehensive troubleshooting process.

Common commands:
    kubectl get svc -n <ns> <svc-name> -o wide
    kubectl get endpoints -n <ns> <svc-name>
    kubectl get endpointslice -n <ns>

CoreDNS's `kubernetes` plugin relies on Services and EndpointSlices to provide service discovery. ([coredns.io](https://coredns.io/plugins/kubernetes/?utm_source=chatgpt.com))

#### Issues to focus on:
- Whether the Service exists
- If the ClusterIP is functioning correctly
- Whether Endpoints / EndpointSlice are present

---

### 7. Directly testing DNS service accessibility from PodA
It's possible to directly query the DNS server IP within PodA to determine whether the issue lies in an incorrect name or an unreachable DNS service.

First, check the DNS address:
    cat /etc/resolv.conf

Then perform tests:
    nslookup kubernetes.default.svc.cluster.local <dns-ip>
    dig @<dns-ip> kubernetes.default.svc.cluster.local

`kubernetes.default.svc.cluster.local` is a commonly used benchmark target. Official DNS debugging documentation also recommends using this known Service name to verify DNS functionality. ([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/?utm_source=chatgpt.com))

#### Typical conclusions:
- If `kubernetes.default.svc.cluster.local` cannot be resolved, the problem is likely with the DNS service itself or its connectivity.
- If this target can be resolved but the desired Service cannot, the issue may lie in the name, namespace, or Service record.

---

### 8. Checking if NetworkPolicy blocks DNS access
The communication between PodA and CoreDNS is essentially a network access issue.
If there are default denial egress policies that prevent access to port 53 of kube-dns/CoreDNS, it will result in the inability to resolve domain names within the Pod. ([kubernetes.io](https://kubernetes.io/docs/tasks/debug/?utm_source=chatgpt.com))

Common commands:
    kubectl get netpol -A
    kubectl describe netpol -n <ns>

Key points to check:
- Whether there are default denial egress policies in the namespace where PodA resides
- If access to CoreDNS/kube-dns is allowed
- Whether both UDP and TCP port 53 are permitted

---

### 9. Checking if NodeLocal DNSCache is enabled
In some clusters, Pods do not directly access CoreDNS Service but use a NodeLocal DNSCache instead to access the local DNS cache on the node.
Kubernetes's official documentation provides detailed information on NodeLocal DNSCache. ([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/?utm_source=chatgpt.com))

If this feature is enabled, further checks are required:
- Whether the DaemonSet for NodeLocal DNSCache is functioning correctly
- If the Pod's `nameserver` is set to the local cache address
- If there are any issues with the node-side local DNS proxy

---

### 10. Final checks on kube-proxy/CNI/node network
If none of the previous steps identify a problem, consider the following possibilities:
- Abnormalities in kube-proxy rules
- CNI-related issues
- Node network problems
- Blockages in the path from the Pod to the DNS Service IP

If necessary, perform additional checks:
    kubectl get pod -o wide -n kube-system
    kubectl describe pod <podA> -n <ns>

If needed, go directly to the node to investigate:
- kube-proxy configuration
- iptables/nft settings
- CNI performance
- Connectivity between the node and the DNS service

---

If the DNS service itself is unreachable, I will proceed to check whether NetworkPolicy is blocking UDP/TCP port 53, as well as whether NodeLocal DNSCache or kube-proxy are enabled in the cluster, or if there are any abnormalities with the node network. ([kubernetes.io](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/?utm_source=chatgpt.com))

---

## Memory Template
First, remember this main sequence:

Name / FQDN  
→ resolv.conf  
→ dnsPolicy  
→ CoreDNS  
→ Service / EndpointSlice  
→ DNS availability  
→ NetworkPolicy  
→ NodeLocal DNSCache  
→ Node network

Also, remember this statement:

Always consider it a DNS issue first, not a port issue.

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
#Troubleshooting
#Cloud-native interview
#Ops interview